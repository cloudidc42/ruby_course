# Part 003: String methods, String interpolation, Heredoc, และ Regular Expression เบื้องต้น

> **Step ครอบคลุมใน Part นี้:** Step 21–30
> **ระดับ:** เริ่มต้น–ปานกลาง (ต้องผ่าน Part 001–002 มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

String คือชนิดข้อมูลที่เราจะเจอบ่อยที่สุดในการเขียนโปรแกรมจริง ไม่ว่าจะเป็นการรับ input
จากผู้ใช้ การแสดงผล การประมวลผลข้อความจากไฟล์หรือ API หรือแม้แต่การสร้าง SQL query
ใน Rails เอง (แม้ Rails จะมี ORM ช่วยแล้ว แต่พื้นฐาน String ยังจำเป็นมากในการทำ data
processing, validation, และ text formatting) Part นี้จะพาไปรู้จัก String ของ Ruby อย่าง
ละเอียด ตั้งแต่การสร้าง, method ที่ใช้บ่อยที่สุด, การต่อ string อย่างถูกวิธี, Heredoc
สำหรับข้อความหลายบรรทัด และปิดท้ายด้วยพื้นฐาน Regular Expression ที่จำเป็นสำหรับงาน
validation และ text processing ในอนาคต

## สารบัญของ Part นี้

- Step 21: การสร้าง String — single quote vs double quote และ escape sequence
- Step 22: String methods พื้นฐาน: length, upcase/downcase, capitalize, strip
- Step 23: ค้นหาและแทนที่: include?, start_with?/end_with?, split, gsub/sub
- Step 24: เจาะลึก index และการเข้าถึงตัวอักษร: slice/[], reverse, chars, each_char
- Step 25: String interpolation เชิงลึก — `#{}` ใส่ expression อะไรก็ได้
- Step 26: การต่อ String: `+` vs `<<` vs `concat` และเรื่อง mutability
- Step 27: frozen_string_literal และ String ที่ immutable
- Step 28: Heredoc — ข้อความหลายบรรทัดแบบมืออาชีพ (`<<~`, `<<-`, `<<`)
- Step 29: Regular Expression เบื้องต้น — Regexp literal, match?, match, =~
- Step 30: แบบฝึกหัดโปรเจกต์ — Mini Template Engine และ Text Report Generator

---

## Step 21: การสร้าง String — single quote vs double quote และ escape sequence

Ruby มีสองวิธีหลักในการสร้าง String literal คือใช้ single quote (`'...'`) และ double quote
(`"..."`) ซึ่งทำงานต่างกันในรายละเอียดสำคัญ

```ruby
# frozen_string_literal: true

name = "Ruby"

single = 'สวัสดี #{name}'   # #{} ไม่ทำงานใน single quote
double = "สวัสดี #{name}"   # #{} ทำงานใน double quote

puts single  # => สวัสดี #{name}
puts double  # => สวัสดี Ruby
```

**ความแตกต่างหลักระหว่าง single quote และ double quote:**

1. **String interpolation** (`#{}`) ทำงานเฉพาะใน double quote เท่านั้น
2. **Escape sequence** (เช่น `\n`, `\t`) ถูกตีความเฉพาะใน double quote เท่านั้น
   single quote จะเก็บ backslash ไว้ตรงๆ (ยกเว้น `\\` และ `\'` ที่ยังทำงานได้)

```ruby
puts "บรรทัดที่ 1\nบรรทัดที่ 2"
# บรรทัดที่ 1
# บรรทัดที่ 2

puts 'บรรทัดที่ 1\nบรรทัดที่ 2'
# บรรทัดที่ 1\nบรรทัดที่ 2   (ไม่ตีความ \n ให้)
```

### Escape sequence ที่ใช้บ่อย

```ruby
puts "Tab:\tจบ Tab"          # \t = tab character
puts "บรรทัดใหม่\nบรรทัดถัดไป"  # \n = newline
puts "เครื่องหมาย \"คำพูด\" ใน string"  # \" = double quote ตัวจริง
puts "แบ็กสแลชตัวจริง: \\"        # \\ = backslash ตัวจริง
puts "รหัส Unicode: \u0E44\u0E17"  # \u = Unicode code point (ไทย)
```

### เมื่อไหร่ควรใช้ single quote เมื่อไหร่ควรใช้ double quote

แนวปฏิบัติที่นิยมในวงการ Ruby (และเป็นกฎ default ของ RuboCop คือ `Style/StringLiterals`):

- ใช้ **single quote** เป็นค่าเริ่มต้น เมื่อ string นั้นไม่มี interpolation และไม่มี
  escape sequence ที่ต้องใช้ เพราะอ่านง่ายกว่าเล็กน้อยและสื่อเจตนาว่า "นี่คือ string ตรงๆ
  ไม่มีอะไรซ่อนอยู่"
- ใช้ **double quote** เมื่อจำเป็นต้องใช้ interpolation หรือ escape sequence

```ruby
# แนวปฏิบัติที่ดี
title = 'Ruby on Rails'              # ไม่มี interpolation ใช้ single quote
greeting = "สวัสดี, #{title}!"        # มี interpolation ใช้ double quote
path = 'C:\Users\name'               # single quote ดีกว่าเพราะไม่ต้อง escape \
```

> **หมายเหตุ:** ประสิทธิภาพของ single quote กับ double quote ที่ไม่มี interpolation
> แทบไม่ต่างกันเลยใน Ruby ยุคปัจจุบัน การเลือกใช้จึงเป็นเรื่อง **code style** และ
> **ความชัดเจนของเจตนา** มากกว่าเรื่อง performance

### `%q` และ `%Q` — ทางเลือกเมื่อ string มีเครื่องหมายคำพูดเยอะ

```ruby
# %q เทียบเท่า single quote, %Q เทียบเท่า double quote
message = %q(เขาพูดว่า "สวัสดี" กับฉัน)
message2 = %Q(ชื่อของฉันคือ #{name})

puts message
puts message2
```

`%q`/`%Q` มีประโยชน์เมื่อ string มีทั้ง `'` และ `"` ปนกันเยอะ เพราะเลือก delimiter อื่น
(เช่น `()`, `{}`, `[]`, `||`) แทนได้ ไม่ต้อง escape ให้วุ่นวาย

---

## Step 22: String methods พื้นฐาน: length, upcase/downcase, capitalize, strip

Ruby มี method ให้ String มาให้ใช้งานเยอะมาก (ลองรัน `"abc".methods.count` ดูจะพบว่ามี
มากกว่า 100 method) ใน Step นี้จะเน้นที่ใช้บ่อยที่สุดในงานจริง

### วัดความยาว

```ruby
s = "Ruby on Rails"

s.length  # => 13
s.size    # => 13 (เป็น alias ของ length ผลลัพธ์เหมือนกันทุกประการ)
s.empty?  # => false

"".empty?     # => true
"   ".empty?  # => false (มี whitespace อยู่ ไม่ใช่ string ว่างเปล่า)
```

### เปลี่ยนตัวพิมพ์เล็ก/ใหญ่

```ruby
s = "Ruby on Rails"

s.upcase       # => "RUBY ON RAILS"
s.downcase     # => "ruby on rails"
s.capitalize   # => "Ruby on rails"   (ตัวแรกใหญ่ ตัวที่เหลือเล็กหมด)
s.swapcase     # => "rUBY ON rAILS"   (สลับตัวใหญ่เป็นเล็ก เล็กเป็นใหญ่)

# ทุก method ข้างต้นคืนค่า string ใหม่ ไม่แก้ตัวต้นฉบับ
puts s  # => "Ruby on Rails" (ยังเหมือนเดิม)
```

**สังเกตข้อสำคัญ:** `upcase`, `downcase`, `capitalize` ทั้งหมดนี้ **ไม่แก้ไข string ต้นฉบับ**
แต่คืนค่า string ใหม่ออกมา ถ้าต้องการแก้ไขต้นฉบับโดยตรง (mutate in place) ต้องใช้ version
ที่มี `!` ต่อท้าย:

```ruby
s = "Ruby on Rails"
s.upcase!   # แก้ s ให้เป็น "RUBY ON RAILS" ทันที
puts s      # => "RUBY ON RAILS"
```

> **คำเตือนเรื่อง `!` :** ธรรมเนียมของ Ruby คือ method ที่ลงท้ายด้วย `!` มักจะ "อันตราย"
> กว่า version ปกติ (mutate ค่าเดิม หรือ raise error แทนที่จะคืนค่า nil) ไม่ใช่กฎตายตัว
> 100% แต่เป็นสัญญาณเตือนที่ควรระวังเสมอเวลาเห็น `!` ต่อท้าย method — เราจะพูดเรื่องนี้
> ละเอียดอีกครั้งใน Step 26 เรื่อง mutability

### ตัดช่องว่าง (whitespace)

```ruby
s = "   Hello World   "

s.strip    # => "Hello World"       (ตัดทั้งซ้ายและขวา)
s.lstrip   # => "Hello World   "    (ตัดเฉพาะซ้าย)
s.rstrip   # => "   Hello World"    (ตัดเฉพาะขวา)

# ใช้บ่อยมากตอนรับ input จากผู้ใช้
name = gets.chomp.strip
```

`strip` เป็น method ที่ควรใช้แทบทุกครั้งเมื่อรับ input จากผู้ใช้ เพราะผู้ใช้มักพิมพ์
ช่องว่างเกินมาโดยไม่ตั้งใจ (เช่น พิมพ์ " สมชาย " แทนที่จะเป็น "สมชาย")

### ตัวอย่างการใช้งานร่วมกัน

```ruby
# frozen_string_literal: true

raw_input = "   john.doe@Example.COM   "

email = raw_input.strip.downcase
puts email  # => "john.doe@example.com"
```

ตัวอย่างนี้แสดงให้เห็นว่า Ruby รองรับ **method chaining** ได้อย่างเป็นธรรมชาติ —
เพราะ `strip` คืนค่า String ใหม่ออกมา จึงเรียก `.downcase` ต่อได้ทันทีโดยไม่ต้องเก็บ
ตัวแปรกลาง ทำให้โค้ดกระชับและอ่านเป็นลำดับขั้นตอนได้ง่าย

---

## Step 23: ค้นหาและแทนที่: include?, start_with?/end_with?, split, gsub/sub

### ตรวจสอบว่ามีข้อความอยู่หรือไม่

```ruby
email = "john.doe@example.com"

email.include?("@")           # => true
email.include?("example")     # => true
email.include?("gmail")       # => false

email.start_with?("john")     # => true
email.end_with?(".com")       # => true
email.end_with?(".com", ".net", ".org")  # => true (เช็คได้หลายตัวพร้อมกัน)
```

Method เหล่านี้คืนค่าเป็น `true`/`false` เสมอ (สังเกตว่าชื่อ method ลงท้ายด้วย `?`
ซึ่งเป็นธรรมเนียมของ Ruby สำหรับ method ที่คืนค่า boolean) จึงเหมาะมากสำหรับใช้ในเงื่อนไข
`if`

```ruby
if email.include?("@")
  puts "รูปแบบอีเมลดูสมเหตุสมผล"
else
  puts "นี่ไม่ใช่อีเมล"
end
```

### แบ่ง String ด้วย split

```ruby
csv_line = "สมชาย,25,กรุงเทพ"
fields = csv_line.split(",")
p fields  # => ["สมชาย", "25", "กรุงเทพ"]

name, age, city = fields
puts "#{name} อายุ #{age} ปี อยู่ที่ #{city}"

# split ไม่ใส่ argument = แบ่งตาม whitespace โดยอัตโนมัติ (รวมช่องว่างซ้ำ, tab, newline)
sentence = "Ruby   on\tRails"
words = sentence.split
p words  # => ["Ruby", "on", "Rails"]

# จำกัดจำนวนส่วนที่แบ่งด้วย argument ตัวที่สอง
"a-b-c-d".split("-", 2)  # => ["a", "b-c-d"]
```

`split` เป็น method ที่ใช้บ่อยมากในการแปลง string เดี่ยวๆ ให้เป็น Array เพื่อประมวลผล
ต่อ (เราจะเรียนเรื่อง Array แบบเต็มใน Part 004)

### แทนที่ข้อความด้วย gsub และ sub

```ruby
text = "I love Java. Java is great."

# sub แทนที่แค่ตัวแรกที่เจอ
text.sub("Java", "Ruby")   # => "I love Ruby. Java is great."

# gsub (global substitution) แทนที่ทุกตัวที่เจอ
text.gsub("Java", "Ruby")  # => "I love Ruby. Ruby is great."

# ทั้ง sub และ gsub ไม่แก้ text ต้นฉบับ (ต้องใช้ sub!/gsub! ถ้าต้องการ mutate)
puts text  # => "I love Java. Java is great." (ยังเหมือนเดิม)
```

```ruby
# gsub รับ Hash เพื่อแทนที่หลายค่าพร้อมกันได้
template = "สวัสดี {name}, วันนี้อากาศ {weather}"
result = template.gsub(/\{(\w+)\}/, "{name}" => "มานี", "{weather}" => "ร้อนมาก")
puts result  # => สวัสดี มานี, วันนี้อากาศ ร้อนมาก
```

ตัวอย่างสุดท้ายนี้แอบใช้ Regular Expression ไปแล้ว (`/\{(\w+)\}/`) — เราจะอธิบาย syntax
นี้อย่างละเอียดใน Step 29–30 แต่ใส่ให้เห็นตั้งแต่ตอนนี้ว่า `gsub` ทำงานร่วมกับ Regexp
ได้อย่างทรงพลัง ไม่ใช่แค่แทนที่ string ตรงๆ

---

## Step 24: เจาะลึก index และการเข้าถึงตัวอักษร: slice/[], reverse, chars, each_char

### เข้าถึงตัวอักษรหรือช่วงของ String ด้วย `[]`

Ruby นับ index ของ String เริ่มจาก 0 (เหมือนภาษาโปรแกรมมิ่งส่วนใหญ่) และรองรับ index
ติดลบเพื่อนับจากท้าย string ด้วย

```ruby
s = "Ruby on Rails"

s[0]      # => "R"        (ตัวอักษรที่ index 0)
s[-1]     # => "s"        (ตัวอักษรสุดท้าย)
s[0, 4]   # => "Ruby"     (เริ่มที่ index 0 ยาว 4 ตัว)
s[0..3]   # => "Ruby"     (ช่วง index 0 ถึง 3 แบบรวมปลาย)
s[0...4]  # => "Ruby"     (ช่วง index 0 ถึง 3 แบบไม่รวมปลาย — เหมือนผลลัพธ์ด้านบน)
s[-5..]   # => "Rails"    (จาก index -5 ถึงจบ string)
s[100]    # => nil        (index เกินความยาว string ได้ nil ไม่ error)
```

`slice` เป็น method ชื่อเต็มของ `[]` ทำงานเหมือนกันทุกประการ — `s.slice(0, 4)` ให้ผลลัพธ์
เดียวกับ `s[0, 4]`

```ruby
s = "Ruby on Rails"
s.slice(0, 4)   # => "Ruby"
s.slice(0..1)   # => "Ru"
```

### หาตำแหน่งของ substring

```ruby
s = "Ruby on Rails"
s.index("on")     # => 5   (ตำแหน่งที่พบครั้งแรก, นับจาก 0)
s.index("Python")  # => nil  (ไม่พบ)
s.rindex("a")      # => 11  (ค้นหาจากท้าย string, ตำแหน่งที่พบครั้งสุดท้าย)
```

### กลับด้าน (reverse)

```ruby
"Ruby".reverse       # => "ybuR"
"ยินดี".reverse       # => "ดีนยิ"

# ตัวอย่างการใช้งานจริง: ตรวจสอบ palindrome (คำที่อ่านหน้าหลังเหมือนกัน)
def palindrome?(word)
  cleaned = word.downcase.gsub(/[^a-z]/, "")
  cleaned == cleaned.reverse
end

palindrome?("racecar")        # => true
palindrome?("A man a plan a canal Panama")  # => true
palindrome?("Ruby")           # => false
```

### แปลง String เป็นตัวอักษรทีละตัวด้วย chars และ each_char

```ruby
s = "Ruby"

s.chars  # => ["R", "u", "b", "y"]   (คืนค่าเป็น Array ของตัวอักษร)

s.each_char do |char|
  puts "ตัวอักษร: #{char}"
end
# ตัวอักษร: R
# ตัวอักษร: u
# ตัวอักษร: b
# ตัวอักษร: y
```

**ความแตกต่างระหว่าง `chars` กับ `each_char`:**

- `chars` คืนค่า Array ทั้งหมดออกมาทันที (ใช้ memory เก็บทุกตัวพร้อมกัน) เหมาะเมื่อต้องการ
  นำ Array นั้นไปทำงานต่อ เช่น `.count`, `.uniq`, `.sort`
- `each_char` เป็น iterator ที่ประมวลผลทีละตัวอักษร (ไม่สร้าง Array ขึ้นมาก่อน) เหมาะเมื่อ
  แค่ต้องการวนลูปทำอะไรกับแต่ละตัวอักษรโดยไม่ต้องเก็บผลลัพธ์เป็น Array

```ruby
# ตัวอย่าง: นับจำนวนสระในคำ
def count_vowels(word)
  vowels = "aeiouAEIOU"
  count = 0
  word.each_char do |char|
    count += 1 if vowels.include?(char)
  end
  count
end

count_vowels("Ruby on Rails")  # => 4
```

### method อื่นๆ ที่ควรรู้จักไว้

```ruby
"5".ljust(10, "0")   # => "5000000000"  (เติมด้านขวาให้ครบความยาวที่กำหนด)
"5".rjust(10, "0")   # => "0000000005"  (เติมด้านซ้าย — ใช้บ่อยตอนจัด format ตัวเลข)
"5".rjust(3, "0")    # => "005"

"a" * 5              # => "aaaaa"  (String repetition — เคยเห็นแล้วใน Part 001)

"Hello".center(11, "*")  # => "***Hello***"
```

---

## Step 25: String interpolation เชิงลึก — `#{}` ใส่ expression อะไรก็ได้

Part 001 และ 002 ได้ใช้ `#{}` มาบ้างแล้ว แต่สิ่งสำคัญที่หลายคนพลาดคือ: **ภายใน `#{}`
สามารถใส่ Ruby expression อะไรก็ได้ ไม่ใช่แค่ชื่อตัวแปรตัวเดียว**

```ruby
name = "Ruby"
version = 3.3

# ใส่ตัวแปรตรงๆ
puts "ภาษา: #{name}"

# ใส่การคำนวณ
puts "ปีหน้า: #{2026}"
puts "ผลรวม: #{1 + 2 + 3}"

# ใส่ method call
puts "ชื่อตัวใหญ่: #{name.upcase}"

# ใส่เงื่อนไข (ternary operator หรือแม้แต่ if/else แบบเต็ม)
score = 85
puts "ผลสอบ: #{score >= 50 ? "ผ่าน" : "ไม่ผ่าน"}"

# ใส่ method call ที่ซับซ้อนหลายขั้น
numbers = [1, 2, 3, 4, 5]
puts "ผลรวมของเลขคู่: #{numbers.select { |n| n.even? }.sum}"

# ใส่ if/else แบบหลายบรรทัดก็ได้ (แม้จะไม่นิยมเพราะอ่านยาก)
puts "สถานะ: #{
  if score >= 80
    "ดีเยี่ยม"
  elsif score >= 50
    "ผ่าน"
  else
    "ต้องปรับปรุง"
  end
}"
```

**หลักการทำงานภายใน:** เมื่อ Ruby เจอ `#{expression}` ใน double-quoted string มันจะ
ประเมินผล (evaluate) expression นั้นก่อน แล้วเรียก `.to_s` กับผลลัพธ์เพื่อแปลงเป็น String
โดยอัตโนมัติ ก่อนจะแทรกเข้าไปในตำแหน่งนั้น

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end

  def to_s
    "(#{@x}, #{@y})"
  end
end

point = Point.new(3, 4)
puts "ตำแหน่งคือ #{point}"  # => ตำแหน่งคือ (3, 4)
# Ruby เรียก point.to_s ให้อัตโนมัติเพื่อแปลงเป็น string ก่อนแทรกเข้าไป
```

> **แนวคิดสำคัญ:** เพราะ `#{}` เรียก `to_s` ให้อัตโนมัติเสมอ การ define method `to_s`
> ใน class ของเราเอง (ดังตัวอย่าง `Point`) จึงเป็นเทคนิคมาตรฐานที่ทำให้ object ของเรา
> แสดงผลสวยงามเมื่อถูกแทรกใน string หรือถูก `puts` โดยตรง — เราจะเรียนเรื่อง class
> และ method นี้อย่างละเอียดใน Part 009

### ข้อควรระวัง: interpolation vs concatenation

```ruby
age = 25

# ทำงานได้ปกติ เพราะ #{} แปลง Integer เป็น String ให้อัตโนมัติ
puts "อายุ #{age} ปี"

# แต่ถ้าใช้ + ต่อ string กับ Integer ตรงๆ จะ error ทันที
# puts "อายุ " + age + " ปี"
# TypeError (no implicit conversion of Integer into String)

# ต้องแปลงเองด้วย .to_s ก่อน ถ้าจะใช้ +
puts "อายุ " + age.to_s + " ปี"
```

นี่คือเหตุผลหนึ่งที่โปรแกรมเมอร์ Ruby นิยมใช้ **interpolation มากกว่า concatenation**
สำหรับการประกอบ string ที่มีค่าจากตัวแปรหลายชนิดปนกัน เพราะปลอดภัยและอ่านง่ายกว่า

---

## Step 26: การต่อ String: `+` vs `<<` vs `concat` และเรื่อง mutability

Ruby มีหลายวิธีในการต่อ (concatenate) string เข้าด้วยกัน แต่ละวิธีมีพฤติกรรมเรื่อง
**mutability** (การแก้ไข object เดิมหรือสร้าง object ใหม่) ที่ต่างกัน ซึ่งสำคัญมากที่
ต้องเข้าใจให้ถูกต้อง เพราะเป็นสาเหตุของบั๊กที่พบบ่อยในโค้ดจริง

### `+` — สร้าง String ใหม่เสมอ

```ruby
a = "Hello"
b = " World"

c = a + b
puts c        # => "Hello World"
puts a        # => "Hello"        (a ไม่ถูกแก้ไข)

puts a.object_id  # เช่น 60
puts c.object_id  # เช่น 80 (คนละ object กับ a)
```

`+` นำ string สองตัวมาสร้าง string **ตัวใหม่** ขึ้นมา โดยไม่แตะต้อง string ต้นฉบับเลย
เหมาะเมื่อเราไม่ต้องการให้ string เดิมถูกแก้ไข

### `<<` (shovel operator) และ `concat` — แก้ไข String เดิม (mutate)

```ruby
a = "Hello"
a << " World"     # แก้ a ให้กลายเป็น "Hello World" ทันที ไม่สร้าง object ใหม่
puts a            # => "Hello World"

b = "Hello"
original_id = b.object_id
b.concat(" World")
puts b.object_id == original_id  # => true (ยังเป็น object เดิม แค่เนื้อหาถูกแก้)
```

`<<` และ `concat` **mutate string เดิม** (แก้ไขเนื้อหาของ object เดิมโดยตรง) ไม่ได้
สร้าง object ใหม่ขึ้นมา ต่างจาก `+` อย่างชัดเจน

### ทำไมความแตกต่างนี้ถึงสำคัญ

```ruby
# ตัวอย่างบั๊กที่เกิดจากไม่เข้าใจ mutability
def build_greeting(name)
  greeting = "Hello, "
  greeting << name
  greeting
end

original = "Hello, "
result = build_greeting("Ruby")
puts result    # => "Hello, Ruby"

# ถ้ามีตัวแปรอื่นชี้ไปที่ object เดียวกัน การ << จะกระทบตัวแปรนั้นด้วย!
shared = "count: "
label = shared          # label ชี้ไปที่ object เดียวกับ shared (ไม่ได้ copy)
label << "5"
puts shared             # => "count: 5"  !! shared ก็เปลี่ยนไปด้วย ทั้งที่เราแก้ label
```

ตัวอย่างสุดท้ายคือกับดักคลาสสิกของโปรแกรมเมอร์มือใหม่: การกำหนดค่า (`label = shared`)
ไม่ได้ copy string แต่เป็นการให้ตัวแปรสองตัวชี้ไปยัง **object เดียวกันในหน่วยความจำ**
เมื่อใช้ method ที่ mutate (เช่น `<<`) กับตัวแปรตัวใดตัวหนึ่ง จะกระทบตัวแปรอีกตัวด้วย
เพราะทั้งคู่ชี้ไปที่ object เดียวกัน

```ruby
# วิธีป้องกัน: ใช้ .dup เพื่อสร้าง copy ใหม่ก่อน mutate
shared = "count: "
label = shared.dup     # dup สร้าง object ใหม่ที่มีเนื้อหาเหมือนกัน
label << "5"
puts shared             # => "count: "   (ไม่กระทบแล้ว)
puts label              # => "count: 5"
```

### ประสิทธิภาพ: `<<` เร็วกว่า `+` เมื่อต่อ string จำนวนมากในลูป

```ruby
# วิธีที่ไม่ควรทำเมื่อต่อ string จำนวนมาก (สร้าง object ใหม่ทุกรอบ วิ่งช้าลงเรื่อยๆ)
result = ""
1000.times { |i| result = result + i.to_s }

# วิธีที่ดีกว่า (แก้ไข object เดิม ไม่ต้องสร้างใหม่ทุกรอบ เร็วกว่ามาก)
result = +""   # unary + บังคับให้เป็น mutable string (อธิบายใน Step 27)
1000.times { |i| result << i.to_s }
```

เหตุผลคือทุกครั้งที่ใช้ `+` ในลูป Ruby ต้องจัดสรร memory ก้อนใหม่และ copy ข้อมูลเดิม
ทั้งหมดไปยังตำแหน่งใหม่ ยิ่ง string ยาวขึ้นเรื่อยๆ การ copy ก็ยิ่งช้าลงเรื่อยๆ
(ความซับซ้อนแบบ O(n²) โดยรวม) ในขณะที่ `<<` แก้ไข buffer เดิมในหน่วยความจำโดยตรง
(ความซับซ้อนแบบ O(n) โดยรวม โดยเฉลี่ย)

---

## Step 27: frozen_string_literal และ String ที่ immutable

Part 001 (Step 9) ได้แนะนำ `# frozen_string_literal: true` ไปแล้วแบบผิวเผิน มา
ทำความเข้าใจอย่างละเอียดใน Step นี้ว่ามันเกี่ยวข้องกับเรื่อง mutability จาก Step 26
อย่างไร

### ปกติแล้ว String ใน Ruby เป็น mutable (แก้ไขได้)

```ruby
s = "Hello"
s << " World"   # ทำงานได้ปกติ เพราะ String ปกติแก้ไขได้ (mutable)
puts s          # => "Hello World"
```

### เมื่อเปิด `frozen_string_literal: true`

```ruby
# frozen_string_literal: true

s = "Hello"
s.frozen?       # => true (string literal ทุกตัวในไฟล์นี้ถูก freeze อัตโนมัติ)

s << " World"
# FrozenError (can't modify frozen String: "Hello")
```

เมื่อเปิด magic comment นี้ที่หัวไฟล์ **String literal ทุกตัวที่เขียนตรงๆ ในไฟล์นั้น**
(เช่น `"Hello"`) จะถูกทำให้เป็น **frozen** (แช่แข็ง) โดยอัตโนมัติ หมายความว่าไม่สามารถ
เรียก method ที่ mutate (`<<`, `concat`, `gsub!`, `upcase!` ฯลฯ) กับมันได้อีก ถ้าพยายาม
เรียกจะได้ `FrozenError` ทันที

### ทำไมต้องใช้ frozen_string_literal

1. **ประสิทธิภาพ:** ถ้าโค้ดมี string literal ค่าเดียวกันซ้ำๆ (เช่น `"active"` ปรากฏใน
   โค้ด 50 จุด) เมื่อ frozen แล้ว Ruby จะ reuse object เดียวกันแทนที่จะสร้าง object
   ใหม่ทุกครั้งที่ interpreter อ่านผ่าน literal นั้น ลด memory allocation
2. **ป้องกันบั๊ก:** บังคับให้โปรแกรมเมอร์ตั้งใจ (explicit) เมื่อต้องการ string ที่
   แก้ไขได้ — ถ้าพยายาม mutate string ที่ไม่ควรถูกแก้ (เช่น constant, ค่าคงที่)
   โปรแกรมจะ error ทันทีแทนที่จะเกิดบั๊กเงียบๆ ในภายหลัง

### เมื่อจำเป็นต้องมี mutable string ในไฟล์ที่ frozen ทั้งไฟล์

```ruby
# frozen_string_literal: true

# ใช้ String.new เพื่อสร้าง string ที่ไม่ frozen แม้ทั้งไฟล์จะ frozen
buffer = String.new
buffer << "Hello"
buffer << " World"
puts buffer  # => "Hello World"

# หรือใช้ unary + หน้า string literal (Ruby จะคืน copy ที่ไม่ frozen)
s = +"Hello"
s << " World"
puts s  # => "Hello World"

# unary - ทำตรงข้าม คือบังคับ freeze (ใช้ยืนยันเจตนาว่าต้องการ frozen string)
readonly = -"Hello"
readonly.frozen?  # => true
```

### ตรวจสอบสถานะ frozen

```ruby
"abc".frozen?         # => true ถ้าไฟล์เปิด frozen_string_literal, false ถ้าไม่เปิด
"abc".dup.frozen?     # => false เสมอ (.dup คืน copy ที่ไม่ frozen)
:symbol.frozen?       # => true เสมอ (Symbol เป็น immutable โดยธรรมชาติอยู่แล้ว)
1.frozen?             # => true เสมอ (Integer/Float ก็ immutable โดยธรรมชาติ)
```

> **แนวปฏิบัติมาตรฐาน:** ตั้งแต่ Part นี้เป็นต้นไป ทุกไฟล์ Ruby ที่เขียนในหลักสูตรนี้
> จะใส่ `# frozen_string_literal: true` ไว้บรรทัดแรกเสมอ ตามธรรมเนียมของโค้ด Ruby
> ระดับมืออาชีพในปัจจุบัน (RuboCop ก็บังคับกฎนี้เป็นค่าเริ่มต้น)

---

## Step 28: Heredoc — ข้อความหลายบรรทัดแบบมืออาชีพ (`<<~`, `<<-`, `<<`)

เมื่อต้องเขียนข้อความยาวหลายบรรทัด (เช่น email template, SQL query, ข้อความช่วยเหลือ
ของโปรแกรม) การต่อ string ด้วย `+` หรือใส่ `\n` เองทีละจุดจะอ่านยากมาก **Heredoc**
คือ syntax พิเศษของ Ruby สำหรับเขียน string หลายบรรทัดให้อ่านง่าย

### Heredoc แบบพื้นฐาน (`<<`)

```ruby
message = <<TEXT
บรรทัดที่หนึ่ง
บรรทัดที่สอง
บรรทัดที่สาม
TEXT

puts message
# บรรทัดที่หนึ่ง
# บรรทัดที่สอง
# บรรทัดที่สาม
```

**โครงสร้าง:** `<<TEXT` เริ่มต้น heredoc โดย `TEXT` คือ "identifier" ที่เราตั้งชื่อเอง
(นิยมใช้ตัวพิมพ์ใหญ่ทั้งหมด เช่น `TEXT`, `EOS`, `SQL`, `HTML`) ทุกอย่างตั้งแต่บรรทัด
ถัดไปจนถึงบรรทัดที่มี `TEXT` ตัวเดียวโดดๆ (ต้องอยู่ต้นบรรทัดพอดี) จะถือเป็นเนื้อหา
ของ string นั้น

### ปัญหาของ Heredoc แบบพื้นฐานเมื่ออยู่ใน indented code

```ruby
def build_message
  text = <<TEXT
สวัสดี
ลาก่อน
TEXT
  text
end
```

โค้ดข้างบนนี้จะ **error** เพราะ closing identifier (`TEXT`) ต้องอยู่ชิดซ้ายสุด (column 0)
เท่านั้นสำหรับ heredoc แบบพื้นฐาน แต่ในโค้ดจริงเรามักเขียนโค้ดแบบ indent (เยื้องเข้าไป
ตาม block/method) ทำให้ heredoc แบบพื้นฐานใช้ยากมากในทางปฏิบัติ

### `<<-` — อนุญาตให้ indent closing identifier ได้ (แต่เนื้อหายังไม่ indent)

```ruby
def build_message
  text = <<-TEXT
    สวัสดี
    ลาก่อน
  TEXT
  text
end

puts build_message
#     สวัสดี
#     ลาก่อน
```

`<<-` แก้ปัญหาให้ **closing identifier** เยื้องได้ แต่สังเกตว่า **เนื้อหาข้างในยังคง
เยื้องตามที่พิมพ์จริง** (มี whitespace นำหน้าติดมาด้วย) ซึ่งมักไม่ใช่สิ่งที่ต้องการ

### `<<~` (squiggly heredoc) — ตัวเลือกที่ดีที่สุด (Ruby >= 2.3)

```ruby
def build_message
  text = <<~TEXT
    สวัสดี
    ลาก่อน
  TEXT
  text
end

puts build_message
# สวัสดี
# ลาก่อน
```

`<<~` (เรียกว่า **squiggly heredoc**) ทำงานเหมือน `<<-` ตรงที่ closing identifier
เยื้องได้ แต่มีความสามารถเพิ่มคือ **ตัด indentation ส่วนเกินออกจากทุกบรรทัดโดยอัตโนมัติ**
โดยยึดจากบรรทัดที่เยื้องน้อยที่สุดเป็นฐาน ทำให้ผลลัพธ์ออกมาสวยงามโดยไม่ต้องกังวลเรื่อง
whitespace ที่ติดมา

> **แนวปฏิบัติมาตรฐาน:** ในโค้ด Ruby ยุคปัจจุบัน (Ruby 2.3 ขึ้นไป) ควรใช้ `<<~` เป็น
> ค่าเริ่มต้นเสมอเมื่อต้องการ heredoc ที่อยู่ใน indented context แทบไม่มีเหตุผลต้องใช้
> `<<-` หรือ `<<` แบบพื้นฐานอีกแล้วในโค้ดใหม่

### Heredoc รองรับ interpolation เหมือน double-quoted string ปกติ

```ruby
name = "มานี"
order_total = 1500

receipt = <<~RECEIPT
  ใบเสร็จรับเงิน
  ----------------------------
  ลูกค้า: #{name}
  ยอดรวม: #{order_total} บาท
  ขอบคุณที่ใช้บริการ
RECEIPT

puts receipt
```

ถ้าต้องการ heredoc ที่ **ไม่** ทำ interpolation (เหมือน single-quoted string) ให้ใส่
identifier ในเครื่องหมาย single quote:

```ruby
literal_text = <<~'TEXT'
  ราคานี้ยังไม่รวม #{tax}
  ให้แสดง #{} ตรงๆ ไม่ต้องแปลผล
TEXT

puts literal_text
# ราคานี้ยังไม่รวม #{tax}
# ให้แสดง #{} ตรงๆ ไม่ต้องแปลผล
```

### ตัวอย่างการใช้งานจริง: สร้าง SQL query หรือ HTML

```ruby
table_name = "users"
condition = "age > 18"

query = <<~SQL
  SELECT *
  FROM #{table_name}
  WHERE #{condition}
  ORDER BY created_at DESC
SQL

puts query
```

Heredoc ใช้บ่อยมากในโค้ด Rails จริง เช่น การเขียน raw SQL, การสร้าง email body แบบ
text-only, การเขียนข้อความ help/usage ของ CLI tool, หรือแม้แต่การเขียน multi-line
string ใน test file

---

## Step 29: Regular Expression เบื้องต้น — Regexp literal, match?, match, =~

**Regular Expression** (มักย่อว่า **regex** หรือ **regexp**) คือภาษาขนาดเล็กสำหรับ
บรรยาย "รูปแบบ" ของข้อความ ใช้สำหรับค้นหา ตรวจสอบ (validate) หรือแยกส่วนข้อความที่มี
รูปแบบซับซ้อนกว่าที่ `include?`/`split` ธรรมดาจะทำได้

### สร้าง Regexp literal

Ruby เขียน Regexp literal ด้วยเครื่องหมาย `/.../ `

```ruby
pattern = /ruby/
pattern.class  # => Regexp

# หรือสร้างผ่าน Regexp.new (ใช้เมื่อ pattern มาจากตัวแปร ต้องสร้างแบบ dynamic)
dynamic_pattern = Regexp.new("ruby")
```

### `match?` — วิธีที่แนะนำเมื่อแค่ต้องการรู้ว่า "match หรือไม่"

```ruby
"I love Ruby".match?(/ruby/)   # => false (Regexp เคสตรงตัวพิมพ์ ต้องพิมพ์ตรงกัน)
"I love Ruby".match?(/Ruby/)   # => true
"I love Ruby".match?(/ruby/i)  # => true  (flag /i = ignore case ไม่สนตัวพิมพ์ใหญ่เล็ก)
```

`match?` คืนค่า `true`/`false` เท่านั้น (ไม่คืนรายละเอียดของสิ่งที่ match ได้) ทำงาน
**เร็วที่สุด** ในบรรดา method ตรวจสอบ regex ทั้งหมด จึงควรใช้เมื่อแค่ต้องการรู้ผลลัพธ์
true/false อย่างเดียว (เช่นใน `if` condition)

### `match` — เมื่อต้องการรายละเอียดของสิ่งที่ match ได้

```ruby
result = "email: john@example.com".match(/[\w.]+@[\w.]+/)
result.class      # => MatchData
result[0]         # => "john@example.com"   (ข้อความทั้งหมดที่ match)
result.pre_match  # => "email: "            (ข้อความก่อนหน้าส่วนที่ match)
result.post_match # => ""                   (ข้อความหลังส่วนที่ match)

no_match = "hello world".match(/\d+/)
no_match  # => nil  (ไม่ match คืนค่า nil ไม่ error)
```

`match` คืนค่าเป็น object ชนิด `MatchData` ถ้าเจอ หรือ `nil` ถ้าไม่เจอ — เหมาะเมื่อ
ต้องการดึงส่วนของข้อความที่ match ออกมาใช้งานต่อ

### `=~` — operator แบบดั้งเดิม คืนค่าตำแหน่ง index

```ruby
"hello world" =~ /world/   # => 6  (ตำแหน่งที่พบ, นับจาก 0)
"hello world" =~ /xyz/     # => nil (ไม่พบ)

# นิยมใช้ใน if โดยตรง เพราะ nil เป็น falsy และตัวเลข (แม้แต่ 0) เป็น truthy
if "hello@example.com" =~ /@/
  puts "มี @ อยู่ในข้อความ"
end
```

> **ข้อควรระวัง:** `=~` มีกับดักคือถ้าตำแหน่งที่พบคือ `0` (match ตั้งแต่ตัวแรกสุด)
> ผลลัพธ์จะเป็น `0` ซึ่งใน Ruby `0` ถือเป็น **truthy** (ไม่ใช่ falsy เหมือนบางภาษา
> เช่น JavaScript/Python) จึงยังทำงานถูกต้องใน `if` — แต่ก็เป็นจุดที่มือใหม่มักสับสน
> ในทางปฏิบัติปัจจุบันแนะนำให้ใช้ `match?` แทน `=~` เมื่อแค่ต้องการรู้ true/false
> เพราะสื่อเจตนาชัดเจนกว่า

### Pattern พื้นฐานที่ใช้บ่อยที่สุด

```ruby
# \d  = ตัวเลข 0-9 หนึ่งตัว        \D = ไม่ใช่ตัวเลข
# \w  = ตัวอักษร/ตัวเลข/underscore  \W = ตรงข้าม
# \s  = whitespace (space, tab, newline)  \S = ไม่ใช่ whitespace
# .   = ตัวอักษรอะไรก็ได้ 1 ตัว (ยกเว้น newline)
# ^   = จุดเริ่มต้นบรรทัด            $  = จุดสิ้นสุดบรรทัด
# \A  = จุดเริ่มต้นของ string จริงๆ  \z = จุดสิ้นสุดของ string จริงๆ
# +   = 1 ตัวขึ้นไป                  *  = 0 ตัวขึ้นไป
# ?   = 0 หรือ 1 ตัว (optional)
# {n} = ต้องมีพอดี n ตัว             {n,m} = มีระหว่าง n ถึง m ตัว
# []  = character class (ตัวใดตัวหนึ่งในกลุ่ม)
# |   = "หรือ" (alternation)

"12345".match?(/\A\d+\z/)         # => true  (ทั้ง string เป็นตัวเลขล้วน)
"12a45".match?(/\A\d+\z/)         # => false (มีตัวอักษรปนอยู่)

"hello123".match?(/\A[a-z]+\d+\z/)  # => true (ตัวอักษรเล็กตามด้วยตัวเลข)

"2026-09-26".match?(/\A\d{4}-\d{2}-\d{2}\z/)  # => true (รูปแบบวันที่ YYYY-MM-DD)

"cat".match?(/^(cat|dog|bird)$/)  # => true (เป็นหนึ่งใน cat, dog, bird)
```

> **`^`/`$` vs `\A`/`\z`:** `^` และ `$` หมายถึงต้นและท้าย **บรรทัด** (line) แต่ `\A`
> และ `\z` หมายถึงต้นและท้ายของ **string ทั้งก้อน** จริงๆ ถ้า string มีหลายบรรทัด
> (มี `\n` อยู่ข้างใน) `^`/`$` อาจ match กลาง string ได้ ในขณะที่ `\A`/`\z` จะไม่ ในงาน
> validation (เช่น ตรวจสอบว่า input ทั้งก้อนเป็นตัวเลขล้วนหรือไม่) ควรใช้ `\A`/`\z`
> เพื่อความปลอดภัยเสมอ

### ตัวอย่าง validation ที่ใช้บ่อยในงานจริง

```ruby
def valid_email?(text)
  text.match?(/\A[\w.+-]+@[\w-]+\.[a-z]{2,}\z/i)
end

valid_email?("john@example.com")   # => true
valid_email?("not-an-email")       # => false
valid_email?("john@example")       # => false (ไม่มี . ตามด้วยตัวอักษร)

def valid_thai_phone?(text)
  text.match?(/\A0\d{9}\z/)  # เบอร์ไทย: ขึ้นต้น 0 ตามด้วยตัวเลข 9 ตัว รวม 10 หลัก
end

valid_thai_phone?("0812345678")  # => true
valid_thai_phone?("081-234-5678") # => false (มีขีดปน)
```

---

## Step 30: แบบฝึกหัดโปรเจกต์ — Mini Template Engine และ Text Report Generator

Step นี้เป็น Step สุดท้ายของ Part 003 เราจะรวมทุกอย่างที่เรียนมา (String methods,
interpolation, Heredoc, Regex) เข้าด้วยกันในโปรเจกต์เดียว พร้อมสอน **named captures**
และ **gsub กับ block** ซึ่งเป็นเทคนิค regex ขั้นสูงอีกเล็กน้อยที่จำเป็นสำหรับโจทย์นี้

### ความรู้เพิ่มเติมก่อนเริ่มโจทย์: Named Captures และ gsub กับ block

```ruby
# Named capture group: ตั้งชื่อให้ส่วนที่ capture ได้ด้วย (?<ชื่อ>...)
text = "วันที่: 2026-09-26"
if (m = text.match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/))
  puts "ปี: #{m[:year]}, เดือน: #{m[:month]}, วัน: #{m[:day]}"
end
# ปี: 2026, เดือน: 09, วัน: 26
```

Named capture ทำให้โค้ดอ่านง่ายกว่าการอ้าง index (`m[1]`, `m[2]`) มาก เพราะสื่อความหมาย
ของแต่ละส่วนที่จับได้ชัดเจน

```ruby
# gsub รับ block ได้ด้วย — block จะถูกเรียกทุกครั้งที่เจอ match พร้อมส่งข้อความ
# ที่ match นั้นเข้าไปเป็น argument (ผ่านตัวแปร implicit หรือรับ argument ตรงๆ)
text = "ราคา 100 บาท และ 250 บาท"

result = text.gsub(/\d+/) { |number| (number.to_i * 1.07).round.to_s }
puts result  # => ราคา 107 บาท และ 268 บาท (คำนวณ VAT 7% ให้ทุกตัวเลขที่เจอ)
```

`gsub` กับ block ทรงพลังมากเพราะแทนที่จะแทนที่ด้วยค่าคงที่ตัวเดียว เราสามารถ **คำนวณ
ค่าที่จะแทนที่แบบไดนามิก** จากข้อความที่ match ได้ในแต่ละครั้ง

### โจทย์

เขียนโปรแกรม `template_engine.rb` ที่ทำหน้าที่เป็น **mini template engine** ง่ายๆ
รับ template string ที่มี placeholder รูปแบบ `{{ชื่อตัวแปร}}` และ Hash ของค่าที่จะ
แทนที่ แล้วคืนค่า string ที่แทนที่ placeholder ทุกตัวเรียบร้อยแล้ว จากนั้นใช้ template
engine นี้สร้าง **ใบเสร็จรับเงิน (receipt)** จากข้อมูลคำสั่งซื้อ พร้อม validate ว่า
อีเมลลูกค้าถูกต้องก่อนออกใบเสร็จ

**ข้อกำหนด:**

1. Method `render(template, data)` รับ template string และ Hash แล้วแทนที่ `{{key}}`
   ทุกตัวด้วยค่าจาก `data[key]` (ถ้าไม่มี key นั้นใน data ให้แทนที่ด้วย `"???"`)
2. ใช้ Heredoc (`<<~`) เขียน template ของใบเสร็จ
3. Validate อีเมลลูกค้าด้วย Regex ก่อนออกใบเสร็จ ถ้าอีเมลผิดรูปแบบให้แจ้งเตือนและไม่
   ออกใบเสร็จ
4. คำนวณ VAT 7% จากยอดรวม แล้วแสดงทั้งยอดก่อน VAT, VAT, และยอดสุทธิ

### เฉลย

```ruby
# frozen_string_literal: true

# template_engine.rb

# แทนที่ {{key}} ทุกตัวใน template ด้วยค่าจาก data
# ถ้า key ไหนไม่มีใน data ให้แทนที่ด้วย "???" แทนที่จะปล่อยให้ error
def render(template, data)
  template.gsub(/\{\{(?<key>\w+)\}\}/) do
    key = Regexp.last_match(:key).to_sym
    data.key?(key) ? data[key].to_s : "???"
  end
end

# ตรวจสอบรูปแบบอีเมลเบื้องต้นด้วย regex
def valid_email?(email)
  email.match?(/\A[\w.+-]+@[\w-]+\.[a-z]{2,}\z/i)
end

# จัดรูปแบบตัวเลขให้มี comma คั่นหลักพัน เช่น 12345 => "12,345"
def format_number(number)
  number.to_s.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
end

# สร้างใบเสร็จจากข้อมูลคำสั่งซื้อ คืนค่า string ใบเสร็จ หรือ nil ถ้าอีเมลไม่ถูกต้อง
def build_receipt(order)
  unless valid_email?(order[:email])
    puts "เกิดข้อผิดพลาด: อีเมล '#{order[:email]}' ไม่ถูกต้อง ไม่สามารถออกใบเสร็จได้"
    return nil
  end

  subtotal = order[:items].sum { |item| item[:price] * item[:quantity] }
  vat = (subtotal * 0.07).round
  total = subtotal + vat

  item_lines = order[:items].map do |item|
    line_total = item[:price] * item[:quantity]
    "  - #{item[:name].ljust(20)} x#{item[:quantity]}  #{format_number(line_total)} บาท"
  end.join("\n")

  template = <<~RECEIPT
    ================================
    ใบเสร็จรับเงิน
    ================================
    ลูกค้า: {{customer_name}}
    อีเมล: {{email}}
    --------------------------------
    {{items}}
    --------------------------------
    ยอดก่อน VAT: {{subtotal}} บาท
    VAT (7%):    {{vat}} บาท
    ยอดสุทธิ:    {{total}} บาท
    ================================
    ขอบคุณที่ใช้บริการ!
  RECEIPT

  render(template, {
    customer_name: order[:customer_name],
    email: order[:email],
    items: item_lines,
    subtotal: format_number(subtotal),
    vat: format_number(vat),
    total: format_number(total)
  })
end

# --- ทดลองใช้งาน ---

order = {
  customer_name: "มานี มีนา",
  email: "manee@example.com",
  items: [
    { name: "หนังสือ Ruby", price: 350, quantity: 2 },
    { name: "เสื้อยืด Rails", price: 590, quantity: 1 },
    { name: "สติกเกอร์", price: 25, quantity: 4 }
  ]
}

receipt = build_receipt(order)
puts receipt if receipt

puts
puts "ทดสอบกับอีเมลผิดรูปแบบ:"
bad_order = order.merge(email: "manee-not-an-email")
build_receipt(bad_order)
```

ทดสอบรัน:

```bash
ruby template_engine.rb
```

ผลลัพธ์ที่ได้ (โดยประมาณ):

```
================================
ใบเสร็จรับเงิน
================================
ลูกค้า: มานี มีนา
อีเมล: manee@example.com
--------------------------------
  - หนังสือ Ruby           x2  700 บาท
  - เสื้อยืด Rails         x1  590 บาท
  - สติกเกอร์              x4  100 บาท
--------------------------------
ยอดก่อน VAT: 1,390 บาท
VAT (7%):    97 บาท
ยอดสุทธิ:    1,487 บาท
================================
ขอบคุณที่ใช้บริการ!

ทดสอบกับอีเมลผิดรูปแบบ:
เกิดข้อผิดพลาด: อีเมล 'manee-not-an-email' ไม่ถูกต้อง ไม่สามารถออกใบเสร็จได้
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `Regexp.last_match(:key)` — ดึงค่าของ named capture group ล่าสุดที่ match ได้
  ใช้ภายใน block ของ `gsub` เพื่ออ้างถึงกลุ่มที่ตั้งชื่อไว้ (`(?<key>\w+)`)
- `data.key?(key)` — ตรวจสอบว่า Hash มี key นี้อยู่หรือไม่ (จะอธิบายละเอียดใน Part 005
  เรื่อง Hash) ใช้ป้องกันไม่ให้ได้ `nil` แล้วพัง เวลา key ไม่ตรงกับที่ template ต้องการ
- `format_number` ใช้เทคนิค `reverse` สองครั้งร่วมกับ regex `(\d{3})(?=\d)` — ตัวอย่างนี้
  ใช้ **lookahead** `(?=\d)` ซึ่งหมายถึง "ตำแหน่งที่ตามด้วยตัวเลข แต่ไม่นับตัวเลขนั้น
  เป็นส่วนหนึ่งของ match" ทำให้ใส่ comma คั่นได้ทุก 3 หลักจากขวาไปซ้ายอย่างถูกต้อง
  โดยไม่ใส่ comma เกินที่ตำแหน่งซ้ายสุด
- `.ljust(20)` จาก Step 24 ถูกใช้จริงเพื่อจัดคอลัมน์ให้ใบเสร็จอ่านง่าย

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม method `valid_thai_phone?(phone)` ที่ตรวจสอบว่าเบอร์โทรเป็นรูปแบบไทยที่ถูกต้อง
   (ขึ้นต้นด้วย `0` ตามด้วยตัวเลข 9 ตัว หรือรูปแบบที่มีขีดคั่น เช่น `081-234-5678`)
   แล้วเพิ่ม key `:phone` ให้ `order` และแสดงเบอร์โทรในใบเสร็จด้วย (ใบ้: เขียน regex
   ที่รองรับทั้งสองรูปแบบด้วย `|` หรือทำให้ขีด `-` เป็น optional ด้วย `-?`)
2. เขียนโปรแกรมแยกต่างหากชื่อ `word_frequency.rb` ที่รับข้อความยาวหนึ่งย่อหน้า (เขียน
   เป็น Heredoc) แล้วนับความถี่ของแต่ละคำ (แปลงเป็นตัวพิมพ์เล็กทั้งหมดก่อนนับ, ตัด
   เครื่องหมายวรรคตอนออกด้วย `gsub(/[^\w\s]/, "")` ก่อน แล้วค่อย `split`) แสดงผลคำที่
   พบบ่อยที่สุด 5 อันดับแรก
3. เขียน method `mask_email(email)` ที่ปิดบังอีเมลบางส่วนเพื่อความเป็นส่วนตัว เช่น
   `"john.doe@example.com"` ให้กลายเป็น `"jo***@example.com"` (แสดง 2 ตัวอักษรแรกของ
   ชื่อผู้ใช้ แล้วตามด้วย `***` ก่อน `@` ที่เหลือ) ใช้ named capture แยกส่วน username
   กับ domain ออกจากกันก่อนประกอบผลลัพธ์ใหม่

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- สร้าง String ได้ทั้งแบบ single quote และ double quote และเข้าใจว่าทำไม interpolation
  กับ escape sequence ทำงานเฉพาะใน double quote
- ใช้ String methods ที่จำเป็นในงานจริงได้คล่อง: `length`, `upcase`/`downcase`,
  `strip`, `include?`, `start_with?`/`end_with?`, `split`, `gsub`/`sub`, `slice`/`[]`,
  `reverse`, `chars`, `each_char`
- เข้าใจ string interpolation อย่างลึกซึ้งว่า `#{}` รับ expression อะไรก็ได้ และ
  เรียก `to_s` ให้อัตโนมัติ
- แยกแยะความแตกต่างระหว่าง `+`, `<<`, `concat` ในแง่ mutability และรู้ว่าเมื่อไหร่
  ควรใช้ตัวไหน รวมถึงเข้าใจกับดักเรื่อง object เดียวกันถูกแก้ไขโดยไม่ตั้งใจ
- เข้าใจ `frozen_string_literal` อย่างละเอียดว่าเกี่ยวข้องกับ mutability อย่างไร และ
  รู้วิธีสร้าง mutable string เมื่อจำเป็นแม้ทั้งไฟล์จะ frozen
- เขียน Heredoc ได้ทั้งสามแบบ (`<<`, `<<-`, `<<~`) และรู้ว่าทำไม `<<~` (squiggly
  heredoc) เป็นตัวเลือกมาตรฐานในโค้ดยุคปัจจุบัน
- เข้าใจพื้นฐาน Regular Expression: Regexp literal, `match?`, `match`, `=~`,
  pattern พื้นฐาน (`\d`, `\w`, `\s`, `+`, `*`, `\A`/`\z`), named capture group,
  และการใช้ `gsub` ร่วมกับ block เพื่อแทนที่ข้อความแบบไดนามิก
- สร้างโปรเจกต์ mini template engine ที่รวม String methods, Heredoc, และ Regex
  เข้าด้วยกันเป็นโปรแกรมที่ใช้งานได้จริง

**ต่อไป (Part 004):** เราจะเริ่มเจาะลึกเรื่อง **Array** — โครงสร้างข้อมูลที่เก็บ
ค่าหลายค่าเรียงกันเป็นลำดับ ตั้งแต่การสร้าง Array หลากหลายวิธี, การวนลูป (iteration)
ด้วย `each`, และ method สำคัญที่ใช้บ่อยที่สุดในโค้ด Ruby/Rails จริงอย่าง `map`,
`select`, `reduce`, และ `each_with_index` ซึ่งเป็นพื้นฐานสำคัญของแนวคิด
**functional-style programming** ที่ Ruby สนับสนุนอย่างเต็มที่
