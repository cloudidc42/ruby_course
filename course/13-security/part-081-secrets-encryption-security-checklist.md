# Part 081: Secrets Management, ActiveRecord Encryption และ Security Checklist ก่อนขึ้น Production — ปิด Phase 13: Security

> **Step ครอบคลุมใน Part นี้:** Step 801–810
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 074 เรื่อง environment variable/Rails credentials มาก่อน — Part
> นี้จะ **recap สั้นๆ** เฉพาะกลไกพื้นฐานของ credentials ไม่สอนซ้ำตั้งแต่ต้น และต้องผ่าน Part 079 เรื่อง
> OWASP Top 10 ในบริบท Rails (SQL Injection, XSS, CSRF) กับ Part 080 เรื่อง mass assignment,
> secure headers, และ Brakeman scan มาก่อนเช่นกัน เพราะ Part นี้จะอ้างอิงหัวข้อเหล่านั้นตอนสรุป
> checklist รวม โดยไม่อธิบายซ้ำว่าแต่ละอย่างคืออะไร)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4, ฐานข้อมูล
> SQLite ที่มากับ Rails 8 default — **ActiveRecord Encryption ทำงานได้กับทุก adapter ที่ Active
> Record รองรับ (SQLite, PostgreSQL, MySQL) โดยไม่ต้องพึ่งบริการภายนอกใดๆ เลย** เป็น feature ที่มากับ
> Rails framework เองตั้งแต่ Rails 7)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม:**
>
> ทุกกลไกใน Part นี้ — `db:encryption:init`, `encrypts`, deterministic query, การ backfill คอลัมน์เก่า,
> และการหมุนเวียนกุญแจ (key rotation) — ทีมผู้เขียนสร้างแอป Rails 8.1.4 จริงขึ้นมาในเครื่อง (แยกจาก
> repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้งหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) แล้วรันคำสั่งทุกคำสั่งใน
> Part นี้จริง รวมถึงกรณีที่ตั้งใจทำให้พัง (ถอดกุญแจเข้ารหัสออกแล้วดูว่า error อย่างไร) ผลลัพธ์ที่แสดง
> เป็น code block คือค่าที่ได้จากการรันจริงทั้งหมด ส่วนกุญแจเข้ารหัส (`primary_key`,
> `deterministic_key`, `key_derivation_salt`) ที่ปรากฏเป็นค่าที่ Rails สุ่มสร้างขึ้นเองตอนทดสอบ
> — **ไม่ใช่ค่าที่ยังใช้งานจริงที่ไหน และไม่ตรงกับรูปแบบ key ของผู้ให้บริการรายใดทั้งสิ้น** (เป็นเพียง
> ตัวเลข/ตัวอักษรสุ่มที่ `SecureRandom` สร้างในเครื่องทดสอบเท่านั้น) เช่นเดียวกับ Part 074 ที่ผ่านมา

ยินดีต้อนรับสู่ Part สุดท้ายของ **Phase 13: Security** — Part 079 พาไปรู้จัก OWASP Top 10 ในบริบทของ
Rails (SQL Injection, XSS, CSRF) และ Part 080 เจาะลึกเรื่อง mass assignment, secure headers, และการใช้
Brakeman สแกนหาช่องโหว่ในโค้ดอัตโนมัติ ทั้งสอง Part ที่ผ่านมาล้วนตอบคำถามว่า **"จะป้องกันไม่ให้คนนอก
โจมตีแอปของเราผ่าน HTTP request ได้อย่างไร"** แต่ Part นี้จะตอบคำถามที่ต่างออกไปโดยสิ้นเชิง — **"ถ้ามี
คนเข้าถึงฐานข้อมูลของเราได้โดยตรง (ไม่ผ่านแอปเลย) เขาจะเห็นอะไรบ้าง และเราจะป้องกันข้อมูลที่ละเอียด
อ่อนที่สุดได้อย่างไรแม้ในสถานการณ์นั้น"** นี่คือปัญหาคนละชั้นกับที่ Part 074 (credentials/ENV var)
และ Part 079–080 (การป้องกัน request ที่เข้ามา) เคยแก้ — และเป็นปัญหาที่ Rails มีเครื่องมือมาตรฐานให้
ใช้ได้ทันทีตั้งแต่ Rails 7 ชื่อว่า **Active Record Encryption**

Part นี้จะปิดท้ายด้วยสิ่งที่สำคัญที่สุดของ Phase นี้ทั้งหมด — **Security Checklist ก่อนขึ้น
production** ที่รวบรวมทุกหัวข้อที่เรียนมาตลอดทั้งหลักสูตร (ไม่ใช่แค่ Phase 13) ให้เป็นรายการตรวจสอบ
เดียวที่ใช้ได้จริงก่อน deploy แอปใดๆ สู่ผู้ใช้งานจริง

## สารบัญของ Part นี้

- Step 801: ทบทวน Secrets ที่ "นิ่ง" ใน Config (Part 074) เทียบกับปัญหาใหม่ — ข้อมูลที่ต้องเข้ารหัส
  ใน Database
- Step 802: Active Record Encryption คืออะไร และทำไมมี Encrypted Disk/Managed Database ไม่พอ
- Step 803: ติดตั้งและเปิดใช้งานจริง — `bin/rails db:encryption:init`, `encrypts :field_name`,
  พิสูจน์ว่า raw DB อ่านไม่ออกแต่ Active Record ถอดรหัสให้อัตโนมัติ
- Step 804: Deterministic vs Non-Deterministic Encryption — เมื่อไหร่ต้องแลกความปลอดภัยเพื่อ query ได้
- Step 805: เข้ารหัสคอลัมน์ที่มีข้อมูลเดิมอยู่แล้ว — Backfill แบบปลอดภัยสำหรับ Migration จริง
- Step 806: Key Rotation สำหรับ Active Record Encryption — หมุนเวียนกุญแจโดยข้อมูลเก่าไม่เสียหาย
- Step 807: ขอบเขตของ Active Record Encryption — สิ่งที่มันไม่ได้ป้องกัน
- Step 808: Security Checklist ก่อนขึ้น Production — รวบรวมทุกหัวข้อจากทั้งหลักสูตร
- Step 809: กระบวนการตรวจสอบรอบสุดท้ายก่อน Launch — Second Pair of Eyes และ Pentest ภายนอก
- Step 810: แบบฝึกหัดปิด Phase — เข้ารหัสฟิลด์ละเอียดอ่อนจริง พิสูจน์ราก DB อ่านไม่ออก แล้วไล่
  checklist เต็มรูปแบบ + ปิด Phase 13

---

## Step 801: ทบทวน Secrets ที่ "นิ่ง" ใน Config (Part 074) เทียบกับปัญหาใหม่ — ข้อมูลที่ต้องเข้ารหัส
ใน Database

### Recap สั้นๆ: สิ่งที่ Part 074 แก้ไปแล้ว (ไม่สอนซ้ำในที่นี้)

Part 074 สอนเรื่อง **secrets ของแอปเอง** — ค่าที่แอปต้องใช้เชื่อมต่อกับโลกภายนอก เช่น Stripe secret
key, database password, secret key base โดยแบ่งเป็นสองที่เก็บหลักตามว่าใครเป็นเจ้าของวงจรชีวิตของค่า
นั้น:

| ที่เก็บ | ใช้กับอะไร | วิธีอ่านค่า |
|---|---|---|
| `ENV["VAR_NAME"]` | ค่าที่ infrastructure/platform กำหนด (`PORT`, `DATABASE_URL`) | `ENV.fetch("KEY", default)` |
| Rails Encrypted Credentials | ค่าที่มนุษย์ในทีมสมัคร/ตั้งเอง (Stripe key, API key) | `Rails.application.credentials.dig(:key, :nested)` |

ทั้งสองที่เก็บนี้ตอบโจทย์ **"config ที่แยกจากโค้ด"** ตามหลัก twelve-factor app — secret ทุกตัวไม่ถูก
hardcode ในซอร์สโค้ด และไม่หลุดเข้า git history (credentials เข้ารหัสไว้, `.env` ถูก gitignore) นี่คือ
สิ่งที่ Part 074 แก้ไปเรียบร้อยแล้ว **secret เหล่านี้ "นิ่ง" อยู่ในไฟล์ config ตอน deploy** ไม่ได้เกี่ยว
กับข้อมูลที่ผู้ใช้กรอกเข้ามาในแอปแต่อย่างใด

### คำถามใหม่ที่ Part 074 ไม่ได้ตอบเลย

ลองนึกภาพแอปที่ผ่าน security review ของ Part 074 มาแล้วทุกข้อ — ไม่มี secret หลุดเข้า git, production
credentials แยกจาก dev, `.env` ถูก gitignore ครบถ้วน แล้วลองถามคำถามนี้:

> **ถ้า DBA คนหนึ่ง (หรือแฮกเกอร์ที่ขโมย database backup ไปได้ หรือช่องโหว่ SQL Injection ที่หลุดรอด
> Brakeman มาได้) เปิดตาราง `users` ขึ้นมาดูตรงๆ ด้วย SQL query ธรรมดา เขาจะเห็นเลขบัตรประชาชน,
> ประวัติการรักษาโรค, หรือบันทึกส่วนตัวของผู้ใช้เป็นตัวหนังสือธรรมดาที่อ่านออกทันทีหรือไม่?**

คำตอบสำหรับแอป Rails ทั่วไปที่ยังไม่เข้ารหัสข้อมูลระดับ column คือ **"เห็นชัดเจน อ่านออกทันที"**
เพราะ `password_digest` เป็นค่าเดียวที่ Rails บังคับให้ hash ไว้แต่แรก (ผ่าน `has_secure_password`)
ส่วนคอลัมน์อื่นทั้งหมด — เลขบัตรประชาชน, เบอร์โทรศัพท์, ที่อยู่, ประวัติการรักษา, บันทึกส่วนตัว —
ถูกเก็บเป็น **plaintext ธรรมดา** ในฐานข้อมูลเสมอ เว้นแต่จะเข้ารหัสเองอย่างจงใจ

นี่คือ **ปัญหาคนละชั้นกับ Part 074 โดยสิ้นเชิง**:

| | Part 074: Secrets/Config | Part 081 (Part นี้): Encrypted Data |
|---|---|---|
| ป้องกันอะไร | ค่าที่แอป**ใช้เชื่อมต่อ**กับบริการภายนอก | ข้อมูล**ของผู้ใช้**ที่เก็บอยู่ในตาราง |
| ที่เก็บ | ไฟล์ config (`credentials.yml.enc`, `.env`) | คอลัมน์ในตาราง database (`users`, `patients`, ฯลฯ) |
| ภัยที่ป้องกัน | secret หลุดเข้า git, คนใน repo เห็น production key | คนที่เข้าถึง**ฐานข้อมูลโดยตรง**เห็นข้อมูลผู้ใช้ |
| เครื่องมือ | Rails Encrypted Credentials, ENV var | **Active Record Encryption** (หัวข้อของ Part นี้) |

จุดที่ต้องเน้นให้ชัด: **credentials ที่ Part 074 สอนไม่ได้ช่วยอะไรกับปัญหานี้เลย** ต่อให้ Stripe
secret key ถูกเก็บอย่างสมบูรณ์แบบที่สุดในโลก ตาราง `patients` ที่มีคอลัมน์ `medical_notes` เป็น
plaintext ก็ยังเสี่ยงเท่าเดิม — ทั้งสองเรื่องต้องแก้ **แยกกัน** ด้วยเครื่องมือคนละตัว

---

## Step 802: Active Record Encryption คืออะไร และทำไมมี Encrypted Disk/Managed Database ไม่พอ

### ข้อโต้แย้งที่พบบ่อย: "ผมใช้ managed database ที่เข้ารหัส disk ให้อยู่แล้ว"

ผู้ให้บริการ database ระดับองค์กรแทบทุกเจ้า (managed PostgreSQL ของ cloud provider ต่างๆ) เปิด
**encryption at rest** ให้เป็นค่า default อยู่แล้ว — ข้อมูลบน disk ถูกเข้ารหัสจริง แต่การเข้ารหัส
ระดับนี้ป้องกัน**เฉพาะ**สถานการณ์ที่คนร้ายขโมย **physical disk** หรือ disk snapshot ไปโดยตรง มันไม่
ป้องกันสถานการณ์ที่พบบ่อยกว่ามากในโลกจริง:

1. **ใครก็ตามที่ query ฐานข้อมูลผ่าน connection ปกติได้** (DBA, นักพัฒนาคนอื่นในทีมที่มีสิทธิ์
   read-only เพื่อ debug, เครื่องมือ BI/analytics ที่เชื่อมต่อฐานข้อมูลตรงๆ) — encryption at rest
   **ไม่มีผลใดๆ** เพราะฐานข้อมูล**ถอดรหัส disk ให้เองโดยอัตโนมัติ**ก่อนตอบผลลัพธ์ query กลับมาเสมอ
2. **SQL Injection ที่หลุดรอดมาได้** (ทบทวน Part 079) — ถ้าคนร้าย inject query ที่ดึงข้อมูลออกมาได้
   สำเร็จ เขาจะเห็นค่าที่ database คืนกลับมาแบบเดียวกับที่แอปเห็น คือ plaintext เต็มรูปแบบ
3. **Database backup/dump ไฟล์** ที่อาจถูกดาวน์โหลดไปเก็บที่อื่น (`pg_dump` output) — ไฟล์ dump เป็น
   plaintext SQL ธรรมดา ไม่ได้เข้ารหัสตาม disk เดิมอีกต่อไป
4. **Log ของ query ที่ระบบ monitoring บันทึกไว้** (slow query log บางระบบ log ค่า parameter ของ query
   ที่ใช้เวลานานเพื่อ debug performance)

สรุปคือ **encryption at rest ของ managed database ป้องกัน "การขโมย disk แบบ physical" เท่านั้น** ไม่
ป้องกัน "การอ่านผ่าน query" เลยแม้แต่น้อย — Active Record Encryption แก้ปัญหาช่องว่างตรงนี้โดยเฉพาะ

### Active Record Encryption คืออะไร

**Active Record Encryption** เป็น feature ที่มากับ Rails framework เองตั้งแต่ **Rails 7.0** (ไม่ต้อง
ติดตั้ง gem เพิ่มเติมเลย) ทำหน้าที่เข้ารหัสค่าของ **คอลัมน์ระดับ field เดียว** ก่อนที่ Active Record
จะส่งค่านั้นไปเก็บลง database และถอดรหัสให้อัตโนมัติตอนอ่านค่ากลับผ่าน object ปกติ — เขียนโค้ดแค่
เพิ่ม `encrypts :field_name` หนึ่งบรรทัดในโมเดล โดยที่โค้ดส่วนอื่นของแอป (controller, view, business
logic) **ไม่ต้องรู้เลยว่าฟิลด์นี้ถูกเข้ารหัสอยู่** — เรียก `patient.ssn` ได้ค่าจริงตามปกติทุกประการ

จุดเด่นที่สำคัญ:

1. **ทำงานได้กับทุก database adapter ที่ Active Record รองรับ** (SQLite, PostgreSQL, MySQL) — ไม่ต้อง
   พึ่งฟีเจอร์เฉพาะของ database ตัวใดตัวหนึ่ง (ต่างจาก PostgreSQL `pgcrypto` ที่ผูกกับ Postgres
   เท่านั้น)
2. **ไม่ต้องพึ่งบริการภายนอกใดๆ** (ไม่ใช่ AWS KMS, ไม่ใช่ Vault) — กุญแจเข้ารหัสเก็บอยู่ใน Rails
   credentials เอง (ทบทวนกลไกจาก Part 074 — ตอนนี้จะเห็นว่า credentials ที่เรียนไปแล้วมาใช้ตรงนี้พอดี)
3. **โปร่งใสต่อ business logic ทั้งหมด** — validation, callback, serialization (`to_json`) ทำงานกับ
   ค่าที่ถอดรหัสแล้วตามปกติ ไม่ต้องเขียนโค้ดพิเศษเพิ่ม

---

## Step 803: ติดตั้งและเปิดใช้งานจริง — `bin/rails db:encryption:init`, `encrypts :field_name`,
พิสูจน์ว่า Raw DB อ่านไม่ออกแต่ Active Record ถอดรหัสให้อัตโนมัติ

### ขั้นตอนที่ 1 — สร้างกุญแจเข้ารหัสด้วย `db:encryption:init`

Active Record Encryption ต้องการกุญแจสามตัวถึงจะทำงานได้: `primary_key` (กุญแจเข้ารหัสหลัก),
`deterministic_key` (กุญแจสำหรับโหมด deterministic — ดู Step 804), และ `key_derivation_salt` (salt
สำหรับสร้างกุญแจจริงจากกุญแจที่ตั้งไว้) Rails มีคำสั่งสร้างกุญแจสุ่มทั้งสามตัวให้ในคำสั่งเดียว:

```bash
bin/rails db:encryption:init
```

**ผลจากการรันจริง:**

```
Add this entry to the credentials of the target environment:

active_record_encryption:
  primary_key: nvOrx39AtHJfiUZZEkhPKpFl6BK86isf
  deterministic_key: C96rFiyRahU0AHHFlVk5qTEVMEBPbLS5
  key_derivation_salt: ElROXBIvdZBAoLkfXYUOIJgfyBjN4oS4
```

คำสั่งนี้**ไม่ได้เขียนอะไรลงไฟล์ให้อัตโนมัติ** แค่พิมพ์ค่าที่สุ่มสร้างขึ้นออกทาง terminal เท่านั้น —
ขั้นตอนต่อไปคือเรานำก้อน YAML นี้ไปวางใน Rails credentials เอง (ทบทวนกลไกจาก Part 074 Step 732)

### ขั้นตอนที่ 2 — วางกุญแจลงใน Credentials

```bash
EDITOR="code --wait" bin/rails credentials:edit
```

แล้ววางบล็อก `active_record_encryption:` ที่ได้จาก Step ก่อนหน้าต่อท้ายไฟล์ **ทดสอบจริง** ด้วยการ
เปิดดูเนื้อหาหลังบันทึก:

```bash
bin/rails credentials:show
```

```yaml
# ... (secret_key_base และค่าอื่นที่มีอยู่เดิม) ...

active_record_encryption:
  primary_key: nvOrx39AtHJfiUZZEkhPKpFl6BK86isf
  deterministic_key: C96rFiyRahU0AHHFlVk5qTEVMEBPbLS5
  key_derivation_salt: ElROXBIvdZBAoLkfXYUOIJgfyBjN4oS4
```

> **สำคัญมาก — เชื่อมกับ Part 074 โดยตรง:** เพราะกุญแจนี้ผ่าน `bin/rails credentials:edit
> --environment production` แยกได้เหมือนกันทุกประการกับ credentials ทั่วไป **production ควรมีกุญแจ
> เข้ารหัสของตัวเอง แยกจาก development/test เสมอ** ด้วยเหตุผลเดียวกับ Step 732 ของ Part 074 —
> นักพัฒนาที่ทำงานบนเครื่อง dev ไม่ควรมีกุญแจที่ถอดรหัสข้อมูลจริงของผู้ใช้ใน production ได้เลย

### ขั้นตอนที่ 3 — สร้างโมเดลตัวอย่างและเปิดใช้ `encrypts`

สมมติแอปมีตาราง `patients` เก็บข้อมูลคนไข้ ซึ่งมีฟิลด์ละเอียดอ่อนสามฟิลด์: เลขบัตรประชาชน (`ssn`),
บันทึกทางการแพทย์ (`notes`), และอีเมล (`email`) ที่ต้องเข้ารหัสด้วยแต่ยังต้องค้นหาได้ (ดู Step 804):

```bash
bin/rails generate model Patient name:string email:string ssn:string notes:text
bin/rails db:migrate
```

```ruby
# app/models/patient.rb
class Patient < ApplicationRecord
  encrypts :ssn
  encrypts :notes
  encrypts :email, deterministic: true, downcase: true
end
```

**อธิบาย:** `encrypts :ssn` และ `encrypts :notes` คือรูปแบบพื้นฐานที่สุด — เข้ารหัสแบบ
**non-deterministic** (ค่าเข้ารหัสได้ผลลัพธ์ต่างกันทุกครั้งแม้เนื้อหาเดิม ปลอดภัยที่สุด) ส่วน
`encrypts :email, deterministic: true` ให้เหตุผลเต็มใน Step 804

### พิสูจน์ว่า Raw Database มองไม่เห็นค่าจริง

สร้าง record หนึ่งตัวผ่าน Active Record ตามปกติ:

```ruby
Patient.create!(
  name: "สมชาย ใจดี",
  email: "Somchai@Example.com",
  ssn: "1-2345-67890-12-3",
  notes: "มีประวัติแพ้ยา penicillin"
)
```

**อ่านค่ากลับผ่าน Active Record ตามปกติ (ทดสอบจริง):**

```ruby
p = Patient.find(1)
p.ssn    # => "1-2345-67890-12-3"
p.email  # => "somchai@example.com"   (แปลงเป็นตัวพิมพ์เล็กให้ เพราะตั้ง downcase: true)
p.notes  # => "มีประวัติแพ้ยา penicillin"
```

**ตอนนี้คือจุดสำคัญที่สุดของ Step นี้** — ข้าม Active Record แล้วอ่านค่าตรงจาก database ด้วย SQL ดิบๆ
เพื่อจำลองสถานการณ์ที่ DBA หรือคนร้ายที่เข้าถึงฐานข้อมูลโดยตรงจะเห็น (**ทดสอบจริง** ด้วย
`ActiveRecord::Base.connection.select_all` เพื่อข้ามชั้น decrypt ของโมเดลไปเลย):

```ruby
ActiveRecord::Base.connection.select_all(
  "SELECT id, name, email, ssn, notes FROM patients"
).to_a
```

**ผลลัพธ์จริงจาก raw SQL:**

```ruby
{"id"=>1,
 "name"=>"สมชาย ใจดี",
 "email"=>
  "{\"p\":\"fOPtVf7dtciDXVVCida61NVc7w==\",\"h\":{\"iv\":\"O3gkDsl/BYvB9pTm\",\"at\":\"od1iNEHwtBtvdEZrAEUHeQ==\"}}",
 "ssn"=>
  "{\"p\":\"uTDNv7c640MrPFMOAfp/lpw=\",\"h\":{\"iv\":\"MzBseYzZiJUsbUfe\",\"at\":\"1C3EbAqvouJLMORq/Gl1bg==\"}}",
 "notes"=>
  "{\"p\":\"2RAfs5svNytBKtT6oPQdxg4USFNhzwR5h/zXyu1jMc78ElCvrPhp4NtmiJ8t51oRcjHtCmo=\",\"h\":{\"iv\":\"iSj3uC934b0reYgy\",\"at\":\"XaALnwYRa15t1LYbYG1vGA==\"}}"}
```

ผลลัพธ์นี้ยืนยันสิ่งที่ตั้งใจให้เกิดขึ้นได้ครบถ้วน:

1. คอลัมน์ `name` ที่**ไม่ได้เรียก `encrypts`** ยังคงเป็น plaintext เต็มรูปแบบ ("สมชาย ใจดี" อ่านออก
   ตรงๆ) — เพราะเราไม่ได้เข้ารหัสฟิลด์นี้ตั้งแต่แรก (ตัวอย่างที่ตั้งใจให้เห็น contrast ชัดเจน)
2. คอลัมน์ `ssn`, `notes`, `email` ที่เรียก `encrypts` แล้ว **กลายเป็นตัวอักษรไร้ความหมายโดยสิ้นเชิง**
   เมื่อดูตรงจาก database — ต่อให้เอาค่านี้ไปเปิดใน SQL client ตัวไหนก็ตาม จะเห็นแค่ก้อน JSON ที่มี
   ciphertext (`p`), initialization vector (`iv`), และ authentication tag (`at`) เท่านั้น
3. รูปแบบ JSON ที่เห็นคือ **Rails' built-in message format** ของ Active Record Encryption
   (AES-256-GCM cipher เป็นค่า default) — `p` คือ ciphertext ที่เข้ารหัสแล้ว, `h` คือ header ที่เก็บ
   ค่าที่จำเป็นสำหรับการถอดรหัส (`iv` และ `at`) แต่**ไม่มีข้อมูลใดในนี้ที่ช่วยถอดรหัสได้โดยไม่มี
   กุญแจ** (`primary_key` ที่เก็บอยู่ใน credentials เท่านั้น)

> **ข้อสังเกตสำคัญ:** column type ของ `ssn`/`email` ยังเป็น `string` และ `notes` ยังเป็น `text`
> เหมือนเดิมทุกประการ — Active Record Encryption **ไม่ต้องเปลี่ยน schema type ใดๆ เลย** เพราะมันเก็บ
> ciphertext ที่ serialize เป็น JSON string ธรรมดาลงในคอลัมน์ตัวเดิม (มีนัยสำคัญเรื่องขนาดคอลัมน์ที่
> จะกลับมาพูดถึงใน Step 805)

---

## Step 804: Deterministic vs Non-Deterministic Encryption — เมื่อไหร่ต้องแลกความปลอดภัยเพื่อ Query
ได้

### ปัญหา: เข้ารหัสแล้วค้นหาด้วย `where` ไม่ได้อีกต่อไป

ลองสังเกตผลลัพธ์จาก Step 803 อีกครั้ง — ciphertext ของ `ssn` และ `notes` **ไม่มีวันเหมือนกันแม้เนื้อหา
ต้นฉบับจะเหมือนกันทุกตัวอักษร** เพราะ Active Record Encryption สุ่ม initialization vector (`iv`) ใหม่
ทุกครั้งที่เข้ารหัส (โหมด **non-deterministic** ซึ่งเป็นค่า default) **ทดสอบจริง** ด้วยการสร้าง
record คนไข้สองคนที่มีเลขบัตรประชาชน**เหมือนกันเป๊ะ** (สมมติกรณีทดสอบ ไม่ใช่ค่าจริง):

```ruby
Patient.create!(name: "คนที่หนึ่ง", ssn: "1-2345-67890-12-3", ...)
Patient.create!(name: "คนที่สอง",  ssn: "1-2345-67890-12-3", ...)
```

**ciphertext ที่ได้จริงจาก raw DB:**

```
id=1 ssn={"p":"uTDNv7c640MrPFMOAfp/lpw=", "h":{"iv":"MzBseYzZiJUsbUfe", ...}}
id=2 ssn={"p":"D15wMO1/je6tgeFKtvPZDQk=", "h":{"iv":"6jXJwQDuZyEnHukJ", ...}}
```

**ciphertext ต่างกันโดยสิ้นเชิงแม้ plaintext เหมือนกันทุกตัวอักษร** — นี่คือพฤติกรรมที่ตั้งใจ (ป้องกัน
ไม่ให้คนที่มองแค่ ciphertext อนุมานได้ว่า "record สองตัวนี้มีค่าเดียวกัน") แต่ผลข้างเคียงคือ **ทำให้
เขียน `Patient.where(ssn: "...")` เพื่อค้นหาไม่ได้อีกต่อไปเลย** เพราะ Rails ไม่รู้ล่วงหน้าว่า ciphertext
รูปแบบไหนในตาราง match กับค่าที่ต้องการค้นหา (ต้องถอดรหัสทุกแถวมาเทียบทีละแถว ซึ่งไม่มี query ไหนทำแบบ
นั้นให้อัตโนมัติ)

**ทดสอบจริง** ยืนยันว่า query แบบนี้ไม่ error แต่คืนค่าว่างเปล่าเสมอ (ไม่ match อะไรเลย):

```ruby
Patient.where(ssn: "1-2345-67890-12-3").to_a
# => [] (ไม่ error แต่หาไม่เจอ เพราะ ciphertext ที่สร้างใหม่ตอน query ไม่มีวันตรงกับที่เก็บไว้)
```

### ทางออก: `deterministic: true`

```ruby
class Patient < ApplicationRecord
  encrypts :email, deterministic: true, downcase: true
end
```

`deterministic: true` เปลี่ยนวิธีสร้าง initialization vector จาก "สุ่มทุกครั้ง" เป็น **"คำนวณจาก
เนื้อหาที่เข้ารหัสเอง"** — ผลคือ **เนื้อหาเดียวกันเข้ารหัสกี่ครั้งก็ได้ ciphertext เดิมเป๊ะทุกครั้ง**
ทำให้ query ด้วย `where` ทำงานได้ตามปกติ เพราะ Rails คำนวณ ciphertext ของค่าที่ค้นหา แล้วเทียบ
ciphertext ตรงๆ กับที่เก็บในตาราง (ไม่ต้องถอดรหัสทั้งตารางมาเทียบ)

**ทดสอบจริง** ยืนยันว่า query ด้วย field ที่ตั้ง deterministic ทำงานได้ปกติทุกประการ:

```ruby
Patient.where(email: "somchai@example.com").exists?
# => true

Patient.find_by(email: "SOMCHAI@example.com")&.name
# => "สมชาย ใจดี"   (ตรงเพราะ downcase: true แปลงเป็นตัวพิมพ์เล็กให้ทั้งตอนบันทึกและตอน query)
```

`downcase: true` มีประโยชน์เฉพาะกับ deterministic encryption — ทำให้ query แบบไม่สนตัวพิมพ์เล็ก/ใหญ่
ทำงานได้ (เหมือน `LOWER(email) = LOWER(?)` ใน SQL ปกติ) โดยไม่ต้องเขียน query พิเศษเอง แต่ต้องแลกด้วย
การ**สูญเสียตัวพิมพ์เดิม**ไปถาวร (ถ้าต้องการเก็บตัวพิมพ์เดิมไว้ด้วยพร้อม query แบบไม่สนตัวพิมพ์ Active
Record Encryption มี option `ignore_case: true` ที่เก็บคอลัมน์คู่ขนานสำรองตัวพิมพ์เดิมไว้ให้ — เหมาะกับ
กรณีที่ต้องแสดงอีเมลกลับให้ผู้ใช้เห็นตามที่พิมพ์มาจริง)

### Trade-off ที่ต้องเข้าใจให้ชัดก่อนเลือกใช้ `deterministic: true`

| | Non-deterministic (default) | Deterministic (`deterministic: true`) |
|---|---|---|
| ciphertext ของค่าเดียวกัน | ต่างกันทุกครั้ง | เหมือนกันทุกครั้ง |
| Query ด้วย `where`/`find_by` | ทำไม่ได้ | ทำได้ปกติ |
| `validates_uniqueness_of` | ทำไม่ได้อย่างแม่นยำ (ต้อง query ผ่าน ciphertext ที่ต่างกัน) | ทำได้ตามปกติ |
| ความปลอดภัย | สูงสุด — ไม่รั่วไหลรูปแบบข้อมูลเลย | **อ่อนกว่าเล็กน้อยโดยธรรมชาติ** |
| ความเสี่ยงที่แลกมา | - | คนที่มองแค่ ciphertext ในตาราง (เช่น DBA) เห็นว่า "record ไหนมีค่าเหมือนกัน" แม้ไม่รู้ค่าจริง (frequency analysis) |

จุดที่ต้องเน้น**ให้ชัดเจนที่สุด**: การที่ ciphertext ของค่าเดียวกันออกมาเหมือนกันทุกครั้ง หมายความว่า
คนที่มีสิทธิ์อ่าน raw database (ไม่มีกุญแจถอดรหัส) **ยังคงเห็นได้ว่า "user คนไหนมีอีเมลเดียวกันกับ user
อีกคน"** แม้จะไม่รู้ว่าอีเมลนั้นคืออะไร — นี่คือการรั่วไหลของ**รูปแบบข้อมูล** (pattern) แม้เนื้อหาจริง
ยังปลอดภัยอยู่ ด้วยเหตุนี้จึงมีกฎการตัดสินใจที่ใช้ได้จริง:

> **ใช้ `deterministic: true` เฉพาะฟิลด์ที่ "จำเป็นต้อง query หรือบังคับ unique" เท่านั้น
> (เช่น email ที่ใช้ login) ฟิลด์ที่ไม่เคยต้อง `where`/`find_by`/uniqueness เลย (เช่น ssn, บันทึกทาง
> การแพทย์) ให้ใช้ non-deterministic เสมอ เพื่อความปลอดภัยสูงสุด**

ในตัวอย่างของ Part นี้ `ssn` และ `notes` ใช้ non-deterministic เพราะแอปไม่เคย query หาคนไข้ด้วยเลข
บัตรประชาชนหรือเนื้อหาบันทึกโดยตรง (ค้นด้วย `id` หรือ `name` เป็นหลัก) ส่วน `email` ใช้ deterministic
เพราะต้องใช้ log in ด้วยอีเมล ซึ่งจำเป็นต้อง query แบบเป๊ะ

---

## Step 805: เข้ารหัสคอลัมน์ที่มีข้อมูลเดิมอยู่แล้ว — Backfill แบบปลอดภัยสำหรับ Migration จริง

Step 803–804 สาธิตกับตารางที่**สร้างใหม่ ยังไม่มีข้อมูล** — สถานการณ์ที่พบได้บ่อยกว่ามากในงานจริงคือ
**ตารางมีข้อมูล plaintext อยู่แล้วเป็นแสนแถว แล้ววันหนึ่งทีม security บอกว่าต้องเข้ารหัสคอลัมน์นี้**
Step นี้จะไล่ทีละขั้นตอนที่ปลอดภัย ทดสอบจริงทุกจุด

### ขั้นตอนที่ 1 — เปิด `support_unencrypted_data` ชั่วคราว

ถ้าเพิ่ม `encrypts :phone` เข้าไปในโมเดลที่มีข้อมูล plaintext เดิมอยู่แล้วทันที Active Record จะพยายาม
**ถอดรหัส** ค่าที่อ่านมาจากทุกแถวเสมอ ซึ่งแถวเก่าที่ยังเป็น plaintext จะทำให้ decrypt ล้มเหลว (ไม่ใช่
รูปแบบ JSON ที่เข้ารหัสไว้) ต้องเปิด flag นี้ไว้ชั่วคราวระหว่างช่วง migrate:

```ruby
# config/application.rb (หรือ config/environments/production.rb เฉพาะ environment ที่กำลัง backfill)
config.active_record.encryption.support_unencrypted_data = true
```

ทดสอบจำลองสถานการณ์นี้จริง — สมมติตาราง `customers` มีคอลัมน์ `phone` เป็น plaintext อยู่แล้วสองแถว
ก่อนเปิด encryption ใดๆ:

```
== ก่อนเปิด encryption (raw DB) ==
id=1 phone(raw)=0812345678
id=2 phone(raw)=0898765432
```

เพิ่ม `encrypts :phone` เข้าไปในโมเดล พร้อมเปิด `support_unencrypted_data = true` แล้วอ่านค่ากลับผ่าน
Active Record ทันที (**ทดสอบจริง**):

```ruby
class Customer < ApplicationRecord
  encrypts :phone
end
```

```
== อ่านผ่าน AR ทันทีหลังเปิด encrypts (ต้องอ่าน plaintext เดิมได้แบบ graceful) ==
id=1 phone(AR)=0812345678
id=2 phone(AR)=0898765432
```

**ยืนยันแล้ว:** เมื่อเปิด `support_unencrypted_data = true` Active Record Encryption ฉลาดพอที่จะ
**ตรวจจับเองว่าค่านี้เป็น plaintext เก่าหรือ ciphertext ใหม่** แล้วอ่านให้ถูกต้องทั้งสองแบบโดยไม่ error
— นี่คือกลไกที่ทำให้ deploy โค้ดที่เพิ่ม `encrypts` เข้าไปได้โดย **ไม่มี downtime และไม่มี request ไหน
พังระหว่างที่ยังมีข้อมูลเก่าปนอยู่**

### ขั้นตอนที่ 2 — Backfill ข้อมูลเก่าให้กลายเป็น Ciphertext จริง

นี่คือจุดที่ผู้เขียนหลักสูตรพบ**ข้อผิดพลาดที่พบบ่อยที่สุด**เวลาทำจริง ลองวิธีที่ "ดูเหมือนจะถูก" ก่อน
(**ทดสอบจริงเพื่อพิสูจน์ว่าวิธีนี้ใช้ไม่ได้**):

```ruby
# วิธีที่ "ดูเหมือน" น่าจะบังคับให้เขียนค่าใหม่ (encrypt) แต่จริงๆ ใช้ไม่ได้!
Customer.find_each { |c| c.update!(phone: c.phone) }
```

```
== หลัง update!(phone: c.phone) (raw DB) ==
id=1 phone(raw)=0812345678   -- ยังเป็น plaintext เหมือนเดิมทุกตัวอักษร!
id=2 phone(raw)=0898765432   -- ไม่มีการเข้ารหัสเกิดขึ้นเลย
```

**ทดสอบยืนยันสาเหตุ:**

```ruby
c = Customer.first
c.phone = c.phone
c.changed?         # => false
c.phone_changed?   # => false
```

**สาเหตุ:** Active Record มีกลไก **dirty tracking** ที่ตรวจสอบว่า attribute ถูกแก้ไขจริงหรือไม่ก่อน
ตัดสินใจว่าจะสร้างคำสั่ง `UPDATE` หรือไม่ — เมื่อกำหนดค่าเดิมกลับเข้าไปตัวเดิมเป๊ะ (`c.phone = c.phone`)
Rails เห็นว่า**ค่าที่ deserialize แล้วเหมือนเดิมทุกประการ** จึงสรุปว่า "ไม่มีอะไรเปลี่ยน" แล้ว **ข้าม
การเขียนคำสั่ง SQL ทั้งหมด** — `update!` จึงทำงาน "สำเร็จ" (ไม่ error) แต่ไม่ได้เขียนอะไรลง database
เลยจริงๆ ผลคือคอลัมน์ยังเป็น plaintext เหมือนเดิมทุกประการ นี่คือกับดักที่ตรวจจับได้ยากมากถ้าไม่ทดสอบ
กับ raw SQL แบบที่ทำอยู่นี้ (โค้ดดูเหมือนรันผ่านปกติทุกอย่าง ไม่มี error ให้เห็นเลย)

### วิธีที่ถูกต้อง: `record.encrypt`

Active Record Encryption มี public API ที่ออกแบบมาเพื่อการนี้โดยเฉพาะ — method `#encrypt` ที่มากับทุก
model ที่เรียก `encrypts` อย่างน้อยหนึ่งฟิลด์:

```ruby
Customer.find_each(&:encrypt)
```

**ทดสอบจริง:**

```
== ก่อน backfill (raw DB) ==
id=4 phone=0812345678
id=5 phone=0898765432

== หลัง backfill ด้วย record.encrypt (raw DB) ==
id=4 phone={"p":"8F2o/+iRgfi4uQ==","h":{"iv":"iKq5xu0E1eB6PjLg","at":"guYguTGcWZaVkqBLsfBSEw=="}}
id=5 phone={"p":"/NBSlljYvIPPLg==","h":{"iv":"JF7lFdKgZWrNfBrw","at":"8TRVCmenesmhYWS9Qejsdg=="}}

== อ่านผ่าน AR หลัง backfill (ต้องได้ค่าเดิมกลับมา) ==
id=4 phone=0812345678
id=5 phone=0898765432
```

**ยืนยันได้ครบถ้วน:** `record.encrypt` เข้ารหัสจริงและ**เขียนลง database ด้วย `update_columns`
โดยตรง** (ข้าม dirty tracking ไปเลยตามการออกแบบภายในของ method นี้ ไม่ผ่าน validation/callback ซ้ำ
ด้วย เพื่อความเร็วตอน backfill ข้อมูลจำนวนมาก) ทำให้ ciphertext ถูกเขียนจริงทุกครั้งไม่ว่าค่าจะ
"เปลี่ยน" ในมุมมองของ dirty tracking หรือไม่

### สคริปต์ backfill ที่ใช้งานได้จริงสำหรับตารางขนาดใหญ่

```ruby
# lib/tasks/encrypt_customers.rake
namespace :db do
  namespace :encryption do
    desc "Backfill: เข้ารหัสคอลัมน์ phone ของ Customer ที่มีอยู่เดิมทั้งหมด"
    task backfill_customer_phone: :environment do
      total = Customer.count
      Customer.find_each(batch_size: 1000).with_index do |customer, index|
        customer.encrypt
        puts "#{index + 1}/#{total} เข้ารหัสแล้ว" if (index + 1) % 1000 == 0
      end
      puts "เสร็จสิ้น — เข้ารหัส Customer ทั้งหมด #{total} รายการแล้ว"
    end
  end
end
```

`find_each(batch_size: 1000)` (ทบทวนหลักการ batch iteration จาก Part 034 เรื่อง query interface)
ป้องกันไม่ให้โหลดทุกแถวเข้า memory พร้อมกันในตารางขนาดใหญ่ระดับล้านแถว — สำคัญมากเพราะการ backfill
มักรันกับตารางที่มีข้อมูลจริงจำนวนมากอยู่แล้ว

### อย่าลืมปิด `support_unencrypted_data` กลับหลัง Backfill เสร็จ

```ruby
# ลบหรือ comment บรรทัดนี้ออกหลัง backfill เสร็จสมบูรณ์แล้วทุกแถว
# config.active_record.encryption.support_unencrypted_data = true
```

การปล่อยให้ flag นี้เปิดค้างไว้ตลอดไปมีความเสี่ยง — ถ้าโค้ดที่ไหนเผลอเขียน plaintext ลงคอลัมน์นี้ตรงๆ
(ข้าม Active Record เช่นผ่าน raw SQL) แอปจะ**ยอมอ่านมันโดยไม่แจ้งเตือนเลย** ซึ่งเท่ากับเปิดช่องให้ข้อมูล
ที่ไม่ได้เข้ารหัสหลุดเข้าคอลัมน์นี้ได้อีกโดยไม่มีใครรู้ ควรปิด flag นี้ทันทีที่มั่นใจว่าทุกแถวถูก
backfill ครบแล้ว (ตรวจสอบด้วยการนับว่ามีแถวไหนที่ `ciphertext_for(:phone)` ยังไม่ใช่รูปแบบเข้ารหัส
เหลืออยู่หรือไม่ ก่อนปิด flag)

### กับดักที่สองที่ต้องรู้: ขนาดคอลัมน์กับ Ciphertext ที่ใหญ่กว่า Plaintext มาก

Active Record Encryption เพิ่ม validation ความยาวให้อัตโนมัติกับทุกฟิลด์ที่ `encrypts` (ผ่าน
`config.active_record.encryption.validate_column_size` ที่เป็น `true` โดย default) แต่**ทดสอบจริง
พบพฤติกรรมที่ต้องระวัง**:

```ruby
# คอลัมน์ short_field ถูก migrate ด้วย limit: 50 (varchar(50))
value_40_chars = "a" * 40
c = Customer.new(name: "x", short_field: value_40_chars)
c.valid?  # => true  (ผ่าน validation เพราะ plaintext ยาว 40 ตัวอักษร < limit 50)

c.ciphertext_for(:short_field).bytesize
# => 126   -- ciphertext จริงยาวถึง 126 byte ทั้งที่ column limit ตั้งไว้แค่ 50!
```

**สิ่งที่ทดสอบนี้เปิดเผย:** validation ความยาวที่ Active Record Encryption เพิ่มให้อัตโนมัติ **เทียบ
ความยาวของ plaintext กับ limit ของคอลัมน์ตรงๆ** ไม่ได้คำนวณว่า ciphertext (ที่ต้องห่อด้วย JSON +
base64 + IV + authentication tag) จะใหญ่กว่า plaintext แค่ไหน — ผลคือ validation นี้ **ป้องกันได้แค่
กรณีสุดโต่ง** (plaintext ยาวเกิน limit ตรงๆ) แต่**ไม่รับประกันว่า ciphertext จะพอดีกับคอลัมน์เสมอไป**
ในตัวอย่างข้างต้น ciphertext ยาวกว่า limit ถึง 2.5 เท่า แต่ validation กลับผ่าน — ถ้าคอลัมน์จริงเป็น
`varchar(50)` การเขียนค่านี้ลง PostgreSQL/MySQL ที่บังคับ limit เข้มงวดจะทำให้เกิด database error ตอน
`INSERT`/`UPDATE` จริง (ไม่ใช่ validation error ที่จับได้สวยงามในแอป)

> **แนวปฏิบัติที่ถูกต้อง:** ฟิลด์ที่จะเรียก `encrypts` ควรใช้ column type **`:text`** (ไม่มี limit
> ตายตัว) เสมอ แทน `:string` ที่มักมี limit 255 ตัวอักษรโดย default ในหลาย database หรือถ้าจำเป็นต้อง
> ใช้ `:string` จริงๆ ให้ตั้ง `limit:` ให้กว้างกว่าที่คิดไว้มากพอสมควร (ciphertext ของ Active Record
> Encryption ใหญ่กว่า plaintext ประมาณ 1.5–3 เท่า ขึ้นกับความยาวต้นฉบับ เพราะ overhead ของ JSON
> wrapper + base64 encoding + IV + auth tag ที่คงที่ต่อค่าหนึ่งค่า) — ในตัวอย่างของ Part นี้ `notes`
> ถูก generate เป็น `:text` มาตั้งแต่แรกแล้ว (ไม่มี limit) จึงไม่มีความเสี่ยงนี้เลย

---

## Step 806: Key Rotation สำหรับ Active Record Encryption — หมุนเวียนกุญแจโดยข้อมูลเก่าไม่เสียหาย

### ทำไมต้อง Rotate กุญแจเข้ารหัสข้อมูล

เหตุผลเดียวกับที่ Part 074 Step 738 สอนเรื่อง rotate API key — นโยบายความปลอดภัยตามรอบเวลา, พนักงานที่
เคยมีสิทธิ์เข้าถึงกุญแจลาออก, หรือสงสัยว่ากุญแจรั่วไหล แต่กุญแจเข้ารหัสข้อมูล (`primary_key`) มีความ
ท้าทายเพิ่มเติมที่ credentials ทั่วไปไม่มี: **ข้อมูลนับล้านแถวที่เข้ารหัสด้วยกุญแจเก่าไปแล้วยังต้อง
อ่านได้อยู่** จะไปเปลี่ยนกุญแจแล้วปล่อยให้ข้อมูลเก่าอ่านไม่ออกเลยไม่ได้ (ต่างจาก rotate API key ของ
บริการภายนอกที่แค่เปลี่ยนค่าไปข้างหน้าอย่างเดียวได้)

### กลไก: `primary_key` รับเป็น Array ได้

```yaml
# config/credentials.yml.enc — รูปแบบตอน rotate (มีสองกุญแจพร้อมกันชั่วคราว)
active_record_encryption:
  primary_key:
    - <กุญแจเก่า>
    - <กุญแจใหม่>
  deterministic_key:
    - <กุญแจเก่า>
    - <กุญแจใหม่>
  key_derivation_salt: <salt เดิม ไม่ต้องเปลี่ยน>
```

**กฎสำคัญที่ต้องจำให้แม่น:** เมื่อ `primary_key` เป็น Array — **กุญแจตัวสุดท้ายในลิสต์เท่านั้นที่ถูก
ใช้เข้ารหัสข้อมูลใหม่** ส่วน**กุญแจทุกตัวในลิสต์**จะถูกลองใช้ถอดรหัสเสมอเมื่ออ่านค่าที่มีอยู่แล้ว —
พฤติกรรมนี้มาจาก source code จริงของ `ActiveRecord::Encryption::KeyProvider`:

```ruby
# ActiveRecord::Encryption::KeyProvider (จาก activerecord-8.1.4 — อ่านโค้ดจริง)
def encryption_key
  @keys.last  # ใช้กุญแจตัวสุดท้ายเข้ารหัสข้อมูลใหม่เสมอ
end

def decryption_keys(encrypted_message)
  @keys       # ลองกุญแจทุกตัวในลิสต์เพื่อถอดรหัส
end
```

### พิสูจน์ด้วยการทดสอบจริงทั้งสามช่วง (Phase)

**ช่วงที่ 1 — มีกุญแจเดียว (`key1`) ก่อน rotate:**

```ruby
r1 = RotTest.create!(name: "เข้ารหัสด้วย key1", ...)
r1.reload.name
# => "เข้ารหัสด้วย key1"
```

**ช่วงที่ 2 — Rotate: เพิ่ม `key2` เข้าไปเป็นกุญแจตัวใหม่ (ตัวสุดท้ายในลิสต์) แต่ยังเก็บ `key1` ไว้:**

```ruby
# primary_key: [key1, key2]  -- key2 คือกุญแจเข้ารหัสตัวใหม่ (อยู่ท้ายลิสต์)
r1_reloaded = RotTest.find(r1.id)
r1_reloaded.name
# => "เข้ารหัสด้วย key1"   -- ข้อมูลเก่ายังอ่านได้ปกติ เพราะ key1 ยังอยู่ในลิสต์ decryption

r2 = RotTest.create!(name: "เข้ารหัสด้วย key2 (ตัวใหม่)", ...)
r2.reload.name
# => "เข้ารหัสด้วย key2 (ตัวใหม่)"   -- record ใหม่เข้ารหัสด้วย key2 (ตัวท้ายลิสต์) แล้ว
```

**ช่วงที่ 3 — ถอด `key1` ออกจาก config ทั้งหมด (จำลองว่า key1 ถูก decommission แล้วจริงๆ):**

```ruby
# primary_key: [key2]  -- เหลือแค่ key2 ตัวเดียว
RotTest.find(r1.id).name
# => raise ActiveRecord::Encryption::Errors::Decryption
#    (r1 เข้ารหัสด้วย key1 ที่ถูกถอดออกจาก config แล้ว จึงถอดรหัสไม่ได้อีกต่อไป!)

RotTest.find(r2.id).name
# => "เข้ารหัสด้วย key2 (ตัวใหม่)"   -- r2 ยังอ่านได้ปกติ เพราะเข้ารหัสด้วย key2 มาตั้งแต่แรก
```

**ผลการทดสอบทั้งสามช่วงนี้คือของจริงทั้งหมด** ยืนยันกลไกได้ครบถ้วน — **ถ้าจะถอดกุญแจเก่าออกจาก config
จริงๆ ต้องมั่นใจก่อนว่าไม่มี record ไหนในระบบที่ยังเข้ารหัสด้วยกุญแจนั้นค้างอยู่เลย**

### ขั้นตอนการ Rotate ที่ใช้ได้จริงในงานจริง

1. เพิ่มกุญแจใหม่ต่อท้าย array ใน credentials (`primary_key: [key_เก่า, key_ใหม่]`) แล้ว deploy —
   ตั้งแต่จุดนี้ record ใหม่ทุกตัวเข้ารหัสด้วยกุญแจใหม่ทันที ส่วน record เก่ายังอ่านได้ปกติ
2. **Re-encrypt ข้อมูลเก่าทั้งหมดด้วยกุญแจใหม่** — ใช้ pattern เดียวกับ Step 805 (`record.encrypt`
   ผ่าน rake task) เพื่อบังคับให้ทุกแถวถูกเขียนใหม่ด้วยกุญแจตัวล่าสุด (`record.encrypt` เขียนด้วย
   `encryption_key` ปัจจุบันเสมอ ซึ่งคือกุญแจตัวท้ายสุดของ array)
3. ตรวจสอบให้แน่ใจว่า**ไม่มีแถวไหนเหลือที่เข้ารหัสด้วยกุญแจเก่า** ก่อนไปขั้นตอนถัดไป (เขียน script
   ตรวจสอบ หรือดูจาก key reference ถ้าเปิด `config.active_record.encryption.store_key_references =
   true` ที่ฝัง id ของกุญแจที่ใช้เข้ารหัสไว้ใน header ของแต่ละ ciphertext ทำให้ query ได้ว่าแถวไหนยัง
   ใช้กุญแจเก่าอยู่)
4. เมื่อมั่นใจว่า re-encrypt ครบทุกแถวแล้ว ถึงจะถอดกุญแจเก่าออกจาก `primary_key` array ได้อย่าง
   ปลอดภัย (`primary_key: [key_ใหม่]` เท่านั้น) แล้ว deploy อีกครั้ง
5. บันทึกลง runbook/incident log เหมือนที่ Part 074 Step 738 สอนไว้เรื่อง secrets rotation ทั่วไป

---

## Step 807: ขอบเขตของ Active Record Encryption — สิ่งที่มันไม่ได้ป้องกัน

การเข้าใจ**ขอบเขต**ของเครื่องมือความปลอดภัยทุกตัวสำคัญพอๆ กับการเข้าใจว่ามันป้องกันอะไร — เพราะการ
เข้าใจผิดว่าเครื่องมือหนึ่งป้องกันได้ทุกอย่างคือสาเหตุของช่องโหว่ที่อันตรายที่สุด (ความประมาทที่มาจาก
ความมั่นใจผิดๆ)

### 1. ไม่ป้องกันคนที่เข้าถึง Rails process ที่กำลังรันอยู่

**ทดสอบจริงที่ทำไปแล้วใน Step 803–804 คือหลักฐานของข้อนี้เอง** — `bin/rails console` หรือ
`bin/rails runner` ที่รันในเครื่อง production (ซึ่งมีสิทธิ์อ่าน credentials ที่ deploy ไปด้วย) อ่านค่า
`patient.ssn` ออกมาเป็น plaintext ได้ตามปกติทุกประการ — Active Record Encryption **ปกป้องเฉพาะ
"ช่องทางที่ข้าม Active Record ไปอ่าน raw storage ตรงๆ"** เท่านั้น (raw SQL, database dump, disk) ไม่ได้
ปกป้อง"ช่องทางที่ผ่านแอปตามปกติ" เลย

ผลกระทบเชิงปฏิบัติ: ถ้าคนร้ายมี **Remote Code Execution (RCE)** บนเซิร์ฟเวอร์ production ได้ (เช่นผ่าน
ช่องโหว่ gem ที่ไม่ได้ patch — เหตุผลที่ Part 075 สอนเรื่อง `bundler-audit`) หรือขโมย credentials ของ
production ไปได้ (เหตุผลที่ Part 074 ทั้ง Part สอนเรื่องปกป้อง credentials) เขาสามารถรันโค้ด Ruby ที่
เรียก `Patient.all.map(&:ssn)` เพื่อดึงข้อมูลที่ถอดรหัสแล้วออกมาทั้งหมดได้ทันที — Active Record
Encryption **ไม่ใช่ทางออกสุดท้าย** แต่เป็น **defense-in-depth ชั้นหนึ่ง** ที่เพิ่มเข้ามาเสริมกับการ
ป้องกัน application-level compromise ที่ยังต้องทำอยู่ดี (ปิดช่องโหว่ตาม Part 079–080, patch
dependency ตาม Part 075, จำกัดสิทธิ์ตาม Pundit/CanCanCan ที่เรียนใน Phase 5)

### 2. ไม่ทดแทน Authorization

Active Record Encryption ตอบคำถาม "คนที่เข้าถึง database โดยตรงเห็นอะไร" — มันไม่ตอบคำถาม "user คนนี้
**ควรมีสิทธิ์**เห็นข้อมูลของคนไข้คนนี้ผ่านหน้าเว็บหรือไม่" เลย นั่นยังเป็นหน้าที่ของ authorization
layer (Pundit/CanCanCan จาก Phase 5) ทั้งหมด — ต่อให้เข้ารหัส `ssn` ไว้สมบูรณ์แบบ ถ้า controller ไม่มี
`authorize @patient` ที่ถูกต้อง user คนไหนก็ log in เข้ามาดู `ssn` ของคนไข้คนไหนก็ได้ผ่านหน้าเว็บปกติ
อยู่ดี (Active Record ถอดรหัสให้เพราะแอปมีกุญแจ — encryption ไม่รู้เรื่อง "สิทธิ์" เลย)

### 3. ไม่ป้องกันการรั่วไหลผ่าน Log หรือช่องทางอื่นที่แอปเขียนเอง

```ruby
# อันตราย — ค่าที่ Active Record ถอดรหัสให้แล้วหลุดเข้า log ตามปกติ ไม่มีอะไรพิเศษป้องกันให้
Rails.logger.info("Processing patient: #{patient.ssn}")

# อันตรายเท่ากัน — error tracking service (Sentry ที่เรียนใน Part 078) capture ค่านี้ไปเก็บด้วย
raise "Invalid SSN format: #{patient.ssn}"
```

เมื่อค่าถูกถอดรหัสแล้วในหน่วยความจำของแอป (เพื่อเอาไปแสดงผล/ประมวลผล) **มันคือ String ธรรมดา** ที่ไม่มี
กลไกป้องกันพิเศษใดๆ ติดตัวไปด้วยเลย ถ้าโค้ดส่วนอื่นเผลอ log มันหรือส่งมันไปที่อื่น (external API,
analytics event) มันจะรั่วไหลได้ไม่ต่างจากข้อมูลธรรมดาใดๆ — `config.filter_parameters` (ที่ Part 074
Step 739 พูดถึง) ช่วยกรอง parameter ของ HTTP request ที่ชื่อเข้าเงื่อนไขจาก log อัตโนมัติ แต่ไม่ช่วย
กรองสิ่งที่แอปเขียนลง log เองตรงๆ แบบนี้

### 4. ไม่ทดแทน HTTPS/TLS

Active Record Encryption เข้ารหัสข้อมูล**ตอนพัก** (at rest — ในฐานข้อมูล) เท่านั้น มันไม่เกี่ยวข้องกับ
ข้อมูล**ระหว่างเดินทาง** (in transit — ระหว่าง browser กับ server) เลยแม้แต่น้อย — นั่นยังเป็นหน้าที่
ของ `config.force_ssl` และ TLS certificate (Step 808 จะพูดถึงอีกครั้งในบริบท checklist)

### สรุปขอบเขต

| สถานการณ์ | Active Record Encryption ช่วยไหม |
|---|---|
| DBA/นักวิเคราะห์ query database ตรงๆ | **ช่วย** — เห็นแค่ ciphertext |
| Database backup/dump รั่วไหล | **ช่วย** — dump ก็มีแต่ ciphertext |
| SQL Injection ที่หลุดรอดมาได้ | **ช่วย** — query ที่ inject ได้ผลลัพธ์เป็น ciphertext เช่นกัน |
| คนร้ายมี RCE บนเซิร์ฟเวอร์ที่รันแอปอยู่ | **ไม่ช่วย** — แอปมีกุญแจ ถอดรหัสให้ได้ทันที |
| Credentials ของ production รั่วไหล (รวมกุญแจเข้ารหัส) | **ไม่ช่วย** — เหมือนแจกกุญแจให้คนร้ายไปด้วย |
| Controller ไม่ตรวจสอบสิทธิ์ user ก่อนแสดงข้อมูล | **ไม่ช่วย** — encryption ไม่รู้จัก "สิทธิ์" |
| แอป log ค่าที่ถอดรหัสแล้วโดยไม่ตั้งใจ | **ไม่ช่วย** — ค่าที่ถอดรหัสแล้วคือ String ธรรมดา |
| ข้อมูลระหว่างเดินทางบนเครือข่าย (HTTP) | **ไม่เกี่ยวข้อง** — เป็นหน้าที่ของ HTTPS/TLS |

---

## Step 808: Security Checklist ก่อนขึ้น Production — รวบรวมทุกหัวข้อจากทั้งหลักสูตร

นี่คือหัวใจของ Part นี้ — checklist ที่รวบรวมทุกสิ่งที่หลักสูตรสอนมาตลอดตั้งแต่ Phase 3 จนถึงตอนนี้ ให้
เป็นรายการเดียวที่ใช้ตรวจสอบได้จริงก่อน deploy แอปใดๆ สู่ผู้ใช้งานจริง **แต่ละข้อจะบอกว่า Part ไหน
เคยสอนละเอียดไปแล้ว** (ไม่สอนซ้ำในที่นี้ เป็นแค่การ recap เพื่อรวมเป็นภาพเดียว)

### หมวดที่ 1 — ป้องกัน Request ที่เข้ามา (Input/Request Layer)

- [ ] **Strong Parameters** ใช้กับทุก controller action ที่รับ input จากผู้ใช้ — ไม่มี `params.permit!`
      หรือการส่ง `params` ทั้งก้อนเข้า `.create`/`.update` ตรงๆ โดยไม่ผ่าน `permit` (Part 080)
- [ ] **CSRF protection** เปิดอยู่ (`protect_from_forgery` เป็นค่า default ของ `ApplicationController`
      อยู่แล้ว — ตรวจสอบว่าไม่มีใครปิดมันทิ้งโดยไม่มีเหตุผลที่ดีพอ) (Part 079)
- [ ] **ไม่มี Raw SQL ที่ interpolate ค่าจากผู้ใช้ตรงๆ** — ใช้ parameterized query (`where("name = ?",
      value)` หรือ `where(name: value)`) เสมอ ไม่ใช่ `where("name = '#{value}'")` (Part 079)
- [ ] **View ไม่ใช้ `raw`/`html_safe`/`.html_safe` กับค่าที่มาจากผู้ใช้โดยไม่ sanitize ก่อน** — ป้องกัน
      Stored/Reflected XSS (Part 079)
- [ ] **Brakeman scan ผ่านโดยไม่มี warning ที่ยังไม่ได้ตรวจสอบ** — รัน `bundle exec brakeman` ก่อน
      deploy ทุกครั้ง (แนะนำให้ผูกเข้ากับ CI ตาม Part 075) และทุก warning ที่ตั้งใจ ignore ต้องมี
      เหตุผลบันทึกไว้ใน `config/brakeman.ignore` (Part 080)
- [ ] **Rate limiting** เปิดใช้งานสำหรับ endpoint ที่เสี่ยงถูก brute-force หรือ abuse (login, API
      endpoint สาธารณะ) ผ่าน `rack-attack` (Part 060)

### หมวดที่ 2 — Secrets และ Config

- [ ] `.env`, `config/master.key`, `config/credentials/*.key` **ไม่ถูก track ใน git**
      (`git ls-files | grep -E '\.env$|\.key$'` ต้องไม่มี output) (Part 074)
- [ ] Production มี credentials **แยกไฟล์**จาก development/test (Part 074)
- [ ] ไม่มี `Rails.logger`/`puts`/`p` ที่ print ค่า secret หรือข้อมูลส่วนบุคคลตรงๆ ในโค้ด (Part 074,
      Step 807 ของ Part นี้)
- [ ] มี runbook ขั้นตอนการ rotate secret แต่ละตัว (Part 074 Step 738, Step 806 ของ Part นี้)

### หมวดที่ 3 — ข้อมูลที่เก็บใน Database (หัวข้อของ Part นี้)

- [ ] **ฟิลด์ที่เป็นข้อมูลส่วนบุคคลละเอียดอ่อน** (เลขบัตรประชาชน, ข้อมูลทางการแพทย์, ข้อมูลการเงิน,
      บันทึกส่วนตัว) ถูก `encrypts` ไว้แล้ว — เลือก `deterministic: true` เฉพาะฟิลด์ที่จำเป็นต้อง
      query/unique เท่านั้น (Step 802–804)
- [ ] คอลัมน์ที่ `encrypts` ใช้ type `:text` หรือมี `limit` ที่กว้างพอสำหรับ ciphertext (Step 805)
- [ ] Backfill ข้อมูลเก่าครบทุกแถวแล้ว และปิด `support_unencrypted_data` กลับเรียบร้อย (Step 805)
- [ ] มีแผน/runbook สำหรับ key rotation ของ `active_record_encryption` เช่นเดียวกับ credentials
      ทั่วไป (Step 806)
- [ ] Database user ที่แอปใช้เชื่อมต่อมีสิทธิ์แบบ **least privilege** — ไม่ใช่ superuser/admin ของ
      database เสมอไป (เช่น ไม่มีสิทธิ์ `DROP TABLE`/`DROP DATABASE` ถ้าแอปไม่จำเป็นต้องใช้สิทธิ์นั้น
      เลยในโค้ดปกติ, แยก database user ระหว่าง application กับที่ใช้รัน migration ถ้าทำได้)

### หมวดที่ 4 — Transport และ Session

- [ ] `config.force_ssl = true` เปิดอยู่ใน `config/environments/production.rb` (Rails 8 generate ให้
      เป็นค่า default อยู่แล้ว — ตรวจสอบว่าไม่มีใคร comment มันออกไป) บังคับทุก request ผ่าน HTTPS
      และตั้งค่า `Strict-Transport-Security` header ให้อัตโนมัติ
- [ ] Session cookie มี flag ความปลอดภัยครบ — `secure: true` (ถูกเปิดให้อัตโนมัติเมื่อ `force_ssl =
      true`), `httponly: true` (ค่า default ของ Rails session cookie อยู่แล้ว ป้องกัน JavaScript
      ฝั่ง client อ่าน cookie ผ่าน `document.cookie`), และ `same_site` protection (Rails ตั้งค่าให้ตาม
      `config.action_dispatch.cookies_same_site_protection` โดย default เป็น `:lax` ซึ่งป้องกัน CSRF
      ผ่าน cross-site request ได้ระดับหนึ่งอยู่แล้ว)
- [ ] **Secure headers**/Content-Security-Policy ตั้งค่าเหมาะสมกับแอป ไม่ใช้ค่า default ที่กว้างเกินไป
      (Part 080)

### หมวดที่ 5 — Dependency และ Infrastructure

- [ ] `bundle exec bundler-audit check --update` ผ่านโดยไม่มี known vulnerability ที่ยังไม่ patch
      (ผูกเข้า CI ตาม Part 075)
- [ ] Gem ทุกตัวอัปเดตเป็นเวอร์ชันที่ยัง maintain อยู่ (ไม่ใช้ gem ที่ archive/deprecated ไปนานแล้ว)
- [ ] Docker image (ถ้าใช้ตาม Part 073) build จาก base image ที่ patch security อัปเดตล่าสุด ไม่ใช้
      image ที่ค้างเวอร์ชันเก่ามานาน

### Checklist สรุปแบบ Quick-Reference (พิมพ์ติดผนัง หรือใช้เป็น PR template ก่อน deploy)

```markdown
## Pre-Production Security Checklist

### Request/Input Layer
- [ ] Strong Parameters ครบทุก controller
- [ ] CSRF protection เปิดอยู่ (default)
- [ ] ไม่มี raw SQL interpolation จาก user input
- [ ] ไม่มี `raw`/`html_safe` กับ user input ที่ไม่ sanitize
- [ ] `bundle exec brakeman` ผ่านสะอาด
- [ ] Rate limiting เปิดใช้กับ endpoint ที่เสี่ยง

### Secrets/Config
- [ ] `.env`, `*.key` ไม่ถูก track ใน git
- [ ] Production credentials แยกจาก dev/test
- [ ] ไม่มี secret หลุดเข้า log

### Database Encryption
- [ ] ฟิลด์ข้อมูลส่วนบุคคลละเอียดอ่อนถูก encrypts แล้ว
- [ ] Backfill ครบ + ปิด support_unencrypted_data
- [ ] Column type รองรับขนาด ciphertext
- [ ] Database user เป็น least privilege

### Transport/Session
- [ ] config.force_ssl = true
- [ ] Session cookie: secure + httponly + same_site
- [ ] Security headers/CSP ตั้งค่าแล้ว

### Dependencies
- [ ] bundler-audit ผ่านสะอาด
- [ ] Gem/base image อัปเดตล่าสุด
```

---

## Step 809: กระบวนการตรวจสอบรอบสุดท้ายก่อน Launch — Second Pair of Eyes และ Pentest ภายนอก

Checklist ใน Step 808 เป็นสิ่งที่**ทีมเดียวทำเองได้** แต่ประสบการณ์จริงในวงการยืนยันตรงกันว่า **คนที่
เขียนโค้ดเองมักมองข้ามช่องโหว่ในโค้ดของตัวเองได้ง่ายที่สุด** (เพราะรู้ "เจตนา" ของโค้ดอยู่แล้ว จึงไม่คิด
ที่จะลองมุมมองที่ผิดเจตนา) ด้วยเหตุนี้กระบวนการก่อน launch จริงของทีมมืออาชีพจึงมีอย่างน้อยสองชั้น
เพิ่มเติมจาก checklist ที่ทำเองได้

### ชั้นที่ 1 — Second Pair of Eyes (Code Review โดยคนอื่นในทีม)

หลักการเดียวกับที่ Part 097 (Phase 17) จะสอนเรื่อง code review ระดับมืออาชีพ แต่สำหรับบริบทความ
ปลอดภัยโดยเฉพาะ ควรมีคนที่**ไม่ได้เขียนฟีเจอร์นั้นเอง**ตรวจทานอย่างน้อยหนึ่งคนก่อน merge โค้ดที่แตะ:

- Controller action ใหม่ที่รับ input จากผู้ใช้ (โดยเฉพาะ endpoint ที่ไม่ต้อง authenticate)
- โมเดลที่เพิ่มฟิลด์ที่เก็บข้อมูลส่วนบุคคล (ควรถามทุกครั้งว่า "ฟิลด์นี้ต้อง `encrypts` ไหม")
- การเปลี่ยนแปลง authorization logic (policy ของ Pundit/CanCanCan)
- Dependency ใหม่ที่เพิ่มเข้า `Gemfile` (โดยเฉพาะ gem ที่ยังไม่ค่อยมีคนรู้จัก หรือ maintain โดยคนเดียว)

### ชั้นที่ 2 — External Penetration Test สำหรับแอปที่มีความเสี่ยงสูง

สำหรับแอปที่จัดการข้อมูลที่มีผลกระทบรุนแรงถ้ารั่วไหล (ข้อมูลทางการแพทย์, ข้อมูลการเงิน, ข้อมูลเด็ก)
หรือแอปที่ต้องผ่านมาตรฐานกำกับดูแล (compliance) เฉพาะทาง checklist ที่ทำเองไม่เพียงพอ — ควรจ้างทีม
**pentest ภายนอก** ที่ไม่มีความรู้ล่วงหน้าเกี่ยวกับโค้ด (black-box) หรือมีสิทธิ์เข้าถึงบางส่วน
(grey-box) มาทดสอบโจมตีแอปจริงในสภาพแวดล้อมที่แยกจาก production (staging ที่มีข้อมูลจำลอง)

**เกณฑ์คร่าวๆ ว่าแอปควรทำ pentest ภายนอกหรือไม่:**

| เกณฑ์ | ควรทำ pentest ภายนอก |
|---|---|
| จัดการข้อมูลทางการแพทย์/การเงิน/เด็ก | ใช่ชัดเจน |
| มีผู้ใช้งานจำนวนมาก (ผลกระทบถ้ารั่วไหลกว้าง) | ใช่ |
| ต้องผ่าน compliance เฉพาะทาง (เช่น PCI-DSS ถ้าจัดการบัตรเครดิตเอง) | ใช่ — มักเป็นข้อบังคับ |
| แอปภายในองค์กรที่ใช้ไม่กี่คน ไม่มีข้อมูลอ่อนไหว | ไม่จำเป็น (checklist ภายในพอ) |

### กระบวนการ Pre-Launch Review แบบเป็นขั้นตอน

1. รัน checklist ของ Step 808 ให้ผ่านครบทุกข้อก่อน (เป็น baseline ที่ต้องผ่านเสมอ ไม่ว่าแอปเล็กหรือ
   ใหญ่)
2. Code review รอบสุดท้ายโดยคนที่ไม่ได้เขียนฟีเจอร์หลักด้วยตัวเอง โฟกัสเฉพาะจุดที่กระทบความปลอดภัย
   (ไม่ต้อง review ทุกบรรทัดซ้ำ แค่จุดที่ checklist ชี้เป้าไว้)
3. ถ้าเข้าเกณฑ์ตารางด้านบน จ้าง pentest ภายนอก **ก่อน** launch จริง ให้เวลาพอสำหรับแก้ไขปัญหาที่พบ
   (ไม่ใช่ pentest วันเดียวกับที่จะ launch)
4. บันทึกผล checklist + code review + pentest report ไว้เป็นเอกสารอ้างอิง (สำคัญมากถ้าต้อง audit
   ย้อนหลัง หรือเกิดเหตุการณ์ที่ต้องสืบสวนภายหลัง)
5. กำหนดรอบการทำซ้ำ — checklist ควรทำก่อน**ทุก** deploy ใหญ่ (ไม่ใช่แค่ครั้งแรก) ส่วน pentest ภายนอก
   มักทำเป็นรอบ (เช่นทุก 6–12 เดือน หรือก่อนเปิดฟีเจอร์ใหญ่ที่กระทบข้อมูลอ่อนไหวใหม่)

---

## Step 810: แบบฝึกหัดปิด Phase — เข้ารหัสฟิลด์ละเอียดอ่อนจริง พิสูจน์ Raw DB อ่านไม่ออก แล้วไล่
Checklist เต็มรูปแบบ

### โจทย์

สมมติแอป Rails มีโมเดล `Employee` ที่เก็บข้อมูลพนักงาน มีคอลัมน์ `bank_account_number` (เลขบัญชีธนาคาร
สำหรับโอนเงินเดือน) เป็น `string` เก็บเป็น plaintext อยู่แล้ว **และมีข้อมูลพนักงานอยู่แล้ว 3 คน** ให้ทำ
ครบทุกข้อต่อไปนี้:

1. เพิ่ม Active Record Encryption ให้คอลัมน์นี้ โดยไม่ต้อง query/unique ด้วยเลขบัญชีเลย (เลือกโหมด
   ที่ปลอดภัยที่สุด)
2. Backfill ข้อมูลเก่าทั้ง 3 แถวให้กลายเป็น ciphertext อย่างถูกต้อง (ห้ามใช้วิธีที่พิสูจน์แล้วว่าใช้
   ไม่ได้ใน Step 805)
3. พิสูจน์ด้วย raw SQL ว่า raw database อ่านค่าไม่ออกแล้วจริง
4. ไล่ checklist จาก Step 808 กับแอปสมมตินี้ อย่างน้อย 5 ข้อ พร้อมระบุว่าแต่ละข้อ "ผ่าน" หรือ "ต้องแก้
   อะไรเพิ่ม"

### เฉลย

**ขั้นตอนที่ 1 — สร้างกุญแจเข้ารหัส (ถ้ายังไม่มีในแอป)**

```bash
bin/rails db:encryption:init
```

นำผลลัพธ์ไปวางใน credentials:

```bash
EDITOR="code --wait" bin/rails credentials:edit
```

**ขั้นตอนที่ 2 — เปิด `support_unencrypted_data` ชั่วคราว**

```ruby
# config/application.rb
config.active_record.encryption.support_unencrypted_data = true
```

**ขั้นตอนที่ 3 — เพิ่ม `encrypts` ในโมเดล**

```ruby
# app/models/employee.rb
class Employee < ApplicationRecord
  # เลขบัญชีธนาคาร: ไม่มีการ query/unique ด้วยฟิลด์นี้เลยในระบบ
  # → เลือก non-deterministic (ค่า default ของ encrypts) เพื่อความปลอดภัยสูงสุด
  # ไม่ระบุ deterministic: true เลย เพราะไม่จำเป็นต้องแลกความปลอดภัยเพื่อ query
  encrypts :bank_account_number
end
```

**ขั้นตอนที่ 4 — ตรวจสอบขนาดคอลัมน์ก่อน backfill (ทบทวนกับดักจาก Step 805)**

```ruby
Employee.columns_hash["bank_account_number"].limit
# ถ้าได้ผลลัพธ์เป็นตัวเลข (เช่น 255) ที่แคบเกินไป ต้องขยายคอลัมน์ก่อน backfill
```

```ruby
# db/migrate/xxxx_widen_bank_account_number_on_employees.rb (ถ้าจำเป็น)
class WidenBankAccountNumberOnEmployees < ActiveRecord::Migration[8.1]
  def change
    change_column :employees, :bank_account_number, :text
  end
end
```

**ขั้นตอนที่ 5 — Backfill ด้วยวิธีที่ถูกต้อง (ไม่ใช้ `update!(x: x)`)**

```ruby
# lib/tasks/encrypt_employees.rake
namespace :db do
  namespace :encryption do
    desc "Backfill: เข้ารหัส bank_account_number ของ Employee ที่มีอยู่เดิม"
    task backfill_employee_bank_account: :environment do
      count = 0
      Employee.find_each(batch_size: 500) do |employee|
        employee.encrypt
        count += 1
      end
      puts "เข้ารหัสสำเร็จ #{count} รายการ"
    end
  end
end
```

```bash
bin/rails db:encryption:backfill_employee_bank_account
```

**ขั้นตอนที่ 6 — ปิด `support_unencrypted_data` กลับ**

```ruby
# config/application.rb — ลบหรือ comment บรรทัดนี้ออกหลังยืนยันว่า backfill ครบทุกแถวแล้ว
# config.active_record.encryption.support_unencrypted_data = true
```

**ขั้นตอนที่ 7 — พิสูจน์ด้วย raw SQL**

```ruby
ActiveRecord::Base.connection.select_all(
  "SELECT id, name, bank_account_number FROM employees"
).to_a
```

ผลลัพธ์ที่คาดหวัง (รูปแบบเดียวกับที่ทดสอบจริงใน Step 803):

```ruby
{"id"=>1, "name"=>"พนักงาน ก",
 "bank_account_number"=>"{\"p\":\"...\",\"h\":{\"iv\":\"...\",\"at\":\"...\"}}"}
```

ถ้า `bank_account_number` ยังเห็นเป็นตัวเลขบัญชีธรรมดา (ไม่ใช่ JSON) แปลว่า backfill ยังไม่สำเร็จ —
กลับไปตรวจ Step 5 ว่าใช้ `record.encrypt` จริงหรือยังใช้ `update!` แบบที่พิสูจน์แล้วว่าใช้ไม่ได้

**ขั้นตอนที่ 8 — ไล่ Checklist 5 ข้อ กับแอปสมมตินี้**

| ข้อจาก Checklist | ผลตรวจสอบ |
|---|---|
| ฟิลด์ข้อมูลอ่อนไหวถูก `encrypts` แล้ว | **ผ่าน** — `bank_account_number` เข้ารหัสแล้วตาม Step 1–6 |
| คอลัมน์ที่ `encrypts` เป็น `:text` หรือ `limit` กว้างพอ | **ผ่าน** — ตรวจสอบและขยายเป็น `:text` แล้วใน Step 4 |
| Backfill ครบ + ปิด `support_unencrypted_data` | **ผ่าน** — ยืนยันด้วย raw SQL ใน Step 7 แล้วปิด flag ใน Step 6 |
| Database user เป็น least privilege | **ต้องแก้เพิ่ม** — สมมติฐานนี้ยังไม่ได้ตรวจสอบเลยในโจทย์ ต้องไปดู `config/database.yml` และสิทธิ์ของ database user จริงที่ใช้เชื่อมต่อ production |
| ไม่มี secret/ข้อมูลอ่อนไหวหลุดเข้า log | **ต้องแก้เพิ่ม** — ต้อง grep โค้ดทั้งโปรเจกต์หา `Rails.logger`/`puts`/`p` ที่อาจ print `bank_account_number` ตรงๆ (เช่นใน exception handler ที่ log ทั้ง object) |

**สรุปแบบฝึกหัด:** เห็นได้ชัดว่าแม้จะทำ encryption ถูกต้องครบทุกขั้นตอนแล้ว checklist ยังชี้ให้เห็นว่า
มีอีกอย่างน้อย 2 จุด (database user privilege, log hygiene) ที่ต้องตรวจสอบเพิ่มเติมก่อนมั่นใจได้ว่า
ฟีเจอร์นี้ "ปลอดภัยพอสำหรับ production" — นี่คือเหตุผลที่ checklist ต้องครอบคลุมหลายมิติ ไม่ใช่แค่มิติ
เดียว (encryption อย่างเดียวไม่พอ)

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **ทดลอง key rotation จริงกับแอปทดลองของตัวเอง** — สร้างแอป Rails scratch ใหม่ (ลบทิ้งหลังทดลองเสร็จ
   เหมือนที่ทำในหลักสูตรนี้) เปิด Active Record Encryption ให้โมเดลหนึ่งตัว สร้างข้อมูลด้วยกุญแจแรก
   แล้วทำตามขั้นตอน rotate เต็มรูปแบบจาก Step 806 (เพิ่มกุญแจใหม่ → backfill ด้วย `record.encrypt` →
   ถอดกุญแจเก่าออก) แล้วพิสูจน์ด้วยตัวเองว่าข้อมูลเก่าไม่หายและกุญแจเก่าใช้ถอดรหัสไม่ได้จริงหลัง
   ถอดออกจาก config
2. **เขียน RSpec test ที่ยืนยันว่าฟิลด์สำคัญถูกเข้ารหัสจริง** — เขียน test ที่ query raw SQL (ผ่าน
   `ActiveRecord::Base.connection.select_all`) เพื่อยืนยันว่าค่าที่เก็บในคอลัมน์ที่คาดว่าเข้ารหัส
   ไว้แล้ว **ไม่เท่ากับ** plaintext ที่ส่งเข้าไป (ทบทวนรูปแบบการเขียน model spec จาก Part 046) — test
   แบบนี้ป้องกันไม่ให้มีใครในทีมเผลอลบ `encrypts` ออกไปในอนาคตโดยไม่มีใครสังเกตเห็น
3. **ทำ security checklist ของ Step 808 ให้เป็น GitHub Actions job อัตโนมัติ** — ทบทวน Part 075 เรื่อง
   CI/CD แล้วเพิ่ม job ใหม่ที่รัน `bundle exec brakeman --no-pager` และ `bundle exec bundler-audit
   check --update` เป็นส่วนหนึ่งของ pipeline ที่ block การ merge PR ถ้าพบปัญหา (ทำให้ checklist บาง
   ข้อไม่ต้องพึ่งความจำของมนุษย์อีกต่อไป)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจความแตกต่างระหว่าง **secrets ของแอป** (Part 074 — เก็บใน credentials/ENV) กับ **ข้อมูลของ
  ผู้ใช้ที่ต้องเข้ารหัสในฐานข้อมูล** (Part นี้) — เป็นปัญหาคนละชั้น ต้องแก้ด้วยเครื่องมือคนละตัว
- เข้าใจว่า encryption at rest ของ managed database ป้องกันได้แค่การขโมย disk แบบ physical เท่านั้น
  ไม่ป้องกันการอ่านผ่าน query, SQL Injection, หรือ database dump เลย
- ติดตั้งและใช้งาน **Active Record Encryption** ได้จริง — `bin/rails db:encryption:init` สร้างกุญแจ,
  เก็บกุญแจใน Rails credentials, เปิดใช้งานด้วย `encrypts :field_name` บรรทัดเดียว
- พิสูจน์ได้ด้วยตัวเองว่า raw database มองเห็นแค่ ciphertext ขณะที่ Active Record ถอดรหัสให้โปร่งใส
  ผ่าน object ตามปกติ
- เข้าใจความแตกต่างของ **deterministic vs non-deterministic encryption** — เมื่อไหร่ต้องแลกความ
  ปลอดภัยเพื่อให้ query/unique ได้ และความเสี่ยงเรื่อง pattern leakage ที่แลกมา
- รู้วิธี **backfill คอลัมน์ที่มีข้อมูลเดิมอยู่แล้ว** อย่างปลอดภัยด้วย `support_unencrypted_data` และ
  `record.encrypt` พร้อมรู้กับดักสำคัญสองข้อ (`update!(x: x)` ไม่ทำงานเพราะ dirty tracking, ciphertext
  ใหญ่กว่า plaintext มากจนต้องใช้คอลัมน์ `:text`)
- เข้าใจกลไก **key rotation** ของ Active Record Encryption ผ่าน `primary_key` แบบ Array (กุญแจตัว
  สุดท้ายเข้ารหัสข้อมูลใหม่ ทุกกุญแจในลิสต์ใช้ถอดรหัสได้) และขั้นตอน rotate ที่ปลอดภัยในงานจริง
- เข้าใจ **ขอบเขต**ของ Active Record Encryption อย่างชัดเจน — ไม่ป้องกัน application-level compromise,
  ไม่ทดแทน authorization, ไม่ป้องกัน log ที่รั่วไหล, ไม่เกี่ยวข้องกับ HTTPS/TLS
- รวบรวม **Security Checklist ก่อนขึ้น production** ที่ครอบคลุมทั้งหลักสูตร ตั้งแต่ request layer,
  secrets/config, database encryption, transport/session, ไปจนถึง dependency scanning
- เข้าใจความสำคัญของกระบวนการตรวจสอบรอบสุดท้าย — second pair of eyes และ external pentest สำหรับแอป
  ที่มีความเสี่ยงสูง

## สรุปภาพรวม Phase 13: Security

ยินดีด้วย! ตอนนี้ **Phase 13: Security (Part 079–081, Step 781–810)** เสร็จสมบูรณ์แล้ว เราเดินทางจาก
การรู้จักภัยคุกคามพื้นฐานที่สุดของเว็บแอปพลิเคชันไปจนถึงการป้องกันข้อมูลระดับลึกที่สุด:

- **Part 079** พาไปรู้จัก **OWASP Top 10** ในบริบทของ Rails — SQL Injection, Cross-Site Scripting
  (XSS), และ Cross-Site Request Forgery (CSRF) พร้อมกลไกที่ Rails มีป้องกันให้เป็นค่า default อยู่แล้ว
- **Part 080** เจาะลึกลงไปที่ **mass assignment** (ทำไม Strong Parameters ถึงจำเป็น), **secure
  headers** (การตั้งค่า header ที่ป้องกันการโจมตีระดับ browser), และการใช้ **Brakeman** สแกนหาช่องโหว่
  ในโค้ดอัตโนมัติก่อนที่จะกลายเป็นปัญหาจริง
- **Part 081** (Part นี้) ปิดท้ายด้วยการป้องกันข้อมูลในชั้นที่ลึกที่สุด — **Active Record Encryption**
  สำหรับข้อมูลที่ต้องปลอดภัยแม้จากคนที่เข้าถึงฐานข้อมูลโดยตรง และปิดท้ายด้วย **Security Checklist**
  ที่รวบรวมทุกอย่างจากทั้งหลักสูตรเป็นรายการเดียวที่ใช้ได้จริง

สิ่งที่สำคัญที่สุดที่ Phase นี้ทั้งหมดพยายามปลูกฝังไม่ใช่แค่ "เทคนิคการป้องกัน" แต่ละอย่าง แต่คือ
**กรอบความคิดแบบ defense-in-depth** — ไม่มีเครื่องมือตัวเดียวที่ป้องกันได้ทุกอย่าง Strong Parameters
ป้องกัน mass assignment แต่ไม่ป้องกัน XSS, Brakeman จับช่องโหว่ในโค้ดได้แต่ไม่ป้องกัน database ที่ถูก
ขโมยไปตรงๆ, Active Record Encryption ป้องกันข้อมูลจาก database compromise แต่ไม่ป้องกัน application
compromise — ความปลอดภัยที่แท้จริงมาจาก**การซ้อนชั้นการป้องกันหลายชั้นเข้าด้วยกัน** โดยแต่ละชั้นคุม
คนละความเสี่ยง เมื่อชั้นหนึ่งพลาด ชั้นอื่นยังคุ้มกันอยู่

**ต่อไป (Part 082 — เปิด Phase 14: Architecture & Scaling):** เมื่อแอป Rails ผ่านทุกด้าน — ทำงานถูก
ต้อง (Phase 3–4), มี authentication/authorization (Phase 5), ทดสอบครบ (Phase 6), ปลอดภัย (Phase 13
ที่เพิ่งจบไป) — คำถามถัดไปที่นักพัฒนาระดับ Senior ต้องตอบให้ได้คือ **"เมื่อโค้ดโตขึ้นเรื่อยๆ จะจัดระเบียบ
business logic ที่ซับซ้อนอย่างไรไม่ให้ Controller และ Model บวมจนดูแลไม่ไหว"** Part 082 จะเริ่มตอบคำถาม
นี้ด้วยสอง pattern ที่ใช้กันแพร่หลายที่สุดในวงการ Rails มืออาชีพ — **Service Object pattern** (ห่อหุ้ม
business logic ที่ซับซ้อนไว้ในคลาสเดียวที่ทดสอบง่าย) และ **Form Object pattern** (จัดการฟอร์มที่ซับซ้อน
เกินกว่าจะ map ตรงกับ ActiveRecord model เดียว) — จุดเริ่มต้นของ **Phase 14: Architecture & Scaling**
ที่จะพาไปสู่การออกแบบสถาปัตยกรรมระดับองค์กรอย่างแท้จริง
