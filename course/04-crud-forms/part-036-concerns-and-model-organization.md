# Part 036: Concern (ActiveSupport::Concern) — การจัดโครงสร้างโมเดลขนาดใหญ่

> **Step ครอบคลุมใน Part นี้:** Step 351–360
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 010 เรื่อง Module/Mixin/MRO มาก่อน และ Part 035 เรื่อง
> Callback ขั้นสูงมาก่อน — Part นี้เริ่มต้นจากจุดที่ Part 035 ทิ้งไว้พอดี คือ Model ที่มี
> validation, callback, และ method จำนวนมากอัดอยู่ในไฟล์เดียว)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

Part 010 สอน Module ในฐานะ Mixin ไปแล้วอย่างละเอียด — `include`, `extend`, และ Method
Resolution Order (MRO) ทั้งหมดนั้นยังใช้ได้เป๊ะๆ ไม่มีอะไรเปลี่ยน Part นี้**จะไม่สอนเรื่องนั้นซ้ำ**
แต่จะพาไปดูว่าเมื่อเอา Module มาใช้กับ **ActiveRecord Model** จริง (ที่มี callback, validation,
scope ผสมกันเต็มไปหมด) Module แบบธรรมดาที่เรียนใน Part 010 มีจุดอ่อนบางอย่างที่ทำให้ Rails ต้อง
สร้างกลไกเสริมขึ้นมาชื่อ **`ActiveSupport::Concern`**

Part 035 จบท้ายไว้ด้วย `Order` Model ที่มี callback, validation, และ method หลายตัวรวมกันอยู่ใน
ไฟล์เดียว — Part นี้จะเริ่มจากปัญหานั้นตรงๆ: เมื่อ Model โตขึ้นเรื่อยๆ ควรแบ่งมันออกอย่างไร,
`ActiveSupport::Concern` ช่วยแก้ปัญหาอะไรได้จริง, และที่สำคัญไม่แพ้กันคือ **เมื่อไหร่ที่ Concern
แค่ซ่อนปัญหาไว้ใต้พรมโดยไม่ได้แก้อะไรเลย**

## สารบัญของ Part นี้

- Step 351: ปัญหา "Fat Model" ในแอปที่โตขึ้นเรื่อยๆ — Model จริงที่ validation, scope, callback,
  และ business logic ปนกันหมด
- Step 352: ทำไม Module ธรรมดาไม่พอ — ปัญหาลำดับการ `include` ที่ module หนึ่งต้องพึ่งอีก module
- Step 353: `ActiveSupport::Concern` — `extend ActiveSupport::Concern`, `included do`,
  `class_methods do`
- Step 354: สกัด `Sluggable` ใช้ร่วมกันจริงระหว่าง `Post` และ `Category`
- Step 355: สกัด `Searchable` — scope ที่ต้องพึ่งการตั้งค่าเฉพาะของแต่ละ Model
- Step 356: ธรรมเนียมของ `app/models/concerns/` และการทำ namespace ให้ concern เฉพาะโมเดล
- Step 357: Dependency ระหว่าง Concern และทำไม concern-chain ที่ลึกเกินไปถึงอ่านยาก
- Step 358: ข้อถกเถียงที่ไม่มีคำตอบเดียว — Concern คือ "code smell" จริงหรือไม่
- Step 359: ทางเลือกที่ดีกว่าสำหรับ business logic จริง — Service Object, Value Object,
  Query Object (preview Part 082–083)
- Step 360: แบบฝึกหัด — สกัด `Sluggable` และ `Archivable` ใช้ร่วมกันระหว่างสองโมเดล

---

## เตรียมโดเมนสำหรับ Part นี้

Part นี้ใช้โดเมนต่อเนื่องจาก Part 034/035 (`Author`, `Category`, `Post`, `Comment`) แต่เพิ่ม
column ที่จำเป็นสำหรับสาธิตเรื่อง slug และ archive:

```bash
rails new concerns_demo --minimal
cd concerns_demo

bin/rails generate model Author name:string nationality:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string body:text slug:string published:boolean \
  views:integer archived_at:datetime author:references category:references
bin/rails generate model Comment commenter:string body:text post:references

# Category ในตอนแรกยังไม่มี slug/archived_at เพิ่มทีหลังด้วย migration แยก
# (สถานการณ์นี้จำลองของจริง: ตอนสร้าง Category ครั้งแรกไม่มีใครคิดว่าจะต้องมี slug)
bin/rails generate migration AddSlugAndArchivedAtToCategories slug:string archived_at:datetime
bin/rails generate migration AddIndexOnSlugToPostsAndCategories
```

แก้ migration ของ `Post` ให้ `published`/`views` มี default ระดับฐานข้อมูล (ตามแนวทาง Part 026/034):

```ruby
# db/migrate/..._create_posts.rb
t.boolean :published, default: false, null: false
t.integer :views, default: 0, null: false
```

เพิ่ม unique index ให้ `slug` ทั้งสองตาราง:

```ruby
# db/migrate/..._add_index_on_slug_to_posts_and_categories.rb
class AddIndexOnSlugToPostsAndCategories < ActiveRecord::Migration[8.1]
  def change
    add_index :posts, :slug, unique: true
    add_index :categories, :slug, unique: true
  end
end
```

```bash
bin/rails db:create db:migrate
```

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :posts
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :posts
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

---

## Step 351: ปัญหา "Fat Model" ในแอปที่โตขึ้นเรื่อยๆ

Part 026 เคยเตือนสั้นๆ ไว้แล้วว่าอย่ายัด business logic ลง callback มั่วซั่ว แต่ในงานจริงแม้จะ
ระวังแล้ว **`Post` Model ก็ยังโตขึ้นเรื่อยๆ ตามธรรมชาติของฟีเจอร์ที่เพิ่มเข้ามาทีละนิด** — ทุก
ฟีเจอร์ดูสมเหตุสมผลเดี่ยวๆ ในตัวมันเอง แต่พอกองรวมกันในไฟล์เดียว ผลลัพธ์คือ Model ที่อ่านยากมาก

```ruby
# app/models/post.rb — เวอร์ชัน "fat model" ที่โตมาตามธรรมชาติ ทีละ feature
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true
  validates :slug, uniqueness: true, allow_nil: true

  before_validation :strip_whitespace
  before_save :assign_slug
  before_save :ensure_excerpt

  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { where("views > ?", 100) }
  scope :not_archived, -> { where(archived_at: nil) }
  scope :search, lambda { |keyword|
    where("title LIKE :kw OR body LIKE :kw", kw: "%#{keyword}%")
  }

  BANNED_WORDS = %w[spam scam].freeze

  def to_param
    slug.presence || id.to_s
  end

  def reading_time_minutes
    words = body.to_s.split.size
    (words / 200.0).ceil.clamp(1, Float::INFINITY).to_i
  end

  def excerpt(length: 140)
    body.to_s.truncate(length)
  end

  def archived?
    archived_at.present?
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def contains_profanity?
    BANNED_WORDS.any? { |word| body.to_s.downcase.include?(word) }
  end

  private

  def strip_whitespace
    self.title = title.to_s.strip
    self.body = body.to_s.strip
  end

  def assign_slug
    return if title.blank?
    return if slug.present?

    base = title.parameterize
    base = "post" if base.blank?
    self.slug = "#{base}-#{SecureRandom.hex(3)}"
  end

  def ensure_excerpt
    true # เผื่ออนาคตจะมี column excerpt แยกต่างหาก
  end
end
```

ทดสอบจริงว่าไฟล์นี้ทำงานถูกต้องทุกจุดผ่าน `bin/rails runner`:

```ruby
author = Author.create!(name: "Somchai", nationality: "Thai")
category = Category.create!(name: "Ruby")

post = Post.create!(
  title: "  Learning Ruby Concerns  ",
  body: "This is a normal post body about Ruby. " * 30,
  author: author, category: category, views: 150
)

puts "slug: #{post.slug}"
puts "reading_time: #{post.reading_time_minutes} min"
puts "excerpt: #{post.excerpt(length: 40)}"
post.archive!
puts "archived?: #{post.archived?}"
```

ผลลัพธ์จริง:

```
slug: learning-ruby-concerns-80d077
reading_time: 2 min
excerpt: This is a normal post body about Ruby...
archived?: true
```

**โค้ดนี้ทำงานถูกต้องสมบูรณ์ ไม่มีบัคแม้แต่จุดเดียว** — และนั่นคือเหตุผลที่ fat model เป็นปัญหา
ที่ค่อยๆ สะสม ไม่มีจุดไหนที่ "เห็นชัดว่าผิด" ในตอนที่เขียน แต่ปัญหาจริงจะโผล่มาทีหลังในรูปแบบนี้:

1. **`assign_slug` ใช้ซ้ำใน `Category` ไม่ได้เลย** — ถ้า `Category` ก็ต้องการ slug เหมือนกัน
   (ซึ่งเป็นเรื่องปกติมากในแอปจริง) ทางเลือกเดียวตอนนี้คือ **copy โค้ด 6 บรรทัดนี้ไปวางซ้ำ** ใน
   `category.rb` — ผิดหลัก DRY ที่เรียนมาตั้งแต่ Part 001 ตรงๆ
2. **ทดสอบ `reading_time_minutes` อย่างเดียวต้องสร้าง `Author`/`Category` ก่อนเสมอ** เพราะ
   `belongs_to` แบบไม่มี `optional: true` บังคับให้ต้องมี association ครบก่อนถึงจะ valid —
   ทั้งที่ logic การนับเวลาอ่านไม่เกี่ยวอะไรกับ author/category เลยแม้แต่นิดเดียว
3. **คนใหม่เข้าทีมเปิดไฟล์นี้มาแล้วงงว่า "ตรงไหนคือ core ของ Post จริงๆ"** — slug, search,
   archive, reading time, profanity check ปนกันหมดโดยไม่มีการจัดกลุ่มใดๆ ให้เห็นเลยว่าอะไรเกี่ยว
   กับอะไร
4. **ไฟล์นี้จะยาวขึ้นเรื่อยๆ แบบไม่มีเพดาน** — ทุกฟีเจอร์ใหม่ "แค่เพิ่มอีก 5-10 บรรทัด" ดูไม่เยอะ
   ทีละครั้ง แต่พอผ่านไปหลายเดือนไฟล์อาจยาวเกิน 300-400 บรรทัด

> **ข้อสังเกตสำคัญ:** สังเกตว่าปัญหาทั้ง 4 ข้อนี้**ไม่มีข้อไหนเลยที่เป็นเรื่อง performance หรือ
> ความถูกต้องของโค้ด** — โค้ดข้างบนรันได้ถูกต้อง 100% ปัญหาทั้งหมดเป็นเรื่อง **maintainability**
> ล้วนๆ ซึ่งเป็นสิ่งที่มองไม่เห็นในวันที่เขียน แต่จะเจ็บปวดในอีก 6 เดือนข้างหน้าเมื่อทีมโตขึ้นและ
> ต้องแก้ไขไฟล์เดียวกันพร้อมกันหลายคน

ทางแก้ตามสัญชาตญาณคือ "เอา `assign_slug`/`to_param` ไปทำเป็น Module แบบที่เรียนใน Part 010"
มาลองทำดูจริงๆ ใน Step ถัดไป — และจะเจอปัญหาที่ไม่คาดคิดทันที

---

## Step 352: ทำไม Module ธรรมดาไม่พอ — ปัญหาลำดับการ `include`

ลองแยก logic เป็น 2 module ตามความรับผิดชอบ: `PlainSearchable` (จัดการ scope ค้นหา) และ
`PlainSluggable` (จัดการ slug) โดยให้ `PlainSluggable` "รู้จัก" field ที่ `PlainSearchable`
เตรียมไว้ (สถานการณ์จำลองว่า concern หนึ่งต้องพึ่งอีก concern หนึ่ง ซึ่งเกิดขึ้นบ่อยมากในของจริง)

```ruby
module PlainSearchable
  def self.included(base)
    base.class_attribute :searchable_fields, default: []
  end
end

module PlainSluggable
  def self.included(base)
    base.searchable_fields += [:slug]   # ต้องการให้ slug ถูกนับเป็นช่องค้นหาด้วย
  end
end
```

ถ้า include ตามลำดับที่ "ถูกต้อง" (Searchable ก่อน Sluggable) จะได้ผลลัพธ์ตามที่ตั้งใจ:

```ruby
class GoodOrder
  include PlainSearchable
  include PlainSluggable
end

GoodOrder.searchable_fields
# => [:slug]
```

แต่ถ้ามีใครในทีม (หรือแม้แต่ตัวเราเองในอีก 3 เดือนข้างหน้า) เขียนสลับลำดับกันโดยไม่ได้ตั้งใจ —
ซึ่งไม่มีอะไรเตือนเลยว่าลำดับสำคัญ เพราะโค้ด `include` สองบรรทัดนี้ **ดูเหมือนเป็นอิสระต่อกัน
โดยสมบูรณ์**:

```ruby
class BadOrder
  include PlainSluggable    # สลับลำดับ! Sluggable มาก่อน Searchable
  include PlainSearchable
end
```

```
NoMethodError: undefined method 'searchable_fields' for class BadOrder
```

**พังทันทีตอนโหลดไฟล์** — และข้อความ error ก็ไม่ได้บอกเลยว่า "ต้อง include `PlainSearchable`
ก่อน" มันแค่บอกว่าไม่มี method `searchable_fields` ให้ ซึ่งทำให้ debug ยากมากถ้าไม่รู้จุดที่แท้จริง
มาก่อน

### ลองแก้ด้วยการ "ประกาศ dependency ไว้ในตัว module เอง"

แนวคิดที่ดูสมเหตุสมผลคือ ถ้า `PlainSluggable` ต้องพึ่ง `PlainSearchable` เสมอ ก็ให้
`PlainSluggable` `include PlainSearchable` ไว้ในตัวเองไปเลย จะได้ไม่ต้องพึ่งให้ผู้ใช้จำลำดับ:

```ruby
module PlainSluggable2
  include PlainSearchable2   # พยายามแก้ปัญหาโดยประกาศ dependency ไว้ในตัวเอง

  def self.included(base)
    base.searchable_fields += [:slug]
  end
end
```

```ruby
class NaiveFix
  include PlainSluggable2
end
```

```
undefined method 'class_attribute' for module PlainSluggable2 (NoMethodError)
    base.class_attribute :searchable_fields, default: []
        ^^^^^^^^^^^^^^^^
```

**พังแรงกว่าเดิมอีก** — คราวนี้ error เกิดขึ้น**ตอนนิยาม `module PlainSluggable2`เอง** (ก่อนที่
`NaiveFix` จะแตะโค้ดนี้ด้วยซ้ำ) เพราะเหตุผลที่ลึกกว่าที่คิด: มาดูสาเหตุที่แท้จริงด้วยตัวอย่างที่
เรียบง่ายที่สุดเท่าที่จะทำได้

```ruby
module B
  def self.included(base)
    puts "B included into: #{base}"
  end
end

module A
  include B          # A พึ่ง B โดยประกาศไว้ในตัวเอง
  def self.included(base)
    puts "A included into: #{base}"
  end
end

class C
  include A
end
```

```
B included into: A
A included into: C
```

**นี่คือรากของปัญหา:** เมื่อ `A` เขียน `include B` ไว้ในตัวเอง Ruby จะเรียก `B.included(base)`
**ทันที ณ ตอนนิยาม `module A`** — และ `base` ที่ส่งเข้าไปคือ **`A` (ตัว module เอง)**
**ไม่ใช่ `C`** เพราะ ณ ตอนนั้น Ruby ยังไม่รู้เลยว่า `A` จะถูกเอาไป `include` ใน `C` ในอนาคต
`self.included` ของ plain Ruby module รู้จักแค่ **"ใครเพิ่งเรียก `include` ฉันโดยตรง"** เท่านั้น
ไม่สามารถมองทะลุไปถึง class ปลายทางที่แท้จริงได้เลย

นี่คือเหตุผลที่ `base.class_attribute` (ตัวอย่างจริงจาก `NaiveFix`) พังตั้งแต่ยังไม่ทันสร้าง
class ใดๆ เลย — เพราะ `base` ในตอนนั้นคือ `PlainSluggable2` (module ธรรมดา) ซึ่งไม่มี method
`class_attribute` ให้เรียก (`class_attribute` เป็น method ที่ ActiveSupport เพิ่มให้ class เท่านั้น)

> **สรุปปัญหาของ Module ธรรมดาใน Step นี้:** (1) ถ้า module หนึ่งพึ่งอีก module หนึ่ง ผู้ที่
> `include` ต้องจำลำดับให้ถูกต้องเอง ไม่มีกลไกบังคับหรือเตือนใดๆ และ (2) ถ้าพยายามแก้โดยให้
> module ประกาศ dependency ของตัวเอง (`include OtherModule` ในตัว module) จะพังทันทีเพราะ
> `self.included(base)` ได้ `base` ที่ผิด (ได้ module ตัวกลาง ไม่ใช่ class ปลายทาง) — ปัญหานี้
> ชัดเจนที่สุดเมื่อ `included` hook ต้องเรียก method ระดับ class ของ ActiveRecord/ActiveSupport
> (เช่น `class_attribute`, `validates`, `scope`, `has_many`) ซึ่งเป็นสถานการณ์ปกติมากในโลก Rails
> จริง — นี่คือช่องว่างที่ `ActiveSupport::Concern` ถูกสร้างขึ้นมาเพื่ออุดพอดี

---

## Step 353: `ActiveSupport::Concern` — `extend ActiveSupport::Concern`, `included do`, `class_methods do`

`ActiveSupport::Concern` เป็น module ที่มากับ Rails (มาจาก gem `activesupport`) ใช้แก้ปัญหาทั้ง
สองข้อจาก Step 352 พร้อมกันในคราวเดียว วิธีใช้คือ `extend ActiveSupport::Concern` ที่หัว module
แล้วเปลี่ยนวิธีเขียน 2 จุด:

1. **`included do ... end`** — แทนที่ `def self.included(base); base.class_eval do ... end; end`
   ของ plain Ruby โค้ดข้างในบล็อกนี้จะถูกรันโดยมี **`self` เป็น host class ที่แท้จริงเสมอ**
   (ไม่ว่าจะถูก include ผ่าน module ตัวกลางกี่ชั้นก็ตาม)
2. **`class_methods do ... end`** — นิยาม method ที่จะกลายเป็น **class method** ของ host
   (เทียบเท่ากับ `extend` ของ Part 010 Step 97 แต่เขียนสะดวกกว่า ไม่ต้องแยก module ลูกเอง)

### แก้ปัญหาจาก Step 352 ด้วย Concern

```ruby
module ConcernSearchable
  extend ActiveSupport::Concern

  included do
    class_attribute :searchable_fields, default: []
  end
end

module ConcernSluggable
  extend ActiveSupport::Concern

  include ConcernSearchable   # ประกาศ dependency ไว้ตรงนี้ที่เดียว ปลอดภัยแล้ว

  included do
    self.searchable_fields += [:slug]
  end
end

class FixedOrder
  include ConcernSluggable    # include ตัวที่ "ต้องพึ่ง" อีกตัว โดยไม่ต้องกังวลเรื่องลำดับเลย
end

FixedOrder.searchable_fields
# => [:slug]

FixedOrder.ancestors.first(4)
# => [FixedOrder, ConcernSluggable, ConcernSearchable, ActiveSupport::Dependencies::RequireDependency]
```

ทดสอบจริงยืนยันว่า **`FixedOrder.searchable_fields` ได้ค่า `[:slug]` ถูกต้อง** ทั้งที่เขียน
`include ConcernSluggable` เพียงบรรทัดเดียว ไม่ต้องกังวลเรื่องลำดับเหมือน Step 352 เลย

**กลไกเบื้องหลัง (โดยสรุป ไม่ลงลึกถึงระดับ source code):** `ActiveSupport::Concern` เก็บรายการ
module ที่ถูก `include` ไว้ในตัวมันเอง (`ConcernSearchable` ในที่นี้) ไว้เป็น **dependency list**
แทนที่จะรันทันทีแบบ plain Ruby เมื่อ `FixedOrder` `include ConcernSluggable` จริงๆ Concern จะ
เช็คก่อนว่ามี dependency ที่ `FixedOrder` ยังไม่ได้ include หรือไม่ ถ้ามีก็จะ **`include` ให้
`FixedOrder` โดยตรงก่อน** (ให้ `ConcernSearchable`'s `included do` รันด้วย `self == FixedOrder`
จริงๆ) แล้วค่อยรัน `included do` ของ `ConcernSluggable` เอง ตามลำดับ — แก้ปัญหา "base ผิดตัว"
จาก Step 352 ได้ตรงจุดพอดี

### `class_methods do` — เพิ่ม class method อย่างเป็นทางการ

```ruby
module Trackable
  extend ActiveSupport::Concern

  included do
    class_attribute :tracked_events, default: []
  end

  class_methods do
    def track(event_name)
      self.tracked_events += [event_name]
    end
  end

  def tracking_summary
    "#{self.class.name} ติดตามอยู่ #{self.class.tracked_events.size} เหตุการณ์: " \
      "#{self.class.tracked_events.join(', ')}"
  end
end

class Widget
  include Trackable

  track :created
  track :updated
end

Widget.new.tracking_summary
```

```
Widget ติดตามอยู่ 2 เหตุการณ์: created, updated
```

ทดสอบยืนยันเพิ่มเติม:

```ruby
Widget.respond_to?(:track)              # => true (class method จาก class_methods do)
Widget.new.respond_to?(:tracking_summary) # => true (instance method จาก def ธรรมดาในตัว module)
```

**สังเกตความแตกต่างของ 3 ส่วนใน `ActiveSupport::Concern`:**

| ส่วนของโค้ด | ผลลัพธ์ | เทียบเท่า Plain Ruby (Part 010) |
|---|---|---|
| `def some_method` เขียนตรงๆ ในตัว module | instance method ของ host | เหมือน `include` ธรรมดา |
| `included do ... end` | โค้ดที่รันครั้งเดียวตอน include จริง โดย `self` คือ host class | เหมือน `def self.included(base); base.class_eval { ... }; end` **แต่ `base` ถูกต้องเสมอ** |
| `class_methods do ... end` | method ที่กลายเป็น class method ของ host | เหมือนเขียน submodule แยก แล้ว `base.extend(SubModule)` เอง |

> **ข้อควรรู้:** `ActiveSupport::Concern` ไม่ใช่ syntax พิเศษของ Ruby — มันเป็น**module ธรรมดา
> ตัวหนึ่ง**ที่ override method `append_features`/`included` ของ `Module` ให้ฉลาดขึ้น ทุกอย่างที่
> เรียนจาก Part 010 (ancestors, MRO, `include` vs `extend`) ยังใช้ได้เป๊ะกับ Concern ทุกประการ —
> Concern แค่**แก้ปัญหาการจัดลำดับ dependency**ที่ Step 352 แสดงให้เห็น ไม่ได้เปลี่ยนกลไก mixin
> พื้นฐานของ Ruby แต่อย่างใด

---

## Step 354: สกัด `Sluggable` ใช้ร่วมกันจริงระหว่าง `Post` และ `Category`

ตอนนี้พร้อมแก้ปัญหาข้อ 1 จาก Step 351 แล้ว: สกัด logic การสร้าง slug ออกมาเป็น Concern ที่ใช้ได้
ทั้ง `Post` และ `Category`

```ruby
# app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern

  included do
    validates :slug, uniqueness: true, allow_nil: true

    before_validation :assign_slug, if: -> { slug.blank? && slug_source.present? }
  end

  def to_param
    slug.presence || id.to_s
  end

  private

  # โมเดลที่ include Sluggable ต้องมี method นี้ (หรือ column title/name)
  # นี่คือ "implicit interface" แบบเดียวกับที่ Part 010 Step 96 อธิบายไว้เรื่อง mixin
  def slug_source
    respond_to?(:title) ? title : name
  end

  def assign_slug
    base = slug_source.to_s.parameterize
    base = self.class.name.underscore if base.blank?
    self.slug = "#{base}-#{SecureRandom.hex(3)}"
  end
end
```

ใช้งานใน `Post`:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Sluggable

  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true
end
```

และใน `Category` — Model ที่**ไม่มีความสัมพันธ์แบบ inheritance ใดๆ กับ `Post` เลย** แต่ก็ใช้
Concern ตัวเดียวกันได้ทันที:

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  include Sluggable

  has_many :posts

  validates :name, presence: true
end
```

ทดสอบจริงผ่าน `bin/rails runner`:

```ruby
category = Category.create!(name: "Ruby on Rails")
puts "category.slug: #{category.slug}"
puts "category.to_param: #{category.to_param}"

author = Author.create!(name: "Somchai", nationality: "Thai")
post = Post.create!(title: "  Learning ActiveSupport::Concern  ",
                     body: "Concern content about Ruby and Rails " * 20,
                     author: author, category: category)
puts "post.slug: #{post.slug}"
puts "post.to_param: #{post.to_param}"
```

ผลลัพธ์จริง:

```
category.slug: ruby-on-rails-91a30f
category.to_param: ruby-on-rails-91a30f
post.slug: learning-activesupport-concern-a922bf
post.to_param: learning-activesupport-concern-a922bf
```

**สิ่งที่เพิ่งเกิดขึ้นคือชัยชนะของ DRY ตัวจริง:** โค้ด `assign_slug`/`slug_source`/`to_param`
ถูกเขียนไว้ **ที่เดียว** ในไฟล์เดียว แล้วทั้ง `Post` และ `Category` — ที่ไม่มีความสัมพันธ์แบบ
`is-a` ต่อกันเลยแม้แต่นิดเดียว — ได้พฤติกรรมเดียวกันเป๊ะๆ โดยไม่ต้อง copy-paste ตรงตามหลักการ
mixin จาก Part 010 Step 96 (แก้ปัญหาที่ single inheritance ทำไม่ได้) เพียงแต่คราวนี้เขียนด้วย
syntax ของ `ActiveSupport::Concern` ที่ทนทานต่อปัญหาลำดับ `include` ตามที่พิสูจน์ใน Step 352–353

ตรวจสอบ `ancestors` ยืนยันว่ากลไกเบื้องหลังยังเป็น mixin ธรรมดาทุกประการ (ทบทวน MRO จาก
Part 010 Step 98):

```ruby
Post.ancestors.select { |m| m.to_s == "Sluggable" }
# => [Sluggable]

Post.instance_method(:to_param).owner
# => Sluggable   (ยืนยันว่า to_param มาจาก Concern จริง ไม่ใช่ ActiveRecord::Base เดิม)
```

---

## Step 355: สกัด `Searchable` — scope ที่ต้องพึ่งการตั้งค่าเฉพาะของแต่ละ Model

`Sluggable` ใน Step 354 ใช้ **convention** (เดาว่า Model มี `title` หรือ `name`) เพื่อทำงานแบบ
"zero-config" แต่ Concern บางตัวไม่สามารถเดาแบบนั้นได้ — การค้นหา (`search`) ต้องรู้ชัดเจนว่า
"คอลัมน์ไหนบ้างที่ควรค้นหา" ซึ่งต่างกันไปในแต่ละ Model จริงๆ (`Post` ค้นทั้ง `title`/`body`,
`Category` ค้นแค่ `name`) — นี่คือกรณีที่ต้องใช้ **`class_methods do`** ร่วมกับการบังคับให้ Model
ผู้ include ต้อง "ประกาศ" ค่า config ของตัวเอง

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  class_methods do
    def search(keyword)
      return all if keyword.blank?

      columns = searchable_columns.map { |col| "#{table_name}.#{col} LIKE :kw" }.join(" OR ")
      where(columns, kw: "%#{keyword}%")
    end

    # แต่ละ Model ที่ include Searchable ต้อง override method นี้ ระบุว่าจะค้นหาคอลัมน์ไหนบ้าง
    def searchable_columns
      raise NotImplementedError, "#{name} ต้องนิยาม self.searchable_columns"
    end
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Sluggable
  include Searchable
  # ...

  def self.searchable_columns
    %w[title body]
  end
end

# app/models/category.rb
class Category < ApplicationRecord
  include Sluggable
  include Searchable
  # ...

  def self.searchable_columns
    %w[name]
  end
end
```

ทดสอบจริง:

```ruby
Post.search("Concern").to_a.map(&:title)
# => ["Learning ActiveSupport::Concern"]

Category.search("Rails").to_a.map(&:name)
# => ["Ruby on Rails"]

Post.search("").to_sql
# => SELECT "posts".* FROM "posts"   (keyword ว่าง -> คืน .all เฉยๆ ไม่ error)
```

ถ้าลืม override `searchable_columns` — ทดสอบด้วย `Comment` ที่ include `Searchable` แต่ไม่ได้
เขียน `self.searchable_columns`:

```ruby
class Comment < ApplicationRecord
  include Searchable
  belongs_to :post
end

Comment.search("x")
```

```
NotImplementedError: Comment ต้องนิยาม self.searchable_columns
```

**Error message บอกชัดเจนตรงจุด** ว่าต้องแก้อะไร — นี่คือรูปแบบการออกแบบ Concern ที่ดี:
เมื่อ Concern ต้องพึ่งข้อมูลเฉพาะของแต่ละ Model ให้ **บังคับด้วย `NotImplementedError`** แทนที่
จะปล่อยให้พฤติกรรมผิดเงียบๆ (เช่น คืน Array ว่างโดยไม่มีใครสังเกต) — หลักการเดียวกับ abstract
method ใน OOP ภาษาอื่น แม้ Ruby จะไม่มี syntax บังคับแบบ `abstract` ตรงๆ ก็ตาม

> **เทียบกับ `Sluggable`:** `Sluggable` ใช้ **convention over configuration** (เดาจาก
> `respond_to?(:title)`) เพราะ 90% ของ Model ที่ต้องการ slug มักมี `title`/`name` อยู่แล้ว ในขณะ
> ที่ `Searchable` ใช้ **explicit configuration** (บังคับให้ override `searchable_columns`)
> เพราะคอลัมน์ที่ควรค้นหาต่างกันมากเกินกว่าจะเดาได้ถูกทุกครั้ง — การเลือกว่า Concern ตัวไหนควรใช้
> แนวทางไหนเป็นการตัดสินใจออกแบบที่ต้องดูเนื้อหาจริงของแต่ละกรณี ไม่มีกฎตายตัว

---

## Step 356: ธรรมเนียมของ `app/models/concerns/` และการทำ Namespace

Rails สร้างโฟลเดอร์ `app/models/concerns/` (และ `app/controllers/concerns/`) ให้อัตโนมัติตั้งแต่
`rails new` และเพิ่มเข้า **autoload path** ให้แล้วโดยไม่ต้องตั้งค่าอะไรเพิ่มเอง — ตรวจสอบได้จริง:

```ruby
Rails.autoloaders.main.dirs.grep(/concerns/)
```

```
["/.../app/controllers/concerns", "/.../app/models/concerns"]
```

**กฎการตั้งชื่อไฟล์/module ตาม Zeitwerk** (ตัวโหลดไฟล์อัตโนมัติของ Rails ที่เจาะลึกใน Phase 15)
เหมือนไฟล์ Ruby ทั่วไปในแอป: ชื่อไฟล์ตรงกับชื่อ module/class แบบ `snake_case` ↔ `CamelCase`

### Concern ที่ใช้ร่วมกันหลาย Model — วางตรงๆ ใน `concerns/`

`Sluggable`/`Searchable` ที่ทำไปแล้วเป็นตัวอย่างที่ถูกต้องของกรณีนี้ — ใช้ร่วมกันได้จริงข้ามหลาย
Model จึงวางไฟล์ตรงๆ ที่ `app/models/concerns/sluggable.rb`, `app/models/concerns/searchable.rb`
โดยไม่ต้อง namespace ใดๆ

### Concern ที่ใช้กับ Model เดียวเท่านั้น — namespace ด้วยชื่อ Model

บางครั้ง Concern ไม่ได้มีไว้เพื่อ**แชร์ข้ามหลาย Model** แต่มีไว้เพื่อ**แบ่งไฟล์ที่ยาวเกินไปของ
Model เดียว**ออกเป็นส่วนๆ ตามความรับผิดชอบ (เช่น แยก logic การแจ้งเตือนของ `Post` ออกจากไฟล์หลัก
แม้จะไม่มี Model อื่นใช้ซ้ำเลยก็ตาม) — กรณีนี้ Rails แนะนำให้ **namespace ด้วยชื่อ Model** เพื่อ
สื่อสารชัดเจนว่า "นี่ไม่ใช่ของใช้ร่วม เป็นส่วนหนึ่งของ `Post` เท่านั้น"

```ruby
# app/models/concerns/post/notifiable.rb
module Post::Notifiable
  extend ActiveSupport::Concern

  included do
    after_create_commit :notify_subscribers
  end

  private

  def notify_subscribers
    Rails.logger.info("[Post##{id}] แจ้งผู้ติดตามว่ามีบทความใหม่: #{title}")
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Sluggable
  include Searchable
  include Post::Notifiable
  # ...
end
```

ทดสอบจริงผ่าน `bin/rails runner` (ยืนยันว่า Zeitwerk resolve `Post::Notifiable` จากตำแหน่งไฟล์
`app/models/concerns/post/notifiable.rb` ได้ถูกต้องอัตโนมัติ ไม่ต้อง `require` เอง):

```ruby
post = Post.create!(title: "Namespace Concern Test", body: "Body text here " * 10,
                     author: author, category: category)
```

ตรวจ `log/development.log`:

```
[Post#1] แจ้งผู้ติดตามว่ามีบทความใหม่: Namespace Concern Test
```

ทำงานถูกต้อง — `include Post::Notifiable` เขียนแบบมีจุด `::` ให้เห็นชัดเจนในตัว `post.rb` เองว่า
"concern นี้เป็นของ `Post` โดยเฉพาะ ไม่ใช่ของใช้ร่วม" ต่างจาก `include Sluggable`/
`include Searchable` ที่ไม่มี namespace บอกให้รู้ทันทีว่าเป็นของใช้ร่วมข้าม Model

> **กฎตัดสินใจในทางปฏิบัติ:** ถามตัวเองว่า **"ถ้ามี Model ตัวที่สองที่ไม่ใช่ `Post` ต้องการ
> พฤติกรรมนี้ จะ `include` ไฟล์นี้ตรงๆ ได้เลยไหม โดยไม่ต้องแก้อะไรในไฟล์นี้เลย"** — ถ้าใช่ ให้
> วางตรงๆ ไม่ต้อง namespace (`Sluggable`, `Searchable`) — ถ้าคำตอบคือ "ไม่ได้ เพราะมันผูกกับ
> ความหมายของ `Post` โดยเฉพาะ" ให้ namespace ด้วยชื่อ Model (`Post::Notifiable`) เพื่อสื่อสาร
> เจตนานั้นให้คนอ่านโค้ดเห็นชัดตั้งแต่บรรทัดแรกที่เจอชื่อมัน

---

## Step 357: Dependency ระหว่าง Concern และทำไม Concern-chain ที่ลึกเกินไปถึงอ่านยาก

Step 353 แสดงวิธีประกาศ dependency ด้วยการ `include OtherConcern` ที่**ระดับ module** (ก่อน
`included do`) ไปแล้ว — แต่มีอีกรูปแบบหนึ่งที่พบในโค้ด Rails จริงเหมือนกัน คือการ `include` ไว้
**ข้างใน `included do`**

```ruby
module DepB
  extend ActiveSupport::Concern
  included do
    class_attribute :from_b, default: true
  end
end

module DepA
  extend ActiveSupport::Concern
  included do
    include DepB   # include ผ่าน included do แทนที่จะ include ตรงๆ ที่ module body
  end
end

class Host
  include DepA
end

Host.from_b
# => true
```

ทดสอบจริงพิสูจน์ว่าทำงานได้เหมือนกัน (เพราะโค้ดใน `included do` รันด้วย `self == Host` อยู่แล้ว
`include DepB` ข้างในจึงเทียบเท่ากับ `Host.include(DepB)` ตรงๆ) แต่มีจุดต่างที่ละเอียดอ่อนซ่อนอยู่
— ลอง compare `ancestors` ของทั้งสองแบบ:

```ruby
# แบบ include ที่ module body (Step 353)
FixedOrder.ancestors.first(3)
# => [FixedOrder, ConcernSluggable, ConcernSearchable]
#    (ตัวที่ "พึ่ง" อยู่ใกล้ host กว่า ตัวที่ "ถูกพึ่ง" อยู่ไกลกว่า)

# แบบ include ใน included do (Step 357)
Host.ancestors.first(3)
# => [Host, DepB, DepA]
#    (สลับกัน! ตัวที่ "ถูกพึ่ง" (DepB) กลับมาอยู่ใกล้ host กว่าตัวที่ "พึ่ง" (DepA))
```

ความต่างนี้มีผลจริงถ้าทั้งสอง module มี method ชื่อซ้ำกัน — ตาม MRO ที่เรียนใน Part 010 Step 98
**module ที่อยู่ใกล้ host กว่าจะถูกเรียกก่อนเสมอ** ดังนั้นแบบแรก (`include` ที่ module body)
`DepA`/`ConcernSluggable` (ตัวเฉพาะเจาะจงกว่า) จะชนะเมื่อ method ชนกัน ซึ่งตรงกับสัญชาตญาณทั่วไป
("ตัวที่ต่อยอดควร override ตัวฐาน") ในขณะที่แบบที่สอง (`include` ใน `included do`) กลับให้ผล
ตรงข้าม

> **แนวปฏิบัติที่แนะนำ:** ประกาศ dependency ด้วย **`include OtherConcern` ที่ module body ก่อน
> `included do`** เสมอ (แบบ Step 353) เพราะให้ลำดับ ancestors ที่คาดเดาได้ตรงกับสัญชาตญาณ —
> รูปแบบ `included do; include OtherConcern; end` มีใช้ในโค้ดบางที่จริง (มักเจอตอนต้องการ
> เงื่อนไขก่อน include เช่น `include OtherConcern if some_condition`) แต่ควรรู้ผลข้างเคียงเรื่อง
> ลำดับ ancestors ที่เปลี่ยนไปก่อนเลือกใช้

### ทำไม concern-chain ที่ลึกเกินไปถึงอ่านยาก

ลองจินตนาการสถานการณ์ที่ concern พึ่งกันเป็นทอดๆ: `Auditable` พึ่ง `Trackable` ซึ่งพึ่ง
`Timestampable` ซึ่งพึ่ง `Serializable` — 4 ชั้น กระจายอยู่คนละไฟล์ ปัญหาที่เกิดขึ้นจริงเมื่อต้อง
debug:

1. **เปิด `post.rb` เห็นแค่ `include Auditable`** — ต้องเปิดไฟล์ `auditable.rb` เพื่อรู้ว่ามันพึ่ง
   `Trackable` อีกที แล้วเปิด `trackable.rb` ต่อ... **ไล่เปิดไฟล์ 4 ชั้นกว่าจะเห็นภาพรวมทั้งหมด**
   ว่า `Post` มีพฤติกรรมอะไรบ้างจริงๆ
2. **`grep` หา method ที่ error บอกว่า "undefined"** อาจไม่เจอในไฟล์ไหนเลยที่คาดไว้ เพราะ method
   นั้นอาจถูกนิยามอยู่ใน concern ชั้นที่ 3 หรือ 4 ที่ไม่มีใครคาดคิดว่าเกี่ยวข้อง
3. **`instance_method(:some_method).owner`** กลายเป็นเครื่องมือที่ต้องใช้บ่อยเกินความจำเป็น
   เพียงเพื่อหาว่า "method นี้มาจากไฟล์ไหนกันแน่" ทั้งที่ในโค้ดปกติไม่ควรต้องมาถึงขั้นนี้เลย

**กฎปฏิบัติ:** จำกัด concern-chain ไว้ไม่เกิน 1-2 ชั้น (concern พึ่ง concern อื่นได้ แต่นั้นไม่ควร
ไปพึ่ง concern อื่นต่ออีก) ถ้ารู้สึกว่าจำเป็นต้องมี dependency มากกว่า 2 ชั้น มักเป็นสัญญาณว่า
ควรรวม concern เหล่านั้นเป็นก้อนเดียวที่มีความหมายชัดเจนกว่า หรือย้าย logic ร่วมนั้นไปเป็น
plain Ruby class ที่ concern แต่ละตัวเรียกใช้แทนการ include ต่อกันเป็นทอดๆ

---

## Step 358: ข้อถกเถียงที่ไม่มีคำตอบเดียว — Concern คือ "code smell" จริงหรือไม่

ถึงจุดนี้ Concern ดูเหมือนแก้ปัญหา fat model ได้อย่างสวยงาม — แต่ในชุมชน Rails มีข้อถกเถียงที่ดัง
และต่อเนื่องมานานหลายปีว่า **`ActiveSupport::Concern` เป็นเครื่องมือที่ทำให้ fat model แย่ลง
ไม่ใช่ดีขึ้น** เรื่องนี้ควรรู้ทั้งสองฝั่งอย่างเป็นธรรม เพราะเป็นการตัดสินใจออกแบบจริงที่ต้องเจอ
ในงาน production

### ฝั่งวิจารณ์: Concern ซ่อนความซับซ้อนไว้ ไม่ได้ลดมันจริง

มาดูตัวอย่างที่แสดงปัญหานี้ให้เห็นชัด — สมมติ `Order` มี concern 5 ตัว:

```ruby
class Order
  include Priceable    # total_with_tax
  include Shippable    # estimated_delivery_days
  include Notifiable   # notify_customer!
  include Cancellable  # cancellable?
  include Refundable   # refundable?

  attr_accessor :subtotal
end
```

```ruby
order = Order.new
order.public_methods(false)
# => [:subtotal, :subtotal=]
```

**`public_methods(false)` (เฉพาะที่นิยามตรงใน `Order` เอง) เห็นแค่ 2 method** ทั้งที่ `Order`
object จริงตอบสนอง (`respond_to?`) `total_with_tax`, `estimated_delivery_days`,
`notify_customer!`, `cancellable?`, `refundable?` ด้วย — **ความซับซ้อนทั้งหมดไม่ได้หายไปไหน
มันแค่ถูกยกไปไว้ใน 5 ไฟล์ที่แยกกัน** เปิด `order.rb` ไฟล์เดียวจะไม่มีทางรู้เลยว่า `Order` ทำอะไร
ได้บ้างทั้งหมด ต้องไล่เปิดทีละไฟล์เหมือนที่อธิบายไว้ใน Step 357

นักวิจารณ์ (ที่มีชื่อเสียงที่สุดคือบทความ "Vertical Slice" และการพูดในงาน Rails conference
หลายครั้งช่วงปี 2012-2016) ชี้ว่า:

> **Concern แบ่งโค้ดตาม "ลักษณะทางเทคนิค" (technical aspect) ไม่ใช่ตาม "ความรับผิดชอบทางธุรกิจ"
> (domain responsibility)** — `Priceable`/`Shippable`/`Notifiable` ไม่ใช่แนวคิดที่มีอยู่จริงใน
> ธุรกิจ มันเป็นแค่การจัดกลุ่ม method ตามรูปแบบไวยากรณ์ (ทุกตัวมี `class_attribute` เหมือนกัน,
> ทุกตัวมี callback เหมือนกัน) ไม่ใช่ตามความหมายที่นักวิเคราะห์ธุรกิจจะเข้าใจ ผลคือได้ Model ที่
> ดู "บาง" ในไฟล์เดียว แต่ยัง**หนักเท่าเดิม**เมื่อนับรวมทุกไฟล์ — เป็นแค่การซ่อน fat model ไว้
> หลังม่านของหลายไฟล์ ไม่ใช่การลด fat model จริง

### ฝั่งสนับสนุน: มีที่ทางจริงสำหรับพฤติกรรม "ข้ามโดเมน" ที่เป็นเรื่องเทคนิคล้วนๆ

ฝั่งตรงข้ามชี้ว่าข้อวิจารณ์ข้างต้น**ถูกต้องเมื่อใช้ Concern ผิดจุด** แต่ไม่ได้แปลว่า Concern
เป็นของไม่ดีในตัวมันเองเสมอไป — กรณีของ `Sluggable`/`Searchable`/`Archivable` ที่ทำไปแล้วใน
Part นี้**ไม่ใช่ business logic** เลยแม้แต่นิดเดียว มันคือ **พฤติกรรมทางเทคนิคที่ใช้ซ้ำได้จริง
ข้ามหลาย Model ที่ไม่มีความสัมพันธ์กันทางธุรกิจ**:

- "การมี slug" ไม่ใช่กฎธุรกิจของ `Post` หรือ `Category` — มันคือรายละเอียดทางเทคนิคของการทำ
  URL ให้อ่านง่าย ที่บังเอิญ`Post`และ`Category`ต้องการเหมือนกัน
- ทดสอบ `Sluggable` แยกเป็นเอกเทศได้ (สร้าง fake class ที่มี `title` แล้ว include เข้าไปตรงๆ)
  โดยไม่ต้องแตะ business logic ของ `Post`/`Category` เลย
- ถ้าลบ `Sluggable` ออกจาก `Post` พรุ่งนี้ ธุรกิจของบล็อกไม่เปลี่ยนแปลงเลยแม้แต่นิดเดียว
  (แค่ URL จะกลับไปใช้ `id` ธรรมดา) — นี่คือสัญญาณว่ามันเป็นเรื่อง**เทคนิค**ไม่ใช่เรื่อง**โดเมน**

เทียบกับตัวอย่าง `Order` ที่ถูกวิจารณ์ข้างบน — `Priceable#total_with_tax` (การคำนวณภาษี),
`Shippable#estimated_delivery_days` (กฎการจัดส่ง), `Cancellable#cancellable?` (กฎว่าเมื่อไหร่
ยกเลิกได้) **ทั้งหมดนี้คือ core business rule ของระบบสั่งซื้อ** — การลบ `Cancellable` ออกจาก
`Order` เปลี่ยนพฤติกรรมทางธุรกิจจริงๆ ทันที นี่คือความแตกต่างที่ชี้ขาดว่าเมื่อไหร่ Concern ช่วย
และเมื่อไหร่ Concern แค่ซ่อนปัญหา

### สรุปอย่างเป็นธรรม — คำถามที่ควรถามตัวเองก่อนสร้าง Concern ใหม่ทุกครั้ง

> **"ถ้าลบ Concern นี้ออก แล้วเอา method ทั้งหมดของมันไปยัดกลับเข้า Model ตรงๆ — logic ทางธุรกิจ
> ของระบบเปลี่ยนไปหรือไม่ หรือแค่ 'จัดระเบียบไฟล์ให้ดูเรียบร้อยขึ้น' เท่านั้น?"**
>
> - ถ้าคำตอบคือ **"แค่จัดระเบียบไฟล์ ไม่กระทบธุรกิจ"** (เช่น `Sluggable`, `Searchable`,
>   `Archivable`, `Timestampable`) — Concern เหมาะสม ใช้ได้เต็มที่
> - ถ้าคำตอบคือ **"ธุรกิจเปลี่ยนจริง เพราะนี่คือกฎสำคัญของระบบ"** (เช่น การคำนวณราคา, เงื่อนไข
>   การยกเลิกออเดอร์, state machine ของสถานะ) — Concern **ไม่ใช่เครื่องมือที่เหมาะสม** ต่อให้
>   เขียนด้วย `ActiveSupport::Concern` อย่างถูกไวยากรณ์ทุกประการก็ตาม ควรใช้ Service
>   Object/Value Object/Query Object แทน ซึ่งจะเรียนใน Step ถัดไป

ข้อสรุปนี้ตรงกับสิ่งที่ Part 026 เตือนไว้ตั้งแต่แรกเรื่อง callback: **เครื่องมือไม่ใช่ปัญหา
การใช้เครื่องมือผิดจุดต่างหากคือปัญหา** — `ActiveSupport::Concern` เป็นเครื่องมือที่ดีมากสำหรับ
พฤติกรรมทางเทคนิคที่ใช้ซ้ำได้ แต่กลายเป็นสถานที่ซ่อนปัญหาทันทีเมื่อถูกใช้แทนที่การออกแบบโดเมน
อย่างเหมาะสม

---

## Step 359: ทางเลือกที่ดีกว่าสำหรับ Business Logic จริง — Service Object, Value Object, Query Object

จาก Step 358 คำถามที่ตามมาคือ "ถ้าไม่ใช่ Concern แล้ว business logic แบบ `Cancellable`/
`Priceable` ควรอยู่ตรงไหน?" — คำตอบเต็มรูปแบบคือหัวข้อของ **Phase 14 (Part 082-083)** ทั้งเฟส
Step นี้จะแนะนำแค่พอให้เห็นภาพว่าทางเลือกเหล่านั้นหน้าตาเป็นอย่างไร เพื่อให้ตัดสินใจได้ถูกต้องว่า
"ตอนนี้ควรใช้ Concern หรือรอไปใช้เครื่องมือพวกนี้แทน"

### Service Object — ห่อหุ้ม "การกระทำ" ทางธุรกิจหนึ่งอย่างที่มีหลายขั้นตอน

```ruby
# preview เท่านั้น — เจาะลึกเต็มรูปแบบใน Part 082
# app/services/orders/mark_as_shipped.rb
module Orders
  class MarkAsShipped
    def initialize(order)
      @order = order
    end

    def call
      return false if @order.archived_at

      @order.archived_at = nil
      true
    end
  end
end

Orders::MarkAsShipped.new(order).call
```

ต่างจาก Concern ตรงที่ Service Object **ไม่ได้ผสมเข้าไปเป็นส่วนหนึ่งของ Model** — มันเป็น class
แยกต่างหากที่ **รับ Model เข้ามาเป็น argument** แล้วทำงานกับมันจากภายนอก เหมาะกับ "การกระทำ"
(verb) ที่มีหลายขั้นตอน อาจแตะหลาย Model พร้อมกัน (เช่น หักสต๊อกสินค้า + เปลี่ยนสถานะออเดอร์ +
ส่งอีเมล ในการกระทำเดียว "จัดส่งออเดอร์") ต่างจาก Concern ที่เหมาะกับ "การเป็น" (adjective/is-a
capability) อย่าง "เป็น object ที่ slug ได้", "เป็น object ที่ archive ได้"

### Value Object — ห่อหุ้มค่าที่มีกฎของตัวเอง แทนที่จะใช้ primitive ตรงๆ

```ruby
# preview เท่านั้น — เจาะลึกเต็มรูปแบบใน Part 082
Money = Struct.new(:cents) do
  def +(other) = Money.new(cents + other.cents)
  def to_s = format("%.2f บาท", cents / 100.0)
end

(Money.new(1000) + Money.new(250)).to_s
# => "12.50 บาท"
```

ปัญหาของ `Priceable#total_with_tax` ใน Step 358 คือมันคำนวณด้วย `Integer`/`Float` ตรงๆ
(`subtotal * 1.07`) ซึ่งไม่มีที่ไหนบังคับว่าหน่วยคือ "บาท" หรือ "สตางค์" — Value Object อย่าง
`Money` ห่อหุ้มกฎการคำนวณและหน่วยไว้ในตัวมันเอง ทำให้ผิดพลาดยากขึ้นและทดสอบแยกได้ง่ายกว่ามาก
เมื่อเทียบกับการฝัง logic คำนวณราคาไว้ใน Concern ของ `Order`

### Query Object — ห่อหุ้ม query ที่ซับซ้อนเกินกว่าจะเป็น scope บรรทัดเดียว

```ruby
# preview เท่านั้น — เจาะลึกเต็มรูปแบบใน Part 083
class PostsFilterQuery
  def initialize(relation)
    @relation = relation
  end

  def call(keyword:)
    return @relation if keyword.blank?

    @relation.where("title LIKE ?", "%#{keyword}%")
  end
end

PostsFilterQuery.new(Post.published).call(keyword: "Ruby")
```

Part 034 สอน named scope ไปแล้วว่าเหมาะกับเงื่อนไข query ง่ายๆ — แต่เมื่อ query filter มีเงื่อนไข
ประกอบกันหลายสิบบรรทัด (เช่น หน้า search ขั้นสูงที่มี filter 10 ตัวพร้อมกัน) การยัดทุกอย่างไว้เป็น
scope ในตัว Model (หรือแย่กว่านั้นคือยัดใน Concern) ทำให้ Model บวมขึ้นด้วยเหตุผลเดียวกับ Step 351
— Query Object แยก logic การกรองที่ซับซ้อนออกมาเป็น class ของตัวเอง ทดสอบและอ่านแยกจาก Model ได้

### กฎตัดสินใจสรุป: Concern vs Service/Value/Query Object

| ลักษณะพฤติกรรม | ใช้ Concern | ใช้ Service/Value/Query Object |
|---|---|---|
| เป็นเรื่องเทคนิคล้วนๆ (URL-friendly slug, full-text search, soft delete) | ใช่ | |
| ใช้ซ้ำได้จริงข้ามหลาย Model ที่ไม่เกี่ยวข้องกันทางธุรกิจ | ใช่ | |
| ลบออกแล้วธุรกิจไม่เปลี่ยน แค่ "จัดระเบียบไฟล์" | ใช่ | |
| เป็นกฎธุรกิจ/state machine/การคำนวณเฉพาะของ Model นี้ | | ใช่ |
| มีหลายขั้นตอน แตะหลาย Model พร้อมกัน ("การกระทำ" หนึ่งครั้ง) | | ใช่ (Service Object) |
| เป็น "ค่า" ที่มีกฎการคำนวณ/format ของตัวเอง | | ใช่ (Value Object) |
| query ซับซ้อนเกินกว่า scope บรรทัดเดียวจะอ่านง่าย | | ใช่ (Query Object) |

จำง่ายๆ ด้วยประโยคเดียว: **Concern เหมาะกับ "ความสามารถทางเทคนิคที่ Model เป็นเจ้าของ" ส่วน
Service/Value/Query Object เหมาะกับ "กระบวนการ/กฎ/การคำนวณทางธุรกิจที่เกิดขึ้นกับ Model"**

---

## Step 360: แบบฝึกหัด — สกัด `Sluggable` และ `Archivable` ใช้ร่วมกันระหว่างสองโมเดล

### โจทย์

ต่อยอดจากโดเมน `Post`/`Category` ที่ทำมาตลอด Part นี้ ให้:

1. เขียน Concern `Archivable` ที่ให้ความสามารถ **soft-delete แบบเบา** (ไม่ใช่ soft-delete gem
   เต็มรูปแบบ) ประกอบด้วย column `archived_at`, method `archived?`/`archive!`/`unarchive!`
2. เพิ่ม scope กรอง record ที่ยังไม่ถูก archive — **ต้องใช้ named scope ธรรมดา ห้ามใช้
   `default_scope`** ตามคำเตือนใน Part 034 Step 337 (ทบทวน: `default_scope` ซ่อนเงื่อนไขไว้ใน
   ทุก query โดยไม่มีใครเห็น และชนกับ `find`/`find_by` แม้ระบุ primary key ตรงเป๊ะ)
3. Include ทั้ง `Sluggable` (จาก Step 354) และ `Archivable` ใหม่นี้เข้าไปใน **ทั้ง `Post` และ
   `Category`** พิสูจน์ว่าใช้ร่วมกันได้จริงทั้งสอง Model
4. ทดสอบให้เห็นว่า record ที่ archive แล้วยังหาเจอผ่าน `Model.find`/`Model.all` ตามปกติ
   (ต่างจาก `default_scope` ที่จะซ่อนมันไปเงียบๆ) แต่ถูกกรองออกถ้าเรียกผ่าน scope `not_archived`
   ตรงๆ เท่านั้น

### เฉลย

**1) Concern `Archivable`**

```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  included do
    # ตั้งใจใช้ named scope ธรรมดา ไม่ใช้ default_scope
    # (ทบทวนเหตุผลจาก Part 034 Step 337: default_scope ซ่อนเงื่อนไขไว้ในทุก query
    # รวมถึง find/find_by และหน้า admin ที่อาจต้องการเห็นข้อมูลทั้งหมดด้วย)
    scope :not_archived, -> { where(archived_at: nil) }
    scope :archived, -> { where.not(archived_at: nil) }
  end

  def archived?
    archived_at.present?
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def unarchive!
    update!(archived_at: nil)
  end
end
```

**2) Include เข้าไปในทั้งสอง Model**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Sluggable
  include Searchable
  include Archivable
  include Post::Notifiable

  belongs_to :author
  belongs_to :category
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true

  before_validation :strip_whitespace

  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :popular, -> { where("views > ?", 100) }

  def self.searchable_columns
    %w[title body]
  end

  def reading_time_minutes
    words = body.to_s.split.size
    (words / 200.0).ceil.clamp(1, Float::INFINITY).to_i
  end

  def excerpt(length: 140)
    body.to_s.truncate(length)
  end

  private

  def strip_whitespace
    self.title = title.to_s.strip
    self.body = body.to_s.strip
  end
end
```

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  include Sluggable
  include Searchable
  include Archivable

  has_many :posts

  validates :name, presence: true

  def self.searchable_columns
    %w[name]
  end
end
```

**สังเกตความเปลี่ยนแปลงของ `post.rb`:** เทียบกับเวอร์ชัน "fat model" ใน Step 351 (57 บรรทัด
ปนกันหมดในไฟล์เดียว) ตอนนี้ไฟล์เหลือเฉพาะสิ่งที่เป็น**ของ `Post` จริงๆ** (`title`/`body`
validation, `published`/`popular` scope, `reading_time_minutes`, `excerpt`) ส่วน slug, search,
archive, notification ถูกแยกออกไปเป็น concern คนละไฟล์ที่มีชื่อสื่อความหมายชัดเจน — และที่สำคัญ
`Category` ก็ได้ `Sluggable`/`Searchable`/`Archivable` มาฟรีๆ โดยไม่ต้อง copy โค้ดสักบรรทัดเดียว

**3) ทดสอบจริงผ่าน `bin/rails runner`**

```ruby
author = Author.create!(name: "Somchai", nationality: "Thai")
category = Category.create!(name: "Ruby on Rails")
post = Post.create!(title: "Concern Exercise", body: "เนื้อหาทดสอบ " * 20,
                     author: author, category: category)

puts "post.slug: #{post.slug}"
puts "category.slug: #{category.slug}"

# archive ทั้งสอง record
post.archive!
category.archive!

puts "post.archived?: #{post.archived?}"
puts "category.archived?: #{category.archived?}"

# find ตรงๆ ยังหาเจอปกติ (ต่างจาก default_scope ที่จะซ่อนไปเงียบๆ)
puts "Post.find(#{post.id}) เจอไหม: #{Post.find(post.id).present?}"
puts "Post.all.count: #{Post.all.count}"

# ต้องเรียก scope ตรงๆ ถึงจะถูกกรอง
puts "Post.not_archived.count: #{Post.not_archived.count}"
puts "Category.not_archived.count: #{Category.not_archived.count}"

# unarchive กลับ
post.unarchive!
puts "Post.not_archived.count หลัง unarchive: #{Post.not_archived.count}"
```

ผลลัพธ์จริง:

```
post.slug: concern-exercise-3f1a02
category.slug: ruby-on-rails-91a30f
post.archived?: true
category.archived?: true
Post.find(1) เจอไหม: true
Post.all.count: 1
Post.not_archived.count: 0
Category.not_archived.count: 0
Post.not_archived.count หลัง unarchive: 1
```

**พิสูจน์ครบตามโจทย์:** `Post.find`/`Post.all` ยัง**เห็น record ที่ archive แล้วตามปกติ** (ต่าง
จาก `default_scope` ที่ Part 034 เตือนไว้ว่าจะทำให้ `find` โยน `RecordNotFound` แม้ record จะยัง
อยู่จริง) และต้อง**เรียก `.not_archived` ตรงๆ อย่างชัดเจน** ถึงจะถูกกรองออก — ทุกจุดที่เรียกใช้
มองเห็นเจตนาชัดเจนในโค้ด ไม่มีอะไรถูกซ่อนไว้เบื้องหลังเหมือนที่ Step 337 เตือนไว้

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. เขียน concern ทั่วไปชื่อ `Notifiable` (ไม่ namespace ด้วยชื่อ Model แบบ `Post::Notifiable`
   ในบทเรียน) ที่รับพารามิเตอร์ตอน `include` ผ่าน class method (เช่น
   `include Notifiable; notifies_on :create, message: ->(record) { "..." }`) แล้วใช้กับทั้ง
   `Post` และ `Category` ด้วยข้อความแจ้งเตือนที่ต่างกัน (ใบ้: เก็บค่าที่ config ไว้ด้วย
   `class_attribute` ใน `included do` แล้วให้ `class_methods do` มี method สำหรับตั้งค่า)
2. เพิ่ม concern `Auditable` ที่บันทึกทุกครั้งที่ record มีการเปลี่ยนแปลง (ใช้
   `after_update :log_changes` ร่วมกับ `saved_changes` ที่เรียนจาก Part 035 Step 344) ลง
   `Rails.logger` ก่อน แล้วลองขยายให้บันทึกลง Model ใหม่ชื่อ `AuditLog` แทน — พิจารณาด้วยว่า
   `Auditable` แบบนี้เข้าข่าย "เทคนิคล้วนๆ" ตามเกณฑ์ Step 358 จริงหรือไม่ ถ้ามันเริ่มมี logic
   ตัดสินใจว่า field ไหนสำคัญพอจะ audit (ซึ่งอาจต่างกันไปในแต่ละ Model ตามกฎธุรกิจ) จุดไหนที่
   มันข้ามเส้นจาก "เทคนิค" ไปเป็น "ธุรกิจ" แล้ว
3. กลับไปดูตัวอย่าง `Order` ที่มี 5 concern จาก Step 358 (`Priceable`, `Shippable`,
   `Notifiable`, `Cancellable`, `Refundable`) แล้วจัดกลุ่มใหม่ตามเกณฑ์ตารางใน Step 359:
   concern ตัวไหนควรยังเป็น Concern ต่อไป และตัวไหนควรย้ายไปเป็น Service Object/Value Object
   แทน พร้อมให้เหตุผลประกอบทีละตัว (ยังไม่ต้องเขียนโค้ดจริง เพราะ Service Object เต็มรูปแบบจะ
   เรียนใน Part 082 — แค่ฝึกตัดสินใจตามเกณฑ์ก่อน)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Fat model เกิดขึ้นเองตามธรรมชาติ** เมื่อฟีเจอร์ถูกเพิ่มเข้า Model ทีละนิดโดยที่แต่ละจุดดู
  สมเหตุสมผลในตัวเอง แต่ปัญหาที่แท้จริงคือ **maintainability** ไม่ใช่ความถูกต้องของโค้ด — logic
  อย่าง slug generation ที่ใช้ซ้ำใน Model อื่นไม่ได้เลยถ้าไม่แยกออกมา
- **Module ธรรมดา (Part 010) มีจุดอ่อนเมื่อใช้กับ ActiveRecord จริง**: ถ้า module หนึ่งต้องพึ่ง
  อีก module หนึ่ง ผู้ include ต้องจำลำดับเอง (พังถ้าสลับ) และถ้าพยายามแก้ด้วยการ `include` ซ้อน
  กันเองใน module จะพังทันทีเพราะ `self.included(base)` ได้ `base` ผิดตัว (module ตัวกลาง
  ไม่ใช่ host class จริง) — พิสูจน์ด้วยโค้ดจริงทั้งสองกรณี
- **`ActiveSupport::Concern`** แก้ปัญหานี้ด้วย `extend ActiveSupport::Concern`,
  `included do ... end` (รันด้วย `self` เป็น host class เสมอ ไม่ว่าจะผ่านกี่ชั้น), และ
  `class_methods do ... end` (เพิ่ม class method อย่างเป็นทางการ) — ยังคงเป็น mixin ธรรมดาตาม
  MRO ของ Part 010 ทุกประการ ไม่ใช่ syntax พิเศษของภาษา
- สกัด **`Sluggable`** (ใช้ convention เดา `title`/`name`) และ **`Searchable`** (บังคับ
  `searchable_columns` ด้วย `NotImplementedError`) ใช้ร่วมกันจริงระหว่าง `Post` และ `Category`
  ที่ไม่มีความสัมพันธ์แบบ inheritance กันเลย
- **ธรรมเนียม `app/models/concerns/`**: concern ที่ใช้ร่วมหลาย Model วางตรงๆ ไม่ namespace
  (`Sluggable`, `Searchable`) ส่วน concern ที่ใช้กับ Model เดียว (แค่แบ่งไฟล์ให้อ่านง่าย) ควร
  namespace ด้วยชื่อ Model (`Post::Notifiable`) — Zeitwerk resolve namespace จากตำแหน่งไฟล์ให้
  อัตโนมัติ
- **Dependency ระหว่าง concern** ประกาศด้วย `include OtherConcern` ที่ module body (ไม่ใช่ใน
  `included do`) เพื่อให้ลำดับ `ancestors` ตรงตามสัญชาตญาณ และควรจำกัด concern-chain ไว้ไม่เกิน
  1-2 ชั้น เพราะยิ่งลึกยิ่งต้องไล่เปิดหลายไฟล์กว่าจะเห็นภาพรวม
- **ข้อถกเถียงที่ต้องรู้ทั้งสองฝั่ง**: Concern ถูกวิจารณ์ว่าจัดกลุ่มโค้ดตามลักษณะทางเทคนิคแทนที่
  จะตามความรับผิดชอบทางธุรกิจ ทำให้ Model ดู "บาง" ในไฟล์เดียวแต่ความซับซ้อนจริงไม่ได้ลดลง
  (พิสูจน์ด้วย `public_methods(false)` ที่เห็นแค่ 2 method ทั้งที่ object ตอบสนองอีกหลายสิบ
  method) — แต่ Concern ก็มีที่ทางจริงสำหรับพฤติกรรมทางเทคนิคที่ใช้ซ้ำได้ข้ามโดเมน (`Sluggable`,
  `Searchable`, `Archivable`) กฎตัดสินคือ **"ลบออกแล้วธุรกิจเปลี่ยนไหม"**
- **ทางเลือกสำหรับ business logic จริง** (preview Part 082-083): **Service Object** สำหรับ
  การกระทำที่มีหลายขั้นตอน/แตะหลาย Model, **Value Object** สำหรับค่าที่มีกฎการคำนวณของตัวเอง,
  **Query Object** สำหรับ query ที่ซับซ้อนเกินกว่า scope บรรทัดเดียว
- แบบฝึกหัด: สกัด **`Archivable`** ด้วย **named scope** (`not_archived`/`archived`) แทน
  `default_scope` ตามคำเตือนของ Part 034 Step 337 อย่างเคร่งครัด แล้วใช้ร่วมกับ `Sluggable`
  ในทั้ง `Post` และ `Category` พิสูจน์ว่า `find`/`all` ยังเห็นข้อมูลครบ ต้องเรียก scope ตรงๆ
  เท่านั้นถึงจะถูกกรอง

**ต่อไป (Part 037):** Part 031 แนะนำ `nested_attributes`/`fields_with` แบบเบื้องต้นไปแล้วสำหรับ
ความสัมพันธ์เดียว (เช่น ฟอร์มเดียวที่แก้ทั้ง `Post` และ author คนเดียว) Part 037 จะเจาะลึกเรื่อง
**Multiple Models Form** เต็มรูปแบบ: `accepts_nested_attributes_for` กับความสัมพันธ์แบบ
`has_many` จริง (เช่น ฟอร์มเดียวที่สร้าง `Order` พร้อม `LineItem` หลายรายการพร้อมกัน), การใช้
`fields_for` วน iterate เพื่อสร้างฟอร์มย่อยหลายชุด, การเพิ่ม/ลบแถวแบบ dynamic ด้วย JavaScript,
และ `_destroy` field สำหรับลบ nested record ผ่านฟอร์มเดียวกัน — ทักษะที่จำเป็นสำหรับหน้าฟอร์ม
ระดับ production ที่ซับซ้อนกว่าการแก้ record เดียวมาก
