# Part 013: Enumerable module ขั้นสูง — each_slice, group_by, partition, flat_map, lazy enumerator

> **Step ครอบคลุมใน Part นี้:** Step 121–130
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 004 (Array), Part 005 (Hash), และ Part 010 Step 99
> เรื่อง `Enumerable` เบื้องต้นมาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6)

Part นี้อยู่ใน **Phase 2: Ruby Deep Dive** ต่อจาก Part 012 เรื่อง File I/O, CSV, JSON, YAML
ใน Part 010 (Step 99) เราเห็นแล้วว่าแค่เขียน method `each` ตัวเดียวแล้ว `include Enumerable`
ก็ปลดล็อก `map`, `select`, `reduce` และเพื่อนๆ ให้ใช้งานฟรีทั้งหมด — แต่นั่นเป็นแค่ปลายยอด
ของภูเขาน้ำแข็ง `Enumerable` ยังมี method อีกจำนวนมากที่ทรงพลังมาก โดยเฉพาะกลุ่มที่ใช้จัดกลุ่ม
ข้อมูล (`group_by`, `partition`, `chunk_while`), กลุ่มที่ประมวลผลเป็นชุด (`each_slice`,
`each_cons`), และกลุ่มที่ทำงานแบบ **lazy evaluation** ซึ่งเปิดประตูสู่การจัดการข้อมูลขนาดใหญ่
หรือแม้แต่ข้อมูลที่ไม่มีที่สิ้นสุดโดยไม่ทำให้โปรแกรมค้างหรือหน่วยความจำเต็ม

Part นี้จะพาไปสำรวจ method เหล่านี้อย่างละเอียด พร้อมตัวอย่างการใช้งานจริงที่เจอบ่อยในโค้ด
production เช่น การแบ่งข้อมูลเป็นหน้า (pagination), การคำนวณค่าเฉลี่ยเคลื่อนที่ (moving
average), การจัดกลุ่มข้อมูลยอดขาย, และการประมวลผล stream ข้อมูลขนาดใหญ่แบบประหยัดหน่วยความจำ

## สารบัญของ Part นี้

- Step 121: ทบทวน Enumerable — `each` ตัวเดียว ปลดล็อกกว่า 60 method ได้อย่างไร
- Step 122: `each_slice` — แบ่งข้อมูลเป็นชุด (batching, pagination)
- Step 123: `each_cons` — หน้าต่างเลื่อน (sliding window) และค่าเฉลี่ยเคลื่อนที่
- Step 124: `group_by` — จัดกลุ่มข้อมูลเป็น Hash of Arrays
- Step 125: `partition` ทบทวน + การผสม `partition` กับ `group_by`
- Step 126: `flat_map` และความแตกต่างจาก `map` + `flatten`
- Step 127: `chunk_while` และ `slice_when` — จัดกลุ่มสมาชิกที่ต่อเนื่องกัน
- Step 128: `tally` — นับความถี่แบบไม่ต้องเขียน `reduce`/`Hash.new(0)` เอง
- Step 129: `min_by`, `max_by`, `minmax` — หาค่าสุดขั้วจาก key ที่คำนวณเอง
- Step 130: Enumerator และ Lazy Evaluation + แบบฝึกหัดปิด Part: วิเคราะห์ข้อมูลออเดอร์ขนาดใหญ่

---

## Step 121: ทบทวน Enumerable — `each` ตัวเดียว ปลดล็อกกว่า 60 method ได้อย่างไร

ใน Part 004 เราเรียน method ของ Array อย่าง `map`, `select`, `reduce`, `each_with_index`,
`find`, `all?`/`any?`/`none?`, `flatten`, `zip`, `sort_by` ไปแล้ว และใน Part 005 เราเรียน
การวนซ้ำ Hash ด้วย `each`, `each_pair`, `transform_values`/`transform_keys` — สิ่งที่หลายคน
อาจไม่ทันสังเกตคือ method เหล่านี้ **เกือบทั้งหมดไม่ได้เป็นของ `Array` หรือ `Hash` โดยตรง**
แต่มาจาก module `Enumerable` ที่ทั้ง `Array` และ `Hash` (และ `Range`, `Set` และอื่นๆ อีกมาก)
`include` เข้าไปเหมือนกัน — นี่คือเหตุผลที่ `map`/`select`/`reduce` ใช้ได้กับทั้ง Array, Hash,
Range โดยไม่ต้องเขียนแยกกันสามรอบ

Part 010 (Step 99) แสดงให้เห็นแล้วว่าเราสร้าง class ของตัวเองที่ใช้ `map`/`select`/`sort`/
`include?` ได้ทันทีโดยไม่ต้องเขียนเอง เพียงแค่นิยาม `each` ตัวเดียวแล้ว `include Enumerable`
ทบทวนตัวอย่าง `Bookshelf` จาก Step 99 อีกครั้ง เพราะเราจะใช้หลักการเดียวกันนี้ต่อยอดตลอด
Part นี้:

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

  # ต้องนิยาม each เพียง method เดียวเท่านั้น — ทุก method ที่เหลือมาจาก Enumerable
  def each
    return enum_for(:each) unless block_given?

    @books.each { |book| yield book }
  end
end

shelf = Bookshelf.new
shelf.add("Ruby").add("Rails").add("Clean Code")
```

มาดูกันว่า `Enumerable` ให้ method มาทั้งหมดกี่ตัว:

```ruby
p Enumerable.instance_methods(false).size
# => 61  (ใน Ruby 3.3.6 — ตัวเลขอาจขยับขึ้นเล็กน้อยในเวอร์ชันถัดๆ ไป แต่ไม่เคยน้อยลง)

p Enumerable.instance_methods(false).sort.first(10)
# => [:all?, :any?, :chain, :chunk, :chunk_while, :collect, :collect_concat,
#     :compact, :count, :cycle]
```

**อธิบาย:** ตัวเลข 61 นี้คือจำนวน method ที่ `Bookshelf` ของเราได้มา**ฟรี**ทั้งหมด โดยเขียนโค้ด
เองแค่ 4 บรรทัดใน `each` — เราใช้ไปแล้วแค่หยิบมือเดียวใน Step 99 (`map`, `select`, `sort`,
`include?`) ที่เหลือใน Part นี้เราจะไปสำรวจกลุ่มที่ทรงพลังและใช้บ่อยที่สุดในงานจริงที่ยังไม่เคย
แตะ:

```ruby
puts shelf.each_slice(2).to_a.inspect     # => [["Ruby", "Rails"], ["Clean Code"]]
puts shelf.min_by(&:length)                # => Ruby
puts shelf.max_by(&:length)                # => Clean Code
puts shelf.tally.inspect                   # => {"Ruby"=>1, "Rails"=>1, "Clean Code"=>1}
```

สังเกตว่าทั้งสี่บรรทัดนี้ทำงานได้ทันทีบน `Bookshelf` ทั้งที่มันไม่ใช่ Array เลยแม้แต่น้อย —
นี่คือพลังของการออกแบบผ่าน mixin ที่เรียนใน Part 010: เขียน interface เดียว (`each`) แล้วได้
พฤติกรรมนับสิบมาฟรี

> **สำคัญ:** `Array`, `Hash`, `Range`, `Set` (และ `ActiveRecord::Relation` ที่จะเจอใน Rails)
> ล้วน `include Enumerable` เหมือนกันทั้งหมด method ทุกตัวที่เรียนใน Part นี้จึงใช้ได้กับทั้ง
> Array, Hash, Range และ custom class ของเราเองแบบเดียวกันเป๊ะ — ตัวอย่างส่วนใหญ่ใน Part นี้
> จะใช้ Array เพราะเห็นภาพง่ายที่สุด แต่หลักการเดียวกันนำไปใช้กับ Hash หรือ class ของตัวเองได้
> ทันที

```ruby
# ตัวอย่าง: group_by ใช้กับ Hash ได้เหมือนกัน (จัดกลุ่มคู่ key-value ตามเงื่อนไขจาก value)
scores = { alice: 85, bob: 40, carol: 92, dave: 55 }
p scores.group_by { |_name, score| score >= 50 }
# => {true=>[[:alice, 85], [:carol, 92], [:dave, 55]], false=>[[:bob, 40]]}
```

---

## Step 122: `each_slice` — แบ่งข้อมูลเป็นชุด (batching, pagination)

`each_slice(n)` แบ่งสมาชิกทั้งหมดออกเป็นกลุ่มย่อยกลุ่มละ `n` ตัว เรียงตามลำดับเดิม (กลุ่ม
สุดท้ายอาจมีสมาชิกน้อยกว่า `n` ถ้าหารไม่ลงตัว) เป็น method ที่ใช้บ่อยมากเมื่อต้องประมวลผล
ข้อมูลจำนวนมากเป็น "ชุด" (batch) แทนที่จะทำทีละตัวหรือทั้งหมดพร้อมกัน

```ruby
# frozen_string_literal: true

items = (1..23).to_a

batches = items.each_slice(5).to_a
p batches
# => [[1, 2, 3, 4, 5], [6, 7, 8, 9, 10], [11, 12, 13, 14, 15],
#     [16, 17, 18, 19, 20], [21, 22, 23]]

p batches.size   # => 5   (23 ตัว หารเป็นกลุ่มละ 5 ได้ 5 กลุ่ม กลุ่มสุดท้ายเหลือ 3 ตัว)
```

**อธิบาย:**

- `each_slice(5)` เมื่อเรียกโดยไม่มี block จะคืน **Enumerator** (ยังไม่ได้ประมวลผลจริง) ต้อง
  ต่อ `.to_a` เพื่อบังคับให้กลายเป็น Array จริงๆ (เรื่อง Enumerator เจาะลึกใน Step 129)
- ถ้าเรียกพร้อม block จะวน loop ทันทีโดยไม่ต้องเรียก `.to_a`:

```ruby
items.each_slice(10) { |batch| puts "ประมวลผลชุดละ #{batch.size} รายการ" }
# => ประมวลผลชุดละ 10 รายการ
# => ประมวลผลชุดละ 10 รายการ
# => ประมวลผลชุดละ 3 รายการ
```

### กรณีใช้งานจริงที่ 1: Batch Processing

เวลาต้องประมวลผลข้อมูลจำนวนมาก (เช่น ส่งอีเมลหา user 100,000 คน หรือ insert ข้อมูลลง
database) การทำทีละตัวช้าเกินไป แต่การทำทั้งหมดในครั้งเดียวก็อาจกินหน่วยความจำหรือ timeout
ได้ `each_slice` ช่วยแบ่งงานเป็นก้อนขนาดที่จัดการได้พอดี

```ruby
user_ids = (1..10_547).to_a

user_ids.each_slice(500) do |batch|
  puts "ส่งอีเมลให้ user ID #{batch.first}–#{batch.last} (#{batch.size} คน)"
  # ในโค้ดจริง: UserMailer ส่งแบบ bulk ทีละ batch แทนการยิงทีละคน
  # หรือ database insert ทีละ batch แทนการ insert ทีละแถว (เร็วกว่ามาก)
end
# => ส่งอีเมลให้ user ID 1–500 (500 คน)
# => ส่งอีเมลให้ user ID 501–1000 (500 คน)
# => ... (ต่อไปเรื่อยๆ)
# => ส่งอีเมลให้ user ID 10501–10547 (47 คน)
```

> **preview Rails:** ActiveRecord มี method `find_each` และ `in_batches` ที่ทำงานบนหลักการ
> เดียวกับ `each_slice` เป๊ะๆ (`User.find_each(batch_size: 500) { |user| ... }`) เพื่อดึงข้อมูล
> จาก database ทีละก้อนแทนที่จะโหลดทั้งตารางเข้าหน่วยความจำในครั้งเดียว — เราจะเรียนละเอียด
> ใน Phase 4

### กรณีใช้งานจริงที่ 2: Pagination

`each_slice` ยังใช้ทำ pagination (แบ่งหน้า) แบบง่ายๆ ได้ทันที โดยที่แต่ละ "หน้า" ก็คือหนึ่งชุด
ของ `each_slice` นั่นเอง

```ruby
def paginate(items, page:, per_page:)
  pages = items.each_slice(per_page).to_a
  pages[page - 1] || []   # page เริ่มนับจาก 1 แต่ index ของ array เริ่มจาก 0
end

products = (1..47).map { |i| "สินค้า ##{i}" }

p paginate(products, page: 1, per_page: 10)
# => ["สินค้า #1", ..., "สินค้า #10"]

p paginate(products, page: 5, per_page: 10)
# => ["สินค้า #41", "สินค้า #42", ..., "สินค้า #47"]  (หน้าสุดท้ายมีแค่ 7 ตัว)

p paginate(products, page: 99, per_page: 10)
# => []  (หน้าที่ไม่มีอยู่จริง คืน array ว่างอย่างปลอดภัย)
```

> **ข้อควรระวังเรื่องประสิทธิภาพ:** วิธีนี้เหมาะกับข้อมูลที่**อยู่ใน memory แล้ว** (เช่น
> ผลลัพธ์จาก API หรือ array ที่คำนวณเสร็จแล้ว) เพราะ `each_slice(...).to_a` ต้องสร้าง Array
> ของทุกหน้าขึ้นมาก่อนเสมอ ถ้าข้อมูลมาจาก database โดยตรง ควรใช้ pagination gem อย่าง Kaminari
> หรือ Pagy ที่ทำ `LIMIT`/`OFFSET` ในระดับ SQL แทน (จะเรียนใน Phase 4 Part 038) ซึ่งไม่ต้องโหลด
> ข้อมูลทั้งหมดเข้า memory ก่อน

---

## Step 123: `each_cons` — หน้าต่างเลื่อน (sliding window) และค่าเฉลี่ยเคลื่อนที่

`each_cons(n)` คล้าย `each_slice(n)` แต่ต่างกันตรงที่ **หน้าต่างเลื่อนทีละ 1 ตำแหน่ง และซ้อนทับ
กันได้** (ไม่ใช่แบ่งเป็นชุดที่แยกขาดจากกัน) เหมาะสำหรับงานที่ต้องเปรียบเทียบ/คำนวณจากสมาชิกที่
"อยู่ติดกัน"

```ruby
# frozen_string_literal: true

prices = [100, 102, 101, 105, 110, 108, 112, 115]

pairs = prices.each_cons(2).to_a
p pairs
# => [[100, 102], [102, 101], [101, 105], [105, 110], [110, 108], [108, 112], [112, 115]]
```

**เปรียบเทียบกับ `each_slice`:**

```ruby
p prices.each_slice(2).to_a
# => [[100, 102], [101, 105], [110, 108], [112, 115]]   <- แบ่งเป็นชุด ไม่ซ้อนทับกัน (4 กลุ่ม)

p prices.each_cons(2).to_a
# => [[100, 102], [102, 101], [101, 105], ...]           <- เลื่อนทีละ 1 ซ้อนทับกัน (7 คู่)
```

`each_slice(8)` จาก array ขนาด 8 ตัวจะได้ 4 กลุ่ม (8 ÷ 2) ในขณะที่ `each_cons(2)` จะได้
`8 - 2 + 1 = 7` คู่ เพราะสมาชิกตัวกลางๆ ถูกใช้ซ้ำในสองคู่ (ทั้งเป็นตัวหลังของคู่ก่อน และตัวแรก
ของคู่ถัดไป)

### กรณีใช้งานจริงที่ 1: คำนวณผลต่างระหว่างวัน

```ruby
prices = [100, 102, 101, 105, 110, 108, 112, 115]

daily_changes = prices.each_cons(2).map { |prev, curr| curr - prev }
p daily_changes
# => [2, -1, 4, 5, -2, 4, 3]
```

**อธิบาย:** `each_cons(2)` คืนคู่ `[prev, curr]` ของราคาสองวันติดกัน แล้ว `map` แปลงแต่ละคู่
เป็นผลต่าง — เขียนแบบนี้กระชับกว่าการใช้ `each_with_index` แล้วเทียบกับ `index - 1` เอง
มากเพราะไม่ต้องกังวลเรื่อง index ติดลบตอนต้น array

### กรณีใช้งานจริงที่ 2: ค่าเฉลี่ยเคลื่อนที่ (Moving Average)

ค่าเฉลี่ยเคลื่อนที่เป็นเทคนิคที่ใช้บ่อยมากในการวิเคราะห์ข้อมูล time-series (ราคาหุ้น, ยอดขาย
รายวัน, จำนวนผู้ใช้งาน) เพื่อลด "สัญญาณรบกวน" และเห็นแนวโน้มที่ชัดเจนขึ้น

```ruby
prices = [100, 102, 101, 105, 110, 108, 112, 115]

moving_avg_3 = prices.each_cons(3).map { |window| (window.sum / window.size.to_f).round(2) }
p moving_avg_3
# => [101.0, 102.67, 105.33, 107.67, 110.0, 111.67]
```

**อธิบาย:** `each_cons(3)` สร้างหน้าต่างขนาด 3 วันเลื่อนไปเรื่อยๆ (`[100,102,101]`,
`[102,101,105]`, `[101,105,110]`, ...) แล้วคำนวณค่าเฉลี่ยของแต่ละหน้าต่าง — ผลลัพธ์ที่ได้มี
ขนาดเล็กกว่า array ต้นฉบับ `n - window_size + 1` ตัวเสมอ (ในที่นี้ 8 - 3 + 1 = 6 ค่า)

### กรณีใช้งานจริงที่ 3: ตรวจจับแนวโน้มต่อเนื่อง

```ruby
temperatures = [25, 26, 27, 26, 24, 23, 22, 24]

# หาช่วงที่อุณหภูมิลดลงติดต่อกัน 3 วันขึ้นไป
declining_start = temperatures.each_cons(2).each_with_index.find do |(prev, curr), _index|
  curr < prev
end
p declining_start
# => [[27, 26], 2]   (พบการลดลงครั้งแรกที่คู่ index 2 คือระหว่างวันที่ 3 กับ 4)
```

> **ข้อควรระวัง:** ถ้า `n` ที่ระบุใน `each_cons`/`each_slice` มากกว่าจำนวนสมาชิกทั้งหมดใน
> collection จะได้ผลลัพธ์เป็น array ว่าง ไม่ error:
>
> ```ruby
> p [1, 2].each_cons(5).to_a   # => []
> ```

---

## Step 124: `group_by` — จัดกลุ่มข้อมูลเป็น Hash of Arrays

ใน Part 005 Step 42 เราเคยใช้ `Hash.new { |hash, key| hash[key] = [] }` เพื่อจัดกลุ่มคำตาม
ความยาว และมีหมายเหตุทิ้งท้ายไว้ว่างานแบบนี้ Ruby มี method สำเร็จรูปให้ใช้ — นี่คือ method
นั้น: `group_by`

`group_by` วนผ่านทุกสมาชิก คำนวณ **key** จาก block ที่ให้มา แล้วจัดกลุ่มสมาชิกทั้งหมดที่ได้
key เดียวกันไว้ด้วยกัน ผลลัพธ์เป็น **Hash ที่ key คือค่าที่ block คำนวณได้ และ value คือ Array
ของสมาชิกทั้งหมดที่ตรงกับ key นั้น**

```ruby
# frozen_string_literal: true

words = %w[cat dog elephant ant bee lion tiger]

grouped = words.group_by(&:length)
p grouped
# => {3=>["cat", "dog", "ant", "bee"], 8=>["elephant"], 4=>["lion"], 5=>["tiger"]}
```

เทียบกับวิธีที่เขียนด้วย `Hash.new` + block จาก Part 005:

```ruby
# วิธีเดิมจาก Part 005 (ยาวกว่า)
grouped_manual = Hash.new { |hash, key| hash[key] = [] }
words.each { |word| grouped_manual[word.length] << word }
p grouped_manual   # ผลลัพธ์เหมือนกันทุกประการกับ group_by(&:length)
```

`group_by` ทำสิ่งเดียวกันในบรรทัดเดียว อ่านง่ายกว่า และสื่อเจตนา ("จัดกลุ่มตามความยาว") ชัดเจน
กว่ามาก

### กรณีใช้งานจริง: จัดกลุ่มข้อมูลออเดอร์

```ruby
Order = Struct.new(:id, :customer, :amount, :category)

orders = [
  Order.new(1, "สมชาย", 250, "อาหาร"),
  Order.new(2, "สมหญิง", 500, "เสื้อผ้า"),
  Order.new(3, "สมชาย", 100, "อาหาร"),
  Order.new(4, "มานี", 800, "อิเล็กทรอนิกส์"),
  Order.new(5, "สมหญิง", 150, "อาหาร"),
  Order.new(6, "สมชาย", 300, "เสื้อผ้า")
]

# จัดกลุ่มตามลูกค้า
by_customer = orders.group_by(&:customer)
by_customer.each { |customer, list| puts "#{customer}: #{list.size} รายการ" }
# => สมชาย: 3 รายการ
# => สมหญิง: 2 รายการ
# => มานี: 1 รายการ

# จัดกลุ่มตามหมวดหมู่ แล้วนับจำนวนแต่ละหมวด
by_category = orders.group_by(&:category)
p by_category.transform_values(&:size)
# => {"อาหาร"=>3, "เสื้อผ้า"=>2, "อิเล็กทรอนิกส์"=>1}
```

> **หมายเหตุ:** `Order` ในตัวอย่างนี้สร้างด้วย `Struct.new` ซึ่งเป็นวิธีสร้าง class ง่ายๆ ที่มี
> แค่ attribute ไม่กี่ตัวแบบรวดเร็ว โดยไม่ต้องเขียน `attr_accessor`/`initialize` เอง —
> เดี๋ยวจะเรียนละเอียดใน **Part 014 (Step 131–140)** ตรงนี้ขอยืมมาใช้ก่อนเพราะสะดวกสำหรับสร้าง
> ข้อมูลตัวอย่างที่มีโครงสร้างชัดเจน

`group_by` ไม่จำกัดแค่ key ที่เป็นค่าตรงๆ จาก attribute เดียว แต่คำนวณ key ด้วย logic อะไรก็ได้
ผ่าน block:

```ruby
# จัดกลุ่มตามช่วงราคา (คำนวณ key เองด้วย case/when)
by_range = orders.group_by do |order|
  case order.amount
  when 0..199 then :small
  when 200..499 then :medium
  else :large
  end
end

p by_range.transform_values { |list| list.map(&:id) }
# => {:medium=>[1, 6], :large=>[2, 4], :small=>[3, 5]}
```

**อธิบาย:** block ของ `group_by` รับสมาชิกแต่ละตัวแล้วต้อง**คืนค่า key** ที่จะใช้จัดกลุ่ม
— key นี้เป็นอะไรก็ได้ (Symbol, String, Integer, `true`/`false`, หรือแม้แต่ Array) ตราบใดที่
เปรียบเทียบความเท่ากันได้ `Order` สองตัวที่ block คำนวณ key ออกมาเหมือนกันจะถูกจัดไว้ด้วยกัน
เสมอ ไม่ว่า key นั้นจะมาจาก attribute ตรงๆ หรือคำนวณซับซ้อนแค่ไหนก็ตาม

### `group_by` บน Hash

เพราะ `Hash` ก็ `include Enumerable` เหมือนกัน `group_by` จึงใช้กับ Hash ได้เลย (block จะรับ
`[key, value]` เป็นคู่):

```ruby
scores = { alice: 85, bob: 40, carol: 92, dave: 55, eve: 30 }

by_pass = scores.group_by { |_name, score| score >= 50 }
p by_pass
# => {true=>[[:alice, 85], [:carol, 92], [:dave, 55]], false=>[[:bob, 40], [:eve, 30]]}
```

---

## Step 125: `partition` ทบทวน + การผสม `partition` กับ `group_by`

Part 004 (Step 36) แนะนำ `partition` ไปสั้นๆ แล้ว: มันแบ่งข้อมูลเป็น**สองกลุ่มเท่านั้น** ตาม
เงื่อนไข true/false คืนค่าเป็น Array ของ Array สองตัว `[กลุ่มที่ผ่าน, กลุ่มที่ไม่ผ่าน]`

```ruby
# frozen_string_literal: true

scores = [55, 82, 45, 90, 60, 38, 71]

passed, failed = scores.partition { |s| s >= 50 }
p passed   # => [55, 82, 90, 60, 71]
p failed   # => [45, 38]
```

**ความแตกต่างระหว่าง `partition` กับ `group_by` ที่ต้องแยกให้ออก:**

| | `partition` | `group_by` |
|---|---|---|
| จำนวนกลุ่มผลลัพธ์ | เสมอ 2 กลุ่ม (true/false) | กี่กลุ่มก็ได้ ตามจำนวน key ที่แตกต่างกัน |
| ชนิดผลลัพธ์ | Array ของ Array 2 ตัว `[[...], [...]]` | Hash `{key => [...], ...}` |
| block ต้องคืนค่า | truthy/falsy | ค่าอะไรก็ได้ที่ใช้เป็น key |
| เหมาะกับ | เงื่อนไข yes/no ชัดเจน (ผ่าน/ตก, active/inactive) | จัดหมวดหมู่หลายกลุ่ม (ตามจังหวัด, ตามหมวดสินค้า) |

พูดง่ายๆ: **`partition` คือ `group_by` แบบพิเศษที่ถูกจำกัดให้มีแค่ 2 กลุ่มเสมอ** ถ้าโจทย์คือ
"แบ่งเป็นสองฝั่ง" ให้ใช้ `partition` เพราะสื่อเจตนาชัดกว่าและได้ Array กลับมาตรงๆ (ไม่ต้องดึงผ่าน
key `true`/`false` ของ Hash) แต่ถ้าจำนวนกลุ่มไม่คงที่หรือมากกว่า 2 กลุ่ม ต้องใช้ `group_by`

### ผสม `partition` กับ `group_by` เข้าด้วยกัน

ในงานจริง เรามักต้องแบ่งข้อมูลตามเงื่อนไขหลักก่อน (partition) แล้วค่อยจัดกลุ่มย่อยภายในแต่ละ
ฝั่งอีกที (group_by) — ตัวอย่างเช่น แยกนักเรียนที่สอบผ่าน/ตกก่อน แล้วค่อยแบ่งกลุ่มคนที่ผ่านออก
เป็นระดับเกียรตินิยม/ผ่านธรรมดา

```ruby
students = [
  { name: "A", score: 85 },
  { name: "B", score: 40 },
  { name: "C", score: 92 },
  { name: "D", score: 55 },
  { name: "E", score: 30 }
]

passed, failed = students.partition { |s| s[:score] >= 50 }

# จัดกลุ่มย่อยเฉพาะฝั่งที่ผ่านแล้ว: เกียรตินิยม (>= 80) กับผ่านธรรมดา
grouped_passed = passed.group_by { |s| s[:score] >= 80 ? :distinction : :pass }
p grouped_passed.transform_values { |list| list.map { |s| s[:name] } }
# => {:distinction=>["A", "C"], :pass=>["D"]}

puts "สอบตก: #{failed.map { |s| s[:name] }.join(", ")}"
# => สอบตก: B, E
```

**ทำไมไม่ใช้ `group_by` เดี่ยวๆ 3 กลุ่มไปเลย:** ทำได้เช่นกัน (ดูด้านล่าง) แต่การแยก
`partition` ก่อนมีข้อดีตรงที่**สื่อเจตนาสองชั้นแยกกันชัดเจน** — ชั้นแรกคือ "ตัดสินผ่าน/ตก"
(logic ทางธุรกิจสำคัญ) ชั้นที่สองคือ "จัดระดับของคนที่ผ่าน" (รายละเอียดเสริม) การแยกสอง
ขั้นตอนทำให้โค้ดแต่ละส่วนอ่านและทดสอบแยกกันได้ง่ายกว่า:

```ruby
# แบบ group_by อย่างเดียว 3 กลุ่ม — ทำได้ในบรรทัดเดียว แต่ logic ปนกันหมด
bucketed = students.group_by do |s|
  if s[:score] < 50
    :fail
  elsif s[:score] < 80
    :pass
  else
    :distinction
  end
end
p bucketed.transform_values { |list| list.map { |s| s[:name] } }
# => {:distinction=>["A", "C"], :fail=>["B", "E"], :pass=>["D"]}
```

ทั้งสองวิธีให้ข้อมูลครบเหมือนกัน แต่เลือกใช้ตามว่าต้องการเน้นการแบ่งขั้นตอนความคิดแบบไหน —
ยิ่งเงื่อนไขแรก (ผ่าน/ตก) เป็นจุดตัดสินสำคัญที่ต้องใช้ต่อในหลายที่ของโปรแกรม ยิ่งควรแยก
`partition` ออกมาเป็นขั้นตอนของตัวเอง

---

## Step 126: `flat_map` และความแตกต่างจาก `map` + `flatten`

Part 004 (Step 35) แนะนำ `flat_map` ไว้สั้นๆ ว่าเป็น "`map` ผสม `flatten(1)`" ตอนนี้มาดูให้
ลึกและแม่นยำขึ้นว่า **แบนราบแค่ "1 ชั้น" เท่านั้น** หมายความว่าอย่างไรกันแน่ เพราะเป็นจุดที่
มือใหม่เข้าใจผิดบ่อย

```ruby
# frozen_string_literal: true

sentences = ["hello world", "ruby is fun"]

# ถ้าใช้ map เฉยๆ จะได้ array ซ้อน array (แต่ละประโยคกลายเป็น array ของคำ)
mapped = sentences.map { |s| s.split(" ") }
p mapped
# => [["hello", "world"], ["ruby", "is", "fun"]]

# flat_map แบนราบผลลัพธ์ให้เป็น array ชั้นเดียวให้อัตโนมัติ
flat = sentences.flat_map { |s| s.split(" ") }
p flat
# => ["hello", "world", "ruby", "is", "fun"]

# เทียบเท่ากับ map แล้วต่อ .flatten(1) เอง
p sentences.map { |s| s.split(" ") }.flatten(1)
# => ["hello", "world", "ruby", "is", "fun"]   (ผลลัพธ์เหมือนกัน)
```

### `flat_map` แบนราบแค่ 1 ชั้น ไม่ใช่ทุกชั้นเหมือน `flatten` เฉยๆ

นี่คือจุดสำคัญที่สุดที่ต้องเข้าใจ: `flat_map` เทียบเท่า `map` แล้ว `flatten(1)` (ระบุความลึก
`1` เสมอ) **ไม่ใช่** `flatten` แบบไม่ระบุ argument ที่แบนราบทุกชั้นจนสุด

```ruby
# block คืนค่าเป็น array ซ้อน array (2 ชั้น) ต่อไอเทม
p [1, 2].flat_map { |n| [[n, n]] }
# => [[1, 1], [2, 2]]   <- แบนราบไปแค่ "ชั้นนอกสุด" ที่ block คืนมา ไม่แตะชั้นในอีก

# เทียบกับ flatten แบบไม่ระบุความลึก ที่จะแบนราบทุกชั้นจนหมด
p [[1, [2, 3]], [4]].flatten
# => [1, 2, 3, 4]   <- แบนราบลึกสุด (deep flatten) ต่างจาก flat_map โดยสิ้นเชิง
```

**อธิบายกลไก:** `flat_map` เรียก block กับสมาชิกแต่ละตัวเหมือน `map` ทุกประการ แต่แทนที่จะ
เก็บผลลัพธ์แต่ละรอบเป็น **element เดียว** ของ array ผลลัพธ์ (แบบ `map` ธรรมดา) มันจะ
**concat (ต่อ) เนื้อหาของ array ที่ block คืนมาเข้ากับผลลัพธ์โดยตรง** — ถ้า block คืนค่าที่
ไม่ใช่ array (เช่น ตัวเลขธรรมดา) `flat_map` จะทำงานเหมือน `map` เป๊ะๆ ไม่มีอะไรให้แบน:

```ruby
p [1, 2, 3].flat_map { |n| n * 2 }        # => [2, 4, 6]  (เหมือน map ทุกประการ)
p [1, 2, 3].map { |n| n * 2 }              # => [2, 4, 6]  (ผลลัพธ์เดียวกัน)
```

### กรณีใช้งานจริง

```ruby
# ตัดคำจากหลายประโยคแล้วนับความยาวคำทั้งหมดในครั้งเดียว (ไม่ต้องแยก map แล้ว flatten เอง)
paragraphs = ["Ruby is elegant", "Rails is productive", "Testing matters"]

all_words = paragraphs.flat_map { |line| line.split(" ") }
p all_words
# => ["Ruby", "is", "elegant", "Rails", "is", "productive", "Testing", "matters"]

p all_words.tally.sort_by { |_word, count| -count }.first(3)
# => [["is", 2], ["Ruby", 1], ["elegant", 1]]

# ดึง tag ทั้งหมดจาก array ของ post ที่แต่ละ post มี tags เป็น array ย่อย
posts = [
  { title: "Post A", tags: ["ruby", "rails"] },
  { title: "Post B", tags: ["ruby", "testing"] },
  { title: "Post C", tags: [] }
]

all_tags = posts.flat_map { |post| post[:tags] }
p all_tags
# => ["ruby", "rails", "ruby", "testing"]

unique_tags = all_tags.uniq.sort
p unique_tags
# => ["rails", "ruby", "testing"]
```

**กฎการเลือกใช้:** ทุกครั้งที่เขียน `.map { ... }.flatten` (หรือ `.flatten(1)`) ต่อกัน ให้
เปลี่ยนเป็น `.flat_map { ... }` เสมอ — สั้นกว่า อ่านง่ายกว่า และไม่ต้องสร้าง array กลาง
(intermediate array) ก่อนแบนราบ จึงมีประสิทธิภาพดีกว่าเล็กน้อยด้วย

---

## Step 127: `chunk_while` และ `slice_when` — จัดกลุ่มสมาชิกที่ต่อเนื่องกัน

`group_by` จัดกลุ่มตาม **key ที่คำนวณได้** โดยไม่สนใจตำแหน่งในลิสต์ (สมาชิกที่ key เหมือนกัน
ถูกรวมกลุ่มแม้จะอยู่ห่างกันคนละที่) แต่บางครั้งเราต้องการจัดกลุ่มสมาชิกที่ **"ติดกัน" ในลำดับ
เดิม** เท่านั้น — นี่คือหน้าที่ของ `chunk_while` และ `slice_when` ซึ่งเป็นคู่ที่ทำงานตรงข้ามกัน
แต่ให้ผลลัพธ์เหมือนกันได้เมื่อกลับเงื่อนไข

### `chunk_while` — รวมกลุ่มต่อเมื่อเงื่อนไขระหว่างคู่ที่ติดกันเป็นจริง

```ruby
# frozen_string_literal: true

numbers = [1, 2, 4, 5, 6, 9, 10, 11, 20]

consecutive_groups = numbers.chunk_while { |a, b| b - a == 1 }.to_a
p consecutive_groups
# => [[1, 2], [4, 5, 6], [9, 10, 11], [20]]
```

**อธิบายกลไก:** `chunk_while` เรียก block กับ**คู่สมาชิกที่ติดกัน** (`a`, `b`) ไปเรื่อยๆ
ถ้า block คืนค่า **true** สมาชิกทั้งสองจะถูกจัดอยู่ **กลุ่มเดียวกัน** (รวมกลุ่มต่อไป) ถ้า
block คืนค่า **false** จะ**ตัดกลุ่มใหม่** ตรงนั้นทันที — ในตัวอย่างนี้ `b - a == 1` หมายถึง
"เป็นเลขที่ต่อเนื่องกัน (ห่างกัน 1)" จึงได้กลุ่มของเลขที่เรียงติดกัน

### `slice_when` — ตัดกลุ่มใหม่เมื่อเงื่อนไขเป็นจริง (ตรงข้ามความหมายกับ `chunk_while`)

```ruby
split_groups = numbers.slice_when { |a, b| b - a > 1 }.to_a
p split_groups
# => [[1, 2], [4, 5, 6], [9, 10, 11], [20]]   (ผลลัพธ์เหมือน chunk_while ด้านบนทุกประการ)
```

**อธิบาย:** `slice_when` ทำงานตรงข้ามกับ `chunk_while` ในเชิงความหมาย: `slice_when` จะ
**ตัดกลุ่มใหม่**ทุกครั้งที่ block คืนค่า **true** (แปลว่า "ตรงนี้ควรแยก") ในขณะที่
`chunk_while` จะ **รวมกลุ่มต่อ**ทุกครั้งที่ block คืนค่า true (แปลว่า "ตรงนี้ควรอยู่ด้วยกัน")
— นี่คือเหตุผลที่เงื่อนไขของทั้งสองในตัวอย่างข้างต้นเป็นตรงข้ามกันพอดี (`== 1` กับ `> 1`) แต่
ให้ผลลัพธ์เดียวกัน เลือกใช้ตัวไหนขึ้นกับว่าโจทย์ในหัวเราตั้งเป็นประโยคแบบ "ควรอยู่ด้วยกันเมื่อ
..." (ใช้ `chunk_while`) หรือ "ควรแยกกันเมื่อ ..." (ใช้ `slice_when`)

> **method ที่เกี่ยวข้อง:** `chunk` (ไม่มี `_while`) จัดกลุ่มสมาชิกที่ติดกัน**ตาม key ที่
> คำนวณจาก block เหมือน `group_by`** แต่หยุดรวมกลุ่มทันทีที่ key เปลี่ยน (ต่างจาก `group_by`
> ที่รวมทุกที่ที่ key เหมือนกันไม่ว่าจะอยู่ตำแหน่งไหน):
>
> ```ruby
> p [1, 1, 2, 2, 3, 1, 1].chunk { |n| n.odd? }.to_a
> # => [[true, [1, 1]], [false, [2, 2]], [true, [3, 1, 1]]]
> # สังเกตว่า 1 ตัวแรกกับ 1 ตัวท้าย (key เดียวกันคือ true) ไม่ถูกรวมกลุ่มเดียวกัน
> # เพราะมี false คั่นกลาง — นี่คือความต่างสำคัญจาก group_by
> ```

### กรณีใช้งานจริงที่ 1: ตรวจจับ Streak (ช่วงต่อเนื่อง) ของวันที่ login

```ruby
login_days = [1, 2, 3, 5, 6, 8, 9, 10, 15]   # เลขวันที่ในเดือน ที่ user login

streaks = login_days.chunk_while { |a, b| b - a == 1 }.to_a
p streaks
# => [[1, 2, 3], [5, 6], [8, 9, 10], [15]]

streaks.each { |s| puts "ติดต่อกัน #{s.size} วัน: วันที่ #{s.first}-#{s.last}" }
# => ติดต่อกัน 3 วัน: วันที่ 1-3
# => ติดต่อกัน 2 วัน: วันที่ 5-6
# => ติดต่อกัน 3 วัน: วันที่ 8-10
# => ติดต่อกัน 1 วัน: วันที่ 15-15

longest_streak = streaks.max_by(&:size)
puts "Streak ที่ยาวที่สุด: #{longest_streak.size} วัน"
# => Streak ที่ยาวที่สุด: 3 วัน
```

### กรณีใช้งานจริงที่ 2: แบ่งกลุ่มราคาตามช่องว่าง (price tiers)

```ruby
prices = [10, 12, 15, 50, 52, 55, 200, 210]

# ตัดกลุ่มใหม่ทุกครั้งที่ราคากระโดดห่างกันเกิน 20 บาท
tiers = prices.slice_when { |a, b| b - a > 20 }.to_a
p tiers
# => [[10, 12, 15], [50, 52, 55], [200, 210]]
```

โจทย์แบบนี้ตอบสนองด้วย `slice_when` ได้เป็นธรรมชาติกว่า เพราะประโยคในหัวคือ "ตัดกลุ่มใหม่เมื่อ
ราคากระโดดห่างเกินไป" ตรงกับความหมายของ `slice_when` ตรงๆ

---

## Step 128: `tally` — นับความถี่แบบไม่ต้องเขียน `reduce`/`Hash.new(0)` เอง

Part 004 (Step 37) และ Part 005 (Step 42) สอนวิธีนับความถี่ของสมาชิกด้วย `reduce` หรือ
`Hash.new(0)` มาแล้ว — `tally` (Ruby 2.7+) คือ method สำเร็จรูปที่ทำสิ่งเดียวกันในคำเดียว

```ruby
# frozen_string_literal: true

words = %w[apple banana apple cherry banana apple]

p words.tally
# => {"apple"=>3, "banana"=>2, "cherry"=>1}
```

เทียบกับวิธีเดิมจาก Part 004:

```ruby
# วิธีเดิมด้วย reduce + Hash.new(0) จาก Part 004 Step 37
word_counts = words.reduce(Hash.new(0)) do |counts, word|
  counts[word] += 1
  counts
end
p word_counts   # => {"apple"=>3, "banana"=>2, "cherry"=>1}   (ผลลัพธ์เดียวกันเป๊ะ)
```

`tally` ทำสิ่งเดียวกันในบรรทัดเดียว ไม่มีความเสี่ยงจะลืม return accumulator เหมือนที่เตือนไว้
ใน Part 004 (ปัญหาคลาสสิกของ `reduce`) — เมื่อโจทย์คือ "นับว่าแต่ละค่าปรากฏกี่ครั้ง" ตรงๆ ไม่มี
logic อื่นปนอยู่ ให้ใช้ `tally` เสมอ

### กรณีใช้งานจริง: หาผู้ชนะโหวต

```ruby
votes = ["A", "B", "A", "C", "B", "A", "A"]

result = votes.tally
p result
# => {"A"=>4, "B"=>2, "C"=>1}

winner = result.max_by { |_candidate, count| count }
p winner
# => ["A", 4]

puts "ผู้ชนะคือ #{winner[0]} ด้วยคะแนน #{winner[1]} เสียง"
# => ผู้ชนะคือ A ด้วยคะแนน 4 เสียง
```

### ผสม `tally` กับ `sort_by` เพื่อจัดอันดับ

```ruby
page_visits = %w[home home product home cart product home checkout product]

ranking = page_visits.tally.sort_by { |_page, count| -count }
p ranking
# => [["home", 4], ["product", 3], ["cart", 1], ["checkout", 1]]

ranking.each_with_index do |(page, count), index|
  puts "อันดับ #{index + 1}: #{page} (#{count} ครั้ง)"
end
# => อันดับ 1: home (4 ครั้ง)
# => อันดับ 2: product (3 ครั้ง)
# => อันดับ 3: cart (1 ครั้ง)
# => อันดับ 4: checkout (1 ครั้ง)
```

> **หมายเหตุเรื่องเวอร์ชัน:** `tally` ต้องการ Ruby 2.7 ขึ้นไป (หลักสูตรนี้ใช้ 3.3.x จึงไม่มี
> ปัญหา) ส่วน `tally_by` (รับ block เพื่อคำนวณ key ก่อนนับ คล้าย `group_by` + `tally`) เพิ่ง
> เพิ่มเข้ามาใน Ruby 3.4 ซึ่งใหม่กว่าเวอร์ชันที่หลักสูตรนี้ใช้ ถ้าต้องการผลแบบเดียวกันใน
> Ruby 3.3.x ให้ใช้ `group_by` แล้ว `transform_values(&:size)` แทน:
>
> ```ruby
> numbers = [1, 2, 3, 4, 5, 6]
> p numbers.group_by { |n| n.even? ? :even : :odd }.transform_values(&:size)
> # => {:odd=>3, :even=>3}
> ```

---

## Step 129: `min_by`, `max_by`, `minmax` — หาค่าสุดขั้วจาก key ที่คำนวณเอง

`min`/`max` ธรรมดา (จาก Part 004) เปรียบเทียบสมาชิกโดยตรงด้วย `<=>` — ใช้ได้ดีกับตัวเลขหรือ
String แต่ใช้ยากเมื่อสมาชิกเป็น Hash หรือ object ที่ซับซ้อน เหมือนกับที่ `sort_by` แก้ปัญหาของ
`sort` ไว้ใน Part 004 Step 39, `min_by`/`max_by` ก็แก้ปัญหาเดียวกันให้ `min`/`max`

```ruby
# frozen_string_literal: true

products = [
  { name: "หูฟัง", price: 990 },
  { name: "เมาส์", price: 350 },
  { name: "คีย์บอร์ด", price: 1200 },
  { name: "จอมอนิเตอร์", price: 4500 }
]

cheapest = products.min_by { |p| p[:price] }
priciest = products.max_by { |p| p[:price] }

p cheapest    # => {:name=>"เมาส์", :price=>350}
p priciest    # => {:name=>"จอมอนิเตอร์", :price=>4500}
```

**อธิบาย:** `min_by`/`max_by` คำนวณ key จาก block (เหมือน `sort_by`) แล้วเปรียบเทียบ key
เหล่านั้นแทนที่จะเปรียบเทียบ object ทั้งก้อนโดยตรง — เขียนแบบนี้อ่านง่ายกว่าการต้องนิยาม `<=>`
เองบน Hash มาก (หรือถ้าเป็น object ที่ยัง `include Comparable` ไม่ได้ ก็ยังใช้ได้ทันที)

### `minmax_by` — หาทั้ง min และ max ในการวนลิสต์ครั้งเดียว

```ruby
cheapest2, priciest2 = products.minmax_by { |p| p[:price] }
p [cheapest2[:name], priciest2[:name]]
# => ["เมาส์", "จอมอนิเตอร์"]
```

เทียบกับการเรียก `min_by` และ `max_by` แยกกันสองครั้ง (ซึ่งวนลิสต์สองรอบ) `minmax_by` วนแค่
รอบเดียวแล้วคืนทั้งคู่มาพร้อมกันเป็น Array 2 ช่อง `[min, max]`

### `min_by(n)`/`max_by(n)` — หา Top N

ทั้ง `min_by` และ `max_by` รับ argument ตัวเลขเพื่อขอ **หลายตัว** พร้อมกัน เรียงจากสุดขั้วที่สุด
ไปหาน้อยที่สุด:

```ruby
top2_expensive = products.max_by(2) { |p| p[:price] }
p top2_expensive.map { |p| p[:name] }
# => ["จอมอนิเตอร์", "คีย์บอร์ด"]

cheapest2_items = products.min_by(2) { |p| p[:price] }
p cheapest2_items.map { |p| p[:name] }
# => ["เมาส์", "หูฟัง"]
```

นี่คือวิธีที่กระชับกว่าการเขียน `sort_by { ... }.first(2)` หรือ `sort_by { ... }.last(2)` เอง
มาก เพราะ `max_by(n)`/`min_by(n)` ไม่ต้องเรียงทั้งลิสต์ทั้งหมดก่อน (ใช้ algorithm ที่มี
ประสิทธิภาพดีกว่าการ sort เต็มรูปแบบเมื่อ `n` เล็กกว่าขนาด array มาก)

### `minmax` ธรรมดา (ไม่มี block) — บนค่าที่เปรียบเทียบกันได้อยู่แล้ว

```ruby
numbers = [5, 3, 9, 1, 7]

p numbers.minmax   # => [1, 9]
p numbers.min       # => 1
p numbers.max       # => 9
```

> **เชื่อมโยงกับ Comparable (Part 010 Step 99):** ถ้า class ของเรา `include Comparable`
> และนิยาม `<=>` ไว้แล้ว (เหมือนตัวอย่าง `Product` ใน Step 99) `min`/`max`/`minmax` แบบไม่มี
> block จะใช้งานได้ทันทีโดยอาศัย `<=>` ที่เรานิยามไว้ ในขณะที่ `min_by`/`max_by` ใช้ได้แม้
> class นั้นจะยังไม่ได้ `include Comparable` เลยก็ตาม เพราะมันไม่ได้พึ่ง `<=>` ของ object
> แต่เปรียบเทียบ key ที่ block คำนวณให้แทน

---

## Step 130: Enumerator และ Lazy Evaluation — ประมวลผล Stream ข้อมูลขนาดใหญ่/ไม่จำกัด

นี่คือ concept ที่ทรงพลังที่สุดของ Part นี้ ทุก method ที่เรียนมาตลอด Part 004, 005 และ Part
นี้ (เช่น `each_slice(5)`) เมื่อเรียก**โดยไม่มี block** จะไม่ทำงานทันที แต่คืนค่าเป็น
**`Enumerator`** object แทน — เราเห็นสิ่งนี้มาตลอดโดยไม่ทันสังเกต (`return enum_for(:each)
unless block_given?` ใน `Bookshelf#each` ของ Part 010 ก็คือการคืน `Enumerator` นี่เอง)

### `Enumerator` คืออะไร

`Enumerator` คือ object ที่ **"ห่อหุ้มวิธีการวนซ้ำ"** ไว้ โดยยังไม่ได้ประมวลผลจริงจนกว่าจะถูก
เรียกใช้ (เช่นด้วย `.each`, `.next`, `.to_a`, หรือ `.first`) มันคือสิ่งที่ทำให้เราเขียน
`.each_slice(5).to_a` ต่อ `.map` หรือ `.with_index` ได้อย่างลื่นไหลโดยไม่ต้องสร้าง Array
กลางที่ไม่จำเป็น

### สร้าง Enumerator ด้วย `to_enum`/`enum_for`

จาก class ที่มี `each` อยู่แล้ว เราสร้าง `Enumerator` ของมันเองได้ด้วย `to_enum` (หรือ
`enum_for` ที่เห็นมาตั้งแต่ Part 010 — ทั้งสองชื่อทำงานเหมือนกันทุกประการ):

```ruby
# frozen_string_literal: true

class Playlist
  def initialize(songs)
    @songs = songs
  end

  def each
    return to_enum(:each) unless block_given?

    @songs.each { |song| yield song }
  end
end

playlist = Playlist.new(["Song A", "Song B", "Song C"])

enum = playlist.each          # เรียกโดยไม่มี block -> ได้ Enumerator กลับมา ไม่ทำงานทันที
p enum.class                   # => Enumerator

p enum.next                    # => "Song A"   (ดึงค่าถัดไปทีละตัว)
p enum.next                    # => "Song B"
p enum.next                    # => "Song C"

begin
  enum.next
rescue StopIteration
  puts "หมดเพลงแล้ว"
end
# => หมดเพลงแล้ว
```

**อธิบาย:** `Enumerator#next` ดึงค่าถัดไปทีละตัว **โดยไม่ต้องวน loop เอง** และจำตำแหน่งปัจจุบัน
ไว้ในตัว — เมื่อหมดข้อมูลจะ raise `StopIteration` (ไม่ใช่คืน `nil` เฉยๆ) นี่คือกลไกเบื้องหลัง
ที่ทำให้ `for`-loop และ `each` ทำงานได้ในภาษาโปรแกรมมิ่งจำนวนมาก — Ruby แค่เปิดให้เราควบคุมมัน
เองแบบ manual ได้ผ่าน `Enumerator`

### สร้าง Enumerator เองตั้งแต่ต้นด้วย `Enumerator.new`

นอกจากห่อหุ้ม `each` ที่มีอยู่แล้ว เรายังสร้าง `Enumerator` ขึ้นมาใหม่ทั้งหมดได้ด้วย
`Enumerator.new` ซึ่งเปิดโอกาสให้นิยามลำดับข้อมูลที่**ไม่มีที่สิ้นสุด** ได้โดยไม่ทำให้โปรแกรม
ค้าง (ตราบใดที่เราไม่พยายามดึงค่าออกมาทั้งหมด)

```ruby
fibonacci = Enumerator.new do |yielder|
  a, b = 0, 1
  loop do
    yielder << a          # เทียบเท่า yielder.yield(a)
    a, b = b, a + b
  end
end

p fibonacci.take(10)
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

p fibonacci.first(5)
# => [0, 1, 1, 2, 3]
```

**อธิบายกลไก:** block ที่ส่งเข้า `Enumerator.new` รับ parameter ชื่อ `yielder` แล้วเรียก
`yielder << value` (หรือ `yielder.yield(value)`) ทุกครั้งที่ต้องการ "ส่งค่าถัดไปออกไป" —
สังเกตว่าข้างในมี `loop do ... end` ซึ่งไม่มีวันจบเอง! ถ้าเราลองเรียก `fibonacci.to_a`
ตรงๆ โปรแกรมจะค้างทันทีเพราะพยายามสร้างลิสต์ Fibonacci ที่ไม่มีที่สิ้นสุด — แต่ `.take(10)`
และ `.first(5)` ปลอดภัย เพราะทั้งสอง method รู้จัก "หยุดเมื่อได้ครบจำนวนที่ต้องการ" โดยไม่
พยายามรันจน `loop` จบ (ซึ่งไม่มีทางจบอยู่แล้ว)

### `.lazy` — ทำให้ chain ของ Enumerable method ทำงานแบบ lazy

ปัญหาที่ `Enumerator.new` ยังแก้ไม่หมดคือ ถ้าเราต้องการ **ต่อ chain method หลายตัว** เช่น
`map` แล้ว `select` บนลำดับที่ไม่มีที่สิ้นสุด ปัญหาจะกลับมาทันที เพราะ `map`/`select` ปกติ
(แบบ eager) ต้องวนจนจบก่อนถึงจะคืนผลลัพธ์ได้ — ถ้าต้นทางไม่มีวันจบ มันก็ไม่มีวันคืนอะไรกลับมา
เลย นี่คือจุดที่ `.lazy` เข้ามาแก้ปัญหา

```ruby
# ปลอดภัย: เพราะ .lazy ทำให้ map ทำงานทีละตัวแบบ "ขี้เกียจ" (lazy) ไม่ใช่ทำทั้งหมดล่วงหน้า
p (1..Float::INFINITY).lazy.map { |n| n * 2 }.first(5)
# => [2, 4, 6, 8, 10]

# chain หลาย method พร้อมกันบนลำดับที่ไม่มีที่สิ้นสุด
result = (1..Float::INFINITY).lazy
  .select { |n| n % 3 == 0 }
  .map { |n| n * n }
  .first(5)
p result
# => [9, 36, 81, 144, 225]
```

**อธิบายกลไก:** `.lazy` เปลี่ยน `Range`/`Enumerator` ให้กลายเป็น `Enumerator::Lazy` ซึ่งเมื่อ
ต่อ `.map`, `.select`, `.reject`, `.flat_map` ฯลฯ ต่อกันไปเรื่อยๆ **จะไม่ทำงานทันที** แต่จะแค่
"จดจำ" ว่าต้องทำ operation อะไรบ้างตามลำดับ จนกว่าจะมีการเรียก method ที่บังคับให้ประมวลผลจริง
เช่น `.first(n)`, `.take(n)`, หรือ `.force`/`.to_a` — และที่สำคัญที่สุด: มันประมวลผล
**"ทีละสมาชิก ผ่านทุก operation ในลำดับ"** แทนที่จะประมวลผล**ทีละ operation ผ่านสมาชิกทั้งหมด**
แบบ eager

พูดให้เห็นภาพ: สมาชิกตัวที่ 1 จะถูก `select` เช็คก่อน ถ้าผ่านค่อยเข้า `map` ทันที แล้วค่อยไป
ตัวที่ 2 ต่อ — ไม่ใช่ "select ทุกตัวให้เสร็จก่อน แล้วค่อย map ทุกตัวที่เหลือ" ซึ่งเป็นวิธีที่
`select`/`map` ปกติทำ (และเป็นเหตุผลที่มันต้องรู้ขนาดข้อมูลทั้งหมดล่วงหน้า)

### พิสูจน์ความแตกต่างด้วยการนับจำนวนครั้งที่ block ถูกเรียก

```ruby
def count_calls_eager(range_end)
  count = 0
  (1..range_end).map { |n| count += 1; n * 2 }.select { |n| n > 6 }.first(1)
  count
end

def count_calls_lazy(range_end)
  count = 0
  (1..range_end).lazy.map { |n| count += 1; n * 2 }.select { |n| n > 6 }.first(1)
  count
end

p count_calls_eager(1_000_000)   # => 1000000   (map ต้องคำนวณครบทุกตัวก่อน select จะเริ่มทำงาน)
p count_calls_lazy(1_000_000)    # => 4          (พอเจอตัวที่ 4 (n=4, n*2=8>6) ก็หยุดทันที)
```

**นี่คือหัวใจของเหตุผลที่ lazy evaluation สำคัญ:** เวอร์ชัน eager ต้องรัน `map` ครบทั้ง
1,000,000 ตัวก่อนที่ `select` จะได้เริ่มทำงานด้วยซ้ำ (ทั้งที่ต้องการแค่ผลลัพธ์ตัวแรก) ในขณะที่
เวอร์ชัน lazy ประมวลผลแค่ 4 ตัวแรกแล้วหยุดทันทีที่ได้คำตอบที่ต้องการ — ยิ่งข้อมูลตั้งต้นใหญ่
หรือไม่มีที่สิ้นสุด ความต่างนี้ยิ่งมีความหมายมาก (จาก "ค้างตลอดไป" กลายเป็น "เสร็จเกือบทันที")

### ใช้ `.lazy` กับ Enumerator ที่สร้างเอง

```ruby
fibonacci = Enumerator.new do |yielder|
  a, b = 0, 1
  loop do
    yielder << a
    a, b = b, a + b
  end
end

first_5_even_fibs = fibonacci.lazy.select(&:even?).first(5)
p first_5_even_fibs
# => [0, 2, 8, 34, 144]
```

### เมื่อไหร่ควรใช้ `.lazy` ในทางปฏิบัติ

- **อ่านไฟล์ขนาดใหญ่ทีละบรรทัด** แล้วหาแค่ผลลัพธ์แรกๆ ที่ตรงเงื่อนไข โดยไม่ต้องโหลดทั้งไฟล์
  เข้า memory ก่อน (เช่น `File.foreach("huge.log").lazy.select { |line| line.include?("ERROR") }.first(10)`)
- **ประมวลผลลำดับที่ไม่มีที่สิ้นสุดในเชิงแนวคิด** เช่นจำนวนเฉพาะ, Fibonacci, หรือ stream ของ
  event ที่มาเรื่อยๆ ไม่รู้จบ
- **Chain การแปลง/กรองข้อมูลหลายขั้นตอนบนข้อมูลขนาดใหญ่มาก** เมื่อสนใจแค่ผลลัพธ์บางส่วน
  (เช่น `.first(n)`) ไม่ใช่ผลลัพธ์ทั้งหมด

> **ข้อควรระวัง:** ถ้าท้ายที่สุดต้องใช้ผลลัพธ์**ทั้งหมด**อยู่ดี (ไม่ได้ `.first(n)` หรือหยุด
> กลางทาง) `.lazy` แทบไม่ได้ประโยชน์ด้านประสิทธิภาพเลย เพราะยังไงก็ต้องประมวลผลทุกตัวเหมือนกัน
> — บาง operation ก็ **ไม่สามารถ lazy ได้จริงๆ** เพราะต้องเห็นข้อมูลครบก่อนถึงจะตอบได้ เช่น
> `sort`/`sort_by` (ต้องรู้ค่าทุกตัวก่อนถึงจะเรียงได้ว่าอะไรมาก่อนหลัง) หรือ `group_by`/`tally`
> (ต้องนับ/จัดกลุ่มให้ครบก่อนถึงจะสรุปผลได้) — `.lazy` ช่วยแค่ช่วง `map`/`select`/`reject`/
> `flat_map`/`take_while` ที่ประมวลผล "ทีละตัวได้จริงๆ" โดยไม่ต้องรอดูตัวอื่น

---

## แบบฝึกหัด: วิเคราะห์ข้อมูลออเดอร์ขนาดใหญ่

### โจทย์

เขียนโปรแกรม `order_analytics.rb` ที่จำลองข้อมูลออเดอร์ร้านค้าออนไลน์จำนวน 1,000 รายการ
แล้ววิเคราะห์ข้อมูลต่อไปนี้ โดยต้องใช้ method ที่เรียนใน Part นี้ให้ครบ:

1. **จัดกลุ่มออเดอร์ตามลูกค้า** ด้วย `group_by` แล้วแสดงจำนวนออเดอร์ของแต่ละคน
2. **คำนวณสถิติแบบแบ่งชุด** ด้วย `each_slice` (ชุดละ 100 รายการ) — ยอดรวมและค่าเฉลี่ยของ
   แต่ละชุด
3. **หาลูกค้าที่ใช้จ่ายสูงสุด 3 อันดับแรก** โดยใช้ `.lazy` ในขั้นตอนการดึงผลลัพธ์

### เฉลย

```ruby
# frozen_string_literal: true

# order_analytics.rb
require "date"

Order = Struct.new(:id, :customer, :amount, :date) do
  def to_s
    "##{id} #{customer}: #{amount} บาท (#{date})"
  end
end

def generate_orders(count)
  customers = %w[สมชาย สมหญิง มานี วิชัย อรทัย นภา ประยุทธ สุดา]
  rng = Random.new(42) # fix seed เพื่อให้ผลลัพธ์ reproducible ตอนทดสอบ

  Array.new(count) do |i|
    Order.new(
      i + 1,
      customers.sample(random: rng),
      rng.rand(50..3000),
      Date.new(2026, 1, 1) + (i % 90)
    )
  end
end

orders = generate_orders(1000)
puts "จำนวนออเดอร์ทั้งหมด: #{orders.size}"

# ============================================================
# 1) จัดกลุ่มออเดอร์ตามลูกค้าด้วย group_by
# ============================================================
by_customer = orders.group_by(&:customer)

puts "\n--- จำนวนออเดอร์ต่อลูกค้า ---"
by_customer.sort_by { |_customer, list| -list.size }.each do |customer, list|
  puts "  #{customer}: #{list.size} ออเดอร์"
end

# ============================================================
# 2) สถิติแบบแบ่งชุดด้วย each_slice (ชุดละ 100 รายการ)
# ============================================================
puts "\n--- สถิติแบบแบ่งชุด (each_slice 100) ---"
batch_stats = orders.each_slice(100).with_index(1).map do |batch, index|
  total = batch.sum(&:amount)
  average = (total / batch.size.to_f).round(2)
  { batch: index, count: batch.size, total: total, average: average }
end

batch_stats.each do |stat|
  puts "  ชุดที่ #{stat[:batch]}: #{stat[:count]} รายการ, " \
       "รวม #{stat[:total]} บาท, เฉลี่ย #{stat[:average]} บาท"
end

# ============================================================
# 3) หาลูกค้าที่ใช้จ่ายสูงสุด 3 อันดับแรก (ใช้ .lazy ตอนดึงผล)
# ============================================================
spend_by_customer = by_customer.transform_values { |list| list.sum(&:amount) }

# หมายเหตุ: sort_by ต้องเห็นข้อมูลทุกตัวก่อนถึงจะเรียงได้ (จึงยัง "eager" อยู่ตรงนี้เสมอ
# ไม่มีทางเลี่ยง) แต่การต่อ .lazy.first(3) ยังคงเป็นรูปแบบที่ถูกต้องและพบได้บ่อยในโค้ดจริง —
# มันทำให้ Enumerable chain นี้ไม่ต้องสร้าง Array ผลลัพธ์เต็มอีกชุดหลัง sort_by ทันที
# และเป็นรูปแบบเดียวกับที่จะใช้ต่อยอดเมื่อข้อมูลต้นทางมาจาก stream/pagination ขนาดใหญ่จริงๆ
top_spenders = spend_by_customer
               .sort_by { |_customer, total| -total }
               .lazy
               .first(3)

puts "\n--- ลูกค้าที่ใช้จ่ายสูงสุด 3 อันดับแรก ---"
top_spenders.each_with_index do |(customer, total), index|
  puts "  อันดับ #{index + 1}: #{customer} (#{total} บาท)"
end

puts "\nยอดขายรวมทั้งหมด: #{orders.sum(&:amount)} บาท"
```

ทดสอบรัน:

```bash
ruby order_analytics.rb
```

ตัวอย่างผลลัพธ์ (ใช้ seed คงที่ `Random.new(42)` ผลลัพธ์จะเหมือนกันทุกครั้งที่รัน):

```
จำนวนออเดอร์ทั้งหมด: 1000

--- จำนวนออเดอร์ต่อลูกค้า ---
  สมชาย: 144 ออเดอร์
  อรทัย: 132 ออเดอร์
  สมหญิง: 129 ออเดอร์
  วิชัย: 126 ออเดอร์
  มานี: 122 ออเดอร์
  ประยุทธ: 121 ออเดอร์
  สุดา: 115 ออเดอร์
  นภา: 111 ออเดอร์

--- สถิติแบบแบ่งชุด (each_slice 100) ---
  ชุดที่ 1: 100 รายการ, รวม 160042 บาท, เฉลี่ย 1600.42 บาท
  ชุดที่ 2: 100 รายการ, รวม 157321 บาท, เฉลี่ย 1573.21 บาท
  ชุดที่ 3: 100 รายการ, รวม 149270 บาท, เฉลี่ย 1492.7 บาท
  ...
  ชุดที่ 10: 100 รายการ, รวม 152671 บาท, เฉลี่ย 1526.71 บาท

--- ลูกค้าที่ใช้จ่ายสูงสุด 3 อันดับแรก ---
  อันดับ 1: สมชาย (219734 บาท)
  อันดับ 2: สมหญิง (203656 บาท)
  อันดับ 3: อรทัย (187864 บาท)

ยอดขายรวมทั้งหมด: 1506980 บาท
```

โค้ดนี้ทดสอบรันจริงแล้วบน Ruby 3.3.6 ให้ผลลัพธ์ตรงตามที่แสดงทุกจุด

### จุดที่ควรสังเกตในเฉลย

- **`Struct.new`** ใช้สร้าง `Order` แบบรวดเร็วโดยไม่ต้องเขียน `attr_accessor`/`initialize` เอง
  — เป็นการหยิบยืมมาใช้ก่อนที่จะเรียนละเอียดใน Part 014 เพราะสะดวกสำหรับตัวอย่างข้อมูลจำลอง
- **`group_by(&:customer)`** ใช้ Symbol-to-proc (จาก Part 008) แทนการเขียน
  `{ |order| order.customer }` เต็มรูปแบบ — ทั้งสองแบบเทียบเท่ากันทุกประการ
- **`each_slice(100).with_index(1).map { ... }`** แสดงให้เห็นว่า `each_slice` เมื่อไม่มี
  block จะคืน `Enumerator` ที่ต่อ `.with_index` และ `.map` ได้ตามปกติ (ทบทวนจาก Step 129)
  `.with_index(1)` เริ่มนับจาก 1 แทนที่จะเริ่มจาก 0 ให้เลขชุดอ่านง่ายขึ้น
- **`.lazy` ใน section 3** เป็นตัวอย่างที่ตรงไปตรงมาว่า **ไม่ใช่ทุก operation ที่ lazy ได้
  จริง** — `sort_by` ยังคงต้องวนดูข้อมูลทั้งหมดก่อนเสมอ (ไม่มีทางเลี่ยง เพราะการจะรู้ว่าใคร
  "สูงสุด" ต้องเห็นทุกคนก่อน) แต่การเขียน `.lazy.first(3)` ต่อท้ายยังคงเป็นรูปแบบที่ถูกต้อง
  และเตรียมโค้ดให้พร้อมขยายไปใช้กับข้อมูลที่มาจาก stream ขนาดใหญ่จริงๆ ในอนาคตได้ทันที (เช่น
  ถ้าเปลี่ยนมาอ่านจากไฟล์ log ที่มาเรื่อยๆ แทน)
- **`orders.sum(&:amount)`** ใช้ `sum` ซึ่งเป็นอีก method หนึ่งจาก `Enumerable` ที่ไม่ต้อง
  เขียน `reduce(:+)` เอง (ใช้ block เพื่อบอกว่าจะ sum จาก attribute ไหนของแต่ละ object)

> **สะพานเชื่อมสู่ Rails:** รูปแบบ `group_by` + `transform_values` ที่ทำใน exercise นี้คือ
> สิ่งเดียวกับที่ Rails ทำให้อัตโนมัติผ่าน `Order.group(:customer_id).sum(:amount)` (SQL
> `GROUP BY` + `SUM`) ที่จะเรียนใน Phase 4 — ความแตกต่างคือ Rails ทำการจัดกลุ่ม/รวมยอดที่ระดับ
> database โดยตรง (เร็วกว่ามากเมื่อข้อมูลมีหลักล้านแถว เพราะไม่ต้องโหลดทุกแถวเข้า Ruby ก่อน)
> ในขณะที่ตัวอย่างนี้ทำในระดับ Ruby object ล้วนๆ — แต่ **แนวคิดการจัดกลุ่มและสรุปผลเหมือนกัน
> เป๊ะ** ทักษะที่ฝึกในนี้จึงย้ายไปอ่าน/เขียน ActiveRecord query ได้ทันทีเมื่อถึง Phase 4

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **หา top 3 วันที่มียอดขายรวมสูงสุด** — จัดกลุ่มออเดอร์ตาม `date` ด้วย `group_by` แล้วหา
   ผลรวมของแต่ละวัน จากนั้นใช้ `max_by(3)` เพื่อหา 3 วันที่มียอดขายสูงสุด (ใบ้: ผลลัพธ์ของ
   `group_by(&:date)` คือ Hash ที่ value เป็น Array ของ Order — ต้อง `transform_values` ให้
   เป็นยอดรวมก่อน)
2. **ตรวจจับ "ลูกค้าประจำ" ด้วย `chunk_while`** — สมมติว่า orders เรียงตามวันที่อยู่แล้ว ให้
   หาลูกค้าที่สั่งซื้อ**ติดต่อกันทุกวัน**อย่างน้อย 3 วันขึ้นไป (ใบ้: ต้อง `group_by(&:customer)`
   ก่อน แล้วค่อยใช้ `chunk_while` กับวันที่ของแต่ละคนแยกกัน เหมือนตัวอย่าง login streak ใน
   Step 127)
3. **เขียน Lazy Enumerator สำหรับ pagination แบบ infinite scroll** — สร้าง
   `Enumerator.new` ที่ jield ออเดอร์ทีละหน้า (page) โดยจำลองว่าแต่ละหน้าต้อง "ดึงข้อมูลช้าๆ"
   (ใส่ `sleep 0.01` เพื่อจำลอง latency) แล้วใช้ `.lazy.first(2)` เพื่อดึงแค่ 2 หน้าแรกโดยไม่
   ต้องรอให้หน้าอื่นๆ ที่เหลือถูกสร้างขึ้นก่อน — สังเกตความต่างของเวลาที่ใช้ระหว่างมี `.lazy`
   กับไม่มี (ใช้ `Time.now` จับเวลาก่อน/หลัง เทียบกัน)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ทบทวนว่า `map`, `select`, `reduce` ที่เรียนใน Part 004/005 ล้วนมาจาก module `Enumerable`
  เดียวกัน ซึ่งให้ method มากกว่า 60 ตัวฟรีจากการนิยาม `each` เพียง method เดียว
- ใช้ `each_slice` แบ่งข้อมูลเป็นชุดสำหรับ batch processing และ pagination
- ใช้ `each_cons` สร้างหน้าต่างเลื่อน (sliding window) สำหรับคำนวณค่าเฉลี่ยเคลื่อนที่และ
  ผลต่างระหว่างสมาชิกที่ติดกัน
- ใช้ `group_by` จัดกลุ่มข้อมูลเป็น Hash of Arrays ตาม key ที่คำนวณเอง และเข้าใจความแตกต่าง
  จาก `partition` ที่แบ่งได้แค่ 2 กลุ่มเสมอ พร้อมรู้วิธีผสมทั้งสองเข้าด้วยกัน
- เข้าใจ `flat_map` อย่างละเอียดว่าแบนราบแค่ "1 ชั้น" ต่างจาก `flatten` แบบไม่ระบุ argument
  ที่แบนราบทุกชั้น
- ใช้ `chunk_while`/`slice_when` จัดกลุ่มสมาชิกที่ **ต่อเนื่องกันในลำดับเดิม** ซึ่งต่างจาก
  `group_by` ที่ไม่สนใจตำแหน่ง
- ใช้ `tally` นับความถี่ในคำเดียว แทนการเขียน `reduce`/`Hash.new(0)` เอง
- ใช้ `min_by`/`max_by`/`minmax_by` หาค่าสุดขั้วจาก key ที่คำนวณเอง รวมถึงการขอ Top N ด้วย
  `max_by(n)`
- เข้าใจ **`Enumerator`** ทั้งการสร้างจาก `to_enum`/`enum_for` และการสร้างใหม่ด้วย
  `Enumerator.new` + `yielder`
- เข้าใจ **`.lazy`** และหลักการ lazy evaluation อย่างลึกซึ้ง — ทำไมมันประหยัดการประมวลผลได้
  จริง (พิสูจน์ด้วยการนับจำนวนครั้งที่ block ถูกเรียก) และรู้ข้อจำกัดว่า operation ประเภท
  `sort`/`group_by`/`tally` lazy ไม่ได้เพราะต้องเห็นข้อมูลครบก่อนเสมอ
- ประยุกต์ใช้ทุก method ข้างต้นร่วมกันในโปรเจกต์วิเคราะห์ข้อมูลออเดอร์ขนาด 1,000 รายการ

**ต่อไป (Part 014):** เราจะเรียนรู้ **Comparable module** อย่างละเอียดขึ้นกว่าที่เห็นใน Part
010 (`between?`, `clamp`), **`Struct`** ที่เพิ่งหยิบยืมมาใช้ใน exercise ของ Part นี้ (มาดูกันว่า
มันทำงานอย่างไรเบื้องหลัง และมี method อะไรให้ใช้บ้าง), **`OpenStruct`** สำหรับสร้าง object
ที่มี attribute แบบไดนามิก และ **`Data` class** ฟีเจอร์ใหม่ของ Ruby 3.2+ ที่ออกแบบมาสำหรับสร้าง
value object แบบ immutable โดยเฉพาะ — ทั้งหมดนี้คือเครื่องมือที่ช่วยลดโค้ด boilerplate ในการ
สร้าง class ง่ายๆ ที่เก็บแค่ข้อมูลไม่กี่ฟิลด์ ซึ่งเราใช้ `Struct` ไปแล้วบางส่วนใน Part นี้โดย
ยังไม่ได้อธิบายละเอียด
