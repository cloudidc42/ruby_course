# Part 038: Pagination (Kaminari/Pagy), Sorting, Filtering แบบมืออาชีพ

> **Step ครอบคลุมใน Part นี้:** Step 371–380
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 034 เรื่อง Query Interface มาก่อน โดยเฉพาะ `order`,
> `limit`/`offset`, และ scope — Part นี้ต่อยอดจากกลไก `LIMIT`/`OFFSET` ที่ Part 034 Step 333
> อธิบายไว้แล้วโดยตรง)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / pagy 9.4.x / kaminari 1.2.x (ทุกคำสั่ง, SQL log,
> และผลลัพธ์ HTTP ในเอกสารนี้รันจริงและ capture จริงบน Ruby 3.3.6 + Rails 8.1.4 + SQLite3)

Part 034 Step 333 บอกไว้สั้นๆ ว่า `limit`/`offset` คือกลไกพื้นฐานของ pagination และ "เทคนิค
pagination แบบมืออาชีพที่ใช้ gem อย่าง Kaminari/Pagy จะเรียนใน Part 038" — ถึงเวลานั้นแล้ว
Part นี้ยังคงใช้โดเมน `Post`/`Author`/`Category` เดียวกับ Part 034 (บทความ, ผู้เขียน, หมวดหมู่)
แต่เพิ่มมิติที่หน้า index ของเว็บแอปจริงแทบทุกหน้าต้องมี: **แบ่งหน้า (pagination)**,
**เรียงลำดับที่ผู้ใช้เลือกเองได้ (sorting)**, และ **กรองข้อมูล (filtering)** — สามเรื่องนี้ดูเหมือน
แยกกัน แต่ในโค้ด production ต้องทำงานร่วมกันในทุก request เดียวกันเสมอ

## สารบัญของ Part นี้

- Step 371: ทำไม pagination ถึงสำคัญ — อย่า render `Model.all` แบบไม่จำกัดใน production
- Step 372: ติดตั้งและใช้งาน `pagy` เบื้องต้น — `Pagy::Backend`, `Pagy::Frontend`, `pagy_nav`
- Step 373: ปรับแต่ง pagy — จำนวนรายการต่อหน้าแบบปลอดภัย, `pagy_info`, ตั้งค่า default ทั้งระบบ
- Step 374: Kaminari — ทางเลือกที่ "ครบเครื่อง" กว่า และวิธีอ่านโค้ด legacy ที่ใช้มัน
- Step 375: ปัญหาของการ sort จาก params ตรงๆ — ช่องโหว่ SQL Injection ที่มองข้ามได้ง่าย
- Step 376: Safe sorting — pattern `sort_column`/`sort_direction` แบบ allowlist
- Step 377: Sortable table header ที่รักษาค่า filter/page เดิมไว้เสมอ
- Step 378: Filter Object / Search Object — แยก logic การกรองออกจาก controller
- Step 379: รวม filter + sort + pagination เข้าด้วยกัน และ ransack ในฐานะทางเลือกสำหรับ admin
- Step 380: ทดสอบ Filter Object แบบแยกหน่วย (RSpec) — ผูกกับ Part 019

---

## เตรียมโปรเจกต์สำหรับ Part นี้

```bash
rails new pagination_demo --minimal
cd pagination_demo

bin/rails generate model Author name:string nationality:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string body:text published:boolean views:integer \
  published_at:datetime author:references category:references
bin/rails generate controller Posts index
```

แก้ migration ของ `Post` ให้ `published`/`views` มี default ระดับฐานข้อมูล (แนวทางเดียวกับ
Part 026 และ Part 034):

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
      t.references :author, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

ตั้งค่า association และ scope (`published` scope นี้เหมือนที่เรียนใน Part 034 Step 336 เป๊ะ):

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

Seed ข้อมูล 45 บทความ กระจาย 4 ผู้เขียนและ 4 หมวดหมู่เท่าๆ กัน (9 บทความเผยแพร่แล้วต่อหมวดหมู่)
เพื่อให้เห็นการแบ่งหน้า/เรียง/กรองชัดเจน:

```ruby
# db/seeds.rb
authors = {
  "Nichada" => "Thai", "Somsak" => "Thai",
  "Wanda" => "American", "Kenji" => "Japanese"
}.map { |name, nat| Author.create!(name: name, nationality: nat) }

categories = %w[Ruby Rails DevOps Career].map { |n| Category.create!(name: n) }

topics = [
  "เริ่มต้นกับ Ruby", "Metaprogramming เบื้องต้น", "Block กับ Proc",
  # ... รวม 45 หัวข้อ (ดูตัวอย่างเต็มใน Part 034 seeds.rb แล้วขยายเป็น 45 รายการ)
]

topics.each_with_index do |title, i|
  author = authors[i % authors.size]
  category = categories[i % categories.size]
  published = i % 5 != 0 # 4 ใน 5 เผยแพร่แล้ว
  Post.create!(
    title: title, body: "เนื้อหาตัวอย่างของบทความเรื่อง #{title}",
    author: author, category: category, published: published,
    views: (i * 37) % 1000,
    published_at: published ? (topics.size - i).days.ago : nil
  )
end
```

```bash
bin/rails db:seed
# Authors: 4
# Categories: 4
# Posts: 45 (published: 36)
```

```bash
bin/rails runner 'Category.all.each { |c| puts "id=#{c.id} #{c.name}: #{c.posts.published.count} published" }'
# id=1 Ruby: 9 published
# id=2 Rails: 9 published
# id=3 DevOps: 9 published
# id=4 Career: 9 published
```

เพิ่ม route:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts, only: [:index]
  root "posts#index"
end
```

---

## Step 371: ทำไม pagination ถึงสำคัญ — อย่า render `Model.all` แบบไม่จำกัดใน production

ก่อนแตะโค้ดเลยสักบรรทัด ต้องเข้าใจ "ทำไม" ให้ชัดก่อน เพราะ pagination ไม่ใช่ฟีเจอร์ตกแต่ง UI
แต่เป็น **ข้อกำหนดพื้นฐานด้าน performance และความปลอดภัย** ที่ทุก index action ระดับ production
ต้องมี

### สิ่งที่เกิดขึ้นถ้าไม่มี pagination

```ruby
require "logger"
ActiveRecord::Base.logger = Logger.new(STDOUT)

Post.published.order(created_at: :desc).to_a
```

```
Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE ORDER BY "posts"."created_at" DESC
```

ด้วยข้อมูลตัวอย่างแค่ 36 แถว คำสั่งนี้ไม่มีปัญหาอะไรเลย แต่ลองจินตนาการว่าตาราง `posts` นี้เป็น
ระบบบล็อกที่รันมา 5 ปีแล้วมี **2 ล้านแถว** — คำสั่งเดียวกันนี้จะทำสามอย่างพร้อมกัน:

1. **ฐานข้อมูลต้องอ่านทั้ง 2 ล้านแถวจากดิสก์** และเรียงลำดับทั้งหมดก่อนส่งกลับมา แม้ผู้ใช้จะ
   ต้องการดูแค่ 20 รายการแรกก็ตาม
2. **Rails ต้องสร้าง ActiveRecord object 2 ล้านตัว** ขึ้นมาใน memory ของ process เดียว — แต่ละ
   object ใช้ memory หลายร้อยไบต์ รวมกันอาจถึงหลักกิกะไบต์ ทำให้ process ใช้ RAM พุ่งขึ้นฉับพลัน
   จนกระทบ process อื่นบนเครื่องเดียวกัน หรือถูก OS kill (`OOM killed`) ก่อนที่ request จะเสร็จด้วยซ้ำ
3. **View ต้อง render HTML 2 ล้านแถว** ส่งกลับไปให้ browser — ทั้ง server และ browser ฝั่งผู้ใช้
   จะค้างหรือหน่วงอย่างรุนแรง ทั้งที่ไม่มีใครเลื่อนดูเกินหน้าที่ 2-3 อยู่แล้วในทางปฏิบัติจริง

นี่คือเหตุผลที่ **`Model.all` (หรือ scope ใดๆ ที่ไม่มี `limit`) ที่ส่งตรงเข้า view โดยไม่ผ่าน
pagination ถือเป็น anti-pattern ที่ควรถูก flag ทันทีใน code review** ไม่ว่าตอนเขียนโค้ดข้อมูลจะมี
กี่แถวก็ตาม — เพราะข้อมูลในตารางจะโตขึ้นเรื่อยๆ ตามอายุของระบบเสมอ โค้ดที่ "ใช้ได้ตอนนี้" กับข้อมูล
50 แถว จะกลายเป็นตัวทำลาย production เมื่อข้อมูลโตถึงหลักแสน-หลักล้านแถวโดยไม่มีใครแก้ไขทัน

> **ย้อนกลับไป Part 034 Step 333:** เราเรียนไปแล้วว่า `OFFSET` ที่มีค่าสูงมากช้าลงเรื่อยๆ ตามขนาด
> ตาราง (แก้ด้วย cursor-based pagination ที่จะเจาะลึกใน Part 064) — นั่นคือปัญหาของ **pagination ที่
> มีอยู่แล้วแต่ไถลไปหน้าลึกๆ** ส่วน Step นี้พูดถึงปัญหาที่ร้ายแรงกว่านั้นคือ **ไม่มี pagination เลย**
> ซึ่งพังตั้งแต่ request แรกที่ข้อมูลโตเกินจุดหนึ่ง ไม่ต้องรอให้ไถลไปหน้าลึกด้วยซ้ำ

### กฎปฏิบัติจากนี้ไป

**ทุก index action ที่ query จาก collection ที่ไม่มีขอบเขตจำนวนแน่นอน (ตาราง user, post, order,
log ฯลฯ) ต้องผ่าน pagination เสมอ ไม่มีข้อยกเว้น** — แม้แต่หน้า admin ที่ "ดูแค่คนในทีมใช้" ก็ควรมี
เพราะข้อมูลจะโตขึ้นเรื่อยๆ ตามเวลา ส่วนตารางที่รู้แน่ชัดว่าจำนวนแถวจะจำกัดตลอดไป (เช่น
`categories` ที่มีแค่ไม่กี่สิบแถวตายตัวตามการออกแบบระบบ) ไม่จำเป็นต้องมี pagination

Rails มี gem pagination หลักสองตัวที่ community ใช้กันแพร่หลาย: **pagy** (เบา เร็ว ทันสมัย ดูแล
ต่อเนื่องอย่างจริงจัง) และ **kaminari** (เก่าแก่กว่า มีของครบกว่าออกจากกล่อง) — Part นี้ใช้ **pagy
เป็นเครื่องมือหลัก** ตลอดทั้ง Part แล้วแนะนำ kaminari ไว้ใน Step 374 สำหรับตอนที่ต้องอ่านหรือ
maintain โค้ด legacy ที่เลือกใช้ตัวนั้นไปแล้ว

---

## Step 372: ติดตั้งและใช้งาน `pagy` เบื้องต้น — `Pagy::Backend`, `Pagy::Frontend`, `pagy_nav`

### ติดตั้ง

```ruby
# Gemfile
gem "pagy", "~> 9.4"
```

```bash
bundle install
```

pagy **ไม่มี install generator** เหมือน scaffold ทั่วไป (และไม่จำเป็นต้องมี) — ใช้งานได้ทันทีด้วย
การ `include` สอง module เข้าที่ที่ถูกต้อง:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pagy::Backend

  allow_browser versions: :modern
end
```

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  include Pagy::Frontend
end
```

**`Pagy::Backend`** ให้ private method ชื่อ `pagy` ที่ใช้ใน **controller** สำหรับคำนวณ pagination
จาก collection ที่ส่งเข้าไป — **`Pagy::Frontend`** ให้ helper method อย่าง `pagy_nav`/`pagy_info`
ที่ใช้ใน **view** สำหรับ render UI ของ pagination ทั้งสองฝั่งแยกกันชัดเจนตามหลัก MVC

### ใช้งานใน controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @pagy, @posts = pagy(Post.published.includes(:author, :category).order(created_at: :desc))
  end
end
```

`pagy(collection)` รับ `ActiveRecord::Relation` (จาก Part 034 Step 331 — ยังไม่ query จริงตอนนี้)
แล้วคืนค่าเป็น **สอง object**: `pagy` object (เก็บข้อมูล metadata ของการแบ่งหน้า — หน้าปัจจุบัน,
จำนวนหน้าทั้งหมด, จำนวนรายการทั้งหมด ฯลฯ) และ collection ที่ตัดมาแล้วเฉพาะหน้าปัจจุบัน (ใช้ `page`
จาก `params[:page]` ให้อัตโนมัติ)

> **สังเกตว่ามี `.includes(:author, :category)` ต่อท้าย** — นี่คือการนำความรู้จาก Part 034 Step 335
> มาใช้ป้องกัน N+1 ตั้งแต่ต้น ลองดูว่าถ้าลืมใส่จะเกิดอะไรขึ้น: เมื่อ view เรียก `post.author.name`
> และ `post.category.name` ในทุกแถวของ 20 แถวต่อหน้า จะเกิด query แยก 40 ครั้ง (2 association ×
> 20 แถว) รวมกับ query หลักและ count กลายเป็น **34 query ต่อ 1 request** ที่วัดได้จริงจาก
> `ActiveRecord: (34 queries, 24 cached)` ใน log — พอเพิ่ม `.includes` เข้าไป ตัวเลขลดเหลือแค่
> **4 query คงที่** ไม่ว่าจะมีกี่แถวต่อหน้าก็ตาม pagination ไม่ได้แก้ปัญหา N+1 ให้อัตโนมัติ ต้อง
> จัดการคู่กันเสมอ

### ใช้งานใน view

```erb
<%# app/views/posts/index.html.erb %>
<h1>บทความทั้งหมด</h1>

<%== pagy_info(@pagy) %>

<table>
  <thead>
    <tr>
      <th>ชื่อบทความ</th>
      <th>ผู้เขียน</th>
      <th>หมวดหมู่</th>
      <th>ยอดวิว</th>
      <th>เผยแพร่เมื่อ</th>
    </tr>
  </thead>
  <tbody>
    <% @posts.each do |post| %>
      <tr>
        <td><%= post.title %></td>
        <td><%= post.author.name %></td>
        <td><%= post.category.name %></td>
        <td><%= post.views %></td>
        <td><%= post.published_at&.to_date %></td>
      </tr>
    <% end %>
  </tbody>
</table>

<%== pagy_nav(@pagy) %>
```

> **ทำไมต้องใช้ `<%==` แทน `<%=`:** `pagy_info`/`pagy_nav` คืนค่าเป็น HTML string ที่ปลอดภัยอยู่แล้ว
> (ผ่านการ escape ค่าที่มาจาก user ให้เอง) `<%==` คือ shorthand ของ `raw(...)`/`.html_safe` บอก ERB
> ว่า "อย่า escape HTML นี้ซ้ำ" ถ้าใช้ `<%=` ธรรมดา จะเห็น HTML tag เป็นตัวอักษรดิบๆ บนหน้าเว็บแทน
> ที่จะ render จริง

รันเซิร์ฟเวอร์แล้วทดสอบจริง:

```bash
bin/rails server
curl -s http://localhost:3000/posts | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 1-20 of 36 in total</span>

curl -s http://localhost:3000/posts | grep -oP '<nav.*?</nav>'
# <nav class="pagy nav" aria-label="Pages">...<a href="/posts?page=2">2</a>...</nav>

curl -s "http://localhost:3000/posts?page=2" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 21-36 of 36 in total</span>
```

ผลลัพธ์ตรงตามที่คาด: หน้าแรกมี 20 แถว, หน้าสองมี 16 แถวที่เหลือ (36 - 20) และ `pagy_nav` สร้างลิงก์
ไปหน้า 2 ให้เองโดยที่เราไม่ต้องเขียนลอจิกคำนวณเลขหน้าเองสักบรรทัด

### เบื้องหลัง: SQL ที่ pagy ยิงจริง

```
Post Count (0.1ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."published" = TRUE
Post Load  (3.0ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
                     ORDER BY "posts"."created_at" DESC LIMIT 20 OFFSET 0
```

หน้า 2:

```
Post Count (0.2ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."published" = TRUE
Post Load  (0.2ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
                     ORDER BY "posts"."created_at" DESC LIMIT 20 OFFSET 20
```

ตรงตามที่ Part 034 Step 333 อธิบายไว้เป๊ะ — pagy แค่ **คำนวณ `LIMIT`/`OFFSET` ที่ถูกต้องให้
อัตโนมัติ** จาก `params[:page]` บวก query นับจำนวนทั้งหมด (`COUNT`) เพื่อรู้ว่ามีกี่หน้า ไม่มีเวทมนตร์
อะไรซ่อนอยู่เบื้องหลังเลย — สิ่งที่ pagy ทำให้คือ **ไม่ต้องเขียนคำนวณ `offset = (page - 1) * limit`
เองทุกที่ที่ต้องมี pagination** และมี UI (`pagy_nav`) ให้พร้อมใช้

---

## Step 373: ปรับแต่ง pagy — จำนวนรายการต่อหน้าแบบปลอดภัย, `pagy_info`, ตั้งค่า default ทั้งระบบ

### ให้ผู้ใช้เลือกจำนวนรายการต่อหน้าได้ — แต่ต้อง allowlist เสมอ

ความต้องการที่พบบ่อย: ให้ผู้ใช้เลือกได้ว่าจะดูหน้าละกี่รายการ (10/20/50) ผ่าน dropdown — จุดที่
มือใหม่มักพลาดคือปล่อยให้ `params[:items]` กำหนด `limit` ได้โดยตรงแบบไม่ตรวจสอบ ซึ่งเปิดช่องให้
ผู้ใช้ส่ง `?items=999999` มาบังคับให้ query ดึงข้อมูลมหาศาลในครั้งเดียว (กลับไปเจอปัญหาเดียวกับ
Step 371 ทั้งที่เพิ่ง fix ไปแล้ว)

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  ALLOWED_ITEMS = [10, 20, 50].freeze

  def index
    @pagy, @posts = pagy(
      Post.published.includes(:author, :category).order(created_at: :desc),
      limit: items_per_page
    )
  end

  private

  def items_per_page
    items = params[:items].to_i
    ALLOWED_ITEMS.include?(items) ? items : 20
  end
end
```

ทดสอบจริงทั้งกรณีปกติและกรณีพยายามหลีกเลี่ยง allowlist:

```bash
curl -s "http://localhost:3000/posts?items=10" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 1-10 of 36 in total</span>

curl -s "http://localhost:3000/posts?items=999" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 1-20 of 36 in total</span>   <- fallback เป็น 20 อัตโนมัติ
```

`items=999` ไม่ผ่าน allowlist จึงถูก fallback เป็นค่า default (20) อย่างเงียบๆ โดยไม่ error — นี่คือ
**หลักการเดียวกับที่จะใช้ซ้ำใน Step 376 เรื่อง sorting**: ไม่ว่า input จาก `params` จะมีค่าอะไรก็ตาม
โค้ดฝั่ง server ต้อง **ตรวจสอบกับชุดค่าที่อนุญาตไว้ล่วงหน้าเสมอ (allowlist)** ไม่ใช่พยายามคาดเดาว่า
"input แบบไหนน่าจะปลอดภัย" (blocklist) ซึ่งพลาดง่ายกว่ามาก

### `pagy_info` — ข้อความสรุปให้ผู้ใช้เห็น

```erb
<%== pagy_info(@pagy) %>
```

```html
<span class="pagy info">Displaying items 1-20 of 36 in total</span>
```

ปรับ label ของรายการที่แสดงได้ด้วย `item_name:`:

```erb
<%== pagy_info(@pagy, item_name: "บทความ") %>
```

```html
<span class="pagy info">Displaying บทความ 1-20 of 36 in total</span>
```

สังเกตว่าคำว่า "Displaying" กับ "of ... in total" ยังเป็นภาษาอังกฤษอยู่ — pagy รองรับหลายภาษาผ่าน
ไฟล์ locale ของตัวเอง (`pagy/locales/th.yml` มีให้พร้อมใน gem) แต่การตั้งค่า locale ทั้งระบบให้เป็น
ภาษาไทยอย่างถูกต้อง (รวมถึง error message ของ validation ที่เจอมาตั้งแต่ Part 026/032) จะเจาะลึก
เต็มรูปแบบใน **Part 039: Internationalization (I18n)** — ตอนนี้รู้ไว้ก่อนว่า option นี้มีอยู่และ
ปรับแต่งได้

### ตั้งค่า default ทั้งระบบผ่าน initializer

ถ้าไม่อยากส่ง `limit:`/`size:` ซ้ำทุกครั้งที่เรียก `pagy(...)` ตั้งเป็นค่า default กลางได้:

```ruby
# config/initializers/pagy.rb
# frozen_string_literal: true

Pagy::DEFAULT[:limit] = 20   # จำนวนรายการต่อหน้า (ค่าเริ่มต้นของ pagy เองก็คือ 20 อยู่แล้ว)
Pagy::DEFAULT[:size]  = 7    # จำนวนปุ่มเลขหน้าที่แสดงใน pagy_nav ก่อนเริ่มใช้ "..." (gap)
```

ยืนยันว่าใช้งานได้จริงด้วย `rails runner`:

```ruby
Pagy::DEFAULT[:limit] = 10
pagy, posts = pagy(Post.published)  # ไม่ส่ง limit: เลย
puts "limit=#{pagy.vars[:limit]} pages=#{pagy.pages}"
```

```
limit=10 pages=4
```

`Pagy::DEFAULT` เป็น global config — ตั้งครั้งเดียวใน initializer มีผลกับทุกจุดที่เรียก `pagy(...)`
โดยไม่ระบุ `limit:` เอง (จุดที่ระบุ `limit:` ตรงๆ แบบ Step ก่อนหน้ายัง override ได้เหมือนเดิม)

---

## Step 374: Kaminari — ทางเลือกที่ "ครบเครื่อง" กว่า และวิธีอ่านโค้ด legacy ที่ใช้มัน

**Kaminari** เป็น pagination gem ที่เก่าแก่กว่า pagy มาก (เปิดตัวปี 2011) และเคยเป็นตัวเลือก
มาตรฐานของวงการ Rails มานานหลายปี ยังคงถูกดูแลต่อเนื่องและใช้อยู่ในโปรเจกต์จำนวนมากจนถึงทุกวันนี้
— สิ่งสำคัญคือ **ต้องอ่านโค้ดที่ใช้ kaminari ออก** แม้จะเลือก pagy เป็นเครื่องมือหลักในหลักสูตรนี้
ก็ตาม เพราะมีโอกาสสูงที่จะเจอในโปรเจกต์จริงหรือ codebase ของบริษัทที่เข้าทำงาน

### ติดตั้งและใช้งาน

```ruby
# Gemfile
gem "kaminari"
```

```bash
bundle install
```

Kaminari **ไม่ต้อง include module ใดๆ** — มัน monkey-patch `ActiveRecord::Relation` ให้มี method
`page`/`per` ติดตัวไปเลยทันทีที่ gem โหลด:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.published.order(created_at: :desc).page(params[:page]).per(10)
  end
end
```

```erb
<%# app/views/posts/index.html.erb %>
<%= paginate @posts %>
```

ทดสอบจริง:

```ruby
result = Post.published.order(created_at: :desc).page(2).per(10)
puts "current_page=#{result.current_page} total_pages=#{result.total_pages} total_count=#{result.total_count}"
```

```
current_page=2 total_pages=4 total_count=36
```

SQL ที่ได้:

```
Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."published" = TRUE
                    ORDER BY "posts"."created_at" DESC LIMIT 10 OFFSET 10
Post Count (0.1ms)  SELECT COUNT(*) FROM "posts" WHERE "posts"."published" = TRUE
```

หน้าตา SQL เหมือน pagy เป๊ะ (`LIMIT`/`OFFSET` + `COUNT`) — เพราะกลไกพื้นฐานของทั้งสอง gem คือสิ่ง
เดียวกันที่ Part 034 Step 333 สอนไปแล้ว ต่างกันแค่ **API และ object ที่ห่อผลลัพธ์**

### สรุปเทียบ syntax คู่กัน

| สิ่งที่ต้องการ | pagy | kaminari |
|---|---|---|
| แบ่งหน้าใน controller | `@pagy, @posts = pagy(scope)` | `@posts = scope.page(params[:page]).per(20)` |
| Render UI ใน view | `pagy_nav(@pagy)` | `paginate @posts` |
| ต้อง include module ไหม | ต้อง `include Pagy::Backend`/`Pagy::Frontend` | ไม่ต้อง (monkey-patch ให้อัตโนมัติ) |
| หน้าปัจจุบัน | `@pagy.page` | `@posts.current_page` |
| จำนวนหน้าทั้งหมด | `@pagy.pages` | `@posts.total_pages` |
| จำนวนรายการทั้งหมด | `@pagy.count` | `@posts.total_count` |

### ทำไม Part นี้เลือก pagy เป็นตัวหลัก

1. **เบากว่ามาก** — pagy ไม่ patch `ActiveRecord::Relation` เลย (ไม่มี global monkey-patch)
   ทำให้ไม่มีผลข้างเคียงกับโค้ดส่วนอื่นของระบบ และ benchmark หลายครั้งจากทีมพัฒนา pagy เองแสดงว่า
   เร็วกว่าและใช้ memory น้อยกว่า kaminari อย่างมีนัยสำคัญเมื่อ scale ขึ้น
2. **Active maintenance ต่อเนื่อง** — อัปเดตรองรับฟีเจอร์ใหม่ของ Rails/Ruby สม่ำเสมอ
3. **Modular ผ่านระบบ "extras"** — ต้องการฟีเจอร์เสริม (เช่น pagination สำหรับ Array, GraphQL,
   Elasticsearch) ค่อย `require` extra นั้นเพิ่มเฉพาะตอนจำเป็น ไม่ต้องแบกของที่ไม่ได้ใช้

**Kaminari ยังคงเป็นตัวเลือกที่ดีมาก** โดยเฉพาะถ้าต้องการ pagination UI ที่ "ใช้ได้ทันที" แบบ
out-of-the-box มากกว่า (view partial สวยพร้อมใช้, generator สำหรับ customize theme) — เลือกตัวไหน
ก็ได้สำหรับโปรเจกต์ใหม่ตามความชอบของทีม แต่ **ต้องอ่านทั้งสองแบบออก** เพราะจะเจอทั้งคู่ในโลกจริง

---

## Step 375: ปัญหาของการ sort จาก params ตรงๆ — ช่องโหว่ SQL Injection ที่มองข้ามได้ง่าย

ฟีเจอร์ที่มาคู่กับ pagination เกือบทุกครั้งคือ **ให้ผู้ใช้คลิกหัวตารางเพื่อเรียงลำดับ** — ดูเผินๆ
เหมือนเป็นแค่การส่ง column name ผ่าน `params[:sort]` แล้ว `order` ตามนั้น แต่จุดนี้เป็นหนึ่งใน
**ช่องโหว่ SQL Injection ที่พบบ่อยที่สุดในโค้ด Rails จริง** เพราะดูไม่อันตรายเหมือนการต่อ string
เข้า `WHERE` ตรงๆ

### เวอร์ชันไร้เดียงสา (อันตราย)

```ruby
# อย่าเขียนแบบนี้!
def index
  sort = params[:sort]         # ผู้ใช้ควบคุมค่านี้ได้ 100%
  direction = params[:direction]
  @posts = Post.order("#{sort} #{direction}")
end
```

```ruby
sort = "views"
direction = "desc"
Post.order("#{sort} #{direction}").to_sql
```

```
SELECT "posts".* FROM "posts" ORDER BY views desc
```

ดูเหมือนทำงานถูกต้องกับ input ปกติ — ปัญหาคือ **`sort`/`direction` มาจาก `params` ที่ผู้ใช้ควบคุม
ได้เต็มที่** ตรงกับกฎที่ Part 034 Step 332 เตือนไว้แล้วเรื่อง array condition:
`ห้ามใช้ string interpolation ตรงๆ กับค่าที่มาจาก user input เด็ดขาด`

### Rails 8 มีเกราะป้องกันเบื้องต้นให้ — แต่ไม่ควรพึ่งมันอย่างเดียว

ลองส่ง string ที่ดูน่าสงสัยเข้า `order`:

```ruby
malicious = "(SELECT CASE WHEN (1=1) THEN views ELSE id END)"
Post.order("#{malicious} desc").to_sql
```

```
ActiveRecord::UnknownAttributeReference: Dangerous query method (method whose arguments are used
as raw SQL) called with non-attribute argument(s): "(SELECT CASE WHEN (1=1) THEN views ELSE id END) desc".
This method should not be called with user-provided values, such as request parameters or model
attributes. Known-safe values can be passed by wrapping them in Arel.sql().
```

Rails ตรวจจับได้ว่า string ที่ส่งเข้า `order` **ไม่ใช่ชื่อคอลัมน์ปกติ** (มีวงเล็บ, keyword SQL)
แล้ว raise error ป้องกันไว้ก่อนโดยอัตโนมัติ — เป็นเกราะป้องกันที่มีประโยชน์มาก แต่ **นี่คือจุดที่
นักพัฒนาที่ไม่เข้าใจสาเหตุที่แท้จริงมักพลาด**: เจอ error นี้แล้วคิดว่า "ต้องห่อด้วย `Arel.sql` เพื่อ
บอกว่าปลอดภัย" โดยไม่ได้ตรวจสอบค่าที่เข้ามาจริงๆ เลย

```ruby
# นักพัฒนา "แก้" error โดยห่อด้วย Arel.sql ตรงๆ โดยไม่กรองอะไรเพิ่ม
Post.order(Arel.sql("#{malicious} desc")).to_sql
```

```
SELECT "posts".* FROM "posts" ORDER BY (SELECT CASE WHEN (1=1) THEN views ELSE id END) desc
```

**Error หายไปทันที และ subquery ที่ผู้ใช้ส่งมาถูกรันจริงในฐานข้อมูล** — `Arel.sql(...)` คือการบอก
Rails ตรงๆ ว่า "ค่านี้ปลอดภัย ไม่ต้องตรวจสอบให้" ซึ่งเป็นสัญญาที่นักพัฒนาให้กับ Rails เอง ถ้าค่านั้น
มาจาก `params` โดยไม่ผ่านการตรวจสอบใดๆ ก่อน สัญญานี้ก็เป็นเท็จทันที และช่องโหว่ SQL Injection แบบ
เต็มรูปแบบก็เปิดออก — ผู้โจมตีสามารถส่ง subquery ใดๆ ก็ได้เข้าไปใน `ORDER BY` ผ่าน `params[:sort]`
ตราบใดที่ syntax ยัง valid ในตำแหน่งนั้น ซึ่งอาจนำไปสู่การอ่านข้อมูลจากตารางอื่นที่ไม่ควรเข้าถึงได้,
ทำให้ query ช้าลงมากจนเป็น denial-of-service, หรือกรณีเลวร้ายที่สุดคือใช้เป็นจุดเริ่มต้นขยายผลไปยัง
ช่องโหว่อื่นของระบบ (รายละเอียดเทคนิคการโจมตี SQL Injection แบบเต็มรูปแบบจะเจาะลึกใน **Part 079:
OWASP Top 10** — Step นี้แค่ต้องเข้าใจว่า **ทำไมจุดนี้ถึงเป็นความเสี่ยง** ให้ชัดพอที่จะไม่พลาด)

> **กฎจำง่าย:** ถ้าเห็น error `ActiveRecord::UnknownAttributeReference` แล้วสิ่งที่ทำคือห่อ
> `Arel.sql(...)` รอบค่าที่ **มาจาก `params` โดยตรงแบบไม่ผ่านการตรวจสอบ** — นั่นคือสัญญาณเตือนว่า
> กำลังปิดเกราะป้องกันของ Rails ด้วยมือตัวเอง วิธีแก้ที่ถูกต้องไม่ใช่การทำให้ error หายไป แต่คือ
> **ไม่ปล่อยให้ค่าที่ผู้ใช้ควบคุมได้เข้าถึงตำแหน่งนั้นเลยตั้งแต่แรก** — ดู Step 376

---

## Step 376: Safe sorting — pattern `sort_column`/`sort_direction` แบบ allowlist

วิธีแก้ที่ถูกต้องคือหลักการเดียวกับ Step 373: **allowlist ชื่อคอลัมน์ที่อนุญาตให้เรียงได้ไว้ล่วงหน้า
เป็น constant ในโค้ด** แล้วตรวจสอบว่า `params[:sort]` อยู่ในรายการนั้นหรือไม่ ก่อนนำไปใช้สร้าง query

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  SORTABLE_COLUMNS = %w[title views created_at].freeze

  def index
    @posts = Post.published.order(sort_column => sort_direction)
  end

  private

  def sort_column
    SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
  end

  def sort_direction
    params[:direction] == "asc" ? :asc : :desc
  end
end
```

จุดสำคัญของ pattern นี้:

1. **`SORTABLE_COLUMNS` เป็น allowlist แบบ hardcode ในโค้ด** — ไม่มีทางที่ค่าจาก `params` จะ
   "หลุด" ไปเป็นชื่อคอลัมน์อื่นนอกเหนือจากที่ระบุไว้ได้เลย ไม่ว่า input จะเป็นอะไรก็ตาม
2. **ใช้ hash syntax `order(column => direction)`** แทนการต่อ string (`"#{column} #{direction}"`)
   — เมื่อ `column` ผ่านการ allowlist มาแล้วว่าเป็นหนึ่งใน `%w[title views created_at]` แน่ๆ การส่ง
   เป็น hash key ให้ ActiveRecord ประกอบเป็น SQL เอง (แบบเดียวกับที่ Part 034 Step 333 สอน
   `Post.order(views: :desc)`) ปลอดภัยเต็มร้อยเพราะ ActiveRecord รู้จักและ escape ชื่อคอลัมน์ให้เอง
3. **`sort_direction` allowlist แค่สองค่า** (`:asc`/`:desc`) ด้วยตรรกะ ternary ง่ายๆ — ค่าอื่นใดที่
   ไม่ใช่ `"asc"` พอดี จะ fallback เป็น `:desc` เสมอ ไม่มีทางหลุดเป็นค่าอื่น

### ทดสอบด้วย request จริง

```bash
curl -s "http://localhost:3000/posts?sort=views&direction=asc" | grep -oP '<td>\d+</td>'
# <td>36</td>
# <td>37</td>
# <td>73</td>
# <td>74</td>
# <td>111</td>
# ... (เรียงจากน้อยไปมากถูกต้อง)
```

ทดสอบ payload ที่พยายามฉีด SQL ผ่าน `params[:sort]` แบบเดียวกับ Step 375:

```bash
curl -s -w "HTTP %{http_code}\n" \
  "http://localhost:3000/posts?sort=views);DROP+TABLE+posts;--" -o /tmp/response.html
# HTTP 200

grep -oP '<td>[^<]+</td>' /tmp/response.html | head -3
# <td>Rails 8 Highlights</td>
# <td>Nichada</td>
# <td>Ruby</td>

bin/rails runner 'puts Post.count'
# 45
```

**HTTP 200 ปกติ ตารางยังอยู่ครบ 45 แถวเหมือนเดิม** — payload ทั้งก้อนไม่ผ่าน
`SORTABLE_COLUMNS.include?(...)` เลย จึง fallback เป็น `sort_column` = `"created_at"` แบบเงียบๆ
ไม่มี error, ไม่มีการรั่วไหลของข้อมูล, และไม่มีทางที่ SQL แปลกปลอมจะไปถึงฐานข้อมูลได้เลย — นี่คือ
ความแตกต่างสำคัญระหว่าง "ป้องกันด้วย error handling" (ซึ่งยังต้องพึ่งว่า Rails จะตรวจจับ pattern
อันตรายได้ทุกกรณี) กับ **"ป้องกันด้วย allowlist ที่โครงสร้างข้อมูลรับประกันเอง"** (ซึ่งไม่มีทางหลุด
เพราะเป็น element ของ array คงที่ที่เขียนไว้ในโค้ด ไม่ใช่ regex หรือ pattern matching ที่อาจมีรูรั่ว)

> **หลักการทั่วไปที่ใช้ซ้ำได้ทุกที่:** เมื่อไหร่ก็ตามที่ต้องนำค่าจาก `params` ไปกำหนด **โครงสร้าง
> ของ query** (ชื่อคอลัมน์, ทิศทางการเรียง, ชื่อตาราง) แทนที่จะเป็น **ค่าของเงื่อนไข** (เช่นค่าที่
> เอาไปเทียบใน `WHERE` ซึ่ง Rails escape ให้อัตโนมัติผ่าน placeholder `?` อยู่แล้ว) ให้ผ่าน allowlist
> เสมอ ไม่มีข้อยกเว้น — ไม่ว่าจะเป็น sort column, ชื่อ scope ที่จะเรียกแบบ dynamic, หรือชื่อ
> association ที่จะ `includes`

---

## Step 377: Sortable table header ที่รักษาค่า filter/page เดิมไว้เสมอ

มี pattern การ sort ที่ปลอดภัยแล้ว ขั้นต่อไปคือทำหัวตารางให้คลิกได้ — จุดที่มักพลาดตรงนี้คือ
**ลืมรักษาค่า filter/page เดิมที่ผู้ใช้ตั้งไว้อยู่แล้ว** เวลาสร้างลิงก์เรียงลำดับใหม่ (เช่นผู้ใช้
กำลังดูหน้า 3 ของผลการค้นหา "Ruby" อยู่ พอกดเรียงตาม "ยอดวิว" กลับกระโดดไปหน้า 1 และล้างคำค้นหา
ทิ้งไปเฉยๆ — ประสบการณ์ใช้งานแย่มาก)

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  include Pagy::Frontend

  def sortable_header(column, label)
    current_column = params[:sort]
    current_direction = params[:direction] == "asc" ? "asc" : "desc"

    next_direction = (current_column == column && current_direction == "asc") ? "desc" : "asc"
    arrow = if current_column == column
              current_direction == "asc" ? " ▲" : " ▼"
            else
              ""
            end

    link_to "#{label}#{arrow}".html_safe,
      request.params.merge(sort: column, direction: next_direction, page: nil)
  end
end
```

จุดสำคัญ: **`request.params.merge(...)`** เอา query params ทั้งหมดของ request ปัจจุบัน (รวมทั้ง
`q`, `category_id`, `items` ที่จะเพิ่มเข้ามาใน Step ถัดไป) มาเป็นฐาน แล้ว **merge เฉพาะ `sort`/
`direction` ใหม่ทับเข้าไป** พร้อมล้าง `page` ทิ้ง (`page: nil`) เพราะเปลี่ยนการเรียงลำดับควรกลับไป
เริ่มที่หน้า 1 เสมอ — ส่วนค่า filter อื่นๆ ที่ผู้ใช้ตั้งไว้จะยังอยู่ครบเหมือนเดิม

```erb
<th><%= sortable_header "title", "ชื่อบทความ" %></th>
<th>ผู้เขียน</th>
<th>หมวดหมู่</th>
<th><%= sortable_header "views", "ยอดวิว" %></th>
<th><%= sortable_header "created_at", "สร้างเมื่อ" %></th>
```

ทดสอบว่า `items=10` ที่ผู้ใช้ตั้งไว้ถูกรักษาไว้ในลิงก์ sort จริงหรือไม่:

```bash
curl -s "http://localhost:3000/posts?items=10" | grep -oP '<th>.*?</th>'
```

```html
<th><a href="/posts?direction=asc&amp;items=10&amp;sort=title">ชื่อบทความ</a></th>
<th>ผู้เขียน</th>
<th>หมวดหมู่</th>
<th><a href="/posts?direction=asc&amp;items=10&amp;sort=views">ยอดวิว</a></th>
<th><a href="/posts?direction=asc&amp;items=10&amp;sort=created_at">สร้างเมื่อ</a></th>
```

**`items=10` ติดไปกับทุกลิงก์โดยอัตโนมัติ** ไม่ต้องเขียนโค้ดพิเศษอะไรเพิ่มเลยเพราะ
`request.params` จับมาให้ครบเอง — และเมื่อกดซ้ำที่คอลัมน์เดิมที่กำลัง sort อยู่ ทิศทางจะสลับ
asc/desc พร้อมลูกศรบอกทิศทางปัจจุบัน (▲/▼) ด้วย

---

## Step 378: Filter Object / Search Object — แยก logic การกรองออกจาก controller

โจทย์ต่อไป: ให้ผู้ใช้ **ค้นหาบทความจากชื่อเรื่อง** และ **กรองตามหมวดหมู่** พร้อมกันได้ วิธีที่มือใหม่
มักเขียนคือยัด `if` เช็คทุกเงื่อนไขไว้ใน controller action ตรงๆ:

```ruby
# อย่าทำแบบนี้ในระยะยาว — action บวมขึ้นเรื่อยๆ ทุกครั้งที่เพิ่ม filter ใหม่
def index
  @posts = Post.published
  @posts = @posts.where("title LIKE ?", "%#{params[:q]}%") if params[:q].present?
  @posts = @posts.where(category_id: params[:category_id]) if params[:category_id].present?
  @posts = @posts.order(sort_column => sort_direction)
end
```

โค้ดแบบนี้ **ใช้งานได้จริง** และใช้หลักการ lazy evaluation จาก Part 034 Step 331 ถูกต้อง แต่มี
ปัญหาเชิงโครงสร้างเมื่อระบบโตขึ้น: controller action กลายเป็นที่รวมของทุก filter logic, ทดสอบยาก
(ต้องยิง request จริงหรือ mock controller ทั้งก้อนเพื่อทดสอบแค่ตรรกะการกรอง), และใช้ซ้ำที่อื่นไม่ได้
(เช่นถ้ามี API endpoint หรือ background job ที่ต้องการ filter logic เดียวกัน)

### แก้ด้วย Filter Object (หรือเรียกว่า Search Object) — เป็นแค่ PORO ธรรมดา

**Filter Object** คือ Plain Old Ruby Object (PORO — ไม่ใช่ ActiveRecord model, ไม่ inherit อะไร
พิเศษ) ที่มีหน้าที่เดียว: **รับ relation กับ params เข้ามา แล้วคืน relation ที่ผ่านการกรองแล้ว**

```ruby
# app/models/post_filter.rb
class PostFilter
  def initialize(relation = Post.all, params = {})
    @relation = relation
    @params = params
  end

  def call
    relation = @relation
    relation = filter_by_search(relation)
    relation = filter_by_category(relation)
    relation
  end

  private

  def filter_by_search(relation)
    return relation if @params[:q].blank?

    relation.where("title LIKE ?", "%#{sanitize_like(@params[:q])}%")
  end

  def filter_by_category(relation)
    return relation if @params[:category_id].blank?

    relation.where(category_id: @params[:category_id])
  end

  def sanitize_like(term)
    term.gsub(/[\\%_]/) { |char| "\\#{char}" }
  end
end
```

จุดที่ควรสังเกต:

1. **`initialize` รับ `relation` เป็น argument แรก ไม่ hardcode `Post.all` ไว้ข้างใน method
   เดียว** — ทำให้เรียกต่อจาก relation ที่ผ่านการกรองมาแล้วบางส่วนได้ (เช่น
   `PostFilter.new(Post.published, params)` แบบที่จะเห็นใน Step 379) หรือทดสอบด้วย relation ปลอม
   ก็ได้โดยไม่ต้องแตะฐานข้อมูลจริงถ้าต้องการ
2. **`call` คืนค่าเป็น `ActiveRecord::Relation` เสมอ ไม่ใช่ Array** — รักษาหลักการ lazy evaluation
   ไว้ครบถ้วน ผู้เรียกยังเอาไป `.order`/`.includes`/ต่อ pagination ได้ตามปกติทุกประการ (Step 379
   จะแสดงให้เห็นชัด)
3. **`filter_by_search`/`filter_by_category` แต่ละตัวมี early return ด้วย `.blank?`** — ถ้าไม่มี
   ค่านั้นใน params เลย ก็ข้ามเงื่อนไขนั้นไปเฉยๆ ไม่ error และไม่ query เพิ่มโดยไม่จำเป็น
4. **`sanitize_like` escape อักขระพิเศษของ `LIKE`** (`%`, `_`, `\`) ก่อนนำไปใช้ — ถ้าผู้ใช้ค้นหาคำ
   ที่บังเอิญมี `%` หรือ `_` อยู่ในคำค้นหาจริงๆ (เช่นค้นหา "50%") อักขระเหล่านี้มีความหมายพิเศษใน SQL
   `LIKE` (`%` = wildcard ใดๆ, `_` = ตัวอักษรเดียวใดๆ) ถ้าไม่ escape ก่อน ผลการค้นหาจะผิดเพี้ยนจาก
   ที่ผู้ใช้ตั้งใจ (ไม่ใช่ช่องโหว่ความปลอดภัย เพราะ `?` placeholder ป้องกัน SQL Injection ให้อยู่แล้ว
   ตาม Part 034 Step 332 — แต่เป็นความถูกต้องของผลลัพธ์การค้นหา)

### ทำไมไม่ยัด logic นี้ลง scope ใน `Post` model ไปเลย

ย้อนกลับไป Part 034 Step 338 ที่สอนไว้ว่า scope เหมาะกับเงื่อนไขง่ายๆ ไม่มี logic แตกแขนง — filter
ที่รับ **หลาย parameter พร้อมกันจาก request** และต้องมี early-return หลายจุดแบบนี้ เขียนเป็น scope
เดี่ยวๆ ได้ไม่สวยเท่า class ที่แยกออกมาต่างหาก และที่สำคัญกว่านั้น: **`Post` model ไม่ควรรู้จัก
`params` เลยแม้แต่น้อย** — `params` เป็นแนวคิดของ HTTP request/controller layer ไม่ใช่ของ
ActiveRecord model การเอา `params` ไปยัดไว้ใน scope ของ model ทำให้ model ผูกติดกับ web layer
โดยไม่จำเป็น (ละเมิดหลักการแยกชั้นความรับผิดชอบ — SOLID ที่เจอไปแล้วใน Part 016) — Filter Object
แก้ปัญหานี้ได้ตรงจุด: **มันคือ "ตัวกลาง" ที่รู้จักทั้ง `params` และ `ActiveRecord::Relation`
โดยเฉพาะ ไม่ใช่หน้าที่ของ model หรือ controller**

> **หมายเหตุ:** pattern นี้เป็นญาติใกล้ชิดกับ **Query Object** ที่จะเจาะลึกอย่างเป็นทางการใน
> **Part 083** (พร้อม namespace `app/queries/` และ convention มาตรฐานของทีมใหญ่) — ตอนนี้ยังใช้
> `app/models/post_filter.rb` แบบง่ายๆ ไปก่อนเพื่อโฟกัสที่หลักการ ค่อยจัดระเบียบให้เป็นระบบมากขึ้น
> เมื่อถึง Part นั้น

---

## Step 379: รวม filter + sort + pagination เข้าด้วยกัน และ ransack ในฐานะทางเลือกสำหรับ admin

ตอนนี้มีทุกชิ้นส่วนแล้ว — มาประกอบเข้าด้วยกันใน controller เดียว:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  ALLOWED_ITEMS = [10, 20, 50].freeze
  SORTABLE_COLUMNS = %w[title views created_at].freeze

  def index
    filtered = PostFilter.new(Post.published, params).call
    sorted = filtered.includes(:author, :category).order(sort_column => sort_direction)

    @pagy, @posts = pagy(sorted, limit: items_per_page)
  end

  private

  def items_per_page
    items = params[:items].to_i
    ALLOWED_ITEMS.include?(items) ? items : 20
  end

  def sort_column
    SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
  end

  def sort_direction
    params[:direction] == "asc" ? :asc : :desc
  end
end
```

สังเกตว่า action นี้อ่านออกเป็นประโยคเดียวได้ตรงตัว: **"กรองจากบทความที่เผยแพร่แล้ว → เรียงลำดับ
ตามที่ผู้ใช้เลือก → แบ่งหน้า"** — แต่ละขั้นตอนเป็น method/object ที่แยกความรับผิดชอบชัดเจน ไม่มี
`if` เกลื่อนกลาดปนกันเหมือนตัวอย่างแรกของ Step 378 เลย ทั้งหมดนี้ยังคง **lazy** จนถึงบรรทัดสุดท้าย
ที่ `pagy(...)` เรียก `.count`/`.to_a` จริง (หลักการจาก Part 034 Step 331)

เพิ่ม UI ค้นหาใน view:

```erb
<%= form_with url: posts_path, method: :get do %>
  <%= text_field_tag :q, params[:q], placeholder: "ค้นหาชื่อบทความ" %>
  <%= select_tag :category_id,
        options_for_select(Category.pluck(:name, :id), params[:category_id]),
        include_blank: "ทุกหมวดหมู่" %>
  <%= submit_tag "ค้นหา" %>
<% end %>
```

> **`Category.pluck(:name, :id)`** — เทคนิคจาก Part 034 Step 339 พอดี: ดึงมาแค่สองคอลัมน์ที่
> ต้องใช้สำหรับ dropdown โดยไม่สร้าง ActiveRecord object เต็มรูปแบบให้เปลืองโดยไม่จำเป็น

### ทดสอบด้วย request จริงหลายรูปแบบ

```bash
# ค้นหาด้วยคำว่า "Ruby" อย่างเดียว
curl -s "http://localhost:3000/posts?q=Ruby" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying 1 item</span>

# กรองตามหมวดหมู่ (id=1 คือ Ruby) + เรียงตามยอดวิวจากน้อยไปมาก + 10 รายการต่อหน้า
curl -s "http://localhost:3000/posts?category_id=1&sort=views&direction=asc&items=10" \
  | grep -oP '<td>\d+</td>'
# <td>36</td>
# <td>148</td>
# <td>184</td>
# <td>296</td>
# <td>332</td>
# <td>444</td>
# <td>592</td>
# <td>628</td>
# <td>888</td>
# (9 บทความในหมวด Ruby ที่เผยแพร่แล้ว เรียงจากยอดวิวน้อยไปมากถูกต้อง อยู่ในหน้าเดียวเพราะ items=10)

# รวม q + category_id พร้อมกัน (กรองแบบ AND)
curl -s -w "HTTP %{http_code}\n" "http://localhost:3000/posts?q=Rails&category_id=1" \
  -o /tmp/combo.html
# HTTP 200
```

ทุกเงื่อนไขทำงานร่วมกันถูกต้อง — `PostFilter` กรองด้วย `q`/`category_id`, controller เรียงลำดับ
ด้วย allowlist, pagy แบ่งหน้า และ `sortable_header`/`pagy_nav` รักษาทุก query param เดิมไว้ในทุก
ลิงก์ที่สร้างขึ้นมา (ตรวจสอบได้จาก Step 377) — ไม่มีจุดไหนที่ผู้ใช้เปลี่ยนหน้าหรือเปลี่ยนการเรียง
แล้วเงื่อนไขค้นหาที่ตั้งไว้หายไปเลย

### ransack — ทางเลือกสำหรับหน้า admin search ที่ซับซ้อนมาก

เมื่อความต้องการ filter ซับซ้อนขึ้นเรื่อยๆ (ผู้ใช้ต้องเลือกได้ว่าจะค้นหาแบบ "contains", "greater
than", "starts with" กับ column ไหนก็ได้แบบ dynamic — ทั่วไปคือหน้า admin dashboard) การเขียน
Filter Object เองทุก field จะเริ่มซ้ำซากมาก ตรงนี้คือจุดที่ gem **ransack** เข้ามาช่วยได้

```ruby
# Gemfile
gem "ransack"
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  def self.ransackable_attributes(_auth_object = nil)
    %w[title views created_at]
  end

  def self.ransackable_associations(_auth_object = nil)
    %w[author category]
  end
end
```

```ruby
# app/controllers/posts_controller.rb
def index
  @q = Post.published.ransack(params[:q])
  @posts = @q.result.order(created_at: :desc)
end
```

```erb
<%= search_form_for @q do |f| %>
  <%= f.search_field :title_cont %>            <%# title LIKE '%...%' %>
  <%= f.select :category_id_eq, Category.pluck(:name, :id) %>
  <%= f.submit %>
<% end %>
```

ransack สร้าง query จาก **ชื่อ field พิเศษ** เช่น `title_cont` (contains), `views_gteq` (greater
than or equal), `category_id_eq` (equals) โดยไม่ต้องเขียน `where` เองสักบรรทัด — สะดวกมากสำหรับ
หน้า admin ที่ต้องรองรับการค้นหาหลาย field พร้อม operator หลากหลายแบบ dynamic

> **ประวัติความปลอดภัยที่ควรรู้ก่อนใช้:** ransack เวอร์ชันเก่าเคย **อนุญาตให้ค้นหา/เรียงลำดับผ่าน
> column และ association ใดๆ ของ model ได้ทั้งหมดโดยอัตโนมัติ** ถ้าไม่ได้ตั้งค่าอะไรเพิ่ม รวมถึง
> column ที่ไม่ควรเปิดให้ค้นหาได้เลย เช่น `password_digest`, token ต่างๆ, หรือ column ของ
> association ที่เชื่อมโยงไปยังข้อมูลที่ควรเป็นความลับ — ปัญหานี้ทำให้ ransack เวอร์ชันปัจจุบัน
> (ตั้งแต่รุ่นที่ปรับปรุงเรื่องความปลอดภัยเป็นต้นมา) **เปลี่ยน default เป็นปฏิเสธทุกอย่างก่อน**
> (deny-by-default) แล้วบังคับให้นักพัฒนา **ต้องเขียน `ransackable_attributes`/
> `ransackable_associations` ใน model เองอย่างชัดเจน** เพื่อ allowlist ว่า column/association ไหน
> อนุญาตให้ค้นหาหรือเรียงลำดับได้บ้าง — โค้ดตัวอย่างข้างบนที่ override สอง class method นี้ **ไม่ใช่
> ทางเลือก แต่เป็นข้อบังคับ** ของ ransack เวอร์ชันปัจจุบัน ถ้าเจอโปรเจกต์เก่าที่ไม่มีสอง method นี้
> อยู่ใน model เลย ควรตรวจสอบเวอร์ชัน ransack และปรับปรุงทันที — นี่คือ **ตัวอย่างจริงของหลักการ
> allowlist ที่ Step 376 สอนไว้** ถูกนำไปใช้ในระดับ gem ทั้งวงการหลังจากเรียนรู้จากปัญหาความ
> ปลอดภัยที่เกิดขึ้นจริง

**กฎการเลือกใช้ในทางปฏิบัติ:** สำหรับหน้า index ทั่วไปที่มี filter ไม่กี่แบบตายตัว (แบบ Step 378-379
ทั้งหมด) **Filter Object ที่เขียนเองดีกว่าเสมอ** — ควบคุมได้เต็มที่, อ่านง่าย, ทดสอบง่าย, ไม่มี
dependency เพิ่ม และไม่มีความเสี่ยงจาก default ที่อาจเปลี่ยนแปลงระหว่าง version ของ gem ภายนอก —
**ransack เหมาะกับหน้า admin dashboard ที่ต้องการความยืดหยุ่นสูงมาก** (ผู้ใช้เลือก field +
operator ได้เองแบบ dynamic UI) ที่การเขียน Filter Object เองทุก field จะกลายเป็นภาระมากเกินไป

---

## Step 380: ทดสอบ Filter Object แบบแยกหน่วย (RSpec)

ข้อดีที่ชัดเจนที่สุดของการแยก `PostFilter` ออกมาเป็น PORO ต่างหาก (แทนที่จะฝังไว้ใน controller
action) คือ **ทดสอบได้โดยไม่ต้องยิง HTTP request หรือแตะ controller เลย** — ทบทวนจาก **Part 019**
เรื่อง RSpec เบื้องต้น (`describe`/`context`/`it`, `let`, `before`/`after`):

```ruby
# spec/models/post_filter_spec.rb
require "rails_helper"

RSpec.describe PostFilter do
  let(:ruby) { Category.create!(name: "Ruby") }
  let(:rails_cat) { Category.create!(name: "Rails") }
  let(:author) { Author.create!(name: "Nichada", nationality: "Thai") }

  let!(:ruby_post) do
    Post.create!(title: "เริ่มต้นกับ Ruby", body: "...", author: author,
                 category: ruby, published: true)
  end
  let!(:rails_post) do
    Post.create!(title: "Rails Routing เบื้องต้น", body: "...", author: author,
                 category: rails_cat, published: true)
  end

  describe "#call" do
    context "เมื่อไม่มี filter params" do
      it "คืนค่า relation เดิมทั้งหมด" do
        result = described_class.new(Post.all, {}).call
        expect(result).to match_array([ruby_post, rails_post])
      end
    end

    context "เมื่อค้นหาด้วย q" do
      it "คืนเฉพาะบทความที่ title ตรงกับคำค้นหา" do
        result = described_class.new(Post.all, { q: "Ruby" }).call
        expect(result).to contain_exactly(ruby_post)
      end

      it "ค้นหาแบบ partial match ได้ (ไม่ต้องพิมพ์เต็มคำ)" do
        result = described_class.new(Post.all, { q: "Routing" }).call
        expect(result).to contain_exactly(rails_post)
      end
    end

    context "เมื่อกรองด้วย category_id" do
      it "คืนเฉพาะบทความในหมวดหมู่นั้น" do
        result = described_class.new(Post.all, { category_id: ruby.id }).call
        expect(result).to contain_exactly(ruby_post)
      end
    end

    context "เมื่อใช้ทั้ง q และ category_id พร้อมกัน" do
      it "กรองแบบ AND ทั้งสองเงื่อนไข" do
        result = described_class.new(Post.all, { q: "Ruby", category_id: rails_cat.id }).call
        expect(result).to be_empty
      end
    end

    context "เมื่อ params เป็นค่าว่าง" do
      it "ไม่ raise error และคืน relation ที่ยังไม่ query จริง (lazy)" do
        expect { described_class.new(Post.all, { q: "", category_id: "" }).call }
          .not_to raise_error
      end
    end
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/models/post_filter_spec.rb
```

```
......

Finished in 0.08 seconds (files took 0.99 seconds to load)
6 examples, 0 failures
```

**ทั้ง 6 test นี้ไม่มีจุดไหนเรียก HTTP request, ไม่มี controller, ไม่มี view เข้ามาเกี่ยวข้องเลย** —
ทดสอบ business logic ของการกรองข้อมูลได้ตรงจุดและเร็วมาก (0.08 วินาทีสำหรับ 6 tests) เพราะ
`PostFilter` เป็น PORO ธรรมดาที่ทดสอบแยกได้อย่างสมบูรณ์ — นี่คือประโยชน์ที่จับต้องได้ของการแยก
responsibility ออกจาก controller ตามที่ Step 378 อธิบายเหตุผลไว้

ส่วน controller เองก็ยังทดสอบได้ด้วย **request spec** (จะเจาะลึกเต็มรูปแบบใน **Part 046**) เพื่อ
ยืนยันว่าทุกชิ้นส่วนประกอบกันถูกต้องในระดับ HTTP จริง:

```ruby
# spec/requests/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :request do
  it "ปฏิเสธ sort param นอก allowlist อย่างปลอดภัย (fallback แทน error)" do
    get posts_path, params: { sort: "1);DROP TABLE posts;--" }
    expect(response).to have_http_status(:ok)
    expect(Post.count).to be_positive # ตารางไม่ได้หายไป
  end
end
```

```
.

Finished in 0.06 seconds
1 example, 0 failures
```

**หลักการแบ่งชั้นการทดสอบ:** ตรรกะการกรอง/คำนวณที่ซับซ้อน → ทดสอบที่ระดับ PORO/model โดยตรง (เร็ว,
เจาะจง) ส่วนการทำงานร่วมกันของทุกชิ้นส่วนในระดับ HTTP (routing, params parsing, response status) →
ทดสอบด้วย request spec (ช้ากว่าแต่ครอบคลุมภาพรวมจริง) — ไม่ควรทดสอบทุกกรณีของ `PostFilter` ผ่าน
request spec เพราะจะช้าโดยไม่จำเป็นและทดสอบสิ่งเดียวกันซ้ำซ้อนในสองระดับ

---

## แบบฝึกหัด: `PostsController#index` ที่รวม pagination + safe sorting + Filter Object ครบวงจร

### โจทย์

สร้าง `PostsController#index` ที่:

1. ใช้ **pagy** แบ่งหน้า โดยจำนวนรายการต่อหน้าเลือกได้จาก `params[:items]` แต่ต้องผ่าน allowlist
   `[10, 20, 50]` เท่านั้น
2. เรียงลำดับได้ตาม `title`, `views`, หรือ `created_at` ผ่าน `params[:sort]`/`params[:direction]`
   ด้วย pattern allowlist ที่ปลอดภัย (ไม่มีทาง SQL Injection ได้ไม่ว่า input จะเป็นอะไร)
3. ใช้ `PostFilter` PORO กรองด้วย **คำค้นหาจากชื่อเรื่อง** (`params[:q]`) และ **หมวดหมู่**
   (`params[:category_id]`) พร้อมกันได้
4. ทดสอบด้วย curl จริงหลายชุด query param เพื่อยืนยันว่าทุกฟีเจอร์ทำงานถูกต้องและปลอดภัย

### เฉลย

**1) Model และ Filter Object**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category

  scope :published, -> { where(published: true) }
end
```

```ruby
# app/models/post_filter.rb
class PostFilter
  def initialize(relation = Post.all, params = {})
    @relation = relation
    @params = params
  end

  def call
    relation = @relation
    relation = filter_by_search(relation)
    relation = filter_by_category(relation)
    relation
  end

  private

  def filter_by_search(relation)
    return relation if @params[:q].blank?

    relation.where("title LIKE ?", "%#{sanitize_like(@params[:q])}%")
  end

  def filter_by_category(relation)
    return relation if @params[:category_id].blank?

    relation.where(category_id: @params[:category_id])
  end

  def sanitize_like(term)
    term.gsub(/[\\%_]/) { |char| "\\#{char}" }
  end
end
```

**2) Controller**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  ALLOWED_ITEMS = [10, 20, 50].freeze
  SORTABLE_COLUMNS = %w[title views created_at].freeze

  def index
    filtered = PostFilter.new(Post.published, params).call
    sorted = filtered.includes(:author, :category).order(sort_column => sort_direction)

    @pagy, @posts = pagy(sorted, limit: items_per_page)
  end

  private

  def items_per_page
    items = params[:items].to_i
    ALLOWED_ITEMS.include?(items) ? items : 20
  end

  def sort_column
    SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
  end

  def sort_direction
    params[:direction] == "asc" ? :asc : :desc
  end
end
```

**3) View**

```erb
<%# app/views/posts/index.html.erb %>
<h1>บทความทั้งหมด</h1>

<%= form_with url: posts_path, method: :get do %>
  <%= text_field_tag :q, params[:q], placeholder: "ค้นหาชื่อบทความ" %>
  <%= select_tag :category_id,
        options_for_select(Category.pluck(:name, :id), params[:category_id]),
        include_blank: "ทุกหมวดหมู่" %>
  <%= submit_tag "ค้นหา" %>
<% end %>

<%== pagy_info(@pagy) %>

<table>
  <thead>
    <tr>
      <th><%= sortable_header "title", "ชื่อบทความ" %></th>
      <th>ผู้เขียน</th>
      <th>หมวดหมู่</th>
      <th><%= sortable_header "views", "ยอดวิว" %></th>
      <th><%= sortable_header "created_at", "สร้างเมื่อ" %></th>
    </tr>
  </thead>
  <tbody>
    <% @posts.each do |post| %>
      <tr>
        <td><%= post.title %></td>
        <td><%= post.author.name %></td>
        <td><%= post.category.name %></td>
        <td><%= post.views %></td>
        <td><%= post.published_at&.to_date %></td>
      </tr>
    <% end %>
  </tbody>
</table>

<%== pagy_nav(@pagy) %>
```

**4) ตรวจสอบด้วย curl จริง**

```bash
# กรณีปกติ: หน้าแรก, 20 รายการ, เรียงตาม created_at desc (default)
curl -s "http://localhost:3000/posts" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 1-20 of 36 in total</span>

# ค้นหาด้วยคำว่า "Ruby"
curl -s "http://localhost:3000/posts?q=Ruby" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying 1 item</span>

# กรองหมวดหมู่ Ruby (id=1) + เรียงยอดวิวน้อยไปมาก + 10 รายการต่อหน้า
curl -s "http://localhost:3000/posts?category_id=1&sort=views&direction=asc&items=10" \
  | grep -oP '<td>\d+</td>'
# <td>36</td> <td>148</td> <td>184</td> <td>296</td> <td>332</td>
# <td>444</td> <td>592</td> <td>628</td> <td>888</td>

# ค้นหา + กรองหมวดหมู่พร้อมกัน (AND)
curl -s -w "HTTP %{http_code}\n" "http://localhost:3000/posts?q=Rails&category_id=1" \
  -o /dev/null
# HTTP 200

# ทดสอบ security: sort param ที่พยายามฉีด SQL
curl -s -w "HTTP %{http_code}\n" \
  "http://localhost:3000/posts?sort=1);DROP+TABLE+posts;--" -o /dev/null
# HTTP 200

bin/rails runner 'puts Post.count'
# 45   <- ตารางไม่ได้ถูกลบ ยังอยู่ครบ

# ทดสอบ items นอก allowlist
curl -s "http://localhost:3000/posts?items=99999" | grep -oP '<span class="pagy info">.*?</span>'
# <span class="pagy info">Displaying items 1-20 of 36 in total</span>   <- fallback เป็น 20
```

ทุกกรณีทำงานถูกต้องตามที่ออกแบบไว้: filter/sort/pagination ทำงานร่วมกันได้อย่างถูกต้องในทุก
combination ของ query param และ **ไม่มี input ใดๆ ที่ทำให้เกิด error 500, SQL Injection, หรือ
ผลลัพธ์ที่ผิดเพี้ยนได้เลย** เพราะทุกจุดที่รับ input จาก `params` ไปกำหนดโครงสร้าง query (`sort`,
`items`) ผ่าน allowlist ทั้งหมด ส่วนจุดที่ใช้เป็นค่าเงื่อนไข (`q`, `category_id`) ก็ผ่าน
parameterized query ที่ ActiveRecord escape ให้อัตโนมัติ

**5) Test แยกหน่วยสำหรับ `PostFilter`**

```ruby
# spec/models/post_filter_spec.rb
require "rails_helper"

RSpec.describe PostFilter do
  let(:ruby) { Category.create!(name: "Ruby") }
  let(:author) { Author.create!(name: "Nichada", nationality: "Thai") }
  let!(:post) do
    Post.create!(title: "เริ่มต้นกับ Ruby", body: "...", author: author,
                 category: ruby, published: true)
  end

  it "กรองด้วย q และ category_id พร้อมกันได้ถูกต้อง" do
    result = described_class.new(Post.all, { q: "Ruby", category_id: ruby.id }).call
    expect(result).to contain_exactly(post)
  end
end
```

```bash
bundle exec rspec spec/models/post_filter_spec.rb spec/requests/posts_spec.rb
```

```
.........

Finished in 0.19 seconds (files took 1.03 seconds to load)
9 examples, 0 failures
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม filter ใหม่ให้ `PostFilter` สำหรับกรองตามช่วงยอดวิว (`params[:min_views]`,
   `params[:max_views]`) โดยใช้ range condition แบบที่เรียนใน Part 034 Step 332
   (`where(views: min..max)`) — ต้องจัดการกรณีที่มีแค่ `min_views` อย่างเดียว, มีแค่ `max_views`
   อย่างเดียว, หรือไม่มีทั้งคู่ ให้ครบทุกกรณีโดยไม่ error พร้อมเขียน RSpec test ครอบคลุมทั้ง 4 กรณี
   (มี min อย่างเดียว, มี max อย่างเดียว, มีทั้งคู่, ไม่มีเลย)
2. เพิ่ม sortable column ใหม่ที่ต้อง sort ผ่านตารางที่ join มา เช่น sort ตามชื่อผู้เขียน
   (`authors.name`) — ต้องแก้ `SORTABLE_COLUMNS`/`sort_column` อย่างไรให้ยังปลอดภัยอยู่ (ใบ้:
   allowlist ต้องแยกระหว่าง "ชื่อคอลัมน์ที่ผู้ใช้เห็น" กับ "SQL expression จริงที่จะใช้" เป็นสอง
   ชุดข้อมูลคนละอันเสมอ ไม่ควรให้ผู้ใช้ส่ง raw SQL แม้จะผ่าน allowlist ก็ตาม แล้วอย่าลืม `.joins`/
   `.includes` ตารางที่เกี่ยวข้องด้วย)
3. ลองติดตั้ง kaminari คู่ขนานกับ pagy ในโปรเจกต์ทดลองแยกต่างหาก แล้วเขียน index action เดียวกัน
   ทุกประการด้วย kaminari แทน pagy ทั้งหมด (รวม sortable header ที่ยังต้องใช้ `request.params`
   เหมือนเดิม) เปรียบเทียบว่าโค้ดส่วนไหนเหมือนกัน ส่วนไหนต่างกัน และ query count/timing ต่างกัน
   มากน้อยแค่ไหนด้วย `ActiveSupport::Notifications` แบบที่เรียนใน Part 034 แบบฝึกหัด

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Pagination ไม่ใช่ตัวเลือก แต่เป็นข้อกำหนดพื้นฐาน** — `Model.all`/scope ที่ไม่มี `limit` ส่งตรง
  เข้า view จะพังเมื่อข้อมูลโตขึ้นตามอายุของระบบเสมอ (สิ้นเปลือง memory, ทำให้ query ช้า, render
  ช้า) ไม่ว่าตอนเขียนโค้ดจะมีข้อมูลกี่แถวก็ตาม
- **pagy** ติดตั้งง่าย (`include Pagy::Backend` ใน controller, `include Pagy::Frontend` ใน
  helper) เบา เร็ว และ maintain อย่างต่อเนื่อง — `pagy(collection)` คืนค่า `[pagy_object,
  paginated_collection]`, `pagy_nav`/`pagy_info` render UI ให้ ปรับจำนวนรายการต่อหน้าได้ผ่าน
  `limit:` แต่ **ต้อง allowlist ค่าที่รับจาก `params` เสมอ** และตั้งค่า default ทั้งระบบผ่าน
  `Pagy::DEFAULT` ใน initializer ได้
- **kaminari** เป็นทางเลือกที่ครบเครื่องกว่าออกจากกล่อง (`.page(params[:page]).per(20)`,
  `paginate @posts`) ที่ยังพบได้ทั่วไปในโค้ด legacy จำนวนมาก — ต้องอ่านออกทั้งสองแบบแม้จะเลือกใช้
  แค่ตัวใดตัวหนึ่งเป็นหลักในโปรเจกต์ใหม่
- **การ sort จาก `params` ตรงๆ (`Post.order("#{sort} #{direction}")`) คือช่องโหว่ SQL Injection**
  — Rails 8 มีเกราะป้องกันเบื้องต้น (`ActiveRecord::UnknownAttributeReference`) แต่การ "แก้"
  ด้วยการห่อ `Arel.sql(...)` รอบค่าที่ไม่ผ่านการตรวจสอบ คือการปิดเกราะป้องกันนั้นด้วยมือตัวเอง
- **วิธีแก้ที่ถูกต้อง:** allowlist ชื่อคอลัมน์ที่อนุญาตให้เรียงได้เป็น constant
  (`SORTABLE_COLUMNS`) แล้วตรวจสอบ `params[:sort]` กับ allowlist นั้นก่อนใช้ — ค่าที่ไม่ผ่านจะ
  fallback เป็นค่า default อย่างเงียบๆ ไม่ error และไม่มีทาง SQL แปลกปลอมหลุดรอดไปถึงฐานข้อมูลได้
  หลักการนี้ใช้ได้กับทุกกรณีที่ `params` กำหนด **โครงสร้างของ query** ไม่ใช่แค่การ sort
- **Sortable table header ต้องรักษาค่า filter/page เดิมไว้เสมอ** ด้วย `request.params.merge(...)`
  — ล้างแค่ `page` (กลับไปหน้า 1) แต่คงค่า filter อื่นๆ ที่ผู้ใช้ตั้งไว้ครบ
- **Filter Object / Search Object** คือ PORO ที่แยก logic การกรองออกจาก controller — รับ
  `relation`/`params` เข้ามา คืน `ActiveRecord::Relation` ที่ผ่านการกรองแล้วออกไป (ยัง lazy อยู่
  เหมือนเดิม) ทำให้ controller อ่านง่าย, ทดสอบแยกหน่วยได้เร็วโดยไม่ต้องยิง HTTP request, และไม่ทำให้
  `Post` model ต้องรู้จัก `params` ซึ่งเป็นแนวคิดของ web layer
- **รวม filter + sort + pagination ในโค้ดเดียวกันได้อย่างสะอาด** เพราะทุกชิ้นส่วนคืนค่าเป็น
  `ActiveRecord::Relation` เหมือนกันหมด เชื่อมต่อกันได้ตามหลัก lazy evaluation จาก Part 034
- **ransack** เหมาะกับหน้า admin search ที่ต้องการความยืดหยุ่นสูงมาก (`title_cont`,
  `views_gteq` ฯลฯ) แต่มีประวัติเรื่องความปลอดภัยที่ต้องรู้: เวอร์ชันปัจจุบันบังคับให้ต้องเขียน
  `ransackable_attributes`/`ransackable_associations` เป็น allowlist ใน model เองเสมอ (deny-by-
  default) — สำหรับ filter ที่ตายตัวไม่กี่แบบ Filter Object ที่เขียนเองยังเป็นทางเลือกที่ดีกว่า
- **ทดสอบ Filter Object แยกจาก controller** ด้วย RSpec (ทบทวนจาก Part 019) ทำได้เร็วและตรงจุด
  เพราะเป็น PORO ล้วนๆ ไม่ต้องพึ่ง HTTP request หรือ view — ส่วนการทำงานร่วมกันของทุกชิ้นส่วนในระดับ
  controller ทดสอบด้วย request spec แยกอีกชั้นหนึ่ง

**ต่อไป (Part 039):** สังเกตว่า `pagy_info` ที่ใช้ตลอด Part นี้ยังแสดงข้อความเป็นภาษาอังกฤษ
("Displaying items 1-20 of 36 in total") ทั้งที่ label อื่นในหน้าเว็บเป็นภาษาไทยหมดแล้ว — Part
ถัดไปจะแก้จุดนี้อย่างเป็นระบบด้วย **Internationalization (I18n)**: การตั้งค่า locale ของทั้งแอป
ให้เป็นภาษาไทย, การแปล error message ของ validation (ที่เจอมาตั้งแต่ Part 026/032) ให้เป็นภาษาไทย
ที่อ่านเป็นธรรมชาติ, การแปล label ของ form fields, ปุ่ม, และข้อความของ gem ภายนอกอย่าง pagy เอง
ให้เป็นภาษาไทยทั้งระบบ พร้อมโครงสร้างไฟล์ `config/locales/` ที่จัดการง่ายเมื่อแอปโตขึ้น
