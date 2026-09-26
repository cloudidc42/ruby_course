# Part 014: Comparable เจาะลึก, Struct, OpenStruct, และ Data class (Ruby 3.2+)

> **Step ครอบคลุมใน Part นี้:** Step 131–140
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 011–013 มาก่อน โดยเฉพาะ Part 010 Step 99 เรื่อง Comparable เบื้องต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6 — หัวข้อ `Data.define`
> ต้องการ **Ruby >= 3.2** เท่านั้น ถ้าใช้ Ruby เวอร์ชันเก่ากว่านี้จะเจอ `NameError: uninitialized
> constant Data` หรือ `NoMethodError: undefined method 'define' for Data`)

Part ที่แล้วเราเจาะลึก `Enumerable` เพื่อประมวลผล collection ได้อย่างมืออาชีพ Part นี้จะเปลี่ยน
โฟกัสไปที่การ**ออกแบบโครงสร้างข้อมูล**เอง — ตั้งแต่การใช้ `Comparable` ให้เต็มศักยภาพ ไปจนถึง
เครื่องมือ 3 ตัวที่ Ruby มีให้สำหรับสร้าง "value object" แบบรวดเร็ว โดยไม่ต้องเขียน
`class ... initialize ... attr_reader` เองทุกครั้ง: `Struct`, `OpenStruct`, และตัวใหม่ล่าสุด
`Data` (Ruby 3.2+) เราจะเปรียบเทียบทั้งสามตัวอย่างละเอียด เพื่อให้เลือกใช้ถูกต้องในงานจริง

## สารบัญของ Part นี้

- Step 131: `Comparable` เจาะลึก — method ที่ได้มาฟรีทั้งหมดจากการนิยาม `<=>` เดียว
- Step 132: `Struct.new` แบบ positional — สร้าง value object แบบรวดเร็ว
- Step 133: `Struct.new` แบบ `keyword_init: true` และการผสมทั้งสองแบบใน Ruby 3.2+
- Step 134: เพิ่ม custom method ให้ `Struct` ด้วย block และการ subclass
- Step 135: `Struct` vs Custom Class เต็มรูปแบบ — เมื่อไหร่ควรใช้อะไร
- Step 136: `OpenStruct` — dynamic attribute แบบไม่ต้องประกาศ field ล่วงหน้า
- Step 137: ข้อเสียของ `OpenStruct` — ไม่ป้องกัน typo, ช้ากว่า, และความเสี่ยงด้านความปลอดภัย
- Step 138: `Data.define` (Ruby 3.2+) — Value Object แบบ Immutable ยุคใหม่
- Step 139: ตารางเปรียบเทียบ `Struct` vs `Data` vs `OpenStruct` vs `Hash`
- Step 140: แบบฝึกหัด — สร้าง `Point` และ `Money` ด้วย `Data.define` พร้อม operator overloading

---

## Step 131: `Comparable` เจาะลึก — method ที่ได้มาฟรีทั้งหมดจากการนิยาม `<=>` เดียว

ใน **Part 010 Step 99** เราแนะนำ `Comparable` ไปแล้วว่า แค่นิยาม `<=>` (spaceship operator)
ตัวเดียวใน class ของเราเอง แล้ว `include Comparable` ก็จะปลดล็อก method เปรียบเทียบอีกหลายตัว
ให้ใช้ฟรีทันที Part นี้จะขยายความละเอียดว่า "หลายตัว" ที่ว่านั้นมีอะไรบ้าง แต่ละตัวมี
พฤติกรรมยังไง และมีข้อควรระวังอะไรบ้างที่มือใหม่มักพลาด

### รายชื่อ method ทั้งหมดที่ `Comparable` มอบให้

| Method | ความหมาย |
|---|---|
| `<` | น้อยกว่า |
| `<=` | น้อยกว่าหรือเท่ากับ |
| `==` | เท่ากับ (เปรียบเทียบผ่าน `<=>` ไม่ใช่ `Object#==` เดิม) |
| `>` | มากกว่า |
| `>=` | มากกว่าหรือเท่ากับ |
| `between?(min, max)` | อยู่ระหว่าง `min` และ `max` (รวมขอบทั้งสองด้าน) หรือไม่ |
| `clamp(min, max)` | "หนีบ" ค่าให้อยู่ในช่วง `min..max` — ถ้าเกินขอบจะถูกปรับให้เท่ากับขอบนั้น |
| `clamp(range)` | เหมือนด้านบนแต่รับเป็น `Range` โดยตรง (`min..max` หรือเปิดขอบด้วย `..max`/`min..`) |

ทุก method ในตารางนี้ทำงานโดยเรียก `<=>` ของเราเบื้องหลังเท่านั้น — Ruby ไม่รู้อะไรเลยเกี่ยวกับ
"ความหมาย" ของ object เรา มันแค่ถาม `<=>` ซ้ำๆ ตามกฎที่กำหนดไว้แล้วแปลผลลัพธ์ (`-1`, `0`, `1`)
เป็น method เปรียบเทียบที่เหมาะสม

```ruby
# frozen_string_literal: true

class Temperature
  include Comparable

  attr_reader :celsius

  def initialize(celsius)
    @celsius = celsius
  end

  def <=>(other)
    celsius <=> other.celsius
  end

  def to_s
    "#{celsius}°C"
  end
end

cold = Temperature.new(5)
mild = Temperature.new(20)
hot = Temperature.new(35)

puts cold < mild                    # => true
puts hot >= mild                    # => true
puts mild == Temperature.new(20)    # => true
puts mild.between?(cold, hot)       # => true
puts Temperature.new(-5).between?(cold, hot) # => false

# clamp: หนีบค่าให้อยู่ในช่วงที่กำหนด
puts Temperature.new(50).clamp(cold, hot)   # => 35°C  (เกินขอบบน ถูกหนีบเป็น hot)
puts Temperature.new(-10).clamp(cold, hot)  # => 5°C   (ต่ำกว่าขอบล่าง ถูกหนีบเป็น cold)
puts Temperature.new(20).clamp(cold, hot)   # => 20°C  (อยู่ในช่วงอยู่แล้ว ไม่เปลี่ยน)

# clamp แบบรับ Range โดยตรง (สะดวกกว่าเมื่อ min/max เป็น object เดียวกันเสมอ)
puts Temperature.new(50).clamp(cold..hot)   # => 35°C
```

**อธิบาย:**

- `clamp` ต่างจาก `between?` ตรงที่ `between?` แค่**ตอบคำถาม** (true/false) ส่วน `clamp`
  **คืนค่าใหม่** ที่ถูกปรับให้อยู่ในขอบเขตแล้ว — ใช้บ่อยมากในสถานการณ์อย่างการจำกัดค่า
  volume, opacity, หรือ rating ให้ไม่หลุดขอบที่กำหนด
- `clamp(range)` (รับ `Range` ตัวเดียว) เป็น syntax ที่กระชับกว่าและอ่านง่ายกว่าเมื่อมีตัวแปร
  Range อยู่แล้ว รองรับ Range ที่เปิดขอบด้านเดียวได้ด้วย เช่น `clamp(..hot)` (ไม่จำกัดขอบล่าง)
  หรือ `clamp(cold..)` (ไม่จำกัดขอบบน)

### ข้อควรระวัง: `Comparable#==` ไม่ได้แปลว่า `eql?`/`hash` ถูกปรับตามไปด้วย

จุดที่มือใหม่มักพลาดคือ เมื่อ `include Comparable` แล้ว `==` จะถูกโอเวอร์ไรด์ให้เทียบผ่าน `<=>`
ทันที **แต่ `eql?` และ `hash` (ที่ Ruby ใช้ตอนหา key ใน `Hash` หรือ item ใน `Set`) จะไม่ถูก
แก้ตามให้อัตโนมัติ** — ยังคงเป็นพฤติกรรมเดิมจาก `Object` ที่เทียบด้วย `object_id`

```ruby
t1 = Temperature.new(20)
t2 = Temperature.new(20)

puts t1 == t2        # => true  (Comparable#== เทียบผ่าน <=>)
puts t1.eql?(t2)      # => false (Object#eql? เดิม เทียบว่าเป็น object เดียวกันหรือไม่)

require "set"
temps = Set.new
temps << t1
temps << t2
puts temps.size       # => 2  (ไม่ใช่ 1! เพราะ Set ใช้ eql?/hash ไม่ใช่ ==)
```

> **แนวคิดสำคัญ:** ถ้าต้องการให้ object ของเราถือว่า "เท่ากัน" ทั้งตอนใช้ `==` **และ**
> ตอนใช้เป็น key ของ `Hash`/item ของ `Set` ต้องนิยาม `eql?` และ `hash` เพิ่มเองอย่างชัดเจน
> (มักเขียนคู่กันเป็น `def eql?(other) = self == other` และ `def hash = celsius.hash`) —
> `Comparable` แก้ปัญหาแค่การ**เรียงลำดับและเปรียบเทียบ** ไม่ได้แก้ปัญหาการเป็น **hash key**
> ให้ ซึ่งเป็นคนละกลไกกันโดยสิ้นเชิงใน Ruby

```ruby
class Temperature
  def eql?(other)
    other.is_a?(Temperature) && self == other
  end

  def hash
    celsius.hash
  end
end

temps = Set.new([Temperature.new(20), Temperature.new(20)])
puts temps.size  # => 1 (ตอนนี้ถือว่าเป็นตัวเดียวกันแล้ว)
```

---

## Step 132: `Struct.new` แบบ positional — สร้าง value object แบบรวดเร็ว

หลายครั้งเราต้องการ class เล็กๆ ที่แค่ "ห่อ" ข้อมูลไม่กี่ฟิลด์เข้าด้วยกัน (เช่น พิกัด x, y
หรือคู่ชื่อ-นามสกุล) โดยไม่ต้องมี logic ซับซ้อนอะไรเลย การเขียน `class` เต็มรูปแบบพร้อม
`attr_accessor`, `initialize` ทุกครั้งจะดูเวิ่นเว้อเกินไป **`Struct`** คือเครื่องมือของ Ruby
ที่สร้าง class แบบนี้ให้ในบรรทัดเดียว

```ruby
# frozen_string_literal: true

Point = Struct.new(:x, :y)

p1 = Point.new(1, 2)

puts p1.x        # => 1
puts p1.y        # => 2
puts p1[0]        # => 1  (เข้าถึงแบบ index ได้ด้วย เหมือน Array)
puts p1[:x]       # => 1  (เข้าถึงแบบ key ได้ด้วย เหมือน Hash)

puts p1.to_a.inspect  # => [1, 2]
puts p1.to_h.inspect  # => {x: 1, y: 2}
puts p1.members.inspect # => [:x, :y]

p2 = Point.new(1, 2)
puts p1 == p2     # => true (Struct เปรียบเทียบ "ค่าภายในทั้งหมด" ให้อัตโนมัติ ไม่ใช่ object_id)
```

**อธิบาย:**

- `Struct.new(:x, :y)` คืน **class ใหม่** ที่มี `attr_accessor :x` และ `attr_accessor :y`
  รวมถึง `initialize(x, y)` ให้ครบโดยอัตโนมัติ — เก็บ class นี้ไว้ในตัวแปร (มักตั้งชื่อขึ้นต้น
  ด้วยตัวใหญ่อย่าง `Point` เพื่อให้ใช้งานเหมือน class ปกติทุกประการ)
- เข้าถึงค่าได้ 3 ทาง: ผ่านชื่อ method (`p1.x`), ผ่าน index แบบ Array (`p1[0]`), และผ่าน key
  แบบ Hash (`p1[:x]`) — ยืดหยุ่นกว่า Hash ธรรมดาเพราะมี dot notation ให้ด้วย
- `to_a`/`to_h`/`members` ติดมาให้ฟรี ทำให้แปลงไปมาระหว่าง Struct, Array, Hash ได้สะดวก
- `Struct` override `==` ให้เปรียบเทียบตามค่าภายในทั้งหมด (เหมือนที่ `Comparable` ทำกับ
  `<=>` แต่ `Struct` ทำเรื่องนี้ให้ตั้งแต่แรกโดยไม่ต้องขอ) — ต่างจาก `class` ปกติที่ถ้าไม่
  override `==` เอง จะเทียบแบบ `object_id` เท่านั้น

### `Struct` เป็นแบบ mutable (แก้ค่าได้หลังสร้าง)

```ruby
p1.x = 100
puts p1.x   # => 100
puts p1     # => #<struct Point x=100, y=2>
```

ต่างจาก `Data` ที่จะเรียนใน Step 138 ซึ่งออกแบบมาให้ **immutable** โดยเฉพาะ — นี่คือความต่าง
สำคัญที่สุดข้อหนึ่งระหว่างสองเครื่องมือนี้

### `Struct` ยัง `include Enumerable` มาให้ด้วย

```ruby
puts p1.map { |v| v * 2 }.inspect  # => [200, 4]  (each field ถูกวนด้วย Enumerable methods ได้)
```

---

## Step 133: `Struct.new` แบบ `keyword_init: true` และการผสมทั้งสองแบบใน Ruby 3.2+

การสร้าง `Point.new(1, 2)` แบบ positional มีปัญหาเดียวกับ method ที่มี argument เยอะๆ ที่เคย
เจอใน Part 007 — เมื่อฟิลด์มากกว่า 2-3 ตัว จะสับสนได้ง่ายว่าตัวไหนคือตัวไหน `Struct` จึงมี
ตัวเลือก `keyword_init: true` ให้บังคับสร้างด้วย keyword argument แทน

```ruby
# frozen_string_literal: true

Address = Struct.new(:street, :city, :zip_code, keyword_init: true)

home = Address.new(street: "123 ถ.สุขุมวิท", city: "กรุงเทพฯ", zip_code: "10110")

puts home.city     # => กรุงเทพฯ

# home = Address.new("123 ถ.สุขุมวิท", "กรุงเทพฯ", "10110")
# => ArgumentError: wrong number of arguments (given 3, expected 0)
# (เมื่อระบุ keyword_init: true แล้ว ห้ามสร้างด้วย positional args อีกเด็ดขาด)
```

**อธิบาย:**

- `keyword_init: true` ทำให้ `Address.new` **ต้อง**ระบุทุกฟิลด์เป็น `key: value` เท่านั้น
  ป้องกันความสับสนเรื่องลำดับ argument โดยเฉพาะเมื่อมีฟิลด์ประเภทเดียวกันติดกัน (เช่น
  `street`/`city` ที่เป็น String ทั้งคู่ — ถ้าสลับลำดับกันตอนสร้างแบบ positional จะไม่มี error
  ใดๆ เตือนเลย เพราะชนิดข้อมูลตรงกัน)
- อ่าน `Address.new(street: ..., city: ..., zip_code: ...)` ก็เข้าใจได้ทันทีว่าแต่ละค่าคือ
  อะไร โดยไม่ต้องเปิดดูนิยาม `Struct.new` ก่อน — ข้อดีเดียวกับ keyword argument ใน method
  ทั่วไปที่เรียนใน Part 007

### Ruby 3.2+: ไม่ระบุ `keyword_init` เลยก็รองรับทั้งสองแบบได้อัตโนมัติ

ตั้งแต่ Ruby 3.2 เป็นต้นมา ถ้า**ไม่ระบุ** `keyword_init` เลย (ปล่อยเป็นค่า default ซึ่งคือ
`nil`) `Struct` จะฉลาดพอที่จะรับได้ทั้งสองรูปแบบโดยอัตโนมัติ:

```ruby
# Ruby >= 3.2 เท่านั้น — ไม่ระบุ keyword_init ก็ใช้ได้ทั้งสองแบบ
Point = Struct.new(:x, :y)

a = Point.new(1, 2)          # positional -> ใช้ได้
b = Point.new(x: 1, y: 2)    # keyword    -> ใช้ได้เช่นกัน

puts a == b   # => true
```

> **แนวคิดสำคัญ:** แม้ Ruby 3.2+ จะรองรับทั้งสองแบบโดยไม่ต้องระบุ `keyword_init` แต่ในทาง
> ปฏิบัติ **แนะนำให้ระบุ `keyword_init: true` อย่างชัดเจนเมื่อ Struct มีฟิลด์ตั้งแต่ 3 ตัว
> ขึ้นไป** เพื่อบังคับให้ทีมของเราเขียนโค้ดที่อ่านง่ายเสมอ ไม่ปล่อยให้เลือกใช้ positional
> แบบสับสนได้ — ความชัดเจนสำคัญกว่าความสั้นกระชับในกรณีนี้

---

## Step 134: เพิ่ม custom method ให้ `Struct` ด้วย block และการ subclass

`Struct` ไม่ได้จำกัดแค่เก็บข้อมูลเฉยๆ — เราใส่ method เพิ่มเข้าไปได้ 2 วิธีหลัก

### วิธีที่ 1: ส่ง block ให้ `Struct.new` (แนะนำ — กระชับและอ่านง่ายที่สุด)

```ruby
# frozen_string_literal: true

Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

origin = Point.new(0, 0)
target = Point.new(3, 4)

puts target.distance_to(origin)  # => 5.0
puts target                      # => (3, 4)
```

### วิธีที่ 2: ประกาศเป็น class แยก โดย inherit จาก `Struct.new(...)` ตรงๆ

```ruby
class Point < Struct.new(:x, :y)
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end
```

**อธิบาย:**

- ทั้งสองวิธีให้ผลลัพธ์เหมือนกันทุกประการในทางปฏิบัติ — วิธีที่ 1 (ส่ง block) เป็นที่นิยม
  มากกว่าในโค้ดยุคใหม่ เพราะไม่ต้องเขียนชื่อ class ซ้ำสองที่ (`class Point < Struct.new(...)`
  ดูแปลกตาเพราะ `Struct.new(...)` เป็น anonymous class ที่ไม่มีชื่อ แล้วเราค่อยตั้งชื่อทับ
  ด้วย `class Point < ...` ซึ่งบาง Ruby style guide ถือว่าน่าสับสน)
- ใน block เราเรียก `x`, `y` ตรงๆ ได้เลยโดยไม่ต้องใส่ `self.` เพราะ `attr_accessor` ที่
  `Struct` สร้างให้ทำงานเหมือน method ปกติทุกประการ
- เพิ่ม `include Comparable` เข้าไปใน block ได้ปกติ (ทบทวนจาก Step 99 ใน Part 010) เพื่อทำให้
  `Point` เรียงลำดับได้:

```ruby
Point = Struct.new(:x, :y) do
  include Comparable

  def <=>(other)
    Math.sqrt(x**2 + y**2) <=> Math.sqrt(other.x**2 + other.y**2)
  end
end

points = [Point.new(3, 4), Point.new(1, 1), Point.new(0, 2)]
puts points.sort.map(&:to_a).inspect
# => [[1, 1], [0, 2], [3, 4]]  (เรียงตามระยะห่างจากจุดกำเนิด)
```

---

## Step 135: `Struct` vs Custom Class เต็มรูปแบบ — เมื่อไหร่ควรใช้อะไร

`Struct` เร็วและกระชับ แต่ไม่ได้เหมาะกับทุกสถานการณ์ ต่อไปนี้คือเกณฑ์การเลือกใช้ในทางปฏิบัติ

| สถานการณ์ | ใช้ `Struct` | ใช้ Custom Class เต็มรูปแบบ |
|---|---|---|
| แค่ต้องการห่อกลุ่มข้อมูล (data grouping) ไม่มี business logic ซับซ้อน | ✅ เหมาะมาก | ทำได้แต่เกินความจำเป็น |
| มี validation ตอนสร้าง (เช่น ตรวจ `age` ต้อง >= 0) | ทำได้ผ่าน block แต่เริ่มดูอึดอัด | ✅ เหมาะกว่า — `initialize` เต็มรูปแบบควบคุมได้ยืดหยุ่น |
| ต้องการ private method ช่วยคำนวณภายใน | ทำได้ผ่าน block | ✅ เหมาะกว่า — จัดระเบียบ public/private ชัดเจน |
| จะมี subclass ต่อยอดหลายชั้น | ไม่แนะนำ (โครงสร้างเริ่มอ่านยาก) | ✅ เหมาะกว่า — inheritance ปกติชัดเจนกว่า |
| ทีมต้องอ่านโค้ดแล้วเห็นภาพรวม field ทั้งหมดในบรรทัดเดียว | ✅ `Struct.new(:a, :b, :c)` เห็นชัดทันที | ต้องไล่อ่าน `initialize`/`attr_reader` หลายบรรทัด |
| ต้องการ mutable object ที่แก้ค่าฟิลด์ได้ตรงๆ โดยไม่ผ่าน method พิเศษ | ✅ setter มีให้ฟรีทุกฟิลด์ | ต้องเขียน `attr_accessor` เองถ้าต้องการ |

**หลักคิดสั้นๆ:** เริ่มต้นด้วย `Struct` เสมอเมื่อสร้าง value object ง่ายๆ — **ถ้าวันหนึ่ง
`Struct` นั้นเริ่มมี method เกิน 3-4 ตัว มี validation ซับซ้อน หรือเริ่มมี dependency กับ
class อื่น ให้ "เลื่อนขั้น" (promote) เป็น custom class เต็มรูปแบบทันที** เพราะการแปลง
`Struct` เป็น class ปกติทำได้ง่ายมาก (แค่เปลี่ยน `Struct.new(:x, :y) do ... end` เป็น
`class Point; attr_accessor :x, :y; def initialize(x:, y:) ... end; ... end`) จึงไม่มีต้นทุน
สูงที่ต้อง "เดาถูกตั้งแต่แรก"

```ruby
# ตัวอย่างสัญญาณว่าถึงเวลาเลื่อนขั้นจาก Struct เป็น Class เต็มรูปแบบ
BankAccount = Struct.new(:owner, :balance) do
  def withdraw(amount)
    raise ArgumentError, "ยอดเงินไม่พอ" if amount > balance
    raise ArgumentError, "จำนวนต้องมากกว่า 0" if amount <= 0

    self.balance -= amount
  end

  def deposit(amount)
    raise ArgumentError, "จำนวนต้องมากกว่า 0" if amount <= 0

    self.balance += amount
  end
end
# เมื่อเริ่มมี validation หลายเงื่อนไข + logic ทางธุรกิจแบบนี้ นี่คือสัญญาณชัดเจนว่า
# BankAccount ควรเป็น "class BankAccount" เต็มรูปแบบแล้ว ไม่ใช่ Struct อีกต่อไป
# เพราะ Struct ควรเก็บไว้ใช้กับ "ข้อมูลบวก method ช่วยเล็กๆ น้อยๆ" เท่านั้น
```

---

## Step 136: `OpenStruct` — dynamic attribute แบบไม่ต้องประกาศ field ล่วงหน้า

`OpenStruct` (อยู่ใน standard library ต้อง `require "ostruct"` ก่อนใช้) ต่างจาก `Struct`
ตรงที่**ไม่ต้องประกาศชื่อฟิลด์ล่วงหน้าเลย** — สร้าง attribute ใหม่ได้ทันทีตอนกำหนดค่า
เหมาะมากสำหรับสร้างต้นแบบ (prototype) เร็วๆ หรือแปลงข้อมูลที่ schema ไม่แน่นอน (เช่น จาก
JSON/YAML ภายนอกที่โครงสร้างเปลี่ยนไปเรื่อยๆ)

```ruby
# frozen_string_literal: true

require "ostruct"

config = OpenStruct.new(host: "localhost", port: 3000)

puts config.host   # => localhost
puts config.port   # => 3000

# เพิ่ม attribute ใหม่ได้ทันที โดยไม่ต้องประกาศไว้ก่อนเลย
config.timeout = 30
puts config.timeout # => 30

# ลบ attribute ได้ด้วย
config.delete_field(:timeout)
puts config.timeout # => nil

puts config.to_h.inspect  # => {host: "localhost", port: 3000}
```

**อธิบาย:**

- `OpenStruct.new(host: ..., port: ...)` รับ Hash เข้ามาแล้วเปลี่ยนทุก key เป็น method
  เรียกได้แบบ dot notation ทันที (`config.host` แทนที่จะต้องเขียน `config[:host]`)
- `config.timeout = 30` เป็นการ**สร้าง attribute ใหม่** ที่ไม่ได้ประกาศไว้ตอนสร้าง object
  เลย — สิ่งนี้เป็นไปไม่ได้เลยกับ `Struct` (ซึ่งจะโยน `NoMethodError` ทันทีถ้าลองกำหนด field
  ที่ไม่ได้ประกาศไว้ตั้งแต่แรก)
- เหมาะมากสำหรับกรณีอย่างการแปลง JSON response จาก external API ที่โครงสร้างอาจไม่คงที่
  ให้เข้าถึงด้วย dot notation แทนที่จะต้องเขียน `data["key"]["nested_key"]` ซ้ำๆ

```ruby
require "ostruct"
require "json"

json_data = '{"name": "Somchai", "address": {"city": "Bangkok"}}'
parsed = JSON.parse(json_data, object_class: OpenStruct)

puts parsed.name            # => Somchai
puts parsed.address.city    # => Bangkok  (nested Hash ก็ถูกแปลงเป็น OpenStruct ให้อัตโนมัติ)
```

> **preview:** เบื้องหลังที่ทำให้ `config.timeout = 30` สร้าง method `timeout=` ขึ้นมาแบบ
> dynamic ได้แบบนี้ คือกลไก **`method_missing`** ซึ่งเป็นหัวใจของ metaprogramming ใน Ruby —
> เราจะเรียนกลไกนี้แบบเต็มรูปแบบใน **Part 015** ต่อจาก Part นี้ทันที

---

## Step 137: ข้อเสียของ `OpenStruct` — ไม่ป้องกัน typo, ช้ากว่า, และความเสี่ยงด้านความปลอดภัย

ความยืดหยุ่นของ `OpenStruct` มาพร้อมต้นทุนที่ต้องเข้าใจก่อนนำไปใช้ในโค้ด production จริง

### 1. ไม่มีการป้องกัน typo เลย

```ruby
require "ostruct"

user = OpenStruct.new(name: "Somchai", email: "somchai@example.com")

puts user.emial   # พิมพ์ผิด! แต่ไม่มี error ใดๆ เลย
# => nil

# เทียบกับ Struct ที่ป้องกัน typo แบบนี้ให้ทันที
User = Struct.new(:name, :email)
user2 = User.new("Somchai", "somchai@example.com")
user2.emial
# => NoMethodError: undefined method 'emial' for an instance of User
```

การพิมพ์ผิด `user.emial` แทน `user.email` กับ `OpenStruct` จะได้ `nil` เงียบๆ โดยไม่มี error
เตือนเลย — บั๊กแบบนี้อาจไหลลึกเข้าไปในระบบก่อนถูกจับได้ (เช่น ส่งอีเมลไม่สำเร็จเพราะที่อยู่
เป็น `nil` แต่โปรแกรมไม่ crash ให้เห็นทันที) ในขณะที่ `Struct` (และ `class` ปกติ) จะโยน
`NoMethodError` ทันทีที่เรียก method ที่ไม่มีอยู่จริง ทำให้เจอบั๊กเร็วกว่ามาก

### 2. ช้ากว่า `Struct` และ `Hash` อย่างมีนัยสำคัญ

```ruby
require "ostruct"
require "benchmark"

StructPoint = Struct.new(:x, :y)

Benchmark.bm(12) do |bm|
  bm.report("Struct") do
    100_000.times { StructPoint.new(1, 2).x }
  end

  bm.report("OpenStruct") do
    100_000.times { OpenStruct.new(x: 1, y: 2).x }
  end

  bm.report("Hash") do
    100_000.times { { x: 1, y: 2 }[:x] }
  end
end
# ผลลัพธ์โดยประมาณ (เวลาจริงขึ้นกับเครื่อง):
#              user     system      total        real
# Struct     0.015000   0.000000   0.015000 (  0.015234)
# OpenStruct 0.180000   0.010000   0.190000 (  0.195678)
# Hash       0.008000   0.000000   0.008000 (  0.008123)
```

**เหตุผลที่ `OpenStruct` ช้ากว่า:** `OpenStruct` เก็บข้อมูลภายในเป็น `Hash` แล้วใช้
`method_missing` เพื่อ**สร้าง method ใหม่แบบ dynamic ทุกครั้ง**ที่เจอ attribute ที่ไม่เคย
เรียกมาก่อน กระบวนการนี้มีค่าใช้จ่ายสูงกว่าการเรียก `attr_accessor` ธรรมดาของ `Struct` มาก
(ซึ่ง `Struct` สร้าง method จริงไว้ล่วงหน้าตั้งแต่ตอนนิยาม ไม่ต้องพึ่ง `method_missing`
ระหว่างรัน) นอกจากนี้ทุก instance ของ `OpenStruct` ยังมี**ตารางเก็บ attribute ของตัวเอง**
ทำให้ใช้หน่วยความจำมากกว่าด้วย

### 3. ความเสี่ยงด้านความปลอดภัยเมื่อรับข้อมูลจากภายนอกโดยไม่ระวัง

```ruby
require "ostruct"

# อันตราย: แปลง params จากผู้ใช้เป็น OpenStruct ตรงๆ โดยไม่กรอง
untrusted_params = { name: "Somchai", is_admin: true, delete_all_data: true }
user_settings = OpenStruct.new(untrusted_params)

# ทุก key ที่ผู้ใช้ส่งมากลายเป็น attribute ที่เรียกได้ทันที แม้เราไม่ได้ตั้งใจให้มีอยู่
puts user_settings.is_admin          # => true  (ผู้ใช้ "แอบใส่" attribute นี้เข้ามาเอง!)
puts user_settings.delete_all_data   # => true
```

เพราะ `OpenStruct` ยอมรับ key อะไรก็ได้จาก Hash ที่ส่งเข้ามาโดยไม่มีการตรวจสอบ "รายการ field
ที่อนุญาต" เลย ถ้านำ Hash ที่มาจาก input ของผู้ใช้ (เช่น `params` ใน Rails) ไปสร้าง
`OpenStruct` ตรงๆ โดยไม่กรองก่อน อาจเปิดช่องให้ผู้ใช้ "แอบใส่" attribute ที่ไม่ควรมีสิทธิ์
กำหนดเข้ามาได้ (คล้ายปัญหา mass assignment ที่จะเรียนละเอียดใน Phase 4 และ Phase 13)

> **สรุปกฎการใช้ `OpenStruct` ในทางปฏิบัติ:** ใช้ได้ดีสำหรับงาน**ชั่วคราว**อย่างสคริปต์
> ทดลอง, แปลงข้อมูล config ที่เชื่อถือได้ (เช่น จากไฟล์ YAML ของโปรเจกต์เราเอง), หรือ
> prototype ที่ยังไม่รู้ schema แน่ชัด — แต่**หลีกเลี่ยงการใช้ในโค้ด production ที่ประมวลผล
> ข้อมูลจากภายนอกโดยตรง หรือในจุดที่ performance สำคัญ** ให้ใช้ `Struct`, `Data`, หรือ
> custom class แทนเสมอเมื่อรู้ schema ล่วงหน้าแล้ว

---

## Step 138: `Data.define` (Ruby 3.2+) — Value Object แบบ Immutable ยุคใหม่

`Data` เป็น class ใหม่ที่เพิ่มเข้ามาใน **Ruby 3.2** ออกแบบมาเพื่อแก้จุดอ่อนของ `Struct`
โดยเฉพาะสำหรับกรณีที่เราต้องการ **value object แบบ immutable** (สร้างแล้วแก้ค่าไม่ได้อีกเลย)
— แนวคิดแบบเดียวกับ `record` ใน Java หรือ `data class` ใน Kotlin

> **ย้ำอีกครั้ง:** `Data.define` ใช้ได้เฉพาะ **Ruby >= 3.2** เท่านั้น ถ้าโปรเจกต์ที่ทำงานอยู่
> ใช้ Ruby เวอร์ชันเก่ากว่านี้ (พบได้บ่อยในระบบเก่า) ต้องใช้ `Struct` แทนไปก่อน

```ruby
# frozen_string_literal: true

Point = Data.define(:x, :y)

# สร้างได้ทั้งแบบ keyword และ positional (Data.define รองรับทั้งสองแบบให้ทันทีโดยไม่ต้อง
# ตั้งค่าอะไรเพิ่ม ต่างจาก Struct ที่ต้องเลือก keyword_init เอง)
p1 = Point.new(x: 1, y: 2)
p2 = Point.new(1, 2)

puts p1.x        # => 1
puts p1.y        # => 2
puts p1 == p2    # => true (เปรียบเทียบตามค่าภายในเหมือน Struct)

puts p1.frozen?  # => true  (ทุก instance ของ Data ถูก freeze ให้อัตโนมัติทันทีที่สร้าง)

p1.x = 99
# => NoMethodError: undefined method 'x=' for an instance of Point
# (Data ไม่มี setter ให้เลย — ออกแบบมาให้ immutable โดยสมบูรณ์)
```

**อธิบาย:**

- `Data.define(:x, :y)` คืน class ใหม่เหมือน `Struct.new` แต่**ไม่มี method `x=`/`y=`ให้เลย**
  — เมื่อสร้าง instance แล้ว ค่าจะไม่มีทางเปลี่ยนแปลงได้อีก (`frozen?` เป็น `true` เสมอ)
- รองรับทั้งการสร้างแบบ positional (`Point.new(1, 2)`) และ keyword (`Point.new(x: 1, y: 2)`)
  พร้อมกันตั้งแต่แรก โดยไม่ต้องเลือกโหมดเหมือน `Struct`
- `Data` **ไม่ `include Enumerable`** และ**ไม่มี `#[]`, `#each`, `#to_a`** เหมือน `Struct` —
  เป็นการตัดใจของทีม Ruby core เพื่อให้ `Data` เป็น value object ที่ "บริสุทธิ์" (แค่เก็บค่า
  กับเปรียบเทียบค่า) ไม่แบกความสามารถแบบ collection มาด้วยเหมือน `Struct`

### `with` — วิธี "แก้ไข" ค่าของ object ที่เป็น immutable (โดยสร้างตัวใหม่)

เพราะแก้ค่าตรงๆ ไม่ได้ วิธีมาตรฐานในการ "เปลี่ยนค่า" ของ `Data` คือสร้าง **instance ใหม่**
ที่มีบางฟิลด์เปลี่ยนไป ผ่าน method `with`

```ruby
p1 = Point.new(x: 1, y: 2)
p3 = p1.with(x: 100)

puts p3.x    # => 100
puts p3.y    # => 2 (ฟิลด์ที่ไม่ได้ระบุใน with จะคงค่าเดิมจาก p1)
puts p1.x    # => 1 (p1 เองไม่เปลี่ยนแปลงเลย — with คืน object ใหม่เสมอ)
```

### `to_h`, `deconstruct`, `deconstruct_keys` — รองรับ Pattern Matching เต็มรูปแบบ

```ruby
p1 = Point.new(x: 1, y: 2)

puts p1.to_h.inspect              # => {x: 1, y: 2}
puts p1.deconstruct.inspect       # => [1, 2]           (สำหรับ array pattern)
puts p1.deconstruct_keys(nil).inspect  # => {x: 1, y: 2} (สำหรับ hash pattern)

case p1
in { x:, y: } if x == y
  puts "อยู่บนเส้นทแยงมุม"
in { x: 0, y: }
  puts "อยู่บนแกน Y ที่ y=#{y}"
in { x:, y: }
  puts "จุด (#{x}, #{y})"
end
# => จุด (1, 2)
```

`Data` ถูกออกแบบมาให้ทำงานร่วมกับ **pattern matching** (`case`/`in` ที่จะเรียนละเอียดใน
Part หลังๆ) ได้อย่างเป็นธรรมชาติที่สุดในบรรดาเครื่องมือทั้งหมดใน Part นี้ เพราะมี
`deconstruct`/`deconstruct_keys` ติดตัวมาให้ฟรีทันที

### เพิ่ม custom method ให้ `Data.define` ด้วย block เหมือน `Struct`

```ruby
Point = Data.define(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

origin = Point.new(x: 0, y: 0)
target = Point.new(x: 3, y: 4)

puts target.distance_to(origin)  # => 5.0
puts target                      # => (3, 4)
```

---

## Step 139: ตารางเปรียบเทียบ `Struct` vs `Data` vs `OpenStruct` vs `Hash`

ตอนนี้เราเห็นเครื่องมือทั้ง 4 ตัวครบแล้ว (รวม `Hash` ธรรมดาที่เรียนไปตั้งแต่ Part 005)
มาสรุปเป็นตารางเดียวเพื่อใช้อ้างอิงตอนตัดสินใจเลือกใช้ในงานจริง

| คุณสมบัติ | `Struct` | `Data` (Ruby 3.2+) | `OpenStruct` | `Hash` |
|---|---|---|---|---|
| ต้องประกาศ field ล่วงหน้า | ต้อง | ต้อง | **ไม่ต้อง** (เพิ่มเมื่อไหร่ก็ได้) | ไม่ต้อง |
| แก้ค่าฟิลด์ได้หลังสร้าง (mutable) | ได้ | **ไม่ได้** (immutable, frozen) | ได้ | ได้ |
| ป้องกัน typo ตอนเรียก field ที่ไม่มี | ได้ (`NoMethodError`) | ได้ (`NoMethodError`) | **ไม่ได้** (คืน `nil` เงียบๆ) | ไม่ได้ (คืน `nil` เงียบๆ) |
| ความเร็ว | เร็ว | เร็วที่สุด (ออกแบบมาเพื่อ perf โดยเฉพาะ) | ช้ากว่าชัดเจน (`method_missing`) | เร็ว (built-in C) |
| เข้าถึงด้วย dot notation (`obj.field`) | ได้ | ได้ | ได้ | ไม่ได้ (ต้อง `hash[:field]`) |
| เข้าถึงด้วย `[]` เหมือน Array/Hash | ได้ (`obj[0]`, `obj[:x]`) | **ไม่ได้** | ไม่ได้ (ใช้ dot เท่านั้น) | ได้ (`hash[:x]`) |
| `include Enumerable` (มี `map`, `each` ฯลฯ) | มี | **ไม่มี** | ไม่มี | มี (ผ่าน Hash เอง) |
| รองรับ Pattern Matching (`case`/`in`) | ได้ | **ได้ดีที่สุด** (ออกแบบมาเพื่อสิ่งนี้) | ไม่ได้ | ได้ |
| ต้อง `require` เพิ่ม | ไม่ต้อง | ไม่ต้อง | **ต้อง** `require "ostruct"` | ไม่ต้อง |
| เหมาะกับ | value object ทั่วไปที่ยังต้องแก้ค่าได้ | value object แบบ immutable (Money, Coordinate, event record) | prototype เร็วๆ, config ที่เชื่อถือได้, แปลง JSON แบบ schema ไม่แน่นอน | ข้อมูล key-value ทั่วไปที่ไม่ต้องการชื่อ type ชัดเจน |

### ตัวอย่างการใช้ `Data.define` ในสถานการณ์จริง: `Money`

```ruby
# frozen_string_literal: true

Money = Data.define(:amount, :currency) do
  def +(other)
    raise ArgumentError, "สกุลเงินไม่ตรงกัน" unless currency == other.currency

    Money.new(amount: amount + other.amount, currency: currency)
  end

  def to_s
    format("%.2f %s", amount, currency)
  end
end

price1 = Money.new(amount: 100.50, currency: "THB")
price2 = Money.new(amount: 49.50, currency: "THB")

total = price1 + price2
puts total          # => 150.00 THB
puts price1         # => 100.50 THB  (price1 ไม่เปลี่ยนแปลงเลย เพราะ + คืน object ใหม่เสมอ)

usd = Money.new(amount: 10, currency: "USD")
price1 + usd
# => ArgumentError: สกุลเงินไม่ตรงกัน
```

**ทำไมตัวอย่างนี้ถึงเหมาะกับ `Data` มากกว่า `Struct`:** เงินเป็นแนวคิดที่ควร**ไม่เปลี่ยนแปลง
ค่าเดิม**เมื่อทำการคำนวณ (เหมือน `Integer`/`String` ใน Ruby ที่ operation อย่าง `+` คืนค่า
ใหม่เสมอ ไม่แก้ตัวเดิม) — ถ้าใช้ `Struct` แล้วมีโค้ดส่วนไหนเผลอเขียน `price1.amount += 50`
ตรงๆ จะกลายเป็นบั๊กที่ตามหายากมาก เพราะ `price1` ที่ถูกส่งต่อไปที่อื่นแล้วจะถูกแก้ไขค่าไป
ด้วยโดยไม่มีใครคาดคิด `Data` ป้องกันบั๊กประเภทนี้ได้ตั้งแต่ระดับภาษา (compile-time ในทาง
ความหมาย แม้ Ruby จะเป็น interpreted language ก็ตาม — คือ error จะเกิดทันทีที่ลอง assign)

---

## Step 140: แบบฝึกหัด — สร้าง `Point` และ `Money` ด้วย `Data.define` พร้อม Operator Overloading

### โจทย์

สร้างไฟล์ `value_objects.rb` ที่มี 2 value object ดังนี้ โดยใช้ `Data.define` ทั้งคู่:

1. **`Point`** — เก็บพิกัด `x`, `y` (ตัวเลข) ต้องมี:
   - `+` และ `-` สำหรับบวก/ลบพิกัดสองจุด (คืน `Point` ใหม่)
   - `magnitude` คำนวณระยะห่างจากจุดกำเนิด (0, 0)
   - `include Comparable` โดยเรียงลำดับตาม `magnitude` (จุดที่อยู่ใกล้จุดกำเนิดกว่าถือว่า
     "น้อยกว่า")
   - `to_s` แสดงผลรูปแบบ `"(x, y)"`

2. **`Money`** — เก็บ `amount` (Float) และ `currency` (String) ต้องมี:
   - `+` และ `-` (บวก/ลบเฉพาะสกุลเงินเดียวกัน ไม่งั้น `raise ArgumentError`)
   - `*` สำหรับคูณด้วยตัวเลข (เช่น คำนวณราคารวมจากจำนวนชิ้น)
   - `include Comparable` โดยเรียงลำดับตาม `amount` (ต้องเช็คสกุลเงินตรงกันก่อนเทียบด้วย
     ไม่งั้น `raise ArgumentError`)
   - `to_s` แสดงผลรูปแบบ `"100.50 THB"`

### เฉลย

```ruby
# frozen_string_literal: true

# value_objects.rb

# ============================================================
# Point — พิกัด 2 มิติ แบบ immutable value object
# ============================================================
Point = Data.define(:x, :y) do
  include Comparable

  def +(other)
    Point.new(x: x + other.x, y: y + other.y)
  end

  def -(other)
    Point.new(x: x - other.x, y: y - other.y)
  end

  def magnitude
    Math.sqrt((x**2) + (y**2))
  end

  def <=>(other)
    magnitude <=> other.magnitude
  end

  def to_s
    "(#{x}, #{y})"
  end
end

# ============================================================
# Money — จำนวนเงินพร้อมสกุลเงิน แบบ immutable value object
# ============================================================
Money = Data.define(:amount, :currency) do
  include Comparable

  def +(other)
    ensure_same_currency!(other)
    Money.new(amount: amount + other.amount, currency: currency)
  end

  def -(other)
    ensure_same_currency!(other)
    Money.new(amount: amount - other.amount, currency: currency)
  end

  def *(multiplier)
    Money.new(amount: amount * multiplier, currency: currency)
  end

  def <=>(other)
    ensure_same_currency!(other)
    amount <=> other.amount
  end

  def to_s
    format("%.2f %s", amount, currency)
  end

  private

  def ensure_same_currency!(other)
    return if currency == other.currency

    raise ArgumentError, "ไม่สามารถคำนวณข้ามสกุลเงินได้: #{currency} กับ #{other.currency}"
  end
end

# ============================================================
# ทดสอบใช้งาน
# ============================================================
if __FILE__ == $PROGRAM_NAME
  # --- Point ---
  a = Point.new(x: 3, y: 4)
  b = Point.new(x: 1, y: 1)

  puts "a = #{a}, b = #{b}"                # => a = (3, 4), b = (1, 1)
  puts "a + b = #{a + b}"                  # => a + b = (4, 5)
  puts "a - b = #{a - b}"                  # => a - b = (2, 3)
  puts "a.magnitude = #{a.magnitude}"      # => a.magnitude = 5.0
  puts "a > b? #{a > b}"                   # => a > b? true (a ไกลจากจุดกำเนิดกว่า)

  points = [a, b, Point.new(x: 0, y: 0)]
  puts "sort: #{points.sort.map(&:to_s)}"
  # => sort: ["(0, 0)", "(1, 1)", "(3, 4)"]

  puts "-" * 40

  # --- Money ---
  price = Money.new(amount: 100.0, currency: "THB")
  shipping = Money.new(amount: 30.0, currency: "THB")

  total = price + shipping
  puts "ยอดรวม: #{total}"                  # => ยอดรวม: 130.00 THB

  bulk_price = price * 3
  puts "3 ชิ้น ราคา: #{bulk_price}"        # => 3 ชิ้น ราคา: 300.00 THB

  prices = [Money.new(amount: 50, currency: "THB"),
            Money.new(amount: 200, currency: "THB"),
            Money.new(amount: 10, currency: "THB")]
  puts "ราคาถูกที่สุด: #{prices.min}"       # => ราคาถูกที่สุด: 10.00 THB
  puts "ราคาแพงที่สุด: #{prices.max}"       # => ราคาแพงที่สุด: 200.00 THB

  begin
    price + Money.new(amount: 5, currency: "USD")
  rescue ArgumentError => e
    puts "จับ error ได้ถูกต้อง: #{e.message}"
    # => จับ error ได้ถูกต้อง: ไม่สามารถคำนวณข้ามสกุลเงินได้: THB กับ USD
  end

  # ยืนยันว่าเป็น immutable จริง
  puts "price ยังคงเดิมหลังบวก: #{price}"   # => price ยังคงเดิมหลังบวก: 100.00 THB
  puts "price.frozen? = #{price.frozen?}"  # => price.frozen? = true
end
```

ทดสอบรัน:

```bash
ruby value_objects.rb
```

**จุดที่ควรสังเกตในเฉลย:**

- **`ensure_same_currency!` เป็น private method** ภายใน block ของ `Data.define` — ยืนยันว่า
  block ที่ส่งให้ `Data.define`/`Struct.new` ทำงานเหมือน class body ทุกประการ รวมถึง
  `private` ก็ใช้งานได้ปกติ (ทบทวนแนวคิด `private` จาก Part 010 Step 93)
- **`<=>` ของ `Money` เรียก `ensure_same_currency!` ก่อนเทียบเสมอ** — ป้องกันไม่ให้ `sort`,
  `min`, `max` เอาเงินคนละสกุลมาเทียบกันโดยไม่รู้ตัว ซึ่งเป็นบั๊กเชิงตรรกะที่ร้ายแรงถ้าเกิด
  ในระบบการเงินจริง
- **`+`, `-`, `*` ทุกตัวคืน `Money`/`Point` ตัวใหม่เสมอ ไม่เคยแก้ `self`** — สอดคล้องกับ
  ธรรมชาติ immutable ของ `Data` ที่เรียนใน Step 138 ทำให้ `price` เดิมไม่มีทางถูกเปลี่ยนแปลง
  ค่าโดยไม่ตั้งใจแม้จะผ่านการคำนวณไปแล้วกี่ครั้งก็ตาม
- **`include Comparable` ใช้ได้ปกติทั้งใน `Struct.new` และ `Data.define`** — ยืนยันว่าความรู้
  เรื่อง `Comparable` จาก Step 131 (และ Part 010 Step 99) นำไปใช้กับเครื่องมือสร้าง value
  object ตัวไหนก็ได้เหมือนกันหมด เพราะ `Comparable` แค่ต้องการ `<=>` เท่านั้น ไม่สนใจว่า
  class นั้นมาจากไหน

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **สร้าง `TimeRange`** ด้วย `Data.define(:start_time, :end_time)` ที่มี method
   `duration_in_minutes`, `overlaps?(other)` (เช็คว่าช่วงเวลาสองช่วงทับซ้อนกันหรือไม่), และ
   `include Comparable` โดยเรียงตาม `start_time` — ลองใช้ `Time.now` และ `Time.now + 3600`
   ทดสอบ

2. **แปลง `BankAccount` (จาก Step 135 ที่เป็น `Struct`)** ให้เป็น custom class เต็มรูปแบบตาม
   หลักเกณฑ์ใน Step 135 พร้อมเพิ่ม validation ว่า `balance` เริ่มต้นต้องไม่ติดลบ (ถ้าติดลบ
   ให้ `raise ArgumentError` ใน `initialize`) — นี่คือตัวอย่างการ "เลื่อนขั้น" จาก `Struct`
   เป็น class จริงตามที่อธิบายไว้

3. **เปรียบเทียบ performance จริง** — เขียน benchmark ของตัวเองเปรียบเทียบความเร็วการสร้าง
   object และการเข้าถึง attribute ระหว่าง `Struct`, `Data`, `OpenStruct`, และ `Hash` แบบมี
   field 5 ตัว (คล้ายตัวอย่างใน Step 137 แต่ขยายให้ครบทั้ง 4 แบบ) แล้วสรุปผลเป็นตารางของ
   ตัวเอง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เจาะลึก **`Comparable`** ครบทุก method ที่ได้มาฟรี (`<`, `<=`, `==`, `>`, `>=`,
  `between?`, `clamp`) จากการนิยาม `<=>` เพียงตัวเดียว พร้อมเข้าใจข้อควรระวังสำคัญว่า
  `Comparable#==` ไม่ได้ปรับ `eql?`/`hash` ให้อัตโนมัติ ต้องนิยามแยกต่างหากถ้าจะใช้ object
  เป็น key ของ `Hash`/`Set`
- สร้าง value object แบบรวดเร็วด้วย **`Struct.new`** ทั้งแบบ positional และ
  `keyword_init: true` รวมถึงรู้ว่า Ruby 3.2+ รองรับทั้งสองแบบพร้อมกันได้โดยไม่ต้องระบุ
  `keyword_init`
- เพิ่ม custom method ให้ `Struct` ผ่าน block และรู้เกณฑ์ตัดสินใจว่าเมื่อไหร่ควร "เลื่อนขั้น"
  จาก `Struct` ไปเป็น custom class เต็มรูปแบบ
- ใช้ **`OpenStruct`** สำหรับสร้าง dynamic attribute โดยไม่ต้องประกาศ field ล่วงหน้า พร้อม
  เข้าใจข้อเสีย 3 ด้าน: ไม่ป้องกัน typo, ช้ากว่าอย่างมีนัยสำคัญ (จาก `method_missing`), และ
  ความเสี่ยงด้านความปลอดภัยเมื่อรับข้อมูลจากภายนอกโดยไม่กรอง
- ใช้ **`Data.define`** (Ruby 3.2+) สร้าง value object แบบ **immutable** ยุคใหม่ พร้อม
  `with` สำหรับสร้าง instance ใหม่จากการเปลี่ยนบางฟิลด์ และ `deconstruct`/`deconstruct_keys`
  ที่รองรับ pattern matching ได้ดีที่สุดในบรรดาเครื่องมือทั้งหมด
- เปรียบเทียบ **`Struct` vs `Data` vs `OpenStruct` vs `Hash`** ในตารางเดียว เพื่อใช้เลือก
  เครื่องมือให้เหมาะกับสถานการณ์ได้ทันทีในงานจริง
- สร้าง value object จริง (`Point`, `Money`) ด้วย `Data.define` พร้อม operator overloading
  (`+`, `-`, `*`) และ `Comparable` ผสมกัน เป็นรูปแบบที่จะเจอบ่อยมากตอนออกแบบ domain model
  ใน Rails application ตั้งแต่ Phase 3 เป็นต้นไป

**ต่อไป (Part 015):** เราจะเจาะลึก **Metaprogramming เบื้องต้น** — กลไกเบื้องหลังที่ทำให้
`OpenStruct` สร้าง method แบบ dynamic ได้อย่างที่เห็นใน Part นี้ ผ่าน `method_missing`,
`define_method`, `send`, และ `respond_to?` ซึ่งเป็นเทคนิคขั้นสูงที่ gem ดังๆ ในวงการ Ruby
(รวมถึง Rails เองก็ใช้หนักมาก) ใช้สร้าง API ที่ดูเหมือนเวทมนตร์ — เข้าใจกลไกนี้แล้วจะทำให้
อ่านโค้ดของ Rails framework เองได้ลึกซึ้งขึ้นมากในอนาคต
