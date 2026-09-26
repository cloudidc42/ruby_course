# Part 011: Exception Handling — begin/rescue/ensure, Custom Exception, retry

> **Step ครอบคลุมใน Part นี้:** Step 101–110
> **ระดับ:** ปานกลาง (ต้องผ่าน Phase 1 ทั้งหมดมาก่อน โดยเฉพาะ Part 009–010 เรื่อง Class/Object และ Inheritance)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6)

Part นี้คือ Part แรกของ **Phase 2: Ruby Deep Dive** เราเพิ่งปิด Phase 1 ไปด้วยการสร้างโปรแกรม
OOP เต็มรูปแบบใน Part 010 — ตอนนี้ถึงเวลาเปลี่ยนโหมดจาก "เขียนโค้ดให้ทำงานได้" มาเป็น
**"เขียนโค้ดให้ทนทานต่อความผิดพลาดในโลกจริง"** ซึ่งเป็นทักษะที่แยกโปรแกรมเมอร์มือใหม่กับ
มืออาชีพออกจากกันอย่างชัดเจน โปรแกรมจริงต้องรับมือกับไฟล์ที่ไม่มีอยู่ การเชื่อมต่อ network
ที่ล่ม ผู้ใช้กรอกข้อมูลผิด และเงื่อนไขทางธุรกิจที่ไม่เป็นไปตามคาด — **Exception Handling**
คือกลไกของ Ruby ที่ให้เราจัดการกับสถานการณ์เหล่านี้อย่างเป็นระบบ แทนที่จะปล่อยให้โปรแกรม
ล่มกลางคันโดยไม่มีคำอธิบาย

## สารบัญของ Part นี้

- Step 101: กายวิภาคของ `begin`/`rescue`/`else`/`ensure` แบบเต็มรูปแบบ
- Step 102: `raise` — วิธีโยน Exception ทั้ง 4 รูปแบบ และการ re-raise
- Step 103: Exception Class Hierarchy — ทำไมต้อง rescue `StandardError` ไม่ใช่ `Exception`
- Step 104: Rescue เฉพาะเจาะจง และการเรียงลำดับ `rescue` หลายบรรทัด
- Step 105: Rescue หลายชนิด Exception ในบรรทัดเดียว
- Step 106: สร้าง Custom Exception Class ของตัวเอง
- Step 107: `retry` — รูปแบบการลองทำงานซ้ำแบบ Retry with Backoff
- Step 108: การรับประกันของ `ensure` — แม้มี `return`/`raise` ซ้อนอยู่ข้างใน
- Step 109: Anti-pattern — "rescue ทุกอย่างแล้วกลืนหายไปเงียบๆ" อันตรายอย่างไร
- Step 110: แบบฝึกหัดรวบยอด — Payment Processing Simulator พร้อม Custom Exception, Retry, Ensure

---

## Step 101: กายวิภาคของ `begin`/`rescue`/`else`/`ensure` แบบเต็มรูปแบบ

**Exception** คือสัญญาณที่ Ruby ส่งออกมาเมื่อเกิดสถานการณ์ผิดปกติระหว่างการทำงานของโปรแกรม
เช่น หารด้วยศูนย์, เรียก method ที่ไม่มีอยู่, หรือแปลงชนิดข้อมูลไม่ได้ ถ้าไม่มีการจัดการ
Exception จะ "ระเบิด" ขึ้นไปเรื่อยๆ จนถึงจุดบนสุดของโปรแกรมแล้วทำให้โปรแกรมหยุดทำงานทันที
พร้อมพิมพ์ error message และ backtrace ออกมา

```ruby
# frozen_string_literal: true

# ถ้าไม่จัดการ Exception เลย โปรแกรมจะหยุดทำงานทันที
def divide(a, b)
  a / b
end

puts divide(10, 2)
# => 5

puts divide(10, 0)
# => ZeroDivisionError (divided by 0)
#    โปรแกรมหยุดทำงานตรงนี้ทันที บรรทัดถัดไปจะไม่ถูกรันเลย
```

โครงสร้างพื้นฐานที่สุดสำหรับ "ดัก" Exception คือ `begin`/`rescue`/`end`:

```ruby
begin
  result = 10 / 0
rescue ZeroDivisionError => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
  result = nil
end

puts result.inspect
# => เกิดข้อผิดพลาด: divided by 0
# => nil
```

แต่ `begin` ยังมีอีก 2 ส่วนที่มือใหม่มักไม่รู้จัก คือ `else` และ `ensure` — กายวิภาคเต็มรูปแบบ
มีลำดับดังนี้:

```ruby
# frozen_string_literal: true

def process(value)
  begin
    puts "1) begin: กำลังประมวลผล #{value}..."
    result = 100 / value
  rescue ZeroDivisionError => e
    puts "2) rescue: จับ error ได้ -> #{e.message}"
    result = 0
  else
    puts "3) else: ทำงานเฉพาะเมื่อ begin สำเร็จ ไม่มี error เกิดขึ้นเลย"
    result *= 2
  ensure
    puts "4) ensure: ทำงานเสมอ ไม่ว่าจะสำเร็จ, เกิด error, หรือ error ไม่ถูกจับ"
  end

  result
end

puts process(5)
# 1) begin: กำลังประมวลผล 5...
# 3) else: ทำงานเฉพาะเมื่อ begin สำเร็จ ไม่มี error เกิดขึ้นเลย
# 4) ensure: ทำงานเสมอ ไม่ว่าจะสำเร็จ, เกิด error, หรือ error ไม่ถูกจับ
# => 40

puts process(0)
# 1) begin: กำลังประมวลผล 0...
# 2) rescue: จับ error ได้ -> divided by 0
# 4) ensure: ทำงานเสมอ ไม่ว่าจะสำเร็จ, เกิด error, หรือ error ไม่ถูกจับ
# => 0
```

**อธิบายลำดับการทำงาน:**

| ส่วน | ทำงานเมื่อไหร่ |
|---|---|
| `begin` | โค้ดหลักที่ "อาจ" เกิด Exception |
| `rescue ExceptionClass => e` | ทำงานเฉพาะเมื่อ `begin` โยน Exception ที่ตรงกับ `ExceptionClass` |
| `else` | ทำงานเฉพาะเมื่อ `begin` **ไม่มี Exception เกิดขึ้นเลย** (ตรงข้ามกับ `rescue`) |
| `ensure` | ทำงาน **เสมอ** ไม่ว่า `begin` จะสำเร็จ, ตกไปที่ `rescue`, หรือแม้แต่ Exception นั้นจะ**ไม่ถูกจับเลยก็ตาม** |

**ทำไมต้องมี `else` ทั้งที่เขียนต่อท้าย `begin` เลยก็ได้:** เพราะถ้าเขียนโค้ดที่ "ควรทำงาน
เฉพาะตอนสำเร็จ" ไว้ต่อจากบรรทัดที่อาจ error ใน `begin` ตรงๆ แล้วบังเอิญโค้ดนั้นเองก็ throw
Exception ประเภทเดียวกันขึ้นมาอีกที Exception นั้นจะถูก `rescue` จับไปด้วยอย่างไม่ตั้งใจ
การแยกไว้ใน `else` ทำให้ชัดเจนว่า "ส่วนนี้ไม่ได้อยู่ในขอบเขตการดัก error อีกต่อไป"

### `begin`/`end` แบบไม่ต้องมี block ครอบทั้ง method

ในทางปฏิบัติ ถ้า method ทั้ง method ต้องการ `rescue`/`ensure` ครอบทั้งหมดอยู่แล้ว ไม่จำเป็น
ต้องเขียน `begin`...`end` ซ้อนเข้าไปข้างใน — Ruby อนุญาตให้เขียน `rescue`/`ensure` ต่อท้าย
`def` ได้โดยตรง (เรียกว่า **implicit begin** ของ method):

```ruby
# frozen_string_literal: true

# แบบที่ 1: มี begin/end ซ้อนอยู่ข้างใน (ซ้ำซ้อนโดยไม่จำเป็น)
def divide_v1(a, b)
  begin
    a / b
  rescue ZeroDivisionError
    nil
  end
end

# แบบที่ 2: ใช้ implicit begin ของ method เอง (แนวปฏิบัติที่แนะนำ เมื่อ rescue ครอบทั้ง method)
def divide_v2(a, b)
  a / b
rescue ZeroDivisionError
  nil
end

puts divide_v1(10, 0)  # => nil
puts divide_v2(10, 0)  # => nil
```

ทั้งสองแบบให้ผลเหมือนกันทุกประการ แต่แบบที่ 2 อ่านง่ายกว่าและเป็นสำนวนมาตรฐานที่พบใน
โค้ด Ruby/Rails มืออาชีพ — จะใช้ `begin`/`end` ซ้อนเข้าไปข้างในก็ต่อเมื่อต้องการ `rescue`
เฉพาะบางส่วนของ method เท่านั้น ไม่ใช่ทั้ง method

> **แนวคิดสำคัญ:** `block`, `def`, และ `class`/`module` ทุกตัวใน Ruby มี implicit begin
> ในตัวเองอยู่แล้ว หมายความว่าเขียน `rescue`/`ensure` ต่อท้าย `do...end` block ได้เช่นกัน
> (ตั้งแต่ Ruby 2.6 เป็นต้นมา) แต่รูปแบบที่พบบ่อยที่สุดในทางปฏิบัติคือต่อท้าย `def`

---

## Step 102: `raise` — วิธีโยน Exception ทั้ง 4 รูปแบบ และการ re-raise

`raise` คือคำสั่งที่ใช้ "โยน" (throw) Exception ออกมาเอง มี 4 รูปแบบหลักที่ต้องแยกให้ออก:

```ruby
# frozen_string_literal: true

# รูปแบบ 1: raise "ข้อความ" -> เป็น shorthand ของ raise RuntimeError, "ข้อความ"
def check_v1(age)
  raise "อายุต้องไม่ติดลบ" if age.negative?

  age
end

# รูปแบบ 2: raise ExceptionClass, "ข้อความ" -> ระบุชนิด Exception ให้ตรงตามความหมาย
def check_v2(age)
  raise ArgumentError, "อายุต้องไม่ติดลบ" if age.negative?

  age
end

# รูปแบบ 3: raise ExceptionClass.new("ข้อความ") -> เทียบเท่ารูปแบบ 2 ทุกประการ
def check_v3(age)
  raise ArgumentError.new("อายุต้องไม่ติดลบ") if age.negative?

  age
end

# รูปแบบ 4: raise ExceptionClass -> ไม่มีข้อความ จะใช้ชื่อ class เป็น message แทน
def check_v4(age)
  raise ArgumentError if age.negative?

  age
end

begin
  check_v1(-5)
rescue RuntimeError => e
  puts "v1: #{e.class} -> #{e.message}"
end

begin
  check_v2(-5)
rescue ArgumentError => e
  puts "v2: #{e.class} -> #{e.message}"
end

begin
  check_v4(-5)
rescue ArgumentError => e
  puts "v4: #{e.class} -> #{e.message}"
end

# v1: RuntimeError -> อายุต้องไม่ติดลบ
# v2: ArgumentError -> อายุต้องไม่ติดลบ
# v4: ArgumentError -> ArgumentError
```

**อธิบาย:**

- `raise "ข้อความ"` สะดวกแต่ **ควรหลีกเลี่ยงในโค้ดจริง** เพราะมันโยน `RuntimeError` เสมอ
  ทำให้ผู้เรียก `rescue` แยกไม่ออกว่า error นี้เกิดจากสาเหตุอะไรกันแน่ ถ้าเทียบกับ error
  อื่นๆ ในระบบที่ก็เป็น `RuntimeError` เหมือนกันหมด
- `raise ArgumentError, "ข้อความ"` คือรูปแบบที่แนะนำที่สุดในทางปฏิบัติ — ระบุทั้ง**ชนิด**
  (ใช้ในการ `rescue` แยกแยะ) และ**ข้อความ** (อธิบายรายละเอียดให้มนุษย์อ่าน) ในบรรทัดเดียว
- `raise ArgumentError.new("ข้อความ")` ให้ผลเหมือนกับข้างบนทุกประการ แต่ยาวกว่า — จะเลือกใช้
  รูปแบบนี้เมื่อต้องสร้าง exception object ที่มี attribute พิเศษเพิ่มเติม (จะเห็นใน Step 106)
- `raise ArgumentError` (ไม่มีข้อความ) ใช้ได้แต่ข้อความที่ได้จะเป็นแค่ชื่อ class เฉยๆ
  ซึ่งไม่ช่วยให้ debug ง่ายขึ้นเลย ควรใส่ข้อความอธิบายเสมอในทางปฏิบัติ

### Bare `raise` — re-raise Exception ปัจจุบัน

เมื่อเขียน `raise` **โดยไม่มี argument ใดๆ เลย** ภายใน `rescue` block ความหมายจะไม่ใช่
"โยน RuntimeError เปล่าๆ" แต่คือ **re-raise Exception ตัวเดิมที่เพิ่งถูกจับได้** พร้อม
backtrace เดิมครบถ้วน:

```ruby
# frozen_string_literal: true

def risky_operation
  raise ArgumentError, "ข้อมูลไม่ถูกต้อง"
end

def process_with_logging
  risky_operation
rescue ArgumentError => e
  puts "[LOG] บันทึก error ไว้ก่อน: #{e.message}"
  raise  # re-raise ตัวเดิม (ArgumentError, "ข้อมูลไม่ถูกต้อง") ต่อขึ้นไปให้ผู้เรียกจัดการ
end

begin
  process_with_logging
rescue ArgumentError => e
  puts "จับได้อีกครั้งที่ชั้นบนสุด: #{e.message}"
end

# [LOG] บันทึก error ไว้ก่อน: ข้อมูลไม่ถูกต้อง
# จับได้อีกครั้งที่ชั้นบนสุด: ข้อมูลไม่ถูกต้อง
```

รูปแบบนี้เรียกว่า **"log and re-raise"** — เหมาะมากเมื่อต้องการบันทึก log หรือทำความสะอาด
บางอย่างที่จุดหนึ่ง แต่ยังต้องการให้ error เดินทางต่อไปให้โค้ดชั้นบนสุด (เช่น web framework)
เป็นคนตัดสินใจว่าจะแสดงหน้า error ยังไงให้ผู้ใช้เห็น

> **ข้อควรระวัง:** ถ้าเขียน `raise` เปล่าๆ นอก `rescue` block (คือตอนนั้นไม่มี Exception
> ใดๆ กำลังถูกจัดการอยู่เลย ตัวแปรพิเศษ `$!` เป็น `nil`) Ruby จะโยน `RuntimeError` ที่มี
> ข้อความ `"unhandled exception"` แทน — จึงควรใช้ bare `raise` เฉพาะภายใน `rescue` เท่านั้น
> เพื่อให้ความหมายชัดเจนว่ากำลัง re-raise

---

## Step 103: Exception Class Hierarchy — ทำไมต้อง rescue `StandardError` ไม่ใช่ `Exception`

Exception ทุกตัวใน Ruby เป็น object ที่สืบทอดมาจาก class `Exception` (ทบทวน Inheritance
จาก Part 010) โครงสร้างลำดับชั้น (ย่อเฉพาะส่วนสำคัญ) มีดังนี้:

```
Exception
├── NoMemoryError
├── ScriptError
│   ├── LoadError
│   ├── NotImplementedError
│   └── SyntaxError
├── SecurityError
├── SignalException
│   └── Interrupt                 # เกิดตอนผู้ใช้กด Ctrl+C
├── SystemExit                     # เกิดตอนเรียก Kernel#exit
├── SystemStackError                # stack overflow
└── StandardError                  # <<< base class ของ error ที่ "เกิดจากการรันโค้ดตามปกติ"
    ├── ArgumentError
    │   └── UncaughtThrowError
    ├── EncodingError
    ├── FiberError
    ├── IOError
    │   └── EOFError
    ├── IndexError
    │   └── KeyError
    ├── LocalJumpError
    ├── NameError
    │   └── NoMethodError
    ├── RangeError
    │   └── FloatDomainError
    ├── RegexpError
    ├── RuntimeError                # <<< default class เมื่อเขียน raise "ข้อความ" เฉยๆ
    │   └── FrozenError
    ├── ThreadError
    ├── TypeError
    └── ZeroDivisionError
```

ดูโครงสร้างจริงในเครื่องได้ด้วย:

```ruby
puts ArgumentError.ancestors.first(5).inspect
# => [ArgumentError, StandardError, Exception, Object, Kernel]

puts StandardError.superclass
# => Exception
```

**กฎที่สำคัญที่สุดในเรื่อง Exception ทั้งหมดของ Ruby:**

```ruby
# เมื่อเขียน rescue โดยไม่ระบุ class เลย -> Ruby จะ rescue เฉพาะ StandardError (และ subclass)
# โดยอัตโนมัติ ไม่ใช่ rescue ทุกอย่างที่สืบทอดจาก Exception
begin
  raise ArgumentError, "จับได้เพราะเป็น StandardError"
rescue => e
  puts "จับได้: #{e.class}"
end
# => จับได้: ArgumentError

begin
  raise SystemExit, "ผู้ใช้เรียก exit"
rescue => e
  puts "จับได้: #{e.class}"
end
# ผลลัพธ์: โปรแกรมออกทันที ไม่มีอะไรถูกพิมพ์ เพราะ rescue เปล่าๆ ไม่จับ SystemExit
```

**ทำไม `rescue` เปล่าๆ ถึงจับแค่ `StandardError`:** เพราะ Exception ที่อยู่**นอก**
`StandardError` (เช่น `SystemExit`, `Interrupt`, `NoMemoryError`, `SecurityError`,
`SystemStackError`) ล้วนเป็นสัญญาณของสถานการณ์ที่**ร้ายแรงระดับระบบ** ซึ่งโปรแกรมไม่ควร
พยายาม "แก้ปัญหาต่อ" — เช่น ถ้า `rescue` ดัก `Interrupt` (สัญญาณจากการกด Ctrl+C) ไปด้วย
ผู้ใช้จะกด Ctrl+C เพื่อหยุดโปรแกรมไม่ได้อีกเลย! หรือถ้าดัก `SystemExit` ไปด้วย การเรียก
`exit` ในโปรแกรมจะไม่ทำให้โปรแกรมออกจริงๆ

### ทำไมไม่ควรเขียน `rescue Exception => e` (แม้จะระบุ class ชัดเจน)

```ruby
# frozen_string_literal: true

# อันตราย: rescue Exception จับทุกอย่างจริงๆ รวมถึง SystemExit, Interrupt, NoMemoryError
def broken_loop
  loop do
    begin
      puts "กำลังทำงาน... (กด Ctrl+C เพื่อหยุด)"
      sleep 1
    rescue Exception => e   # <<< ห้ามทำแบบนี้!
      puts "จับ error ได้: #{e.class}"
    end
  end
end

# broken_loop เมื่อรันจริง: กด Ctrl+C จะไม่หยุดโปรแกรมเลย เพราะ Interrupt ถูก
# rescue Exception จับไปเงียบๆ ทุกครั้ง ผู้ใช้ต้องปิด terminal ทิ้งสถานเดียว
```

> **กฎทองของ Exception Handling ใน Ruby:** เขียน `rescue` เปล่าๆ หรือ
> `rescue StandardError => e` เกือบทุกครั้งในโค้ดธุรกิจปกติ **ห้ามเขียน
> `rescue Exception => e` เด็ดขาด** ยกเว้นกรณีพิเศษสุดๆ ระดับ framework/top-level handler
> ที่ต้องการ log ทุกอย่างก่อนโปรแกรมจะปิดตัวจริงๆ (ซึ่งถึงตอนนั้นก็ยังต้อง re-raise
> `SystemExit`/`Interrupt` ต่อไปอยู่ดี) — RuboCop มี cop ชื่อ `Lint/RescueException` ที่คอย
> เตือนเรื่องนี้โดยเฉพาะ

### `#message`, `#class`, `#backtrace` — ข้อมูลที่ติดมากับทุก Exception object

```ruby
begin
  1 / 0
rescue => e
  puts e.class        # => ZeroDivisionError
  puts e.message       # => divided by 0
  puts e.backtrace.first  # => บรรทัดและไฟล์ที่เกิด error (มีประโยชน์มากตอน debug production)
end
```

`backtrace` คือ Array ของ String บอก "เส้นทาง" ที่ error เดินทางผ่านมา (จากจุดเกิด error
ไล่ย้อนขึ้นไปถึงจุดเรียก) — เครื่องมือ error tracking อย่าง Sentry (จะเรียนใน Phase 12)
ใช้ข้อมูลนี้เป็นหลักในการแสดงผลให้ทีมพัฒนาสามารถหาสาเหตุของ error ใน production ได้

> **preview Rails:** Rails สร้าง custom Exception ของตัวเองมากมายที่สืบทอดจาก
> `StandardError` เช่น `ActiveRecord::RecordNotFound` (หา record ไม่เจอ),
> `ActiveRecord::RecordInvalid` (validation ไม่ผ่าน), `ActionController::RoutingError`
> (route ไม่ตรงกับที่กำหนด) — ทั้งหมดนี้ใช้หลักการเดียวกับที่เรียนใน Step นี้ทุกประการ
> เพียงแต่ Rails เตรียม exception class และ handler มาตรฐานไว้ให้ล่วงหน้าเท่านั้นเอง

---

## Step 104: Rescue เฉพาะเจาะจง และการเรียงลำดับ `rescue` หลายบรรทัด

`begin` หนึ่งก้อนสามารถมี `rescue` ได้หลายบรรทัด แต่ละบรรทัดดักคนละชนิด Exception —
Ruby จะไล่ตรวจสอบทีละบรรทัด**จากบนลงล่าง** และใช้บรรทัดแรกที่ตรงกับชนิดของ Exception ที่
เกิดขึ้นจริงเท่านั้น (คล้ายกับ `case`/`when`)

```ruby
# frozen_string_literal: true

def parse_and_divide(numerator_str, denominator_str)
  numerator = Integer(numerator_str)      # โยน ArgumentError ถ้าแปลงเป็นตัวเลขไม่ได้
  denominator = Integer(denominator_str)
  numerator / denominator                  # โยน ZeroDivisionError ถ้า denominator เป็น 0
rescue ZeroDivisionError => e
  puts "หารด้วยศูนย์ไม่ได้: #{e.message}"
  nil
rescue ArgumentError => e
  puts "ข้อมูลไม่ใช่ตัวเลข: #{e.message}"
  nil
rescue StandardError => e
  puts "เกิดข้อผิดพลาดอื่นที่ไม่คาดคิด: #{e.class} - #{e.message}"
  nil
end

puts parse_and_divide("10", "2")     # => 5
puts parse_and_divide("10", "0")     # หารด้วยศูนย์ไม่ได้: divided by 0
puts parse_and_divide("abc", "2")    # ข้อมูลไม่ใช่ตัวเลข: invalid value for Integer(): "abc"
```

**กฎการเรียงลำดับที่สำคัญมาก: เรียงจาก "เฉพาะเจาะจงที่สุด" ไปหา "กว้างที่สุด" เสมอ**

```ruby
# frozen_string_literal: true

# ผิด! -- StandardError กว้างกว่า ZeroDivisionError จึงดักไปก่อนเสมอ
# ทำให้ rescue ZeroDivisionError ด้านล่างไม่มีวันถูกใช้งานเลย (RuboCop เตือนเป็น dead code)
def broken_order(a, b)
  a / b
rescue StandardError => e
  puts "ทั่วไป: #{e.class}"
rescue ZeroDivisionError => e
  puts "หารศูนย์: #{e.class}"  # <<< บรรทัดนี้ไม่มีวันถูกเรียกเลย!
end

broken_order(10, 0)
# => ทั่วไป: ZeroDivisionError   (ไม่ใช่ "หารศูนย์" ตามที่ตั้งใจ)
```

เพราะ `ZeroDivisionError` **เป็น** `StandardError` ด้วย (ผ่าน inheritance chain) เมื่อ Ruby
เจอ `rescue StandardError` เป็นบรรทัดแรกที่ตรงเงื่อนไข มันจะหยุดค้นหาทันทีและไม่มีทางไปถึง
บรรทัด `rescue ZeroDivisionError` ที่อยู่ด้านล่างเลย — RuboCop cop
`Lint/DuplicateBranch`/`Lint/UselessRescue` จะเตือนกรณีแบบนี้โดยอัตโนมัติในโค้ดจริง

> **แนวคิดสำคัญ:** จำง่ายๆ ว่า **"เฉพาะเจาะจงอยู่บน กว้างอยู่ล่าง"** เหมือนกับการเขียน
> `case`/`when` ที่ต้องเรียง condition แคบไปกว้างเช่นกัน — และควรวาง
> `rescue StandardError => e` (ถ้ามี) ไว้เป็นบรรทัดสุดท้ายเสมอ เพื่อดัก error ประเภทอื่นที่
> ไม่คาดคิดไว้เป็นตาข่ายรองรับสุดท้าย (safety net) เท่านั้น

---

## Step 105: Rescue หลายชนิด Exception ในบรรทัดเดียว

บางครั้งเราต้องการจัดการ Exception หลายชนิดด้วย**พฤติกรรมเดียวกันเป๊ะๆ** — แทนที่จะเขียน
`rescue` แยกกันหลายบรรทัดแล้วเขียนโค้ดซ้ำ (ผิดหลัก DRY จาก Part 001) Ruby อนุญาตให้ระบุ
หลาย class คั่นด้วยจุลภาคในบรรทัดเดียวได้:

```ruby
# frozen_string_literal: true

require "net/http"

def fetch_config(source)
  case source
  when :file then File.read("config.yml")
  when :network then raise Timeout::Error, "network timeout"
  when :bad_format then raise ArgumentError, "invalid format"
  end
end

def load_config(source)
  fetch_config(source)
rescue Errno::ENOENT, ArgumentError, Timeout::Error => e
  # ทั้ง 3 ชนิดนี้ไม่มีความสัมพันธ์ inheritance ร่วมกันโดยตรง แต่จัดการเหมือนกัน:
  # ใช้ค่า default แทน
  puts "โหลด config ไม่สำเร็จ (#{e.class}: #{e.message}) ใช้ค่า default แทน"
  "default_config"
end

puts load_config(:network)
# => โหลด config ไม่สำเร็จ (Timeout::Error: network timeout) ใช้ค่า default แทน
# => default_config

puts load_config(:bad_format)
# => โหลด config ไม่สำเร็จ (ArgumentError: invalid format) ใช้ค่า default แทน
# => default_config
```

**อธิบาย:**

- `rescue Errno::ENOENT, ArgumentError, Timeout::Error => e` ดักทั้ง 3 ชนิดพร้อมกันในบรรทัด
  เดียว — Exception ที่เกิดขึ้นจริงจะถูก assign เข้าตัวแปร `e` เหมือนเดิม ไม่ว่าจะเป็นชนิดไหน
  ในลิสต์นี้
- ใช้รูปแบบนี้เมื่อ **การจัดการเหมือนกันทุกประการ** ระหว่างหลาย Exception เท่านั้น ถ้า
  แต่ละชนิดต้องมี logic ตอบสนองต่างกัน ให้แยกเป็นคนละ `rescue` บรรทัด ตามที่เรียนใน
  Step 104 แทน — อย่าฝืนรวมเข้าด้วยกันแล้วใช้ `case e when ... end` ข้างในเพื่อแยกพฤติกรรม
  เพราะจะทำให้อ่านยากกว่าแยก `rescue` แต่ต้น
- สามารถผสมกับ `rescue` หลายบรรทัดปกติได้ในก้อนเดียวกัน เช่น มี
  `rescue TypeError, ArgumentError => e` หนึ่งบรรทัด แล้วตามด้วย
  `rescue StandardError => e` อีกบรรทัดเป็นตาข่ายรองรับสุดท้าย

```ruby
# ตัวอย่างผสมทั้งสองแบบในก้อนเดียว
def safe_calculate(a, b, operation)
  case operation
  when :divide then a / b
  when :parse  then Integer(a) + Integer(b)
  else raise NotImplementedError, "ไม่รู้จัก operation: #{operation}"
  end
rescue ZeroDivisionError, ArgumentError => e
  puts "ปัญหาจากข้อมูล input: #{e.message}"
  nil
rescue NotImplementedError => e
  puts "ปัญหาจาก operation ที่ไม่รองรับ: #{e.message}"
  nil
end
```

---

## Step 106: สร้าง Custom Exception Class ของตัวเอง

Exception มาตรฐานของ Ruby (`ArgumentError`, `TypeError`, ฯลฯ) ใช้ได้กับ error ทั่วไปในระดับ
ภาษา แต่เมื่อเขียนโปรแกรมที่มี**เงื่อนไขทางธุรกิจ** (business logic) ของตัวเอง เช่น
"ยอดเงินไม่พอถอน" หรือ "สินค้าหมดสต็อก" การสร้าง **Custom Exception Class** ที่สื่อความหมาย
ตรงตัวจะทำให้โค้ดอ่านง่ายขึ้นมหาศาล และให้ผู้เรียก `rescue` เฉพาะเจาะจงตามสถานการณ์ทางธุรกิจ
ได้จริง

### รูปแบบพื้นฐานที่สุด: สืบทอดจาก `StandardError` เปล่าๆ

```ruby
# frozen_string_literal: true

class InsufficientFundsError < StandardError; end

class BankAccount
  attr_reader :balance

  def initialize(balance)
    @balance = balance
  end

  def withdraw(amount)
    raise InsufficientFundsError, "ยอดเงินไม่พอ (มี #{balance} บาท)" if amount > balance

    @balance -= amount
  end
end

account = BankAccount.new(100)

begin
  account.withdraw(500)
rescue InsufficientFundsError => e
  puts "ทำรายการไม่สำเร็จ: #{e.message}"
end
# => ทำรายการไม่สำเร็จ: ยอดเงินไม่พอ (มี 100 บาท)
```

แค่นี้ `InsufficientFundsError` ก็เป็น Exception ที่สมบูรณ์แล้ว เพราะสืบทอดพฤติกรรมทั้งหมด
มาจาก `StandardError` (ซึ่งสืบทอดมาจาก `Exception` อีกที) — รับ `message` ผ่าน `raise`,
มี `.class`, `.backtrace` ครบทุกอย่างเหมือน Exception มาตรฐานทุกประการ

> **สำคัญ:** เสมอสืบทอดจาก `StandardError` (หรือ subclass ของมัน) **ไม่ใช่**
> `Exception` ตรงๆ ด้วยเหตุผลเดียวกับ Step 103 — ถ้า custom exception สืบทอดจาก
> `Exception` ตรงๆ, `rescue => e` เปล่าๆ ของผู้เรียกจะไม่จับมันเลย ทำให้ error หลุดรอด
> ไปทำให้โปรแกรมล่มโดยไม่ได้ตั้งใจ

### ใส่ attribute และ `initialize` ของตัวเองให้ Exception

Exception ธรรมดามีแค่ `message` แต่บางสถานการณ์เราต้องการแนบ**ข้อมูลเพิ่มเติม**ไปกับ error
ด้วย เช่น จำนวนเงินที่ขาด หรือ record ID ที่มีปัญหา — ทำได้โดย override `initialize` และ
เรียก `super` เพื่อส่งข้อความไปตั้งค่า `message` ตามปกติ (ทบทวนหลักการ `super` จาก Part 010
Step 92):

```ruby
# frozen_string_literal: true

class InsufficientFundsError < StandardError
  attr_reader :balance, :amount_requested, :shortfall

  def initialize(balance:, amount_requested:)
    @balance = balance
    @amount_requested = amount_requested
    @shortfall = amount_requested - balance
    super("ยอดเงินไม่พอ: มี #{balance} บาท แต่ขอถอน #{amount_requested} บาท " \
          "(ขาดอีก #{@shortfall} บาท)")
  end
end

class BankAccount
  attr_reader :balance

  def initialize(balance)
    @balance = balance
  end

  def withdraw(amount)
    if amount > balance
      raise InsufficientFundsError.new(balance: balance, amount_requested: amount)
    end

    @balance -= amount
  end
end

account = BankAccount.new(100)

begin
  account.withdraw(350)
rescue InsufficientFundsError => e
  puts e.message
  puts "ยอดคงเหลือปัจจุบัน: #{e.balance}"
  puts "จำนวนที่ขอถอน: #{e.amount_requested}"
  puts "ต้องเติมเงินอีก: #{e.shortfall} บาท จึงจะถอนได้"
end

# ยอดเงินไม่พอ: มี 100 บาท แต่ขอถอน 350 บาท (ขาดอีก 250 บาท)
# ยอดคงเหลือปัจจุบัน: 100
# จำนวนที่ขอถอน: 350
# ต้องเติมเงินอีก: 250 บาท จึงจะถอนได้
```

**อธิบาย:**

- `initialize` ของ custom exception รับ keyword argument (`balance:`, `amount_requested:`)
  เหมือน class ทั่วไปที่เคยเรียนใน Part 009 ทุกประการ — Exception ก็คือ object ธรรมดา
  ชนิดหนึ่งที่ Ruby ให้ความหมายพิเศษเรื่องการ `raise`/`rescue` เท่านั้นเอง
- **ต้องเรียก `super(message_string)`** เสมอ เพื่อส่งข้อความไปให้ `Exception#initialize`
  ตั้งค่า `.message` ให้ถูกต้อง ถ้าลืมเรียก `super` เลย `e.message` จะกลับไปเป็นแค่ชื่อ class
  เฉยๆ (พฤติกรรม default ของ `Exception#message` เมื่อไม่เคยตั้งค่าไว้)
- เพราะ `attr_reader :balance, :amount_requested, :shortfall` ผู้เรียก `rescue` จึงเข้าถึง
  ข้อมูลดิบเหล่านี้ได้โดยตรง ไม่ต้อง parse เอาจาก `message` string (ซึ่งเปราะบางมาก ถ้า
  format ข้อความเปลี่ยนในอนาคตโค้ดที่ parse จะพังทันที) — นี่คือประโยชน์หลักของการสร้าง
  custom exception ที่มี attribute ของตัวเอง

### จัดกลุ่ม custom exception ด้วย base class ร่วม

เมื่อโปรเจกต์มี custom exception หลายตัว นิยมสร้าง base error class กลางของแอปพลิเคชันไว้
หนึ่งตัว แล้วให้ error เฉพาะเจาะจงทั้งหมดสืบทอดจากมัน:

```ruby
# frozen_string_literal: true

module BankingErrors
  class Error < StandardError; end                          # base ของทั้งระบบ
  class InsufficientFundsError < Error; end
  class AccountFrozenError < Error; end
  class InvalidAmountError < Error; end
end

def process_transaction
  raise BankingErrors::AccountFrozenError, "บัญชีถูกระงับ"
end

begin
  process_transaction
rescue BankingErrors::Error => e
  # rescue ตัวเดียวจับ error ทุกชนิดที่เกี่ยวกับ banking ได้หมด เพราะทุกตัวสืบทอดจาก
  # BankingErrors::Error ร่วมกัน โดยไม่ต้องเขียน rescue แยกทีละชนิด
  puts "เกิดข้อผิดพลาดเกี่ยวกับระบบธนาคาร: #{e.class} - #{e.message}"
end
# => เกิดข้อผิดพลาดเกี่ยวกับระบบธนาคาร: BankingErrors::AccountFrozenError - บัญชีถูกระงับ
```

รูปแบบนี้ให้ทั้งความยืดหยุ่นในการ `rescue` แบบเฉพาะเจาะจง (`rescue InsufficientFundsError`)
และแบบกว้าง (`rescue BankingErrors::Error`) ในระบบเดียวกัน — เป็นสถาปัตยกรรมมาตรฐานที่ gem
และแอปพลิเคชัน Ruby ระดับ production เกือบทุกตัวใช้ (สังเกตว่าใช้ `module` เป็น namespace
ตามที่เรียนใน Part 010 Step 95 ด้วย)

---

## Step 107: `retry` — รูปแบบการลองทำงานซ้ำแบบ Retry with Backoff

บาง Exception เป็น**ความผิดพลาดชั่วคราว** (transient error) ที่ถ้าลองใหม่อีกครั้งอาจสำเร็จ
ได้ เช่น network timeout, database connection ล่มชั่วขณะ, API ปลายทางตอบช้าเกินเวลา —
สถานการณ์แบบนี้เหมาะกับการใช้คำสั่ง `retry` ซึ่งจะกระโดดกลับไปเริ่ม `begin` block ใหม่
ตั้งแต่ต้น

```ruby
# frozen_string_literal: true

class NetworkTimeoutError < StandardError; end

attempts = 0

begin
  attempts += 1
  puts "พยายามเชื่อมต่อ ครั้งที่ #{attempts}..."
  raise NetworkTimeoutError, "connection timeout" if attempts < 3

  puts "เชื่อมต่อสำเร็จ!"
rescue NetworkTimeoutError => e
  puts "  ล้มเหลว: #{e.message}"
  retry if attempts < 3
end

# พยายามเชื่อมต่อ ครั้งที่ 1...
#   ล้มเหลว: connection timeout
# พยายามเชื่อมต่อ ครั้งที่ 2...
#   ล้มเหลว: connection timeout
# พยายามเชื่อมต่อ ครั้งที่ 3...
# เชื่อมต่อสำเร็จ!
```

**อธิบาย:** `retry` ทำให้ interpreter กระโดดกลับไปที่บรรทัดแรกของ `begin` แล้วรันใหม่
ทั้งหมด (ตัวแปร `attempts` ยังคงค่าเดิมไว้เพราะประกาศไว้**นอก** `begin`) — ถ้าไม่มีเงื่อนไข
จำกัดจำนวนครั้ง (`if attempts < 3`) โปรแกรมจะ `retry` วนไม่รู้จบเมื่อยังคง error อยู่เรื่อยๆ
ซึ่งเป็นอันตรายมาก **ต้องมีเพดานจำนวนครั้งเสมอ**

### รูปแบบมาตรฐานในโลกจริง: Exponential Backoff

การ `retry` ทันทีติดกันรัวๆ (เช่น เรียก API ที่กำลังล่มซ้ำๆ 100 ครั้งต่อวินาที) อาจทำให้
ปัญหาแย่ลงไปอีก (เรียกว่า "thundering herd") แนวปฏิบัติมาตรฐานในโลก production คือ
**Exponential Backoff** — เพิ่มเวลารอเป็นเท่าตัวในแต่ละครั้งที่ล้มเหลว:

```ruby
# frozen_string_literal: true

class NetworkTimeoutError < StandardError; end

def fetch_from_flaky_api(fail_until_attempt: 3)
  attempts = 0
  max_attempts = 5

  begin
    attempts += 1
    puts "[attempt #{attempts}] กำลังเรียก API..."
    raise NetworkTimeoutError, "API timeout" if attempts < fail_until_attempt

    puts "[attempt #{attempts}] สำเร็จ! ได้ข้อมูลกลับมา"
    { status: "ok", data: [1, 2, 3] }
  rescue NetworkTimeoutError => e
    if attempts < max_attempts
      wait_seconds = 2**attempts  # 2, 4, 8, 16 วินาที ตามลำดับ
      puts "[attempt #{attempts}] ล้มเหลว: #{e.message} -> รอ #{wait_seconds} วินาทีแล้วลองใหม่"
      sleep(0.01)  # ในตัวอย่างจริงจะใช้ sleep(wait_seconds) แต่ลดเวลาไว้เพื่อสาธิตให้เร็ว
      retry
    else
      puts "[attempt #{attempts}] ลองครบ #{max_attempts} ครั้งแล้วยังไม่สำเร็จ ยอมแพ้"
      raise  # re-raise ให้ผู้เรียกระดับบนสุดตัดสินใจต่อ (เช่น แจ้งผู้ใช้, ส่ง alert)
    end
  end
end

result = fetch_from_flaky_api(fail_until_attempt: 3)
puts result.inspect
# [attempt 1] กำลังเรียก API...
# [attempt 1] ล้มเหลว: API timeout -> รอ 2 วินาทีแล้วลองใหม่
# [attempt 2] กำลังเรียก API...
# [attempt 2] ล้มเหลว: API timeout -> รอ 4 วินาทีแล้วลองใหม่
# [attempt 3] กำลังเรียก API...
# [attempt 3] สำเร็จ! ได้ข้อมูลกลับมา
# => {:status=>"ok", :data=>[1, 2, 3]}

begin
  fetch_from_flaky_api(fail_until_attempt: 99)  # ตั้งใจให้ fail ตลอด ไม่มีวันถึง attempt 99
rescue NetworkTimeoutError => e
  puts "โปรแกรมหลักจับ error ที่ retry ครบแล้วได้: #{e.message}"
end
# ... (attempt 1-5 ล้มเหลวหมด) ...
# [attempt 5] ลองครบ 5 ครั้งแล้วยังไม่สำเร็จ ยอมแพ้
# โปรแกรมหลักจับ error ที่ retry ครบแล้วได้: API timeout
```

**อธิบาย:**

- `2**attempts` คือการคำนวณเวลารอแบบ**เพิ่มขึ้นทวีคูณ** (exponential) — ครั้งที่ 1 รอ 2
  วินาที, ครั้งที่ 2 รอ 4 วินาที, ครั้งที่ 3 รอ 8 วินาที ไปเรื่อยๆ ยิ่งล้มเหลวติดต่อกันนาน
  เท่าไหร่ ยิ่งรอนานขึ้นเพื่อลดโหลดที่ปลายทาง — เป็นสูตรมาตรฐานที่ gem อย่าง `retriable`
  หรือ background job framework อย่าง Sidekiq (จะเรียนใน Phase 9) ใช้จริง
- ต้องมี **เพดานจำนวนครั้งสูงสุด** (`max_attempts`) เสมอ พร้อม `else`/`raise` เมื่อครบแล้ว
  ยังไม่สำเร็จ — ไม่เช่นนั้นโปรแกรมจะติดอยู่ใน retry loop ตลอดไปถ้าปัญหาไม่ใช่ปัญหาชั่วคราว
  จริงๆ (เช่น API key ผิด ซึ่ง retry ไปเท่าไหร่ก็ไม่มีวันสำเร็จ)
- เมื่อ retry ครบแล้วยังไม่สำเร็จ ควร `raise` (re-raise) ต่อให้ผู้เรียกจัดการ ไม่ใช่กลืน
  error ไว้เงียบๆ (เดี๋ยวจะเห็นว่าทำไมสำคัญมากใน Step 109)

> **ข้อควรระวัง:** `retry` ใช้ได้เฉพาะภายใน `rescue` block เท่านั้น (หรือใน `begin` ที่มี
> `rescue` คู่กัน) ถ้าเขียน `retry` ไว้นอกบริบทนี้จะเกิด `SyntaxError` ทันที และควร `retry`
> เฉพาะ Exception ที่ "มีโอกาสหายเองได้จริง" เท่านั้น เช่น network/timeout error — ไม่ควร
> `retry` กับ error ประเภท validation หรือ business logic เช่น `InsufficientFundsError`
> เพราะยอดเงินจะไม่มีวันเพิ่มขึ้นเองไม่ว่าจะลองกี่ครั้งก็ตาม

---

## Step 108: การรับประกันของ `ensure` — แม้มี `return`/`raise` ซ้อนอยู่ข้างใน

`ensure` คือส่วนที่ Ruby **รับประกัน** ว่าจะทำงานเสมอ ไม่ว่าจะเกิดอะไรขึ้นใน `begin` ก็ตาม
— ใช้บ่อยที่สุดสำหรับการ**ทำความสะอาดทรัพยากร** (cleanup) เช่น ปิดไฟล์, ปิดการเชื่อมต่อ
database, คืน connection กลับ pool

### `ensure` ทำงานแม้ `begin` จะจบด้วย `return`

```ruby
# frozen_string_literal: true

def read_setting
  puts "เปิดไฟล์ config..."
  return "production"  # <<< return ออกจาก method ทันที
ensure
  puts "ปิดไฟล์ config เสมอ ไม่ว่า return จะทำงานหรือไม่"
end

puts read_setting
# เปิดไฟล์ config...
# ปิดไฟล์ config เสมอ ไม่ว่า return จะทำงานหรือไม่
# => production
```

แม้ `return "production"` จะสั่งให้ method จบการทำงานทันที แต่ Ruby ยังคง**แวะรัน**
`ensure` ก่อนที่จะส่งค่ากลับออกไปจริงๆ เสมอ — นี่คือสิ่งที่ทำให้ `ensure` เชื่อถือได้สำหรับ
งาน cleanup ต่างจากการเขียนโค้ด cleanup ไว้บรรทัดสุดท้ายของ method ตรงๆ (ซึ่งจะไม่ถูกรัน
เลยถ้ามี `return` แทรกก่อนหน้า)

### `ensure` ทำงานแม้ Exception จะไม่ถูก `rescue` เลย

```ruby
# frozen_string_literal: true

def risky_step
  puts "เริ่มทำงาน..."
  raise "เกิดข้อผิดพลาดร้ายแรง"
ensure
  puts "cleanup ทำงานแน่นอน แม้ error จะไม่ถูก rescue ใน method นี้เลยก็ตาม"
end

begin
  risky_step
rescue => e
  puts "จับได้ที่ชั้นบนสุด: #{e.message}"
end

# เริ่มทำงาน...
# cleanup ทำงานแน่นอน แม้ error จะไม่ถูก rescue ใน method นี้เลยก็ตาม
# จับได้ที่ชั้นบนสุด: เกิดข้อผิดพลาดร้ายแรง
```

สังเกตลำดับผลลัพธ์: `ensure` ใน `risky_step` ทำงาน**ก่อน**ที่ Exception จะเดินทางขึ้นไปถึง
`rescue` ของผู้เรียก — นี่คือกลไกที่ทำให้ `ensure` ใช้ทำความสะอาด resource ได้อย่างปลอดภัย
โดยไม่ต้องสนใจเลยว่า error จะถูกจัดการที่ไหน หรือจะถูกจัดการหรือไม่ก็ตาม

### ตัวอย่างจริง: ปิดไฟล์เสมอไม่ว่าจะเกิดอะไรขึ้น

```ruby
# frozen_string_literal: true

def process_file(path)
  file = File.open(path, "r")
  content = file.read
  raise "ไฟล์ว่างเปล่า ผิดปกติ" if content.empty?

  content.upcase
ensure
  file&.close  # &. ป้องกันกรณี File.open เอง error จนไม่มีตัวแปร file (จะเรียนละเอียด Part 012)
  puts "ปิดไฟล์เรียบร้อย (ไม่ว่าจะสำเร็จหรือ error)"
end
```

`file&.close` ใช้ safe navigation operator เผื่อกรณีที่ error เกิดขึ้น**ก่อน**บรรทัด
`File.open` จะสำเร็จเสียอีก (เช่น ไฟล์ไม่มีอยู่จริง) ซึ่งตอนนั้น `file` ยังเป็น `nil` อยู่
ถ้าเขียน `file.close` ตรงๆ จะเกิด `NoMethodError` ซ้อนขึ้นมาอีกชั้นหนึ่งภายใน `ensure` เอง

### อันตราย! `return`/`break` ภายใน `ensure` จะกลืน Exception ทั้งหมด

นี่คือกับดักที่อันตรายที่สุดจุดหนึ่งของ `ensure` — ถ้าเขียน `return` (หรือ `break`,
`throw`) ไว้**ภายใน** `ensure` เอง ค่านั้นจะ**เขียนทับ**ทั้งค่าที่ `begin` จะ return และ
เขียนทับ**แม้แต่ Exception ที่กำลังเดินทางออกไป** ทำให้ error หายไปเงียบๆ โดยไม่มีใครรู้เลย:

```ruby
# frozen_string_literal: true

# อันตราย! ห้ามทำแบบนี้ในโค้ดจริง
def dangerous_withdraw(balance, amount)
  raise "ยอดเงินไม่พอ" if amount > balance

  balance - amount
ensure
  return balance  # <<< return ใน ensure กลืนทั้งค่าปกติและ Exception ทิ้งหมด!
end

result = dangerous_withdraw(100, 500)
puts result
# => 100   (ไม่มี Exception ใดๆ ถูกโยนออกมาให้เห็นเลย ทั้งที่ raise ทำงานจริงข้างใน!)
```

**สิ่งที่เกิดขึ้นจริงเบื้องหลัง:** `raise "ยอดเงินไม่พอ"` ทำงานจริง และ Exception กำลัง
เดินทางออกจาก method — แต่ก่อนที่มันจะออกไปถึงผู้เรียก Ruby ต้องรัน `ensure` ก่อนเสมอ และ
เมื่อ `ensure` เองมี `return` ของตัวเอง คำสั่ง `return` นั้นจะ**ยกเลิก Exception ที่กำลัง
เดินทางอยู่ทันที** แล้วใช้ค่าจาก `ensure` แทนโดยสมบูรณ์ ผู้เรียกจึงไม่มีทางรู้เลยว่าเกิด
ข้อผิดพลาดขึ้นจริง — โปรแกรมดำเนินต่อไปราวกับไม่มีอะไรเกิดขึ้น ทั้งที่ business logic
ผิดพลาดร้ายแรง (ถอนเงินเกินยอดสำเร็จแบบเงียบๆ)

> **กฎเหล็ก:** **ห้ามใช้ `return`, `break`, หรือ `throw` ภายใน `ensure` เด็ดขาด** — `ensure`
> ควรมีแค่โค้ด cleanup (ปิดไฟล์, ปิด connection, log ข้อความ) เท่านั้น ไม่ควรพยายาม
> ควบคุม control flow ของ method เลย RuboCop มี cop ชื่อ `Lint/EnsureReturn` ที่ตรวจจับ
> pattern อันตรายนี้โดยเฉพาะและจะ flag เป็น error ทันทีถ้าเจอ

---

## Step 109: Anti-pattern — "rescue ทุกอย่างแล้วกลืนหายไปเงียบๆ" อันตรายอย่างไร

Anti-pattern ที่พบบ่อยที่สุดในโค้ด Ruby ของมือใหม่ (และบางครั้งแม้แต่มืออาชีพที่รีบเร่ง)
คือการเขียน `rescue` แบบกว้างที่สุดเท่าที่จะทำได้ แล้วไม่ทำอะไรกับ Exception ที่จับได้เลย
นอกจากปล่อยให้โปรแกรม "ดูเหมือนทำงานต่อได้ปกติ"

```ruby
# frozen_string_literal: true

# อันตรายระดับที่ 1: rescue แล้วไม่ทำอะไรเลย (silent swallow)
def save_user_v1(user)
  user.save!
rescue
  # ว่างเปล่า -- ไม่ log, ไม่แจ้งใคร, ไม่ทำอะไรเลยแม้แต่น้อย
end

# อันตรายระดับที่ 2: rescue Exception กว้างเกินไป (ตามที่เตือนไว้ใน Step 103)
def save_user_v2(user)
  user.save!
rescue Exception
  puts "เกิดข้อผิดพลาด"  # ข้อความทั่วไปจนไม่มีประโยชน์ แถมยังจับ SystemExit/Interrupt ไปด้วย
end

# อันตรายระดับที่ 3: rescue แล้ว return ค่า default โดยไม่แยกแยะสาเหตุ
def fetch_price_v3(product_id)
  ExternalApi.get_price(product_id)
rescue StandardError
  0  # ราคาสินค้ากลายเป็น 0 อย่างเงียบๆ ไม่ว่าสาเหตุจะเป็น network error หรือ product_id ผิด
end
```

**ทำไมโค้ดแบบนี้ถึงอันตรายมาก แม้จะ "ทำให้โปรแกรมไม่ crash" ก็ตาม:**

1. **ซ่อนบั๊กจนหาไม่เจอ** — ถ้า `save_user_v1` error เพราะ typo ใน method name (ซึ่งจะเป็น
   `NoMethodError` ที่ควรถูกแก้โค้ดทันที) มันจะถูกกลืนหายไปเงียบๆ เหมือนกับ error ที่คาดหวัง
   ไว้แล้วทุกประการ — ทีมพัฒนาจะไม่มีทางรู้เลยว่ามีบั๊กอยู่ จนกว่าจะมีคนมาสังเกตเห็นว่า
   "ผู้ใช้บ่นว่าข้อมูลหาย" ซึ่งอาจจะสายเกินไปแล้ว
2. **ข้อมูลเสียหายแบบเงียบๆ** — `fetch_price_v3` คืนค่า `0` ทุกครั้งที่ error โดยไม่บอกใคร
   ถ้าโค้ดส่วนอื่นเอาราคานี้ไปคำนวณต่อ (เช่น คิดเงินลูกค้า) อาจเกิดความเสียหายทางธุรกิจจริง
   โดยไม่มีร่องรอยให้ตรวจสอบย้อนหลังเลยว่าทำไมราคาถึงผิด
3. **ปิดกั้นสัญญาณฉุกเฉินของระบบ** — `rescue Exception` (อันตรายระดับ 2) ปิดกั้นแม้แต่
   `SystemExit`/`Interrupt` ตามที่เรียนใน Step 103 ทำให้ควบคุมโปรแกรมจากภายนอกไม่ได้เลย
4. **Debug ยากขึ้นแบบทวีคูณ** — เมื่อไม่มี log และไม่มี backtrace เหลืออยู่เลย การหาสาเหตุ
   ต้นตอของปัญหาใน production กลายเป็นงานที่แทบเป็นไปไม่ได้ เพราะไม่มีข้อมูลอะไรให้ตามรอย

### วิธีแก้ที่ถูกต้อง

```ruby
# frozen_string_literal: true

# ดีขึ้นระดับที่ 1: rescue เฉพาะเจาะจง + log เสมอ
def save_user_fixed(user)
  user.save!
rescue ActiveRecord::RecordInvalid => e
  # rescue เฉพาะ error ที่ "คาดว่าจะเกิดได้" จาก .save! เท่านั้น (validation ไม่ผ่าน)
  # ส่วน error อื่นที่ไม่คาดคิด (เช่น typo, bug) จะยังคง raise ต่อไปตามปกติ ไม่ถูกกลืน
  Rails.logger.error("บันทึกผู้ใช้ไม่สำเร็จ: #{e.message}")
  false
end

# ดีขึ้นระดับที่ 2: แยกแยะสาเหตุ แล้วจัดการต่างกันตามความเหมาะสม
def fetch_price_fixed(product_id)
  ExternalApi.get_price(product_id)
rescue ExternalApi::NotFoundError => e
  puts "[WARN] ไม่พบสินค้า #{product_id}: #{e.message}"
  nil  # nil สื่อความหมายชัดเจนกว่า 0 ว่า "ไม่มีข้อมูล" ไม่ใช่ "ราคาเป็นศูนย์จริงๆ"
rescue ExternalApi::TimeoutError => e
  puts "[ERROR] API หมดเวลา สำหรับสินค้า #{product_id}: #{e.message}"
  raise  # timeout เป็นปัญหาระบบที่ควรถูก monitor -- re-raise ให้ error tracking เห็น
end
```

**หลักปฏิบัติที่ควรจำ:**

- **`rescue` เฉพาะ Exception ที่รู้แน่ชัดว่า "จะเกิดขึ้นได้และมีวิธีรับมือ"** เท่านั้น อย่า
  `rescue` แบบกว้างเพื่อ "เผื่อไว้" — ปล่อยให้ error ที่ไม่คาดคิดพังออกมาให้เห็นชัดเจน
  ดีกว่าซ่อนมันไว้
- **ทุกจุดที่ `rescue` ต้องทำอย่างน้อยหนึ่งใน 2 อย่างนี้เสมอ:** (ก) log/บันทึกข้อมูลไว้
  เพื่อ debug ภายหลัง หรือ (ข) `raise`/`raise` ใหม่ต่อให้ชั้นบนจัดการ — **ห้ามมี `rescue`
  ที่ทั้งไม่ log และไม่ raise ต่อเด็ดขาด**
- คืนค่า `nil` หรือ error object ที่สื่อความหมายชัดเจนว่า "ล้มเหลว" แทนค่า default ที่ดู
  เหมือนค่าจริง (เช่น `0`, `""`, `[]`) เพราะค่าเหล่านี้กลืนหายไปในระบบต่อโดยไม่มีใครสังเกต
  ได้ว่าจริงๆ แล้วมันคือสัญญาณของความล้มเหลว

> **preview Rails/production:** ใน Rails ระดับ production มักมี global exception handler
> (`rescue_from` ใน `ApplicationController`, จะเรียนใน Phase 3) ที่ log ทุก error ไปยัง
> ระบบ error tracking เช่น Sentry ก่อนแสดงหน้า error ที่เหมาะสมให้ผู้ใช้เห็น — หลักการ
> เบื้องหลังคือสิ่งเดียวกับที่เรียนใน Step นี้ทุกประการ: **ไม่เคยมี error ใดๆ ที่ถูกกลืนหาย
> ไปเงียบๆ โดยไม่มีร่องรอยเหลือไว้เลย**

---

## Step 110: แบบฝึกหัดรวบยอด — Payment Processing Simulator

### โจทย์

เขียนไฟล์ `payment_simulator.rb` จำลองระบบประมวลผลการชำระเงินที่ต้องติดต่อ "บริการภายนอก"
(simulate ด้วย method ที่ error แบบสุ่ม) โดยต้องมีครบทุกองค์ประกอบที่เรียนมาใน Part นี้:

1. **Custom Exception อย่างน้อย 3 ชนิด** สืบทอดจาก base error class ร่วมกัน:
   - `InsufficientFundsError` — ยอดเงินในบัญชีไม่พอ (ไม่ควร retry เพราะ retry ไปก็ไม่หาย)
   - `PaymentGatewayTimeoutError` — บริการชำระเงินภายนอกหมดเวลาตอบสนอง (ควร retry ได้)
   - `InvalidCardError` — ข้อมูลบัตรไม่ถูกต้อง (ไม่ควร retry)
2. **`retry` พร้อม exponential backoff** เฉพาะกรณี `PaymentGatewayTimeoutError` เท่านั้น
   จำกัดไม่เกิน 3 ครั้ง
3. **`ensure`** ที่รับประกันว่า "การเชื่อมต่อกับ payment gateway" จะถูกปิดเสมอ ไม่ว่าผลลัพธ์
   จะเป็นอย่างไร
4. **ไม่มีจุดไหนกลืน Exception ไว้เงียบๆ** — ทุก error ต้อง log และ/หรือ re-raise ตามความ
   เหมาะสม (ตามหลักการ Step 109)

### เฉลยฉบับเต็ม

```ruby
# frozen_string_literal: true

# payment_simulator.rb — Payment Processing Simulator
# แบบฝึกหัดรวบยอด Part 011: Exception Handling

# ============================================================
# Custom Exception Hierarchy
# ============================================================
module PaymentErrors
  # Base class กลางของทุก error ที่เกี่ยวกับการชำระเงินในระบบนี้
  class Error < StandardError; end

  class InsufficientFundsError < Error
    attr_reader :balance, :amount_requested

    def initialize(balance:, amount_requested:)
      @balance = balance
      @amount_requested = amount_requested
      super("ยอดเงินไม่พอ: มี #{balance} บาท แต่ต้องชำระ #{amount_requested} บาท")
    end
  end

  class PaymentGatewayTimeoutError < Error
    def initialize(gateway_name)
      super("บริการชำระเงิน \"#{gateway_name}\" ไม่ตอบสนองภายในเวลาที่กำหนด")
    end
  end

  class InvalidCardError < Error
    attr_reader :card_number_masked

    def initialize(card_number)
      @card_number_masked = "**** **** **** #{card_number[-4..]}"
      super("หมายเลขบัตร #{@card_number_masked} ไม่ถูกต้องหรือหมดอายุ")
    end
  end
end

# ============================================================
# PaymentGateway — จำลองบริการชำระเงินภายนอกที่ไม่แน่นอน
# ============================================================
class PaymentGateway
  GATEWAY_NAME = "MockPay"

  def initialize(fail_count: 0)
    # fail_count: จำนวนครั้งแรกที่จะจำลองว่า timeout ก่อนจะสำเร็จ (ใช้ทดสอบ retry)
    @fail_count = fail_count
    @attempt = 0
    @connected = false
  end

  def connect
    @connected = true
    puts "  [gateway] เชื่อมต่อกับ #{GATEWAY_NAME} สำเร็จ"
  end

  def disconnect
    return unless @connected

    @connected = false
    puts "  [gateway] ตัดการเชื่อมต่อกับ #{GATEWAY_NAME} เรียบร้อย"
  end

  def charge(card_number:, amount:)
    raise "ต้อง connect ก่อนเรียก charge" unless @connected

    @attempt += 1

    if card_number.length != 16
      raise PaymentErrors::InvalidCardError, card_number
    end

    if @attempt <= @fail_count
      raise PaymentErrors::PaymentGatewayTimeoutError, GATEWAY_NAME
    end

    { transaction_id: "TXN-#{rand(100_000..999_999)}", amount: amount, status: "success" }
  end
end

# ============================================================
# BankAccount — เก็บยอดเงินและตรวจสอบยอดคงเหลือ
# ============================================================
class BankAccount
  attr_reader :owner, :balance

  def initialize(owner, balance)
    @owner = owner
    @balance = balance
  end

  def ensure_sufficient_funds!(amount)
    return if amount <= balance

    raise PaymentErrors::InsufficientFundsError.new(balance: balance, amount_requested: amount)
  end

  def deduct(amount)
    @balance -= amount
  end
end

# ============================================================
# PaymentProcessor — ประสาน BankAccount + PaymentGateway เข้าด้วยกัน
# พร้อม retry with backoff และ ensure cleanup
# ============================================================
class PaymentProcessor
  MAX_RETRIES = 3

  def initialize(account, gateway)
    @account = account
    @gateway = gateway
  end

  def process(card_number:, amount:)
    @account.ensure_sufficient_funds!(amount)

    attempts = 0
    @gateway.connect

    begin
      attempts += 1
      puts "  [processor] ครั้งที่ #{attempts}: กำลังเรียกเก็บเงิน #{amount} บาท..."
      result = @gateway.charge(card_number: card_number, amount: amount)
      @account.deduct(amount)
      puts "  [processor] สำเร็จ! Transaction ID: #{result[:transaction_id]}"
      result
    rescue PaymentErrors::PaymentGatewayTimeoutError => e
      if attempts < MAX_RETRIES
        wait_seconds = 2**attempts
        puts "  [processor] #{e.message} -> รอ #{wait_seconds} วินาทีแล้วลองใหม่ " \
             "(ครั้งที่ #{attempts}/#{MAX_RETRIES})"
        sleep(0.01) # ย่อเวลาไว้เพื่อสาธิต ในระบบจริงจะใช้ sleep(wait_seconds)
        retry
      else
        puts "  [processor] ลองครบ #{MAX_RETRIES} ครั้งแล้วยังไม่สำเร็จ ยกเลิกการทำรายการ"
        raise
      end
    rescue PaymentErrors::InvalidCardError => e
      # ข้อมูลบัตรผิด -- ไม่มีประโยชน์ที่จะ retry เลย ต้อง raise ทันที ให้ผู้ใช้แก้ข้อมูล
      puts "  [processor] ข้อมูลบัตรไม่ถูกต้อง: #{e.message}"
      raise
    ensure
      # รับประกันว่าปิดการเชื่อมต่อเสมอ ไม่ว่าผลลัพธ์จะสำเร็จ, retry ครบ, หรือ error ชนิดใดก็ตาม
      @gateway.disconnect
    end
  end
end

# ============================================================
# main — ทดสอบ scenario ต่างๆ
# ============================================================
def run_scenario(title)
  puts "\n#{'=' * 60}"
  puts "Scenario: #{title}"
  puts "=" * 60
  yield
rescue PaymentErrors::Error => e
  puts ">> ทำรายการไม่สำเร็จ (#{e.class}): #{e.message}"
end

# Scenario 1: ยอดเงินไม่พอ -- ควร raise ทันทีโดยไม่แตะ gateway เลย
run_scenario("ยอดเงินไม่พอ") do
  account = BankAccount.new("สมชาย", 100)
  gateway = PaymentGateway.new
  processor = PaymentProcessor.new(account, gateway)
  processor.process(card_number: "1234567812345678", amount: 5000)
end

# Scenario 2: บัตรไม่ถูกต้อง (เลขบัตรสั้นเกินไป)
run_scenario("บัตรไม่ถูกต้อง") do
  account = BankAccount.new("สมหญิง", 1000)
  gateway = PaymentGateway.new
  processor = PaymentProcessor.new(account, gateway)
  processor.process(card_number: "1234", amount: 500)
end

# Scenario 3: Gateway timeout 2 ครั้งแรก แล้วสำเร็จในครั้งที่ 3 -- ทดสอบ retry
run_scenario("Gateway timeout แล้ว retry จนสำเร็จ") do
  account = BankAccount.new("มานี", 1000)
  gateway = PaymentGateway.new(fail_count: 2)
  processor = PaymentProcessor.new(account, gateway)
  result = processor.process(card_number: "1234567812345678", amount: 300)
  puts "ยอดคงเหลือหลังหักเงิน: #{account.balance} บาท"
  result
end

# Scenario 4: Gateway timeout เกิน MAX_RETRIES -- ควร raise ออกมาในที่สุด
run_scenario("Gateway timeout เกินจำนวนครั้งที่ยอมให้ลอง") do
  account = BankAccount.new("มานะ", 1000)
  gateway = PaymentGateway.new(fail_count: 99) # fail ตลอด ไม่มีวันถึง success
  processor = PaymentProcessor.new(account, gateway)
  processor.process(card_number: "1234567812345678", amount: 300)
end
```

### ทดสอบรัน

```bash
ruby payment_simulator.rb
```

ผลลัพธ์ที่คาดหวัง (ย่อบางส่วน):

```
============================================================
Scenario: ยอดเงินไม่พอ
============================================================
>> ทำรายการไม่สำเร็จ (PaymentErrors::InsufficientFundsError): ยอดเงินไม่พอ: มี 100 บาท แต่ต้องชำระ 5000 บาท

============================================================
Scenario: บัตรไม่ถูกต้อง
============================================================
  [gateway] เชื่อมต่อกับ MockPay สำเร็จ
  [processor] ครั้งที่ 1: กำลังเรียกเก็บเงิน 500 บาท...
  [processor] ข้อมูลบัตรไม่ถูกต้อง: หมายเลขบัตร **** **** **** 1234 ไม่ถูกต้องหรือหมดอายุ
  [gateway] ตัดการเชื่อมต่อกับ MockPay เรียบร้อย
>> ทำรายการไม่สำเร็จ (PaymentErrors::InvalidCardError): หมายเลขบัตร **** **** **** 1234 ไม่ถูกต้องหรือหมดอายุ

============================================================
Scenario: Gateway timeout แล้ว retry จนสำเร็จ
============================================================
  [gateway] เชื่อมต่อกับ MockPay สำเร็จ
  [processor] ครั้งที่ 1: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] บริการชำระเงิน "MockPay" ไม่ตอบสนองภายในเวลาที่กำหนด -> รอ 2 วินาทีแล้วลองใหม่ (ครั้งที่ 1/3)
  [processor] ครั้งที่ 2: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] บริการชำระเงิน "MockPay" ไม่ตอบสนองภายในเวลาที่กำหนด -> รอ 4 วินาทีแล้วลองใหม่ (ครั้งที่ 2/3)
  [processor] ครั้งที่ 3: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] สำเร็จ! Transaction ID: TXN-483920
  [gateway] ตัดการเชื่อมต่อกับ MockPay เรียบร้อย
ยอดคงเหลือหลังหักเงิน: 700 บาท

============================================================
Scenario: Gateway timeout เกินจำนวนครั้งที่ยอมให้ลอง
============================================================
  [gateway] เชื่อมต่อกับ MockPay สำเร็จ
  [processor] ครั้งที่ 1: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] บริการชำระเงิน "MockPay" ไม่ตอบสนองภายในเวลาที่กำหนด -> รอ 2 วินาทีแล้วลองใหม่ (ครั้งที่ 1/3)
  [processor] ครั้งที่ 2: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] บริการชำระเงิน "MockPay" ไม่ตอบสนองภายในเวลาที่กำหนด -> รอ 4 วินาทีแล้วลองใหม่ (ครั้งที่ 2/3)
  [processor] ครั้งที่ 3: กำลังเรียกเก็บเงิน 300 บาท...
  [processor] ลองครบ 3 ครั้งแล้วยังไม่สำเร็จ ยกเลิกการทำรายการ
  [gateway] ตัดการเชื่อมต่อกับ MockPay เรียบร้อย
>> ทำรายการไม่สำเร็จ (PaymentErrors::PaymentGatewayTimeoutError): บริการชำระเงิน "MockPay" ไม่ตอบสนองภายในเวลาที่กำหนด
```

โค้ดนี้ทดสอบรันจริงแล้วบน Ruby 3.3.6 ครบทั้ง 4 scenario ทำงานถูกต้องตามที่ออกแบบไว้ทุกจุด

### จุดที่ควรสังเกตในเฉลย

- **`ensure_sufficient_funds!` ถูกเรียกก่อน `@gateway.connect`** — ตรวจสอบเงื่อนไขทางธุรกิจ
  ที่ไม่เกี่ยวกับ network (ยอดเงินพอหรือไม่) ให้เสร็จก่อน จะได้ไม่ต้องเสียเวลาเชื่อมต่อ
  บริการภายนอกโดยเปล่าประโยชน์เมื่อรู้อยู่แล้วว่าจะทำรายการไม่ได้แน่ๆ — นี่คือเหตุผลที่
  Scenario 1 ไม่มี log `[gateway] เชื่อมต่อ...` เลย
- **`ensure` ใน `PaymentProcessor#process` ครอบเฉพาะส่วนที่แตะ gateway เท่านั้น** ไม่ครอบ
  `ensure_sufficient_funds!` ที่อยู่นอก `begin` — ออกแบบให้ `ensure` รับผิดชอบ cleanup
  เฉพาะ resource ที่มันเปิดขึ้นมาเองจริงๆ (`@gateway.connect`) เท่านั้น ตรงตามหลักการจาก
  Step 108
- **`rescue PaymentErrors::PaymentGatewayTimeoutError` กับ `rescue PaymentErrors::InvalidCardError`
  แยกกันคนละบรรทัด** เพราะพฤติกรรมต่างกันโดยสิ้นเชิง (retry ได้ vs raise ทันที) — สอดคล้อง
  กับหลักการ Step 104 ว่าให้แยก `rescue` เมื่อ logic ต่างกัน และ Step 105 ว่าให้รวมกันเฉพาะ
  เมื่อ logic เหมือนกันเป๊ะๆ
- **ไม่มี `rescue` จุดไหนเลยที่กลืน error แบบเงียบๆ** — ทุก `rescue` ใน `PaymentProcessor`
  ทั้ง log ด้วย `puts` และจบด้วย `raise`/`retry` เสมอ ส่วน `run_scenario` ที่ชั้นบนสุดคือ
  จุดเดียวที่ "จบ" การจัดการ error จริงๆ (แสดงผลให้ผู้ใช้เห็นและไม่ raise ต่อ) ซึ่งเหมาะสม
  เพราะเป็นจุดบนสุดของโปรแกรมแล้ว ตรงตามหลักการ Step 109
- **`run_scenario` rescue เฉพาะ `PaymentErrors::Error`** (base class) ไม่ใช่
  `StandardError` เปล่าๆ — เพราะต้องการจับเฉพาะ error ที่เกี่ยวกับการชำระเงินที่ "รู้จัก"
  เท่านั้น ถ้าเกิด error อื่นที่ไม่คาดคิด (เช่น `NoMethodError` จาก bug ในโค้ด) มันจะยังคง
  หลุดออกมาทำให้โปรแกรมหยุดทำงานและแสดง backtrace ให้เห็นชัดเจน แทนที่จะถูกกลืนไปกับ error
  ทางธุรกิจปกติ

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม `DailyLimitExceededError`** — เพิ่มเงื่อนไขว่าบัญชีหนึ่งชำระเงินได้ไม่เกิน
   10,000 บาทต่อวัน (เก็บยอดสะสมไว้ใน `BankAccount`) ถ้าเกิน limit ให้ raise error ใหม่นี้
   โดยไม่ต้อง retry (เหมือน `InsufficientFundsError`)
2. **เขียน retry logic ที่นับจำนวนครั้งรวมข้าม method** — ปัจจุบัน `attempts` เริ่มนับใหม่
   ทุกครั้งที่เรียก `process` ลองแก้ให้ `PaymentGateway` เก็บสถิติจำนวนครั้งที่ timeout สะสม
   ทั้งหมดในอายุการใช้งานของ object (ไม่ reset ทุกครั้งที่เรียก `charge`) แล้วปฏิเสธไม่ให้
   ใช้งาน gateway ตัวนั้นต่อถ้า timeout สะสมเกิน 10 ครั้ง (raise error ใหม่ทันทีโดยไม่ต้อง
   ลองเชื่อมต่อเลย)
3. **เพิ่ม logging เป็นไฟล์จริง** — แทนที่ `puts` ทั้งหมดด้วยการเขียน log ลงไฟล์
   `payment.log` ผ่าน class `Logger` มาตรฐานของ Ruby (`require "logger"`) พร้อมแยกระดับ
   `logger.info`/`logger.warn`/`logger.error` ให้เหมาะสมกับแต่ละสถานการณ์ (คำใบ้: เรื่อง
   การเขียนไฟล์แบบละเอียดจะเรียนเต็มใน Part 012 ถัดไป ลองค้นเอกสาร `Logger` มาใช้ล่วงหน้าดู
   ก็ได้)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจกายวิภาคเต็มรูปแบบของ `begin`/`rescue`/`else`/`ensure` และรู้ว่าแต่ละส่วนทำงาน
  เมื่อไหร่ รวมถึงรูปแบบ implicit begin ที่เขียน `rescue`/`ensure` ต่อท้าย `def` ได้โดยตรง
- ใช้ `raise` ได้ทั้ง 4 รูปแบบ (bare message, class+message, `.new(message)`, class เปล่า)
  และเข้าใจการ re-raise ด้วย bare `raise` ภายใน `rescue`
- เข้าใจ Exception Class Hierarchy ทั้งหมด และรู้เหตุผลที่ต้อง `rescue StandardError`
  (หรือปล่อยให้ Ruby เลือกให้อัตโนมัติ) ไม่ใช่ `rescue Exception` ตรงๆ
- เขียน `rescue` หลายบรรทัดเรียงจากเฉพาะเจาะจงไปกว้าง และรวม Exception หลายชนิดในบรรทัด
  เดียวได้เมื่อพฤติกรรมการจัดการเหมือนกัน
- สร้าง **Custom Exception Class** ของตัวเองได้ ทั้งแบบพื้นฐานและแบบมี attribute/`initialize`
  เฉพาะตัว พร้อมจัดกลุ่มด้วย base error class ร่วมกันผ่าน module namespace
- ใช้ `retry` เขียนรูปแบบ **Retry with Exponential Backoff** สำหรับ error ชั่วคราวได้อย่าง
  ปลอดภัย (มีเพดานจำนวนครั้งเสมอ)
- เข้าใจการรับประกันของ `ensure` อย่างลึกซึ้ง รวมถึงอันตรายของการใช้ `return`/`break`
  ภายใน `ensure` ที่จะกลืน Exception ทั้งหมดไปอย่างเงียบๆ
- รู้จัก anti-pattern "rescue แล้วกลืนหายเงียบๆ" ในทุกระดับความอันตราย และหลักปฏิบัติที่
  ถูกต้อง: ทุก `rescue` ต้อง log และ/หรือ raise ต่อเสมอ ไม่มีข้อยกเว้น
- สร้างโปรเจกต์ **Payment Processing Simulator** ที่รวม custom exception hierarchy, retry
  with backoff, และ ensure cleanup เข้าด้วยกันเป็นระบบที่ทนทานต่อความผิดพลาดจริง

**ต่อไป (Part 012):** เราจะเรียนรู้ **File I/O** — การอ่านและเขียนไฟล์ในรูปแบบต่างๆ, การ
จัดการไฟล์ CSV, JSON, และ YAML ซึ่งเป็นทักษะพื้นฐานที่ต้องใช้ควบคู่กับ Exception Handling
ที่เพิ่งเรียนจบไปเสมอ (เพราะการอ่าน/เขียนไฟล์คือหนึ่งในแหล่งที่มาของ error ที่พบบ่อยที่สุด
ในโปรแกรมจริง เช่น ไฟล์ไม่มีอยู่ หรือสิทธิ์การเข้าถึงไม่พอ)
