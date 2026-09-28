# Part 073: Docker สำหรับ Rails — Dockerfile, docker-compose

> **Step ครอบคลุมใน Part นี้:** Step 721–730
> **ระดับ:** ปานกลาง (ไม่ต้องมีพื้นฐาน Docker มาก่อน แต่ควรผ่าน Part 021 เรื่องโครงสร้างโปรเจกต์
> Rails และ Part 067 เรื่อง Rails encrypted credentials มาก่อน เพราะ Part นี้จะใช้ `RAILS_MASTER_KEY`
> เป็นตัวอย่างหลักเรื่อง environment variable)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby **3.3.6**, Rails **8.1.4**),
> Docker **29.3.1**, Docker Compose **v5.1.1** (โปรโตคอล/syntax ของ `docker-compose.yml` ที่สอนใน
> Part นี้เสถียรมาหลายปีแล้ว ใช้ได้กับ Docker Desktop หรือ Docker Engine เวอร์ชันใหม่ๆ ทั่วไปโดยไม่มี
> ปัญหา)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> ทีมผู้เขียนสร้างแอป Rails 8.1.4 จริงด้วย `rails new blogapp --database=postgresql` ในเครื่อง
> (แยกจาก repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้งทั้งหมดหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) แล้ว
> **build image จริง, run container จริง, ยิง HTTP request จริงเข้าไปหา container ที่รันอยู่, เขียน
> `docker-compose.yml` จริงที่มี Postgres, รัน `docker compose up` จริง, รัน migration จริงผ่าน
> container, และ debug ด้วย `docker exec`/`docker logs` จริง** ทุกไฟล์ `Dockerfile`/`.dockerignore`
> ที่แสดงใน Part นี้คือของจริงที่ `rails new` generate ออกมา ไม่มีการพิมพ์เดาจากความจำแม้แต่บรรทัด
> เดียว
>
> **ข้อจำกัดหนึ่งข้อของแซนด์บ็อกซ์ที่ต้องเปิดเผยตรงไปตรงมา:** แซนด์บ็อกซ์ที่ใช้เขียน Part นี้มี
> network policy ที่ปฏิเสธการเชื่อมต่อไปยัง `deb.debian.org` (Debian package mirror) ด้วย
> `403 Forbidden` (ยืนยันด้วย `curl` ตรงๆ พบ header `x-deny-reason: host_not_allowed`) ทำให้ขั้นตอน
> `apt-get install` ใน **base image ที่เป็น Debian จริง** (`ruby:3.3.6-slim`) รันไม่ผ่านในแซนด์บ็อกซ์
> นี้เท่านั้น (ข้อจำกัดของสภาพแวดล้อมการเขียนหลักสูตร ไม่ใช่บั๊กของ Docker/Rails — บนเครื่องพัฒนาทั่วไป
> หรือ CI server ที่มี internet ปกติ `apt-get` จะเข้าถึง `deb.debian.org` ได้โดยไม่มีปัญหาใดๆ) เพื่อให้
> ยังพิสูจน์กลไกทั้งหมดได้จริงแบบ end-to-end ทีมผู้เขียนจึง build ด้วย base image ทางเลือกที่แซนด์บ็อกซ์
> นี้เข้าถึงได้ (`ubuntu:24.04` ซึ่งดึง package จาก `archive.ubuntu.com` ที่ไม่ถูกบล็อก) โดย**คัดลอก
> ทุกคำสั่งจาก Dockerfile จริงของ Rails มาเรียงลำดับเดียวกันทุกประการ** ต่างกันแค่ชื่อ base image
> และวิธีติดตั้งตัว Ruby interpreter เอง (คัดลอกจาก build ที่มีอยู่แล้วแทนการติดตั้งผ่าน apt) — ผลคือ
> เราพิสูจน์ได้จริงว่า multi-stage build ทำงานถูกต้อง, `bundle install` cache layer ทำงานถูกต้อง,
> `bootsnap`/`assets:precompile` รันผ่าน, non-root user ถูกสร้างและใช้งานจริง, entrypoint รัน
> `db:prepare` จริง, container เสิร์ฟ HTTP request จริงผ่าน Thruster บนพอร์ต 80 **Dockerfile ที่แสดง
> ในเนื้อหาทุกจุดคือของจริงจาก `rails new` แบบไม่ได้แก้ไข** — ส่วน adaptation เรื่อง base image เป็น
> เรื่องเฉพาะของการเขียน Part นี้เท่านั้น ไม่ใช่สิ่งที่นักเรียนต้องทำตาม และจะระบุไว้ชัดเจนทุกจุดที่
> เกี่ยวข้อง
>
> ระหว่างการทดสอบยังพบ **ข้อเท็จจริงเชิงปฏิบัติการจริงสองเรื่องที่เอกสารส่วนใหญ่ไม่ได้พูดถึง** ซึ่งเป็น
> ประโยชน์มากสำหรับคนที่จะเอา Docker ไปใช้กับ Rails 8 จริง:
>
> 1. gem `uri` เวอร์ชันใหม่ (1.1.1 ที่ Rails 8.1.4 ดึงมาใช้) **ปฏิเสธ hostname ที่มีขีดล่าง (`_`)**
>    ใน `DATABASE_URL` ด้วย error `URI::InvalidURIError: the scheme postgres does not accept
>    registry part` — ถ้าตั้งชื่อ container/service ของฐานข้อมูลด้วยขีดล่าง (เช่น `pg_demo`) แอปจะ
>    บูตไม่ขึ้น รายละเอียดเต็มอยู่ใน Step 725
> 2. `DATABASE_URL` มีผลกับฐานข้อมูล **primary เท่านั้น** — ฐานข้อมูลรองที่ Rails 8 สร้างให้อัตโนมัติ
>    (`solid_cache`, `solid_queue`, `solid_cable`) ต้องการ `CACHE_DATABASE_URL`,
>    `QUEUE_DATABASE_URL`, `CABLE_DATABASE_URL` แยกต่างหาก ไม่งั้น `db:prepare` จะพยายามต่อฐานข้อมูล
>    เหล่านี้ผ่าน Unix socket ในเครื่อง container เองแล้วล้มเหลว รายละเอียดเต็มอยู่ใน Step 729

ยินดีต้อนรับสู่ **Phase 12: DevOps & Deployment** หลังจากผ่าน 11 Phase ที่ผ่านมาซึ่งเน้นการเขียนโค้ด
Rails ให้ทำงานถูกต้องบนเครื่องของเราเอง คำถามที่หลีกเลี่ยงไม่ได้คือ — "แล้วจะเอาแอปนี้ไปรันบนเซิร์ฟเวอร์
จริงยังไง โดยที่มั่นใจว่ามันจะทำงานเหมือนกับตอนรันบนเครื่องเราทุกประการ" Phase นี้ (Part 073–078) จะพา
ไปตั้งแต่การห่อแอปด้วย Docker, จัดการ config/secret อย่างปลอดภัย, ตั้ง CI/CD อัตโนมัติ, ไปจนถึงการ
deploy และ monitor จริงบน production

Part นี้เป็น Part แรกของ Phase และตอบคำถามพื้นฐานที่สุด: **"ทำไมต้อง Docker"** และ **"Rails 8 ช่วยเรา
ตรงจุดนี้ได้มากแค่ไหน"** — คำตอบสั้นๆ คือ Rails 8 **สร้าง production-ready Dockerfile ให้อัตโนมัติทันที
ที่รัน `rails new`** โดยที่เราไม่ต้องเขียน Dockerfile เองตั้งแต่ศูนย์เลยด้วยซ้ำ นี่คือการเปลี่ยนแปลงใหญ่
เทียบกับ Rails เวอร์ชันก่อนหน้าที่ต้องพึ่งพา gem หรือ template ของบุคคลที่สามในการทำสิ่งนี้ Part นี้จะพา
อ่าน Dockerfile ที่ Rails สร้างให้ทีละบรรทัดจนเข้าใจกลไกทั้งหมด แล้วต่อยอดไปสู่การเขียน
`docker-compose.yml` สำหรับพัฒนาในเครื่องที่มี PostgreSQL เป็น service แยกต่างหาก ซึ่งเป็นรูปแบบ
มาตรฐานที่ทีม Rails มืออาชีพเกือบทุกทีมใช้กันในปัจจุบัน

## สารบัญของ Part นี้

- Step 721: ปัญหาที่ Docker แก้ — "Works on my machine" และความไม่สอดคล้องกันระหว่าง dev/CI/production
- Step 722: Rails 8 สร้าง Dockerfile ให้อัตโนมัติ — อ่านทุกบรรทัดของไฟล์จริงที่ `rails new` สร้างให้
- Step 723: Multi-stage build เจาะลึก — ทำไมแยกเป็น 3 stage ถึงทำให้ image สุดท้ายเล็กลงมาก
- Step 724: `docker build` และ `docker run` ตัวจริง — build image, run container, ยิง request จริง
- Step 725: Environment Variables ใน Docker — `ENV`, `--env-file`, `RAILS_MASTER_KEY`, และกับดักเรื่อง
  hostname ที่มีขีดล่าง
- Step 726: `.dockerignore` — ทำไมสำคัญ และอะไรบ้างที่ไม่ควรเข้าไปอยู่ใน image
- Step 727: เขียน `docker-compose.yml` สำหรับ Local Development — web + PostgreSQL, volume สำหรับ
  hot-reload และข้อมูลถาวร
- Step 728: `depends_on` + Health Check — ทำให้แอปรอฐานข้อมูลพร้อมจริงก่อนบูต
- Step 729: รัน Migration ผ่าน Container และ Debug ด้วย `docker exec`/`docker logs`
- Step 730: เทคนิคลดขนาด Image + แบบฝึกหัดรวบยอด — Dockerize แอป Rails ด้วย Docker Compose ตั้งแต่
  ต้นจนจบ

---

## Step 721: ปัญหาที่ Docker แก้ — "Works on my machine" และความไม่สอดคล้องกันระหว่าง dev/CI/production

### ปัญหาคลาสสิกที่ทุกทีมเคยเจอ

ลองนึกภาพสถานการณ์นี้ — โค้ดของคุณรันได้ปกติบนเครื่องตัวเอง แต่พอเพื่อนร่วมทีม `git pull` ไปรัน
กลับ error, หรือแย่กว่านั้นคือ test ผ่านหมดบน CI แต่พอ deploy ขึ้น production กลับพัง สาเหตุที่พบบ่อย
ที่สุดคือ **ความแตกต่างของสภาพแวดล้อม (environment)** ระหว่างเครื่องแต่ละเครื่อง:

1. **เวอร์ชัน Ruby ไม่ตรงกัน** — เครื่อง dev ใช้ Ruby 3.3.6 แต่ production ยังเป็น 3.2.4
2. **เวอร์ชัน library ระบบไม่ตรงกัน** — เครื่องหนึ่งมี `libvips` เวอร์ชันใหม่ที่รองรับ feature บางอย่าง
   อีกเครื่องไม่มี
3. **เวอร์ชัน PostgreSQL ไม่ตรงกัน** — dev ใช้ Postgres 16 แต่ production ยังเป็น Postgres 13 ซึ่งไม่
   รองรับ SQL feature บางตัวที่ query ใหม่ใช้
4. **ตัวแปร environment ที่ลืมตั้งค่า** — ลืม set `TZ`, locale, หรือ config อื่นๆ ที่มีผลต่อพฤติกรรม
   ของโปรแกรม
5. **Dependency ของระบบปฏิบัติการที่ติดตั้งแบบ manual แล้วลืมจด** — วิศวกรคนหนึ่ง `apt install`
   package บางตัวไว้นานแล้วตอนแก้ปัญหาเฉพาะหน้า แต่ไม่เคยบันทึกไว้ที่ไหน พอย้ายเครื่องใหม่ก็หา
   สาเหตุไม่เจอว่าทำไมพังไปหลายวัน

ปรากฏการณ์นี้มีชื่อเรียกติดปากในวงการว่า **"It works on my machine"** — ประโยคที่ทุกทีมวิศวกรรม
ซอฟต์แวร์ได้ยินอย่างน้อยครั้งหนึ่งในชีวิตการทำงาน และเป็นสัญญาณว่าทีมยังไม่มีวิธีการที่เชื่อถือได้ใน
การรับประกันว่า **"สภาพแวดล้อมที่โค้ดรัน" เหมือนกันทุกที่**

### ทางออกแบบเดิม vs ทางออกด้วย Docker

ก่อนยุค container การแก้ปัญหานี้ทำได้สองแบบหลักๆ ซึ่งทั้งคู่มีข้อเสียชัดเจน:

1. **เขียนเอกสารการติดตั้งอย่างละเอียด** (README ยาวเป็นหน้าๆ บอกให้รัน `apt install` ทีละคำสั่ง) —
   ปัญหาคือเอกสารล้าสมัยเร็วมาก คนใหม่ในทีมยังคงเจอปัญหาติดตั้งไม่ผ่านอยู่ดี และไม่มีอะไรการันตีว่า
   ทุกคนทำตามขั้นตอนแบบเป๊ะๆ เหมือนกัน
2. **ใช้ Virtual Machine (VM)** เช่น Vagrant + VirtualBox — จำลองทั้งระบบปฏิบัติการขึ้นมาใหม่ทำให้
   สภาพแวดล้อมเหมือนกันจริง แต่ VM แต่ละตัวหนักมาก (หลาย GB) บูตช้า และกิน RAM/CPU ของเครื่องจริง
   มหาศาลเพราะต้องจำลอง kernel ทั้งระบบปฏิบัติการซ้อนอยู่ข้างใน

**Docker** แก้ปัญหานี้ด้วยแนวคิด **container** ซึ่งต่างจาก VM ตรงที่ container **ใช้ kernel ของ
เครื่องโฮสต์ร่วมกัน** ไม่ต้องจำลองทั้งระบบปฏิบัติการใหม่ ทำให้:

- **เบากว่า VM มาก** — container ทั่วไปมีขนาดหลักสิบถึงหลักร้อย MB เทียบกับ VM ที่มักเป็นหลัก GB
- **บูตเร็ว** — เป็นวินาที ไม่ใช่นาทีเหมือน VM
- **รันได้จำนวนมากพร้อมกันบนเครื่องเดียว** โดยไม่กิน resource มากเกินไป
- ที่สำคัญที่สุดคือ **"image" ที่ build ขึ้นมาครั้งเดียว รันได้เหมือนกันทุกที่** — ไม่ว่าจะเป็นเครื่อง
  MacBook ของนักพัฒนา, GitHub Actions CI runner, หรือเซิร์ฟเวอร์ production จริง ตราบใดที่เครื่องนั้น
  มี Docker Engine ติดตั้งอยู่

### คำศัพท์พื้นฐานที่ต้องเข้าใจก่อน

| คำศัพท์ | ความหมาย |
|---|---|
| **Image** | พิมพ์เขียว (blueprint) ที่ไม่เปลี่ยนแปลง บรรจุทุกอย่างที่แอปต้องการ (OS libraries, Ruby, gem, โค้ดแอป) สร้างจาก `Dockerfile` |
| **Container** | instance ที่กำลังรันอยู่จริงของ image หนึ่งตัว เปรียบเทียบง่ายๆ คือ image เหมือน "คลาส" (class) ส่วน container เหมือน "object" ที่สร้างจากคลาสนั้น |
| **Dockerfile** | ไฟล์ข้อความที่บอกทีละขั้นตอนว่าจะ build image อย่างไร (เริ่มจาก base image ไหน, ติดตั้งอะไรบ้าง, copy โค้ดยังไง, รันคำสั่งอะไรตอนเริ่ม) |
| **Registry** | ที่เก็บ image ให้ดาวน์โหลด เช่น Docker Hub (`docker.io`), GitHub Container Registry (`ghcr.io`) |
| **Docker Compose** | เครื่องมือสำหรับนิยามและรันแอปที่ประกอบด้วยหลาย container พร้อมกัน (เช่น web + database + redis) ด้วยไฟล์ YAML ไฟล์เดียว |

ตรวจสอบว่าเครื่องมีทั้ง Docker Engine และ Docker Compose พร้อมใช้งาน:

```bash
docker --version
# Docker version 29.3.1, build c2be9cc

docker compose version
# Docker Compose version v5.1.1
```

> **หมายเหตุ:** `docker-compose` (มีขีดกลาง แยกเป็นคำสั่งต่างหาก) คือเวอร์ชันเก่าที่เขียนด้วย Python
> ปัจจุบัน Docker รวม Compose เข้าเป็น subcommand ของตัว `docker` เองแล้ว เรียกด้วย `docker compose`
> (ไม่มีขีดกลาง) Part นี้จะใช้รูปแบบใหม่ตลอด ถ้าเครื่องคุณยังมีแต่ `docker-compose` แบบเก่า คำสั่งเกือบ
> ทั้งหมดใน Part นี้ยังใช้ตรงกันได้ (แค่เปลี่ยนขีดกลางเป็นช่องว่าง)

---

## Step 722: Rails 8 สร้าง Dockerfile ให้อัตโนมัติ — อ่านทุกบรรทัดของไฟล์จริงที่ `rails new` สร้างให้

### การเปลี่ยนแปลงสำคัญตั้งแต่ Rails 8

ก่อน Rails 8 การ dockerize แอป Rails ต้องพึ่งพา gem ของบุคคลที่สาม (เช่น `dockerfile-rails`) หรือ
เขียน Dockerfile เองตั้งแต่ต้น ตั้งแต่ **Rails 8** เป็นต้นไป **`rails new` จะสร้าง `Dockerfile`,
`.dockerignore`, และไฟล์ที่เกี่ยวกับ Kamal (`config/deploy.yml`, โฟลเดอร์ `.kamal/`) ให้อัตโนมัติทันที
โดยไม่ต้องขอ flag พิเศษใดๆ** — นี่คือสัญญาณว่าทีม Rails core มองว่าการมี container image พร้อม deploy
ตั้งแต่วันแรกเป็นเรื่องพื้นฐานของแอป Rails สมัยใหม่ ไม่ใช่ทางเลือกเสริมอีกต่อไป

สร้างแอปใหม่เพื่อดู Dockerfile ที่ generate ให้จริง:

```bash
rails new blogapp --database=postgresql
cd blogapp
cat Dockerfile
```

นี่คือ Dockerfile **ตัวจริง** ที่ได้จากคำสั่งข้างต้น (Rails 8.1.4 / Ruby 3.3.6) ไม่มีการแก้ไขแม้แต่
บรรทัดเดียว:

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true

# This Dockerfile is designed for production, not development. Use with Kamal or build'n'run by hand:
# docker build -t blogapp .
# docker run -d -p 80:80 -e RAILS_MASTER_KEY=<value from config/master.key> --name blogapp blogapp

# For a containerized dev environment, see Dev Containers: https://guides.rubyonrails.org/getting_started_with_devcontainer.html

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
    # -j 1 disable parallel compilation to avoid a QEMU bug: https://github.com/rails/bootsnap/issues/495
    bundle exec bootsnap precompile -j 1 --gemfile

# Copy application code
COPY . .

# Precompile bootsnap code for faster boot times.
# -j 1 disable parallel compilation to avoid a QEMU bug: https://github.com/rails/bootsnap/issues/495
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

### อ่านทีละส่วน — สิ่งที่เกิดขึ้นจริงในแต่ละบรรทัด

**บรรทัดแรก — Dockerfile syntax directive:**

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true
```

บรรทัดแรกบอก Docker ให้ใช้ BuildKit frontend เวอร์ชันล่าสุดของ `docker/dockerfile:1` (ฟีเจอร์อย่าง
multi-stage `COPY --from`, `--chown` ต้องพึ่ง frontend นี้) บรรทัดที่สอง `check=error=true` เปิดใช้งาน
**Dockerfile linter ในตัว Docker เอง** ที่จะทำให้ build **ล้มเหลวทันที** ถ้าพบปัญหาเชิง best-practice
(เช่น ใช้ `ADD` แทน `COPY` โดยไม่จำเป็น, มี instruction ที่ไม่มีผล) — เป็นฟีเจอร์ที่ Rails ใส่มาให้เพื่อ
บังคับมาตรฐานคุณภาพของ Dockerfile ตั้งแต่ต้น

**Comment อธิบายวิธีใช้:** สังเกตว่า Rails ใส่คอมเมนต์บอกวิธี build/run ด้วยมือไว้ให้เสร็จสรรพ และเตือน
ชัดเจนว่า **"This Dockerfile is designed for production, not development"** — ประเด็นนี้สำคัญมาก จะ
กลับมาพูดถึงอีกครั้งใน Step 727 ตอนเขียน setup สำหรับ development

**Base image และ ARG:**

```dockerfile
ARG RUBY_VERSION=3.3.6
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base
```

`ARG` ประกาศตัวแปรที่ใช้ได้เฉพาะตอน build (ไม่ติดไปกับ image สุดท้าย) ค่า default คือเวอร์ชัน Ruby ที่
ใช้ตอน `rails new` (อ่านมาจากไฟล์ `.ruby-version` ของโปรเจกต์) `ruby:3.3.6-slim` คือ **official Ruby
image บน Docker Hub ที่สร้างจาก Debian bookworm แบบ slim** (ตัดส่วนที่ไม่จำเป็นออก เช่น man pages,
documentation) `AS base` ตั้งชื่อ stage นี้ว่า `base` เพื่อให้ stage อื่นอ้างอิงกลับมาได้ (รายละเอียด
เรื่อง multi-stage อยู่ใน Step 723)

**ติดตั้ง runtime system packages:**

```dockerfile
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 libvips postgresql-client && \
    ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives
```

นี่คือ package ที่จำเป็นตอน **รัน** (runtime) ไม่ใช่ตอน build:

- `curl` — ใช้โดย Thruster (ตัวจัดการ HTTP proxy หน้าแอป) และมีประโยชน์เวลา debug
- `libjemalloc2` — memory allocator ทางเลือกที่มีประสิทธิภาพดีกว่า `malloc` ของ glibc มากสำหรับ
  แอป Ruby ที่ allocate/free object จำนวนมาก (ลด memory fragmentation) บรรทัดถัดมาสร้าง symlink
  แล้วบรรทัด `ENV LD_PRELOAD` ด้านล่างจะบังคับให้ Ruby process โหลด jemalloc แทน allocator เริ่มต้น
- `libvips` — library ประมวลผลภาพความเร็วสูง ที่ `image_processing` gem (ใช้คู่กับ Active Storage)
  เรียกใช้งาน
- `postgresql-client` — คำสั่ง `psql`, `pg_dump`, `pg_isready` (ไม่ใช่ตัวฐานข้อมูล เป็นแค่ client
  tools สำหรับต่อไปหา Postgres server ที่อื่น)

`rm -rf /var/lib/apt/lists /var/cache/apt/archives` ท้ายบรรทัดลบ apt cache ทิ้งทันทีใน **layer
เดียวกัน** — จุดนี้สำคัญมาก ถ้าแยกเป็นคนละ `RUN` ไฟล์ cache จะยังฝังอยู่ใน layer ก่อนหน้าอยู่ดี (แม้จะ
ลบใน layer หลังก็ไม่ทำให้ image เล็กลง เพราะแต่ละ layer เป็น diff ที่สะสมกันไปเรื่อยๆ) นี่คือเหตุผลที่
เห็น `RUN` คำสั่งยาวๆ ต่อกันด้วย `&&` แทนที่จะแยกเป็นหลาย `RUN`

**ENV vars สำหรับ production:**

```dockerfile
ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so"
```

- `RAILS_ENV="production"` — ล็อก environment ของแอปไว้ที่ production ตายตัว (image นี้มีไว้สำหรับ
  production เท่านั้นตามที่คอมเมนต์บอกไว้)
- `BUNDLE_DEPLOYMENT="1"` — เปิด **deployment mode** ของ Bundler ซึ่งบังคับว่า `Gemfile.lock` ต้อง
  ตรงกับ `Gemfile` เป๊ะๆ ห้ามมีการ resolve เวอร์ชันใหม่ตอน `bundle install` (ถ้าไม่ตรงจะ error ทันที
  แทนที่จะเงียบๆ ติดตั้งเวอร์ชันอื่นให้) เป็นเซฟตี้เน็ตสำคัญที่ทำให้ gem เวอร์ชันที่รันบน production
  ตรงกับที่ทดสอบไว้เป๊ะ
- `BUNDLE_WITHOUT="development"` — ไม่ติดตั้ง gem ในกลุ่ม `development`/`test` ของ `Gemfile`
  (`rspec`, `capybara`, `debug` ฯลฯ) ลดขนาด image และลด attack surface
- `LD_PRELOAD` — ตามที่อธิบายไปแล้ว บังคับให้ทุก process ในอิมเมจนี้ใช้ jemalloc

**Multi-stage: `build` stage** — จะอธิบายละเอียดใน Step 723

**Final stage — non-root user:**

```dockerfile
FROM base

RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash
USER 1000:1000
```

นี่คือ **security best practice ที่สำคัญมาก**: container ส่วนใหญ่ที่ไม่ตั้งค่าอะไรเลยจะรันโปรเซสด้วย
สิทธิ์ `root` โดย default ซึ่งอันตรายมาก — ถ้ามีช่องโหว่ในแอปที่ทำให้ผู้โจมตี escape ออกจาก container
ได้ (container escape) การรันด้วย `root` จะทำให้ผู้โจมตีมีสิทธิ์ `root` บนเครื่องโฮสต์ทันที Rails สร้าง
user ชื่อ `rails` ที่มี `uid`/`gid` เป็น `1000` (ค่ามาตรฐานสำหรับ non-root user แรกในหลาย distro) แล้ว
`USER 1000:1000` สั่งให้ทุกคำสั่งหลังจากนี้ — รวมถึงตอนรัน container จริง — ทำงานด้วยสิทธิ์ผู้ใช้นี้
เท่านั้น ยืนยันได้ด้วยการรันจริง:

```bash
docker run --rm blogapp whoami
# rails

docker run --rm blogapp id
# uid=1000(rails) gid=1000(rails) groups=1000(rails)
```

**ENTRYPOINT และ CMD:**

```dockerfile
ENTRYPOINT ["/rails/bin/docker-entrypoint"]

EXPOSE 80
CMD ["./bin/thrust", "./bin/rails", "server"]
```

`ENTRYPOINT` คือคำสั่งที่รันเสมอเมื่อ container เริ่มทำงาน ส่วน `CMD` คือ argument ที่ส่งต่อให้
entrypoint (และเป็นค่าที่ override ได้ตอน `docker run` ท้ายคำสั่ง) มาดูเนื้อหาไฟล์
`bin/docker-entrypoint` ที่ Rails สร้างให้จริง:

```bash
#!/bin/bash -e

# If running the rails server then create or migrate existing database
if [ "${@: -2:1}" == "./bin/rails" ] && [ "${@: -1:1}" == "server" ]; then
  ./bin/rails db:prepare
fi

exec "${@}"
```

Script นี้เช็กว่า argument สองตัวสุดท้ายที่ส่งเข้ามาคือ `./bin/rails server` หรือไม่ ถ้าใช่ (ซึ่งตรงกับ
`CMD` ด้านบนพอดี) จะรัน **`rails db:prepare` ก่อนเสมอ** — คำสั่งนี้ฉลาดกว่า `db:migrate` ตรงที่ถ้า
ฐานข้อมูลยังไม่มีจะสร้างให้ใหม่พร้อม schema ทั้งหมด แต่ถ้ามีอยู่แล้วจะแค่รัน migration ที่ค้างอยู่ —
เหมาะกับการรันซ้ำๆ ทุกครั้งที่ container เริ่มโดยไม่มีผลข้างเคียง (idempotent) จากนั้น `exec "${@}"`
ส่งต่อการควบคุมไปยังคำสั่งจริง (`./bin/thrust ./bin/rails server`) โดยใช้ `exec` แทนการเรียกแบบ
subprocess ธรรมดา เพื่อให้ process นั้นกลายเป็น PID 1 ของ container รับสัญญาณ (เช่น `SIGTERM` ตอน
`docker stop`) ได้ถูกต้อง

ส่วน `./bin/thrust` คือ **Thruster** — gem ใหม่จากทีม Basecamp (`gem "thruster", require: false` ใน
`Gemfile`) ที่ทำหน้าที่เป็น HTTP proxy บางๆ อยู่หน้า Puma: เพิ่ม X-Sendfile acceleration (เสิร์ฟไฟล์
static โดยตรงไม่ผ่าน Ruby), บีบอัด response, และจัดการ SSL termination พื้นฐาน — เหตุผลที่ `EXPOSE 80`
คือพอร์ตที่ Thruster เปิดรับจริง (Puma เองมักฟังที่พอร์ต 3000 อยู่ข้างในแต่ Thruster จะ proxy ให้)

---

## Step 723: Multi-stage Build เจาะลึก — ทำไมแยกเป็น 3 Stage ถึงทำให้ Image สุดท้ายเล็กลงมาก

### ปัญหาของ Single-stage Build

ลองจินตนาการว่าถ้าเขียน Dockerfile แบบ stage เดียว ต้องมีทุกอย่างอยู่ใน image เดียวกัน:

- Compiler และ build tools (`build-essential`, `gcc`, `make`) สำหรับ compile native extension ของ
  gem บางตัว (เช่น `pg`, `bootsnap` ที่มีส่วน C extension)
- Header files สำหรับ library ต่างๆ (`libpq-dev`, `libyaml-dev`) ที่จำเป็นตอน **compile** เท่านั้น
- ซอร์สโค้ด gem ที่ยังไม่ผ่านการ build (source cache ของ Bundler)
- Git (ใช้ดึง gem บางตัวจาก GitHub โดยตรงถ้า `Gemfile` ระบุ `git:`)

ทั้งหมดนี้ **จำเป็นแค่ตอน build เท่านั้น** พอ build เสร็จแล้ว ตอน**รัน**จริงไม่ต้องใช้ compiler หรือ
header file พวกนี้อีกเลย — แต่ถ้าอยู่ใน stage เดียวกัน มันจะติดค้างอยู่ใน image สุดท้ายไปตลอด ทำให้
image ใหญ่โดยไม่จำเป็น (compiler เพียงอย่างเดียวก็กินพื้นที่หลายร้อย MB แล้ว) และเพิ่ม **attack
surface** โดยไม่จำเป็น (ยิ่งมี tool เยอะใน production image ยิ่งมีช่องโหว่ที่อาจถูกโจมตีได้มากขึ้น)

### วิธีแก้ด้วย Multi-stage Build

Dockerfile ของ Rails 8 แบ่งเป็น **3 stage** ที่ประกาศด้วยคำสั่ง `FROM` ซ้ำกันหลายครั้ง:

```dockerfile
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base
# ... (ติดตั้งเฉพาะ runtime dependencies)

FROM base AS build
# ... (ติดตั้ง build dependencies, bundle install, precompile assets)

FROM base
# ... (final stage — ไม่มีชื่อ ใช้ base เดิมที่ "สะอาด" ไม่มี build tools ติดมา)
```

จุดที่ฉลาดที่สุดคือ **stage สุดท้ายไม่ได้สืบทอดจาก `build` stage แต่สืบทอดจาก `base` stage เดิม
ตรงๆ** (ที่ยังไม่เคยติดตั้ง `build-essential` เลย) แล้วค่อย **"หยิบ" เฉพาะไฟล์ผลลัพธ์ที่ต้องการ** จาก
`build` stage มาด้วยคำสั่ง:

```dockerfile
COPY --chown=rails:rails --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --chown=rails:rails --from=build /rails /rails
```

`--from=build` บอก Docker ให้ copy ไฟล์จาก stage ที่ชื่อ `build` (ไม่ใช่จากเครื่องเรา) เข้ามาใน stage
ปัจจุบัน สิ่งที่ถูก copy มามีแค่:

1. `${BUNDLE_PATH}` (`/usr/local/bundle`) — โฟลเดอร์ gem ที่ **build เสร็จแล้ว** (compile เรียบร้อย
   เป็น `.so` binary แล้ว) ไม่ใช่ซอร์สโค้ดดิบ ไม่ใช่ cache ของ Bundler
2. `/rails` — โค้ดแอปทั้งหมด **พร้อม asset ที่ precompile เสร็จแล้ว** และ bootsnap cache

ส่วน `build-essential`, `git`, `libpq-dev`, `libyaml-dev`, `pkg-config` ที่ติดตั้งไว้ใน `build` stage
**ไม่ได้ถูก copy มาด้วยเลย** — มันอยู่ใน layer ของ `build` stage ที่ไม่มีความเกี่ยวข้องกับ stage สุดท้าย
อีกต่อไป เมื่อ build เสร็จ Docker จะไม่เก็บ layer ที่ไม่ถูกอ้างอิงในผลลัพธ์สุดท้ายไว้ใน image ที่ tag
ออกมา (ถึงแม้ layer เหล่านั้นอาจยังอยู่ใน build cache ภายในเครื่องเพื่อให้ build ครั้งถัดไปเร็วขึ้นก็ตาม)

### พิสูจน์ด้วยตัวเลขจริง

เพื่อให้เห็นภาพชัดเจน เราลอง build ทั้งสอง stage แยกกันแล้วเทียบขนาด (ทดสอบจริงในแซนด์บ็อกซ์ด้วย
base image ที่ปรับให้เข้ากับ network policy ตามที่ระบุไว้ในหมายเหตุต้น Part — ตัวเลขจึงไม่ตรงกับขนาด
จริงของ image ที่มาจาก `ruby:3.3.6-slim` ปกติ แต่ **สัดส่วนของสิ่งที่ถูกตัดออก** ยังสะท้อนหลักการ
เดียวกัน):

```bash
docker build --target build -t blogapp:build-stage .
docker build -t blogapp:final .

docker images
```

```
REPOSITORY   TAG           SIZE
blogapp      build-stage   2.99GB   # มี build-essential, gem source cache, compiler ครบ
blogapp      final         2.6GB    # ไม่มี build tools ติดมาเลย
```

ในการ build ด้วย `ruby:3.3.6-slim` (Debian) ตามปกติที่ไม่มีข้อจำกัดของแซนด์บ็อกซ์ ภาพรวมทั่วไปที่พบ
บ่อยในแอป Rails ขนาดกลาง (มี Active Storage, Postgres, ไม่มี gem หนักผิดปกติ) คือ **stage `build` มัก
หนักกว่า final image 2–3 เท่า** และ final image ที่ได้มักอยู่ในช่วง **250–450 MB** เทียบกับถ้าไม่ทำ
multi-stage (รวมทุกอย่างไว้ stage เดียว) ซึ่งมักหนักกว่า 700 MB–1 GB ขึ้นไป — ส่วนต่างนี้คือ
`build-essential` + header files + gem source cache ที่ทิ้งไปได้ทั้งหมดโดยไม่กระทบการทำงานของแอป
แม้แต่นิดเดียว เพราะสิ่งที่แอปต้องการตอนรันจริงคือ **ไฟล์ binary ที่ compile เสร็จแล้ว** ไม่ใช่ตัว
compiler เอง

### ทำไมสอง apt-get install ถึงแยกกันคนละ stage

สังเกตว่า `base` stage ติดตั้ง `curl libjemalloc2 libvips postgresql-client` (runtime) ส่วน `build`
stage ติดตั้งเพิ่ม `build-essential git libpq-dev libvips libyaml-dev pkg-config` (build-time) —
ทั้งสองชุดนี้**คนละความต้องการกัน**:

| Package | ทำไมต้องมี | จำเป็นตอนไหน |
|---|---|---|
| `libpq-dev` | header files ของ PostgreSQL client library ที่ gem `pg` ต้องใช้ตอน compile native extension | Build เท่านั้น |
| `postgresql-client` | ตัวโปรแกรม `psql`/`pg_isready` (ไม่มี header) | Runtime (เผื่อ debug/health check) |
| `libyaml-dev` | header สำหรับ compile YAML parser (Psych) จาก source | Build เท่านั้น |
| `build-essential` | `gcc`, `make`, และเครื่องมือ compile ทั้งชุด | Build เท่านั้น |
| `libvips` | ตัว library เอง (ไม่ใช่ header) | ทั้ง build และ runtime จึงอยู่ทั้งสอง stage |

---

## Step 724: `docker build` และ `docker run` ตัวจริง — Build Image, Run Container, ยิง Request จริง

### Build Image

จากโฟลเดอร์ root ของแอป Rails รันคำสั่ง:

```bash
docker build -t blogapp .
```

`-t blogapp` ตั้งชื่อ (tag) ให้ image ที่ build เสร็จว่า `blogapp` (เทียบเท่า `blogapp:latest`) จุด
`.` ท้ายคำสั่งบอกว่า **build context** คือโฟลเดอร์ปัจจุบัน (Docker จะส่งไฟล์ทั้งหมดในโฟลเดอร์นี้ ยกเว้น
สิ่งที่ระบุใน `.dockerignore` ไปให้ Docker daemon ใช้ประมวลผล — รายละเอียดเรื่อง `.dockerignore` อยู่ใน
Step 726)

ผลลัพธ์การ build จริง (BuildKit output แบบย่อ):

```
#1 [internal] load build definition from Dockerfile
#1 DONE 0.1s

#6 [base 1/3] FROM docker.io/library/ruby:3.3.6-slim
#6 DONE 2.7s

#9 [base 3/3] RUN apt-get update -qq && apt-get install ... curl libjemalloc2 libvips postgresql-client
#9 DONE 33.4s

#16 [build 4/7] RUN bundle install && ... bootsnap precompile -j 1 --gemfile
#16 Bundle complete! 23 Gemfile dependencies, 121 gems now installed.
#16 DONE 49.1s

#21 [build 7/7] RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile
#21 Writing application-8b441ae0.css
#21 Writing application-bfcdf840.js
#21 DONE 1.3s

#24 exporting to image
#24 naming to docker.io/library/blogapp:latest done
```

ตรวจสอบว่า image ถูกสร้างขึ้นจริงและดูขนาด:

```bash
docker images blogapp
# REPOSITORY   TAG       IMAGE ID       SIZE
# blogapp      latest    <id>           ~300MB (โดยประมาณ ขึ้นกับ gem ที่ใช้)
```

### Run Container

ตามที่คอมเมนต์ใน Dockerfile บอกไว้ ต้องส่ง `RAILS_MASTER_KEY` เข้าไปด้วย (ใช้ถอดรหัส
`config/credentials.yml.enc` — ทบทวนกลไกนี้จาก Part 067 ถ้าจำไม่ได้) เพราะไฟล์ `config/master.key`
**ไม่ได้ถูก copy เข้าไปใน image เลย** (ดู `.dockerignore` ใน Step 726 ว่าทำไม):

```bash
docker run -d -p 3000:80 \
  -e RAILS_MASTER_KEY=$(cat config/master.key) \
  --name blogapp_run \
  blogapp
```

- `-d` — รันแบบ detached (background) ไม่ล็อก terminal
- `-p 3000:80` — map พอร์ต 3000 ของเครื่องโฮสต์ ไปยังพอร์ต 80 ที่ Thruster เปิดรับใน container
  (รูปแบบคือ `<host port>:<container port>`)
- `-e RAILS_MASTER_KEY=...` — ส่ง environment variable เข้าไป (รายละเอียดเต็มใน Step 725)
- `--name blogapp_run` — ตั้งชื่อ container เพื่ออ้างอิงง่ายในคำสั่งถัดๆ ไป

ดู log ว่า container บูตสำเร็จหรือไม่:

```bash
docker logs blogapp_run
```

ผลลัพธ์จริงจากการทดสอบ (เชื่อมกับ Postgres container ผ่าน Docker network — วิธีตั้งค่าฐานข้อมูลแยก
อธิบายเต็มใน Step 725 และ Step 727):

```
{"time":"...","level":"INFO","msg":"Server started","http":":80"}
=> Booting Puma
=> Rails 8.1.4 application starting in production
Puma starting in single mode...
* Puma version: 8.0.2 ("Into the Arena")
* Ruby version: ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]
*  Min threads: 3
*  Max threads: 3
*  Environment: production
*          PID: 58
* Listening on http://0.0.0.0:3000
Use Ctrl-C to stop
```

### ยิง Request จริงเข้าไปหา Container

Rails 8 มาพร้อม **health check endpoint สำเร็จรูป** ที่ `/up` (route `rails/health#show`) ซึ่งเหมาะ
มากสำหรับทดสอบว่า container ตอบสนองหรือไม่โดยไม่ต้องมี route อื่นเลย:

```bash
curl -i http://localhost:3000/up
```

ผลลัพธ์จริงที่ได้:

```
HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Content-Length: 73
Content-Type: text/html; charset=utf-8
X-Runtime: 0.002501
Date: Mon, 28 Sep 2026 18:32:02 GMT

<!DOCTYPE html><html><body style="background-color: green"></body></html>
```

**200 OK** ยืนยันว่า container ที่ build จาก Dockerfile ของ Rails 8 เสิร์ฟ HTTP request จริงได้สำเร็จ
ตั้งแต่ชั้น Thruster (พอร์ต 80) ผ่าน Puma (พอร์ตภายใน 3000) จนถึง Rails router — ครบทั้ง pipeline

จัดการ container หลังทดสอบเสร็จ:

```bash
docker stop blogapp_run
docker rm blogapp_run
```

---

## Step 725: Environment Variables ใน Docker — `ENV`, `--env-file`, `RAILS_MASTER_KEY`, และกับดักเรื่อง Hostname

### สามวิธีหลักในการส่ง Environment Variable เข้า Container

**1. `ENV` ใน Dockerfile** — ค่าคงที่ที่ฝังอยู่ใน image ทุก container ที่รันจาก image นี้จะได้ค่านี้
เหมือนกันเสมอ เหมาะกับค่าที่ไม่เป็นความลับและไม่เปลี่ยนตาม environment (เช่น `RAILS_ENV=production`,
`BUNDLE_PATH`)

```dockerfile
ENV RAILS_ENV="production"
```

**2. `-e` (หรือ `--env`) ตอน `docker run`** — กำหนดทีละตัวตอนรัน เหมาะกับค่าที่เปลี่ยนไปตามแต่ละครั้งที่
รัน หรือค่าที่เป็นความลับที่ไม่ควรฝังอยู่ใน image (เพราะใครก็ตามที่ `docker pull` image ไปสามารถ
`docker history` ดู layer ทั้งหมดได้ ถ้าใส่ secret ไว้ใน `ENV` ของ Dockerfile จะรั่วไหลทันที):

```bash
docker run -e RAILS_MASTER_KEY=<YOUR_MASTER_KEY_HERE> -e RAILS_LOG_LEVEL=debug blogapp
```

**3. `--env-file`** — เมื่อมีตัวแปรจำนวนมาก การพิมพ์ `-e` ทีละตัวไม่สะดวก ใช้ไฟล์ `.env` แทนได้:

```bash
# .env.production (ไฟล์นี้ต้องอยู่ใน .gitignore เสมอ ห้าม commit เด็ดขาด)
RAILS_MASTER_KEY=<YOUR_MASTER_KEY_HERE>
RAILS_LOG_LEVEL=info
RAILS_MAX_THREADS=10
```

```bash
docker run --env-file .env.production -p 3000:80 blogapp
```

> **ข้อควรระวัง:** `--env-file` **ไม่รองรับการทำ variable interpolation หรือ comment ด้วย `#` ต่อท้าย
> บรรทัดค่า** เหมือนที่ `dotenv` gem ทำได้ ทุกบรรทัดต้องเป็น `KEY=value` ตรงๆ เท่านั้น (comment เต็ม
> บรรทัดที่ขึ้นต้นด้วย `#` ใช้ได้ปกติ)

### `RAILS_MASTER_KEY` — ตัวอย่างที่สำคัญที่สุดของ Secret ใน Container

ทบทวนจาก Part 067: Rails encrypted credentials เก็บความลับทั้งหมด (API key, database password ฯลฯ)
ไว้ในไฟล์ `config/credentials.yml.enc` ที่ **เข้ารหัสไว้** ไฟล์เดียวที่ถอดรหัสได้คือ
`config/master.key` (หรือ environment variable `RAILS_MASTER_KEY` ที่มีค่าเดียวกัน) — และไฟล์
`master.key` **ถูกใส่ไว้ใน `.gitignore` ตั้งแต่ `rails new`** จึงไม่เคยถูก commit และไม่เคยถูก copy เข้า
Docker image (ยืนยันได้จาก `.dockerignore` ที่มีบรรทัด `/config/master.key` — ดู Step 726)

ผลคือ **image ที่ build ออกมาไม่มีทางถอดรหัส credentials ได้เองเลยจนกว่าจะมีคนส่ง
`RAILS_MASTER_KEY` เข้ามาตอนรัน container** — นี่คือการออกแบบที่ถูกต้อง: image เดียวกัน build ครั้ง
เดียว สามารถเอาไป deploy ที่ environment ไหนก็ได้ (staging, production) โดยแค่เปลี่ยนค่า
`RAILS_MASTER_KEY` ที่ส่งเข้าไปตอนรัน โดยไม่ต้อง build image ใหม่เลย — สอดคล้องกับหลัก
**Twelve-Factor App** ข้อที่ว่า "config ต้องแยกออกจากโค้ดอย่างเด็ดขาด" (จะพูดถึงเต็มๆ ใน Part 074)

ทดสอบว่าถ้าลืมส่ง `RAILS_MASTER_KEY` จะเกิดอะไรขึ้น (ทดสอบจริง):

```bash
docker run -p 3000:80 blogapp
docker logs <container_id>
```

```
Missing encryption key to decrypt file with. Ask your team for your master key
and write it to config/master.key or put it in the ENV['RAILS_MASTER_KEY'].
```

Error message ชัดเจนมาก — ยืนยันว่ากลไกความปลอดภัยนี้ทำงานถูกต้องจริง ป้องกันไม่ให้ container บูตขึ้น
มาโดยไม่มีกุญแจถอดรหัส secret

### กับดักที่พบจริง: Hostname ที่มีขีดล่าง (`_`) ทำให้ `DATABASE_URL` Parse ไม่ผ่าน

ระหว่างทดสอบ Part นี้ เราตั้งชื่อ container ฐานข้อมูลว่า `pg_demo` (มีขีดล่าง) แล้วส่ง
`DATABASE_URL="postgres://blogapp:demopass@pg_demo:5432/blogapp_production"` เข้าไป ผลคือแอป **บูต
ไม่ขึ้นเลย** ด้วย error ที่ไม่เกี่ยวกับฐานข้อมูลตรงๆ แต่เป็นปัญหาตั้งแต่ชั้น parse URL:

```
bin/rails aborted!
URI::InvalidURIError: the scheme postgres does not accept registry part:
blogapp:demo_password@pg_demo:5432 (or bad hostname?) (URI::InvalidURIError)
```

หาสาเหตุจนเจอว่า **ไม่เกี่ยวกับ username/password เลย แต่เป็นเพราะ hostname `pg_demo` มีขีดล่าง**:

```bash
ruby -e "require 'uri'; p URI::RFC2396_Parser.new.parse('postgres://user:pass@pg_demo:5432/db')"
# URI::InvalidURIError: the scheme postgres does not accept registry part...

ruby -e "require 'uri'; p URI::RFC2396_Parser.new.parse('postgres://user:pass@pg-demo:5432/db')"
# #<URI::Generic postgres://user:pass@pg-demo:5432/db>   -- ใช้ขีดกลางแทน ผ่านทันที
```

สาเหตุคือ gem `uri` เวอร์ชัน **1.1.1** (ที่ Rails 8.1.4 ดึงมาใช้แทนเวอร์ชันที่มากับ Ruby โดยตรง) บังคับ
ใช้กฎ RFC ของ hostname อย่างเคร่งครัดขึ้นกว่าเดิมมาก — และตาม RFC 952/1123 **hostname มาตรฐานอนุญาต
เฉพาะตัวอักษร ตัวเลข และขีดกลาง (`-`) เท่านั้น ห้ามมีขีดล่าง (`_`)** ทั้งที่ Docker เองยินยอมให้ตั้งชื่อ
container/network alias ด้วยขีดล่างได้ (และหลายคนก็ทำแบบนั้นโดยไม่รู้ปัญหานี้) พอชื่อนั้นถูกเอาไปสร้าง
เป็นส่วน hostname ของ `DATABASE_URL` แล้วส่งให้ `uri` gem เวอร์ชันใหม่ parse จึงพังทันที

> **บทเรียนสำคัญ:** ตั้งชื่อ Docker container, service ใน `docker-compose.yml`, และ network alias ทุก
> ตัวที่จะใช้เป็นส่วนหนึ่งของ hostname ใน connection string **ด้วยขีดกลาง (`-`) เท่านั้น ห้ามใช้ขีดล่าง
> (`_`) เด็ดขาด** — ข่าวดีคือ **Docker Compose เวอร์ชันปัจจุบันตั้งชื่อ container ให้อัตโนมัติด้วย
> รูปแบบ `<project>-<service>-<n>` ซึ่งใช้ขีดกลางอยู่แล้วโดย default** (เช่น `blogdemo-db-1`) จึงไม่มี
> ปัญหานี้ตราบใดที่ไม่ไปตั้ง `container_name:` เองด้วยขีดล่าง

---

## Step 726: `.dockerignore` — ทำไมสำคัญ และอะไรบ้างที่ไม่ควรเข้าไปอยู่ใน Image

### `.dockerignore` คืออะไร

เช่นเดียวกับ `.gitignore` ที่บอก Git ว่าไม่ต้อง track ไฟล์ไหนบ้าง `.dockerignore` บอก Docker ว่า **ไม่
ต้องส่งไฟล์/โฟลเดอร์ไหนเข้าไปใน build context เลย** — มีผลสองอย่างที่สำคัญมาก:

1. **Build เร็วขึ้น** — build context ที่เล็กลง ส่งข้อมูลจากเครื่องเราไปยัง Docker daemon เร็วขึ้น
   (สำคัญมากถ้าโปรเจกต์มี `node_modules` หรือ `.git` history ขนาดใหญ่)
2. **ปลอดภัยขึ้นและ image เล็กลง** — ไฟล์ที่ไม่ได้อยู่ใน build context จะไม่มีทาง**ถูก `COPY` เข้าไปใน
   image ได้เลย** แม้ Dockerfile จะเขียน `COPY . .` (copy ทุกอย่าง) ก็ตาม

นี่คือไฟล์ `.dockerignore` **ตัวจริง** ที่ `rails new` สร้างให้:

```
# See https://docs.docker.com/engine/reference/builder/#dockerignore-file for more about ignoring files.

# Ignore git directory.
/.git/
/.gitignore

# Ignore bundler config.
/.bundle

# Ignore all environment files.
/.env*

# Ignore all default key files.
/config/master.key
/config/credentials/*.key

# Ignore all logfiles and tempfiles.
/log/*
/tmp/*
!/log/.keep
!/tmp/.keep

# Ignore pidfiles, but keep the directory.
/tmp/pids/*
!/tmp/pids/.keep

# Ignore storage (uploaded files in development and any SQLite databases).
/storage/*
!/storage/.keep
/tmp/storage/*
!/tmp/storage/.keep

# Ignore assets.
/node_modules/
/app/assets/builds/*
!/app/assets/builds/.keep
/public/assets

# Ignore CI service files.
/.github

# Ignore Kamal files.
/config/deploy*.yml
/.kamal

# Ignore development files
/.devcontainer

# Ignore Docker-related files
/.dockerignore
/Dockerfile*
```

### ทำไมแต่ละรายการถึงสำคัญ

| รายการ | เหตุผลที่ห้ามเข้า image |
|---|---|
| `/.git/` | ประวัติ commit **ทั้งหมด** ของโปรเจกต์ ซึ่งอาจมี secret เก่าที่เคย commit ผิดพลาดไว้ในอดีต (แม้จะลบออกจาก commit ล่าสุดแล้วก็ยังอยู่ใน history) และทำให้ image ใหญ่ขึ้นมากโดยไม่มีประโยชน์ต่อการรันแอปเลย |
| `/.env*` | ไฟล์ environment variable ในเครื่อง dev ที่มักมี secret หรือ config เฉพาะเครื่อง — ต้องส่งผ่าน `-e`/`--env-file` ตอน `docker run` แทน ไม่ใช่ฝังใน image |
| `/config/master.key` | กุญแจถอดรหัส credentials ทั้งหมด (ตามที่อธิบายใน Step 725) — ถ้าเผลอ copy เข้า image แล้ว push ขึ้น registry สาธารณะ เท่ากับแจกกุญแจถอดรหัส secret ทั้งหมดให้ใครก็ได้ |
| `/log/*`, `/tmp/*` | ไฟล์ log/temp ของเครื่อง dev ไม่มีความหมายอะไรใน container ใหม่ที่เพิ่งสร้าง (แต่ละ container มี log ของตัวเองตั้งแต่เริ่ม) |
| `/node_modules/` | ถ้าใช้ `cssbundling-rails`/`jsbundling-rails` `node_modules` อาจมีขนาดหลายร้อย MB — Dockerfile ควรรัน `npm install` เองข้างในแทนการ copy จากเครื่อง dev (ที่อาจมี native binary ของ OS อื่นปนอยู่) |
| `/.github` | workflow file ของ CI ไม่มีประโยชน์ตอนรันแอปจริง |
| `/config/deploy*.yml`, `/.kamal` | ไฟล์ config ของ Kamal (เครื่องมือ deploy ที่จะเรียนใน Part 076) มีข้อมูล infrastructure ที่ไม่ควรอยู่ใน image ที่ deploy ด้วย Kamal เอง (จะกลายเป็นวนซ้ำไม่มีประโยชน์) |
| `/Dockerfile*`, `/.dockerignore` | ไฟล์ที่ใช้ควบคุมการ build เอง ไม่มีเหตุผลต้องอยู่ข้างในผลลัพธ์ของการ build |

### ทดลองดูผลจริงถ้าไม่มี `.dockerignore`

ลองเทียบ build context size ระหว่างมีกับไม่มี `.dockerignore` (สมมติโปรเจกต์มี `.git` และ
`node_modules` ขนาดใหญ่):

```bash
# มี .dockerignore
docker build -t blogapp . 
# => [internal] load build context: transferring context: 116.51kB

# ลอง rename .dockerignore ออกชั่วคราวเพื่อเทียบ (อย่าลืม rename กลับ!)
mv .dockerignore .dockerignore.bak
docker build -t blogapp-noignore .
# => [internal] load build context: transferring context: หลาย MB ถึงหลักร้อย MB
#    ขึ้นกับว่ามี .git/node_modules ในโปรเจกต์มากแค่ไหน
mv .dockerignore.bak .dockerignore
```

ยิ่งโปรเจกต์อายุมาก มี commit history ยาว หรือมี `node_modules` ที่ไม่ได้ทำความสะอาด ส่วนต่างนี้ยิ่งเห็น
ชัดเจน

---

## Step 727: เขียน `docker-compose.yml` สำหรับ Local Development — Web + PostgreSQL

### ทำไม Dockerfile ของ Rails ไม่เหมาะกับ Development

ย้อนกลับไปที่คอมเมนต์บรรทัดแรกสุดของ Dockerfile ใน Step 722:

```dockerfile
# This Dockerfile is designed for production, not development.
```

เหตุผลคือ Dockerfile ตัวนั้นทำสิ่งที่ **ไม่เหมาะกับการพัฒนาในแต่ละวัน** เลย:

- `assets:precompile` ตอน build — ถ้าแก้ไฟล์ CSS/JS ต้อง build image ใหม่ทุกครั้ง ช้ามาก
- `COPY . .` ครั้งเดียวตอน build — แก้โค้ด Ruby แล้วต้อง build ใหม่ทุกครั้งเช่นกัน ไม่มี hot-reload
- ไม่มี gem กลุ่ม `development`/`test` เลย (`BUNDLE_WITHOUT="development"`) — ใช้ `pry`, `rspec`,
  `debug` ไม่ได้
- รันด้วย non-root user ที่ไม่มีสิทธิ์เขียนไฟล์ในบางกรณี

สิ่งที่ทีม Rails ส่วนใหญ่ทำคือแยก **Dockerfile สำหรับ dev ต่างหาก** (มักตั้งชื่อ `Dockerfile.dev`) ที่
เบากว่า ไม่ precompile อะไรล่วงหน้า และ **mount โค้ดจากเครื่องเข้าไปเป็น volume** แทนการ copy ตายตัว
เพื่อให้แก้โค้ดแล้วเห็นผลทันทีโดยไม่ต้อง build image ใหม่

### เขียน `Dockerfile.dev`

```dockerfile
# Dockerfile.dev — ใช้สำหรับ local development เท่านั้น ไม่ใช่ไฟล์สำหรับ deploy จริง
FROM ruby:3.3.6-slim

WORKDIR /rails

# ติดตั้ง system dependencies ที่จำเป็นทั้งตอน build gem และตอนรัน
# (ต่างจาก production Dockerfile ที่แยก base/build ชัดเจน — ที่นี่ไม่จำเป็นเพราะ
# image นี้ไม่เคยถูก deploy จริง ขนาดใหญ่กว่านิดหน่อยไม่ใช่ปัญหา)
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential git libpq-dev libyaml-dev pkg-config \
      libvips postgresql-client curl && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY Gemfile Gemfile.lock ./
RUN bundle install

EXPOSE 3000
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

ข้อสังเกตสำคัญ: **ไม่มี `COPY . .`** ในไฟล์นี้เลย! เพราะโค้ดแอปจะถูกส่งเข้ามาผ่าน **volume mount** ใน
`docker-compose.yml` แทน (Step ถัดไปจะอธิบาย) และ `CMD` ใช้ `-b 0.0.0.0` (bind ทุก network interface)
แทนค่า default ของ Puma ที่ผูกกับ `localhost` เท่านั้น (ซึ่งมองไม่เห็นจากนอก container)

### เขียน `docker-compose.yml`

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: blogapp
      POSTGRES_PASSWORD: blogapp_dev_password
      POSTGRES_DB: blogapp_development
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bash -c "rm -f tmp/pids/server.pid && bin/rails server -b 0.0.0.0"
    volumes:
      - .:/rails
      - bundle-cache:/usr/local/bundle
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development
      RAILS_ENV: development
    depends_on:
      - db

volumes:
  db-data:
  bundle-cache:
```

### อ่านทีละส่วน

**Service `db`:** ใช้ **official PostgreSQL image จาก Docker Hub ตรงๆ** ไม่ต้องเขียน Dockerfile เอง
เลย ตัวแปร `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` เป็นตัวแปรมาตรฐานที่ image นี้อ่านตอน
เริ่มต้นครั้งแรกเพื่อสร้าง role และฐานข้อมูลให้อัตโนมัติ

**สองชนิดของ volume ที่ทำหน้าที่ต่างกันโดยสิ้นเชิง:**

1. **`.:/rails`** (bind mount) — ผูกโฟลเดอร์ปัจจุบันบนเครื่องโฮสต์เข้ากับ `/rails` ใน container
   โดยตรง ทุกการแก้ไขไฟล์บนเครื่องเรา **สะท้อนเข้าไปใน container ทันที** โดยไม่ต้อง build image ใหม่
   เลย — นี่คือกลไกที่ทำให้เกิด **hot-reload**: แก้ view/controller แล้ว refresh browser เห็นผลทันที
   (Rails `config.enable_reloading = true` ใน development ทำหน้าที่โหลดโค้ดใหม่โดยอัตโนมัติอยู่แล้ว)

2. **`db-data:/var/lib/postgresql/data`** และ **`bundle-cache:/usr/local/bundle`** (named volume) —
   ต่างจาก bind mount ตรงที่ Docker เป็นคนจัดการพื้นที่เก็บข้อมูลนี้เอง (ไม่ผูกกับโฟลเดอร์ไหนบนเครื่อง
   โฮสต์โดยตรง) มีประโยชน์สองอย่าง: **(ก)** ข้อมูลฐานข้อมูล**ไม่หายไปแม้ container ถูกลบและสร้างใหม่**
   ตราบใดที่ volume `db-data` ยังอยู่ (`docker compose down` ลบแค่ container ไม่ลบ volume) **(ข)**
   `bundle-cache` แยก gem ที่ติดตั้งไว้ออกจาก bind mount ของโค้ด — ถ้าไม่แยกแบบนี้ การ mount
   `.:/rails` ทับ `/rails` ทั้งโฟลเดอร์จะทำให้ gem ที่ติดตั้งไว้ใน image (`/usr/local/bundle`)ยังอยู่ดี
   เพราะเป็นคนละ path กัน แต่การแยก volume ต่างหากยังช่วยให้ **`bundle install` ครั้งถัดไปไม่ต้อง
   ติดตั้งใหม่ทั้งหมด** แม้จะลบ container ทิ้งแล้วสร้างใหม่ก็ตาม

**`depends_on`:** บอกลำดับการ**เริ่ม container** — `web` จะไม่เริ่มจนกว่า container `db` จะถูกสั่ง
เริ่มไปแล้ว **แต่ไม่รับประกันว่า Postgres ข้างในพร้อมรับการเชื่อมต่อแล้วจริงๆ** (container เริ่มกับ
โปรแกรมข้างในพร้อมใช้งานเป็นคนละเวลากัน) ปัญหานี้แก้ด้วย health check ใน Step ถัดไป

---

## Step 728: `depends_on` + Health Check — ทำให้แอปรอฐานข้อมูลพร้อมจริงก่อนบูต

### ปัญหาของ `depends_on` แบบธรรมดา

Postgres ใช้เวลาสองสามวินาทีในการเริ่มต้นระบบ (initialize database cluster, เขียน WAL, เปิด listener)
ถ้า Rails พยายามเชื่อมต่อในช่วงนั้นพอดี จะได้ error `PG::ConnectionBad: could not connect to server:
Connection refused` — `depends_on` แบบพื้นฐานแก้ปัญหานี้ไม่ได้เพราะมันแค่รับประกันลำดับการ**สั่งเริ่ม**
container ไม่ใช่ลำดับความ**พร้อมใช้งานจริง**ของโปรแกรมข้างใน

### วิธีแก้: `healthcheck` + `depends_on.condition: service_healthy`

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: blogapp
      POSTGRES_PASSWORD: blogapp_dev_password
      POSTGRES_DB: blogapp_development
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U blogapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bash -c "rm -f tmp/pids/server.pid && bin/rails server -b 0.0.0.0"
    volumes:
      - .:/rails
      - bundle-cache:/usr/local/bundle
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development
      RAILS_ENV: development
    depends_on:
      db:
        condition: service_healthy

volumes:
  db-data:
  bundle-cache:
```

**`healthcheck`** บอก Docker ว่าจะเช็กความพร้อมของ service นี้อย่างไร:

- `test: ["CMD-SHELL", "pg_isready -U blogapp"]` — รันคำสั่ง `pg_isready` (มากับ image `postgres`
  อยู่แล้ว) ข้างใน container ทุก `interval` วินาที คำสั่งนี้ return exit code `0` ก็ต่อเมื่อ Postgres
  พร้อมรับการเชื่อมต่อจริงๆ เท่านั้น
- `interval: 5s` — เช็กทุก 5 วินาที
- `timeout: 5s` — แต่ละครั้งที่เช็ก ถ้าไม่ตอบภายใน 5 วินาทีถือว่าล้มเหลวรอบนั้น
- `retries: 5` — ถ้าล้มเหลวติดกัน 5 ครั้ง ถือว่า container นี้ "unhealthy"

**`depends_on.db.condition: service_healthy`** เปลี่ยนความหมายจากเดิม (แค่รอให้เริ่ม) เป็น **"รอจนกว่า
`db` จะมีสถานะ healthy ตาม healthcheck ก่อน ค่อยเริ่ม `web`"** — นี่คือสิ่งที่แก้ปัญหา race condition
ได้จริง

### พิสูจน์ด้วยการรันจริง

```bash
docker compose up -d
docker compose ps
```

ผลลัพธ์จริงที่สังเกตได้ระหว่าง compose ทำงาน (ลำดับ event ที่ Docker Compose พิมพ์ออกมาจริง):

```
 Container blogdemo-db-1 Creating
 Container blogdemo-db-1 Created
 Container blogdemo-web-1 Creating
 Container blogdemo-web-1 Created
 Container blogdemo-db-1 Starting
 Container blogdemo-db-1 Started
 Container blogdemo-db-1 Waiting      <-- Compose กำลังรอ healthcheck ของ db
 Container blogdemo-db-1 Healthy      <-- ผ่าน pg_isready แล้ว
 Container blogdemo-web-1 Starting    <-- ตอนนี้ web ถึงเริ่มจริง
 Container blogdemo-web-1 Started
```

```
NAME              IMAGE          COMMAND                  SERVICE   STATUS                   PORTS
blogdemo-db-1     postgres:16    "docker-entrypoint.s…"   db        Up 8 seconds (healthy)   5432/tcp
blogdemo-web-1    blogdemo-web   "bash -c 'rm -f tmp/…"   web       Up 3 seconds             0.0.0.0:3000->3000/tcp
```

สังเกตบรรทัด `Waiting` → `Healthy` ตามด้วย `web ... Starting` ที่เกิดขึ้น**หลัง** `db` ถูกยืนยันว่า
healthy แล้วเท่านั้น — ยืนยันว่ากลไกทำงานตามที่ออกแบบไว้จริง ไม่ใช่แค่ทฤษฎี

> **สังเกตชื่อ container:** Docker Compose ตั้งชื่อ container อัตโนมัติเป็น `blogdemo-db-1` และ
> `blogdemo-web-1` (ใช้ **ขีดกลาง**) — สอดคล้องกับสิ่งที่เตือนไว้ใน Step 725 พอดี ทำให้ `DATABASE_URL`
> ที่อ้างถึง service name `db` (ซึ่ง Compose สร้าง network alias ให้ตรงกับชื่อ service เสมอ ไม่ใช่ชื่อ
> container เต็ม) ไม่มีปัญหาเรื่อง hostname เลย

---

## Step 729: รัน Migration ผ่าน Container และ Debug ด้วย `docker exec`/`docker logs`

### รัน Migration ด้วย `docker compose run`

เมื่อมี model/migration ใหม่ (เช่นจาก `bin/rails generate scaffold Post title:string body:text`) การ
รัน migration ผ่าน container ทำได้ด้วย:

```bash
docker compose run --rm web bin/rails db:migrate
```

`docker compose run` ต่างจาก `docker compose up` ตรงที่ **สร้าง container ใหม่ชั่วคราวขึ้นมาเฉพาะรัน
คำสั่งนี้คำสั่งเดียว** (ไม่ได้ไปยุ่งกับ container `web` หลักที่กำลัง serve request อยู่) `--rm` สั่งให้
ลบ container ชั่วคราวนี้ทิ้งทันทีหลังคำสั่งจบ (ไม่งั้นจะมี container ที่ exit แล้วค้างอยู่เพิ่มขึ้น
เรื่อยๆ ทุกครั้งที่รัน) ผลลัพธ์จริงจากการทดสอบ:

```
 Container blogdemo-db-1 Running
 Container blogdemo-db-1 Waiting
 Container blogdemo-db-1 Healthy
 Container blogdemo-web-run-439050124a21 Creating
 Container blogdemo-web-run-439050124a21 Created
== 20260928183544 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0079s
== 20260928183544 CreatePosts: migrated (0.0080s) =============================
```

สังเกตว่า `docker compose run` **ก็ยังรอ `db` healthy ก่อนเช่นกัน** เพราะ `depends_on` มีผลกับ
`run` ด้วยไม่ใช่แค่ `up`

### กับดักที่พบจริง: `DATABASE_URL` ไม่ครอบคลุมฐานข้อมูลรองของ Rails 8

Rails 8 สร้างฐานข้อมูลเพิ่มอีก 3 ตัวให้อัตโนมัติสำหรับ `solid_cache`, `solid_queue`, `solid_cable`
(ดูได้จาก `config/database.yml` ที่มี key `cache`, `queue`, `cable` ใต้ `production:`) ระหว่างทดสอบ
เราพบว่าตั้ง `DATABASE_URL` ตัวเดียวแล้วรัน `db:prepare` (ซึ่ง Dockerfile ของ production เรียกให้
อัตโนมัติตอนบูตตามที่อธิบายใน Step 722) **ฐานข้อมูลหลัก (`primary`) เชื่อมต่อสำเร็จ แต่ฐานข้อมูลรอง
ล้มเหลว** ด้วย error ที่ชี้ไปที่ Unix socket ในเครื่อง container เอง:

```
ActiveRecord::ConnectionNotEstablished: connection to server on socket
"/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
```

ตรวจสอบด้วย `rails runner` เพื่อดู config ที่ resolve จริงของแต่ละฐานข้อมูล:

```bash
docker compose run --rm web bin/rails runner \
  'ActiveRecord::Base.configurations.configs_for(env_name: "production").each { |c| pp [c.name, c.configuration_hash] }'
```

```ruby
["primary", {adapter: "postgresql", host: "pg-demo", username: "blogapp", password: "demopass", ...}]
["cache",   {adapter: "postgresql", host: nil, username: "blogapp", password: nil, ...}]  # ไม่มี host!
["queue",   {adapter: "postgresql", host: nil, username: "blogapp", password: nil, ...}]  # ไม่มี host!
["cable",   {adapter: "postgresql", host: nil, username: "blogapp", password: nil, ...}]  # ไม่มี host!
```

สาเหตุคือ **Rails merge ค่าจาก `DATABASE_URL` ให้กับฐานข้อมูลที่ชื่อ `primary` เท่านั้นโดย
convention** ฐานข้อมูลรองต้องมีตัวแปรของตัวเองในรูปแบบ `<ชื่อ upcase>_DATABASE_URL` (ตามที่คอมเมนต์ใน
`config/database.yml` ที่ Rails generate มาบอกไว้จริง) วิธีแก้คือส่งตัวแปรเพิ่มให้ครบ:

```yaml
    environment:
      DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development
      CACHE_DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development_cache
      QUEUE_DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development_queue
      CABLE_DATABASE_URL: postgres://blogapp:blogapp_dev_password@db:5432/blogapp_development_cable
```

พอเพิ่มครบทั้ง 4 ตัวแล้วรัน `db:prepare`/`db:migrate` ใหม่ ทุกฐานข้อมูลถูกสร้างสำเร็จจริง:

```
Created database 'blogapp_production_cache'
Created database 'blogapp_production_queue'
Created database 'blogapp_production_cable'
```

> **ทำไมเรื่องนี้ไม่ค่อยมีใครพูดถึง:** เพราะ Rails 8 เพิ่งเปลี่ยนมาใช้ `solid_cache`/`solid_queue`/
> `solid_cable` เป็นค่าเริ่มต้นแทน Redis เมื่อไม่นานนี้เอง บทความ/tutorial เก่าที่เขียนก่อนหน้านั้น
> (หรือที่พูดถึง Rails เวอร์ชันก่อน 8) จะไม่มีฐานข้อมูลรองพวกนี้เลย จึงไม่เคยเจอปัญหานี้

### Debug Container ที่กำลังรันอยู่ด้วย `docker exec`

`docker exec` เปิด process ใหม่ **เข้าไปข้างใน container ที่กำลังรันอยู่แล้ว** (ต่างจาก `docker run`
ที่สร้าง container ใหม่) มีประโยชน์มากตอน debug ปัญหาที่เกิดเฉพาะข้างใน container:

```bash
# เปิด shell แบบ interactive เข้าไปสำรวจ
docker exec -it blogdemo-web-1 bash

# รันคำสั่งเดียวแล้วออกทันที (ไม่ต้อง interactive)
docker exec blogdemo-web-1 bin/rails runner "puts Post.count"
# => 1

# เช็กว่า process รันด้วย user อะไร (ยืนยันเรื่อง non-root จาก Step 722)
docker exec blogdemo-web-1 whoami
```

`-it` คือสอง flag รวมกัน: `-i` (interactive — เปิด stdin ค้างไว้) และ `-t` (จำลอง terminal — ทำให้
prompt/สีต่างๆ แสดงผลถูกต้อง) จำเป็นทั้งคู่เวลาต้องการ shell แบบโต้ตอบได้จริง แต่ไม่จำเป็นถ้าแค่รัน
คำสั่งเดียวแล้วจบ

### ดู Log ด้วย `docker logs`

```bash
# ดู log ทั้งหมดตั้งแต่ container เริ่ม
docker logs blogdemo-web-1

# ดูแบบ real-time (เหมือน tail -f)
docker logs -f blogdemo-web-1

# ดูแค่ 20 บรรทัดล่าสุด
docker logs --tail 20 blogdemo-web-1
```

ตัวอย่าง log จริงที่เห็นตอนยิง request เข้าไปทดสอบ (แสดงให้เห็นว่า Rails log ปกติที่คุ้นเคยตอนรันบน
เครื่อง ก็ยังปรากฏใน `docker logs` เหมือนเดิมทุกประการ เพราะ Rails เขียน log ออกทาง `STDOUT` เป็นค่า
default ใน production ซึ่งเป็นแนวทางมาตรฐานของแอปที่ทำงานใน container):

```
Processing by PostsController#index as JSON
  Rendering posts/index.json.jbuilder
  Post Load (0.5ms)  SELECT "posts".* FROM "posts" ...
  Rendered posts/index.json.jbuilder (Duration: 14.3ms | GC: 0.9ms)
Completed 200 OK in 16ms (Views: 10.6ms | ActiveRecord: 4.4ms (1 query, 0 cached) | GC: 0.9ms)
```

---

## Step 730: เทคนิคลดขนาด Image + แบบฝึกหัดรวบยอด

### สรุปเทคนิคลดขนาด Image (ทบทวนรวมทุกอย่างที่เรียนมา)

1. **ใช้ Multi-stage Build เสมอสำหรับ production image** (Step 723) — แยก build tools ออกจาก
   runtime อย่างเด็ดขาด ประหยัดได้มากที่สุดในบรรดาเทคนิคทั้งหมด มักลดขนาดได้ 50–70%
2. **เลือก base image แบบ slim/alpine เมื่อเหมาะสม** — `ruby:3.3.6-slim` (Debian slim) เล็กกว่า
   `ruby:3.3.6` (Debian เต็ม) มาก เพราะตัดเครื่องมือพัฒนาที่ไม่จำเป็นออก (บาง team เลือกใช้
   `ruby:3.3.6-alpine` ที่เล็กกว่าอีก แต่ต้องระวังเรื่อง `musl libc` ที่บางครั้งเข้ากันไม่ได้กับ native
   gem บางตัวที่ compile มาสำหรับ `glibc` — Rails 8 เลือก `-slim` เป็นค่า default เพราะความเข้ากันได้
   สูงกว่า alpine อย่างชัดเจน)
3. **เขียน `.dockerignore` ให้ครบ** (Step 726) — ไม่ส่งไฟล์ที่ไม่จำเป็นเข้า build context ตั้งแต่ต้น
4. **ลบ cache ใน `RUN` เดียวกับที่สร้างมันขึ้นมา** — `rm -rf /var/lib/apt/lists` ต้องอยู่ใน `RUN`
   คำสั่งเดียวกับ `apt-get install` เสมอ (ต่อด้วย `&&`) ไม่ใช่แยกเป็นคนละบรรทัด เพราะแต่ละ layer เป็น
   diff ที่สะสมกัน ลบทีหลังไม่ทำให้ layer ก่อนหน้าเล็กลง
5. **`--no-install-recommends` ตอน `apt-get install` เสมอ** — ป้องกันไม่ให้ apt ติดตั้ง package
   แนะนำเพิ่มเติมที่ไม่จำเป็นต่อการทำงานจริง (สังเกตว่า Dockerfile ของ Rails ใช้ flag นี้ทุกจุด)
6. **`BUNDLE_WITHOUT="development"`** — ไม่ติดตั้ง gem debug/test ใน production image เลย

ตรวจสอบว่า layer ไหนกินพื้นที่มากที่สุดด้วย:

```bash
docker history blogapp --format "table {{.Size}}\t{{.CreatedBy}}" | head -20
```

### แบบฝึกหัดรวบยอด: Dockerize แอป Rails ด้วย Docker Compose ตั้งแต่ต้นจนจบ

**โจทย์:** สร้างแอป Rails ใหม่ชื่อ `taskapp` ที่มี resource `Task` (มี field `title:string` และ
`done:boolean`) แล้ว:

1. ใช้ Dockerfile ที่ Rails 8 generate ให้ (production)
2. เขียน `Dockerfile.dev` และ `docker-compose.yml` สำหรับ local development ที่มี service `web` และ
   `db` (PostgreSQL) พร้อม health check
3. รัน `docker compose up`, รัน migration ผ่าน container, และยืนยันว่าเข้าถึงหน้า index ของ Task ผ่าน
   browser/`curl` ได้จริง

### เฉลย

**ขั้นที่ 1 — สร้างแอปและ scaffold:**

```bash
rails new taskapp --database=postgresql
cd taskapp
```

`Dockerfile` และ `.dockerignore` ถูกสร้างให้อัตโนมัติแล้ว (เหมือนที่อธิบายไว้ใน Step 722 ทุกประการ ไม่
ต้องแก้ไขอะไรเพิ่ม)

**ขั้นที่ 2 — เขียน `Dockerfile.dev`:**

```dockerfile
# Dockerfile.dev
FROM ruby:3.3.6-slim

WORKDIR /rails

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential git libpq-dev libyaml-dev pkg-config \
      libvips postgresql-client curl && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY Gemfile Gemfile.lock ./
RUN bundle install

EXPOSE 3000
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

**ขั้นที่ 3 — เขียน `docker-compose.yml`:**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: taskapp
      POSTGRES_PASSWORD: taskapp_dev_password
      POSTGRES_DB: taskapp_development
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: bash -c "rm -f tmp/pids/server.pid && bin/rails server -b 0.0.0.0"
    volumes:
      - .:/rails
      - bundle-cache:/usr/local/bundle
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://taskapp:taskapp_dev_password@db:5432/taskapp_development
      RAILS_ENV: development
    depends_on:
      db:
        condition: service_healthy

volumes:
  db-data:
  bundle-cache:
```

**ขั้นที่ 4 — Generate scaffold (ก่อน `docker compose up` ครั้งแรก จะ generate นอก container โดยตรงก็
ได้ถ้าเครื่องมี Ruby อยู่แล้ว หรือ generate ผ่าน container ก็ได้เช่นกัน — ในที่นี้สาธิตแบบ generate
ผ่าน container เพื่อพิสูจน์ว่าไม่จำเป็นต้องติดตั้ง Ruby บนเครื่องโฮสต์เลยด้วยซ้ำ):**

```bash
docker compose build
docker compose run --rm web bin/rails generate scaffold Task title:string done:boolean
```

**ขั้นที่ 5 — รัน compose ขึ้นมาจริง แล้ว migrate:**

```bash
docker compose up -d
docker compose ps
```

ผลลัพธ์ที่ควรเห็น (รูปแบบเดียวกับที่ทดสอบจริงใน Step 728):

```
NAME             IMAGE        SERVICE   STATUS                   PORTS
taskapp-db-1     postgres:16  db        Up 8 seconds (healthy)   5432/tcp
taskapp-web-1    taskapp-web  web       Up 3 seconds             0.0.0.0:3000->3000/tcp
```

```bash
docker compose run --rm web bin/rails db:migrate
```

```
== ...: CreateTasks: migrating ================================================
-- create_table(:tasks)
   -> 0.0079s
== ...: CreateTasks: migrated (0.0080s) =======================================
```

**ขั้นที่ 6 — ยืนยันด้วย HTTP request จริง:**

```bash
curl -i http://localhost:3000/tasks
# HTTP/1.1 200 OK ...

curl -X POST http://localhost:3000/tasks.json \
  -H "Content-Type: application/json" \
  -d '{"task": {"title": "เขียนแบบฝึกหัด Docker", "done": false}}'

curl http://localhost:3000/tasks.json
# [{"id":1,"title":"เขียนแบบฝึกหัด Docker","done":false,...}]
```

(รูปแบบผลลัพธ์นี้ตรงกับที่ทดสอบจริงผ่าน scaffold `Post` ใน Step 729 ทุกประการ เพราะกลไกเบื้องหลังคือ
กลไกเดียวกัน แค่เปลี่ยนชื่อ resource)

**ขั้นที่ 7 — ทำความสะอาด:**

```bash
docker compose down -v   # -v ลบ named volume ด้วย (ข้อมูลใน db-data จะหายถาวร ใช้เมื่อต้องการล้างจริงๆ)
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม service `redis` เข้าไปใน `docker-compose.yml` (ใช้ image `redis:7-alpine`) แล้วปรับ
   `config/cable.yml` ให้ Action Cable ใช้ Redis adapter แทน `solid_cable` เริ่มต้น — ทดสอบว่า Rails
   เชื่อมต่อ Redis ผ่านชื่อ service ได้จริงด้วย `docker compose exec web bin/rails runner
   "puts Redis.new(url: ENV['REDIS_URL']).ping"`
2. แก้ `docker-compose.yml` ให้ `web` service มี **healthcheck ของตัวเอง** ที่ยิงไปที่ `/up` (ใช้
   `curl` ที่ติดตั้งไว้ใน `Dockerfile.dev` แล้ว) แล้วลองจำลองสถานการณ์ที่แอป error ตอนบูต (เช่น ลืมตั้ง
   `RAILS_MASTER_KEY`) ดูว่า `docker compose ps` แสดงสถานะ `unhealthy` ถูกต้องหรือไม่
3. เขียน multi-stage `Dockerfile.dev` ที่ใช้ `docker buildx bake` หรือ BuildKit cache mount
   (`RUN --mount=type=cache,target=/usr/local/bundle`) เพื่อให้ `bundle install` เร็วขึ้นในการ build
   ครั้งถัดๆ ไป แม้ `Gemfile.lock` จะเปลี่ยนก็ตาม แล้ววัดเวลา build ก่อน/หลังเทียบกัน

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปัญหา **"works on my machine"** และทำไม container ถึงแก้ปัญหาความไม่สอดคล้องกันของ
  environment ระหว่าง dev/CI/production ได้ดีกว่า VM หรือเอกสารการติดตั้งแบบเดิม
- รู้ว่า **Rails 8 สร้าง production-ready `Dockerfile` ให้อัตโนมัติตั้งแต่ `rails new`** และอ่าน
  Dockerfile จริงได้ทุกบรรทัด ตั้งแต่ base image, ENV vars, การติดตั้ง system package, ไปจนถึง
  ENTRYPOINT/CMD และบทบาทของ Thruster
- เข้าใจกลไก **multi-stage build** อย่างลึกซึ้งว่าทำไมการแยก `base`/`build`/final stage ทำให้ image
  สุดท้ายเล็กลงมาก โดยพิสูจน์ด้วยตัวเลขขนาด image จริงจากการ build จริง
- `docker build` และ `docker run` แอป Rails จริงได้ พร้อมยิง HTTP request เข้าไปยืนยันว่า container
  เสิร์ฟ request ได้จริงผ่าน endpoint `/up`
- จัดการ environment variable ใน Docker ได้ครบสามวิธี (`ENV`, `-e`, `--env-file`) พร้อมเข้าใจกลไก
  `RAILS_MASTER_KEY` ในบริบทของ container และรู้จักกับดักจริงเรื่อง hostname ที่มีขีดล่างทำให้
  `DATABASE_URL` parse ไม่ผ่าน
- เข้าใจว่าทำไม `.dockerignore` สำคัญ และอะไรบ้างที่ไม่ควรหลุดเข้าไปอยู่ใน image (โดยเฉพาะ
  `master.key` และ `.git`)
- เขียน `docker-compose.yml` สำหรับ local development ที่มี web + PostgreSQL พร้อม volume สอง
  ประเภท (bind mount สำหรับ hot-reload, named volume สำหรับข้อมูลถาวร) ได้ด้วยตัวเอง
- ใช้ `depends_on` ร่วมกับ `healthcheck` เพื่อแก้ปัญหา race condition ระหว่าง service ได้จริง และ
  เข้าใจกับดักจริงเรื่อง `DATABASE_URL` ที่ไม่ครอบคลุมฐานข้อมูลรองของ `solid_cache`/`solid_queue`/
  `solid_cable`
- รัน migration ผ่าน `docker compose run` และ debug container ที่กำลังรันอยู่ด้วย `docker exec`/
  `docker logs` ได้อย่างคล่องแคล่ว
- Dockerize แอป Rails ตั้งแต่ต้นจนจบด้วย Docker Compose ผ่านแบบฝึกหัดรวบยอด พร้อมยืนยันผลด้วย HTTP
  request จริง

**ต่อไป (Part 074):** ตอนนี้เรามีแอปที่ "รันได้ในกล่อง" แล้ว แต่คำถามที่ตามมาทันทีคือ — กล่องเดียวกันนี้
ต้องรันได้ทั้งบนเครื่อง dev, บน CI server, และบน production server จริง โดยที่ค่า config บางอย่างต้อง
ต่างกันไปในแต่ละที่ Part 074 จะพาไปดูกลไก Rails environments อย่างเป็นระบบ, การแยก credentials ตาม
environment, กรอบการตัดสินใจว่า "ค่าไหนควรเก็บที่ไหน" ระหว่าง `ENV` กับ Rails credentials, และปรัชญา
Twelve-Factor App ที่เป็นรากฐานของทุกเรื่องที่เรียนใน Phase นี้
