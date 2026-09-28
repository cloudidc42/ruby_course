# Part 078: Monitoring & Logging — Structured Logging, Error Tracking (Sentry), Uptime Monitoring — ปิด Phase 12 ด้วยชุดเครื่องมือมองเห็นระบบ Production

> **Step ครอบคลุมใน Part นี้:** Step 771–780
> **ระดับ:** สูง (ต้องผ่าน Part 073 Docker, Part 074 Environment/Credentials, Part 075 CI/CD,
> Part 076 Kamal, และ Part 077 Deployment ทางเลือก/Safe Migration มาก่อน — Part นี้ไม่สอนวิธี
> deploy ซ้ำ แต่สอนว่า **หลัง deploy สำเร็จแล้ว จะรู้ได้อย่างไรว่าแอปยังทำงานถูกต้องอยู่**)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6, Rails 8.1.4, `lograge` 0.15.0, `sentry-ruby`/`sentry-rails`
> 5.28.1 — ทุกคำสั่ง, log output, และผลการรันในเอกสารนี้ทดสอบจริงบนแอป Rails ทดลองที่สร้างขึ้น
> เฉพาะสำหรับ Part นี้ (ไม่มีตัวเลขหรือ output ใดที่แต่งขึ้นเอง) ยกเว้นจุดเดียวที่ระบุไว้ชัดเจน
> ว่า **verify ไม่ได้ในสภาพแวดล้อมทดลอง** คือการเห็น error ปรากฏจริงในหน้า dashboard ของ
> Sentry.io เพราะต้องมีบัญชีและ DSN จริงซึ่งอยู่นอกเหนือขอบเขตของ sandbox นี้

Part นี้คือ **Part สุดท้ายของ Phase 12: DevOps & Deployment** ตลอด 5 Part ที่ผ่านมาเราเรียนรู้
วิธี "ส่งแอปออกไปอยู่ในโลกจริง" ทีละขั้น — บรรจุแอปเป็น Docker image (Part 073), จัดการ
environment variable และ credentials ให้ปลอดภัย (Part 074), ตั้ง CI/CD ให้ทดสอบและตรวจสอบโค้ด
อัตโนมัติก่อนปล่อยออกไป (Part 075), deploy ขึ้นเครื่อง production จริงด้วย Kamal (Part 076),
และรู้จักทางเลือกอื่นๆ พร้อมวิธี migrate ฐานข้อมูลอย่างปลอดภัย (Part 077) — แต่คำถามที่ยังไม่มี
คำตอบตลอดมาคือ **"แล้วหลังจากนั้นล่ะ?"** แอปรันอยู่บนเซิร์ฟเวอร์ที่ไม่มีใครนั่งจ้องหน้าจอตลอด 24
ชั่วโมง จะรู้ได้อย่างไรว่ามันยังทำงานถูกต้อง มี error เกิดขึ้นหรือไม่ ช้าลงหรือเปล่า หรือกระทั่ง
"ตายไปแล้ว" โดยไม่มีใครรู้ Part นี้คือคำตอบ: เราจะเรียนรู้ **observability** สามด้านที่ทำงาน
ร่วมกัน — log ที่มีโครงสร้างอ่านง่ายด้วยเครื่องจักร (`lograge`), การติดตามข้อผิดพลาดแบบรวมศูนย์
(Sentry), และการเฝ้าดูว่าแอปยัง "มีชีวิต" อยู่หรือไม่ (uptime monitoring ผ่าน endpoint `/up`
ที่ Rails 8 เตรียมไว้ให้) ปิดท้ายด้วยกรอบความคิดเวลาเกิดเหตุฉุกเฉินจริง (incident response) ที่
ร้อยทุกอย่างจาก Part 073–077 กลับมาใช้งานพร้อมกัน

## สารบัญของ Part นี้

- Step 771: ทำไมต้องมี Observability — เมื่อแนบ debugger เข้ากับ production ไม่ได้เหมือนตอน dev
- Step 772: ข้อจำกัดของ Rails default logger และแนวคิด Structured/JSON Logging
- Step 773: ติดตั้ง `lograge` — เปรียบเทียบ log ก่อน/หลังแบบรันจริง
- Step 774: เพิ่ม custom field เข้า log (`request_id`, `current_user` id) ผ่าน `custom_payload`
- Step 775: Log level (`debug`/`info`/`warn`/`error`/`fatal`) และการตั้งค่าต่อ environment
- Step 776: แนวคิด Centralized Log Aggregation — ทำไม `tail -f` ใช้ไม่ได้อีกต่อไป
- Step 777: Error Tracking ด้วย Sentry — ติดตั้ง, `Sentry.init`, automatic exception capture
- Step 778: รายงาน exception ที่ rescue ไว้ด้วยมือ (`Sentry.capture_exception`) + เพิ่ม context
  อย่างปลอดภัย (ไม่ให้ PII/secrets หลุด)
- Step 779: Uptime Monitoring ภายนอก + เจาะลึก endpoint `/up` ของ Rails 8 (default จริง)
- Step 780: Incident Response Mindset + แบบฝึกหัดปิด Phase 12 (ครบทุกเครื่องมือของ Phase นี้)

---

## Step 771: ทำไมต้องมี Observability — เมื่อแนบ debugger เข้ากับ production ไม่ได้เหมือนตอน dev

### ปัญหาที่ Part นี้แก้

ทบทวน Part 001 Step 5: ตอนพัฒนาบนเครื่องตัวเอง ถ้าโค้ดพังหรือพฤติกรรมแปลกๆ เราแทรก
`binding.pry` เข้าไปตรงจุดที่สงสัย โปรแกรมจะหยุดรอ ณ จุดนั้นทันที ให้เราตรวจสอบค่าตัวแปรทีละตัว
แบบ interactive ได้เต็มที่ — วิธีนี้ใช้ได้ดีมากตอน dev เพราะมีแค่เราคนเดียวที่ใช้แอปอยู่ และหยุด
โปรแกรมไว้นานแค่ไหนก็ไม่มีผลกระทบต่อใคร

แต่บน **production server** สถานการณ์ต่างไปโดยสิ้นเชิง:

1. **มีผู้ใช้จริงหลายร้อยหลายพันคนใช้งานพร้อมกัน** — ถ้าแทรก breakpoint แล้ว process หยุดรอ
   จริงๆ, request อื่นทั้งหมดที่ใช้ thread/worker เดียวกันจะค้างตามไปด้วย
2. **ไม่รู้ล่วงหน้าว่าปัญหาจะเกิดที่ไหน** — ตอน dev เรามักรู้อยู่แล้วว่ากำลังทดสอบฟีเจอร์ไหน
   จึงวาง breakpoint ถูกจุด แต่ปัญหาที่เกิดใน production มักเป็นสิ่งที่ไม่มีใครคาดคิดมาก่อน
   (edge case ที่ test ไม่ครอบคลุม, ข้อมูลจริงที่แปลกกว่าที่คิด, load สูงกว่าที่ทดสอบ)
3. **มีหลาย instance พร้อมกัน** — ผลจาก Part 076 (Kamal) และ Part 073 (Docker): production
   จริงมักรันแอปเดียวกันพร้อมกันหลาย container/หลายเครื่อง อยู่เบื้องหลัง load balancer คุณไม่รู้
   ด้วยซ้ำว่า request ที่มีปัญหาไปตกที่ instance ไหนในจำนวนนั้น จะ SSH เข้าไปนั่ง debug ทีละเครื่อง
   ไม่ทัน
4. **container เป็นของชั่วคราว (ephemeral)** — เมื่อ deploy ใหม่ (Part 076) container เก่าจะถูก
   ทำลายทิ้งทันที ถ้าไม่มีอะไรเก็บร่องรอยไว้ก่อน หลักฐานทั้งหมดของปัญหาที่เพิ่งเกิดจะหายไปพร้อมกับ
   container นั้น

สรุปสั้นๆ: **เราไม่สามารถ "ไปถามคำถามกับแอปที่กำลังรันอยู่" แบบ interactive ได้เหมือนตอน dev**
สิ่งเดียวที่พอทำได้คือให้แอป **ทิ้งร่องรอย (telemetry)** ไว้ระหว่างที่มันทำงานอยู่ตลอดเวลา แล้ว
เราไปตรวจสอบร่องรอยเหล่านั้นทีหลัง (หรือดูสดๆ แบบ real-time) — นี่คือความหมายของคำว่า
**Observability** (การสังเกตการณ์ได้)

### เชื่อมโยงกับ Part 065 — คนละโหมด แต่เป้าหมายคล้ายกัน

Part 065 (Performance Profiling) สอนหลักการ **"วัดก่อน อย่าเดา"** ด้วยเครื่องมือ
`rack-mini-profiler`, `memory_profiler` ฯลฯ — แต่เครื่องมือเหล่านั้นเป็น **on-demand**: เปิดใช้
ตอน dev/staging เมื่อ*สงสัย*ว่าหน้าเว็บหนึ่งช้า แล้ววัดครั้งเดียวเพื่อตอบคำถามเฉพาะจุด (จึงมักปิด
ไว้ใน production เพราะเพิ่ม overhead ให้ทุก request)

Part นี้ตรงข้ามกัน — เครื่องมือทั้งหมดที่จะเรียนเป็น **continuous**: เปิดทิ้งไว้ตลอดเวลาใน
production เพื่อตอบคำถามที่ยังไม่มีใครถามด้วยซ้ำ เช่น "มี error อะไรเกิดขึ้นบ้างคืนที่ผ่านมาตอน
ไม่มีใครเฝ้า", "แอปยังตอบสนองอยู่ไหมตอนตี 3" — เปรียบได้กับความต่างระหว่างการไปตรวจสุขภาพประจำปี
แบบเจาะลึก (Part 065 profiling) กับการติดเครื่องวัดสัญญาณชีพไว้ตลอดเวลาข้างเตียงผู้ป่วย (Part
นี้) — ทั้งสองอย่างจำเป็นต้องมีคู่กัน แต่ทำหน้าที่คนละแบบ

> **preview การเชื่อมโยงกับ Phase 12 ทั้งหมด:** Part 073–077 คือชุดเครื่องมือที่ทำให้ "ส่งโค้ด
> ออกไปทำงานจริงได้" (build, config, test, deploy, migrate) — Part นี้คือชุดเครื่องมือที่ทำให้
> "รู้ว่ามันทำงานถูกต้องหลังจากนั้น" ทั้งสองส่วนรวมกันจึงจะเรียกได้ว่าเป็น DevOps ที่ครบวงจรจริงๆ
> ไม่ใช่แค่ "deploy แล้วภาวนา" (deploy and pray)

### สามเสาหลักของ Observability ที่จะเจอใน Part นี้

| เสาหลัก | ตอบคำถามว่า | เครื่องมือใน Part นี้ |
|---|---|---|
| **Logs** | เกิดอะไรขึ้นบ้าง ตามลำดับเวลา | Rails logger + `lograge` (Step 772–776) |
| **Error Tracking** | มี exception อะไรเกิดขึ้น ที่ไหน บ่อยแค่ไหน | Sentry (Step 777–778) |
| **Uptime/Health** | ระบบยัง "มีชีวิต" ตอบสนองอยู่ไหมตอนนี้ | `/up` + uptime monitor ภายนอก (Step 779) |

ทั้งสามอย่างนี้ไม่ใช่ของแยกกันโดยสิ้นเชิง — Step 780 จะแสดงให้เห็นว่าตอนเกิดเหตุฉุกเฉินจริง เรา
ใช้ทั้งสามอย่างประกอบกันเป็นขั้นตอนเดียว

---

## Step 772: ข้อจำกัดของ Rails default logger และแนวคิด Structured/JSON Logging

### แอปทดลองที่ใช้ตลอด Part นี้

เพื่อให้ทุก log output ในเอกสารนี้เป็นของจริง เราสร้างแอป Rails ทดลองขึ้นมาหนึ่งตัว (ไม่รวมอยู่ใน
โค้ดหลักสูตร เป็นแค่ sandbox สำหรับสาธิต เหมือนที่ Part 065 ทำ) ด้วยคำสั่ง

```bash
rails new monitoring_demo --minimal
cd monitoring_demo
bin/rails g scaffold Post title:string body:text
bin/rails db:migrate
```

พร้อม action เสริม 2 ตัวใน `PostsController` สำหรับสาธิต error tracking ใน Step 777–778:

```ruby
# app/controllers/posts_controller.rb (ตัดมาเฉพาะส่วนที่เพิ่ม)
class PostsController < ApplicationController
  # unrescued exception -- ใช้ทดสอบว่า sentry-rails ดัก exception อัตโนมัติได้จริงหรือไม่
  def crash
    raise "unrescued crash for sentry-rails auto-capture test"
  end

  # rescued exception -- ใช้ทดสอบการรายงานมือด้วย Sentry.capture_exception (Step 778)
  def boom
    1 / 0
  rescue ZeroDivisionError => e
    Sentry.capture_exception(e)
    Rails.logger.error("Rescued in #boom: #{e.class}: #{e.message}")
    render plain: "handled", status: :ok
  end
end
```

### ทดลองยิง request จริง แล้วดู log เริ่มต้นของ Rails

```bash
curl -X POST http://127.0.0.1:3099/posts \
  -d "post[title]=Hello&post[body]=World" -H "Accept: text/html"
```

นี่คือเนื้อหาไฟล์ `log/development.log` หลัง request นี้ผ่านไป (คัดลอกมาตรงๆ จากการรันจริง —
สังเกตรหัสสี ANSI `[1m[36m...[0m` ที่ปนอยู่ในไฟล์ด้วย เพราะ Rails เปิด `colorize_logging`
ไว้เป็นค่า default ใน development):

```text
Started POST "/posts" for 127.0.0.1 at 2026-09-28 18:09:46 +0000
  [1m[36mActiveRecord::SchemaMigration Load (0.2ms)[0m  [1m[34mSELECT "schema_migrations"."version" FROM "schema_migrations" ORDER BY "schema_migrations"."version" ASC /*application='MonitoringDemo'*/[0m
Processing by PostsController#create as HTML
  Parameters: {"post"=>{"title"=>"Hello", "body"=>"World"}}
  [1m[36mTRANSACTION (0.1ms)[0m  [1m[35mBEGIN immediate TRANSACTION /*action='create',application='MonitoringDemo',controller='posts'*/[0m
  ↳ app/controllers/posts_controller.rb:35:in `create'
  [1m[36mPost Create (0.9ms)[0m  [1m[32mINSERT INTO "posts" ("title", "body", "created_at", "updated_at") VALUES ('Hello', 'World', '2026-09-28 18:09:47.073156', '2026-09-28 18:09:47.073156') RETURNING "id" /*action='create',application='MonitoringDemo',controller='posts'*/[0m
  ↳ app/controllers/posts_controller.rb:35:in `create'
  [1m[36mTRANSACTION (1.1ms)[0m  [1m[35mCOMMIT TRANSACTION /*action='create',application='MonitoringDemo',controller='posts'*/[0m
  ↳ app/controllers/posts_controller.rb:35:in `create'
Redirected to http://127.0.0.1:3099/posts/1
↳ app/controllers/posts_controller.rb:36:in `create'
Completed 302 Found in 44ms (ActiveRecord: 3.5ms (1 query, 0 cached) | GC: 15.6ms)
```

### ปัญหาที่แท้จริงของ format นี้ (ไม่ใช่แค่ "อ่านไม่สวย")

request เดียว (`POST /posts`) ใช้ไป**สิบบรรทัด** ซึ่งเป็นปัญหาจริง 3 ระดับ ไม่ใช่แค่ความสวยงาม:

1. **1 request = หลายบรรทัดที่กระจัดกระจาย** — ถ้ามี 50 request เข้ามาพร้อมกัน (ปกติมากใน
   production ที่มีหลาย thread/worker) บรรทัดของแต่ละ request จะ**สลับปนกัน** (interleaved)
   ในไฟล์ log เดียวกัน ไม่มีทางรู้เลยว่าบรรทัด `Parameters: {...}` บรรทัดหนึ่งเป็นของ
   `Completed` บรรทัดไหนใน 50 request ที่กำลังทำงานพร้อมกันอยู่นั้น (ในตัวอย่างข้างต้นมี
   `request_id` ซ่อนอยู่ในหน่วยความจำของ Rails ก็จริง แต่**ไม่ได้ถูกพิมพ์ลง log บรรทัดไหนเลย**
   ตามค่า default)
2. **เป็น human-readable ไม่ใช่ machine-parseable** — ข้อความเช่น
   `Completed 302 Found in 44ms (ActiveRecord: 3.5ms (1 query, 0 cached) | GC: 15.6ms)` มนุษย์
   อ่านเข้าใจได้ทันที แต่ถ้าอยากเขียนโปรแกรมถามว่า "ขอ duration เฉลี่ยของทุก request ที่
   status เป็น 500 ใน 1 ชั่วโมงที่ผ่านมา" ต้องเขียน regular expression ที่เปราะบางมาก
   (พังทันทีถ้า Rails เปลี่ยน format ข้อความแม้เพียงเล็กน้อยในเวอร์ชันถัดไป)
3. **รหัสสี ANSI ปนอยู่ในไฟล์จริง** — อย่างที่เห็นข้างบน `[1m[36m` ไม่ใช่แค่ตอนแสดงผลบนหน้าจอ
   แต่ถูกเขียนลงไฟล์ log จริงๆ เครื่องมือใดๆ ที่มาอ่านไฟล์นี้ต้องกรองอักขระเหล่านี้ออกก่อน
   ถึงจะ parse เนื้อหาจริงได้

### ทางแก้ระดับแนวคิด: Structured Logging

**Structured logging** คือแนวทางเขียน log ให้เป็น **ข้อมูลที่มีโครงสร้างชัดเจน (เช่น JSON)**
แทนประโยคภาษาธรรมชาติ — แต่ละ event มี key/value ที่แน่นอน เครื่องมือใดๆ (แม้กระทั่ง `jq`
ธรรมดาบน command line) อ่านและ query ได้ทันทีโดยไม่ต้องเขียน regex เดา

Rails มีจุดเชื่อมสำหรับเปลี่ยนรูปแบบการเขียน log อยู่แล้วคือ `config.log_formatter` — มาดู
กลไกจริงเบื้องหลังก่อนแนะนำเครื่องมือสำเร็จรูปในขั้นถัดไป (ตรวจสอบจาก source code ของ Rails
8.1.4 จริง):

```ruby
# railties-8.1.4/lib/rails/application/configuration.rb (ค่า default จริง)
@log_formatter = ActiveSupport::Logger::SimpleFormatter.new

# railties-8.1.4/lib/rails/application/bootstrap.rb (จุดที่ค่านี้ถูกนำไปใช้จริงตอน boot)
logger.formatter = config.log_formatter
```

ลองเขียน formatter เองแบบง่ายๆ เพื่อดูว่ากลไกนี้ทำงานอย่างไร (ทดสอบรันจริงแบบ standalone
ด้วย Ruby ล้วน ก่อนเอาไปใส่ใน Rails):

```ruby
# frozen_string_literal: true

require "json"
require "time" # ต้อง require เพื่อให้ Time มี method #iso8601 (ทบทวน Part 001 Step 10)
require "logger"

class JsonLogFormatter < ::Logger::Formatter
  def call(severity, time, progname, msg)
    JSON.generate({
      severity: severity,
      time: time.utc.iso8601(3),
      progname: progname,
      message: msg.is_a?(String) ? msg : msg.inspect
    }) + "\n"
  end
end

logger = Logger.new($stdout)
logger.formatter = JsonLogFormatter.new
logger.info("hello structured world")
logger.warn("something looks off")
```

ผลการรันจริง:

```text
{"severity":"INFO","time":"2026-09-28T18:11:12.746Z","progname":null,"message":"hello structured world"}
{"severity":"WARN","time":"2026-09-28T18:11:12.746Z","progname":null,"message":"something looks off"}
```

แต่ละบรรทัดตอนนี้เป็น JSON object สมบูรณ์หนึ่งชิ้น — `jq '.severity'` หรือเครื่องมือ log
aggregator ใดๆ (Step 776) อ่านและ filter ได้ทันที

**แต่การเปลี่ยนแค่ `log_formatter` ยังไม่พอ** — ถ้าเอา formatter นี้ไปใส่ใน
`config.log_formatter` ของแอป Rails ตรงๆ ปัญหาข้อ 2 (parse ยาก) จะหายไป แต่ปัญหาข้อ 1 (1
request กระจายหลายบรรทัด) **ยังอยู่เหมือนเดิม** เพราะ Rails logger ยังคงเรียก `.info`/`.debug`
แยกกันหลายครั้งต่อ 1 request (`Started`, `Processing`, แต่ละ SQL query, `Completed`) — จะได้
JSON หลายบรรทัดต่อ 1 request แทนที่จะเป็นบรรทัดเดียว นี่คือเหตุผลที่ Step 773 แนะนำเครื่องมือ
สำเร็จรูปที่แก้ปัญหานี้โดยเฉพาะ แทนที่จะเขียน formatter เองต่อ

---

## Step 773: ติดตั้ง `lograge` — เปรียบเทียบ log ก่อน/หลังแบบรันจริง

**`lograge`** เป็น gem ที่ถูกออกแบบมาเพื่อแก้ปัญหาทั้งสองข้อจาก Step 772 พร้อมกัน — มันจะ
**รวบรวมทุก event ของ 1 request ให้เป็น log บรรทัดเดียว** ในรูปแบบที่ config ได้ (JSON, logfmt,
ฯลฯ)

### ติดตั้ง

```ruby
# Gemfile
gem "lograge", "~> 0.15"
```

```bash
bundle install
```

```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new
end
```

### เปรียบเทียบผลลัพธ์แบบรันจริง (ก่อน/หลัง)

รัน request เดิมทุกประการ (`POST /posts` สร้าง Post ใหม่) หลังเปิด `lograge` แล้ว
restart server:

**ก่อนติดตั้ง `lograge`** (จาก Step 772 — 10 บรรทัด, human-readable):

```text
Started POST "/posts" for 127.0.0.1 at 2026-09-28 18:09:46 +0000
  ActiveRecord::SchemaMigration Load (0.2ms)  SELECT ...
Processing by PostsController#create as HTML
  Parameters: {"post"=>{"title"=>"Hello", "body"=>"World"}}
  TRANSACTION (0.1ms)  BEGIN immediate TRANSACTION ...
  Post Create (0.9ms)  INSERT INTO "posts" ...
  TRANSACTION (1.1ms)  COMMIT TRANSACTION ...
Redirected to http://127.0.0.1:3099/posts/1
Completed 302 Found in 44ms (ActiveRecord: 3.5ms (1 query, 0 cached) | GC: 15.6ms)
```

**หลังติดตั้ง `lograge`** (จากการรันจริงบน request เดียวกันเป๊ะ — 1 บรรทัด, JSON):

```json
{"method":"POST","path":"/posts","format":"html","controller":"PostsController","action":"create","status":302,"allocations":20112,"duration":42.1,"view":0.0,"db":3.99,"location":"http://127.0.0.1:3099/posts/2"}
```

**สังเกต:** บรรทัด `Started`, `Processing`, `Parameters`, และ `Completed` ทั้งหมดถูกรวมเป็น
JSON object เดียว ที่มีข้อมูลเทียบเท่ากัน (`method`, `path`, `controller`, `action`, `status`,
`duration`) แถมยังมี `db` (เวลารวมของทุก SQL query) และ `view` (เวลา render view) แยกเป็นตัวเลข
ชัดเจน ซึ่งบรรทัด `Completed` แบบเดิมต้องแกะออกมาจากวงเล็บซ้อนกัน

### กลไกเบื้องหลัง — เชื่อมกับ Part 065 ที่เรียนมาแล้ว

ทบทวน Part 065 Step 647: เราเรียนรู้ **`ActiveSupport::Notifications`** ว่าเป็นกลไกที่ Rails
"broadcast" เหตุการณ์สำคัญออกมาระหว่างประมวลผล request หนึ่งครั้ง (เช่น query ฐานข้อมูล,
render view) `lograge` **subscribe เข้ากับ event ชื่อ `process_action.action_controller`**
โดยตรง ซึ่งเป็น event เดียวกับที่ `rack-mini-profiler` ใน Part 065 ใช้อ่านข้อมูลเช่นกัน — event
นี้ยิงเพียงครั้งเดียวเมื่อ controller action ทำงาน "เสร็จสมบูรณ์" (ไม่ว่าจะสำเร็จหรือ error)
พร้อม payload ที่มีข้อมูลครบทั้ง controller, action, status, view runtime, db runtime อยู่แล้ว
— `lograge` แค่หยิบ payload นั้นมาจัดรูปแบบเป็นบรรทัดเดียวแล้วพิมพ์ออกไปแทนที่ log เดิมทั้งหมด

> **ข้อควรรู้ที่ยังไม่ถูกแก้:** SQL query log แต่ละบรรทัด (`TRANSACTION`, `Post Create`) มาจาก
> **`ActiveRecord::LogSubscriber`** ซึ่งเป็นคนละ subscriber กับที่ `lograge` เข้าไปแทนที่ —
> ดังนั้นแม้เปิด `lograge` แล้ว บรรทัด SQL ดิบๆ ยังคงถูก log แยกต่างหากเหมือนเดิม (เห็นได้จากผล
> การรันจริงด้านล่างใน Step 774) ถ้าต้องการปิดให้เงียบสนิทด้วย ต้องปรับ log level แยก (Step
> 775) หรือปิด SQL logging ไปเลยด้วย `config.active_record.logger = nil` ใน production
> (แลกกับการเสียความสามารถ debug SQL ตรงๆ จาก log file)

### Formatter อื่นที่ `lograge` รองรับ

| Formatter | รูปแบบผลลัพธ์ | เหมาะกับ |
|---|---|---|
| `Lograge::Formatters::Json` | `{"method":"GET",...}` | ส่งเข้า log aggregator ที่อ่าน JSON (Step 776) |
| `Lograge::Formatters::KeyValue` | `method=GET path=/posts status=200` | รูปแบบ logfmt อ่านด้วยตาคนง่ายกว่า JSON เล็กน้อย |
| `Lograge::Formatters::Logstash` | JSON ที่ปรับ field ให้ตรงกับ Logstash/ELK stack | ใช้ร่วมกับ ELK/OpenSearch โดยตรง |

---

## Step 774: เพิ่ม custom field เข้า log (`request_id`, `current_user` id) ผ่าน `custom_payload`

log บรรทัดจาก Step 773 ยังขาดข้อมูลสำคัญสองอย่างที่จำเป็นมากตอนสืบสวนปัญหาจริง: **ใครเป็นคนยิง
request นี้** (`user_id`) และ **request นี้คือ "ตัวเดียวกัน" กับ log บรรทัดอื่นๆ ที่มาจากส่วนอื่น
ของระบบหรือไม่** (`request_id`) — `lograge` เปิดช่องให้เพิ่มฟิลด์เองผ่าน
`config.lograge.custom_payload`

```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new

  # เพิ่มฟิลด์ที่ Rails ไม่ได้ใส่มาให้ default (request_id, current_user id ฯลฯ)
  # custom_payload รับ controller instance ของ request นั้นเป็น argument จึงเรียก method
  # ของ controller ได้โดยตรง (เช่น current_user จาก Part 041/042, request.request_id)
  config.lograge.custom_payload do |controller|
    {
      request_id: controller.request.request_id,
      user_id: controller.respond_to?(:current_user, true) ? controller.send(:current_user)&.id : nil,
      ip: controller.request.remote_ip
    }
  end
end
```

ทดสอบรันจริงอีกครั้งหลังเพิ่ม `custom_payload`:

```json
{"method":"POST","path":"/posts","format":"html","controller":"PostsController","action":"create","status":302,"allocations":20112,"duration":42.1,"view":0.0,"db":3.99,"location":"http://127.0.0.1:3099/posts/2","request_id":"b5a55208-8ad0-4cc6-ad8b-22e667012792","user_id":null,"ip":"127.0.0.1"}
```

(ในแอปทดลองนี้ไม่มีระบบ authentication ติดตั้งไว้ `user_id` จึงเป็น `null` เสมอ — ในแอปจริงที่มี
Devise/`has_secure_password` จาก Part 041–042 ค่านี้จะเป็น id ของผู้ใช้ที่ล็อกอินอยู่)

**อธิบาย:**

- `controller.request.request_id` คือ UUID ที่ Rails สร้างขึ้นเองอัตโนมัติทุก request (ผ่าน
  middleware `ActionDispatch::RequestId`) — ค่าเดียวกันนี้ถูกส่งกลับไปใน HTTP response header
  `X-Request-Id` ด้วย ทำให้เวลามี bug report จากผู้ใช้ ("หน้าเว็บ error ตอน 14:32") เราขอ
  request id จาก network tab ของ browser แล้วค้นหา log บรรทัดที่ตรงกันได้ทันทีแบบเจาะจง
  ไม่ต้องไล่ดูทุกบรรทัดในช่วงเวลานั้น
- `controller.respond_to?(:current_user, true)` ตรวจสอบก่อนว่า controller นี้มี method
  `current_user` หรือไม่ (เผื่อ controller บางตัวไม่ต้อง login เลย เช่น health check) — ใส่
  `true` เป็น argument ที่สองเพื่อให้ตรวจสอบ private method ด้วย (ทบทวนความหมาย `respond_to?`
  จาก Part 009)
- **ทำไม `request_id` สำคัญกว่าที่คิด:** เมื่อระบบโตขึ้นจนมีหลาย service เชื่อมกัน (Phase 14
  จะพูดถึง microservices) การส่ง `request_id` เดียวกันผ่านทุกจุดที่ request นั้นเดินทางผ่าน
  (เรียกว่า **correlation id**) คือวิธีเดียวที่ทำให้ตามรอย request หนึ่งได้ครบวงจรข้ามระบบ
  โดยไม่หลงทาง — แนวคิดนี้เป็นจุดเริ่มต้นของสิ่งที่เรียกว่า **distributed tracing**

> **ทางเลือกอีกแบบ:** `lograge` ยังมี `config.lograge.custom_options` ที่รับ `event` (จาก
> `ActiveSupport::Notifications`) แทนที่จะรับ controller instance โดยตรง เหมาะกับกรณีที่ต้อง
> การข้อมูลจาก payload ของ event เอง (เช่น `event.payload[:exception]`) ส่วน `custom_payload`
> ที่ใช้ข้างต้นเหมาะกว่าเมื่อต้องเรียก method ของ controller ตรงๆ อย่าง `current_user`

---

## Step 775: Log level (`debug`/`info`/`warn`/`error`/`fatal`) และการตั้งค่าต่อ environment

### ลำดับความสำคัญของ log level

Ruby's `Logger` (ที่ Rails ใช้เป็นฐาน) มี severity level เรียงจากละเอียดที่สุดไปหยาบที่สุด:

| Level | ใช้เมื่อ |
|---|---|
| `debug` | ข้อมูลละเอียดยิบสำหรับ debug เท่านั้น (SQL query เต็ม, ค่า params ทุกตัว) |
| `info` | เหตุการณ์ปกติที่ควรรู้ (request เข้ามา, background job เริ่มทำงาน) |
| `warn` | สิ่งที่ยังไม่ใช่ error แต่ควรจับตา (deprecation warning, retry ครั้งที่ 2) |
| `error` | มีบางอย่างผิดพลาดแต่แอปยังทำงานต่อได้ (rescue exception ไว้แล้ว) |
| `fatal` | ข้อผิดพลาดร้ายแรงที่อาจทำให้แอปหยุดทำงาน |

เมื่อตั้ง log level เป็นระดับใดระดับหนึ่ง **เฉพาะข้อความที่ระดับนั้นขึ้นไปเท่านั้นที่จะถูก
บันทึก** เช่นตั้งเป็น `:warn` จะเห็นเฉพาะ `warn`, `error`, `fatal` — ข้อความระดับ `debug` และ
`info` จะถูกทิ้งไปเลย (ไม่ถูกเขียนลง log แม้แต่บรรทัดเดียว ไม่ใช่แค่ซ่อนไว้)

### ค่า default ที่ Rails ตั้งไว้ให้ต่อ environment (ตรวจสอบจาก generated app จริง)

```ruby
# config/environments/production.rb (สร้างโดย `rails new` เวอร์ชัน 8.1.4 จริง — ไม่ได้แก้เอง)
config.logger   = ActiveSupport::TaggedLogging.logger(STDOUT)
config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")
```

**สังเกตสองจุดสำคัญ:**

1. **`production` default คือ `info` ไม่ใช่ `debug`** — เหตุผลคือ `debug` มี volume มหาศาล
   (SQL query เต็มรูปแบบ, ทุก parameter ของทุก request) ถ้าเปิดทิ้งไว้ใน production ที่มี
   traffic สูง จะกิน storage และ I/O มหาศาลโดยไม่จำเป็น ส่วน `development` **ไม่ได้ตั้งค่า
   `log_level` ไว้เลย** จึงใช้ค่า default ของ Rails ทั่วไปคือ `debug` — สมเหตุสมผลเพราะตอน
   dev เรา*ต้องการ*เห็นทุกอย่างเพื่อ debug
2. **อ่านค่าจาก `ENV.fetch("RAILS_LOG_LEVEL", "info")`** — หมายความว่าปรับ log level ตอน
   production ได้โดยไม่ต้องแก้โค้ดหรือ deploy ใหม่เลย แค่ตั้ง environment variable
   `RAILS_LOG_LEVEL` (ทบทวนวิธีตั้งค่า env var ให้ container จาก Part 074, และวิธีส่งเข้า
   Kamal deploy config จาก Part 076) เช่นตั้งเป็น `warn` ชั่วคราวตอน traffic สูงผิดปกติเพื่อ
   ลด log volume โดยไม่กระทบโค้ดเลย

### ใช้งานใน code จริง

```ruby
Rails.logger.debug("query params: #{params.inspect}")   # เห็นเฉพาะตอน log_level = debug
Rails.logger.info("Order ##{order.id} created")           # เห็นเมื่อ log_level เป็น info ลงไป
Rails.logger.warn("Payment gateway responded slowly (#{duration}ms)")
Rails.logger.error("Failed to charge customer: #{e.message}")
Rails.logger.fatal("Database connection pool exhausted")
```

**แนวทางปฏิบัติที่แนะนำ:** ใช้ `info` เป็นค่ามาตรฐานสำหรับ production ทั่วไป (สมดุลระหว่างเห็น
เหตุการณ์สำคัญพอกับไม่เปลือง storage) และพิจารณาขยับเป็น `warn` เฉพาะระบบที่ traffic สูงมากและ
ต้นทุนเก็บ log เป็นประเด็นจริงจัง (เชื่อมกับ Step 776 เรื่องต้นทุนของ log aggregation)

---

## Step 776: แนวคิด Centralized Log Aggregation — ทำไม `tail -f` ใช้ไม่ได้อีกต่อไป

### ปัญหาที่เกิดขึ้นเมื่อ deploy ตาม Part 076

ตอนแอปรันอยู่บนเครื่องเดียว คำสั่ง `tail -f log/production.log` ก็เพียงพอสำหรับดู log สดๆ แต่
เมื่อทำตาม Part 076 (Kamal) deploy แอปเดียวกันขึ้นไปรันพร้อมกันหลาย container หรือหลายเครื่อง
เบื้องหลัง load balancer สถานการณ์เปลี่ยนไปทันที:

1. **ต้อง SSH เข้าไป `tail -f` ทีละเครื่อง** — ถ้ามี 5 instance ต้องเปิด terminal 5 หน้าต่าง
   ไล่ดูทีละเครื่องว่า request ที่มีปัญหาไปตกที่เครื่องไหน
2. **container เป็นของชั่วคราว (ephemeral filesystem)** — ผลจาก Part 073 (Docker): เมื่อ
   deploy เวอร์ชันใหม่ container เก่าจะถูกทำลายทิ้ง **พร้อมกับไฟล์ log ทั้งหมดที่อยู่ข้างในมัน**
   ถ้าไม่มีอะไรดูดข้อมูลออกไปเก็บที่อื่นก่อน หลักฐานของปัญหาที่เพิ่งเกิดเมื่อ 5 นาทีก่อน deploy
   จะหายไปตลอดกาล
3. **ค้นหาข้าม instance ไม่ได้** — ต่อให้ SSH เข้าไปดูทีละเครื่องได้ ก็ไม่มีทางเขียนคำสั่งเดียว
   ถามว่า "ทุก instance รวมกัน มี error กี่ครั้งใน 1 ชั่วโมงที่ผ่านมา"

### ทางแก้: ส่ง log ทุก instance ไปรวมศูนย์ที่เดียว

**Centralized log aggregation** คือการให้ทุก instance ส่ง log ของตัวเองไปเก็บไว้ที่ "จุดรวม"
เดียวกัน ซึ่งเปิดให้ query/ค้นหา/ตั้ง alert ได้จากที่เดียว โดยไม่สนใจว่า log บรรทัดนั้นมาจาก
เครื่องไหน

เครื่องมือที่นิยมในวงการ (แนะนำแค่ระดับรู้จัก ไม่ลงลึกตัวใดตัวหนึ่งเป็นพิเศษ เพราะแต่ละทีมเลือก
ตามงบประมาณและ infrastructure ที่มีอยู่แล้ว):

| เครื่องมือ | ลักษณะ |
|---|---|
| **CloudWatch Logs** (AWS) | ผูกกับ AWS โดยตรง เหมาะถ้า infra อยู่บน AWS อยู่แล้ว |
| **Datadog** | แพลตฟอร์ม observability ครบวงจร (log + metric + trace + APM) ราคาสูงกว่าแต่ครบ |
| **Papertrail** | เรียบง่าย ราคาถูก เหมาะกับทีมเล็ก/สตาร์ทอัพ |
| **Better Stack** (เดิมชื่อ Logtail) | UI ทันสมัย มี uptime monitoring ในตัวด้วย (เกี่ยวกับ Step 779) |
| **ELK / OpenSearch Stack** | self-hosted (Elasticsearch + Logstash/Fluentd + Kibana) ควบคุมเองได้เต็มที่ แต่ต้องดูแลเอง |

### ทำไม Rails 8 เปลี่ยนมา log ออก STDOUT แทนไฟล์

สังเกตจาก Step 775 อีกครั้งว่า `production.rb` ที่ Rails generate ให้ตั้งแต่เวอร์ชันปัจจุบันคือ

```ruby
config.logger = ActiveSupport::TaggedLogging.logger(STDOUT)
```

**ไม่ใช่เขียนลงไฟล์ `log/production.log` เหมือนเวอร์ชันเก่าๆ ของ Rails** นี่ไม่ใช่เรื่องบังเอิญ
— เป็นไปตามหลักการ **"Logs as event streams"** จาก [12-Factor App](https://12factor.net/logs)
ที่บอกว่าแอปไม่ควรสนใจเลยว่า log ของตัวเองจะถูกเก็บที่ไหนหรือ routing ไปทางไหน หน้าที่ของแอปคือ
**พ่น log ออกทาง STDOUT ให้ไวที่สุด** ส่วนการ "ดูด" ออกไปเก็บที่ไหนต่อเป็นหน้าที่ของ
**สภาพแวดล้อมที่รันแอปอยู่** (execution environment) ไม่ใช่หน้าที่ของโค้ดแอปเอง

สอดคล้องพอดีกับ Part 073 (Docker) และ Part 076 (Kamal): เมื่อแอปรันเป็น container, Docker log
driver หรือ agent ภายนอก (CloudWatch agent, Datadog agent, Vector, Fluentd ฯลฯ) จะดักจับ STDOUT
ของทุก container โดยอัตโนมัติแล้วส่งต่อไปยังปลายทางที่ตั้งค่าไว้ **โดยที่โค้ด Rails ไม่ต้องรู้จัก
หรือ config อะไรเกี่ยวกับปลายทางนั้นเลยแม้แต่บรรทัดเดียว** — นี่คือเหตุผลเบื้องหลังที่แท้จริงว่า
ทำไม default ของ Rails ถึงเปลี่ยนมาเป็น STDOUT ในยุคที่ Docker-first deployment (แบบ Kamal)
กลายเป็นมาตรฐาน

> **ข้อควรระวังเรื่องต้นทุน:** log aggregator เกือบทุกเจ้าคิดค่าบริการตามปริมาณ log ที่ส่งเข้าไป
> (GB ต่อเดือน) — นี่คือเหตุผลที่ Step 775 (เลือก log level ให้เหมาะสม) และ Step 773 (ย่อ 10
> บรรทัดให้เหลือ 1 บรรทัดด้วย `lograge`) ไม่ใช่แค่เรื่องความสะดวกในการอ่าน แต่ส่งผลต่อค่าใช้จ่าย
> จริงของระบบโดยตรงเมื่อ traffic โตขึ้น

---

## Step 777: Error Tracking ด้วย Sentry — ติดตั้ง, `Sentry.init`, automatic exception capture

### ทำไม log อย่างเดียวไม่พอ ต้องมี error tracker แยกต่างหาก

Log (แม้จะ structured แล้วด้วย `lograge`) ตอบคำถาม "เกิดอะไรขึ้นบ้างตามลำดับเวลา" ได้ดี แต่ไม่ได้
ออกแบบมาเพื่อตอบคำถามเฉพาะทางเกี่ยวกับ exception เช่น:

- exception ตัวนี้เกิดขึ้น**กี่ครั้ง**ในสัปดาห์ที่ผ่านมา เพิ่มขึ้นหรือลดลง?
- นี่คือ exception **ตัวเดิม**ที่เคยเกิดเมื่อวานหรือเป็นตัวใหม่ที่ไม่เคยเจอมาก่อน?
- ใครควรได้รับแจ้งเตือนทันทีที่ error ใหม่เกิดขึ้นเป็นครั้งแรก?

**Error tracking service** (เช่น Sentry) ถูกออกแบบมาตอบคำถามกลุ่มนี้โดยเฉพาะ — มันจะ
**จัดกลุ่ม (group)** exception ที่มี stack trace คล้ายกันให้เป็น "issue" เดียว นับจำนวนครั้งที่
เกิดซ้ำ, แจ้งเตือนอัตโนมัติผ่าน Slack/email ทันทีที่มี issue ใหม่หรือ issue เดิมเกิดถี่ผิดปกติ,
และเก็บ stack trace พร้อม context แวดล้อม (browser, request params, user) ไว้ให้ดูย้อนหลังได้
ครบถ้วนกว่าที่จะ grep เอาจาก log ธรรมดา

### ติดตั้ง `sentry-ruby` และ `sentry-rails`

```ruby
# Gemfile
gem "sentry-ruby", "~> 5.28"
gem "sentry-rails", "~> 5.28"
```

```bash
bundle install
```

```ruby
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"] # อ่านจาก credentials/ENV เสมอ ไม่ hardcode ในโค้ด (ทบทวน Part 074)
  config.breadcrumbs_logger = [:active_support_logger]

  # ส่ง environment ปัจจุบันไปด้วย เพื่อแยก error จาก development/staging/production ใน dashboard
  config.environment = Rails.env

  # สัดส่วน transaction ที่เก็บสำหรับ performance monitoring (1.0 = เก็บ 100%)
  # production จริงมักตั้งต่ำกว่านี้มาก เพื่อลดค่าใช้จ่ายและ overhead
  config.traces_sample_rate = Rails.env.production? ? 0.1 : 1.0

  # กัน PII หลุดไปกับ error report โดยไม่ตั้งใจ (รายละเอียดเต็มใน Step 778)
  config.send_default_pii = false
end
```

### สิ่งที่ verify ได้จริงในสภาพแวดล้อมทดลอง กับสิ่งที่ verify ไม่ได้

sandbox นี้**ไม่มีบัญชี Sentry.io จริงและไม่มี DSN จริง** (ตั้งใจปล่อยให้ `SENTRY_DSN` เป็น
`nil`) — นี่คือสถานการณ์ที่ตรงกับความเป็นจริงตอนพัฒนา: นักพัฒนาส่วนใหญ่ไม่มีสิทธิ์เข้าถึง Sentry
production account ของบริษัทตอนเขียนโค้ดบนเครื่องตัวเอง **สิ่งที่ยืนยันได้จริงและมีประโยชน์คือ
โค้ด integration ถูกต้อง ไม่ทำให้แอปพังแม้ไม่มี DSN**

boot server แล้วสังเกต log:

```text
=> Booting Puma
=> Rails 8.1.4 application starting in development
Initializing the Sentry background worker with 2 threads
[Sessions] Sessions won't be captured without a valid release
```

**สองบรรทัดสุดท้ายพิสูจน์ว่า `Sentry.init` โหลดและ config สำเร็จ** — ไม่มี exception ตอน boot,
gem รู้ตัวเองว่าไม่มี DSN ที่ใช้งานได้จริง (จึงเตือนว่า sessions จะไม่ถูกส่ง) แต่**ไม่ทำให้แอปหยุด
ทำงาน** — พฤติกรรมนี้เป็นการออกแบบที่ตั้งใจของ Sentry SDK: ถ้าไม่มี DSN มันจะทำงานเป็น "no-op"
เงียบๆ แทนที่จะ raise error ทำให้ปลอดภัยสำหรับ environment ที่ยังไม่ได้ตั้งค่า Sentry จริง (เช่น
เครื่อง dev ของนักพัฒนาแต่ละคน)

ทดสอบยิง unrescued exception จริงเพื่อยืนยันว่า sentry-rails ไม่รบกวนการทำงานปกติของ Rails:

```bash
curl -o /dev/null -w "crash:%{http_code}\n" http://127.0.0.1:3099/posts/crash
# => crash:500
```

แอปตอบกลับ **500 ตามปกติของ Rails เอง** (แสดงหน้า error page ตามที่ Part 011 สอนไว้) — ยืนยัน
ว่า middleware/Railtie ของ `sentry-rails` ที่แทรกตัวเข้าไปดักจับ exception อัตโนมัติ**อยู่ร่วม
กับกลไก error handling ปกติของ Rails ได้โดยไม่ชนกัน** และไม่ทำให้ request ค้างหรือแอป crash เพิ่ม

**สิ่งที่ยืนยันไม่ได้ในสภาพแวดล้อมนี้ และต้องบอกตรงๆ:** การเห็น error `crash` ตัวนี้ปรากฏขึ้นจริง
เป็น issue ใหม่ในหน้า dashboard ของ Sentry.io — เพราะไม่มี DSN จริงเชื่อมต่ออยู่ เมื่อคุณทำตาม
Part นี้ในเครื่องของตัวเอง ขั้นตอนที่ต้องทำเองเพิ่มเติมคือ:

1. สมัครบัญชีที่ [sentry.io](https://sentry.io) (มี free tier เพียงพอสำหรับโปรเจกต์เรียนรู้)
2. สร้าง project ใหม่ เลือกแพลตฟอร์ม "Ruby on Rails"
3. คัดลอกค่า DSN ที่ได้ ไปเก็บไว้ใน Rails credentials หรือ environment variable
   `SENTRY_DSN` (ทบทวนวิธีจัดการ secret อย่างปลอดภัยจาก Part 074 — **ห้าม commit DSN ลง
   git โดยตรง** แม้ DSN จะไม่ใช่ความลับระดับสูงเท่า API key แต่ก็ไม่ควร hardcode ในโค้ด)
4. deploy หรือรันแอปแล้วลองทำให้เกิด error จริง — คราวนี้จะเห็น issue ปรากฏในหน้า dashboard
   ภายในไม่กี่วินาที พร้อม stack trace เต็มรูปแบบ

### Automatic exception capture ทำงานอย่างไร

`sentry-rails` ติดตั้งตัวเองเป็น Railtie ที่ hook เข้ากับ chain การจัดการ exception ของ Rails
โดยอัตโนมัติทันทีที่ gem ถูกโหลด (ไม่ต้องเขียนโค้ดเพิ่มเติมเลยสักบรรทัด) — **ทุก exception ที่ไม่
ได้ถูก `rescue` ไว้ในโค้ดของเรา** (จะกลายเป็นหน้า error 500 ตามปกติ) จะถูกส่งเข้า Sentry
โดยอัตโนมัติไปพร้อมกัน นี่คือสิ่งที่ทดสอบยืนยันแล้วข้างต้นด้วย action `crash`

> **คำถามสำคัญที่ตามมา:** แล้ว exception ที่เรา `rescue` ไว้เองล่ะ (เช่น action `boom` ที่
> rescue `ZeroDivisionError` แล้วแสดงผลปกติให้ user เห็น) — Sentry จะรู้ไหมว่ามันเกิดขึ้น?
> **คำตอบคือไม่รู้โดยอัตโนมัติ** เพราะเมื่อ rescue ไว้แล้ว Rails ก็ถือว่า request นั้น "จบแบบ
> ปกติ" ไม่มี exception หลุดออกไปให้ automatic capture จับได้อีก — นี่คือเหตุผลที่ต้องมี Step
> 778 ต่อไป

---

## Step 778: รายงาน exception ที่ rescue ไว้ด้วยมือ + เพิ่ม context อย่างปลอดภัย

### เมื่อไหร่ต้อง manual capture

ทบทวน Part 011 (Exception Handling): เรา `rescue` exception เพื่อให้แอปทำงานต่อได้อย่างสวยงาม
แทนที่จะแสดงหน้า error ดิบๆ ให้ผู้ใช้เห็น — แต่นั่นหมายความว่า **เราตั้งใจซ่อนปัญหาจากผู้ใช้**
ซึ่งดีต่อ user experience แต่ถ้าซ่อนจากทีมพัฒนาไปด้วยโดยไม่ตั้งใจ ปัญหาที่เกิดซ้ำๆ อยู่เบื้องหลัง
จะไม่มีใครรู้เลยจนกว่าจะสายเกินไป (เช่น ระบบชำระเงินจาก Part 071 ล้มเหลวเงียบๆ ซ้ำหลายครั้งโดยที่
ผู้ใช้แค่เห็นข้อความ "กรุณาลองใหม่อีกครั้ง" — ไม่มีใครในทีมรู้ว่ามันเกิดขึ้นบ่อยแค่ไหน)

`Sentry.capture_exception` แก้ปัญหานี้: รายงาน exception ไปที่ Sentry **โดยที่ยังคง flow การ
rescue และแสดงผลปกติให้ผู้ใช้ไว้เหมือนเดิมทุกประการ**

```ruby
# app/controllers/posts_controller.rb (ทดสอบรันจริงแล้ว)
def boom
  1 / 0
rescue ZeroDivisionError => e
  Sentry.capture_exception(e)
  Rails.logger.error("Rescued in #boom: #{e.class}: #{e.message}")
  render plain: "handled", status: :ok
end
```

```bash
curl -o /dev/null -w "boom:%{http_code}\n" http://127.0.0.1:3099/posts/boom
# => boom:200   -- ผู้ใช้เห็นผลลัพธ์ปกติ (200 OK) แม้เบื้องหลังจะเกิด error จริง
```

log ที่ได้ (ผสมกันระหว่าง `Rails.logger.error` กับบรรทัดสรุปของ `lograge`):

```text
Rescued in #boom: ZeroDivisionError: divided by 0
{"method":"GET","path":"/posts/boom","format":"*/*","controller":"PostsController","action":"boom","status":200,"allocations":628,"duration":1.55,"view":1.04,"db":0.0,"request_id":"ebb5ff03-be4f-48ba-a40e-cffbc86624f5","user_id":null,"ip":"127.0.0.1"}
```

สังเกตว่า `status` ใน log เป็น `200` (เพราะเรา rescue แล้วตอบกลับปกติ) — ถ้าไม่มี
`Sentry.capture_exception` เรียกไว้ ไม่มีทางเลยที่จะรู้จาก log บรรทัดนี้ว่ามี exception เกิดขึ้น
ระหว่างทาง `Rails.logger.error` ช่วยได้ระดับหนึ่ง แต่ไม่มี dashboard ที่นับจำนวนครั้งหรือแจ้งเตือน
อัตโนมัติให้เหมือน Sentry

### เพิ่ม context ให้ error report มีประโยชน์มากขึ้น

Sentry ให้ context เพิ่มเติมนอกเหนือจากตัว exception เองได้ เพื่อให้ตอนสืบสวนภายหลังรู้ทันทีว่า
"ใคร" และ "ทำอะไร" ตอนที่ error เกิด (ทดสอบรันจริงผ่าน `bin/rails runner` — ยืนยันว่าไม่ crash
แม้ไม่มี DSN):

```ruby
Sentry.set_user(id: 42)
Sentry.set_context("request_meta", { path: "/posts", method: "GET" })
Sentry.capture_message("test message from runner")
```

ผลการรันจริง: `OK - no crash` — ทั้งสาม method ทำงานได้ปกติแม้ไม่มี DSN จริง (เป็น no-op
เงียบๆ เช่นเดียวกับ `capture_exception` ใน Step 777)

### คำเตือนความปลอดภัยที่สำคัญมาก: อย่าให้ PII/secrets หลุดไปกับ error report

นี่คือความเสี่ยงด้านความปลอดภัยที่เกิดขึ้นจริงบ่อยมากในทีมที่เพิ่งเริ่มใช้ error tracker — เพราะ
ความตั้งใจดี (อยากเก็บ context ให้ครบเพื่อ debug ง่าย) กลับกลายเป็นช่องโหว่โดยไม่รู้ตัว:

1. **`config.send_default_pii = false`** (ตั้งไว้แล้วใน Step 777) — ค่านี้ป้องกันไม่ให้ Sentry
   SDK เก็บข้อมูลอ่อนไหวแบบอัตโนมัติ เช่น IP address เต็ม, cookie header, request body ดิบๆ
   ถ้าตั้งเป็น `true` (ซึ่งเป็นค่า default ของบาง SDK version เก่า) ข้อมูลอ่อนไหวเหล่านี้จะถูก
   ส่งไปเก็บที่ Sentry โดยอัตโนมัติทุกครั้งที่มี error โดยไม่มีใครในทีมรู้ตัว
2. **ระวัง `params` ที่ manual ส่งเข้า `set_context` เอง** — ถ้าเขียนโค้ดแบบ
   `Sentry.set_context("params", params.to_unsafe_h)` ตรงๆ โดยไม่กรองก่อน จะมีความเสี่ยงสูงมาก
   ที่ field อย่าง `password`, `password_confirmation`, `credit_card_number`, `api_key` จะหลุด
   ไปอยู่ใน error report ที่ทุกคนในทีม (รวมถึงคนนอกทีมที่มีสิทธิ์ดู Sentry) มองเห็นได้
3. **ใช้ `config.before_send` เป็นชั้นกรองสุดท้ายก่อนส่งออก** — Rails เองมีกลไก
   `config.filter_parameters` อยู่แล้วสำหรับกรอง log (ตั้งค่า default ให้กรอง `password` ให้
   อัตโนมัติ) Sentry มีกลไกคล้ายกันชื่อ `before_send` ที่เรียกก่อนส่ง event ทุกครั้ง ใช้ scrub
   ข้อมูลอ่อนไหวออกได้ทันที:

```ruby
# config/initializers/sentry.rb (เพิ่มเข้าไปจาก Step 777)
Sentry.init do |config|
  # ... config เดิมจาก Step 777 ...

  config.before_send = lambda do |event, _hint|
    sensitive_keys = %w[password password_confirmation credit_card token api_key]

    event.request&.data&.each do |key, _value|
      event.request.data[key] = "[FILTERED]" if sensitive_keys.include?(key.to_s)
    end

    event
  end
end
```

> **preview Phase 13:** เรื่อง PII, secrets, และการป้องกันข้อมูลรั่วไหลจะถูกเจาะลึกอีกครั้งใน
> Part 081 (Secrets management, encryption) — Part นี้แค่ปูพื้นว่า "จุดที่ข้อมูลอ่อนไหวหลุดได้
> ง่ายที่สุด" จุดหนึ่งคือ error tracking tool ที่ทีมส่วนใหญ่มองข้าม เพราะคิดว่าเป็นแค่
> "เครื่องมือ debug ภายใน" ทั้งที่จริงแล้วมันคือช่องทางส่งข้อมูลออกไปยัง third-party service
> เหมือนกับ Part 071 (Stripe) หรือ Part 072 (OAuth) ทุกประการ

---

## Step 779: Uptime Monitoring ภายนอก + เจาะลึก endpoint `/up` ของ Rails 8

### แนวคิด Uptime Monitoring

log และ error tracker (Step 772–778) ตอบคำถามได้ดีเมื่อมี request เข้ามาแล้วเกิดปัญหา แต่ถ้า
**ทั้งแอปล่มไปเลย ไม่มี request ไหนเข้าถึงได้อีกต่อไป** — จะไม่มี log หรือ error ใหม่เกิดขึ้นเลย
เพราะไม่มีอะไรทำงานให้เกิด log! นี่คือสถานการณ์ที่ต้องอาศัยเครื่องมือคนละแบบ: **uptime monitor**
บริการภายนอกที่คอยยิง request ไปหาแอปของเราเป็นระยะ (เช่น ทุก 1 นาที) ถ้าไม่ได้รับคำตอบ 200
ภายในเวลาที่กำหนด หรือ timeout เกิดขึ้น จะแจ้งเตือนทันที (SMS, email, Slack, หรือกระทั่งโทรศัพท์
โทรหา)

**ทำไมต้องเป็นบริการ "ภายนอก" (external) เท่านั้น:** ถ้าระบบ monitor อยู่ใน infrastructure
เดียวกับแอป (เช่น รันบน server เดียวกัน) แล้วทั้ง network หรือ data center ของเราล่มทั้งหมด ระบบ
monitor เองก็จะล่มตามไปด้วย — ไม่มีใครแจ้งเตือนเลยตอนที่ควรแจ้งเตือนมากที่สุด หลักการพื้นฐานของ
uptime monitoring คือต้องสังเกตการณ์จาก**มุมมองเดียวกับผู้ใช้จริง** คือจากภายนอกเครือข่ายของเรา

บริการที่นิยม (แนะนำแค่ระดับรู้จัก ตามขอบเขตของ Part นี้):

| บริการ | จุดเด่น |
|---|---|
| **UptimeRobot** | มี free tier ใจกว้าง (check ได้ 50 endpoint ทุก 5 นาที) เหมาะกับโปรเจกต์เรียนรู้/เริ่มต้น |
| **Pingdom** | เก่าแก่ เชื่อถือได้ มี performance monitoring เสริม |
| **Better Uptime** | UI ทันสมัย รวม incident management/status page ในตัว |
| **StatusCake** | คล้าย UptimeRobot มี feature ตรวจ SSL certificate expiry ด้วย |

### Rails 8 มี health check endpoint มาให้แล้วโดย default — verify จริง

ข่าวดีคือ Rails 8 ไม่ต้องสร้าง endpoint สำหรับ uptime monitor เอง — ตรวจสอบไฟล์ `routes.rb`
ที่ `rails new` สร้างให้ (verify จริง ไม่ได้เพิ่มเอง):

```ruby
# config/routes.rb (บรรทัดที่ rails new สร้างให้อัตโนมัติ)
# Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
# Can be used by load balancers and uptime monitors to verify that the app is live.
get "up" => "rails/health#show", as: :rails_health_check
```

ทดสอบยิงจริง:

```bash
curl -o /dev/null -w "up:%{http_code}\n" http://127.0.0.1:3099/up
# => up:200
```

### `/up` เช็คอะไรบ้างจริงๆ — อ่านจาก source code ของ Rails เอง

เปิดดู source จริงของ `Rails::HealthController` (จาก gem `railties` เวอร์ชัน 8.1.4 ที่ติดตั้ง
อยู่ในเครื่อง ไม่ใช่คำอธิบายจากความจำ):

```ruby
# railties-8.1.4/lib/rails/health_controller.rb
class HealthController < ActionController::Base
  rescue_from(Exception) { render_down }

  def show
    render_up
  end

  private
    def render_up
      respond_to do |format|
        format.html { render html: html_status(color: "green") }
        format.json { render json: { status: "up", timestamp: Time.current.iso8601 } }
      end
    end

    def render_down
      respond_to do |format|
        format.html { render html: html_status(color: "red"), status: 500 }
        format.json { render json: { status: "down", timestamp: Time.current.iso8601 }, status: 500 }
      end
    end
end
```

จาก comment ต้นไฟล์ (เขียนไว้โดยทีม Rails เอง):

> "This endpoint will return a 200 status code if the app has booted with no exceptions, and
> a 500 status code otherwise... **This endpoint does not reflect the status of all of your
> application's dependencies, such as the database or Redis cluster.**"

พูดง่ายๆ คือ `/up` เช็คแค่ **"Rails process นี้ยังมีชีวิตอยู่ และ boot สำเร็จโดยไม่มี
exception หลุดออกมา"** เท่านั้น — มันไม่เช็คว่า database ต่อได้ไหม, Redis (Part 062) ต่อได้ไหม,
Sidekiq queue ทำงานอยู่ไหม, หรือ external API อย่าง Stripe (Part 071) ตอบสนองอยู่ไหม

### ทำไมการเช็คแบบ "ตื้น" (shallow) นี้ถึงเป็นการออกแบบที่ตั้งใจ ไม่ใช่ข้อจำกัด

comment ในไฟล์เดียวกันเตือนไว้ตรงๆ ว่า:

> "Think carefully about what you want to check as it can lead to situations where your
> application is being restarted due to a third-party service going bad."

นี่คือ trade-off สำคัญที่ต้องเข้าใจ: ถ้า health check เช็คลึกถึง database/Redis/external API
แล้วบริการเหล่านั้นเกิดช้าหรือล่มชั่วคราว (ซึ่งเป็นปัญหาที่แอปควรจัดการแบบ graceful ไม่ใช่ล้มไป
ด้วย) load balancer หรือ orchestrator (Kubernetes, Kamal) ที่เฝ้าดู health check อยู่จะเข้าใจผิด
ว่า **instance ของแอปเองตายไปด้วย** แล้วสั่ง restart รัวๆ ซ้ำแล้วซ้ำเล่า — กลายเป็นการซ้ำเติม
ปัญหาให้แย่ลงไปอีก (เรียกว่า **cascading failure**) แทนที่จะช่วยแก้ปัญหา

### แยก Liveness กับ Readiness — ขยาย health check เองเมื่อจำเป็น

ถ้าต้องการเช็คลึกกว่านี้จริงๆ (เช่น สำหรับ uptime monitor ภายนอกที่อยากรู้ว่า "ระบบพร้อมให้
บริการเต็มรูปแบบจริงหรือไม่") แนวทางที่ถูกต้องคือ**แยก endpoint ใหม่ต่างหาก** ไม่ไปแก้ `/up`
เดิม เพื่อให้ load balancer ยังใช้ `/up` (ตื้น, เร็ว, ปลอดภัยจาก cascading failure) ต่อไปได้ตาม
เดิม:

```ruby
# app/controllers/health_controller.rb
class HealthController < ActionController::Base
  def deep
    checks = { database: database_ok?, cache: cache_ok? }

    if checks.values.all?
      render json: { status: "up", checks: checks }
    else
      render json: { status: "down", checks: checks }, status: :service_unavailable
    end
  end

  private
    def database_ok?
      ActiveRecord::Base.connection.execute("SELECT 1")
      true
    rescue StandardError
      false
    end

    def cache_ok?
      Rails.cache.write("health_check", "1", expires_in: 5.seconds)
      Rails.cache.read("health_check") == "1"
    rescue StandardError
      false
    end
end
```

```ruby
# config/routes.rb
get "up/deep" => "health#deep"
```

| Endpoint | ความหมาย | ใครควรเรียก |
|---|---|---|
| `/up` (default ของ Rails) | **Liveness** — process ยังมีชีวิตอยู่ไหม | Load balancer, Kamal, Docker healthcheck (เช็คถี่ ต้องเร็วและเบา) |
| `/up/deep` (สร้างเอง) | **Readiness** — dependency สำคัญพร้อมใช้งานจริงไหม | Uptime monitor ภายนอก (UptimeRobot ฯลฯ) เท่านั้น (เช็คห่างกว่า และยอมรับได้ถ้าช้ากว่าเล็กน้อย) |

---

## Step 780: Incident Response Mindset + แบบฝึกหัดปิด Phase 12

### เมื่อ alert ดัง ควรทำอะไรตามลำดับ

ทุกเครื่องมือที่เรียนมาใน Part นี้ (log, Sentry, uptime monitor) มีประโยชน์สูงสุดตอนที่มันทำงาน
ร่วมกันเป็นกระบวนการเดียว ไม่ใช่แยกกันคนละเครื่องมือ นี่คือลำดับขั้นตอนมาตรฐานที่ทีม engineering
มืออาชีพใช้เมื่อมี alert ดังขึ้นตอนตี 3:

1. **ยืนยันว่าเป็นปัญหาจริง ไม่ใช่ false positive** — เช็คจากมากกว่าหนึ่งแหล่งถ้าเป็นไปได้ (เช่น
   uptime monitor แจ้งพร้อมกับ error rate ใน Sentry พุ่งขึ้นจริง ไม่ใช่แค่ network hiccup
   ชั่วครู่ของ monitor เอง)
2. **เช็ค log ที่รวมศูนย์ไว้แล้ว (Step 776)** — หาช่วงเวลาที่ปัญหาเริ่มเกิดแม่นยำที่สุด ดู error
   rate และ pattern ว่ากระจุกอยู่ที่ controller/action ไหนเป็นพิเศษหรือเกิดทั่วทั้งระบบ
3. **เช็ค error tracker (Sentry, Step 777–778)** — ดู stack trace ล่าสุดของ issue ที่เกี่ยวข้อง
   ว่าเป็น exception ตัวใหม่ที่ไม่เคยเจอมาก่อน หรือเป็นตัวเดิมที่แค่เกิดถี่ขึ้นผิดปกติ ดู context
   (user, params ที่กรองแล้ว) ที่แนบมาเพื่อหา pattern ร่วม
4. **เช็ค recent deploy (เชื่อม Part 075 CI/CD และ Part 076 Kamal)** — เวลาที่ปัญหาเริ่มตรงกับ
   เวลา deploy ล่าสุดหรือไม่? นี่คือคำถามแรกๆ ที่ควรถามเสมอ เพราะสถิติจริงในวงการชี้ว่า
   **incident ส่วนใหญ่ในระบบที่เพิ่งเปลี่ยนแปลงมักมาจากการเปลี่ยนแปลงนั้นเอง** ไม่ใช่เหตุบังเอิญ
5. **ถ้าตรงกับ deploy ล่าสุด → rollback ทันที** ก่อนสืบสวนสาเหตุแบบละเอียด — ใช้ `kamal rollback`
   (Part 076) หรือ revert commit แล้วปล่อยผ่าน CI/CD pipeline อีกรอบ (Part 075) หลักการสำคัญคือ
   **"กู้คืนระบบก่อน วิเคราะห์สาเหตุทีหลัง"** (recovery first, root-cause analysis later) เมื่อ
   production กำลังมีปัญหาจริง ยิ่งปล่อยไว้นานผู้ใช้ยิ่งเดือดร้อนมากขึ้น การนั่งหาสาเหตุให้ชัดเจน
   100% ก่อน deploy แก้เป็นความคิดที่ผิดในสถานการณ์ฉุกเฉิน
6. **ถ้าไม่ตรงกับ deploy** → มักเป็นปัญหาจาก dependency ภายนอก (เช่น Stripe จาก Part 071 ล่ม,
   database เต็ม, third-party API ที่ Part 072 เชื่อมไว้ตอบช้าผิดปกติ) หรือ data-specific bug
   ที่ trigger จากข้อมูลบางชุดพอดี — context ที่เก็บไว้ใน Sentry (Step 778) มีประโยชน์มากที่สุด
   ตรงจุดนี้
7. **เขียน postmortem แบบไม่กล่าวโทษ (blameless)** หลังสถานการณ์คลี่คลาย — บันทึกไทม์ไลน์ของ
   เหตุการณ์, สาเหตุที่แท้จริง, และสิ่งที่จะป้องกันไม่ให้เกิดซ้ำ ไม่ใช่เพื่อหาคนผิด แต่เพื่อให้
   ทีมเรียนรู้ร่วมกัน (จะเจาะลึกการเขียนเอกสารลักษณะนี้อย่างมืออาชีพใน Part 097)

### ตารางสรุป Incident Response Checklist

| ลำดับ | คำถาม | เครื่องมือที่ใช้ |
|---|---|---|
| 1 | จริงหรือ false positive? | Uptime monitor + Sentry เทียบกัน |
| 2 | เริ่มเมื่อไหร่ กระจุกตรงไหน? | Log aggregator (Step 776) |
| 3 | exception อะไร ใหม่หรือเก่า? | Sentry issue + context (Step 777–778) |
| 4 | ตรงกับ deploy ล่าสุดไหม? | CI/CD deploy history (Part 075), Kamal (Part 076) |
| 5 | ตรง → ทำอะไร? | `kamal rollback` ทันที |
| 6 | ไม่ตรง → ทำอะไร? | ตรวจ dependency ภายนอก + data เฉพาะจุด |
| 7 | หลังจบเหตุการณ์ → ทำอะไร? | เขียน blameless postmortem |

---

### แบบฝึกหัด: เพิ่ม Monitoring เต็มรูปแบบให้แอป Rails ที่มีอยู่

**โจทย์:** สมมติมีแอป Rails ที่ deploy ตาม Part 073–077 ไปแล้ว (เช่น Blog app จาก Part 030 หรือ
ระบบร้านค้าจาก Part 040) ให้เพิ่ม monitoring 3 อย่างต่อไปนี้ให้ครบ:

1. เพิ่ม `lograge` พร้อม custom field `request_id` และ `user_id`
2. ติดตั้ง `sentry-ruby`/`sentry-rails` (verify ว่าโหลดได้โดยไม่ทำให้แอปพัง แม้ยังไม่มี DSN จริง)
3. ยืนยันว่า endpoint `/up` ทำงานถูกต้อง

### เฉลย

**ขั้นที่ 1 — Gemfile**

```ruby
# Gemfile
gem "lograge", "~> 0.15"
gem "sentry-ruby", "~> 5.28"
gem "sentry-rails", "~> 5.28"
```

```bash
bundle install
```

**ขั้นที่ 2 — ตั้งค่า `lograge`**

```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new

  config.lograge.custom_payload do |controller|
    {
      request_id: controller.request.request_id,
      user_id: controller.respond_to?(:current_user, true) ? controller.send(:current_user)&.id : nil
    }
  end
end
```

**ขั้นที่ 3 — ตั้งค่า Sentry**

```ruby
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV["SENTRY_DSN"]
  config.breadcrumbs_logger = [:active_support_logger]
  config.environment = Rails.env
  config.traces_sample_rate = Rails.env.production? ? 0.1 : 1.0
  config.send_default_pii = false

  config.before_send = lambda do |event, _hint|
    sensitive_keys = %w[password password_confirmation credit_card token api_key]
    event.request&.data&.each do |key, _value|
      event.request.data[key] = "[FILTERED]" if sensitive_keys.include?(key.to_s)
    end
    event
  end
end
```

**ขั้นที่ 4 — verify ว่าใช้งานได้จริง**

```bash
# 1) boot server แล้วดูว่า Sentry โหลดสำเร็จไม่ error
bin/rails server
# ควรเห็น: "Initializing the Sentry background worker with 2 threads"

# 2) ยิง request ปกติ แล้วดูว่า log กลายเป็น JSON บรรทัดเดียว พร้อม request_id/user_id
curl http://localhost:3000/posts

# 3) verify /up
curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/up
# ควรได้ 200
```

**ผลที่ควรเห็น (อ้างอิงจากผลจริงที่ทดสอบไว้ตลอด Part นี้):**

```json
{"method":"GET","path":"/posts","format":"html","controller":"PostsController","action":"index","status":200,"allocations":...,"duration":...,"view":...,"db":...,"request_id":"<uuid>","user_id":null}
```

ยืนยันครบทั้ง 3 อย่างตามโจทย์: log เป็น structured JSON พร้อม custom field, Sentry โหลดไม่พัง,
และ `/up` ตอบ 200

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **สร้าง readiness endpoint แยกจาก liveness** — เขียน `/up/deep` ที่เช็ค database connection
   จริง (`ActiveRecord::Base.connection.execute("SELECT 1")`) และถ้าแอปมี Sidekiq/Redis จาก
   Part 062 ให้เช็คการเชื่อมต่อ Redis ด้วย โดย**ต้องแยก route ออกจาก `/up` เดิมเสมอ** ตาม
   หลักการ liveness/readiness ที่สอนใน Step 779 (ห้ามแก้ `Rails::HealthController` เดิม)
2. **กรอง exception ที่ไม่สำคัญออกจาก Sentry** — ใช้ `config.excluded_exceptions` เพิ่ม
   `ActiveRecord::RecordNotFound` เข้าไป (exception นี้มักเกิดจากผู้ใช้เดา URL สุ่มๆ ไม่ใช่บั๊ก
   จริงในระบบ) แล้วทดสอบว่า route ที่ raise exception นี้ยังคงตอบ 404 ตามปกติ แต่ไม่ไปสร้าง
   issue รกใน Sentry dashboard
3. **ออกแบบ alert threshold ของทีมตัวเอง (เชิงเอกสาร ไม่ต้องมี Sentry account จริง)** — เขียน
   แผนสั้นๆ ว่าจะตั้งกฎแจ้งเตือนอย่างไรถ้า error rate เกิน X% ภายใน 5 นาที โดยอ้างอิงหลักการ
   Step 780 (ยืนยันก่อนตื่นตระหนก → เช็ค log → เช็ค Sentry → เช็ค deploy → ตัดสินใจ
   rollback) ระบุด้วยว่าใครควรเป็นคนรับ alert ก่อน และเมื่อไหร่ควร escalate ไปหาคนถัดไป

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Observability** จำเป็นเพราะไม่สามารถแนบ debugger เข้ากับ production ได้เหมือน
  ตอน dev และแตกต่างจาก Part 065 (profiling) ตรงที่เป็นการเฝ้าดูแบบต่อเนื่องตลอดเวลา ไม่ใช่
  วัดครั้งเดียวตอนสงสัย
- เข้าใจข้อจำกัดของ Rails default logger (กระจายหลายบรรทัดต่อ request, human-readable ไม่ใช่
  machine-parseable) และกลไก `config.log_formatter` เบื้องหลัง
- ติดตั้งและใช้งาน **`lograge`** จริง เห็นความต่างของ log ก่อน/หลังแบบเป็นรูปธรรม (10 บรรทัด
  เหลือ 1 บรรทัด JSON) และเข้าใจว่ามันทำงานผ่าน `ActiveSupport::Notifications` เชื่อมกับที่
  เรียนใน Part 065
- เพิ่ม custom field (`request_id`, `user_id`) เข้า log ด้วย `lograge.custom_payload` และ
  เข้าใจความสำคัญของ `request_id` ในฐานะ correlation id
- เข้าใจ log level ทั้ง 5 ระดับ และรู้ว่า Rails ตั้ง `production` เป็น `info` ผ่าน
  `ENV["RAILS_LOG_LEVEL"]` เพื่อปรับได้โดยไม่ต้อง deploy ใหม่
- เข้าใจแนวคิด centralized log aggregation, ทำไม `tail -f` ใช้ไม่ได้อีกต่อไปเมื่อมีหลาย
  instance, และทำไม Rails 8 log ออก STDOUT แทนไฟล์ (เชื่อมกับ Docker/Kamal จาก Part 073/076)
- ติดตั้ง **Sentry** (`sentry-ruby`/`sentry-rails`) จริง, ตั้งค่า `Sentry.init`, และยืนยันว่า
  automatic exception capture ทำงานร่วมกับ Rails error handling ได้โดยไม่ชนกัน แม้ไม่มี DSN
  จริงในสภาพแวดล้อมทดลอง (พร้อมเข้าใจว่าอะไร verify ได้และอะไรต้องมีบัญชีจริงถึงจะเห็น)
- รายงาน exception ที่ rescue ไว้ด้วยมือผ่าน `Sentry.capture_exception`, เพิ่ม context ด้วย
  `Sentry.set_user`/`set_context`, และเข้าใจความเสี่ยงด้านความปลอดภัยเรื่อง PII/secrets รั่วไหล
  ผ่าน error tracker พร้อมวิธีป้องกันด้วย `send_default_pii = false` และ `before_send`
- เข้าใจแนวคิด uptime monitoring จากภายนอก และเจาะลึก endpoint `/up` ที่ Rails 8 ให้มาโดย
  default — รู้ว่ามันเช็คแค่ "process ยังมีชีวิตอยู่ไหม" ไม่เช็ค dependency ลึกๆ โดยตั้งใจ
  (เพื่อป้องกัน cascading failure) และรู้วิธีขยายเป็น readiness check แยกต่างหากเมื่อจำเป็น
- มีกรอบความคิด **incident response** ที่ร้อยทุกเครื่องมือจาก Part 073–078 เข้าด้วยกัน: alert
  → log → error tracker → recent deploy → rollback → postmortem

## สรุปภาพรวม Phase 12: DevOps & Deployment

ยินดีด้วย! ตอนนี้ **Phase 12: DevOps & Deployment (Part 073–078, Step 721–780)** เสร็จสมบูรณ์
แล้ว เราเดินทางจากการบรรจุแอปให้พกพาไปรันที่ไหนก็ได้ด้วย **Docker** (Part 073), จัดการความลับ
และ config ให้ปลอดภัยต่างกันไปตาม environment (Part 074), ตั้ง **CI/CD** ให้ทดสอบและตรวจโค้ด
อัตโนมัติก่อนปล่อยออกไปทุกครั้ง (Part 075), **deploy จริง** ขึ้น production ด้วย Kamal สไตล์
Rails 8 (Part 076), รู้จักทางเลือกอื่นในการ deploy พร้อมวิธี migrate ฐานข้อมูลโดยไม่ทำให้ระบบ
ล่ม (Part 077) จนมาถึง **monitoring และ logging** ใน Part นี้ที่ปิดวงจรด้วยการทำให้ระบบที่
deploy ไปแล้ว "มองเห็นได้" (observable) — รู้ว่ามัน**ทำงานถูกต้องอยู่หรือไม่ ตลอดเวลา** ไม่ใช่
แค่รู้ว่า "deploy สำเร็จแล้ว" ในตอนที่กดปุ่ม

ทักษะทั้ง 6 Part นี้รวมกันคือ **toolkit ที่ครบสมบูรณ์สำหรับเอาแอป Rails จากเครื่องพัฒนาไปสู่
production จริง แล้วดูแลรักษามันต่อเนื่องได้** — ไม่ใช่แค่ "รู้วิธี deploy" แบบทำตามขั้นตอนโดย
ไม่เข้าใจ แต่เข้าใจว่าทำไมแต่ละขั้นตอนถึงถูกออกแบบมาแบบนั้น (ทำไม container ต้อง stateless,
ทำไม secrets ต้องแยกจากโค้ด, ทำไม CI ต้องรันก่อน merge เสมอ, ทำไม rollback ต้องทำได้เร็ว, ทำไม
health check ต้องตื้น) — นี่คือความแตกต่างระหว่างนักพัฒนาที่ "เคย deploy ได้" กับวิศวกรที่
"รับผิดชอบระบบ production จริง" ได้อย่างมั่นใจ

**ต่อไป (Part 079 — เปิด Phase 13: Security):** ตอนนี้แอปของเรา deploy ได้แล้ว, มองเห็นปัญหา
ได้แล้ว — แต่ยังมีคำถามสำคัญอีกข้อที่ยังไม่ได้ตอบ: **แอปนี้ปลอดภัยแค่ไหน?** Phase 13 จะพาไปรู้จัก
**OWASP Top 10** ช่องโหว่ความปลอดภัยที่พบบ่อยที่สุดในเว็บแอปพลิเคชันทั่วโลก โดยเน้นเฉพาะ context
ของ Rails — เริ่มจากสามช่องโหว่คลาสสิกที่อันตรายที่สุดใน Part 079: **SQL Injection** (การแอบฝัง
คำสั่ง SQL ผ่าน input ของผู้ใช้), **XSS (Cross-Site Scripting)** (การแอบฝัง JavaScript ที่เป็น
อันตรายผ่านหน้าเว็บ), และ **CSRF (Cross-Site Request Forgery)** (การหลอกให้ผู้ใช้ที่ล็อกอินอยู่
ทำ action โดยไม่ตั้งใจ) — ที่จริงแล้วเราเคยเจอ CSRF token มาแล้วโดยไม่รู้ตัวตั้งแต่ Part นี้เอง
(สังเกตข้อความ `Can't verify CSRF token authenticity` ตอนทดสอบ `POST /posts` ด้วย `curl` ตรงๆ
ใน Step 772) — Part 079 จะอธิบายให้กระจ่างว่าทำไม Rails ถึงป้องกันไว้ให้อัตโนมัติขนาดนั้น
