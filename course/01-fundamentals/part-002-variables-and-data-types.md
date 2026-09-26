# Part 002: ตัวแปร และชนิดข้อมูลพื้นฐานของ Ruby

> **Step ครอบคลุมใน Part นี้:** Step 11–20
> **ระดับ:** เริ่มต้น (ต่อเนื่องจาก Part 001)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 11: ตัวแปรใน Ruby — การประกาศ, การตั้งชื่อ, และ scope เบื้องต้น
- Step 12: Integer — จำนวนเต็ม, การคำนวณ, และ method ที่ใช้บ่อย
- Step 13: Float — จำนวนทศนิยม และปัญหาความแม่นยำที่ต้องระวัง
- Step 14: String — การสร้าง string, เครื่องหมายคำพูด, และเรื่อง encoding
- Step 15: Symbol vs String — ต่างกันอย่างไร และทำไม Rails ชอบใช้ Symbol เป็น key
- Step 16: nil — ค่า "ไม่มีอะไร" และพฤติกรรมของมัน
- Step 17: true/false และ Truthiness — อะไรคือ falsy ใน Ruby
- Step 18: การแปลงชนิดข้อมูลแบบ implicit method (`to_s`, `to_i`, `to_f`, `to_sym`, `to_a`)
- Step 19: `Integer()` และ `Float()` — แปลงชนิดข้อมูลแบบเข้มงวด ต่างจาก `to_i`/`to_f` อย่างไร
- Step 20: แบบฝึกหัดโปรเจกต์ — โปรแกรมคำนวณใบเสร็จร้านค้าเล็กๆ

---

## Step 11: ตัวแปรใน Ruby — การประกาศ, การตั้งชื่อ, และ scope เบื้องต้น

### การประกาศตัวแปร

Ruby เป็นภาษาแบบ **dynamically typed** คือไม่ต้องประกาศชนิดข้อมูลของตัวแปรล่วงหน้า
(ต่างจาก Java หรือ C ที่ต้องเขียนเช่น `int x = 5;`) แค่กำหนดค่าให้ตัวแปรก็พร้อมใช้งานทันที:

```ruby
# frozen_string_literal: true

name = "Ruby"
age = 30
price = 19.99

puts name   # => Ruby
puts age    # => 30
puts price  # => 19.99
```

**อธิบาย:** เครื่องหมาย `=` คือ assignment operator (ไม่ใช่ "เท่ากับ" แบบทางคณิตศาสตร์)
ทำหน้าที่ผูกชื่อตัวแปรทางซ้ายเข้ากับ object ทางขวา ตัวแปรใน Ruby ไม่ได้ "เก็บค่า" ตรงๆ
แต่เป็นเหมือน **ป้ายชื่อ (reference)** ที่ชี้ไปยัง object ในหน่วยความจำ

```ruby
a = "hello"
b = a          # b ชี้ไปยัง object เดียวกับ a (ไม่ได้ copy string ใหม่)

b << " world"  # << ต่อท้าย string เดิม (mutate object)
puts a         # => hello world  (a เปลี่ยนตามด้วย! เพราะชี้ไปที่ object เดียวกัน)
puts a.equal?(b) # => true (equal? เช็คว่าเป็น object เดียวกันใน memory จริงหรือไม่)
```

> **สิ่งสำคัญที่มือใหม่มักพลาด:** การ assign ตัวแปรใหม่ (`b = a`) ไม่ได้ copy ข้อมูล
> แต่เป็นการ copy "reference" เท่านั้น ถ้าไปแก้ไข object ผ่านการเรียก method ที่ mutate
> (เช่น `<<`, `upcase!`, `push`) ตัวแปรอื่นที่ชี้ไปยัง object เดียวกันจะเห็นการเปลี่ยนแปลงด้วย
> เรื่องนี้จะสำคัญมากขึ้นเมื่อเราเรียน Array และ Hash ใน Part ถัดๆ ไป

### กฎการตั้งชื่อตัวแปร

```ruby
# ตัวแปรปกติ (local variable) ต้องขึ้นต้นด้วยตัวพิมพ์เล็กหรือ underscore
user_name = "สมชาย"     # snake_case คือธรรมเนียมมาตรฐานของ Ruby (ไม่ใช้ camelCase)
_unused_variable = 42    # ขึ้นต้นด้วย _ นิยมใช้บอกว่า "ตัวแปรนี้ตั้งใจไม่ใช้"
age2 = 25                # มีตัวเลขได้ แต่ห้ามขึ้นต้นด้วยตัวเลข

# ชื่อที่ผิดกฎ (จะทำให้เกิด SyntaxError)
# 2age = 25        # ผิด - ขึ้นต้นด้วยตัวเลขไม่ได้
# user-name = "x"  # ผิด - มี - ไม่ได้ (Ruby จะตีความ - เป็นเครื่องหมายลบ)
```

**ธรรมเนียมการตั้งชื่อ (naming convention) ที่ใช้ตลอดหลักสูตรนี้:**

| รูปแบบ | ใช้กับ | ตัวอย่าง |
|--------|--------|----------|
| `snake_case` | ตัวแปร, method | `first_name`, `calculate_total` |
| `SCREAMING_SNAKE_CASE` | ค่าคงที่ (constant) | `MAX_RETRY_COUNT`, `TAX_RATE` |
| `PascalCase` (UpperCamelCase) | Class, Module | `UserAccount`, `Payment` |
| ลงท้ายด้วย `?` | method ที่คืนค่า true/false | `empty?`, `valid?` |
| ลงท้ายด้วย `!` | method ที่ "อันตราย" กว่าปกติ (มักหมายถึง mutate ตัวเอง) | `upcase!`, `sort!` |

```ruby
MAX_RETRY_COUNT = 3    # Constant - ตัวแปรที่ไม่ควรถูกเปลี่ยนค่าอีก (ขึ้นต้นด้วยตัวพิมพ์ใหญ่)
puts MAX_RETRY_COUNT   # => 3

MAX_RETRY_COUNT = 5    # Ruby "อนุญาต" ให้เปลี่ยนได้ แต่จะเตือน warning
# warning: already initialized constant MAX_RETRY_COUNT
```

> **แนวคิดสำคัญ:** Ruby ไม่ได้บังคับว่า constant ห้ามเปลี่ยนค่าจริงๆ (แค่เตือน warning)
> เป็นเพียง "สัญญาทางสังคม" ระหว่างนักพัฒนาว่าค่านี้ไม่ควรถูกแก้ไขระหว่างโปรแกรมทำงาน
> ถ้าต้องการค่าที่ห้ามแก้ไขจริงๆ (immutable) ต้องเรียก `.freeze` เพิ่ม เช่น
> `MAX_RETRY_COUNT = 3.freeze`

### Scope ของตัวแปรเบื้องต้น

Ruby มีตัวแปรหลายประเภทแยกตาม scope โดยดูจาก**สัญลักษณ์นำหน้าชื่อ**:

```ruby
$global_variable = "เข้าถึงได้จากทุกที่ในโปรแกรม"  # Global variable - ขึ้นต้นด้วย $
CONSTANT_NAME = "ค่าคงที่"                          # Constant - ขึ้นต้นด้วยตัวพิมพ์ใหญ่
local_variable = "เข้าถึงได้เฉพาะใน scope ปัจจุบัน"  # Local variable - ขึ้นต้นด้วยตัวพิมพ์เล็ก/_

# @instance_variable และ @@class_variable จะสอนละเอียดใน Part ที่พูดถึง OOP (Step 81-90)
```

```ruby
# ตัวอย่างที่แสดงให้เห็นว่า local variable มี scope จำกัดอยู่ใน block/method
def greet
  message = "สวัสดี"  # message เป็น local variable ภายใน method นี้เท่านั้น
  puts message
end

greet
# puts message  # => NameError: undefined local variable or method 'message'
#                    (message มองไม่เห็นจากนอก method - เพราะ scope ถูกจำกัดไว้)
```

> **หมายเหตุ:** เรื่อง scope แบบละเอียด (โดยเฉพาะ scope ของ block กับ method) จะกลับมา
> อธิบายซ้ำอย่างละเอียดอีกครั้งใน Part ที่พูดถึงเรื่อง Method (Step 61–70) และ Block
> (Step 71–80) — ตอนนี้ขอให้จำแค่หลักการว่า **local variable ที่สร้างใน method หนึ่ง
> จะไม่รั่วไหลออกไปนอก method นั้น**

---

## Step 12: Integer — จำนวนเต็ม, การคำนวณ, และ method ที่ใช้บ่อย

### พื้นฐาน

```ruby
a = 10
b = 3

puts a + b   # => 13   (บวก)
puts a - b   # => 7    (ลบ)
puts a * b   # => 30   (คูณ)
puts a / b   # => 3    (หาร - สังเกตว่าได้ 3 ไม่ใช่ 3.333... เพราะเป็นการหารแบบ Integer)
puts a % b   # => 1    (modulo - หาเศษที่เหลือจากการหาร)
puts a ** b  # => 1000 (ยกกำลัง - a ยกกำลัง b)
```

**จุดที่มือใหม่พลาดบ่อยที่สุด: Integer division**

```ruby
puts 10 / 3     # => 3   (ไม่ใช่ 3.333...)
puts 7 / 2      # => 3
puts(-7 / 2)    # => -4  (ปัดลงเสมอ ไม่ใช่ปัดเข้าใกล้ศูนย์ - ต้องระวัง!)
```

**เหตุผล:** เมื่อ operand ทั้งสองฝั่งเป็น Integer ทั้งคู่ Ruby จะทำ **integer division**
คือตัดเศษทศนิยมทิ้งไปเลย (ปัดลงหาค่าลบอนันต์เสมอ ไม่ใช่ตัดทศนิยมทิ้งแบบตรงไปตรงมา
ซึ่งต่างจากบางภาษาอย่าง C ที่ปัดเข้าหาศูนย์) ถ้าต้องการผลลัพธ์เป็นทศนิยม ต้องแปลงให้อย่างน้อย
หนึ่งฝั่งเป็น Float ก่อน:

```ruby
puts 10.0 / 3     # => 3.3333333333333335
puts 10 / 3.0     # => 3.3333333333333335
puts 10.to_f / 3  # => 3.3333333333333335
```

### Method ที่ใช้บ่อยกับ Integer

```ruby
puts 10.even?      # => true   - เช็คว่าเป็นเลขคู่
puts 10.odd?       # => false  - เช็คว่าเป็นเลขคี่
puts (-5).abs      # => 5      - ค่าสัมบูรณ์ (absolute value)
puts 5.zero?       # => false  - เช็คว่าเป็น 0 หรือไม่
puts 0.zero?       # => true
puts 5.positive?   # => true
puts (-5).negative? # => true
puts 5.times { |i| print "#{i} " }  # => 0 1 2 3 4  (วนซ้ำ 5 ครั้ง, i คือ index เริ่มจาก 0)
puts
puts 5.next        # => 6      - ค่าถัดไป
puts 5.pred        # => 4      - ค่าก่อนหน้า
puts 10.gcd(15)    # => 5      - ตัวหารร่วมมาก (greatest common divisor)
puts 4.lcm(6)      # => 12     - ตัวคูณร่วมน้อย (least common multiple)
puts 255.to_s(16)  # => "ff"   - แปลงเป็น string ในฐาน 16 (hexadecimal)
puts "ff".to_i(16) # => 255    - แปลงกลับจาก hex string เป็น Integer
```

### Integer ใน Ruby ไม่มี overflow

จุดเด่นสำคัญของ Ruby ที่ต่างจากหลายภาษา (เช่น C, Java `int`) คือ **Integer ของ Ruby
ไม่มีขอบเขตขนาดสูงสุด** — เมื่อค่าตัวเลขใหญ่เกินขนาด machine word ปกติ (`Fixnum` เดิม)
Ruby จะเปลี่ยนไปใช้ `Bignum` โดยอัตโนมัติแบบไร้รอยต่อ (ตั้งแต่ Ruby 2.4 เป็นต้นมา `Fixnum`
และ `Bignum` ถูกรวมเป็น class เดียวคือ `Integer`):

```ruby
big_number = 2**100
puts big_number
# => 1267650600228229401496703205376
puts big_number.class  # => Integer  (ไม่มีการ overflow หรือ error ใดๆ)
```

### เขียนตัวเลขให้อ่านง่ายด้วย underscore

```ruby
population = 1_000_000       # underscore ใช้คั่นหลักเพื่อให้อ่านง่าย ไม่มีผลต่อค่าจริง
puts population              # => 1000000

hex_value = 0xff             # เลขฐาน 16 (hexadecimal)
octal_value = 0o17           # เลขฐาน 8 (octal)
binary_value = 0b1010        # เลขฐาน 2 (binary)
puts hex_value, octal_value, binary_value  # => 255, 15, 10
```

---

## Step 13: Float — จำนวนทศนิยม และปัญหาความแม่นยำที่ต้องระวัง

### พื้นฐาน

```ruby
price = 19.99
tax_rate = 0.07

puts price.class        # => Float
puts price + tax_rate   # => 20.06 (ในทางทฤษฎี... แต่ระวังปัญหาด้านล่าง!)
```

### ปัญหาความแม่นยำของ Float (Floating-Point Precision)

นี่คือ**กับดักที่สำคัญที่สุด**เรื่องหนึ่งของการเขียนโปรแกรมทุกภาษา ไม่ใช่เฉพาะ Ruby
เพราะคอมพิวเตอร์เก็บเลขทศนิยมในรูปแบบ **binary floating-point (IEEE 754)** ซึ่งไม่สามารถ
แทนค่าทศนิยมบางค่าได้เป๊ะๆ (เหมือนที่เลขฐานสิบไม่สามารถแทนค่า 1/3 ได้เป๊ะๆ เช่นกัน):

```ruby
puts 0.1 + 0.2          # => 0.30000000000000004  (ไม่ใช่ 0.3 ตรงๆ!)
puts 0.1 + 0.2 == 0.3   # => false  (เปรียบเทียบตรงๆ ผิดพลาดได้!)

puts 10.01 - 10.0       # => 0.009999999999999787 (ไม่ใช่ 0.01)
```

**ทำไมถึงเป็นแบบนี้:** เลข 0.1 และ 0.2 ในฐาน 2 (binary) เป็นเลขทศนิยมไม่รู้จบ (คล้ายกับที่
1/3 ในฐาน 10 คือ 0.3333... ไม่รู้จบ) คอมพิวเตอร์จึงต้องปัดเก็บในจำนวนบิตจำกัด ทำให้เกิด
ความคลาดเคลื่อนเล็กน้อยสะสม

**วิธีแก้ปัญหาในทางปฏิบัติ:**

1. **อย่าเปรียบเทียบ Float ด้วย `==` ตรงๆ** ให้เปรียบเทียบว่าค่าห่างกันน้อยกว่า threshold
   ที่ยอมรับได้แทน:

```ruby
a = 0.1 + 0.2
b = 0.3

puts (a - b).abs < 0.0001   # => true (เปรียบเทียบแบบมี tolerance)
```

2. **เมื่อทำงานกับเงิน ให้ใช้ `Integer` (หน่วยสตางค์/เซ็นต์) หรือ `BigDecimal` แทน `Float`**
   เพราะเรื่องเงินต้องการความแม่นยำสัมบูรณ์ ห้ามคลาดเคลื่อนแม้แต่สตางค์เดียว:

```ruby
require "bigdecimal"

price = BigDecimal("19.99")
tax   = BigDecimal("0.07")

puts price + (price * tax)  # => 0.213893e2 (แสดงแบบ BigDecimal notation)
puts (price + (price * tax)).to_f.round(2)  # => 21.39 (ค่าถูกต้องแม่นยำ)

# เทียบกับการใช้ Float ตรงๆ ซึ่งอาจคลาดเคลื่อนสะสมได้เมื่อคำนวณซ้ำหลายรอบ
```

> **แนวคิดสำคัญที่จะพบอีกครั้งตอนเรียน Rails:** ตาราง `orders` หรือ `products` ใน
> database ที่เก็บราคาสินค้า ควรใช้ column type `decimal` (ไม่ใช่ `float`) และในฝั่ง Ruby
> จะได้ค่ากลับมาเป็น `BigDecimal` โดยอัตโนมัติ — เหตุผลคือปัญหาความแม่นยำที่เพิ่งเห็นด้านบนนี้เอง

### Method ที่ใช้บ่อยกับ Float

```ruby
puts 3.14159.round(2)   # => 3.14   (ปัดเศษ ทศนิยม 2 ตำแหน่ง)
puts 3.14159.ceil(2)    # => 3.15   (ปัดขึ้น)
puts 3.14159.floor(2)   # => 3.14   (ปัดลง)
puts 3.14159.truncate(2) # => 3.14  (ตัดทิ้งตรงๆ ไม่ปัด)
puts 7.0.to_i            # => 7     (แปลงเป็น Integer - ตัดทศนิยมทิ้ง ไม่ปัดเศษ!)
puts 7.9.to_i            # => 7     (ระวัง: to_i ไม่ปัดเศษ แค่ตัดทิ้ง)
puts 7.9.round           # => 8     (round ปัดเศษจริงๆ)
puts Float::INFINITY     # => Infinity
puts (0.0 / 0.0).nan?    # => true  (NaN = Not a Number เกิดจากการดำเนินการที่ไม่นิยาม)
```

---

## Step 14: String — การสร้าง string, เครื่องหมายคำพูด, และเรื่อง encoding

### การสร้าง String

```ruby
single = 'สวัสดี'                    # single quote
double = "สวัสดี"                    # double quote
name = "Ruby"
greeting = "สวัสดี #{name}"          # double quote รองรับ string interpolation

puts single
puts double
puts greeting  # => สวัสดี Ruby
```

**ความแตกต่างสำคัญระหว่าง single quote กับ double quote:**

```ruby
name = "Ruby"

puts 'Hello #{name}'   # => Hello #{name}  (single quote ไม่ interpolate ตีความตรงตัว)
puts "Hello #{name}"   # => Hello Ruby     (double quote interpolate ค่าตัวแปรจริง)

puts 'Line1\nLine2'    # => Line1\nLine2   (single quote ไม่ตีความ escape sequence)
puts "Line1\nLine2"    # => Line1
                        #    Line2          (double quote ตีความ \n เป็นขึ้นบรรทัดใหม่จริง)
```

> **แนวปฏิบัติที่ดี:** ใช้ single quote `'...'` เมื่อ string เป็นข้อความตรงๆ ไม่มีการแทรก
> ตัวแปรหรือ escape sequence เพราะเร็วกว่าเล็กน้อย (ไม่ต้อง parse escape/interpolation) และ
> สื่อเจตนาชัดเจนว่า "นี่คือข้อความคงที่" ส่วน double quote `"..."` ใช้เมื่อต้องการ
> interpolation หรือ escape sequence เท่านั้น — นี่คือกฎที่ RuboCop บังคับใช้เป็นค่า default
> (`Style/StringLiterals`)

Step 9 ใน Part 001 ได้แนะนำ `# frozen_string_literal: true` ไปแล้ว มาดูว่ามันมีผลอย่างไรกับ
String โดยตรง:

```ruby
# frozen_string_literal: true

name = "Ruby"
# name << " on Rails"  # => FrozenError: can't modify frozen String
#                             (เพราะ string literal ทุกตัวในไฟล์นี้ถูก freeze อัตโนมัติ)

name = name + " on Rails"  # วิธีนี้ทำได้ - เพราะสร้าง string object ใหม่ ไม่ได้แก้ตัวเดิม
puts name  # => Ruby on Rails
```

### String encoding

Ruby ใช้ **UTF-8** เป็น default encoding ตั้งแต่ Ruby 2.0 เป็นต้นมา ทำให้รองรับภาษาไทย
และ Unicode อื่นๆ ได้โดยไม่ต้องตั้งค่าเพิ่มเติม:

```ruby
text = "สวัสดีชาวโลก"
puts text.encoding         # => UTF-8
puts text.length            # => 12  (นับเป็นตัวอักษร ไม่ใช่ byte)
puts text.bytesize          # => 36  (แต่ละตัวอักษรไทยใช้ 3 byte ใน UTF-8)
puts text.valid_encoding?   # => true
```

**ทำไมเรื่องนี้สำคัญ:** ถ้าเขียนโค้ดที่ต้องอ่านไฟล์จากภายนอก (เช่น ไฟล์ CSV เก่าที่เข้ารหัส
แบบ TIS-620 หรือ Windows-874 ที่พบได้บ่อยในระบบเก่าของไทย) แล้วไม่ระบุ encoding ให้ถูกต้อง
จะได้ error หรือข้อความตัวอักษรเพี้ยน (mojibake) ตัวอย่างการแปลง encoding:

```ruby
# สมมติอ่านไฟล์ที่เข้ารหัสแบบ TIS-620 (encoding เก่าของไทย)
# content = File.read("old_file.txt", encoding: "TIS-620")
# utf8_content = content.encode("UTF-8")

# ตัวอย่างการแปลง encoding ของ string ที่มีอยู่แล้ว
thai_text = "ทดสอบ"
puts thai_text.encoding                        # => UTF-8
force_encoded = thai_text.dup.force_encoding("ASCII-8BIT")
puts force_encoded.encoding                    # => ASCII-8BIT (เปลี่ยน "ป้ายกำกับ" โดยไม่แปลง byte จริง)
```

> **หมายเหตุ:** `force_encoding` แค่เปลี่ยนป้ายกำกับว่า Ruby ควรตีความ byte ชุดนี้ด้วย
> encoding ไหน (ไม่ได้แปลง byte จริง) ในขณะที่ `.encode` จะแปลง byte จริงๆ ให้ตรงกับ
> encoding ใหม่ — เรื่องนี้จะกลับมาเจอบ่อยตอนทำงานกับไฟล์ CSV/Excel ใน Part 012
> (File I/O, CSV, JSON, YAML)

### เทียบ String กับ Array ด้วย method พื้นฐาน

```ruby
s = "hello"
puts s.length      # => 5
puts s.empty?      # => false
puts "".empty?     # => true
puts s[0]          # => h    (index access เหมือน Array)
puts s[0..2]       # => hel  (range access)
puts s[-1]         # => o    (index ติดลบนับจากท้าย)
puts s.reverse     # => olleh
```

> **หมายเหตุ:** method อื่นๆ ของ String เช่น `upcase`, `downcase`, `split`, `strip`,
> `gsub`, string interpolation แบบเจาะลึก และ Heredoc จะสอนอย่างละเอียดใน **Part 003**
> Part นี้ตั้งใจโฟกัสที่ "String คือชนิดข้อมูลแบบไหน" ก่อน แล้วค่อยไปลงลึกเรื่อง method

---

## Step 15: Symbol vs String — ต่างกันอย่างไร และทำไม Rails ชอบใช้ Symbol เป็น key

### Symbol คืออะไร

**Symbol** คือชนิดข้อมูลที่มีลักษณะคล้าย String แต่เขียนด้วยเครื่องหมาย `:` นำหน้า
และมีคุณสมบัติสำคัญที่ต่างจาก String อย่างสิ้นเชิง:

```ruby
sym = :name
str = "name"

puts sym.class   # => Symbol
puts str.class   # => String

puts sym         # => name  (แสดงผลโดยไม่มี : นำหน้า)
p sym            # => :name (inspect จะเห็น : ชัดเจน)
```

### ความแตกต่างสำคัญ: Immutability และ Object Identity

```ruby
# String: ทุกครั้งที่สร้าง string literal เดียวกัน จะได้ object คนละตัวกันใน memory
str1 = "name"
str2 = "name"
puts str1.object_id  # => (เลขหนึ่ง เช่น 60)
puts str2.object_id  # => (เลขอีกตัว เช่น 80 - คนละ object!)
puts str1.equal?(str2)  # => false

# Symbol: Symbol ชื่อเดียวกันจะเป็น object เดียวกันเสมอ (interned/singleton)
sym1 = :name
sym2 = :name
puts sym1.object_id  # => (เลขหนึ่ง เช่น 1234568)
puts sym2.object_id  # => (เลขเดียวกันเป๊ะ!)
puts sym1.equal?(sym2)  # => true
```

**ทำไมถึงต่างกัน:** String เป็น **mutable** (แก้ไขค่าได้ เช่น `<<`, `upcase!`) ดังนั้น
Ruby ต้องสร้าง object ใหม่ทุกครั้งที่เจอ string literal เพื่อป้องกันไม่ให้การแก้ไข string
หนึ่งไปกระทบ string อื่นที่บังเอิญมีค่าเหมือนกัน ในขณะที่ Symbol เป็น **immutable**
(แก้ไขค่าไม่ได้เลยตลอดชีวิตโปรแกรม) Ruby จึงสามารถ "แคช" Symbol ไว้ใช้ object เดียวกันซ้ำได้
ทำให้:

1. **ประหยัดหน่วยความจำ** — ไม่ต้องสร้าง object ใหม่ซ้ำๆ สำหรับค่าเดียวกัน
2. **เปรียบเทียบเร็วกว่า** — การเทียบ Symbol ใช้การเทียบ object_id (เร็วมาก, O(1))
   ในขณะที่การเทียบ String ต้องไล่เทียบตัวอักษรทีละตัว (O(n))

```ruby
require "benchmark"

n = 5_000_000

Benchmark.bm(20) do |x|
  x.report("String comparison:") { n.times { "hello" == "hello" } }
  x.report("Symbol comparison:") { n.times { :hello == :hello } }
end
# ผลลัพธ์โดยประมาณ (จะต่างกันตามเครื่อง):
#                            user     system      total        real
# String comparison:     0.350000   0.000000   0.350000 (  0.352000)
# Symbol comparison:     0.180000   0.000000   0.180000 (  0.181000)
```

### ทำไม Rails (และ Ruby community) นิยมใช้ Symbol เป็น Hash key

```ruby
# Hash key แบบ String
user_string_key = { "name" => "สมชาย", "age" => 30 }

# Hash key แบบ Symbol (นิยมกว่ามากในโค้ด Ruby/Rails สมัยใหม่)
user_symbol_key = { name: "สมชาย", age: 30 }   # syntax แบบย่อของ { :name => "สมชาย", :age => 30 }

puts user_symbol_key[:name]  # => สมชาย
```

**เหตุผลที่ Rails เลือกใช้ Symbol เป็น Hash key แทบทุกที่** (เช่น `params[:id]`,
`render json: { status: "ok" }`, options ของ method ต่างๆ):

1. **ประสิทธิภาพ** — Hash ที่ใช้ Symbol เป็น key เร็วกว่า String key เพราะการ hash
   Symbol เร็วกว่า (ค่า hash ถูกคำนวณครั้งเดียวและ cache ไว้ เพราะ Symbol immutable)
2. **ความหมายชัดเจน (semantic)** — Symbol สื่อความหมายว่า "นี่คือชื่อ/label ที่คงที่
   ไม่ใช่ข้อมูล (data) ที่ผันแปรได้" ในขณะที่ String มักสื่อว่า "นี่คือข้อมูลจริงที่มาจาก
   ผู้ใช้หรือฐานข้อมูล" — การแยกแบบนี้ช่วยให้อ่านโค้ดเข้าใจเจตนาได้ง่ายขึ้น
3. **ป้องกัน Hash key พิมพ์ผิดโดยไม่รู้ตัว** — ถ้าพิมพ์ `:nmae` แทน `:name` อย่างน้อย
   Ruby ก็ยังสร้าง Symbol ใหม่ได้ตามปกติ (ไม่ error ทันที) แต่ในเชิงปฏิบัติ Editor/LSP
   สมัยใหม่และ static analysis tool จะช่วยตรวจจับได้ง่ายกว่าถ้าใช้ Symbol เป็น key
   แบบตายตัว

```ruby
# ตัวอย่างที่จะคุ้นเคยมากเมื่อเรียน Rails ในเฟส 3
# params เป็น Hash ที่ Rails ส่งมาให้ controller โดยใช้ Symbol เป็น key เสมอ
# params[:id]              # => "42"  (ค่าที่ได้จาก URL/form เป็น String)
# render json: { status: "success", data: users }   # keyword ของ options เป็น Symbol
```

### แปลงไปมาระหว่าง String กับ Symbol

```ruby
str = "hello"
sym = str.to_sym    # String -> Symbol
puts sym.class      # => Symbol

sym2 = :world
str2 = sym2.to_s     # Symbol -> String
puts str2.class      # => String

# ข้อควรระวัง: ห้าม mutate Symbol - Symbol ไม่มี method ที่ลงท้ายด้วย ! เพราะแก้ไขค่าไม่ได้
# :hello.upcase!  # => NoMethodError: undefined method 'upcase!' for :hello:Symbol
puts :hello.upcase  # => HELLO (ได้ Symbol ใหม่ ไม่ใช่แก้ตัวเดิม)
```

> **กฎเลือกใช้ในทางปฏิบัติ:** ใช้ **Symbol** เมื่อค่าคงที่ตายตัว รู้ล่วงหน้าตอนเขียนโค้ด
> (hash key, method name, option flag เช่น `:asc`/`:desc`) ใช้ **String** เมื่อเป็นข้อมูล
> ที่มาจากภายนอก เปลี่ยนแปลงได้ หรือต้องประมวลผลด้วย string method (ชื่อผู้ใช้, ข้อความ,
> เนื้อหาจากฟอร์ม, ข้อมูลจากฐานข้อมูล)

---

## Step 16: nil — ค่า "ไม่มีอะไร" และพฤติกรรมของมัน

### nil คืออะไร

`nil` คือ object พิเศษที่แทน **"การไม่มีค่า"** หรือ "ความว่างเปล่า" ใน Ruby ทุกอย่างใน
Ruby เป็น object รวมถึง `nil` ด้วย — `nil` เป็น instance เดียวของ class `NilClass`:

```ruby
value = nil
puts value.class     # => NilClass
puts value.nil?      # => true
puts value.inspect   # => nil

x = 5
puts x.nil?          # => false
```

### เมื่อไหร่ที่ Ruby คืนค่า nil ให้อัตโนมัติ

```ruby
# 1. ตัวแปรที่ประกาศไว้แต่ยังไม่ได้กำหนดค่า (ในบาง context)
if false
  x = 10
end
puts x  # => nil (x ถูก "ประกาศ" จากการที่ Ruby เห็น x = 10 ใน source code
        #          แม้ branch นั้นไม่ถูกรัน ก็ยังได้ nil ไม่ใช่ NameError)

# 2. การเข้าถึง index/key ที่ไม่มีอยู่จริงใน Array/Hash
arr = [1, 2, 3]
puts arr[10]           # => nil (ไม่ error แค่คืน nil)

hash = { name: "Ruby" }
puts hash[:age]         # => nil (key ไม่มี ก็คืน nil)

# 3. method ที่ไม่พบผลลัพธ์
puts [1, 2, 3].find { |n| n > 100 }  # => nil (หาไม่เจอ คืน nil)

# 4. method ที่ไม่ได้ตั้งใจ return อะไร (เช่น puts เอง)
result = puts "hello"
p result  # => nil (puts คืนค่า nil เสมอ)
```

### NoMethodError ที่พบบ่อยที่สุดในชีวิตนักพัฒนา Ruby

```ruby
user = nil
# puts user.name
# => NoMethodError: undefined method 'name' for nil
```

นี่คือ error ที่นักพัฒนา Ruby/Rails ทุกคนต้องเจอบ่อยที่สุดในชีวิต เรียกกันติดปากว่า
**"NoMethodError on nil"** เกิดจากการพยายามเรียก method บน object ที่กลายเป็น `nil`
โดยไม่คาดคิด (เช่น หา record ในฐานข้อมูลไม่เจอ, key ใน Hash ไม่มีอยู่จริง)

**วิธีป้องกันและจัดการ nil ในทางปฏิบัติ:**

```ruby
user_name = nil

# วิธีที่ 1: เช็คด้วย nil? ก่อน
if user_name.nil?
  puts "ไม่พบชื่อผู้ใช้"
else
  puts user_name.upcase
end

# วิธีที่ 2: ใช้ safe navigation operator &. (แนะนำ - นิยมมากใน Rails)
puts user_name&.upcase  # => nil (ไม่ error! ถ้า user_name เป็น nil จะคืน nil เลยทันที)

# วิธีที่ 3: กำหนดค่า default ด้วย ||
display_name = user_name || "ผู้ใช้นิรนาม"
puts display_name  # => ผู้ใช้นิรนาม

# วิธีที่ 4: ใช้ &.  ต่อกันหลาย method (เรียกว่า "safe navigation chaining")
address = nil
puts address&.city&.upcase  # => nil (หยุดทันทีตั้งแต่ address เป็น nil ไม่ error)
```

> **แนวคิดสำคัญ:** operator `&.` (safe navigation operator, มาตั้งแต่ Ruby 2.3) เป็น
> syntax ที่ใช้บ่อยที่สุดตัวหนึ่งในโค้ด Rails ระดับ production ช่วยลดโค้ดแบบ
> `if x.nil? ... else x.method ... end` ที่ยืดยาวให้เหลือบรรทัดเดียว — จะใช้บ่อยมากเมื่อ
> ดึงข้อมูลจาก association ของ ActiveRecord เช่น `order&.customer&.email`

### ระวัง: nil ไม่เท่ากับ false, 0, หรือ "" (empty string)

```ruby
puts nil == false   # => false (คนละ object กัน คนละความหมายกัน)
puts nil == 0       # => false
puts nil == ""      # => false

puts nil.to_s       # => ""   (แปลงเป็น string ว่าง)
puts nil.to_a       # => []   (แปลงเป็น array ว่าง)
puts nil.to_i       # => 0    (แปลงเป็น 0)
```

การที่ `nil.to_s == ""` แต่ `nil != ""` เป็นจุดที่ทำให้มือใหม่สับสนบ่อย — ต้องแยกให้ออกว่า
**"แปลงได้เป็นค่านั้น" ไม่ได้แปลว่า "เท่ากับค่านั้น"**

---

## Step 17: true/false และ Truthiness — อะไรคือ falsy ใน Ruby

### true และ false เป็น object

```ruby
puts true.class    # => TrueClass
puts false.class   # => FalseClass

puts true & false  # => false  (logical AND แบบ method call)
puts true | false  # => true   (logical OR)
puts true ^ true    # => false (logical XOR)
```

### Truthiness — กฎที่สำคัญที่สุดข้อหนึ่งของ Ruby

นี่คือกฎที่**ต่างจากภาษาโปรแกรมมิ่งอื่นอย่างชัดเจน** และเป็นสิ่งที่ทำให้มือใหม่ (โดยเฉพาะ
คนที่มาจาก JavaScript, Python, PHP) สับสนบ่อยที่สุด:

> **ใน Ruby มีเพียง 2 ค่าเท่านั้นที่เป็น falsy (ถือว่าเป็นเท็จใน condition): `nil` และ
> `false` — ค่าอื่นทุกตัว "ถือว่าเป็นจริง (truthy)" หมดเลย ไม่มีข้อยกเว้น**

```ruby
puts "เป็นจริง" if 0            # => เป็นจริง!  (ต่างจาก JS/Python/PHP ที่ 0 เป็น falsy)
puts "เป็นจริง" if ""           # => เป็นจริง!  (string ว่างก็ยังเป็น truthy)
puts "เป็นจริง" if []           # => เป็นจริง!  (array ว่างก็ยังเป็น truthy)
puts "เป็นจริง" if {}           # => เป็นจริง!  (hash ว่างก็ยังเป็น truthy)
puts "เป็นจริง" if "false"      # => เป็นจริง!  (string ที่มีคำว่า "false" อยู่ข้างในก็ยัง truthy)

puts "เป็นเท็จ" if nil   # (ไม่แสดงอะไร เพราะ nil เป็น falsy)
puts "เป็นเท็จ" if false # (ไม่แสดงอะไร เพราะ false เป็น falsy)
```

**ตารางสรุปให้จำง่ายๆ:**

| ค่า | Truthy หรือ Falsy |
|-----|-------------------|
| `nil` | **Falsy** |
| `false` | **Falsy** |
| `0` | Truthy (ต่างจากภาษาอื่นมาก ระวัง!) |
| `""` (empty string) | Truthy |
| `[]` (empty array) | Truthy |
| `{}` (empty hash) | Truthy |
| ค่าอื่นๆ ทั้งหมด | Truthy |

### ผลกระทบในทางปฏิบัติ

```ruby
count = 0

# ผิด! ถ้าตั้งใจเช็คว่า count เป็น 0 หรือไม่ แล้วเขียนแบบนี้จะไม่มีวันเข้า else
if count
  puts "count มีค่า (เข้าเงื่อนไขนี้เสมอ ไม่ว่า count จะเป็น 0 หรือค่าอะไรก็ตาม)"
else
  puts "count เป็น nil หรือ false เท่านั้น (โค้ดส่วนนี้จะไม่มีวันทำงานถ้า count = 0)"
end
# => count มีค่า (เข้าเงื่อนไขนี้เสมอ ไม่ว่า count จะเป็น 0 หรือค่าอะไรก็ตาม)

# ถูกต้อง! ถ้าต้องการเช็คว่าเป็น 0 หรือไม่ ต้องเทียบตรงๆ
if count == 0
  puts "count เท่ากับ 0 พอดี"
end
# => count เท่ากับ 0 พอดี

# ถูกต้อง! ถ้าต้องการเช็คว่า array ว่างหรือไม่ ต้องใช้ empty? ไม่ใช่ความเป็น truthy
items = []
if items.empty?
  puts "ไม่มีสินค้าในตะกร้า"
end
# => ไม่มีสินค้าในตะกร้า
```

> **บทเรียนสำคัญ:** เมื่อมาจากภาษาอื่น (โดยเฉพาะ JavaScript/Python ที่ `0`, `""`, `[]`
> ล้วนเป็น falsy) ต้องปรับความเข้าใจใหม่ทั้งหมด — ใน Ruby ถ้าต้องการเช็คว่าตัวแปรว่างเปล่า
> หรือเป็นศูนย์ ต้องเช็คแบบเจาะจง (`== 0`, `.empty?`, `.zero?`) ไม่ใช่พึ่งพา truthiness
> เพียงอย่างเดียว — ความเข้าใจผิดเรื่องนี้เป็นสาเหตุของบั๊กที่พบบ่อยมากในทีมที่มีนักพัฒนา
> ย้ายมาจากภาษาอื่น

### Operator `&&`, `||`, `!` และค่าที่คืนกลับมาจริงๆ

```ruby
# && และ || ไม่ได้คืนค่า true/false เสมอไป แต่คืน "operand ตัวที่ตัดสินผลลัพธ์"
puts (nil || "default")     # => default  (nil เป็น falsy จึงไปประเมินฝั่งขวา คืนค่าฝั่งขวา)
puts ("hello" || "default") # => hello    ("hello" เป็น truthy จึงคืนค่าฝั่งซ้ายทันที ไม่แตะฝั่งขวา)
puts (nil && "default")     # => nil      (nil เป็น falsy จึงคืนค่า nil ทันที ไม่แตะฝั่งขวา)
puts ("hello" && "default") # => default  ("hello" เป็น truthy จึงไปประเมินฝั่งขวา คืนค่าฝั่งขวา)

puts !nil       # => true
puts !false     # => true
puts !0         # => false (0 เป็น truthy ดังนั้น !0 คือ false)
puts !""        # => false (string ว่างเป็น truthy เช่นกัน)
```

พฤติกรรมนี้เองที่ทำให้ pattern `ตัวแปร || ค่า_default` (ที่เห็นใน `greet.rb` ตอน Part 001
Step 7: `name = ARGV[0] || "World"`) ทำงานได้ — เพราะ `||` คืนค่า operand ตัวแรกที่เป็น
truthy นั่นเอง

---

## Step 18: การแปลงชนิดข้อมูลแบบ implicit method (`to_s`, `to_i`, `to_f`, `to_sym`, `to_a`)

Ruby มี method ตระกูล `to_*` สำหรับแปลงชนิดข้อมูลไปมา จุดเด่นของ method กลุ่มนี้คือ
**"พยายามแปลงให้ได้เสมอ ไม่ error แม้แปลงไม่สำเร็จจริง"** (คืนค่า default ที่สมเหตุสมผล
แทน)

### `to_s` — แปลงเป็น String

```ruby
puts 42.to_s          # => "42"
puts 3.14.to_s        # => "3.14"
puts nil.to_s         # => ""
puts true.to_s        # => "true"
puts :symbol.to_s     # => "symbol"
puts [1, 2, 3].to_s   # => "[1, 2, 3]"
puts({a: 1}.to_s)     # => "{a: 1}"
```

### `to_i` — แปลงเป็น Integer (แบบ "ยอมความ" ไม่ error)

```ruby
puts "42".to_i         # => 42
puts "42abc".to_i      # => 42     (อ่านได้เท่าไหน่ก็เอาเท่านั้น หยุดที่ตัวอักษรตัวแรก)
puts "abc".to_i        # => 0      (แปลงไม่ได้เลย -> คืน 0 แทน ไม่ error!)
puts "  42  ".to_i     # => 42     (ตัด whitespace รอบข้างให้อัตโนมัติ)
puts "3.99".to_i       # => 3      (หยุดที่จุดทศนิยม ไม่ปัดเศษ)
puts 3.99.to_i         # => 3      (ตัดทศนิยมทิ้ง ไม่ปัด - เหมือนที่เห็นใน Step 13)
puts nil.to_i          # => 0
```

### `to_f` — แปลงเป็น Float (พฤติกรรมคล้าย `to_i`)

```ruby
puts "3.14".to_f       # => 3.14
puts "3.14abc".to_f    # => 3.14
puts "abc".to_f        # => 0.0    (แปลงไม่ได้ -> คืน 0.0 แทน ไม่ error!)
puts "  3.14  ".to_f   # => 3.14
```

### `to_sym` — แปลงเป็น Symbol

```ruby
puts "hello".to_sym    # => :hello
puts "hello world".to_sym  # => :"hello world"  (มี space จะแสดงด้วย quote ครอบ)
```

### `to_a` — แปลงเป็น Array

```ruby
puts nil.to_a          # => []
puts (1..5).to_a       # => [1, 2, 3, 4, 5]  (แปลง Range เป็น Array)
puts({a: 1, b: 2}.to_a) # => [[:a, 1], [:b, 2]]  (Hash แปลงเป็น array ของ key-value pair)
```

### สรุปข้อควรระวังสำคัญของ method ตระกูล `to_*`

```ruby
# อันตราย! to_i ไม่บอกเลยว่าแปลงไม่สำเร็จ ทำให้บั๊กแอบซ่อนได้ง่าย
user_input = "abc"       # สมมติผู้ใช้พิมพ์ผิด ตั้งใจจะใส่ตัวเลข
quantity = user_input.to_i
puts quantity             # => 0  (ไม่ error ทำให้โปรแกรมเดินหน้าต่อด้วยค่าที่ผิดเงียบๆ)

# ถ้าโค้ดถัดไปเอา quantity ไปคำนวณราคาต่อ จะได้ผลลัพธ์ผิดโดยไม่มีการแจ้งเตือนใดๆ
price_per_item = 100
total = quantity * price_per_item
puts total  # => 0  (ผิดแบบเงียบๆ - ไม่รู้เลยว่าเกิดจาก input ผิด หรือของจริงคือ 0 ชิ้น)
```

นี่คือเหตุผลว่าทำไมเราต้องรู้จัก method อีกชุดหนึ่งที่ทำงาน "เข้มงวดกว่า" — นั่นคือ
`Integer()` และ `Float()` ที่จะพูดถึงใน Step ถัดไป

---

## Step 19: `Integer()` และ `Float()` — แปลงชนิดข้อมูลแบบเข้มงวด ต่างจาก `to_i`/`to_f` อย่างไร

### ความแตกต่างหลัก: error เมื่อแปลงไม่ได้

`Integer()` และ `Float()` (สังเกตว่าเป็นตัวพิมพ์ใหญ่ขึ้นต้น และเรียกแบบ method
ไม่ใช่แบบ `.to_i`) เป็น **Kernel method** ที่ทำงานตรงข้ามกับ `to_i`/`to_f` โดยสิ้นเชิงใน
แง่การจัดการ error:

```ruby
puts Integer("42")     # => 42
puts Float("3.14")     # => 3.14

# แต่ถ้าแปลงไม่ได้ จะ raise error ทันที ไม่ยอมความแบบเงียบๆ!
begin
  Integer("abc")
rescue ArgumentError => e
  puts "แปลงไม่ได้: #{e.message}"
  # => แปลงไม่ได้: invalid value for Integer(): "abc"
end

begin
  Integer("42abc")
rescue ArgumentError => e
  puts "แปลงไม่ได้: #{e.message}"
  # => แปลงไม่ได้: invalid value for Integer(): "42abc"
  #    (สังเกตว่า Integer() เข้มงวดกว่า to_i มาก - "42abc" ก็ยังถือว่าแปลงไม่ได้)
end
```

**ตารางเปรียบเทียบให้เห็นภาพชัดเจน:**

| Input | `.to_i` | `Integer()` |
|-------|---------|-------------|
| `"42"` | `42` | `42` |
| `"42abc"` | `42` | raise `ArgumentError` |
| `"abc"` | `0` | raise `ArgumentError` |
| `""` | `0` | raise `ArgumentError` |
| `nil` | `0` | raise `TypeError` |
| `"  42  "` | `42` | `42` (ตัด whitespace รอบข้างได้เหมือนกัน) |

### เมื่อไหร่ควรใช้ตัวไหน

```ruby
# ใช้ .to_i / .to_f เมื่อ:
# - ยอมรับให้ผิดพลาดแบบเงียบๆ ได้ หรือมี default ที่สมเหตุสมผลอยู่แล้ว
# - แน่ใจอยู่แล้วว่า input มาจากแหล่งที่เชื่อถือได้ (เช่นแปลงจาก Float เป็น Integer)
rounded_down = 3.99.to_i  # ใช้ .to_i แบบนี้สมเหตุสมผล ไม่มีปัญหา

# ใช้ Integer() / Float() เมื่อ:
# - ต้องการ validate ข้อมูลที่มาจากภายนอก (user input, API response, ไฟล์ CSV)
#   และต้องการรู้ทันทีว่า "ข้อมูลนี้ผิดรูปแบบ" แทนที่จะปล่อยผ่านไปแบบเงียบๆ

print "กรุณาใส่จำนวนสินค้า: "
input = gets.chomp

begin
  quantity = Integer(input)
  puts "จำนวนสินค้าที่ถูกต้อง: #{quantity}"
rescue ArgumentError
  puts "ข้อผิดพลาด: กรุณาใส่ตัวเลขจำนวนเต็มเท่านั้น (คุณใส่: '#{input}')"
end
```

> **แนวคิดสำคัญ (จะเจอบ่อยเมื่อเรียน Rails):** หลักการ **"fail fast"** (ทำให้โปรแกรม
> ล้มเหลวทันทีเมื่อข้อมูลผิดปกติ แทนที่จะปล่อยให้ทำงานต่อด้วยข้อมูลผิดๆ) เป็นแนวคิดที่ดี
> ในการเขียนโปรแกรมระดับมืออาชีพ — เมื่อรับข้อมูลจากภายนอก (ผู้ใช้, API, ไฟล์) ควรพิจารณา
> ใช้ `Integer()`/`Float()` ร่วมกับ `begin/rescue` (จะสอนละเอียดใน Part 011: Exception
> Handling) แทนการใช้ `to_i`/`to_f` เงียบๆ โดยเฉพาะเมื่อความถูกต้องของข้อมูลมีผลกระทบสูง
> (เช่น จำนวนเงิน, จำนวนสินค้าในคำสั่งซื้อ)

### `Integer()` รองรับการระบุฐานเลขด้วย

```ruby
puts Integer("ff", 16)     # => 255  (แปลงจาก hex string)
puts Integer("0xff", 16)   # => 255  (มี prefix 0x ก็รับได้)
puts Integer("1010", 2)    # => 10   (แปลงจาก binary string)

# ข้อควรระวัง: Integer() default จะตีความ "0" นำหน้าเป็นเลขฐาน 8 (octal) โดยอัตโนมัติ!
puts Integer("010")        # => 8  (ตีความเป็นฐาน 8 ไม่ใช่สิบ! ต้องระวังถ้า input มี 0 นำหน้า)
puts Integer("010", 10)    # => 10 (ถ้าต้องการฐาน 10 ตรงๆ ต้องระบุฐานให้ชัดเจน)
```

> **กับดักในโลกจริง:** ถ้าเขียนฟอร์มรับ "รหัสไปรษณีย์" หรือ "เลขที่บ้าน" ที่อาจขึ้นต้นด้วย
> เลข 0 (เช่น "010") แล้วใช้ `Integer(input)` ตรงๆ โดยไม่ระบุฐาน จะได้ผลลัพธ์ผิดพลาด
> เพราะ Ruby จะตีความเป็นเลขฐาน 8 ให้อัตโนมัติ วิธีแก้คือระบุฐาน 10 ให้ชัดเจนเสมอ:
> `Integer(input, 10)` หรือใช้ `.to_i` แทนถ้าไม่ต้องการ validate เข้มงวด

---

## Step 20: แบบฝึกหัดโปรเจกต์ — โปรแกรมคำนวณใบเสร็จร้านค้าเล็กๆ

### โจทย์

เขียนโปรแกรม `receipt.rb` ที่ทำงานดังนี้:

1. ถามชื่อสินค้า, ราคาต่อชิ้น (เป็นทศนิยม), และจำนวนที่ซื้อ (เป็นจำนวนเต็ม)
2. ตรวจสอบว่าราคาและจำนวนที่กรอกเป็นตัวเลขที่ถูกต้อง (ใช้ `Float()`/`Integer()` แบบเข้มงวด
   พร้อม `rescue` เพื่อแจ้ง error ที่ชัดเจนถ้ากรอกผิด)
3. คำนวณราคารวมก่อนภาษี, ภาษีมูลค่าเพิ่ม (VAT 7%), และราคารวมสุทธิ
4. ใช้ `Symbol` เป็น key ของ Hash ที่เก็บข้อมูลใบเสร็จ (เพื่อฝึกใช้ Symbol ตามที่เรียนมา)
5. แสดงใบเสร็จให้อ่านง่าย จัดรูปแบบราคาเป็นทศนิยม 2 ตำแหน่งเสมอ

### เฉลย

```ruby
# frozen_string_literal: true

# receipt.rb
VAT_RATE = 0.07  # Constant - อัตราภาษีมูลค่าเพิ่ม 7% (ตามธรรมเนียม constant เขียนตัวพิมพ์ใหญ่)

print "ชื่อสินค้า: "
product_name = gets.chomp

# ใช้ loop เพื่อถามซ้ำจนกว่าจะได้ราคาที่ถูกต้อง (แนวคิด loop จะสอนละเอียดใน Part 006)
unit_price = nil
until unit_price
  print "ราคาต่อชิ้น (บาท): "
  input = gets.chomp
  begin
    unit_price = Float(input)
  rescue ArgumentError
    puts "  -> ผิดพลาด: กรุณาใส่ตัวเลข เช่น 25.50 (คุณใส่: '#{input}')"
  end
end

quantity = nil
until quantity
  print "จำนวนที่ซื้อ (ชิ้น): "
  input = gets.chomp
  begin
    quantity = Integer(input, 10)
  rescue ArgumentError
    puts "  -> ผิดพลาด: กรุณาใส่จำนวนเต็ม เช่น 3 (คุณใส่: '#{input}')"
  end
end

# ใช้ Symbol เป็น key ของ Hash ตามธรรมเนียมของ Ruby/Rails
receipt = {
  product_name: product_name,
  unit_price: unit_price,
  quantity: quantity
}

subtotal = receipt[:unit_price] * receipt[:quantity]
vat_amount = subtotal * VAT_RATE
grand_total = subtotal + vat_amount

puts "=" * 40
puts "ใบเสร็จรับเงิน"
puts "=" * 40
puts "สินค้า:      #{receipt[:product_name]}"
puts "ราคาต่อชิ้น: #{format('%.2f', receipt[:unit_price])} บาท"
puts "จำนวน:       #{receipt[:quantity]} ชิ้น"
puts "-" * 40
puts "ราคารวม:     #{format('%.2f', subtotal)} บาท"
puts "VAT (7%):    #{format('%.2f', vat_amount)} บาท"
puts "-" * 40
puts "รวมสุทธิ:    #{format('%.2f', grand_total)} บาท"
puts "=" * 40
```

ทดสอบรัน:

```bash
ruby receipt.rb
# ชื่อสินค้า: กาแฟดริป
# ราคาต่อชิ้น (บาท): abc
#   -> ผิดพลาด: กรุณาใส่ตัวเลข เช่น 25.50 (คุณใส่: 'abc')
# ราคาต่อชิ้น (บาท): 65.50
# จำนวนที่ซื้อ (ชิ้น): 3
# ========================================
# ใบเสร็จรับเงิน
# ========================================
# สินค้า:      กาแฟดริป
# ราคาต่อชิ้น: 65.50 บาท
# จำนวน:       3 ชิ้น
# ----------------------------------------
# ราคารวม:     196.50 บาท
# VAT (7%):    13.76 บาท
# ----------------------------------------
# รวมสุทธิ:    210.26 บาท
# ========================================
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `Float()`/`Integer()` แบบเข้มงวดร่วมกับ `begin/rescue` — ป้องกันไม่ให้ข้อมูลผิดรูปแบบ
  หลุดเข้าไปคำนวณแบบเงียบๆ (ตามหลักการ "fail fast" ที่พูดถึงใน Step 19) ต่างจากการใช้
  `to_f`/`to_i` ตรงๆ ที่จะได้ `0.0`/`0` แบบไม่รู้ตัวถ้าผู้ใช้กรอกผิด
- `Integer(input, 10)` — ระบุฐาน 10 อย่างชัดเจน เพื่อป้องกันปัญหาเลข `0` นำหน้าที่อธิบาย
  ไว้ใน Step 19 (เช่น ถ้าผู้ใช้กรอกจำนวน "010" โดยไม่ตั้งใจ)
- `Symbol` (`:product_name`, `:unit_price`, `:quantity`) เป็น Hash key — ตามธรรมเนียม
  ที่อธิบายไว้ใน Step 15
- `format('%.2f', number)` — จัดรูปแบบตัวเลขทศนิยมให้แสดงผลเป็นทศนิยม 2 ตำแหน่งเสมอ
  (แม้ค่าจะลงตัวพอดี เช่น `196.5` ก็จะแสดงเป็น `196.50`) — วิธีนี้แม่นยำกว่าการใช้ `.round(2)`
  เพียงอย่างเดียว เพราะ `.round(2)` จะไม่เติมเลข 0 ต่อท้ายให้ (เช่น `196.5.round(2)` ยังคง
  ได้ `196.5` ไม่ใช่ `196.50`) — เรื่อง `format`/`sprintf` แบบละเอียดจะสอนต่อใน Part 003

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. แก้โปรแกรมให้รองรับการซื้อหลายสินค้าในใบเสร็จเดียว (ใบ้: เก็บแต่ละสินค้าเป็น Hash
   แยกกัน แล้วถามผู้ใช้ว่า "ต้องการเพิ่มสินค้าอีกหรือไม่ (y/n)" วนซ้ำจนกว่าจะตอบ "n" —
   ยังไม่ต้องใช้ Array ก็ได้ในตอนนี้ เก็บผลรวมสะสมไว้ในตัวแปรก็พอ เพราะ Array จะสอนเต็มๆ
   ใน Part 004)
2. เพิ่มการรับส่วนลด (discount) เป็นเปอร์เซ็นต์ (0-100) โดยต้อง validate ว่าค่าที่กรอก
   อยู่ในช่วง 0-100 เท่านั้น ถ้าไม่อยู่ในช่วงให้แจ้งเตือนและถามใหม่
3. ทดลองเขียนโปรแกรมเล็กๆ ชื่อ `truthiness_quiz.rb` ที่ประกาศตัวแปรค่าต่างๆ (`0`, `""`,
   `[]`, `nil`, `false`, `"0"`) แล้วใช้ `if` เช็คทีละตัวว่า truthy หรือ falsy พร้อม `puts`
   อธิบายผลลัพธ์ — ทำเพื่อฝึกความเข้าใจเรื่อง Truthiness ใน Step 17 ให้แม่นยำจนไม่ลืม

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ตัวแปรใน Ruby เป็น reference ที่ชี้ไปยัง object ไม่ใช่กล่องเก็บค่าโดยตรง และมีธรรมเนียม
  การตั้งชื่อ (`snake_case`, `SCREAMING_SNAKE_CASE`, `PascalCase`) ที่ต้องยึดตามตลอดหลักสูตร
- `Integer` ใน Ruby ไม่มี overflow และการหารระหว่าง Integer จะได้ผลลัพธ์เป็น Integer เสมอ
  (ตัดทศนิยมทิ้ง)
- `Float` มีปัญหาความแม่นยำโดยธรรมชาติ (`0.1 + 0.2 != 0.3`) จึงห้ามใช้ `Float` กับการคำนวณ
  เรื่องเงิน ควรใช้ `BigDecimal` หรือหน่วยจำนวนเต็มแทน
- `String` เป็น mutable, รองรับ UTF-8 โดย default, และมีความแตกต่างสำคัญระหว่าง single
  quote กับ double quote (interpolation, escape sequence)
- `Symbol` เป็น immutable และถูก cache เป็น object เดียวกันเสมอ ทำให้เร็วกว่าและประหยัด
  หน่วยความจำกว่า `String` — นี่คือเหตุผลที่ Rails นิยมใช้ Symbol เป็น Hash key ทุกที่
- `nil` คือค่าที่แทน "ไม่มีอะไร" และเป็นต้นตอของ error ที่พบบ่อยที่สุดในโลก Ruby
  (`NoMethodError` on nil) — มี safe navigation operator `&.` ช่วยจัดการ
- Truthiness ใน Ruby มีแค่ `nil` และ `false` เท่านั้นที่เป็น falsy — `0`, `""`, `[]` ล้วน
  เป็น truthy ทั้งหมด (ต่างจากภาษาอื่นอย่างชัดเจน)
- การแปลงชนิดข้อมูลมีสองสไตล์: `to_i`/`to_f`/`to_s`/`to_sym`/`to_a` (แปลงแบบยอมความ
  ไม่ error) กับ `Integer()`/`Float()` (แปลงแบบเข้มงวด raise error เมื่อแปลงไม่ได้) —
  ต้องเลือกใช้ให้เหมาะกับสถานการณ์ โดยเฉพาะเมื่อรับข้อมูลจากภายนอก

**ต่อไป (Part 003):** เราจะเจาะลึกเรื่อง String methods อย่างละเอียด (การแปลงตัวพิมพ์,
`split`, `strip`, `gsub` และอื่นๆ), string interpolation แบบเชิงลึก, การเขียนข้อความ
หลายบรรทัดด้วย Heredoc, และพื้นฐาน Regular Expression สำหรับตรวจสอบรูปแบบข้อความ
(เช่น ตรวจสอบอีเมล, เบอร์โทรศัพท์) ซึ่งเป็นทักษะที่ใช้แทบทุกวันในการเขียน Rails
