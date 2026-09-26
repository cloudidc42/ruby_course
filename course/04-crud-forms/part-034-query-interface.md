# Part 034: Query Interface เชิงลึก — `where`, `order`, `joins`, `includes` (N+1), `scope`

> **Step ครอบคลุมใน Part นี้:** Step 331–340
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 027 เรื่อง Association เบื้องต้น และ Part 033 เรื่อง
> Association ขั้นสูงมาก่อน — โดยเฉพาะความเข้าใจเรื่อง N+1 query ที่แนะนำไว้คร่าวๆ ใน Part 027
> Step 269)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทุกคำสั่งและทุก SQL ในเอกสารนี้รันจริงและ capture
> log จริงบน Ruby 3.3.6 + Rails 8.1.4 + ฐานข้อมูล SQLite3)

ใน Part 027 เราเจอปัญหา N+1 query เป็นครั้งแรก และแก้แบบเบื้องต้นด้วย `includes` — ตอนนั้นบอกไว้ว่า
"รายละเอียดเชิงลึกเรื่อง query interface ทั้งหมดจะไปเจาะลึกอีกทีใน Part 034" ถึงเวลานั้นแล้ว

Part นี้จะรวมสองบริบทที่เคยแยกกันมาก่อน — **Mini Blog** จาก Part 030 (`Post`) และแนวคิด
`Author` จาก Part 027 — เข้าเป็นโดเมนเดียวที่สมจริงขึ้น: บทความ (`Post`) แต่ละอันมีผู้เขียน
(`Author`) และอยู่ในหมวดหมู่หนึ่ง (`Category`) พร้อมความคิดเห็น (`Comment`) ผูกอยู่ แล้วใช้โดเมนนี้
เจาะลึก **ActiveRecord Query Interface** ทั้งหมด ตั้งแต่กลไกพื้นฐานที่สุด (`ActiveRecord::Relation`
คืออะไร ทำไม query ถึง "ยังไม่ยิง" จนกว่าจะถึงเวลาจริง) ไปจนถึงเครื่องมือที่มืออาชีพใช้แก้ปัญหา
ประสิทธิภาพทุกวัน (`where` ทุกรูปแบบ, `joins`/`includes`/`preload`/`eager_load`, scope,
`pluck`/`select`, `find_each`) — ทุกหัวข้อมาพร้อม SQL log จริงที่ capture จากการรันจริง ไม่ใช่การ
เดาว่า Rails "น่าจะ" สร้าง SQL หน้าตาแบบไหน

## สารบัญของ Part นี้

- Step 331: `ActiveRecord::Relation` คืออะไร และ Lazy Query Evaluation
- Step 332: `where` เชิงลึก — hash conditions, array conditions, range, `not`, `.or`/`.and`
- Step 333: `order`, `limit`, `offset`, `distinct`
- Step 334: `joins` — INNER JOIN ที่ไม่โหลด column ของอีกตารางมาด้วย
- Step 335: `includes` vs `preload` vs `eager_load` และทำไม `references` ถึงจำเป็น
- Step 336: Named scope และการ chain scope เข้าด้วยกัน
- Step 337: `default_scope` และกับดักที่อันตรายที่สุดของมัน
- Step 338: Class method vs scope — เมื่อไหร่ควรใช้อะไร
- Step 339: `pluck`/`select` — โหลดเฉพาะคอลัมน์ที่ต้องใช้จริง
- Step 340: `find_each`/`find_in_batches` ประมวลผลตารางใหญ่แบบไม่ระเบิด memory + `explain` เบื้องต้น

---

## เตรียมโดเมนสำหรับ Part นี้: `Author`, `Category`, `Post`, `Comment`

ก่อนเข้า Step แรก มาตั้งโปรเจกต์ทดลองที่จะใช้ตลอดทั้ง Part นี้กันก่อน (ในโปรเจกต์จริงของคุณ ถ้า
ต่อยอดจาก Part 030 อยู่แล้ว ให้ปรับตามแนวทางเดียวกันนี้)

```bash
rails new query_demo --minimal
cd query_demo

bin/rails generate model Author name:string nationality:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string body:text published:boolean views:integer \
  published_at:datetime deleted_at:datetime author:references category:references
bin/rails generate model Comment commenter:string body:text post:references
```

แก้ migration ของ `Post` ให้ `published` และ `views` มีค่า default ที่ระดับฐานข้อมูล (แนวทางเดียว
กับที่เรียนใน Part 026 เรื่อง `null: false` คู่กับ default ที่เหมาะสม):

```ruby
# db/migrate/..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body
      t.boolean :published, default: false, null: false
      t.integer :views, default: 0, null: false
      t.datetime :published_at
      t.datetime :deleted_at
      t.references :author, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

ตั้งค่า association ให้ครบ:

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :posts
end
```

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :posts
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

```bash
bin/rails db:create db:migrate
```

Seed ข้อมูลตัวอย่าง — ผู้เขียน 4 คน, หมวดหมู่ 4 หมวด, บทความ 30 อัน (ประมาณ 2 ใน 3 เผยแพร่แล้ว),
และ comment กระจายไม่เท่ากันในแต่ละบทความ (ตั้งใจให้ไม่เท่ากัน เพื่อให้เห็นปัญหา N+1 ชัดเจนตอน
เข้า Step 335):

```ruby
# db/seeds.rb
authors = {
  "Nichada" => "Thai", "Somsak" => "Thai",
  "Wanda" => "American", "Kenji" => "Japanese"
}.map { |name, nat| Author.create!(name: name, nationality: nat) }

categories = %w[Ruby Rails DevOps Career].map { |n| Category.create!(name: n) }

titles = ["เริ่มต้นกับ Ruby", "Metaprogramming เบื้องต้น", # ... รวม 30 หัวข้อ
          ]

titles.each_with_index do |title, i|
  author = authors[i % authors.size]
  category = categories[i % categories.size]
  published = i % 3 != 0
  post = Post.create!(
    title: title, body: "เนื้อหาตัวอย่างของ #{title}",
    author: author, category: category, published: published,
    views: (i * 17) % 500,
    published_at: published ? i.days.ago : nil
  )
  rand(0..3).times { |c| post.comments.create!(commenter: "ผู้อ่าน #{c + 1}", body: "ความเห็น #{c + 1}") }
end
```

```bash
bin/rails db:seed
# Authors: 4
# Categories: 4
# Posts: 30 (published: 20)
# Comments: 47
```

ตลอด Part นี้ เราจะเปิด SQL log ออกมาดูตรงๆ ผ่าน `bin/rails runner script.rb` ด้วยการตั้ง
`ActiveRecord::Base.logger = Logger.new(STDOUT)` ที่หัวสคริปต์ทุกครั้ง (แทนที่จะเข้า console
ทีละคำสั่ง) เพื่อให้เห็น query ที่เกิดขึ้นแบบเรียงลำดับชัดเจน

---

## Step 331: `ActiveRecord::Relation` คืออะไร และ Lazy Query Evaluation

ทบทวนสั้นๆ จาก Part 025 Step 248: `Model.where(...)` **ไม่ได้คืนค่าเป็น Array** แต่คืนค่าเป็น
object ชนิด `ActiveRecord::Relation` — Step นี้จะพิสูจน์ให้เห็นว่าทำไม "การไม่เป็น Array" ถึงสำคัญ
มากในทางปฏิบัติ

```ruby
require "logger"
ActiveRecord::Base.logger = Logger.new(STDOUT)

relation = Post.where(published: true)
puts relation.class
# => Post::ActiveRecord_Relation
# (ไม่มี SQL log เกิดขึ้นเลยจากบรรทัดนี้!)
```

**ยังไม่มี query ใดๆ ถูกยิงไปที่ฐานข้อมูลเลย** ทั้งที่เขียน `.where` ไปแล้ว — นี่คือหลักการ
**Lazy Query Evaluation**: `ActiveRecord::Relation` เป็นแค่ "คำอธิบายว่าจะ query อะไร" ที่เก็บไว้ใน
หน่วยความจำ ยังไม่ใช่ผลลัพธ์จริง

### `.to_sql` — แปลงเป็น SQL string โดยไม่ยิง query จริง

```ruby
puts relation.to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
```

`.to_sql` มีประโยชน์มากตอน debug — เอาไว้ดูว่า Rails จะสร้าง SQL หน้าตาแบบไหนจาก method chain
ที่เขียนไว้ **โดยไม่ต้องรัน query จริงกับฐานข้อมูลเลย** ยิ่ง chain ต่อไปเรื่อยๆ ก็ยังไม่ query:

```ruby
relation2 = relation.order(views: :desc).limit(3)
puts relation2.to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE ORDER BY "posts"."views" DESC LIMIT 3
```

### จุดที่ query ถูกยิงจริง — "Trigger method"

query จะถูกยิงก็ต่อเมื่อ **ต้องการผลลัพธ์จริง** เท่านั้น เช่น `.to_a`, `.each`, `.map`, `.first`,
`.last`, `.count` (ในบางกรณี), หรือแม้แต่ `inspect`/`p` ตอน debug ใน console:

```ruby
relation2.to_a
```

```
Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE ORDER BY "posts"."views" DESC LIMIT 3
```

query เพิ่งเกิดขึ้น ณ จุดนี้เอง ไม่ใช่ตอนเขียน `.where`/`.order`/`.limit` ก่อนหน้า

### ผลลัพธ์ถูก cache ไว้ใน Relation object เดียวกัน

```ruby
relation2.to_a   # ครั้งแรก: มี query
relation2.each { |p| }   # ครั้งที่สอง: ไม่มี query ใหม่เกิดขึ้นเลย (ใช้ผลลัพธ์ที่ cache ไว้)
```

เหมือนหลักการ association caching ที่เรียนไปแล้วใน Part 027 Step 265 — พอ `relation2` ถูก
"execute" ครั้งแรกแล้ว ผลลัพธ์จะถูกเก็บไว้ใน object `relation2` เอง เรียกซ้ำกี่ครั้งก็ไม่ query
ซ้ำอีก **แต่ถ้าสร้าง Relation ใหม่จากมัน** (เช่นต่อเงื่อนไขเพิ่ม) จะได้ Relation คนละตัว ที่ยัง
ไม่เคย execute จึง query ใหม่เสมอ:

```ruby
relation3 = relation2.where("views > 0")   # ได้ Relation object ใหม่ทันที
relation3.to_a
```

```
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE AND (views > 0) ORDER BY "posts"."views" DESC LIMIT 3
```

> **ทำไมเรื่องนี้สำคัญในทางปฏิบัติ:** เพราะ Lazy Evaluation ทำให้เขียนโค้ดแบบ **"ประกอบเงื่อนไข
> ทีละขั้น"** ได้อย่างมีประสิทธิภาพ เช่นใน controller ที่ต้อง filter ตามหลาย parameter ที่ผู้ใช้
> เลือก (คำค้นหา, หมวดหมู่, ช่วงวันที่) — เขียน `if` เพิ่มเงื่อนไขเข้า Relation ทีละบรรทัดได้เรื่อยๆ
> โดยที่ **ยังไม่มี query เกิดขึ้นเลยจนกว่าจะถึงบรรทัดสุดท้ายที่ render view** ต่างจากภาษาที่ query
> ทันทีที่เขียนแบบ eager evaluation ซึ่งจะต้อง query ซ้ำทุกครั้งที่เพิ่มเงื่อนไข

```ruby
# ตัวอย่างการใช้ประโยชน์จาก lazy evaluation ใน controller
posts = Post.all
posts = posts.where(category_id: params[:category_id]) if params[:category_id].present?
posts = posts.where("title LIKE ?", "%#{params[:q]}%") if params[:q].present?
posts = posts.order(created_at: :desc)
# ยังไม่มี query จนกว่าจะถึงจุดที่ view เรียก posts.each หรือ posts.to_a
```

---

## Step 332: `where` เชิงลึก — hash conditions, array conditions, range, `not`, `.or`/`.and`

### Hash conditions — รูปแบบที่ใช้บ่อยที่สุด

```ruby
Post.where(published: true).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
```

ปลอดภัยจาก SQL injection โดยอัตโนมัติ (ค่าถูก escape ให้เสมอ) และอ่านง่ายที่สุด — ใช้เป็น
ตัวเลือกแรกเสมอเมื่อเงื่อนไขเป็นแค่ "column เท่ากับค่านี้"

### Array conditions — เมื่อต้องการ operator ที่ hash ทำไม่ได้ (`>`, `<`, `LIKE`, ฯลฯ)

```ruby
Post.where("views > ?", 100).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE (views > 100)
```

`?` เป็น placeholder ที่ Rails แทนที่ค่าให้แบบ parameterized query (ป้องกัน SQL injection เหมือน
hash condition) **ห้ามใช้ string interpolation ตรงๆ** แบบ `where("views > #{params[:min]}")`
เด็ดขาด เพราะเปิดช่องให้ SQL injection ทันทีถ้าค่านั้นมาจาก user input — นี่คือกฎความปลอดภัยข้อ
สำคัญที่สุดข้อหนึ่งของ ActiveRecord (จะเจาะลึกเรื่อง SQL injection เต็มๆ ใน Part 079)

ใช้ named placeholder ได้เมื่อมีหลายค่าและอยากให้อ่านง่ายขึ้น:

```ruby
Post.where("views > :min_views", min_views: 100).to_sql
# => SELECT "posts".* FROM "posts" WHERE (views > 100)
```

### Range conditions — แปลงเป็น `BETWEEN` อัตโนมัติ

```ruby
Post.where(views: 100..300).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."views" BETWEEN 100 AND 300
```

ใช้ range แบบ **exclusive end** (`...`) ได้ด้วย ผลลัพธ์ SQL จะเปลี่ยนจาก `BETWEEN` เป็น
`>= AND <` แทน (เพราะ SQL `BETWEEN` เป็น inclusive ทั้งสองด้านเสมอ ไม่มีรูปแบบ exclusive):

```ruby
Post.where(views: 100...300).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."views" >= 100 AND "posts"."views" < 300
```

Range ยังใช้กับ column ชนิดวันที่ได้เหมือนกัน (เช่น `Post.where(published_at: 1.week.ago..)`
สำหรับ "ตั้งแต่สัปดาห์ที่แล้วจนถึงตอนนี้" — range แบบไม่มีจุดสิ้นสุด (`beginless`/`endless range`)
ก็ใช้ได้ตั้งแต่ Ruby 2.6/2.7 เป็นต้นมา)

### `not` — ปฏิเสธเงื่อนไข

```ruby
Post.where.not(published: true).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" != TRUE
```

ใช้กับ array ได้ด้วย จะกลายเป็น `NOT IN`:

```ruby
category_ids = Category.where(name: %w[Ruby Rails]).pluck(:id)
Post.where.not(category_id: category_ids).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."category_id" NOT IN (1, 2)
```

### `.or` — รวมสองเงื่อนไขด้วย OR

```ruby
q1 = Post.where(published: true)
q2 = Post.where("views > 400")
q1.or(q2).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE ("posts"."published" = TRUE OR views > 400)
```

> **ข้อจำกัดของ `.or`:** ทั้งสอง Relation ที่เอามา `.or` กัน **ต้องมีโครงสร้างตรงกัน** ในส่วนของ
> `order`/`limit`/`offset`/`distinct` (คือมีหรือไม่มีเหมือนกันทั้งคู่) มิเช่นนั้นจะได้
> `ArgumentError: Relation passed to #or must be structurally compatible` — ข้อจำกัดนี้มีเพราะ
> Rails ไม่รู้ว่าจะรวม `LIMIT`/`ORDER BY` ที่ต่างกันของสอง Relation เข้าด้วยกันอย่างไรให้สมเหตุสมผล

### `.and` — รวมสองเงื่อนไขด้วย AND (Rails 6.1+)

```ruby
q3 = Post.where(published: true)
q4 = Post.where("views > 100")
q3.and(q4).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE AND (views > 100)
```

ในทางปฏิบัติ `.and` ใช้น้อยกว่า `.or` มาก เพราะการต่อ `.where` สองครั้งตรงๆ
(`Post.where(published: true).where("views > 100")`) ก็ได้ผลลัพธ์ SQL เหมือนกันทุกประการอยู่แล้ว
(ActiveRecord รวม `.where` หลายครั้งด้วย `AND` โดย default เป็นทุนเดิม) `.and` มีประโยชน์เฉพาะ
ตอนที่มี Relation สอง object แยกกันมาก่อนแล้ว (เช่นส่งมาจาก method คนละตัว) แล้วต้องการรวมเข้า
ด้วยกันแบบ AND อย่างชัดเจน

---

## Step 333: `order`, `limit`, `offset`, `distinct`

### `order`

```ruby
Post.order(:views).to_sql
# => SELECT "posts".* FROM "posts" ORDER BY "posts"."views" ASC

Post.order(views: :desc).to_sql
# => SELECT "posts".* FROM "posts" ORDER BY "posts"."views" DESC
```

เรียงหลายคอลัมน์พร้อมกันได้ (ลำดับที่เขียนคือลำดับความสำคัญ — เรียงตามตัวแรกก่อน ถ้าเท่ากันค่อย
เรียงตามตัวถัดไป):

```ruby
Post.order(category_id: :asc, views: :desc).to_sql
```

```
SELECT "posts".* FROM "posts" ORDER BY "posts"."category_id" ASC, "posts"."views" DESC
```

### `limit` และ `offset`

```ruby
Post.order(:id).limit(5).offset(10).to_sql
```

```
SELECT "posts".* FROM "posts" ORDER BY "posts"."id" ASC LIMIT 5 OFFSET 10
```

นี่คือกลไกพื้นฐานที่สุดของ **pagination** — `limit` กำหนดจำนวนต่อหน้า, `offset` กำหนดว่า "ข้าม
กี่แถวจากต้น" ก่อนเริ่มนับ (หน้า 3 ขนาด 5 แถวต่อหน้า = `limit(5).offset(10)`) เทคนิค pagination
แบบมืออาชีพที่ใช้ gem อย่าง Kaminari/Pagy จะเรียนใน **Part 038** — แต่ทุก gem เหล่านั้นก็สร้าง SQL
`LIMIT`/`OFFSET` แบบนี้อยู่เบื้องหลังเหมือนกันทั้งหมด

> **ข้อควรระวังเรื่อง performance:** `OFFSET` ที่มีค่าสูงมากๆ (เช่น `offset(100_000)`) จะช้าลง
> เรื่อยๆ ตามค่า offset เพราะฐานข้อมูลต้อง "นับข้าม" แถวก่อนหน้าทั้งหมดก่อนเริ่มอ่านจริง — สำหรับ
> ตารางขนาดใหญ่มากในโปรดักชัน มีเทคนิค **cursor-based pagination** (ใช้ `WHERE id > :last_id
> LIMIT :n` แทน) ที่เร็วกว่ามาก จะกลับมาพูดถึงอีกครั้งใน **Part 064** ตอนเรียนเรื่อง database
> performance เชิงลึก

### `distinct`

```ruby
Post.select(:category_id).distinct.to_sql
```

```
SELECT DISTINCT "posts"."category_id" FROM "posts"
```

`distinct` ใส่ `DISTINCT` เข้าไปใน SQL ตรงๆ — ลำดับการเขียน `.select(...).distinct` หรือ
`.distinct.select(...)` ให้ผลลัพธ์เหมือนกัน (ActiveRecord ประกอบ SQL ให้ถูกต้องไม่ว่าจะ chain
ลำดับไหน) มีประโยชน์มากตอนต้องการ "ค่าที่ไม่ซ้ำกัน" ของคอลัมน์ใดคอลัมน์หนึ่ง โดยไม่ต้องดึงทั้งแถว
มาแล้วมา `.uniq` เอาเองใน Ruby (ซึ่งจะโหลดข้อมูลซ้ำซ้อนที่ไม่จำเป็นเข้า memory ก่อนกรองทีหลัง —
ให้ฐานข้อมูลกรองให้เสมอเมื่อทำได้ เพราะเร็วกว่าและใช้ memory น้อยกว่ามาก)

---

## Step 334: `joins` — INNER JOIN ที่ไม่โหลด column ของอีกตารางมาด้วย

`joins` ใช้เมื่อต้องการ **กรองข้อมูลด้วยเงื่อนไขที่อยู่ในอีกตาราง** โดยไม่ได้ต้องการดึง column
ของตารางนั้นมาแสดงผลด้วย

```ruby
rel = Post.joins(:author).where(authors: { nationality: "Thai" })
puts rel.to_sql
```

```
SELECT "posts".* FROM "posts" INNER JOIN "authors" ON "authors"."id" = "posts"."author_id" WHERE "authors"."nationality" = 'Thai'
```

สังเกตสองจุดสำคัญ:

1. **`INNER JOIN`** — คืนเฉพาะ `Post` ที่มี `Author` จับคู่ได้จริงเท่านั้น (ถ้า `author_id` เป็น
   `NULL` หรือชี้ไปยัง author ที่ไม่มีอยู่จริง แถวนั้นจะหายไปจากผลลัพธ์ทันที)
2. **`SELECT "posts".*`** — ดึงมาแค่คอลัมน์ของ `posts` เท่านั้น **ไม่มีคอลัมน์ของ `authors`ติดมา
   เลย** ทั้งที่เพิ่ง join ตารางนั้นไป — `joins` ใช้ตาราง `authors` แค่เพื่อ "กรอง" ใน `WHERE`
   เท่านั้น

พิสูจน์ว่าไม่ได้ preload attribute มาด้วยจริง:

```ruby
posts = rel.to_a
posts.first.author   # เรียก association บน record ที่ได้จาก joins
```

```
Author Load (0.1ms)  SELECT "authors".* FROM "authors" WHERE "authors"."id" = 1 LIMIT 1
```

**ยิง query แยกใหม่ทันที** — นี่คือกับดักที่พบบ่อยมาก: หลายคนเข้าใจผิดว่า `joins` "โหลด
association มาให้ด้วย" เพราะเห็นคำว่า JOIN ใน SQL แต่จริงๆ แล้ว **`joins` ไม่ preload อะไรให้เลย**
ถ้าเอา record จาก `joins` ไป loop แล้วเรียก `post.author.name` จะเกิด N+1 เหมือนเดิมทุกประการ —
`joins` แก้ปัญหา "กรองข้ามตาราง" เท่านั้น ไม่ใช่เครื่องมือแก้ N+1

> **กฎการเลือกใช้:** ใช้ `joins` เมื่อ **แค่ต้องการกรอง** ด้วยเงื่อนไขข้ามตาราง โดยไม่ได้ต้องการ
> ข้อมูลของตารางนั้นไปแสดงผลต่อ (เช่น `Post.joins(:author).where(authors: {nationality: "Thai"})`
> เพื่อนับจำนวนหรือกรองเฉยๆ) — ถ้าต้องการทั้งกรองและเอาข้อมูลไปแสดงผลด้วย ต้องดู Step 335 เรื่อง
> `includes`/`eager_load`

---

## Step 335: `includes` vs `preload` vs `eager_load` และทำไม `references` ถึงจำเป็น

Part 027 แนะนำ `includes` ไปแบบผิวเผินแล้วว่าแก้ N+1 ได้ — Step นี้จะแกะดูว่า `includes` ทำงาน
อย่างไรจริงๆ เบื้องหลัง เทียบกับ `preload` และ `eager_load` ซึ่งเป็นน้องสองตัวที่ `includes` เลือก
ใช้งานให้โดยอัตโนมัติ

### `includes` แบบไม่มีเงื่อนไขอ้างถึงตารางที่ join

```ruby
Post.includes(:author).to_a
```

```
Post Load (0.1ms)  SELECT "posts".* FROM "posts"
Author Load (0.1ms)  SELECT "authors".* FROM "authors" WHERE "authors"."id" IN (1, 2, 3, 4)
```

**2 query แยกกัน** — ดึง `Post` ทั้งหมดก่อน แล้วดึง `Author` ทุกตัวที่เกี่ยวข้องในคำสั่งเดียวด้วย
`IN (...)` (เหมือนที่เห็นใน Part 027 Step 269 เป๊ะ) พฤติกรรมนี้เหมือนกับ `preload` ทุกประการ

### `includes` + เงื่อนไขแบบ hash บนตารางที่ join (Rails auto-detect ให้)

```ruby
Post.includes(:author).where(authors: { nationality: "Thai" }).to_a
```

```
Post Eager Load (0.2ms)  SELECT "posts"."id", "posts"."title", ..., "authors"."id", "authors"."name", "authors"."nationality", ...
  FROM "posts" LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
  WHERE "authors"."nationality" = 'Thai'
```

พฤติกรรมเปลี่ยนไปทันที — เหลือ **query เดียว** ที่เป็น `LEFT OUTER JOIN` และ `SELECT` คอลัมน์ของ
ทั้งสองตารางมาพร้อมกัน (สังเกตชื่อ log เปลี่ยนจาก `Post Load` เป็น **`Post Eager Load`**) — Rails
**ตรวจจับได้เองอัตโนมัติ** ว่า `where(authors: {...})` เป็นเงื่อนไขบนตาราง `authors` ที่ต้อง join
มา เพราะเขียนเป็น **hash ที่ใช้ชื่อตารางเป็น key**

### `includes` + เงื่อนไขแบบ string บนตารางที่ join (ต้องใส่ `references` เอง)

ถ้าเขียนเงื่อนไขเป็น raw SQL string แทน (เช่นตอนใช้ operator ที่ hash ทำไม่ได้) Rails
**parse string ไม่ออกว่าอ้างถึงตารางไหน** ต้องบอกเองด้วย `.references`:

```ruby
Post.includes(:author).where("authors.nationality = ?", "Thai").to_a
```

```
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE (authors.nationality = 'Thai')
```

```
ActiveRecord::StatementInvalid: SQLite3::SQLException: no such column: authors.nationality
```

**Error ทันที!** Rails ไม่รู้ว่าต้อง join ตาราง `authors` มาด้วย (เพราะไม่ได้ตรวจจับ raw SQL
string ให้อัตโนมัติ) เลยพยายามยิง query กับแค่ตาราง `posts` เดี่ยวๆ ที่ไม่มีคอลัมน์ `authors.
nationality` อยู่จริง แก้ได้ด้วยการเติม `.references(:author)`:

```ruby
Post.includes(:author).where("authors.nationality = ?", "Thai").references(:author).to_a
```

```
Post Eager Load (0.1ms)  SELECT "posts"."id", ..., "authors"."id", ...
  FROM "posts" LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
  WHERE (authors.nationality = 'Thai')
```

คราวนี้ได้ `LEFT OUTER JOIN` มาถูกต้อง — `.references(:table)` คือการบอก Rails ตรงๆ ว่า "เงื่อนไข
ที่เขียนอ้างถึงตารางนี้ด้วยนะ ต้อง join มา" จำเป็นทุกครั้งที่ใช้ **raw SQL string** ร่วมกับ
`includes` และเงื่อนไขนั้นอ้างถึงตารางของ association ที่ include ไว้

> **กฎจำง่าย:** เขียนเงื่อนไขเป็น **hash** (`where(authors: {...})`) → Rails auto-reference ให้
> เอง ไม่ต้องเติมอะไร | เขียนเป็น **string** (`where("authors.nationality = ?", ...)`) → ต้องเติม
> `.references(:author)` เองเสมอ ไม่งั้น error หรือแย่กว่านั้นคือได้ผลลัพธ์ผิดแบบเงียบๆ ในบาง
> เวอร์ชันของ Rails รุ่นเก่า (Rails 8.1 ที่ใช้ในหลักสูตรนี้ raise error ให้ทันทีซึ่งปลอดภัยกว่า)

### `preload` — บังคับแยก query เสมอ ไม่ว่าจะมีเงื่อนไขอะไรก็ตาม

```ruby
Post.preload(:author).to_a
```

```
Post Load (0.0ms)  SELECT "posts".* FROM "posts"
Author Load (0.0ms)  SELECT "authors".* FROM "authors" WHERE "authors"."id" IN (1, 2, 3, 4)
```

หน้าตาเหมือน `includes` แบบไม่มีเงื่อนไขทุกประการ เพราะ `includes` **เลือกใช้กลยุทธ์แบบเดียวกับ
`preload` เป็นค่า default** อยู่แล้วเมื่อไม่มีเหตุผลต้อง join แต่ `preload` มีข้อแตกต่างสำคัญ:
**`preload` ไม่มีทาง "อัปเกรด" เป็น JOIN ให้อัตโนมัติแบบ `includes`** ลองใส่เงื่อนไขบนตารางที่
preload ดู:

```ruby
Post.preload(:author).where("authors.nationality = ?", "Thai").to_a
```

```
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE (authors.nationality = 'Thai')
```

```
ActiveRecord::StatementInvalid: SQLite3::SQLException: no such column: authors.nationality
```

**Error เหมือนกับ `includes` ที่ไม่ใส่ `references`** — และที่สำคัญคือ **`preload` ไม่มี
`.references` ให้ใช้แก้ปัญหานี้เลย** เพราะ `preload` ถูกออกแบบมาให้ "แยก query เสมอ" โดยเจตนา
ไม่มี mode ที่ join ตารางเข้าด้วยกัน — ถ้าต้องการ filter ด้วยเงื่อนไขข้ามตาราง **ต้องใช้
`includes`/`eager_load` เท่านั้น** `preload` ใช้ได้แค่กรณี "preload attribute มาเฉยๆ ไม่มีเงื่อนไข
กรองข้ามตาราง"

### `eager_load` — บังคับ `LEFT OUTER JOIN` เสมอ ไม่ต้องมีเงื่อนไขก็ join

```ruby
Post.eager_load(:author).to_sql
```

```
SELECT "posts"."id", ..., "authors"."id", ...
FROM "posts" LEFT OUTER JOIN "authors" ON "authors"."id" = "posts"."author_id"
```

ต่างจาก `includes`/`preload` ตรงที่ **`eager_load` join เสมอไม่ว่าจะมีเงื่อนไขอ้างถึงตารางนั้น
หรือไม่ก็ตาม** — พูดอีกแบบคือ `includes(:author).where(authors: {...})` คือสิ่งที่ Rails "แปลงให้
เป็น" `eager_load` โดยอัตโนมัติเมื่อจำเป็นเท่านั้นเอง `eager_load` คือกลยุทธ์ที่ `includes` เลือก
ใช้เบื้องหลังเมื่อ Rails ตัดสินใจว่าต้อง join

### ตารางสรุปเปรียบเทียบทั้ง 4 ตัว

| | จำนวน query | รูปแบบ SQL | filter ตาม column ของ association ได้ไหม | ใช้เมื่อ |
|---|---|---|---|---|
| `joins` | 1 | `INNER JOIN` | ได้ (นี่คือจุดประสงค์หลัก) | กรองข้ามตารางอย่างเดียว ไม่ต้องการข้อมูลตารางนั้นไปแสดงผล |
| `includes` (ไม่มีเงื่อนไขข้ามตาราง) | 2 (แยกเหมือน `preload`) | `SELECT ... WHERE id IN (...)` | ไม่ (ใช้ `where(table: {...})` แล้วมันจะเปลี่ยนกลยุทธ์เป็น `eager_load` ให้อัตโนมัติ) | preload ข้อมูลไปแสดงผล ป้องกัน N+1 เป็นกรณีทั่วไป |
| `includes` + เงื่อนไขบนตาราง include | 1 | `LEFT OUTER JOIN` (auto-upgrade เป็น eager_load) | ได้ | ต้องการทั้งกรองข้ามตารางและ preload ไปแสดงผลด้วย |
| `preload` | 2 เสมอ | `SELECT ... WHERE id IN (...)` | **ไม่ได้เลย** (error ถ้าลอง) | ต้องการ preload แน่ๆ ไม่ต้องการให้ Rails เปลี่ยนกลยุทธ์เป็น JOIN ให้เอง (คุมพฤติกรรมชัดเจน) |
| `eager_load` | 1 เสมอ | `LEFT OUTER JOIN` เสมอ | ได้ | ต้องการบังคับ JOIN แน่นอน (บางกรณี `LEFT OUTER JOIN` เร็วกว่าสองรอบ query แยกจริงๆ) |

> **แนวทางปฏิบัติจริง:** ใช้ **`includes`** เป็นค่าเริ่มต้นเสมอ เพราะฉลาดพอที่จะเลือกกลยุทธ์ที่
> เหมาะสมให้อัตโนมัติทั้งสองแบบ — ใช้ `preload`/`eager_load` ตรงๆ เฉพาะตอนที่ **ต้องการควบคุม
> พฤติกรรมให้แน่นอน** เท่านั้น (เช่น ทีมมีกฎว่า query รายงานสำคัญต้องเป็น JOIN เดียวเสมอ ไม่ยอมให้
> เผลอกลายเป็น 2 query โดยไม่ตั้งใจ — ใช้ `eager_load` ล็อกไว้ตรงๆ)

---

## Step 336: Named scope และการ chain scope เข้าด้วยกัน

**Scope** คือวิธีตั้งชื่อ query condition ที่ใช้บ่อยๆ ให้เรียกใช้สั้นและอ่านง่ายขึ้น — เขียนด้วย
class method `scope` ที่รับชื่อ (symbol) กับ lambda ที่คืนค่าเป็น `ActiveRecord::Relation`:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy

  scope :published, -> { where(published: true) }
  scope :recent, -> { order(published_at: :desc) }
  scope :by_author, ->(author) { where(author: author) }
end
```

เรียกใช้เหมือน class method ธรรมดา:

```ruby
Post.published.to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
```

### Chain หลาย scope เข้าด้วยกัน

จุดเด่นที่สุดของ scope คือ **chain กันได้เหมือน `where`/`order` ปกติทุกประการ** เพราะแต่ละ scope
คืนค่าเป็น `ActiveRecord::Relation` เสมอ (ไม่ใช่ Array):

```ruby
author = Author.find_by(name: "Somsak")
Post.published.recent.by_author(author).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE AND "posts"."author_id" = 2 ORDER BY "posts"."published_at" DESC
```

Rails ประกอบเงื่อนไขจากทั้งสาม scope เข้าเป็น SQL เดียวให้อัตโนมัติ — อ่านออกเสียงเป็นภาษาธรรมชาติ
ได้ตรงตัวเลย: "โพสต์ที่เผยแพร่แล้ว เรียงล่าสุดก่อน ของผู้เขียนคนนี้"

### Scope รับ argument ได้ (เหมือน `by_author` ด้านบน) และ chain กับ `where` ปกติได้

```ruby
Post.published.where("views < ?", 400).to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE AND (views < 400)
```

scope ไม่ได้ "พิเศษ" ไปกว่า `where`/`order` เขียนตรงๆ เลยแม้แต่น้อย — มันคือแค่ **การตั้งชื่อให้
เรียกซ้ำได้สะดวก** เพื่อไม่ต้องเขียนเงื่อนไขเดิมซ้ำๆ ทุกที่ในโค้ด (หลักการ DRY ที่เจอมาตั้งแต่
Part 001)

### Scope เรียกจาก association ก็ได้เหมือนกัน

```ruby
author.posts.published.to_sql
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."author_id" = 1 AND "posts"."published" = TRUE
```

`author.posts` คืน `CollectionProxy` ที่มีพฤติกรรมเหมือน `ActiveRecord::Relation` (ทบทวนจาก
Part 027 Step 265) จึงเรียก scope ต่อได้ตามปกติ

---

## Step 337: `default_scope` และกับดักที่อันตรายที่สุดของมัน

`default_scope` คือเงื่อนไขที่ถูก**เติมเข้าไปในทุก query ของ Model นั้นโดยอัตโนมัติ** โดยไม่ต้อง
เรียก scope เอง — ฟังดูสะดวก แต่ในทางปฏิบัติ **เป็นฟีเจอร์ที่แนะนำให้หลีกเลี่ยงในเกือบทุกกรณี**
มาดูสถานการณ์คลาสสิกที่สุดที่คนมักเจอ: **soft delete**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  default_scope { where(deleted_at: nil) }
  # ... association และ scope อื่นๆ เหมือนเดิม
end
```

เจตนาคือ "ไม่อยากให้ query ปกติเจอ record ที่ถูก soft-delete แล้ว (มี `deleted_at` ตั้งค่าไว้)"
ลอง soft-delete โพสต์หนึ่งอัน:

```ruby
Post.count
# => 30

target = Post.first
target.update_column(:deleted_at, Time.current)

Post.count
```

```
Post Count (0.1ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."deleted_at" IS NULL
# => 29
```

ดูเผินๆ เหมือนทำงานถูกต้อง — `default_scope` ถูกเติมเข้าไปใน `Post.count` ให้อัตโนมัติจริง แต่
กับดักตัวแรกโผล่มาทันทีตรงนี้:

```ruby
Post.find(target.id)
```

```
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."deleted_at" IS NULL AND "posts"."id" = 1 LIMIT 1
ActiveRecord::RecordNotFound: Couldn't find Post with 'id'=1 [WHERE "posts"."deleted_at" IS NULL]
```

**`Post.find(target.id)` หา "ไม่เจอ" ทั้งที่ record ยังอยู่จริงในตารางเป๊ะๆ!** เพราะ `default_scope`
ถูกเติมเข้าไปใน `find` ด้วยเหมือนกัน (ไม่ได้แยกแค่ query ที่ผู้เขียนตั้งใจ) นี่คือปัญหาแรก: ทุก
query ของ Model นี้ **ไม่มีทางรู้ล่วงหน้าว่าจะโดน `default_scope` แอบเติมเงื่อนไขเข้าไปหรือเปล่า**
โดยที่โค้ดตรงจุดนั้นไม่ได้เขียนอะไรเกี่ยวกับ `deleted_at` เลยสักคำ

### กับดักที่อันตรายกว่า: หน้า admin ก็ถูกกรองไปด้วยเงียบๆ

```ruby
Post.pluck(:id).size
```

```
Post Pluck (0.1ms)  SELECT "posts"."id" FROM "posts" WHERE "posts"."deleted_at" IS NULL
# => 29
```

สมมติมีหน้า admin ที่ต้องการ "ดูโพสต์ทั้งหมดในระบบ รวมถึงที่ถูกลบไปแล้ว เพื่อกู้คืน" — โปรแกรมเมอร์
คนที่เขียนหน้า admin เขียน `Post.all` ตรงๆ ตามความเข้าใจปกติ **แต่ `default_scope` จะกรอง record
ที่ถูกลบออกให้เงียบๆ โดยไม่มีการเตือนใดๆ เลย** ผู้ดูแลระบบจะไม่มีทางรู้ว่ามีโพสต์ที่ถูกลบไปแล้ว
1 รายการซ่อนอยู่ ทั้งที่ต้องการเห็นมันในหน้านี้โดยเฉพาะ — บั๊กแบบนี้อันตรายเพราะ **ไม่มี error
ใดๆ เกิดขึ้นเลย ข้อมูลแค่ "หายไป" อย่างเงียบๆ**

### วิธีข้าม `default_scope`: `unscoped`

```ruby
Post.unscoped.count
```

```
Post Count (0.1ms)  SELECT COUNT(*) FROM "posts"
# => 30
```

```ruby
Post.unscoped.find(target.id)
```

```
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" = 1 LIMIT 1
# => เจอ record ปกติ ไม่มี default_scope ครอบแล้ว
```

`unscoped` ล้าง `default_scope` (และเงื่อนไข scope อื่นที่ chain มาก่อนหน้าทั้งหมด) ออกจาก
Relation — แต่นี่แหละคือปัญหาต่อมา: **ต้องจำให้ได้ทุกจุดในโค้ดว่าตรงไหน "ควร" ใช้ `unscoped`**
(เช่นหน้า admin, background job ที่ต้อง process ทุก record จริงๆ) ซึ่งเป็นภาระทางความคิดที่เพิ่ม
ขึ้นมาโดยไม่จำเป็น เมื่อเทียบกับการไม่มี `default_scope` เลยตั้งแต่แรก

> **ทำไม `default_scope` ถึงเป็นความคิดที่ไม่ดีในเกือบทุกกรณี:**
>
> 1. **ซ่อนพฤติกรรมของ query ไว้ในที่ที่มองไม่เห็น** — คนอ่านโค้ดที่เขียน `Post.where(...)`
>    เฉยๆ ไม่มีทางรู้ได้เลยว่ามีเงื่อนไขซ่อนอยู่ ต้องไปเปิดดู model file ก่อนถึงจะรู้
> 2. **ยากต่อการ override ให้ถูกต้องครบทุกจุด** — ลืม `unscoped` แม้แต่จุดเดียวในหน้า admin หรือ
>    background job ก็เกิดบั๊กข้อมูลหายแบบเงียบๆ ทันที
> 3. **ชนกับ `find`/`find_by` แม้จะมี primary key ตรงเป๊ะ** — ทำให้เกิด `RecordNotFound` ในที่ที่
>    ไม่ควรเกิด
> 4. **สร้างปัญหาซ้อนกับ association** — `post.comments` ที่ผ่าน `belongs_to`/`has_many` ก็โดน
>    `default_scope` ของอีกฝั่งกรองด้วยเช่นกัน ทำให้ debug ยากขึ้นไปอีกชั้น
>
> **ทางเลือกที่ดีกว่าเสมอ:** ใช้ **named scope ธรรมดา** (`scope :active, -> { where(deleted_at:
> nil) }`) แล้ว **เรียกมันตรงๆ ทุกจุดที่ต้องการ** (`Post.active.where(...)`) แทนที่จะให้มันทำงาน
> อัตโนมัติ — เขียนเพิ่มขึ้นมาแค่คำเดียว (`.active`) แต่แลกมาด้วยความชัดเจนที่มองเห็นได้ตรงๆ ใน
> ทุกจุดที่เรียกใช้ ไม่มีอะไรถูกซ่อนไว้เบื้องหลัง — ถ้าเจอ gem/legacy code ที่มี `default_scope`
> อยู่แล้ว ให้ระวังจุดที่ query "ควรเห็นข้อมูลทั้งหมด" เป็นพิเศษเสมอ และพิจารณา refactor ออกเมื่อ
> มีโอกาส

---

## Step 338: Class method vs scope — เมื่อไหร่ควรใช้อะไร

scope สะดวกก็จริง แต่มีข้อจำกัดสำคัญ: **lambda ของ scope ต้อง evaluate เป็น `ActiveRecord::
Relation` เสมอ** ทำให้เขียน logic ที่ซับซ้อนกว่านั้น (เช่น early return เมื่อ argument ว่าง)
ได้ไม่สะดวกเท่า class method ธรรมดา

```ruby
class Post
  scope :published, -> { where(published: true) }

  def self.search(keyword)
    return all if keyword.blank?
    where("title LIKE ? OR body LIKE ?", "%#{keyword}%", "%#{keyword}%")
  end
end
```

```ruby
Post.search("Ruby").to_sql
```

```
SELECT "posts".* FROM "posts" WHERE (title LIKE '%Ruby%' OR body LIKE '%Ruby%')
```

```ruby
Post.search(nil).to_sql
```

```
SELECT "posts".* FROM "posts"
```

`self.search` เขียนเป็น class method ธรรมดา ใช้ `return all if keyword.blank?` เพื่อ **ข้าม
เงื่อนไขทั้งหมดแล้วคืน Relation ที่ไม่มี filter** เมื่อไม่มีคำค้นหา — โค้ดแบบนี้เขียนเป็น `scope`
ตรงๆ ไม่ได้สะดวกเท่า เพราะ `scope :search, ->(keyword) { ... }` ที่มี `if/else`/`return` ข้างใน
lambda จะเริ่มอ่านยากและ debug ยากกว่า method ธรรมดาเมื่อ logic ซับซ้อนขึ้น

จุดสำคัญคือ **class method ที่คืนค่าเป็น Relation (ผ่าน `where`/`all`/scope อื่น) chain ต่อกับ
scope ได้ตามปกติทุกประการ** เพราะทั้งคู่คืนชนิดข้อมูลเดียวกัน:

```ruby
Post.search("Rails").published.to_sql
```

```
SELECT "posts".* FROM "posts" WHERE (title LIKE '%Rails%' OR body LIKE '%Rails%') AND "posts"."published" = TRUE
```

> **กฎการเลือกใช้ในทางปฏิบัติ:**
>
> | สถานการณ์ | ใช้ |
> |---|---|
> | เงื่อนไขคงที่ ไม่มี logic แตกแขนง (`where(published: true)`) | **scope** — สั้น อ่านง่าย |
> | เงื่อนไขที่รับ argument ธรรมดาแล้วส่งต่อ `where` ตรงๆ (`by_author(author)`) | **scope** |
> | ต้องมี early return / เงื่อนไขซับซ้อนกว่า one-liner / ต้องเรียก method อื่นประกอบกันหลาย
>   ขั้นตอนก่อนสร้าง query | **class method** (`def self.xxx`) |
> | ต้อง raise exception หรือทำ side effect อื่นนอกเหนือจากคืน Relation | **class method** เท่านั้น
>   (scope ไม่ควรมี side effect เพราะเรียกได้จากหลายที่โดยไม่รู้ตัว) |
>
> ทั้งสองแบบให้ผลลัพธ์เหมือนกันในแง่การ chain ต่อ — เลือกจาก **ความอ่านง่ายของโค้ดในกรณีนั้นๆ**
> เป็นหลัก ไม่ใช่กฎตายตัวว่าต้องใช้แบบใดแบบหนึ่งเสมอ

---

## Step 339: `pluck`/`select` — โหลดเฉพาะคอลัมน์ที่ต้องใช้จริง

Query ปกติ (`Post.all`, `Post.where(...)`) ดึงมา **ทุกคอลัมน์** และสร้างเป็น ActiveRecord Object
เต็มรูปแบบเสมอ (`SELECT "posts".*`) ซึ่งเปลืองทั้ง memory และเวลา query โดยไม่จำเป็นเมื่อต้องการ
แค่ 1-2 คอลัมน์ไปใช้งาน

### `select` — ยังได้ ActiveRecord Object แต่โหลดเฉพาะคอลัมน์ที่ระบุ

```ruby
posts = Post.select(:id, :title).limit(2)
puts posts.to_sql
posts.each { |p| puts "id=#{p.id} title=#{p.title}" }
```

```
SELECT "posts"."id", "posts"."title" FROM "posts" LIMIT 2
id=1 title=เริ่มต้นกับ Ruby
id=2 title=Metaprogramming เบื้องต้น
```

`p.id`/`p.title` เรียกได้ปกติเพราะถูก select มา แต่ลองเรียก attribute ที่ **ไม่ได้ select**:

```ruby
posts.first.views
```

```
ActiveModel::MissingAttributeError: missing attribute 'views' for Post
```

**Error ทันที** — นี่คือความแตกต่างสำคัญจาก `Post.all` ปกติที่ทุก attribute เข้าถึงได้เสมอ (เพราะ
select มาครบ) `select` เหมาะกับกรณีที่ยังต้องการ **ActiveRecord Object ที่ใช้ method ของ Model
ได้** (เช่น association, custom method) แต่รู้แน่ชัดว่าจะใช้แค่บาง attribute

### `pluck` — ไม่สร้าง ActiveRecord Object เลย ได้ Array ตรงๆ

```ruby
Post.where(published: true).pluck(:id, :title).first(3)
```

```
Post Pluck (0.1ms)  SELECT "posts"."id", "posts"."title" FROM "posts" WHERE "posts"."published" = TRUE
# => [[2, "Metaprogramming เบื้องต้น"], [3, "Block กับ Proc"], [5, "Association ขั้นสูง"]]
```

`pluck` ยิง SQL ที่ select เฉพาะคอลัมน์ที่ระบุเหมือนกับ `select` แต่ **ข้ามขั้นตอนการสร้าง
ActiveRecord Object ไปเลย** คืนเป็น `Array` ของค่าตรงๆ (หรือ `Array` ของ `Array` ถ้าระบุหลาย
คอลัมน์) — เร็วกว่าและใช้ memory น้อยกว่า `select` เมื่อไม่ต้องการเรียก method ของ Model ต่อ
เลยสักนิด

**กรณีใช้งานที่เจอบ่อยที่สุด: สร้าง dropdown list**

```ruby
options = Author.pluck(:name, :id)
```

```
Author Pluck (0.1ms)  SELECT "authors"."name", "authors"."id" FROM "authors"
# => [["Nichada", 1], ["Somsak", 2], ["Wanda", 3], ["Kenji", 4]]
```

รูปแบบ `[[label, value], ...]` นี้ป้อนเข้า `options_for_select`/`select_tag` ใน view ได้ตรงๆ
โดยไม่ต้องโหลด `Author` object เต็มรูปแบบมาทั้งหมด (ซึ่งจะเปลืองทั้ง query time และ memory
โดยไม่จำเป็น เมื่อสิ่งที่ view ต้องการมีแค่ `name` กับ `id` เท่านั้น) — เรื่อง form helper เต็มรูปแบบ
จะเจาะลึกใน **Part 031**

> **กฎการเลือกใช้:** ใช้ `pluck` เมื่อ **ต้องการแค่ค่าดิบๆ ไปใช้ต่อ** (dropdown, export CSV,
> ส่งเป็น JSON API, คำนวณสถิติ) — ใช้ `select` เมื่อ **ยังต้องการเรียก method ของ Model** (เช่น
> custom instance method, association) แต่อยากประหยัด memory ด้วยการไม่โหลดทุกคอลัมน์ — ใช้
> `Model.all`/`where` ปกติเมื่อ **ต้องการ attribute ครบทุกตัว** เช่นหน้า edit form ที่ต้องแสดง
> ทุกฟิลด์

---

## Step 340: `find_each`/`find_in_batches` ประมวลผลตารางใหญ่แบบไม่ระเบิด memory + `explain` เบื้องต้น

ทบทวนจาก Part 013 (Enumerable ขั้นสูง): `lazy` enumerator ช่วยประมวลผล collection ขนาดใหญ่โดยไม่
โหลดทุกตัวเข้า memory พร้อมกัน — `find_each`/`find_in_batches` คือแนวคิดเดียวกันที่ ActiveRecord
ออกแบบมาสำหรับ **ตารางฐานข้อมูลขนาดใหญ่** โดยเฉพาะ

### ปัญหาของ `Post.all.each`

```ruby
Post.all.each do |post|
  # ทำอะไรสักอย่างกับ post
end
```

`Post.all` ยิง query เดียวที่โหลด **ทุกแถวเข้า memory พร้อมกันทีเดียว** ก่อนเริ่ม loop — ถ้าตาราง
มี 10 แถวไม่มีปัญหา แต่ถ้ามี 10 ล้านแถว โปรแกรมจะกิน memory มหาศาลจนอาจทำให้ process ล่มได้ก่อนที่
loop จะเริ่มทำงานด้วยซ้ำ

### `find_each` — ประมวลผลทีละแถว แต่ query เป็น batch เบื้องหลัง

```ruby
count = 0
Post.find_each(batch_size: 10) do |post|
  count += 1
end
puts count
```

```
Post Load (0.2ms)  SELECT "posts".* FROM "posts" ORDER BY "posts"."id" ASC LIMIT 10
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" > 10 ORDER BY "posts"."id" ASC LIMIT 10
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" > 20 ORDER BY "posts"."id" ASC LIMIT 10
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" > 30 ORDER BY "posts"."id" ASC LIMIT 10
# => 30
```

จาก 30 แถวกับ `batch_size: 10` เกิด **4 query** (batch ละ 10 แถว จนกว่าจะหมด) — เทคนิคเบื้องหลัง
คือ **`WHERE id > :last_id ORDER BY id LIMIT :batch_size`** (คล้าย cursor-based pagination ที่พูด
ถึงใน Step 333) แทนที่จะ `OFFSET` ไปเรื่อยๆ ทำให้เร็วสม่ำเสมอไม่ว่าจะประมวลผลไปไกลแค่ไหนแล้ว
ในโค้ดของเรามองเห็นแค่ `|post|` ทีละตัวเหมือน `.each` ปกติ **แต่เบื้องหลัง ไม่มีตอนไหนเลยที่ทุกแถว
อยู่ใน memory พร้อมกันทั้งหมด** — มีแค่ batch ปัจจุบัน (10 แถว) เท่านั้นที่ถูกโหลดไว้ ณ เวลาใดเวลา
หนึ่ง

> **ข้อจำกัดสำคัญ: `find_each` ไม่รับประกันลำดับตาม `.order` ที่ตั้งเอง**
>
> ```ruby
> Post.order(views: :desc).find_each { |p| }
> ```
>
> ```
> WARN: Scoped order is ignored, use :cursor with :order to configure custom order.
> Post Load (0.2ms)  SELECT "posts".* FROM "posts" ORDER BY "posts"."id" ASC LIMIT 1000
> ```
>
> Rails **เพิกเฉยต่อ `.order(views: :desc))` ที่ตั้งไว้ทันที** พร้อม warning เตือน แล้วบังคับ
> เรียงตาม primary key (`id`) แทนเสมอ เพราะกลไก batch ของ `find_each` ต้องใช้คอลัมน์ที่มีลำดับ
> ต่อเนื่องและไม่ซ้ำ (โดย default คือ primary key) ในการคำนวณ "แถวถัดไป" — ถ้าต้องการลำดับอื่น
> จริงๆ ต้องใช้ option `:cursor` ระบุคอลัมน์ที่จะใช้ไล่ลำดับแทน (เหมาะกับตารางที่มี column เรียง
> ลำดับต่อเนื่องอื่น เช่น `created_at` ที่มี index) — ในทางปฏิบัติ **ถ้าต้องการลำดับที่แน่นอนสำหรับ
> การแสดงผล ให้ใช้ query ปกติกับ `limit`/`offset` แทน `find_each`** เพราะ `find_each` ออกแบบมา
> สำหรับ "ประมวลผลให้ครบทุกแถว" ไม่ใช่ "แสดงผลตามลำดับที่ผู้ใช้ต้องการ"

### `find_in_batches` — ได้ Array ทีละก้อน แทนที่จะเป็นทีละแถว

```ruby
Post.find_in_batches(batch_size: 12) do |batch|
  puts "batch: #{batch.size} รายการ, id ตั้งแต่ #{batch.first.id} ถึง #{batch.last.id}"
end
```

```
Post Load (0.1ms)  SELECT "posts".* FROM "posts" ORDER BY "posts"."id" ASC LIMIT 12
batch: 12 รายการ, id ตั้งแต่ 1 ถึง 12
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" > 12 ORDER BY "posts"."id" ASC LIMIT 12
batch: 12 รายการ, id ตั้งแต่ 13 ถึง 24
Post Load (0.1ms)  SELECT "posts".* FROM "posts" WHERE "posts"."id" > 24 ORDER BY "posts"."id" ASC LIMIT 12
batch: 6 รายการ, id ตั้งแต่ 25 ถึง 30
```

ใช้กลไก batch เดียวกับ `find_each` เป๊ะ ต่างกันแค่ block รับ **`Array` ของทั้ง batch** แทนที่จะ
รับทีละ record — มีประโยชน์เมื่อต้องการทำ operation แบบ bulk กับทั้ง batch พร้อมกัน เช่น
`Post.where(id: batch.map(&:id)).update_all(...)` หรือส่งทั้ง batch เข้า background job เดียวกัน
แทนที่จะสร้าง job แยกทีละแถว

> **เมื่อไหร่ใช้ตัวไหน:** `find_each` เมื่อ logic เป็นแบบ "ทำอะไรสักอย่างกับแต่ละแถว" (ส่งอีเมล
> ทีละคน, sync ข้อมูลทีละแถวไป API ภายนอก) — `find_in_batches` เมื่อ logic เป็นแบบ "ทำ operation
> กับทั้งกลุ่มพร้อมกันทีละ batch" (bulk update, ส่งทั้ง batch เข้า queue เดียว)

### `explain` — ดู query plan เบื้องต้น

```ruby
p Post.where(published: true).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
2|0|216|SCAN posts
```

`explain` ส่งคำสั่ง `EXPLAIN` ไปให้ฐานข้อมูลอธิบายว่า **จะดำเนินการ query นี้อย่างไร** — ผลลัพธ์
ข้างบนบอกว่า SQLite ใช้วิธี `SCAN posts` คือ **สแกนทั้งตารางทีละแถว** (ไม่มี index ให้ใช้กับเงื่อนไข
`published = TRUE`) เทียบกับ query ที่มี index รองรับ:

```ruby
p Post.joins(:author).where(authors: { nationality: "Thai" }).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" INNER JOIN "authors" ON "authors"."id" = "posts"."author_id" WHERE "authors"."nationality" = 'Thai'
4|0|216|SCAN authors
8|0|61|SEARCH posts USING INDEX index_posts_on_author_id (author_id=?)
```

สังเกตว่าฝั่ง `posts` ใช้ **`SEARCH ... USING INDEX index_posts_on_author_id`** (เร็วกว่า SCAN
มาก เพราะ `add_reference` ใน Part 027 Step 264 สร้าง index ให้อัตโนมัติ) ในขณะที่ฝั่ง `authors`
ยังเป็น `SCAN` เพราะ `nationality` ไม่มี index

> **ข้อควรระวังตอนเรียก `explain`:** ต้องใช้ `p` หรือ `.inspect` ไม่ใช่ `puts` ตรงๆ — เพราะ
> `.explain` คืนค่าเป็น `ActiveRecord::Relation::ExplainProxy` ที่ override เฉพาะ `#inspect` ให้
> รันคำสั่ง `EXPLAIN` จริงแล้วคืนข้อความ ในขณะที่ `#to_s` ปกติของมันยังเป็นค่า default ของ Ruby
> object (`puts` เรียก `#to_s` ไม่ใช่ `#inspect`) เป็นจุดสับสนเล็กๆ ที่พบได้บ่อยตอนลองใช้ `explain`
> ครั้งแรก
>
> Step นี้แนะนำ `explain` แค่เบื้องต้นให้รู้จักว่ามีเครื่องมือนี้อยู่และอ่านผลลัพธ์คร่าวๆ ได้ —
> การอ่าน query plan อย่างละเอียด (การเลือก index ให้เหมาะสม, `EXPLAIN ANALYZE` บน PostgreSQL,
> การตรวจจับ N+1 อัตโนมัติด้วย gem `bullet`) จะเจาะลึกเต็มรูปแบบใน **Part 064: Database
> performance**

---

## แบบฝึกหัด: สร้างชุด Scope ของ `Post`, เปรียบเทียบ `joins` vs `includes` จริง, และใช้ `pluck`/`find_each`

### โจทย์

ใช้โดเมน `Post`/`Author`/`Category`/`Comment` ที่ตั้งไว้ต้น Part นี้ ทำดังนี้:

1. เพิ่ม scope ให้ `Post` ครบ 3 ตัว: `published`, `recent`, `by_author(author)`
2. เขียนสคริปต์ที่ chain ทั้งสาม scope เข้าด้วยกัน แล้วดู SQL ที่ได้
3. เขียน query รายงาน "โพสต์ที่เผยแพร่แล้ว พร้อมชื่อผู้เขียนและชื่อหมวดหมู่" สองแบบ — แบบ `joins`
   กับแบบ `includes` — แล้วเปรียบเทียบจำนวน query จริงที่เกิดขึ้นเมื่อ loop แสดงผลจริง (ไม่ใช่แค่
   เทียบ `.to_sql`)
4. ใช้ `pluck` สร้างรายการหมวดหมู่แบบเบาที่สุดสำหรับ dropdown
5. ใช้ `find_each` ประมวลผลทุกโพสต์ในระบบโดยไม่โหลดทั้งหมดเข้า memory พร้อมกัน

### เฉลย

**1) เพิ่ม scope**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy

  scope :published, -> { where(published: true) }
  scope :recent, -> { order(published_at: :desc) }
  scope :by_author, ->(author) { where(author: author) }
end
```

**2) Chain scope**

```ruby
# rails runner exercise_scopes.rb
author = Author.find_by(name: "Somsak")
chained = Post.published.recent.by_author(author)
puts chained.to_sql
chained.each { |p| puts "- #{p.title} (published_at=#{p.published_at})" }
```

SQL ที่ได้จริง:

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE AND "posts"."author_id" = 2 ORDER BY "posts"."published_at" DESC
```

ผลลัพธ์ (6 บทความของ Somsak ที่เผยแพร่แล้ว เรียงล่าสุดก่อน):

```
- Metaprogramming เบื้องต้น (published_at=2026-09-25 03:35:00 UTC)
- Query Interface ที่ควรรู้ (published_at=2026-09-21 03:35:00 UTC)
- TDD Workflow (published_at=2026-09-13 03:35:00 UTC)
- Rate Limiting (published_at=2026-09-09 03:35:00 UTC)
- Caching กลยุทธ์ (published_at=2026-09-01 03:35:00 UTC)
- Multi-tenancy (published_at=2026-08-28 03:35:00 UTC)
```

**3) เปรียบเทียบ `joins` vs `includes` สำหรับ query รายงานจริง**

```ruby
# ===== แบบ joins =====
report_joins = Post.joins(:author, :category).where(published: true)
puts report_joins.to_sql
posts = report_joins.to_a
posts.each { |post| post.author.name; post.category.name }
```

```
SELECT "posts".* FROM "posts"
  INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"
  INNER JOIN "categories" ON "categories"."id" = "posts"."category_id"
  WHERE "posts"."published" = TRUE
```

```ruby
# ===== แบบ includes =====
report_includes = Post.includes(:author, :category).where(published: true)
puts report_includes.to_sql
posts2 = report_includes.to_a
posts2.each { |post| post.author.name; post.category.name }
```

```
SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
SELECT "authors".* FROM "authors" WHERE "authors"."id" IN (2, 3, 1, 4)
SELECT "categories".* FROM "categories" WHERE "categories"."id" IN (2, 3, 1, 4)
```

นับจำนวน query จริงด้วย `ActiveSupport::Notifications` (มี 20 โพสต์ที่เผยแพร่แล้วในข้อมูลตัวอย่าง):

```ruby
query_count = 0
ActiveSupport::Notifications.subscribe("sql.active_record") { query_count += 1 }

query_count = 0
Post.joins(:author, :category).where(published: true).to_a.each { |p| p.author.name; p.category.name }
puts "joins: #{query_count} queries"

query_count = 0
Post.includes(:author, :category).where(published: true).to_a.each { |p| p.author.name; p.category.name }
puts "includes: #{query_count} queries"
```

ผลลัพธ์จริง:

```
joins: 41 queries
includes: 3 queries
```

**อธิบายตัวเลข:** แบบ `joins` ยิง query แรกครั้งเดียวจริง (`INNER JOIN` ทั้งสองตาราง) แต่พอ loop
เรียก `post.author.name`/`post.category.name` ที่ **ไม่ได้ถูก preload มาด้วย** จะเกิด N+1 ทันที —
20 โพสต์ × 2 association (author + category) = 40 query แยก บวกกับ query แรก รวมเป็น **41 query**
ส่วนแบบ `includes` ใช้แค่ **3 query คงที่** (posts, authors, categories) ไม่ว่าจะมีโพสต์กี่ร้อยกี่
พันอันก็ตาม — นี่คือหลักฐานที่จับต้องได้ว่าทำไม `joins` ถึง **ไม่ใช่เครื่องมือแก้ N+1** อย่างที่
Step 334 อธิบายไว้ ในขณะที่ `includes` คือคำตอบที่ถูกต้องเมื่อต้องเอาข้อมูลของ association ไป
แสดงผลจริง

**4) `pluck` สำหรับ dropdown**

```ruby
category_options = Category.pluck(:name, :id)
puts category_options.inspect
```

```
SELECT "categories"."name", "categories"."id" FROM "categories"
# => [["Ruby", 1], ["Rails", 2], ["DevOps", 3], ["Career", 4]]
```

**5) `find_each` ประมวลผลทุกโพสต์**

```ruby
processed = 0
Post.find_each(batch_size: 10) { |post| processed += 1 }
puts "ประมวลผล Post ทั้งหมด #{processed} รายการ"
```

```
SELECT "posts".* FROM "posts" ORDER BY "posts"."id" ASC LIMIT 10
SELECT "posts".* FROM "posts" WHERE "posts"."id" > 10 ORDER BY "posts"."id" ASC LIMIT 10
SELECT "posts".* FROM "posts" WHERE "posts"."id" > 20 ORDER BY "posts"."id" ASC LIMIT 10
SELECT "posts".* FROM "posts" WHERE "posts"."id" > 30 ORDER BY "posts"."id" ASC LIMIT 10
ประมวลผล Post ทั้งหมด 30 รายการ
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม scope `commented` ที่คืนเฉพาะ `Post` ที่มี comment อย่างน้อยหนึ่งอัน โดยลองเขียนสองแบบ
   แล้วเทียบ SQL ที่ได้: แบบแรกใช้ `joins(:comments).distinct` (สังเกตว่าทำไมต้องมี `.distinct`
   ด้วย — ลองเอาออกแล้วดูว่าเกิดอะไรขึ้นถ้าโพสต์หนึ่งมีหลาย comment) แบบที่สองใช้
   `where(id: Comment.select(:post_id))` (subquery) — ผลลัพธ์ตรงกันไหม และ SQL ต่างกันอย่างไร
2. เขียน class method `Post.report_by_category` ที่คืนจำนวนโพสต์ที่เผยแพร่แล้วของแต่ละหมวดหมู่
   (ใบ้: `Post.published.group(:category_id).count` — `group`/`having` จะเจาะลึกเต็มรูปแบบใน
   Part หลังๆ แต่ลองสำรวจดูตอนนี้ก่อนได้) แล้วลองใช้ `includes(:category)` ประกอบกับผลลัพธ์เพื่อ
   แสดงชื่อหมวดหมู่แทน id
3. ทดลองสร้างสถานการณ์เหมือน Step 337 (`default_scope` soft-delete) แล้วเขียนสคริปต์ที่ใช้
   `find_in_batches` วน `Post.unscoped` ทั้งหมดเพื่อ "ตรวจสอบและรายงาน" ว่ามีกี่รายการที่ถูก
   soft-delete ไปแล้ว (`deleted_at` ไม่เป็น `nil`) โดยไม่โดน `default_scope` กรองออกจากการตรวจสอบ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **`ActiveRecord::Relation`** ไม่ใช่ Array — เป็น "คำอธิบายของ query" ที่ **lazy evaluation**
  (ไม่ query จริงจนกว่าจะเรียก `.to_a`/`.each`/`.first` ฯลฯ) ทำให้ chain เงื่อนไขทีละขั้นในโค้ด
  จริง (เช่น controller ที่ filter ตาม parameter) ได้อย่างมีประสิทธิภาพ ส่วน `.to_sql` ใช้ดู SQL
  ล่วงหน้าได้โดยไม่ query จริง
- **`where`** มีหลายรูปแบบ: hash conditions (ปลอดภัยและอ่านง่ายที่สุด), array conditions
  (`"col > ?", value` — ต้องใช้ placeholder เสมอ ห้าม interpolate ตรงๆ), range (`100..300` แปลง
  เป็น `BETWEEN`), `.not`, และรวมหลายเงื่อนไขด้วย `.or`/`.and`
- **`order`/`limit`/`offset`/`distinct`** คือกลไกพื้นฐานของ sorting, pagination, และการดึงค่าไม่
  ซ้ำ — `OFFSET` ที่มีค่าสูงมากช้าลงเรื่อยๆ ตามขนาดตาราง (แก้ด้วย cursor-based pagination ใน
  Part 064)
- **`joins`** สร้าง `INNER JOIN` เพื่อ **กรอง** ข้ามตารางเท่านั้น **ไม่ preload attribute ของ
  association มาด้วย** — ถ้าเข้าถึง association หลัง `joins` จะเกิด N+1 เหมือนเดิม
- **`includes`** ฉลาดเลือกกลยุทธ์ให้อัตโนมัติ: แยก query แบบ `preload` เมื่อไม่มีเงื่อนไขข้าม
  ตาราง หรือกลายเป็น `LEFT OUTER JOIN` แบบ `eager_load` เมื่อมีเงื่อนไข — เงื่อนไขแบบ **hash**
  (`where(table: {...})`) ถูก auto-reference ให้ แต่เงื่อนไขแบบ **string** ต้องเติม
  **`.references(:table)`** เองเสมอ ไม่งั้น error `no such column`
- **`preload`** บังคับแยก query เสมอ ไม่มีทาง filter ข้ามตารางได้เลย — **`eager_load`** บังคับ
  `LEFT OUTER JOIN` เสมอไม่ว่าจะมีเงื่อนไขหรือไม่ — ใช้ `includes` เป็นค่าเริ่มต้น ใช้อีกสองตัว
  เฉพาะตอนต้องการควบคุมพฤติกรรมให้แน่นอน
- **Named scope** (`scope :name, -> { ... }`) คืน Relation เสมอ จึง **chain กันได้ไม่จำกัด**
  (`Post.published.recent.by_author(x)`) — **`default_scope`** ฟังดูสะดวกแต่เป็นกับดักอันตราย
  เพราะซ่อนเงื่อนไขไว้ในทุก query โดยไม่มีใครเห็น รวมถึงหน้า admin ที่อาจต้องการเห็นข้อมูลทั้งหมด
  — ใช้ named scope ธรรมดาเรียกตรงๆ แทนเสมอ
- **Class method vs scope**: ใช้ scope กับเงื่อนไขง่ายๆ ที่ไม่มี logic แตกแขนง ใช้ class method
  ธรรมดา (`def self.xxx`) เมื่อต้องการ early return หรือ logic ที่ซับซ้อนกว่านั้น — ทั้งสองแบบ
  chain ต่อกันได้เพราะคืน Relation เหมือนกัน
- **`pluck`** คืน Array ของค่าดิบโดยไม่สร้าง ActiveRecord Object เลย (เร็วและประหยัด memory ที่สุด
  สำหรับ dropdown/export) ส่วน **`select`** ยังได้ ActiveRecord Object แต่โหลดเฉพาะคอลัมน์ที่ระบุ
  (เรียก attribute ที่ไม่ได้ select จะได้ `ActiveModel::MissingAttributeError`)
- **`find_each`/`find_in_batches`** ประมวลผลตารางขนาดใหญ่โดยแบ่งเป็น batch คำสั่ง SQL หลายรอบ
  แทนที่จะโหลดทุกแถวเข้า memory พร้อมกันทีเดียว (เหมือนหลักการ `lazy` enumerator จาก Part 013
  แต่ทำงานที่ระดับฐานข้อมูล) — ไม่รับประกันลำดับตาม `.order` ที่ตั้งเอง (เรียงตาม primary key
  เสมอ เว้นแต่ระบุ `:cursor`) — **`explain`** ให้ดู query plan เบื้องต้นได้ (ต้องใช้ `p`/`.inspect`
  ไม่ใช่ `puts`) รายละเอียดเชิงลึกเรื่อง performance รอใน Part 064

**ต่อไป (Part 035):** ตอนนี้เรารู้จัก query interface ครบทุกมุมแล้ว Part ถัดไปจะกลับไปที่วงจรชีวิต
ของ ActiveRecord object เอง — **Callback ขั้นสูง** ตั้งแต่ `before_validation` จนถึง
`after_commit`/`after_rollback` พร้อม **lifecycle diagram เต็มรูปแบบ** ที่แสดงลำดับการทำงานของทุก
callback เมื่อ `save`/`update`/`destroy` ถูกเรียก และข้อควรระวังที่มือใหม่มักพลาด เช่น การใช้
callback ทำ side effect ที่ไม่ควรอยู่ใน Model (เช่นส่งอีเมล) และปัญหาที่เกิดเมื่อ callback หนึ่งไป
กระตุ้น callback ของ record อื่นเป็นทอดๆ โดยไม่ตั้งใจ
