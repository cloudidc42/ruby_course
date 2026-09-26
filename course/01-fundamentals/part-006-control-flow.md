# Part 006: Control Flow — if/unless/case, ternary, loop, while, until, for, break/next/redo

> **Step ครอบคลุมใน Part นี้:** Step 51–60
> **ระดับ:** เริ่มต้น–กลาง (ต้องผ่าน Part 001–005 มาก่อน โดยเฉพาะเรื่อง Array/Hash และ truthiness เบื้องต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

Part นี้ว่าด้วยเรื่อง **control flow** — กลไกที่ทำให้โปรแกรมตัดสินใจ (decision) และทำงานซ้ำ
(iteration) แม้ Part 004 จะแนะนำ `each`, `map` ไปแล้ว แต่นั่นคือ iteration ผ่าน method ของ
Enumerable ส่วน Part นี้จะเจาะลึก **keyword ควบคุมการทำงานระดับภาษา** ที่เป็นรากฐานของทุก
โปรแกรม ไม่ว่าจะเขียน Ruby ธรรมดาหรือ Rails ก็ตาม

## สารบัญของ Part นี้

- Step 51: `if` / `elsif` / `else` และการทบทวน truthiness ใน Ruby
- Step 52: `unless` และทำไมควรเลี่ยง `unless...else`, modifier `if`/`unless`
- Step 53: Ternary operator (`condition ? a : b`)
- Step 54: `case`/`when` พื้นฐาน — เทียบค่าด้วย `===`, หลายเงื่อนไขใน `when` เดียว
- Step 55: `case`/`when` ขั้นสูง — Range, Regexp, Class และ `case` แบบไม่มี argument
- Step 56: `loop do...end`, `break`, และการคืนค่าจาก loop ด้วย `break value`
- Step 57: `while` / `until` — block form และ modifier form
- Step 58: `for` loop และทำไม Rubyist ส่วนใหญ่เลี่ยงมันแล้วใช้ `each` แทน
- Step 59: ความหมายของ `break` / `next` / `redo` แบบเจาะลึกพร้อมตัวอย่างเทียบกัน
- Step 60: แบบฝึกหัด — เกมทายตัวเลข (Number Guessing Game) และ FizzBuzz ด้วย `case/when`

---

## Step 51: `if` / `elsif` / `else` และการทบทวน truthiness ใน Ruby

### โครงสร้างพื้นฐานของ `if`

```ruby
# frozen_string_literal: true

age = 20

if age >= 18
  puts "คุณเป็นผู้ใหญ่"
end
# => คุณเป็นผู้ใหญ่
```

**สังเกต:** Ruby ไม่ต้องใส่วงเล็บ `()` รอบเงื่อนไข (ใส่ได้แต่ไม่นิยม) และปิดบล็อกด้วย `end`
เสมอ ไม่ใช้ `{}` เหมือน C/JavaScript

### `if...elsif...else` แบบเต็มรูปแบบ

```ruby
score = 75

if score >= 90
  puts "เกรด A"
elsif score >= 80
  puts "เกรด B"
elsif score >= 70
  puts "เกรด C"
elsif score >= 60
  puts "เกรด D"
else
  puts "เกรด F"
end
# => เกรด C
```

**อธิบาย:**

- สะกดว่า `elsif` (ไม่มี `e` ตัวที่สอง) — เป็นจุดพลาดยอดฮิตของคนมาจากภาษาอื่น (Python ใช้
  `elif`, หลายภาษาใช้ `else if`)
- ใส่ `elsif` กี่อันก็ได้ และ `else` จะไม่ใส่ก็ได้ (ถ้าไม่มีเงื่อนไขไหน match และไม่มี `else`
  ทั้งนิพจน์จะคืนค่า `nil`)

### `if` เป็น **expression** ไม่ใช่แค่ statement

จุดที่ Ruby ต่างจากหลายภาษาอย่างชัดเจน: `if` มีค่าเป็นผลลัพธ์ (return value) ได้ ทำให้เขียน
แบบนี้ได้ทันที โดยไม่ต้องมี ternary หรือใช้ตัวแปรกลาง:

```ruby
score = 75

grade =
  if score >= 90
    "A"
  elsif score >= 70
    "C"
  else
    "F"
  end

puts grade
# => C
```

ค่าที่ `if` คืนคือค่าของบรรทัดสุดท้ายที่ถูกรันใน branch ที่ match — เหมือนกับที่ method คืนค่า
จากบรรทัดสุดท้ายโดยไม่ต้องมี `return` (จะพูดถึงละเอียดใน Part 007)

### ทบทวน Truthiness ใน Ruby

กติกาของ Ruby ง่ายมาก และ **ต่างจากภาษาอื่นอย่างมีนัยสำคัญ**:

> ค่าทุกค่าเป็น truthy ยกเว้น `nil` และ `false` เท่านั้น

```ruby
if 0
  puts "0 เป็น truthy"  # => พิมพ์บรรทัดนี้! (ต่างจาก C/JS/Python ที่ 0 เป็น falsy)
end

if ""
  puts "string ว่างก็เป็น truthy"  # => พิมพ์บรรทัดนี้! (ต่างจาก JS/Python)
end

if []
  puts "array ว่างก็เป็น truthy"  # => พิมพ์บรรทัดนี้! (ต่างจาก JS/PHP)
end

if nil
  puts "จะไม่ถูกพิมพ์"
end

if false
  puts "จะไม่ถูกพิมพ์"
end
```

**ทำไมเรื่องนี้สำคัญมาก:** โปรแกรมเมอร์ที่มาจาก JavaScript/Python มักเผลอเขียน
`if some_array.length` เพื่อเช็คว่า array ว่างหรือไม่ แล้วแปลกใจว่าทำไม `[]` (ที่ length เป็น
0) กลับเข้า `if` เพราะจริงๆ แล้ว `0` ใน Ruby เป็น truthy — วิธีที่ถูกต้องคือเช็คตรงๆ ด้วย
`.empty?`:

```ruby
list = []

# ผิด — 0 เป็น truthy เสมอ ไม่ว่า array จะว่างหรือไม่
if list.length
  puts "โค้ดนี้รันเสมอไม่ว่า list จะว่างหรือไม่" # => รันเสมอ
end

# ถูก
if list.empty?
  puts "list ว่าง"
else
  puts "list มีข้อมูล"
end
# => list ว่าง
```

### `&&`, `||`, `!` และการ short-circuit

```ruby
age = 25
has_id_card = true

if age >= 18 && has_id_card
  puts "เข้าได้"
end

if age < 18 || !has_id_card
  puts "เข้าไม่ได้"
else
  puts "เข้าได้"
end
```

- `&&` (and) คืนค่าฝั่งขวาถ้าฝั่งซ้ายเป็น truthy, ไม่งั้นคืนฝั่งซ้ายทันที (short-circuit)
- `||` (or) คืนค่าฝั่งซ้ายถ้าเป็น truthy, ไม่งั้นคืนฝั่งขวา
- `!` กลับค่าความจริง (negation)

> **ข้อควรระวัง:** Ruby มีคำสั่ง `and`/`or`/`not` แบบคำเต็มด้วย แต่ **precedence (ลำดับความ
> สำคัญ) ต่ำกว่า `&&`/`||`/`!` มาก** จนทำให้เกิดบั๊กที่ดักได้ยาก แนวปฏิบัติมาตรฐานคือ
> **ใช้ `&&`, `||`, `!` เสมอสำหรับ logic ในเงื่อนไข** และสงวน `and`/`or` ไว้แค่การคุม
> control flow แบบพิเศษ (ซึ่งแทบไม่ใช้ในโค้ดจริง)

```ruby
# ตัวอย่างกับดักของ `or`
def save_user
  false or true  # นิพจน์นี้ = (false or true) = true ปกติ แต่...
end

# แต่ปัญหาจริงเกิดตอนผสมกับ = (assignment) เพราะ `=` มี precedence สูงกว่า `and`/`or`
result = false or true
p result
# => false !! (ตีความเป็น (result = false) or true ไม่ใช่ result = (false or true))

result2 = false || true
p result2
# => true (ตามที่คาดหวัง เพราะ || มี precedence สูงกว่า =)
```

---

## Step 52: `unless` และทำไมควรเลี่ยง `unless...else`, modifier `if`/`unless`

### `unless` คือ `if` แบบกลับด้าน

```ruby
logged_in = false

unless logged_in
  puts "กรุณาเข้าสู่ระบบ"
end
# => กรุณาเข้าสู่ระบบ

# เทียบเท่ากับ
if !logged_in
  puts "กรุณาเข้าสู่ระบบ"
end
```

`unless condition` คือ `if !condition` นั่นเอง — ใช้เมื่อการเขียนแบบปฏิเสธ (negative) อ่านแล้ว
เป็นธรรมชาติกว่า เช่น "ถ้ายังไม่ได้ login" ฟังลื่นกว่า "ถ้าไม่ได้ login เป็น true"

### ทำไมควรเลี่ยง `unless...else`

`unless` รองรับ `else` ได้ทางไวยากรณ์ แต่ **แนวปฏิบัติของชุมชน Ruby (RuboCop มี cop ชื่อ
`Style/UnlessElse` เพื่อเตือนเรื่องนี้โดยเฉพาะ) แนะนำให้เลี่ยง** เพราะทำให้อ่านยาก — สมองมนุษย์
ต้องประมวลผลปฏิเสธซ้อนปฏิเสธ (double negative) ซึ่งสับสนง่าย

```ruby
stock = 0

# เขียนได้ แต่ไม่ควรเขียนแบบนี้
unless stock > 0
  puts "สินค้าหมด"
else
  puts "มีสินค้า"
end
```

อ่านแล้วต้องคิดว่า "ถ้า *ไม่ใช่* ว่ามีสต็อกมากกว่า 0 ให้บอกว่าหมด ไม่งั้น (คือถ้ามีสต็อก)
บอกว่ามี" — ต้องกลับตรรกะในหัวสองรอบ ให้เขียนด้วย `if` แทนจะอ่านตรงไปตรงมากว่ามาก:

```ruby
if stock > 0
  puts "มีสินค้า"
else
  puts "สินค้าหมด"
end
```

**กฎที่ยึดได้เสมอ:** `unless` ใช้ได้ดีเมื่อไม่มี `else` เท่านั้น ถ้ามีทั้งสองแบบ (positive และ
negative case) ให้สลับกลับไปใช้ `if` เสมอ

### Modifier `if` / `unless` (statement modifier)

เมื่อ body มีแค่บรรทัดเดียว Ruby อนุญาตให้เขียนเงื่อนไขต่อท้ายในบรรทัดเดียวได้ เรียกว่า
**statement modifier** — เป็นสำนวนที่ใช้บ่อยมากในโค้ด Ruby/Rails จริง เพราะกระชับและอ่านลื่น
เหมือนภาษาอังกฤษ

```ruby
age = 15

puts "ยังไม่บรรลุนิติภาวะ" if age < 20

puts "รหัสผ่านว่างเปล่า" unless age.is_a?(Integer)

# เทียบเท่า if แบบเต็ม
if age < 20
  puts "ยังไม่บรรลุนิติภาวะ"
end
```

ตัวอย่างที่พบบ่อยในโค้ด production จริง — ใช้เป็น **guard clause** เพื่อออกจาก method ตั้งแต่
ต้นถ้าเงื่อนไขไม่ผ่าน (ลดการซ้อน `if` หลายชั้น):

```ruby
def withdraw(balance, amount)
  return "จำนวนเงินต้องมากกว่า 0" if amount <= 0
  return "ยอดเงินไม่พอ" if amount > balance

  balance - amount
end

puts withdraw(1000, -50)   # => จำนวนเงินต้องมากกว่า 0
puts withdraw(1000, 5000)  # => ยอดเงินไม่พอ
puts withdraw(1000, 300)   # => 700
```

> **แนวปฏิบัติ:** ใช้ modifier `if`/`unless` เฉพาะเมื่อ body สั้น (บรรทัดเดียว) และไม่มี
> `elsif`/`else` เท่านั้น ถ้าเงื่อนไขซับซ้อนหรือ body ยาว ให้ใช้ `if` แบบเต็มรูปแบบเพื่อความ
> อ่านง่าย — นี่เป็นกฎที่ RuboCop บังคับผ่าน cop `Style/IfUnlessModifier` ด้วย (ความยาวรวมไม่
> ควรเกิน 1 บรรทัด/ประมาณ 80 ตัวอักษร)

---

## Step 53: Ternary operator (`condition ? a : b`)

Ternary operator คือ `if...else` แบบบรรทัดเดียว เขียนด้วยรูปแบบ `condition ? value_if_true :
value_if_false`

```ruby
age = 20

status = age >= 18 ? "ผู้ใหญ่" : "ผู้เยาว์"
puts status
# => ผู้ใหญ่
```

**เทียบเท่ากับ:**

```ruby
status =
  if age >= 18
    "ผู้ใหญ่"
  else
    "ผู้เยาว์"
  end
```

### เมื่อไหร่ควรใช้ ternary vs `if`

Ternary เหมาะกับกรณีที่:

1. เป็นการเลือกค่า (assign ตัวแปร หรือส่งเป็น argument) ไม่ใช่การรันคำสั่งหลายคำสั่ง
2. เงื่อนไขและทั้งสองค่าสั้น อ่านจบในสายตาเดียว

```ruby
# ใช้ดี — สั้น กระชับ ชัดเจน
temperature = 35
message = temperature > 30 ? "ร้อนมาก" : "อากาศปกติ"

# ใช้เป็น argument ของ method ได้ทันที
puts temperature > 30 ? "ร้อนมาก" : "อากาศปกติ"
```

```ruby
# ไม่ควรใช้ — ซ้อน ternary หลายชั้นอ่านยากมาก (nested ternary = อ่านไม่รู้เรื่อง)
grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F"
# หลีกเลี่ยงแบบนี้! ให้ใช้ if/elsif หรือ case/when แทนเมื่อมีมากกว่า 2 ทางเลือก
```

> **กฎ:** ternary ใช้ได้ดีที่สุดกับการตัดสินใจแบบ **2 ทางเลือก (binary choice)** เท่านั้น ถ้ามี
> มากกว่า 2 ทางเลือก ให้ใช้ `if/elsif/else` หรือ `case/when` — RuboCop มี cop
> `Style/NestedTernaryOperator` ที่ห้าม ternary ซ้อน ternary โดยเฉพาะ

### Ternary กับ method call

```ruby
def even_or_odd(n)
  n.even? ? "คู่" : "คี่"
end

puts even_or_odd(4)  # => คู่
puts even_or_odd(7)  # => คี่
```

---

## Step 54: `case`/`when` พื้นฐาน — เทียบค่าด้วย `===`

เมื่อมีเงื่อนไขหลายทางเลือกที่เทียบกับค่าตัวแปรเดียว `case/when` มักอ่านง่ายกว่า
`if/elsif/else` ยาวๆ มาก

### โครงสร้างพื้นฐาน

```ruby
day = "Monday"

case day
when "Monday"
  puts "วันจันทร์"
when "Tuesday"
  puts "วันอังคาร"
when "Wednesday"
  puts "วันพุธ"
else
  puts "วันอื่นๆ"
end
# => วันจันทร์
```

### หลายเงื่อนไขใน `when` เดียวกัน (คั่นด้วย comma)

```ruby
day = "Saturday"

case day
when "Saturday", "Sunday"
  puts "วันหยุดสุดสัปดาห์"
else
  puts "วันทำงาน"
end
# => วันหยุดสุดสัปดาห์
```

### `case` เป็น expression คืนค่าได้เหมือน `if`

```ruby
day = "Sunday"

day_type =
  case day
  when "Saturday", "Sunday"
    "วันหยุด"
  else
    "วันทำงาน"
  end

puts day_type
# => วันหยุด
```

### เบื้องหลังการทำงาน: `case/when` ใช้ `===` ไม่ใช่ `==`

นี่คือกลไกสำคัญที่ทำให้ `case/when` ทรงพลังกว่า `if/elsif` มาก — Ruby เปรียบเทียบแต่ละ
`when` ด้วย method **`===` (case equality operator)** ที่เรียกจาก **ฝั่งซ้าย** (ตัวที่อยู่หลัง
`when`) ไม่ใช่จากตัวแปรที่อยู่หลัง `case`

```ruby
case day
when "Saturday", "Sunday"
  # ...
end

# เบื้องหลังเทียบเท่ากับ
if "Saturday" === day || "Sunday" === day
  # ...
end
```

สำหรับ String, `String#===` ทำงานเหมือน `==` ธรรมดา แต่คลาสอื่น เช่น `Range`, `Regexp`,
`Class` **override `===` ให้มีความหมายต่างออกไป** ซึ่งนี่คือกุญแจของ Step ถัดไป

```ruby
p "hello" === "hello"   # => true  (String#=== เหมือน ==)
p (1..10) === 5          # => true  (Range#=== คือ "อยู่ในช่วงหรือไม่")
p Integer === 5          # => true  (Module#=== คือ "เป็น instance ของคลาสนี้หรือไม่")
p /^h/ === "hello"       # => true  (Regexp#=== คือ "match หรือไม่")
```

---

## Step 55: `case`/`when` ขั้นสูง — Range, Regexp, Class และ `case` แบบไม่มี argument

เพราะ `case/when` ใช้ `===` เราจึงใช้ **Range, Regexp, และ Class** เป็นเงื่อนไขใน `when` ได้
โดยตรง — เป็นสำนวนที่ใช้บ่อยมากในโค้ด Ruby ระดับมืออาชีพ

### 1) `case` กับ Range — เขียนแทน `if score >= 90 && score <= 100` ยาวๆ

```ruby
score = 85

grade =
  case score
  when 90..100
    "A"
  when 80...90
    "B"
  when 70...80
    "C"
  when 60...80
    "D"
  else
    "F"
  end

puts grade
# => B
```

**อธิบาย:** `90..100` คือ Range แบบรวมค่าปลาย (inclusive, รวม 100), `80...90` คือ Range แบบ
ไม่รวมค่าปลาย (exclusive, ไม่รวม 90) — จำไว้ว่า `..` (2 จุด) รวมปลาย, `...` (3 จุด) ไม่รวมปลาย
เทียบกับการเขียนด้วย `if/elsif` แบบเดิม (`if score >= 90 && score <= 100 ... elsif score >=
80 && score < 90 ...`) จะเห็นว่า `case` กับ Range **อ่านง่ายและพิมพ์น้อยกว่ามาก**

### 2) `case` กับ Regexp — จับรูปแบบของ String

```ruby
def classify_input(text)
  case text
  when /\A\d+\z/
    "เป็นตัวเลขล้วน"
  when /\A[a-zA-Z]+\z/
    "เป็นตัวอักษรภาษาอังกฤษล้วน"
  when /\A\S+@\S+\.\S+\z/
    "ดูเหมือนอีเมล"
  else
    "รูปแบบอื่นๆ"
  end
end

puts classify_input("12345")              # => เป็นตัวเลขล้วน
puts classify_input("hello")               # => เป็นตัวอักษรภาษาอังกฤษล้วน
puts classify_input("user@example.com")    # => ดูเหมือนอีเมล
puts classify_input("รหัส-99")             # => รูปแบบอื่นๆ
```

### 3) `case` กับ Class — แยกโค้ดตามชนิดของข้อมูล (คล้าย type-checking)

```ruby
def describe(value)
  case value
  when Integer
    "เป็นจำนวนเต็ม: #{value}"
  when Float
    "เป็นทศนิยม: #{value}"
  when String
    "เป็นข้อความ: #{value}"
  when Array
    "เป็น Array ที่มี #{value.size} สมาชิก"
  when NilClass
    "เป็นค่า nil"
  else
    "ไม่รู้จักชนิดข้อมูลนี้"
  end
end

puts describe(42)          # => เป็นจำนวนเต็ม: 42
puts describe(3.14)        # => เป็นทศนิยม: 3.14
puts describe("Ruby")      # => เป็นข้อความ: Ruby
puts describe([1, 2, 3])   # => เป็น Array ที่มี 3 สมาชิก
puts describe(nil)         # => เป็นค่า nil
```

**ทำไมใช้ได้:** `Integer === 42` คือการถาม "42 เป็น instance ของ `Integer` (หรือคลาสลูก)
หรือไม่" ซึ่งตรงกับ `42.is_a?(Integer)` — นี่คือกลไกเดียวกับที่ `rescue SomeError` ใน
exception handling ใช้เทียบชนิด exception (จะเรียนละเอียดใน Part 011)

### 4) `case` แบบไม่มี argument — ใช้แทน `if/elsif` ที่มีหลายเงื่อนไขไม่เกี่ยวข้องกัน

เมื่อแต่ละ `when` เป็นเงื่อนไข boolean คนละเรื่องกัน (ไม่ได้เทียบกับตัวแปรตัวเดียว) เราเขียน
`case` โดยไม่ใส่ argument ได้ แต่ละ `when` จะถูกประเมินเป็น boolean expression ตรงๆ
(เทียบเท่ากับเขียน `when true`)

```ruby
temperature = 38
humidity = 85

weather_alert =
  case
  when temperature > 40
    "อากาศร้อนจัดอันตราย"
  when temperature > 35 && humidity > 80
    "ร้อนและชื้นมาก ระวังฮีทสโตรก"
  when temperature < 15
    "อากาศหนาว"
  else
    "อากาศปกติ"
  end

puts weather_alert
# => ร้อนและชื้นมาก ระวังฮีทสโตรก
```

**เทียบกับ `if/elsif` เดิม:**

```ruby
weather_alert =
  if temperature > 40
    "อากาศร้อนจัดอันตราย"
  elsif temperature > 35 && humidity > 80
    "ร้อนและชื้นมาก ระวังฮีทสโตรก"
  elsif temperature < 15
    "อากาศหนาว"
  else
    "อากาศปกติ"
  end
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ — เป็นเรื่องของสไตล์ล้วนๆ หลายทีมเลือกใช้ `case` แบบไม่มี
argument เมื่อมีเงื่อนไข 3 อันขึ้นไป เพราะคำว่า `when` แต่ละบรรทัดดูเรียงเป็นระเบียบกว่า
`elsif` ที่ซ้ำคำว่า `if` ทุกบรรทัด — เลือกใช้แบบไหนก็ได้ตามธรรมเนียมของทีม ขอแค่สม่ำเสมอ

---

## Step 56: `loop do...end`, `break`, และการคืนค่าจาก loop ด้วย `break value`

### `loop` คือ infinite loop ที่ต้องหยุดด้วย `break`

```ruby
count = 0

loop do
  count += 1
  puts "รอบที่ #{count}"
  break if count >= 3
end

puts "จบการวนลูป"
# => รอบที่ 1
# => รอบที่ 2
# => รอบที่ 3
# => จบการวนลูป
```

**อธิบาย:**

- `loop do ... end` คือ method `Kernel#loop` ที่รับ block แล้ววนซ้ำไปเรื่อยๆ **ไม่มีเงื่อนไข
  หยุดในตัวเอง** — ต้องมี `break` อยู่ข้างในเสมอ ไม่งั้นจะวนไม่รู้จบ (infinite loop ค้าง
  โปรแกรม)
- `break` จะหยุดการวนลูปทันที และโปรแกรมไปทำงานบรรทัดถัดจาก `end` ของ loop

### `break` พร้อมค่า — ให้ `loop` คืนค่ากลับมาได้

จุดที่หลายคนไม่รู้: `loop` (และ loop ทุกชนิดใน Ruby) เป็น **expression** ที่คืนค่าได้ ถ้า
`break` มีค่าตามหลัง ค่านั้นจะกลายเป็นค่าที่ `loop` คืนกลับมา

```ruby
result =
  loop do
    number = rand(1..100)
    break number if number > 90
  end

puts "ได้เลขที่มากกว่า 90 คือ: #{result}"
```

ตัวอย่างที่เป็นรูปธรรมกว่า — หาตัวเลขสุ่มตัวแรกที่หารด้วย 7 ลงตัว:

```ruby
found =
  loop do
    n = rand(1..1000)
    break n if n % 7 == 0
  end

puts "เจอเลขที่หาร 7 ลงตัว: #{found}"
```

ถ้า `loop` จบโดยไม่มี `break` ที่ให้ค่า (เช่นจบด้วย exception หรือไม่มี `break` เลยซึ่งไม่ควร
เกิดขึ้น) ค่าที่ได้คือ `nil`

### เปรียบเทียบกับ `while true` — ทำไมนิยม `loop do` มากกว่า

```ruby
# แบบ while true — ใช้ได้ แต่ RuboCop จะเตือนให้เปลี่ยนเป็น loop do
count = 0
while true
  count += 1
  break if count >= 3
end

# แบบ loop do — นิยมกว่าในโค้ด Ruby สมัยใหม่ อ่านเจตนาชัดกว่า ("วนไปเรื่อยๆ จนกว่าจะ break")
count = 0
loop do
  count += 1
  break if count >= 3
end
```

`loop` สื่อเจตนาได้ตรงกว่า: "ฉันตั้งใจวนไม่รู้จบ แล้วจะ break เอง" ในขณะที่ `while true` ดู
เหมือนเขียนเงื่อนไขผิดโดยไม่ตั้งใจ

---

## Step 57: `while` / `until` — block form และ modifier form

### `while` — วนซ้ำตราบใดที่เงื่อนไขเป็น true

```ruby
i = 1

while i <= 5
  puts "i = #{i}"
  i += 1
end
# => i = 1
# => i = 2
# => i = 3
# => i = 4
# => i = 5
```

**อธิบาย:** `while condition ... end` วนซ้ำ body ตราบใดที่ `condition` เป็น truthy — ตรวจสอบ
เงื่อนไข **ก่อน** รันแต่ละรอบเสมอ (pre-test loop) ถ้าเงื่อนไขเป็น false ตั้งแต่แรก body จะไม่
ถูกรันเลยสักครั้ง

```ruby
x = 10
while x < 5
  puts "จะไม่ถูกพิมพ์เลย"
end
puts "จบแล้ว"
# => จบแล้ว
```

### `until` — คือ `while` แบบกลับด้าน (วนจนกว่าเงื่อนไขจะเป็น true)

```ruby
i = 1

until i > 5
  puts "i = #{i}"
  i += 1
end
# ผลลัพธ์เหมือนตัวอย่าง while ด้านบนทุกประการ
```

`until condition` เทียบเท่ากับ `while !condition` — เลือกใช้ตัวไหนขึ้นอยู่กับว่าประโยคไหน
อ่านเป็นธรรมชาติกว่า เช่น "ทำไปเรื่อยๆ **จนกว่า** คิวจะว่าง" ฟังลื่นกว่า "ทำไปเรื่อยๆ
**ตราบใดที่** คิวไม่ว่าง"

```ruby
queue = [1, 2, 3]

until queue.empty?
  item = queue.shift
  puts "ประมวลผล: #{item}"
end
# => ประมวลผล: 1
# => ประมวลผล: 2
# => ประมวลผล: 3
```

### Modifier form ของ `while`/`until`

เหมือน `if`/`unless` ทั้ง `while` และ `until` เขียนเป็น statement modifier ได้เมื่อ body มี
แค่บรรทัดเดียว:

```ruby
i = 1
i += 1 while i < 5
puts i
# => 5

count = 10
count -= 1 until count <= 0
puts count
# => 0
```

### ข้อควรระวัง: `begin...end while` คือ **post-test loop** (do-while)

Ruby มีรูปแบบพิเศษที่ทำให้ body รันอย่างน้อย 1 ครั้งก่อนเช็คเงื่อนไข (เหมือน `do...while`
ในภาษา C/Java) โดยห่อ body ด้วย `begin...end` แล้วต่อท้ายด้วย `while`/`until`:

```ruby
i = 10

# while ปกติ (pre-test) — เช็คก่อนรัน จึงไม่รันเลยสักครั้ง
while i < 5
  puts "จะไม่ถูกพิมพ์"
  i += 1
end

# begin...end while (post-test) — รันก่อน 1 ครั้งเสมอ แล้วค่อยเช็คเงื่อนไข
i = 10
begin
  puts "รันอย่างน้อย 1 ครั้งเสมอ: i = #{i}"
  i += 1
end while i < 5
# => รันอย่างน้อย 1 ครั้งเสมอ: i = 10
```

รูปแบบนี้ใช้บ่อยเวลาต้องรับ input จากผู้ใช้อย่างน้อยหนึ่งครั้งก่อนตรวจสอบ เช่น เมนูโปรแกรมที่
ต้องแสดงอย่างน้อยหนึ่งรอบเสมอ:

```ruby
answer = nil

begin
  print "พิมพ์ 'exit' เพื่อออกจากโปรแกรม: "
  answer = gets.chomp
  puts "คุณพิมพ์: #{answer}"
end while answer != "exit"
```

> **หมายเหตุ:** รูปแบบ `begin...end while` ไม่ค่อยพบในโค้ด production สมัยใหม่มากนัก
> (ส่วนใหญ่ใช้ `loop do ... break if condition ... end` แทน เพราะสื่อเจตนาชัดกว่า) แต่ควรรู้
> จักไว้เพราะเจอในโค้ดเก่าได้บ่อย

---

## Step 58: `for` loop และทำไม Rubyist ส่วนใหญ่เลี่ยงมันแล้วใช้ `each` แทน

### ไวยากรณ์ของ `for`

```ruby
for i in 1..5
  puts "i = #{i}"
end
# => i = 1
# => i = 2
# => i = 3
# => i = 4
# => i = 5

fruits = ["apple", "banana", "cherry"]
for fruit in fruits
  puts fruit
end
# => apple
# => banana
# => cherry
```

หน้าตาดูคล้าย `each` มาก และให้ผลลัพธ์การพิมพ์เหมือนกัน แต่ **มีความต่างเชิงกลไกที่สำคัญ**

### ความต่างที่ 1: `for` ไม่สร้าง scope ใหม่ — ตัวแปร leak ออกมาข้างนอก

```ruby
for i in 1..3
  x = i * 10
end

puts i  # => 3   (!) ตัวแปร i ยัง "รั่ว" ออกมาใช้ได้นอก for
puts x  # => 30  (!) ตัวแปร x ก็รั่วออกมาด้วยเช่นกัน
```

เทียบกับ `each` ที่ variable ในการ block เป็นของ **local scope ของ block เท่านั้น**:

```ruby
(1..3).each do |i|
  y = i * 10
end

puts i rescue puts "เกิด error: ไม่มีตัวแปร i นอก block"
# => เกิด error: ไม่มีตัวแปร i นอก block
puts y rescue puts "เกิด error: ไม่มีตัวแปร y นอก block"
# => เกิด error: ไม่มีตัวแปร y นอก block
```

การที่ตัวแปรรั่วออกมานอก scope ที่ตั้งใจ เป็นแหล่งบั๊กคลาสสิก — เผลอใช้ชื่อตัวแปรซ้ำ
(shadowing) แล้วค่าจากลูปเก่าไปปนกับโค้ดส่วนอื่นโดยไม่ได้ตั้งใจ นี่คือเหตุผลหลักที่ Ruby
community หลีกเลี่ยง `for`

### ความต่างที่ 2: `for` ทำงานกับ Enumerable ได้จำกัดกว่า `each`

`each` เป็น method ที่ทุก Enumerable (Array, Hash, Range, Set ฯลฯ) มี และยังต่อ method chain
อื่นๆ ได้ทันที เช่น `each_with_index`, `each.with_index`, ในขณะที่ `for` ใช้ได้เฉพาะกับสิ่งที่
ตอบสนอง `each` แต่เขียน chain แบบเดียวกันไม่ได้เป็นธรรมชาติ

```ruby
# each ต่อ method อื่นได้ลื่นไหล (จำได้จาก Part 004)
["a", "b", "c"].each_with_index do |item, index|
  puts "#{index}: #{item}"
end

# for ทำแบบนี้ไม่ได้โดยตรง ต้องใช้ .each_with_index ผ่าน for ...in ซึ่งพิสดารและไม่มีใครเขียน
```

### สรุป: แนวปฏิบัติมาตรฐานของวงการ Ruby

> **ใช้ `each` (หรือ `map`, `select` ฯลฯ ตาม Part 004) แทน `for` เสมอ** RuboCop มี cop ชื่อ
> `Style/For` ที่ default บังคับให้แปลง `for` เป็น `each` โดยอัตโนมัติ (`rubocop -A`) — จะไม่
> เจอ `for` loop ในโค้ด Rails/Ruby มืออาชีพเกือบทุกที่ ยกเว้นในโค้ดเก่ามากๆ หรือโค้ดที่ port
> มาจากภาษาอื่นแบบตรงตัว

```ruby
# ไม่แนะนำ
for n in [1, 2, 3]
  puts n * n
end

# แนะนำ
[1, 2, 3].each do |n|
  puts n * n
end
```

เหตุผลเชิงลึกอีกข้อ: `for` เป็น **keyword ของภาษา** (ส่วนหนึ่งของ syntax) ในขณะที่ `each` เป็น
**method ธรรมดา** ที่รับ block — ปรัชญาของ Ruby คือทำให้สิ่งต่างๆ เป็น object/method ให้มาก
ที่สุดเท่าที่จะทำได้ (แม้แต่ตัวเลขก็เป็น object) การใช้ `each` จึงสอดคล้องกับปรัชญานี้มากกว่า
และยังเปิดทางให้ override หรือขยายพฤติกรรมผ่าน method ปกติได้ ซึ่ง `for` ทำไม่ได้

---

## Step 59: ความหมายของ `break` / `next` / `redo` แบบเจาะลึก

สามคำสั่งนี้ควบคุมการไหลของ loop ในแบบที่ต่างกันโดยสิ้นเชิง สับสนกันบ่อยมากสำหรับผู้เริ่มต้น
เรามาดูทีละตัวพร้อมตัวอย่างเทียบกันตรงๆ

### `break` — หยุด loop ทั้งหมดทันที

`break` ออกจาก loop ทั้งก้อนทันที ไม่สนใจว่าเหลืออีกกี่รอบ

```ruby
(1..5).each do |i|
  break if i == 3
  puts i
end
# => 1
# => 2
# (หยุดตรงนี้ ไม่พิมพ์ 3, 4, 5 เลย)
```

### `next` — ข้ามไปทำรอบถัดไปทันที (เหมือน `continue` ในภาษาอื่น)

`next` หยุดแค่รอบปัจจุบัน แล้วกระโดดไปเริ่มรอบถัดไปเลย โดยไม่ทำโค้ดที่เหลือใน block ของรอบ
นี้

```ruby
(1..5).each do |i|
  next if i == 3
  puts i
end
# => 1
# => 2
# => 4    (ข้าม 3 ไปเลย แต่ยังทำรอบ 4, 5 ต่อ)
# => 5
```

**เปรียบเทียบ `break` vs `next` ในสถานการณ์เดียวกัน** เพื่อให้เห็นความต่างชัดๆ:

```ruby
puts "--- ใช้ break ---"
[10, 20, 30, 40, 50].each do |n|
  break if n == 30
  puts n
end
# => 10
# => 20
# (จบทันที ไม่แตะ 30, 40, 50)

puts "--- ใช้ next ---"
[10, 20, 30, 40, 50].each do |n|
  next if n == 30
  puts n
end
# => 10
# => 20
# => 40   (ข้ามเฉพาะ 30 ตัวเดียว)
# => 50
```

### `next` พร้อมค่า — กำหนดค่าที่ block คืนกลับให้ method อย่าง `map`

`next value` มีประโยชน์มากเมื่อใช้กับ method ที่สนใจ **ค่าที่ block คืนกลับ** เช่น `map`,
`select`, `reduce` — `next` จะกำหนดว่า element ปัจจุบันของ `map` ควรถูกแปลงเป็นค่าอะไร

```ruby
numbers = [1, 2, 3, 4, 5]

doubled_unless_three =
  numbers.map do |n|
    next 0 if n == 3   # ถ้าเป็น 3 ให้ผลลัพธ์ของ map ตรงนี้เป็น 0 แทน แล้วไปรอบถัดไป
    n * 2
  end

p doubled_unless_three
# => [2, 4, 0, 8, 10]
```

> **ข้อควรรู้:** `break` ใช้กำหนดค่าคืนของ **loop ทั้งก้อน** (ตามที่เห็นใน Step 56) ในขณะที่
> `next` ใช้กำหนดค่าคืนของ **การวน 1 รอบ** เท่านั้น — คนละระดับกันโดยสิ้นเชิง

### `redo` — ทำรอบปัจจุบันซ้ำอีกครั้ง โดยไม่ขยับไป element ถัดไป

`redo` แปลกกว่าสองตัวข้างบน: มันสั่งให้ block รันซ้ำ **รอบเดิม** (element เดิม, ตัวแปร loop
เดิม) โดยไม่เลื่อนไปข้างหน้า — ใช้น้อยกว่ามากในโค้ดจริง แต่มีประโยชน์เวลาต้องให้ผู้ใช้กรอก
input ใหม่จนกว่าจะถูกต้อง

```ruby
attempts = 0

[1, 2, 3].each do |n|
  attempts += 1
  puts "กำลังประมวลผล n = #{n} (ครั้งที่ #{attempts})"

  if n == 2 && attempts < 5
    redo  # วนซ้ำที่ n == 2 อีกครั้ง ไม่ไปที่ n == 3
  end
end
```

ผลลัพธ์:

```
กำลังประมวลผล n = 1 (ครั้งที่ 1)
กำลังประมวลผล n = 2 (ครั้งที่ 2)
กำลังประมวลผล n = 2 (ครั้งที่ 3)
กำลังประมวลผล n = 2 (ครั้งที่ 4)
กำลังประมวลผล n = 2 (ครั้งที่ 5)
กำลังประมวลผล n = 3 (ครั้งที่ 6)
```

**สังเกต:** `n` ค้างอยู่ที่ `2` ซ้ำ 4 ครั้ง (จนกว่า `attempts` จะถึง 5) ก่อนจะขยับไป `3` — นี่คือ
ความต่างสำคัญจาก `next` ที่จะเลื่อนไป element ถัดไปเสมอ ในขณะที่ `redo` วนที่ element เดิมซ้ำ

### ตัวอย่างการใช้ `redo` ที่สมจริงกว่า — วนถามซ้ำจนกว่า input จะถูกต้อง

```ruby
["ก", "ข", "ค"].each do |letter|
  print "พิมพ์ตัวอักษร '#{letter}' เพื่อยืนยัน: "
  input = gets.chomp
  redo unless input == letter
  puts "ยืนยัน #{letter} สำเร็จ"
end
```

โค้ดนี้จะถามผู้ใช้ซ้ำที่ตัวอักษรเดิมไปเรื่อยๆ จนกว่าจะพิมพ์ตรงกัน แล้วค่อยขยับไปตัวถัดไป

### ตารางสรุปเปรียบเทียบ

| Keyword | ผลลัพธ์ | ใช้บ่อยแค่ไหน |
|---------|---------|----------------|
| `break` | ออกจาก loop ทั้งก้อนทันที (ให้ค่าคืนของ loop ได้ด้วย) | บ่อยมาก |
| `next`  | ข้ามไปยังรอบถัดไป (ให้ค่าคืนของรอบนั้นได้ เช่นใน `map`) | บ่อยมาก |
| `redo`  | ทำรอบปัจจุบันซ้ำ โดยไม่เลื่อนไป element ถัดไป | ใช้น้อย เฉพาะกรณีพิเศษ |

---

## Step 60: แบบฝึกหัด — เกมทายตัวเลข และ FizzBuzz ด้วย `case/when`

### โจทย์ที่ 1: เกมทายตัวเลข (Number Guessing Game)

เขียนโปรแกรม `guessing_game.rb` ที่ทำงานดังนี้:

1. สุ่มตัวเลขคำตอบระหว่าง 1–100 (ใช้ `rand(1..100)`)
2. ให้ผู้ใช้กรอกตัวเลขทายซ้ำได้ไม่จำกัดจำนวนครั้ง โดยใช้ `loop do...end`
3. ทุกครั้งที่ทาย บอกผู้ใช้ว่าทายสูงไป, ต่ำไป, หรือถูกต้อง (ใช้ `case` เทียบด้วย `<=>` หรือ
   `if/elsif`)
4. ถ้าทายถูก ให้แสดงจำนวนครั้งที่ทายทั้งหมด แล้วจบเกมด้วย `break`
5. ป้องกันกรณีผู้ใช้กรอกสิ่งที่ไม่ใช่ตัวเลข ให้แจ้งเตือนแล้ววนถามใหม่ (ใช้ `next`)

### เฉลย

```ruby
# frozen_string_literal: true

# guessing_game.rb
answer = rand(1..100)
attempts = 0

puts "=" * 40
puts "เกมทายตัวเลข! ทายตัวเลขระหว่าง 1-100"
puts "=" * 40

loop do
  print "ทายตัวเลขของคุณ: "
  input = gets.chomp

  # ป้องกัน input ที่ไม่ใช่ตัวเลข ด้วย regex ตรวจสอบก่อนแปลง
  unless input.match?(/\A\d+\z/)
    puts "กรุณากรอกเป็นตัวเลขเท่านั้น"
    next
  end

  guess = input.to_i
  attempts += 1

  case guess <=> answer
  when -1
    puts "ทายต่ำไป ลองใหม่อีกครั้ง!"
  when 1
    puts "ทายสูงไป ลองใหม่อีกครั้ง!"
  when 0
    puts "-" * 40
    puts "ถูกต้อง! คำตอบคือ #{answer}"
    puts "คุณทายทั้งหมด #{attempts} ครั้ง"
    puts "-" * 40
    break
  end
end
```

ทดสอบรัน (ตัวอย่างสมมติว่าคำตอบคือ 42):

```bash
ruby guessing_game.rb
# ========================================
# เกมทายตัวเลข! ทายตัวเลขระหว่าง 1-100
# ========================================
# ทายตัวเลขของคุณ: 50
# ทายสูงไป ลองใหม่อีกครั้ง!
# ทายตัวเลขของคุณ: abc
# กรุณากรอกเป็นตัวเลขเท่านั้น
# ทายตัวเลขของคุณ: 25
# ทายต่ำไป ลองใหม่อีกครั้ง!
# ทายตัวเลขของคุณ: 42
# ----------------------------------------
# ถูกต้อง! คำตอบคือ 42
# คุณทายทั้งหมด 3 ครั้ง
# ----------------------------------------
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `rand(1..100)` — สุ่มตัวเลขจำนวนเต็มใน Range ที่กำหนด (รวมค่า 100 ด้วย เพราะใช้ `..`)
- `guess <=> answer` — **spaceship operator** เปรียบเทียบสองค่า คืน `-1` ถ้าฝั่งซ้ายน้อยกว่า,
  `0` ถ้าเท่ากัน, `1` ถ้าฝั่งซ้ายมากกว่า — เหมาะมากกับ `case/when` เพราะได้ผลลัพธ์เป็นค่าคงที่
  สามค่าพอดี ทำให้เขียน `when -1`, `when 1`, `when 0` ได้ตรงไปตรงมา
- `input.match?(/\A\d+\z/)` — ตรวจสอบว่า string เป็นตัวเลขล้วนก่อนแปลงด้วย `to_i` (ป้องกัน
  กรณีผู้ใช้กรอกตัวอักษรแล้ว `to_i` เงียบๆ คืนค่า `0` โดยไม่แจ้งเตือน)
- `next` ใน `loop` — ข้ามการประมวลผลที่เหลือของรอบนั้น กลับไปถาม input ใหม่ทันที โดยไม่เพิ่ม
  `attempts`

### โจทย์ที่ 2: FizzBuzz แบบขยาย ด้วย `case/when`

เขียนโปรแกรม `fizzbuzz.rb` ที่วนตัวเลข 1 ถึง 30 และพิมพ์ตามกฎ (ใช้ `case` แบบไม่มี argument
เพื่อฝึกสิ่งที่เรียนใน Step 55):

- ถ้าหารด้วย 15 ลงตัว → พิมพ์ `"FizzBuzz"`
- ถ้าหารด้วย 3 ลงตัว (แต่ไม่ใช่ 15) → พิมพ์ `"Fizz"`
- ถ้าหารด้วย 5 ลงตัว (แต่ไม่ใช่ 15) → พิมพ์ `"Buzz"`
- ถ้าเป็นจำนวนเฉพาะ (prime) → พิมพ์ `"Prime: <เลขนั้น>"`
- นอกนั้น → พิมพ์ตัวเลขตามปกติ

### เฉลย

```ruby
# frozen_string_literal: true

# fizzbuzz.rb
def prime?(n)
  return false if n < 2

  (2..Math.sqrt(n)).none? { |i| (n % i).zero? }
end

(1..30).each do |n|
  result =
    case
    when (n % 15).zero?
      "FizzBuzz"
    when (n % 3).zero?
      "Fizz"
    when (n % 5).zero?
      "Buzz"
    when prime?(n)
      "Prime: #{n}"
    else
      n.to_s
    end

  puts result
end
```

ผลลัพธ์บางส่วน:

```bash
ruby fizzbuzz.rb
# 1
# Prime: 2
# Fizz
# Prime: 5... (5 หาร 5 ลงตัวก่อน จึงเป็น Buzz ไม่ใช่ Prime — ดู "สังเกต" ด้านล่าง)
```

> **สังเกตสำคัญ (จุดฝึกอ่านลำดับความสำคัญของ `case`):** เนื่องจาก `case` ไล่ตรวจ `when` จาก
> บนลงล่างและหยุดที่ตัวแรกที่ match เลข `5` จะเข้า `when (n % 5).zero?` ก่อนถึงบรรทัด
> `prime?(n)` เสมอ (แม้ 5 จะเป็นทั้งจำนวนเฉพาะและหารด้วย 5 ลงตัว) ผลคือพิมพ์ `"Buzz"` ไม่ใช่
> `"Prime: 5"` — นี่คือเหตุผลว่าทำไม **ลำดับของ `when` ใน case มีผลต่อผลลัพธ์เสมอ** ต้องวาง
> เงื่อนไขที่เฉพาะเจาะจงกว่าไว้ก่อนเงื่อนไขที่กว้างกว่า ถ้าโจทย์ต้องการให้ 5 แสดงเป็น
> `"Prime: 5"` ต้องสลับให้ `when prime?(n)` มาก่อน `when (n % 5).zero?`

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `Math.sqrt(n)` — คำนวณรากที่สอง ใช้เพื่อลดจำนวนรอบที่ต้องเช็คว่าเป็นจำนวนเฉพาะหรือไม่
  (ไม่ต้องเช็คถึง `n` ตรงๆ เช็คถึง `√n` ก็พอ)
- `.none? { ... }` — Enumerable method ที่คืน `true` ถ้าไม่มี element ไหนทำให้ block เป็น
  true เลย (ทบทวนจาก Part 004)
- `(n % i).zero?` — เขียนแทน `n % i == 0` ให้อ่านเป็นธรรมชาติมากขึ้น (`.zero?` เป็น method ของ
  `Numeric`)

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. ปรับปรุง `guessing_game.rb` ให้จำกัดจำนวนครั้งในการทายไม่เกิน 7 ครั้ง โดยใช้ `while
   attempts < 7` แทน `loop do`; ถ้าทายครบ 7 ครั้งแล้วยังไม่ถูก ให้แสดงข้อความ "หมดโอกาสแล้ว!
   คำตอบคือ..." (ใบ้: ต้องคิดจุดที่จะ `break` ออกจาก `while` ทั้งกรณีทายถูกและกรณีครบจำนวน
   ครั้ง)
2. เขียนโปรแกรม `traffic_light.rb` ที่รับ input เป็นสี (`"red"`, `"yellow"`, `"green"`) แล้วใช้
   `case/when` แสดงคำสั่ง ("หยุด", "เตรียมตัว", "ไปได้") ถ้า input ไม่ตรงกับสีใดเลยให้แสดง
   "สีไม่ถูกต้อง" — จากนั้นดัดแปลงให้ใช้ `loop` วนรับ input ซ้ำได้เรื่อยๆ จนกว่าจะพิมพ์คำว่า
   `"exit"` (ใช้ `break if input == "exit"`)
3. เขียนโปรแกรมตรวจสอบรหัสผ่าน (`password_checker.rb`) ที่รับ input ทีละ 1 บรรทัด แล้วใช้
   `redo` เพื่อบังคับให้ผู้ใช้กรอกรหัสผ่านใหม่ทันทีถ้ารหัสผ่านมีความยาวน้อยกว่า 8 ตัวอักษร
   (ห้ามใช้ `loop` ซ้อน `case`/`if` แบบภายนอก ให้ใช้ `redo` ภายใน `each` ตามที่เรียนใน Step
   59 เท่านั้น) แล้วต่อยอดเพิ่มเงื่อนไขว่ารหัสผ่านต้องมีทั้งตัวเลขและตัวอักษรปนกัน (ใช้ regex
   `/\d/` และ `/[a-zA-Z]/` ร่วมกับ `&&`)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ใช้ `if/elsif/else` ได้อย่างถูกต้อง และเข้าใจว่า `if` เป็น expression ที่คืนค่าได้เหมือน
  method
- ทบทวนกฎ truthiness ของ Ruby: มีแค่ `nil` และ `false` เท่านั้นที่ falsy (`0`, `""`, `[]` เป็น
  truthy ทั้งหมด — ต่างจากภาษาอื่นอย่างมีนัยสำคัญ)
- ใช้ `unless` ได้อย่างเหมาะสม และรู้ว่าทำไมควรเลี่ยง `unless...else` เพื่อความอ่านง่าย
- ใช้ modifier `if`/`unless`/`while`/`until` เพื่อเขียนเงื่อนไขบรรทัดเดียวแบบกระชับ รวมถึงรู้
  จักสำนวน guard clause
- ใช้ ternary operator (`? :`) ได้ถูกที่ถูกเวลา และรู้ว่าไม่ควรซ้อน ternary หลายชั้น
- เข้าใจกลไกเบื้องหลัง `case/when` ที่ใช้ `===` และนำไปใช้กับ Range, Regexp, Class ได้ รวมถึง
  ใช้ `case` แบบไม่มี argument แทน `if/elsif` ยาวๆ ได้
- แยกความแตกต่างของ `loop`, `while`, `until`, `for` ได้ครบ และรู้ว่า `break value` ทำให้
  loop คืนค่าได้
- เข้าใจว่าทำไม Ruby community เลี่ยง `for` เพราะปัญหาเรื่อง scope ที่ตัวแปรรั่วออกมา และใช้
  `each` แทนเป็นมาตรฐาน
- แยกความแตกต่างระหว่าง `break` (หยุดทั้ง loop), `next` (ข้ามไปรอบถัดไป), และ `redo` (ทำรอบ
  เดิมซ้ำ) ได้อย่างแม่นยำ พร้อมรู้ว่า `next value` กำหนดค่าคืนให้กับ `map` ได้อย่างไร
- ประยุกต์ทุกเรื่องที่เรียนมาสร้างโปรแกรมเกมทายตัวเลขและ FizzBuzz ขั้นสูงได้ครบวงจร

**ต่อไป (Part 007):** เราจะเจาะลึกเรื่อง **Methods** — การนิยาม method ด้วย `def`, รูปแบบ
argument ทั้งหมดที่ Ruby รองรับ (positional, keyword, default value, splat `*args`, double
splat `**kwargs`), และกลไกการคืนค่า (return values) รวมถึงข้อแตกต่างระหว่าง implicit return
กับการใช้ `return` ชัดเจน
