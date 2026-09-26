# Part 007: Methods — การนิยาม method, arguments ทุกรูปแบบ, และ return values

> **Step ครอบคลุมใน Part นี้:** Step 61–70
> **ระดับ:** เริ่มต้น–กลาง (ต้องผ่าน Part 001–006 มาก่อน โดยเฉพาะเรื่อง control flow)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 61: การนิยาม method ด้วย `def`/`end`, การเรียกใช้, implicit return vs explicit return
- Step 62: Positional arguments — argument แบบตำแหน่ง และ arity
- Step 63: Default argument values และกับดักเรื่องลำดับการ evaluate
- Step 64: Keyword arguments — required และ default, ทำไม Rails ชอบใช้ keyword arguments
- Step 65: Splat argument (`*args`) — รับ argument จำนวนไม่จำกัด
- Step 66: Double splat argument (`**kwargs`) — รับ keyword arguments จำนวนไม่จำกัด
- Step 67: ผสม argument ทุกแบบเข้าด้วยกัน และกฎลำดับที่ Ruby บังคับ
- Step 68: ธรรมเนียมตั้งชื่อ method ด้วย `?` และ `!`
- Step 69: Method visibility — `public`, `private`, `protected` และเกริ่น `method_missing`
- Step 70: แบบฝึกหัด — ระบบคำนวณยอดสั่งซื้อ (Order Total Calculator)

---

## Step 61: การนิยาม method ด้วย `def`/`end`, การเรียกใช้, implicit return vs explicit return

### ทำไมต้องมี method

ตั้งแต่ Part 001–006 เราเขียนโค้ดแบบ top-level มาตลอด ปัญหาคือถ้าต้องทำสิ่งเดียวกันซ้ำๆ
หลายที่ (เช่น คำนวณราคาหลังหักส่วนลด) เราต้อง copy-paste โค้ดชุดเดิมซ้ำไปเรื่อยๆ ซึ่งขัดกับ
หลักการ **DRY (Don't Repeat Yourself)** ที่กล่าวถึงใน Part 001

**Method** คือการห่อหุ้มชุดคำสั่งไว้เป็นก้อนเดียว ตั้งชื่อให้มัน แล้วเรียกใช้ซ้ำได้ทุกที่
ที่ต้องการ เป็นหน่วยพื้นฐานที่สุดของการจัดโครงสร้างโปรแกรม และเป็นรากฐานสำคัญก่อนจะไปถึง
เรื่อง Object-Oriented Programming (OOP) ใน Part 009–010

### syntax พื้นฐาน

```ruby
# frozen_string_literal: true

def greet
  puts "สวัสดี!"
end

greet       # เรียกแบบไม่มีวงเล็บ => สวัสดี!
greet()     # เรียกแบบมีวงเล็บ ผลลัพธ์เหมือนกัน => สวัสดี!
```

**อธิบาย:**

- `def` เปิดการนิยาม method, `end` ปิด (คู่กันเหมือน `if...end`, `while...end` ที่เจอมาแล้ว)
- ชื่อ method ต้องเป็น **snake_case** ตามธรรมเนียม Ruby ทั้งหมด (`greet`, `calculate_total`,
  ไม่ใช่ `Greet` หรือ `calculateTotal` แบบ camelCase ของภาษาอื่น)
- การเรียก method ที่ไม่มี argument สามารถละวงเล็บได้ — ในทางปฏิบัติ Ruby community นิยม
  ละวงเล็บเมื่อไม่มี argument และใส่วงเล็บเมื่อมี argument เพื่อความอ่านง่าย

### method ที่รับ argument

```ruby
def greet(name)
  puts "สวัสดี, #{name}!"
end

greet("มานี")   # => สวัสดี, มานี!
greet "สมชาย"   # => สวัสดี, สมชาย! (ละวงเล็บได้ แต่ไม่แนะนำเมื่อโค้ดซับซ้อนขึ้น)
```

ถ้าเรียก method โดยจำนวน argument ไม่ตรงกับที่นิยามไว้ Ruby จะโยน `ArgumentError` ทันที
(รายละเอียดเรื่อง arity จะพูดถึงใน Step 62)

```ruby
greet  # ArgumentError: wrong number of arguments (given 0, expected 1)
```

### Implicit return — ค่าที่ method คืนกลับโดยอัตโนมัติ

จุดที่แตกต่างจากภาษาอื่นอย่างชัดเจน: **method ใน Ruby จะคืนค่าของ expression บรรทัด
สุดท้ายที่ถูกรันเสมอ โดยไม่ต้องเขียน `return`**

```ruby
def add(a, b)
  a + b   # บรรทัดสุดท้าย -> ค่านี้จะถูก return โดยอัตโนมัติ
end

result = add(3, 5)
p result  # => 8
```

```ruby
def describe_number(n)
  if n.even?
    "#{n} เป็นเลขคู่"
  else
    "#{n} เป็นเลขคี่"
  end
  # ค่าของ if/else expression บรรทัดสุดท้ายคือค่า return ของ method
end

puts describe_number(4)  # => 4 เป็นเลขคู่
puts describe_number(7)  # => 7 เป็นเลขคี่
```

**ทำไมเรื่องนี้สำคัญ:** เพราะใน Ruby แทบทุกอย่างเป็น expression ที่มีค่า (`if`, `case`,
`begin/rescue` ก็มีค่า) การเข้าใจ implicit return ทำให้เขียนโค้ดสั้นกระชับได้โดยไม่ต้องพึ่ง
`return` พร่ำเพรื่อ — สไตล์นี้เป็นสิ่งที่โค้ด Rails ระดับมืออาชีพใช้เป็นมาตรฐาน

### Explicit return — เมื่อไหร่ควรใช้ `return`

`return` ใช้เพื่อ **ออกจาก method ทันที** พร้อมคืนค่าที่ระบุ มีประโยชน์ในกรณี early return
(ออกจาก method ก่อนถึงบรรทัดสุดท้าย เพื่อลด nested if)

```ruby
def discount_price(price, member: false)
  return price if price <= 0   # guard clause: ป้องกันราคาติดลบหรือศูนย์ ออกจาก method ทันที

  if member
    price * 0.9
  else
    price
  end
end

p discount_price(1000, member: true)   # => 900.0
p discount_price(0, member: true)      # => 0 (ออกจาก method ตั้งแต่บรรทัดแรก)
```

**เทียบสไตล์การเขียน 2 แบบ:**

```ruby
# แบบใช้ return ชัดเจนทุกจุด (สไตล์ภาษาอื่น เช่น Java/JavaScript)
def classify_age_v1(age)
  if age < 13
    return "เด็ก"
  elsif age < 20
    return "วัยรุ่น"
  else
    return "ผู้ใหญ่"
  end
end

# แบบใช้ implicit return (สไตล์ Ruby ที่นิยมมากกว่า)
def classify_age_v2(age)
  if age < 13
    "เด็ก"
  elsif age < 20
    "วัยรุ่น"
  else
    "ผู้ใหญ่"
  end
end
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ แต่แบบที่สองเป็นที่นิยมกว่าในโค้ด Ruby/Rails มืออาชีพ
เพราะสั้นกว่าและลด noise ที่ไม่จำเป็น RuboCop (ที่ติดตั้งใน Part 001) มี cop ชื่อ
`Style/RedundantReturn` ที่จะเตือนถ้าใส่ `return` ที่บรรทัดสุดท้ายโดยไม่จำเป็น

> **กฎง่ายๆ ในการเลือกใช้:** ใช้ implicit return เป็นค่าเริ่มต้นเสมอ ใช้ `return` เฉพาะตอน
> ต้องการ **ออกจาก method ก่อนถึงบรรทัดสุดท้าย** (early return / guard clause) เท่านั้น

### การคืนค่าหลายค่า (multiple return values)

Ruby ไม่มี syntax พิเศษสำหรับ "return หลายค่า" จริงๆ แต่ใช้กลไก Array ร่วมกับ
destructuring (ที่เคยเห็นตอน Array ใน Part 004) ทำให้ดูเหมือนคืนค่าได้หลายค่า

```ruby
def min_max(numbers)
  [numbers.min, numbers.max]   # จริงๆ คืนค่าเป็น Array 1 ตัว ที่มี 2 element
end

low, high = min_max([5, 2, 9, 1, 7])
puts "ต่ำสุด: #{low}, สูงสุด: #{high}"
# => ต่ำสุด: 1, สูงสุด: 9

result = min_max([3, 8, 1])
p result  # => [1, 8]  (ถ้าไม่ destructure ก็ได้ Array ตรงๆ)
```

---

## Step 62: Positional arguments — argument แบบตำแหน่ง และ arity

**Positional argument** คือ argument รูปแบบพื้นฐานที่สุด ที่ Ruby จับคู่ค่ากับ parameter
ตาม **ตำแหน่ง (position)** ที่ส่งเข้ามา ไม่ใช่ตามชื่อ

```ruby
def full_name(first, last)
  "#{first} #{last}"
end

puts full_name("สมชาย", "ใจดี")   # => สมชาย ใจดี
# first = "สมชาย" (ตำแหน่งที่ 1), last = "ใจดี" (ตำแหน่งที่ 2)

puts full_name("ใจดี", "สมชาย")   # => ใจดี สมชาย
# สลับตำแหน่ง -> ผลลัพธ์สลับตาม เพราะจับคู่ตามตำแหน่งเท่านั้น ไม่รู้ความหมาย
```

นี่คือจุดอ่อนสำคัญของ positional argument: **เมื่อ method มี argument หลายตัว โอกาสสลับ
ลำดับผิดโดยไม่มี error เตือนมีสูงมาก** โดยเฉพาะถ้า argument มี type เดียวกัน (เช่น
String ทั้งคู่) — ปัญหานี้จะถูกแก้ด้วย keyword argument ใน Step 64

### Arity — จำนวน argument ที่ method ต้องการ

**Arity** คือจำนวน argument ที่ method คาดหวัง เราตรวจสอบได้ด้วย `Method#arity`

```ruby
def add(a, b)
  a + b
end

puts method(:add).arity   # => 2  (ต้องการ argument พอดี 2 ตัว)
```

ถ้าจำนวน argument ที่ส่งเข้ามาไม่ตรงกับ arity ของ method (สำหรับ positional argument
ล้วนๆ) Ruby จะโยน `ArgumentError` ทันทีตอนเรียก ไม่ใช่ตอนรันโปรแกรมเสร็จ — เป็นการป้องกัน
บัคตั้งแต่เนิ่นๆ (fail fast)

```ruby
add(1)          # ArgumentError: wrong number of arguments (given 1, expected 2)
add(1, 2, 3)    # ArgumentError: wrong number of arguments (given 3, expected 2)
add(1, 2)       # => 3 (ถูกต้อง)
```

### method หลาย argument ต้องระวังลำดับ

```ruby
def create_rectangle_area(width, height)
  width * height
end

# ปัญหา: ถ้า width/height เป็น Integer ทั้งคู่ สลับกันก็ไม่มี error
# แต่ผลลัพธ์อาจผิดถ้าสูตรไม่ symmetric เช่นสูตรคำนวณดอกเบี้ย, ระยะทาง ฯลฯ

def calculate_discount(price, percent)
  price - (price * percent / 100.0)
end

puts calculate_discount(1000, 10)   # => 900.0 (ถูกต้อง: ราคา 1000 ลด 10%)
puts calculate_discount(10, 1000)   # => -90.0 (สลับผิด! ได้ผลลัพธ์แปลกๆ แต่ไม่มี error เตือน)
```

ตัวอย่างข้างบนแสดงให้เห็นว่า positional argument เหมาะกับกรณีที่ **จำนวน argument น้อย
(1–2 ตัว) และลำดับเป็นธรรมชาติชัดเจน** (เช่น `add(a, b)`, `full_name(first, last)`)
แต่เมื่อ argument เยอะขึ้นหรือความหมายไม่ชัดเจนจากตำแหน่ง ควรเปลี่ยนไปใช้ keyword
argument แทน (Step 64)

---

## Step 63: Default argument values และกับดักเรื่องลำดับการ evaluate

บ่อยครั้งเราต้องการให้ argument บางตัว "มีค่าเริ่มต้น" ถ้าผู้เรียกไม่ได้ระบุมา
Ruby รองรับด้วย **default argument value**

```ruby
def greet(name, greeting = "สวัสดี")
  "#{greeting}, #{name}!"
end

puts greet("มานี")                # => สวัสดี, มานี!  (ใช้ default)
puts greet("มานี", "หวัดดี")      # => หวัดดี, มานี!  (override default)
```

### กฎ: parameter ที่มี default ต้องอยู่หลัง (หรือ Ruby จะพยายามเดา)

```ruby
# นิยมวาง argument ที่ไม่มี default ไว้ก่อน แล้วตามด้วยตัวที่มี default
def order_summary(item, quantity = 1, currency = "THB")
  "#{item} x#{quantity} (#{currency})"
end

puts order_summary("กาแฟ")                    # => กาแฟ x1 (THB)
puts order_summary("กาแฟ", 3)                  # => กาแฟ x3 (THB)
puts order_summary("กาแฟ", 3, "USD")           # => กาแฟ x3 (USD)
```

Ruby อนุญาตให้ argument ที่ไม่มี default ตามหลัง argument ที่มี default ได้เหมือนกัน
(เช่น `def foo(a, b = 1, c)`) โดย Ruby จะพยายามจับคู่ตำแหน่งให้ฉลาดที่สุด แต่ **ไม่แนะนำ
ให้เขียนแบบนี้** เพราะอ่านยากและสร้างความสับสน ให้ยึดกฎง่ายๆ ว่า: **argument ที่ไม่มี
default มาก่อน ตามด้วย argument ที่มี default เสมอ**

```ruby
# เขียนได้ (Ruby รองรับ) แต่ไม่ควรทำ เพราะอ่านสับสน
def confusing(a, b = 10, c)
  [a, b, c]
end

p confusing(1, 2)      # => [1, 10, 2]  (b ใช้ default, c = 2 — งงมาก!)
p confusing(1, 2, 3)   # => [1, 2, 3]
```

### กับดักสำคัญ: default value ถูก evaluate ใหม่ทุกครั้งที่เรียก method

จุดที่มือใหม่มักเข้าใจผิดคือคิดว่า default value ถูกคำนวณครั้งเดียวตอนนิยาม method
(เหมือนบางภาษา เช่น Python ที่ default argument เป็น mutable object จะถูกสร้างครั้งเดียว
และแชร์กันข้ามการเรียกทุกครั้ง — เป็นบัคที่โด่งดังใน Python) **แต่ Ruby ไม่เป็นแบบนั้น**
default expression จะถูก evaluate **ใหม่ทุกครั้ง** ที่ method ถูกเรียกและไม่มีการส่ง
argument ตัวนั้นมา

```ruby
def add_item(item, list = [])
  list << item
  list
end

p add_item("แอปเปิ้ล")   # => ["แอปเปิ้ล"]
p add_item("กล้วย")      # => ["กล้วย"]  (ไม่ใช่ ["แอปเปิ้ล", "กล้วย"]!)
# เพราะทุกครั้งที่เรียกโดยไม่ส่ง list มา จะได้ [] ใหม่เอี่ยมเสมอ ไม่ใช่ Array ตัวเดิมที่แชร์กัน
```

นี่คือพฤติกรรมที่ **ถูกต้องและปลอดภัยกว่า** เมื่อเทียบกับภาษาที่มี mutable default
argument เป็นกับดัก แต่สิ่งที่ต้องระวังคือทิศทางตรงข้าม: **ถ้าตั้งใจจะแชร์ state ข้าม
การเรียก ต้องคิดใหม่** เพราะ Ruby จะไม่ทำแบบนั้นให้อัตโนมัติ

### default value อ้างอิง parameter ตัวอื่นได้

อีกจุดที่มีประโยชน์มาก: default expression สามารถอ้างอิงถึง parameter ตัวก่อนหน้าได้
(เพราะ evaluate ตามลำดับซ้ายไปขวาตอนเรียกจริง)

```ruby
def rectangle_info(width, height = width)
  # ถ้าไม่ระบุ height จะใช้ค่า width (สี่เหลี่ยมจัตุรัส)
  area = width * height
  "กว้าง #{width} ยาว #{height} พื้นที่ #{area}"
end

puts rectangle_info(5)       # => กว้าง 5 ยาว 5 พื้นที่ 25
puts rectangle_info(5, 3)    # => กว้าง 5 ยาว 3 พื้นที่ 15
```

```ruby
# ตัวอย่างที่ default อ้างอิง method อื่น (ถูก evaluate สดใหม่ทุกครั้ง)
def log_event(message, timestamp = Time.now)
  "[#{timestamp}] #{message}"
end

puts log_event("เริ่มระบบ")
sleep 1
puts log_event("จบการทำงาน")
# timestamp ของทั้ง 2 บรรทัดต่างกัน เพราะ Time.now ถูกเรียกใหม่ทุกครั้ง ไม่ใช่ตอนนิยาม method
```

---

## Step 64: Keyword arguments — required และ default, ทำไม Rails ชอบใช้ keyword arguments

**Keyword argument** คือการส่ง argument โดยระบุ "ชื่อ" กำกับไปด้วย แทนที่จะพึ่งตำแหน่ง
เพียงอย่างเดียว แก้ปัญหาที่พูดถึงใน Step 62 (สลับลำดับ argument โดยไม่รู้ตัว) ได้ตรงจุด

### keyword argument แบบ required (บังคับต้องระบุ)

```ruby
def create_user(name:, email:)
  "สร้างผู้ใช้: #{name} (#{email})"
end

puts create_user(name: "มานี", email: "manee@example.com")
# => สร้างผู้ใช้: มานี (manee@example.com)

# สลับลำดับได้อย่างปลอดภัย เพราะจับคู่ด้วยชื่อ ไม่ใช่ตำแหน่ง
puts create_user(email: "somchai@example.com", name: "สมชาย")
# => สร้างผู้ใช้: สมชาย (somchai@example.com)
```

**สังเกต syntax:** parameter ที่ประกาศด้วย `name:` (มี colon ต่อท้าย ไม่มีค่า default)
หมายถึง keyword argument ที่ **จำเป็นต้องระบุ** ถ้าไม่ระบุจะได้ `ArgumentError`

```ruby
create_user(name: "มานี")
# ArgumentError: missing keyword: :email
```

### keyword argument ที่มี default value

```ruby
def create_user(name:, email:, role: "member", active: true)
  "#{name} (#{email}) — role: #{role}, active: #{active}"
end

puts create_user(name: "มานี", email: "manee@example.com")
# => มานี (manee@example.com) — role: member, active: true

puts create_user(name: "แอดมิน", email: "admin@example.com", role: "admin")
# => แอดมิน (admin@example.com) — role: admin, active: true

puts create_user(name: "ปิดใช้งาน", email: "x@example.com", active: false)
# => ปิดใช้งาน (x@example.com) — role: member, active: false
```

**ข้อดีที่เห็นชัดจากตัวอย่างนี้:** ผู้เรียก method สามารถส่งเฉพาะ keyword ที่ต้องการ
override โดยไม่ต้องสนใจลำดับ และไม่ต้องส่งค่าที่ต้องการใช้ default ซ้ำๆ ต่างจาก
positional default argument ที่ถ้าต้องการ override ตัวสุดท้าย ต้องระบุตัวก่อนหน้าทั้งหมด
มาด้วย

### ทำไม Rails/Ruby community นิยมใช้ keyword arguments

1. **อ่านง่ายตรงจุดที่เรียกใช้ (self-documenting call site)** — โค้ดที่เรียก method
   บอกความหมายของแต่ละค่าในตัวมันเอง ไม่ต้องเปิดไปดู definition ของ method

   ```ruby
   # positional: ต้องเดาว่า true ตัวไหนคือ active, ตัวไหนคือ verified
   create_account("มานี", true, false, "admin")

   # keyword: อ่านแล้วเข้าใจทันทีโดยไม่ต้องดู method definition
   create_account(name: "มานี", active: true, verified: false, role: "admin")
   ```

2. **สลับลำดับได้อย่างปลอดภัย** — แก้ปัญหา arity trap ที่กล่าวถึงใน Step 62 ได้เต็มๆ

3. **เพิ่ม/ลด parameter ในอนาคตโดยไม่พัง call site เดิม** — ถ้าเพิ่ม keyword ใหม่ที่มี
   default ทุก call site เดิมยังทำงานปกติ ต่างจาก positional argument ที่การแทรก
   parameter ใหม่ตรงกลางจะทำให้ลำดับทั้งหมดหลังจากนั้นเลื่อนและพังหมด

4. **Rails ใช้ pattern นี้เป็นมาตรฐานทั่วทั้ง framework** เช่น

   ```ruby
   # ตัวอย่างที่จะเจอจริงตอนเรียน Rails (แสดงให้ดูล่วงหน้าเพื่อให้เห็นภาพ)
   redirect_to root_path, notice: "บันทึกสำเร็จ", status: :see_other
   validates :email, presence: true, uniqueness: true
   has_many :orders, dependent: :destroy, class_name: "Order"
   ```

   จะสังเกตได้ว่า API ของ Rails แทบทุกจุดเลือกใช้ keyword argument (ในรูป Hash ท้าย
   method call) เพราะทำให้โค้ดอ่านเหมือนประโยคภาษาอังกฤษ ตรงกับปรัชญา "Optimize for
   programmer happiness" ที่กล่าวถึงใน Part 001

### ข้อควรระวัง: keyword argument ที่ไม่รู้จักจะ error

```ruby
def create_user(name:, email:)
  "#{name} - #{email}"
end

create_user(name: "มานี", email: "x@example.com", age: 20)
# ArgumentError: unknown keyword: :age
```

พฤติกรรมนี้ (error ทันทีถ้าส่ง keyword ที่ไม่รู้จัก) เป็นข้อดี เพราะช่วยจับ typo ได้ตั้งแต่
เนิ่นๆ เช่นถ้าพิมพ์ `emial:` ผิดแทน `email:` จะได้ error ทันทีแทนที่จะเงียบแล้วค่อยพังทีหลัง

---

## Step 65: Splat argument (`*args`) — รับ argument จำนวนไม่จำกัด

บางครั้งเราไม่รู้ล่วงหน้าว่าผู้เรียก method จะส่ง positional argument มากี่ตัว
**Splat operator (`*`)** แก้ปัญหานี้ด้วยการรวบรวม argument ที่เหลือทั้งหมดไว้ใน Array

```ruby
def sum(*numbers)
  numbers.sum
end

p sum(1, 2)              # => 3
p sum(1, 2, 3, 4, 5)     # => 15
p sum()                  # => 0  (ไม่ส่ง argument เลยก็ได้ numbers = [])
p sum(10)                # => 10
```

**อธิบาย:** ภายใน method, `numbers` คือ Array ธรรมดาที่เก็บ argument ทั้งหมดที่ส่งเข้ามา
ไม่ว่าจะกี่ตัวก็ตาม ทำให้ method เดียวรองรับจำนวน argument ที่ยืดหยุ่นได้เต็มที่

### ผสม splat กับ positional argument ปกติ

```ruby
def introduce(leader, *members)
  puts "หัวหน้าทีม: #{leader}"
  puts "สมาชิก: #{members.join(', ')}"
end

introduce("มานี", "สมชาย", "สมหญิง", "วิชัย")
# => หัวหน้าทีม: มานี
# => สมาชิก: สมชาย, สมหญิง, วิชัย

introduce("มานี")
# => หัวหน้าทีม: มานี
# => สมาชิก:   (members = [] เพราะไม่มี argument เหลือ)
```

`leader` จับคู่ตำแหน่งแรกเสมอ ส่วนที่เหลือทั้งหมด (ไม่ว่ากี่ตัว) จะถูกรวบเข้า `members`

### การ "แยกร่าง" Array ตอนเรียก method ด้วย splat (การ unpack)

Splat operator ใช้ได้ทั้ง 2 ทิศทาง — ตอนนิยาม method (รวบรวม) และ **ตอนเรียก method
(กระจาย Array ออกเป็น argument แยก)**

```ruby
def add_three(a, b, c)
  a + b + c
end

numbers = [1, 2, 3]
puts add_three(*numbers)   # เทียบเท่า add_three(1, 2, 3) => 6

# ถ้าไม่ใช้ * จะ error เพราะ Ruby มองว่าส่ง argument แค่ 1 ตัว (คือ Array ทั้งก้อน)
puts add_three(numbers)
# ArgumentError: wrong number of arguments (given 1, expected 3)
```

นี่คือประโยชน์สำคัญของ splat: แปลง Array ที่มีอยู่แล้วให้กลายเป็น argument list ได้ทันที
โดยไม่ต้องเขียน `numbers[0], numbers[1], numbers[2]` เอง

### ตั้งชื่อ splat parameter ให้สื่อความหมาย

```ruby
# ไม่ดี: ชื่อ args ไม่สื่อความหมาย
def log(*args)
  puts args.join(" | ")
end

# ดีกว่า: ตั้งชื่อให้บอกว่าเก็บอะไร
def log_messages(*messages)
  puts messages.join(" | ")
end

log_messages("เริ่มระบบ", "โหลดข้อมูล", "พร้อมใช้งาน")
# => เริ่มระบบ | โหลดข้อมูล | พร้อมใช้งาน
```

---

## Step 66: Double splat argument (`**kwargs`) — รับ keyword arguments จำนวนไม่จำกัด

เหมือนกับที่ `*` รวบรวม positional argument ที่ไม่รู้จำนวนแน่นอนเข้า Array,
**double splat (`**`)** ก็ทำหน้าที่เดียวกันแต่กับ **keyword argument** โดยรวบรวมเข้าเป็น
Hash

```ruby
def build_query(**options)
  options
end

p build_query(status: "active", limit: 10)
# => {status: "active", limit: 10}

p build_query(name: "มานี", age: 25, city: "กรุงเทพ")
# => {name: "มานี", age: 25, city: "กรุงเทพ"}

p build_query()
# => {}
```

### ตัวอย่างที่ใช้งานได้จริง: สร้าง HTML attribute string

```ruby
def html_attributes(**attrs)
  attrs.map { |key, value| %(#{key}="#{value}") }.join(" ")
end

puts html_attributes(id: "main", class: "container", data_role: "form")
# => id="main" class="container" data_role="form"
```

ตัวอย่างนี้ใกล้เคียงกับสิ่งที่ Rails view helper อย่าง `content_tag` หรือ `link_to`
ทำงานเบื้องหลังจริงๆ — รับ keyword จำนวนเท่าไหร่ก็ได้ แล้วแปลงเป็น HTML attribute

### ผสม required keyword กับ double splat

```ruby
def create_product(name:, price:, **extra_attributes)
  puts "สินค้า: #{name}, ราคา: #{price}"
  puts "attribute เพิ่มเติม: #{extra_attributes}" unless extra_attributes.empty?
end

create_product(name: "เสื้อยืด", price: 250)
# => สินค้า: เสื้อยืด, ราคา: 250

create_product(name: "เสื้อยืด", price: 250, color: "แดง", size: "L")
# => สินค้า: เสื้อยืด, ราคา: 250
# => attribute เพิ่มเติม: {color: "แดง", size: "L"}
```

`name:` และ `price:` ถูกดึงออกมาจับคู่กับ parameter ที่ระบุชื่อไว้ก่อน ส่วน keyword
argument ที่เหลือทั้งหมด (ที่ไม่ตรงกับชื่อไหนเลย) จะถูกรวบเข้า `extra_attributes`
โดยอัตโนมัติ — มีประโยชน์มากตอนสร้าง method ที่ต้อง "รับทุกอย่างแล้วส่งต่อ" (delegation)

### การ "แยกร่าง" Hash ตอนเรียก method ด้วย double splat

เช่นเดียวกับ splat, double splat ใช้ตอนเรียก method เพื่อกระจาย Hash ออกเป็น keyword
argument ได้เช่นกัน

```ruby
def create_user(name:, email:, role: "member")
  "#{name} (#{email}) - #{role}"
end

user_data = { name: "มานี", email: "manee@example.com", role: "admin" }
puts create_user(**user_data)
# => มานี (manee@example.com) - admin

# เทียบเท่ากับ
puts create_user(name: "มานี", email: "manee@example.com", role: "admin")
```

นี่เป็นเทคนิคที่ใช้บ่อยมากใน Rails เช่นตอนส่งต่อ `params` (ที่เป็น Hash-like object)
เข้า method อื่น: `SomeService.call(**params)`

---

## Step 67: ผสม argument ทุกแบบเข้าด้วยกัน และกฎลำดับที่ Ruby บังคับ

ในทางปฏิบัติ method หนึ่งตัวสามารถมี argument ได้หลายแบบผสมกัน แต่ **Ruby บังคับลำดับ
การประกาศ parameter อย่างเคร่งครัด** ถ้าเขียนผิดลำดับจะได้ `SyntaxError` ทันที

### ลำดับที่ถูกต้องคือ:

```
def method_name(
  positional_required,       # 1. positional argument แบบบังคับ
  positional_optional = val, # 2. positional argument แบบมี default
  *splat_args,                # 3. splat (รวบ positional ที่เหลือ)
  keyword_required:,          # 4. keyword argument แบบบังคับ
  keyword_optional: val,      # 5. keyword argument แบบมี default
  **double_splat_kwargs,      # 6. double splat (รวบ keyword ที่เหลือ)
  &block                      # 7. block argument (จะสอนละเอียดใน Part 008)
)
```

### ตัวอย่างจริงที่ผสมทุกแบบ

```ruby
def process_order(customer, quantity = 1, *extra_items, priority:, note: "ไม่มี", **metadata)
  puts "ลูกค้า: #{customer}"
  puts "จำนวนหลัก: #{quantity}"
  puts "สินค้าเพิ่มเติม: #{extra_items}" unless extra_items.empty?
  puts "ความสำคัญ: #{priority}"
  puts "หมายเหตุ: #{note}"
  puts "ข้อมูลเพิ่มเติม: #{metadata}" unless metadata.empty?
end

process_order("มานี", priority: :high)
# => ลูกค้า: มานี
# => จำนวนหลัก: 1
# => ความสำคัญ: high
# => หมายเหตุ: ไม่มี

process_order("สมชาย", 3, "ของแถม A", "ของแถม B", priority: :normal, note: "ส่งด่วน", gift_wrap: true)
# => ลูกค้า: สมชาย
# => จำนวนหลัก: 3
# => สินค้าเพิ่มเติม: ["ของแถม A", "ของแถม B"]
# => ความสำคัญ: normal
# => หมายเหตุ: ส่งด่วน
# => ข้อมูลเพิ่มเติม: {gift_wrap: true}
```

**อธิบายการจับคู่ทีละส่วน (เรียกครั้งที่ 2):**

1. `"สมชาย"` -> `customer` (positional required ตัวแรกเสมอ)
2. `3` -> `quantity` (positional optional ตัวที่สอง มีค่ามาก็ override default)
3. `"ของแถม A", "ของแถม B"` -> `extra_items` (ที่เหลือจาก positional ถูกรวบด้วย splat)
4. `priority: :normal` -> ตรงกับ `priority:` (required keyword)
5. `note: "ส่งด่วน"` -> ตรงกับ `note:` (optional keyword, override default)
6. `gift_wrap: true` -> ไม่ตรงกับ keyword ไหนที่ประกาศไว้ -> ถูกรวบเข้า `**metadata`

### ทำไม Ruby ต้องบังคับลำดับนี้

เหตุผลเชิงเทคนิคคือ Ruby ต้อง parse argument list จากซ้ายไปขวาแบบไม่กำกวม
(unambiguous) — ถ้าอนุญาตให้ positional required ตามหลัง splat หรือ keyword ตามหลัง
double splat ได้ จะไม่มีทางรู้ได้แน่ชัดว่า argument ตัวไหนควรจับคู่กับ parameter ไหน
กฎลำดับนี้จึงเป็นสิ่งที่ทุกคนที่เขียน Ruby ต้องจำ

```ruby
# ตัวอย่าง SyntaxError ถ้าเขียนผิดลำดับ
def bad_order(*args, positional)
end
# SyntaxError: unexpected local variable, expecting '='... (ตำแหน่งไม่ถูกต้อง)
```

### แนวปฏิบัติในโค้ดจริง

ในทางปฏิบัติ **ไม่ควรใช้ argument ครบทุกแบบในตัวเดียวกัน** เพราะจะทำให้ method อ่านยาก
และ maintain ยาก กฎทั่วไปที่ทีมมืออาชีพใช้:

- ถ้า method มี argument ไม่เกิน 1–2 ตัวและความหมายชัดเจนจากตำแหน่ง -> ใช้ positional
- ถ้า method มี argument ตั้งแต่ 2 ตัวขึ้นไป โดยเฉพาะที่มีตัวเลือก (optional) -> ใช้
  keyword argument
- ใช้ splat/double splat เมื่อต้องการความยืดหยุ่นจริงๆ เช่น method ที่ห่อหุ้ม
  (wrap/delegate) การเรียก method อื่นอีกที

```ruby
# ตัวอย่างการใช้ splat + double splat เพื่อ "ส่งต่อ" argument ทั้งหมดให้ method อื่น
# (pattern นี้เจอบ่อยมากตอนเขียน Rails: delegate/decorator pattern)
def logged_call(method_name, *args, **kwargs)
  puts "กำลังเรียก #{method_name} ด้วย args=#{args}, kwargs=#{kwargs}"
  send(method_name, *args, **kwargs)
end
```

---

## Step 68: ธรรมเนียมตั้งชื่อ method ด้วย `?` และ `!`

Ruby อนุญาตให้ชื่อ method ลงท้ายด้วยเครื่องหมาย `?` หรือ `!` ได้ (ต่างจากภาษาอื่นส่วนใหญ่)
ซึ่งไม่ใช่แค่ syntax sugar แต่เป็น **ธรรมเนียมที่สื่อความหมายสำคัญให้คนอ่านโค้ด**

### method ที่ลงท้ายด้วย `?` — Predicate method

ใช้กับ method ที่ **คืนค่า boolean เท่านั้น (`true`/`false`)** เพื่อให้อ่านโค้ดแล้วรู้ทันที
ว่า method นี้ใช้ตอบคำถาม ใช่/ไม่ใช่

เราเจอ predicate method ของ Ruby มาแล้วหลาย Part เช่น `even?`, `empty?`, `nil?`,
`include?` (Part 002–006) — ตอนนี้เราสามารถสร้างของตัวเองได้

```ruby
def adult?(age)
  age >= 20
end

puts adult?(25)   # => true
puts adult?(15)   # => false

if adult?(18)
  puts "เข้าถึงเนื้อหาได้"
else
  puts "ยังไม่บรรลุนิติภาวะ"
end
```

```ruby
def valid_email?(email)
  email.match?(/\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i)
end

puts valid_email?("manee@example.com")   # => true
puts valid_email?("ไม่ใช่อีเมล")           # => false
```

> **กฎสำคัญ:** method ที่ลงท้าย `?` ควรคืนค่า `true`/`false` เท่านั้น (หรืออย่างน้อยก็
> ค่าที่ใช้เป็น boolean ได้อย่างชัดเจน) ไม่ควรคืน String หรือ Integer เพราะจะทำให้คนอ่าน
> โค้ดเข้าใจผิด — ถือเป็นสัญญา (convention) ที่ทั้ง Ruby community ยึดถือร่วมกัน

### method ที่ลงท้ายด้วย `!` — Dangerous method

ใช้กับ method ที่ **"อันตราย" กว่าคู่ของมันที่ไม่มี `!`** ความหมาย "อันตราย" ใน Ruby
โดยทั่วไปหมายถึง 2 กรณี:

1. **แก้ไขค่าเดิม (mutate) แทนที่จะคืนค่าใหม่** — เคยเจอมาแล้วใน Part 003–004:
   `upcase` vs `upcase!`, `sort` vs `sort!`, `reverse` vs `reverse!`

   ```ruby
   name = "มานี"
   upper = name.upcase     # คืนค่าใหม่ ไม่แก้ name เดิม
   puts name                # => มานี (เหมือนเดิม)
   puts upper                # => MANEE (String อังกฤษถูก upcase; ตัวอย่างสมมติ)

   greeting = "hello"
   greeting.upcase!          # แก้ไข greeting เดิมโดยตรง (mutate in place)
   puts greeting              # => HELLO (เปลี่ยนไปแล้ว!)
   ```

2. **โยน exception เมื่อ operation ล้มเหลว แทนที่จะคืน `nil` เงียบๆ** —
   ตัวอย่างคลาสสิกคือ `save` vs `save!` ใน Active Record (จะเรียนใน Part 025):
   `save` คืน `false` ถ้าบันทึกไม่สำเร็จ ส่วน `save!` จะโยน exception ทันที

### เขียน method คู่ `!` เอง

```ruby
def normalize(text)
  text.strip.downcase
end

def normalize!(text)
  text.strip!
  text.downcase!
  text
end

original = "  HELLO WORLD  "
result = normalize(original)
puts original   # =>   HELLO WORLD   (ไม่เปลี่ยน)
puts result      # => hello world

text2 = "  RUBY IS FUN  "
normalize!(text2)
puts text2   # => ruby is fun  (ตัวแปรเดิมถูกแก้ไข เพราะ String เป็น mutable object)
```

> **ข้อควรระวัง:** `?` และ `!` เป็นแค่ **ธรรมเนียม** ไม่ใช่กฎที่ Ruby บังคับทาง syntax
> เราสามารถเขียน method ที่ลงท้าย `!` แต่ไม่ mutate อะไรเลยก็ได้ (Ruby ไม่ error)
> แต่การทำแบบนั้นจะทำให้คนอ่านโค้ดเข้าใจผิดและเป็นบัคทางการสื่อสาร (footgun) — ควรยึด
> ธรรมเนียมนี้อย่างเคร่งครัดเสมอ

### สรุปกฎการตั้งชื่อ

| ท้ายชื่อ | ความหมาย | ตัวอย่างจาก Ruby core |
|---|---|---|
| (ไม่มี) | method ปกติ คืนค่าใหม่ ไม่แก้ต้นฉบับ | `upcase`, `sort`, `map` |
| `?` | คืนค่า boolean, ใช้ถามคำถาม | `even?`, `empty?`, `nil?` |
| `!` | อันตราย: mutate ต้นฉบับ หรือโยน exception เมื่อล้มเหลว | `upcase!`, `sort!`, `save!` |

---

## Step 69: Method visibility — `public`, `private`, `protected` และเกริ่น `method_missing`

เรื่อง visibility เกี่ยวข้องโดยตรงกับ class/object (ซึ่งจะเรียนเต็มรูปแบบใน Part 009)
แต่เนื่องจากเป็นคุณสมบัติของ **method** โดยตรง เราจะปูพื้นฐานไว้ตรงนี้ก่อน แล้วไปขยายผล
ต่อในเรื่อง OOP

### `public` — ค่าเริ่มต้นของทุก method

Method ทุกตัวที่นิยามด้วย `def` ภายใน class จะเป็น `public` โดยอัตโนมัติ ถ้าไม่ได้ระบุ
เป็นอย่างอื่น หมายความว่าใครก็เรียกใช้ผ่าน object ได้จากภายนอก

```ruby
class Calculator
  def add(a, b)     # public โดยอัตโนมัติ
    a + b
  end
end

calc = Calculator.new
puts calc.add(3, 5)   # => 8 (เรียกจากภายนอกได้ปกติ)
```

### `private` — เรียกได้เฉพาะภายใน object เดียวกัน

Method ที่เป็น `private` **ไม่สามารถเรียกโดยระบุ receiver (explicit receiver) ได้เลย
แม้แต่ `self`** — เรียกได้แค่แบบ implicit จากภายใน method อื่นของ object เดียวกันเท่านั้น
มีประโยชน์สำหรับซ่อน implementation detail ที่ไม่ควรให้ภายนอกยุ่งเกี่ยวโดยตรง

```ruby
class OrderCalculator
  def total(price, quantity)
    subtotal = price * quantity
    subtotal + calculate_tax(subtotal)   # เรียก private method จากภายในได้ปกติ
  end

  private

  def calculate_tax(amount)
    amount * 0.07
  end
end

order = OrderCalculator.new
puts order.total(100, 2)          # => 214.0 (100*2 = 200, ภาษี 7% = 14, รวม 214)
order.calculate_tax(200)          # NoMethodError: private method 'calculate_tax' called
```

**อธิบาย:** ทุก method ที่ประกาศ **หลังคำว่า `private`** ในตัว class จะถูกเปลี่ยนเป็น
private ทั้งหมด (แบบมีผลไปจนจบ class หรือจนกว่าจะเจอ `public`/`protected` อีกครั้ง)
เหตุผลที่ `calculate_tax` ควรเป็น private เพราะเป็น "รายละเอียดการคำนวณภายใน" ที่ผู้ใช้
class ภายนอกไม่จำเป็นต้องรู้หรือเรียกตรงๆ — สิ่งที่ผู้ใช้ต้องรู้มีแค่ `total` เท่านั้น
หลักการนี้เรียกว่า **Encapsulation** ซึ่งเป็นหนึ่งในหลักการหลักของ OOP (จะเรียนลึกใน
Part 009)

### `protected` — เรียกได้ระหว่าง object ประเภทเดียวกัน

`protected` คล้าย `private` แต่ยืดหยุ่นกว่าเล็กน้อย: อนุญาตให้ object เรียก
protected method ของ **object อื่นที่เป็น class เดียวกัน (หรือ subclass)** ได้ ตราบใดที่
เรียกจากภายใน method ของ object ตัวเอง — มีประโยชน์เวลาต้อง "เทียบ" ค่าภายในระหว่าง
2 object

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
  end

  def richer_than?(other_account)
    balance > other_account.balance   # เรียก protected method ของ object อื่นได้
  end

  protected

  attr_reader :balance   # ทำให้ `balance` เป็น protected method
end

account1 = BankAccount.new(5000)
account2 = BankAccount.new(3000)

puts account1.richer_than?(account2)   # => true
account1.balance                        # NoMethodError: protected method 'balance' called
```

**สรุปความแตกต่างสั้นๆ:**

| Visibility | เรียกจากภายนอกได้ไหม | เรียกข้าม object (class เดียวกัน) ได้ไหม |
|---|---|---|
| `public` | ได้ | ได้ |
| `protected` | ไม่ได้ | ได้ |
| `private` | ไม่ได้ | ไม่ได้ (เรียกได้แค่แบบ implicit ใน object ตัวเอง) |

> **หมายเหตุ:** เรื่อง `class`, `initialize`, `attr_reader` ในตัวอย่างข้างบนจะเรียนเต็ม
> รูปแบบใน Part 009 — ในที่นี้แค่ต้องการแสดงให้เห็นว่า visibility ทำงานอย่างไรใน
> context จริง ไม่ต้องเข้าใจทุกรายละเอียดตอนนี้

### เกริ่นนำ `method_missing` (จะเรียนเต็มรูปแบบใน Part 015)

ปกติเมื่อเราเรียก method ที่ไม่มีอยู่จริงบน object ใดๆ Ruby จะโยน `NoMethodError` ทันที
แต่ Ruby มี hook พิเศษชื่อ `method_missing` ที่ให้เราดักจับการเรียก method ที่ไม่มีอยู่
แล้วกำหนดพฤติกรรมเองได้ — เป็นรากฐานของเทคนิค **metaprogramming** ขั้นสูง

```ruby
class GhostResponder
  def method_missing(method_name, *args, **kwargs)
    "คุณเรียก method ชื่อ '#{method_name}' ที่ไม่มีอยู่จริง พร้อม args: #{args}"
  end

  def respond_to_missing?(method_name, include_private = false)
    true   # บอกว่า object นี้ "ตอบสนอง" ทุก method (คู่กับ method_missing เสมอ)
  end
end

ghost = GhostResponder.new
puts ghost.fly_to_the_moon(3, 2, 1)
# => คุณเรียก method ชื่อ 'fly_to_the_moon' ที่ไม่มีอยู่จริง พร้อม args: [3, 2, 1]
```

`method_missing` คือกลไกเบื้องหลังของ Rails feature ที่ดูเหมือน "เวทมนตร์" หลายอย่าง
เช่น dynamic finder แบบเก่า (`find_by_name`), `method_missing`-based delegation ฯลฯ
เป็นเทคนิคที่ทรงพลังมากแต่ก็ต้องใช้อย่างระมัดระวัง (debug ยาก, performance cost) —
**Part 015 (Metaprogramming)** จะอธิบายกลไกนี้อย่างละเอียด รวมถึงทางเลือกที่ปลอดภัยกว่า
อย่าง `define_method` ในตอนนี้แค่รู้ว่ามันมีอยู่และหน้าตาประมาณนี้ก็เพียงพอแล้ว

---

## Step 70: แบบฝึกหัด — ระบบคำนวณยอดสั่งซื้อ (Order Total Calculator)

### โจทย์

สร้างไฟล์ `order_calculator.rb` ที่รวม method หลายตัวเข้าด้วยกันเพื่อคำนวณยอดสั่งซื้อของ
ร้านค้าออนไลน์เล็กๆ โดยต้องออกแบบ method ให้ครอบคลุมทุกรูปแบบ argument ที่เรียนมาใน
Part นี้:

1. `line_item_total(price, quantity = 1)` — คำนวณราคารวมของสินค้า 1 รายการ
   (positional + default argument)
2. `apply_discount(amount, percent: 0)` — หักส่วนลดเป็นเปอร์เซ็นต์ (keyword argument
   ที่มี default)
3. `calculate_shipping(*item_weights, base_fee: 30, per_kg: 15)` — คำนวณค่าส่งจาก
   น้ำหนักสินค้าหลายชิ้น (splat + keyword arguments)
4. `order_total(subtotal, discount_percent: 0, **fees)` — รวมยอดสุทธิ บวกค่าธรรมเนียม
   อื่นๆ ที่ระบุมาเป็น keyword เพิ่มเติมได้ไม่จำกัด (double splat)
5. `valid_order?(subtotal)` — ตรวจสอบว่ายอดสั่งซื้อถูกต้องหรือไม่ (predicate method,
   ต้องมากกว่า 0)
6. `finalize_order!(order)` — "ปิด" order โดยแก้ Hash ต้นฉบับให้มี key `:finalized`
   เป็น `true` (dangerous method ที่ mutate ค่าเดิม)

### เฉลย

```ruby
# frozen_string_literal: true

# order_calculator.rb
# ระบบคำนวณยอดสั่งซื้อสำหรับร้านค้าออนไลน์ขนาดเล็ก

# --- 1. positional + default argument ---
def line_item_total(price, quantity = 1)
  price * quantity
end

# --- 2. keyword argument พร้อม default ---
def apply_discount(amount, percent: 0)
  return amount if percent <= 0

  amount - (amount * percent / 100.0)
end

# --- 3. splat + keyword arguments ---
def calculate_shipping(*item_weights, base_fee: 30, per_kg: 15)
  total_weight = item_weights.sum
  base_fee + (total_weight * per_kg)
end

# --- 4. double splat: รับค่าธรรมเนียมเพิ่มเติมได้ไม่จำกัด ---
def order_total(subtotal, discount_percent: 0, **fees)
  after_discount = apply_discount(subtotal, percent: discount_percent)
  extra_fees_total = fees.values.sum
  after_discount + extra_fees_total
end

# --- 5. predicate method ---
def valid_order?(subtotal)
  subtotal.positive?
end

# --- 6. dangerous method ที่ mutate ต้นฉบับ ---
def finalize_order!(order)
  raise ArgumentError, "order ต้องเป็น Hash" unless order.is_a?(Hash)

  order[:finalized] = true
  order[:finalized_at] = Time.now
  order
end

# ============================================================
# ทดสอบการทำงาน
# ============================================================

puts "=== 1. line_item_total ==="
puts line_item_total(150, 3)     # => 450 (150 บาท x 3 ชิ้น)
puts line_item_total(200)         # => 200 (quantity default = 1)

puts "\n=== 2. apply_discount ==="
puts apply_discount(1000, percent: 10)   # => 900.0
puts apply_discount(1000)                 # => 1000 (ไม่มีส่วนลด)

puts "\n=== 3. calculate_shipping ==="
puts calculate_shipping(1.5, 2.0, 0.5)                          # base_fee=30, per_kg=15
# => 30 + (4.0 * 15) = 90.0
puts calculate_shipping(1.0, base_fee: 50, per_kg: 20)           # => 50 + 20 = 70.0
puts calculate_shipping()                                          # ไม่มีสินค้า => 30 + 0 = 30

puts "\n=== 4. order_total ==="
subtotal = line_item_total(150, 3) + line_item_total(200, 1)   # 450 + 200 = 650
total1 = order_total(subtotal, discount_percent: 10)
puts total1   # => (650 - 65) = 585.0

total2 = order_total(subtotal, discount_percent: 10, shipping: 50, wrapping_fee: 20)
puts total2   # => 585.0 + 50 + 20 = 655.0

puts "\n=== 5. valid_order? ==="
puts valid_order?(500)    # => true
puts valid_order?(0)      # => false
puts valid_order?(-10)    # => false

puts "\n=== 6. finalize_order! ==="
my_order = { customer: "มานี", subtotal: 650, finalized: false }
p my_order

finalize_order!(my_order)
p my_order
# => {customer: "มานี", subtotal: 650, finalized: true, finalized_at: <เวลาปัจจุบัน>}
# สังเกตว่า Hash ต้นฉบับ my_order ถูกแก้ไขโดยตรง เพราะ finalize_order! เป็น dangerous method
```

ทดสอบรัน:

```bash
ruby order_calculator.rb
```

**ผลลัพธ์ที่คาดหวัง (ประมาณ):**

```
=== 1. line_item_total ===
450
200

=== 2. apply_discount ===
900.0
1000

=== 3. calculate_shipping ===
90.0
70.0
30

=== 4. order_total ===
585.0
655.0

=== 5. valid_order? ===
true
false
false

=== 6. finalize_order! ===
{:customer=>"มานี", :subtotal=>650, :finalized=>false}
{:customer=>"มานี", :subtotal=>650, :finalized=>true, :finalized_at=>2026-09-26 ...}
```

### สิ่งที่ได้ฝึกจากเฉลยนี้

- ผสม positional default (`line_item_total`), keyword default (`apply_discount`),
  splat (`calculate_shipping`), และ double splat (`order_total`) ในโปรเจกต์เดียวกัน
- เห็นว่าทำไม `order_total` ที่ใช้ `**fees` ถึงยืดหยุ่นกว่า: เพิ่มค่าธรรมเนียมชนิดใหม่
  (`shipping:`, `wrapping_fee:` หรืออะไรก็ได้) โดยไม่ต้องแก้ signature ของ method เลย
- ใช้ guard clause (`return amount if percent <= 0`) เป็น explicit return ตามที่เรียน
  ใน Step 61
- แยกความรับผิดชอบของแต่ละ method ให้ทำหน้าที่เดียว (single responsibility) ซึ่งเป็น
  หลักการที่จะเจอซ้ำๆ ตลอดหลักสูตรนี้ โดยเฉพาะใน Part 082 (Service Object pattern)

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม method `bulk_discount_percent(quantity)` ที่คืนเปอร์เซ็นต์ส่วนลดอัตโนมัติตาม
   จำนวนที่ซื้อ: ซื้อ 1–4 ชิ้น ไม่มีส่วนลด, 5–9 ชิ้น ลด 5%, 10 ชิ้นขึ้นไป ลด 10%
   (ใช้ `case` ที่เรียนใน Part 006) แล้วนำไปต่อยอดกับ `apply_discount` ที่มีอยู่แล้ว
   โดยไม่ต้องแก้ signature ของ `apply_discount` เลย

2. เขียน method `summarize_order(**line_items)` ที่รับ keyword argument แบบไม่จำกัด
   โดยแต่ละ key คือชื่อสินค้า (Symbol) และ value คือราคา เช่น
   `summarize_order(coffee: 60, cake: 120, water: 15)` แล้วคืนค่าเป็น String สรุป
   ยอดรวมทั้งหมด พร้อมรายการสินค้าที่ราคาแพงที่สุด (ใบ้: ใช้ `.max_by` ที่เรียนใน Part 004)

3. เขียน method คู่ `?`/`!` ของตัวเอง: `expired?(order)` ตรวจสอบว่า order หมดอายุหรือยัง
   (สมมติว่า order เป็น Hash ที่มี key `:created_at` และถือว่าหมดอายุถ้าผ่านไปเกิน 7 วัน)
   และ `cancel!(order)` ที่ mutate order เดิมให้มี `:status => "cancelled"` — ทดลองเรียก
   `cancel!` กับ order ที่ `expired?` เป็น `true` แล้วให้โยน `RuntimeError` แทนที่จะยกเลิก
   เงียบๆ (ใบ้: `raise RuntimeError, "ข้อความ"` — จะเรียนเรื่อง exception handling อย่าง
   ละเอียดใน Part 011)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- นิยาม method ด้วย `def`/`end` และเข้าใจ **implicit return** (ค่าของบรรทัดสุดท้ายถูก
  return อัตโนมัติ) เทียบกับ explicit `return` ที่ใช้ตอนต้องการออกจาก method ก่อนเวลา
  (guard clause / early return)
- ใช้ **positional argument** ได้อย่างถูกต้อง และเข้าใจข้อจำกัดเรื่อง arity กับความเสี่ยง
  สลับลำดับผิดโดยไม่มี error เตือน
- ตั้งค่า **default argument value** ได้ ทั้งแบบ positional และ keyword พร้อมเข้าใจว่า
  default expression ถูก evaluate ใหม่ทุกครั้งที่เรียก (ไม่ใช่ครั้งเดียวตอนนิยาม)
- ใช้ **keyword arguments** ทั้งแบบ required และมี default และเข้าใจว่าทำไม Ruby/Rails
  community ถึงนิยมใช้ keyword arguments เป็นมาตรฐานสำหรับ method ที่มี argument
  หลายตัว
- ใช้ **splat (`*args`)** รวบรวม positional argument จำนวนไม่จำกัด และ **double splat
  (`**kwargs`)** รวบรวม keyword argument จำนวนไม่จำกัด ทั้งตอนนิยามและตอนเรียก (unpack)
- จำกฎลำดับการประกาศ parameter ที่ Ruby บังคับได้ เมื่อผสม argument หลายแบบเข้าด้วยกัน
- เข้าใจธรรมเนียมตั้งชื่อ method ด้วย `?` (predicate) และ `!` (dangerous/mutating) และ
  เขียน method ที่ยึดธรรมเนียมนี้ได้เอง
- รู้จัก method visibility `public`/`private`/`protected` ในระดับพื้นฐาน และได้เห็น
  ตัวอย่าง `method_missing` เป็นการเกริ่นนำก่อนเรียนลึกใน Part 015
- ลงมือเขียนระบบคำนวณยอดสั่งซื้อที่ผสมผสาน argument ทุกรูปแบบเข้าด้วยกันในโปรเจกต์เดียว

**ต่อไป (Part 008):** เราจะเจาะลึกเรื่อง **Blocks, `yield`, `Proc`, และ `Lambda`** —
กลไกที่ทำให้ method รับ "ก้อนโค้ด" เป็น argument ได้ (ซึ่งเป็นรากฐานของ method อย่าง
`each`, `map`, `select` ที่ใช้มาตั้งแต่ Part 004) และเป็นพื้นฐานสำคัญก่อนจะไปเรียน
metaprogramming และการเขียน DSL (Domain Specific Language) แบบที่ Rails ใช้ทั่วทั้ง
framework
