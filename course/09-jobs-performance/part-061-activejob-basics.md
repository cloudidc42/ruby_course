# Part 061: ActiveJob เบื้องต้น และ Adapter ต่างๆ — เปิด Phase 9: Background Jobs & Performance

> **Step ครอบคลุมใน Part นี้:** Step 601–610
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 025 เรื่อง ActiveRecord เบื้องต้น, Part 041 ที่เคยพูดถึง
> `deliver_later` แบบผิวเผิน, และ Part 018–019 เรื่อง Minitest/RSpec มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4,
> `solid_queue` 1.7.0, `rspec-rails` 8.0.4)

ยินดีต้อนรับสู่ **Phase 9: Background Jobs & Performance** หลังจากใช้เวลาทั้ง Phase 8 อยู่กับ
การตอบสนอง HTTP request แบบ **synchronous** (client ส่ง request มาแล้วรอผลลัพธ์ทันทีจนกว่า
server จะตอบกลับ) ถึงเวลาแล้วที่จะเรียนรู้การทำงานแบบ **asynchronous** — งานบางอย่างไม่ควร
ถูกทำระหว่างที่ผู้ใช้กำลังรอหน้าเว็บโหลดอยู่ Part นี้จะพาไปรู้จัก **ActiveJob** เฟรมเวิร์ก
มาตรฐานของ Rails สำหรับ background job และสิ่งที่สำคัญที่สุดของ Rails 8 ในเรื่องนี้คือ
**Solid Queue** — ระบบคิวงานที่เก็บข้อมูลลงฐานข้อมูลโดยตรง (ไม่ต้องมี Redis) ที่ Rails 8
ติดตั้งให้เป็นค่าเริ่มต้นตั้งแต่ `rails new`

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยการสร้างแอป Rails ชื่อ
> `jobs_demo` ใน `/tmp` (แยกจาก repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้งหลังทดสอบเสร็จ) รัน
> `bin/rails generate job`, `bin/rails generate solid_queue:install`, `bin/jobs` จริง และยืนยัน
> การทำงานแบบ asynchronous ด้วย log จริงจาก `log/development.log` ทุกบรรทัด รวมถึงกรณีที่
> job "หาย" จริงเมื่อใช้ adapter `:async` แล้ว process จบ — ไม่ใช่โค้ดที่เขียนขึ้นลอยๆ

## สารบัญของ Part นี้

- Step 601: ทำไมต้องมี background job — ปัญหาของการทำทุกอย่างแบบ synchronous ใน request
- Step 602: ActiveJob คืออะไร — เฟรมเวิร์กที่ไม่ผูกติดกับ adapter ใดๆ (adapter-agnostic)
- Step 603: Generate job แรก และกายวิภาคของ Job class — `perform`, `queue_as`, `ApplicationJob`
- Step 604: `perform_later` vs `perform_now`, และการส่ง ActiveRecord object เป็น argument
  ผ่าน Global ID (พิสูจน์ว่า job ดึงข้อมูลสดใหม่เสมอ ไม่ใช่ snapshot เก่า)
- Step 605: Adapter แรกที่ควรรู้จัก — `:async` (รันในโปรเซสเดียวกัน เหมาะกับ dev เท่านั้น
  งานหายเมื่อ restart — พิสูจน์จริง)
- Step 606: Solid Queue เจาะลึก — ตั้งค่าจริง ฐานข้อมูลแยก คงทน (durable) ไม่ต้องมี Redis
  รันงานจริงผ่าน `bin/jobs`
- Step 607: Sidekiq และ GoodJob — ภาพรวม adapter อื่นที่ควรรู้จัก (Sidekiq เจาะลึกใน Part 062)
- Step 608: ตั้งค่า adapter (`config.active_job.queue_adapter`), ชื่อคิว และลำดับความสำคัญ
  (`queue_as`, priority)
- Step 609: พื้นฐาน `retry_on` และ `discard_on` (เจาะลึกกว่านี้ใน Part 062)
- Step 610: การทดสอบ job — `perform_enqueued_jobs`, `assert_enqueued_with`, RSpec
  `have_enqueued_job`/`have_performed_job` + แบบฝึกหัดปิด Step 610

---

## Step 601: ทำไมต้องมี background job — ปัญหาของการทำทุกอย่างแบบ synchronous ใน request

### ทบทวน request/response cycle

ทุก HTTP request ที่เข้ามาที่ Rails app จะถูกจัดการโดย thread หนึ่งของ web server (Puma)
ตั้งแต่ controller action เริ่มทำงาน จนกระทั่ง response ถูกส่งกลับไปหา browser **thread นั้น
จะถูกใช้งานอยู่ตลอดเวลา (blocked) จนกว่าทุกบรรทัดโค้ดใน action จะรันเสร็จ** ผู้ใช้ที่กด submit
ฟอร์มจะเห็นหน้าเว็บค้าง (loading spinner) อยู่ตลอดช่วงเวลานั้น

ลองดูตัวอย่างงานที่ "ใช้เวลานาน" ที่เว็บแอปทั่วไปต้องทำบ่อยๆ:

| งาน | เวลาโดยประมาณ | เหตุผลที่ช้า |
|-----|---------------|---------------|
| ส่งอีเมล (SMTP) | 0.5–3 วินาที | ต้องต่อ connection ไปยัง mail server ภายนอก |
| Resize/ประมวลผลรูปภาพ | 1–10 วินาที | ใช้ CPU หนัก โดยเฉพาะรูปขนาดใหญ่หรือหลายขนาด |
| สร้างรายงาน PDF/Excel | 2–30 วินาที | ต้อง query ข้อมูลจำนวนมากแล้ว render เป็นเอกสาร |
| เรียก API ภายนอก (payment, SMS) | ไม่แน่นอน (0.1 วินาที – timeout) | ขึ้นกับ network และความเร็วของฝั่งตรงข้าม |
| ประมวลผล batch ข้อมูลจำนวนมาก | เป็นนาทีถึงชั่วโมง | จำนวน record มาก |

ถ้าโค้ดใน controller เรียกงานเหล่านี้แบบตรงๆ (synchronous) ผู้ใช้ทุกคนที่กด "สมัครสมาชิก"
หรือ "อัปโหลดรูปโปรไฟล์" จะต้องรอจนกว่าอีเมลจะถูกส่งจริง หรือรูปจะถูก resize เสร็จ — แม้ว่า
ผลลัพธ์ของงานเหล่านั้น**ไม่จำเป็นต้องรู้ทันที**ก็ตาม (ผู้ใช้ไม่ต้องรู้ว่าอีเมลถูกส่งไปถึงกล่องจดหมาย
จริงๆ เมื่อไหร่ แค่รู้ว่า "ระบบรับคำขอแล้ว" ก็เพียงพอ)

### ทบทวน Part 041: `deliver_later`

ใน **Part 041 Step 408** ตอนสร้างระบบ Reset Password เราเคยเขียนโค้ดนี้ไปแล้วโดยยังไม่ได้
อธิบายเบื้องหลังแบบเต็มรูปแบบ:

```ruby
# app/controllers/passwords_controller.rb (ทบทวนจาก Part 041)
def create
  if user = User.find_by(email_address: params[:email_address])
    PasswordsMailer.reset(user).deliver_later
  end

  redirect_to new_session_path, notice: "Password reset instructions sent (if user with that email address exists)."
end
```

ตอนนั้นเราอธิบายไว้สั้นๆ ว่า `deliver_later` ส่งอีเมลผ่าน **background job** แทนที่จะบล็อก
request ปัจจุบัน และบอกไว้ว่า **"เรื่อง Active Job และ background job แบบเต็มรูปแบบจะเรียน
ละเอียดใน Part 061"** — นี่คือ Part นั้น เบื้องหลังของ `deliver_later` คือ Action Mailer สร้าง
`ActionMailer::MailDeliveryJob` (ซึ่งเป็น ActiveJob ธรรมดาตัวหนึ่ง) แล้วเรียก `perform_later`
ให้เราโดยอัตโนมัติ — หลักการเดียวกันเป๊ะกับที่ Part นี้กำลังจะสอน

### ภาพเปรียบเทียบ: Synchronous vs Asynchronous

```
แบบ Synchronous (ไม่ดี สำหรับงานที่ใช้เวลานาน):
Browser ──POST /signup──> Controller ──> สร้าง User ──> ส่งอีเมลจริง (รอ 2 วินาที) ──> Response
                                                                                        (รวม 2+ วินาที)

แบบ Asynchronous ด้วย ActiveJob (ดี):
Browser ──POST /signup──> Controller ──> สร้าง User ──> enqueue job (ทันที) ──> Response
                                                              │                  (ไม่ถึง 0.1 วินาที)
                                                              ▼
                                                    Background worker process
                                                    (แยกจาก web server) รับงาน
                                                    ไปส่งอีเมลจริงทีหลัง
```

หลักการสำคัญที่จะพา Part นี้ไปตลอด: **"งานที่ผู้ใช้ไม่จำเป็นต้องเห็นผลลัพธ์ทันที ควรส่งไปทำ
เบื้องหลัง (background) เพื่อให้ request ตอบกลับได้เร็วที่สุดเท่าที่จะทำได้"**

---

## Step 602: ActiveJob คืออะไร — เฟรมเวิร์กที่ไม่ผูกติดกับ adapter ใดๆ (adapter-agnostic)

**ActiveJob** คือเฟรมเวิร์กมาตรฐานของ Rails (เป็นส่วนหนึ่งของ Rails core ตั้งแต่ Rails 4.2)
สำหรับประกาศงานที่จะรันเบื้องหลัง (background job) จุดเด่นที่สำคัญที่สุดคือ **ActiveJob ไม่ได้
เป็นระบบคิวงานเอง แต่เป็น "เลเยอร์นามธรรม" (abstraction layer) ที่คั่นกลางระหว่างโค้ดของเรากับ
ระบบคิวงานจริง (queue backend)**

พูดง่ายๆ คือ: **เราเขียนโค้ด job ครั้งเดียว แล้วสลับ backend เบื้องหลังได้โดยไม่ต้องแก้โค้ด
business logic แม้แต่บรรทัดเดียว** — เหมือนกับที่ ActiveRecord ทำให้เราเขียน Ruby code เดียว
ใช้ได้กับทั้ง PostgreSQL, MySQL, SQLite โดยไม่ต้องเขียน SQL เฉพาะของแต่ละฐานข้อมูล

### Adapter ที่ Rails รองรับ

Rails core มี adapter มาให้พร้อมใช้งานทันที (ไม่ต้องติดตั้ง gem เพิ่ม):

- **`:async`** — รันใน thread pool ภายในโปรเซสเดียวกับ Rails app (ค่า default ของ ActiveJob
  เองถ้าไม่ตั้งค่าอะไรเลย) — รายละเอียดเต็มใน Step 605
- **`:inline`** — รันทันทีแบบ synchronous เหมือนเรียก method ตรงๆ (มีประโยชน์ตอน debug)
- **`:test`** — ใช้อัตโนมัติใน test environment เพื่อดักจับ job ที่ถูก enqueue โดยไม่รันจริง
  (รายละเอียดใน Step 610)

และ adapter ที่ต้องติดตั้ง gem เพิ่มเติม (แต่ใช้ API แบบเดียวกันทุกตัว):

- **`:solid_queue`** — DB-backed, เป็นค่า default ของ Rails 8 (Step 606)
- **`:sidekiq`** — Redis-backed, นิยมมากที่สุดในวงการมานาน (Step 607, เจาะลึกใน Part 062)
- **`:good_job`** — DB-backed อีกตัว (ส่วนใหญ่ใช้กับ PostgreSQL) (Step 607)
- อื่นๆ ที่เก่ากว่าและพบน้อยลงเรื่อยๆ ในโปรเจกต์ใหม่: `:delayed_job`, `:resque`, `:que`,
  `:sneakers`, `:backburner`, `:queue_classic`

### ทำไม "adapter-agnostic" ถึงสำคัญ

ลองนึกภาพทีมที่เริ่มโปรเจกต์เล็กๆ ด้วย `:async` เพราะยังไม่มี production traffic เมื่อโปรเจกต์
โตขึ้นและต้องการความคงทน (durability) สามารถสลับไปใช้ `:solid_queue` ได้โดย**เปลี่ยน config
บรรทัดเดียว** — ไม่ต้องเขียน `SendWelcomeEmailJob` ใหม่ ไม่ต้องแก้ที่เรียก `perform_later` แม้แต่
จุดเดียวในทั้งแอป นี่คือพลังของการออกแบบแบบ abstraction layer ที่ดี — Part นี้จะพิสูจน์เรื่องนี้
ให้เห็นจริงในทางปฏิบัติ

---

## Step 603: Generate job แรก และกายวิภาคของ Job class

### เตรียมแอปทดสอบ

```bash
rails new jobs_demo -d sqlite3
cd jobs_demo
```

### สร้าง Job ด้วย generator

```bash
bin/rails generate job SendWelcomeEmail
```

ผลลัพธ์จริง:

```
      invoke  test_unit
      create    test/jobs/send_welcome_email_job_test.rb
      create  app/jobs/send_welcome_email_job.rb
```

ไฟล์ `app/jobs/send_welcome_email_job.rb` ที่ generate มา:

```ruby
class SendWelcomeEmailJob < ApplicationJob
  queue_as :default

  def perform(*args)
    # Do something later
  end
end
```

### กายวิภาคของ Job class

- **`ApplicationJob`** — base class กลางของ job ทุกตัวในแอป (คล้าย `ApplicationRecord` สำหรับ
  model, `ApplicationController` สำหรับ controller) ดูเนื้อหาข้างในที่ Rails generate ให้ตอน
  `rails new`:

  ```ruby
  # app/jobs/application_job.rb
  class ApplicationJob < ActiveJob::Base
    # Automatically retry jobs that encountered a deadlock
    # retry_on ActiveRecord::Deadlocked

    # Most jobs are safe to ignore if the underlying records are no longer available
    # discard_on ActiveJob::DeserializationError
  end
  ```

  สังเกตว่า Rails **คอมเมนต์ตัวอย่าง `retry_on`/`discard_on` ที่พบบ่อยที่สุดไว้ให้เลย** —
  บอกใบ้ชัดเจนว่านี่คือสิ่งที่ทีม Rails core แนะนำให้ทุกโปรเจกต์พิจารณาใช้ (รายละเอียดเต็มใน
  Step 609)

- **`queue_as :default`** — กำหนดชื่อคิวที่ job ตัวนี้จะถูกส่งไป (ค่า default คือ `"default"`
  ถ้าไม่เรียก) ใช้แยกงานตามความสำคัญ/ประเภท (รายละเอียดเต็มใน Step 608)

- **`def perform(*args)`** — method หลักเพียงตัวเดียวที่**ต้อง**เขียน คือโค้ดจริงที่จะรันเมื่อ
  job ถูกประมวลผล รับ argument ได้ตามต้องการ (`*args` คือ splat ที่ generator ใส่มาให้เป็น
  ค่าเริ่มต้น เราสามารถเปลี่ยนเป็น parameter ที่ชัดเจนได้ตามต้องการ)

### เตรียม Model สำหรับทดลอง

สร้าง model `Contact` ไว้เป็นตัวอย่างสำหรับส่ง ActiveRecord object เข้า job (Step 604):

```bash
bin/rails generate model Contact name:string email:string
bin/rails db:prepare
```

### เขียน logic จริงลงใน job

```ruby
# app/jobs/send_welcome_email_job.rb
class SendWelcomeEmailJob < ApplicationJob
  queue_as :default

  def perform(contact)
    sleep 2 # จำลอง network latency ตอนต่อ SMTP server จริง
    Rails.logger.info "[SendWelcomeEmailJob] ส่งอีเมลต้อนรับให้ #{contact.name} <#{contact.email}>"
  end
end
```

(ในโปรเจกต์จริง บรรทัด `sleep 2` จะถูกแทนที่ด้วย `WelcomeMailer.welcome(contact).deliver_now`
— ใช้ `deliver_now` ไม่ใช่ `deliver_later` เพราะ job ตัวนี้เอง**คือ**ตัวที่ทำงานเบื้องหลังอยู่แล้ว
ไม่จำเป็นต้อง enqueue job ซ้อน job อีกชั้น)

---

## Step 604: `perform_later` vs `perform_now` และการส่ง ActiveRecord object ผ่าน Global ID

### `perform_later` — เข้าคิว ไม่รอผล

```ruby
contact = Contact.create!(name: "สมชาย", email: "somchai@example.com")
SendWelcomeEmailJob.perform_later(contact)
```

`perform_later` จะ**เข้าคิวทันที**และคืนค่ากลับมาแทบจะในทันที (ไม่รอให้ `perform` รันเสร็จ)
นี่คือ method ที่ควรใช้เกือบทุกครั้งใน controller action หรือที่อื่นๆ ในโค้ด "จริง"

### `perform_now` — รันทันที แบบ synchronous บล็อก thread

```ruby
t0 = Time.now
SendWelcomeEmailJob.perform_now(contact)
puts "ใช้เวลา #{(Time.now - t0).round(2)} วินาที"
```

ผลลัพธ์จริงจากการรัน:

```
ใช้เวลา 2.01 วินาที (บล็อก thread หลักจริง)
```

`perform_now` **ไม่ผ่านคิวเลย** เรียก `perform` ตรงๆ ทันที เหมือนเรียก method ธรรมดา — มี
ประโยชน์ตอน debug หรือทดสอบ logic ของ job แบบเร็วๆ แต่**ไม่ควรใช้ใน production code** ที่ต้องการ
ประโยชน์จาก background processing เพราะมันเสียจุดประสงค์ทั้งหมดของ ActiveJob ไปเลย (บล็อก
request เหมือนเดิม)

> **จำง่ายๆ:** `perform_later` = "ทำทีหลัง" (asynchronous), `perform_now` = "ทำเดี๋ยวนี้"
> (synchronous) — ชื่อ method บอกพฤติกรรมตรงตัวอยู่แล้ว

### การส่ง ActiveRecord object เป็น argument — Global ID

จุดที่มือใหม่มักสงสัยคือ: เวลาส่ง `contact` (ActiveRecord object) เข้า `perform_later` แล้ว
job ไปรันในอีก process หนึ่ง (คนละ process กับที่ enqueue) มันส่ง object ข้าม process ไปได้
อย่างไร คำตอบคือ **ActiveJob ไม่ได้ serialize ทั้ง object เก็บไว้ แต่ใช้ระบบที่ชื่อว่า
`GlobalID` แปลง record เป็น "ที่อยู่อ้างอิง" (URI) แทน**

พิสูจน์ได้ด้วยการดู argument ที่ถูก serialize จริงก่อนเข้าคิว:

```ruby
job = SendWelcomeEmailJob.new(contact)
puts job.serialize.inspect
```

ผลลัพธ์จริง:

```
{"job_class"=>"SendWelcomeEmailJob", "job_id"=>"4f9eded6-3b77-4dea-94a1-64192cf6520a",
 "provider_job_id"=>nil, "queue_name"=>"default", "priority"=>nil,
 "arguments"=>[{"_aj_globalid"=>"gid://jobs-demo/Contact/1"}],
 "executions"=>0, "exception_executions"=>{}, "locale"=>"en", "timezone"=>"UTC",
 "enqueued_at"=>"2026-09-26T08:10:30.952788938Z", "scheduled_at"=>nil}
```

สังเกต `"arguments"=>[{"_aj_globalid"=>"gid://jobs-demo/Contact/1"}]` — **สิ่งที่ถูกเก็บลงคิวจริงๆ
คือ string `"gid://jobs-demo/Contact/1"` ไม่ใช่ hash ของ attribute ทั้งหมดของ contact**
รูปแบบ `gid://<app-name>/<ClassName>/<id>` นี้คือ Global ID — เมื่อ worker หยิบ job นี้ไปรัน
มันจะ**สั่ง `Contact.find(1)` ใหม่จากฐานข้อมูลอีกครั้ง** ก่อนส่งเข้า `perform`

### ทำไมเรื่องนี้ถึงสำคัญมาก: ป้องกันข้อมูลเก่า (stale data)

เพราะ job ดึงข้อมูล**สดใหม่จากฐานข้อมูล**ทุกครั้งที่รัน (ไม่ใช่ snapshot ตอน enqueue) ถ้า record
มีการเปลี่ยนแปลงระหว่างที่รอคิวอยู่ ผลลัพธ์ที่ job เห็นจะเป็นค่าล่าสุดเสมอ มาพิสูจน์กันจริง:

```ruby
contact = Contact.find_by(name: "สมชาย")
SendWelcomeEmailJob.set(wait: 8.seconds).perform_later(contact)   # ตั้งให้รันใน 8 วินาทีข้างหน้า
contact.update!(name: "สมชาย (เปลี่ยนชื่อแล้ว)")                   # เปลี่ยนชื่อทันทีหลัง enqueue
```

รอให้ worker (Solid Queue จาก Step 606) รันงานนี้ แล้วดู log จริงที่ได้:

```
[ActiveJob] [SendWelcomeEmailJob] [...] Performing SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) enqueued at 2026-09-26T08:10:45... with arguments:
  #<GlobalID:0x... @uri=#<URI::GID gid://jobs-demo/Contact/1>>
[ActiveJob] [SendWelcomeEmailJob] [...] [SendWelcomeEmailJob] ส่งอีเมลต้อนรับให้
  สมชาย (เปลี่ยนชื่อแล้ว) <somchai@example.com>
[ActiveJob] [SendWelcomeEmailJob] [...] Performed SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) in 2048.78ms
```

**ชื่อที่ log ออกมาคือ "สมชาย (เปลี่ยนชื่อแล้ว)"** — ค่าที่ถูก `update!` หลังจาก enqueue ไปแล้ว
ไม่ใช่ "สมชาย" ที่เป็นค่าตอน enqueue นี่คือหลักฐานชัดเจนว่า **argument ที่เป็น ActiveRecord
object จะถูกดึงข้อมูลใหม่เสมอตอน job รันจริง ไม่ใช่ค่าที่ถูกแช่แข็งไว้ตอน enqueue**

> **ข้อควรระวังที่ตามมา:** ถ้า record ถูก**ลบ**ไปก่อนที่ job จะรัน การ deserialize Global ID
> จะหา record ไม่เจอและ raise `ActiveJob::DeserializationError` — เรื่องนี้จะพิสูจน์และแก้ด้วย
> `discard_on` ใน Step 609

---

## Step 605: Adapter แรกที่ควรรู้จัก — `:async` (รันในโปรเซสเดียวกัน เหมาะกับ dev เท่านั้น)

### `:async` คืออะไร

**`:async`** คือค่า default ของ ActiveJob เอง (ที่ framework level) เมื่อไม่มีการตั้งค่า
`queue_adapter` ใดๆ เลย มันทำงานโดยใช้ **thread pool ของ gem `concurrent-ruby`** ที่อยู่
**ภายในโปรเซส Ruby เดียวกัน**กับ Rails app — ไม่มีคิวงานแยกต่างหาก ไม่มีฐานข้อมูลเก็บสถานะ
ไม่ต้องติดตั้งอะไรเพิ่มเลย ใช้งานได้ทันทีตั้งแต่ `rails new`

พิสูจน์ว่า Rails 8 app ใหม่เอี่ยม (ที่ยังไม่ตั้งค่าอะไร) ใช้ `:async` ใน development จริง:

```bash
bin/rails runner 'puts ActiveJob::Base.queue_adapter.class'
# => ActiveJob::QueueAdapters::AsyncAdapter
```

### ข้อจำกัดสำคัญ: ไม่คงทน (not durable) — งานหายเมื่อ process จบ

เพราะ job ที่ enqueue ด้วย `:async` อยู่แค่ใน**หน่วยความจำ (memory)** ของโปรเซสนั้น ถ้าโปรเซส
จบการทำงาน (deploy ใหม่, server restart, หรือแค่ปิด process) **job ที่ยังไม่ได้รันจะหายไปทันที
โดยไม่มี error ใดๆ แจ้งเตือนเลย** มาพิสูจน์กันจริง:

```ruby
# รันผ่าน `bin/rails runner` แล้ว process จะจบทันทีหลัง script รันเสร็จ
ActiveJob::Base.queue_adapter = :async
contact = Contact.first
SendWelcomeEmailJob.set(wait: 30.seconds).perform_later(contact)
puts "enqueue งานแบบ :async ให้รันใน 30 วินาที แต่โปรแกรม (process) นี้กำลังจะจบเดี๋ยวนี้..."
```

log ที่เกิดขึ้นตอน enqueue (แสดงว่า enqueue สำเร็จจริง):

```
[ActiveJob] Enqueued SendWelcomeEmailJob (Job ID: fdf526f9-...) to Async(default)
  at 2026-09-26 08:11:50 UTC with arguments: #<GlobalID:... gid://jobs-demo/Contact/1>
```

จากนั้นรอ **32 วินาที** (เกิน 30 วินาทีที่ตั้งไว้) แล้วเช็ค log อีกครั้ง — **ไม่มีบรรทัด
"Performing"/"Performed" ปรากฏขึ้นมาเลย** เพราะ `bin/rails runner` เป็น process ที่จบตัวเอง
ทันทีหลัง script รันจบ (ก่อนถึงเวลา 30 วินาที) thread pool ที่ถือ job นั้นไว้ก็ถูกทำลายไปพร้อม
กับ process — **job หายไปตลอดกาล ไม่มีทางกู้คืน**

> **สรุป `:async`:** เหมาะสำหรับ **development และ test เท่านั้น** ที่ไม่สนใจเรื่องความคงทน
> ของข้อมูล **ห้ามใช้ใน production เด็ดขาด** เพราะ deploy ครั้งถัดไป (ซึ่งเกิดขึ้นบ่อยมากใน
> โปรเจกต์จริง) จะทำให้ job ที่ค้างอยู่ในคิวหายไปหมด — อีเมลที่ควรส่งไม่ถูกส่ง รายงานที่ควร
> สร้างไม่ถูกสร้าง โดยไม่มีใครรู้ตัว

### สิ่งที่ Rails 8 ทำเรื่องนี้แตกต่างจากเดิม

Rails เวอร์ชันก่อนหน้า (7 และเก่ากว่า) ปล่อยให้ทีมพัฒนาต้องหา solution เอง (ส่วนใหญ่คือติดตั้ง
Sidekiq + Redis) **Rails 8 เปลี่ยนเกมด้วยการติดตั้ง Solid Queue มาให้ตั้งแต่ `rails new` และตั้งค่า
เป็น production default ให้อัตโนมัติ** — รายละเอียดเต็มรออยู่ใน Step 606

---

## Step 606: Solid Queue เจาะลึก — DB-backed, คงทน, ไม่ต้องมี Redis

### Solid Queue คืออะไร

**Solid Queue** เป็น gem ที่ DHH และทีม Rails core เขียนขึ้น เปิดตัวพร้อม Rails 8 หลักการ
คือ **เก็บงานทุกงานเป็น row ในตารางฐานข้อมูล** (ใช้ฐานข้อมูลเดียวกับที่แอปใช้อยู่แล้ว หรือแยก
ฐานข้อมูลก็ได้) แทนที่จะเก็บในหน่วยความจำหรือต้องพึ่ง Redis เป็นส่วนหนึ่งของ **"Solid Trifecta"**
— สามระบบที่มาแทนที่สิ่งที่เคยต้องใช้ Redis เกือบทุกโปรเจกต์:

| ระบบเดิม (ต้องมี Redis) | ระบบใหม่ของ Rails 8 (DB-backed) | ใช้ทำอะไร |
|---|---|---|
| Sidekiq/Resque | **Solid Queue** | Background job |
| Rails.cache ผ่าน Redis | **Solid Cache** | Low-level caching |
| Redis pub/sub | **Solid Cable** | Action Cable (WebSocket) |

### พิสูจน์ว่า Solid Queue เป็นค่า default ตั้งแต่ `rails new`

```bash
grep -n "solid_queue" Gemfile
# => gem "solid_queue"
```

ทุกแอป Rails 8 ที่สร้างด้วย `rails new` (โหมด default ไม่ได้เลือก `--skip-solid`) จะมี
`gem "solid_queue"` อยู่ใน Gemfile ให้อัตโนมัติ พร้อมไฟล์ config ที่ generator ของมันสร้างให้
(`bin/rails generate solid_queue:install` ถูกเรียกอัตโนมัติตอน `rails new` ผ่าน after-bundle
hook):

```bash
bin/rails generate solid_queue:install
```

ผลลัพธ์จริง:

```
      create  config/queue.yml
      create  config/recurring.yml
      create  db/queue_schema.rb
      create  bin/jobs
        gsub  config/environments/production.rb
```

และไฟล์ `bin/jobs` ที่ได้มา (คือคำสั่งที่ใช้รัน worker):

```ruby
#!/usr/bin/env ruby

require_relative "../config/environment"
require "solid_queue/cli"

SolidQueue::Cli.start(ARGV)
```

### สิ่งสำคัญที่ต้องรู้: default ของ Rails 8 คือ Solid Queue **เฉพาะ production**

ตรวจดู `config/environments/production.rb` หลังรัน generator:

```ruby
# Replace the default in-process and non-durable queuing backend for Active Job.
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }
```

**บรรทัดนี้ถูกใส่ไว้ให้อัตโนมัติเฉพาะใน `production.rb` เท่านั้น** — ถ้าเปิดดู
`config/environments/development.rb` และ `test.rb` จะไม่พบบรรทัด `queue_adapter` เลย ซึ่งหมายความ
ว่า **ใน development เปล่าๆ Rails 8 app ยังคงใช้ `:async` ตามค่า default ของ ActiveJob เอง**
(ตามที่พิสูจน์ไปแล้วใน Step 605) — Solid Queue "ติดตั้งมาให้พร้อม" (gem อยู่ใน Gemfile, ไฟล์
config ถูกสร้างไว้) แต่ **ยังไม่ถูก "เปิดใช้งาน" ใน development จนกว่าเราจะตั้งค่าเอง**

> **ทำไม Rails ถึงออกแบบมาแบบนี้:** ใน development เรามักรีสตาร์ท server บ่อยมาก (แก้โค้ด,
> restart), การใช้ `:async` (เร็ว ไม่ต้องรอ query DB) ก็เพียงพอสำหรับการพัฒนาปกติ ส่วนใน
> production ที่ต้องการความคงทนจริงจัง Rails จึงเลือก Solid Queue ให้อัตโนมัติ อย่างไรก็ตาม
> การใช้ Solid Queue ใน development ด้วยก็เป็นแนวทางที่แนะนำมากขึ้นเรื่อยๆ เพราะพฤติกรรมจะ
> เหมือน production ทุกประการ (ไม่มี surprise ตอน deploy จริง) — Step ถัดไปจะตั้งค่าให้ครบ

### ตั้งค่า Solid Queue ให้ใช้งานได้ใน development ด้วย

**1) เพิ่ม config ใน `config/environments/development.rb`:**

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ... โค้ดเดิม ...

  config.active_job.queue_adapter = :solid_queue
  config.solid_queue.connects_to = { database: { writing: :queue } }
end
```

**2) เพิ่ม database role `queue` ให้ development ใน `config/database.yml`** (เลียนแบบโครงสร้าง
ที่ `production` มีอยู่แล้ว):

```yaml
development:
  primary:
    <<: *default
    database: storage/development.sqlite3
  queue:
    <<: *default
    database: storage/development_queue.sqlite3
    migrations_paths: db/queue_migrate
```

**3) เตรียมฐานข้อมูล:**

```bash
bin/rails db:prepare
```

ตรวจสอบว่าตาราง Solid Queue ถูกสร้างขึ้นจริงในฐานข้อมูล `queue` แยกต่างหาก (ไม่ปนกับตาราง
ของแอป):

```
storage/development.sqlite3        → contacts, schema_migrations, ...
storage/development_queue.sqlite3  → solid_queue_jobs, solid_queue_ready_executions,
                                      solid_queue_claimed_executions, solid_queue_failed_executions,
                                      solid_queue_scheduled_executions, solid_queue_processes, ...
```

### รัน worker จริงด้วย `bin/jobs`

```bash
bin/jobs
```

log จริงตอน worker เริ่มทำงาน:

```
SolidQueue-1.7.0 Register Supervisor(fork)  pid: 32549, hostname: "vm", ...
SolidQueue-1.7.0 Started Supervisor(fork)   pid: 32549, ...
SolidQueue-1.7.0 Register Dispatcher  pid: 32558, name: "dispatcher-..."
SolidQueue-1.7.0 Started Dispatcher   polling_interval: 1, batch_size: 500, ...
SolidQueue-1.7.0 Register Worker  pid: 32562, name: "worker-..."
SolidQueue-1.7.0 Started Worker   polling_interval: 1, queues: "*", pool_size: 3
```

โครงสร้างของ Solid Queue มี 3 ส่วนหลักที่ทำงานร่วมกันภายใต้ **Supervisor** เดียว:

- **Dispatcher** — คอยเช็คงานที่ตั้งเวลาไว้ล่วงหน้า (scheduled jobs) แล้วย้ายเข้าคิวจริงเมื่อ
  ถึงเวลา
- **Worker** — ดึงงานจากคิวมารันจริง (มี thread pool ของตัวเอง ตามที่ตั้งค่าใน `queue.yml`)

เมื่อ enqueue job ตามปกติ (`SendWelcomeEmailJob.perform_later(contact)`) แล้วปล่อยให้
`bin/jobs` ทำงานอยู่ จะเห็น log แบบเดียวกับที่พิสูจน์ไปแล้วใน Step 604 — งานถูกรันจริงโดย
worker ที่แยก process ออกจาก web server อย่างสมบูรณ์

ทางเลือกอื่นในการรัน Solid Queue (ทำงานเหมือนกันทุกประการ):

```bash
bin/rails solid_queue:start

# ตรวจสอบ config ว่าถูกต้องโดยไม่เริ่มรันจริง (มีประโยชน์ก่อน deploy)
bin/rails solid_queue:check
# => Solid Queue configuration is valid.
```

### พิสูจน์ความคงทน (durability) — จุดต่างที่สำคัญที่สุดเทียบกับ `:async`

Enqueue งานที่ตั้งเวลาไว้ล่วงหน้า **โดยไม่มี worker ใดๆ รันอยู่เลย**:

```ruby
contact = Contact.first
SendWelcomeEmailJob.set(wait: 15.seconds).perform_later(contact)
```

ตรวจสอบตรงๆ ในฐานข้อมูล (ไม่ผ่าน Rails เลย) ว่างานนี้ถูกบันทึกเป็น row จริง:

```ruby
# ตรวจสอบ storage/development_queue.sqlite3 โดยตรง
[{"id"=>2, "class_name"=>"SendWelcomeEmailJob",
  "scheduled_at"=>"2026-09-26 08:12:16.639019", "finished_at"=>nil}]
```

**นี่คือหลักฐานสำคัญที่สุดของ Part นี้:** job อยู่เป็น **row จริงในตาราง `solid_queue_jobs`**
พร้อม `finished_at` เป็น `nil` (ยังไม่เสร็จ) แม้ว่าจะไม่มี worker process ใดๆ รันอยู่เลยในขณะนั้น
ก็ตาม เทียบกับ `:async` ที่ต้องพึ่งพา thread pool ในหน่วยความจำที่หายไปทันทีเมื่อ process จบ —
Solid Queue **ไม่สนใจว่า worker จะรันอยู่หรือไม่ ข้อมูลปลอดภัยอยู่ในฐานข้อมูลเสมอ** ต่อให้
server ทั้งเครื่อง reboot ก็ตาม เมื่อ `bin/jobs` เริ่มทำงานอีกครั้ง (ไม่ว่าจะช้าไปกี่นาทีก็ตาม)
มันจะไปหยิบงานที่ค้างอยู่มารันต่อทันที

รัน `bin/jobs` อีกครั้งแล้วดูผล — job ที่ scheduled ไว้ก่อนหน้าถูกรันสำเร็จจริงเมื่อถึงเวลา:

```
[ActiveJob] [SendWelcomeEmailJob] [...] Performing SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) with arguments: #<GlobalID:... gid://jobs-demo/Contact/1>
[ActiveJob] [SendWelcomeEmailJob] [...] Performed SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) in 2000.97ms
```

---

## Step 607: Sidekiq และ GoodJob — ภาพรวม adapter อื่นที่ควรรู้จัก

### Sidekiq — ตัวเลือกที่ครองตลาดมานานที่สุด

**Sidekiq** เป็น background job library ที่ไม่ได้มาพร้อม Rails (ต้องติดตั้ง gem `sidekiq`
เพิ่มเอง) ใช้ **Redis เป็นที่เก็บคิวงาน** จุดเด่นที่ทำให้ Sidekiq เป็นมาตรฐานของวงการ Ruby/Rails
มานานกว่า 10 ปี:

- **เร็วมาก** — Redis เป็น in-memory data store การอ่าน/เขียนคิวเร็วกว่า query ฐานข้อมูล
  เชิงสัมพันธ์อย่างมาก
- **Sidekiq Web UI** — dashboard สำเร็จรูปสวยงาม ดู queue size, retry, failed jobs, scheduled
  jobs ได้แบบ real-time ผ่านเบราว์เซอร์
- **Ecosystem ใหญ่มาก** — ผ่านการใช้งานจริงในโปรเจกต์ระดับ enterprise นับไม่ถ้วน มี
  Sidekiq Pro/Enterprise สำหรับฟีเจอร์ระดับสูง (batches, reliable push, rate limiting)
- **ข้อเสีย** — ต้องมี **Redis server แยกต่างหาก** ให้ดูแล (deploy, monitor, backup) ซึ่งเป็น
  moving part เพิ่มเติมที่ Solid Queue ไม่ต้องมี

เพราะ Sidekiq ยังคงเป็น adapter ที่พบเจอบ่อยมากในโปรเจกต์จริง (โดยเฉพาะโปรเจกต์เก่าที่สร้างก่อน
Rails 8 และโปรเจกต์ที่ต้องการ throughput สูงมากๆ) **Part 062 จะเจาะลึก Sidekiq แบบเต็มรูปแบบ**
ทั้งการติดตั้ง, queue priority, retry strategy, และ scheduled job

### GoodJob — อีกทางเลือกแบบ DB-backed

**GoodJob** เป็น gem DB-backed อีกตัวหนึ่ง (เขียนโดย Ben Sheldon) ที่เปิดตัวมาก่อน Solid Queue
หลายปี และเป็นหนึ่งในแรงบันดาลใจสำคัญที่ทีม Rails core ใช้ออกแบบ Solid Queue (แม้แต่ฟีเจอร์
recurring/cron job ของ Solid Queue ก็ยอมรับตรงๆ ใน README ว่า
["inspired by what GoodJob does"](https://github.com/rails/solid_queue)) จุดเด่นของ GoodJob:

- ทำงานได้ดีที่สุดกับ **PostgreSQL** (ใช้ `LISTEN/NOTIFY` และ advisory lock ของ Postgres
  เพื่อความเร็วในการดึงงาน)
- มี **web dashboard ในตัว** (`GoodJob::Engine`) ที่ครบเครื่องกว่า Solid Queue ในบางแง่ ณ
  ปัจจุบัน (เช่น การดู job graph, cron schedule ทั้งหมด)
- เหมาะกับทีมที่ใช้ PostgreSQL อยู่แล้วและต้องการ dashboard สำเร็จรูปโดยไม่ต้องพึ่ง Redis

โปรเจกต์ Rails 8 ใหม่ส่วนใหญ่จะเลือก **Solid Queue** เป็นค่าเริ่มต้นเพราะมันมากับ Rails ให้
อัตโนมัติแล้ว แต่การรู้จัก GoodJob ไว้ก็มีประโยชน์เพราะยังพบได้บ่อยในโปรเจกต์ Rails 7 ที่สร้าง
ก่อน Solid Queue จะถือกำเนิด

### ตารางเปรียบเทียบสรุป

| Adapter | เก็บข้อมูลที่ไหน | ต้องมี service แยก | คงทน (durable) | Default ของ Rails 8 |
|---|---|---|---|---|
| `:async` | Memory (thread pool) | ไม่ต้อง | ❌ ไม่ | ❌ (เฉพาะ dev/test ที่ไม่ตั้งค่าอะไร) |
| `:inline` | ไม่มีคิวเลย รันทันที | ไม่ต้อง | - | ❌ |
| `:solid_queue` | Database (ฐานเดียวกับแอปหรือแยกก็ได้) | ไม่ต้อง | ✅ ใช่ | ✅ (production) |
| `:sidekiq` | Redis | ✅ ต้องมี Redis | ✅ ใช่ | ❌ |
| `:good_job` | Database (เน้น PostgreSQL) | ไม่ต้อง | ✅ ใช่ | ❌ |

---

## Step 608: ตั้งค่า adapter, ชื่อคิว และลำดับความสำคัญ

### ตั้งค่า adapter

ตั้งค่าแบบ **global** (ทุก environment) ใน `config/application.rb`:

```ruby
# config/application.rb
module JobsDemo
  class Application < Rails::Application
    config.active_job.queue_adapter = :solid_queue
  end
end
```

หรือตั้งค่าแบบ **เฉพาะ environment** ใน `config/environments/production.rb`,
`development.rb`, `test.rb` (ค่าที่ตั้งใน environment file จะ override ค่าที่ตั้งใน
`application.rb` เสมอ) — วิธีนี้คือวิธีที่ Rails 8 ใช้เองตามที่เห็นใน Step 606

```ruby
# ตัวอย่างสลับไปใช้ Sidekiq (หลังติดตั้ง gem "sidekiq" — รายละเอียดใน Part 062)
config.active_job.queue_adapter = :sidekiq
```

สามารถเปลี่ยน adapter runtime ได้ด้วย (มีประโยชน์ตอนเขียน script ทดสอบใน `rails runner`
อย่างที่ทำมาตลอด Part นี้):

```ruby
ActiveJob::Base.queue_adapter = :inline
```

### ชื่อคิว (`queue_as`)

```ruby
class ImageResizeJob < ApplicationJob
  queue_as :high_priority

  def perform(contact)
    sleep 3
    Rails.logger.info "[ImageResizeJob] resize รูปโปรไฟล์ของ #{contact.name} เสร็จแล้ว"
  end
end
```

รัน job นี้แล้วดู log จริง — สังเกตชื่อคิวที่ปรากฏ:

```
[ActiveJob] [ImageResizeJob] [...] Performing ImageResizeJob (Job ID: ...)
  from SolidQueue(high_priority) enqueued at ... with arguments: #<GlobalID:...>
[ActiveJob] [ImageResizeJob] [...] [ImageResizeJob] resize รูปโปรไฟล์ของ ทดสอบ Controller เสร็จแล้ว
[ActiveJob] [ImageResizeJob] [...] Performed ImageResizeJob (Job ID: ...)
  from SolidQueue(high_priority) in 3054.06ms
```

`from SolidQueue(high_priority)` ยืนยันว่า job ตัวนี้ถูกส่งเข้าคิวชื่อ `high_priority` จริง
(แยกจากคิว `default` ที่ `SendWelcomeEmailJob` ใช้) โดยที่**ไม่ต้องตั้งค่าอะไรเพิ่มใน
`queue.yml` เลย** เพราะค่า default ของ worker คือ `queues: "*"` (รับงานจากทุกคิว)

`queue_as` ยังรองรับการกำหนดคิวแบบ dynamic ได้ด้วย block:

```ruby
class NotifyJob < ApplicationJob
  queue_as do
    contact = arguments.first
    contact.vip? ? :high_priority : :default
  end
end
```

### กำหนด worker pool แยกตามคิวใน `queue.yml`

ไฟล์ `config/queue.yml` ที่ Solid Queue สร้างให้ควบคุมว่า **worker process/thread ไหนดูแล
คิวไหนบ้าง** — ตัวอย่างการแยก worker pool สำหรับคิวสำคัญออกจากคิวทั่วไป (ปรับจาก README ของ
`solid_queue`):

```yaml
# config/queue.yml
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 3
      polling_interval: 1
    - queues: [ high_priority ]
      threads: 5
      polling_interval: 0.1
      processes: 2
```

หลักการสำคัญที่ README ของ `solid_queue` (เวอร์ชัน 1.7.0) เน้นย้ำไว้:

> "ถ้าระบุคิวเป็น list เช่น `real_time,background` งานจะถูกดึงตามลำดับที่ระบุ ไม่มีงานไหนจาก
> `background` ถูกหยิบเลยถ้ายังมีงานเหลือใน `real_time`"
>
> "ActiveJob รองรับ priority เป็นตัวเลขบวกด้วย ยิ่งค่าน้อยยิ่งสำคัญมาก (default คือ `0`) แต่
> **ลำดับของคิวจะมีผลเหนือกว่า priority เสมอ** — เราแนะนำว่าอย่าผสมทั้งสองแบบเข้าด้วยกัน
> ให้เลือกใช้อย่างใดอย่างหนึ่ง"

ตั้งค่า priority ระดับ job ได้ผ่าน `.set`:

```ruby
SendWelcomeEmailJob.set(priority: 10).perform_later(contact)   # priority เริ่มต้นคือ 0 (สำคัญสุด)
```

> **ข้อควรระวัง:** ความหมายของตัวเลข priority (น้อย=สำคัญ หรือ มาก=สำคัญ) และวิธีตั้งค่า
> **แตกต่างกันไปในแต่ละ adapter** — Sidekiq ใช้แนวคิดคนละแบบ (priority ต่อคิว ไม่ใช่ต่อ job)
> รายละเอียดเปรียบเทียบเต็มรูปแบบรอไว้ใน Part 062

---

## Step 609: พื้นฐาน `retry_on` และ `discard_on`

งานเบื้องหลังมักเจอ error ที่**เกิดขึ้นชั่วคราว** (network timeout, deadlock ชั่วขณะ) ซึ่งควร
**ลองใหม่อัตโนมัติ** ต่างจาก error ที่**ไม่มีทางสำเร็จได้อีก** (record ที่อ้างอิงถูกลบไปแล้ว)
ซึ่งควร**เลิกทำไปเลย (discard)** โดยไม่ต้องแจ้งเตือนเป็น failure

### `retry_on` — ลองใหม่อัตโนมัติเมื่อเจอ error ที่กำหนด

```ruby
class FlakyJob < ApplicationJob
  queue_as :default
  retry_on StandardError, wait: 3.seconds, attempts: 3

  def perform
    count_file = Rails.root.join("tmp/retry_test/count.txt")
    count = File.exist?(count_file) ? File.read(count_file).to_i : 0
    count += 1
    File.write(count_file, count)
    Rails.logger.info "[FlakyJob] พยายามครั้งที่ #{count}"
    raise "จำลอง error ชั่วคราว (เช่น network timeout)" if count < 3
    Rails.logger.info "[FlakyJob] สำเร็จในที่สุด!"
  end
end
```

รันจริงผ่าน `FlakyJob.perform_later` แล้วปล่อยให้ `bin/jobs` ประมวลผล — log จริงที่ได้:

```
[ActiveJob] [FlakyJob] [...] Retrying FlakyJob (Job ID: ...) after 2 attempts in 3 seconds,
  due to a RuntimeError (จำลอง error ชั่วคราว (เช่น network timeout)).
[ActiveJob] [FlakyJob] [...] Performing FlakyJob (Job ID: ...) from SolidQueue(default) ...
[ActiveJob] [FlakyJob] [...] [FlakyJob] พยายามครั้งที่ 3
[ActiveJob] [FlakyJob] [...] [FlakyJob] สำเร็จในที่สุด!
[ActiveJob] [FlakyJob] [...] Performed FlakyJob (Job ID: ...) from SolidQueue(default) in 0.73ms
```

สังเกตว่า Solid Queue **จัดการ retry ให้อัตโนมัติทั้งหมด** — ครั้งที่ 1 และ 2 ล้มเหลว (raise
error) ระบบรอ 3 วินาทีตามที่ตั้งไว้แล้วลองใหม่เอง จนครั้งที่ 3 สำเร็จ โดยที่**เราไม่ต้องเขียน
โค้ด retry logic เอง** เพียงประกาศ `retry_on` บรรทัดเดียวเท่านั้น

พารามิเตอร์ที่ใช้บ่อย:

- **`wait:`** — เวลาที่รอก่อนลองใหม่ (รับ `Duration` ธรรมดา หรือ `:polynomially_longer`
  สำหรับ exponential backoff)
- **`attempts:`** — จำนวนครั้งสูงสุดที่จะลอง (default คือ 5)

### `discard_on` — เลิกทำทันทีเมื่อเจอ error ที่ไม่มีทางแก้ได้

ทบทวนปัญหาจาก Step 604: ถ้า `Contact` ที่ job อ้างถึงถูกลบไปก่อนที่ job จะรันจริง จะเกิด
`ActiveJob::DeserializationError` มาดูพฤติกรรม**ก่อน**ใส่ `discard_on`:

```ruby
c = Contact.create!(name: "ทดสอบลบ", email: "del@example.com")
SendWelcomeEmailJob.perform_later(c)
c.destroy!
```

log ที่ได้เมื่อ worker พยายามรัน (ยังไม่มี `discard_on`):

```
[ActiveJob] [SendWelcomeEmailJob] [...] Error performing SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) in 8.46ms: ActiveJob::DeserializationError
  (Error while trying to deserialize arguments: Couldn't find Contact with 'id'="2"):
```

job กลายเป็น **failed execution** ที่ค้างอยู่ใน `solid_queue_failed_executions` — ต้องมีคน
ไปดูแลจัดการ (retry ด้วยมือ หรือลบทิ้ง) ทั้งที่ในกรณีนี้**ไม่มีทางแก้ไขได้เลย** (record ถูกลบ
ไปแล้วจริงๆ ลองใหม่กี่ครั้งก็ไม่มีวันสำเร็จ)

เพิ่ม `discard_on` ที่ `ApplicationJob` (ตามที่ Rails คอมเมนต์แนะนำไว้ให้ตั้งแต่ Step 603):

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  discard_on ActiveJob::DeserializationError
end
```

ทำซ้ำการทดสอบเดิม — log ที่ได้เปลี่ยนไปทันที:

```
[ActiveJob] [SendWelcomeEmailJob] [...] Discarded SendWelcomeEmailJob (Job ID: ...)
  due to a ActiveJob::DeserializationError
  (Error while trying to deserialize arguments: Couldn't find Contact with 'id'="4").
[ActiveJob] [SendWelcomeEmailJob] [...] Performed SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) in 6.04ms
```

**"Discarded"** แทนที่ **"Error"** — job ถูกทำเครื่องหมายว่าเสร็จสิ้น (ไม่ค้างเป็น failed
execution ให้ต้องจัดการ) โดยไม่มีข้อมูลอะไรเสียหาย เพราะไม่มีอะไรให้ "ทำสำเร็จ" ได้อยู่แล้ว
ตั้งแต่แรก

> **หลักการเลือกใช้:** ใช้ `discard_on` กับ error ที่บ่งบอกว่า **สภาวะของระบบเปลี่ยนไปแล้วจน
> งานนี้ไม่มีความหมายอีกต่อไป** (record ถูกลบ, business rule เปลี่ยน) ใช้ `retry_on` กับ error
> ที่เป็น**ความล้มเหลวชั่วคราว**ของระบบภายนอกที่มีโอกาสสำเร็จถ้าลองใหม่ (network, database
> deadlock, rate limit) — Part 062 จะสอน retry strategy ขั้นสูงกว่านี้ (exponential backoff,
> dead job queue, การแจ้งเตือนเมื่อ retry ครบจำนวนแล้วยังไม่สำเร็จ)

---

## Step 610: การทดสอบ Job — `perform_enqueued_jobs`, `assert_enqueued_with`, RSpec matcher

### Test environment ใช้ adapter `:test` ให้อัตโนมัติ

ไม่ต้องตั้งค่าอะไรเอง Rails จะสลับไปใช้ `ActiveJob::QueueAdapters::TestAdapter` ให้ทันทีเมื่อ
รันใน test environment (ไม่ว่าจะตั้ง `:solid_queue` ไว้ใน `development.rb`/`production.rb`
ก็ตาม):

```bash
RAILS_ENV=test bin/rails runner 'puts ActiveJob::Base.queue_adapter.class'
# => ActiveJob::QueueAdapters::TestAdapter
```

adapter นี้ **ไม่รันงานจริง** แต่เก็บรายการ job ที่ถูก enqueue ไว้ในหน่วยความจำให้เราตรวจสอบได้
— นี่คือสิ่งที่ทำให้ทดสอบ job ได้เร็วและไม่ต้องพึ่งพา worker จริงหรือฐานข้อมูลคิวเลย

### Minitest (ทบทวนจาก Part 018)

```ruby
# test/jobs/send_welcome_email_job_test.rb
require "test_helper"

class SendWelcomeEmailJobTest < ActiveJob::TestCase
  test "enqueue SendWelcomeEmailJob เข้าคิว default พร้อม contact ที่ถูกต้อง" do
    contact = Contact.create!(name: "ทดสอบ", email: "test@example.com")

    assert_enqueued_with(job: SendWelcomeEmailJob, args: [ contact ], queue: "default") do
      SendWelcomeEmailJob.perform_later(contact)
    end
  end

  test "perform_enqueued_jobs รันงานจริงจนจบและไม่เหลือ job ค้างคิว" do
    contact = Contact.create!(name: "มานี", email: "manee@example.com")

    perform_enqueued_jobs do
      SendWelcomeEmailJob.perform_later(contact)
    end

    assert_enqueued_jobs 0
  end
end
```

- **`assert_enqueued_with`** — ตรวจสอบว่ามี job ที่ตรงกับเงื่อนไข (class, arguments, queue)
  ถูก enqueue ภายใน block ที่กำหนด **โดยไม่รันจริง**
- **`perform_enqueued_jobs`** — สั่งให้ job ที่ถูก enqueue ภายใน block **รันจริงทันที**
  (เหมือนมี worker อยู่ในโปรเซสทดสอบชั่วคราว) มีประโยชน์เมื่อต้องการทดสอบผลลัพธ์ปลายทางของ
  job (side effect เช่น การส่งอีเมลจริง, การเขียนข้อมูลลง log)
- **`assert_enqueued_jobs 0`** — ยืนยันว่าหลังจากรันเสร็จแล้วไม่มี job เหลือค้างอยู่ในคิว

รันจริง:

```bash
bin/rails test test/jobs/send_welcome_email_job_test.rb
```

```
Running 2 tests in a single process (parallelization threshold is 50)
Run options: --seed 49824

# Running:

..

Finished in 2.087737s, 0.9580 runs/s, 1.9160 assertions/s.
2 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

(สังเกตว่าใช้เวลา ~2 วินาทีจริง เพราะ `perform_enqueued_jobs` รันโค้ดใน `perform` จริง
รวมถึง `sleep 2` ที่เราใส่จำลองไว้ — นี่คือเหตุผลที่ test ที่ทดสอบ job ควรแยกออกจาก
test ที่ทดสอบแค่ "มีการ enqueue หรือไม่" ซึ่งเร็วกว่ามาก)

### RSpec (ทบทวนจาก Part 019 และ Part 046)

ติดตั้ง `rspec-rails` แล้วเขียน job spec ด้วย `type: :job`:

```ruby
# spec/jobs/send_welcome_email_job_spec.rb
require "rails_helper"

RSpec.describe SendWelcomeEmailJob, type: :job do
  include ActiveJob::TestHelper

  let(:contact) { Contact.create!(name: "สมหญิง", email: "somying@example.com") }

  it "enqueue เข้าคิว default" do
    expect { SendWelcomeEmailJob.perform_later(contact) }
      .to have_enqueued_job(SendWelcomeEmailJob)
      .with(contact)
      .on_queue("default")
  end

  it "perform_enqueued_jobs รันงานจริงจนจบ (have_performed_job)" do
    expect {
      perform_enqueued_jobs do
        SendWelcomeEmailJob.perform_later(contact)
      end
    }.to have_performed_job(SendWelcomeEmailJob).with(contact)
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/jobs/send_welcome_email_job_spec.rb
```

```
..

Finished in 2.08 seconds (files took 0.69933 seconds to load)
2 examples, 0 failures
```

> **บั๊กที่เจอจริงระหว่างเขียนตัวอย่างนี้ (คุ้มค่าที่จะรู้ไว้):** ตอนแรกเขียน test ที่สองเป็น
> `expect(SendWelcomeEmailJob).to have_been_enqueued.exactly(1).times` วางไว้**หลัง**
> `perform_enqueued_jobs` แล้ว test fail ด้วยข้อความ `expected to enqueue exactly 1 jobs,
> but enqueued 0` เหตุผลคือ **`have_been_enqueued` เช็คจากรายการ "enqueued jobs" ที่ยังไม่ถูก
> รัน** แต่ `perform_enqueued_jobs` ได้ย้ายงานนั้นจาก "enqueued" ไปเป็น "performed" เรียบร้อย
> แล้วก่อนที่ assertion จะรัน วิธีแก้คือใช้ **`have_performed_job`** แทน (เช็คจากรายการงานที่
> ถูก**รันจบแล้ว**) ตามที่แก้ไขไว้ในโค้ดด้านบน — ตัวอย่างที่ดีว่า matcher สองตัวที่ชื่อคล้ายกัน
> (`have_been_enqueued` vs `have_performed_job`) เช็คคนละสถานะของ job และเลือกผิดตัวทำให้
> test fail อย่างงงๆ ได้ง่ายมาก

ตารางสรุปเทียบ Minitest กับ RSpec matcher ที่ใช้บ่อยที่สุด:

| ต้องการตรวจสอบ | Minitest (`ActiveJob::TestCase`) | RSpec (`rspec-rails`) |
|---|---|---|
| มี job ถูก enqueue ไหม (ยังไม่รัน) | `assert_enqueued_with(job:, args:, queue:)` | `have_enqueued_job(Job).with(...).on_queue(...)` |
| จำนวน job ที่ค้างในคิว | `assert_enqueued_jobs n` | `have_enqueued_job` ร่วมกับ `.exactly(n).times` |
| รัน job ที่ enqueue ไว้จริง | `perform_enqueued_jobs { ... }` | `perform_enqueued_jobs { ... }` (ใช้ method เดียวกัน) |
| job รันจบแล้วสำเร็จไหม | ตรวจ side effect เอง (เช่น query DB) | `have_performed_job(Job).with(...)` |

---

## แบบฝึกหัด: enqueue job จริงจาก controller แล้วรันผ่าน Solid Queue

### โจทย์

สร้าง endpoint `POST /contacts` ที่:

1. สร้าง `Contact` ใหม่จาก parameter ที่รับมา
2. Enqueue **สองงาน**: `SendWelcomeEmailJob` (คิว `default`, ใช้เวลา 2 วินาที) และ
   `ImageResizeJob` (คิว `high_priority`, ใช้เวลา 3 วินาที)
3. ตอบกลับ response **ทันที** โดยไม่รอทั้งสองงานให้เสร็จก่อน
4. พิสูจน์ด้วยเวลาจริง (`curl` วัดเวลา) และ log จริงจาก `bin/jobs` ว่าทั้งสองงานถูกรันเบื้องหลัง
   จริงๆ **หลังจาก** response ถูกส่งกลับไปแล้ว

### เฉลย

**Routes:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :contacts, only: %i[ create ]
end
```

**Controller:**

```ruby
# app/controllers/contacts_controller.rb
class ContactsController < ApplicationController
  skip_forgery_protection

  def create
    contact = Contact.create!(contact_params)

    # จุดสำคัญ: perform_later ไม่รอผลลัพธ์ ตัว request จะตอบกลับทันที
    SendWelcomeEmailJob.perform_later(contact)
    ImageResizeJob.perform_later(contact)

    render json: {
      id: contact.id,
      name: contact.name,
      message: "สร้าง contact สำเร็จ งานเบื้องหลังถูกส่งเข้าคิวแล้ว"
    }, status: :created
  end

  private
    def contact_params
      params.require(:contact).permit(:name, :email)
    end
end
```

**ทดสอบจริง:** เปิด terminal สองหน้าต่าง — หน้าต่างแรกรัน web server, หน้าต่างที่สองรัน worker:

```bash
# Terminal 1
bin/rails server -p 3057

# Terminal 2
bin/jobs
```

ยิง request จริงพร้อมวัดเวลา:

```bash
curl -s -w "\nHTTP %{http_code}, time_total=%{time_total}s\n" -X POST http://localhost:3057/contacts \
  -H "Content-Type: application/json" \
  -d '{"contact":{"name":"ทดสอบ Controller","email":"ctrl@example.com"}}'
```

ผลลัพธ์จริง:

```
{"id":5,"name":"ทดสอบ Controller","message":"สร้าง contact สำเร็จ งานเบื้องหลังถูกส่งเข้าคิวแล้ว"}
HTTP 201, time_total=0.420571s
```

**Request ตอบกลับใน 0.42 วินาที** — เร็วกว่าเวลารวมของทั้งสองงาน (2+3 = 5 วินาที ถ้าเป็น
`perform_now`) อย่างมหาศาล ตรวจสอบ `log/development.log` หลังจากนั้นเพื่อดูว่างานทั้งสองรันจริง
เบื้องหลังไปพร้อมๆ กัน (คนละ thread ใน worker pool):

```
[ActiveJob] [SendWelcomeEmailJob] [...] Performing SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) enqueued at 2026-09-26T08:16:00.589538997Z with arguments: ...
[ActiveJob] [ImageResizeJob] [...] Performing ImageResizeJob (Job ID: ...)
  from SolidQueue(high_priority) enqueued at 2026-09-26T08:16:00.653907109Z with arguments: ...
[ActiveJob] [SendWelcomeEmailJob] [...] [SendWelcomeEmailJob] ส่งอีเมลต้อนรับให้
  ทดสอบ Controller <ctrl@example.com>
[ActiveJob] [SendWelcomeEmailJob] [...] Performed SendWelcomeEmailJob (Job ID: ...)
  from SolidQueue(default) in 2050.1ms
[ActiveJob] [ImageResizeJob] [...] [ImageResizeJob] resize รูปโปรไฟล์ของ ทดสอบ Controller เสร็จแล้ว
[ActiveJob] [ImageResizeJob] [...] Performed ImageResizeJob (Job ID: ...)
  from SolidQueue(high_priority) in 3054.06ms
```

**สังเกตช่วงเวลา:** ทั้งสองงานถูก enqueue ตอน `08:16:00` (พร้อมกันเกือบเป๊ะ ตอนที่ controller
action ทำงาน) แต่ response ถูกส่งกลับไปให้ผู้ใช้ตั้งแต่ก่อนหน้านั้นแล้ว (`time_total=0.42s`)
ส่วน `SendWelcomeEmailJob` ใช้เวลารัน 2050ms และ `ImageResizeJob` ใช้เวลา 3054ms — **ทั้งสอง
เกิดขึ้น "หลังจาก" ที่ผู้ใช้ได้รับ response ไปแล้วเรียบร้อย** นี่คือหัวใจของ background job
ที่ทำให้ผู้ใช้ไม่ต้องรอ ในขณะที่งานที่ใช้เวลานานยังคงถูกทำจนเสร็จอย่างน่าเชื่อถือ (เพราะเก็บอยู่
ในฐานข้อมูล Solid Queue ไม่ใช่แค่ในหน่วยความจำ)

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **แยก worker pool สำหรับคิว `high_priority` ออกจาก `default`** ใน `config/queue.yml`
   ตามรูปแบบที่สอนใน Step 608 (`threads: 5, processes: 2` เฉพาะ `high_priority`) แล้วจำลอง
   สถานการณ์ที่คิว `default` มีงานค้างอยู่จำนวนมาก (enqueue `SendWelcomeEmailJob` รัวๆ 20 ครั้ง)
   พิสูจน์ด้วย log ว่า job ในคิว `high_priority` ที่ enqueue เข้ามาทีหลังยังคงถูกรันได้ทันที
   ไม่ต้องรอคิว `default` ให้ว่างก่อน

2. **เขียน `ReportGenerationJob`** ที่จำลองการเรียก API ภายนอกด้วย custom exception ของตัวเอง
   (เช่น `class ExternalApiTimeoutError < StandardError; end`) ใส่ `retry_on
   ExternalApiTimeoutError, wait: :polynomially_longer, attempts: 5` และใส่ `discard_on` สำหรับ
   กรณีที่ record ต้นทางถูกลบไปแล้ว จากนั้นเขียนทั้ง Minitest (`assert_enqueued_with`) และ
   RSpec (`have_enqueued_job`) มายืนยันพฤติกรรมทั้งสองกรณี

3. **เปรียบเทียบ `:async` กับ `:solid_queue` ด้วยมือของตัวเอง:** สลับ
   `config.active_job.queue_adapter` เป็น `:async` ใน development แล้ว enqueue job ที่ตั้งเวลา
   ล่วงหน้าไว้ 20 วินาที จากนั้น **restart `bin/rails server` ก่อนถึงเวลา** พิสูจน์ว่า job
   หายไปจริง (เหมือนที่ทำใน Step 605) แล้วทำซ้ำขั้นตอนเดียวกันด้วย `:solid_queue` พิสูจน์ว่า
   job ยังอยู่และรันสำเร็จหลัง restart เขียนสรุปเปรียบเทียบเป็นตารางของตัวเอง พร้อมระบุว่า
   ในสถานการณ์ใดที่ `:async` ยังพอรับได้ และสถานการณ์ใดที่ต้องใช้ adapter ที่คงทนเท่านั้น

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่าทำไมงานที่ใช้เวลานาน (ส่งอีเมล, ประมวลผลรูปภาพ, สร้างรายงาน, เรียก API ภายนอก)
  ไม่ควรทำแบบ synchronous ใน request/response cycle และเชื่อมโยงกับ `deliver_later` ที่เคย
  ใช้แบบผิวเผินใน Part 041
- เข้าใจ **ActiveJob** ในฐานะเฟรมเวิร์กที่ **ไม่ผูกติดกับ backend ใดๆ (adapter-agnostic)** —
  เขียนโค้ด job ครั้งเดียว สลับ adapter ได้โดยไม่แก้ business logic
- `rails generate job` และกายวิภาคของ Job class: `ApplicationJob`, `queue_as`, `perform`
- ความแตกต่างระหว่าง **`perform_later`** (asynchronous, เข้าคิว) กับ **`perform_now`**
  (synchronous, บล็อกทันที) พร้อมพิสูจน์เวลาจริง
- **Global ID**: การส่ง ActiveRecord object เข้า job จริงๆ แล้วส่งแค่ "ที่อยู่อ้างอิง"
  (`gid://app/Model/id`) ไม่ใช่ snapshot ของข้อมูล และ job จะดึงข้อมูล**สดใหม่จากฐานข้อมูล
  เสมอ**ตอนรันจริง (พิสูจน์ด้วยการเปลี่ยนชื่อ record หลัง enqueue)
- Adapter `:async` — รันในโปรเซสเดียวกัน เหมาะกับ dev/test เท่านั้น **งานหายถาวรเมื่อ process
  จบ** (พิสูจน์จริงว่า job ที่ enqueue ไว้ไม่เคยรันเลยหลัง process จบก่อนถึงเวลา)
- **Solid Queue** — ระบบคิวงานแบบ DB-backed ที่ **Rails 8 ติดตั้งให้เป็นค่า default ตั้งแต่
  `rails new`** (แต่เปิดใช้งานอัตโนมัติเฉพาะ production เท่านั้น) ไม่ต้องมี Redis ตั้งค่าให้
  ทำงานใน development ได้เอง และพิสูจน์ความคงทน (durability) จริงว่า job ยังอยู่ในฐานข้อมูล
  แม้ไม่มี worker รันอยู่เลย
- ภาพรวม **Sidekiq** (Redis-backed, มาตรฐานเก่าแก่ของวงการ, มี Web UI, เจาะลึกเต็มรูปแบบใน
  Part 062) และ **GoodJob** (DB-backed อีกตัว เน้น PostgreSQL)
- ตั้งค่า adapter ผ่าน `config.active_job.queue_adapter`, กำหนดชื่อคิวด้วย `queue_as`
  (รวมถึงแบบ dynamic ด้วย block), และเข้าใจว่าลำดับของคิวใน `queue.yml` มีผลเหนือกว่า
  `priority` ที่เป็นตัวเลข
- พื้นฐาน **`retry_on`** (ลองใหม่อัตโนมัติสำหรับ error ชั่วคราว) และ **`discard_on`**
  (เลิกทำทันทีสำหรับ error ที่ไม่มีทางแก้ได้ เช่น `ActiveJob::DeserializationError`) พร้อม
  หลักฐาน log จริงของทั้งสองกรณี
- ทดสอบ job ด้วย Minitest (`assert_enqueued_with`, `perform_enqueued_jobs`,
  `assert_enqueued_jobs`) และ RSpec (`have_enqueued_job`, `have_performed_job`) พร้อมบั๊กจริง
  ที่เจอระหว่างทาง (เลือก matcher ผิดตัวระหว่าง enqueued กับ performed)
- สร้าง endpoint ที่ enqueue สองงานพร้อมกันจาก controller แล้วพิสูจน์ด้วยเวลาจริงว่า response
  ตอบกลับเร็วกว่าผลรวมเวลาของงานเบื้องหลังอย่างมาก

**ต่อไป (Part 062):** เราจะเจาะลึก **Sidekiq** แบบเต็มรูปแบบ — ติดตั้งพร้อม Redis จริง,
Sidekiq Web UI, การจัดการ **queue** หลายระดับความสำคัญ, **retry** strategy ขั้นสูง (custom
backoff, dead job queue, การแจ้งเตือนเมื่อ job ล้มเหลวถาวร), และ **scheduled job** (cron-like
job ที่รันตามเวลาที่กำหนดซ้ำๆ) — ทักษะที่จำเป็นสำหรับระบบระดับ production ที่ต้องการ throughput
สูงและการมองเห็น (observability) การทำงานของ background job ได้ชัดเจนที่สุด
