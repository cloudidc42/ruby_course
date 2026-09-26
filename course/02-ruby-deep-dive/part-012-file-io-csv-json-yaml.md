# Part 012: File I/O, การอ่าน/เขียนไฟล์, CSV, JSON, YAML

> **Step ครอบคลุมใน Part นี้:** Step 111–120
> **ระดับ:** ปานกลาง (ต้องผ่าน Phase 1 มาก่อน โดยเฉพาะ Part 007 เรื่อง Method และ Part 011
> เรื่อง Exception Handling)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6)

Part ที่แล้ว (Part 011) เราเรียนวิธีจัดการข้อผิดพลาดด้วย `begin`/`rescue`/`ensure` ไปแล้ว —
ทักษะนั้นจะถูกใช้ซ้ำอย่างต่อเนื่องใน Part นี้ เพราะการทำงานกับไฟล์คือจุดที่ error เกิดขึ้นได้
บ่อยที่สุดในโปรแกรมจริง (ไฟล์ไม่มีอยู่, ไม่มีสิทธิ์เขียน, format ข้อมูลผิด ฯลฯ)

Part นี้เราจะเรียนวิธีให้โปรแกรม Ruby **อ่านและเขียนข้อมูลลงดิสก์** ตั้งแต่การเปิด/ปิดไฟล์
พื้นฐาน ไปจนถึงการทำงานกับไฟล์ข้อมูล 3 รูปแบบที่พบบ่อยที่สุดในงานจริง คือ **CSV** (ตาราง/
spreadsheet), **JSON** (แลกเปลี่ยนข้อมูลกับ API/JavaScript), และ **YAML** (ไฟล์ config —
สิ่งที่ Rails ใช้เป็นมาตรฐานสำหรับ `config/database.yml`, `config/credentials.yml.enc` และอื่นๆ
อีกมาก) ทักษะเหล่านี้คือรากฐานที่จำเป็นก่อนเข้าสู่เรื่อง ActiveRecord ใน Phase 3 เพราะ
เบื้องหลังของ ORM ก็คือการอ่าน/เขียนข้อมูลแบบมีโครงสร้างนี่เอง เพียงแต่เปลี่ยนจากไฟล์เป็น
ฐานข้อมูล

## สารบัญของ Part นี้

- Step 111: `File.open` แบบ block form (auto-close) กับแบบ manual open/close
- Step 112: Read mode ต่างๆ — `"r"`, `"w"`, `"a"`, `"r+"`
- Step 113: `File.read`, `File.write`, `File.readlines`
- Step 114: อ่านทีละบรรทัดด้วย `each_line`/`foreach` — ประหยัด memory กว่า `File.read` แค่ไหน
- Step 115: ตรวจสอบไฟล์ก่อนใช้งาน — `File.exist?`, `File.delete`, `File.size`, `File.rename`
- Step 116: จัดการ path — `File.join`, `File.expand_path`, `__dir__`, `Dir.glob`
- Step 117: `require "csv"` — อ่าน/เขียน CSV พร้อม header
- Step 118: `require "json"` — `JSON.generate`/`JSON.parse`, `symbolize_names`
- Step 119: `require "yaml"` — `YAML.load`/`YAML.dump` และทำไม Rails ใช้ YAML เป็นไฟล์ config
- Step 120: แบบฝึกหัดปิด Part — สร้างรายงานสต็อกสินค้าจาก CSV ส่งออกเป็น JSON และ YAML

---

## Step 111: `File.open` แบบ block form (auto-close) กับแบบ manual open/close

การเปิดไฟล์ใน Ruby ทำได้ 2 แบบหลักๆ ทั้งสองแบบเปิดไฟล์เหมือนกัน แต่ต่างกันตรง **ใครเป็นคน
ปิดไฟล์และเมื่อไหร่**

```ruby
# frozen_string_literal: true

# แบบที่ 1: Manual open/close — ต้องเรียก .close เองเสมอ
file = File.open("note1.txt", "w")
file.write("บรรทัดที่หนึ่ง\n")
file.write("บรรทัดที่สอง\n")
file.close

puts File.read("note1.txt")
# => บรรทัดที่หนึ่ง
#    บรรทัดที่สอง

# แบบที่ 2: Block form — Ruby ปิดไฟล์ให้อัตโนมัติเมื่อ block จบ (แนะนำให้ใช้เป็นค่าเริ่มต้น)
File.open("note2.txt", "w") do |f|
  f.write("เขียนด้วย block form\n")
  f.puts "puts ก็ใช้ได้เหมือนกัน"
end

puts File.read("note2.txt")
# => เขียนด้วย block form
#    puts ก็ใช้ได้เหมือนกัน

# พิสูจน์ว่า block form ปิดไฟล์ให้จริง
f2 = nil
File.open("note3.txt", "w") { |f| f2 = f; f.puts "test" }
puts f2.closed?
# => true
```

**อธิบาย:**

- `File.open(path, mode)` แบบไม่มี block คืนค่าเป็น `File` object ที่ยัง**เปิดอยู่** — ต้อง
  เรียก `.close` เองเสมอ ถ้าลืมปิด ข้อมูลบางส่วนที่ยังค้างอยู่ใน buffer อาจไม่ถูกเขียนลงดิสก์
  จริง (flush ไม่ครบ) และไฟล์ handle จะค้างอยู่จนกว่า garbage collector จะมาเก็บ ซึ่งเป็น
  ทรัพยากรของระบบปฏิบัติการที่มีจำกัด (ถ้าเปิดไฟล์ค้างไว้เยอะเกินไปจะเจอ error
  `Errno::EMFILE: too many open files`)
- `File.open(path, mode) { |f| ... }` แบบมี block จะปิดไฟล์ให้อัตโนมัติทันทีที่ block จบการ
  ทำงาน **ไม่ว่า block จะจบแบบปกติหรือเกิด exception ระหว่างทางก็ตาม** (ทำงานคล้าย `ensure`
  ที่เรียนใน Part 011 — Ruby รับประกันว่าไฟล์จะถูกปิดเสมอ)
- **แนวปฏิบัติมาตรฐาน:** ใช้ **block form เป็นค่าเริ่มต้นเสมอ** เพราะปลอดภัยกว่า ไม่มีทาง
  ลืมปิดไฟล์ ใช้ manual open/close เฉพาะกรณีพิเศษที่ต้องส่ง file handle เดียวกันข้าม method
  หลายตัว (พบไม่บ่อยในงานทั่วไป)

---

## Step 112: Read mode ต่างๆ — `"r"`, `"w"`, `"a"`, `"r+"`

argument ตัวที่สองของ `File.open` คือ **mode** บอกว่าจะเปิดไฟล์เพื่อทำอะไร แต่ละ mode มี
พฤติกรรมต่างกันชัดเจน สับสนกันบ่อยที่สุดในหมู่มือใหม่

| Mode | ความหมาย | ถ้าไฟล์ยังไม่มี | ถ้าไฟล์มีอยู่แล้ว |
|------|----------|-----------------|-------------------|
| `"r"` | อ่านอย่างเดียว (ค่าเริ่มต้นถ้าไม่ระบุ mode) | Error (`Errno::ENOENT`) | เปิดอ่านตั้งแต่ต้นไฟล์ |
| `"w"` | เขียนอย่างเดียว | สร้างไฟล์ใหม่ | **ล้างเนื้อหาเดิมทิ้งทั้งหมด** แล้วเริ่มเขียนใหม่ |
| `"a"` | เขียนต่อท้าย (append) | สร้างไฟล์ใหม่ | เก็บเนื้อหาเดิมไว้ เขียนต่อท้ายสุด |
| `"r+"` | อ่านและเขียนได้ทั้งคู่ | Error (`Errno::ENOENT`) | เปิดจากต้นไฟล์ เขียนทับได้จากตำแหน่ง pointer ปัจจุบัน (ไม่ล้างไฟล์) |

```ruby
# frozen_string_literal: true

# "w" - เขียนทับทั้งไฟล์
File.open("log.txt", "w") { |f| f.puts "เริ่มต้น log" }
puts File.read("log.txt")
# => เริ่มต้น log

# "a" - append ต่อท้ายไฟล์เดิม เรียกซ้ำกี่ครั้งก็ไม่ล้างของเดิม
File.open("log.txt", "a") { |f| f.puts "บันทึกเพิ่มเติม 1" }
File.open("log.txt", "a") { |f| f.puts "บันทึกเพิ่มเติม 2" }
puts File.read("log.txt")
# => เริ่มต้น log
#    บันทึกเพิ่มเติม 1
#    บันทึกเพิ่มเติม 2

# "r" - อ่านอย่างเดียว เขียนไม่ได้
File.open("log.txt", "r") do |f|
  puts f.read
  begin
    f.write("ทดสอบเขียน")
  rescue IOError => e
    puts "เขียนไม่ได้: #{e.message}"
  end
end
# => เขียนไม่ได้: not opened for writing
```

**อธิบาย:**

- `"w"` เหมาะกับกรณี "สร้างไฟล์ใหม่ทุกครั้ง" เช่น สร้างรายงานฉบับล่าสุด ไม่สนใจฉบับเก่า
- `"a"` เหมาะกับ **log file** ที่ต้องการเก็บประวัติสะสมไปเรื่อยๆ โดยไม่ลบของเดิม
- `"r"` คือ mode ที่ปลอดภัยที่สุด ใช้เมื่อต้องการแค่อ่านข้อมูล ป้องกันบัคจากการเขียนไฟล์โดย
  ไม่ตั้งใจ (ถ้าพยายามเขียนจะได้ `IOError` ทันที เหมือนตัวอย่างข้างบน)

### `"r+"` — mode ที่ต้องระวังที่สุด

`"r+"` **ไม่ได้ล้างไฟล์ก่อนเขียน** เหมือน `"w"` แต่เขียนทับข้อมูลเดิมนับจากตำแหน่ง pointer
เท่านั้น ถ้าข้อความใหม่สั้นกว่าข้อความเดิม จะมีเศษข้อมูลเก่าตกค้างอยู่ท้ายไฟล์

```ruby
# frozen_string_literal: true

File.write("counter.txt", "count=000\n")

File.open("counter.txt", "r+") do |f|
  f.write("count=042")
end

puts File.read("counter.txt")
# => count=042
#    (ขึ้นบรรทัดใหม่ท้ายไฟล์ยังอยู่ เพราะเขียนทับแค่ 9 ตัวอักษรแรก ไม่ได้ล้างไฟล์ทั้งหมด)

# ถ้าเขียนทับด้วยข้อความที่ "สั้นกว่า" ความยาวเดิม จะเหลือเศษข้อมูลเก่าติดอยู่ท้ายไฟล์
File.write("counter2.txt", "count=999999\n")
File.open("counter2.txt", "r+") { |f| f.write("count=1") }
puts File.read("counter2.txt")
# => count=199999
#    (เขียนทับแค่ 7 ตัวอักษรแรก "count=1" ทำให้เลข 9 เดิม 5 ตัวหลังยังค้างอยู่ 5 ตัว)
```

**สรุป:** `"r+"` เหมาะกับงานเฉพาะทางที่ต้องการแก้ไขบางส่วนของไฟล์ขนาดใหญ่โดยไม่อ่าน-เขียน
ทั้งไฟล์ใหม่ (เช่น แก้ header ของไฟล์ binary) แต่สำหรับงานทั่วไป ถ้าต้องการ "แก้ไขเนื้อหาไฟล์"
วิธีที่ปลอดภัยและเข้าใจง่ายกว่ามากคือ **อ่านทั้งไฟล์ด้วย `"r"` มาแก้ไขใน memory แล้วเขียนทับ
ทั้งไฟล์ใหม่ด้วย `"w"`** ซึ่งเป็นแนวทางที่จะใช้ตลอด Part นี้

---

## Step 113: `File.read`, `File.write`, `File.readlines`

Ruby มี **class method** ที่เป็นทางลัด ไม่ต้องเปิด/ปิดไฟล์เองเลยสำหรับงานอ่าน/เขียนไฟล์
แบบง่ายๆ ที่พบบ่อยที่สุด

```ruby
# frozen_string_literal: true

File.write("poem.txt", "บรรทัดหนึ่ง\nบรรทัดสอง\nบรรทัดสาม\n")

# File.read - อ่านทั้งไฟล์มาเป็น String เดียว (เปิด-อ่าน-ปิดให้ในคำสั่งเดียว)
content = File.read("poem.txt")
puts content.class
# => String
puts content
# => บรรทัดหนึ่ง
#    บรรทัดสอง
#    บรรทัดสาม

# File.readlines - อ่านทั้งไฟล์มาเป็น Array โดยแบ่งเป็นบรรทัดๆ (แต่ละ element ยังมี "\n" ติดท้าย)
lines = File.readlines("poem.txt")
p lines
# => ["บรรทัดหนึ่ง\n", "บรรทัดสอง\n", "บรรทัดสาม\n"]
puts lines.size
# => 3

# chomp: true (มีตั้งแต่ Ruby 2.5) ตัด "\n" ออกจากทุกบรรทัดให้อัตโนมัติ ไม่ต้อง .map(&:chomp) เอง
lines_no_newline = File.readlines("poem.txt", chomp: true)
p lines_no_newline
# => ["บรรทัดหนึ่ง", "บรรทัดสอง", "บรรทัดสาม"]

# File.write - เขียนทับทั้งไฟล์ในคำสั่งเดียว (เทียบเท่า File.open(path, "w") { |f| f.write(...) })
bytes_written = File.write("short.txt", "hi")
puts bytes_written
# => 2  (File.write คืนค่าจำนวน byte ที่เขียนสำเร็จ)
```

**อธิบาย:**

- `File.read`/`File.write`/`File.readlines` เป็น method ที่**เหมาะกับไฟล์ขนาดเล็กถึงกลาง**
  เพราะทำงานง่าย โค้ดสั้น อ่านง่าย ใช้บ่อยที่สุดในงานสคริปต์ทั่วไป
- `File.write(path, content)` เทียบเท่ากับ `File.open(path, "w") { |f| f.write(content) }`
  แต่สั้นกว่ามาก — ใช้ `"w"` mode เสมอ (เขียนทับทั้งไฟล์)
- ข้อควรระวัง: ทั้งสาม method นี้ **โหลดเนื้อหาทั้งหมดเข้า memory ในครั้งเดียว** ถ้าไฟล์มี
  ขนาดใหญ่มาก (เช่น หลาย GB) อาจทำให้โปรแกรมใช้ memory สูงเกินความจำเป็น — ประเด็นนี้จะอธิบาย
  ต่อใน Step 114

---

## Step 114: อ่านทีละบรรทัดด้วย `each_line`/`foreach` — ประหยัด memory กว่า `File.read` แค่ไหน

เมื่อไฟล์มีขนาดใหญ่ (หลักร้อย MB ถึงหลาย GB) การใช้ `File.read` จะโหลดข้อมูลทั้งหมดเข้า
memory พร้อมกัน ซึ่งอาจทำให้โปรแกรมช้าหรือถึงขั้น crash ได้ถ้า memory ไม่พอ ทางแก้คือ**อ่าน
ทีละบรรทัด** แทนที่จะอ่านทั้งไฟล์รวดเดียว

```ruby
# frozen_string_literal: true

File.open("big.txt", "w") do |f|
  1000.times { |i| f.puts "แถวที่ #{i}" }
end

# วิธีที่ไม่ประหยัด memory: โหลดทั้งไฟล์เป็น String เดียวก่อน แล้วค่อย split เป็นบรรทัด
count1 = 0
File.read("big.txt").each_line do |line|
  count1 += 1 if line.include?("9")
end
puts count1
# => 271

# วิธีที่ประหยัด memory: each_line อ่านทีละบรรทัดจาก disk ตรงๆ ไม่โหลดทั้งไฟล์เข้า memory
# พร้อมกัน (มี buffer ภายในเพียงเล็กน้อยเท่านั้น ไม่ว่าไฟล์จะใหญ่แค่ไหนก็ใช้ memory คงที่)
count2 = 0
File.open("big.txt") do |f|
  f.each_line do |line|
    count2 += 1 if line.include?("9")
  end
end
puts count2
# => 271

# File.foreach - shortcut ที่รวมการเปิด+อ่านทีละบรรทัด+ปิดไว้ในคำสั่งเดียว เหมาะกับงานอ่าน
# อย่างเดียวที่อยากได้โค้ดสั้นที่สุด
count3 = 0
File.foreach("big.txt") do |line|
  count3 += 1 if line.include?("9")
end
puts count3
# => 271
```

**อธิบาย:**

- `File.read("big.txt")` ต้องสร้าง String ที่มีเนื้อหาทั้งไฟล์อยู่ใน memory **ก่อน**ที่จะเริ่ม
  ประมวลผลอะไรเลย ถ้าไฟล์มีขนาด 2 GB โปรแกรมก็ต้องใช้ memory อย่างน้อย 2 GB ทันที
- `f.each_line` (หรือ `File.foreach`) อ่านไฟล์แบบ **streaming** — อ่านทีละบรรทัดจาก disk,
  ประมวลผล, แล้วค่อยอ่านบรรทัดถัดไป ไม่ต้องเก็บทั้งไฟล์ไว้ใน memory พร้อมกัน ทำให้ใช้ memory
  คงที่ (constant memory) ไม่ว่าไฟล์จะมีขนาดกี่ GB ก็ตาม
- **กฎการเลือกใช้ในทางปฏิบัติ:** ไฟล์เล็ก (ต่ำกว่าหลักสิบ MB, เช่น ไฟล์ config, ไฟล์ CSV
  ทั่วไปในงานธุรกิจขนาดกลาง) ใช้ `File.read`/`File.readlines` ได้สบายๆ เพราะโค้ดอ่านง่ายกว่า
  แต่พอเป็นไฟล์ log ขนาดใหญ่, ไฟล์ export ข้อมูลระดับล้านแถว, หรือสถานการณ์ที่ไม่รู้ขนาด
  ไฟล์ล่วงหน้า ควรใช้ `each_line`/`foreach` เป็นค่าเริ่มต้นไว้ก่อนเพื่อความปลอดภัย

---

## Step 115: ตรวจสอบไฟล์ก่อนใช้งาน — `File.exist?`, `File.delete`, `File.size`, `File.rename`

ก่อนอ่าน/เขียน/ลบไฟล์ ควรตรวจสอบสถานะของไฟล์ก่อนเสมอ เพื่อป้องกัน exception ที่ไม่จำเป็น

```ruby
# frozen_string_literal: true

path = "temp_note.txt"

puts File.exist?(path)
# => false

File.write(path, "ข้อมูลชั่วคราว")
puts File.exist?(path)
# => true

puts File.size(path)
# => 42   (ขนาดไฟล์เป็น byte)

puts File.file?(path)       # => true  (เป็นไฟล์ธรรมดา ไม่ใช่ directory)
puts File.directory?(path)  # => false

File.rename(path, "renamed_note.txt")
puts File.exist?(path)                    # => false (ชื่อเดิมหายไปแล้ว)
puts File.exist?("renamed_note.txt")      # => true

if File.exist?("renamed_note.txt")
  File.delete("renamed_note.txt")
  puts "ลบไฟล์แล้ว"
end
puts File.exist?("renamed_note.txt")
# => false

# ลบไฟล์ที่ไม่มีอยู่จริงโดยไม่เช็คก่อนจะ raise Errno::ENOENT
begin
  File.delete("no_such_file.txt")
rescue Errno::ENOENT => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
end

# Dir มี method คู่กันสำหรับจัดการ "โฟลเดอร์" แทนไฟล์
Dir.mkdir("reports") unless Dir.exist?("reports")
puts Dir.exist?("reports")
# => true
```

**อธิบาย:**

- **`File.exist?(path)`** คือ guard clause ที่ควรใช้ก่อนทุกครั้งที่จะอ่านไฟล์ที่ไม่แน่ใจว่ามี
  อยู่จริงหรือไม่ (เช่น ไฟล์ config ที่ผู้ใช้อาจยังไม่สร้าง) — ป้องกัน `Errno::ENOENT: No such
  file or directory`
- **`File.delete(path)`** (มี alias คือ `File.unlink`) ลบไฟล์ถาวร **ไม่มีถังขยะ กู้คืนไม่ได้**
  ควรเช็ค `File.exist?` ก่อนเสมอ หรือครอบด้วย `begin/rescue Errno::ENOENT` ถ้ายอมรับได้ว่า
  "ไม่มีไฟล์ให้ลบก็ไม่เป็นไร"
- **`File.size(path)`** คืนขนาดไฟล์เป็น byte มีประโยชน์เวลาต้องตัดสินใจว่าจะใช้ `File.read`
  หรือ `each_line` ตามที่พูดถึงใน Step 114 (เช่น ถ้า `File.size(path) > 100 * 1024 * 1024`
  ให้สลับไปใช้ streaming แทน)
- **`Dir.mkdir`/`Dir.exist?`** ทำงานแบบเดียวกับฝั่งไฟล์ แต่ใช้กับ **โฟลเดอร์** — สำคัญมาก
  เวลาจะเขียนไฟล์ลงโฟลเดอร์ที่อาจยังไม่มีอยู่ (เช่นโฟลเดอร์ `reports/`) ต้องสร้างโฟลเดอร์นั้น
  ก่อน ไม่เช่นนั้น `File.write("reports/x.txt", ...)` จะ error ทันทีถ้าโฟลเดอร์ `reports`
  ไม่มีอยู่จริง

---

## Step 116: จัดการ path — `File.join`, `File.expand_path`, `__dir__`, `Dir.glob`

โปรแกรมจริงแทบไม่เคย hardcode path แบบเต็มเส้นเลย เพราะโปรแกรมอาจถูกรันจากที่ไหนก็ได้ และ
ต้องทำงานได้ทั้งบน Linux/macOS (ใช้ `/` คั่น) และ Windows (ใช้ `\`) — Ruby มี method ที่จัดการ
เรื่องนี้ให้ครบ ไม่ต้องต่อ string เอง

```ruby
# frozen_string_literal: true

# File.join ประกอบ path จากชิ้นส่วนต่างๆ โดยไม่ต้องกังวลเรื่องเครื่องหมาย "/" ซ้ำหรือขาด
path = File.join("data", "reports", "2026", "summary.csv")
puts path
# => data/reports/2026/summary.csv

# __dir__ คือ path แบบเต็มของโฟลเดอร์ที่ไฟล์ .rb ปัจจุบันตั้งอยู่ (ไม่ขึ้นกับว่ารันคำสั่งจาก
# โฟลเดอร์ไหน) — ใช้บ่อยมากตอนอยากอ้างอิงไฟล์อื่นที่อยู่ "ข้างๆ" ไฟล์ปัจจุบันแบบชัวร์ๆ
puts __dir__
# => /path/to/current/script/folder

# File.expand_path แปลง relative path ให้กลายเป็น absolute path เต็มรูปแบบ
puts File.expand_path("data/reports")
# => /path/to/current/dir/data/reports

# ถ้าระบุ argument ตัวที่สอง จะใช้เป็น "จุดเริ่มต้น" ในการคำนวณ path แทน current directory
puts File.expand_path("../sibling_folder", __dir__)
# => /path/to/sibling_folder  (ขึ้นไปหนึ่งชั้นจาก __dir__ แล้วเข้าโฟลเดอร์ sibling_folder)

# Dir.glob ค้นหาไฟล์ตาม pattern (คล้าย wildcard ในหน้า command line เช่น *.csv)
Dir.mkdir("data") unless Dir.exist?("data")
File.write("data/a.csv", "x")
File.write("data/b.csv", "x")
File.write("data/notes.txt", "x")

csv_files = Dir.glob("data/*.csv")
p csv_files.sort
# => ["data/a.csv", "data/b.csv"]

all_files = Dir.glob("data/*")
p all_files.sort
# => ["data/a.csv", "data/b.csv", "data/notes.txt"]

# Dir.children คืนเฉพาะ "ชื่อไฟล์" (ไม่รวม path) ภายในโฟลเดอร์ ไม่รวม "." และ ".." ให้อัตโนมัติ
p Dir.children("data").sort
# => ["a.csv", "b.csv", "notes.txt"]
```

**อธิบาย:**

- **`File.join`** เป็นวิธีที่ถูกต้องในการต่อ path เสมอ แทนการเขียน `"data" + "/" + "reports"`
  เอง เพราะทำงานถูกต้องข้ามระบบปฏิบัติการ และจัดการเครื่องหมาย `/` ซ้ำ/ขาดให้อัตโนมัติ
- **`__dir__`** สำคัญมากเมื่อสคริปต์ต้องอ่านไฟล์ข้อมูลที่อยู่ในโฟลเดอร์เดียวกับตัวมันเอง เช่น
  `File.join(__dir__, "config.yml")` — วิธีนี้ทำงานถูกต้องเสมอไม่ว่าจะรันสคริปต์จากโฟลเดอร์
  ไหนก็ตาม ต่างจากการใช้ relative path เปล่าๆ ที่ผลลัพธ์จะขึ้นกับว่า "รันคำสั่งจากที่ไหน"
- **`File.expand_path`** ใช้เมื่อต้องการ path แบบเต็ม (absolute) เพื่อความชัดเจนไม่กำกวม เช่น
  ตอนเขียน log ว่า "บันทึกไฟล์ไว้ที่ไหน" ควรแสดง absolute path ให้ผู้ใช้เห็นชัดเจน
- **`Dir.glob`** เป็นเครื่องมือที่ทรงพลังมากสำหรับงาน batch processing เช่น "ประมวลผลไฟล์
  CSV ทุกไฟล์ในโฟลเดอร์ `imports/`" ทำได้ด้วย `Dir.glob("imports/*.csv").each { |f| ... }`
  รองรับ pattern ซับซ้อนได้ด้วย เช่น `Dir.glob("**/*.rb")` ค้นหาไฟล์ `.rb` ทุกไฟล์แบบ
  recursive ในทุกโฟลเดอร์ย่อย (`**` หมายถึง "ลึกกี่ชั้นก็ได้")

> **preview Rails:** `__dir__`/`File.join` คือรูปแบบเดียวกับที่เห็นในไฟล์ `config/application.rb`
> ของทุกโปรเจกต์ Rails (เช่น `require_relative "boot"`) และ `Dir.glob` คือกลไกเบื้องหลังที่
> Rails ใช้ค้นหาไฟล์ migration ทั้งหมดในโฟลเดอร์ `db/migrate/` ก่อนรัน `rails db:migrate`

---

## Step 117: `require "csv"` — อ่าน/เขียน CSV พร้อม header

**CSV** (Comma-Separated Values) คือ format ไฟล์ตารางข้อมูลที่ใช้กันแพร่หลายที่สุด (เปิดได้
ด้วย Excel/Google Sheets) Ruby standard library มี `CSV` class ให้ใช้ทันทีเพียง `require "csv"`
ไม่ต้องแยก parse comma เอง (ซึ่งอันตรายมาก เพราะข้อมูลอาจมี comma หรือ quote อยู่ในค่าเอง)

```ruby
# frozen_string_literal: true

require "csv"

# เขียน CSV พร้อม header แถวแรก
CSV.open("products.csv", "w") do |csv|
  csv << ["name", "category", "price", "stock"]
  csv << ["เมาส์ไร้สาย", "อุปกรณ์คอมพิวเตอร์", 350, 20]
  csv << ["คีย์บอร์ดกลไก", "อุปกรณ์คอมพิวเตอร์", 1200, 8]
  csv << ["เก้าอี้เกมมิ่ง", "เฟอร์นิเจอร์", 4500, 3]
  csv << ["โคมไฟตั้งโต๊ะ", "เฟอร์นิเจอร์", 590, 0]
end

puts File.read("products.csv")
# => name,category,price,stock
#    เมาส์ไร้สาย,อุปกรณ์คอมพิวเตอร์,350,20
#    คีย์บอร์ดกลไก,อุปกรณ์คอมพิวเตอร์,1200,8
#    เก้าอี้เกมมิ่ง,เฟอร์นิเจอร์,4500,3
#    โคมไฟตั้งโต๊ะ,เฟอร์นิเจอร์,590,0

# CSV.generate_line จัดการ escape ค่าที่มี comma อยู่ในตัวเองให้อัตโนมัติ ด้วยการครอบ ""
line = CSV.generate_line(["a", "b, มีจุลภาค", "c"])
print line
# => a,"b, มีจุลภาค",c
```

### อ่าน CSV แบบไม่มี header vs มี header

```ruby
# frozen_string_literal: true

require "csv"

# อ่านแบบธรรมดา ไม่ระบุ headers - แต่ละแถวเป็น Array ธรรมดา (รวมแถว header ปนมาด้วย)
CSV.foreach("products.csv") do |row|
  p row
end
# => ["name", "category", "price", "stock"]
#    ["เมาส์ไร้สาย", "อุปกรณ์คอมพิวเตอร์", "350", "20"]
#    ["คีย์บอร์ดกลไก", "อุปกรณ์คอมพิวเตอร์", "1200", "8"]
#    ...

# อ่านแบบ headers: true - แถวแรกถูกใช้เป็นชื่อ column แทนที่จะเป็นข้อมูล
# แต่ละแถวที่เหลือกลายเป็น CSV::Row เข้าถึงค่าด้วยชื่อ column ได้เหมือน Hash
CSV.foreach("products.csv", headers: true) do |row|
  puts "#{row['name']} - #{row['price']} บาท (คงเหลือ #{row['stock']})"
end
# => เมาส์ไร้สาย - 350 บาท (คงเหลือ 20)
#    คีย์บอร์ดกลไก - 1200 บาท (คงเหลือ 8)
#    เก้าอี้เกมมิ่ง - 4500 บาท (คงเหลือ 3)
#    โคมไฟตั้งโต๊ะ - 590 บาท (คงเหลือ 0)

# CSV.read (ไม่ใช่ foreach) โหลดทั้งไฟล์เข้า memory ครั้งเดียว คืนค่าเป็น CSV::Table
# เมื่อใช้ headers: true (ทำงานคล้าย Array ของ Hash)
table = CSV.read("products.csv", headers: true)
puts table.class   # => CSV::Table
puts table.size    # => 4

# แปลงเป็น Array of Hash ปกติ เพื่อใช้ Enumerable method ที่คุ้นเคยจาก Part 004/013 ต่อได้ทันที
products = table.map(&:to_h)
p products.first
# => {"name"=>"เมาส์ไร้สาย", "category"=>"อุปกรณ์คอมพิวเตอร์", "price"=>"350", "stock"=>"20"}

# ข้อควรระวังสำคัญที่สุดของ CSV: ทุกค่าที่อ่านได้เป็น String เสมอ ต้องแปลงชนิดเอง
puts products.first["price"].class
# => String  (ไม่ใช่ Integer! ต้อง .to_i เอง ถ้าจะเอาไปคำนวณ)
```

**อธิบาย:**

- `CSV.open(path, "w") { |csv| csv << [...] }` เหมือน `File.open` แต่ทำงานกับ CSV โดยเฉพาะ
  — `<<` เพิ่มหนึ่งแถว (row) ต่อครั้ง จัดการ escape comma/quote ให้อัตโนมัติ
- `CSV.foreach(path) { |row| ... }` คือ streaming version (คล้าย `File.foreach` ใน Step 114)
  เหมาะกับไฟล์ CSV ขนาดใหญ่ เพราะอ่านทีละแถวไม่โหลดทั้งไฟล์เข้า memory พร้อมกัน
- `headers: true` เปลี่ยนแถวแรกจาก "ข้อมูล" ให้กลายเป็น "ชื่อ column" — ทำให้เข้าถึงค่าด้วย
  `row["column_name"]` แทนที่จะต้องจำ index ตัวเลข (`row[0]`, `row[1]`) ซึ่งอ่านโค้ดยากกว่า
  มากและเปราะบางถ้ามีคนไปสลับลำดับ column ในไฟล์
- **ทุกค่าที่อ่านจาก CSV เป็น `String` เสมอ** ไม่มีข้อยกเว้น แม้ในไฟล์จะดูเหมือนตัวเลข
  (`"350"`) ก็ตาม — ต้อง `.to_i`/`.to_f` เองทุกครั้งก่อนนำไปคำนวณ นี่คือบัคที่มือใหม่พลาด
  บ่อยที่สุดเวลาทำงานกับ CSV (เช่น เอา `"350"` ไปบวกกับ `"1200"` จะได้ string
  `"3501200"` ไม่ใช่ `1550`)

---

## Step 118: `require "json"` — `JSON.generate`/`JSON.parse`, `symbolize_names`

**JSON** (JavaScript Object Notation) คือ format มาตรฐานสำหรับแลกเปลี่ยนข้อมูลระหว่างระบบ
โดยเฉพาะระหว่าง backend (Ruby/Rails) กับ frontend (JavaScript) หรือระหว่าง API ต่างๆ Ruby
standard library มี `JSON` module ให้แปลงไปมาระหว่าง Ruby object กับ JSON string ได้ทันที

```ruby
# frozen_string_literal: true

require "json"

data = {
  name: "เมาส์ไร้สาย",
  price: 350,
  in_stock: true,
  tags: ["คอมพิวเตอร์", "อุปกรณ์เสริม"],
  detail: nil
}

# Hash/Array -> JSON string ด้วย JSON.generate
json_string = JSON.generate(data)
puts json_string
# => {"name":"เมาส์ไร้สาย","price":350,"in_stock":true,"tags":["คอมพิวเตอร์","อุปกรณ์เสริม"],"detail":null}

# to_json ทำสิ่งเดียวกัน แค่เขียนสั้นกว่า (Ruby เพิ่ม method นี้ให้ Hash/Array/String/... ทุกตัว)
puts data.to_json

# pretty_generate จัดรูปแบบให้อ่านง่าย มีเว้นบรรทัดและเยื้อง เหมาะกับการเขียนลงไฟล์ให้คนอ่าน
puts JSON.pretty_generate(data)
# => {
#      "name": "เมาส์ไร้สาย",
#      "price": 350,
#      "in_stock": true,
#      "tags": [
#        "คอมพิวเตอร์",
#        "อุปกรณ์เสริม"
#      ],
#      "detail": null
#    }
```

### JSON string -> Ruby object และ `symbolize_names`

```ruby
# frozen_string_literal: true

require "json"

json_string = '{"name":"เมาส์ไร้สาย","price":350,"in_stock":true}'

# JSON.parse - แปลง JSON string กลับเป็น Ruby object โดยค่าเริ่มต้น key จะเป็น String เสมอ
parsed = JSON.parse(json_string)
p parsed
# => {"name"=>"เมาส์ไร้สาย", "price"=>350, "in_stock"=>true}
puts parsed["name"]           # => เมาส์ไร้สาย
puts parsed.keys.first.class  # => String

# symbolize_names: true ทำให้ key ทุกตัวกลายเป็น Symbol แทน String
# มีประโยชน์มากเวลาต้องการเข้าถึง key ด้วย .fetch(:name) แบบเดียวกับ Hash ที่เขียนในโค้ด Ruby เอง
parsed_sym = JSON.parse(json_string, symbolize_names: true)
p parsed_sym
# => {:name=>"เมาส์ไร้สาย", :price=>350, :in_stock=>true}
puts parsed_sym[:name]           # => เมาส์ไร้สาย
puts parsed_sym.keys.first.class # => Symbol

# เขียน/อ่านไฟล์ JSON จริง (pattern ที่ใช้บ่อยที่สุด)
File.write("product.json", JSON.pretty_generate(parsed))
loaded = JSON.parse(File.read("product.json"), symbolize_names: true)
p loaded

# JSON string ที่ format ผิด จะ raise JSON::ParserError เสมอ ต้อง rescue ถ้ารับข้อมูลจาก
# แหล่งภายนอกที่ไม่น่าเชื่อถือ (เช่น response จาก API หรือ input จากผู้ใช้)
begin
  JSON.parse("{invalid json")
rescue JSON::ParserError => e
  puts "parse error: #{e.class}"
end
# => parse error: JSON::ParserError
```

**อธิบาย:**

- `JSON.generate`/`to_json` และ `JSON.parse` เป็นคู่ตรงข้ามกันเสมอ — `generate` แปลง Ruby ->
  JSON string, `parse` แปลง JSON string -> Ruby
- ค่าเริ่มต้นของ `JSON.parse` คืน Hash ที่มี **key เป็น String เสมอ** ต่างจาก Hash literal ที่
  เขียนในโค้ด Ruby ที่มักใช้ Symbol (`name:` คือ `:name`) — นี่คือเหตุผลที่ `symbolize_names:
  true` มีประโยชน์มาก โดยเฉพาะเมื่อจะเอาผลลัพธ์ไปใช้กับ method ที่คาดหวัง keyword argument
  หรือ pattern matching แบบ Hash
- Type ที่ JSON รองรับมีจำกัดกว่า Ruby: String, Number (Integer/Float), Boolean
  (`true`/`false`), `null` (แปลงเป็น `nil`), Array, Object (แปลงเป็น Hash) — Symbol, Range,
  Time ในรูปแบบดั้งเดิม ฯลฯ **ไม่มีใน JSON โดยตรง** ต้องแปลงเป็น String ก่อนเสมอ (เช่น
  `time.iso8601` ก่อนใส่ลง Hash ที่จะ `to_json`)
- ควร `rescue JSON::ParserError` เสมอเมื่อ parse ข้อมูลที่มาจากแหล่งภายนอก (API, ไฟล์ที่
  ผู้ใช้ upload มา) เพราะข้อมูลอาจไม่ใช่ JSON ที่ถูกต้องเสมอไป

> **preview Rails:** เมื่อเขียน Rails API (Phase 8) `render json: @product` เบื้องหลังคือ
> การเรียก `to_json` แบบเดียวกันนี้เป๊ะๆ และเวลารับ JSON request body เข้ามา Rails ก็ parse
> ด้วยกลไกเดียวกับ `JSON.parse` ที่เพิ่งเรียนไป

---

## Step 119: `require "yaml"` — `YAML.load`/`YAML.dump` และทำไม Rails ใช้ YAML เป็นไฟล์ config

**YAML** (YAML Ain't Markup Language) คือ format ที่เน้นให้**มนุษย์อ่าน/แก้ไขเองได้ง่าย**
มากกว่า JSON — ไม่มีวงเล็บปีกกา ไม่มี comma ท้ายบรรทัด ใช้การเยื้องบรรทัด (indentation) แทน
โครงสร้าง จึงเป็น format ยอดนิยมสำหรับ **ไฟล์ config** ที่คนต้องเปิดมาแก้ไขด้วยมือบ่อยๆ

```ruby
# frozen_string_literal: true

require "yaml"

config = {
  "app_name" => "MyShop",
  "version" => 1.2,
  "features" => ["cart", "wishlist", "reviews"],
  "database" => {
    "host" => "localhost",
    "port" => 5432,
    "pool" => 5
  }
}

# YAML.dump - แปลง Ruby object เป็น YAML string
yaml_string = YAML.dump(config)
puts yaml_string
# => ---
#    app_name: MyShop
#    version: 1.2
#    features:
#    - cart
#    - wishlist
#    - reviews
#    database:
#      host: localhost
#      port: 5432
#      pool: 5

# YAML.load - แปลง YAML string กลับเป็น Ruby object
loaded = YAML.load(yaml_string)
p loaded
# => {"app_name"=>"MyShop", "version"=>1.2, "features"=>["cart", "wishlist", "reviews"],
#     "database"=>{"host"=>"localhost", "port"=>5432, "pool"=>5}}
puts loaded["database"]["host"]
# => localhost

# เขียน/อ่านไฟล์ .yml จริง - YAML.load_file สะดวกกว่า YAML.load(File.read(...)) เพราะจัดการ
# เปิด-อ่าน-ปิดไฟล์ให้ในคำสั่งเดียว
File.write("config.yml", YAML.dump(config))
loaded_from_file = YAML.load_file("config.yml")
p loaded_from_file
```

### ข้อควรรู้สำคัญ: Ruby 3.1+ (Psych 4) ทำให้ `YAML.load` ปลอดภัยเป็นค่าเริ่มต้นแล้ว

ใน Ruby รุ่นเก่า `YAML.load` เคยเป็นอันตรายถ้ารับ YAML จากแหล่งที่ไม่น่าเชื่อถือ เพราะสามารถ
สร้าง Ruby object ชนิดอันตรายขึ้นมาได้ตามที่ระบุใน YAML — จึงมี `YAML.safe_load` แยกไว้ใช้เพื่อ
ความปลอดภัย **แต่ตั้งแต่ Ruby 3.1 เป็นต้นมา (Psych เวอร์ชัน 4) `YAML.load` ทำงานแบบปลอดภัย
เหมือน `safe_load` เป็นค่าเริ่มต้นแล้ว** (อนุญาตแค่ type พื้นฐาน เช่น String, Integer, Array,
Hash) ยังคงเห็น `YAML.safe_load` อยู่ในโค้ดเก่าจำนวนมาก แต่ในโค้ดใหม่ใช้ `YAML.load` ได้เลย

ผลข้างเคียงที่สำคัญของความปลอดภัยนี้คือ **`YAML.load` ไม่อนุญาตให้ใช้ YAML anchor/alias
(`&`/`*`) โดยอัตโนมัติ** ต้องเปิดใช้งานเองผ่าน `aliases: true` — ซึ่งเป็นเทคนิคที่ไฟล์
`config/database.yml` ของ Rails ใช้บ่อยมาก (ดูตัวอย่างด้านล่าง)

```ruby
# frozen_string_literal: true

require "yaml"

# ตัวอย่าง YAML ที่ใช้ anchor (&default) และ alias (*default) เพื่อไม่ต้องเขียนค่าซ้ำ
# รูปแบบนี้คือรูปแบบเดียวกับที่เห็นในไฟล์ config/database.yml ของ Rails ทุกโปรเจกต์
database_yaml = <<~YAML
  default: &default
    adapter: postgresql
    pool: 5

  development:
    <<: *default
    database: myapp_development

  production:
    <<: *default
    database: myapp_production
YAML

# ต้องระบุ aliases: true เพื่ออนุญาตให้ resolve *default -> ค่าจาก &default
db_config = YAML.load(database_yaml, aliases: true)
p db_config
# => {"default"=>{"adapter"=>"postgresql", "pool"=>5},
#     "development"=>{"adapter"=>"postgresql", "pool"=>5, "database"=>"myapp_development"},
#     "production"=>{"adapter"=>"postgresql", "pool"=>5, "database"=>"myapp_production"}}

puts db_config["development"]["adapter"]  # => postgresql (สืบทอดมาจาก default ผ่าน alias)
puts db_config["production"]["database"]  # => myapp_production

# ถ้าไม่ใส่ aliases: true จะ raise Psych::AliasesNotEnabled ทันที
begin
  YAML.load(database_yaml)
rescue Psych::AliasesNotEnabled => e
  puts "error: #{e.class}"
end
# => error: Psych::AliasesNotEnabled
```

**อธิบาย:**

- `&default` คือการตั้ง **anchor** (จุดอ้างอิง) ให้กับ block ค่าหนึ่ง ส่วน `*default` คือ
  **alias** ที่ดึงค่าจาก anchor นั้นมาใช้ซ้ำ, `<<: *default` เป็นการ **merge** ค่าจาก anchor
  เข้ากับ key ปัจจุบัน (คล้ายการทำ `default.merge(...)` ใน Ruby) — ทำให้ไม่ต้องพิมพ์
  `adapter: postgresql` และ `pool: 5` ซ้ำในทุก environment
- **ทำไม Rails ใช้ YAML เป็นไฟล์ config มาตรฐาน (`config/database.yml`,
  `config/credentials.yml.enc`, locale file `config/locales/th.yml`):**
  1. **อ่าน/แก้ไขง่ายด้วยมือ** — ไม่มี syntax รกอย่าง `{}`, `,`, `""` ทุกจุดเหมือน JSON
     เหมาะกับไฟล์ที่ DevOps/ทีม infra ต้องเปิดแก้บ่อยๆ
  2. **รองรับหลาย environment ในไฟล์เดียว** ผ่าน anchor/alias อย่างที่เห็นข้างบน — ไม่ต้อง
     แยกไฟล์ `database_development.yml`, `database_production.yml` ให้ซ้ำซ้อน
  3. **แปลงเป็น Ruby Hash ได้ทันทีด้วย `YAML.load_file`** โดยไม่ต้องเขียน parser เอง ตรงตาม
     โครงสร้างข้อมูลที่ Ruby ใช้งานอยู่แล้วพอดี
- **สิ่งที่จะเจอจริงตอนเข้า Phase 3 (Rails Fundamentals):** ไฟล์ `config/database.yml` จะมี
  โครงสร้างเกือบเหมือนตัวอย่างข้างบนนี้เป๊ะๆ เพียงแต่ Rails จะโหลดผ่าน ERB ก่อนด้วย (เพื่อ
  แทรกค่าจาก environment variable เช่น `<%= ENV["DATABASE_PASSWORD"] %>`) แล้วค่อยส่งผลลัพธ์
  ให้ `YAML.load` ประมวลผลต่อ — กลไกเบื้องหลังคือสิ่งที่เพิ่งเรียนใน Step นี้ทั้งหมด

---

## Step 120: แบบฝึกหัดปิด Part — สร้างรายงานสต็อกสินค้าจาก CSV ส่งออกเป็น JSON และ YAML

### โจทย์

เขียนสคริปต์ `inventory_report.rb` ที่ทำงานดังนี้:

1. อ่านข้อมูลสินค้าจากไฟล์ `inventory.csv` (มี column: `name`, `category`, `price`, `stock`)
2. แปลงค่า `price`/`stock` จาก String เป็นตัวเลข
3. คำนวณและกรองข้อมูล:
   - รายการสินค้าที่ **ใกล้หมดสต็อก** (stock น้อยกว่าค่าที่กำหนด)
   - รายการสินค้าที่ **หมดสต็อกแล้ว** (stock เท่ากับ 0)
   - สรุปยอดตาม **หมวดหมู่** (จำนวนสินค้า และมูลค่าสต็อกรวมต่อหมวดหมู่)
   - มูลค่าสต็อกรวมทั้งหมด
4. ส่งออกผลลัพธ์แบบละเอียดเป็นไฟล์ `report.json` (สำหรับส่งต่อให้ระบบอื่นอ่าน)
5. ส่งออกผลสรุปแบบย่อเป็นไฟล์ `summary.yml` (สำหรับให้คนอ่านตรวจสอบด้วยตา)

### เฉลย

```ruby
# frozen_string_literal: true

# inventory_report.rb
require "csv"
require "json"
require "yaml"

INPUT_FILE = "inventory.csv"
JSON_REPORT_FILE = "report.json"
YAML_SUMMARY_FILE = "summary.yml"
LOW_STOCK_THRESHOLD = 5

unless File.exist?(INPUT_FILE)
  abort "ไม่พบไฟล์ #{INPUT_FILE} กรุณาสร้างไฟล์ก่อนรันสคริปต์นี้"
end

# ---------- 1) อ่านและแปลงชนิดข้อมูล ----------
products = CSV.read(INPUT_FILE, headers: true).map do |row|
  {
    name: row["name"],
    category: row["category"],
    price: row["price"].to_i,
    stock: row["stock"].to_i
  }
end

# ---------- 2) กรองและคำนวณ ----------
low_stock_products = products.select { |p| p[:stock] < LOW_STOCK_THRESHOLD }
out_of_stock_products = products.select { |p| p[:stock].zero? }

by_category = products.group_by { |p| p[:category] }
category_summary = by_category.transform_values do |items|
  {
    "count" => items.size,
    "total_stock_value" => items.sum { |p| p[:price] * p[:stock] }
  }
end

total_stock_value = products.sum { |p| p[:price] * p[:stock] }

# ---------- 3) สร้างรายงาน JSON แบบละเอียด ----------
report = {
  generated_products_count: products.size,
  total_stock_value: total_stock_value,
  low_stock_threshold: LOW_STOCK_THRESHOLD,
  low_stock_products: low_stock_products,
  out_of_stock_products: out_of_stock_products,
  products: products
}

File.write(JSON_REPORT_FILE, JSON.pretty_generate(report))
puts "สร้างรายงาน JSON แล้วที่ #{JSON_REPORT_FILE}"

# ---------- 4) สร้างสรุป YAML แบบย่อ ----------
summary = {
  "generated_at" => Time.now.strftime("%Y-%m-%d"),
  "total_products" => products.size,
  "total_stock_value" => total_stock_value,
  "categories" => category_summary.transform_keys(&:to_s),
  "alerts" => {
    "low_stock_count" => low_stock_products.size,
    "out_of_stock_count" => out_of_stock_products.size,
    "out_of_stock_names" => out_of_stock_products.map { |p| p[:name] }
  }
}

File.write(YAML_SUMMARY_FILE, YAML.dump(summary))
puts "สร้างสรุป YAML แล้วที่ #{YAML_SUMMARY_FILE}"
```

### ทดสอบรัน

สมมติไฟล์ `inventory.csv` มีเนื้อหาดังนี้:

```
name,category,price,stock
เมาส์ไร้สาย,อุปกรณ์คอมพิวเตอร์,350,20
คีย์บอร์ดกลไก,อุปกรณ์คอมพิวเตอร์,1200,8
เก้าอี้เกมมิ่ง,เฟอร์นิเจอร์,4500,3
โคมไฟตั้งโต๊ะ,เฟอร์นิเจอร์,590,0
หูฟังบลูทูธ,อุปกรณ์คอมพิวเตอร์,990,2
จอมอนิเตอร์ 24 นิ้ว,อุปกรณ์คอมพิวเตอร์,3990,12
```

```bash
ruby inventory_report.rb
# สร้างรายงาน JSON แล้วที่ report.json
# สร้างสรุป YAML แล้วที่ summary.yml
```

ไฟล์ `report.json` ที่ได้ (ตัดมาเฉพาะส่วนสำคัญ):

```json
{
  "generated_products_count": 6,
  "total_stock_value": 79960,
  "low_stock_threshold": 5,
  "low_stock_products": [
    { "name": "เก้าอี้เกมมิ่ง", "category": "เฟอร์นิเจอร์", "price": 4500, "stock": 3 },
    { "name": "โคมไฟตั้งโต๊ะ", "category": "เฟอร์นิเจอร์", "price": 590, "stock": 0 },
    { "name": "หูฟังบลูทูธ", "category": "อุปกรณ์คอมพิวเตอร์", "price": 990, "stock": 2 }
  ],
  "out_of_stock_products": [
    { "name": "โคมไฟตั้งโต๊ะ", "category": "เฟอร์นิเจอร์", "price": 590, "stock": 0 }
  ],
  "products": [ "... สินค้าทั้ง 6 รายการแบบเต็ม ..." ]
}
```

ไฟล์ `summary.yml` ที่ได้ (แสดงผลจริงทั้งหมด):

```yaml
---
generated_at: '2026-09-26'
total_products: 6
total_stock_value: 79960
categories:
  อุปกรณ์คอมพิวเตอร์:
    count: 4
    total_stock_value: 66460
  เฟอร์นิเจอร์:
    count: 2
    total_stock_value: 13500
alerts:
  low_stock_count: 3
  out_of_stock_count: 1
  out_of_stock_names:
  - โคมไฟตั้งโต๊ะ
```

โค้ดนี้ทดสอบรันจริงแล้วบน Ruby 3.3.6 ผลลัพธ์ตรงตามที่แสดงไว้ทุกจุด

### จุดที่ควรสังเกตในเฉลย

- **`unless File.exist?(INPUT_FILE)` + `abort`** — ตรวจสอบไฟล์ก่อนเสมอตามที่เรียนใน Step 115
  `abort(message)` พิมพ์ข้อความไปที่ stderr แล้วหยุดโปรแกรมทันทีด้วย exit code ที่ไม่ใช่ 0
  เหมาะกับ error ที่ทำให้โปรแกรมทำงานต่อไม่ได้เลย
- **`CSV.read(..., headers: true).map { ... }`** แปลง `CSV::Table` เป็น Array of Hash พร้อม
  แปลงชนิดข้อมูล (`.to_i`) ในขั้นตอนเดียว — เป็น pattern มาตรฐานเมื่อรับข้อมูลจาก CSV เข้ามา
  ประมวลผลต่อในโปรแกรม (ทบทวน Step 117 เรื่อง CSV คืนค่าเป็น String เสมอ)
- **`group_by` + `transform_values`** สร้างสรุปตามหมวดหมู่โดยไม่ต้องเขียน loop ซ้อน loop เอง
  — `group_by { |p| p[:category] }` จัดกลุ่มสินค้าตามหมวดหมู่ก่อน แล้ว `transform_values`
  แปลงแต่ละกลุ่ม (Array ของสินค้า) ให้กลายเป็นสรุปตัวเลข (จำนวน + มูลค่ารวม) — นี่คือตัวอย่าง
  เล็กๆ ของ Enumerable ขั้นสูงที่จะเรียนเต็มรูปแบบใน Part 013
- **`report` เก็บ key เป็น Symbol ส่วน `summary` เก็บ key เป็น String โดยตั้งใจ** — `report`
  จะถูก `to_json` ซึ่ง JSON แปลง Symbol เป็น String ให้อัตโนมัติอยู่แล้วจึงไม่มีปัญหา แต่
  `summary` จะถูก `YAML.dump` ตรงๆ ถ้าปล่อยเป็น Symbol จะได้ YAML ที่มี key แบบ `:count:`
  (มี `:` นำหน้า) ซึ่งอ่านแปลกกว่าปกติ — จึงแปลงเป็น String ด้วย `transform_keys(&:to_s)` ก่อน
  เพื่อให้ไฟล์ YAML ที่ได้อ่านง่ายและเป็นธรรมชาติสำหรับคนเปิดดูด้วยตา ตรงตามจุดประสงค์ของไฟล์
  summary ที่เน้นความอ่านง่ายเป็นหลัก (ตามที่อธิบายใน Step 119)
- **แยกไฟล์ JSON (สำหรับระบบอ่านต่อ) กับไฟล์ YAML (สำหรับคนอ่าน)** สะท้อนหลักการเลือก format
  ตามผู้ใช้งานปลายทาง: JSON เหมาะกับการส่งต่อให้โปรแกรมอื่นประมวลผลต่อ (compact, มาตรฐานสากล)
  ส่วน YAML เหมาะกับรายงานสรุปที่มนุษย์จะเปิดอ่านโดยตรง — เป็นการตัดสินใจแบบเดียวกับที่วิศวกร
  ต้องทำจริงเวลาออกแบบระบบ export ข้อมูล

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มการตรวจสอบข้อมูลผิดพลาด (validation)** — ถ้าในไฟล์ CSV มีแถวที่ `price` หรือ
   `stock` ไม่ใช่ตัวเลข (เช่น เว้นว่างหรือมีตัวอักษรปน) ให้ข้ามแถวนั้นไปพร้อมพิมพ์คำเตือน
   แทนที่จะให้โปรแกรม error หรือคำนวณผิดเงียบๆ (ใบ้: ใช้ regex `/\A\d+\z/` ตรวจสอบก่อน
   `.to_i` เหมือนที่เคยฝึกใน Part 001 Step 10)
2. **รองรับหลายไฟล์ CSV พร้อมกัน** — แก้โปรแกรมให้ใช้ `Dir.glob("inventory/*.csv")` (ทบทวน
   Step 116) อ่านไฟล์ CSV ทุกไฟล์ในโฟลเดอร์ `inventory/` มารวมเป็นรายการสินค้าเดียวกันก่อน
   คำนวณสรุป — เหมาะกับสถานการณ์ที่แต่ละสาขาส่งออกไฟล์สต็อกของตัวเองมาแยกไฟล์กัน
3. **เขียนโปรแกรมย้อนกลับ** — เขียนสคริปต์ `import_from_json.rb` ที่อ่านไฟล์ `report.json`
   ที่สร้างไว้ กลับมาเป็น Ruby object แล้วเขียนออกเป็นไฟล์ CSV ใหม่ชื่อ `inventory_export.csv`
   (ต้องใช้ทั้ง `JSON.parse` และ `CSV.open` ร่วมกัน — เป็นสถานการณ์ที่พบบ่อยเวลาต้อง sync
   ข้อมูลระหว่างระบบสองระบบที่ใช้ format ต่างกัน)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เปิด/ปิดไฟล์ด้วย `File.open` ทั้งแบบ block form (auto-close ปลอดภัยกว่า) และแบบ manual
  open/close พร้อมเข้าใจว่าทำไมควรใช้ block form เป็นค่าเริ่มต้น
- แยกความแตกต่างของ read mode `"r"`, `"w"`, `"a"`, `"r+"` ได้ชัดเจน โดยเฉพาะข้อควรระวังของ
  `"r+"` ที่ไม่ล้างไฟล์ก่อนเขียนทับ
- ใช้ `File.read`/`File.write`/`File.readlines` สำหรับงานอ่าน-เขียนไฟล์ขนาดเล็กถึงกลางแบบ
  รวดเร็ว
- เข้าใจความแตกต่างของการอ่านทั้งไฟล์ (`File.read`) กับการอ่านทีละบรรทัด (`each_line`/
  `foreach`) และรู้ว่าเมื่อไหร่ควรเลือกแบบไหนตามขนาดไฟล์
- ตรวจสอบและจัดการไฟล์ได้ครบวงจรด้วย `File.exist?`, `File.delete`, `File.size`,
  `File.rename`, `Dir.mkdir`, `Dir.exist?`
- จัดการ path ข้ามระบบปฏิบัติการได้ถูกต้องด้วย `File.join`, `File.expand_path`, `__dir__`
  และค้นหาไฟล์ตาม pattern ด้วย `Dir.glob`
- อ่าน/เขียนไฟล์ **CSV** พร้อม header ด้วย `require "csv"` และรู้ข้อควรระวังว่าค่าทุกค่าจาก
  CSV เป็น String เสมอ
- แปลง Ruby object ไปมากับ **JSON** ด้วย `JSON.generate`/`JSON.parse`/`to_json` และใช้
  `symbolize_names` ให้เหมาะกับการใช้งานต่อ
- แปลง Ruby object ไปมากับ **YAML** ด้วย `YAML.dump`/`YAML.load`/`YAML.load_file` เข้าใจเรื่อง
  ความปลอดภัยของ `YAML.load` ใน Ruby 3.1+ และเหตุผลที่ Rails เลือกใช้ YAML เป็นไฟล์ config
  มาตรฐาน (เช่น `config/database.yml`)
- สร้างสคริปต์รวบยอดที่อ่านข้อมูลจาก CSV แปลง/กรอง/สรุปข้อมูล แล้วส่งออกเป็นทั้ง JSON และ YAML
  ตามความเหมาะสมของผู้ใช้งานปลายทาง

**ต่อไป (Part 013):** เราจะกลับมาเจาะลึก **Enumerable module ขั้นสูง** ต่อจากที่แตะไปเล็กน้อย
ใน Part นี้ (`group_by`, `transform_values`) — จะเรียนรู้ method ทรงพลังอีกชุดที่ใช้บ่อยมากใน
งานประมวลผลข้อมูลจริง ได้แก่ `each_slice` (แบ่งข้อมูลเป็นกลุ่มย่อยตามจำนวน), `partition`
(แบ่งข้อมูลเป็นสองกลุ่มตามเงื่อนไข), `flat_map` (map แล้วแบนข้อมูลซ้อนให้เรียบในขั้นตอนเดียว)
และ **lazy enumerator** (ประมวลผลข้อมูลขนาดใหญ่หรือไม่จำกัด โดยไม่โหลดทั้งหมดเข้า memory —
แนวคิดต่อยอดโดยตรงจากเรื่อง memory efficiency ที่เพิ่งเรียนใน Step 114 ของ Part นี้)
