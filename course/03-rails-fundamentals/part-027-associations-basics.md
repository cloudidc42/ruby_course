# Part 027: Association เบื้องต้น — `belongs_to`, `has_many`, `has_one`

> **Step ครอบคลุมใน Part นี้:** Step 261–270
> **ระดับ:** กลาง (ต้องผ่าน Part 025 เรื่อง Model/Migration และ Part 026 เรื่อง
> Validation/Callback มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

ใน Part 025 เราสร้าง Model `Book` ที่มี column `author` เป็นแค่ **string ธรรมดา** — ใช้งานได้
สำหรับเก็บ "ชื่อผู้แต่ง" เฉยๆ แต่พอเริ่มอยากรู้ว่า "ผู้แต่งคนนี้เขียนหนังสือกี่เล่ม", อยากเก็บข้อมูล
เพิ่มเติมเกี่ยวกับผู้แต่ง (สัญชาติ, ปีเกิด, ประวัติ), หรืออยากมั่นใจว่าไม่มีใครพิมพ์ชื่อผู้แต่งคนเดียวกัน
ผิดเพี้ยนกันคนละแบบในแต่ละแถว — string ธรรมดาเริ่มไม่พอ นี่คือจุดที่ฐานข้อมูลเชิงสัมพันธ์ (relational
database) แสดงพลังที่แท้จริงของมัน: การแยกข้อมูลออกเป็นหลายตาราง แล้ว **เชื่อมโยง (associate)**
เข้าด้วยกันด้วย **foreign key**

Part นี้จะพาไปสร้างตาราง `authors` แยกออกมาจริงๆ เชื่อมกับ `books` ด้วย foreign key และเรียนรู้
สาม association หลักที่ใช้บ่อยที่สุดใน Rails: **`belongs_to`**, **`has_many`**, **`has_one`**
พร้อมกับปัญหาคลาสสิกที่มาพร้อมกับ association เสมอคือ **N+1 query problem** — เราจะเห็นปัญหานี้
ด้วยตาเปล่าผ่าน SQL log จริง แล้วแก้ด้วย `includes` เป็นครั้งแรก (รายละเอียดเชิงลึกเรื่อง query
interface ทั้งหมดจะไปเจาะลึกอีกทีใน Part 034)

## สารบัญของ Part นี้

- Step 261: ปัญหาของการอ้างอิงข้อมูลข้ามตารางด้วยมือ และ Foreign Key คืออะไร
- Step 262: `belongs_to` เบื้องต้น — Rails 8 บังคับให้ต้องมี record อ้างอิงอยู่จริงโดย default
- Step 263: `optional: true` และกับดักสำคัญ: validation ระดับ Model vs constraint ระดับฐานข้อมูล
- Step 264: มุมมองฝั่ง Migration — `t.references`, `add_reference`, foreign key constraint
- Step 265: `has_many` — ฝั่งตรงข้ามของ `belongs_to`
- Step 266: `has_one` — ความสัมพันธ์แบบหนึ่งต่อหนึ่ง
- Step 267: สร้าง/จัดการ record ผ่าน association โดยตรง (`build`, `create`, `<<`, `=`)
- Step 268: `dependent:` — `:destroy`, `:destroy_async`, `:nullify`, `:restrict_with_error`
  และอันตรายของการเลือกผิด
- Step 269: ปัญหา N+1 query — เห็นด้วยตาเปล่าผ่าน SQL log แล้วแก้เบื้องต้นด้วย `includes`
- Step 270: `inverse_of`, method ที่ Rails generate ให้อัตโนมัติทั้งหมด + แบบฝึกหัด

---

## Step 261: ปัญหาของการอ้างอิงข้อมูลข้ามตารางด้วยมือ และ Foreign Key คืออะไร

### ย้อนดู Model `Book` จาก Part 025

```ruby
# app/models/book.rb (จาก Part 025)
class Book < ApplicationRecord
end
```

```
# ตาราง books มี column: title, author (string), pages, read
```

ลองนึกภาพว่าเรามีหนังสือของ Haruki Murakami อยู่ 3 เล่มในตาราง `books`:

```irb
irb(main):001> Book.create(title: "Norwegian Wood", author: "Haruki Murakami", pages: 296)
irb(main):002> Book.create(title: "Kafka on the Shore", author: "Haruki Murakami", pages: 505)
irb(main):003> Book.create(title: "1Q84", author: "Haruki Murakami", pages: 925)
```

ดูเผินๆ เหมือนไม่มีปัญหา แต่พอใช้งานจริงจะเจอปัญหาซ้ำๆ กันหลายแบบ:

1. **พิมพ์ชื่อผิดเพี้ยนกันได้** — แถวหนึ่งพิมพ์ `"Haruki Murakami"` อีกแถวพิมพ์
   `"Haruki  Murakami"` (เว้นวรรคซ้ำ) หรือ `"H. Murakami"` ฐานข้อมูลไม่รู้ว่าทั้งสามคำนี้คือ
   คนเดียวกัน การ query "หนังสือทั้งหมดของ Murakami" จะพลาดบางแถวไปเงียบๆ
2. **เก็บข้อมูลเพิ่มเติมของผู้แต่งไม่ได้** — ถ้าอยากรู้สัญชาติหรือปีเกิดของผู้แต่ง ต้องเพิ่ม column
   `author_nationality`, `author_birth_year` เข้าไปใน `books` ซึ่งข้อมูลนี้จะถูก **บันทึกซ้ำ**
   ในทุกแถวที่เป็นหนังสือของผู้แต่งคนเดียวกัน (data duplication) — ถ้าผู้แต่งเปลี่ยนสัญชาติ ต้อง
   ไปแก้ทุกแถวพร้อมกัน ไม่งั้นข้อมูลจะไม่ตรงกัน (data inconsistency)
3. **นับจำนวนหนังสือของผู้แต่งคนหนึ่งต้อง query ด้วย string matching** ซึ่งช้าและเสี่ยงพลาด:

```irb
irb(main):004> Book.where(author: "Haruki Murakami").count
=> 3
```

query นี้ใช้งานได้ก็จริง แต่ถ้าสะกดคำค้นหาไม่ตรงกับที่บันทึกไว้เป๊ะแม้แต่ตัวเดียว (ตัวพิมพ์ใหญ่-เล็ก,
ช่องว่าง) ผลลัพธ์จะผิดทันทีโดยไม่มี error เตือนเลย

### วิธีแก้แบบฐานข้อมูลเชิงสัมพันธ์: แยกตาราง + Foreign Key

**Foreign Key** คือ column ในตารางหนึ่ง (ฝั่ง "ลูก" หรือ child) ที่เก็บค่า primary key ของแถวใน
อีกตารางหนึ่ง (ฝั่ง "แม่" หรือ parent) ไว้ เพื่อ "ชี้" ไปยังแถวนั้นแทนการก๊อปปี้ข้อมูลทั้งหมดมาเก็บซ้ำ

```
authors                          books
+----+----------+-------------+  +----+---------------------+-------+-----------+
| id | name     | nationality |  | id | title               | pages | author_id |
+----+----------+-------------+  +----+---------------------+-------+-----------+
| 1  | Murakami | Japanese    |  | 1  | Norwegian Wood      | 296   | 1         | --+
+----+----------+-------------+  | 2  | Kafka on the Shore  | 505   | 1         | --+-- ชี้ไปยัง authors.id = 1
                                  | 3  | 1Q84                | 925   | 1         | --+
                                  +----+---------------------+-------+-----------+
```

ตอนนี้ชื่อผู้แต่งถูกเก็บไว้ **ที่เดียว** (แถวเดียวในตาราง `authors`) หนังสือแต่ละเล่มแค่เก็บ
`author_id` ซึ่งเป็นตัวเลขชี้กลับไปหาแถวนั้น แก้ปัญหาทั้งสามข้อด้านบนได้ทันที: พิมพ์ชื่อผิดไม่ได้อีก
(เพราะพิมพ์แค่ครั้งเดียวตอนสร้าง `Author`), เพิ่ม column ข้อมูลผู้แต่งได้โดยไม่ซ้ำซ้อน, และนับจำนวน
หนังสือทำได้ด้วยการ join ตัวเลข `id` ซึ่งเร็วกว่าเทียบ string มาก

### ลองทำด้วยมือก่อน (ยังไม่ใช้ `belongs_to`/`has_many`)

มาสร้างตาราง `authors` และเพิ่ม column `author_id` ในตาราง `books` ด้วย migration ธรรมดา
(รายละเอียดเต็มเรื่อง migration syntax แบบนี้อยู่ใน Step 264):

```bash
bin/rails generate model Author name:string nationality:string
```

```
      invoke  active_record
      create    db/migrate/20260926040001_create_authors.rb
      create    app/models/author.rb
```

```bash
bin/rails generate migration AddAuthorIdToBooks author_id:integer
bin/rails db:migrate
```

ตอนนี้ `app/models/author.rb` และ `app/models/book.rb` ยังเป็น class เปล่าๆ ที่ **ไม่มี
association ใดๆ ทั้งสิ้น**:

```ruby
class Author < ApplicationRecord
end
```

```ruby
class Book < ApplicationRecord
end
```

ลองใช้งานแบบ "เชื่อมเอง" ด้วยมือทั้งหมด:

```irb
irb(main):001> author = Author.create(name: "Haruki Murakami", nationality: "Japanese")
=> #<Author id: 1, name: "Haruki Murakami", nationality: "Japanese", ...>

irb(main):002> book = Book.create(title: "Norwegian Wood", pages: 296, author_id: author.id)
=> #<Book id: 1, title: "Norwegian Wood", pages: 296, author_id: 1, ...>

irb(main):003> book.author_id
=> 1

irb(main):004> book.author
NoMethodError (undefined method 'author' for #<Book ...>)
```

เจอปัญหาแรกทันที: ต่อให้ column `author_id` มีค่าถูกต้องอยู่ในฐานข้อมูลแล้ว **`book.author` ก็ยัง
เรียกไม่ได้** เพราะ Rails ไม่รู้ว่า `author_id` แปลว่า "เชื่อมไปยัง Model `Author`" — เราต้อง
เขียน query เองทุกครั้งที่ต้องการข้อมูลผู้แต่งของหนังสือเล่มหนึ่ง:

```irb
irb(main):005> Author.find(book.author_id)
=> #<Author id: 1, name: "Haruki Murakami", nationality: "Japanese", ...>
```

และถ้าอยากรู้ว่าผู้แต่งคนหนึ่งมีหนังสืออะไรบ้าง ก็ต้อง query กลับทางเองอีก:

```irb
irb(main):006> Book.where(author_id: author.id)
=> #<ActiveRecord::Relation [#<Book id: 1, title: "Norwegian Wood", ...>]>
```

ใช้งานได้ แต่ **น่าเบื่อและเสี่ยง**: ทุกจุดในแอปที่ต้องข้ามไปมาระหว่างสองตารางนี้ ต้องเขียน
`Author.find(...)` หรือ `Book.where(author_id: ...)` เองซ้ำๆ, ไม่มีการตรวจสอบว่า `author_id`
ที่ใส่เข้าไปมีตัวตนอยู่จริงในตาราง `authors` หรือเปล่า (ใส่ `author_id: 999` ที่ไม่มีอยู่จริงก็ยัง
`create` ผ่านเฉยๆ), และไม่มีกลไกอัตโนมัติจัดการตอนต้องการลบข้อมูล (ลบ author แล้ว book ที่เหลือ
จะเป็นยังไง — เดี๋ยวเจอใน Step 268)

> **นี่คือปัญหาที่ Association ของ ActiveRecord เกิดมาเพื่อแก้โดยเฉพาะ** — Step ถัดไปเราจะเพิ่ม
> `belongs_to :author` บรรทัดเดียวใน `Book` แล้วปัญหาทั้งหมดข้างบนนี้จะหายไปในทันที

---

## Step 262: `belongs_to` เบื้องต้น — Rails 8 บังคับให้ต้องมี record อ้างอิงอยู่จริงโดย default

เพิ่มบรรทัดเดียวใน `app/models/book.rb`:

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```

`belongs_to :author` บอก ActiveRecord ว่า "ตาราง `books` มี foreign key ชื่อ `author_id` ที่ชี้
ไปยัง Model `Author`" — ชื่อ symbol (`:author`) ที่ใส่ต้องตรงกับชื่อ column แบบ **เอกพจน์ ไม่มี
`_id` ต่อท้าย** ตาม convention (`author_id` -> `:author`) ลองเรียกใช้ใหม่:

```irb
irb(main):001> book = Book.find(1)
irb(main):002> book.author
=> #<Author id: 1, name: "Haruki Murakami", nationality: "Japanese", ...>
```

`book.author` ใช้งานได้ทันทีโดยไม่ต้องเขียน `Author.find(book.author_id)` เองอีกต่อไป —
ActiveRecord จัดการ query เบื้องหลังให้ทั้งหมด (ยิง `SELECT * FROM authors WHERE id = ?`)

### `belongs_to` เป็น required โดย default ตั้งแต่ Rails 5

จุดที่คนย้ายมาจาก Rails เวอร์ชันเก่ามักแปลกใจคือ **ตั้งแต่ Rails 5.0 เป็นต้นมา (รวมถึง Rails 8.1
ที่ใช้ในหลักสูตรนี้) `belongs_to` จะถือว่า record ที่อ้างอิงถึง "ต้องมีอยู่จริงเสมอ" โดย default** —
พูดอีกแบบคือ Rails เพิ่ม `validates :author, presence: true` ให้อัตโนมัติทันทีที่เขียน
`belongs_to :author` โดยไม่ต้องเขียนเอง ลองพิสูจน์:

```irb
irb(main):003> book2 = Book.new(title: "Coraline", pages: 208)
irb(main):004> book2.valid?
=> false
irb(main):005> book2.errors.full_messages
=> ["Author must exist"]
irb(main):006> book2.save
=> false
```

`book2` ไม่มี `author` เลย (ทั้งไม่ได้ตั้ง `author_id` และไม่ได้ตั้ง `author`) — พอเรียก `valid?`
หรือ `save` จะไม่ผ่านทันที พร้อม error message `"Author must exist"` โดยที่เราไม่ได้เขียน
validation อะไรเองเลยสักบรรทัด

พอใส่ `author` ให้ครบ ก็ผ่านตามปกติ:

```irb
irb(main):007> book2.author = Author.find(1)
irb(main):008> book2.valid?
=> true
irb(main):009> book2.save
=> true
```

### ทำไม Rails ถึงเปลี่ยน default เป็น required

เหตุผลคือ **ในทางปฏิบัติ ความสัมพันธ์แบบ `belongs_to` ส่วนใหญ่ในแอปจริงควรมีค่าเสมอ** เช่น
`Comment belongs_to :post` — คอมเมนต์ที่ไม่มีโพสต์ที่มันสังกัดอยู่ไม่ควรมีอยู่ได้ตั้งแต่แรก การตั้งค่า
default เป็น required จึงช่วยดักข้อมูลผิดพลาดตั้งแต่ต้นทาง (fail fast) แทนที่จะปล่อยให้ record
ที่ไม่มี parent หลุดเข้าไปในฐานข้อมูลแล้วค่อยมาพังตอน `book.author.name` (`NoMethodError` เพราะ
`book.author` เป็น `nil`) ในหน้า view ทีหลัง

ถ้าต้องการปิดพฤติกรรมนี้ทั้งแอปพร้อมกัน (ไม่แนะนำ เว้นแต่ทำ legacy app ที่มีข้อมูลเก่าไม่ครบ)
ตั้งค่าได้ที่ `config/application.rb` หรือ `config/initializers/new_framework_defaults.rb`:

```ruby
# config/application.rb
config.active_record.belongs_to_required_by_default = false
```

แต่ในทางปฏิบัติ แนะนำให้ตั้งเป็นรายตัวด้วย `optional: true` เฉพาะ association ที่ควรเป็น optional
จริงๆ (Step 263) มากกว่าปิดทั้งแอป เพราะ association ส่วนใหญ่ควรเป็น required อยู่แล้ว

---

## Step 263: `optional: true` และกับดักสำคัญ: validation ระดับ Model vs constraint ระดับฐานข้อมูล

บาง association ควร **เป็น optional จริงๆ** เช่น หนังสือบางเล่มอาจยังไม่ได้ระบุผู้แต่ง (ไม่ทราบ
ผู้แต่ง, เป็นเอกสารไม่มีชื่อผู้เขียน) กรณีนี้เพิ่ม `optional: true` ที่ `belongs_to`:

```ruby
class Book < ApplicationRecord
  belongs_to :author, optional: true
end
```

ลองสร้าง book โดยไม่ระบุ author:

```irb
irb(main):001> book = Book.new(title: "ไม่ระบุผู้แต่ง", pages: 12)
irb(main):002> book.valid?
=> true
irb(main):003> book.save
```

```
ActiveRecord::NotNullViolation: SQLite3::ConstraintException: NOT NULL constraint failed: books.author_id
```

**เกิด error ทั้งที่ `valid?` คืน `true`!** นี่คือกับดักสำคัญที่ต้องเข้าใจให้แม่น: `optional: true`
ปิดแค่ **validation ระดับ Model** (Rails จะไม่ raise "Author must exist" ให้อีกต่อไป) แต่ถ้า
column `author_id` ในฐานข้อมูลถูกสร้างด้วย `null: false` (ซึ่งเป็นค่า default ของ `t.references`
ใน Rails 8 ตามที่จะเห็นใน Step 264) **ฐานข้อมูลเองจะยังปฏิเสธค่า `NULL` อยู่ดี** — เกิดเป็น
`ActiveRecord::NotNullViolation` ซึ่งเป็น exception ระดับต่ำกว่า validation ทั่วไป ไม่ใช่แค่
`save` คืน `false` เฉยๆ

**บทเรียนสำคัญ:** `optional: true` ที่ Model กับ `null: true` ที่ migration เป็นคนละเรื่องกัน
ต้องแก้ **ทั้งสองระดับ**ให้สอดคล้องกันเสมอ ถ้าต้องการให้ author เป็น optional จริง:

```ruby
class AllowNullAuthorIdOnBooks < ActiveRecord::Migration[8.1]
  def change
    change_column_null :books, :author_id, true
  end
end
```

```bash
bin/rails db:migrate
```

ทดสอบใหม่หลัง migrate:

```irb
irb(main):004> book = Book.new(title: "ไม่ระบุผู้แต่ง", pages: 12)
irb(main):005> book.valid?
=> true
irb(main):006> book.save
=> true
irb(main):007> book.author
=> nil
```

คราวนี้ผ่านทั้งสองระดับ บันทึกสำเร็จ และ `book.author` คืน `nil` อย่างปลอดภัย (ไม่ raise error)

> **แนวคิดสำคัญ:** ให้มองว่า **validation ระดับ Model ป้องกัน "ตอนที่แอปกำลังรัน"** (เช่น เช็ค
> ก่อน save ผ่าน form) ส่วน **constraint ระดับฐานข้อมูล (NOT NULL, foreign key, unique index)
> คือตาข่ายรองรับชั้นสุดท้าย** ที่ป้องกันแม้กระทั่งข้อมูลที่หลุดเข้ามาทาง path อื่นที่ไม่ผ่าน
> ActiveRecord เลย (เช่น script bulk import ที่ยิง SQL ตรงๆ, หรือ console ที่ใช้
> `update_column`/`insert_all` ซึ่งข้าม validation) ทั้งสองชั้นทำงานคนละหน้าที่ ไม่ใช่ตัวใดตัวหนึ่ง
> ใช้แทนกันได้ **โปรเจกต์ที่ดีควรมีทั้งสองชั้นตรงกันเสมอ** — ถ้า Model บอกว่า optional column
> ในฐานข้อมูลก็ควร `null: true` ด้วย และในทางกลับกัน ถ้า Model บอกว่า required (ค่า default)
> column ก็ควรเป็น `null: false` เพื่อป้องกันสองชั้นให้ตรงกัน (ซึ่งพอดีเป็นสิ่งที่ Rails 8 generator
> ทำให้อัตโนมัติอยู่แล้วอย่างที่จะเห็นใน Step 264)

---

## Step 264: มุมมองฝั่ง Migration — `t.references`, `add_reference`, foreign key constraint

กลับไปดูว่า Rails สร้าง migration ให้อย่างไรตอนใช้ generator แบบมาตรฐาน (ไม่ต้องทำเองทีละขั้น
แบบ Step 261) ลบโปรเจกต์ทดลองแล้วเริ่มใหม่ให้สะอาด หรือใช้ต่อจาก schema ปัจจุบันก็ได้ — ในหัวข้อนี้
จะอธิบายว่าเบื้องหลัง `author:references` ที่เห็นผ่านมาทำอะไรบ้าง

```bash
bin/rails generate model Author name:string nationality:string
bin/rails generate model Book title:string pages:integer author:references
```

สังเกต argument `author:references` — ชนิดข้อมูลพิเศษที่หมายถึง "column นี้เป็น foreign key
ชี้ไปยังอีกตารางหนึ่ง" migration ที่ได้:

```ruby
class CreateBooks < ActiveRecord::Migration[8.1]
  def change
    create_table :books do |t|
      t.string :title
      t.integer :pages
      t.references :author, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

และที่น่าทึ่งกว่านั้น — **`app/models/book.rb` ที่ generator สร้างให้มี `belongs_to :author`
เขียนไว้ให้เสร็จเรียบร้อยแล้วโดยอัตโนมัติ**:

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```

นี่คือเหตุผลที่ argument ชนิด `references` มีประโยชน์มากกว่าการเขียน `t.integer :author_id` เอง
ตรงๆ — generator ฉลาดพอที่จะรู้ว่าถ้าเจอ column ชนิด `references` ต้องเพิ่ม `belongs_to`
ให้ใน Model ทันที ไม่ต้องมาเขียนเองอีกที

### วิเคราะห์ `t.references :author, null: false, foreign_key: true`

| ส่วนประกอบ | ความหมาย |
|--------------|------------|
| `t.references :author` | สร้าง column `author_id` ชนิด `integer` พร้อม index ให้อัตโนมัติ (`index_books_on_author_id`) — index สำคัญมากเพราะ column นี้จะถูกใช้ `WHERE`/`JOIN` บ่อยที่สุด |
| `null: false` | **ค่า default ใหม่ตั้งแต่ Rails 7.1** — บังคับที่ระดับฐานข้อมูลว่าห้ามเป็น `NULL` ให้ตรงกับพฤติกรรม `belongs_to` required by default ที่ระดับ Model (Step 262) ทั้งสองชั้นถูกออกแบบมาให้ **สอดคล้องกันอัตโนมัติ** |
| `foreign_key: true` | เพิ่ม **foreign key constraint จริงในฐานข้อมูล** (ไม่ใช่แค่ column ตัวเลขธรรมดา) — ฐานข้อมูลจะปฏิเสธทันทีถ้าพยายามใส่ `author_id` ที่ไม่มีอยู่จริงในตาราง `authors`, และปฏิเสธการลบ `author` ที่ยังมี `book` ชี้มาอยู่ (ถ้าไม่มี `dependent:` จัดการไว้ก่อน — ดู Step 268) |

รัน migrate แล้วดู `db/schema.rb` ที่ได้:

```bash
bin/rails db:create db:migrate
```

```ruby
ActiveRecord::Schema[8.1].define(version: ...) do
  create_table "authors", force: :cascade do |t|
    t.string "name"
    t.string "nationality"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end

  create_table "books", force: :cascade do |t|
    t.string "title"
    t.integer "pages"
    t.integer "author_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["author_id"], name: "index_books_on_author_id"
  end

  add_foreign_key "books", "authors"
end
```

สังเกตบรรทัดสุดท้าย `add_foreign_key "books", "authors"` — ถูกแยกออกมาต่างหากจาก
`create_table` เสมอใน `schema.rb` (เป็นรูปแบบมาตรฐานของ Rails ไม่ต้องกังวล)

### เพิ่ม reference เข้าตารางที่มีอยู่แล้วด้วย `add_reference`

ถ้าตาราง `books` มีอยู่แล้วและอยากเพิ่มความสัมพันธ์กับ `authors` ทีหลัง (เหมือนสถานการณ์ใน
Step 261 ที่เราค่อยๆ เพิ่ม column เข้าไป) ใช้ `generate migration` ตามชื่อ convention
`Add<Column>To<Table>` แบบที่เรียนไปแล้วใน Part 025 Step 245:

```bash
bin/rails generate migration AddAuthorToBooks author:references
```

```ruby
class AddAuthorToBooks < ActiveRecord::Migration[8.1]
  def change
    add_reference :books, :author, null: false, foreign_key: true
  end
end
```

`add_reference` คือ method คู่กันกับ `t.references` แต่ใช้นอก block `create_table` (สำหรับ
แก้ไขตารางที่มีอยู่แล้ว) พฤติกรรมเหมือนกันทุกประการ — สร้าง column, index, และ (ถ้าระบุ)
foreign key constraint ให้ครบในคำสั่งเดียว

> **ข้อควรระวัง:** ถ้าตาราง `books` มีข้อมูลอยู่แล้วตอนรัน migration นี้ และใส่ `null: false`
> ไปด้วย migration จะ **fail ทันที** เพราะแถวเก่าทั้งหมดจะมี `author_id` เป็น `NULL` ซึ่งขัดกับ
> `null: false` ที่เพิ่งประกาศ วิธีแก้คือใส่ `default:` ให้ชี้ไปยัง author ชั่วคราว, เขียน
> `up`/`down` แยกเพื่ออัปเดตข้อมูลเก่าก่อนเติม constraint, หรือปล่อยเป็น `null: true` ไปก่อนแล้ว
> ค่อยตามด้วย migration `change_column_null` อีกไฟล์หลัง backfill ข้อมูลเสร็จ — เทคนิคจัดการ
> migration กับข้อมูลจริงจำนวนมากแบบปลอดภัยจะสอนละเอียดใน Part 035

---

## Step 265: `has_many` — ฝั่งตรงข้ามของ `belongs_to`

ตอนนี้ `Book belongs_to :author` ทำให้เดินจาก "ลูก" ไปหา "แม่" ได้แล้ว (`book.author`) แต่เดิน
ย้อนกลับจาก "แม่" ไปหา "ลูกทั้งหมด" (`author.books`) ยังทำไม่ได้ — ต้องเพิ่ม `has_many` ใน
`Author`:

```ruby
class Author < ApplicationRecord
  has_many :books
end
```

กฎการตั้งชื่อ: `has_many :books` ใช้ชื่อ **พหูพจน์** ของ Model ที่เชื่อมโยง (เพราะฝั่งนี้มีได้
หลาย record) และ ActiveRecord จะไปหา Model `Book` (เอกพจน์ของ `books`) ที่มี `belongs_to
:author` ชี้กลับมาหาตัวเองโดยอัตโนมัติผ่านการ join `books.author_id = authors.id`

```irb
irb(main):001> author = Author.find_by(name: "Haruki Murakami")
irb(main):002> author.books
=>
#<ActiveRecord::Relation [#<Book id: 1, title: "Norwegian Wood", ...>,
                          #<Book id: 2, title: "Kafka on the Shore", ...>,
                          #<Book id: 3, title: "1Q84", ...>]>

irb(main):003> author.books.count
=> 3

irb(main):004> author.books.pluck(:title)
=> ["Norwegian Wood", "Kafka on the Shore", "1Q84"]
```

`author.books` คืนค่าเป็น **`ActiveRecord::Relation`-like object ที่เรียกว่า
`CollectionProxy`** (เทคนิคกว่านั้น class จริงคือ `Book::ActiveRecord_Associations_CollectionProxy`
ซึ่ง subclass ของ `ActiveRecord::Relation` อีกที) มีพฤติกรรมเหมือน `Book.where(author_id:
author.id)` ทุกประการ — ต่อเงื่อนไขเพิ่มได้ (`author.books.where(pages: ...)`,
`author.books.order(:title)`) และยัง lazy evaluation เหมือนที่เรียนใน Part 025 Step 248

### SQL ที่เกิดขึ้นเบื้องหลัง

```sql
-- book.author (จาก belongs_to)
SELECT "authors".* FROM "authors" WHERE "authors"."id" = ? LIMIT 1

-- author.books (จาก has_many)
SELECT "books".* FROM "books" WHERE "books"."author_id" = ?
```

ทั้งสองทิศทางแค่เป็น `SELECT` ธรรมดาที่มีเงื่อนไข `WHERE` ตรงๆ — ไม่มีเวทมนตร์อะไรซับซ้อน
`belongs_to`/`has_many` เป็นเพียง **syntax สะดวก (syntactic sugar)** ที่ครอบ query แบบนี้ไว้ให้
เขียนสั้นลงและอ่านเป็นภาษาธรรมชาติมากขึ้นเท่านั้นเอง

### Association caching — เรียกซ้ำไม่ query ซ้ำ

```irb
irb(main):005> author.books.to_a   # ครั้งแรก: มี SQL query
irb(main):006> author.books.to_a   # ครั้งที่สอง: ไม่มี SQL query ใหม่ (ใช้ผลลัพธ์ที่ cache ไว้)
```

พิสูจน์ด้วย log จริง (`log/development.log` หรือเปิด SQL log ใน console):

```
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 2
-- (เรียกซ้ำครั้งที่สอง ไม่มี query ใหม่เกิดขึ้นเลย)
```

ครั้งแรกที่ `author.books` ถูกเข้าถึง (ไม่ว่าจะผ่าน `.to_a`, `.each`, `.map` ฯลฯ) ActiveRecord
จะยิง query แล้วเก็บผลลัพธ์ไว้ใน object เรียกว่า **`target`** — การเรียกซ้ำครั้งถัดไปบน
**instance เดียวกัน** จะใช้ค่าที่ cache ไว้ทันทีโดยไม่ query ซ้ำ ถ้าต้องการบังคับให้ query ใหม่จาก
ฐานข้อมูล (เช่น มีการเปลี่ยนแปลงข้อมูลจากที่อื่นระหว่างนั้น) ใช้ `.reload`:

```irb
irb(main):007> author.books.reload   # บังคับ query ใหม่เสมอ
```

> **ข้อควรระวัง:** cache นี้อยู่ใน **object instance เดียวกันเท่านั้น** — ถ้าโหลด `author` คนละ
> ครั้ง (`Author.find(2)` สองรอบ) จะได้ object คนละตัว แต่ละตัวมี cache ของตัวเอง ไม่ได้แชร์กัน
> ข้าม request หรือข้าม object และไม่เกี่ยวกับ Rails cache (`Rails.cache`) ที่จะเรียนใน Part 063
> แต่อย่างใด

---

## Step 266: `has_one` — ความสัมพันธ์แบบหนึ่งต่อหนึ่ง

`has_many` ใช้เมื่อฝั่งหนึ่งมีได้ **หลาย** record (Author หนึ่งคนมีหลายเล่ม) แต่บางความสัมพันธ์
ควรมีได้ **แค่หนึ่งเดียว** เท่านั้น เช่น ผู้แต่งหนึ่งคนมีโปรไฟล์ประวัติได้แค่ชุดเดียว — กรณีนี้ใช้
`has_one` แทน

สร้าง Model `Profile` ใหม่:

```bash
bin/rails generate model Profile bio:text author:references
```

migration ที่ได้ (เพิ่ม `index: { unique: true }` เองเพื่อบังคับที่ระดับฐานข้อมูลว่า author หนึ่งคน
มี profile ได้แค่แถวเดียว — ป้องกันข้อมูลซ้ำซ้อนแม้จะมีคนพยายามสร้างสองแถวพร้อมกันแบบ race
condition ก็ตาม):

```ruby
class CreateProfiles < ActiveRecord::Migration[8.1]
  def change
    create_table :profiles do |t|
      t.text :bio
      t.references :author, null: false, foreign_key: true, index: { unique: true }

      t.timestamps
    end
  end
end
```

```bash
bin/rails db:migrate
```

`app/models/profile.rb` ที่ generator สร้างให้ (เหมือน Step 264 คือได้ `belongs_to` มาให้ฟรี):

```ruby
class Profile < ApplicationRecord
  belongs_to :author
end
```

เพิ่ม `has_one :profile` ใน `Author`:

```ruby
class Author < ApplicationRecord
  has_many :books
  has_one :profile
end
```

**สังเกตความแตกต่างจาก `has_many`:** `has_one :profile` ใช้ชื่อ **เอกพจน์** (ไม่ใช่ `profiles`)
เพราะ Rails รู้อยู่แล้วว่าฝั่งนี้จะมีได้แค่หนึ่งเดียว

### ใช้งาน `has_one`

```irb
irb(main):001> author = Author.find_by(name: "Haruki Murakami")
irb(main):002> author.profile
=> nil
```

ก่อนสร้างจะได้ `nil` เพราะยังไม่มี record profile อยู่จริง (ต่างจาก `has_many` ที่ไม่มี record
เลยจะได้ Relation ว่างเปล่า `[]` ไม่ใช่ `nil`) — สร้างผ่าน association โดยตรง:

```irb
irb(main):003> profile = author.create_profile(bio: "นักเขียนชาวญี่ปุ่น มีชื่อเสียงจากแนวสัจนิยมมหัศจรรย์")
=> #<Profile id: 1, bio: "...", author_id: 2, ...>

irb(main):004> author.reload.profile.bio
=> "นักเขียนชาวญี่ปุ่น มีชื่อเสียงจากแนวสัจนิยมมหัศจรรย์"
```

`create_profile` เป็น method พิเศษที่ `has_one`/`has_many` generate ให้อัตโนมัติ (รายละเอียด
method ทั้งหมดสรุปเป็นตารางใน Step 270) — เทียบเท่ากับ `Profile.create(author: author, bio:
...)` แต่เขียนจากฝั่ง `author` ได้เลยโดยไม่ต้องระบุ `author:` เอง

ลองสร้างซ้ำเป็นตัวที่สอง:

```irb
irb(main):005> author.create_profile(bio: "อีกฉบับ")
```

```
ActiveRecord::RecordNotUnique: SQLite3::ConstraintException: UNIQUE constraint failed:
profiles.author_id
```

**unique index** ที่ใส่ไว้ตอน migration (`index: { unique: true }`) ทำงานตามหน้าที่ — ป้องกัน
ไม่ให้ author คนเดียวกันมี profile ซ้ำสองแถวได้ในระดับฐานข้อมูล นี่คือเหตุผลว่าทำไมทุกครั้งที่ใช้
`has_one` ควรใส่ unique index คู่กับ foreign key เสมอ ไม่งั้นแม้ฝั่ง Ruby จะมองว่า "มีได้แค่หนึ่ง"
แต่ฐานข้อมูลจะยอมให้มีหลายแถวชนกันได้เงียบๆ ถ้ามีสอง request มาสร้างพร้อมกันพอดี (race
condition) — `has_one` ที่ฝั่ง Model บอกแค่ "เวลาเราไป **อ่าน** จะดึงมาแค่แถวแรก" เท่านั้น
ไม่ได้บังคับที่ระดับฐานข้อมูลให้อัตโนมัติ

> **`has_one` vs `belongs_to`:** ทั้งคู่แสดงถึงความสัมพันธ์ "หนึ่ง" เหมือนกัน แต่ต่างกันที่ว่า
> **foreign key column อยู่ฝั่งไหน** — `belongs_to` ใช้กับฝั่งที่ **มี** foreign key column
> (`profiles.author_id`) ส่วน `has_one`/`has_many` ใช้กับฝั่งที่ **ไม่มี** foreign key column
> ของตัวเอง (ตาราง `authors` ไม่มี column `profile_id`) กฎง่ายๆ ในการจำ: **column `xxx_id`
> อยู่ตารางไหน ตารางนั้นเขียน `belongs_to :xxx`**

---

## Step 267: สร้าง/จัดการ record ผ่าน association โดยตรง (`build`, `create`, `<<`, `=`)

เมื่อมี `has_many`/`has_one` แล้ว การสร้าง record ใหม่ที่ผูกกับ parent อัตโนมัติทำได้สะดวกกว่า
`Model.create(foreign_key: parent.id)` แบบ Step 261 มาก

### `association.create` — สร้างและ save พร้อมผูก foreign key ให้อัตโนมัติ

```irb
irb(main):001> author = Author.find_by(name: "Haruki Murakami")
irb(main):002> book = author.books.create(title: "Norwegian Wood", pages: 296)
=> #<Book id: 1, title: "Norwegian Wood", pages: 296, author_id: 2, ...>

irb(main):003> book.author_id == author.id
=> true
```

ไม่ต้องเขียน `author_id: author.id` เองเลย — `author.books.create(...)` รู้เองว่ากำลังสร้าง
`Book` ที่ควรผูกกับ `author` ตัวนี้ มี `create!` (มี `!`) คู่กันที่ raise exception แทนคืน `false`
เมื่อ validation ไม่ผ่าน เหมือนหลักการ `!` ที่เรียนมาตั้งแต่ Part 025

### `association.build` — สร้างในหน่วยความจำ ยังไม่ save

```irb
irb(main):004> book2 = author.books.build(title: "Kafka on the Shore", pages: 505)
irb(main):005> book2.persisted?
=> false
irb(main):006> book2.author_id
=> 2
irb(main):007> book2.save
=> true
```

`build` เทียบเท่ากับ `Book.new(author: author, title: ..., pages: ...)` — **แต่ต่างจาก
`Model.new` ตรงๆ ตรงที่ `author_id` ถูกเติมให้ทันทีตั้งแต่ตอน build** (สังเกตขั้นตอนที่ 6 ข้างบน
`book2.author_id` มีค่าอยู่แล้วทั้งที่ยังไม่ `save`) มีประโยชน์มากเวลาสร้าง form สำหรับ nested
resource (เช่น "เพิ่มหนังสือใหม่ให้ผู้แต่งคนนี้") ซึ่งจะเจอเต็มๆ ใน Part 031 และ Part 037

### `<<` — เพิ่มเข้า collection แล้ว save ทันที (ถ้า parent persisted แล้ว)

```irb
irb(main):008> new_book = Book.new(title: "1Q84", pages: 925)
irb(main):009> author.books << new_book
irb(main):010> new_book.persisted?
=> true
```

`<<` (หรือใช้ชื่อเต็ม `.concat`) รับ object ที่สร้างไว้แล้ว (ไม่ว่าจะ save แล้วหรือยังก็ตาม) มาเพิ่ม
เข้ากลุ่มและ save ให้ทันที (ยิง `UPDATE` ตั้งค่า `author_id` ให้ ถ้า `new_book` ยังไม่เคย save)

### `association.ids` และ `id_ids=` — จัดการทั้งกลุ่มผ่านรายการ id

```irb
irb(main):011> author.book_ids
=> [1, 2, 3]

irb(main):012> author.book_ids = [1, 2]
```

`book_ids=` ตั้งค่าทั้งกลุ่มใหม่ทั้งหมดในคำสั่งเดียวจากรายการ id — เล่มที่ **หลุดออกจากรายการ**
(id 3 ในตัวอย่างนี้) จะถูกจัดการตาม `dependent:` ที่ตั้งไว้ (ถ้าเป็น `:destroy` จะถูกลบทิ้งไปเลย
ถ้าเป็น `:nullify` จะแค่เอา `author_id` ออก — รายละเอียดเต็มอยู่ใน Step 268) มีประโยชน์มากตอน
เขียน form ที่ใช้ multiple-select เลือกความสัมพันธ์

### `association.delete` — เอาออกจากกลุ่ม (ไม่เท่ากับ `destroy`)

```irb
irb(main):013> book_to_remove = author.books.last
irb(main):014> author.books.delete(book_to_remove)
```

`delete` เอา record ออกจาก "กลุ่มของ author คนนี้" เท่านั้น — record ตัวนั้นจะถูกลบทิ้งจริงหรือแค่
ตัด foreign key ออก (กลายเป็น orphan หรือ `nil`) ก็ขึ้นอยู่กับ `dependent:` ที่ตั้งไว้เหมือนกัน
(อธิบายละเอียดใน Step 268)

### `count`, `size`, `length` — คนละพฤติกรรมที่มักสับสน

```irb
irb(main):015> author.books.count    # ยิง SQL COUNT(*) เสมอ ไม่สนใจว่า cache ไว้หรือยัง
irb(main):016> author.books.size     # ถ้า cache ไว้แล้วนับจาก memory, ถ้ายังไม่ cache ยิง COUNT
irb(main):017> author.books.length   # โหลดทั้งหมดเข้า memory ก่อนเสมอ แล้วนับความยาว Array
```

กฎทางปฏิบัติ: **ใช้ `size` เป็นค่าเริ่มต้นเสมอ** เพราะฉลาดเลือกวิธีที่มี query น้อยที่สุดให้อัตโนมัติ
ใช้ `count` เฉพาะตอนต้องการนับอย่างเดียวจริงๆ โดยไม่สนใจข้อมูลอื่น (และรู้ว่าจะไม่ใช้ผลลัพธ์ที่
โหลดมาแล้วต่อ) และใช้ `length` เฉพาะตอนที่ **แน่ใจว่าโหลดข้อมูลมาใช้ต่ออยู่แล้ว** ในโค้ดส่วนนั้น
(เช่น loop `each` ต่อทันที) เพื่อไม่ต้อง query ซ้ำสองรอบ

---

## Step 268: `dependent:` — `:destroy`, `:destroy_async`, `:nullify`, `:restrict_with_error` และอันตรายของการเลือกผิด

คำถามสำคัญที่ยังไม่ได้ตอบ: **ถ้าลบ `Author` ที่ยังมี `Book` ผูกอยู่ จะเกิดอะไรขึ้นกับ Book
เหล่านั้น?** คำตอบขึ้นอยู่กับตัวเลือก `dependent:` ที่ใส่ไว้ใน `has_many`/`has_one` — ถ้าไม่ใส่
อะไรเลย (ค่า default) จะเกิดพฤติกรรมที่ **อันตรายมาก** ต้องเข้าใจให้ครบก่อนเลือกใช้

### ไม่ใส่ `dependent:` เลย — อันตรายซ่อนอยู่

```ruby
class Author < ApplicationRecord
  has_many :books   # ไม่มี dependent:
end
```

```irb
irb(main):001> author = Author.create!(name: "No Dependent Author")
irb(main):002> author.books.create!(title: "Orphan Candidate", pages: 50)
irb(main):003> author.destroy
```

ถ้าตาราง `books` มี **foreign key constraint จริงในฐานข้อมูล** (จาก `foreign_key: true` ตอน
migration ใน Step 264) ผลลัพธ์คือ:

```
ActiveRecord::InvalidForeignKey: SQLite3::ConstraintException: FOREIGN KEY constraint failed
```

ฐานข้อมูลปฏิเสธการลบทันที เพราะยังมี `Book` ชี้มาที่ `author` ตัวนี้อยู่ — **นี่ยังถือว่าปลอดภัย**
เพราะอย่างน้อยก็ไม่ปล่อยให้ข้อมูลเสียหาย แต่ผู้ใช้จะเจอ error หน้าตาน่ากลัวที่ไม่มีการจัดการ
(ถ้าไม่ได้ดักไว้ใน controller)

**แต่ถ้าตารางไม่มี foreign key constraint ระดับฐานข้อมูล** (เช่น migration เก่าที่ไม่ได้ใส่
`foreign_key: true`, หรือฐานข้อมูลบางระบบที่ไม่บังคับ) ผลลัพธ์จะแย่กว่ามาก — **author ถูกลบไป
เฉยๆ โดยไม่มีการเตือนใดๆ** ปล่อยให้ book กลายเป็น **orphaned record** (ข้อมูลกำพร้า) ที่
`author_id` ชี้ไปยัง id ที่ไม่มีอยู่จริงในระบบแล้ว:

```irb
irb(main):004> orphan = Book.find(book.id)
irb(main):005> orphan.author_id
=> 5   # <- author id นี้ไม่มีอยู่จริงในตาราง authors แล้ว!
irb(main):006> orphan.author
=> nil   # เข้าถึงแล้วได้ nil เงียบๆ ไม่ error แต่ข้อมูลเสียหายไปแล้ว
```

โค้ดส่วนอื่นในแอปที่สมมติไว้ว่า `book.author` ต้องมีค่าเสมอ (เช่น view ที่เขียน
`book.author.name` ตรงๆ โดยไม่เช็ค `nil` ก่อน) จะพังทันทีด้วย `NoMethodError` ตอนรันจริง
ซึ่งกว่าจะรู้ตัวก็อาจสายไปแล้ว — **นี่คือเหตุผลว่าทำไมทุก `has_many`/`has_one` ควรตัดสินใจเรื่อง
`dependent:` อย่างตั้งใจเสมอ ไม่ปล่อยว่างไว้โดยไม่คิด**

### `dependent: :destroy` — ลบตามไปด้วยทั้งหมด (ตัวเลือกปลอดภัยที่สุดโดยทั่วไป)

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end
```

```irb
irb(main):001> author = Author.create!(name: "Destroy Author")
irb(main):002> author.books.create!(title: "Book A", pages: 100)
irb(main):003> author.books.create!(title: "Book B", pages: 200)
irb(main):004> Author.count
=> 1
irb(main):005> Book.count
=> 2

irb(main):006> author.destroy
irb(main):007> Author.count
=> 0
irb(main):008> Book.count
=> 0
```

`dependent: :destroy` เรียก `.destroy` กับ book ทุกเล่มที่ผูกอยู่ **ก่อน** ลบ author (เรียงลำดับ
ถูกต้อง ไม่ชน foreign key constraint) และเพราะเป็น `.destroy` (ไม่ใช่ `.delete`) จึงรัน callback
และ validation ของ `Book` ตามปกติ, และถ้า `Book` เองก็มี `has_many` ของตัวเองที่ตั้ง
`dependent: :destroy` ไว้ ก็จะไล่ลบต่อเป็นทอดๆ (cascade) ทั้งสาย — เหมาะกับข้อมูลที่ "ไม่มี
ความหมายถ้าไม่มี parent" เช่น `Comment` ที่ไม่ควรอยู่ได้โดยไม่มี `Post`

**ข้อควรระวัง:** ถ้า parent มี child จำนวนมาก (หลักหมื่น-แสนแถว) `dependent: :destroy` จะ
**ช้ามาก** เพราะต้องโหลดทุกแถวเข้า memory แล้วยิง `DELETE` ทีละแถวเพื่อให้ callback ทำงานครบ
ทุกตัว — กรณีนี้ควรพิจารณา `:destroy_async` แทน

### `dependent: :destroy_async` — ลบแบบ background job (Rails 6.1+)

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy_async
end
```

```irb
irb(main):001> author = Author.create!(name: "Async Author")
irb(main):002> author.books.create!(title: "Async Book", pages: 20)
irb(main):003> author.destroy
irb(main):004> Book.count
=> 1   # <- ยังไม่ถูกลบทันที! รอ background job ทำงานก่อน
irb(main):005> sleep 1
irb(main):006> Book.count
=> 0   # <- background job ทำงานเสร็จแล้ว
```

log ที่เกิดขึ้นเบื้องหลังแสดงให้เห็นชัดว่ามีการ enqueue job แยกออกไปทำงานทีหลัง:

```
[ActiveJob] Enqueued ActiveRecord::DestroyAssociationAsyncJob (Job ID: ...) to Async(default)
  with arguments: {:owner_model_name=>"Author", :owner_id=>11, :association_class=>"Book", ...}
...
[ActiveJob] [ActiveRecord::DestroyAssociationAsyncJob] Performing ... from Async(default)
[ActiveJob] [ActiveRecord::DestroyAssociationAsyncJob]   Book Destroy (1.1ms)  DELETE FROM "books" WHERE "books"."id" = 4
[ActiveJob] [ActiveRecord::DestroyAssociationAsyncJob] Performed ... in 6.43ms
```

`author` ถูกลบทันทีในทรานแซกชันหลัก ส่วนการลบ `book` แต่ละเล่มถูกส่งไปทำเป็น background job
ผ่าน Active Job (ค่า default ใช้ adapter `:async` ซึ่งรันใน thread pool ของ process เดียวกัน —
โปรดักชันจริงมักตั้งเป็น Sidekiq/Solid Queue ซึ่งจะสอนใน Part 061–062) ทำให้ request/response
ของผู้ใช้ไม่ต้องรอการลบข้อมูลจำนวนมากให้เสร็จก่อน

> **กับดักสำคัญของ `:destroy_async`:** ถ้าตารางมี **foreign key constraint ระดับฐานข้อมูล**
> (`foreign_key: true`) การลบ `author` แบบ **synchronous ในทรานแซกชันเดียวกัน** จะ **ล้มเหลว
> ทันที** ด้วย `ActiveRecord::InvalidForeignKey` เพราะฐานข้อมูลเห็นว่า book ยังชี้มาอยู่ตอนที่
> พยายามลบ author (การลบ book ถูกเลื่อนไปทำใน job หลัง commit แต่ FK constraint ตรวจสอบ
> ทันทีในทรานแซกชันปัจจุบัน) วิธีแก้ในทางปฏิบัติ (โดยเฉพาะบน PostgreSQL) คือประกาศ foreign key
> เป็น `deferrable: :deferred` ตอนสร้าง — รายละเอียดเชิงลึกเรื่องนี้จะกลับมาพูดถึงใน Part 064
> ตอนนี้ให้จำหลักไว้ก่อนว่า **`:destroy_async` กับ foreign key constraint ที่เข้มงวดไปด้วยกันได้
> ไม่ราบรื่นเสมอไป ต้องทดสอบให้แน่ใจก่อนใช้จริงกับตารางที่มี constraint**

### `dependent: :nullify` — ตัดความสัมพันธ์ ไม่ลบข้อมูล

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :nullify
end
```

```irb
irb(main):001> author = Author.create!(name: "Nullify Author")
irb(main):002> author.books.create!(title: "Nullify Candidate", pages: 30)
irb(main):003> author.destroy
```

```
ActiveRecord::NotNullViolation: NOT NULL constraint failed: books.author_id
```

**เกิด error ทันที เพราะ `nullify` พยายามตั้ง `author_id = NULL`** แต่ column นี้ถูกสร้างด้วย
`null: false` มาตั้งแต่ Step 264 — ฐานข้อมูลปฏิเสธ นี่คือตัวอย่างชัดเจนของ **"เลือก `dependent:`
ผิดประเภทให้เข้ากับ schema"** — `:nullify` ใช้ได้เฉพาะเมื่อ foreign key column เป็น **nullable
เท่านั้น** (`optional: true` คู่กับ `null: true` แบบ Step 263) ลองใหม่หลังปรับ column ให้
nullable:

```irb
irb(main):004> author2 = Author.create!(name: "Nullify Author 2")
irb(main):005> book = author2.books.create!(title: "Nullify Candidate 2", pages: 30)
irb(main):006> author2.destroy
irb(main):007> Author.count
=> 0
irb(main):008> Book.count
=> 1   # <- book ยังอยู่ ไม่ถูกลบ
irb(main):009> book.reload.author_id
=> nil   # <- แค่ถูกตัดความสัมพันธ์ออก
```

`:nullify` เหมาะกับกรณีที่ child record **ยังมีความหมายอยู่ได้ด้วยตัวเอง** แม้ไม่มี parent แล้ว
เช่น `Order belongs_to :promoted_by_user, optional: true` — ถ้าลบ user คนที่แนะนำ order นั้นยัง
ต้องเก็บไว้เป็นประวัติ แค่ไม่ต้องรู้แล้วว่าใครแนะนำ

### `dependent: :restrict_with_error` — ห้ามลบถ้ายังมี child อยู่

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :restrict_with_error
end
```

```irb
irb(main):001> author = Author.create!(name: "Restrict Author")
irb(main):002> author.books.create!(title: "Protected Book", pages: 40)

irb(main):003> result = author.destroy
=> false
irb(main):004> author.errors.full_messages
=> ["Cannot delete record because dependent books exist"]
irb(main):005> Author.count
=> 1   # <- ยังอยู่ครบ ไม่ถูกลบ
irb(main):006> Book.count
=> 1
```

`:restrict_with_error` ทำให้ `destroy` **ล้มเหลวอย่างสุภาพ** (คืน `false` พร้อม error message
อ่านได้ ไม่ raise exception ดิบๆ แบบ foreign key constraint) เหมาะกับสถานการณ์ที่ต้องการบังคับ
ให้ผู้ใช้ "จัดการ child ให้เรียบร้อยก่อน" ถึงจะลบ parent ได้ เช่น ห้ามลบ `Category` ที่ยังมี
`Product` อยู่ในหมวดนั้น จนกว่าจะย้าย/ลบสินค้าทั้งหมดออกก่อน (มี
`:restrict_with_exception` เป็นตัวเลือกคู่กันที่ raise
`ActiveRecord::DeleteRestrictionError` แทนการคืน `false` — ใช้น้อยกว่าเพราะจัดการยากกว่า
ใน controller)

### ตารางสรุปเปรียบเทียบ

| `dependent:` | เกิดอะไรกับ child | รัน callback ของ child ไหม | เหมาะกับ |
|---------------|----------------------|--------------------------------|-----------|
| (ไม่ใส่) | ขึ้นกับ DB constraint: error ถ้ามี FK, **orphan เงียบๆ ถ้าไม่มี FK** | ไม่ | ไม่ควรปล่อยว่างโดยไม่คิด |
| `:destroy` | ลบทุก child ทีละตัว (cascade ต่อได้) | รัน | child ไม่มีความหมายถ้าไม่มี parent (เช่น Comment) |
| `:destroy_async` | เหมือน `:destroy` แต่ทำเป็น background job | รัน (ใน job) | child จำนวนมาก, ไม่อยากให้ request ค้าง |
| `:delete_all` | ลบด้วย SQL ตรงๆ ไม่ผ่าน callback | **ไม่รัน** | ต้องการความเร็ว มั่นใจว่าไม่ต้องการ callback |
| `:nullify` | ตัด foreign key เป็น `NULL` (ต้องเป็น nullable column) | ไม่ (แค่ update) | child อยู่ได้เองโดยไม่มี parent |
| `:restrict_with_error` | ห้ามลบ parent, เติม error แทน | ไม่ (ยกเลิกทั้งหมด) | ต้องการบังคับให้จัดการ child ก่อนเสมอ |

> **หลักการเลือก:** ถามตัวเองก่อนเสมอว่า "child record นี้ยังมีความหมายอยู่ได้ไหมถ้าไม่มี parent
> แล้ว" — ถ้า**ไม่มีความหมาย** ใช้ `:destroy`/`:destroy_async`, ถ้า**ยังมีความหมายอยู่** ใช้
> `:nullify`, และถ้า**ไม่ควรให้ลบ parent ได้เลยตราบใดที่ยังมี child** ใช้
> `:restrict_with_error` — ที่แน่นอนที่สุดคือ**อย่าปล่อยว่างไว้โดยไม่ตัดสินใจ** เพราะพฤติกรรม
> default เปลี่ยนไปตามว่ามี foreign key constraint ในฐานข้อมูลหรือไม่ ซึ่งเป็นเรื่องที่คาดเดายากและ
> อันตรายที่สุดในบรรดาตัวเลือกทั้งหมด

---

## Step 269: ปัญหา N+1 query — เห็นด้วยตาเปล่าผ่าน SQL log แล้วแก้เบื้องต้นด้วย `includes`

Association ทำให้เขียนโค้ดอ่านง่ายขึ้นมาก แต่มาพร้อมกับดักประสิทธิภาพที่คลาสสิกที่สุดของ
ActiveRecord — **N+1 query problem** ลองสร้างข้อมูลตัวอย่าง 3 คน คนละ 2 เล่ม:

```irb
irb(main):001> authors = [
irb(main):002>   Author.create!(name: "Haruki Murakami"),
irb(main):003>   Author.create!(name: "Neil Gaiman"),
irb(main):004>   Author.create!(name: "Yuval Noah Harari")
irb(main):005> ]
irb(main):006> authors.each_with_index do |a, i|
irb(main):007>   a.books.create!(title: "Book #{i}-A", pages: 100)
irb(main):008>   a.books.create!(title: "Book #{i}-B", pages: 200)
irb(main):009> end
```

### แบบมี N+1 — ไม่ใช้ `includes`

```ruby
Author.all.each do |author|
  puts "#{author.name}: #{author.books.map(&:title).join(', ')}"
end
```

SQL log ที่เกิดขึ้นจริง:

```
Author Load (0.1ms)  SELECT "authors".* FROM "authors"
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 16
Haruki Murakami: Book 0-A, Book 0-B
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 17
Neil Gaiman: Book 1-A, Book 1-B
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 18
Yuval Noah Harari: Book 2-A, Book 2-B
```

นับ query ได้ **4 ครั้ง**: 1 query แรกดึง `Author` ทั้งหมด (`N` = 3 คน) บวกอีก **3 query**
แยกไปดึง `books` ของผู้แต่งแต่ละคนทีละคน — สูตรคือ **1 (ดึง parent) + N (ดึง child ทีละตัว) =
N+1 query** ถ้ามีผู้แต่ง 1,000 คน จะกลายเป็น 1,001 query ทันที ทั้งที่ต้องการแค่ "แสดงชื่อหนังสือ
ของทุกคน" เท่านั้น — ยิ่งข้อมูลเยอะ ยิ่งช้าเป็นเส้นตรง (linear) ตามจำนวนแถว เป็นสาเหตุอันดับ
ต้นๆ ของเว็บ Rails ที่ช้าในโลกจริง

### แบบแก้ด้วย `includes` — Eager Loading

```ruby
Author.includes(:books).each do |author|
  puts "#{author.name}: #{author.books.map(&:title).join(', ')}"
end
```

SQL log ที่เกิดขึ้น:

```
Author Load (0.1ms)  SELECT "authors".* FROM "authors"
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" IN (16, 17, 18)
Haruki Murakami: Book 0-A, Book 0-B
Neil Gaiman: Book 1-A, Book 1-B
Yuval Noah Harari: Book 2-A, Book 2-B
```

เหลือแค่ **2 query เท่านั้น** ไม่ว่าจะมีผู้แต่งกี่คนก็ตาม: query แรกดึง `Author` ทั้งหมดเหมือนเดิม
แต่ query ที่สองดึง `Book` **ของทุกคนในครั้งเดียว** ด้วย `WHERE author_id IN (16, 17, 18)`
แล้ว ActiveRecord จะจับคู่ผลลัพธ์ให้ตรงกับ author แต่ละคนใน memory เอง (ไม่ต้อง query แยกทีละคน
อีกต่อไป) — นี่คือเทคนิคที่เรียกว่า **Eager Loading** (โหลดล่วงหน้าทีเดียวแทนโหลดทีละตัวตามต้องใช้)

### ทำไม `includes` ถึงลดจาก N+1 เหลือ 2

`Author.includes(:books)` บอก ActiveRecord ว่า "รู้อยู่แล้วว่าต้องใช้ `books` ของทุก author ที่
ดึงมา ดังนั้นดึงมาให้ครบตั้งแต่ต้นเลย" — พอถึงตอน loop เรียก `author.books` แต่ละครั้ง
ActiveRecord จะเจอว่า **cache ไว้อยู่แล้ว** จากที่ preload มาตอนต้น (เหมือนหลักการ association
caching ที่เรียนใน Step 265) จึงไม่ต้อง query ซ้ำอีกเลยสักครั้งเดียวตลอด loop

> **หมายเหตุสำคัญ:** `includes` เป็นแค่หนึ่งในหลายเทคนิคแก้ N+1 (ยังมี `preload`, `eager_load`
> ที่พฤติกรรมต่างกันเล็กน้อย, และเครื่องมือช่วยตรวจจับ N+1 อัตโนมัติอย่าง gem `bullet`) Part นี้
> แค่แนะนำให้รู้จักปัญหาและวิธีแก้เบื้องต้นที่สุดก่อน — รายละเอียดเชิงลึกเรื่อง query interface
> ทั้งหมด (`where`, `joins`, `includes` vs `preload` vs `eager_load`, `scope`) จะไปเรียนเต็มๆ
> ใน **Part 034**

---

## Step 270: `inverse_of`, method ที่ Rails generate ให้อัตโนมัติทั้งหมด + แบบฝึกหัด

### ปัญหาที่ `inverse_of` แก้: object คนละตัวกันในหน่วยความจำ

โดยปกติ Rails ฉลาดพอที่จะรู้เอง (**automatic inverse detection**) ว่า `has_many :books` ใน
`Author` กับ `belongs_to :author` ใน `Book` คือ "อีกฝั่งของความสัมพันธ์เดียวกัน" ตราบใดที่ตั้งชื่อ
ตาม convention ปกติ ทำให้เดินไปมาระหว่างสองฝั่งได้โดยไม่ query ซ้ำและได้ object instance
เดียวกันเป๊ะ:

```irb
irb(main):001> author = Author.create!(name: "ต้นฉบับ")
irb(main):002> book = author.books.build(title: "เล่มปกติ", pages: 100)
irb(main):003> book.author.equal?(author)
=> true
```

แต่พอไหนก็ตามที่ **ชื่อ association สองฝั่งไม่ตรงกันตาม convention** (เช่น ตั้งชื่อ custom
เอง, ใช้ `class_name`/`foreign_key` ที่ไม่ตรงมาตรฐาน) automatic detection จะ**ล้มเหลว**
ทำให้ได้ object คนละตัวกันเวลาเดินข้ามไปมา — ลองดูตัวอย่างที่จำลองปัญหานี้:

```ruby
# app/models/book.rb — ตั้งชื่อ association ว่า :writer แทน :author (custom name)
class Book < ApplicationRecord
  belongs_to :writer, class_name: "Author", foreign_key: "author_id"
end
```

```irb
irb(main):001> author = Author.create!(name: "ต้นฉบับ")
irb(main):002> author.books.create!(title: "เล่มหนึ่ง", pages: 88)
irb(main):003> author = Author.find(author.id)   # โหลดใหม่จาก DB

irb(main):004> book = author.books.first
irb(main):005> book.writer.equal?(author)
=> false   # <- คนละ object กัน! ทั้งที่ควรจะเป็น author คนเดียวกัน

irb(main):006> author.name = "แก้ไขใน memory"
irb(main):007> book.writer.name
=> "ต้นฉบับ"   # <- ยังเป็นค่าเก่า! ไม่เห็นการแก้ไขที่เพิ่งทำ เพราะเป็นคนละ object กัน
```

`book.writer` ยิง query ใหม่ไปหาฐานข้อมูลเองเงียบๆ (แทนที่จะใช้ `author` ที่โหลดมาแล้ว) ได้
object คนละตัวที่มีข้อมูลเป็น **snapshot ตอน query** — ถ้าโค้ดส่วนอื่นแก้ไข `author` ใน memory
ไปแล้วยังไม่ save, ฝั่ง `book.writer` จะไม่มีทางเห็นการเปลี่ยนแปลงนั้นเลย เป็นบั๊กที่ debug ยากมาก
เพราะไม่มี error ใดๆ เกิดขึ้น แค่ข้อมูล "เพี้ยน" เงียบๆ

### แก้ด้วย `inverse_of` — บอก Rails ตรงๆ ว่าอีกฝั่งคือใคร

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :books, inverse_of: :writer
end

# app/models/book.rb
class Book < ApplicationRecord
  belongs_to :writer, class_name: "Author", foreign_key: "author_id", inverse_of: :books
end
```

ต้องใส่ `inverse_of:` **ทั้งสองฝั่ง** ให้ชี้กลับไปหาชื่อ association ของอีกฝั่ง ทดสอบใหม่:

```irb
irb(main):001> author = Author.create!(name: "ต้นฉบับ")
irb(main):002> author.books.create!(title: "เล่มหนึ่ง", pages: 88)
irb(main):003> author = Author.find(author.id)

irb(main):004> book = author.books.first
irb(main):005> book.writer.equal?(author)
=> true   # <- object เดียวกันแล้ว

irb(main):006> author.name = "แก้ไขใน memory"
irb(main):007> book.writer.name
=> "แก้ไขใน memory"   # <- เห็นการเปลี่ยนแปลงทันที เพราะเป็น object เดียวกันจริงๆ
```

> **เมื่อไหร่ต้องใส่ `inverse_of` เอง:** Rails auto-detect ให้ได้เองในเกือบทุกกรณีมาตรฐาน (ชื่อ
> association ตรงตาม convention ทั้งสองฝั่ง แม้จะมี scope หรือเงื่อนไขเพิ่มก็ตาม) กรณีที่ต้อง
> ใส่เองมักเจอตอน: (1) ตั้งชื่อ association ไม่ตรงกับชื่อ Model แบบ `:writer` ในตัวอย่างนี้,
> (2) ใช้ `foreign_key`/`class_name` ที่ทำให้เดาชื่อกลับไม่ได้, (3) ใช้กับ `has_many :through`
> หรือ polymorphic association (Part 033) ซึ่ง auto-detect ไม่รองรับเลย ถ้าไม่แน่ใจ ตรวจสอบได้
> ด้วย `Model.reflect_on_association(:name).inverse_of` — ถ้าได้ `nil` แปลว่ายังไม่ถูกตั้งค่า
> ควรใส่ `inverse_of:` เองให้ชัดเจน

### ตารางสรุป: Method ที่ Rails generate ให้อัตโนมัติทั้งหมด

จำหลักการจาก Part 025 Step 250 ได้ไหม — ActiveRecord อ่าน schema แล้ว generate attribute
method ให้อัตโนมัติ หลักการเดียวกันนี้ใช้กับ **association** ด้วย: เขียน `belongs_to`/
`has_many`/`has_one` หนึ่งบรรทัด แล้ว Rails จะ define method อีกนับสิบตัวให้ใช้งานได้ทันที
โดยไม่ต้องเขียนเอง สรุปครบทุกตัวที่ตรวจสอบแล้วว่าใช้งานได้จริง:

**สำหรับ `belongs_to :author` (เขียนใน `Book`):**

| Method | ทำอะไร |
|---------|---------|
| `book.author` | คืน `Author` ที่ผูกอยู่ (หรือ `nil` ถ้าเป็น optional และยังไม่ตั้งค่า) |
| `book.author=(author)` | ตั้งค่า author ใหม่ (เปลี่ยน `author_id` ใน memory ทันที) |
| `book.build_author(attrs)` | สร้าง `Author` ใหม่ในหน่วยความจำ (ยังไม่ save) แล้วผูกกับ `book` ทันที |
| `book.create_author(attrs)` | สร้างและ save `Author` ใหม่ แล้วผูกกับ `book` ทันที |
| `book.create_author!(attrs)` | เหมือนข้างบนแต่ raise exception ถ้า validation ไม่ผ่าน |
| `book.reload_author` | บังคับโหลด author ใหม่จากฐานข้อมูล (ไม่ใช้ cache) |

**สำหรับ `has_many :books` (เขียนใน `Author`):**

| Method | ทำอะไร |
|---------|---------|
| `author.books` | คืน `CollectionProxy` ของหนังสือทั้งหมด (ใช้เหมือน `ActiveRecord::Relation`) |
| `author.books=(array)` | แทนที่ทั้งกลุ่มด้วย array ใหม่ (ตัวที่หลุดถูกจัดการตาม `dependent:`) |
| `author.book_ids` | คืน array ของ `id` ทั้งหมด แทนที่จะเป็น object เต็ม |
| `author.book_ids=(ids)` | แทนที่ทั้งกลุ่มด้วยรายการ id |
| `author.books.build(attrs)` / `.new(attrs)` | สร้าง `Book` ใหม่ในหน่วยความจำ ผูก `author_id` ให้แล้ว |
| `author.books.create(attrs)` | สร้างและ save `Book` ใหม่ทันที |
| `author.books.create!(attrs)` | เหมือนข้างบนแต่ raise exception ถ้า validation ไม่ผ่าน |
| `author.books << book` | เพิ่ม book เข้ากลุ่มแล้ว save ทันที |
| `author.books.delete(book)` | เอาออกจากกลุ่ม (destroy/nullify ตาม `dependent:`) |
| `author.books.destroy(book)` | เอาออกและ destroy เสมอ (ไม่ว่า `dependent:` จะตั้งเป็นอะไร) |
| `author.books.size` / `.count` / `.length` | นับจำนวน (ต่างกันเรื่อง query ตาม Step 267) |
| `author.books.empty?` | `true` ถ้าไม่มีเลยสักเล่ม |
| `author.books.exists?` | `true` ถ้ามีอย่างน้อยหนึ่งเล่ม (query แบบเบาที่สุด ไม่โหลดข้อมูลเต็ม) |
| `author.books.clear` | เอาทุกตัวออกจากกลุ่มพร้อมกัน (ตาม `dependent:`) |
| `author.books.reload` | บังคับ query ใหม่จากฐานข้อมูล ไม่ใช้ cache |

**สำหรับ `has_one :profile` (เขียนใน `Author`):**

| Method | ทำอะไร |
|---------|---------|
| `author.profile` | คืน `Profile` ที่ผูกอยู่ (หรือ `nil` ถ้ายังไม่มี) |
| `author.profile=(profile)` | ตั้งค่า profile ใหม่ (ตัวเก่าจะถูกจัดการตาม `dependent:`) |
| `author.build_profile(attrs)` | สร้างในหน่วยความจำ ยังไม่ save |
| `author.create_profile(attrs)` | สร้างและ save ทันที |
| `author.create_profile!(attrs)` | เหมือนข้างบนแต่ raise exception ถ้าไม่ผ่าน validation |
| `author.reload_profile` | บังคับโหลดใหม่จากฐานข้อมูล |

ทั้งหมดนี้ **เกิดจากการเขียนแค่ 3 บรรทัด** (`belongs_to :author`, `has_many :books`,
`has_one :profile`) — ยิ่งตอกย้ำหลักการ Convention over Configuration ที่เจอมาตลอดทั้งหลักสูตร:
ยิ่งทำตาม convention มากเท่าไหร่ ยิ่งได้ method ฟรีจาก Rails มากเท่านั้น

### ตัวอย่างเจาะลึก: ลองเรียกใช้ method ที่ไม่คุ้นตาสักสองสามตัว

```irb
irb(main):001> author = Author.create!(name: "ผู้แต่ง A")
irb(main):002> b1 = author.books.create!(title: "หนังสือ 1", pages: 10)
irb(main):003> b2 = author.books.create!(title: "หนังสือ 2", pages: 20)

irb(main):004> author.book_ids
=> [30, 31]

irb(main):005> author.books.exists?
=> true

irb(main):006> b3 = author.books.build(title: "หนังสือ 3", pages: 30)
irb(main):007> b3.new_record?
=> true

irb(main):008> author.books.delete(b2)
irb(main):009> author.reload.books.pluck(:title)
=> ["หนังสือ 1"]   # (สมมติ dependent: :destroy ทำให้ b2 ถูกลบไปด้วย)

irb(main):010> book_new = Book.new(title: "หนังสือ 4")
irb(main):011> built_author = book_new.build_author(name: "ผู้แต่งใหม่")
irb(main):012> book_new.author.equal?(built_author)
=> true
```

---

## แบบฝึกหัด: สร้างความสัมพันธ์ `Author has_many :books` ครบวงจร พร้อมแก้ N+1 จริง

### โจทย์

ต่อยอดจากแบบฝึกหัด `Book` model ใน Part 025 (Step 250) ให้ทำดังนี้:

1. สร้าง Model `Author` ใหม่ ที่มี `name:string` และ `nationality:string`
2. แก้ไขตาราง `books` ให้มี `author_id` แบบ reference ที่ถูกต้อง (มี foreign key constraint,
   `null: false`) แทนที่ column `author` (string) เดิม
3. เพิ่ม `belongs_to :author` ใน `Book` และ `has_many :books, dependent: :destroy` ใน
   `Author`
4. Seed ข้อมูล: สร้างผู้แต่ง 3 คน คนละ 2 เล่ม
5. เขียนสคริปต์ (รันด้วย `bin/rails runner`) ที่แสดงให้เห็น N+1 query problem จริงผ่าน SQL log
   แล้วแก้ด้วย `includes` พร้อมนับจำนวน query ที่ลดลง

### เฉลย

**1) สร้าง Model `Author`**

```bash
bin/rails generate model Author name:string nationality:string
```

```
      invoke  active_record
      create    db/migrate/20260926050001_create_authors.rb
      create    app/models/author.rb
      invoke    test_unit
      create      test/models/author_test.rb
      create      test/fixtures/authors.yml
```

**2) เพิ่ม reference เข้าตาราง `books` ที่มีอยู่แล้ว**

ถ้า `books` ยังไม่เคยมีข้อมูลสำคัญอยู่ (กรณีฝึกหัดในเครื่องตัวเอง) วิธีง่ายที่สุดคือลบ column
`author` แบบ string เดิมทิ้ง แล้วเพิ่ม `author` แบบ reference เข้าไปแทนในคำสั่งเดียว:

```bash
bin/rails generate migration ReplaceAuthorStringWithReferenceOnBooks
```

แก้ไฟล์ migration ที่ได้ให้เป็น:

```ruby
class ReplaceAuthorStringWithReferenceOnBooks < ActiveRecord::Migration[8.1]
  def change
    remove_column :books, :author, :string
    add_reference :books, :author, null: false, foreign_key: true
  end
end
```

> **หมายเหตุ:** `remove_column`/`add_column` ที่มีการลบข้อมูลจริงแบบนี้ ActiveRecord **อนุมาน
> ทิศทางย้อนกลับให้ไม่ได้อัตโนมัติ** (จำได้จาก Part 025 Step 243 ไหม) ถ้าต้องการให้ migration
> นี้ `rollback` ได้ ต้องระบุชนิดข้อมูลกำกับไว้ตอน `remove_column` แบบที่เขียนไว้ข้างบน
> (`:string`) มิเช่นนั้น Rails จะไม่รู้ว่าตอน rollback ต้องสร้าง column กลับมาเป็นชนิดอะไร

```bash
bin/rails db:migrate
```

```
== 20260926050002 ReplaceAuthorStringWithReferenceOnBooks: migrating =========
-- remove_column(:books, :author, :string)
   -> 0.0012s
-- add_reference(:books, :author, {:null=>false, :foreign_key=>true})
   -> 0.0028s
== 20260926050002 ReplaceAuthorStringWithReferenceOnBooks: migrated (0.0041s)
```

`db/schema.rb` หลัง migrate:

```ruby
create_table "books", force: :cascade do |t|
  t.string "title"
  t.string "author"      # <- คอลัมน์เก่าถูกลบไปแล้ว หายจาก schema
  t.integer "pages"
  t.boolean "read"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.integer "author_id", null: false
  t.index ["author_id"], name: "index_books_on_author_id"
end

add_foreign_key "books", "authors"
```

**3) ตั้งค่า association ทั้งสอง Model**

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end
```

```ruby
# app/models/book.rb
class Book < ApplicationRecord
  belongs_to :author
end
```

**4) Seed ข้อมูล**

```ruby
# db/seeds.rb
Book.destroy_all
Author.destroy_all

data = {
  "Haruki Murakami" => ["Norwegian Wood", "Kafka on the Shore"],
  "Neil Gaiman" => ["American Gods", "Coraline"],
  "Yuval Noah Harari" => ["Sapiens", "Homo Deus"]
}

data.each do |name, titles|
  author = Author.create!(name: name)
  titles.each do |title|
    author.books.create!(title: title, pages: rand(150..600))
  end
end

puts "สร้างผู้แต่ง #{Author.count} คน, หนังสือ #{Book.count} เล่ม"
```

```bash
bin/rails db:seed
```

```
สร้างผู้แต่ง 3 คน, หนังสือ 6 เล่ม
```

**5) สคริปต์แสดง N+1 แล้วแก้ด้วย `includes`**

```ruby
# n_plus_one_demo.rb (รันด้วย bin/rails runner n_plus_one_demo.rb)
require "logger"
ActiveRecord::Base.logger = Logger.new(STDOUT)

puts "==================== แบบมี N+1 (ไม่ใช้ includes) ===================="
Author.all.each do |author|
  puts "#{author.name}: #{author.books.map(&:title).join(', ')}"
end

puts "==================== แบบแก้ด้วย includes ===================="
Author.includes(:books).each do |author|
  puts "#{author.name}: #{author.books.map(&:title).join(', ')}"
end
```

```bash
bin/rails runner n_plus_one_demo.rb
```

ผลลัพธ์ (ตัด log ส่วน timestamp ออกเพื่อความอ่านง่าย):

```
==================== แบบมี N+1 (ไม่ใช้ includes) ====================
Author Load (0.1ms)  SELECT "authors".* FROM "authors"
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 1
Haruki Murakami: Norwegian Wood, Kafka on the Shore
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 2
Neil Gaiman: American Gods, Coraline
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" = 3
Yuval Noah Harari: Sapiens, Homo Deus
==================== แบบแก้ด้วย includes ====================
Author Load (0.1ms)  SELECT "authors".* FROM "authors"
Book Load (0.1ms)  SELECT "books".* FROM "books" WHERE "books"."author_id" IN (1, 2, 3)
Haruki Murakami: Norwegian Wood, Kafka on the Shore
Neil Gaiman: American Gods, Coraline
Yuval Noah Harari: Sapiens, Homo Deus
```

**สรุปผลลัพธ์:** แบบไม่ใช้ `includes` ใช้ **4 query** (1 + 3 ผู้แต่ง = N+1 ตามสูตร) ส่วนแบบใช้
`includes(:books)` เหลือแค่ **2 query คงที่** ไม่ว่าจะมีผู้แต่งกี่คนก็ตาม — ยิ่งข้อมูลเยอะขึ้น
ส่วนต่างของประสิทธิภาพยิ่งเห็นชัดขึ้นเรื่อยๆ นี่คือเหตุผลว่าทำไม `includes` (และเทคนิคที่เกี่ยวข้อง
ใน Part 034) ถึงเป็นสิ่งที่นักพัฒนา Rails ทุกคนต้องรู้จักและใช้เป็นนิสัยเมื่อ loop ผ่าน parent
record แล้วเข้าถึง association ของมัน

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม Model `Review` ใหม่ (`belongs_to :book`, `content:text`, `rating:integer`) แล้วลอง
   ตัดสินใจเองว่าควรใช้ `dependent:` แบบไหนใน `Book has_many :reviews` — เขียนเหตุผล
   ประกอบการตัดสินใจเป็นคอมเมนต์ไว้เหนือบรรทัด `has_many` นั้น จากนั้นทดสอบจริงว่าลบ `Book`
   แล้ว review ที่เกี่ยวข้องเป็นไปตามที่ตั้งใจหรือไม่
2. ลองสร้างสถานการณ์ N+1 จริงกับ 3 ชั้นความสัมพันธ์ (เช่น `Author -> Book -> Review`) แล้วเขียน
   loop ที่ดึง `author.books.first.reviews.count` สำหรับผู้แต่งทุกคน สังเกต SQL log ว่าเกิด
   กี่ query จากนั้นลองแก้ด้วย `includes(books: :reviews)` (nested includes) แล้วเทียบจำนวน
   query ก่อน-หลัง
3. ทดลองตั้ง `belongs_to :author, optional: true` ใน `Book` โดย **ไม่** แก้ migration ให้
   `null: true` ตาม แล้วสังเกต error ที่เกิดขึ้นตอน `save` — อธิบายด้วยคำพูดตัวเองว่าทำไม
   `valid?` ถึงคืน `true` ทั้งที่ `save` ล้มเหลว (ทบทวนจาก Step 263)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Foreign key** คือ column ที่เก็บค่า primary key ของอีกตารางไว้เพื่อ "ชี้" ไปยังแถวนั้น
  แทนการก๊อปปี้ข้อมูลซ้ำ — และการเชื่อมโยงด้วยมือ (`Author.find(book.author_id)`) มีปัญหาเรื่อง
  ความสะดวกและความปลอดภัยของข้อมูลมาก จนต้องมี association มาช่วยจัดการ
- **`belongs_to`** เขียนที่ฝั่งตารางที่มี foreign key column และเป็น **required by default
  ตั้งแต่ Rails 5** (ต้องมี record ที่อ้างอิงอยู่จริงเสมอ) ใช้ `optional: true` เมื่อต้องการยกเว้น
  แต่ต้องแก้ migration ให้ `null: true` ที่ระดับฐานข้อมูลด้วยเสมอ ไม่งั้นจะเจอ
  `NotNullViolation` แม้ `valid?` จะผ่าน
- **`t.references`/`add_reference`** สร้าง foreign key column พร้อม index และ (ถ้าระบุ)
  foreign key constraint ในคำสั่งเดียว และ generator จะเพิ่ม `belongs_to` ให้ใน Model
  อัตโนมัติเมื่อใช้ argument ชนิด `references`
- **`has_many`** เขียนที่ฝั่งตรงข้ามของ `belongs_to` (ใช้ชื่อพหูพจน์) ส่วน **`has_one`** ใช้เมื่อ
  ความสัมพันธ์เป็นแบบหนึ่งต่อหนึ่ง (ใช้ชื่อเอกพจน์ และควรมี unique index คู่กับ foreign key เสมอ)
- สร้าง record ผ่าน association โดยตรงได้สะดวกด้วย `build`, `create`, `create!`, `<<`,
  รวมถึงจัดการทั้งกลุ่มด้วย `ids=`/`= array` และ association มี **caching** ในตัวที่ต้องใช้
  `.reload` เพื่อบังคับ query ใหม่
- **`dependent:`** (`:destroy`, `:destroy_async`, `:nullify`, `:restrict_with_error`)
  ควบคุมชะตากรรมของ child record ตอน parent ถูกลบ — **ไม่เลือกเลยเป็นตัวเลือกที่อันตรายที่สุด**
  เพราะพฤติกรรมขึ้นกับว่ามี foreign key constraint ในฐานข้อมูลหรือไม่ อาจปล่อยให้เกิด orphaned
  record แบบเงียบๆ ได้
- **N+1 query problem** เกิดขึ้นเมื่อ loop ผ่าน parent แล้วเข้าถึง association ของแต่ละตัวแยกกัน
  ทำให้เกิด `1 + N` query แทนที่จะเป็นแค่ 2 query — แก้เบื้องต้นด้วย **`includes`** ซึ่งดึงข้อมูล
  ที่เกี่ยวข้องทั้งหมดมาล่วงหน้าในคำสั่งเดียว (รายละเอียดเชิงลึกเรื่อง query interface ทั้งหมดรอใน
  Part 034)
- **`inverse_of`** ทำให้สอง object ที่เป็นความสัมพันธ์เดียวกันชี้ไปยัง instance เดียวกันจริงๆ ใน
  หน่วยความจำ — Rails auto-detect ให้ได้ในกรณีมาตรฐานส่วนใหญ่ แต่ต้องระบุเองเมื่อชื่อ
  association ไม่ตรง convention หรือใช้กับ `:through`/polymorphic
- สรุปตาราง method ที่ Rails generate ให้อัตโนมัติครบทั้ง `belongs_to`, `has_many`, `has_one`
  — ยิ่งตอกย้ำหลักการ Convention over Configuration ว่าเขียนโค้ดแค่บรรทัดเดียวแต่ได้ method
  ใช้งานฟรีนับสิบตัว

**ต่อไป (Part 028):** ตอนนี้เราเขียน Model, Migration, และ Association ด้วยมือมาหลาย Part
แล้ว — Part 028 จะแนะนำ **Scaffold** เครื่องมือ generator ตัวใหญ่ที่สุดของ Rails ที่สร้าง Model,
Migration, Controller, View, และ Route ให้ครบในคำสั่งเดียว พร้อมพาไปอ่านโค้ดที่ scaffold
generate มาทีละไฟล์ ว่าแต่ละส่วนที่เรียนแยกกันมาตลอด Part 021–027 ถูกประกอบร่างเข้าด้วยกัน
เป็นแอปที่ใช้งานได้จริงอย่างไร
