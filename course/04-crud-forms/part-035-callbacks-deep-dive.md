# Part 035: ActiveRecord Callback ขั้นสูง — Lifecycle เต็มรูปแบบ และข้อควรระวัง

> **Step ครอบคลุมใน Part นี้:** Step 341–350
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 026 มาก่อน โดยเฉพาะเรื่อง callback lifecycle เบื้องต้น,
> `before_save`/`after_create`/`after_commit` และ fat-model anti-pattern)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

Part 026 พาไปรู้จัก callback lifecycle เบื้องต้นแล้ว — ลำดับคร่าวๆ ของ `before_save`,
`after_create`, `after_commit` และคำเตือนเรื่อง fat model แต่นั่นเป็นแค่ผิวเผิน ในงานจริงระดับ
production มีรายละเอียดอีกมากที่ถ้าไม่เข้าใจให้ลึกพอ จะนำไปสู่บั๊กที่ตามรอยยากมาก เช่น
ธุรกรรมที่ "rollback ไม่จริง", callback ที่วนเรียกตัวเองจน server ค้าง, หรือโค้ดที่ copy มาจาก
tutorial เก่าที่เขียนด้วยความเชื่อผิดๆ ว่า `return false` ยังหยุด callback chain ได้เหมือน Rails 4

Part นี้จะไม่พูดถึงพื้นฐานที่ Part 026 สอนไปแล้วซ้ำอีก แต่จะเจาะลึกกลไกที่อยู่ "ใต้ฝาครอบ" ของ
callback: `around_*` callback ที่ครอบทั้งสองฝั่ง, วิธีหยุด callback chain ที่ถูกต้องในยุคปัจจุบัน,
callback แบบ object ที่แยกทดสอบได้, callback ในความสัมพันธ์ระหว่าง Model, transaction ที่ทำงาน
คู่กับ callback อย่างไร, และกับดักที่วิศวกร Rails มืออาชีพเจอมาแล้วจริงในงาน production

## สารบัญของ Part นี้

- Step 341: ตารางอ้างอิงฉบับสมบูรณ์ — ลำดับ callback ทั้งหมดตอน create/update/destroy (verified)
- Step 342: `around_save`/`around_create`/`around_update`/`around_destroy` และ `yield`
- Step 343: หยุด callback chain ให้ถูกต้อง — `throw :abort` vs ความเชื่อผิดๆ เรื่อง `return false`
- Step 344: Conditional callback ในทางปฏิบัติ — `if:`/`unless:` แบบเจาะลึกกว่าที่เคยเห็นใน Part 026
- Step 345: Callback Class/Object — แยก callback logic ออกมาให้ทดสอบได้อิสระจาก Model
- Step 346: Callback บน Association — `before_destroy` ของลูก, `autosave: true`
- Step 347: Transaction กับ Callback — nested transaction, `requires_new:`, และ rollback อัตโนมัติ
- Step 348: `touch: true` — cascade `updated_at` ขึ้นไปยัง parent และการใช้ทำ cache invalidation
- Step 349: กับดักจริง — Callback วนเรียกตัวเองไม่รู้จบ (infinite callback loop)
- Step 350: การเทสต์ Model ที่มี Callback — เมื่อไหร่ควร skip และเมื่อไหร่ห้าม skip

---

## Step 341: ตารางอ้างอิงฉบับสมบูรณ์ — ลำดับ Callback ทั้งหมด

Part 026 Step 257 แสดงลำดับ callback หลักๆ ไปแล้ว แต่ยังไม่ครอบคลุม `around_*` ซึ่งเป็น
callback อีกประเภทหนึ่งที่ "ครอบ" ทั้งก่อนและหลังการกระทำไว้ในตัวเดียว มาสร้าง Model ทดลองที่มี
callback **ครบทุกประเภทที่ ActiveRecord รองรับ** เพื่อพิสูจน์ลำดับที่แท้จริงอีกครั้งแบบสมบูรณ์

```ruby
# app/models/callback_demo.rb
class CallbackDemo < ApplicationRecord
  attr_accessor :log

  before_validation { record_step("before_validation") }
  after_validation  { record_step("after_validation") }

  before_save       { record_step("before_save") }
  around_save       :around_save_wrapper

  before_create     { record_step("before_create") }
  around_create     :around_create_wrapper
  after_create      { record_step("after_create") }

  before_update     { record_step("before_update") }
  around_update     :around_update_wrapper
  after_update      { record_step("after_update") }

  after_save        { record_step("after_save") }

  before_destroy    { record_step("before_destroy") }
  around_destroy    :around_destroy_wrapper
  after_destroy     { record_step("after_destroy") }

  after_commit      { record_step("after_commit") }
  after_rollback    { record_step("after_rollback") }

  private

  def record_step(name)
    (self.log ||= []) << name
  end

  def around_save_wrapper
    record_step("around_save (ก่อน yield)")
    yield
    record_step("around_save (หลัง yield)")
  end

  def around_create_wrapper
    record_step("around_create (ก่อน yield)")
    yield
    record_step("around_create (หลัง yield)")
  end

  def around_update_wrapper
    record_step("around_update (ก่อน yield)")
    yield
    record_step("around_update (หลัง yield)")
  end

  def around_destroy_wrapper
    record_step("around_destroy (ก่อน yield)")
    yield
    record_step("around_destroy (หลัง yield)")
  end
end
```

### ทดสอบจริงผ่าน `bin/rails runner`

```ruby
demo = CallbackDemo.new(name: "แรก")
demo.save
demo.log
```

ผลลัพธ์จริงที่ได้ (ทดสอบบน Rails 8.1.4):

```
["before_validation", "after_validation", "before_save",
 "around_save (ก่อน yield)", "before_create", "around_create (ก่อน yield)",
 "around_create (หลัง yield)", "after_create", "around_save (หลัง yield)",
 "after_save", "after_commit"]
```

```ruby
demo.log = []
demo.update(name: "แก้ไขแล้ว")
demo.log
```

```
["before_validation", "after_validation", "before_save",
 "around_save (ก่อน yield)", "before_update", "around_update (ก่อน yield)",
 "around_update (หลัง yield)", "after_update", "around_save (หลัง yield)",
 "after_save", "after_commit"]
```

```ruby
demo.log = []
demo.destroy
demo.log
```

```
["before_destroy", "around_destroy (ก่อน yield)",
 "around_destroy (หลัง yield)", "after_destroy", "after_commit"]
```

### ตารางอ้างอิงฉบับสมบูรณ์ (ยึดตามผลทดสอบจริงด้านบน)

**ตอน `create`:**

| # | Callback | หมายเหตุ |
|---|-----------|-----------|
| 1 | `before_validation` | |
| 2 | *(validation ทำงาน)* | |
| 3 | `after_validation` | |
| 4 | `before_save` | ทำงานทั้ง create/update |
| 5 | `around_save` (ก่อน `yield`) | ครอบตั้งแต่จุดนี้ไปจนถึง after_save |
| 6 | `before_create` | เฉพาะ create |
| 7 | `around_create` (ก่อน `yield`) | ครอบเฉพาะช่วง INSERT |
| 8 | *(คำสั่ง SQL `INSERT`)* | เกิดขึ้นตรงจุดที่ `around_create` เรียก `yield` |
| 9 | `around_create` (หลัง `yield`) | |
| 10 | `after_create` | เฉพาะ create |
| 11 | `around_save` (หลัง `yield`) | |
| 12 | `after_save` | ทำงานทั้ง create/update |
| 13 | `after_commit` (หรือ `after_rollback`) | หลัง transaction commit/rollback จริง |

**ตอน `update`:** เหมือนกันทุกจุด สลับ `before_create`/`around_create`/`after_create` เป็น
`before_update`/`around_update`/`after_update`

**ตอน `destroy`:**

| # | Callback |
|---|-----------|
| 1 | `before_destroy` |
| 2 | `around_destroy` (ก่อน `yield`) |
| 3 | *(คำสั่ง SQL `DELETE`)* |
| 4 | `around_destroy` (หลัง `yield`) |
| 5 | `after_destroy` |
| 6 | `after_commit` (หรือ `after_rollback`) |

> **จุดสำคัญที่สุดของตารางนี้:** `around_save` "ครอบ" ทั้ง `around_create`/`around_update`
> อีกทีหนึ่ง เพราะ `before_save`/`after_save` ทำงานทั้ง create และ update ในขณะที่
> `around_create`/`around_update` เจาะจงเฉพาะกรณีของตัวเอง — จำง่ายๆ ว่า
> **`save` ครอบ `create`/`update` เสมอ ไม่ว่าจะเป็น callback ประเภท before/around/after ก็ตาม**

---

## Step 342: `around_*` Callback และ `yield`

`around_save`/`around_create`/`around_update`/`around_destroy` คือ callback ประเภทที่สาม
(นอกจาก before/after) — ฟังก์ชันเดียวเขียน logic ได้ทั้ง "ก่อน" และ "หลัง" การกระทำ โดยใช้
`yield` เป็นจุดแบ่ง คล้ายกับการเขียน middleware หรือ `around_action` ใน Controller (ที่เรียนไป
แล้วใน Part 023)

### Use case จริง: จับเวลาการบันทึกเพื่อ log performance

```ruby
class Post < ApplicationRecord
  around_save :log_save_duration

  private

  def log_save_duration
    started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at) * 1000).round(2)
    Rails.logger.info("[Post##{id}] save ใช้เวลา #{duration_ms}ms")
  end
end
```

จุดสำคัญ: **ต้องเรียก `yield` เสมอ** ไม่เช่นนั้นการบันทึกจริง (`INSERT`/`UPDATE`) จะไม่เกิดขึ้นเลย
— `yield` ในที่นี้คือจุดที่ ActiveRecord จะรัน callback ชั้นในถัดไป (หรือ SQL จริง ถ้าเป็น
callback ชั้นในสุด) ถ้าลืม `yield` ไป การบันทึกจะดูเหมือน "เงียบหาย" โดยไม่มี error ใดๆ เลย
(เหมือนตัวอย่างในหัวข้อถัดไป)

### กับดัก: `throw :abort` ใช้กับ `around_*` ไม่ได้แบบที่คิด

หลายคนเข้าใจผิดว่า `throw :abort` ใช้หยุด callback chain ได้ทุกที่เหมือนกันหมด แต่ทดสอบจริง
พบว่า **`throw :abort` ที่เรียกจากภายใน `around_*` callback จะ raise `UncaughtThrowError`**
ไม่ใช่หยุด chain แบบเงียบๆ เหมือนที่ `before_*` ทำ:

```ruby
class AroundAbortDemo < ActiveRecord::Base
  self.table_name = "callback_demos"

  around_save :guard_around_save

  def guard_around_save
    throw :abort   # ผิด! จะ raise UncaughtThrowError ไม่ใช่หยุด save เงียบๆ
    yield
  end
end

AroundAbortDemo.new(name: "test").save
```

```
guard_around_save': uncaught throw :abort (UncaughtThrowError)
```

**เหตุผลเชิงกลไก:** `throw :abort` ทำงานได้เพราะ ActiveSupport ห่อ callback แต่ละตัวประเภท
`before` ไว้ด้วย `catch(:abort) { ... }` เป็นรายตัว ส่วน `around_*` callback ไม่ได้ถูกห่อด้วย
`catch(:abort)` แบบเดียวกัน — วิธีที่ถูกต้องในการหยุด save จากภายใน `around_*` คือ**ไม่เรียก
`yield` เลย**:

```ruby
class AroundGuardDemo < ActiveRecord::Base
  self.table_name = "callback_demos"

  around_save :guard_around_save

  def guard_around_save
    return if name.blank?   # เงื่อนไขไม่ผ่าน -> ไม่เรียก yield -> ไม่มี INSERT/UPDATE เกิดขึ้น

    yield
  end
end

r = AroundGuardDemo.new(name: nil)
result = r.save
# result => nil, r.persisted? => false (การไม่ yield ทำให้ save คืนค่า nil ไม่ใช่ false ตรงๆ
#   ถ้าต้องการให้ระบบภายนอกรู้ชัดว่า "ถูกปฏิเสธ" ควรใช้ before_save + throw :abort แทน
#   เพราะสื่อความหมายชัดกว่าและให้ผลลัพธ์ที่คาดเดาได้ง่ายกว่า)
```

> **แนวทางปฏิบัติ:** ในเกือบทุกกรณีจริง ควรใช้ `before_*`/`after_*` ธรรมดาคู่กับ `throw :abort`
> เมื่อต้องการหยุด callback chain แบบมีเงื่อนไข ส่วน `around_*` เหมาะกับงานที่ต้อง "ครอบ" การ
> กระทำทั้งสองฝั่งจริงๆ (จับเวลา, เปิด/ปิด resource, logging แบบ before-after คู่กัน) ไม่ใช่ใช้
> เป็นเครื่องมือหยุด callback chain

---

## Step 343: หยุด Callback Chain ให้ถูกต้อง — `throw :abort` vs ความเชื่อผิดๆ เรื่อง `return false`

นี่คือจุดที่ tutorial เก่าจำนวนมาก (รวมถึงคำตอบเก่าใน Stack Overflow ที่เขียนไว้สมัย Rails 4)
สอนผิดพลาดกันอย่างแพร่หลาย: **"return false จาก before_callback จะหยุดการบันทึก"** — ประโยคนี้
เป็นจริง**เฉพาะ Rails 4 หรือเก่ากว่าเท่านั้น** ตั้งแต่ **Rails 5 เป็นต้นมา การ return false จาก
callback ไม่มีผลใดๆ ต่อ callback chain อีกต่อไป**

### พิสูจน์ด้วยโค้ดจริง

```ruby
class HaltDemo < ActiveRecord::Base
  self.table_name = "callback_demos"

  attr_accessor :should_return_false, :should_throw_abort

  before_save :maybe_return_false
  before_save :maybe_throw_abort
  before_save { puts "  -> before_save ตัวสุดท้าย ทำงานหรือไม่?" }

  def maybe_return_false
    if should_return_false
      puts "  -> maybe_return_false: กำลัง return false"
      false # Rails 5+ : ค่านี้ไม่มีผลใดๆ ต่อ callback chain เลย
    end
  end

  def maybe_throw_abort
    if should_throw_abort
      puts "  -> maybe_throw_abort: กำลัง throw :abort"
      throw :abort
    end
  end
end
```

```ruby
r1 = HaltDemo.new(name: "return false demo", should_return_false: true)
result1 = r1.save
```

ผลลัพธ์จริง:

```
  -> maybe_return_false: กำลัง return false
  -> before_save ตัวสุดท้าย ทำงานหรือไม่?
save คืนค่า: true, persisted?: true
```

สังเกตว่า **แม้ `maybe_return_false` จะ `return false` แต่ callback ตัวถัดไปก็ยังทำงานต่อ และ
`save` ก็ยัง commit สำเร็จ** (`persisted? == true`) — พิสูจน์ชัดเจนว่า `return false` ใน Rails 5+
ไม่มีความหมายพิเศษใดๆ ในบริบท callback (มันแค่เป็นค่า return ธรรมดาของ method ที่ไม่มีใครสนใจ)

```ruby
r2 = HaltDemo.new(name: "throw abort demo", should_throw_abort: true)
result2 = r2.save
```

```
  -> maybe_throw_abort: กำลัง throw :abort
save คืนค่า: false, persisted?: false
```

ในทางกลับกัน `throw :abort` หยุด chain ได้จริง — callback ตัวถัดไป (`before_save` บล็อกสุดท้าย)
**ไม่ทำงานเลย** และ `save` คืนค่า `false` ทันที

### ทำไมถึงเปลี่ยนพฤติกรรมตั้งแต่ Rails 5

Rails team เปลี่ยนพฤติกรรมนี้เพราะปัญหาที่พบบ่อยมากใน Rails 4: บาง method ที่ใช้เป็น callback
บังเอิญ `return false` ธรรมดาโดยไม่ได้ตั้งใจจะหยุดอะไรเลย (เช่น `return false` ท้าย method เป็น
ผลพลอยได้จากการเขียน conditional ผิดรูปแบบ) แล้ว save/update ทั้งระบบพังแบบเงียบๆ โดยไม่มีใคร
ตั้งใจ — การบังคับให้ต้องเขียน `throw :abort` อย่างชัดเจนทำให้เจตนา "อยากหยุด" ต้องเป็นสิ่งที่
นักพัฒนาเขียนขึ้นมาโดยตั้งใจเท่านั้น ไม่ใช่ผลข้างเคียงที่เกิดขึ้นโดยบังเอิญ

> **คำเตือนสำหรับ Part นี้:** ถ้าเจอโค้ดตัวอย่างเก่า (บทความ, วิดีโอ, หรือแม้แต่โค้ดของทีมเก่าที่
> ไม่ได้อัปเดตมานาน) ที่เขียน `before_save { false if some_condition }` แล้วคาดหวังว่าจะหยุด
> save — โค้ดนั้น **ใช้ไม่ได้แล้วตั้งแต่ Rails 5** ต้องแก้เป็น `throw :abort if some_condition`
> เสมอ

### รูปแบบที่ถูกต้องในทางปฏิบัติ

```ruby
class Order < ApplicationRecord
  before_destroy :prevent_destroy_if_paid

  private

  def prevent_destroy_if_paid
    if status == "paid"
      errors.add(:base, "ไม่สามารถลบออเดอร์ที่ชำระเงินแล้วได้")
      throw :abort
    end
  end
end
```

**แนวปฏิบัติที่ดี:** เมื่อ `throw :abort` ควรเรียก `errors.add` คู่กันไปด้วยเสมอ (แม้จะไม่ผ่าน
`valid?` ตามปกติ) เพื่อให้ผู้ที่เรียก `save`/`destroy` แล้วเช็ค `record.errors.full_messages`
สามารถรู้เหตุผลได้ว่าทำไม operation ถึงถูกปฏิเสธ ไม่ใช่แค่ได้ `false` เปล่าๆ กลับมาโดยไม่รู้สาเหตุ

---

## Step 344: Conditional Callback ในทางปฏิบัติ

Part 026 Step 255 สอน `if:`/`unless:` ของ **validation** ไปแล้ว — callback ก็รับตัวเลือกชุด
เดียวกันทุกประการ (symbol, lambda, Array) แต่มีรายละเอียดเชิงปฏิบัติเพิ่มเติมที่สำคัญกับ callback
โดยเฉพาะ

### ใช้ `saved_change_to_*?` เป็นเงื่อนไขที่แม่นยำกว่า `*_changed?` ธรรมดา

```ruby
class Post < ApplicationRecord
  after_update :notify_status_change, if: :saved_change_to_status?

  private

  def notify_status_change
    Rails.logger.info("[Post##{id}] status เปลี่ยนเป็น #{status}")
  end
end
```

`saved_change_to_status?` ต่างจาก `status_changed?` ตรงจุดสำคัญ: **`status_changed?` เช็ค
การเปลี่ยนแปลงที่ยังไม่ถูกบันทึก (in-memory)** ในขณะที่ **`saved_change_to_status?` เช็คว่า
การเปลี่ยนแปลงนั้น ถูกบันทึกลง database ไปแล้วจริง**ๆ ในรอบ `save` ล่าสุด — ความแตกต่างนี้สำคัญ
มากเมื่อใช้ใน `after_*` callback (ซึ่งทำงาน**หลัง** SQL ทำงานไปแล้ว) เพราะถ้าใช้ `status_changed?`
ใน `after_update` มันจะคืน `false` เสมอ (ค่าถูก "confirm" เป็นค่าปัจจุบันไปแล้วตั้งแต่ก่อนเข้า
`after_update`) — นี่เป็นบั๊กที่พบได้บ่อยมากเวลาย้ายโค้ดจาก `before_*` ไป `after_*` โดยลืมเปลี่ยน
ชื่อ method ตาม

| ใช้ในจุดไหน | Method ที่ถูกต้อง |
|---------------|---------------------|
| `before_validation`/`before_save`/`before_update` | `status_changed?`, `status_was` |
| `after_save`/`after_update`/`after_commit` | `saved_change_to_status?`, `status_before_last_save` |

### รวมหลายเงื่อนไขด้วย Array (AND ทั้งหมด)

```ruby
before_save :recalculate_totals, if: [:published?, :price_changed?]
```

Callback นี้ทำงานก็ต่อเมื่อ **ทั้ง `published?` และ `price_changed?` เป็น `true` พร้อมกัน**
(เทียบเท่ากับเขียน `if: -> { published? && price_changed? }` แต่แยกเป็น Array อ่านง่ายกว่าเมื่อ
มีหลายเงื่อนไข)

### `unless:` ที่ใช้เป็น "ประตูหลัง" สำหรับ script ภายใน

```ruby
class Post < ApplicationRecord
  attr_accessor :skip_notification

  after_create_commit :send_new_post_notification, unless: :skip_notification
end
```

```ruby
# ใน seed script ที่ไม่อยากให้ยิง notification จริง 500 ครั้ง
500.times do |i|
  Post.create!(title: "Seed Post #{i}", skip_notification: true, ...)
end
```

รูปแบบนี้ (เพิ่ม `attr_accessor` ที่ไม่ผูกกับ column ในฐานข้อมูล มาใช้เป็น flag ควบคุม callback)
เป็นเทคนิคที่ใช้บ่อยมากในโค้ด production จริง — ปลอดภัยกว่าการปิด callback ทั้งหมดด้วย
`skip_callback` (จะสอนใน Step 350) เพราะเจาะจงเฉพาะ instance ที่ตั้งใจ ไม่กระทบ record อื่นที่
กำลังถูกสร้างพร้อมกันในระบบ (เช่น ถ้ามี background job อื่นกำลังสร้าง Post พร้อมกันอยู่พอดี
`skip_callback` แบบ class-level จะปิด callback ของทุก instance ทั้งระบบชั่วคราว ซึ่งอันตรายกว่า
มาก)

---

## Step 345: Callback Class/Object — แยก Logic ออกมาให้ทดสอบได้อิสระ

Part 026 Step 260 สอนเรื่อง Service Object สำหรับแยก **business logic ที่ orchestrate หลาย
ระบบ** ออกจาก Model ไปแล้ว แต่มี pattern อีกแบบหนึ่งที่เจาะจงกว่านั้น: เมื่อ logic ของ callback
**ตัวเดียว** ซับซ้อนพอที่จะมี branch หลายทาง หรือใช้ซ้ำได้ในหลาย Model — เขียนเป็น **callback
class/object** แยกออกมาได้ โดยไม่ต้องรอจนถึงจุดที่ต้องแยกเป็น Service Object เต็มรูปแบบ

### กลไก: object ใดก็ได้ที่มี method ชื่อตรงกับ callback

```ruby
# app/models/slug_assigner.rb
class SlugAssigner
  def before_save(record)
    return if record.title.blank?
    return if record.slug.present?

    base = record.title.parameterize
    base = "post" if base.blank?
    record.slug = "#{base}-#{SecureRandom.hex(3)}"
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  before_save SlugAssigner.new
end
```

เมื่อ ActiveRecord เจอ **object** (ไม่ใช่ symbol หรือ block) ถูกส่งให้ `before_save` มันจะเรียก
method ที่ชื่อตรงกับ callback นั้น (`before_save`, `after_create`, ฯลฯ) บน object นั้น โดยส่ง
**record ปัจจุบันเป็น argument ตัวแรก** — รูปแบบนี้เรียกว่า **callback object** (บางที่เรียก
"observer-style callback" แม้จะไม่ใช่ `ActiveRecord::Observer` ที่ถูกถอดออกไปตั้งแต่ Rails 4.0
ก็ตาม)

```ruby
author = Author.create!(name: "สมชาย ใจดี")
draft = Post.create!(title: "My Draft Post", body: "เนื้อหาบางส่วน", status: "draft",
                      published: false, author: author)
draft.slug
# => "my-draft-post-1401d4"
```

### ทำไม pattern นี้ถึงมีประโยชน์

1. **ทดสอบ `SlugAssigner` แยกจาก `Post` ได้เลย** — ไม่ต้องสร้าง ActiveRecord object จริง สร้าง
   `OpenStruct`/PORO (Plain Old Ruby Object) ธรรมดาที่มี `title`/`slug` แล้วเรียก
   `SlugAssigner.new.before_save(fake_record)` ตรงๆ ก็ทดสอบได้ครบทุก branch โดยไม่แตะฐานข้อมูล
   เลยแม้แต่ครั้งเดียว:

   ```ruby
   FakeRecord = Struct.new(:title, :slug)

   fake = FakeRecord.new("Hello World", nil)
   SlugAssigner.new.before_save(fake)
   fake.slug # => "hello-world-xxxxxx" (ทดสอบได้โดยไม่ต้องมี Rails/DB เลย)
   ```

2. **ใช้ซ้ำได้หลาย Model** — ถ้ามีทั้ง `Post` และ `Article` ที่ต้องการ auto-slug แบบเดียวกัน
   ก็ `before_save SlugAssigner.new` ได้ทั้งคู่ ไม่ต้อง copy method เดิมไปวางซ้ำ (หรือใช้
   `ActiveSupport::Concern` ซึ่งจะสอนเจาะลึกใน **Part 036** ที่ตามมาถัดจาก Part นี้พอดี)
3. **แยก dependency ที่ inject เข้ามาได้** — เช่นถ้า `SlugAssigner` ต้องพึ่ง external service
   สร้าง slug (เรียก API เช็คว่า slug นี้ไม่ชนกับระบบภายนอก) ก็ inject ผ่าน `initialize` ได้ตรงๆ
   โดยไม่ต้องยุ่งกับ Model เลย: `before_save SlugAssigner.new(checker: ExternalSlugChecker.new)`

> **เมื่อไหร่ใช้ callback object แทน private method ธรรมดา?** ใช้เมื่อ logic นั้นมีความซับซ้อน
> พอที่อยากมี unit test แยกเป็นเอกเทศ, ใช้ซ้ำได้มากกว่า 1 Model, หรือต้องการ inject dependency
> เข้ามา (ทำ dependency injection) — ถ้าเป็น logic ง่ายๆ บรรทัดเดียวจบ การเขียนเป็น private
> method ตรงๆ ใน Model (แบบที่ Part 026 สอน) ยังคงเป็นทางเลือกที่เหมาะสมกว่า ไม่ต้องซับซ้อนเกิน
> ความจำเป็น

---

## Step 346: Callback บน Association — `before_destroy` ของลูก และ `autosave: true`

Callback ไม่ได้ทำงานแค่ในตัว Model เดียวเท่านั้น — เมื่อ Model มีความสัมพันธ์กัน
(`belongs_to`/`has_many` ที่เรียนจาก Part 027 และ Part 033) callback ของแต่ละฝั่งก็ยังทำงาน
ตามปกติ และมีตัวเลือกพิเศษที่ควบคุมว่าจะ "พ่วง" การบันทึก/ลบของฝั่งตรงข้ามอย่างไร

### `before_destroy` ของ Model ลูก ทำงานจริงตอน parent ถูกลบแบบ cascade

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post

  before_destroy :log_before_destroy

  private

  def log_before_destroy
    Rails.logger.info("[Comment] กำลังจะถูกลบ id=#{id} body=#{body.to_s.truncate(20)}")
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy, autosave: true
end
```

```ruby
post = Post.create!(title: "Post With Comments", body: "...", status: "published",
                     published: true, author: author)
post.comments.create!(body: "ความเห็นที่ 1")
post.comments.create!(body: "ความเห็นที่ 2")

post.destroy
```

```
[Comment] กำลังจะถูกลบ id=12 body="ความเห็นที่ 1"
[Comment] กำลังจะถูกลบ id=13 body="ความเห็นที่ 2"
```

```ruby
Comment.where(post_id: post.id).count
# => 0
```

`dependent: :destroy` สั่งให้ ActiveRecord **เรียก `.destroy` กับ comment ทุกตัวทีละรายการ**
ก่อนที่จะลบ `post` เอง — เพราะเป็นการเรียก `.destroy` จริง (ไม่ใช่ `DELETE ... WHERE post_id = ?`
เดียวจบแบบ SQL ตรงๆ) callback `before_destroy`/`after_destroy` ของ `Comment` **ทำงานครบทุกตัว
จริง** นี่คือความแตกต่างสำคัญจาก `dependent: :delete_all` ที่ยิง SQL `DELETE` ตรงๆ ไม่ผ่าน
callback ของ Model ลูกเลยแม้แต่ตัวเดียว (ถ้า `Comment` มี logic สำคัญใน `before_destroy` เช่น
การเช็คสิทธิ์หรือทำ cleanup ที่จำเป็น ต้องใช้ `dependent: :destroy` เท่านั้น ไม่ใช่
`delete_all`)

### `autosave: true` — บันทึกพ่วง record ที่เกี่ยวข้องแม้เป็น record เก่าที่แก้ไข

ปกติ ActiveRecord จะบันทึก **record ใหม่** (`new_record?` เป็น `true`) ที่ถูก build ผ่าน
association ให้อัตโนมัติอยู่แล้วแม้ไม่ใส่ `autosave:` เลย แต่ **record เก่าที่ถูกแก้ไข
(in-memory) จะไม่ถูกบันทึกซ้ำให้อัตโนมัติ** เว้นแต่จะระบุ `autosave: true` ชัดเจน:

```ruby
comment = post.comments.create!(body: "ข้อความเดิม")

comment.body = "ข้อความที่แก้ไขแล้ว (ยังไม่ได้เรียก comment.save ตรงๆ)"
post.title = "เปลี่ยน title ของ post ด้วย"
post.save!

comment.reload.body
# => "ข้อความที่แก้ไขแล้ว (ยังไม่ได้เรียก comment.save ตรงๆ)"
```

เพราะ `has_many :comments, autosave: true` ทำให้ `post.save!` **เดินเข้าไปเช็คทุก object ใน
`post.comments` ที่ถูกโหลดเข้า memory แล้ว ถ้าตัวไหน `changed?` เป็น `true` ก็จะเรียก `save`
ให้อัตโนมัติในธุรกรรมเดียวกัน** — มีประโยชน์มากเวลาต้องการแก้ไข parent พร้อม child หลายตัวใน
คำสั่งเดียว (เช่น หน้าฟอร์มแก้ไขออเดอร์พร้อมรายการสินค้าในออเดอร์ทีเดียว ซึ่งเดี๋ยวจะเจอรูปแบบ
เต็มในเรื่อง `accepts_nested_attributes_for` ที่ Part 037)

> **ข้อควรระวัง:** `autosave: true` ทำให้การ save หนึ่งครั้งอาจแอบบันทึก record อื่นที่ไม่ได้
> ตั้งใจไปด้วย ถ้ามีการแก้ไข object ในหน่วยความจำไว้ก่อนหน้าโดยไม่รู้ตัว (เช่น callback ตัวอื่น
> ที่แก้ attribute ของ association ทิ้งไว้) ควรใช้อย่างมีสติและรู้ผลกระทบเสมอ ไม่ใช่เปิดไว้เป็น
> ค่า default ของทุก association

---

## Step 347: Transaction กับ Callback — Nested Transaction และ `requires_new:`

Part 026 Step 259 แนะนำแล้วว่าทุก `save`/`destroy` ถูกห่อด้วย database transaction โดย
อัตโนมัติ แต่ยังไม่ได้พูดถึงสิ่งที่อันตรายที่สุดเรื่องหนึ่งใน Rails: **nested transaction ที่ไม่มี
`requires_new: true` ไม่ได้ rollback จริงตามที่คิด**

### ทดลองให้เห็นปัญหาจริง

```ruby
author = Author.create!(name: "Nested TX Author")

Post.transaction do
  outer = Post.create!(title: "Outer post", body: "x", status: "draft",
                        published: false, author: author)

  Post.transaction do
    Post.create!(title: "Inner post (no requires_new)", body: "x", status: "draft",
                  published: false, author: author)
    raise ActiveRecord::Rollback
  end
end

Post.where(title: ["Outer post", "Inner post (no requires_new)"]).count
```

ผลลัพธ์จริงที่ได้: **`2`** — ทั้ง `"Outer post"` และ `"Inner post (no requires_new)"` ถูกบันทึก
จริงลงฐานข้อมูล **ทั้งคู่** แม้จะมี `raise ActiveRecord::Rollback` อยู่ในบล็อกด้านในก็ตาม!

**เหตุผลเชิงกลไก:** `Post.transaction do ... end` ที่เขียนซ้อนกันโดยไม่มี `requires_new: true`
**ไม่ได้เปิด transaction ใหม่จริงในระดับฐานข้อมูล** — มันแค่ "เข้าร่วม" (join) transaction ของ
ชั้นนอกที่เปิดอยู่แล้ว ดังนั้นเมื่อ `raise ActiveRecord::Rollback` ถูกเรียกในบล็อกด้านใน Rails
จะจับ exception นี้ไว้เงียบๆ ที่บล็อกนั้น (ทำให้โค้ดหลังบล็อกเดินต่อได้ตามปกติ) **แต่ไม่มีการสั่ง
`ROLLBACK` ไปที่ฐานข้อมูลจริงๆ เลย** เพราะยังอยู่ใน transaction เดียวกันกับชั้นนอกที่ยังไม่ได้
commit — ผลคือ INSERT ที่เพิ่งทำไปในบล็อกด้านในจะถูก commit ไปพร้อมกับชั้นนอกในตอนท้ายเหมือน
ไม่มีอะไรเกิดขึ้น นี่คือกับดักที่ทำให้วิศวกรจำนวนมากเข้าใจผิดว่า "nested transaction rollback
เฉพาะส่วนของตัวเองได้" ทั้งที่ความจริงไม่ใช่แบบนั้นเลย

### ทางแก้: `requires_new: true`

```ruby
Post.transaction do
  outer = Post.create!(title: "Outer post 2", body: "x", status: "draft",
                        published: false, author: author)

  Post.transaction(requires_new: true) do
    Post.create!(title: "Inner post (requires_new)", body: "x", status: "draft",
                  published: false, author: author)
    raise ActiveRecord::Rollback
  end
end

Post.exists?(title: "Outer post 2")
# => true

Post.exists?(title: "Inner post (requires_new)")
# => false
```

`requires_new: true` สั่งให้ ActiveRecord สร้าง **SQL SAVEPOINT** จริงในฐานข้อมูล ทำให้บล็อก
ด้านในกลายเป็นหน่วยที่ rollback แยกเป็นอิสระได้จริงโดยไม่กระทบชั้นนอก — `"Outer post 2"` ยังอยู่
(เพราะ transaction ชั้นนอกยัง commit ได้ตามปกติ) แต่ `"Inner post (requires_new)"` หายไปจริง
(เพราะถูก rollback กลับไปที่ savepoint)

### กฎปฏิบัติ

> **ถ้าต้องการให้ nested transaction บล็อกใดบล็อกหนึ่ง rollback ได้อย่างอิสระโดยไม่กระทบ
> transaction ชั้นนอก ต้องใส่ `requires_new: true` เสมอ ไม่มีข้อยกเว้น** ถ้าลืมใส่ พฤติกรรมที่
> ได้จะดูเหมือนใช้งานได้ปกติในเทสต์ง่ายๆ (เพราะไม่มีใคร assert การมีอยู่ของ record ในบล็อกด้านใน
> โดยตรง) แต่จะกลายเป็นบั๊กที่ผิดเงียบๆ ใน production เมื่อข้อมูลที่ควรถูกยกเลิกกลับถูกบันทึกจริง

### ความสัมพันธ์กับ `after_commit`/`after_rollback`

จาก Step 341 เราเห็นแล้วว่า `after_commit` ทำงานเมื่อ transaction **ทั้งก้อน** (นับทุกชั้นที่
ซ้อนกัน) commit สำเร็จจริง ส่วน `after_rollback` ทำงานเมื่อธุรกรรมนั้นถูกยกเลิกจริง — ประเด็นคือ
ถ้าใช้ nested transaction โดยไม่มี `requires_new: true` อย่างที่เห็นข้างบน `after_commit` ของ
record ที่ "คิดว่า" ถูก rollback ไปแล้ว **จะถูกเรียกจริงตอนชั้นนอก commit** เพราะข้อมูลนั้นถูก
commit จริงนั่นเอง — เป็นเหตุผลเพิ่มเติมว่าทำไมถ้าเขียน nested transaction แล้วคาดหวัง rollback
เฉพาะจุด ต้องใส่ `requires_new: true` ให้ครบทุกครั้ง ไม่เช่นนั้น side effect ใน `after_commit`
(ส่งอีเมล, enqueue job) จะทำงานทั้งที่ข้อมูลที่เกี่ยวข้อง "ควร" ถูกยกเลิกไปแล้วตามที่ตั้งใจ

### Automatic rollback เมื่อเกิด Exception ทั่วไป

ไม่จำเป็นต้อง `raise ActiveRecord::Rollback` เสมอไป — **exception อะไรก็ได้** ที่เกิดขึ้นภายใน
`transaction do...end` (รวมถึง custom exception ของเราเอง) จะทำให้ transaction rollback
อัตโนมัติเหมือนกัน เพียงแต่ exception นั้นจะยัง**ถูก re-raise ต่อออกไปนอกบล็อก** (ต่างจาก
`ActiveRecord::Rollback` ที่ถูกจับไว้เงียบๆ ไม่ re-raise ต่อ):

```ruby
class StockError < StandardError; end

begin
  Product.transaction do
    product.update!(stock: product.stock - 100)  # สมมติทำให้ stock ติดลบ
    raise StockError, "สต๊อกไม่พอ" if product.stock.negative?
  end
rescue StockError => e
  puts "จับ error ได้: #{e.message}"
end

product.reload.stock
# กลับเป็นค่าเดิมก่อนเข้า transaction เพราะ StockError ทำให้ทั้งบล็อก rollback อัตโนมัติ
```

รูปแบบนี้ (raise custom exception ธรรมดาแทน `ActiveRecord::Rollback`) เป็นแนวทางที่แนะนำมากกว่า
ในโค้ด production เพราะทำให้โค้ดที่เรียก transaction รู้**สาเหตุที่แท้จริง**ของการ rollback ผ่าน
`rescue` ได้ตรงจุด ไม่ใช่แค่รู้ว่า "ถูกยกเลิก" เฉยๆ แบบ `ActiveRecord::Rollback` — จะใช้ pattern
นี้เต็มรูปแบบในแบบฝึกหัดท้าย Part นี้

---

## Step 348: `touch: true` — Cascade `updated_at` ขึ้นไปยัง Parent

`touch: true` เป็นตัวเลือกของ `belongs_to` ที่ทำให้ ActiveRecord อัปเดต `updated_at` (หรือ
column อื่นที่ระบุ) ของ **parent** โดยอัตโนมัติทุกครั้งที่ child ถูกสร้าง แก้ไข หรือลบ — เป็น
กลไกระดับ database ที่แยกจาก callback ทั่วไป แต่ทำงานร่วมกับ lifecycle เดียวกัน

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author, touch: true
end
```

```ruby
author = Author.create!(name: "ผู้เขียนทดสอบ")
post = Post.create!(title: "...", body: "...", status: "published", published: true, author: author)

author.updated_at
# => 2026-09-26 03:34:25 UTC

sleep 1
post.update!(body: "แก้ไขเนื้อหาอีกครั้ง")

author.reload.updated_at
# => 2026-09-26 03:34:26 UTC  (เปลี่ยนไปแล้ว แม้ไม่มีใครแก้ไข author โดยตรงเลย)
```

ทุกครั้งที่ `post.save` ทำงานสำเร็จ (ไม่ว่าจะสร้างหรือแก้ไข) ActiveRecord จะยิง
`UPDATE authors SET updated_at = ... WHERE id = ?` เพิ่มให้อัตโนมัติในธุรกรรมเดียวกัน — ถ้าลบ
`post` ก็เช่นกัน `author.updated_at` จะถูกอัปเดตด้วย

### Use case จริงที่สำคัญที่สุด: Cache Invalidation แบบ Russian-doll

เหตุผลหลักที่ `touch: true` ถูกออกแบบมาคือเพื่อรองรับรูปแบบการทำ **fragment caching** ที่ผูก
cache key เข้ากับ `updated_at` ของ object (`cache post do ... end` ใน view จะสร้าง cache key
จาก `post.cache_key_with_version` ซึ่งอ้างอิง `updated_at`) เมื่อหน้าเว็บแสดงรายชื่อผู้เขียน
พร้อมจำนวนโพสต์ล่าสุด ถ้า cache ของหน้า "ผู้เขียน" ผูกกับ `author.updated_at` เฉยๆ แต่ไม่รู้เลย
ว่ามีโพสต์ใหม่ถูกเพิ่มเข้ามา (เพราะแก้แค่ตาราง `posts` ไม่ได้แตะตาราง `authors` เลย) หน้าเว็บนั้น
จะแสดงข้อมูลเก่าค้างอยู่ (stale cache) โดยไม่รู้ตัว — `touch: true` แก้ปัญหานี้โดยทำให้
`author.updated_at` "ขยับ" ทุกครั้งที่มีการเปลี่ยนแปลงใน `posts` ที่เกี่ยวข้อง ทำให้ cache key
เปลี่ยนตามไปด้วยโดยอัตโนมัติ — เรื่อง caching แบบเต็มรูปแบบจะสอนลึกกว่านี้มากใน **Part 063**
แต่ควรรู้จักความเชื่อมโยงกับ `touch:` ไว้ตั้งแต่ตอนนี้

### ระวัง: `touch: true` เพิ่ม query และอาจชนกับ callback ตัวอื่นของ parent

ทุกครั้งที่ `touch:` ทำงาน มันจะ **trigger callback ของ parent เองด้วย** (`before_save`,
`after_save`, `after_commit` ของ `Author` ในตัวอย่างนี้) เพราะเบื้องหลังคือการเรียก `.touch`
ซึ่งก็คือการ `save` ตามปกติของ parent นั่นเอง — ถ้า `Author` มี callback หนักๆ (เช่น
reindex search, ส่ง notification) การแก้ไข `Post` ทีละนิดจำนวนมากๆ (เช่น bulk import 10,000
แถว) จะไป trigger callback หนักๆ ของ `Author` ซ้ำ 10,000 ครั้งโดยไม่ตั้งใจ — วิธีป้องกันคือใช้
`ActiveRecord::Base.no_touching` ห่อ block ที่ไม่ต้องการให้ touch ทำงาน (จะสาธิตเต็มรูปแบบใน
Step 350)

---

## Step 349: กับดักจริง — Callback วนเรียกตัวเองไม่รู้จบ (Infinite Callback Loop)

นี่คือบั๊กระดับ production ที่ทำให้ server ค้างหรือ crash ด้วย `SystemStackError` และเป็นเหตุผล
หนึ่งที่ Part 026 เตือนไว้ว่า "callback ที่แก้ไข attribute ของตัวเองแบบไม่มีเงื่อนไข" อันตราย
มาก มาดูสถานการณ์จริงที่ทำให้เกิดปัญหานี้

### สร้างปัญหาให้เห็นจริง (ระวัง: โค้ดนี้ทำให้เกิด `SystemStackError` จริง — สาธิตเพื่อการศึกษา
เท่านั้น ห้ามใช้ pattern นี้ในโค้ดจริง)

```ruby
class LoopyPost < ActiveRecord::Base
  self.table_name = "posts"

  belongs_to :author

  # อันตราย: after_update ที่เรียก update ตัวเองแบบไม่มีเงื่อนไข
  # -> เรียก after_update ซ้ำ -> เรียก update อีก -> วนไม่รู้จบ
  after_update :recalc_dangerous

  def recalc_dangerous
    update(body: "recalculated at #{Time.current}")
  end
end
```

```ruby
post = LoopyPost.create!(title: "Loop test", body: "x", status: "draft",
                          published: false, author: author)
post.update(title: "trigger the loop")
```

ผลลัพธ์จริง: **`SystemStackError: stack level too deep`** — `recalc_dangerous` เรียก `update`
ซึ่ง trigger `after_update` อีกครั้ง ซึ่งเรียก `recalc_dangerous` อีกครั้ง วนซ้ำจนกองซ้อน
(call stack) ของ Ruby เต็มและ crash โปรแกรมทันที ต่างจาก loop เดินไม่รู้จบทั่วไปที่อาจกิน CPU
ไปเรื่อยๆ อย่างเงียบๆ ความจริง `SystemStackError` จะเกิดขึ้น**เร็วมาก** (เสี้ยววินาที) เพราะ
memory ของ call stack เต็มก่อน CPU จะทันทำงานหนักด้วยซ้ำ

### สาเหตุที่แท้จริง

`update` เรียกซ้ำ **ทุก field ของ record เดียวกัน** โดยไม่มีเงื่อนไขใดๆ คั่นไว้ — ทุกครั้งที่
`update` สำเร็จ จะ trigger `after_update` ใหม่อีกรอบเสมอ (เพราะ ActiveRecord ไม่รู้ว่า "นี่คือ
การ update ที่มาจาก callback ของ update ก่อนหน้า" — มันแค่เห็นว่ามีการเรียก `update` เข้ามาใหม่
เท่านั้น)

### ทางแก้ที่ถูกต้อง: guard ด้วยเงื่อนไข + `update_columns` เพื่อตัดวงจร

```ruby
class SafePost < ActiveRecord::Base
  self.table_name = "posts"

  belongs_to :author

  after_update :recalc_safe, if: :saved_change_to_title?

  def recalc_safe
    # update_columns เขียนลง DB ด้วย UPDATE เดียวตรงๆ ไม่ trigger callback/validation ใดๆ อีก
    # จึงตัดวงจรไม่ให้เรียก after_update ซ้ำ
    update_columns(body: "recalculated safely at #{Time.current}")
  end
end
```

```ruby
post = SafePost.create!(title: "Safe loop test", body: "x", status: "draft",
                         published: false, author: author)
post.update!(title: "trigger once")
post.reload.body
# => "recalculated safely at 2026-09-26 03:36:28 UTC"

post.update!(status: "published")   # แก้ field อื่นที่ไม่ใช่ title
post.reload.body
# => "recalculated safely at 2026-09-26 03:36:28 UTC" (ไม่เปลี่ยน เพราะ guard กันไว้)
```

การแก้มีสองชั้นป้องกันซ้อนกันโดยตั้งใจ:

1. **`if: :saved_change_to_title?`** — จำกัดให้ callback ทำงานเฉพาะตอน `title` เปลี่ยนจริง
   ไม่ใช่ทุกครั้งที่มีการ update field ใดก็ตาม (ตัดสาเหตุตั้งแต่ต้นทาง)
2. **`update_columns`** แทน `update` — ยิง SQL `UPDATE` ตรงไปที่ database โดย**ข้าม
   validation และ callback ทั้งหมด** (รวมถึงตัวมันเองด้วย) จึงไม่มีทางเข้า `after_update` ซ้ำ
   ได้อีกไม่ว่าเงื่อนไขข้อ 1 จะเผลอพลาดหรือไม่ก็ตาม (defense in depth)

> **กฎจำง่าย:** callback ที่แก้ไข attribute ของ **ตัวเองใน table เดียวกัน** ต้องมีเงื่อนไข
> (`if:`) เจาะจงเสมอว่าจะทำงานเมื่อไหร่ และควรพิจารณาใช้ `update_column`/`update_columns`
> (เขียนตรงไม่ผ่าน callback) แทน `save`/`update` ธรรมดา เพื่อป้องกันการวนซ้ำโดยเด็ดขาด — หลักการ
> เดียวกันนี้ใช้ได้กับกรณีที่ callback ของ Model สองตัวเรียกกันไปมา (เช่น `Order` แก้ `Product`
> แล้ว `Product` มี callback แก้ `Order` กลับ) ซึ่งตามรอยยากกว่ามากเพราะ error trace จะกระโดด
> ข้าม Model ไปมา ไม่ได้อยู่ใน method เดียวให้เห็นชัดเหมือนตัวอย่างนี้

---

## Step 350: การเทสต์ Model ที่มี Callback

Part 018/019 สอนพื้นฐาน Minitest/RSpec ไปแล้ว มาดูว่าเมื่อ Model มี callback เยอะขึ้น ควรเขียน
เทสต์อย่างไรให้ยัง**เชื่อถือได้** โดยไม่ทำให้เทสต์ช้าหรือเปราะบางเกินจำเป็น

### เครื่องมือที่ใช้ปิด Callback ชั่วคราว

**1) `skip_callback`/`set_callback`** — ปิด/เปิด callback ที่ระดับ class ทั้งหมด (มีผลกับทุก
instance จนกว่าจะเปิดกลับ):

```ruby
Order.skip_callback(:update, :before, :prevent_cancelling_shipped_order)
begin
  shipped_order.update!(status: "cancelled")  # ผ่านได้แล้วเพราะ callback ถูกปิด
ensure
  Order.set_callback(:update, :before, :prevent_cancelling_shipped_order)
end
```

**2) `ActiveRecord::Base.no_touching`** — ปิดเฉพาะ `touch:` cascade ชั่วคราว (ไม่กระทบ
callback อื่น):

```ruby
ActiveRecord::Base.no_touching do
  post.update!(body: "แก้ไขภายใต้ no_touching")
end
author.reload.updated_at
# ไม่เปลี่ยน แม้ post ที่เป็นลูกจะถูกแก้ไขไปแล้วก็ตาม
```

**3) `Model.suppress`** — ปิดการบันทึกทั้งหมดของ instance ที่สร้างขึ้นภายใน block (มีประโยชน์
ตอน seed ข้อมูลที่ association สร้าง record ซ้ำโดยไม่ตั้งใจ) — ทดสอบพฤติกรรมนี้ได้เองด้วยการอ่าน
เอกสาร `ActiveRecord::Suppressor`

### เมื่อไหร่ "ควร" skip callback ในเทสต์

- **เขียนเทสต์ของ Model A แต่ callback ของมันไปยุ่งกับ Model B ที่ไม่เกี่ยวข้องกับสิ่งที่กำลัง
  ทดสอบเลย** เช่น ทดสอบว่า `Post#title` ถูก normalize ถูกต้อง ไม่จำเป็นต้องให้
  `after_create_commit :send_welcome_email` ทำงานจริงทุกครั้งที่รันเทสต์ (ควร stub/mock
  mailer แทนการปิด callback ทั้งหมด — ปิด callback ทั้งก้อนหยาบเกินไป)
- **สคริปต์ import ข้อมูลเก่าจำนวนมาก (data migration)** ที่รู้แน่ชัดว่าข้อมูลผ่านการตรวจสอบ
  มาแล้วจากระบบเดิม ไม่จำเป็นต้องรัน callback ที่ทำงานหนัก (reindex, ส่งอีเมล) ซ้ำอีกรอบสำหรับ
  ข้อมูลเก่าที่ไม่ใช่เหตุการณ์ใหม่จริงๆ

### เมื่อไหร่ "ห้าม" skip callback ในเทสต์

- **ห้าม skip callback ที่เป็น business rule ที่กำลังถูกทดสอบอยู่โดยตรง** เช่น ถ้ากำลังเขียน
  เทสต์ยืนยันว่า "ห้ามยกเลิกออเดอร์ที่ shipped แล้ว" (`prevent_cancelling_shipped_order`) การ
  skip callback ตัวนั้นในเทสต์เท่ากับ**ลบการทดสอบสิ่งที่สำคัญที่สุดออกไป** เทสต์จะผ่านเสมอไม่ว่า
  โค้ดจริงจะพังหรือไม่ก็ตาม (false confidence ที่อันตรายกว่าไม่มีเทสต์เลยด้วยซ้ำ)
- **ห้าม skip `after_commit` แล้วสรุปว่า notification "ทำงานถูกต้อง"** — ถ้าเทสต์ต้องการยืนยันว่า
  ระบบส่ง notification จริงตอน commit สำเร็จ ต้อง**ทดสอบว่ามันถูกเรียกจริง** (ผ่านการ stub ตัว
  mailer/job แล้วเช็คว่าถูกเรียก ไม่ใช่ปิด callback ทิ้งแล้วสมมติว่ามันคงทำงานถูกต้อง)

### ตัวอย่างการเขียนเทสต์ที่ stub อย่างถูกจุด (รูปแบบ Minitest ตามที่ Part 018 สอน)

```ruby
# test/models/order_test.rb
require "test_helper"

class OrderTest < ActiveSupport::TestCase
  test "ship! หักสต๊อกสินค้าและเปลี่ยนสถานะเป็น shipped เมื่อสต๊อกพอ" do
    product = Product.create!(name: "Widget", stock: 10)
    order = Order.create!(status: "paid", total_cents: 1000)
    order.line_items.create!(product: product, quantity: 3)

    order.ship!

    assert_equal "shipped", order.reload.status
    assert_equal 7, product.reload.stock
  end

  test "ship! rollback ทั้งหมดเมื่อสต๊อกไม่พอ" do
    product = Product.create!(name: "Widget", stock: 1)
    order = Order.create!(status: "paid", total_cents: 1000)
    order.line_items.create!(product: product, quantity: 5)

    assert_raises(Order::InsufficientStockError) { order.ship! }

    assert_equal "paid", order.reload.status   # ยังไม่เปลี่ยนเป็น shipped
    assert_equal 1, product.reload.stock       # สต๊อกไม่ถูกหักเลยแม้แต่หน่วยเดียว
  end

  test "cancel! ถูกปฏิเสธถ้าออเดอร์ shipped ไปแล้ว (ทดสอบ throw :abort ตรงๆ ไม่ skip)" do
    product = Product.create!(name: "Widget", stock: 10)
    order = Order.create!(status: "paid", total_cents: 1000)
    order.line_items.create!(product: product, quantity: 1)
    order.ship!

    assert_raises(ActiveRecord::RecordNotSaved) { order.cancel! }
    assert_equal "shipped", order.reload.status
  end

  test "เปลี่ยนสถานะสำเร็จแล้วต้อง enqueue การแจ้งเตือน (ทดสอบ after_commit ด้วย stub ไม่ skip)" do
    order = Order.create!(status: "pending", total_cents: 1000)

    order.stub(:send_status_notification, -> { order.instance_variable_set(:@notified, true) }) do
      order.update!(status: "paid")
    end

    assert order.instance_variable_get(:@notified)
  end
end
```

สังเกตว่าเทสต์ทั้งหมดนี้ **ไม่มีตัวไหน skip callback เลย** เพราะทุกเทสต์กำลังทดสอบพฤติกรรมของ
callback นั้นๆ โดยตรง — การ stub ในเทสต์สุดท้ายก็ยัง**ปล่อยให้ `after_commit` ทำงานจริง** เพียง
แค่แทนที่ method ที่จะไปเรียก external service ด้วย method ปลอมชั่วคราว (ผ่าน `Minitest::Mock`
ที่ Part 018 แนะนำไว้) ไม่ได้ปิด callback ทั้งกลไก — นี่คือความแตกต่างสำคัญระหว่าง **"stub สิ่งที่
callback เรียก"** (ปลอดภัย ยังทดสอบว่า callback ทำงานจริง) กับ **"skip ตัว callback เอง"**
(อันตราย เพราะลบการทดสอบพฤติกรรมที่สำคัญที่สุดออกไปเลย)

---

## แบบฝึกหัด: `Order` Model พร้อม State-transition Callback ที่ปลอดภัย

### โจทย์

สร้าง Model `Order` ที่มี:

1. `status` (`pending`/`paid`/`shipped`/`cancelled`) และมี `line_items` (`has_many`) แต่ละอัน
   ผูกกับ `Product` (`stock` เป็นจำนวนสินค้าคงเหลือ)
2. **Callback บันทึก log ทุกครั้งที่ status เปลี่ยน** (`before_update`)
3. **`ship!` method** ที่ใช้ transaction หักสต๊อกสินค้าของทุก `line_item` แบบ atomic (ถ้า
   สินค้าตัวใดตัวหนึ่งสต๊อกไม่พอ ต้อง rollback ทั้งหมด ไม่หักสต๊อกตัวอื่นไปครึ่งๆ กลางๆ) แล้ว
   ค่อยเปลี่ยนสถานะเป็น `"shipped"`
4. **`after_commit` จำลองการส่งแจ้งเตือนลูกค้า** ทุกครั้งที่ status เปลี่ยนสำเร็จจริง
5. **`throw :abort` ป้องกันการยกเลิกออเดอร์ที่ `shipped` ไปแล้ว**

### เฉลย

**1) Migration และ Model**

```bash
bin/rails generate model Product name:string stock:integer
bin/rails generate model Order status:string total_cents:integer
bin/rails generate model LineItem order:references product:references quantity:integer
bin/rails db:migrate
```

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  has_many :line_items

  validates :stock, numericality: { greater_than_or_equal_to: 0 }
end
```

```ruby
# app/models/line_item.rb
class LineItem < ApplicationRecord
  belongs_to :order
  belongs_to :product
end
```

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  class InsufficientStockError < StandardError; end

  STATUSES = %w[pending paid shipped cancelled].freeze

  has_many :line_items, dependent: :destroy
  has_many :products, through: :line_items

  validates :status, inclusion: { in: STATUSES }

  before_update :prevent_cancelling_shipped_order, if: :status_changed?
  before_update :log_status_transition, if: :status_changed?

  after_commit :send_status_notification, on: :update, if: :saved_change_to_status?

  def ship!
    transaction do
      line_items.each do |line_item|
        product = line_item.product.lock!   # pessimistic lock กันสอง request แย่ง stock กัน

        if product.stock < line_item.quantity
          raise InsufficientStockError,
                "สินค้า #{product.name} มีไม่พอ (คงเหลือ #{product.stock}, ต้องการ #{line_item.quantity})"
        end

        product.update!(stock: product.stock - line_item.quantity)
      end

      update!(status: "shipped")
    end
  end

  def cancel!
    update!(status: "cancelled")
  end

  private

  def log_status_transition
    Rails.logger.info("[Order##{id}] status: #{status_was.inspect} -> #{status.inspect}")
  end

  def prevent_cancelling_shipped_order
    if status_was == "shipped" && status == "cancelled"
      errors.add(:status, "ไม่สามารถยกเลิกออเดอร์ที่จัดส่งไปแล้วได้")
      throw :abort
    end
  end

  def send_status_notification
    # ระบบจริงจะ enqueue ActiveJob เพื่อส่งอีเมล/SMS ยืนยัน ที่นี่จำลองด้วย logger
    Rails.logger.info("[Notification] แจ้งลูกค้า: ออเดอร์ ##{id} เปลี่ยนสถานะเป็น #{status} แล้ว")
  end
end
```

**อธิบายจุดสำคัญของเฉลย:**

- **ลำดับของ `before_update` สองตัวมีผลจริง:** `prevent_cancelling_shipped_order` ถูกวางไว้
  **ก่อน** `log_status_transition` โดยตั้งใจ — เพื่อให้การพยายามยกเลิกออเดอร์ที่ shipped แล้วถูก
  บล็อกด้วย `throw :abort` **ก่อน** ที่จะมีการ log ว่า "กำลังจะเปลี่ยนสถานะ" เกิดขึ้น (ถ้าสลับ
  ลำดับกัน log จะถูกบันทึกไปแล้วทั้งที่การเปลี่ยนแปลงไม่เคยเกิดขึ้นจริง — ตรงกับหลักการใน
  Step 341 ที่ว่าลำดับการประกาศ callback ประเภทเดียวกันมีผลต่อพฤติกรรมจริงเสมอ)
- **`product.lock!`** ใช้ pessimistic locking (`SELECT ... FOR UPDATE`) ก่อนเช็คสต๊อก เพื่อกัน
  race condition แบบเดียวกับที่ Part 026 เตือนไว้เรื่อง `uniqueness` validator — ถ้าไม่ lock
  สอง request ที่สั่งซื้อสินค้าตัวสุดท้ายพร้อมกันอาจอ่านค่า `stock` เดิมพร้อมกันแล้วหักซ้ำจนติดลบ
  ได้ในทางทฤษฎี
- **`raise InsufficientStockError`** แทน `raise ActiveRecord::Rollback` — ตามหลักการ Step 347
  เพราะโค้ดที่เรียก `ship!` ต้องรู้**สาเหตุที่แท้จริง**ของความล้มเหลว (สต๊อกไม่พอ) ไม่ใช่แค่รู้ว่า
  "ถูกยกเลิก" เฉยๆ
- **`send_status_notification` อยู่ใน `after_commit`** ไม่ใช่ `after_update` ตามหลักการ Part 026
  Step 259 — ถ้า `ship!` ล้มเหลวกลางคันเพราะสต๊อกไม่พอ (transaction rollback) จะไม่มีการแจ้งเตือน
  ลูกค้าผิดๆ ว่า "สินค้าจัดส่งแล้ว" ทั้งที่ยังไม่ได้ส่งจริง

**2) ทดสอบผ่าน `bin/rails runner`**

```ruby
widget = Product.create!(name: "Widget", stock: 10)
gadget = Product.create!(name: "Gadget", stock: 2)

order = Order.create!(status: "pending", total_cents: 5000)
order.line_items.create!(product: widget, quantity: 3)
order.line_items.create!(product: gadget, quantity: 1)

order.update!(status: "paid")
order.ship!

order.reload.status        # => "shipped"
widget.reload.stock        # => 7
gadget.reload.stock        # => 1
```

```
[Order#1] status: "pending" -> "paid"
[Notification] แจ้งลูกค้า: ออเดอร์ #1 เปลี่ยนสถานะเป็น paid แล้ว
[Order#1] status: "paid" -> "shipped"
[Notification] แจ้งลูกค้า: ออเดอร์ #1 เปลี่ยนสถานะเป็น shipped แล้ว
```

**กรณีสต๊อกไม่พอ — ต้อง rollback ทั้งหมด:**

```ruby
order2 = Order.create!(status: "pending", total_cents: 2000)
order2.line_items.create!(product: gadget, quantity: 5)  # มีแค่ 1 ชิ้น ไม่พอ
order2.update!(status: "paid")

begin
  order2.ship!
rescue Order::InsufficientStockError => e
  puts "จับ error ได้: #{e.message}"
end

order2.reload.status   # => "paid" (ไม่เปลี่ยนเป็น shipped)
gadget.reload.stock    # => 1 (ไม่ถูกหักเลย แม้แต่หน่วยเดียว)
```

```
จับ error ได้: สินค้า Gadget มีไม่พอ (คงเหลือ 1, ต้องการ 5)
```

**กรณี `throw :abort` — ห้ามยกเลิกออเดอร์ที่ shipped แล้ว:**

```ruby
begin
  order.cancel!
rescue ActiveRecord::RecordNotSaved => e
  puts "ถูกป้องกันไว้: #{e.message}"
end

order.reload.status   # => "shipped" (ยังเป็น shipped อยู่ ไม่ถูกยกเลิก)
```

```
ถูกป้องกันไว้: Failed to save the record
```

สังเกตว่าไม่มี log บรรทัด `[Order#1] status: "shipped" -> "cancelled"` ปรากฏขึ้นเลย ยืนยันว่า
`prevent_cancelling_shipped_order` (ที่ถูกจัดลำดับไว้ก่อน) บล็อกไว้ได้ทันก่อนที่
`log_status_transition` จะได้ทำงาน

**ยกเลิกออเดอร์ที่ยังไม่ shipped ทำได้ตามปกติ:**

```ruby
order2.cancel!
order2.reload.status
# => "cancelled"
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม method `Order#refund!` ที่คืนสต๊อกสินค้ากลับเข้าคลัง (ต้องอยู่ใน transaction เดียวกับ
   การเปลี่ยนสถานะเป็น `"refunded"` เพิ่มเข้าไปใน `STATUSES`) และป้องกันไม่ให้ refund ออเดอร์ที่
   ยังไม่เคย `"shipped"` ด้วย `throw :abort` (ใบ้: เขียน guard คล้ายกับ
   `prevent_cancelling_shipped_order` แต่ตรวจเงื่อนไขตรงข้ามกัน)
2. เขียน callback object แยกต่างหากชื่อ `OrderNumberGenerator` (แบบเดียวกับ `SlugAssigner` ใน
   Step 345) ที่ generate เลขที่ออเดอร์รูปแบบ `"ORD-XXXXXX"` ให้ `Order` ก่อนบันทึกครั้งแรก
   เท่านั้น แล้วเขียนเทสต์แยกทดสอบ callback object ตัวนี้โดยสร้าง fake object (`Struct`) ที่มี
   attribute แค่ `order_number` พอ ไม่ต้องพึ่ง `Order`/ฐานข้อมูลจริงเลย
3. จำลองสถานการณ์ infinite callback loop ข้าม 2 Model ด้วยตัวเอง (ต่างจาก Step 349 ที่วนใน
   Model เดียว): ให้ `Order#after_update` ไปแก้ `total_cents` ของ record ใน `LineItem` แบบไม่มี
   เงื่อนไข แล้วให้ `LineItem#after_update` มี callback ย้อนกลับมาแก้ `Order#total_cents` อีกที
   — รันดูให้เห็น `SystemStackError` จริงก่อน แล้วแก้ด้วย guard condition +
   `update_columns` ตามหลักการ Step 349 ให้ทั้งสอง Model อัปเดตกันได้โดยไม่วนซ้ำ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **ตารางลำดับ callback ฉบับสมบูรณ์** ครอบคลุมทั้ง `around_*` ที่ Part 026 ยังไม่ได้พูดถึง —
  `around_save` ครอบ `around_create`/`around_update` อีกที เพราะ `save` ครอบ `create`/`update`
  เสมอไม่ว่าจะเป็น callback ประเภทไหน
- **`around_*` callback** ใช้ `yield` เป็นจุดแบ่งก่อน/หลัง เหมาะกับงานจับเวลา, เปิด/ปิด resource
  — และมีกับดักสำคัญ: **`throw :abort` ใช้ไม่ได้ใน `around_*`** ต้องหยุดด้วยการไม่เรียก `yield`
  แทน
- **`throw :abort` คือวิธีหยุด callback chain ที่ถูกต้องตั้งแต่ Rails 5 เป็นต้นมา** — ส่วน
  `return false` **ไม่มีผลใดๆ อีกต่อไป** พิสูจน์ด้วยโค้ดจริงว่า callback ตัวถัดไปยังทำงานต่อและ
  save ยัง commit สำเร็จแม้ callback ก่อนหน้าจะ `return false`
- **Conditional callback** ควรใช้ `saved_change_to_*?` ใน `after_*` (ไม่ใช่ `*_changed?` ที่ใช้
  ใน `before_*`) และใช้ instance flag (`attr_accessor` ที่ไม่ผูกกับ DB) เป็น "ประตูหลัง" สำหรับ
  ปิด callback เฉพาะ instance แทนการปิดทั้ง class
- **Callback class/object** (`before_save SomeObject.new`) แยก logic ของ callback ที่ซับซ้อน
  หรือใช้ซ้ำได้ออกจาก Model ให้ทดสอบเป็นเอกเทศได้ โดยไม่ต้องรอจนถึงขั้นต้องแยกเป็น Service Object
  เต็มรูปแบบ
- **Callback บน association**: `dependent: :destroy` เรียก callback ของลูกจริงทุกตัว (ต่างจาก
  `delete_all` ที่ข้าม callback ไปเลย) และ `autosave: true` บันทึกพ่วง record ที่ถูกแก้ไขใน
  หน่วยความจำแม้จะไม่ใช่ record ใหม่
- **Nested transaction ที่ไม่มี `requires_new: true` ไม่ได้ rollback จริง** — พิสูจน์ด้วยโค้ด
  จริงว่าข้อมูลที่ "ควรถูกยกเลิก" กลับถูก commit จริงไปพร้อมชั้นนอก ต้องใส่ `requires_new: true`
  เสมอถ้าต้องการให้ rollback เฉพาะจุดทำงานได้จริง
- **`touch: true`** cascade `updated_at` ของ parent ขึ้นไปโดยอัตโนมัติ ใช้ทำ cache invalidation
  แบบ Russian-doll ได้ แต่ต้องระวังว่ามัน trigger callback ของ parent เองด้วย
- **Infinite callback loop** เกิดจาก callback ที่แก้ไข attribute ของตัวเองแบบไม่มีเงื่อนไข แก้ได้
  ด้วย guard condition (`if: :saved_change_to_*?`) ร่วมกับ `update_columns` เพื่อตัดวงจรอย่าง
  เด็ดขาด
- **เทสต์ Model ที่มี callback**: `skip_callback`/`no_touching` ใช้เมื่อ callback นั้นไม่เกี่ยว
  กับสิ่งที่กำลังทดสอบ แต่**ห้าม skip callback ที่เป็น business rule ที่กำลังถูกทดสอบอยู่โดยตรง**
  — ควร stub สิ่งที่ callback เรียก (เช่น mailer) แทนการปิดตัว callback เอง

**ต่อไป (Part 036):** ตอนนี้ `Order` มี callback, validation, และ method หลายตัวรวมกันอยู่ใน
ไฟล์เดียว ถ้า Model ใหญ่ขึ้นเรื่อยๆ (มี state machine เต็มรูปแบบ, มี logic เกี่ยวกับการคำนวณราคา,
มี logic เกี่ยวกับการจัดส่ง) ไฟล์เดียวจะยาวจนอ่านยากและดูแลลำบาก Part 036 จะสอน
**`ActiveSupport::Concern`** — กลไกมาตรฐานของ Rails สำหรับแบ่งโมดูลขนาดใหญ่ออกเป็นส่วนๆ ที่มี
ความรับผิดชอบชัดเจน (เช่น แยก `Order::StateTransitions`, `Order::Notifiable` ออกจากไฟล์หลัก)
โดยยังคง behavior เดิมทุกประการ พร้อมข้อควรระวังว่าเมื่อไหร่ `Concern` ช่วยได้จริง และเมื่อไหร่
มันแค่ซ่อนปัญหาของ fat model ไว้ใต้พรมโดยไม่ได้แก้อะไรเลย
