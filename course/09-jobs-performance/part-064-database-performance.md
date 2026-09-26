# Part 064: Database Performance เชิงลึก — Index, EXPLAIN, และ N+1 Detection ด้วย Bullet

> **Step ครอบคลุมใน Part นี้:** Step 631–640
> **ระดับ:** สูง (ต้องผ่าน Part 034 เรื่อง Query Interface มาก่อน โดยเฉพาะ Step 335 เรื่อง
> `includes`/`preload`/`eager_load` และ Step 340 ที่แนะนำ `explain` แบบผิวเผินไว้)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทุกคำสั่งทดสอบจริงบน Ruby 3.3.6 + Rails 8.1.4 +
> `bullet` 8.2.0 บนฐานข้อมูล SQLite3 กับตารางขนาด **1,000,000 แถวจริง** — และ benchmark ที่
> เกี่ยวกับ `EXPLAIN ANALYZE` เปรียบเทียบเพิ่มเติมด้วย **PostgreSQL 16 ของจริง** บนตารางขนาด
> เดียวกัน เพื่อให้เห็นความแตกต่างระหว่างสองฐานข้อมูลอย่างตรงไปตรงมา)

Part 034 Step 340 ทิ้งท้ายไว้ว่า "การอ่าน query plan อย่างละเอียด, การเลือก index ให้เหมาะสม,
`EXPLAIN ANALYZE` บน PostgreSQL, และการตรวจจับ N+1 อัตโนมัติด้วย gem `bullet`" จะเจาะลึกเต็ม
รูปแบบใน Part 064 — ถึงเวลานั้นแล้ว

Part นี้จะสานต่อโดเมน `Author`/`Category`/`Post`/`Comment` เดิมจาก Part 034 แต่ครั้งนี้เราจะ
**scale ข้อมูลขึ้นไปถึง 1 ล้านแถว** เพราะบทเรียนเรื่อง index บนตารางเล็กๆ (30 แถวแบบ Part 034)
มองไม่เห็นผลต่างอะไรเลย — index จะมีความหมายก็ต่อเมื่อตารางใหญ่พอที่ "การอ่านทุกแถว" กับ
"การอ่านเฉพาะแถวที่ต้องการ" ต่างกันจริงๆ ในทางปฏิบัติ ทุกตัวเลขในเอกสารนี้ (เวลา query, ขนาดไฟล์
บนดิสก์, ข้อความ warning ของ Bullet) คือค่าที่รันได้จริงและ capture มาโดยตรง ไม่มีค่าที่แต่งขึ้น

## สารบัญของ Part นี้

- Step 631: Index ทำงานอย่างไร (B-Tree) และวัดผลต่างจริงบนตาราง 1,000,000 แถว
- Step 632: เพิ่ม Index ผ่าน Migration — Single-column และ Composite Index (ลำดับคอลัมน์สำคัญ)
- Step 633: คอลัมน์ไหนที่ควรมี Index — Foreign Key, `WHERE`/`ORDER BY`/`JOIN`
- Step 634: Unique Index กับความถูกต้องของข้อมูลระดับฐานข้อมูล (ทบทวน Part 032)
- Step 635: ข้อเสียของการทำ Index มากเกินไป (Over-indexing)
- Step 636: อ่าน `EXPLAIN`/`.explain` อย่างละเอียด + เทียบกับ PostgreSQL `EXPLAIN ANALYZE` ของจริง
- Step 637: ติดตั้ง Bullet และจับ N+1 Query จริงแบบสด
- Step 638: ความสามารถอื่นของ Bullet — Unused Eager Loading, Counter Cache Suggestion
- Step 639: เรื่องจริงของ Index ที่ "ไม่ช่วยอะไรเลย" — Cardinality ต่ำ และการตัดสินใจของ Query
  Planner
- Step 640: `strict_loading` — เซฟตี้เน็ตของ Rails เองที่ทำงานคู่กับ Bullet

---

## เตรียมโดเมนสำหรับ Part นี้: สานต่อ `Author`/`Category`/`Post`/`Comment` แต่ scale เป็น 1 ล้านแถว

ถ้าคุณต่อยอดจากโปรเจกต์ `query_demo` ของ Part 034 อยู่แล้ว ให้เพิ่มคอลัมน์และ gem ตามด้านล่างนี้
เข้าไป — ถ้าเริ่มใหม่ ให้สร้างโปรเจกต์และโมเดลตามนี้:

```bash
rails new perf_demo --minimal
cd perf_demo

bin/rails generate model Author name:string nationality:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string slug:string body:text published:boolean \
  views:integer published_at:datetime author:references category:references
bin/rails generate model Comment commenter:string body:text post:references
```

แก้ migration ของ `Post` ให้มี default ที่ระดับฐานข้อมูล (แนวทางเดียวกับ Part 026/034):

```ruby
# db/migrate/..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.string :slug
      t.text :body
      t.boolean :published, default: false, null: false
      t.integer :views, default: 0, null: false
      t.datetime :published_at
      t.references :author, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

ตั้งค่า association ให้ครบ (เหมือน Part 034):

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
  has_many :comments, dependent: :destroy
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

เพิ่ม gem `bullet` ใน group `:development` (เดี๋ยวจะใช้ตั้งแต่ Step 637 เป็นต้นไป):

```ruby
# Gemfile
group :development do
  gem "bullet"
end
```

```bash
bundle install
bin/rails db:create db:migrate
```

### Seed ข้อมูลจริง 1,000,000 แถว

ต่างจาก Part 034 ที่ seed แค่ 30 บทความ Part นี้ต้องการตารางที่ **ใหญ่พอจะเห็นผลของ index จริงๆ**
เราจะ seed บทความ **1,000,000 อัน** ผู้เขียน 20 คน และหมวดหมู่ 8 หมวด — ใช้ `insert_all` เป็น
batch ละ 10,000 แถวแทนการเรียก `Post.create!` ทีละอัน (ซึ่งช้าเกินไปสำหรับข้อมูลระดับล้านแถว —
นี่คือหลักการเดียวกับ `find_in_batches` ที่เรียนใน Part 034 Step 340 แต่กลับด้าน: แบ่ง batch ตอน
**เขียน** แทนที่จะเป็นตอน**อ่าน**):

```ruby
# db/seeds.rb
puts "Seeding..."

conn = ActiveRecord::Base.connection
conn.execute("PRAGMA synchronous = OFF")   # เร่งความเร็วตอน seed เท่านั้น (ไม่ใช้ใน production)
conn.execute("PRAGMA journal_mode = MEMORY")

Comment.delete_all
Post.delete_all
Category.delete_all
Author.delete_all

authors = 20.times.map { |i| Author.create!(name: "Author #{i}", nationality: %w[Thai American Japanese].sample) }
categories = %w[Ruby Rails DevOps Career Database Testing Frontend Security].map { |n| Category.create!(name: n) }

TOTAL_POSTS = 1_000_000
now = Time.current

Post.transaction do
  (TOTAL_POSTS / 10_000).times do |batch_i|
    rows = 10_000.times.map do |j|
      i = batch_i * 10_000 + j
      author = authors[i % authors.size]
      category = categories[i % categories.size]
      published = i % 3 != 0
      {
        title: "Post number #{i}",
        slug: "post-number-#{i}",
        body: "Body content for post #{i}",
        published: published,
        views: rand(0..500_000),
        published_at: published ? (now - rand(1..900).days) : nil,
        author_id: author.id,
        category_id: category.id,
        created_at: now,
        updated_at: now
      }
    end
    Post.insert_all(rows)
  end
end

puts "Posts: #{Post.count}"

# comments: กระจายไม่เท่ากัน เฉพาะ 200 โพสต์แรก (พอสำหรับสาธิต N+1 ในบทเรียนนี้)
Post.order(:id).limit(200).find_each do |post|
  rand(0..5).times { |c| Comment.create!(post: post, commenter: "Reader #{c + 1}", body: "Comment #{c + 1}") }
end

puts "Comments: #{Comment.count}"
```

```bash
bin/rails db:seed
```

```
Seeding...
Posts: 1000000
Comments: 483
```

รันจริงใช้เวลาประมาณ **2 นาที** บนเครื่องทดสอบ (`real 2m0.656s`) — เป็นเรื่องปกติสำหรับข้อมูล
ระดับล้านแถว ไม่ต้องตกใจถ้ารอนานกว่าการ seed ทั่วไปที่เคยเจอในหลักสูตรนี้

---

## Step 631: Index ทำงานอย่างไร (B-Tree) และวัดผลต่างจริงบนตาราง 1,000,000 แถว

### แนวคิด: Index คือ "สารบัญท้ายเล่ม" ของตาราง

นึกภาพหนังสือหนา 1,000,000 หน้าที่ไม่มีสารบัญ — ถ้าอยากหาว่าคำว่า "Rails" พูดถึงตรงหน้าไหนบ้าง
วิธีเดียวที่ทำได้คือ **เปิดอ่านทีละหน้าตั้งแต่หน้าแรกจนจบเล่ม** นี่คือสิ่งที่ฐานข้อมูลทำเมื่อไม่มี
index — เรียกว่า **full table scan** (SQLite เรียกขั้นตอนนี้ว่า `SCAN`, PostgreSQL/MySQL เรียกว่า
`Seq Scan`)

**Index** คือโครงสร้างข้อมูลแยกต่างหาก (ส่วนใหญ่เป็น **B-Tree** — ต้นไม้ที่สมดุลและเรียงลำดับ)
ที่เก็บคู่ "ค่าของคอลัมน์ → ตำแหน่งแถวจริงในตาราง" เรียงลำดับไว้ล่วงหน้า ทำให้ฐานข้อมูล**กระโดด
ตรงไปยังตำแหน่งที่ต้องการได้ทันที** โดยไม่ต้องไล่อ่านทุกแถว — เหมือนสารบัญท้ายเล่มที่บอกเลขหน้า
ตรงๆ ความซับซ้อนเปลี่ยนจาก **O(n)** (ต้องอ่านทุกแถว) เป็น **O(log n)** (เดินลงต้นไม้ B-Tree
ไม่กี่ระดับ) ซึ่งสำหรับตาราง 1,000,000 แถว หมายความว่าแทนที่จะเทียบค่า 1,000,000 ครั้ง อาจใช้แค่
ประมาณ 20 ครั้ง (log₂ 1,000,000 ≈ 20)

```
ไม่มี Index (Full Table Scan)          มี Index (B-Tree Search)
┌─────────────────────┐               ┌─────┐
│ row 1: views=88231   │  อ่านทุกแถว   │ 250K│  ระดับ 1
│ row 2: views=4102    │  ทีละแถว      └──┬──┘
│ row 3: views=304381  │  จนกว่าจะเจอ  ┌───┴───┐
│ ...                  │  (หรือจบตาราง)│100K│400K│  ระดับ 2
│ row 1,000,000        │               └─┬─┘ └─┬─┘
└─────────────────────┘                 ...   ... → ตำแหน่งแถวจริง
```

### วัดผลจริง: query บนคอลัมน์ที่ยังไม่มี index

ตาราง `posts` ของเรามี 1,000,000 แถว คอลัมน์ `views` เป็น `integer` ธรรมดาที่**ยังไม่มี index**
ลอง query หาโพสต์ที่ `views` อยู่ในช่วงแคบๆ (matched ประมาณ 500-600 แถวจาก 1 ล้านแถว):

```ruby
p Post.where(views: 100_000..100_300).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."views" BETWEEN 100000 AND 100300
2|0|216|SCAN posts
```

`SCAN posts` ยืนยันว่า SQLite ต้องไล่อ่านทั้ง 1,000,000 แถวเพื่อตรวจสอบทีละแถวว่าตรงเงื่อนไข
หรือไม่ — ไม่ว่าผลลัพธ์ที่ตรงเงื่อนไขจะมีกี่แถวก็ตาม (ในที่นี้ตรง 578 แถว) **ต้นทุนของ full scan
คงที่ตามขนาดตาราง ไม่ใช่ตามขนาดผลลัพธ์**

### วัดเวลาจริงแบบ cold cache (จำลองสภาพการอ่านจากดิสก์แบบ production)

เวลา query แบบปกติที่รันซ้ำๆ ในเครื่องเดียวกันมักเร็วผิดปกติ เพราะ OS และ SQLite cache หน้า
(page) ของไฟล์ไว้ใน RAM ตั้งแต่ครั้งแรกที่อ่าน ทำให้การวัดผลครั้งต่อๆ ไปไม่สะท้อนความจริงของ
ฐานข้อมูลขนาดใหญ่ใน production ที่ข้อมูลส่วนใหญ่ไม่ได้อยู่ใน cache ตลอดเวลา เทคนิคที่ใช้วัดผล
อย่างยุติธรรมคือ **สั่งล้าง OS page cache ก่อนวัดทุกครั้ง** (`echo 3 > /proc/sys/vm/drop_caches`
บน Linux ที่มีสิทธิ์ root — ใช้ได้เฉพาะเครื่องทดสอบ ห้ามทำบน production):

```ruby
require "benchmark"

def drop_os_cache
  system("sync")
  File.write("/proc/sys/vm/drop_caches", "3")
end

conn = ActiveRecord::Base.connection
conn.execute("DROP INDEX IF EXISTS index_posts_on_views")

ActiveRecord::Base.connection_pool.disconnect!
drop_os_cache
t1 = Benchmark.realtime { Post.where(views: 100_000..100_300).to_a }
puts "cold scan time: #{(t1 * 1000).round(2)} ms"
```

ผลลัพธ์จริง (รันสองรอบแยกกันเพื่อให้เห็นว่าค่ามี variance ตามสภาพ cache ของเครื่อง แต่ทิศทาง
เดียวกันเสมอ):

```
รอบที่ 1: cold scan time: 13.28 ms
รอบที่ 2: cold scan time: 53.3 ms
```

### เพิ่ม Index แล้ววัดซ้ำ

```ruby
conn.execute("CREATE INDEX index_posts_on_views ON posts (views)")

p Post.where(views: 100_000..100_300).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."views" BETWEEN 100000 AND 100300
3|0|163|SEARCH posts USING INDEX index_posts_on_views (views>? AND views<?)
```

`SCAN` เปลี่ยนเป็น **`SEARCH ... USING INDEX`** — หลักฐานที่ระดับ query plan ว่า SQLite ใช้
index จริง วัดเวลาซ้ำแบบ cold cache เดียวกัน:

```
รอบที่ 1: cold index-search time: 2.67 ms   (จาก 13.28 ms → เร็วขึ้น 5.0 เท่า)
รอบที่ 2: cold index-search time: 3.12 ms   (จาก 53.3 ms  → เร็วขึ้น 17.1 เท่า)
```

> **ทำไมตัวเลขไม่นิ่ง แต่ทิศทางเดียวกันเสมอ:** เวลาที่วัดได้ผันผวนตามสภาพของเครื่อง (background
> process อื่น, การจัดสรร memory ของ OS ในขณะนั้น) — นี่คือความจริงของการวัด performance ที่ต้อง
> ยอมรับเสมอ ไม่มีตัวเลขเดียวที่ "ถูกต้องที่สุด" สิ่งที่สำคัญกว่าตัวเลขเดี่ยวๆ คือ **ทิศทางที่
> สม่ำเสมอ** (index เร็วกว่าเสมอในทุกรอบที่วัด) และ **หลักฐานเชิงโครงสร้าง** จาก `EXPLAIN` (`SCAN`
> vs `SEARCH`) ที่ไม่ขึ้นกับความผันผวนของเครื่อง — ควรดูทั้งสองอย่างประกอบกันเสมอ ไม่ใช่เชื่อ
> ตัวเลขเวลาเพียงค่าเดียว

---

## Step 632: เพิ่ม Index ผ่าน Migration — Single-column และ Composite Index

### Index คอลัมน์เดียว

```bash
bin/rails generate migration AddIndexToPostsViews
```

```ruby
# db/migrate/..._add_index_to_posts_views.rb
class AddIndexToPostsViews < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, :views
  end
end
```

```bash
bin/rails db:migrate
```

```
== AddIndexToPostsViews: migrating ============================================
-- add_index(:posts, :views)
   -> 0.0015s
== AddIndexToPostsViews: migrated (0.0015s) ===================================
```

นี่คือ syntax เดียวกับที่ใช้เพิ่ม index ให้ foreign key แบบ manual เช่น `add_index :posts,
:author_id` — แม้ในกรณีนี้จะไม่จำเป็น เพราะ `t.references :author` สร้าง index ให้อัตโนมัติ
อยู่แล้ว (รายละเอียดเต็มใน Step 633) แต่ syntax นี้จำเป็นเมื่อคุณเพิ่ม foreign-key-like column
ด้วย `t.integer :author_id` ตรงๆ โดยไม่ผ่าน `t.references` (เช่น legacy migration เก่าที่เขียน
ก่อนจะรู้จัก `t.references`)

### Composite Index — Index จากหลายคอลัมน์รวมกัน

สถานการณ์จริงที่พบบ่อยมาก: หน้า "บทความล่าสุดในหมวดหมู่นี้" ต้อง **กรองตาม `category_id`
พร้อมกับเรียงตาม `published_at`**:

```ruby
Post.where(category_id: 3).order(published_at: :desc).limit(20).explain
```

ก่อนมี composite index (มีแค่ index เดี่ยวบน `category_id` จาก `t.references`):

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."category_id" = 3
  ORDER BY "posts"."published_at" DESC LIMIT 20
5|0|61|SEARCH posts USING INDEX index_posts_on_category_id (category_id=?)
29|0|0|USE TEMP B-TREE FOR ORDER BY
```

สังเกตบรรทัดที่สอง: **`USE TEMP B-TREE FOR ORDER BY`** — SQLite ใช้ index หา `category_id`
ได้เร็วก็จริง แต่พอต้องเรียงตาม `published_at` มันต้องสร้าง **B-Tree ชั่วคราว** ขึ้นมาใหม่ในหน่วย
ความจำเพื่อ sort ผลลัพธ์ที่กรองมาได้ — เป็นงานเพิ่มที่ไม่จำเป็นถ้ามี index ที่ครอบคลุมทั้งสอง
คอลัมน์

เพิ่ม composite index ที่ตรงกับ pattern การ query นี้เป๊ะๆ:

```bash
bin/rails generate migration AddCompositeIndexToPosts
```

```ruby
class AddCompositeIndexToPosts < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, [:category_id, :published_at]
  end
end
```

```ruby
Post.where(category_id: 3).order(published_at: :desc).limit(20).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."category_id" = 3
  ORDER BY "posts"."published_at" DESC LIMIT 20
5|0|61|SEARCH posts USING INDEX index_posts_on_category_id_and_published_at (category_id=?)
```

**`USE TEMP B-TREE FOR ORDER BY` หายไปทันที** — เพราะ index แบบ composite เก็บข้อมูลเรียงตาม
`category_id` ก่อน แล้วภายใน `category_id` เดียวกันเรียงตาม `published_at` ต่ออีกชั้น ทำให้
ฐานข้อมูล**อ่านผลลัพธ์ที่เรียงมาให้พร้อมแล้วจาก index โดยตรง** ไม่ต้อง sort เพิ่มเอง

### ลำดับคอลัมน์ใน Composite Index สำคัญมาก (Leftmost-Prefix Rule)

ลองสลับลำดับคอลัมน์ดู — สร้าง index แบบ `[:published_at, :category_id]` แทน (สลับกับของเดิม):

```ruby
conn.execute("DROP INDEX index_posts_on_category_id_and_published_at")
conn.execute("CREATE INDEX index_posts_on_published_at_and_category_id ON posts (published_at, category_id)")

Post.where(category_id: 3).order(published_at: :desc).limit(20).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."category_id" = 3
  ORDER BY "posts"."published_at" DESC LIMIT 20
5|0|61|SEARCH posts USING INDEX index_posts_on_category_id (category_id=?)
29|0|0|USE TEMP B-TREE FOR ORDER BY
```

**Query plan กลับไปเหมือนตอนไม่มี composite index เลย!** SQLite เพิกเฉยต่อ
`index_posts_on_published_at_and_category_id` ที่เพิ่งสร้างไปโดยสิ้นเชิง แล้วใช้ index เดี่ยว
บน `category_id` แทน พร้อม `USE TEMP B-TREE FOR ORDER BY` กลับมาเหมือนเดิม

**เหตุผล:** composite index ทำงานเหมือนสมุดโทรศัพท์ที่เรียงตาม "นามสกุล แล้วค่อยชื่อ" — ถ้าอยาก
หาทุกคนที่นามสกุล "สมิท" สมุดเล่มนี้ช่วยได้ดีมาก (ไปหาช่วง "สมิท" แล้วไล่อ่านทุกชื่อในช่วงนั้น
ได้เลย) แต่ถ้าอยากหาทุกคนที่ **ชื่อ** ขึ้นต้นด้วย "จอห์น" ไม่ว่านามสกุลอะไร สมุดเล่มนี้**ช่วย
อะไรไม่ได้เลย** เพราะข้อมูลไม่ได้เรียงตามชื่อเป็นหลัก ต้องไล่อ่านทั้งเล่ม — นี่คือ **Leftmost-Prefix
Rule**: composite index `[A, B]` ใช้ได้ดีกับเงื่อนไขที่เริ่มจาก `A` (หรือ `A` อย่างเดียว) แต่ใช้
กรอง/เรียงตาม `B` อย่างเดียวโดยไม่มี `A` แทบไม่ได้ผลเลย

```ruby
# กู้กลับมาเป็น composite index ที่ถูกต้อง
conn.execute("DROP INDEX index_posts_on_published_at_and_category_id")
conn.execute("CREATE INDEX index_posts_on_category_id_and_published_at ON posts (category_id, published_at)")
```

> **กฎการเรียงคอลัมน์ใน composite index:** วางคอลัมน์ที่ใช้ใน **equality (`=`)** ไว้ก่อน
> (`category_id`) แล้วตามด้วยคอลัมน์ที่ใช้ **`ORDER BY`/`range` (`>`, `<`, `BETWEEN`)** ไว้หลัง
> (`published_at`) — ถ้าเขียนสลับกัน index จะช่วยได้แค่บางส่วนหรือไม่ช่วยเลยตามที่พิสูจน์ไปข้างต้น

---

## Step 633: คอลัมน์ไหนที่ควรมี Index

### 1) Foreign Key — Rails สร้าง Index ให้อัตโนมัติอยู่แล้ว (ถ้าใช้ `t.references`)

ทบทวนจาก Part 027 Step 264: `t.references :author` ไม่ได้แค่สร้างคอลัมน์ `author_id` แต่ยัง
เพิ่ม **`foreign_key: true`** และ **index ให้อัตโนมัติ** ด้วย — พิสูจน์ได้จาก `db/schema.rb`
ที่ Rails generate ให้เองหลัง migrate:

```ruby
# db/schema.rb (ส่วนที่เกี่ยวข้อง)
create_table "posts", force: :cascade do |t|
  # ...
  t.integer "author_id", null: false
  t.integer "category_id", null: false
  t.index ["author_id"], name: "index_posts_on_author_id"
  t.index ["category_id"], name: "index_posts_on_category_id"
end

create_table "comments", force: :cascade do |t|
  t.integer "post_id", null: false
  t.index ["post_id"], name: "index_comments_on_post_id"
end
```

**ทำไมต้องมี index บน foreign key เสมอ:** ทุกครั้งที่เรียก `author.posts` (ผ่าน `has_many`) Rails
สร้าง SQL `WHERE author_id = ?` — ถ้าไม่มี index บน `author_id` การเรียก association นี้จะเป็น
full table scan ทุกครั้ง แม้จะเป็นการ query ที่ดูเรียบง่ายที่สุดก็ตาม นี่คือเหตุผลที่ Rails
ตัดสินใจสร้าง index นี้ให้อัตโนมัติตั้งแต่ต้น — ถ้าคุณเพิ่มคอลัมน์ที่ทำหน้าที่เป็น foreign key
โดยไม่ผ่าน `t.references` (เช่น `t.integer :author_id` ตรงๆ) **ต้องเพิ่ม `add_index` เอง**
มิเช่นนั้นจะพลาด index ที่สำคัญที่สุดตัวหนึ่งไปโดยไม่รู้ตัว

### 2) คอลัมน์ที่ใช้ใน `WHERE` บ่อยๆ

เหมือน `views` ใน Step 631 — คอลัมน์ไหนก็ตามที่ปรากฏใน `.where(...)` เป็นประจำ (โดยเฉพาะใน
endpoint หรือหน้าเว็บที่มีคนเข้าบ่อย) เป็นตัวเลือกแรกที่ควรพิจารณาเพิ่ม index

### 3) คอลัมน์ที่ใช้ใน `ORDER BY` บ่อยๆ

แม้จะไม่มี `WHERE` เลย การ `ORDER BY` คอลัมน์ที่ไม่มี index บนตารางใหญ่ก็ต้องสร้าง temp B-Tree
สำหรับ sort ทั้งตาราง (เหมือนที่เห็นใน Step 632) — ถ้าหน้าแรกของเว็บต้องแสดง "บทความล่าสุด"
เสมอ (`order(created_at: :desc)`) และตารางมีข้อมูลนับแสนนับล้านแถว ควรมี index บน `created_at`

### 4) คอลัมน์ที่ใช้ใน `JOIN` (นอกเหนือจาก foreign key ที่มี index อยู่แล้ว)

ทบทวนจาก Part 034 Step 334: `Post.joins(:author).where(authors: { nationality: "Thai" })`
JOIN ผ่าน `author_id` (มี index อยู่แล้ว) แต่กรองด้วย `authors.nationality` (ไม่มี index) — ถ้า
ตาราง `authors` มีข้อมูลมากและ query แบบนี้ถูกเรียกบ่อย ควรพิจารณาเพิ่ม index บน `nationality`
ของตาราง `authors` ด้วย ไม่ใช่แค่ตาราง `posts` ฝั่งเดียว

> **หลักคิดโดยรวม:** ก่อนเพิ่ม index ให้ถามตัวเองว่า "คอลัมน์นี้ปรากฏใน `WHERE`/`ORDER BY`/`JOIN`
> ของ query ที่รันบ่อยแค่ไหน และตารางนี้มีข้อมูลเยอะแค่ไหน" — index มีประโยชน์มากที่สุดกับคอลัมน์
> ที่ query บ่อยบนตารางใหญ่ ส่วนตารางเล็ก (หลักร้อยแถว) แทบไม่ได้ประโยชน์อะไรจาก index เลย
> (รายละเอียดว่าทำไมถึง "แทบไม่ได้ประโยชน์" แม้จะมี index จะเจาะลึกใน Step 639)

---

## Step 634: Unique Index กับความถูกต้องของข้อมูลระดับฐานข้อมูล

ทบทวนจาก Part 032 Step 317: `uniqueness` validator ของ Rails ทำงานที่ระดับ Ruby ล้วนๆ (ยิง
`SELECT` ธรรมดาไปเช็คก่อน) จึงมีช่องโหว่จาก **race condition** ได้เสมอ — การรับประกันที่แท้จริง
ต้องมาจาก **unique index ระดับฐานข้อมูล** เท่านั้น เพราะฐานข้อมูลตรวจสอบและปฏิเสธการเขียนที่ซ้ำ
กันแบบ **atomic** (ไม่มีช่วงเวลาที่สองคำสั่งแทรกพร้อมกันแล้วหลุดผ่านทั้งคู่)

Part นี้เพิ่มคอลัมน์ `slug` ให้ `Post` (ใช้เป็น permalink) — เพิ่ม unique index ให้:

```bash
bin/rails generate migration AddUniqueIndexToPostsSlug
```

```ruby
class AddUniqueIndexToPostsSlug < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, :slug, unique: true
  end
end
```

```bash
bin/rails db:migrate
```

ลองสร้างโพสต์ที่มี `slug` ซ้ำกับที่มีอยู่แล้ว:

```ruby
author = Author.first
category = Category.first

begin
  Post.create!(title: "Dup", slug: "post-number-5", body: "x", author: author, category: category)
rescue => e
  puts "#{e.class}: #{e.message}"
end
```

```
ActiveRecord::RecordNotUnique: SQLite3::ConstraintException: UNIQUE constraint failed: posts.slug
```

`ActiveRecord::RecordNotUnique` เหมือนที่พิสูจน์ไว้ใน Part 032 เป๊ะๆ — ไม่ว่าจะใช้ SQLite หรือ
PostgreSQL (ซึ่งจะได้ `PG::UniqueViolation` ห่อด้วย class กลางเดียวกันนี้) ในโค้ด production
ควร `rescue ActiveRecord::RecordNotUnique` แล้วแปลงเป็น error message ที่เป็นมิตรกับผู้ใช้ตาม
รูปแบบที่ Part 032 Step 317 สอนไว้ — คู่กับ `validates :slug, uniqueness: true` ที่ฝั่ง Ruby
เพื่อ UX ที่ดี **ทั้งสองชั้นต้องมีคู่กันเสมอ**: validator ให้ประสบการณ์ผู้ใช้ที่ดี unique index
ให้การรับประกันที่แท้จริง

> **ทำไม unique index ถึงถูกจัดอยู่ในบทเรียนเรื่อง performance:** เพราะ unique index ไม่ได้ให้
> แค่ความถูกต้องของข้อมูล — มันเป็น **B-Tree index ธรรมดา** ที่ได้ประโยชน์ด้าน query speed
> เหมือน index ทั่วไปทุกประการด้วย (`Post.find_by(slug: "...")` เร็วขึ้นเหมือนที่พิสูจน์ใน
> Step 631) — การเพิ่ม `unique: true` คือ "index ปกติ + กฎบังคับไม่ให้ซ้ำ" ไม่ใช่ฟีเจอร์คนละตัว
> กับ index ทั่วไป

---

## Step 635: ข้อเสียของการทำ Index มากเกินไป (Over-indexing)

Index ไม่ใช่ของฟรี — มันมีต้นทุนสองด้านที่มักถูกมองข้าม: **การเขียนช้าลง** และ **พื้นที่ดิสก์ที่
เพิ่มขึ้น** เพราะทุกครั้งที่ `INSERT`/`UPDATE`/`DELETE` แถวหนึ่ง ฐานข้อมูลต้องอัปเดต **ทุก index**
ที่มีอยู่บนตารางนั้นให้ตรงกันด้วย ไม่ใช่แค่ตารางหลัก

### วัดผลจริง: Insert 50,000 แถวเข้าตารางที่มี Index ต่างกัน

สร้างตารางทดสอบสองตารางที่มีโครงสร้างคอลัมน์เหมือนกันทุกประการ ต่างกันแค่จำนวน index:

```ruby
require "benchmark"
conn = ActiveRecord::Base.connection

conn.execute(<<~SQL)
  CREATE TABLE few_idx_demo (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    col_a INTEGER, col_b INTEGER, col_c TEXT, col_d TEXT, col_e INTEGER, col_f TEXT
  )
SQL

conn.execute(<<~SQL)
  CREATE TABLE many_idx_demo (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    col_a INTEGER, col_b INTEGER, col_c TEXT, col_d TEXT, col_e INTEGER, col_f TEXT
  )
SQL

%w[col_a col_b col_c col_d col_e col_f].each do |col|
  conn.execute("CREATE INDEX idx_many_#{col} ON many_idx_demo (#{col})")
end
```

`few_idx_demo` มีแค่ primary key (index เดียวที่หลีกเลี่ยงไม่ได้) `many_idx_demo` มี index เพิ่ม
อีก **6 ตัว** (หนึ่งตัวต่อหนึ่งคอลัมน์) — insert ข้อมูลชุดเดียวกัน 50,000 แถวเข้าทั้งสองตาราง
แล้ววัดเวลา:

```
Insert 50000 rows, table with ONLY primary key: 16089.5 ms
Insert 50000 rows, table with 6 extra indexes:  17052.3 ms
Slowdown factor: 1.06x
```

**ช้าลงประมาณ 6-7%** จากการมี index เพิ่มอีก 6 ตัว — ตัวเลขนี้ดูไม่มากในตอนแรก แต่ถ้าตารางนั้น
เป็นตารางที่ระบบเขียนบ่อยมาก (เช่น log, event tracking, การนับ view) ผลกระทบสะสมจะชัดเจนขึ้น
เรื่อยๆ ตามปริมาณการเขียน

### วัดพื้นที่ดิสก์จริงที่ Index ใช้ (ผ่าน `dbstat`)

```ruby
result = conn.select_all(<<~SQL)
  SELECT name, SUM(pgsize) as bytes FROM dbstat
  WHERE name = 'few_idx_demo' OR name = 'many_idx_demo' OR name LIKE 'idx_many_%'
  GROUP BY name
SQL
result.each { |row| puts row }
```

```
{"name"=>"few_idx_demo", "bytes"=>2293760}
{"name"=>"idx_many_col_a", "bytes"=>614400}
{"name"=>"idx_many_col_b", "bytes"=>569344}
{"name"=>"idx_many_col_c", "bytes"=>1011712}
{"name"=>"idx_many_col_d", "bytes"=>962560}
{"name"=>"idx_many_col_e", "bytes"=>561152}
{"name"=>"idx_many_col_f", "bytes"=>839680}
{"name"=>"many_idx_demo", "bytes"=>2293760}
```

ตารางข้อมูลจริง (`many_idx_demo`) ใช้พื้นที่ **2,293,760 bytes (~2.19 MB)** แต่ **index ทั้ง 6
ตัวรวมกันใช้พื้นที่ถึง 4,558,848 bytes (~4.35 MB)** — เกือบ **2 เท่า** ของขนาดข้อมูลจริงในตาราง
เอง! นี่คือหลักฐานที่จับต้องได้ว่า **index กินพื้นที่ดิสก์ได้มากกว่าข้อมูลจริงด้วยซ้ำ** ถ้าเพิ่ม
เกินความจำเป็น

> **แนวทางปฏิบัติจริง:** อย่าเพิ่ม index ให้ทุกคอลัมน์ "เผื่อไว้" — เพิ่มเฉพาะคอลัมน์ที่มีหลักฐาน
> จริงว่าถูกใช้ใน `WHERE`/`ORDER BY`/`JOIN` บ่อยจากการอ่าน slow query log หรือ APM (Application
> Performance Monitoring) จริง (Part 065 จะพูดถึงเครื่องมือ profiling ที่ช่วยหา query ที่ควร
> optimize) ตารางที่เขียนบ่อยมาก (write-heavy) ต้องพิจารณาให้รอบคอบเป็นพิเศษว่า index แต่ละตัว
> คุ้มกับต้นทุนการเขียนที่เพิ่มขึ้นหรือไม่ — index ที่สร้างไว้แต่ไม่มี query ไหนใช้เลยคือต้นทุน
> ล้วนๆ โดยไม่ได้ประโยชน์อะไรตอบแทน

---

## Step 636: อ่าน `EXPLAIN`/`.explain` อย่างละเอียด + เทียบกับ PostgreSQL `EXPLAIN ANALYZE`

### ทบทวนและขยายความจาก Part 034: `EXPLAIN QUERY PLAN` ของ SQLite

Part 034 Step 340 แนะนำ `.explain` ไว้แค่ผิวเผิน — สรุปรูปแบบผลลัพธ์ที่ SQLite คืนมาอย่างเป็น
ระบบ:

```
2|0|216|SCAN posts
```

อ่านเป็นคอลัมน์: `id|parent|notused|detail` — ส่วนที่สำคัญที่สุดคือคอลัมน์สุดท้าย (`detail`)
ซึ่งมีคำสำคัญอยู่ 2 แบบหลักที่ต้องจำ:

| คำสำคัญ | ความหมาย | ควรระวังเมื่อ |
|---|---|---|
| `SCAN <table>` | เต็มตาราง ไล่อ่านทุกแถว | ตารางมีข้อมูลเยอะและมีเงื่อนไข `WHERE` แคบ |
| `SEARCH <table> USING INDEX <name>` | ใช้ index หาแบบ B-Tree | ปกติเร็วกว่า `SCAN` (แต่ไม่เสมอไป — ดู Step 639) |
| `USE TEMP B-TREE FOR ORDER BY` | ต้องสร้าง B-Tree ชั่วคราวเพื่อ sort ผลลัพธ์ | `ORDER BY` คอลัมน์ที่ไม่มี index รองรับ |

ตัวเลขตัวที่ 3 (เช่น `216` ใน `SCAN posts`, `61` ใน `SEARCH ... `) คือ**ค่าประมาณต้นทุน** ที่
SQLite optimizer ใช้เปรียบเทียบแผนการ query ต่างๆ กันเอง **ไม่ใช่หน่วยเวลาจริง** (ms หรือวินาที)
— ใช้เปรียบเทียบ "แผนไหนถูกกว่าแผนไหน" ในสายตาของ optimizer เท่านั้น ไม่ควรตีความเป็นเวลาจริง

### ข้อจำกัดของ `EXPLAIN QUERY PLAN` แบบ SQLite: ไม่มีเวลาจริง ไม่มีจำนวนแถวจริง

จุดอ่อนที่สำคัญของ `EXPLAIN QUERY PLAN` แบบ SQLite คือ **มันบอกแค่ "แผนการ" ไม่ได้รันจริงและ
ไม่บอกว่าใช้เวลาจริงเท่าไหร่ หรือจับคู่ได้กี่แถวจริง** — เป็นการ "ทำนาย" ไม่ใช่ "การวัดผลจริง"
ถ้าต้องการเวลาจริงต้องวัดด้วย `Benchmark`/`ActiveSupport::Notifications` แยกต่างหากตามที่ทำใน
Step 631

### PostgreSQL `EXPLAIN ANALYZE` — รันจริงและบอกเวลาจริง

โปรเจกต์ production ส่วนใหญ่ใช้ **PostgreSQL** ไม่ใช่ SQLite ซึ่ง PostgreSQL มีคำสั่งที่ทรงพลัง
กว่ามาก: **`EXPLAIN ANALYZE`** — ต่างจาก `EXPLAIN QUERY PLAN` ของ SQLite ตรงที่ **`ANALYZE`
สั่งให้รัน query จริงแล้วรายงานเวลาที่ใช้จริงและจำนวนแถวจริงที่พบ** กลับมาด้วย (ทดสอบจริงบน
PostgreSQL 16 กับตาราง `posts` ขนาด 1,000,000 แถวชุดเดียวกัน โครงสร้างเทียบเท่ากับที่ใช้ใน SQLite):

```sql
EXPLAIN ANALYZE SELECT * FROM posts WHERE views BETWEEN 100000 AND 100300;
```

**ก่อนมี index** (ผลลัพธ์จริงจาก PostgreSQL 16):

```
 Gather  (cost=1000.00..19558.60 rows=606 width=61) (actual time=0.494..36.217 rows=603 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on posts  (cost=0.00..18498.00 rows=252 width=61) (actual time=0.359..26.196 rows=201 loops=3)
         Filter: ((views >= 100000) AND (views <= 100300))
         Rows Removed by Filter: 333132
 Planning Time: 0.394 ms
 Execution Time: 36.297 ms
```

**`Seq Scan`** คือชื่อเรียกของ PostgreSQL สำหรับสิ่งที่ SQLite เรียกว่า `SCAN` (full table scan)
— ในที่นี้ PostgreSQL ยังฉลาดพอที่จะแบ่งงานสแกนออกเป็น **`Parallel Seq Scan`** โดยใช้ 2 worker
process ช่วยกันสแกนพร้อมกัน (ความสามารถที่ SQLite ไม่มี เพราะ SQLite เป็น embedded database
แบบ single-process) — สังเกต **`actual time=0.494..36.217`** และ **`Execution Time: 36.297
ms`** ซึ่งเป็น **เวลาที่รันจริง** ไม่ใช่ค่าประมาณเหมือน SQLite

หลังเพิ่ม `CREATE INDEX index_posts_on_views ON posts (views);` แล้วรันซ้ำ:

```
 Bitmap Heap Scan on posts  (cost=14.64..2001.27 rows=606 width=61) (actual time=0.178..2.612 rows=603 loops=1)
   Recheck Cond: ((views >= 100000) AND (views <= 100300))
   Heap Blocks: exact=588
   ->  Bitmap Index Scan on index_posts_on_views  (cost=0.00..14.49 rows=606 width=0) (actual time=0.111..0.112 rows=603 loops=1)
         Index Cond: ((views >= 100000) AND (views <= 100300))
 Planning Time: 0.526 ms
 Execution Time: 2.718 ms
```

จาก **36.297 ms → 2.718 ms** — เร็วขึ้นประมาณ **13 เท่า** วัดจากเวลารันจริงล้วนๆ ไม่ใช่การประมาณ
`Bitmap Heap Scan` + `Bitmap Index Scan` เป็นหนึ่งในรูปแบบของ index scan ที่ PostgreSQL เลือก
ใช้เมื่อผลลัพธ์ตรงเงื่อนไขมีจำนวนพอสมควร (ไม่น้อยจนใช้ index scan ตรงๆ ได้เลย แต่ก็ไม่มากจนต้อง
สแกนเต็มตาราง) — สำหรับ query ที่ตรงเงื่อนไขน้อยกว่านี้มาก PostgreSQL จะใช้ **`Index Scan`**
ตรงๆ:

```sql
EXPLAIN ANALYZE SELECT * FROM posts WHERE views = 254321;
```

```
 Index Scan using index_posts_on_views on posts  (cost=0.42..16.48 rows=3 width=61) (actual time=0.077..0.078 rows=1 loops=1)
   Index Cond: (views = 254321)
 Planning Time: 0.478 ms
 Execution Time: 0.128 ms
```

### สรุปคำศัพท์ที่ต้องรู้เมื่อสลับไปมาระหว่าง SQLite และ PostgreSQL

| แนวคิด | SQLite (`EXPLAIN QUERY PLAN`) | PostgreSQL (`EXPLAIN ANALYZE`) |
|---|---|---|
| เต็มตาราง | `SCAN posts` | `Seq Scan` (หรือ `Parallel Seq Scan`) |
| ใช้ index | `SEARCH posts USING INDEX ...` | `Index Scan` / `Bitmap Index Scan` |
| sort ที่ไม่มี index รองรับ | `USE TEMP B-TREE FOR ORDER BY` | `Sort` node แยกต่างหาก (มักมาคู่กับ `Sort Method: quicksort`/`external merge`) |
| รันจริงหรือแค่ประมาณ | แค่แสดงแผน ไม่รันจริง ไม่มีเวลา | รันจริงจริง (`ANALYZE`) ได้ `actual time`/`rows`/`loops` |
| คำสั่งฐาน (แค่ดูแผน ไม่รันจริง) | `EXPLAIN` (ไม่มี `QUERY PLAN`) ให้ opcode ระดับ VM ที่อ่านยากกว่ามาก | `EXPLAIN` เฉยๆ (ไม่มี `ANALYZE`) ก็ยังให้ cost estimate อ่านง่ายกว่า SQLite |

> **ข้อควรระวังของ `EXPLAIN ANALYZE`:** เพราะมันรันคำสั่งจริง ถ้าใช้กับคำสั่งที่แก้ไขข้อมูล
> (`UPDATE`/`DELETE`/`INSERT`) การเรียก `EXPLAIN ANALYZE` จะทำให้ข้อมูลถูกแก้ไขจริงด้วย! ควรใช้
> ภายใน transaction ที่ `ROLLBACK` เสมอถ้าทดสอบกับคำสั่งเหล่านี้บนฐานข้อมูลที่มีข้อมูลจริงอยู่
> (`BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`)

---

## Step 637: ติดตั้ง Bullet และจับ N+1 Query จริงแบบสด

ตั้งแต่ Part 027 เป็นต้นมา เราแก้ N+1 ด้วยการ**มองเห็น SQL log ด้วยตาเปล่า** — วิธีนี้ใช้ได้ตอน
เรียนหรือตอนเทสต์เล็กๆ แต่ในโปรเจกต์จริงที่มี view/controller หลายร้อยไฟล์ ไม่มีทางไล่อ่าน log
ทุกบรรทัดได้ทัน **Bullet** คือ gem ที่ตรวจจับ N+1 (และปัญหาการ query ที่ไม่เหมาะสมอื่นๆ) **แบบ
อัตโนมัติ** ระหว่างพัฒนา แล้วแจ้งเตือนทันทีที่เจอ

### ติดตั้งและตั้งค่า

```ruby
# Gemfile
group :development do
  gem "bullet"
end
```

```bash
bundle install
```

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ... config อื่นๆ ตามปกติ ...

  if defined?(Bullet)
    config.after_initialize do
      Bullet.enable = true
      Bullet.alert = false           # true = แสดง popup alert() ในเบราว์เซอร์ (รบกวนมาก ปกติปิดไว้)
      Bullet.bullet_logger = true    # เขียน warning ลง log/bullet.log แยกไฟล์
      Bullet.console = true          # ส่ง warning ไปที่ browser console (console.log)
      Bullet.rails_logger = true     # เขียนลง Rails.logger (log/development.log) ด้วย
      Bullet.add_footer = true       # แปะสรุปตัวเลข N+1 ที่มุมล่างของหน้าเว็บ
      Bullet.raise = true            # raise UnoptimizedQueryError ทันทีที่เจอ — เข้มงวดที่สุด เหมาะกับ dev/test
    end
  end
end
```

`Bullet.raise = true` คือโหมดที่เข้มงวดที่สุด: ทันทีที่ตรวจพบ N+1 ระหว่าง request ใดๆ ในโหมด
development หรือ test มันจะ **raise exception ทันที** ทำให้ทีมเห็นปัญหาตั้งแต่ตอนพัฒนา ไม่ใช่
มาเจอตอน production ช้าแล้ว — ทางเลือกที่ผ่อนปรนกว่าคือปิด `raise` ไว้แล้วใช้แค่ `bullet_logger`/
`rails_logger`/`console` เพื่อแค่ "แจ้งเตือน" โดยไม่หยุดการทำงาน เหมาะกับทีมที่ยังมี N+1 เดิม
ค้างอยู่เยอะและยังไม่พร้อมแก้ทั้งหมดทันที

### จับ N+1 จริงแบบสด

ในโค้ด controller/service จริง Bullet ทำงานผ่าน Rack middleware ที่ครอบทุก request โดยอัตโนมัติ
— แต่เพื่อทดลองนอก request (เช่นใน `rails runner` หรือ background job) ใช้ `Bullet.profile`
ครอบโค้ดที่ต้องการตรวจสอบได้เหมือนกัน:

```ruby
begin
  Bullet.profile do
    posts = Post.where(published: true).order(created_at: :desc).limit(20).to_a
    posts.each { |post| post.author.name }   # เรียก association โดยไม่ preload มาก่อน
  end
rescue => e
  puts "RAISED: #{e.class}"
  puts e.message
end
```

ผลลัพธ์จริงที่ได้ (raise ทันทีตามที่ตั้งค่าไว้):

```
RAISED: Bullet::Notification::UnoptimizedQueryError

USE eager loading detected
  Post => [:author]
  Add to your query: .includes([:author])
Call stack
```

นี่คือ**ข้อความเตือนจริง**ที่ Bullet สร้างขึ้น ไม่ใช่ log ทั่วไป — สังเกตว่ามันบอกครบทั้ง 3 อย่าง:
1. **ปัญหาคืออะไร** — `USE eager loading detected` (ควรใช้ eager loading แต่ไม่ได้ใช้)
2. **เกิดที่ไหน** — `Post => [:author]` (class `Post`, association `author`)
3. **วิธีแก้ที่แนะนำ** — `Add to your query: .includes([:author])` (บอกโค้ดที่ต้องเติมตรงๆ)

แก้ตามที่ Bullet แนะนำ แล้วรันซ้ำ:

```ruby
Bullet.profile do
  posts = Post.includes(:author).where(published: true).order(created_at: :desc).limit(20).to_a
  posts.each { |post| post.author.name }
end
puts "OK: no N+1 detected"
```

```
OK: no N+1 detected
```

ไม่มี exception เกิดขึ้นเลย — Bullet เห็นว่า `author` ถูก preload มาแล้วด้วย `includes` ก่อนถูก
เรียกใช้จริงในลูป จึงไม่ถือว่าเป็น N+1

> **`Bullet.profile` vs Rack middleware:** ในเว็บแอปจริง คุณ**ไม่ต้อง**เขียน `Bullet.profile`
> เองเลย เพราะ Bullet ติดตั้ง Rack middleware ที่ครอบทุก request ให้อัตโนมัติตั้งแต่ตอน
> `Bullet.enable = true` — `Bullet.profile` มีประโยชน์เฉพาะตอนทดสอบนอก request เช่นใน
> `rails runner`, background job, หรือ rake task ที่ไม่มี request/response cycle ให้ middleware
> ทำงานผ่าน

---

## Step 638: ความสามารถอื่นของ Bullet — Unused Eager Loading และ Counter Cache Suggestion

Bullet ตรวจจับปัญหาการ query ได้มากกว่าแค่ N+1 — มีอีกสองแบบที่ใช้บ่อยไม่แพ้กัน

### 1) Unused Eager Loading — สั่ง `includes` แต่ไม่ได้ใช้จริง

ตรงข้ามกับ N+1 คือ**การ preload มาเกินความจำเป็น** — สั่ง `includes` ไว้แล้วไม่เคยเรียกใช้
association นั้นเลยในโค้ดจริง ทำให้เสีย query โดยเปล่าประโยชน์:

```ruby
begin
  Bullet.profile do
    posts = Post.includes(:author, :category).where(published: true).order(created_at: :desc).limit(20).to_a
    posts.each { |post| post.title }   # ใช้แค่ title ไม่ได้แตะ author/category เลย
  end
rescue => e
  puts "RAISED: #{e.class}"
  puts e.message
end
```

```
RAISED: Bullet::Notification::UnoptimizedQueryError

AVOID eager loading detected
  Post => [:author, :category]
  Remove from your query: .includes([:author, :category])
Call stack
```

**`AVOID eager loading detected`** — คำแนะนำตรงข้ามกับ N+1 เป๊ะๆ: ครั้งนี้บอกให้**เอา
`.includes` ออก** เพราะมันสร้าง query เพิ่มโดยไม่มีใครใช้ผลลัพธ์นั้นเลย บั๊กแบบนี้มักเกิดตอน
refactor โค้ด — ลบส่วนที่ใช้ `post.author` ออกจาก view แล้ว แต่ลืมลบ `.includes(:author)` ออก
จาก controller ด้วย

> **ข้อควรระวัง: `.size`/`.count` บน association ที่ preload มาแล้วอาจทำให้เกิด false positive**
> ถ้าเรียก `post.comments.size` บน collection ที่โหลดมาแล้วจริงผ่าน `includes(:comments)`
> Ruby's `Array#size` (ไม่ใช่ method ของ ActiveRecord) จะถูกเรียกตรงๆ โดยไม่ผ่าน patch ของ
> Bullet เลย ทำให้ Bullet **เข้าใจผิดว่า association นี้ไม่ได้ถูกใช้** และแจ้ง `AVOID eager
> loading` ทั้งที่จริงๆ ถูกใช้แล้ว — วิธีเลี่ยงคือเรียก `.to_a.size` หรือ iterate ผ่าน `.each`
> อย่างน้อยหนึ่งครั้งก่อน ถ้าเจอ warning แบบนี้ที่ดูขัดกับสามัญสำนึก ให้สงสัยเคสนี้ไว้ก่อน

### 2) Counter Cache Suggestion — เรียก `.count`/`.size` บน association ซ้ำๆ โดยไม่มี counter cache

```ruby
begin
  Bullet.profile do
    posts = Post.order(:id).limit(50).to_a
    posts.each { |post| post.comments.count }   # นับ comment ของทุกโพสต์ ทีละ query
  end
rescue => e
  puts "RAISED: #{e.class}"
  puts e.message
end
```

```
RAISED: Bullet::Notification::UnoptimizedQueryError

Need Counter Cache with Active Record size
  Post => [:comments]
```

**`Need Counter Cache`** — Bullet ตรวจพบว่าโค้ดเรียก `.count` บน `has_many :comments` ซ้ำๆ ในลูป
โดยไม่มีคอลัมน์ counter cache รองรับ ทำให้ยิง `SELECT COUNT(*) FROM comments WHERE post_id = ?`
แยกทุกครั้ง (เป็น N+1 อีกรูปแบบหนึ่งที่ specific กับการนับจำนวน) — วิธีแก้ที่แท้จริงคือเพิ่ม
**counter cache column**:

```bash
bin/rails generate migration AddCommentsCountToPosts comments_count:integer
```

```ruby
class AddCommentsCountToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
  end
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
```

```bash
bin/rails db:migrate
# แล้ว backfill ค่าคอลัมน์ใหม่จากข้อมูลที่มีอยู่แล้ว
```

```ruby
Post.update_all("comments_count = (SELECT COUNT(*) FROM comments WHERE comments.post_id = posts.id)")
```

รันซ้ำด้วย `.size` (ไม่ใช่ `.count`):

```ruby
Bullet.profile do
  posts = Post.order(:id).limit(50).to_a
  posts.each { |post| post.comments.size }
end
puts "no notification"
```

```
no notification
```

> **กับดักสำคัญ: `.size` ใช้ counter cache แต่ `.count` ไม่ใช้!** แม้จะเพิ่ม `counter_cache: true`
> และ column `comments_count` ครบแล้ว ถ้ายังเรียก `post.comments.count` (ไม่ใช่ `.size`)
> Rails **ยังคงยิง SQL `COUNT(*)` เหมือนเดิมทุกครั้ง ไม่แตะ counter cache column เลย** —
> พิสูจน์แล้วจริงว่าเปลี่ยนแค่ `.count` เป็น `.size` เท่านั้นที่ทำให้ Bullet warning หายไปและ
> Rails ใช้คอลัมน์ cache แทนการ query จริง `.size`/`.length` บน collection association เช็ค
> counter cache column ก่อนเสมอถ้ามี แต่ `.count` ถูกออกแบบให้รองรับการนับแบบมีเงื่อนไข
> (`comments.where(approved: true).count`) จึงยิง SQL ตรงๆ เสมอโดยไม่สนใจ cache — จำกฎ:
> **อยากได้ประโยชน์จาก counter cache ให้ใช้ `.size` ไม่ใช่ `.count`**

---

## Step 639: เรื่องจริงของ Index ที่ "ไม่ช่วยอะไรเลย" — Cardinality ต่ำ และการตัดสินใจของ Query Planner

บทเรียนที่สำคัญที่สุดข้อหนึ่งของเรื่อง index: **การมี index ไม่ได้แปลว่า query จะเร็วขึ้นเสมอไป**
— ทุกอย่างขึ้นอยู่กับ **cardinality** (จำนวนค่าที่เป็นไปได้ต่างกันในคอลัมน์นั้น เทียบกับจำนวนแถว
ทั้งหมด) คอลัมน์ `published` เป็น `boolean` ที่มีแค่ 2 ค่าเท่านั้น (`true`/`false`) — ในตาราง
1,000,000 แถวของเรา มี `published = true` อยู่ **666,666 แถว (66.7%)**

### บน SQLite: Planner เลือกใช้ Index ทั้งที่ไม่ช่วยอะไรเลย

```ruby
conn.execute("CREATE INDEX index_posts_on_published ON posts (published)")
conn.execute("ANALYZE")

p Post.where(published: true).explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
3|0|212|SEARCH posts USING INDEX index_posts_on_published (published=?)
```

Query plan บอกว่าใช้ index (`SEARCH ... USING INDEX`) จริง — ดูเหมือนจะดี แต่วัดเวลาจริงแบบ
cold cache เทียบกับตอนไม่มี index:

```
=== ไม่มี index (SCAN) ===
cold time: 604.31 ms

=== มี index บน published (SEARCH) ===
cold time: 612.24 ms

ratio: 1.01x   ← ไม่เร็วขึ้นเลย บางรอบยังช้ากว่าเดิมด้วยซ้ำ
```

**แทบไม่มีความแตกต่างเลย — บางรอบยังช้ากว่าตอนไม่มี index ด้วยซ้ำ!** ทั้งที่ query plan บอกว่า
"ใช้ index" อยู่ชัดๆ เหตุผลคือ: การกรอง `published = true` ตรงกับ **66.7% ของทั้งตาราง** ทำให้
การใช้ index ต้อง**กระโดดไปมาแบบสุ่ม (random access)** เพื่อดึงแถวจริงกลับมาถึง 666,666 ครั้ง
ซึ่งในทางปฏิบัติไม่ได้เร็วไปกว่าการอ่านทุกแถวเรียงตามลำดับ (sequential scan) เลย — **ยิ่งผลลัพธ์
ที่ตรงเงื่อนไขเป็นสัดส่วนสูงของตารางเท่าไหร่ ประโยชน์ของ index ยิ่งลดลงเท่านั้น** จนถึงจุดที่
กลายเป็นภาระมากกว่าประโยชน์

### บน PostgreSQL: Planner ฉลาดกว่า — เลือก "ไม่ใช้" Index ตัวเดียวกันนี้เอง

นี่คือจุดที่น่าสนใจที่สุด: ทดสอบเงื่อนไขเดียวกันทุกประการบน PostgreSQL 16 (index มีอยู่จริง แต่
statistics ของ PostgreSQL ฉลาดกว่า SQLite มาก):

```sql
CREATE INDEX index_posts_on_published ON posts (published);
ANALYZE posts;

EXPLAIN ANALYZE SELECT * FROM posts WHERE published = true;
```

```
 Seq Scan on posts  (cost=0.00..22248.00 rows=664433 width=61) (actual time=0.009..93.994 rows=666666 loops=1)
   Filter: published
   Rows Removed by Filter: 333334
 Planning Time: 0.382 ms
 Execution Time: 116.459 ms
```

**PostgreSQL เลือก `Seq Scan` (full table scan) เองโดยตรง — ทั้งที่ index `index_posts_on_published`
มีอยู่จริงและพร้อมใช้งาน!** PostgreSQL เก็บสถิติการกระจายตัวของค่าในแต่ละคอลัมน์ไว้ (จาก
`ANALYZE`) แล้ว **คำนวณต้นทุนทั้งสองแผนเปรียบเทียบกันจริงๆ** ก่อนตัดสินใจ — เมื่อรู้ว่าเงื่อนไข
นี้ตรงกับ 66% ของตาราง มันคำนวณได้ว่า sequential scan ถูกกว่า จึง**ปฏิเสธที่จะใช้ index ที่มีอยู่
โดยเจตนา** นี่คือพฤติกรรมที่ถูกต้องและฉลาดกว่า SQLite ในสถานการณ์นี้

### บทเรียนที่ต้องจำ

ความจริงสองด้านที่ Step นี้พิสูจน์ให้เห็นพร้อมกัน:

1. **Index ที่มี cardinality ต่ำ (ค่าซ้ำกันเยอะ เทียบกับขนาดตาราง) มักไม่ช่วยอะไรเลย** —
   ยิ่งเงื่อนไขตรงกับสัดส่วนของตารางมากเท่าไหร่ ยิ่งใกล้เคียงกับ "อ่านทั้งตารางอยู่ดี"
2. **แค่เห็นคำว่า `SEARCH`/`Index Scan` ใน query plan ไม่ได้แปลว่า query นั้นเร็วที่สุดเท่าที่
   เป็นไปได้** — SQLite (planner ที่เรียบง่ายกว่า) เลือกใช้ index ที่ไม่ช่วยอะไรเลยในสถานการณ์นี้
   ในขณะที่ PostgreSQL (planner ที่ซับซ้อนกว่า พร้อม cost-based optimizer และสถิติจริง)
   **ปฏิเสธ index ตัวเดียวกันนี้เอง** เพราะรู้ว่าจะไม่ช่วย

ต้องวัดผลจริงเสมอ (ตาม Step 631 และ 636) ไม่ใช่เชื่อแค่ว่า "มี index แล้วต้องเร็วขึ้น" — คอลัมน์
ที่เหมาะกับ index คือคอลัมน์ที่ **ค่าที่ query กระจายตัวสูง (high cardinality)** เช่น `id`,
`email`, `slug`, `views` (แต่ละค่าไม่ซ้ำหรือซ้ำน้อย) ไม่ใช่คอลัมน์แบบ `boolean`/`status` ที่มีค่า
ให้เลือกน้อยมากเมื่อเทียบกับขนาดตาราง — ถ้าจำเป็นต้อง query ตามคอลัมน์ cardinality ต่ำบ่อยๆ
ทางเลือกที่ดีกว่าคือทำ **composite index** ที่รวมคอลัมน์นั้นกับคอลัมน์ cardinality สูงกว่า
(เช่น `[:published, :category_id]`) เพื่อให้ผลลัพธ์แคบลงจริงก่อนถึง index จะช่วยได้

---

## Step 640: `strict_loading` — เซฟตี้เน็ตของ Rails เองที่ทำงานคู่กับ Bullet

Bullet เป็น gem ภายนอกที่ต้องติดตั้งเพิ่มและทำงานเฉพาะตอน development/test เท่านั้น (ปกติไม่ควร
เปิดใน production) — **`strict_loading`** คือฟีเจอร์ที่ **มากับ Rails เอง** (ตั้งแต่ Rails 6.1)
ทำงานคนละแบบกับ Bullet: แทนที่จะ "แจ้งเตือน" มันจะ **ทำให้เกิด error ทันทีที่มีการ lazy-load
association ที่ถูกทำเครื่องหมายไว้** — เข้มงวดกว่า และใช้ได้แม้ใน production

### `strict_loading` ระดับ instance

```ruby
post = Post.strict_loading.find(1)

begin
  post.author.name
rescue => e
  puts "#{e.class}: #{e.message}"
end
```

```
ActiveRecord::StrictLoadingViolationError: `Post` is marked for strict_loading. The Author
association named `:author` cannot be lazily loaded.
```

`Post.strict_loading.find(1)` โหลด record เดียวมาแต่ **ทำเครื่องหมายไว้ว่าห้าม lazy-load
association ใดๆ ทั้งสิ้น** — ถ้าโค้ดพยายามเรียก `post.author` โดยไม่ preload มาก่อน จะได้
`ActiveRecord::StrictLoadingViolationError` ทันที ไม่ใช่แค่ warning เหมือน Bullet — เหมาะกับ
จุดที่รู้แน่ชัดว่า "ต้องไม่มี query เพิ่มเติมโดยเด็ดขาด" เช่น API endpoint ที่ serialize response
จาก record เดียว

### `strict_loading` ระดับ Association (บังคับทุกครั้งที่โหลดผ่าน association นี้)

```ruby
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy, strict_loading: true
end
```

```ruby
post = Post.find(1)

begin
  post.comments.to_a
rescue => e
  puts "#{e.class}: #{e.message}"
end
```

```
ActiveRecord::StrictLoadingViolationError: `Post` is marked for strict_loading. The Comment
association named `:comments` cannot be lazily loaded.
```

การประกาศ `strict_loading: true` ที่ตัว association เลย (ไม่ใช่ที่ query) หมายความว่า **ทุก
`Post` object ที่โหลดมาจากที่ไหนก็ตามในระบบ จะ error ทันทีถ้ามีใคร lazy-load `comments`** —
เข้มงวดกว่าการเรียก `.strict_loading` ที่ query เพราะไม่ต้องพึ่งว่าทุกจุดในโค้ดต้อง remember
เรียก `.strict_loading` เอง

ถ้า preload มาก่อนด้วย `includes` จะไม่ error เลย เพราะไม่ใช่การ **lazy** load อีกต่อไป:

```ruby
post = Post.includes(:comments).find(1)
puts post.comments.to_a.size   # โหลดมาแล้วล่วงหน้า ไม่ error
```

```
3
```

### `strict_loading` vs Bullet — ใช้เมื่อไหร่

| | Bullet | `strict_loading` |
|---|---|---|
| ติดตั้งเพิ่ม | ต้องเพิ่ม gem | มากับ Rails core |
| ใช้ได้ใน production | ไม่แนะนำ (overhead + noise) | ได้ (แต่ปกติเปิดเฉพาะจุดสำคัญ) |
| พฤติกรรมเมื่อเจอปัญหา | แจ้งเตือน/log/raise (เลือกได้) | raise เสมอ (all-or-nothing ต่อจุดที่ตั้งค่า) |
| ขอบเขต | ทั้งแอปอัตโนมัติผ่าน middleware | ต้องเลือกเปิดทีละ query/association เอง |
| ตรวจจับอะไรได้บ้าง | N+1, unused eager loading, counter cache | เฉพาะการ lazy-load ที่ถูกห้ามไว้ |

ในทางปฏิบัติ **ทั้งสองใช้ร่วมกันได้และเสริมกันดี**: เปิด Bullet ไว้ตลอดใน development/test เพื่อ
จับ N+1 แบบกว้างๆ ทั้งแอป ส่วน `strict_loading` เอาไว้ล็อกจุดที่รู้ชัดเจนแล้วว่าห้ามมี query
เพิ่มเติมโดยเด็ดขาด (เช่น response ของ API ที่ serialize ข้อมูลจำนวนมาก หรือ background job ที่
ประมวลผลนับล้าน record — ที่ N+1 หนึ่งจุดขยายผลกระทบมหาศาล) นอกจากนี้ Rails 8.1 ยังมี
`config.active_record.strict_loading_by_default = true` ที่เปิด strict loading เป็นค่าเริ่มต้น
ให้ **ทุก** query ในแอป เหมาะกับทีมที่อยากบังคับวินัยเรื่อง eager loading ตั้งแต่ต้นโปรเจกต์ใหม่

---

## แบบฝึกหัด: Profile Query ช้าบนตาราง 1,000,000 แถว, เพิ่ม Index ที่ถูกต้อง, และจับ/แก้ N+1 จริงด้วย Bullet

### โจทย์

ใช้ตาราง `posts` (1,000,000 แถว) จาก Part นี้ ทำดังนี้:

1. Profile query `Post.where(slug: "post-number-999999")` (หาโพสต์จาก slug ตรงๆ — เหมือนหน้า
   permalink ของบทความ) วัดเวลาจริงแบบ cold cache **ก่อน** มี index บน `slug`
2. เพิ่ม unique index บน `slug` ผ่าน migration แล้ววัดเวลาซ้ำ เปรียบเทียบผลต่าง
3. เขียนสคริปต์ที่โหลดโพสต์ 50 อันแรก แล้วแสดงชื่อผู้เขียนกับจำนวนคอมเมนต์ของแต่ละโพสต์ (แบบที่
   หน้า "รายการบทความ" ทั่วไปต้องทำ) — เปิด Bullet ตรวจจับปัญหาที่เกิดขึ้น
4. แก้ปัญหาที่ Bullet รายงานทั้งหมด จนกว่าจะไม่มี warning เหลือ

### เฉลย

**1) Profile ก่อนมี index**

```ruby
# profile_slug.rb
require "benchmark"

conn = ActiveRecord::Base.connection

def drop_os_cache
  system("sync")
  File.write("/proc/sys/vm/drop_caches", "3")
end

p Post.where(slug: "post-number-999999").explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."slug" = 'post-number-999999'
2|0|215|SCAN posts
```

`SCAN posts` ยืนยันว่าต้องไล่ตรวจทุกแถวจากทั้งหมด 1,000,000 แถวเพื่อหา slug ที่ตรงกันแค่แถว
เดียว — วัดเวลาจริงแบบ cold cache:

```ruby
ActiveRecord::Base.connection_pool.disconnect!
drop_os_cache
t1 = Benchmark.realtime { Post.where(slug: "post-number-999999").to_a }
puts "before index: #{(t1 * 1000).round(2)} ms"
```

```
before index: 274.67 ms
```

**2) เพิ่ม unique index แล้ววัดซ้ำ**

```bash
bin/rails generate migration AddUniqueIndexToPostsSlug
```

```ruby
class AddUniqueIndexToPostsSlug < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, :slug, unique: true
  end
end
```

```bash
bin/rails db:migrate
```

```ruby
p Post.where(slug: "post-number-999999").explain
```

```
EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."slug" = 'post-number-999999'
3|0|39|SEARCH posts USING INDEX index_posts_on_slug (slug=?)
```

```ruby
ActiveRecord::Base.connection_pool.disconnect!
drop_os_cache
t2 = Benchmark.realtime { Post.where(slug: "post-number-999999").to_a }
puts "after index: #{(t2 * 1000).round(2)} ms"
puts "speedup: #{(t1 / t2).round(1)}x"
```

```
after index: 3.6 ms
speedup: 76.3x
```

**เร็วขึ้นถึง 76 เท่า** — ต่างจากกรณี `published` ใน Step 639 อย่างสิ้นเชิง เพราะ `slug` มี
**cardinality สูงสุด** (ทุกค่าไม่ซ้ำกันเลย เพราะมี unique index บังคับอยู่) ทำให้ index ช่วยได้
เต็มที่ — นี่คือตัวอย่างที่ตรงข้ามกับ Step 639 พอดี: ยิ่ง cardinality สูง ยิ่งได้ประโยชน์จาก
index มากเท่านั้น

**3) โหลดโพสต์แสดงผู้เขียนและจำนวนคอมเมนต์ พร้อมเปิด Bullet**

```ruby
# exercise_bullet.rb
Bullet.raise = false   # ดู warning ทั้งหมดก่อน ยังไม่ raise

Bullet.profile do
  posts = Post.order(:id).limit(50).to_a
  posts.each do |post|
    post.author.name
    post.comments.count
  end
end
```

ตรวจสอบ `log/bullet.log` หลังรัน:

```
USE eager loading detected
  Post => [:author]
  Add to your query: .includes([:author])
Call stack

Need Counter Cache with Active Record size
  Post => [:comments]

USE eager loading detected
  Post => [:comments]
  Add to your query: .includes([:comments])
Call stack
```

Bullet เจอปัญหา **สามอย่างพร้อมกัน**: (1) `author` ควร eager load, (2) `comments` ควรมี counter
cache, (3) `comments` ก็ควร eager load ด้วยเช่นกัน (สำหรับตอนที่ counter cache ยังไม่พร้อม)

**4) แก้ทั้งหมด**

```ruby
# app/models/post.rb — เพิ่ม comments_count ผ่าน migration ตามที่ Step 638 สอนไว้ก่อน
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end
```

```bash
bin/rails generate migration AddCommentsCountToPosts comments_count:integer
# แก้ migration ให้เป็น: add_column :posts, :comments_count, :integer, default: 0, null: false
bin/rails db:migrate
```

```ruby
Post.update_all("comments_count = (SELECT COUNT(*) FROM comments WHERE comments.post_id = posts.id)")
```

```ruby
> Post.reset_column_information

Bullet.raise = false
Bullet.profile do
  posts = Post.includes(:author).order(:id).limit(50).to_a
  posts.each do |post|
    post.author.name
    post.comments.size   # .size ไม่ใช่ .count — ใช้ counter cache column แทนการยิง SQL
  end
end
```

ตรวจสอบ `log/bullet.log` อีกครั้ง — ไม่มี warning ใหม่เลย ปัญหาทั้งหมดถูกแก้เรียบร้อย
(สังเกตว่าไม่ต้อง `.includes(:comments)` อีกต่อไป เพราะ `.size` อ่านจาก `comments_count` column
ที่ preload มาพร้อมกับตัว `Post` เองอยู่แล้ว ไม่ต้อง query ตาราง `comments` เพิ่มเลย)

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. ทำ composite index `[:category_id, :published_at]` ตาม Step 632 บนเครื่องของตัวเอง แล้วลอง
   query แบบ `Post.where(category_id: X).order(published_at: :desc).limit(20)` เทียบ `.explain`
   ก่อน/หลังเพิ่ม index — ยืนยันด้วยตัวเองว่า `USE TEMP B-TREE FOR ORDER BY` หายไปจริง จากนั้นลอง
   สลับลำดับคอลัมน์เป็น `[:published_at, :category_id]` แล้วพิสูจน์ด้วยตัวเองว่า SQLite
   เพิกเฉยต่อ index ที่ลำดับผิดจริงตามที่ Step 632 อธิบายไว้
2. เปิด `Post.strict_loading.find(1)` แล้วลองเข้าถึง `post.category.name` โดยไม่ preload —
   ควรได้ `ActiveRecord::StrictLoadingViolationError` เหมือนที่ Step 640 พิสูจน์ไว้กับ `author`
   จากนั้นแก้ด้วย `Post.includes(:category).strict_loading.find(1)` แล้วยืนยันว่าไม่ error
3. ทดลองสร้างสถานการณ์ "unused eager loading" ของตัวเอง (สั่ง `.includes` สอง association
   แต่ใช้จริงแค่ตัวเดียว) แล้วดู log ของ Bullet ว่าบอก association ไหนที่ควรเอาออก — ลองแก้สอง
   แบบเปรียบเทียบกัน: (ก) เอา `.includes` ของตัวที่ไม่ใช้ออก กับ (ข) เพิ่มโค้ดให้ใช้ association
   นั้นจริงๆ ในลูป (เช่นแสดงผลเพิ่ม) แล้วสังเกตว่า warning หายไปทั้งสองวิธีแต่ด้วยเหตุผลคนละแบบ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Index คือ B-Tree** ที่เก็บคู่ "ค่าคอลัมน์ → ตำแหน่งแถวจริง" เรียงลำดับไว้ล่วงหน้า เปลี่ยนการ
  ค้นหาจาก O(n) (ไล่อ่านทุกแถว หรือ **full table scan** — SQLite เรียก `SCAN`, PostgreSQL เรียก
  `Seq Scan`) เป็น O(log n) (**`SEARCH`/`Index Scan`**) — วัดผลจริงบนตาราง 1,000,000 แถว: จาก
  ~13-53 ms (SCAN) เหลือ ~2.7-3.1 ms (SEARCH ด้วย index) เมื่อวัดแบบ cold cache
- เพิ่ม index ผ่าน `add_index :posts, :author_id` (คอลัมน์เดียว) หรือ `add_index :posts,
  [:category_id, :published_at]` (**composite index** — วางคอลัมน์ equality ก่อน range/order
  เสมอ) **ลำดับคอลัมน์สำคัญมาก**: composite index `[A, B]` ใช้กรอง/เรียงตาม `B` อย่างเดียวแทบ
  ไม่ได้ผล (leftmost-prefix rule) — พิสูจน์แล้วว่า SQLite เพิกเฉยต่อ index ที่ลำดับผิดโดยสิ้นเชิง
- `t.references` สร้าง **index ให้ foreign key อัตโนมัติ** อยู่แล้ว (ยืนยันจาก `schema.rb`) —
  คอลัมน์ที่ควรพิจารณาเพิ่ม index เองคือคอลัมน์ที่ใช้ใน `WHERE`/`ORDER BY`/`JOIN` บ่อยบนตารางใหญ่
- **Unique index** (`add_index :col, unique: true`) คือ index ปกติ + กฎบังคับไม่ซ้ำระดับ
  ฐานข้อมูล — คู่กับ `uniqueness` validator เสมอ (ทบทวน Part 032): validator ให้ UX ที่ดี, unique
  index ให้การรับประกันที่แท้จริงที่ป้องกัน race condition ได้
- **Index ไม่ใช่ของฟรี**: วัดจริงพบว่า insert 50,000 แถวเข้าตารางที่มี 6 index ช้ากว่าตารางที่มี
  แค่ primary key ประมาณ 6-7% และ index ทั้ง 6 ตัวรวมกันใช้ **พื้นที่ดิสก์เกือบ 2 เท่า** ของ
  ขนาดข้อมูลจริงในตาราง — เพิ่มเฉพาะที่มีหลักฐานว่าคุ้มค่าจริง
- **`EXPLAIN`/`.explain`** ของ SQLite ให้แค่แผนการ query (`SCAN`/`SEARCH`/`USE TEMP B-TREE FOR
  ORDER BY`) ไม่มีเวลาจริง ส่วน **PostgreSQL `EXPLAIN ANALYZE`** รันจริงและรายงาน `actual time`/
  `rows`/`Execution Time` จริง (`Seq Scan`/`Parallel Seq Scan` ↔ `Index Scan`/`Bitmap Index
  Scan`) — วัดจริงบน PostgreSQL 16: จาก 36.297 ms (Seq Scan) เหลือ 2.718 ms (Bitmap Index Scan)
- **`bullet`** gem ตรวจจับปัญหาการ query แบบอัตโนมัติผ่าน Rack middleware: **N+1** (`USE eager
  loading detected` พร้อมบอกโค้ดที่ต้องเติมตรงๆ), **unused eager loading** (`AVOID eager loading
  detected` เมื่อ `.includes` มาแล้วไม่ได้ใช้ — ระวัง false positive จาก `.size` บน association
  ที่โหลดแล้ว), และ **counter cache suggestion** (`Need Counter Cache` เมื่อเรียก `.count` ซ้ำๆ
  — แต่ `.size` เท่านั้นที่ใช้ counter cache column จริง `.count` ยิง SQL เสมอ) `Bullet.raise =
  true` ทำให้ raise exception ทันทีที่เจอปัญหาในโหมด dev/test
- **Index ไม่ได้ช่วยเสมอไป**: คอลัมน์ cardinality ต่ำ (เช่น `boolean` ที่ match 66% ของตาราง)
  พิสูจน์แล้วว่า SQLite ยังคงเลือกใช้ index (`SEARCH`) ทั้งที่**ไม่เร็วขึ้นเลยจากการวัดจริง**
  ในขณะที่ PostgreSQL (ด้วยสถิติจริงจาก `ANALYZE`) **ปฏิเสธที่จะใช้ index ตัวเดียวกันนี้เอง**
  แล้วเลือก `Seq Scan` แทน — บทเรียน: เห็นคำว่า `SEARCH`/`Index Scan` ใน plan ไม่ได้แปลว่าเร็ว
  ที่สุดเสมอ ต้องวัดเวลาจริงประกอบเสมอ
- **`strict_loading`** (มากับ Rails core ไม่ต้องติดตั้งเพิ่ม) ทำให้เกิด
  `ActiveRecord::StrictLoadingViolationError` ทันทีที่มีการ lazy-load association ที่ถูกล็อกไว้
  — ใช้ได้ทั้งระดับ query เดี่ยว (`Post.strict_loading.find(1)`) และระดับ association ถาวร
  (`has_many :comments, strict_loading: true`) เป็นเซฟตี้เน็ตที่เข้มงวดกว่า Bullet และใช้ได้ใน
  production ด้วย — ทั้งสองเครื่องมือเสริมกันได้ดี ไม่ต้องเลือกใช้แค่ตัวเดียว

**ต่อไป (Part 065):** เรารู้วิธีเจาะลึกปัญหาที่ **ฐานข้อมูล** แล้ว — Part ถัดไปจะขยับมาที่การ
**profile ทั้งแอปพลิเคชัน** ด้วย **`rack-mini-profiler`** (เห็นเวลาที่ใช้ในแต่ละ query/view/
partial ตรงๆ บนหน้าเว็บทุกหน้าที่โหลดในโหมด development) และ **memory profiling** (หา object ที่
กิน memory เกินจำเป็น ด้วย `memory_profiler` และ `derailed_benchmarks`) — Part 065 คือ Part
สุดท้ายของเฟส 9 (Background Jobs & Performance) ที่จะรวบยอดทุกเรื่องที่เรียนมาตั้งแต่ ActiveJob,
Sidekiq, Caching จนถึง Database Performance เข้าด้วยกันเป็นภาพรวมเดียว
