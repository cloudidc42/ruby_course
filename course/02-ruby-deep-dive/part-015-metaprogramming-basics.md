# Part 015: Metaprogramming เบื้องต้น — method_missing, define_method, send, respond_to?

> **Step ครอบคลุมใน Part นี้:** Step 141–150
> **ระดับ:** กลาง (ต้องผ่าน Part 001–014 มาก่อน โดยเฉพาะ Part 007 เรื่อง Methods,
> Part 009–010 เรื่อง OOP/Module และ Part 014 เรื่อง Struct/OpenStruct/Data)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 141: Metaprogramming คืออะไร และทำไม Rails ถึงพึ่งพามันหนักมาก
- Step 142: `send` และ `public_send` — เรียก method แบบไดนามิกด้วยชื่อ/symbol
- Step 143: `respond_to?` — ตรวจสอบก่อนว่า object มี method นี้จริงไหม
- Step 144: `define_method` — นิยาม method แบบไดนามิกตอนรัน class
- Step 145: ใช้ loop ร่วมกับ `define_method` เพื่อ generate method หลายตัวแบบ DRY
- Step 146: `method_missing` + `respond_to_missing?` — คู่หูที่ต้องมาด้วยกันเสมอ
- Step 147: ตัวอย่างจริง — สร้าง dynamic attribute object คล้าย ActiveRecord/OpenStruct
- Step 148: `instance_variable_get`/`instance_variable_set` — เข้าถึง instance variable แบบไดนามิก
- Step 149: `class_eval` และ `instance_eval` เบื้องต้น
- Step 150: แบบฝึกหัด — สร้าง `DynamicRecord` และข้อควรระวังในการใช้ metaprogramming

---

## Step 141: Metaprogramming คืออะไร และทำไม Rails ถึงพึ่งพามันหนักมาก

### นิยาม

**Metaprogramming** คือการเขียนโค้ดที่ "เขียนหรือดัดแปลงโค้ดอื่น" ในขณะที่โปรแกรมกำลังรัน
อยู่ (runtime) แทนที่จะกำหนดทุกอย่างตายตัวตอนเขียนโค้ด (compile-time/เขียนมือ) พูดง่ายๆ
คือแทนที่จะเขียน method ทีละตัวด้วยมือ เราเขียนโค้ดที่ **สร้าง method ขึ้นมาเอง** ตาม
เงื่อนไขหรือข้อมูลที่มีอยู่

Ruby เหมาะกับ metaprogramming เป็นพิเศษเพราะเป็นภาษาที่ **dynamic และ reflective** สูงมาก
นั่นคือ:

- Class ใน Ruby ไม่ได้ "ปิดตาย" หลังนิยามแล้ว — เปิด (reopen) แล้วเพิ่ม method เข้าไปทีหลัง
  ได้เสมอ (เรียกว่า **open classes** หรือ **monkey patching**)
- Object ทุกตัวสามารถ "สำรวจตัวเอง" ได้ว่ามี method อะไรบ้าง มี instance variable อะไรบ้าง
  ผ่าน method อย่าง `methods`, `instance_variables`, `respond_to?`
- เราเขียนโค้ดที่ "เขียนโค้ด" ได้จริงๆ ผ่าน `define_method`, `class_eval`, `method_missing`
  ที่จะเรียนใน Part นี้ทั้งหมด

```ruby
# frozen_string_literal: true

# ตัวอย่างง่ายที่สุดของ "การสำรวจตัวเอง" (introspection) ซึ่งเป็นรากฐานของ metaprogramming
puts 42.class                 # => Integer
puts 42.is_a?(Numeric)         # => true
puts "hello".respond_to?(:upcase)   # => true
puts "hello".respond_to?(:fly)      # => false
p "hello".methods.grep(/case/)      # => [:upcase, :downcase, :swapcase, :casecmp, ...]
```

**อธิบาย:** `.class`, `.is_a?`, `.respond_to?`, `.methods` ทั้งหมดนี้คือ **reflection API**
ที่ Ruby เตรียมไว้ให้ object ทุกตัวสามารถตอบคำถามเกี่ยวกับตัวเองได้ — นี่คือพื้นฐานที่ทำให้
metaprogramming เป็นไปได้ เพราะก่อนจะ "สร้างหรือเรียก method แบบไดนามิก" เราต้องมีวิธี
"ถาม" object ก่อนว่ามันเป็นอะไร มี method อะไรบ้าง

### ทำไม Rails ถึงพึ่งพา metaprogramming อย่างหนัก

ถ้าใครเคยเห็นโค้ด Rails มาก่อน (หรือดูตัวอย่างที่โผล่มาใน Part 007) น่าจะเคยสงสัยว่า
โค้ดแบบนี้ "ทำงานได้ยังไง":

```ruby
# ตัวอย่างที่จะเจอจริงตอนเรียน Rails (แสดงให้ดูล่วงหน้าเพื่อให้เห็นภาพความเชื่อมโยง)
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
  validates :title, presence: true, length: { maximum: 200 }
end

post = Post.new(title: "Hello Rails")
post.title              # => "Hello Rails"  (method นี้ "โผล่มาเอง" จากไหน?)
post.title = "Changed"  # => setter ก็โผล่มาเองเช่นกัน
post.comments           # => association ที่ไม่มีใครเขียน method นี้ไว้เลย
```

คำตอบคือ: **ไม่มีใครเขียน method `title`, `title=`, `comments` ไว้ตรงๆ ในโค้ด** ทั้งหมดนี้
ถูก **สร้างขึ้นแบบไดนามิกตอนรันโปรแกรม** โดยใช้เทคนิคที่เราจะเรียนใน Part นี้ทั้งสิ้น:

1. `has_many :comments` — เบื้องหลังใช้ `define_method` (หรือกลไกที่คล้ายกัน) สร้าง method
   `comments`, `comments=`, `comment_ids`, `build_comment` ฯลฯ ขึ้นมาให้อัตโนมัติ จาก
   ชื่อ `:comments` เพียงตัวเดียว
2. `validates :title, presence: true` — เป็น method ธรรมดาที่รับ arguments แล้วไปลง
   "ทะเบียน validation" ของ class แบบไดนามิก (เราจะเห็นกลไกคล้ายกันตอนสร้าง DSL ด้วย
   `class_eval` ใน Step 149)
3. `post.title` — ActiveRecord อ่านชื่อ column จากฐานข้อมูล (`title`, `body`, `created_at`
   ฯลฯ) แล้ว **generate getter/setter method ให้ทุก column โดยอัตโนมัติ** ผ่านเทคนิคที่ใกล้
   เคียงกับ `define_method` ที่เราจะเขียนเองใน Step 144–145
4. `rails generate model Post title:string` — เป็นเครื่องมือ generator ที่ "เขียนไฟล์โค้ด"
   ให้อัตโนมัติจาก template — เป็นอีกรูปแบบหนึ่งของการ "เขียนโค้ดที่สร้างโค้ด"

พูดสั้นๆ คือ **Rails ไม่ได้มี "เวทมนตร์"** แต่ใช้เทคนิค metaprogramming มาตรฐานของ Ruby
(ที่เราจะเรียนตรงนี้) มาผสมกันอย่างเป็นระบบ เพื่อลดโค้ดซ้ำซ้อนและทำให้ API อ่านเหมือน
ภาษาอังกฤษ ตรงกับปรัชญา "Optimize for programmer happiness" ที่กล่าวถึงตั้งแต่ Part 001

> **สิ่งสำคัญที่ต้องจำ:** เมื่อเข้าใจ `send`, `respond_to?`, `define_method`, และ
> `method_missing` แล้ว จะไม่มีโค้ด Rails ตัวไหนดู "เป็นเวทมนตร์" อีกต่อไป — ทุกอย่าง
> อธิบายได้ด้วยกลไกพื้นฐานของ Ruby ล้วนๆ

### ข้อควรรู้ล่วงหน้า

Metaprogramming เป็นเครื่องมือที่ **ทรงพลังมากแต่ก็มีต้นทุน** (debug ยากขึ้น, IDE
autocomplete พังบางส่วน, performance อาจช้ากว่าโค้ดตรงๆ) รายละเอียดเรื่องต้นทุนนี้จะพูดถึง
อย่างจริงจังใน Step 150 ท้าย Part แต่ให้จำไว้ตั้งแต่ตอนนี้ว่า: **เรียนรู้เพื่อ "อ่านเข้าใจ"
โค้ด Rails ก่อน แล้วค่อยใช้ "เขียนเอง" อย่างระมัดระวังและมีเหตุผลรองรับชัดเจน**

---

## Step 142: `send` และ `public_send` — เรียก method แบบไดนามิกด้วยชื่อ/symbol

ปกติเราเรียก method ด้วยชื่อที่เขียนตรงๆ ในโค้ด (`obj.some_method`) แต่บางสถานการณ์เราไม่รู้
ล่วงหน้าว่าจะเรียก method ชื่ออะไร (เช่น ชื่อ method มาจาก user input, มาจาก config,
หรือมาจากการวนลูป) — `send` แก้ปัญหานี้โดยให้เรา **เรียก method โดยส่งชื่อ (String หรือ
Symbol) เป็น argument แทน**

```ruby
# frozen_string_literal: true

text = "hello world"

puts text.upcase             # เรียกแบบปกติ => HELLO WORLD
puts text.send(:upcase)      # เรียกผ่าน send ด้วย Symbol => HELLO WORLD
puts text.send("upcase")     # เรียกผ่าน send ด้วย String ก็ได้เหมือนกัน => HELLO WORLD
```

**อธิบาย:** `send(:upcase)` มีผลเทียบเท่ากับ `.upcase` ทุกประการ ต่างกันแค่ **ชื่อ method
ที่จะเรียกกลายเป็นค่าที่กำหนดตอนรัน (runtime) แทนที่จะ hardcode ไว้ตอนเขียนโค้ด**

### ตัวอย่างที่เห็นประโยชน์ชัดเจน: เลือก method จากตัวแปร

```ruby
def apply_transformation(text, method_name)
  text.send(method_name)
end

puts apply_transformation("Ruby", :upcase)     # => RUBY
puts apply_transformation("Ruby", :downcase)   # => ruby
puts apply_transformation("Ruby", :reverse)    # => ybuR
```

ถ้าไม่มี `send` เราจะต้องเขียน `if/elsif` เทียบชื่อ method ทีละตัวแบบนี้แทน ซึ่งไม่ยืดหยุ่น
และต้องแก้โค้ดทุกครั้งที่มี method ใหม่เพิ่มเข้ามา:

```ruby
# แบบที่ไม่ยืดหยุ่น (ถ้าไม่ใช้ send)
def apply_transformation_verbose(text, method_name)
  case method_name
  when :upcase   then text.upcase
  when :downcase then text.downcase
  when :reverse  then text.reverse
  else raise ArgumentError, "ไม่รู้จัก method #{method_name}"
  end
end
```

### `send` สามารถส่ง argument เพิ่มเติมได้เหมือน method call ปกติ

```ruby
numbers = [5, 3, 8, 1, 9]

puts numbers.send(:[], 2)          # เทียบเท่า numbers[2] => 8
puts numbers.send(:push, 100)      # เทียบเท่า numbers.push(100)
p numbers                            # => [5, 3, 8, 1, 9, 100]

# ส่ง block ผ่าน send ก็ได้ด้วย
result = numbers.send(:select) { |n| n > 5 }
p result   # => [8, 9, 100]
```

### จุดสำคัญที่สุด: `send` เรียก **private method ได้ด้วย**

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง `send` กับการเรียก method แบบปกติ และเป็นจุดที่
ต้องระวังมาก:

```ruby
class OrderCalculator
  def total(price, quantity)
    subtotal = price * quantity
    subtotal + calculate_tax(subtotal)
  end

  private

  def calculate_tax(amount)
    amount * 0.07
  end
end

order = OrderCalculator.new

order.calculate_tax(200)
# NoMethodError: private method 'calculate_tax' called for #<OrderCalculator>

order.send(:calculate_tax, 200)
# => 14.0  (send เจาะทะลุ private ได้ตรงๆ!)
```

**อธิบาย:** `send` **ไม่สนใจ method visibility เลย** (`public`/`private`/`protected` ที่
เรียนใน Part 007 Step 69) มันเรียก method ได้หมดไม่ว่าจะเป็น visibility อะไร นี่คือทั้ง
จุดแข็งและจุดอ่อนของ `send`:

- **จุดแข็ง:** มีประโยชน์มากตอนเขียน **test** ที่ต้องการทดสอบ private method โดยตรง
  (แม้จะมีข้อถกเถียงในวงการว่าควรทดสอบ private method หรือไม่ — ส่วนใหญ่แนะนำให้ทดสอบผ่าน
  public interface แทน แต่บางกรณีก็จำเป็น) หรือใช้ใน metaprogramming ที่ต้องเรียก method
  ภายในของ object แบบยืดหยุ่น
- **จุดอ่อน (ความเสี่ยงด้านความปลอดภัย):** ถ้า `method_name` ที่ส่งเข้า `send` มาจาก
  **user input โดยตรง** (เช่นจาก web form, API parameter) ผู้ใช้ที่ประสงค์ร้ายอาจส่งชื่อ
  private/internal method ที่อันตรายเข้ามาได้ เช่น method ที่ลบข้อมูลหรือเข้าถึงข้อมูลลับ

```ruby
class UserAccount
  def initialize(balance)
    @balance = balance
  end

  def display_balance
    "ยอดเงิน: #{@balance}"
  end

  private

  def wipe_all_data!
    @balance = 0
    "ข้อมูลถูกล้างแล้ว!"
  end
end

account = UserAccount.new(5000)

# สมมติโค้ดนี้รับชื่อ method จาก parameter ของ request โดยไม่ระวัง (อันตรายมาก!)
untrusted_method_name = "wipe_all_data!"   # สมมติมาจาก params[:action] ที่ผู้ใช้ควบคุมได้
puts account.send(untrusted_method_name)
# => "ข้อมูลถูกล้างแล้ว!"  <- เรียก private method ที่ไม่ควรเรียกจากภายนอกได้สำเร็จ!
```

### `public_send` — ทางเลือกที่ปลอดภัยกว่าเมื่อรับชื่อ method จากแหล่งที่ไม่น่าเชื่อถือ

`public_send` ทำงานเหมือน `send` ทุกประการ **ยกเว้นว่ามันเรียกได้เฉพาะ public method
เท่านั้น** ถ้าชื่อ method ที่ส่งเข้ามาเป็น private/protected จะโยน `NoMethodError` ทันที
เหมือนการเรียกแบบปกติ

```ruby
# class UserAccount เดิมจากตัวอย่างก่อนหน้า (แสดงซ้ำเพื่อให้โค้ดนี้รันได้ครบในตัวเอง)
class UserAccount
  def initialize(balance)
    @balance = balance
  end

  def display_balance
    "ยอดเงิน: #{@balance}"
  end

  private

  def wipe_all_data!
    @balance = 0
    "ข้อมูลถูกล้างแล้ว!"
  end
end

account = UserAccount.new(5000)

puts account.public_send(:display_balance)   # => ยอดเงิน: 5000 (public method เรียกได้ปกติ)

account.public_send(:wipe_all_data!)
# NoMethodError: private method 'wipe_all_data!' called for #<UserAccount>
# ป้องกันการเรียก private method จากภายนอกได้สำเร็จ แม้ชื่อ method จะมาจาก input ก็ตาม
```

> **กฎปฏิบัติที่สำคัญมาก:** เมื่อชื่อ method ที่จะเรียกแบบไดนามิกมาจากแหล่งที่ **ไม่น่า
> เชื่อถือ** (user input, query parameter, ข้อมูลจากภายนอก) **ให้ใช้ `public_send`
> เสมอ ห้ามใช้ `send`** เพราะ `send` เปิดช่องให้เรียก private method ที่อาจเป็นอันตรายได้
> ใช้ `send` เฉพาะเมื่อชื่อ method มาจากโค้ดที่เราควบคุมเองทั้งหมด (เช่น hardcode ไว้ใน
> array ของ symbol ที่เราเขียนเอง) หรือใช้ในบริบทของการทดสอบ

```ruby
# แนวปฏิบัติที่ปลอดภัย: ถ้าต้องรับชื่อ method จาก user input จริงๆ
# ให้ whitelist ชื่อ method ที่อนุญาตไว้ล่วงหน้าเสมอ (defense in depth)
ALLOWED_SORT_METHODS = %i[upcase downcase reverse].freeze

def safe_transform(text, method_name)
  method_sym = method_name.to_sym
  raise ArgumentError, "method ไม่ได้รับอนุญาต: #{method_sym}" unless ALLOWED_SORT_METHODS.include?(method_sym)

  text.public_send(method_sym)
end

puts safe_transform("Ruby", "upcase")   # => RUBY
safe_transform("Ruby", "wipe_all_data!")
# ArgumentError: method ไม่ได้รับอนุญาต: wipe_all_data!
```

---

## Step 143: `respond_to?` — ตรวจสอบก่อนว่า object มี method นี้จริงไหม

เมื่อเราจะเรียก method แบบไดนามิกด้วย `send`/`public_send` เราไม่สามารถแน่ใจได้ 100%
ว่า object นั้นมี method ที่ชื่อตรงกันจริง — ถ้าเรียก method ที่ไม่มีอยู่จะได้
`NoMethodError` ทันที **`respond_to?`** ช่วยให้เรา **ตรวจสอบก่อนเรียก** ได้อย่างปลอดภัย

```ruby
puts "hello".respond_to?(:upcase)     # => true
puts "hello".respond_to?(:fly)        # => false
puts 42.respond_to?(:+)                 # => true (operator ก็คือ method!)
puts nil.respond_to?(:upcase)           # => false
```

### ใช้ร่วมกับ `send`/`public_send` เพื่อป้องกัน error

```ruby
def safe_call(object, method_name, *args)
  if object.respond_to?(method_name)
    object.public_send(method_name, *args)
  else
    "object นี้ไม่มี method '#{method_name}'"
  end
end

puts safe_call("Ruby", :upcase)        # => RUBY
puts safe_call("Ruby", :fly_to_moon)   # => object นี้ไม่มี method 'fly_to_moon'
puts safe_call(42, :+, 8)               # => 50
```

**อธิบาย:** pattern `respond_to? -> send/public_send` เป็นรูปแบบที่พบบ่อยมากในโค้ด Ruby
ที่ต้องทำงานกับ object หลายชนิดที่ไม่รู้ interface แน่ชัดล่วงหน้า (polymorphism แบบ
duck typing ซึ่งจะเรียนลึกใน Part 016) เพราะช่วยป้องกัน `NoMethodError` ที่ทำให้โปรแกรม
ล่มกลางทาง

### ตรวจสอบ private method ด้วย `respond_to?`

โดย default `respond_to?` จะเช็คเฉพาะ public (และ protected) method เท่านั้น ถ้าต้องการ
เช็ครวม private method ด้วย ต้องส่ง argument ตัวที่สองเป็น `true`

```ruby
class OrderCalculator
  def total(price, quantity)
    price * quantity
  end

  private

  def calculate_tax(amount)
    amount * 0.07
  end
end

order = OrderCalculator.new

puts order.respond_to?(:calculate_tax)         # => false (default ไม่นับ private)
puts order.respond_to?(:calculate_tax, true)   # => true (นับ private ด้วยเมื่อระบุ true)
```

> **ข้อควรระวัง:** ถ้าใช้ `respond_to?(:method, true)` เพื่อเช็คว่ามี private method แล้ว
> ตามด้วย `send` (ไม่ใช่ `public_send`) นั่นหมายความว่าเรากำลัง**ตั้งใจ**เจาะ private
> boundary แล้ว — ควรทำเฉพาะกรณีที่จำเป็นจริงๆ (เช่น เขียน test, เขียน debugging tool)
> ไม่ใช่ในโค้ด business logic ทั่วไป

### ตัวอย่างจริง: เลือกวิธี serialize object ตามความสามารถของมัน

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end

  def to_a
    [@x, @y]
  end
end

def serialize(object)
  if object.respond_to?(:to_h)
    object.to_h
  elsif object.respond_to?(:to_a)
    object.to_a
  elsif object.respond_to?(:to_s)
    object.to_s
  else
    raise TypeError, "ไม่รู้วิธี serialize object ชนิด #{object.class}"
  end
end

p serialize({ name: "มานี", age: 25 })   # => {:name=>"มานี", :age=>25} (Hash มี to_h อยู่แล้ว)
p serialize(Point.new(1, 2))               # => [1, 2] (มีแค่ to_a ไม่มี to_h)
p serialize(42)                             # => "42" (มีแค่ to_s)
```

ตัวอย่างนี้แสดงแนวคิดสำคัญ: แทนที่จะเช็ค `object.is_a?(Hash)` หรือ `object.class == Hash`
(ซึ่งผูกติดกับ **ชนิด** ของ object) เราเช็คว่า object **"ทำอะไรได้บ้าง"** ผ่าน
`respond_to?` แทน — นี่คือหัวใจของ **duck typing** ("ถ้ามันร้องเหมือนเป็ด เดินเหมือนเป็ด
มันก็คือเป็ด") ที่จะเรียนอย่างละเอียดใน Part 016

---

## Step 144: `define_method` — นิยาม method แบบไดนามิกตอนรัน class

จนถึงตอนนี้เราสร้าง method ด้วย `def...end` เสมอ ซึ่งเป็นการนิยามแบบ "ตายตัว" (static)
ตอนเขียนโค้ด **`define_method`** ให้เราสร้าง method ได้แบบ **ไดนามิก** โดยรับชื่อ method
เป็น Symbol/String และรับ block เป็นเนื้อหาของ method นั้น

```ruby
# frozen_string_literal: true

class Greeter
  define_method(:hello) do
    "สวัสดี!"
  end
end

g = Greeter.new
puts g.hello   # => สวัสดี!
```

**อธิบาย:** `define_method(:hello) do ... end` มีผลเทียบเท่ากับการเขียน
`def hello; "สวัสดี!"; end` ตรงๆ ทุกประการ — ต่างกันแค่ **ชื่อ method (`:hello`) เป็นค่า
ที่กำหนดได้แบบไดนามิก** (มาจากตัวแปร, มาจากการวนลูป, มาจาก array ของชื่อ ฯลฯ) ซึ่ง `def`
ทำแบบนั้นไม่ได้เพราะชื่อ method หลัง `def` ต้องเขียนตายตัวในโค้ด

### `define_method` รับ parameter ได้เหมือน method ปกติทุกประการ

```ruby
class Calculator
  define_method(:add) do |a, b|
    a + b
  end

  define_method(:greet) do |name, greeting: "สวัสดี"|
    "#{greeting}, #{name}!"
  end
end

calc = Calculator.new
puts calc.add(3, 5)                        # => 8
puts calc.greet("มานี")                    # => สวัสดี, มานี!
puts calc.greet("สมชาย", greeting: "หวัดดี")  # => หวัดดี, สมชาย!
```

ทั้ง positional argument, default value, และ keyword argument ที่เรียนมาทั้งหมดใน Part 007
ใช้งานได้ปกติกับ `define_method` เพราะ block ที่ส่งเข้าไปทำหน้าที่เหมือน method body ทุก
ประการ (เบื้องหลัง `define_method` แปลง block ให้กลายเป็น method จริงๆ ผูกกับ class)

### จุดต่างสำคัญจาก `def`: `define_method` เป็น **method เรียกได้** ไม่ใช่ keyword

เพราะ `define_method` เป็น method ธรรมดา (ไม่ใช่ keyword แบบ `def`) เราจึงเรียกมันภายใน
loop หรือภายใน method อื่นได้อย่างอิสระ — นี่คือสิ่งที่ `def` ทำไม่ได้ และเป็นกุญแจสำคัญ
ของหัวข้อถัดไป (Step 145)

```ruby
class Config
  # define_method ถูกเรียกแบบไดนามิกจากค่าใน array ได้ (def ทำแบบนี้ไม่ได้)
  %i[host port timeout].each do |setting_name|
    define_method(setting_name) do
      "ค่า #{setting_name} ยังไม่ถูกตั้งค่า"
    end
  end
end

config = Config.new
puts config.host       # => ค่า host ยังไม่ถูกตั้งค่า
puts config.port       # => ค่า port ยังไม่ถูกตั้งค่า
puts config.timeout    # => ค่า timeout ยังไม่ถูกตั้งค่า
```

### เปรียบเทียบ `define_method` กับ `attr_accessor`

ที่จริง `attr_accessor`, `attr_reader`, `attr_writer` ที่เรียนมาแล้วใน Part 009 ก็คือ
metaprogramming รูปแบบหนึ่ง — เป็น method ที่เมื่อเรียกแล้ว **สร้าง method getter/setter
ให้อัตโนมัติ** เบื้องหลังการทำงานของมันใกล้เคียงกับการเขียน `define_method` เอง

```ruby
class PersonManual
  # เขียนเอง: attr_accessor เทียบเท่ากับการ define_method 2 ตัวแบบนี้
  define_method(:name) { @name }
  define_method(:name=) { |value| @name = value }
end

p1 = PersonManual.new
p1.name = "มานี"
puts p1.name   # => มานี

class PersonWithAccessor
  attr_accessor :name   # เขียนสั้นกว่ามาก และทำสิ่งเดียวกัน
end

p2 = PersonWithAccessor.new
p2.name = "สมชาย"
puts p2.name   # => สมชาย
```

**ข้อคิด:** `attr_accessor` คือตัวอย่างจริงของ metaprogramming ที่เราใช้มาตั้งแต่ Part 009
โดยไม่รู้ตัว — มันคือ method ที่เรียกครั้งเดียวแล้ว "สร้าง method อื่นให้อัตโนมัติ" นี่คือ
สิ่งที่เราจะเขียนเองในระดับที่ซับซ้อนขึ้นในหัวข้อถัดไป

---

## Step 145: ใช้ loop ร่วมกับ `define_method` เพื่อ generate method หลายตัวแบบ DRY

จุดแข็งที่สุดของ `define_method` คือใช้ร่วมกับ loop เพื่อสร้าง method **หลายตัวที่มี
รูปแบบคล้ายกัน** โดยไม่ต้องเขียน `def` ซ้ำๆ ด้วยมือ — ตรงตามหลักการ **DRY (Don't Repeat
Yourself)** ที่เน้นย้ำมาตลอดหลักสูตร

### ปัญหา: การเขียน method ที่คล้ายกันซ้ำๆ ด้วยมือ

```ruby
# แบบไม่ DRY: เขียน predicate method ทีละตัวด้วยมือ ทั้งที่ pattern เหมือนกันทุกตัว
class Order
  STATUSES = %w[pending paid shipped delivered cancelled].freeze

  def initialize(status)
    @status = status
  end

  def pending?
    @status == "pending"
  end

  def paid?
    @status == "paid"
  end

  def shipped?
    @status == "shipped"
  end

  def delivered?
    @status == "delivered"
  end

  def cancelled?
    @status == "cancelled"
  end
end
```

ปัญหาชัดเจน: ถ้าเพิ่ม status ใหม่ (เช่น `"refunded"`) ต้องมาเพิ่ม method ใหม่ซ้ำ pattern
เดิมด้วยมืออีกครั้ง เสี่ยงต่อการพิมพ์ผิดหรือลืม

### วิธีแก้ด้วย `define_method` + loop

```ruby
# frozen_string_literal: true

class Order
  STATUSES = %w[pending paid shipped delivered cancelled].freeze

  def initialize(status)
    raise ArgumentError, "status ไม่ถูกต้อง: #{status}" unless STATUSES.include?(status)

    @status = status
  end

  # สร้าง predicate method (pending?, paid?, shipped?, ...) ให้ทุก status โดยอัตโนมัติ
  STATUSES.each do |status_name|
    define_method("#{status_name}?") do
      @status == status_name
    end
  end
end

order1 = Order.new("paid")
puts order1.pending?     # => false
puts order1.paid?        # => true
puts order1.shipped?     # => false

order2 = Order.new("shipped")
puts order2.shipped?     # => true
puts order2.delivered?   # => false
```

**อธิบายทีละขั้น:**

1. `STATUSES.each do |status_name|` วนลูปผ่านค่าใน Array ตอนที่ **class ถูกโหลด**
   (class body รันจากบนลงล่างเหมือนโค้ดทั่วไป)
2. ในแต่ละรอบ เรียก `define_method("#{status_name}?")` เพื่อสร้าง method ใหม่ 1 ตัว
   ชื่อตามรูปแบบ `"pending?"`, `"paid?"` ฯลฯ
3. เนื้อหาของ method (ใน `do...end`) คือ `@status == status_name` — จุดที่ต้องเข้าใจ
   ให้ดีคือ **`status_name` ที่ใช้ในตัว method นี้ถูกจับเป็น closure** (concept ที่เรียน
   จาก Part 008 เรื่อง Block/Proc) หมายความว่า **แต่ละ method ที่สร้างขึ้นจะ "จำ" ค่า
   `status_name` ของรอบ loop ตอนที่มันถูกสร้างไว้ตลอดไป** แม้ loop จะจบไปแล้วก็ตาม
4. ถ้าต้องการเพิ่ม status ใหม่ แค่เพิ่มคำใน `STATUSES` array ตัวเดียว — method ใหม่จะถูก
   สร้างให้อัตโนมัติทันที ไม่ต้องแตะโค้ดส่วนอื่นเลย

```ruby
# พิสูจน์ว่าการเพิ่ม status ใหม่ทำให้ method ใหม่เกิดขึ้นทันที โดยไม่ต้องแก้อะไรเพิ่ม
class Order2
  STATUSES = %w[pending paid shipped delivered cancelled refunded].freeze  # เพิ่ม "refunded"

  def initialize(status)
    @status = status
  end

  STATUSES.each do |status_name|
    define_method("#{status_name}?") { @status == status_name }
  end
end

order = Order2.new("refunded")
puts order.refunded?   # => true (method นี้ไม่มีใครเขียนด้วยมือเลย!)
```

### ตัวอย่างที่ 2: generate getter/setter จากรายชื่อ column (จำลอง ActiveRecord)

นี่คือตัวอย่างที่ **ใกล้เคียงกับสิ่งที่ ActiveRecord ทำจริงมาก** เมื่อสร้าง attribute
method จากชื่อ column ในฐานข้อมูล (จะเรียนเต็มรูปแบบใน Part 025)

```ruby
class SimpleModel
  def self.attribute(name)
    # generate getter
    define_method(name) do
      instance_variable_get("@#{name}")
    end

    # generate setter
    define_method("#{name}=") do |value|
      instance_variable_set("@#{name}", value)
    end
  end

  attribute :title
  attribute :price
end

item = SimpleModel.new
item.title = "หนังสือ Ruby"
item.price = 350

puts item.title   # => หนังสือ Ruby
puts item.price   # => 350
```

**อธิบาย:** `self.attribute(name)` เป็น **class method** (เรียนจาก Part 009) ที่เมื่อถูก
เรียกแล้วจะไปสร้าง instance method 2 ตัว (`name` และ `name=`) ให้อัตโนมัติ pattern นี้คือ
รากฐานที่แท้จริงของ `attr_accessor` เอง — และเป็นแนวทางเดียวกับที่ ActiveRecord ใช้สร้าง
method จากชื่อ column ในตาราง (เพียงแต่ ActiveRecord อ่านชื่อ column จากฐานข้อมูลแทนที่จะ
รับเป็น argument ตรงๆ) เรื่อง `instance_variable_get`/`instance_variable_set` ที่ใช้ในนี้
จะอธิบายละเอียดใน Step 148

> **ข้อสังเกตสำคัญ:** ในตัวอย่างนี้เราเขียน `self.attribute` เอง แต่ Ruby ก็มี
> `attr_accessor` ให้ใช้อยู่แล้วซึ่งทำสิ่งเดียวกันและปลอดภัยกว่า — จุดประสงค์ของตัวอย่างนี้
> ไม่ใช่ให้ไปแทนที่ `attr_accessor` แต่เพื่อ **แสดงให้เห็นกลไกเบื้องหลัง** ว่า class-level
> DSL แบบที่ Rails ใช้ (`has_many`, `validates`, `belongs_to`) ทำงานด้วยหลักการเดียวกันนี้
> ทั้งหมด

---

## Step 146: `method_missing` + `respond_to_missing?` — คู่หูที่ต้องมาด้วยกันเสมอ

ใน Part 007 (Step 69) เราได้เกริ่นนำ `method_missing` ไปแล้วสั้นๆ ทบทวนก่อนว่ามันคืออะไร
แล้วค่อยขยายให้ลึกและถูกต้องยิ่งขึ้น

### ทบทวน: `method_missing` คืออะไร

ปกติเมื่อเรียก method ที่ไม่มีอยู่จริงบน object Ruby จะโยน `NoMethodError` ทันที แต่ก่อน
จะทำแบบนั้น Ruby จะ **ลองเรียก method พิเศษชื่อ `method_missing` ก่อนเสมอ** ถ้าเรา
override `method_missing` เอง เราก็ดักจับพฤติกรรม "เรียก method ที่ไม่มีอยู่" แล้วกำหนด
เองได้ว่าจะทำอะไร

```ruby
# frozen_string_literal: true

class GhostResponder
  def method_missing(method_name, *args, **kwargs)
    "คุณเรียก method ชื่อ '#{method_name}' ที่ไม่มีอยู่จริง พร้อม args: #{args}"
  end
end

ghost = GhostResponder.new
puts ghost.fly_to_the_moon(3, 2, 1)
# => คุณเรียก method ชื่อ 'fly_to_the_moon' ที่ไม่มีอยู่จริง พร้อม args: [3, 2, 1]
```

**อธิบาย parameter ของ `method_missing`:**

- `method_name` — Symbol ของชื่อ method ที่ถูกเรียก (แต่ไม่มีอยู่จริง)
- `*args` — positional argument ทั้งหมดที่ส่งเข้ามาตอนเรียก
- `**kwargs` — keyword argument ทั้งหมด (ถ้ามี)
- (ถ้ามี block ส่งเข้ามาด้วย ต้องรับด้วย `&block` เพิ่มอีกตัว)

### ปัญหาสำคัญถ้าลืมทำสิ่งนี้: ไม่ override `respond_to_missing?`

นี่คือจุดที่มือใหม่พลาดบ่อยที่สุด และเป็นสาเหตุของบัคที่ตรวจจับยาก ลองดูปัญหาที่เกิดขึ้น
ถ้า override แค่ `method_missing` เพียงอย่างเดียว:

```ruby
class GhostResponderBad
  def method_missing(method_name, *args)
    "ได้รับการเรียก #{method_name}"
  end
end

ghost = GhostResponderBad.new
puts ghost.anything_at_all   # => ได้รับการเรียก anything_at_all (ทำงานได้ปกติ)

# แต่ปัญหาคือ...
puts ghost.respond_to?(:anything_at_all)   # => false !! (ทั้งที่จริงๆ เรียกได้)
```

**ทำไมถึงเป็นปัญหา:** `respond_to?` **ไม่รู้เรื่อง `method_missing` เลย** โดย default
มันแค่เช็คว่า method นั้นถูก `def` ไว้จริงๆ ในตัว class หรือเปล่า ทำให้เกิดความขัดแย้งที่
อันตราย: **object บอกว่า "ฉันไม่มี method นี้" (`respond_to?` คืน `false`) แต่พอเรียกจริง
กลับทำงานได้ (`method_missing` รับไว้ให้)** — โค้ดอื่นที่ใช้ pattern
`respond_to? -> send` (ที่เรียนใน Step 143) จะพังทันทีเมื่อเจอ object แบบนี้ เพราะมัน
เชื่อ `respond_to?` แล้วเลือกที่จะไม่เรียก method นั้น

```ruby
class GhostResponderBad
  def method_missing(method_name, *args)
    "ได้รับการเรียก #{method_name}"
  end
end

ghost = GhostResponderBad.new

def try_greet(obj)
  if obj.respond_to?(:greet)
    obj.greet
  else
    "object นี้ทักทายไม่เป็น"
  end
end

puts try_greet(ghost)
# => object นี้ทักทายไม่เป็น
# ทั้งที่จริงๆ ghost.greet เรียกได้ผ่าน method_missing! นี่คือบัคที่เกิดจากการลืม
# respond_to_missing?
```

### ทางแก้: ต้อง override `respond_to_missing?` คู่กันเสมอ

```ruby
class GhostResponderGood
  def method_missing(method_name, *args)
    "ได้รับการเรียก #{method_name}"
  end

  def respond_to_missing?(method_name, include_private = false)
    true   # บอกความจริงว่า object นี้ "รับมือ" ได้ทุก method ผ่าน method_missing
  end
end

def try_greet(obj)
  if obj.respond_to?(:greet)
    obj.greet
  else
    "object นี้ทักทายไม่เป็น"
  end
end

ghost = GhostResponderGood.new
puts ghost.anything_at_all              # => ได้รับการเรียก anything_at_all
puts ghost.respond_to?(:anything_at_all)   # => true (ตอนนี้สอดคล้องกันแล้ว)
puts try_greet(ghost)                     # => ได้รับการเรียก greet (ทำงานถูกต้อง)
```

**อธิบาย `respond_to_missing?`:**

- เป็น method ที่ Ruby เรียกให้อัตโนมัติเมื่อ `respond_to?` เช็คแล้วไม่เจอ method ที่ถูก
  `def` ไว้ตรงๆ — เป็น "ทางออกสำรอง" ให้เราบอก Ruby เองว่า "จริงๆ ฉันตอบสนอง method นี้
  ได้นะ (ผ่าน method_missing)"
- parameter ตัวแรก `method_name` คือชื่อ method ที่กำลังถูกเช็ค
- parameter ตัวที่สอง `include_private` (default `false`) บอกว่ากำลังเช็ครวม private
  method ด้วยหรือไม่ (สอดคล้องกับ `respond_to?(:name, true)` ที่เรียนใน Step 143)
- ต้อง `def` เป็น (ไม่ใช่ `private` method) และควรคืนค่า `true`/`false` เท่านั้น
  (เป็น predicate method ตามธรรมเนียมที่เรียนใน Part 007 Step 68)

> **กฎเหล็กของ metaprogramming: ทุกครั้งที่ override `method_missing` ต้อง override
> `respond_to_missing?` คู่กันเสมอ ไม่มีข้อยกเว้น** ถ้าไม่ทำคู่กัน object จะ "โกหก"
> เกี่ยวกับความสามารถของตัวเอง ทำให้โค้ดอื่นที่พึ่งพา `respond_to?` (รวมถึง Ruby internals
> บางส่วน เช่น `method()`, `Marshal`, และหลาย gem ที่เช็ค `respond_to?` ก่อนเรียก) ทำงาน
> ผิดพลาดโดยไม่มี error ให้เห็นชัดเจน — เป็นบัคประเภทที่ debug ยากที่สุดประเภทหนึ่ง

### เขียน `method_missing` อย่างมีความรับผิดชอบ: อย่าลืมเรียก `super`

อีกจุดที่สำคัญมากคือ **`method_missing` ควรจัดการเฉพาะกรณีที่เรารู้จักเท่านั้น** ส่วนกรณี
อื่นที่ไม่รู้จัก ต้องส่งต่อให้ Ruby จัดการแบบปกติ (คือโยน `NoMethodError`) ด้วยการเรียก
`super` มิเช่นนั้นโปรแกรมจะ "กลืน" ทุกการเรียก method แม้แต่ method ที่พิมพ์ผิดจริงๆ
ทำให้ debug ยากมาก (error ที่ควรเกิดกลับไม่เกิด)

```ruby
class SafeGhostResponder
  def method_missing(method_name, *args, **kwargs, &block)
    if method_name.to_s.start_with?("echo_")
      "เสียงสะท้อน: #{method_name.to_s.delete_prefix('echo_')}"
    else
      super   # ส่งต่อให้ Ruby จัดการปกติ -> โยน NoMethodError ถ้าไม่รู้จักจริงๆ
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_name.to_s.start_with?("echo_") || super
  end
end

responder = SafeGhostResponder.new
puts responder.echo_hello        # => เสียงสะท้อน: hello
puts responder.respond_to?(:echo_hello)   # => true

responder.totally_unknown_method
# NoMethodError: undefined method 'totally_unknown_method' for #<SafeGhostResponder>
# (error แบบปกติ เพราะเราเรียก super ส่งต่อให้ Ruby จัดการ)

puts responder.respond_to?(:totally_unknown_method)   # => false (ถูกต้อง เพราะเรียก super ด้วย)
```

**อธิบาย:** ทั้ง `method_missing` และ `respond_to_missing?` เรียก `super` เมื่อไม่เข้า
เงื่อนไขที่เรากำหนด — `super` ในที่นี้จะไปเรียก `method_missing`/`respond_to_missing?`
ดั้งเดิมของ `Object` (บรรพบุรุษของทุก class ใน Ruby) ซึ่งมีพฤติกรรมมาตรฐานคือโยน
`NoMethodError` (สำหรับ `method_missing`) หรือคืน `false` (สำหรับ `respond_to_missing?`)

---

## Step 147: ตัวอย่างจริง — สร้าง dynamic attribute object คล้าย ActiveRecord/OpenStruct

ใน Part 014 เราเรียนเรื่อง `OpenStruct` ที่ให้เราสร้าง object ที่มี attribute แบบไดนามิก
โดยไม่ต้องนิยาม class ล่วงหน้า ตอนนี้เราจะ **เขียนกลไกที่คล้ายกันเองตั้งแต่ต้น** โดยใช้
`method_missing` เพื่อให้เข้าใจว่าเบื้องหลัง `OpenStruct` (และบางส่วนของ ActiveRecord
attribute system) ทำงานอย่างไร

```ruby
# frozen_string_literal: true

class DynamicAttributes
  def initialize(attributes = {})
    @attributes = attributes.transform_keys(&:to_sym)
  end

  def method_missing(method_name, *args)
    method_str = method_name.to_s

    if method_str.end_with?("=")
      # เรียกในรูปแบบ setter เช่น obj.name = "value"
      key = method_str.delete_suffix("=").to_sym
      @attributes[key] = args.first
    elsif @attributes.key?(method_name)
      # เรียกในรูปแบบ getter เช่น obj.name และมี key นี้อยู่จริง
      @attributes[method_name]
    else
      super
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_str = method_name.to_s
    method_str.end_with?("=") || @attributes.key?(method_name) || super
  end

  def to_h
    @attributes.dup
  end
end

person = DynamicAttributes.new(name: "มานี", age: 25)

puts person.name    # => มานี
puts person.age      # => 25

person.email = "manee@example.com"   # เพิ่ม attribute ใหม่แบบไดนามิกได้เลย
puts person.email    # => manee@example.com

p person.to_h   # => {:name=>"มานี", :age=>25, :email=>"manee@example.com"}

puts person.respond_to?(:name)     # => true
puts person.respond_to?(:phone)    # => false (ยังไม่เคยตั้งค่านี้)

person.phone   # NoMethodError: undefined method 'phone' for #<DynamicAttributes>
# ถูกต้อง เพราะ "phone" ไม่ใช่ setter (ไม่ลงท้าย =) และยังไม่มี key นี้ใน @attributes เลย
```

**อธิบายทีละส่วน:**

1. `initialize` เก็บ Hash ของ attribute ทั้งหมดไว้ใน `@attributes` โดยแปลง key ให้เป็น
   Symbol เสมอ (`transform_keys(&:to_sym)` เรียนจาก Part 013 Enumerable ขั้นสูง) เพื่อให้
   `person.name` (ที่ Ruby ส่ง `method_name` มาเป็น Symbol `:name` เสมอ) จับคู่กับ key ใน
   Hash ได้ถูกต้อง ไม่ว่าตอนสร้าง object จะส่ง key มาเป็น String หรือ Symbol ก็ตาม
2. เมื่อเรียก method ที่ไม่มีอยู่จริง (เช่น `person.name`) Ruby เรียก `method_missing`
   พร้อมส่ง `method_name = :name`
3. เช็คก่อนว่าชื่อ method ลงท้ายด้วย `=` หรือไม่ (เป็นการเรียกแบบ setter เช่น
   `person.email = "..."`) ถ้าใช่ ให้ตัด `=` ออกแล้วบันทึกค่าลง `@attributes`
4. ถ้าไม่ใช่ setter และมี key นั้นอยู่ใน `@attributes` แล้ว ให้คืนค่านั้นออกไป (getter)
5. ถ้าไม่เข้าเงื่อนไขไหนเลย (ทั้งไม่ใช่ setter และไม่มี key อยู่จริง) ให้เรียก `super`
   เพื่อโยน `NoMethodError` ตามปกติ — ป้องกันไม่ให้ object "โกหก" ว่ามี attribute ที่
   ไม่เคยตั้งค่าไว้เลย
6. `respond_to_missing?` ตรวจสอบเงื่อนไขแบบเดียวกันกับ `method_missing` (ต้องสอดคล้องกัน
   เสมอตามกฎที่เรียนใน Step 146) — ถ้า `method_missing` จะรับมือ (handle) การเรียกนี้ได้
   `respond_to_missing?` ต้องคืน `true` เช่นกัน

### ทำไมสิ่งนี้ใกล้เคียงกับ ActiveRecord

เมื่อเราเขียน `Post.find(1).title` ActiveRecord ก็ทำงานตามหลักการคล้ายกันมาก (ในเวอร์ชัน
เก่าใช้ `method_missing` ล้วนๆ ส่วนเวอร์ชันปัจจุบันใช้ `define_method` ผสมเพื่อ performance
ที่ดีกว่า แต่แนวคิดหลักเหมือนกัน): เก็บค่า column ทั้งหมดไว้ใน Hash ภายใน (`@attributes`)
แล้วดักจับการเรียก method ที่ตรงกับชื่อ column เพื่อคืนค่าหรือตั้งค่าให้ — สิ่งที่เราเขียน
ในตัวอย่างข้างบนคือ "แบบจำลองขนาดเล็ก" ของกลไกนี้เป๊ะๆ

> **ทำไมใน Step 145 เราใช้ `define_method` ส่วน Step 147 นี้ใช้ `method_missing`:**
> `define_method` เหมาะกับกรณีที่ **รู้ล่วงหน้าแล้วว่ามี attribute อะไรบ้าง** (เช่น
> รู้จาก schema ของฐานข้อมูลตอน class ถูกโหลด) ส่วน `method_missing` เหมาะกับกรณีที่
> **ไม่รู้ล่วงหน้า** ว่าจะมี attribute อะไรบ้าง (เช่น รับ Hash ที่มี key อะไรก็ได้ตอน
> `initialize` เหมือนตัวอย่างนี้) — ActiveRecord จริงๆ ใช้ผสมกันทั้งสองแบบ: ใช้
> `method_missing` เป็น fallback แรกสุด แล้วค่อย `define_method` แบบ "แคช" method นั้นไว้
> จริงๆ ในการเรียกครั้งถัดไปเพื่อความเร็ว (เทคนิคที่เรียกว่า "method generation" ซึ่งเป็น
> หัวข้อขั้นสูงเกินขอบเขตของ Part นี้)

---

## Step 148: `instance_variable_get`/`instance_variable_set` — เข้าถึง instance variable แบบไดนามิก

ปกติเราเข้าถึง instance variable ผ่านชื่อที่เขียนตรงๆ ในโค้ด (`@name`) แต่บางครั้ง
เราต้องการเข้าถึง instance variable โดยที่ **ชื่อของมันเป็นค่าที่กำหนดตอนรัน** (มาจาก
ตัวแปร, มาจาก string) — `instance_variable_get` และ `instance_variable_set` ทำหน้าที่นี้

```ruby
class Product
  def initialize(name, price)
    @name = name
    @price = price
  end
end

item = Product.new("หนังสือ", 250)

puts item.instance_variable_get(:@name)    # => หนังสือ
puts item.instance_variable_get("@price")  # => 250  (รับทั้ง Symbol และ String)

item.instance_variable_set(:@price, 300)
puts item.instance_variable_get(:@price)   # => 300
```

**ข้อสังเกตสำคัญ:** ชื่อที่ส่งให้ `instance_variable_get`/`instance_variable_set` **ต้องมี
เครื่องหมาย `@` นำหน้าเสมอ** (ต่างจาก `send` ที่ไม่ต้องมี `@` เพราะชื่อ method ไม่มี `@`)
ถ้าลืมใส่ `@` จะเกิด error:

```ruby
item.instance_variable_get(:name)
# NameError: `name' is not allowed as an instance variable name
```

### จุดสำคัญ: เข้าถึงได้แม้ว่า instance variable นั้นจะไม่มี `attr_accessor` เลย

นี่คือสิ่งที่ทำให้ `instance_variable_get`/`set` ทรงพลัง (และอันตราย) เหมือนกับที่ `send`
เจาะทะลุ private method ได้ `instance_variable_get`/`set` ก็ **เจาะทะลุการห่อหุ้ม
(encapsulation) ของ object ได้เต็มที่** โดยไม่สนใจว่ามี getter/setter นิยามไว้หรือไม่

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance   # ไม่มี attr_accessor หรือ getter ใดๆ เลย ตั้งใจซ่อนไว้
  end
end

account = BankAccount.new(1000)

account.balance
# NoMethodError: undefined method 'balance' for #<BankAccount>

# แต่เจาะเข้าไปอ่าน/แก้ค่าได้ตรงๆ ผ่าน instance_variable_get/set แม้ไม่มี method ให้เรียกเลย
puts account.instance_variable_get(:@balance)   # => 1000
account.instance_variable_set(:@balance, 999_999)
puts account.instance_variable_get(:@balance)   # => 999999
```

> **ข้อควรระวัง:** เช่นเดียวกับ `send`, การใช้ `instance_variable_get`/`set` ในโค้ด
> business logic ทั่วไปถือเป็น **anti-pattern** เพราะทำลายหลักการ encapsulation ที่เรียน
> มาตั้งแต่ Part 007–009 โดยสิ้นเชิง ควรใช้เฉพาะในบริบทของ **การเขียน library/framework
> ระดับ infrastructure** (เช่น การเขียน serializer, ORM, test helper, debugging tool)
> ที่จำเป็นต้อง "มองทะลุ" object ทุกชนิดโดยไม่รู้โครงสร้างล่วงหน้า — ไม่ควรใช้แทนการเรียก
> public method ปกติในโค้ดแอปพลิเคชันทั่วไป

### ตัวอย่างการใช้งานที่เหมาะสม: เขียน generic inspector/serializer

```ruby
class Product
  def initialize(name, price)
    @name = name
    @price = price
  end
end

def inspect_all_ivars(object)
  object.instance_variables.each_with_object({}) do |ivar_name, result|
    # ivar_name ได้มาจาก instance_variables ซึ่งคืน array ของ Symbol เช่น [:@name, :@price]
    key = ivar_name.to_s.delete_prefix("@").to_sym
    result[key] = object.instance_variable_get(ivar_name)
  end
end

item = Product.new("หนังสือ", 250)
p inspect_all_ivars(item)   # => {:name=>"หนังสือ", :price=>250}
```

**อธิบาย:** `object.instance_variables` (ไม่มี `_get`) คืน Array ของ **ชื่อ** instance
variable ทั้งหมดที่ object มีอยู่ (เป็น Symbol พร้อม `@` นำหน้า) — ผสมกับ
`instance_variable_get` ทำให้เราเขียน method ที่ "สำรวจ" object ใดๆ ก็ได้โดยไม่ต้องรู้
ล่วงหน้าว่ามันมี instance variable ชื่ออะไรบ้าง นี่คือหลักการเดียวกับที่ gem อย่าง
`awesome_print`, `pry`, หรือ serializer หลายตัวใช้ในการแสดงผล/แปลง object เป็น Hash/JSON
แบบอัตโนมัติ

---

## Step 149: `class_eval` และ `instance_eval` เบื้องต้น

`class_eval` และ `instance_eval` เป็นเทคนิคขั้นสูงกว่าที่ให้เรา **"เปลี่ยนบริบท" ของโค้ด**
ชั่วคราว เพื่อรันโค้ดราวกับว่าเรากำลังเขียนอยู่ **ข้างในนิยามของ class นั้น** (สำหรับ
`class_eval`) หรือ **ข้างในบริบทของ object ตัวใดตัวหนึ่งโดยตรง** (สำหรับ `instance_eval`)
ทั้งสองคือรากฐานสำคัญของการเขียน **DSL (Domain Specific Language)** ที่ Rails ใช้ทั่ว
ทั้ง framework เช่น `validates`, `has_many`, `namespace`/`resources` ใน routes

### `class_eval` — เปิด class ที่มีอยู่แล้ว แล้วเพิ่ม method เข้าไปแบบไดนามิก

```ruby
class Greeter
  def hello
    "สวัสดี!"
  end
end

# เปิด class Greeter อีกครั้ง (ผ่านตัวแปรที่เก็บ reference ของ class ไว้) แล้วเพิ่ม method
Greeter.class_eval do
  def goodbye
    "ลาก่อน!"
  end
end

g = Greeter.new
puts g.hello       # => สวัสดี!
puts g.goodbye     # => ลาก่อน! (method ที่เพิ่มเข้ามาทีหลังผ่าน class_eval)
```

**อธิบาย:** `class_eval` ทำสิ่งที่คล้ายกับการเปิด class ซ้ำแบบปกติ (`class Greeter ... end`
ที่เรียนจาก Part 010 — open classes) แต่ต่างกันตรงที่ **`class_eval` รับตัวแปรที่เก็บ
reference ของ class ไว้ได้** ทำให้เปิด class ที่ **ชื่อเป็นค่าที่กำหนดตอนรัน** (เช่น มาจาก
`const_get`, จาก loop ของหลาย class) ได้ ซึ่งการเขียน `class Greeter` ตรงๆ ทำไม่ได้ถ้าชื่อ
class ไม่ตายตัว

```ruby
# ตัวอย่าง: เพิ่ม method เดียวกันให้กับหลาย class พร้อมกันด้วย loop (def ตรงๆ ทำแบบนี้ไม่ได้)
[String, Symbol].each do |klass|
  klass.class_eval do
    def shout
      "#{self.to_s.upcase}!!!"
    end
  end
end

puts "hello".shout   # => HELLO!!!
puts :ruby.shout     # => RUBY!!!
```

> **คำเตือนสำคัญ:** ตัวอย่างข้างบนคือ **monkey patching** — การแก้ไข class ของ Ruby เอง
> (`String`) จากภายนอก ซึ่งเป็นเทคนิคที่ **อันตรายมากในโค้ด production จริง** เพราะอาจไป
> ชนกับ method ชื่อเดียวกันจาก gem อื่น หรือทำให้พฤติกรรมของ `String` เปลี่ยนไปทั่วทั้ง
> แอปพลิเคชันโดยไม่มีใครคาดคิด ตัวอย่างนี้มีไว้เพื่อ **สาธิตกลไกของ `class_eval`** เท่านั้น
> ในทางปฏิบัติควรหลีกเลี่ยง monkey patching core class เว้นแต่จำเป็นจริงๆ และมักมีทางเลือก
> ที่ปลอดภัยกว่าเสมอ เช่นการเขียน Module มา `include`/`extend` แทน (เรียนจาก Part 010)

### `class_eval` ใช้ทำอะไรได้อีก: อ่าน string เป็นโค้ด และรับ arguments แบบไดนามิก

```ruby
class DynamicMethodBuilder
end

# สร้าง method จากชื่อที่กำหนดแบบไดนามิก (คล้ายกับที่เราเขียนด้วย define_method ใน Step 144
# แต่ class_eval ให้ยืดหยุ่นกว่าเพราะรันโค้ด Ruby จริงๆ ในบริบทของ class นั้น)
%w[red green blue].each do |color|
  DynamicMethodBuilder.class_eval do
    define_method("#{color}?") { false }
  end
end

builder = DynamicMethodBuilder.new
puts builder.red?     # => false
puts builder.green?   # => false
```

ในทางปฏิบัติ ตัวอย่างนี้เขียนด้วย `define_method` ตรงๆ (แบบ Step 145) ได้อยู่แล้วโดยไม่ต้อง
พึ่ง `class_eval` เลย — สิ่งที่ `class_eval` เพิ่มมาคือ **ความสามารถเปิด class ที่ไม่ได้
อยู่ใน scope ปัจจุบัน** (เช่น class ที่เก็บไว้ในตัวแปร หรือดึงมาจาก `Object.const_get`)
ซึ่งมีประโยชน์มากตอนเขียน gem/library ที่ต้องแก้ไข class ของผู้ใช้จากภายนอก

### `instance_eval` — รันโค้ดในบริบทของ **object ตัวเดียว** ไม่ใช่ทั้ง class

`instance_eval` คล้ายกับ `class_eval` แต่ต่างกันตรงที่ผลลัพธ์ของมันมีผลกับ **object
ตัวเดียว** ที่เรียกเท่านั้น ไม่กระทบ object อื่นของ class เดียวกัน

```ruby
class Product
  def initialize(name, price)
    @name = name
    @price = price
  end
end

item1 = Product.new("หนังสือ", 250)
item2 = Product.new("ปากกา", 15)

# เพิ่ม method ให้เฉพาะ item1 ตัวเดียว (เรียกว่า "singleton method")
item1.instance_eval do
  def special_discount
    "#{@name} ลดพิเศษเหลือ #{@price * 0.5}"
  end
end

puts item1.special_discount   # => หนังสือ ลดพิเศษเหลือ 125.0

item2.special_discount
# NoMethodError: undefined method 'special_discount' for #<Product> (item2 ไม่ได้รับผลกระทบ)
```

**อธิบาย:** ภายใน block ของ `instance_eval`, `self` จะเปลี่ยนไปชี้เป็น `item1` ชั่วคราว
ทำให้ `@name`, `@price` ภายใน block เข้าถึง instance variable ของ `item1` ได้โดยตรง (แม้
`@name`/`@price` จะเป็น instance variable ที่ "ส่วนตัว" ของแต่ละ object ก็ตาม) และ
`def special_discount` ที่เขียนในนี้จะกลายเป็น method ที่ผูกกับ `item1` ตัวเดียวเท่านั้น
(เทคนิคนี้เรียกว่า **singleton method** — เรื่องที่จะเจอลึกขึ้นตอนเรียน Ruby internals
ใน Part 087)

### เปรียบเทียบสั้นๆ ระหว่าง `class_eval` กับ `instance_eval`

| | `class_eval` | `instance_eval` |
|---|---|---|
| เรียกกับอะไร | Class (เช่น `Greeter.class_eval`) | Object ใดๆ (เช่น `item1.instance_eval`) |
| `def` ข้างในสร้างอะไร | instance method ของทุก object ในอนาคตของ class นั้น | singleton method เฉพาะ object นั้นตัวเดียว |
| ใช้ทำอะไรบ่อย | เพิ่ม method ให้ class ทั้งหมดแบบไดนามิก, เขียน DSL ระดับ class (เช่น `validates`) | ปรับพฤติกรรมของ object เฉพาะตัว, เขียน DSL ที่ต้องอ้างอิง instance variable ของ object นั้น (เช่น RSpec's `describe`/`it` block) |

> **หมายเหตุ:** `class_eval` และ `instance_eval` เป็นเทคนิคที่ใช้บ่อยตอนเขียน **gem หรือ
> library** มากกว่าโค้ดแอปพลิเคชันทั่วไป ในหลักสูตรนี้พอเข้าใจกลไกและอ่านโค้ดที่ใช้เทคนิค
> นี้ได้ก็เพียงพอสำหรับตอนนี้ — จะกลับมาเจอเทคนิคนี้อีกครั้งอย่างจริงจังตอนเขียน DSL ของ
> ตัวเองใน Part 089 (การสร้าง gem)

---

## Step 150: แบบฝึกหัด — สร้าง `DynamicRecord` และข้อควรระวังในการใช้ metaprogramming

### ก่อนลงมือ: ต้นทุนของ metaprogramming ที่ต้องรู้จักไว้เสมอ

Metaprogramming เป็นเครื่องมือที่ทรงพลังมาก แต่ในโค้ด production จริง ทีมมืออาชีพจะใช้
อย่างระมัดระวังมาก เพราะมีต้นทุนที่ชัดเจนหลายด้าน:

1. **Debug ยากขึ้นมาก** — เมื่อเกิด error ตรง method ที่ถูกสร้างแบบไดนามิก stack trace
   มักจะชี้ไปที่บรรทัดของ `define_method`/`method_missing` แทนที่จะชี้ไปที่ตำแหน่งจริงที่
   method ควรจะถูกนิยาม ทำให้ต้องเสียเวลาสืบสาวว่า method นี้ "มาจากไหนกันแน่"

2. **IDE และเครื่องมือ static analysis ตามไม่ทัน** — Autocomplete, go-to-definition,
   type checker (เช่น Sorbet, RBS) ส่วนใหญ่วิเคราะห์โค้ดแบบ static (ไม่ได้รันจริง) จึง
   "มองไม่เห็น" method ที่ถูกสร้างขึ้นตอนรันโปรแกรมจาก `define_method`/`method_missing`
   ผลคือ VS Code หรือเครื่องมือ RuboCop ที่ติดตั้งไว้ตั้งแต่ Part 001 อาจเตือนผิดๆ ว่า
   method ไม่มีอยู่ ทั้งที่จริงมันมีตอนรันจริง

3. **Performance cost** — `method_missing` ช้ากว่าการเรียก method ที่ `def` ไว้ตรงๆ
   เสมอ (ต้องผ่านกลไกตรวจสอบเพิ่มเติมก่อนถึงจะรู้ว่าต้องไปเข้า `method_missing`)
   ในโค้ดที่ต้องเรียกซ้ำจำนวนมาก (hot path) ผลต่างด้าน performance นี้สะสมได้มาก
   (นี่คือเหตุผลที่ ActiveRecord รุ่นใหม่หันไปใช้ `define_method` แบบ "แคช" มากกว่าพึ่ง
   `method_missing` ล้วนๆ อย่างที่กล่าวไว้ท้าย Step 147)

4. **โค้ดอ่านยากสำหรับคนใหม่ในทีม** — โค้ดที่ "สร้างโค้ดขึ้นมาเอง" ต้องใช้ความเข้าใจ
   Ruby ระดับลึกกว่าปกติ เพื่อนร่วมทีมที่เพิ่งเข้ามาอาจงงว่า method ที่เรียกอยู่ "มาจาก
   ไหน" ถ้าไม่มี comment อธิบายกำกับไว้ชัดเจน

> **แนวปฏิบัติที่ทีมมืออาชีพยึดถือ: ใช้ metaprogramming อย่างประหยัด (sparingly) และ
> เขียน comment อธิบายกำกับไว้เสมอ** ก่อนจะใช้ `method_missing`/`define_method`/
> `class_eval` ให้ถามตัวเองก่อนว่า "เขียน method ตรงๆ ด้วย `def` ธรรมดาได้ไหม" ถ้าได้
> ให้เลือกวิธีธรรมดาเสมอ ใช้ metaprogramming เฉพาะเมื่อ**ความซ้ำซ้อนของโค้ดมากจนคุ้มค่า
> กับต้นทุนที่ต้องแลก** จริงๆ (เช่น กรณีที่เรียนใน Step 145 ที่ต้อง generate method
   จำนวนมากจาก data ที่มีอยู่แล้ว) และเมื่อใช้แล้วต้องมี test ครอบคลุมพฤติกรรมของ method
   ที่สร้างขึ้นมาอย่างแน่นหนาเสมอ (จะเรียนเรื่อง test อย่างจริงจังใน Part 018–019)

### โจทย์

สร้างไฟล์ `dynamic_record.rb` ที่มี class ชื่อ `DynamicRecord` ซึ่งจำลองการทำงานส่วนเล็กๆ
ของ ActiveRecord attribute system โดยมีคุณสมบัติดังนี้:

1. `initialize(attributes = {})` รับ Hash ของ attribute เริ่มต้น
2. เข้าถึง attribute ผ่าน getter/setter แบบไดนามิกด้วย `method_missing` +
   `respond_to_missing?` (เหมือนที่เรียนใน Step 147) — เช่น `record.name`,
   `record.name = "ค่าใหม่"`
3. มี predicate method `has_attribute?(key)` เพื่อเช็คว่ามี attribute นี้อยู่จริงไหม
   (ไม่ผ่าน `method_missing`, เป็น method ปกติที่ `def` ตรงๆ)
4. มี method `attributes` ที่คืนค่า Hash ของ attribute ทั้งหมด (คืนค่าเป็น copy ไม่ใช่
   reference ตรงเพื่อป้องกันการแก้ไขจากภายนอกโดยไม่ผ่าน setter)
5. มี method `save_log` ที่คืน String สรุปว่า attribute ไหนถูกแก้ไขไปแล้วบ้าง (บันทึกด้วย
   `@changed_attributes` เป็น Array ของชื่อ attribute ที่เคยถูก `=` ตั้งค่าใหม่ หลังจาก
   `initialize`)
6. ถ้าเรียก getter ของ attribute ที่ไม่เคยมีอยู่เลย (ไม่เคยตั้งค่าตอน `initialize` และไม่
   เคยถูก set ทีหลัง) ต้องโยน `NoMethodError` ตามปกติ (ผ่าน `super`) ไม่ใช่คืน `nil` เงียบๆ

### เฉลย

```ruby
# frozen_string_literal: true

# dynamic_record.rb
# จำลองการทำงานส่วนเล็กๆ ของ ActiveRecord attribute system ด้วย method_missing

class DynamicRecord
  def initialize(attributes = {})
    @attributes = attributes.transform_keys(&:to_sym)
    @changed_attributes = []
  end

  # --- method ปกติที่ def ตรงๆ (ไม่ผ่าน method_missing) ---

  def has_attribute?(key)
    @attributes.key?(key.to_sym)
  end

  def attributes
    @attributes.dup
  end

  def save_log
    if @changed_attributes.empty?
      "ยังไม่มีการแก้ไข attribute ใดเลย"
    else
      "attribute ที่ถูกแก้ไข: #{@changed_attributes.uniq.join(', ')}"
    end
  end

  # --- ส่วน metaprogramming: getter/setter แบบไดนามิก ---

  def method_missing(method_name, *args)
    method_str = method_name.to_s

    if method_str.end_with?("=")
      key = method_str.delete_suffix("=").to_sym
      @changed_attributes << key
      @attributes[key] = args.first
    elsif @attributes.key?(method_name)
      @attributes[method_name]
    else
      super
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_str = method_name.to_s
    method_str.end_with?("=") || @attributes.key?(method_name) || super
  end
end

# ============================================================
# ทดสอบการทำงาน
# ============================================================

record = DynamicRecord.new(name: "มานี", age: 25, city: "กรุงเทพ")

puts "=== อ่านค่าเริ่มต้น ==="
puts record.name    # => มานี
puts record.age      # => 25
puts record.city     # => กรุงเทพ

puts "\n=== has_attribute? ==="
puts record.has_attribute?(:name)     # => true
puts record.has_attribute?(:email)    # => false

puts "\n=== attributes ==="
p record.attributes   # => {:name=>"มานี", :age=>25, :city=>"กรุงเทพ"}

puts "\n=== แก้ไขค่า ==="
record.age = 26
record.email = "manee@example.com"   # attribute ใหม่ที่ไม่เคยมีมาก่อนก็ตั้งค่าได้
puts record.age      # => 26
puts record.email    # => manee@example.com

puts "\n=== save_log ==="
puts record.save_log
# => attribute ที่ถูกแก้ไข: age, email

puts "\n=== respond_to? ==="
puts record.respond_to?(:name)     # => true
puts record.respond_to?(:phone)    # => false

puts "\n=== ตรวจสอบว่า attributes คืน copy ไม่ใช่ reference ==="
snapshot = record.attributes
snapshot[:name] = "ถูกแก้จากภายนอก"
puts record.name   # => มานี (ไม่เปลี่ยน เพราะ attributes คืน .dup)

puts "\n=== เรียก attribute ที่ไม่มีอยู่จริง ==="
begin
  record.phone
rescue NoMethodError => e
  puts "เกิด error ตามที่คาดหวัง: #{e.message.lines.first}"
end
```

ทดสอบรัน:

```bash
ruby dynamic_record.rb
```

**ผลลัพธ์ที่คาดหวัง (ประมาณ):**

```
=== อ่านค่าเริ่มต้น ===
มานี
25
กรุงเทพ

=== has_attribute? ===
true
false

=== attributes ===
{:name=>"มานี", :age=>25, :city=>"กรุงเทพ"}

=== แก้ไขค่า ===
26
manee@example.com

=== save_log ===
attribute ที่ถูกแก้ไข: age, email

=== respond_to? ===
true
false

=== ตรวจสอบว่า attributes คืน copy ไม่ใช่ reference ===
มานี

=== เรียก attribute ที่ไม่มีอยู่จริง ===
เกิด error ตามที่คาดหวัง: undefined method 'phone' for #<DynamicRecord:0x... >
```

### สิ่งที่ได้ฝึกจากเฉลยนี้

- ผสม `method_missing` + `respond_to_missing?` เข้าด้วยกันอย่างสอดคล้อง (consistent)
  ตามกฎเหล็กที่เรียนใน Step 146
- แยกแยะชัดเจนว่า method ไหนควรเป็น method ปกติ (`has_attribute?`, `attributes`,
  `save_log`) และ method ไหนควรพึ่งพา `method_missing` (getter/setter ที่ไม่รู้ชื่อ
  attribute ล่วงหน้า) — หลักการคือ **ใช้ `def` ตรงๆ เป็นค่าเริ่มต้นเสมอ ใช้
  `method_missing` เฉพาะจุดที่จำเป็นจริงๆ**
- เรียก `super` ใน `method_missing`/`respond_to_missing?` เพื่อไม่ให้ object "โกหก" ว่า
  ตอบสนอง attribute ที่ไม่เคยมีอยู่จริง
- ใช้ `.dup` ใน `attributes` เพื่อป้องกันการแก้ไขข้อมูลภายในจากภายนอกโดยไม่ผ่าน setter —
  เป็นการรักษาหลักการ encapsulation แม้จะเปิดช่องให้เข้าถึงข้อมูลแบบไดนามิกก็ตาม
- เห็นภาพรวมว่า attribute-based object เช่นนี้ (ที่คล้าย ActiveRecord/OpenStruct) ทำงาน
  อย่างไรเบื้องหลัง ทำให้เข้าใจ Rails ในภายหลังได้ลึกขึ้นมาก

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม method `revert!(key)` ให้กับ `DynamicRecord` ที่คืนค่า attribute กลับไปเป็นค่า
   ที่ตั้งไว้ตอน `initialize` ครั้งแรก (ใบ้: ต้องเก็บ `@original_attributes` แยกไว้ตอน
   `initialize` โดย `.dup` ไว้ตั้งแต่แรก แล้วดึงค่ากลับมาจากตรงนั้น) และเป็น dangerous
   method ตามธรรมเนียมที่เรียนใน Part 007 (ลงท้ายด้วย `!` เพราะ mutate ค่าเดิม)

2. เขียน class `EventLogger` ที่ใช้ `define_method` ร่วมกับ loop (แบบ Step 145) สร้าง
   method `log_info`, `log_warning`, `log_error` จาก Array `%i[info warning error]`
   โดยแต่ละ method รับ message เป็น argument แล้วคืน String รูปแบบ
   `"[LEVEL] message"` เช่น `logger.log_warning("พื้นที่จัดเก็บใกล้เต็ม")` ต้องได้
   `"[WARNING] พื้นที่จัดเก็บใกล้เต็ม"`

3. เขียน method `call_if_supported(object, method_name, *args)` ที่ใช้ `respond_to?`
   ร่วมกับ `public_send` (ตามหลักที่เรียนใน Step 142–143) เพื่อเรียก method บน object
   ใดๆ ก็ได้อย่างปลอดภัย โดยถ้า `method_name` ไม่ใช่ public method ของ object นั้น (ไม่ว่า
   จะไม่มีอยู่จริงหรือเป็น private) ให้คืนค่า `nil` แทนที่จะโยน exception — แล้วทดสอบกับ
   ทั้ง object ปกติ, `DynamicRecord` จากแบบฝึกหัดก่อนหน้า, และชื่อ method ที่เป็น private
   ของ object เพื่อยืนยันว่าฟังก์ชันนี้ปลอดภัยจริง (ใบ้: ผลลัพธ์ควรต่างจากการใช้ `send`
   ตรงๆ ตามที่เรียนเรื่องความเสี่ยงด้านความปลอดภัยใน Step 142)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **metaprogramming** คือการเขียนโค้ดที่สร้าง/ดัดแปลงโค้ดอื่นตอนรันโปรแกรม และ
  เห็นภาพชัดเจนว่า Rails feature ที่ดู "เป็นเวทมนตร์" (`has_many`, `validates`,
  ActiveRecord attribute methods, generator) ล้วนสร้างจากเทคนิคพื้นฐานของ Ruby ทั้งสิ้น
- ใช้ `send`/`public_send` เรียก method แบบไดนามิกด้วยชื่อ/symbol ได้ และเข้าใจความเสี่ยง
  ด้านความปลอดภัยของ `send` เมื่อชื่อ method มาจาก input ที่ไม่น่าเชื่อถือ พร้อมรู้ว่าเมื่อ
  ไหร่ควรใช้ `public_send` แทน
- ใช้ `respond_to?` ตรวจสอบว่า object มี method ก่อนเรียกได้ และเข้าใจ pattern
  `respond_to? -> send/public_send` ที่พบบ่อยในโค้ดที่ต้องรองรับ object หลายชนิด
- ใช้ `define_method` นิยาม method แบบไดนามิกได้ และประยุกต์ร่วมกับ loop เพื่อ generate
  method จำนวนมากจาก data ที่มีอยู่แล้วอย่าง DRY (ทั้ง predicate method และ
  getter/setter)
- เข้าใจกฎเหล็ก **`method_missing` ต้องมาคู่กับ `respond_to_missing?` เสมอ** และรู้ว่าทำไม
  การลืม override `respond_to_missing?` ถึงทำให้ object "โกหก" เกี่ยวกับความสามารถของ
  ตัวเอง รวมถึงความสำคัญของการเรียก `super` เมื่อไม่เข้าเงื่อนไขที่กำหนด
- เขียน dynamic attribute object ที่คล้าย ActiveRecord/OpenStruct ได้เองด้วย
  `method_missing` และเข้าใจว่าเบื้องหลังของทั้งสองทำงานอย่างไร
- ใช้ `instance_variable_get`/`instance_variable_set` เข้าถึง instance variable แบบ
  ไดนามิกได้ พร้อมเข้าใจว่าเทคนิคนี้ทำลาย encapsulation จึงควรใช้เฉพาะในบริบทของ
  library/infrastructure code เท่านั้น
- เข้าใจความแตกต่างและการใช้งานเบื้องต้นของ `class_eval` (เปิด class ทั้งหมด) กับ
  `instance_eval` (ปรับเฉพาะ object ตัวเดียว) ซึ่งเป็นรากฐานของการเขียน DSL แบบที่ Rails
  ใช้ทั่วทั้ง framework
- ตระหนักถึงต้นทุนของ metaprogramming (debug ยาก, IDE/static tool ตามไม่ทัน, performance
  cost, อ่านยากสำหรับคนใหม่) และยึดหลัก **ใช้อย่างประหยัดและมี comment อธิบายกำกับเสมอ**
- ลงมือสร้าง `DynamicRecord` ที่รวมทุกเทคนิคใน Part นี้เข้าด้วยกันในโปรเจกต์เดียว

**ต่อไป (Part 016):** เราจะเรียนเรื่อง **Duck typing, หลักการ SOLID ใน Ruby, และ design
pattern เบื้องต้น (Strategy, Observer)** — ต่อยอดจาก `respond_to?` ที่เรียนใน Part นี้
เพื่อเข้าใจว่าทำไม Ruby ถึงให้ความสำคัญกับ "พฤติกรรมที่ object ทำได้" มากกว่า "ชนิดของ
object" และเรียนรู้หลักการออกแบบซอฟต์แวร์ที่จะเป็นรากฐานสำคัญก่อนไปเขียน Rails application
ขนาดใหญ่ในเฟสถัดไป
