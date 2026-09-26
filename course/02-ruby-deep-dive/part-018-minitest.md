# Part 018: Testing เบื้องต้นด้วย Minitest — Unit Test, Assertion, และ Test Doubles

> **Step ครอบคลุมใน Part นี้:** Step 171–180
> **ระดับ:** กลาง (ควรผ่าน Part 001–017 มาก่อน โดยเฉพาะ Class, Module, Exception handling,
> และ Gem/Bundler)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Minitest 5.x (เป็น default gem ที่มากับ Ruby อยู่แล้ว
> ไม่ต้องติดตั้งเพิ่ม)

## สารบัญของ Part นี้

- Step 171: ทำไมต้องเขียน Automated Test — Regression Safety, Documentation, Design Feedback
- Step 172: ติดตั้งความเข้าใจ Minitest และเขียนไฟล์เทสต์แรก
- Step 173: Assertion หลักที่ใช้บ่อยที่สุด
- Step 174: `refute` และ assertion สำหรับตัวเลขทศนิยม/collection
- Step 175: `setup` และ `teardown` — เตรียมและล้างสภาพแวดล้อมของแต่ละเทสต์
- Step 176: จัดระเบียบเทสต์หลายไฟล์ และรันพร้อมกัน
- Step 177: Testing กับ Dependency — แนวคิด Test Doubles และการทำ Stub ด้วย Ruby ล้วน
- Step 178: `Minitest::Mock` — Mock และ Verify แบบเป็นทางการ
- Step 179: ข้อตกลงการตั้งชื่อเทสต์ หลักการ one-assertion-per-test และ Minitest::Spec DSL
- Step 180: รันเทสต์ด้วย Rakefile และแบบฝึกหัดโปรเจกต์ ShoppingCart

---

## Step 171: ทำไมต้องเขียน Automated Test — Regression Safety, Documentation, Design Feedback

ตลอด 17 Part ที่ผ่านมา เวลาจะตรวจสอบว่าโค้ดทำงานถูกต้องหรือไม่ เรามักจะเปิด `irb`/`pry`
ขึ้นมาทดลองเรียก method ดูผลลัพธ์ด้วยตา — วิธีนี้เรียกว่า **manual testing** ซึ่งมีปัญหา
สำคัญเมื่อโปรเจกต์โตขึ้น:

```ruby
# frozen_string_literal: true

# bank_account.rb
class BankAccount
  class InsufficientFundsError < StandardError; end

  attr_reader :balance

  def initialize(balance: 0)
    @balance = balance
  end

  def deposit(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" if amount <= 0

    @balance += amount
  end

  def withdraw(amount)
    raise InsufficientFundsError, "ยอดเงินไม่พอ" if amount > balance

    @balance -= amount
  end
end
```

ลองนึกภาพว่าเรากำลังพัฒนา `BankAccount` แล้วอยากตรวจสอบว่า `deposit` และ `withdraw`
ทำงานถูกต้อง เราอาจเปิด `irb` แล้วพิมพ์:

```irb
irb> require_relative "bank_account"
irb> account = BankAccount.new(balance: 100)
irb> account.deposit(50)
=> 150
irb> account.withdraw(30)
=> 120
irb> account.withdraw(1000)
# BankAccount::InsufficientFundsError (ยอดเงินไม่พอ)
```

วิธีนี้ใช้ได้ผลตอนนั้น แต่มีปัญหาสามข้อใหญ่ๆ:

1. **ไม่มี Regression Safety** — สมมติอีก 2 เดือนต่อมา เพื่อนร่วมทีมแก้โค้ด `withdraw`
   เพื่อเพิ่มฟีเจอร์ค่าธรรมเนียม แล้วบังเอิญทำให้ `deposit` พังโดยไม่ตั้งใจ (regression)
   ถ้าไม่มีเทสต์อัตโนมัติ ไม่มีใครรู้จนกว่าจะไปเจอ bug ใน production
2. **ไม่ใช่ Documentation ที่เชื่อถือได้** — โค้ดที่พิมพ์ใน `irb` หายไปทันทีที่ปิดหน้าต่าง
   ไม่มีใครย้อนมาดูได้ว่า "เคยทดสอบ case อะไรไปแล้วบ้าง"
3. **ไม่ได้ทำให้ออกแบบโค้ดดีขึ้น** — Manual test ทำให้เราขี้เกียจทดสอบ edge case ที่พิมพ์
   ยาก (เช่น การใส่ dependency ปลอมเข้าไปแทนของจริง) ผลคือดีไซน์ของ class มักจะผูก
   (coupled) กับ dependency ภายนอกแน่นเกินไปโดยไม่รู้ตัว

**Automated Testing** คือการเขียนโค้ดอีกชุดหนึ่งที่ทำหน้าที่ตรวจสอบโค้ดหลักของเราแทนที่จะ
ใช้สายตา แนวคิดพื้นฐานที่สุดคือ "เขียนเงื่อนไขที่คาดหวัง แล้วให้โปรแกรม `raise` เมื่อผลลัพธ์
ไม่ตรง":

```ruby
# frozen_string_literal: true

# ทดลองเขียน "เทสต์" แบบมือ ก่อนรู้จัก Minitest — เพื่อให้เห็นแก่นแท้ของมัน
require_relative "bank_account"

account = BankAccount.new(balance: 100)
account.deposit(50)

raise "FAILED: expected 150 got #{account.balance}" unless account.balance == 150

puts "PASSED: deposit ทำงานถูกต้อง"
```

```bash
ruby manual_test.rb
# => PASSED: deposit ทำงานถูกต้อง
```

นี่คือแก่นแท้ของทุก testing framework: **เขียนสิ่งที่คาดหวัง (expectation) แล้วเปรียบเทียบ
กับผลลัพธ์จริง ถ้าไม่ตรงให้แจ้งเตือนทันที** สิ่งที่ framework อย่าง Minitest หรือ RSpec
เพิ่มเข้ามาคือ:

- รวบรวมผลการทดสอบทั้งหมด (แม้บาง test จะ fail ก็ยังรันตัวอื่นต่อ ไม่หยุดทั้งโปรแกรม)
- รายงานผลสรุปที่อ่านง่าย (กี่ผ่าน กี่ไม่ผ่าน อยู่บรรทัดไหน)
- ให้ syntax สั้นกระชับสำหรับเขียนเงื่อนไขเปรียบเทียบ (assertion)
- จัดกลุ่มเทสต์เป็นหมวดหมู่ พร้อมกลไก setup/teardown

**สามเหตุผลหลักที่ทีมพัฒนามืออาชีพเขียน automated test:**

| เหตุผล | อธิบาย |
|--------|--------|
| **Regression Safety** | เมื่อแก้โค้ดเก่าหรือเพิ่มฟีเจอร์ใหม่ รันเทสต์ทั้งหมดอีกครั้งเพื่อยืนยันว่าไม่ได้ทำของเดิมพัง ยิ่งโปรเจกต์ใหญ่ ยิ่งสำคัญ |
| **Documentation ที่ไม่มีวันตกยุค** | เทสต์คือตัวอย่างการใช้งานโค้ดที่ "รันได้จริงเสมอ" ต่างจาก comment หรือเอกสารที่อาจล้าสมัยไม่ตรงกับโค้ดปัจจุบัน |
| **Design Feedback** | โค้ดที่ "เทสต์ยาก" มักเป็นสัญญาณว่าออกแบบไม่ดี (เช่น ผูกกับ dependency ภายนอกแน่นเกินไป, method ทำหลายอย่างเกินไป) การเขียนเทสต์บังคับให้เราคิดเรื่อง dependency injection และ single responsibility ตั้งแต่แรก |

ตลอด Part นี้เราจะใช้ **Minitest** — testing framework ที่มากับ Ruby standard library
โดยตรง (เป็น default gem ไม่ต้อง `gem install` เพิ่ม) และเป็นพื้นฐานที่ Rails เองก็ใช้ภายใน
สำหรับสร้าง test suite เริ่มต้นของโปรเจกต์ใหม่ (`rails new` ที่ไม่ได้เลือก `--skip-test`
จะสร้าง test suite แบบ Minitest มาให้)

---

## Step 172: ติดตั้งความเข้าใจ Minitest และเขียนไฟล์เทสต์แรก

เพราะ Minitest เป็น default gem ที่มากับ Ruby ตั้งแต่ติดตั้ง เราตรวจสอบได้ทันทีโดยไม่ต้อง
ติดตั้งอะไรเพิ่ม:

```bash
gem list minitest
# *** LOCAL GEMS ***
# minitest (5.25.1)

ruby -e 'require "minitest/autorun"; puts Minitest::VERSION'
# => 5.25.1
```

> **หมายเหตุ:** เวอร์ชันที่แสดงอาจต่างกันไปตามเวอร์ชัน Ruby ที่ติดตั้ง แต่ตราบใดที่ใช้
> Ruby 3.x ขึ้นไป จะมี Minitest 5.x ติดมาด้วยเสมอ

### โครงสร้างไฟล์เทสต์ขั้นต่ำ

ไฟล์เทสต์ Minitest ต้องมีสามส่วนหลัก: (1) `require "minitest/autorun"`,
(2) class ที่สืบทอดจาก `Minitest::Test`, (3) method ที่ขึ้นต้นด้วย `test_`

สร้างไฟล์ `calculator.rb`:

```ruby
# frozen_string_literal: true

# calculator.rb
class Calculator
  def add(a, b)
    a + b
  end

  def subtract(a, b)
    a - b
  end

  def multiply(a, b)
    a * b
  end

  def divide(a, b)
    raise ZeroDivisionError, "หารด้วยศูนย์ไม่ได้" if b.zero?

    a / b.to_f
  end
end
```

สร้างไฟล์ `calculator_test.rb` ในโฟลเดอร์เดียวกัน:

```ruby
# frozen_string_literal: true

# calculator_test.rb
require "minitest/autorun"
require_relative "calculator"

class CalculatorTest < Minitest::Test
  def setup
    @calculator = Calculator.new
  end

  def test_add_returns_sum_of_two_numbers
    assert_equal 5, @calculator.add(2, 3)
  end

  def test_subtract_returns_difference
    assert_equal 1, @calculator.subtract(3, 2)
  end

  def test_divide_by_zero_raises_error
    assert_raises(ZeroDivisionError) do
      @calculator.divide(10, 0)
    end
  end
end
```

รันด้วยคำสั่ง:

```bash
ruby calculator_test.rb
```

ผลลัพธ์ที่ควรเห็น:

```
Run options: --seed 47821

# Running:

...

Finished in 0.001532s, 1958.7 runs/s, 1958.7 assertions/s.

3 runs, 3 assertions, 0 failures, 0 errors, 0 skips
```

**อธิบาย:**

- `require "minitest/autorun"` — โหลด Minitest และตั้งค่าให้รันเทสต์ทั้งหมดโดยอัตโนมัติ
  เมื่อโปรแกรม (process) จบการทำงาน ไม่ต้องเขียนโค้ดสั่งรันเอง
- `Minitest::Test` — base class ของทุก test class ให้ method อย่าง `assert_equal`,
  `assert_raises` มาให้ใช้
- `setup` — method พิเศษที่ Minitest เรียกอัตโนมัติ **ก่อน** ทุก `test_` method
  (รายละเอียดเต็มใน Step 175)
- แต่ละ method ที่ขึ้นต้นด้วย `test_` จะถูก Minitest มองว่าเป็นหนึ่งเทสต์ (test case)
  โดยอัตโนมัติผ่านกลไก `method_missing`/reflection ภายใน — ถ้าไม่ขึ้นต้นด้วย `test_`
  จะไม่ถูกรัน
- แต่ละจุด (`.`) ใน output แทนหนึ่งเทสต์ที่ผ่าน — ถ้ามีเทสต์ fail จะแสดง `F`
  ถ้ามี error (exception ที่ไม่คาดคิด) จะแสดง `E`
- บรรทัดสุดท้าย `3 runs, 3 assertions` หมายถึงมี 3 test method ถูกรัน และมี assertion
  ทั้งหมด 3 ครั้ง (นับทุกครั้งที่เรียก `assert_*`)

ลองทำให้เทสต์ fail ดูเพื่อดูว่า error message มีประโยชน์แค่ไหน แก้ `test_add_returns_sum_of_two_numbers`
ชั่วคราวเป็น `assert_equal 999, @calculator.add(2, 3)`:

```
Run options: --seed 12345

# Running:

F..

Finished in 0.002011s, 1492.3 runs/s, 1492.3 assertions/s.

  1) Failure:
CalculatorTest#test_add_returns_sum_of_two_numbers [calculator_test.rb:10]:
Expected: 999
  Actual: 5

3 runs, 3 assertions, 1 failures, 0 errors, 0 skips
```

Minitest บอกชื่อเทสต์ที่ fail, เลขบรรทัด, ค่าที่คาดหวัง (`Expected`) กับค่าจริง
(`Actual`) ครบถ้วน — นี่คือเหตุผลที่ automated test มีประโยชน์กว่าการดูผลด้วยตาใน `irb`
มาก เวลาผิดพลาดจะรู้ทันทีว่าผิดตรงไหน

---

## Step 173: Assertion หลักที่ใช้บ่อยที่สุด

Minitest มี assertion method ให้เลือกใช้หลายสิบตัว ในทางปฏิบัติจะใช้ไม่กี่ตัวบ่อยที่สุด
มาดูผ่านตัวอย่าง class `Product`:

```ruby
# frozen_string_literal: true

# product.rb
class Product
  attr_reader :name, :price, :tags

  def initialize(name:, price:, tags: [])
    @name = name
    @price = price
    @tags = tags
  end

  def discounted_price(percent)
    price - (price * percent / 100.0)
  end

  def on_sale?
    tags.include?("sale")
  end
end
```

```ruby
# frozen_string_literal: true

# product_test.rb
require "minitest/autorun"
require_relative "product"

class ProductTest < Minitest::Test
  def setup
    @product = Product.new(name: "เสื้อยืด", price: 300, tags: ["clothing", "sale"])
  end

  def test_name_is_set_correctly
    assert_equal "เสื้อยืด", @product.name
  end

  def test_product_is_instance_of_product_class
    assert_instance_of Product, @product
  end

  def test_price_is_a_numeric_value
    assert_kind_of Numeric, @product.price
  end

  def test_product_responds_to_discounted_price
    assert_respond_to @product, :discounted_price
  end

  def test_discounted_price_calculation
    assert_equal 270.0, @product.discounted_price(10)
  end

  def test_product_is_truthy_on_sale
    assert @product.on_sale?
  end

  def test_tags_include_clothing
    assert_includes @product.tags, "clothing"
  end

  def test_name_contains_substring_yeud
    assert_match(/ยืด/, @product.name)
  end

  def test_no_description_attribute_present
    assert_nil @product.instance_variable_get(:@description)
  end
end
```

```bash
ruby product_test.rb
# => 9 runs, 9 assertions, 0 failures, 0 errors, 0 skips
```

**สรุป assertion ที่ใช้ในตัวอย่างนี้:**

| Assertion | ใช้เมื่อ |
|-----------|----------|
| `assert(bool)` | ตรวจว่าค่าเป็น truthy (ไม่ใช่ `nil`/`false`) |
| `assert_equal(expected, actual)` | ตรวจว่าสองค่าเท่ากันด้วย `==` |
| `assert_nil(value)` | ตรวจว่าค่าเป็น `nil` พอดี |
| `assert_instance_of(klass, obj)` | ตรวจว่า `obj.class == klass` เป๊ะๆ (ไม่นับ subclass) |
| `assert_kind_of(klass, obj)` | ตรวจว่า `obj.is_a?(klass)` (นับ subclass และ module ที่ include ด้วย) |
| `assert_respond_to(obj, :method_name)` | ตรวจว่า object มี method นั้นให้เรียกใช้ (duck typing check) |
| `assert_includes(collection, item)` | ตรวจว่า Array/Hash/String มีสมาชิกนั้นอยู่ |
| `assert_match(regex, string)` | ตรวจว่า string ตรงกับ regular expression |
| `assert_raises(ErrorClass) { block }` | ตรวจว่าโค้ดใน block ทำให้เกิด exception ชนิดที่ระบุ |

**ข้อควรระวังเรื่อง `assert_equal`:** อาร์กิวเมนต์แรกเสมอคือ **ค่าที่คาดหวัง (expected)**
อาร์กิวเมนต์ที่สองคือ **ค่าจริงที่ได้จากการรันโค้ด (actual)** — เขียนสลับกันได้ผลลัพธ์
pass/fail เหมือนเดิม (เพราะ `==` สมมาตร) แต่ error message เวลา fail จะสลับ Expected/Actual
ทำให้อ่านสับสน จึงควรจำอันดับนี้ให้แม่น: `assert_equal(expected, actual)`

`assert_raises` ยังคืนค่า exception object ที่จับได้ ทำให้ตรวจสอบรายละเอียดต่อได้ เช่น
ข้อความ error:

```ruby
def test_divide_by_zero_error_has_correct_message
  calculator = Calculator.new
  error = assert_raises(ZeroDivisionError) do
    calculator.divide(10, 0)
  end
  assert_equal "หารด้วยศูนย์ไม่ได้", error.message
end
```

---

## Step 174: `refute` และ Assertion สำหรับตัวเลขทศนิยม/Collection

ทุก assertion ที่ขึ้นต้นด้วย `assert_` มีคู่ตรงข้ามที่ขึ้นต้นด้วย `refute_` เสมอ
(ยกเว้น `assert` เฉยๆ ที่คู่คือ `refute` เฉยๆ) ใช้เมื่อต้องการยืนยันว่า **บางอย่างไม่เป็นจริง**

```ruby
# frozen_string_literal: true

# product_refute_test.rb
require "minitest/autorun"
require_relative "product"

class ProductRefuteTest < Minitest::Test
  def setup
    @regular_product = Product.new(name: "กระเป๋า", price: 500, tags: ["bag"])
  end

  def test_regular_product_is_not_on_sale
    refute @regular_product.on_sale?
  end

  def test_tags_do_not_include_sale
    refute_includes @regular_product.tags, "sale"
  end

  def test_name_is_not_nil
    refute_nil @regular_product.name
  end

  def test_price_is_not_zero
    refute_equal 0, @regular_product.price
  end

  def test_tags_are_not_empty
    refute_empty @regular_product.tags
  end

  def test_name_does_not_contain_digits
    refute_match(/\d/, @regular_product.name)
  end
end
```

**เหตุผลที่มี `refute` แยกจาก `assert !condition`:** ถ้าเขียน `assert(!obj.on_sale?)`
เวลา fail จะได้ error message ที่ไม่มีประโยชน์ (แค่บอกว่า "Expected false to be truthy")
แต่ `refute(obj.on_sale?)` ถูกออกแบบมาให้ error message อ่านง่ายกว่าและสื่อเจตนาชัดเจนกว่า
ว่า "เราคาดหวังว่าค่านี้จะเป็น falsy"

### เปรียบเทียบตัวเลขทศนิยม: ทำไม `assert_equal` ถึงอันตราย

ตัวเลขทศนิยม (`Float`) ในคอมพิวเตอร์เก็บด้วยระบบ binary floating point ซึ่งมีค่าคลาด
เคลื่อนเล็กน้อยเสมอ ทำให้การเทียบเท่ากันตรงๆ ด้วย `==` อาจได้ผลลัพธ์ที่ไม่คาดคิด:

```ruby
# frozen_string_literal: true

# float_math_test.rb
require "minitest/autorun"
require_relative "product"

class FloatMathTest < Minitest::Test
  def test_naive_equality_fails_due_to_floating_point
    # 0.1 + 0.2 ไม่เท่ากับ 0.3 เป๊ะๆ ในระบบ floating point!
    refute_equal 0.3, 0.1 + 0.2
  end

  def test_correct_way_to_compare_floats
    # assert_in_delta ยอมให้มีค่าคลาดเคลื่อนได้ในขอบเขตที่กำหนด (0.0001)
    assert_in_delta 0.3, 0.1 + 0.2, 0.0001
  end

  def test_discounted_price_within_acceptable_range
    product = Product.new(name: "รองเท้า", price: 999, tags: [])
    assert_in_delta 899.1, product.discounted_price(10), 0.01
  end
end
```

```bash
ruby float_math_test.rb
# => 3 runs, 3 assertions, 0 failures, 0 errors, 0 skips
```

**อธิบาย:**

- `0.1 + 0.2` ใน Ruby จริงๆ แล้วได้ `0.30000000000000004` ไม่ใช่ `0.3` เป๊ะ (ลองพิมพ์ใน
  `irb` ดูได้) นี่ไม่ใช่บั๊กของ Ruby แต่เป็นข้อจำกัดของมาตรฐาน IEEE 754 floating point
  ที่ทุกภาษาโปรแกรมมิ่งใช้ร่วมกัน
- `assert_in_delta(expected, actual, delta)` ตรวจว่า `(expected - actual).abs <= delta`
  แทนการเทียบเท่ากันตรงๆ จึงเหมาะกับการเทียบ Float เสมอ
- กฎปฏิบัติ: **ห้ามใช้ `assert_equal` กับ Float ที่มาจากการคำนวณ** ให้ใช้
  `assert_in_delta` แทนเสมอ (ยกเว้นกรณี Float ที่เป็นค่าคงที่ตรงๆ ไม่ผ่านการคำนวณ
  ซับซ้อน เช่น `assert_equal 0.0, balance` หลัง initialize)

Assertion อื่นที่ใช้บ่อยกับ collection: `assert_empty` / `refute_empty` ตรวจว่า
`.empty?` เป็น `true`/`false` และให้ error message ที่ดีกว่าการเช็ค `.size == 0` เอง

---

## Step 175: `setup` และ `teardown` — เตรียมและล้างสภาพแวดล้อมของแต่ละเทสต์

หลายเทสต์ต้องการสภาพแวดล้อมเริ่มต้นที่เหมือนกัน (เช่น object ตัวใหม่ที่ยังไม่ถูกแก้ไข)
และบางเทสต์ต้องทำความสะอาดทรัพยากรหลังใช้งาน (เช่น ไฟล์ชั่วคราว, การเชื่อมต่อ) Minitest
มี hook สองตัวสำหรับเรื่องนี้: `setup` และ `teardown`

```ruby
# frozen_string_literal: true

# logger.rb
class Logger
  attr_reader :path

  def initialize(path)
    @path = path
    File.write(@path, "") unless File.exist?(@path)
  end

  def log(message)
    File.open(@path, "a") { |f| f.puts(message) }
  end

  def lines
    File.readlines(@path, chomp: true)
  end
end
```

```ruby
# frozen_string_literal: true

# logger_test.rb
require "minitest/autorun"
require_relative "logger"

class LoggerTest < Minitest::Test
  def setup
    # object_id ทำให้แต่ละเทสต์ได้ไฟล์คนละชื่อ ป้องกันการชนกันถ้ารันหลายเทสต์พร้อมกัน
    @path = "test_log_#{object_id}.txt"
    @logger = Logger.new(@path)
  end

  def teardown
    File.delete(@path) if File.exist?(@path)
  end

  def test_log_writes_a_line_to_the_file
    @logger.log("เริ่มระบบ")
    assert_equal ["เริ่มระบบ"], @logger.lines
  end

  def test_log_appends_multiple_lines_in_order
    @logger.log("บรรทัดที่ 1")
    @logger.log("บรรทัดที่ 2")
    assert_equal ["บรรทัดที่ 1", "บรรทัดที่ 2"], @logger.lines
  end

  def test_new_logger_starts_with_empty_file
    assert_empty @logger.lines
  end
end
```

```bash
ruby logger_test.rb
# => 3 runs, 3 assertions, 0 failures, 0 errors, 0 skips

# ตรวจสอบว่าไฟล์ทดสอบไม่หลงเหลืออยู่จริง
ls test_log_*.txt
# => ls: cannot access 'test_log_*.txt': No such file or directory
```

**อธิบายลำดับการทำงาน:**

1. Minitest สร้าง **instance ใหม่** ของ `LoggerTest` สำหรับ**ทุกๆ** `test_` method
   (ไม่ใช่ instance เดียวรันทุกเทสต์) — นี่คือเหตุผลที่ `@logger` ในแต่ละเทสต์ไม่ปนกัน
2. ก่อนแต่ละ `test_` method จะรัน จะเรียก `setup` ก่อนเสมอ (สร้าง `@logger`, `@path` ใหม่)
3. หลังจาก `test_` method นั้นรันเสร็จ (ไม่ว่าจะ pass, fail หรือ error) จะเรียก
   `teardown` เสมอ (ลบไฟล์ทิ้ง)
4. ลำดับนี้ทำให้แต่ละเทสต์**เป็นอิสระจากกันโดยสมบูรณ์ (test isolation)** — รันเทสต์ไหน
   ก่อนหลัง หรือรันแค่เทสต์เดียว ก็ได้ผลลัพธ์เหมือนกันเสมอ ไม่มี state หลงเหลือข้ามเทสต์

> **ข้อควรระวัง:** ถ้า `teardown` ไม่ทำงาน (เช่นเทสต์ crash รุนแรงจน process ตาย)
> ไฟล์ชั่วคราวอาจหลงเหลือ ในการเขียนโค้ด production จริงมักใช้ `Dir.mktmpdir` ที่ระบบ
> จัดการลบให้อัตโนมัติ แทนการสร้างไฟล์เองแบบในตัวอย่างนี้ (ตัวอย่างนี้ทำให้เข้าใจง่ายก่อน)

---

## Step 176: จัดระเบียบเทสต์หลายไฟล์ และรันพร้อมกัน

เมื่อโปรเจกต์มีหลาย class ธรรมเนียมปฏิบัติมาตรฐานของ Ruby community คือแยกโค้ดจริงไว้ใน
`lib/` และเทสต์ไว้ใน `test/` โดยตั้งชื่อไฟล์เทสต์ให้ตรงกับไฟล์จริงบวก `_test.rb` ต่อท้าย:

```
my_project/
├── lib/
│   ├── calculator.rb
│   └── product.rb
├── test/
│   ├── test_helper.rb
│   ├── calculator_test.rb
│   └── product_test.rb
└── Rakefile
```

สร้างไฟล์กลาง `test/test_helper.rb` ที่ทุกไฟล์เทสต์ `require` ก่อนเสมอ เพื่อรวม
การตั้งค่าที่ใช้ร่วมกัน:

```ruby
# frozen_string_literal: true

# test/test_helper.rb
$LOAD_PATH.unshift(File.expand_path("../lib", __dir__))
require "minitest/autorun"
```

จากนั้นแต่ละไฟล์เทสต์เปลี่ยนมา `require_relative "test_helper"` แทนการ
`require "minitest/autorun"` ตรงๆ:

```ruby
# frozen_string_literal: true

# test/calculator_test.rb
require_relative "test_helper"
require "calculator"

class CalculatorTest < Minitest::Test
  def setup
    @calculator = Calculator.new
  end

  def test_add_returns_sum_of_two_numbers
    assert_equal 5, @calculator.add(2, 3)
  end
end
```

```ruby
# frozen_string_literal: true

# test/product_test.rb
require_relative "test_helper"
require "product"

class ProductTest < Minitest::Test
  def test_name_is_set_correctly
    product = Product.new(name: "เสื้อยืด", price: 300)
    assert_equal "เสื้อยืด", product.name
  end
end
```

### รันหลายไฟล์เทสต์พร้อมกัน

```bash
# ระบุไฟล์ทีละไฟล์
ruby -Ilib -Itest test/calculator_test.rb test/product_test.rb

# หรือใช้ Dir.glob หาไฟล์ที่ลงท้ายด้วย _test.rb ทั้งหมดแล้ว require มารวมกัน
ruby -Ilib -Itest -e 'Dir.glob("test/**/*_test.rb").each { |f| require File.expand_path(f) }'
```

ผลลัพธ์จะรวมเทสต์จากทุกไฟล์เข้าเป็นรายงานเดียว:

```
Run options: --seed 33021

# Running:

....

Finished in 0.003211s, 1245.7 runs/s, 1245.7 assertions/s.

4 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

**อธิบาย:**

- `-Ilib -Itest` คือ flag ของคำสั่ง `ruby` ที่เพิ่มโฟลเดอร์ `lib/` และ `test/` เข้าไปใน
  `$LOAD_PATH` ทำให้ `require "calculator"` (ไม่ต้องมี `./` หรือ path เต็ม) หาไฟล์เจอ
- การรันหลายไฟล์ `.rb` ที่แต่ละไฟล์มี `require "minitest/autorun"` (ผ่าน
  `test_helper.rb`) พร้อมกันในคำสั่งเดียว จะไม่ทำให้เทสต์รันซ้ำหรือชนกัน เพราะ
  `require` จะโหลดไฟล์ซ้ำแค่ครั้งเดียวเสมอ (แม้ `test_helper.rb` จะถูก require จาก
  หลายไฟล์) และ `minitest/autorun` ฉลาดพอที่จะรวบรวมทุก test class ที่ประกาศไว้ในทุก
  ไฟล์ที่ถูกโหลด แล้วรันพร้อมกันตอนจบโปรแกรมครั้งเดียว
- วิธีพิมพ์คำสั่งยาวๆ แบบนี้ทุกครั้งไม่สะดวก — ใน Step 180 เราจะใช้ **Rake** เพื่อสร้าง
  คำสั่งสั้นๆ อย่าง `rake test` แทน

---

## Step 177: Testing กับ Dependency — แนวคิด Test Doubles และการทำ Stub ด้วย Ruby ล้วน

ในโลกจริง class มักมี dependency ที่เชื่อมต่อกับสิ่งภายนอก เช่น payment gateway, API,
ฐานข้อมูล ซึ่งมีปัญหาเวลาเขียนเทสต์: ช้า, ไม่แน่นอน (flaky), มีค่าใช้จ่ายจริงทุกครั้งที่เรียก
หรือแม้แต่ไม่มีอยู่จริงในเครื่อง dev

```ruby
# frozen_string_literal: true

# payment_gateway.rb — ของจริงที่เรียก API ภายนอก (ช้า, ไม่แน่นอน, อาจมีค่าธรรมเนียมจริง)
class PaymentGateway
  def charge(amount)
    # สมมติว่าตรงนี้ยิง HTTP request ไปหา payment provider จริง
    # เช่น Stripe, Omise ฯลฯ — ใช้เวลาหลายร้อย ms และต้องมี network/API key จริง
    raise NotImplementedError, "ต้องเชื่อมต่อ payment provider จริงเท่านั้น"
  end
end
```

```ruby
# frozen_string_literal: true

# order_processor.rb
class OrderProcessor
  class PaymentFailedError < StandardError; end

  def initialize(payment_gateway:)
    @payment_gateway = payment_gateway
  end

  def process(amount)
    raise ArgumentError, "จำนวนเงินต้องมากกว่า 0" if amount <= 0

    success = @payment_gateway.charge(amount)
    raise PaymentFailedError unless success

    "ชำระเงินสำเร็จ #{amount} บาท"
  end
end
```

สังเกตว่า `OrderProcessor.new` รับ `payment_gateway:` เป็น keyword argument แทนที่จะ
สร้าง `PaymentGateway.new` เองภายใน — เทคนิคนี้เรียกว่า **Dependency Injection**
ทำให้เราสามารถ "สอดของปลอม" เข้าไปแทนของจริงตอนเทสต์ได้ ถ้า `OrderProcessor` สร้าง
`PaymentGateway` เองภายใน เราจะไม่มีทางเทสต์ได้เลยโดยไม่เรียก API จริง

### คำศัพท์: Test Doubles มีกี่แบบ

**Test Double** คือชื่อเรียกรวมของ "ของปลอม" ทุกชนิดที่ใช้แทน dependency จริงตอนเทสต์
(มาจากคำว่า "stunt double" ในวงการภาพยนตร์) แบ่งเป็น 5 แบบหลักตามพฤติกรรม:

| ชนิด | พฤติกรรม |
|------|----------|
| **Dummy** | object ปลอมที่ไม่ได้ถูกเรียกใช้จริง แค่ต้องส่งผ่านเข้าไปเพื่อให้ signature ตรง (เช่น `nil` ที่ method ไม่แตะต้อง) |
| **Stub** | คืนค่าคงที่ที่กำหนดไว้ล่วงหน้าเสมอ ไม่สนใจว่าถูกเรียกกี่ครั้งหรือด้วย argument อะไร ไม่มีการตรวจสอบ (verify) ว่าถูกเรียกหรือไม่ |
| **Mock** | คล้าย stub แต่มาพร้อม "ความคาดหวัง" (expectation) ล่วงหน้าว่าต้องถูกเรียกด้วย argument อะไร กี่ครั้ง แล้ว**ตรวจสอบ (verify)** ทีหลังว่าเป็นไปตามนั้นจริง ถ้าไม่ตรงจะทำให้เทสต์ fail |
| **Spy** | คล้าย stub แต่บันทึกการถูกเรียกไว้ ทำให้เราถามย้อนหลังได้ว่า "ถูกเรียกไปแล้วกี่ครั้ง ด้วย argument อะไร" โดยไม่ต้องกำหนดความคาดหวังไว้ล่วงหน้า |
| **Fake** | มีการทำงานจริงแบบง่ายๆ ใช้แทนของจริงได้ทั้งหมด แต่ไม่เหมาะกับ production (เช่น in-memory database แทน PostgreSQL จริงตอนเทสต์) |

### ทำ Stub ด้วย Ruby ธรรมดา ไม่ต้องใช้ gem ใดๆ

จุดเด่นของ Ruby คือ **duck typing** — `OrderProcessor` ไม่สนใจว่า `payment_gateway`
เป็น class อะไร ขอแค่มี method `charge` ให้เรียกก็พอ เราจึงสร้าง class ปลอมง่ายๆ
ขึ้นมาแทนได้เลย ไม่ต้องพึ่ง library ใดๆ:

```ruby
# frozen_string_literal: true

# order_processor_test.rb
require "minitest/autorun"
require_relative "order_processor"

# Stub แบบง่ายที่สุด: object ปลอมที่ตอบค่าคงที่เสมอ ไม่สนใจว่าถูกเรียกด้วย argument อะไร
class AlwaysApprovePaymentStub
  def charge(_amount)
    true
  end
end

class AlwaysDeclinePaymentStub
  def charge(_amount)
    false
  end
end

class OrderProcessorTest < Minitest::Test
  def test_process_succeeds_when_payment_is_approved
    processor = OrderProcessor.new(payment_gateway: AlwaysApprovePaymentStub.new)
    result = processor.process(500)
    assert_equal "ชำระเงินสำเร็จ 500 บาท", result
  end

  def test_process_raises_error_when_payment_is_declined
    processor = OrderProcessor.new(payment_gateway: AlwaysDeclinePaymentStub.new)
    assert_raises(OrderProcessor::PaymentFailedError) do
      processor.process(500)
    end
  end

  def test_process_raises_error_for_non_positive_amount
    processor = OrderProcessor.new(payment_gateway: AlwaysApprovePaymentStub.new)
    assert_raises(ArgumentError) do
      processor.process(0)
    end
  end
end
```

```bash
ruby order_processor_test.rb
# => 3 runs, 3 assertions, 0 failures, 0 errors, 0 skips
```

สังเกตว่าเทสต์เหล่านี้รันเร็วมาก (ไม่มีการเรียก network จริง) และผลลัพธ์แน่นอน 100%
ทุกครั้งที่รัน (ไม่ flaky) เพราะ stub ไม่มีทางเปลี่ยนพฤติกรรม

หากไม่อยากประกาศ class ใหม่เพียงเพื่อใช้ครั้งเดียว สามารถสร้าง stub แบบด่วนด้วย
`Object.new` และ `define_singleton_method` ได้เช่นกัน:

```ruby
def test_process_using_a_one_off_stub_object
  stub = Object.new
  def stub.charge(_amount) = true

  processor = OrderProcessor.new(payment_gateway: stub)
  assert_equal "ชำระเงินสำเร็จ 200 บาท", processor.process(200)
end
```

วิธีนี้ใช้ได้เพราะ Ruby ไม่สนใจ class ของ object เลย สนใจแค่ว่า "ตอบสนอง method
`charge` ได้หรือไม่" (duck typing) — เป็นตัวอย่างที่ดีของสิ่งที่เรียนไปแล้วใน Part 016

---

## Step 178: `Minitest::Mock` — Mock และ Verify แบบเป็นทางการ

Stub ที่เขียนเองใน Step 177 ตอบคำถามได้แค่ "ถ้าเรียก `charge` จะได้ผลลัพธ์อะไร"
แต่ไม่เคยตรวจสอบว่า **`charge` ถูกเรียกจริงหรือไม่ ด้วย argument ที่ถูกต้องหรือไม่**
เมื่อไหร่ที่ต้องการยืนยันแบบนี้ เราต้องการ **Mock** ซึ่ง Minitest มีให้พร้อมใช้งานผ่าน
`Minitest::Mock`

```ruby
# frozen_string_literal: true

# order_processor_mock_test.rb
require "minitest/autorun"
require "minitest/mock"
require_relative "order_processor"

class OrderProcessorMockTest < Minitest::Test
  def test_process_calls_charge_with_correct_amount
    mock_gateway = Minitest::Mock.new
    # expect(method_name, ค่าที่ต้องการให้คืนกลับมา, [argument ที่ต้องถูกเรียกด้วย])
    mock_gateway.expect(:charge, true, [500])

    processor = OrderProcessor.new(payment_gateway: mock_gateway)
    result = processor.process(500)

    assert_equal "ชำระเงินสำเร็จ 500 บาท", result
    # verify ตรวจว่าทุก expectation ที่ตั้งไว้ถูกเรียกจริงครบถ้วน ถ้าไม่ครบจะ raise
    mock_gateway.verify
  end

  def test_process_raises_when_gateway_called_with_wrong_amount
    mock_gateway = Minitest::Mock.new
    mock_gateway.expect(:charge, true, [999]) # คาดหวังว่าจะถูกเรียกด้วย 999

    processor = OrderProcessor.new(payment_gateway: mock_gateway)

    # แต่โค้ดจริงเรียก charge(500) ไม่ตรงกับที่ mock คาดไว้ -> Minitest::Mock จะ raise ทันที
    assert_raises(MockExpectationError) do
      processor.process(500)
    end
  end
end
```

```bash
ruby order_processor_mock_test.rb
# => 2 runs, 3 assertions, 0 failures, 0 errors, 0 skips
```

**อธิบาย:**

- `mock_gateway.expect(:charge, true, [500])` ตั้งความคาดหวังว่า: "ถ้ามีใครเรียก method
  `charge` ด้วย argument `500` ให้คืนค่า `true` กลับไป" ถ้าไม่มีการเรียกที่ตรงกับที่
  ประกาศไว้เลย จะถือว่าเทสต์ผิดพลาด
- ถ้าเรียก `charge` ด้วย argument ที่**ไม่ตรง**กับที่ `expect` ไว้ (เช่นเทสต์ที่สองเรียก
  ด้วย `500` แต่ mock คาดหวัง `999`) `Minitest::Mock` จะ raise
  `MockExpectationError` **ทันที** ณ จุดที่เรียก ไม่ต้องรอถึง `.verify`
- `mock_gateway.verify` ใช้ตรวจสอบว่า **ทุก** expectation ที่ตั้งไว้ถูกเรียกจริงครบ
  (กรณีตั้ง `expect` ไว้แต่ไม่มีการเรียกเลยสักครั้ง `.verify` จะเป็นตัวจับได้)
- นี่คือความต่างสำคัญระหว่าง **Stub กับ Mock**: Stub สนใจแค่ "คืนค่าอะไร" ไม่สนใจว่า
  ถูกเรียกยังไง ส่วน Mock สนใจทั้ง "คืนค่าอะไร" และ "ถูกเรียกอย่างถูกต้องหรือไม่"

### `Object#stub` — วิธี stub method เดียวแบบชั่วคราวโดยไม่ต้องสร้าง class ใหม่

`minitest/mock` ยังแถม method ชื่อ `stub` ที่เพิ่มเข้าไปในทุก Object ให้ใช้ override
method ใดๆ ของ object จริง **ชั่วคราวเฉพาะภายใน block** แล้วคืนพฤติกรรมเดิมกลับ
อัตโนมัติเมื่อออกจาก block — มีประโยชน์มากเวลาต้องการ stub method ของ object ที่มีอยู่
จริงแล้ว (เช่น `Time.now`) โดยไม่ต้องสร้าง test double ทั้งก้อน

```ruby
# frozen_string_literal: true

# order_processor_stub_method_test.rb
require "minitest/autorun"
require "minitest/mock"
require_relative "order_processor"
require_relative "payment_gateway"

class OrderProcessorStubMethodTest < Minitest::Test
  def test_process_using_object_stub_helper
    gateway = PaymentGateway.new # ของจริง ที่ปกติ charge จะ raise NotImplementedError
    processor = OrderProcessor.new(payment_gateway: gateway)

    # ระหว่างอยู่ใน block นี้เท่านั้น gateway.charge(อะไรก็ตาม) จะคืนค่า true เสมอ
    gateway.stub(:charge, true) do
      result = processor.process(300)
      assert_equal "ชำระเงินสำเร็จ 300 บาท", result
    end
    # ออกจาก block แล้ว gateway.charge จะกลับไป raise NotImplementedError เหมือนเดิม
  end
end
```

```bash
ruby order_processor_stub_method_test.rb
# => 1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

**เมื่อไหร่ควรใช้อะไร:**

- ใช้ **stub ธรรมดา** (Step 177) เมื่อต้องการแค่ "ตัด dependency ที่ช้า/ไม่แน่นอนออก
  จากเทสต์" และไม่สนใจว่าถูกเรียกกี่ครั้งด้วย argument อะไร
- ใช้ **`Minitest::Mock`** เมื่อพฤติกรรมที่สำคัญที่สุดที่ต้องการทดสอบ**คือการเรียก
  dependency นั้นถูกต้อง** เช่น "ต้องแน่ใจว่าเราส่งจำนวนเงินที่ถูกต้องไปเรียกเก็บเงินจริง"
- ใช้ **`Object#stub`** เมื่อต้องการ override method เดียวของ object ที่มีอยู่แล้วแบบ
  ชั่วคราว โดยไม่อยากสร้าง class ปลอมทั้งก้อน

---

## Step 179: ข้อตกลงการตั้งชื่อเทสต์ หลักการ One-Assertion-Per-Test และ Minitest::Spec DSL

### การตั้งชื่อ test method

ชื่อ `test_` method ที่ดีควรอ่านแล้วเข้าใจได้ทันทีว่า **ทดสอบอะไร ภายใต้เงื่อนไขแบบไหน
คาดหวังผลลัพธ์อย่างไร** รูปแบบที่นิยม:

```
test_<method_ที่ทดสอบ>_<เงื่อนไข>_<ผลลัพธ์ที่คาดหวัง>
```

```ruby
def test_divide_by_zero_raises_zero_division_error   # ดี: ชัดเจนทั้ง 3 ส่วน
def test_divide_normal_numbers_returns_float_result   # ดี
def test_divide                                        # แย่: ไม่รู้ว่าทดสอบเงื่อนไขไหน
def test_1                                              # แย่มาก: ไม่สื่อความหมายเลย
```

ชื่อที่ดีมีประโยชน์เพิ่มเติมคือเวลารายงานผล fail จะขึ้นชื่อ method นั้นมาด้วย ทำให้อ่าน
รายงานแล้วเข้าใจทันทีว่าโค้ดส่วนไหนพังโดยไม่ต้องเปิดไฟล์เทสต์ไปดู

### หลักการ One-Assertion-Per-Test (และข้อยกเว้นที่ใช้ได้จริง)

แนวทางที่แนะนำโดยทั่วไปคือ **หนึ่งเทสต์ควรมี assertion เดียว** เพื่อให้:

1. เวลา fail รู้ทันทีว่าพังจุดไหน (ไม่ต้องมานั่งดูว่า assertion บรรทัดที่เท่าไหร่ที่ error)
2. ชื่อเทสต์สื่อความหมายได้ชัดเจน เพราะทดสอบพฤติกรรมเดียว

```ruby
# ไม่ควรทำ — รวมหลายพฤติกรรมที่ไม่เกี่ยวข้องกันไว้ในเทสต์เดียว
def test_calculator
  calc = Calculator.new
  assert_equal 5, calc.add(2, 3)
  assert_equal 1, calc.subtract(3, 2)
  assert_equal 6, calc.multiply(2, 3)
end

# ควรทำ — แยกเทสต์ตามพฤติกรรม แต่ละเทสต์ล้มเหลวอย่างอิสระจากกัน
def test_add_returns_sum
  assert_equal 5, Calculator.new.add(2, 3)
end

def test_subtract_returns_difference
  assert_equal 1, Calculator.new.subtract(3, 2)
end

def test_multiply_returns_product
  assert_equal 6, Calculator.new.multiply(2, 3)
end
```

**ข้อยกเว้นที่ใช้ได้จริง (pragmatic exception):** เมื่อ assertion หลายตัวล้วนอธิบาย
**ข้อเท็จจริงเดียวกัน** (เช่น "object ที่สร้างขึ้นมาใหม่มีสถานะเริ่มต้นถูกต้อง")
การรวมไว้ในเทสต์เดียวก็สมเหตุสมผล เพราะทุก assertion วัดผลลัพธ์ของการกระทำเดียวกัน:

```ruby
def test_new_order_has_correct_default_state
  order = Order.new(customer: "สมชาย")

  assert_equal "สมชาย", order.customer
  assert_equal "pending", order.status
  assert_empty order.items
end
```

ถ้า `assert_equal "สมชาย", order.customer` fail แต่สองบรรทัดล่างไม่ทันได้รัน (Minitest
หยุดที่ assertion แรกที่ fail ในแต่ละ test method) ก็ยังอ่านรายงานแล้วรู้ทันทีว่า
"การสร้าง Order มีปัญหาตรง customer" ซึ่งเพียงพอแล้วสำหรับกรณีนี้ — กฎนี้จึงเป็น
**แนวทาง (guideline) ไม่ใช่กฎตายตัว** ให้ใช้วิจารณญาณว่า assertion เหล่านั้น "เล่าเรื่อง
เดียวกัน" หรือไม่

### Minitest::Spec — เขียนเทสต์แบบ `describe`/`it`

นอกจากสไตล์ `class ... < Minitest::Test` แล้ว Minitest ยังมี DSL แบบ "spec" ให้เขียน
ในรูปประโยคภาษาธรรมชาติมากขึ้น โดยใช้ `describe`/`it`/`before` แทน `class`/`test_`/`setup`
— รูปแบบนี้เป็น API เดียวกับที่ **RSpec** (Part 019) ใช้ ทำให้เรียน Minitest::Spec
ก่อนแล้วจะย้ายไป RSpec ได้ง่ายขึ้นมาก:

```ruby
# frozen_string_literal: true

# calculator_spec_test.rb
require "minitest/autorun"
require_relative "calculator"

describe Calculator do
  before do
    @calculator = Calculator.new
  end

  it "returns the sum of two numbers" do
    _(@calculator.add(2, 3)).must_equal 5
  end

  it "returns the difference of two numbers" do
    _(@calculator.subtract(3, 2)).must_equal 1
  end

  it "raises ZeroDivisionError when dividing by zero" do
    _(-> { @calculator.divide(10, 0) }).must_raise ZeroDivisionError
  end
end
```

```bash
ruby calculator_spec_test.rb
# => 3 runs, 3 assertions, 0 failures, 0 errors, 0 skips
```

**อธิบาย:**

- `describe Calculator do ... end` สร้าง class ลูกของ `Minitest::Spec` (ซึ่งตัวมันเอง
  สืบทอดมาจาก `Minitest::Test` อีกที) ขึ้นมาโดยอัตโนมัติ เบื้องหลังคือกลไกเดียวกับที่
  เราเขียนด้วยมือใน Step 172–178 ทุกประการ เพียงแต่ห่อด้วย syntax ที่อ่านลื่นกว่า
- `before do ... end` คือ alias ของ `setup`
- `it "คำอธิบายพฤติกรรม" do ... end` แปลงเป็น `test_` method ให้อัตโนมัติ (ชื่อ method
  ถูกสร้างจากข้อความใน `it` เช่น `test_0001_returns the sum of two numbers`)
- `must_equal`, `must_raise` คือ **matcher** ของฝั่ง expectation API เทียบเท่ากับ
  `assert_equal`, `assert_raises` ในฝั่ง assertion API ทุกประการ แค่เขียนคนละสไตล์
- `_( ... )` คือฟังก์ชันตัวช่วย (มาแทนสไตล์เก่าที่เคยเขียน `@calculator.add(2,3).must_equal 5`
  ตรงๆ) ใช้ห่อค่าที่จะตรวจสอบ เพื่อไม่ให้ Minitest ต้องแอบเพิ่ม method อย่าง
  `must_equal` เข้าไปใน **ทุก Object** ของโปรแกรม (ซึ่งเป็นแนวทางที่ทันสมัยและปลอดภัย
  กว่าการ monkey-patch `Object` ทั้งระบบ)

สไตล์ไหนดีกว่ากัน? ไม่มีคำตอบตายตัว — บาง team ชอบ `class`/`assert_equal` เพราะเป็น
Ruby ธรรมดาไม่มี DSL ซ่อนเวทมนตร์ บาง team ชอบ `describe`/`it` เพราะอ่านเหมือนประโยค
ภาษาอังกฤษ (readable specification) สิ่งสำคัญคือ **เลือกสไตล์เดียวแล้วใช้ให้สม่ำเสมอ
ทั้งโปรเจกต์**

---

## Step 180: รันเทสต์ด้วย Rakefile และแบบฝึกหัดโปรเจกต์ ShoppingCart

### รวมเทสต์ทั้งหมดด้วย Rake

**Rake** คือเครื่องมือจัดการ "task" อัตโนมัติที่มากับ Ruby (เราจะเจาะลึกเต็มรูปแบบใน
Part 020) ตอนนี้ขอแนะนำแค่ส่วนที่เกี่ยวกับ Minitest: `Rake::TestTask` ที่ทำให้พิมพ์
คำสั่งสั้นๆ `rake test` แทนคำสั่งยาวๆ ใน Step 176 ได้

```ruby
# frozen_string_literal: true

# Rakefile
require "rake/testtask"

Rake::TestTask.new(:test) do |t|
  t.libs << "test"
  t.libs << "lib"
  t.test_files = FileList["test/**/*_test.rb"]
  t.verbose = true
end

task default: :test
```

```bash
rake test
# หรือเพราะตั้ง `task default: :test` ไว้ พิมพ์แค่ rake เฉยๆ ก็รันเทสต์ทั้งหมดได้
rake
```

ผลลัพธ์จะรวมทุกไฟล์ที่แมตช์ `test/**/*_test.rb` เข้าเป็นรายงานเดียว — สะดวกกว่าการไล่
พิมพ์ `ruby` ทีละไฟล์มาก และเป็นคำสั่งมาตรฐานที่ CI/CD pipeline (Part 075) จะเรียกใช้
ต่อไปในอนาคต

---

### แบบฝึกหัด: เขียน Minitest Suite ให้ `ShoppingCart` พร้อม Dependency `TaxCalculator`

**โจทย์:** มีโค้ด production สองไฟล์ในโฟลเดอร์ `lib/` ดังนี้ ให้เขียนชุดเทสต์ครบถ้วนใน
`test/` โดยใช้ทั้งเทคนิค stub และ `Minitest::Mock` ตามความเหมาะสม

```ruby
# frozen_string_literal: true

# lib/tax_calculator.rb
class TaxCalculator
  DEFAULT_RATE = 0.07 # ภาษีมูลค่าเพิ่ม (VAT) 7%

  def initialize(rate: DEFAULT_RATE)
    @rate = rate
  end

  def calculate(subtotal)
    (subtotal * @rate).round(2)
  end
end
```

```ruby
# frozen_string_literal: true

# lib/shopping_cart.rb
CartItem = Struct.new(:name, :unit_price, :quantity) do
  def subtotal
    unit_price * quantity
  end
end

class ShoppingCart
  attr_reader :items

  def initialize(tax_calculator:)
    @items = []
    @tax_calculator = tax_calculator
  end

  def add_item(name:, unit_price:, quantity: 1)
    raise ArgumentError, "ราคาต้องมากกว่า 0" if unit_price <= 0
    raise ArgumentError, "จำนวนต้องมากกว่า 0" if quantity <= 0

    existing = items.find { |item| item.name == name }
    if existing
      existing.quantity += quantity
    else
      items << CartItem.new(name, unit_price, quantity)
    end
  end

  def remove_item(name)
    items.reject! { |item| item.name == name }
  end

  def subtotal
    items.sum(&:subtotal)
  end

  def tax
    @tax_calculator.calculate(subtotal)
  end

  def total
    subtotal + tax
  end

  def empty?
    items.empty?
  end
end
```

`ShoppingCart` มี dependency คือ `tax_calculator` ที่ต้องมี method `calculate(subtotal)`
— นี่คือจุดที่ต้องใช้ Test Doubles: ถ้าเทสต์ `ShoppingCart` โดยพึ่ง `TaxCalculator` จริง
ทุกครั้ง เทสต์จะพังทุกครั้งที่มีคนแก้อัตราภาษีจริง (เพราะ tax เปลี่ยน ตัวเลขคาดหวังทั้งหมด
ต้องแก้ตาม) ทั้งที่พฤติกรรมของ `ShoppingCart` เองไม่ได้เปลี่ยนเลย — จึงควร stub
`tax_calculator` ในเทสต์ของ `ShoppingCart` และแยกเทสต์ `TaxCalculator` เองต่างหาก

### เฉลย

```ruby
# frozen_string_literal: true

# test/tax_calculator_test.rb
require "minitest/autorun"
require_relative "../lib/tax_calculator"

class TaxCalculatorTest < Minitest::Test
  def test_default_rate_is_seven_percent
    calculator = TaxCalculator.new
    assert_in_delta 7.0, calculator.calculate(100), 0.001
  end

  def test_custom_rate_can_be_provided
    calculator = TaxCalculator.new(rate: 0.10)
    assert_in_delta 10.0, calculator.calculate(100), 0.001
  end

  def test_calculate_rounds_result_to_two_decimal_places
    calculator = TaxCalculator.new(rate: 0.07)
    assert_in_delta 7.35, calculator.calculate(105), 0.001
  end
end
```

```ruby
# frozen_string_literal: true

# test/shopping_cart_test.rb
require "minitest/autorun"
require "minitest/mock"
require_relative "../lib/shopping_cart"

# Stub ภาษีแบบง่าย: คืนค่าคงที่เสมอไม่ว่า subtotal จะเป็นเท่าไหร่
# ใช้ในเทสต์ที่สนใจ "พฤติกรรมของ ShoppingCart" ไม่ใช่ตัวเลขภาษีจริง
class FixedTaxStub
  def initialize(fixed_tax)
    @fixed_tax = fixed_tax
  end

  def calculate(_subtotal)
    @fixed_tax
  end
end

class ShoppingCartTest < Minitest::Test
  def setup
    @tax_stub = FixedTaxStub.new(0)
    @cart = ShoppingCart.new(tax_calculator: @tax_stub)
  end

  def test_new_cart_is_empty
    assert @cart.empty?
    assert_equal 0, @cart.subtotal
  end

  def test_add_item_adds_a_new_line_to_the_cart
    @cart.add_item(name: "หนังสือ Ruby", unit_price: 350, quantity: 2)

    refute @cart.empty?
    assert_equal 1, @cart.items.size
  end

  def test_add_item_with_same_name_increases_quantity_instead_of_duplicating
    @cart.add_item(name: "ปากกา", unit_price: 10, quantity: 2)
    @cart.add_item(name: "ปากกา", unit_price: 10, quantity: 3)

    assert_equal 1, @cart.items.size
    assert_equal 5, @cart.items.first.quantity
  end

  def test_add_item_with_non_positive_price_raises_error
    assert_raises(ArgumentError) do
      @cart.add_item(name: "ของแถม", unit_price: 0, quantity: 1)
    end
  end

  def test_add_item_with_non_positive_quantity_raises_error
    assert_raises(ArgumentError) do
      @cart.add_item(name: "ของแถม", unit_price: 10, quantity: 0)
    end
  end

  def test_subtotal_sums_all_line_items
    @cart.add_item(name: "สมุด", unit_price: 25, quantity: 4)
    @cart.add_item(name: "ยางลบ", unit_price: 5, quantity: 2)

    assert_equal 110, @cart.subtotal
  end

  def test_remove_item_deletes_it_from_the_cart
    @cart.add_item(name: "ไม้บรรทัด", unit_price: 15, quantity: 1)
    @cart.remove_item("ไม้บรรทัด")

    assert @cart.empty?
  end

  def test_total_includes_subtotal_and_tax_from_stub
    tax_stub = FixedTaxStub.new(20)
    cart = ShoppingCart.new(tax_calculator: tax_stub)
    cart.add_item(name: "กระเป๋า", unit_price: 100, quantity: 1)

    assert_equal 100, cart.subtotal
    assert_equal 20, cart.tax
    assert_equal 120, cart.total
  end

  def test_total_calls_tax_calculator_with_correct_subtotal
    mock_tax_calculator = Minitest::Mock.new
    mock_tax_calculator.expect(:calculate, 21.0, [300])

    cart = ShoppingCart.new(tax_calculator: mock_tax_calculator)
    cart.add_item(name: "เสื้อ", unit_price: 150, quantity: 2)

    assert_equal 321.0, cart.total
    mock_tax_calculator.verify
  end
end
```

รันทั้งหมดด้วย Rake:

```bash
rake test
```

ผลลัพธ์ที่คาดหวัง:

```
Run options: --seed 8842

# Running:

.............

Finished in 0.004921s, 2641.7 runs/s, 2438.5 assertions/s.

13 runs, 12 assertions, 0 failures, 0 errors, 0 skips
```

**สิ่งสำคัญที่เฉลยนี้สาธิต:**

- แยกเทสต์ `TaxCalculator` (dependency) กับเทสต์ `ShoppingCart` (ตัวที่ใช้ dependency
  นั้น) ออกจากกันอย่างชัดเจน คนละไฟล์ คนละความรับผิดชอบ
- ใน `ShoppingCartTest` ส่วนใหญ่ใช้ `FixedTaxStub` เพราะสิ่งที่อยากยืนยันคือพฤติกรรม
  ของตะกร้าสินค้า (การบวก item, การนับจำนวน, การคำนวณ subtotal) ไม่ใช่ตัวเลขภาษี
- เทสต์สุดท้าย `test_total_calls_tax_calculator_with_correct_subtotal` ใช้
  `Minitest::Mock` เพราะครั้งนี้สิ่งที่อยากยืนยันคือ **`ShoppingCart` ส่ง subtotal ที่
  ถูกต้อง (300) ไปให้ `tax_calculator` คำนวณ** ซึ่งเป็นพฤติกรรม (interaction) ที่ stub
  ธรรมดาตรวจสอบไม่ได้ — ต้องใช้ mock เท่านั้น
- ทุกเทสต์ตั้งชื่อสื่อความหมาย ทำตาม pattern
  `test_<สิ่งที่ทดสอบ>_<เงื่อนไข/ผลลัพธ์>` ตามที่เรียนใน Step 179

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม method `apply_discount(percent)` ให้ `ShoppingCart` ที่ลดราคาจาก `subtotal`
   ก่อนคิดภาษี แล้วเขียนเทสต์ครอบคลุมกรณี percent = 0, percent = 100 (ราคาสุดท้ายควร
   เป็น 0 บวกภาษีที่คำนวณจาก 0), และกรณี percent ติดลบหรือเกิน 100 ควร `raise
   ArgumentError`
2. เขียนเทสต์เพิ่มสำหรับ `remove_item` กรณีเรียกด้วยชื่อสินค้าที่**ไม่มีอยู่**ในตะกร้า
   (ควรไม่เกิด error ใดๆ และตะกร้าไม่มีการเปลี่ยนแปลงเลย)
3. ใช้ `Minitest::Mock` เขียนเทสต์ยืนยันว่า `tax_calculator.calculate` ถูกเรียก
   **เพียงครั้งเดียว** ต่อการเรียก `total` หนึ่งครั้ง (ไม่ใช่ถูกเรียกซ้ำจากภายใน
   `subtotal`, `tax`, และ `total` รวมกันหลายครั้งโดยไม่จำเป็น) — ใบ้: ลองเรียก
   `cart.total` แล้วดูว่า mock ที่ตั้ง `expect` ไว้แค่ครั้งเดียวจะ raise
   `MockExpectationError` หรือไม่ถ้าโค้ดจริงเรียกเกินหนึ่งครั้ง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่าทำไม automated testing ถึงสำคัญ: ป้องกัน regression, เป็น documentation ที่
  รันได้จริงเสมอ, และให้ feedback ต่อการออกแบบโค้ด (บังคับให้คิดเรื่อง dependency
  injection)
- ตั้งค่าและเขียนไฟล์เทสต์ Minitest แรกได้: `require "minitest/autorun"`, class ที่
  สืบทอดจาก `Minitest::Test`, method ที่ขึ้นต้นด้วย `test_`
- ใช้ assertion หลักได้อย่างถูกต้อง: `assert`, `assert_equal`, `assert_nil`,
  `assert_raises`, `assert_includes`, `assert_kind_of`, `assert_instance_of`,
  `assert_respond_to`, `assert_match` รวมถึงคู่ตรงข้าม `refute_*` ทั้งหมด
- รู้วิธีเปรียบเทียบ Float อย่างปลอดภัยด้วย `assert_in_delta` แทนการใช้ `assert_equal`
  ตรงๆ
- ใช้ `setup`/`teardown` เตรียมและล้างสภาพแวดล้อมให้แต่ละเทสต์เป็นอิสระจากกัน
  (test isolation)
- จัดโครงสร้างโปรเจกต์แบบ `lib/`/`test/` และรันหลายไฟล์เทสต์พร้อมกันได้
- เข้าใจคำศัพท์ Test Doubles ทั้ง 5 แบบ (Dummy, Stub, Mock, Spy, Fake) และเขียน stub
  ด้วย Ruby ธรรมดาผ่าน dependency injection ได้โดยไม่ต้องพึ่ง gem ใดๆ
- ใช้ `Minitest::Mock` ตั้งความคาดหวัง (`expect`) และตรวจสอบ (`verify`) การเรียกใช้
  dependency ได้ รวมถึงรู้จัก `Object#stub` สำหรับ override method ชั่วคราว
- เข้าใจข้อตกลงการตั้งชื่อเทสต์ หลักการ one-assertion-per-test พร้อมรู้ว่าเมื่อไหร่ควร
  ยืดหยุ่นกับกฎนี้ และเขียนเทสต์แบบ `describe`/`it` ด้วย Minitest::Spec ได้
- รวมเทสต์ทั้งโปรเจกต์เข้าด้วยกันและรันผ่านคำสั่งเดียวด้วย `Rakefile` และ
  `Rake::TestTask`
- เขียน test suite ที่สมบูรณ์ให้ class ที่มี dependency จริง (`ShoppingCart` +
  `TaxCalculator`) โดยผสมทั้ง stub และ mock ตามความเหมาะสม

**ต่อไป (Part 019):** เราจะเรียนรู้ **RSpec** — testing framework ที่ได้รับความนิยม
สูงสุดในวงการ Ruby on Rails ระดับมืออาชีพ ซึ่งใช้ syntax คล้ายกับ Minitest::Spec ที่เพิ่ง
เห็นใน Step 179 มาก แต่มี ecosystem และ matcher ที่ทรงพลังกว่ามาก เราจะเจาะลึก
`describe`/`context`/`it`, matcher หลากหลายรูปแบบ, `let`/`let!` สำหรับสร้างข้อมูลทดสอบ
แบบ lazy-loaded, และ `before`/`after` hook ในรูปแบบของ RSpec — ซึ่งจะเป็นพื้นฐานสำคัญ
ก่อนเข้าสู่การเทสต์ Rails application เต็มรูปแบบใน Part 046 เป็นต้นไป
