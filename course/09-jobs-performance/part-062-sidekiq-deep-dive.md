# Part 062: Sidekiq เชิงลึก — queue, retry, scheduled job

> **Step ครอบคลุมใน Part นี้:** Step 611–620
> **ระดับ:** กลาง (ต้องผ่าน Part 061 ActiveJob เบื้องต้นมาก่อน — รู้จัก `perform_later`, adapter, และการ generate job แล้ว)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / gem `sidekiq` 8.x (ทดสอบจริงกับ Redis 7.0)

Part นี้ไม่สอนพื้นฐาน ActiveJob ซ้ำ (`rails generate job`, `perform_later`, ภาพรวม adapter) เพราะ
Part 061 ได้ปูพื้นไว้แล้ว — ที่นี่เราจะลงลึกเฉพาะ **Sidekiq** ตัวเดียว ตั้งแต่เหตุผลที่ยังต้องใช้
Sidekiq ในยุคที่ Rails 8 มี Solid Queue เป็นค่าเริ่มต้น ไปจนถึง queue priority, retry/backoff
algorithm, dead job set, idempotency และ scheduled job — ครบทุกอย่างที่ต้องรู้ก่อนเอาไปใช้จริงใน
production

## สารบัญของ Part นี้

- Step 611: ทำไม Sidekiq ยังคงเป็นที่นิยม แม้ Rails 8 จะมี Solid Queue เป็นค่าเริ่มต้น
- Step 612: ติดตั้ง Sidekiq + Redis และตั้งค่า `queue_adapter`
- Step 613: รัน Sidekiq process และ Sidekiq Web UI (พร้อม authentication)
- Step 614: Native Sidekiq::Job syntax เทียบกับการใช้ผ่าน ActiveJob
- Step 615: Queue และ priority — `sidekiq_options queue:`, `config/sidekiq.yml`
- Step 616: Retry behavior — exponential backoff algorithm และ dead job set
- Step 617: `retry_on`/`discard_on` ของ ActiveJob เทียบกับ retry ของ Sidekiq เอง
- Step 618: Idempotency — ทำไม job อาจถูกรันซ้ำ และวิธีออกแบบให้ปลอดภัย
- Step 619: Scheduled/delayed job ด้วย `perform_in`/`perform_at`
- Step 620: Monitoring และ observability ผ่าน Sidekiq Web UI และ Sidekiq API

---

## Step 611: ทำไม Sidekiq ยังคงเป็นที่นิยม แม้ Rails 8 จะมี Solid Queue เป็นค่าเริ่มต้น

Rails 8 เปลี่ยน default ของ `config.active_job.queue_adapter` มาเป็น **Solid Queue** — เก็บ job
ไว้ในฐานข้อมูล (RDBMS ตัวเดียวกับแอป หรือแยก database ก็ได้) แทนที่จะต้องพึ่ง Redis เพิ่มอีกตัว
เป็นส่วนหนึ่งของปรัชญา "no PaaS required" ของ Rails 8 (deploy ด้วย Kamal ลง VM เดียว ไม่ต้องง้อ
managed service เยอะๆ)

คำถามที่ตามมาคือ — ถ้า Rails 8 มี Solid Queue มาให้ฟรีแล้ว ทำไมทีมจำนวนมาก (รวมถึงโปรเจกต์
ระดับองค์กร) ยังเลือก Sidekiq อยู่? คำตอบสั้นๆ คือ **Sidekiq ยังเก่งกว่าในหลายมิติที่สำคัญมากสำหรับ
งานที่ปริมาณ/ความซับซ้อนสูง** และมี ecosystem ที่เก่าแก่และแข็งแกร่งกว่ามาก

### เปรียบเทียบแบบตรงไปตรงมา

| หัวข้อ | Solid Queue | Sidekiq |
|--------|-------------|---------|
| Storage | ฐานข้อมูล (MySQL/PostgreSQL/SQLite) | Redis (in-memory data store) |
| Concurrency model | Thread + process (polling ฐานข้อมูล) | Multi-threaded ต่อ process เดียว ประสิทธิภาพสูงมากเพราะ Redis เร็วกว่าการ poll DB |
| Throughput ที่ปริมาณสูงมาก (หลักหมื่น-แสน job/นาที) | ยังทำได้ แต่ DB อาจกลายเป็นคอขวด (lock contention, I/O) | ออกแบบมาเพื่อสิ่งนี้โดยเฉพาะ เป็นที่มาของชื่อ "Simple, efficient background processing" |
| Infrastructure เพิ่มเติม | ไม่ต้อง (ใช้ DB ที่มีอยู่แล้ว) | ต้องมี Redis server แยกต่างหาก |
| ต้นทุน operational | ต่ำกว่า — ไม่มีระบบใหม่ให้ดูแล | สูงกว่าเล็กน้อย — ต้อง monitor/backup Redis เพิ่ม |
| ความเก่าแก่/mature | ใหม่ (เปิดตัวจริงจังใน Rails 7.1–8.0) | เปิดตัวปี 2012 ผ่านการใช้งานจริงมากว่า 10 ปี |
| Ecosystem | ยังเล็ก กำลังเติบโต | ใหญ่มาก: gem เสริมนับร้อย, ปลั๊กอิน monitoring, batch job, cron ผ่าน `sidekiq-cron` ฯลฯ |
| Enterprise features | ยังไม่มีระดับ "Pro/Enterprise" แยกขาย | มี **Sidekiq Pro** (reliability เพิ่ม, batches, throttling) และ **Sidekiq Enterprise** (unique jobs, rate limiting, encryption, multi-process) ให้ซื้อเมื่อจำเป็น |
| Track record ระดับ scale ใหญ่ | ยังไม่มีเคสระดับ Shopify/GitHub มายืนยันยาวนาน | ใช้จริงในระบบระดับหลักล้าน job/วันของบริษัทใหญ่จำนวนมากมาหลายปี |
| Web UI สำหรับ monitor | มี (Mission Control - Jobs, แยก gem) | มีในตัว (`Sidekiq::Web`) ครบเครื่องมาก มีมานานและเสถียร |
| เหมาะกับ | โปรเจกต์ใหม่ที่อยากลด infrastructure, งาน background ปริมาณปานกลาง | ระบบที่มี job ปริมาณสูงมาก, ต้องการ retry/scheduling ที่ปรับแต่งได้ละเอียด, ทีมที่คุ้นเคยกับ Sidekiq อยู่แล้ว, หรือระบบเดิมที่ใช้ Sidekiq มาก่อนแล้วไม่มีเหตุผลต้อง migrate |

### สรุป: เมื่อไหร่ควรยังเลือก Sidekiq

1. **ปริมาณ job สูงมาก** — ระบบที่ยิง job หลักแสนถึงล้านครั้งต่อวัน (เช่น ส่งอีเมล/แจ้งเตือนแบบ
   fan-out, ประมวลผล webhook จำนวนมาก) Redis ในฐานะ in-memory queue จะเร็วและ scale ง่ายกว่า
   การ poll ตาราง SQL ที่นับวันจะโตขึ้นเรื่อยๆ
2. **ต้องการ retry/scheduling logic ที่ซับซ้อน** — Sidekiq ให้ควบคุม backoff, dead set, batch
   job ได้ละเอียดกว่า (จะเห็นตลอด Part นี้)
3. **ทีมมีประสบการณ์ Sidekiq อยู่แล้ว** — cost ของการเรียนรู้ใหม่ (Solid Queue) อาจไม่คุ้มถ้า
   ระบบเดิมทำงานได้ดีอยู่แล้ว
4. **ต้องการ feature ระดับ enterprise** — unique jobs (ป้องกัน job ซ้ำโดยไม่ต้องเขียน guard เอง),
   rate limiting ระดับ Redis, encrypted payload ฯลฯ ผ่าน Sidekiq Pro/Enterprise
5. **Redis ถูกใช้อยู่แล้วในระบบ** (เช่นเป็น cache store) — เพิ่มต้นทุน operational แทบไม่มี เพราะ
   infra มีอยู่แล้ว

กลับกัน ถ้าเป็นโปรเจกต์ใหม่ขนาดเล็ก-กลาง ไม่อยากเพิ่ม Redis เข้ามาดูแล และปริมาณ job ไม่ได้สูงมาก
— Solid Queue ก็เพียงพอและลดความซับซ้อนของ infrastructure ได้จริง เพราะฉะนั้นการเลือกไม่ใช่
"Sidekiq ดีกว่าเสมอ" แต่คือ "เลือกให้เหมาะกับสถานการณ์" — และ Part นี้จะสอนให้ใช้ Sidekiq ได้ลึก
พอที่จะตัดสินใจแบบมีข้อมูลได้เอง

---

## Step 612: ติดตั้ง Sidekiq + Redis และตั้งค่า `queue_adapter`

### ติดตั้ง Redis

Sidekiq เก็บ job queue, retry set, scheduled set ฯลฯ ไว้ใน Redis ทั้งหมด ต้องมี Redis server
รันอยู่ก่อนเสมอ

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y redis-server

# macOS (Homebrew)
brew install redis
brew services start redis

# ตรวจสอบว่า Redis รันอยู่และตอบสนอง
redis-cli ping
# => PONG
```

> ในเครื่อง dev/CI ที่ไม่มี privilege ติดตั้ง service ถาวร สามารถรัน Redis แบบ foreground/daemon
> ตรงๆ ได้เช่นกัน: `redis-server --daemonize yes` แล้วเช็คด้วย `redis-cli ping`

### เพิ่ม gem `sidekiq`

```ruby
# Gemfile
gem "sidekiq"
```

```bash
bundle install
```

ตัวอย่างนี้ทดสอบจริงด้วย `sidekiq (8.1.7)` บน Ruby 3.3.6 + Rails 8.1.4

### ตั้งค่า queue adapter ให้ ActiveJob ใช้ Sidekiq

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.load_defaults 8.1

    config.active_job.queue_adapter = :sidekiq
  end
end
```

จะตั้งเฉพาะ environment ก็ได้ (พบบ่อยในทีมที่ใช้ `async` adapter ตอน dev แต่ใช้ Sidekiq จริงบน
staging/production):

```ruby
# config/environments/production.rb
Rails.application.configure do
  config.active_job.queue_adapter = :sidekiq
end
```

### ตั้งค่า Redis URL (ถ้าไม่ใช่ default `localhost:6379`)

Sidekiq client (ฝั่งที่ enqueue job จาก Rails app) และ Sidekiq server (process ที่ประมวลผล job)
ต่างก็ต้องรู้ว่า Redis อยู่ที่ไหน กำหนดผ่าน `Sidekiq.configure_client` / `Sidekiq.configure_server`
ในไฟล์ initializer เดียวกันได้:

```ruby
# config/initializers/sidekiq.rb
redis_url = ENV.fetch("REDIS_URL", "redis://localhost:6379/0")

Sidekiq.configure_server do |config|
  config.redis = { url: redis_url }
end

Sidekiq.configure_client do |config|
  config.redis = { url: redis_url }
end
```

> **แนวคิดสำคัญ:** ถ้าไม่ตั้งค่า `REDIS_URL` เลย Sidekiq จะต่อ `redis://localhost:6379/0` เป็น
> ค่าเริ่มต้นให้อัตโนมัติ — สำหรับ dev บนเครื่องเดียวจึงไม่ต้องตั้งค่าอะไรเพิ่มเลยก็ใช้งานได้ทันที
> (นี่คือสิ่งที่เราทดสอบจริงในเครื่องสำหรับ Part นี้)

**ทดสอบผ่าน `bin/rails console`:**

```irb
irb> Rails.application.config.active_job.queue_adapter
=> :sidekiq

irb> HardJob.perform_async("alice", 1)
=> "859321992e789450e89cf1aa"   # นี่คือ Sidekiq job id (jid)
```

การ enqueue สำเร็จ ได้ jid กลับมา หมายความว่า Rails process คุยกับ Redis ได้แล้ว — แต่ job จะยัง
"ค้าง" อยู่ใน queue จนกว่าจะมี Sidekiq process จริงมาดึงไปประมวลผล (Step ถัดไป)

---

## Step 613: รัน Sidekiq process และ Sidekiq Web UI (พร้อม authentication)

### รัน Sidekiq worker process

Rails server (Puma) **ไม่ได้** ประมวลผล background job ให้เอง — ต้องรัน Sidekiq เป็น process
แยกต่างหาก:

```bash
bundle exec sidekiq
```

ผลลัพธ์ตอนบูตจะหน้าตาประมาณนี้ (ทดสอบจริง):

```
INFO: Sidekiq 8.1.7 connecting to Redis with options {:size=>10, :pool_name=>"internal", :url=>nil}
INFO: Running in ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]
INFO: Booted Rails 8.1.4 application in development environment
```

เมื่อ process นี้รันอยู่ มันจะไปดึง job จาก Redis มา execute ทันที ทดสอบจริงแล้ว: enqueue
`HardJob.perform_async("alice", 1)` จาก console แล้วสตาร์ท `bundle exec sidekiq` — log แสดง:

```
INFO: jid=859321992e789450e89cf1aa class=HardJob: start
INFO: jid=859321992e789450e89cf1aa class=HardJob elapsed=0.158: done
```

**ใน production** จะรัน process นี้ผ่าน process manager เช่น systemd, หรือใน Docker/Kamal จะเป็น
container แยกที่รันคำสั่งเดียวกัน โดยทั่วไป 1 เครื่อง (หรือ 1 container) รัน 1 Sidekiq process
ที่มีหลาย thread ข้างใน (ตั้งค่าด้วย `:concurrency` — จะพูดใน Step 615)

ตัวอย่าง systemd unit file:

```ini
# /etc/systemd/system/sidekiq.service
[Unit]
Description=Sidekiq background job processor
After=syslog.target network.target

[Service]
Type=simple
WorkingDirectory=/var/www/myapp/current
ExecStart=/usr/bin/bundle exec sidekiq -e production
User=deploy
RestartSec=1
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Sidekiq Web UI

Sidekiq มี dashboard สำเร็จรูปเป็น Rack app ชื่อ `Sidekiq::Web` — mount เข้า router ได้ทันที:

```ruby
# config/routes.rb
require "sidekiq/web"

Rails.application.routes.draw do
  mount Sidekiq::Web => "/sidekiq"
  # ...
end
```

เปิด `http://localhost:3000/sidekiq` จะเห็นหน้า dashboard แสดง jobs ที่กำลังประมวลผล
(Busy), queue แต่ละคิว, retry set, scheduled set, และ dead set

### ⚠️ ข้อควรระวังด้านความปลอดภัย: ห้าม expose Web UI แบบไม่มี authentication เด็ดขาด

Sidekiq Web UI เปิดให้ดู **ข้อมูลภายในระบบทั้งหมด** — arguments ของ job (อาจมี PII หรือ token),
error message/backtrace แบบละเอียด, และที่อันตรายที่สุดคือปุ่ม **"Retry now"**, **"Delete"** และ
**"Kill"** ที่ทุกคนที่เข้าถึง URL ได้สามารถกดสั่งงานได้ทันที ถ้า mount แบบไม่มี auth เท่ากับเปิด
ให้ใครก็ได้ที่รู้ path `/sidekiq` เข้ามาดู/แก้ไข job queue ของระบบจริง — เป็นช่องโหว่ที่พบได้บ่อย
ใน security scan จริง

**วิธีที่ 1: Basic Auth ตรงๆ (เหมาะกับระบบเล็ก/internal tool)**

```ruby
# config/routes.rb
require "sidekiq/web"

Sidekiq::Web.use(Rack::Auth::Basic) do |user, password|
  # ใช้ secure_compare กัน timing attack เสมอ อย่าใช้ == ตรงๆ
  ActiveSupport::SecurityUtils.secure_compare(user, Rails.application.credentials.sidekiq_web_user) &
    ActiveSupport::SecurityUtils.secure_compare(password, Rails.application.credentials.sidekiq_web_password)
end

Rails.application.routes.draw do
  mount Sidekiq::Web => "/sidekiq"
end
```

ทดสอบจริงแล้ว (curl ยิงตรงไปที่ route ที่ mount ไว้):

```bash
curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/sidekiq
# => 401   (ไม่ใส่ credential เลย)

curl -o /dev/null -w "%{http_code}\n" -u admin:wrongpass http://localhost:3000/sidekiq
# => 401   (password ผิด)

curl -o /dev/null -w "%{http_code}\n" -u admin:secret123 http://localhost:3000/sidekiq
# => 200   (ถูกต้อง)
```

**วิธีที่ 2: ผูกกับระบบ authentication ของแอปเอง (แนะนำสำหรับระบบที่มี admin role อยู่แล้ว)**

ถ้าแอปมี Devise + role-based access อยู่แล้ว ใช้ `constraints` ของ router แทนได้ ทำให้เฉพาะ
user ที่ login และเป็น admin เท่านั้นที่เข้าได้ (ไม่ต้องจำ password แยกอีกชุด):

```ruby
# config/routes.rb
require "sidekiq/web"

Rails.application.routes.draw do
  authenticate :user, ->(user) { user.admin? } do
    mount Sidekiq::Web => "/sidekiq"
  end
end
```

**วิธีที่ 3: จำกัดด้วย network layer เพิ่มอีกชั้น** — ถึงจะมี auth ในแอปแล้ว ทีม production
จริงจำนวนมากยังกัน `/sidekiq` ด้วย IP allowlist หรือ VPN ที่ระดับ load balancer/nginx ซ้อนอีกชั้น
เป็น defense in depth เพราะ auth ในแอปอาจมีบั๊ก หรือ credential รั่วได้

> **สรุปกฎเหล็ก:** ทุกครั้งที่ `mount Sidekiq::Web`, auth ต้องมาคู่กันเสมอ ไม่มีข้อยกเว้น
> แม้แต่ในระบบที่ "ไม่น่าจะมีใครเดา URL เจอ" — security through obscurity ไม่ใช่ security

---

## Step 614: Native Sidekiq::Job syntax เทียบกับการใช้ผ่าน ActiveJob

Sidekiq ให้เขียน job ได้ 2 แบบ ซึ่งมีข้อแลกเปลี่ยนที่ต้องเข้าใจก่อนเลือกใช้

### แบบที่ 1: Native Sidekiq (`include Sidekiq::Job`)

```ruby
# app/jobs/hard_job.rb
class HardJob
  include Sidekiq::Job
  sidekiq_options queue: "high", retry: 3

  def perform(name, count)
    # ทำงานจริง
    ExternalApi.notify(name, count)
  end
end
```

เรียกใช้ด้วย method ของ Sidekiq เอง ไม่ใช่ `perform_later`:

```ruby
HardJob.perform_async("alice", 1)          # enqueue ทันที คืนค่า jid (String)
HardJob.perform_in(60, "alice", 1)         # enqueue แบบ delay 60 วินาที
HardJob.perform_at(1.hour.from_now, "a", 1) # enqueue ที่เวลาที่กำหนด
```

ทดสอบจริง: `HardJob.perform_async("alice", 1)` คืนค่า jid เป็น string เช่น
`"859321992e789450e89cf1aa"` — และ log ตอนประมวลผลจะสั้นกระชับ ไม่มี wrapper:

```
INFO: jid=859321992e789450e89cf1aa class=HardJob: start
INFO: jid=859321992e789450e89cf1aa class=HardJob elapsed=0.158: done
```

### แบบที่ 2: ผ่าน ActiveJob (`< ApplicationJob` + `sidekiq_options`)

เมื่อตั้ง `queue_adapter = :sidekiq` แล้ว ActiveJob class ทุกตัวที่มี `queue_as`/`perform_later`
อยู่แล้วจะถูกส่งไปที่ Sidekiq โดยอัตโนมัติ และยังสามารถเรียก `sidekiq_options` เพิ่มเติมได้ตรงๆ
บน ActiveJob class เลย (Sidekiq เติม method นี้เข้าไปให้ผ่าน integration module ของมันเอง):

```ruby
# app/jobs/report_job.rb
class ReportJob < ApplicationJob
  queue_as :low
  sidekiq_options retry: 5   # feature เฉพาะของ Sidekiq ใช้ได้แม้เขียนแบบ ActiveJob

  def perform(report_id)
    GenerateReport.call(report_id)
  end
end
```

```ruby
ReportJob.perform_later(42)                       # enqueue ทันที
ReportJob.set(wait: 5.seconds).perform_later(42)   # enqueue แบบ delay
```

ทดสอบจริง: log ของงานที่ enqueue ผ่าน ActiveJob จะมีรายละเอียดมากกว่า เพราะ Sidekiq ห่อ
ActiveJob ไว้ใน wrapper class ภายใน (`Sidekiq::ActiveJob::Wrapper`) แล้ว dispatch ต่อให้
ActiveJob เป็นคนเรียก `perform` จริง:

```
INFO: jid=264c9dd27c42ee2931f31a11 class=ReportJob: start
INFO: jid=264c9dd27c42ee2931f31a11 class=ReportJob: Performing ReportJob
      (Job ID: 6dae6fdb-...) from Sidekiq(low) enqueued at ... with arguments: 42
INFO: jid=264c9dd27c42ee2931f31a11 class=ReportJob: Performed ReportJob
      (Job ID: 6dae6fdb-...) from Sidekiq(low) in 10.32ms
INFO: jid=264c9dd27c42ee2931f31a11 class=ReportJob elapsed=0.179: done
```

สังเกตว่ามี 2 ID: `jid` (ของ Sidekiq) และ `Job ID` (ของ ActiveJob เอง, เป็น UUID) — เพราะมี
2 เลเยอร์ห่อกันอยู่

### ตารางเปรียบเทียบ tradeoff

| ประเด็น | Native `Sidekiq::Job` | ActiveJob (`< ApplicationJob`) |
|---------|------------------------|----------------------------------|
| Portability ข้าม adapter | ❌ ผูกกับ Sidekiq ตายตัว เปลี่ยนไปใช้ Solid Queue/GoodJob ทีหลังต้องเขียนใหม่ | ✅ เปลี่ยน adapter แค่ตัวเดียวใน config โค้ด job ไม่ต้องแก้ |
| Method เรียกใช้ | `perform_async` / `perform_in` / `perform_at` | `perform_later` / `set(wait:).perform_later` |
| Argument ที่รองรับ | เฉพาะ JSON-native type (String, Integer, Array, Hash) เท่านั้น — **ห้าม** ส่ง ActiveRecord object ตรงๆ | รองรับ Global ID serialization ผ่าน ActiveJob (ส่ง ActiveRecord object ได้ตรงๆ ปลอดภัยกว่า) |
| Feature เฉพาะของ Sidekiq (`sidekiq_options`, `sidekiq_retry_in`, `sidekiq_retries_exhausted`) | ✅ ใช้ได้เต็มที่ ตรงไปตรงมา | ✅ ใช้ได้เช่นกัน (Sidekiq เติม `sidekiq_options` ให้ ActiveJob::Base โดยอัตโนมัติเมื่อ adapter เป็น sidekiq) |
| Retry system ที่ทำงานจริง | Sidekiq retry set + backoff algorithm ของ Sidekiq | ถ้าใช้ `retry_on`/`discard_on` ของ ActiveJob จะ**ไม่**เข้า Sidekiq retry set (ดู Step 617) แต่ถ้าใช้ `sidekiq_options retry:` บน ActiveJob class ก็ยังเข้า Sidekiq retry set ตามปกติ |
| Logging | กระชับ | ละเอียดกว่า (มี wrapper log ของ ActiveJob ซ้อนด้วย) |
| เหมาะกับ | Job ที่ผูกกับ Sidekiq โดยตั้งใจอยู่แล้ว ต้องการ feature เฉพาะทางเต็มรูปแบบ (batches, unique jobs ของ Pro/Enterprise) | Job ทั่วไปในแอป Rails มาตรฐาน ที่อยากคง flexibility เผื่อเปลี่ยน adapter ในอนาคต หรือใช้ฟีเจอร์ Rails เช่น `enqueue_after_transaction_commit`, testing helper ของ ActiveJob |

**คำแนะนำในทางปฏิบัติ:** ทีมส่วนใหญ่เลือกใช้ **ActiveJob (`< ApplicationJob`) เป็นค่าเริ่มต้น**
เพราะได้ portability และ integration กับ Rails testing/transaction commit ฟรี แล้วค่อยเติม
`sidekiq_options` เข้าไปเฉพาะจุดที่ต้องการ fine-tune พฤติกรรมของ Sidekiq จริงๆ — ใช้ native
`Sidekiq::Job` เฉพาะกรณีที่รู้ตัวว่าจะผูกกับ Sidekiq ระยะยาวแน่นอน (เช่นใช้ batch job ของ
Sidekiq Pro ที่ ActiveJob ไม่มี concept นี้)

---

## Step 615: Queue และ priority — `sidekiq_options queue:`, `config/sidekiq.yml`

### กำหนดคิวให้ job แต่ละตัว

```ruby
class HardJob
  include Sidekiq::Job
  sidekiq_options queue: "high"
  # ...
end

class ReportJob < ApplicationJob
  queue_as :low   # ActiveJob ใช้ queue_as ได้ตามปกติ ก็ map ไปที่ Sidekiq queue "low"
  # ...
end
```

### ทำไมต้องมีหลาย queue

Sidekiq ไม่มี concept "priority ต่อ job" ในคิวเดียวกัน (ไม่ใช่ priority queue แบบ heap) — วิธีจัด
ลำดับความสำคัญคือ**แยก job ออกเป็นหลาย queue แล้วกำหนดน้ำหนัก (weight) ให้แต่ละคิว**ตอนรัน
Sidekiq process

```yaml
# config/sidekiq.yml
---
:concurrency: 5

:queues:
  - [high, 3]
  - [default, 2]
  - [low, 1]
```

ตัวเลขหลังชื่อ queue คือ weight — ทดสอบจริงแล้ว: ด้วยค่านี้ Sidekiq จะสุ่มเลือกดึง job จาก
`high` บ่อยกว่า `default` และ `default` บ่อยกว่า `low` ตามสัดส่วน 3:2:1 (ไม่ใช่การรับประกันว่า
`high` จะถูกดึงหมดก่อนเสมอ — เป็น weighted random เพื่อไม่ให้คิวที่ priority ต่ำ "อดอาหาร"
(starvation) ไปเลย)

รันด้วยไฟล์ config นี้:

```bash
bundle exec sidekiq -C config/sidekiq.yml
```

ถ้าต้องการ**รับประกันลำดับเคร่งครัด** (queue ที่มาก่อนต้องว่างก่อนถึงจะไปดึงคิวถัดไป) ใช้ syntax
แบบไม่ระบุ weight แทน ลำดับในไฟล์คือลำดับความสำคัญ:

```yaml
:queues:
  - critical
  - default
  - low
```

### `:concurrency` คืออะไร

`:concurrency: 5` หมายถึง Sidekiq process นี้จะมี **thread pool 5 thread** ทำงานพร้อมกัน — job
5 ตัวสามารถถูกประมวลผลพร้อมกันได้ในเวลาเดียวกันภายใน process เดียว (คนละเรื่องกับจำนวน process
ที่รัน — จะรันหลาย Sidekiq process ต่อเครื่องก็ได้ ถ้ามี CPU core เหลือ)

ทดสอบจริง: ตั้ง `concurrency: 5` แล้ว enqueue 4 job พร้อมกัน (2 คิว `high`/`low` คนละ 2 job) — log
แสดงว่าทั้ง 4 job เริ่ม (`start`) เกือบพร้อมกันในหน่วย millisecond เดียวกัน ยืนยันว่าประมวลผล
แบบขนานจริงผ่าน thread ไม่ใช่ทีละตัว

> **ข้อควรระวัง:** thread ทั้งหมดใน process เดียวกันใช้ CPU core และ memory ร่วมกัน ถ้า job
> เป็นงานที่กิน CPU หนัก (CPU-bound เช่น image processing, การคำนวณหนักๆ) การเพิ่ม concurrency
> เกินจำนวน CPU core จะไม่ช่วยอะไร (thread แย่งกันเอง) แต่ถ้า job ส่วนใหญ่เป็น I/O-bound (เรียก
> API ภายนอก, query DB, รอ network) concurrency สูงๆ จะช่วยได้มากเพราะ thread ที่รอ I/O จะ
> คืน control ให้ thread อื่นทำงานต่อ (Ruby GVL ปลดล็อกตอนรอ I/O)

### กำหนด concurrency ต่อ environment ผ่าน ENV

แนวปฏิบัติทั่วไปคือให้ปรับ concurrency ผ่าน environment variable แทนการ hardcode ในไฟล์ yml
เพื่อปรับได้ตาม instance size โดยไม่ต้อง deploy โค้ดใหม่:

```yaml
# config/sidekiq.yml
---
:concurrency: <%= ENV.fetch("SIDEKIQ_CONCURRENCY", 5) %>

:queues:
  - [high, 3]
  - [default, 2]
  - [low, 1]
```

(ไฟล์ `sidekiq.yml` รองรับ ERB โดยอัตโนมัติ)

---

## Step 616: Retry behavior — exponential backoff algorithm และ dead job set

### เปิด/ปิด retry และกำหนดจำนวนครั้ง

```ruby
class HardJob
  include Sidekiq::Job
  sidekiq_options retry: 5   # retry ได้สูงสุด 5 ครั้งก่อนยอมแพ้
end
```

ค่า `retry:` ที่เป็นไปได้:

- **`retry: true`** (ค่า default ถ้าไม่ระบุอะไรเลย) — retry ได้สูงสุด **25 ครั้ง**
  (`Sidekiq::JobRetry::DEFAULT_MAX_RETRY_ATTEMPTS = 25`, ยืนยันจาก source code จริง)
- **`retry: <Integer>`** — retry สูงสุดตามจำนวนที่กำหนด เช่น `retry: 5`
- **`retry: 0`** — **ไม่ retry เลย** ถ้า fail ครั้งแรกจะเข้า dead set ทันที
- **`retry: false`** — ไม่ retry และ**ไม่**เข้า dead set ด้วย (job หายไปเงียบๆ เมื่อ fail)

ทดสอบจริง (ยืนยันด้วย job ที่ตั้ง `retry: 0` แล้วโยน exception เสมอ):

```ruby
class DoomedJob
  include Sidekiq::Job
  sidekiq_options queue: "default", retry: 0

  def perform
    raise "always fails"
  end
end

DoomedJob.perform_async
```

log ที่ได้:

```
INFO: jid=0a291183211a9268bacf3a2a class=DoomedJob elapsed=0.033: fail
INFO: context=Job raised exception job={"retry"=>0, ...}:
      .../doomed_job.rb:6:in 'perform': always fails (RuntimeError)
```

และตรวจสอบผ่าน Sidekiq API ทันทีหลัง fail:

```irb
irb> require "sidekiq/api"
irb> Sidekiq::DeadSet.new.size
=> 1
irb> Sidekiq::DeadSet.new.first.klass
=> "DoomedJob"
```

ยืนยันว่า `retry: 0` ทำให้ job เข้า **dead set** (เรียกอีกชื่อว่า "morgue" ในโค้ด/URL ของ Web UI)
ทันทีโดยไม่มีการ retry เลยแม้แต่ครั้งเดียว

### สูตร exponential backoff ของ Sidekiq

เมื่อ `retry` มากกว่า 0 และ job fail, Sidekiq จะไม่ retry ทันที แต่จะคำนวณเวลาที่จะ retry ครั้ง
ถัดไปด้วยสูตร (อ่านจาก source code `sidekiq/job_retry.rb` ตรงๆ):

```
delay = (count ** 4) + 15
jitter = rand(10 * (count + 1))
retry_at = ตอนนี้ + delay + jitter
```

โดย `count` คือจำนวนครั้งที่ retry ไปแล้ว (เริ่มจาก 0 ในความล้มเหลวครั้งแรก) ตารางเวลาโดยประมาณ:

| ครั้งที่ fail (count) | delay หลัก | ช่วง jitter | รวมประมาณ |
|----|------|------|------|
| 1 (count=0) | 15 วิ | +0–9 วิ | ~15–24 วินาที |
| 2 (count=1) | 16 วิ | +0–19 วิ | ~16–35 วินาที |
| 3 (count=2) | 31 วิ | +0–29 วิ | ~31–60 วินาที |
| 4 (count=3) | 96 วิ | +0–39 วิ | ~1.6–2.25 นาที |
| 5 (count=4) | 271 วิ | +0–49 วิ | ~4.5–5.3 นาที |
| ... | เพิ่มแบบ `count^4` | | ไปจนถึงหลายชั่วโมงในครั้งท้ายๆ |

การเติม `jitter` (สุ่มบวกเพิ่ม) มีไว้ป้องกัน **thundering herd** — ถ้า job จำนวนมากพัง
พร้อมกัน (เช่น external API ล่มพร้อมกันหมด) ไม่ให้ทุก job กลับมา retry ในวินาทีเดียวกันเป๊ะๆ
จนกระหน่ำ API ที่เพิ่งจะฟื้นซ้ำอีกรอบ

**ทดสอบจริง:** เขียน job ที่ fail 2 ครั้งแรกแล้วสำเร็จครั้งที่ 3 (`retry: 2`) แล้ววัดเวลาจริงจาก
log:

```
attempt 1  ที่เวลา 08:10:38
attempt 2  ที่เวลา 08:10:56   (ห่างจากครั้งก่อน 18 วินาที — อยู่ในช่วง count=0 → 15-24s ✓)
attempt 3  ที่เวลา 08:11:20   (ห่างจากครั้งก่อน 24 วินาที — อยู่ในช่วง count=1 → 16-35s ✓)
```

ตรงกับสูตรเป๊ะ และหลัง attempt 3 สำเร็จ (ไม่ raise exception อีก) `Sidekiq::RetrySet.new.size`
กลับมาเป็น `0` ทันที ยืนยันว่า job หลุดออกจาก retry set แล้วเมื่อสำเร็จ

### กำหนด backoff เอง — `sidekiq_retry_in`

ถ้าไม่ต้องการสูตร `count**4 + 15` ของ Sidekiq เอง กำหนด custom backoff ได้ผ่าน
`sidekiq_retry_in` (ทดสอบจริงแล้วในตัวอย่างท้าย Part นี้ — ดู `ProcessOrderJob`):

```ruby
class ProcessOrderJob
  include Sidekiq::Job
  sidekiq_options retry: 5

  # backoff เชิงเส้น: 10, 20, 30, 40, 50 วินาที (ก่อนบวก jitter ที่ Sidekiq เติมให้อัตโนมัติ)
  sidekiq_retry_in do |count, exception, jobhash|
    10 * (count + 1)
  end

  def perform(order_id)
    # ...
  end
end
```

Block นี้รับ `count` (จำนวนครั้งที่ retry ไปแล้ว), `exception` (exception object ที่ทำให้ fail),
และ `jobhash` (payload ทั้งหมดของ job) — คืนค่าเป็นจำนวนวินาทีที่จะรอ หรือคืน symbol พิเศษ:

- คืน `:discard` — ยกเลิก job ทันทีแบบเงียบๆ ไม่เข้า dead set
- คืน `:kill` — ส่งเข้า dead set ทันที (เหมือน exhausted แล้ว)

### เมื่อ retry ครบตามจำนวนแล้วยังไม่สำเร็จ — dead set

เมื่อ retry ครบจำนวนที่กำหนดใน `retry:` แล้วยัง fail อยู่ Sidekiq จะย้าย job เข้า **dead set**
(sorted set ชื่อ `dead` ใน Redis, เข้าถึงผ่าน Web UI ที่ path `/sidekiq/morgue`) — job ในนี้จะ
**ไม่ถูกประมวลผลอีก** จนกว่าจะมีคนกด "Retry" เองผ่าน Web UI หรือเรียกผ่าน API

```irb
irb> require "sidekiq/api"
irb> dead_job = Sidekiq::DeadSet.new.first
irb> dead_job.klass       # => "DoomedJob"
irb> dead_job.args        # => []
irb> dead_job["error_message"]  # => "always fails"
irb> dead_job.retry        # สั่ง requeue กลับเข้า queue เดิมทันที
```

รัน callback ตอน retry หมดได้ด้วย `sidekiq_retries_exhausted` — มีประโยชน์มากสำหรับแจ้งเตือนทีม
เมื่อ job สำคัญ fail ถาวร (จะใช้จริงในแบบฝึกหัดท้าย Part นี้):

```ruby
sidekiq_retries_exhausted do |job, exception|
  Rails.logger.error("[ProcessOrderJob] order_id=#{job['args'].first} exhausted all retries: #{exception.message}")
  # หรือส่งแจ้งเตือนไป Slack, Sentry, PagerDuty ฯลฯ
end
```

> **ข้อควรระวัง:** dead set มีขนาดจำกัด (default เก็บสูงสุด 10,000 job และเก็บได้นานสุด 180 วัน
> — ปรับได้ผ่าน `Sidekiq.default_configuration[:dead_max_jobs]` และ `[:dead_timeout_in_seconds]`)
> ถ้ามี job ตายเยอะเกินไปโดยไม่มีใครมาดู เก่าที่สุดจะถูกทิ้งอัตโนมัติ — dead set ไม่ใช่ที่เก็บถาวร
> เป็นแค่ safety net ให้ human มาตรวจสอบ/สั่ง retry เอง

---

## Step 617: `retry_on`/`discard_on` ของ ActiveJob เทียบกับ retry ของ Sidekiq เอง

เมื่อเขียน job แบบ ActiveJob (`< ApplicationJob`) มี retry mechanism ให้เลือกใช้ถึง **2 ระบบซ้อน
กัน** ซึ่งทำงานคนละแบบ และนี่คือจุดที่มือใหม่สับสนบ่อยที่สุด

### ระบบที่ 1: `retry_on`/`discard_on` ของ ActiveJob เอง

```ruby
class AjRetryJob < ApplicationJob
  queue_as :default
  retry_on StandardError, wait: 3.seconds, attempts: 2

  def perform
    # ...
    raise "fail via ActiveJob retry_on"
  end
end
```

**ทดสอบจริงแล้ว** — enqueue แล้วดู log ของ Sidekiq process:

```
INFO: jid=08da7a03d968be11609ae04a class=AjRetryJob: start
INFO: class=AjRetryJob: Performing AjRetryJob (Job ID: 97b3e2d2-...) from Sidekiq(default) ...
INFO: class=AjRetryJob: Retrying AjRetryJob (Job ID: 97b3e2d2-...) after 1 attempts in 3 seconds,
      due to a RuntimeError (fail via ActiveJob retry_on).
INFO: class=AjRetryJob: Performed AjRetryJob (Job ID: 97b3e2d2-...) from Sidekiq(default) in 1.87ms
INFO: jid=08da7a03d968be11609ae04a class=AjRetryJob elapsed=0.012: done
```

สังเกตบรรทัดสุดท้าย — Sidekiq job (`jid=08da7a03d968be11609ae04a`) จบแบบ **`done`** (สำเร็จ!)
ไม่ใช่ `fail` ทั้งที่ข้างในมี exception เกิดขึ้นจริง เพราะอะไร?

เพราะ `retry_on` ของ ActiveJob **ดักจับ exception เอาไว้เอง** ก่อนที่มันจะลอยออกไปถึง Sidekiq
แล้วสร้าง Sidekiq job **ใหม่** (jid ใหม่) enqueue กลับเข้า queue เดิมด้วยตัวเอง (ผ่าน
`enqueue_at`) จากมุมมองของ Sidekiq มันเห็นแค่ "job สำเร็จ แล้วก็มี job ใหม่โผล่เข้ามาที่เวลา
ที่กำหนด" — ไม่ได้เห็นว่านี่คือการ retry ของ job เดิม

**ผลที่ตามมา (ยืนยันจากการทดสอบจริง):**

```irb
irb> Sidekiq::RetrySet.new.size
=> 0   # ตลอดกระบวนการ AjRetryJob ไม่เคยเข้า Sidekiq retry set เลยแม้แต่ครั้งเดียว
```

- ❌ ไม่โผล่ในแท็บ "Retries" ของ Sidekiq Web UI เลย (เพราะไม่เคยเข้า retry set จริงๆ)
- ❌ ไม่มีข้อมูล error/backtrace เก็บไว้ให้ดูใน Sidekiq UI (retry ของ ActiveJob ไม่ได้บันทึกแบบ
  เดียวกับ Sidekiq)
- ❌ กด "Retry now" จาก Sidekiq UI ไม่ได้ เพราะไม่มี entry ให้กด
- ❌ ไม่เข้า Sidekiq dead set เมื่อ retry ครบ (ActiveJob จัดการ "หมดจำนวนครั้ง" เองแยกต่างหาก
  ผ่าน `after_discard`/`retry_stopped` callback)
- ✅ แต่ portable — ถ้าเปลี่ยนไปใช้ Solid Queue หรือ adapter อื่น โค้ด `retry_on` ตัวนี้ยังทำงาน
  เหมือนเดิมทุกอย่าง เพราะเป็น mechanism ระดับ ActiveJob ไม่ใช่ Sidekiq

### ระบบที่ 2: `sidekiq_options retry:` บน ActiveJob class

```ruby
class ReportJob < ApplicationJob
  queue_as :low
  sidekiq_options retry: 5   # ไม่ใช้ retry_on เลย ปล่อยให้ exception หลุดออกไปให้ Sidekiq จัดการ

  def perform(report_id)
    # ...
  end
end
```

แบบนี้คือปล่อยให้ exception ลอยออกจาก `perform` ตามปกติ (ไม่ rescue เอง) — Sidekiq จะเป็นคนจับ
exception แล้วเข้า retry set ของ Sidekiq เอง ได้ behavior ครบทุกอย่างเหมือน native
`Sidekiq::Job`: โผล่ในแท็บ Retries, มี backtrace ให้ดู, กด retry/kill ได้จาก Web UI, เข้า dead
set เมื่อครบจำนวน

### สรุปกฎการเลือกใช้

| ใช้ `retry_on`/`discard_on` เมื่อ... | ใช้ `sidekiq_options retry:` (ปล่อย exception หลุด) เมื่อ... |
|---|---|
| ต้องการ retry logic ที่ portable ข้าม adapter | ผูกกับ Sidekiq อยู่แล้ว ต้องการเห็น retry ทั้งหมดใน Sidekiq Web UI |
| ต้องการ retry แบบ conditional ซับซ้อนตาม exception class หลายแบบพร้อม custom `wait:` ต่อ exception | ต้องการ backoff algorithm ของ Sidekiq (หรือ custom ผ่าน `sidekiq_retry_in`) และ dead set ที่มองเห็นได้จาก UI |
| งานที่ error ไม่ค่อยเยอะ ไม่จำเป็นต้อง monitor ผ่าน dashboard | งาน critical ที่ทีม ops ต้อง monitor retry/dead job อย่างใกล้ชิดทุกวัน |

**ในทางปฏิบัติ ทีมที่ใช้ Sidekiq จริงจังส่วนใหญ่เลือกทางที่ 2** (ปล่อยให้ Sidekiq จัดการ retry
เอง ผ่าน `sidekiq_options`) เพราะได้ observability ผ่าน Web UI ครบ — และค่อยใช้ `discard_on` ของ
ActiveJob เฉพาะกรณีพิเศษที่ต้องการ "เจอ error แบบนี้แล้วทิ้งเงียบๆ ไม่ต้อง retry เลย" เช่น
`discard_on ActiveJob::DeserializationError` (record ที่ job อ้างถึงถูกลบไปแล้ว ไม่มีประโยชน์
จะ retry)

---

## Step 618: Idempotency — ทำไม job อาจถูกรันซ้ำ และวิธีออกแบบให้ปลอดภัย

### "At least once" ไม่ใช่ "Exactly once"

Sidekiq (เหมือน background job system ส่วนใหญ่) รับประกันแค่ **"at least once delivery"** —
job จะถูกประมวลผล**อย่างน้อย 1 ครั้ง** แต่**ไม่รับประกันว่าจะพอดี 1 ครั้ง** สถานการณ์ที่ทำให้
job ถูกรันซ้ำเกิดขึ้นได้จริงในโลกจริง เช่น:

1. **Worker process ถูก kill กลางคัน** — เช่น deploy ใหม่ทับ container เก่า (`SIGKILL`) ขณะ
   job กำลังรันอยู่ครึ่งทาง งานข้างในอาจทำสำเร็จไปแล้วบางส่วน (เช่น เรียก payment gateway
   สำเร็จแล้ว) แต่ Sidekiq ยังไม่ทันบันทึกว่า job นี้เสร็จ (เพราะไม่มีการ "ack" กลับไปที่ Redis
   จนกว่า `perform` จะ return) → เมื่อ process ใหม่ขึ้นมา job จะถูกดึงมารันใหม่ตั้งแต่ต้น
2. **Network hiccup ระหว่าง Sidekiq กับ Redis** ตอนกำลังจะลบ job ออกจาก queue หลังทำสำเร็จ
3. **Retry ที่เกิดจาก exception ที่ไม่เกี่ยวกับ business logic** เช่น timeout ตอนเขียนผลลัพธ์
   ลง log แต่งานหลักได้ทำสำเร็จไปแล้ว
4. **คนกด "Retry now" ซ้ำหลายทีจาก Web UI โดยไม่ทันสังเกตว่า job แรกผ่านไปแล้ว**

เพราะฉะนั้น **ทุก job ที่มีผลข้างเคียง (side effect) ที่ไม่อยากให้เกิดซ้ำ ต้องออกแบบให้เป็น
idempotent** — เรียกกี่ครั้งก็ได้ผลลัพธ์เหมือนเรียกครั้งเดียว

### วิธีออกแบบ idempotency ที่ใช้บ่อยที่สุด: processed-flag guard

```ruby
class ProcessOrderJob
  include Sidekiq::Job
  sidekiq_options queue: "high", retry: 5

  def perform(order_id)
    order = Order.find(order_id)

    # guard ตรวจสอบก่อนทำงานที่มีผลข้างเคียง (เรียก external API ที่คิดเงินจริง)
    if order.payment_processed?
      Rails.logger.info("[ProcessOrderJob] order=#{order_id} already processed, skipping")
      return
    end

    charge_payment!(order)                                  # เรียก payment gateway จริง
    order.update!(payment_processed: true, status: "paid")  # mark ว่าทำแล้ว
  end
end
```

**ทดสอบจริงแล้ว:** เรียก `ProcessOrderJob.new.perform(order.id)` ซ้ำอีกครั้งบน order ที่
`payment_processed?` เป็น `true` แล้ว — method คืนค่าทันทีโดยไม่แตะ `charge_payment!` เลย
(ไฟล์ log ที่นับจำนวนครั้งที่เรียก payment gateway ยังคงมี 3 บรรทัดเท่าเดิม ไม่เพิ่มเป็น 4)
ยืนยันว่า guard ทำงานถูกต้อง 100%

### เทคนิคเสริมอื่นๆ สำหรับ idempotency

**1. ใช้ database unique constraint แทนการเช็คด้วยโค้ดอย่างเดียว** — ถ้ามี race condition
(2 worker thread รัน job เดียวกันพร้อมกันพอดี) การเช็ค `if` เฉยๆ ไม่ปลอดภัย 100% ต้องพึ่งฐาน
ข้อมูลช่วยด้วย:

```ruby
class SendWelcomeEmailJob
  include Sidekiq::Job

  def perform(user_id)
    # unique index บน (user_id) ที่ตาราง sent_emails ป้องกันการ insert ซ้ำระดับ DB
    SentEmail.create!(user_id: user_id, kind: "welcome")
    UserMailer.welcome(user_id).deliver_now
  rescue ActiveRecord::RecordNotUnique
    Rails.logger.info("Welcome email already sent to user=#{user_id}, skipping")
  end
end
```

**2. ออกแบบ operation ให้เป็น idempotent โดยธรรมชาติ (แนะนำที่สุดถ้าทำได้)** — แทนที่จะ
"บวกเพิ่ม" ให้ "set ค่าสุดท้าย" แทน:

```ruby
# ❌ ไม่ idempotent — รันซ้ำแล้วยอดจะบวกซ้ำ
def perform(user_id, amount)
  User.find(user_id).increment!(:credit_balance, amount)
end

# ✅ idempotent — รันซ้ำกี่ครั้งก็ได้ผลลัพธ์เดียวกัน เพราะระบุ "จำนวนครั้งที่" ไม่ใช่ "บวกอีกเท่าไหร่"
def perform(user_id, transaction_id, amount)
  return if CreditTransaction.exists?(transaction_id: transaction_id)
  CreditTransaction.create!(transaction_id: transaction_id, user_id: user_id, amount: amount)
  User.find(user_id).update!(credit_balance: User.find(user_id).credit_transactions.sum(:amount))
end
```

**3. ใช้ Sidekiq Enterprise's unique jobs (ถ้ามีงบ)** — ฟีเจอร์ paid tier ที่ป้องกันไม่ให้ job
ที่มี argument เดียวกันถูก enqueue ซ้ำในช่วงเวลาที่กำหนด แต่นี่แก้ปัญหาแค่ "enqueue ซ้ำ" ไม่ใช่
"at least once delivery" ที่มาจากขั้นตอน processing เอง — **ยังต้องทำ idempotency guard ในระดับ
โค้ดควบคู่กันเสมอ** ไม่ควรพึ่ง unique jobs อย่างเดียว

> **หลักคิดสำคัญที่สุด:** อย่าคิดว่า "job ของฉันไม่น่าจะรันซ้ำหรอก" — ให้คิดตั้งแต่ต้นว่า **job
> ทุกตัวจะถูกเรียกซ้ำสักวันหนึ่งแน่นอน** (ไม่ใช่ "ถ้า" แต่เป็น "เมื่อไหร่") โดยเฉพาะ job ที่แตะ
> เงิน, ส่งอีเมล/SMS ให้ลูกค้า, หรือเรียก API ภายนอกที่มีผลข้างเคียงจริง

---

## Step 619: Scheduled/delayed job ด้วย `perform_in`/`perform_at`

Sidekiq เก็บ job ที่ยังไม่ถึงเวลาไว้ใน Redis sorted set ชื่อ **scheduled set** แล้วมี process
ภายในชื่อ `Scheduled::Poller` คอยเช็คทุกๆ ไม่กี่วินาที (ปรับได้ด้วย `:poll_interval_average` ใน
`config/sidekiq.yml`) ว่ามี job ไหนถึงเวลาแล้วบ้าง ถ้าถึงเวลาแล้วก็ย้ายเข้า queue ปกติทันที

### `perform_in` — delay เป็นจำนวนวินาที

```ruby
HardJob.perform_in(5, "bob", 2)          # รันอีก 5 วินาทีข้างหน้า
HardJob.perform_in(1.hour, "bob", 2)     # ActiveSupport::Duration ก็ใช้ได้ (แปลงเป็นวินาทีอัตโนมัติ)
```

**ทดสอบจริงแล้ว:** enqueue `HardJob.perform_in(5, "bob", 2)` แล้วเช็คทันที —

```irb
irb> Sidekiq::ScheduledSet.new.size
=> 1
```

job อยู่ใน scheduled set (ยังไม่เข้า queue ปกติ) พอรอครบ ~5 วินาที log แสดงว่า job ถูกดึงมา
ประมวลผลจริง

### `perform_at` — กำหนดเวลาที่แน่นอน

```ruby
HardJob.perform_at(5.seconds.from_now, "future", 99)
HardJob.perform_at(Time.zone.parse("2026-12-25 09:00:00"), "future", 99)
```

**ทดสอบจริงแล้ว:** `HardJob.perform_at(5.seconds.from_now, "future", 99)` — job ถูกประมวลผล
ตรงเวลาที่กำหนดพอดี (คลาดเคลื่อนในระดับ poll interval เท่านั้น ไม่ใช่คลาดเคลื่อนแบบสุ่มเหมือน
retry backoff ที่มี jitter)

### ผ่าน ActiveJob ก็ใช้ syntax เดิมของ Rails ได้

```ruby
ReportJob.set(wait: 5.seconds).perform_later(99)
ReportJob.set(wait_until: Date.tomorrow.noon).perform_later(99)
```

**ทดสอบจริงแล้ว** — ทั้งสองแบบ enqueue ผ่าน adapter ของ Sidekiq สำเร็จ และ job ถูกประมวลผลตรง
เวลาที่กำหนดเป๊ะเช่นกัน (Sidekiq adapter แปลง `wait`/`wait_until` เป็น `perform_at` ให้อัตโนมัติ
ผ่าน `Setter#at` ภายใน)

### ใช้ทำอะไรได้บ้างในทางปฏิบัติ

- **Follow-up job** — หลังทำ step หนึ่งเสร็จ ให้ schedule step ถัดไปให้รันอีกสักพัก (ตัวอย่างใน
  แบบฝึกหัดท้าย Part นี้: `ProcessOrderJob` เสร็จแล้ว schedule `SendOrderConfirmationJob` ให้
  รันอีก 5 วินาทีถัดไป)
- **Reminder/notification ล่วงหน้า** — เช่น ส่งอีเมลเตือนก่อนวันนัดหมาย 1 วัน
- **Rate limiting เชิง manual** — กระจาย job จำนวนมากให้ทยอยรันห่างกัน แทนที่จะยิงพร้อมกันหมด
- **Delayed retry แบบ custom** — เช่น รอ 24 ชั่วโมงก่อนลองเรียก API ภายนอกใหม่ถ้าล้มเหลวแบบที่รู้
  อยู่แล้วว่าต้องรอนาน (เช่น rate limit ของ third-party API)

> **ข้อควรรู้:** scheduled job ไม่ใช่ cron job — ถ้าต้องการ job ที่รันซ้ำตามตารางเวลาเป็นประจำ
> (เช่น "ทุกวันตี 2") ต้องใช้ gem เสริมอย่าง `sidekiq-cron` หรือ `sidekiq-scheduler` เพิ่มต่างหาก
> `perform_in`/`perform_at` คือ "รันครั้งเดียวในอนาคต" เท่านั้น

---

## Step 620: Monitoring และ observability ผ่าน Sidekiq Web UI และ Sidekiq API

### หน้าต่างๆ ของ Sidekiq Web UI (ทดสอบจริงทุก path)

หลัง mount `Sidekiq::Web => "/sidekiq"` แล้ว มี sub-page ที่มีประโยชน์มากสำหรับ debug/monitor:

| Path | แสดงอะไร |
|------|----------|
| `/sidekiq` | Dashboard สรุปภาพรวม: จำนวน job ที่ processed/failed สะสม, กราฟ 24 ชม.-1 ปี, จำนวน busy/enqueued/scheduled/retries/dead ปัจจุบัน |
| `/sidekiq/busy` | Job ที่กำลังประมวลผลอยู่ตอนนี้จริงๆ (real-time) รวมถึงรายชื่อ Sidekiq process ที่ต่ออยู่ทั้งหมดและ thread ของแต่ละ process |
| `/sidekiq/queues` | รายชื่อ queue ทั้งหมดพร้อมจำนวน job ที่รอในแต่ละคิว กด latency ดูได้ว่า job เก่าสุดในคิวรอมานานแค่ไหนแล้ว (บอกได้ว่าคิวตันหรือเปล่า) |
| `/sidekiq/retries` | Job ที่กำลังรอ retry อยู่ พร้อม error message, เวลา retry ครั้งถัดไป, ปุ่ม "Retry now" / "Delete" |
| `/sidekiq/scheduled` | Job ที่ตั้ง `perform_in`/`perform_at` ไว้ล่วงหน้า ยังไม่ถึงเวลา |
| `/sidekiq/morgue` | **Dead set** (ทดสอบจริงแล้วว่า path คือ `morgue` ไม่ใช่ `/dead` — เข้า `/sidekiq/dead` ตรงๆ จะได้ 404) แสดง job ที่ retry ครบแล้วยังไม่สำเร็จ พร้อม full backtrace |
| `/sidekiq/metrics` | Metrics ระดับ per-job-class (execution time percentile ฯลฯ ถ้าเปิดใช้ Sidekiq metrics) |

**ทดสอบจริง:** หลังรัน `DoomedJob` จน fail เข้า dead set แล้ว เปิด `/sidekiq/morgue` ผ่าน curl
(พร้อม basic auth) — พบคำว่า `DoomedJob` อยู่ในหน้า HTML ที่ตอบกลับมาจริง ยืนยันว่า record
แสดงผลถูกต้อง

### ตรวจสอบ/ควบคุมผ่าน `Sidekiq::API` โดยตรง (ไม่ผ่านหน้าเว็บ)

มีประโยชน์มากสำหรับเขียน monitoring script เอง, health check endpoint, หรือ admin task:

```ruby
require "sidekiq/api"

# ขนาดคิว
Sidekiq::Queue.new("high").size      # => จำนวน job ที่รออยู่ใน queue "high"
Sidekiq::Queue.new("default").size

# ดู job แต่ละตัวในคิว
Sidekiq::Queue.new("high").each { |job| puts "#{job.klass} #{job.args}" }

# scheduled set
Sidekiq::ScheduledSet.new.size

# retry set — วนดู error ของแต่ละ job ที่กำลังรอ retry
Sidekiq::RetrySet.new.each do |job|
  puts "#{job.klass} retry_count=#{job['retry_count']} error=#{job['error_message']} at=#{job.at}"
end

# dead set — สั่ง retry job ที่ตายแล้วกลับเข้าคิวใหม่ทั้งหมด
Sidekiq::DeadSet.new.each(&:retry)

# สถิติรวมของทั้งระบบ
stats = Sidekiq::Stats.new
stats.processed      # จำนวน job ที่ประมวลผลสำเร็จสะสมทั้งหมด
stats.failed         # จำนวนครั้งที่ fail สะสมทั้งหมด
stats.enqueued       # จำนวน job ที่รออยู่รวมทุกคิวตอนนี้
stats.workers_size   # จำนวน worker thread ที่กำลังทำงานอยู่ตอนนี้

# ดู process ที่ต่ออยู่ทั้งหมด (ถ้ามีหลายเครื่อง/หลาย container)
Sidekiq::ProcessSet.new.each do |process|
  puts "#{process['hostname']} concurrency=#{process['concurrency']} busy=#{process['busy']}"
end
```

### แนวทาง monitoring ระดับ production

1. **Export metrics ไปยัง monitoring system ภายนอก** (Datadog, New Relic, Prometheus) —
   ส่วนใหญ่ทำผ่าน scheduled task ที่อ่านค่าจาก `Sidekiq::Stats` ทุกๆ 1 นาทีแล้วส่งออกเป็น
   metric เช่น `sidekiq.queue.high.size`, `sidekiq.dead_set.size`
2. **Alert เมื่อ queue latency สูงผิดปกติ** — ถ้า job เก่าสุดในคิวรอมานานกว่า threshold ที่ตั้ง
   ไว้ (เช่น 5 นาที) แปลว่า worker ไม่พอ หรือ worker ตายหมด ต้อง alert ทันที
3. **Alert เมื่อ dead set โตขึ้นเรื่อยๆ** — dead set ที่มี item ค้างอยู่แปลว่ามีบางอย่างพังจริง
   ต้องมีคนไปดู ไม่ใช่ปล่อยไว้เฉยๆ
4. **ใช้ `sidekiq_retries_exhausted` เชื่อมกับระบบแจ้งเตือน** (Slack/PagerDuty/Sentry) แทนการ
   รอให้คนเปิด Web UI มาเช็คเอง — โดยเฉพาะ job ที่กระทบเงินหรือลูกค้าโดยตรง

---

## แบบฝึกหัด: ระบบประมวลผลคำสั่งซื้อแบบ retry + idempotent + scheduled follow-up

### โจทย์

สร้างระบบประมวลผล order ที่ต้องทำสิ่งต่อไปนี้ให้ครบ (โจทย์นี้ถูกเขียนและทดสอบจริงกับ Sidekiq +
Redis แล้วทั้งหมดในการเตรียม Part นี้):

1. `ProcessOrderJob` — เรียก "payment gateway" (จำลอง) เพื่อคิดเงิน order
   - ใช้ `sidekiq_options retry: 5` และกำหนด custom backoff ผ่าน `sidekiq_retry_in` เป็นแบบ
     เชิงเส้น (10, 20, 30, 40, 50 วินาที) แทนสูตร default ของ Sidekiq
   - ต้อง **idempotent** — ถ้า order ถูก process (คิดเงิน) สำเร็จไปแล้ว การเรียกซ้ำต้อง skip
     ทันทีโดยไม่คิดเงินซ้ำ
   - เมื่อคิดเงินสำเร็จ ให้ schedule `SendOrderConfirmationJob` ด้วย `perform_in` ให้รันอีก 5
     วินาทีถัดไป
   - ถ้า retry ครบ 5 ครั้งแล้วยังไม่สำเร็จ (`sidekiq_retries_exhausted`) ให้ log แจ้งเตือนทีมงาน
2. `SendOrderConfirmationJob` — ส่ง "อีเมลยืนยัน" (จำลอง) ก็ต้อง idempotent เช่นกัน

### เฉลย

**Migration:**

```ruby
# db/migrate/xxxxxx_create_orders.rb
class CreateOrders < ActiveRecord::Migration[8.1]
  def change
    create_table :orders do |t|
      t.string :status, null: false, default: "pending"
      t.boolean :payment_processed, null: false, default: false
      t.boolean :confirmation_sent, null: false, default: false

      t.timestamps
    end
  end
end
```

**`app/jobs/process_order_job.rb`:**

```ruby
class ProcessOrderJob
  include Sidekiq::Job

  sidekiq_options queue: "high", retry: 5

  # backoff เชิงเส้น 10, 20, 30, 40, 50 วินาที แทนสูตร exponential default ของ Sidekiq
  sidekiq_retry_in do |count, exception, jobhash|
    10 * (count + 1)
  end

  # เมื่อ retry ครบ 5 ครั้งแล้วยังไม่สำเร็จ (เข้า dead set) แจ้งเตือนทีมงาน
  sidekiq_retries_exhausted do |job, exception|
    Rails.logger.error(
      "[ProcessOrderJob] order_id=#{job['args'].first} exhausted all retries: #{exception.message}"
    )
  end

  def perform(order_id)
    order = Order.find(order_id)

    # --- Idempotency guard ---
    # Sidekiq การันตีแค่ "at least once" ไม่ใช่ "exactly once" เพราะฉะนั้น perform
    # ตัวนี้อาจถูกเรียกซ้ำได้ (เช่น worker ถูก kill กลางคันหลัง process เสร็จแต่ก่อน ack)
    # จึง guard ด้วย flag ในฐานข้อมูลก่อนทำงานที่มีผลข้างเคียงจริง
    if order.payment_processed?
      Rails.logger.info("[ProcessOrderJob] order=#{order_id} already processed, skipping")
      return
    end

    charge_payment!(order)

    order.update!(payment_processed: true, status: "paid")

    # ส่ง job ตามไปทำงานต่อแบบ scheduled (delay 5 วินาทีถัดจากนี้)
    SendOrderConfirmationJob.perform_in(5, order.id)
  end

  private

  def charge_payment!(order)
    # ในสถานการณ์จริงตรงนี้จะเป็นการเรียก Stripe/Omise API จริง
    ExternalPaymentGateway.charge!(order_id: order.id, amount: order.total)
  end
end
```

**`app/jobs/send_order_confirmation_job.rb`:**

```ruby
class SendOrderConfirmationJob
  include Sidekiq::Job

  sidekiq_options queue: "default", retry: 3

  def perform(order_id)
    order = Order.find(order_id)

    if order.confirmation_sent?
      Rails.logger.info("[SendOrderConfirmationJob] order=#{order_id} confirmation already sent, skipping")
      return
    end

    OrderMailer.confirmation(order).deliver_now

    order.update!(confirmation_sent: true, status: "completed")
  end
end
```

### ผลการทดสอบจริง (รันกับ Sidekiq 8.1.7 + Redis 7.0 จริง ไม่ใช่การจำลอง)

จำลอง `charge_payment!` ให้ล้มเหลว 2 ครั้งแรกแล้วสำเร็จครั้งที่ 3 (เพื่อบังคับให้เห็น retry
behavior จริง) ผลลัพธ์ที่วัดได้:

```
สร้าง order id=1 (status=pending)
ProcessOrderJob.perform_async(1)

attempt 1 ที่ 08:12:46  -> raise "payment gateway timeout (attempt 1)"
  (Sidekiq log: elapsed=0.102: fail, retry scheduled ด้วย custom backoff 10s+jitter)

attempt 2 ที่ 08:13:06  -> ห่างจากครั้งก่อน 20 วินาที (ตรงกับ count=0 → 10s + jitter) -> fail อีก
  (retry ครั้งถัดไปตั้งไว้ด้วย count=1 → 20s + jitter)

attempt 3 ที่ 08:13:29  -> ห่างจากครั้งก่อน 23 วินาที (ตรงกับ count=1 → 20s + jitter) -> สำเร็จ!
  order.update!(payment_processed: true, status: "paid")
  SendOrderConfirmationJob.perform_in(5, 1) ถูก enqueue

08:13:35 (5 วินาทีถัดมา) -> SendOrderConfirmationJob รัน:
  "confirmation sent for order=1"
  order.update!(confirmation_sent: true, status: "completed")

ตรวจสอบ order สุดท้าย:
{status: "completed", payment_processed: true, confirmation_sent: true}
```

**ทดสอบ idempotency แยกต่างหาก:** เรียก `ProcessOrderJob.new.perform(1)` ซ้ำอีกครั้งบน order
ที่เสร็จแล้ว — ได้ log `"already processed, skipping"` และไฟล์ log ที่นับจำนวนครั้งที่เรียก
`charge_payment!` ยังคงมี 3 บรรทัดเท่าเดิม **ไม่เพิ่มเป็น 4** — ยืนยันว่า guard ทำงานถูกต้อง
100% ไม่มีการคิดเงินซ้ำ

**ตรวจสอบ dead set/retry set หลังจบกระบวนการ:**

```irb
irb> Sidekiq::RetrySet.new.size
=> 0   # job สำเร็จแล้ว หลุดออกจาก retry set เรียบร้อย ไม่มีตกค้าง
```

การทดสอบนี้ยืนยันว่าทั้ง 3 กลไก — custom backoff, idempotency guard, scheduled follow-up job —
ทำงานร่วมกันได้ถูกต้องแบบ end-to-end จริง ไม่ใช่แค่ในทฤษฎี

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม unique constraint ระดับฐานข้อมูลให้ `SendOrderConfirmationJob` ปลอดภัยขึ้นอีกชั้น (เผื่อ
   race condition ที่ 2 thread รัน job เดียวกันพร้อมกันพอดี) โดยสร้างตาราง
   `order_confirmations` ที่มี unique index บน `order_id` แล้วใช้ `create!` +
   rescue `ActiveRecord::RecordNotUnique` แทนการเช็ค `if` เฉยๆ
2. เขียน `RefundOrderJob` ที่ใช้ `sidekiq_options retry: 0` (ไม่ retry เลย เพราะ refund ผิดพลาด
   ต้องให้คนตรวจสอบเองเท่านั้น ห้ามลองซ้ำอัตโนมัติ) แล้วเขียน rake task ที่ดึงรายการ
   `Sidekiq::DeadSet` ที่เป็น `RefundOrderJob` ทั้งหมดออกมาแสดงเป็นรายงานประจำวัน
3. ตั้งค่า `config/sidekiq.yml` ให้มี 4 queue: `critical`, `high`, `default`, `low` ด้วย weight
   `4, 3, 2, 1` ตามลำดับ แล้วเขียนสคริปต์ enqueue job เข้าแต่ละคิว 100 ตัวพร้อมกัน สังเกตจาก log
   ว่าสัดส่วนการประมวลผลจริงใกล้เคียงสัดส่วน weight ที่ตั้งไว้หรือไม่ (ทดลองปรับ `:concurrency`
   ค่าต่างๆ ดูผลกระทบด้วย)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจเหตุผลเชิงเทคนิคและ operational ว่าทำไม Sidekiq ยังเป็นตัวเลือกที่ดีสำหรับหลายสถานการณ์
  แม้ Rails 8 จะมี Solid Queue เป็นค่าเริ่มต้นแล้ว และรู้ว่าเมื่อไหร่ควรเลือกตัวไหน
- ติดตั้ง Sidekiq + Redis, ตั้งค่า `queue_adapter`, รัน Sidekiq process จริง และ mount
  Sidekiq Web UI พร้อม authentication อย่างปลอดภัย (ทดสอบยืนยันด้วย HTTP status code จริง)
- เขียน job ได้ทั้ง 2 แบบ (native `Sidekiq::Job` และผ่าน `ApplicationJob`) พร้อมเข้าใจ tradeoff
  เรื่อง portability กับ feature เฉพาะทาง
- กำหนด queue, priority (weight), และ concurrency ผ่าน `config/sidekiq.yml` ได้
- เข้าใจสูตร exponential backoff ของ Sidekiq (`count**4 + 15 + jitter`) แบบอ่านจาก source code
  จริง และรู้จัก dead set/`sidekiq_retries_exhausted`
- แยกความแตกต่างระหว่าง `retry_on`/`discard_on` ของ ActiveJob กับ retry ของ Sidekiq เอง —
  พิสูจน์แล้วว่า `retry_on` ไม่โผล่ใน Sidekiq retry set เลย
- ออกแบบ job ให้ **idempotent** ได้อย่างถูกต้อง เข้าใจว่า "at least once delivery" หมายความว่า
  อย่างไรในทางปฏิบัติ
- ใช้ `perform_in`/`perform_at` สร้าง scheduled/delayed job และเข้าใจกลไก scheduled set
- Monitor ระบบผ่าน Sidekiq Web UI ทุกหน้า และผ่าน `Sidekiq::API` โดยตรงในโค้ด

**ต่อไป (Part 063):** เราจะเปลี่ยนโฟกัสจาก background job ไปที่ **Caching** — fragment cache,
Russian doll caching (cache ที่ซ้อนกันแบบแม่ลูก invalidate อัตโนมัติเมื่อลูกเปลี่ยน), และ
low-level caching ด้วย `Rails.cache` — เทคนิคที่ช่วยลด load ของ database และเพิ่มความเร็วของหน้า
เว็บได้อย่างมหาศาลโดยไม่ต้องเปลี่ยนโครงสร้างโค้ดทั้งระบบ
