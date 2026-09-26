# Part 016: Duck Typing, SOLID Principles ใน Ruby, และ Design Pattern เบื้องต้น (Strategy, Observer)

> **Step ครอบคลุมใน Part นี้:** Step 151–160
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 007–010 เรื่อง Methods/OOP/Inheritance/Module มาก่อน
> และควรผ่าน Part 015 เรื่อง Metaprogramming มาแล้ว โดยเฉพาะ `respond_to?`)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 151: Duck typing ทบทวนและเจาะลึก — protocol โดยนัยของ Ruby
- Step 152: Single Responsibility Principle (SRP) — class ควรมีเหตุผลให้เปลี่ยนแค่เหตุผลเดียว
- Step 153: Open/Closed Principle (OCP) — เปิดสำหรับขยาย ปิดสำหรับแก้ไข
- Step 154: Liskov Substitution Principle (LSP) — บทเรียนคลาสสิกจาก Rectangle/Square
- Step 155: Interface Segregation Principle (ISP) — Ruby ไม่มี interface แต่มี module
- Step 156: Dependency Inversion Principle (DIP) — inject dependency แทนการ hardcode class
- Step 157: Strategy Pattern แบบ Ruby — ใช้ Proc/Lambda และ object เล็กๆ แทนโครงสร้างหนักแบบ Java
- Step 158: Observer Pattern — เขียนเองตั้งแต่ต้นด้วย module
- Step 159: Observer Pattern ด้วย `Observable` module ของ Ruby standard library
- Step 160: แบบฝึกหัด — ระบบแจ้งเตือนที่รวม Strategy (ช่องทางส่ง) และ Observer (เหตุการณ์)

---

## Step 151: Duck typing ทบทวนและเจาะลึก — protocol โดยนัยของ Ruby

### ทบทวนสั้นๆ จาก Part 015

ใน Part 015 Step 143 เราเจอประโยคนี้ไปแล้วตอนพูดถึง `respond_to?`:

> แทนที่จะเช็ค `object.is_a?(Hash)` หรือ `object.class == Hash` (ซึ่งผูกติดกับ **ชนิด**
> ของ object) เราเช็คว่า object **"ทำอะไรได้บ้าง"** ผ่าน `respond_to?` แทน — นี่คือหัวใจของ
> **duck typing**

Part นี้จะขยายแนวคิดนั้นให้เป็นระบบ และต่อยอดไปสู่หลักการออกแบบซอฟต์แวร์ระดับสูงขึ้น
(SOLID) ที่ล้วนพึ่งพา duck typing เป็นรากฐานทั้งสิ้น

### นิยามของ Duck Typing

**Duck typing** มาจากสำนวนภาษาอังกฤษ:

> "If it walks like a duck and quacks like a duck, it must be a duck."
> (ถ้ามันเดินเหมือนเป็ด ร้องเหมือนเป็ด มันก็คือเป็ด — ไม่ว่ามันจะเรียกตัวเองว่าอะไรก็ตาม)

แปลเป็นภาษาโปรแกรมมิ่ง: **Ruby ไม่สนใจว่า object เป็น "ชนิด" (class) อะไร สนใจแค่ว่า
object นั้น "ตอบสนอง method ที่เราจะเรียก" ได้หรือไม่** ต่างจากภาษาที่เป็น static typing
อย่าง Java/C# ที่ต้องประกาศ `interface` ตายตัวก่อน แล้วบังคับให้ class ที่จะใช้งานร่วมกัน
ต้อง `implements` interface นั้นอย่างชัดเจน

```ruby
# frozen_string_literal: true

class Dog
  def speak
    "โฮ่ง!"
  end
end

class Cat
  def speak
    "เมี้ยว!"
  end
end

class Robot
  def speak
    "บี๊บ บี๊บ!"
  end
end

[Dog.new, Cat.new, Robot.new].each do |animal|
  puts animal.speak
end
# => โฮ่ง!
# => เมี้ยว!
# => บี๊บ บี๊บ!
```

**อธิบาย:** `Dog`, `Cat`, `Robot` **ไม่มีความสัมพันธ์กันเลยแม้แต่น้อย** — ไม่ได้สืบทอด
class เดียวกัน ไม่ได้ include module ร่วมกัน ไม่มี interface ใดๆ บังคับไว้ แต่ loop
`each { |animal| animal.speak }` ทำงานได้กับทั้งสามตัวเป๊ะๆ เพราะ Ruby ไม่เคยถามว่า
"animal คือ class อะไร" มันแค่เรียก `.speak` แล้วเชื่อว่า object จะ "ตอบสนอง" ได้ถูกต้อง

### เปรียบเทียบ: ตรวจสอบชนิด (type-checking) เทียบกับ duck typing

```ruby
# แบบตรวจสอบชนิดตรงๆ (type-checking) — ผูกติดกับ class ที่รู้จักไว้ล่วงหน้าเท่านั้น
def make_it_speak_bad(animal)
  if animal.is_a?(Dog) || animal.is_a?(Cat)
    animal.speak
  else
    raise TypeError, "ไม่รู้จัก object ชนิดนี้"
  end
end

make_it_speak_bad(Robot.new)
# TypeError: ไม่รู้จัก object ชนิดนี้
# ทั้งที่ Robot ก็มี .speak เหมือนกัน! โค้ดนี้ตัดโอกาส Robot ทิ้งไปโดยไม่จำเป็น

# แบบ duck typing — สนใจแค่ "ทำอะไรได้บ้าง"
def make_it_speak_good(animal)
  if animal.respond_to?(:speak)
    animal.speak
  else
    raise TypeError, "object นี้พูดไม่เป็น"
  end
end

puts make_it_speak_good(Robot.new)   # => บี๊บ บี๊บ! (ทำงานได้ทันที ไม่ต้องแก้โค้ดเดิมเลย)
```

**ข้อคิดสำคัญ:** `make_it_speak_bad` ต้องแก้โค้ด (`if animal.is_a?(...)`) ทุกครั้งที่มี
class ใหม่ที่ "พูดได้" เกิดขึ้น ส่วน `make_it_speak_good` **ใช้งานได้กับ class ใดๆ ในอนาคต
โดยไม่ต้องแก้เลย** ตราบใดที่ class นั้นมี method `speak` — นี่คือตัวอย่างแรกที่ปูทางไปสู่
หลักการ **Open/Closed Principle** ที่จะเรียนใน Step 153

### ขยายตัวอย่าง `serialize` จาก Part 015

ใน Part 015 Step 143 เราเขียน method `serialize` ที่เลือกวิธีแปลง object เป็นข้อมูล
ตามความสามารถของมัน (`to_h` > `to_a` > `to_s`) มาดูอีกครั้งพร้อมเพิ่มชนิดใหม่เข้าไป
โดย **ไม่ต้องแก้ method `serialize` แม้แต่บรรทัดเดียว**:

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

# class ใหม่ที่ยังไม่เคยมีในตอนที่เขียน serialize — ไม่มีความสัมพันธ์ใดๆ กับ Point เลย
class TemperatureReading
  def initialize(celsius)
    @celsius = celsius
  end

  def to_s
    "#{@celsius}°C"
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

p serialize({ name: "มานี", age: 25 })       # => {:name=>"มานี", :age=>25}
p serialize(Point.new(1, 2))                   # => [1, 2]
p serialize(TemperatureReading.new(32))        # => "32°C"  (class ใหม่ทำงานได้ทันที)
```

`serialize` ไม่เคยรู้จัก `TemperatureReading` มาก่อนตอนถูกเขียน แต่ก็ใช้งานร่วมกันได้ทันที
เพราะมันไม่ได้ถามหา "ชนิด" ของ object เลยสักครั้ง — นี่คือพลังที่แท้จริงของ duck typing

### Protocol โดยนัยของ Ruby เอง (implicit protocols)

Ruby ไม่ได้ใช้แนวคิดนี้แค่ในโค้ดที่เราเขียน แต่ตัวภาษาและ standard library เองก็สร้างขึ้น
บน duck typing ทั้งระบบ ผ่าน "protocol" ที่ไม่มีการประกาศเป็นทางการ แต่ทุก object ที่มี
method ตามชื่อที่กำหนดจะได้รับสิทธิ์เข้าร่วมพฤติกรรมนั้นทันที:

**1. `to_s` — ทุกที่ที่ Ruby ต้องแสดงผล object เป็นข้อความ จะเรียก `to_s` ให้อัตโนมัติ**

```ruby
class Money
  def initialize(amount)
    @amount = amount
  end

  def to_s
    "#{@amount} บาท"
  end
end

puts Money.new(100)              # => 100 บาท (puts เรียก to_s ให้เอง)
puts "ราคา: #{Money.new(250)}"     # => ราคา: 250 บาท (string interpolation ก็เรียก to_s เช่นกัน)
```

**2. `each` + `Enumerable` — แค่มี `each` ตัวเดียวก็ได้ `map`, `select`, `count`,
`sort`, `reduce` ฯลฯ ฟรีทั้งหมด** (เรียนพื้นฐาน `Enumerable` ไปแล้วใน Part 013)

```ruby
class Playlist
  include Enumerable

  def initialize(songs)
    @songs = songs
  end

  def each
    @songs.each { |song| yield song }
  end
end

playlist = Playlist.new(["เพลง A", "เพลง B", "เพลง C"])

p playlist.map(&:upcase)                       # => ["เพลง A", "เพลง B", "เพลง C"] (ผลลัพธ์ภาษาไทยไม่มีตัวพิมพ์ใหญ่/เล็ก แต่ method ทำงานได้)
puts playlist.count                              # => 3
p playlist.select { |song| song.include?("B") }  # => ["เพลง B"]
```

**อธิบาย:** `Enumerable` **ไม่รู้จัก `Playlist` มาก่อนเลย** มันแค่ "เชื่อ" ว่า class ใดก็ตาม
ที่ include มันไว้ต้องมี method `each` ที่ yield ค่าออกมาได้ — เมื่อเงื่อนไขนี้เป็นจริง
`map`/`select`/`count` ทั้งหมด (ซึ่งเขียนโดยใช้ `each` เป็นแกนกลางภายใน `Enumerable`) ก็
ทำงานได้ทันที นี่คือตัวอย่างจริงของ duck typing ที่ฝังอยู่ในแกนกลางของ Ruby เอง

**3. `<=>` + `Comparable` — แค่มี `<=>` ตัวเดียวก็ได้ `<`, `>`, `==`, `between?`,
`clamp`, `sort` ฟรีทั้งหมด** (เรียนพื้นฐาน `Comparable` ไปแล้วใน Part 014)

```ruby
class Version
  include Comparable

  attr_reader :major, :minor

  def initialize(major, minor)
    @major = major
    @minor = minor
  end

  def <=>(other)
    [major, minor] <=> [other.major, other.minor]
  end

  def to_s
    "#{major}.#{minor}"
  end
end

puts Version.new(1, 5) < Version.new(2, 0)   # => true
versions = [Version.new(2, 0), Version.new(1, 9), Version.new(1, 2)]
puts versions.sort.map(&:to_s).join(", ")      # => 1.2, 1.9, 2.0
```

> **ข้อคิดสำคัญที่จะใช้ตลอด Part นี้:** `Enumerable` และ `Comparable` ทั้งสองไม่ได้เช็ค
> `is_a?` หรือ `class ==` เลยแม้แต่ครั้งเดียว — มันพึ่งพา duck typing ล้วนๆ นี่คือเหตุผลที่
> object ใดๆ ใน Ruby ก็สามารถ "เข้าร่วม" ระบบเหล่านี้ได้ทันทีเพียงแค่มี method ที่ถูกต้อง
> หลักการออกแบบซอฟต์แวร์ (SOLID) ที่กำลังจะเรียนต่อไปในทุก Step ที่เหลือของ Part นี้ ล้วน
> เป็นการนำแนวคิด "พึ่งพาพฤติกรรม ไม่ใช่พึ่งพาชนิด" นี้ไปประยุกต์ใช้อย่างเป็นระบบทั้งสิ้น

---

## Step 152: Single Responsibility Principle (SRP) — class ควรมีเหตุผลให้เปลี่ยนแค่เหตุผลเดียว

### ภาพรวมของ SOLID

**SOLID** เป็นตัวย่อของหลักการออกแบบซอฟต์แวร์เชิงวัตถุ 5 ข้อ ที่ Robert C. Martin
("Uncle Bob") รวบรวมไว้ เพื่อช่วยให้โค้ดดูแลรักษาง่าย ขยายได้โดยไม่พังของเดิม และทดสอบได้
สะดวก:

| ตัวย่อ | ชื่อเต็ม | แนวคิดหลัก | Step |
|---|---|---|---|
| **S** | Single Responsibility Principle | class ควรมี "เหตุผลให้เปลี่ยน" แค่เหตุผลเดียว | 152 |
| **O** | Open/Closed Principle | เปิดให้ขยายพฤติกรรมได้ แต่ปิดไม่ให้แก้โค้ดเดิม | 153 |
| **L** | Liskov Substitution Principle | subclass ต้องแทนที่ parent class ได้โดยไม่พังพฤติกรรม | 154 |
| **I** | Interface Segregation Principle | อย่าบังคับ object ให้พึ่งพา method ที่มันไม่ได้ใช้ | 155 |
| **D** | Dependency Inversion Principle | พึ่งพา abstraction ไม่ใช่พึ่งพา class ที่เจาะจงตายตัว | 156 |

หลักการเหล่านี้ถูกคิดขึ้นในบริบทของภาษา static typing (เช่น Java/C++) เป็นหลัก แต่ทุกข้อ
**นำมาใช้กับ Ruby ได้ และมักจะเขียนได้ **สั้นและเป็นธรรมชาติกว่า** เพราะ duck typing
ที่เรียนใน Step 151 ทำให้ไม่ต้องมี interface/abstract class อย่างเป็นทางการเหมือนภาษาอื่น

### นิยามของ SRP

> **A class should have only one reason to change.**
> (class หนึ่งควรมี "เหตุผลให้ต้องแก้ไข" เพียงเหตุผลเดียว)

"เหตุผลให้เปลี่ยน" ในที่นี้หมายถึง **กลุ่มผู้มีส่วนได้ส่วนเสีย (stakeholder) หรือความ
รับผิดชอบ** ที่ทำให้ต้องแก้โค้ด — ถ้า class หนึ่งรับผิดชอบหลายเรื่องที่ไม่เกี่ยวข้องกัน
การแก้ไขเรื่องหนึ่งอาจไปกระทบอีกเรื่องโดยไม่ตั้งใจ และทำให้ class นั้นแก้ยาก ทดสอบยาก

### ตัวอย่าง "ก่อน" ที่ละเมิด SRP

```ruby
# frozen_string_literal: true

class InvoiceReport
  def initialize(items)
    @items = items   # array ของ { name:, price:, quantity: }
  end

  # ความรับผิดชอบที่ 1: คำนวณ
  def total
    @items.sum { |item| item[:price] * item[:quantity] }
  end

  # ความรับผิดชอบที่ 2: จัดรูปแบบข้อความ
  def to_text
    lines = @items.map do |item|
      "#{item[:name]} x#{item[:quantity]} = #{item[:price] * item[:quantity]}"
    end
    "#{lines.join("\n")}\nรวม: #{total}"
  end

  # ความรับผิดชอบที่ 3: เขียนไฟล์ (I/O)
  def save_to_file(path)
    File.write(path, to_text)
  end

  # ความรับผิดชอบที่ 4: ส่งอีเมล (I/O + external service)
  def send_email(to)
    puts "ส่งอีเมลถึง #{to}:\n#{to_text}"
  end
end
```

**ปัญหา:** `InvoiceReport` มี **4 เหตุผลที่ทำให้ต้องแก้ไข**:

1. สูตรคำนวณยอดรวมเปลี่ยน (เช่น เพิ่มภาษี/ส่วนลด)
2. รูปแบบข้อความที่แสดงเปลี่ยน (เช่น เปลี่ยนจากข้อความธรรมดาเป็น HTML)
3. วิธีเขียนไฟล์เปลี่ยน (เช่น เปลี่ยนจากไฟล์ local เป็น cloud storage)
4. ผู้ให้บริการอีเมลเปลี่ยน (เช่น เปลี่ยนจาก SMTP เป็น API ของบุคคลที่สาม)

ถ้าทีมการตลาดขอเปลี่ยนแค่ "รูปแบบข้อความ" แต่ต้องมาแก้ไฟล์เดียวกับที่มีโค้ดคำนวณเงินอยู่
ความเสี่ยงที่จะเผลอทำโค้ดคำนวณพังตามไปด้วยก็สูงขึ้น — นี่คือสิ่งที่ SRP พยายามป้องกัน

### ตัวอย่าง "หลัง" ที่ทำตาม SRP

```ruby
# frozen_string_literal: true

# ความรับผิดชอบเดียว: เก็บข้อมูลและคำนวณยอดรวม
class Invoice
  attr_reader :items

  def initialize(items)
    @items = items
  end

  def total
    items.sum { |item| item[:price] * item[:quantity] }
  end
end

# ความรับผิดชอบเดียว: จัดรูปแบบ invoice ให้เป็นข้อความ
class InvoiceTextFormatter
  def initialize(invoice)
    @invoice = invoice
  end

  def to_text
    lines = @invoice.items.map do |item|
      "#{item[:name]} x#{item[:quantity]} = #{item[:price] * item[:quantity]}"
    end
    "#{lines.join("\n")}\nรวม: #{@invoice.total}"
  end
end

# ความรับผิดชอบเดียว: เขียนไฟล์
class InvoiceFileWriter
  def initialize(formatter)
    @formatter = formatter
  end

  def save(path)
    File.write(path, @formatter.to_text)
  end
end

# ความรับผิดชอบเดียว: ส่งอีเมล
class InvoiceMailer
  def initialize(formatter)
    @formatter = formatter
  end

  def send_to(email)
    puts "ส่งอีเมลถึง #{email}:\n#{@formatter.to_text}"
  end
end

items = [
  { name: "หนังสือ Ruby", price: 350, quantity: 2 },
  { name: "ปากกา", price: 15, quantity: 5 }
]

invoice = Invoice.new(items)
formatter = InvoiceTextFormatter.new(invoice)

puts formatter.to_text
# => หนังสือ Ruby x2 = 700
# => ปากกา x5 = 75
# => รวม: 775

InvoiceMailer.new(formatter).send_to("customer@example.com")
```

**อธิบาย:** ตอนนี้แต่ละ class มี "เหตุผลให้เปลี่ยน" เพียงเหตุผลเดียว:

- `Invoice` เปลี่ยนเมื่อ**กฎการคำนวณ**เปลี่ยน
- `InvoiceTextFormatter` เปลี่ยนเมื่อ**รูปแบบการแสดงผล**เปลี่ยน
- `InvoiceFileWriter` เปลี่ยนเมื่อ**วิธีเขียนไฟล์**เปลี่ยน
- `InvoiceMailer` เปลี่ยนเมื่อ**วิธีส่งอีเมล**เปลี่ยน

และเพราะทุก class เล็กและโฟกัสเรื่องเดียว การเขียน unit test ให้แต่ละ class (ซึ่งจะเรียน
จริงจังใน Part 018–019) ก็ทำได้ง่ายกว่าและแคบกว่ามาก — ทดสอบ `Invoice#total` ไม่จำเป็น
ต้องยุ่งกับการเขียนไฟล์หรือส่งอีเมลเลย

> **ข้อควรระวัง:** SRP **ไม่ได้แปลว่า "class ต้องมี method เดียว"** เป็นความเข้าใจผิดที่พบ
> บ่อย ประเด็นคือ "เหตุผลให้เปลี่ยน" ไม่ใช่ "จำนวน method" — `Invoice` มี method เดียว
> (`total`) ในตัวอย่างนี้เพราะโมเดลง่าย แต่ถ้ามันมี method อื่นที่ยังเกี่ยวกับ "การคำนวณ/
> ข้อมูลของใบแจ้งหนี้" เพิ่มเข้ามาอีก (เช่น `subtotal`, `tax`, `discount`) ก็ยังถือว่าทำตาม
> SRP อยู่ ตราบใดที่ยังเป็นเหตุผลเดียวกัน (การคำนวณ) ไม่ได้ปนกับการจัดรูปแบบหรือ I/O

---

## Step 153: Open/Closed Principle (OCP) — เปิดสำหรับขยาย ปิดสำหรับแก้ไข

### นิยาม

> **Software entities should be open for extension, but closed for modification.**
> (ควรเพิ่มพฤติกรรมใหม่ได้ โดยไม่ต้องแก้ไขโค้ดเดิมที่ทำงานอยู่แล้ว)

หลักการนี้ต่อยอดโดยตรงจากตัวอย่าง `make_it_speak_bad` vs `make_it_speak_good` ใน Step 151
— ทุกครั้งที่ต้องแก้ `if/elsif`/`case` เดิมเพื่อรองรับกรณีใหม่ นั่นคือสัญญาณว่ากำลังละเมิด OCP

### ตัวอย่าง "ก่อน" ที่ละเมิด OCP

```ruby
# frozen_string_literal: true

class DiscountCalculator
  def calculate(customer_type, price)
    case customer_type
    when :regular
      price
    when :member
      price * 0.9
    when :vip
      price * 0.8
    else
      raise ArgumentError, "ไม่รู้จักประเภทลูกค้า: #{customer_type}"
    end
  end
end

calculator = DiscountCalculator.new
puts calculator.calculate(:member, 1000)   # => 900.0
```

**ปัญหา:** ถ้าธุรกิจเพิ่มลูกค้าประเภท `:gold` เข้ามา เราต้อง**แก้ไขโค้ดเดิมของ
`DiscountCalculator#calculate`** โดยตรง (เพิ่ม `when :gold` เข้าไปใน `case`) ทุกครั้งที่
เพิ่มประเภทลูกค้าใหม่ ความเสี่ยงคือการแก้ไข method นี้อาจไปกระทบ logic ของประเภทลูกค้าเดิม
ที่ทำงานถูกต้องอยู่แล้วโดยไม่ตั้งใจ (regression) และยิ่งจำนวนประเภทมากขึ้น `case` ก็ยิ่ง
ยาวและอ่านยากขึ้นเรื่อยๆ

### ตัวอย่าง "หลัง" ที่ทำตาม OCP ด้วย polymorphism

```ruby
# frozen_string_literal: true

class RegularPricing
  def calculate(price)
    price
  end
end

class MemberPricing
  def calculate(price)
    price * 0.9
  end
end

class VipPricing
  def calculate(price)
    price * 0.8
  end
end

class DiscountCalculator
  def initialize(pricing_strategy)
    @pricing_strategy = pricing_strategy
  end

  def calculate(price)
    @pricing_strategy.calculate(price)
  end
end

member_calculator = DiscountCalculator.new(MemberPricing.new)
puts member_calculator.calculate(1000)   # => 900.0

# เพิ่มลูกค้าประเภทใหม่ทั้งหมด โดยไม่แก้โค้ดเดิมของ DiscountCalculator แม้แต่บรรทัดเดียว
class GoldPricing
  def calculate(price)
    price * 0.7
  end
end

gold_calculator = DiscountCalculator.new(GoldPricing.new)
puts gold_calculator.calculate(1000)   # => 700.0
```

**อธิบาย:** `DiscountCalculator` ตอนนี้**ไม่รู้จักประเภทลูกค้าเลยแม้แต่น้อย** มันรู้แค่ว่า
สิ่งที่ส่งเข้ามาต้องมี method `calculate(price)` (duck typing อีกครั้ง!) การเพิ่มประเภท
ลูกค้าใหม่คือการ **สร้าง class ใหม่** (`GoldPricing`) โดย**ไม่ต้องแตะโค้ดของ
`DiscountCalculator`, `RegularPricing`, `MemberPricing`, `VipPricing` ที่มีอยู่แล้วเลย**
— นี่คือความหมายของ "เปิดสำหรับขยาย (เพิ่ม class ใหม่ได้เสมอ) ปิดสำหรับแก้ไข (โค้ดเดิม
ไม่ต้องถูกแตะต้อง)"

> **หมายเหตุ:** รูปแบบ `RegularPricing`, `MemberPricing`, `VipPricing` ที่ implement
> `calculate` เหมือนกันแต่มีพฤติกรรมต่างกัน และถูก inject เข้าไปให้ object อื่นเรียกใช้
> คือแก่นของ **Strategy Pattern** ที่จะเรียนอย่างเป็นทางการใน Step 157 — OCP และ Strategy
> Pattern จึงเป็นเรื่องเดียวกันมองจากคนละมุม

---

## Step 154: Liskov Substitution Principle (LSP) — บทเรียนคลาสสิกจาก Rectangle/Square

### นิยาม

> **Objects of a superclass should be replaceable with objects of its subclasses without
> breaking the application.**
> (แทนที่ object ของ class แม่ด้วย object ของ class ลูกได้ โดยไม่ทำให้พฤติกรรมของโปรแกรม
> ผิดเพี้ยนไป)

พูดง่ายๆ คือ: ถ้า `Square` สืบทอดจาก `Rectangle` โค้ดทุกที่ที่ใช้งาน `Rectangle` ได้ ต้อง
ใช้งาน `Square` แทนได้เหมือนกันทุกประการ โดยไม่มีอะไรพัง — ถ้าไม่เป็นแบบนั้น แปลว่าความ
สัมพันธ์แบบ inheritance นั้น**ผิด** ตั้งแต่ต้น

### ตัวอย่างคลาสสิกที่ละเมิด LSP: Rectangle/Square

ในทางคณิตศาสตร์ "สี่เหลี่ยมจัตุรัสก็คือสี่เหลี่ยมผืนผ้าชนิดหนึ่ง" (ด้านเท่ากันหมด) ทำให้
ดูสมเหตุสมผลที่จะเขียน `Square < Rectangle` แต่พอนำมาเขียนโปรแกรมจะเกิดปัญหาทันที:

```ruby
# frozen_string_literal: true

class Rectangle
  attr_accessor :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    width * height
  end
end

class Square < Rectangle
  def initialize(side)
    super(side, side)
  end

  # override เพื่อรักษากฎ "ด้านทุกด้านต้องเท่ากัน" ของสี่เหลี่ยมจัตุรัส
  def width=(value)
    @width = value
    @height = value
  end

  def height=(value)
    @width = value
    @height = value
  end
end

def resize_and_check(rectangle)
  rectangle.width = 5
  rectangle.height = 4
  expected_area = 20
  actual_area = rectangle.area
  puts "คาดหวัง: #{expected_area}, ได้จริง: #{actual_area}, ถูกต้องไหม? #{expected_area == actual_area}"
end

resize_and_check(Rectangle.new(2, 2))
# => คาดหวัง: 20, ได้จริง: 20, ถูกต้องไหม? true

resize_and_check(Square.new(2))
# => คาดหวัง: 20, ได้จริง: 16, ถูกต้องไหม? false   <- ละเมิด LSP!
```

**อธิบายว่าทำไมพัง:** `resize_and_check` เขียนขึ้นโดยอ้างอิงจาก **สัญญา (contract)** ของ
`Rectangle` ที่ว่า "ตั้ง `width` และ `height` แยกกันอิสระได้ และ `area` จะเท่ากับผลคูณของ
ทั้งสองค่าล่าสุดเสมอ" แต่ `Square` **ทำลายสัญญานั้น** เพราะการตั้งค่า `width` จะไปเปลี่ยน
`height` โดยอัตโนมัติด้วย (เพื่อรักษาความเป็นสี่เหลี่ยมจัตุรัส) ทำให้เมื่อ `resize_and_check`
เรียก `rectangle.width = 5` ตามด้วย `rectangle.height = 4` ผลลัพธ์ของ `Square` กลับกลาย
เป็น `4 x 4 = 16` แทนที่จะเป็น `5 x 4 = 20` ที่โค้ดคาดหวังไว้ — แม้ `Square` จะ "เป็น"
`Rectangle` ทาง type ก็ตาม (`Square.new(2).is_a?(Rectangle)` คืน `true`) แต่มัน**แทนที่
กันไม่ได้จริงในทางพฤติกรรม** ซึ่งคือแก่นของ LSP

### ทางแก้: อย่าบังคับความสัมพันธ์แบบ inheritance ที่พฤติกรรมไม่ตรงกัน

```ruby
# frozen_string_literal: true

# ไม่มี inheritance ระหว่างกันอีกต่อไป — แยกเป็นสอง class อิสระ
class Rectangle
  attr_reader :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end

  def area
    width * height
  end
end

class Square
  attr_reader :side

  def initialize(side)
    @side = side
  end

  def area
    side * side
  end
end

# ทั้งสอง class ไม่มีความสัมพันธ์ทาง class เลย แต่ยังใช้งานร่วมกันได้ผ่าน duck typing
# (เหมือนหลักการที่เรียนใน Step 151) เพราะทั้งคู่มี method #area เหมือนกัน
shapes = [Rectangle.new(3, 4), Square.new(5)]
shapes.each { |shape| puts shape.area }
# => 12
# => 25
```

**อธิบาย:** เมื่อไม่มีความสัมพันธ์ inheritance ที่ผิดพลาด ก็ไม่มีสัญญาไหนถูกละเมิดอีกต่อไป
`Rectangle` และ `Square` เป็น object คนละแบบที่**บังเอิญ**มี method ชื่อ `area` เหมือนกัน
— นี่คือจุดที่ duck typing (Step 151) เข้ามาแทนที่ inheritance ได้อย่างปลอดภัยกว่า เมื่อ
สอง class มีพฤติกรรมคล้ายกันแต่**ไม่ได้เป็นชนิดย่อยของกันจริงๆ** ในทุกแง่มุม

> **กฎการตัดสินใจที่ใช้ได้จริง:** ก่อนเขียน `class B < A` ให้ถามตัวเองว่า "ทุก object
> ของ `B` ยังคงรักษาสัญญา (invariant) ทุกข้อของ `A` ได้จริงหรือไม่ ในทุกสถานการณ์ที่
> โค้ดซึ่งใช้งาน `A` คาดหวังไว้" ถ้าคำตอบคือ "ไม่แน่ใจ" หรือ "มีข้อยกเว้น" ให้เลือกใช้
> **composition** (มี object ของอีก class เป็น attribute) หรือ **duck typing ผ่าน
> module ร่วมกัน** (จะเรียนใน Step 155) แทนการสืบทอด — LSP จึงมักถูกสรุปสั้นๆ ว่า
> **"favor composition over inheritance"**

---

## Step 155: Interface Segregation Principle (ISP) — Ruby ไม่มี interface แต่มี module

### นิยาม

> **Clients should not be forced to depend on interfaces they do not use.**
> (ไม่ควรบังคับให้ class ต้อง implement method ที่มันไม่เคยใช้เลย)

ภาษาอย่าง Java/C# มี keyword `interface` ที่กำหนด "สัญญา" ของ method ที่ class ต้อง
implement ครบทุกตัว ถ้า interface นั้น "อ้วนเกินไป" (มี method เยอะเกินความจำเป็น)
class ที่ implement มันจะถูกบังคับให้เขียน method ที่ตัวเองไม่เคยใช้เลย

**Ruby ไม่มี `interface` keyword อย่างเป็นทางการ** — "สัญญา" ใน Ruby คือ duck typing
ล้วนๆ (มี method ชื่อนี้ไหม ตอบสนองได้ไหม) และเครื่องมือที่ใช้จัดกลุ่ม method ที่เกี่ยวข้อง
กันคือ **module** (เรียนพื้นฐานไปแล้วใน Part 010) ซึ่งทำให้ ISP นำมาใช้ได้เป็นธรรมชาติ
มากกว่าภาษาที่มี interface บังคับเสียอีก

### ตัวอย่าง "ก่อน" ที่ละเมิด ISP: interface อ้วนเกินไป

```ruby
# frozen_string_literal: true

# module นี้ทำหน้าที่เหมือน "interface อ้วน" ที่รวมทุกความสามารถของเครื่องพิมพ์ทุกรุ่นไว้ที่เดียว
module MultiFunctionDevice
  def print_document(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement print_document"
  end

  def scan_document(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement scan_document"
  end

  def send_fax(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement send_fax"
  end
end

class OldFashionedPrinter
  include MultiFunctionDevice

  def print_document(doc)
    "กำลังพิมพ์: #{doc}"
  end

  # ถูกบังคับให้ implement ทั้งที่เครื่องนี้สแกน/แฟกซ์ไม่ได้จริง
  def scan_document(_doc)
    raise NotImplementedError, "เครื่องพิมพ์รุ่นนี้สแกนไม่ได้"
  end

  def send_fax(_doc)
    raise NotImplementedError, "เครื่องพิมพ์รุ่นนี้แฟกซ์ไม่ได้"
  end
end
```

**ปัญหา:** `OldFashionedPrinter` **ถูกบังคับให้พึ่งพา method ที่มันไม่ได้ใช้เลย**
(`scan_document`, `send_fax`) แค่เพราะมัน include module ที่ "อ้วนเกินไป" — โค้ดฝั่งเรียกใช้
(client) ที่เห็นว่า object นี้ include `MultiFunctionDevice` อาจสันนิษฐานผิดว่าเรียก
`scan_document` ได้เสมอ แล้วมาพังตอนรันจริงแทน

### ทางแก้: แยก module ให้เล็กและโฟกัสเฉพาะเรื่อง

```ruby
# frozen_string_literal: true

module Printable
  def print_document(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement print_document"
  end
end

module Scannable
  def scan_document(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement scan_document"
  end
end

module Faxable
  def send_fax(doc)
    raise NotImplementedError, "#{self.class} ต้อง implement send_fax"
  end
end

# include เฉพาะความสามารถที่ตัวเองมีจริงเท่านั้น
class SimplePrinter
  include Printable

  def print_document(doc)
    "กำลังพิมพ์: #{doc}"
  end
end

class AllInOnePrinter
  include Printable
  include Scannable
  include Faxable

  def print_document(doc)
    "กำลังพิมพ์: #{doc}"
  end

  def scan_document(doc)
    "กำลังสแกน: #{doc}"
  end

  def send_fax(doc)
    "กำลังส่งแฟกซ์: #{doc}"
  end
end

# ฝั่งที่เรียกใช้งานก็ตรวจสอบความสามารถผ่าน duck typing (respond_to? ที่เรียนใน Part 015)
# แทนที่จะสมมติว่า object ทุกตัวทำได้ทุกอย่าง
def process_scan(device, doc)
  if device.respond_to?(:scan_document)
    puts device.scan_document(doc)
  else
    puts "#{device.class} ไม่รองรับการสแกน"
  end
end

process_scan(AllInOnePrinter.new, "รายงานประจำเดือน")   # => กำลังสแกน: รายงานประจำเดือน
process_scan(SimplePrinter.new, "รายงานประจำเดือน")     # => SimplePrinter ไม่รองรับการสแกน
```

**อธิบาย:** ตอนนี้ `SimplePrinter` **ไม่ถูกบังคับให้รู้จักหรือ implement**
`scan_document`/`send_fax` เลยแม้แต่น้อย มัน include เฉพาะ `Printable` ที่มันใช้จริง —
"ลูกค้า" (client code ที่เรียก `process_scan`) ก็ไม่ได้พึ่งพา interface ที่ใหญ่เกินจำเป็น
เพราะมันเช็คทีละความสามารถผ่าน `respond_to?` แทน

> **ทำไม ISP ถึงเข้ากับ Ruby ได้เป็นธรรมชาติ:** เพราะ Ruby ไม่มี "สัญญาบังคับ" แบบ
> `interface` ของภาษา static typing ตั้งแต่ต้น — module ใน Ruby เป็นแค่ "กลุ่มของ method
> ที่พร้อมให้หยิบไป include" ไม่ใช่ "ข้อบังคับ" จึงเป็นธรรมชาติอยู่แล้วที่จะแยก module
> ให้เล็กและเฉพาะเจาะจง (**focused module** / **role-based module**) แทนที่จะรวมทุกอย่าง
> ไว้ที่เดียว ต่างจากภาษาอื่นที่ต้องอาศัยวินัยของผู้เขียน interface เอง

---

## Step 156: Dependency Inversion Principle (DIP) — inject dependency แทนการ hardcode class

### นิยาม

> **High-level modules should not depend on low-level modules. Both should depend on
> abstractions.**
> (โมดูลระดับสูง [ตรรกะทางธุรกิจ] ไม่ควรพึ่งพาโมดูลระดับล่าง [รายละเอียดการทำงาน] โดยตรง
> ทั้งคู่ควรพึ่งพา "สัญญากลาง" ร่วมกันแทน)

พูดง่ายๆ คือ: class ที่ทำหน้าที่เป็น "policy" หรือ "business logic" ไม่ควร `new` class
ที่เป็นรายละเอียดการทำงาน (เช่น การส่งอีเมล, การเขียนไฟล์, การเรียก API) ขึ้นมาใช้เองตรงๆ
ภายในตัวมันเอง แต่ควร**รับมันเข้ามาจากภายนอก** (dependency injection) แทน

### ตัวอย่าง "ก่อน" ที่ละเมิด DIP

```ruby
# frozen_string_literal: true

class EmailSender
  def send_message(to, body)
    puts "ส่งอีเมลถึง #{to}: #{body}"
  end
end

class OrderProcessor
  def initialize
    @sender = EmailSender.new   # hardcode ผูกติดกับ class ที่เจาะจงตายตัวไว้ข้างใน
  end

  def complete_order(order)
    # ... logic การจัดการคำสั่งซื้อ (ตัดสต็อก, บันทึกฐานข้อมูล ฯลฯ) ...
    @sender.send_message(order[:customer_contact], "คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว")
  end
end
```

**ปัญหา:**

1. `OrderProcessor` (โมดูลระดับสูง ที่ดูแล "ตรรกะการทำคำสั่งซื้อให้เสร็จ") ผูกติดแน่นกับ
   `EmailSender` (โมดูลระดับล่าง ที่เป็นแค่รายละเอียดว่า "แจ้งเตือนยังไง") — ถ้าธุรกิจ
   อยากเปลี่ยนไปแจ้งเตือนผ่าน SMS แทน ต้องแก้โค้ดข้างใน `OrderProcessor` โดยตรง
2. เขียน test ให้ `OrderProcessor` ได้ยาก เพราะทุกครั้งที่ทดสอบ `complete_order` จะมีการ
   `puts`/ส่งอีเมลจริงติดมาด้วยเสมอ ไม่มีทางแยกทดสอบ "ตรรกะ" ออกจาก "การแจ้งเตือนจริง" ได้

### ตัวอย่าง "หลัง" ที่ทำตาม DIP — inject ผ่าน constructor

```ruby
# frozen_string_literal: true

class EmailSender
  def send_message(to, body)
    puts "ส่งอีเมลถึง #{to}: #{body}"
  end
end

class SmsSender
  def send_message(to, body)
    puts "ส่ง SMS ถึง #{to}: #{body}"
  end
end

class OrderProcessor
  # รับ notifier เข้ามาจากภายนอกผ่าน keyword argument พร้อม default value
  # ไม่ได้ระบุ "class" ที่ยอมรับไว้เลย — รับได้ทุก object ที่ตอบสนอง #send_message
  def initialize(notifier: EmailSender.new)
    @notifier = notifier
  end

  def complete_order(order)
    @notifier.send_message(order[:customer_contact], "คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว")
  end
end

default_processor = OrderProcessor.new
default_processor.complete_order(customer_contact: "manee@example.com")
# => ส่งอีเมลถึง manee@example.com: คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว

sms_processor = OrderProcessor.new(notifier: SmsSender.new)
sms_processor.complete_order(customer_contact: "0812345678")
# => ส่ง SMS ถึง 0812345678: คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว
```

**อธิบาย:** `OrderProcessor` **ไม่รู้จัก `EmailSender` หรือ `SmsSender` เป็นการเฉพาะเจาะจง
เลย** มันรู้แค่ว่า `@notifier` ต้องตอบสนอง `send_message(to, body)` ได้ (duck typing
อีกครั้ง — นี่คือ "abstraction" ที่ DIP พูดถึง เพียงแต่ Ruby ไม่ต้องเขียน `interface`
เป็นทางการเหมือนภาษาอื่น มันคือ "สัญญาโดยนัย" ที่สื่อสารผ่านชื่อ method และเอกสาร) การจะ
สลับช่องทางแจ้งเตือนทำได้แค่ **ส่ง object คนละตัวเข้าไปตอนสร้าง** โดยไม่ต้องแตะโค้ดภายใน
`OrderProcessor` แม้แต่บรรทัดเดียว — ตรงตามทั้ง DIP และ OCP (Step 153) ไปพร้อมกัน

### ประโยชน์ที่ชัดเจนที่สุด: การทดสอบด้วย fake object

```ruby
# class OrderProcessor เดิมจากตัวอย่างก่อนหน้า (แสดงซ้ำเพื่อให้โค้ดนี้รันได้ครบในตัวเอง)
class OrderProcessor
  def initialize(notifier: EmailSender.new)
    @notifier = notifier
  end

  def complete_order(order)
    @notifier.send_message(order[:customer_contact], "คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว")
  end
end

class FakeNotifier
  attr_reader :messages

  def initialize
    @messages = []
  end

  def send_message(to, body)
    @messages << [to, body]
  end
end

fake = FakeNotifier.new
test_processor = OrderProcessor.new(notifier: fake)
test_processor.complete_order(customer_contact: "test@example.com")

p fake.messages
# => [["test@example.com", "คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว"]]
```

**อธิบาย:** เพราะ `OrderProcessor` ไม่เคยผูกติดกับ `EmailSender` ตายตัว เราจึงสามารถส่ง
`FakeNotifier` (object ปลอมที่แค่เก็บข้อความไว้ตรวจสอบ แทนที่จะส่งจริง) เข้าไปแทนได้อย่าง
ง่ายดาย เพื่อตรวจสอบว่า `complete_order` เรียก notifier ด้วยข้อมูลที่ถูกต้องหรือไม่ โดย
**ไม่ต้องส่งอีเมล/SMS จริงเลยสักครั้งระหว่างทดสอบ** — นี่คือเทคนิคพื้นฐานที่เรียกว่า
**test double** ซึ่งจะเรียนอย่างเป็นระบบใน Part 018 (Minitest) และ Part 019 (RSpec)

> **สรุปความเชื่อมโยงของ SOLID ทั้ง 5 ข้อกับ duck typing:** สังเกตว่าทุกตัวอย่างใน
> Step 152–156 ใช้ pattern เดียวกันซ้ำๆ คือ "ส่ง object ที่มี method ตามที่ต้องการเข้าไป
> แทนการเจาะจง class ตายตัว" — นี่ไม่ใช่เรื่องบังเอิญ Ruby ทำให้ SOLID เขียนได้เป็น
> ธรรมชาติมากเพราะ duck typing (Step 151) คือกลไกที่ทำให้ "abstraction" ที่ SOLID พูดถึง
> ไม่จำเป็นต้องมีรูปแบบ syntax พิเศษ (เช่น `interface`, `abstract class`) เลยแม้แต่น้อย

---

## Step 157: Strategy Pattern แบบ Ruby — ใช้ Proc/Lambda และ object เล็กๆ แทนโครงสร้างหนักแบบ Java

### ทบทวน: Strategy Pattern คืออะไร

**Strategy Pattern** คือการห่อ "อัลกอริทึม" หรือ "พฤติกรรม" หนึ่งๆ ไว้เป็น object ที่
แลกเปลี่ยนกันได้ (interchangeable) แล้ว inject เข้าไปให้ object อื่นเรียกใช้ — จริงๆ แล้ว
เราเขียน Strategy Pattern ไปแล้วโดยไม่รู้ตัวถึง 2 ครั้งในหัวข้อก่อนหน้า:

- `RegularPricing`/`MemberPricing`/`VipPricing`/`GoldPricing` ใน Step 153 (OCP)
- `EmailSender`/`SmsSender` ที่ inject เข้า `OrderProcessor` ใน Step 156 (DIP)

ใน GoF (Gang of Four) design pattern ดั้งเดิม (ที่ออกแบบมาสำหรับภาษาอย่าง C++/Java)
Strategy Pattern ต้องมี: `Strategy` (abstract interface), `ConcreteStrategyA`,
`ConcreteStrategyB`, ... implement interface นั้น, และ `Context` ที่ถือ reference ไปยัง
`Strategy` แล้วเรียกผ่าน interface — เป็นโครงสร้างที่ค่อนข้างหนักเพราะภาษาพวกนั้นต้องการ
`interface`/`abstract class` ที่ประกาศไว้ล่วงหน้าเสมอ

**ใน Ruby เราไม่จำเป็นต้องมี "abstract Strategy class" เลย** เพราะ duck typing ทำให้แค่
มี object ที่ตอบสนอง method ชื่อเดียวกันก็ใช้แทนกันได้ทันที และในหลายกรณี **แค่ Proc หรือ
lambda ตัวเดียวก็เพียงพอ** โดยไม่ต้องสร้าง class ใหม่เลยด้วยซ้ำ

### รูปแบบที่ 1: Strategy เป็น class เล็กๆ ที่มี method `#call`

```ruby
# frozen_string_literal: true

class PercentageDiscount
  def initialize(percentage)
    @percentage = percentage
  end

  def call(price)
    price - (price * @percentage / 100.0)
  end
end

class FixedDiscount
  def initialize(amount)
    @amount = amount
  end

  def call(price)
    [price - @amount, 0].max
  end
end

class NoDiscount
  def call(price)
    price
  end
end

class PriceCalculator
  def initialize(discount_strategy: NoDiscount.new)
    @discount_strategy = discount_strategy
  end

  def final_price(price)
    @discount_strategy.call(price)
  end
end

regular = PriceCalculator.new
percentage = PriceCalculator.new(discount_strategy: PercentageDiscount.new(15))
fixed = PriceCalculator.new(discount_strategy: FixedDiscount.new(50))

puts regular.final_price(1000)      # => 1000
puts percentage.final_price(1000)   # => 850.0
puts fixed.final_price(1000)        # => 950
```

**อธิบาย:** ทุก strategy class (`PercentageDiscount`, `FixedDiscount`, `NoDiscount`)
implement method ชื่อเดียวกันคือ `call(price)` — ใช้ชื่อ `call` โดยเจตนา เพราะเป็นชื่อ
มาตรฐานที่ Ruby ใช้กับ object ที่ "เรียกได้" (`Proc`, `Lambda`, `Method` ทุกตัวมี `#call`)
ทำให้ `PriceCalculator` สามารถรับได้ทั้ง object ของ class เหล่านี้ **และ** lambda ธรรมดา
เข้าไปแทนกันได้ทันที ตามรูปแบบถัดไป

### รูปแบบที่ 2: Strategy เป็น Proc/Lambda ตรงๆ (ไม่ต้องสร้าง class เลย)

```ruby
# frozen_string_literal: true

class PriceCalculator
  def initialize(discount_strategy: ->(price) { price })
    @discount_strategy = discount_strategy
  end

  def final_price(price)
    @discount_strategy.call(price)
  end
end

no_discount = PriceCalculator.new
member = PriceCalculator.new(discount_strategy: ->(price) { price * 0.9 })
vip = PriceCalculator.new(discount_strategy: ->(price) { price * 0.8 })

puts no_discount.final_price(1000)   # => 1000
puts member.final_price(1000)        # => 900.0
puts vip.final_price(1000)           # => 800.0

# class PercentageDiscount เดิมจากรูปแบบที่ 1 (แสดงซ้ำเพื่อให้โค้ดนี้รันได้ครบในตัวเอง)
class PercentageDiscount
  def initialize(percentage)
    @percentage = percentage
  end

  def call(price)
    price - (price * @percentage / 100.0)
  end
end

# ผสม strategy class เข้ากับ lambda ได้ในที่เดียวกัน เพราะทั้งคู่ตอบสนอง #call เหมือนกัน
puts PriceCalculator.new(discount_strategy: PercentageDiscount.new(15)).final_price(1000)
# => 850.0
```

**อธิบาย:** `->(price) { price * 0.9 }` คือ lambda literal (เรียนจาก Part 008) ที่สร้าง
object ตอบสนอง `#call` ได้ทันที **โดยไม่ต้องประกาศ class ใหม่เลยสักตัว** สำหรับพฤติกรรม
สั้นๆ แบบนี้ การใช้ lambda ตรงๆ จึงกระชับกว่าการสร้าง class ทั้งกระบิ้งแบบภาษาอื่นมาก และ
นี่คือเหตุผลที่ Ruby "ไม่ต้องมี" GoF Strategy Pattern แบบเต็มรูปแบบเลยในหลายสถานการณ์ —
และเพราะทั้ง class เล็กๆ กับ lambda ต่างก็ตอบสนอง `#call` เหมือนกันทุกประการ จึงผสมใช้
งานแทนกันได้อย่างอิสระโดย `PriceCalculator` ไม่ต้องรู้เลยว่าได้รับ object ชนิดไหนมา

### รูปแบบที่ 3: เก็บ strategy ไว้ใน Hash เพื่อเลือกด้วย key (dispatch table)

```ruby
# frozen_string_literal: true

DISCOUNT_STRATEGIES = {
  regular: ->(price) { price },
  member: ->(price) { price * 0.9 },
  vip: ->(price) { price * 0.8 }
}.freeze

def final_price(price, tier)
  strategy = DISCOUNT_STRATEGIES.fetch(tier) { DISCOUNT_STRATEGIES[:regular] }
  strategy.call(price)
end

puts final_price(1000, :member)     # => 900.0
puts final_price(1000, :unknown)    # => 1000 (fallback ไปที่ :regular โดยอัตโนมัติ)
```

**อธิบาย:** รูปแบบนี้เหมาะกับกรณีที่ต้อง "เลือก" strategy จากค่าที่กำหนดตอนรัน (เช่น มาจาก
parameter, จากฐานข้อมูล) — `Hash#fetch` พร้อม block เป็นค่า default (เรียนจาก Part 005)
ทำให้ไม่ต้องเขียน `if/case` เพื่อเลือก strategy อีกต่อไป ยิ่งลดโครงสร้างที่ซับซ้อนของ GoF
Strategy Pattern ดั้งเดิมลงไปอีกขั้นหนึ่ง

> **สรุปเปรียบเทียบกับ Strategy Pattern แบบ Java/C++ ดั้งเดิม:**
>
> | | แบบ Java/C++ (GoF ดั้งเดิม) | แบบ Ruby (idiomatic) |
> |---|---|---|
> | นิยาม "สัญญา" ของ strategy | ต้องมี `interface Strategy` ประกาศไว้ก่อนเสมอ | ไม่ต้องมี — แค่ตอบสนอง `#call` (หรือชื่อ method ใดๆ ที่ตกลงกัน) ก็พอ |
> | Strategy ที่ทำอะไรง่ายๆ | ต้องสร้าง class ทั้งกระบิ้งเสมอ | ใช้ lambda บรรทัดเดียวได้ |
> | การเลือก strategy หลายตัว | มักใช้ Factory Pattern เพิ่มอีกชั้น | ใช้ Hash ธรรมดาเป็น dispatch table ได้เลย |
>
> ข้อคิดสำคัญ: **design pattern ไม่ใช่สูตรตายตัวที่ต้องเขียนตามโครงสร้างเป๊ะๆ** มันคือ
> "แนวคิด" ในการแก้ปัญหา ภาษาที่ยืดหยุ่นแบบ Ruby (มี first-class function ผ่าน
> Proc/Lambda, มี duck typing) มักทำให้ implement แนวคิดเดียวกันได้ **สั้นกว่าและเป็น
> ธรรมชาติกว่า** โดยไม่เสียความหมายของ pattern นั้นไปเลย

---

## Step 158: Observer Pattern — เขียนเองตั้งแต่ต้นด้วย module

### นิยาม

**Observer Pattern** คือรูปแบบที่ object หนึ่ง (**subject** หรือ **publisher**) เก็บ
รายชื่อ object อื่นๆ ที่ "สนใจติดตาม" (**observers** หรือ **subscribers**) ไว้ และเมื่อมี
เหตุการณ์บางอย่างเกิดขึ้นกับ subject มันจะ **แจ้งเตือน (notify)** observer ทุกตัวโดย
อัตโนมัติ — เป็นรูปแบบความสัมพันธ์แบบ **หนึ่งต่อกลาย (one-to-many)** ที่ subject
**ไม่จำเป็นต้องรู้จัก** ว่า observer แต่ละตัวจะเอาข้อมูลไปทำอะไรต่อ

### เขียน Observer Pattern เองตั้งแต่ต้น

```ruby
# frozen_string_literal: true

module Publisher
  def observers
    @observers ||= []
  end

  def add_observer(observer)
    observers << observer
  end

  def remove_observer(observer)
    observers.delete(observer)
  end

  def notify_observers(*args)
    observers.each { |observer| observer.update(*args) }
  end
end

class Stock
  include Publisher

  attr_reader :symbol, :price

  def initialize(symbol, price)
    @symbol = symbol
    @price = price
  end

  def price=(new_price)
    old_price = @price
    @price = new_price
    notify_observers(self, old_price, new_price) if old_price != new_price
  end
end

class PriceLogger
  def update(stock, old_price, new_price)
    puts "[LOG] #{stock.symbol}: #{old_price} -> #{new_price}"
  end
end

class PriceAlert
  def initialize(threshold)
    @threshold = threshold
  end

  def update(stock, old_price, new_price)
    return unless new_price > @threshold && old_price <= @threshold

    puts "[แจ้งเตือน] #{stock.symbol} ทะลุ #{@threshold} แล้ว (ราคาปัจจุบัน #{new_price})"
  end
end

stock = Stock.new("RUBY", 90)

stock.add_observer(PriceLogger.new)
stock.add_observer(PriceAlert.new(100))

stock.price = 95
# => [LOG] RUBY: 90 -> 95

stock.price = 105
# => [LOG] RUBY: 95 -> 105
# => [แจ้งเตือน] RUBY ทะลุ 100 แล้ว (ราคาปัจจุบัน 105)
```

**อธิบายทีละส่วน:**

1. `Publisher` เป็น module ที่ให้ความสามารถ "เป็น subject ที่แจ้งเตือนได้" กับ class
   ใดก็ตามที่ `include` มัน — `observers` เก็บ Array ของ observer ทั้งหมด, `add_observer`
   เพิ่มเข้าไป, `notify_observers` วนลูปเรียก `update` บน observer ทุกตัว
2. `Stock` include `Publisher` แล้วเรียก `notify_observers` เมื่อ `price=` ถูกเรียกและ
   ราคาเปลี่ยนไปจริง (ไม่ใช่ทุกครั้งที่เรียก setter เฉยๆ — ป้องกันการแจ้งเตือนซ้ำโดยไม่
   จำเป็น)
3. `PriceLogger` และ `PriceAlert` เป็น observer สองตัวที่**ไม่รู้จักกันเลย** และ `Stock`
   ก็**ไม่รู้จักว่า `PriceLogger`/`PriceAlert` คือ class อะไร** มันรู้แค่ว่า observer
   ทุกตัวต้องตอบสนอง `update(stock, old_price, new_price)` ได้ (duck typing อีกครั้ง —
   `Publisher` ไม่เคยเช็ค `is_a?` เลยสักครั้ง)
4. เพิ่ม observer ใหม่ในอนาคต (เช่น `EmailNotifier` ที่ส่งอีเมลเมื่อราคาตก) ทำได้โดย
   **สร้าง class ใหม่แล้ว `add_observer` เข้าไป** โดยไม่ต้องแก้ `Stock` หรือ observer
   ตัวอื่นเลย — ตรงตาม Open/Closed Principle (Step 153) ไปพร้อมกันด้วย

> **จุดแข็งที่สำคัญที่สุดของ Observer Pattern:** มัน **ลดการผูกติดกัน (decoupling)**
> ระหว่าง "สิ่งที่เกิดเหตุการณ์" (`Stock`) กับ "สิ่งที่ต้องทำเมื่อเหตุการณ์เกิดขึ้น"
> (`PriceLogger`, `PriceAlert`) — `Stock` ไม่จำเป็นต้องรู้ (และไม่ควรรู้) ว่าใครกำลัง
> ติดตามมันอยู่บ้าง หรือพวกเขาจะเอาข้อมูลไปทำอะไรต่อ นี่คือหลักการเดียวกับที่ Rails ใช้ใน
> ActiveRecord callback (`after_create`, `after_save` ที่จะเรียนใน Part 026) และ
> ActiveSupport::Notifications (ระบบ event ภายใน Rails เอง)

---

## Step 159: Observer Pattern ด้วย `Observable` module ของ Ruby standard library

Ruby standard library มี module ชื่อ `Observable` ที่ทำหน้าที่เหมือน `Publisher` ที่เรา
เขียนเองใน Step 158 มาให้พร้อมใช้งานอยู่แล้ว ต้อง `require "observer"` ก่อนเสมอ (เป็น
standard library ที่ไม่ได้โหลดมาให้อัตโนมัติ เหมือนกับ `require "date"`/`"json"` ที่เจอ
มาแล้วในหลาย Part ก่อนหน้า)

```ruby
# frozen_string_literal: true

require "observer"

class StockTicker
  include Observable

  attr_reader :symbol, :price

  def initialize(symbol, price)
    @symbol = symbol
    @price = price
  end

  def price=(new_price)
    old_price = @price
    @price = new_price
    return unless old_price != new_price

    changed              # (1) ต้องเรียกก่อนเสมอ เพื่อบอกว่า "มีการเปลี่ยนแปลงเกิดขึ้นแล้ว"
    notify_observers(self, old_price, new_price)   # (2) แจ้งเตือน observer ทุกตัว
  end
end

class PriceLogger
  def update(ticker, old_price, new_price)
    puts "[LOG] #{ticker.symbol}: #{old_price} -> #{new_price}"
  end
end

ticker = StockTicker.new("RUBY", 100)
logger = PriceLogger.new

ticker.add_observer(logger)
ticker.price = 105
# => [LOG] RUBY: 100 -> 105
```

**อธิบาย method ที่ `Observable` มีให้:**

- `add_observer(observer, func = :update)` — เพิ่ม observer เข้าไปในรายการ (พารามิเตอร์
  ตัวที่สองกำหนดชื่อ method ที่จะถูกเรียกได้ด้วย ถ้าไม่อยากใช้ชื่อ `update` default)
- `delete_observer(observer)` / `delete_observers` — เอา observer ออกทีละตัว/ทั้งหมด
- `count_observers` — นับจำนวน observer ปัจจุบัน
- **`changed(state = true)`** — ตั้ง flag ภายในว่า "state เปลี่ยนแปลงแล้ว" **ต้องเรียก
  ก่อน `notify_observers` เสมอ** มิเช่นนั้น `notify_observers` จะ**ไม่ทำอะไรเลย** (design
  นี้จงใจให้ยืดหยุ่น เผื่อกรณีที่ subject มีการเปลี่ยนแปลงภายในหลายจุด แต่อยากรวมเป็นการ
  แจ้งเตือนครั้งเดียวตอนท้ายเท่านั้น)
- **`notify_observers(*args)`** — เรียก method `update` (หรือชื่อที่กำหนดตอน
  `add_observer`) ของทุก observer พร้อมส่ง `*args` ต่อให้ **แต่ทำงานก็ต่อเมื่อเรียก
  `changed` มาก่อนแล้วเท่านั้น**

### พิสูจน์ว่าลืมเรียก `changed` แล้วจะเกิดอะไรขึ้น

```ruby
require "observer"

class Broken
  include Observable

  def trigger
    notify_observers("บางอย่างเกิดขึ้น")   # ลืมเรียก changed ก่อน!
  end
end

class Listener
  def update(message)
    puts "ได้รับ: #{message}"
  end
end

broken = Broken.new
broken.add_observer(Listener.new)
broken.trigger
# (ไม่มีอะไรถูกพิมพ์ออกมาเลย เพราะไม่เคยเรียก changed มาก่อน notify_observers)
```

> **ข้อควรจำที่สำคัญที่สุดของ `Observable`:** `changed` ต้องถูกเรียกก่อน
> `notify_observers` **ทุกครั้ง** ไม่เช่นนั้นการแจ้งเตือนจะไม่เกิดขึ้นเลยโดยไม่มี error
> ใดๆ แจ้งให้ทราบ (เป็นบัคเงียบที่ debug ยากพอสมควรถ้าไม่รู้กลไกนี้มาก่อน)

### เปรียบเทียบ `Publisher` (เขียนเอง) กับ `Observable` (stdlib)

| | `Publisher` (Step 158, เขียนเอง) | `Observable` (stdlib) |
|---|---|---|
| ต้อง `require` ไหม | ไม่ต้อง (เป็น module ที่เราเขียนเอง) | ต้อง `require "observer"` |
| แจ้งเตือนทันทีไหม | ทันที เมื่อเรียก `notify_observers` | ต้องเรียก `changed` ก่อนเสมอ ไม่งั้นเงียบ |
| ปรับแต่งชื่อ method ที่ observer ต้อง implement | ต้องแก้โค้ด `Publisher` เอง | มี parameter `func` ใน `add_observer` ให้ปรับได้ทันที |
| เหมาะกับ | เข้าใจกลไกอย่างละเอียด, ปรับแต่ง behavior เฉพาะทาง | งานทั่วไปที่ต้องการ Observer Pattern มาตรฐานเร็วๆ โดยไม่อยากเขียนเอง |

> **แนวปฏิบัติจริง:** ในโค้ด production ส่วนใหญ่ที่ไม่ใช่ Rails (ซึ่งมีระบบ callback/
> notification ของตัวเอง) การเข้าใจกลไกของ `Publisher` ที่เขียนเองใน Step 158 มีค่ามาก
> เพราะช่วยให้อ่านโค้ดที่ใช้ Observer Pattern แบบใดก็ได้ออกทันที ส่วน `Observable` จาก
> stdlib ก็มีไว้ใช้งานจริงเมื่อไม่อยากเขียนกลไกพื้นฐานซ้ำเอง — ทั้งสองแนวทางมีที่ใช้
> ต่างกันไปตามบริบท

---

## Step 160: แบบฝึกหัด — ระบบแจ้งเตือนที่รวม Strategy (ช่องทางส่ง) และ Observer (เหตุการณ์)

### โจทย์

สร้างไฟล์ `notification_system.rb` ที่จำลอง**ระบบแจ้งเตือนของร้านค้าออนไลน์** โดยรวม
สอง pattern ที่เรียนมาใน Part นี้เข้าด้วยกัน:

1. **Strategy Pattern** สำหรับ**ช่องทางการส่ง** (delivery channel) — ต้องมีอย่างน้อย
   3 ช่องทาง: อีเมล (`EmailChannel`), SMS (`SmsChannel`), push notification
   (`PushChannel`) โดยแต่ละ channel ต้องตอบสนอง method `deliver(recipient, message)`
   เหมือนกัน (duck typing) เพื่อให้สลับ/เพิ่มช่องทางใหม่ได้โดยไม่ต้องแก้โค้ดเดิม (OCP)
2. **Observer Pattern** สำหรับ**เหตุการณ์ของคำสั่งซื้อ** — `OrderSystem` ต้องเป็น
   subject ที่แจ้งเตือน observer เมื่อมีเหตุการณ์เกิดขึ้น อย่างน้อย 2 เหตุการณ์:
   `order_placed` (สั่งซื้อสำเร็จ) และ `order_shipped` (จัดส่งแล้ว)
3. `NotificationCenter` ทำหน้าที่เป็น **observer** ตัวหนึ่งของ `OrderSystem` และเมื่อ
   ได้รับการแจ้งเตือน ให้ส่งข้อความผ่านทุก channel ที่มันถูก inject เข้ามาตอนสร้าง
   (dependency injection ตามหลัก DIP — ห้าม hardcode ชื่อ class ของ channel ไว้ข้างใน)
4. ต้องสามารถเพิ่ม observer ตัวอื่นที่ไม่เกี่ยวกับการแจ้งเตือนลูกค้าเลย (เช่น บันทึก log)
   เข้าไปพร้อมกันได้ โดยไม่กระทบการทำงานของ `NotificationCenter`

### เฉลย

```ruby
# frozen_string_literal: true

# notification_system.rb

# ============================================================
# ส่วนที่ 1: Strategy — ช่องทางการส่งข้อความ
# แต่ละ channel ตอบสนอง #deliver(recipient, message) เหมือนกัน (duck typing)
# เพิ่ม channel ใหม่ได้เสมอโดยไม่ต้องแก้โค้ดที่มีอยู่แล้วเลย (OCP)
# ============================================================

class EmailChannel
  def deliver(recipient, message)
    puts "[EMAIL] ถึง #{recipient[:email]}: #{message}"
  end
end

class SmsChannel
  def deliver(recipient, message)
    puts "[SMS] ถึง #{recipient[:phone]}: #{message}"
  end
end

class PushChannel
  def deliver(recipient, message)
    puts "[PUSH] ถึงอุปกรณ์ #{recipient[:device_id]}: #{message}"
  end
end

# ============================================================
# ส่วนที่ 2: Observer — กลไกแจ้งเตือนเหตุการณ์ (เขียนเองแบบ Step 158)
# ============================================================

module Publisher
  def observers
    @observers ||= []
  end

  def add_observer(observer)
    observers << observer
  end

  def remove_observer(observer)
    observers.delete(observer)
  end

  def notify_observers(*args)
    observers.each { |observer| observer.update(*args) }
  end
end

class OrderSystem
  include Publisher

  def place_order(order)
    puts "รับคำสั่งซื้อ ##{order[:id]} เรียบร้อย"
    notify_observers(:order_placed, order)
  end

  def ship_order(order)
    puts "จัดส่งคำสั่งซื้อ ##{order[:id]} แล้ว"
    notify_observers(:order_shipped, order)
  end
end

# ============================================================
# ส่วนที่ 3: NotificationCenter — observer ที่รวม Strategy เข้ากับ Observer
# รับ channel ทั้งหมดผ่าน constructor (dependency injection ตามหลัก DIP)
# ============================================================

class NotificationCenter
  EVENT_MESSAGES = {
    order_placed: ->(order) { "คำสั่งซื้อ ##{order[:id]} ของคุณได้รับการยืนยันแล้ว" },
    order_shipped: ->(order) { "คำสั่งซื้อ ##{order[:id]} กำลังจัดส่งถึงคุณ" }
  }.freeze

  def initialize(channels:)
    @channels = channels   # array ของ object ใดๆ ที่ตอบสนอง #deliver(recipient, message)
  end

  # method นี้คือสิ่งที่ OrderSystem (subject) จะเรียกผ่าน notify_observers
  def update(event_name, order)
    message_builder = EVENT_MESSAGES[event_name]
    return unless message_builder   # เมินเฉยต่อ event ที่เราไม่รู้จัก แทนที่จะ error

    message = message_builder.call(order)
    @channels.each { |channel| channel.deliver(order[:customer], message) }
  end
end

# ============================================================
# ส่วนที่ 4 (เสริม): observer อีกตัวที่ไม่เกี่ยวกับ NotificationCenter เลย
# แสดงว่า OrderSystem รองรับ observer หลายตัวพร้อมกันได้ (one-to-many)
# ============================================================

class AuditLogObserver
  def update(event_name, order)
    puts "[AUDIT LOG] เหตุการณ์ #{event_name} เกิดกับคำสั่งซื้อ ##{order[:id]}"
  end
end

# ============================================================
# ทดลองใช้งาน
# ============================================================

order_system = OrderSystem.new

notification_center = NotificationCenter.new(
  channels: [EmailChannel.new, SmsChannel.new]
)

order_system.add_observer(notification_center)
order_system.add_observer(AuditLogObserver.new)

order = {
  id: 1001,
  customer: {
    email: "manee@example.com",
    phone: "0812345678",
    device_id: "dev-42"
  }
}

puts "--- place_order ---"
order_system.place_order(order)
# => รับคำสั่งซื้อ #1001 เรียบร้อย
# => [EMAIL] ถึง manee@example.com: คำสั่งซื้อ #1001 ของคุณได้รับการยืนยันแล้ว
# => [SMS] ถึง 0812345678: คำสั่งซื้อ #1001 ของคุณได้รับการยืนยันแล้ว
# => [AUDIT LOG] เหตุการณ์ order_placed เกิดกับคำสั่งซื้อ #1001

puts "\n--- ship_order ---"
order_system.ship_order(order)
# => จัดส่งคำสั่งซื้อ #1001 แล้ว
# => [EMAIL] ถึง manee@example.com: คำสั่งซื้อ #1001 กำลังจัดส่งถึงคุณ
# => [SMS] ถึง 0812345678: คำสั่งซื้อ #1001 กำลังจัดส่งถึงคุณ
# => [AUDIT LOG] เหตุการณ์ order_shipped เกิดกับคำสั่งซื้อ #1001

puts "\n--- เปลี่ยนช่องทางแจ้งเตือนโดยไม่แก้โค้ดเดิมเลย ---"
push_only_center = NotificationCenter.new(channels: [PushChannel.new])
order_system.remove_observer(notification_center)
order_system.add_observer(push_only_center)
order_system.ship_order(order)
# => จัดส่งคำสั่งซื้อ #1001 แล้ว
# => [PUSH] ถึงอุปกรณ์ dev-42: คำสั่งซื้อ #1001 กำลังจัดส่งถึงคุณ
# => [AUDIT LOG] เหตุการณ์ order_shipped เกิดกับคำสั่งซื้อ #1001
```

ทดสอบรัน:

```bash
ruby notification_system.rb
```

### สิ่งที่ได้ฝึกจากเฉลยนี้ (เชื่อมโยงกับทุกหลักการใน Part นี้)

- **Duck typing (Step 151):** `NotificationCenter#update` เรียก `channel.deliver` โดย
  ไม่สนใจว่า `channel` เป็น `EmailChannel`, `SmsChannel`, หรือ `PushChannel` — สนใจแค่ว่า
  มี method `deliver` ให้เรียก
- **SRP (Step 152):** แต่ละ channel รับผิดชอบแค่ "วิธีส่งข้อความ" ของตัวเอง,
  `OrderSystem` รับผิดชอบแค่ "ตรรกะของคำสั่งซื้อและการแจ้งเหตุการณ์", `NotificationCenter`
  รับผิดชอบแค่ "แปลง event เป็นข้อความแล้วกระจายไปตาม channel"
- **OCP (Step 153):** เพิ่ม channel ใหม่ (เช่น `SlackChannel`) หรือ observer ใหม่ (เช่น
  `AuditLogObserver`) ได้โดยไม่ต้องแก้ `OrderSystem` หรือ `NotificationCenter` เลย
- **ISP (Step 155):** channel แต่ละตัวมี method เดียวที่ต้อง implement (`deliver`) ไม่มี
  ใครถูกบังคับให้ implement method ที่ตัวเองไม่เกี่ยวข้อง
- **DIP (Step 156):** `NotificationCenter` รับ `channels:` เข้ามาทาง constructor แทนการ
  `new` channel ขึ้นมาเองข้างใน ทำให้ทดสอบและสลับช่องทางได้อย่างอิสระ (เห็นได้จากตัวอย่าง
  `push_only_center` ที่สลับช่องทางได้โดยไม่แตะ `OrderSystem` เลย)
- **Strategy Pattern (Step 157):** channel ทั้งสามคือ strategy ที่แลกเปลี่ยนกันได้,
  `EVENT_MESSAGES` เองก็เป็น dispatch table ของ lambda อีกชั้นหนึ่งซ้อนอยู่ข้างใน
- **Observer Pattern (Step 158–159):** `OrderSystem` เป็น subject ที่ไม่รู้จัก
  `NotificationCenter`/`AuditLogObserver` เป็นการเฉพาะเจาะจงเลย รู้แค่ว่า observer ทุกตัว
  ต้องมี `update`

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม channel ใหม่ชื่อ `SlackChannel` ที่ deliver ข้อความไปยัง Slack (จำลองด้วย
   `puts "[SLACK] ..."` ก็พอ ไม่ต้องเรียก API จริง) แล้วส่งเข้าไปใน `channels:` ของ
   `NotificationCenter` ตัวใหม่ **โดยห้ามแก้โค้ดของ `OrderSystem`, `NotificationCenter`,
   หรือ channel เดิมทั้งสามตัวแม้แต่บรรทัดเดียว** — ถ้าทำสำเร็จแปลว่าเข้าใจ OCP จริง

2. เพิ่ม event ใหม่ชื่อ `order_cancelled` พร้อม method `cancel_order(order)` ใน
   `OrderSystem` และเพิ่มข้อความที่เหมาะสมลงใน `EVENT_MESSAGES` ของ
   `NotificationCenter` — ทดสอบว่าทั้ง `NotificationCenter` และ `AuditLogObserver`
   ตอบสนองต่อเหตุการณ์ใหม่นี้ได้โดยอัตโนมัติหรือไม่ (ใบ้: `AuditLogObserver` ควรทำงานได้
   ทันทีโดยไม่ต้องแก้เลย เพราะมันรับ `event_name` แบบทั่วไป)

3. เขียน `RetryChannel` ที่ **ห่อ (wrap)** channel อื่นไว้ข้างใน (รับ channel ตัวจริงเข้า
   มาทาง constructor เหมือนกับที่ `NotificationCenter` รับ channels) แล้วจำลองการส่งที่
   ล้มเหลวแบบสุ่มด้วย `rand < 0.5` — ถ้าส่งไม่สำเร็จให้ลองส่งซ้ำสูงสุด 3 ครั้งก่อนจะยอมแพ้
   และพิมพ์ข้อความแจ้งว่าส่งไม่สำเร็จ ทดสอบว่าใช้แทน `EmailChannel`/`SmsChannel` ตรงๆ ใน
   `NotificationCenter` ได้โดยไม่ต้องแก้ `NotificationCenter` เลย (ใบ้: `RetryChannel`
   ต้อง implement `deliver(recipient, message)` เหมือน channel อื่นทุกประการ เพื่อรักษา
   duck typing — นี่คือแนวคิดเดียวกับ Decorator Pattern ที่จะเจอเพิ่มเติมใน Part 083)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจ **duck typing** อย่างลึกซึ้ง ("ถ้ามันร้องเหมือนเป็ด มันก็คือเป็ด") และเห็นว่า
  `Enumerable`/`Comparable` ของ Ruby เองก็สร้างขึ้นบนหลักการนี้ทั้งหมด ผ่าน protocol
  โดยนัยอย่าง `each`, `<=>`, `to_s`
- เข้าใจภาพรวมของ **SOLID** ทั้ง 5 ข้อ และรู้ว่าทำไม Ruby (ด้วย duck typing) ทำให้นำ
  หลักการเหล่านี้มาใช้ได้เป็นธรรมชาติกว่าภาษาที่ต้องมี `interface`/`abstract class`
  บังคับไว้ก่อน
- **SRP:** แยก class ตาม "เหตุผลให้เปลี่ยน" ผ่านตัวอย่าง refactor `InvoiceReport` ที่ทำ
  หลายหน้าที่ปนกัน ให้กลายเป็น `Invoice`/`InvoiceTextFormatter`/`InvoiceFileWriter`/
  `InvoiceMailer` ที่แต่ละตัวโฟกัสเรื่องเดียว
- **OCP:** เปลี่ยนจาก `case/when` ที่ต้องแก้โค้ดเดิมทุกครั้งที่มีกรณีใหม่ ไปเป็น
  polymorphism ที่เพิ่มพฤติกรรมใหม่ได้โดยสร้าง class ใหม่อย่างเดียว
- **LSP:** เข้าใจปัญหาคลาสสิกของ `Square < Rectangle` ที่ดูสมเหตุสมผลทางคณิตศาสตร์แต่
  ละเมิดสัญญาทางพฤติกรรม และรู้จักใช้ composition/duck typing แทน inheritance เมื่อ
  พฤติกรรมไม่ตรงกันจริง
- **ISP:** รู้ว่า Ruby ไม่มี `interface` แต่บรรลุเป้าหมายเดียวกันด้วยการแยก module ให้เล็ก
  และโฟกัสเฉพาะเรื่อง (`Printable`, `Scannable`, `Faxable`) แทนการรวมทุกอย่างไว้ที่เดียว
- **DIP:** inject dependency ผ่าน constructor (`initialize(notifier: EmailSender.new)`)
  แทนการ hardcode class ไว้ข้างใน ทำให้สลับพฤติกรรมและทดสอบด้วย fake object ได้ง่ายขึ้น
  มาก
- **Strategy Pattern:** implement แบบ idiomatic Ruby ด้วย class เล็กที่มี `#call`,
  ด้วย Proc/Lambda ตรงๆ, และด้วย Hash เป็น dispatch table — เห็นว่าโครงสร้างหนักแบบ
  GoF ดั้งเดิมมักไม่จำเป็นในภาษาที่มี first-class function
- **Observer Pattern:** เขียน `Publisher` module เองตั้งแต่ต้น และใช้ `Observable`
  module ของ standard library (`require "observer"`) พร้อมเข้าใจกฎสำคัญที่ต้องเรียก
  `changed` ก่อน `notify_observers` เสมอ
- ประกอบทุกหลักการเข้าด้วยกันในโปรเจกต์ **ระบบแจ้งเตือน** ที่ใช้ Strategy สำหรับช่องทาง
  การส่ง (email/SMS/push) และ Observer สำหรับเหตุการณ์ของคำสั่งซื้อ พร้อมเห็นว่าเพิ่ม
  ช่องทาง/เหตุการณ์ใหม่ได้โดยไม่ต้องแก้โค้ดเดิมเลย

**ต่อไป (Part 017):** เราจะเรียนเรื่อง **Gem คืออะไร, Bundler, Gemfile เชิงลึก, และการ
สร้าง gem ของตัวเอง (เบื้องต้น)** — นำหลักการออกแบบซอฟต์แวร์ที่เพิ่งเรียนใน Part นี้
(โดยเฉพาะ SRP และ DIP) ไปใช้จริงตอนออกแบบโครงสร้างของ gem ที่แจกจ่ายให้คนอื่นใช้งานได้
พร้อมเข้าใจว่าเบื้องหลัง gem ที่เราติดตั้งมาตั้งแต่ Part 001 (`bundler`, `rspec`, `pry`)
ถูกสร้างและ publish ขึ้นมาอย่างไร
