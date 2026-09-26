# Part 004: Array — การสร้าง, Iteration และ Method สำคัญ

> **Step ครอบคลุมใน Part นี้:** Step 31–40
> **ระดับ:** เริ่มต้น–ปานกลาง (ต้องผ่าน Part 001–003 มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 31: Array คืออะไร และวิธีสร้าง Array ทุกแบบ
- Step 32: Indexing, Negative Index, และ Slicing
- Step 33: Method ที่แก้ไข Array ตัวเอง (Mutating) กับที่ไม่แก้ไข (Non-mutating)
- Step 34: `each`, `each_with_index`, `each_with_object`
- Step 35: `map`/`collect` — สร้าง Array ใหม่จากการแปลงค่า
- Step 36: `select`/`filter`, `reject` — กรองข้อมูล
- Step 37: `reduce`/`inject` — พับข้อมูลทั้ง Array ให้เหลือค่าเดียว
- Step 38: `find`/`detect`, `all?`/`any?`/`none?`/`count` — method ตรวจสอบเงื่อนไข
- Step 39: `flatten`, `zip`, `sort_by`
- Step 40: แบบฝึกหัดโปรเจกต์ — วิเคราะห์คะแนนนักเรียน

---

## Step 31: Array คืออะไร และวิธีสร้าง Array ทุกแบบ

**Array** คือโครงสร้างข้อมูลที่เก็บรายการของค่าหลายๆ ค่าเรียงตามลำดับ (ordered collection)
โดยแต่ละค่าจะมี **index** (ตำแหน่ง) กำกับ เริ่มนับจาก `0` เสมอ

จุดเด่นของ Array ใน Ruby คือ **เก็บข้อมูลต่างชนิดกันในตัวเดียวกันได้** (ไม่เหมือนภาษาที่เป็น
static type อย่าง Java ที่ array ต้องเป็นชนิดเดียวกันหมด)

```ruby
# frozen_string_literal: true

numbers = [1, 2, 3, 4, 5]
names = ["สมชาย", "สมหญิง", "มานี"]
mixed = [1, "two", :three, 4.0, true, nil, [5, 6]]  # ผสมชนิดข้อมูล และมี Array ซ้อน Array ได้
empty = []

p numbers
p mixed
```

```
[1, 2, 3, 4, 5]
[1, "two", :three, 4.0, true, nil, [5, 6]]
```

### วิธีสร้าง Array แบบต่างๆ

```ruby
# 1) Array literal — วิธีที่ใช้บ่อยที่สุด
fruits = ["apple", "banana", "cherry"]

# 2) Array.new — สร้าง array ว่าง
empty_array = Array.new
p empty_array  # => []

# 3) Array.new(size) — สร้าง array ขนาดที่กำหนด เติมด้วย nil
sized_array = Array.new(3)
p sized_array  # => [nil, nil, nil]

# 4) Array.new(size, default_value) — เติมด้วยค่าเริ่มต้นเดียวกันทุกช่อง
zeros = Array.new(5, 0)
p zeros  # => [0, 0, 0, 0, 0]

# 5) Array.new(size) { |i| ... } — สร้างโดยใช้ block คำนวณค่าแต่ละช่องจาก index
squares = Array.new(5) { |i| i * i }
p squares  # => [0, 1, 4, 9, 16]

# 6) %w[] — shortcut สร้าง Array ของ String โดยไม่ต้องใส่ quote และ comma
colors = %w[red green blue]
p colors  # => ["red", "green", "blue"]

# 7) %i[] — shortcut สร้าง Array ของ Symbol
statuses = %i[active pending closed]
p statuses  # => [:active, :pending, :closed]

# 8) Array() — แปลงค่าให้เป็น Array อย่างปลอดภัย (ใช้บ่อยเวลาไม่แน่ใจว่าค่าที่ได้มาเป็น
#    Array อยู่แล้วหรือเป็นค่าเดี่ยว)
p Array(nil)        # => []
p Array([1, 2])     # => [1, 2]  (เป็น Array อยู่แล้ว คืนค่าเดิม)
p Array("x")        # => ["x"]  (ห่อค่าเดี่ยวด้วย Array)
```

### ทำไม `Array.new(size, default_value)` ถึงอันตรายเมื่อ default เป็น object ที่ mutable

นี่คือกับดักที่มือใหม่เจอบ่อยมาก และเป็นตัวอย่างสำคัญที่ปูทางไปสู่เรื่อง **aliasing** ใน
Step 33

```ruby
# ระวัง! default_value ตัวเดียวกันถูกใช้ "อ้างอิง object เดียวกัน" ในทุกช่อง
danger = Array.new(3, [])
danger[0] << "a"
p danger  # => [["a"], ["a"], ["a"]]  <- เปลี่ยนช่องเดียว แต่กระทบทุกช่อง! (bug ร้ายแรง)

# วิธีแก้ที่ถูกต้อง: ใช้ block เพื่อให้แต่ละช่องได้ Array คนละตัว (แยก object กันจริงๆ)
safe = Array.new(3) { [] }
safe[0] << "a"
p safe  # => [["a"], [], []]  <- แก้เฉพาะช่องที่ต้องการ ถูกต้อง
```

**สาเหตุ:** เมื่อ `default_value` เป็น object (เช่น Array, Hash, String ที่ไม่ frozen)
Ruby จะไม่สร้าง object ใหม่ให้แต่ละช่อง แต่ใช้ **object เดียวกัน** (reference เดียวกัน)
ซ้ำๆ ทุกช่อง พอแก้ไข object นั้นผ่านช่องใดช่องหนึ่ง ทุกช่องที่ชี้ไปยัง object เดียวกันก็จะ
เปลี่ยนตามไปด้วย ส่วนการใช้ `{ }` (block) จะเรียก block ใหม่ทุกรอบ ทำให้ได้ object ใหม่จริงๆ
ทุกครั้ง

---

## Step 32: Indexing, Negative Index, และ Slicing

### Index พื้นฐานและ Negative Index

```ruby
fruits = ["apple", "banana", "cherry", "durian", "elderberry"]
#            0         1         2         3           4
#           -5        -4        -3        -2          -1

p fruits[0]     # => "apple"    (ตัวแรก)
p fruits[2]     # => "cherry"
p fruits[-1]    # => "elderberry"  (ตัวสุดท้าย — ใช้บ่อยมาก ไม่ต้องรู้ความยาว array ก่อน)
p fruits[-2]    # => "durian"      (ตัวรองสุดท้าย)
p fruits[10]    # => nil           (index เกินขอบเขต ไม่ error แค่คืน nil)

# method อ่านง่ายกว่าการเดา index
p fruits.first        # => "apple"     (เทียบเท่า fruits[0])
p fruits.last         # => "elderberry" (เทียบเท่า fruits[-1])
p fruits.first(2)      # => ["apple", "banana"]      (เอา n ตัวแรก)
p fruits.last(2)       # => ["durian", "elderberry"]  (เอา n ตัวสุดท้าย)
```

### Slicing — ดึงข้อมูลหลายตัวพร้อมกัน

```ruby
fruits = ["apple", "banana", "cherry", "durian", "elderberry"]

# แบบ start, length
p fruits[1, 2]      # => ["banana", "cherry"]  (เริ่มที่ index 1 เอามา 2 ตัว)

# แบบ Range
p fruits[1..3]      # => ["banana", "cherry", "durian"]     (รวม index 3)
p fruits[1...3]     # => ["banana", "cherry"]                (ไม่รวม index 3, exclusive)
p fruits[2..]        # => ["cherry", "durian", "elderberry"] (ตั้งแต่ index 2 ถึงท้าย)
p fruits[..2]        # => ["apple", "banana", "cherry"]      (ตั้งแต่ต้นถึง index 2)
p fruits[-3..-1]     # => ["cherry", "durian", "elderberry"] (นับจากท้ายด้วย negative range)

# method slice / values_at
p fruits.slice(1, 2)         # => ["banana", "cherry"] (เหมือน fruits[1, 2] แต่อ่านชื่อชัดกว่า)
p fruits.values_at(0, 2, 4)   # => ["apple", "cherry", "elderberry"] (ดึงหลาย index ไม่ต่อเนื่อง)
```

### `dig` — เข้าถึงข้อมูลใน Array ซ้อน Array (nested) อย่างปลอดภัย

```ruby
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

p matrix[1][2]      # => 6   (วิธีปกติ: index ต่อ index)
p matrix.dig(1, 2)   # => 6   (เทียบเท่ากัน แต่ dig ปลอดภัยกว่าเมื่อโครงสร้างอาจไม่สมบูรณ์)

nested = [1, [2, [3, [4, 5]]]]
p nested.dig(1, 1, 1, 0)  # => 4

# ทำไม dig ปลอดภัยกว่า: ถ้า path ระหว่างทางเจอ nil จะคืน nil แทนที่จะ error
incomplete = [[1, 2], nil, [5, 6]]
# incomplete[1][0]     # => NoMethodError: undefined method `[]' for nil (โปรแกรม crash!)
p incomplete.dig(1, 0)  # => nil (ไม่ crash แค่คืน nil)
```

### การกำหนดค่าผ่าน index (Assignment) — ขยาย Array อัตโนมัติ

```ruby
arr = [1, 2, 3]
arr[1] = "TWO"
p arr  # => [1, "TWO", 3]

# กำหนดค่าที่ index เกินขอบเขตปัจจุบัน -> Ruby ขยาย array ให้อัตโนมัติ เติม nil ตรงกลาง
arr[6] = "SEVEN"
p arr  # => [1, "TWO", 3, nil, nil, nil, "SEVEN"]
```

---

## Step 33: Method ที่แก้ไข Array ตัวเอง (Mutating) กับที่ไม่แก้ไข (Non-mutating)

หลักการสำคัญของ Ruby ที่ต้องจำให้ขึ้นใจ: **method ที่ลงท้ายด้วย `!` มักจะแก้ไข object เดิม
(mutate) และคืนค่า object เดิมนั้น** ส่วน method ที่ไม่มี `!` มักจะ**คืน object ใหม่**
โดยไม่แตะต้อง object ตั้งต้น (ยกเว้น method ที่ออกแบบมาให้ mutate อยู่แล้วโดยธรรมชาติ เช่น
`push`, `<<`, `pop`, `shift`, `unshift` ซึ่งไม่มี `!` แต่ก็ mutate เสมอ)

### เพิ่ม/ลบสมาชิกท้าย Array — `push`, `<<`, `pop`

```ruby
stack = [1, 2, 3]

# push และ << ทำงานเหมือนกัน: เพิ่มสมาชิกต่อท้าย และ mutate array เดิม
stack.push(4)
p stack   # => [1, 2, 3, 4]

stack << 5
p stack   # => [1, 2, 3, 4, 5]

# << สามารถต่อกันเป็นลูกโซ่ได้ (method นี้ return ตัว array เอง)
stack << 6 << 7
p stack   # => [1, 2, 3, 4, 5, 6, 7]

# push รับหลายค่าพร้อมกันได้
stack.push(8, 9)
p stack   # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

# pop เอาสมาชิกท้ายสุดออก และคืนค่าที่เอาออกมา (ไม่ใช่คืน array)
last = stack.pop
p last    # => 9
p stack   # => [1, 2, 3, 4, 5, 6, 7, 8]

# pop(n) เอาออก n ตัวจากท้าย คืนเป็น array
p stack.pop(2)  # => [7, 8]
p stack          # => [1, 2, 3, 4, 5, 6]
```

### เพิ่ม/ลบสมาชิกหัว Array — `unshift`, `shift`

```ruby
queue = [3, 4, 5]

# unshift เพิ่มสมาชิกด้านหน้า (ช้ากว่า push เพราะต้องเลื่อน index ทุกตัว)
queue.unshift(2)
p queue   # => [2, 3, 4, 5]

queue.unshift(0, 1)
p queue   # => [0, 1, 2, 3, 4, 5]

# shift เอาสมาชิกตัวแรกออก และคืนค่านั้น
first = queue.shift
p first   # => 0
p queue   # => [1, 2, 3, 4, 5]
```

> **เกร็ดประสิทธิภาพ:** `push`/`pop` มีความซับซ้อน O(1) (เร็ว ไม่ขึ้นกับขนาด array) เพราะ
> ทำงานที่ปลายท้าย แต่ `unshift`/`shift` มีความซับซ้อน O(n) เพราะ Ruby ต้องเลื่อนตำแหน่ง
> สมาชิกทุกตัวหนึ่งช่อง ถ้าต้องเพิ่ม/ลบข้อมูลที่หัว array บ่อยๆ ในโค้ดที่ประสิทธิภาพสำคัญ
> ควรพิจารณาโครงสร้างข้อมูลอื่น (เช่น กลับลำดับการเก็บข้อมูล)

### `sort` vs `sort!`, `uniq` vs `uniq!` — คู่ mutating/non-mutating

```ruby
numbers = [5, 3, 1, 4, 1, 5, 9, 2, 6]

# sort คืน array ใหม่ที่เรียงแล้ว โดยไม่แตะ numbers เดิม
sorted = numbers.sort
p sorted    # => [1, 1, 2, 3, 4, 5, 5, 6, 9]
p numbers   # => [5, 3, 1, 4, 1, 5, 9, 2, 6]  <- ไม่เปลี่ยน!

# sort! เรียงแล้วแก้ไข numbers เดิมเลย คืนค่าเป็น numbers ตัวเดิม (ที่เรียงแล้ว)
numbers.sort!
p numbers   # => [1, 1, 2, 3, 4, 5, 5, 6, 9]  <- เปลี่ยนแล้ว!

# uniq / uniq! ก็เช่นเดียวกัน
with_dupes = [1, 2, 2, 3, 3, 3, 4]
unique = with_dupes.uniq
p unique       # => [1, 2, 3, 4]
p with_dupes    # => [1, 2, 2, 3, 3, 3, 4]  <- ไม่เปลี่ยน

with_dupes.uniq!
p with_dupes    # => [1, 2, 3, 4]  <- เปลี่ยนแล้ว

# reverse / reverse! เช่นกัน
p [1, 2, 3].reverse   # => [3, 2, 1] (array ใหม่)
arr = [1, 2, 3]
arr.reverse!
p arr                  # => [3, 2, 1] (mutate ตัวเดิม)

# ข้อควรระวัง: uniq! คืน nil ถ้าไม่มีอะไรถูกลบออก (ต่างจาก uniq ที่คืน array เสมอ)
already_unique = [1, 2, 3]
result = already_unique.uniq!
p result           # => nil  !! (ระวังถ้าเอาไปเช็คเงื่อนไขต่อ)
p already_unique   # => [1, 2, 3] (ตัว array เองไม่เปลี่ยนแปลง แต่ return value เป็น nil)
```

> **แนวปฏิบัติที่ดี:** method ที่มี `!` ส่วนใหญ่ (`sort!`, `uniq!`, `map!`, `select!`,
> `reject!`, `compact!`, `flatten!`) จะ **คืนค่า `nil` ถ้าไม่มีการเปลี่ยนแปลงเกิดขึ้น**
> เพราะฉะนั้นห้ามเขียนแบบ `arr = arr.uniq!` เด็ดขาด เพราะถ้า `arr` ไม่มีค่าซ้ำอยู่แล้ว
> `arr` จะกลายเป็น `nil` ทันที! ให้เรียก `arr.uniq!` เฉยๆ แล้วไปดูค่าใน `arr` แทน

### Aliasing — กับดักที่อันตรายที่สุดของ Mutable Object

นี่คือแนวคิดสำคัญที่สุดของ Part นี้ Array เป็น **mutable object** (แก้ไขค่าภายในได้โดยไม่
ต้องสร้างตัวใหม่) และตัวแปรใน Ruby เป็นเพียง**ป้ายชื่อที่ชี้ไปยัง object** ไม่ใช่ตัว object
เอง ดังนั้นการกำหนดตัวแปรหนึ่งด้วยอีกตัวแปรหนึ่ง (`b = a`) จึงเป็นการทำให้ทั้งสองชื่อ
**ชี้ไปยัง object เดียวกัน** ไม่ใช่การคัดลอกข้อมูล

```ruby
original = [1, 2, 3]
alias_ref = original   # ไม่ได้ copy! แค่ทำให้ alias_ref ชี้ไปยัง object เดียวกับ original

alias_ref << 4
p original    # => [1, 2, 3, 4]  <- เปลี่ยนตามไปด้วย! (เพราะเป็น object เดียวกัน)
p alias_ref   # => [1, 2, 3, 4]

p original.object_id == alias_ref.object_id  # => true (ยืนยันว่าเป็น object เดียวกันจริงๆ)
```

นี่คือสาเหตุที่บั๊กแบบ "แก้ตัวแปรหนึ่ง แล้วอีกตัวเปลี่ยนตามโดยไม่ได้ตั้งใจ" เกิดขึ้นบ่อยมาก
โดยเฉพาะเมื่อส่ง Array เข้า method แล้ว method นั้นแก้ไข array ที่รับมาโดยตรง

```ruby
def add_bonus_item!(cart)
  cart << "ของแถม"   # ใช้ << ซึ่ง mutate array ที่ส่งเข้ามาโดยตรง
end

shopping_cart = ["เสื้อ", "กางเกง"]
add_bonus_item!(shopping_cart)
p shopping_cart  # => ["เสื้อ", "กางเกง", "ของแถม"]  <- ถูกแก้จากใน method โดยไม่รู้ตัว
```

### `dup` และ `clone` — วิธีคัดลอก Array จริงๆ

```ruby
original = [1, 2, 3]
copy = original.dup   # dup สร้าง object ใหม่ที่มีข้อมูลเหมือนกัน (shallow copy)

copy << 4
p original  # => [1, 2, 3]      <- ไม่เปลี่ยน เพราะ copy เป็นคนละ object แล้ว
p copy      # => [1, 2, 3, 4]

p original.object_id == copy.object_id  # => false (คนละ object กันแน่นอน)
```

**`dup` vs `clone`:** ทั้งคู่ทำ shallow copy เหมือนกัน แต่ `clone` จะคัดลอกสถานะ `frozen`
มาด้วย ส่วน `dup` จะไม่คัดลอกสถานะ `frozen` มา (object ที่ได้จาก `dup` จะไม่ frozen เสมอ
ไม่ว่าต้นฉบับจะ frozen หรือไม่)

```ruby
frozen_arr = [1, 2, 3].freeze
p frozen_arr.frozen?  # => true

dup_arr = frozen_arr.dup
p dup_arr.frozen?     # => false  (dup ไม่รักษาสถานะ frozen)

clone_arr = frozen_arr.clone
p clone_arr.frozen?   # => true   (clone รักษาสถานะ frozen ไว้)
```

### ข้อควรระวัง: `dup`/`clone` เป็น Shallow Copy เท่านั้น

ถ้า Array มี Array/Hash/Object อื่นซ้อนอยู่ข้างใน `dup` จะคัดลอกแค่ **ชั้นนอกสุด** ส่วน
object ที่ซ้อนอยู่ข้างในยังคงเป็น object เดียวกัน (ยัง alias กันอยู่)

```ruby
nested_original = [[1, 2], [3, 4]]
nested_copy = nested_original.dup

nested_copy[0] << 99         # แก้ array ที่อยู่ "ข้างใน" ชั้นแรก
p nested_original  # => [[1, 2, 99], [3, 4]]  <- เปลี่ยนตามด้วย! เพราะเป็น sub-array ตัวเดียวกัน
p nested_copy      # => [[1, 2, 99], [3, 4]]

nested_copy << [5, 6]         # เพิ่ม array ใหม่ที่ "ชั้นนอกสุด"
p nested_original  # => [[1, 2, 99], [3, 4]]         <- ไม่เปลี่ยน (ชั้นนอกสุดคัดลอกแยกกันแล้ว)
p nested_copy      # => [[1, 2, 99], [3, 4], [5, 6]]
```

การจะคัดลอกแบบ **deep copy** (คัดลอกทุกชั้นจริงๆ) ต้องทำเอง เช่นใช้
`Marshal.load(Marshal.dump(nested_original))` หรือ (ใน Ruby บางเวอร์ชัน) เขียน recursive
copy เอง — รายละเอียดเชิงลึกเรื่องนี้จะกลับมาพูดอีกครั้งตอนเรียนเรื่อง object และ
metaprogramming

### `freeze` — ป้องกันการแก้ไข Array โดยไม่ตั้งใจ

```ruby
constants = [1, 2, 3].freeze
p constants.frozen?  # => true

begin
  constants << 4
rescue FrozenError => e
  puts "แก้ไขไม่ได้: #{e.message}"
  # => แก้ไขไม่ได้: can't modify frozen Array: [1, 2, 3]
end

# แต่ method ที่ไม่ mutate ยังใช้ได้ปกติ เพราะแค่ "อ่าน" ไม่ได้ "แก้ไข"
p constants.map { |n| n * 2 }  # => [2, 4, 6] (ทำงานได้ปกติ ไม่ error)
```

> **แนวคิดสำคัญ:** `freeze` ป้องกันแค่การแก้ไข**ชั้นนอกสุด** เท่านั้น (shallow freeze
> เหมือนกับ `dup`) ถ้ามี Array ซ้อนอยู่ข้างใน sub-array เหล่านั้นยังแก้ไขได้ตามปกติ เว้นแต่
> จะ `.freeze` แต่ละชั้นเองด้วย — ใช้ `freeze` เมื่อต้องการประกาศว่า "ค่าคงที่นี้ห้ามใครมา
> แก้ไขทีหลังโดยไม่ตั้งใจ" ซึ่งเป็นแนวปฏิบัติที่ดีสำหรับค่าคงที่ระดับ constant ของโปรแกรม

---

## Step 34: `each`, `each_with_index`, `each_with_object`

### `each` — วน loop ผ่านทุกสมาชิก

```ruby
fruits = ["apple", "banana", "cherry"]

fruits.each do |fruit|
  puts "ผลไม้: #{fruit}"
end
# => ผลไม้: apple
# => ผลไม้: banana
# => ผลไม้: cherry

# each คืนค่า array ตัวเดิม (ไม่ใช่ผลลัพธ์จากการวน loop) — จุดนี้สำคัญมาก
result = fruits.each { |fruit| fruit.upcase }
p result  # => ["apple", "banana", "cherry"]  <- คืน array เดิมเป๊ะๆ ไม่ใช่ค่าที่ upcase แล้ว
```

**นี่คือสิ่งที่ทำให้มือใหม่สับสนบ่อยที่สุด:** `each` ใช้สำหรับ**การกระทำ (side effect)**
เช่น พิมพ์ค่าออกจอ, บันทึกลงไฟล์, ส่ง request ไปเซิร์ฟเวอร์ — ไม่ใช่สำหรับสร้างข้อมูลใหม่
ถ้าต้องการ**แปลงข้อมูล**และได้ array ใหม่กลับมา ต้องใช้ `map` (Step 35) แทน

### `each_with_index` — วน loop พร้อมรู้ตำแหน่ง index

```ruby
fruits = ["apple", "banana", "cherry"]

fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end
# => 1. apple
# => 2. banana
# => 3. cherry
```

**อธิบาย:** `each_with_index` ส่งค่าเข้า block เป็นคู่ `(สมาชิก, index)` เสมอ — สังเกตว่า
สมาชิกมาก่อน index (คนละแบบกับบางภาษาที่ index มาก่อน) ใช้บ่อยเวลาต้องการแสดงลำดับที่ หรือ
ต้องเทียบสมาชิกกับตัวข้างเคียงโดยรู้ตำแหน่ง

```ruby
# ตัวอย่างการใช้ index จริงจัง: หาตำแหน่งของค่าที่ตรงเงื่อนไข
scores = [55, 80, 45, 90, 70]

scores.each_with_index do |score, index|
  puts "นักเรียนคนที่ #{index + 1} สอบตก (ได้ #{score} คะแนน)" if score < 50
end
# => นักเรียนคนที่ 3 สอบตก (ได้ 45 คะแนน)
```

### `each_with_object` — วน loop แล้วสะสมผลลัพธ์ลงใน object ที่กำหนด

`each_with_object` มีประโยชน์เมื่อต้องการ "สะสม" ผลลัพธ์ลงใน object (มักเป็น Hash หรือ
Array) ระหว่างวน loop โดยไม่ต้องประกาศตัวแปรสะสมแยกไว้ข้างนอกและ return มันเองตอนท้าย

```ruby
words = ["apple", "banana", "avocado", "blueberry", "cherry"]

# จัดกลุ่มคำตามตัวอักษรตัวแรก โดยใช้ Hash เป็น object ที่สะสมผล
grouped = words.each_with_object({}) do |word, hash|
  first_letter = word[0]
  hash[first_letter] ||= []       # ถ้ายังไม่มี key นี้ ให้สร้าง array ว่างก่อน
  hash[first_letter] << word
end

p grouped
# => {"a"=>["apple", "avocado"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}
```

**อธิบาย:** `each_with_object({})` เริ่มต้นด้วย Hash ว่าง `{}` เป็น object สะสม แล้วส่งเข้า
block เป็นค่าที่สอง (`hash`) ทุกรอบของ loop เราแก้ไข `hash` โดยตรง (mutate) และเมื่อ loop
จบ `each_with_object` จะ**คืนค่า object สะสมนั้นกลับมาเอง** — ต่างจาก `each` ที่คืน array
ตั้งต้นเสมอ

```ruby
# เทียบกับวิธีเขียนแบบไม่ใช้ each_with_object (ยาวกว่า ต้องประกาศตัวแปรแยก)
grouped_manual = {}
words.each do |word|
  first_letter = word[0]
  grouped_manual[first_letter] ||= []
  grouped_manual[first_letter] << word
end
p grouped_manual  # ผลลัพธ์เหมือนกันทุกประการ แต่ each_with_object กระชับกว่า
```

---

## Step 35: `map`/`collect` — สร้าง Array ใหม่จากการแปลงค่า

`map` (มีชื่อพ้อง `collect` ทำงานเหมือนกันทุกประการ) คือ method ที่ใช้บ่อยที่สุดตัวหนึ่งใน
Ruby ทำหน้าที่ **แปลงสมาชิกทุกตัวใน Array ด้วย block ที่กำหนด แล้วคืน Array ใหม่ที่มีขนาด
เท่าเดิม** โดยไม่แก้ไข array ต้นฉบับ

```ruby
numbers = [1, 2, 3, 4, 5]

doubled = numbers.map { |n| n * 2 }
p doubled   # => [2, 4, 6, 8, 10]
p numbers   # => [1, 2, 3, 4, 5]  <- ไม่เปลี่ยน (map ไม่ mutate)

# collect ทำงานเหมือน map เป๊ะๆ (เป็น alias กัน) — ในโค้ดจริงนิยมใช้ map มากกว่า
p numbers.collect { |n| n * 2 }  # => [2, 4, 6, 8, 10]
```

### ความแตกต่างสำคัญระหว่าง `map` กับ `each`

```ruby
names = ["somchai", "somying", "manee"]

# each: ใช้เพื่อทำ side effect (พิมพ์, บันทึก) — คืน array เดิมเสมอ ไม่สนใจค่าที่ block คืน
each_result = names.each { |name| name.capitalize }
p each_result  # => ["somchai", "somying", "manee"]  <- ไม่เปลี่ยนแปลง (capitalize ไม่ mutate name)

# map: ใช้เพื่อ "แปลงค่า" — คืน array ใหม่ที่รวบรวมค่าที่ block return กลับมาทุกรอบ
map_result = names.map { |name| name.capitalize }
p map_result   # => ["Somchai", "Somying", "Manee"]  <- ได้ array ใหม่ที่แปลงค่าแล้ว
```

**กฎการเลือกใช้:** ถามตัวเองว่า "ฉันต้องการ Array ใหม่ที่มีค่าถูกแปลงหรือเปล่า" ถ้าใช่ ใช้
`map` ถ้าแค่ต้องการทำอะไรกับแต่ละสมาชิกโดยไม่สนใจค่าที่ return กลับมา (เช่น พิมพ์ออกจอ)
ใช้ `each`

### `map!` — แปลงค่าและ mutate array ต้นฉบับเลย

```ruby
numbers = [1, 2, 3, 4, 5]
numbers.map! { |n| n ** 2 }
p numbers  # => [1, 4, 9, 16, 25]  <- array เดิมถูกแทนที่ด้วยค่าที่แปลงแล้ว
```

### ตัวอย่างการใช้งานจริง: แปลง Array ของ Hash

```ruby
students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 92 },
  { name: "มานี", score: 78 }
]

# ดึงเฉพาะชื่อออกมาเป็น array ใหม่
names = students.map { |student| student[:name] }
p names  # => ["สมชาย", "สมหญิง", "มานี"]

# สร้างข้อความสรุปสำหรับแต่ละคน
summaries = students.map { |s| "#{s[:name]}: #{s[:score]} คะแนน" }
p summaries
# => ["สมชาย: 85 คะแนน", "สมหญิง: 92 คะแนน", "มานี: 78 คะแนน"]

# map สามารถต่อ chain กับ method อื่นได้ (Enumerable chaining)
top_names = students.map { |s| s[:name] }.sort
p top_names  # => ["มานี", "สมชาย", "สมหญิง"]  (เรียงตามตัวอักษร)
```

### `flat_map` — ผสม `map` กับ `flatten(1)` ในตัวเดียว (แนะนำให้รู้จักไว้)

```ruby
sentences = ["hello world", "ruby is fun"]

# ถ้าใช้ map เฉยๆ จะได้ array ซ้อน array
p sentences.map { |s| s.split(" ") }
# => [["hello", "world"], ["ruby", "is", "fun"]]

# flat_map จะ "แบน" ผลลัพธ์ให้เป็น array ชั้นเดียวให้อัตโนมัติ
p sentences.flat_map { |s| s.split(" ") }
# => ["hello", "world", "ruby", "is", "fun"]
```

---

## Step 36: `select`/`filter`, `reject` — กรองข้อมูล

`select` (มีชื่อพ้อง `filter` ทำงานเหมือนกันทุกประการ) คือ method ที่คืน Array ใหม่ที่มี
เฉพาะสมาชิกที่ทำให้ block คืนค่าเป็น truthy (ไม่ใช่ `false`/`nil`) เท่านั้น

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

evens = numbers.select { |n| n.even? }
p evens    # => [2, 4, 6, 8, 10]

# filter เป็น alias ของ select ทำงานเหมือนกันเป๊ะๆ (มาจาก convention ของภาษาอื่นๆ เช่น
# JavaScript ที่ใช้ filter) ในโค้ด Ruby ทั่วไปเจอทั้งสองชื่อ แล้วแต่ความชอบของทีม
p numbers.filter { |n| n.even? }  # => [2, 4, 6, 8, 10]

p numbers   # => [1, 2, 3, ..., 10]  <- select/filter ไม่ mutate array ต้นฉบับ
```

### `reject` — ตรงข้ามกับ `select` (คัดออกสิ่งที่ตรงเงื่อนไข)

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

odds = numbers.reject { |n| n.even? }
p odds  # => [1, 3, 5, 7, 9]  <- เก็บเฉพาะตัวที่ block คืนค่าเป็น false

# reject { cond } เทียบเท่ากับ select { !cond } เสมอ — เลือกใช้ตัวที่อ่านแล้วเข้าใจง่ายกว่า
p numbers.select { |n| !n.even? }  # => [1, 3, 5, 7, 9]  (ผลลัพธ์เหมือนกัน แต่ reject อ่านง่ายกว่า)
```

### `select!` / `reject!` — mutate array ต้นฉบับ

```ruby
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
numbers.select! { |n| n > 5 }
p numbers  # => [6, 7, 8, 9, 10]  <- array เดิมถูกแก้ไข เหลือเฉพาะตัวที่ผ่านเงื่อนไข

numbers.reject! { |n| n.even? }
p numbers  # => [7, 9]
```

### ตัวอย่างการใช้งานจริง

```ruby
students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 45 },
  { name: "มานี", score: 78 },
  { name: "วิชัย", score: 30 }
]

passed = students.select { |s| s[:score] >= 50 }
failed = students.reject { |s| s[:score] >= 50 }

puts "สอบผ่าน: #{passed.map { |s| s[:name] }.join(", ")}"
puts "สอบตก: #{failed.map { |s| s[:name] }.join(", ")}"
# => สอบผ่าน: สมชาย, มานี
# => สอบตก: สมหญิง, วิชัย
```

### `partition` — ได้ทั้งสองกลุ่มในครั้งเดียว (ไม่ต้องวน loop สองรอบ)

```ruby
students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 45 },
  { name: "มานี", score: 78 },
  { name: "วิชัย", score: 30 }
]

passed, failed = students.partition { |s| s[:score] >= 50 }
p passed.map { |s| s[:name] }  # => ["สมชาย", "มานี"]
p failed.map { |s| s[:name] }  # => ["สมหญิง", "วิชัย"]
```

**อธิบาย:** `partition` แบ่ง array ออกเป็น**สอง array** ตามเงื่อนไขใน block คืนค่าเป็น
Array ของ Array สองตัว `[กลุ่มที่ผ่าน, กลุ่มที่ไม่ผ่าน]` ซึ่งใช้เทคนิค **multiple
assignment** (`passed, failed = ...`) แกะค่าออกมาเก็บในตัวแปรสองตัวพร้อมกัน — สะดวกกว่า
การเรียก `select` และ `reject` แยกกันสองครั้ง (ซึ่งจะวน loop ผ่าน array ถึงสองรอบ)

---

## Step 37: `reduce`/`inject` — พับข้อมูลทั้ง Array ให้เหลือค่าเดียว

`reduce` (มีชื่อพ้อง `inject` ทำงานเหมือนกันทุกประการ) คือ method ที่ทรงพลังที่สุดตัวหนึ่ง
ใน Enumerable ใช้สำหรับ "พับ" (fold) สมาชิกทั้งหมดใน Array ให้เหลือเป็น**ค่าเดียว** เช่น
ผลรวม, ผลคูณ, ค่าสูงสุด หรือแม้แต่ Hash/Array ที่ประกอบขึ้นใหม่

### รูปแบบที่ 1: symbol shorthand (ใช้เมื่อ operation ง่ายๆ)

```ruby
numbers = [1, 2, 3, 4, 5]

sum = numbers.reduce(:+)
p sum  # => 15   (1 + 2 + 3 + 4 + 5)

product = numbers.reduce(:*)
p product  # => 120  (1 * 2 * 3 * 4 * 5)

# inject เป็นชื่อพ้องของ reduce เขียนแบบนี้ก็ได้ผลลัพธ์เดียวกัน
p numbers.inject(:+)  # => 15
```

**อธิบายกลไก:** `reduce(:+)` บอกให้ Ruby เอาสมาชิกสองตัวแรกมาบวกกัน (`+`) ได้ผลลัพธ์
ชั่วคราว แล้วเอาผลลัพธ์นั้นไปบวกกับสมาชิกตัวถัดไปเรื่อยๆ จนครบทุกตัว เทียบเท่ากับ
`((((1 + 2) + 3) + 4) + 5)`

### รูปแบบที่ 2: กำหนดค่าเริ่มต้น (initial value) + symbol

```ruby
numbers = [1, 2, 3, 4, 5]

# กำหนดค่าเริ่มต้นเป็น 100 ก่อนเริ่มบวก
sum_with_initial = numbers.reduce(100, :+)
p sum_with_initial  # => 115  (100 + 1 + 2 + 3 + 4 + 5)
```

การมีค่าเริ่มต้นสำคัญมากเมื่อ Array อาจว่างเปล่า:

```ruby
empty = []
p empty.reduce(:+)      # => nil   !! (ไม่มีสมาชิกให้บวก จึงไม่มีอะไรให้ return)
p empty.reduce(0, :+)    # => 0    (ปลอดภัยกว่า เพราะมีค่าเริ่มต้นเป็นฐานเสมอ)
```

### รูปแบบที่ 3: explicit block (ใช้เมื่อ logic ซับซ้อนกว่าการบวก/คูณธรรมดา)

```ruby
numbers = [1, 2, 3, 4, 5]

# block รับ 2 parameter: (accumulator, current_element)
sum = numbers.reduce(0) { |accumulator, n| accumulator + n }
p sum  # => 15

# accumulator คือค่าที่สะสมมาจากรอบก่อนหน้า (เริ่มต้นจาก initial value ที่กำหนด)
max_value = numbers.reduce { |max, n| n > max ? n : max }
p max_value  # => 5  (เทียบเท่ากับ numbers.max แต่เขียนเองด้วย reduce)
```

### ตัวอย่างการใช้งานจริง: reduce กับ Hash เพื่อนับความถี่

```ruby
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]

word_counts = words.reduce(Hash.new(0)) do |counts, word|
  counts[word] += 1
  counts   # สำคัญมาก: ต้อง return accumulator กลับไปทุกรอบ ไม่งั้นรอบถัดไปจะพัง!
end

p word_counts  # => {"apple"=>3, "banana"=>2, "cherry"=>1}
```

**คำเตือนสำคัญที่สุดของ `reduce` แบบ block:** block ของ `reduce` **ต้อง return ค่า
accumulator กลับไปเสมอ** เป็นบรรทัดสุดท้ายของ block เพราะค่าที่ return จากรอบนี้จะกลายเป็น
`accumulator` ของรอบถัดไป ถ้าลืม return (เช่น บรรทัดสุดท้ายเป็นอย่างอื่น) ผลลัพธ์จะผิดทันที

```ruby
# ตัวอย่างข้อผิดพลาดที่พบบ่อย: ลืม return accumulator
broken = words.reduce(Hash.new(0)) do |counts, word|
  counts[word] += 1
  puts "กำลังนับ #{word}"    # <- บรรทัดสุดท้ายกลายเป็น puts ซึ่ง return nil เสมอ!
end
# บรรทัดต่อไปจะ error ทันที เพราะ accumulator กลายเป็น nil ตั้งแต่รอบที่ 2:
# NoMethodError: undefined method `[]' for nil
```

> เมื่อเทียบกับ `each_with_object` ในหัวข้อก่อน: **ใช้ `reduce` เมื่อ accumulator เป็นค่า
> ธรรมดาที่สร้างใหม่ทุกรอบได้ (เช่น ตัวเลข, string)** และ **ใช้ `each_with_object` เมื่อ
> accumulator เป็น object ที่ mutate ได้โดยตรง (เช่น Hash, Array) และไม่อยาก return ซ้ำ
> ทุกรอบ** — ทั้งสองวิธีแก้ปัญหาการนับความถี่ข้างต้นได้เหมือนกัน แต่ `each_with_object`
> อ่านง่ายกว่าเล็กน้อยเมื่อ accumulator เป็น Hash/Array

---

## Step 38: `find`/`detect`, `all?`/`any?`/`none?`/`count`

### `find`/`detect` — หาสมาชิกตัวแรกที่ตรงเงื่อนไข

```ruby
numbers = [1, 3, 5, 8, 9, 12]

first_even = numbers.find { |n| n.even? }
p first_even  # => 8  (หยุดค้นหาทันทีที่เจอตัวแรกที่ผ่านเงื่อนไข ไม่วนต่อ)

# detect เป็นชื่อพ้องของ find ทำงานเหมือนกันเป๊ะๆ
p numbers.detect { |n| n > 10 }  # => 12

# ถ้าไม่มีสมาชิกใดตรงเงื่อนไขเลย คืนค่า nil (ไม่ error)
p numbers.find { |n| n > 100 }  # => nil
```

**ข้อแตกต่างสำคัญจาก `select`:** `select` จะวนผ่านทุกสมาชิกและคืน**ทุกตัว**ที่ตรงเงื่อนไข
เป็น array ส่วน `find` จะ**หยุดทันที**ที่เจอตัวแรกที่ตรงเงื่อนไข และคืน**สมาชิกตัวนั้นตัว
เดียว** (ไม่ใช่ array) — ถ้าต้องการแค่ "มีไหม" หรือ "ตัวแรกคืออะไร" ใช้ `find` จะเร็วกว่า
`select` มากในกรณี array ขนาดใหญ่ เพราะไม่ต้องวนจนครบทุกตัว

```ruby
students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 92 },
  { name: "มานี", score: 78 }
]

top_student = students.find { |s| s[:score] > 90 }
p top_student  # => {name: "สมหญิง", score: 92}
```

### `all?`, `any?`, `none?` — ตรวจสอบเงื่อนไขของทั้ง Array

```ruby
scores = [65, 78, 82, 90, 55]

# all? -> true ถ้าทุกสมาชิก "ผ่าน" เงื่อนไข (ถ้า array ว่างเปล่า all? จะคืน true เสมอ)
p scores.all? { |s| s >= 50 }   # => true   (ทุกคนสอบผ่านเกณฑ์ 50)
p scores.all? { |s| s >= 60 }   # => false  (มี 55 ที่ไม่ผ่าน)

# any? -> true ถ้ามีสมาชิกอย่างน้อยหนึ่งตัวผ่านเงื่อนไข
p scores.any? { |s| s >= 90 }   # => true   (มี 90 อยู่)
p scores.any? { |s| s >= 100 }  # => false  (ไม่มีใครได้ 100 ขึ้นไป)

# none? -> true ถ้าไม่มีสมาชิกใดเลยที่ผ่านเงื่อนไข (ตรงข้ามกับ any?)
p scores.none? { |s| s < 0 }    # => true   (ไม่มีคะแนนติดลบ)
p scores.none? { |s| s < 60 }   # => false  (มี 55 ซึ่งน้อยกว่า 60)

# ทั้งสาม method ใช้แบบไม่มี block ได้ด้วย -> เช็คว่า "เป็น truthy" ของแต่ละสมาชิกเอง
p [1, 2, 3].all?           # => true  (ทุกตัวเป็น truthy)
p [1, nil, 3].all?          # => false (nil เป็น falsy)
p [nil, false].any?         # => false (ไม่มีตัวไหนเป็น truthy เลย)
```

### `count` — นับจำนวนสมาชิก (ทั้งหมด, ที่ตรงเงื่อนไข, หรือที่เท่ากับค่าที่กำหนด)

```ruby
scores = [65, 78, 82, 90, 55, 90]

p scores.count             # => 6           (จำนวนสมาชิกทั้งหมด เหมือน .length/.size)
p scores.count(90)          # => 2           (นับว่ามีค่า 90 กี่ตัว)
p scores.count { |s| s >= 80 }  # => 2       (นับตามเงื่อนไขใน block)
```

### สรุปเปรียบเทียบ method กลุ่มนี้

```ruby
numbers = [2, 4, 6, 8]

p numbers.all? { |n| n.even? }    # => true  (ทุกตัวเป็นเลขคู่)
p numbers.any? { |n| n.odd? }     # => false (ไม่มีเลขคี่เลย)
p numbers.none? { |n| n.odd? }    # => true  (ยืนยันว่าไม่มีเลขคี่เลย)
p numbers.find { |n| n > 5 }      # => 6     (ตัวแรกที่มากกว่า 5)
p numbers.count { |n| n > 5 }     # => 2     (จำนวนตัวที่มากกว่า 5 คือ 6 กับ 8)
p numbers.select { |n| n > 5 }    # => [6, 8] (ตัวที่มากกว่า 5 ทั้งหมด เป็น array)
```

---

## Step 39: `flatten`, `zip`, `sort_by`

### `flatten` — แปลง Array ซ้อน Array ให้แบนราบ

```ruby
nested = [1, [2, 3], [4, [5, 6]], 7]

p nested.flatten     # => [1, 2, 3, 4, 5, 6, 7]  (แบนราบทุกชั้นโดย default)
p nested.flatten(1)  # => [1, 2, 3, 4, [5, 6], 7] (แบนราบแค่ 1 ชั้น ระบุความลึกได้)

p nested   # => [1, [2, 3], [4, [5, 6]], 7]  <- flatten ไม่ mutate ต้นฉบับ

# flatten! สำหรับ mutate ต้นฉบับ (คืน nil ถ้า array แบนราบอยู่แล้ว เหมือนกฎ ! ทั่วไป)
arr = [1, [2, 3]]
arr.flatten!
p arr  # => [1, 2, 3]
```

### `zip` — จับคู่สมาชิกจาก Array หลายตัวตามตำแหน่ง

```ruby
names = ["สมชาย", "สมหญิง", "มานี"]
scores = [85, 92, 78]

paired = names.zip(scores)
p paired
# => [["สมชาย", 85], ["สมหญิง", 92], ["มานี", 78]]

# zip กับหลาย array พร้อมกันได้ (ไม่จำกัดแค่ 2 ตัว)
subjects = ["คณิต", "วิทย์", "ไทย"]
grades = ["A", "B", "A"]
combined = names.zip(scores, grades)
p combined
# => [["สมชาย", 85, "A"], ["สมหญิง", 92, "B"], ["มานี", 78, "A"]]

# ถ้า array ที่ zip เข้ามามีขนาดสั้นกว่า จะเติม nil ในตำแหน่งที่ขาด
short = [1, 2]
long = [10, 20, 30, 40]
p short.zip(long)  # => [[1, 10], [2, 20]]        (ตัดตาม array หลัก ไม่เกิน 2 คู่)
p long.zip(short)  # => [[10, 1], [20, 2], [30, nil], [40, nil]]  (เติม nil ให้ที่ขาด)
```

**การใช้งานจริง:** `zip` มีประโยชน์มากเมื่อมีข้อมูลหลายชุดที่ index สอดคล้องกัน (มาจาก
คนละแหล่ง เช่น อ่านจากไฟล์ CSV คนละคอลัมน์) แล้วต้องการรวมเป็นชุดข้อมูลเดียวกัน สามารถ
ต่อกับ `each` หรือ `map` เพื่อประมวลผลต่อได้ทันที

```ruby
names.zip(scores).each do |name, score|
  puts "#{name}: #{score} คะแนน"
end
# => สมชาย: 85 คะแนน
# => สมหญิง: 92 คะแนน
# => มานี: 78 คะแนน
```

### `sort_by` — เรียงลำดับตาม key ที่คำนวณจาก block

`sort` ธรรมดาต้องอาศัยการเปรียบเทียบ (`<=>`) ระหว่างสมาชิกโดยตรง ซึ่งใช้ยากเมื่อสมาชิก
เป็น Hash หรือ Object ที่ซับซ้อน `sort_by` แก้ปัญหานี้โดยให้ระบุ **key สำหรับใช้เรียง**
ผ่าน block แทน

```ruby
students = [
  { name: "สมชาย", score: 85 },
  { name: "สมหญิง", score: 92 },
  { name: "มานี", score: 78 }
]

# sort_by: เรียงจากน้อยไปมากตาม score
by_score = students.sort_by { |s| s[:score] }
p by_score.map { |s| s[:name] }  # => ["มานี", "สมชาย", "สมหญิง"]

# เรียงจากมากไปน้อย: ใส่เครื่องหมายลบหน้าค่าตัวเลข (trick ที่ใช้บ่อยมาก)
by_score_desc = students.sort_by { |s| -s[:score] }
p by_score_desc.map { |s| s[:name] }  # => ["สมหญิง", "สมชาย", "มานี"]

# หรือใช้ .reverse ต่อท้ายก็ได้ผลเดียวกัน (อ่านง่ายกว่าเมื่อ key ไม่ใช่ตัวเลข)
by_score_desc2 = students.sort_by { |s| s[:score] }.reverse
p by_score_desc2.map { |s| s[:name] }  # => ["สมหญิง", "สมชาย", "มานี"]

# sort_by เทียบกับ sort แบบ block ธรรมดา (ต้องเขียน <=> เอง ยุ่งยากกว่า)
by_score_manual = students.sort { |a, b| a[:score] <=> b[:score] }
p by_score_manual.map { |s| s[:name] }  # => ["มานี", "สมชาย", "สมหญิง"]  (ผลลัพธ์เหมือนกัน)
```

**ทำไม `sort_by` มักดีกว่า `sort` แบบ block:** `sort_by` คำนวณ key ของแต่ละสมาชิกเพียง
**ครั้งเดียว** แล้วเก็บไว้เปรียบเทียบ (เทคนิคที่เรียกว่า Schwartzian Transform) ในขณะที่
`sort` แบบ block ต้องเรียก block ซ้ำๆ ทุกครั้งที่เปรียบเทียบคู่ ทำให้ `sort_by` เร็วกว่า
อย่างชัดเจนเมื่อการคำนวณ key มีต้นทุนสูง (เช่น ต้องเรียก method หลายชั้น) และย่านอ่านง่าย
กว่าเสมอ

```ruby
# เรียงตามหลาย key พร้อมกัน: ใช้ array เป็น key เพื่อเรียงตาม key แรกก่อน ถ้าเท่ากันค่อยดู key ถัดไป
data = [
  { dept: "IT", name: "สมชาย" },
  { dept: "HR", name: "มานี" },
  { dept: "IT", name: "อรทัย" }
]

sorted = data.sort_by { |d| [d[:dept], d[:name]] }
p sorted.map { |d| "#{d[:dept]}-#{d[:name]}" }
# => ["HR-มานี", "IT-สมชาย", "IT-อรทัย"]  (เรียงตาม dept ก่อน แล้วค่อยเรียงตามชื่อ)
```

---

## Step 40: แบบฝึกหัดโปรเจกต์ — วิเคราะห์คะแนนนักเรียน

### โจทย์

เขียนโปรแกรม `grade_report.rb` ที่มีข้อมูลนักเรียนเป็น Array ของ Hash ดังนี้:

```ruby
students = [
  { name: "สมชาย", scores: [80, 75, 90] },
  { name: "สมหญิง", scores: [95, 88, 92] },
  { name: "มานี", scores: [40, 55, 45] },
  { name: "วิชัย", scores: [60, 58, 70] },
  { name: "อรทัย", scores: [30, 45, 35] }
]
```

โปรแกรมต้องทำสิ่งต่อไปนี้ (เกณฑ์สอบผ่าน: คะแนนเฉลี่ย >= 50):

1. คำนวณคะแนนเฉลี่ย (average) ของนักเรียนแต่ละคน
2. หานักเรียนที่ได้คะแนนเฉลี่ยสูงสุด (top student)
3. แบ่งกลุ่มนักเรียนเป็น "สอบผ่าน" กับ "สอบตก" ตามเกณฑ์คะแนนเฉลี่ย
4. แสดงรายงานสรุปทั้งหมดอย่างเป็นระเบียบ เรียงจากคะแนนเฉลี่ยมากไปน้อย

### เฉลย

```ruby
# frozen_string_literal: true

# grade_report.rb
students = [
  { name: "สมชาย", scores: [80, 75, 90] },
  { name: "สมหญิง", scores: [95, 88, 92] },
  { name: "มานี", scores: [40, 55, 45] },
  { name: "วิชัย", scores: [60, 58, 70] },
  { name: "อรทัย", scores: [30, 45, 35] }
]

PASSING_AVERAGE = 50

# Step 1: คำนวณคะแนนเฉลี่ยของนักเรียนแต่ละคน แล้วเพิ่ม key :average เข้าไปใน hash เดิม
# ใช้ map เพราะต้องการ array ใหม่ที่มีข้อมูลเพิ่มเติม (ไม่แก้ students ต้นฉบับตรงๆ)
students_with_average = students.map do |student|
  total = student[:scores].reduce(0, :+)
  average = total.to_f / student[:scores].size
  student.merge(average: average.round(2))
end

# Step 2: หานักเรียนที่คะแนนเฉลี่ยสูงสุด ด้วย max_by (Enumerable method ที่คล้าย sort_by
# แต่คืนแค่สมาชิกที่มี key สูงสุดตัวเดียว ไม่ต้องเรียงทั้ง array)
top_student = students_with_average.max_by { |s| s[:average] }

# Step 3: แบ่งกลุ่มสอบผ่าน/สอบตก ด้วย partition
passed, failed = students_with_average.partition { |s| s[:average] >= PASSING_AVERAGE }

# เรียงแต่ละกลุ่มจากคะแนนเฉลี่ยมากไปน้อยด้วย sort_by + ค่าติดลบ
passed_sorted = passed.sort_by { |s| -s[:average] }
failed_sorted = failed.sort_by { |s| -s[:average] }

# Step 4: แสดงรายงาน
puts "=" * 50
puts "รายงานผลการเรียน"
puts "=" * 50

puts "\n--- นักเรียนทั้งหมด (เรียงตามคะแนนเฉลี่ย) ---"
all_sorted = students_with_average.sort_by { |s| -s[:average] }
all_sorted.each_with_index do |student, index|
  status = student[:average] >= PASSING_AVERAGE ? "ผ่าน" : "ตก"
  puts "#{index + 1}. #{student[:name]}: เฉลี่ย #{student[:average]} คะแนน (#{status})"
end

puts "\n--- นักเรียนดีเด่น ---"
puts "#{top_student[:name]} ได้คะแนนเฉลี่ยสูงสุดที่ #{top_student[:average]} คะแนน"

puts "\n--- สรุปกลุ่มสอบผ่าน (#{passed_sorted.size} คน) ---"
passed_sorted.each { |s| puts "  #{s[:name]}: #{s[:average]}" }

puts "\n--- สรุปกลุ่มสอบตก (#{failed_sorted.size} คน) ---"
if failed_sorted.empty?
  puts "  (ไม่มีนักเรียนสอบตก)"
else
  failed_sorted.each { |s| puts "  #{s[:name]}: #{s[:average]}" }
end

# สถิติภาพรวมทั้งห้อง คำนวณด้วย reduce
class_average = students_with_average.reduce(0) { |sum, s| sum + s[:average] } / students_with_average.size.to_f
puts "\n--- สถิติภาพรวม ---"
puts "คะแนนเฉลี่ยทั้งห้อง: #{class_average.round(2)}"
puts "จำนวนนักเรียนทั้งหมด: #{students_with_average.size} คน"
puts "อัตราการสอบผ่าน: #{(passed_sorted.size.to_f / students_with_average.size * 100).round(1)}%"
```

ทดสอบรัน:

```bash
ruby grade_report.rb
```

ผลลัพธ์ที่ได้:

```
==================================================
รายงานผลการเรียน
==================================================

--- นักเรียนทั้งหมด (เรียงตามคะแนนเฉลี่ย) ---
1. สมหญิง: เฉลี่ย 91.67 คะแนน (ผ่าน)
2. สมชาย: เฉลี่ย 81.67 คะแนน (ผ่าน)
3. วิชัย: เฉลี่ย 62.67 คะแนน (ผ่าน)
4. มานี: เฉลี่ย 46.67 คะแนน (ตก)
5. อรทัย: เฉลี่ย 36.67 คะแนน (ตก)

--- นักเรียนดีเด่น ---
สมหญิง ได้คะแนนเฉลี่ยสูงสุดที่ 91.67 คะแนน

--- สรุปกลุ่มสอบผ่าน (3 คน) ---
  สมหญิง: 91.67
  สมชาย: 81.67
  วิชัย: 62.67

--- สรุปกลุ่มสอบตก (2 คน) ---
  มานี: 46.67
  อรทัย: 36.67

--- สถิติภาพรวม ---
คะแนนเฉลี่ยทั้งห้อง: 63.87
จำนวนนักเรียนทั้งหมด: 5 คน
อัตราการสอบผ่าน: 60.0%
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `student.merge(average: ...)` — สร้าง Hash ใหม่โดยรวม key เดิมกับ key ใหม่ที่เพิ่มเข้ามา
  (ไม่แก้ไข Hash เดิม เพราะ `merge` เป็น non-mutating method) จะอธิบาย Hash แบบละเอียดใน
  Part 005
- `max_by` — เหมือน `sort_by` แต่คืนสมาชิกที่มี key **สูงสุด**เพียงตัวเดียว (มี `min_by`
  คู่กันสำหรับหาค่าต่ำสุด) ประสิทธิภาพดีกว่าการ `sort_by(...).last` เพราะไม่ต้องเรียงทั้ง
  array
- การรวม `map` (แปลงข้อมูล), `partition` (แบ่งกลุ่ม), `sort_by` (เรียงลำดับ), `reduce`
  (คำนวณสรุป) เข้าด้วยกัน — นี่คือรูปแบบการเขียนโค้ดแบบ **functional pipeline** ที่พบได้
  ทั่วไปในโค้ด Ruby ระดับมืออาชีพ: แปลงข้อมูลผ่าน method หลายตัวต่อกันเป็นทอดๆ แทนการเขียน
  loop ยาวๆ ด้วยมือ

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่มการจัดอันดับเกรด (A/B/C/D/F) ให้แต่ละคนตามคะแนนเฉลี่ย โดยใช้เกณฑ์
   `>= 80` = A, `>= 70` = B, `>= 60` = C, `>= 50` = D, ต่ำกว่านั้น = F แล้วใช้
   `group_by` (ลองค้นดูเอกสาร method นี้ — ทำงานคล้าย `each_with_object` ที่จัดกลุ่มให้
   อัตโนมัติ) เพื่อสร้าง Hash ที่ key เป็นเกรด และ value เป็น array รายชื่อนักเรียนในเกรด
   นั้น
2. เขียนโปรแกรมแยกที่รับ Array ของราคาสินค้า (เช่น `[120, 350, 89, 500, 45]`) แล้วใช้
   `reduce`/`inject` หาผลรวมราคาทั้งหมด, ใช้ `select`/`count` หาว่ามีสินค้ากี่ชิ้นที่ราคา
   เกิน 100 บาท, และใช้ `sort` กับ `first`/`last` หาสินค้าที่ถูกที่สุดกับแพงที่สุด (ห้าม
   ใช้ `.min`/`.max` โดยตรง ให้ฝึกเขียนด้วย `sort` ก่อน แล้วค่อยลองเทียบกับการใช้
   `.min`/`.max` ว่าผลลัพธ์เหมือนกันไหม)
3. จำลองสถานการณ์ "ตะกร้าสินค้า" เป็น Array ของ Hash ที่มี `:name`, `:price`, `:quantity`
   เขียนโค้ดคำนวณราคารวมทั้งตะกร้า (ผลรวมของ `price * quantity` ทุกชิ้น) ด้วย `reduce` หรือ
   `sum` (ลองค้น method `sum` ของ Enumerable ที่รับ block ได้ ว่าทำงานคล้าย `reduce` ยังไง)
   จากนั้นทดลองสร้างบั๊ก aliasing ตามที่เรียนใน Step 33 โดยตั้งใจ (ส่ง array ตะกร้าเข้า
   method ที่แก้ไขมันโดยตรงด้วย `<<`) แล้วแก้บั๊กด้วย `dup`

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- สร้าง Array ได้หลายรูปแบบ (`[]`, `Array.new` พร้อม block, `%w`, `%i`) และรู้จักกับดักของ
  `Array.new(size, default_value)` เมื่อ default เป็น mutable object
- เข้าถึงข้อมูลด้วย index, negative index, slicing (`[start, length]`, Range) และ `dig`
  สำหรับโครงสร้างซ้อนกันอย่างปลอดภัย
- เข้าใจความแตกต่างระหว่าง mutating method (`push`, `<<`, `pop`, `shift`, `unshift`,
  `sort!`, `uniq!`) กับ non-mutating method (`sort`, `uniq`) และกฎทั่วไปของ method ที่มี `!`
- เข้าใจ **aliasing** อย่างลึกซึ้ง — ทำไมการกำหนดตัวแปรใหม่จาก Array เดิมไม่ใช่การ copy
  และวิธีใช้ `dup`/`clone`/`freeze` แก้ปัญหานี้ รวมถึงข้อจำกัดของ shallow copy
- ใช้ `each`, `each_with_index`, `each_with_object` วน loop ได้ตามสถานการณ์ที่เหมาะสม
- แยกแยะ `map` (แปลงข้อมูล คืน array ใหม่) จาก `each` (side effect คืน array เดิม) ได้ชัดเจน
- กรองข้อมูลด้วย `select`/`filter`, `reject`, และแบ่งกลุ่มด้วย `partition`
- ใช้ `reduce`/`inject` พับข้อมูลทั้ง array ให้เหลือค่าเดียว ทั้งแบบ symbol shorthand และ
  explicit block พร้อมข้อควรระวังเรื่องการ return accumulator
- ใช้ `find`/`detect` หาค่าตัวแรกที่ตรงเงื่อนไข และ `all?`/`any?`/`none?`/`count` ตรวจสอบ
  เงื่อนไขของทั้ง collection
- ใช้ `flatten` แบนโครงสร้างซ้อนกัน, `zip` จับคู่ข้อมูลจากหลาย array, และ `sort_by`
  เรียงลำดับข้อมูลซับซ้อนอย่างมีประสิทธิภาพ
- ประยุกต์ใช้ method เหล่านี้ร่วมกันแบบ pipeline เพื่อวิเคราะห์ข้อมูลจริง (คะแนนนักเรียน)

**ต่อไป (Part 005):** เราจะเจาะลึกเรื่อง **Hash** — การสร้าง Hash ทุกรูปแบบ, การวน loop
ผ่าน key-value, nested hash (Hash ซ้อน Hash/Array), และความแตกต่างที่สำคัญระหว่างการใช้
Symbol กับ String เป็น key ซึ่งเป็นพื้นฐานสำคัญที่จะใช้ตลอดทั้งหลักสูตร Rails
</content>
