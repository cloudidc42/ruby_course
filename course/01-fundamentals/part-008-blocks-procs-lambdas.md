# Part 008: Blocks, yield, Proc, Lambda เบื้องต้น

> **Step ครอบคลุมใน Part นี้:** Step 71–80
> **ระดับ:** เริ่มต้น–กลาง (ควรผ่าน Part 007 เรื่อง Method มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

Block, Proc และ Lambda เป็นหัวใจสำคัญที่ทำให้ Ruby เขียนโค้ดได้ "หรู" และเป็นธรรมชาติ
เวลาเราเขียน `[1, 2, 3].each { |n| puts n }` หรือ `users.map(&:name)` เบื้องหลังคือกลไก
เหล่านี้ทั้งหมด และเมื่อไปเรียน Rails จะเจอโค้ดแบบนี้แทบทุกไฟล์ — ตั้งแต่ `respond_to do |format|`
ไปจนถึง callback อย่าง `before_save { self.slug ||= generate_slug }` ดังนั้น Part นี้จะปูพื้นฐาน
ให้แน่นก่อนไปต่อ

## สารบัญของ Part นี้

- Step 71: Block คืออะไร — `do...end` vs `{}` และธรรมเนียมการเลือกใช้
- Step 72: การส่ง block ให้ method ที่มีอยู่แล้ว (`each`, `map`, `times` ฯลฯ)
- Step 73: `yield` — เรียกใช้ block จากภายใน method ของเราเอง
- Step 74: `block_given?` — ทำให้ block เป็นตัวเลือก (optional)
- Step 75: เขียน method ของตัวเองที่รับ block แบบเต็มรูปแบบ (`my_each`, `my_map`, `my_select`)
- Step 76: แปลง block เป็น Proc ด้วย `&block`, `Proc.new`/`proc`, และการเรียก Proc
- Step 77: Lambda — `lambda do...end` และ stabby lambda `->() {}`
- Step 78: Proc vs Lambda — arity และพฤติกรรมของ `return`
- Step 79: Closures — การจับตัวแปร (variable capture) และตัวอย่าง counter generator
- Step 80: แบบฝึกหัด Part นี้ — Retry-with-block utility และ Event Callback System

---

## Step 71: Block คืออะไร — `do...end` vs `{}` และธรรมเนียมการเลือกใช้

**Block** คือกลุ่มคำสั่ง (chunk of code) ที่เราส่งแนบไปกับการเรียก method ได้ โดยไม่ต้องตั้งชื่อ
method ให้มัน (ต่างจาก Proc/Lambda ที่เป็น object และเก็บไว้ในตัวแปรได้) block เป็นเพียง
"syntax" ที่แนบไปกับการเรียก method ครั้งนั้นๆ เท่านั้น เขียนได้ 2 แบบ

```ruby
# frozen_string_literal: true

# แบบที่ 1: ใช้ do...end (นิยมใช้กับ block หลายบรรทัด)
[1, 2, 3].each do |n|
  puts n * 2
end
# => 2
# => 4
# => 6

# แบบที่ 2: ใช้ {} (นิยมใช้กับ block บรรทัดเดียว)
[1, 2, 3].each { |n| puts n * 2 }
# => 2
# => 4
# => 6
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ — สิ่งที่อยู่ระหว่าง `do...end` หรือ `{}` คือโค้ดที่จะถูก
เรียกใช้ (execute) เมื่อ method ที่รับ block นั้น "เรียก" มันกลับมา (ผ่าน `yield` ซึ่งจะสอนใน
Step 73) ส่วน `|n|` คือ **block parameter** เทียบเท่ากับ parameter ของ method ทั่วไป

### ทำไมต้องมี 2 แบบ ใช้ตัวไหนดี

Ruby community มีธรรมเนียม (convention) ที่ยึดถือกันแทบทุกทีมคือ **"เส้นเดียวใช้ `{}`,
หลายเส้นใช้ `do...end`"** เหตุผลไม่ใช่แค่ความสวยงาม แต่เกี่ยวกับ **operator precedence**
(ลำดับความสำคัญในการประมวลผล) ด้วย

```ruby
# {} จับกับ method ที่อยู่ติดกันที่สุด (เกาะแน่นกว่า)
# do...end จับกับ method ตัวซ้ายสุดของ expression (เกาะหลวมกว่า)

# ตัวอย่างที่แสดงความต่างชัดเจน
def greet(name)
  yield(name) if block_given?
end

# puts รับ argument เป็นผลลัพธ์ของ greet(...) { ... } ทั้งก้อน เพราะ {} เกาะกับ greet ก่อน
puts greet("Ruby") { |n| "Hello, #{n}!" }
# => Hello, Ruby!

# แต่ถ้าใช้ do...end ผลจะต่างออกไป เพราะ do...end เกาะกับ puts (ซ้ายสุด) ไม่ใช่ greet
puts greet("Ruby") do |n|
  "Hello, #{n}!"
end
# => (บรรทัดว่างเปล่า) เพราะ do...end ไปผูกกับ puts ซึ่งไม่รับ block กลายเป็นแค่ argument
#    เฉยๆ ที่ไม่มีใครใช้ ส่วน greet("Ruby") ถูกเรียกโดยไม่มี block เลย block_given? จึงเป็น
#    false, yield ไม่ถูกเรียก, greet จึงคืนค่า nil แล้ว puts(nil) ก็พิมพ์บรรทัดว่างออกมา
```

**กฎปฏิบัติที่ยึดถือกันทั่วไปในวงการ Ruby (ตาม Ruby Style Guide):**

1. ใช้ `{}` เมื่อ block มีแค่ 1 บรรทัด และเน้นว่ามันคือ "ค่าที่คืนออกมา" (expression-oriented)
   เช่น `numbers.select { |n| n.even? }`
2. ใช้ `do...end` เมื่อ block มีหลายบรรทัด และเน้นว่ามันคือ "การกระทำหลายขั้นตอน"
   (procedure-oriented) เช่นตัวอย่าง `each do |n| ... end` ด้านบน
3. ถ้า block บรรทัดเดียวแต่ค่าที่คืนกลับมาไม่ได้ถูกใช้ต่อ (เช่นแค่ต้องการ side effect
   อย่าง `puts`) จะใช้ `do...end` หรือ `{}` ก็ได้ แต่ทีมส่วนใหญ่จะยึดกฎ "สั้นใช้ `{}`" อยู่ดี
   เพื่อความสม่ำเสมอ

```ruby
# ตัวอย่างที่เป็นไปตามธรรมเนียม (ใช้บ่อยเวลาไป Rails — จำลอง user object ด้วย Struct)
User = Struct.new(:name, :active) do
  def active?
    active
  end
end

users = [User.new("Somchai", true), User.new("Malee", false), User.new("Piti", true)]

active_users = users.select { |u| u.active? }                 # {} สั้น เน้นค่าที่ได้กลับมา
p active_users.map(&:name)
# => ["Somchai", "Piti"]

users.each do |user|                                          # do...end หลายบรรทัด เน้นการกระทำ
  status = user.active? ? "ใช้งานอยู่" : "ไม่ได้ใช้งาน"
  puts "#{user.name}: #{status}"
end
# => Somchai: ใช้งานอยู่
# => Malee: ไม่ได้ใช้งาน
# => Piti: ใช้งานอยู่
```

> **หมายเหตุ:** RuboCop (ตัวตรวจ style ที่เราติดตั้งใน Part 001) มี cop ชื่อ
> `Style/BlockDelimiters` ที่บังคับใช้กฎนี้โดยอัตโนมัติ ถ้าเขียนผิดธรรมเนียม RuboCop จะเตือน

---

## Step 72: การส่ง block ให้ method ที่มีอยู่แล้ว (`each`, `map`, `times` ฯลฯ)

ก่อนจะเขียน method ของตัวเองที่รับ block เราควรคุ้นเคยกับการ "ใช้" block กับ method ที่ Ruby
มีให้อยู่แล้วก่อน (หลายตัวเราเคยเห็นมาแล้วใน Part 004 เรื่อง Array แต่รอบนี้จะมองในมุมของ
"block" ให้ชัดขึ้น)

```ruby
# frozen_string_literal: true

# 1) Integer#times — รัน block ซ้ำตามจำนวนที่กำหนด
3.times { |i| puts "รอบที่ #{i}" }
# => รอบที่ 0
# => รอบที่ 1
# => รอบที่ 2

# 2) Array#each — วนแต่ละสมาชิก ไม่สนใจค่า return ของ block เพราะ each คืนตัว array เดิม
result = [1, 2, 3].each { |n| n * 100 }
p result
# => [1, 2, 3]   (each คืนค่า array ต้นฉบับเสมอ ไม่ใช่ผลลัพธ์จาก block)

# 3) Array#map — ใช้ค่า return ของ block มาสร้าง array ใหม่
squared = [1, 2, 3].map { |n| n * n }
p squared
# => [1, 4, 9]

# 4) Array#select / Array#reject — ใช้ค่า return ของ block (truthy/falsy) ในการกรอง
evens = (1..10).select { |n| n.even? }
p evens
# => [2, 4, 6, 8, 10]

# 5) Array#reduce (inject) — สะสมค่าโดยใช้ block รับ 2 parameter (accumulator, current)
sum = [1, 2, 3, 4].reduce(0) { |acc, n| acc + n }
p sum
# => 10

# 6) Hash#each — block รับได้ทั้งแบบ 1 parameter (คู่ key-value) หรือ 2 parameter
{ a: 1, b: 2 }.each { |pair| p pair }
# => [:a, 1]
# => [:b, 2]

{ a: 1, b: 2 }.each { |key, value| puts "#{key} => #{value}" }
# => a => 1
# => b => 2
```

**สังเกตสิ่งสำคัญ:** method แต่ละตัว "ตัดสินใจเอง" ว่าจะทำอะไรกับ block ที่ส่งเข้ามา —
`each` แค่รัน block ทีละรอบแล้วทิ้งค่า return, `map` เก็บค่า return มาสร้าง array ใหม่,
`select` ใช้ค่า return เป็นเงื่อนไข truthy/falsy นี่คือเหตุผลว่าทำไม block ถึงยืดหยุ่นมาก —
ตัว method เป็นคนกำหนดความหมายของ block เอง ไม่ใช่ syntax ของภาษา

### Block รับได้มากกว่า 1 parameter และใช้ `_` แทน parameter ที่ไม่ใช้

```ruby
[["Somchai", 25], ["Malee", 30]].each do |name, age|
  puts "#{name} อายุ #{age} ปี"
end
# => Somchai อายุ 25 ปี
# => Malee อายุ 30 ปี

# ถ้าไม่สนใจ index ให้ใช้ _ เพื่อสื่อความหมายว่า "ตั้งใจไม่ใช้ตัวแปรนี้"
["a", "b", "c"].each_with_index do |value, _index|
  puts value
end
# => a
# => b
# => c
```

---

## Step 73: `yield` — เรียกใช้ block จากภายใน method ของเราเอง

ตอนนี้เราใช้ block กับ method ของ Ruby มาเยอะแล้ว ทีนี้มาดูว่า method อย่าง `each` รู้ได้
อย่างไรว่าต้อง "เรียก" block ตอนไหน — คำตอบคือคำสั่ง **`yield`**

```ruby
# frozen_string_literal: true

def say_hello
  puts "ก่อนเรียก block"
  yield
  puts "หลังเรียก block"
end

say_hello { puts "สวัสดี จาก block!" }
# => ก่อนเรียก block
# => สวัสดี จาก block!
# => หลังเรียก block
```

`yield` คือคำสั่งที่บอกว่า "ให้ไปรัน block ที่ผู้เรียก method ส่งเข้ามา ณ จุดนี้" ทำงานคล้าย
การเรียก method ที่ไม่มีชื่อ (anonymous method call) `yield` ส่ง argument ให้ block ได้ด้วย
และรับค่า return จาก block กลับมาได้เหมือนเรียก method ปกติ

```ruby
def calculate
  result = yield(10, 5)
  puts "ผลลัพธ์จาก block คือ #{result}"
end

calculate { |a, b| a + b }
# => ผลลัพธ์จาก block คือ 15

calculate { |a, b| a * b }
# => ผลลัพธ์จาก block คือ 50
```

### ตัวอย่างจำลอง `each` ของเราเอง (เข้าใจกลไกเบื้องหลัง)

```ruby
class SimpleList
  def initialize(items)
    @items = items
  end

  def my_each
    index = 0
    while index < @items.length
      yield(@items[index])   # เรียก block พร้อมส่งค่า element ปัจจุบันเข้าไป
      index += 1
    end
    self
  end
end

list = SimpleList.new(["apple", "banana", "cherry"])
list.my_each { |fruit| puts "ผลไม้: #{fruit}" }
# => ผลไม้: apple
# => ผลไม้: banana
# => ผลไม้: cherry
```

นี่คือหลักการเดียวกับที่ `Array#each`, `Hash#each`, `Integer#times` ใช้ภายในของ Ruby เอง
(ในระดับ C extension) — **block ทำให้ผู้ใช้ method ควบคุมได้ว่า "จะทำอะไร" กับแต่ละรอบ
โดยที่ method ต้นทางไม่จำเป็นต้องรู้ล่วงหน้าเลยว่าจะถูกใช้งานแบบไหน** นี่คือแก่นของ
การออกแบบ API แบบ Ruby ที่เรียกว่า **inversion of control**

### `yield` กับ argument ที่ไม่ตรงจำนวน

เหมือนกับ method ทั่วไป — ถ้า `yield` ส่ง argument มากกว่าที่ block รับ ส่วนเกินจะถูกทิ้ง
ถ้าน้อยกว่า parameter ที่เหลือจะเป็น `nil` (block **ไม่เข้มงวดเรื่อง arity** เหมือน method
ปกติ ซึ่งเป็นเรื่องสำคัญที่จะพูดถึงอีกครั้งตอนเทียบกับ Lambda ใน Step 78)

```ruby
def demo
  yield(1, 2, 3)
end

demo { |a| puts a }              # => 1   (รับแค่ตัวเดียว ตัวที่เหลือถูกทิ้ง ไม่ error)
demo { |a, b, c, d| puts d.inspect }  # => nil (d ไม่มีค่าส่งมา ได้ nil ไม่ error)
```

---

## Step 74: `block_given?` — ทำให้ block เป็นตัวเลือก (optional)

ถ้า method มี `yield` แต่ผู้เรียกไม่ได้ส่ง block มาให้ จะเกิด `LocalJumpError`

```ruby
def broken
  yield
end

broken
# => LocalJumpError (no block given (yield))
```

เพื่อป้องกันปัญหานี้ Ruby มี method `block_given?` ที่คืนค่า `true`/`false` บอกว่ามีการส่ง
block เข้ามาหรือไม่ ทำให้เราออกแบบ method ที่ **block เป็นตัวเลือก** ได้ (ทำงานได้ทั้งแบบ
มี block และไม่มี)

```ruby
# frozen_string_literal: true

def process_numbers(numbers)
  if block_given?
    numbers.map { |n| yield(n) }
  else
    numbers   # ไม่มี block ก็คืนค่าเดิมกลับไปเฉยๆ
  end
end

p process_numbers([1, 2, 3])
# => [1, 2, 3]

p process_numbers([1, 2, 3]) { |n| n * 10 }
# => [10, 20, 30]
```

### ตัวอย่างที่ใช้บ่อยในโค้ดจริง: logging แบบมี block เสริม

```ruby
def with_logging(task_name)
  puts "[START] #{task_name}"
  result = block_given? ? yield : nil
  puts "[DONE]  #{task_name}"
  result
end

with_logging("คำนวณผลรวม") do
  total = (1..100).sum
  puts "ผลรวมคือ #{total}"
end
# => [START] คำนวณผลรวม
# => ผลรวมคือ 5050
# => [DONE]  คำนวณผลรวม

with_logging("งานที่ไม่มี block")
# => [START] งานที่ไม่มี block
# => [DONE]  งานที่ไม่มี block
```

รูปแบบนี้ (`if block_given? ... else ...`) เป็น pattern มาตรฐานที่จะเจอบ่อยมากใน Rails —
เช่น `respond_to do |format| ... end` หรือ helper method จำนวนมากที่ยอมให้ override
พฤติกรรมผ่าน block ได้ ถ้าไม่ส่ง block มาก็มี default behavior ให้ใช้แทน

---

## Step 75: เขียน method ของตัวเองที่รับ block แบบเต็มรูปแบบ (`my_each`, `my_map`, `my_select`)

คราวนี้มาสร้าง method จำลอง `each`, `map`, `select` ของ Enumerable ขึ้นมาเองทั้งหมด
เพื่อความเข้าใจกลไกภายในอย่างลึกซึ้ง (นี่คือแบบฝึกหัดคลาสสิกที่ใช้สอนเรื่อง block ทั่วโลก)

```ruby
# frozen_string_literal: true

module MyEnumerable
  # จำลอง Array#each
  def my_each(array)
    i = 0
    while i < array.length
      yield(array[i])
      i += 1
    end
    array
  end
  module_function :my_each

  # จำลอง Array#map — เก็บผลลัพธ์จาก yield ใส่ array ใหม่
  def my_map(array)
    result = []
    my_each(array) { |item| result << yield(item) }
    result
  end
  module_function :my_map

  # จำลอง Array#select — เก็บเฉพาะ item ที่ block คืนค่า truthy
  def my_select(array)
    result = []
    my_each(array) { |item| result << item if yield(item) }
    result
  end
  module_function :my_select

  # จำลอง Array#reduce (แบบมี initial value)
  def my_reduce(array, initial)
    accumulator = initial
    my_each(array) { |item| accumulator = yield(accumulator, item) }
    accumulator
  end
  module_function :my_reduce
end

numbers = [1, 2, 3, 4, 5]

MyEnumerable.my_each(numbers) { |n| print "#{n} " }
puts
# => 1 2 3 4 5

p MyEnumerable.my_map(numbers) { |n| n**2 }
# => [1, 4, 9, 16, 25]

p MyEnumerable.my_select(numbers) { |n| n.odd? }
# => [1, 3, 5]

p MyEnumerable.my_reduce(numbers, 0) { |sum, n| sum + n }
# => 15
```

**ประเด็นสำคัญที่ควรสังเกต:**

- `my_map` และ `my_select` ทั้งคู่ "ใช้" `my_each` ภายใน แล้ว `yield` (ของ `my_each`)
  ก็ยังส่งต่อไปเรียก block ที่ผู้ใช้ `my_map`/`my_select` ส่งมาให้ — เพราะ `yield` ใน
  `my_each` block (`{ |item| result << yield(item) }`) หมายถึง block ของ method ที่กำลัง
  รันอยู่ ณ ขณะนั้น (คือ `my_map`) ไม่ใช่ของ `my_each`
- นี่คือรูปแบบเดียวกับที่ Ruby แท้ๆ ใช้สร้าง `Enumerable` module (ที่เราจะเรียนเรื่อง
  `Enumerable` แบบเจาะลึกใน Part 013) — ทุก method ใน `Enumerable` ถูกสร้างจาก `each`
  เพียงตัวเดียวเท่านั้น

### เขียนเป็น instance method ของ class จริง

```ruby
class Playlist
  include Enumerable   # ต้องมี method #each ให้ Enumerable ใช้ แล้วจะได้ map/select/... ฟรี

  def initialize
    @songs = []
  end

  def add(song)
    @songs << song
    self   # คืน self เพื่อให้ chain method ต่อได้ เช่น playlist.add("A").add("B")
  end

  def each
    return enum_for(:each) unless block_given?   # รองรับกรณีเรียกโดยไม่มี block (คืน Enumerator)

    @songs.each { |song| yield(song) }
  end
end

playlist = Playlist.new
playlist.add("Bohemian Rhapsody").add("Imagine").add("Hotel California")

playlist.each { |song| puts "กำลังเล่น: #{song}" }
# => กำลังเล่น: Bohemian Rhapsody
# => กำลังเล่น: Imagine
# => กำลังเล่น: Hotel California

# เพราะ include Enumerable + มี #each → ได้ method อื่นๆ มาฟรีทันที!
p playlist.map(&:upcase)
# => ["BOHEMIAN RHAPSODY", "IMAGINE", "HOTEL CALIFORNIA"]

p playlist.select { |song| song.include?("H") }
# => ["Bohemian Rhapsody", "Hotel California"]

p playlist.count
# => 3
```

นี่คือตัวอย่างที่ทรงพลังมาก — แค่นิยาม `each` ตัวเดียวให้ถูกต้อง แล้ว `include Enumerable`
เราจะได้ method อีกกว่า 50 ตัว (`map`, `select`, `reduce`, `sort`, `find`, `count`, ...)
มาใช้ฟรีทันที เพราะทุก method เหล่านั้นถูกสร้างขึ้นจาก `each` + block ภายใต้ Ruby standard
library เอง

---

## Step 76: แปลง block เป็น Proc ด้วย `&block`, `Proc.new`/`proc`, และการเรียก Proc

จนถึงตอนนี้ block เป็นเพียง "syntax แนบท้าย" ที่ใช้ครั้งเดียวแล้วหายไป แต่บางครั้งเราอยากจะ
**เก็บ block ไว้ในตัวแปร** เพื่อส่งต่อไปที่อื่น หรือเรียกใช้ซ้ำหลายครั้ง — นี่คือหน้าที่ของ
**Proc** (ย่อมาจาก "procedure") ซึ่งเป็น object ที่ห่อหุ้มโค้ดก้อนหนึ่งเอาไว้

### แปลง block ที่รับมาให้เป็น Proc ด้วย `&`

```ruby
# frozen_string_literal: true

def capture_block(&block)
  puts "class ของ block คือ #{block.class}"
  block   # คืนค่า Proc object ออกไป
end

my_proc = capture_block { |x| x * 2 }
# => class ของ block คือ Proc

puts my_proc.call(5)
# => 10
```

เมื่อเราใส่ `&` นำหน้า parameter สุดท้ายของ method (เช่น `&block`) Ruby จะ:

1. รับ block ที่ถูกส่งมาตอนเรียก method (ไม่ว่าจะเขียนด้วย `do...end` หรือ `{}`)
2. แปลงมันเป็น **Proc object** แล้วเก็บไว้ในตัวแปรที่ตั้งชื่อไว้ (ในที่นี้คือ `block`)

หลังจากนั้นเราจะ `yield` แบบเดิม หรือจะเรียก `block.call(...)` ตรงๆ ก็ได้ผลเหมือนกัน

```ruby
def with_named_proc(&callback)
  callback.call("ทดสอบ")   # เทียบเท่ากับ yield("ทดสอบ") ถ้าใช้ yield แทน
end

with_named_proc { |msg| puts "ได้รับข้อความ: #{msg}" }
# => ได้รับข้อความ: ทดสอบ
```

**เมื่อไหร่ควรใช้ `yield` เมื่อไหร่ควรใช้ `&block`:**

- ใช้ `yield` ตรงๆ เมื่อแค่ต้องการเรียก block ครั้งเดียวหรือไม่กี่ครั้ง (เร็วกว่าเล็กน้อย
  เพราะไม่ต้องสร้าง Proc object)
- ใช้ `&block` เมื่อต้องการ **ส่ง block ต่อ** ไปให้ method อื่น, เก็บไว้เรียกทีหลัง,
  หรือตรวจสอบ/manipulate มันในฐานะ object (เช่น เช็ค `block.arity`)

### สร้าง Proc ได้โดยตรงด้วย `Proc.new` หรือ `proc`

```ruby
# วิธีที่ 1: Proc.new รับ block ตอนสร้าง
say_hi = Proc.new { |name| puts "Hi, #{name}!" }

# วิธีที่ 2: ใช้ Kernel#proc (นิยมมากกว่า สั้นกว่า และเป็นที่แนะนำใน style guide ปัจจุบัน)
say_hello = proc { |name| puts "Hello, #{name}!" }

say_hi.call("Somchai")
say_hello.call("Malee")
# => Hi, Somchai!
# => Hello, Malee!
```

### 3 วิธีเรียก Proc ที่เทียบเท่ากันทุกประการ

```ruby
add = proc { |a, b| a + b }

puts add.call(2, 3)   # => 5   (วิธีมาตรฐาน อ่านง่ายที่สุด)
puts add.(2, 3)       # => 5   (syntax sugar ของ .call เรียกว่า "dot-paren call")
puts add[2, 3]        # => 5   (เหมือนเรียก array/hash แต่จริงๆ คือ alias ของ .call)
puts add.yield(2, 3)  # => 5   (มีน้อยคนใช้ แต่ก็ทำงานได้เหมือนกัน)
```

ทั้ง 4 แบบทำงานเหมือนกันทุกประการ ในทางปฏิบัติ `.call` เป็นที่นิยมที่สุดเพราะอ่านง่ายและ
สื่อความหมายชัดเจนที่สุดว่า "กำลังเรียกใช้งาน Proc นี้"

### Proc ยืดหยุ่นเรื่องจำนวน argument (arity) เหมือน block ทุกประการ

```ruby
lenient = proc { |a, b, c| [a, b, c] }
p lenient.call(1)          # => [1, nil, nil]  (argument ขาด ไม่ error ได้ nil แทน)
p lenient.call(1, 2, 3, 4) # => [1, 2, 3]      (argument เกิน ถูกทิ้งไปเฉยๆ)
```

พฤติกรรม "ยืดหยุ่น ไม่เข้มงวดเรื่อง arity" นี้เป็นจุดที่ทำให้ Proc ต่างจาก Lambda อย่างชัดเจน
ซึ่งจะอธิบายเปรียบเทียบใน Step 78

---

## Step 77: Lambda — `lambda do...end` และ stabby lambda `->() {}`

**Lambda** เป็น object ชนิดเดียวกับ Proc (ตรวจสอบด้วย `.class` จะได้ `Proc` เหมือนกัน) แต่มี
พฤติกรรมบางอย่างต่างออกไป (จะเทียบละเอียดใน Step 78) Ruby มี syntax สร้าง Lambda 2 แบบ

### วิธีที่ 1: `lambda do...end` หรือ `lambda { }`

```ruby
# frozen_string_literal: true

greet = lambda { |name| "สวัสดี, #{name}!" }
puts greet.call("Ruby")
# => สวัสดี, Ruby!

multi_line = lambda do |a, b|
  sum = a + b
  sum * 2
end
puts multi_line.call(3, 4)
# => 14
```

### วิธีที่ 2: Stabby lambda `->() { }` (นิยมมากที่สุดในโค้ดยุคปัจจุบัน)

```ruby
# ไม่มี parameter
say_hi = -> { puts "Hi!" }
say_hi.call
# => Hi!

# มี parameter — วงเล็บใส่ก่อน {}
square = ->(x) { x * x }
puts square.call(5)
# => 25

# หลาย parameter
add = ->(a, b) { a + b }
puts add.call(2, 3)
# => 5

# หลายบรรทัด ใช้ do...end แทน {}
process = ->(x) do
  doubled = x * 2
  doubled + 1
end
puts process.call(10)
# => 21
```

**ทำไมเรียกว่า "stabby"?** เพราะสัญลักษณ์ `->` มีรูปร่างคล้ายลูกศรแทงเข้าไป (stab) เป็น
syntax ที่ได้รับความนิยมมากในโค้ด Ruby ยุคใหม่ เพราะกระชับกว่า `lambda { }` และเห็นชัดเจนว่า
นี่คือ Lambda ไม่ใช่ Proc ทั่วไป

```ruby
# เทียบ syntax แบบเคียงข้างกัน — ทั้งสองบรรทัดทำงานเหมือนกันทุกประการ
double_v1 = lambda { |x| x * 2 }
double_v2 = ->(x) { x * 2 }

puts double_v1.call(5)  # => 10
puts double_v2.call(5)  # => 10
puts double_v1.class    # => Proc  (!)
puts double_v2.class    # => Proc  (!)
puts double_v1.lambda?  # => true
puts double_v2.lambda?  # => true
```

> **ข้อสังเกตสำคัญ:** ทั้ง Proc และ Lambda มี class เดียวกันคือ `Proc` — Lambda ไม่ใช่ class
> แยกต่างหาก แต่เป็น Proc object ที่ถูกตั้ง **flag พิเศษ** ให้ทำงานเข้มงวดขึ้น ตรวจสอบได้
> ด้วย method `#lambda?`

### ส่ง Lambda เป็น argument หรือเก็บใน Hash/Array ได้เหมือน object ทั่วไป

```ruby
operations = {
  add: ->(a, b) { a + b },
  subtract: ->(a, b) { a - b },
  multiply: ->(a, b) { a * b }
}

puts operations[:add].call(10, 5)       # => 15
puts operations[:subtract].call(10, 5)  # => 5
puts operations[:multiply].call(10, 5)  # => 50
```

รูปแบบนี้ (เก็บ Lambda ไว้ใน Hash แล้วเรียกตามชื่อ key) เป็นเทคนิคที่ใช้แทน `case/when` หรือ
`if/elsif` ยาวๆ ได้อย่างสวยงาม และจะเจอบ่อยเมื่อไปเรียน design pattern อย่าง Strategy
Pattern ใน Part 016

---

## Step 78: Proc vs Lambda — arity และพฤติกรรมของ `return`

แม้ Proc กับ Lambda จะเป็น class เดียวกัน แต่มีความต่างสำคัญ 2 เรื่องที่ **ต้องจำให้แม่น**
เพราะเป็นสาเหตุของบั๊กที่พบบ่อยมากสำหรับมือใหม่

### ความต่างที่ 1: ความเข้มงวดเรื่องจำนวน argument (arity)

```ruby
# frozen_string_literal: true

# Proc: ยืดหยุ่น ไม่สนใจจำนวน argument ที่ไม่ตรง
my_proc = proc { |a, b| p [a, b] }
my_proc.call(1)         # => [1, nil]        ไม่ error
my_proc.call(1, 2, 3)   # => [1, 2]          argument เกินถูกทิ้ง ไม่ error

# Lambda: เข้มงวดเหมือน method ทั่วไป argument ต้องตรงเป๊ะ
my_lambda = ->(a, b) { p [a, b] }
my_lambda.call(1, 2)    # => [1, 2]          ปกติ
begin
  my_lambda.call(1)     # จำนวน argument ไม่ตรง
rescue ArgumentError => e
  puts "เกิด error: #{e.message}"
end
# => เกิด error: wrong number of arguments (given 1, expected 2)
```

**สรุป:** Lambda ตรวจสอบ arity เข้มงวดเหมือนเรียก method ปกติ ส่วน Proc (และ block) จะ
"ปล่อยผ่าน" แล้วเติม `nil` ให้ argument ที่ขาด หรือทิ้ง argument ที่เกิน

### ความต่างที่ 2: พฤติกรรมของ `return` ภายใน (สำคัญที่สุด และเป็นสาเหตุบั๊กบ่อยที่สุด)

```ruby
def test_lambda_return
  my_lambda = -> { return 10 }
  result = my_lambda.call
  puts "หลังเรียก lambda: #{result}"   # บรรทัดนี้ถูกรันแน่นอน
  "ค่าสุดท้ายของ method"
end

puts test_lambda_return
# => หลังเรียก lambda: 10
# => ค่าสุดท้ายของ method
```

`return` ภายใน Lambda ทำหน้าที่เหมือน `return` ใน method ธรรมดา — คือ **คืนค่าออกจาก
Lambda เท่านั้น** แล้วโค้ดที่เหลือใน method แม่ก็ทำงานต่อไปตามปกติ

```ruby
def test_proc_return
  my_proc = proc { return 10 }
  result = my_proc.call
  puts "หลังเรียก proc: #{result}"   # บรรทัดนี้ "ไม่มีทางถูกรัน"
  "ค่าสุดท้ายของ method"
end

puts test_proc_return
# => 10
# (ไม่มีบรรทัด "หลังเรียก proc" ปรากฏเลย เพราะ return ใน Proc ทำให้ method ทั้งก้อน
#  จบการทำงานทันที ไม่ใช่แค่จบ Proc)
```

`return` ภายใน Proc (และภายใน block) จะพยายาม **return ออกจาก method ที่ล้อมรอบมันอยู่**
ไม่ใช่แค่ออกจาก Proc — นี่คือกับดักที่อันตรายมาก โดยเฉพาะถ้า Proc นั้นถูกเรียกใช้นอก
method ที่มันถูกสร้างขึ้น (เช่น เก็บไว้ในตัวแปร global หรือส่งออกไปนอก scope) จะทำให้เกิด
`LocalJumpError: unexpected return`

```ruby
def create_broken_proc
  proc { return "escaping!" }
end

leaked_proc = create_broken_proc
begin
  leaked_proc.call
rescue LocalJumpError => e
  puts "เกิด error: #{e.message}"
end
# => เกิด error: unexpected return
```

### ตารางสรุปความแตกต่าง

| หัวข้อ | Block | Proc | Lambda |
|--------|-------|------|--------|
| เป็น object เก็บในตัวแปรได้? | ไม่ได้ (ต้องแปลงเป็น Proc ก่อน) | ได้ | ได้ |
| ตรวจสอบ arity เข้มงวดไหม | ไม่ (ยืดหยุ่น) | ไม่ (ยืดหยุ่น) | ใช่ (เข้มงวดเหมือน method) |
| `return` ภายในทำอะไร | return ออกจาก method ที่ล้อมรอบ | return ออกจาก method ที่ล้อมรอบ | return ออกจาก lambda เอง |
| สร้างด้วย | `do...end` / `{}` แนบท้าย method call | `Proc.new { }`, `proc { }` | `lambda { }`, `->() { }` |
| `.lambda?` | — | `false` | `true` |

**คำแนะนำเชิงปฏิบัติ:** ถ้าต้องการพฤติกรรมแบบ method ทั่วไป (arity เข้มงวด, `return`
ปลอดภัย) ให้ใช้ **Lambda** ถ้าต้องการความยืดหยุ่นแบบ block (เช่นใช้เป็น callback ที่ไม่รู้
ล่วงหน้าว่าจะถูกเรียกด้วย argument กี่ตัว) ให้ใช้ **Proc** ในโค้ด Rails จริง เราจะเห็น
Lambda ถูกใช้บ่อยกว่ามาก โดยเฉพาะใน scope (`scope :active, -> { where(active: true) }`)
และ validation (`validates :email, format: { with: -> (obj) { ... } }`)

---

## Step 79: Closures — การจับตัวแปร (variable capture) และตัวอย่าง counter generator

**Closure** คือคุณสมบัติที่ block, Proc และ Lambda ทุกตัวมีเหมือนกัน คือมัน "จดจำ" ตัวแปร
ท้องถิ่น (local variable) จาก scope ที่มันถูกสร้างขึ้นเอาไว้ได้ แม้ว่า scope นั้นจะจบไปแล้ว
ก็ตาม — เรียกว่า Proc/Lambda "ปิดล้อม" (close over) ตัวแปรเหล่านั้นไว้

```ruby
# frozen_string_literal: true

def make_greeter(greeting)
  # greeting เป็น local variable ของ make_greeter
  # lambda ด้านล่างถูกสร้างขึ้นภายใน scope นี้ จึง "จำ" greeting ไว้ได้ตลอดไป
  ->(name) { "#{greeting}, #{name}!" }
end

hello_greeter = make_greeter("Hello")
sawasdee_greeter = make_greeter("สวัสดี")

puts hello_greeter.call("Ruby")       # => Hello, Ruby!
puts sawasdee_greeter.call("Ruby")    # => สวัสดี, Ruby!
```

แม้ `make_greeter` จะรันจบไปแล้ว (return ค่าออกมาแล้ว) แต่ตัวแปร `greeting` ยัง "มีชีวิตอยู่"
ต่อไปภายใน lambda ที่ถูกคืนออกมา — และที่สำคัญ `hello_greeter` กับ `sawasdee_greeter`
**แต่ละตัวมีชุดตัวแปรของตัวเอง แยกจากกันอย่างสมบูรณ์** เพราะแต่ละครั้งที่เรียก
`make_greeter` คือการสร้าง scope ใหม่ทุกครั้ง

### ตัวอย่างคลาสสิก: Counter Generator

นี่คือตัวอย่างที่แสดงพลังของ closure ได้ชัดเจนที่สุด — การสร้าง "ตัวนับ" ที่จดจำสถานะของ
ตัวเองได้ โดยไม่ต้องใช้ instance variable หรือ class เลย

```ruby
def make_counter(start = 0)
  count = start

  increment = -> { count += 1 }
  decrement = -> { count -= 1 }
  current   = -> { count }

  { increment: increment, decrement: decrement, current: current }
end

counter_a = make_counter
counter_b = make_counter(100)

counter_a[:increment].call
counter_a[:increment].call
counter_a[:increment].call
puts counter_a[:current].call
# => 3

counter_b[:increment].call
counter_b[:decrement].call
puts counter_b[:current].call
# => 100

# counter_a และ counter_b ไม่ยุ่งเกี่ยวกันเลย เพราะแต่ละตัวมี "count" ของตัวเอง
puts counter_a[:current].call
# => 3 (ไม่เปลี่ยนแปลงจากการเรียก counter_b)
```

ตัวแปร `count` ถูก "ปิดล้อม" ไว้ภายใน 3 lambda (`increment`, `decrement`, `current`) และ
ทั้ง 3 ตัวใช้ตัวแปร `count` **ตัวเดียวกัน** (share สภาพแวดล้อมเดียวกัน) แต่ `count` นี้
**มองไม่เห็นจากภายนอก** เลย — ไม่มีทางเข้าถึง `count` โดยตรงได้ นอกจากผ่าน lambda ที่เรา
ตั้งใจ expose ออกมาเท่านั้น นี่คือการทำ **encapsulation** แบบไม่ต้องใช้ class เลยด้วยซ้ำ

### ระวัง: closure จับ "ตัวแปร" ไม่ใช่ "ค่า" ณ ขณะสร้าง

กับดักที่พบบ่อยมากคือการสร้าง closure ใน loop แล้วคาดหวังว่าแต่ละตัวจะจำค่าที่ต่างกัน

```ruby
# ตัวอย่างที่ "ดูเหมือนจะ" ทำงานถูก — แต่ต้องเข้าใจให้ถูกต้อง
callbacks = []

[1, 2, 3].each do |n|
  callbacks << -> { puts "ค่าคือ #{n}" }
end

callbacks.each(&:call)
# => ค่าคือ 1
# => ค่าคือ 2
# => ค่าคือ 3
```

โค้ดข้างบนทำงานถูกต้อง เพราะ `n` เป็น **block parameter ใหม่ในทุกรอบของ `each`**
(แต่ละรอบสร้างตัวแปร `n` ใหม่ ไม่ได้ใช้ตัวแปรตัวเดียวกันซ้ำ) จึงต่างจากบางภาษา (เช่น
JavaScript แบบเก่าที่ใช้ `var` ใน `for` loop) ที่จะเจอปัญหาตัวแปรถูก share กันทุก closure
ใน Ruby เรื่องนี้ปลอดภัยกว่ามากเพราะ block parameter ถูกสร้างใหม่ (scoped ใหม่) ทุกครั้งที่
`yield` ถูกเรียก

---

## Step 80: แบบฝึกหัด Part นี้ — Retry-with-block utility และ Event Callback System

### โจทย์

สร้างไฟล์ `resilient_task.rb` ที่มี 2 ส่วนดังนี้:

**ส่วนที่ 1 — `with_retry`:** เขียน method `with_retry(max_attempts:, delay: 0)` ที่รับ block
เป็นงานที่ต้องการรัน โดย:

- ถ้า block ทำงานสำเร็จ (ไม่ throw exception) ให้คืนค่าผลลัพธ์ของ block ทันที
- ถ้า block เกิด exception ให้ลองรันใหม่ (retry) จนกว่าจะครบ `max_attempts` ครั้ง
- ถ้าลองครบจำนวนครั้งแล้วยัง fail ให้ throw exception ตัวสุดท้ายออกไป
- ทุกครั้งที่ลองใหม่ ให้ print ข้อความแจ้งเตือนบอกว่ากำลังลองครั้งที่เท่าไหร่

**ส่วนที่ 2 — `EventEmitter`:** เขียน class `EventEmitter` ที่เก็บ callback (Proc/Lambda หรือ
block) ผูกกับชื่อ event ได้หลายตัว (`on`) และเรียก callback ทั้งหมดที่ผูกกับ event นั้นเมื่อ
ถูก trigger (`emit`)

### เฉลย

```ruby
# frozen_string_literal: true

# resilient_task.rb

# ---------- ส่วนที่ 1: with_retry ----------
def with_retry(max_attempts:, delay: 0)
  attempt = 1

  begin
    yield(attempt)
  rescue StandardError => e
    if attempt < max_attempts
      puts "  [retry] พยายามครั้งที่ #{attempt} ล้มเหลว (#{e.message}) — ลองใหม่..."
      attempt += 1
      sleep(delay) if delay.positive?
      retry
    else
      puts "  [retry] ลองครบ #{max_attempts} ครั้งแล้ว ยังล้มเหลว — ยอมแพ้"
      raise
    end
  end
end

# ทดสอบ 1: งานที่สำเร็จตั้งแต่ครั้งแรก
puts "=== ทดสอบ 1: งานสำเร็จทันที ==="
result = with_retry(max_attempts: 3) { |attempt| "สำเร็จในความพยายามครั้งที่ #{attempt}" }
puts result
# => === ทดสอบ 1: งานสำเร็จทันที ===
# => สำเร็จในความพยายามครั้งที่ 1

# ทดสอบ 2: งานที่ fail 2 ครั้งแรก แล้วสำเร็จครั้งที่ 3
puts "\n=== ทดสอบ 2: fail 2 ครั้งแรก แล้วสำเร็จ ==="
counter = 0
result = with_retry(max_attempts: 5) do |attempt|
  counter += 1
  raise "เครือข่ายขัดข้อง" if counter < 3

  "สำเร็จในความพยายามครั้งที่ #{attempt}"
end
puts result
# => === ทดสอบ 2: fail 2 ครั้งแรก แล้วสำเร็จ ===
# =>   [retry] พยายามครั้งที่ 1 ล้มเหลว (เครือข่ายขัดข้อง) — ลองใหม่...
# =>   [retry] พยายามครั้งที่ 2 ล้มเหลว (เครือข่ายขัดข้อง) — ลองใหม่...
# => สำเร็จในความพยายามครั้งที่ 3

# ทดสอบ 3: งานที่ fail ตลอด จนหมดจำนวนครั้ง
puts "\n=== ทดสอบ 3: fail ตลอด จนหมดโควตา ==="
begin
  with_retry(max_attempts: 2) { raise "เซิร์ฟเวอร์ไม่ตอบสนอง" }
rescue StandardError => e
  puts "จับ exception สุดท้ายได้: #{e.message}"
end
# => === ทดสอบ 3: fail ตลอด จนหมดโควตา ===
# =>   [retry] พยายามครั้งที่ 1 ล้มเหลว (เซิร์ฟเวอร์ไม่ตอบสนอง) — ลองใหม่...
# =>   [retry] ลองครบ 2 ครั้งแล้ว ยังล้มเหลว — ยอมแพ้
# => จับ exception สุดท้ายได้: เซิร์ฟเวอร์ไม่ตอบสนอง

# ---------- ส่วนที่ 2: EventEmitter ----------
class EventEmitter
  def initialize
    @listeners = Hash.new { |hash, key| hash[key] = [] }
  end

  # ผูก callback เข้ากับชื่อ event — รับได้ทั้ง block และ Proc/Lambda ที่ส่งผ่าน argument
  def on(event_name, callback = nil, &block)
    handler = callback || block
    raise ArgumentError, "ต้องส่ง callback หรือ block มาอย่างใดอย่างหนึ่ง" unless handler

    @listeners[event_name] << handler
    self
  end

  # เรียก callback ทุกตัวที่ผูกกับ event นี้ พร้อมส่ง argument ต่อให้ทุกตัว
  def emit(event_name, *args)
    @listeners[event_name].each { |handler| handler.call(*args) }
  end

  def listener_count(event_name)
    @listeners[event_name].length
  end
end

puts "\n=== ทดสอบ EventEmitter ==="
emitter = EventEmitter.new

# ผูก listener หลายตัวเข้ากับ event เดียวกันด้วย block
emitter.on(:user_registered) do |name|
  puts "[Email] ส่งอีเมลต้อนรับให้ #{name}"
end

emitter.on(:user_registered) do |name|
  puts "[Log]   บันทึก log: ผู้ใช้ใหม่ชื่อ #{name} สมัครสมาชิก"
end

# ผูก listener ด้วย Lambda ที่สร้างไว้ก่อนแล้ว
notify_admin = ->(name) { puts "[Admin] แจ้งเตือนแอดมิน: มีผู้ใช้ใหม่ #{name}" }
emitter.on(:user_registered, notify_admin)

puts "จำนวน listener ของ :user_registered = #{emitter.listener_count(:user_registered)}"
emitter.emit(:user_registered, "สมชาย")
# => === ทดสอบ EventEmitter ===
# => จำนวน listener ของ :user_registered = 3
# => [Email] ส่งอีเมลต้อนรับให้ สมชาย
# => [Log]   บันทึก log: ผู้ใช้ใหม่ชื่อ สมชาย สมัครสมาชิก
# => [Admin] แจ้งเตือนแอดมิน: มีผู้ใช้ใหม่ สมชาย

# event ที่ไม่มีใคร subscribe ไว้ ก็แค่ไม่มีอะไรเกิดขึ้น ไม่ error
emitter.emit(:no_one_listens, "ข้อมูลอะไรก็ได้")
puts "จบการทดสอบ"
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `retry` ภายใน `rescue` — สั่งให้กลับไปเริ่มรัน `begin` block ใหม่ตั้งแต่ต้น (จะเรียนเจาะลึก
  เรื่อง `retry` และ exception handling เต็มรูปแบบใน Part 011)
- `Hash.new { |hash, key| hash[key] = [] }` — สร้าง Hash ที่มี **default block**: เมื่อเข้าถึง
  key ที่ยังไม่เคยมี จะรัน block นี้เพื่อสร้างค่าเริ่มต้น (Array ว่าง) ให้อัตโนมัติ แทนที่จะ
  ต้องเช็ค `@listeners[event_name] ||= []` เองทุกครั้ง — เป็นการใช้ block ผสมกับ Hash
  ที่ทรงพลังมาก
- `callback = nil, &block` แล้วเลือกใช้ `callback || block` — ทำให้ method รับได้ทั้ง
  "block ตรงๆ" และ "Proc/Lambda ที่ประกาศไว้ก่อนแล้วส่งเป็น argument"
- `handler.call(*args)` — ใช้ `*args` (splat) กระจาย argument ที่ `emit` รับมาส่งต่อให้
  callback ทุกตัวเหมือนกัน ไม่ว่า callback จะรับ argument กี่ตัว

รูปแบบ `EventEmitter` นี้คือแนวคิดเดียวกับที่ Rails ใช้ในหลายจุด เช่น
`ActiveSupport::Notifications`, callback ของ ActiveRecord (`after_create`, `before_save`)
และ Stimulus controller action ฝั่ง frontend — ทั้งหมดคือ "เก็บ block/callback ไว้ แล้ว
เรียกมันตอนเหตุการณ์บางอย่างเกิดขึ้น"

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม method `off(event_name, callback)` ให้ `EventEmitter` เพื่อยกเลิกการ subscribe
   listener ตัวใดตัวหนึ่งออกจาก event (ใบ้: ใช้ `Array#delete` กับ `@listeners[event_name]`)
2. แก้ `with_retry` ให้รับ `backoff:` เป็น Proc/Lambda ที่รับจำนวนครั้งที่พยายามแล้ว แล้วคืนค่า
   จำนวนวินาทีที่ควรรอก่อนลองใหม่ (เช่น exponential backoff `->(n) { 2**n }`) แทนที่จะใช้
   `delay` คงที่แบบเดิม
3. เขียน method `once(event_name, &block)` ให้ `EventEmitter` ที่ผูก listener ที่ **ทำงาน
   ได้แค่ครั้งเดียว** แล้วถูกลบออกจากรายการ listener อัตโนมัติหลังถูกเรียก 1 ครั้ง
   (ใบ้: ห่อ block เดิมด้วย Proc ใหม่ที่เรียก `off` ตัวเองหลังทำงานเสร็จ)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **block** คือโค้ดที่แนบไปกับการเรียก method โดยไม่ต้องตั้งชื่อ และรู้ธรรมเนียม
  การเลือกใช้ `do...end` กับ `{}` ตาม Ruby Style Guide
- ใช้ block กับ method มาตรฐานของ Ruby (`each`, `map`, `select`, `reduce`, `times`) ได้
  อย่างคล่องแคล่ว และเข้าใจว่าแต่ละ method ตีความค่าที่ block คืนกลับมาต่างกันอย่างไร
- ใช้ `yield` และ `block_given?` เขียน method ของตัวเองที่รับ block ได้ ทั้งแบบบังคับและ
  แบบเป็นตัวเลือก
- เขียน method จำลอง `each`/`map`/`select` ของตัวเอง และเข้าใจว่า `include Enumerable`
  ทำงานอย่างไรเบื้องหลัง
- แปลง block เป็น **Proc** ด้วย `&block`, สร้าง Proc ด้วย `Proc.new`/`proc`, และเรียกด้วย
  `.call`/`.()`/`[]`
- สร้าง **Lambda** ได้ทั้งแบบ `lambda do...end` และ stabby lambda `->() {}`
- แยกความแตกต่างระหว่าง Proc กับ Lambda ได้ชัดเจน ทั้งเรื่อง **arity** (เข้มงวด vs ยืดหยุ่น)
  และพฤติกรรมของ **`return`** (จบแค่ตัวเอง vs จบ method ที่ล้อมรอบ)
- เข้าใจ **closure** และการจับตัวแปร (variable capture) ผ่านตัวอย่าง counter generator
  และรู้วิธี encapsulate state โดยไม่ต้องใช้ class
- นำความรู้ทั้งหมดมาประยุกต์สร้าง `with_retry` (retry utility) และ `EventEmitter`
  (callback system) ซึ่งเป็นรูปแบบที่ใช้จริงในโค้ด production ระดับมืออาชีพ

**ต่อไป (Part 009):** เราจะเข้าสู่โลกของ **Object-Oriented Programming (OOP)** ใน Ruby
อย่างเป็นทางการ — เรียนรู้การนิยาม `class`, การสร้าง `Object`, `attr_accessor`/`attr_reader`/
`attr_writer`, method พิเศษ `initialize`, และความแตกต่างระหว่าง **instance variable**
(`@name`) กับ **class variable** (`@@count`) ซึ่งเป็นรากฐานสำคัญที่สุดก่อนไปเรียน Rails
เพราะทุกอย่างใน Rails (Model, Controller, View) ล้วนสร้างจาก class ทั้งสิ้น
