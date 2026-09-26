# Part 019: RSpec เบื้องต้น — describe/context/it, matcher, let, before/after

> **Step ครอบคลุมใน Part นี้:** Step 181–190
> **ระดับ:** กลาง (ต้องผ่าน Part 018 เรื่อง Minitest มาก่อน โดยเฉพาะแนวคิดพื้นฐานว่า
> "ทำไมต้องเขียนเทสต์", unit test, assertion และ test double เบื้องต้น — Part นี้จะไม่พูดซ้ำ
> เรื่องเหล่านั้น แต่จะโฟกัสที่ตัว RSpec framework โดยตรง)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, gem `rspec` เวอร์ชัน 3.13.x (RSpec 3 ล่าสุด)

## สารบัญของ Part นี้

- Step 181: RSpec คืออะไร ต่างจาก Minitest อย่างไร และการติดตั้ง
- Step 182: โครงสร้างไฟล์ spec, ชื่อไฟล์ตามธรรมเนียม `*_spec.rb`, `describe`/`it` เบื้องต้น
- Step 183: `describe` vs `context` — ความหมายเชิงความหมาย (semantic) ที่ต่างกัน
- Step 184: `expect(...).to` matcher syntax เทียบกับ `should` syntax แบบเก่า
- Step 185: Matcher มาตรฐานที่ใช้บ่อย: `include`, `match`, `be_within`, `raise_error`, `change`
- Step 186: `let`/`let!` — lazy-evaluated memoized test data เทียบกับ instance variable ใน `before`
- Step 187: `before`/`after` hooks — `:each` กับ `:context`/`:all`
- Step 188: `subject` และ `described_class`
- Step 189: `shared_examples`/`it_behaves_like` — เขียน spec แบบ DRY ข้ามหลาย class
- Step 190: RSpec doubles/mocks: `double`, `instance_double`, `allow`, `expect(...).to receive`

---

## Step 181: RSpec คืออะไร ต่างจาก Minitest อย่างไร และการติดตั้ง

### RSpec คืออะไร

**RSpec** เป็น testing framework สำหรับ Ruby ที่ได้รับความนิยมสูงมากในวงการ Ruby/Rails
เชิงพาณิชย์ (production) แม้ Rails เองจะติดตั้ง **Minitest** มาเป็นค่าเริ่มต้น (ตามที่เรียนใน
Part 018) แต่ทีมพัฒนาส่วนใหญ่ในอุตสาหกรรมกลับเลือกใช้ RSpec แทน เพราะปรัชญาการออกแบบที่
ต่างกันอย่างชัดเจน

RSpec ถูกออกแบบตามแนวคิด **BDD (Behavior-Driven Development)** ซึ่งเน้นการเขียนเทสต์ให้
อ่านออกเสียงได้เหมือน "ข้อกำหนดเชิงพฤติกรรม" (specification) ของระบบ มากกว่าจะเป็นแค่โค้ด
ตรวจสอบ assertion ธรรมดา ชื่อไฟล์ของ RSpec จึงเรียกว่า **spec** (ย่อจาก specification) ไม่ใช่
"test" เหมือน Minitest

### เปรียบเทียบ syntax ตัวต่อตัว

Minitest (สิ่งที่เรียนไปใน Part 018):

```ruby
# frozen_string_literal: true

require "minitest/autorun"

class CalculatorTest < Minitest::Test
  def test_add_returns_sum_of_two_numbers
    calculator = Calculator.new
    assert_equal 5, calculator.add(2, 3)
  end
end
```

RSpec (สิ่งที่จะเรียนใน Part นี้):

```ruby
# frozen_string_literal: true

RSpec.describe Calculator do
  it "บวกเลขสองจำนวนแล้วได้ผลรวมที่ถูกต้อง" do
    calculator = Calculator.new
    expect(calculator.add(2, 3)).to eq(5)
  end
end
```

**ความต่างเชิงปรัชญาที่สำคัญ (ไม่ใช่แค่ syntax):**

| ประเด็น | Minitest | RSpec |
|---|---|---|
| สไตล์ | xUnit (class + method ขึ้นต้นด้วย `test_`) | BDD DSL (`describe`/`context`/`it`) |
| การเขียนคำอธิบาย | ชื่อ method (`test_add_returns_sum`) | ประโยคภาษาธรรมชาติ (`it "บวกเลขสองจำนวน..."`) |
| Assertion | `assert_equal expected, actual` | `expect(actual).to eq(expected)` |
| การจัดกลุ่มเทสต์ | class เดียว, ไม่มีการซ้อนกลุ่มย่อยในตัว | `describe`/`context` ซ้อนกันได้ไม่จำกัดชั้น |
| Mock/stub | ต้องพึ่ง `Minitest::Mock` หรือ gem เสริม | มี `double`/`instance_double`/`allow`/`receive` ในตัว (rspec-mocks) |
| ปรัชญา | เรียบง่าย ใกล้เคียง plain Ruby ที่สุด | อ่านคล้ายเอกสารข้อกำหนด (living documentation) |
| ต้นทุน | โหลดเร็ว, debug ง่าย (ไม่มี DSL ซับซ้อน) | มี DSL/metaprogramming เยอะกว่า โหลดช้ากว่าเล็กน้อย |

ทั้งสองแบบทดสอบ "สิ่งเดียวกัน" ได้ครบถ้วนเหมือนกันทุกประการ — ไม่มีฝั่งไหน "ทดสอบได้มากกว่า"
อีกฝั่ง ความต่างอยู่ที่ **ความอ่านง่ายของผลลัพธ์และการจัดโครงสร้าง** RSpec เหมาะกับทีมที่
ต้องการให้ผลการรันเทสต์อ่านเป็นประโยคสมบูรณ์ (ตัวอย่างด้านล่าง) ส่วน Minitest เหมาะกับคนที่
อยากให้เทสต์เป็น "แค่ Ruby ธรรมดาๆ" ไม่มี DSL แปลกใหม่ให้ต้องจำเพิ่ม

ลองเทียบผลลัพธ์การรันแบบ documentation format จริงๆ:

```
Calculator
  บวกเลขสองจำนวนแล้วได้ผลรวมที่ถูกต้อง

Finished in 0.00142 seconds (files took 0.04512 seconds to load)
1 example, 0 failures
```

สังเกตว่าผลลัพธ์อ่านได้เป็นประโยค "Calculator บวกเลขสองจำนวนแล้วได้ผลรวมที่ถูกต้อง" ทันที
นี่คือจุดขายหลักของ RSpec ที่ทำให้หลายทีมเลือกใช้ แม้จะต้องเรียน DSL เพิ่มเติมก็ตาม

> **ข้อควรระวัง:** อย่ามองว่า RSpec "ดีกว่า" Minitest เสมอไป ทั้งสองเป็นเครื่องมือที่ใช้กัน
> จริงจังในอุตสาหกรรม โปรเจกต์ Rails เองก็มีทั้งสองค่ายอยู่จำนวนมาก สิ่งสำคัญคือเข้าใจทั้งคู่
> เพราะเมื่อเข้าทำงานจริงจะต้องเจอโปรเจกต์ที่ใช้ทั้งสองแบบแน่นอน

### ติดตั้ง RSpec

วิธีที่ 1 — ติดตั้งแบบ global ด้วย `gem install` (เหมาะกับการทดลองเร็วๆ):

```bash
gem install rspec

rspec --version
# => RSpec 3.13
#      - rspec-core 3.13.6
#      - rspec-expectations 3.13.5
#      - rspec-mocks 3.13.8
#      - rspec-support 3.13.7
```

วิธีที่ 2 — ติดตั้งผ่าน Bundler (แนะนำสำหรับโปรเจกต์จริง เพราะล็อกเวอร์ชันได้แน่นอน):

```bash
mkdir -p ~/ruby-course-workspace/part-019
cd ~/ruby-course-workspace/part-019

bundle init
```

แก้ไข `Gemfile` ที่ถูกสร้างขึ้นมา:

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

gem "rspec", "~> 3.13"
```

```bash
bundle install

bundle exec rspec --version
# => RSpec 3.13
#      - rspec-core 3.13.6
#      - rspec-expectations 3.13.5
#      - rspec-mocks 3.13.8
#      - rspec-support 3.13.7
```

> **แนวคิดสำคัญ:** ในโปรเจกต์จริงควรเรียกผ่าน `bundle exec rspec` เสมอ (เหมือนที่เรียนเรื่อง
> `bundle exec` ใน Part 001 และ Part 017) เพื่อให้แน่ใจว่าใช้เวอร์ชัน RSpec ตรงกับที่ล็อกไว้ใน
> `Gemfile.lock` ในบทเรียนนี้จะย่อเขียนเป็น `rspec` เฉยๆ เพื่อความกระชับ แต่ในการใช้งานจริง
> ให้เติม `bundle exec` นำหน้าเสมอ

### `rspec --init` — สร้างโครงไฟล์เริ่มต้น

```bash
rspec --init
```

คำสั่งนี้สร้างไฟล์ 2 ไฟล์:

```
create   .rspec
create   spec/spec_helper.rb
```

ไฟล์ `.rspec` (ตั้งค่า command line options ที่จะใช้ทุกครั้งที่รัน `rspec` โดยไม่ต้องพิมพ์ซ้ำ):

```
--require spec_helper
```

ไฟล์ `spec/spec_helper.rb` (ตัดมาเฉพาะส่วนสำคัญ ของจริงจะมี comment อธิบายยาวกว่านี้มาก):

```ruby
# frozen_string_literal: true

RSpec.configure do |config|
  # เมื่อ mock ยิง exception ระหว่างรัน RSpec เอง ให้แสดง error backtrace แบบเต็ม
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  # ตรวจสอบว่า object ที่ถูก partial double (stub บาง method ของ object จริง)
  # มี method นั้นอยู่จริงหรือไม่ — ช่วยจับบั๊กจากการพิมพ์ชื่อ method ผิด
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  # ให้ metadata ของ shared context (จะเรียนใน Step 189) ใช้ได้ทั้งกับ example และ group
  config.shared_context_metadata_behavior = :apply_to_host_groups

  # จำกัดให้ RSpec โฟกัสรันเฉพาะ example ที่ติด focus: true เท่านั้น เวลามีการ tag ไว้
  config.filter_run_when_matching :focus

  # จำ 10 เทสต์ล่าสุดที่ fail ไว้ รันซ้ำง่ายๆ ด้วย --only-failures
  config.example_status_persistence_file_path = "spec/examples.txt"

  # สุ่มลำดับการรันเทสต์ทุกครั้ง เพื่อจับบั๊กที่เทสต์หนึ่งพึ่งพาผลข้างเคียงจากอีกเทสต์
  config.order = :random
  Kernel.srand config.seed
end
```

**บรรทัดที่สำคัญที่สุดสำหรับผู้เริ่มต้น:** `config.order = :random` คือสิ่งที่บังคับให้เทสต์
ทุกตัว **ต้องเป็นอิสระจากกัน** ห้าม test A ตัวหนึ่งพึ่งพาผลลัพธ์จาก test B ที่รันก่อนหน้า
เพราะลำดับการรันจะสุ่มใหม่ทุกครั้ง — นี่คือแนวปฏิบัติที่ดีที่ RSpec บังคับมาให้ตั้งแต่แรกเริ่ม

---

## Step 182: โครงสร้างไฟล์ spec, ชื่อไฟล์ตามธรรมเนียม `*_spec.rb`, `describe`/`it` เบื้องต้น

### ธรรมเนียมการตั้งชื่อไฟล์และโฟลเดอร์

RSpec มีข้อกำหนด (convention) ที่ชัดเจนมาก:

1. ไฟล์เทสต์ทั้งหมดอยู่ในโฟลเดอร์ `spec/` ที่ root ของโปรเจกต์
2. ไฟล์เทสต์ต้องลงท้ายด้วย `_spec.rb` เสมอ (เช่น `calculator_spec.rb` ทดสอบ `calculator.rb`)
3. โครงสร้างโฟลเดอร์ภายใน `spec/` มักจะ mirror โครงสร้างของ `lib/` (เช่น `lib/models/user.rb`
   คู่กับ `spec/models/user_spec.rb`)

เมื่อรันคำสั่ง `rspec` เฉยๆ โดยไม่ระบุ path ระบบจะสแกนหาไฟล์ `spec/**/*_spec.rb` ทั้งหมด
โดยอัตโนมัติและรันทุกไฟล์

```bash
mkdir -p lib spec
```

สร้าง `lib/calculator.rb`:

```ruby
# frozen_string_literal: true

class Calculator
  def add(a, b)
    a + b
  end

  def divide(a, b)
    raise ZeroDivisionError, "หารด้วยศูนย์ไม่ได้" if b.zero?

    a / b.to_f
  end
end
```

สร้าง `spec/calculator_spec.rb`:

```ruby
# frozen_string_literal: true

require_relative "../lib/calculator"

RSpec.describe Calculator do
  it "บวกเลขสองจำนวนได้ผลลัพธ์ถูกต้อง" do
    calculator = Calculator.new
    result = calculator.add(2, 3)
    expect(result).to eq(5)
  end

  it "หารเลขสองจำนวนได้ผลลัพธ์เป็นทศนิยม" do
    calculator = Calculator.new
    result = calculator.divide(10, 4)
    expect(result).to eq(2.5)
  end
end
```

### รันเทสต์

```bash
rspec spec/calculator_spec.rb
```

ผลลัพธ์แบบค่าเริ่มต้น (progress format — แสดง `.` ต่อ 1 example ที่ผ่าน):

```
..

Finished in 0.00187 seconds (files took 0.04321 seconds to load)
2 examples, 0 failures
```

รันแบบอ่านง่ายขึ้นด้วย `--format documentation`:

```bash
rspec spec/calculator_spec.rb --format documentation
```

```
Calculator
  บวกเลขสองจำนวนได้ผลลัพธ์ถูกต้อง
  หารเลขสองจำนวนได้ผลลัพธ์เป็นทศนิยม

Finished in 0.00159 seconds (files took 0.04287 seconds to load)
2 examples, 0 failures
```

รันทั้งโปรเจกต์โดยไม่ระบุไฟล์ (สแกน `spec/` ทั้งหมด):

```bash
rspec
```

### กายวิภาคของ spec file

- `RSpec.describe Calculator do ... end` — ประกาศกลุ่มเทสต์ (**example group**) สำหรับ class
  `Calculator` รับ argument เป็น class/module/string ก็ได้ (string ใช้เมื่อไม่มี class จริง
  เช่น อธิบาย feature รวมๆ)
- `it "คำอธิบาย" do ... end` — ประกาศเทสต์ 1 เคส เรียกว่า **example** คำอธิบายควรเขียนให้
  ต่อกับชื่อ `describe`/`context` ด้านนอกแล้วอ่านเป็นประโยคสมบูรณ์ได้
- `expect(result).to eq(5)` — **expectation** ตรวจสอบว่า `result` เท่ากับ `5` โดยใช้ matcher
  ชื่อ `eq` (จะเรียนละเอียดใน Step 184–185)
- ถ้า expectation ทุกตัวใน example ผ่านหมด → example นั้น "pass" ถ้ามีตัวใดตัวหนึ่ง fail →
  example นั้น "fail" ทันที (คล้ายหลักการเดียวกับ assertion ใน Minitest)

### ตัวอย่างเมื่อเทสต์ fail

ลองแก้ expectation ให้ผิดโดยตั้งใจเพื่อดูผลลัพธ์:

```ruby
it "ตัวอย่างที่ตั้งใจให้ fail" do
  calculator = Calculator.new
  expect(calculator.add(2, 3)).to eq(999)
end
```

```
Failures:

  1) Calculator ตัวอย่างที่ตั้งใจให้ fail
     Failure/Error: expect(calculator.add(2, 3)).to eq(999)

       expected: 999
            got: 5

       (compared using ==)
     # ./spec/calculator_spec.rb:15:in `block (2 levels) in <top (required)>'

Finished in 0.00203 seconds (files took 0.04198 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/calculator_spec.rb:14 # Calculator ตัวอย่างที่ตั้งใจให้ fail
```

สังเกตว่า RSpec บอกทั้ง `expected` (ค่าที่คาดหวัง) และ `got` (ค่าจริงที่ได้) พร้อมวิธีเปรียบเทียบ
(`compared using ==`) และยังให้คำสั่ง `rspec ./spec/calculator_spec.rb:14` ที่ copy ไปรันซ้ำ
เฉพาะเทสต์ที่ fail ตัวนั้นได้ทันที — สะดวกมากเวลามีเทสต์เป็นร้อยตัวแล้วอยากรีรันเฉพาะตัวที่พัง

---

## Step 183: `describe` vs `context` — ความหมายเชิงความหมาย (semantic) ที่ต่างกัน

### ความจริงทางเทคนิค: `context` คือ alias ของ `describe`

ในเชิง implementation `context` เป็นเพียง **alias** ของ `describe` เฉยๆ ทั้งสองคำสั่งทำงาน
เหมือนกันทุกประการ 100% ไม่มีความต่างด้าน behavior ใดๆ เลย — แต่ RSpec แยกคำสั่งไว้ 2 ชื่อ
เพื่อให้ **นักพัฒนาสื่อสารความหมายที่ต่างกัน** ผ่านการเลือกใช้คำ

### กฎการเลือกใช้ในทางปฏิบัติ

- **`describe`** — ใช้อธิบาย "สิ่งที่กำลังถูกทดสอบ" (the subject) เช่น class, module,
  หรือ method หนึ่งตัว ธรรมเนียมของวงการคือใส่ `#` นำหน้าชื่อ instance method และ `.` นำหน้า
  ชื่อ class method
- **`context`** — ใช้อธิบาย "สถานการณ์/เงื่อนไข" (the state or condition) ที่กำลังทดสอบ
  ธรรมเนียมคือขึ้นต้นด้วย `"when "`, `"with "`, `"without "` แล้วตามด้วยเงื่อนไข

```ruby
# frozen_string_literal: true

require_relative "../lib/calculator"

RSpec.describe Calculator do
  describe "#divide" do
    context "เมื่อตัวหารไม่ใช่ศูนย์" do
      it "คืนค่าผลหารเป็นทศนิยม" do
        calculator = Calculator.new
        expect(calculator.divide(10, 4)).to eq(2.5)
      end
    end

    context "เมื่อตัวหารเป็นศูนย์" do
      it "raise ZeroDivisionError" do
        calculator = Calculator.new
        expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError)
      end
    end
  end

  describe "#add" do
    it "บวกเลขบวกสองจำนวนได้ถูกต้อง" do
      calculator = Calculator.new
      expect(calculator.add(2, 3)).to eq(5)
    end

    context "เมื่อมีตัวเลขติดลบปนอยู่" do
      it "ยังคงบวกได้ถูกต้อง" do
        calculator = Calculator.new
        expect(calculator.add(-2, 5)).to eq(3)
      end
    end
  end
end
```

รันด้วย `--format documentation` แล้วดูว่าผลลัพธ์อ่านง่ายขนาดไหนเมื่อซ้อนกันหลายชั้น:

```
Calculator
  #divide
    เมื่อตัวหารไม่ใช่ศูนย์
      คืนค่าผลหารเป็นทศนิยม
    เมื่อตัวหารเป็นศูนย์
      raise ZeroDivisionError
  #add
    บวกเลขบวกสองจำนวนได้ถูกต้อง
    เมื่อมีตัวเลขติดลบปนอยู่
      ยังคงบวกได้ถูกต้อง

Finished in 0.00214 seconds (files took 0.04365 seconds to load)
4 examples, 0 failures
```

**ทำไมต้องใส่ใจเรื่องนี้ทั้งที่มันทำงานเหมือนกัน:** เพราะผลลัพธ์นี้จะกลายเป็น **เอกสาร
ประกอบระบบ (living documentation)** ที่คนอื่นในทีมอ่านแล้วเข้าใจพฤติกรรมของโค้ดได้ทันทีโดย
ไม่ต้องเปิดโค้ดจริง ถ้าใช้ `describe`/`context` ปนกันมั่วๆ ผลลัพธ์จะอ่านไม่รู้เรื่อง เสีย
จุดขายสำคัญที่สุดของ RSpec ไปเลย ทีมมืออาชีพส่วนใหญ่ยังตั้งกฎเสริมด้วย RuboCop cop ชื่อ
`RSpec/ContextWording` เพื่อบังคับให้ `context` ต้องขึ้นต้นด้วย "when"/"with"/"without" เท่านั้น

> **จำง่ายๆ:** `describe` ตอบคำถาม "กำลังทดสอบอะไร" ส่วน `context` ตอบคำถาม "ภายใต้เงื่อนไข
> แบบไหน"

---

## Step 184: `expect(...).to` matcher syntax เทียบกับ `should` syntax แบบเก่า

### `should` syntax (แบบเก่า — ไม่ควรใช้แล้ว)

RSpec เวอร์ชันก่อน 3.0 ใช้ syntax แบบนี้เป็นค่าเริ่มต้น:

```ruby
# แบบเก่า — ห้ามใช้ในโค้ดใหม่ แสดงไว้เพื่อให้จำได้เวลาเจอในโค้ด legacy เท่านั้น
1.should == 1
[1, 2, 3].should include(2)
calculator.add(2, 3).should eq(5)
```

`should` ทำงานได้เพราะ RSpec **monkey-patch** method `#should` เข้าไปใน `Object` ทุกตัวใน
ระบบ (ตามที่เรียนเรื่อง open classes ใน Part 015) ปัญหาคือ:

1. **ชนกับ library อื่น** — ถ้า gem ตัวอื่นก็นิยาม method `#should` เอาไว้เหมือนกัน (หรือมี
   object ที่ไม่ยอมให้ patch เช่น `BasicObject` บางกรณี) จะเกิดความขัดแย้งที่ debug ยาก
2. **มองไม่เห็นจากภายนอก object** — เพราะ `should` เป็น method ของ object นั้นเอง ทำให้ยากที่
   จะเขียน custom matcher หรือขยายความสามารถอย่างสะอาด
3. **แก้ syntax ให้ตรงกับ Ruby core class ยาก** — object ที่ override `method_missing` (เช่น
   dynamic object จาก Part 015) อาจทำให้ `should` ทำงานผิดเพี้ยนโดยไม่รู้ตัว

### `expect(...).to` syntax (มาตรฐานปัจจุบัน — ใช้เสมอ)

```ruby
expect(1).to eq(1)
expect([1, 2, 3]).to include(2)
expect(calculator.add(2, 3)).to eq(5)
```

`expect` เป็น **method ระดับ top-level ของ RSpec เอง** (ไม่ใช่ monkey-patch ของ `Object`)
รับค่าที่จะตรวจสอบเป็น argument แล้วคืน object ตัวกลางที่มี method `.to`/`.not_to` สำหรับ
เทียบกับ matcher — ไม่ไปแตะต้อง class ใดๆ ใน Ruby เลย จึงไม่มีปัญหาชนกับ library อื่น และ
ทำงานได้แม้กับ object แปลกๆ ที่ override `method_missing` ไว้

**ปฏิเสธ (negative) expectation:**

```ruby
expect(calculator.add(2, 3)).not_to eq(999)
expect([1, 2, 3]).not_to include(5)
```

### Matcher พื้นฐานที่ต้องรู้ก่อน: `eq`, `eql`, `equal`/`be`

Matcher ทั้งสามตัวนี้ดู "เหมือนกัน" แต่เปรียบเทียบด้วยกลไกคนละแบบ ต้องแยกให้ออก:

```ruby
# frozen_string_literal: true

# eq -- เปรียบเทียบด้วย == (value equality แบบยืดหยุ่น)
expect(1).to eq(1)        # ผ่าน: 1 == 1
expect(1).to eq(1.0)      # ผ่าน! เพราะ 1 == 1.0 เป็น true ใน Ruby (ข้าม type ได้)

# eql -- เปรียบเทียบด้วย eql? (value equality แบบเข้มงวดเรื่อง type)
expect(1).to eql(1)       # ผ่าน: 1.eql?(1) เป็น true (type และ value ตรงกัน)
expect(1).to eql(1.0)     # ไม่ผ่าน! เพราะ 1.eql?(1.0) เป็น false (Integer ไม่ใช่ Float)

# equal / be -- เปรียบเทียบด้วย equal? (object identity, เช็คว่าเป็น object เดียวกันจริงๆ)
a = "hello"
b = "hello"
c = a

expect(a).to eq(b)        # ผ่าน: ค่าเหมือนกัน ("hello" == "hello")
expect(a).to eql(b)       # ผ่าน: type และ value เหมือนกัน
expect(a).to equal(b)     # ไม่ผ่าน! a กับ b คือ String object คนละตัวกัน (object_id ต่างกัน)
expect(a).to equal(c)     # ผ่าน: c คือตัวแปรที่ชี้ไปยัง object เดียวกับ a (c = a)

# be เป็น alias ของ equal
expect(a).to be(c)        # ผ่าน เหมือนกับ equal(c) ทุกประการ
```

**กฎการเลือกใช้ในทางปฏิบัติ:**

- ใช้ `eq` เป็นค่าเริ่มต้นเกือบตลอดเวลา (ยืดหยุ่นที่สุด ตรงกับสามัญสำนึกของ "ค่าเท่ากันไหม")
- ใช้ `eql` เฉพาะเมื่อต้องการเข้มงวดเรื่อง type จริงๆ (พบไม่บ่อยในโค้ดทั่วไป)
- ใช้ `equal`/`be` เฉพาะเมื่อต้องการยืนยันว่าเป็น **object เดียวกันในหน่วยความจำ** จริงๆ เช่น
  ตรวจสอบว่า method คืนค่า `self` กลับมา (สำหรับ method chaining) หรือตรวจสอบ singleton
  object เช่น `nil`, `true`, `false`, symbol

```ruby
class Calculator
  def add(a, b)
    a + b
  end

  def reset_and_return_self
    self
  end
end

calculator = Calculator.new
expect(calculator.reset_and_return_self).to be(calculator)  # ยืนยันว่าคืน self ตัวเดิมจริงๆ
```

---

## Step 185: Matcher มาตรฐานที่ใช้บ่อย: `include`, `match`, `be_within`, `raise_error`, `change`

RSpec มี matcher สำเร็จรูปให้ใช้จำนวนมาก ในหัวข้อนี้จะรวบรวมตัวที่ใช้บ่อยที่สุดในงานจริง

### Truthy / Falsey / Nil

```ruby
expect(1).to be_truthy      # ผ่านถ้าค่าเป็น truthy (ไม่ใช่ nil และไม่ใช่ false)
expect(nil).to be_falsey    # ผ่านถ้าค่าเป็น falsey (nil หรือ false)
expect(false).to be_falsey
expect(nil).to be_nil       # ผ่านเฉพาะเมื่อเป็น nil เท่านั้น (เข้มงวดกว่า be_falsey)

# ข้อควรระวัง: be_truthy ไม่ใช่ eq(true) — ตัวเลข, string, array ที่ไม่ใช่ nil/false
# ล้วนเป็น truthy หมด (ตามกฎ truthy/falsy ของ Ruby ที่เรียนใน Part 006)
expect(0).to be_truthy      # ผ่าน! เพราะใน Ruby เลข 0 เป็น truthy (ต่างจากภาษาอื่น)
expect("").to be_truthy     # ผ่าน! string ว่างก็เป็น truthy เช่นกัน
```

### Predicate matcher — จับคู่อัตโนมัติกับ method ที่ลงท้ายด้วย `?`

RSpec แปลง `be_xxx` หรือ `be_a_xxx`/`be_an_xxx` เป็นการเรียก method `xxx?` บน object ให้
อัตโนมัติ (ใช้ `method_missing` เบื้องหลัง เชื่อมโยงกับที่เรียนใน Part 015):

```ruby
expect([]).to be_empty          # เรียก [].empty?   เบื้องหลัง
expect(4).to be_even            # เรียก 4.even?     เบื้องหลัง
expect(3).to be_odd             # เรียก 3.odd?      เบื้องหลัง
expect(5).to be_a(Integer)      # เรียก 5.is_a?(Integer) เบื้องหลัง
expect(5).to be_an_instance_of(Integer)  # เรียก 5.instance_of?(Integer)
expect(nil).to be_nil           # เรียก nil.nil?
```

### `include` — ตรวจสอบว่ามีสมาชิกอยู่ (Array, Hash, String)

```ruby
expect([1, 2, 3]).to include(2)               # Array มีสมาชิก 2 อยู่ไหม
expect([1, 2, 3]).to include(2, 3)            # ตรวจได้หลายค่าพร้อมกัน (ต้องมีครบทุกตัว)
expect({ name: "มานี", age: 25 }).to include(name: "มานี")  # Hash มี key/value คู่นี้ไหม
expect("Hello, Ruby!").to include("Ruby")     # String มี substring นี้อยู่ไหม
```

### `match` — จับคู่กับ Regexp หรือโครงสร้างข้อมูลซับซ้อน

```ruby
expect("089-123-4567").to match(/\A\d{3}-\d{3}-\d{4}\z/)   # ตรวจ format เบอร์โทร
expect("hello@example.com").to match(/\A[^@\s]+@[^@\s]+\z/)

# match ใช้กับโครงสร้าง Hash/Array ที่ซับซ้อนได้ด้วย พร้อม matcher ซ้อนกันเอง
user = { name: "มานี", age: 25, tags: ["admin", "active"] }
expect(user).to match(
  name: "มานี",
  age: be > 18,
  tags: include("admin")
)
```

### `be_within` — เปรียบเทียบตัวเลขทศนิยมแบบมีค่าคลาดเคลื่อนที่ยอมรับได้

เลขทศนิยม (`Float`) ไม่ควรเทียบด้วย `eq` ตรงๆ เพราะปัญหา floating-point precision (ปัญหา
คลาสสิกในทุกภาษาโปรแกรมมิ่ง ไม่ใช่แค่ Ruby):

```ruby
# ตัวอย่างปัญหา floating-point ที่พบได้ในทุกภาษา
puts 0.1 + 0.2 == 0.3    # => false! (ผลจริงคือ 0.30000000000000004)

# วิธีแก้ด้วย be_within
expect(0.1 + 0.2).to be_within(0.0001).of(0.3)   # ผ่าน: คลาดเคลื่อนไม่เกิน 0.0001

calculator = Calculator.new
expect(calculator.divide(10, 3)).to be_within(0.01).of(3.33)   # ผ่าน: 3.333... ≈ 3.33
```

### `raise_error` — ตรวจสอบว่ามีการ raise exception

ต้องใช้คู่กับ **block form** ของ `expect { }` เสมอ (ไม่ใช่ `expect()` แบบมีวงเล็บ) เพราะ
ต้องดักจับ exception ตอนที่โค้ด**กำลังรัน** ไม่ใช่ค่าที่รันเสร็จแล้ว:

```ruby
calculator = Calculator.new

# ตรวจแค่ว่า raise exception class นี้ (ไม่สนข้อความ)
expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError)

# ตรวจทั้ง class และข้อความ (exact match)
expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError, "หารด้วยศูนย์ไม่ได้")

# ตรวจข้อความด้วย regex (เมื่อไม่อยากเทียบข้อความแบบตรงเป๊ะ)
expect { calculator.divide(10, 0) }.to raise_error(ZeroDivisionError, /หารด้วยศูนย์/)

# ตรวจว่า "ไม่มี" exception ใดๆ ถูก raise เลย
expect { calculator.add(2, 3) }.not_to raise_error

# ผิดวิธี! ห้ามเขียนแบบนี้ เพราะ exception จะถูก raise ก่อนที่ expect() จะได้ทำงาน
# expect(calculator.divide(10, 0)).to raise_error(ZeroDivisionError)   # โปรแกรมจะ crash ตรงนี้เลย
```

### `change` — ตรวจสอบว่าค่าของบางสิ่งเปลี่ยนไปหลังรันโค้ด (block form เช่นกัน)

```ruby
class Counter
  attr_reader :value

  def initialize
    @value = 0
  end

  def increment
    @value += 1
  end
end

counter = Counter.new

# ตรวจว่าค่าเปลี่ยนไปเท่าไหร่
expect { counter.increment }.to change { counter.value }.by(1)

# ตรวจว่าเปลี่ยนจากค่าหนึ่งไปอีกค่าหนึ่งแบบเจาะจง
expect { counter.increment }.to change { counter.value }.from(1).to(2)

# ตรวจแค่ว่า "มีการเปลี่ยนแปลงเกิดขึ้น" โดยไม่สนทิศทาง/จำนวน
expect { counter.increment }.to change { counter.value }

# ตรวจว่า "ไม่มี" การเปลี่ยนแปลงเกิดขึ้น
expect { counter.value }.not_to change { counter.value }
```

**สรุปตารางเปรียบเทียบ matcher ที่เรียนไปทั้งหมด:**

| Matcher | ใช้เมื่อ |
|---|---|
| `eq` | เปรียบเทียบค่าทั่วไปด้วย `==` (ใช้เป็นค่าเริ่มต้นเสมอ) |
| `eql` | เปรียบเทียบค่า+type แบบเข้มงวดด้วย `eql?` |
| `equal` / `be` | เปรียบเทียบว่าเป็น object เดียวกัน (identity) |
| `be_truthy` / `be_falsey` / `be_nil` | ตรวจสอบ truthiness ตามกฎ Ruby |
| `be_xxx` (predicate) | จับคู่อัตโนมัติกับ method `xxx?` |
| `include` | ตรวจว่า Array/Hash/String มีสมาชิก/substring นี้อยู่ |
| `match` | จับคู่กับ Regexp หรือโครงสร้างข้อมูลซับซ้อน |
| `be_within(delta).of(x)` | เทียบเลขทศนิยมแบบมีค่าคลาดเคลื่อนที่ยอมรับได้ |
| `raise_error` | ตรวจว่ามีการ raise exception (ต้องใช้ block form) |
| `change` | ตรวจว่าค่าบางอย่างเปลี่ยนไปหลังรันโค้ด (ต้องใช้ block form) |

---

## Step 186: `let`/`let!` — lazy-evaluated memoized test data เทียบกับ instance variable ใน `before`

### ปัญหาของการใช้ instance variable ใน `before` แบบดั้งเดิม

```ruby
# frozen_string_literal: true

require_relative "../lib/calculator"

RSpec.describe Calculator do
  before do
    @calculator = Calculator.new
  end

  it "บวกได้ถูกต้อง" do
    expect(@calculator.add(2, 3)).to eq(5)
  end

  it "หารได้ถูกต้อง" do
    expect(@calculator.divide(10, 2)).to eq(5.0)
  end
end
```

โค้ดนี้ **ใช้งานได้ปกติ** ไม่มีอะไรผิด แต่มีข้อเสียเชิงปฏิบัติ:

1. `@calculator` ถูกสร้างขึ้น**ทุก example** แม้ example นั้นจะไม่ได้ใช้งานเลยก็ตาม
   (สิ้นเปลืองถ้าการสร้างมีต้นทุนสูง เช่น เชื่อมต่อฐานข้อมูล)
2. ถ้าพิมพ์ชื่อผิด (`@calculater` แทน `@calculator`) Ruby จะไม่ error แต่คืนค่า `nil` เงียบๆ
   (ตามพฤติกรรมของ instance variable ที่ยังไม่ได้ตั้งค่า) ทำให้ debug ยาก
3. ไม่มีการรับประกันว่า `before` รันก่อนเสมอถ้ามีหลาย `before` ซ้อนกันในหลายชั้น

### `let` — แก้ปัญหาด้วย lazy evaluation + memoization

```ruby
# frozen_string_literal: true

require_relative "../lib/calculator"

RSpec.describe Calculator do
  let(:calculator) { Calculator.new }

  it "บวกได้ถูกต้อง" do
    expect(calculator.add(2, 3)).to eq(5)
  end

  it "หารได้ถูกต้อง" do
    expect(calculator.divide(10, 2)).to eq(5.0)
  end

  it "ตัวอย่างที่ไม่ได้ใช้ calculator เลย" do
    expect(1 + 1).to eq(2)
    # calculator ไม่เคยถูกสร้างขึ้นเลยใน example นี้ เพราะไม่มีการเรียกใช้ชื่อ calculator
  end
end
```

**`let(:calculator) { ... }` ทำงานอย่างไร:**

- นิยาม **method** ชื่อ `calculator` ให้ example group นั้น (ไม่ใช่แค่ตัวแปร) block ด้านใน
  จะยังไม่ถูกรันจนกว่าจะมีการเรียก `calculator` ครั้งแรกภายใน example (**lazy evaluation**)
- เมื่อเรียก `calculator` ครั้งแรกในหนึ่ง example ผลลัพธ์จะถูก **memoize** (จำค่าไว้) ดังนั้น
  ถ้าเรียก `calculator` ซ้ำอีกหลายครั้งใน example เดียวกัน จะได้ **object เดียวกันเป๊ะ** ไม่ถูก
  สร้างใหม่ทุกครั้ง
- แต่พอขึ้น example ถัดไป ค่าที่ memoize ไว้จะถูกล้างทิ้ง แล้วเริ่มใหม่ทั้งหมด — แต่ละ
  example จึงเป็นอิสระจากกันเสมอ (ตรงตามหลักการที่เรียนใน Step 181 เรื่อง `config.order = :random`)
- ถ้าพิมพ์ชื่อผิด (`calculater`) Ruby จะ raise `NoMethodError` ทันที เพราะไม่มี method นี้
  นิยามอยู่จริง — ต่างจาก instance variable ที่คืน `nil` เงียบๆ

พิสูจน์เรื่อง memoization ด้วยตัวอย่างนี้:

```ruby
RSpec.describe "การพิสูจน์ memoization ของ let" do
  let(:random_number) { rand(1..1_000_000) }

  it "เรียกซ้ำในตัวอย่างเดียวกันได้ค่าเท่ากันเสมอ" do
    first_call = random_number
    second_call = random_number
    expect(first_call).to eq(second_call)   # ผ่านเสมอ เพราะถูก memoize ไว้แล้ว
  end
end
```

### `let!` — บังคับให้ evaluate ทันทีก่อนทุก example (eager)

บางครั้งเราต้องการให้ side effect ของการสร้างข้อมูลเกิดขึ้นจริงๆ ก่อนที่ example จะเริ่มรัน
แม้ว่า example นั้นจะไม่ได้เอ่ยชื่อ `let` ตัวนั้นเลยก็ตาม (พบบ่อยมากเวลาต้อง insert ข้อมูลลง
ฐานข้อมูลไว้ล่วงหน้าแล้วเทสต์ query ที่ไม่รู้จัก record นั้นโดยตรง — จะเห็นชัดเจนขึ้นมากตอน
เรียน Rails model spec ใน Part 046)

```ruby
class AuditLog
  @entries = []

  class << self
    attr_reader :entries

    def record(message)
      @entries << message
    end

    def reset!
      @entries = []
    end
  end
end

RSpec.describe AuditLog do
  before { described_class.reset! }

  context "ใช้ let (lazy) — entry จะไม่ถูกสร้างถ้าไม่เรียกใช้" do
    let(:entry) { described_class.record("เหตุการณ์ A") }

    it "ไม่มี entry ใดๆ เลยถ้าไม่เรียก entry ในเทสต์นี้" do
      expect(described_class.entries).to be_empty
    end
  end

  context "ใช้ let! (eager) — entry ถูกสร้างเสมอก่อนเทสต์เริ่ม" do
    let!(:entry) { described_class.record("เหตุการณ์ B") }

    it "มี entry อยู่แล้วแม้ไม่ได้เรียกชื่อ entry ในเทสต์นี้เลย" do
      expect(described_class.entries).to eq(["เหตุการณ์ B"])
    end
  end
end
```

`let!(:entry) { ... }` เทียบเท่ากับการเขียน `before { entry }` ต่อท้าย `let(:entry) { ... }`
โดยอัตโนมัติ (เรียก method `entry` ใน hook `before` เพื่อบังคับให้ block ทำงาน)

**กฎการเลือกใช้ในทางปฏิบัติ:** ใช้ `let` เป็นค่าเริ่มต้นเสมอ เปลี่ยนไปใช้ `let!` เฉพาะเมื่อ
ต้องการ side effect (เช่น การบันทึกข้อมูล) เกิดขึ้นก่อน example เริ่ม โดยที่ example นั้นไม่ได้
อ้างอิงชื่อตัวแปรนั้นตรงๆ

---

## Step 187: `before`/`after` hooks — `:each` กับ `:context`/`:all`

RSpec มี hook 4 แบบหลัก แบ่งตาม **timing** (ก่อน/หลัง) และ **scope** (ต่อ example หรือ
ต่อทั้งกลุ่ม):

| Hook | Alias | ความถี่ในการรัน |
|---|---|---|
| `before(:each)` | `before(:example)` (หรือ `before` เฉยๆ) | รันก่อน **ทุก** example |
| `after(:each)` | `after(:example)` (หรือ `after` เฉยๆ) | รันหลัง **ทุก** example |
| `before(:context)` | `before(:all)` | รัน **ครั้งเดียว** ก่อน example ตัวแรกในกลุ่ม |
| `after(:context)` | `after(:all)` | รัน **ครั้งเดียว** หลัง example ตัวสุดท้ายในกลุ่ม |

### ทดลองดูลำดับการรันจริง

```ruby
# frozen_string_literal: true

RSpec.describe "Hook lifecycle" do
  before(:context) { puts "\n[before context] เริ่มต้นชุดทดสอบ (รันครั้งเดียว)" }
  after(:context)  { puts "[after context] จบชุดทดสอบ (รันครั้งเดียว)" }

  before(:each) { puts "  [before each] เตรียมข้อมูลก่อนแต่ละ example" }
  after(:each)  { puts "  [after each] เก็บกวาดหลังแต่ละ example" }

  it "ตัวอย่างที่ 1" do
    puts "    -> รัน example ที่ 1"
  end

  it "ตัวอย่างที่ 2" do
    puts "    -> รัน example ที่ 2"
  end
end
```

```bash
rspec hook_lifecycle_spec.rb --format documentation
```

ผลลัพธ์ (สังเกตว่า RSpec formatter พิมพ์ชื่อ example group ก่อน แล้วค่อยพิมพ์บรรทัด
"ตัวอย่างที่ N" ต่อท้ายทันทีที่ example นั้นรันเสร็จ จึงเห็น `puts` จากในโค้ดแทรกอยู่ก่อน
บรรทัดชื่อ example เสมอ):

```
Hook lifecycle

[before context] เริ่มต้นชุดทดสอบ (รันครั้งเดียว)
  [before each] เตรียมข้อมูลก่อนแต่ละ example
    -> รัน example ที่ 1
  [after each] เก็บกวาดหลังแต่ละ example
  ตัวอย่างที่ 1
  [before each] เตรียมข้อมูลก่อนแต่ละ example
    -> รัน example ที่ 2
  [after each] เก็บกวาดหลังแต่ละ example
  ตัวอย่างที่ 2
[after context] จบชุดทดสอบ (รันครั้งเดียว)

Finished in 0.00091 seconds (files took 0.06654 seconds to load)
2 examples, 0 failures
```

สังเกตลำดับ: `before(:context)` รันแค่ครั้งเดียวตอนต้น จากนั้น `before(:each)`/`after(:each)`
รันวนซ้ำรอบละ 1 example และ `after(:context)` รันแค่ครั้งเดียวตอนท้ายสุด

### ข้อควรระวังสำคัญของ `before(:context)`

```ruby
RSpec.describe "ตัวอย่างข้อผิดพลาดที่พบบ่อย" do
  before(:context) do
    @shared_list = []   # ตั้งค่าครั้งเดียว แชร์กันทุก example ในกลุ่มนี้
  end

  it "เพิ่มข้อมูลเข้า list" do
    @shared_list << "a"
    expect(@shared_list).to eq(["a"])
  end

  it "คาดหวังว่า list จะว่างเปล่าเหมือนเทสต์ทั่วไป -- แต่จะ FAIL!" do
    # @shared_list ไม่ได้ถูกรีเซ็ตระหว่าง example เพราะสร้างใน before(:context)
    # ค่า "a" จาก example ก่อนหน้ายังคงค้างอยู่ -- ทำให้เทสต์นี้พังอย่างไม่คาดคิด
    expect(@shared_list).to be_empty   # FAIL: expected [] got ["a"]
  end
end
```

**เหตุผล:** instance variable ที่ตั้งค่าใน `before(:context)` **ไม่ถูกรีเซ็ตระหว่าง example**
เพราะ hook นี้รันแค่ครั้งเดียวสำหรับทั้งกลุ่ม ถ้าตัว object นั้น mutable (เช่น Array, Hash) และ
มี example ใด example หนึ่งไปแก้ไขมัน จะเกิดผลข้างเคียงข้ามไปยัง example อื่นทันที ทำให้ผล
การเทสต์ขึ้นอยู่กับ**ลำดับการรัน** ซึ่งขัดกับหลักการพื้นฐานที่เรียนใน Step 181
(`config.order = :random`)

**แนวปฏิบัติที่ถูกต้อง:** ใช้ `before(:context)` เฉพาะกับข้อมูลที่ **อ่านอย่างเดียว
(read-only) และมีต้นทุนการสร้างสูงจริงๆ** เช่น โหลดไฟล์ config ขนาดใหญ่ หรือเชื่อมต่อ
ทรัพยากรภายนอกที่ใช้เวลานาน ส่วนข้อมูลที่ต้อง mutate หรือแตกต่างกันในแต่ละ example ให้ใช้
`before(:each)` หรือ `let`/`let!` เสมอ — ในทางปฏิบัติ ทีมส่วนใหญ่แทบไม่ใช้ `before(:context)`
เลย เพราะความเสี่ยงสูงกว่าประโยชน์ที่ได้

### hook ซ้อนกันหลายชั้น (nested describe/context)

```ruby
RSpec.describe Calculator do
  before { puts "[outer before] ระดับนอกสุด" }

  describe "#add" do
    before { puts "  [inner before] ระดับ #add" }

    it "ตัวอย่างที่ 1" do
      puts "    -> example รันจริง"
    end
  end
end
```

ผลลัพธ์: `[outer before]` จะรันก่อน `[inner before]` เสมอ (hook ของกลุ่มนอกรันก่อนกลุ่มใน
เสมอ ไม่ว่าจะซ้อนกี่ชั้นก็ตาม) — เป็นหลักการเดียวกับที่ `super` เรียก method ของ parent
class ก่อนใน inheritance chain ตามที่เรียนใน Part 010

---

## Step 188: `subject` และ `described_class`

### `subject` — ค่าที่ "กำลังถูกทดสอบ" แบบไม่ต้องตั้งชื่อเอง

เมื่อ example group ส่วนใหญ่ใน `describe` เดียวกันทดสอบ object ตัวเดียวกันซ้ำๆ RSpec เตรียม
`subject` ให้ใช้แทน `let` ที่ต้องตั้งชื่อเอง:

```ruby
RSpec.describe Calculator do
  subject { Calculator.new }

  it "บวกได้ถูกต้อง" do
    expect(subject.add(2, 3)).to eq(5)
  end
end
```

`subject` ที่ไม่ตั้งชื่อ ถ้า `describe` รับ argument เป็น class จะสร้าง instance ของ class
นั้นด้วย `.new` ให้อัตโนมัติเป็นค่าเริ่มต้น (ไม่ต้องเขียน block เองด้วยซ้ำถ้า constructor ไม่รับ
argument):

```ruby
RSpec.describe Calculator do
  # ไม่ต้องเขียน subject { Calculator.new } เลยก็ได้ -- RSpec สร้างให้อัตโนมัติ

  it "บวกได้ถูกต้อง" do
    expect(subject.add(2, 3)).to eq(5)
  end
end
```

### ตั้งชื่อ subject ด้วย `subject(:name)` — แนะนำให้ใช้เสมอในโค้ดจริง

```ruby
RSpec.describe Calculator do
  subject(:calculator) { described_class.new }

  it "บวกได้ถูกต้อง" do
    expect(calculator.add(2, 3)).to eq(5)
  end
end
```

การตั้งชื่อทำให้โค้ดอ่านง่ายขึ้นมาก (`calculator.add` อ่านชัดกว่า `subject.add`) และยังเรียก
ผ่านชื่อ `subject` เฉยๆ ได้เหมือนเดิมด้วย — `subject(:calculator)` สร้างทั้ง method `subject`
และ method `calculator` ที่ชี้ไปยัง object เดียวกัน

### `described_class` — อ้างอิง class ที่ `describe` ครอบอยู่แบบไม่ต้องพิมพ์ชื่อซ้ำ

```ruby
RSpec.describe Calculator do
  it "described_class คือ Calculator" do
    expect(described_class).to eq(Calculator)
  end

  subject(:calculator) { described_class.new }   # เท่ากับเขียน Calculator.new ตรงๆ
end
```

**ประโยชน์หลักของ `described_class`:** ถ้าวันหนึ่งเปลี่ยนชื่อ class จาก `Calculator` เป็น
`AdvancedCalculator` เราแก้แค่บรรทัด `RSpec.describe AdvancedCalculator do` บรรทัดเดียว โดย
ไม่ต้องไล่แก้ทุกจุดที่เคยพิมพ์ชื่อ `Calculator.new` ซ้ำๆ ทั่วทั้งไฟล์ นอกจากนี้ยังเป็นกุญแจ
สำคัญที่ทำให้ `shared_examples` (Step 189) ใช้ซ้ำข้ามหลาย class ได้จริง เพราะไม่ต้องรู้ล่วง
หน้าว่า class ปลายทางชื่ออะไร

### One-liner syntax — `is_expected.to`

เมื่อมี `subject` (ไม่ว่าตั้งชื่อหรือไม่) และ example ต้องการตรวจสอบ subject โดยตรงแบบสั้นๆ
ใช้ `is_expected.to` แทน `expect(subject).to` ได้:

```ruby
RSpec.describe String do
  describe "#upcase" do
    subject { "hello".upcase }

    it { is_expected.to eq("HELLO") }
  end

  describe "#empty?" do
    context "เมื่อ string ว่างเปล่า" do
      subject { "" }

      it { is_expected.to be_empty }
    end

    context "เมื่อ string มีตัวอักษร" do
      subject { "ruby" }

      it { is_expected.not_to be_empty }
    end
  end
end
```

`is_expected.to eq("HELLO")` เทียบเท่ากับ `expect(subject).to eq("HELLO")` ทุกประการ รูปแบบ
นี้เหมาะกับเทสต์สั้นๆ ที่ตรวจสอบเงื่อนไขเดียวชัดเจน (มักไม่ใส่คำอธิบายใน `it` เลยด้วยซ้ำ เพราะ
บรรทัดชื่อ `context`/`describe` ด้านนอกอธิบายครบอยู่แล้ว) แต่ไม่ควรใช้พร่ำเพรื่อกับเทสต์ที่
ซับซ้อนหลายขั้นตอน เพราะจะทำให้อ่านยากลงแทน

---

## Step 189: `shared_examples`/`it_behaves_like` — เขียน spec แบบ DRY ข้ามหลาย class

เมื่อมีหลาย class ที่มีพฤติกรรมร่วมกันบางส่วน (เช่น implement interface เดียวกัน หรือสืบทอด
จาก parent class เดียวกัน) การเขียน spec แยกกันซ้ำๆ ทุก class จะผิดหลัก DRY ที่เรียนมาตั้งแต่
Part 001 — RSpec จึงมี `shared_examples` ให้แก้ปัญหานี้โดยตรง

### ตัวอย่าง: สอง class ที่มีพฤติกรรม deposit ร่วมกัน

```ruby
# frozen_string_literal: true

# lib/savings_account.rb
class SavingsAccount
  attr_reader :balance

  def initialize(balance = 0)
    @balance = balance
  end

  def deposit(amount)
    @balance += amount
  end
end
```

```ruby
# frozen_string_literal: true

# lib/checking_account.rb
class CheckingAccount
  OVERDRAFT_LIMIT = 500

  attr_reader :balance

  def initialize(balance = 0)
    @balance = balance
  end

  def deposit(amount)
    @balance += amount
  end

  def withdraw(amount)
    raise "เกินวงเงิน overdraft" if amount > balance + OVERDRAFT_LIMIT

    @balance -= amount
  end
end
```

ทั้งสอง class มี method `#deposit` ที่ควรมีพฤติกรรมเหมือนกันทุกประการ — เขียน
`shared_examples` แยกไว้ครั้งเดียว แล้วเรียกใช้ซ้ำได้:

```ruby
# frozen_string_literal: true

require_relative "../lib/savings_account"
require_relative "../lib/checking_account"

RSpec.shared_examples "a depositable account" do
  describe "#deposit" do
    it "เพิ่มยอดเงินตามจำนวนที่ฝากเข้าไป" do
      expect { subject.deposit(100) }.to change { subject.balance }.by(100)
    end

    it "คืนค่ายอดเงินใหม่หลังฝาก" do
      expect(subject.deposit(50)).to eq(subject.balance)
    end
  end
end

RSpec.describe SavingsAccount do
  subject { described_class.new(200) }

  it_behaves_like "a depositable account"
end

RSpec.describe CheckingAccount do
  subject { described_class.new(200) }

  it_behaves_like "a depositable account"
end
```

```bash
rspec spec/savings_and_checking_spec.rb --format documentation
```

```
SavingsAccount
  behaves like a depositable account
    #deposit
      เพิ่มยอดเงินตามจำนวนที่ฝากเข้าไป
      คืนค่ายอดเงินใหม่หลังฝาก

CheckingAccount
  behaves like a depositable account
    #deposit
      เพิ่มยอดเงินตามจำนวนที่ฝากเข้าไป
      คืนค่ายอดเงินใหม่หลังฝาก

Finished in 0.00231 seconds (files took 0.04298 seconds to load)
4 examples, 0 failures
```

โค้ดใน `shared_examples` เขียนครั้งเดียว แต่ถูกรันจริงกับทั้ง `SavingsAccount` และ
`CheckingAccount` เพราะแต่ละ `describe` กำหนด `subject` ของตัวเองไว้แตกต่างกัน — นี่คือเหตุผล
ที่ `described_class`/`subject` (Step 188) สำคัญมาก เพราะ `shared_examples` ต้องเขียนโดย
**ไม่รู้ล่วงหน้าว่า class ปลายทางชื่ออะไร** อ้างอิงผ่าน `subject` เท่านั้น

### ส่งพารามิเตอร์เข้า shared_examples

```ruby
RSpec.shared_examples "a depositable account" do |starting_balance|
  it "เริ่มต้นด้วยยอดเงินที่กำหนด" do
    expect(subject.balance).to eq(starting_balance)
  end
end

RSpec.describe SavingsAccount do
  subject { described_class.new(500) }

  it_behaves_like "a depositable account", 500
end
```

### `it_behaves_like` vs `include_examples`

- **`it_behaves_like "ชื่อ"`** — สร้าง **nested context ใหม่** ครอบ example ที่ share มา
  (ผลลัพธ์จะขึ้นบรรทัด "behaves like ..." แยกต่างหากตามที่เห็นด้านบน) เหมาะกับกรณีทั่วไป
- **`include_examples "ชื่อ"`** — แทรก example เข้าไปในระดับเดียวกับจุดที่เรียก โดยไม่สร้าง
  context ใหม่ครอบ (ผลลัพธ์จะไม่มีบรรทัด "behaves like" คั่น) ใช้เมื่อต้องการให้ผลลัพธ์แบน
  ราบไม่ซ้อนชั้นเพิ่ม

```ruby
RSpec.describe SavingsAccount do
  subject { described_class.new(200) }

  include_examples "a depositable account", 200   # ไม่มี context "behaves like" คั่น
end
```

---

## Step 190: RSpec doubles/mocks — `double`, `instance_double`, `allow`, `expect(...).to receive`

Part 018 แนะนำแนวคิด test double ไว้คร่าวๆ ด้วย `Minitest::Mock` แล้ว หัวข้อนี้จะเจาะลึก
เครื่องมือของ **rspec-mocks** (ไลบรารีที่มากับ gem `rspec` โดยตรง ไม่ต้องติดตั้งเพิ่ม) ซึ่งมี
API ที่ยืดหยุ่นและอ่านง่ายกว่ามาก

### ทำไมต้องใช้ double

สมมติมี class ที่ขึ้นกับบริการภายนอกที่ช้า/ไม่แน่นอน/มีค่าใช้จ่ายจริงตอนเรียก (เช่น
payment gateway, ส่ง SMS, เรียก API ภายนอก) — เราไม่อยากให้ unit test เรียกของจริงทุกครั้ง
ที่รันเทสต์ จึงใช้ **double** เป็น object ปลอมแทนที่ของจริงชั่วคราว

```ruby
# frozen_string_literal: true

class NotificationService
  def notify_low_balance(owner, balance)
    # ในระบบจริงจะยิง SMS/Email ผ่านบริการภายนอกตรงนี้
    puts "[notify] แจ้งเตือน #{owner}: ยอดเงินเหลือ #{balance} บาท"
  end
end
```

### `double` — test double แบบพื้นฐาน (bare double)

```ruby
RSpec.describe "ตัวอย่าง double พื้นฐาน" do
  it "สร้าง double และ stub method ด้วย allow" do
    notifier = double("NotificationService")
    allow(notifier).to receive(:notify_low_balance)

    notifier.notify_low_balance("สมชาย", 50)

    # ไม่มี expectation ใดๆ ล้มเหลว เพราะแค่ stub ไว้เฉยๆ ไม่ได้บังคับว่าต้องถูกเรียก
  end
end
```

`double("NotificationService")` สร้าง object เปล่าที่ **ไม่มี method ใดๆ เลยจนกว่าจะ stub**
ชื่อ `"NotificationService"` เป็นแค่ label ไว้ช่วย debug (จะโผล่ในข้อความ error) ไม่ได้ผูก
กับ class จริงแต่อย่างใด

### `allow(...).to receive(...)` — stub (กำหนดค่าคืนกลับ ไม่บังคับว่าต้องถูกเรียก)

```ruby
notifier = double("NotificationService")
allow(notifier).to receive(:notify_low_balance).and_return(true)

result = notifier.notify_low_balance("สมชาย", 50)
expect(result).to eq(true)
```

### `expect(...).to receive(...)` — message expectation (บังคับว่าต้องถูกเรียก ไม่งั้น fail)

```ruby
RSpec.describe "ตัวอย่าง message expectation" do
  it "fail ถ้าไม่มีการเรียก method ที่คาดหวังไว้จริง" do
    notifier = double("NotificationService")
    expect(notifier).to receive(:notify_low_balance)

    # ตั้งใจไม่เรียก notifier.notify_low_balance เลย เพื่อดูว่า RSpec จับได้จริง
  end
end
```

```
Failures:

  1) ตัวอย่าง message expectation fail ถ้าไม่มีการเรียก method ที่คาดหวังไว้จริง
     Failure/Error: expect(notifier).to receive(:notify_low_balance)

       (Double "NotificationService").notify_low_balance(*(any args))
           expected: 1 time with any arguments
           received: 0 times with any arguments
```

ตรวจสอบ argument และจำนวนครั้งที่ถูกเรียกได้ละเอียดด้วย:

```ruby
notifier = double("NotificationService")

expect(notifier).to receive(:notify_low_balance).with("สมชาย", 50)   # ต้องเรียกด้วย argument นี้เท่านั้น
expect(notifier).to receive(:notify_low_balance).exactly(2).times    # ต้องถูกเรียกพอดี 2 ครั้ง
expect(notifier).to receive(:notify_low_balance).once                # ต้องถูกเรียกพอดี 1 ครั้ง
expect(notifier).to receive(:notify_low_balance).at_least(:once)     # ต้องถูกเรียกอย่างน้อย 1 ครั้ง
```

### Spy pattern — `allow` ก่อน แล้วค่อยตรวจด้วย `have_received` ทีหลัง

```ruby
notifier = double("NotificationService")
allow(notifier).to receive(:notify_low_balance)

notifier.notify_low_balance("สมชาย", 50)

expect(notifier).to have_received(:notify_low_balance).with("สมชาย", 50)
```

รูปแบบนี้เหมาะเมื่ออยากเขียนโค้ดที่ทดสอบก่อนแล้วค่อยเขียน assertion ทีหลัง (อ่านเป็นลำดับ
Arrange-Act-Assert ได้ชัดเจนกว่า `expect(...).to receive` ที่ต้องตั้ง expectation ก่อน action)

### `instance_double` — verifying double (แนะนำให้ใช้แทน `double` เปล่าๆ เสมอในโค้ดจริง)

ปัญหาของ `double` เปล่าคือ **ไม่มีการตรวจสอบใดๆ เลยว่า method ที่ stub ไว้มีอยู่จริงใน class
ต้นแบบหรือไม่** ถ้าวันหนึ่งมีคนไปลบ/เปลี่ยนชื่อ method `notify_low_balance` ใน
`NotificationService` จริง เทสต์ที่ใช้ `double` เปล่าจะยัง **ผ่านหลอกๆ** ต่อไปโดยไม่รู้ตัว
เพราะ double ไม่เคยรู้จัก class จริงเลย

`instance_double(NotificationService)` แก้ปัญหานี้โดยตรวจสอบกับ class จริงทุกครั้งที่ stub:

```ruby
RSpec.describe "instance_double ตรวจสอบ signature จริง" do
  it "stub method ที่มีอยู่จริงได้ตามปกติ" do
    notifier = instance_double(NotificationService)
    allow(notifier).to receive(:notify_low_balance)

    notifier.notify_low_balance("สมชาย", 50)
  end

  it "จับได้ทันทีถ้า stub method ที่ไม่มีอยู่จริงใน class ต้นแบบ" do
    notifier = instance_double(NotificationService)

    expect do
      allow(notifier).to receive(:send_email)   # NotificationService ไม่มี method นี้จริง
    end.to raise_error(/does not implement/)
  end
end
```

`instance_double` ยังตรวจสอบจำนวน argument (arity) ให้ตรงกับ method จริงอีกด้วย ถ้า stub
ด้วยจำนวน argument ที่ไม่ตรงกับ signature จริงของ `notify_low_balance(owner, balance)` เช่น
เรียกด้วย argument เดียว จะถูก error ทันทีเช่นกัน

**กฎการเลือกใช้ในทางปฏิบัติ:** ใช้ `instance_double(RealClass)` แทน `double("ชื่อ")` เปล่าๆ
**เสมอ** เมื่อกำลังปลอมตัวแทน class ที่มีอยู่จริงในระบบ เพราะช่วยจับบั๊กจากการที่ API ของ
dependency เปลี่ยนไปโดยที่เทสต์ไม่รู้ตัว — ใช้ `double` เปล่าๆ เฉพาะกรณีที่ยังไม่มี class จริง
ให้ตรวจสอบ (เช่นเขียนเทสต์นำหน้าการออกแบบ class ตามสไตล์ TDD)

---

## แบบฝึกหัด: เขียน RSpec suite เต็มรูปแบบให้ `BankAccount`

### โจทย์

สร้างไฟล์ `lib/bank_account.rb` ที่มี class `BankAccount` ทำงานดังนี้:

1. `initialize(owner, balance: 0, notifier: nil)` — รับชื่อเจ้าของบัญชี, ยอดเงินเริ่มต้น
   (ค่าเริ่มต้น 0), และ dependency สำหรับแจ้งเตือน (`notifier`, จะเป็น `nil` ก็ได้)
2. `#deposit(amount)` — ฝากเงิน ต้องมากกว่า 0 ไม่เช่นนั้น raise `ArgumentError` คืนค่า `self`
3. `#withdraw(amount)` — ถอนเงิน ต้องมากกว่า 0 ไม่เช่นนั้น raise `ArgumentError`, ถ้ายอดเงิน
   ไม่พอให้ raise `InsufficientFundsError`, ถ้าถอนแล้วยอดเหลือต่ำกว่า `LOW_BALANCE_THRESHOLD`
   (กำหนดเป็น 100) ให้เรียก `notifier.notify_low_balance(owner, balance)` (ถ้ามี notifier)
   คืนค่า `self`

จากนั้นเขียน `spec/bank_account_spec.rb` ทดสอบให้ครอบคลุมทุก branch โดยใช้ `let`, `subject`,
`context` แยกตามสถานะ และ `instance_double` แทนที่ `NotificationService` จริง

### เฉลย

```ruby
# frozen_string_literal: true

# lib/bank_account.rb
class InsufficientFundsError < StandardError; end

class BankAccount
  LOW_BALANCE_THRESHOLD = 100

  attr_reader :owner, :balance, :notifier

  def initialize(owner, balance: 0, notifier: nil)
    @owner = owner
    @balance = balance
    @notifier = notifier
  end

  def deposit(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" unless amount.positive?

    @balance += amount
    self
  end

  def withdraw(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" unless amount.positive?
    raise InsufficientFundsError, "ยอดเงินไม่พอ (มี #{balance} บาท)" if amount > balance

    @balance -= amount
    notifier&.notify_low_balance(owner, balance) if balance < LOW_BALANCE_THRESHOLD

    self
  end
end
```

```ruby
# frozen_string_literal: true

# lib/notification_service.rb
class NotificationService
  def notify_low_balance(owner, balance)
    puts "[notify] แจ้งเตือน #{owner}: ยอดเงินเหลือ #{balance} บาท"
  end
end
```

```ruby
# frozen_string_literal: true

# spec/bank_account_spec.rb
require_relative "../lib/bank_account"
require_relative "../lib/notification_service"

RSpec.describe BankAccount do
  let(:owner) { "สมชาย" }
  let(:notifier) { instance_double(NotificationService) }
  subject(:account) { described_class.new(owner, balance: 500, notifier: notifier) }

  describe "#deposit" do
    context "เมื่อจำนวนเงินถูกต้อง (มากกว่า 0)" do
      it "เพิ่มยอดเงินตามจำนวนที่ฝาก" do
        expect { account.deposit(200) }.to change { account.balance }.from(500).to(700)
      end

      it "คืนค่า self เพื่อให้ chain method ต่อได้" do
        expect(account.deposit(100)).to be(account)
      end
    end

    context "เมื่อจำนวนเงินไม่ถูกต้อง" do
      it "raise ArgumentError เมื่อฝาก 0 บาท" do
        expect { account.deposit(0) }.to raise_error(ArgumentError, "จำนวนเงินต้องมากกว่า 0")
      end

      it "raise ArgumentError เมื่อฝากจำนวนติดลบ" do
        expect { account.deposit(-50) }.to raise_error(ArgumentError)
      end

      it "ไม่เปลี่ยนแปลงยอดเงินเมื่อ raise error" do
        expect { account.deposit(0) }.to raise_error(ArgumentError)
        expect(account.balance).to eq(500)
      end
    end
  end

  describe "#withdraw" do
    context "เมื่อยอดเงินเพียงพอและถอนแล้วยอดยังสูงกว่า threshold" do
      it "หักยอดเงินตามจำนวนที่ถอน" do
        expect { account.withdraw(300) }.to change { account.balance }.by(-300)
      end

      it "ไม่เรียก notifier เลย" do
        expect(notifier).not_to receive(:notify_low_balance)
        account.withdraw(100)
      end

      it "คืนค่า self เพื่อให้ chain method ต่อได้" do
        expect(account.withdraw(100)).to be(account)
      end
    end

    context "เมื่อยอดเงินไม่เพียงพอ" do
      it "raise InsufficientFundsError" do
        expect { account.withdraw(1000) }.to raise_error(InsufficientFundsError, /ยอดเงินไม่พอ/)
      end

      it "ไม่หักยอดเงินเลยเมื่อ raise error" do
        expect { account.withdraw(1000) }.to raise_error(InsufficientFundsError)
        expect(account.balance).to eq(500)
      end
    end

    context "เมื่อถอนจำนวนไม่ถูกต้อง (0 หรือติดลบ)" do
      it "raise ArgumentError" do
        expect { account.withdraw(0) }.to raise_error(ArgumentError)
      end
    end

    context "เมื่อถอนแล้วยอดเงินต่ำกว่า threshold ที่กำหนด" do
      it "เรียก notifier.notify_low_balance ด้วยยอดเงินคงเหลือที่ถูกต้อง" do
        expect(notifier).to receive(:notify_low_balance).with(owner, 50)
        account.withdraw(450)
      end
    end
  end

  describe "ค่าเริ่มต้นตอนสร้างบัญชี" do
    subject(:new_account) { described_class.new("มานี") }

    it { expect(new_account.balance).to eq(0) }
    it { expect(new_account.notifier).to be_nil }
  end
end
```

ทดสอบรัน:

```bash
rspec spec/bank_account_spec.rb --format documentation
```

**ผลลัพธ์ที่คาดหวัง:**

```
BankAccount
  #deposit
    เมื่อจำนวนเงินถูกต้อง (มากกว่า 0)
      เพิ่มยอดเงินตามจำนวนที่ฝาก
      คืนค่า self เพื่อให้ chain method ต่อได้
    เมื่อจำนวนเงินไม่ถูกต้อง
      raise ArgumentError เมื่อฝาก 0 บาท
      raise ArgumentError เมื่อฝากจำนวนติดลบ
      ไม่เปลี่ยนแปลงยอดเงินเมื่อ raise error
  #withdraw
    เมื่อยอดเงินเพียงพอและถอนแล้วยอดยังสูงกว่า threshold
      หักยอดเงินตามจำนวนที่ถอน
      ไม่เรียก notifier เลย
      คืนค่า self เพื่อให้ chain method ต่อได้
    เมื่อยอดเงินไม่เพียงพอ
      raise InsufficientFundsError
      ไม่หักยอดเงินเลยเมื่อ raise error
    เมื่อถอนจำนวนไม่ถูกต้อง (0 หรือติดลบ)
      raise ArgumentError
    เมื่อถอนแล้วยอดเงินต่ำกว่า threshold ที่กำหนด
      เรียก notifier.notify_low_balance ด้วยยอดเงินคงเหลือที่ถูกต้อง
  ค่าเริ่มต้นตอนสร้างบัญชี
    is expected to eq 0
    is expected to be nil

Finished in 0.01246 seconds (files took 0.07614 seconds to load)
14 examples, 0 failures
```

### สิ่งที่ได้ฝึกจากเฉลยนี้

- ใช้ `let`/`subject(:account)` แทนการสร้าง object ซ้ำในทุก `it` block ทำให้แก้ค่าเริ่มต้น
  ได้จากจุดเดียว
- แบ่ง `context` ตาม**สถานะ**ของระบบอย่างชัดเจน (จำนวนเงินถูก/ผิด, ยอดพอ/ไม่พอ, เกิน/ไม่เกิน
  threshold) ทำให้ผลลัพธ์ documentation format อ่านแล้วเข้าใจ business rule ทั้งหมดได้ทันที
  โดยไม่ต้องเปิดโค้ด `lib/bank_account.rb` เลย
- ใช้ `instance_double(NotificationService)` แทนที่จะสร้าง `NotificationService.new` จริง
  เพื่อแยก unit test ของ `BankAccount` ออกจากรายละเอียดการแจ้งเตือนจริง (ไม่ต้องกังวลว่าการ
  รันเทสต์จะไป `puts` แจ้งเตือนจริงๆ ทุกครั้ง)
- ใช้ `expect(notifier).to receive(...).with(...)` ยืนยันว่า business rule "แจ้งเตือนเมื่อ
  ยอดต่ำกว่า threshold" ทำงานถูกต้อง และใช้ `not_to receive(...)` ยืนยันในเคสตรงข้ามว่า
  **ไม่ได้** เรียกแจ้งเตือนโดยไม่จำเป็น (การเทสต์ negative case แบบนี้สำคัญพอๆ กับ positive
  case)
- ใช้ `change { }.from(...).to(...)` และ `.by(...)` แสดงความเปลี่ยนแปลงของยอดเงินได้ชัดเจน
  กว่าการเทียบค่าก่อน/หลังด้วย `eq` สองบรรทัดแยกกัน

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เขียน class `ShoppingCart` ที่มี method `add_item(name, price, quantity: 1)`,
   `remove_item(name)`, `total_price` (คืนผลรวมราคา × จำนวนของทุกชิ้น), และ `empty?` แล้ว
   เขียน RSpec suite ให้ครบทุก branch โดยใช้ `context` แยกตามสถานะอย่างน้อย 3 สถานการณ์:
   ตะกร้าว่างเปล่า, ตะกร้ามีสินค้าชิ้นเดียว, ตะกร้ามีหลายชิ้นที่มีจำนวน (`quantity`) ต่างกัน
   (ใบ้: ใช้ `let` เก็บ cart และใช้ `let!` ถ้าต้องการให้มีสินค้าอยู่ในตะกร้าก่อนทุก example
   ในบาง `context`)

2. เพิ่ม method `apply_discount(percentage)` ให้ `ShoppingCart` ที่ลดราคารวมลงตามเปอร์เซ็นต์
   ที่กำหนด (เช่น `apply_discount(10)` ลดราคารวม 10%) แล้วเขียน `shared_examples` ชื่อ
   `"a discountable total"` ที่ตรวจสอบว่าราคาหลังลดคำนวณถูกต้อง จากนั้นเรียกใช้ shared
   example นี้ทั้งกับ `ShoppingCart` และอีก class หนึ่งที่คุณออกแบบเอง (เช่น `Invoice`) ที่มี
   method `apply_discount` แบบเดียวกัน เพื่อฝึกใช้ `it_behaves_like` ข้าม class จริง

3. สมมติว่า `ShoppingCart#total_price` ต้องเรียกใช้ dependency ภายนอกชื่อ `TaxCalculator`
   ที่มี method `calculate(subtotal:)` คืนค่าภาษีที่ต้องบวกเพิ่ม เขียนเทสต์ที่ใช้
   `instance_double(TaxCalculator)` แทนของจริง พร้อมยืนยันด้วย
   `expect(tax_calculator).to receive(:calculate).with(subtotal: ...)` ว่า `ShoppingCart`
   เรียก `TaxCalculator` ด้วยค่า `subtotal` ที่ถูกต้อง แล้วลองจงใจเปลี่ยนชื่อ method ใน
   `TaxCalculator` จริงเป็นชื่ออื่น สังเกตว่า `instance_double` จับความผิดพลาดนี้ได้ทันทีต่าง
   จากถ้าใช้ `double` เปล่าๆ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปรัชญา BDD ของ RSpec และความต่างเชิงโครงสร้าง/syntax เมื่อเทียบกับ Minitest ที่เรียน
  ใน Part 018 พร้อมรู้ว่าทั้งสองใช้กันจริงจังคู่ขนานในอุตสาหกรรม
- ติดตั้ง RSpec ทั้งแบบ global และผ่าน Bundler, ใช้ `rspec --init` สร้าง `.rspec` และ
  `spec/spec_helper.rb`, เข้าใจธรรมเนียมไฟล์ `*_spec.rb` ใต้โฟลเดอร์ `spec/`
- แยกความต่างเชิงความหมายระหว่าง `describe` (อธิบายสิ่งที่ถูกทดสอบ) กับ `context` (อธิบาย
  เงื่อนไข/สถานะ) แม้ทั้งคู่จะเป็น alias กันทาง technical ก็ตาม
- เปลี่ยนจาก `should` syntax แบบเก่า (monkey-patch) มาใช้ `expect(...).to` syntax มาตรฐาน
  ปัจจุบัน พร้อมแยกแยะ `eq`/`eql`/`equal`(`be`) ได้ถูกต้อง
- ใช้ matcher มาตรฐานที่ใช้บ่อยในงานจริงได้ครบ: `be_truthy`/`be_falsey`, predicate matcher,
  `include`, `match`, `be_within...of`, `raise_error` (block form), `change` (block form)
- ใช้ `let`/`let!` แทน instance variable ใน `before` ได้อย่างเหมาะสม เข้าใจกลไก lazy
  evaluation และ memoization ที่อยู่เบื้องหลัง
- ใช้ `before`/`after` ทั้งแบบ `:each` และ `:context` ได้ถูกต้อง พร้อมรู้ข้อควรระวังเรื่อง
  shared mutable state ข้าม example เมื่อใช้ `before(:context)` ผิดวิธี
- ใช้ `subject`/`described_class` ลดการเขียนโค้ดซ้ำ และใช้ one-liner syntax `is_expected.to`
  กับเทสต์สั้นๆ ได้
- เขียน `shared_examples`/`it_behaves_like` เพื่อแชร์ spec เดียวกันข้ามหลาย class ที่มี
  พฤติกรรมร่วมกันได้ โดยไม่ผิดหลัก DRY
- ใช้ `double`, `instance_double`, `allow(...).to receive`, `expect(...).to receive` แยก
  unit test ออกจาก dependency ภายนอกได้ และเข้าใจว่าทำไม `instance_double` (verifying
  double) ปลอดภัยกว่า `double` เปล่าๆ ในโค้ดจริงเสมอ
- ลงมือเขียน RSpec suite เต็มรูปแบบให้ `BankAccount` ที่รวมทุกเทคนิคใน Part นี้: `let`,
  `subject`, `context` แยกตามสถานะ, matcher หลากหลายชนิด, และ `instance_double` สำหรับ
  dependency ภายนอก

**ต่อไป (Part 020):** เราจะเรียนเรื่อง **Rake และ Rakefile** สำหรับสร้าง automation task
ของโปรเจกต์ Ruby (เช่น task รัน test, task เตรียมข้อมูล) จากนั้นนำทุกอย่างที่เรียนมาตลอด
เฟส 2 ทั้งหมด — exception handling, file I/O, Enumerable ขั้นสูง, Struct/Data,
metaprogramming, SOLID/design pattern, gem/Bundler, Minitest, และ RSpec ใน Part นี้ — มา
รวมกันสร้าง **โปรเจกต์ปิดเฟส: Todo CLI** เครื่องมือจัดการ to-do list แบบ command line ที่
เขียนด้วย Ruby ล้วนพร้อมชุดเทสต์ครบถ้วน เป็นการปิดท้ายเฟส 2 (Ruby Deep Dive) ก่อนจะเข้าสู่
เฟส 3 ที่เริ่มเรียน Rails framework จริงจัง
