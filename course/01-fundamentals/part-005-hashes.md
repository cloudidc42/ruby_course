# Part 005: Hash — การสร้าง, Iteration, Nested Hash, Symbol vs String Keys

> **Step ครอบคลุมใน Part นี้:** Step 41–50
> **ระดับ:** เริ่มต้น–กลาง (ต้องผ่าน Part 004 เรื่อง Array มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 41: Hash คืออะไร และการสร้างด้วย Hash literal `{}`
- Step 42: `Hash.new` พร้อม default value และ default block
- Step 43: Symbol keys vs String keys และ `key:` shorthand syntax
- Step 44: การเข้าถึงและแก้ไขค่าใน Hash (`[]`, `[]=`, `fetch`, `store`, `delete`, `key?`)
- Step 45: การวนซ้ำ Hash: `each`, `each_pair`, `each_key`, `each_value`
- Step 46: `map` บน Hash (ทำไมได้ Array กลับมา) และการแปลงกลับด้วย `to_h`
- Step 47: `select`, `reject`, `filter_map` บน Hash
- Step 48: `merge` และ `merge!` — การรวม Hash และการแก้ conflict ด้วย block
- Step 49: Nested Hash, method `dig`, และ safe navigation `&.`
- Step 50: `transform_keys`, `transform_values`, `to_a`, และแบบฝึกหัดโปรเจกต์: แปลง config ซ้อนกัน

---

## Step 41: Hash คืออะไร และการสร้างด้วย Hash literal `{}`

**Hash** คือโครงสร้างข้อมูลที่เก็บคู่ **key-value** (คีย์-ค่า) เทียบเท่ากับ `Object`/`dict` ใน
JavaScript/Python หรือ `HashMap` ใน Java จุดเด่นคือค้นหาค่าจาก key ได้เร็วมาก (ประมาณ O(1))
ต่างจาก Array ที่ต้องรู้ตำแหน่ง (index) ก่อนถึงจะเข้าถึงข้อมูลได้

ทำไมต้องมี Hash ทั้งที่มี Array แล้ว: ลองนึกภาพเก็บข้อมูลผู้ใช้คนหนึ่ง — ถ้าใช้ Array
`["สมชาย", 25, "กรุงเทพ"]` เราต้องจำเองว่า index 0 คือชื่อ, index 1 คืออายุ, index 2 คือจังหวัด
ซึ่งอ่านโค้ดแล้วไม่รู้ความหมาย แต่ถ้าใช้ Hash `{ name: "สมชาย", age: 25, city: "กรุงเทพ" }`
โค้ดจะสื่อความหมายในตัวเองทันที นี่คือเหตุผลที่ Hash ถูกใช้เป็น "โครงสร้างข้อมูลเริ่มต้น"
สำหรับการส่งข้อมูลที่มีหลายฟิลด์ในโลก Ruby/Rails (เช่น `params` ใน Rails controller ก็คือ
Hash-like object)

### สร้าง Hash ด้วย literal `{}`

```ruby
# Hash ว่าง
empty_hash = {}
p empty_hash          # => {}
p empty_hash.class     # => Hash

# Hash ที่มี key เป็น Symbol (แนะนำให้ใช้แบบนี้เป็นหลัก จะอธิบายเหตุผลใน Step 43)
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }
p person
# => {name: "สมชาย", age: 25, city: "กรุงเทพ"}

# Hash ที่มี key เป็น String
person_string_key = { "name" => "สมหญิง", "age" => 30 }
p person_string_key
# => {"name" => "สมหญิง", "age" => 30}
```

**สังเกต syntax สองแบบ:**

- `key: value` — เรียกว่า **modern hash syntax** (บางทีเรียก "JSON-like syntax") ใช้ได้เฉพาะ
  เมื่อ key เป็น Symbol และเขียนเป็นชื่อ method/variable ได้ (ไม่มีช่องว่างหรืออักขระพิเศษ)
- `"key" => value` — เรียกว่า **hash rocket syntax** (สัญลักษณ์ `=>` หน้าตาเหมือนลูกศร)
  ใช้ได้กับ key ทุกชนิด (String, Integer, Array, ฯลฯ) ไม่ใช่แค่ Symbol

```ruby
# hash rocket ใช้กับ key ชนิดใดก็ได้
mixed_key_hash = {
  "string_key" => 1,
  :symbol_key => 2,
  42 => "answer",
  [1, 2] => "array as key"
}
p mixed_key_hash[42]         # => "answer"
p mixed_key_hash[[1, 2]]     # => "array as key"
```

> **ข้อควรระวัง:** ถ้า key เป็น Symbol แต่มีอักขระพิเศษ (เช่นมีเครื่องหมาย `-`) จะใช้ syntax
> `key:` ไม่ได้ ต้องใช้ hash rocket แทน เช่น `{ :"weird-key" => 1 }`
> แต่กรณีแบบนี้เจอน้อยมากในทางปฏิบัติ

### ตรวจสอบ key และ value ของ Hash

```ruby
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }

p person.keys        # => [:name, :age, :city]   (ได้ Array ของ key ทั้งหมด)
p person.values       # => ["สมชาย", 25, "กรุงเทพ"]  (ได้ Array ของ value ทั้งหมด)
p person.size         # => 3   (จำนวนคู่ key-value)
p person.length       # => 3   (alias ของ size)
p person.empty?       # => false
p({}.empty?)          # => true
```

`keys` และ `values` มีลำดับตรงกันเสมอ (index เดียวกันของทั้งสอง Array คือคู่กัน) เพราะ Ruby
Hash **รักษาลำดับการใส่ข้อมูล (insertion order)** ไว้เสมอตั้งแต่ Ruby 1.9 เป็นต้นมา —
ต่างจากภาษาอื่นบางภาษาที่ Hash/Dict ไม่รับประกันลำดับ

---

## Step 42: `Hash.new` พร้อม default value และ default block

นอกจาก literal `{}` แล้ว เราสร้าง Hash ได้ด้วย `Hash.new` ซึ่งมีความสามารถพิเศษที่ literal
ทำไม่ได้ คือการกำหนด **ค่า default** เมื่อเข้าถึง key ที่ไม่มีอยู่จริง

### พฤติกรรม default ของ Hash ธรรมดา

```ruby
scores = { math: 90, english: 85 }
p scores[:math]      # => 90
p scores[:science]   # => nil   (key ที่ไม่มีอยู่ จะได้ nil ไม่ error)
```

การได้ `nil` เงียบๆ บางครั้งเป็นปัญหา เพราะทำให้บั๊กซ่อนอยู่ในโค้ดได้ (เช่นถ้าเอา `nil` ไปบวก
เลขต่อจะได้ `NoMethodError` ที่จุดอื่นซึ่งตามรอยยาก) `Hash.new` ช่วยให้เรากำหนดพฤติกรรม
default ได้ชัดเจนขึ้น

### `Hash.new(default_value)` — ค่า default แบบคงที่

```ruby
# ตั้งค่า default เป็น 0 แทน nil
counter = Hash.new(0)
p counter[:apple]   # => 0  (ยังไม่เคยเก็บค่า แต่ได้ 0 แทน nil)

# ประโยชน์ชัดเจนที่สุด: การนับความถี่ (frequency count)
words = %w[apple banana apple orange banana apple]
count = Hash.new(0)
words.each { |word| count[word] += 1 }
p count
# => {"apple" => 3, "banana" => 2, "orange" => 1}
```

ถ้าไม่ใช้ `Hash.new(0)` เราต้องเขียนโค้ดยาวขึ้นแบบนี้:

```ruby
# แบบไม่ใช้ default value ต้องเช็คเองทุกครั้ง
words = %w[apple banana apple orange banana apple]
count = {}
words.each do |word|
  count[word] = count[word] ? count[word] + 1 : 0 + 1
end
p count
```

จะเห็นว่า `Hash.new(0)` ทำให้โค้ดสั้นและอ่านง่ายขึ้นมาก — เป็น pattern ที่พบบ่อยมากในการเขียน
Ruby ระดับมืออาชีพ

> **ข้อควรระวังสำคัญ:** ห้ามใช้ `Hash.new([])` หรือ `Hash.new({})` เพราะ default value
> ตัวเดียวกัน (object เดียวกันใน memory) จะถูกแชร์ใช้ร่วมกันทุก key ที่ยังไม่เคยตั้งค่า
> ทำให้เกิดบั๊กที่ตามยากมาก:

```ruby
# ผิด! ระวังกับดักนี้
wrong = Hash.new([])
wrong[:fruits] << "apple"
wrong[:vegetables] << "carrot"
p wrong[:fruits]       # => ["apple", "carrot"]  (เอ๊ะ! ทำไมมี carrot ด้วย)
p wrong[:vegetables]   # => ["apple", "carrot"]  (นี่ก็ผิด)
# เพราะ Array [] ตัวเดียวกันถูกใช้เป็น default ของทุก key ที่ยังไม่มีค่า
# การ << ใส่ Array นั้นเลยกระทบทุก key พร้อมกัน
```

### `Hash.new { |hash, key| ... }` — default block (คำนวณ default แบบ dynamic)

วิธีแก้ปัญหาด้านบน และวิธีที่ทรงพลังกว่า คือใช้ **default block** ซึ่งจะรันทุกครั้งที่มีการ
เข้าถึง key ที่ยังไม่มีอยู่ และสามารถสร้าง object ใหม่ทุกครั้งได้

```ruby
# ถูกต้อง! ใช้ default block เพื่อสร้าง Array ใหม่ทุกครั้งที่เจอ key ใหม่
correct = Hash.new { |hash, key| hash[key] = [] }
correct[:fruits] << "apple"
correct[:vegetables] << "carrot"
p correct[:fruits]       # => ["apple"]
p correct[:vegetables]   # => ["carrot"]   (แยกจากกันถูกต้อง)
```

**อธิบายกลไก:** block รับ 2 argument คือ `hash` (ตัว Hash เอง) และ `key` (key ที่ถูกเข้าถึงแต่
ยังไม่มีอยู่) การเขียน `hash[key] = []` ภายใน block คือการ "สร้างและบันทึก" ค่า default ใหม่
ลงใน Hash จริงๆ ทันทีที่ key นั้นถูกเข้าถึงครั้งแรก (เทคนิคนี้เรียกว่า **autovivification**)

ตัวอย่างการใช้งานจริง: จัดกลุ่มคำตาม length

```ruby
words = %w[cat dog elephant ant bee lion tiger]
grouped = Hash.new { |hash, key| hash[key] = [] }

words.each do |word|
  grouped[word.length] << word
end

p grouped
# => {3 => ["cat", "dog", "ant", "bee"], 8 => ["elephant"], 4 => ["lion"], 5 => ["tiger"]}
```

> **หมายเหตุ:** งานแบบนี้ Ruby มี method สำเร็จรูปชื่อ `group_by` ที่ทำให้สั้นกว่านี้อีก
> (`words.group_by(&:length)`) จะสอนละเอียดใน Part 013 (Enumerable ขั้นสูง) แต่การเข้าใจ
> `Hash.new` with block ก่อน จะช่วยให้เข้าใจว่า `group_by` ทำงานอย่างไรเบื้องหลัง

---

## Step 43: Symbol keys vs String keys และ `key:` shorthand syntax

นี่คือหัวข้อที่มือใหม่ Ruby/Rails มักสับสน: **เมื่อไหร่ควรใช้ Symbol เป็น key เมื่อไหร่ควรใช้
String**

### ทบทวน: Symbol กับ String ต่างกันอย่างไร (จาก Part 002)

```ruby
p :name.object_id == :name.object_id           # => true  (Symbol เดียวกันคือ object เดียวกัน)
p "name".object_id == "name".object_id          # => false (String คนละตัวคือคนละ object เสมอ)
```

Symbol เป็น **immutable** (แก้ไขค่าไม่ได้) และ Ruby เก็บ Symbol ค่าเดียวกันไว้เป็น object
เดียวกันเสมอใน memory (เรียกว่า interned) ในขณะที่ String ทุกครั้งที่สร้างใหม่จะเป็นคนละ
object กัน (แม้ค่าจะเหมือนกัน)

### ทำไม Ruby/Rails community นิยม Symbol key มากกว่า String key

1. **ประสิทธิภาพ (Performance)** — เพราะ Symbol เดียวกันคือ object เดียวกันเสมอ การเทียบ
   ความเท่ากัน (`==`) และการคำนวณ hash code (`hash`) ทำได้เร็วกว่า String มาก โดยเฉพาะเมื่อ
   Hash มีขนาดใหญ่หรือถูกเข้าถึงบ่อยๆ (เช่น `params` ใน request ของเว็บที่มีทราฟฟิกสูง)

2. **ประหยัด memory** — Symbol `:name` ถูกสร้างครั้งเดียวและใช้ซ้ำได้ตลอดโปรแกรม ในขณะที่
   String `"name"` แต่ละครั้งที่พิมพ์ (ถ้าไม่ใช่ frozen string literal) จะจองหน่วยความจำใหม่

3. **สื่อความหมายว่าเป็น "identifier" ไม่ใช่ "ข้อมูล"** — ในทางความหมาย (semantic) Symbol
   สื่อว่านี่คือ "ชื่อ/label ที่คงที่" (เช่น ชื่อฟิลด์ของ record) ในขณะที่ String สื่อว่านี่คือ
   "ข้อมูลที่แปรผันได้" (เช่น ข้อความที่ผู้ใช้พิมพ์) — Rails ทั้ง framework ยึดธรรมเนียมนี้:
   `params[:id]`, `attributes[:name]`, `options[:method]` ล้วนใช้ Symbol เป็น key แทบทั้งหมด

4. **ธรรมเนียมของวงการ (community convention)** — เมื่อทุกคนใช้ Symbol key เหมือนกัน โค้ด
   จะอ่านง่ายและเข้ากันได้กับ library อื่นๆ ในวงการ Ruby/Rails ได้ทันที

```ruby
# แนวปฏิบัติที่แนะนำ: ใช้ Symbol key เมื่อ key เป็น "ชื่อฟิลด์ที่รู้ล่วงหน้า" (เขียนตายตัวในโค้ด)
user = { name: "สมชาย", email: "somchai@example.com", role: :admin }

# ใช้ String key เมื่อ key มาจากภายนอกโปรแกรม เช่น จากไฟล์ JSON, จากผู้ใช้กรอก, จาก API
# (เพราะ JSON ไม่มีแนวคิด Symbol — JSON.parse จะได้ String key เสมอโดย default)
require "json"
json_data = '{"name": "สมหญิง", "age": 30}'
parsed = JSON.parse(json_data)
p parsed          # => {"name" => "สมหญิง", "age" => 30}   <- String key
p parsed.class     # => Hash

# ถ้าต้องการ Symbol key จาก JSON ต้องแปลงเอง (เดี๋ยวเรียนวิธีทำใน Step 50)
parsed_symbolized = JSON.parse(json_data, symbolize_names: true)
p parsed_symbolized   # => {:name => "สมหญิง", :age => 30}
```

### `key:` shorthand syntax (Ruby 3.1+) — เมื่อชื่อตัวแปรตรงกับชื่อ key

Ruby 3.1 เพิ่ม syntax ย่อที่สะดวกมาก: ถ้าเรามีตัวแปรที่ชื่อตรงกับ key ที่ต้องการ ไม่ต้องเขียน
`key: value` ซ้ำ เขียนแค่ `key:` เฉยๆ ได้เลย (คล้าย object shorthand property ใน JavaScript)

```ruby
name = "สมชาย"
age = 25
city = "กรุงเทพ"

# แบบเดิม (เขียนซ้ำชื่อตัวแปร)
person_old_style = { name: name, age: age, city: city }
p person_old_style

# แบบ shorthand (Ruby 3.1+) — สั้นและอ่านง่ายกว่า
person_shorthand = { name:, age:, city: }
p person_shorthand
# => {name: "สมชาย", age: 25, city: "กรุงเทพ"}
```

Shorthand นี้มีประโยชน์มากในทางปฏิบัติ เพราะโค้ดจริงมักมีรูปแบบ "รับตัวแปรมาแล้วประกอบเป็น
Hash เพื่อส่งต่อ" อยู่บ่อยมาก (เช่นตอนสร้าง record ใน Rails: `User.new(name:, email:)`)

> **หมายเหตุเรื่องเวอร์ชัน:** shorthand `key:` ใช้ได้ตั้งแต่ Ruby 3.1 เป็นต้นไป หลักสูตรนี้ใช้
> Ruby 3.3.x จึงใช้ได้แน่นอน แต่ถ้าไปเจอโค้ด Ruby เวอร์ชันเก่ากว่านี้ (2.x) จะไม่มี syntax นี้
> ต้องเขียน `key: key` แบบเต็มเสมอ

### แปลง String key ↔ Symbol key

```ruby
string_keyed = { "name" => "สมชาย", "age" => 25 }

# แปลง String key เป็น Symbol key ทั้งหมด
symbol_keyed = string_keyed.transform_keys(&:to_sym)
p symbol_keyed   # => {name: "สมชาย", age: 25}

# แปลงกลับ Symbol key เป็น String key
back_to_string = symbol_keyed.transform_keys(&:to_s)
p back_to_string # => {"name" => "สมชาย", "age" => 25}
```

(`transform_keys` จะอธิบายละเอียดใน Step 50 — ตรงนี้ยกมาให้เห็นภาพก่อนว่าแปลงไปมาได้ง่าย)

---

## Step 44: การเข้าถึงและแก้ไขค่าใน Hash

### เข้าถึงค่าด้วย `[]`

```ruby
person = { name: "สมชาย", age: 25 }

p person[:name]     # => "สมชาย"
p person[:email]     # => nil   (key ไม่มีอยู่ ได้ nil เงียบๆ ไม่ error)
```

### `fetch` — เข้าถึงค่าแบบเข้มงวด (strict) พร้อมควบคุม default เอง

ปัญหาของ `[]` คือได้ `nil` เงียบๆ เมื่อ key ไม่มีอยู่ ทำให้บางครั้งเราไม่รู้ว่า "ค่าคือ nil จริงๆ"
หรือ "key นี้สะกดผิด/ไม่มีอยู่กันแน่" `fetch` แก้ปัญหานี้โดย throw error ทันทีถ้าไม่พบ key
(เว้นแต่จะระบุ default เอง)

```ruby
person = { name: "สมชาย", age: 25 }

p person.fetch(:name)              # => "สมชาย"
p person.fetch(:email, "ไม่ระบุ")   # => "ไม่ระบุ"  (ระบุ default value เป็น argument ที่ 2)

# ถ้าไม่ระบุ default และ key ไม่มีอยู่ จะได้ KeyError ทันที
begin
  person.fetch(:email)
rescue KeyError => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
  # => เกิดข้อผิดพลาด: key not found: :email
end

# fetch รับ block เป็น default ได้เช่นกัน (เรียกเฉพาะตอนไม่พบ key เท่านั้น - lazy evaluation)
p person.fetch(:email) { "ไม่พบอีเมลของ #{person[:name]}" }
# => "ไม่พบอีเมลของสมชาย"
```

**กฎการเลือกใช้ในทางปฏิบัติ:** ใช้ `fetch` เมื่อ key นั้น "ควรมีอยู่เสมอ" และถ้าไม่มีคือสัญญาณ
ว่าโค้ดมีบั๊ก (fail fast แทนที่จะปล่อยให้ `nil` ไหลต่อไปทำให้ error โผล่ที่อื่นซึ่งตามยากกว่า)
ใช้ `[]` เมื่อ key นั้น "ไม่มีก็ได้เป็นปกติ" และเราจัดการ `nil` เองอยู่แล้ว

### แก้ไข/เพิ่มค่าด้วย `[]=` และ `store`

```ruby
person = { name: "สมชาย" }

person[:age] = 25          # เพิ่ม key ใหม่
person[:name] = "สมหญิง"    # แก้ไขค่าของ key ที่มีอยู่แล้ว
p person   # => {name: "สมหญิง", age: 25}

# store เป็น alias ของ []= (ทำงานเหมือนกันทุกประการ)
person.store(:city, "กรุงเทพ")
p person   # => {name: "สมหญิง", age: 25, city: "กรุงเทพ"}
```

### ตรวจสอบว่ามี key/value อยู่หรือไม่

```ruby
person = { name: "สมชาย", age: 25 }

p person.key?(:name)        # => true
p person.has_key?(:email)   # => false   (has_key? เป็น alias ของ key?)
p person.include?(:age)     # => true    (include? ก็เป็น alias ของ key? เช่นกัน)
p person.member?(:age)      # => true    (member? ก็เป็น alias ของ key? เช่นกัน)

p person.value?(25)         # => true    (ตรวจสอบจาก value แทน key)
p person.has_value?(30)     # => false
```

> **ทำไมมี alias หลายชื่อ:** ทั้งหมดทำงานเหมือนกันทุกประการ มีไว้ให้เลือกใช้ชื่อที่อ่าน
> ลื่นไหลตามบริบท แต่ในทางปฏิบัติทีมงานควรตกลงกันใช้ชื่อเดียวให้สม่ำเสมอทั้งโปรเจกต์

### ลบ key ด้วย `delete`

```ruby
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }

removed_value = person.delete(:city)
p removed_value   # => "กรุงเทพ"   (delete คืนค่าที่ถูกลบออกมา)
p person          # => {name: "สมชาย", age: 25}

# ลบ key ที่ไม่มีอยู่ จะได้ nil (ไม่ error)
p person.delete(:not_exist)   # => nil

# delete รับ block เป็น default เมื่อไม่พบ key (คล้าย fetch)
p person.delete(:not_exist) { "ไม่มี key นี้อยู่แล้ว" }
# => "ไม่มี key นี้อยู่แล้ว"
```

### `values_at` — ดึงหลาย value พร้อมกัน

```ruby
person = { name: "สมชาย", age: 25, city: "กรุงเทพ", email: "somchai@example.com" }

p person.values_at(:name, :city)   # => ["สมชาย", "กรุงเทพ"]  (ได้ Array ตามลำดับที่ระบุ)
```

### `dup` — คัดลอก Hash (แบบ shallow copy)

```ruby
original = { name: "สมชาย", tags: ["ruby", "rails"] }
copy = original.dup

copy[:name] = "สมหญิง"
p original[:name]  # => "สมชาย"  (แก้ key ระดับบนสุด ไม่กระทบ original)

copy[:tags] << "web"      # แต่แก้ Array ข้างในโดยตรง (ไม่ได้แทนที่ทั้ง key)
p original[:tags]          # => ["ruby", "rails", "web"]   <- original เปลี่ยนไปด้วย!
```

> **ข้อควรระวัง (shallow copy):** `dup` คัดลอกแค่ชั้นบนสุด ส่วน value ที่เป็น object mutable
> เช่น Array/Hash ซ้อนกัน จะยังชี้ไปที่ object เดิมร่วมกันระหว่าง `original` กับ `copy` —
> เรื่องนี้สำคัญมากเมื่อทำงานกับ Nested Hash ใน Step 49

---

## Step 45: การวนซ้ำ Hash: `each`, `each_pair`, `each_key`, `each_value`

### `each` — วนซ้ำทีละคู่ key-value

```ruby
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }

person.each do |key, value|
  puts "#{key}: #{value}"
end
# => name: สมชาย
# => age: 25
# => city: กรุงเทพ
```

**สังเกต:** block ของ `each` บน Hash รับ 2 parameter (`key`, `value`) เพราะแต่ละ element ของ
Hash จริงๆ แล้วคือคู่ `[key, value]` (ในเชิง implementation คือ Array 2 ช่อง) — ถ้าเขียนรับแค่
parameter เดียว ก็ยังทำงานได้ แต่จะได้เป็น Array `[key, value]` แทน:

```ruby
person.each do |pair|
  p pair   # => [:name, "สมชาย"]  แล้วก็ [:age, 25]  แล้วก็ [:city, "กรุงเทพ"]
  puts "#{pair[0]} = #{pair[1]}"
end
```

`each` คืนค่า Hash ตัวเดิมกลับมา (เหมือน `each` บน Array) จึงเรียกต่อ (chain) ได้:

```ruby
person.each { |k, v| puts "#{k}: #{v}" }.class
# => Hash  (each คืน Hash ตัวเดิม)
```

### `each_pair` — เหมือน `each` ทุกประการ (เป็น alias)

```ruby
person.each_pair do |key, value|
  puts "#{key} => #{value}"
end
# ทำงานเหมือน each ทุกอย่าง — มีไว้เพื่อให้โค้ดสื่อความหมายชัดขึ้นว่ากำลังวนคู่ key-value
```

### `each_key` — วนซ้ำเฉพาะ key

```ruby
person.each_key do |key|
  puts "key: #{key}"
end
# => key: name
# => key: age
# => key: city
```

### `each_value` — วนซ้ำเฉพาะ value

```ruby
person.each_value do |value|
  puts "value: #{value}"
end
# => value: สมชาย
# => value: 25
# => value: กรุงเทพ
```

**เลือกใช้ตัวไหนดี:** ถ้าต้องใช้ทั้ง key และ value ในการประมวลผล ใช้ `each`/`each_pair`
ถ้าต้องการแค่ key หรือแค่ value อย่างเดียว การใช้ `each_key`/`each_value` ทำให้โค้ดสื่อ
เจตนาชัดเจนกว่า (readability) แม้ผลลัพธ์จะเขียนด้วย `each` แล้วดึงเฉพาะตัวที่ต้องการก็ได้
เหมือนกัน

### `with_index` ร่วมกับ `each` บน Hash

```ruby
person = { name: "สมชาย", age: 25, city: "กรุงเทพ" }

person.each.with_index do |(key, value), index|
  puts "#{index}: #{key} = #{value}"
end
# => 0: name = สมชาย
# => 1: age = 25
# => 2: city = กรุงเทพ
```

**สังเกต:** ต้องใส่วงเล็บ `(key, value)` ครอบ เพื่อบอก Ruby ว่าให้ **destructure** (แกะ) Array
`[key, value]` ออกเป็น 2 ตัวแปรแยกกัน ก่อนรับ `index` เป็น parameter ที่สอง — ถ้าลืมวงเล็บ
จะได้ผลลัพธ์ผิดเพี้ยน

---

## Step 46: `map` บน Hash (ทำไมได้ Array กลับมา) และการแปลงกลับด้วย `to_h`

### `map` บน Hash คืนค่าเป็น Array เสมอ

```ruby
prices = { apple: 30, banana: 15, orange: 25 }

result = prices.map { |fruit, price| "#{fruit}: #{price} บาท" }
p result
# => ["apple: 30 บาท", "banana: 15 บาท", "orange: 25 บาท"]
p result.class
# => Array
```

**ทำไม `map` บน Hash ถึงคืน Array ไม่คืน Hash:** เพราะ `map` เป็น method จาก module
`Enumerable` (เหมือนกับที่ใช้กับ Array) หลักการของ `map` คือ "แปลง element แต่ละตัวเป็นค่าใหม่
อะไรก็ได้ แล้วรวบรวมผลลัพธ์เป็นลิสต์" — ปัญหาคือ block สามารถคืนค่าอะไรก็ได้ (String, Integer,
Array, หรือแม้แต่ nil) ซึ่งไม่มีความหมายเป็นคู่ key-value เสมอไป ดังนั้น Ruby จึงออกแบบให้
`map` รวบรวมผลลัพธ์เป็น **Array แบบเดียวกันเสมอ** ไม่ว่าจะเรียกจาก Array หรือ Hash — นี่คือ
พฤติกรรมมาตรฐานของ `Enumerable#map` ทั้งหมด ไม่ใช่ข้อยกเว้นเฉพาะ Hash

ถ้าอยากได้ Hash กลับมา ต้องแปลงต่อด้วย `to_h` หรือใช้ method เฉพาะทาง
(`transform_values`/`transform_keys` ซึ่งจะพูดใน Step 50)

### แปลง Hash เป็น Hash ใหม่ด้วย `map` + `to_h`

```ruby
prices = { apple: 30, banana: 15, orange: 25 }

# เพิ่มราคาทุกอย่าง 10%
discounted = prices.map { |fruit, price| [fruit, (price * 1.1).round] }.to_h
p discounted
# => {apple: 33, banana: 17, orange: 28}
```

**อธิบาย:** block ต้องคืนค่าเป็น Array 2 ช่อง `[key, value]` ทุกครั้ง แล้ว `to_h` จะแปลง Array
ของคู่ `[key, value]` เหล่านั้นกลับเป็น Hash (`to_h` จะอธิบายละเอียดใน Step 50)

### `to_h` แบบมี block (Ruby 2.6+) — ทำในขั้นตอนเดียว

```ruby
prices = { apple: 30, banana: 15, orange: 25 }

discounted = prices.to_h { |fruit, price| [fruit, (price * 1.1).round] }
p discounted
# => {apple: 33, banana: 17, orange: 28}
```

วิธีนี้ทำงานเหมือน `map.to_h` แต่เขียนสั้นกว่าและไม่ต้องสร้าง Array กลาง (intermediate array)
จึงมีประสิทธิภาพดีกว่าเล็กน้อยด้วย — เป็นวิธีที่แนะนำในโค้ด Ruby สมัยใหม่

---

## Step 47: `select`, `reject`, `filter_map` บน Hash

### `select` (alias: `filter`) — กรองเอาเฉพาะคู่ที่ block คืนค่าจริง

```ruby
prices = { apple: 30, banana: 15, orange: 25, mango: 60 }

expensive = prices.select { |fruit, price| price > 20 }
p expensive
# => {apple: 30, orange: 25, mango: 60}
p expensive.class
# => Hash   <- ต่างจาก map! select บน Hash คืน Hash กลับมา (ไม่ใช่ Array)
```

**ทำไม `select` คืน Hash ได้ ในขณะที่ `map` คืน Array:** เพราะ `select`/`reject` เป็นการ
"คัดเลือกบางส่วนออกจากของเดิม" (คู่ key-value เดิมไม่ถูกแก้ไข แค่ถูกเลือกหรือไม่ถูกเลือก) จึง
รักษาโครงสร้างเดิม (Hash) ไว้ได้อย่างสมเหตุสมผล ในขณะที่ `map` คือ "แปลงเป็นค่าใหม่ที่ไม่รู้
รูปร่าง" จึงต้องคืนเป็น Array แบบทั่วไปที่สุดเสมอ — นี่คือหลักการออกแบบ API ที่สอดคล้องกันของ
`Enumerable`: method ที่ **แปลงข้อมูล** (map, flat_map) คืน Array, method ที่ **กรองข้อมูล**
โดยไม่เปลี่ยนรูปร่างคู่ key-value (select, reject, filter_map บางกรณี) คืน Hash

### `reject` — ตรงข้ามกับ `select`

```ruby
prices = { apple: 30, banana: 15, orange: 25, mango: 60 }

cheap = prices.reject { |fruit, price| price > 20 }
p cheap
# => {banana: 15}
```

### `select!` และ `reject!` — แก้ไข Hash เดิมโดยตรง (destructive)

```ruby
prices = { apple: 30, banana: 15, orange: 25, mango: 60 }

prices.select! { |fruit, price| price > 20 }
p prices
# => {apple: 30, orange: 25, mango: 60}   (prices ตัวเดิมถูกแก้ไขเลย ไม่ได้สร้างใหม่)
```

> เหมือนกับ Array ที่เรียนใน Part 004: method ที่ลงท้าย `!` จะแก้ไข object เดิมโดยตรง
> (mutate) และคืนค่า `nil` ถ้าไม่มีอะไรเปลี่ยนแปลง — ต้องระวังเรื่อง side effect เสมอ

### `filter_map` — รวม filter กับ map ในขั้นตอนเดียว (Ruby 2.7+)

```ruby
prices = { apple: 30, banana: 15, orange: 25, mango: 60 }

# ต้องการ: ชื่อผลไม้ (ตัวพิมพ์ใหญ่) เฉพาะที่ราคาเกิน 20
labels = prices.filter_map { |fruit, price| fruit.to_s.upcase if price > 20 }
p labels
# => ["APPLE", "ORANGE", "MANGO"]
p labels.class
# => Array
```

**อธิบาย:** `filter_map` คือ `map` ที่จะ**ทิ้งผลลัพธ์ที่เป็น `nil` หรือ `false`** ออกจาก Array
สุดท้ายให้อัตโนมัติ ในตัวอย่างนี้ ถ้า `price > 20` เป็น false, expression `fruit.to_s.upcase
if price > 20` จะคืน `nil` (เพราะไม่มี `else`) แล้ว `filter_map` จะตัด `nil` เหล่านั้นทิ้งไปเอง
— ผลลัพธ์เสมอเป็น Array (เหมือน `map` เพราะโดยธรรมชาติแล้วเป็นการ "แปลง" ไม่ใช่ "คัดเลือกคู่
key-value เดิมไว้ทั้งคู่")

ถ้าไม่มี `filter_map` เราต้องเขียนแบบนี้แทน (ยาวกว่า):

```ruby
labels_old_way = prices.select { |fruit, price| price > 20 }
                        .map { |fruit, price| fruit.to_s.upcase }
p labels_old_way   # ผลลัพธ์เดียวกัน แต่วนซ้ำ Hash 2 รอบ (ประสิทธิภาพแย่กว่า)
```

`filter_map` วนซ้ำแค่รอบเดียวและได้โค้ดที่กระชับกว่า จึงเป็นตัวเลือกที่แนะนำเมื่อโจทย์คือ
"กรองแล้วแปลง" พร้อมกัน

---

## Step 48: `merge` และ `merge!` — การรวม Hash และการแก้ conflict ด้วย block

### `merge` — รวม Hash สองตัวเข้าด้วยกัน (ไม่แก้ไข original)

```ruby
defaults = { color: "red", size: "M", quantity: 1 }
custom = { size: "L" }

result = defaults.merge(custom)
p result
# => {color: "red", size: "L", quantity: 1}
p defaults
# => {color: "red", size: "M", quantity: 1}   <- defaults ไม่เปลี่ยนแปลง (merge ไม่ mutate)
```

**กฎการชนกันของ key (conflict):** ถ้า key เดียวกันปรากฏอยู่ในทั้งสอง Hash ค่าจาก Hash ที่ส่ง
เข้าไปเป็น argument (`custom`) จะ**ชนะเสมอ** (ทับค่าเดิมของ `defaults`) — pattern นี้ใช้บ่อย
มากสำหรับการทำ "ค่าเริ่มต้น + ค่าที่ผู้ใช้กำหนดเอง" เช่นใน library ต่างๆ ที่รับ `options` hash

```ruby
def create_button(options = {})
  settings = { color: "blue", size: "medium", disabled: false }.merge(options)
  puts "สร้างปุ่ม: #{settings}"
end

create_button
# => สร้างปุ่ม: {color: "blue", size: "medium", disabled: false}

create_button(color: "red", disabled: true)
# => สร้างปุ่ม: {color: "red", size: "medium", disabled: true}
```

### รวม Hash หลายตัวพร้อมกัน

```ruby
h1 = { a: 1, b: 2 }
h2 = { b: 20, c: 3 }
h3 = { c: 30, d: 4 }

combined = h1.merge(h2, h3)
p combined
# => {a: 1, b: 20, c: 30, d: 4}   (Hash หลังสุดที่มี key ซ้ำจะชนะ)
```

### `merge!` (alias: `update`) — รวมและแก้ไข Hash เดิมโดยตรง

```ruby
settings = { color: "blue", size: "M" }
settings.merge!(color: "red")
p settings
# => {color: "red", size: "M"}   (settings ตัวเดิมถูกแก้ไขเลย)

# update เป็น alias ของ merge! ทำงานเหมือนกันทุกประการ
settings.update(size: "L")
p settings
# => {color: "red", size: "L"}
```

### ควบคุมการชนกันของ key ด้วย block

ถ้าไม่ต้องการให้ค่าใหม่ทับค่าเก่าเสมอ (default behavior) สามารถส่ง block เข้าไปเพื่อกำหนด
เองว่าเมื่อ key ชนกัน จะเลือกค่าไหน หรือคำนวณค่าใหม่อย่างไร

```ruby
inventory_warehouse_a = { apple: 10, banana: 5, orange: 8 }
inventory_warehouse_b = { apple: 3, banana: 12, mango: 7 }

# ต้องการรวมสต็อกจากสองคลัง: ถ้า key ซ้ำ ให้บวกจำนวนกัน (ไม่ใช่ทับ)
total_inventory = inventory_warehouse_a.merge(inventory_warehouse_b) do |key, old_val, new_val|
  old_val + new_val
end
p total_inventory
# => {apple: 13, banana: 17, orange: 8, mango: 7}
```

**อธิบาย block:** block รับ 3 parameter คือ `key` (key ที่ชนกัน), `old_val` (ค่าจาก Hash
ตัวแรก/ตัวที่เรียก merge), `new_val` (ค่าจาก Hash ที่ส่งเข้าไปเป็น argument) — block ต้องคืน
ค่าที่จะใช้เป็นผลลัพธ์สุดท้ายของ key นั้น block นี้จะถูกเรียก**เฉพาะ key ที่ชนกันเท่านั้น**
ส่วน key ที่ไม่ชนกัน (มีอยู่ใน Hash เดียว) จะถูกเก็บไว้ตามปกติโดยไม่ผ่าน block

ตัวอย่างอีกแบบ: เลือกค่าที่มากกว่าเมื่อชนกัน

```ruby
high_scores_session1 = { alice: 85, bob: 92 }
high_scores_session2 = { alice: 90, bob: 88, carol: 95 }

best_scores = high_scores_session1.merge(high_scores_session2) do |player, score1, score2|
  [score1, score2].max
end
p best_scores
# => {alice: 90, bob: 92, carol: 95}
```

---

## Step 49: Nested Hash, method `dig`, และ safe navigation `&.`

ข้อมูลในโลกจริงมักซับซ้อนกว่า flat hash ธรรมดา (เช่น response จาก API, config file) จึงต้อง
ใช้ **nested hash** — Hash ที่มี value เป็น Hash หรือ Array ซ้อนกันไปอีกหลายชั้น

### สร้างและเข้าถึง Nested Hash

```ruby
user = {
  name: "สมชาย",
  address: {
    city: "กรุงเทพ",
    district: "บางรัก",
    zipcode: "10500"
  },
  contacts: {
    email: "somchai@example.com",
    phone: "081-234-5678"
  }
}

# เข้าถึงข้อมูลชั้นในด้วยการ chain [] ต่อกัน
p user[:address][:city]      # => "กรุงเทพ"
p user[:contacts][:email]    # => "somchai@example.com"
```

### ปัญหาของการ chain `[]` เมื่อข้อมูลไม่ครบ (missing key)

```ruby
incomplete_user = { name: "สมหญิง" }   # ไม่มี key :address

# เข้าถึงตรงๆ จะพัง!
begin
  p incomplete_user[:address][:city]
rescue NoMethodError => e
  puts "เกิด error: #{e.message}"
  # => เกิด error: undefined method `[]' for nil
end
```

**อธิบายว่าทำไม error:** `incomplete_user[:address]` ไม่มีอยู่ จึงคืน `nil` แล้วเราพยายามเรียก
`[:city]` บน `nil` ต่อ ซึ่ง `nil` ไม่มี method `[]` แบบ Hash จึงเกิด `NoMethodError` — นี่คือ
ปัญหาคลาสสิกที่เจอบ่อยมากเวลาทำงานกับข้อมูลซ้อนกันหลายชั้น (โดยเฉพาะข้อมูลจาก API ภายนอกที่
โครงสร้างไม่แน่นอน)

### `dig` — เข้าถึงข้อมูลซ้อนกันอย่างปลอดภัย

```ruby
incomplete_user = { name: "สมหญิง" }

p incomplete_user.dig(:address, :city)   # => nil  (ไม่ error! คืน nil เมื่อ path ไหนไม่มีอยู่)

user = {
  name: "สมชาย",
  address: { city: "กรุงเทพ", district: "บางรัก" }
}
p user.dig(:address, :city)        # => "กรุงเทพ"
p user.dig(:address, :country)     # => nil  (มี :address แต่ไม่มี :country ข้างใน)
p user.dig(:contacts, :email)      # => nil  (ไม่มี :contacts เลย)
```

**อธิบายกลไก:** `dig` จะไล่เข้าไปทีละชั้นตาม argument ที่ส่งเข้ามา ถ้าชั้นไหน "เจอ `nil`
ระหว่างทาง" จะหยุดทันทีและคืน `nil` เลย โดยไม่พยายามเข้าถึงชั้นถัดไป (ซึ่งจะทำให้ error) —
นี่คือเหตุผลที่ `dig` ปลอดภัยกว่าการ chain `[]` มากเมื่อไม่แน่ใจว่าโครงสร้างข้อมูลครบถ้วนหรือไม่

`dig` ใช้ได้กับ Array ด้วย และผสมกันได้ (Hash ซ้อน Array ซ้อน Hash):

```ruby
company = {
  name: "Tech Corp",
  employees: [
    { name: "สมชาย", skills: ["Ruby", "Rails"] },
    { name: "สมหญิง", skills: ["JavaScript", "React"] }
  ]
}

p company.dig(:employees, 0, :name)          # => "สมชาย"
p company.dig(:employees, 0, :skills, 1)     # => "Rails"
p company.dig(:employees, 5, :name)          # => nil  (index 5 ไม่มีอยู่ ก็ไม่ error)
```

### Safe navigation operator `&.` — สำหรับเรียก method ต่อเนื่องอย่างปลอดภัย

`dig` ใช้เข้าถึง key/index ซ้อนกัน แต่ถ้าต้องการ**เรียก method** ต่อจากผลลัพธ์ (ไม่ใช่แค่ดึงค่า)
และไม่แน่ใจว่าค่านั้นเป็น `nil` หรือไม่ ให้ใช้ `&.` (safe navigation operator หรือเรียกเล่นๆ ว่า
"lonely operator" เพราะหน้าตาเหมือนคนเดียวเหงาๆ `&.`)

```ruby
user = { name: "สมชาย", address: { city: "กรุงเทพ" } }

# ถ้าใช้ . ธรรมดา แล้วค่าที่ได้จาก dig เป็น nil จะพัง
country = user.dig(:address, :country)   # => nil
begin
  puts country.upcase   # NoMethodError: undefined method `upcase' for nil
rescue NoMethodError => e
  puts "error: #{e.message}"
end

# ใช้ &. แทน . จะปลอดภัย: ถ้า object เป็น nil จะคืน nil ทันที ไม่เรียก method ต่อ
puts country&.upcase.inspect   # => nil   (ไม่ error)

city = user.dig(:address, :city)
puts city&.upcase.inspect      # => "กรุงเทพ"   (ถ้าไม่ใช่ nil ก็ทำงานปกติ)
```

### ผสม `dig` กับ `&.` และค่า default ด้วย `||`

Pattern ที่ใช้บ่อยมากในโค้ดจริง คือรวมทั้งสามอย่างเข้าด้วยกัน: `dig` เพื่อความปลอดภัยตอน
เข้าถึงข้อมูลซ้อน, `&.` เพื่อความปลอดภัยตอนเรียก method ต่อ, และ `||` เพื่อกำหนดค่า default
สุดท้ายเมื่อทุกอย่างเป็น `nil`

```ruby
def display_city(user)
  city = user.dig(:address, :city)&.strip&.capitalize
  city_display = city || "ไม่ระบุที่อยู่"
  puts "เมือง: #{city_display}"
end

display_city({ name: "สมชาย", address: { city: "  กรุงเทพ  " } })
# => เมือง: กรุงเทพ

display_city({ name: "สมหญิง" })
# => เมือง: ไม่ระบุที่อยู่
```

### แก้ไขค่าใน Nested Hash

```ruby
user = { name: "สมชาย", address: { city: "กรุงเทพ" } }

# แก้ไขค่าซ้อนอยู่ข้างใน — ต้อง chain [] เข้าไปถึงชั้นที่ต้องการแก้
user[:address][:city] = "เชียงใหม่"
p user
# => {name: "สมชาย", address: {city: "เชียงใหม่"}}

# ถ้า key ชั้นกลางยังไม่มีอยู่ ต้องสร้างก่อนถึงจะแก้ไขข้างในได้ (ไม่งั้นจะ error)
another_user = { name: "สมหญิง" }
another_user[:address] ||= {}          # สร้าง Hash ว่างถ้ายังไม่มี key :address
another_user[:address][:city] = "ภูเก็ต"
p another_user
# => {name: "สมหญิง", address: {city: "ภูเก็ต"}}
```

**อธิบาย `||=`:** `another_user[:address] ||= {}` แปลว่า "ถ้า `another_user[:address]` เป็น
`nil` หรือ `false` ให้กำหนดค่าเป็น `{}` แทน แต่ถ้ามีค่าอยู่แล้วให้คงค่าเดิมไว้" เป็น idiom ที่
พบบ่อยมากสำหรับการ "สร้าง key ถ้ายังไม่มี" ก่อนที่จะเข้าไปแก้ไขข้างในต่อ

---

## Step 50: `transform_keys`, `transform_values`, `to_a`, และแบบฝึกหัดโปรเจกต์

### `transform_values` — แปลง value ทุกตัว โดย key ยังเหมือนเดิม

```ruby
prices = { apple: 30, banana: 15, orange: 25 }

# เพิ่มราคาทุกอย่าง 10% — คืน Hash กลับมาเลย (ไม่ต้อง .to_h แบบ map)
discounted = prices.transform_values { |price| (price * 1.1).round }
p discounted
# => {apple: 33, banana: 17, orange: 28}
p discounted.class
# => Hash
```

เทียบกับวิธี `map.to_h` ใน Step 46 จะเห็นว่า `transform_values` สั้นกว่าและสื่อเจตนาชัดกว่า
มากเมื่อโจทย์คือ "แก้เฉพาะ value โดย key เหมือนเดิมทุกประการ" — ควรเลือกใช้ `transform_values`
แทน `map.to_h` เสมอเมื่อทำได้ เพราะอ่านง่ายกว่าและบอกเจตนาได้ชัดเจนกว่า

### `transform_keys` — แปลง key ทุกตัว โดย value ยังเหมือนเดิม

```ruby
snake_case_data = { first_name: "สมชาย", last_name: "ใจดี" }

# แปลง key เป็น String (เผื่อจะส่งออกเป็น JSON)
string_keys = snake_case_data.transform_keys(&:to_s)
p string_keys
# => {"first_name" => "สมชาย", "last_name" => "ใจดี"}

# แปลง key ด้วย custom logic (เช่นแปลง snake_case เป็น camelCase สำหรับส่งให้ frontend)
camel_case_data = snake_case_data.transform_keys do |key|
  parts = key.to_s.split("_")
  camel = parts[0] + parts[1..].map(&:capitalize).join
  camel.to_sym
end
p camel_case_data
# => {firstName: "สมชาย", lastName: "ใจดี"}
```

### `transform_values!` / `transform_keys!` — เวอร์ชัน mutate

```ruby
data = { a: 1, b: 2 }
data.transform_values! { |v| v * 100 }
p data
# => {a: 100, b: 200}   (data ตัวเดิมถูกแก้ไขเลย)
```

### `to_a` — แปลง Hash เป็น Array ของคู่ `[key, value]`

```ruby
person = { name: "สมชาย", age: 25 }

array_form = person.to_a
p array_form
# => [[:name, "สมชาย"], [:age, 25]]
p array_form.class
# => Array
```

ทุกคู่ key-value กลายเป็น Array ย่อยขนาด 2 ช่อง (`[key, value]`) เรียงกันเป็น Array ใหญ่ —
รูปแบบนี้เรียกว่า **Array of pairs** ซึ่งเป็นสะพานเชื่อมระหว่าง Hash กับ Array ทำให้เรานำ
method ของ Array (เช่น `sort`, `sort_by`) มาใช้กับข้อมูลที่มาจาก Hash ได้

### `to_h` — แปลง Array of pairs กลับเป็น Hash

```ruby
pairs = [[:name, "สมชาย"], [:age, 25]]

hash_form = pairs.to_h
p hash_form
# => {name: "สมชาย", age: 25}
```

### ตัวอย่างการใช้ `to_a` เพื่อ sort Hash ตาม value แล้วแปลงกลับ

Hash เองไม่มี method `sort` ที่ใช้งานตรงๆ ได้สะดวกเท่า Array จึงมัก "แปลงเป็น Array ก่อน
sort แล้วแปลงกลับเป็น Hash"

```ruby
scores = { alice: 85, bob: 92, carol: 78, dave: 95 }

# เรียงจากคะแนนมากไปน้อย
sorted_by_score = scores.sort_by { |name, score| -score }.to_h
p sorted_by_score
# => {dave: 95, bob: 92, alice: 85, carol: 78}

# Ruby มี Hash#sort_by ให้ใช้ตรงๆ (คืน Array of pairs เหมือนกัน) จึงต้อง .to_h ต่อเสมอ
p scores.sort_by { |name, score| -score }.class
# => Array
```

> **หมายเหตุ:** `sort` และ `sort_by` เป็น method จาก `Enumerable` เหมือน `map` จึงมีกฎเดียวกัน
> คือคืนค่าเป็น Array เสมอ ต้อง `.to_h` เองถ้าต้องการ Hash กลับมา

### สรุปตารางเปรียบเทียบ: method ไหนคืน Hash, ไหนคืน Array

| Method | คืนค่าเป็น | เหตุผล |
|---|---|---|
| `each`, `each_pair` | Hash (ตัวเดิม) | แค่วนซ้ำ ไม่ได้สร้างข้อมูลใหม่ |
| `select`, `reject`, `filter` | Hash | คัดเลือกคู่เดิม ไม่เปลี่ยนรูปร่าง |
| `transform_values`, `transform_keys` | Hash | แปลงแต่รักษาคู่ key-value ไว้ |
| `merge`, `merge!` | Hash | รวมโครงสร้างเดิมเข้าด้วยกัน |
| `map`, `flat_map`, `sort_by`, `sort`, `filter_map` | Array | สร้างค่าใหม่ที่ไม่รับประกันรูปร่างคู่ key-value |
| `to_a` | Array | แปลงชนิดโดยตรง |
| `to_h` | Hash | แปลงชนิดโดยตรง (ทั้งจาก Array of pairs และจาก Hash+block) |

---

## แบบฝึกหัด: แปลงและวิเคราะห์ Nested Config/User Profile

### โจทย์

สมมติได้รับข้อมูล config ของระบบในรูปแบบ nested hash ที่มาจากการ parse ไฟล์ JSON (ดังนั้น
key ทั้งหมดเป็น **String** ไม่ใช่ Symbol) ให้เขียนโปรแกรม `config_parser.rb` ที่ทำสิ่งต่อไปนี้:

1. รับ config เป็น nested hash ที่มีโครงสร้างตามตัวอย่างด้านล่าง
2. แปลง key ทั้งหมด (ทุกชั้น) จาก String เป็น Symbol เพื่อให้ใช้งานสะดวกแบบ Ruby-idiomatic
3. ดึงรายชื่อผู้ใช้ทั้งหมดที่มี role เป็น `"admin"` ออกมา
4. คำนวณจำนวนผู้ใช้ทั้งหมดที่ active (ใช้ `count` ของ Array)
5. ใช้ `dig` เพื่อดึงค่า `timeout` จาก `settings.network.timeout` อย่างปลอดภัย (ต้องไม่พังแม้
   ไม่มี key นั้นอยู่)
6. สร้างรายงานสรุปเป็น Hash ใหม่: `{ total_users:, active_users:, admin_names:, timeout: }`

### เฉลย

```ruby
# frozen_string_literal: true

# config_parser.rb

raw_config = {
  "app_name" => "MyApp",
  "settings" => {
    "network" => {
      "timeout" => 30,
      "retries" => 3
    },
    "theme" => "dark"
  },
  "users" => [
    { "name" => "สมชาย", "role" => "admin", "active" => true },
    { "name" => "สมหญิง", "role" => "member", "active" => true },
    { "name" => "วิชัย", "role" => "admin", "active" => false },
    { "name" => "มานี", "role" => "member", "active" => true }
  ]
}

# ---------------------------------------------------------------
# ขั้นที่ 1: แปลง key ทุกชั้นจาก String เป็น Symbol แบบ recursive
# ---------------------------------------------------------------
# เขียนเป็น method แยก เพราะ transform_keys ธรรมดาแปลงได้แค่ชั้นบนสุด
# (ชั้นลึกกว่านั้นต้องเรียกซ้ำ - recursion)
def deep_symbolize_keys(obj)
  case obj
  when Hash
    obj.each_with_object({}) do |(key, value), result|
      result[key.to_sym] = deep_symbolize_keys(value)
    end
  when Array
    obj.map { |item| deep_symbolize_keys(item) }
  else
    obj
  end
end

config = deep_symbolize_keys(raw_config)

puts "=== Config หลังแปลง key เป็น Symbol ==="
p config
puts

# ---------------------------------------------------------------
# ขั้นที่ 2: ดึงรายชื่อ admin ทั้งหมด
# ---------------------------------------------------------------
admin_names = config[:users]
                .select { |user| user[:role] == "admin" }
                .map { |user| user[:name] }

puts "=== รายชื่อ admin ==="
p admin_names
puts

# ---------------------------------------------------------------
# ขั้นที่ 3: นับจำนวนผู้ใช้ที่ active
# ---------------------------------------------------------------
active_users_count = config[:users].count { |user| user[:active] }
total_users_count = config[:users].size

puts "=== สถิติผู้ใช้ ==="
puts "ผู้ใช้ทั้งหมด: #{total_users_count} คน"
puts "ผู้ใช้ที่ active: #{active_users_count} คน"
puts

# ---------------------------------------------------------------
# ขั้นที่ 4: ใช้ dig ดึงค่า timeout อย่างปลอดภัย
# ---------------------------------------------------------------
timeout = config.dig(:settings, :network, :timeout)
missing_value = config.dig(:settings, :database, :timeout)  # path ที่ไม่มีอยู่จริง

puts "=== ทดสอบ dig ==="
puts "timeout: #{timeout.inspect}"                # => 30
puts "missing_value: #{missing_value.inspect}"     # => nil (ไม่ error)
puts

# ---------------------------------------------------------------
# ขั้นที่ 5: สร้างรายงานสรุปเป็น Hash ใหม่
# ---------------------------------------------------------------
report = {
  total_users: total_users_count,
  active_users: active_users_count,
  admin_names:,               # ใช้ shorthand syntax (Ruby 3.1+)
  timeout: timeout || 60      # ถ้า timeout เป็น nil ให้ใช้ค่า default 60
}

puts "=== รายงานสรุป ==="
report.each do |key, value|
  puts "#{key}: #{value}"
end
```

ทดสอบรัน:

```bash
ruby config_parser.rb
```

ผลลัพธ์ที่คาดหวัง (บางส่วน):

```
=== รายชื่อ admin ===
["สมชาย", "วิชัย"]

=== สถิติผู้ใช้ ===
ผู้ใช้ทั้งหมด: 4 คน
ผู้ใช้ที่ active: 3 คน

=== ทดสอบ dig ===
timeout: 30
missing_value: nil

=== รายงานสรุป ===
total_users: 4
active_users: 3
admin_names: ["สมชาย", "วิชัย"]
timeout: 30
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `case obj when Hash ... when Array ... else ...` — pattern matching แบบง่ายด้วย `case`
  เพื่อตรวจสอบชนิดข้อมูลและจัดการ recursive conversion (จะเรียนละเอียดเรื่อง `case` ใน
  Part 006 ที่กำลังจะถึง)
- `each_with_object({})` — คล้าย `reduce`/`inject` ที่เรียนใน Part 004 แต่สะดวกกว่าเมื่อ
  ต้องการ "สร้าง object ใหม่แล้วยัดข้อมูลใส่ทีละตัว" (ในที่นี้คือสร้าง Hash ใหม่)
- การเขียน method แบบ **recursive** (เรียกตัวเองซ้ำ) เพื่อจัดการโครงสร้างข้อมูลที่ซ้อนกันได้
  ไม่จำกัดชั้น — เทคนิคนี้สำคัญมากเมื่อทำงานกับ JSON/config ที่มีโครงสร้างซับซ้อน
- การผสาน `dig`, `||` (ค่า default), และ shorthand `key:` syntax เข้าด้วยกันในสถานการณ์จริง

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เขียน method `deep_stringify_keys(obj)` ที่ทำงานตรงข้ามกับ `deep_symbolize_keys` ในเฉลย
   (แปลง Symbol key กลับเป็น String key แบบ recursive ทุกชั้น) แล้วทดสอบกับ `config` ที่ได้
   จากเฉลยด้านบน ว่าแปลงกลับไปได้ตรงกับ `raw_config` เดิมหรือไม่

2. เพิ่มฟีเจอร์ในโปรแกรม: จัดกลุ่มผู้ใช้ตาม `role` โดยใช้ `each_with_object` หรือ
   `Hash.new { |h, k| h[k] = [] }` ให้ได้ผลลัพธ์รูปแบบ
   `{ admin: ["สมชาย", "วิชัย"], member: ["สมหญิง", "มานี"] }` (เฉพาะชื่อ ไม่เอาทั้ง Hash
   ของ user)

3. เขียนโปรแกรมแยกต่างหากชื่อ `merge_configs.rb` ที่จำลองสถานการณ์ "รวม config หลาย
   environment": มี `base_config` (ค่าพื้นฐาน) และ `production_config` (ค่าเฉพาะ production
   ที่จะ override ค่าบางส่วน) ทั้งสองเป็น nested hash ที่มี `settings` ซ้อนกันข้างใน — ให้เขียน
   method `deep_merge(hash1, hash2)` ที่รวม nested hash แบบ "ลึกถึงชั้นใน" (ถ้า `merge`
   ธรรมดาจะทับทั้งก้อนของ nested hash ไม่ใช่รวมกันในชั้นลึก ให้ลองสังเกตความต่างนี้ก่อนเขียน
   แล้วแก้ปัญหาด้วย recursion เหมือนโจทย์หลัก)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- สร้าง Hash ได้ทั้งแบบ literal `{}` และ `Hash.new` พร้อมกำหนด default value หรือ default
  block เพื่อจัดการกรณี key ไม่มีอยู่ได้อย่างสะดวกและปลอดภัย (พร้อมรู้จักกับดัก
  `Hash.new([])`)
- เข้าใจว่าทำไม Ruby/Rails community นิยมใช้ **Symbol key** มากกว่า String key (ประสิทธิภาพ,
  memory, semantic) และใช้ `key:` shorthand syntax (Ruby 3.1+) ได้อย่างคล่องแคล่ว
- เข้าถึง/แก้ไข/ลบค่าใน Hash ได้หลายวิธี (`[]`, `fetch`, `store`, `delete`) และรู้ว่าเมื่อไหร่
  ควรใช้ `fetch` แทน `[]` เพื่อป้องกันบั๊กจาก `nil` ที่ซ่อนอยู่
- วนซ้ำ Hash ได้ด้วย `each`, `each_pair`, `each_key`, `each_value` และเข้าใจว่าทำไม `map` บน
  Hash คืนค่าเป็น Array เสมอ (ต่างจาก `select`/`reject`/`transform_values` ที่คืน Hash)
- ใช้ `select`, `reject`, `filter_map` กรอง/แปลงข้อมูลใน Hash ได้อย่างมีประสิทธิภาพ
- รวม Hash ด้วย `merge`/`merge!` ได้ พร้อมควบคุมการชนกันของ key ด้วย block เมื่อไม่ต้องการ
  ให้ค่าใหม่ทับค่าเก่าแบบอัตโนมัติ
- จัดการ **Nested Hash** ได้อย่างปลอดภัยด้วย `dig` และ safe navigation operator `&.` โดยไม่
  ต้องกลัว `NoMethodError` เมื่อโครงสร้างข้อมูลไม่ครบ
- แปลง key/value ทั้ง Hash ด้วย `transform_keys`/`transform_values` และแปลงไปมาระหว่าง Hash
  กับ Array of pairs ด้วย `to_a`/`to_h` ได้
- ประยุกต์ใช้ทุกความรู้ในบทนี้เขียนโปรแกรมแปลงและวิเคราะห์ nested config/user profile
  แบบ recursive ได้ครบวงจร

**ต่อไป (Part 006):** เราจะเจาะลึกเรื่อง **Control flow** ทั้งหมดของ Ruby — `if`/`unless`/
`case`, ternary operator, การวนลูปแบบต่างๆ (`loop`, `while`, `until`, `for`), และการควบคุม
การวนลูปด้วย `break`/`next`/`redo` ซึ่งจะทำให้เราเขียนโปรแกรมที่ตัดสินใจและทำงานซ้ำได้อย่าง
ยืดหยุ่นกว่าที่เคยเรียนมา
