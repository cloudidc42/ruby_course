# Part 068: Full-text Search ด้วย pg_search

> **Step ครอบคลุมใน Part นี้:** Step 671–680
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 034 เรื่อง Query Interface และ Part 038 เรื่อง Pagination/
> Sorting/Filtering มาก่อน — Part นี้ต่อยอด `where`/`order`/scope และรูปแบบ pagination ที่สอง
> Part นั้นวางไว้โดยตรง)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / PostgreSQL 16 / `pg_search` 2.4.x (ทุกคำสั่ง, SQL
> ที่ generate จริง, และผลลัพธ์การค้นหาในเอกสารนี้ **รันจริง** บน Ruby 3.3.6 + Rails 8.1.4 +
> PostgreSQL 16.13 + pg_search 2.4.0 ไม่มีค่าที่แต่งขึ้นมาเอง)

> **สำคัญมาก — สลับฐานข้อมูล:** ทุก Part ที่ผ่านมาในคอร์สนี้ใช้ **SQLite** เป็นค่าเริ่มต้นเพราะ
> ติดตั้งง่ายและเพียงพอสำหรับการเรียนรู้ Rails ทั่วไป แต่ `pg_search` ใช้ฟีเจอร์ full-text search
> ที่เป็นของ **PostgreSQL โดยเฉพาะ** (`tsvector`, `tsquery`, extension `pg_trgm`) ซึ่ง SQLite
> ไม่มีให้ใช้แทนเลย — Part นี้จึงต้องสร้างโปรเจกต์ใหม่ด้วย `--database=postgresql` และต้องมี
> PostgreSQL server รันอยู่จริงในเครื่องก่อนเริ่ม Step 672

## สารบัญของ Part นี้

- Step 671: ทำไม `WHERE title LIKE '%keyword%'` ไม่พอ และ Full-text Search คืออะไรจริงๆ
- Step 672: สลับ Rails ไปใช้ PostgreSQL — `rails new --database=postgresql`, `config/database.yml`
- Step 673: ติดตั้ง `pg_search` และสร้าง search scope แรก — `pg_search_scope :search_by_title`
- Step 674: ค้นหาหลายคอลัมน์และข้าม Association — `against: [...]`, `associated_against`
- Step 675: ปรับแต่งการค้นหา — `prefix`, `dictionary` (stemming), `negation`, `any_word`
- Step 676: ข้อจำกัดสำคัญ — PostgreSQL Full-text Search กับภาษาไทยไปด้วยกันไม่ค่อยได้ (พิสูจน์จริง)
- Step 677: ทางออกที่ไทยเป็นมิตรกว่า — Trigram Search ด้วย extension `pg_trgm`
- Step 678: เรียงผลลัพธ์ตาม Relevance — `pg_search_rank`, `with_pg_search_rank`, `ranked_by`
- Step 679: รวม pg_search กับ ActiveRecord Scope/Pagination และเตรียมพร้อมสำหรับตอนสเกล
- Step 680: แบบฝึกหัด — Post model ที่มีทั้ง search scope แบบอังกฤษและแบบไทยใช้งานคู่กัน

---

## Step 671: ทำไม `WHERE title LIKE '%keyword%'` ไม่พอ และ Full-text Search คืออะไรจริงๆ

นักพัฒนาที่เพิ่งเริ่มทำระบบค้นหามักเริ่มจากโค้ดแบบนี้ (จำได้จาก Part 034 เรื่อง Query Interface):

```ruby
Post.where("title LIKE ?", "%#{params[:q]}%")
```

ใช้งานได้จริงในระดับ demo แต่มีปัญหาสามข้อที่ทำให้ **ใช้ใน production ไม่ได้เมื่อข้อมูลโตขึ้น**:

### ปัญหาที่ 1: ไม่มี Relevance Scoring (การจัดอันดับความเกี่ยวข้อง)

`LIKE` ตอบได้แค่ "match หรือไม่ match" — ไม่มีแนวคิดว่าแถวไหน "เกี่ยวข้องมากกว่า" แถวอื่น ผลลัพธ์
ที่ได้เรียงตาม `ORDER BY` ที่ระบุเอง (หรือไม่เรียงเลยถ้าไม่ใส่ `order`) ไม่ใช่เรียงตามว่า keyword
ปรากฏบ่อยแค่ไหน อยู่ในตำแหน่งสำคัญแค่ไหน (เช่นอยู่ใน title vs ฝังอยู่กลาง body) ลองดูตัวอย่างจริง:

```sql
-- ข้อมูลจริงจากตาราง posts ในเอกสารนี้
SELECT title FROM posts WHERE title LIKE '%PostgreSQL%';
```

```
                    title
----------------------------------------------
 วิธีติดตั้ง PostgreSQL บน Ubuntu
 Understanding Full-Text Search in PostgreSQL
 Introduction to PostgreSQL Extensions
```

ผลลัพธ์เรียงตาม `id` (ลำดับที่ insert) ไม่ใช่ตามความเกี่ยวข้อง — บทความที่ชื่อพูดถึง PostgreSQL
ตรงๆ ("Understanding Full-Text Search **in PostgreSQL**") กับบทความที่ PostgreSQL เป็นแค่ส่วน
หนึ่งของหัวข้อ ("วิธีติดตั้ง **PostgreSQL** บน Ubuntu") ได้ตำแหน่งเท่ากันหมด

### ปัญหาที่ 2: ช้าบนตารางใหญ่ เพราะ Index ปกติใช้กับมันไม่ได้

`LIKE '%keyword%'` มี wildcard (`%`) นำหน้า keyword ซึ่งทำให้ query planner **ใช้ B-Tree index
แบบปกติไม่ได้เลย** แม้จะสร้าง index ให้คอลัมน์นั้นไว้แล้วก็ตาม พิสูจน์จริงด้วย `EXPLAIN`:

```sql
CREATE INDEX idx_posts_title_btree ON posts (title);

EXPLAIN SELECT * FROM posts WHERE title LIKE '%PostgreSQL%';
```

```
                      QUERY PLAN
--------------------------------------------------------
 Seq Scan on posts  (cost=0.00..1.07 rows=1 width=128)
   Filter: ((title)::text ~~ '%PostgreSQL%'::text)
```

แม้มี index บนคอลัมน์ `title` อยู่แล้ว PostgreSQL ก็ยังเลือก **`Seq Scan`** (อ่านทุกแถว) เพราะ
B-Tree index เก็บค่าเรียงตามตัวอักษรจากซ้ายไปขวา การมี `%` นำหน้าหมายความว่า "คำนี้อยู่ตรงไหนก็ได้
ในข้อความ" ซึ่ง B-Tree ตอบคำถามแบบนี้ไม่ได้ (มันตอบได้แค่ "ค่าที่ขึ้นต้นด้วยอะไร" ผ่าน
range scan) — ตารางมี 6 แถวตอนนี้เลยไม่รู้สึกอะไร แต่ตารางระดับล้านแถวแบบที่ Part 064 สาธิตไว้
`Seq Scan` แบบนี้คือค่าใช้จ่ายที่แพงมาก และยิ่งแพงขึ้นเรื่อยๆ ตามขนาดตาราง ไม่ว่าจะสร้าง index
เพิ่มอีกกี่ตัวก็ช่วยไม่ได้ ตราบใดที่ยังใช้ `LIKE '%...%'`

### ปัญหาที่ 3: ไม่รองรับคำที่สะกดต่างกันหรือรูปคำที่ผันไป

`LIKE` เทียบตัวอักษรตรงตัวเท่านั้น:

- ค้นหา `"running"` จะไม่เจอบทความที่มีคำว่า `"run"` หรือ `"runs"` (คนละรูปคำ)
- ค้นหา `"PostgreSQL"` (พิมพ์ผิดเป็น `"Postgre SQL"` มีช่องว่าง หรือ `"Postgres"`) จะไม่เจออะไรเลย
- ไม่มีแนวคิดเรื่อง "คำหยุด" (stop words เช่น "the", "a", "และ") ที่ควรถูกมองข้ามไปตอนค้นหา

### แล้ว "Full-text Search" ที่แท้จริงคืออะไร

Full-text search แก้ปัญหาทั้งสามข้อข้างบนด้วยกลไกหลัก 3 อย่าง:

1. **Tokenization (การตัดคำ)** — แยกข้อความยาวๆ ออกเป็นหน่วยคำ (token/lexeme) ก่อน แทนที่จะ
   เก็บเป็น string ก้อนเดียว เช่น `"Understanding Full-Text Search in PostgreSQL"` จะถูกตัดเป็น
   `understand`, `full-text`, `text`, `search`, `postgresql` (ตัด stop word อย่าง `"in"` ทิ้งไป)
2. **Stemming (การตัดรูปคำให้เหลือราก)** — แปลงคำที่ผันรูปต่างกันให้กลายเป็นรากศัพท์เดียวกัน เช่น
   `understanding` → `understand`, `searches`/`searching` → `search` ทำให้ค้นหาคำไหนก็เจอทุกรูปคำ
3. **Ranking (การจัดอันดับ)** — คำนวณคะแนนความเกี่ยวข้องของแต่ละแถวจากปัจจัยต่างๆ เช่น
   keyword ปรากฏกี่ครั้ง, อยู่ในคอลัมน์ที่ "สำคัญ" กว่าหรือไม่ (title สำคัญกว่า body), คำที่ค้น
   อยู่ติดกันเป็นวลีหรือกระจัดกระจาย แล้วเรียงผลลัพธ์จากคะแนนสูงไปต่ำ

ลองดูของจริงผ่าน `psql` (ยังไม่ต้องมี Rails app เลยด้วยซ้ำ — นี่คือฟีเจอร์ของ PostgreSQL ล้วนๆ):

```sql
SELECT to_tsvector('english', 'Understanding Full-Text Search in PostgreSQL');
```

```
                              to_tsvector
------------------------------------------------------------------------
 'full':3 'full-text':2 'postgresql':7 'search':5 'text':4 'understand':1
```

สังเกต 3 อย่าง: (1) `"in"` หายไปเพราะเป็น stop word, (2) `Understanding` กลายเป็น `understand`
(stemmed แล้ว), (3) ตัวเลขต่อท้ายแต่ละคำคือ **ตำแหน่งของคำในประโยค** (`tsvector` นี้เรียกว่า
"document" ในศัพท์ PostgreSQL) เก็บไว้ใช้คำนวณ ranking และ phrase matching ทีหลัง

ลองค้นหาด้วยรูปคำที่ต่างออกไป:

```sql
SELECT to_tsvector('english', 'Understanding Full-Text Search in PostgreSQL')
       @@ to_tsquery('english', 'understands');
-- => t   (true — 'understands' ถูก stem เป็น 'understand' เหมือนกัน จึง match)
```

นี่คือสิ่งที่ `LIKE '%understands%'` ทำไม่ได้เลย — และนี่คือสิ่งที่ gem `pg_search` จะมาห่อหุ้มให้
เขียนเป็นภาษา Ruby/ActiveRecord ธรรมดาแทนที่จะเขียน SQL ดิบแบบนี้ทุกครั้ง

---

## Step 672: สลับ Rails ไปใช้ PostgreSQL — `rails new --database=postgresql`

ก่อนเริ่ม ตรวจสอบว่ามี PostgreSQL server รันอยู่ในเครื่องแล้ว:

```bash
pg_lsclusters
# Ver Cluster Port Status Owner    Data directory              Log file
# 16  main    5432 online postgres /var/lib/postgresql/16/main ...

psql --version
# psql (PostgreSQL) 16.13 (Ubuntu 16.13-0ubuntu0.24.04.1)
```

ถ้ายังไม่ได้ติดตั้ง (Ubuntu/Debian):

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
sudo service postgresql start
```

### สร้างโปรเจกต์ใหม่ด้วย `--database=postgresql`

ต่างจากทุก Part ก่อนหน้าที่ใช้ `rails new app_name --minimal` (SQLite เป็นค่าเริ่มต้น) Part นี้
ต้องระบุ flag เพิ่ม:

```bash
rails new pgsearch_demo --database=postgresql
cd pgsearch_demo
```

Rails จะ generate `Gemfile` ที่มี `gem "pg", "~> 1.1"` แทน `gem "sqlite3"` และ generate
`config/database.yml` ที่ตั้งค่า adapter เป็น `postgresql` ให้อัตโนมัติ:

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  max_connections: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: pgsearch_demo_development
  # host: localhost      # ปกติไม่ต้องเปิดถ้าใช้ Unix socket ในเครื่อง dev
  # username: ...
  # password: ...

test:
  <<: *default
  database: pgsearch_demo_test

production:
  <<: *default
  database: pgsearch_demo_production
  username: pgsearch_demo
  password: <%= ENV["PGSEARCH_DEMO_DATABASE_PASSWORD"] %>
```

**อธิบายความต่างจาก SQLite ที่คุ้นเคย:** SQLite เก็บฐานข้อมูลเป็นไฟล์เดียวในโปรเจกต์
(`storage/development.sqlite3`) ไม่ต้องมี server แยก แต่ PostgreSQL เป็น **client-server
database** ต้องมี process `postgres` รันอยู่ต่างหาก แล้ว Rails app เชื่อมต่อเข้าไปผ่าน TCP หรือ
Unix socket — เครื่อง dev ทั่วไปเชื่อมผ่าน Unix socket ได้เลยโดยไม่ต้องตั้ง `host`/`username`/
`password` (ใช้สิทธิ์ของ user ระบบที่รันคำสั่งอยู่ ที่เรียกว่า **peer authentication**) แต่บน
production ควรระบุ `username`/`password` หรือ `DATABASE_URL` เสมอ (ตามที่ comment ในไฟล์บอกไว้)

### สร้างฐานข้อมูล

```bash
bin/rails db:create
```

```
Created database 'pgsearch_demo_development'
Created database 'pgsearch_demo_test'
```

ถ้าเจอ error `FATAL: role "..." does not exist` หรือ `Peer authentication failed` แปลว่า
PostgreSQL user ของระบบยังไม่มีสิทธิ์สร้างฐานข้อมูล แก้ด้วยการสร้าง role ให้ตรงกับ user ปัจจุบัน
ก่อน (รันครั้งเดียวตอน setup เครื่อง):

```bash
sudo -u postgres createuser --superuser $(whoami)
```

---

## Step 673: ติดตั้ง `pg_search` และสร้าง Search Scope แรก

เพิ่ม gem ใน `Gemfile`:

```ruby
# Gemfile
gem "pg", "~> 1.1"
gem "pg_search"
```

```bash
bundle install
```

```
Bundle complete! 8 Gemfile dependencies, 71 gems now installed.
```

### เตรียมโมเดลสำหรับ Part นี้

```bash
bin/rails generate model Author name:string
bin/rails generate model Post title:string body:text author:references
bin/rails db:migrate
```

```
== CreateAuthors: migrated ====
== CreatePosts: migrated ====
```

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :posts
end
```

### `include PgSearch::Model` — จุดเริ่มต้นที่ขาดไม่ได้

`pg_search` ทำงานผ่าน module `PgSearch::Model` ที่ต้อง `include` เข้าไปในโมเดลก่อนถึงจะมี method
`pg_search_scope` ให้ใช้ ลืม include แล้วจะได้ error ทันที:

```ruby
# ทดลอง: ถ้าลืม include PgSearch::Model
class Author < ApplicationRecord
  pg_search_scope :search_by_name, against: :name
end
# => undefined method 'pg_search_scope' for class Author
```

โค้ดที่ถูกต้อง:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include PgSearch::Model

  belongs_to :author

  pg_search_scope :search_by_title, against: :title
end
```

`pg_search_scope` สร้าง **class method** (จริงๆ แล้วเป็น scope ที่ return
`ActiveRecord::Relation` เหมือน scope ปกติที่เรียนใน Part 034) ชื่อ `search_by_title` ที่รับ
คำค้นหาเป็น argument เดียว — `against: :title` บอกว่าให้ค้นหาในคอลัมน์ `title` เท่านั้น

### Seed ข้อมูลและทดลองค้นหา

```ruby
# db/seeds.rb
somchai = Author.create!(name: "สมชาย ใจดี")
jane = Author.create!(name: "Jane Smith")

Post.create!(author: somchai, title: "Ruby on Rails คืออะไร",
             body: "Ruby on Rails เป็น web framework ที่เขียนด้วยภาษา Ruby " \
                    "เหมาะสำหรับการพัฒนาเว็บแอปพลิเคชันอย่างรวดเร็ว")
Post.create!(author: somchai, title: "การเขียนโปรแกรมภาษาไทยเบื้องต้น",
             body: "บทความนี้จะพาไปทำความรู้จักกับการเขียนโปรแกรมภาษาไทย " \
                    "และการจัดการข้อความภาษาไทยใน Ruby on Rails")
Post.create!(author: somchai, title: "วิธีติดตั้ง PostgreSQL บน Ubuntu",
             body: "คู่มือการติดตั้งฐานข้อมูล PostgreSQL บนระบบปฏิบัติการ Ubuntu " \
                    "แบบทีละขั้นตอนสำหรับผู้เริ่มต้น")
Post.create!(author: jane, title: "Understanding Full-Text Search in PostgreSQL",
             body: "PostgreSQL provides powerful full-text search capabilities " \
                    "through tsvector and tsquery data types.")
Post.create!(author: jane, title: "Ruby on Rails Performance Tuning",
             body: "Learn how to optimize your Rails application for better " \
                    "database and full-text search performance.")
Post.create!(author: jane, title: "Introduction to PostgreSQL Extensions",
             body: "This article explains pg_trgm and other useful PostgreSQL " \
                    "extensions for search and indexing.")
```

```bash
bin/rails db:seed
bin/rails console
```

```irb
irb> Post.search_by_title("PostgreSQL").map(&:title)
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu",
    "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
```

ผลลัพธ์ตรงกับตัวอย่าง `LIKE` ใน Step 671 เป๊ะ (เพราะข้อมูลชุดนี้ยังไม่ได้แสดงข้อได้เปรียบของ
stemming) แต่เบื้องหลังต่างกันโดยสิ้นเชิง — ลองดู SQL ที่ `pg_search_scope` generate ให้:

```irb
irb> puts Post.search_by_title("PostgreSQL").to_sql
```

```sql
SELECT "posts".* FROM "posts" INNER JOIN (
  SELECT "posts"."id" AS pg_search_id,
    (ts_rank((to_tsvector('simple', coalesce(cast("posts"."title" AS text), ''))),
              (to_tsquery('simple', ''' ' || 'PostgreSQL' || ' ''')), 0)) AS rank
  FROM "posts"
  WHERE ((to_tsvector('simple', coalesce(cast("posts"."title" AS text), '')))
         @@ (to_tsquery('simple', ''' ' || 'PostgreSQL' || ' ''')))
) pg_search_... ON "posts"."id" = pg_search_....pg_search_id
ORDER BY pg_search_....rank DESC, "posts"."id" ASC
```

`pg_search` เขียน `to_tsvector`/`to_tsquery`/`ts_rank` ให้ครบตามหลักการที่อธิบายใน Step 671 —
join กับ subquery ที่คำนวณ rank แล้วเรียงตาม rank จากมากไปน้อย และผลลัพธ์ยังเป็น
`ActiveRecord::Relation` ธรรมดาที่ chain ต่อ `.where`, `.limit`, `.order` ได้เหมือน scope ทั่วไป

> **สังเกต:** SQL ที่ generate ใช้ `to_tsvector('simple', ...)` ไม่ใช่ `'english'` ทั้งที่ตัวอย่าง
> ใน Step 671 ใช้ `'english'` — เพราะ **ค่าเริ่มต้นของ `pg_search` คือ dictionary `"simple"`
> ซึ่งไม่ทำ stemming เลย** ต้องระบุ `dictionary: "english"` เองถ้าต้องการ stemming (รายละเอียด
> เต็มใน Step 675) นี่คือรายละเอียดที่มือใหม่มักพลาดแล้วงงว่าทำไม stemming ไม่ทำงาน

---

## Step 674: ค้นหาหลายคอลัมน์และข้าม Association

### ค้นหาหลายคอลัมน์พร้อมกันด้วย `against: [...]`

โจทย์จริงแทบทุกระบบต้องค้นทั้ง title และ body พร้อมกัน ไม่ใช่แค่ title เดี่ยวๆ:

```ruby
# app/models/post.rb
pg_search_scope :search_full_text, against: %i[title body]
```

```irb
irb> Post.search_full_text("full-text search").map(&:title)
=> ["Understanding Full-Text Search in PostgreSQL",
    "Ruby on Rails Performance Tuning"]
```

บทความที่สองไม่มีคำว่า "full-text search" ใน title เลย แต่มีในคำว่า "search performance" ที่
body — `against: %i[title body]` ทำให้ `pg_search` รวมทั้งสองคอลัมน์เป็น `tsvector` เดียวกัน
ก่อนค้นหา (ใช้ operator `||` ต่อ tsvector สองอันเข้าด้วยกันใน SQL ที่ generate)

**ข้อสังเกตสำคัญ:** ค่าเริ่มต้นของ `pg_search` คือ query ต้องมี **ทุกคำ** ที่ระบุ (AND) ไม่ใช่
คำใดคำหนึ่งก็พอ (OR):

```irb
irb> Post.search_full_text("Ruby PostgreSQL").map(&:title)
=> []   # ไม่มีบทความไหนมีทั้ง "Ruby" และ "PostgreSQL" พร้อมกันในคอลัมน์เดียวกัน (นับรวม title+body)
```

(จะเปลี่ยนพฤติกรรมนี้ด้วย option `any_word: true` ใน Step 675)

### ค้นหาข้าม Association ด้วย `associated_against`

โจทย์ที่พบบ่อยกว่านั้นคือ "ค้นหาบทความจากชื่อผู้เขียนด้วย" ทั้งที่ `name` อยู่คนละตาราง
(`authors`) ไม่ใช่ `posts` — `pg_search` รองรับผ่าน `associated_against`:

```ruby
# app/models/post.rb
pg_search_scope :search_with_author,
                 against: %i[title body],
                 associated_against: { author: [:name] }
```

```irb
irb> Post.search_with_author("Jane").map { |p| "#{p.title} (by #{p.author.name})" }
=> ["Understanding Full-Text Search in PostgreSQL (by Jane Smith)",
    "Ruby on Rails Performance Tuning (by Jane Smith)",
    "Introduction to PostgreSQL Extensions (by Jane Smith)"]
```

ค้นหาคำว่า `"Jane"` ที่ไม่ปรากฏใน `title`/`body` ของบทความเลยแม้แต่ตัวเดียว แต่เจอครบทั้ง 3
บทความของ Jane Smith เพราะ `associated_against` ไป join ตาราง `authors` เข้ามาด้วย ดู SQL ที่
generate จะเห็นว่า `pg_search` สร้าง `LEFT OUTER JOIN` ไปหา `authors` แล้วรวม `tsvector` ของ
`authors.name` เข้ากับของ `posts.title`/`posts.body` ก่อนคำนวณ rank เดียวกันทั้งหมด — ไม่ต้อง
เขียน join เองเลย และผลลัพธ์ก็ยังเป็น `Post` records ตามปกติ (ไม่ใช่ `Author`)

> **ข้อควรระวังเรื่อง N+1:** `search_with_author` คืนค่าเป็น `Post` ธรรมดา ถ้าจะเข้าถึง
> `post.author.name` ต่อในลูป (เหมือนตัวอย่างข้างบน) ควร `.includes(:author)` ต่อท้ายเพื่อเลี่ยง
> N+1 ตามหลักการ Part 034/064 เดิม: `Post.search_with_author("Jane").includes(:author)`

---

## Step 675: ปรับแต่งการค้นหา — `prefix`, `dictionary`, `negation`, `any_word`

ก่อนลงรายละเอียด option ต่างๆ ต้องเข้าใจก่อนว่า `pg_search` รองรับ **กลไกการค้นหา 3 แบบ**
(เรียกว่า `:using` ใน config) ซึ่งแต่ละแบบมีจุดแข็ง/จุดอ่อนต่างกันโดยสิ้นเชิง:

| กลไก (`using:`) | ทำงานอย่างไร | ต้องมี extension | จุดเด่น | จุดอ่อน |
|---|---|---|---|---|
| `:tsearch` (ค่าเริ่มต้น) | ใช้ `tsvector`/`tsquery` ของ PostgreSQL — ตัดคำตามขอบเขต **ช่องว่าง/เครื่องหมายวรรคตอน** แล้ว stem ตามภาษา | ไม่ต้อง (core PostgreSQL) | เร็ว, มี ranking ในตัว, รองรับ stemming/prefix/negation | ต้องมีขอบเขตคำที่ชัดเจน (ปัญหากับภาษาที่ไม่เว้นวรรคระหว่างคำ เช่นไทย — ดู Step 676) |
| `:trigram` | เทียบ **substring 3 ตัวอักษร** (trigram) ที่ overlap กันระหว่าง query กับข้อความ | `pg_trgm` | ไม่สนใจขอบเขตคำเลย ทนต่อคำผิด/สะกดต่าง ใช้ได้กับทุกภาษารวมถึงไทย | ไม่มี ranking ที่แม่นยำแบบ tsearch, query ยาวๆ ต้องคำนวณ similarity ทุกแถว |
| `:dmetaphone` | แปลงคำเป็นรหัสเสียง (phonetic code) แล้วเทียบเสียงที่ใกล้เคียงกัน | `fuzzystrmatch` | หาคำที่ "ออกเสียงคล้ายกัน" แม้สะกดต่างกันมาก (เช่น "Smith"/"Smyth") | ออกแบบมาสำหรับภาษาอังกฤษล้วน ใช้กับภาษาไทยไม่ได้เลย |

Step นี้เจาะ `:tsearch` (ค่าเริ่มต้น) ให้ครบก่อน ส่วน `:trigram` จะเจาะลึกใน Step 677 และ
`:dmetaphone` จะพูดถึงสั้นๆ ท้าย Step นี้

### `prefix: true` — ค้นหาแบบ "พิมพ์ไม่ครบคำ" (คล้าย autocomplete)

ปกติ `tsearch` ต้องพิมพ์คำให้ครบ (เทียบ token ต่อ token) แต่บางหน้า เช่น search box ที่ยิง
request ทุกครั้งที่พิมพ์ ต้องการให้เจอผลลัพธ์แม้พิมพ์ยังไม่ครบคำ:

```ruby
pg_search_scope :search_prefix,
                 against: %i[title body],
                 using: { tsearch: { prefix: true } }
```

```irb
irb> Post.search_prefix("Postgre").map(&:title)   # พิมพ์แค่ "Postgre" ไม่ครบคำ "PostgreSQL"
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu",
    "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
```

ไม่ใส่ `prefix: true` ผลลัพธ์จะว่างเปล่า เพราะ `"Postgre"` ไม่ตรงกับ token `"postgresql"`
ครบคำ — `prefix: true` เปลี่ยน `to_tsquery` ให้ค้นหาแบบ "ขึ้นต้นด้วย" แทน

### `dictionary:` — เปิด/ปิด Stemming และเลือกภาษา

ตามที่สังเกตไว้ท้าย Step 673 ค่าเริ่มต้นของ `pg_search` คือ dictionary `"simple"` ซึ่ง **ไม่ทำ
stemming เลย** ต้องระบุ `dictionary: "english"` เองถ้าต้องการให้คำที่ผันรูปต่างกัน match กัน:

```ruby
pg_search_scope :search_stemmed_en,
                 against: :body,
                 using: { tsearch: { dictionary: "english" } }
```

```irb
irb> # body มีคำว่า "optimize" — ลองค้นด้วยรูปคำอื่น
irb> Post.search_stemmed_en("optimizing").map(&:title)
=> ["Ruby on Rails Performance Tuning"]

irb> # เทียบกับ scope เดิมที่ไม่ระบุ dictionary (ใช้ "simple" ค่าเริ่มต้น)
irb> Post.search_full_text("optimizing").map(&:title)
=> []   # ไม่เจอ! เพราะ "simple" ไม่ stem คำให้
```

นี่คือ**กับดักที่พบบ่อยที่สุด**ของมือใหม่ที่ใช้ `pg_search` — ตั้ง scope ธรรมดาไม่ระบุ
`dictionary` แล้วงงว่าทำไม stemming (ที่เรียนไว้ใน Step 671) ไม่ทำงานเลย คำตอบคือค่าเริ่มต้น
"simple" ตั้งใจไม่ stem อะไร เพื่อไม่ให้ over-match ในภาษาที่ dictionary ภาษาอังกฤษไม่รองรับ —
ต้องเลือกเองว่าจะใช้ `"english"` (มี stemming + stop words ภาษาอังกฤษ) หรือ `"simple"`
(ตัดคำอย่างเดียว ไม่ stem ไม่ตัด stop word)

### `negation: true` — ค้นหาแบบ "ต้องมีคำนี้ แต่ห้ามมีคำนั้น"

```ruby
pg_search_scope :search_negation,
                 against: %i[title body],
                 using: { tsearch: { negation: true } }
```

เติม `!` นำหน้าคำที่ต้องการ "ห้ามมี" ในคำค้นหา:

```irb
irb> Post.search_negation("PostgreSQL !Ubuntu").map(&:title)
=> ["Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
# บทความ "วิธีติดตั้ง PostgreSQL บน Ubuntu" ถูกตัดออกเพราะมีคำว่า Ubuntu อยู่

irb> # เทียบกับไม่ใช้ negation — ค้น "PostgreSQL Ubuntu" ต้องมีทั้งสองคำ (AND ตามปกติ)
irb> Post.search_full_text("PostgreSQL Ubuntu").map(&:title)
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu"]
```

### `any_word: true` — เปลี่ยนจาก AND (ค่าเริ่มต้น) เป็น OR

```ruby
pg_search_scope :search_any_word,
                 against: :body,
                 using: { tsearch: { any_word: true } }
```

```irb
irb> Post.search_any_word("Ruby PostgreSQL").map(&:title)
=> ["Ruby on Rails คืออะไร", "การเขียนโปรแกรมภาษาไทยเบื้องต้น",
    "วิธีติดตั้ง PostgreSQL บน Ubuntu", "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
# ได้ผลลัพธ์เยอะขึ้นมาก เพราะแค่มีคำใดคำหนึ่ง (Ruby หรือ PostgreSQL) ก็ผ่านแล้ว
```

### `:dmetaphone` — ค้นหาแบบ "เสียงคล้ายกัน" (สั้นๆ)

`:dmetaphone` ต้อง enable extension `fuzzystrmatch` และรัน generator เฉพาะของ `pg_search`
เพื่อสร้าง SQL function เสริม:

```bash
bin/rails generate migration EnableFuzzystrmatch
```

```ruby
class EnableFuzzystrmatch < ActiveRecord::Migration[8.1]
  def change
    enable_extension "fuzzystrmatch"
  end
end
```

```bash
bin/rails g pg_search:migration:dmetaphone
bin/rails db:migrate
```

```ruby
pg_search_scope :search_soundalike, against: :title, using: :dmetaphone
```

```irb
irb> Post.search_soundalike("Poastgreskuel").map(&:title)   # สะกดผิดเยอะมาก แต่ออกเสียงคล้าย PostgreSQL
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu",
    "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
```

ใช้งานได้จริงกับคำภาษาอังกฤษที่สะกดผิดแบบ "เสียงเหมือนกัน" แต่ **`:dmetaphone` ออกแบบมาสำหรับ
สัทศาสตร์ภาษาอังกฤษล้วนๆ ใช้กับคำไทยไม่ได้เลย** — เป็นอีกเหตุผลที่ Step ถัดไปต้องพูดถึงข้อจำกัด
เรื่องภาษาไทยอย่างจริงจัง

---

## Step 676: ข้อจำกัดสำคัญ — PostgreSQL Full-text Search กับภาษาไทยไปด้วยกันไม่ค่อยได้

> **นี่คือ Step ที่สำคัญที่สุดของ Part นี้สำหรับคอร์สภาษาไทย** ทุก Step ก่อนหน้านี้สาธิตด้วย
> ข้อความภาษาอังกฤษเป็นหลักไม่ใช่เรื่องบังเอิญ — เพราะ `to_tsvector`/`to_tsquery` (กลไก
> `:tsearch` ทั้งหมด) **ตัดคำ (tokenize) ตามช่องว่างและเครื่องหมายวรรคตอนเป็นหลัก** ซึ่งภาษาไทย
> **ไม่เว้นวรรคระหว่างคำ** ทำให้ PostgreSQL มองประโยคไทยทั้งประโยคเป็น "คำเดียว" ไม่ใช่หลายคำ

### พิสูจน์ด้วยของจริง: `to_tsvector` กับประโยคไทย

```ruby
title = "การเขียนโปรแกรมภาษาไทยเบื้องต้น"

ActiveRecord::Base.connection.execute(
  ActiveRecord::Base.sanitize_sql(["SELECT to_tsvector('simple', ?) AS tsv", title])
).first["tsv"]
# => "'การเขียนโปรแกรมภาษาไทยเบื้องต้น':1"
```

เทียบกับ dictionary `"english"` (ผลเหมือนกันเป๊ะ เพราะปัญหาไม่ได้อยู่ที่ stemming แต่อยู่ที่
tokenization):

```ruby
ActiveRecord::Base.connection.execute(
  ActiveRecord::Base.sanitize_sql(["SELECT to_tsvector('english', ?) AS tsv", title])
).first["tsv"]
# => "'การเขียนโปรแกรมภาษาไทยเบื้องต้น':1"
```

เทียบกับประโยคภาษาอังกฤษความยาวใกล้เคียงกัน:

```ruby
ActiveRecord::Base.connection.execute(
  ActiveRecord::Base.sanitize_sql(
    ["SELECT to_tsvector('english', ?) AS tsv", "Understanding Full-Text Search in PostgreSQL"]
  )
).first["tsv"]
# => "'full':3 'full-text':2 'postgresql':7 'search':5 'text':4 'understand':1"
```

ประโยคอังกฤษถูกตัดเป็น **6 token** ส่วนประโยคไทยความยาวใกล้เคียงกันกลายเป็น **1 token เดียว**
ทั้งประโยค — เพราะ PostgreSQL ไม่รู้ว่าตรงไหนคือขอบเขตของแต่ละคำในภาษาไทย (ไม่มีช่องว่างให้อ้างอิง)

### ผลที่ตามมา: ค้นหาบางส่วนของประโยคไม่เจอ

```irb
irb> Post.search_by_title("การเขียนโปรแกรมภาษาไทยเบื้องต้น").map(&:title)   # พิมพ์ครบทั้งประโยค
=> ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]   # เจอ! เพราะ token ตรงกันเป๊ะทั้งก้อน

irb> Post.search_by_title("เขียนโปรแกรม").map(&:title)   # คำที่เป็นส่วนหนึ่งของ title จริงๆ
=> []   # ไม่เจอเลย!

irb> Post.search_by_title("โปรแกรม").map(&:title)   # คำเดียวสั้นๆ ที่อยู่ในประโยคจริงๆ
=> []   # ไม่เจอเช่นกัน
```

พฤติกรรมนี้ตรงข้ามกับที่ผู้ใช้ทั่วไปคาดหวังโดยสิ้นเชิง — ผู้ใช้พิมพ์คำค้นหาเป็นคำสั้นๆ (ไม่ใช่
ทั้งประโยค) แล้วคาดหวังว่าจะเจอบทความที่มีคำนั้นอยู่ แต่ `:tsearch` มาตรฐานของ PostgreSQL ต้อง
match "token" ทั้งก้อน และ token ของภาษาไทยที่ไม่ได้ตั้งค่าพิเศษก็คือทั้งประโยคเป๊ะๆ — เท่ากับว่า
**ฟีเจอร์ค้นหาที่ใช้งานได้แค่กรณีผู้ใช้พิมพ์ประโยคซ้ำเป๊ะเท่านั้น ซึ่งไม่มีประโยชน์ในทางปฏิบัติ**

### ทำไมถึงเป็นแบบนี้ (สรุปสาเหตุให้ชัด)

PostgreSQL ตัดคำผ่านกลไกที่เรียกว่า **text search parser** ซึ่งมีกฎที่ hardcode ไว้ตั้งแต่ระดับ
C (ไม่ใช่ dictionary ธรรมดาที่สลับได้ง่ายๆ) กฎเหล่านี้ออกแบบมาสำหรับภาษาในตระกูลที่ **เว้นวรรค
ระหว่างคำ** (ภาษาอังกฤษ ภาษาในยุโรปส่วนใหญ่) ภาษาในตระกูลที่ไม่เว้นวรรคระหว่างคำ เช่น
**ไทย จีน ญี่ปุ่น** ต้องใช้อัลกอริทึมการตัดคำที่ซับซ้อนกว่านั้นมาก (ต้องรู้จักคำศัพท์ในภาษานั้น
หรือใช้ machine learning ช่วยตัดคำ) ซึ่ง **PostgreSQL core ไม่มีให้มาเลย** ต้องพึ่ง extension
ภายนอกที่ไม่ได้มาตรฐานและซับซ้อนกว่ามากในการติดตั้ง/ดูแล

### ทางออกที่มี (ภาพรวม — รายละเอียดใน Step ถัดไปและ Part 069)

1. **Trigram search (`pg_trgm`)** — เปลี่ยนวิธีคิดจาก "ตัดเป็นคำ" เป็น "ตัดเป็นชิ้นตัวอักษร 3
   ตัวติดกัน" (character-based ไม่ใช่ word-based) วิธีนี้ **ไม่สนใจภาษาเลย** เพราะทำงานระดับ
   ตัวอักษรล้วนๆ จึงใช้ได้กับภาษาไทยได้ในระดับที่ "พอใช้งานได้จริง" แม้จะไม่สมบูรณ์แบบเท่า
   full-text search ของภาษาที่รองรับดี — นี่คือทางออกที่ทำได้ทันทีด้วย PostgreSQL ล้วนๆ
   ไม่ต้องพึ่งระบบภายนอก (รายละเอียดเต็ม Step 677)
2. **Elasticsearch/OpenSearch** — มี analyzer/tokenizer เฉพาะสำหรับภาษาที่ตัดคำยาก (CJK และไทย)
   ที่ดีกว่า PostgreSQL มาก เพราะออกแบบมาเพื่อ full-text search โดยเฉพาะและมี ecosystem ปลั๊กอิน
   ตัดคำหลายภาษา — เป็นทางออกที่ "ถูกต้องกว่า" สำหรับระบบที่ค้นหาภาษาไทยเป็นหลักและข้อมูลเยอะจริง
   จัง (**Part 069 จะเจาะลึกเรื่องนี้เต็มรูปแบบ**)

---

## Step 677: ทางออกที่ไทยเป็นมิตรกว่า — Trigram Search ด้วย `pg_trgm`

### Trigram คืออะไร

Trigram คือการตัดข้อความเป็นชิ้นย่อยทีละ **3 ตัวอักษรติดกัน** (ไม่สนใจว่าตรงนั้นเป็นขอบเขตคำ
หรือไม่) เช่นคำว่า `"เขียน"` ตัดเป็น: `"  เ"`, `" เข"`, `"เขี"`, `"ขีย"`, `"ียน"`, `"ยน "` (มี
ช่องว่างขนาบหัวท้ายเสมอ) การเทียบสองข้อความว่า "คล้ายกันแค่ไหน" (similarity) คือการดูว่า
trigram ของทั้งสองฝั่ง overlap กันกี่เปอร์เซ็นต์ — วิธีนี้ **ไม่ต้องรู้จักคำศัพท์ในภาษาเลย**
เป็น string algorithm ล้วนๆ จึงใช้ได้กับทุกภาษารวมถึงไทย

### เปิดใช้งาน Extension `pg_trgm`

```bash
bin/rails generate migration EnablePgTrgm
```

```ruby
class EnablePgTrgm < ActiveRecord::Migration[8.1]
  def change
    enable_extension "pg_trgm"
  end
end
```

```bash
bin/rails db:migrate
```

### สร้าง Trigram Scope

```ruby
# app/models/post.rb
pg_search_scope :search_trigram_title,
                 against: :title,
                 using: :trigram
```

```irb
irb> Post.search_trigram_title("เขียนโปรแกรม").map(&:title)
=> []   # !! ยังว่างอยู่ แม้ใช้ trigram แล้ว
```

ทำไมยังว่างอยู่? เพราะ **threshold เริ่มต้นของ trigram search คือ 0.3** (ต้อง similarity อย่าง
น้อย 30%) ลองเช็ค similarity จริงด้วย SQL function `similarity()` ที่มากับ `pg_trgm`:

```sql
SELECT title, similarity(title, 'เขียนโปรแกรม') AS sim
FROM posts ORDER BY sim DESC;
```

```
                    title                      |    sim
------------------------------------------------+-----------
 การเขียนโปรแกรมภาษาไทยเบื้องต้น                  | 0.2857143
 Ruby on Rails คืออะไร                          |         0
 ...
```

Similarity ได้ **0.2857** ซึ่ง**ต่ำกว่า threshold 0.3 นิดเดียว** จึงถูกกรองทิ้งไป ต้องปรับ
threshold ให้ต่ำลงผ่าน option:

```ruby
pg_search_scope :search_trigram_title,
                 against: :title,
                 using: { trigram: { threshold: 0.15 } }
```

```irb
irb> Post.search_trigram_title("เขียนโปรแกรม").map(&:title)
=> ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]   # เจอแล้ว!
```

### พิสูจน์ว่าทนต่อคำผิด/สะกดต่างกันได้จริง (ทั้งไทยและอังกฤษ)

```irb
irb> # พิมพ์ผิด "แกรม" เป็น "แกลม" (ม↔ล)
irb> Post.search_trigram_title("เขียนโปรแกลม").map(&:title)
=> ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]   # ยังเจอ! เพราะ trigram ส่วนใหญ่ยัง overlap กันอยู่

irb> # ภาษาอังกฤษ พิมพ์ขาดตัวอักษร "PostgeSQL" (ขาด r)
irb> Post.search_trigram_title("PostgeSQL").map(&:title)
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu",
    "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]

irb> # เทียบกับ tsearch ปกติ — พิมพ์ผิดแบบเดียวกันแล้วหาไม่เจอเลย
irb> Post.search_by_title("PostgeSQL")
=> []
```

นี่คือจุดแข็งที่สุดของ trigram: **ทนต่อคำผิดได้ในตัวโดยไม่ต้องเขียนโค้ดเพิ่ม** ต่างจาก `:tsearch`
ที่ต้องตรงคำ (หรือ prefix) เป๊ะๆ เท่านั้น

### กับดักที่ต้องระวัง: `against` หลายคอลัมน์ทำให้ Similarity เจือจางลง

ลอง `against: %i[title body]` (เหมือน scope อื่นๆ ก่อนหน้า) แทนที่จะเป็น `:title` เดี่ยวๆ:

```ruby
pg_search_scope :search_trigram, against: %i[title body], using: { trigram: { threshold: 0.15 } }
```

```irb
irb> Post.search_trigram("เขียนโปรแกรม").map(&:title)
=> []   # ว่างอีกแล้ว! ทั้งที่ threshold เดียวกับที่เพิ่งใช้ได้ผลกับ :title เดี่ยวๆ
```

เช็ค similarity เทียบกัน:

```sql
SELECT title,
       similarity(title, 'เขียนโปรแกรม') AS sim_title,
       similarity(title || ' ' || body, 'เขียนโปรแกรม') AS sim_combined
FROM posts ORDER BY sim_title DESC;
```

```
                    title                     | sim_title | sim_combined
-----------------------------------------------+-----------+--------------
 การเขียนโปรแกรมภาษาไทยเบื้องต้น                  | 0.2857143 |  0.104166664
```

**สาเหตุ:** similarity คำนวณจากสัดส่วน trigram ที่ overlap เทียบกับ trigram **ทั้งหมด** ของฝั่ง
ที่เอามาเทียบ ยิ่งข้อความยาว (title ต่อกับ body ที่ยาวกว่ามาก) จำนวน trigram รวมก็ยิ่งเยอะขึ้น
ทำให้สัดส่วนที่ match ได้ (คำค้นหาสั้นๆ) ดู "เล็กลง" เมื่อเทียบกับส่วนที่ไม่ match ทั้งหมด —
similarity จาก 0.2857 (เทียบกับ title อย่างเดียว) ลดเหลือ 0.104 (เทียบกับ title+body รวมกัน)

**แนวทางแก้ในทางปฏิบัติ:**

1. **จำกัด trigram search ไว้ที่คอลัมน์สั้นๆ ที่มีความหมายชัด** เช่น `title`, `name`, `sku`
   แทนที่จะโยนทุกคอลัมน์รวมกัน (คอลัมน์ยาวอย่าง `body` เหมาะกับ `:tsearch` มากกว่า)
2. **ถ้าจำเป็นต้องรวมหลายคอลัมน์จริงๆ ให้ลด threshold ลงอีก** (เช่น 0.05–0.1) แล้วทดสอบกับข้อมูล
   จริงของตัวเองว่า false positive (ผลลัพธ์ที่ไม่เกี่ยวข้องแต่ผ่าน threshold) เยอะเกินไปหรือไม่

---

## Step 678: เรียงผลลัพธ์ตาม Relevance — `pg_search_rank` และ `ranked_by`

### เข้าถึงคะแนน Rank ด้วย `with_pg_search_rank`

`pg_search` คำนวณ rank ให้ทุก scope อยู่แล้ว (ใช้เรียงลำดับผลลัพธ์เป็นค่าเริ่มต้น) แต่ **ใน
pg_search 2.x ต้องเรียก `.with_pg_search_rank` ต่อท้ายก่อน** ถึงจะอ่านค่า `pg_search_rank`
บน record แต่ละตัวได้ ไม่งั้นจะได้ error:

```irb
irb> Post.search_full_text("PostgreSQL").first.pg_search_rank
# => PgSearch::PgSearchRankNotSelected:
#    You must chain .with_pg_search_rank after the pg_search_scope to access
#    the pg_search_rank attribute on returned records

irb> Post.search_full_text("PostgreSQL full-text search").with_pg_search_rank.each do |post|
irb*   puts "#{post.title} => rank=#{post.pg_search_rank.round(5)}"
irb* end
Understanding Full-Text Search in PostgreSQL => rank=0.96791
```

ค่า rank สูง (เข้าใกล้ 1) หมายถึงเกี่ยวข้องมาก — ในหน้าเว็บจริงมักใช้ค่านี้แสดงเป็น "ความ
เกี่ยวข้อง" หรือใช้ตัดสินใจว่าจะแสดงผลลัพธ์กี่รายการ (ตัดผลลัพธ์ที่ rank ต่ำเกินไปทิ้ง)

### ปัญหา: Rank ของ Trigram Search กลับเป็น 0 เสมอ (แม้ผลลัพธ์ถูกต้อง)

ลองอ่าน rank ของ `search_trigram_title` ที่สร้างไว้ใน Step 677:

```irb
irb> Post.search_trigram_title("เขียนโปรแกรม").with_pg_search_rank.each do |post|
irb*   puts "#{post.title} => rank=#{post.pg_search_rank}"
irb* end
การเขียนโปรแกรมภาษาไทยเบื้องต้น => rank=0.0
```

ผลลัพธ์ถูกต้อง (เจอบทความที่ถูกต้อง) แต่ **rank กลับเป็น 0.0** ทั้งที่ similarity จริงคือ 0.2857
ตรวจสอบ SQL ที่ generate จะพบสาเหตุ:

```sql
-- ส่วน rank ที่ pg_search generate ให้ scope ที่ using: :trigram (แบบไม่ปรับแต่ง)
(ts_rank(
  (to_tsvector('simple', coalesce(cast("posts"."title" AS text), ''))),
  (to_tsquery('simple', ''' ' || 'เขียนโปรแกรม' || ' ''')), 0
)) AS rank
```

**สาเหตุ:** ค่าเริ่มต้าน `pg_search` คำนวณ **rank** ด้วย `ts_rank`/`to_tsquery` เสมอ (กลไก
`:tsearch`) ไม่ว่าจะตั้ง `using: :trigram` เพื่อ**กรอง**ผลลัพธ์ก็ตาม — การกรอง (filter, ส่วน
`WHERE`) กับการเรียงคะแนน (rank, ส่วน `ORDER BY`) เป็นคนละกลไกกันใน `pg_search` ถ้าไม่สั่งเจาะจง
มันจะกรองด้วย `similarity()` (ตามที่ตั้ง `using:` ไว้) แต่ยังไปคำนวณ rank ด้วย `ts_rank`
เหมือนเดิม ซึ่งสำหรับคำค้นหาที่เป็นสับเซ็ตของ token เดียวยาวๆ (ซึ่งเป็นกรณีปกติของภาษาไทยตาม
Step 676) `to_tsquery` จะไม่ match เป๊ะกับ token เต็ม จึงได้ rank = 0 เสมอ

### แก้ด้วย `ranked_by:` — สั่งให้ใช้ Similarity เป็นคะแนน Rank โดยตรง

```ruby
pg_search_scope :search_trigram_ranked,
                 against: :title,
                 using: { trigram: { threshold: 0.15 } },
                 ranked_by: ":trigram"
```

```irb
irb> Post.search_trigram_ranked("เขียนโปรแกรม").with_pg_search_rank.each do |post|
irb*   puts "#{post.title} => rank=#{post.pg_search_rank.round(4)}"
irb* end
การเขียนโปรแกรมภาษาไทยเบื้องต้น => rank=0.2857
```

ตอนนี้ rank ตรงกับค่า similarity จริง (0.2857) แล้ว — `":trigram"` เป็น placeholder พิเศษที่
`pg_search` แปลงเป็น expression `similarity(...)` ให้อัตโนมัติ **บทเรียนสำคัญ:** เวลาใช้
`using: :trigram` (หรือ `:dmetaphone`) เพื่อค้นหา **ควรตั้ง `ranked_by:` ให้ตรงกับกลไกที่ใช้กรอง
เสมอ** ไม่งั้นค่า rank ที่ได้จะไม่มีความหมายอะไรเลย แม้ตัวผลลัพธ์ (records ที่คืนมา) จะถูกต้อง

### รวมหลายกลไกเข้าด้วยกัน (Bonus: ความรู้ต่อยอด)

`pg_search` รองรับ `using:` เป็น array และ `ranked_by:` เป็น expression ผสมได้ เช่น ใช้
`:tsearch` เป็นตัวหลัก (มี ranking แม่นยำ) เสริมด้วย `:trigram` (ทนคำผิด) แล้วผสมคะแนนสองฝั่ง:

```ruby
pg_search_scope :search_combo,
                 against: %i[title body],
                 using: {
                   tsearch: { dictionary: "english", prefix: true },
                   trigram: { threshold: 0.2 }
                 },
                 ranked_by: ":tsearch + (0.5 * :trigram)"
```

```irb
irb> # "Postgres" ไม่ครบคำ "PostgreSQL" — tsearch (prefix) หาไม่เจอเป๊ะ แต่ trigram ช่วยดันขึ้นมา
irb> Post.search_combo("Postgres").with_pg_search_rank.each { |p| puts "#{p.title} => #{p.pg_search_rank.round(4)}" }
Understanding Full-Text Search in PostgreSQL => 0.1193
Introduction to PostgreSQL Extensions => 0.1177
วิธีติดตั้ง PostgreSQL บน Ubuntu => 0.1164
```

เทคนิคนี้เหมาะกับระบบที่มีทั้งเนื้อหาภาษาอังกฤษปริมาณมาก (อยากได้ ranking แม่นยำจาก `:tsearch`)
และอยากให้ทนต่อคำพิมพ์ผิดไปพร้อมกัน — ใช้จริงต้อง benchmark กับข้อมูลจริงของตัวเองเสมอว่า
สัดส่วนถ่วงน้ำหนัก (`0.5` ในตัวอย่าง) เหมาะสมหรือไม่

---

## Step 679: รวม pg_search กับ ActiveRecord Scope/Pagination และเตรียมพร้อมสำหรับตอนสเกล

### `pg_search_scope` คืน `ActiveRecord::Relation` — Chain ต่อได้เหมือน Scope ทั่วไป

ทุก scope ที่สร้างมาทั้ง Part นี้ทำงานร่วมกับเทคนิคจาก Part 034 (Query Interface) และ Part 038
(Pagination) ได้ทันที เพราะผลลัพธ์เป็น `ActiveRecord::Relation` ธรรมดา:

```irb
irb> scope = Post.search_full_text("PostgreSQL")
irb*          .with_pg_search_rank
irb*          .where.not(id: nil)   # เงื่อนไข ActiveRecord ปกติ เช่น .where(published: true) ในระบบจริง
irb*          .limit(2)
irb> scope.map { |p| "#{p.title} (rank=#{p.pg_search_rank.round(4)})" }
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu (rank=0.076)",
    "Understanding Full-Text Search in PostgreSQL (rank=0.076)"]
```

ลำดับผลลัพธ์ยังคงเรียงตาม rank (มาจาก `pg_search_scope`) ก่อน แล้วค่อย filter/limit ทับด้วย
`ActiveRecord` เหมือนเดิม — ไม่มีความขัดแย้งกันเลย

### Pagination ด้วย Pagy (ตามรูปแบบ Part 038)

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  include Pagy::Backend

  def index
    scope = params[:q].present? ? Post.search_full_text(params[:q]) : Post.all
    @pagy, @posts = pagy(scope, limit: 10)
  end
end
```

`pagy` ทำงานกับผลลัพธ์จาก `pg_search_scope` ได้เหมือนกับ `ActiveRecord::Relation` ทั่วไปทุก
ประการ — คำนวณ `LIMIT`/`OFFSET` ทับ subquery ที่ `pg_search` สร้างไว้ ไม่ต้องเขียนอะไรพิเศษ
เพิ่ม (ดูตัวอย่างการเทียบ page ด้วย `.limit`/`.offset` ตรงๆ):

```irb
irb> page_size = 2
irb> page1 = Post.search_full_text("PostgreSQL").limit(page_size).offset(0)
irb> page2 = Post.search_full_text("PostgreSQL").limit(page_size).offset(page_size)
irb> page1.map(&:title)
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu", "Understanding Full-Text Search in PostgreSQL"]
irb> page2.map(&:title)
=> ["Introduction to PostgreSQL Extensions"]
```

### เตรียมพร้อมสำหรับตอนสเกล: Generated `tsvector` Column + GIN Index

`pg_search_scope` แบบพื้นฐานที่ใช้มาตลอด Part นี้ (`against: :title` เฉยๆ) จะคำนวณ
`to_tsvector(...)` **สดใหม่ทุกครั้งที่ query** — บนตารางเล็กไม่มีปัญหา แต่ตารางระดับล้านแถว
(แบบที่ Part 064 สาธิตไว้กับเรื่อง index) การคำนวณ `to_tsvector` ทุกแถวทุกครั้งที่ค้นหาคือ
ภาระที่หนักมาก ทางแก้คือสร้าง **generated column ชนิด `tsvector` แบบ stored** แล้วสร้าง
**GIN index** ทับคอลัมน์นั้น (แทนที่จะคำนวณสดทุกครั้ง):

```bash
bin/rails generate migration AddSearchVectorToPosts
```

```ruby
class AddSearchVectorToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :search_vector, :tsvector,
               as: "to_tsvector('simple', coalesce(title, '') || ' ' || coalesce(body, ''))",
               stored: true

    add_index :posts, :search_vector, using: :gin
  end
end
```

```bash
bin/rails db:migrate
```

`add_column ... as: "...", stored: true` คือ syntax ของ Rails 8 สำหรับสร้าง **PostgreSQL
generated column** (คำนวณค่าอัตโนมัติจากคอลัมน์อื่นทุกครั้งที่ insert/update ไม่ต้องเขียนโค้ด
Ruby มา sync เอง) — `schema.rb` จะแสดงผลเป็น:

```ruby
create_table "posts", force: :cascade do |t|
  t.string "title"
  t.text "body"
  t.bigint "author_id", null: false
  t.virtual "search_vector", type: :tsvector,
            as: "to_tsvector('simple'::regconfig, ...)", stored: true
  t.index ["search_vector"], name: "index_posts_on_search_vector", using: :gin
end
```

ตั้งค่า `pg_search_scope` ให้ใช้คอลัมน์นี้แทนการคำนวณสดด้วย option `tsvector_column`:

```ruby
pg_search_scope :search_indexed,
                 against: %i[title body],
                 using: { tsearch: { tsvector_column: "search_vector" } }
```

```irb
irb> Post.search_indexed("PostgreSQL").map(&:title)
=> ["วิธีติดตั้ง PostgreSQL บน Ubuntu",
    "Understanding Full-Text Search in PostgreSQL",
    "Introduction to PostgreSQL Extensions"]
```

ผลลัพธ์เหมือนเดิมทุกประการ แต่คราวนี้ SQL ที่ generate อ้างถึงคอลัมน์ `search_vector` ตรงๆ
(`WHERE search_vector @@ to_tsquery(...)`) แทนที่จะคำนวณ `to_tsvector(coalesce(title,...))`
ใหม่ทุกครั้ง — query planner จึงมีโอกาสเลือกใช้ **GIN index** ที่สร้างไว้แทน sequential scan

> **หมายเหตุตรงไปตรงมา:** บนตารางแค่ 6 แถวของเอกสารนี้ `EXPLAIN` ยังแสดง `Seq Scan` อยู่ดี
> (planner ฉลาดพอที่จะรู้ว่าตารางเล็กมาก อ่านทุกแถวเร็วกว่าเปิด index) — นี่คือหลักการเดียวกับที่
> Part 064 พิสูจน์ไว้แล้วเรื่อง cost-based query planner: **index ไม่ได้แปลว่าถูกใช้เสมอ**
> ต้องมีข้อมูลมากพอ planner ถึงจะเห็นว่าคุ้มค่า ถ้าอยากเห็นตัวเลขวัดจริงว่า GIN index ต่างจาก
> sequential scan แค่ไหนตอนสเกลเป็นแสน-ล้านแถว ย้อนกลับไปอ่าน Part 064 Step 636 ที่พิสูจน์
> หลักการเดียวกันนี้ด้วย `EXPLAIN ANALYZE` บนตาราง 1,000,000 แถวจริง — แนวทางเดียวกันนำมาใช้กับ
> `search_vector`/GIN index ของ pg_search ได้ตรงๆ ไม่ต่างกัน

---

## Step 680: แบบฝึกหัด — Post Model ที่มีทั้ง Search Scope แบบอังกฤษและแบบไทยใช้งานคู่กัน

### โจทย์

ต่อยอดโปรเจกต์ `pgsearch_demo` ที่สร้างมาตลอด Part นี้ ให้ `Post` model มี search scope
**สองแบบใช้งานคู่กัน**:

1. `search_title_en` — เหมาะกับเนื้อหาภาษาอังกฤษ ใช้ `:tsearch` + `dictionary: "english"` +
   `prefix: true` (รองรับ stemming และพิมพ์ไม่ครบคำ)
2. `search_title_th` — เหมาะกับเนื้อหาภาษาไทย ใช้ `:trigram` (threshold ที่ปรับจูนแล้ว) พร้อม
   `ranked_by: ":trigram"` เพื่อให้ rank มีความหมายจริง

แล้วเขียนสคริปต์ทดสอบที่แสดงผลลัพธ์ของทั้งสอง scope เทียบกันกับคำค้นหาชุดเดียวกัน (ทั้งไทยและ
อังกฤษ) เพื่อพิสูจน์ว่า scope ไหนเหมาะกับกรณีไหน

### เฉลย

**1) Model**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include PgSearch::Model

  belongs_to :author

  # เหมาะกับภาษาอังกฤษ: มี stemming, รองรับพิมพ์ไม่ครบคำ, ranking แม่นยำ
  pg_search_scope :search_title_en,
                   against: %i[title body],
                   using: {
                     tsearch: { dictionary: "english", prefix: true }
                   }

  # เหมาะกับภาษาไทย: ไม่สนใจขอบเขตคำ ทนคำผิด/สะกดต่าง แต่ ranking ต้องสั่งเจาะจงเอง
  pg_search_scope :search_title_th,
                   against: :title,
                   using: { trigram: { threshold: 0.15 } },
                   ranked_by: ":trigram"
end
```

**2) Migration ที่ต้องมี (เปิด `pg_trgm` และเตรียมตาราง)**

```ruby
class EnablePgTrgm < ActiveRecord::Migration[8.1]
  def change
    enable_extension "pg_trgm"
  end
end
```

**3) สคริปต์ทดสอบเทียบสอง scope**

```ruby
# script/compare_search.rb — รันด้วย: bin/rails runner script/compare_search.rb
queries = [
  "PostgreSQL",              # อังกฤษ พิมพ์ครบคำ
  "Postgre",                 # อังกฤษ พิมพ์ไม่ครบคำ (prefix)
  "optimizing",              # อังกฤษ รูปคำที่ผันไป (ต้อง stem เป็น optimize)
  "เขียนโปรแกรม",              # ไทย คำสั้นๆ ที่เป็นส่วนหนึ่งของประโยคยาว
  "การเขียนโปรแกรมภาษาไทยเบื้องต้น"  # ไทย พิมพ์ครบทั้งประโยค
]

queries.each do |q|
  puts "=" * 60
  puts "คำค้นหา: #{q.inspect}"

  en_results = Post.search_title_en(q).limit(5).map(&:title)
  th_results = Post.search_title_th(q).limit(5).map(&:title)

  puts "  search_title_en (tsearch)  => #{en_results.presence || "(ไม่พบ)"}"
  puts "  search_title_th (trigram)  => #{th_results.presence || "(ไม่พบ)"}"
end
```

**ผลลัพธ์จริงที่ได้เมื่อรันกับข้อมูล seed ของ Part นี้:**

```
============================================================
คำค้นหา: "PostgreSQL"
  search_title_en (tsearch)  => ["วิธีติดตั้ง PostgreSQL บน Ubuntu", "Understanding Full-Text Search in PostgreSQL", "Introduction to PostgreSQL Extensions"]
  search_title_th (trigram)  => ["วิธีติดตั้ง PostgreSQL บน Ubuntu", "Understanding Full-Text Search in PostgreSQL", "Introduction to PostgreSQL Extensions"]
============================================================
คำค้นหา: "Postgre"
  search_title_en (tsearch)  => ["วิธีติดตั้ง PostgreSQL บน Ubuntu", "Understanding Full-Text Search in PostgreSQL", "Introduction to PostgreSQL Extensions"]
  search_title_th (trigram)  => (ไม่พบ)
============================================================
คำค้นหา: "optimizing"
  search_title_en (tsearch)  => ["Ruby on Rails Performance Tuning"]
  search_title_th (trigram)  => (ไม่พบ)
============================================================
คำค้นหา: "เขียนโปรแกรม"
  search_title_en (tsearch)  => (ไม่พบ)
  search_title_th (trigram)  => ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]
============================================================
คำค้นหา: "การเขียนโปรแกรมภาษาไทยเบื้องต้น"
  search_title_en (tsearch)  => ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]
  search_title_th (trigram)  => ["การเขียนโปรแกรมภาษาไทยเบื้องต้น"]
```

**บทวิเคราะห์ที่ต้องอธิบายในคำตอบ (สรุปจากผลลัพธ์จริงข้างบน):**

| คำค้นหา | `search_title_en` เจอไหม | `search_title_th` เจอไหม | ทำไม |
|---|---|---|---|
| `"PostgreSQL"` | ✅ | ✅ | คำครบ ไม่มีอะไรพิเศษ ทั้งสองกลไกจับได้ |
| `"Postgre"` (prefix) | ✅ (เพราะเปิด `prefix: true`) | ❌ | trigram ไม่มีแนวคิดเรื่อง "prefix" มันเทียบ similarity ของทั้งก้อนคำ ซึ่งคำสั้นเทียบกับคำเต็มจะ similarity ต่ำ |
| `"optimizing"` (stemmed) | ✅ (เพราะเปิด `dictionary: "english"`) | ❌ | trigram ไม่รู้จัก stemming เลย เทียบตัวอักษรตรงๆ เท่านั้น `"optimizing"` กับ `"optimize"` มี trigram ทับซ้อนกันไม่พอผ่าน threshold |
| `"เขียนโปรแกรม"` (ไทย คำสั้น) | ❌ (ตามข้อจำกัด Step 676 — token เดียวยาวทั้งประโยค) | ✅ | trigram ไม่สนใจขอบเขตคำ เลยจับ substring ได้ |
| ประโยคไทยเต็ม | ✅ (token ตรงกันพอดี) | ✅ | ทั้งสองกลไกเจอ แต่ด้วยเหตุผลคนละแบบ (tsearch เพราะ exact-token match, trigram เพราะ similarity สูง) |

**สรุปให้ชัด:** ไม่มีกลไกไหน "ดีกว่า" อีกฝั่งเสมอไป — **`:tsearch` (พร้อม `dictionary` ที่ถูกต้อง)
เหมาะกับเนื้อหาภาษาอังกฤษ** เพราะได้ stemming และ ranking ที่แม่นยำ ส่วน **`:trigram` เหมาะกับ
เนื้อหาภาษาไทย** เพราะไม่ติดปัญหาการตัดคำเลย แต่แลกมาด้วยการไม่มี stemming/prefix ที่แท้จริง
และต้องปรับ `ranked_by` เองถึงจะได้ ranking ที่มีความหมาย — ระบบจริงที่มีทั้งสองภาษาปะปนกัน
ควรใช้ **ทั้งสอง scope พร้อมกัน** (หรือรวมเป็น scope เดียวด้วย `using: [:tsearch, :trigram]`
ตามที่โชว์ไว้ท้าย Step 678) แล้วเลือกแสดงผลตามภาษาที่ตรวจพบในคำค้นหา หรือรวมผลลัพธ์จากทั้งสอง
scope เข้าด้วยกันแล้ว dedupe ตาม `id`

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม GIN index เฉพาะสำหรับ trigram (`CREATE INDEX ... USING gin (title gin_trgm_ops)`) ให้
   `search_title_th` แล้วใช้ `EXPLAIN` เทียบก่อน/หลังว่า planner เลือกใช้ index หรือไม่บนตาราง
   ขนาดเล็กของ Part นี้ จากนั้นลอง seed ข้อมูลเพิ่มแบบที่ Part 064 สอน (`insert_all` เป็น batch)
   จนถึงหลักหมื่น-แสนแถว แล้วพิสูจน์ด้วยตัวเองว่า planner "เปลี่ยนใจ" มาใช้ index ตอนไหน
2. รวม `:tsearch` และ `:trigram` เข้าเป็น scope เดียวด้วย `using: [:tsearch, :trigram]` และ
   `ranked_by: ":tsearch + (0.5 * :trigram)"` ตามที่ Step 678 แนะนำไว้ท้าย Step แล้วทดสอบว่า
   คำค้นหาภาษาอังกฤษที่พิมพ์ผิดยังหาเจอ พร้อมกับยังจัดอันดับผลลัพธ์ได้สมเหตุสมผลอยู่หรือไม่
   (ลองปรับตัวเลขถ่วงน้ำหนัก `0.5` เป็นค่าอื่นดูว่าผลเปลี่ยนไปอย่างไร)
3. เพิ่ม `Comment` model ที่ `belongs_to :post` แล้วสร้าง search scope บน `Post` ที่ค้นหาข้าม
   ไปถึงเนื้อหาของ comment ด้วย `associated_against: { comments: [:body] }` จากนั้นสร้างหน้า
   `PostsController#index` จริงที่รับ `params[:q]`, ใช้ `pagy` แบ่งหน้าผลลัพธ์ตามรูปแบบ Part 038
   และแสดงคะแนน `pg_search_rank` ของแต่ละแถวในหน้าเว็บด้วย

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **`LIKE '%keyword%'` ไม่ใช่ full-text search**: ไม่มี relevance scoring (ผลลัพธ์เรียงตาม `id`
  ไม่ใช่ความเกี่ยวข้อง), ไม่รองรับคำที่ผันรูปต่างกัน, และที่สำคัญที่สุด **wildcard นำหน้าทำให้
  B-Tree index ใช้ไม่ได้เลย** — พิสูจน์จริงว่าแม้สร้าง index ให้คอลัมน์แล้ว `EXPLAIN` ก็ยังแสดง
  `Seq Scan` เสมอเมื่อใช้ `LIKE '%...%'`
- **Full-text search ที่แท้จริง** ประกอบด้วย 3 กลไก: **tokenization** (ตัดข้อความเป็นคำ),
  **stemming** (รวมรูปคำที่ผันต่างกันให้เป็นรากเดียว), และ **ranking** (คำนวณคะแนนความเกี่ยวข้อง
  ด้วย `ts_rank`) — ทั้งหมดนี้ PostgreSQL มีให้ผ่าน `tsvector`/`tsquery` อยู่แล้วในตัว
- Part นี้เป็น Part แรกของคอร์สที่ต้อง **สลับจาก SQLite ไป PostgreSQL** จริงจัง เพราะ `pg_search`
  พึ่งพาฟีเจอร์เฉพาะของ PostgreSQL ล้วนๆ — `rails new --database=postgresql` และแก้
  `config/database.yml` คือจุดเริ่มต้น
- `pg_search` gem ห่อหุ้ม SQL ที่ซับซ้อนเหล่านี้ไว้เป็น `pg_search_scope` ที่เขียนแบบ Ruby ล้วนๆ
  — `include PgSearch::Model` ก่อนเสมอ, `against:` รับได้ทั้งคอลัมน์เดียวและ array หลายคอลัมน์,
  `associated_against:` ค้นหาข้าม association ได้โดยไม่ต้องเขียน join เอง
- Config สำคัญของ `:tsearch`: **`prefix: true`** (ค้นหาแบบพิมพ์ไม่ครบคำ), **`dictionary:`**
  (ค่าเริ่มต้นคือ `"simple"` ที่**ไม่ stem อะไรเลย** ต้องระบุ `"english"` เองถึงจะได้ stemming —
  กับดักที่มือใหม่พลาดบ่อยที่สุด), **`negation: true`** (ใช้ `!คำ` เพื่อคัดคำนั้นออก),
  **`any_word: true`** (เปลี่ยนจาก AND เป็น OR)
- **ข้อจำกัดที่สำคัญที่สุดสำหรับคอร์สภาษาไทย**: `to_tsvector`/`to_tsquery` ตัดคำตามช่องว่าง/
  เครื่องหมายวรรคตอนเป็นหลัก ซึ่งภาษาไทยไม่เว้นวรรคระหว่างคำ — พิสูจน์จริงว่าประโยคไทยทั้งประโยค
  กลายเป็น **token เดียว** ทำให้ค้นหาด้วยคำสั้นๆ ที่เป็นส่วนหนึ่งของประโยคไม่เจอผลลัพธ์เลย
  (ต่างจากภาษาอังกฤษที่ตัดคำได้ปกติ)
- **Trigram search (`pg_trgm`)** เป็นทางออกที่ทำได้ทันทีด้วย PostgreSQL ล้วนๆ เพราะทำงานระดับ
  ตัวอักษร (character-based) ไม่สนใจภาษา — ใช้งานได้จริงกับภาษาไทย และทนต่อคำสะกดผิดได้ทั้งสอง
  ภาษา แต่ต้อง**ปรับ `threshold` เอง** (ค่าเริ่มต้น 0.3 อาจเข้มเกินไป) และระวังว่า `against`
  หลายคอลัมน์รวมกันจะทำให้ similarity เจือจางลง (พิสูจน์จริง: 0.2857 เทียบ title อย่างเดียว vs
  0.104 เทียบ title+body รวมกัน)
- **`pg_search_rank`** ต้อง `.with_pg_search_rank` ก่อนถึงจะอ่านค่าได้ (pg_search 2.x) และเมื่อ
  ใช้ `using: :trigram` **ต้องตั้ง `ranked_by: ":trigram"` เอง** ไม่งั้น rank จะเป็น 0 เสมอ
  (เพราะ default rank ยังคำนวณผ่าน `ts_rank`/`to_tsquery` ที่ใช้กับภาษาไทยไม่ได้อยู่ดี) —
  `ranked_by` ยังผสมหลายกลไกเข้าด้วยกันได้ เช่น `":tsearch + (0.5 * :trigram)"`
  - ผลลัพธ์จาก `pg_search_scope` เป็น `ActiveRecord::Relation` ธรรมดา ใช้ร่วมกับ `.where`,
  `.limit`, `.offset`, และ `pagy`/Kaminari (Part 038) ได้ทันทีไม่ต้องแปลงอะไรเพิ่ม — สำหรับ
  ตอนสเกล ควรสร้าง **generated `tsvector` column แบบ `stored: true` + GIN index** แล้วชี้
  `pg_search_scope` ไปที่คอลัมน์นั้นผ่าน `tsvector_column:` แทนการคำนวณสดทุกครั้ง (หลักการ
  วัดผล index เดียวกับที่ Part 064 พิสูจน์ไว้บนตาราง 1,000,000 แถว)

**ต่อไป (Part 069):** Trigram search แก้ปัญหาภาษาไทยได้แค่ "พอใช้งานได้" ไม่ใช่คำตอบที่สมบูรณ์
แบบ — Part ถัดไปจะพาไปรู้จัก **Elasticsearch/OpenSearch** ซึ่งเป็นระบบค้นหาที่ออกแบบมาเพื่อ
full-text search โดยเฉพาะ มี **analyzer/tokenizer สำหรับภาษาที่ตัดคำยาก** (รวมถึงไทยและ CJK)
ที่ดีกว่า PostgreSQL มาก จะได้เห็นการติดตั้ง Elasticsearch/OpenSearch, เชื่อมกับ Rails ผ่าน gem
อย่าง `searchkick` หรือ `elasticsearch-rails`, สร้าง index, และเปรียบเทียบผลการค้นหาภาษาไทย
ระหว่าง PostgreSQL trigram กับ Elasticsearch ตัวต่อตัวด้วยข้อมูลชุดเดียวกันกับ Part นี้
