# Part 021: ติดตั้ง Rails, `rails new`, โครงสร้างโฟลเดอร์ และการรัน Server ครั้งแรก

> **Step ครอบคลุมใน Part นี้:** Step 201–210
> **ระดับ:** เริ่มต้น (ต้องผ่าน Phase 1–2 มาก่อนทั้งหมด โดยเฉพาะ Part 001 เรื่อง Convention over
> Configuration และ `bundle exec`, Part 010 โปรเจกต์ Library CLI, และ Part 020 โปรเจกต์ Todo CLI
> + `Rakefile`)
> **เวอร์ชันที่ใช้:** Rails 8.1.x บน Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบรันจริงบน Ruby 3.3.6 +
> Rails 8.1.4) — แนวคิดหลักทั้งหมดที่สอนใน Part นี้ (โครงสร้างโฟลเดอร์, MVC, server, console,
> environment) เสถียรมาตั้งแต่ Rails 7.1 จนถึง 8.x เพราะเป็นพฤติกรรมพื้นฐานของสถาปัตยกรรม Rails
> ที่ไม่เปลี่ยนบ่อย ต่างจาก syntax ปลีกย่อยของแต่ละ gem ที่อาจขยับได้ตามเวอร์ชัน

ยินดีต้อนรับสู่ **Phase 3: Rails Fundamentals — MVC** นี่คือจุดเริ่มต้นอย่างเป็นทางการของการเป็น
นักพัฒนา Ruby on Rails หลังจากใช้เวลา 20 Part เต็มไปกับการฝึกฝน **Ruby ล้วนๆ** (Phase 1–2)
มาถึงจุดนี้แล้ว สิ่งที่เขียนมาตลอดทางไม่ได้สูญเปล่าเลยสักนิด — มันคือ**รากฐานที่ Rails สร้างขึ้น
มาอยู่บนนั้นโดยตรง**:

- **Part 010 (Library CLI)** สอนให้แยก `class` ตามหน้าที่ความรับผิดชอบ (เก็บข้อมูลหนังสือ vs
  จัดการ collection ของหนังสือ) — นี่คือแนวคิดเดียวกับการแยก **Model** ใน Rails
- **Part 020 (Todo CLI)** สอนให้แยกไฟล์เป็น `lib/todo_cli/{task,task_list,cli}.rb` พร้อม
  `Rakefile` ที่ root และมี `bin/todo` เป็น entry point — โครงสร้างนี้แทบจะ**เหมือนกันเป๊ะ**กับ
  โครงสร้างของ Rails application ที่กำลังจะเห็นใน Part นี้ เพียงแค่ `TaskList` (เก็บข้อมูล +
  business logic) จะกลายเป็น **Model**, ส่วน `CLI` (รับ input, เรียก logic, แสดงผล) จะถูกแยก
  ออกเป็น **Controller** (รับ HTTP request, เรียก Model) และ **View** (render HTML แทนการ
  `puts` ออกทาง terminal)

พูดให้ชัดที่สุด: **Rails ไม่ใช่ภาษาใหม่ และไม่ใช่วิธีคิดใหม่** — มันคือ **Ruby ล้วนๆ ที่มี
โครงสร้างไฟล์มาตรฐาน (Convention) และ library (gem) จำนวนมากที่ทำให้เราไม่ต้องเขียนสิ่งที่
เว็บแอปพลิเคชันแทบทุกตัวต้องมีซ้ำ**เอง (routing, การอ่านเขียนฐานข้อมูล, การ render HTML, การจัดการ
session) Part นี้จะพาไปติดตั้ง Rails, สร้างโปรเจกต์แรกด้วย `rails new`, และสำรวจโครงสร้างโฟลเดอร์
ทั้งหมดที่ Rails สร้างให้แบบละเอียดทีละจุด ก่อนที่ Part 022 เป็นต้นไปจะเจาะลึกแต่ละส่วนของ MVC
ทีละชั้น

## สารบัญของ Part นี้

- Step 201: Rails คืออะไร — Framework บน Ruby + Rack และภาพรวม MVC แบบ Bird's-eye View
- Step 202: ติดตั้ง Rails ด้วย `gem install rails` และตรวจสอบเวอร์ชัน
- Step 203: `rails new` และ flag สำคัญที่ควรรู้ (`--minimal`, `--api`, `--database`, `--css`)
- Step 204: โครงสร้างโฟลเดอร์ระดับบนสุด และเหตุผลเบื้องหลัง Convention over Configuration
- Step 205: เจาะลึก `app/` — models, views, controllers, helpers, jobs, mailers, channels
- Step 206: `config/routes.rb`, `config/database.yml`, `config/environments/*.rb` แบบภาพรวม
- Step 207: รัน Development Server ด้วย `bin/rails server` และ `bin/dev`
- Step 208: Rails Console (`bin/rails console`) — irb ที่รู้จักแอปทั้งแอปของเรา
- Step 209: Environment (development/test/production), `RAILS_ENV`, และ `bin/` vs `bundle exec`
- Step 210: แบบฝึกหัด — สร้างแอป Rails แรกของคุณเอง (`bookshelf`) ตั้งแต่ศูนย์

---

## Step 201: Rails คืออะไร — Framework บน Ruby + Rack และภาพรวม MVC แบบ Bird's-eye View

ทบทวนจาก **Part 001 Step 1**: Ruby on Rails คือ Web Application Framework ที่เขียนด้วยภาษา Ruby
เปิดตัวปี 2004 โดย David Heinemeier Hansson (DHH) จุดเด่นที่สุดคือ **Convention over
Configuration (CoC)** และ **Don't Repeat Yourself (DRY)** — ตอนนั้นเป็นแค่คำนิยามลอยๆ แต่ตอนนี้
เราพร้อมแล้วที่จะเห็นมันเป็นรูปธรรม

### Rails สร้างอยู่บนอะไร

Rails ไม่ได้ "คุย" กับ web browser โดยตรง มันวางอยู่บนสถาปัตยกรรม 2 ชั้นที่ซ้อนกัน:

```
Web Browser
     │  HTTP Request/Response
     ▼
┌─────────────────────────────────────┐
│              Rack                    │  ← interface มาตรฐานกลางที่ web server ภาษา Ruby
│  (Web Server Interface มาตรฐาน)      │     ทุกตัว (Puma, Unicorn, Passenger) เข้าใจตรงกัน
└─────────────────────────────────────┘
     ▲
     │  Rails คือ Rack application ตัวหนึ่ง (ขนาดใหญ่มาก) ที่ประกอบด้วยหลาย middleware
┌─────────────────────────────────────┐
│         Ruby on Rails                │  ← Framework ที่เราจะเรียนตลอดหลักสูตรนี้
│  (Routing, MVC, ActiveRecord, ...)   │
└─────────────────────────────────────┘
     ▲
     │  เขียนด้วยภาษา Ruby ล้วนๆ (Phase 1–2 ที่เราเพิ่งเรียนจบ)
┌─────────────────────────────────────┐
│              Ruby                    │
└─────────────────────────────────────┘
```

**Rack** คือ interface (ข้อตกลงร่วม) ที่นิยามว่า web application ภาษา Ruby ตัวไหนก็ตาม ต้องตอบสนอง
ต่อ HTTP request ด้วยรูปแบบเดียวกัน (รับ environment hash คืนค่าเป็น `[status, headers, body]`)
— เพราะมี Rack เป็นมาตรฐานกลางนี้เอง ทำให้ web server ตัวไหน (Puma ที่ Rails ใช้เป็น default,
หรือตัวอื่น) ก็รัน Rails app ได้โดยไม่ต้องเขียนโค้ดผูกกับ server ตัวใดตัวหนึ่งโดยเฉพาะ — เราจะ
เจาะลึก Rack จริงจังใน **Phase 15 (Part 087)** ตอนนี้แค่รู้ไว้ว่า **Rails = Framework ขนาดใหญ่ที่
เขียนด้วย Ruby และวางอยู่บน Rack** ก็เพียงพอสำหรับเริ่มต้น

### MVC แบบ Bird's-eye View (ภาพรวมก่อนลงรายละเอียด)

**MVC (Model-View-Controller)** คือรูปแบบการแบ่งความรับผิดชอบของโค้ดออกเป็น 3 ส่วน ที่ Rails
ยึดถือเป็นแกนกลางของทุกอย่าง:

```
                    HTTP Request (เช่น GET /books/1)
                              │
                              ▼
                       ┌─────────────┐
                       │   Router    │  config/routes.rb ตัดสินใจว่า request นี้
                       │             │  ควรส่งไปหา Controller#action ไหน
                       └──────┬──────┘
                              │
                              ▼
                       ┌─────────────┐
                       │ Controller  │  รับ request, ดึงข้อมูลจาก Model,
                       │             │  ตัดสินใจว่าจะ render View ไหนกลับไป
                       └──────┬──────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
             ┌─────────────┐     ┌─────────────┐
             │    Model    │     │    View     │
             │             │     │             │
             │ ข้อมูล +     │     │ สร้าง HTML   │
             │ business    │     │ ที่จะแสดงผล  │
             │ logic       │     │ กลับไปหา     │
             │ (คุยกับ DB) │     │ browser     │
             └─────────────┘     └─────────────┘
```

| ส่วน | หน้าที่ | เทียบกับที่เคยเขียนใน Phase 2 |
|---|---|---|
| **Model** | เก็บข้อมูล + กติกาทางธุรกิจ (business logic) + คุยกับฐานข้อมูล | `TaskList`/`Task` ใน Part 020 (เก็บข้อมูล, validate, `save_to`/`load_from`) |
| **View** | สร้างหน้าตาที่ผู้ใช้เห็น (HTML) จากข้อมูลที่ Controller ส่งมาให้ | ส่วน `puts`/`print` ใน `CLI#handle_list` ของ Part 020 ที่ "แสดงผล" |
| **Controller** | รับ request, เรียก Model มาทำงาน, ตัดสินใจว่าจะแสดง View ไหน | ส่วน `CLI#dispatch`/`CLI#handle_add` ของ Part 020 ที่ "รับ input แล้วเรียก logic" |

สังเกตว่า `CLI` class ใน Part 020 ทำหน้าที่ของทั้ง **Controller และ View รวมกัน** (รับ input
ผ่าน `gets`, เรียก `TaskList` มาทำงาน, แล้ว `puts` แสดงผลกลับไปในไฟล์เดียวกัน) — สิ่งที่ Rails
ทำเพิ่มเติมคือ **แยก Controller ออกจาก View อย่างเด็ดขาด** เพราะเว็บแอปพลิเคชันมักมีหลายหน้า
(view) ต่อ 1 action และการปนโค้ด "ตัดสินใจ" กับโค้ด "แสดงผล HTML" ไว้ด้วยกันจะยิ่งอ่านยากขึ้น
เรื่อยๆ เมื่อโปรเจกต์โต — เราจะเจาะลึก Controller ใน **Part 023** และ View ใน **Part 024**

> **สิ่งที่ Part นี้ยังไม่สอน (ตั้งใจ):** Part นี้เป็นแค่การ "ปูพื้น" ให้เห็นโครงสร้างไฟล์และรู้จัก
> คำสั่งพื้นฐานเท่านั้น รายละเอียดเชิงลึกของแต่ละส่วน — Routing เต็มรูปแบบ (Part 022), Controller
> action/params/session (Part 023), View/ERB/partial (Part 024), Model/ActiveRecord/Migration
> (Part 025) — จะแยกสอนทีละ Part ในเฟส 3 นี้ทั้งหมด อย่าเพิ่งกังวลถ้ายังเห็นภาพไม่ครบ 100%

---

## Step 202: ติดตั้ง Rails ด้วย `gem install rails` และตรวจสอบเวอร์ชัน

ทบทวนจาก **Part 001 Step 4**: Rails คือ **gem** ตัวหนึ่ง (จริงๆ แล้วเป็น "meta-gem" ที่รวม gem
ย่อยอีกหลายสิบตัวไว้ด้วยกัน เช่น `activerecord`, `actionpack`, `activesupport`) ติดตั้งผ่าน
`gem install` เหมือน gem ทั่วไปทุกประการ ไม่มีขั้นตอนพิเศษอะไรเพิ่มเติม

### ติดตั้ง Rails

```bash
# ตรวจสอบว่า Ruby พร้อมใช้งานก่อนเสมอ (ทบทวน Part 001)
ruby -v
# => ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]

# ติดตั้ง Rails เวอร์ชันล่าสุด
gem install rails

# ตรวจสอบว่าติดตั้งสำเร็จ
rails -v
# => Rails 8.1.4
```

> **หมายเหตุสภาพแวดล้อมของหลักสูตรนี้:** เครื่องที่ใช้ทำแบบฝึกหัดใน Part นี้ติดตั้ง Ruby 3.3.6
> และ Rails 8.1.4 ไว้ล่วงหน้าแล้วผ่าน `gem install rails` ถ้าเปิด terminal แล้วพิมพ์ `rails -v`
> ไม่เจอคำสั่ง (`command not found: rails`) ให้ลองสั่ง
> `export PATH="/opt/rbenv/versions/3.3.6/bin:$PATH"` ก่อน แล้วลองใหม่อีกครั้ง — ปัญหานี้เกิดจาก
> `PATH` ของ shell ยังไม่รู้จักตำแหน่งที่ `rbenv` ติดตั้ง Ruby/gem ไว้ (ทบทวนเรื่อง `rbenv`/`PATH`
> จาก Part 001 Step 2)

### ติดตั้ง Rails เวอร์ชันเจาะจง

ถ้าต้องการเวอร์ชันที่ตรงกับที่ทีมหรือหลักสูตรใช้เป๊ะๆ (สำคัญมากในงานจริง เพราะแต่ละเวอร์ชันของ
Rails มี breaking change ปะปนอยู่บ้าง) ระบุเวอร์ชันต่อท้ายชื่อ gem ได้เหมือน gem ทั่วไป:

```bash
gem install rails -v 8.1.4

# ถ้าเครื่องมี Rails หลายเวอร์ชันติดตั้งพร้อมกัน ดูรายการทั้งหมดได้ด้วย
gem list rails --local
# => rails (8.1.4, 7.1.5)   ตัวอย่างถ้ามี 2 เวอร์ชันอยู่ในเครื่อง

# เลือกรันเวอร์ชันใดเวอร์ชันหนึ่งแบบเจาะจงตอนสร้างโปรเจกต์ใหม่
rails _8.1.4_ new myapp
```

### ตรวจสอบ dependency ที่ Rails ต้องการ

Rails ไม่ได้ทำงานตัวเดียวโดดๆ — มันดึง gem อื่นๆ เข้ามาด้วยตอนติดตั้ง ลองดูรายการ gem ที่มากับ
Rails หลักได้ด้วยคำสั่งนี้:

```bash
gem dependency rails --version 8.1.4
```

ผลลัพธ์จะแสดง gem ย่อยของ Rails ทั้งหมด เช่น `activesupport`, `activerecord`, `actionpack`,
`actionview`, `activejob`, `actionmailer`, `actioncable`, `activestorage`, `actiontext`,
`actionmailbox`, `railties` — **แต่ละตัวคือ 1 "component" ของ Rails ที่ตรงกับสิ่งที่จะเรียนใน
Phase ต่างๆ ของหลักสูตรนี้** (`activerecord` → Model/Migration ใน Part 025, `actioncable` →
Part 095 เรื่อง real-time chat) การเห็นชื่อเหล่านี้ตั้งแต่ตอนนี้จะช่วยให้จำแนก error message ได้
ง่ายขึ้นในอนาคต เวลาเจอ `ActiveRecord::RecordNotFound` หรือ `ActionController::ParameterMissing`
จะรู้ทันทีว่า error นั้นมาจาก "ส่วน" ไหนของ Rails

---

## Step 203: `rails new` และ flag สำคัญที่ควรรู้ (`--minimal`, `--api`, `--database`, `--css`)

เมื่อมี Rails พร้อมใช้งานแล้ว คำสั่งแรกสุดที่ใช้เริ่มทุกโปรเจกต์คือ **`rails new`**

### สร้างโปรเจกต์แบบพื้นฐานที่สุด

```bash
mkdir -p ~/ruby-course-workspace/part-021
cd ~/ruby-course-workspace/part-021

rails new myapp
```

คำสั่งนี้จะสร้างโฟลเดอร์ `myapp/` พร้อมไฟล์ทั้งหมดหลายร้อยไฟล์ แล้วรัน `bundle install` ให้
อัตโนมัติ (ทบทวน `bundler` จาก Part 001 Step 4) — บนเครื่องที่ยังไม่เคย compile native extension
ของ gem อย่าง `sqlite3` มาก่อน ขั้นตอนนี้**อาจใช้เวลาตั้งแต่ 30 วินาทีถึงหลายนาที** เป็นเรื่องปกติ
ไม่ต้องตกใจถ้าหน้าจอค้างอยู่กับข้อความ `run bundle install --quiet` สักพัก (ในการทดสอบจริงระหว่าง
เขียนบทเรียนนี้ ใช้เวลาประมาณ 50 วินาทีบนเครื่องที่ยังไม่มี gem cache ใดๆ)

เมื่อเสร็จแล้วจะเห็นข้อความสรุปประมาณนี้ต่อท้าย:

```
       rails  solid_cache:install solid_queue:install solid_cable:install
      create  config/cache.yml
      create  db/cache_schema.rb
      create  config/queue.yml
      create  config/recurring.yml
      create  db/queue_schema.rb
      create  bin/jobs
      create  db/cable_schema.rb
```

### Flag ที่ควรรู้จักตั้งแต่วันแรก

`rails new` มี flag ให้เลือกจำนวนมาก (ดูทั้งหมดด้วย `rails new --help`) แต่ 4 ตัวต่อไปนี้คือตัวที่
จะเจอบ่อยที่สุดในงานจริงและตลอดหลักสูตรนี้:

#### `--minimal` — โปรเจกต์แบบเรียบง่ายที่สุด

```bash
rails new bookshelf --minimal
```

`--minimal` ตัดฟีเจอร์ที่ไม่จำเป็นสำหรับการเรียนรู้ MVC พื้นฐานออกไปเกือบทั้งหมด — **ทดสอบจริง
แล้วขั้นตอนนี้ใช้เวลาเพียง ~5 วินาที** (เทียบกับ ~50 วินาทีของแบบเต็ม) เพราะ `bundle install`
มี gem ให้ติดตั้งน้อยกว่ามาก โดยจะ**ไม่มี** สิ่งเหล่านี้ (เทียบ `Gemfile` แบบเต็มกับแบบ `--minimal`
ที่ทดสอบจริง):

| มีในแบบเต็ม (`rails new myapp`) | มีใน `--minimal` หรือไม่ |
|---|---|
| `propshaft` (asset pipeline) | ✅ มี (asset pipeline ยังจำเป็นเสมอ) |
| `sqlite3`, `puma` | ✅ มี (ฐานข้อมูล + web server ยังจำเป็นเสมอ) |
| `importmap-rails`, `turbo-rails`, `stimulus-rails` (Hotwire) | ❌ ไม่มี |
| `jbuilder` | ❌ ไม่มี |
| `solid_cache`, `solid_queue`, `solid_cable` | ❌ ไม่มี |
| `bootsnap` | ❌ ไม่มี |
| `kamal`, `thruster` (deploy-related) | ❌ ไม่มี |
| `image_processing` | ❌ ไม่มี |
| `web-console`, `capybara`, `selenium-webdriver`, `brakeman`, `rubocop-rails-omakase` | ❌ ไม่มี |
| `app/jobs/`, `app/mailers/`, `app/javascript/channels/` | ❌ ไม่ถูกสร้าง |

เหมาะมากสำหรับ**การเรียนรู้ MVC แบบไม่ให้ของที่ยังไม่ได้เรียนมากวนใจ** — Part นี้และ Part 022–030
ในเฟส 3 ทั้งหมดจะใช้ `--minimal` เป็นหลัก แล้วค่อยๆ เพิ่มความสามารถกลับเข้าไปทีละอย่างตามที่แต่ละ
Part สอน (เช่น Part 029 จะกลับมาพูดเรื่อง asset pipeline อย่างละเอียด)

#### `--api` — โหมด API-only (ไม่มี View)

```bash
rails new my_api --api
```

สร้างโปรเจกต์ที่ตัดทุกอย่างที่เกี่ยวกับการ render HTML ออกไปทั้งหมด (ไม่มี `app/views/`,
ไม่มี `app/helpers/`, ไม่มี `app/assets/`, ไม่มี cookie/session middleware บางตัว) เหมาะกับการ
สร้าง backend ที่ตอบกลับเป็น JSON อย่างเดียว (เช่น backend สำหรับ mobile app หรือ frontend แยก
เช่น React) — `ApplicationController` ที่ได้จะสืบทอดจาก `ActionController::API` แทนที่จะเป็น
`ActionController::Base` ตามปกติ:

```ruby
# app/controllers/application_controller.rb (จากโปรเจกต์ที่สร้างด้วย --api)
class ApplicationController < ActionController::API
end
```

จะเรียนเรื่องนี้เต็มรูปแบบใน **Phase 8 (Part 056)** ตอนนี้แค่รู้จักไว้ก่อนว่ามี flag นี้อยู่

#### `--database=postgresql` — เลือกฐานข้อมูลอื่นแทน SQLite

ค่า default ของ Rails ตั้งแต่เวอร์ชัน 8 คือ **SQLite** (ไฟล์ฐานข้อมูลเดียว ไม่ต้องติดตั้ง database
server แยก — เหมาะมากสำหรับพัฒนาและเรียนรู้) แต่งานจริงระดับ production มักใช้ PostgreSQL หรือ
MySQL แทน เพราะรองรับ concurrent connection จำนวนมากและมี feature ระดับองค์กรมากกว่า

```bash
rails new my_app --database=postgresql
```

ผลลัพธ์ที่ต่างไปจากค่า default (ทดสอบจริงแล้ว): `Gemfile` จะมี `gem "pg", "~> 1.1"` แทน
`gem "sqlite3"` และ `config/database.yml` จะเปลี่ยนรูปแบบไปตามชนิดฐานข้อมูลใหม่ทันที:

```yaml
# config/database.yml (บางส่วน จากโปรเจกต์ที่สร้างด้วย --database=postgresql)
# PostgreSQL. Versions 9.5 and up are supported.
#
# Install the pg driver:
#   gem install pg
default: &default
  adapter: postgresql
  encoding: unicode
  # ...
```

> **หมายเหตุ:** flag นี้แค่**เปลี่ยน config ให้ตรงกับฐานข้อมูลที่เลือก** เท่านั้น — มันไม่ได้
> ติดตั้ง PostgreSQL server ให้อัตโนมัติ ถ้าเลือก `--database=postgresql` แต่เครื่องยังไม่มี
> PostgreSQL server รันอยู่ คำสั่ง `rails db:create` (จะเรียนใน Part 025) จะ error ทันที
> ค่า `Possible values` ทั้งหมดของ flag นี้คือ `mysql, trilogy, postgresql, sqlite3,
> mariadb-mysql, mariadb-trilogy`

#### `--css=<ตัวเลือก>` — เพิ่ม CSS framework ตั้งแต่สร้างโปรเจกต์

```bash
rails new my_app --css=tailwind
```

ติดตั้งและตั้งค่า Tailwind CSS (หรือ `bootstrap`, `bulma`, `postcss`, `sass` — ดูค่าที่รองรับ
ทั้งหมดด้วย `rails new --help`) ให้พร้อมใช้งานทันทีผ่าน gem `cssbundling-rails` แทนที่จะต้องมา
ติดตั้งเองทีหลัง — จะเจาะลึกเรื่อง CSS framework ใน Rails ใน **Part 054**

### สรุป Flag ที่ใช้บ่อย

| Flag | ความหมาย |
|---|---|
| `rails new myapp` | สร้างโปรเจกต์แบบเต็ม (Hotwire, Active Job, Active Storage ครบ) |
| `rails new myapp --minimal` | สร้างแบบเรียบง่าย ตัดฟีเจอร์ที่ยังไม่จำเป็นออก (ใช้ในหลักสูตรนี้เป็นหลักช่วงเฟส 3) |
| `rails new myapp --api` | โหมด API-only ไม่มี View/asset |
| `rails new myapp --database=postgresql` | เปลี่ยนฐานข้อมูลจาก SQLite เป็น PostgreSQL |
| `rails new myapp --css=tailwind` | ติดตั้ง Tailwind CSS ให้อัตโนมัติ |
| `rails new myapp --skip-git` | ไม่สั่ง `git init` ให้ (มีประโยชน์เวลาทำแบบฝึกหัดในหลักสูตรนี้) |
| `rails new --help` | ดู flag ทั้งหมดที่รองรับ (มีมากกว่า 40 ตัว) |

---

## Step 204: โครงสร้างโฟลเดอร์ระดับบนสุด และเหตุผลเบื้องหลัง Convention over Configuration

สร้างโปรเจกต์ตัวอย่างสำหรับสำรวจโครงสร้าง (ใช้แบบเต็มในรอบนี้ เพื่อให้เห็นโฟลเดอร์ครบทุกแบบ):

```bash
cd ~/ruby-course-workspace/part-021
rails new myapp
cd myapp
```

รันคำสั่งนี้ดูโครงสร้างทั้งหมด (ทดสอบจริงแล้ว):

```bash
find . -maxdepth 1 | sort
```

```
.
./.dockerignore
./.github
./.kamal
./.rubocop.yml
./.ruby-version
./Dockerfile
./Gemfile
./Gemfile.lock
./README.md
./Rakefile
./app
./bin
./config
./config.ru
./db
./lib
./log
./public
./script
./storage
./test
./tmp
./vendor
```

### เหตุผลเบื้องหลัง: ทำไม Rails ต้องแยกโฟลเดอร์แบบนี้

ทบทวนจาก **Part 001 Step 1**: หลักการข้อแรกของ Rails คือ **Convention over Configuration (CoC)**
— แทนที่จะให้ผู้พัฒนาแต่ละคน/แต่ละบริษัทตัดสินใจเองว่า "ไฟล์ประเภทนี้ควรอยู่ตรงไหน" (แล้วต้องมา
เขียน config บอก framework ให้รู้ทุกครั้ง) Rails **กำหนดตำแหน่งมาตรฐานไว้ให้แล้ว** — ถ้าทำตาม
convention นี้ ทุกอย่างจะทำงานได้ทันทีโดยแทบไม่ต้อง config อะไรเพิ่มเลย

ทบทวนจาก **Part 020 Step 197**: โปรเจกต์ Todo CLI ก็ใช้หลักการเดียวกันนี้อยู่แล้วในสเกลเล็ก
(`lib/todo_cli/` แยกตามหน้าที่, `bin/todo` เป็น entry point, `Rakefile` อยู่ที่ root) — ข้อ
แตกต่างเดียวคือ **Rails บังคับ convention นี้อย่างเข้มงวดกว่ามาก** และมี "เครื่องจักร" (Zeitwerk,
routing, ฯลฯ) ที่อ่าน convention เหล่านี้แล้วทำงานให้อัตโนมัติ ไม่ใช่แค่ทำให้อ่านง่ายเฉยๆ เหมือน
ตอนที่เราจัดโฟลเดอร์เองด้วยมือใน Ruby ล้วน

ตารางต่อไปนี้อธิบายหน้าที่ของแต่ละโฟลเดอร์/ไฟล์ระดับบนสุด:

| โฟลเดอร์/ไฟล์ | หน้าที่ | เทียบกับ Part 020 (Todo CLI) |
|---|---|---|
| **`app/`** | หัวใจของแอปพลิเคชัน — โค้ด MVC ทั้งหมด (models, views, controllers และอื่นๆ) | เทียบเท่า `lib/todo_cli/` |
| **`bin/`** | executable script สำหรับรันคำสั่งต่างๆ ของโปรเจกต์นี้โดยเฉพาะ (`bin/rails`, `bin/rake`, `bin/setup`) | เทียบเท่า `bin/todo` |
| **`config/`** | การตั้งค่าทั้งหมดของแอป — routes, database, environment, credentials | ไม่มีเทียบเท่าตรงๆ ใน Todo CLI (เพราะโปรเจกต์เล็กเกินไปจะต้องมี config แยก) |
| **`db/`** | migration files (ประวัติการเปลี่ยนแปลงโครงสร้างฐานข้อมูล), `schema.rb`, `seeds.rb` | เทียบเท่าไฟล์ `todos.json` (แต่ซับซ้อนกว่ามาก — จะเรียนใน Part 025) |
| **`lib/`** | โค้ดที่ไม่ใช่ core ของแอป (ใช้ร่วมข้ามโปรเจกต์ได้, custom Rake task) — **ไม่ใช่** ที่เก็บ Model/Controller เหมือนที่เข้าใจผิดกันบ่อย | เทียบเท่า `lib/tasks/` ย่อยในธีม Rake |
| **`log/`** | ไฟล์ log ของแต่ละ environment (`development.log`, `production.log`) | ไม่มีใน Todo CLI |
| **`public/`** | ไฟล์ static ที่ web server เสิร์ฟตรงๆ โดยไม่ผ่าน Rails routing (หน้า error 404/500, favicon) | ไม่มีใน Todo CLI |
| **`storage/`** | ที่เก็บไฟล์ฐานข้อมูล SQLite (default) และไฟล์ที่อัปโหลดผ่าน Active Storage | เทียบเท่าตำแหน่งที่เก็บ `todos.json` |
| **`test/`** | ชุดทดสอบ (Rails ใช้ **Minitest** เป็น default ไม่ใช่ RSpec — ทบทวนจาก Part 018) | เทียบเท่า `spec/` (แต่ Rails ใช้ Minitest เป็นค่าเริ่มต้น) |
| **`tmp/`** | ไฟล์ชั่วคราวที่ระบบสร้างเอง (cache, pid ของ server ที่กำลังรัน) — ลบทิ้งได้เสมอ | ไม่มีใน Todo CLI |
| **`vendor/`** | โค้ด/asset จาก third-party ที่ไม่ได้มาจาก RubyGems โดยตรง (พบน้อยในโปรเจกต์สมัยใหม่) | ไม่มีใน Todo CLI |
| **`Gemfile`** / **`Gemfile.lock`** | รายการ gem ที่โปรเจกต์นี้ต้องการ + เวอร์ชันที่ล็อกไว้แน่นอน | เหมือนกันเป๊ะกับ `Gemfile` ของ Todo CLI (ทบทวน Part 001 Step 4) |
| **`Rakefile`** | เรียก Rake task ทั้งหมดของโปรเจกต์ (ทบทวนเต็มรูปแบบจาก Part 020) | เหมือนกันเป๊ะ ตำแหน่งเดียวกัน กลไกเดียวกัน |
| **`config.ru`** | ไฟล์ config สำหรับ **Rack** (บอกว่าเมื่อมี request เข้ามา ให้เริ่มต้นแอปอย่างไร) | ไม่มีใน Todo CLI (เพราะ Todo CLI ไม่ใช่ web app) |
| **`.ruby-version`** | ระบุเวอร์ชัน Ruby ที่โปรเจกต์นี้ต้องการ — `rbenv`/`asdf` จะอ่านไฟล์นี้อัตโนมัติ | เทียบเท่า `.ruby-version`/`.tool-versions` ที่เรียนใน Part 001 Step 3 |
| **`.dockerignore`**, **`Dockerfile`** | ไฟล์สำหรับ containerize แอปด้วย Docker (Rails 8 สร้างให้อัตโนมัติทันที) | จะเรียนเต็มรูปแบบใน Part 073 |
| **`.kamal/`** | ไฟล์ config สำหรับ deploy ด้วย Kamal (เครื่องมือ deploy มาตรฐานของ Rails 8) | จะเรียนเต็มรูปแบบใน Part 076 |

> **สังเกตสิ่งสำคัญ:** `Rakefile` และแนวคิดการแยกไฟล์ตามหน้าที่ (`app/` แยกเป็นโฟลเดอร์ย่อยตาม
> role, ไม่ใช่ตามความสะดวก) **ไม่ใช่เรื่องใหม่เลย** — มันคือสิ่งเดียวกับที่ Part 020 สอนไว้แล้ว
> ทั้งหมด เพียงแค่ Rails บังคับใช้อย่างเข้มงวดกว่าและมี tooling รองรับมากกว่าเท่านั้นเอง

### ทำไมต้องมีทั้ง `test/` และ `db/` แยกจาก `app/`

หลักการออกแบบที่สำคัญคือ **แยกสิ่งที่เป็น "โค้ดของแอป" ออกจาก "ข้อมูลเกี่ยวกับแอป"**:

- `app/` = โค้ดที่รันจริงตอนแอปทำงาน (production code)
- `test/` = โค้ดที่รันเฉพาะตอนทดสอบ ไม่ถูก deploy ไปทำงานจริง (คล้าย `spec/` ใน Todo CLI)
- `db/` = ประวัติ/schema ของฐานข้อมูล ไม่ใช่โค้ด logic
- `config/` = การตั้งค่า ไม่ใช่โค้ด logic

การแยกชัดเจนแบบนี้ทำให้เครื่องมือต่างๆ (เช่น test runner, deployment script) รู้ได้ทันทีว่าต้อง
จัดการกับไฟล์กลุ่มไหนอย่างไร โดยไม่ต้องมานั่งไล่เปิดไฟล์ดูทีละไฟล์ว่า "อันนี้คือโค้ดจริงหรือโค้ด
ทดสอบกันแน่"

---

## Step 205: เจาะลึก `app/` — models, views, controllers, helpers, jobs, mailers, channels

`app/` คือโฟลเดอร์ที่จะใช้เวลาส่วนใหญ่ของหลักสูตรนี้อยู่ในนั้น มาดูโครงสร้างเต็มของมัน (ทดสอบ
จริงจากโปรเจกต์แบบเต็มใน Step 204):

```bash
find app -maxdepth 2 | sort
```

```
app
app/assets
app/assets/images
app/assets/stylesheets
app/controllers
app/controllers/application_controller.rb
app/controllers/concerns
app/helpers
app/helpers/application_helper.rb
app/javascript
app/javascript/application.js
app/javascript/controllers
app/jobs
app/jobs/application_job.rb
app/mailers
app/mailers/application_mailer.rb
app/models
app/models/application_record.rb
app/models/concerns
app/views
app/views/layouts
app/views/pwa
```

### `app/models/` — Model

เก็บ class ที่เป็นตัวแทนของ "สิ่งของ" ในระบบ (เช่น `Book`, `User`, `Order`) แต่ละ Model
สืบทอดจาก `ApplicationRecord` ที่ Rails สร้างให้:

```ruby
# app/models/application_record.rb (ไฟล์ที่ Rails สร้างให้ตั้งแต่แรก)
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
end
```

ทุก Model ที่จะสร้างต่อจากนี้ (`class Book < ApplicationRecord`) จะได้ความสามารถทั้งหมดของ
**ActiveRecord** มาฟรีทันที — บันทึก/ค้นหา/แก้ไข/ลบข้อมูลในฐานข้อมูลโดยแทบไม่ต้องเขียน SQL เอง
เลยสักบรรทัด (จะเรียนเต็มรูปแบบใน **Part 025**) เทียบได้กับ `Task`/`TaskList` ใน Part 020 ที่
เก็บข้อมูล + มี business logic ในตัวเอง เพียงแต่ ActiveRecord ผูกกับฐานข้อมูลจริงแทนไฟล์ JSON

`app/models/concerns/` เป็นที่เก็บ **Module** ที่ใช้เป็น mixin แชร์ behavior ร่วมกันระหว่าง
หลาย Model (ทบทวนแนวคิด Module/Mixin จาก Part 010 Step 91–100) — จะเจาะลึกเรื่องนี้ใน
**Part 036 (ActiveSupport::Concern)**

### `app/views/` — View

เก็บ template ที่ใช้ render HTML กลับไปหา browser ปกติจะมีโฟลเดอร์ย่อยตามชื่อ controller
(เช่น `app/views/books/` คู่กับ `BooksController`) แต่ตอนนี้ในโปรเจกต์เปล่ามีแค่:

```
app/views/layouts/          # layout หลักที่ครอบทุกหน้า (application.html.erb)
app/views/pwa/               # ไฟล์สำหรับทำ Progressive Web App (manifest.json, service-worker.js)
```

`app/views/layouts/application.html.erb` คือ layout กลางที่ทุกหน้าของแอปจะถูก "ห่อ" ด้วยไฟล์นี้
เสมอ (มี `<head>`, การโหลด CSS/JS, และ `<%= yield %>` ตรงกลางที่เนื้อหาของแต่ละหน้าจะถูกแทรกเข้าไป)
— จะเจาะลึกไฟล์นี้และระบบ template แบบเต็มใน **Part 024**

### `app/controllers/` — Controller

เก็บ class ที่รับ HTTP request แล้วตัดสินใจว่าจะทำอะไรต่อ ทุก controller สืบทอดจาก
`ApplicationController`:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
end
```

`app/controllers/concerns/` เก็บ Module ที่แชร์ behavior ร่วมกันระหว่างหลาย Controller (เช่น
logic การตรวจสอบสิทธิ์ที่ใช้ซ้ำหลาย controller) — เทียบเท่าแนวคิดเดียวกับ `app/models/concerns/`
แต่ใช้กับฝั่ง Controller — จะเจาะลึก Controller เต็มรูปแบบใน **Part 023**

### `app/helpers/` — Helper

เก็บ Module ที่รวม method ช่วยเหลือสำหรับใช้ใน View (เช่น method จัดรูปแบบวันที่, แปลงราคาเป็น
สกุลเงิน) เพื่อไม่ให้ logic การจัดรูปแบบข้อมูลไปปนอยู่ใน template HTML โดยตรง:

```ruby
# app/helpers/application_helper.rb (ไฟล์เปล่าที่ Rails สร้างให้ตั้งแต่แรก)
module ApplicationHelper
end
```

จะเจาะลึกเรื่อง helper ใน **Part 024**

### `app/jobs/` — Background Job

เก็บงานที่ต้องรันแบบ **asynchronous** (เบื้องหลัง ไม่บล็อกการตอบกลับ request ของผู้ใช้) เช่น
ส่งอีเมลจำนวนมาก, ประมวลผลไฟล์รูปภาพ ทุก job สืบทอดจาก `ApplicationJob`:

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
end
```

จะเจาะลึกเรื่อง Active Job และ background job ใน **Phase 9 (Part 061–062)** — โฟลเดอร์นี้จะ**ไม่
ถูกสร้าง**ถ้าใช้ `rails new --minimal`

### `app/mailers/` — Mailer

เก็บ class ที่จัดการการส่งอีเมลจากแอป (เช่น อีเมลยืนยันการสมัครสมาชิก, อีเมลแจ้งเตือน) ทุก mailer
สืบทอดจาก `ApplicationMailer`:

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: "from@example.com"
  layout "mailer"
end
```

จะเจาะลึกเรื่อง Action Mailer ใน **Part 072** — เช่นเดียวกับ `app/jobs/` โฟลเดอร์นี้จะ**ไม่ถูก
สร้าง**ถ้าใช้ `--minimal`

### `app/channels/` — Channel (สำหรับ Action Cable / WebSocket)

**ข้อสังเกตสำคัญที่ทดสอบจริงแล้ว:** ต่างจาก Rails เวอร์ชันเก่าที่เคยสร้าง
`app/channels/application_cable/{channel,connection}.rb` ให้ตั้งแต่ `rails new` เลยทันที
**Rails 8.1 ไม่สร้างโฟลเดอร์ `app/channels/` ให้อัตโนมัติแล้ว** (เพราะเปลี่ยนไปใช้
`solid_cable` เป็น adapter หลักของ Action Cable แทน) โฟลเดอร์นี้จะปรากฏขึ้นเมื่อสั่ง generator
เพื่อสร้าง channel แรกเท่านั้น:

```bash
bin/rails generate channel Chat
```

```
      create    app/channels/application_cable/channel.rb
      create    app/channels/application_cable/connection.rb
      create    app/channels/chat_channel.rb
      create    test/channels/chat_channel_test.rb
```

**Channel** คือส่วนที่จัดการการสื่อสารแบบ real-time สองทางผ่าน WebSocket (เช่น แชทสด, การแจ้งเตือน
แบบ live) — จะเจาะลึกเรื่อง Action Cable ใน **Part 095 (Capstone 3: Real-time chat)**

### `app/assets/` และ `app/javascript/` — Asset ฝั่ง Frontend

```
app/assets/images/          # รูปภาพของแอป
app/assets/stylesheets/     # ไฟล์ CSS (application.css เป็นไฟล์หลัก)
app/javascript/             # ไฟล์ JavaScript (application.js เป็น entry point เมื่อใช้ importmap)
```

Rails 8 ใช้ **Propshaft** เป็น asset pipeline เริ่มต้น (**ไม่ใช่ Sprockets** เหมือน Rails
เวอร์ชันเก่าที่หลายบทความเก่าในอินเทอร์เน็ตยังอ้างถึงอยู่) — Propshaft เรียบง่ายกว่ามาก มีหน้าที่
แค่ "หาไฟล์ asset แล้วเสิร์ฟออกไปพร้อม fingerprint กันปัญหา cache" โดยไม่ต้อง compile/preprocess
อะไรซับซ้อน จะเจาะลึกเรื่องนี้ใน **Part 029**

### สรุปตารางความรับผิดชอบของ `app/`

| โฟลเดอร์ | เก็บอะไร | สืบทอดจาก | Part ที่เจาะลึก |
|---|---|---|---|
| `models/` | ข้อมูล + business logic + คุยกับฐานข้อมูล | `ApplicationRecord` | Part 025 |
| `views/` | template สำหรับ render HTML | (ไม่มี class ให้สืบทอด — เป็นไฟล์ `.erb`) | Part 024 |
| `controllers/` | รับ request, ตัดสินใจ, เรียก Model | `ApplicationController` | Part 023 |
| `helpers/` | method ช่วยจัดรูปแบบข้อมูลใน View | (Module ธรรมดา) | Part 024 |
| `jobs/` | งานที่รันเบื้องหลังแบบ async | `ApplicationJob` | Part 061–062 |
| `mailers/` | การส่งอีเมล | `ApplicationMailer` | Part 072 |
| `channels/` | การสื่อสารแบบ real-time (WebSocket) | `ApplicationCable::Channel` | Part 095 |

> **ทบทวน autoloading จาก Part 020:** จำได้ไหมว่าใน Todo CLI เราต้อง `require_relative` ทุกไฟล์
> ด้วยมือเองตามลำดับ dependency? **ใน `app/` ของ Rails ไม่ต้องทำแบบนั้นเลยแม้แต่บรรทัดเดียว** —
> ระบบชื่อ **Zeitwerk** จะสแกนทุกไฟล์ใน `app/` แล้วโหลด class/module ให้อัตโนมัติ โดยอาศัย
> convention ง่ายๆ ข้อเดียว: **ชื่อไฟล์ต้องตรงกับชื่อ class/module แบบ snake_case** เช่น
> `app/models/book.rb` ต้องมี `class Book` อยู่ข้างใน ถ้าตั้งชื่อไฟล์/class ไม่ตรงกัน Zeitwerk
> จะ error ทันทีตอนโหลดแอป — นี่คือเหตุผลที่โครงสร้างโฟลเดอร์ที่เข้มงวดใน Rails ไม่ใช่แค่เรื่อง
> "ความสวยงาม" แต่คือกลไกที่ทำให้แอปทำงานได้เลยจริงๆ

---

## Step 206: `config/routes.rb`, `config/database.yml`, `config/environments/*.rb` แบบภาพรวม

`config/` คือโฟลเดอร์ควบคุมพฤติกรรมของทั้งแอป มาดูภาพรวมของไฟล์สำคัญ 3 ไฟล์ — **รายละเอียดเชิงลึก
ของแต่ละไฟล์จะแยกสอนใน Part ถัดๆ ไป** ตอนนี้แค่รู้จักหน้าตาและหน้าที่คร่าวๆ ก็พอ

### `config/routes.rb` — แผนที่ URL ของแอป

```ruby
# config/routes.rb (ไฟล์ที่ Rails สร้างให้ตั้งแต่แรก)
Rails.application.routes.draw do
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Render dynamic PWA files from app/views/pwa/* (remember to link manifest in application.html.erb)
  # get "manifest" => "rails/pwa#manifest", as: :pwa_manifest
  # get "service-worker" => "rails/pwa#service_worker", as: :pwa_service_worker

  # Defines the root path route ("/")
  # root "posts#index"
end
```

ไฟล์นี้คือจุดแรกที่ตัดสินใจว่า "URL แบบนี้ ควรส่งไปให้ Controller ไหน, Action ไหน" — สังเกตว่ามี
route หนึ่งที่ทำงานอยู่แล้วตั้งแต่แรกคือ `get "up" => "rails/health#show"` (endpoint สำหรับให้
load balancer เช็คว่าแอปยังมีชีวิตอยู่ไหม) เดี๋ยวจะลองเรียกดูจริงใน Step 207 — เจาะลึกการเขียน
route แบบเต็มรูปแบบ (`resources`, RESTful convention, route helper อย่าง `books_path`) ใน
**Part 022**

### `config/database.yml` — การตั้งค่าฐานข้อมูลแยกตาม environment

```yaml
# config/database.yml (ค่า default เมื่อใช้ SQLite)
default: &default
  adapter: sqlite3
  max_connections: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

test:
  <<: *default
  database: storage/test.sqlite3

production:
  primary:
    <<: *default
    database: storage/production.sqlite3
  # ... (cache, queue, cable — ฐานข้อมูลย่อยสำหรับ Solid Cache/Queue/Cable)
```

สังเกตว่ามี**ฐานข้อมูลแยกกันคนละไฟล์**ระหว่าง `development`, `test`, `production` โดยอัตโนมัติ
— นี่คือเหตุผลสำคัญที่ต้องเข้าใจเรื่อง **environment** ให้ชัดเจน (Step 209) เพราะข้อมูลที่ทดลอง
เล่นตอนพัฒนา (`development`) จะ**ไม่ปนกับ**ข้อมูลที่ใช้รัน automated test (`test`) และทั้งคู่ก็
ไม่ปนกับข้อมูลจริงของผู้ใช้ (`production`) เด็ดขาด — เจาะลึกการตั้งค่าฐานข้อมูลและ migration ใน
**Part 025**

### `config/environments/*.rb` — ค่า config เฉพาะแต่ละ environment

```bash
ls config/environments/
```

```
development.rb
production.rb
test.rb
```

แต่ละไฟล์คือค่า config ที่ใช้เฉพาะตอนแอปรันใน environment นั้นๆ เช่น เปิด/ปิดการ cache หน้าเว็บ,
ระดับความละเอียดของ log ตัวอย่างค่าที่ต่างกันชัดเจนระหว่างสอง environment (จาก
`config/environments/development.rb` เทียบกับ `production.rb`):

```ruby
# config/environments/development.rb (บางส่วน)
Rails.application.configure do
  # ในโหมดพัฒนา ให้แสดง error page แบบละเอียด (stack trace เต็ม) เพื่อ debug ง่าย
  config.consider_all_requests_local = true

  # cache หน้าเว็บ ปิดไว้ตอนพัฒนา เพื่อให้เห็นผลการแก้โค้ดทันทีโดยไม่ต้อง clear cache เอง
  config.cache_classes = false
end
```

```ruby
# config/environments/production.rb (บางส่วน)
Rails.application.configure do
  # ในโหมด production ไม่ควรให้ผู้ใช้เห็น stack trace ของ error (เป็นความเสี่ยงด้านความปลอดภัย)
  config.consider_all_requests_local = false

  # เปิด caching เต็มรูปแบบเพื่อประสิทธิภาพสูงสุด
  config.cache_classes = true
end
```

จะเจาะลึกการปรับแต่ง config แต่ละ environment แบบเต็มรูปแบบใน **Part 074**

### ไฟล์อื่นๆ ใน `config/` ที่ควรรู้จักชื่อไว้ก่อน (ยังไม่ต้องเข้าใจลึก)

| ไฟล์ | หน้าที่คร่าวๆ | Part ที่เจาะลึก |
|---|---|---|
| `config/application.rb` | การตั้งค่ากลางของทั้งแอป ไม่แยกตาม environment | Part 074 |
| `config/credentials.yml.enc` | เก็บ secret/API key แบบเข้ารหัส | Part 074, 081 |
| `config/storage.yml` | การตั้งค่า Active Storage (เก็บไฟล์อัปโหลดที่ไหน) | Part 066 |
| `config/puma.rb` | การตั้งค่า Puma web server | Part 077 |
| `config/importmap.rb` | รายการ JavaScript package ที่ pin ไว้ (โหมด importmap) | Part 051 |

---

## Step 207: รัน Development Server ด้วย `bin/rails server` และ `bin/dev`

ถึงเวลาเห็นผลลัพธ์เป็นหน้าเว็บจริงครั้งแรก!

### รันด้วย `bin/rails server`

```bash
cd ~/ruby-course-workspace/part-021/myapp
bin/rails server
```

ผลลัพธ์ (ทดสอบจริงแล้ว):

```
=> Booting Puma
=> Rails 8.1.4 application starting in development
=> Run `bin/rails server --help` for more startup options
Puma starting in single mode...
* Puma version: 6.x.x
* Listening on http://127.0.0.1:3000
* Listening on http://[::1]:3000
Use Ctrl-C to stop
```

เปิด browser ไปที่ **`http://localhost:3000`** จะเห็นหน้า **"Ruby on Rails 8.1.4"** — หน้า
welcome เริ่มต้นของ Rails ที่แสดงโลโก้ Rails พร้อมข้อมูลเวอร์ชัน:

```html
<!-- ส่วนสำคัญของหน้า welcome (ทดสอบรันจริงแล้ว) -->
<title>Ruby on Rails 8.1.4</title>
...
<ul>
  <li><strong>Rails version:</strong> 8.1.4</li>
  <li><strong>Rack version:</strong> 3.2.7</li>
  <li><strong>Ruby version:</strong> ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]</li>
</ul>
```

หน้านี้จะปรากฏก็ต่อเมื่อยังไม่ได้กำหนด **root route** ไว้ใน `config/routes.rb` (สังเกตบรรทัด
`# root "posts#index"` ที่ถูก comment ไว้ใน Step 206) — เมื่อไหร่ที่กำหนด root route เอง หน้านี้
จะหายไปทันทีแล้วแสดงหน้าที่เรากำหนดแทน (จะทำจริงใน Part 022)

ลองทดสอบ endpoint ที่มีอยู่แล้วจาก Step 206 ด้วย (health check endpoint):

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:3000/up
# => HTTP 200
```

กด **`Ctrl+C`** ใน terminal ที่รัน server อยู่เพื่อหยุด server

### รันแบบ background (สำหรับทดสอบเร็วๆ)

```bash
# -d รันแบบ daemon (background), -p ระบุ port, -P ระบุตำแหน่งเก็บ process id
bin/rails server -d -p 3000 -P tmp/pids/server.pid

# หยุด server ที่รันแบบ background
kill $(cat tmp/pids/server.pid)
```

### `bin/dev` — ทางเลือกที่ครอบคลุมกว่าในโปรเจกต์ที่มี asset build process

```bash
cat bin/dev
```

```ruby
#!/usr/bin/env ruby
exec "./bin/rails", "server", *ARGV
```

ในโปรเจกต์ `--minimal` หรือโปรเจกต์ที่ยังไม่ได้เพิ่ม CSS framework แบบ Tailwind `bin/dev` จะเป็น
แค่ **wrapper บางๆ ที่เรียก `bin/rails server` ต่ออีกที** เท่านั้น (เหมือนที่ `bin/todo` ใน Part
020 เป็น entry point ที่เรียก logic จริงใน `lib/` ต่อ) แต่ถ้าเลือก `--css=tailwind` ตอน
`rails new` ไฟล์นี้จะถูกแก้ให้รันทั้ง Rails server **และ** กระบวนการ build CSS พร้อมกัน (ผ่าน
`Procfile.dev` และ gem `foreman`) — เป็นเหตุผลที่คู่มือ Rails สมัยใหม่มักแนะนำให้ใช้ `bin/dev`
แทน `bin/rails server` ตรงๆ เพราะมันคือ "จุดเริ่มต้นเดียว" ที่ครอบคลุมทุก process ที่แอปต้องการ
ตอนพัฒนา ไม่ว่าโปรเจกต์จะซับซ้อนแค่ไหน — สำหรับโปรเจกต์เรียบง่ายในหลักสูตรนี้ ทั้งสองคำสั่งจะให้
ผลเหมือนกันทุกประการ

```bash
./bin/dev
# ทำงานเหมือนกับ bin/rails server ทุกประการในโปรเจกต์ --minimal
```

---

## Step 208: Rails Console (`bin/rails console`) — irb ที่รู้จักแอปทั้งแอปของเรา

ทบทวนจาก **Part 001 Step 5**: `irb` คือ REPL สำหรับทดลองรันโค้ด Ruby แบบ interactive — **Rails
console คือ `irb` ตัวเดียวกันเป๊ะ เพียงแค่โหลดทั้งแอปพลิเคชันของเรา (Model ทุกตัว, config ทุกอย่าง,
gem ทุกตัวที่อยู่ใน `Gemfile`) เข้ามาให้พร้อมใช้งานตั้งแต่บรรทัดแรก**

```bash
bin/rails console
```

```
Loading development environment (Rails 8.1.4)
irb(main):001>
```

ลองทดสอบดู (ทดสอบจริงแล้ว):

```irb
irb(main):001> Rails.env
=> "development"

irb(main):002> Rails.application.class.name
=> "Myapp::Application"

irb(main):003> 21 * 2
=> 42

irb(main):004> exit
```

**สิ่งที่พิเศษกว่า `irb` ธรรมดา:** เมื่อสร้าง Model ในอนาคต (Part 025) จะสามารถพิมพ์
`Book.all`, `Book.create(title: "...")`, `Book.find(1)` ได้ทันทีจากใน console นี้เลย โดยไม่ต้อง
`require` อะไรเพิ่มเติมสักบรรทัดเดียว เพราะ Rails console โหลด environment ทั้งหมดของแอปให้
อัตโนมัติผ่านไฟล์ `config/environment.rb` — เทียบได้กับตอนที่เปิด `irb` แล้วต้อง
`require_relative "lib/todo_cli"` เองก่อนถึงจะเรียก `TodoCli::TaskList.new` ได้ใน Part 020
ส่วน Rails console **ทำขั้นตอนนั้นให้อัตโนมัติทั้งหมด**

### shortcut ที่ใช้บ่อย

```bash
# ย่อ "console" เป็น "c" ได้
bin/rails c

# เปิด console โดยไม่ล้าง transaction ฐานข้อมูลอัตโนมัติ (sandbox mode)
# ทุกอย่างที่แก้ไขข้อมูลใน console จะถูก rollback ทันทีที่ exit — ปลอดภัยมากเวลาทดลองกับข้อมูลจริง
bin/rails console --sandbox
```

`--sandbox` จะมีประโยชน์มากตอนเรียน ActiveRecord ใน Part 025 เป็นต้นไป เพราะสามารถลองสร้าง/ลบ
ข้อมูลเล่นได้อย่างอิสระโดยไม่ต้องกังวลว่าข้อมูลทดสอบจะไปปนกับข้อมูลจริงในฐานข้อมูล development

---

## Step 209: Environment (development/test/production), `RAILS_ENV`, และ `bin/` vs `bundle exec`

### Environment คืออะไร และทำไมต้องมี 3 แบบ

Rails แยกการทำงานออกเป็น **environment** (สภาพแวดล้อมการทำงาน) 3 แบบตั้งแต่สร้างโปรเจกต์:

| Environment | ใช้ตอนไหน | ฐานข้อมูล | ลักษณะเด่น |
|---|---|---|---|
| **`development`** | ตอนเขียนโค้ด/พัฒนาฟีเจอร์ในเครื่องตัวเอง | `storage/development.sqlite3` | เห็น error แบบละเอียด, ไม่ cache เพื่อให้เห็นผลแก้โค้ดทันที |
| **`test`** | ตอนรัน automated test (Minitest/RSpec) | `storage/test.sqlite3` | ฐานข้อมูลถูกล้าง/สร้างใหม่บ่อยๆ ระหว่างรัน test |
| **`production`** | ตอนแอปทำงานจริงให้ผู้ใช้จริงเข้าถึง | `storage/production.sqlite3` | ปิดการแสดง error แบบละเอียด (ความปลอดภัย), เปิด caching เต็มรูปแบบ |

การแยก 3 สภาพแวดล้อมนี้คือคำตอบของคำถามที่ค้างมาจาก Step 206: **ทำไม `config/database.yml`
ต้องมีฐานข้อมูลแยกกันคนละไฟล์** — เพราะถ้าใช้ฐานข้อมูลเดียวกันทั้งพัฒนาและทดสอบ ทุกครั้งที่รัน
test suite แล้ว test ไป `delete` ข้อมูลทิ้งเพื่อทดสอบ ข้อมูลที่กำลังทดลองเล่นอยู่ตอนพัฒนาก็จะหาย
ไปด้วย — การแยก environment คือกลไกป้องกันปัญหานี้ตั้งแต่ระดับโครงสร้าง

### `RAILS_ENV` — ตัวแปรที่ควบคุมว่าใช้ environment ไหน

```bash
# ค่า default เมื่อไม่ระบุ RAILS_ENV คือ "development" เสมอ
bin/rails console
# Loading development environment (Rails 8.1.4)

# ระบุ environment ผ่าน environment variable (ทบทวนแนวคิด ENV จาก Part 012)
RAILS_ENV=test bin/rails console
```

ทดสอบจริง — สังเกตว่า `Rails.env` เปลี่ยนตาม `RAILS_ENV` ทันที:

```irb
# RAILS_ENV=test bin/rails console
Loading test environment (Rails 8.1.4)
irb(main):001> Rails.env
=> "test"
```

คำสั่งเดียวกัน แค่เปลี่ยนค่า `RAILS_ENV` ก็ทำให้ Rails โหลด `config/environments/test.rb` และ
เชื่อมต่อ `storage/test.sqlite3` แทนที่จะเป็นของ `development` ทันที — คำสั่ง Rails หลายตัวที่
จะเจอในอนาคต เช่น `bin/rails db:migrate RAILS_ENV=production` (ใช้ตอน deploy จริง) ก็ใช้หลักการ
เดียวกันนี้เป๊ะๆ

> **ข้อควรระวัง:** ตอน deploy ขึ้น production server จริง ต้องตั้งค่า `RAILS_ENV=production`
> ให้ระบบเสมอ (ปกติตั้งเป็น environment variable ถาวรไว้ที่ server) ไม่ใช่พิมพ์นำหน้าคำสั่งทุกครั้ง
> ด้วยมือ — ถ้าลืมตั้งและปล่อยให้ Rails รันด้วยค่า default (`development`) บน production server
> จริง จะเจอปัญหาใหญ่ทันที (ไม่มี caching, เปิดเผย stack trace ให้ผู้ใช้เห็น เป็นความเสี่ยงด้าน
> ความปลอดภัยตามที่ Step 206 พูดถึง) เรื่องนี้จะเจาะลึกอีกครั้งใน **Part 074**

### `bin/` binstub vs `bundle exec rails` — ควรใช้ตัวไหน

ทบทวนจาก **Part 001 Step 4**: ต้องใช้ `bundle exec` นำหน้าคำสั่งเสมอ เพื่อการันตีว่าใช้เวอร์ชัน
gem ตรงตาม `Gemfile.lock` — คำถามที่ตามมาคือ แล้ว **`bin/rails`** ที่เจอมาตลอด Part นี้ต่างจาก
`bundle exec rails` ตรงไหน?

```bash
# ทั้งสองคำสั่งนี้ให้ผลลัพธ์เหมือนกันทุกประการ (ทดสอบจริงแล้ว — diff ไม่มีความต่าง)
bin/rails -v
bundle exec rails -v
# => Rails 8.1.4  (ทั้งคู่)
```

**`bin/rails` คือ binstub** — สคริปต์ wrapper สั้นๆ ที่ Rails สร้างไว้ให้ตั้งแต่ `rails new`
มีหน้าที่**เดียวกันกับ `bundle exec rails` ทุกประการ** เพียงแค่พิมพ์สั้นกว่า:

```ruby
# bin/rails (เนื้อหาจริงของไฟล์นี้ — ทดสอบจากโปรเจกต์จริง)
#!/usr/bin/env ruby
APP_PATH = File.expand_path("../config/application", __dir__)
require_relative "../config/boot"
require "rails/commands"
```

สังเกตบรรทัด `require_relative "../config/boot"` — ไฟล์ `config/boot.rb` คือจุดที่ตั้งค่า
`Bundler.setup` ให้อัตโนมัติ (ถ้าเปิดไฟล์นั้นดูจะเห็น `require "bundler/setup"`) นั่นหมายความว่า
**`bin/rails` ทำหน้าที่ของ `bundle exec` ให้เราโดยอัตโนมัติอยู่แล้วในตัว** — จึงไม่จำเป็นต้องพิมพ์
`bundle exec` นำหน้า `bin/rails` อีกชั้นหนึ่ง (พิมพ์ซ้อนได้แต่ไม่มีประโยชน์อะไรเพิ่ม)

| คำสั่ง | ใช้ได้ไหม | หมายเหตุ |
|---|---|---|
| `bin/rails server` | ✅ แนะนำ — สั้นกว่า และรับประกัน `bundle exec` ให้อัตโนมัติ | ใช้เป็นค่าเริ่มต้นตลอดหลักสูตรนี้ |
| `bundle exec rails server` | ✅ ใช้ได้ ผลลัพธ์เหมือนกัน | ยาวกว่า แต่ชัดเจนว่ากำลังทำอะไรสำหรับมือใหม่ |
| `rails server` (ไม่มี `bin/` และไม่มี `bundle exec`) | ⚠️ ใช้ได้ในโปรเจกต์ส่วนใหญ่ แต่**เสี่ยง** | อาจไปเรียก Rails เวอร์ชัน global ของเครื่องแทนเวอร์ชันที่ล็อกไว้ใน `Gemfile.lock` ถ้าเครื่องมีหลายเวอร์ชันติดตั้งอยู่ |

**กฎปฏิบัติที่ใช้ตลอดหลักสูตรนี้จากนี้ไป:** ใช้ `bin/rails <คำสั่ง>` เป็นค่าเริ่มต้นเสมอ (เช่น
`bin/rails server`, `bin/rails console`, `bin/rails generate`) เพราะสั้นกว่า `bundle exec rails`
และให้ผลลัพธ์เดียวกันทุกประการ ส่วน `bin/rake`/`rake` ก็ใช้หลักการเดียวกัน (`bin/rake` ก็เป็น
binstub ที่ทำหน้าที่แทน `bundle exec rake`)

---

## Step 210: แบบฝึกหัด — สร้างแอป Rails แรกของคุณเอง (`bookshelf`) ตั้งแต่ศูนย์

ถึงเวลาลงมือทำเองตั้งแต่ต้นจนจบ ไม่มีโค้ดให้ copy-paste — ทำตามขั้นตอนแล้วสังเกตผลลัพธ์ทุกจุด

### โจทย์

1. สร้างแอป Rails ใหม่ชื่อ `bookshelf` แบบ `--minimal` (เพื่อให้เห็นโครงสร้างที่กระชับที่สุด)
2. สำรวจโครงสร้างโฟลเดอร์ที่ได้ เทียบกับที่เรียนใน Step 204–205 ว่าตรงกันไหม และมีอะไรที่
   **ไม่ถูกสร้าง** เพราะใช้ `--minimal` บ้าง
3. รัน development server แล้วเปิดดูหน้า welcome ผ่าน browser (หรือ `curl`)
4. เปิด Rails console แล้วลองสั่งคำสั่งพื้นฐานสัก 2–3 คำสั่งดู
5. ทดสอบสลับ `RAILS_ENV` แล้วสังเกตว่า `Rails.env` เปลี่ยนตาม

### เฉลย

**ขั้นตอนที่ 1: สร้างแอป**

```bash
cd ~/ruby-course-workspace/part-021
rails new bookshelf --minimal
cd bookshelf
```

ผลลัพธ์ (ทดสอบจริงแล้ว — ใช้เวลาเพียงไม่กี่วินาที เพราะ `--minimal` มี gem ให้ติดตั้งน้อยกว่ามาก):

```
      create  config/application.rb
      ...
      remove  app/jobs
      remove  app/mailers
      remove  app/javascript/channels
         run  bundle install --quiet
Resolving dependencies...
```

**ขั้นตอนที่ 2: สำรวจโครงสร้าง**

```bash
find app -maxdepth 2 | sort
```

ผลลัพธ์จริง — สังเกตว่า**ไม่มี** `app/jobs/`, `app/mailers/`, และ `app/javascript/channels/`
เทียบกับโปรเจกต์แบบเต็มใน Step 205:

```
app
app/assets
app/assets/images
app/assets/stylesheets
app/controllers
app/controllers/application_controller.rb
app/controllers/concerns
app/helpers
app/helpers/application_helper.rb
app/models
app/models/application_record.rb
app/models/concerns
app/views
app/views/layouts
app/views/pwa
```

ตรวจสอบ `Gemfile` เทียบกับ Step 203 — จะเห็นว่ามีแค่ gem ที่จำเป็นที่สุด (`rails`, `propshaft`,
`sqlite3`, `puma`, `debug`) ไม่มี Hotwire, Active Job adapter, หรือ deploy-related gem ใดๆ

**ขั้นตอนที่ 3: รัน server**

```bash
bin/rails server -d -p 3000 -P tmp/pids/server.pid
sleep 3

curl -s -o /dev/null -w "หน้าแรก: HTTP %{http_code}\n" http://localhost:3000/
curl -s -o /dev/null -w "health check: HTTP %{http_code}\n" http://localhost:3000/up
```

ผลลัพธ์ (ทดสอบจริงแล้ว):

```
หน้าแรก: HTTP 200
health check: HTTP 200
```

ทั้งสอง endpoint ตอบ `200 OK` — ยืนยันว่าแอปที่สร้างด้วย `--minimal` ยัง**ทำงานได้สมบูรณ์เต็ม
รูปแบบ** ไม่ได้ขาดอะไรที่จำเป็นสำหรับการเรียน MVC พื้นฐานเลย เพียงแค่ตัดฟีเจอร์ขั้นสูงที่ยังไม่ได้
เรียนออกไปเท่านั้น

```bash
# หยุด server
kill $(cat tmp/pids/server.pid)
```

**ขั้นตอนที่ 4: Rails console**

```bash
bin/rails console
```

```irb
irb(main):001> Rails.env
=> "development"

irb(main):002> Rails.application.class.name
=> "Bookshelf::Application"

irb(main):003> Rails.root
=> #<Pathname:/home/user/ruby-course-workspace/part-021/bookshelf>

irb(main):004> 5.times { |i| puts "หนังสือเล่มที่ #{i + 1}" }
หนังสือเล่มที่ 1
หนังสือเล่มที่ 2
หนังสือเล่มที่ 3
หนังสือเล่มที่ 4
หนังสือเล่มที่ 5
=> 5

irb(main):005> exit
```

สังเกตว่า `Rails.application.class.name` คืนค่า `"Bookshelf::Application"` — Rails ตั้งชื่อ
module หลักของแอปตามชื่อโปรเจกต์ที่ตั้งตอน `rails new` โดยอัตโนมัติ (แปลง `bookshelf` เป็น
`Bookshelf` แบบ CamelCase) และคำสั่ง Ruby ธรรมดาอย่าง `5.times { ... }` ก็ใช้งานได้ปกติทุกอย่าง
เพราะ Rails console ก็คือ irb ที่โหลด environment ของแอปเพิ่มเข้ามาเท่านั้นเอง ไม่ได้เปลี่ยน
พฤติกรรมพื้นฐานของภาษา Ruby แต่อย่างใด

**ขั้นตอนที่ 5: ทดสอบสลับ environment**

```bash
RAILS_ENV=test bin/rails runner 'puts Rails.env'
# => test

RAILS_ENV=production bin/rails runner 'puts Rails.env'
# => production

bin/rails runner 'puts Rails.env'
# => development   (ค่า default เมื่อไม่ระบุ RAILS_ENV)
```

> **เกร็ดเพิ่มเติม:** `bin/rails runner '<โค้ด Ruby>'` เป็นอีกวิธีหนึ่งในการรันโค้ด Ruby ภายใต้
> environment ของแอป โดยไม่ต้องเปิด console แบบ interactive — เหมาะกับการเขียน script สั้นๆ
> หรือเรียกจาก cron job/CI pipeline (จะเจาะลึกการใช้งานร่วมกับ background job ใน Phase 9)

**สรุปผลการทดลอง:** แอป `bookshelf` ที่สร้างด้วย `--minimal` ทำงานได้ครบทุกอย่างที่จำเป็น —
สร้างโปรเจกต์เร็ว, โครงสร้างโฟลเดอร์กระชับตรงตาม convention, server รันได้, console ใช้งานได้,
และสลับ environment ได้ถูกต้องตามที่ออกแบบไว้ — พร้อมแล้วสำหรับการเพิ่ม Model/Controller/View
จริงในเนื้อหา Part ถัดไป

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เปรียบเทียบ `--api` กับแบบปกติ** — สร้างโปรเจกต์ใหม่ชื่อ `bookshelf_api` ด้วย
   `rails new bookshelf_api --api --minimal` แล้วเปรียบเทียบ `find app -maxdepth 2` กับผลลัพธ์
   ของ `bookshelf` ใน Step 210 — โฟลเดอร์ไหนหายไปบ้าง? เปิดดู
   `app/controllers/application_controller.rb` ว่าสืบทอดจาก class อะไรต่างจากเดิม
2. **ลองรัน `bin/rails -T` เต็มรูปแบบ** — ในโปรเจกต์ `bookshelf` รันคำสั่ง `bin/rails -T` แล้ว
   scroll ดู task ทั้งหมดที่ Rails ลงทะเบียนไว้ให้อัตโนมัติ (มีเป็นร้อยตัว) ลองหา task ที่ขึ้นต้น
   ด้วย `db:` (จะได้เรียนละเอียดใน Part 025) และ `assets:` (จะได้เรียนใน Part 029) มาดูคำอธิบาย
   ของแต่ละตัวว่าพอเดาได้ไหมว่ามันทำอะไร ทบทวนแนวคิด `desc`/`namespace` จาก Part 020 ไปพร้อมกัน
3. **ทดลองสร้างโปรเจกต์ด้วย `--css=tailwind`** — สร้างโปรเจกต์ใหม่ชื่อ `bookshelf_ui` ด้วย
   `rails new bookshelf_ui --css=tailwind --minimal` แล้วเปิดดูไฟล์ `bin/dev` เทียบกับ `bin/dev`
   ของ `bookshelf` (แบบไม่มี CSS framework) ว่าเนื้อหาต่างกันอย่างไร และลองหาไฟล์
   `Procfile.dev` ที่ปรากฏขึ้นมาใหม่ — มันสั่งรัน process อะไรบ้าง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Rails คือ Framework ที่เขียนด้วย Ruby ล้วนๆ และวางอยู่บน Rack** ไม่ใช่ภาษาใหม่หรือ
  วิธีคิดที่แยกขาดจาก Ruby ที่เรียนมาตลอด Phase 1–2
- เข้าใจภาพรวมของ **MVC** (Model-View-Controller) แบบ Bird's-eye View และเห็นว่าโปรเจกต์
  Todo CLI (Part 020) มีโครงสร้างที่ใกล้เคียงกับ MVC อยู่แล้วตั้งแต่ต้น เพียงแค่ `CLI` ทำหน้าที่
  ทั้ง Controller และ View รวมกัน
- ติดตั้ง Rails ด้วย `gem install rails` และตรวจสอบเวอร์ชันที่ติดตั้งได้
- สร้างโปรเจกต์ใหม่ด้วย `rails new` พร้อมรู้จัก flag สำคัญ: `--minimal`, `--api`,
  `--database=postgresql`, `--css=tailwind`
- อ่านและเข้าใจหน้าที่ของ**โฟลเดอร์ระดับบนสุดทุกตัว** (`app/`, `config/`, `db/`, `lib/`, `test/`,
  `bin/`, `public/`) และเหตุผลเบื้องหลัง **Convention over Configuration**
- เจาะลึกโฟลเดอร์ย่อยใน `app/` ทั้งหมด (models, views, controllers, helpers, jobs, mailers,
  channels) พร้อมรู้ว่าตัวไหนจะเจาะลึกใน Part ไหนต่อไป
- เห็นภาพรวมของ `config/routes.rb`, `config/database.yml`, `config/environments/*.rb` แบบ
  คร่าวๆ ก่อนที่จะเจาะลึกทีละไฟล์ในเฟส 3 นี้
- รัน **Development Server** ได้ทั้งผ่าน `bin/rails server` และ `bin/dev` แล้วเห็นหน้า welcome
  ของ Rails ผ่าน `localhost:3000` จริง
- ใช้ **Rails Console** (`bin/rails console`) เป็น irb ที่โหลดทั้งแอปของเราให้พร้อมใช้งาน
- เข้าใจแนวคิด **environment** (`development`/`test`/`production`) และตัวแปร `RAILS_ENV`
  รวมถึงเหตุผลที่ `bin/rails` ทำหน้าที่แทน `bundle exec rails` ได้โดยอัตโนมัติ
- ลงมือสร้างแอป Rails ตัวแรกด้วยตัวเองตั้งแต่ศูนย์ (`bookshelf`) ครบทุกขั้นตอน ตั้งแต่ `rails new`
  จนถึงการรัน server และเปิด console

**ต่อไป (Part 022):** ตอนนี้รู้จักโครงสร้างทั้งหมดของ Rails application แล้ว ถึงเวลาเจาะลึกจุด
แรกที่ HTTP request ทุกตัวต้องผ่าน — **Routing** เราจะเรียนวิธีเขียน `config/routes.rb` แบบเต็ม
รูปแบบ ทั้ง `get`/`post`/`patch`/`delete` แบบ manual, การใช้ `resources` เพื่อสร้าง **RESTful
route** ทั้ง 7 เส้นทางมาตรฐานในบรรทัดเดียว, และ **route helper** อย่าง `books_path`/
`book_path(1)` ที่ทำให้ไม่ต้องเขียน URL เป็น string ตรงๆ ในโค้ดอีกต่อไป — จุดเริ่มต้นของการทำให้
`config/routes.rb` ที่ว่างเปล่าตอนนี้ กลายเป็นแผนที่ของเว็บแอปพลิเคชันจริง
