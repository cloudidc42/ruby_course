# Part 042: Devise — ติดตั้ง, Custom, Confirmable, Lockable

> **Step ครอบคลุมใน Part นี้:** Step 411–420
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 041 มาก่อน โดยเฉพาะแนวคิด `has_secure_password`,
> session-based login/logout, และ `before_action` ใน controller)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem `devise` 5.0.x (ตัวอย่างทั้งหมดทดสอบจริงบน
> Ruby 3.3.6, Rails 8.1.4, Devise 5.0.4, ฐานข้อมูล SQLite3)

ใน Part 041 เราสร้างระบบ login/logout เองทั้งหมดด้วยมือ โดยใช้ `has_secure_password` ที่มากับ
Rails (bcrypt ใต้ฝาครอบ) ผสมกับ session-based authentication แบบ vanilla — ข้อดีคือเราเห็นและ
ควบคุมโค้ดทุกบรรทัด แต่ถ้าจะทำให้ระบบสมบูรณ์ระดับ production จริง (ยืนยันอีเมล, ล็อกบัญชีเมื่อ
ใส่รหัสผ่านผิดหลายครั้ง, ลืมรหัสผ่าน, "จำฉันไว้", ล็อกอินผ่าน Google/Facebook) เราต้องเขียนเองอีก
เยอะมาก — และแต่ละจุดล้วนเป็นจุดที่พลาดแล้วกลายเป็นช่องโหว่ความปลอดภัยได้ง่าย

**Devise** คือ gem authentication ที่ได้รับความนิยมสูงสุดในระบบนิเวศ Ruby on Rails มาตั้งแต่ปี
2009 แก้ปัญหานี้ด้วยการมอบฟีเจอร์เหล่านี้ให้แบบสำเร็จรูป ผ่านการเปิดใช้ "module" ทีละตัว
Part นี้จะพาไปติดตั้ง Devise ตั้งแต่ศูนย์ในแอปใหม่ ปรับแต่ง view/controller ให้เข้ากับความ
ต้องการเฉพาะของเรา และเจาะลึกสอง module ที่ทรงพลังที่สุดคือ `:confirmable` (ยืนยันอีเมล) กับ
`:lockable` (ล็อกบัญชีอัตโนมัติ) โดยสาธิตการทำงานจริงทุกขั้นตอน ไม่ใช่แค่ทฤษฎี

## สารบัญของ Part นี้

- Step 411: ทำไมทีมส่วนใหญ่เลือก Devise แทนการเขียนเอง — จุดแข็งและข้อเสียที่ต้องรู้ก่อนตัดสินใจ
- Step 412: ติดตั้ง Devise — `Gemfile`, `rails generate devise:install`, การตั้งค่าที่จำเป็น
- Step 413: Generate Devise model และอ่าน Migration — 10 module ของ Devise มีอะไรบ้าง
- Step 414: `authenticate_user!`, route helper อัตโนมัติ, `current_user`, `user_signed_in?`
- Step 415: Customize Views — `rails generate devise:views` เพราะ view เริ่มต้นไม่มีสไตล์
- Step 416: Customize Controller — สืบทอด `Devise::RegistrationsController` เพื่อ logic เฉพาะ
- Step 417: `:confirmable` เชิงลึก — บังคับยืนยันอีเมลก่อน login ได้ (สาธิตจริงทุกขั้นตอน)
- Step 418: `:lockable` เชิงลึก — ล็อกบัญชีอัตโนมัติเมื่อใส่รหัสผิดซ้ำ (จำลอง brute-force จริง)
- Step 419: Strong Parameters กับ Devise — `devise_parameter_sanitizer` สำหรับ field ที่เพิ่มเอง
- Step 420: แบบฝึกหัด — สร้างระบบสมัครสมาชิกแบบเต็มด้วย Devise ครบทุก module ที่เรียนมา

---

## Step 411: ทำไมทีมส่วนใหญ่เลือก Devise แทนการเขียนเอง — จุดแข็งและข้อเสียที่ต้องรู้ก่อนตัดสินใจ

### เทียบกับสองทางเลือกจาก Part 041

Part 041 สอนสองแนวทางที่เป็น **"เขียนเองแบบมองเห็นทุกบรรทัด"**:

1. `has_secure_password` + session-based login แบบมือเปล่า (เขียน `SessionsController` เอง)
2. Generator ของ Rails 8 เอง (`rails generate authentication`) ที่ scaffold โค้ดชุดเดียวกันให้
   แบบไฟล์จริงอยู่ใน `app/` ของเราทั้งหมด แก้ตรงไหนก็ได้ทันที

ทั้งสองแบบมีจุดร่วมกันคือ **โค้ดทุกบรรทัดอยู่ในแอปของเรา** — ไม่มี "มายากล" ซ่อนอยู่ในที่อื่น
เหมาะกับการเรียนรู้และเหมาะกับแอปที่ requirement การ auth เรียบง่าย

**Devise** เดินเกมตรงข้าม: มันคือ **gem** ที่เอา logic การ authenticate เกือบทั้งหมดไปเก็บไว้ใน
ตัว gem เอง (สร้างขึ้นบน [Warden](https://github.com/wardencommunity/warden) — Rack middleware
สำหรับ authentication) แอปของเราแค่ "เปิดใช้ module" ที่ต้องการผ่าน DSL สั้นๆ

### จุดแข็งของ Devise

1. **Battle-tested มานานกว่า 15 ปี** — ใช้งานจริงในโปรดักชันนับหมื่นแอปทั่วโลก บั๊กเกี่ยวกับ
   ความปลอดภัยพื้นฐาน (timing attack ตอนเทียบรหัสผ่าน, session fixation, CSRF บนฟอร์ม login)
   ถูกแก้ไปหมดแล้วตั้งแต่หลายปีก่อน — เราไม่ต้องมานั่งค้นพบเองแล้วแก้เอง
2. **ฟีเจอร์ครบตั้งแต่วันแรก ผ่านการเปิด module** — ไม่ต้องเขียนเองสักบรรทัด:

   | Module | สิ่งที่ได้มาฟรี |
   |---|---|
   | `:database_authenticatable` | เข้ารหัสรหัสผ่านด้วย bcrypt + validate ตอน login (แกนหลัก) |
   | `:registerable` | หน้าสมัครสมาชิก/แก้ไข/ลบบัญชีให้ครบ |
   | `:recoverable` | ลืมรหัสผ่าน → ส่งอีเมลลิงก์รีเซ็ต |
   | `:rememberable` | "จำฉันไว้" ด้วย cookie ที่ปลอดภัย |
   | `:trackable` | บันทึกจำนวนครั้ง/เวลา/IP ที่ login ล่าสุดและก่อนหน้า |
   | `:validatable` | validation มาตรฐานของ email/password (format, ความยาว, uniqueness) |
   | `:confirmable` | บังคับยืนยันอีเมลก่อนใช้งานได้ (Step 417) |
   | `:lockable` | ล็อกบัญชีอัตโนมัติเมื่อใส่รหัสผิดซ้ำหลายครั้ง (Step 418) |
   | `:timeoutable` | ตัด session อัตโนมัติเมื่อไม่มีการใช้งานนานเกินกำหนด |
   | `:omniauthable` | เชื่อมต่อ OAuth (Google/Facebook/GitHub) ผ่าน OmniAuth (Part 045) |

   ถ้าจะเขียนเองทั้ง 10 module นี้ตาม pattern ของ Part 041 จะใช้เวลาหลายวันถึงหลายสัปดาห์ และ
   ทดสอบให้ครอบคลุมทุก edge case (เช่น token รีเซ็ตรหัสผ่านหมดอายุ, ล็อกบัญชีแล้วปลดล็อกด้วย
   เวลา vs อีเมล) เอง
3. **ชุมชนใหญ่ หาโค้ดตัวอย่าง/แก้ปัญหาได้ง่าย** — คำถาม Devise แทบทุกแบบมีคนถามใน Stack
   Overflow มาก่อนแล้ว และ Rails developer ส่วนใหญ่ในตลาดงานคุ้นเคยกับ Devise อยู่แล้ว ลด
   learning curve เวลาทีมขยาย

### ข้อเสียที่ต้องยอมรับก่อนเลือกใช้

1. **"มายากล" มากกว่า — โค้ดที่ทำงานจริงไม่ได้อยู่ใน `app/` ของเรา** ตัวอย่างเช่น เมื่อ user
   กด submit ฟอร์ม login, logic การเช็ครหัสผ่าน, การสร้าง session, การนับ `failed_attempts`
   ทั้งหมดรันอยู่ใน source code ของ gem (`devise-5.0.4/app/controllers/devise/...` ในโฟลเดอร์
   `gems/`) ไม่ใช่ใน `app/controllers/` ของเรา การ debug เมื่อพฤติกรรมไม่ตรงตามคาดจึงต้องเปิด
   อ่าน source ของ gem เอง หรือพึ่งเอกสาร/community — ต่างจาก Part 041 ที่กด "go to definition"
   แล้วเจอโค้ดของตัวเองทันที
2. **Customize เชิงลึกทำได้ยากกว่า** เช่น ฟอร์มสมัครสมาชิกแบบหลายขั้นตอน (multi-step wizard),
   logic การ authenticate ที่ผูกกับ multi-tenancy ซับซ้อน (เช่น user คนเดียวกันมีหลาย role ใน
   หลายองค์กร), หรือ requirement เฉพาะทางธุรกิจที่ Devise ไม่ได้ออกแบบไว้แต่แรก — ต้อง override
   ให้ถูกจุดในสถาปัตยกรรมของ Warden/Devise ซึ่งมีเส้นทางการเรียกที่ซับซ้อนกว่าโค้ดที่เขียนเอง
   ล้วนๆ (เราจะฝึก override บางส่วนใน Step 416 แต่การ override ระดับลึกกว่านั้นต้องอ่าน
   [Devise wiki](https://github.com/heartcombo/devise/wiki) เพิ่มเติม)
3. **เพิ่ม dependency เข้าโปรเจกต์** — ติดตั้ง Devise แล้วจะพ่วง gem อื่นเข้ามาด้วยอัตโนมัติ
   (`warden`, `orm_adapter`, `responders`, `bcrypt`) ทำให้ dependency surface ของโปรเจกต์ใหญ่ขึ้น
   และผูกวงจรอัปเกรดของระบบ auth ทั้งระบบไว้กับ release cycle ของทีม Devise (แม้ในทางปฏิบัติ
   Devise ค่อนข้าง stable และอัปเดต breaking change ไม่บ่อย)
4. **View เริ่มต้นไม่มีสไตล์เลย** (Step 415) — ต้อง generate view ออกมาปรับเองเสมอ ไม่ต่างจาก
   ต้องเขียนเพิ่มอยู่ดี เพียงแต่เป็นการปรับแต่ง ไม่ใช่การเขียน logic ตั้งแต่ศูนย์

### กรอบการตัดสินใจ

| เกณฑ์ | Part 041 (เขียนเอง/generator ของ Rails) | Devise |
|---|---|---|
| ความเร็วในการมี auth ใช้งานได้ | เร็ว แต่ครอบคลุมแค่ login/logout พื้นฐาน | เร็วกว่าถ้านับรวมฟีเจอร์ครบ (confirm/lock/recover) |
| การมองเห็น/ควบคุมโค้ด 100% | ใช่ | ไม่ใช่ (โค้ดหลักอยู่ใน gem) |
| ฟีเจอร์ระดับ production ครบ | ต้องเขียนเพิ่มเองทุกตัว | มีให้ผ่านการเปิด module |
| เหมาะกับ requirement เฉพาะทาง/ซับซ้อนมาก | เหมาะกว่า (ควบคุมได้ทุกจุด) | ทำได้แต่มีเพดาน ต้องเข้าใจ internals |
| ทีมคุ้นเคยอยู่แล้ว (ตลาดงาน) | ขึ้นกับทีม | สูงมาก แทบเป็นมาตรฐานอุตสาหกรรม |

**สรุปเชิงปฏิบัติ:** แอป MVP เล็กๆ ที่ auth requirement เรียบง่ายและอยากควบคุมทุกบรรทัด → ใช้
แนวทาง Part 041 ต่อไปได้เลย ไม่ต้องเปลี่ยน แต่แอปที่ต้องการฟีเจอร์ auth ครบเครื่องเร็ว (โดย
เฉพาะ confirmable/lockable/recoverable) และทีมยอมรับการพึ่งพา gem ภายนอกได้ → Devise คือ
ตัวเลือกที่ทีมส่วนใหญ่ในอุตสาหกรรมเลือกใช้จริง

---

## Step 412: ติดตั้ง Devise — `Gemfile`, `rails generate devise:install`, การตั้งค่าที่จำเป็น

> **หมายเหตุสำคัญ:** Devise ต้องการเป็นเจ้าของ schema ของ `User` model เอง (คอลัมน์
> `encrypted_password`, `reset_password_token` ฯลฯ) การเอา Devise มาติดตั้งทับ `User` ที่มี
> `has_secure_password` อยู่แล้วจาก Part 041 จะชนกัน (คอลัมน์ `password_digest` vs
> `encrypted_password` เป็นคนละชื่อ คนละ convention) ในทางปฏิบัติเวลาทีมตัดสินใจย้ายจาก
> hand-rolled auth มาเป็น Devise จะต้องเขียน migration data ย้ายข้อมูลเองอย่างระมัดระวัง —
> เรื่องนั้นนอกขอบเขตของ Part นี้ ในที่นี้เราจะสร้างแอปใหม่ชื่อ `devise_demo` เพื่อโฟกัสที่ตัว
> Devise ล้วนๆ

### สร้างแอปใหม่และเพิ่ม gem

```bash
rails new devise_demo -d sqlite3
cd devise_demo
```

เพิ่ม Devise เข้า `Gemfile`:

```ruby
# Gemfile
gem "devise"
```

```bash
bundle install
```

ผลลัพธ์จริง (ตัดบางบรรทัดที่ไม่เกี่ยวออก):

```
Fetching devise 5.0.4
Installing orm_adapter 0.5.0
Installing responders 3.2.1
Installing warden 1.2.9
Installing devise 5.0.4
Bundle complete! 21 Gemfile dependencies, 118 gems now installed.
```

สังเกตว่า Devise ดึง gem เพิ่มมา 3 ตัว: `orm_adapter` (เชื่อม Devise กับ ORM ต่างๆ ได้ ไม่ผูกกับ
ActiveRecord ตายตัว), `warden` (Rack middleware ที่ Devise สร้างต่อยอด), `responders` (ช่วยจัดการ
response format หลาย action) — นี่คือ "dependency surface" ที่พูดถึงใน Step 411 ข้อ 3

> **เวอร์ชัน compatibility:** Devise 5.0+ รองรับ Rails 8.1 อย่างเป็นทางการ (บทเรียนนี้ทดสอบจริง
> กับ devise 5.0.4 บน Rails 8.1.4) ถ้าโปรเจกต์เก่าของคุณค้าง Devise เวอร์ชัน 4.x ไว้ ให้ตรวจสอบ
> [CHANGELOG ของ Devise](https://github.com/heartcombo/devise/blob/main/CHANGELOG.md) ก่อน
> อัปเกรดข้าม major version เสมอ เพราะมีการเปลี่ยน API ของ controller บางจุด

### รัน `devise:install`

```bash
bin/rails generate devise:install
```

ผลลัพธ์:

```
      create  config/initializers/devise.rb
      create  config/locales/devise.en.yml

===============================================================================

Depending on your application's configuration some manual setup may be required:

  1. Ensure you have defined default url options in your environments files...
       config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

  2. Ensure you have defined root_url to *something* in your config/routes.rb.
       root to: "home#index"

  3. Ensure you have flash messages in app/views/layouts/application.html.erb.
       <p class="notice"><%= notice %></p>
       <p class="alert"><%= alert %></p>

  4. You can copy Devise views (for customization) to your app by running:
       rails g devise:views

===============================================================================
```

Generator สร้างไฟล์ให้ 2 ไฟล์ และพิมพ์ checklist 4 ข้อที่ต้องทำเอง มาไล่ทีละข้อ

**ข้อ 1 — `default_url_options`:** Devise ต้องสร้าง URL แบบเต็ม (มี host) ในอีเมล เช่น ลิงก์
ยืนยันบัญชี เพราะอีเมลถูกเปิดนอก context ของ request ปกติที่ Rails รู้ host เองอัตโนมัติ — ข่าวดี
คือ **Rails 8 ตั้งค่านี้ให้อัตโนมัติแล้วใน `config/environments/development.rb`** ตอน
`rails new`:

```ruby
# config/environments/development.rb (Rails 8 สร้างให้อัตโนมัติ ไม่ต้องเพิ่มเอง)
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```

แต่ฝั่ง production ต้องเข้าไปแก้ให้เป็นโดเมนจริงเสมอ (Rails 8 ใส่ค่า placeholder ไว้ให้เตือน):

```ruby
# config/environments/production.rb
config.action_mailer.default_url_options = { host: "example.com" }  # แก้เป็นโดเมนจริงก่อน deploy
```

**ข้อ 2 — root route:** ต้องมี route `root` เสมอ เพราะหลาย action ของ Devise (เช่น หลัง
sign out, หลังสมัครสมาชิกสำเร็จ) จะ redirect กลับไปที่ root path เป็นค่า default

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "home#index"
end
```

**ข้อ 3 — flash messages ใน layout:** Devise ใช้ `flash[:notice]` และ `flash[:alert]` สื่อสาร
ผลลัพธ์กลับมา (เช่น "Signed in successfully.", "Invalid email or password.") ต้องมีจุดแสดงผลใน
layout ไม่งั้นข้อความจะถูกตั้งไว้ใน flash แต่ไม่มีที่ไหนแสดงให้ user เห็น

```erb
<%# app/views/layouts/application.html.erb %>
<body>
  <p class="notice"><%= notice %></p>
  <p class="alert"><%= alert %></p>

  <%= yield %>
</body>
```

**ข้อ 4 — customize views:** จะทำใน Step 415

### ภาพรวม `config/initializers/devise.rb`

ไฟล์นี้มีค่า config เกือบ 200 บรรทัด (ส่วนใหญ่ comment ไว้เป็นค่า default) จุดที่ควรรู้จักตั้งแต่
วันแรก:

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  # อีเมลผู้ส่งเวลา Devise ส่งอีเมล (ยืนยันบัญชี, รีเซ็ตรหัสผ่าน, ปลดล็อกบัญชี)
  config.mailer_sender = "please-change-me-at-config-initializers-devise@example.com"

  # ความยาวรหัสผ่านที่ยอมรับ (ค่า default)
  config.password_length = 6..128

  # regex ตรวจสอบรูปแบบอีเมลคร่าวๆ (ไม่ได้ตรวจว่ามีอยู่จริง แค่ตรวจ format)
  config.email_regexp = /\A[^@\s]+@[^@\s]+\z/
end
```

`config.mailer_sender` เป็นจุดที่ต้องแก้ก่อนขึ้น production เสมอ (ค่า default เป็นข้อความเตือน
ให้เปลี่ยน ไม่ใช่อีเมลจริง) — ส่วนอื่นๆ ที่เกี่ยวกับ `:confirmable` และ `:lockable` เราจะกลับมา
ดูละเอียดใน Step 417–418

---

## Step 413: Generate Devise Model และอ่าน Migration — 10 Module ของ Devise มีอะไรบ้าง

### รัน generator

```bash
bin/rails generate devise User
```

ผลลัพธ์:

```
      invoke  active_record
      create    db/migrate/20260926062403_devise_create_users.rb
      create    app/models/user.rb
       route  devise_for :users
```

Generator ทำ 3 อย่างให้อัตโนมัติ: สร้าง migration, สร้าง model, และเพิ่มบรรทัด
`devise_for :users` ใน `config/routes.rb` (Step 414 จะเจาะลึก route ที่บรรทัดนี้สร้างให้)

### อ่าน `app/models/user.rb`

```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable
end
```

ค่า default เปิดให้ 5 module (comment บอกอีก 5 module ที่เหลือให้เราไปเปิดเพิ่มเอง) เมธอด
`devise` ตรงนี้คือจุดเดียวที่ควบคุมว่า `User` model จะมีความสามารถอะไรบ้าง — ไม่มี DSL อื่นให้
จำเพิ่ม แค่เพิ่ม/ลบ symbol ในบรรทัดนี้

### อ่าน Migration แบบเต็ม

```ruby
# db/migrate/20260926062403_devise_create_users.rb
# (timestamp ของคุณจะไม่ตรงกับตัวอย่างนี้ เป็นเรื่องปกติ — ทบทวนได้ที่ Part 025 Step 243)
class DeviseCreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      # t.integer  :sign_in_count, default: 0, null: false
      # t.datetime :current_sign_in_at
      # t.datetime :last_sign_in_at
      # t.string   :current_sign_in_ip
      # t.string   :last_sign_in_ip

      ## Confirmable
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email # Only if using reconfirmable

      ## Lockable
      # t.integer  :failed_attempts, default: 0, null: false # Only if lock strategy is :failed_attempts
      # t.string   :unlock_token # Only if unlock strategy is :email or :both
      # t.datetime :locked_at

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end
end
```

นี่คือจุดที่ทำให้ Devise "อ่านง่ายกว่าที่คิด" — migration ไม่ได้ซ่อนอะไรไว้เลย มัน **generate
คอลัมน์ของทุก module ไว้ให้ล่วงหน้าเป็น comment** column กลุ่มไหน comment ทิ้งไว้ (Trackable,
Confirmable, Lockable) แปลว่า module นั้นยังไม่ถูกเปิดใช้ใน `User` model ตอนนี้ (ตรงกับ 5 module
default ใน Step ก่อนหน้า) การเปิด module เพิ่มทำแค่ **uncomment คอลัมน์กลุ่มนั้น** แล้วเพิ่ม
symbol ใน `devise :...` ของ model เท่านั้น ไม่มี migration ใหม่ที่ต้องเขียนเอง

### ตารางสรุปคอลัมน์ทั้ง 10 Module

| Module | คอลัมน์ที่เพิ่ม | เปิดใช้ default หรือไม่ |
|---|---|---|
| `database_authenticatable` | `email`, `encrypted_password` | เปิด |
| `registerable` | (ไม่มีคอลัมน์เฉพาะ — เพิ่ม controller/route สมัครสมาชิก) | เปิด |
| `recoverable` | `reset_password_token`, `reset_password_sent_at` | เปิด |
| `rememberable` | `remember_created_at` | เปิด |
| `validatable` | (ไม่มีคอลัมน์เฉพาะ — เพิ่ม validation) | เปิด |
| `trackable` | `sign_in_count`, `current_sign_in_at`, `last_sign_in_at`, `current_sign_in_ip`, `last_sign_in_ip` | ปิด (comment ไว้) |
| `confirmable` | `confirmation_token`, `confirmed_at`, `confirmation_sent_at`, `unconfirmed_email` | ปิด (comment ไว้) |
| `lockable` | `failed_attempts`, `unlock_token`, `locked_at` | ปิด (comment ไว้) |
| `timeoutable` | (ไม่มีคอลัมน์ — ใช้ session timestamp) | ปิด |
| `omniauthable` | (ไม่มีคอลัมน์ในตาราง `users` เอง — มักแยกตาราง `identities`/`authorizations`) | ปิด |

### เปิด `:confirmable`, `:lockable`, `:trackable` และเพิ่มคอลัมน์ของเราเอง

Part นี้จะสาธิต `:confirmable` และ `:lockable` แบบเต็มรูปแบบ จึงต้องเปิดทั้งคู่ (บวก
`:trackable` เพื่อโชว์สถิติการ login ประกอบ) แก้ `app/models/user.rb`:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable
end
```

แล้ว uncomment คอลัมน์ที่เกี่ยวใน migration พร้อมเพิ่ม `name` ที่เป็น custom field ของเราเอง
(จะใช้สาธิต strong parameters ใน Step 419):

```ruby
# db/migrate/20260926062403_devise_create_users.rb
class DeviseCreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string   :current_sign_in_ip
      t.string   :last_sign_in_ip

      ## Confirmable
      t.string   :confirmation_token
      t.datetime :confirmed_at
      t.datetime :confirmation_sent_at
      t.string   :unconfirmed_email # Only if using reconfirmable

      ## Lockable
      t.integer  :failed_attempts, default: 0, null: false # Only if lock strategy is :failed_attempts
      t.string   :unlock_token # Only if unlock strategy is :email or :both
      t.datetime :locked_at

      # Custom field เพิ่มเอง (นอกเหนือจากที่ Devise สร้างให้)
      t.string :name

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :confirmation_token,   unique: true
    add_index :users, :unlock_token,         unique: true
  end
end
```

```bash
bin/rails db:create db:migrate
```

```
== 20260926062403 DeviseCreateUsers: migrating ================================
-- create_table(:users)
-- add_index(:users, :email, {:unique=>true})
-- add_index(:users, :reset_password_token, {:unique=>true})
-- add_index(:users, :confirmation_token, {:unique=>true})
-- add_index(:users, :unlock_token, {:unique=>true})
== 20260926062403 DeviseCreateUsers: migrated (0.0126s) =======================
```

> **ข้อควรระวัง:** ถ้า migrate ไปแล้วค่อยมาแก้ไฟล์ migration ทีหลัง (เหมือนที่เราทำข้างบน)
> ใช้ได้เฉพาะตอนที่ยัง **ไม่เคย push ขึ้น branch ที่คนอื่น pull ไปแล้ว** เท่านั้น ถ้า migration
> ถูก merge เข้า main แล้ว ต้องสร้าง migration ใหม่ (`add_column` ฯลฯ) แทนการแก้ไฟล์เก่า —
> กฎเดียวกับที่สอนไว้ใน Part 025 Step 245

---

## Step 414: `authenticate_user!`, Route Helper อัตโนมัติ, `current_user`, `user_signed_in?`

### Route ที่ `devise_for :users` สร้างให้

```bash
bin/rails routes | grep -i user
```

ผลลัพธ์ (จัดคอลัมน์ใหม่ให้อ่านง่าย):

```
                  Prefix Verb   URI Pattern                     Controller#Action
       new_user_session GET    /users/sign_in                   devise/sessions#new
           user_session POST   /users/sign_in                   devise/sessions#create
   destroy_user_session DELETE /users/sign_out                  devise/sessions#destroy
      new_user_password GET    /users/password/new              devise/passwords#new
     edit_user_password GET    /users/password/edit             devise/passwords#edit
          user_password PATCH  /users/password                  devise/passwords#update
                        POST   /users/password                  devise/passwords#create
  new_user_registration GET    /users/sign_up                   devise/registrations#new
 edit_user_registration GET    /users/edit                      devise/registrations#edit
      user_registration PATCH  /users                           devise/registrations#update
                        DELETE /users                           devise/registrations#destroy
                        POST   /users                            devise/registrations#create
  new_user_confirmation GET    /users/confirmation/new          devise/confirmations#new
      user_confirmation GET    /users/confirmation              devise/confirmations#show
                        POST   /users/confirmation              devise/confirmations#create
        new_user_unlock GET    /users/unlock/new                devise/unlocks#new
            user_unlock GET    /users/unlock                    devise/unlocks#show
                        POST   /users/unlock                    devise/unlocks#create
```

Route เดียว (`devise_for :users`) สร้าง route helper และ controller ให้ครบ 18 บรรทัด — ครอบคลุม
sign in/out, สมัคร/แก้ไข/ลบบัญชี, ลืมรหัสผ่าน, ยืนยันอีเมล และปลดล็อกบัญชี ทั้งหมดชี้ไปที่
controller ในเนมสเปซ `devise/` ซึ่งเป็นโค้ดที่อยู่ใน gem (ตามที่อธิบายไว้ใน Step 411 ข้อเสียข้อ 1)

สังเกตว่าชื่อ helper อิงจากชื่อโมเดล (`devise_for :users` → `user_session_path`,
`new_user_registration_path` ฯลฯ) — ถ้ามีหลาย model ที่ใช้ Devise พร้อมกัน (เช่น `Admin` แยกจาก
`User`) แต่ละตัวจะได้ชุด route/helper ของตัวเองไม่ชนกัน

### ป้องกัน controller ด้วย `authenticate_user!`

Devise เพิ่ม helper method `authenticate_<model>!` ให้อัตโนมัติตามชื่อโมเดล (ในที่นี้คือ
`authenticate_user!`) ใช้เป็น `before_action` แบบเดียวกับที่ Part 023/041 สอนไว้ทุกประการ —
Devise ไม่ได้เปลี่ยนวิธีเขียน controller เลย แค่เสริม method ให้ใช้:

```ruby
# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  before_action :authenticate_user!

  def show
  end
end
```

ทดสอบเข้าหน้านี้โดยไม่ login:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/dashboard/show
# => 302  (redirect ไปหน้า sign_in อัตโนมัติ พร้อม flash "You need to sign in or sign up before continuing.")
```

### `current_user` และ `user_signed_in?` ในทุกที่ของแอป

Devise เพิ่ม helper สองตัวนี้ให้ใช้ได้ทั้งใน controller และ view โดยอัตโนมัติ (ไม่ต้อง
`helper_method` เองแบบที่ Part 041 สอน):

```erb
<%# app/views/home/index.html.erb %>
<% if user_signed_in? %>
  <p>สวัสดี, <%= current_user.name.presence || current_user.email %>!</p>
  <p><%= link_to "Edit profile", edit_user_registration_path %></p>
  <p><%= button_to "Sign out", destroy_user_session_path, method: :delete %></p>
<% else %>
  <p><%= link_to "Sign up", new_user_registration_path %></p>
  <p><%= link_to "Log in", new_user_session_path %></p>
<% end %>
```

```erb
<%# app/views/dashboard/show.html.erb %>
<h1>Dashboard (ต้อง login เท่านั้น)</h1>
<p>ยินดีต้อนรับ <%= current_user.email %></p>
<p>เข้าสู่ระบบล่าสุด: <%= current_user.current_sign_in_at %></p>
<p>เข้าสู่ระบบก่อนหน้า: <%= current_user.last_sign_in_at %></p>
<p>จำนวนครั้งที่เข้าสู่ระบบ: <%= current_user.sign_in_count %></p>
```

ผลลัพธ์จริงหลัง login (สังเกตว่า `:trackable` ที่เปิดไว้ใน Step 413 ทำงานให้อัตโนมัติ ไม่ต้อง
เขียน callback อัปเดตค่าพวกนี้เอง):

```
ยินดีต้อนรับ somying@example.com
เข้าสู่ระบบล่าสุด: 2026-09-26 06:27:51 UTC
เข้าสู่ระบบก่อนหน้า: 2026-09-26 06:27:51 UTC
จำนวนครั้งที่เข้าสู่ระบบ: 1
```

> **เทียบกับ Part 041:** โครงสร้าง `before_action :authenticate_user!` เหมือนกับ
> `before_action :require_login` ที่เขียนเองใน Part 041 เป๊ะ ต่างกันแค่ Devise generate
> method ชื่อนี้ให้อัตโนมัติตามชื่อโมเดล และ `current_user`/`user_signed_in?` มาจาก Warden
> session แทนที่จะอ่าน `session[:user_id]` ตรงๆ แบบที่เขียนเอง — แนวคิดเบื้องหลังเหมือนกันทุก
> ประการ เพียงแค่ Devise ทำให้อัตโนมัติ

---

## Step 415: Customize Views — `rails generate devise:views` เพราะ View เริ่มต้นไม่มีสไตล์

ลองเข้าหน้า `/users/sign_up` ตอนนี้จะเห็น HTML ดิบๆ ไม่มี CSS class อะไรเลย เพราะ view ของ
Devise ที่ compile มาจาก gem ถูกออกแบบให้ **ใช้งานได้ทันทีแต่ไม่สวย** เพื่อให้นักพัฒนาทุกคน
ปรับแต่งเป็นสไตล์ของตัวเองได้อย่างอิสระ

### Copy view ออกมาไว้ในแอป

```bash
bin/rails generate devise:views
```

ผลลัพธ์:

```
      invoke  Devise::Generators::SharedViewsGenerator
      create    app/views/devise/shared
      create    app/views/devise/shared/_error_messages.html.erb
      create    app/views/devise/shared/_links.html.erb
      invoke  form_for
      create    app/views/devise/confirmations
      create    app/views/devise/confirmations/new.html.erb
      create    app/views/devise/passwords
      create    app/views/devise/passwords/edit.html.erb
      create    app/views/devise/passwords/new.html.erb
      create    app/views/devise/registrations
      create    app/views/devise/registrations/edit.html.erb
      create    app/views/devise/registrations/new.html.erb
      create    app/views/devise/sessions
      create    app/views/devise/sessions/new.html.erb
      create    app/views/devise/unlocks
      create    app/views/devise/unlocks/new.html.erb
      invoke  erb
      create    app/views/devise/mailer
      create    app/views/devise/mailer/confirmation_instructions.html.erb
      create    app/views/devise/mailer/email_changed.html.erb
      create    app/views/devise/mailer/password_change.html.erb
      create    app/views/devise/mailer/reset_password_instructions.html.erb
      create    app/views/devise/mailer/unlock_instructions.html.erb
```

จากนี้ไปไฟล์เหล่านี้เป็นไฟล์จริงใน `app/views/devise/` ของเรา แก้ไขได้เหมือน view ทั่วไปทุก
ประการ (Devise จะให้ความสำคัญกับไฟล์ในแอปก่อน source ของ gem เสมอ — Rails view lookup path
มองหาใน `app/views/` ก่อนเสมอตามที่ Part 024 สอนไว้)

### เปิดดู `registrations/new.html.erb` ต้นฉบับ

```erb
<h2>Sign up</h2>

<%= form_for(resource, as: resource_name, url: registration_path(resource_name)) do |f| %>
  <%= render "devise/shared/error_messages", resource: resource %>

  <div class="field">
    <p><%= f.label :email %></p>
    <p><%= f.email_field :email, autofocus: true, autocomplete: "email" %></p>
  </div>

  <div class="field">
    <p><%= f.label :password %></p>
    <% if @minimum_password_length %>
      <p><em>(<%= @minimum_password_length %> characters minimum)</em></p>
    <% end %>
    <p><%= f.password_field :password, autocomplete: "new-password" %></p>
  </div>

  <div class="field">
    <p><%= f.label :password_confirmation %></p>
    <p><%= f.password_field :password_confirmation, autocomplete: "new-password" %></p>
  </div>

  <div class="actions">
    <%= f.submit "Sign up" %>
  </div>
<% end %>

<%= render "devise/shared/links" %>
```

จุดที่ควรสังเกต:

- ใช้ `form_for` ไม่ใช่ `form_with` ที่ Part 031 สอน — เป็น legacy helper ที่ยังทำงานได้ปกติใน
  Rails 8.1 (ภายในถูก implement บน `form_with` อยู่แล้ว) จะเปลี่ยนเป็น `form_with` เองก็ได้ ไม่มี
  ผลต่อการทำงาน แค่เป็นเรื่องความชอบส่วนตัว/มาตรฐานทีม
- `resource`, `resource_name` เป็นตัวแปรพิเศษที่ Devise เตรียมไว้ให้ทุก view — `resource` คือ
  instance ของโมเดล (เทียบเท่า `@user`), `resource_name` คือ symbol `:user` — ออกแบบมาให้ view
  ชุดเดียวใช้ได้กับหลายโมเดลที่ enable Devise (`User`, `Admin` ฯลฯ) โดยไม่ต้อง hardcode ชื่อ
- `render "devise/shared/error_messages"` — partial แสดง validation error แบบมาตรฐาน แก้ style
  ได้อิสระ

### เพิ่ม field ที่ custom (`name`) เข้าไปในฟอร์ม

เตรียมไว้สำหรับ Step 419 — เพิ่มช่อง `name` ในทั้งฟอร์มสมัครสมาชิกและฟอร์มแก้ไขโปรไฟล์:

```erb
<%# app/views/devise/registrations/new.html.erb — เพิ่มก่อน field email %>
<div class="field">
  <p><%= f.label :name %></p>
  <p><%= f.text_field :name, autofocus: true %></p>
</div>
```

```erb
<%# app/views/devise/registrations/edit.html.erb — เพิ่มก่อน field email เช่นกัน %>
<div class="field">
  <p><%= f.label :name %></p>
  <p><%= f.text_field :name, autofocus: true %></p>
</div>
```

> **ระวัง:** แค่เพิ่ม field ในฟอร์มยังไม่พอ — ถ้าลองสมัครตอนนี้ค่า `name` จะไม่ถูกบันทึกลง
> ฐานข้อมูล เพราะ Devise controller ยังไม่ได้ permit parameter ตัวนี้ (strong parameters
> ป้องกันไว้ตามหลักการของ Part 031) เราจะแก้ปัญหานี้ใน Step 419

### Generate view เฉพาะบาง scope (ทางเลือก)

ถ้าอยากได้แค่บางกลุ่ม view ไม่ต้อง copy ทั้งหมด:

```bash
# เฉพาะ view ของ registrations กับ sessions
bin/rails generate devise:views -v registrations sessions

# ถ้ามีหลายโมเดลที่ enable Devise เช่น User และ Admin แยก view กันคนละชุด
bin/rails generate devise:views users
```

---

## Step 416: Customize Controller — สืบทอด `Devise::RegistrationsController` เพื่อ Logic เฉพาะ

การแก้ view อย่างเดียวพอสำหรับเรื่องหน้าตา แต่ถ้าต้องการเปลี่ยน **พฤติกรรม** ของ action (เช่น
ทำอะไรเพิ่มหลังสมัครสมาชิกสำเร็จ, permit parameter เพิ่ม, เปลี่ยนหน้าที่ redirect ไปหลัง login)
ต้องสร้าง controller ของเราเองที่สืบทอดจาก controller ของ Devise แล้วบอก route ให้ใช้ตัวใหม่
แทนของ gem

### สร้าง Controller ที่สืบทอด

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]
  before_action :configure_account_update_params, only: [:update]

  protected

  # อนุญาตให้ parameter :name ผ่านเข้ามาตอนสมัครสมาชิก (รายละเอียดเต็มใน Step 419)
  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
  end

  # อนุญาตให้ parameter :name ผ่านเข้ามาตอนแก้ไขโปรไฟล์
  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, keys: [:name])
  end

  # ตัวอย่าง custom logic: หลังสมัครสมาชิกสำเร็จ ให้ log ข้อความเพิ่มเติม
  # (โปรเจกต์จริงอาจ enqueue job ส่งอีเมลต้อนรับ, สร้างข้อมูลเริ่มต้นให้ user ฯลฯ)
  def after_inactive_sign_up_path_for(resource)
    Rails.logger.info("[Users::RegistrationsController] สมัครสมาชิกใหม่: #{resource.email}")
    super
  end
end
```

**สิ่งสำคัญของโครงสร้างนี้:**

- Class ตั้งอยู่ใน namespace `Users::` (ไฟล์ต้องอยู่ที่ `app/controllers/users/`) — เป็น
  convention ที่ Devise แนะนำ เพื่อไม่ให้ชนกับ controller อื่นชื่อ `RegistrationsController`
  เฉยๆ ถ้ามีหลายโมเดลที่ enable Devise
- สืบทอดจาก `Devise::RegistrationsController` (มาจาก gem) ไม่ใช่ `ApplicationController` —
  ทำให้ action ทั้งหมด (`new`, `create`, `edit`, `update`, `destroy`, `cancel`) ที่ Devise
  implement ไว้ยังทำงานเหมือนเดิม เราแค่ override เฉพาะจุดที่ต้องการ (`protected` methods ที่
  Devise เปิดช่องให้ override ไว้ตั้งแต่ต้น) — รูปแบบเดียวกับการสืบทอด `ApplicationController`
  ที่ Part 023 สอน เพียงแค่จุดเริ่มต้นเป็น controller ของ gem แทน
- `configure_sign_up_params`/`configure_account_update_params` เป็นชื่อ method ที่ Devise
  **คาดหวัง** ให้เราสร้าง (เรียกจาก `before_action` ที่เราเพิ่มเอง ไม่ใช่ magic callback ของ
  gem) — ตรวจสอบชื่อ method ที่ override ได้ทั้งหมดจาก source ของ
  `Devise::RegistrationsController`

### บอก route ให้ใช้ controller ใหม่

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: {
    registrations: "users/registrations"
  }

  root "home#index"
end
```

ตรวจสอบด้วย `bin/rails routes | grep -i registration`:

```
                  Prefix Verb   URI Pattern           Controller#Action
cancel_user_registration GET    /users/cancel         users/registrations#cancel
   new_user_registration GET    /users/sign_up        users/registrations#new
  edit_user_registration GET    /users/edit           users/registrations#edit
       user_registration PATCH  /users                users/registrations#update
                         PUT    /users                users/registrations#update
                         DELETE /users                users/registrations#destroy
                         POST   /users                users/registrations#create
```

สังเกตคอลัมน์ขวาสุดเปลี่ยนจาก `devise/registrations#*` เป็น `users/registrations#*` — route
helper ชื่อเดิมทุกตัว (`new_user_registration_path` ฯลฯ) ยังใช้ได้เหมือนเดิม ไม่ต้องแก้ view ที่
เรียก helper พวกนี้เลยแม้แต่บรรทัดเดียว

Devise มี controller ให้ override ได้ทั้งหมด 7 ตัว ตาม option key ของ `controllers:` — `sessions`,
`registrations`, `passwords`, `confirmations`, `unlocks`, `omniauth_callbacks`,
`registrations` — ใช้ pattern เดียวกันทุกตัวคือ: สร้าง class สืบทอด, override method ที่ต้องการ,
แล้วชี้ route ไปที่ controller ใหม่

---

## Step 417: `:confirmable` เชิงลึก — บังคับยืนยันอีเมลก่อน Login ได้ (สาธิตจริงทุกขั้นตอน)

`:confirmable` คือ module ที่ทำให้ user **ต้องคลิกลิงก์ยืนยันในอีเมลก่อน** ถึงจะ login ได้ —
ฟีเจอร์มาตรฐานของทุกเว็บสมัยใหม่ที่ป้องกันการสมัครด้วยอีเมลปลอม/อีเมลคนอื่น

### Config ที่เกี่ยวข้องใน `devise.rb`

```ruby
# config/initializers/devise.rb

# ให้ user เข้าใช้งานได้กี่วันโดยยังไม่ยืนยันอีเมล ก่อนจะถูกบล็อก
# ค่า default คือ 0.days (บล็อกทันทีถ้ายังไม่ยืนยัน) — เราใช้ค่า default นี้
# config.allow_unconfirmed_access_for = 2.days

# ระยะเวลาที่ token ยืนยันยังใช้ได้ (default: nil = ไม่จำกัดเวลา)
# config.confirm_within = 3.days

# ถ้า true: การเปลี่ยนอีเมลต้องยืนยันอีเมลใหม่ก่อนถึงจะมีผล (ค่า default ของ Devise 5.x)
config.reconfirmable = true
```

เราใช้ค่า default ทั้งหมด (`allow_unconfirmed_access_for = 0.days` หมายความว่า **บล็อกทันที**
ที่สมัครเสร็จจนกว่าจะยืนยัน) ซึ่งเป็นพฤติกรรมที่เข้มงวดที่สุดและเหมาะกับแอปส่วนใหญ่

### ทดสอบ Flow เต็มด้วย `curl` (ผลลัพธ์จริงจากการรันทดสอบ)

**1) สมัครสมาชิก**

```bash
curl -s -c cookies.txt http://localhost:3000/users/sign_up -o form.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' form.html | sed 's/.*value="//;s/"$//')

curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[name]=สมหญิง รักเรียน" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[password]=password123" \
  --data-urlencode "user[password_confirmation]=password123"
```

ตรวจ flash message ที่ได้:

```
A message with a confirmation link has been sent to your email address.
Please follow the link to activate your account.
```

**2) ดู log ของ ActionMailer (development ส่งอีเมลลง log แทนการส่งจริง)**

```
Rendering devise/mailer/confirmation_instructions.html.erb
Delivered mail ...

<p>Welcome somying@example.com!</p>
<p>You can confirm your account email through the link below:</p>
<p><a href="http://localhost:3000/users/confirmation?confirmation_token=jEXCzz7u6Q3s8v4bWruk">
  Confirm my account
</a></p>
```

สังเกตว่า `confirmation_token` ถูกฝังใน URL — นี่คือค่าที่ถูก hash เก็บไว้ในคอลัมน์
`confirmation_token` ของแถวนั้นในตาราง `users`

**3) พยายาม login ก่อนยืนยัน → ถูกบล็อก**

```bash
curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users/sign_in \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[password]=password123"
```

flash ที่ได้กลับมา (ข้อความจริงจากการทดสอบ):

```
You have to confirm your email address before continuing.
```

Login ถูกปฏิเสธทันที **แม้ email/password จะถูกต้องทั้งคู่** — นี่คือแก่นของ `:confirmable`

**4) คลิกลิงก์ยืนยัน**

```bash
curl -s "http://localhost:3000/users/confirmation?confirmation_token=jEXCzz7u6Q3s8v4bWruk"
```

flash หลังยืนยันสำเร็จ:

```
Your email address has been successfully confirmed.
```

**5) Login อีกครั้งหลังยืนยัน → สำเร็จ**

```bash
curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users/sign_in \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[password]=password123"
```

```
Signed in successfully.
```

### เช็คสถานะยืนยันในโค้ด

`:confirmable` เพิ่ม method `confirmed?` ให้ instance ของ `User` ใช้ตรวจสอบได้ทุกที่:

```ruby
user = User.find_by(email: "somying@example.com")
user.confirmed?      # => true (หลังยืนยันแล้ว)
user.confirmed_at    # => 2026-09-26 06:26:22 UTC
```

มีประโยชน์เวลาต้องการกันบางฟีเจอร์ไว้เฉพาะ user ที่ยืนยันแล้วเท่านั้น (แม้ Devise จะบล็อกตอน
login ให้อยู่แล้ว แต่บาง flow เช่น API token อาจต้องเช็คซ้ำ):

```ruby
class PremiumFeatureController < ApplicationController
  before_action :authenticate_user!
  before_action :require_confirmed_email!

  private

  def require_confirmed_email!
    return if current_user.confirmed?

    redirect_to root_path, alert: "กรุณายืนยันอีเมลก่อนใช้งานฟีเจอร์นี้"
  end
end
```

---

## Step 418: `:lockable` เชิงลึก — ล็อกบัญชีอัตโนมัติเมื่อใส่รหัสผิดซ้ำ (จำลอง Brute-force จริง)

`:lockable` ป้องกันการเดารหัสผ่านแบบ brute-force โดยล็อกบัญชีชั่วคราวหลังใส่รหัสผิดติดต่อกัน
ครบจำนวนที่กำหนด

### Config ที่เกี่ยวข้อง

```ruby
# config/initializers/devise.rb

# กลยุทธ์การล็อก: :failed_attempts (ล็อกตามจำนวนครั้งที่ผิด) หรือ :none
config.lock_strategy = :failed_attempts

# กลยุทธ์การปลดล็อก: :email (ส่งลิงก์ปลดล็อกทางอีเมล), :time (ปลดล็อกอัตโนมัติเมื่อครบเวลา),
# :both (เปิดทั้งสองทาง — ค่า default ของ Devise), :none (ต้องปลดล็อกเองผ่าน admin)
config.unlock_strategy = :both

# จำนวนครั้งที่ใส่รหัสผิดก่อนถูกล็อก (ค่า default ของ Devise คือ 20 — เราลดเหลือ 3
# เฉพาะในบทเรียนนี้ เพื่อให้สาธิต/ทดสอบได้เร็วโดยไม่ต้องพิมพ์รหัสผิด 20 รอบ)
config.maximum_attempts = 3

# เวลาที่ต้องรอก่อนบัญชีปลดล็อกอัตโนมัติ ถ้า unlock_strategy รวม :time ไว้ด้วย
config.unlock_in = 1.hour
```

> **หมายเหตุ:** `lock_strategy = :failed_attempts`, `unlock_strategy = :both` และ
> `unlock_in = 1.hour` ทั้งสามค่านี้คือค่า **default ของ Devise อยู่แล้ว** เราเขียนไว้ชัดเจนใน
> initializer เพื่อให้เห็นตรงๆ ว่ากำลังใช้ค่าอะไรอยู่ — มีแค่ `maximum_attempts` เท่านั้นที่เรา
> เปลี่ยนจากค่า default (20) เป็น 3 เพื่อจุดประสงค์ในการสาธิตของบทเรียนนี้โดยเฉพาะ ในโปรเจกต์
> จริงค่า default 20 มักเหมาะสมกว่าเพราะ 3 ครั้งง่ายเกินไปที่ user จริงจะโดนล็อกโดยไม่ตั้งใจ
> (พิมพ์รหัสผ่านผิดมือแค่ 2-3 ครั้งก็เกิดขึ้นได้บ่อย)

### จำลอง Brute-force จริงด้วย `curl` (ผลลัพธ์จริงจากการทดสอบ)

ยิง login ด้วยรหัสผ่านผิดติดต่อกัน 3 ครั้ง (ครบ `maximum_attempts` ที่ตั้งไว้):

```bash
for i in 1 2 3; do
  curl -s -c cookies.txt http://localhost:3000/users/sign_in -o form.html
  TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' form.html | sed 's/.*value="//;s/"$//')
  curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users/sign_in \
    --data-urlencode "authenticity_token=$TOKEN" \
    --data-urlencode "user[email]=somying@example.com" \
    --data-urlencode "user[password]=WRONGPASSWORD"
done
```

flash message ที่ได้ในแต่ละรอบ (ข้อความจริงจากการทดสอบ — สังเกตว่า Devise **เตือนล่วงหน้า**
ก่อนล็อกจริง):

```
ครั้งที่ 1: Invalid email or password.
ครั้งที่ 2: You have one more attempt before your account is locked.
ครั้งที่ 3: Your account is locked.
```

`config.last_attempt_warning = true` (ค่า default) คือสิ่งที่ทำให้ข้อความรอบที่ 2 (เหลืออีก 1
ครั้ง) ปรากฏขึ้นมาแตกต่างจากรอบอื่น — เป็น UX ที่ดีเพราะเตือน user จริงก่อนจะโดนล็อกเต็มๆ

**พิสูจน์ว่ารหัสผ่าน "ถูก" ก็ยังเข้าไม่ได้ระหว่างถูกล็อก:**

```bash
curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users/sign_in \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[password]=password123"
```

```
Your account is locked.
```

แม้จะใส่รหัสผ่านที่ถูกต้อง 100% ก็ยังถูกปฏิเสธ — นี่คือหัวใจของ `:lockable` การล็อกผูกกับ
**บัญชี** ไม่ใช่ผูกกับความถูกต้องของรหัสผ่านที่ใส่ครั้งล่าสุด

**ตรวจสถานะในฐานข้อมูลโดยตรง:**

```ruby
u = User.find_by(email: "somying@example.com")
u.locked_at       # => 2026-09-26 06:27:59 UTC
u.failed_attempts # => 4  (นับต่อเนื่องแม้บัญชีถูกล็อกแล้ว การพยายาม login ด้วยรหัสถูกก็ยังไม่รีเซ็ต)
u.unlock_token    # => "f35085970992e372ec8cc9e843d186941c32998aed4e2cf41a01c3fee2c24995"
```

### ปลดล็อกด้วยลิงก์ในอีเมล (เพราะตั้ง `unlock_strategy = :both`)

ตอนบัญชีถูกล็อก Devise ส่งอีเมลปลดล็อกให้อัตโนมัติทันที (log จริงจากการทดสอบ):

```
Devise::Mailer#unlock_instructions: processed outbound mail in 2.6ms

<p>Hello somying@example.com!</p>
<p>Your account has been locked due to an excessive number of unsuccessful sign in attempts.</p>
<p><a href="http://localhost:3000/users/unlock?unlock_token=dFTnNEyZtzX5M8S_vNp8">
  Unlock my account
</a></p>
```

คลิกลิงก์ (หรือ curl):

```bash
curl -s "http://localhost:3000/users/unlock?unlock_token=dFTnNEyZtzX5M8S_vNp8"
```

ตรวจฐานข้อมูลอีกครั้งหลังปลดล็อก:

```ruby
u.reload
u.locked_at       # => nil
u.failed_attempts # => 0   (รีเซ็ตกลับเป็น 0 อัตโนมัติ)
```

Login ด้วยรหัสผ่านที่ถูกต้องอีกครั้ง — สำเร็จตามปกติ:

```
Signed in successfully.
```

### สองกลยุทธ์การปลดล็อกที่ต่างกัน

| `unlock_strategy` | พฤติกรรม | เหมาะกับ |
|---|---|---|
| `:email` | ต้องคลิกลิงก์ในอีเมลเท่านั้นถึงจะปลดล็อก (ไม่ปลดเองแม้รอนาน) | ต้องการควบคุมเข้มงวด ยืนยันว่าเจ้าของอีเมลจริงเป็นคนขอปลดล็อก |
| `:time` | รอครบ `unlock_in` (ตัวอย่างตั้งไว้ 1 ชั่วโมง) แล้วปลดล็อกอัตโนมัติ ไม่ต้องทำอะไร | ลด friction ให้ user ไม่ต้องเช็คอีเมล แลกกับช่องโหว่เล็กน้อยที่ผู้โจมตีแค่ต้องรอ |
| `:both` (ค่า default ของบทเรียนนี้) | ปลดล็อกได้ทั้งสองทาง — คลิกลิงก์ทันที หรือรอครบเวลาก็ได้ | สมดุลระหว่างความปลอดภัยกับ UX ส่วนใหญ่เลือกทางนี้ |

---

## Step 419: Strong Parameters กับ Devise — `devise_parameter_sanitizer` สำหรับ Field ที่เพิ่มเอง

### ปัญหา: ทำไม field ที่เพิ่มเองไม่ถูกบันทึก

Part 031 สอนไว้ว่า Rails บังคับ whitelist parameter ทุกตัวก่อน mass-assign เข้าโมเดล
(`params.require(:user).permit(:email, :password)`) — Devise controller ก็ทำแบบเดียวกัน
แต่เขียน whitelist ไว้ **ใน source code ของ gem** ซึ่งอนุญาตเฉพาะ field มาตรฐานของแต่ละ module
เท่านั้น (`email`, `password`, `password_confirmation`, `current_password` ฯลฯ)

field ที่เราเพิ่มเอง เช่น `name` จาก Step 413/415 **จะถูกกรองทิ้งเงียบๆ** ทันทีที่ผ่าน strong
parameters ของ Devise ถึงแม้ฟอร์มจะส่งค่ามาถูกต้อง และแม้คอลัมน์ `name` จะมีอยู่ในตารางแล้วก็ตาม
— ไม่มี error โผล่ขึ้นมาเตือนเลย ทำให้เป็นบั๊กที่ debug ยากถ้าไม่รู้จักกลไกนี้มาก่อน

### ทางแก้: `devise_parameter_sanitizer`

Devise เตรียม object ชื่อ `devise_parameter_sanitizer` ไว้ในทุก controller ของมัน ใช้เพิ่ม key
เข้าไปใน whitelist ของแต่ละ action ได้ (ทำไปแล้วบางส่วนตั้งแต่ Step 416 — มาดูรายละเอียดเต็ม
ตรงนี้):

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]
  before_action :configure_account_update_params, only: [:update]

  protected

  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
  end

  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, keys: [:name])
  end
end
```

**สังเกตว่ามี 2 action ที่ต้อง permit แยกกัน:**

- `:sign_up` — ใช้ตอน `create` (สมัครสมาชิกใหม่)
- `:account_update` — ใช้ตอน `update` (แก้ไขโปรไฟล์)

ลืม permit action ใดไปหนึ่งจะทำให้ field นั้นใช้ได้แค่ตอนสมัคร แต่แก้ไขทีหลังไม่ได้ (หรือกลับกัน)
— เป็นจุดพลาดที่พบบ่อยมาก

### พิสูจน์ว่าทำงานจริง — ทดสอบสมัครสมาชิกพร้อมชื่อ

```bash
curl -s -b cookies.txt -c cookies.txt -X POST http://localhost:3000/users \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[name]=สมหญิง รักเรียน" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[password]=password123" \
  --data-urlencode "user[password_confirmation]=password123"
```

ตรวจ SQL ที่รันจริง (จาก development log):

```sql
INSERT INTO "users" (
  "email", "encrypted_password", ..., "name", "created_at", "updated_at"
) VALUES (
  'somying@example.com', '$2a$12$...', ..., 'สมหญิง รักเรียน', ...
)
```

คอลัมน์ `name` ถูก INSERT พร้อมค่าจริงที่ส่งมาจากฟอร์ม — ยืนยันว่า sanitizer ทำงาน

### พิสูจน์การแก้ไขโปรไฟล์ (`account_update`)

```bash
curl -s -b cookies.txt http://localhost:3000/users/edit -o edit_form.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' edit_form.html | head -1 | sed 's/.*value="//;s/"$//')

curl -s -b cookies.txt -c cookies.txt -X PUT http://localhost:3000/users \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[name]=สมหญิง รักเรียน (แก้ไขแล้ว)" \
  --data-urlencode "user[email]=somying@example.com" \
  --data-urlencode "user[current_password]=password123"
```

> **สังเกต:** ฟอร์มแก้ไขโปรไฟล์ของ Devise บังคับให้กรอก `current_password` เสมอ (ดูใน
> `registrations/edit.html.erb` ที่ generate มา) เป็น security measure มาตรฐาน — ป้องกันไม่ให้
> ใครก็ตามที่ขโมย session cookie ไปแก้อีเมล/รหัสผ่านของ user โดยไม่รู้รหัสผ่านเดิม

ตรวจฐานข้อมูลหลัง update:

```ruby
User.find_by(email: "somying@example.com").name
# => "สมหญิง รักเรียน (แก้ไขแล้ว)"
```

ยืนยันว่า `name` ถูกอัปเดตสำเร็จผ่านทั้งสอง action

### เทียบกับ Strong Parameters แบบมือเขียนใน Part 031

| | Part 031 (Controller ของเราเอง) | Devise |
|---|---|---|
| จุดที่เขียน whitelist | `params.require(:post).permit(:title, :body)` ใน action ตรงๆ | เขียนใน `configure_sign_up_params`/`configure_account_update_params` แยกจาก action |
| field มาตรฐาน (email, password) | ต้อง permit เองทั้งหมด | Devise permit ให้อัตโนมัติแล้ว (มาจาก module ที่เปิดใช้) |
| field ที่เพิ่มเอง | permit ตรงจุดเดียวที่ action ใช้งาน | ต้องรู้จัก `devise_parameter_sanitizer` และ permit แยกตาม action key ของ Devise |

หลักการเบื้องหลังเหมือนกันทุกประการ (whitelist parameter ก่อน mass-assign) — Devise แค่ย้ายจุด
ที่เขียน whitelist ไปอยู่ใน method ที่มันกำหนดชื่อไว้ให้เท่านั้น

---

## Step 420: แบบฝึกหัด — สร้างระบบสมัครสมาชิกแบบเต็มด้วย Devise ครบทุก Module ที่เรียนมา

### โจทย์

สร้างแอป Rails ใหม่ (หรือใช้แอป `devise_demo` จาก Part นี้ต่อได้เลย) แล้วทำให้ครบตามนี้:

1. ติดตั้ง Devise และ generate `User` model ที่เปิดใช้ module: `:database_authenticatable`,
   `:registerable`, `:recoverable`, `:confirmable`, `:lockable` (ไม่ต้องมี `:rememberable`,
   `:trackable`, `:validatable` ก็ได้ในโจทย์นี้ — ฝึกเลือกเฉพาะ module ที่ต้องใช้จริง)
2. เพิ่ม custom field `:name` (string, บังคับกรอก — เพิ่ม validation `presence: true` ใน
   model เอง เพราะ Devise ไม่ validate field ที่เราเพิ่มเองให้)
3. Generate และปรับแต่ง view ของ `registrations` ให้มีช่อง `name`
4. สร้าง `Users::RegistrationsController` ที่ permit `:name` ผ่าน
   `devise_parameter_sanitizer` ทั้งตอนสมัครและตอนแก้ไขโปรไฟล์
5. ทดสอบ flow เต็ม end-to-end: สมัครสมาชิก → ยืนยันไม่ได้ทันที (ต้องเช็คอีเมลก่อน) → ยืนยัน
   สำเร็จ → login สำเร็จ → ใส่รหัสผ่านผิด 3 ครั้งติด (ตั้ง `maximum_attempts = 3`) → บัญชีถูกล็อก
   → ปลดล็อกด้วยลิงก์ในอีเมล → login สำเร็จอีกครั้ง

### เฉลย

**1) Gemfile และติดตั้ง**

```ruby
# Gemfile
gem "devise"
```

```bash
bundle install
bin/rails generate devise:install
```

ทำ 3 ข้อ setup ที่จำเป็น (Step 412):

```ruby
# config/routes.rb
root "home#index"
```

```erb
<%# app/views/layouts/application.html.erb %>
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

(`default_url_options` ของ dev environment Rails 8 ตั้งให้อัตโนมัติแล้ว ไม่ต้องแก้)

**2) Generate model และแก้ module ที่เปิดใช้**

```bash
bin/rails generate devise User
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :confirmable, :lockable

  validates :name, presence: true
end
```

**3) แก้ Migration ให้มีเฉพาะคอลัมน์ของ module ที่เปิดใช้ + `name`**

```ruby
# db/migrate/..._devise_create_users.rb
class DeviseCreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Confirmable
      t.string   :confirmation_token
      t.datetime :confirmed_at
      t.datetime :confirmation_sent_at
      t.string   :unconfirmed_email

      ## Lockable
      t.integer  :failed_attempts, default: 0, null: false
      t.string   :unlock_token
      t.datetime :locked_at

      # Custom field
      t.string :name, null: false

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :confirmation_token,   unique: true
    add_index :users, :unlock_token,         unique: true
  end
end
```

```bash
bin/rails db:create db:migrate
```

**4) ตั้งค่า `maximum_attempts` สำหรับทดสอบ**

```ruby
# config/initializers/devise.rb
config.lock_strategy = :failed_attempts
config.unlock_strategy = :both
config.maximum_attempts = 3
config.unlock_in = 1.hour
```

**5) Generate view และเพิ่มช่อง `name`**

```bash
bin/rails generate devise:views
```

```erb
<%# app/views/devise/registrations/new.html.erb — เพิ่มก่อน field email %>
<div class="field">
  <p><%= f.label :name %></p>
  <p><%= f.text_field :name, autofocus: true %></p>
</div>
```

```erb
<%# app/views/devise/registrations/edit.html.erb — เพิ่มก่อน field email %>
<div class="field">
  <p><%= f.label :name %></p>
  <p><%= f.text_field :name %></p>
</div>
```

**6) Custom controller + route**

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]
  before_action :configure_account_update_params, only: [:update]

  protected

  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
  end

  def configure_account_update_params
    devise_parameter_sanitizer.permit(:account_update, keys: [:name])
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: { registrations: "users/registrations" }
  root "home#index"
end
```

**7) ทดสอบ flow เต็ม (ผลลัพธ์จริงจากการทดสอบ)**

```bash
# สมัครสมาชิก
curl -s -c c.txt http://localhost:3000/users/sign_up -o f.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' f.html | sed 's/.*value="//;s/"$//')
curl -s -b c.txt -c c.txt -X POST http://localhost:3000/users \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[name]=ทดสอบ ระบบ" \
  --data-urlencode "user[email]=test@example.com" \
  --data-urlencode "user[password]=password123" \
  --data-urlencode "user[password_confirmation]=password123"
# => flash: "A message with a confirmation link has been sent..."

# พยายาม login ก่อนยืนยัน
# => flash: "You have to confirm your email address before continuing."

# ยืนยันด้วย token จาก log อีเมล
curl -s "http://localhost:3000/users/confirmation?confirmation_token=<TOKEN_FROM_EMAIL>"
# => flash: "Your email address has been successfully confirmed."

# login สำเร็จ
# => flash: "Signed in successfully."

# ใส่รหัสผิด 3 ครั้งติด
# ครั้งที่ 1 => "Invalid email or password."
# ครั้งที่ 2 => "You have one more attempt before your account is locked."
# ครั้งที่ 3 => "Your account is locked."

# ปลดล็อกด้วย token จาก log อีเมล unlock
curl -s "http://localhost:3000/users/unlock?unlock_token=<UNLOCK_TOKEN_FROM_EMAIL>"

# login สำเร็จอีกครั้ง
# => flash: "Signed in successfully."
```

ผลลัพธ์ครบทุกขั้นตอนตรงตามที่ระบุในโจทย์ ยืนยันว่าโมดูล `:confirmable`, `:lockable`,
custom field ผ่าน `devise_parameter_sanitizer`, และ custom view/controller ทำงานร่วมกันได้
ถูกต้องทั้งระบบ

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. เปิดใช้ module `:timeoutable` เพิ่มในโจทย์ข้างต้น ตั้ง `config.timeout_in = 10.seconds`
   (ตั้งสั้นมากเพื่อทดสอบ) แล้วพิสูจน์ด้วย `curl`/`rails console` ว่า session หมดอายุจริงหลัง
   ไม่มีการใช้งานเกิน 10 วินาที (ใบ้: login แล้วรอ, แล้วลองเข้าหน้าที่มี
   `before_action :authenticate_user!` ดูว่าถูก redirect กลับไป sign_in หรือไม่)
2. เปลี่ยน `unlock_strategy` เป็น `:time` อย่างเดียว (ไม่มี `:email`) พร้อมตั้ง
   `unlock_in = 30.seconds` เพื่อทดสอบเร็ว แล้วพิสูจน์ว่าบัญชีปลดล็อกอัตโนมัติได้โดยไม่ต้องมี
   อีเมลปลดล็อกส่งมาเลย (สังเกตว่า mailer log จะไม่มี "Unlock Instructions" ปรากฏขึ้นอีกต่อไป)
3. เพิ่มคอลัมน์ `role` (string, default "member") ลงในตาราง `users` แล้วเขียน
   `before_action` ใหม่ชื่อ `require_admin!` ที่อนุญาตเฉพาะ user ที่ `role == "admin"` เข้าหน้า
   `/admin/dashboard` ได้ (ยังไม่ต้องใช้ gem authorization ใดๆ เขียนเช็คตรงๆ ใน controller
   ไปก่อน) — แบบฝึกหัดนี้เป็นสะพานเชื่อมไปสู่ Part 043 ที่จะสอนวิธีจัดการ authorization แบบมี
   โครงสร้างและทดสอบง่ายกว่าการเขียน `if` กระจายอยู่ทั่ว controller

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **เมื่อไหร่ควรเลือก Devise แทนการเขียนเอง** — Devise ให้ฟีเจอร์ auth ระดับ production
  (confirmable, lockable, recoverable, trackable, timeoutable, omniauthable) แบบสำเร็จรูปผ่าน
  การเปิด module แลกกับการมี "มายากล" ที่ต้องเข้าใจ internals มากขึ้น, customize เชิงลึกยากขึ้น,
  และเพิ่ม dependency เข้าโปรเจกต์
- **ติดตั้ง Devise ครบวงจร** — `Gemfile` → `rails generate devise:install` → checklist 4 ข้อ
  (`default_url_options`, root route, flash message, views) → `rails generate devise <Model>`
- **อ่าน migration ของ Devise ได้** — คอลัมน์ของทุก module ถูก generate มาเป็น comment
  ล่วงหน้าทั้งหมด การเปิด module = uncomment คอลัมน์ + เพิ่ม symbol ใน `devise :...`
- **`authenticate_<model>!`, `current_user`, `user_signed_in?`** ทำงานแบบเดียวกับที่เขียนเองใน
  Part 041 ทุกประการ เพียงแค่ Devise generate ให้อัตโนมัติตามชื่อโมเดล
- **Customize view** ด้วย `rails generate devise:views` (เพราะ view เริ่มต้นไม่มีสไตล์เลย) และ
  **customize controller** ด้วยการสืบทอด `Devise::RegistrationsController` แล้วชี้ route ผ่าน
  `devise_for :users, controllers: { registrations: "..." }`
- **`:confirmable`** บล็อกการ login จนกว่าจะคลิกลิงก์ยืนยันในอีเมล — สาธิตจริงทุกขั้นตอนตั้งแต่
  สมัคร, ถูกบล็อก, ยืนยัน, จนถึง login สำเร็จ
- **`:lockable`** ล็อกบัญชีอัตโนมัติหลังใส่รหัสผิดครบ `maximum_attempts` ครั้ง พร้อมปลดล็อกได้
  ทั้งทาง `:email` และ `:time` (หรือ `:both`) — จำลอง brute-force จริงและพิสูจน์ว่าแม้รหัสผ่าน
  ถูกต้องก็ยังเข้าไม่ได้ระหว่างถูกล็อก
- **`devise_parameter_sanitizer`** คือกลไก strong parameters เฉพาะของ Devise สำหรับ field ที่
  เพิ่มเอง (เช่น `:name`) ต้อง permit แยกทั้ง `:sign_up` และ `:account_update` ไม่งั้น field จะ
  ถูกกรองทิ้งเงียบๆ โดยไม่มี error เตือน

**ต่อไป (Part 043):** ตอนนี้เรามีระบบ **authentication** (ยืนยันตัวตนว่า "คุณคือใคร") ที่แข็งแรง
แล้วทั้งจาก Part 041 และ Devise ใน Part นี้ แต่ยังไม่มีระบบ **authorization** (ตัดสินว่า "คุณทำ
สิ่งนี้ได้หรือไม่") ที่เป็นระบบ — แบบฝึกหัดข้อ 3 ข้างต้นที่เขียน `if current_user.role == "admin"`
ตรงๆ ใน controller คือจุดเริ่มต้นของปัญหาที่จะบานปลายเมื่อกฎการอนุญาตซับซ้อนขึ้น Part 043 จะสอน
**Pundit** เชิงลึก — gem authorization ที่ได้รับความนิยมสูงสุดคู่กับ Devise ผ่านแนวคิด `Policy`
(กฎการอนุญาตแยกเป็น class ของตัวเอง ทดสอบได้อิสระ) และ `Scope` (กรอง collection ตามสิทธิ์ของ
user แต่ละคน)
