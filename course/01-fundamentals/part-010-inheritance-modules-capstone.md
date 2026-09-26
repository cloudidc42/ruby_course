# Part 010: Inheritance, Module, Mixin — และโปรเจกต์ปิด Phase 1: Library Management CLI

> **Step ครอบคลุมใน Part นี้:** Step 91–100
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 001–009 มาก่อน โดยเฉพาะ Part 009 เรื่อง Class/Object พื้นฐาน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6)

Part นี้คือ Part สุดท้ายของ **Phase 1: Ruby Fundamentals** เราจะต่อยอดจาก Class/Object ที่
เรียนไปใน Part 009 ด้วยแนวคิดสำคัญ 2 เรื่องคือ **Inheritance (การสืบทอด)** และ
**Module/Mixin** ซึ่งเป็นรากฐานของการออกแบบโค้ด OOP แบบ Ruby ที่แท้จริง แล้วปิดท้ายด้วย
โปรเจกต์รวบยอดขนาดเล็ก **Library Management CLI** ที่ดึงทุกอย่างที่เรียนมาตลอด 10 Part
มาประกอบร่างเป็นโปรแกรมที่ใช้งานได้จริง

## สารบัญของ Part นี้

- Step 91: Class Inheritance พื้นฐาน — `class Child < Parent`
- Step 92: `super` — เรียก method ของ parent class (bare, with args, explicit args)
- Step 93: `protected` vs `private` ในบริบทของ inheritance
- Step 94: ตรวจสอบชนิดของ object — `is_a?`, `kind_of?`, `instance_of?`, `ancestors`
- Step 95: Module เป็น Namespace
- Step 96: Module เป็น Mixin ผ่าน `include` — เพิ่ม instance method
- Step 97: `extend` — เพิ่ม class method / singleton method
- Step 98: Method Resolution Order (MRO) — ห่วงโซ่การค้นหา method อย่างละเอียด
- Step 99: Comparable และ Enumerable — Mixin ตัวจริงจาก Ruby Standard Library
- Step 100: แบบฝึกหัดปิด Phase 1 — โปรเจกต์ Library Management CLI

---

## Step 91: Class Inheritance พื้นฐาน — `class Child < Parent`

**Inheritance (การสืบทอด)** คือกลไกที่ให้ class หนึ่ง (subclass/child class) รับเอา method
และ attribute ทั้งหมดของอีก class หนึ่ง (superclass/parent class) มาใช้งานต่อได้ทันที
โดยไม่ต้องเขียนซ้ำ

หลักการเลือกใช้ inheritance ที่ถูกต้องคือความสัมพันธ์แบบ **"is-a"** (เป็นชนิดหนึ่งของ) เช่น
"Dog is an Animal", "Book is an Item" — ถ้าความสัมพันธ์เป็นแบบ **"has-a"** (มี) เช่น
"Car has an Engine" ควรใช้ **composition** (เก็บ object อื่นไว้เป็น attribute) แทน ไม่ใช่
inheritance

```ruby
# frozen_string_literal: true

class Animal
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def speak
    "#{name} ส่งเสียงบางอย่าง..."
  end

  def sleep
    "#{name} กำลังนอนหลับ zzz"
  end
end

class Dog < Animal
  def speak
    "#{name} เห่า: โฮ่ง โฮ่ง!"
  end
end

class Cat < Animal
  # ไม่ override speak — จะใช้ของ Animal ที่รับมาแบบเต็มๆ
end

dog = Dog.new("โปโป้")
cat = Cat.new("มะลิ")

puts dog.speak
# => โปโป้ เห่า: โฮ่ง โฮ่ง!

puts cat.speak
# => มะลิ ส่งเสียงบางอย่าง...

puts dog.sleep
# => โปโป้ กำลังนอนหลับ zzz  (สืบทอดมาจาก Animal โดยไม่ต้องเขียนใหม่)
```

**อธิบาย:**

- `class Dog < Animal` หมายความว่า `Dog` **สืบทอด (inherit)** จาก `Animal` — เครื่องหมาย `<`
  ใช้บอกว่าอะไรคือ superclass
- `Dog` ได้ `attr_reader :name`, `initialize`, และ `sleep` มาจาก `Animal` โดยอัตโนมัติ
  ไม่ต้องเขียนซ้ำ
- `Dog#speak` **override (เขียนทับ)** method `speak` ของ `Animal` — เมื่อเรียก `dog.speak`
  Ruby จะใช้ version ของ `Dog` ก่อนเสมอ (ดูกลไกละเอียดใน Step 98)
- `Cat` ไม่ override `speak` เลย จึงใช้ version ของ `Animal` ทันที
- ทุก class ใน Ruby ที่ไม่ได้ระบุ superclass จะสืบทอดจาก `Object` โดยอัตโนมัติอยู่แล้ว
  (`class Animal` จริงๆ คือ `class Animal < Object`)

> **แนวคิดสำคัญ:** Ruby รองรับ **single inheritance** เท่านั้น (class หนึ่งมี superclass ได้
> แค่ 1 ตัว ไม่เหมือน C++ ที่มี multiple inheritance) แต่ Ruby แก้ปัญหาการใช้โค้ดร่วมกันจาก
> หลายแหล่งด้วย **Module/Mixin** แทน ซึ่งเราจะเรียนใน Step 95–98 — นี่คือเหตุผลที่ Ruby
> มี Module เป็นแนวคิดสำคัญคู่กับ Class

### ทำไม Inheritance ถึงสำคัญต่อการเรียน Rails

ใน Rails ทุก Model จะเขียนแบบ `class Post < ApplicationRecord` และทุก Controller เขียนแบบ
`class PostsController < ApplicationController` — นี่คือ inheritance ที่เราเพิ่งเรียนนี่เอง
`ApplicationRecord`/`ApplicationController` เป็น class กลางที่ทำหน้าที่ใส่พฤติกรรมร่วม
(shared behavior) ให้ทุก Model/Controller ในโปรเจกต์โดยอัตโนมัติ

---

## Step 92: `super` — เรียก method ของ parent class

เมื่อ subclass override method ของ parent แล้ว บ่อยครั้งเราไม่ได้ต้องการ "แทนที่ทั้งหมด"
แต่ต้องการ **"ต่อยอด"** จากพฤติกรรมเดิม — ทำได้ด้วยคำสั่ง `super`

`super` มี 3 รูปแบบที่ให้ผลต่างกัน **ต้องแยกให้ออกเพราะเป็นจุดที่มือใหม่สับสนบ่อยที่สุด**:

| รูปแบบ | พฤติกรรม |
|--------|----------|
| `super` (ไม่มีวงเล็บ) | ส่ง argument **ชุดเดียวกับที่ method ปัจจุบันได้รับ** ไปให้ parent โดยอัตโนมัติ |
| `super(a, b)` | ส่ง argument **ที่ระบุเอง** ไปให้ parent (จะส่งค่าอะไรก็ได้ ไม่จำเป็นต้องตรงกับที่รับมา) |
| `super()` (วงเล็บว่าง) | เรียก parent method โดย **ไม่ส่ง argument ใดๆ เลย** |

```ruby
# frozen_string_literal: true

class Base
  def greet(name, greeting: "สวัสดี")
    "#{greeting}, #{name}!"
  end
end

# 1) bare super — ส่ง (name, greeting: greeting) ที่ตัวเองได้รับ ไปให้ parent ทั้งหมด
class Loud < Base
  def greet(name, greeting: "สวัสดี")
    super.upcase
  end
end

# 2) super พร้อม argument ที่ระบุเอง — เปลี่ยน greeting ก่อนส่งต่อ
class Formal < Base
  def greet(name, greeting: "สวัสดี")
    super(name, greeting: "เรียนคุณ")
  end
end

# 3) super() วงเล็บว่าง — ไม่ส่ง argument เลย ทั้งที่ parent method ต้องการ name
class NoArgs < Base
  def greet(name, greeting: "สวัสดี")
    super()
  rescue ArgumentError => e
    "ผิดพลาด: #{e.message}"
  end
end

puts Loud.new.greet("มานี")
# => สวัสดี, มานี! -> .upcase -> "สวัสดี, มานี!".upcase => "สวัสดี, มานี!" เป็นภาษาไทยจึง
#    ไม่มีตัวพิมพ์ใหญ่ (ตัวอย่างนี้ชัดกว่าถ้าลองกับข้อความอังกฤษ)

puts Formal.new.greet("มานี")
# => เรียนคุณ, มานี!

puts NoArgs.new.greet("มานี")
# => ผิดพลาด: wrong number of arguments (given 0, expected 1)
```

**อธิบาย:**

- `super` (bare) เหมาะเมื่อต้องการ **"ทำงานเดิมของ parent ให้เสร็จก่อน แล้วค่อยเพิ่มพฤติกรรม"**
  โดยไม่สนใจว่า argument คืออะไร — พบบ่อยที่สุดในการเขียน `initialize`
- `super(args)` เหมาะเมื่อต้องการ **แก้ไข/เพิ่มเติม argument ก่อนส่งต่อ** ให้ parent
- `super()` เหมาะเมื่อ parent method ไม่ต้องการ argument เลย หรือเราต้องการยืนยันชัดเจนว่า
  "ไม่ส่งอะไรไปเลย" — ตัวอย่างข้างบนแสดงให้เห็นว่าถ้า parent method ต้องการ argument
  แต่เราตั้งใจส่ง `super()` แบบว่างเปล่า จะเกิด `ArgumentError` ทันที

### ตัวอย่างที่ใช้บ่อยที่สุด: `super` ใน `initialize`

```ruby
class Item
  attr_reader :title, :year

  def initialize(title:, year:)
    @title = title
    @year = year
  end
end

class Book < Item
  attr_reader :isbn

  def initialize(title:, year:, isbn:)
    super(title: title, year: year)  # ส่งเฉพาะ keyword ที่ parent ต้องการ
    @isbn = isbn
  end
end

book = Book.new(title: "The Ruby Way", year: 2015, isbn: "978-0-13-345659-7")
puts book.title  # => The Ruby Way (มาจาก parent initialize ผ่าน super)
puts book.isbn   # => 978-0-13-345659-7 (ตั้งค่าเองใน Book#initialize)
```

รูปแบบนี้ — เรียก `super` ก่อนเพื่อให้ parent ตั้งค่า attribute ของตัวเอง แล้วค่อยตั้งค่า
attribute ที่ subclass เพิ่มเข้ามาเอง — คือ pattern มาตรฐานที่จะใช้ซ้ำๆ ตลอดทั้งโปรเจกต์
Library Management CLI ใน Step 100

---

## Step 93: `protected` vs `private` ในบริบทของ inheritance

ใน Part 009 เราแตะเรื่อง `private` ไปเล็กน้อย ตอนนี้ถึงเวลาเจาะลึกความแตกต่างระหว่าง
`private` กับ `protected` ซึ่งชัดเจนที่สุดเมื่อมี inheritance หรือการเปรียบเทียบ object
ประเภทเดียวกันเข้ามาเกี่ยวข้อง

| Access level | เรียกจากภายนอก (มี receiver) | เรียกจาก instance method อื่นของ class เดียวกัน/subclass (ไม่มี receiver) | เรียกโดยระบุ receiver เป็น object อื่นของ class เดียวกัน (จาก method ภายใน) |
|---|---|---|---|
| `public` | ได้ | ได้ | ได้ |
| `protected` | **ไม่ได้** | ได้ | **ได้** |
| `private` | **ไม่ได้** | ได้ | **ไม่ได้** (ยกเว้น setter บาง case) |

พูดง่ายๆ: **`private` ห้ามระบุ receiver เด็ดขาด** (เรียกได้แต่ตัวเองแบบ implicit `self`
เท่านั้น) ส่วน **`protected` ห้ามคนนอกเรียก แต่อนุญาตให้ object ประเภทเดียวกันเรียกหากันเอง
ผ่าน receiver ได้** — เคสคลาสสิกที่ต้องใช้ `protected` คือการเขียน method เปรียบเทียบ
ค่าภายในของ object สองตัว

```ruby
# frozen_string_literal: true

class BankAccount
  def initialize(owner, balance)
    @owner = owner
    @balance = balance
  end

  def >(other)
    balance > other.balance   # ต้องเข้าถึง balance ของ "other" ซึ่งเป็น object อีกตัว
  end

  def display
    "#{@owner}: #{format_currency(@balance)}"
  end

  protected

  # protected: เข้าถึง @balance ของ object อื่นที่เป็น BankAccount เหมือนกันได้ ผ่าน other.balance
  def balance
    @balance
  end

  private

  # private: ใช้ได้แค่ภายใน method ของตัวเองเท่านั้น ห้ามระบุ receiver แม้แต่ self.
  def format_currency(amount)
    "#{amount} บาท"
  end
end

a = BankAccount.new("A", 1000)
b = BankAccount.new("B", 500)

puts a.display        # => A: 1000 บาท
puts(a > b)            # => true  (เปรียบเทียบผ่าน protected method ได้)

a.balance
# => NoMethodError: protected method `balance' called for an instance of BankAccount

a.format_currency(10)
# => NoMethodError: private method `format_currency' called for an instance of BankAccount
```

**อธิบาย:**

- `a > b` ทำงานได้เพราะภายใน method `>` ของ `BankAccount` เราเรียก `other.balance` — `other`
  เป็น `BankAccount` เหมือนกัน และ `balance` เป็น `protected` จึงอนุญาต
- ถ้าเปลี่ยน `balance` เป็น `private` โค้ด `other.balance` จะ error ทันที เพราะ `private`
  ไม่ยอมให้ระบุ receiver แม้ receiver จะเป็น object ประเภทเดียวกันก็ตาม
- `a.balance` และ `a.format_currency(10)` ที่เรียกจากภายนอก (top-level) ทั้งคู่ error
  เพราะทั้ง `protected` และ `private` ปิดกั้นการเรียกจากภายนอก class เหมือนกัน — ความต่าง
  อยู่ที่ "การเรียกข้าม object ของ class เดียวกัน" เท่านั้น

**ในบริบทของ inheritance:** ทั้ง `protected` และ `private` method ของ parent class จะถูก
สืบทอดไปยัง subclass ตามปกติ และใช้กฎเดียวกัน เช่น ถ้า `Dog < Animal` และ `Animal` มี
`protected` method ชื่อ `energy_level` ก็สามารถเขียน method ใน `Dog` ที่เทียบ
`energy_level` ของ `Dog` ตัวหนึ่งกับอีกตัวหนึ่งได้ (ตราบใดที่ทั้งคู่เป็น subclass ของ
`Animal` เหมือนกัน)

> **กฎการเลือกใช้ในทางปฏิบัติ:** เริ่มต้นด้วย `public` เสมอสำหรับ method ที่เป็น API
> ของ object แล้วเปลี่ยนเป็น `private` สำหรับ implementation detail ภายใน ใช้ `protected`
> เฉพาะกรณีพิเศษที่ต้องเปรียบเทียบ/เข้าถึงข้อมูลภายในระหว่าง object ประเภทเดียวกัน (เช่น
> `<=>`, `==` แบบ custom) ซึ่งพบไม่บ่อยเท่า `private`

---

## Step 94: ตรวจสอบชนิดของ object — `is_a?`, `kind_of?`, `instance_of?`, `ancestors`

เมื่อมี inheritance และ module เข้ามาเกี่ยวข้อง การถามว่า "object นี้เป็นชนิดอะไรกันแน่"
มีความซับซ้อนขึ้น Ruby มี method หลายตัวให้ตรวจสอบ โดยแต่ละตัวตอบคำถามคนละแบบ

```ruby
# frozen_string_literal: true

module Flyable
end

class Bird
  include Flyable
end

class Sparrow < Bird
end

sparrow = Sparrow.new

sparrow.is_a?(Sparrow)     # => true  (เป็น Sparrow โดยตรง)
sparrow.is_a?(Bird)        # => true  (Sparrow สืบทอดจาก Bird)
sparrow.is_a?(Flyable)     # => true  (Bird include Flyable มาด้วย)
sparrow.kind_of?(Bird)     # => true  (kind_of? คือ alias ของ is_a? ทุกประการ)
sparrow.instance_of?(Bird) # => false (instance_of? เช็คเฉพาะ class ตรงตัว ไม่นับ superclass)
sparrow.instance_of?(Sparrow) # => true

Sparrow.ancestors
# => [Sparrow, Bird, Flyable, Object, Kernel, BasicObject]
```

**อธิบาย:**

- **`is_a?`** และ **`kind_of?`** เป็น method เดียวกัน (alias กัน) ตรวจสอบว่า object นั้นอยู่
  ใน **ancestor chain** ของ class ที่ระบุหรือไม่ — นับรวมทั้ง superclass ทุกชั้น **และ**
  module ที่ถูก `include` เข้ามาด้วย
- **`instance_of?`** เข้มงวดกว่ามาก — เช็คเฉพาะว่า object เป็น instance ของ **class นั้นตรงๆ**
  เท่านั้น ไม่นับ superclass หรือ module ใดๆ เลย
- **`ancestors`** (เรียกจาก class ไม่ใช่ instance) คืน Array ของลำดับการค้นหา method
  ทั้งหมดของ class นั้น เรียงจากใกล้สุดไปไกลสุด — นี่คือ **Method Resolution Order (MRO)**
  ที่เราจะเจาะลึกใน Step 98

```ruby
# ตัวอย่างการใช้งานจริง: guard clause ตรวจสอบ argument
def render_item(item)
  raise TypeError, "ต้องการ Item เท่านั้น" unless item.is_a?(Item)
  puts item.to_s
end
```

> **แนวคิดสำคัญ — Duck Typing:** แม้ Ruby จะมี method ตรวจสอบชนิดให้ครบ แต่ปรัชญาของ Ruby
> คือ **"ถ้ามันเดินเหมือนเป็ด ร้องเหมือนเป็ด ก็ถือว่าเป็นเป็ด"** — เรามักเช็คว่า object
> "ตอบสนอง method ที่ต้องการหรือไม่" (`respond_to?`) มากกว่าเช็คว่า "เป็น class อะไร"
> เพื่อให้โค้ดยืดหยุ่นและรองรับ object ต่างชนิดที่ทำหน้าที่เดียวกันได้ (เราจะเรียนเรื่อง
> Duck Typing แบบเต็มใน Part 016) การใช้ `is_a?`/`instance_of?` จึงควรสงวนไว้ใช้เมื่อจำเป็น
> จริงๆ เช่น validate ชนิด argument หรือแยกเงื่อนไขตาม type

---

## Step 95: Module เป็น Namespace

**Module** ใน Ruby มี 2 บทบาทหลัก คือ (1) เป็น **namespace** สำหรับจัดกลุ่มโค้ดและป้องกัน
ชื่อชนกัน และ (2) เป็น **Mixin** สำหรับแบ่งปันพฤติกรรมข้าม class (Step 96–97) เริ่มจาก
บทบาทแรกก่อน

```ruby
# frozen_string_literal: true

module Shop
  class Order
    def summary
      "Shop::Order"
    end
  end
end

module Warehouse
  class Order
    def summary
      "Warehouse::Order"
    end
  end
end

puts Shop::Order.new.summary
# => Shop::Order

puts Warehouse::Order.new.summary
# => Warehouse::Order
```

**อธิบาย:**

- `module Shop; class Order; ... end; end` ประกาศ class `Order` **ภายใน** module `Shop`
  เรียกใช้งานผ่าน `Shop::Order` (เครื่องหมาย `::` คือ **scope resolution operator**)
- ทั้งสองไฟล์นี้มี class ชื่อ `Order` เหมือนกัน แต่ไม่ชนกันเลยเพราะอยู่คนละ namespace
  (`Shop::Order` ≠ `Warehouse::Order`) — ถ้าไม่มี module ครอบ การประกาศ `class Order`
  ซ้ำสองครั้งจะเป็นการ "เปิด class เดิมมาแก้ไขต่อ" (reopening) ไม่ใช่สร้างใหม่ ซึ่งจะทำให้
  method ปนกันโดยไม่ตั้งใจ
- Module ที่ใช้เป็น namespace **ไม่จำเป็นต้องมี method ของตัวเอง** หน้าที่หลักคือห่อหุ้ม
  (group) กลุ่ม class ที่เกี่ยวข้องกันไว้ด้วยกัน

```ruby
# module ยังใช้เก็บ constant และ module method (utility function) ได้ด้วย
module MathUtils
  PI_APPROX = 3.14159

  def self.circle_area(radius)
    PI_APPROX * radius**2
  end
end

puts MathUtils::PI_APPROX          # => 3.14159
puts MathUtils.circle_area(2)      # => 12.56636
```

`def self.circle_area` ภายใน module คือการนิยาม **module method** เรียกใช้ตรงๆ ผ่าน
`MathUtils.circle_area(...)` โดยไม่ต้องสร้าง instance ใดๆ เลย (เหมือนกับ class method ที่
เรียนใน Part 009) รูปแบบนี้เหมาะกับฟังก์ชัน utility ที่ไม่ต้องมี state

> **preview Rails:** โครงสร้างนี้คือสิ่งที่ Rails ใช้จริงเวลาแบ่ง namespace ของ Controller
> เช่น `Admin::UsersController` (สำหรับหน้าแอดมิน) แยกจาก `UsersController` (สำหรับผู้ใช้
> ทั่วไป) — สิ่งที่เรียนตรงนี้ใช้ต่อได้ทันทีตอนเข้า Rails ใน Phase 3

---

## Step 96: Module เป็น Mixin ผ่าน `include` — เพิ่ม instance method

นี่คือบทบาทที่สำคัญที่สุดของ Module ใน Ruby: การเป็น **Mixin** — ชุดของ method ที่เขียนไว้
ครั้งเดียว แล้วนำไปผสม (mix in) เข้ากับหลาย class ที่ไม่มีความสัมพันธ์แบบ inheritance กันเลย
ก็ได้ แก้ปัญหาที่ Ruby ไม่รองรับ multiple inheritance (ตามที่กล่าวไว้ใน Step 91)

```ruby
# frozen_string_literal: true

module Walkable
  def move
    "#{name} เดิน"
  end
end

module Swimmable
  def move
    "#{name} ว่ายน้ำ"
  end
end

class Duck
  include Walkable
  include Swimmable

  attr_reader :name

  def initialize(name)
    @name = name
  end
end

puts Duck.new("โดนัลด์").move
# => โดนัลด์ ว่ายน้ำ

Duck.ancestors.first(4)
# => [Duck, Swimmable, Walkable, Object]
```

**อธิบาย:**

- `include Walkable` นำ instance method ทั้งหมดของ `Walkable` มาผสมเข้ากับ `Duck` — เรียก
  ใช้ `duck.move` ได้ราวกับว่า `move` ถูกเขียนอยู่ใน `Duck` เอง
- เมื่อ `include` มากกว่า 1 module และมี method ชื่อซ้ำกัน (`move` ทั้งใน `Walkable` และ
  `Swimmable`) **module ที่ include ทีหลังจะถูกวางไว้ใกล้ตัว class มากกว่า** จึงถูกค้นเจอก่อน
  — จาก `ancestors` จะเห็นว่า `Swimmable` อยู่เหนือ `Walkable` เพราะ `include Swimmable`
  มาทีหลัง ผลคือ `duck.move` เรียกไปเจอ version ของ `Swimmable`
- Module ที่จะใช้เป็น Mixin **ไม่ต้องมี `initialize` ของตัวเอง** — มันไม่ใช่ superclass และ
  ไม่ได้ถูกสร้าง instance ตรงๆ (ไม่สามารถเขียน `Walkable.new` ได้เลย, module สร้าง instance
  ไม่ได้) มันแค่ "แปะ" method เข้าไปในสายการค้นหาของ class ที่ include เท่านั้น
- Method ใน module มักจะอ้างอิงถึง method อื่นที่คาดหวังว่า class ผู้ include จะมีให้
  (ในตัวอย่างคือ `name`) — นี่คือรูปแบบ **implicit interface** ที่ module กำหนดว่า "ถ้าจะ
  include ฉัน คุณต้องมี method `name` ให้ฉันเรียกใช้ได้ด้วยนะ"

```ruby
# ตรวจสอบว่า class หนึ่ง include module ไหนบ้างได้ด้วย include?
Duck.include?(Walkable)   # => true
Duck.include?(Comparable) # => false
```

---

## Step 97: `extend` — เพิ่ม class method / singleton method

ถ้า `include` เพิ่ม **instance method** ให้ทุก object ของ class นั้น `extend` ก็คือฝาแฝดที่
ทำงานตรงข้าม — มันเพิ่ม method ของ module เข้าไปเป็น **method ของ object เดียวที่เรียก
`extend` เท่านั้น** (หรือถ้าเรียก `extend` ที่ตัว class เอง ก็จะกลายเป็น **class method**
เพราะ class ก็คือ object ตัวหนึ่งเหมือนกันใน Ruby)

```ruby
# frozen_string_literal: true

module Auditable
  def audit_log
    "[AUDIT] #{name} class ถูกเรียกใช้งาน"
  end
end

class Report
  extend Auditable      # extend ที่ class -> ได้ class method

  def self.name = "Report"
end

puts Report.audit_log
# => [AUDIT] Report class ถูกเรียกใช้งาน

# --- extend กับ object เดี่ยวๆ ---
module Shoutable
  def shout
    upcase + "!!!"
  end
end

obj = String.new("hello")
obj.extend(Shoutable)   # extend ที่ instance ตัวเดียว -> ได้ singleton method เฉพาะตัวนี้
puts obj.shout
# => HELLO!!!

other = String.new("world")
other.shout
# => NoMethodError: undefined method `shout' for an instance of String
# (other ไม่ได้ extend Shoutable จึงไม่มี method นี้ — ผลกระทบจำกัดแค่ obj ตัวเดียว)
```

### ตารางเปรียบเทียบ `include` vs `extend`

| | `include Module` | `extend Module` |
|---|---|---|
| เพิ่ม method ให้ | **instance ทุกตัว** ของ class | **object เดียว** ที่เรียก (หรือทุก class method ถ้าเรียกที่ class) |
| ตำแหน่งใน ancestor chain | แทรกอยู่ระหว่าง class กับ superclass | แทรกเข้า **singleton class** ของ object นั้น ไม่ปรากฏใน `ClassName.ancestors` ปกติ |
| ใช้เมื่อ | ต้องการให้ทุก instance มีพฤติกรรมร่วมกัน | ต้องการ utility method ระดับ class หรือปรับพฤติกรรม object เฉพาะตัว |

### สำนวนที่พบบ่อย: `extend self`

```ruby
module MathHelpers
  extend self  # ทำให้เรียก method ผ่าน MathHelpers.square(...) ได้โดยตรง

  def square(x)
    x * x
  end
end

puts MathHelpers.square(5)
# => 25
```

`extend self` เป็นสำนวนยอดนิยมสำหรับเขียน module ที่รวม utility function ไว้ — ทำให้ method
ที่นิยามแบบ instance method ธรรมดา (`def square`) ถูกเรียกได้ทั้งแบบ module method
(`MathHelpers.square`) และยังนำไป `include` เข้า class อื่นเพื่อใช้แบบ instance method ได้
เหมือนเดิมถ้าต้องการ — ยืดหยุ่นกว่าการนิยาม `def self.square` ตรงๆ ตายตัว

---

## Step 98: Method Resolution Order (MRO) — ห่วงโซ่การค้นหา method อย่างละเอียด

เมื่อเรียก method บน object ตัวหนึ่ง Ruby ไม่ได้ "เดา" ว่า method อยู่ตรงไหน แต่ไล่ค้นหา
ตามลำดับที่แน่นอนเสมอ ลำดับนี้เรียกว่า **Method Resolution Order (MRO)** และดูได้ตรงๆ ด้วย
`ClassName.ancestors`

**กฎการเรียงลำดับ:**

1. Singleton class ของ object (ถ้ามี method ที่ใส่ผ่าน `extend`/`def obj.method`)
2. Class ของ object เอง
3. Module ที่ `include` เข้า class นั้น เรียงจาก **module ที่ include ทีหลังสุดอยู่ใกล้สุด**
4. Superclass ของ class นั้น
5. Module ที่ superclass include
6. ไล่ขึ้นไปเรื่อยๆ จนถึง `Object` → `Kernel` (module) → `BasicObject`

```ruby
# frozen_string_literal: true

module M1
  def who
    "M1"
  end
end

module M2
  def who
    "M2"
  end
end

class C
  include M1
  include M2
end

puts C.new.who
# => M2   (M2 include ทีหลัง จึงอยู่ใกล้ C มากกว่า ถูกค้นเจอก่อน)

C.ancestors
# => [C, M2, M1, Object, Kernel, BasicObject]
```

Ruby ค้นหา method ของ `C.new.who` โดยไล่ตามลิสต์ `ancestors` ทีละตัวจากซ้ายไปขวา —
เจอ `who` ใน `M2` ก่อน (เพราะอยู่ก่อน `M1` ใน chain) จึงหยุดค้นและใช้ตัวนั้นทันที ถ้า `C`
เองมีการนิยาม `def who` ตรงๆ ด้วย method นั้นจะชนะทุกอย่างเพราะ `C` อยู่หัวลิสต์เสมอ

### `super` เดินตาม ancestor chain นี้เป๊ะๆ

จุดที่ทำให้ MRO สำคัญมากคือ `super` (ที่เรียนใน Step 92) **ไม่ได้เรียก "parent class"
เท่านั้น** แต่เรียก **method ถัดไปใน ancestor chain** ซึ่งอาจเป็น module ก็ได้ ไม่ใช่แค่
superclass เสมอไป — เช่นถ้า `C < D` และ `D` include module ที่มี `who` ด้วย `super` ใน `C`
ก็จะไปเจอ module นั้นก่อนที่จะไปถึงตัว `D` เองเสียอีก (ตาม `ancestors` ที่แสดงจริง)

### หมายเหตุ: `prepend` (แนวคิดขั้นสูงที่ควรรู้จักไว้)

นอกจาก `include` (แทรกเข้า chain **ใต้** class) แล้ว Ruby ยังมี `prepend` ซึ่งแทรก module
เข้าไป **เหนือ** class แทน (module นั้นจะถูกค้นหา**ก่อน**ตัว class เองด้วยซ้ำ) ใช้สำหรับ
เทคนิคขั้นสูงอย่างการ wrap พฤติกรรมเดิมของ method โดยไม่ต้องแก้โค้ดต้นฉบับ (พบใน gem บาง
ตัวของ Rails ecosystem) — ยังไม่ต้องใช้ตอนนี้ แค่รู้จักชื่อไว้ก่อน จะกลับมาเจอโดยละเอียดใน
Part 015 (Metaprogramming)

> **สรุป MRO ในหนึ่งประโยค:** เมื่อเรียก method บน object ใดๆ Ruby ไล่หาจากตัว object เอง
> ก่อน แล้วค่อยไล่ตาม `ancestors` ของ class นั้นจากซ้ายไปขวา เจอตัวแรกที่ไหนก็ใช้ตัวนั้นทันที
> การเข้าใจลำดับนี้คือกุญแจสำคัญในการ debug ปัญหา "ทำไม method นี้ทำงานไม่เหมือนที่คิด"
> เมื่อโค้ดมีทั้ง inheritance และ mixin ผสมกันหลายชั้น

---

## Step 99: Comparable และ Enumerable — Mixin ตัวจริงจาก Ruby Standard Library

ตอนนี้เราเข้าใจ mixin ในเชิงทฤษฎีแล้ว มาดู mixin 2 ตัวที่ Ruby เตรียมไว้ให้ใช้งานจริง และ
เป็นตัวอย่างที่ดีที่สุดว่า mixin ทรงพลังแค่ไหน — เพียงเรานิยาม method หลักตัวเดียว
ก็ได้ method อื่นอีกนับสิบตัวมาใช้ฟรีๆ

### Comparable — นิยาม `<=>` ตัวเดียว ได้ `<`, `<=`, `==`, `>`, `>=`, `between?`, `clamp` ครบ

```ruby
# frozen_string_literal: true

class Product
  include Comparable

  attr_reader :name, :price

  def initialize(name, price)
    @name = name
    @price = price
  end

  # <=> คือ "spaceship operator" ต้องคืนค่า -1 (น้อยกว่า), 0 (เท่ากับ), 1 (มากกว่า)
  def <=>(other)
    price <=> other.price
  end

  def to_s
    "#{name} (#{price})"
  end
end

p1 = Product.new("หูฟัง", 100)
p2 = Product.new("เมาส์", 50)

puts p1 > p2                              # => true
puts p1 == Product.new("ของอื่น", 100)     # => true (ราคาเท่ากันถือว่า == )
puts [p1, p2].sort.map(&:to_s).inspect     # => ["เมาส์ (50)", "หูฟัง (100)"]
puts [p1, p2].min                          # => เมาส์ (50)
```

**อธิบาย:** เราเขียนแค่ `<=>` (spaceship operator) ตัวเดียว โดยบอกว่า "เทียบ `Product` สอง
ตัวโดยดูจาก `price`" แล้ว `include Comparable` ก็ปลดล็อก `<`, `<=`, `==`, `>`, `>=`,
`between?`, `clamp`, และทำให้ `sort`, `min`, `max` บน Array ของ `Product` ใช้งานได้ทันที
โดยไม่ต้องเขียนเองสักบรรทัด — นี่คือพลังของ mixin: เขียน method เดียว ได้พฤติกรรมเป็นสิบ

### Enumerable — นิยาม `each` ตัวเดียว ได้ `map`, `select`, `sort`, `reduce`, `include?` ฯลฯ ครบ

```ruby
# frozen_string_literal: true

class Bookshelf
  include Enumerable

  def initialize
    @books = []
  end

  def add(book)
    @books << book
    self
  end

  # ต้องนิยาม each เพียง method เดียวเท่านั้น
  def each
    return enum_for(:each) unless block_given?

    @books.each { |book| yield book }
  end
end

shelf = Bookshelf.new
shelf.add("Ruby").add("Rails").add("Clean Code")

puts shelf.map(&:upcase).inspect             # => ["RUBY", "RAILS", "CLEAN CODE"]
puts shelf.select { |b| b.length > 4 }.inspect # => ["Rails", "Clean Code"]
puts shelf.sort.inspect                       # => ["Clean Code", "Rails", "Ruby"]
puts shelf.include?("Rails")                  # => true
```

**อธิบาย:**

- `Bookshelf` ไม่ใช่ Array แต่มี `each` ของตัวเองที่ yield หนังสือทีละเล่มออกไป — แค่นี้
  `include Enumerable` ก็ทำให้ `Bookshelf` ใช้ `map`, `select`, `sort`, `reduce`, `count`,
  `include?`, `min`, `max`, `first`, `to_a` และอีกหลายสิบ method จาก Part 004 (Array
  methods) ได้ทั้งหมด ทั้งที่ไม่ได้สืบทอดจาก `Array` เลยแม้แต่น้อย
- `return enum_for(:each) unless block_given?` เป็นแนวปฏิบัติมาตรฐาน — ถ้าเรียก `each`
  โดยไม่ส่ง block มา (เช่นเรียกผ่าน `shelf.each.with_index`) จะคืน Enumerator object แทนที่
  จะ error
- `Enumerable` เป็นตัวอย่างว่า **composition ผ่าน mixin** ทรงพลังกว่าการพยายามสืบทอดจาก
  `Array` ตรงๆ มาก เพราะ `Bookshelf` มีอิสระเก็บข้อมูลภายในเป็นอะไรก็ได้ (ในตัวอย่างนี้คือ
  `@books` Array แต่จะเป็น Hash หรือโครงสร้างอื่นก็ได้) ขอแค่มี `each` ที่ถูกต้อง

> **preview Rails:** `ActiveRecord::Relation` (ผลลัพธ์จาก `Post.where(...)` ที่จะเรียนใน
> Phase 3–4) ก็ `include Enumerable` เช่นกัน นี่คือเหตุผลที่เขียน
> `Post.where(published: true).map(&:title)` หรือ `.select { ... }` ได้ทันทีราวกับมันเป็น
> Array — มันไม่ใช่ Array จริงๆ แต่ mixin ทำให้ "ทำตัวเหมือน" Array ได้ครบถ้วน

---

## Step 100: แบบฝึกหัดปิด Phase 1 — โปรเจกต์ Library Management CLI

ถึงเวลาปิด Phase 1 ด้วยโปรเจกต์ที่รวมทุกอย่างที่เรียนมาตลอด Part 001–010 เข้าด้วยกัน:
ตัวแปร/ชนิดข้อมูล, String, Array/Hash, control flow, method, block, class/object, และ
inheritance/module ที่เพิ่งเรียนจบ

### โจทย์

เขียนโปรแกรม `library_cli.rb` เป็นระบบจัดการห้องสมุดแบบ command line โดยต้องมี:

1. **Class hierarchy ด้วย inheritance:** `Item` เป็น base class, มี `Book` และ `Magazine`
   เป็น subclass ที่มี attribute เพิ่มเติมเฉพาะตัว (ISBN/จำนวนหน้า สำหรับ `Book`,
   ฉบับที่/สำนักพิมพ์ สำหรับ `Magazine`)
2. **Mixin:** module `Borrowable` ที่ให้ความสามารถยืม-คืนแก่ `Item` ผ่าน `include`
3. **Class variable/class method:** นับจำนวนรายการทั้งหมดที่เคยสร้างในระบบ
4. **`attr_reader`/`attr_accessor`** สำหรับ attribute ต่างๆ
5. **Block/`each`** สำหรับแสดงรายการ
6. **Command loop** (`gets` + `case`) เป็นเมนู: เพิ่มรายการ, แสดงรายการทั้งหมด, ยืม, คืน,
   ค้นหา, แสดงสถิติ, ออกจากโปรแกรม

### แนวคิดการออกแบบ (Design Decisions)

ก่อนดูเฉลย ลองพิจารณาเหตุผลเบื้องหลังการออกแบบแต่ละจุด — เพราะเหตุผลเหล่านี้คือสิ่งที่จะ
ย้ายไปใช้ตอนออกแบบ Model/Controller ใน Rails ตั้งแต่ Phase 3 เป็นต้นไป:

- **`Item` เป็น base class, `Book`/`Magazine` เป็น subclass** — เพราะทั้งสามอย่างมี
  attribute ร่วมกัน (`title`, `author`, `year`) และพฤติกรรมร่วมกัน (การยืม-คืน, การแสดงผล)
  แต่ก็มีข้อมูลเฉพาะตัวที่ต่างกัน นี่คือความสัมพันธ์แบบ "is-a" ที่เหมาะกับ inheritance
  ตรงตามที่อธิบายไว้ใน Step 91 — **ใน Rails: จะกลายเป็น `Book < Item` และ
  `Magazine < Item` ด้วย Single Table Inheritance (STI) ซึ่งเรียนใน Phase 4 เป๊ะๆ**
- **`Borrowable` เป็น module แยกออกมา ไม่ยัดเข้าไปใน `Item` ตรงๆ** — เพราะ "การยืมได้"
  เป็นความสามารถ (capability) ที่แยกจาก "การเป็น Item" — ถ้าวันหนึ่งอยากเพิ่ม class
  `MeetingRoom` ที่ไม่ใช่ `Item` เลย แต่ก็ "ยืมได้" เหมือนกัน ก็แค่ `include Borrowable`
  เข้าไปได้ทันทีโดยไม่ต้องพึ่ง inheritance เลย — นี่คือประโยชน์ของการแยก mixin ตามที่
  อธิบายใน Step 96
- **`@@total_items` เป็น class variable ของ `Item`** — เพราะเราต้องการนับจำนวนรวมของ
  **ทุก** Item ที่เคยสร้าง ไม่ว่าจะเป็น `Book` หรือ `Magazine` ก็ตาม class variable ที่อยู่
  บน superclass จะถูกใช้ร่วมกันโดย subclass ทั้งหมดโดยอัตโนมัติ (ทบทวนจาก Part 009)
- **`Library` เป็น class แยกต่างหาก ไม่ใช่ global array** — เพื่อห่อหุ้ม (encapsulate)
  logic การค้นหา/ยืม/คืนไว้ในที่เดียว แทนที่จะกระจายโค้ดจัดการ Array ไปทั่วทั้งโปรแกรม
  — **นี่คือแนวคิดเดียวกับที่ Controller ใน Rails เรียก Model แทนที่จะจัดการ database
  ตรงๆ เอง**
- **CLI loop แยกจาก logic ของ class ทั้งหมด** — ฟังก์ชัน `print_menu`, `prompt_new_item`
  และ `loop do ... end` ทำหน้าที่แค่ "รับ input และเรียก method ของ `Library`/`Item`"
  ไม่มี logic ทางธุรกิจ (business logic) ปนอยู่เลย — **นี่คือหลักการเดียวกับที่ Controller
  ใน Rails ไม่ควรมี business logic เอง แต่ควรเรียกใช้ Model แทน**

### เฉลยฉบับเต็ม

```ruby
# frozen_string_literal: true

# library_cli.rb — Library Management CLI
# โปรเจกต์ปิด Phase 1: Ruby Fundamentals

# ============================================================
# Mixin: Borrowable
# เพิ่ม "ความสามารถในการยืม-คืน" ให้กับ class ใดก็ได้ที่ include เข้าไป
# ============================================================
module Borrowable
  def borrow!(borrower_name)
    if borrowed?
      puts "  [!] \"#{title}\" ถูกยืมอยู่แล้วโดย #{@borrowed_by}"
      return false
    end

    @borrowed_by = borrower_name
    @borrowed_at = Time.now
    puts "  [OK] #{borrower_name} ยืม \"#{title}\" เรียบร้อยแล้ว"
    true
  end

  def return!
    unless borrowed?
      puts "  [!] \"#{title}\" ไม่ได้ถูกยืมอยู่ ไม่ต้องคืน"
      return false
    end

    puts "  [OK] คืน \"#{title}\" เรียบร้อยแล้ว (ก่อนหน้านี้ยืมโดย #{@borrowed_by})"
    @borrowed_by = nil
    @borrowed_at = nil
    true
  end

  def borrowed?
    !@borrowed_by.nil?
  end

  def status
    borrowed? ? "ถูกยืมโดย #{@borrowed_by}" : "ว่าง (พร้อมให้ยืม)"
  end
end

# ============================================================
# Base class: Item
# ============================================================
class Item
  include Borrowable
  include Comparable

  @@total_items = 0

  attr_reader :id, :title, :author, :year

  def initialize(title:, author:, year:)
    @@total_items += 1
    @id = @@total_items
    @title = title
    @author = author
    @year = year
    @borrowed_by = nil
    @borrowed_at = nil
  end

  def self.total_items
    @@total_items
  end

  # ให้ Item เรียงลำดับ (sort) กันได้ตามชื่อเรื่อง โดยอาศัย Comparable
  def <=>(other)
    title <=> other.title
  end

  def category
    "ทั่วไป"
  end

  def to_s
    "##{id} [#{category}] \"#{title}\" โดย #{author} (#{year}) - #{status}"
  end
end

# ============================================================
# Subclass: Book
# ============================================================
class Book < Item
  attr_reader :isbn, :pages

  def initialize(title:, author:, year:, isbn:, pages:)
    super(title: title, author: author, year: year)
    @isbn = isbn
    @pages = pages
  end

  def category
    "หนังสือ"
  end

  def to_s
    super() + " | ISBN: #{isbn}, #{pages} หน้า"
  end
end

# ============================================================
# Subclass: Magazine
# ============================================================
class Magazine < Item
  attr_reader :issue_number, :publisher

  def initialize(title:, author:, year:, issue_number:, publisher:)
    super(title: title, author: author, year: year)
    @issue_number = issue_number
    @publisher = publisher
  end

  def category
    "นิตยสาร"
  end

  def to_s
    super() + " | ฉบับที่ #{issue_number}, สำนักพิมพ์ #{publisher}"
  end
end

# ============================================================
# Library — เก็บและจัดการ Item ทั้งหมด
# ============================================================
class Library
  attr_reader :name, :items

  def initialize(name)
    @name = name
    @items = []
  end

  def add_item(item)
    @items << item
    puts "เพิ่ม \"#{item.title}\" เข้าห้องสมุด \"#{name}\" เรียบร้อยแล้ว (ID: #{item.id})"
  end

  def list_items
    if @items.empty?
      puts "ยังไม่มีรายการในห้องสมุด"
      return
    end
    @items.sort.each { |item| puts item }
  end

  def find_by_id(id)
    @items.find { |item| item.id == id }
  end

  def search(keyword)
    downcased = keyword.downcase
    @items.select do |item|
      item.title.downcase.include?(downcased) || item.author.downcase.include?(downcased)
    end
  end

  def borrow_item(id, borrower_name)
    item = find_by_id(id)
    return puts("ไม่พบรายการ ID #{id}") unless item

    item.borrow!(borrower_name)
  end

  def return_item(id)
    item = find_by_id(id)
    return puts("ไม่พบรายการ ID #{id}") unless item

    item.return!
  end

  def borrowed_count
    items.count(&:borrowed?)
  end
end

# ============================================================
# CLI — ส่วนติดต่อผู้ใช้
# ============================================================
def print_menu
  puts "\n#{'=' * 55}"
  puts "  ระบบจัดการห้องสมุด (Library Management CLI)"
  puts "=" * 55
  puts "1) เพิ่มรายการใหม่ (หนังสือ/นิตยสาร)"
  puts "2) แสดงรายการทั้งหมด"
  puts "3) ยืมรายการ"
  puts "4) คืนรายการ"
  puts "5) ค้นหารายการ"
  puts "6) แสดงสถิติห้องสมุด"
  puts "0) ออกจากโปรแกรม"
  print "เลือกเมนู: "
end

def prompt_new_item
  print "ชนิด (1=หนังสือ, 2=นิตยสาร): "
  type = gets.chomp
  print "ชื่อเรื่อง: "
  title = gets.chomp
  print "ผู้แต่ง: "
  author = gets.chomp
  print "ปีที่พิมพ์: "
  year = gets.chomp.to_i

  if type == "1"
    print "ISBN: "
    isbn = gets.chomp
    print "จำนวนหน้า: "
    pages = gets.chomp.to_i
    Book.new(title: title, author: author, year: year, isbn: isbn, pages: pages)
  else
    print "ฉบับที่: "
    issue_number = gets.chomp
    print "สำนักพิมพ์: "
    publisher = gets.chomp
    Magazine.new(title: title, author: author, year: year,
                 issue_number: issue_number, publisher: publisher)
  end
end

library = Library.new("ห้องสมุดชุมชน Ruby")

loop do
  print_menu
  choice = gets&.chomp

  case choice
  when "1"
    library.add_item(prompt_new_item)
  when "2"
    library.list_items
  when "3"
    print "ID ของรายการที่จะยืม: "
    id = gets.chomp.to_i
    print "ชื่อผู้ยืม: "
    borrower = gets.chomp
    library.borrow_item(id, borrower)
  when "4"
    print "ID ของรายการที่จะคืน: "
    id = gets.chomp.to_i
    library.return_item(id)
  when "5"
    print "คำค้นหา (ชื่อเรื่อง/ผู้แต่ง): "
    keyword = gets.chomp
    results = library.search(keyword)
    if results.empty?
      puts "ไม่พบรายการที่ตรงกับ \"#{keyword}\""
    else
      results.each { |item| puts item }
    end
  when "6"
    puts "จำนวนรายการทั้งหมดที่เคยสร้างในระบบ: #{Item.total_items}"
    puts "จำนวนรายการใน \"#{library.name}\": #{library.items.size}"
    puts "กำลังถูกยืมอยู่: #{library.borrowed_count} รายการ"
  when "0", nil
    puts "ขอบคุณที่ใช้บริการ ลาก่อน!"
    break
  else
    puts "กรุณาเลือกเมนูให้ถูกต้อง"
  end
end
```

### ทดสอบรัน

```bash
ruby library_cli.rb
```

ตัวอย่าง session การใช้งานจริง (เพิ่มหนังสือ 1 เล่ม → ยืม → ค้นหา → ดูสถิติ → คืน):

```
เลือกเมนู: 1
ชนิด (1=หนังสือ, 2=นิตยสาร): 1
ชื่อเรื่อง: The Ruby Way
ผู้แต่ง: Hal Fulton
ปีที่พิมพ์: 2015
ISBN: 978-0-13-345659-7
จำนวนหน้า: 600
เพิ่ม "The Ruby Way" เข้าห้องสมุด "ห้องสมุดชุมชน Ruby" เรียบร้อยแล้ว (ID: 1)

เลือกเมนู: 3
ID ของรายการที่จะยืม: 1
ชื่อผู้ยืม: Somchai
  [OK] Somchai ยืม "The Ruby Way" เรียบร้อยแล้ว

เลือกเมนู: 5
คำค้นหา (ชื่อเรื่อง/ผู้แต่ง): ruby
#1 [หนังสือ] "The Ruby Way" โดย Hal Fulton (2015) - ถูกยืมโดย Somchai | ISBN: 978-0-13-345659-7, 600 หน้า

เลือกเมนู: 6
จำนวนรายการทั้งหมดที่เคยสร้างในระบบ: 1
จำนวนรายการใน "ห้องสมุดชุมชน Ruby": 1
กำลังถูกยืมอยู่: 1 รายการ

เลือกเมนู: 4
ID ของรายการที่จะคืน: 1
  [OK] คืน "The Ruby Way" เรียบร้อยแล้ว (ก่อนหน้านี้ยืมโดย Somchai)

เลือกเมนู: 0
ขอบคุณที่ใช้บริการ ลาก่อน!
```

โค้ดนี้ทดสอบรันจริงแล้วบน Ruby 3.3.6 ครบทุกเมนู (เพิ่มหนังสือ, เพิ่มนิตยสาร, แสดงรายการ,
ยืม, ค้นหา, ดูสถิติ, คืน, ออกจากโปรแกรม) ทำงานถูกต้องตามที่ออกแบบไว้ทุกจุด

### จุดที่ควรสังเกตในเฉลย

- **`super()` ใน `to_s` ของ `Book`/`Magazine`** — ใช้วงเล็บว่างเพราะ `to_s` ไม่รับ argument
  ใดๆ เลย การเขียน `super()` ชัดเจนกว่า bare `super` ว่าตั้งใจไม่ส่งอะไรไปเพิ่ม (แม้ผลลัพธ์
  จะเหมือนกันในเคสนี้เพราะไม่มี argument ให้ส่งต่ออยู่แล้ว) ส่วน `super(title: title, ...)`
  ใน `initialize` ใช้รูปแบบ "ส่ง argument ที่ระบุเอง" เพราะต้องกรองเอาเฉพาะ keyword ที่
  `Item#initialize` ต้องการ ไม่ส่ง `isbn`/`pages` ปนไปด้วย (ทบทวน Step 92)
- **`include Comparable` บน `Item`** ทำให้ `@items.sort` ใน `list_items` ใช้งานได้ทันที
  โดยเรียงตามชื่อเรื่อง (ทบทวน Step 99) — และเพราะ `Book`/`Magazine` สืบทอด `Item` มา
  จึงได้ `Comparable` มาโดยอัตโนมัติผ่าน ancestor chain (ทบทวน Step 98)
- **`category` เป็น method ที่แต่ละ subclass override** (`"ทั่วไป"` → `"หนังสือ"` /
  `"นิตยสาร"`) เป็นตัวอย่างของ **polymorphism**: `Library#list_items` เรียก `item.to_s`
  แบบเดียวกันกับทุก item โดยไม่ต้องรู้เลยว่าตัวไหนเป็น `Book` หรือ `Magazine` — แต่ละ object
  จัดการแสดงผลของตัวเองถูกต้องเสมอ (ทบทวนแนวคิด override จาก Step 91)
- **`gets&.chomp`** ใช้ safe navigation operator (`&.`) เผื่อกรณี input stream ถูกปิด
  (เช่น กด Ctrl+D) ซึ่งจะทำให้ `gets` คืน `nil` — ถ้าไม่มี `&.` โปรแกรมจะ error แทนที่จะ
  ออกโปรแกรมอย่างนุ่มนวลผ่าน `when "0", nil`

> **สะพานเชื่อมสู่ Rails:** สังเกตว่าโครงสร้างนี้ — `Item`/`Book`/`Magazine` (ข้อมูล +
> พฤติกรรม), `Library` (จัดการ collection), CLI loop (รับ input, เรียก Model, แสดงผล) —
> คือ**โครงร่างเดียวกันกับ MVC** ที่จะเรียนตั้งแต่ Phase 3 เป๊ะๆ: `Item`/`Book`/`Magazine`
> จะกลายเป็น **ActiveRecord Model** (พร้อม STI), `Library` logic จะกระจายไปอยู่ใน
> **Controller actions** และ database queries, ส่วนเมนู CLI จะถูกแทนที่ด้วย **View** (HTML
> form/link) ที่ผู้ใช้คลิกแทนการพิมพ์ตัวเลข ทักษะ OOP design ที่ฝึกในโปรเจกต์นี้จึงไม่ได้
> "ทิ้งไป" เมื่อเข้า Rails — มันคือรากฐานเดียวกัน แค่เปลี่ยนวิธีรับ input/แสดง output เท่านั้น

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **Persist ข้อมูลลงไฟล์ JSON** — แก้โปรแกรมให้บันทึกรายการทั้งหมดลงไฟล์ `library.json`
   ทุกครั้งที่มีการเพิ่ม/ยืม/คืน และโหลดข้อมูลกลับมาตอนเริ่มโปรแกรมใหม่ (ใบ้: ใช้
   `require "json"`, เขียน method `to_h` ใน `Item` และ subclass เพื่อแปลง object เป็น Hash
   ก่อน serialize — เรื่อง JSON แบบเต็มจะเรียนใน Part 012)
2. **เพิ่ม class `Member`** — สร้าง class ผู้ใช้บริการห้องสมุดที่มี `name`, `member_id`,
   และ limit จำนวนหนังสือที่ยืมพร้อมกันได้สูงสุด (เช่น 3 เล่ม) แก้ `borrow_item` ให้รับ
   `Member` object แทน string ชื่อ แล้วตรวจสอบ limit ก่อนอนุญาตให้ยืม
3. **เพิ่มวันครบกำหนดคืน (due date) และค่าปรับ** — เมื่อยืม ให้กำหนดวันครบกำหนดคืนอัตโนมัติ
   (เช่น 14 วันจากวันที่ยืม โดยใช้ `Date`/`Time` ที่เคยเจอใน Part 001) แล้วเมื่อคืนช้ากว่า
   กำหนด ให้คำนวณค่าปรับตามจำนวนวันที่เกิน

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ใช้ **Class Inheritance** (`class Child < Parent`) สร้างความสัมพันธ์แบบ "is-a" และเข้าใจ
  ว่าเมื่อใดควรใช้ inheritance เมื่อใดควรใช้ composition แทน
- เข้าใจ `super` ทั้ง 3 รูปแบบ (bare, with args, explicit empty args) และรู้ว่าแต่ละแบบ
  ส่ง argument ต่างกันอย่างไร
- แยกความแตกต่างของ `protected` กับ `private` ได้ชัดเจน โดยเฉพาะกรณีเปรียบเทียบ object
  ประเภทเดียวกัน
- ตรวจสอบชนิดของ object ได้ด้วย `is_a?`/`kind_of?`/`instance_of?`/`ancestors` และรู้ว่า
  เมื่อไหร่ควรใช้ duck typing แทน
- ใช้ Module เป็น **namespace** เพื่อจัดกลุ่มโค้ดและป้องกันชื่อชนกัน
- ใช้ Module เป็น **Mixin** ผ่าน `include` (เพิ่ม instance method) และ `extend` (เพิ่ม
  class/singleton method) พร้อมเข้าใจความแตกต่างของทั้งสอง
- เข้าใจ **Method Resolution Order (MRO)** และวิธีที่ Ruby ค้นหา method ผ่าน `ancestors`
  รวมถึงบทบาทของ `super` ในห่วงโซ่นี้
- ใช้ **Comparable** (`<=>`) และ **Enumerable** (`each`) ซึ่งเป็น mixin ตัวจริงจาก Ruby
  standard library เป็นตัวอย่างพลังของการออกแบบด้วย mixin
- สร้างโปรเจกต์ **Library Management CLI** ที่รวม inheritance, mixin, class
  variable/method, attr_accessor, block, และ command loop เข้าด้วยกันเป็นโปรแกรมที่ใช้งาน
  ได้จริง พร้อมเข้าใจว่าโครงสร้างนี้จะแปลงเป็น Rails MVC ได้อย่างไรในอนาคต

## สรุปภาพรวม Phase 1: Ruby Fundamentals

ยินดีด้วย! ตอนนี้ **Phase 1: Ruby Fundamentals (Part 001–010, Step 1–100)** เสร็จสมบูรณ์
แล้ว เราเดินทางมาจากการติดตั้ง Ruby และพิมพ์ "Hello World" (Part 001) ผ่านชนิดข้อมูล
พื้นฐาน, String, Array, Hash (Part 002–005), control flow และ method (Part 006–007),
block/Proc/Lambda (Part 008), OOP เบื้องต้นด้วย Class/Object (Part 009), จนมาถึง
Inheritance/Module/Mixin และโปรเจกต์รวบยอด Library Management CLI ใน Part นี้ — ครบทั้ง
syntax พื้นฐาน, การจัดการข้อมูล, และหลักการออกแบบ OOP ที่เป็นรากฐานของทุกอย่างที่จะเรียน
ต่อจากนี้ ไม่ว่าจะเป็น Ruby ขั้นสูงใน Phase 2 หรือ Rails framework เองตั้งแต่ Phase 3
เป็นต้นไป

**ต่อไป (Part 011 — เปิด Phase 2: Ruby Deep Dive):** เราจะเรียนรู้การจัดการข้อผิดพลาดอย่าง
เป็นระบบด้วย **Exception Handling** — `begin`/`rescue`/`ensure`, การสร้าง custom exception
class ของตัวเอง, และกลไก `retry` สำหรับลองทำงานซ้ำเมื่อเกิดข้อผิดพลาดชั่วคราว ซึ่งเป็นทักษะ
จำเป็นสำหรับเขียนโปรแกรมที่ทนทานต่อความผิดพลาดในโลกจริง (เช่น การเชื่อมต่อ network ล้มเหลว
หรือข้อมูล input ที่ไม่ถูกต้อง)
