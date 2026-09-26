# Part 026: Validation เบื้องต้น และ Callback เบื้องต้น (before_save, after_create)

> **Step ครอบคลุมใน Part นี้:** Step 251–260
> **ระดับ:** กลาง (ต้องผ่าน Part 025 มาก่อน โดยเฉพาะเรื่อง Model, Migration และ CRUD ผ่าน console)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

ท้าย Part 025 เราเจอเรื่องที่น่าตกใจเรื่องหนึ่ง: ลองสั่ง `post.update(title: nil)` ดูแล้วพบว่า
มัน**สำเร็จ** — คืนค่า `true` และ `title` กลายเป็น `nil` จริงๆ ในฐานข้อมูล ทั้งที่ตามธรรมชาติแล้ว
โพสต์ที่ไม่มีหัวข้อไม่ควรถูกบันทึกได้เลย นี่ไม่ใช่บั๊กของ Rails แต่เป็นพฤติกรรมที่ตั้งใจออกแบบไว้:
**ActiveRecord ไม่ตัดสินใจแทนเราว่าข้อมูลแบบไหน "ถูกต้อง"** หน้าที่นั้นเป็นของนักพัฒนาที่ต้อง
ประกาศกฎเอง — และเครื่องมือที่ใช้ประกาศกฎนั้นคือ **Validation**

Part นี้จะพาไปรู้จัก Validation แบบเต็มรูปแบบ ตั้งแต่ validator พื้นฐานที่ Rails มีให้ในตัว ไปจนถึง
การเขียน custom validation logic เอง จากนั้นจะพาไปรู้จัก **Callback** — กลไกที่ให้เราแทรกโค้ด
ของเราเข้าไปในจุดต่างๆ ตลอด "วงจรชีวิต" (lifecycle) ของ object ตั้งแต่เกิดจนตาย ทั้งสองเรื่องนี้
เป็นรากฐานสำคัญที่ใช้ตลอดทั้งหลักสูตรที่เหลือ ก่อนจะไปเรียนเรื่อง Association ใน Part 027

## สารบัญของ Part นี้

- Step 251: ปัญหาที่ Validation แก้ และ `validates :field, presence: true` ตัวแรก
- Step 252: `valid?`, `errors.full_messages`, และความแตกต่างของ `save`/`save!`/`create`/`create!`
- Step 253: Built-in validator ที่ใช้บ่อย — length, numericality, uniqueness, format, inclusion/exclusion
- Step 254: Custom validation method ด้วย `validate :method_name` และ `errors.add`
- Step 255: Conditional validation — `if:`/`unless:` แบบ symbol และ lambda
- Step 256: Validation context — `on: :create`/`on: :update` และ custom context
- Step 257: วงจรชีวิตของ ActiveRecord callback แบบเต็มรูปแบบ และลำดับการทำงานจริง
- Step 258: `before_validation`, `before_save`, `before_create` — use case ที่ใช้จริง
- Step 259: `after_create` vs `after_commit` — ทำไม transaction safety ถึงสำคัญ
- Step 260: ข้อควรระวัง callback overuse (callback hell / fat model) และทางออกระยะยาว

---

## Step 251: ปัญหาที่ Validation แก้ และ `validates :field, presence: true` ตัวแรก

### ย้อนดูปัญหาจาก Part 025 อีกครั้งให้ชัดเจน

เปิด `rails console` ในแอป `blog_app` ที่มี Model `Post` จาก Part 025 (ตอนนี้ `app/models/post.rb`
ยังว่างเปล่า ไม่มี validation ใดๆ):

```ruby
# app/models/post.rb (สภาพก่อนเริ่ม Part นี้)
class Post < ApplicationRecord
end
```

```irb
irb(main):001> post = Post.create(title: nil, body: "เนื้อหา", published: true)
irb(main):002> post.persisted?
=> true
irb(main):003> post.title
=> nil
irb(main):004> Post.count
=> 1
```

`Post.create(title: nil, ...)` สำเร็จอย่างสมบูรณ์ — ไม่มี error, ไม่มีคำเตือน, `persisted?` เป็น
`true` เหมือนสร้างสำเร็จปกติทุกอย่าง ปัญหานี้จะยิ่งชัดเมื่อมองจากมุมของแอปจริง: ถ้า `title` เป็น
`nil` แล้วมีหน้า view ที่เขียน `<h1><%= post.title.upcase %></h1>` แอปจะ crash ทันทีด้วย
`NoMethodError: undefined method 'upcase' for nil` ที่หน้าแสดงผลจริง (production) แทนที่จะถูก
ดักไว้ตั้งแต่ตอนบันทึกข้อมูล

**เหตุผลเชิงโครงสร้าง:** ฐานข้อมูล (schema) รู้แค่ "ชนิดข้อมูล" ของ column เช่น `title` เป็น
`VARCHAR` ที่เก็บ `NULL` ได้ (เพราะ migration ไม่ได้ระบุ `null: false`) แต่ฐานข้อมูลไม่รู้ "กฎทาง
ธุรกิจ" เช่น "โพสต์ทุกอันต้องมีหัวข้อ" — กฎแบบนี้ต้องประกาศไว้ที่ระดับ **Model** ด้วยตัวเราเอง

### `validates` — ประกาศกฎแรก

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
end
```

`validates` เป็น class method ที่ ActiveRecord เพิ่มให้ (มาจาก `ActiveModel::Validations` ซึ่ง
ActiveRecord include เข้ามาอีกที) รูปแบบพื้นฐานคือ:

```
validates :ชื่อ_attribute, ชื่อ_validator: ตัวเลือก
```

`presence: true` คือหนึ่งใน validator สำเร็จรูปที่ Rails มีให้ — ตรวจว่า attribute นั้น "มีค่า"
(ไม่ใช่ `nil` และไม่ใช่ string ว่างเปล่าหรือ whitespace ล้วน) ไม่ต้องรีสตาร์ท server หรือรัน
migration ใดๆ เพิ่ม เพราะ validation ทำงานอยู่ฝั่ง Ruby/ActiveRecord ไม่เกี่ยวกับโครงสร้าง
ฐานข้อมูลเลย

### `valid?` — ตรวจสอบโดยไม่ต้อง save

```irb
irb(main):001> reload!
Reloading...
irb(main):002> post = Post.new(title: nil, body: "เนื้อหา")
irb(main):003> post.valid?
=> false
```

`valid?` รัน validation ทั้งหมดที่ประกาศไว้ใน Model ทันที **โดยไม่แตะฐานข้อมูลเลย** แล้วคืนค่า
`true`/`false` ว่าผ่านหรือไม่ ระหว่างนั้น ActiveRecord จะเก็บรายละเอียดว่า field ไหนผิดพลาด
อย่างไรไว้ใน object พิเศษชื่อ `errors` (ชนิด `ActiveModel::Errors`)

```irb
irb(main):004> post.errors.full_messages
=> ["Title can't be blank"]
irb(main):005> post.errors[:title]
=> ["can't be blank"]
```

- `errors.full_messages` คืน Array ของข้อความ error แบบอ่านง่าย รวมชื่อ field ไว้ด้วย (เหมาะ
  แสดงในหน้า form ให้ผู้ใช้เห็น — จะใช้บ่อยมากใน Part 031 ตอนทำ error message บนฟอร์ม)
- `errors[:title]` (หรือเขียนเต็มว่า `errors.on(:title)` ในบาง gem แต่ Rails 8 ใช้ `[]` ตรงๆ)
  คืนข้อความ error เฉพาะ field นั้น **ไม่มีชื่อ field นำหน้า** เผื่อเอาไปประกอบข้อความเองแบบอื่น

> **หมายเหตุ (Rails 6.1+):** `errors[:title]` เป็น alias ของ `errors.messages[:title]` เดิม
> `errors[:title]` เคยคืน object ชนิดอื่นในเวอร์ชันเก่ามาก แต่ตั้งแต่ Rails 6.1 เป็นต้นมาพฤติกรรม
> เสถียรแล้วคือคืน Array ของ string ตามที่เห็นด้านบน

ลองใส่ `title` ที่มีค่าแล้วเช็คอีกครั้ง:

```irb
irb(main):006> post.title = "หัวข้อจริง"
irb(main):007> post.valid?
=> true
irb(main):008> post.errors.full_messages
=> []
```

สังเกตว่า `errors` จะถูกเคลียร์และคำนวณใหม่ทุกครั้งที่เรียก `valid?` — ไม่ใช่ error ที่สะสมค้างไว้
จากการเรียกครั้งก่อน

---

## Step 252: `save`/`save!`/`create`/`create!` เมื่อ validation เข้ามาเกี่ยวข้อง

### `save` และ `create` คืน `false` แทนที่จะ raise error

จำได้ไหมว่าใน Part 025 บอกไว้ว่า `save` คืน `true`/`false` — ตอนนั้นยังไม่มี validation เลย
`save` เลยคืน `true` เสมอ ตอนนี้เมื่อมี validation แล้ว พฤติกรรมจะชัดเจนขึ้นมาก:

```irb
irb(main):001> post = Post.new(title: nil)
irb(main):002> post.save
=> false
irb(main):003> post.persisted?
=> false
irb(main):004> Post.count
=> 0
```

`save` จะ**รัน validation ก่อนเสมอ** (เทียบเท่ากับเรียก `valid?` ให้อัตโนมัติ) ถ้าไม่ผ่านจะไม่ยิง
SQL `INSERT`/`UPDATE` ไปที่ฐานข้อมูลเลย แล้วคืน `false` กลับมาเงียบๆ — โค้ดที่เรียก `save` ต้อง
เช็ค return value เองว่าสำเร็จหรือไม่ (จะเห็นรูปแบบนี้เขียนในทุก controller action ที่มี form
ตั้งแต่ Part 031 เป็นต้นไป)

```irb
irb(main):005> Post.create(title: nil)
```
```
=>
#<Post id: nil, title: nil, body: nil, published: nil, created_at: nil, updated_at: nil>
```

`Post.create` ก็เช่นกัน — **คืน object กลับมาเสมอ ไม่ว่าจะสำเร็จหรือไม่** สังเกตว่า `id: nil`
แปลว่าไม่ถูกบันทึกจริง ต้องเช็คด้วย `persisted?` หรือ `errors.any?` เพื่อรู้ว่า `create` สำเร็จจริง
หรือแค่คืน object เปล่าๆ ที่ validate ไม่ผ่านกลับมา:

```irb
irb(main):006> post = Post.create(title: nil)
irb(main):007> post.persisted?
=> false
irb(main):008> post.errors.any?
=> true
irb(main):009> post.errors.full_messages
=> ["Title can't be blank"]
```

### `save!`/`create!` — raise exception แทนที่จะคืน false เงียบๆ

บางสถานการณ์เราไม่อยากให้โปรแกรมเดินต่อแบบเงียบๆ ถ้า validation ไม่ผ่าน (เช่น สคริปต์
import ข้อมูลที่ควรหยุดทันทีถ้าเจอข้อมูลเสีย) ใช้ version ที่มี `!` ต่อท้าย:

```irb
irb(main):010> Post.create!(title: nil)
```
```
ActiveRecord::RecordInvalid (Validation failed: Title can't be blank)
```

```irb
irb(main):011> post = Post.new(title: nil)
irb(main):012> post.save!
```
```
ActiveRecord::RecordInvalid (Validation failed: Title can't be blank)
```

`save!`/`create!` ทำงานเหมือน `save`/`create` ทุกอย่าง เพียงแต่ถ้า validation ไม่ผ่านจะ
**raise `ActiveRecord::RecordInvalid`** แทนที่จะคืน `false`/object เปล่า ข้อความ exception จะ
รวม `errors.full_messages` ไว้ให้อัตโนมัติ ดักจับได้ด้วย `begin/rescue` แบบที่เรียนมาแล้วใน
Part 011:

```ruby
begin
  Post.create!(title: nil)
rescue ActiveRecord::RecordInvalid => e
  puts "สร้างโพสต์ไม่สำเร็จ: #{e.message}"
  puts e.record.errors.full_messages
end
```

**ตารางสรุปเลือกใช้แบบไหนเมื่อไหร่:**

| Method | คืนค่าเมื่อสำเร็จ | คืนค่า/พฤติกรรมเมื่อ validation ไม่ผ่าน |
|---------|---------------------|--------------------------------------------|
| `save` | `true` | คืน `false` |
| `save!` | `true` | raise `ActiveRecord::RecordInvalid` |
| `create` | object ที่ persisted แล้ว | คืน object ที่ `persisted? == false` พร้อม `errors` |
| `create!` | object ที่ persisted แล้ว | raise `ActiveRecord::RecordInvalid` |
| `update` | `true` | คืน `false` (ไม่ raise) |
| `update!` | `true` | raise `ActiveRecord::RecordInvalid` |

**แนวทางปฏิบัติจริง:** ใน controller ของแอปเว็บทั่วไป (Part 031) นิยมใช้ `save`/`create`/`update`
แบบไม่มี `!` แล้วเช็ค return value เอง เพื่อ render ฟอร์มพร้อม error message กลับไปให้ผู้ใช้แก้ไข
ส่วน `!` version เหมาะกับสคริปต์เบื้องหลัง (background job, rake task, data seed) ที่ถ้าข้อมูลผิด
ควรทำให้ระบบ error ทันทีเพื่อดักให้เห็นชัดเจน ไม่ปล่อยผ่านแบบเงียบๆ

---

## Step 253: Built-in Validator ที่ใช้บ่อย

Rails มี validator สำเร็จรูปให้มากกว่า 10 ตัว แต่ที่ใช้บ่อยที่สุดในงานจริงมีประมาณ 6 ตัว มา
เพิ่ม column ใหม่ในตาราง `posts` เพื่อสาธิตให้ครบทุกแบบ:

```bash
bin/rails generate migration AddValidationDemoFieldsToPosts slug:string status:string priority:integer
```

```ruby
# db/migrate/20260926030421_add_validation_demo_fields_to_posts.rb
class AddValidationDemoFieldsToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :slug, :string
    add_column :posts, :status, :string
    add_column :posts, :priority, :integer
  end
end
```

```bash
bin/rails db:migrate
```

### `length` — ควบคุมความยาว string

```ruby
validates :title, presence: true, length: { maximum: 100 }
validates :title, length: { minimum: 5 }, on: :create   # เดี๋ยวอธิบาย on: ใน Step 256
validates :body, length: { minimum: 10 }, allow_nil: true
```

```irb
irb(main):001> Post.new(title: "สั้น").tap(&:valid?).errors[:title]
=> ["is too short (minimum is 5 characters)"]
irb(main):002> Post.new(title: "ก" * 101).tap(&:valid?).errors[:title]
=> ["is too long (maximum is 100 characters)"]
irb(main):003> Post.new(title: "หัวข้อปกติ", body: "สั้นไป").tap(&:valid?).errors[:body]
=> ["is too short (minimum is 10 characters)"]
```

ตัวเลือกของ `length` ที่ใช้บ่อย: `minimum`, `maximum`, `in: 5..100` (กำหนดช่วงในคำสั่งเดียว),
`is: 13` (ต้องยาวเท่านี้พอดี เช่น ISBN-13), และ `allow_nil: true`/`allow_blank: true` เพื่อบอก
ว่าถ้า attribute ยังไม่มีค่า ให้ข้ามการเช็คความยาวไปก่อน (เพราะถ้าไม่ใส่ไว้ `nil` จะถูกมองว่า
"ความยาว 0" แล้วอาจ error ซ้ำกับ `presence` แบบไม่จำเป็น)

### `numericality` — ตรวจว่าเป็นตัวเลขตามเงื่อนไข

```ruby
validates :priority, numericality: { only_integer: true, greater_than: 0, less_than_or_equal_to: 5 },
                      allow_nil: true
```

```irb
irb(main):004> Post.new(title: "หัวข้อปกติ", priority: 0).tap(&:valid?).errors[:priority]
=> ["must be greater than 0"]
irb(main):005> Post.new(title: "หัวข้อปกติ", priority: 1.5).tap(&:valid?).errors[:priority]
=> ["must be an integer"]
irb(main):006> Post.new(title: "หัวข้อปกติ", priority: 10).tap(&:valid?).errors[:priority]
=> ["must be less than or equal to 5"]
irb(main):007> Post.new(title: "หัวข้อปกติ", priority: 3).tap(&:valid?).errors[:priority]
=> []
```

ตัวเลือกของ `numericality` ที่ใช้บ่อย: `only_integer`, `greater_than`, `greater_than_or_equal_to`,
`less_than`, `less_than_or_equal_to`, `equal_to`, `odd`/`even` — ผสมกันได้หลายตัวในคำสั่งเดียว

### `uniqueness` — ห้ามค่าซ้ำในตาราง

```ruby
validates :slug, uniqueness: true, allow_blank: true
```

```irb
irb(main):008> Post.create!(title: "โพสต์แรกสุด", slug: "first-post", status: "published")
irb(main):009> Post.new(title: "โพสต์ซ้ำสอง", slug: "first-post", status: "draft").tap(&:valid?).errors[:slug]
=> ["has already been taken"]
```

> **ข้อควรระวังสำคัญ:** `uniqueness` validator ทำงานด้วยการยิง query `SELECT` ไปเช็คก่อนว่ามี
> ค่าซ้ำอยู่แล้วหรือไม่ (ไม่ใช่การพึ่ง database constraint) ซึ่งมีช่องโหว่เล็กน้อยจาก **race
> condition**: ถ้ามี 2 request เข้ามาพร้อมกันพอดี ทั้งคู่อาจเช็คผ่านพร้อมกันก่อนที่ทั้งคู่จะ
> `INSERT` ทำให้ค่าซ้ำหลุดเข้าฐานข้อมูลได้ในทางทฤษฎี วิธีป้องกันที่แน่นอน 100% คือเพิ่ม
> **unique index** ที่ระดับฐานข้อมูลคู่กันไปด้วยเสมอ (`add_index :posts, :slug, unique: true`)
> — รายละเอียดเรื่องนี้จะสอนเชิงลึกใน Part 032

### `format` — ตรวจรูปแบบด้วย Regular Expression

```ruby
validates :slug, format: { with: /\A[a-z0-9\-]+\z/,
                            message: "ใช้ได้เฉพาะตัวพิมพ์เล็ก ตัวเลข และขีดกลาง (-) เท่านั้น" },
                  allow_blank: true
```

```irb
irb(main):010> Post.new(title: "หัวข้อปกติ", slug: "Invalid Slug!", status: "draft").tap(&:valid?).errors[:slug]
=> ["ใช้ได้เฉพาะตัวพิมพ์เล็ก ตัวเลข และขีดกลาง (-) เท่านั้น"]
```

`with:` รับ Regular Expression (ทบทวนเรื่อง regex ได้ที่ Part 003) — ต้อง match ทั้ง string ถึงจะ
ผ่าน ตัว `\A` และ `\z` สำคัญมาก (anchor จุดเริ่มต้น/จุดสิ้นสุดจริงของ string ต่างจาก `^`/`$` ที่
จับแค่ต้น/ท้ายบรรทัด ซึ่งเป็นช่องโหว่ security ที่พบบ่อยถ้าใช้ `^`/`$` ผิดที่) มี `without:` เป็น
ตัวเลือกตรงข้าม (ต้อง**ไม่** match ถึงจะผ่าน) ด้วย

### `inclusion` / `exclusion` — จำกัดค่าให้อยู่ใน (หรือไม่อยู่ใน) รายการที่กำหนด

```ruby
STATUSES = %w[draft published archived].freeze
RESERVED_SLUGS = %w[admin new edit].freeze

validates :status, inclusion: { in: STATUSES, message: "%{value} ไม่ใช่สถานะที่รองรับ" }
validates :slug, exclusion: { in: RESERVED_SLUGS, message: "%{value} เป็นคำสงวน ใช้เป็น slug ไม่ได้" },
                  allow_blank: true
```

```irb
irb(main):011> Post.new(title: "หัวข้อปกติ", slug: "some-post", status: "unknown").tap(&:valid?).errors[:status]
=> ["unknown ไม่ใช่สถานะที่รองรับ"]
irb(main):012> Post.new(title: "หัวข้อปกติ", slug: "admin", status: "draft").tap(&:valid?).errors[:slug]
=> ["admin เป็นคำสงวน ใช้เป็น slug ไม่ได้"]
```

`%{value}` ใน `message:` เป็น placeholder พิเศษที่ ActiveRecord แทนที่ด้วยค่าจริงที่ผู้ใช้ส่งเข้า
มาให้อัตโนมัติ (ใช้กับ validator ไหนก็ได้ ไม่ใช่แค่ `inclusion`/`exclusion`) ทำให้ error message
เจาะจงและอ่านเข้าใจง่ายกว่าข้อความทั่วไป

### ดู error หลาย field พร้อมกันในคำสั่งเดียว

```irb
irb(main):013> bad = Post.new(title: nil, slug: "admin", status: "unknown", priority: -1)
irb(main):014> bad.valid?
=> false
irb(main):015> bad.errors.full_messages
=>
["Title can't be blank",
 "Title is too short (minimum is 5 characters)",
 "Slug admin เป็นคำสงวน ใช้เป็น slug ไม่ได้",
 "Status unknown ไม่ใช่สถานะที่รองรับ",
 "Priority must be greater than 0"]
```

สังเกตว่า `valid?` เช็ค**ทุก validation ทุกตัวจนครบ** ไม่ได้หยุดตั้งแต่ตัวแรกที่ error — ทำให้
`errors.full_messages` แสดงปัญหาทั้งหมดในครั้งเดียว ผู้ใช้จึงแก้ไขได้ครบในรอบเดียวโดยไม่ต้อง
ลองส่งฟอร์มใหม่ทีละจุด

---

## Step 254: Custom Validation Method ด้วย `validate` และ `errors.add`

Built-in validator ครอบคลุมกรณีทั่วไปได้มาก แต่บางกฎทางธุรกิจซับซ้อนเกินกว่าจะเขียนด้วย
`validates` บรรทัดเดียว เช่น "ต้องมี slug ก่อนเผยแพร่โพสต์เท่านั้น (ไม่บังคับตอนเป็น draft)"
กรณีนี้ต้องเขียนเป็น **custom validation method** โดยใช้ `validate` (ไม่มี `s` ต่อท้าย — สังเกต
ความต่างจาก `validates` ที่มี `s` ให้ดีๆ)

```ruby
class Post < ApplicationRecord
  # ...

  validate :slug_required_when_published

  private

  def slug_required_when_published
    return unless status == "published"

    errors.add(:slug, "ต้องระบุก่อนเผยแพร่โพสต์ (published)") if slug.blank?
  end
end
```

`validate :method_name` บอก ActiveRecord ให้เรียก instance method นั้นเป็นส่วนหนึ่งของขั้นตอน
validation ภายใน method เราต้องเช็คเงื่อนไขเองแล้วเรียก `errors.add(:field, "ข้อความ")` เองด้วย
มือ ถ้าเงื่อนไขผ่าน (ไม่เรียก `errors.add`) ก็ถือว่า field นั้นไม่มี error จาก method นี้

```irb
irb(main):001> post = Post.new(title: "โพสต์ที่จะเผยแพร่", status: "published", slug: nil)
irb(main):002> post.valid?
=> false
irb(main):003> post.errors[:slug]
=> ["ต้องระบุก่อนเผยแพร่โพสต์ (published)"]
irb(main):004> post.slug = "post-that-will-publish"
irb(main):005> post.valid?
=> true
```

### `errors.add(:base, ...)` — error ที่ไม่ผูกกับ field ใดๆ

บางกฎเกี่ยวข้องกับหลาย field พร้อมกัน ไม่ควรผูกไว้กับ field ใด field หนึ่งเป็นการเฉพาะ — ใช้
`:base` แทนชื่อ field:

```ruby
validate :title_must_differ_from_slug

private

def title_must_differ_from_slug
  return if title.blank? || slug.blank?

  errors.add(:base, "Title และ Slug ห้ามเป็นข้อความเดียวกัน") if title == slug
end
```

```irb
irb(main):006> dup_post = Post.new(title: "same-text", slug: "same-text", status: "draft")
irb(main):007> dup_post.valid?
=> false
irb(main):008> dup_post.errors[:base]
=> ["Title และ Slug ห้ามเป็นข้อความเดียวกัน"]
irb(main):009> dup_post.errors.full_messages
=> ["Title และ Slug ห้ามเป็นข้อความเดียวกัน"]
```

สังเกตว่า `errors.full_messages` ของ `:base` **ไม่มีชื่อ field นำหน้า** (ต่างจาก field อื่นที่จะ
มีคำว่า "Title", "Slug" นำหน้าเสมอ) เพราะ error แบบนี้พูดถึง record ทั้งก้อน ไม่ใช่ field เดียว —
เหมาะกับกฎแบบ "รวม field เข้าด้วยกันแล้วขัดกัน" เช่น "วันที่เริ่มต้นต้องมาก่อนวันที่สิ้นสุด"

> **เมื่อไหร่ควรใช้ `validate` (custom method) แทน `validates` (built-in)?** ใช้ built-in
> validator ก่อนเสมอถ้าตรงกับสิ่งที่ต้องการ (อ่านง่ายกว่า, เทสแล้วโดยทีม Rails) เขียน custom
> validation method เฉพาะตอนที่กฎซับซ้อนกว่าการเช็ค field เดียวตรงๆ เช่น ต้องเทียบค่าระหว่าง
> หลาย field, ต้องเรียก method อื่นประกอบการตัดสินใจ, หรือเงื่อนไขเปลี่ยนไปตามค่าของ field อื่น
> อย่างตัวอย่างข้างบน

---

## Step 255: Conditional Validation — `if:`/`unless:`

บางครั้งกฎ validation ควรบังคับใช้เฉพาะบางสถานการณ์เท่านั้น ไม่ใช่ทุกครั้ง — ทั้ง `validates`
และ `validate` รับตัวเลือก `if:`/`unless:` ได้เหมือนกัน โดยรับได้ 2 รูปแบบหลัก: **symbol**
(ชื่อ method ที่คืน `true`/`false`) หรือ **lambda** (โค้ดสั้นๆ ที่ประเมินผลตรงจุด)

### แบบ symbol — เรียก instance method ที่มีอยู่แล้ว

```ruby
validates :priority, presence: true, if: :published?
```

`published?` คือ boolean query method ที่ ActiveRecord สร้างให้อัตโนมัติจาก column `published`
(ทบทวนได้จาก Part 025 Step 250) — บรรทัดนี้แปลว่า "priority ต้องมีค่า **เฉพาะเมื่อ** published
เป็น true เท่านั้น":

```irb
irb(main):001> draft = Post.new(title: "หัวข้อฉบับร่าง", published: false, priority: nil, status: "draft")
irb(main):002> draft.valid?
=> true
irb(main):003> draft.errors[:priority]
=> []

irb(main):004> live = Post.new(title: "หัวข้อจริง", published: true, priority: nil, status: "published",
                                 slug: "live-post", body: "เนื้อหาที่ยาวพอสำหรับผ่าน validation แน่นอน")
irb(main):005> live.valid?
=> false
irb(main):006> live.errors[:priority]
=> ["can't be blank"]
```

### แบบ lambda — เหมาะกับเงื่อนไขสั้นๆ ที่ไม่คุ้มค่าจะแยกเป็น method

```ruby
validates :body, length: { minimum: 20 }, if: -> { status == "published" }
```

Lambda จะถูกรันในบริบทของ instance นั้น (สามารถอ่าน attribute ของ record ได้ตรงๆ เหมือนอยู่
ใน instance method) เหมาะกับเงื่อนไขที่ใช้ครั้งเดียว ไม่จำเป็นต้องมีชื่อ method แยกต่างหาก

```irb
irb(main):007> short_live = Post.new(title: "หัวข้อจริง", published: true, priority: 1, status: "published",
                                       slug: "short-live", body: "สั้นไป")
irb(main):008> short_live.valid?
=> false
irb(main):009> short_live.errors[:body]
=> ["is too short (minimum is 10 characters)", "is too short (minimum is 20 characters)"]
```

สังเกตว่ามี error 2 ข้อความพร้อมกัน — เพราะ `body` มีทั้ง `validates :body, length: { minimum: 10 }`
(บังคับใช้เสมอ) และ validation แบบมีเงื่อนไข `minimum: 20` (บังคับใช้เฉพาะตอน published) การ
ประกาศ `validates` field เดียวกันหลายบรรทัดแบบนี้เป็นเรื่องปกติมาก — แต่ละบรรทัดคือกฎคนละข้อ
ที่ตรวจแยกอิสระจากกัน ไม่ได้เขียนทับกัน

### ใช้ `if:` และ `unless:` พร้อมกันได้

```ruby
validates :body, length: { minimum: 20 },
                  if: -> { status == "published" },
                  unless: :skip_body_check?
```

`unless:` คือด้านตรงข้ามของ `if:` — validation จะทำงาน**เฉพาะเมื่อเงื่อนไขนั้นเป็น false**
ถ้าใส่ทั้งคู่พร้อมกัน validation จะทำงานก็ต่อเมื่อ `if:` เป็น true **และ** `unless:` เป็น false
เท่านั้น:

```irb
irb(main):010> bypassed = Post.new(title: "หัวข้อจริง", published: true, priority: 1, status: "published",
                                     slug: "bypassed-post", body: "สั้นไป", skip_body_check: true)
irb(main):011> bypassed.valid?
=> false
irb(main):012> bypassed.errors[:body]
=> ["is too short (minimum is 10 characters)"]
```

สังเกตว่า error เหลือแค่ข้อความเดียว (จาก validation ที่บังคับใช้เสมอ) เพราะ `unless:
:skip_body_check?` ทำให้ validation เงื่อนไข `minimum: 20` ถูกข้ามไป — เป็นรูปแบบที่มีประโยชน์
มากเวลาต้องการ "ประตูหลัง" ให้ระบบภายใน (เช่น admin, migration script) ข้ามการเช็คบางอย่างได้
โดยไม่ต้องปิด validation ทั้งหมด

> **ตัวเลือกเพิ่มเติม:** `if:`/`unless:` รับเป็น Array ของ symbol/lambda ได้ด้วย (ต้องผ่าน
> **ทุกตัว** ถึงจะนับว่าเงื่อนไขเป็นจริง) เช่น `if: [:published?, :has_author?]` — มีประโยชน์
> เมื่อต้องรวมหลายเงื่อนไขเข้าด้วยกันแบบ AND โดยไม่ต้องเขียน lambda ยาวๆ รวมกันเอง

---

## Step 256: Validation Context — `on: :create`/`on: :update` และ Context กำหนดเอง

ตามค่า default แล้ว validation ทุกตัวทำงาน**ทุกครั้ง**ที่ `valid?`/`save` ถูกเรียก ไม่ว่าจะเป็น
การสร้างใหม่หรือแก้ไข แต่บางกฎควรบังคับใช้เฉพาะตอนสร้างใหม่ หรือเฉพาะตอนแก้ไขเท่านั้น — ตัวเลือก
`on:` แก้ปัญหานี้

### `on: :create` และ `on: :update`

```ruby
validates :title, presence: true, length: { maximum: 100 }   # บังคับใช้เสมอ (ไม่มี on:)
validates :title, length: { minimum: 5 }, on: :create          # เฉพาะตอนสร้างใหม่
validates :title, length: { minimum: 8 }, on: :update           # เฉพาะตอนแก้ไข (เข้มกว่า)
```

ActiveRecord รู้เองว่าตอนนี้ควรใช้ context ไหน โดยดูจาก `new_record?`: ถ้า record ยังไม่เคย
persisted จะใช้ context `:create` โดยอัตโนมัติ ถ้า persisted แล้วจะใช้ `:update`

```irb
irb(main):001> new_post = Post.new(title: "สั้น", slug: "short-title-post", status: "draft")
irb(main):002> new_post.valid?        # persisted? == false -> ใช้ context :create
=> false
irb(main):003> new_post.errors[:title]
=> ["is too short (minimum is 5 characters)"]

irb(main):004> new_post.title = "หัวข้อพอดี"
irb(main):005> new_post.save
=> true

irb(main):006> new_post.title = "แก้ไข"       # 5 ตัวอักษร ผ่าน :create แต่ไม่ผ่าน :update
irb(main):007> new_post.valid?                # persisted? == true แล้ว -> ใช้ context :update
=> false
irb(main):008> new_post.errors[:title]
=> ["is too short (minimum is 8 characters)"]
```

สังเกตว่า `title` "แก้ไข" (5 ตัวอักษร) เคยผ่าน `on: :create` (ขั้นต่ำ 5) มาแล้วตอนสร้างครั้งแรก
แต่พอเปลี่ยนมาแก้ไข record เดิม context เปลี่ยนเป็น `:update` ทำให้กฎขั้นต่ำ 8 ตัวอักษรมามีผลแทน
— นี่คือประโยชน์ของ `on:`: ใช้กฎที่ต่างกันได้ตามช่วงชีวิตของ record โดยไม่ต้องเขียน `if:`
เช็ค `persisted?` เอง

### Context ที่กำหนดเอง (custom context)

นอกจาก `:create`/`:update` ที่ ActiveRecord ให้มาโดยอัตโนมัติ เรากำหนด context ชื่ออะไรก็ได้
เองแล้วเรียกใช้ตอนต้องการ:

```ruby
validates :priority, presence: true, on: :publishing
```

Context แบบนี้จะ**ไม่ทำงานอัตโนมัติ**จาก `valid?`/`save` ธรรมดา ต้องเรียกระบุ context ชัดเจน:

```irb
irb(main):009> draft = Post.create!(title: "ฉบับร่างที่ยังไม่พร้อม", slug: "not-ready-yet",
                                      status: "draft", priority: nil)
irb(main):010> draft.valid?                 # context ปกติ (update) -> priority ไม่บังคับตรงนี้
=> true
irb(main):011> draft.valid?(:publishing)    # เช็คด้วย custom context -> priority ต้องมีค่า
=> false
irb(main):012> draft.errors[:priority]
=> ["can't be blank"]

irb(main):013> draft.priority = 3
irb(main):014> draft.save(context: :publishing)
=> true
```

`valid?(:context_name)` และ `save(context: :context_name)` คือสองวิธีเรียกใช้ custom context
— รูปแบบนี้มีประโยชน์มากเวลามีขั้นตอนหลายขั้นในระบบ เช่น "บันทึกฉบับร่างได้อย่างอิสระ แต่ก่อน
กดปุ่ม 'เผยแพร่จริง' ต้องผ่านการเช็คที่เข้มงวดกว่า" โดยไม่ต้องแยก Model หรือเขียน service
object ตั้งแต่ตอนนี้ (การออกแบบด้วย Form Object ที่ทำเรื่องนี้ได้ยืดหยุ่นกว่าจะสอนใน Part 082)

---

## Step 257: วงจรชีวิตของ ActiveRecord Callback แบบเต็มรูปแบบ

Validation ตอบคำถามว่า "ข้อมูลนี้ถูกต้องไหม" ส่วน **Callback** ตอบคำถามว่า "ควรทำอะไรเพิ่มเติม
ณ จุดไหนของกระบวนการบันทึก/ลบข้อมูล" — Callback คือ method ที่เราลงทะเบียนไว้ล่วงหน้าให้
ActiveRecord เรียกอัตโนมัติ ณ จุดต่างๆ ตลอด "วงจรชีวิต" ของ object

### ทดลองดูลำดับการทำงานจริง

สร้าง Model ทดลองที่มี callback ครบทุกจุด แล้วบันทึกลำดับการทำงานไว้ดู:

```bash
bin/rails generate model CallbackDemo name:string
bin/rails db:migrate
```

```ruby
# app/models/callback_demo.rb
class CallbackDemo < ApplicationRecord
  attr_accessor :log

  before_validation { (self.log ||= []) << "before_validation" }
  after_validation  { (self.log ||= []) << "after_validation" }

  before_save       { (self.log ||= []) << "before_save" }
  before_create     { (self.log ||= []) << "before_create" }
  before_update     { (self.log ||= []) << "before_update" }

  after_create      { (self.log ||= []) << "after_create" }
  after_update      { (self.log ||= []) << "after_update" }
  after_save        { (self.log ||= []) << "after_save" }

  before_destroy    { (self.log ||= []) << "before_destroy" }
  after_destroy     { (self.log ||= []) << "after_destroy" }

  after_commit      { (self.log ||= []) << "after_commit" }
  after_rollback    { (self.log ||= []) << "after_rollback" }
end
```

```irb
irb(main):001> demo = CallbackDemo.new(name: "แรก")
irb(main):002> demo.save
irb(main):003> demo.log
=>
["before_validation", "after_validation", "before_save", "before_create",
 "after_create", "after_save", "after_commit"]

irb(main):004> demo.log = []
irb(main):005> demo.update(name: "แก้ไขแล้ว")
irb(main):006> demo.log
=>
["before_validation", "after_validation", "before_save", "before_update",
 "after_update", "after_save", "after_commit"]

irb(main):007> demo.log = []
irb(main):008> demo.destroy
irb(main):009> demo.log
=> ["before_destroy", "after_destroy", "after_commit"]
```

### ตารางสรุปลำดับ callback ทางการ (ตรงกับผลทดสอบจริงด้านบน)

**ตอน `create` (record ใหม่):**

| ลำดับ | Callback | หมายเหตุ |
|-------|-----------|-----------|
| 1 | `before_validation` | ก่อนเริ่มตรวจ validation |
| 2 | *(validation ทำงาน)* | |
| 3 | `after_validation` | หลังตรวจ validation เสร็จ (ไม่ว่าจะผ่านหรือไม่) |
| 4 | `before_save` | ก่อนบันทึก (ทำงานทั้งตอน create และ update) |
| 5 | `before_create` | ก่อนบันทึก เฉพาะตอน create เท่านั้น |
| 6 | *(INSERT SQL)* | |
| 7 | `after_create` | หลังบันทึกเสร็จ เฉพาะตอน create |
| 8 | `after_save` | หลังบันทึกเสร็จ ทำงานทั้ง create และ update |
| 9 | `after_commit` (หรือ `after_rollback` ถ้าธุรกรรมล้มเหลว) | หลัง transaction ยืนยันจริง |

**ตอน `update`:** เหมือนกันทุกจุด เพียงแค่สลับ `before_create`/`after_create` เป็น
`before_update`/`after_update`

**ตอน `destroy`:** `before_destroy` → *(DELETE SQL)* → `after_destroy` →
`after_commit`/`after_rollback`

> **สังเกตสำคัญ:** `before_validation`/`after_validation` ทำงานทั้งตอน create และ update
> เหมือนกัน (validation ไม่สนใจว่าเป็น record ใหม่หรือเก่า) ส่วน `before_save`/`after_save`
> ทำงาน "ครอบ" ทั้งสองกรณี ในขณะที่ `before_create`/`after_create` และ
> `before_update`/`after_update` เจาะจงเฉพาะกรณีของตัวเอง — ความแตกต่างนี้สำคัญมากตอนเลือกใช้
> callback ให้ตรงจุดใน Step 258

### `after_rollback` — เมื่อ transaction ถูกยกเลิก

```irb
irb(main):010> demo2 = CallbackDemo.new(name: "สอง")
irb(main):011> CallbackDemo.transaction do
irb(main):012>   demo2.save
irb(main):013>   raise ActiveRecord::Rollback
irb(main):014> end
irb(main):015> demo2.log
=>
["before_validation", "after_validation", "before_save", "before_create",
 "after_create", "after_save", "after_rollback"]
irb(main):016> demo2.persisted?
=> false
```

สังเกตว่าทุกอย่างทำงานเหมือนตอน create สำเร็จปกติ **ยกเว้นบรรทัดสุดท้าย** — แทนที่จะเป็น
`after_commit` กลายเป็น `after_rollback` เพราะ `raise ActiveRecord::Rollback` ภายใน block
`transaction` สั่งยกเลิกธุรกรรมทั้งหมด ทำให้ข้อมูลไม่ถูกบันทึกจริง (`persisted? == false`) —
รายละเอียดเรื่องนี้จะสำคัญมากใน Step 259

---

## Step 258: `before_validation`, `before_save`, `before_create` — ใช้ตอนไหนดี

ตอนนี้เห็นลำดับ callback ทั้งหมดแล้ว มาดูว่าในทางปฏิบัติควรเลือกใช้ callback ตัวไหนสำหรับงาน
แบบไหน โดยใช้ Model `Post` เป็นตัวอย่าง

### `before_validation` — ปรับข้อมูลก่อนที่ validation จะตรวจ

ใช้เมื่อต้องการ "ทำความสะอาด" หรือ normalize ข้อมูลก่อนที่ validation จะทำงาน เพื่อให้ validation
ตรวจสอบค่าที่ถูกต้องแล้ว ไม่ใช่ค่าดิบจากผู้ใช้:

```ruby
before_validation :normalize_title
before_validation :downcase_slug

private

def normalize_title
  self.title = title.strip if title.present?
end

def downcase_slug
  self.slug = slug.downcase if slug.present?
end
```

```irb
irb(main):001> post = Post.new(title: "   หัวข้อมีช่องว่างเกิน   ", slug: "MySlug")
irb(main):002> post.valid?
irb(main):003> post.title
=> "หัวข้อมีช่องว่างเกิน"
irb(main):004> post.slug
=> "myslug"
irb(main):005> post.errors[:slug]
=> []
```

จุดสำคัญคือ **ต้องเป็น `before_validation` ไม่ใช่ `before_save`** เพราะถ้า normalize หลัง
validation ไปแล้ว (เช่นใช้ `before_save`) format validator (`/\A[a-z0-9\-]+\z/`) จะเจอค่า
`"MySlug"` (มีตัวพิมพ์ใหญ่) ก่อนที่จะถูก downcase แล้วปฏิเสธค่านั้นไปทั้งที่จริงๆ แล้วถูกต้อง
หลัง normalize — **ลำดับของ callback มีผลต่อผลลัพธ์ของ validation โดยตรง** เป็นเหตุผลว่าทำไม
ต้องเข้าใจตารางลำดับใน Step 257 ให้แม่น

### `before_save` — ทำงานก่อนบันทึกทุกครั้ง (ทั้ง create และ update)

ใช้เมื่อต้องการให้ logic ทำงาน **ทุกครั้งที่มีการบันทึก** ไม่ว่าจะเป็นการสร้างใหม่หรือแก้ไข เช่น
คำนวณค่าบาง attribute ใหม่ทุกครั้งก่อนเขียนลงฐานข้อมูล:

```ruby
before_save { (self.callback_log ||= []) << "before_save" }
```

### `before_create` — ทำงานครั้งเดียวตอนสร้างใหม่เท่านั้น

ใช้เมื่อต้องการกำหนดค่าเริ่มต้นที่ควรเกิดขึ้น **ครั้งเดียวตอนสร้าง** และไม่ควรถูกเขียนทับซ้ำอีก
ในการ update ครั้งต่อๆ ไป เช่น สร้าง slug อัตโนมัติให้โพสต์ที่ผู้ใช้ไม่ได้ระบุมาเอง:

```ruby
before_create :assign_auto_slug_if_blank

private

def assign_auto_slug_if_blank
  self.slug = "post-#{SecureRandom.hex(4)}" if slug.blank?
end
```

```irb
irb(main):006> draft = Post.new(title: "โพสต์ฉบับร่างไม่มี slug", status: "draft")
irb(main):007> draft.save
irb(main):008> draft.slug
=> "post-9e3ddc69"
```

### เปรียบเทียบ `before_save` กับ `before_create` ให้เห็นชัดในโค้ดเดียวกัน

```ruby
before_save   { (self.callback_log ||= []) << "before_save" }
before_create { (self.callback_log ||= []) << "before_create" }
```

```irb
irb(main):009> draft.callback_log
=> ["before_save", "before_create", "after_create: SEND_EMAIL (id=8)", ...]

irb(main):010> draft.callback_log = []
irb(main):011> draft.update(title: "หัวข้อที่แก้ไขแล้วยาวพอ")
irb(main):012> draft.callback_log
=> ["before_save"]
```

ตอน `update` มีแค่ `"before_save"` ใน log ไม่มี `"before_create"` เลย — ยืนยันชัดเจนว่า
`before_create` ทำงานแค่ตอน insert record ใหม่ครั้งเดียวเท่านั้น ต่อให้เรียก `save` อีกกี่ครั้งใน
อนาคตก็จะไม่ทำงานซ้ำอีก (ต่างจาก `before_save` ที่ทำงานทุกครั้ง)

```irb
irb(main):013> draft.slug = ""
irb(main):014> draft.save(validate: false)
irb(main):015> draft.slug
=> ""
```

ลองเคลียร์ `slug` แล้ว save อีกครั้ง (ปิด validation ชั่วคราวด้วย `validate: false` เพื่อทดสอบ
เฉพาะ callback) — `slug` ยังว่างอยู่ ไม่มีใครมาเติมให้อัตโนมัติ เพราะ `before_create` ที่เคยเติม
slug ให้ตอนสร้างครั้งแรก **จะไม่ทำงานอีกแล้ว** เนื่องจากนี่คือการ update ไม่ใช่ create — ถ้า
ต้องการให้ auto-fill ทำงานทุกครั้งที่ slug ว่าง ไม่ว่าจะ create หรือ update ต้องใช้
`before_save` แทน ไม่ใช่ `before_create`

**สรุปการเลือกใช้:**

| Callback | ทำงานตอน | ใช้เมื่อ |
|-----------|-----------|-----------|
| `before_validation` | ก่อนตรวจ validation (ทั้ง create/update) | normalize/clean ข้อมูลที่ validation จะตรวจ |
| `before_save` | ก่อนบันทึก (ทั้ง create/update) | logic ที่ต้องทำงานทุกครั้งที่บันทึก |
| `before_create` | ก่อนบันทึก เฉพาะตอนสร้างใหม่ | ค่าเริ่มต้นที่ควรตั้งครั้งเดียวตอนเกิด ไม่ควรเปลี่ยนซ้ำ |

---

## Step 259: `after_create` vs `after_commit` — ทำไม Transaction Safety ถึงสำคัญ

ทุก `save` ของ ActiveRecord ถูกห่อด้วย **database transaction** โดยอัตโนมัติ (แม้จะดูเหมือนสั่ง
`INSERT` คำสั่งเดียว) เหตุผลคือ record หนึ่งตัวอาจมี callback หลายตัวที่ต้องยิง SQL เพิ่มเติม
(เช่น อัปเดต counter cache หรือบันทึก association อื่น) — Rails ต้องการให้**ทุกอย่างสำเร็จ
พร้อมกันหมด หรือไม่สำเร็จเลยสักอย่าง** (all-or-nothing) เพื่อป้องกันข้อมูลครึ่งๆ กลางๆ

นี่คือจุดที่ `after_create`/`after_save` กับ `after_commit` **ต่างกันอย่างมีนัยสำคัญ**:
`after_create` ทำงาน**ระหว่าง**ที่ transaction ยังไม่ปิด (ยังไม่รู้ว่าสุดท้ายจะ commit จริงหรือ
ถูก rollback) ส่วน `after_commit` ทำงาน**หลังจาก**ที่ transaction ยืนยันสำเร็จแล้วเท่านั้น

### สาธิตปัญหาจริงด้วยตัวอย่าง

```ruby
after_create  { (self.callback_log ||= []) << "after_create: SEND_EMAIL (id=#{id})" }
after_commit(on: :create) { (self.callback_log ||= []) << "after_commit: ENQUEUE_JOB (id=#{id})" }
```

**กรณีปกติ — transaction สำเร็จ:**

```irb
irb(main):001> post = Post.new(title: "โพสต์ปกติทำงานสำเร็จ", status: "draft")
irb(main):002> post.save
irb(main):003> post.callback_log
=>
["before_save", "before_create", "after_create: SEND_EMAIL (id=9)",
 "after_commit: ENQUEUE_JOB (id=9)"]
```

ดูเหมือนไม่มีปัญหาอะไร — ทั้งคู่ถูกเรียกตามลำดับปกติ

**กรณีอันตราย — transaction ถูก rollback หลัง save สำเร็จไปแล้ว:**

```irb
irb(main):004> post2 = Post.new(title: "โพสต์ที่จะถูกยกเลิก", status: "draft")
irb(main):005> Post.transaction do
irb(main):006>   post2.save!
irb(main):007>   # จำลองว่ามีงานอื่นในธุรกรรมเดียวกันทำพลาดทีหลัง
irb(main):008>   raise ActiveRecord::Rollback, "จำลองข้อผิดพลาดหลัง save"
irb(main):009> end

irb(main):010> post2.callback_log
=> ["before_save", "before_create", "after_create: SEND_EMAIL (id=10)"]
irb(main):011> post2.persisted?
=> false
irb(main):012> Post.exists?(title: "โพสต์ที่จะถูกยกเลิก")
=> false
```

**นี่คือปัญหาตัวจริง:** `after_create` (`SEND_EMAIL`) ถูกเรียกไปแล้วเรียบร้อย — ถ้าเป็นโค้ดจริง
ที่ส่งอีเมลหรือ SMS แจ้งเตือนผู้ใช้ อีเมลนั้น**ถูกส่งออกไปแล้วจริงๆ** ทั้งที่สุดท้ายข้อมูลทั้งหมด
ถูก rollback ทิ้งไปเลย (`persisted? == false`, ไม่มี record นี้ในฐานข้อมูลจริง) ผลลัพธ์คือผู้ใช้
ได้รับอีเมลเกี่ยวกับข้อมูลที่ไม่มีอยู่จริง เป็นบั๊กที่พบบ่อยมากในแอป Rails ที่เขียน callback
ส่ง notification ด้วย `after_create`/`after_save` โดยตรง

ในทางกลับกัน `after_commit: ENQUEUE_JOB` **ไม่ถูกเรียกเลย** เพราะไม่มีการ commit จริงเกิดขึ้น —
นี่คือพฤติกรรมที่ถูกต้อง

```irb
irb(main):013> job_logged = post2.callback_log.any? { |line| line.include?("ENQUEUE_JOB") }
irb(main):014> job_logged
=> false
```

### กฎปฏิบัติ: side effect ที่ "มองเห็นได้จากภายนอก" ต้องอยู่ใน `after_commit`

**Side effect** ในที่นี้หมายถึงการกระทำที่ส่งผลกระทบออกไปนอกฐานข้อมูลของแอปเราเอง และ
**ย้อนกลับไม่ได้** ถ้าทำไปแล้ว เช่น:

- ส่งอีเมล/SMS/push notification
- enqueue background job (Part 061) ที่จะไปเรียก external API
- เรียก external API โดยตรง (เช่น payment gateway, webhook)
- เขียนไฟล์ หรือ broadcast ผ่าน ActionCable/Turbo Streams

งานเหล่านี้ **ต้องใช้ `after_commit`** (หรือ `after_create_commit`/`after_update_commit` ที่เป็น
shortcut ของ `after_commit(on: :create)`/`after_commit(on: :update)`) เพราะรับประกันว่าข้อมูล
ที่เกี่ยวข้องถูกบันทึกจริงลงฐานข้อมูลแล้วแน่นอนก่อนที่ side effect จะเกิดขึ้น — ป้องกันสถานการณ์
"ส่งอีเมลไปแล้วแต่ข้อมูลไม่มีจริง" ตามตัวอย่างข้างบน

ส่วน `after_create`/`after_save`/`after_update` เหมาะกับงานที่ **อยู่ภายในขอบเขต transaction
เดียวกัน** เช่น อัปเดต attribute อื่นในตารางเดียวกันหรือตารางที่เกี่ยวข้อง (เพราะถ้า rollback
งานเหล่านี้จะถูก rollback ไปด้วยพร้อมกัน ไม่มีปัญหาเรื่อง "สิ่งที่ย้อนกลับไม่ได้" เกิดขึ้น)

> **ในทางปฏิบัติ (Rails 6+):** ใช้ shortcut `after_create_commit`, `after_update_commit`,
> `after_destroy_commit` แทน `after_commit(on: :create)` แบบเต็มได้ อ่านง่ายกว่าและสื่อความหมาย
> ชัดเจนกว่า:
> ```ruby
> after_create_commit :send_welcome_email
> ```

---

## Step 260: ข้อควรระวัง — Callback Overuse และ "Fat Model" Anti-pattern

Callback เป็นเครื่องมือที่ทรงพลังมาก แต่ก็เป็นจุดที่นักพัฒนามือใหม่ (และมือเก๋าจำนวนไม่น้อย)
มักใช้เกินความจำเป็นจนกลายเป็นปัญหา — ปรากฏการณ์นี้มีชื่อเรียกในวงการว่า **"callback hell"**
หรือ **"fat model" anti-pattern**

### ปัญหาของการยัด business logic ไว้ใน callback มากเกินไป

```ruby
class FatPost < Post
  after_create :reindex_search_bad
  after_create :notify_subscribers_bad
  after_create :invalidate_cache_bad
  after_create :bump_author_stats_bad

  def reindex_search_bad
    SearchIndexer.reindex(self)
  end

  def notify_subscribers_bad
    NotificationMailer.new_post(self).deliver_later
  end

  def invalidate_cache_bad
    Rails.cache.delete("posts/index")
  end

  def bump_author_stats_bad
    author.increment!(:posts_count)
  end
end
```

```irb
irb(main):001> fat = FatPost.create!(title: "โพสต์ที่สร้างผ่าน FatPost", status: "draft")
```
```
  -> [SearchIndexer] reindex post id=10
  -> [NotificationMailer] แจ้งเตือนผู้ติดตามว่ามีโพสต์ใหม่ id=10
  -> [CacheInvalidator] ล้าง cache หน้ารายการโพสต์
  -> [AuthorStats] เพิ่มตัวนับจำนวนโพสต์ของผู้เขียน
```

โค้ดแบบนี้ "ทำงานได้" แต่มีปัญหาเชิงโครงสร้างสะสมหลายอย่าง:

1. **มองไม่เห็นลำดับการทำงานจากจุดเดียว** — ใครก็ตามที่เรียก `Post.create!` ธรรมดาจะไม่รู้เลยว่า
   จริงๆ แล้วมีการ reindex, ส่งอีเมล, ล้าง cache, และอัปเดตสถิติเกิดขึ้นเบื้องหลังโดยอัตโนมัติ
   ทั้งที่โค้ดที่มองเห็นตรงหน้ามีแค่บรรทัดเดียว
2. **ทดสอบยาก** — การเขียน test ให้ `Post.create!` ต้อง mock/stub side effect ทั้ง 4 อย่างเสมอ
   ทั้งที่บาง test อาจสนใจแค่ "บันทึกโพสต์สำเร็จไหม" ไม่ได้สนใจเรื่อง search index เลย
3. **ปิดการทำงานบางส่วนไม่ได้** — ถ้าต้องการ seed ข้อมูลทดสอบ 1,000 โพสต์โดยไม่อยากส่งอีเมล
   แจ้งเตือนจริง 1,000 ฉบับ จะทำไม่ได้เลยถ้า logic นี้ผูกติดอยู่กับ callback ของ model ตรงๆ
4. **Callback ซ้อน callback** — ถ้า `bump_author_stats_bad` เปลี่ยนค่า `author` แล้ว `Author`
   model ก็มี callback ของตัวเองอีก อาจเกิดลูกโซ่ของ callback ที่ยากจะตามรอยว่าอะไรเรียกอะไร
   ต่อ (เคยเจอบั๊กจาก callback วนเรียกกันเองจนเกิด infinite loop ก็มีในโปรเจกต์จริง)

### ทางออก: แยก business logic ออกจาก model ให้ model บางเบา

```ruby
class PublishPost
  def initialize(post)
    @post = post
  end

  def call
    @post.save!
    SearchIndexer.reindex(@post)
    NotificationMailer.new_post(@post).deliver_later
    Rails.cache.delete("posts/index")
    @post.author.increment!(:posts_count)
    @post
  end
end
```

```irb
irb(main):002> plain_post = Post.new(title: "โพสต์ที่สร้างผ่าน Service Object", status: "draft")
irb(main):003> PublishPost.new(plain_post).call
```
```
  -> [SearchIndexer] reindex post id=11
  -> [NotificationMailer] แจ้งเตือนผู้ติดตามว่ามีโพสต์ใหม่ id=11
  -> [CacheInvalidator] ล้าง cache หน้ารายการโพสต์
  -> [AuthorStats] เพิ่มตัวนับจำนวนโพสต์ของผู้เขียน
```

ผลลัพธ์การทำงานเหมือนกันทุกประการ แต่ตอนนี้:

- เห็นลำดับการทำงานทั้งหมดจากจุดเดียว (`PublishPost#call`) อ่านแล้วเข้าใจ flow ทันที
- ทดสอบ `Post` model เฉยๆ ได้โดยไม่ต้องยุ่งกับ side effect ใดๆ เลย (`Post.new(...).save`
  ธรรมดาจะไม่ trigger อะไรเพิ่ม)
- ทดสอบ `PublishPost` แยกต่างหากได้ พร้อม mock แต่ละ dependency อย่างชัดเจน
- ต้องการ skip ขั้นตอนไหนก็แค่ไม่เรียก `PublishPost` แต่เรียก `post.save` ตรงๆ แทน (เช่น
  ตอน seed ข้อมูล)

**Class แบบนี้เรียกว่า "Service Object"** — เป็นแนวคิดออกแบบที่ใช้กันแพร่หลายมากในโปรเจกต์
Rails ระดับ production เพื่อแยก "สิ่งที่ record หนึ่งตัว**เป็น**" (attribute, validation พื้นฐาน
ของตัวมันเอง — สิ่งที่ควรอยู่ใน Model) ออกจาก "สิ่งที่ระบบ**ทำ**เมื่อเกิดเหตุการณ์หนึ่งขึ้น"
(orchestration ของหลาย object/หลายบริการ — สิ่งที่ไม่ควรฝังอยู่ใน Model) เรื่องนี้จะสอนอย่าง
เป็นทางการพร้อมรูปแบบมาตรฐานและตัวอย่างเต็มรูปแบบใน **Part 082 (Service Object pattern,
Form Object pattern)**

### แนวทางเลือกใช้ callback อย่างมีวินัย (สรุปเป็นกฎจำง่าย)

1. **ใช้ callback กับสิ่งที่เป็น "คุณสมบัติของ record เอง" เท่านั้น** เช่น normalize ข้อมูลของ
   ตัวเอง (`before_validation`), ตั้งค่า default ของตัวเอง (`before_create`) — ไม่ใช่การไป
   ยุ่งกับระบบอื่นหรือ object อื่นที่ไม่เกี่ยวกับตัวมันโดยตรง
2. **ถ้า callback หนึ่งตัวต้องเรียกมากกว่า 1 service ภายนอก (mailer, external API, job) ให้
   สงสัยไว้ก่อนว่าอาจถึงเวลาแยกเป็น service object แล้ว**
3. **นับจำนวน callback ใน Model เดียวกัน** ถ้าเกิน 4–5 ตัวขึ้นไป (ไม่นับ validation) มักเป็น
   สัญญาณว่า Model นั้นกำลังจะกลายเป็น "fat model" ควรพิจารณาแยกความรับผิดชอบออก
4. **หลีกเลี่ยง callback ที่แก้ไข attribute ของ object อื่น** (เช่น `author.increment!` ใน
   ตัวอย่างข้างบน) เพราะทำให้เกิดผลข้างเคียงข้าม Model ที่ตามรอยยาก — ให้ orchestrate จาก
   service object แทนเสมอ
5. **side effect ที่ย้อนกลับไม่ได้ต้องอยู่ใน `after_commit` เท่านั้น** ตามที่เรียนใน Step 259
   ไม่ว่าจะเขียนอยู่ใน Model หรือ service object ก็ตาม

> **ไม่ได้แปลว่าห้ามใช้ callback เลย** — callback ที่ใช้อย่างเหมาะสม (normalize ข้อมูล, ตั้งค่า
> default, ทำ audit log ง่ายๆ ของ record ตัวเอง) ยังเป็นเครื่องมือที่ดีและเหมาะสมมาก ปัญหาอยู่ที่
> การใช้**เกินขอบเขต**ของสิ่งที่ Model ควรรับผิดชอบ ไม่ใช่ตัว callback เองที่ผิด

---

## แบบฝึกหัด: เพิ่ม Validation และ Callback ให้ `Book` Model

### โจทย์

จาก Model `Book` ที่สร้างไว้ในแบบฝึกหัด Part 025 (มี `title`, `author`, `pages`, `read`) ให้เพิ่ม:

1. **Validation:**
   - `title` ต้องมีค่า (presence)
   - `pages` ต้องมีค่า และต้องเป็นจำนวนเต็มที่มากกว่า 0
   - `isbn` (เพิ่ม column ใหม่) ต้องไม่ซ้ำกับเล่มอื่น (uniqueness) แต่เว้นว่างได้
2. **Callback:**
   - `before_save` — normalize ตัวพิมพ์ใหญ่เล็กของ `title` ให้แต่ละคำขึ้นต้นด้วยตัวพิมพ์ใหญ่
     (เช่น `"the great gatsby"` → `"The Great Gatsby"`)
   - `after_create` — log การสร้างหนังสือใหม่ (ใช้ `Rails.logger.info`)

### เฉลย

**1) เพิ่ม column `isbn`**

```bash
bin/rails generate migration AddIsbnToBooks isbn:string
bin/rails db:migrate
```

```ruby
# db/migrate/..._add_isbn_to_books.rb
class AddIsbnToBooks < ActiveRecord::Migration[8.1]
  def change
    add_column :books, :isbn, :string
  end
end
```

**2) แก้ไข `app/models/book.rb`**

```ruby
class Book < ApplicationRecord
  before_save :normalize_title

  after_create :log_creation

  validates :title, presence: true
  validates :pages, presence: true, numericality: { only_integer: true, greater_than: 0 }
  validates :isbn, uniqueness: true, allow_blank: true

  private

  def normalize_title
    return if title.blank?

    self.title = title.strip.split(" ").map(&:capitalize).join(" ")
  end

  def log_creation
    Rails.logger.info(
      "[Book] สร้างหนังสือใหม่แล้ว id=#{id} title=#{title.inspect} author=#{author.inspect}"
    )
  end
end
```

**อธิบายจุดที่ควรสังเกต:**

- `pages` ใช้ `presence: true` ควบคู่กับ `numericality`: ถ้าไม่ใส่ `presence` ไปด้วย ค่า `nil`
  จะยังโดน `numericality` จับ error ("is not a number") อยู่ดี แต่ error message
  "can't be blank" อ่านเข้าใจง่ายกว่าสำหรับผู้ใช้ปลายทาง — ใส่คู่กันจึงได้ทั้งสองข้อความ
  ครอบคลุมทุกกรณี
- `isbn` ใช้ `allow_blank: true` เพราะบางเล่มอาจยังไม่รู้ ISBN ตอนบันทึกครั้งแรก (หนังสือเก่ามาก
  หรือเอกสารที่ไม่มี ISBN จริง) — validation จึงอนุญาตให้เว้นว่างได้ แต่ถ้ามีค่าแล้วต้องไม่ซ้ำ
  กับเล่มอื่นเท่านั้น
- `normalize_title` ใช้ `before_save` (ไม่ใช่ `before_create`) เพราะต้องการให้ normalize
  ทำงานทุกครั้งที่บันทึก ไม่ว่าจะสร้างใหม่หรือแก้ไขชื่อภายหลัง
- `log_creation` ใช้ `Rails.logger.info` แทน `puts` เพราะเป็นแนวทางที่ถูกต้องในโค้ด production
  จริง (`puts` จะหายไปเมื่อรันเป็น background process ที่ไม่มี terminal ติดอยู่ ส่วน
  `Rails.logger` เขียนลงไฟล์ log เสมอไม่ว่าจะรันแบบไหน)

**3) ทดสอบผ่าน `rails console`**

```irb
irb(main):001> Book.create(title: nil, author: "Unknown", pages: 100).errors.full_messages
=> ["Title can't be blank"]

irb(main):002> Book.new(title: "Some Title", pages: nil).tap(&:valid?).errors[:pages]
=> ["can't be blank", "is not a number"]
irb(main):003> Book.new(title: "Some Title", pages: 0).tap(&:valid?).errors[:pages]
=> ["must be greater than 0"]
irb(main):004> Book.new(title: "Some Title", pages: -5).tap(&:valid?).errors[:pages]
=> ["must be greater than 0"]
irb(main):005> Book.new(title: "Some Title", pages: 3.5).tap(&:valid?).errors[:pages]
=> ["must be an integer"]

irb(main):006> Book.create!(title: "Clean Code", author: "Robert C. Martin", pages: 464,
                              isbn: "978-0132350884")
irb(main):007> dup_isbn = Book.new(title: "Clean Code (สำเนา)", author: "Robert C. Martin",
                                     pages: 464, isbn: "978-0132350884")
irb(main):008> dup_isbn.valid?
=> false
irb(main):009> dup_isbn.errors[:isbn]
=> ["has already been taken"]

irb(main):010> no_isbn_ok = Book.new(title: "หนังสือไม่มี ISBN", author: "ไม่ระบุ", pages: 100)
irb(main):011> no_isbn_ok.valid?
=> true

irb(main):012> messy = Book.create!(title: "  the great gatsby  ", author: "F. Scott Fitzgerald",
                                       pages: 180)
irb(main):013> messy.title
=> "The Great Gatsby"
```

`normalize_title` ใช้ `.strip` ตัด whitespace หัวท้ายออกก่อน แล้ว `.split(" ")` แยกเป็น Array
ของคำ ตาม `.map(&:capitalize)` ทำให้ตัวอักษรแรกของแต่ละคำเป็นตัวพิมพ์ใหญ่ (ตัวที่เหลือเป็นตัว
พิมพ์เล็ก) แล้ว `.join(" ")` รวมกลับเป็น string เดียว — ผลลัพธ์คือ `"  the great gatsby  "`
กลายเป็น `"The Great Gatsby"` เรียบร้อย

ตรวจไฟล์ `log/development.log` เพื่อยืนยันว่า `after_create` ทำงานจริง:

```bash
tail -5 log/development.log
```

```
[Book] สร้างหนังสือใหม่แล้ว id=3 title="Sapiens" author="Yuval Noah Harari"
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม validation ให้ `Book#author` ต้องมีค่าเสมอเช่นกัน (presence) จากนั้นเพิ่ม custom
   validation method ชื่อ `pages_reasonable_for_genre` ที่ตรวจว่าถ้า `genre` (เพิ่ม column ใหม่
   ชนิด string) เป็น `"เด็ก"` แล้ว `pages` ต้องไม่เกิน 100 หน้า (ใบ้: ใช้ `validate
   :pages_reasonable_for_genre` แล้วเรียก `errors.add(:pages, "...")` ข้างในเมื่อเงื่อนไขไม่ผ่าน)
2. เปลี่ยน `normalize_title` ให้ทำงานผ่าน `before_validation` แทน `before_save` แล้วสังเกตว่า
   ผลลัพธ์ต่างจากเดิมหรือไม่ อธิบายด้วยตัวเองว่าทำไมในกรณีนี้ (ที่ยังไม่มี validation อื่นมา
   ตรวจสอบรูปแบบของ `title`) การย้ายไป `before_validation` แทบไม่เปลี่ยนพฤติกรรมที่สังเกตเห็น
   ได้ ต่างจากตัวอย่าง `downcase_slug` ใน Step 258 ที่ย้ายแล้วมีผลชัดเจนมาก
3. เพิ่ม `after_commit(on: :create)` ให้ `Book` ที่ log ข้อความ `"[Book] commit สำเร็จแล้ว
   id=..."` แยกจาก `after_create` เดิม แล้วเขียนสคริปต์ (รันด้วย `bin/rails runner`) ที่จำลอง
   การ rollback กลางคันแบบ Step 259 (ใช้ `Book.transaction do ... raise
   ActiveRecord::Rollback end`) เพื่อพิสูจน์ด้วยตัวเองว่า `after_create` (log การสร้าง) ถูก
   เรียกไปแล้ว แต่ `after_commit` ไม่ถูกเรียกเลยเมื่อ rollback

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Validation** คือกฎที่ประกาศไว้ในระดับ Model เพื่อป้องกันข้อมูลผิดพลาดก่อนบันทึกลงฐานข้อมูล
  แก้ปัญหาที่พบใน Part 025 ที่ `title: nil` เคยบันทึกผ่านได้อย่างเงียบๆ
- `valid?` ตรวจสอบได้โดยไม่ต้อง save จริง และ `errors.full_messages`/`errors[:field]` ใช้ดู
  รายละเอียดปัญหาได้ครบทุก field พร้อมกัน
- `save`/`create` คืน `false`/object ที่ไม่ persisted เมื่อ validation ไม่ผ่าน ส่วน
  `save!`/`create!` raise `ActiveRecord::RecordInvalid` แทน — เลือกใช้ให้เหมาะกับบริบท
- Built-in validator ที่ใช้บ่อย: `presence`, `length`, `numericality`, `uniqueness`, `format`,
  `inclusion`/`exclusion` — แต่ละตัวมีตัวเลือกละเอียดปรับได้ตามต้องการ พร้อม `allow_nil:`/
  `allow_blank:` ควบคุมกรณีค่าว่าง
- เขียน custom validation ด้วย `validate :method_name` และ `errors.add(:field, ...)` หรือ
  `errors.add(:base, ...)` สำหรับกฎที่ซับซ้อนเกินกว่า built-in validator จะรองรับ
- ใช้ `if:`/`unless:` (symbol หรือ lambda) ทำ conditional validation และ `on: :create`/
  `on: :update`/custom context เพื่อบังคับใช้กฎต่างกันตามช่วงชีวิตของ record
- **Callback** คือ hook ที่แทรกโค้ดเข้าไปในวงจรชีวิตของ object ได้ — เข้าใจลำดับทางการ
  (`before_validation` → `after_validation` → `before_save` → `before_create`/`before_update`
  → `after_create`/`after_update` → `after_save` → `after_commit`/`after_rollback`) อย่างแม่นยำ
- เลือกใช้ `before_validation` สำหรับ normalize ข้อมูลก่อนตรวจ, `before_save` สำหรับ logic ที่
  ต้องทำงานทุกครั้ง, `before_create` สำหรับค่าเริ่มต้นที่ตั้งครั้งเดียวตอนสร้าง
- **`after_commit` ปลอดภัยกว่า `after_create`/`after_save`** สำหรับ side effect ที่ย้อนกลับ
  ไม่ได้ (ส่งอีเมล, enqueue job, เรียก external API) เพราะรับประกันว่า transaction commit
  สำเร็จแล้วจริงก่อนที่ side effect จะเกิดขึ้น — พิสูจน์ด้วยตัวอย่างจริงที่ rollback กลางคัน
- ระวัง **callback overuse / fat model anti-pattern** — callback ควรใช้กับสิ่งที่เป็นคุณสมบัติ
  ของ record เอง ไม่ใช่ orchestration ของหลายระบบ ซึ่งควรแยกไปเป็น **Service Object** แทน
  (จะสอนเต็มรูปแบบใน Part 082)

**ต่อไป (Part 027):** ตอนนี้ `Post` และ `Book` ยังเป็น Model เดี่ยวๆ ที่ไม่เชื่อมโยงกับตารางอื่น
เลย แต่แอปจริงแทบทุกตัวมีความสัมพันธ์ระหว่างข้อมูล เช่น หนึ่งโพสต์มีหลายคอมเมนต์ หรือหนึ่ง
หนังสือมีผู้แต่งหนึ่งคน Part 027 จะสอนเรื่อง **Association** — `belongs_to`, `has_many`, และ
`has_one` ซึ่งเป็นกลไกที่ทำให้ ActiveRecord จัดการความสัมพันธ์ระหว่างตารางให้เราโดยแทบไม่ต้อง
เขียน SQL JOIN เองเลย
