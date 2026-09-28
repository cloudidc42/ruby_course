# Part 074: Environment Variable, Credentials, และ Multi-Environment Config

> **Step ครอบคลุมใน Part นี้:** Step 731–740
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 067 เรื่อง Rails encrypted credentials เบื้องต้น และ Part 071
> เรื่องการเก็บ API key ของ Stripe มาก่อน — Part นี้จะ **recap สั้นๆ** เฉพาะกลไกพื้นฐาน ไม่สอนซ้ำ
> ตั้งแต่ต้น แล้วขยายไปสู่เรื่อง multi-environment credentials, `config_for`, `config.x`, และ
> ปรัชญา twelve-factor app ที่ยังไม่เคยพูดถึง)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem `dotenv-rails` (ทดสอบจริงบน Ruby 3.3.6,
> Rails 8.1.4, dotenv-rails 3.2.0)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม:**
>
> Part นี้พิเศษกว่า Part อื่นๆ ตรงที่ทุกกลไกที่สอน — environment switching, encrypted credentials,
> `config_for`, `config.x`, dotenv — **เป็นกลไกที่ทำงานอยู่ในเครื่องล้วนๆ ไม่ต้องพึ่งเครือข่ายหรือ
> บัญชีบริการภายนอกใดๆ เลย** ทีมผู้เขียนจึงสร้างแอป Rails 8.1.4 จริงขึ้นมาในเครื่อง (แยกจาก
> repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้งหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) แล้วรันคำสั่งทุกคำสั่ง
> ที่ปรากฏใน Part นี้จริง — รวมถึงกรณี **ที่ตั้งใจทำให้พัง** เช่น ลบ `master.key` ทิ้งแล้วดูว่า Rails
> error อย่างไร เพื่อให้เนื้อหาส่วน "ความเป็นจริงเชิงปฏิบัติการ" แม่นยำร้อยเปอร์เซ็นต์ ไม่ใช่การเดา
> ผลลัพธ์ทุกอย่างที่แสดงเป็น code block ผลลัพธ์จริง (ไม่ใช่ตัวอย่างสมมติ) คือค่าที่ได้จากการรันจริง
> ส่วน API key/secret ทุกตัวที่ปรากฏเป็นค่าปลอมที่ตั้งชื่อให้ชัดเจน (เช่น
> `<YOUR_WEATHER_API_KEY_HERE>`) ไม่ใช่รูปแบบ key จริงของผู้ให้บริการรายใด

ยินดีต้อนรับสู่ Part ที่สองของ **Phase 12: DevOps & Deployment** ต่อจาก Part 073 ที่สอนการห่อแอป
Rails ด้วย Docker ตอนนี้เรามีแอปที่ "รันได้ในกล่อง" แล้ว แต่คำถามที่ตามมาทันทีคือ — **กล่องเดียวกันนี้
ต้องรันได้ทั้งบนเครื่อง dev ของเรา, บน CI server ตอนรัน test, และบน production server จริง โดยที่
ค่า config บางอย่างต้องต่างกันไปในแต่ละที่** (เช่น database คนละตัว, log level คนละระดับ) และ
**ค่าบางอย่างต้องเป็นความลับที่ไม่มีใครเห็นได้นอกจากคนที่ควรเห็น** (เช่น API key ของบริการภายนอก)
Part นี้จะตอบคำถามนี้อย่างเป็นระบบ ตั้งแต่กลไกพื้นฐานของ Rails environments ไปจนถึงกรอบการตัดสินใจ
ว่า "ค่าไหนควรเก็บที่ไหน" ซึ่งเป็นทักษะที่วิศวกร Rails ทุกคนต้องใช้จริงในการทำงานทุกวัน

## สารบัญของ Part นี้

- Step 731: Rails Environments สามระดับ — `development`/`test`/`production` และ `RAILS_ENV` คุมอะไร
- Step 732: Credentials แยกตาม Environment — `bin/rails credentials:edit --environment production`
- Step 733: `ENV["VAR_NAME"]` vs Rails Credentials — กรอบการตัดสินใจว่าอะไรไปที่ไหน
- Step 734: `dotenv-rails` — โหลดไฟล์ `.env` ใน development และทำไมต้อง gitignore เสมอ
- Step 735: `Rails.application.config_for` — Custom structured config ต่อ environment
- Step 736: Custom config namespace ด้วย `config.x`
- Step 737: ปรัชญา Twelve-Factor App — "strict separation of config from code"
- Step 738: Secrets Rotation — ความเป็นจริงเชิงปฏิบัติการเมื่อ key รั่วไหล
- Step 739: Security Checklist สำหรับ Config/Credentials
- Step 740: แบบฝึกหัด — ออกแบบระบบ multi-environment config ให้บริการภายนอกจริงจังหนึ่งตัว

---

## Step 731: Rails Environments สามระดับ — `development`/`test`/`production` และ `RAILS_ENV` คุมอะไร

### Environment คืออะไร

**Environment** ใน Rails คือ "โหมดการทำงาน" ของแอปที่กำหนดว่าโค้ดชุดเดียวกันควรมีพฤติกรรมต่างกัน
อย่างไรในแต่ละบริบท ตั้งแต่ `rails new` Rails สร้างไฟล์ config มาให้สามไฟล์เสมอ:

```bash
ls config/environments/
# development.rb
# production.rb
# test.rb
```

แต่ละไฟล์คือ Ruby block ที่ configure ค่าต่างๆ ของแอปสำหรับ environment นั้นโดยเฉพาะ:

```ruby
# config/environments/development.rb (ตัดมาบางส่วน จากแอปที่ generate จริงด้วย Rails 8.1.4)
Rails.application.configure do
  config.enable_reloading = true       # โหลดโค้ดใหม่ทุกครั้งที่แก้ไฟล์ ไม่ต้อง restart server
  config.eager_load = false            # ไม่โหลดทุกคลาสตั้งแต่ boot (boot เร็วขึ้น)
  config.consider_all_requests_local = true  # หน้า error แสดง stack trace เต็ม
  config.action_controller.perform_caching = false
end
```

```ruby
# config/environments/production.rb (ตัดมาบางส่วน)
Rails.application.configure do
  config.enable_reloading = false      # ห้ามโหลดโค้ดใหม่ระหว่างรัน (เร็วกว่า ปลอดภัยกว่า)
  config.eager_load = true             # โหลดทุกคลาสตั้งแต่ boot (ตรวจ error ได้ก่อน request แรก)
  config.consider_all_requests_local = false # หน้า error ไม่แสดง stack trace ให้ผู้ใช้เห็น
  config.action_controller.perform_caching = true
  config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
end
```

สังเกตว่า `development.rb` กับ `production.rb` ตั้งค่าตรงข้ามกันแทบทุกจุด — นี่คือเหตุผลที่ Rails
แยกไฟล์ให้ตั้งแต่แรก: **development เน้นความเร็วในการพัฒนา (feedback loop สั้น), production เน้น
ความเร็วและความปลอดภัยตอนรันจริง**

ส่วน `test.rb` จะคล้าย production ในหลายจุด (ไม่ reload code) แต่เน้นความเร็วในการรัน test suite
เป็นหลัก (เช่นปิด logging ที่ไม่จำเป็น)

### `RAILS_ENV` คือสวิตช์ที่บอกว่าจะโหลดไฟล์ไหน

```bash
# ค่า default เมื่อไม่ตั้ง RAILS_ENV คือ "development"
bin/rails runner 'puts Rails.env'
# => development

# ตั้ง RAILS_ENV ตรงๆ เพื่อสลับ environment
RAILS_ENV=test bin/rails runner 'puts Rails.env'
# => test

RAILS_ENV=production bin/rails runner 'puts Rails.env'
# => production
```

**ทดสอบจริงกับแอปทดลอง** ยืนยันพฤติกรรมนี้ตรงตามที่คาดไว้ทุกกรณี — ค่า default เป็น
`development` เสมอเมื่อไม่ตั้ง `RAILS_ENV` และค่าจะเปลี่ยนตาม environment variable ทันที

`Rails.env` ไม่ใช่ String ธรรมดา แต่เป็น `ActiveSupport::StringInquirer` ทำให้เขียนเช็คแบบอ่านง่าย
ได้:

```ruby
Rails.env.production?   # => true ถ้า RAILS_ENV=production
Rails.env.development?  # => true ถ้า RAILS_ENV=development (หรือไม่ตั้งค่าเลย)
Rails.env.test?         # => true ถ้า RAILS_ENV=test
```

### คำสั่งอื่นๆ ที่ผูกกับ `RAILS_ENV` เหมือนกัน

```bash
# เปิด console ใน environment ที่ต้องการ (เทียบเท่า RAILS_ENV=production bin/rails console)
bin/rails console -e production

# รัน server ใน environment ที่ต้องการ
bin/rails server -e production

# รัน migration เฉพาะ environment
RAILS_ENV=production bin/rails db:migrate
```

> **ข้อควรระวัง:** `RACK_ENV` เป็นคนละตัวกับ `RAILS_ENV` (Rack ใช้ `RACK_ENV`, Rails ใช้
> `RAILS_ENV`) Rails สมัยใหม่จะ sync สองค่านี้ให้อัตโนมัติเมื่อบูตผ่าน `bin/rails`/`config.ru`
> มาตรฐาน แต่ถ้าเขียนสคริปต์ boot เอง หรือใช้ deployment tool บางตัวที่ตั้งแค่ `RACK_ENV` ให้ตรวจสอบ
> ว่า `RAILS_ENV` ถูกตั้งไปด้วยเสมอ ไม่เช่นนั้นแอปอาจบูตด้วย environment ที่ไม่ตั้งใจ

### สร้าง environment ที่สี่ได้ไหม (เช่น `staging`)

ได้ — Rails ไม่ได้จำกัดไว้แค่สามชื่อนี้ วิธีทำคือคัดลอก `config/environments/production.rb` เป็น
`config/environments/staging.rb` แล้วปรับค่าที่ต่างจาก production (เช่น `config.log_level = :debug`
เพื่อ debug ง่ายกว่า production จริง) จากนั้นรันด้วย `RAILS_ENV=staging` ได้ทันที — แนวคิดนี้จะกลับมา
เกี่ยวข้องอีกครั้งใน Step 732 เรื่อง credentials แยกต่อ environment เพราะ credentials ผูกกับชื่อ
environment โดยตรง ไม่ได้จำกัดแค่สามชื่อมาตรฐานเช่นกัน

| Environment | ใช้ตอนไหน | eager_load | reload code | error page | cache |
|---|---|---|---|---|---|
| `development` | เขียนโค้ดบนเครื่องตัวเอง | ปิด | เปิด | เห็น stack trace เต็ม | ปิด |
| `test` | รัน automated test (RSpec/Minitest) | ปิด | ปิด | - | ปิด |
| `production` | เซิร์ฟเวอร์จริงที่ผู้ใช้เข้าถึง | เปิด | ปิด | หน้า error ทั่วไป | เปิด |

---

## Step 732: Credentials แยกตาม Environment

### Recap สั้นๆ: Rails Encrypted Credentials (ทบทวนจาก Part 067/071)

Part 067 และ 071 สอนไปแล้วว่า Rails มีระบบเก็บ secret แบบเข้ารหัสในตัว: ไฟล์
`config/credentials.yml.enc` ถูกเข้ารหัสด้วย master key ที่อยู่ใน `config/master.key`
(ไฟล์ `.key` ไม่ถูก commit เข้า git — `.gitignore` กันไว้ให้อัตโนมัติตั้งแต่ `rails new`) เปิดแก้ด้วย:

```bash
EDITOR="code --wait" bin/rails credentials:edit
```

แล้วอ่านค่าในโค้ดด้วย `Rails.application.credentials.dig(:key, :nested_key)`

**สิ่งที่ Part นี้จะเพิ่มเข้ามา** คือ: ไฟล์ `credentials.yml.enc` ไฟล์เดียวที่ใช้ร่วมกันทุก
environment มีปัญหาจริงจังข้อหนึ่ง — **ถ้าใครก็ตามที่มี `master.key` เปิดไฟล์นี้ได้ จะเห็น secret
ของ "ทุก" environment พร้อมกัน** รวมถึง production secret key ตัวจริงที่ต้องอยู่บนเซิร์ฟเวอร์
production เท่านั้น นักพัฒนาที่ทำงานบนเครื่อง dev ปกติไม่ควรมีสิทธิ์เห็น production API key เลย
ด้วยซ้ำ — Rails จึงรองรับการแยก credentials ต่อ environment มาตั้งแต่ Rails 6

### สร้าง credentials เฉพาะ production

```bash
EDITOR="code --wait" bin/rails credentials:edit --environment production
```

**ผลจากการรันจริง** (ทดสอบด้วยแอปทดลองจริง — ครั้งแรกที่รันคำสั่งนี้กับ environment ใหม่):

```
Adding config/master.key to store the encryption key: 3a7b4247785fc040550b58f4038a6fad

Save this in a password manager your team can access.

If you lose the key, no one, including you, can access anything encrypted with it.

      create  config/credentials/production.key
      append  .gitignore

Editing config/credentials/production.yml.enc...
File encrypted and saved.
```

สังเกตสิ่งที่เกิดขึ้น:

1. Rails สร้างไฟล์ **ใหม่ทั้งคู่** — `config/credentials/production.yml.enc` (encrypted content)
   และ `config/credentials/production.key` (master key เฉพาะของ production เท่านั้น — **คนละไฟล์
   คนละคีย์กับ `config/master.key` ที่ใช้กับ credentials ตัว default**)
2. `.gitignore` ถูก append บรรทัดใหม่ให้อัตโนมัติ (`/config/credentials/*.key`) — ไม่ต้องเพิ่มเอง
3. ไฟล์ `.enc` (encrypted) ยัง commit เข้า git ได้ตามปกติ มีแค่ `.key` เท่านั้นที่ห้าม commit

โครงสร้างไฟล์หลังรันคำสั่งนี้:

```
config/
├── master.key                        # ไม่ commit — key ของ credentials.yml.enc (default/dev/test)
├── credentials.yml.enc                # commit ได้ — ใช้เมื่อไม่มีไฟล์เฉพาะ environment
└── credentials/
    ├── production.key                 # ไม่ commit — key ของ production.yml.enc เท่านั้น
    └── production.yml.enc             # commit ได้ — secret เฉพาะ production
```

### พิสูจน์ด้วยการทดสอบจริงว่าแต่ละ environment อ่านค่าคนละไฟล์กันจริง

ตั้งค่า credentials คนละค่ากันระหว่างไฟล์ default (ใช้เป็น dev) กับไฟล์ production:

```yaml
# config/credentials.yml.enc (ถอดรหัสแล้ว — ใช้กับ development/test เพราะไม่มีไฟล์เฉพาะ)
secret_key_base: 92394a0c2dba3075c70b765bf42cbc605839cb944d37bfbc28be6c6021023ff...

weather_api:
  api_key: "<YOUR_DEV_WEATHER_API_KEY_HERE>"
```

```yaml
# config/credentials/production.yml.enc (ถอดรหัสแล้ว — ใช้เฉพาะ RAILS_ENV=production)
secret_key_base: b1e6f2a4c8d0e3f5a7b9c1d3e5f7a9b1c3d5e7f9a1b3c5d7e9f1a3b5c7d9e1f3a5b7...

weather_api:
  api_key: "<YOUR_PROD_WEATHER_API_KEY_HERE>"
```

อ่านค่าด้วยโค้ดเดียวกันเป๊ะ แต่รันคนละ `RAILS_ENV`:

```bash
bin/rails runner 'puts Rails.application.credentials.dig(:weather_api, :api_key)'
# => <YOUR_DEV_WEATHER_API_KEY_HERE>

RAILS_ENV=production bin/rails runner 'puts Rails.application.credentials.dig(:weather_api, :api_key)'
# => <YOUR_PROD_WEATHER_API_KEY_HERE>

RAILS_ENV=test bin/rails runner 'puts Rails.application.credentials.dig(:weather_api, :api_key)'
# => <YOUR_DEV_WEATHER_API_KEY_HERE>   -- test ไม่มีไฟล์เฉพาะ จึง fallback ไปใช้ไฟล์ default
```

**ผลลัพธ์นี้คือของจริงจากการรันบนแอปทดลอง** ยืนยันกลไกได้ครบสองข้อสำคัญ:

1. `RAILS_ENV=production` จะมองหา `config/credentials/production.yml.enc` +
   `config/credentials/production.key` **ก่อนเสมอ** ถ้าเจอคู่ไฟล์นี้จะใช้ทันที
2. environment ที่**ไม่มี**ไฟล์เฉพาะของตัวเอง (เช่น `test` ในตัวอย่างนี้) จะ **fallback** ไปใช้
   `config/credentials.yml.enc` + `config/master.key` แทนโดยอัตโนมัติ — ดังนั้นถ้าต้องการให้ `test`
   มี credentials ของตัวเองจริงๆ ก็สั่ง `bin/rails credentials:edit --environment test` ได้เช่นกัน

### ดูเนื้อหาโดยไม่เปิด editor

```bash
bin/rails credentials:show
bin/rails credentials:show --environment production
```

มีประโยชน์มากตอน debug ว่า credentials ที่ deploy ไปคือค่าที่ตั้งใจจริงหรือไม่ (รันบนเซิร์ฟเวอร์
production ได้โดยตรง ถ้ามี `production.key` อยู่ในเครื่องนั้น)

### ทำไมการแยกไฟล์นี้สำคัญในทางปฏิบัติ

- นักพัฒนาทุกคนในทีมมี `config/master.key` (ใช้ตอน dev) แต่ **ไม่จำเป็นต้องมี**
  `config/credentials/production.key` เลย — จำกัดคนที่เห็น production secret ได้จริงตาม
  **principle of least privilege**
- Deploy pipeline ใส่ไฟล์ `production.key` เข้าไปในเซิร์ฟเวอร์ production เท่านั้น (ผ่าน secret
  manager ของแพลตฟอร์ม ไม่ใช่ commit เข้า repo) — Part 076 เรื่อง Kamal deploy จะกลับมาสอนวิธี
  ส่ง key นี้เข้าเซิร์ฟเวอร์จริงแบบปลอดภัยอีกครั้ง
- ถ้า `master.key` ของเครื่อง dev คนใดคนหนึ่งรั่วไหล **production ไม่กระทบเลย** เพราะคนละคีย์กัน
  คนละไฟล์ (ตรงข้ามกับตอนใช้ไฟล์เดียวรวมทุก environment ที่ key เดียวรั่วเท่ากับทุกอย่างรั่ว)

---

## Step 733: `ENV["VAR_NAME"]` vs Rails Credentials — กรอบการตัดสินใจว่าอะไรไปที่ไหน

นี่คือคำถามที่นักพัฒนา Rails มือใหม่สับสนบ่อยที่สุด: "ค่านี้ควรเก็บใน credentials หรือใน ENV
variable?" คำตอบสั้นๆ คือ **ทั้งสองเป็นที่เก็บค่าที่ถูกต้องทั้งคู่ แต่ตอบโจทย์คนละแบบ**

### ข้อเท็จจริงจากโค้ดจริงของ Rails 8.1 (ที่ generate มาให้ตั้งแต่ `rails new`)

ลองดูว่า Rails เองเลือกใช้ ENV var ตรงไหนบ้างในไฟล์ default ที่ generate ให้:

```ruby
# config/puma.rb (Rails 8.1.4 generate ให้ตรงนี้เป๊ะ — ไม่ได้ดัดแปลง)
threads_count = ENV.fetch("RAILS_MAX_THREADS", 3)
threads threads_count, threads_count

port ENV.fetch("PORT", 3000)
```

```ruby
# config/environments/production.rb (ตัดมาบางส่วน)
config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
```

```yaml
# config/database.yml — รูปแบบมาตรฐานเมื่อ deploy ด้วย DATABASE_URL (ทีม PaaS อย่าง Render/Heroku/
# Fly.io ฉีดค่านี้ให้อัตโนมัติเมื่อ provision database ให้)
production:
  url: <%= ENV["DATABASE_URL"] %>
```

**ทดสอบจริง:** ยืนยันว่ากลไกนี้ทำงานถูกต้อง — เมื่อตั้ง `DATABASE_URL` เป็น connection string
รูปแบบ `postgresql://user:pass@host:5432/db_name` แล้วรัน `ActiveRecord::Base.configurations`
Active Record จะ**อ่าน adapter จาก scheme ของ URL เอง** (`postgresql://` → ใช้
`postgresql_adapter`) โดยไม่ต้องระบุ `adapter:` แยกต่างหากเลย — พิสูจน์ได้จาก error message ที่ได้
ตอนทดสอบ (สภาพแวดล้อมทดสอบใช้ SQLite ไม่มี gem `pg` ติดตั้ง) ซึ่งเป็น error ที่บอกชัดเจนว่า
"กำลังจะโหลด postgresql adapter แต่หา gem `pg` ไม่เจอ" — ยืนยันว่า Rails **parse URL ได้ถูกต้อง
และเลือก adapter ถูกต้อง** ก่อนจะไปสะดุดที่ปัญหาอื่น (ไม่มี gem `pg` ในสภาพแวดล้อมทดสอบซึ่งเป็นเรื่อง
คาดไว้อยู่แล้ว ในแอปจริงของหลักสูตรที่มี `gem "pg"` ใน Gemfile ตาม Part 021 จะไม่เจอปัญหานี้)

สังเกตว่าทั้งสี่ตัวอย่างนี้ — `RAILS_MAX_THREADS`, `PORT`, `RAILS_LOG_LEVEL`, `DATABASE_URL` — **ไม่
มีตัวไหนอยู่ใน credentials เลย** ทั้งที่ `DATABASE_URL` มักมีรหัสผ่านฝังอยู่ในตัว! นี่คือกุญแจสำคัญ
ของการตัดสินใจ

### กรอบการตัดสินใจ

| คำถามที่ต้องถาม | คำตอบคือ ENV var | คำตอบคือ Rails Credentials |
|---|---|---|
| ใครเป็นคนกำหนดค่านี้? | **แพลตฟอร์ม/infra** เป็นคนสร้างและฉีดให้ตอนรัน (PaaS, orchestrator, container runtime) | **มนุษย์ในทีม** เป็นคนเลือกและตั้งค่าเอง (สมัคร API key จากเว็บผู้ให้บริการ) |
| ค่านี้ต่างกันในแต่ละ instance ของ environment เดียวกันไหม? | ใช่ได้ (เช่น `PORT` ที่ platform สุ่มให้แต่ละ container) | ปกติไม่ต่าง (secret เดียวกันใช้กับทุก instance ของ production) |
| เป็นความลับที่ถ้ารั่วแล้วอันตรายไหม? | อาจจะ (เช่น `DATABASE_URL` มีรหัสผ่าน) — แต่ "ความลับ" ระดับนี้ เก็บใน platform's own secret store อยู่แล้ว | ใช่ชัดเจน (API secret key, private key, webhook signing secret) |
| ต้องการให้ versioned คู่กับ commit ของโค้ดไหม? | ไม่จำเป็น — เป็นเรื่องของ infra ไม่ใช่ business logic | ควร — ถ้า deploy โค้ดเวอร์ชันเก่ากลับไป อยากได้ secret ชุดที่คู่กันจริง |
| ต้อง rotate บ่อยแค่ไหน โดยใครเป็นคน rotate? | เปลี่ยนได้จาก dashboard ของแพลตฟอร์มทันที ไม่ต้อง deploy โค้ดใหม่ | เปลี่ยนต้อง `credentials:edit` แล้ว deploy โค้ดใหม่ (แต่ปลอดภัยกว่าเพราะ audit ผ่าน git history ได้) |
| ตัวอย่าง | `PORT`, `RAILS_MAX_THREADS`, `WEB_CONCURRENCY`, `DATABASE_URL`, `REDIS_URL`, `RAILS_LOG_LEVEL` | `stripe.secret_key`, `aws.secret_access_key`, `weather_api.api_key`, `secret_key_base` |

### กฎง่ายๆ ที่ใช้จำได้ในทางปฏิบัติ

> **"ถ้า infrastructure เป็นคนสร้าง/จัดการค่านี้ → ใช้ ENV var
> ถ้ามนุษย์ในทีมเป็นคนเลือกค่านี้เอง (สมัคร API key จากเว็บ, ตั้ง secret เอง) → ใช้ Rails
> credentials"**

จุดที่คนสับสนบ่อยที่สุดคือ `DATABASE_URL` เพราะมันมีรหัสผ่านฝังอยู่ ดูเหมือนต้องเป็น "secret" — แต่
เหตุผลที่มันอยู่ใน ENV ไม่ใช่ credentials คือ **ฐานข้อมูลถูก provision โดยแพลตฟอร์ม (Managed
Postgres addon) ไม่ใช่โดยทีมเรา** แพลตฟอร์มสร้าง connection string ใหม่ทุกครั้งที่ provision
database ใหม่ (เช่นตอนสร้าง staging environment ชั่วคราว) เราจึงไม่สามารถ "เขียนค่านี้ลง credentials
ที่ commit คู่กับโค้ด" ได้ตั้งแต่แรกอยู่ดี เพราะค่ายังไม่มีอยู่จนกว่า infrastructure จะสร้าง database
ให้เสร็จ — ต่างจาก Stripe secret key ที่มนุษย์คนหนึ่งไปสมัครที่เว็บ Stripe มาเองแล้วนำมาใส่ในระบบ

**สรุปสั้น:** ทั้งสองช่องทางคือ "ที่เก็บ config ที่แยกจากโค้ด" เหมือนกันตามหลัก twelve-factor
(Step 737) ต่างกันแค่ *ใครเป็นเจ้าของวงจรชีวิตของค่านั้น*

---

## Step 734: `dotenv-rails` — โหลดไฟล์ `.env` ใน development และทำไมต้อง gitignore เสมอ

### ปัญหา: ตอน dev ไม่มี "แพลตฟอร์ม" ที่จะฉีด ENV var ให้

บน production จริง ค่าอย่าง `PORT`, `DATABASE_URL` ถูกตั้งโดยแพลตฟอร์ม deploy (Kamal, Render,
Fly.io ฯลฯ) แต่บนเครื่อง dev ของนักพัฒนาแต่ละคน **ไม่มีแพลตฟอร์มแบบนั้น** ถ้าต้องการทดสอบพฤติกรรม
ที่ขึ้นกับ ENV var (เช่น เปลี่ยน `RAILS_MAX_THREADS` ทดสอบ concurrency) ก็ต้อง `export` เองทุกครั้ง
ก่อนรันคำสั่ง ซึ่งลืมง่ายและไม่ shareable ระหว่างทีม — gem `dotenv-rails` แก้ปัญหานี้ด้วยการอ่านไฟล์
`.env` แล้วตั้งเป็น ENV variable ให้อัตโนมัติตอน boot

### ติดตั้ง

```ruby
# Gemfile
group :development, :test do
  gem "dotenv-rails"
end
```

```bash
bundle install
```

**ทดสอบจริง:** ติดตั้งสำเร็จ `dotenv-rails 3.2.0` เข้ากับ Rails 8.1.4 โดยไม่มี conflict ใดๆ

> **สำคัญ:** ใส่ gem นี้ไว้ใน group `:development, :test` **เท่านั้น** ห้ามใส่แบบ global (ไม่มี
> group) เพราะ production ไม่ควรพึ่งไฟล์ `.env` เลย — ค่าทั้งหมดใน production ต้องมาจาก ENV จริงที่
> แพลตฟอร์ม deploy ฉีดให้ (ดู Step 737) การปล่อยให้ `dotenv-rails` โหลดใน production เป็นความเสี่ยง
> ที่ไม่จำเป็น (เช่นถ้ามีไฟล์ `.env` หลงเหลือบนเซิร์ฟเวอร์จากการ deploy ผิดพลาด จะถูกโหลดโดยไม่ตั้งใจ)

### สร้างไฟล์ `.env`

```bash
# .env (อยู่ที่ root ของโปรเจกต์ ระดับเดียวกับ Gemfile)
PORT=3001
RAILS_MAX_THREADS=3
GREETING_FROM_DOTENV=hello-from-dotenv
```

**ทดสอบจริง** อ่านค่ากลับผ่าน `bin/rails runner`:

```bash
bin/rails runner 'puts ENV["PORT"]; puts ENV["RAILS_MAX_THREADS"]; puts ENV["GREETING_FROM_DOTENV"]'
# => 3001
# => 3
# => hello-from-dotenv
```

### ยืนยันว่า production ไม่โดนกระทบ (ตามที่ตั้งใจ)

```bash
RAILS_ENV=production bin/rails runner 'puts ENV["GREETING_FROM_DOTENV"].inspect'
# => nil
```

**ผลลัพธ์นี้คือของจริง** ยืนยันว่าเพราะ `dotenv-rails` อยู่ใน group `:development, :test` เท่านั้น
Bundler จึงไม่โหลด gem นี้เลยตอนรันด้วย `RAILS_ENV=production` ค่าที่ตั้งใน `.env` จึงไม่มีผลใดๆ
ต่อ production ตามที่ต้องการ

### ไฟล์ `.env` แยกต่อ environment ได้เช่นกัน

`dotenv-rails` รองรับไฟล์เฉพาะ environment คล้ายกับ credentials:

```bash
# .env.test (override เฉพาะตอนรันด้วย RAILS_ENV=test)
GREETING_FROM_DOTENV=hello-from-dotenv-test
```

**ทดสอบจริง:**

```bash
RAILS_ENV=test bin/rails runner 'puts ENV["GREETING_FROM_DOTENV"]'
# => hello-from-dotenv-test
```

ยืนยันว่า `.env.test` มีความสำคัญกว่า `.env` เมื่อรันด้วย `RAILS_ENV=test` ลำดับการโหลดไฟล์ของ
`dotenv-rails` (ไฟล์ที่โหลดทีหลังจะไม่ทับค่าที่ตั้งไปแล้วก่อนหน้า) คือ:

1. `.env.#{RAILS_ENV}.local` (เฉพาะเครื่องนี้ ไม่ commit)
2. `.env.local` (เฉพาะเครื่องนี้ ไม่ commit, ใช้ทุก environment)
3. `.env.#{RAILS_ENV}` (เช่น `.env.test` — commit ได้ถ้าไม่มี secret จริง)
4. `.env` (ค่า default ของทุก environment)

### ทำไม `.env` ต้อง gitignore เสมอ — ตรวจสอบว่า Rails กันให้แล้วจริงหรือไม่

สมมติฐานที่ต้องพิสูจน์: Rails 8 สร้าง `.gitignore` เริ่มต้นที่กัน `.env` ไว้แล้วโดยไม่ต้องทำอะไรเพิ่ม
— **ทดสอบจริง** ด้วยการรัน `rails new` แล้วเปิดดู `.gitignore` ที่ได้:

```bash
cat .gitignore
```

```gitignore
# Ignore bundler config.
/.bundle

# Ignore all environment files.
/.env*

# Ignore all logfiles and tempfiles.
/log/*
/tmp/*
!/log/.keep
!/tmp/.keep

# ... (ส่วนอื่นตัดออก)

# Ignore key files for decrypting credentials and more.
/config/*.key
```

**ยืนยันแล้วว่าเป็นจริง:** `rails new` (Rails 8.1.4) สร้างบรรทัด `/.env*` ไว้ให้ตั้งแต่แรกเริ่ม —
รูปแบบ `.env*` (มี `*` ต่อท้าย) ครอบคลุมทั้ง `.env`, `.env.local`, `.env.test`, `.env.production`
ทุกไฟล์ในตระกูลนี้พร้อมกันด้วย wildcard เดียว ทดสอบยืนยันด้วย `git check-ignore -v .env` ในโปรเจกต์
ทดลองที่ init git แล้ว ได้ผลลัพธ์:

```
.gitignore:11:/.env*	.env
```

หมายความว่าไฟล์ `.env` ถูก ignore เพราะ pattern ที่บรรทัด 11 ของ `.gitignore` ตรงตามที่คาดไว้

> **ข้อควรระวังสำหรับแอปเก่า:** ถ้าโปรเจกต์ถูกสร้างด้วย Rails เวอร์ชันเก่ากว่านี้มาก (ก่อนที่ Rails
> จะเพิ่มบรรทัดนี้เข้า `.gitignore` เริ่มต้น) หรือถ้ามีใครเผลอลบบรรทัดนี้ออกไป **ต้องตรวจสอบด้วยมือ
> ทุกครั้งที่เริ่มโปรเจกต์ใหม่หรือรับช่วงโปรเจกต์เก่าต่อ** — รันคำสั่ง
> `git check-ignore -v .env` ถ้าไม่มี output ใดๆ แปลว่า `.env` **ไม่ได้** ถูก ignore และมีความเสี่ยง
> ที่จะถูก commit เข้าไปโดยไม่ตั้งใจ (ครั้งเดียวก็เพียงพอที่จะทำให้ secret หลุดเข้า git history
> ถาวร แม้จะลบไฟล์ออกในภายหลังก็ตาม เพราะ git เก็บ history ทุก commit ไว้)

### แนวปฏิบัติที่ดี: commit ไฟล์ตัวอย่างแทน

แม้ `.env` ห้าม commit แต่ทีมควร commit ไฟล์ `.env.example` (หรือ `.env.sample`) ที่มีชื่อ key
ครบทุกตัวแต่ใส่ค่าปลอม/ค่าว่าง เพื่อให้สมาชิกใหม่ในทีมรู้ว่าต้องตั้งค่าอะไรบ้าง:

```bash
# .env.example (commit เข้า git ได้ปกติ — ไม่มีค่าจริง)
PORT=3000
RAILS_MAX_THREADS=3
GREETING_FROM_DOTENV=
```

```bash
# สมาชิกใหม่ในทีมรัน
cp .env.example .env
# แล้วกรอกค่าจริงเอง
```

---

## Step 735: `Rails.application.config_for` — Custom Structured Config ต่อ Environment

### ปัญหาที่ ENV var และ credentials ยังไม่ตอบโจทย์

บางครั้งค่า config ที่ต้องใช้:

- **ไม่ใช่ความลับ** (ไม่จำเป็นต้องเข้ารหัส) แต่ก็ **ไม่ควรเป็น ENV var เดี่ยวๆ** เพราะมีโครงสร้าง
  ซับซ้อน (มีหลาย key ซ้อนกัน)
- ต้อง**ต่างกันไปตาม environment** (base URL ของ third-party service ตอน dev อาจชี้ไป sandbox/mock
  server, ตอน production ชี้ไป server จริง)
- อยากเก็บเป็น **YAML ที่ commit เข้า git ได้** เพราะไม่ใช่ secret (เช่น timeout, หน่วย, feature
  flag)

Rails มี `Rails.application.config_for` ตอบโจทย์นี้โดยเฉพาะ

### สร้างไฟล์ config

```yaml
# config/weather_service.yml
shared:
  base_url: "https://api.weather-example.com/v1"
  timeout_seconds: 5
  units: "metric"

development:
  timeout_seconds: 10

test:
  base_url: "https://api.weather-example.test/v1"
  timeout_seconds: 1
```

> **จุดที่คนเข้าใจผิดบ่อยที่สุด:** คีย์สำหรับ "ค่าเริ่มต้นที่ใช้ร่วมกันทุก environment" **ชื่อ
> `shared:` ไม่ใช่ `default:`** — นี่ไม่ใช่ธรรมเนียมตั้งชื่อทั่วไป แต่เป็นชื่อคีย์ตายตัวที่
> `config_for` มองหาจริงในซอร์สโค้ด (`rails/application.rb`) ถ้าตั้งชื่อผิดเป็น `default:` จะไม่มี
> การ merge ใดๆ เกิดขึ้นเลย (แต่ละ environment จะเห็นเฉพาะ key ของตัวเองเท่านั้น ไม่ error ให้เห็น
> ด้วย ทำให้ bug แบบนี้ตรวจจับยากมากถ้าไม่ทดสอบจริง) — Part นี้ตรวจสอบกับซอร์สโค้ดจริงของ
> `railties-8.1.4` แล้วว่ากลไกคือ `shared.deep_merge(config)` ซึ่งหมายความว่าเป็น **deep merge**
> ไม่ใช่แค่ shallow merge ระดับบนสุด (ถ้ามี hash ซ้อนกันหลายชั้น ค่าที่ environment override จะ
> merge ลงไปถึงชั้นในสุดโดยไม่ทับทั้งก้อน)

### อ่านค่าด้วย `config_for`

```bash
bin/rails runner 'pp Rails.application.config_for(:weather_service)'
```

**ผลการทดสอบจริงในแต่ละ environment:**

```
# development
{:base_url=>"https://api.weather-example.com/v1",
 :timeout_seconds=>10,
 :units=>"metric"}

# test (RAILS_ENV=test)
{:base_url=>"https://api.weather-example.test/v1",
 :timeout_seconds=>1,
 :units=>"metric"}

# production (RAILS_ENV=production — ไม่มี key "production:" ในไฟล์เลยด้วยซ้ำ)
{:base_url=>"https://api.weather-example.com/v1",
 :timeout_seconds=>5,
 :units=>"metric"}
```

สังเกตว่า **production ไม่จำเป็นต้องมี key `production:` ในไฟล์เลย** ถ้าไม่มี key ของ environment
นั้นอยู่ `config_for` จะใช้ค่าจาก `shared:` ล้วนๆ — ทดสอบแล้วยืนยันว่าไม่ error และคืนค่า `shared`
ที่ merge เข้ากับ hash ว่างได้อย่างถูกต้อง

### เข้าถึงค่าด้วย dot notation

ค่าที่ได้กลับมาไม่ใช่ Hash ธรรมดา แต่เป็น `ActiveSupport::OrderedOptions` (ยืนยันด้วย
`cfg.class` ตอนทดสอบจริง) ทำให้เข้าถึงแบบ dot notation ได้:

```ruby
config = Rails.application.config_for(:weather_service)
config.base_url          # => "https://api.weather-example.com/v1"
config.timeout_seconds   # => 10 (บน development)
config.nonexistent_key   # => nil (ไม่ raise error แม้ key ไม่มีอยู่จริง — ทดสอบยืนยันแล้ว)
```

### ใช้ร่วมกับ ERB ได้เหมือน `database.yml`/`storage.yml`

`config_for` ประมวลผลไฟล์ผ่าน ERB ก่อนเสมอ (เหมือน `config/database.yml`) จึงผสมกับ ENV var หรือ
credentials ในไฟล์เดียวกันได้:

```yaml
# config/weather_service.yml
shared:
  base_url: "https://api.weather-example.com/v1"
  timeout_seconds: <%= ENV.fetch("WEATHER_TIMEOUT_SECONDS", 5) %>
  units: "metric"
```

---

## Step 736: Custom Config Namespace ด้วย `config.x`

### ปัญหา: จะรวม config_for + credentials เข้าด้วยกันแล้วเรียกใช้จากที่เดียวยังไง

Step 735 ให้ค่า config ที่ไม่ใช่ secret (base_url, timeout) ส่วน Step 732 ให้ค่า secret (api_key)
— ในโค้ดจริงมักอยากรวมสองอย่างนี้เป็น object เดียวเพื่อเรียกใช้สะดวก Rails มี namespace ชื่อ
`config.x` ไว้สำหรับ **การตั้งค่าที่เป็นของแอปเราเอง ไม่ได้ผูกกับ gem ตัวไหนโดยเฉพาะ** (ต่างจาก
`config.action_controller.xxx` ที่เป็นของ Action Controller หรือ `config.active_record.xxx` ที่
เป็นของ Active Record)

### เขียน initializer

```ruby
# config/initializers/weather_service.rb
Rails.application.configure do
  weather_config = config_for(:weather_service)

  config.x.weather_service.base_url = weather_config.base_url
  config.x.weather_service.timeout_seconds = weather_config.timeout_seconds
  config.x.weather_service.units = weather_config.units
  config.x.weather_service.api_key = Rails.application.credentials.dig(:weather_api, :api_key)
end
```

**ทดสอบจริง** เรียกค่ากลับจากทุก environment:

```bash
bin/rails runner 'x = Rails.application.config.x.weather_service
  puts x.base_url; puts x.timeout_seconds; puts x.units; puts x.api_key'
# => https://api.weather-example.com/v1
# => 10
# => metric
# => <YOUR_DEV_WEATHER_API_KEY_HERE>

RAILS_ENV=production bin/rails runner 'x = Rails.application.config.x.weather_service
  puts x.base_url; puts x.timeout_seconds; puts x.units; puts x.api_key'
# => https://api.weather-example.com/v1
# => 5
# => metric
# => <YOUR_PROD_WEATHER_API_KEY_HERE>
```

ยืนยันว่า `config.x.weather_service` รวมค่าจากทั้ง `config_for` (base_url, timeout_seconds,
units ที่ต่างกันตาม environment) และ `credentials` (api_key ที่ต่างกันตาม environment เช่นกัน)
เข้าเป็น object เดียวสำเร็จ ในโค้ดส่วนอื่นของแอป (เช่น service object ที่เรียก weather API) เรียกใช้
แค่ `Rails.application.config.x.weather_service.api_key` โดยไม่ต้องรู้เลยว่าเบื้องหลังค่านี้มาจาก
credentials หรือ config_for — **ซ่อนรายละเอียดว่าค่าเก็บที่ไหนไว้จุดเดียว (initializer)**

### `config.x` ทำงานอย่างไรเบื้องหลัง (auto-vivifying เหมือน OpenStruct)

**ทดสอบจริงเพื่อยืนยันพฤติกรรม:**

```bash
bin/rails runner '
  puts Rails.application.config.x.class
  puts Rails.application.config.x.totally_undeclared_thing.class
  puts Rails.application.config.x.totally_undeclared_thing.foo.inspect
'
# => Rails::Application::Configuration::Custom
# => ActiveSupport::OrderedOptions
# => nil
```

ผลลัพธ์นี้ยืนยันว่า `config.x` เป็น object ชนิด `Rails::Application::Configuration::Custom` ที่ทำงาน
คล้าย `OpenStruct` — **เข้าถึง key ไหนก็ได้แม้ไม่เคย declare มาก่อน** โดยจะ auto-vivify เป็น
`ActiveSupport::OrderedOptions` ใหม่ให้เอง (ไม่ raise `NoMethodError` แบบ object ทั่วไป) และการเข้าถึง
key ที่ไม่มีอยู่จริงจะได้ `nil` กลับมาเสมอ ไม่ error — คุณสมบัตินี้สะดวกมากตอนเขียนโค้ดเร็วๆ แต่ก็มี
ข้อเสีย: **พิมพ์ชื่อ key ผิดจะไม่มี error เตือนเลย** (ได้ `nil` เงียบๆ) จึงควรมี test คลุมค่าที่สำคัญ
เสมอ (ตัวอย่างแบบฝึกหัด Step 740 จะโชว์วิธีเขียน test คลุมจุดนี้)

### ทำไมไม่เก็บเป็น constant หรือ class variable ธรรมดาไปเลย

ข้อดีของการผ่าน `config.x` แทนการเขียน constant ตรงๆ ในโค้ด:

1. **สอดคล้องกับวงจรชีวิตของ Rails app** — ตั้งค่าตอน initialize ครั้งเดียว ไม่ใช่ทุกครั้งที่เรียก
   method ซ้ำ (ต่างจากการเรียก `config_for`/`credentials.dig` กระจายอยู่ทั่วโค้ด ซึ่งอ่านไฟล์ซ้ำ
   ทุกครั้งถ้าไม่ cache เอง)
2. **Mock/stub ง่ายตอนเขียน test** — เปลี่ยนค่าชั่วคราวด้วย
   `Rails.application.config.x.weather_service.api_key = "test-key"` ในเทสได้ตรงๆ
3. **จุดเดียวที่ต้องแก้เมื่อเปลี่ยนแหล่งที่มาของ config** — ถ้าวันหนึ่งย้าย `base_url` จาก YAML
   ไปเป็นดึงจาก database แทน แก้แค่ initializer จุดเดียว โค้ดที่เรียกใช้ `config.x.weather_service`
   ไม่ต้องแก้เลย

---

## Step 737: ปรัชญา Twelve-Factor App — "Strict Separation of Config from Code"

### Twelve-Factor App คืออะไร

**The Twelve-Factor App** เป็นแนวทางปฏิบัติ (methodology) สำหรับสร้างซอฟต์แวร์แบบ SaaS ที่เผยแพร่
โดยทีม Heroku ในปี 2011 ประกอบด้วยหลักการ 12 ข้อ ที่ Rails ecosystem รับมาใช้เป็นมาตรฐานโดยปริยาย
Part นี้ไม่ได้ครอบคลุมทั้ง 12 ข้อ (จะพูดถึงข้ออื่นกระจายไปในหลาย Part ของ Phase 12) แต่จะเจาะเฉพาะ
**ข้อที่ 3: Config** เพราะเกี่ยวข้องโดยตรงกับทุก Step ที่ผ่านมา

### หลักการ: "Store config in the environment"

ข้อความต้นฉบับจากเอกสาร twelve-factor คือ:

> "The twelve-factor app stores config in environment variables... Config varies substantially
> across deploys, code does not."

หลักการนี้แบ่งสิ่งที่อยู่ในแอปออกเป็นสองกลุ่มอย่างเข้มงวด (**strict separation**):

- **Code** — สิ่งที่เหมือนกันทุก deploy (dev, staging, production ใช้โค้ด commit เดียวกันได้)
- **Config** — สิ่งที่ต่างกันไปในแต่ละ deploy (database connection, API key, feature flag,
  hostname) — **ทุกอย่างที่อาจต่างกันระหว่าง environment ถือเป็น config หมด**

### เกณฑ์ทดสอบที่ตัวเอกสารเสนอ (litmus test)

Twelve-factor เสนอวิธีทดสอบง่ายๆ ว่าค่าหนึ่งควรเป็น "config" หรือไม่: **ถ้า codebase สามารถ
open-source ได้ทันทีโดยไม่ทำให้ credential ใดๆ หลุดออกไป แสดงว่าแยก config ออกจากโค้ดได้ถูกต้องแล้ว**
ลองเอาเกณฑ์นี้มาทดสอบกับโค้ดตัวอย่างที่เขียนมาตลอด Part นี้:

```ruby
# ผิดหลักการ — ถ้า push โค้ดนี้ขึ้น public repo, secret หลุดทันที
Stripe.api_key = "sk_live_abc123FAKE_EXAMPLE_KEY"

# ถูกหลักการ — โค้ดนี้ push ขึ้น public repo ได้อย่างปลอดภัย ไม่มี secret หลุดออกไปเลย
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

`config/credentials.yml.enc` ที่ commit เข้า git ผ่านเกณฑ์นี้เพราะ **เข้ารหัสไว้** — คนอ่าน repo
เห็นแค่ตัวอักษรไร้ความหมาย ไม่เห็น secret จริง (ต้องมี `master.key` ที่ไม่ได้ commit ถึงจะถอดรหัสได้)
ส่วน `.env` ผ่านเกณฑ์นี้เพราะ **ไม่ commit เลย** (ถูก gitignore ตั้งแต่ Step 734)

### ทำไมหลักการนี้สำคัญกับความปลอดภัยและ portability

**ด้านความปลอดภัย:** การแยก config ออกจากโค้ดอย่างเข้มงวดทำให้ "ใครมีสิทธิ์อ่านโค้ด" กับ "ใครมี
สิทธิ์เห็น secret" เป็นคนละกลุ่มกันได้ นักพัฒนาใหม่ที่เพิ่งเข้าทีมสามารถ clone repo มาอ่านโค้ดทั้งหมด
ได้ทันที (โค้ดไม่มีความลับ) แต่ยังไม่มีสิทธิ์รัน production จริงจนกว่าจะได้รับ credentials
production แยกต่างหาก (ตรงกับ principle of least privilege ที่จะกล่าวถึงใน Step 739)

**ด้าน portability:** โค้ดชุดเดียวกันเป๊ะ (commit เดียวกัน) รันได้ทั้งบนเครื่อง dev, CI server,
staging, และ production โดยไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียว — ต่างกันแค่ค่า config ที่ environment
นั้นมอบให้ (ผ่าน ENV var หรือ credentials file ที่ต่างกัน) นี่คือรากฐานที่ทำให้ Part 073 (Docker)
ทำงานได้จริง: **image เดียวกันเป๊ะที่ build ครั้งเดียว รันได้ทุก environment** เพราะไม่มี config
ฝังอยู่ใน image เลย ทุกอย่างถูกฉีดเข้าไปตอน runtime ผ่าน ENV var/mounted credentials file

### ข้อผิดพลาดคลาสสิกที่ละเมิดหลักการนี้

```ruby
# ผิด — ผูกโค้ดกับ environment โดยตรง แทนที่จะผูกกับ config
if Rails.env.production?
  API_BASE_URL = "https://api.realservice.com"
else
  API_BASE_URL = "https://api.sandbox-service.com"
end
```

โค้ดแบบนี้มีปัญหาสองข้อ: (1) เพิ่ม environment ใหม่ (เช่น `staging`) ต้องแก้โค้ดนี้ทุกครั้ง (2) ทดสอบ
พฤติกรรมของ "production URL" จากเครื่อง dev ไม่ได้เลยนอกจากปลอม `Rails.env` ทั้งที่ควรทำได้ด้วยการ
เปลี่ยนแค่ config เท่านั้น (`config_for` ตาม Step 735 แก้ปัญหานี้ได้ตรงจุด — เพิ่ม environment ใหม่
แค่เพิ่ม key ใน YAML ไม่ต้องแตะโค้ด Ruby เลย)

---

## Step 738: Secrets Rotation — ความเป็นจริงเชิงปฏิบัติการเมื่อ Key รั่วไหล

ทุกระบบที่ทำถูกต้องตาม Step 731–737 จะมีวันหนึ่งที่ต้อง **rotate** (เปลี่ยน) secret ไม่ว่าจะเป็น
เพราะ key รั่วไหลจริง (เช่น commit หลุดเข้า git โดยไม่ตั้งใจ, พนักงานลาออกที่เคยมีสิทธิ์เข้าถึง,
หรือ log แสดง secret โดยไม่ตั้งใจ) หรือเป็นการ rotate ตามนโยบายความปลอดภัยตามรอบเวลา (เช่นทุก 90 วัน)
Step นี้จะพาไล่ดู "ความเป็นจริงเชิงปฏิบัติการ" ของสองสถานการณ์ที่ต่างกันมาก

### สถานการณ์ที่ 1: API key ของบริการภายนอกรั่วไหล (เช่น `weather_api.api_key`)

นี่คือกรณีที่**ง่ายกว่า**เพราะกลไก per-environment credentials ที่สอนใน Step 732 ช่วยจำกัดผลกระทบ
ไว้แล้ว ขั้นตอนที่ต้องทำ:

1. **ไปสร้าง key ใหม่ที่ผู้ให้บริการ** (อย่าเพิ่งลบ key เก่า — ให้ใช้คู่กันชั่วคราวถ้าผู้ให้บริการ
   รองรับ เพื่อป้องกัน downtime ระหว่างเปลี่ยนผ่าน)
2. **อัปเดต credentials ของ environment ที่เกี่ยวข้อง** — ถ้า key ที่รั่วเป็นของ production ก็ต้อง
   `bin/rails credentials:edit --environment production` (ต้องมี `production.key` ถึงจะทำได้ ตรงกับ
   หลัก least privilege ที่จำกัดคนทำขั้นตอนนี้ไว้แค่คนที่ควรมีสิทธิ์)
3. **Deploy โค้ดใหม่** — เพราะ credentials ถูก commit เป็นไฟล์ `.enc` เข้า git จริง การเปลี่ยนค่า
   คือการเปลี่ยนไฟล์ที่ commit ต้อง deploy ให้ไฟล์ใหม่ไปถึง production ก่อนถึงจะมีผลจริง
4. **Restart/redeploy process ที่รันอยู่** — process ที่รันอยู่ก่อนหน้าอ่าน credentials เข้า
   memory ไปแล้วตอน boot (`config.x` ที่ตั้งใน initializer อ่านครั้งเดียวตอน boot ตาม Step 736)
   ต้อง restart ถึงจะอ่านค่าใหม่
5. **ยืนยันว่าใช้งานได้จริงด้วย key ใหม่** แล้วค่อย **revoke key เก่าที่ผู้ให้บริการ**
6. **ตรวจสอบ log/audit ของผู้ให้บริการ** ว่า key เก่าถูกใช้งานผิดปกติไปหรือไม่ในช่วงที่รั่ว
   (ถ้าผู้ให้บริการมี usage log — ควรเลือกผู้ให้บริการที่มี audit log เสมอสำหรับ credential สำคัญ)

### สถานการณ์ที่ 2: `RAILS_MASTER_KEY` เองรั่วไหล (กรณีร้ายแรงกว่ามาก)

ถ้า `master.key` เองรั่วไหล (เช่น เผลอ commit ไฟล์ `.key` เข้า git — Step 739 จะพูดถึงวิธีป้องกัน)
**ทุก secret ที่เข้ารหัสไว้ในไฟล์ `credentials.yml.enc` ตัวนั้นถือว่ารั่วทั้งหมดทันที** เพราะใครก็ตาม
ที่มี master key ถอดรหัสไฟล์ได้ทั้งไฟล์ ไม่ใช่แค่ key เดียว วิธีแก้คือต้อง **สร้างชุด credentials
ใหม่ทั้งหมด** ไม่ใช่แค่แก้ค่าเดียว — **ทดสอบขั้นตอนนี้จริงทั้งหมด** บนแอปทดลอง:

```bash
# ขั้นตอนที่ 1: backup เนื้อหาปัจจุบันไว้ก่อน (ถอดรหัสด้วย master key เดิมที่ยังใช้ได้อยู่)
bin/rails credentials:show > /tmp/rotated_plain.yml

# ขั้นตอนที่ 2: ลบไฟล์ credentials + master key เดิมทิ้งทั้งคู่
rm config/master.key config/credentials.yml.enc

# ขั้นตอนที่ 3: รัน credentials:edit ใหม่ — Rails จะสร้าง master.key ใหม่ให้อัตโนมัติ
EDITOR="code --wait" bin/rails credentials:edit
# วาง secret_key_base เดิมกลับเข้าไป (ห้ามสุ่มใหม่ ไม่งั้น session/cookie ของผู้ใช้ทุกคนจะ invalid
# ทันที) ส่วน secret อื่นที่ต้อง rotate จริงๆ (เช่น weather_api.api_key ที่เป็นต้นเหตุ) ให้ใส่ค่าใหม่
```

**ผลจากการรันจริง:**

```
Adding config/master.key to store the encryption key: 3a033c8a5a96c3109f23e0e799c1ab36

Save this in a password manager your team can access.

If you lose the key, no one, including you, can access anything encrypted with it.

      create  config/master.key

Editing config/credentials.yml.enc...
File encrypted and saved.
```

จากนั้นยืนยันว่า **master key เดิม ใช้ไม่ได้อีกต่อไปแล้วจริง** (ทดสอบด้วยการตั้ง
`RAILS_MASTER_KEY` เป็นค่าเดิมที่ถูกแทนที่ไปแล้ว):

```bash
RAILS_MASTER_KEY="<คีย์เดิมที่ถูกแทนที่ไปแล้ว>" bin/rails runner 'puts 1'
```

```
ActiveSupport::MessageEncryptor::InvalidMessage
  from .../active_support/encrypted_file.rb:72:in `decrypt'
```

**ยืนยันได้ชัดเจน:** ทันทีที่สร้าง master key ใหม่ ไฟล์เก่าที่เข้ารหัสด้วย key เดิมจะถอดรหัสไม่ได้
อีกต่อไปแม้จะรู้ค่า key เดิมก็ตาม (เพราะไฟล์ `.enc` ถูกเข้ารหัสใหม่ทั้งไฟล์ด้วย key ใหม่แล้ว) — นี่คือ
พฤติกรรมที่ต้องการพอดี เพราะ key เดิมถือว่า "ประนีประนอม" (compromised) ไปแล้ว

> **กับดักสำคัญที่ต้องรู้ก่อนทำจริง:** ทดสอบพบว่าถ้า `master.key` **หายไปทั้งไฟล์** (ไม่มีอยู่เลย
> และไม่มี `RAILS_MASTER_KEY` ใน ENV) แอปที่ `RAILS_ENV=development`/`test` จะยัง **boot ผ่านได้
> เฉยๆ** และ `credentials.dig(...)` จะคืนค่า `nil` เงียบๆ แทนที่จะ error ให้เห็นชัดเจน — แต่ถ้าเป็น
> `RAILS_ENV=production` และไม่มีการตั้ง `secret_key_base` ไว้ที่อื่นเลย (ไม่มีทั้ง
> `credentials/production.key` และไม่มี `RAILS_MASTER_KEY`) แอปจะ **บูตไม่ขึ้นเลย** ด้วย error
> ที่ชัดเจนมาก:
> ```
> ArgumentError: Missing `secret_key_base` for 'production' environment,
> set this string with `bin/rails credentials:edit`
> ```
> ส่วนกรณีที่ไฟล์ key **มีอยู่แต่ผิด** (ไม่ตรงกับไฟล์ `.enc`) จะ raise
> `ActiveSupport::MessageEncryptor::InvalidMessage` ทันทีที่ boot **ทุก environment รวมถึง
> development** — สรุปคือ "ไม่มี key เลย" กับ "มี key แต่ผิด" ให้ผลลัพธ์ต่างกัน ต้องรู้ทั้งสองแบบ
> เพื่อ debug อาการ "deploy ไม่ขึ้น" ได้ถูกจุดตอนหน้างานจริง

### สิ่งที่ต้อง "sync" ให้ครบเมื่อ rotate

จุดที่ผิดพลาดบ่อยที่สุดในการ rotate จริงคือ **แก้ไม่ครบทุกที่ที่ค่านั้นถูกเก็บซ้ำ** ทีมที่ rotate
key ต้องไล่เช็คให้ครบ:

1. `config/credentials.yml.enc` (หรือไฟล์เฉพาะ environment ใน `config/credentials/`)
2. ไฟล์ `.env`/`.env.production` บนเครื่องที่เกี่ยวข้อง (ถ้าค่านั้นดันไปอยู่ใน ENV แทนที่จะเป็น
   credentials — เตือนว่านี่คือเหตุผลสำคัญที่ Step 733 ต้องตัดสินใจให้ถูกตั้งแต่แรก เพื่อไม่ให้ secret
   เดียวกันกระจายอยู่หลายที่จนตามแก้ไม่ครบ)
3. **Secret store ของแพลตฟอร์ม deploy เอง** (เช่น GitHub Actions secrets ที่จะสอนใน Part 075,
   หรือ secret ของ Kamal ใน Part 076) — ถ้า CI/CD pipeline อ่าน secret ตัวนี้ไปใช้ตอน build/test
   ด้วย ต้องอัปเดตที่นั่นด้วย ไม่ใช่แค่ในโค้ด
4. **Cache ที่อาจ cache ค่าเก่าไว้** เช่นถ้าเคยอ่าน config ผ่าน `Rails.cache` เอง (rare กรณีนี้แต่
   เกิดขึ้นได้ถ้ามีคน cache credentials ไว้เพื่อลด I/O)
5. **บันทึกลง incident log/runbook** ว่า rotate เมื่อไหร่ เพราะอะไร ใครเป็นคนทำ — สำคัญมากถ้าต้อง
   สืบสวนย้อนหลังว่าช่วงที่ key รั่วมีการใช้งานผิดปกติเกิดขึ้นหรือไม่

---

## Step 739: Security Checklist สำหรับ Config/Credentials

รวบรวมทุกหลักการจาก Step 731–738 เป็น checklist ที่ใช้ตรวจสอบได้จริงก่อนขึ้น production:

### ไม่ log secret เด็ดขาด

```ruby
# ผิดร้ายแรง — secret หลุดเข้า log file ทันที (log มักถูกส่งไป log aggregator ภายนอก มีคนเห็นเยอะกว่า
# ที่คิด และ log เก็บย้อนหลังได้นานกว่า credentials file เสียอีก)
Rails.logger.info("Calling weather API with key: #{Rails.application.config.x.weather_service.api_key}")

# ถูกต้อง — log แค่ข้อมูลที่จำเป็นต่อการ debug โดยไม่มี secret ปน
Rails.logger.info("Calling weather API for city=#{city}")
```

Rails มี `config.filter_parameters` ที่กรอง parameter ที่ชื่อเข้าเงื่อนไข (เช่น `password`,
`token`) ออกจาก log ของ request โดยอัตโนมัติ (Rails 8 ตั้งค่า default ให้บางส่วนแล้ว) แต่ **ไม่ได้
ครอบคลุมค่าที่เรา `Rails.logger.info` เอง** ต้องระวังเองเสมอเวลาเขียน log statement ที่มีตัวแปรที่
อาจมี secret ปนอยู่

### ไม่ commit `.env`/`*.key` เด็ดขาด — และตรวจสอบซ้ำเสมอ

- ตรวจสอบว่า `.gitignore` มี `/.env*` และ `/config/*.key` (และ `/config/credentials/*.key` ถ้าใช้
  per-environment credentials) — Step 734 แสดงวิธีตรวจสอบด้วย `git check-ignore -v` แล้ว
- ก่อน commit ทุกครั้งที่แตะไฟล์ config ให้รัน `git status` ตรวจดูว่าไม่มีไฟล์ที่ไม่ควร track
  ติดมาด้วย (`git add -A` แบบไม่ดูอะไรเลยเป็นสาเหตุคลาสสิกของ credential หลุด)
- ติดตั้ง pre-commit hook หรือใช้ CI scan (เช่น `gitleaks`, GitHub secret scanning ที่ผูกกับ
  push protection) เป็นแนวป้องกันชั้นที่สอง ไม่ใช่พึ่ง `.gitignore` อย่างเดียว

### ใช้ credentials คนละชุดต่อ environment เสมอ

- อย่างน้อย production ควรมี credentials แยกจาก development/test เสมอ (ตาม Step 732) —
  โดยเฉพาะแอปที่มีทีมใหญ่ขึ้นเรื่อยๆ ยิ่งสำคัญ เพราะจำนวนคนที่ "ควร" เห็น production secret ควรน้อย
  กว่าจำนวนคนที่เข้าถึงเครื่อง dev ได้มาก
- อย่าใช้ API key ตัวเดียวกันทั้ง sandbox/test mode และ live mode ของบริการภายนอก (ย้อนกลับไปที่
  บทเรียนจาก Part 071 เรื่อง Stripe test key vs live key)

### Principle of Least Privilege สำหรับ API key

- สมัคร API key ของบริการภายนอกด้วย**สิทธิ์ที่จำเป็นเท่านั้น** ถ้าผู้ให้บริการรองรับการจำกัดสิทธิ์
  (scope/permission) เช่น AWS IAM policy ที่จำกัดแค่ bucket เดียว แทนที่จะให้สิทธิ์เต็ม account,
  หรือ API key ที่ตั้งค่าเป็น read-only ถ้าแอปไม่จำเป็นต้องเขียนข้อมูล
- จำกัด IP/domain ที่อนุญาตให้เรียกได้ถ้าผู้ให้บริการรองรับ (ลดผลกระทบถ้า key รั่วแต่ยังใช้จากที่อื่น
  ไม่ได้)
- แยก key ตาม service/ทีมที่ใช้งาน แทนที่จะใช้ key เดียวกันทั่วทั้งองค์กร — ทำให้ revoke เฉพาะจุดได้
  โดยไม่กระทบระบบอื่น และตรวจสอบได้ว่า key ไหนถูกใช้งานผิดปกติ

### Checklist สรุป (ใช้ตรวจก่อน deploy จริงทุกครั้ง)

- [ ] `.env`, `config/master.key`, `config/credentials/*.key` ไม่ถูก track ใน git (`git ls-files`
      ต้องไม่มีชื่อไฟล์เหล่านี้)
- [ ] production มี credentials แยกจาก development/test
- [ ] ไม่มี `Rails.logger`/`puts`/`p` ที่ print ค่า secret ตรงๆ ในโค้ด
- [ ] ค่าที่มาจาก infrastructure (`PORT`, `DATABASE_URL`, ฯลฯ) ใช้ ENV var, ค่าที่มนุษย์ตั้งเอง
      (API key, secret key) ใช้ credentials — ไม่ปนกันจนสับสนว่า source of truth อยู่ที่ไหน
- [ ] มี runbook/เอกสารขั้นตอนการ rotate secret แต่ละตัว (ใครทำได้, ทำอย่างไร, ต้องแจ้งใครบ้าง)
- [ ] API key ของบริการภายนอกทุกตัวตั้งสิทธิ์แบบจำกัดที่สุดเท่าที่ใช้งานได้จริง (least privilege)
- [ ] มี `.env.example` ให้สมาชิกใหม่ในทีม ไม่ต้องเดาว่าต้องตั้งค่าอะไรบ้าง

---

## Step 740: แบบฝึกหัด — ออกแบบระบบ Multi-Environment Config ให้บริการภายนอกจริงจังหนึ่งตัว

### โจทย์

สมมติแอปของเราต้องเชื่อมต่อกับบริการแจ้งเตือนผ่าน SMS ชื่อสมมติ **"QuickSMS"** ซึ่งต้องใช้ทั้งค่าที่
เป็นความลับ (API key) และค่าที่เป็น config ทั่วไปที่ต่างกันไปตาม environment (base URL, timeout,
เบอร์ sender เริ่มต้น) จงสร้างระบบ config ให้ QuickSMS ตามหลักการทั้งหมดที่เรียนมาใน Part นี้:

1. Credentials แยกระหว่าง development (ใช้ไฟล์ default) กับ production (ไฟล์เฉพาะ)
2. Custom config ผ่าน `config_for` สำหรับค่าที่ไม่ใช่ secret ครอบคลุมทั้ง `development`, `test`,
   `production`
3. รวมทุกอย่างไว้ใต้ `config.x.quick_sms`
4. เขียน service object ง่ายๆ ที่ใช้ config นี้ พร้อม test ยืนยันว่า config ถูกต้องในแต่ละ environment

### เฉลย

**ขั้นตอนที่ 1 — ตั้ง credentials**

```bash
# ตั้งค่า default (ใช้กับ development และ test เพราะยังไม่แยกไฟล์เฉพาะให้ test)
EDITOR="code --wait" bin/rails credentials:edit
```

```yaml
# config/credentials.yml.enc (ถอดรหัสแล้ว)
secret_key_base: ...

quick_sms:
  api_key: "<YOUR_DEV_QUICKSMS_API_KEY_HERE>"
```

```bash
# ตั้งค่าเฉพาะ production
EDITOR="code --wait" bin/rails credentials:edit --environment production
```

```yaml
# config/credentials/production.yml.enc (ถอดรหัสแล้ว)
secret_key_base: ...

quick_sms:
  api_key: "<YOUR_PROD_QUICKSMS_API_KEY_HERE>"
```

**ขั้นตอนที่ 2 — สร้างไฟล์ config ที่ไม่ใช่ secret**

```yaml
# config/quick_sms.yml
shared:
  base_url: "https://api.quicksms-example.com/v1"
  timeout_seconds: 5
  default_sender_id: "MYAPP"

development:
  base_url: "https://sandbox.quicksms-example.com/v1"
  timeout_seconds: 10

test:
  base_url: "https://sandbox.quicksms-example.com/v1"
  timeout_seconds: 1
  default_sender_id: "TEST"
```

**ขั้นตอนที่ 3 — Initializer รวมทุกอย่างไว้ใต้ `config.x.quick_sms`**

```ruby
# config/initializers/quick_sms.rb
Rails.application.configure do
  quick_sms_config = config_for(:quick_sms)

  config.x.quick_sms.base_url = quick_sms_config.base_url
  config.x.quick_sms.timeout_seconds = quick_sms_config.timeout_seconds
  config.x.quick_sms.default_sender_id = quick_sms_config.default_sender_id
  config.x.quick_sms.api_key = Rails.application.credentials.dig(:quick_sms, :api_key)
end
```

**ขั้นตอนที่ 4 — Service object ที่ใช้ config นี้**

```ruby
# app/services/quick_sms_client.rb
class QuickSmsClient
  class ConfigurationError < StandardError; end

  def initialize(config: Rails.application.config.x.quick_sms)
    @config = config
    validate_configuration!
  end

  def deliver(to:, message:, sender_id: @config.default_sender_id)
    {
      endpoint: "#{@config.base_url}/messages",
      timeout: @config.timeout_seconds,
      sender_id: sender_id,
      to: to,
      message: message,
      # ในระบบจริงตรงนี้จะเป็นการเรียก HTTP client จริง (Net::HTTP/Faraday) พร้อมแนบ
      # Authorization header ด้วย @config.api_key — Part นี้เน้นเรื่อง config ไม่ใช่ HTTP client
      # จึงคืนค่าเป็น Hash ให้เห็นว่าค่า config แต่ละตัวถูกใช้ตรงไหนบ้าง
    }
  end

  private

  def validate_configuration!
    raise ConfigurationError, "QuickSMS base_url ไม่ได้ตั้งค่า" if @config.base_url.blank?
    raise ConfigurationError, "QuickSMS api_key ไม่ได้ตั้งค่า" if @config.api_key.blank?
  end
end
```

**ทดสอบจริงทุก environment** (นี่คือส่วนที่ verify ว่าโจทย์ "verified across development/test"
ผ่านจริง):

```bash
bin/rails runner '
  client = QuickSmsClient.new
  pp client.deliver(to: "+66812345678", message: "สวัสดี")
'
```

```
# ผลลัพธ์จริงบน development
{:endpoint=>"https://sandbox.quicksms-example.com/v1/messages",
 :timeout=>10,
 :sender_id=>"MYAPP",
 :to=>"+66812345678",
 :message=>"สวัสดี"}
```

```bash
RAILS_ENV=test bin/rails runner '
  client = QuickSmsClient.new
  pp client.deliver(to: "+66812345678", message: "สวัสดี")
'
```

```
# ผลลัพธ์จริงบน test
{:endpoint=>"https://sandbox.quicksms-example.com/v1/messages",
 :timeout=>1,
 :sender_id=>"TEST",
 :to=>"+66812345678",
 :message=>"สวัสดี"}
```

```bash
RAILS_ENV=production bin/rails runner '
  client = QuickSmsClient.new
  pp client.deliver(to: "+66812345678", message: "สวัสดี")
'
```

```
# ผลลัพธ์จริงบน production
{:endpoint=>"https://api.quicksms-example.com/v1/messages",
 :timeout=>5,
 :sender_id=>"MYAPP",
 :to=>"+66812345678",
 :message=>"สวัสดี"}
```

**ยืนยันครบทุกจุดของโจทย์:** `base_url` ต่างกันสามค่าตามที่ตั้งใจ (`sandbox...` สำหรับ dev/test,
`api...` จริงสำหรับ production), `timeout_seconds` ต่างกันทั้งสามค่าตามที่ override ไว้,
`sender_id` เป็นค่า `shared` (`MYAPP`) ยกเว้น test ที่ override เป็น `TEST` และค่า `api_key` (แม้จะ
ไม่ได้ print ออกมาตรงๆ ในตัวอย่างนี้ แต่ผ่านการตรวจสอบใน `validate_configuration!` แล้วว่าไม่ blank
ในทุก environment — ถ้าลืมตั้ง credentials ให้ environment ไหน `QuickSmsClient.new` จะ raise
`ConfigurationError` ทันทีตั้งแต่ initialize แทนที่จะปล่อยให้ error ไปโผล่ตอนเรียก API จริง)

**เขียน test ยืนยันด้วย RSpec (แนวทางจาก Part 046):**

```ruby
# spec/services/quick_sms_client_spec.rb
require "rails_helper"

RSpec.describe QuickSmsClient do
  describe "#deliver" do
    it "ใช้ base_url, timeout, และ sender_id จาก config ของ environment ปัจจุบัน" do
      client = described_class.new
      result = client.deliver(to: "+66812345678", message: "test")

      expect(result[:endpoint]).to eq("#{Rails.application.config.x.quick_sms.base_url}/messages")
      expect(result[:timeout]).to eq(Rails.application.config.x.quick_sms.timeout_seconds)
      expect(result[:sender_id]).to eq(Rails.application.config.x.quick_sms.default_sender_id)
    end

    it "อนุญาตให้ override sender_id เฉพาะครั้งได้" do
      client = described_class.new
      result = client.deliver(to: "+66812345678", message: "test", sender_id: "CUSTOM")

      expect(result[:sender_id]).to eq("CUSTOM")
    end
  end

  describe "#initialize" do
    it "raise ConfigurationError ถ้า api_key ไม่ได้ตั้งค่า" do
      fake_config = Rails.application.config.x.quick_sms.dup
      fake_config.api_key = nil

      expect { described_class.new(config: fake_config) }
        .to raise_error(QuickSmsClient::ConfigurationError, /api_key/)
    end
  end
end
```

test ตัวสุดท้ายสาธิตประโยชน์ของการรับ `config:` เป็น dependency injection ใน `initialize` (แทนที่
จะอ่าน `Rails.application.config.x.quick_sms` ตรงๆ ในทุก method) — ทำให้ทดสอบ "กรณี config ผิดพลาด"
ได้โดยไม่ต้องไปยุ่งกับ credentials ไฟล์จริงเลย

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม environment `staging` เข้าไปในระบบ QuickSMS นี้ (สร้าง
   `config/environments/staging.rb` จากการคัดลอก `production.rb`, เพิ่ม key `staging:` ใน
   `config/quick_sms.yml` ให้ชี้ไป sandbox เหมือน development แต่ตั้ง `default_sender_id` เป็น
   `"STAGING"`, และสร้าง credentials เฉพาะ staging ด้วย
   `bin/rails credentials:edit --environment staging`) แล้วรัน
   `RAILS_ENV=staging bin/rails runner` ยืนยันว่าได้ค่าที่ถูกต้องครบทุกจุด
2. เขียน rake task ชื่อ `config:audit` ที่ไล่ตรวจสอบทุก environment ที่มีอยู่ (development, test,
   production, staging) ว่า `Rails.application.config.x.quick_sms.api_key` ไม่ใช่ `nil`/ค่าว่าง
   สักตัว (ใบ้: ใช้ `Rails::Command::Environment` หรือรัน `system("RAILS_ENV=#{env} bin/rails
   runner ...")` แยกกระบวนการสำหรับแต่ละ environment เพราะ Rails boot ได้ครั้งเดียวต่อหนึ่ง process
   เท่านั้น เปลี่ยน `Rails.env` กลางคันไม่ได้)
3. ปรับ `QuickSmsClient` ให้รองรับการ "dry run" ในทุก environment ยกเว้น production
   (`config.x.quick_sms.dry_run = !Rails.env.production?` ตั้งใน initializer) โดยที่ `dry_run: true`
   จะ log ข้อความที่จะส่งแทนการเรียก endpoint จริง — ออกแบบให้ยังคง raise
   `ConfigurationError` เหมือนเดิมถ้า config ไม่ครบ แม้จะอยู่ในโหมด dry run ก็ตาม (เพราะการตรวจสอบ
   config ควรเข้มงวดเท่ากันทุกโหมด ต่างกันแค่ตอนเรียก API จริงหรือไม่)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า Rails environments ทั้งสามระดับ (`development`/`test`/`production`) คือชุด config ที่
  ต่างกันโดยสิ้นเชิงในเรื่อง eager loading, code reloading, error page, และ caching และ `RAILS_ENV`
  คือสวิตช์ตัวเดียวที่กำหนดว่าไฟล์ `config/environments/*.rb` ไหนจะถูกโหลด — พร้อมรู้วิธีเพิ่ม
  environment ที่สี่ (เช่น `staging`) ได้เอง
- ตั้งค่า **credentials แยกต่อ environment** ได้จริงด้วย
  `bin/rails credentials:edit --environment production` และเข้าใจกลไก fallback ที่ environment
  ไม่มีไฟล์เฉพาะจะย้อนไปใช้ `credentials.yml.enc`/`master.key` ตัว default โดยอัตโนมัติ — ทดสอบยืนยัน
  จริงว่าแต่ละไฟล์แยกคีย์เข้ารหัสกันเด็ดขาด
- มีกรอบการตัดสินใจที่ชัดเจนว่าค่าไหนควรเป็น **ENV var** (ค่าที่ infrastructure เป็นคนกำหนด เช่น
  `PORT`, `DATABASE_URL`, `RAILS_MAX_THREADS`) กับค่าไหนควรเป็น **Rails credentials** (ค่าที่มนุษย์
  เลือกและจัดการเอง เช่น API secret key) แทนที่จะเดาสุ่มหรือทำตามความเคยชิน
- ใช้ `dotenv-rails` โหลดไฟล์ `.env` ใน development/test ได้ถูกต้อง และยืนยันแล้วว่า Rails 8 กันไฟล์
  `.env` ออกจาก git ให้อัตโนมัติตั้งแต่ `rails new` ผ่าน pattern `/.env*` ใน `.gitignore`
- ใช้ `Rails.application.config_for` สร้าง custom structured config ต่อ environment ได้ พร้อมรู้จุด
  ที่คนเข้าใจผิดบ่อยที่สุด — คีย์สำหรับ merge ค่าที่ใช้ร่วมกันชื่อ **`shared:`** ไม่ใช่ `default:`
  (ตรวจสอบกับซอร์สโค้ด Rails จริงแล้ว)
- รวม config หลายแหล่ง (credentials + config_for) เข้าเป็น object เดียวด้วย **`config.x`**
  namespace ที่ทำงานแบบ OpenStruct auto-vivifying และเข้าใจข้อดี/ข้อเสียของความยืดหยุ่นนี้
- เข้าใจปรัชญา **Twelve-Factor App** ข้อ "Config" อย่างลึกซึ้ง — หลักการ strict separation ระหว่าง
  code กับ config และ litmus test ที่ใช้ตรวจสอบได้จริงว่าระบบทำถูกต้องหรือไม่
- รู้ **ความเป็นจริงเชิงปฏิบัติการของการ rotate secret** ทั้งกรณี API key รั่วธรรมดา และกรณีร้ายแรง
  กว่าคือ `RAILS_MASTER_KEY` เองรั่วไหล พร้อมทดสอบจริงว่า Rails behave อย่างไรเมื่อ key หาย/ผิด/ถูก
  แทนที่ในแต่ละ environment
- มี **security checklist ที่ตรวจสอบได้จริง** ก่อน deploy ทุกครั้ง ครอบคลุมเรื่อง logging, git
  hygiene, per-environment credentials, และ principle of least privilege

**ต่อไป (Part 075):** ตอนนี้แอปของเรามี config ที่ปลอดภัยและแยกตาม environment อย่างถูกต้องแล้ว
Part ถัดไปจะพาไปสู่ **CI/CD ด้วย GitHub Actions สำหรับ Rails** — สอนวิธีตั้ง workflow ที่รัน test
suite, RuboCop, และ security scan (Brakeman) อัตโนมัติทุกครั้งที่ push โค้ด ซึ่งจะได้ใช้ความรู้เรื่อง
**GitHub Actions secrets** โดยตรงจาก Part นี้ — เพราะ secrets ของ CI/CD pipeline เองก็เป็นอีกหนึ่ง
"ที่เก็บ config แยกจากโค้ด" ตามหลัก twelve-factor เช่นกัน เพียงแต่เป็นคนละชั้นกับที่แอป Rails ใช้ตอน
รันจริง — Part 075 จะแสดงให้เห็นว่าเรื่องนี้เชื่อมโยงกับ credentials ที่เรียนใน Part นี้อย่างไร
