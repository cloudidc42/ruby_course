# Part 025: Model & ActiveRecord เบื้องต้น — Migration, Schema, และ CRUD ผ่าน Console

> **Step ครอบคลุมใน Part นี้:** Step 241–250
> **ระดับ:** กลาง (ต้องผ่าน Part 021–024 มาก่อน โดยเฉพาะเรื่อง `rails new`, Routing และ Controller)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3 ซึ่งเป็นค่าเริ่มต้นของ Rails 8 ตอนสร้างโปรเจกต์ใหม่)

จาก Part 021–024 เราเรียนเรื่อง Routing, Controller และ View ไปแล้ว — นั่นคือสองในสามของ
MVC แต่ทุกแอปที่เราทำมาจนถึงตอนนี้ยังไม่มี "ข้อมูลจริง" ที่เก็บถาวรอยู่เลย ข้อมูลหายไปทุกครั้ง
ที่ restart server ใน Part นี้เราจะมาเติมส่วนที่เหลือของ MVC นั่นคือ **Model** พร้อมกับ
**ActiveRecord** ซึ่งเป็นเลเยอร์ที่ทำให้ Ruby object คุยกับฐานข้อมูลได้โดยแทบไม่ต้องเขียน SQL เอง

## สารบัญของ Part นี้

- Step 241: ActiveRecord คืออะไร — ORM (Object-Relational Mapping) และบทบาทใน MVC
- Step 242: สร้าง Model ด้วย `rails generate model` และไฟล์ที่ถูกสร้างขึ้นมาทั้งหมด
- Step 243: Migration file และ method `change` — `create_table`, ชนิดข้อมูลของ column
- Step 244: รัน Migration ด้วย `db:migrate` และ `db/schema.rb` คือ source of truth
- Step 245: Rollback migration ด้วย `db:rollback` และการเพิ่ม column ด้วย migration ใหม่
- Step 246: Naming Convention — Convention over Configuration ระหว่าง Model กับ Table
- Step 247: CRUD ผ่าน `rails console` ส่วนที่ 1 — Create (`create`, `new` + `save`)
- Step 248: CRUD ผ่าน `rails console` ส่วนที่ 2 — Read (`find`, `find_by`, `where`, `all`)
- Step 249: CRUD ผ่าน `rails console` ส่วนที่ 3 — Update และ Delete
- Step 250: Attribute method ที่ Rails generate ให้อัตโนมัติ + แบบฝึกหัด Book model

---

## Step 241: ActiveRecord คืออะไร — ORM (Object-Relational Mapping) และบทบาทใน MVC

### ปัญหาที่ ORM แก้

ฐานข้อมูลเชิงสัมพันธ์ (Relational Database) อย่าง PostgreSQL, MySQL, SQLite เก็บข้อมูลเป็น
**ตาราง (table)** ที่มี **แถว (row)** และ **คอลัมน์ (column)** ส่วน Ruby เป็นภาษาเชิงวัตถุที่
ทำงานกับ **object** ที่มี **attribute** และ **method** สองโลกนี้มีรูปร่างข้อมูลไม่เหมือนกัน

ถ้าไม่มี ORM เราจะต้องเขียน SQL ตรงๆ แล้วแปลงผลลัพธ์เป็น Ruby object เอง เช่น

```ruby
# แบบไม่มี ORM (เขียน SQL ตรงๆ ด้วย library เชื่อมต่อ database เอง)
require "sqlite3"

db = SQLite3::Database.new("storage/development.sqlite3")
db.results_as_hash = true

rows = db.execute("SELECT * FROM posts WHERE published = 1")
rows.each do |row|
  puts row["title"]  # ต้องเข้าถึงผ่าน Hash key เอง ไม่มี method .title ให้ใช้
end
```

โค้ดแบบนี้ใช้งานได้ แต่มีปัญหาหลายอย่าง: ต้องเขียน SQL เองทุกครั้ง, เสี่ยงต่อ SQL Injection
ถ้าไม่ระวังการต่อ string, ผลลัพธ์เป็น Hash ธรรมดาไม่ใช่ object ที่มี behavior, และ SQL แต่ละ
database (SQLite, PostgreSQL, MySQL) มี syntax แตกต่างกันเล็กน้อย ทำให้ย้าย database ยาก

**ORM (Object-Relational Mapping)** คือเทคนิคการ "แปลง" ระหว่างตารางในฐานข้อมูลกับ class ใน
ภาษาโปรแกรมมิ่งโดยอัตโนมัติ กฎการแปลงพื้นฐานคือ:

| ฝั่งฐานข้อมูล (Database) | ฝั่ง Ruby/ActiveRecord |
|---------------------------|--------------------------|
| ตาราง (table) เช่น `posts` | Class เช่น `Post` |
| แถว (row) หนึ่งแถว | Object หนึ่งตัว (instance ของ `Post`) |
| คอลัมน์ (column) เช่น `title` | Attribute/method เช่น `post.title` |
| Primary key (`id`) | `post.id` |
| ความสัมพันธ์ระหว่างตาราง (foreign key) | Association เช่น `has_many`, `belongs_to` (Part 027) |

### ActiveRecord คือ ORM ของ Rails

**ActiveRecord** คือชื่อของ ORM layer ที่มากับ Rails โดยอัตโนมัติ (เป็นหนึ่งใน gem หลักที่อยู่ใน
`Gemfile` ตั้งแต่ `rails new` — ชื่อ gem คือ `activerecord`) หน้าที่ของมันคือ:

1. **Mapping** — เชื่อมโยง Ruby class เข้ากับตารางในฐานข้อมูลให้อัตโนมัติตามชื่อ (Convention
   over Configuration — จะสอนละเอียดใน Step 246)
2. **Query Interface** — ให้ method ภาษา Ruby แทนการเขียน SQL เช่น `Post.where(published: true)`
   แทน `SELECT * FROM posts WHERE published = 1`
3. **Migration** — ระบบจัดการการเปลี่ยนแปลงโครงสร้างตาราง (schema) แบบมีเวอร์ชัน (Step 243–245)
4. **Validation & Callback** — ตรวจสอบความถูกต้องของข้อมูลก่อนบันทึก และ hook เข้าไปในวงจรชีวิต
   ของ object (Part 026)
5. **Association** — จัดการความสัมพันธ์ระหว่างตาราง เช่น หนึ่งโพสต์มีหลายคอมเมนต์ (Part 027)

ActiveRecord เป็นการ implement ดีไซน์แพทเทิร์นที่ชื่อว่า **Active Record pattern** (ตรงกับชื่อ
gem พอดี) ซึ่งมีหลักการว่า object หนึ่งตัวควรจะรู้วิธี "บันทึกตัวเองลงฐานข้อมูล" และ "โหลดตัวเอง
ขึ้นมาจากฐานข้อมูล" ได้โดยตรง ไม่ต้องมี class แยกต่างหากสำหรับจัดการการอ่าน/เขียนฐานข้อมูล

### ตำแหน่งของ Model ใน MVC

```
Browser ──> Router ──> Controller ──> Model (ActiveRecord) ──> Database
                            │                                       │
                            └──────────────< View <─────────────────┘
```

Controller (Part 023) ไม่คุยกับฐานข้อมูลโดยตรง แต่จะเรียกใช้ Model ให้ไปดึง/บันทึกข้อมูลแทน
แล้วส่งผลลัพธ์ (ซึ่งเป็น Ruby object ธรรมดา) ต่อให้ View (Part 024) แสดงผล การแยกหน้าที่แบบนี้
ทำให้แต่ละส่วนทดสอบและแก้ไขแยกกันได้ ไม่ผูกติดกัน

> **หมายเหตุ:** ActiveRecord ไม่ใช่ ORM เดียวที่ใช้กับ Rails ได้ — มี gem ทางเลือกอย่าง
> `Sequel` หรือการต่อ MongoDB ผ่าน `Mongoid` แต่ ActiveRecord คือค่าเริ่มต้นและเป็นที่นิยม
> ที่สุดในระบบนิเวศ Rails หลักสูตรนี้จะใช้ ActiveRecord เป็นหลักตลอดทั้งหลักสูตร

---

## Step 242: สร้าง Model ด้วย `rails generate model` และไฟล์ที่ถูกสร้างขึ้นมาทั้งหมด

สมมติว่าตอนนี้เรามีแอป Rails ชื่อ `blog_app` จาก Part 021 อยู่แล้ว (ถ้ายังไม่มีให้สร้างใหม่ด้วย
`rails new blog_app` แล้ว `cd blog_app`) เราจะสร้าง Model ตัวแรกชื่อ `Post` สำหรับเก็บบทความ

```bash
bin/rails generate model Post title:string body:text published:boolean
```

ผลลัพธ์:

```
      invoke  active_record
      create    db/migrate/20260926024818_create_posts.rb
      create    app/models/post.rb
      invoke    test_unit
      create      test/models/post_test.rb
      create      test/fixtures/posts.yml
```

(ตัวเลขนำหน้าชื่อไฟล์ migration คือ timestamp ตอนสร้าง ของคุณจะไม่ตรงกับตัวอย่างนี้ — เป็น
เรื่องปกติ อ่านต่อได้ใน Step 243)

### วิเคราะห์รูปแบบคำสั่ง

```
rails generate model <ชื่อ Model แบบเอกพจน์ ขึ้นต้นด้วยตัวใหญ่> <ชื่อ column>:<ชนิดข้อมูล> ...
```

- **ชื่อ Model ต้องเป็นเอกพจน์ (singular)** เช่น `Post` ไม่ใช่ `Posts` — Rails จะแปลงเป็นชื่อ
  ตารางพหูพจน์ให้เองว่า `posts` (รายละเอียดเต็มใน Step 246)
- แต่ละ argument หลังชื่อ Model คือคู่ `column_name:type` คั่นด้วยช่องว่าง
- ถ้าไม่ระบุชนิดข้อมูล เช่น `rails generate model Post title` จะได้ column ชนิด `string` เป็น
  ค่า default

`generate model` (ย่อได้เป็น `g model`) สร้างไฟล์ให้ 3 กลุ่ม:

1. **Migration file** (`db/migrate/..._create_posts.rb`) — สคริปต์ที่บอก Rails ว่าต้องแก้
   โครงสร้างฐานข้อมูลอย่างไร (สอนละเอียด Step 243)
2. **Model file** (`app/models/post.rb`) — class Ruby ที่เป็นตัวแทนของตาราง `posts`
3. **ไฟล์ทดสอบ** (`test/models/post_test.rb` และ `test/fixtures/posts.yml`) — โครงสำหรับเขียน
   test ด้วย Minitest (สอนไปแล้วใน Part 018) ในหลักสูตรนี้ยังไม่ได้เปิดใช้ RSpec แทนอย่างเป็น
   ทางการจนถึง Part 046 ตอนนี้ปล่อยไฟล์เหล่านี้ไว้เฉยๆ ได้ก่อน

### เปิดดู `app/models/post.rb`

```ruby
class Post < ApplicationRecord
end
```

สั้นมาก — ไม่มี `attr_accessor`, ไม่มี `initialize`, ไม่มี method อะไรเลย แต่ class นี้จะมี
method อย่าง `.create`, `.find`, `#title`, `#save` ให้ใช้ได้ทันทีหลัง migrate เพราะมันสืบทอด
(`<`) มาจาก `ApplicationRecord`

```ruby
# app/models/application_record.rb (Rails สร้างให้อัตโนมัติตอน rails new)
class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
end
```

`ApplicationRecord` เป็น **abstract class** (`primary_abstract_class` บอก Rails ว่าคลาสนี้ไม่มี
ตารางของตัวเอง ใช้เป็น base class ให้ Model อื่น inherit เท่านั้น) ที่สืบทอดมาจาก
`ActiveRecord::Base` อีกที — และ `ActiveRecord::Base` นี่เองคือจุดที่ method วิเศษทั้งหมดของ
ActiveRecord ถูกนิยามไว้ Model ทุกตัวในแอป Rails ควรสืบทอดจาก `ApplicationRecord` (ไม่ใช่
`ActiveRecord::Base` ตรงๆ) เพื่อให้มีจุดกลางไว้เพิ่ม method ที่ Model ทุกตัวควรมีร่วมกันในอนาคต

> **เทียบกับ Controller:** จำได้ไหมว่าใน Part 023 ทุก Controller สืบทอดจาก
> `ApplicationController` ซึ่งสืบทอดจาก `ActionController::Base` — โครงสร้างเดียวกันเป๊ะ
> เพียงแค่เปลี่ยนจาก Action Controller framework เป็น Active Record framework เท่านั้น
> รูปแบบ "มี Base class กลางของแอปเอง คั่นกลางระหว่าง framework กับ class ของเรา" นี้เป็น
> convention ที่ Rails ใช้ซ้ำในหลายจุด

---

## Step 243: Migration file และ method `change` — `create_table`, ชนิดข้อมูลของ column

เปิดไฟล์ `db/migrate/20260926024818_create_posts.rb` (ชื่อไฟล์ของคุณจะมี timestamp ต่างออกไป):

```ruby
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body
      t.boolean :published

      t.timestamps
    end
  end
end
```

### Migration คืออะไร

**Migration** คือไฟล์ Ruby ที่อธิบาย "การเปลี่ยนแปลงโครงสร้างฐานข้อมูลหนึ่งขั้น" เช่น สร้างตาราง
ใหม่, เพิ่ม column, ลบ column, เพิ่ม index ฯลฯ จุดสำคัญคือ **migration ไม่ได้แก้ฐานข้อมูลทันที
ที่สร้างไฟล์** — มันเป็นแค่ "แผนงาน" ที่ยังไม่ถูกรัน จนกว่าจะสั่ง `rails db:migrate` (Step 244)

เหตุผลที่ Rails ออกแบบให้การเปลี่ยนโครงสร้างฐานข้อมูลต้องผ่านไฟล์ migration แทนการไปแก้ตาราง
ตรงๆ ผ่านเครื่องมือ database GUI:

1. **มีประวัติ (history)** — เห็นชัดว่าโครงสร้างฐานข้อมูลเปลี่ยนไปอย่างไรบ้างตามเวลา และ commit
   เข้า git ได้เหมือนโค้ดทั่วไป
2. **ทำงานเป็นทีมได้** — เพื่อนร่วมทีม pull โค้ดมาแล้วรัน `db:migrate` ก็ได้ฐานข้อมูลโครงสร้าง
   เดียวกันทันที ไม่ต้องส่งคำสั่ง SQL ให้กันเอง
3. **Reproducible** — deploy ขึ้น production หรือสร้าง environment ทดสอบใหม่ก็สั่ง migrate
   รันตามลำดับได้เหมือนเดิมทุกครั้ง
4. **ย้อนกลับได้ (reversible)** — ถ้า migration เขียนถูกวิธี จะ rollback ย้อนกลับไปโครงสร้าง
   ก่อนหน้าได้ (Step 245)

### ชื่อไฟล์คือ timestamp + ชื่อ class

`20260926024818_create_posts.rb` แบ่งเป็นสองส่วน:

- `20260926024818` — timestamp (ปี-เดือน-วัน-ชั่วโมง-นาที-วินาที) ณ ตอนที่สั่ง generate
  Rails ใช้ค่านี้เป็น **version number** ของ migration และใช้เรียงลำดับว่า migration ไหนควร
  รันก่อน-หลัง
- `create_posts` — สอดคล้องกับชื่อ class `CreatePosts` ข้างใน (แปลงจาก CamelCase เป็น
  snake_case)

`ActiveRecord::Migration[8.1]` ในการประกาศ class บอกว่า migration นี้เขียนด้วย syntax/พฤติกรรม
ของ Rails เวอร์ชัน 8.1 — เลขนี้ถูกใส่อัตโนมัติตาม Rails version ที่ใช้ generate ทำให้ถึงแม้จะ
อัปเกรด Rails เป็นเวอร์ชันใหม่ในอนาคต migration เก่าๆ ก็ยังทำงานด้วยพฤติกรรมเดิมที่เขียนไว้
ไม่พังจากการเปลี่ยนแปลงที่ breaking ใน API รุ่นใหม่

### method `change` — บอกทิศทางเดียว ได้ทั้งไปและกลับ

จุดที่ทรงพลังที่สุดของ migration คือ method `change` — เราเขียนแค่ทิศทาง "ไปข้างหน้า"
(เช่น สร้างตาราง) แต่ ActiveRecord ฉลาดพอที่จะอนุมานทิศทาง "ถอยหลัง" (ลบตาราง) ได้เองสำหรับ
คำสั่งพื้นฐานส่วนใหญ่ — จึงไม่ต้องเขียน method `up`/`down` แยกกันแบบ Rails รุ่นเก่า

คำสั่งที่ `change` อนุมานย้อนกลับได้เองมีเช่น `create_table` (ย้อนกลับ = `drop_table`),
`add_column` (ย้อนกลับ = `remove_column`), `add_index`, `rename_column`, `rename_table`
ส่วนคำสั่งที่ไม่สามารถอนุมานย้อนกลับได้เอง (เช่น การลบ column ที่มีข้อมูลอยู่แล้ว จะไม่รู้ว่า
ตอน rollback ต้องใส่ข้อมูลอะไรกลับเข้าไป) ต้องเขียน method `up`/`down` แยกเอง — เรื่องนี้จะ
กลับมาเจอรายละเอียดเพิ่มเติมใน Part 035

### `create_table` และชนิดข้อมูลของ column

```ruby
create_table :posts do |t|
  t.string  :title
  t.text    :body
  t.boolean :published

  t.timestamps
end
```

- `create_table :posts` รับชื่อตารางเป็น **พหูพจน์** (สอดคล้องกับ Step 246) และให้ block ที่มี
  ตัวแปร `t` (ย่อจาก table) ไว้นิยาม column ทีละบรรทัด
- `t.timestamps` เป็น shortcut พิเศษที่สร้าง 2 column พร้อมกันคือ `created_at` และ
  `updated_at` (ชนิด `datetime`, ห้ามเป็น `null`) — ActiveRecord จะอัปเดตค่าทั้งสองให้อัตโนมัติ
  ทุกครั้งที่สร้าง/แก้ไข record โดยไม่ต้องเขียนโค้ดเพิ่มเลย (แนะนำให้ใส่ `t.timestamps` ไว้
  แทบทุกตารางเสมอ เพราะมีประโยชน์มากตอน debug และเรียงลำดับข้อมูล)
- `id` column (primary key, auto-increment) ถูกสร้างให้อัตโนมัติโดยไม่ต้องเขียนเอง

ชนิดข้อมูล (column type) หลักที่ใช้บ่อยที่สุด:

| Type ใน migration | ชนิดข้อมูลจริงใน DB (SQLite) | ใช้เก็บอะไร |
|---------------------|-------------------------------|----------------|
| `:string` | VARCHAR (จำกัดความยาว ~255 ตัวอักษรโดย convention) | ข้อความสั้น เช่น ชื่อ, หัวข้อ, อีเมล |
| `:text` | TEXT (ไม่จำกัดความยาว) | ข้อความยาว เช่น เนื้อหาบทความ, คำบรรยาย |
| `:integer` | INTEGER | เลขจำนวนเต็ม เช่น อายุ, จำนวนหน้า |
| `:float` | FLOAT | เลขทศนิยม (แม่นยำน้อยกว่า decimal) |
| `:decimal` | DECIMAL | เลขทศนิยมที่ต้องแม่นยำ เช่น ราคาเงิน |
| `:boolean` | BOOLEAN (SQLite เก็บเป็น 0/1) | true/false เช่น สถานะเผยแพร่ |
| `:date` | DATE | วันที่ (ไม่มีเวลา) เช่น วันเกิด |
| `:datetime` | DATETIME | วันที่+เวลา เช่น เวลาที่โพสต์ |
| `:references` | INTEGER + สร้าง index/foreign key ให้ | เชื่อมไปยังตารางอื่น เช่น `user_id` (Part 027) |
| `:binary` | BLOB | ข้อมูลไบนารี (ปัจจุบันมักใช้ Active Storage แทน — Part 066) |

`t.string`, `t.text`, `t.boolean` ในตัวอย่างข้างบนเป็น syntax แบบย่อของ
`t.column :title, :string` — ทั้งสองแบบเทียบเท่ากัน แต่แบบ `t.string :title` อ่านง่ายกว่าและ
เป็นที่นิยมมากกว่าในทางปฏิบัติ

> **แนวคิดสำคัญ:** migration ที่ถูก `rails generate model` สร้างให้เป็นเพียงจุดเริ่มต้น
> เราแก้ไขไฟล์นี้เพิ่มเติมได้อย่างอิสระ **ตราบใดที่ยังไม่ได้ `db:migrate`** (ยังไม่เคยรันจริง)
> แต่ถ้า migrate ไปแล้วและ commit เข้า git ไปแล้ว (โดยเฉพาะถ้าคนอื่นในทีม pull ไปรันแล้ว)
> ห้ามย้อนไปแก้ไฟล์เดิมอีก — ให้สร้าง migration ไฟล์ใหม่มาแก้ไขต่อแทนเสมอ (ดู Step 245)

---

## Step 244: รัน Migration ด้วย `db:migrate` และ `db/schema.rb` คือ source of truth

### ขั้นแรก: สร้างฐานข้อมูล

ถ้ายังไม่เคยสร้างฐานข้อมูลของโปรเจกต์นี้มาก่อน (เช่นเพิ่ง `rails new` มาสดๆ) ให้สร้างก่อน:

```bash
bin/rails db:create
```

```
Created database 'storage/development.sqlite3'
Created database 'storage/test.sqlite3'
```

Rails 8 ใช้ SQLite3 เป็นค่าเริ่มต้นสำหรับ environment `development` และ `test` — ไฟล์ฐานข้อมูล
ทั้งสองถูกสร้างเป็นไฟล์ธรรมดาในโฟลเดอร์ `storage/` ไม่ต้องติดตั้งหรือรัน database server แยก
ต่างหากเลย (ต่างจาก PostgreSQL/MySQL ที่ต้องรัน service อยู่เบื้องหลัง) ทำให้เหมาะมากสำหรับ
เรียนรู้และพัฒนาเบื้องต้น ส่วนการสลับไปใช้ PostgreSQL ตอนขึ้น production จะพูดถึงใน Part 073+

### รัน migration ที่ค้างอยู่ทั้งหมด

```bash
bin/rails db:migrate
```

```
== 20260926024818 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0017s
== 20260926024818 CreatePosts: migrated (0.0017s) =============================
```

`db:migrate` จะมองหา migration ทุกไฟล์ใน `db/migrate/` ที่ **ยังไม่เคยรัน** (ยังไม่มี version
number นั้นบันทึกไว้ในตาราง `schema_migrations` ของฐานข้อมูล) แล้วรันตามลำดับ timestamp
จากเก่าไปใหม่ ทีละไฟล์ — ถ้ารันซ้ำอีกครั้งตอนไม่มี migration ใหม่ค้างอยู่ จะไม่มีอะไรเกิดขึ้น
(idempotent ในความหมายนี้)

ตรวจสอบสถานะ migration ทั้งหมดในโปรเจกต์ด้วย:

```bash
bin/rails db:migrate:status
```

```

database: storage/development.sqlite3

 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20260926024818  Create posts
```

`up` แปลว่า migration นี้ถูกรันแล้ว (โครงสร้างมีอยู่ในฐานข้อมูลปัจจุบัน) ถ้าเห็น `down` แปลว่า
ยังไม่ได้รัน (หรือถูก rollback ไปแล้ว)

### `db/schema.rb` — source of truth ของโครงสร้างฐานข้อมูลปัจจุบัน

ทุกครั้งที่ `db:migrate` สำเร็จ Rails จะเขียนไฟล์ `db/schema.rb` ใหม่ให้อัตโนมัติ ลองเปิดดู:

```ruby
# This file is auto-generated from the current state of the database. Instead
# of editing this file, please use the migrations feature of Active Record to
# incrementally modify your database, and then regenerate this schema definition.
#
# This file is the source Rails uses to define your schema when running `bin/rails
# db:schema:load`. When creating a new database, `bin/rails db:schema:load` tends to
# be faster and is potentially less error prone than running all of your
# migrations from scratch. Old migrations may fail to apply correctly if those
# migrations use external dependencies or application code.
#
# It's strongly recommended that you check this file into your version control system.

ActiveRecord::Schema[8.1].define(version: 2026_09_26_024818) do
  create_table "posts", force: :cascade do |t|
    t.string "title"
    t.text "body"
    t.boolean "published"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end
end
```

จุดสำคัญที่ต้องเข้าใจให้แม่น:

- **`schema.rb` คือภาพรวมของโครงสร้างฐานข้อมูล ณ ปัจจุบัน** ไม่ใช่ประวัติการเปลี่ยนแปลง —
  ต่างจาก migration files ที่แต่ละไฟล์คือ "การเปลี่ยนแปลงหนึ่งขั้น", `schema.rb` คือ
  "ผลลัพธ์สุดท้ายรวมทุกขั้น" ไฟล์เดียว
- **ห้ามแก้ไฟล์นี้ด้วยมือ** — คอมเมนต์บนสุดของไฟล์เตือนไว้ชัดเจนแล้วว่าไฟล์นี้ auto-generated
  ถ้าอยากเปลี่ยนโครงสร้างฐานข้อมูล ต้องสร้าง migration ใหม่เสมอ แล้วปล่อยให้ `db:migrate`
  อัปเดตไฟล์นี้ให้เอง
- **`version: 2026_09_26_024818`** คือ timestamp ของ migration ล่าสุดที่รันไปแล้ว บอกว่า
  ฐานข้อมูลนี้ "ทันสมัย" ถึงจุดไหนแล้ว
- **ควร commit ไฟล์นี้เข้า git เสมอ** เพราะเวลาตั้งค่า environment ใหม่ (เช่น เพื่อนร่วมทีม
  clone โปรเจกต์ไปใหม่ หรือ deploy ขึ้น production ครั้งแรก) คำสั่ง `rails db:schema:load`
  จะอ่านไฟล์นี้แล้วสร้างตารางทั้งหมดในทีเดียว **เร็วกว่า**การไล่รัน migration ทุกไฟล์ตั้งแต่
  ไฟล์แรกสุดทีละไฟล์มาก โดยเฉพาะโปรเจกต์ที่มี migration สะสมมาเป็นร้อยไฟล์
- คำสั่ง `bin/rails db:setup` และ `bin/rails db:prepare` ที่มักใช้ตอนเริ่มต้นโปรเจกต์ใหม่หรือ
  deploy จะเรียก `db:schema:load` นี้เบื้องหลัง ไม่ได้ไล่รัน migration ทีละไฟล์

> **สรุปความสัมพันธ์:** Migration files = ประวัติการเปลี่ยนแปลงทีละขั้น (เหมือน commit history
> ใน git) ส่วน `schema.rb` = สถานะล่าสุดรวมทุกอย่าง (เหมือนไฟล์จริงในโฟลเดอร์ทำงานตอนนี้)
> ทั้งสองไฟล์ต้องสอดคล้องกันเสมอ — ถ้า `db:migrate` รันสำเร็จ ทั้งสองจะ sync กันอัตโนมัติ

---

## Step 245: Rollback migration ด้วย `db:rollback` และการเพิ่ม column ด้วย migration ใหม่

### Rollback migration ล่าสุด

ระหว่างพัฒนา บางครั้งเราสร้าง migration ผิด (เช่น สะกดชื่อ column ผิด หรือเลือกชนิดข้อมูลผิด)
และยังไม่ได้ commit/push ให้ใครเห็น — กรณีนี้แก้ได้ง่ายด้วยการ rollback แล้วแก้ไฟล์แล้ว migrate
ใหม่:

```bash
bin/rails db:rollback
```

```
== 20260926024818 CreatePosts: reverting ======================================
-- drop_table(:posts)
   -> 0.0015s
== 20260926024818 CreatePosts: reverted (0.0057s) =============================
```

`db:rollback` จะย้อน migration **ล่าสุดที่รันแล้ว** กลับไปหนึ่งขั้น (เรียก `drop_table` ซึ่งเป็น
ทิศทางย้อนกลับที่ `change` อนุมานให้จาก `create_table` ตามที่อธิบายใน Step 243) เช็ค
`schema.rb` อีกครั้งจะเห็นว่ากลับไปว่างเปล่า:

```ruby
ActiveRecord::Schema[8.1].define(version: 0) do
end
```

`version: 0` แปลว่าไม่มี migration ไหนถูกรันเลย ตรวจสอบสถานะจะเห็นว่ากลับเป็น `down`:

```bash
bin/rails db:migrate:status
```

```

database: storage/development.sqlite3

 Status   Migration ID    Migration Name
--------------------------------------------------
  down    20260926024818  Create posts
```

ถ้าต้องการย้อนกลับหลายขั้นในทีเดียว ใช้ตัวเลือก `STEP`:

```bash
bin/rails db:rollback STEP=3   # ย้อนกลับ 3 migration ล่าสุด
```

ตอนนี้ยังไม่มีอะไรผิด แค่ต้องการกลับมาที่สถานะ migrate แล้วเหมือนเดิม ให้สั่ง migrate ใหม่:

```bash
bin/rails db:migrate
```

```
== 20260926024818 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0019s
== 20260926024818 CreatePosts: migrated (0.0020s) =============================
```

### กฎเหล็ก: migration ที่ migrate ไปแล้วและแชร์กับคนอื่นแล้ว ห้ามแก้ไฟล์เดิม

สมมติภายหลังเราอยากเพิ่ม column `views_count` เข้าไปในตาราง `posts` ที่มีอยู่แล้ว **ห้าม**
กลับไปแก้ไฟล์ `..._create_posts.rb` เดิมเด็ดขาด (โดยเฉพาะถ้า migrate ไปแล้วและมีคนอื่นในทีม
pull โค้ดไปรันแล้ว — เครื่องของเขาจะเห็นว่า migration นี้ `up` ไปแล้ว ต่อให้เราแก้ไฟล์เพิ่ม
column เข้าไป เครื่องเขาก็จะไม่รันไฟล์นี้ซ้ำ ทำให้ฐานข้อมูลของแต่ละคนไม่ตรงกัน) วิธีที่ถูกต้องคือ
**สร้าง migration ใหม่** เสมอ:

```bash
bin/rails generate migration AddViewsCountToPosts views_count:integer
```

```
      invoke  active_record
      create    db/migrate/20260926024840_add_views_count_to_posts.rb
```

สังเกตว่าคำสั่งนี้ใช้ `generate migration` (ไม่ใช่ `generate model` เพราะเราไม่ได้สร้าง Model
ใหม่ แค่แก้ไขตารางเดิม) และตั้งชื่อ migration แบบ `Add<ColumnName>To<TableName>` — เป็น
convention ที่ Rails รู้จักเป็นพิเศษ: ถ้าตั้งชื่อ migration ตามรูปแบบนี้พร้อมระบุ
`column:type` ต่อท้าย Rails จะเดาและสร้างเนื้อหาไฟล์ migration ให้เกือบสมบูรณ์ทันที:

```ruby
class AddViewsCountToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :views_count, :integer
  end
end
```

เราสามารถแก้ไขเพิ่มเติมให้มี default value และ `null: false` ได้ (แนะนำมากสำหรับ column ตัวเลข
ที่ไม่ควรเป็น `NULL`):

```ruby
class AddViewsCountToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :views_count, :integer, default: 0, null: false
  end
end
```

รัน migrate:

```bash
bin/rails db:migrate
```

```
== 20260926024840 AddViewsCountToPosts: migrating =============================
-- add_column(:posts, :views_count, {:default=>0, :null=>false})
   -> 0.0025s
== 20260926024840 AddViewsCountToPosts: migrated (0.0026s) ====================
```

`schema.rb` อัปเดตอัตโนมัติให้มี column ใหม่ต่อท้าย:

```ruby
ActiveRecord::Schema[8.1].define(version: 2026_09_26_024840) do
  create_table "posts", force: :cascade do |t|
    t.string "title"
    t.text "body"
    t.boolean "published"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.integer "views_count", default: 0, null: false
  end
end
```

นี่คือรูปแบบการทำงานจริงกับ ActiveRecord migration ตลอดอายุโปรเจกต์: **ตารางแต่ละตารางค่อยๆ
เติบโตทีละ migration** ไม่มีการแก้ไฟล์เก่าซ้ำ ไฟล์ migration ทุกไฟล์ที่เคย commit เข้า git แล้ว
ถือเป็นของตายที่แก้ไม่ได้อีก (ยกเว้นยังไม่เคย push ให้ใครเห็นเลย) — คำสั่ง generate migration
อื่นๆ ที่ใช้บ่อยมีเช่น `RemoveColumnFromTable`, `RenameColumnInTable` และการเขียน
`add_index` เอง ซึ่งจะพูดถึงเชิงลึกใน Part 034 และ Part 064

---

## Step 246: Naming Convention — Convention over Configuration ระหว่าง Model กับ Table

จำหลักการ **Convention over Configuration (CoC)** จาก Part 001 ได้ไหม — Rails กำหนดมาตรฐาน
การตั้งชื่อไว้ล่วงหน้า ถ้าเราทำตามมาตรฐานนี้ ระบบจะเชื่อมโยงทุกอย่างให้อัตโนมัติโดยไม่ต้องเขียน
config เพิ่มเลยสักบรรทัด นี่คือจุดที่ CoC แสดงพลังชัดเจนที่สุดจุดหนึ่งในทั้งเฟรมเวิร์ก:

| สิ่งที่ต้องตั้งชื่อ | รูปแบบ | ตัวอย่าง |
|----------------------|---------|-----------|
| ชื่อไฟล์ Model | snake_case เอกพจน์ | `app/models/post.rb` |
| ชื่อ class Model | CamelCase เอกพจน์ | `class Post` |
| ชื่อตารางในฐานข้อมูล | snake_case **พหูพจน์** | `posts` |
| Primary key column | `id` เสมอ (auto) | `posts.id` |
| Foreign key column (Part 027) | `<ชื่อโมเดลเอกพจน์>_id` | `posts.user_id` |

ActiveRecord จะเดาชื่อตารางจากชื่อ class โดยอัตโนมัติผ่านกระบวนการ **inflection** (การผัน
คำนามระหว่างเอกพจน์-พหูพจน์) — Rails มาพร้อม inflector ที่รู้จักกฎภาษาอังกฤษพื้นฐานและ
ข้อยกเว้นที่พบบ่อยเป็นจำนวนมาก:

```ruby
"Post".underscore.pluralize      # => "posts"      (กฎปกติ เติม s)
"Category".underscore.pluralize  # => "categories" (y -> ies)
"Person".underscore.pluralize    # => "people"     (ข้อยกเว้นที่ไม่ปกติ)
```

จะเห็นว่า `Person` ถูกแปลงเป็น `people` ไม่ใช่ `persons` — Rails รู้กฎภาษาอังกฤษแบบไม่ปกติ
(irregular plural) ที่พบบ่อยไว้ในตัวอยู่แล้ว ถ้า Model ของเราชื่อ `Person` แล้วสั่ง
`rails generate model Person name:string` ตารางที่ได้จะชื่อ `people` โดยอัตโนมัติ ไม่ต้องตั้งค่า
อะไรเพิ่มเลย

ถ้ามีคำเฉพาะทาง (เช่น คำย่อ, ชื่อแบรนด์) ที่ inflector เดาผิด สามารถสอนกฎเพิ่มเติมได้ที่ไฟล์
`config/initializers/inflections.rb`:

```ruby
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.irregular "octopus", "octopi"
end
```

### ทำไมต้องยึด convention นี้

ถ้าเราตั้งชื่อ Model หรือ table ไม่ตรง convention (เช่นตั้งตารางชื่อ `Post` เอกพจน์ตรงๆ) Model
`Post` จะหาไม่เจอว่าต้องอ่านตารางไหน แล้วจะ error ทันทีที่เรียกใช้ครั้งแรก:

```
ActiveRecord::StatementInvalid: Could not find table 'posts'
```

จริงอยู่ที่ ActiveRecord มีวิธี**บังคับ override**ชื่อตารางเองได้ด้วย `self.table_name = "..."`
ในตัว Model แต่ไม่แนะนำให้ทำถ้าไม่จำเป็นจริงๆ (เช่น ต้องเชื่อมกับฐานข้อมูลเก่าที่ตั้งชื่อไว้ก่อน
แล้วแก้ไม่ได้) เพราะจะทำให้โค้ดของเราต่างจากมาตรฐานที่นักพัฒนา Rails คนอื่นคาดหวัง และเสียข้อดี
ของ CoC ไป — **แนวทางที่ถูกต้องคือตั้งชื่อ Model ตาม convention ตั้งแต่แรก** แล้วปล่อยให้ Rails
เดาชื่อตารางให้เองเสมอ

---

## Step 247: CRUD ผ่าน `rails console` ส่วนที่ 1 — Create (`create`, `new` + `save`)

**CRUD** ย่อมาจาก **C**reate, **R**ead, **U**pdate, **D**elete — ปฏิบัติการพื้นฐาน 4 อย่างที่
แอปพลิเคชันแทบทุกตัวต้องทำกับข้อมูล ActiveRecord ให้ method ระดับสูงสำหรับทำ CRUD ทั้งหมดนี้
โดยไม่ต้องเขียน SQL เอง

เราจะทดลองผ่าน **`rails console`** ซึ่งเป็นเหมือน `irb`/`pry` (Part 001) แต่โหลด environment
ของ Rails ทั้งหมดมาด้วย ทำให้เรียก Model, เรียก helper ต่างๆ ของแอปได้ทันที

```bash
bin/rails console
# หรือย่อ: bin/rails c
```

```
Loading development environment (Rails 8.1.4)
irb(main):001>
```

> **เคล็ดลับ:** ใช้ `bin/rails console --sandbox` (หรือ `bin/rails c -s`) เวลาอยากทดลอง
> คำสั่งที่แก้ไขข้อมูล แต่ไม่อยากให้เปลี่ยนแปลงจริง — ทุกอย่างที่ทำใน sandbox mode จะถูก
> **rollback (ยกเลิกทั้งหมด) อัตโนมัติทันทีที่ออกจาก console** เหมาะมากตอนฝึกลองผิดลองถูก
> ```
> Loading development environment in sandbox (Rails 8.1.4)
> Any modifications you make will be rolled back on exit
> ```

### `Model.create` — สร้างและบันทึกในขั้นตอนเดียว

```irb
irb(main):001> Post.create(title: "Hello Rails", body: "โพสต์แรกของเรา", published: true)
=>
#<Post id: 1, title: "Hello Rails", body: "โพสต์แรกของเรา", published: true,
created_at: "2026-09-26 02:53:21.387640000 +0000", updated_at: "2026-09-26 02:53:21.387640000 +0000",
views_count: 0>
```

`Post.create(...)` รับ Hash ของ attribute เป็น argument แล้วทำ 2 อย่างพร้อมกันในคำสั่งเดียว:
1. สร้าง object ใหม่ในหน่วยความจำ (in-memory) พร้อมกำหนดค่า attribute ตามที่ส่งมา
2. สั่ง `INSERT` เข้าฐานข้อมูลทันที

สังเกตว่า attribute ที่เราไม่ได้ระบุ (`views_count`) ใช้ค่า `default: 0` ที่ตั้งไว้ตอน migration
(Step 245) และ `created_at`/`updated_at` ถูกเติมให้อัตโนมัติจาก `t.timestamps`

### `Model.new` + `.save` — แยกขั้นตอนสร้างกับบันทึก

บางครั้งเราอยากสร้าง object ไว้ก่อน ปรับแต่งค่าเพิ่มเติม แล้วค่อยตัดสินใจบันทึกทีหลัง —
ใช้ `.new` แยกจาก `.save` ได้:

```irb
irb(main):002> post = Post.new(title: "Draft Post", body: "ยังไม่เผยแพร่")
irb(main):003> post.persisted?
=> false
irb(main):004> post.new_record?
=> true
irb(main):005> post.save
=> true
irb(main):006> post.persisted?
=> true
irb(main):007> post
=>
#<Post id: 2, title: "Draft Post", body: "ยังไม่เผยแพร่", published: nil,
created_at: "2026-09-26 02:53:21.394008000 +0000", updated_at: "2026-09-26 02:53:21.394008000 +0000",
views_count: 0>
```

- `Post.new(...)` สร้าง object ในหน่วยความจำอย่างเดียว **ยังไม่แตะฐานข้อมูล**
- `post.persisted?` บอกว่า object นี้เคยถูกบันทึกลงฐานข้อมูลแล้วหรือยัง (มี `id` จริงในตาราง
  หรือไม่) — ก่อน `.save` เป็น `false`
- `post.new_record?` คือด้านตรงข้ามของ `persisted?` — ก่อน save เป็น `true`
- `post.save` สั่งบันทึกจริง คืนค่า `true` ถ้าสำเร็จ (หรือ `false` ถ้าบันทึกไม่ผ่าน เช่น
  validation ไม่ผ่าน — เรื่อง validation จะสอนเต็มใน Part 026)
- สังเกตว่า `published` ไม่ได้ระบุตอนสร้าง เลยเป็น `nil` (ไม่ใช่ `false`) — นี่คือเหตุผลที่
  Part 026 จะสอนเรื่อง validation และการตั้งค่า default ที่ระดับ Model เพิ่มเติม เพื่อป้องกัน
  ค่าที่ไม่ควรเป็น `nil` หลุดเข้าฐานข้อมูล

```irb
irb(main):008> Post.count
=> 2
```

**เลือกใช้แบบไหนดี:** ใช้ `create` เมื่อมีข้อมูลครบและอยากบันทึกทันที (เช่น รับข้อมูลจาก form
ใน controller — จะเจอบ่อยมากใน Part 031) ใช้ `new` + `save` แยกกันเมื่อต้องการ logic คั่นกลาง
ระหว่างสร้างกับบันทึก เช่น ต้องคำนวณค่าบาง attribute ก่อน หรือต้องการตรวจสอบ `valid?` ก่อน
ตัดสินใจ save จริง

---

## Step 248: CRUD ผ่าน `rails console` ส่วนที่ 2 — Read (`find`, `find_by`, `where`, `all`)

ต่อจาก Step 247 ตอนนี้ตาราง `posts` มี 2 แถว (`id: 1` "Hello Rails" กับ `id: 2` "Draft Post")

### ดึงทั้งหมด, ตัวแรก, ตัวสุดท้าย

```irb
irb(main):001> Post.all
=>
#<ActiveRecord::Relation [#<Post id: 1, title: "Hello Rails", ...>, #<Post id: 2, title: "Draft Post", ...>]>

irb(main):002> Post.first
=> #<Post id: 1, title: "Hello Rails", body: "โพสต์แรกของเรา", published: true, ...>

irb(main):003> Post.last
=> #<Post id: 2, title: "Draft Post", body: "ยังไม่เผยแพร่", published: nil, ...>

irb(main):004> Post.count
=> 2
```

`Post.all` ไม่ได้คืน Array ธรรมดา แต่คืน **`ActiveRecord::Relation`** ซึ่งเป็น object พิเศษที่
มีพฤติกรรมคล้าย Array มาก (loop ด้วย `each` ได้, ใช้ `map`/`select` ได้เหมือน Enumerable ปกติ
จาก Part 013) แต่ฉลาดกว่าตรงที่ **query ยังไม่ถูกยิงไปที่ฐานข้อมูลจริงจนกว่าจะต้องใช้ผลลัพธ์จริง
(lazy evaluation)** — เรื่องนี้จะเจาะลึกเรื่อง query interface และ N+1 problem ใน Part 034

### ค้นหาด้วย primary key: `find`

```irb
irb(main):005> Post.find(1)
=> #<Post id: 1, title: "Hello Rails", body: "โพสต์แรกของเรา", published: true, ...>
```

`find` รับค่า `id` แล้วคืน object ตัวเดียว **ถ้าหาไม่เจอจะ raise exception ทันที**
(`ActiveRecord::RecordNotFound`) ไม่คืน `nil` เงียบๆ:

```irb
irb(main):006> Post.find(999)
ActiveRecord::RecordNotFound (Couldn't find Post with 'id'=999)
```

ต้องดักด้วย `begin/rescue` (Part 011) ถ้าไม่อยากให้แอป crash:

```ruby
begin
  Post.find(999)
rescue ActiveRecord::RecordNotFound => e
  puts "ไม่พบโพสต์: #{e.message}"
end
# => ไม่พบโพสต์: Couldn't find Post with 'id'=999
```

ในทางปฏิบัติ เวลาเขียน controller (Part 023) เราแทบไม่ต้อง `rescue` เองเลย เพราะ Rails
ดักข้อผิดพลาดนี้ให้อัตโนมัติแล้วแสดงหน้า 404 ในโหมด production

### ค้นหาแบบไม่ raise error: `find_by`

```irb
irb(main):007> Post.find_by(title: "Draft Post")
=> #<Post id: 2, title: "Draft Post", body: "ยังไม่เผยแพร่", published: nil, ...>

irb(main):008> Post.find_by(title: "ไม่มีจริง")
=> nil
```

`find_by` รับเงื่อนไขเป็น Hash (ไม่จำกัดแค่ `id`) และคืน **object ตัวแรกที่ตรงเงื่อนไข หรือ
`nil` ถ้าไม่เจอเลย** — ไม่ raise error เหมือน `find` เหมาะกับสถานการณ์ที่ "ไม่เจอ" ถือเป็นกรณี
ปกติ เช่น ค้นหา user จากอีเมลตอน login (ถ้าไม่เจอก็แค่แจ้งว่า "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
ไม่ใช่ error ของระบบ)

### ค้นหาแบบมีเงื่อนไข หลายผลลัพธ์: `where`

```irb
irb(main):009> Post.where(published: true)
=> #<ActiveRecord::Relation [#<Post id: 1, title: "Hello Rails", ...>]>

irb(main):010> Post.where(published: true).count
=> 1
```

`where` คืน `ActiveRecord::Relation` เสมอ (แม้จะเจอแค่ตัวเดียวหรือไม่เจอเลยก็ตาม) ต่างจาก
`find`/`find_by` ที่คืน object เดี่ยว หรือ error/nil เพราะจุดประสงค์ของ `where` คือเตรียมชุด
ผลลัพธ์ไว้กรอง/เรียงลำดับ/ต่อเงื่อนไขเพิ่มได้อีก (`Post.where(published: true).order(:title)`)
— เรื่อง query แบบต่อเนื่องนี้จะเรียนเต็มใน Part 034

**สรุปตารางเปรียบเทียบ method อ่านข้อมูลหลัก:**

| Method | รับ argument | ถ้าไม่เจอ | คืนค่าเมื่อเจอ |
|---------|---------------|------------|------------------|
| `find(id)` | primary key | raise `RecordNotFound` | object เดี่ยว |
| `find_by(hash)` | เงื่อนไขใดก็ได้ | คืน `nil` | object เดี่ยว (ตัวแรกที่เจอ) |
| `where(hash)` | เงื่อนไขใดก็ได้ | คืน Relation ว่าง | `ActiveRecord::Relation` (หลายตัว) |
| `all` | ไม่มี | — | `ActiveRecord::Relation` (ทุกแถว) |
| `first` / `last` | ไม่มี | คืน `nil` | object เดี่ยว |

---

## Step 249: CRUD ผ่าน `rails console` ส่วนที่ 3 — Update และ Delete

### Update แบบที่ 1: assign ทีละ attribute แล้ว `save`

```irb
irb(main):001> post1 = Post.find(1)
irb(main):002> post1.title = "Hello Rails (แก้ไขแล้ว)"
irb(main):003> post1.changed?
=> true
irb(main):004> post1.title_changed?
=> true
irb(main):005> post1.save
=> true
irb(main):006> post1.title
=> "Hello Rails (แก้ไขแล้ว)"
```

การ assign ค่าใหม่ผ่าน setter (`post1.title = ...`) จะเปลี่ยนแค่ค่าใน object ที่อยู่ในหน่วยความ
จำเท่านั้น **ยังไม่ถูกเขียนลงฐานข้อมูลจนกว่าจะเรียก `.save`** ระหว่างนั้น ActiveRecord จะคอย
ติดตาม (track) ว่า attribute ไหนถูกแก้ไปแล้วบ้างผ่าน method อย่าง `changed?` และ
`<attribute>_changed?` — มีประโยชน์มากตอนเขียน callback (Part 026) ที่ต้องเช็คว่า attribute
บางตัวถูกแก้ไหมก่อนจะทำ logic บางอย่าง

### Update แบบที่ 2: `update` — assign + save ในคำสั่งเดียว

```irb
irb(main):007> post2 = Post.find(2)
irb(main):008> post2.update(title: "Published Post", published: true)
=> true
irb(main):009> post2
=>
#<Post id: 2, title: "Published Post", body: "ยังไม่เผยแพร่", published: true,
created_at: "...", updated_at: "...", views_count: 0>
```

`update(hash)` เทียบเท่ากับการเขียน `assign_attributes(hash); save` สองบรรทัดในคำสั่งเดียว
รับ Hash ของ attribute ที่จะเปลี่ยน (ไม่จำเป็นต้องส่งครบทุก column — attribute ที่ไม่ระบุจะคงค่า
เดิมไว้) คืน `true`/`false` เหมือน `.save` เป็น method ที่ใช้บ่อยที่สุดในการ update ทั้งใน
console และใน controller จริง (เช่น edit form)

### `update_attribute` — update ทีละ attribute เดียว (ข้าม validation)

```irb
irb(main):010> post1.update_attribute(:views_count, 10)
=> true
irb(main):011> post1.reload.views_count
=> 10
```

`update_attribute` (เอกพจน์ ไม่มี `s`) รับ attribute เดียวกับค่าใหม่ แล้ว save ให้ทันที
**ข้าม validation ทั้งหมด** (แต่ยังรัน callback ปกติ) ต่างจาก `update` (พหูพจน์) ที่รับได้หลาย
attribute พร้อมกันและ **ยังคง run validation ตามปกติ** เพราะ `update_attribute` ข้าม validation
จึงควรใช้อย่างระมัดระวัง เหมาะกับกรณีที่ตั้งใจข้ามการตรวจสอบจริงๆ เช่น การอัปเดต counter cache
ภายในระบบเอง ไม่ใช่ข้อมูลที่มาจากผู้ใช้โดยตรง

`post1.reload` สั่งให้ object โหลดข้อมูลล่าสุดจากฐานข้อมูลกลับเข้ามาทับค่าที่มีอยู่ในหน่วยความจำ
มีประโยชน์เวลาต้องการยืนยันว่าค่าที่บันทึกไปจริงๆ ตรงกับที่คาดหวัง

> **สังเกตความสำคัญของ validation:** ลองสั่ง `post1.update(title: nil)` ตอนนี้ (ก่อนที่จะเรียน
> validation ใน Part 026) จะพบว่ามันคืนค่า `true` และ `title` กลายเป็น `nil` จริงๆ! เพราะเรายัง
> ไม่ได้เขียนกฎอะไรบอก ActiveRecord ว่า `title` ห้ามว่าง — นี่คือเหตุผลว่าทำไม Part ถัดไปถึง
> สำคัญมาก: โดย default แล้ว ActiveRecord ยอมให้บันทึกข้อมูลอะไรก็ได้ลงไป ตราบใดที่ชนิดข้อมูล
> ไม่ขัดกับ column type การป้องกันข้อมูลผิดพลาดเป็นหน้าที่ของเราที่ต้องเขียน validation เพิ่มเอง

### `destroy` vs `delete` — ลบ record

```irb
irb(main):012> Post.count
=> 2
irb(main):013> post2 = Post.find(2)
irb(main):014> post2.destroy
=>
#<Post id: 2, title: "Published Post", body: "ยังไม่เผยแพร่", published: true, ...>
irb(main):015> Post.count
=> 1
```

```irb
irb(main):016> Post.delete(1)
=> 1
irb(main):017> Post.count
=> 0
irb(main):018> Post.all
=> #<ActiveRecord::Relation []>
```

ความแตกต่างสำคัญระหว่างสองคำสั่งนี้:

| | `destroy` (instance method) | `delete` (class method) |
|---|-------------------------------|-----------------------------|
| เรียกผ่าน | `post.destroy` (ต้องมี object อยู่แล้ว) | `Post.delete(id)` (ระบุแค่ id) |
| รัน validation/callback | รัน (`before_destroy`, `after_destroy` ฯลฯ — Part 026) | **ไม่รัน** |
| ลบ association ที่ `dependent: :destroy` ไว้ | ลบตาม (Part 027) | **ไม่ลบตาม** |
| ความเร็ว | ช้ากว่าเล็กน้อย (มี overhead ของ callback) | เร็วกว่า (`DELETE` SQL ตรงๆ) |
| คืนค่า | object ที่ถูกลบ (frozen แล้ว) | จำนวนแถวที่ถูกลบ |

กฎทางปฏิบัติ: **ใช้ `destroy` เป็นค่าเริ่มต้นเสมอ** เพราะปลอดภัยกว่า — callback และ dependent
association เป็นกลไกสำคัญที่ป้องกันข้อมูล "กำพร้า" (orphaned record) เช่น ถ้าลบ `Post` ที่มี
`Comment` ผูกอยู่ ควรลบ comment ที่เกี่ยวข้องตามไปด้วย ใช้ `delete`/`delete_all` เฉพาะกรณีที่
มั่นใจแล้วว่าไม่ต้องการ callback ใดๆ และต้องการความเร็วสูงสุด เช่น ลบข้อมูล log เก่าจำนวนมาก
เป็น batch

`delete_all` และ `destroy_all` ก็มีความแตกต่างแบบเดียวกัน แต่ทำงานกับหลายแถวพร้อมกัน:

```irb
irb(main):019> Book.delete_all
=> 2
```

---

## Step 250: Attribute method ที่ Rails generate ให้อัตโนมัติ + แบบฝึกหัด Book model

### ย้อนกลับไปดู Part 009: ทำไมเราไม่ต้องเขียน `attr_accessor` เอง

จำได้ไหมว่าใน Part 009 เวลาสร้าง class ธรรมดา เราต้องเขียน `attr_accessor` เองเพื่อให้เข้าถึง
instance variable จากภายนอกได้:

```ruby
# วิธีเดิมจาก Part 009 — class ธรรมดา ไม่ใช่ ActiveRecord
class Dog
  attr_accessor :name, :breed

  def initialize(name, breed)
    @name = name
    @breed = breed
  end
end

rex = Dog.new("Rex", "Golden Retriever")
rex.name   # => "Rex"      (ต้องมี attr_accessor :name ถึงเรียกได้)
```

ถ้าไม่มี `attr_accessor :name` บรรทัดนั้น `rex.name` จะ error ทันทีด้วย `NoMethodError`
เพราะ Ruby ไม่รู้จัก method `name` ให้เลย — เราต้องประกาศเองทุก attribute

แต่กับ `Post < ApplicationRecord` เราไม่เคยเขียน `attr_accessor` เลยสักบรรทัด และ
`post.title`, `post.title = "..."`, `post.published?` ก็ใช้งานได้หมด ลองพิสูจน์:

```irb
irb(main):001> Post.column_names
=> ["id", "title", "body", "published", "created_at", "updated_at", "views_count"]

irb(main):002> Post.new.respond_to?(:title)
=> true
irb(main):003> Post.new.respond_to?(:title=)
=> true
irb(main):004> Post.new.attributes
=>
{"id"=>nil, "title"=>nil, "body"=>nil, "published"=>nil,
 "created_at"=>nil, "updated_at"=>nil, "views_count"=>0}
```

### เบื้องหลัง: ActiveRecord อ่าน schema แล้ว define method ให้อัตโนมัติ

ตอนแอปเริ่มทำงาน (หรือตอน Model ถูกโหลดครั้งแรก) ActiveRecord จะ:

1. เชื่อมต่อฐานข้อมูล แล้วถาม (introspect) ว่าตาราง `posts` มี column อะไรบ้าง ชนิดอะไร
   (ข้อมูลนี้มาจาก `db/schema.rb` ที่เราสร้างผ่าน migration ตั้งแต่ Step 242–245 นั่นเอง)
2. สำหรับทุก column ที่เจอ **สร้าง (define) getter method, setter method (`<col>=`), และ
   สำหรับ column ชนิด boolean สร้าง query method (`<col>?`) เพิ่มให้อัตโนมัติ** ผ่านกลไก
   metaprogramming แบบที่เรียนใน Part 015 (`method_missing` และ `define_method` เบื้องหลัง
   การ implement จริงของ ActiveRecord ซับซ้อนกว่านี้มากเพื่อเรื่อง performance แต่หลักการ
   แนวคิดคือแบบนี้)

```irb
irb(main):005> post = Post.new(title: "ทดสอบ", body: "เนื้อหา", published: false)
irb(main):006> post.title
=> "ทดสอบ"
irb(main):007> post.published?
=> false
```

`published?` เป็น **boolean query method** ที่ Rails สร้างให้อัตโนมัติเฉพาะ column ชนิด
`boolean` — จะคืนค่า `true`/`false` เสมอ (ไม่คืน `nil`) ทำให้เขียนเงื่อนไขใน view/controller
ได้อ่านง่ายกว่าการเรียก `post.published == true` ตรงๆ

**นี่คือสิ่งที่ Rails "อัตโนมัติ" ให้เรา** — เราไม่ต้องเขียน `attr_accessor`, ไม่ต้องเขียน
`initialize` รับ Hash เอง, ไม่ต้องเขียน getter/setter สำหรับทุก column เอง เพราะ **โครงสร้าง
ตาราง (schema) ก็คือ "สัญญา" (contract) ที่บอกอยู่แล้วว่า object นี้ควรมี attribute อะไรบ้าง**
ActiveRecord อ่านสัญญานั้นแล้วสร้างโค้ดให้เราหมด — ถ้าอยากเพิ่ม attribute ใหม่ ก็แค่เพิ่ม
migration เพิ่ม column (Step 245) ไม่ต้องมาแก้ไฟล์ Model เพื่อเพิ่ม `attr_accessor` เองเลย

> **ข้อควรระวัง:** ตรงกันข้ามกับ class ธรรมดาที่ attribute ทั้งหมดถูกกำหนดตายตัวตอนเขียนโค้ด
> attribute ของ ActiveRecord model **ขึ้นอยู่กับโครงสร้างฐานข้อมูลจริง ณ ขณะนั้น** ถ้าเพิ่ม
> column ในฐานข้อมูลโดยตรง (ไม่ผ่าน migration) แล้วไม่มีไฟล์ migration บันทึกไว้ คนอื่นในทีม
> หรือ production server ที่ migrate จากศูนย์จะไม่มี column นั้น — ยิ่งตอกย้ำว่าทำไมต้องเปลี่ยน
> โครงสร้างฐานข้อมูลผ่าน migration เท่านั้น (Step 243–245)

---

## แบบฝึกหัด: สร้าง `Book` model ครบวงจร (Model, Migration, CRUD)

### โจทย์

สร้าง Model ชื่อ `Book` สำหรับเก็บรายการหนังสือ มี attribute ดังนี้:

- `title` (string) — ชื่อหนังสือ
- `author` (string) — ชื่อผู้แต่ง
- `pages` (integer) — จำนวนหน้า
- `read` (boolean) — อ่านจบแล้วหรือยัง

จากนั้นเขียนสคริปต์ทดสอบผ่าน `rails console` (หรือไฟล์แล้วรันด้วย `rails runner`) ที่ทำ CRUD
ครบทั้ง 4 อย่าง พร้อมพิมพ์ผลลัพธ์ยืนยันในแต่ละขั้นตอน

### เฉลย

**1) Generate model และ migrate**

```bash
bin/rails generate model Book title:string author:string pages:integer read:boolean
```

```
      invoke  active_record
      create    db/migrate/20260926025151_create_books.rb
      create    app/models/book.rb
      invoke    test_unit
      create      test/models/book_test.rb
      create      test/fixtures/books.yml
```

ตรวจดูไฟล์ migration ที่ได้ (ตรงตามโจทย์ทุก attribute พอดี ไม่ต้องแก้ไข):

```ruby
class CreateBooks < ActiveRecord::Migration[8.1]
  def change
    create_table :books do |t|
      t.string :title
      t.string :author
      t.integer :pages
      t.boolean :read

      t.timestamps
    end
  end
end
```

```bash
bin/rails db:migrate
```

```
== 20260926025151 CreateBooks: migrating ======================================
-- create_table(:books)
   -> 0.0015s
== 20260926025151 CreateBooks: migrated (0.0015s) =============================
```

`app/models/book.rb` ที่ได้ (ไม่ต้องแก้ไขอะไรเพิ่ม เพราะยังไม่เรียน validation/association):

```ruby
class Book < ApplicationRecord
end
```

**2) CRUD walkthrough เต็มรูปแบบผ่าน `rails console`**

```irb
irb(main):001> book1 = Book.create(title: "ไอน์สไตน์พบ พระพุทธเจ้าเห็น", author: "ทันตแพทย์สม สุจริตกุล", pages: 320, read: false)
=>
#<Book id: 1, title: "ไอน์สไตน์พบ พระพุทธเจ้าเห็น", author: "ทันตแพทย์สม สุจริตกุล", pages: 320,
read: false, created_at: "...", updated_at: "...">

irb(main):002> book2 = Book.new(title: "Sapiens", author: "Yuval Noah Harari", pages: 443)
irb(main):003> book2.save
=> true

irb(main):004> book3 = Book.create(title: "Clean Code", author: "Robert C. Martin", pages: 464, read: true)

irb(main):005> Book.count
=> 3
```

**Read — ดึงข้อมูลกลับมาตรวจสอบ**

```irb
irb(main):006> Book.all.pluck(:id, :title, :author, :read)
=>
[[1, "ไอน์สไตน์พบ พระพุทธเจ้าเห็น", "ทันตแพทย์สม สุจริตกุล", false],
 [2, "Sapiens", "Yuval Noah Harari", nil],
 [3, "Clean Code", "Robert C. Martin", true]]

irb(main):007> Book.find(1)
=> #<Book id: 1, title: "ไอน์สไตน์พบ พระพุทธเจ้าเห็น", ...>

irb(main):008> Book.find_by(author: "Yuval Noah Harari")
=> #<Book id: 2, title: "Sapiens", author: "Yuval Noah Harari", pages: 443, read: nil, ...>

irb(main):009> Book.where(read: false).count
=> 1
irb(main):010> Book.where(read: false).pluck(:title)
=> ["ไอน์สไตน์พบ พระพุทธเจ้าเห็น"]
```

(`pluck(:col1, :col2, ...)` ดึงเฉพาะค่า column ที่ระบุออกมาเป็น Array ตรงๆ โดยไม่ต้องโหลดทั้ง
object — มีประโยชน์มากเวลาต้องการแค่บาง field เพื่อความเร็ว จะพูดถึงเชิงลึกใน Part 034)

**Update**

```irb
irb(main):011> book2 = Book.find_by(title: "Sapiens")
irb(main):012> book2.read = true
irb(main):013> book2.save
=> true

irb(main):014> Book.find(book2.id).update(pages: 498)
=> true

irb(main):015> book1.update_attribute(:read, true)
=> true

irb(main):016> Book.where(read: true).count
=> 3
```

**Delete**

```irb
irb(main):017> Book.count
=> 3
irb(main):018> book3 = Book.find_by(title: "Clean Code")
irb(main):019> book3.destroy
=> #<Book id: 3, title: "Clean Code", author: "Robert C. Martin", pages: 464, read: true, ...>
irb(main):020> Book.count
=> 2

irb(main):021> Book.delete_all
=> 2
irb(main):022> Book.count
=> 0
```

**สรุปผลลัพธ์:** สร้าง 3 เล่ม, อ่าน/กรองข้อมูลได้ทุกรูปแบบ (find, find_by, where, pluck),
อัปเดตได้ทั้งแบบ assign+save, `update`, `update_attribute`, และลบได้ทั้งแบบ `destroy` ทีละเล่ม
กับ `delete_all` ล้างทั้งตารางในคำสั่งเดียว — ครบวงจร CRUD ตามโจทย์

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. สร้าง Model ใหม่ชื่อ `Category` ที่มี attribute เดียวคือ `name:string` แล้วตรวจสอบด้วยตัวเอง
   ผ่าน `rails console` ว่าชื่อตารางที่ ActiveRecord map ให้กลายเป็นอะไร (ใบ้: ลองรัน
   `Category.table_name`) พร้อมอธิบายว่าทำไมถึงเป็นชื่อนั้น
2. เพิ่ม column ใหม่ชื่อ `genre` (ชนิด string) เข้าไปในตาราง `books` ด้วยการสร้าง migration
   ใหม่ (ห้ามแก้ไฟล์ `create_books` เดิม) ตั้งชื่อ migration ตาม convention จาก Step 245
   (`AddGenreToBooks genre:string`) แล้ว `db:migrate` จากนั้นเปิด `rails console` มาอัปเดต
   หนังสือที่มีอยู่ให้มีค่า `genre` ด้วย `update`
3. ทดลองสั่ง `Book.find(999)` ใน console แล้วสังเกต error ที่ได้ จากนั้นเขียนไฟล์ Ruby เล็กๆ
   (รันด้วย `bin/rails runner`) ที่ใช้ `begin/rescue` ดัก `ActiveRecord::RecordNotFound`
   แล้วพิมพ์ข้อความภาษาไทยว่า "ไม่พบหนังสือเล่มนี้" แทนที่จะปล่อยให้โปรแกรม crash

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **ActiveRecord คือ ORM** ของ Rails ที่แปลงตารางในฐานข้อมูลให้เป็น Ruby class และแถวข้อมูล
  ให้เป็น object โดยอัตโนมัติ ไม่ต้องเขียน SQL ตรงๆ
- สร้าง Model พร้อม migration ในคำสั่งเดียวด้วย `rails generate model` และเข้าใจไฟล์ทั้งหมด
  ที่ถูกสร้างขึ้นมา
- Migration file ใช้ method `change` เขียนทิศทางเดียว แต่ ActiveRecord อนุมานทิศทางย้อนกลับ
  ให้เองสำหรับคำสั่งพื้นฐาน (`create_table`, `add_column` ฯลฯ)
- `db/schema.rb` คือ source of truth ของโครงสร้างฐานข้อมูล ณ ปัจจุบัน ถูกสร้างใหม่อัตโนมัติทุก
  ครั้งที่ `db:migrate` — ห้ามแก้ไฟล์นี้ด้วยมือ
- `db:rollback` ย้อน migration กลับได้ แต่กฎเหล็กคือ **ห้ามแก้ไฟล์ migration เดิมที่แชร์กับคน
  อื่นแล้ว** ต้องสร้าง migration ใหม่มาต่อยอดเสมอ
- Convention over Configuration ระหว่าง Model (เอกพจน์ CamelCase) กับตาราง (พหูพจน์
  snake_case) ทำให้ไม่ต้องตั้งค่าการเชื่อมโยงเอง แม้แต่คำนามที่ผันไม่ปกติอย่าง `Person` ->
  `people` ก็รองรับให้แล้ว
- ทำ CRUD ครบผ่าน `rails console`: `create`/`new`+`save` (Create), `find`/`find_by`/`where`
  (Read), `update`/`update_attribute` (Update), `destroy`/`delete` (Delete) พร้อมเข้าใจ
  ความแตกต่างของแต่ละคู่ method
- เข้าใจว่า attribute getter/setter ของ ActiveRecord model ถูก generate อัตโนมัติจาก schema
  ของฐานข้อมูล ต่างจาก class ธรรมดาใน Part 009 ที่ต้องเขียน `attr_accessor` เอง

**ต่อไป (Part 026):** เราจะสังเกตเห็นแล้วว่าตอนนี้ ActiveRecord ยอมให้บันทึกข้อมูลอะไรก็ได้
ลงไปโดยไม่ตรวจสอบเลย (`title` เป็น `nil` ก็ยัง save ผ่าน) — Part 026 จะสอนเรื่อง **Validation**
(เช่น `validates :title, presence: true`) เพื่อป้องกันข้อมูลผิดพลาดตั้งแต่ระดับ Model และ
**Callback** (`before_save`, `after_create` ฯลฯ) เพื่อรันโค้ดอัตโนมัติตามจุดต่างๆ ในวงจรชีวิต
ของ object ซึ่งเป็นรากฐานสำคัญก่อนจะไปเรียนเรื่อง Association ใน Part 027
