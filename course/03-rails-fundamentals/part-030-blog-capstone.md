# Part 030: โปรเจกต์รวบยอดเฟส 3 — Blog แบบง่าย (Post + Comment) CRUD ครบ

> **Step ครอบคลุมใน Part นี้:** Step 291–300
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 021–029 มาก่อนทั้งหมด — โดยเฉพาะ Part 022 Routing,
> Part 023 Controller, Part 024 View, Part 025 Model/ActiveRecord, Part 026 Validation/Callback,
> Part 027 Association, Part 028 Scaffold/Generator, และ Part 029 Asset Pipeline/Propshaft)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (โปรเจกต์ทั้งหมดใน Part นี้สร้างและทดสอบรันจริง
> บน Ruby 3.3.6 + Rails 8.1.4 — ทุก request ที่แสดงในเอกสารนี้ยิงด้วย `curl` จริงกับ server ที่
> รันอยู่จริง ไม่ใช่โค้ดที่เขียนลอยๆ)

นี่คือ **Part สุดท้ายของ Phase 3: Rails Fundamentals — MVC** ตลอด 9 Part ที่ผ่านมา เราเรียนรู้
แต่ละชั้นของ MVC แยกกันทีละส่วน: การติดตั้ง Rails และโครงสร้างโฟลเดอร์ (Part 021), Routing
(Part 022), Controller (Part 023), View (Part 024), Model/ActiveRecord/Migration (Part 025),
Validation/Callback (Part 026), Association (Part 027), Scaffold/Generator (Part 028), และ Asset
Pipeline ด้วย Propshaft (Part 029) — ถึงเวลาแล้วที่จะเอาทุกชิ้นส่วนเหล่านั้นมาประกอบเป็น
**เว็บแอปพลิเคชันที่ใช้งานได้จริงตัวแรก**

โปรเจกต์ปิดท้ายเฟสนี้คือ **Mini Blog** — ระบบบล็อกอย่างง่ายที่มี 2 โมเดลหลัก:

- **Post** (บทความ) — เขียน อ่าน แก้ไข ลบได้ครบ (CRUD เต็มรูปแบบ)
- **Comment** (ความคิดเห็น) — แต่ละความคิดเห็น "เป็นของ" บทความหนึ่งอันเสมอ (`belongs_to :post`)
  เพิ่ม/ลบได้ (ไม่ต้องมีหน้าแก้ไขความคิดเห็นแยก เพื่อให้ scope ของโปรเจกต์อยู่ในระดับที่จัดการ
  ได้ภายใน 10 Step)

ขอบเขตของโปรเจกต์นี้ **ตั้งใจทำให้เล็กและจบได้จริง** — ไม่มีระบบ login, ไม่มี tag, ไม่มี
pagination (ทั้งหมดนี้จะเป็นหัวข้อของเฟสถัดๆ ไป) เป้าหมายของ Part นี้คือพิสูจน์ว่า **เข้าใจ MVC
ครบวงจรจริงๆ แล้ว** — routing ส่ง request ไปหา controller ที่ถูกต้อง, controller คุยกับ model
ผ่าน strong parameters, model validate ข้อมูลก่อนบันทึกลงฐานข้อมูลจริง, และ view render ผลลัพธ์
กลับมาเป็น HTML ที่ใช้งานได้จริงในเบราว์เซอร์ — ไม่ใช่แค่ทฤษฎีแยกส่วนเหมือน 9 Part ที่ผ่านมาอีก
ต่อไป

## สารบัญของ Part นี้

- Step 291: วางแผนโปรเจกต์ — `rails new mini_blog` และออกแบบโดเมน Post/Comment
- Step 292: Model `Post` — migration, validation, scope
- Step 293: Model `Comment` — migration, association, validation
- Step 294: `config/routes.rb` — nested resources แบบ `shallow` สำหรับ comments
- Step 295: `PostsController` — CRUD ครบ 7 action พร้อม strong parameters และ flash/redirect
- Step 296: `CommentsController` — เฉพาะ `create`/`destroy` ที่ nest ใต้ post
- Step 297: Views — layout, partial ของฟอร์ม, และ `render partial:, collection:` สำหรับ comment
- Step 298: Stylesheet ผ่าน Propshaft และการใช้ view helper (`time_ago_in_words`, `truncate`,
  `pluralize`) อย่างมีความหมาย
- Step 299: `db/seeds.rb` และทดสอบ user flow เต็มรูปแบบด้วยข้อมูลจริง
- Step 300: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 3

---

## Step 291: วางแผนโปรเจกต์ — `rails new mini_blog` และออกแบบโดเมน Post/Comment

### สร้างโปรเจกต์ใหม่

ทบทวนจาก **Part 021 Step 203**: เฟส 3 ทั้งหมดใช้ flag `--minimal` เพื่อตัดฟีเจอร์ที่ยังไม่ได้
เรียน (Hotwire, Active Job, Active Storage) ออกไปก่อน ให้โฟกัสที่ MVC ล้วนๆ — โปรเจกต์ปิดท้าย
เฟสนี้ก็ยังคงใช้หลักการเดียวกัน เพื่อให้เห็นชัดว่า **แค่ HTML form ธรรมดา + Rails MVC ก็สร้าง
เว็บแอปที่ใช้งานได้จริงแล้ว** โดยไม่ต้องพึ่ง JavaScript framework ใดๆ เลย

```bash
mkdir -p ~/ruby-course-workspace/part-030
cd ~/ruby-course-workspace/part-030

rails new mini_blog --minimal
cd mini_blog
```

ตรวจสอบ `Gemfile` ที่ได้ (ทดสอบจริงแล้ว — ตรงตามตารางเปรียบเทียบใน Part 021 Step 203 เป๊ะ):

```ruby
# Gemfile (ส่วนที่ไม่ใช่ comment)
source "https://rubygems.org"

gem "rails", "~> 8.1.4"
gem "propshaft"
gem "sqlite3", ">= 2.1"
gem "puma", ">= 5.0"
gem "tzinfo-data", platforms: %i[ windows jruby ]

group :development, :test do
  gem "debug", platforms: %i[ mri windows ], require: "debug/prelude"
end
```

สังเกตว่ามี **`propshaft`** ติดมาแม้จะใช้ `--minimal` (asset pipeline ยังจำเป็นเสมอ ตามตารางใน
Part 021) แต่ไม่มี `turbo-rails`/`importmap-rails` — หมายความว่าโปรเจกต์นี้**ไม่มี JavaScript
ใดๆ เข้ามาเกี่ยวข้องเลย** ทุกฟอร์มจะเป็น HTML form ธรรมดาที่ submit แบบ full-page reload
100% — ปุ่ม "ลบ" ที่ปกติในแอป Rails เต็มรูปแบบมักใช้ `data: { turbo_confirm: "..." }` เพื่อเด้ง
popup ยืนยันจะ**ใช้ไม่ได้**ในโปรเจกต์นี้ (ไม่มี Turbo คอยดัก JavaScript event) เราจะใช้
`button_to` แทน `link_to ... method: :delete` เสมอสำหรับปุ่มลบ เพราะ `button_to` render เป็น
`<form>` HTML ล้วนๆ ที่ทำงานได้แน่นอนแม้ไม่มี JavaScript เลยสักบรรทัด (ทบทวนเรื่อง HTML form
ส่ง verb ปลอมผ่าน `_method` hidden field จาก Part 022 Step 211)

### ออกแบบโดเมน: Post กับ Comment สัมพันธ์กันอย่างไร

ก่อนเขียนโค้ดสักบรรทัด ให้วาดภาพความสัมพันธ์ของข้อมูลให้ชัดเจนก่อนเสมอ (ทบทวนหลักการนี้จาก
**Part 025 Step 241** เรื่องออกแบบ schema ก่อนเขียน migration):

```
┌─────────────────────┐          ┌─────────────────────┐
│        Post          │  1    N  │       Comment        │
├─────────────────────┤─────────►├─────────────────────┤
│ id                   │          │ id                   │
│ title    (string)    │          │ post_id  (FK)        │
│ body     (text)      │          │ commenter (string)   │
│ created_at           │          │ body      (text)     │
│ updated_at           │          │ created_at           │
└─────────────────────┘          │ updated_at           │
                                   └─────────────────────┘
```

- **Post** `has_many :comments` — บทความหนึ่งอันมีความคิดเห็นได้หลายอัน (ทบทวน **Part 027**)
- **Comment** `belongs_to :post` — ความคิดเห็นหนึ่งอันต้องเป็นของบทความอันใดอันหนึ่งเสมอ
  (ไม่มี comment ที่ "ลอย" อยู่โดยไม่มี post เจ้าของ)
- **ไม่มีโมเดล User** ในโปรเจกต์นี้โดยตั้งใจ — เก็บชื่อผู้แสดงความคิดเห็นเป็น string ธรรมดา
  (`commenter`) แทนการอ้างอิงไปยัง user account เพราะระบบ authentication ยังไม่ได้เรียน (จะ
  เรียนใน **Phase 5**) — Step 300 จะพูดถึงจุดนี้อีกครั้งตอนวิเคราะห์ว่าโปรเจกต์ขาดอะไรสำหรับ
  production

### วางแผนหน้าจอ (Screens) ที่ต้องมีให้ครบ

| หน้าจอ | URL | หน้าที่ | อ้างอิง Part |
|---|---|---|---|
| รายการบทความ | `GET /posts` | แสดงบทความทั้งหมด เรียงใหม่สุดก่อน | Part 022 (`index`) |
| ดูบทความ + ความคิดเห็น | `GET /posts/:id` | แสดงเนื้อหาเต็ม + comment ทั้งหมด + ฟอร์มเพิ่ม comment | Part 022, 027 |
| เขียนบทความใหม่ | `GET /posts/new` → `POST /posts` | ฟอร์มสร้าง + บันทึก | Part 023, 026 |
| แก้ไขบทความ | `GET /posts/:id/edit` → `PATCH /posts/:id` | ฟอร์มแก้ไข + บันทึก | Part 023, 026 |
| ลบบทความ | `DELETE /posts/:id` | ลบพร้อม comment ทั้งหมด (`dependent: :destroy`) | Part 027 |
| เพิ่มความคิดเห็น | `POST /posts/:post_id/comments` | บันทึก comment ใหม่ใต้ post | Part 022 (nested) |
| ลบความคิดเห็น | `DELETE /comments/:id` | ลบ comment เดียว (URL สั้นด้วย `shallow: true`) | Part 022 Step 217 |

### โครงสร้างไฟล์ที่จะสร้างใน Part นี้

```
mini_blog/
├── app/
│   ├── models/
│   │   ├── post.rb
│   │   └── comment.rb
│   ├── controllers/
│   │   ├── posts_controller.rb
│   │   └── comments_controller.rb
│   ├── views/
│   │   ├── layouts/application.html.erb
│   │   ├── posts/
│   │   │   ├── index.html.erb
│   │   │   ├── show.html.erb
│   │   │   ├── new.html.erb
│   │   │   ├── edit.html.erb
│   │   │   ├── _form.html.erb
│   │   │   └── _post.html.erb
│   │   └── comments/
│   │       └── _comment.html.erb
│   └── assets/stylesheets/application.css
├── config/routes.rb
├── db/
│   ├── migrate/xxx_create_posts.rb
│   ├── migrate/xxx_create_comments.rb
│   ├── schema.rb
│   └── seeds.rb
```

สังเกตว่าโครงสร้างนี้**ไม่มีอะไรใหม่เลยสักไฟล์เดียว** — ทุกโฟลเดอร์คือสิ่งที่เรียนมาแล้วใน
Part 021 Step 205 (`app/models`, `app/controllers`, `app/views`) นี่คือเหตุผลที่โปรเจกต์นี้
เรียกว่า "รวบยอด" (capstone) — ไม่มีเทคนิคใหม่ มีแต่การเอาของเก่าที่เรียนแยกกันมาประกอบให้เป็น
ระบบเดียวที่ทำงานจริง

---

## Step 292: Model `Post` — migration, validation, scope

### สร้าง Model ด้วย generator

ทบทวนจาก **Part 025 Step 246**: `rails generate model` สร้างทั้ง migration file และ model
class เปล่าให้พร้อมกันในคำสั่งเดียว

```bash
bin/rails generate model Post title:string body:text
```

```
      invoke  active_record
      create    db/migrate/20260926030346_create_posts.rb
      create    app/models/post.rb
      invoke    test_unit
      create      test/models/post_test.rb
      create      test/fixtures/posts.yml
```

### แก้ไข migration ให้บังคับ `null: false`

Migration ที่ generator สร้างให้ยังไม่ได้ป้องกันค่า `NULL` ที่ระดับฐานข้อมูล เพิ่ม
`null: false` เข้าไปเอง (ทบทวนแนวคิด "validate สองชั้น" — ทั้งระดับ Model และระดับฐานข้อมูล —
จาก **Part 026**):

```ruby
# db/migrate/20260926030346_create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title, null: false
      t.text :body, null: false

      t.timestamps
    end
  end
end
```

**ทำไมต้องมี `null: false` ทั้งที่ Model จะมี `validates :title, presence: true` อยู่แล้ว
(ด้านล่าง):** เพราะ validation ระดับ Model ทำงานเฉพาะตอนบันทึกผ่าน ActiveRecord method
(`save`, `create`, `update`) เท่านั้น — ถ้ามีใครเขียน SQL ตรงๆ ไปที่ฐานข้อมูล (ผ่าน
`ActiveRecord::Base.connection.execute` หรือ script ภายนอกอื่นๆ) validation ของ Model จะไม่ถูก
เรียกเลย การมี constraint `NOT NULL` ที่ระดับฐานข้อมูลคือ **แนวป้องกันชั้นสุดท้าย**ที่รับประกัน
ความถูกต้องของข้อมูลไม่ว่าทางเข้าจะเป็นทางไหนก็ตาม

### `app/models/post.rb`

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true, length: { minimum: 10 }

  scope :recent, -> { order(created_at: :desc) }
end
```

**อธิบายทีละบรรทัด:**

- `has_many :comments, dependent: :destroy` — ประกาศความสัมพันธ์ (ทบทวน **Part 027**) พร้อม
  `dependent: :destroy` ที่สั่งให้ **ลบ comment ทุกอันของ post นี้ไปด้วยเสมอ**เมื่อ post ถูกลบ
  — ถ้าไม่ใส่ตัวเลือกนี้ การลบ post จะทิ้ง comment ที่ `post_id` ชี้ไปยัง post ที่ไม่มีอยู่แล้ว
  ไว้ในฐานข้อมูล (orphaned record) ซึ่งเป็นข้อมูลขยะที่อาจทำให้ query อื่นพังภายหลัง
- `validates :title, presence: true, length: { maximum: 120 }` — บังคับว่าต้องมีชื่อบทความ
  เสมอ และห้ามยาวเกิน 120 ตัวอักษร (ป้องกันชื่อบทความที่ยาวจนทำให้ layout หน้า index พัง)
- `validates :body, presence: true, length: { minimum: 10 }` — บังคับว่าเนื้อหาต้องมีอย่างน้อย
  10 ตัวอักษร (ป้องกันการโพสต์เนื้อหาที่สั้นเกินจะเรียกว่าเป็นบทความจริงๆ) — ทบทวนรูปแบบ
  `length: { minimum:, maximum: }` จาก **Part 026**
- `scope :recent, -> { order(created_at: :desc) }` — สร้าง class method `Post.recent` ที่คืน
  บทความเรียงจากใหม่ไปเก่า ใช้ `scope` (ไม่ใช่ `def self.recent`) เพราะเป็นการ query ธรรมดาไม่มี
  logic ซับซ้อน และได้ประโยชน์ที่ scope ต่อ chain กับ query อื่นได้ (เช่น
  `Post.recent.limit(5)`) — ทบทวนแนวคิด scope นี้จาก **Part 025 Step 250**

### ทดสอบ Model ผ่าน Rails Console

ทบทวนจาก **Part 021 Step 208**: `bin/rails console` คือ `irb` ที่รู้จักแอปทั้งแอป ใช้ทดสอบ
Model ก่อนต่อกับ Controller/View เสมอเป็นนิสัยที่ดี

```bash
bin/rails console
```

```irb
irb> Post.create(title: "", body: "สั้น")
=> #<Post id: nil, title: "", body: "สั้น", ...> (unpersisted — validation ไม่ผ่าน)

irb> post = Post.new(title: "", body: "สั้น")
irb> post.valid?
=> false
irb> post.errors.full_messages
=> ["Title can't be blank", "Body is too short (minimum is 10 characters)"]

irb> post = Post.create!(title: "บทความแรก", body: "นี่คือเนื้อหาของบทความแรกที่มีความยาวพอสมควร")
=> #<Post id: 1, title: "บทความแรก", ...>

irb> Post.recent
=> [#<Post id: 1, ...>]
```

`errors.full_messages` แสดงข้อความ validation เป็นภาษาอังกฤษตาม default locale ของ Rails —
เรื่องการแปลข้อความ error เป็นภาษาไทยจะเรียนเต็มรูปแบบใน **Part 039 (I18n)** ตอนนี้แค่รู้ว่า
validation ทำงานถูกต้องก็เพียงพอ

---

## Step 293: Model `Comment` — migration, association, validation

### สร้าง Model ด้วย `t.references` เพื่อสร้าง foreign key อัตโนมัติ

```bash
bin/rails generate model Comment post:references commenter:string body:text
```

```
      invoke  active_record
      create    db/migrate/20260926030347_create_comments.rb
      create    app/models/comment.rb
      invoke    test_unit
      create      test/models/comment_test.rb
      create      test/fixtures/comments.yml
```

สังเกตว่าการระบุ `post:references` ตอนเรียก generator ทำให้ Rails **สร้าง `belongs_to :post`
ให้ใน model อัตโนมัติทันที** (ทบทวนเทคนิคนี้จาก **Part 027**) — ต่างจาก `Post` ที่ไม่มี field
พิเศษ จึงต้องเติม `has_many :comments` เองด้วยมือใน Step 292

แก้ migration ให้บังคับ `null: false` เหมือนกับ `Post`:

```ruby
# db/migrate/20260926030347_create_comments.rb
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :post, null: false, foreign_key: true
      t.string :commenter, null: false
      t.text :body, null: false

      t.timestamps
    end
  end
end
```

`t.references :post, null: false, foreign_key: true` สร้าง column `post_id` (integer) พร้อม
**foreign key constraint** ที่ระดับฐานข้อมูลจริง (`foreign_key: true`) — ป้องกันไม่ให้สร้าง
comment ที่ `post_id` ชี้ไปยัง post ที่ไม่มีอยู่จริงได้เลยแม้จะพยายามผ่านทาง SQL ตรงๆ ก็ตาม
(ทบทวนความแตกต่างระหว่าง "แค่ column ธรรมดา" กับ "foreign key ที่มี constraint" จาก
**Part 027**)

### รัน Migration ทั้งสองไฟล์

```bash
bin/rails db:migrate
```

```
== 20260926030346 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0014s
== 20260926030346 CreatePosts: migrated (0.0014s) =============================

== 20260926030347 CreateComments: migrating ===================================
-- create_table(:comments)
   -> 0.0018s
== 20260926030347 CreateComments: migrated (0.0019s) ==========================
```

ตรวจสอบ `db/schema.rb` ที่ Rails สร้างใหม่ให้อัตโนมัติ (ทบทวนว่า `schema.rb` คือ source of
truth จาก **Part 025 Step 244**):

```ruby
# db/schema.rb
ActiveRecord::Schema[8.1].define(version: 2026_09_26_030347) do
  create_table "comments", force: :cascade do |t|
    t.integer "post_id", null: false
    t.string "commenter", null: false
    t.text "body", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["post_id"], name: "index_comments_on_post_id"
  end

  create_table "posts", force: :cascade do |t|
    t.string "title", null: false
    t.text "body", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end

  add_foreign_key "comments", "posts"
end
```

สังเกต 2 จุดที่ `t.references` สร้างให้อัตโนมัติ:

1. **`t.index ["post_id"], ...`** — สร้าง database index บน `post_id` ให้ทันที เพราะแทบทุก
   query ของ comment จะกรองด้วย `post_id` เสมอ (เช่น `post.comments`) index นี้ทำให้ query
   เหล่านั้นเร็วขึ้นมากเมื่อข้อมูลเยอะ (เจาะลึกเรื่อง index ใน **Part 064**)
2. **`add_foreign_key "comments", "posts"`** — constraint ระดับฐานข้อมูลที่พูดถึงข้างต้น

### `app/models/comment.rb`

```ruby
class Comment < ApplicationRecord
  belongs_to :post

  validates :commenter, presence: true, length: { maximum: 60 }
  validates :body, presence: true, length: { minimum: 2, maximum: 500 }
end
```

**อธิบาย:**

- `belongs_to :post` — สร้างมาให้อัตโนมัติจากตอนรัน generator (Rails 5+ ทำให้
  `belongs_to` เป็น **required by default** อยู่แล้ว หมายความว่าการสร้าง `Comment.new` โดยไม่
  ระบุ `post` จะ validate ไม่ผ่านทันทีแม้จะไม่ได้เขียน `validates :post, presence: true` เอง
  ก็ตาม — พฤติกรรมนี้ควบคุมได้ผ่าน `config.active_record.belongs_to_required_by_default` ใน
  `config/application.rb`)
- `validates :commenter` และ `validates :body` — เหตุผลเดียวกับ `Post` ใน Step 292 คือป้องกัน
  ความคิดเห็นว่างเปล่าหรือชื่อผู้แสดงความคิดเห็นว่างเปล่า พร้อมจำกัดความยาวสูงสุดไม่ให้ข้อความ
  ยาวเกินจนทำลาย layout ของหน้า

### ทดสอบ Association ผ่าน Rails Console

```bash
bin/rails console
```

```irb
irb> post = Post.first
irb> comment = post.comments.create(commenter: "ทดสอบ", body: "ความคิดเห็นแรก")
=> #<Comment id: 1, post_id: 1, commenter: "ทดสอบ", body: "ความคิดเห็นแรก", ...>

irb> post.comments.count
=> 1

irb> comment.post
=> #<Post id: 1, title: "บทความแรก", ...>

irb> Comment.new(commenter: "x", body: "y")
irb> Comment.new(commenter: "x", body: "y").valid?
=> false   # ไม่ผ่านเพราะไม่มี post (belongs_to required by default)

irb> post.destroy
irb> Comment.count
=> 0   # comment ถูกลบตามไปด้วย เพราะ dependent: :destroy ที่ตั้งไว้ใน Post
```

บรรทัดสุดท้ายพิสูจน์ว่า `dependent: :destroy` ที่ประกาศไว้ใน `Post` (Step 292) ทำงานจริง — ลบ
`post` ครั้งเดียว `comment` ทุกตัวที่เป็นของ post นั้นหายไปด้วยอัตโนมัติ ไม่ทิ้งข้อมูลขยะไว้เลย

---

## Step 294: `config/routes.rb` — nested resources แบบ `shallow` สำหรับ comments

ทบทวนจาก **Part 022 Step 217**: comment เป็นตัวอย่างคลาสสิกของ resource ที่ควร nest ใต้ post
เพราะ comment ไม่มีความหมายถ้าไม่มี post เป็นเจ้าของ แต่ควรใช้ `shallow: true` เพื่อไม่ให้ URL
ของ action ที่ไม่จำเป็นต้องรู้ post (`destroy`) ยาวเกินความจำเป็น

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  root "posts#index"

  resources :posts do
    resources :comments, only: %i[create destroy], shallow: true
  end
end
```

**อธิบายการตัดสินใจแต่ละบรรทัด:**

- `root "posts#index"` — หน้าแรกของเว็บคือรายการบทความทั้งหมด (ทบทวน **Part 022 Step 220**)
  วางไว้บนสุดของไฟล์ (หลัง health check) ตามธรรมเนียมที่แนะนำไว้
- `resources :posts` — ไม่ใส่ `only:`/`except:` เพราะ Post ต้องการครบทั้ง 7 action จริง
  (index, show, new, create, edit, update, destroy)
- `resources :comments, only: %i[create destroy]` — **จำกัดเหลือแค่ 2 action** (ทบทวน
  **Part 022 Step 213**) เพราะโปรเจกต์นี้ไม่มีหน้า "ดูความคิดเห็นทีละอัน" หรือ "แก้ไข
  ความคิดเห็น" แยกต่างหาก — comment ถูกสร้างและลบเท่านั้น (แสดงผลรวมอยู่ในหน้า `posts#show`
  อยู่แล้ว)
- `shallow: true` — ทำให้ `create` ยังคงอยู่ใต้ `/posts/:post_id/comments` (เพราะตอนสร้างต้อง
  รู้ว่าเป็น comment ของ post ไหน) แต่ `destroy` สั้นลงเหลือแค่ `/comments/:id` (เพราะการลบรู้
  แค่ id ของ comment ก็เพียงพอ ไม่ต้องรู้ post)

### ตรวจสอบผลลัพธ์ด้วย `bin/rails routes`

```bash
bin/rails routes
```

ผลลัพธ์จริง (ทดสอบบน Rails 8.1.4):

```
            Prefix Verb   URI Pattern                        Controller#Action
rails_health_check GET    /up(.:format)                      rails/health#show
              root GET    /                                  posts#index
     post_comments POST   /posts/:post_id/comments(.:format) comments#create
           comment DELETE /comments/:id(.:format)            comments#destroy
             posts GET    /posts(.:format)                   posts#index
                   POST   /posts(.:format)                   posts#create
          new_post GET    /posts/new(.:format)               posts#new
         edit_post GET    /posts/:id/edit(.:format)          posts#edit
              post GET    /posts/:id(.:format)               posts#show
                   PATCH  /posts/:id(.:format)               posts#update
                   PUT    /posts/:id(.:format)               posts#update
                   DELETE /posts/:id(.:format)               posts#destroy
```

ตรงตามที่คาดไว้ทุกประการ — สังเกตว่า:

- `post_comments` (POST) มี `:post_id` ใน URL — ต้องส่ง post เข้า route helper เสมอ:
  `post_comments_path(@post)`
- `comment` (DELETE) **ไม่มี** `:post_id` เพราะ `shallow: true` ตัดออกไปแล้ว — เรียกผ่าน
  `comment_path(comment)` โดยไม่ต้องรู้ post เลย
- ไม่มี route `new_post_comment`/`edit_comment`/`post_comment` (show) เลย เพราะ `only:
  %i[create destroy]` ตัดออกหมด — ตรงตามที่ตั้งใจว่าไม่มีหน้าความคิดเห็นแยกต่างหาก

> **ทดสอบ error ที่ตั้งใจให้เห็นก่อน** — ถ้าลืมใส่ `shallow: true` แล้วเผลอเรียก
> `comment_path(comment)` (ไม่ส่ง post) ในหน้า view จะได้ error ทันทีตอน render:
> `ActionController::UrlGenerationError: No route matches {:action=>"destroy",
> :controller=>"comments"}, missing required keys: [:post_id]` — เป็น error ที่พบบ่อยมาก
> เวลาทำ nested resources ผิดจุด ถ้าเจอ error นี้ให้กลับมาเช็คที่ `shallow:` ก่อนเป็นอันดับแรก

---

## Step 295: `PostsController` — CRUD ครบ 7 action พร้อม strong parameters และ flash/redirect

ทบทวนรูปแบบทั้งหมดในหัวข้อนี้จาก **Part 023**: `before_action`, `params`, `flash`/`flash.now`,
`redirect_to` vs `render`, และ Post/Redirect/Get pattern

### `app/controllers/posts_controller.rb`

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: %i[show edit update destroy]

  def index
    @posts = Post.recent
  end

  def show
    @comment = Comment.new
  end

  def new
    @post = Post.new
  end

  def create
    @post = Post.new(post_params)

    if @post.save
      redirect_to @post, notice: "สร้างบทความ \"#{@post.title}\" สำเร็จแล้ว"
    else
      flash.now[:alert] = "สร้างบทความไม่สำเร็จ กรุณาตรวจสอบข้อมูลอีกครั้ง"
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "แก้ไขบทความ \"#{@post.title}\" สำเร็จแล้ว"
    else
      flash.now[:alert] = "แก้ไขบทความไม่สำเร็จ กรุณาตรวจสอบข้อมูลอีกครั้ง"
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: "ลบบทความ \"#{@post.title}\" แล้ว", status: :see_other
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body)
  end
end
```

**อธิบายทีละจุดที่สำคัญ:**

- **`before_action :set_post, only: %i[show edit update destroy]`** — 4 action นี้ต้องหา
  `@post` จาก `params[:id]` เหมือนกันทุกตัว การดึงออกมาไว้ใน `before_action` ตัวเดียว (ทบทวน
  **Part 023 Step 228**) ทำให้ไม่ต้องเขียน `Post.find(params[:id])` ซ้ำ 4 รอบ — สังเกตว่า
  `index`, `new`, `create` ไม่อยู่ในรายการนี้เพราะ `index`/`new` ยังไม่มี id ให้หา และ `create`
  กำลัง**สร้าง**ใหม่ ไม่ใช่หา post ที่มีอยู่แล้ว
- **`index`** — เรียก `Post.recent` (scope ที่นิยามไว้ใน Step 292) แทน `Post.all` เพื่อให้
  บทความใหม่ล่าสุดขึ้นก่อนเสมอ
- **`show`** — นอกจาก `@post` ที่ `before_action` เตรียมไว้ให้แล้ว ยังสร้าง `@comment =
  Comment.new` ไว้ล่วงหน้าด้วย เพราะหน้า `show` มีฟอร์มเพิ่มความคิดเห็นอยู่ในหน้าเดียวกัน (ดู
  Step 297) — `form_with model: @comment` ใน view ต้องการ object เปล่าตัวนี้เพื่อสร้างฟอร์ม
  ให้ถูกต้อง (ทบทวนแนวคิดนี้จาก **Part 023**: controller เตรียมข้อมูลทุกอย่างที่ view ต้องใช้
  ไว้ล่วงหน้าเสมอ ไม่ควรให้ view ไปสร้าง object เอง)
- **`create`** — ใช้รูปแบบ if/else มาตรฐานที่เรียนใน **Part 023 Step 227** เป๊ะ: สำเร็จ
  → `redirect_to @post` (polymorphic routing ที่เดา `post_path(@post)` ให้อัตโนมัติ) พร้อม
  `notice:` แบบ shorthand (เทียบเท่า `flash[:notice] = "..."` ตามด้วย `redirect_to`) ล้มเหลว
  → `flash.now[:alert]` + `render :new, status: :unprocessable_entity` (ไม่ redirect เพราะ
  ต้องการให้ผู้ใช้เห็นข้อมูลที่กรอกไว้เดิมพร้อมข้อความ error โดยไม่ต้องกรอกใหม่ทั้งหมด)
- **`update`** — โครงสร้างเหมือน `create` ทุกประการ เพียงแค่เรียก `@post.update(post_params)`
  แทน `Post.new(post_params)` เพราะ `@post` มีอยู่แล้วจาก `before_action`
- **`destroy`** — เรียก `@post.destroy` (ซึ่งลบ comment ทั้งหมดไปด้วยเพราะ `dependent:
  :destroy`) แล้ว `redirect_to posts_path` กลับไปหน้ารายการ **สังเกต `status: :see_other`**
  ที่เพิ่มเข้ามา — นี่คือ convention มาตรฐานของ Rails ตั้งแต่ 7.1 เป็นต้นมา: เมื่อ redirect
  ตามหลัง request ที่ไม่ใช่ `GET` (เช่น `DELETE`, `POST`, `PATCH`) ควรตอบกลับด้วย **HTTP 303
  See Other** แทน 302 ธรรมดา เพื่อบอกเบราว์เซอร์อย่างชัดเจนว่า **request ถัดไปต้องเป็น `GET`
  เสมอ** ไม่ว่า request เดิมจะเป็น verb อะไรก็ตาม — ป้องกันปัญหาที่บาง proxy/เบราว์เซอร์รุ่นเก่า
  พยายาม redirect ด้วย verb เดิม (`DELETE`) ซ้ำ ซึ่งจะทำให้ error เพราะ `DELETE
  /posts` ไม่มี route
- **`post_params` (strong parameters)** — `params.require(:post).permit(:title, :body)` คือ
  รูปแบบมาตรฐานที่เรียนใน **Part 023 Step 224**: `require(:post)` บังคับว่าต้องมี key
  `"post"` ครอบอยู่ใน params เสมอ (ไม่งั้น raise `ActionController::ParameterMissing`) ส่วน
  `permit(:title, :body)` คือ **whitelist** ของฟิลด์ที่อนุญาตให้ mass-assign ได้เท่านั้น — ถ้า
  ผู้ใช้ (หรือผู้ไม่หวังดี) พยายามส่ง field อื่นที่ไม่ได้ permit มาด้วย (เช่น
  `post[created_at]=...`) ฟิลด์นั้นจะถูกตัดทิ้งเงียบๆ โดยอัตโนมัติ ไม่ทำให้เกิด mass
  assignment vulnerability

### ทดสอบด้วย `curl` ยืนยันว่า validation error คืน HTTP 422 จริง

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/new -o new.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=" \
  --data-urlencode "post[body]=สั้น" \
  -o invalid_resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

ทดสอบจริงยืนยันว่าเมื่อส่ง `title` ว่างเปล่าและ `body` สั้นเกินไป จะได้ **HTTP 422
Unprocessable Entity** กลับมาพร้อมหน้าฟอร์มเดิมที่แสดงข้อความ error ("is too short" มาจาก
validation `length: { minimum: 10 }`) ตรงตามที่ออกแบบไว้ทุกประการ ไม่มีการสร้าง record ที่ไม่
ถูกต้องหลุดลงฐานข้อมูลเลย

---

## Step 296: `CommentsController` — เฉพาะ `create`/`destroy` ที่ nest ใต้ post

### `app/controllers/comments_controller.rb`

```ruby
class CommentsController < ApplicationController
  before_action :set_post, only: %i[create]
  before_action :set_comment, only: %i[destroy]

  def create
    @comment = @post.comments.build(comment_params)

    if @comment.save
      redirect_to post_path(@post), notice: "เพิ่มความคิดเห็นสำเร็จแล้ว"
    else
      redirect_to post_path(@post), alert: "เพิ่มความคิดเห็นไม่สำเร็จ: #{@comment.errors.full_messages.to_sentence}"
    end
  end

  def destroy
    post = @comment.post
    @comment.destroy
    redirect_to post_path(post), notice: "ลบความคิดเห็นแล้ว", status: :see_other
  end

  private

  def set_post
    @post = Post.find(params[:post_id])
  end

  def set_comment
    @comment = Comment.find(params[:id])
  end

  def comment_params
    params.require(:comment).permit(:commenter, :body)
  end
end
```

**อธิบายจุดที่ต่างจาก `PostsController` อย่างมีนัยสำคัญ:**

- **`set_post` ใช้ `params[:post_id]` ไม่ใช่ `params[:id]`** — เพราะ route ของ `create` คือ
  `POST /posts/:post_id/comments` (ทบทวนจาก **Part 022 Step 217**) `:post_id` มาจากส่วน
  nested ของ URL ส่วน `:id` (ถ้ามี) จะหมายถึง id ของ comment เอง
- **`@post.comments.build(comment_params)`** — ใช้ `.build` ผ่าน **association** (`@post.
  comments`) แทนการเขียน `Comment.new(comment_params.merge(post_id: @post.id))` ตรงๆ เพราะ
  `build` ผ่าน association จะ **set `post_id` ให้อัตโนมัติ** โดยไม่ต้องยุ่งกับ id เองเลย
  (ทบทวนแนวคิดนี้จาก **Part 027**) — ปลอดภัยกว่าด้วย เพราะ `comment_params` ที่มาจากฟอร์มไม่มี
  ทางมี key `:post_id` ปนมา (ไม่ได้ permit ไว้) จึงไม่มีช่องให้ผู้ใช้ปลอม `post_id` ไปเป็น
  post ของคนอื่น
- **`create` เมื่อ validation ไม่ผ่าน ก็ยัง `redirect_to` เหมือนกรณีสำเร็จ** (ต่างจาก
  `PostsController#create` ที่ `render :new` กลับ) — เพราะฟอร์มเพิ่มความคิดเห็นอยู่ใน**หน้า
  เดียวกัน**กับที่แสดง comment ทั้งหมด (`posts#show`) การ `render` จาก `CommentsController`
  ตรงๆ จะต้อง render view ของ controller อื่น (ยุ่งยากและผิดหลัก separation of concerns) จึง
  เลือก `redirect_to` กลับไปหน้า post พร้อม `flash[:alert]` ที่รวมข้อความ error ทั้งหมดไว้แทน
  — เป็น trade-off ที่ยอมรับได้ เพราะข้อมูลที่กรอกในฟอร์ม comment (ปกติสั้น) จะหายไปต้องพิมพ์
  ใหม่ ต่างจากฟอร์มบทความที่มักยาวกว่ามากซึ่งไม่อยากให้ผู้ใช้พิมพ์ซ้ำ
- **`errors.full_messages.to_sentence`** — `to_sentence` เป็น method ของ `Array` ที่ Rails
  เพิ่มเข้ามา (ผ่าน ActiveSupport) แปลง Array ของ String ให้เป็นประโยคเดียวที่อ่านง่าย เช่น
  `["Commenter can't be blank", "Body is too short"]` กลายเป็น `"Commenter can't be blank
  and Body is too short"` — สะดวกมากเวลาต้องการยัดข้อความ error หลายอันไว้ใน `flash` ค่าเดียว
- **`destroy` เก็บ `post = @comment.post` ไว้ก่อนลบ** — เพราะหลังจาก `@comment.destroy` แล้ว
  การเรียก `@comment.post` ยังทำงานได้จริง (association ที่ load ไว้แล้วยังอยู่ใน memory) แต่
  การเขียนแบบนี้ชัดเจนกว่าและปลอดภัยกว่าในระยะยาว — ป้องกันกรณีมีการแก้ไขโค้ดภายหลังที่อาจทำให้
  ลำดับการทำงานเปลี่ยนไป

### ทดสอบด้วย `curl` ยืนยัน redirect กลับไปหน้า post หลังลบ comment

```bash
curl -s -c cookies.txt -b cookies.txt -X DELETE http://localhost:3000/comments/4 \
  -H "X-CSRF-Token: $TOKEN" \
  -D - -o /dev/null -w "HTTP %{http_code}\n"
```

```
HTTP/1.1 303 See Other
location: http://localhost:3000/posts/4
HTTP 303
```

ทดสอบจริงยืนยันว่า URL `/comments/4` (สั้น ไม่มี `post_id` ตามที่ `shallow: true` กำหนด) ลบ
comment สำเร็จ แล้ว redirect กลับไปที่ `/posts/4` (post ที่ comment นั้นเป็นสมาชิกอยู่) ด้วย
status 303 ตรงตามที่ออกแบบไว้

---

## Step 297: Views — layout, partial ของฟอร์ม, และ `render partial:, collection:` สำหรับ comment

ทบทวนรูปแบบทั้งหมดในหัวข้อนี้จาก **Part 024**: layout + `yield`, partial ที่ขึ้นต้นด้วย `_`,
`render partial:, collection:` สำหรับแสดงรายการ, และ `form_with` ที่แชร์ระหว่างหน้า new/edit

### `app/views/layouts/application.html.erb`

```erb
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "Mini Blog" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="application-name" content="Mini Blog">
    <meta name="mobile-web-app-capable" content="yes">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= yield :head %>

    <link rel="icon" href="/icon.png" type="image/png">
    <link rel="icon" href="/icon.svg" type="image/svg+xml">
    <link rel="apple-touch-icon" href="/icon.png">

    <%# Includes all stylesheet files in app/assets/stylesheets %>
    <%= stylesheet_link_tag :app %>
  </head>

  <body>
    <div class="wrapper">
      <% flash.each do |type, message| %>
        <div class="flash flash-<%= type %>"><%= message %></div>
      <% end %>

      <%= yield %>
    </div>
  </body>
</html>
```

**สิ่งที่เพิ่มเข้ามาจาก layout เปล่าที่ `rails new` สร้างให้ (ทบทวนหน้าตาเดิมจาก
Part 021 Step 205 และ Part 024 Step 233):**

- **`<% flash.each do |type, message| %>`** — วนลูปแสดง `flash` **ทุก key** ที่มีอยู่ (ไม่ใช่
  แค่ `:notice` กับ `:alert` ที่เจาะจงชื่อ) ไว้ที่จุดเดียวใน layout เพื่อให้ทุกหน้าของแอป
  แสดงข้อความ flash ได้โดยอัตโนมัติ โดยไม่ต้องเขียน `<% if flash[:notice] %>...<% end %>` ซ้ำ
  ในทุกไฟล์ view — ทบทวนกลไกของ `flash` จาก **Part 023 Step 226** ที่ค่าจะถูกลบทิ้งอัตโนมัติ
  หลังแสดงผลครั้งเดียว
- **`<div class="wrapper">`** ครอบ `<%= yield %>` ไว้ — จุดที่ Propshaft จะเข้ามาเกี่ยวข้องคือ
  class `wrapper` นี้ต้องมี CSS rule รออยู่ในไฟล์ที่ `stylesheet_link_tag :app` โหลดมา (ดู
  Step 298)

### `app/views/posts/_form.html.erb` — partial ที่ใช้ร่วมกันระหว่าง new และ edit

```erb
<%= form_with model: post, local: true do |f| %>
  <% if post.errors.any? %>
    <div class="form-errors">
      <h3><%= pluralize(post.errors.count, "ข้อผิดพลาด", plural: "ข้อผิดพลาด") %>ทำให้บันทึกบทความนี้ไม่ได้:</h3>
      <ul>
        <% post.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= f.label :title, "ชื่อบทความ" %>
    <%= f.text_field :title %>
  </div>

  <div class="field">
    <%= f.label :body, "เนื้อหา" %>
    <%= f.text_area :body, rows: 8 %>
  </div>

  <div class="actions">
    <%= f.submit "บันทึกบทความ" %>
  </div>
<% end %>
```

**อธิบาย:**

- `form_with model: post` — Rails **เดาเองอัตโนมัติ**ว่าถ้า `post` เป็น record ใหม่ (ยังไม่มี
  id) ให้ submit แบบ `POST` ไปที่ `posts_path`, แต่ถ้าเป็น record ที่มีอยู่แล้ว (มี id) ให้
  submit แบบ `PATCH` ไปที่ `post_path(post)` แทน — นี่คือเหตุผลที่ partial ตัวเดียวใช้ได้กับ
  ทั้งหน้า `new` และ `edit` โดยไม่ต้องเขียนแยก 2 ไฟล์ (ทบทวนกลไกนี้จาก **Part 024**)
- **`local: true`** — สำคัญมากในโปรเจกต์นี้เพราะ**ไม่มี** `turbo-rails` (ตามที่ตั้งค่าไว้ตั้งแต่
  Step 291 ด้วย `--minimal`) ถ้าไม่ใส่ `local: true` ฟอร์มจะพยายาม submit แบบ Turbo/AJAX โดย
  default ซึ่งจะไม่ทำงานถูกต้องเมื่อไม่มี JavaScript คอยดักจับ event — `local: true` บังคับให้
  ฟอร์มเป็น HTML form ธรรมดา submit แบบ full-page reload เสมอ รับประกันว่าทำงานได้แน่นอนไม่ว่า
  เบราว์เซอร์จะปิด JavaScript หรือไม่ก็ตาม
- `pluralize(post.errors.count, "ข้อผิดพลาด", plural: "ข้อผิดพลาด")` — ทบทวน gotcha ของ
  `pluralize` กับคำนามภาษาไทยจาก **Part 024**: คำนามไทยไม่ผัน ต้องระบุ `plural:` ให้เป็นคำ
  เดียวกับ singular เสมอ มิฉะนั้นจะได้ "1 ข้อผิดพลาดs" ที่ผิดหลักภาษา
- `post.errors.each do |error|` แล้วใช้ `error.full_message` — รูปแบบมาตรฐานของ Rails 6.1+
  สำหรับวนลูปแสดง error ทีละข้อความแบบเต็ม (เทียบเท่ากับการวนลูป
  `post.errors.full_messages` แต่ `errors.each` ให้ object `error` ที่เข้าถึงรายละเอียดเพิ่ม
  ได้ เช่น `error.attribute`, `error.type` ถ้าต้องการ custom formatting)

### `app/views/posts/index.html.erb`

```erb
<% content_for(:title, "Mini Blog — บทความทั้งหมด") %>

<header class="page-header">
  <h1>Mini Blog</h1>
  <%= link_to "+ เขียนบทความใหม่", new_post_path, class: "btn btn-primary" %>
</header>

<% if @posts.any? %>
  <div class="post-list">
    <%= render partial: "post", collection: @posts, as: :post %>
  </div>
<% else %>
  <p class="empty-state">ยังไม่มีบทความในระบบ — ลองเขียนบทความแรกของคุณดูสิ!</p>
<% end %>
```

`render partial: "post", collection: @posts, as: :post` คือรูปแบบที่แนะนำที่สุดสำหรับ render
collection ตามที่เรียนใน **Part 024 Step 236** — เขียนแบบเต็มชัดเจนกว่ารูปย่อ
`render @posts` และเลือก `as: :post` เพื่อตั้งชื่อ local variable ในไฟล์ partial ให้อ่านง่าย
(แทนที่จะใช้ชื่อ default ที่มาจากชื่อ partial อัตโนมัติ)

### `app/views/posts/_post.html.erb` — partial ของบทความหนึ่งอันในหน้ารายการ

```erb
<article class="post-card">
  <h2><%= link_to post.title, post_path(post) %></h2>
  <p class="post-meta">
    เขียนเมื่อ <%= time_ago_in_words(post.created_at) %>ที่แล้ว ·
    <%= pluralize(post.comments.count, "ความคิดเห็น", plural: "ความคิดเห็น") %>
  </p>
  <p class="post-excerpt"><%= truncate(post.body, length: 160, separator: " ") %></p>
</article>
```

(รายละเอียดการใช้ `time_ago_in_words`, `truncate`, `pluralize` ในไฟล์นี้อธิบายเต็มรูปแบบใน
Step 298 — ที่นี่ขอโฟกัสที่โครงสร้าง partial ก่อน)

### `app/views/posts/show.html.erb`

```erb
<% content_for(:title, "#{@post.title} — Mini Blog") %>

<%= link_to "« กลับไปหน้ารายการบทความ", posts_path %>

<article class="post-full">
  <h1><%= @post.title %></h1>
  <p class="post-meta">
    เขียนเมื่อ <%= time_ago_in_words(@post.created_at) %>ที่แล้ว
    (<%= @post.created_at.strftime("%d %b %Y เวลา %H:%M น.") %>)
  </p>

  <div class="post-body"><%= simple_format(@post.body) %></div>

  <div class="post-actions">
    <%= link_to "แก้ไขบทความ", edit_post_path(@post), class: "btn" %>
    <%= button_to "ลบบทความ", post_path(@post), method: :delete, class: "btn btn-danger" %>
  </div>
</article>

<section class="comments-section">
  <h2><%= pluralize(@post.comments.count, "ความคิดเห็น", plural: "ความคิดเห็น") %></h2>

  <% if @post.comments.any? %>
    <%= render partial: "comments/comment", collection: @post.comments.order(:created_at), as: :comment %>
  <% else %>
    <p class="empty-state">ยังไม่มีความคิดเห็น เป็นคนแรกที่แสดงความคิดเห็นสิ!</p>
  <% end %>

  <h3>แสดงความคิดเห็น</h3>
  <%= form_with model: @comment, url: post_comments_path(@post), local: true do |f| %>
    <% if @comment.errors.any? %>
      <div class="form-errors">
        <ul>
          <% @comment.errors.each do |error| %>
            <li><%= error.full_message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div class="field">
      <%= f.label :commenter, "ชื่อผู้แสดงความคิดเห็น" %>
      <%= f.text_field :commenter %>
    </div>

    <div class="field">
      <%= f.label :body, "ข้อความ" %>
      <%= f.text_area :body, rows: 3 %>
    </div>

    <div class="actions">
      <%= f.submit "ส่งความคิดเห็น" %>
    </div>
  <% end %>
</section>
```

**อธิบายจุดที่น่าสนใจ:**

- **`button_to "ลบบทความ", post_path(@post), method: :delete`** — ทบทวนจาก Step 291:
  โปรเจกต์นี้ไม่มี Turbo จึงใช้ `button_to` แทน `link_to ... method: :delete` เสมอ `button_to`
  render ออกมาเป็น `<form method="post">` ที่มี hidden field `_method=delete` ซ่อนอยู่ข้างใน
  (ทบทวนกลไก `_method` hidden field จาก **Part 022 Step 211**) ทำงานได้แน่นอน 100% โดยไม่ต้อง
  พึ่ง JavaScript เลย
- **`form_with model: @comment, url: post_comments_path(@post)`** — ต่างจากฟอร์มบทความที่ปล่อย
  ให้ Rails เดา URL เอง ฟอร์ม comment ต้องระบุ `url:` เอง**เสมอ** เพราะ `@comment` เป็น record
  ใหม่ (`Comment.new` จาก controller) ถ้าไม่ระบุ `url:` Rails จะพยายามเดาไปที่
  `comments_path` เฉยๆ (ไม่มี `post_id`) ซึ่งไม่ตรงกับ route ที่ประกาศไว้ใน Step 294 (ที่
  ต้องการ `/posts/:post_id/comments`) — นี่คือ pattern มาตรฐานเวลาสร้างฟอร์มสำหรับ nested
  resource เสมอ
- **`@post.comments.order(:created_at)`** — เรียง comment จากเก่าไปใหม่ (อ่านเป็นลำดับการ
  สนทนาธรรมชาติ) ต่างจากหน้า `index` ของ post ที่เรียงใหม่ไปเก่า (`Post.recent`) — เป็นตัวอย่าง
  ที่ดีว่าการเรียงลำดับที่ "ถูกต้อง" ขึ้นอยู่กับบริบทของหน้าจอ ไม่ใช่กฎตายตัวเดียวทั้งแอป
- **`simple_format(@post.body)`** — helper ของ Rails ที่แปลงข้อความธรรมดา (ที่มีแค่ขึ้นบรรทัด
  ใหม่ `\n\n` คั่นย่อหน้า) ให้กลายเป็น HTML ที่มี `<p>` ครอบแต่ละย่อหน้าอัตโนมัติ พร้อม escape
  HTML พิเศษให้ปลอดภัยจาก XSS ในตัว (ทบทวนหลักการ auto-escape ของ ERB จาก **Part 024**) —
  สะดวกกว่าการเก็บเนื้อหาเป็น HTML ดิบไว้ในฐานข้อมูลตรงๆ ซึ่งเสี่ยงต่อการโดน inject โค้ดอันตราย

### `app/views/posts/new.html.erb` และ `edit.html.erb`

```erb
<%# app/views/posts/new.html.erb %>
<% content_for(:title, "เขียนบทความใหม่ — Mini Blog") %>

<h1>เขียนบทความใหม่</h1>

<%= render "form", post: @post %>

<%= link_to "« กลับไปหน้ารายการบทความ", posts_path %>
```

```erb
<%# app/views/posts/edit.html.erb %>
<% content_for(:title, "แก้ไขบทความ — Mini Blog") %>

<h1>แก้ไขบทความ</h1>

<%= render "form", post: @post %>

<%= link_to "« กลับไปหน้าบทความ", post_path(@post) %>
```

ทั้งสองไฟล์เหมือนกันเกือบทุกประการ ต่างกันแค่หัวข้อและลิงก์ย้อนกลับ — ส่วนฟอร์มจริงทั้งหมดอยู่
ใน `_form.html.erb` ไฟล์เดียว (Step ก่อนหน้า) พิสูจน์คุณค่าของการแยก partial ตามหลัก **DRY**
ที่เรียนมาตั้งแต่ **Part 001**: ถ้าวันหนึ่งต้องเพิ่มฟิลด์ใหม่ให้ `Post` (เช่นเพิ่ม field
`published` ใน Step 300 แบบฝึกหัดเพิ่มเติม) แก้ที่ `_form.html.erb` ไฟล์เดียว ทั้งหน้า new และ
edit จะได้ฟิลด์ใหม่พร้อมกันทันที

### `app/views/comments/_comment.html.erb`

```erb
<div class="comment-card">
  <p class="comment-meta">
    <strong><%= comment.commenter %></strong> ·
    <%= time_ago_in_words(comment.created_at) %>ที่แล้ว
  </p>
  <p class="comment-body"><%= comment.body %></p>
  <%= button_to "ลบ", comment_path(comment), method: :delete, class: "btn btn-small btn-danger" %>
</div>
```

สังเกตว่า partial นี้อยู่ที่ `app/views/comments/_comment.html.erb` (โฟลเดอร์ `comments/`
ไม่ใช่ `posts/`) แต่ถูกเรียกใช้จาก `posts/show.html.erb` ผ่าน `render partial:
"comments/comment", ...` — Rails ยอมให้ partial อยู่คนละโฟลเดอร์กับ view ที่เรียกใช้ได้เสมอ
ตราบใดที่ระบุ path เต็มเป็น string (ทบทวนเรื่องนี้จาก **Part 024**) การแยกไว้ที่
`comments/` (ตรงกับชื่อ resource ของมันเอง) ทำให้โครงสร้างไฟล์สื่อความหมายชัดเจนว่า partial นี้
"เป็นของ" concept ไหน แม้จะไม่มี `CommentsController#show`/`index` ที่ render มันโดยตรงเลยก็ตาม

---

## Step 298: Stylesheet ผ่าน Propshaft และการใช้ view helper อย่างมีความหมาย

ทบทวนจาก **Part 029**: Rails 8 ใช้ **Propshaft** เป็น asset pipeline หน้าที่หลักของมันคือหา
ไฟล์ asset (CSS/JS/รูปภาพ) แล้วเสิร์ฟออกไปพร้อม fingerprint (hash ต่อท้ายชื่อไฟล์) เพื่อแก้ปัญหา
browser cache เก่าค้าง โดยไม่ต้อง compile/preprocess อะไรซับซ้อนเหมือน Sprockets ยุคก่อน — ต่าง
จากเดโมเล็กๆ ใน Part 029 คราวนี้เราจะเขียน stylesheet เต็มรูปแบบสำหรับทั้งแอปจริง

### `app/assets/stylesheets/application.css`

```css
/*
 * This is a manifest file that'll be compiled into application.css.
 *
 * With Propshaft, assets are served efficiently without preprocessing steps. You can still include
 * application-wide styles in this file, but keep in mind that CSS precedence will follow the standard
 * cascading order, meaning styles declared later in the document or manifest will override earlier ones,
 * depending on specificity.
 *
 * Consider organizing styles into separate files for maintainability.
 */

body {
  font-family: "Segoe UI", "Sarabun", sans-serif;
  background: #f5f6f8;
  color: #222;
  margin: 0;
  padding: 0;
}

.wrapper {
  max-width: 720px;
  margin: 0 auto;
  padding: 24px 16px 64px;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.page-header h1 {
  margin: 0;
}

.flash {
  padding: 12px 16px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.flash-notice {
  background: #e3f7e9;
  color: #1b6b3c;
  border: 1px solid #b7e6c6;
}

.flash-alert {
  background: #fdecec;
  color: #a3231f;
  border: 1px solid #f5c2c0;
}

.post-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.post-card,
.post-full,
.comment-card {
  background: #fff;
  border: 1px solid #e2e4e8;
  border-radius: 8px;
  padding: 16px 20px;
}

.post-card h2 {
  margin: 0 0 4px;
}

.post-card h2 a {
  color: #1a3d7c;
  text-decoration: none;
}

.post-meta,
.comment-meta {
  color: #6b7280;
  font-size: 0.85rem;
  margin: 4px 0 8px;
}

.post-excerpt {
  margin: 0;
  line-height: 1.5;
}

.post-body {
  line-height: 1.7;
  margin: 16px 0;
}

.post-actions {
  display: flex;
  gap: 8px;
  margin-top: 16px;
}

.comments-section {
  margin-top: 32px;
}

.comment-card {
  margin-bottom: 12px;
}

.comment-body {
  margin: 4px 0 8px;
}

.empty-state {
  color: #6b7280;
  font-style: italic;
}

.field {
  margin-bottom: 16px;
}

.field label {
  display: block;
  font-weight: 600;
  margin-bottom: 4px;
}

.field input[type="text"],
.field textarea {
  width: 100%;
  box-sizing: border-box;
  padding: 8px 10px;
  border: 1px solid #cbd2d9;
  border-radius: 6px;
  font-size: 1rem;
  font-family: inherit;
}

.form-errors {
  background: #fdecec;
  border: 1px solid #f5c2c0;
  color: #a3231f;
  padding: 12px 16px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.form-errors h3 {
  margin: 0 0 8px;
}

.btn,
input[type="submit"] {
  display: inline-block;
  padding: 8px 16px;
  border-radius: 6px;
  border: 1px solid #cbd2d9;
  background: #fff;
  color: #222;
  cursor: pointer;
  text-decoration: none;
  font-size: 0.95rem;
}

.btn-primary,
input[type="submit"] {
  background: #1a3d7c;
  border-color: #1a3d7c;
  color: #fff;
}

.btn-danger {
  background: #fff;
  border-color: #d9534f;
  color: #d9534f;
}

.btn-small {
  padding: 4px 10px;
  font-size: 0.8rem;
}
```

ทดสอบว่า Propshaft เสิร์ฟไฟล์นี้ถูกต้องจริงด้วย `curl`:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:3000/assets/application.css
```

```
HTTP 200
```

`stylesheet_link_tag :app` ใน layout (Step 297) เรียก Propshaft ให้หาไฟล์ที่ชื่อขึ้นต้นด้วย
`application` ในโฟลเดอร์ `app/assets/stylesheets/` แล้วสร้าง `<link>` tag ที่ชี้ไปยัง URL ที่มี
fingerprint ต่อท้าย (เช่น `/assets/application-a1b2c3....css`) ให้อัตโนมัติในโหมด production
ส่วนในโหมด development (ที่เรากำลังทดสอบอยู่) Propshaft จะเสิร์ฟไฟล์แบบไม่ fingerprint เพื่อให้
เห็นผลการแก้ CSS ทันทีโดยไม่ต้อง clear cache — พฤติกรรมนี้อธิบายไว้ละเอียดแล้วใน **Part 029**

### การใช้ view helper อย่างมีความหมาย — ไม่ใช่แค่โชว์ syntax

โปรเจกต์นี้ใช้ helper ที่เรียนมาจาก **Part 024 Step 237–238** ใน 3 จุดที่มีบทบาทจริงต่อ UX ของ
แอป ไม่ใช่แค่ตัวอย่างลอยๆ:

**1. `time_ago_in_words` — บอกเวลาแบบสัมพัทธ์แทนตัวเลขดิบ**

```erb
เขียนเมื่อ <%= time_ago_in_words(post.created_at) %>ที่แล้ว
```

ในหน้า index และ show ทุกจุดที่แสดงเวลาที่โพสต์/คอมเมนต์ถูกสร้าง ใช้ `time_ago_in_words` แทน
`post.created_at.to_s` ตรงๆ เพราะผู้อ่านบล็อกทั่วไปสนใจ **"นานแค่ไหนแล้ว"** มากกว่า
**"เวลาที่แน่นอน"** ("2 นาทีที่แล้ว" มีความหมายกับผู้อ่านมากกว่า "2026-09-26 02:36:30 UTC")
ผลลัพธ์จริงที่ทดสอบแล้ว:

```irb
irb> helper.time_ago_in_words(5.minutes.ago)
=> "5 minutes"
irb> helper.time_ago_in_words(2.days.ago)
=> "2 days"
```

(หน้า `show` เพิ่ม `strftime` แสดงเวลาที่แน่นอนไว้ในวงเล็บถัดจากมันด้วย เผื่อผู้อ่านต้องการรู้
เวลาที่แน่ชัดจริงๆ — ผสมทั้งสองแบบไว้ด้วยกันเป็น pattern ที่เว็บข่าว/บล็อกจริงใช้กันทั่วไป)

**2. `truncate` — ตัดตัวอย่างเนื้อหาในหน้ารายการให้พอดี**

```erb
<%= truncate(post.body, length: 160, separator: " ") %>
```

หน้า `index` แสดงบทความหลายอันพร้อมกัน ถ้าแสดงเนื้อหาเต็มทุกอันจะทำให้หน้าเว็บยาวเกินไปและอ่าน
ยาก `truncate(length: 160, separator: " ")` ตัดข้อความไม่ให้เกิน 160 ตัวอักษร โดยตัดที่ช่องว่าง
คำล่าสุดเสมอ (ไม่ตัดกลางคำ) แล้วเติม `...` ต่อท้ายให้อัตโนมัติ ทำให้ผู้อ่านเห็น "ตัวอย่าง" ของ
บทความก่อนตัดสินใจว่าจะกด "อ่านต่อ" (คลิกลิงก์ไปหน้า `show`) หรือไม่

**3. `pluralize` — นับจำนวนความคิดเห็นให้ถูกไวยากรณ์**

```erb
<%= pluralize(post.comments.count, "ความคิดเห็น", plural: "ความคิดเห็น") %>
```

ปรากฏทั้งในหน้า `index` (สรุปย่อใต้ชื่อบทความ) และหน้า `show` (หัวข้อ section ความคิดเห็น) —
ทบทวน gotcha ภาษาไทยจาก **Part 024**: ต้องระบุ `plural:` ให้เป็นคำเดียวกับ singular เสมอ
เพราะคำนามไทยไม่ผันตามจำนวน ผลลัพธ์ที่ได้คือ `"0 ความคิดเห็น"`, `"1 ความคิดเห็น"`,
`"5 ความคิดเห็น"` — ถูกต้องทุกจำนวนโดยไม่มีคำว่า "s" แปลกปลอมติดมาแบบที่จะเกิดขึ้นถ้าลืมใส่
`plural:`

> **ทดสอบจริงยืนยันพฤติกรรมทั้ง 3 helper พร้อมกันในหน้าเดียว:** เปิดหน้า `/posts/1` (ที่มี
> comment 2 อันจาก seed data ใน Step 299) จะเห็นข้อความ `"เขียนเมื่อ X นาทีที่แล้ว"`,
> เนื้อหาเต็มที่ไม่ถูกตัด (เพราะหน้า `show` ไม่ใช้ `truncate`), และหัวข้อ
> `"2 ความคิดเห็น"` พร้อมกันในหน้าเดียว — ครบทั้ง 3 helper ทำงานร่วมกันอย่างมีความหมาย ไม่ใช่
> แค่ตัวอย่างแยกโดดๆ

---

## Step 299: `db/seeds.rb` และทดสอบ user flow เต็มรูปแบบด้วยข้อมูลจริง

### `db/seeds.rb`

```ruby
# frozen_string_literal: true

# db/seeds.rb — สร้างข้อมูลตัวอย่างสำหรับทดลองใช้งาน Mini Blog
# รันด้วยคำสั่ง: bin/rails db:seed

puts "กำลังล้างข้อมูลเก่า..."
Comment.destroy_all
Post.destroy_all

puts "กำลังสร้างบทความตัวอย่าง..."

post1 = Post.create!(
  title: "เริ่มต้นกับ Ruby on Rails",
  body: "Ruby on Rails เป็น Web Application Framework ที่ช่วยให้เราสร้างเว็บแอปพลิเคชันได้อย่าง " \
        "รวดเร็ว ด้วยหลักการ Convention over Configuration และ Don't Repeat Yourself ทำให้ไม่ต้อง " \
        "เขียนโค้ดซ้ำซ้อน และมีโครงสร้างไฟล์ที่เป็นมาตรฐานเดียวกันในทุกโปรเจกต์"
)

post2 = Post.create!(
  title: "MVC คืออะไร",
  body: "MVC (Model-View-Controller) คือรูปแบบการแบ่งความรับผิดชอบของโค้ดออกเป็น 3 ส่วน " \
        "Model จัดการข้อมูลและ business logic, View สร้างหน้าตาที่ผู้ใช้เห็น, " \
        "และ Controller รับ request แล้วประสานงานระหว่าง Model กับ View"
)

post3 = Post.create!(
  title: "ทำไมต้องเขียน Automated Test",
  body: "การเขียน test อัตโนมัติช่วยให้มั่นใจได้ว่าโค้ดที่เขียนไว้ยังทำงานถูกต้องอยู่เสมอ " \
        "แม้จะแก้ไขโค้ดส่วนอื่นในภายหลัง ลดความเสี่ยงที่จะทำฟีเจอร์เดิมพังโดยไม่รู้ตัว " \
        "และช่วยให้กล้า refactor โค้ดได้มากขึ้น"
)

puts "กำลังสร้างความคิดเห็นตัวอย่าง..."

post1.comments.create!(commenter: "สมชาย", body: "บทความดีมากครับ อ่านเข้าใจง่าย")
post1.comments.create!(commenter: "สมหญิง", body: "รอติดตามบทความต่อไปเลยค่ะ")
post2.comments.create!(commenter: "วิชัย", body: "อธิบาย MVC ได้ชัดเจนดีครับ")

puts "เสร็จสิ้น! สร้างบทความทั้งหมด #{Post.count} บทความ, ความคิดเห็นทั้งหมด #{Comment.count} รายการ"
```

**อธิบาย:**

- `Comment.destroy_all` ก่อน `Post.destroy_all` — ต้องลบ comment ก่อนเสมอในทางทฤษฎี (แม้จริงๆ
  `dependent: :destroy` จะจัดการให้ถ้าลบ `Post` ก่อนก็ตาม) การเขียนลำดับที่ถูกต้องด้วยมือทำให้
  โค้ด seed อ่านแล้วเข้าใจ dependency ได้ทันทีโดยไม่ต้องพึ่งพฤติกรรม cascade ที่มองไม่เห็นในโค้ด
- `Post.create!` (มี `!`) แทน `Post.create` — ทบทวนจาก **Part 011/026**: ถ้า validation ไม่
  ผ่าน `create!` จะ **raise `ActiveRecord::RecordInvalid` ทันที** ทำให้ script หยุดทำงานและ
  แสดง error ชัดเจน ต่างจาก `create` ธรรมดาที่จะคืนค่า object ที่ยังไม่ถูกบันทึกเงียบๆ โดยไม่มี
  ใครสังเกตเห็น — สำหรับ seed data (ที่ควรจะถูกต้องเสมอเพราะเราเขียนเอง) การใช้ `!` ช่วยจับบัค
  ได้ทันทีถ้าพิมพ์ข้อมูลผิดจน validation ไม่ผ่าน

รันด้วยคำสั่ง:

```bash
bin/rails db:seed
```

```
กำลังล้างข้อมูลเก่า...
กำลังสร้างบทความตัวอย่าง...
กำลังสร้างความคิดเห็นตัวอย่าง...
เสร็จสิ้น! สร้างบทความทั้งหมด 3 บทความ, ความคิดเห็นทั้งหมด 3 รายการ
```

### ทดสอบ User Flow เต็มรูปแบบด้วย `curl` จริง

ทบทวนจาก **Part 020 Step 200**: การทดสอบด้วยการยิง request จริง (ไม่ใช่แค่เดาว่าโค้ดน่าจะ
ทำงาน) คือทักษะสำคัญ — Part นี้ยังไม่ได้เรียน automated test แบบ Rails (จะเรียนใน **Phase 6**)
จึงทดสอบด้วยมือผ่าน `curl` ให้ครบทุกขั้นตอนของ user flow จริง โดยรัน server ก่อน:

```bash
bin/rails server -p 3000
```

**ขั้นที่ 1 — เปิดหน้ารายการบทความ**

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:3000/posts
```

```
HTTP 200
```

**ขั้นที่ 2 — สร้างบทความใหม่ (ต้องดึง CSRF token จากหน้าฟอร์มก่อนเสมอ)**

ทบทวนจาก **Part 023**: Rails ป้องกัน CSRF (Cross-Site Request Forgery) ให้อัตโนมัติทุก
`POST`/`PATCH`/`PUT`/`DELETE` request ต้องแนบ token ที่ตรงกับ session ปัจจุบันมาด้วยเสมอ
เวลาทดสอบด้วย `curl` (ที่ไม่ใช่เบราว์เซอร์จริงที่จัดการเรื่องนี้ให้อัตโนมัติ) ต้องดึง token
จาก `<meta name="csrf-token">` ในหน้า HTML ก่อนเอง:

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/new -o new.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบจาก curl" \
  --data-urlencode "post[body]=นี่คือเนื้อหาทดสอบการสร้างบทความผ่าน curl เพื่อยืนยันว่า create ทำงานถูกต้อง" \
  -D - -o /dev/null -w "HTTP %{http_code}\n" | grep -E "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/posts/4
HTTP 302
```

ได้ **302 Found** พร้อม `location: /posts/4` — บทความถูกสร้างสำเร็จเป็น id 4 (ต่อจาก 3
บทความที่มาจาก seed) ตรงตามรูปแบบ Post/Redirect/Get ที่ออกแบบไว้ใน `PostsController#create`

**ขั้นที่ 3 — เปิดดูบทความที่เพิ่งสร้าง**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 -o show4.html \
  -w "HTTP %{http_code}\n"
grep -o 'บทความทดสอบจาก curl' show4.html
```

```
HTTP 200
บทความทดสอบจาก curl
```

**ขั้นที่ 4 — เพิ่มความคิดเห็นในบทความนี้**

```bash
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' show4.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts/4/comments \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "comment[commenter]=ผู้ทดสอบ" \
  --data-urlencode "comment[body]=ความคิดเห็นทดสอบจาก curl" \
  -D - -o /dev/null -w "HTTP %{http_code}\n" | grep -E "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/posts/4
HTTP 302
```

ตรวจสอบว่าความคิดเห็นปรากฏจริงในหน้า post (comment ได้ id 4 ต่อจาก 3 comment ของ seed):

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 -o show4b.html
grep -o 'ความคิดเห็นทดสอบจาก curl' show4b.html
grep -oE '/comments/[0-9]+' show4b.html | sort -u
```

```
ความคิดเห็นทดสอบจาก curl
/comments/4
```

**ขั้นที่ 5 — แก้ไขบทความ**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4/edit -o edit4.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' edit4.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X PATCH http://localhost:3000/posts/4 \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบจาก curl (แก้ไขแล้ว)" \
  --data-urlencode "post[body]=นี่คือเนื้อหาทดสอบการสร้างบทความผ่าน curl เพื่อยืนยันว่า create ทำงานถูกต้อง (อัปเดตแล้ว)" \
  -w "HTTP %{http_code}\n" -o /dev/null

curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 | \
  grep -o 'บทความทดสอบจาก curl (แก้ไขแล้ว)'
```

```
HTTP 302
บทความทดสอบจาก curl (แก้ไขแล้ว)
```

ชื่อบทความเปลี่ยนเป็นเวอร์ชันที่แก้ไขแล้วจริง — `update` action ทำงานถูกต้อง

**ขั้นที่ 6 — ลบความคิดเห็น**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 -o show4c.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' show4c.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X DELETE http://localhost:3000/comments/4 \
  -H "X-CSRF-Token: $TOKEN" \
  -D - -o /dev/null -w "HTTP %{http_code}\n" | grep -E "HTTP|location"

curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 | \
  grep -c 'ความคิดเห็นทดสอบจาก curl'
```

```
HTTP/1.1 303 See Other
location: http://localhost:3000/posts/4
HTTP 303
0
```

`303 See Other` ตามที่ตั้งค่า `status: :see_other` ไว้ใน `CommentsController#destroy` และผล
`grep -c` เป็น `0` ยืนยันว่าความคิดเห็นหายไปจากหน้าจริง

**ขั้นที่ 7 — ลบบทความ**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/4 -o show4d.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' show4d.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X DELETE http://localhost:3000/posts/4 \
  -H "X-CSRF-Token: $TOKEN" \
  -D - -o /dev/null -w "HTTP %{http_code}\n" | grep -E "HTTP|location"

curl -s -o /dev/null -w "confirm gone: HTTP %{http_code}\n" http://localhost:3000/posts/4
curl -s http://localhost:3000/posts | grep -c 'บทความทดสอบจาก curl'
```

```
HTTP/1.1 303 See Other
location: http://localhost:3000/posts
HTTP 303
confirm gone: HTTP 404
0
```

บทความถูกลบสำเร็จ, redirect กลับไปหน้ารายการ, เปิด `/posts/4` ซ้ำได้ **HTTP 404** (เพราะ
`Post.find(params[:id])` raise `ActiveRecord::RecordNotFound` เมื่อไม่พบ record — Rails แปลง
exception นี้เป็น response 404 ให้อัตโนมัติในโหมด production/ที่ทดสอบ) และ `grep -c` ในหน้า
`/posts` เป็น `0` ยืนยันว่าไม่มีร่องรอยของบทความที่ลบไปเหลืออยู่เลย

### สรุป user flow ที่ทดสอบครบทั้งหมด

| ขั้นตอน | Action ที่ทำงาน | HTTP Status ที่ได้จริง |
|---|---|---|
| สร้างบทความ | `PostsController#create` | 302 → redirect ไป `show` |
| ดูบทความ | `PostsController#show` | 200 |
| เพิ่มความคิดเห็น | `CommentsController#create` | 302 → redirect ไป `show` |
| แก้ไขบทความ | `PostsController#update` | 302 → redirect ไป `show` |
| ลบความคิดเห็น | `CommentsController#destroy` | 303 → redirect ไป `show` |
| ลบบทความ | `PostsController#destroy` | 303 → redirect ไป `index` |
| เปิดบทความที่ถูกลบแล้ว | `Post.find` raise `RecordNotFound` | 404 |
| สร้างบทความด้วยข้อมูลไม่ถูกต้อง | `PostsController#create` (validation fail) | 422 |

ทุกแถวในตารางนี้คือผลลัพธ์ที่ทดสอบรันจริงระหว่างเขียนบทเรียนนี้ ไม่ใช่ค่าที่คาดเดาไว้ล่วงหน้า
— **โปรเจกต์ Mini Blog ทำงานได้ครบวงจรจริงตั้งแต่ต้นจนจบ**

---

## Step 300: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 3

มาถึง Step สุดท้ายของเฟส 3 ขอใช้ Step นี้ทำสิ่งที่วิศวกรมืออาชีพทำเป็นประจำ — **code review**
คือการเดินย้อนกลับไปดูโค้ดที่เขียนเสร็จแล้วทั้งหมด แล้วถามตัวเองว่า "ทำไมโค้ดตรงนี้ถึงเขียน
แบบนี้" และ "อะไรที่ยังขาดอยู่"

### เดินย้อนดูว่าแต่ละไฟล์สาธิต Part ไหนบ้าง

| ไฟล์ / โค้ด | สาธิตอะไร | Part ที่สอน |
|---|---|---|
| `rails new mini_blog --minimal` | โครงสร้างโฟลเดอร์มาตรฐาน, Convention over Configuration | Part 021 |
| `config/routes.rb` (`resources`, `shallow: true`) | RESTful routing, nested resources | Part 022 |
| `before_action`, `params`, `flash`/`flash.now`, `redirect_to`/`render`, strong parameters | Controller layer ทั้งหมด | Part 023 |
| `form_with`, partial (`_form`, `_post`, `_comment`), `render partial:, collection:`, `content_for` | View layer, template reuse | Part 024 |
| `Post`/`Comment` migration, `ApplicationRecord`, `Post.create!` ใน console | ActiveRecord พื้นฐาน, migration, schema | Part 025 |
| `validates :title, presence: true`, `length: { minimum:, maximum: }` | Validation | Part 026 |
| `has_many :comments, dependent: :destroy`, `belongs_to :post`, `@post.comments.build` | Association | Part 027 |
| (แนวคิด) โครงสร้าง controller/view ที่เขียนมือ vs. ที่ scaffold จะ generate ให้ | Scaffold/Generator | Part 028 |
| `app/assets/stylesheets/application.css`, `stylesheet_link_tag :app` | Propshaft/Asset pipeline | Part 029 |
| ทั้งโปรเจกต์รวมกัน | **MVC ทำงานร่วมกันเป็นระบบเดียว** | **Part 030 (Part นี้)** |

สังเกตว่า **Part 028 (Scaffold)** ไม่มีโค้ดปรากฏโดยตรงในโปรเจกต์นี้ — เป็นการตัดสินใจที่ตั้งใจ:
เราเขียน `PostsController`/`CommentsController`/views ทั้งหมด**ด้วยมือ** แทนที่จะรัน `rails
generate scaffold Post title:string body:text` ครั้งเดียวจบ เพื่อให้แน่ใจว่าเข้าใจ**ทุกบรรทัด
ของโค้ดที่เกิดขึ้นจริง** ไม่ใช่แค่รู้ว่า "มีคำสั่งที่สร้างให้อัตโนมัติ" — นี่คือเหตุผลที่
Part 028 สอนเรื่อง "การอ่านโค้ดที่ generate มา" ไม่ใช่แค่ "วิธีรัน scaffold" เพราะทักษะสำคัญ
กว่าคือ**อ่านและเข้าใจโค้ดที่เกิดขึ้น** ไม่ว่าจะพิมพ์เองหรือให้เครื่องมือช่วยสร้าง — ลองรัน
`rails generate scaffold` ในโปรเจกต์ทดลองแยกต่างหากแล้วเทียบโค้ดที่ได้กับโค้ดใน Part นี้ดูเป็น
แบบฝึกหัดที่ดีมาก (จะเห็นว่าโครงสร้างคล้ายกันมาก แต่ scaffold จะสร้าง JSON response ให้ทุก
action ด้วย ซึ่งโปรเจกต์นี้ตัดออกเพราะยังไม่ได้เรียน API ใน Phase 8)

### จุดออกแบบที่ควรพิจารณาอีกครั้ง (แม้จะทำงานถูกต้องแล้ว)

1. **`CommentsController#create` ใช้ `redirect_to` เสมอแม้ validation ไม่ผ่าน** (Step 296) —
   ทำให้ข้อมูลที่ผู้ใช้กรอกในฟอร์ม comment หายไปต้องพิมพ์ใหม่ทั้งหมดเมื่อกรอกผิด ต่างจาก
   `PostsController#create` ที่ `render` กลับพร้อมข้อมูลเดิม — เป็น trade-off ที่ยอมรับได้ใน
   ตอนนี้ (comment สั้น พิมพ์ใหม่ไม่เสียเวลามาก) แต่ถ้าจะทำให้สมบูรณ์แบบกว่านี้ ต้องใช้เทคนิค
   เก็บค่าที่กรอกผิดไว้ใน `flash[:comment_params]` ชั่วคราวแล้วดึงกลับมาแสดงในฟอร์ม — ซับซ้อน
   เกินความจำเป็นสำหรับ Part นี้ จึงเลือกวิธีง่ายกว่าไว้ก่อน
2. **ไม่มีการตรวจสอบว่า `commenter` เป็นตัวตนจริงหรือไม่** — ใครก็ตามพิมพ์ชื่ออะไรก็ได้ในช่อง
   "ชื่อผู้แสดงความคิดเห็น" (แม้แต่ชื่อของคนอื่น) เพราะยังไม่มีระบบ authentication เลย — จุดนี้
   คือสิ่งที่ **Phase 5 (Authentication & Authorization)** จะเข้ามาแก้ไขโดยตรง
3. **ไม่มี rate limiting ใดๆ** — ในทางทฤษฎีมีคนสแปมเพิ่ม comment ได้ไม่จำกัดจำนวนต่อวินาที —
   ในโลกจริงต้องมี mechanism ป้องกัน (เช่น `rack-attack`) ซึ่งจะเรียนใน **Part 060**

### สิ่งที่ยังขาดชัดเจนสำหรับ Production (ตั้งใจปล่อยไว้ให้เฟสถัดไปแก้)

Part นี้ปิด Phase 3 ด้วยแอปที่ "ทำงานได้จริง" แต่ **ยังไม่พร้อมสำหรับ production จริง** อย่าง
น้อย 3 เรื่องใหญ่ต่อไปนี้ ซึ่งเป็นการเตรียมทางไปสู่เฟสถัดๆ ไปโดยตรง:

**1. ไม่มีระบบ Authentication/Authorization เลย**

ตอนนี้**ใครก็ตาม**ที่เข้าเว็บนี้ได้สามารถแก้ไขหรือลบบทความของคนอื่นได้ทั้งหมด (ไม่มีแนวคิด
"เจ้าของบทความ" เลย) — `/posts/1/edit` เปิดได้จากทุกคนไม่มีข้อจำกัด นี่คือช่องโหว่ร้ายแรงถ้าจะ
เอาไปใช้งานจริง **Phase 5 (Part 041–045)** จะสอนตั้งแต่ `has_secure_password` พื้นฐาน ไปจนถึง
Devise, Pundit/CanCanCan สำหรับ authorization เต็มรูปแบบ

**2. ไม่มี Automated Test สักบรรทัดเดียว**

ทุกอย่างที่ "ทดสอบ" ใน Part นี้ (Step 299) เป็นการยิง `curl` ด้วยมือ ซึ่ง:

- ไม่มีทางรันซ้ำอัตโนมัติได้ทุกครั้งที่แก้โค้ด (ต้องพิมพ์คำสั่งเองใหม่ทุกรอบ)
- ไม่ครอบคลุมทุกกรณี edge case (เช่น สร้าง comment ที่ `post_id` ไม่มีอยู่จริง, ส่ง params
  แปลกๆ ที่ไม่ผ่าน strong parameters)
- ไม่มีใครรู้ทันทีว่าการแก้โค้ดในอนาคตทำให้ฟีเจอร์เดิมพังหรือไม่ (ต้องมานั่งไล่ `curl` ใหม่ทุก
  ครั้ง)

ทบทวนจาก **Part 018–020**: เราเคยเขียน Minitest และ RSpec test suite เต็มรูปแบบมาแล้วสำหรับ
Todo CLI — ทักษะเดียวกันนั้นจะนำมาใช้กับ Rails ใน **Phase 6 (Part 046–050)** ที่จะสอน RSpec
สำหรับ Rails โดยเฉพาะ (model spec, request spec, system spec ด้วย Capybara) ตอนนั้นทุกขั้นตอน
ที่เพิ่งทดสอบด้วยมือใน Step 299 จะถูกเขียนเป็น automated test ที่รันซ้ำได้ในไม่กี่วินาที

**3. ไม่มี Pagination**

`Post.recent` ใน `PostsController#index` (Step 295) ดึง**บทความทั้งหมด**มาแสดงในหน้าเดียว
ไม่ว่าจะมี 3 บทความหรือ 30,000 บทความก็ตาม — เมื่อข้อมูลเยอะขึ้นเรื่อยๆ หน้านี้จะโหลดช้าลง
เรื่อยๆ จนในที่สุดจะพังเพราะดึงข้อมูลมากเกินไปในครั้งเดียว **Part 038** จะสอนการทำ pagination
ด้วย gem อย่าง Kaminari/Pagy เพื่อแบ่งแสดงทีละหน้า (เช่น 10 บทความต่อหน้า) แก้ปัญหานี้โดยตรง

> **ทำไมถึงตั้งใจไม่แก้ 3 เรื่องนี้ใน Part นี้:** เพราะแต่ละเรื่องมีความลึกพอที่จะเป็นเนื้อหา
> ทั้ง Part หรือทั้ง Phase (authentication ทั้ง Phase 5, testing ทั้ง Phase 6, pagination เป็น
> ส่วนหนึ่งของ Part 038) การพยายามยัดทุกอย่างไว้ใน capstone ตัวเดียวจะทำให้ Part นี้ยาวเกินไป
> และสอนแต่ละเรื่องแบบผิวเผินเกินไป — หลักการของหลักสูตรนี้คือ**สอนให้ลึกทีละเรื่อง** ดีกว่า
> สอนกว้างแต่ตื้นทุกเรื่องพร้อมกัน

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มระบบ tag ให้บทความ** — ให้ผู้ใช้ใส่ tag คั่นด้วยจุลภาค (เช่น "ruby, rails,
   tutorial") ตอนสร้าง/แก้ไขบทความ แล้วแสดง tag เป็นป้าย (badge) ในหน้า index/show พร้อมทำ
   หน้ากรองบทความตาม tag เดียว (`GET /posts?tag=ruby`) — ระดับง่ายสุดใช้ column `tags` เป็น
   `string` ธรรมดาแล้ว `split(",")` เอง (ยังไม่ต้องสร้างตาราง `Tag` แยก เดี๋ยว
   `has_many :through` จะเรียนใน **Part 033**)
2. **เพิ่มสถานะ published/draft ให้บทความ** — เพิ่ม column `published` (boolean, default
   `false`) ให้ `Post` แล้วแก้ `PostsController#index` ให้แสดงเฉพาะบทความที่ `published:
   true` เท่านั้น (ใช้ `scope :published, -> { where(published: true) }`) พร้อมเพิ่มปุ่ม
   "เผยแพร่"/"เก็บเป็นฉบับร่าง" ในหน้า `show` (ใบ้: ทบทวนเทคนิค custom action บน resource ด้วย
   `member do patch :publish end` จาก **Part 022 Step 215**)
3. **เพิ่มกล่องค้นหาบทความ** — เพิ่มฟอร์มค้นหาในหน้า `index` ที่ค้นหาจากชื่อบทความ (`title`)
   ด้วย SQL `LIKE` (เช่น `Post.where("title LIKE ?", "%#{params[:q]}%")`) แสดงผลลัพธ์พร้อม
   ข้อความ "พบ N บทความที่ตรงกับคำค้นหา" (ใช้ `pluralize` ที่เรียนใน Step 298) — ระวังเรื่อง
   SQL injection ด้วยการใช้ `?` placeholder เสมอ **ห้าม** interpolate ค่าจาก `params` เข้าไปใน
   string ของ query ตรงๆ เด็ดขาด (จะเจาะลึกเรื่อง query interface และความปลอดภัยเต็มรูปแบบใน
   **Part 034** และ **Phase 13**)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- วางแผนโดเมนของแอปพลิเคชัน (Post `has_many` Comment, Comment `belongs_to` Post) ก่อนลงมือ
  เขียนโค้ด และออกแบบขอบเขตโปรเจกต์ให้ "เล็กแต่จบได้จริง" แทนที่จะพยายามทำทุกฟีเจอร์พร้อมกัน
- เขียน Migration ที่มี `null: false` และ `foreign_key: true` เพื่อป้องกันข้อมูลผิดพลาดที่
  ระดับฐานข้อมูล ควบคู่กับ `validates` ที่ระดับ Model (ป้องกันสองชั้น)
- ประกาศ Association (`has_many ... dependent: :destroy`, `belongs_to`) และเห็นผลจริงว่า
  `dependent: :destroy` ป้องกัน orphaned record ได้อย่างไร
- เขียน nested routes แบบ `shallow: true` ให้ URL ของ comment สั้นกระชับในจุดที่ไม่จำเป็นต้อง
  รู้ post แต่ยังคง URL ที่มี `post_id` ไว้ในจุดที่จำเป็น (`create`)
- เขียน Controller ครบ CRUD พร้อม strong parameters, `before_action`, และรูปแบบ flash/redirect
  ที่ถูกต้องตามหลัก Post/Redirect/Get รวมถึงเข้าใจว่าทำไม redirect หลัง `DELETE` ควรใช้
  `status: :see_other`
- เขียน View ที่แยก partial ตามหลัก DRY (`_form` ใช้ร่วมกันระหว่าง new/edit, `_post`/
  `_comment` สำหรับ `render partial:, collection:`) และใช้ `form_with ... local: true` ใน
  โปรเจกต์ที่ไม่มี Turbo
- ใช้ view helper (`time_ago_in_words`, `truncate`, `pluralize`, `simple_format`) อย่างมี
  เป้าหมายจริงเพื่อปรับปรุง UX ไม่ใช่แค่โชว์ syntax
- เขียน `db/seeds.rb` ที่ใช้ `create!` เพื่อจับ error ทันทีถ้าข้อมูลไม่ถูกต้อง
- ทดสอบ user flow เต็มรูปแบบด้วย `curl` จริง (สร้าง → ดู → คอมเมนต์ → แก้ไข → ลบคอมเมนต์ → ลบ
  บทความ) และยืนยัน HTTP status code ที่ถูกต้องในทุกขั้นตอน
- ทำ code review ย้อนหลังทั้งโปรเจกต์ ระบุได้ว่าแต่ละส่วนของโค้ดสาธิตหลักการจาก Part ไหนบ้าง
  และวิเคราะห์ได้ว่าโปรเจกต์ยังขาดอะไรสำหรับ production จริง (authentication, automated test,
  pagination)

## สรุปภาพรวม Phase 3: Rails Fundamentals

ยินดีด้วย! ตอนนี้ **Phase 3: Rails Fundamentals — MVC (Part 021–030, Step 201–300)** เสร็จ
สมบูรณ์แล้ว เราเดินทางจากการติดตั้ง Rails และทำความรู้จักโครงสร้างโฟลเดอร์ (Part 021), Routing
เต็มรูปแบบตั้งแต่ `resources` พื้นฐานไปจนถึง nested resources และ constraints (Part 022),
Controller layer ครบทุกกลไก (`params`, `session`, `flash`, `before_action`, `rescue_from`)
(Part 023), View layer ทั้งระบบ (`layout`, partial, `render collection:`, `content_for`, view
helper) (Part 024), ActiveRecord และ Migration พื้นฐาน (Part 025), Validation และ Callback
(Part 026), Association สามแบบหลัก (Part 027), Scaffold และการอ่านโค้ดที่ generator สร้างให้
(Part 028), Asset Pipeline ด้วย Propshaft (Part 029) จนมาถึงโปรเจกต์รวบยอด Mini Blog ใน Part
นี้ที่ประกอบทุกชิ้นส่วนเข้าด้วยกันเป็นเว็บแอปพลิเคชันที่ใช้งานได้จริงตั้งแต่ต้นจนจบ

สิ่งที่สำคัญที่สุดที่ควรพาติดตัวไปจากเฟสนี้ไม่ใช่แค่ "รู้จัก syntax ของ Rails" แต่คือ
**ความเข้าใจว่า MVC ทำงานร่วมกันเป็นวงจรอย่างไรจริงๆ**: Router ตัดสินใจปลายทางจาก URL + HTTP
verb → Controller ดึง/สร้าง/แก้ไข/ลบข้อมูลผ่าน Model (ที่ validate ตัวเองก่อนบันทึกเสมอ) →
View render HTML กลับไปหาผู้ใช้ → และทั้งหมดนี้เกิดขึ้นซ้ำในทุก request ที่เข้ามา วงจรนี้คือ
หัวใจของเว็บแอปพลิเคชัน Rails **ทุกตัว** ไม่ว่าจะเล็กแค่ Mini Blog นี้ หรือใหญ่ระดับ GitHub/
Shopify ก็ตาม — สิ่งที่ต่างกันมีแค่**ความซับซ้อนของแต่ละชั้น**เท่านั้นเอง ไม่ใช่สถาปัตยกรรมพื้น
ฐานที่เปลี่ยนไป

Phase 3 ยังทิ้งช่องว่างไว้อย่างตั้งใจ 3 เรื่องใหญ่ (authentication, automated test,
pagination/performance) ตามที่วิเคราะห์ไว้ใน Step 300 — ช่องว่างเหล่านี้ไม่ใช่ความผิดพลาด แต่
เป็น**แผนที่ที่ชี้ทางไปสู่เฟสถัดไปของหลักสูตรอย่างชัดเจน**

**ต่อไป (Part 031 — เปิด Phase 4: CRUD, Forms, ActiveRecord ขั้นสูง):** เราจะกลับมาเจาะลึก
สิ่งที่ Part นี้ใช้แบบผิวเผินให้ลึกซึ้งยิ่งขึ้น เริ่มจาก **`form_with` เชิงลึก** (custom URL,
`fields_for`, การจัดการ file upload เบื้องต้น), **strong parameters ขั้นสูง** (nested
parameters, arrays), และ **nested attributes** (`accepts_nested_attributes_for`) ที่จะทำให้
สร้างบทความพร้อมความคิดเห็นแรกได้ในฟอร์มเดียว — เทคนิคที่ปูทางไปสู่ฟอร์มที่ซับซ้อนขึ้นเรื่อยๆ
ตลอดเฟส 4 จนถึงโปรเจกต์รวบยอดถัดไป: **ระบบจัดการร้านค้าเล็ก (Product, Category, Order)** ใน
Part 040
