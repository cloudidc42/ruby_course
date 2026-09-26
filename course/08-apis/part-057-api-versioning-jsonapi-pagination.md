# Part 057: API Versioning, JSON:API Spec, และ Pagination สำหรับ API

> **Step ครอบคลุมใน Part นี้:** Step 561–570
> **ระดับ:** สูง (ต้องผ่าน Part 056 เรื่อง Rails API-only mode/JSON response/serializer พื้นฐาน
> และ Part 038 เรื่อง Pagination ด้วย pagy มาก่อน — Part นี้ต่อยอดกลไก `pagy(collection)` ที่
> Part 038 สอนไว้สำหรับหน้าเว็บ HTML มาปรับใช้กับ JSON API โดยเฉพาะ)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x โหมด `--api` / pagy 9.4.x (extras: `keyset`,
> `headers`, `metadata`) / jsonapi-serializer 2.2.x (ทุกคำสั่ง, JSON response, และ HTTP header
> ในเอกสารนี้รันจริงและ capture จริงบน Ruby 3.3.6 + Rails 8.1.4 + SQLite3)

Part 056 สอนให้สร้าง Rails API-only app ที่ตอบกลับเป็น JSON ได้แล้ว — `render json: post`,
serializer เบื้องต้น, structure ของ `app/controllers` ที่ inherit จาก `ActionController::API`
แทน `ActionController::Base` Part นี้พาไปดูคำถามที่ทีมพัฒนา API ทุกทีมต้องเจอทันทีที่มี
**client ภายนอกจริงๆ มาเรียกใช้ API** (ไม่ใช่แค่ frontend ของทีมตัวเอง): ถ้าต้องเปลี่ยน response
shape ในอนาคต จะทำอย่างไรไม่ให้ client เดิมพัง? มี "มาตรฐาน" อะไรบ้างสำหรับ shape ของ JSON
response ที่ทำให้ client tooling (SDK, codegen, Postman collection) ทำงานร่วมกับ API ได้ง่ายขึ้น?
และ pagination แบบที่ใช้ `pagy_nav`/`pagy_info` render เป็น HTML ใน Part 038 นั้น — เมื่อไม่มี
HTML ให้ render แล้ว จะส่งข้อมูลการแบ่งหน้าให้ client ในรูปแบบไหนถึงจะถูกต้องและปลอดภัยจริงๆ

ทั้งสามเรื่องนี้ (versioning, JSON:API, pagination) ดูเหมือนแยกกัน แต่จริงๆ เกี่ยวโยงกันแน่นมาก:
**การเปลี่ยนรูปแบบ pagination metadata ก็คือ breaking change ชนิดหนึ่งที่ต้อง versioning
รองรับ** และ **JSON:API spec เองก็นิยาม convention ของ pagination metadata ไว้ให้แล้วบางส่วน**
(`links.next`, `meta`) — Part นี้จะร้อยทั้งสามเรื่องเข้าด้วยกันผ่านโปรเจกต์ตัวอย่างเดียว: API
รายการบทความ (`Post`) ที่มี `Author`/`Category` เหมือน Part 034/038 แต่คราวนี้เป็น JSON API
ล้วนๆ ไม่มี view/HTML เข้ามาเกี่ยวข้องเลย

## สารบัญของ Part นี้

- Step 561: ทำไม API versioning ถึงสำคัญ — Breaking Change กับ Mobile Client ที่ Force-update
  ไม่ได้
- Step 562: เปรียบเทียบกลยุทธ์ Versioning — URL Path vs Accept Header vs Custom Header (พร้อม
  Trade-off จริงที่ทดสอบได้)
- Step 563: Implement URL-path Versioning — `namespace`, `Api::V1::BaseController`,
  `Api::V1::PostsController`
- Step 564: JSON:API Specification คืออะไร — `data`/`attributes`/`relationships`/`included`/
  `meta`/`links`
- Step 565: Implement JSON:API Response ด้วยมือ แล้ว Refactor ด้วย `jsonapi-serializer`
- Step 566: Offset-based vs Cursor-based Pagination — ปัญหา Consistency ที่เกิดขึ้นจริงเมื่อมี
  Concurrent Write
- Step 567: Cursor-based Pagination ด้วย pagy (`extras/keyset`) + Pagination Metadata + Link
  Header แบบ GitHub
- Step 568: เขียนสัญญา (Contract) ของ Pagination ให้ API Consumer อ่านและพึ่งพาได้
- Step 569: HATEOAS คืออะไร — และควรใช้จริงในระบบมากแค่ไหน
- Step 570: Content Negotiation — `Accept`/`Content-Type`, `respond_to` รองรับทั้ง JSON ธรรมดา
  และ JSON:API

---

## เตรียมโปรเจกต์สำหรับ Part นี้

สร้าง Rails app โหมด `--api` ใหม่ (ต่างจาก Part 001–055 ที่สร้างแบบเต็ม เพราะ API ไม่ต้องการ
asset pipeline, view, cookie-based session ใดๆ เลย — Part 056 อธิบายความต่างของโหมดนี้ไว้แล้ว):

```bash
rails new blog_api --api --minimal
cd blog_api
```

ติดตั้ง gem ที่ใช้ตลอด Part นี้:

```ruby
# Gemfile
gem "pagy", "~> 9.4"
gem "jsonapi-serializer", "~> 2.2"
```

```bash
bundle install
```

> **หมายเหตุ:** `jsonapi-serializer` คือชื่อปัจจุบันของ gem ที่เดิมชื่อ `fast_jsonapi` (สร้างโดย
> ทีม Netflix แล้วหยุดดูแล) — community เข้ามารับช่วงดูแลต่อภายใต้ชื่อใหม่ namespace ในโค้ดยังคง
> เป็น `JSONAPI::Serializer` เหมือนเดิม ถ้าเจอโค้ด legacy ที่ `require "fast_jsonapi"` หรือ
> `include FastJsonapi::ObjectSerializer` ตรงๆ คือโค้ดที่เขียนก่อนเปลี่ยนชื่อ gem — ยังใช้งานได้
> เพราะ `jsonapi-serializer` ยังคง module เดิมไว้เป็น alias ภายใน

สร้าง model เดิมจาก Part 034/038 (`Author`, `Category`, `Post`) แต่ตัดทุกอย่างที่เกี่ยวกับ view
ออก เพราะ Part นี้ไม่มี HTML เลย:

```bash
bin/rails generate model Author name:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string body:text views:integer published:boolean \
  author:references category:references
```

แก้ migration ของ `Post` ให้มี default ระดับฐานข้อมูล (แนวทางเดียวกับ Part 026/034/038):

```ruby
# db/migrate/..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body
      t.integer :views, default: 0, null: false
      t.boolean :published, default: false, null: false
      t.references :author, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

ตั้งค่า association และ scope:

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :posts
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :posts
end

# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category

  scope :published, -> { where(published: true) }
end
```

```bash
bin/rails db:create db:migrate
```

Seed ข้อมูล 25 บทความ กระจาย 3 ผู้เขียนและ 3 หมวดหมู่ (แบบเดียวกับสูตรกระจายข้อมูลของ Part 038):

```ruby
# db/seeds.rb
# frozen_string_literal: true

authors = { "Nichada" => "Thai", "Somsak" => "Thai", "Wanda" => "American" }
  .map { |name, _nat| Author.create!(name: name) }

categories = %w[Ruby Rails DevOps].map { |n| Category.create!(name: n) }

25.times do |i|
  Post.create!(
    title: "บทความ ##{i + 1}",
    body: "เนื้อหาตัวอย่างของบทความลำดับที่ #{i + 1}",
    views: (i * 37) % 500,
    published: i % 5 != 0, # 4 ใน 5 เผยแพร่แล้ว เหมือนสูตรของ Part 038
    author: authors[i % authors.size],
    category: categories[i % categories.size]
  )
end

puts "Authors: #{Author.count}, Categories: #{Category.count}, " \
     "Posts: #{Post.count} (published: #{Post.published.count})"
```

```bash
bin/rails db:seed
# Authors: 3, Categories: 3, Posts: 25 (published: 20)
```

ตรวจสอบว่า id ของบทความที่เผยแพร่แล้วเป็นอย่างไร (จะใช้ตัวเลขชุดนี้อ้างอิงตลอด Part):

```bash
bin/rails runner 'puts Post.published.order(id: :asc).pluck(:id).inspect'
# [2, 3, 4, 5, 7, 8, 9, 10, 12, 13, 14, 15, 17, 18, 19, 20, 22, 23, 24, 25]
```

`config/routes.rb` เริ่มต้นแบบเปล่าๆ ก่อน (จะเพิ่ม route จริงใน Step 563):

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check
end
```

---

## Step 561: ทำไม API versioning ถึงสำคัญ — Breaking Change กับ Mobile Client ที่ Force-update ไม่ได้

เว็บแอปที่ render HTML (Part 001–055 ทั้งหมด) มีคุณสมบัติหนึ่งที่ทำให้ "การเปลี่ยนแปลง" ไม่ค่อย
น่ากลัว: **ทุกครั้งที่ผู้ใช้เปิดหน้าเว็บ browser จะโหลด HTML/CSS/JS ชุดล่าสุดจาก server เสมอ**
เปลี่ยน backend เมื่อไหร่ ผู้ใช้ก็เห็นผลทันทีโดยไม่ต้องทำอะไรเพิ่ม

**API ไม่มีคุณสมบัตินี้** ลองนึกภาพสถานการณ์นี้:

1. ทีมสร้าง endpoint `GET /posts/1` ที่ตอบกลับ `{ "id": 1, "views": 120 }`
2. แอปมือถือ (iOS/Android) เวอร์ชัน 1.0 ถูกปล่อยออกไป โดยโค้ดฝั่ง client เขียนไว้ว่า
   `let viewCount = response["views"] as! Int`
3. ผู้ใช้ดาวน์โหลดแอปเวอร์ชัน 1.0 ไปแล้ว **หลายแสนคน** ติดตั้งอยู่ในเครื่องจริง
4. ทีม backend ตัดสินใจว่าชื่อ field `views` กำกวม (สับสนกับ "จำนวนคนดูวันนี้" vs "ยอดสะสม")
   เลยเปลี่ยนชื่อเป็น `view_count` แล้ว deploy backend ใหม่ทันที

**เกิดอะไรขึ้น:** แอปมือถือเวอร์ชัน 1.0 ที่ติดตั้งอยู่ในเครื่องผู้ใช้หลายแสนเครื่องนั้น **ยังคง
เรียก field ชื่อ `views` เหมือนเดิม** เพราะโค้ดของแอปถูก compile ไปแล้วตั้งแต่ตอนปล่อยเวอร์ชัน —
`response["views"]` จะได้ `nil` ทันที ไม่ว่า Swift/Kotlin จะ crash แอปเลย (force unwrap `nil`)
หรือแสดงตัวเลข 0 ผิดๆ ให้ผู้ใช้เห็น **โดยที่ทีม backend ไม่มีทางรู้เลยจนกว่าจะมีคน report bug
หรือ crash analytics แจ้งเตือน**

จุดนี้คือความต่างสำคัญที่สุดระหว่างเว็บแอปกับ API:

| | เว็บแอป (render HTML) | Mobile App (เรียก API) |
|---|---|---|
| ใครควบคุมเวอร์ชันของ "หน้าตา" | Server (ส่ง HTML ใหม่ทุก request) | Client (binary ที่ติดตั้งในเครื่อง) |
| อัปเดตพร้อมกันได้ไหม | ได้ทันที deploy ปุ๊บเห็นปั๊บ | **ไม่ได้** — ต้องรอผู้ใช้อัปเดตแอปเอง |
| บังคับให้อัปเดตได้ไหม | ไม่ต้องบังคับ (ไม่มีแนวคิดนี้ด้วยซ้ำ) | บังคับได้แค่บางส่วน (force-update
  screen) แต่ **ทำใน App Store/Play Store ล่าช้าเป็นวัน** และผู้ใช้บางคนปิดอินเทอร์เน็ต/ไม่กดอัปเดต
  อยู่ดี |
| อายุของเวอร์ชันเก่าที่ยังใช้งานจริง | แทบเป็นศูนย์ (refresh หน้าเดียวจบ) | **หลักเดือนถึงหลักปี**
  — มีผู้ใช้บางส่วนที่ค้างอยู่เวอร์ชันเก่ามากเสมอ (เครื่องเก่า, ปิด auto-update, ไม่มีเน็ตตอนแอป
  เตือนอัปเดต) |

**สถานการณ์เดียวกันนี้เกิดกับ client ประเภทอื่นด้วย** ไม่ใช่แค่ mobile app: SPA เวอร์ชันเก่าที่
cache ไว้ใน browser ของผู้ใช้ที่ไม่ได้ refresh หน้า, third-party integration ที่ลูกค้าเขียนสคริปต์
เชื่อมต่อ API ของเราเองแล้วไม่มาอัปเดตโค้ดตามเรา, หรือ IoT device ที่ flash firmware ใหม่ยากมาก
— **ทุกกรณีมีจุดร่วมเดียวกัน: backend เปลี่ยนได้เร็วกว่า client เสมอ**

### กฎที่ตามมา: Breaking Change ต้องมี "ทางออก" ให้ Client เก่า

**Breaking change** คือการเปลี่ยนแปลงที่ทำให้ client เดิมที่เขียนโค้ดพึ่งพา response shape เดิม
**พังหรือทำงานผิดพลาด** ตัวอย่างที่เป็น breaking change เสมอ:

- เปลี่ยนชื่อ field (`views` → `view_count`)
- ลบ field ที่เคยมี
- เปลี่ยนชนิดข้อมูลของ field (`"views": 120` → `"views": "120"`)
- เปลี่ยนโครงสร้าง response ทั้งหมด (จาก flat object เป็น JSON:API shape ที่จะเรียนใน Step 564)
- เปลี่ยนความหมายของ field โดยไม่เปลี่ยนชื่อ (เช่น `views` เคยหมายถึง "ยอดสะสม" เปลี่ยนเป็น
  "ยอดวันนี้" แบบเงียบๆ — อันตรายที่สุดเพราะ client ไม่ error แต่ได้ข้อมูลผิด)

ส่วนสิ่งที่ **ไม่ใช่** breaking change (ทำได้อย่างปลอดภัยโดยไม่ต้อง version ใหม่):

- **เพิ่ม field ใหม่** เข้าไปใน response (client เก่าไม่รู้จัก field ใหม่ ก็แค่เพิกเฉยมันไป —
  นี่คือเหตุผลที่ควรออกแบบ client ให้ **อ่านเฉพาะ field ที่ต้องใช้** ไม่ใช่ deserialize ทั้ง object
  แบบ strict schema ที่ reject field แปลกปลอม)
- เพิ่ม endpoint ใหม่
- แก้บั๊กที่ทำให้ response ผิดจากที่ "ควรจะเป็น" ตาม documentation อยู่แล้ว (แต่ในทางปฏิบัติต้อง
  ระวังมาก เพราะ client บางตัวอาจ "พึ่งพาบั๊กนั้นโดยไม่รู้ตัว" — Hyrum's Law: "ระบบที่มีผู้ใช้มากพอ
  ทุกพฤติกรรมที่สังเกตได้จะถูกมีคนพึ่งพา ไม่ว่าจะตั้งใจให้เป็น contract หรือไม่")

**วิธีแก้ปัญหานี้ในทางปฏิบัติมีสองทาง:** (1) ไม่ทำ breaking change เลย พยายาม evolve API แบบ
backward-compatible เสมอ (เพิ่ม field ใหม่แทนเปลี่ยน field เดิม) หรือ (2) เมื่อจำเป็นต้อง breaking
change จริงๆ **ให้ client เก่ายังเรียก behavior เก่าต่อไปได้** ในขณะที่ client ใหม่เรียก behavior
ใหม่ — นี่คือหน้าที่ของ **API versioning** ซึ่งเป็นหัวข้อของ Part นี้

---

## Step 562: เปรียบเทียบกลยุทธ์ Versioning — URL Path vs Accept Header vs Custom Header

มีสามกลยุทธ์หลักที่ใช้กันจริงในวงการ แต่ละแบบมี trade-off ที่ต่างกันชัดเจน ไม่มีแบบไหน "ถูกที่สุด"
ในทุกกรณี — Part นี้จะ implement ทั้งสามแบบจริงเพื่อให้เห็นว่าทำงานต่างกันอย่างไร แล้วเลือกแบบ
**URL-path versioning เป็นกลยุทธ์หลัก** สำหรับ endpoint ที่เหลือทั้งหมดของ Part

### 1) URL-path Versioning — `/api/v1/posts`

ใส่เลขเวอร์ชันไว้ใน path ตรงๆ:

```
GET /api/v1/posts
GET /api/v2/posts
```

**ข้อดี:**
- เข้าใจง่ายที่สุด ไม่ต้องอธิบายอะไรเพิ่ม — เปิด URL ใน browser หรือ curl ธรรมดาก็ทดสอบได้ทันที
- **Cache ง่าย** เพราะ URL ต่างกันจริงๆ ทั้ง browser cache, CDN, reverse proxy (nginx/Varnish)
  มองเป็นคนละ resource โดยธรรมชาติ ไม่ต้องตั้งค่า `Vary` header เพิ่มเติม
- routing ทำได้ตรงไปตรงมาด้วย `namespace` ของ Rails (ดู Step 563)
- **ผู้ใช้ส่วนใหญ่ของวงการเลือกแบบนี้เป็นอันดับแรก** (Twitter/X API, Stripe บางส่วน, ระบบภายใน
  องค์กรจำนวนมาก) เพราะ debug/support ง่ายที่สุดเวลาลูกค้าแจ้งปัญหา ("เรียก v1 อยู่ใช่ไหม")

**ข้อเสีย:**
- ผิดหลัก REST ที่เคร่งครัด — ตามทฤษฎีแล้ว **URI ควรระบุ "resource" ไม่ใช่ "เวอร์ชันของ
  representation"** `/v1/posts/1` กับ `/v2/posts/1` ควรเป็น "บทความเดียวกัน" แต่ path ต่างกันทำให้
  ดูเหมือนเป็นคนละ resource
- Controller/code ซ้ำซ้อนมากขึ้นเมื่อมีหลายเวอร์ชันพร้อมกัน (`Api::V1::PostsController`,
  `Api::V2::PostsController` ที่ logic คล้ายกันแต่ shape ต่าง)

### 2) Accept Header Versioning (Media Type / Vendor Content Type)

ระบุเวอร์ชันผ่าน `Accept` header แทน โดยใช้ **vendor-specific media type**:

```
GET /api/posts
Accept: application/vnd.myapp.v1+json
```

**ข้อดี:**
- **ตรงหลัก REST มากกว่า** — URI เดียวกัน (`/api/posts`) แทน "resource" เดียวกันเสมอ ส่วน
  `Accept` header บอกแค่ "อยาก **ดู representation แบบไหน** ของ resource นั้น" ตรงตามเจตนาดั้งเดิม
  ของ HTTP content negotiation
- **GitHub API ใช้แนวทางนี้จริง** (`Accept: application/vnd.github.v3+json`) เป็นตัวอย่างที่
  เป็นที่รู้จักมากที่สุดของกลยุทธ์นี้ในโลกจริง
- ไม่ต้อง fork URL structure ทั้งระบบเมื่อมีเวอร์ชันใหม่

**ข้อเสีย:**
- **ทดสอบยากกว่ามาก** — เปิดผ่าน browser URL bar ตรงๆ ไม่ได้ ต้องใช้ curl/Postman ตั้ง header
  เอง ทีม frontend/QA ที่ไม่คุ้น HTTP header จะงงว่าทำไม URL เดียวกันได้ผลลัพธ์ต่างกัน
- **Cache ซับซ้อนขึ้น** — เพราะ URL เดียวกันให้ผลต่างกันตาม header ต้องตั้ง `Vary: Accept` ให้
  ถูกต้องเสมอ ไม่งั้น cache/CDN อาจส่ง response ของเวอร์ชันผิดกลับไปให้ client อื่น (ดูผลจริงใน
  header ด้านล่าง — Rails ใส่ `Vary: Accept` ให้อัตโนมัติเมื่อใช้ `respond_to`)
- ต้องคอยจำ media type string ที่ถูกต้องเป๊ะ พิมพ์ผิดนิดเดียวก็ไม่ match

### 3) Custom Header Versioning

สร้าง header เฉพาะกิจของตัวเองขึ้นมา:

```
GET /api/posts
X-API-Version: 1
```

**ข้อดี:**
- Implement ง่ายที่สุด (แค่เทียบ string) ไม่ต้องยุ่งกับ media type syntax ที่ซับซ้อน
- แยกเรื่อง "เวอร์ชันของ API" ออกจากเรื่อง "รูปแบบข้อมูล" (`Content-Type`/`Accept`) ชัดเจน — ในทาง
  ทฤษฎี `Accept` ควรใช้เพื่อบอก **format** (JSON/XML/JSON:API) ไม่ใช่ **เวอร์ชันของ API** ปนกัน
  (Stripe เลือกทางนี้จริง แต่ใช้ชื่อ header ว่า `Stripe-Version` พร้อมค่าเป็นวันที่ เช่น
  `2024-06-20` แทนเลขเวอร์ชันธรรมดา)

**ข้อเสีย:**
- **ไม่ใช่มาตรฐานสากล** — เป็น convention ที่แต่ละบริษัทคิดขึ้นเอง ผู้ใช้ API ต้องอ่าน
  documentation ก่อนถึงจะรู้ว่าต้องส่ง header ชื่อนี้
- **มองไม่เห็นจาก URL bar เหมือนกัน** และ CDN/proxy บางตัว **default ไม่ vary cache ตาม custom
  header** (ต้องตั้งค่า `Vary: X-API-Version` เองอย่างชัดเจน ไม่ได้มาให้ฟรีเหมือน `Accept` ที่
  framework ส่วนใหญ่รู้จักอยู่แล้ว)

### ทดสอบทั้งสามกลยุทธ์จริงในเวลาเดียวกัน

Part นี้ implement ทั้งสามกลยุทธ์ไว้ในโปรเจกต์เดียวเพื่อพิสูจน์ว่าทำงานได้จริงทั้งหมด (แม้จะเลือก
URL-path เป็นหลักในการสอน Step ที่เหลือ) — เขียน routing constraint สองตัวใน `config/routes.rb`:

```ruby
# config/routes.rb
# frozen_string_literal: true

# --- Route constraint สาธิต "Accept header versioning" ---
class AcceptHeaderVersion
  def initialize(version)
    @version = version
  end

  def matches?(request)
    request.headers["Accept"].to_s.include?("application/vnd.myapp.v#{@version}+json")
  end
end

# --- Route constraint สาธิต "Custom header versioning" ---
class CustomHeaderVersion
  def initialize(version)
    @version = version.to_s
  end

  def matches?(request)
    request.headers["X-API-Version"].to_s == @version
  end
end

Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  # กลยุทธ์หลัก: URL-path versioning (implement เต็มรูปแบบใน Step 563)
  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show]
    end
  end

  # เส้นทางเดียวกัน (/api/posts) แต่ dispatch ด้วย Accept header แทน path
  get "api/posts", to: "api/v1/posts#index", constraints: AcceptHeaderVersion.new(1)

  # เส้นทางเดียวกัน (/api/posts) แต่ dispatch ด้วย custom header แทน path
  get "api/posts", to: "api/v1/posts#index", constraints: CustomHeaderVersion.new(1)
end
```

**อธิบาย:** Rails routing รองรับ `constraints:` เป็น **object ใดๆ ที่ตอบสนอง method
`matches?(request)`** (ไม่จำเป็นต้องเป็น regex หรือ hash เท่านั้น) — ถ้า `matches?` คืนค่า falsy
route นั้นจะถูกข้ามไปเหมือนไม่มีอยู่ ทำให้ Rails มองหา route ถัดไปที่ตรงกัน (ในที่นี้คือ
`CustomHeaderVersion`) ถ้าไม่มี route ไหน match เลย ก็จะได้ 404 ตามปกติ — สังเกตว่า **สองเส้นทาง
ที่ path ซ้ำกันเป๊ะ (`api/posts`) วางต่อกันได้เลย** เพราะ constraint ทำหน้าที่แยกความกำกวมให้

ทดสอบทั้งสามกลยุทธ์ด้วย curl จริง (endpoint `Api::V1::PostsController#index` จะ implement เต็ม
ใน Step 563 แต่ routing ทำงานถูกต้องได้ทันทีแม้ controller ยังไม่สมบูรณ์):

```bash
# 1) URL-path versioning (กลยุทธ์หลัก)
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Accept: application/vnd.api+json" "http://localhost:3000/api/v1/posts"
# 200

# 2) Accept-header versioning ตรงเวอร์ชัน
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Accept: application/vnd.myapp.v1+json" "http://localhost:3000/api/posts"
# 200

# Accept header ไม่ match เวอร์ชันที่มี route รองรับ -> ไม่มี route ไหน match เลย
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Accept: application/json" "http://localhost:3000/api/posts"
# 404

# 3) Custom header versioning ตรงเวอร์ชัน
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "X-API-Version: 1" -H "Accept: application/json" "http://localhost:3000/api/posts"
# 200

# Custom header ผิดเวอร์ชัน
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "X-API-Version: 2" -H "Accept: application/json" "http://localhost:3000/api/posts"
# 404

# ไม่ส่ง version header ใดๆ เลย -> ไม่มี route ไหน match
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Accept: application/json" "http://localhost:3000/api/posts"
# 404
```

ทั้งสามกลยุทธ์ทำงานถูกต้องตามที่ออกแบบไว้จริง — เห็นความต่างชัดเจนว่า URL-path versioning **แยก
เวอร์ชันด้วยตัว URL เอง** ในขณะที่อีกสองแบบ **แยกด้วย request header บน URL เดียวกัน**

> **ทำไม Part นี้เลือก URL-path เป็นกลยุทธ์หลักสำหรับ Step ที่เหลือ:** เหตุผลเดียวกับที่ Part 038
> เลือก pagy เป็นตัวหลัก (Step 374) — ไม่มีแบบไหนถูกที่สุดในทุกกรณี แต่ URL-path เป็นแบบที่
> **debug ง่ายที่สุด, ทดสอบง่ายที่สุดด้วย curl/browser ธรรมดา, และเป็นแบบที่เจอบ่อยที่สุดในงาน
> จริงระดับเริ่มต้น-กลาง** ส่วน Accept-header versioning แบบ GitHub เหมาะกับทีมที่ยึดหลัก REST
> อย่างเคร่งครัดและมี client tooling ที่จัดการ header ได้ดีอยู่แล้ว — **ทั้งสามแบบยัง implement
> อยู่ในโปรเจกต์นี้ต่อไป** เพื่อให้เห็นว่าเลือกใช้แบบไหนก็ไม่มีผลต่อ business logic ข้างในเลย
> (ทุกเส้นทางชี้ไปที่ `Api::V1::PostsController` ตัวเดียวกัน)

---

## Step 563: Implement URL-path Versioning — `namespace`, `Api::V1::BaseController`, `Api::V1::PostsController`

การจัด namespace ให้ถูกต้องตั้งแต่ต้นสำคัญมาก เพราะโปรเจกต์จริงมักมีหลายเวอร์ชันพร้อมกันอยู่เสมอ
(v1 ยังมี client ใช้อยู่ ในขณะที่กำลังพัฒนา v2 คู่ขนาน)

### Routing

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show]
    end
  end
end
```

```bash
bin/rails routes | grep posts
```

```
   api_v1_posts GET  /api/v1/posts(.:format)     api/v1/posts#index
    api_v1_post GET  /api/v1/posts/:id(.:format) api/v1/posts#show
```

`namespace :api do namespace :v1 do ... end end` ทำสองอย่างพร้อมกัน: (1) เติม prefix `/api/v1`
เข้า URL และ (2) **คาดหวังว่า controller อยู่ใน module `Api::V1`** (`app/controllers/api/v1/
posts_controller.rb` class `Api::V1::PostsController`) — ตรงกับหลัก "ตั้งชื่อไฟล์/class ตาม
convention แล้ว Rails หาเจอเอง" ที่เจอมาตั้งแต่ Part 001

### `Api::V1::BaseController` — จุดรวม behavior ที่ทุก controller ในเวอร์ชันนี้ใช้ร่วมกัน

แยก base controller เฉพาะของแต่ละเวอร์ชันออกจาก `ApplicationController` เพื่อให้ตอนมี v2 ในอนาคต
**ปรับ behavior ของ v1 กับ v2 แยกจากกันได้อย่างอิสระ** โดยไม่กระทบกัน:

```ruby
# app/controllers/api/v1/base_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class BaseController < ActionController::API
      include Pagy::Backend
      include ActionController::MimeResponds

      JSONAPI_CONTENT_TYPE = "application/vnd.api+json"

      rescue_from ActiveRecord::RecordNotFound, with: :render_not_found

      private

      def render_not_found(exception)
        render json: {
          errors: [{ status: "404", title: "Record Not Found", detail: exception.message }]
        }, content_type: JSONAPI_CONTENT_TYPE, status: :not_found
      end
    end
  end
end
```

จุดที่ควรสังเกต:

1. **inherit จาก `ActionController::API` ตรงๆ ไม่ใช่ `ApplicationController`** — เพราะ
   `ApplicationController` ในแอป API-only ของ Part 056 ก็ inherit จาก `ActionController::API`
   อยู่แล้วเหมือนกัน การให้ `Api::V1::BaseController` inherit ตรงจาก `ActionController::API` เอง
   (แทนที่จะผ่าน `ApplicationController`) ทำให้ **เวอร์ชัน API แต่ละตัวเป็นอิสระจากกันเต็มที่**
   ไม่ต้องกังวลว่า `before_action` ที่เพิ่มใน `ApplicationController` ของ v2 ในอนาคตจะกระทบ v1
   โดยไม่ตั้งใจ (แนวทางนี้เหมาะกับ API ที่คาดว่าจะมีหลายเวอร์ชันคู่ขนานนาน — ถ้ามั่นใจว่ามีแค่
   เวอร์ชันเดียวตลอด ก็ inherit จาก `ApplicationController` แทนได้ ไม่ผิดหลักการอะไร)
2. **`include ActionController::MimeResponds`** — จำเป็นเพราะ `ActionController::API` **ไม่ได้
   รวม `respond_to`/`MimeResponds` ไว้ให้โดย default** (ต่างจาก `ActionController::Base` ที่มีให้
   ฟรี) รายละเอียดเต็มเรื่อง content negotiation อยู่ใน Step 570 — บรรทัดนี้คือสิ่งที่ทำให้
   `respond_to` ใช้งานได้ในทุก controller ที่ inherit จากตัวนี้
3. **`rescue_from ActiveRecord::RecordNotFound`** — วางไว้ที่ base controller เพื่อให้ทุก
   controller ของ v1 ตอบ error shape เดียวกันเสมอเมื่อหา record ไม่เจอ (`404` พร้อม body ตาม
   แนวทาง JSON:API error object ที่จะเรียนใน Step 564) แทนที่จะปล่อยให้แต่ละ controller เขียน
   error handling ซ้ำๆ กันเอง

### `Api::V1::PostsController` — โครงเริ่มต้น

```ruby
# app/controllers/api/v1/posts_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class PostsController < BaseController
      def index
        posts = Post.published.includes(:author, :category).order(id: :asc)
        render json: posts.as_json(only: %i[id title body views published])
      end

      def show
        post = Post.includes(:author, :category).find(params[:id])
        render json: post.as_json(only: %i[id title body views published])
      end
    end
  end
end
```

ทดสอบว่า routing + controller ทำงานถูกต้อง:

```bash
curl -s "http://localhost:3000/api/v1/posts" | head -c 200
# [{"id":2,"title":"บทความ #2","body":"เนื้อหาตัวอย่างของบทความลำดับที่ 2","views":37,...
```

โครงนี้ยังไม่ใช่ shape สุดท้าย — response ยังเป็น array/object ธรรมดา (มักเรียกว่า "flat JSON" หรือ
"naive JSON") ไม่มีมาตรฐานอะไรกำกับไว้เลย Step 564 จะพาไปดูว่าทำไมการมีมาตรฐานกำกับ shape ของ
response ถึงสำคัญ แล้ว Step 565 จะ implement shape นั้นแทนที่โครงนี้

---

## Step 564: JSON:API Specification คืออะไร — `data`/`attributes`/`relationships`/`included`/`meta`/`links`

### ปัญหาของ "flat JSON" ที่ไม่มีมาตรฐาน

โครง `Api::V1::PostsController#show` ใน Step 563 ตอบกลับแบบนี้:

```json
{ "id": 2, "title": "บทความ #2", "body": "...", "views": 37, "published": true }
```

ดูเผินๆ ไม่มีปัญหาอะไร แต่พอ API โตขึ้นจะเจอคำถามซ้ำๆ ที่ **แต่ละทีมตอบไม่เหมือนกัน** เพราะไม่มี
มาตรฐานกำกับ:

- ถ้า `show` ต้องรวมข้อมูล `author`/`category` ไปด้วย จะใส่ยังไง? บาง API ทำ
  `{ "author_name": "..." }` (flatten เข้าไปเลย), บาง API ทำ `{ "author": { "id": 1,
  "name": "..." } }` (nested object), บาง API ทำ `{ "author_id": 1 }` (แค่ id ให้ client ไป fetch
  เอง) — **แต่ละแบบไม่เหมือนกันเลย แม้จะเป็นปัญหาเดียวกัน**
- error response หน้าตาเป็นอย่างไร? `{ "error": "..." }` หรือ `{ "errors": ["..."] }` หรือ
  `{ "message": "...", "code": 404 }`?
- pagination metadata ใส่ตรงไหน? ใน HTTP header, ใน field `meta`, หรือปนกับข้อมูลจริงเลย?

เมื่อแต่ละทีมในองค์กร (หรือแต่ละบริษัทที่ทำ public API) ตอบคำถามเหล่านี้ต่างกันไปเรื่อยๆ **client
tooling ที่ต้องการทำงานข้าม API หลายตัว** (API client library อัตโนมัติ, SDK generator, Postman
collection ที่ import จาก schema, GraphQL-like client cache ฝั่ง frontend) **ทำงานร่วมกันไม่ได้
เลย** เพราะแต่ละ API มี shape ที่ไม่มีแพทเทิร์นร่วมกันให้เขียนโค้ดทั่วไปมารองรับได้

### JSON:API เข้ามาแก้ปัญหานี้ — มาตรฐานเปิดสำหรับ shape ของ JSON response

**[JSON:API](https://jsonapi.org)** คือ specification (ข้อกำหนดแบบเปิด ไม่ใช่ library หรือ
framework เฉพาะภาษาใด) ที่นิยาม **โครงสร้างมาตรฐานของ JSON response** สำหรับ API แบบ resource-
based โดยกำหนด media type เฉพาะคือ `application/vnd.api+json`

โครงสร้างหลักของ JSON:API document:

```json
{
  "data": {
    "id": "2",
    "type": "posts",
    "attributes": {
      "title": "บทความ #2",
      "body": "เนื้อหาตัวอย่างของบทความลำดับที่ 2",
      "views": 37,
      "published": true
    },
    "relationships": {
      "author": { "data": { "id": "2", "type": "authors" } },
      "category": { "data": { "id": "2", "type": "categories" } }
    }
  },
  "included": [
    { "id": "2", "type": "authors", "attributes": { "name": "Somsak" } },
    { "id": "2", "type": "categories", "attributes": { "name": "Rails" } }
  ],
  "meta": { "...": "..." },
  "links": { "...": "..." }
}
```

อธิบายแต่ละส่วน:

| ส่วน | ความหมาย |
|---|---|
| `data` | ตัวข้อมูลหลัก (resource object เดี่ยว หรือ array ของ resource object) |
| `data.id` | id ของ resource **เป็น string เสมอ** (แม้ database เก็บเป็น integer) เพราะ spec ต้องการ
  ให้ id เปรียบเทียบ/ใช้เป็น key ได้แน่นอนไม่ว่า backend จะเก็บเป็นชนิดข้อมูลอะไรก็ตาม |
| `data.type` | ชื่อประเภทของ resource (พหูพจน์ ตัวเล็ก ตามธรรมเนียม เช่น `"posts"`) ใช้คู่กับ
  `id` เป็น **compound key ที่ระบุ resource ได้ไม่ซ้ำกันทั้งระบบ** |
| `data.attributes` | field ข้อมูลจริงของ resource (ไม่รวม `id` เพราะย้ายออกไปเป็น field แยก
  ข้างบนแล้ว) |
| `data.relationships` | ระบุว่า resource นี้เชื่อมกับ resource อื่นอย่างไร โดยอ้างผ่าน
  `{ id, type }` เท่านั้น (ไม่ embed ข้อมูลเต็มไว้ตรงนี้ — คล้ายกับ foreign key) |
| `included` | ข้อมูลเต็มของ resource ที่ถูกอ้างถึงใน `relationships` (ถ้า client ขอมาผ่าน query
  param `?include=author,category`) วางแยกจาก `data` เพื่อไม่ให้ข้อมูลซ้ำซ้อนเวลามีหลาย post ที่
  ใช้ author คนเดียวกัน |
| `meta` | ข้อมูล metadata ที่ไม่ใช่ตัว resource เอง เช่น pagination info (Step 567) |
| `links` | URL ที่เกี่ยวข้อง เช่น URL ของหน้าถัดไป (pagination), URL ของ resource นี้เอง (`self`) |

### ทำไมแยก `relationships` ออกจาก `included` — จุดที่ชาญฉลาดที่สุดของ spec นี้

สมมติ endpoint คืน 20 บทความ ที่เขียนโดยผู้เขียนแค่ 3 คน (ตรงกับ seed ของ Part นี้พอดี) —
ถ้า embed ข้อมูล author เต็มๆ ไว้ใน `attributes` ของทุกบทความตรงๆ (แบบที่ API จำนวนมากทำ) ข้อมูล
ของผู้เขียนคนเดียวกันจะถูกส่งซ้ำ **20 ครั้ง** (ครั้งละเท่ากันทุกตัวอักษร) แต่ด้วยโครงสร้าง JSON:API
— `data` แต่ละ post มีแค่ `relationships.author.data` ที่เป็น `{ id, type }` (เบามาก) ส่วนข้อมูล
เต็มของผู้เขียนทั้ง 3 คนอยู่ใน `included` **แค่ตัวละ 1 ชุดเท่านั้น ไม่ว่าจะมีกี่บทความอ้างถึงก็ตาม**
— ประหยัด bandwidth ได้มากเมื่อ response มีหลาย record ที่แชร์ relationship เดียวกัน และ client
ฝั่ง frontend ที่มี local cache (เช่น Redux/normalized store) สามารถ normalize ข้อมูลตาม `{ id,
type }` ได้ทันทีโดยไม่ต้องเขียน logic เอง เพราะ shape นี้ **ถูกออกแบบมาให้ normalize ได้อยู่แล้ว
โดยธรรมชาติ**

### ข้อดีของการยึด JSON:API ในทางปฏิบัติ

1. **มี library ฝั่ง client สำเร็จรูป** ในแทบทุกภาษา/framework (JS: `jsonapi-client`, iOS:
   `JSONAPI` libraries, Android เช่นกัน) ที่ deserialize response เป็น object พร้อมใช้ทันทีโดยไม่
   ต้องเขียน parser เอง — เพราะ shape มาตรฐานเดียวกันทุก API ที่ยึด spec นี้
2. **Error format มาตรฐาน** — spec นิยาม shape ของ `errors` array ไว้ด้วย (`status`, `title`,
   `detail`, `source`) ทำให้ error handling ฝั่ง client เขียนแบบ generic ได้ ไม่ต้องเขียนแยกทีละ
   endpoint
3. **Pagination/filtering/sparse fieldset มี convention มาตรฐาน** ใน query param (`page[...]`,
   `filter[...]`, `fields[...]`) แม้ Part นี้จะไม่ implement ทุกส่วนของ spec แบบเข้มงวด 100% ก็ตาม
   (ดูหมายเหตุท้าย Step)

### ข้อควรรู้ก่อนตัดสินใจใช้จริง

JSON:API **ไม่ใช่ตัวเลือกเดียว** และไม่ใช่ทุก API ควรใช้ — ทางเลือกอื่นที่พบบ่อยพอกัน: **GraphQL**
(จะเรียนเต็มรูปแบบใน Part 058–059 ที่แก้ปัญหาคนละมุม — client เลือก field เองได้ทั้งหมดแทนที่จะ
รับ shape ตายตัวจาก server), **plain/flat JSON แบบกำหนดเองแต่มี convention ภายในทีมชัดเจน** (ใช้
ได้ดีถ้า API มีแค่ client ภายในทีมเดียวกัน ไม่มีความจำเป็นต้องรองรับ generic client tooling),
หรือ **HAL/Siren** (มาตรฐาน hypermedia คู่แข่งที่เบากว่า JSON:API แต่ได้รับความนิยมน้อยกว่ามาก)

**กฎการตัดสินใจในทางปฏิบัติ:** ถ้า API เปิดให้ third-party ใช้เป็นสาธารณะ หรือมี client หลายทีม/
หลายภาษาที่ไม่ได้อยู่ใต้การควบคุมเดียวกัน — JSON:API คุ้มค่าเพราะได้ tooling สำเร็จรูปมาฟรี ถ้า
API มีแค่ frontend ทีมเดียวกันใช้ภายในบริษัท และทีม frontend ยินดีเขียน parser เองตาม shape ที่
กำหนด — flat JSON ธรรมดาที่มี convention ชัดเจนพอ ไม่จำเป็นต้องแบก overhead ของ JSON:API เลย

---

## Step 565: Implement JSON:API Response ด้วยมือ แล้ว Refactor ด้วย `jsonapi-serializer`

### แบบที่ 1: เขียนด้วยมือ (ทำความเข้าใจ shape ให้ถ่องแท้ก่อน)

ก่อนจะพึ่ง gem ใดๆ ควรเห็นว่า shape ของ Step 564 สร้างขึ้นเองด้วยโค้ด Ruby ธรรมดาได้ไม่ยากเลย —
เขียน `serialize_post` เป็น private method ใน controller ตรงๆ:

```ruby
# ทดลองใน bin/rails runner (ยังไม่ใช่ controller จริง — สาธิตให้เห็นโครงสร้างก่อน)
post = Post.find(2)

hand_rolled = {
  data: {
    id: post.id.to_s,
    type: "posts",
    attributes: {
      title: post.title,
      body: post.body,
      views: post.views,
      published: post.published
    },
    relationships: {
      author: { data: { id: post.author_id.to_s, type: "authors" } },
      category: { data: { id: post.category_id.to_s, type: "categories" } }
    }
  }
}

puts JSON.pretty_generate(hand_rolled)
```

```json
{
  "data": {
    "id": "2",
    "type": "posts",
    "attributes": {
      "title": "บทความ #2",
      "body": "เนื้อหาตัวอย่างของบทความลำดับที่ 2",
      "views": 37,
      "published": true
    },
    "relationships": {
      "author": { "data": { "id": "2", "type": "authors" } },
      "category": { "data": { "id": "2", "type": "categories" } }
    }
  }
}
```

**ทำงานได้ถูกต้องตาม spec 100%** — สำหรับ endpoint ง่ายๆ ที่มีไม่กี่ resource type การเขียนด้วยมือ
แบบนี้ไม่ใช่เรื่องผิดอะไรเลย และมีข้อดีคือ **ไม่มี dependency เพิ่ม ควบคุมได้เต็มที่ทุกบรรทัด**
ปัญหาจะเริ่มปรากฏเมื่อ API โตขึ้น: ต้อง handle `included` (เขียน logic รวม/ลบข้อมูลซ้ำเอง), ต้อง
handle `meta`/`links` ของ pagination (Step 567), ต้องทำแบบนี้ซ้ำกับทุก model ในระบบ — โค้ดซ้ำซ้อน
มากขึ้นเรื่อยๆ แบบเดียวกับปัญหา `if` เกลื่อนกลาดที่ Part 038 Step 378 เจอตอนเขียน filter logic เอง
ทุก field

### แบบที่ 2: ใช้ gem `jsonapi-serializer`

> **ทำไมไม่ใช้ Blueprinter จาก Part 056 ต่อเลย:** Part 056 เลือก Blueprinter เป็นเครื่องมือหลัก
> สำหรับสร้าง JSON output ทั่วไป (`fields`, `view`) ซึ่งยังเป็นตัวเลือกที่ดีมากสำหรับ flat JSON —
> แต่ Blueprinter **ไม่ได้ออกแบบมาให้ผลิต shape ตาม JSON:API spec โดยเฉพาะ** (ไม่มีแนวคิด
> `relationships`/`included`/compound `{id, type}` ในตัว) การจะเค้น shape ของ Step 564 ออกมาจาก
> Blueprinter ต้องเขียน field แบบ nested เองทั้งหมด ไม่ต่างจากเขียนด้วยมือเลย — Part นี้จึงแนะนำ
> `jsonapi-serializer` เป็นเครื่องมือ **เฉพาะทาง** สำหรับตอนที่ต้องการ shape ตาม JSON:API จริงๆ
> เท่านั้น ไม่ได้แปลว่า Blueprinter ใช้ไม่ได้ — endpoint ที่ไม่จำเป็นต้องยึด JSON:API spec (ส่วนใหญ่
> ของระบบ) ยังคงใช้ Blueprinter ตามที่ Part 056 วางแนวทางไว้ได้ตามปกติ

`jsonapi-serializer` แก้ปัญหานี้ด้วยการให้ประกาศ **serializer class แยกต่างหากสำหรับแต่ละ model**
คล้ายกับแนวคิด Presenter/Decorator ที่จะเรียนเต็มรูปแบบใน Part 083:

```ruby
# app/serializers/author_serializer.rb
# frozen_string_literal: true

class AuthorSerializer
  include JSONAPI::Serializer

  set_type :authors
  attributes :name
end
```

```ruby
# app/serializers/category_serializer.rb
# frozen_string_literal: true

class CategorySerializer
  include JSONAPI::Serializer

  set_type :categories
  attributes :name
end
```

```ruby
# app/serializers/post_serializer.rb
# frozen_string_literal: true

class PostSerializer
  include JSONAPI::Serializer

  set_type :posts
  attributes :title, :body, :views, :published

  belongs_to :author
  belongs_to :category
end
```

ใช้งานใน controller:

```ruby
def show
  post = Post.includes(:author, :category).find(params[:id])
  render json: PostSerializer.new(post, include: [:author, :category]).serializable_hash.to_json,
         content_type: "application/vnd.api+json"
end
```

ทดสอบด้วย curl จริง:

```bash
curl -s -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts/2?include=author,category" | jq .
```

```json
{
  "data": {
    "id": "2",
    "type": "posts",
    "attributes": {
      "title": "บทความ #2",
      "body": "เนื้อหาตัวอย่างของบทความลำดับที่ 2",
      "views": 37,
      "published": true
    },
    "relationships": {
      "author": { "data": { "id": "2", "type": "authors" } },
      "category": { "data": { "id": "2", "type": "categories" } }
    }
  },
  "included": [
    { "id": "2", "type": "authors", "attributes": { "name": "Somsak" } },
    { "id": "2", "type": "categories", "attributes": { "name": "Rails" } }
  ]
}
```

**`data` ตรงกับที่เขียนด้วยมือทุกตัวอักษร** — พิสูจน์ว่า gem แค่ทำสิ่งเดียวกับที่เขียนเองได้ แต่ทำ
ให้อัตโนมัติ พร้อมเพิ่ม `included` ให้ฟรีเมื่อส่ง `include:` option (ไม่ต้องเขียน logic dedup เอง)
สังเกตว่า **`include: [:author, :category]` ในโค้ด Ruby มาจาก `params[:include]` ที่ client ส่งมา
ผ่าน query string `?include=author,category`** — ต้องแปลงและ **allowlist ค่าที่รับมาก่อนเสมอ**
(หลักการเดียวกับ Part 038 Step 376 เป๊ะ — ไม่ปล่อยให้ string จาก `params` กำหนดโครงสร้าง response
ได้อย่างอิสระโดยไม่ตรวจสอบ):

```ruby
def include_relationships
  params[:include].to_s.split(",").map(&:strip) & %w[author category]
end
```

`& %w[author category]` (set intersection) รับประกันว่าค่าที่ได้ **เป็นสับเซตของ allowlist เสมอ**
ไม่ว่า client จะส่งอะไรมาก็ตาม — ถ้าส่ง `?include=author,secret_internal_field` ผลลัพธ์จะเหลือแค่
`["author"]` โดยอัตโนมัติ ไม่ raise error และไม่มีทางหลุดรอด field ที่ไม่ได้ตั้งใจให้ expose

ทดสอบ `index` action ที่คืนหลาย record พร้อม `included` ที่ dedup ให้อัตโนมัติ:

```bash
curl -s -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts?limit=5&include=author,category" | jq '.included | length'
# 6   <- 3 authors + 3 categories เท่านั้น ไม่ว่า posts จะมีกี่รายการที่อ้างถึงคนเดียวกันซ้ำก็ตาม
```

### กฎการเลือกใช้ในทางปฏิบัติ

**เขียนด้วยมือ:** เหมาะกับ API เล็กๆ ที่มี resource type ไม่กี่ตัว หรือทีมที่อยากควบคุม logic ทุก
บรรทัดเอง (ไม่มี "เวทมนตร์" จาก gem ที่ debug ยากเวลาเจอ edge case แปลกๆ) **ใช้ gem:** เหมาะกับ API
ที่มี resource type เยอะ ต้องการ `included`/sparse fieldset ที่ทำงานถูกต้องตาม spec เป๊ะโดยไม่ต้อง
เขียน dedup logic เอง และต้องการความสม่ำเสมอของ shape ทั่วทั้งระบบโดยอัตโนมัติ — Part นี้ใช้ gem
ต่อไปตลอด Step ที่เหลือ เพราะ Step 567 จะต้องผสาน pagination metadata เข้ากับ serializer ด้วย ซึ่ง
gem รองรับ option `meta:`/`links:` ให้พร้อมใช้อยู่แล้ว

---

## Step 566: Offset-based vs Cursor-based Pagination — ปัญหา Consistency ที่เกิดขึ้นจริงเมื่อมี Concurrent Write

Part 038 สอน pagination ด้วย `LIMIT`/`OFFSET` มาตลอด (`pagy(collection)`, `.page(n).per(20)`) ซึ่ง
ใช้งานได้ดีมากสำหรับหน้าเว็บ HTML ที่มีปุ่มกดเลขหน้า 1, 2, 3 — **แต่สำหรับ API ที่ clientดึงข้อมูล
ต่อเนื่องเป็น stream (infinite scroll, sync ข้อมูลเป็นระยะ, mobile app โหลดเพิ่มเรื่อยๆ) มีปัญหา
ที่ไม่ปรากฏชัดตอนใช้กับหน้าเว็บทั่วไป: **ความไม่สอดคล้องกัน (consistency) เมื่อข้อมูลมีการเปลี่ยน
แปลงระหว่างที่ client กำลังเปิดหน้าถัดไปทีละหน้า**

### สาธิตปัญหาจริงด้วยข้อมูลจริง: รายการซ้ำเมื่อมีการเพิ่มข้อมูลใหม่

ข้อมูล published posts ปัจจุบันเรียงจากใหม่ไปเก่า (`order(id: :desc)`):

```
[25, 24, 23, 22, 20, 19, 18, 17, 15, 14, 13, 12, 10, 9, 8, 7, 5, 4, 3, 2]
```

Client ขอหน้าแรก (`offset=0, limit=5`):

```ruby
Post.published.order(id: :desc).offset(0).limit(5).pluck(:id)
# => [25, 24, 23, 22, 20]
```

ระหว่างที่ client กำลังจะขอหน้า 2 (เหตุการณ์ที่เกิดขึ้นได้เสมอในระบบจริงที่มีคนใช้พร้อมกันหลายคน)
**มีบทความใหม่ id=26 ถูกเผยแพร่:**

```ruby
Post.create!(title: "บทความด่วน", body: "...", author: Author.first,
             category: Category.first, published: true, views: 0)
```

Client ขอหน้า 2 ต่อ (`offset=5, limit=5`) — เพราะ `order(id: :desc)` มีบทความใหม่แทรกเข้ามาที่
**ตำแหน่งแรกสุด** ทำให้ทุกอย่างที่เคยอยู่หลังจากนั้นเลื่อนตำแหน่งลงไปหนึ่งตำแหน่งทั้งหมด:

```ruby
Post.published.order(id: :desc).offset(5).limit(5).pluck(:id)
# => [20, 19, 18, 17, 15]
```

```
Page 1: [25, 24, 23, 22, 20]
Page 2: [20, 19, 18, 17, 15]
รายการที่ซ้ำกันระหว่างสองหน้า: [20]
```

**`id=20` ปรากฏอยู่ในทั้งสองหน้า** — client ที่ทำ infinite scroll จะเห็นบทความเดิมโผล่ซ้ำขึ้นมา
อีกครั้งตอนโหลดหน้าถัดไป ทั้งที่ไม่มีอะไรผิดพลาดฝั่ง client เลยแม้แต่น้อย เกิดจากกลไก
`OFFSET`/`LIMIT` **นับตำแหน่งจากศูนย์ใหม่ทุกครั้งที่ query** โดยไม่รู้ว่า client "เคยเห็นอะไรไปแล้ว
บ้าง" — ถ้ามีข้อมูลแทรกเข้ามาก่อนตำแหน่งที่ query อยู่ ตัวเลข offset เดิมจะชี้ไปที่ตำแหน่งอื่นทันที

### ปัญหากลับด้าน: รายการหายไปเมื่อมีการลบข้อมูล

สลับสถานการณ์: แทนที่จะเพิ่มข้อมูลใหม่ คราวนี้ระหว่างสอง request **แอดมินลบบทความ id=22** ซึ่ง
อยู่ในหน้า 1 ที่แสดงไปแล้ว (`before = [25,24,23,22,20,19,18,...]`, `page1 = [25,24,23,22,20]`) —
ผลลัพธ์ของหน้า 2 (`offset(5).limit(5)`) หลังลบคือ `[18, 17, 15, 14, 13]` สิ่งที่น่าตกใจคือ
**`id=19` ซึ่งไม่เคยถูกลบหรือแก้ไขอะไรเลย** หายไปจากทั้งสองหน้า: ก่อนลบมันอยู่ที่ตำแหน่งดัชนี 5
(พอดีจะเป็นรายการแรกของหน้า 2) แต่พอ `id=22` ที่อยู่ก่อนหน้าถูกลบ ทุกอย่างหลังจากนั้นเลื่อนตำแหน่ง
ขึ้นมาหนึ่งตำแหน่งทั้งหมด — `id=19` เลื่อนไปอยู่ตำแหน่งสุดท้ายของหน้า 1 ที่ **แสดงไปแล้วก่อนการลบ
จะเกิดขึ้น** ผลคือมันไม่เคยถูกแสดงให้ผู้ใช้เห็นเลยสักครั้ง ทั้งที่ตัวมันเองไม่ได้ถูกแตะต้องอะไร
เลยแม้แต่น้อย — เป็นเหยื่อของช่องว่างที่เกิดจากรายการ **อื่น** ถูกลบไปเท่านั้นเอง

### ทำไม cursor-based pagination ไม่มีปัญหานี้

**Cursor-based pagination** เปลี่ยนวิธีคิดจาก "ขอรายการที่ตำแหน่งที่ N" (offset นับตำแหน่ง) เป็น
**"ขอรายการที่มาหลังจากตัวสุดท้ายที่เคยเห็น"** (cursor อ้างอิงตัว record จริง ไม่ใช่ตำแหน่ง) — ใน
ทางเทคนิคคือเปลี่ยนจาก `OFFSET n LIMIT m` เป็น `WHERE id > :last_seen_id ORDER BY id LIMIT m`

ทดสอบสถานการณ์เดียวกับ demo แรก (มีข้อมูลใหม่แทรกเข้ามาระหว่าง request) แต่ใช้ cursor แทน offset:

```ruby
page1 = Post.published.order(id: :asc).limit(5).pluck(:id)
# => [2, 3, 4, 5, 7]

cursor = page1.last # = 7 (จำ id ตัวสุดท้ายที่เห็นไว้ แทนที่จะจำ "ตำแหน่ง")
```

มีบทความใหม่ id=26 ถูกเผยแพร่ระหว่าง request (เหตุการณ์เดียวกับ demo แรกเป๊ะ):

```ruby
Post.create!(title: "บทความด่วน", body: "...", author: Author.first,
             category: Category.first, published: true, views: 0)
```

```ruby
page2 = Post.published.where("id > ?", cursor).order(id: :asc).limit(5).pluck(:id)
# => [8, 9, 10, 12, 13]
```

```
Page 1: [2, 3, 4, 5, 7]
Page 2: [8, 9, 10, 12, 13]
รายการที่ซ้ำกันระหว่างสองหน้า: []   <- ว่างเปล่าเสมอ ไม่ว่าจะมีข้อมูลใหม่แทรกเข้ามากี่ตัวก็ตาม
```

**ไม่มีการซ้ำเลย** เพราะ `WHERE id > 7` หมายถึง "เอาเฉพาะรายการที่มาหลัง id=7 จริงๆ" ไม่สนใจว่า
ระหว่างทางจะมีอะไรถูกเพิ่ม/ลบไปกี่รายการก็ตาม — บทความใหม่ id=26 ที่ถูกสร้างขึ้นจะได้ id มากกว่า 7
เสมอ (เพราะ auto-increment) เมื่อไปถึงหน้าถัดไปจริงๆ ก็จะเจอมันในลำดับที่ถูกต้องตามธรรมชาติ ไม่ทำ
ให้ของเดิมเลื่อนตำแหน่งเลยแม้แต่น้อย

### ตารางเปรียบเทียบตรงประเด็น

| ประเด็น | Offset-based (`LIMIT`/`OFFSET`) | Cursor-based (`WHERE id > :cursor`) |
|---|---|---|
| Consistency เมื่อมี insert/delete ระหว่างทาง | **ซ้ำ/ขาดหายได้** (สาธิตข้างบน) | สอดคล้องเสมอ
  (immune ต่อการเปลี่ยนแปลงนอกช่วงที่ query) |
| กระโดดไปหน้าที่ N ได้ไหม (`?page=5`) | ได้ — random access เต็มรูปแบบ | **ไม่ได้โดยง่าย** —
  เดินหน้าได้ (`next`) เท่านั้นตามธรรมชาติของกลไก |
| รู้จำนวนหน้า/รายการทั้งหมดไหม (`total_pages`) | รู้ได้ทันทีจาก `COUNT(*)` | **โดยทั่วไปไม่มีให้**
  (ต้อง `COUNT` แยกต่างหากซึ่งขัดกับเหตุผลด้าน performance ที่เลือกใช้ cursor ตั้งแต่แรก) |
| Performance ที่ offset สูงมากๆ (หน้าลึกๆ) | **ช้าลงเรื่อยๆ** ตามขนาดตาราง (ต้อง scan ข้ามแถวที่
  ไม่เอาทิ้งไปก่อนถึงตำแหน่งที่ต้องการ) | **เร็วคงที่** ไม่ว่าจะอยู่ "หน้าลึก" แค่ไหน (ใช้ index บน
  คอลัมน์ cursor ตรงๆ ผ่าน `WHERE column > value`) |
| เหมาะกับ | หน้าเว็บที่มีปุ่มเลขหน้า, ข้อมูลนิ่งไม่ค่อยเปลี่ยน, ต้องกระโดดข้ามหน้าได้ (แบบ Part
  038 ทั้งหมด) | API แบบ infinite scroll, feed ที่มีข้อมูลเปลี่ยนตลอดเวลา, sync ข้อมูลเป็นชุด,
  dataset ขนาดใหญ่มากที่ query หน้าลึกบ่อย |

> **ข้อควรระวัง:** cursor-based pagination **ไม่ใช่คำตอบที่ดีกว่าเสมอไป** — มันแลก "ความ
> สอดคล้องกันและความเร็วคงที่" มากับ "ความสามารถกระโดดไปหน้าใดก็ได้ตามใจ" และ "รู้จำนวนหน้า
> ทั้งหมด" ที่หายไป ถ้า UI ต้องการปุ่ม "ไปหน้า 5 เลย" หรือแสดง "หน้า 3 จาก 12" ตรงๆ — offset-based
> (แบบที่ Part 038 สอน) ยังคงเป็นตัวเลือกที่เหมาะสมกว่า **เลือกให้ตรงกับ UI/UX ที่ต้องการจริงๆ**
> ไม่ใช่เลือกเพราะ "cursor ดูทันสมัยกว่า"

---

## Step 567: Cursor-based Pagination ด้วย pagy (`extras/keyset`) + Pagination Metadata + Link Header แบบ GitHub

pagy ที่ใช้มาตั้งแต่ Part 038 มี **extra ชื่อ `keyset`** ที่ implement cursor-based pagination ให้
พร้อมใช้เลย (ไม่ต้องเขียน `WHERE id > :cursor` เองแบบ Step 566 ที่เป็นแค่การสาธิตหลักการ) เปิดใช้
extras ที่ต้องใช้ตลอด Part นี้ผ่าน initializer:

```ruby
# config/initializers/pagy.rb
# frozen_string_literal: true

require "pagy/extras/keyset"   # cursor-based pagination
require "pagy/extras/headers"  # สร้าง Link/meta HTTP header แบบ RFC 8288
require "pagy/extras/metadata" # helper สร้าง hash metadata (ใช้กับ offset-based ด้านล่าง)

Pagy::DEFAULT[:limit] = 10
```

### ส่วนที่ 1: Cursor-based pagination ด้วย `pagy_keyset` (ใช้เป็นค่าเริ่มต้นของ endpoint หลัก)

```ruby
# app/controllers/api/v1/posts_controller.rb (เพิ่มเข้าไปจาก Step 563/565)
module Api
  module V1
    class PostsController < BaseController
      ALLOWED_LIMITS = [5, 10, 25].freeze

      def index
        scope = Post.published.includes(:author, :category).order(id: :asc)
        @pagy, posts = pagy_keyset(scope, limit: items_per_page)
        pagy_headers_merge(@pagy)

        render json: jsonapi_collection(posts), content_type: BaseController::JSONAPI_CONTENT_TYPE
      end

      private

      def items_per_page
        limit = params[:limit].to_i
        ALLOWED_LIMITS.include?(limit) ? limit : 10
      end

      def jsonapi_collection(posts)
        PostSerializer.new(
          posts,
          meta: { limit: @pagy.limit, next_cursor: @pagy.next },
          links: { self: request.original_url, next: @pagy.next ? pagy_keyset_next_url(@pagy) : nil }
        ).serializable_hash.to_json
      end
    end
  end
end
```

จุดสำคัญ:

1. **`.order(id: :asc)` เป็นข้อบังคับ ไม่ใช่ทางเลือก** — `pagy_keyset` ต้องรู้ว่า column ไหนใช้
   เป็น "ตัวอ้างอิงลำดับที่ไม่ซ้ำกันเลย" (unique + sequential) เพื่อสร้าง cursor query ที่ถูกต้อง
   ถ้าลืมใส่ `.order(...)` จะได้ `Pagy::Keyset::InternalError: the set must be ordered` ทันที —
   นี่คือ **allowlist โดยธรรมชาติของกลไกเอง**: เลือก column ที่จะ sort ตามใจไม่ได้เหมือนที่ Part
   038 Step 376 ต้องคอย allowlist ชื่อ column เอง เพราะ cursor ผูกกับ column ที่ `.order()` ระบุ
   ไว้ตรงๆ เท่านั้น
2. **`@pagy.next` คือ cursor ของหน้าถัดไป ไม่ใช่ตัวเลขหน้า** — เป็น string ที่ถูก encode ด้วย
   Base64URL ห่อ JSON ของค่า column ที่ใช้ sort (ในที่นี้คือ `{"id": 7}`) client **ไม่ควรพยายาม
   ถอดรหัสหรือตีความค่านี้เอง** ควรถือเป็น "opaque token" ที่ส่งกลับมาตรงๆ ตอนขอหน้าถัดไป (เดี๋ยว
   Step 568 อธิบายเหตุผลว่าทำไมข้อนี้สำคัญมาก)
3. **ไม่มี `total_pages`/`total_count` ให้เลย** — ตรงตามที่ตารางเปรียบเทียบใน Step 566 บอกไว้ นี่
   ไม่ใช่ข้อจำกัดของ pagy แต่เป็นข้อจำกัดโดยธรรมชาติของ cursor-based pagination เอง (การนับ
   `COUNT(*)` ทุกครั้งขัดกับเหตุผลด้าน performance ที่เลือกใช้ cursor ตั้งแต่แรก)

ทดสอบด้วย curl จริง:

```bash
curl -s -H "Accept: application/vnd.api+json" "http://localhost:3000/api/v1/posts?limit=5" | jq .
```

```json
{
  "data": [
    { "id": "2", "type": "posts", "attributes": { "title": "บทความ #2", "...": "..." }, "...": {} },
    { "id": "3", "type": "posts", "...": {} },
    { "id": "4", "type": "posts", "...": {} },
    { "id": "5", "type": "posts", "...": {} },
    { "id": "7", "type": "posts", "...": {} }
  ],
  "meta": { "limit": 5, "next_cursor": "eyJpZCI6N30" },
  "links": {
    "self": "http://localhost:3000/api/v1/posts?limit=5",
    "next": "/api/v1/posts?limit=5&page=eyJpZCI6N30"
  }
}
```

ถอดรหัส cursor เพื่อดูว่าข้างในเป็นอะไร (แค่เพื่อความเข้าใจ — ในโค้ดจริงไม่ต้องทำแบบนี้เลย):

```bash
python3 -c "import base64; print(base64.urlsafe_b64decode('eyJpZCI6N30' + '=='))"
# b'{"id":7}'
```

เรียกหน้าถัดไปด้วย cursor ที่ได้มา (ส่งกลับผ่าน `?page=<cursor>` ตรงๆ):

```bash
curl -s -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts?limit=5&page=eyJpZCI6N30" | jq '[.data[].id]'
# ["8", "9", "10", "12", "13"]
```

ไม่มี `id=7` ปนมาซ้ำอีกเลย ตรงตามที่ Step 566 พิสูจน์ไว้ด้วยข้อมูลจริงแล้วว่า cursor-based ไม่มี
ปัญหาซ้ำ/ขาดหาย

ตรวจสอบ HTTP header ที่ `pagy_headers_merge` เติมให้อัตโนมัติ (extra `headers` ที่เปิดไว้ใน
initializer):

```bash
curl -s -D - -o /dev/null -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts?limit=5"
```

```
HTTP/1.1 200 OK
link: <http://localhost:3000/api/v1/posts?limit=5&page>; rel="first", <http://localhost:3000/api/v1/posts?limit=5&page=eyJpZCI6N30>; rel="next"
page-items: 5
content-type: application/vnd.api+json; charset=utf-8
```

`Link` header ตาม **RFC 8288** (มาตรฐานเดียวกับที่ GitHub API ใช้จริง) — client ที่รู้จัก convention
นี้ **ไม่จำเป็นต้อง parse body เพื่อหา URL หน้าถัดไปเลย** อ่านจาก header ตรงๆ ได้ทันที สังเกตว่า
สำหรับ keyset pagination จะมีแค่ `rel="first"` กับ `rel="next"` เท่านั้น (ไม่มี `rel="prev"`/
`rel="last"` เพราะเหตุผลเดียวกับที่ตารางเปรียบเทียบบอกไว้ — เดินหน้าได้ทางเดียว)

### ส่วนที่ 2: Offset-based pagination พร้อม `meta`/`Link` header แบบเต็ม (สำหรับ endpoint ที่เลือกใช้ offset จริงๆ)

ไม่ใช่ทุก endpoint ต้องใช้ cursor — บาง resource (เช่นหน้า admin ที่ต้องการปุ่ม "ไปหน้า N" ตรงๆ
หรือ dataset ที่ไม่ได้เปลี่ยนแปลงบ่อย) offset-based ยังเป็นตัวเลือกที่เหมาะสมกว่า ต่อไปนี้คือวิธี
ทำ **meta ตามรูปแบบที่พบได้ทั่วไปในหลาย JSON API** (`current_page`/`total_pages`/`total_count`)
พร้อม Link header แบบเต็มด้วย `pagy` ตัวปกติ (ไม่ใช้ extra `keyset`):

```ruby
# app/controllers/api/v1/posts_controller.rb
def index_offset
  @pagy, posts = pagy(Post.published.includes(:author, :category).order(id: :asc), limit: items_per_page)
  pagy_headers_merge(@pagy)

  render json: {
    data: posts.as_json(only: %i[id title body views published]),
    meta: {
      current_page: @pagy.page,
      total_pages: @pagy.pages,
      total_count: @pagy.count
    }
  }
end
```

```ruby
# config/routes.rb (เพิ่มเข้าไปในกลุ่ม namespace :api do namespace :v1 do ... end end)
get "posts_legacy", to: "posts#index_offset"
```

ทดสอบ:

```bash
curl -s -D - "http://localhost:3000/api/v1/posts_legacy?limit=5" | head -10
```

```
HTTP/1.1 200 OK
link: <.../posts_legacy?limit=5&page=1>; rel="first", <.../posts_legacy?limit=5&page=2>; rel="next", <.../posts_legacy?limit=5&page=4>; rel="last"
current-page: 1
page-items: 5
total-pages: 4
total-count: 20
content-type: application/json; charset=utf-8
```

```json
{
  "data": [ { "id": 2, "title": "บทความ #2", "...": "..." }, "...4 รายการถัดไป..." ],
  "meta": { "current_page": 1, "total_pages": 4, "total_count": 20 }
}
```

เรียกหน้า 2 เพื่อดู `Link` header เต็มรูปแบบ (มี `rel="prev"` เพิ่มขึ้นมาด้วยเพราะไม่ใช่หน้าแรกแล้ว):

```bash
curl -s -D - -o /dev/null "http://localhost:3000/api/v1/posts_legacy?limit=5&page=2" \
  | grep -i "^link\|current-page"
```

```
link: <.../posts_legacy?limit=5&page=1>; rel="first", <.../posts_legacy?limit=5&page=1>; rel="prev", <.../posts_legacy?limit=5&page=3>; rel="next", <.../posts_legacy?limit=5&page=4>; rel="last"
current-page: 2
```

**นี่คือ Link header สไตล์ GitHub เต็มรูปแบบ** (`first`/`prev`/`next`/`last` ครบทั้ง 4 relation)
เพราะ offset-based pagination รองรับ random access เต็มที่ ต่างจาก keyset ที่มีแค่ `first`/`next`
ในส่วนที่ 1 — เห็นความต่างเชิงโครงสร้างของทั้งสองแบบชัดเจนจาก header ตรงๆ โดยไม่ต้องอ่านโค้ด
เบื้องหลังเลยด้วยซ้ำ

---

## Step 568: เขียนสัญญา (Contract) ของ Pagination ให้ API Consumer อ่านและพึ่งพาได้

Pagination metadata ที่ Step 567 สร้างไว้ **มีประโยชน์ก็ต่อเมื่อ client รู้กติกาที่แน่นอนของมัน**
— นี่คือจุดที่ต่างจากหน้าเว็บ HTML ของ Part 038 ชัดเจน: `pagy_nav`/`pagy_info` render ปุ่มให้ผู้ใช้
กดตรงๆ ไม่ต้องมีใคร "อ่าน spec" เพราะเป็น UI ที่มองเห็นได้ แต่ **API เป็น contract ที่นักพัฒนาอีก
ทีมหนึ่ง (ซึ่งอาจไม่เคยคุยกับเราเลย) ต้องเขียนโค้ดตามให้ถูกต้อง** — สัญญาที่เขียนไม่ชัดจะนำไปสู่
บั๊กที่ debug ยากในฝั่ง client เสมอ

### สิ่งที่ documentation ของ pagination endpoint ต้องระบุให้ชัดเจนเสมอ

1. **ใช้กลยุทธ์ไหน (offset หรือ cursor)** และ **ทำไม** — ไม่ใช่แค่บอกวิธีใช้ แต่บอกเหตุผลด้วย
   เพื่อให้ client เข้าใจข้อจำกัด (เช่น "endpoint นี้ใช้ cursor-based เพราะข้อมูลเปลี่ยนบ่อยมาก
   จึงไม่มี `total_count`/`total_pages` ให้ และไม่รองรับการกระโดดข้ามหน้า")
2. **ชื่อ query param ที่แน่นอน** พร้อมชนิดข้อมูล — `?limit=<integer>&page=<opaque cursor
   string>` ระบุชัดว่า `page` **ไม่ใช่เลขหน้า** ในกรณี cursor-based (จุดที่สับสนบ่อยที่สุดเพราะ
   ชื่อ param เดียวกัน `page` ถูกใช้ทั้งสองความหมายในสอง endpoint ของ Part นี้เอง — `posts`
   ใช้แบบ cursor, `posts_legacy` ใช้แบบเลขหน้าจริง)
3. **ค่า default และค่าสูงสุดของ `limit`** — client ต้องรู้ว่าส่ง `limit=999999` แล้วจะเกิดอะไร
   ขึ้น (fallback เงียบๆ ไปที่ค่า default ตาม allowlist ของ Step 567 หรือ error?) เอกสารต้องระบุ
   พฤติกรรมนี้ตรงๆ ไม่ให้ client ต้องเดา
4. **cursor เป็น opaque token ที่ห้าม parse เอง** — ต้องเตือนตรงๆ ว่า format ภายในของ cursor
   (Base64URL ห่อ JSON ในตัวอย่างนี้) **ไม่ใช่ contract ที่รับประกันความเสถียร** ถ้า pagy เปลี่ยน
   วิธี encode cursor ใน version ถัดไป (หรือเปลี่ยนไปใช้ library อื่น) client ที่พึ่งพารูปแบบ
   ภายในของ cursor (เช่น พยายาม decode แล้วอ่านค่า `id` ไปคำนวณอะไรต่อเอง) **จะพังทันทีโดยไม่มี
   ใครแจ้งเตือนล่วงหน้า** เพราะในทางเทคนิคแล้วนี่ไม่ถือเป็น breaking change ของ API เลย (contract
   คือ "ส่ง cursor นี้กลับมาแล้วได้หน้าถัดไป" ไม่ใช่ "cursor นี้มีโครงสร้างภายในแบบนี้ตลอดไป")
5. **จะรู้ได้อย่างไรว่า "หน้าสุดท้าย" แล้ว** — สำหรับ cursor-based คือ `meta.next_cursor` (หรือ
   `links.next`) เป็น `null`/ไม่มี field นี้เลย ระบุให้ชัดว่า client ต้องเช็คเงื่อนไขนี้เพื่อหยุด
   การเรียกหน้าถัดไป ไม่ใช่เช็คจาก `data` เป็น array ว่างเปล่า (เพราะบาง endpoint อาจคืน `data: []`
   ในหน้ากลางๆ ได้ถ้ามี filter ที่ทำให้บางหน้าไม่มีผลลัพธ์พอดี ในขณะที่ยังมีหน้าถัดไปที่มีข้อมูล)

### ตัวอย่างเอกสารสั้นๆ ที่ระบุ contract ได้ครบตามหลักการข้างบน

```markdown
## GET /api/v1/posts

Pagination: cursor-based (เพราะข้อมูลอัปเดตบ่อย ไม่รองรับการกระโดดข้ามหน้า)

Query params:
- `limit` (integer, optional): จำนวนรายการต่อหน้า อนุญาตเฉพาะ 5, 10, 25 — ค่าอื่นจะถูกปรับเป็น
  10 อัตโนมัติโดยไม่ error
- `page` (string, optional): cursor token จาก `meta.next_cursor` ของ response ก่อนหน้า
  **ห้าม parse หรือสร้าง cursor เองเด็ดขาด** ให้ส่งค่าที่ได้รับกลับมาตรงๆ เท่านั้น รูปแบบภายในของ
  cursor อาจเปลี่ยนแปลงได้ทุกเมื่อโดยไม่ถือเป็น breaking change

Response meta:
- `meta.limit`: จำนวนรายการต่อหน้าที่ใช้จริง (หลังผ่าน allowlist)
- `meta.next_cursor`: cursor สำหรับขอหน้าถัดไป — เป็น `null` เมื่อถึงหน้าสุดท้ายแล้ว
  (**ให้เช็ค field นี้เพื่อหยุดวนลูปโหลดหน้าถัดไป ไม่ใช่เช็คว่า `data` ว่างเปล่า**)

ไม่มี `total_count`/`total_pages` เนื่องจากใช้ cursor-based pagination
```

การเขียนสัญญาแบบนี้ **ป้องกันปัญหาที่เกิดขึ้นบ่อยที่สุดในทีมจริง**: นักพัฒนา frontend/mobile เห็น
field `page` เป็น string แปลกๆ แล้วเข้าใจผิดว่าเป็น bug พยายามแปลงเป็นตัวเลขเอง หรือคิดไปเองว่า
เพิ่มค่า `page` ทีละ 1 ได้เหมือน offset-based — สัญญาที่ชัดเจนตัดปัญหานี้ตั้งแต่ต้น

---

## Step 569: HATEOAS คืออะไร — และควรใช้จริงในระบบมากแค่ไหน

### HATEOAS คืออะไรจริงๆ

**HATEOAS** (Hypermedia As The Engine Of Application State) เป็นหนึ่งในหลักการดั้งเดิมของสถาปัตย
กรรม REST (จากวิทยานิพนธ์ปี 2000 ของ Roy Fielding ผู้บัญญัติศัพท์ REST) แนวคิดหลักคือ: **response
ของ API ควรมี "ลิงก์" บอกว่าจาก state ปัจจุบันนี้ ทำอะไรต่อได้บ้าง** โดยที่ client **ไม่จำเป็นต้อง
รู้ URL structure ล่วงหน้าเลย** — เดินตามลิงก์ที่ server ให้มาเรื่อยๆ เหมือนคนเรียกดูเว็บไซต์ด้วย
การคลิกลิงก์ ไม่ใช่พิมพ์ URL เอง

ตัวอย่างรูปธรรม — สมมติ response ของบทความหนึ่งชิ้นมี `links` บอกว่า "ทำอะไรกับบทความนี้ได้บ้าง"
ตาม state ปัจจุบันของมัน:

```json
{
  "data": {
    "id": "2", "type": "posts",
    "attributes": { "title": "บทความ #2", "published": false }
  },
  "links": {
    "self": "/api/v1/posts/2",
    "publish": "/api/v1/posts/2/publish",
    "author": "/api/v1/authors/2"
  }
}
```

ถ้าบทความนี้ **เผยแพร่ไปแล้ว** (`published: true`) server อาจไม่ส่งลิงก์ `publish` กลับมาเลย (เพราะ
ทำไปแล้ว ทำซ้ำไม่ได้อีก) แทนที่ด้วยลิงก์ `unpublish` แทน — **client ไม่ต้องเขียน logic เองเลยว่า
"ปุ่มไหนควรกดได้ตอนนี้"** เพียงแค่เช็คว่า response มีลิงก์ชื่อนั้นอยู่ไหม ถ้ามีก็แสดงปุ่ม ถ้าไม่มี
ก็ซ่อนปุ่ม — state machine ทั้งหมดถูกกำหนดจาก **server ฝั่งเดียว** ผ่านการมี/ไม่มีลิงก์ ไม่ใช่ client
ต้อง hardcode เงื่อนไข `if post.published? then show unpublish button` เอง

### ทำไมแนวคิดนี้ฟังดูดีมากในทฤษฎี

- **ลด coupling ระหว่าง client กับ URL structure** — server เปลี่ยน URL ของ endpoint ไหนก็ได้โดย
  ไม่ทำให้ client พัง (เพราะ client เดินตามลิงก์ที่ได้รับมา ไม่ได้ hardcode URL เอง)
- **Business logic เรื่อง "ทำอะไรได้/ไม่ได้ตอนนี้" อยู่ที่เดียว** (server) ไม่ต้อง sync logic
  เดียวกันซ้ำสองที่ (แบบที่มักเกิดขึ้นจริง: server เช็ค authorization ว่า publish ได้ไหม ในขณะที่
  frontend ก็ต้องเขียนเงื่อนไขเดียวกันซ้ำเพื่อ "โชว์ปุ่ม")

### ทำไมในทางปฏิบัติ API จำนวนมากเลือก "ข้าม" หลักการนี้

แม้ทฤษฎีจะฟังดูดีมาก แต่ **API สาธารณะที่มีชื่อเสียงส่วนใหญ่ในโลกจริง (Stripe, GitHub บางส่วน,
Twitter/X, และ API ภายในองค์กรแทบทั้งหมด) ไม่ได้ implement HATEOAS อย่างเคร่งครัด** ด้วยเหตุผล
ที่ตรงไปตรงมามาก:

1. **ต้นทุนการพัฒนาสูงกว่าที่เห็น** — ทุก endpoint ต้องคำนวณว่า "ตอนนี้ทำอะไรได้บ้าง" ตาม state/
   permission ของ resource นั้นๆ แล้วสร้างลิงก์ให้ถูกต้อง ซึ่งซับซ้อนกว่าการแค่ return ข้อมูลตรงๆ
   มาก โดยเฉพาะเมื่อ permission ขึ้นกับ role ของผู้ใช้ที่เรียก (ผู้เขียนเจ้าของบทความเห็นลิงก์
   `edit`/`delete` แต่ผู้ใช้ทั่วไปไม่เห็น — logic นี้ต้องคำนวณแยกทุก request)
2. **Client tooling ส่วนใหญ่ไม่ได้ถูกออกแบบมาให้ "เดินตามลิงก์" จริงๆ** — SDK ที่ generate จาก
   OpenAPI/Swagger schema (ซึ่งเป็นวิธีที่ทีมส่วนใหญ่สร้าง client library ในทางปฏิบัติ) มักสมมติว่า
   URL pattern คงที่ตายตัว ไม่ได้เขียนมาให้ "ค้นหาลิงก์ชื่อ publish ใน response แล้วเรียกมัน" แบบที่
   HATEOAS ต้องการจริงๆ
3. **Frontend ยุคปัจจุบันมักรู้ URL structure ล่วงหน้าอยู่แล้วโดยธรรมชาติ** — เพราะ frontend กับ
   backend มักพัฒนาโดยทีมเดียวกันหรือทีมที่คุยกันตลอด (ไม่ใช่ third-party ที่ไม่รู้จักกันเลยแบบที่
   HATEOAS ถูกออกแบบมาแก้ปัญหา) การผูก URL ไว้ตรงๆ ในโค้ด frontend จึงไม่ใช่ปัญหาใหญ่ในทางปฏิบัติ
4. **Mobile app ที่ไม่ได้อัปเดตบ่อย (ปัญหาเดียวกับ Step 561) ก็ไม่ได้ประโยชน์เต็มที่จาก HATEOAS
   เท่าที่ควร** — เพราะ business logic ของการ "ตัดสินใจว่าจะแสดงปุ่มไหน" อาจเปลี่ยนแปลงตาม feature
   ใหม่ที่ client เก่าไม่รู้จักคำสั่งใหม่นั้นอยู่ดี ต่อให้ server บอกลิงก์มาให้ก็ตาม

### สิ่งที่ระบบจริงส่วนใหญ่ทำแทน — "HATEOAS แบบเบาๆ"

สิ่งที่ API ระดับ production จำนวนมากทำจริงคือ **ใช้แค่บางส่วนของแนวคิด HATEOAS แบบเจาะจง** ไม่ใช่
ทั้งหมดตามตำรา — ตัวอย่างที่ Part นี้ implement ไปแล้วโดยไม่รู้ตัวคือ **`links.next`/`links.self`
ของ pagination ใน Step 567** นั่นเอง ซึ่งเป็นการใช้ hypermedia link **เฉพาะจุดที่คุ้มค่าจริง**
(บอก URL หน้าถัดไปโดยไม่ต้องให้ client คำนวณ `page`/cursor เอง) โดยไม่ต้อง implement ทั้งระบบ
state machine แบบเต็มรูปแบบ

> **คำแนะนำที่ตรงไปตรงมา:** อย่าเริ่มต้นออกแบบ API ด้วยความคิดว่า "ต้อง implement HATEOAS ให้ครบ
> ตามตำรา REST" เพราะต้นทุนสูงกว่าประโยชน์ที่ได้จริงในระบบส่วนใหญ่ — **ใช้ hypermedia link เฉพาะ
> จุดที่ client ได้ประโยชน์จริงและชัดเจน** (URL ของหน้าถัดไปใน pagination คือตัวอย่างที่คุ้มค่า
> ที่สุด, ลิงก์ไปยัง related resource ที่ client มักต้องเรียกต่อทันทีก็คุ้มค่าเช่นกัน) ส่วนการ
> ออกแบบ state machine เต็มรูปแบบด้วยลิงก์แบบ HATEOAS ทั้งระบบ เก็บไว้สำหรับกรณีที่ระบบมี third-
> party client จำนวนมากที่ไม่รู้จักกันเลยจริงๆ และทีมพร้อมรับต้นทุนด้าน maintenance ที่สูงขึ้นมาก
> เพื่อแลกกับความยืดหยุ่นนั้น

---

## Step 570: Content Negotiation — `Accept`/`Content-Type`, `respond_to` รองรับทั้ง JSON ธรรมดาและ JSON:API

### `Accept` vs `Content-Type` — สองเรื่องคนละทิศทาง

สอง header นี้สับสนกันบ่อยมาก ทั้งที่ความหมายตรงข้ามกันเลย:

| Header | ทิศทาง | ความหมาย |
|---|---|---|
| `Content-Type` | Request/Response body ที่ **แนบมาด้วยจริง** | "ข้อมูลที่ส่งมานี้เป็นรูปแบบ
  อะไร" (client บอก server ตอนส่ง `POST`/`PATCH`, หรือ server บอก client ตอนตอบกลับ) |
| `Accept` | Request เท่านั้น | "อยากได้ response กลับมาเป็นรูปแบบไหน" (client ขอ server —
  server **อาจ**ตอบตามที่ขอ หรือตอบรูปแบบอื่นถ้าไม่รองรับก็ได้ ขึ้นกับว่า server ออกแบบมาเข้มงวด
  แค่ไหน) |

กลไกที่ server ใช้ตัดสินใจว่าจะตอบ format ไหนตาม `Accept` header ที่ client ส่งมา เรียกว่า
**content negotiation** — Rails มี `respond_to` เป็นเครื่องมือสำหรับเรื่องนี้โดยเฉพาะ

### `respond_to` ใน `ActionController::API` — ต้อง include เพิ่มเอง

จุดที่พลาดบ่อยที่สุดตอนย้ายจาก `ActionController::Base` (เว็บแอปปกติ) มาเป็น
`ActionController::API`: **`respond_to`/`MimeResponds` ไม่ได้ถูกรวมไว้ให้โดย default** ลอง
เขียนแบบตรงไปตรงมาโดยไม่ include ดูก่อน:

```ruby
class Api::V1::PostsController < ActionController::API
  def show
    respond_to do |format|
      format.json { render json: Post.find(params[:id]) }
    end
  end
end
```

```
NoMethodError: undefined method 'respond_to' for #<Api::V1::PostsController>
```

ต้อง `include ActionController::MimeResponds` เอง (เหมือนที่ทำไว้ใน `Api::V1::BaseController`
ตั้งแต่ Step 563 แล้ว) — Rails official guide ระบุจุดนี้ไว้ตรงๆ ว่าเป็นโมดูลที่ต้อง "เพิ่มกลับเข้า
มาเอง" ถ้าต้องการใช้ใน API-only mode

### ลงทะเบียน media type ของ JSON:API เป็น format ใหม่

`respond_to`/`format.xxx` ทำงานกับ **format ที่ Rails รู้จักเท่านั้น** (`:json`, `:xml`, `:html`
มีมาให้ตั้งแต่ต้น) ส่วน `application/vnd.api+json` เป็น media type ที่ไม่มีมาให้ ต้องลงทะเบียนเอง
ผ่าน `Mime::Type.register`:

```ruby
# config/initializers/mime_types.rb
# frozen_string_literal: true

Mime::Type.register "application/vnd.api+json", :jsonapi
```

หลังลงทะเบียนแล้ว ใช้ `format.jsonapi` ได้เหมือน format มาตรฐานทั่วไป:

```ruby
def show
  post = Post.includes(:author, :category).find(params[:id])

  respond_to do |format|
    format.jsonapi do
      render json: PostSerializer.new(post, include: [:author, :category]).serializable_hash.to_json,
             content_type: BaseController::JSONAPI_CONTENT_TYPE
    end
    format.json { render json: post.as_json(only: %i[id title body views published]) }
    # client ส่ง Accept header แปลกที่ไม่รู้จักเลย (เช่น vendor media type จาก Step 562)
    # fallback เป็น JSON ธรรมดาแทนที่จะโยน 406/500 ใส่ผู้ใช้
    format.any { render json: post.as_json(only: %i[id title body views published]) }
  end
end
```

ทดสอบ content negotiation จริงด้วย `Accept` header ที่ต่างกัน:

```bash
# ขอแบบ JSON:API
curl -s -H "Accept: application/vnd.api+json" "http://localhost:3000/api/v1/posts/2" | jq .
```

```json
{
  "data": {
    "id": "2", "type": "posts",
    "attributes": { "title": "บทความ #2", "body": "...", "views": 37, "published": true },
    "relationships": { "author": { "...": {} }, "category": { "...": {} } }
  }
}
```

```bash
# ขอแบบ JSON ธรรมดา
curl -s -H "Accept: application/json" "http://localhost:3000/api/v1/posts/2" | jq .
```

```json
{ "id": 2, "title": "บทความ #2", "body": "...", "views": 37, "published": true }
```

**URL เดียวกันเป๊ะ ได้ response shape ต่างกันตาม `Accept` header เท่านั้น** — นี่คือ content
negotiation ที่ทำงานถูกต้องตามหลัก REST: URI ระบุ resource, header ระบุ representation ที่ต้องการ

### ทำไมต้องมี `format.any` fallback เสมอ — บทเรียนจาก error จริงที่เจอตอนพัฒนา Part นี้

ระหว่างพัฒนาตัวอย่างของ Part นี้ ทดสอบส่ง `Accept: application/vnd.myapp.v1+json` (vendor media
type ของ Step 562 ที่ไม่เคยถูก `Mime::Type.register` ไว้เลย) เข้า endpoint ที่มีแค่
`format.jsonapi`/`format.json` โดยไม่มี fallback:

```bash
curl -s -w "\nHTTP %{http_code}\n" \
  -H "Accept: application/vnd.myapp.v1+json" "http://localhost:3000/api/posts"
# (body ว่างเปล่า)
# HTTP 500
```

ตรวจ log แล้วพบสาเหตุตรงๆ:

```
ActionController::UnknownFormat (ActionController::UnknownFormat):
```

เมื่อ `Accept` header ไม่ตรงกับ format ไหนที่ `respond_to` รู้จักเลยสักตัว (`jsonapi`/`json`) และ
ไม่มี `format.any`/`format.all` เป็นทางออกสำรอง Rails จะ raise `ActionController::UnknownFormat`
ทันที — ถ้าไม่ได้ `rescue_from` ไว้ (Part นี้ยัง rescue แค่ `ActiveRecord::RecordNotFound`)
กลายเป็น 500 ที่ client ไม่รู้เหตุผลเลยว่าทำไม request ที่ "ดูปกติดี" ถึง error

เพิ่ม `format.any { ... }` เป็นบรรทัดสุดท้ายในทุก `respond_to` block แก้ปัญหานี้ได้ทันที — เป็น
**pattern มาตรฐานที่ควรมีในทุก endpoint ของ API ที่เปิดให้ third-party client ที่เราไม่ได้ควบคุม
เรียกเข้ามา** เพราะไม่มีทางรู้ล่วงหน้าเลยว่า client จะส่ง `Accept` header อะไรมาบ้าง การ fallback
เป็น JSON ธรรมดาเสมอ **ปลอดภัยกว่าการปล่อยให้ error 500 หลุดออกไปหา client มาก**

> **สรุปหลักการของ Step นี้ที่ใช้ซ้ำได้ทุกที่:** เมื่อ API ต้องรองรับมากกว่าหนึ่ง response shape
> (ในที่นี้คือ JSON ธรรมดากับ JSON:API) ให้ `Accept` header เป็นตัวตัดสินใจผ่าน `respond_to` เสมอ
> ไม่ใช่เขียน `if request.headers["Accept"] == "..."` เองแบบ manual — และ **ต้องมี fallback เสมอ**
> เพราะ input จาก header ก็เป็น "ค่าที่ client ควบคุมได้" เหมือนกับ `params` ทุกประการ (หลักการ
> allowlist + graceful fallback เดียวกับที่เจอมาตลอดทั้ง Part 038 และ Part นี้)

---

## แบบฝึกหัด: `/api/v1/posts` เวอร์ชัน production พร้อม JSON:API + Cursor-based Pagination ครบวงจร

### โจทย์

สร้าง endpoint `GET /api/v1/posts` ที่รวมทุกอย่างที่เรียนใน Part นี้เข้าด้วยกัน:

1. ใช้ **URL-path versioning** (`/api/v1/...`) ผ่าน `Api::V1::BaseController` ที่ใช้ร่วมกันได้กับ
   controller อื่นในอนาคต
2. ตอบกลับเป็น **JSON:API shape** (`data`/`attributes`/`relationships`/`included`) ด้วย
   `jsonapi-serializer` รองรับ `?include=author,category`
3. ใช้ **cursor-based pagination** ด้วย `pagy_keyset` — จำนวนรายการต่อหน้าเลือกได้จาก
   `?limit=` แต่ต้องผ่าน allowlist `[5, 10, 25]` เท่านั้น
4. มี pagination metadata ใน `meta.next_cursor`/`links.next` และ `Link` HTTP header แบบ RFC 8288
5. รองรับทั้ง `Accept: application/vnd.api+json` (JSON:API) และ `Accept: application/json`
   (flat JSON) ผ่าน content negotiation พร้อม fallback ที่ปลอดภัย
6. จัดการ `404` ด้วย error shape ตามแนวทาง JSON:API
7. ทดสอบทุกอย่างด้วย curl จริงหลายชุด ยืนยันว่าการแบ่งหน้าไม่มีรายการซ้ำ/ขาดหายแม้จะมีข้อมูลใหม่
   เกิดขึ้นระหว่างทาง

### เฉลย

ไฟล์ต่อไปนี้เหมือนกับที่สร้างไว้ก่อนหน้านี้ทุกตัวอักษร ไม่ต้องแก้อะไรเพิ่ม — จึงไม่แสดงซ้ำ:
`app/models/{author,category,post}.rb` (ดู "เตรียมโปรเจกต์"), `app/serializers/*.rb` (Step 565),
`config/initializers/pagy.rb` และ `config/initializers/mime_types.rb` (Step 567/570),
`Api::V1::BaseController` (Step 563) ไฟล์เดียวที่ต้องประกอบร่างใหม่ทั้งหมดคือ
`Api::V1::PostsController`:

**`Api::V1::PostsController`**

```ruby
# app/controllers/api/v1/posts_controller.rb
# frozen_string_literal: true

module Api
  module V1
    class PostsController < BaseController
      ALLOWED_LIMITS = [5, 10, 25].freeze

      def index
        scope = Post.published.includes(:author, :category).order(id: :asc)
        @pagy, posts = pagy_keyset(scope, limit: items_per_page)
        pagy_headers_merge(@pagy)

        respond_to do |format|
          format.jsonapi { render json: jsonapi_collection(posts), content_type: BaseController::JSONAPI_CONTENT_TYPE }
          format.json    { render json: plain_json_collection(posts) }
          format.any     { render json: plain_json_collection(posts) }
        end
      end

      def show
        post = Post.includes(:author, :category).find(params[:id])

        respond_to do |format|
          format.jsonapi { render json: jsonapi_resource(post), content_type: BaseController::JSONAPI_CONTENT_TYPE }
          format.json    { render json: plain_json_resource(post) }
          format.any     { render json: plain_json_resource(post) }
        end
      end

      private

      def items_per_page
        limit = params[:limit].to_i
        ALLOWED_LIMITS.include?(limit) ? limit : 10
      end

      def include_relationships
        params[:include].to_s.split(",").map(&:strip) & %w[author category]
      end

      def jsonapi_collection(posts)
        PostSerializer.new(
          posts,
          include: include_relationships.map(&:to_sym),
          meta: pagination_meta(@pagy),
          links: pagination_links(@pagy)
        ).serializable_hash.to_json
      end

      def jsonapi_resource(post)
        PostSerializer.new(post, include: include_relationships.map(&:to_sym)).serializable_hash.to_json
      end

      def plain_json_collection(posts)
        { posts: posts.as_json(only: %i[id title body views published]), meta: pagination_meta(@pagy) }
      end

      def plain_json_resource(post)
        post.as_json(only: %i[id title body views published])
      end

      def pagination_meta(pagy)
        { limit: pagy.limit, next_cursor: pagy.next }
      end

      def pagination_links(pagy)
        { self: request.original_url, next: pagy.next ? pagy_keyset_next_url(pagy) : nil }
      end
    end
  end
end
```

**Routes**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show]
    end
  end
end
```

**ตรวจสอบด้วย curl จริงครบทุกเงื่อนไข**

```bash
# กรณีปกติ: JSON:API shape, limit=5
curl -s -H "Accept: application/vnd.api+json" "http://localhost:3000/api/v1/posts?limit=5" | jq .
```

```json
{
  "data": [
    { "id": "2", "type": "posts", "attributes": { "title": "บทความ #2", "views": 37, "published": true }, "relationships": { "author": { "data": { "id": "2", "type": "authors" } }, "category": { "data": { "id": "2", "type": "categories" } } } },
    { "id": "3", "type": "posts", "...": {} },
    { "id": "4", "type": "posts", "...": {} },
    { "id": "5", "type": "posts", "...": {} },
    { "id": "7", "type": "posts", "...": {} }
  ],
  "meta": { "limit": 5, "next_cursor": "eyJpZCI6N30" },
  "links": {
    "self": "http://localhost:3000/api/v1/posts?limit=5",
    "next": "/api/v1/posts?limit=5&page=eyJpZCI6N30"
  }
}
```

```bash
# เดินหน้าต่อด้วย cursor ที่ได้ — ต้องไม่มี id ซ้ำกับหน้าแรกเลย
curl -s -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts?limit=5&page=eyJpZCI6N30" | jq '[.data[].id]'
# ["8", "9", "10", "12", "13"]   <- ไม่ซ้ำกับหน้าแรก [2,3,4,5,7] เลย

# ทดสอบ include= สำหรับดึงข้อมูล author/category มาด้วย
curl -s -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts?limit=5&include=author,category" | jq '.included | length'
# 6   <- 3 authors + 3 categories เท่านั้น (dedup อัตโนมัติ)

# ทดสอบ allowlist ของ limit: ส่งค่านอก allowlist -> fallback เป็น 10 อัตโนมัติ
curl -s -H "Accept: application/vnd.api+json" "http://localhost:3000/api/v1/posts?limit=999" \
  | jq '.meta.limit'
# 10

# ทดสอบ content negotiation: JSON ธรรมดา
curl -s -H "Accept: application/json" "http://localhost:3000/api/v1/posts/2" | jq .
# {"id": 2, "title": "บทความ #2", "body": "...", "views": 37, "published": true}

# ทดสอบ 404 shape
curl -s -w "\nHTTP %{http_code}\n" -H "Accept: application/vnd.api+json" \
  "http://localhost:3000/api/v1/posts/9999"
```

```json
{ "errors": [{ "status": "404", "title": "Record Not Found", "detail": "Couldn't find Post with 'id'=\"9999\"" }] }
```
```
HTTP 404
```

```bash
# พิสูจน์ความสอดคล้อง (consistency) ของ cursor-based pagination ด้วยข้อมูลจริง
bin/rails runner '
  page1 = Post.published.order(id: :asc).limit(5).pluck(:id)
  cursor = page1.last
  Post.create!(title: "บทความด่วน", body: "...", author: Author.first,
               category: Category.first, published: true, views: 0)
  page2 = Post.published.where("id > ?", cursor).order(id: :asc).limit(5).pluck(:id)
  puts "page1=#{page1} page2=#{page2} duplicated=#{page1 & page2}"
'
# page1=[2, 3, 4, 5, 7] page2=[8, 9, 10, 12, 13] duplicated=[]
```

ทุกเงื่อนไขทำงานถูกต้องตามที่ออกแบบไว้: versioning ผ่าน URL path, response shape ตาม JSON:API
spec, pagination แบบ cursor ที่พิสูจน์แล้วว่าไม่มีรายการซ้ำแม้มีข้อมูลใหม่แทรกเข้ามาระหว่างทาง,
content negotiation รองรับทั้งสอง format, และ error handling ตอบ shape มาตรฐานเดียวกันทุกจุด

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม endpoint `GET /api/v1/authors/:id` ที่คืน JSON:API shape ของผู้เขียนคนเดียว พร้อม
   `?include=posts` ที่ดึงบทความทั้งหมดของผู้เขียนคนนั้นมาด้วย (ใบ้: `AuthorSerializer` ต้องเพิ่ม
   `has_many :posts` และต้อง `includes(:posts)` ใน controller ก่อน serialize เพื่อป้องกัน N+1
   ตามหลักการ Part 034 Step 335)
2. เพิ่มเวอร์ชัน `v2` ของ `Api::V1::PostsController` ที่เปลี่ยนชื่อ field `views` เป็น
   `view_count` (จำลอง breaking change จาก Step 561 ตรงๆ) โดยที่ `v1` ยังคงทำงานด้วยชื่อ `views`
   เหมือนเดิมทุกประการ ทั้งสองเวอร์ชันต้องอยู่คู่กันได้พร้อมกันจริงในระบบเดียว (ใบ้: สร้าง
   `Api::V2::BaseController`/`Api::V2::PostsController` แยกใหม่ทั้งหมด อย่าให้ `Api::V2` inherit
   จาก `Api::V1` เพราะจะทำให้สองเวอร์ชันผูกติดกันโดยไม่ตั้งใจ — ทดสอบด้วย curl ยิงทั้งสอง URL
   คู่ขนานกันแล้วยืนยันว่าฟิลด์ต่างกันจริงตามที่ตั้งใจ)
3. เขียน request spec (ทบทวนจาก Part 046) ที่ยืนยันพฤติกรรม `format.any` fallback ของ Step 570:
   ส่ง `Accept` header ที่เป็น vendor media type แปลกๆ ที่ไม่เคยถูก register เลย แล้วยืนยันว่า
   response ยังคงเป็น `200 OK` พร้อม body เป็น JSON ธรรมดา (ไม่ใช่ `500`) — เพิ่ม test อีกเคสที่
   ยืนยันว่า response ปกติ (`Accept: application/vnd.api+json`) ยังคงมี key `data`/`meta`/`links`
   ครบตามโครงสร้างที่ควรจะเป็น

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **API versioning ไม่ใช่ทางเลือก แต่เป็นข้อกำหนดเมื่อมี client ที่ backend ควบคุมเวอร์ชันไม่ได้**
  (mobile app ที่ force-update ไม่ได้ทันที, third-party integration, SPA ที่ cache เก่าไว้ใน
  browser) — breaking change ใดๆ (เปลี่ยนชื่อ/ชนิด/ความหมายของ field) ที่ deploy โดยไม่มี versioning
  รองรับ จะทำให้ client เวอร์ชันเก่าพังทันทีโดยทีม backend ไม่รู้ตัว
- **สามกลยุทธ์ versioning หลัก** — URL-path (`/api/v1/...`, ทดสอบง่ายที่สุด, cache ง่ายที่สุด),
  Accept-header (`Accept: application/vnd.myapp.v1+json`, ตรงหลัก REST มากกว่าแบบ GitHub API),
  custom header (`X-API-Version:`, แยกเรื่อง version ออกจาก content format ชัดเจนแบบ Stripe) —
  ไม่มีแบบไหนถูกที่สุดเสมอไป Part นี้ implement ทั้งสามแบบจริงและเลือก URL-path เป็นหลักเพราะ debug
  ง่ายที่สุด
- **URL-path versioning ทำผ่าน `namespace :api do namespace :v1 do ... end end`** พร้อมแยก
  `Api::V1::BaseController` (inherit จาก `ActionController::API` ตรงๆ ไม่ผ่าน
  `ApplicationController`) เพื่อให้แต่ละเวอร์ชันเปลี่ยน behavior ได้อิสระจากกันในอนาคต
- **JSON:API spec** คือมาตรฐานเปิดของ shape ของ JSON response (`data`/`attributes`/
  `relationships`/`included`/`meta`/`links`) ที่แก้ปัญหา "แต่ละ API ออกแบบ shape ต่างกันไปหมด"
  ทำให้ client tooling (SDK, codegen) ใช้ generic parser ร่วมกันได้ — `relationships` แยกจาก
  `included` เพื่อไม่ให้ข้อมูลซ้ำซ้อนเวลามีหลาย record อ้างถึง resource เดียวกัน
- **`jsonapi-serializer`** (เดิมชื่อ `fast_jsonapi`) implement shape นี้ให้อัตโนมัติผ่าน
  `include JSONAPI::Serializer`, `attributes`, `belongs_to`/`has_many` — output ตรงกับที่เขียน
  ด้วยมือทุกตัวอักษร แต่จัดการ `included`/dedup logic ให้ฟรี ค่า `include:` ที่มาจาก
  `params[:include]` ต้อง allowlist เสมอด้วย set intersection (`& %w[author category]`)
- **Offset-based pagination (`LIMIT`/`OFFSET`) มีปัญหา consistency จริงเมื่อมี concurrent write**
  — พิสูจน์ด้วยข้อมูลจริงแล้วว่า insert ระหว่างสอง request ทำให้เกิดรายการซ้ำข้ามหน้า และ delete
  ระหว่างสอง request ทำให้รายการที่ไม่เกี่ยวข้องเลยหายไปจากทุกหน้า — **cursor-based pagination**
  (`WHERE column > :cursor`) ไม่มีปัญหานี้เพราะอ้างอิงจาก record จริง ไม่ใช่ตำแหน่งเชิงตัวเลข
  แลกมาด้วยการไม่มี random access (`?page=5`) และไม่มี `total_count`/`total_pages`
- **pagy extras `keyset`/`headers`/`metadata`** implement cursor-based pagination ให้พร้อมใช้
  (`pagy_keyset`), สร้าง `Link` HTTP header ตาม RFC 8288 แบบเดียวกับ GitHub API
  (`pagy_headers_merge`) — keyset ให้แค่ `rel="first"`/`rel="next"` (เดินหน้าทางเดียว) ในขณะที่
  offset-based ปกติให้ `rel="first"`/`rel="prev"`/`rel="next"`/`rel="last"` ครบ (random access
  เต็มรูปแบบ)
- **สัญญา (contract) ของ pagination ต้องระบุชัดเจนเสมอ** — กลยุทธ์ที่ใช้และเหตุผล, ชื่อ/ชนิดของ
  query param, พฤติกรรมของค่าที่ผิด allowlist, และย้ำว่า **cursor เป็น opaque token ที่ห้าม client
  parse โครงสร้างภายในเอง** เพราะไม่ถือเป็นส่วนหนึ่งของ contract ที่รับประกันความเสถียร
- **HATEOAS** (ลิงก์บอกว่า "ทำอะไรต่อได้บ้าง" ฝังอยู่ใน response) เป็นหลักการดั้งเดิมของ REST แต่
  API ระดับ production ส่วนใหญ่ไม่ implement เต็มรูปแบบเพราะต้นทุนสูงกว่าประโยชน์ — สิ่งที่ทำจริง
  คือ "HATEOAS แบบเบาๆ" เฉพาะจุดที่คุ้มค่าชัดเจน เช่น `links.next` ของ pagination ที่ Part นี้ทำไป
  แล้วโดยไม่ต้อง implement state machine เต็มรูปแบบ
- **Content negotiation** ผ่าน `Accept`/`Content-Type` header คนละทิศทางกัน (`Accept` = client
  ขอ, `Content-Type` = บอกว่าข้อมูลที่แนบมาเป็นรูปแบบอะไร) — `ActionController::API` ไม่มี
  `respond_to` ให้ฟรีต้อง `include ActionController::MimeResponds` เอง, media type ใหม่ต้อง
  `Mime::Type.register` ก่อนใช้กับ `format.xxx`, และ **ต้องมี `format.any` fallback เสมอ** ไม่งั้น
  `Accept` header ที่ไม่รู้จักจะทำให้เกิด `ActionController::UnknownFormat` (500) ใส่ client แทนที่
  จะตอบอะไรบางอย่างกลับไปอย่างปลอดภัย

**ต่อไป (Part 058):** Part นี้ยัง REST อยู่ — client ยังต้องรู้ล่วงหน้าว่า endpoint ไหนคืน field
อะไรบ้าง (`attributes` ทั้งหมดของ `PostSerializer` เสมอ ต่อให้ client ต้องการแค่ `title` field
เดียว) และถ้าต้องการข้อมูลที่ซ้อนกันหลายชั้น (บทความ + ผู้เขียน + บทความอื่นของผู้เขียนคนนั้น) ต้อง
ยิงหลาย request หรือพึ่ง `?include=` ที่ตายตัวตามที่ server กำหนดไว้ล่วงหน้าเท่านั้น — Part ถัดไปจะ
แนะนำ **GraphQL** ด้วย gem `graphql-ruby` ซึ่งแก้ปัญหานี้จากคนละมุม: **client เป็นฝ่ายเลือก field
และความลึกของข้อมูลที่ต้องการเองทั้งหมดในคำขอเดียว** ผ่านการนิยาม **schema**, **type**, และ
**query** — เริ่มต้นตั้งแต่การสร้าง GraphQL endpoint แรกใน Rails ไปจนถึงการ query ข้อมูลที่มี
ความสัมพันธ์ซับซ้อนในคำขอเดียวโดยไม่ over-fetch หรือ under-fetch เหมือนที่ REST มักเจอ
