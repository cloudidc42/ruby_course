# Part 077: Deployment ทางเลือก (Render/Fly.io/Heroku) และ Database Migration ใน Production

> **Step ครอบคลุมใน Part นี้:** Step 761–770
> **ระดับ:** กลาง–สูง (ควรผ่าน Part 073–076 มาก่อน โดยเฉพาะเรื่อง Docker และ Kamal)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / PostgreSQL 16

> **หมายเหตุเรื่องความน่าเชื่อถือของเนื้อหาใน Part นี้ (อ่านก่อนเริ่ม):**
>
> Part นี้แบ่งเป็น 2 ส่วนที่มีลักษณะการยืนยันความถูกต้องต่างกัน ต้องบอกตรงๆ ให้ชัดเจน:
>
> 1. **ส่วน PaaS platform (Step 761–764)** — รายละเอียดของ Render/Fly.io/Heroku (หน้าตา
>    dashboard, ราคา, พฤติกรรมของปุ่มต่างๆ) เป็นเนื้อหาที่ **เรียบเรียงจากเอกสารทางการของ
>    แต่ละแพลตฟอร์มและข้อมูลที่ค้นล่าสุด ไม่ใช่การ deploy จริง** เพราะ sandbox ที่ใช้เขียน
>    หลักสูตรนี้ไม่มีบัญชีจริงบน Render/Fly.io/Heroku ให้ทดสอบ ไฟล์ config ตัวอย่าง
>    (`render.yaml`, `fly.toml`, `Procfile`) เขียนให้ถูกต้องตาม syntax ที่เอกสารระบุ แต่ยังไม่ได้
>    ผ่านการรัน deploy จริงบนแพลตฟอร์มนั้นๆ
> 2. **ส่วน production migration safety (Step 765–770)** — นี่คือแก่นวิชาการของ Part นี้ และ
>    **ทุกตัวอย่างถูกทดสอบจริง (live-verified)** ด้วยการสร้างแอป Rails 8.1.4 จริงบน
>    PostgreSQL 16 ใส่ข้อมูลจริง 200,000 แถว แล้วรัน migration อันตรายจริงจนเห็นการ lock/error
>    จริง เทียบกับแบบปลอดภัยที่ทำสำเร็จจริง พร้อม capture ผลลัพธ์จาก terminal จริงมาให้ดู
>    ตัวเลขเวลาทั้งหมดที่เห็นใน Step 766–769 คือผลจริงจากการรันบนเครื่องนี้ ไม่ใช่ตัวเลขสมมติ

## สารบัญของ Part นี้

- Step 761: ทำไมต้องมี Managed PaaS — ทางเลือกเมื่อไม่อยากดูแลเซิร์ฟเวอร์เอง
- Step 762: Deploy Rails บน Render แบบละเอียด (`render.yaml` อธิบายทีละบรรทัด)
- Step 763: Fly.io และ Heroku — `fly.toml`, `Procfile`, และสิ่งที่แต่ละแพลตฟอร์ม abstract ไป
- Step 764: Environment Variables และ Secrets บน Managed Platform
- Step 765: Release/Build Hook — จุดที่ `db:migrate` รันอัตโนมัติ และอันตรายที่แฝงอยู่
- Step 766: Postgres Lock 101 — ทำไม `ALTER TABLE` ถึงทำให้ทั้งแอปค้างได้ (สาธิตจริง)
- Step 767: Safe vs Dangerous Migration Patterns (สาธิตจริงทั้งสองแบบ)
- Step 768: Multi-step Safe Pattern สำหรับ Rename/Remove Column (Expand-Contract)
- Step 769: `strong_migrations` — ให้ Ruby ดักจับ migration อันตรายก่อนถึง production
- Step 770: Pre-deploy Checklist และแบบฝึกหัดรวบยอด

---

## Step 761: ทำไมต้องมี Managed PaaS — ทางเลือกเมื่อไม่อยากดูแลเซิร์ฟเวอร์เอง

ใน Part 076 เราเรียนรู้ Kamal ซึ่งให้เรา deploy Rails app ไปยัง VPS (เช่น DigitalOcean, Hetzner,
Linode) ที่เราต้องดูแลเอง — ติดตั้ง Docker, จัดการ SSL, monitor เซิร์ฟเวอร์, ทำ backup เอง
วิธีนี้ให้ control สูงสุดและ**ถูกที่สุดในระยะยาว** แต่แลกมาด้วยภาระงาน operations ที่ต้องรับผิดชอบเอง
ทั้งหมด

**Managed PaaS (Platform as a Service)** คือแนวทางตรงข้าม: เรา "เช่า" แพลตฟอร์มที่จัดการ
เซิร์ฟเวอร์, load balancer, SSL certificate, health check, log aggregation ให้เราทั้งหมด สิ่งที่เรา
ทำคือ push โค้ด (หรือ container image) แล้วแพลตฟอร์มจะ build และรันให้ ตัวอย่างที่ได้รับความนิยม
ในวงการ Rails คือ **Render**, **Fly.io**, และ **Heroku**

### เมื่อไหร่ควรเลือก Managed PaaS แทน Kamal + VPS

| สถานการณ์ | แนวทางที่เหมาะกว่า |
|-----------|---------------------|
| ทีมเล็ก/solo developer ไม่มีคนดูแล infra เฉพาะทาง | Managed PaaS |
| ต้องการ deploy ให้เสร็จภายในไม่กี่นาทีแรกของโปรเจกต์ (MVP, prototype) | Managed PaaS |
| งบประมาณจำกัดมากและมีเวลาดูแลเซิร์ฟเวอร์เอง | Kamal + VPS ราคาถูก |
| ต้องการ control เต็มรูปแบบ (custom kernel tuning, compliance เฉพาะทาง) | Kamal + VPS หรือ self-hosted Kubernetes |
| ทีมมี DevOps engineer อยู่แล้ว และ scale ใหญ่พอที่ต้นทุน PaaS จะแพงกว่า VPS มาก | Kamal หรือ Kubernetes |
| Side project ที่อยากลืมเรื่อง infra ไปเลย | Managed PaaS |

ไม่มีคำตอบที่ถูกเสมอ — บริษัทระดับ unicorn บางแห่งเริ่มต้นบน Heroku แล้วค่อยย้ายออกตอนต้นทุน
เริ่มสูงเกินไป (เช่น GitHub เคยใช้ Heroku ในช่วงแรก) ในขณะที่หลายทีมยังอยู่บน PaaS ไปตลอด
เพราะความเรียบง่ายคุ้มกว่าต้นทุนที่จ่ายเพิ่ม

### ตารางเปรียบเทียบ Render vs Fly.io vs Heroku

| มิติ | Render | Fly.io | Heroku |
|------|--------|--------|--------|
| **โมเดลราคา** | Flat-rate ต่อ service (เช่น instance เริ่มต้นถูกกว่า Heroku พอสมควร) มี free tier แบบจำกัด (sleep เมื่อไม่มี traffic) | จ่ายตามทรัพยากรที่ใช้จริง (CPU/RAM/region) ยืดหยุ่นกว่าแต่คาดเดาราคาล่วงหน้ายากกว่า | ไม่มี free tier แล้วตั้งแต่ปลายปี 2022 เริ่มต้นที่ Eco dyno ราคาถูกแต่ sleep ได้ ราคาสูงขึ้นเร็วเมื่อ scale |
| **ความง่ายในการตั้งค่า** | สูงมาก — connect repo แล้ว deploy อัตโนมัติจาก git push, UI คล้าย Heroku ยุคเก่า | ปานกลาง — ต้องคุ้นเคยกับ CLI (`flyctl`) และแนวคิด "Machines" ควบคุมได้ละเอียดกว่าแต่ต้องเรียนรู้เพิ่ม | สูงมาก — เป็นต้นแบบของ PaaS ยุคแรก `git push heroku main` เรียบง่ายที่สุด |
| **รองรับ Rails โดยเฉพาะ** | ตรวจจับ Rails ได้อัตโนมัติ, รองรับ Postgres/Redis เป็น managed add-on ในตัว | รองรับผ่าน Dockerfile/buildpack, มี `fly launch` ที่ scan Rails app ให้อัตโนมัติ | รองรับ Rails มาตั้งแต่ยุคแรกๆ (Heroku คือ PaaS ที่ Rails community ใช้มากที่สุดในอดีต) buildpack เสถียรมาก |
| **Background worker (Sidekiq)** | ประกาศเป็น service แยกชนิด `worker` ใน `render.yaml` ได้ตรงๆ | ต้องรัน process แยกผ่าน `fly.toml` process group หรือแยก app | ประกาศใน `Procfile` บรรทัด `worker:` ตรงไปตรงมาที่สุด |
| **Database ในตัว** | PostgreSQL/Redis เป็น managed service ผูกกับโปรเจกต์ได้ทันที | Postgres ผ่าน Fly Postgres (คลัสเตอร์ที่ยังจัดการเองระดับหนึ่ง) หรือใช้ external provider | Heroku Postgres เป็นต้นแบบของ managed Postgres บน PaaS ที่ค่ายอื่นเลียนแบบ |
| **Zero-downtime deploy** | รองรับผ่าน health check + rolling instance (ถ้าตั้ง min instances ≥ 2) | รองรับผ่าน rolling deploy ของ Machines โดย default | รองรับแบบพื้นฐาน (dyno ใหม่ต้อง healthy ก่อนสลับ traffic) |
| **สิ่งที่ abstract ไปจาก Kamal** | ไม่ต้องยุ่งกับ Docker registry, reverse proxy (Traefik/kamal-proxy), SSL renewal, server provisioning เลย | เหมือน Render แต่ยังให้เข้าถึงระดับ VM (Machines) ได้มากกว่า ใกล้เคียง "VPS ที่มี API ดีมาก" | เหมือน Render, เป็นระดับ abstraction สูงสุดในสามตัวนี้ |
| **เหมาะกับ** | ทีมที่อยากได้ประสบการณ์แบบ Heroku ยุคก่อนแต่ราคาสมเหตุสมผลกว่า | ทีมที่ต้องการ edge deployment หลาย region หรือ control ระดับ VM มากกว่า | ทีมที่ยึดติดกับ ecosystem/add-on ของ Heroku อยู่แล้ว หรือไม่กังวลเรื่องราคา |

**สรุปสั้นๆ:** ถ้าเริ่มต้นใหม่ในปี 2026 และอยากได้ประสบการณ์ที่ใกล้เคียง Kamal มากที่สุดในแง่
"เขียน config ไฟล์เดียวแล้ว deploy" แต่ไม่ต้องดูแลเซิร์ฟเวอร์เอง **Render** มักเป็นตัวเลือกแรกที่คน
ในวงการ Rails แนะนำกันในตอนนี้ (แนวทาง declarative YAML ของ Render เข้ากับธรรมชาติ
"convention over configuration" ของ Rails ได้ดี) — ในหัวข้อถัดไปเราจะลง deep dive บน Render
เป็นหลัก แล้วสรุป Fly.io/Heroku แบบเปรียบเทียบ

---

## Step 762: Deploy Rails บน Render แบบละเอียด (`render.yaml` อธิบายทีละบรรทัด)

Render รองรับการประกาศ infrastructure ทั้งหมดผ่านไฟล์ `render.yaml` ที่ root ของ repo (แนวคิด
เดียวกับ "Infrastructure as Code" ที่ `config/deploy.yml` ของ Kamal ใช้) เมื่อ push ไฟล์นี้ขึ้น
Render จะอ่านและสร้าง/อัปเดต service ให้ตรงกับที่ประกาศไว้

### โครงสร้างโปรเจกต์ตัวอย่าง

สมมติเรามีแอป Rails ชื่อ `myapp` ที่มี web process, background worker (Sidekiq), และ
PostgreSQL database

```yaml
# render.yaml
databases:
  - name: myapp-db
    plan: starter
    databaseName: myapp_production
    user: myapp
    postgresMajorVersion: "16"

services:
  - type: web
    name: myapp-web
    runtime: ruby
    plan: starter
    region: singapore
    buildCommand: "./bin/render-build.sh"
    startCommand: "bundle exec puma -C config/puma.rb"
    healthCheckPath: /up
    autoDeploy: true
    numInstances: 2
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false
      - key: WEB_CONCURRENCY
        value: "2"
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString

  - type: worker
    name: myapp-worker
    runtime: ruby
    plan: starter
    buildCommand: "./bin/render-build.sh"
    startCommand: "bundle exec sidekiq -C config/sidekiq.yml"
    envVars:
      - key: RAILS_ENV
        value: production
      - key: RAILS_MASTER_KEY
        sync: false
      - key: DATABASE_URL
        fromDatabase:
          name: myapp-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          type: redis
          name: myapp-redis
          property: connectionString

  - type: redis
    name: myapp-redis
    plan: starter
    ipAllowList: []
```

### อธิบายทีละส่วน

**`databases:` block**

- `name: myapp-db` — ชื่อภายในของ database service ใน Render ใช้อ้างอิงจาก service อื่น
- `plan: starter` — ระดับทรัพยากร (RAM/storage) ยิ่งสูงยิ่งแพง เลือกตามขนาดข้อมูลจริง
- `databaseName` / `user` — ชื่อ database และ role ที่จะถูกสร้างจริงบน Postgres server
- `postgresMajorVersion` — ล็อกเวอร์ชัน Postgres major version ให้ตรงกับที่ทดสอบไว้

**`services:` block (แต่ละรายการคือ 1 process)**

- `type: web` — บอก Render ว่านี่คือ process ที่รับ HTTP traffic (จะได้รับ public URL และ SSL
  certificate ให้อัตโนมัติ)
- `type: worker` — process เบื้องหลังที่ไม่รับ HTTP request ตรง (ใช้สำหรับ Sidekiq)
- `runtime: ruby` — บอกให้ Render ใช้ Ruby buildpack ที่มากับระบบ (ตรวจจับเวอร์ชัน Ruby จาก
  `.ruby-version` หรือ `Gemfile` อัตโนมัติ)
- `buildCommand` — สคริปต์ที่รันตอน build **ทุกครั้งที่ deploy** (ดูรายละเอียดด้านล่าง — นี่คือจุดที่
  `db:migrate` มักถูกใส่ไว้ และเป็นหัวใจของอันตรายที่เราจะพูดถึงใน Step 765)
- `startCommand` — คำสั่งที่รัน process จริงหลัง build เสร็จ
- `healthCheckPath: /up` — Render จะยิง HTTP GET ไปที่ path นี้เป็นระยะเพื่อเช็คว่า instance
  healthy หรือไม่ (Rails 7.1+ มี route `/up` ให้ในตัวผ่าน `Rails::HealthController` อยู่แล้ว ไม่ต้อง
  สร้างเอง) — ถ้า instance ใหม่ไม่ผ่าน health check Render จะไม่สลับ traffic มาให้ ทำให้ deploy
  ที่พังไม่กระทบผู้ใช้จริง (นี่คือ safety net สำคัญที่ VPS ธรรมดาไม่มีให้ฟรี)
- `autoDeploy: true` — deploy อัตโนมัติทุกครั้งที่ push เข้า branch ที่เชื่อมไว้ (เทียบได้กับ
  GitHub Actions ที่ trigger `kamal deploy` ใน Part 075)
- `numInstances: 2` — รันอย่างน้อย 2 instance พร้อมกัน ทำให้ rolling deploy เป็นไปได้ (instance
  เก่ายังรับ traffic อยู่จนกว่า instance ใหม่จะ healthy) — **แต่ข้อนี้เองที่ทำให้ migration แบบ
  breaking-change อันตรายกว่าที่คิด** เพราะช่วงเปลี่ยนผ่านจะมีทั้งโค้ดเก่าและใหม่วิ่งพร้อมกันจริงๆ
  (รายละเอียดใน Step 768)
- `envVars` — ตัวแปรสภาพแวดล้อม แต่ละตัวมีที่มาต่างกัน (ดู Step 764)

**`type: redis` block** — Render มี Redis เป็น managed service แยกได้เหมือน database

### สคริปต์ build (`bin/render-build.sh`)

```bash
#!/usr/bin/env bash
# bin/render-build.sh
set -o errexit  # หยุดทันทีถ้าคำสั่งใดคำสั่งหนึ่ง exit ด้วย error code ที่ไม่ใช่ 0

bundle install
bundle exec rails assets:precompile
bundle exec rails assets:clean
bundle exec rails db:migrate
```

ต้อง `chmod +x bin/render-build.sh` ก่อน commit ไม่เช่นนั้น Render จะรันสคริปต์นี้ไม่ได้

**จุดสำคัญที่ต้องสังเกต:** บรรทัดสุดท้าย `bundle exec rails db:migrate` คือสิ่งที่ทำให้ schema ของ
production database เปลี่ยนแปลง**ทุกครั้งที่ deploy** โดยอัตโนมัติ ไม่มีขั้นตอนอนุมัติเพิ่มเติม —
นี่คือประเด็นหลักที่ Step 765 เป็นต้นไปจะเจาะลึก

---

## Step 763: Fly.io และ Heroku — `fly.toml`, `Procfile`, และสิ่งที่แต่ละแพลตฟอร์ม abstract ไป

### Fly.io: `fly.toml`

Fly.io ใช้แนวคิด "Machines" (micro-VM ที่บูตเร็วมาก) แทน container ทั่วไป ควบคุมผ่านไฟล์
`fly.toml` และ CLI ชื่อ `flyctl`

```toml
# fly.toml
app = "myapp"
primary_region = "sin"

[build]
  dockerfile = "Dockerfile"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"

[deploy]
  release_command = "bin/rails db:migrate"
  strategy = "rolling"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 2

  [[http_service.checks]]
    grace_period = "10s"
    interval = "15s"
    method = "GET"
    path = "/up"
    timeout = "5s"

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"

[processes]
  app = "bundle exec puma -C config/puma.rb"
  worker = "bundle exec sidekiq -C config/sidekiq.yml"
```

จุดที่ต่างจาก Render อย่างชัดเจน:

- **`[deploy] release_command`** — นี่คือ "release phase" ของ Fly.io เทียบเท่ากับการใส่
  `db:migrate` ใน `buildCommand` ของ Render แต่แยกเป็น step ที่ชัดเจนกว่า: Fly.io จะรันคำสั่งนี้
  **ใน Machine ชั่วคราว** ก่อนที่ Machine จริงจะเริ่มรับ traffic ถ้า `release_command` ล้มเหลวหรือ
  timeout การ deploy ทั้งหมดจะถูกยกเลิกและ Machine เก่ายังทำงานต่อ — ฟังดูปลอดภัยกว่า แต่ถ้า
  migration ไปค้างเพราะ lock (ตามที่จะสาธิตใน Step 766) deploy ทั้งก้อนจะค้างรอจนกว่าจะ timeout
  เช่นกัน
- **`[processes]`** — ประกาศหลาย process type ในไฟล์เดียว (`app`, `worker`) ต่างจาก Render ที่
  แยกเป็น service คนละตัว
- **`strategy = "rolling"`** — เหมือน `numInstances: 2` ของ Render แต่ระบุชัดเจนว่าใช้ rolling
  strategy (ทยอยแทนที่ Machine ทีละตัว) เทียบกับ `strategy = "immediate"` ที่แทนที่ทันที (เสี่ยง
  downtime กว่าแต่เร็วกว่า)
- **`primary_region = "sin"`** — Fly.io เด่นเรื่อง multi-region deployment ใกล้ผู้ใช้ (Singapore
  ในตัวอย่างนี้) ซึ่ง Render และ Heroku ไม่ได้เน้นจุดนี้เท่า

### Heroku: `Procfile`

Heroku เรียบง่ายที่สุดในสามตัว — ไม่มีไฟล์ config รวมศูนย์แบบ YAML/TOML ส่วนใหญ่ตั้งค่าผ่าน
Heroku Dashboard หรือ Heroku CLI (`heroku config:set`) ไฟล์เดียวที่ต้องมีคือ `Procfile`

```
# Procfile
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
release: bundle exec rails db:migrate
```

- **บรรทัด `release:`** — คือ "release phase" ของ Heroku (แนวคิดเดียวกับ `release_command`
  ของ Fly.io) รันหลัง build เสร็จ ก่อนสลับ traffic ไปยัง dyno ใหม่ ถ้า release phase ล้มเหลว
  deploy จะถูกยกเลิกและ dyno เดิมทำงานต่อ
- Heroku ไม่มีไฟล์สำหรับประกาศ database/redis เพราะจัดการผ่าน add-on marketplace
  (`heroku addons:create heroku-postgresql:standard-0`) แทน

### สรุป: สิ่งที่ทั้งสามแพลตฟอร์ม abstract ไปจาก Kamal

เทียบกับสิ่งที่เราต้องทำเองใน Part 076 (Kamal + VPS):

| สิ่งที่ต้องทำเองใน Kamal | Render | Fly.io | Heroku |
|---|---|---|---|
| ติดตั้ง/อัปเดต Docker บนเซิร์ฟเวอร์ | จัดการให้ | จัดการให้ | จัดการให้ |
| ตั้งค่า reverse proxy (kamal-proxy/Traefik) | จัดการให้ | จัดการให้ | จัดการให้ |
| ขอ/ต่ออายุ SSL certificate | จัดการให้ | จัดการให้ | จัดการให้ |
| ตั้งค่า health check ก่อนสลับ traffic | มีให้ผ่าน `healthCheckPath` | มีให้ผ่าน `[[http_service.checks]]` | มีให้ (ตรวจสอบพื้นฐาน) |
| Provision เซิร์ฟเวอร์ใหม่เมื่อต้อง scale | คลิก/แก้ plan | `flyctl scale` | เลื่อน dyno slider |
| **รัน `db:migrate` ให้ปลอดภัยกับ production traffic** | **ไม่จัดการให้ — ยังเป็นหน้าที่เรา** | **ไม่จัดการให้** | **ไม่จัดการให้** |

บรรทัดสุดท้ายคือประเด็นสำคัญที่สุดของ Part นี้: **ไม่มี PaaS เจ้าไหนป้องกัน migration
อันตรายให้เรา** พวกเขาแค่รันคำสั่งที่เราสั่งไว้ตามลำดับเวลาที่กำหนด ส่วนความปลอดภัยของ
migration นั้นๆ ยังเป็นความรับผิดชอบของนักพัฒนา 100% — ซึ่งเป็นเนื้อหาหลักของ Step
765 เป็นต้นไป

---

## Step 764: Environment Variables และ Secrets บน Managed Platform

ใน Part 074 เราเรียนเรื่อง Rails credentials (`config/credentials.yml.enc` + `RAILS_MASTER_KEY`)
และ environment variable บน VPS ที่เราควบคุมเอง หลักการเดียวกันใช้ได้กับ PaaS แต่วิธีการตั้งค่า
ต่างออกไป เพราะเราไม่มีสิทธิ์ SSH เข้าเซิร์ฟเวอร์เพื่อสร้างไฟล์ `.env` เอง

### หลักการสำคัญ: ไม่ commit secret ลง repo เด็ดขาด

ไม่ว่าจะเป็น Render, Fly.io หรือ Heroku กฎทองเหมือนกันหมด:

- `RAILS_MASTER_KEY`, API key ของบริการภายนอก (Stripe, Sentry, AWS), database password —
  **ตั้งค่าผ่าน dashboard เว็บหรือ CLI ของแพลตฟอร์มเท่านั้น ไม่ใช่ commit ลง `render.yaml`,
  `fly.toml`, หรือ `Procfile` โดยตรง**
- สังเกตในตัวอย่าง `render.yaml` ของ Step 762 บรรทัด:

  ```yaml
  - key: RAILS_MASTER_KEY
    sync: false
  ```

  `sync: false` บอก Render ว่า "ตัวแปรนี้มีอยู่จริง แต่ค่าของมันต้องไปตั้งเองใน dashboard"
  ไฟล์ `render.yaml` เองไม่มีค่าจริงอยู่เลย ปลอดภัยที่จะ commit ขึ้น git

### วิธีตั้งค่าในแต่ละแพลตฟอร์ม

**Render** — ผ่าน Dashboard: Service → Environment → Add Environment Variable หรือผ่าน CLI:

```bash
render env set RAILS_MASTER_KEY=abc123... --service myapp-web
```

**Fly.io** — ผ่าน `flyctl secrets` (ตั้งใจแยกจาก `[env]` ธรรมดาใน `fly.toml` ซึ่งเก็บได้แค่ค่าที่
ไม่ sensitive):

```bash
fly secrets set RAILS_MASTER_KEY=abc123...
fly secrets set STRIPE_SECRET_KEY=sk_live_...
fly secrets list   # ดูรายชื่อ (ไม่โชว์ค่าจริง)
```

**Heroku** — ผ่าน `heroku config`:

```bash
heroku config:set RAILS_MASTER_KEY=abc123...
heroku config:set STRIPE_SECRET_KEY=sk_live_...
heroku config   # ดูค่าทั้งหมด (Heroku โชว์ค่าจริงได้เพราะเป็น private ต่อ account)
```

### DATABASE_URL ถูกฉีดให้อัตโนมัติ

ทั้งสามแพลตฟอร์มมีจุดร่วมที่ดี: เมื่อผูก database service เข้ากับ web service แล้ว
ตัวแปร `DATABASE_URL` จะถูกฉีดเข้า environment ให้อัตโนมัติโดยไม่ต้องตั้งเอง (ดู
`fromDatabase` ใน `render.yaml`, หรือ Heroku ที่ผูก add-on ให้อัตโนมัติ) Rails อ่านค่านี้ผ่าน
`config/database.yml` ที่มีบรรทัด:

```yaml
production:
  primary:
    <<: *default
    url: <%= ENV["DATABASE_URL"] %>
```

(Rails 8 generate ให้แบบนี้อยู่แล้วถ้าเลือก PostgreSQL ตอน `rails new` — ดู Step 021)

> **ข้อควรระวัง:** อย่าใช้ `RAILS_MASTER_KEY` ตัวเดียวกันระหว่าง staging กับ production ถ้า
> secret รั่วจาก environment หนึ่ง อีก environment จะไม่ได้รับผลกระทบ — หลักการ "separate
> secrets per environment" นี้สำคัญไม่แพ้เรื่องไม่ commit ลง git

---

## Step 765: Release/Build Hook — จุดที่ `db:migrate` รันอัตโนมัติ และอันตรายที่แฝงอยู่

นี่คือจุดเปลี่ยนของ Part นี้ — จากเรื่อง "ตั้งค่า PaaS ยังไง" ไปสู่ "ทำไมการตั้งค่าที่ดูสมเหตุสมผล
ที่สุดถึงอันตรายได้"

### รูปแบบที่พบเห็นบ่อยที่สุด (และดูเหมือนถูกต้อง)

ทั้ง Render (`buildCommand`), Fly.io (`release_command`), Heroku (`release:` ใน Procfile)
สนับสนุนแพทเทิร์นเดียวกัน: **รัน `rails db:migrate` อัตโนมัติเป็นส่วนหนึ่งของทุก deploy** เหตุผล
ที่แพทเทิร์นนี้ได้รับความนิยม:

- ไม่มีขั้นตอนแยกที่นักพัฒนาต้องจำไปรันเอง (ลด human error จากการ "ลืม migrate")
- schema กับโค้ดจะ sync กันเสมอ เพราะรันคู่กันทุกครั้ง
- เหมาะกับทีมเล็กที่ deploy บ่อยและไม่มีกระบวนการ review พิเศษ

### แต่ปัญหาคือ...

**migration ที่รันอัตโนมัตินี้รันกับ production database ที่มี traffic จริงอยู่ ณ ขณะนั้น**
ไม่ใช่ database ว่างเปล่าแบบตอนพัฒนา หรือแม้แต่ staging ที่ traffic น้อยกว่ามาก ความแตกต่าง
สำคัญมี 3 ข้อ:

1. **มีข้อมูลจำนวนมาก** — migration ที่รันไม่ถึงวินาทีบนเครื่อง dev (ที่มีข้อมูล 10 แถว) อาจใช้เวลา
   หลายนาทีหรือค้างตลอดกาลบน production ที่มีข้อมูลหลักล้านแถว
2. **มี connection อื่นถือ lock อยู่พร้อมกัน** — request ที่กำลังประมวลผลอยู่ตอนนั้นอาจถือ lock
   บน table เดียวกับที่ migration ต้องการแก้ — เกิดการ "รอคิว" ที่ลุกลามได้ (Step 766 จะสาธิต
   ให้เห็นจริง)
3. **มีโค้ดเวอร์ชันเก่ายังรันอยู่ระหว่าง deploy** — ถ้าใช้ rolling deploy (`numInstances: 2` หรือ
   `strategy = "rolling"`) จะมีช่วงเวลาที่ instance เก่า (โค้ดเก่า) กับ instance ใหม่ (โค้ดใหม่)
   รันพร้อมกัน ถ้า migration เปลี่ยนโครงสร้างตารางแบบที่โค้ดเก่าอ่านไม่ได้ (เช่น rename column)
   instance เก่าจะพังทันทีก่อนที่ deploy จะเสร็จด้วยซ้ำ (Step 768 จะสาธิตให้เห็นจริง)

### ทำไม "ไม่ต้องกลัว เพราะรัน migrate ก่อน boot app เสมอ" ไม่ใช่คำตอบที่สมบูรณ์

หลายคนคิดว่าปัญหาข้อ 3 แก้ได้ด้วยการที่ release phase รันก่อนที่ instance ใหม่จะ boot เสมอ —
จริงส่วนหนึ่ง แต่ไม่ครอบคลุม เพราะ:

- **instance เก่ายังไม่ถูก terminate ทันที** ในทุกแพลตฟอร์มที่รองรับ zero-downtime rolling
  deploy — instance เก่ายังรับ traffic ต่อจนกว่า instance ใหม่จะผ่าน health check ระหว่างนั้น
  instance เก่าใช้โค้ดเก่าคุยกับ schema ใหม่ที่ถูก migrate ไปแล้ว
- แม้ไม่ใช้ rolling deploy (deploy แบบ replace-ทันที) **request ที่ค้างอยู่ (in-flight request)**
  ตอน deploy อาจยังประมวลผลด้วยโค้ดเก่าขณะที่ schema เปลี่ยนไปแล้ว

สรุปคือ: **ระยะเวลาสั้นๆ ที่โค้ดเก่ากับ schema ใหม่อยู่ด้วยกันแทบจะหลีกเลี่ยงไม่ได้เสมอ** ไม่ว่าจะ
ใช้ PaaS เจ้าไหน — คำตอบที่ถูกต้องไม่ใช่การพยายามกำจัดช่วงเวลานี้ให้เป็นศูนย์ แต่คือการ**เขียน
migration ให้ปลอดภัยแม้มีช่วงเวลานี้อยู่** ซึ่งเป็นเนื้อหาของ Step 767–768

ต่อไปนี้เราจะพิสูจน์ทุกข้อกล่าวอ้างข้างต้นด้วยการรันจริง

---

## Step 766: Postgres Lock 101 — ทำไม `ALTER TABLE` ถึงทำให้ทั้งแอปค้างได้ (สาธิตจริง)

### ทฤษฎีสั้นๆ ที่ต้องรู้ก่อน

PostgreSQL ใช้ระบบ lock หลายระดับ คำสั่งที่เปลี่ยนโครงสร้างตาราง (`ALTER TABLE`) ส่วนใหญ่
ต้องการ **`ACCESS EXCLUSIVE` lock** ซึ่งเป็น lock ระดับสูงสุด — ขัดแย้ง (conflict) กับ lock
**ทุกชนิด** รวมถึง `ACCESS SHARE` ที่ query `SELECT` ธรรมดาต้องใช้

ที่สำคัญกว่านั้น: Postgres จัดคิว lock request แบบ **FIFO (First In, First Out)** เพื่อป้องกัน
"writer starvation" — หมายความว่าถ้า `ALTER TABLE` ไปต่อคิวรอ lock อยู่ก่อน (เพราะมี
transaction อื่นถือ lock อยู่) **query ใดๆ ที่มาทีหลังแม้จะขอ lock ชนิดที่ไม่ conflict กับ
transaction เดิมเลยก็ตาม ก็จะต้องรอต่อคิวอยู่ดี** เพราะมันมาทีหลัง `ALTER TABLE` ในคิว

นี่คือกลไกที่ทำให้ "migration ตัวเดียวที่ดูไม่มีพิษภัย" ทำทั้งแอปค้างได้ทั้งระบบ ไม่ใช่แค่ query
ที่เกี่ยวข้องโดยตรง

### สาธิตจริง: เตรียมข้อมูล

สร้างแอป Rails ใหม่พร้อม PostgreSQL แล้ว seed ข้อมูลจริง 200,000 แถว:

```bash
rails new migration_demo --database=postgresql
cd migration_demo
bin/rails generate model User name:string email:string
bin/rails db:create db:migrate
```

```ruby
# seed 200,000 แถวด้วย insert_all เป็น batch (เร็วกว่าสร้างทีละ record มาก)
n = 200_000
batch = []
n.times do |i|
  batch << { name: "User #{i}", email: "user#{i}@example.com",
             created_at: Time.current, updated_at: Time.current }
  if batch.size == 5_000
    User.insert_all(batch)
    batch = []
  end
end
User.insert_all(batch) if batch.any?
```

```
Total users: 200000
```

### การทดลอง: จำลอง 3 session พร้อมกัน

เราจะเปิด 3 การเชื่อมต่อ psql พร้อมกัน จำลองสถานการณ์จริงบน production:

- **Session 1** — จำลอง request ที่กำลังประมวลผลอยู่ (เช่น user คนหนึ่งกด "save profile")
  เปิด transaction ค้างไว้ 8 วินาที
- **Session 2** — จำลอง migration ที่ deploy hook สั่งรัน: `ALTER TABLE users ADD COLUMN age
  integer;`
- **Session 3** — จำลอง request อื่นที่มาทีหลังเล็กน้อย แค่ `SELECT` ธรรมดาที่ดูเหมือนไม่
  เกี่ยวข้องอะไรกับ session 1 เลย

```bash
# Session 1: เปิด transaction ค้างไว้ 8 วินาที (จำลอง request ที่กำลังทำงานอยู่)
psql -d migration_demo_development <<'SQL'
BEGIN;
UPDATE users SET name = name WHERE id = 1;
SELECT pg_sleep(8);
COMMIT;
SQL

# Session 2 (เริ่มหลัง session 1 ไป 1 วินาที): migration ที่ deploy hook สั่งรัน
time psql -d migration_demo_development -c "ALTER TABLE users ADD COLUMN age integer;"

# Session 3 (เริ่มหลัง session 2 ไปอีก 1 วินาที): SELECT ธรรมดา ไม่เกี่ยวกับ session 1 โดยตรง
time psql -d migration_demo_development -c "SELECT id, name FROM users WHERE id = 2;"
```

**ผลลัพธ์จริงที่ capture ได้:**

```
=== Session 1 (ถือ lock 8 วินาที) ===
BEGIN
UPDATE 1
 pg_sleep
----------

(1 row)
COMMIT

=== Session 2: ALTER TABLE (deploy hook) ===
ALTER TABLE

real    0m7.059s   <-- รอคิวเกือบ 7 วินาทีก่อนได้ lock

=== Session 3: SELECT ธรรมดา (ไม่เกี่ยวกับ session 1 เลย) ===
 id |  name
----+--------
  2 | User 1
(1 row)

real    0m6.053s   <-- ค้างไป 6 วินาทีทั้งที่เป็นแค่ SELECT!
```

### วิเคราะห์ผลลัพธ์

- Session 2 (`ALTER TABLE`) ต้องรอ session 1 commit ก่อนถึงจะได้ `ACCESS EXCLUSIVE` lock
  ตามคาด (7 วินาที ใกล้เคียงกับเวลาที่ session 1 sleep คือ 8 วินาที)
- **สิ่งที่น่ากลัวคือ session 3** — เป็นแค่ `SELECT` ธรรมดาที่ปกติควรใช้เวลาต่ำกว่ามิลลิวินาที
  และไม่มีอะไรเกี่ยวข้องกับสิ่งที่ session 1 ทำเลย แต่ต้องรอ **6 วินาที** เพราะมันมาต่อคิวอยู่
  หลัง session 2 ที่กำลังรอ `ACCESS EXCLUSIVE` lock อยู่ก่อน — นี่คือ "lock queue cascade"
  ที่ทำให้ migration เล็กๆ ตัวเดียวสามารถทำให้ **ทุก request ที่แตะตาราง `users`** ค้างพร้อมกัน
  ทั้งหมด ไม่ใช่แค่ request ที่ชนกับ transaction ต้นตอโดยตรง

นี่คือคำอธิบายที่แท้จริงว่าทำไม "deploy migration ตอนเที่ยงคืนที่ traffic น้อย" ยังไม่พอที่จะ
ปลอดภัย 100% — ถ้ามี query แม้เพียงตัวเดียวที่ถือ lock บนตารางเดียวกันอยู่ตอนนั้น (เช่น
background job ที่ยังรันอยู่, analytics query ที่ลืมปิด) migration ก็สามารถกลายเป็นจุดเริ่มต้น
ของการค้างทั้งระบบได้

### ทางแก้: `lock_timeout` — ทำให้ migration ล้มเหลวเร็วแทนที่จะค้างไม่รู้จบ

แทนที่จะปล่อยให้ `ALTER TABLE` รอคิวนานเท่าไหร่ก็ได้ เราตั้งค่า `lock_timeout` ให้ Postgres
ยกเลิกคำสั่งเองถ้ารอ lock นานเกินกำหนด:

```bash
# Session 1 เหมือนเดิม: ถือ lock ไว้ 8 วินาที
psql -d migration_demo_development <<'SQL'
BEGIN;
UPDATE users SET name = name WHERE id = 1;
SELECT pg_sleep(8);
COMMIT;
SQL

# Session 2: ตั้ง lock_timeout ไว้ 2 วินาทีก่อนรัน ALTER TABLE
time psql -d migration_demo_development \
  -c "SET lock_timeout = '2s'; ALTER TABLE users ADD COLUMN shoe_size integer;"
```

**ผลลัพธ์จริง:**

```
SET
ERROR:  canceling statement due to lock timeout

real    0m2.048s
```

migration ล้มเหลว "เร็วและชัดเจน" ภายใน 2 วินาทีแทนที่จะค้างเงียบๆ ไม่รู้จบ (และลาก session อื่น
ไปด้วย) — deploy pipeline เห็น exit code ที่ไม่ใช่ 0 ทันที สามารถ abort deploy และแจ้งเตือนทีม
ได้เร็ว แทนที่จะปล่อยให้เว็บทั้งเว็บค้างแล้วค่อยมีคนสังเกตเห็นทีหลัง

ใน Rails ตั้งค่านี้ได้ที่ระดับ migration หรือระดับ connection:

```ruby
# ตั้งใน migration โดยตรง
class AddAgeToUsers < ActiveRecord::Migration[8.1]
  def change
    safety_assured do
      execute "SET lock_timeout = '5s'"
      add_column :users, :age, :integer
    end
  end
end
```

หรือตั้งเป็นค่า default ของทุก migration ผ่าน `config/database.yml`:

```yaml
production:
  primary:
    <<: *default
    url: <%= ENV["DATABASE_URL"] %>
    variables:
      lock_timeout: 10s
      statement_timeout: 3600s   # กันคำสั่งที่ทำงานปกติแต่ query ช้าเกินไปด้วย
```

(gem `strong_migrations` ที่จะแนะนำใน Step 769 ตั้งค่าเหล่านี้ให้อัตโนมัติผ่าน
`StrongMigrations.lock_timeout` และ `StrongMigrations.statement_timeout`)

---

## Step 767: Safe vs Dangerous Migration Patterns (สาธิตจริงทั้งสองแบบ)

ตอนนี้เราเข้าใจกลไก lock แล้ว มาดูกันว่า migration แบบไหน "ปลอดภัย" และแบบไหน "อันตราย"
พร้อมพิสูจน์ด้วยการรันจริงทุกกรณี

### ปลอดภัย: เพิ่มคอลัมน์ธรรมดา (nullable, ไม่มี default หรือมี default เป็นค่าคงที่)

ตั้งแต่ PostgreSQL 11 เป็นต้นมา การเพิ่มคอลัมน์ที่มี **default เป็นค่าคงที่ (constant)** ไม่ทำให้
Postgres ต้อง rewrite ตารางทั้งตารางอีกต่อไป (Postgres เก็บค่า default ไว้ใน metadata แล้ว
"เติม" ค่านี้ตอนอ่านแถวเก่า แทนที่จะเขียนทับทุกแถวจริง) ทำให้เป็นปฏิบัติการที่เร็วมากแม้ตารางจะมี
ข้อมูลเป็นล้านแถว

```ruby
class AddAgeToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :age, :integer, null: false, default: 0
  end
end
```

**ทดสอบจริงบนตาราง 200,000 แถว:**

```
== AddAgeToUsers: migrating ====================================
-- add_column(:users, :age, :integer, {:null=>false, :default=>0})
   -> 0.0045s
== AddAgeToUsers: migrated (0.0300s) ===========================
```

**เร็วระดับมิลลิวินาที** แม้จะมี `null: false` และ default พร้อมกัน เพราะ default เป็นค่าคงที่
(`0`) — Postgres ไม่ต้อง scan ตารางเลย

### อันตราย #1: เพิ่มคอลัมน์ `NOT NULL` โดยไม่มี default บนตารางที่มีข้อมูลอยู่แล้ว

```ruby
class AddAgeNoDefaultToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :age, :integer, null: false   # ไม่มี default!
  end
end
```

**ผลลัพธ์จริงเมื่อรันบนตารางที่มีข้อมูล 200,000 แถวอยู่แล้ว:**

```
bin/rails aborted!
StandardError: An error has occurred, this and all later migrations canceled:

PG::NotNullViolation: ERROR:  column "age" of relation "users" contains null values
```

**ทำไมถึงพัง:** แถวเก่าทั้ง 200,000 แถวไม่มีค่า `age` แต่คอลัมน์ใหม่บังคับว่าห้าม `NULL` และ
ไม่มี default ให้เติม Postgres จึงปฏิเสธทันที — ข้อดีของกรณีนี้คือ**มันพังทันทีแบบชัดเจน** ไม่ใช่
การค้าง แต่ผลลัพธ์ในทางปฏิบัติก็ยังแย่: **deploy pipeline ทั้งก้อนล้มเหลวกลางทาง** ถ้า deploy
hook เป็น `release_command`/`buildCommand`/`release:` แบบใน Step 765 การ deploy ทั้งหมดจะ
ถูก abort — ถ้าทีมไม่เคยเจอ error นี้มาก่อนใน staging (เพราะ staging database มีข้อมูลน้อยหรือ
ว่างเปล่า) นี่จะเป็นการค้นพบครั้งแรกตอน deploy จริงบน production

### อันตราย #2: เพิ่มคอลัมน์ที่มี default เป็น volatile function

ถ้าอยากให้แถวเก่ามีค่า default ที่ไม่ใช่ค่าคงที่ (เช่น UUID สุ่มใหม่ต่อแถว) หลายคนเขียนแบบนี้โดย
ไม่รู้ว่าอันตราย:

```ruby
class AddUuidToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :uuid, :uuid, default: -> { "gen_random_uuid()" }
  end
end
```

เพราะ default ไม่ใช่ค่าคงที่ (ต้องคำนวณใหม่ทุกแถว) Postgres **บังคับ rewrite ตารางทั้งตาราง**
ภายใต้ `ACCESS EXCLUSIVE` lock ตลอดระยะเวลาที่ rewrite — บนตาราง 200,000 แถวอาจไม่รู้สึก
อะไร แต่บนตารางระดับสิบล้านแถวขึ้นไป การ rewrite อาจใช้เวลาหลายนาทีถึงหลายชั่วโมง **โดยที่
ทั้งตารางถูกล็อกอ่าน-เขียนไม่ได้เลยตลอดเวลานั้น** — เหมือนกับสถานการณ์ที่สาธิตใน Step 766
แต่ระยะเวลานานกว่ามาก

### สรุปตารางเปรียบเทียบ

| Pattern | ปลอดภัยไหม | เหตุผล |
|---|---|---|
| `add_column` โดยไม่มี default (nullable) | ปลอดภัย | ไม่ต้อง backfill, ไม่ rewrite ตาราง |
| `add_column` ที่มี default เป็นค่าคงที่ (PG 11+) | ปลอดภัย | Postgres ไม่ rewrite ตาราง เก็บ default ใน metadata |
| `add_column ... null: false` ไม่มี default บนตารางมีข้อมูล | **อันตราย** | พังทันทีด้วย `NotNullViolation` — deploy ล้มเหลวกลางทาง |
| `add_column` ที่มี default เป็น volatile function/expression | **อันตราย** | บังคับ rewrite ทั้งตารางภายใต้ `ACCESS EXCLUSIVE` lock |
| `change_column_null` (เพิ่ม NOT NULL ให้คอลัมน์ที่มีอยู่แล้ว) | **อันตราย** (ถ้าทำตรงๆ) | ต้อง scan ทั้งตารางภายใต้ `ACCESS EXCLUSIVE` lock |
| `add_index` โดยไม่ใส่ `algorithm: :concurrently` | **อันตราย** | ล็อกการเขียน (write) ทั้งตารางตลอดการสร้าง index |
| `rename_column` / `remove_column` แบบตรงๆ | **อันตราย** | โค้ดเก่าที่ยังรันอยู่จะพังทันที (ดู Step 768) |

Step ถัดไปจะแก้ปัญหา `NOT NULL` และ `rename/remove column` ด้วยแพทเทิร์นแบบ
หลายขั้นตอนที่ปลอดภัยจริง — พิสูจน์ด้วยการรันจริงอีกเช่นกัน

---

## Step 768: Multi-step Safe Pattern สำหรับ Rename/Remove Column (Expand-Contract)

แนวคิดนี้เรียกว่า **Expand-Contract Pattern** (บางที่เรียก "parallel change") หลักการคือ: **ห้าม
เปลี่ยนแปลงที่ทำให้โค้ดเวอร์ชันเก่าอ่าน/เขียนข้อมูลผิดพลาดในทันที** ทุกการเปลี่ยนแปลงโครงสร้าง
ที่ "breaking" ต้องถูกแบ่งเป็นหลายขั้นตอนคร่อมหลาย deploy แทนที่จะทำในการ deploy เดียว

### ทำไมการ rename ตรงๆ ถึงอันตราย — พิสูจน์ด้วยการรันจริง

สมมติเรามีคอลัมน์ `name` และต้องการเปลี่ยนชื่อเป็น `full_name` วิธี "ไร้เดียงสา" คือ:

```ruby
class RenameUsersName < ActiveRecord::Migration[8.1]
  def change
    rename_column :users, :name, :full_name
  end
end
```

เราจำลองสถานการณ์ deploy จริง: รัน migration นี้ตรงๆ ด้วย SQL แล้วดูว่าโค้ด Rails ที่ยังอ้างอิง
`user.name` (โค้ดเวอร์ชันเก่าที่ยังไม่ได้อัปเดตเป็น `full_name`) จะเกิดอะไรขึ้น:

```bash
psql -d migration_demo_development -c "ALTER TABLE users RENAME COLUMN name TO full_name;"
```

แล้วรันโค้ด Rails (เสมือน process ใหม่ที่เพิ่ง boot ขึ้นมาหลัง migration แต่ยังใช้โค้ดเก่าที่เขียน
`user.name` อยู่ — สถานการณ์นี้เกิดขึ้นจริงเสมอในช่วง rolling deploy ตามที่อธิบายไว้ใน Step 765):

```ruby
User.where(name: "User 0").first
User.order(:name).first
User.first.name
```

**ผลลัพธ์จริงที่ capture ได้ — พังทุกจุดที่แตะคอลัมน์เก่า:**

```
OLD CODE .where(name:) FAILED: ActiveRecord::StatementInvalid:
  PG::UndefinedColumn: ERROR: column users.name does not exist

OLD CODE .order(:name) FAILED: ActiveRecord::StatementInvalid:
  PG::UndefinedColumn: ERROR: column "name" does not exist

OLD CODE plain .first FAILED: NoMethodError:
  undefined method `name' for an instance of User
```

**นี่คือหลักฐานที่ชัดเจนที่สุด:** ทันทีที่ column ถูก rename ในฐานข้อมูล **โค้ดเก่าทุกจุดที่อ้างอิง
ชื่อคอลัมน์เดิมจะพังทันที** ไม่ว่าจะเป็นการ query (`where`, `order`) หรือแค่เรียก attribute
method ธรรมดา (`.name`) — ในสถานการณ์ deploy จริงที่มี 2 instance รันคู่กันชั่วคราว (Step 765)
instance ที่ยังใช้โค้ดเก่าจะโยน error นี้ให้ผู้ใช้จริงทันทีที่ migration รันเสร็จ **ก่อนที่ instance
ใหม่จะ boot เสร็จด้วยซ้ำ**

> **ข้อสังเกตที่แอบซ่อนอันตรายกว่านั้น:** ถ้า process เก่า**เคย** query ข้อมูลไปแล้วก่อน rename
> (เช่น cache `user` object ไว้ใน memory) การเรียก `.name` ซ้ำบน object เดิมอาจไม่พังทันที
> เพราะ Rails จับคู่ attribute ตามตำแหน่งคอลัมน์ที่ query มาแล้ว ไม่ใช่ query ใหม่ทุกครั้ง — แต่
> ทันทีที่ process นั้น query ใหม่ (`where`, `order`, หรือแม้แต่ `reload`) จะพังทันที ความไม่
> แน่นอนนี้ทำให้บั๊กประเภทนี้ **debug ยากกว่าที่คิด** เพราะพังแบบสุ่ม (intermittent) ไม่ใช่พังทันที
> ทุกครั้งเหมือนที่คาดไว้

### แพทเทิร์นที่ปลอดภัย: Expand-Contract ข้ามหลาย deploy

แทนที่จะ rename ในขั้นตอนเดียว แบ่งเป็น deploy แยกกันดังนี้:

**Deploy 1 — Expand: เพิ่มคอลัมน์ใหม่ + เขียนสองที่พร้อมกัน (dual write)**

```ruby
# db/migrate/..._add_full_name_to_users.rb
class AddFullNameToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :full_name, :string   # nullable, ปลอดภัย
  end
end
```

```ruby
# app/models/user.rb — เขียนทั้งสองคอลัมน์พร้อมกันทุกครั้งที่ save
class User < ApplicationRecord
  before_save :sync_full_name

  private

  def sync_full_name
    self.full_name = name if name_changed?
  end
end
```

ตอนนี้ทั้งโค้ดเก่า (อ่าน/เขียน `name`) และโค้ดใหม่ (จะอ่าน `full_name` ในอนาคต) ทำงานได้พร้อมกัน
โดยไม่พังเลย เพราะคอลัมน์เก่ายังอยู่ครบ

**Deploy 2 — Backfill: เติมข้อมูลแถวเก่าที่ยังไม่มีค่าใน `full_name`**

ต้อง backfill เป็น batch เล็กๆ ไม่ใช่ `UPDATE users SET full_name = name` ทีเดียวทั้งตาราง
(เพราะจะถือ lock บนทุกแถวที่แก้ไขตลอด transaction เดียวยาวๆ):

```ruby
# lib/tasks/backfill.rake หรือรันผ่าน rails runner ครั้งเดียว
User.where(full_name: nil).in_batches(of: 5_000) do |batch|
  batch.update_all("full_name = name")
  sleep(0.1)   # เว้นจังหวะเล็กน้อยให้ query อื่นแทรกได้ ลด contention บน production จริง
end
```

**ทดสอบจริงบน 200,000 แถว:**

```
Backfilled in 2.77s
Remaining NULLs: 0
```

การแบ่งเป็น batch ละ 5,000 แถว (40 batch) ทำให้แต่ละ transaction สั้นมาก ไม่ถือ lock ค้างนาน
เหมือนการ `UPDATE` ทีเดียวทั้งตาราง — ต่างจาก Step 766 ที่เราเห็นว่าแค่ transaction เดียวที่ถือ
lock 8 วินาทีก็ทำให้ query อื่นค้างไปด้วยแล้ว

**Deploy 3 — Switch reads: เปลี่ยนโค้ดให้อ่านจาก `full_name` แทน `name`**

```ruby
class User < ApplicationRecord
  before_save :sync_full_name

  private

  def sync_full_name
    self.full_name = name if name_changed?
  end
end
```

```erb
<%# views ที่เคยเขียน user.name เปลี่ยนเป็น user.full_name ทั้งหมด %>
<%= user.full_name %>
```

ตอนนี้แอปอ่าน `full_name` แล้ว แต่ยังคง "เขียนสองที่" ไว้ก่อน (เผื่อกรณีต้อง rollback deploy นี้
กลับไปใช้โค้ดเก่าที่ยังอ่าน `name` อยู่ — ถ้าตัด dual-write ออกตอนนี้แล้ว rollback จะทำให้
`name` ไม่ update อีกต่อไป)

**Deploy 4 — Contract: หยุดเขียน `name`, ลบ dual-write code**

```ruby
class User < ApplicationRecord
  # ลบ before_save :sync_full_name ออกแล้ว — ไม่เขียน name อีกต่อไป
end
```

**Deploy 5 — ลบคอลัมน์เก่าอย่างปลอดภัย**

```ruby
class RemoveNameFromUsers < ActiveRecord::Migration[8.1]
  def change
    safety_assured { remove_column :users, :name, :string }
  end
end
```

ก่อน deploy นี้ ต้องมั่นใจว่าไม่มีโค้ดที่ไหนอ้างอิง `name` เหลืออยู่เลย (`ignored_columns` ใน
Step 769 ช่วยป้องกัน edge case ตอน rollback ได้)

### ทำไมต้องแบ่งเป็น 5 deploy แยกกัน (ไม่ใช่ 2-3)

จุดสำคัญคือ **แต่ละ deploy ต้อง "ย้อนกลับได้" (rollback-safe) ด้วยตัวเอง** ถ้า deploy 3 (switch
reads) มีบั๊ก แล้วต้อง rollback กลับไปที่โค้ด deploy 2 — ข้อมูลใน `name` ยังครบถ้วน เพราะ
dual-write ยังทำงานอยู่ ไม่มีข้อมูลสูญหาย ถ้าเราลัดขั้นตอนรวมหลาย step เข้าด้วยกัน (เช่น รวม
deploy 3-4 เป็น deploy เดียว) การ rollback จะทำได้ยากขึ้นมากหรือทำไม่ได้เลย

---

## Step 769: `strong_migrations` — ให้ Ruby ดักจับ migration อันตรายก่อนถึง production

เขียน migration ให้ปลอดภัยเองต้องจำกฎเยอะมาก — gem **`strong_migrations`** (โดย Andrew
Kane) ช่วยดักจับ pattern อันตรายที่พบบ่อย **ก่อนที่ migration จะรันจริงบนฐานข้อมูล** ด้วยการ
ตรวจสอบ argument ที่ส่งให้แต่ละ migration method แบบ static (ไม่ต้องรอให้ query จริงพังก่อน)

### ติดตั้ง

```ruby
# Gemfile
gem "strong_migrations"
```

```bash
bundle install
bin/rails generate strong_migrations:install
```

คำสั่งนี้สร้าง `config/initializers/strong_migrations.rb`:

```ruby
# Mark existing migrations as safe
StrongMigrations.start_after = 20260928181116

# Set timeouts for migrations
StrongMigrations.lock_timeout = 10.seconds
StrongMigrations.statement_timeout = 1.hour

# Analyze tables after indexes are added
StrongMigrations.auto_analyze = true
```

สังเกตว่า gem นี้ตั้ง `lock_timeout` และ `statement_timeout` ให้อัตโนมัติ (ตามที่อธิบายไว้ใน
Step 766) — `start_after` บอกว่า migration ที่เขียนไว้ก่อนติดตั้ง gem ถือว่าปลอดภัยแล้ว (รันไปแล้ว
จริงบน production) ไม่ต้องตรวจสอบซ้ำ

### ทดสอบจริง: strong_migrations ดักจับอะไรได้บ้าง (capture ผลลัพธ์จริงทุกกรณี)

**1) `add_column` ที่มี default เป็น volatile function (callable)**

```ruby
class AddUuidToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :uuid, :uuid, default: -> { "gen_random_uuid()" }
  end
end
```

```
=== Dangerous operation detected #strong_migrations ===

Strong Migrations does not support inspecting callable default values.

If the default value is volatile, add the column without a default value,
then change the default.

class AddUuidToUsers < ActiveRecord::Migration[8.1]
  def up
    add_column :users, :uuid, :uuid
    change_column_default :users, :uuid, -> { ... }
  end

  def down
    remove_column :users, :uuid
  end
end

Then backfill the existing rows in the Rails console or a separate
migration with disable_ddl_transaction!.
```

ถูกดักตั้งแต่ระดับ Ruby code **ก่อน**ที่จะยิง SQL ไปยัง Postgres เลย — ไม่มีการ query ฐานข้อมูล
จริงเกิดขึ้นก่อนจะ error (migration ล้มเหลวภายใน ~1 วินาที ไม่ใช่รอ table rewrite จริงก่อน)

**2) `add_index` โดยไม่ใส่ `algorithm: :concurrently`**

```ruby
class AddIndexToUsersEmail < ActiveRecord::Migration[8.1]
  def change
    add_index :users, :email
  end
end
```

```
=== Dangerous operation detected #strong_migrations ===

Adding an index non-concurrently blocks writes. Instead, use:

class AddIndexToUsersEmail < ActiveRecord::Migration[8.1]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end
```

**3) `change_column_null` (เพิ่ม `NOT NULL` ให้คอลัมน์ที่มีข้อมูลอยู่แล้ว)**

```ruby
class SetEmailNotNull < ActiveRecord::Migration[8.1]
  def change
    change_column_null :users, :email, false
  end
end
```

```
=== Dangerous operation detected #strong_migrations ===

Setting NOT NULL on an existing column blocks reads and writes while
every row is checked. Instead, add a check constraint and validate it
in a separate migration.

class SetEmailNotNull < ActiveRecord::Migration[8.1]
  def change
    add_check_constraint :users, "email IS NOT NULL",
      name: "users_email_null", validate: false
  end
end

class ValidateSetEmailNotNull < ActiveRecord::Migration[8.1]
  def up
    validate_check_constraint :users, name: "users_email_null"
    change_column_null :users, :email, false
    remove_check_constraint :users, name: "users_email_null"
  end

  def down
    add_check_constraint :users, "email IS NOT NULL",
      name: "users_email_null", validate: false
    change_column_null :users, :email, true
  end
end
```

เราทดสอบแพทเทิร์นที่ gem แนะนำนี้จริงบนตาราง 200,000 แถว (ต่อจากที่ backfill คอลัมน์ `age`
เสร็จแล้วใน Step 768):

```bash
psql -c "ALTER TABLE users ADD CONSTRAINT users_age_null CHECK (age IS NOT NULL) NOT VALID;"
psql -c "ALTER TABLE users VALIDATE CONSTRAINT users_age_null;"
psql -c "ALTER TABLE users ALTER COLUMN age SET NOT NULL;"
psql -c "ALTER TABLE users DROP CONSTRAINT users_age_null;"
```

**ผลลัพธ์เวลาจริงของแต่ละคำสั่ง:**

```
ALTER TABLE ... ADD CONSTRAINT ... NOT VALID     Time: 2.139 ms
ALTER TABLE ... VALIDATE CONSTRAINT              Time: 20.249 ms
ALTER TABLE ... SET NOT NULL                     Time: 0.655 ms
ALTER TABLE ... DROP CONSTRAINT                  Time: 0.762 ms
```

**เหตุผลที่แพทเทิร์นนี้ปลอดภัยกว่า `change_column_null` ตรงๆ:**

- `ADD CONSTRAINT ... NOT VALID` — เพิ่ม constraint แบบไม่ตรวจสอบข้อมูลเก่าทันที (เร็วมาก
  metadata-only)
- `VALIDATE CONSTRAINT` — ขั้นตอนที่ต้อง scan ทั้งตารางจริง แต่ใช้ **`SHARE UPDATE
  EXCLUSIVE`** lock เท่านั้น (ไม่ใช่ `ACCESS EXCLUSIVE`) ซึ่ง**ไม่ conflict กับ SELECT หรือ
  UPDATE ปกติ** — เราทดสอบจริงโดยรัน `SELECT` และ `UPDATE` บนตารางพร้อมกันขณะ
  `VALIDATE CONSTRAINT` กำลังทำงาน ทั้งสองคำสั่งเสร็จทันที**ไม่ถูกบล็อกเลย**
  ต่างจากสถานการณ์ใน Step 766 ที่ `SELECT` ธรรมดาถูกบล็อก 6 วินาที
- `SET NOT NULL` — เมื่อมี valid CHECK constraint ที่รับประกันว่าไม่มีค่า `NULL` อยู่แล้ว
  PostgreSQL 12+ ฉลาดพอที่จะ**ข้ามการ scan ตารางซ้ำ** ทำให้ขั้นตอนนี้เป็น metadata-only
  ด้วยเช่นกัน (เห็นได้จากเวลาแค่ 0.655ms)

**4) `rename_column` — ตรงกับที่พิสูจน์ไว้ใน Step 768 เป๊ะ**

```
=== Dangerous operation detected #strong_migrations ===

Renaming a column that's in use will cause errors in your application.
A safer approach is to:

1. Create a new column
2. Write to both columns
3. Backfill data from the old column to the new column
4. Move reads from the old column to the new column
5. Stop writing to the old column
6. Drop the old column
```

สังเกตว่าข้อความนี้คือ Expand-Contract pattern ตัวเดียวกับที่เราสาธิตไปแล้วใน Step 768 ทุก
ขั้นตอน — gem บอกวิธีแก้ที่ถูกต้องให้ตรงๆ ในข้อความ error เลย

**5) `remove_column` — ดักปัญหา attribute caching ของ Active Record**

```
=== Dangerous operation detected #strong_migrations ===

Active Record caches attributes, which causes problems when removing
columns. Be sure to ignore the column:

class User < ApplicationRecord
  self.ignored_columns += ["email"]
end

Deploy the code, then wrap this step in a safety_assured { ... } block.
```

`ignored_columns` บอก Active Record ว่า "อย่าสร้าง attribute method ให้คอลัมน์นี้" ทำให้
schema cache ที่ boot ไว้ตอนแอปเริ่มทำงานไม่ error แม้คอลัมน์จะหายไประหว่างที่แอปยังรันอยู่
(เทียบเท่าขั้นตอน "Deploy code แจ้งว่าจะไม่ใช้คอลัมน์นี้แล้ว" ก่อนค่อยลบจริงในภายหลัง)

### `safety_assured` — บอก gem ว่า "ฉันรู้ว่าทำอะไรอยู่"

บางครั้งเรารู้ดีว่า operation หนึ่งปลอดภัยจริง (เช่น ตารางว่างเปล่า หรือเป็น table ใหม่ที่เพิ่ง
สร้างใน migration เดียวกัน) ใช้ `safety_assured` เพื่อข้าม check ของ gem เฉพาะจุด:

```ruby
class RemoveNameFromUsers < ActiveRecord::Migration[8.1]
  def change
    safety_assured { remove_column :users, :name, :string }
  end
end
```

> **คำเตือน:** `safety_assured` ไม่ได้ทำให้ operation ปลอดภัยขึ้นจริง มันแค่ปิดปาก warning —
> ใช้เฉพาะตอนที่ทำตามขั้นตอน multi-step (Step 768) มาครบแล้วเท่านั้น การใช้
> `safety_assured` พร่ำเพรื่อเพื่อให้ migration ผ่านเร็วๆ คือการทำลายจุดประสงค์ของ gem นี้ทั้งหมด

---

## Step 770: Pre-deploy Checklist และแบบฝึกหัดรวบยอด

### Checklist ก่อน deploy migration ขึ้น production

ใช้ checklist นี้ทุกครั้งก่อนกด merge PR ที่มี migration ไปที่ branch ที่ auto-deploy ขึ้น
production:

**1. ตรวจสอบ migration ด้วย `strong_migrations` ในเครื่อง dev/CI ก่อน**

```bash
RAILS_ENV=test bin/rails db:migrate
```

ถ้า gem ตรวจพบ pattern อันตราย จะ error ทันทีในขั้นตอนนี้ (ควรเป็นส่วนหนึ่งของ CI pipeline
ใน Part 075 — เพิ่ม step `bundle exec rails db:migrate` ใน GitHub Actions workflow ถ้ายัง
ไม่มี)

**2. ตรวจสอบขนาดตารางจริงบน production ก่อนตัดสินใจว่าต้อง backfill แบบ batch หรือไม่**

```sql
SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = 'users';
```

ตารางที่มีมากกว่าหลักหมื่นแถวขึ้นไป ควร treat migration ที่แก้ไขข้อมูลทุกแถวเป็นเรื่องที่ต้อง
ระวังเสมอ

**3. ยืนยันว่ามี backup ล่าสุดและทดสอบ restore ได้จริง**

Managed PaaS ส่วนใหญ่ทำ automated backup ให้ (เช่น Render Postgres มี point-in-time
recovery ในแพลนที่สูงกว่า starter) แต่ **การมี backup อัตโนมัติไม่เท่ากับการรู้ว่า restore ได้จริง**
— ควรทดสอบ restore ไปยัง database ทดสอบอย่างน้อยไตรมาสละครั้ง

```bash
# ตัวอย่างการ backup/restore ด้วยตัวเอง (ใช้ได้กับทุกแพลตฟอร์มที่ให้ DATABASE_URL)
pg_dump "$DATABASE_URL" -Fc -f backup_before_migration.dump
# ทดสอบ restore ไปยัง database ว่างเปล่าอีกตัว
pg_restore -d "$TEST_RESTORE_DATABASE_URL" backup_before_migration.dump
```

**4. มีแผน rollback ที่ชัดเจนก่อน deploy เสมอ**

- migration นี้มี `down` method ที่ใช้งานได้จริงหรือไม่ (ทดสอบ `bin/rails db:rollback` ใน
  staging ก่อน)
- ถ้าเป็น migration แบบ Expand-Contract (Step 768) — deploy นี้อยู่ขั้นตอนไหนของ pattern
  และ rollback กลับไปขั้นก่อนหน้าจะทำให้ข้อมูลเสียหายหรือไม่
- ทีมรู้หรือไม่ว่าถ้า deploy พังกลางทาง ต้องกด rollback ที่ไหน (Render/Fly.io/Heroku
  dashboard, หรือ `git revert` + push ใหม่)

**5. ตรวจสอบว่า deploy hook ตั้ง `lock_timeout` ไว้แล้ว (Step 766, 769)**

ป้องกัน migration ที่ไม่คาดคิดว่าจะเจอ lock contention ไปค้างจน deploy timeout แบบเงียบๆ

**6. Review migration กับเพื่อนร่วมทีมอย่างน้อย 1 คนก่อน merge**

โดยเฉพาะ migration ที่แตะตารางที่มี traffic สูง (`users`, `orders`, ตารางหลักของระบบ) —
checklist ข้อนี้สำคัญไม่แพ้ automated tooling เพราะ `strong_migrations` ดักได้แค่ pattern ที่
รู้จัก ไม่ใช่ business logic ที่ผิดพลาด

---

## แบบฝึกหัด: สาธิต Dangerous Migration ล้มเหลวจริง แล้วแก้ด้วย Safe Multi-step Pattern

### โจทย์

สร้างแอป Rails ใหม่ชื่อ `shop_migration_lab` พร้อม PostgreSQL แล้วทำตามขั้นตอนนี้:

1. สร้างโมเดล `Product` (`name:string`, `price:decimal`) แล้ว seed ข้อมูล 50,000 แถว
2. ติดตั้ง `strong_migrations`
3. เขียน migration ที่เพิ่มคอลัมน์ `sku` แบบ `null: false` โดยไม่มี default แล้วพิสูจน์ว่าล้มเหลว
   จริงด้วย `PG::NotNullViolation`
4. แก้ด้วยแพทเทิร์นที่ปลอดภัย: เพิ่มคอลัมน์แบบ nullable → backfill เป็น batch → เพิ่ม
   `NOT NULL` ด้วยแพทเทิร์น check constraint (`NOT VALID` + `VALIDATE`)
5. เขียน migration ที่ `strong_migrations` ควรดักจับได้ (เช่น `add_index` โดยไม่ใส่
   `algorithm: :concurrently`) แล้วแสดงผลลัพธ์ที่ gem ดักจับให้ดู

### เฉลย

```bash
rails new shop_migration_lab --database=postgresql
cd shop_migration_lab
bin/rails generate model Product name:string price:decimal
bin/rails db:create db:migrate
```

```ruby
# seed.rb — รันผ่าน bin/rails runner seed.rb
n = 50_000
batch = []
n.times do |i|
  batch << { name: "Product #{i}", price: (i % 1000) + 1.0,
             created_at: Time.current, updated_at: Time.current }
  if batch.size == 5_000
    Product.insert_all(batch)
    batch = []
  end
end
Product.insert_all(batch) if batch.any?
puts "Seeded: #{Product.count}"
```

```ruby
# Gemfile
gem "strong_migrations"
```

```bash
bundle install
bin/rails generate strong_migrations:install
```

**ขั้นตอนที่ 3 — พิสูจน์ว่าล้มเหลวจริง:**

```ruby
# db/migrate/..._add_sku_no_default_to_products.rb
class AddSkuNoDefaultToProducts < ActiveRecord::Migration[8.1]
  def change
    add_column :products, :sku, :string, null: false
  end
end
```

```bash
bin/rails db:migrate
```

ผลลัพธ์ที่คาดหวัง (และควรได้จริงถ้าทำตามขั้นตอนถูกต้อง):

```
PG::NotNullViolation: ERROR:  column "sku" of relation "products" contains null values
```

ลบ migration นี้ทิ้ง (`rm db/migrate/..._add_sku_no_default_to_products.rb`) แล้วไปขั้นตอนที่ 4

**ขั้นตอนที่ 4 — แพทเทิร์นที่ปลอดภัย:**

```ruby
# Deploy 1: เพิ่มคอลัมน์แบบ nullable
class AddSkuToProducts < ActiveRecord::Migration[8.1]
  def change
    add_column :products, :sku, :string
  end
end
```

```ruby
# Deploy 2: backfill เป็น batch (รันผ่าน rails runner หรือ rake task แยกจาก migration)
Product.where(sku: nil).find_each(batch_size: 5_000) do |product|
  product.update_column(:sku, "SKU-#{product.id.to_s.rjust(8, '0')}")
end
puts "Remaining NULLs: #{Product.where(sku: nil).count}"
```

(หมายเหตุ: ใช้ `find_each` + `update_column` ทีละแถวในโจทย์นี้เพราะค่า `sku` ต้อง unique
ต่อแถว ถ้า backfill เป็นค่าเดียวกันทั้ง batch ได้ ให้ใช้ `in_batches` + `update_all` แบบ Step
768 จะเร็วกว่ามาก)

```ruby
# Deploy 3: เพิ่ม NOT NULL แบบปลอดภัยด้วย check constraint pattern
class AddSkuNotNullConstraint < ActiveRecord::Migration[8.1]
  def change
    add_check_constraint :products, "sku IS NOT NULL",
      name: "products_sku_null", validate: false
  end
end

class ValidateSkuNotNullConstraint < ActiveRecord::Migration[8.1]
  def up
    validate_check_constraint :products, name: "products_sku_null"
    change_column_null :products, :sku, false
    remove_check_constraint :products, name: "products_sku_null"
  end

  def down
    add_check_constraint :products, "sku IS NOT NULL",
      name: "products_sku_null", validate: false
    change_column_null :products, :sku, true
  end
end
```

รันทั้งสอง migration แล้วตรวจสอบผล:

```bash
bin/rails db:migrate
bin/rails runner 'puts ActiveRecord::Base.connection.columns(:products)
  .find { |c| c.name == "sku" }.null'
# => false
```

**ขั้นตอนที่ 5 — พิสูจน์ว่า `strong_migrations` ดักจับ `add_index` ได้:**

```ruby
class AddIndexToProductsSku < ActiveRecord::Migration[8.1]
  def change
    add_index :products, :sku
  end
end
```

```bash
bin/rails db:migrate
```

ผลลัพธ์ที่คาดหวัง:

```
=== Dangerous operation detected #strong_migrations ===

Adding an index non-concurrently blocks writes. Instead, use:

class AddIndexToProductsSku < ActiveRecord::Migration[8.1]
  disable_ddl_transaction!

  def change
    add_index :products, :sku, algorithm: :concurrently
  end
end
```

แก้ไขตามคำแนะนำแล้วรันใหม่ให้ผ่าน:

```ruby
class AddIndexToProductsSku < ActiveRecord::Migration[8.1]
  disable_ddl_transaction!

  def change
    add_index :products, :sku, algorithm: :concurrently
  end
end
```

```bash
bin/rails db:migrate
# ควรผ่านโดยไม่มี dangerous operation warning
```

**สิ่งที่แบบฝึกหัดนี้พิสูจน์ครบทุกข้อ:**

- migration `NOT NULL` ไม่มี default บนตารางที่มีข้อมูล → ล้มเหลวจริง
- แพทเทิร์น expand (nullable) → backfill → constrain (check constraint pattern) → สำเร็จ
  โดยไม่มีช่วงเวลาที่แอปพัง
- `strong_migrations` ดักจับ `add_index` แบบไม่ปลอดภัยได้จริงก่อนรันจริง

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. ทำ Expand-Contract pattern แบบเต็มรูปแบบสำหรับการ **rename** คอลัมน์ `name` ของ
   `Product` เป็น `title` ให้ครบทั้ง 5 deploy ตามที่อธิบายใน Step 768 (dual-write ผ่าน
   `before_save`, backfill, switch read, หยุด dual-write, ลบคอลัมน์เก่า) แล้วพิสูจน์ระหว่างทาง
   ว่าโค้ดที่ "แสร้งว่ายังเป็นเวอร์ชันเก่า" ยังทำงานได้ปกติในทุกขั้นตอน (ยกเว้นขั้นตอนสุดท้าย)
2. เขียน migration ที่เพิ่ม `foreign_key` จากตาราง `reviews` ไปยัง `products` บนตารางที่มี
   ข้อมูลอยู่แล้ว 50,000 แถว โดยไม่ใส่ `validate: false` แล้วดูว่า `strong_migrations` ดักจับ
   อย่างไร จากนั้นแก้ตามคำแนะนำของ gem แล้วเทียบเวลาที่ใช้ก่อน/หลังแก้
3. เขียนสคริปต์ที่จำลองสถานการณ์ Step 766 (lock queue cascade) แต่เปลี่ยนจาก `ALTER
   TABLE ADD COLUMN` เป็น `add_index` แบบไม่ `concurrently` แทน แล้ววัดว่า write query ปกติ
   (`INSERT`/`UPDATE`) บนตารางเดียวกันถูกบล็อกนานเท่าไหร่ระหว่างสร้าง index บนตาราง
   50,000 แถว เปรียบเทียบกับตอนใช้ `algorithm: :concurrently`

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Managed PaaS (Render, Fly.io, Heroku)** เป็นทางเลือกที่ trade-off "control" แลกกับ
  "ความเรียบง่าย" เทียบกับ Kamal + VPS ของ Part 076 พร้อมตารางเปรียบเทียบราคา ความง่าย
  และสิ่งที่แต่ละแพลตฟอร์ม abstract ไป
- เขียนและอ่าน `render.yaml`, `fly.toml`, `Procfile` ได้ทีละบรรทัด เข้าใจ `buildCommand`,
  `release_command`, และบรรทัด `release:` ว่าคือจุดเดียวกันในทางแนวคิด — จุดที่
  `db:migrate` รันอัตโนมัติทุกครั้งที่ deploy
- ตั้งค่า environment variable และ secret บน managed platform อย่างปลอดภัย (ผ่าน
  dashboard/CLI ไม่ commit ลง repo)
- **เข้าใจกลไก lock ของ PostgreSQL ระดับลึก** และพิสูจน์ด้วยการรันจริงว่า `ALTER TABLE`
  ตัวเดียวสามารถทำให้ `SELECT` ธรรมดาที่ไม่เกี่ยวข้องเลยค้างไปด้วยได้ (lock queue cascade)
  พร้อมทางแก้ด้วย `lock_timeout` ที่ทำให้ migration ล้มเหลวเร็วแทนที่จะค้างไม่รู้จบ
- แยกแยะ migration pattern ที่ปลอดภัย (nullable column, constant default) กับที่อันตราย
  (`NOT NULL` ไม่มี default, volatile default, `change_column_null` ตรงๆ, `add_index` ไม่
  `concurrently`) ด้วยหลักฐานจริงทุกกรณี
- เข้าใจและปฏิบัติตาม **Expand-Contract Pattern** สำหรับ rename/remove column ข้ามหลาย
  deploy พร้อมพิสูจน์ด้วยการรันจริงว่าโค้ดเก่าพังทันทีถ้า rename แบบขั้นตอนเดียว
- ติดตั้งและใช้ `strong_migrations` เพื่อดักจับ migration อันตรายอัตโนมัติก่อนถึง production
  พร้อมทดสอบจริงว่า gem นี้ดักจับอะไรได้บ้าง (volatile default, `add_index`,
  `change_column_null`, `rename_column`, `remove_column`) และแนะนำแพทเทิร์นที่ถูกต้องให้
  ตรงในข้อความ error เลย
- มี pre-deploy checklist ที่ใช้ได้จริง (ตรวจสอบขนาดตาราง, backup/restore, rollback plan,
  lock_timeout, code review) ก่อนปล่อย migration ขึ้น production ทุกครั้ง

**ต่อไป (Part 078):** เมื่อแอป deploy ขึ้น production แล้ว (ไม่ว่าจะผ่าน Kamal หรือ managed
PaaS) คำถามถัดไปคือ "รู้ได้อย่างไรว่าแอปทำงานปกติอยู่จริง" เราจะเรียนเรื่อง **Monitoring &
Logging** — structured logging (`lograge`, JSON log format), error tracking ด้วย **Sentry**
(capture exception พร้อม context, breadcrumb, release tracking), และการตั้งค่า **uptime
monitoring** เพื่อให้รู้ก่อนผู้ใช้จะบ่นว่าเว็บล่ม
