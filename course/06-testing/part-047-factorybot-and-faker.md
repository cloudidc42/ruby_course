# Part 047: FactoryBot, Faker และการจัดการ Test Data

> **Step ครอบคลุมใน Part นี้:** Step 461–470
> **ระดับ:** ปานกลาง (ต่อจาก Part 046 เรื่อง RSpec สำหรับ Rails)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x — ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4,
> `factory_bot_rails` 6.5.1 (คุม `factory_bot` 6.6.0), `faker` 3.8.0

ใน Part 046 เราเขียน model spec และ request spec ด้วย RSpec ไปแล้วหลายสิบ example
โดยสร้างข้อมูลทดสอบด้วยวิธีที่ตรงไปตรงมาที่สุด: เรียก `Post.create(title: "...", body:
"...")` หรือ `User.create(email_address: "...", password: "...")` ตรงๆ ในทุก spec ที่
ต้องใช้ record นั้น — วิธีนี้เข้าใจง่ายและไม่ต้องเรียนรู้อะไรเพิ่ม แต่ Part นี้จะพาไปดู
ปัญหาที่ซ่อนอยู่ของมันแบบ **เห็นของจริง ไม่ใช่แค่ทฤษฎี**: เราจะจำลองสถานการณ์ที่เกิดขึ้น
บ่อยมากในงานจริง — ทีมเพิ่ม column ที่ required เข้าไปในตาราง `users` — แล้วดูว่า spec
ที่เขียนด้วยมือพังกระจายไปกี่จุด ก่อนจะแนะนำ **FactoryBot** เครื่องมือมาตรฐานของวงการ
Rails สำหรับจัดการข้อมูลทดสอบ ที่แก้ปัญหานี้ด้วยหลักการเดียวกับ DRY ที่เราใช้ทั่วทั้ง
หลักสูตรมา และ **Faker** เครื่องมือสร้างข้อมูลสุ่มที่สมจริง

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยการสร้างแอป Rails
> เปล่าๆ ขึ้นมา 1 ตัว (`testdata_demo`) ติดตั้ง `rspec-rails`, `factory_bot_rails`,
> `faker` จริง สร้าง model `User`/`Post`/`Comment` จริง รัน `bundle exec rspec` จริง
> ทุกจุด รวมถึงการจงใจทำให้ spec พังเพื่อดู error message จริง และรัน benchmark จริง
> เพื่อวัดความเร็ว — ทุก output ที่เห็นในเอกสารนี้คัดลอกมาจากผลลัพธ์จริงที่ได้รับ
> ไม่ใช่ตัวอย่างที่เขียนขึ้นลอยๆ

## สารบัญของ Part นี้

- Step 461: ปัญหาของการเขียน `Post.create` มือในทุก spec — ทำไมมันพังกระจายเมื่อ schema เปลี่ยน
- Step 462: ติดตั้ง FactoryBot (`factory_bot_rails`) และโครงสร้าง `spec/factories`
- Step 463: นิยาม factory แก้ปัญหาจาก Step 461 — `create`, `build`, `build_stubbed` ต่างกันอย่างไร
- Step 464: Sequence สำหรับ attribute ที่ต้อง unique (`sequence(:email)`)
- Step 465: Associations ใน factory — implicit vs explicit, การสร้าง object graph
- Step 466: Traits — สร้างข้อมูลหลายแบบโดยไม่ต้องซ้ำ factory
- Step 467: Factory callbacks (`after(:create)`) และการ override attribute ตอนเรียกใช้งาน
- Step 468: ผสาน Faker เข้ากับ factory เพื่อข้อมูลสมจริง (รวม locale ภาษาไทย)
- Step 469: หลีกเลี่ยง anti-pattern — Lean Factory Principle และ `FactoryBot.lint`
- Step 470: แบบฝึกหัด — ชุด factory เต็มรูปแบบสำหรับ Post/Comment/User

---

## Step 461: ปัญหาของการเขียน `Post.create` มือในทุก spec — ทำไมมันพังกระจายเมื่อ schema เปลี่ยน

### สถานการณ์ตั้งต้น (สไตล์ที่ Part 046 ใช้)

สมมติเรามี domain ง่ายๆ: `User` มี `has_many :posts`, `Post` มี `has_many :comments`,
`Comment` `belongs_to :post` และ `belongs_to :user` — เหมือนระบบบล็อกจาก Part 030 แต่มี
authentication จาก Part 041 ผสมเข้ามา (`has_secure_password`)

Model spec 3 ไฟล์ ที่เขียนตามสไตล์ Part 046 (สร้างข้อมูลด้วยมือทุกจุด):

```ruby
# spec/models/user_spec.rb
require "rails_helper"

RSpec.describe User, type: :model do
  it "is valid with an email and name" do
    user = User.create(
      email_address: "user1@example.com",
      name: "สมชาย",
      password: "secret123",
      password_confirmation: "secret123"
    )
    expect(user).to be_persisted
  end

  it "requires an email" do
    user = User.new(name: "สมหญิง", password: "secret123", password_confirmation: "secret123")
    expect(user).not_to be_valid
  end
end
```

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  it "is valid with a title and body" do
    user = User.create(
      email_address: "post-author@example.com",
      name: "ผู้เขียน",
      password: "secret123",
      password_confirmation: "secret123"
    )
    post = Post.create(title: "บทความแรก", body: "เนื้อหาบทความ", user: user)
    expect(post).to be_persisted
  end

  it "requires a title" do
    user = User.create(
      email_address: "post-author2@example.com",
      name: "ผู้เขียน2",
      password: "secret123",
      password_confirmation: "secret123"
    )
    post = Post.new(body: "เนื้อหาบทความ", user: user)
    expect(post).not_to be_valid
  end
end
```

```ruby
# spec/models/comment_spec.rb
require "rails_helper"

RSpec.describe Comment, type: :model do
  it "is valid with a body" do
    user = User.create(
      email_address: "commenter@example.com",
      name: "ผู้แสดงความเห็น",
      password: "secret123",
      password_confirmation: "secret123"
    )
    post_author = User.create(
      email_address: "commenter-post-author@example.com",
      name: "ผู้เขียนโพสต์",
      password: "secret123",
      password_confirmation: "secret123"
    )
    post = Post.create(title: "บทความ", body: "เนื้อหา", user: post_author)
    comment = Comment.create(body: "ความเห็นดีมาก", post: post, user: user)
    expect(comment).to be_persisted
  end
end
```

รันดู — ผ่านหมดตามคาด:

```bash
bundle exec rspec spec/models
```

```
.....

Finished in 0.08265 seconds (files took 2.18 seconds to load)
5 examples, 0 failures
```

สังเกตว่าแค่ 3 ไฟล์ spec นี้ก็มีการเรียก `User.create(...)` แบบเขียนครบทุก field
ซ้ำกันแล้วถึง **5 ครั้ง** และทุกครั้งต้องพิมพ์ `email_address`, `name`, `password`,
`password_confirmation` ครบเหมือนกันหมด — ในโปรเจกต์จริงที่มี model spec, request spec,
system spec รวมกันหลักร้อยไฟล์ ตัวเลขนี้อาจเป็นหลักร้อยหรือหลักพันจุด

### วันหนึ่ง Product Owner ขอเพิ่ม field ที่ required

สมมติทีม product ขอให้ทุก user ต้องมีเบอร์โทรศัพท์ (ไว้ใช้ยืนยันตัวตนแบบ SMS OTP ใน
อนาคต) — เพิ่ม migration และ validation ตามปกติ:

```bash
bin/rails generate migration AddPhoneNumberToUsers phone_number:string
bin/rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy

  validates :email_address, presence: true, uniqueness: true
  validates :name, presence: true
  validates :phone_number, presence: true   # <-- บรรทัดใหม่
end
```

การเปลี่ยนแปลงนี้ดู "เล็กมาก" — เพิ่ม validation หนึ่งบรรทัด แต่มาดูว่าเกิดอะไรขึ้นกับ
spec ทั้ง 3 ไฟล์ที่เพิ่งผ่านหมด:

```bash
bundle exec rspec spec/models
```

ผลลัพธ์จริง:

```
FF.F.

Failures:

  1) Comment is valid with a body
     Failure/Error: post = Post.create(title: "บทความ", body: "เนื้อหา", user: post_author)

     ActiveRecord::NotNullViolation:
       SQLite3::ConstraintException: NOT NULL constraint failed: posts.user_id

  2) Post is valid with a title and body
     Failure/Error: post = Post.create(title: "บทความแรก", body: "เนื้อหาบทความ", user: user)

     ActiveRecord::NotNullViolation:
       SQLite3::ConstraintException: NOT NULL constraint failed: posts.user_id

  3) User is valid with an email and name
     Failure/Error: expect(user).to be_persisted
       expected `#<User id: nil, email_address: [FILTERED], name: "สมชาย", password_digest: [FILTERED], admin: false, created_at: nil, updated_at: nil, phone_number: nil>.persisted?` to be truthy, got false

Finished in 0.05096 seconds (files took 1.71 seconds to load)
5 examples, 3 failures
```

**3 จาก 5 example พังทันที** จากการเปลี่ยนแปลง 1 บรรทัด — และที่อันตรายกว่านั้นคือ
**error message 2 ใน 3 ตัวไม่ได้บอกอะไรเกี่ยวกับ `phone_number` เลย**:

- Failure ที่ 3 (`User is valid...`) พอเข้าใจได้ตรงๆ: `user.persisted?` เป็น `false`
  เพราะ validation ใหม่ทำให้ `save` ล้มเหลว
- แต่ Failure ที่ 1 และ 2 (`Post`/`Comment`) กลับรายงาน
  **`NOT NULL constraint failed: posts.user_id`** — เป็นข้อความที่ทำให้คนอ่านงงว่า
  เกี่ยวอะไรกับ `phone_number` เพราะสาเหตุจริงถูกซ่อนอยู่คนละชั้น: `User.create(...)`
  ใน spec เหล่านั้น **ไม่ได้เช็คผลลัพธ์** ว่า `user` ถูกสร้างสำเร็จหรือไม่ (ไม่มี
  `user.valid?` หรือ `user.persisted?` cheek) เมื่อ `user` สร้างไม่สำเร็จเพราะขาด
  `phone_number` มันจะได้ object ที่ `id` เป็น `nil` กลับมา แล้วโค้ดบรรทัดถัดไปเอา
  `user` ตัวนั้น (ที่ยังไม่มี `id`) ไปสร้าง `Post.create(..., user: user)` — Rails
  พยายาม insert แถว posts ที่มี `user_id = nil` ซึ่งชนกับ `NOT NULL` constraint ระดับ
  ฐานข้อมูลที่ตั้งไว้ตอนสร้างตาราง `posts` (Part 027) ทันที

นี่คือรูปแบบ **cascading failure** ที่พบได้บ่อยมากในสายงานจริง: การเปลี่ยนแปลงเล็กๆ ที่
model หนึ่ง ทำให้เกิด error message ที่ดูไม่เกี่ยวข้องกันเลยกระจายไปทั่วทั้ง suite
คนที่ไม่รู้สาเหตุต้นตอ (เช่น เพื่อนร่วมทีมที่ pull โค้ดมาแล้วรัน spec แล้วพบว่าพัง) อาจ
ใช้เวลานานกว่าจะไล่เจอว่าจริงๆ แล้วต้นเหตุอยู่ที่ `User.create` ใน `post_spec.rb` ไม่ใช่
โค้ดใน `Post` เอง

และนี่แค่ 3 ไฟล์เท่านั้น — ลองจินตนาการว่าถ้าโปรเจกต์มี spec ที่เขียน
`User.create(email_address: ..., name: ..., password: ...)` กระจายอยู่ **50 จุด**
การเพิ่ม field required 1 field จะทำให้ต้องไปตามแก้ **50 จุด** (หรือมากกว่านั้น ถ้า
บาง spec สร้าง user มากกว่า 1 ตัว) นี่คือปัญหาการ **ไม่ DRY** แบบเดียวกับที่เราหลีกเลี่ยง
มาตลอดหลักสูตรตั้งแต่ Part 001 เพียงแต่คราวนี้เกิดขึ้นใน **test code** ไม่ใช่
production code

> **สิ่งที่ต้องการจริงๆ:** วิธีนิยาม "ค่าเริ่มต้นของ record ที่ valid" ไว้ **ที่เดียว**
> แล้วให้ทุก spec ดึงมาใช้ — ถ้าวันหนึ่ง schema เปลี่ยน ก็แก้แค่จุดเดียว spec ทั้งหมด
> จะปรับตามอัตโนมัติ — นี่คือสิ่งที่ **FactoryBot** ทำให้เราตั้งแต่ Step ถัดไป

---

## Step 462: ติดตั้ง FactoryBot (`factory_bot_rails`) และโครงสร้าง `spec/factories`

### `factory_bot` vs `factory_bot_rails`

- **`factory_bot`** คือ gem หลักที่มี DSL สำหรับนิยาม "factory" (พิมพ์เขียวของการสร้าง
  record หนึ่งชนิด) ใช้ได้กับ Ruby ล้วนๆ ไม่ผูกกับ Rails
- **`factory_bot_rails`** คือ gem เสริมที่ทำให้ `factory_bot` **ทำงานร่วมกับ Rails**
  ได้แนบเนียน: โหลดไฟล์ทั้งหมดใน `spec/factories/` ให้อัตโนมัติตอนบูต test
  environment, และที่สำคัญที่สุดคือ **ผูกกับ Rails generator** — ทุกครั้งที่รัน
  `rails generate model` มันจะสร้าง factory stub ให้อัตโนมัติคู่กับ model (คล้ายที่
  `rspec-rails` สร้าง spec file stub ให้ใน Part 046)

เพิ่มลง `Gemfile` (กลุ่ม `:development, :test` เดียวกับ `rspec-rails`):

```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end
```

```bash
bundle install
```

ผลลัพธ์จริง (ย่อ):

```
Fetching faker 3.8.0
Installing faker 3.8.0
Bundle complete! 27 Gemfile dependencies, 133 gems now installed.
```

### สร้าง model แล้วดูว่า generator สร้างอะไรให้บ้าง

```bash
bin/rails generate model User email_address:string name:string password_digest:string admin:boolean
```

ผลลัพธ์จริง — สังเกตบรรทัดที่มีคำว่า `factory_bot`:

```
      invoke  active_record
      create    db/migrate/20260926064531_create_users.rb
      create    app/models/user.rb
      invoke    rspec
      create      spec/models/user_spec.rb
      invoke      factory_bot
      create        spec/factories/users.rb
```

นี่คือประโยชน์ข้อแรกของ `factory_bot_rails`: **ไม่ต้องสร้างไฟล์ factory ด้วยมือ** ทุกครั้งที่
สร้าง model ใหม่ generator จะสร้าง **stub file** ให้ทันที เปิดดู:

```ruby
# spec/factories/users.rb (สร้างโดย generator อัตโนมัติ — ยังใช้งานจริงไม่ได้)
FactoryBot.define do
  factory :user do
    email_address { "MyString" }
    name { "MyString" }
    password_digest { "MyString" }
    admin { false }
  end
end
```

สังเกต pattern ที่ generator เดาให้ตาม column type: `string` → `"MyString"`,
`text` → `"MyText"`, `integer` → `1`, `boolean` → `false`, `date` → วันที่ปัจจุบัน,
`references` → `nil` (ปล่อยว่างไว้ให้เราต้องมาเติมเอง) — **stub นี้ใช้งานจริงไม่ได้ทันที**
ด้วยเหตุผลหลายข้อที่จะแก้ทีละจุดใน Step ถัดๆ ไป:

1. `password_digest { "MyString" }` — ผิดหลักการโดยตรง: `has_secure_password`
   ต้องการให้เราตั้งค่า **virtual attribute** `password`/`password_confirmation`
   (ตามที่เรียนใน Part 041 Step 402) ไม่ใช่ตั้งค่า `password_digest` เป็น string
   ธรรมดาเฉยๆ (ถ้าทำแบบนั้น `authenticate` จะใช้งานไม่ได้เพราะ `"MyString"` ไม่ใช่
   bcrypt hash ที่ถูกต้อง)
2. ถ้าสร้าง user 2 ตัวด้วย factory นี้ตรงๆ จะได้ `email_address` ซ้ำกันทุกตัว
   (`"MyString"` คงที่) ชน uniqueness validation ทันที (แก้ใน Step 464 ด้วย sequence)
3. ยังไม่มี association ให้กับ `Post`/`Comment` (แก้ใน Step 465)

ทำแบบเดียวกันกับ `Post` และ `Comment`:

```bash
bin/rails generate model Post title:string body:text published:boolean published_on:date user:references view_count:integer
bin/rails generate model Comment body:text post:references user:references
```

Stub ที่ได้:

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { "MyString" }
    body { "MyText" }
    published { false }
    published_on { "2026-09-26" }
    user { nil }
    view_count { 1 }
  end
end
```

```ruby
# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { "MyText" }
    post { nil }
    user { nil }
  end
end
```

`user { nil }` และ `post { nil }` คือจุดที่อันตรายที่สุด — ถ้าเรียก `create(:post)`
ตรงๆ ด้วย factory นี้จะได้ `NOT NULL constraint failed` ทันที (เหมือนปัญหาที่เจอใน
Step 461 เป๊ะๆ) เพราะ `Post belongs_to :user` และ generator ไม่รู้ว่าจะสร้าง user
ให้อัตโนมัติอย่างไร — ต้องแก้เป็น association เอง (Step 465)

### เปิดใช้งาน syntax สั้น (`create`/`build` แทน `FactoryBot.create`/`FactoryBot.build`)

ค่าเริ่มต้นต้องเรียกผ่าน `FactoryBot.create(:user)` เต็มๆ ทุกครั้ง เพิ่ม config
ให้เรียกสั้นๆ ได้:

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  # ...(บรรทัดเดิมจาก Part 046)...

  # ทำให้เรียก create/build/build_stubbed ได้ตรงๆ โดยไม่ต้องพิมพ์ FactoryBot.create ทุกครั้ง
  config.include FactoryBot::Syntax::Methods
end
```

จาก Step 463 เป็นต้นไป เราจะเขียน `create(:user)` แทน `FactoryBot.create(:user)`
ทุกจุด (เป็นธรรมเนียมมาตรฐานที่ทีม Rails เกือบทุกทีมทำ)

---

## Step 463: นิยาม factory แก้ปัญหาจาก Step 461 — `create`, `build`, `build_stubbed` ต่างกันอย่างไร

### แก้ stub ให้ใช้งานได้จริง — จุดแก้ไขจุดเดียวที่แก้ปัญหาทั้งหมดของ Step 461

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email_address) { |n| "user#{n}@example.com" }
    name { "ผู้ใช้ทดสอบ" }
    password { "secret123" }
    password_confirmation { "secret123" }
    phone_number { "0800000000" }
    admin { false }
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { "บทความทดสอบ" }
    body { "เนื้อหาบทความทดสอบ" }
    published { false }
    published_on { nil }
    view_count { 0 }
    association :user
  end
end
```

```ruby
# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { "ความเห็นทดสอบ" }
    association :post
    association :user
  end
end
```

(`sequence` และ `association` จะอธิบายละเอียดใน Step 464–465 ตอนนี้ขอโฟกัสที่ผลลัพธ์
ก่อน)

ทีนี้แก้ spec ทั้ง 3 ไฟล์จาก Step 461 ให้ใช้ factory แทนการเขียนมือ:

```ruby
# spec/models/user_spec.rb
require "rails_helper"

RSpec.describe User, type: :model do
  it "is valid with an email and name" do
    user = create(:user)
    expect(user).to be_persisted
  end

  it "requires an email" do
    user = build(:user, email_address: nil)
    expect(user).not_to be_valid
  end
end
```

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  it "is valid with a title and body" do
    post = create(:post)
    expect(post).to be_persisted
  end

  it "requires a title" do
    post = build(:post, title: nil)
    expect(post).not_to be_valid
  end
end
```

```ruby
# spec/models/comment_spec.rb
require "rails_helper"

RSpec.describe Comment, type: :model do
  it "is valid with a body" do
    comment = create(:comment)
    expect(comment).to be_persisted
  end
end
```

```bash
bundle exec rspec spec/models
```

```
.....

Finished in 0.0407 seconds (files took 0.69758 seconds to load)
5 examples, 0 failures
```

**กลับมาผ่านหมดอีกครั้ง** — และจุดสำคัญที่สุดคือ: ถ้าวันนี้ทีม product ขอเพิ่ม field
required อีกตัว เราจะแก้ **แค่ใน `spec/factories/users.rb` ไฟล์เดียว** ไม่ต้องไล่แก้
ทีละ spec เหมือน Step 461 อีกต่อไป — spec ทุกไฟล์ที่เรียก `create(:user)` จะได้รับค่า
ที่ถูกต้องอัตโนมัติทันทีที่ factory ถูกอัปเดต นี่คือหัวใจของ FactoryBot: **นิยาม
"ค่าเริ่มต้นที่ valid" ไว้ที่เดียว**

### `create`, `build`, `build_stubbed` — ความต่างที่ต้องเข้าใจให้แน่น

FactoryBot มี **strategy** หลักอยู่ 3 แบบ ที่มักสับสนกันบ่อยเพราะหน้าตาคล้ายกันมาก
มาพิสูจน์ความต่างด้วย spec จริง:

```ruby
require "rails_helper"

RSpec.describe "FactoryBot strategies", type: :model do
  it "create persists to the database" do
    user = create(:user)
    expect(user.id).not_to be_nil
    expect(User.exists?(user.id)).to eq(true)
  end

  it "build does not persist" do
    user = build(:user)
    expect(user.id).to be_nil
    expect(user.persisted?).to eq(false)
    expect(User.count).to eq(0)
  end

  it "build_stubbed does not touch the database at all, but has a fake id" do
    user = build_stubbed(:user)
    expect(user.id).not_to be_nil
    expect(user.persisted?).to eq(true)
    expect(User.count).to eq(0)
  end

  it "build_stubbed refuses any database access at all (no real DB row backs it)" do
    user = build_stubbed(:user)
    expect { user.reload }.to raise_error(RuntimeError, /not allowed to access the database/)
  end
end
```

```bash
bundle exec rspec spec/models/factory_strategies_spec.rb
```

```
....

Finished in 0.02867 seconds (files took 0.67765 seconds to load)
4 examples, 0 failures
```

ทั้ง 4 example ผ่าน — สิ่งที่น่าสนใจที่สุดคือ example สุดท้าย: ตอนแรกที่เขียน spec
คาดว่าการเรียก `user.reload` บน object จาก `build_stubbed` น่าจะได้
`ActiveRecord::RecordNotFound` (เหมือนหา id ในฐานข้อมูลไม่เจอ) แต่ผลลัพธ์จริงกลับเป็น

```
RuntimeError: stubbed models are not allowed to access the database - User#reload()
```

นี่คือพฤติกรรมที่ตั้งใจของ `build_stubbed`: มันไม่ได้แค่ "ไม่บันทึกลงฐานข้อมูล" เหมือน
`build` แต่มันไป **stub method ที่เกี่ยวกับฐานข้อมูลทั้งหมดให้ raise error ทันทีที่ถูก
เรียก** เพื่อบังคับให้ spec ที่ใช้ `build_stubbed` ต้องเป็น "pure" จริงๆ ไม่แอบยิง query
ลงฐานข้อมูลแบบไม่รู้ตัว (ถ้าโค้ดที่ทดสอบพยายาม query จริงจะ error ทันที บอกชัดเจนว่า
ต้องเปลี่ยนไปใช้ `create` แทน)

สรุปความต่างเป็นตาราง:

| | `create(:post)` | `build(:post)` | `build_stubbed(:post)` |
|---|---|---|---|
| บันทึกลงฐานข้อมูลจริง | ใช่ | ไม่ | ไม่ |
| `id` | เลข id จริงจาก DB | `nil` | เลข id **ปลอม** (fake, เพิ่มทีละ 1) |
| `persisted?` | `true` | `false` | `true` (หลอกให้โค้ดที่เช็ค `persisted?` ทำงานเหมือนมี record จริง) |
| เข้าถึง association จริงได้ | ใช่ (query จริง) | ใช่ (แต่สร้าง record ใหม่ตาม strategy เดียวกัน) | **ไม่ได้** — raise error ทันทีถ้าพยายาม query |
| ความเร็ว | ช้าที่สุด (round-trip DB) | เร็วกว่า create | **เร็วที่สุด** (ไม่แตะ DB เลย) |
| ใช้เมื่อไหร่ | ต้องการ record จริงในฐานข้อมูล (test callback, query, transaction, association จริง) | ทดสอบ validation ที่ไม่ต้องการให้บันทึกจริง | ทดสอบ logic ล้วนๆ ที่ไม่แตะฐานข้อมูล (presenter, serializer, method คำนวณ, view) |

### วัดความเร็วจริง

```ruby
# script/bench.rb (ไฟล์ทดลองชั่วคราว ลบทิ้งหลังใช้งาน)
require_relative "../config/environment"
require "factory_bot"
require "benchmark"

n = 200
Benchmark.bm(20) do |x|
  x.report("create(:user) x#{n}") { n.times { FactoryBot.create(:user) } }
  x.report("build_stubbed x#{n}") { n.times { FactoryBot.build_stubbed(:user) } }
end
```

```bash
RAILS_ENV=test bundle exec ruby script/bench.rb
```

ผลลัพธ์จริง:

```
                           user     system      total        real
create(:user) x200     0.628894   0.019747   0.648641 (  0.650766)
build_stubbed x200     0.267514   0.000000   0.267514 (  0.267519)
```

`build_stubbed` เร็วกว่า `create` ประมาณ **2.4 เท่า** ในการทดลองนี้ — ส่วนหนึ่งเป็น
เพราะ `User` มี `has_secure_password` ซึ่งต้อง hash รหัสผ่านด้วย bcrypt ทุกครั้งที่
`create` (ตามที่เรียนใน Part 041 — bcrypt ถูกออกแบบมาให้ **ช้าโดยตั้งใจ** เพื่อป้องกัน
brute-force) ส่วน `build_stubbed` ข้ามขั้นตอนนี้ไปเลยเพราะไม่มีการเรียก `save` จริง
ในแอปจริงที่มี association ซับซ้อนกว่านี้ (เช่น `Post` ที่ต้อง `create` ทั้ง `user`
และตัวมันเอง) ส่วนต่างนี้จะยิ่งเห็นชัดขึ้นเมื่อ test suite มีหลักพัน example

> **กฎการเลือกใช้ในทางปฏิบัติ:** เริ่มต้นด้วยคำถาม "spec นี้จำเป็นต้องมี record จริงใน
> ฐานข้อมูลไหม" ถ้าทดสอบ controller/request ที่ query ฐานข้อมูลจริง หรือทดสอบ callback/
> validation ที่ทำงานตอน save ต้องใช้ `create` ถ้าแค่ทดสอบ validation logic ที่ไม่สน
> การบันทึกจริง ใช้ `build` ถ้าทดสอบ method ที่ไม่แตะฐานข้อมูลเลย (เช่น presenter,
> serializer, helper, decorator) ใช้ `build_stubbed` เสมอเพื่อความเร็ว

---

## Step 464: Sequence สำหรับ attribute ที่ต้อง unique (`sequence(:email)`)

### ปัญหาที่ `sequence` แก้

ถ้า factory `:user` ตั้ง `email_address { "user@example.com" }` แบบ static เฉยๆ
(ไม่มี `sequence`) การเรียก `create(:user)` สองครั้งติดกันจะพังทันที เพราะ
`validates :email_address, uniqueness: true` (Part 032) จะเจอค่าซ้ำ

วิธีแก้คือ `sequence` — บอก FactoryBot ให้แทรกตัวเลขที่เพิ่มขึ้นทีละ 1 (เริ่มจาก 1)
ลงในค่าทุกครั้งที่เรียก:

```ruby
factory :user do
  sequence(:email_address) { |n| "user#{n}@example.com" }
  # ...
end
```

ทดสอบใน console:

```irb
irb> 5.times { u = FactoryBot.build(:user); puts u.email_address }
user1@example.com
user2@example.com
user3@example.com
user4@example.com
user5@example.com
```

ตัวเลข `n` เพิ่มขึ้นทุกครั้งที่ **นิยาม** attribute นี้ถูกเรียก ไม่ว่าจะผ่าน `build`,
`create`, หรือ `build_stubbed` — และตัวเลขนี้เป็น **ตัวนับระดับ process** (ไม่ผูกกับ
ฐานข้อมูล) เริ่มนับจาก 1 ใหม่ทุกครั้งที่ boot Ruby process ขึ้นมาใหม่ (เช่น รัน
`bundle exec rspec` รอบใหม่ หรือเปิด console ใหม่)

### ข้อควรระวังจากประสบการณ์จริง: sequence ชนกับข้อมูลค้างในฐานข้อมูล test

ระหว่างทดสอบเนื้อหา Part นี้ เราเจอเหตุการณ์จริงที่คุ้มค่ามาก: หลังรัน benchmark
script ใน Step 463 (ซึ่งเรียก `FactoryBot.create(:user)` ตรงๆ ผ่าน `ruby script/bench.rb`
**นอก** RSpec — จึงไม่ได้อยู่ใน transaction ที่ rollback อัตโนมัติแบบที่
`config.use_transactional_fixtures = true` ทำให้ตอนรันผ่าน `rspec`) ข้อมูล 200 แถวที่
สร้างขึ้นถูก **commit ค้างอยู่จริง** ในไฟล์ `test.sqlite3` พอกลับมารัน spec อื่นต่อ
(เช่น spec ที่ทดสอบ trait ใน Step 466) กลับเจอ:

```
ActiveRecord::RecordInvalid:
  Validation failed: Email address has already been taken
```

ทั้งที่ sequence เพิ่งเริ่มนับจาก 1 ใหม่ในกระบวนการ (process) ของ `rspec` รอบนั้น —
สาเหตุคือ `"user1@example.com"` ที่ sequence สร้างขึ้นรอบใหม่ **ชนกับแถวที่ค้างอยู่จริง**
จาก benchmark script ก่อนหน้า ตรวจสอบด้วย:

```bash
RAILS_ENV=test bundle exec rails runner 'puts User.count; puts User.pluck(:email_address).first(5)'
```

```
200
user100@example.com
user101@example.com
user102@example.com
user103@example.com
user104@example.com
```

แก้ไขด้วยการล้างตาราง (เรียงลำดับตาม foreign key ให้ถูก — comments ก่อน แล้ว posts
แล้วค่อย users):

```bash
RAILS_ENV=test bundle exec rails runner 'Comment.delete_all; Post.delete_all; User.delete_all'
```

**บทเรียนจากเหตุการณ์นี้:** `sequence` รับประกัน uniqueness ได้แค่ **ภายใน process
เดียวกัน** เท่านั้น มันไม่รู้จักและไม่เช็คกับข้อมูลที่มีอยู่จริงในฐานข้อมูล ถ้ามีอะไร
ก็ตามที่เขียนข้อมูลลง test database แบบไม่อยู่ใน transaction ที่ rollback (เช่น
รัน `rails runner`/script ตรงๆ, หรือลืม `config.use_transactional_fixtures = true`)
ข้อมูลนั้นจะค้างอยู่ข้ามการรัน spec ครั้งต่อๆ ไป และอาจชนกับค่าที่ sequence สร้างขึ้นใหม่
ได้เสมอ — เป็นเหตุผลสำคัญข้อหนึ่งที่ต้องปล่อยให้ `use_transactional_fixtures` เปิดอยู่
เสมอ (ค่า default ของ `rspec-rails` ตั้งแต่ Part 046) และหลีกเลี่ยงการรันสคริปต์ที่
เขียนข้อมูลจริงใส่ `RAILS_ENV=test` โดยไม่ได้ตั้งใจ

### `sequence` กับข้อมูลชนิดอื่นที่ไม่ใช่ string

`sequence` ใช้ได้กับทุก attribute ที่ต้องการค่าไม่ซ้ำ ไม่จำกัดแค่ email:

```ruby
factory :post do
  sequence(:title) { |n| "บทความที่ #{n}" }
  # ...
end
```

หรือใช้ตัวเลขที่ไม่ต้องผ่าน block ก็ได้ถ้าไม่ต้องปรับ format:

```ruby
sequence(:view_count)   # 1, 2, 3, ...
```

---

## Step 465: Associations ใน factory — implicit vs explicit, การสร้าง object graph

### Explicit association — `association :user`

จาก Step 463 เราแก้ stub `user { nil }` เป็น:

```ruby
factory :post do
  # ...
  association :user
end
```

`association :user` บอก FactoryBot ว่า attribute `user` ของ `Post` ต้องสร้างขึ้นจาก
**factory ที่ชื่อ `:user`** (ค้นหา factory ตามชื่อที่ระบุ ไม่ได้อิงจากชื่อ column หรือ
class อัตโนมัติ) ทุกครั้งที่ `create(:post)` ถูกเรียก FactoryBot จะสร้าง `User` ใหม่ 1
ตัวไปด้วย (ด้วย strategy เดียวกับที่ post ใช้ — ถ้า `create(:post)` ก็จะ `create` user
ให้ด้วย ถ้า `build_stubbed(:post)` ก็จะ `build_stubbed` user ให้ด้วยเช่นกัน ไม่แตะ DB
เลยทั้งคู่)

### Implicit association — เขียนแค่ชื่อ attribute เฉยๆ

FactoryBot มี shorthand ที่สั้นกว่านั้นอีก: ถ้าชื่อ attribute **ตรงกับชื่อ factory ที่
มีอยู่แล้วพอดี** เขียนแค่ชื่อ attribute เปล่าๆ (ไม่ต้องมี `association` หรือ `{ }`)
ก็ได้ผลลัพธ์เดียวกัน:

```ruby
factory :implicit_post, class: "Post" do
  title { "implicit test" }
  body { "body" }
  user   # <-- ไม่ต้องเขียน association :user หรือ user { create(:user) }
end
```

ทดสอบจริง:

```ruby
p = FactoryBot.create(:implicit_post)
puts "user_id: #{p.user_id}, user class: #{p.user.class}, user email: #{p.user.email_address}"
```

```
user_id: 551, user class: User, user email: user1@example.com
```

ยืนยันว่า FactoryBot สร้าง `User` จริงให้อัตโนมัติจากบรรทัด `user` เฉยๆ — นี่คือสิ่งที่
เรียกว่า **implicit association** ทั้งสองแบบ (`association :user` และ `user` เปล่าๆ)
ให้ผลเหมือนกันทุกประการเมื่อชื่อ attribute กับชื่อ factory ตรงกัน ต่างกันแค่ว่า
`association :user` **อ่านง่ายกว่าเมื่อสแกนไฟล์ factory ยาวๆ** เพราะเห็นชัดว่าเป็น
ความสัมพันธ์ ไม่ใช่ attribute ธรรมดา — หลักสูตรนี้แนะนำให้เขียน `association :user`
แบบ explicit เสมอด้วยเหตุผลด้าน readability นี้ ยกเว้นในไฟล์ factory สั้นๆ ที่ไม่ซับซ้อน

### association ที่ต้องระบุ factory คนละชื่อกับ attribute

ถ้าชื่อ attribute ไม่ตรงกับชื่อ factory (เช่น `Post` มี `reviewer` ที่จริงๆ คือ `User`
อีกคนหนึ่ง) ต้องระบุ `factory:` ให้ชัดเจน:

```ruby
factory :post do
  association :reviewer, factory: :user
end
```

### สร้าง object graph หลายชั้นด้วย `association` ซ้อนกัน

`Comment` ต้องมีทั้ง `post` และ `user`:

```ruby
factory :comment do
  body { "ความเห็นทดสอบ" }
  association :post
  association :user
end
```

เรียก `create(:comment)` ครั้งเดียว FactoryBot จะไล่สร้างให้ครบทั้ง graph:

```
Comment
 ├── post (สร้างใหม่ผ่าน factory :post)
 │    └── user (สร้างใหม่ผ่าน factory :user ของ post)
 └── user (สร้างใหม่ผ่าน factory :user ของ comment เอง — คนละคนกับ user ของ post)
```

สังเกตว่า **`user` ของ `post` กับ `user` ของ `comment` เป็นคนละ record กัน** (สร้างแยก
2 ครั้ง) ถ้าต้องการให้ comment กับ post ของมันเป็นของ user คนเดียวกัน (เช่น ทดสอบว่า
"ผู้เขียนคอมเมนต์บนโพสต์ตัวเอง") ต้อง override ตอนเรียกใช้แทน (จะสอนใน Step 467):

```ruby
author = create(:user)
post = create(:post, user: author)
comment = create(:comment, post: post, user: author)  # คนเดียวกัน
```

---

## Step 466: Traits — สร้างข้อมูลหลายแบบโดยไม่ต้องซ้ำ factory

### ปัญหาที่ trait แก้: ข้อมูล "หลายแบบ" ของ model เดียวกัน

`Post` อาจอยู่ได้หลายสถานะ: ร่าง (draft), เผยแพร่แล้ว (published) — ถ้าเขียนแยกเป็น
factory คนละตัว (`:draft_post`, `:published_post`) จะเกิดโค้ดซ้ำซ้อนทันทีเพราะทั้งคู่
ใช้ attribute ส่วนใหญ่ร่วมกัน (`title`, `body`, `user`) ต่างกันแค่ `published` กับ
`published_on` **Trait** คือทางแก้: นิยาม "ส่วนต่าง" ไว้แยกจาก factory หลัก แล้วเลือก
เปิดใช้ตอนเรียกก็พอ

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { "บทความทดสอบ" }
    body { "เนื้อหาบทความทดสอบ" }
    published { false }
    published_on { nil }
    view_count { 0 }
    association :user

    trait :published do
      published { true }
      published_on { Date.today }
    end
  end
end
```

ใช้งาน — ส่งชื่อ trait เป็น symbol ต่อท้ายชื่อ factory:

```ruby
require "rails_helper"

RSpec.describe "Post factory traits", type: :model do
  it "default post is not published" do
    post = create(:post)
    expect(post.published).to eq(false)
    expect(post.published_on).to be_nil
  end

  it ":published trait sets published flag and date" do
    post = create(:post, :published)
    expect(post.published).to eq(true)
    expect(post.published_on).to eq(Date.today)
  end
end
```

```bash
bundle exec rspec spec/models/post_traits_spec.rb
```

```
..

Finished in 0.03177 seconds (files took 0.759 seconds to load)
2 examples, 0 failures
```

`create(:post)` ยังคงได้ post ร่างตามปกติ (ค่า default ไม่เปลี่ยน) ส่วน
`create(:post, :published)` ได้ post ที่เผยแพร่แล้วโดยไม่ต้องเขียน factory ใหม่เลย —
attribute ทุกตัวที่ **ไม่ได้ override** ใน trait ยังคงใช้ค่า default จาก factory หลัก

### รวมหลาย trait พร้อมกัน

ส่ง symbol หลายตัวได้ในคำสั่งเดียว:

```ruby
post = create(:post, :published, :with_comments, title: "รวมหลาย trait")
```

(`:with_comments` จะนิยามใน Step 467 ด้วย callback) — FactoryBot จะไล่ apply
ทุก trait ตามลำดับที่ระบุ แล้วค่อย apply ค่าที่ override ตรงๆ (`title:`) ทับบนสุดอีกที
ทดสอบจริง:

```ruby
it "traits can be combined" do
  post = create(:post, :published, :with_comments, title: "รวมหลาย trait")
  expect(post.published).to eq(true)
  expect(post.title).to eq("รวมหลาย trait")
  expect(post.comments.count).to eq(3)
end
```

```
.

Finished in 0.03498 seconds
1 example, 0 failures
```

> **หลักการเลือกใช้ trait:** ใช้ trait เมื่อ "ความต่าง" ของข้อมูลเป็นแค่บาง attribute
> (สถานะ, flag, ความสัมพันธ์เพิ่มเติม) แต่ถ้าข้อมูล 2 แบบไม่มีอะไรเหมือนกันเลย (คนละ
> class, คนละบริบทการใช้งาน) การสร้างเป็นคนละ factory แยกกันชัดเจนกว่ายังคงเป็นทางเลือกที่
> เหมาะสมกว่า trait

---

## Step 467: Factory callbacks (`after(:create)`) และการ override attribute ตอนเรียกใช้งาน

### Override attribute ตอนเรียกใช้ — สิ่งที่ใช้บ่อยที่สุดในชีวิตจริง

ทุก attribute ที่นิยามไว้ใน factory สามารถ **override** ได้ตรงๆ ตอนเรียก `create`/
`build`/`build_stubbed` โดยส่งเป็น keyword argument:

```ruby
it "overriding an attribute at call time wins over the factory default" do
  post = create(:post, title: "หัวข้อกำหนดเอง")
  expect(post.title).to eq("หัวข้อกำหนดเอง")
end
```

```
.
1 example, 0 failures
```

ค่าที่ override ตอนเรียกจะ **ชนะเสมอ** ไม่ว่า attribute นั้นจะมาจาก factory หลักหรือ
จาก trait ก็ตาม — นี่คือเหตุผลที่ spec ส่วนใหญ่ในโปรเจกต์จริงเขียนแบบ
`create(:post, title: "ชื่อที่ spec นี้ต้องการเจาะจงทดสอบ")` แทนที่จะพึ่งค่า default
ล้วนๆ เพราะทำให้อ่าน spec แล้วเห็นทันทีว่า "ค่าที่สำคัญต่อการทดสอบนี้คืออะไร" โดยไม่ต้อง
เปิดไปดูไฟล์ factory ประกอบ

### Factory callback — `after(:create)`

บางครั้งข้อมูลที่ต้องการไม่ใช่แค่ attribute ของ record เดียว แต่เป็น **record ที่
เกี่ยวข้องกัน** ที่ต้องสร้างขึ้น **หลังจาก** record หลักถูกบันทึกแล้ว (เพราะต้องใช้ id
ของมัน) — FactoryBot มี callback hook ให้ 4 จุด: `after(:build)`, `before(:create)`,
`after(:create)`, `after(:stub)` ที่ใช้บ่อยที่สุดคือ `after(:create)`:

```ruby
# spec/factories/posts.rb
trait :with_comments do
  after(:create) do |post|
    create_list(:comment, 3, post: post)
  end
end
```

`create_list(:comment, 3, post: post)` คือ shorthand ของ FactoryBot สำหรับเรียก
`create(:comment, post: post)` 3 รอบ แล้วคืน Array ของทั้ง 3 comment กลับมา ทดสอบจริง:

```ruby
it ":with_comments trait creates associated comments via after(:create) callback" do
  post = create(:post, :with_comments)
  expect(post.comments.count).to eq(3)
end
```

```
.
1 example, 0 failures
```

`create(:post, :with_comments)` ทำ 2 อย่างเรียงกัน: (1) สร้าง `Post` ตามปกติผ่าน
`create` (2) พอ `Post` ถูกบันทึกเสร็จแล้ว (มี `id` แล้ว) callback `after(:create)`
ทำงานทันที สร้าง `Comment` 3 ตัวที่ผูกกับ `post` นั้น — ลำดับนี้สำคัญมาก: **ต้องรอให้
post มี id ก่อน** comment ถึงจะมี `post_id` ให้ผูกได้ (`before(:create)` ใช้ไม่ได้กับ
กรณีนี้เพราะตอนนั้น post ยังไม่มี id)

### เปรียบเทียบ trait ที่ตั้งแค่ attribute กับ trait ที่มี callback

| | `trait :published` | `trait :with_comments` |
|---|---|---|
| ทำอะไร | ตั้งค่า attribute ของ post เอง | สร้าง record อื่น (comment) ที่เกี่ยวข้อง |
| ทำงานตอนไหน | ตอนประกอบ object (ก่อน save) | หลัง `create` เสร็จแล้ว (`after(:create)`) |
| ใช้ได้กับ `build`/`build_stubbed` ไหม | ได้ปกติ | **callback `after(:create)` จะไม่ทำงานถ้าใช้ `build`** (เพราะไม่มีการ create จริง) — ต้องใช้ `create(:post, :with_comments)` เท่านั้น |

---

## Step 468: ผสาน Faker เข้ากับ factory เพื่อข้อมูลสมจริง (รวม locale ภาษาไทย)

### ทำไมต้องใช้ Faker แทนข้อความ static

Factory ที่เขียนมาถึงตอนนี้ใช้ค่าคงที่ เช่น `title { "บทความทดสอบ" }` — ใช้ได้ในระดับ
หนึ่ง แต่มีข้อเสียเมื่อ suite ใหญ่ขึ้น: record ทุกตัวมีข้อความเหมือนกันหมด ทำให้ debug
ยาก (เห็น `"บทความทดสอบ"` 500 แถวในหน้าจอ ไม่รู้ว่าแถวไหนคือแถวที่ spec กำลังทดสอบอยู่)
และไม่ได้ช่วยจับบั๊กที่เกี่ยวกับความยาวข้อความหรือ edge case ของข้อมูลจริง **Faker**
คือ gem ที่สร้างข้อมูลสุ่มแต่ดูสมจริง (ชื่อคน, อีเมล, ประโยค, ที่อยู่ ฯลฯ)

### ทดลองใช้ Faker เปล่าๆ ก่อน

```bash
bundle exec ruby -e '
require "faker"
puts Faker::Lorem.sentence
puts Faker::Internet.email
puts Faker::Name.name
'
```

ผลลัพธ์จริง:

```
Consequatur occaecati et voluptatem.
laveta_larkin@labadie-mueller.test
Santiago Collins
```

สังเกตว่า default locale ของ Faker คือ `:en` — `Faker::Lorem` สร้างประโยคภาษาละติน
แบบ "lorem ipsum" (ไม่มีความหมายจริง แต่รูปแบบเป็นประโยคสมจริง), `Faker::Internet.email`
สร้างอีเมลปลอมที่ format ถูกต้อง, `Faker::Name.name` สร้างชื่อฝรั่ง

### ผสาน Faker เข้ากับ factory

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email_address) { |n| "user#{n}@example.com" }
    name { Faker::Name.name }
    password { "secret123" }
    password_confirmation { "secret123" }
    phone_number { "0800000000" }
    admin { false }

    trait :admin do
      admin { true }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { Faker::Lorem.sentence(word_count: 5) }
    body { Faker::Lorem.paragraph(sentence_count: 5) }
    published { false }
    published_on { nil }
    view_count { 0 }
    association :user
    # ... traits เดิมจาก Step 466–467
  end
end
```

ทดสอบผลลัพธ์จริง:

```ruby
u = FactoryBot.build(:user)
puts "name: #{u.name}, email: #{u.email_address}"

p2 = FactoryBot.build(:post)
puts "title: #{p2.title}"
puts "body: #{p2.body}"
```

```
name: Derrick Graham, email: user1@example.com
title: Illum qui cum tenetur et.
body: Nulla deleniti amet. Voluptatem ullam omnis. Quasi corrupti nemo. Possimus deleniti occaecati. Nobis omnis repellat.
```

รัน spec เดิมทั้งหมดอีกครั้งเพื่อยืนยันว่าไม่มีอะไรพัง (Faker ไม่ได้ทำให้ validation
หรือ uniqueness เพี้ยน เพราะ `email_address` ยังคุมด้วย `sequence` เหมือนเดิม —
`name`/`title`/`body` เปลี่ยนแค่ "หน้าตา" ของข้อมูล ไม่กระทบ constraint ใดๆ):

```bash
bundle exec rspec spec/models
```

```
..............

Finished in 0.2107 seconds (files took 0.7068 seconds to load)
14 examples, 0 failures
```

> **ข้อควรระวัง:** อย่าใช้ `Faker::Internet.email` (หรือ `.unique.email`) แทน
> `sequence(:email_address)` เพื่อความ unique เพราะ Faker เป็นการสุ่มล้วนๆ ไม่รับประกัน
> ว่าจะไม่ซ้ำ (แม้ `Faker::Internet.unique.email` จะช่วยได้ระดับหนึ่ง แต่ตัวนับ unique
> ของ Faker เองก็ต้องคอย reset ระหว่าง spec คล้ายปัญหาที่เจอใน Step 464 ทำให้จัดการ
> ยากกว่า `sequence` ของ FactoryBot ที่ออกแบบมาเพื่องานนี้โดยเฉพาะ) หลักปฏิบัติที่ดี
> คือ **ใช้ `sequence` คุม uniqueness เสมอ แล้วใช้ Faker แค่แต่งหน้าตาข้อมูลที่ไม่ต้อง
> unique** เช่น `name`, `title`, `body`

### Faker locale ภาษาไทย — ทดสอบจริงว่าครอบคลุมแค่ไหน

Faker รองรับหลาย locale ผ่าน `Faker::Config.locale =` ลองสลับเป็นภาษาไทยดู:

```bash
bundle exec ruby -e '
require "faker"
Faker::Config.locale = :th
puts Faker::Name.name
puts Faker::Address.city
'
```

ผลลัพธ์จริง:

```
สุกัญญา ติณสูลานนท์
ประสานburgh
```

`Faker::Name.name` ได้ชื่อไทยที่ถูกต้องสมบูรณ์ แต่ `Faker::Address.city` ได้ผลลัพธ์
**ผสมภาษาที่ผิดปกติ** (`"ประสาน" + "burgh"`) ตรวจสอบไฟล์ locale ของ gem โดยตรงถึงเหตุผล:

```bash
grep -E "^    [a-z_]+:$" $(gem contents faker | grep locales/th.yml)
```

```
    name:
```

**ไฟล์ locale ภาษาไทยของ `faker` (เวอร์ชัน 3.8.0) นิยามข้อมูลไว้แค่หมวด `name`
เท่านั้น** (380 บรรทัด มีแค่ `first_name`, `last_name` ฯลฯ) หมวดอื่นทั้งหมด (Address,
Lorem, Internet, Company, ฯลฯ) **ไม่มี** คำแปลภาษาไทย — เมื่อเรียก
`Faker::Address.city` ตอน locale เป็น `:th` มันจะ fallback ไปหาคำภาษาอังกฤษบางส่วน
ผสมกับ pattern ที่เหลือ ได้ผลลัพธ์แปลกๆ แบบที่เห็น

**ข้อสรุปที่ใช้งานได้จริง:** ถ้าต้องการชื่อคนไทยสมจริงในข้อมูลทดสอบ (เช่น
สำหรับ demo หรือ screenshot ของแอปที่ใช้ภาษาไทยทั้งหน้า) ใช้
`Faker::Config.locale = :th` ได้อย่างปลอดภัย **เฉพาะกับ `Faker::Name`** เท่านั้น
ส่วน field อื่นที่ไม่ต้องการภาษาไทยจริงจัง (เช่น `title`, `body` ของบทความทดสอบที่
เนื้อหาจริงไม่สำคัญ) ปล่อยเป็นภาษาอังกฤษ (`Faker::Lorem`) ตามปกติได้ เพราะไม่กระทบ
ผลการทดสอบ:

```ruby
factory :user do
  sequence(:email_address) { |n| "user#{n}@example.com" }
  name { Faker::Config.locale = :th; Faker::Name.name }
  # ...
end
```

> **ข้อควรระวังเรื่อง global state:** `Faker::Config.locale =` เป็นการตั้งค่า
> **ระดับ process** ไม่ใช่ระดับ call เดียว ถ้าตั้งไว้ใน factory ตัวหนึ่งแล้วไม่รีเซ็ต
> factory อื่นที่เรียกหลังจากนั้นในกระบวนการเดียวกันจะได้รับผลกระทบไปด้วย (locale
> ยังเป็น `:th` ค้างอยู่) ถ้าต้องการควบคุมให้ปลอดภัยกว่านี้ ให้ใช้
> `Faker::Name.name` แบบส่ง locale เจาะจงต่อการเรียกแทน (Faker เวอร์ชันใหม่รองรับ
> `Faker::Name.name` ธรรมดาโดยไม่ต้องพึ่ง global config ถ้าตั้ง default_locale ไว้ที่
> `config/initializers/faker.rb` แทนการสลับกลางคันในไฟล์ factory)

---

## Step 469: หลีกเลี่ยง anti-pattern — Lean Factory Principle และ `FactoryBot.lint`

### Anti-pattern: factory ที่ "กระตือรือร้นเกินไป" (eager factory)

ข้อผิดพลาดที่พบบ่อยที่สุดของทีมที่เพิ่งเริ่มใช้ FactoryBot คือการทำให้ factory
**สร้างข้อมูลที่เกี่ยวข้องเยอะเกินความจำเป็นเป็นค่า default** เช่น ทำให้ทุก
`create(:post)` สร้าง comment แถมมาด้วยเสมอ (ไม่ใช่แค่ตอนใช้ trait อย่างที่ทำใน Step
467):

```ruby
# ตัวอย่างของ "eager factory" — ไม่ควรทำแบบนี้
factory :eager_post, class: "Post" do
  title { Faker::Lorem.sentence }
  body { Faker::Lorem.paragraph }
  view_count { 0 }
  association :user
  after(:create) { |post| create_list(:comment, 5, post: post) }  # <-- ทำงานทุกครั้ง ไม่ใช่แค่ตอนใช้ trait
end
```

ปัญหาคือ: spec ส่วนใหญ่ที่เรียก `create(:post)` **ไม่ได้สนใจ comment เลย** (เช่น spec
ที่ทดสอบแค่ validation ของ `title`) แต่ต้องจ่าย "ค่าปรับ" เป็นเวลาที่เสียไปกับการสร้าง
comment 5 ตัวที่ไม่ได้ใช้ทุกครั้งอยู่ดี วัดผลจริงด้วย benchmark:

```ruby
n = 50
Benchmark.bm(24) do |x|
  x.report("lean :post x#{n}")        { n.times { FactoryBot.create(:post) } }
  x.report("eager :eager_post x#{n}") { n.times { FactoryBot.create(:eager_post) } }
end
```

ผลลัพธ์จริง:

```
                               user     system      total        real
lean :post x50             0.477406   0.022214   0.499620 (  0.516120)
eager :eager_post x50      1.150527   0.043727   1.194254 (  1.209105)
```

factory แบบ eager ช้ากว่าแบบ lean **ประมาณ 2.3 เท่า** สำหรับสร้าง `Post` แค่ 50 ตัว —
ในโปรเจกต์จริงที่มี spec หลักพัน example และหลายคนเรียก `create(:post)` โดยไม่รู้ว่า
มันแอบสร้าง comment ให้ด้วย ผลรวมของเวลาที่เสียไปจะมหาศาล และเป็นสิ่งที่ debug ยากมาก
เพราะ "ไม่มี error ใดๆ" — แค่ suite ช้าลงเรื่อยๆ แบบไม่มีสาเหตุชัดเจน

### หลักการ Lean Factory

> **Lean Factory Principle:** factory ควรสร้าง **แค่สิ่งที่จำเป็นต่อการทำให้ record
> valid เท่านั้น** — ความสัมพันธ์หรือข้อมูลเพิ่มเติมที่ไม่ได้จำเป็นต่อความ valid ให้
> ใส่เป็น **trait ที่ต้องเรียกใช้เอง** (`:with_comments` แบบ Step 467) ไม่ใช่ default
> behavior — ให้ spec แต่ละไฟล์เป็นคนตัดสินใจว่าต้องการข้อมูลระดับไหน ไม่ใช่ให้ factory
> ตัดสินใจแทน

เช็คลิสต์สั้นๆ ก่อนเพิ่มอะไรลงใน factory หลัก (ไม่ใช่ trait):

1. **จำเป็นหรือไม่ที่จะทำให้ record valid?** (เช่น `belongs_to` ที่ไม่ใช่ `optional:
   true` ต้องมี — ใส่ในนี้ได้)
2. ถ้าใส่แล้ว spec ส่วนใหญ่ (มากกว่าครึ่ง) ในโปรเจกต์ **ใช้จริง** ไหม ถ้าใช้แค่บาง spec
   ให้แยกเป็น trait
3. การสร้างสิ่งนี้ **แพง** แค่ไหน (ยิ่งมี callback, ยิ่งมี nested association เยอะ ยิ่ง
   ควรระวัง)

### `FactoryBot.lint` — ตรวจสอบว่าทุก factory ยังใช้งานได้จริง

ปัญหาหนึ่งที่เกิดขึ้นได้ในโปรเจกต์จริงคือ: factory ถูกนิยามไว้ แต่ไม่มี spec ไหนเรียก
ใช้ตรงๆ (เช่น factory ที่เตรียมไว้สำหรับ trait ที่ยังไม่มีคนใช้) พอ schema เปลี่ยน
แล้ว factory นั้นพังไปเงียบๆ โดยไม่มี test ไหนจับได้ จนกว่าจะมีคนมาใช้จริงวันหนึ่ง —
วิธีป้องกันคือสั่งให้ FactoryBot **สร้างทุก factory ที่มีจริงสักครั้ง** ก่อนรัน suite
เพื่อยืนยันว่าทุกตัว valid

```ruby
# spec/support/factory_bot.rb
RSpec.configure do |config|
  config.before(:suite) do
    FactoryBot.lint
  end
end
```

และเปิดให้ RSpec โหลดไฟล์ใน `spec/support/` อัตโนมัติ (ค่า default ที่ `rspec:install`
สร้างให้เป็น comment ไว้ — เปิดใช้งานโดยลบ `#` ออก):

```ruby
# spec/rails_helper.rb
Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }
```

รันดู — ถ้าทุก factory ถูกต้อง จะไม่มีอะไรแสดงผลผิดปกติ (ผ่านเงียบๆ):

```bash
bundle exec rspec spec/models/user_spec.rb
```

```
..

Finished in 0.04239 seconds (files took 0.72018 seconds to load)
2 examples, 0 failures
```

ทดลองทำให้ factory พังโดยตั้งใจ (ลบ `body` ของ `:comment` ทิ้ง) เพื่อดู error จริงจาก
`lint`:

```ruby
# spec/factories/comments.rb (จงใจทำให้พัง เพื่อดูผลลัพธ์)
factory :comment do
  body { nil }
  association :post
  association :user
end
```

```bash
bundle exec rspec spec/models/user_spec.rb
```

ผลลัพธ์จริง:

```
An error occurred in a `before(:suite)` hook.
Failure/Error: FactoryBot.lint

FactoryBot::InvalidFactoryError:
  The following factories are invalid:

  * comment - Validation failed: Body can't be blank (ActiveRecord::RecordInvalid)

Finished in 0.15437 seconds (files took 0.85209 seconds to load)
0 examples, 0 failures, 1 error occurred outside of examples
```

**ทั้ง suite หยุดทำงานทันทีตั้งแต่ก่อนรัน example แรก** พร้อมบอกชัดเจนว่า factory
ไหนพังและเพราะอะไร (`Body can't be blank`) — นี่คือประโยชน์ของ `FactoryBot.lint`:
จับปัญหา **ที่ตัว factory เอง** ก่อนที่มันจะไปทำให้ spec ที่ไม่เกี่ยวข้องพังแบบงงๆ
(คล้ายปัญหา cascading failure ที่เจอใน Step 461 แต่คราวนี้ระบบชี้ต้นเหตุให้ตรงจุด
ทันที) แก้ไข factory กลับให้ถูกต้องแล้วยืนยันว่า suite กลับมาผ่านครบ:

```bash
bundle exec rspec spec/models
```

```
..............

Finished in 0.16548 seconds (files took 0.87187 seconds to load)
14 examples, 0 failures
```

> **ข้อควรรู้:** `FactoryBot.lint` เรียก `create` กับทุก factory (และทุก trait) จริง
> ดังนั้นควรรันเฉพาะใน CI หรือก่อน push (ไม่จำเป็นต้องรันทุกครั้งที่ save ไฟล์ระหว่าง
> เขียนโค้ด) เพราะจะเพิ่มเวลาบูตของทุกการรัน `rspec` เล็กน้อย ทีมจำนวนมากแยกมันออกเป็น
> rake task/CI step ต่างหาก (`bundle exec rspec spec/factories_lint_spec.rb` หรือ
> job เฉพาะใน GitHub Actions ที่จะเรียนใน Part 075) แทนที่จะใส่ใน `before(:suite)`
> ของทุกการรันปกติ

---

## Step 470: แบบฝึกหัด — ชุด factory เต็มรูปแบบสำหรับ Post/Comment/User

### โจทย์

รวมทุกเทคนิคจาก Step 461–469 สร้างชุด factory ที่สมบูรณ์สำหรับ domain
`User`/`Post`/`Comment` ที่ต้องมีครบ:

1. `factory :user` — ใช้ `sequence` คุม email, ใช้ Faker สร้างชื่อ, มี `trait :admin`
2. `factory :post` — ใช้ Faker สร้าง title/body, มี `association :user`, มี
   `trait :published` และ `trait :with_comments` (ใช้ `after(:create)`)
3. `factory :comment` — มี `association :post` และ `association :user`
4. เขียน spec ที่ใช้ `create`, `build`, `build_stubbed` ให้ถูกจุดตามลักษณะของแต่ละ
   spec (ไม่ใช่ใช้ `create` ทุกที่)
5. เขียน spec อย่างน้อย 1 คู่ที่ **วัดความเร็วจริง** เปรียบเทียบ `create` กับ
   `build_stubbed` สำหรับการทดสอบ method ที่ไม่แตะฐานข้อมูล
6. ตั้งค่า `FactoryBot.lint` ให้รันก่อน suite

### เฉลย

**Factory ทั้ง 3 ไฟล์ (รวมทุกเทคนิคจาก Step ก่อนหน้า):**

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email_address) { |n| "user#{n}@example.com" }
    name { Faker::Name.name }
    password { "secret123" }
    password_confirmation { "secret123" }
    phone_number { "0800000000" }
    admin { false }

    trait :admin do
      admin { true }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { Faker::Lorem.sentence(word_count: 5) }
    body { Faker::Lorem.paragraph(sentence_count: 5) }
    published { false }
    published_on { nil }
    view_count { 0 }
    association :user

    trait :published do
      published { true }
      published_on { Date.today }
    end

    trait :with_comments do
      after(:create) do |post|
        create_list(:comment, 3, post: post)
      end
    end
  end
end
```

```ruby
# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { "ความเห็นทดสอบ" }
    association :post
    association :user
  end
end
```

**Model เพิ่ม method ที่ไม่แตะฐานข้อมูล (สำหรับทดสอบ `build_stubbed`):**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy

  validates :title, presence: true
  validates :body, presence: true

  def excerpt(length = 20)
    body.to_s.truncate(length)
  end

  def status_label
    published? ? "เผยแพร่แล้ว" : "ร่าง"
  end
end
```

**Spec ที่เลือก strategy ให้ถูกจุด:**

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  # ต้องการ record จริงในฐานข้อมูล (เช็ค persisted?) -> create
  it "is valid with a title and body" do
    post = create(:post)
    expect(post).to be_persisted
  end

  # แค่เช็ค validation ไม่ต้องบันทึกจริง -> build
  it "requires a title" do
    post = build(:post, title: nil)
    expect(post).not_to be_valid
  end

  # ทดสอบ method ล้วนๆ ที่ไม่แตะฐานข้อมูล -> build_stubbed (เร็วที่สุด)
  it "status_label returns ร่าง for an unpublished post" do
    post = build_stubbed(:post)
    expect(post.status_label).to eq("ร่าง")
  end

  it "status_label returns เผยแพร่แล้ว for a published post" do
    post = build_stubbed(:post, :published)
    expect(post.status_label).to eq("เผยแพร่แล้ว")
  end

  it "excerpt truncates the body" do
    post = build_stubbed(:post, body: "a" * 50)
    expect(post.excerpt(10).length).to eq(10)
  end
end
```

```bash
bundle exec rspec spec/models/post_spec.rb
```

```
.....

Finished in 0.06213 seconds (files took 0.7213 seconds to load)
5 examples, 0 failures
```

**วัดความเร็วจริง — `create` เทียบ `build_stubbed` บน 100 example ที่ทดสอบ method
เดียวกัน:**

```ruby
# spec/models/post_speed_create_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  100.times do |i|
    it "computes status_label (create ##{i})" do
      post = create(:post)
      expect(post.status_label).to eq("ร่าง")
    end
  end
end
```

```ruby
# spec/models/post_speed_stub_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  100.times do |i|
    it "computes status_label (build_stubbed ##{i})" do
      post = build_stubbed(:post)
      expect(post.status_label).to eq("ร่าง")
    end
  end
end
```

```bash
bundle exec rspec spec/models/post_speed_create_spec.rb
bundle exec rspec spec/models/post_speed_stub_spec.rb
```

ผลลัพธ์จริง:

```
=== create x100 ===
Finished in 0.70114 seconds (files took 0.73046 seconds to load)
100 examples, 0 failures

=== build_stubbed x100 ===
Finished in 0.54145 seconds (files took 0.82457 seconds to load)
100 examples, 0 failures
```

`build_stubbed` เร็วกว่าประมาณ **23%** สำหรับ 100 example ที่ทดสอบ method เดียวกัน
ทุกประการ (ต่างกันแค่ strategy ที่ใช้สร้างข้อมูล) — ตัวเลขนี้ดูไม่เยอะเพราะ SQLite
ในเครื่องทดสอบเร็วอยู่แล้วและ overhead ของตัว RSpec เองก็มีส่วนกลบตัวเลขบางส่วน แต่ถ้า
วัดแบบ raw loop ไม่ผ่าน RSpec (ตามที่ทำใน Step 463) จะเห็นส่วนต่างชัดกว่ามาก (~2.4
เท่า) — ในโปรเจกต์จริงที่ใช้ PostgreSQL ผ่าน network (ไม่ใช่ SQLite ในเครื่องเดียวกัน)
และมี model ที่มี callback/association ซับซ้อนกว่านี้ ส่วนต่างนี้จะยิ่งขยายใหญ่ขึ้น
ตามสัดส่วน

**ตั้งค่า lint:**

```ruby
# spec/support/factory_bot.rb
RSpec.configure do |config|
  config.before(:suite) do
    FactoryBot.lint
  end
end
```

```bash
bundle exec rspec spec/models
```

```
..............................................................................................................

Finished in ...
214 examples, 0 failures
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม `factory :category` (มี `name`) และแก้ `Post` ให้มี `belongs_to :category`
   (ผ่าน migration จริง) จากนั้นเพิ่ม `trait :uncategorized` ให้ `Post` ที่ทำให้
   `category` เป็น `nil` ได้ (ต้องแก้ association ให้เป็น `optional: true` ก่อน) —
   ฝึกความเข้าใจว่า trait ไม่ได้มีไว้ "เพิ่ม" ข้อมูลเสมอไป ใช้ "ลบ/ปิด" ค่า default
   ก็ได้เช่นกัน
2. เขียน `factory :comment` เวอร์ชันใหม่ที่มี `trait :from_guest` ทำให้ `user` เป็น
   `nil` ได้ (ต้องแก้ model ให้ `belongs_to :user, optional: true` ก่อน) แล้วเพิ่ม
   attribute `guest_name` ให้ trait นี้ตั้งค่า Faker name ให้ — ฝึกออกแบบ trait ที่
   เปลี่ยนทั้ง association และ attribute พร้อมกัน
3. ตั้งเวลาการรัน `FactoryBot.lint` เทียบกับตอนไม่มี `after(:create)` ใดๆ ใน factory
   ทั้งหมด แล้วเทียบกับตอนที่ `:post` มี `trait :with_comments` ที่สร้าง comment 3
   ตัว — วัดผลกระทบของ lint ต่อเวลาบูต suite ทั้งหมดด้วยตัวเอง (ใบ้: `lint` ไม่ได้
   apply trait ให้อัตโนมัติ ค่า default เท่านั้นที่ถูกทดสอบ ลองอ่าน README ของ
   `factory_bot` เรื่อง `FactoryBot.lint(traits: true)` เพื่อทำความเข้าใจว่าทำไม)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เห็นปัญหาจริงของการเขียน `Post.create`/`User.create` ด้วยมือกระจายในหลาย spec —
  การเปลี่ยน schema เพียงเล็กน้อยทำให้เกิด cascading failure ที่ error message ไม่ได้
  ชี้ไปที่ต้นเหตุตรงๆ
- ติดตั้งและใช้งาน `factory_bot_rails` ได้ครบวงจร ตั้งแต่ generator สร้าง stub
  อัตโนมัติ ไปจนถึงการแก้ stub ให้ใช้งานได้จริง
- เข้าใจความต่างของ `create`, `build`, `build_stubbed` ทั้งเชิงพฤติกรรม (DB query,
  `persisted?`, `id`) และเชิงความเร็ว (วัดจริงด้วย benchmark)
- ใช้ `sequence` แก้ปัญหา uniqueness validation พร้อมรู้ข้อจำกัดจริงของมัน (นับระดับ
  process เท่านั้น ไม่รู้จักข้อมูลที่ค้างอยู่ในฐานข้อมูล)
- สร้าง association ใน factory ได้ทั้งแบบ implicit และ explicit เข้าใจว่า FactoryBot
  ไล่สร้าง object graph อย่างไร
- ใช้ trait สร้างข้อมูลหลายสถานะโดยไม่ซ้ำ factory และผสาน callback (`after(:create)`)
  เพื่อสร้าง record ที่เกี่ยวข้องอัตโนมัติ
- ผสาน Faker เข้ากับ factory เพื่อข้อมูลสมจริง พร้อมรู้ข้อจำกัดจริงของ locale ภาษาไทย
  (ครอบคลุมแค่ `Faker::Name`)
- รู้จัก anti-pattern ของ "eager factory" และหลักการ Lean Factory พร้อมตัวเลข
  benchmark จริงที่ยืนยันผลกระทบด้านความเร็ว
- ตั้งค่าและทดสอบ `FactoryBot.lint` เพื่อจับ factory ที่พังตั้งแต่ก่อนมันไปทำให้ spec
  อื่นพังแบบงงๆ

**ต่อไป (Part 048):** เราจะขยับจาก model spec ไปสู่ **Feature test / System test ด้วย
Capybara** — การทดสอบแอปทั้งระบบผ่าน browser จำลอง (คลิกปุ่ม, กรอกฟอร์ม, เห็นข้อความ
บนหน้าเว็บจริง) โดยจะใช้ factory ที่สร้างไว้ใน Part นี้เป็นข้อมูลตั้งต้นของทุก system
spec ทันที — เห็นได้ชัดว่าการลงทุนเขียน factory ให้ดีตั้งแต่ต้นใน Part นี้จะส่งผลดีไป
ตลอดทั้งหลักสูตรที่เหลือ
