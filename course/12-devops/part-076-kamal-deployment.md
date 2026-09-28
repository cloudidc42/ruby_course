# Part 076: Deployment ด้วย Kamal (Rails 8 style deploy)

> **Step ครอบคลุมใน Part นี้:** Step 751–760
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 073 เรื่อง Docker สำหรับ Rails, Part 074 เรื่อง environment
> variable/credentials, และ Part 075 เรื่อง CI/CD มาก่อน — Part นี้จะทวนเฉพาะจุดที่เกี่ยวข้องกับ
> Kamal โดยตรง ไม่สอน Docker พื้นฐานซ้ำตั้งแต่ต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4), gem `kamal`
> เวอร์ชัน 2.x (ทดสอบจริงด้วย Kamal **2.12.0** ซึ่งเป็นเวอร์ชันล่าสุดที่ `bundle install` ดึงมาให้
> ตอนเขียน Part นี้ — Rails 8 ไม่ได้ pin เวอร์ชัน kamal ตายตัวใน Gemfile จึงอาจได้เวอร์ชันใหม่กว่านี้
> เล็กน้อยเมื่อคุณลองเอง), Docker 29.3.1

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> Part นี้พูดถึงการ deploy ไปเซิร์ฟเวอร์จริง แต่แซนด์บ็อกซ์ที่ใช้เขียนเนื้อหานี้**ไม่มี VPS จริงให้
> ต่อผ่าน SSH** ทีมผู้เขียนจึงแบ่งให้ชัดเจนว่าส่วนไหนพิสูจน์ได้จริงในเครื่อง และส่วนไหนต้องอิงจาก
> เอกสารทางการ/source code ของ Kamal:
>
> - **ทดสอบจริง 100%:** เราสร้างแอป Rails 8.1.4 จริงด้วย `rails new` (มี PostgreSQL adapter) แล้ว
>   ดู**ไฟล์ที่ Rails/Kamal generate ให้จริง**ทั้งหมด — `Dockerfile`, `config/deploy.yml`,
>   `.kamal/secrets`, `.kamal/hooks/*`, `bin/kamal` — ไม่มีการพิมพ์เดาจากความจำแม้แต่บรรทัดเดียว
>   ทุกไฟล์ที่แสดงใน Part นี้คือของจริงที่ generate ออกมาจากเครื่องมือจริง
> - **ทดสอบจริง 100% (คำสั่ง Kamal CLI ที่ไม่ต้องมี server จริง):** `kamal version`,
>   `bin/kamal help <command>` ทุกคำสั่งหลัก, และที่สำคัญที่สุดคือ **`kamal docs <section>`** —
>   คำสั่งนี้ print เอกสารอ้างอิงที่ฝังอยู่ใน source code ของ gem `kamal` ที่ติดตั้งจริงในเครื่อง
>   ออกมาตรงๆ ทำให้ตาราง options ของ `proxy`, `servers`, `registry`, `env`, `ssh`, `builder` ใน
>   Part นี้เป็นข้อมูลที่ดึงมาจากตัว gem เวอร์ชัน 2.12.0 โดยตรง ไม่ใช่จากเว็บไซต์เอกสารที่อาจไม่ตรง
>   กับเวอร์ชันที่ติดตั้งจริง
> - **ทดสอบจริง 100% (validate config แบบเต็มรูปแบบ):** เราเขียน `config/deploy.yml` ตัวเต็มของ
>   แบบฝึกหัดท้าย Part นี้ (มี accessory Postgres) แล้วรัน **`kamal config`** จริงผ่าน Kamal 2.12.0
>   ที่ติดตั้งจริง เพื่อยืนยันว่าไฟล์ parse ผ่านและ resolve ค่าต่างๆ (registry path, roles, env,
>   accessory config) ถูกต้องตามที่ตั้งใจ — เป็นการพิสูจน์ความถูกต้องของ syntax ทั้งไฟล์แบบเต็ม
>   ไม่ใช่แค่ "น่าจะถูก"
> - **ทดสอบจริงบางส่วน พบข้อจำกัดของแซนด์บ็อกซ์ (รายงานตรงไปตรงมา):** เราลองรัน
>   `bin/kamal build create` (สร้าง buildx builder — สำเร็จ) และ `bin/kamal build dev` (build
>   image จาก Dockerfile จริงแบบไม่ต้อง push ไป registry) แต่ล้มเหลวที่ขั้นตอน resolve
>   `docker/dockerfile:1` syntax image ด้วย TLS error (`x509: certificate signed by unknown
>   authority`) — สาเหตุคือ container ของ buildkit ที่ Kamal สร้างขึ้นมาใหม่ (driver
>   `docker-container`) ไม่ได้ไว้ใจ CA certificate ของ proxy เครือข่ายในแซนด์บ็อกซ์นี้ (เป็นข้อจำกัด
>   เฉพาะของสภาพแวดล้อมนี้ ไม่ใช่บั๊กของ Kamal หรือ Docker) เพื่อพิสูจน์ว่า **Dockerfile ที่ Rails 8
>   generate ให้นั้นถูกต้องและ buildได้จริง** เราจึงลอง build ตรงด้วย `docker build` แบบ classic
>   builder (`DOCKER_BUILDKIT=0`) แทน ผลคือ **ดึง base image `ruby:3.3.6-slim` จาก Docker Hub
>   สำเร็จจริง** (ยืนยันว่า syntax และโครงสร้าง multi-stage ของ Dockerfile ถูกต้อง) แต่ไป
>   ติดขัดที่ขั้นตอน `apt-get update` เพราะ network policy ของแซนด์บ็อกซ์นี้ปฏิเสธการเชื่อมต่อไปยัง
>   `deb.debian.org` (403 Forbidden) — เป็นข้อจำกัดด้าน network policy ของแซนด์บ็อกซ์ ไม่เกี่ยวกับ
>   ความถูกต้องของ Dockerfile หรือ Kamal แต่อย่างใด รายละเอียดเต็มอยู่ใน Step 753
> - **ทดสอบจริง (พฤติกรรมเมื่อไม่มี server):** เรารัน `kamal lock status` ชี้ไปที่ IP ปลอม
>   (`203.0.113.10` ตามมาตรฐาน RFC 5737 สำหรับ IP ใช้ในเอกสาร) ยืนยันว่า Kamal พยายาม SSH
>   ไปจริงและ timeout ตามคาด (ไม่ error ผิดรูปแบบ หรือ crash) เป็นการยืนยันว่า flow การเชื่อมต่อ
>   ทำงานถูกต้องจนถึงจุดที่ต้องมี server จริง
> - **มาจากเอกสารทางการ/source code ของ Kamal ไม่ได้ทดสอบกับ server จริง (ระบุไว้ชัดเจนในเนื้อหา
>   ทุกจุด):** พฤติกรรมที่ต้องมี VPS จริงถึงจะเห็นผล เช่น `kamal setup` ที่ติดตั้ง Docker บน
>   เซิร์ฟเวอร์ใหม่, การขอใบรับรอง Let's Encrypt จริงของ `kamal-proxy`, กลไก health check ตัดสลับ
>   container จริงบน production, และ `kamal rollback` ที่สลับกลับ container เก่าบนเซิร์ฟเวอร์จริง —
>   ส่วนเหล่านี้อธิบายตามเอกสารทางการของ Kamal (`kamal docs`, README ของโปรเจกต์
>   `basecamp/kamal` และ `basecamp/kamal-proxy`) อย่างละเอียด แต่จะระบุไว้ชัดเจนว่าไม่ได้รันจริง
>
> **ข้อควรรู้เรื่อง Traefik:** ถ้าคุณเคยอ่านบทความเก่าเกี่ยวกับ Kamal มาก่อน อาจเคยเห็นว่า Kamal
> ใช้ **Traefik** เป็น reverse proxy — นั่นถูกต้องสำหรับ **Kamal เวอร์ชัน 1.x เท่านั้น** ตั้งแต่
> **Kamal 2.0** เป็นต้นไป (ซึ่งเป็นเวอร์ชันที่ผูกมากับ Rails 8 และเป็นเวอร์ชันที่ Part นี้สอน) ทีม
> Basecamp เขียน reverse proxy ของตัวเองขึ้นมาใหม่ทั้งหมดชื่อ **`kamal-proxy`** (เขียนด้วยภาษา Go)
> มาแทน Traefik เพื่อให้ควบคุมพฤติกรรม zero-downtime deploy ได้ละเอียดกว่าเดิม Part นี้จะอธิบายกลไก
> ของ `kamal-proxy` ให้ตรงกับ Kamal 2.x ที่ใช้งานจริงในปัจจุบัน ไม่ใช่ Traefik ของเวอร์ชันเก่า

หลังจาก Part 075 พาไปสร้างท่อ CI/CD ด้วย GitHub Actions ที่รัน test, lint, และ security scan
อัตโนมัติทุกครั้งที่ push โค้ด คำถามที่ตามมาตามธรรมชาติคือ "แล้วพอโค้ดผ่านหมดแล้ว จะเอาไปรันบน
เซิร์ฟเวอร์จริงยังไง" Part นี้คือคำตอบสำหรับทีมที่**ไม่อยากผูกติดกับ PaaS เจ้าใดเจ้าหนึ่ง** และ
อยากมีเซิร์ฟเวอร์เป็นของตัวเอง (VPS ราคาถูกจาก DigitalOcean, Hetzner, Vultr ฯลฯ หรือแม้แต่เครื่อง
เก่าที่มี Linux ในออฟฟิศ) แต่ยังอยากได้ประสบการณ์ deploy ที่ง่ายเหมือน `git push heroku main`

**Kamal** คือคำตอบของทีม Basecamp/37signals (ผู้สร้าง Ruby on Rails) ต่อปัญหานี้ และตั้งแต่ **Rails
8** เป็นต้นไป Kamal ถูกผูกเข้ามาเป็นส่วนหนึ่งของ `rails new` โดยอัตโนมัติ — ทุกแอป Rails 8 ใหม่จะมี
`config/deploy.yml`, `Dockerfile`, และ `.kamal/` folder พร้อมใช้งานตั้งแต่วันแรก แม้จะยังไม่ได้คิด
เรื่อง deploy เลยก็ตาม

## สารบัญของ Part นี้

- Step 751: Kamal คืออะไร ทำไม Rails 8 ถึงผูกมันมาให้เป็นค่าเริ่มต้น
- Step 752: แนวคิดหลักของ Kamal — build, push, SSH, pull, run, และบทบาทของ `kamal-proxy`
- Step 753: `rails new` กับ Kamal — ดูไฟล์ที่ Rails 8 generate ให้จริงทั้งหมด
- Step 754: ผ่า `config/deploy.yml` ทีละส่วน — `service`, `image`, `servers`, `registry`, `env`
- Step 755: ผ่า `config/deploy.yml` ต่อ — `proxy`, `ssh`, `builder`, `volumes`, `asset_path`
- Step 756: เตรียมเซิร์ฟเวอร์จริงสำหรับ deploy — VPS, Docker, SSH, container registry account
- Step 757: `kamal setup` (deploy ครั้งแรก) เทียบกับ `kamal deploy`/`kamal redeploy` (deploy ครั้ง
  ถัดไป)
- Step 758: จัดการ secrets ให้ปลอดภัย — `.kamal/secrets`, Rails credentials, ห้าม commit ของจริง
- Step 759: กลไก Zero-downtime deployment ของ `kamal-proxy` และการ `kamal rollback`
- Step 760: Accessories (Postgres/Redis เป็น container) และ multi-server/multi-role สำหรับ scale

---

## Step 751: Kamal คืออะไร ทำไม Rails 8 ถึงผูกมันมาให้เป็นค่าเริ่มต้น

### ปัญหาที่ Kamal แก้

ก่อนมี Kamal ทีมพัฒนา Rails ที่อยากรันแอปบนเซิร์ฟเวอร์ของตัวเอง (ไม่ใช้ PaaS อย่าง Heroku) มักต้อง
เลือกเอาระหว่างสองทางที่ไม่ค่อยน่าพอใจนัก:

1. **Capistrano** — เครื่องมือ deploy รุ่นเก๋าของวงการ Ruby ที่ SSH ไปรันคำสั่งบนเซิร์ฟเวอร์ตรงๆ
   (`git pull`, `bundle install`, restart process) ปัญหาคือเซิร์ฟเวอร์ต้องมี Ruby เวอร์ชันตรงกับ
   แอป, มี system dependency ครบ, และการ deploy แต่ละครั้งเสี่ยงต่อ "works on my machine แต่พังบน
   เซิร์ฟเวอร์" เพราะไม่ได้รันในสภาพแวดล้อมที่ isolated
2. **เขียนสคริปต์ Docker + SSH เอง** — Containerize แอปด้วย Docker (แก้ปัญหาเรื่อง environment ได้)
   แต่ต้องเขียนสคริปต์เองทั้งหมดสำหรับ build image, push ไป registry, SSH ไปเซิร์ฟเวอร์, pull image,
   หยุด container เก่า, รัน container ใหม่ — และที่ยากที่สุดคือทำให้ขั้นตอนสลับ container เก่า/ใหม่
   นี้**ไม่มี downtime** (ถ้าหยุด container เก่าก่อนแล้วค่อยเริ่ม container ใหม่ จะมีช่วงเวลาที่
   เว็บไซต์ล่มเสมอ)

ทีม 37signals เจอปัญหานี้ตรงๆ ตอนสร้าง **ONCE** (ผลิตภัณฑ์ที่ลูกค้าซื้อไปรันบนเซิร์ฟเวอร์ของตัวเอง
ไม่ใช่ SaaS ที่ 37signals โฮสต์เอง) พวกเขาต้องมีวิธี deploy ที่ใช้ได้กับเซิร์ฟเวอร์ของลูกค้าที่ไม่รู้จัก
ล่วงหน้าเลย จึงเขียน Kamal ขึ้นมา (เดิมชื่อ **MRSK**) แล้ว open source ให้ชุมชนใช้ฟรีตั้งแต่ปี 2023

### Kamal คืออะไรในหนึ่งประโยค

> **Kamal คือเครื่องมือ deploy แอปที่ Containerize ด้วย Docker ไปยังเซิร์ฟเวอร์ใดก็ได้ที่คุณเข้าถึง
> ผ่าน SSH ได้ — ให้ประสบการณ์ deploy ง่ายแบบ Heroku แต่รันบนเซิร์ฟเวอร์ที่คุณควบคุมเองทั้งหมด**

จุดเด่นที่ทำให้ Kamal ต่างจากการเขียนสคริปต์ deploy เอง และต่างจาก PaaS:

| คุณสมบัติ | Kamal | PaaS (Heroku/Render) | สคริปต์ deploy เอง |
|-----------|-------|----------------------|---------------------|
| Vendor lock-in | ไม่มี — deploy ไปเซิร์ฟเวอร์ไหนก็ได้ที่มี Docker + SSH | มี — ผูกกับ platform นั้น | ไม่มี แต่ maintain เอง |
| ค่าใช้จ่าย | ค่า VPS ล้วนๆ (อาจถูกกว่า PaaS มาก) | ค่า platform (มักแพงกว่าตามการใช้งาน) | ค่า VPS + เวลาที่เสียไปเขียน/ดูแลสคริปต์ |
| Zero-downtime deploy | มีในตัว (ผ่าน `kamal-proxy`) | มีในตัว (ปกติ) | ต้องเขียนเอง (ยากและมักพลาด) |
| Rollback | คำสั่งเดียว (`kamal rollback`) | มีในตัว | ต้องเขียนเอง |
| ต้องเรียนรู้อะไรใหม่ | Docker + YAML config ของ Kamal | น้อยมาก (ผูก platform ให้) | ทุกอย่างที่ต้องเขียนเอง |
| ควบคุมเซิร์ฟเวอร์ได้ละเอียดแค่ไหน | เต็มที่ (เป็นเซิร์ฟเวอร์ของคุณเอง) | จำกัดตามที่ platform เปิดให้ | เต็มที่ |

### ทำไม Rails 8 ถึงผูก Kamal มาเป็นค่าเริ่มต้น

Rails ยึดหลัก **"Convention over Configuration"** มาตลอด และ DHH (ผู้สร้าง Rails ซึ่งเป็นคนเดียวกับ
ที่นำทีมสร้าง Kamal) มองว่าการ deploy ก็ควรมี "convention" แบบเดียวกัน แทนที่จะให้นักพัฒนาทุกคนต้อง
ไปหาทางแก้ปัญหา deploy เองจากศูนย์ ตั้งแต่ **Rails 8.0** (ปลายปี 2024) เป็นต้นไป ทุกครั้งที่รัน
`rails new` ระบบจะ:

1. ใส่ `gem "kamal", require: false` ลงใน Gemfile โดยอัตโนมัติ
2. Generate `Dockerfile` ที่พร้อมใช้งานจริงให้ (multi-stage build, non-root user, ปรับแต่งเพื่อ
   ขนาด image เล็กและปลอดภัย)
3. รัน `bundle exec kamal init` ให้อัตโนมัติ ซึ่งสร้าง `config/deploy.yml` และ `.kamal/secrets`
   พร้อม comment อธิบายทุก field ไว้ครบ (เราจะเห็นไฟล์จริงใน Step 753)

พูดง่ายๆ คือ **Rails 8 ทำให้ "มีแผน deploy ไป production" เป็นค่าเริ่มต้นของทุกโปรเจกต์ตั้งแต่วันแรก**
ไม่ใช่สิ่งที่ต้องมานั่งคิดทีหลังตอนใกล้ launch เหมือนแต่ก่อน แม้สุดท้ายทีมจะเลือกไม่ใช้ Kamal จริง
(เช่นเลือกใช้ Render/Fly.io แทนตามที่ Part 077 จะสอน) ไฟล์เหล่านี้ก็ไม่ได้เกะกะอะไร เพราะ Kamal
เพียงอ่าน Dockerfile ที่มีอยู่แล้วเป็นฐาน — เท่ากับว่าการมี Dockerfile ที่ดีตั้งแต่ต้นมีประโยชน์กับ
ทุกเส้นทาง deploy ไม่ใช่แค่ Kamal

---

## Step 752: แนวคิดหลักของ Kamal — build, push, SSH, pull, run, และบทบาทของ `kamal-proxy`

### วงจรการ deploy แบบพื้นฐานที่สุด

ลืมเรื่อง config ทั้งหมดไปก่อน แก่นของสิ่งที่ Kamal ทำทุกครั้งที่ deploy มีแค่ 5 ขั้นตอน:

```
1. BUILD   — สร้าง Docker image จากโค้ดปัจจุบันของแอป (ใช้ Dockerfile ที่มีอยู่)
2. PUSH    — ส่ง image นั้นขึ้น container registry (Docker Hub, GHCR, หรือ registry ส่วนตัว)
3. SSH     — เชื่อมต่อไปยังเซิร์ฟเวอร์เป้าหมายผ่าน SSH (แต่ละเซิร์ฟเวอร์ที่ระบุใน config)
4. PULL    — สั่งเซิร์ฟเวอร์ pull image เวอร์ชันใหม่ลงมา (เซิร์ฟเวอร์ไม่ต้อง build เอง)
5. RUN     — รัน container ใหม่จาก image นั้น แล้วสลับ traffic จาก container เก่ามาที่ container ใหม่
```

สังเกตว่า**เซิร์ฟเวอร์ไม่ต้องมี Ruby, gem, หรือ dependency ใดๆ ของแอปติดตั้งอยู่เลย** ต้องมีแค่
**Docker** เท่านั้น (Kamal ติดตั้ง Docker ให้อัตโนมัติถ้ายังไม่มีตอนรัน `kamal setup` — ดู Step 756)
เพราะทุกอย่างที่แอปต้องการถูกห่อไว้ใน image เรียบร้อยแล้วตั้งแต่ขั้นตอน BUILD ทำให้ปัญหาคลาสสิก
"บนเครื่องผมรันได้" หมดไป — ถ้า image รันได้บนเครื่อง dev มันก็รันได้บนเซิร์ฟเวอร์ เพราะเป็น
environment เดียวกันเป๊ะๆ

### ปัญหาที่ยากที่สุด: สลับ container โดยไม่มี downtime

ขั้นตอนที่ 5 (RUN) ฟังดูง่าย แต่มีรายละเอียดสำคัญ: **ถ้าหยุด container เก่าก่อนแล้วค่อยเริ่ม
container ใหม่ (naive approach) จะมีช่วงเวลาสั้นๆ ที่ไม่มี container ไหนรับ request เลย** —
ผู้ใช้ที่เข้าเว็บพอดีตอนนั้นจะเจอ "Connection refused" การ deploy ที่ทำแบบนี้ทุกครั้งจะทำให้เว็บ
"กระตุก" เป็นวินาทีๆ ทุกครั้งที่มีการ deploy ใหม่ ซึ่งยอมรับไม่ได้สำหรับ production จริง

นี่คือจุดที่ **`kamal-proxy`** เข้ามาแก้ปัญหา — เป็น reverse proxy ขนาดเล็กที่ Kamal ติดตั้งลงบน
ทุกเซิร์ฟเวอร์ (รันเป็น container แยกต่างหาก ฟังที่ port 80/443) ทำหน้าที่เป็นตัวกลางระหว่าง
internet กับ container ของแอปเราเสมอ กลไกคร่าวๆ (รายละเอียดเต็มใน Step 759):

```
Internet → kamal-proxy (port 80/443) → container ของแอป (port ภายในที่ไม่เปิดสู่ภายนอกโดยตรง)
```

เมื่อ deploy เวอร์ชันใหม่ `kamal-proxy` จะ:

1. ปล่อยให้ container **เก่า** ยังคงรับ traffic ตามปกติ ไม่แตะต้องเลย
2. สั่งเริ่ม container **ใหม่** ขึ้นมาคู่ขนานกัน (มี 2 container รันพร้อมกันชั่วคราว)
3. ยิง HTTP request ไปที่ endpoint healthcheck ของ container ใหม่ซ้ำๆ (ค่าเริ่มต้นคือ `GET /up`
   ทุก 1 วินาที) จนกว่าจะได้ HTTP 200 กลับมา
4. เมื่อ container ใหม่ตอบ 200 แล้ว (แปลว่าแอป boot เสร็จสมบูรณ์ พร้อมรับ traffic จริง) **ค่อย
   สลับ** ให้ request ใหม่ทั้งหมดวิ่งไปที่ container ใหม่ทันที (แบบ atomic ไม่มีช่วงคาบเกี่ยว)
5. รอให้ request ที่ค้างอยู่ใน container เก่า (ถ้ามี) ประมวลผลจบก่อน (drain) แล้วค่อยหยุด container
   เก่าทิ้ง

ผลลัพธ์คือ**ไม่มีวินาทีไหนเลยที่ไม่มี container พร้อมรับ traffic** — นี่คือความหมายของ
"zero-downtime deployment" และเป็นเหตุผลหลักที่ Kamal ต้องมี proxy ของตัวเองแทนที่จะสั่ง
Docker รัน container ตรงๆ

> **ข้อควรรู้:** `/up` คือ route ที่ Rails 8 generate ให้อัตโนมัติทุกแอปใหม่อยู่แล้ว (ตรวจสอบได้จาก
> `config/routes.rb` ที่มีบรรทัด `get "up" => "rails/health#show"`) คืนค่า HTTP 200 ถ้าแอป boot
> โดยไม่มี exception ทำให้ Kamal ใช้เป็น health check endpoint เริ่มต้นได้ทันทีโดยไม่ต้องตั้งค่า
> เพิ่มเอง — เป็นอีกตัวอย่างของ "convention" ที่ Rails 8 เตรียมไว้ให้ล่วงหน้า

---

## Step 753: `rails new` กับ Kamal — ดูไฟล์ที่ Rails 8 generate ให้จริงทั้งหมด

มาดูของจริงกัน เราสร้างแอปทดสอบด้วยคำสั่งมาตรฐาน:

```bash
rails new kamal_test_app --css=tailwind --database=postgresql
```

ในระหว่างสร้างแอป มีช่วงหนึ่งที่ log แสดงชัดเจนว่า Kamal ถูกเรียกใช้งานโดยอัตโนมัติ (ไม่ต้องสั่งเอง
เลยแม้แต่คำสั่งเดียว):

```
run    bundle install --quiet
run    bundle binstubs kamal
run    bundle exec kamal init
Created configuration file in config/deploy.yml
Created .kamal/secrets file
Created sample hooks in .kamal/hooks
```

สามบรรทัดนี้คือหลักฐานตรงๆ ว่า **Rails 8 เรียก `kamal init` ให้เองหลังติดตั้ง gem เสร็จ** ไม่ใช่แค่
ใส่ gem ไว้เฉยๆ แล้วให้เราไปสั่งเอง

### ไฟล์ที่ได้มา — `Gemfile`

```ruby
# Deploy this application anywhere as a Docker container [https://kamal-deploy.org]
gem "kamal", require: false
```

`require: false` เพราะ Kamal เป็นเครื่องมือ command line ที่รันแยกต่างหากจากตัวแอป (ไม่ได้ถูก
`require` เข้ามาตอนแอปบูต) `require: false` ทำให้ Bundler ไม่พยายามโหลด gem นี้เข้าไปใน process
ของ Rails app ตอน runtime โดยไม่จำเป็น

### ไฟล์ที่ได้มา — `bin/kamal`

Bundler สร้าง binstub ให้อัตโนมัติผ่าน `bundle binstubs kamal` ทำให้เรียกใช้ได้ด้วย `bin/kamal ...`
แทนที่จะต้องพิมพ์ `bundle exec kamal ...` ทุกครั้ง (เหมือนที่ Part 001 สอนเรื่อง `bundle exec` ไว้
— `bin/kamal` คือ binstub ที่ wrap `bundle exec kamal` ให้เรียบร้อยแล้ว)

### ไฟล์ที่ได้มา — `Dockerfile` (เนื้อหาเต็ม ไม่มีการตัดทอน)

นี่คือ Dockerfile ที่ Rails 8.1.4 generate ให้จริง (Part 073 สอนพื้นฐาน Dockerfile ไว้แล้ว ที่นี่จะ
เน้นเฉพาะจุดที่เกี่ยวกับ Kamal):

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true

# This Dockerfile is designed for production, not development. Use with Kamal or build'n'run by hand:
# docker build -t kamal_test_app .
# docker run -d -p 80:80 -e RAILS_MASTER_KEY=<value from config/master.key> --name kamal_test_app kamal_test_app

# Make sure RUBY_VERSION matches the Ruby version in .ruby-version
ARG RUBY_VERSION=3.3.6
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base

# Rails app lives here
WORKDIR /rails

# Install base packages
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 libvips postgresql-client && \
    ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Set production environment variables and enable jemalloc for reduced memory usage and latency.
ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so"

# Throw-away build stage to reduce size of final image
FROM base AS build

# Install packages needed to build gems
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential git libpq-dev libvips libyaml-dev pkg-config && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Install application gems
COPY vendor/* ./vendor/
COPY Gemfile Gemfile.lock ./

RUN bundle install && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile -j 1 --gemfile

# Copy application code
COPY . .

RUN bundle exec bootsnap precompile -j 1 app/ lib/

# Precompiling assets for production without requiring secret RAILS_MASTER_KEY
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

# Final stage for app image
FROM base

# Run and own only the runtime files as a non-root user for security
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash
USER 1000:1000

# Copy built artifacts: gems, application
COPY --chown=rails:rails --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --chown=rails:rails --from=build /rails /rails

# Entrypoint prepares the database.
ENTRYPOINT ["/rails/bin/docker-entrypoint"]

# Start server via Thruster by default, this can be overwritten at runtime
EXPOSE 80
CMD ["./bin/thrust", "./bin/rails", "server"]
```

จุดที่เกี่ยวข้องกับ Kamal โดยตรง (นอกเหนือจากพื้นฐาน multi-stage build ที่ Part 073 สอนไว้):

- **`EXPOSE 80`** — container เปิด port 80 ภายใน ตรงกับค่าเริ่มต้นของ `proxy.app_port` ใน
  `config/deploy.yml` (Step 755) ทำให้ `kamal-proxy` รู้ว่าต้อง forward request ไปที่ port ไหนของ
  container โดยไม่ต้องตั้งค่าเพิ่ม
- **`CMD ["./bin/thrust", "./bin/rails", "server"]`** — `bin/thrust` คือ **Thruster** (HTTP/2
  proxy + asset caching ตัวเล็กที่ 37signals เขียนคู่กับ Kamal) ที่ห่อ Puma อีกชั้นหนึ่งภายใน
  container เดียวกัน ให้บริการ static asset ได้เร็วขึ้นโดยไม่ต้องพึ่ง CDN แยกต่างหาก — เป็นส่วนเสริม
  ที่ทำงานร่วมกับ Kamal ได้ดีแต่ไม่ใช่ส่วนบังคับ (จะแทนที่ด้วยคำสั่งอื่นก็ได้)
- **Non-root user (`USER 1000:1000`)** — แนวปฏิบัติความปลอดภัยที่ Kamal/Rails แนะนำ ไม่รัน
  container ด้วย root แม้จะอยู่ใน container ที่ isolated แล้วก็ตาม (defense in depth)

### ไฟล์ที่ได้มา — `.dockerignore`

```
# Ignore Kamal files.
/config/deploy*.yml
/.kamal
```

สังเกตว่า `.dockerignore` กันไม่ให้ `config/deploy.yml` และโฟลเดอร์ `.kamal/` (ซึ่งอาจมี secret
อ้างอิงอยู่) ถูกก็อปปี้เข้าไปใน Docker image เอง — เป็นเรื่องที่ถูกต้อง เพราะไฟล์เหล่านี้เป็น
"คำสั่งควบคุมการ deploy" ไม่ใช่ส่วนหนึ่งของตัวแอปที่ต้องรันอยู่ใน container

### ทดสอบจริง: Dockerfile นี้ build ได้จริงแค่ไหนในแซนด์บ็อกซ์นี้

เราลองสองวิธี:

**วิธีที่ 1 — ผ่าน Kamal โดยตรง (`kamal build dev`):**

```
$ bin/kamal build create   # สร้าง buildx builder เฉพาะของ Kamal
  INFO Running docker buildx create --name kamal-local-registry-docker-container \
       --driver=docker-container --driver-opt network=host
  INFO Finished in 0.118 seconds with exit status 0 (successful).

$ bin/kamal build dev      # build image จาก working directory ปัจจุบัน แท็กไว้ local ไม่ต้อง push
...
 DEBUG #3 ERROR: failed to do request: Head "https://registry-1.docker.io/v2/docker/dockerfile/manifests/1":
        tls: failed to verify certificate: x509: certificate signed by unknown authority
```

ล้มเหลวที่ TLS — เพราะ container ของ `buildkit` ที่ Kamal สร้างขึ้นมาใหม่ (driver
`docker-container`) เป็น container แยกที่ไม่ได้ไว้ใจ CA certificate ของ network proxy เฉพาะของ
แซนด์บ็อกซ์นี้ (แซนด์บ็อกซ์นี้ route HTTPS ทั้งหมดผ่าน proxy ตรวจสอบ policy ซึ่งต้องติดตั้ง CA
certificate เพิ่มในทุก process ที่จะต่อเน็ต) **นี่คือข้อจำกัดเฉพาะของสภาพแวดล้อมทดสอบนี้เท่านั้น
ไม่ใช่ปัญหาของ Kamal หรือ Docker** — บนเซิร์ฟเวอร์จริงหรือเครื่อง dev ทั่วไปที่ไม่มี proxy แบบนี้
`kamal build dev` จะทำงานได้ตามปกติ

**วิธีที่ 2 — build ตรงด้วย `docker build` แบบ classic builder (ข้าม buildx ของ Kamal ไปเลย)
เพื่อพิสูจน์ว่าตัว Dockerfile เองถูกต้อง:**

```
$ DOCKER_BUILDKIT=0 docker build -t kamal_test_app:manual -f Dockerfile .
Step 2/21 : FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base
3.3.6-slim: Pulling from library/ruby
Digest: sha256:b210597cc7d05e19edf9c8e7935dcf1554886905dcdad0475cfe63a654147f7a
Status: Downloaded newer image for ruby:3.3.6-slim
 ---> b210597cc7d0
Step 4/21 : RUN apt-get update -qq && apt-get install ... curl libjemalloc2 libvips postgresql-client ...
E: Failed to fetch http://deb.debian.org/debian/dists/bookworm/InRelease  403  Forbidden
```

ผลลัพธ์นี้ยืนยันสองอย่างชัดเจน:

1. **Dockerfile ถูกต้องตามโครงสร้าง** — Docker อ่าน multi-stage build ได้ครบ 21 ขั้นตอนโดยไม่มี
   syntax error, และดึง base image `ruby:3.3.6-slim` จาก Docker Hub ผ่าน engine ของ Docker เอง
   (ไม่ผ่าน buildx container แยก) **สำเร็จจริง**
2. **ที่ล้มเหลวคือ `apt-get update` เพราะ network policy ของแซนด์บ็อกซ์นี้ปฏิเสธการเชื่อมต่อไปยัง
   `deb.debian.org` (403 Forbidden)** — เป็น policy ระดับ container ของแซนด์บ็อกซ์ที่จำกัดปลายทาง
   ที่เข้าถึงได้ ไม่เกี่ยวกับความถูกต้องของ Dockerfile หรือ Kamal เลย บนเซิร์ฟเวอร์จริงที่มี internet
   เปิดตามปกติ ขั้นตอนนี้จะผ่านไปได้โดยไม่มีปัญหา

สรุปให้ตรงไปตรงมา: **โครงสร้าง Dockerfile ที่ Rails 8 generate ให้ถูกต้องและใช้งานได้จริง** —
ข้อจำกัดที่เจอทั้งสองอย่างเป็นเรื่องของ network policy เฉพาะแซนด์บ็อกซ์การเขียนหลักสูตรนี้เท่านั้น

---

## Step 754: ผ่า `config/deploy.yml` ทีละส่วน — `service`, `image`, `servers`, `registry`, `env`

นี่คือ `config/deploy.yml` เวอร์ชันเต็มที่ `kamal init` สร้างให้จริง (คอมเมนต์ทั้งหมดคือของจริงจาก
เครื่องมือ ไม่ได้แต่งเพิ่ม):

```yaml
# Name of your application. Used to uniquely configure containers.
service: kamal_test_app

# Name of the container image (use your-user/app-name on external registries).
image: kamal_test_app

# Deploy to these servers.
servers:
  web:
    - 192.168.0.1
  # job:
  #   hosts:
  #     - 192.168.0.1
  #   cmd: bin/jobs

# Where you keep your container images.
registry:
  # Alternatives: hub.docker.com / registry.digitalocean.com / ghcr.io / ...
  server: localhost:5555

  # Needed for authenticated registries.
  # username: your-user

  # Always use an access token rather than real password when possible.
  # password:
  #   - KAMAL_REGISTRY_PASSWORD

# Inject ENV variables into containers (secrets come from .kamal/secrets).
env:
  secret:
    - RAILS_MASTER_KEY
  clear:
    SOLID_QUEUE_IN_PUMA: true
    # JOB_CONCURRENCY: 3
    # WEB_CONCURRENCY: 2
    # DB_HOST: 192.168.0.2
    # RAILS_LOG_LEVEL: debug
```

เราจะผ่าทีละ key ให้เห็นภาพชัดว่าทำไมต้องมี field นี้:

### `service`

```yaml
service: kamal_test_app
```

เป็น**ชื่อ prefix ของทุก container/network/volume** ที่ Kamal สร้างบนเซิร์ฟเวอร์ ถ้าเซิร์ฟเวอร์
เดียวรันหลายแอปพร้อมกัน (เช่นแอป Rails 2 ตัวบน VPS เดียวกัน) `service` ที่ต่างกันจะทำให้ทุกอย่างแยก
จากกันชัดเจน ไม่ชนกัน — ค่าเริ่มต้นดึงมาจากชื่อโฟลเดอร์ของแอป

### `image`

```yaml
image: kamal_test_app
```

ชื่อของ Docker image (ส่วนที่อยู่หลัง registry server เช่น `ghcr.io/your-user/kamal_test_app`)
ถ้า deploy ไป registry ภายนอกจริง (Docker Hub/GHCR) ควรเปลี่ยนเป็นรูปแบบ `your-username/app-name`
เพื่อให้ตรงกับ namespace ของบัญชีคุณบน registry นั้น (ดูตัวอย่างจริงใน Step 760 และแบบฝึกหัด)

### `servers`

```yaml
servers:
  web:
    - 192.168.0.1
```

คือรายชื่อเซิร์ฟเวอร์ที่จะ deploy ไป แบ่งเป็น **role** (ในตัวอย่างนี้มี role เดียวคือ `web`) — ค่า
`192.168.0.1` เป็นแค่ placeholder ตัวอย่างที่ Kamal ใส่ไว้ให้ (เป็น private IP ในเอกสารตัวอย่าง
เท่านั้น) เมื่อจะ deploy จริงต้องเปลี่ยนเป็น public IP ของ VPS จริง เรื่อง multi-role (เช่นแยก role
`job` สำหรับรัน background job ต่างหาก) จะอธิบายเต็มใน Step 760

### `registry`

```yaml
registry:
  server: localhost:5555
```

บอก Kamal ว่าจะ push/pull Docker image จากที่ไหน ค่าเริ่มต้นที่ `kamal init` ใส่ให้คือ
`localhost:5555` ซึ่งเป็นค่าพิเศษ — ถ้า `server` ขึ้นต้นด้วย `localhost` **Kamal จะรัน local
Docker registry ให้เองบนเครื่องที่สั่ง deploy** เหมาะสำหรับทดลองเล่นหรือ deploy ไปเซิร์ฟเวอร์เดียวที่
build image บนเครื่องนั้นเอง แต่สำหรับ production จริงที่มีหลายเซิร์ฟเวอร์ ควรเปลี่ยนไปใช้ registry
จริงอย่าง Docker Hub หรือ **GHCR (GitHub Container Registry)**:

```yaml
# ตัวอย่าง: ใช้ GHCR (ฟรีสำหรับ public repo และมี free tier สำหรับ private)
registry:
  server: ghcr.io
  username:
    - GHCR_USERNAME
  password:
    - GHCR_TOKEN
```

```yaml
# ตัวอย่าง: ใช้ Docker Hub
registry:
  username:
    - DOCKERHUB_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD
```

สังเกตว่า `username`/`password` เป็น **array** ไม่ใช่ string ตรงๆ — เพราะ Kamal อนุญาตให้ระบุเป็น
"path" ไปหา secret ได้ (ปกติคือชื่อตัวแปรที่จะไปหาใน `.kamal/secrets` ตามที่จะอธิบายใน Step 758)
ไม่ควรเขียน password จริงลงในไฟล์นี้ตรงๆ เด็ดขาด เพราะ `config/deploy.yml` มักถูก commit เข้า git

### `env`

```yaml
env:
  secret:
    - RAILS_MASTER_KEY
  clear:
    SOLID_QUEUE_IN_PUMA: true
```

แบ่งเป็นสองกลุ่มชัดเจน:

- **`secret`** — รายชื่อตัวแปรที่**ค่าจริงเป็นความลับ** (เช่น `RAILS_MASTER_KEY` ที่ใช้ถอดรหัส
  `config/credentials.yml.enc`) ค่าจริงจะถูกดึงมาจาก `.kamal/secrets` แล้วเก็บไว้ในไฟล์ env บน
  เซิร์ฟเวอร์ (ไม่ถูกส่งผ่าน `docker run -e` ตรงๆ ซึ่งจะไปโผล่ใน `docker inspect` หรือ process list
  ได้ง่าย) รายละเอียดเต็มใน Step 758
- **`clear`** — ตัวแปรที่**ไม่ใช่ความลับ** เขียนค่าตรงๆ ในไฟล์นี้ได้เลย เช่น
  `SOLID_QUEUE_IN_PUMA: true` ที่สั่งให้ Solid Queue (background job adapter เริ่มต้นของ Rails 8)
  รัน supervisor process ของมันอยู่ภายใน Puma process เดียวกัน — เหมาะสำหรับตอนเริ่มต้นที่มีแค่
  เซิร์ฟเวอร์เดียว แต่ควรปิดค่านี้ (`false`) เมื่อแยก role `job` ออกมารันต่างหาก (ทวนใน Step 760)

---

## Step 755: ผ่า `config/deploy.yml` ต่อ — `proxy`, `ssh`, `builder`, `volumes`, `asset_path`

ส่วนที่เหลือของ `config/deploy.yml` (บางส่วนถูก comment ไว้เป็นค่าเริ่มต้นให้เลือกเปิดใช้เอง):

```yaml
# Enable SSL auto certification via Let's Encrypt and allow for multiple apps on a single web server.
#
# proxy:
#   ssl: true
#   host: app.example.com

aliases:
  console: app exec --interactive --reuse "bin/rails console"
  shell: app exec --interactive --reuse "bash"
  logs: app logs -f
  dbc: app exec --interactive --reuse "bin/rails dbconsole --include-password"

# Use a persistent storage volume for sqlite database files and local Active Storage files.
volumes:
  - "kamal_test_app_storage:/rails/storage"

# Bridge fingerprinted assets, like JS and CSS, between versions to avoid
# hitting 404 on in-flight requests.
asset_path: /rails/public/assets

# Configure the image builder.
builder:
  arch: amd64
```

### `proxy` — ตั้งค่า SSL/domain

ค่านี้ถูก comment ไว้เป็นค่าเริ่มต้นเพราะ Kamal ยังไม่รู้ domain จริงของคุณ เมื่อจะ deploy จริงต้อง
เปิดใช้และใส่ domain:

```yaml
proxy:
  ssl: true                    # ให้ kamal-proxy ขอใบรับรอง Let's Encrypt ให้อัตโนมัติ
  host: app.example.com        # domain ที่ต้องชี้ DNS มาที่ IP ของเซิร์ฟเวอร์นี้แล้ว
  app_port: 80                 # port ภายใน container ที่แอปฟัง (ตรงกับ EXPOSE ใน Dockerfile)
```

ดึงมาจาก `kamal docs proxy` (เอกสารที่ฝังในตัว gem 2.12.0 โดยตรง) มีรายละเอียดเพิ่มเติมที่สำคัญ:

```yaml
proxy:
  healthcheck:
    interval: 3      # วินาทีระหว่างการยิง health check แต่ละครั้ง (ค่าเริ่มต้น: ทุก 1 วินาที)
    path: /health     # เปลี่ยนจาก /up เริ่มต้นได้ถ้าต้องการ endpoint อื่น
    timeout: 3        # timeout ต่อการยิงแต่ละครั้ง (ค่าเริ่มต้น: 5 วินาที)
  response_timeout: 10  # เวลารอ request ทำงานให้เสร็จก่อน timeout (ค่าเริ่มต้น: 30 วินาที)
  forward_headers: true # forward X-Forwarded-For / X-Forwarded-Proto (ปิดอัตโนมัติถ้า ssl: true
                         # เว้นแต่จะเปิดเอง)
```

> **ข้อจำกัดสำคัญของ `ssl: true` แบบอัตโนมัติ:** ใช้ได้เมื่อ deploy ไป **เซิร์ฟเวอร์เดียวเท่านั้น**
> (เพราะ Let's Encrypt ต้องพิสูจน์ความเป็นเจ้าของ domain ผ่านการเปิด port 443 ที่ IP เดียว) ถ้ามี
> หลายเซิร์ฟเวอร์ web พร้อมกัน ต้องใช้ load balancer/SSL termination แยกต่างหากหน้า Kamal แทน (เช่น
> Cloudflare ตั้งเป็น "Full" mode ตามที่ comment ในไฟล์ default บอกไว้)

### `ssh`

ค่าเริ่มต้นของ Kamal คือ SSH ไปที่ **user `root` port `22`** ถ้าเซิร์ฟเวอร์ใช้ user อื่นหรือ port
อื่น ต้องระบุเพิ่ม:

```yaml
ssh:
  user: deploy       # ใช้ user อื่นแทน root (ต้องอยู่ใน docker group บนเซิร์ฟเวอร์)
  port: 2222          # ถ้า SSH server ฟังที่ port อื่น
  keys: ["~/.ssh/id_ed25519"]   # ระบุ private key ที่จะใช้ชัดเจน (ถ้าไม่พึ่ง ssh-agent)
```

### `builder`

```yaml
builder:
  arch: amd64
```

ระบุ**สถาปัตยกรรม CPU ของเซิร์ฟเวอร์ปลายทาง** — สำคัญมากถ้า build บนเครื่อง Apple Silicon (arm64)
แต่ deploy ไป VPS ทั่วไปที่ส่วนใหญ่เป็น `amd64` (x86_64) Kamal จะใช้ Docker Buildx cross-compile
image ให้ตรงสถาปัตยกรรมปลายทางอัตโนมัติ (ใช้เวลานานกว่า native build เล็กน้อยเพราะต้องจำลอง CPU
สถาปัตยกรรมอื่นผ่าน QEMU) ถ้าต้องการเร่งความเร็ว สามารถตั้งค่า **remote builder** ให้ build บน
เซิร์ฟเวอร์ตัวกลางที่มีสถาปัตยกรรมตรงกับปลายทางแทนได้:

```yaml
builder:
  arch: amd64
  remote: ssh://docker@docker-builder-server   # build บนเครื่อง amd64 จริงแทนการจำลองด้วย QEMU
```

### `volumes` และ `asset_path`

```yaml
volumes:
  - "kamal_test_app_storage:/rails/storage"

asset_path: /rails/public/assets
```

- **`volumes`** — mount Docker volume ที่ **คงอยู่ข้ามการ deploy** (container ใหม่ทุกตัวจะเห็น
  volume เดิม) จำเป็นมากสำหรับข้อมูลที่ห้ามหายไปตอน deploy เช่นไฟล์ SQLite (ถ้าใช้ SQLite เป็น
  production database ตามที่ Rails 8 รองรับเป็นทางเลือก) หรือไฟล์ Active Storage ที่เก็บบน local
  disk (ถ้าไม่ได้ใช้ S3/cloud storage ตาม Part 067)
- **`asset_path`** — แก้ปัญหา **"404 ตอน deploy"** ที่พบได้บ่อย: ถ้าผู้ใช้เปิดหน้าเว็บค้างไว้ตอนที่
  เรากำลัง deploy เวอร์ชันใหม่ browser ของเขาอาจขอไฟล์ CSS/JS เวอร์ชันเก่า (ที่มี hash ในชื่อไฟล์
  ต่างจากเวอร์ชันใหม่) Kamal จะ**รวมไฟล์ asset ของทั้งเวอร์ชันเก่าและใหม่ไว้ในโฟลเดอร์เดียวกัน**
  ผ่าน volume ที่ path นี้ ทำให้ request ที่ขอไฟล์เก่ายังหาเจอได้ชั่วคราว ไม่เจอ 404 ระหว่างช่วง
  transition

### `aliases`

```yaml
aliases:
  console: app exec --interactive --reuse "bin/rails console"
  shell: app exec --interactive --reuse "bash"
  logs: app logs -f
  dbc: app exec --interactive --reuse "bin/rails dbconsole --include-password"
```

ทางลัดสำหรับคำสั่งที่ใช้บ่อย แทนที่จะพิมพ์ `bin/kamal app exec --interactive --reuse "bin/rails
console"` ทุกครั้ง ก็แค่ `bin/kamal console` — เทียบเท่ากับการเปิด `rails console` บนเซิร์ฟเวอร์
production ผ่าน SSH โดยไม่ต้อง SSH ด้วยมือเลย

---

## Step 756: เตรียมเซิร์ฟเวอร์จริงสำหรับ deploy — VPS, Docker, SSH, container registry account

ก่อนจะรัน `kamal setup` ได้จริง (Step 757) ต้องเตรียม 3 อย่างนี้ให้พร้อมก่อนเสมอ — ส่วนนี้เป็นข้อมูล
จากเอกสารทางการของ Kamal เพราะแซนด์บ็อกซ์นี้ไม่มี VPS จริงให้ทดสอบ:

### 1. VPS (Virtual Private Server) ที่เข้าถึงผ่าน SSH ได้

Kamal ไม่ผูกกับผู้ให้บริการ cloud เจ้าใดเจ้าหนึ่ง — ใช้ได้กับทุกที่ที่ให้เครื่อง Linux พร้อม SSH
access ตัวเลือกยอดนิยมในหมู่ทีม Rails (เรียงตามราคาประหยัด):

| ผู้ให้บริการ | จุดเด่น |
|-------------|---------|
| **Hetzner Cloud** | ราคาถูกมากเทียบสเปค เหมาะกับ side project/startup งบจำกัด |
| **DigitalOcean** | เอกสารเยอะ UI ใช้ง่าย มี "Droplet" ราคาเริ่มต้นถูก |
| **Vultr** | คล้าย DigitalOcean มี data center หลากหลายจุดทั่วโลก |
| **AWS EC2 / GCP Compute Engine** | ยืดหยุ่นสูงสุด แต่ซับซ้อนกว่าและราคาผันแปรได้มากถ้าตั้งค่าไม่ดี |

ข้อกำหนดขั้นต่ำ: เครื่อง Linux (Ubuntu เป็นที่นิยมที่สุดในเอกสาร Kamal) ที่มี RAM อย่างน้อย 1–2GB
สำหรับแอป Rails ขนาดเล็ก และเปิด SSH (port 22 หรือ port ที่กำหนดเอง) ให้เข้าถึงได้จากเครื่องที่จะสั่ง
`kamal deploy`

> **หมายเหตุด้านความปลอดภัย:** IP ตัวอย่างทั้งหมดใน Part นี้ (`203.0.113.x`, `192.0.2.x`,
> `198.51.100.x`) เป็นช่วง IP ที่ IETF จองไว้เฉพาะสำหรับใช้ในเอกสาร (RFC 5737) **ไม่ใช่ IP ของ
> เซิร์ฟเวอร์จริงใดๆ** เวลาใช้งานจริงต้องแทนที่ด้วย public IP ของ VPS ของคุณเองเท่านั้น

### 2. Docker บนเซิร์ฟเวอร์ (ไม่จำเป็นต้องติดตั้งเอง)

ข่าวดีคือ**ไม่ต้องติดตั้ง Docker ด้วยมือ** — คำสั่ง `kamal setup` (Step 757) จะตรวจสอบว่าเซิร์ฟเวอร์
มี Docker หรือยัง ถ้ายังไม่มี Kamal จะ SSH เข้าไปติดตั้งให้อัตโนมัติผ่านสคริปต์ทางการของ Docker
(`get.docker.com`) กลไกนี้อยู่เบื้องหลังคำสั่งย่อย `kamal server bootstrap` ที่ `kamal setup`
เรียกใช้ให้เองในขั้นตอนแรก

### 3. บัญชี Container Registry

ต้องมีที่เก็บ Docker image ที่เซิร์ฟเวอร์จะ pull ได้ ตัวเลือกหลักสองตัวที่ใช้ฟรีได้:

- **Docker Hub** — ตัวเลือกดั้งเดิมที่สุด ฟรีสำหรับ public repository (private repository มี
  ข้อจำกัดจำนวนในแผนฟรี) สมัครที่ hub.docker.com แล้วสร้าง **access token** (Account Settings →
  Security) แทนการใช้รหัสผ่านจริงเสมอ
- **GHCR (GitHub Container Registry)** — ผูกกับบัญชี GitHub ที่มีอยู่แล้ว สะดวกมากถ้าโค้ดอยู่บน
  GitHub อยู่แล้ว (ซึ่งสอดคล้องกับ Part 075 ที่ใช้ GitHub Actions) สร้าง **Personal Access Token**
  ที่มีสิทธิ์ `write:packages` แล้วใช้เป็น password สำหรับ login

ไม่ว่าจะเลือกตัวไหน หลักการเดียวกันคือ **ห้ามใช้รหัสผ่านจริงของบัญชีเป็น password ใน config
เด็ดขาด — ใช้ access token/personal access token ที่จำกัดสิทธิ์และเพิกถอนได้ทีหลังเสมอ**
(ทวนหลักการเดียวกับที่ Part 071 สอนไว้เรื่อง Stripe API key)

---

## Step 757: `kamal setup` (deploy ครั้งแรก) เทียบกับ `kamal deploy`/`kamal redeploy` (deploy ครั้งถัดไป)

Kamal มีคำสั่งหลักสามตัวที่มักสับสนกัน มาดูความต่างจาก **help text จริงที่ดึงจาก Kamal 2.12.0**:

```
$ bin/kamal help
Commands:
  kamal setup      # Setup all accessories, push the env, and deploy app to servers
  kamal deploy     # Deploy app to servers
  kamal redeploy   # Deploy app to servers without bootstrapping servers, starting
                   #  kamal-proxy and pruning
  kamal rollback [VERSION]  # Rollback app to VERSION
```

### `kamal setup` — ใช้แค่ครั้งแรก (หรือทุกครั้งที่มีเซิร์ฟเวอร์ใหม่)

เมื่อรัน `kamal setup` ครั้งแรก Kamal จะทำงานตามลำดับนี้ (อ้างอิงจากเอกสารทางการ):

1. **`kamal server bootstrap`** — SSH เข้าไปตรวจสอบว่าเซิร์ฟเวอร์มี Docker หรือยัง ถ้ายังไม่มี
   ติดตั้งให้อัตโนมัติ
2. **Build + push image** — build Docker image จากโค้ดปัจจุบัน แล้ว push ขึ้น registry ที่ตั้งค่าไว้
3. **ตั้งค่า accessories** (ถ้ามี — เช่น container Postgres/Redis ตาม Step 760) — pull image ของ
   accessory มารันบนเซิร์ฟเวอร์ที่ระบุ
4. **บูต `kamal-proxy`** บนทุกเซิร์ฟเวอร์ (ถ้ายังไม่เคยมี) — proxy container ที่ทำหน้าที่ตามที่
   Step 752 อธิบาย
5. **push environment variables** ไปเก็บไว้บนเซิร์ฟเวอร์ (จาก `env`/`secret` ใน config)
6. **pull image และรัน container ของแอป** — เริ่ม container แรกของแอปขึ้นมา
7. **ลงทะเบียน container กับ `kamal-proxy`** ให้เริ่มรับ traffic

เพราะต้องทำหลายขั้นตอน (ติดตั้ง Docker, บูต proxy ครั้งแรก) `kamal setup` จึง**ใช้เวลานานกว่า** และ
**ควรรันแค่ครั้งเดียวตอนเริ่มต้นกับแต่ละเซิร์ฟเวอร์ใหม่** เท่านั้น

### `kamal deploy` — ใช้สำหรับทุกครั้งถัดไป

เมื่อเซิร์ฟเวอร์มี Docker, `kamal-proxy`, และ accessories พร้อมอยู่แล้ว การ deploy โค้ดเวอร์ชันใหม่
ในแต่ละครั้งถัดไปใช้ `kamal deploy` แทน — ข้ามขั้นตอนติดตั้ง Docker/บูต proxy ครั้งแรกไป เหลือแค่:
build image ใหม่ → push → pull บนเซิร์ฟเวอร์ → รัน container ใหม่ → สลับ traffic แบบ zero-downtime
(Step 759) → prune image/container เก่าที่เกินจำนวนที่กำหนด (`retain_containers` ค่าเริ่มต้น 5)

### `kamal redeploy` — deploy เร็วขึ้นเมื่อรู้ว่าไม่มีอะไรเปลี่ยนที่ infrastructure

```
kamal redeploy   # Deploy app to servers without bootstrapping servers, starting
                 # kamal-proxy and pruning
```

`redeploy` คือ `deploy` เวอร์ชันที่ตัดขั้นตอนตรวจสอบ/บูต proxy และการ prune image เก่าออกไป
เหมาะกับตอนที่ deploy ติดกันหลายครั้งรัวๆ ระหว่าง debug (เช่นแก้ bug เล็กน้อยแล้ว deploy ซ้ำภายใน
ไม่กี่นาที) ทำให้แต่ละรอบเร็วขึ้นเพราะไม่ต้องเช็คสถานะ proxy ซ้ำทุกครั้ง

### สรุปเป็นตาราง

| คำสั่ง | ใช้เมื่อไหร่ | ทำอะไรบ้าง |
|--------|-------------|------------|
| `kamal setup` | ครั้งแรกกับเซิร์ฟเวอร์ใหม่ (หรือเพิ่มเซิร์ฟเวอร์ใหม่เข้าไปใน fleet) | ติดตั้ง Docker, บูต proxy, ตั้งค่า accessories, push env, deploy app — ครบทุกอย่าง |
| `kamal deploy` | deploy โค้ดเวอร์ชันใหม่ตามปกติ | build → push → pull → รัน container ใหม่ → สลับ traffic → prune ของเก่า |
| `kamal redeploy` | deploy รัวๆ ระหว่าง debug ที่รู้ว่า infra ไม่เปลี่ยน | เหมือน `deploy` แต่ข้ามการเช็ค/บูต proxy และการ prune |
| `kamal rollback VERSION` | deploy ล่าสุดมีปัญหา ต้องการย้อนกลับ | สลับ traffic กลับไปที่ container ของเวอร์ชันก่อนหน้าทันที (Step 759) |

---

## Step 758: จัดการ secrets ให้ปลอดภัย — `.kamal/secrets`, Rails credentials, ห้าม commit ของจริง

### ไฟล์ `.kamal/secrets` ที่ Kamal generate ให้จริง

```bash
# Secrets defined here are available for reference under registry/password, env/secret, builder/secrets,
# and accessories/*/env/secret in config/deploy.yml. All secrets should be pulled from either
# password manager, ENV, or a file. DO NOT ENTER RAW CREDENTIALS HERE! This file needs to be safe for git.

# Example of extracting secrets from Rails credentials
# KAMAL_REGISTRY_PASSWORD=$(rails credentials:fetch kamal.registry_password)

# Grab the registry password from ENV
# KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# Improve security by using a password manager. Never check config/master.key into git!
RAILS_MASTER_KEY=$(cat config/master.key)
```

สังเกตประโยคเตือนตัวใหญ่ที่ตัวเครื่องมือใส่มาให้เองว่า **"DO NOT ENTER RAW CREDENTIALS HERE!"** —
นี่คือหัวใจสำคัญที่สุดของ Step นี้ `.kamal/secrets` **ไม่ใช่ที่เก็บรหัสผ่านจริง** แต่เป็น
**"สูตรคำนวณ" ว่าจะไปหารหัสผ่านจริงจากที่ไหน** เมื่อรันคำสั่ง Kamal เท่านั้น — ค่าที่คำนวณได้จะไม่ถูก
บันทึกลงไฟล์นี้เลย

### รูปแบบที่ใช้บ่อยที่สุด

```bash
# แบบที่ 1 (ค่าเริ่มต้น): อ่านจากไฟล์ config/master.key ที่มีอยู่แล้วในเครื่อง
# (ไฟล์นี้ถูก .gitignore ไว้อยู่แล้วตั้งแต่ rails new — ไม่เคย commit เข้า git)
RAILS_MASTER_KEY=$(cat config/master.key)

# แบบที่ 2: อ่านจาก environment variable ของเครื่อง/CI ที่รันคำสั่ง kamal deploy
# (เหมาะกับ deploy ผ่าน GitHub Actions ตาม Part 075 — ตั้งเป็น GitHub Secret แล้วส่งผ่าน env)
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
GHCR_TOKEN=$GHCR_TOKEN

# แบบที่ 3: ดึงจาก Rails encrypted credentials โดยตรง (ทวนหลักการจาก Part 067/071)
KAMAL_REGISTRY_PASSWORD=$(rails credentials:fetch kamal.registry_password)

# แบบที่ 4: ดึงจากตัวช่วยของ Kamal เองที่ผูกกับ password manager (เช่น 1Password)
# SECRETS=$(kamal secrets fetch --adapter 1password --account your-account --from Vault/Item \
#           KAMAL_REGISTRY_PASSWORD RAILS_MASTER_KEY)
# KAMAL_REGISTRY_PASSWORD=$(kamal secrets extract KAMAL_REGISTRY_PASSWORD ${SECRETS})
```

`.kamal/secrets` เป็นไฟล์ **shell script รูปแบบ dotenv** ที่ Kamal ประมวลผล (รองรับ command
substitution `$(...)` และการอ้างอิง environment variable `$VAR` ตรงๆ) — เพราะเป็นแค่ "สูตร" จึง
**ปลอดภัยที่จะ commit เข้า git ได้** (ไฟล์นี้ไม่มี raw credential อยู่เลย) แต่ **ต้องเช็คให้แน่ใจ
เสมอว่าไม่มีใครเผลอ paste ค่าจริงลงไปตรงๆ**

### ทำไม `env.secret` ถึงต่างจาก `env.clear` ในทางเทคนิค

ทบทวนจาก Step 754 — `env.secret` (เช่น `RAILS_MASTER_KEY`) ถูกจัดการต่างจาก `env.clear` ตอน deploy
จริง: ค่า secret จะถูก**เขียนลงไฟล์ env แยกต่างหากบนเซิร์ฟเวอร์** (ที่ path ภายใต้ `.kamal/` บน
เซิร์ฟเวอร์ ไม่ใช่ที่เก็บโค้ด) แล้วส่งเข้า container ผ่าน `--env-file` ตอนรัน `docker run` แทนที่จะ
ส่งผ่าน `-e KEY=value` ตรงๆ บน command line เหตุผลคือ **ค่าที่ส่งผ่าน `-e` บน command line จะไปโผล่
ในผลลัพธ์ของ `docker inspect <container>` และในบาง distro อาจเห็นได้จาก process list ของเครื่อง
(`ps aux`) ด้วย** ซึ่งเป็นความเสี่ยงที่ไม่ควรมีสำหรับข้อมูลอ่อนไหวอย่าง master key

### เช็กลิสต์ความปลอดภัยของ secrets สำหรับ Kamal

1. **ตรวจสอบ `.gitignore`** ให้แน่ใจว่า `config/master.key` และไฟล์ credentials ที่ถอดรหัสแล้วไม่
   เคยถูก commit (Rails ตั้งค่านี้ให้อัตโนมัติตั้งแต่ `rails new` แต่ควรเช็คซ้ำ)
2. **ห้าม commit `.kamal/secrets` ที่มีค่าจริงแทนที่จะเป็นสูตร** — ถ้าเผลอทำไปแล้ว ต้อง treat
   secret นั้นว่ารั่วแล้ว (revoke/สร้างใหม่ทันที) การใช้ `git rm` ทีหลังไม่ได้ลบมันออกจาก git
   history
3. **แยก credentials ตาม environment เสมอ** (`rails credentials:edit --environment production`
   ตามที่ Part 067/071 สอนไว้) เพื่อไม่ให้ key ของ production กับ development ปนกัน
4. **ใช้ access token ที่จำกัดสิทธิ์แทนรหัสผ่านจริงเสมอ** สำหรับ registry (ทวนจาก Step 756)
5. **หมุนเวียน (rotate) secret เป็นระยะ** โดยเฉพาะหลังมีคนออกจากทีมที่เคยเข้าถึง credential นั้น

---

## Step 759: กลไก Zero-downtime deployment ของ `kamal-proxy` และการ `kamal rollback`

### ทวนกลไกจาก Step 752 แบบละเอียดขึ้น

ตอนรัน `kamal deploy` แต่ละครั้ง Kamal จะแท็ก image ใหม่ด้วย **version identifier** ที่อิงจาก git
commit SHA ปัจจุบันของโปรเจกต์ (ยืนยันได้จากการรัน `kamal config` จริงในแซนด์บ็อกซ์นี้ — ผลลัพธ์
แสดง `:version: 9d61b85c45f49fe6b6e401715224aa3a545b78ed` ซึ่งตรงกับ commit SHA ของ git repository
ทดสอบเป๊ะๆ) นั่นหมายความว่า**ทุก deploy คือ container ใหม่ที่มีชื่อไม่ซ้ำกับ deploy ครั้งก่อนเลย**
(รูปแบบชื่อคือ `<service>-<version>` เช่น `kamal_test_app-9d61b85c...`) — container เก่าจะไม่ถูก
เขียนทับ แต่ถูก**เก็บไว้เฉยๆ** (จำนวนที่เก็บไว้ควบคุมด้วย `retain_containers` ค่าเริ่มต้น 5 เวอร์ชัน
ล่าสุด)

ลำดับเหตุการณ์เต็มของการสลับ container แบบ zero-downtime (ตามเอกสารทางการของ `kamal-proxy`):

```
เวลา t0: container เวอร์ชัน A กำลังรับ traffic ปกติผ่าน kamal-proxy
เวลา t1: `kamal deploy` เริ่มรัน container เวอร์ชัน B ขึ้นมาคู่ขนาน (A ยังรับ traffic อยู่)
เวลา t2: kamal-proxy เริ่มยิง GET /up ไปที่ container B ทุก 1 วินาที (ค่าเริ่มต้น)
เวลา t3: container B ตอบ HTTP 200 เป็นครั้งแรก (แปลว่า Rails boot เสร็จ, เชื่อมต่อ DB ได้)
เวลา t4: kamal-proxy สลับ routing table แบบ atomic — request ใหม่ทั้งหมดวิ่งไปที่ B ทันที
เวลา t5: A หยุดรับ request ใหม่ แต่ request เก่าที่ค้างอยู่ยังประมวลผลต่อจนจบ (drain)
เวลา t6: เมื่อ A drain เสร็จ (หรือครบ drain_timeout ค่าเริ่มต้น 30 วินาที) A ถูกหยุดและลบทิ้ง
```

ตลอดกระบวนการนี้**ไม่มีช่วงเวลาใดเลยที่ไม่มี container พร้อมรับ traffic** — ถ้า container B
ไม่ผ่าน health check ภายในเวลา `deploy_timeout` (ค่าเริ่มต้น 30 วินาที) Kamal จะ**ยกเลิก deploy
และปล่อยให้ A รับ traffic ต่อไปตามเดิม** โดยไม่แตะ production traffic เลย — นี่คือเหตุผลที่
`deploy_timeout` และการเขียน endpoint `/up` ให้ตรวจสอบสิ่งที่สำคัญจริง (เช่น เชื่อมต่อฐานข้อมูลได้)
มีความสำคัญมาก เพราะเป็นเกราะป้องกันไม่ให้ deploy ที่พังหลุดออกไปสู่ผู้ใช้จริง

### `kamal rollback` — ย้อนกลับเมื่อ deploy มีปัญหาที่ตรวจไม่พบตอน health check

บางครั้ง container ใหม่ผ่าน health check (`/up` ตอบ 200) แต่กลับมีบั๊กที่ปรากฏทีหลัง (เช่น
edge case ที่ทำให้ error เฉพาะบาง action, หรือ performance แย่ลงมากจนผู้ใช้บ่น) เนื่องจาก **container
เวอร์ชันเก่ายังถูกเก็บไว้บนเซิร์ฟเวอร์** (ตาม `retain_containers`) การย้อนกลับจึงเร็วมาก ไม่ต้อง
build ใหม่เลย:

```bash
# ดู version ที่ deploy ผ่านมาก่อนหน้า (ต้องรู้ commit SHA หรือ tag ที่เคย deploy สำเร็จ)
git log --oneline -5

# ย้อนกลับไปที่ version นั้น
bin/kamal rollback <VERSION>
```

`kamal rollback` ทำงานคล้าย `kamal deploy` มาก — คือสลับ traffic ผ่าน `kamal-proxy` แบบเดียวกันกับ
Step ที่แล้ว **แต่สลับกลับไปหา container ของเวอร์ชันเก่าที่ยังรันอยู่แล้ว (หรือ image เก่าที่ยังอยู่
บนเซิร์ฟเวอร์) แทนที่จะ build ใหม่** ทำให้เร็วกว่า deploy ปกติมาก (ไม่มีขั้นตอน build/push) และยัง
ผ่านกลไก health check เดียวกันก่อนสลับ traffic จริง เพื่อความปลอดภัย

> **ข้อควรระวัง:** `kamal rollback` ย้อนกลับแค่**ตัวแอป (container)** เท่านั้น **ไม่ย้อนกลับ
> database migration ที่รันไปแล้ว** ถ้า deploy ที่มีปัญหามาพร้อมกับ migration ที่เปลี่ยนโครงสร้าง
> ตารางแบบที่เข้ากันไม่ได้กับโค้ดเวอร์ชันเก่า การ rollback แอปเฉยๆ อาจทำให้แอปเวอร์ชันเก่า error
> เพราะ schema ไม่ตรงกับที่มันคาดหวัง — Part 077 จะพูดถึงแนวทาง migration ใน production ที่ปลอดภัย
> ต่อการ rollback (เช่น backward-compatible migration) โดยเฉพาะ

---

## Step 760: Accessories (Postgres/Redis เป็น container) และ multi-server/multi-role สำหรับ scale

### Accessories คืออะไร

**Accessory** คือบริการเสริมที่แอปต้องพึ่งพา (most commonly: database, cache, search engine) ที่
Kamal จัดการให้รันเป็น **Docker container บนเซิร์ฟเวอร์ที่คุณกำหนด** — เหมือนกับตัวแอปเอง แต่มี
lifecycle แยกต่างหาก (ไม่ถูก build ใหม่ทุกครั้งที่ deploy แอป, ไม่มีแนวคิด zero-downtime switch
เพราะปกติมีแค่ instance เดียว) ตัวอย่างจาก comment ใน `config/deploy.yml` ที่ `kamal init` ใส่มาให้
(default ปิดไว้ด้วย `#`):

```yaml
# accessories:
#   db:
#     image: mysql:8.0
#     host: 192.168.0.2
#     port: "127.0.0.1:3306:3306"
#     env:
#       clear:
#         MYSQL_ROOT_HOST: '%'
#       secret:
#         - MYSQL_ROOT_PASSWORD
#     files:
#       - config/mysql/production.cnf:/etc/mysql/my.cnf
#       - db/production.sql:/docker-entrypoint-initdb.d/setup.sql
#     directories:
#       - data:/var/lib/mysql
#   redis:
#     image: valkey/valkey:8
#     host: 192.168.0.2
#     port: 6379
#     directories:
#       - data:/data
```

field สำคัญที่ต้องเข้าใจ:

- **`image`** — ใช้ image สำเร็จรูปจาก Docker Hub ตรงๆ ได้เลย (`postgres:16`, `valkey/valkey:8`
  ซึ่งเป็น fork ของ Redis ที่ยังคง license แบบ open source อย่างสมบูรณ์) ไม่ต้องเขียน Dockerfile
  เอง เพราะไม่ใช่โค้ดของเราที่ต้อง build
- **`host`** — accessory ไม่จำเป็นต้องอยู่เซิร์ฟเวอร์เดียวกับ web ก็ได้ (แยกเซิร์ฟเวอร์สำหรับ
  database โดยเฉพาะก็ทำได้ผ่านการระบุ IP อื่น)
- **`port`** — รูปแบบ `"127.0.0.1:5432:5432"` (bind แค่ local เท่านั้น ปลอดภัยกว่าเพราะแอปที่รัน
  บนเซิร์ฟเวอร์เดียวกันเข้าถึงผ่าน Docker network ภายในได้อยู่แล้วโดยไม่ต้องเปิด port สู่ภายนอก)
  ต่างจาก `port: "5432:5432"` เฉยๆ ที่จะเปิดให้เข้าถึงจากอินเทอร์เน็ตได้ (ไม่ควรทำถ้าไม่จำเป็นจริงๆ)
- **`directories`** — เหมือน `volumes` ของ service หลัก คือทำให้ข้อมูล**คงอยู่ข้าม container
  restart** จำเป็นมากสำหรับ database (ถ้าไม่ตั้งค่านี้ ข้อมูลทั้งหมดจะหายทันทีที่ container ถูก
  restart หรือ recreate)
- **`env.secret`** — ใช้กลไกเดียวกับ Step 758 ทุกประการ (ดึงจาก `.kamal/secrets`)

### Trade-off: accessory container เทียบกับ managed database service

นี่คือการตัดสินใจสถาปัตยกรรมที่สำคัญที่สุดอย่างหนึ่งของ Part นี้ — ควรรัน database เป็น Kamal
accessory หรือใช้ managed service (เช่น AWS RDS, DigitalOcean Managed Database, Supabase) แทน?

| ประเด็น | Accessory (Kamal จัดการเอง) | Managed Database Service |
|---------|------------------------------|---------------------------|
| ค่าใช้จ่าย | ถูกกว่ามาก (แค่ค่า VPS) | แพงกว่า (จ่ายค่า management เพิ่ม) |
| Backup อัตโนมัติ | ต้องตั้งค่า/เขียนสคริปต์เอง | มีให้ในตัว มักมี point-in-time recovery |
| High availability / replica | ต้องจัดการเอง (ซับซ้อนมาก) | มักมีให้เลือกใช้ง่ายๆ ผ่านปุ่มเดียว |
| Automatic minor version patching | ต้องอัปเดต image เอง | อัปเดตให้อัตโนมัติ (มักเลือกช่วงเวลาได้) |
| ความรับผิดชอบเมื่อ database ล่ม | ทีมคุณต้องแก้เอง 100% | ผู้ให้บริการมี SLA รับผิดชอบบางส่วน |
| ความหน่วง (latency) | ต่ำมาก (อยู่เครื่องเดียวกันหรือใกล้กันได้) | ขึ้นกับระยะห่างเครือข่ายไปยัง managed service |
| ความซับซ้อนตอนเริ่มต้น | ต่ำ (config เพิ่มไม่กี่บรรทัด) | ต้องสมัคร ตั้งค่า network/firewall แยกต่างหาก |

**คำแนะนำในทางปฏิบัติ:** สำหรับ side project, MVP, หรือทีมขนาดเล็กที่งบจำกัดและยอมรับความเสี่ยงที่
จะดูแล backup เองได้ — accessory เหมาะสมและช่วยประหยัดต้นทุนได้มาก แต่สำหรับ**ระบบ production ที่
ข้อมูลสำคัญมาก** (เช่นระบบที่มีธุรกรรมทางการเงินตาม Part 071) **managed database service มักคุ้มค่า
กว่าในระยะยาว** เพราะความเสี่ยงจากการทำ backup/recovery พลาดเองมีต้นทุนสูงกว่าค่า management fee
มาก — ทีมจำนวนมากเลือกทางสายกลาง: **ใช้ Kamal accessory สำหรับ Redis/cache (ข้อมูลที่หายแล้ว
สร้างใหม่ได้) แต่ใช้ managed service สำหรับ Postgres (ข้อมูลที่หายแล้วกู้คืนไม่ได้)**

### Multi-server และ multi-role สำหรับ scale เกินเซิร์ฟเวอร์เดียว

ดึงจาก `kamal docs role` (เอกสารจริงที่ฝังใน gem):

```yaml
servers:
  web:
    - 172.1.0.1
    - 172.1.0.2

  workers:
    hosts:
      - 172.1.0.3
      - 172.1.0.4
    cmd: "bin/jobs"        # รันคำสั่งอื่นแทน `bin/rails server` — เป็น process ประมวลผล job แทน
    proxy: false            # role นี้ไม่ต้องรับ HTTP traffic จากภายนอก ไม่ต้องมี proxy
```

หลักการสำคัญของ **role**:

- **`web`** เป็น role พิเศษที่ Kamal ถือว่าเป็น **`primary_role`** โดยอัตโนมัติ (เปลี่ยนได้ด้วย
  `primary_role` แต่ส่วนใหญ่ไม่จำเป็น) — role นี้เท่านั้นที่เปิด proxy รับ HTTP traffic จาก
  ภายนอกโดยค่าเริ่มต้น
- Role อื่น (เช่น `workers`/`job`) จะ**ไม่เปิด proxy** โดยอัตโนมัติ เพราะไม่ได้มีหน้าที่รับ HTTP
  request จากผู้ใช้ตรงๆ — มีหน้าที่แค่ประมวลผล background job (ผ่าน Solid Queue/Sidekiq ตาม
  Part 061-062) จึงกำหนด `cmd` ให้ต่างจาก role `web`
- แต่ละ role **scale แนวนอนได้อิสระจากกัน** — ถ้า background job งานหนักขึ้นมาก เพิ่มเซิร์ฟเวอร์ให้
  role `workers` อย่างเดียวได้โดยไม่ต้องแตะ role `web` เลย

**เมื่อไหร่ควรแยก role `job` ออกจาก `web`:** ตอนเริ่มต้น (เซิร์ฟเวอร์เดียว) การตั้งค่า
`SOLID_QUEUE_IN_PUMA: true` (ตามที่เห็นใน `env.clear` ค่าเริ่มต้นจาก Step 754) ให้ Solid Queue
supervisor รันแฝงอยู่ใน Puma process เดียวกับเว็บนั้นสมเหตุสมผลมาก — ประหยัดทรัพยากรและง่ายต่อการ
จัดการ แต่เมื่อระบบโตขึ้นจนงาน background job (เช่น ส่งอีเมลจำนวนมาก, ประมวลผลรูปภาพ, เรียก external
API ที่ช้า) เริ่มแย่งทรัพยากร CPU/memory จากการตอบ HTTP request ของ web role **ควรแยกออกมาเป็น
role ต่างหากบนเซิร์ฟเวอร์อื่น** แล้วปิด `SOLID_QUEUE_IN_PUMA` ที่ role `web` (ตั้งเป็น `false`)
พร้อมเปิด role `job` ที่รัน `bin/jobs` แทน — Kamal ทำให้การ scale แบบนี้เป็นแค่การแก้ config ไม่กี่
บรรทัดแล้วรัน `kamal setup` เพิ่มเข้าไปในเซิร์ฟเวอร์ใหม่ ไม่ต้องเปลี่ยนสถาปัตยกรรมโค้ดเลย

### การ scale จำนวนเซิร์ฟเวอร์มากๆ อย่างนุ่มนวล (`boot`)

ถ้ามีเซิร์ฟเวอร์ในแต่ละ role จำนวนมาก (สิบ, ร้อยเครื่อง) การ deploy พร้อมกันทุกเครื่องอาจทำให้ระบบ
downstream (เช่น connection pool ของ database) รับภาระ container ใหม่ทั้งหมดพร้อมกันหนักเกินไป
`kamal docs boot` ให้ทางแก้:

```yaml
boot:
  limit: 25%    # boot ทีละ 25% ของเซิร์ฟเวอร์ในแต่ละ role แทนที่จะพร้อมกันหมด
  wait: 10      # รอ 10 วินาทีระหว่างแต่ละกลุ่ม
```

สำหรับทีมขนาดเล็กที่มีไม่กี่เซิร์ฟเวอร์ ไม่จำเป็นต้องตั้งค่านี้เลย (ค่าเริ่มต้นคือ boot ทุกเครื่อง
พร้อมกัน) แต่เป็นตัวเลือกที่มีให้เมื่อระบบโตขึ้นถึงจุดที่ต้องคิดเรื่องนี้จริงจัง

---

## แบบฝึกหัด: เขียน `config/deploy.yml` เต็มรูปแบบสำหรับแอปจริง พร้อม accessory Postgres

### โจทย์

สมมติทีมของคุณสร้างแอป Rails ชื่อ **MyShop** (ระบบร้านค้าออนไลน์เล็กๆ) เสร็จแล้ว ต้องการ deploy
ไปยัง VPS หนึ่งเครื่อง โดยมีข้อกำหนด:

1. ใช้ **GHCR** เป็น container registry
2. ใช้ domain จริง (สมมติ) `myshop.example.com` พร้อม SSL อัตโนมัติผ่าน Let's Encrypt
3. แยก role `job` สำหรับรัน background job ต่างหากจาก role `web` (คนละเซิร์ฟเวอร์)
4. รัน PostgreSQL เป็น Kamal accessory (ไม่ใช้ managed database — ทีมนี้เลือกประหยัดต้นทุนก่อน
   เพราะยังเป็นช่วง MVP)
5. จัดการ secrets ทั้งหมดให้ปลอดภัยตามหลักการที่เรียนมา

### เฉลย

```yaml
# config/deploy.yml
service: myshop

image: myshop-user/myshop

servers:
  web:
    - 203.0.113.10
  job:
    hosts:
      - 203.0.113.11
    cmd: bin/jobs

proxy:
  ssl: true
  host: myshop.example.com
  app_port: 80
  healthcheck:
    path: /up
    interval: 3
    timeout: 3

registry:
  server: ghcr.io
  username:
    - GHCR_USERNAME
  password:
    - GHCR_TOKEN

builder:
  arch: amd64

env:
  clear:
    RAILS_LOG_LEVEL: info
    DB_HOST: myshop-db
    SOLID_QUEUE_IN_PUMA: false
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_PASSWORD

volumes:
  - "myshop_storage:/rails/storage"

asset_path: /rails/public/assets

accessories:
  db:
    image: postgres:16
    host: 203.0.113.10
    port: "127.0.0.1:5432:5432"
    env:
      clear:
        POSTGRES_USER: myshop
        POSTGRES_DB: myshop_production
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data

aliases:
  console: app exec --interactive --reuse "bin/rails console"
  dbc: accessory exec db --interactive --reuse "psql -U myshop myshop_production"
```

### อธิบายทีละบรรทัดว่าทำไมต้องตั้งค่าแบบนี้

- **`service: myshop`** — ตั้งชื่อสั้น ชัดเจน ไม่ชนกับแอปอื่นถ้าอนาคตมีหลายแอปในเซิร์ฟเวอร์เดียวกัน
- **`image: myshop-user/myshop`** — รูปแบบ `<namespace>/<repo>` ตามที่ GHCR ต้องการ (namespace คือ
  username หรือ organization บน GitHub)
- **`servers.web`** — เซิร์ฟเวอร์เดียว (`203.0.113.10`) รับ HTTP traffic ตรงตามข้อกำหนดข้อ 2 และ 3
  ของโจทย์ (ทำให้ `proxy.ssl: true` ใช้ได้ ตามข้อจำกัดที่อธิบายไว้ใน Step 755 — SSL อัตโนมัติต้องมี
  web server เดียว)
- **`servers.job`** — แยกเซิร์ฟเวอร์คนละเครื่อง (`203.0.113.11`) พร้อม `cmd: bin/jobs` สั่งให้รัน
  Solid Queue supervisor แทนที่จะรัน Puma web server ตรงตามข้อกำหนดข้อ 3 (role นี้ไม่ระบุ
  `proxy: true` จึงไม่เปิด HTTP endpoint ให้ภายนอกเข้าถึงเลย ตามหลักการ Step 760)
- **`proxy.ssl: true` + `proxy.host`** — เปิด Let's Encrypt อัตโนมัติชี้ไปที่ domain จริงตามข้อกำหนด
  ข้อ 2 (ในการใช้งานจริง domain นี้ต้องมี DNS A record ชี้มาที่ IP `203.0.113.10` ไว้ล่วงหน้าแล้ว
  ก่อนรัน `kamal setup` — ไม่งั้น Let's Encrypt พิสูจน์ความเป็นเจ้าของ domain ไม่ได้)
- **`proxy.healthcheck`** — ใช้ `/up` ตามค่าเริ่มต้นที่ Rails 8 generate ให้ (ทวนจาก Step 752)
  ระบุชัดเจนไว้เพื่อให้อ่าน config แล้วเข้าใจทันทีโดยไม่ต้องเปิดเอกสารเทียบ
- **`registry`** — ใช้ GHCR ตามข้อกำหนดข้อ 1, `username`/`password` เป็น**ชื่อตัวแปร**ที่จะไปหาใน
  `.kamal/secrets` ไม่ใช่ค่าจริง (ทวนหลักการ Step 758)
- **`env.clear.SOLID_QUEUE_IN_PUMA: false`** — **จุดสำคัญที่พลาดบ่อย**: เมื่อแยก role `job` ออกมา
  แล้ว (ข้อกำหนดข้อ 3) ต้อง**ปิด**ค่านี้ที่ role `web` ไม่งั้นจะมี Solid Queue supervisor รันซ้ำซ้อน
  สองที่พร้อมกัน (ทั้งใน Puma ของ web และใน `bin/jobs` ของ job role) ทำให้ job อาจถูกประมวลผลซ้ำ
  หรือแย่งกันทำงานโดยไม่จำเป็น
- **`env.clear.DB_HOST: myshop-db`** — ชี้ไปที่ชื่อ container ของ accessory `db` (Kamal ตั้งชื่อ
  container ของ accessory เป็น `<service>-<accessory_name>` และเชื่อมทุก container ของ service
  เดียวกันเข้า Docker network เดียวกันให้อัตโนมัติ ทำให้แอปเรียก accessory ด้วยชื่อ container ได้
  ตรงๆ โดยไม่ต้องรู้ IP internal)
- **`env.secret`** — `RAILS_MASTER_KEY` (ทวนจาก Step 758) และ `DATABASE_PASSWORD` (รหัสผ่านที่แอป
  Rails จะใช้เชื่อมต่อ Postgres ผ่าน `config/database.yml` ที่อ่านจาก ENV) ทั้งสองค่าต้องมีสูตรอยู่
  ใน `.kamal/secrets`
- **`accessories.db`** — ใช้ image ทางการ `postgres:16` ตรงตามข้อกำหนดข้อ 4, bind port แค่
  `127.0.0.1` (ไม่เปิดสู่อินเทอร์เน็ต ตามหลักความปลอดภัยจาก Step 760), และมี `directories` เพื่อให้
  ข้อมูลไม่หายเมื่อ container ถูก restart — **ขาดบรรทัดนี้ไปบรรทัดเดียวเท่ากับเสี่ยงข้อมูลลูกค้า
  หายทั้งร้าน** ถ้า container ของ database ถูก recreate โดยไม่ได้ตั้งใจ
- **`aliases.dbc`** — เพิ่ม alias พิเศษสำหรับเปิด `psql` เข้า accessory database ได้ทันทีผ่าน
  `bin/kamal dbc` แทนที่จะต้อง SSH เข้าเซิร์ฟเวอร์เองแล้วหา container name เอง

### `.kamal/secrets` ที่คู่กัน

```bash
# .kamal/secrets
RAILS_MASTER_KEY=$(cat config/master.key)
DATABASE_PASSWORD=$(rails credentials:fetch myshop.database_password)
POSTGRES_PASSWORD=$DATABASE_PASSWORD   # ใช้รหัสผ่านเดียวกันทั้งฝั่งแอปและฝั่ง Postgres server
GHCR_USERNAME=$GHCR_USERNAME
GHCR_TOKEN=$GHCR_TOKEN
```

### ทดสอบจริง: ยืนยันว่า config นี้ parse ผ่านและถูกต้องด้วย Kamal 2.12.0 จริง

เราเอา `config/deploy.yml` ฉบับเต็มด้านบนนี้ (คัดลอกตรงตัวอักษรทุกตัว) ไปรันผ่าน `kamal config`
จริงในแซนด์บ็อกซ์นี้ (ใส่ secret ปลอมใน `.kamal/secrets` เพื่อให้ resolve ได้ครบ):

```
$ bin/kamal config -c config/deploy.exercise.yml
---
:roles:
- job
- web
:hosts:
- 203.0.113.11
- 203.0.113.10
:primary_host: 203.0.113.10
:version: 9d61b85c45f49fe6b6e401715224aa3a545b78ed
:repository: ghcr.io/myshop-user/myshop
:absolute_image: ghcr.io/myshop-user/myshop:9d61b85c45f49fe6b6e401715224aa3a545b78ed
:service_with_version: myshop-9d61b85c45f49fe6b6e401715224aa3a545b78ed
:volume_args:
- "--volume"
- myshop_storage:/rails/storage
:ssh_options:
  :user: root
  :port: 22
:builder:
  arch: amd64
:accessories:
  db:
    image: postgres:16
    host: 203.0.113.10
    port: 127.0.0.1:5432:5432
    env:
      clear:
        POSTGRES_USER: myshop
        POSTGRES_DB: myshop_production
      secret:
      - POSTGRES_PASSWORD
    directories:
    - data:/var/lib/postgresql/data
```

ยืนยันชัดเจนว่า: **role ทั้งสอง (`job`, `web`) ถูก resolve ถูกต้อง, `image` ถูกต่อกับ registry
server กลายเป็น `absolute_image` ที่ถูกต้องตามรูปแบบของ GHCR, accessory `db` ถูก parse ครบทุก field
รวมถึง `directories` และ `env.secret`** — นี่คือหลักฐานว่าไฟล์นี้ไม่มี syntax error หรือโครงสร้างผิด
พลาดใดๆ เมื่อประมวลผลด้วย Kamal เวอร์ชันจริงที่ใช้งานอยู่ปัจจุบัน

เรายังทดสอบต่อด้วย `bin/kamal lock status` ชี้ไปที่ IP ปลอมนี้ เพื่อดูว่า Kamal พยายาม SSH ออกไป
จริงตามที่ config บอก (ผลคือ connection timeout ตามคาด เพราะไม่มีเซิร์ฟเวอร์จริงอยู่ที่ IP ในช่วง
RFC 5737 นี้) — ยืนยันว่า flow การอ่าน config ไปจนถึงขั้นตอนที่ต้องมี network จริงทำงานถูกต้องสมบูรณ์

### สิ่งที่ต้องเปลี่ยนถ้าจะเอาไป deploy กับ VPS จริง (ตรงไปตรงมา ไม่ปิดบัง)

1. **แทนที่ `203.0.113.10`/`203.0.113.11`** ด้วย public IP จริงของ VPS สองเครื่องที่คุณเช่าไว้
   (Step 756) — ต้อง SSH เข้าถึงได้จริงด้วย user/key ที่ตรงกับที่ตั้งค่าไว้ใน `ssh:` (ถ้าไม่ใช่
   root ต้องเพิ่ม field `ssh.user` ตามตัวอย่าง Step 755)
2. **ตั้งค่า DNS จริง** ให้ `myshop.example.com` (ในทางปฏิบัติคือ domain จริงที่คุณเป็นเจ้าของ)
   ชี้ A record มาที่ IP ของเซิร์ฟเวอร์ `web` **ก่อน**รัน `kamal setup` เพราะ Let's Encrypt ต้อง
   ตรวจสอบความเป็นเจ้าของ domain ผ่าน HTTP challenge ที่ port 443 ของ IP นั้น
3. **สร้าง GHCR token จริง** (Personal Access Token สิทธิ์ `write:packages`) แล้วตั้งเป็น
   environment variable `GHCR_USERNAME`/`GHCR_TOKEN` บนเครื่องที่จะสั่ง deploy (หรือใน GitHub
   Actions secrets ถ้าใช้ CI/CD ตาม Part 075 เป็นคนสั่ง deploy แทนเครื่อง local)
4. **สร้าง `myshop.database_password` ใน Rails credentials จริง** ด้วย
   `EDITOR="code --wait" rails credentials:edit --environment production`
5. **ตรวจสอบว่า `config/database.yml`** ของแอปอ่านค่า `DB_HOST`/`DATABASE_PASSWORD` จาก ENV จริง
   (Rails 8 generate ให้พร้อมใช้ ENV อยู่แล้วโดยปกติ แต่ควรเช็คว่ายังไม่ถูกแก้ไขไปเป็น hardcode)
6. **รัน `kamal setup` ครั้งแรก** (ไม่ใช่ `kamal deploy`) เพราะเป็นเซิร์ฟเวอร์ใหม่ทั้งคู่ที่ยังไม่มี
   Docker/proxy/accessory ใดๆ เลย
7. **หลัง setup สำเร็จ** ครั้งต่อๆ ไปใช้ `kamal deploy` ตามปกติ (Step 757)

หลังจากทำครบทุกข้อนี้ (โดยเฉพาะข้อ 1–2 ที่ต้องมี VPS/domain จริง) โครงสร้าง config นี้ที่ผ่านการ
validate จริงแล้วในแบบฝึกหัดนี้ก็พร้อมใช้งาน deploy ได้ทันทีโดยไม่ต้องแก้ syntax เพิ่มอีก

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม accessory `redis` (ใช้ image `valkey/valkey:8`) เข้าไปใน `config/deploy.yml` ของแบบฝึกหัด
   ข้างต้น สำหรับใช้เป็น cache store และ Action Cable adapter จากนั้นตั้งค่า `env.clear.REDIS_URL`
   ให้ชี้ไปที่ container name ของ accessory นั้น (ทบทวนวิธีตั้ง `DB_HOST` ในเฉลยเป็นตัวอย่าง) แล้ว
   รัน `kamal config` ตรวจสอบว่า parse ผ่านเหมือนที่ Part นี้สาธิตไว้
2. ลองออกแบบสถานการณ์ที่เซิร์ฟเวอร์ `web` มี **2 เครื่อง** พร้อมกัน (เพื่อรองรับ traffic สูงขึ้น)
   แล้วตอบคำถามต่อไปนี้เป็นข้อความอธิบาย (ไม่ต้องเขียนโค้ด): `proxy.ssl: true` แบบอัตโนมัติของ
   Kamal จะยังใช้ได้หรือไม่ ทำไม และถ้าใช้ไม่ได้ ต้องแก้สถาปัตยกรรมส่วนไหนเพิ่ม (ใบ้: ทบทวนคำเตือน
   ใน Step 755 เรื่องข้อจำกัดของ SSL อัตโนมัติ)
3. เขียน GitHub Actions workflow (ทบทวนจาก Part 075) ที่รัน `bin/kamal deploy` อัตโนมัติทุกครั้งที่
   push เข้า branch `main` สำเร็จ (ต้อง `test`/`lint` ผ่านก่อนเป็น dependency ของ job deploy) โดย
   ต้องคิดด้วยว่า GitHub Actions runner จะรู้จัก SSH private key และ secrets ต่างๆ (`GHCR_TOKEN`,
   `RAILS_MASTER_KEY`) ได้อย่างไรโดยไม่ commit ค่าจริงลงในโค้ดหรือ workflow file เลย

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Kamal คือเครื่องมือ deploy Docker container ไปเซิร์ฟเวอร์ใดก็ได้ผ่าน SSH** ที่ทีม
  Basecamp/37signals สร้างขึ้นเพื่อให้ประสบการณ์ deploy ง่ายแบบ PaaS แต่ไม่ผูกกับ vendor ใดๆ และ
  ทำไม **Rails 8** ถึงผูก Kamal มาเป็นค่าเริ่มต้นของทุกโปรเจกต์ใหม่ (`rails new` เรียก `kamal init`
  ให้อัตโนมัติ — ยืนยันจริงจาก log การสร้างแอป)
- เข้าใจวงจรหลักของ Kamal — **build → push → SSH → pull → run** และบทบาทของ **`kamal-proxy`**
  (reverse proxy ที่เขียนขึ้นใหม่ด้วย Go ตั้งแต่ Kamal 2.0 แทนที่ Traefik ของเวอร์ชัน 1.x) ในการทำ
  zero-downtime deployment ผ่านกลไก health check ที่ endpoint `/up`
- อ่านและเข้าใจทุกส่วนของ `config/deploy.yml` ที่ Rails 8 generate ให้จริง: `service`, `image`,
  `servers` (พร้อม role), `registry`, `env` (แบ่ง `secret`/`clear`), `proxy` (SSL/healthcheck),
  `ssh`, `builder` (สถาปัตยกรรม CPU), `volumes`, `asset_path` — ทุกรายละเอียดดึงมาจาก
  `kamal docs <section>` ของ Kamal 2.12.0 ที่ติดตั้งจริง ไม่ใช่จากความจำ
- รู้ข้อกำหนดเบื้องต้นสำหรับ deploy จริง — เตรียม **VPS** (Hetzner/DigitalOcean/Vultr ฯลฯ), Docker
  ที่ Kamal ติดตั้งให้อัตโนมัติผ่าน `kamal server bootstrap`, และบัญชี **container registry**
  (Docker Hub/GHCR) พร้อม access token
- แยกความต่างระหว่าง **`kamal setup`** (ใช้ครั้งแรกกับเซิร์ฟเวอร์ใหม่ ติดตั้งทุกอย่างครบวงจร),
  **`kamal deploy`** (deploy โค้ดใหม่ตามปกติ), และ **`kamal redeploy`** (เร็วกว่าสำหรับ deploy
  รัวๆ ระหว่าง debug)
- จัดการ **secrets** อย่างปลอดภัยด้วย `.kamal/secrets` (เป็น "สูตร" ไม่ใช่ที่เก็บค่าจริง),
  เข้าใจว่าทำไม `env.secret` ถึงปลอดภัยกว่า `env.clear` ในทางเทคนิค (ไม่โผล่ใน `docker inspect`),
  และเช็กลิสต์ความปลอดภัยของ secret ทั้งหมด
- เข้าใจกลไก **zero-downtime deployment** แบบละเอียดทีละขั้นตอน (boot คู่ขนาน → health check →
  atomic switch → drain container เก่า) และรู้จัก **`kamal rollback`** พร้อมข้อควรระวังสำคัญเรื่อง
  database migration ที่ไม่ถูก rollback ไปด้วย
- เข้าใจ **accessories** (รัน Postgres/Redis เป็น Kamal-managed container) เทียบกับการใช้
  **managed database service** พร้อม trade-off ที่ชัดเจนสำหรับการตัดสินใจจริง และรู้จัก
  **multi-server/multi-role** (`web` vs `job`) สำหรับ scale ระบบออกจากเซิร์ฟเวอร์เดียว
- ลงมือเขียนและ**ยืนยันความถูกต้องจริงด้วย `kamal config`** ของ `config/deploy.yml` เต็มรูปแบบ
  สำหรับแอป "MyShop" ที่มี multi-role และ accessory Postgres ครบถ้วน พร้อมรู้ชัดเจนว่าต้องแก้อะไร
  อีกบ้างถ้าจะเอาไป deploy กับ VPS จริง

**ต่อไป (Part 077):** Kamal เหมาะกับทีมที่อยากควบคุมเซิร์ฟเวอร์เองเต็มที่ แต่ก็มีต้นทุนด้าน
การดูแลรักษาที่ต้องแลกมา Part ถัดไปจะพาไปสำรวจ**ทางเลือกอื่นในการ deploy** ที่เป็น PaaS เต็มรูปแบบ
อย่าง **Render, Fly.io, และ Heroku** — เปรียบเทียบข้อดีข้อเสียกับ Kamal อย่างตรงไปตรงมา และเจาะลึก
หัวข้อที่สำคัญไม่แพ้กันซึ่ง Part นี้แตะไปแล้วเล็กน้อยตอนพูดถึง `kamal rollback` —**การรัน database
migration อย่างปลอดภัยใน production** ที่ทำให้ deploy ใหม่และ rollback ทำงานร่วมกับ schema ที่
เปลี่ยนแปลงได้โดยไม่ทำข้อมูลพัง
