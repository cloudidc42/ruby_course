# Part 032: Validation ขั้นสูง — Custom Validator, Conditional Validation, Uniqueness with Scope

> **Step ครอบคลุมใน Part นี้:** Step 311–320
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 026 มาก่อน โดยเฉพาะเรื่อง `validates`, `validate`,
> `errors.add`, `if:`/`unless:`, และ `on:` — Part นี้จะไม่สอนพื้นฐานเหล่านั้นซ้ำ)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

Part 026 พาไปรู้จัก validator สำเร็จรูปของ Rails (`presence`, `length`, `numericality`,
`uniqueness`, `format`, `inclusion`/`exclusion`) และวิธีเขียน custom validation method ด้วย
`validate :method_name` — เพียงพอสำหรับกฎทางธุรกิจส่วนใหญ่ที่ผูกอยู่กับ Model เดียว แต่เมื่อแอป
โตขึ้น จะเจอโจทย์ที่ built-in validator และ custom method แบบเดิมเริ่มไม่พอ เช่น:

- กฎเดียวกัน (เช่น "ต้องเป็นอีเมลที่ถูกต้อง") ต้องใช้ซ้ำในหลาย Model — ก็อปวางโค้ดเดิมซ้ำๆ
  ทุกที่ไม่ใช่ทางออกที่ดี
- "slug ต้องไม่ซ้ำ" จริงๆ แล้วหมายถึง "ไม่ซ้ำ**ภายในหมวดหมู่เดียวกัน**" ไม่ใช่ไม่ซ้ำทั้งตาราง
- `uniqueness` validator ที่เรียนใน Part 026 มีคำเตือนทิ้งไว้ว่าเกิด **race condition** ได้ในทาง
  ทฤษฎี — Part นี้จะพิสูจน์ให้เห็นจริงพร้อมวิธีแก้ที่ถูกต้อง
- Object บางตัวไม่ใช่ ActiveRecord เลย (เช่น form ที่รวมข้อมูลจากหลายแหล่ง) แต่ก็อยากได้ระบบ
  validation แบบเดียวกับที่ใช้ใน Model
- Error message ที่เห็นตอนนี้เป็นภาษาอังกฤษล้วน ("can't be blank") ทั้งที่แอปเป็นภาษาไทย

Part นี้จะพาไปแก้ปัญหาทั้งหมดนี้ทีละข้อ ด้วยเครื่องมือระดับ `ActiveModel::Validations` ที่อยู่
เบื้องหลัง `validates`/`validate` มาตั้งแต่ต้น

## สารบัญของ Part นี้

- Step 311: สร้าง Custom Validator แบบใช้ซ้ำได้ด้วย `ActiveModel::EachValidator`
- Step 312: ใส่ Option ให้ Custom Validator และใช้ซ้ำข้ามหลาย Model/หลาย Field
- Step 313: Cross-field Validation แบบง่ายด้วย `validate` และ `errors.add(:base, ...)`
- Step 314: ยกระดับกฎข้าม Field ให้ใช้ซ้ำได้ด้วย `ActiveModel::Validator`
- Step 315: `uniqueness: { scope: ... }` — ไม่ซ้ำแบบมีเงื่อนไข (Composite Uniqueness)
- Step 316: พิสูจน์ Race Condition ของ `uniqueness` Validator ด้วยของจริง
- Step 317: ทางแก้ที่แท้จริง — Unique Index ระดับฐานข้อมูล + `rescue RecordNotUnique`
- Step 318: `validates_associated` และกับดัก Infinite Validation Loop
- Step 319: `ActiveModel::Validations` แบบ Standalone บน Plain Old Ruby Object (Form Object)
- Step 320: ปรับแต่งข้อความ Error — `message:`, I18n, และ `errors.add(:base)` เทียบกับ Field-specific

---

## Step 311: สร้าง Custom Validator แบบใช้ซ้ำได้ด้วย `ActiveModel::EachValidator`

### ทบทวนข้อจำกัดของ custom validation method

จาก Part 026 Step 254 เราเขียน custom validation ได้แบบนี้:

```ruby
class Subscriber < ApplicationRecord
  validate :email_format_is_valid

  private

  def email_format_is_valid
    return if email.blank?

    unless email.match?(/\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i)
      errors.add(:email, "ไม่ใช่รูปแบบอีเมลที่ถูกต้อง")
    end
  end
end
```

โค้ดนี้ใช้งานได้ดีถ้ามีแค่ Model เดียวที่ต้องเช็คอีเมล แต่พอมี Model ที่สองที่ต้องเช็คอีเมล
เหมือนกัน (เช่น `Post` ที่มี field `email` ของผู้เขียนรับเชิญ, `Order` ที่มี `customer_email`)
เราจะต้อง **ก็อปวาง private method เดิมซ้ำทุก Model** — ผิดหลัก DRY (Don't Repeat Yourself)
ที่เรียนมาตั้งแต่ Part 001 และถ้าวันหนึ่งต้องแก้ regex ให้รองรับรูปแบบใหม่ ต้องไล่แก้ทุก Model
ที่ก็อปโค้ดนี้ไปด้วยตัวเอง

### `ActiveModel::EachValidator` — ฐานของ validator เกือบทุกตัวที่ Rails มีให้

ความจริงแล้ว `presence`, `length`, `numericality` ที่ใช้มาตั้งแต่ Part 026 ก็คือ class ที่สืบทอด
มาจาก `ActiveModel::EachValidator` ทั้งนั้น (เช่น `presence` มาจาก class
`ActiveModel::Validations::PresenceValidator`) — `validates :field, xxx: true` เป็นแค่ syntax
สะดวกที่ Rails แปลงเป็นการเรียก validator class ที่ชื่อ `XxxValidator` ให้อัตโนมัติ

เราเขียน validator class ของตัวเองแบบเดียวกันได้ โดยสืบทอดจาก `ActiveModel::EachValidator`
แล้ว override method `validate_each`:

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  FORMAT = /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i

  def validate_each(record, attribute, value)
    return if value.blank?

    record.errors.add(attribute, "ไม่ใช่รูปแบบอีเมลที่ถูกต้อง") unless value.match?(FORMAT)
  end
end
```

**สิ่งสำคัญที่ต้องรู้:**

- ไฟล์ต้องอยู่ใน `app/validators/` (Rails autoload โฟลเดอร์นี้ให้อัตโนมัติ ไม่ต้อง
  `require` เอง) และชื่อไฟล์/class ต้องเป็น `snake_case`/`PascalCase` ที่ตรงกันตามธรรมเนียม
  Rails ทั่วไป (`email_validator.rb` → `EmailValidator`)
- ชื่อ class **ต้องลงท้ายด้วย `Validator`** เสมอ — Rails ใช้ชื่อนี้ในการ map จากคำสั่ง
  `validates :email, email: true` ไปหา class `EmailValidator` โดยอัตโนมัติ (ตัดคำว่า
  `Validator` ออก เหลือ `email` แล้วแปลงเป็น key)
- `validate_each(record, attribute, value)` ถูกเรียก**หนึ่งครั้งต่อหนึ่ง attribute** ที่ประกาศ
  ใช้ validator นี้ — `record` คือ instance ทั้งก้อน (เผื่อต้องอ้างอิง field อื่นประกอบ),
  `attribute` คือชื่อ field ปัจจุบัน (symbol), `value` คือค่าปัจจุบันของ field นั้น
- เรียก `record.errors.add(attribute, "...")` เพื่อรายงาน error เหมือนที่เคยเรียก
  `errors.add` ใน custom validation method ปกติ — ต่างกันแค่ตรงนี้ต้องระบุ `record.` นำหน้า
  เพราะ validator class ไม่ได้เป็น instance ของ Model เอง

### ใช้งาน — สั้นกว่าเดิมมาก และ reuse ได้ทันที

```ruby
# app/models/subscriber.rb
class Subscriber < ApplicationRecord
  validates :email, presence: true, email: true
end
```

สังเกตว่า `validates :email, email: true` เขียนสั้นพอๆ กับ `presence: true` ทั้งที่เบื้องหลัง
มันไปเรียก class `EmailValidator` ที่เราเพิ่งเขียนเอง — **จากมุมมองคนอ่านโค้ด แยกไม่ออกเลยว่า
validator ไหนเป็นของ Rails เอง กับ validator ไหนเป็น custom ที่ทีมเขียนเพิ่ม** ซึ่งเป็นข้อดี
ของแนวทางนี้: API หน้าตาเดียวกันทั้งหมด

ทดสอบผ่าน `bin/rails runner` (หรือ `rails console`):

```irb
irb(main):001> Subscriber.new(email: nil).tap(&:valid?).errors.full_messages
=> ["Email can't be blank"]
irb(main):002> Subscriber.new(email: "not-an-email").tap(&:valid?).errors.full_messages
=> ["Email ไม่ใช่รูปแบบอีเมลที่ถูกต้อง"]
irb(main):003> Subscriber.new(email: "user@example.com").tap(&:valid?).errors.full_messages
=> []
```

> **เมื่อไหร่ควรยกระดับจาก `validate :method_name` เป็น custom validator class?** กฎเดียวกัน
> ถูกใช้ (หรือมีแนวโน้มจะถูกใช้) ในมากกว่า 1 Model, หรือกฎนั้นซับซ้อนพอที่การแยกเป็น class
> ของตัวเองทำให้อ่านง่ายขึ้นและเทสแยกได้อิสระจาก Model ใดๆ ถ้ากฎใช้ที่เดียวและไม่ซับซ้อน
> `validate :method_name` ตามที่เรียนใน Part 026 ยังคงเป็นทางเลือกที่เหมาะสมกว่า — ไม่ต้อง
> ยกระดับทุกอย่างเป็น validator class เสมอไป

---

## Step 312: ใส่ Option ให้ Custom Validator และใช้ซ้ำข้ามหลาย Model/หลาย Field

### รับ option ผ่าน `options` hash

`ActiveModel::EachValidator` มี method `options` ให้ใช้ภายใน `validate_each` เสมอ — เป็น Hash
ของทุก option ที่ส่งเข้ามาตอนเรียกใช้ validator (ยกเว้น key พิเศษอย่าง `:if`/`:unless`/`:on`
ที่ ActiveModel จัดการเองอยู่แล้ว) ทำให้ validator หนึ่งตัวปรับพฤติกรรมได้ตามแต่ละที่ที่เรียกใช้:

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  FORMAT = /\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i

  def validate_each(record, attribute, value)
    return if value.blank?

    unless value.match?(FORMAT)
      record.errors.add(attribute, options[:message] || "ไม่ใช่รูปแบบอีเมลที่ถูกต้อง")
      return
    end

    if options[:domain] && !value.end_with?("@#{options[:domain]}")
      record.errors.add(attribute, "ต้องเป็นอีเมลของโดเมน #{options[:domain]} เท่านั้น")
    end
  end
end
```

เพิ่ม option สองตัว: `message:` (override ข้อความ error เริ่มต้นของ validator ตัวเองได้) และ
`domain:` (option ที่เราออกแบบเอง ไม่ใช่ของ Rails — บังคับว่าอีเมลต้องเป็นของโดเมนที่กำหนด)

### ใช้ validator เดียวกันกับหลาย Model พร้อมคนละ option

```ruby
# app/models/subscriber.rb
class Subscriber < ApplicationRecord
  validates :email, presence: true, email: true   # ทั่วไป ไม่จำกัดโดเมน
end
```

```ruby
# app/models/post.rb — ผู้เขียนรับเชิญต้องใช้อีเมลของบริษัทเท่านั้น
class Post < ApplicationRecord
  belongs_to :category

  validates :email, email: { domain: "company.com" }, allow_blank: true
end
```

```irb
irb(main):001> tech = Category.create!(name: "เทคโนโลยี", slug: "tech")

irb(main):002> Post.new(title: "t", body: "b", category: tech, email: "user@gmail.com")
irb(main):003>   .tap(&:valid?).errors.full_messages
=> ["Email ต้องเป็นอีเมลของโดเมน company.com เท่านั้น"]

irb(main):004> Post.new(title: "t", body: "b", category: tech, email: "user@company.com")
irb(main):005>   .tap(&:valid?).errors.full_messages
=> []

irb(main):006> Post.new(title: "t", body: "b", category: tech, email: nil).valid?
=> true    # allow_blank: true ทำให้ email ว่างได้ (validator เองก็ return ทันทีถ้า value.blank?)
```

**สิ่งที่เกิดขึ้น:** validator class เดียวกัน (`EmailValidator`) ถูกใช้ใน 2 Model ที่ไม่เกี่ยวข้อง
กันเลย (`Subscriber`, `Post`) ด้วยกฎที่ต่างกันเล็กน้อย (`Post` บังคับโดเมนเพิ่ม) — ทั้งหมดนี้
ไม่ต้องแก้ regex ซ้ำที่ไหนเลย ถ้าวันหนึ่งต้องรองรับอีเมลรูปแบบใหม่ (เช่น TLD ยาวขึ้น) แก้ที่
`EmailValidator` ไฟล์เดียวจบ

### ทดสอบ override message ตรงจุดที่เรียกใช้

```ruby
validates :email, email: { message: "รูปแบบอีเมลไม่ถูกต้องนะ" }
```

```irb
irb(main):007> form.valid?
=> false
irb(main):008> form.errors.full_messages
=> ["Email รูปแบบอีเมลไม่ถูกต้องนะ"]
```

`message:` ที่ส่งเข้ามาตอนเรียก `validates` มีค่าความสำคัญเหนือกว่าข้อความ default ที่เขียน
ไว้ใน validator class เอง (`options[:message] || "..."` ในโค้ดด้านบนคือกลไกที่ทำให้เกิด
พฤติกรรมนี้) — รูปแบบนี้ใช้ได้กับ custom validator ทุกตัวที่เขียนแบบนี้ ไม่ใช่แค่ `EmailValidator`

> **เทคนิค:** ถ้า validator มี option บังคับที่ต้องใส่เสมอ ใช้ `options.fetch(:key)` แทน
> `options[:key]` เพื่อให้ Rails raise `KeyError` ทันทีตอนโหลด Model ถ้ามีคนลืมใส่ option
> ที่จำเป็น ดีกว่าปล่อยให้ error เงียบๆ ตอน runtime (จะเห็นเทคนิคนี้ใน Step 314)

---

## Step 313: Cross-field Validation แบบง่ายด้วย `validate` และ `errors.add(:base, ...)`

Custom validator ที่เรียนไป 2 Step ที่แล้ว เหมาะกับกฎที่ตรวจ **field เดียว** (แม้จะรับ option
เพิ่มได้ แต่โฟกัสอยู่ที่ค่าของ attribute เดียว) แต่กฎทางธุรกิจจำนวนมากต้อง **เทียบค่าระหว่าง
field มากกว่าหนึ่งตัวในเรคคอร์ดเดียวกัน** เช่น "วันที่สิ้นสุดกิจกรรมต้องอยู่หลังวันที่เริ่มต้น"
กรณีแบบนี้ทบทวนจาก Part 026 Step 254: ใช้ `validate :method_name` ธรรมดาได้เลย ไม่ต้องใช้
custom validator class

### สร้าง Model `Event` มาสาธิต

```bash
bin/rails generate model Event name:string starts_on:date ends_on:date
bin/rails db:migrate
```

```ruby
# app/models/event.rb
class Event < ApplicationRecord
  validates :name, presence: true

  validate :ends_after_starts

  private

  def ends_after_starts
    return if starts_on.blank? || ends_on.blank?

    errors.add(:ends_on, "ต้องอยู่หลังวันที่เริ่มต้น") if ends_on <= starts_on
  end
end
```

```irb
irb(main):001> e1 = Event.new(name: "Conference", starts_on: Date.new(2026, 10, 10),
irb(main):002>                 ends_on: Date.new(2026, 10, 5))
irb(main):003> e1.valid?
=> false
irb(main):004> e1.errors.full_messages
=> ["Ends on ต้องอยู่หลังวันที่เริ่มต้น"]
```

สังเกตว่า method นี้ต้องเช็ค `blank?` ของทั้งสอง field ก่อนเปรียบเทียบเสมอ — ถ้า `starts_on`
หรือ `ends_on` เป็น `nil` แล้วเอาไปเทียบกันตรงๆ ด้วย `<=` จะได้ `NoMethodError` (เทียบ `nil`
กับ `Date` ไม่ได้) ปล่อยให้ `presence: true` (ถ้ามี) จัดการ field ที่ว่างไปแยกต่างหาก
method นี้สนใจแค่ "ถ้าทั้งคู่มีค่าแล้ว ลำดับถูกต้องไหม"

### เมื่อ error ไม่ควรผูกกับ field ใด field หนึ่ง — `errors.add(:base, ...)`

บางกฎ cross-field ไม่ได้ "โทษ" field ใดเป็นพิเศษ เช่น "ช่วงเวลาจัดงานห้ามเกิน 90 วัน" — จะ
โทษ `starts_on` ก็ไม่ถูก จะโทษ `ends_on` ก็ไม่ถูกเหมือนกัน เพราะปัญหาอยู่ที่**ระยะห่างระหว่าง
สองค่า** ไม่ใช่ค่าใดค่าหนึ่งผิดเดี่ยวๆ — กรณีแบบนี้ใช้ `:base` แทนชื่อ field ตามที่ทบทวนจาก
Part 026 Step 254:

```ruby
# app/models/event.rb
class Event < ApplicationRecord
  validates :name, presence: true

  validate :ends_after_starts
  validate :duration_within_limit

  private

  def ends_after_starts
    return if starts_on.blank? || ends_on.blank?

    errors.add(:ends_on, "ต้องอยู่หลังวันที่เริ่มต้น") if ends_on <= starts_on
  end

  def duration_within_limit
    return if starts_on.blank? || ends_on.blank?

    errors.add(:base, "ช่วงเวลาจัดงานห้ามเกิน 90 วัน") if (ends_on - starts_on) > 90
  end
end
```

```irb
irb(main):005> long_event = Event.new(name: "Expo ยาวเกินไป", starts_on: Date.new(2026, 1, 1),
irb(main):006>                         ends_on: Date.new(2026, 6, 1))
irb(main):007> long_event.valid?
=> false
irb(main):008> long_event.errors.full_messages
=> ["ช่วงเวลาจัดงานห้ามเกิน 90 วัน"]
irb(main):009> long_event.errors[:base]
=> ["ช่วงเวลาจัดงานห้ามเกิน 90 วัน"]
irb(main):010> long_event.errors[:starts_on]
=> []
irb(main):011> long_event.errors[:ends_on]
=> []

irb(main):012> ok_event = Event.new(name: "Conference ปกติ", starts_on: Date.new(2026, 10, 10),
irb(main):013>                       ends_on: Date.new(2026, 10, 12))
irb(main):014> ok_event.valid?
=> true
```

**กฎการเลือกใช้ที่จำง่าย:** ถ้าชี้ได้ชัดเจนว่า field ไหน "ผิด" (แม้จะต้องอ่านค่าจาก field อื่น
ประกอบการตัดสินใจ เช่น `ends_after_starts` ข้างบนที่ยังโทษ `:ends_on` ได้อย่างสมเหตุสมผล)
ให้ใช้ชื่อ field นั้นเป็นเป้าหมายของ `errors.add` เพราะถ้าเอาไปแสดงในฟอร์ม (Part 031/037) จะ
ไฮไลต์ field ที่ถูกต้องให้ผู้ใช้เห็นทันที แต่ถ้ากฎเกี่ยวข้องกับ**ความสัมพันธ์**ของหลาย field
พร้อมกันโดยไม่มีตัวไหนผิดเดี่ยวๆ ให้ใช้ `:base` — ข้อความจะไปโผล่รวมที่ส่วนบนของฟอร์มแทนที่
จะผูกกับ input field ใดเป็นพิเศษ

---

## Step 314: ยกระดับกฎข้าม Field ให้ใช้ซ้ำได้ด้วย `ActiveModel::Validator`

`ends_after_starts` ใน Step ที่แล้วใช้ได้ดีกับ `Event` แต่ถ้าแอปมี Model อื่นที่ต้องเช็คลำดับ
วันที่แบบเดียวกันอีก (เช่น `Booking` ที่มี `check_in`/`check_out`, หรือใน "แบบฝึกหัด" ท้าย Part
นี้ที่ `PromoCode` ก็ต้องเช็ค `starts_on`/`ends_on` เหมือนกันเป๊ะ) — การก็อป
`ends_after_starts` ไปวางซ้ำทุก Model ผิดหลัก DRY เหมือนปัญหาที่เจอใน Step 311

`ActiveModel::EachValidator` ที่ใช้ไปแล้วไม่เหมาะกับกรณีนี้ เพราะมันถูกออกแบบมาให้ตรวจ
**หนึ่ง attribute ต่อหนึ่งครั้ง** ไม่เห็น record ทั้งก้อนในมุมที่สะดวก — ตัวที่เหมาะกว่าคือ
**`ActiveModel::Validator`** ซึ่งเป็น base class ที่กว้างกว่า ให้ validator ตรวจสอบ "ทั้ง record"
ในคราวเดียว

### เขียน `DateRangeValidator` แบบใช้ซ้ำได้

```ruby
# app/validators/date_range_validator.rb
class DateRangeValidator < ActiveModel::Validator
  def validate(record)
    start_field = options.fetch(:start_field)
    end_field = options.fetch(:end_field)

    start_date = record.public_send(start_field)
    end_date = record.public_send(end_field)

    return if start_date.blank? || end_date.blank?

    if end_date <= start_date
      record.errors.add(end_field, "ต้องอยู่หลัง #{human_field_name(record, start_field)}")
    end
  end

  private

  def human_field_name(record, field)
    record.class.human_attribute_name(field)
  end
end
```

**ความต่างจาก `ActiveModel::EachValidator` ที่ต้อง สังเกตให้ชัด:**

- Override method ชื่อ `validate` (ไม่ใช่ `validate_each`) และรับ argument เดียวคือ `record`
  ทั้งก้อน ไม่ได้รับ `attribute`/`value` แยกมาให้ — เพราะ validator ประเภทนี้ตรวจทั้งเรคคอร์ด
  ไม่ได้ผูกกับ attribute ใดเป็นการเฉพาะตั้งแต่แรก
- ใช้ `options.fetch(:start_field)`/`options.fetch(:end_field)` (ไม่ใช่ `options[:key]`)
  เพราะ validator ตัวนี้**ต้อง**รู้ว่าจะเช็ค field คู่ไหน — ถ้าคนเรียกใช้ลืมระบุ ให้ raise
  `KeyError` ทันทีตอนโหลด Model ดีกว่าให้ validator เงียบๆ ไม่ทำอะไรเลย (ตามเทคนิคที่แนะไว้
  ท้าย Step 312)
- `record.public_send(start_field)` ใช้อ่านค่า attribute แบบ dynamic จากชื่อที่ config มา
  (ทบทวน `send`/`public_send` จาก Part 015 — ใช้ `public_send` แทน `send` เพราะเราต้องการ
  เรียกแค่ public method/attribute reader เท่านั้น ไม่ควรไปยุ่งกับ private method ของ record)

### ใช้งานผ่าน `validates_with`

Custom validator ที่สืบทอดจาก `ActiveModel::Validator` (ไม่ใช่ `EachValidator`) เรียกใช้ด้วย
`validates_with` ไม่ใช่ `validates`:

```ruby
# app/models/event.rb
class Event < ApplicationRecord
  validates :name, presence: true

  validates_with DateRangeValidator, start_field: :starts_on, end_field: :ends_on
  validate :duration_within_limit   # กฎเฉพาะของ Event เอง ยังคงเก็บไว้แบบเดิมได้

  private

  def duration_within_limit
    return if starts_on.blank? || ends_on.blank?

    errors.add(:base, "ช่วงเวลาจัดงานห้ามเกิน 90 วัน") if (ends_on - starts_on) > 90
  end
end
```

```irb
irb(main):001> bad = Event.new(name: "Conference", starts_on: Date.new(2026, 10, 10),
irb(main):002>                  ends_on: Date.new(2026, 10, 5))
irb(main):003> bad.valid?
=> false
irb(main):004> bad.errors.full_messages
=> ["Ends on ต้องอยู่หลัง Starts on"]

irb(main):005> ok = Event.new(name: "Conference ปกติ", starts_on: Date.new(2026, 10, 10),
irb(main):006>                 ends_on: Date.new(2026, 10, 12))
irb(main):007> ok.valid?
=> true
```

สังเกตว่าผลลัพธ์เหมือนกับ `ends_after_starts` แบบเดิมทุกประการ (แค่ข้อความ error ต่างไปนิดหน่อย
เพราะใช้ `human_attribute_name` แทนข้อความ hardcode) แต่ตอนนี้ **`DateRangeValidator` เป็น
class แยกต่างหากที่เรียกใช้ซ้ำกับ Model ไหนก็ได้ที่มีคู่ field วันที่ต้องเรียงลำดับกัน** เพียงแค่
เปลี่ยนชื่อ field ผ่าน option — ใน "แบบฝึกหัด" ท้าย Part นี้จะเห็นว่านำไปใช้กับ `PromoCode`
ได้ทันทีโดยไม่ต้องแก้โค้ด `DateRangeValidator` แม้แต่บรรทัดเดียว

> **สรุปการเลือกใช้ 3 แนวทางที่เรียนมา:**
>
> | แนวทาง | ใช้เมื่อ |
> |---------|-----------|
> | `validate :method_name` (Part 026) | กฎเฉพาะของ Model นี้ ไม่มีแผนใช้ซ้ำที่อื่น |
> | `ActiveModel::EachValidator` (Step 311) | กฎตรวจ **field เดียว** ที่อยากใช้ซ้ำหลาย Model/field |
> | `ActiveModel::Validator` + `validates_with` (Step 314) | กฎตรวจ **หลาย field ร่วมกัน** ที่อยากใช้ซ้ำหลาย Model |

---

## Step 315: `uniqueness: { scope: ... }` — ไม่ซ้ำแบบมีเงื่อนไข (Composite Uniqueness)

### ปัญหา: "ไม่ซ้ำ" บางครั้งไม่ได้แปลว่า "ไม่ซ้ำทั้งตาราง"

ทบทวนจาก Part 026 Step 253: `validates :slug, uniqueness: true` ตรวจว่า `slug` ไม่ซ้ำกับ
**ทุกแถวในตาราง `posts`** แต่ในระบบบล็อกจริงจำนวนมาก กฎที่ต้องการจริงๆ คือ "slug ไม่ซ้ำ
**ภายในหมวดหมู่เดียวกัน**" — บล็อกในหมวด "เทคโนโลยี" กับหมวด "ไลฟ์สไตล์" ควรตั้ง slug ชื่อ
เดียวกันได้โดยไม่ชนกัน (เช่น `/tech/intro` กับ `/lifestyle/intro` เป็นคนละหน้ากัน)

### เพิ่มความสัมพันธ์ `Category` ให้ `Post`

```bash
bin/rails generate model Category name:string slug:string
bin/rails generate migration AddSlugCategoryEmailToPosts slug:string category:references email:string
```

```ruby
# db/migrate/..._add_slug_category_email_to_posts.rb
class AddSlugCategoryEmailToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :slug, :string
    add_reference :posts, :category, null: false, foreign_key: true
    add_column :posts, :email, :string
  end
end
```

> **ข้อควรระวังเรื่อง `null: false` กับข้อมูลเก่า:** ถ้าตาราง `posts` มีแถวอยู่แล้วก่อนรัน
> migration นี้ (เช่นบล็อกจาก Part 030 ที่มีข้อมูลจริง) การเพิ่ม column ใหม่แบบ `null: false`
> ตรงๆ จะทำให้ migration ล้มเหลวทันที เพราะแถวเก่าไม่มีค่า `category_id` ให้ ในสถานการณ์จริง
> ต้องแบ่งเป็น 3 ขั้น: (1) เพิ่ม column แบบ `null: true` ก่อน (2) รัน data migration
> backfill ค่า `category_id` ให้แถวเก่าทุกแถว (3) ค่อยตามด้วย migration แยกที่เปลี่ยนเป็น
> `change_column_null :posts, :category_id, false` ในภายหลัง — เรื่องนี้จะสอนละเอียดเรื่อง
> zero-downtime migration ใน Part 077 ตอนนี้ขอสมมติว่าเป็นตารางที่เพิ่งเริ่มสร้างข้อมูลใหม่

```bash
bin/rails db:migrate
```

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :posts

  validates :name, presence: true
end
```

### เพิ่ม `scope:` ให้ `uniqueness`

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :category

  validates :title, presence: true
  validates :slug, presence: true, uniqueness: { scope: :category_id }
  validates :email, email: { domain: "company.com" }, allow_blank: true
end
```

`scope: :category_id` แปลว่า "ตรวจไม่ซ้ำเฉพาะภายในแถวที่มี `category_id` เดียวกันเท่านั้น"
ไม่ใช่ตรวจกับทั้งตารางเหมือนก่อนหน้านี้:

```irb
irb(main):001> tech = Category.create!(name: "เทคโนโลยี", slug: "tech")
irb(main):002> life = Category.create!(name: "ไลฟ์สไตล์", slug: "lifestyle")

irb(main):003> Post.create!(title: "แนะนำ Ruby", body: "b", category: tech, slug: "intro")

irb(main):004> dup_same_category = Post.new(title: "แนะนำ Ruby อีกครั้ง", body: "b",
irb(main):005>                               category: tech, slug: "intro")
irb(main):006> dup_same_category.valid?
=> false
irb(main):007> dup_same_category.errors.full_messages
=> ["Slug has already been taken"]

irb(main):008> different_category = Post.new(title: "แนะนำการใช้ชีวิต", body: "b",
irb(main):009>                                 category: life, slug: "intro")
irb(main):010> different_category.valid?
=> true    # slug "intro" ซ้ำกับโพสต์แรก แต่คนละ category_id จึงผ่าน
```

`scope:` รับได้ทั้ง symbol เดียว (ตามตัวอย่าง) หรือ Array ของหลาย symbol พร้อมกันถ้าต้องเช็ค
ความไม่ซ้ำแบบผสมมากกว่า 2 field เช่น `scope: [:category_id, :locale]` (ไม่ซ้ำเฉพาะภายใน
หมวดหมู่เดียวกัน**และ**ภาษาเดียวกัน)

---

## Step 316: พิสูจน์ Race Condition ของ `uniqueness` Validator ด้วยของจริง

Part 026 Step 253 เคยเตือนไว้สั้นๆ ว่า `uniqueness` validator มีช่องโหว่จาก **race condition**
ในทางทฤษฎี Step นี้จะจำลองสถานการณ์จริงให้เห็นว่ามันเกิดขึ้นได้จริงๆ ไม่ใช่แค่ทฤษฎี

### ทำไม `uniqueness` ถึงไม่ใช่การรับประกันที่แท้จริง

เวลาเรียก `post.valid?` (หรือ `post.save`) `uniqueness` validator ทำงานด้วยการยิง SQL
ประมาณนี้เบื้องหลัง:

```sql
SELECT 1 FROM posts WHERE category_id = ? AND slug = ? LIMIT 1
```

ถ้าไม่เจอแถวไหนตรงเงื่อนไข ก็ถือว่า "ไม่ซ้ำ ผ่าน validation" แล้วปล่อยให้ `INSERT` เกิดขึ้นต่อ
— ปัญหาคือ **ระหว่างช่วงเวลาสั้นๆ ตั้งแต่ `SELECT` เสร็จจนถึง `INSERT` เริ่มทำงานจริง** ถ้ามี
Request อื่นแทรกเข้ามาทำ `SELECT` ของตัวเองในจังหวะเดียวกันพอดี (ก่อนที่ Request แรกจะ
`INSERT` เสร็จ) ทั้งสอง Request จะเห็นผลลัพธ์ `SELECT` แบบเดียวกันคือ "ยังไม่มีใครใช้ค่านี้"
แล้วทั้งคู่ก็ `INSERT` ผ่านไปได้พร้อมกัน สุดท้ายได้ข้อมูลซ้ำเข้าฐานข้อมูลจริงทั้งที่ validation
ผ่านทั้งคู่

### จำลองสถานการณ์นี้ด้วย 2 Thread จริง

เขียนสคริปต์ที่รัน 2 "request" พร้อมกันด้วย Ruby Thread โดยแทรก `sleep` สั้นๆ ไว้ระหว่างตอนที่
`valid?` ตรวจผ่านแล้ว กับตอนที่ค่อยเขียนข้อมูลจริงลงฐานข้อมูล (จำลอง "เวลาที่ request ใช้
ประมวลผลเรื่องอื่นก่อน save จริง" ซึ่งในระบบ production ช่วงเวลานี้อาจสั้นมาก แต่ก็เพียงพอให้
เกิด race ได้เมื่อ traffic สูงพอ):

```ruby
# race_condition_demo.rb — รันด้วย bin/rails runner race_condition_demo.rb
tech = Category.find_or_create_by!(name: "เทคโนโลยี")
results = {}

# Thread A จำลอง Request แรกที่เข้ามาก่อน
thread_a = Thread.new do
  ActiveRecord::Base.connection_pool.with_connection do
    post = Post.new(title: "บทความจาก Request A", body: "b", category: tech, slug: "hot-news")
    results[:a_valid] = post.valid?    # ตรวจผ่าน เพราะยังไม่มีใครใช้ slug นี้
    sleep 0.3                          # จำลองงานอื่นที่ทำให้ request ช้าลงเล็กน้อยก่อน save จริง
    post.save(validate: false)         # เขียนข้อมูลจริง (ข้าม validate ซ้ำ เพื่อจำลอง
    results[:a_saved] = post.persisted?  # จังหวะที่ Request แรก "ตัดสินใจ" ไปแล้วว่าจะบันทึก)
  end
end

# Thread B จำลอง Request ที่สองที่เข้ามาเกือบพร้อมกัน (ห่างกันแค่ 0.1 วินาที)
thread_b = Thread.new do
  ActiveRecord::Base.connection_pool.with_connection do
    sleep 0.1
    post = Post.new(title: "บทความจาก Request B", body: "b", category: tech, slug: "hot-news")
    results[:b_valid] = post.valid?    # ตรวจตอนนี้ Request A ยังไม่ INSERT จริง -> ผ่านเช่นกัน!
    post.save(validate: false)
    results[:b_saved] = post.persisted?
  end
end

thread_a.join
thread_b.join

p results
puts "จำนวนโพสต์ slug ซ้ำกันจริงในฐานข้อมูล: " \
     "#{Post.where(slug: 'hot-news', category: tech).count}"
```

```bash
bin/rails runner race_condition_demo.rb
```

```
{:a_valid=>true, :a_saved=>true, :b_valid=>true, :b_saved=>true}
จำนวนโพสต์ slug ซ้ำกันจริงในฐานข้อมูล: 2
```

**นี่คือหลักฐานที่จับต้องได้:** ทั้ง `results[:a_valid]` และ `results[:b_valid]` เป็น `true`
ทั้งคู่ — validator บอกว่า "ไม่ซ้ำ ผ่านหมด" ทั้งสอง Request แต่สุดท้ายมี**โพสต์สอง Row ที่มี
slug และ category_id เหมือนกันเป๊ะอยู่ในฐานข้อมูลจริง** ทั้งที่ Model ประกาศ
`uniqueness: { scope: :category_id }` ไว้ชัดเจน — พิสูจน์ว่าคำเตือนใน Part 026 เป็นเรื่องจริง
ไม่ใช่แค่ความเสี่ยงทางทฤษฎี

> **ทำไมสถานการณ์นี้เกิดขึ้นได้ง่ายกว่าที่คิดในระบบจริง:** ตัวอย่างข้างบนใช้ `sleep` เพื่อบังคับ
> ให้เห็นผลชัดเจนแบบไม่ต้องพึ่งจังหวะสุ่ม แต่ในระบบ production ช่องโหว่นี้เปิดกว้างขึ้นมากเมื่อ:
> เว็บแอปรันหลาย process/server พร้อมกัน (ธรรมดามากในระบบที่มี traffic สูง), ผู้ใช้กด submit
> ปุ่มซ้ำสองครั้งติดกันเร็วๆ (double-click), หรือมี retry logic ฝั่ง client ที่ยิง request ซ้ำ
> อัตโนมัติเมื่อ network ช้า — ทุกกรณีนี้สร้างจังหวะ "สอง request พร้อมกันเกือบสนิท" ได้ทั้งนั้น

---

## Step 317: ทางแก้ที่แท้จริง — Unique Index ระดับฐานข้อมูล + `rescue RecordNotUnique`

### เหตุผลเชิงโครงสร้าง: validation คือ Ruby, ไม่ใช่ database

`uniqueness` validator ทำงานอยู่ฝั่ง **Ruby/Rails** ล้วนๆ (ยิง `SELECT` ธรรมดา) ไม่มีกลไกใด
รับประกันว่าระหว่าง `SELECT` กับ `INSERT` จะไม่มีใครมาแทรก — การรับประกันแบบนั้นทำได้ที่ระดับ
**ฐานข้อมูล**เท่านั้น ผ่าน **unique index** ซึ่งเป็นกลไกที่ฐานข้อมูลบังคับใช้แบบ atomic
(อะตอมมิก คือรับประกันว่าจะไม่มีสถานะ "ครึ่งๆ กลางๆ" ระหว่างการเช็คกับการเขียนเกิดขึ้นได้เลย
เพราะเป็นการดำเนินการเดียวกันในระดับ storage engine)

### เพิ่ม Unique Index แบบผสม (Composite Unique Index)

```bash
bin/rails generate migration AddUniqueIndexToPostsSlug
```

```ruby
# db/migrate/..._add_unique_index_to_posts_slug.rb
class AddUniqueIndexToPostsSlug < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, [:category_id, :slug], unique: true
  end
end
```

```bash
bin/rails db:migrate
```

`add_index :posts, [:category_id, :slug], unique: true` สร้าง **composite unique index**
(ดัชนีผสมจากสอง column) ที่ตรงกับ scope ของ `uniqueness: { scope: :category_id }` เป๊ะๆ —
กฎที่ตั้งไว้ระดับ Ruby (`uniqueness` + `scope:`) กับกฎที่ตั้งไว้ระดับฐานข้อมูล (unique index)
**ต้องตรงกันเสมอ** ไม่เช่นนั้นจะมีช่องว่างที่ฐานข้อมูลอนุญาตแต่ Model ห้าม หรือกลับกัน

รันสคริปต์จำลอง race condition ของ Step 316 ซ้ำอีกครั้ง (ครั้งนี้มี unique index แล้ว):

```
posts.rb:15:in `save': SQLite3::ConstraintException: UNIQUE constraint failed:
posts.category_id, posts.slug (ActiveRecord::RecordNotUnique)
```

คราวนี้ Thread ที่สองที่พยายาม `INSERT` ซ้ำจะได้รับ **exception จากฐานข้อมูลโดยตรง** ไม่ใช่
ความเงียบแบบเดิม — Rails ห่อ exception ระดับ database (`SQLite3::ConstraintException` ในกรณี
นี้, `PG::UniqueViolation` ถ้าใช้ PostgreSQL) ให้เป็น class กลางที่ไม่ขึ้นกับฐานข้อมูลชื่อ
**`ActiveRecord::RecordNotUnique`** เสมอ ไม่ว่าจะใช้ฐานข้อมูลอะไรอยู่

### จับ exception นี้ให้ถูกต้องในโค้ดจริง

ทางแก้ที่ถูกต้องไม่ใช่การปล่อยให้แอป crash เวลาชนกันจริง แต่คือ `rescue`
`ActiveRecord::RecordNotUnique` แล้วแปลงเป็น error message ที่เป็นมิตรกับผู้ใช้แบบเดียวกับที่
`uniqueness` validator เคยให้:

```ruby
def create_post_with_race_protection(attrs)
  post = Post.new(attrs)
  post.save!
  post
rescue ActiveRecord::RecordNotUnique
  post.errors.add(:slug, "has already been taken")
  post
end
```

```irb
irb(main):001> tech = Category.find_or_create_by!(name: "เทคโนโลยี")
irb(main):002> Post.create!(title: "t", body: "b", category: tech, slug: "safe-slug")

irb(main):003> result = create_post_with_race_protection(
irb(main):004>   title: "t2", body: "b", category: tech, slug: "safe-slug"
irb(main):005> )
irb(main):006> result.persisted?
=> false
irb(main):007> result.errors.full_messages
=> ["Slug has already been taken"]
```

ผลลัพธ์ที่ผู้ใช้เห็นหน้าจอ**เหมือนกับตอนที่ `uniqueness` validator จับได้ตามปกติทุกประการ** —
ต่างกันแค่เบื้องหลัง ครั้งนี้ฐานข้อมูลเป็นคนบอกเราแทน ไม่ใช่การ `SELECT` เช็คล่วงหน้า

> **สรุปสถาปัตยกรรมที่ถูกต้อง (จำเป็นทั้งสองชั้น ไม่ใช่เลือกอย่างใดอย่างหนึ่ง):**
>
> 1. **`validates :field, uniqueness: { scope: ... }`** — ให้ประสบการณ์ผู้ใช้ที่ดี บอก error
>    ทันทีตอน `valid?`/`save` แบบปกติ โดยไม่ต้องรอชน constraint จริงก่อน (เร็วกว่า และไม่ต้อง
>    เขียน `rescue` ทุกจุดที่ save) ครอบคลุมได้ 99%+ ของสถานการณ์จริงที่ไม่มีการชนกันพอดี
> 2. **Unique index ระดับฐานข้อมูล** — เป็นเซฟตี้เน็ตสุดท้ายที่รับประกันจริงว่าข้อมูลซ้ำจะไม่
>    มีทางหลุดเข้าฐานข้อมูลได้ ไม่ว่า traffic จะสูงแค่ไหนหรือมี process กี่ตัวรันพร้อมกัน
>
> ใช้ **แค่ validator อย่างเดียว** ข้อมูลซ้ำหลุดเข้าได้ตามที่พิสูจน์ใน Step 316 ใช้
> **แค่ unique index อย่างเดียว** (ไม่มี validator) ผู้ใช้จะเจอหน้า error หน้าเว็บแตกดิบๆ
> (`ActiveRecord::RecordNotUnique` ที่ไม่ได้ถูก rescue) แทนที่จะเห็นข้อความสวยงามในฟอร์ม —
> **ต้องมีทั้งคู่เสมอสำหรับทุก field ที่ต้องไม่ซ้ำในระบบ production จริง**

---

## Step 318: `validates_associated` และกับดัก Infinite Validation Loop

### `validates_associated` — ตรวจว่า object ที่เชื่อมโยงอยู่ก็ valid ด้วย

บางครั้งเรคคอร์ดหนึ่งจะ "valid" ได้ก็ต่อเมื่อ object ที่มันเชื่อมโยงอยู่ (ผ่าน association ที่
เรียนจาก Part 027) ก็ต้อง valid ด้วยเช่นกัน — `validates_associated` ทำสิ่งนี้ให้อัตโนมัติ:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :category

  validates :title, presence: true
  validates :slug, presence: true, uniqueness: { scope: :category_id }
  validates_associated :category
end
```

```irb
irb(main):001> bad_category = Category.new(name: nil)   # Category ต้องมี name (presence: true)
irb(main):002> post = Post.new(title: "t", body: "b", slug: "s1", category: bad_category)
irb(main):003> post.valid?
=> false
irb(main):004> post.errors.full_messages
=> ["Category is invalid"]
```

สังเกตว่า error message ที่ได้คือ **"Category is invalid"** เฉยๆ ไม่ได้บอกรายละเอียดว่า
`Category` ผิดตรงไหน (`name` ว่าง) — `validates_associated` แค่บอกว่า "associated object ตัวนี้
ไม่ผ่าน" ไม่ได้ดึง error message ของ `Category` มาแสดงให้อัตโนมัติ ถ้าต้องการรายละเอียดที่
ชัดเจนกว่านี้ต้องเข้าถึง `post.category.errors.full_messages` เอง

### กับดักร้ายแรง: ใช้ `validates_associated` สองทางพร้อมกัน = Infinite Loop

ปัญหาใหญ่ที่พบบ่อยเกิดขึ้นเมื่อประกาศ `validates_associated` **ทั้งสองฝั่ง** ของความสัมพันธ์
พร้อมกัน:

```ruby
# app/models/category.rb — ห้ามทำแบบนี้!
class Category < ApplicationRecord
  has_many :posts

  validates :name, presence: true
  validates_associated :posts   # ⚠️ อันตราย
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :category

  validates_associated :category   # ทั้งสองฝั่งเรียกกันไปมา
end
```

ลองเรียก `valid?` ดู:

```irb
irb(main):001> category = Category.new(name: "ข่าว")
irb(main):002> post = Post.new(title: "t", body: "b", slug: "s1", category: category)
irb(main):003> category.posts << post
irb(main):004> category.valid?
```

```
arel/visitors/to_sql.rb:888:in `each': stack level too deep (SystemStackError)
	from arel/visitors/to_sql.rb:888:in `each_with_index'
	from arel/visitors/to_sql.rb:888:in `inject_join'
	... (วนซ้ำแบบนี้อีกหลายพัน frame) ...
```

แอป **crash ทันทีด้วย `SystemStackError`** (stack level too deep) — นี่ไม่ใช่ error ที่ดัก
ด้วย `rescue` ธรรมดาได้ง่ายๆ เพราะเกิดจากการเรียกฟังก์ชันซ้อนกันลึกเกินขีดจำกัดของ Ruby VM เอง
สาเหตุคือ: `category.valid?` → เรียก `validates_associated :posts` → เรียก `post.valid?` ของ
แต่ละโพสต์ → โพสต์แต่ละอันมี `validates_associated :category` → เรียก `category.valid?` อีกรอบ
→ วนกลับไปเรียก `post.valid?` อีก → **วนไม่มีที่สิ้นสุด** จนกว่า stack จะเต็ม

> **ข้อยกเว้นที่ Rails ป้องกันให้บางส่วน:** ActiveRecord มีกลไกภายในที่ป้องกัน infinite loop
> จากการ validate object เดียวกันซ้ำ**ในสาย call เดียวกัน**บางกรณี (ผ่าน
> `already_being_validated` guard) แต่ไม่ครอบคลุมทุกรูปแบบความสัมพันธ์ โดยเฉพาะ `has_many`
> ที่มีหลาย object ในฝั่งตรงข้าม — จึงยังพบเห็น `SystemStackError` แบบนี้ได้จริงในโปรเจกต์จริง
> **อย่าพึ่งพากลไกป้องกันภายในนี้ ให้หลีกเลี่ยงการประกาศ `validates_associated` ทั้งสองทาง
> ตั้งแต่ต้นเป็นกฎเหล็ก**

### แนวทางที่ถูกต้อง

ใช้ `validates_associated` **แค่ทางเดียว** — ฝั่งที่เป็น "เจ้าของ" ความสัมพันธ์ตามความหมายทาง
ธุรกิจ โดยทั่วไปมักเป็นฝั่ง `belongs_to` (Post ต้องมี Category ที่ valid ถึงจะเผยแพร่ได้)
ไม่ใช่ฝั่ง `has_many` (Category ไม่จำเป็นต้องรู้ว่า Post ทุกตัวของมัน valid หรือไม่ ปกติแล้ว
`Post` แต่ละตัวจัดการ validation ของตัวเองอยู่แล้วตอนถูก save):

```ruby
class Category < ApplicationRecord
  has_many :posts
  validates :name, presence: true
  # ไม่มี validates_associated :posts
end

class Post < ApplicationRecord
  belongs_to :category
  validates_associated :category   # ทางเดียวพอ
end
```

> **กฎจำง่าย:** ก่อนใส่ `validates_associated` ทุกครั้ง เช็คก่อนว่าฝั่งตรงข้ามของความสัมพันธ์
> นั้นมี `validates_associated` กลับมาหาเราอยู่แล้วหรือไม่ (ค้นหาด้วย `grep -rn
> "validates_associated" app/models/` ก็ได้) ถ้ามี ให้ตัดออกฝั่งใดฝั่งหนึ่งทันที — และควรใช้
> `validates_associated` อย่างประหยัด เพราะส่วนใหญ่แล้วแค่ละ Model จัดการ validation ของ
> ตัวเองตอน save อยู่แล้วโดยธรรมชาติ (ผ่าน `belongs_to` ที่ Rails บังคับ presence ให้อัตโนมัติ)
> ไม่จำเป็นต้องเช็คซ้ำข้ามฝั่งบ่อยๆ

---

## Step 319: `ActiveModel::Validations` แบบ Standalone บน Plain Old Ruby Object

### ปัญหา: ไม่ใช่ทุก object ที่ต้องการ validation จะเป็น ActiveRecord

ระบบจริงจำนวนมากมี object ที่ต้อง valid ก่อนใช้งาน แต่**ไม่ควรผูกกับตารางในฐานข้อมูลโดยตรง**
เช่น "ฟอร์มสมัครรับข่าวสาร" ที่รวมข้อมูลจากผู้ใช้แล้วเช็คเงื่อนไข แต่สุดท้ายอาจแค่สร้าง
`Subscriber` record เดียว หรืออาจไม่สร้าง record ใดๆ เลยถ้าแค่ต้องการ trigger side effect
บางอย่าง — object ประเภทนี้เรียกว่า **Form Object** (จะสอนอย่างเป็นทางการเต็มรูปแบบใน
Part 082) แต่หลักการ validation เบื้องหลังมันคือสิ่งที่เรียนไปแล้วทั้งหมดใน Part นี้

Rails ทำให้ class ธรรมดา (Plain Old Ruby Object หรือ PORO — ไม่ได้สืบทอดจาก
`ApplicationRecord` เลย) มีระบบ validation แบบเดียวกับ ActiveRecord ได้ ด้วยการ `include`
2 module: **`ActiveModel::Model`** (ให้ `initialize(attributes)`, `valid?`, `errors`,
`persisted?`) และ **`ActiveModel::Attributes`** (ให้ประกาศ attribute พร้อมชนิดข้อมูลได้แบบ
เดียวกับ column ของ ActiveRecord)

### ตัวอย่างจริง: `NewsletterSignupForm`

```ruby
# app/models/newsletter_signup_form.rb
class NewsletterSignupForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :agree_to_terms, :boolean, default: false

  validates :name, presence: true
  validates :email, presence: true, email: true   # ใช้ EmailValidator ตัวเดียวกับ Step 311!
  validate :must_agree_to_terms

  def call
    return false unless valid?

    Subscriber.find_or_create_by!(email: email)
    true
  end

  private

  def must_agree_to_terms
    return if agree_to_terms

    errors.add(:agree_to_terms, "ต้องยอมรับเงื่อนไขการใช้งานก่อนสมัคร")
  end
end
```

**สังเกตสิ่งสำคัญ:** `class NewsletterSignupForm` **ไม่ได้สืบทอดจาก `ApplicationRecord`**
เลยแม้แต่น้อย — ไม่มีตาราง `newsletter_signup_forms` ในฐานข้อมูล ไม่มี `db/migrate` ใดๆ
เกี่ยวข้องกับ class นี้ทั้งสิ้น แต่ `validates`, `validate`, `errors.add`, `errors.full_messages`
ทำงานได้เหมือนกับที่ใช้ใน `ActiveRecord::Base` ทุกประการ — เพราะแท้จริงแล้ว **ActiveRecord
เองก็ได้ระบบ validation ทั้งหมดมาจาก `ActiveModel::Validations` เหมือนกัน** (`ActiveRecord::Base`
include module นี้ไว้อยู่แล้วเป็นค่าเริ่มต้น) เราแค่ไปหยิบ module ชุดเดียวกันมาใช้กับ class
ที่ไม่ผูกกับฐานข้อมูลเท่านั้นเอง

### ทดสอบการทำงาน

```irb
irb(main):001> form = NewsletterSignupForm.new(name: nil, email: "bad", agree_to_terms: false)
irb(main):002> form.valid?
=> false
irb(main):003> form.errors.full_messages
=>
["Name can't be blank",
 "Email ไม่ใช่รูปแบบอีเมลที่ถูกต้อง",
 "Agree to terms ต้องยอมรับเงื่อนไขการใช้งานก่อนสมัคร"]

irb(main):004> ok_form = NewsletterSignupForm.new(name: "สมชาย", email: "somchai@example.com",
irb(main):005>                                     agree_to_terms: true)
irb(main):006> ok_form.valid?
=> true
irb(main):007> ok_form.call
=> true
irb(main):008> Subscriber.exists?(email: "somchai@example.com")
=> true
```

สังเกตว่าเราเรียก `validates :email, ..., email: true` ซ้ำ **validator ตัวเดียวกัน**
(`EmailValidator`) ที่เขียนไว้ตั้งแต่ Step 311 — โดยไม่ต้องแก้อะไรเลยแม้แต่บรรทัดเดียว ทั้งที่
`NewsletterSignupForm` ไม่ใช่ ActiveRecord และ `Subscriber` เป็น ActiveRecord — นี่คือพลังที่
แท้จริงของการแยก validator ออกมาเป็น class อิสระตามที่เรียนใน Step 311–312: **ใช้ซ้ำได้ข้าม
ทั้งขอบเขตของ ActiveRecord กับ PORO เลย ไม่ใช่แค่ข้าม Model**

`call` เป็นแค่ชื่อ method ที่เราตั้งเอง (ไม่ใช่ method พิเศษของ Rails) ตามธรรมเนียมของ
Form Object/Service Object ที่นิยมตั้งชื่อ method หลักว่า `call` — จะสอนแบบเป็นทางการพร้อม
pattern ที่ครบถ้วนกว่านี้ใน Part 082

---

## Step 320: ปรับแต่งข้อความ Error — `message:`, I18n, และ `errors.add(:base)` เทียบกับ Field-specific

### ทบทวน `message:` แบบ inline (จาก Part 026 + Step 312)

วิธีที่เร็วที่สุดในการปรับข้อความ error คือใส่ `message:` ตรงจุดที่เรียก validator:

```ruby
validates :title, presence: { message: "กรุณาใส่หัวข้อของบทความ" }
```

```irb
irb(main):001> Post.new(title: nil, category: tech).tap(&:valid?).errors.full_messages
=> ["Title กรุณาใส่หัวข้อของบทความ"]
```

วิธีนี้เร็วดี แต่มีข้อเสียเมื่อแอปโตขึ้น: ข้อความ error กระจัดกระจายอยู่ทั่วไฟล์ Model ต่างๆ
ทั้งแอป ถ้าต้องการแปลเป็นหลายภาษา หรือแก้คำให้สอดคล้องกันทั้งระบบ ต้องไล่หาทุกจุดที่ใส่
`message:` เอง — ทางออกที่ scale ได้กว่าคือย้ายข้อความทั้งหมดไปไว้ที่ **I18n locale file**

### ย้ายข้อความ error ไปที่ `config/locales/th.yml`

```yaml
# config/locales/th.yml
th:
  activerecord:
    attributes:
      post:
        title: "หัวข้อ"
        slug: "สลัก URL"
      event:
        starts_on: "วันที่เริ่มต้น"
        ends_on: "วันที่สิ้นสุด"
    errors:
      messages:
        blank: "ต้องระบุข้อมูลนี้"
        taken: "ถูกใช้ไปแล้ว"
      models:
        post:
          attributes:
            title:
              blank: "กรุณาใส่หัวข้อของบทความ"
```

โครงสร้าง key ของ `activerecord.errors` มีลำดับความสำคัญจากเจาะจงที่สุดไปทั่วไปที่สุด (Rails
ค้นหาจากบนลงล่าง เจอที่ไหนก่อนใช้ที่นั่นทันที):

1. `activerecord.errors.models.<model>.attributes.<attribute>.<error_type>` — เจาะจงที่สุด
   (เฉพาะ attribute เฉพาะ Model นี้เท่านั้น) เช่น `post.title.blank` ด้านบน
2. `activerecord.errors.models.<model>.<error_type>` — เฉพาะ Model นี้ ทุก attribute
3. `activerecord.errors.messages.<error_type>` — ทั่วทั้งแอป ใช้กับทุก Model/attribute ที่ไม่มี
   key เจาะจงกว่านี้ (เช่น `blank`, `taken` ด้านบน)
4. ค่า default ของ Rails เอง (ภาษาอังกฤษ) — ใช้เมื่อไม่มี key ไหนตรงเลย

ส่วน `activerecord.attributes.<model>.<attribute>` (ไม่มีคำว่า `errors`) ใช้แปลชื่อ field
เอง ซึ่งมีผลกับทั้ง `human_attribute_name` (ที่ `DateRangeValidator` ใน Step 314 เรียกใช้อยู่
แล้ว) และคำนำหน้าที่ `errors.full_messages` ใส่ให้อัตโนมัติ

### ทดสอบผลลัพธ์

ตั้งค่า default locale เป็นไทยที่ `config/application.rb`:

```ruby
config.i18n.default_locale = :th
```

```irb
irb(main):001> tech = Category.create!(name: "เทคโนโลยี")
irb(main):002> Post.create!(title: "แนะนำ Ruby", body: "b", slug: "dup-slug", category: tech)

irb(main):003> dup = Post.new(title: "t2", body: "b", slug: "dup-slug", category: tech)
irb(main):004> dup.valid?
=> false
irb(main):005> dup.errors[:slug]
=> ["ถูกใช้ไปแล้ว"]
irb(main):006> dup.errors.full_messages
=> ["สลัก URL ถูกใช้ไปแล้ว"]

irb(main):007> no_title = Post.new(title: nil, category: tech)
irb(main):008> no_title.valid?
=> false
irb(main):009> no_title.errors[:title]
=> ["กรุณาใส่หัวข้อของบทความ"]
irb(main):010> no_title.errors.full_messages
=> ["หัวข้อ กรุณาใส่หัวข้อของบทความ", "สลัก URL ต้องระบุข้อมูลนี้"]
```

สังเกตผลลัพธ์ที่น่าสนใจ 2 จุด:

- `dup.errors.full_messages` อ่านลื่นเป็นธรรมชาติ: **"สลัก URL ถูกใช้ไปแล้ว"** เพราะข้อความ
  generic (`taken: "ถูกใช้ไปแล้ว"`) ไม่ได้พูดถึงชื่อ field ในตัวเอง ปล่อยให้
  `errors.full_messages` เติมชื่อ field (ที่แปลไว้ใน `activerecord.attributes`) นำหน้าให้
  เองตามปกติ
- `no_title.errors.full_messages` ได้ข้อความที่**ฟังดูซ้ำซ้อนเล็กน้อย**: "หัวข้อ กรุณาใส่หัวข้อ
  ของบทความ" เพราะข้อความเฉพาะของ `post.title.blank` ที่เขียนไว้ ("กรุณาใส่หัวข้อของบทความ")
  พูดถึง "หัวข้อ" อยู่แล้วในตัวมันเอง แล้ว `full_messages` ก็ยังเติมคำว่า "หัวข้อ" (จาก
  `activerecord.attributes.post.title`) นำหน้าซ้ำเข้าไปอีกชั้นตามธรรมเนียม

> **ข้อควรระวังเชิงปฏิบัติเวลาเขียนข้อความภาษาไทย:** เมื่อเขียนข้อความ error เฉพาะเจาะจงระดับ
> `models.<model>.attributes.<attribute>` แบบเต็มประโยค (มีชื่อ field อยู่ในตัวข้อความอยู่แล้ว)
> ให้แสดงผลด้วย `errors[:field].first` หรือ `errors.messages[:field]` แทน
> `errors.full_messages` ในหน้า view เพื่อไม่ให้ชื่อ field ซ้ำสองรอบ — ส่วนข้อความ generic
> (`activerecord.errors.messages.*`) ที่ไม่มีชื่อ field ในตัวเอง ใช้คู่กับ `full_messages`
> ได้ตามปกติโดยไม่มีปัญหา ทีมพัฒนาส่วนใหญ่จึงเลือกแนวทางใดแนวทางหนึ่งให้สม่ำเสมอทั้งแอป
> (นิยมใช้ `errors[:field].first` แสดงข้างช่อง input ในฟอร์มโดยตรง มากกว่า `full_messages`
> ที่มักใช้แสดงเป็นสรุปรวมด้านบนฟอร์มแทน) — เรื่อง I18n และการจัดการ locale file แบบเต็ม
> รูปแบบ (การสลับภาษา, pluralization, การจัดโครงสร้างไฟล์ locale ขนาดใหญ่) จะสอนแบบละเอียด
> ทั้ง Part ใน **Part 039 (Internationalization)**

### `message:` แบบ inline ยังคง override I18n ได้เสมอ

ลำดับความสำคัญสุดท้าย: ถ้าใส่ `message:` ตรงจุดที่เรียก `validates` ไว้ด้วย ข้อความนั้นชนะ
ทุกอย่างใน I18n locale file ทันที (Rails เช็ค option `message:` ที่ส่งมาตรงๆ เป็นอันดับแรก
ก่อนไปค้นหาใน locale file เลย) — มีประโยชน์เวลาต้องการ override เฉพาะจุดเดียวโดยไม่อยากแก้
locale file ที่ใช้ร่วมกับที่อื่น

---

## แบบฝึกหัด: ระบบโปรโมชันร้านค้า (`PromoCode`) และฟอร์มติดต่อ (`ContactForm`)

### โจทย์

ร้านค้าออนไลน์ต้องการระบบโค้ดส่วนลด (`PromoCode`) ที่มีเงื่อนไข:

1. โค้ดส่วนลด (`code`) ต้องไม่ซ้ำกัน**ภายในร้านเดียวกัน**เท่านั้น (`store`) — ร้าน A กับร้าน B
   ใช้โค้ด `SALE10` เหมือนกันได้ เพราะเป็นคนละร้าน
2. `code` ต้องเป็นตัวพิมพ์ใหญ่และตัวเลขเท่านั้น (เช่น `SALE10` ผ่าน, `sale10` ไม่ผ่าน)
3. ช่วงวันที่ใช้งาน (`starts_on`/`ends_on`) ต้องเรียงลำดับถูกต้อง — ใช้ `DateRangeValidator`
   ที่เขียนไว้แล้วใน Step 314 มา reuse ตรงๆ โดยไม่แก้โค้ดของ validator เลย
4. ต้องมี unique index ระดับฐานข้อมูลรองรับ scope นี้ด้วย (ป้องกัน race condition ตามที่
   เรียนใน Step 316–317)

นอกจากนี้ ทีม customer support ต้องการฟอร์มติดต่อ (`ContactForm`) ที่**ไม่ต้องเก็บลงฐานข้อมูล
เลย** (ส่งอีเมลแจ้งทีมแล้วจบ) โดยมีเงื่อนไข: ต้องระบุชื่อ, ข้อความอย่างน้อย 10 ตัวอักษร, และ
ต้องระบุ**อีเมลหรือเบอร์โทรอย่างน้อยหนึ่งช่องทาง** (จะให้ทั้งคู่ก็ได้ แต่ต้องมีอย่างน้อยหนึ่ง)

### เฉลย

**1) สร้าง Model `PromoCode`**

```bash
bin/rails generate model PromoCode code:string store:string starts_on:date ends_on:date
bin/rails generate migration AddUniqueIndexToPromoCodes
```

```ruby
# db/migrate/..._add_unique_index_to_promo_codes.rb
class AddUniqueIndexToPromoCodes < ActiveRecord::Migration[8.1]
  def change
    add_index :promo_codes, [:store, :code], unique: true
  end
end
```

```bash
bin/rails db:migrate
```

**2) เขียน `app/models/promo_code.rb`**

```ruby
# app/models/promo_code.rb
class PromoCode < ApplicationRecord
  validates :code, presence: true,
                    format: { with: /\A[A-Z0-9]+\z/,
                              message: "ใช้ได้เฉพาะตัวพิมพ์ใหญ่และตัวเลขเท่านั้น" }
  validates :code, uniqueness: { scope: :store, case_sensitive: true }
  validates :store, presence: true

  validates_with DateRangeValidator, start_field: :starts_on, end_field: :ends_on

  def self.redeemable?(code:, store:)
    promo = find_by(code: code, store: store)
    return false unless promo

    today = Date.current
    promo.starts_on <= today && today <= promo.ends_on
  end
end
```

สังเกตว่า `validates_with DateRangeValidator, start_field: :starts_on, end_field: :ends_on`
คือบรรทัดเดียวกันเป๊ะกับที่ใช้ใน `Event` (Step 314) — นี่คือผลตอบแทนของการแยก validator
ออกมาเป็น class อิสระตั้งแต่แรก: Model ใหม่ที่มีคู่ field วันที่ต้องเรียงลำดับ **ใช้ซ้ำได้ทันที
โดยไม่ต้องเขียนหรือแก้ `DateRangeValidator` แม้แต่บรรทัดเดียว**

**3) ทดสอบผ่าน `rails console`**

```irb
irb(main):001> PromoCode.create!(code: "SALE10", store: "store-a",
irb(main):002>                    starts_on: Date.new(2026, 9, 1), ends_on: Date.new(2026, 9, 30))

irb(main):003> dup_same_store = PromoCode.new(code: "SALE10", store: "store-a",
irb(main):004>                                  starts_on: Date.new(2026, 10, 1),
irb(main):005>                                  ends_on: Date.new(2026, 10, 31))
irb(main):006> dup_same_store.valid?
=> false
irb(main):007> dup_same_store.errors.full_messages
=> ["Code has already been taken"]

irb(main):008> different_store = PromoCode.new(code: "SALE10", store: "store-b",
irb(main):009>                                    starts_on: Date.new(2026, 10, 1),
irb(main):010>                                    ends_on: Date.new(2026, 10, 31))
irb(main):011> different_store.valid?
=> true    # โค้ดเดียวกัน แต่คนละร้าน ไม่ถือว่าซ้ำ

irb(main):012> bad_format = PromoCode.new(code: "sale10", store: "store-c",
irb(main):013>                              starts_on: Date.new(2026, 1, 1), ends_on: Date.new(2026, 1, 31))
irb(main):014> bad_format.valid?
=> false
irb(main):015> bad_format.errors[:code]
=> ["ใช้ได้เฉพาะตัวพิมพ์ใหญ่และตัวเลขเท่านั้น"]

irb(main):016> bad_range = PromoCode.new(code: "SALE20", store: "store-c",
irb(main):017>                             starts_on: Date.new(2026, 3, 10), ends_on: Date.new(2026, 3, 1))
irb(main):018> bad_range.valid?
=> false
irb(main):019> bad_range.errors.full_messages
=> ["Ends on ต้องอยู่หลัง Starts on"]

irb(main):020> PromoCode.redeemable?(code: "SALE10", store: "store-a")
=> true
irb(main):021> PromoCode.redeemable?(code: "SALE10", store: "store-b")
=> false    # ยังไม่ถึงวันเริ่มใช้งาน
```

**4) เขียน `app/models/contact_form.rb`**

```ruby
# app/models/contact_form.rb
class ContactForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :name, :string
  attribute :email, :string
  attribute :phone, :string
  attribute :message, :string

  validates :name, presence: true
  validates :message, presence: true, length: { minimum: 10 }
  validates :email, email: true, allow_blank: true

  validate :email_or_phone_present

  def deliver
    return false unless valid?

    # แอปจริงจะเรียก ContactMailer.new_inquiry(...).deliver_later ตรงนี้ (สอนเต็มรูปแบบใน
    # Part 072)
    Rails.logger.info("[ContactForm] ส่งข้อความจาก #{name} (#{email.presence || phone})")
    true
  end

  private

  def email_or_phone_present
    return if email.present? || phone.present?

    errors.add(:base, "ต้องระบุอีเมลหรือเบอร์โทรอย่างน้อยหนึ่งช่องทาง")
  end
end
```

**5) ทดสอบ `ContactForm`**

```irb
irb(main):001> missing_both = ContactForm.new(name: "สมชาย", message: "สนใจสินค้าตัวนี้ครับ")
irb(main):002> missing_both.valid?
=> false
irb(main):003> missing_both.errors.full_messages
=> ["ต้องระบุอีเมลหรือเบอร์โทรอย่างน้อยหนึ่งช่องทาง"]
irb(main):004> missing_both.errors[:base]
=> ["ต้องระบุอีเมลหรือเบอร์โทรอย่างน้อยหนึ่งช่องทาง"]

irb(main):005> too_short_message = ContactForm.new(name: nil, message: "สั้น")
irb(main):006> too_short_message.valid?
=> false
irb(main):007> too_short_message.errors.full_messages
=> ["Name can't be blank", "Message is too short (minimum is 10 characters)",
    "ต้องระบุอีเมลหรือเบอร์โทรอย่างน้อยหนึ่งช่องทาง"]

irb(main):008> ok_with_email = ContactForm.new(name: "สมชาย", message: "สนใจสินค้าตัวนี้ครับ",
irb(main):009>                                   email: "user@example.com")
irb(main):010> ok_with_email.valid?
=> true
irb(main):011> ok_with_email.deliver
=> true

irb(main):012> ok_with_phone = ContactForm.new(name: "สมหญิง", message: "สนใจสินค้าค่ะ ติดต่อกลับด้วยนะคะ",
irb(main):013>                                    phone: "0891234567")
irb(main):014> ok_with_phone.valid?
=> true    # มีแค่เบอร์โทร ไม่มีอีเมล ก็ผ่าน เพราะเงื่อนไขคือ "อย่างน้อยหนึ่งช่องทาง"
```

**อธิบายจุดที่ควรสังเกตในเฉลยนี้:**

- `PromoCode` แสดงให้เห็นทั้ง 4 เทคนิคหลักของ Part นี้ในโมเดลเดียว: composite uniqueness
  (`scope: :store`), unique index ระดับฐานข้อมูลที่ตรงกับ scope เป๊ะๆ, reusable custom
  validator (`DateRangeValidator`), และ format validation ปกติที่เรียนจาก Part 026
- `case_sensitive: true` ใน `uniqueness` ของ `code` ระบุไว้อย่างชัดเจน (แม้จะเป็นค่า default
  ของ Rails 8 อยู่แล้ว) เพื่อสื่อสารเจตนาให้คนอ่านโค้ดเห็นชัดว่า `SALE10` กับ `sale10` ถือเป็น
  คนละโค้ดกัน — สอดคล้องกับ `format` validator ที่บังคับให้เป็นตัวพิมพ์ใหญ่อยู่แล้วเป็นทุนเดิม
- `ContactForm` ไม่มี `belongs_to`, ไม่มี migration ใดๆ เกี่ยวข้อง เป็น PORO ล้วนๆ ตามที่เรียน
  ใน Step 319 แต่ยังคงใช้ `validates`/`validate`/`errors.add(:base, ...)` ได้ครบเหมือน
  ActiveRecord ทุกประการ — สาธิตให้เห็นว่าเทคนิคทั้งหมดใน Part นี้ (รวมถึง cross-field
  validation จาก Step 313) ใช้ได้กับทั้ง ActiveRecord และ PORO โดยไม่มีข้อจำกัด

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม column `discount_percent:integer` ให้ `PromoCode` แล้วเขียน custom validator class
   ชื่อ `PercentageValidator < ActiveModel::EachValidator` ที่ตรวจว่าค่าต้องอยู่ระหว่าง 1–100
   เท่านั้น (ห้ามใช้ `numericality` ตรงๆ — จุดประสงค์ของแบบฝึกหัดนี้คือฝึกเขียน
   `ActiveModel::EachValidator` เอง) จากนั้นลองเอา validator ตัวนี้ไปใช้กับ field อื่นที่มี
   ความหมาย "เปอร์เซ็นต์" ในแอป (เช่น เพิ่ม field ทดลองใน `Event`) เพื่อพิสูจน์ว่า reuse ได้จริง
2. จำลอง race condition แบบ Step 316 ซ้ำอีกครั้ง แต่คราวนี้ใช้กับ `PromoCode.code` ที่ scope
   ด้วย `store` (ไม่ใช่ `Post.slug`) แล้วพิสูจน์ว่า unique index composite ที่เพิ่มไว้ในแบบฝึกหัด
   หลักสามารถป้องกันไม่ให้เกิดโค้ดส่วนลดซ้ำกันในร้านเดียวกันได้จริง (ใบ้: เขียน `rescue
   ActiveRecord::RecordNotUnique` ครอบการสร้าง `PromoCode` ในสถานการณ์จำลองนี้ด้วย)
3. เพิ่ม `config/locales/th.yml` ให้ `PromoCode` มีข้อความ error ภาษาไทยครบทุก field
   (`code`, `store`, `starts_on`, `ends_on`) ตามรูปแบบที่เรียนใน Step 320 แล้วทดสอบว่า
   `errors[:field]` และ `errors.full_messages` อ่านเป็นธรรมชาติทั้งคู่ โดยไม่มีคำซ้ำซ้อน
   (ระวังปัญหาแบบที่เจอกับ `post.title` ใน Step 320)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- สร้าง **reusable custom validator** ด้วย `ActiveModel::EachValidator` (ตรวจ field เดียว
  ใช้ซ้ำได้หลาย Model ผ่าน `validates :field, custom_name: true`) และ
  **`ActiveModel::Validator`** คู่กับ `validates_with` (ตรวจทั้งเรคคอร์ด เหมาะกับ cross-field
  validation ที่ต้องใช้ซ้ำ) — ทั้งสองแบบรับ `options` ให้ปรับพฤติกรรมได้ตามจุดที่เรียกใช้
- แยกแยะได้ว่าเมื่อไหร่ควรใช้ `validate :method_name` ธรรมดา (กฎเฉพาะ Model เดียว) เมื่อไหร่
  ควรยกระดับเป็น custom validator class (กฎที่ใช้ซ้ำหลายที่)
- `uniqueness: { scope: ... }` ทำ **composite uniqueness** ได้ (ไม่ซ้ำแบบมีเงื่อนไข เช่น
  slug ไม่ซ้ำเฉพาะภายในหมวดหมู่เดียวกัน) และ**พิสูจน์ด้วยของจริง**ว่า `uniqueness` validator
  เพียงอย่างเดียวมี **race condition** จริง ไม่ใช่แค่คำเตือนทางทฤษฎี
- ทางแก้ race condition ที่ถูกต้องคือต้องมี**ทั้งสองชั้น**เสมอ: `uniqueness` validator (UX ที่ดี)
  + **unique index ระดับฐานข้อมูล** (การรับประกันที่แท้จริง) พร้อม `rescue
  ActiveRecord::RecordNotUnique` เพื่อแปลง exception ให้เป็น error message ที่เป็นมิตร
- `validates_associated` ตรวจว่า associated object ก็ valid ด้วย แต่**ห้ามประกาศทั้งสองทาง
  ของความสัมพันธ์พร้อมกัน** เพราะทำให้เกิด infinite validation loop จนแอป crash ด้วย
  `SystemStackError`
- `ActiveModel::Model` + `ActiveModel::Attributes` ทำให้ **Plain Old Ruby Object (PORO)** มี
  ระบบ `validates`/`errors` แบบเดียวกับ ActiveRecord ได้ทั้งหมด — เปิดทางให้เขียน Form Object
  ที่ไม่ผูกกับฐานข้อมูล (จะเป็นทางการเต็มรูปแบบใน Part 082) และ reuse custom validator เดิม
  ข้ามขอบเขต ActiveRecord/PORO ได้ทันที
- `errors.add(:field, ...)` ใช้เมื่อชี้ได้ว่า field ไหนผิด ส่วน `errors.add(:base, ...)` ใช้
  เมื่อกฎเกี่ยวข้องกับความสัมพันธ์ของหลาย field พร้อมกันโดยไม่มีตัวไหนผิดเดี่ยวๆ
- ปรับแต่งข้อความ error ได้ 2 ระดับ: `message:` แบบ inline (เร็ว เจาะจงจุดเดียว) และ I18n
  locale file (`config/locales/th.yml`) ที่ scale ได้ทั้งแอป พร้อมรู้จักลำดับความสำคัญของ
  key ตั้งแต่เจาะจงที่สุดถึงทั่วไปที่สุด และข้อควรระวังเรื่องชื่อ field ซ้ำซ้อนในข้อความภาษาไทย

**ต่อไป (Part 033):** ตอนนี้ `Post` เชื่อมกับ `Category` ผ่าน `belongs_to`/`has_many` ธรรมดา
ซึ่งเป็นความสัมพันธ์แบบตรงไปตรงมาที่สุดเท่านั้น แอปจริงมักต้องการความสัมพันธ์ที่ซับซ้อนกว่านี้
มาก เช่น โพสต์หนึ่งอันมีได้หลาย tag และ tag หนึ่งอันก็ใช้กับหลายโพสต์ได้ (many-to-many),
หรือ comment หนึ่งอันอาจแปะได้ทั้งใต้ `Post` และใต้ `Product` โดยใช้ตาราง comments ตารางเดียว
(polymorphic) Part 033 จะพาไปรู้จัก **Association ขั้นสูง**: `has_many :through`,
`has_and_belongs_to_many`, และ **polymorphic association** พร้อมตัวอย่างการออกแบบฐานข้อมูล
จริงสำหรับแต่ละแบบ
