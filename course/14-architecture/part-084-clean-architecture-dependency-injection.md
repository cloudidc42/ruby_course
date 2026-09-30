# Part 084: Clean Architecture / Hexagonal ใน Rails, Dependency Injection เบื้องต้น

> **Step ครอบคลุมใน Part นี้:** Step 831–840
> **ระดับ:** สูง (ต้องผ่าน Part 016 เรื่อง Duck Typing/SOLID/DIP มาก่อนอย่างแม่นยำ และควร
> ผ่าน Part 082 เรื่อง Service Object/Form Object กับ Part 083 เรื่อง Query
> Object/Decorator มาแล้ว เพราะ Part นี้คือการนำแนวคิดทั้งหมดนั้นมาขยายเป็นระดับ
> สถาปัตยกรรมทั้งแอป)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงด้วย Ruby 3.3.6 +
> Rails 8.1.4 ผ่าน RSpec)

## สารบัญของ Part นี้

- Step 831: Clean Architecture / Hexagonal Architecture คืออะไร — แนวคิด Ports and Adapters
- Step 832: ความตึงเครียดที่ตรงไปตรงมากับ "Rails Way" — ทำไม Rails ไม่ได้ถูกออกแบบมาเพื่อสิ่งนี้
- Step 833: ทางสายกลางที่ใช้งานได้จริง — ActiveRecord เป็น persistence, PORO เป็น business logic
- Step 834: ตัวอย่างจริง — `OrderTotal` domain object ที่ไม่รู้จัก Rails เลยแม้แต่น้อย
- Step 835: เชื่อม PORO เข้ากับ Rails ผ่าน Service Object — `CheckoutService`
- Step 836: ทบทวน Dependency Injection ผ่าน constructor (จาก Part 016) และขยายไปสู่การ inject "port" ทั้งก้อน
- Step 837: `PaymentGateway` — duck-typed port, `StripeGateway` adapter, `FakePaymentGateway` test double
- Step 838: ทำไม DI ใน Ruby ไม่ต้องมี framework/DI container หนักแบบ Java/C#
- Step 839: Lightweight DI container แบบทำเอง เทียบกับ `dry-container`/`dry-auto_inject`
- Step 840: ประโยชน์ด้าน testing, คำตัดสินใจ (verdict) ที่ตรงไปตรงมา, และแบบฝึกหัดรวบยอด

---

## Step 831: Clean Architecture / Hexagonal Architecture คืออะไร — แนวคิด Ports and Adapters

### จุดเริ่มต้น: ปัญหาที่ทั้งสองแนวคิดพยายามแก้

**Clean Architecture** (เสนอโดย Robert C. Martin หรือ "Uncle Bob" คนเดียวกับที่รวบรวม
SOLID ใน Part 016) และ **Hexagonal Architecture** (หรือ **Ports and Adapters** เสนอโดย
Alistair Cockburn ก่อนหน้า Clean Architecture หลายปี) เป็นแนวคิดที่**แทบจะเหมือนกัน
ทุกประการในทางปฏิบัติ** ต่างกันแค่คำศัพท์และวิธีวาดภาพ ทั้งคู่พยายามแก้ปัญหาเดียวกัน:

> **"ตรรกะทางธุรกิจ (business logic) ของแอปมักถูกทำให้ผูกติดแน่นกับ framework,
> ฐานข้อมูล, หรือบริการภายนอก จนแยกทดสอบ แยกเปลี่ยน หรือแยกทำความเข้าใจไม่ได้เลยถ้าไม่มี
> สิ่งเหล่านั้นอยู่ด้วย"**

ลองนึกภาพแอป e-commerce ทั่วไปที่คำนวณส่วนลด/ภาษี/ยอดรวมของคำสั่งซื้ออยู่ **ข้างใน**
ActiveRecord callback ของ model `Order` โดยตรง — ถ้าอยากทดสอบสูตรคำนวณนี้ ต้อง boot
Rails environment ทั้งหมด, ต้องมีฐานข้อมูลจริง, ต้อง `Order.create!` จริง แค่จะทดสอบ
"ตรรกะทางคณิตศาสตร์" ง่ายๆ อันหนึ่ง ทั้งที่ตรรกะนั้นไม่ได้เกี่ยวอะไรกับฐานข้อมูลเลยจริงๆ

### แนวคิดหลัก: business logic อยู่ตรงกลาง, framework อยู่ที่ขอบ

ภาพที่ Alistair Cockburn วาดไว้ (ทำให้ได้ชื่อว่า "Hexagonal") คือรูปหกเหลี่ยม (จริงๆ จะเป็น
กี่เหลี่ยมก็ได้ ไม่ใช่สาระสำคัญ) ที่มี **core ตรงกลางเป็นตรรกะทางธุรกิจ (domain / business
rules)** ล้อมรอบด้วย **"port"** (จุดเชื่อมต่อที่เป็นนามธรรม — แค่ interface/protocol)
และรอบนอกสุดคือ **"adapter"** (รายละเอียดจริงที่คุยกับโลกภายนอก เช่น ฐานข้อมูล, web
framework, third-party API):

```
                    ┌─────────────────────────────────────┐
                    │              ADAPTERS                 │
                    │   (Rails Controller, ActiveRecord,    │
                    │    Stripe API, Redis, S3, Mailer)      │
                    │                                        │
                    │   ┌───────────────────────────────┐   │
                    │   │            PORTS                │   │
                    │   │  (interface/protocol ที่นิยาม    │   │
                    │   │   ว่า "ข้างนอกคุยกับข้างในยังไง") │   │
                    │   │                                  │   │
                    │   │   ┌─────────────────────────┐   │   │
                    │   │   │      DOMAIN / CORE        │   │   │
                    │   │   │  (business rules ล้วนๆ    │   │   │
                    │   │   │   ไม่รู้จัก Rails เลย)      │   │   │
                    │   │   └─────────────────────────┘   │   │
                    │   │                                  │   │
                    │   └───────────────────────────────┘   │
                    │                                        │
                    └─────────────────────────────────────┘
```

**กฎเหล็กข้อเดียวที่ต้องจำ (Dependency Rule):**

> **Dependency ต้องชี้เข้าหาศูนย์กลางเสมอ (inward) ไม่ใช่ชี้ออก (outward)**
>
> Domain/Core **ห้ามรู้จัก** Rails, ActiveRecord, Stripe SDK, HTTP request ใดๆ ทั้งสิ้น
> ส่วน Adapter (Controller, ActiveRecord model, Stripe client) **รู้จัก** Domain ได้
> เพราะมันเป็นฝ่าย "เรียกใช้" Domain ไม่ใช่ Domain เป็นฝ่ายเรียกมันกลับ

สังเกตว่านี่คือ**หลักการเดียวกับ Dependency Inversion Principle (DIP)** ที่เรียนไปแล้วใน
Part 016 Step 156 เป๊ะๆ เพียงแต่ตอนนั้นเราใช้ DIP กับ class คู่เดียว (`OrderProcessor` กับ
`EmailSender`/`SmsSender`) — Clean/Hexagonal Architecture คือการ**เอา DIP ไปใช้กับทั้ง
แอปพลิเคชัน** ให้ทุกจุดที่ business logic ต้องคุยกับโลกภายนอก (ฐานข้อมูล, web framework,
third-party service) ทำผ่าน "port" (สัญญาที่เป็นนามธรรม) แทนที่จะเรียก class ที่เจาะจง
ของรายละเอียดนั้นตรงๆ

### "Ports and Adapters" แปลว่าอะไรกันแน่

- **Port** = สัญญา/protocol ที่บอกว่า "domain ต้องการความสามารถอะไรจากโลกภายนอก" — ใน
  Ruby ไม่ต้องมี `interface` keyword (เหมือนที่เรียนใน ISP, Part 016 Step 155) port คือ
  **ชุดของ method ที่ตกลงกันไว้** เช่น "อะไรก็ตามที่ต้องการเป็น payment gateway ต้องมี
  method `#charge(amount_cents:, currency:, source:)` ที่คืนค่าที่มี `#success?`"
- **Adapter** = implementation จริงที่ทำให้ port นั้นใช้งานได้กับระบบภายนอกจริงๆ เช่น
  `StripeGateway` (คุยกับ Stripe จริง), `FakePaymentGateway` (จำลองไว้ทดสอบ) — ทั้งสอง
  implement port เดียวกัน จึง**สลับกันได้อย่างอิสระ** โดย domain ไม่รู้ตัวเลยว่าใช้ตัวไหน
  อยู่ (Part นี้จะเขียนตัวอย่างนี้จริงใน Step 834–837)

> **ทำไมถึงเรียก "Clean"?** Uncle Bob ตั้งชื่อโดยเน้นว่าสถาปัตยกรรมแบบนี้ทำให้ core ของ
> แอป "สะอาด" จากรายละเอียดทาง technical — เขาวาดเป็นวงกลมซ้อนกัน (concentric circles)
> แทนหกเหลี่ยม แต่กฎ Dependency Rule เดียวกันทุกประการกับ Hexagonal Architecture ของ
> Cockburn ในทางปฏิบัติ วิศวกร Rails ส่วนใหญ่ใช้คำสองคำนี้แทนกันได้เลย

---

## Step 832: ความตึงเครียดที่ตรงไปตรงมากับ "Rails Way" — ทำไม Rails ไม่ได้ถูกออกแบบมาเพื่อสิ่งนี้

### Rails Way สวนทางกับ Dependency Rule โดยธรรมชาติ

Part 001 Step 1 บอกไว้ว่าจุดเด่นของ Rails คือ **Convention over Configuration** และ
**MVC Architecture** — แต่ในทางปฏิบัติ ActiveRecord (M ใน MVC) ถูกออกแบบมาให้เป็น
**สอง"หน้าที่"รวมอยู่ใน class เดียว** อย่างจงใจ:

1. **Domain object** — เก็บ attribute และมี business logic (validation, calculation,
   state transition) แบบ `order.total`, `order.paid?`, `order.cancel!`
2. **Persistence layer** — รู้วิธีอ่าน/เขียนแถวในตารางฐานข้อมูล ผ่าน `ActiveRecord::Base`
   (`find`, `save`, `where`, `update!` ฯลฯ)

```ruby
class Order < ApplicationRecord   # <- สืบทอดจาก ActiveRecord::Base โดยตรง
  belongs_to :customer            # <- รู้จักโครงสร้างฐานข้อมูล (foreign key)

  def total                       # <- แต่ก็มี business logic ปนอยู่ในตัวเดียวกัน
    line_items.sum { |item| item.price * item.quantity }
  end
end
```

`Order` ในตัวอย่างนี้**ไม่ใช่ domain object บริสุทธิ์**ตามความหมายของ Clean Architecture
เลย — มัน `< ApplicationRecord` ซึ่งแปลว่ามัน**สืบทอดความสามารถทั้งหมดของ ActiveRecord**
(SQL query, connection pool, schema introspection) เข้ามาปนอยู่ในตัวมันด้วยโดยอัตโนมัติ
พูดตรงๆ คือ: **ทุก ActiveRecord model คือการละเมิด Dependency Rule ของ Clean Architecture
อยู่แล้วโดยธรรมชาติ** เพราะมัน**ชี้ออกไปหา framework** (ActiveRecord/ฐานข้อมูล) ไม่ใช่ชี้
เข้าหา domain อย่างเดียว

### ถ้าจะทำ Clean Architecture "แบบเป๊ะ" ใน Rails ต้องแลกกับอะไร

ถ้าอยากทำตามกฎ Dependency Rule อย่างเคร่งครัดจริงๆ ต้องแยก **Domain Entity** (PORO ล้วนๆ
ไม่มี ActiveRecord) ออกจาก **Persistence Model** (ActiveRecord model ที่ทำหน้าที่แค่
อ่าน/เขียนฐานข้อมูล) อย่างสมบูรณ์ แล้วเขียน **Repository** เป็นชั้นกลางที่แปลงไปมาระหว่าง
สองฝั่งนี้:

```ruby
# Domain Entity — PORO ล้วนๆ ไม่รู้จัก ActiveRecord เลย
class OrderEntity
  attr_reader :id, :customer_email, :status, :line_items

  def initialize(id:, customer_email:, status:, line_items:)
    @id = id
    @customer_email = customer_email
    @status = status
    @line_items = line_items
  end
end

# Repository — ชั้นกลางที่แปลง ActiveRecord record <-> Domain Entity
class OrderRepository
  def find(id)
    record = Order.find(id)   # ActiveRecord model แยกต่างหาก ไม่ใช่ตัวเดียวกับ OrderEntity
    OrderEntity.new(
      id: record.id,
      customer_email: record.customer_email,
      status: record.status,
      line_items: record.line_items.to_a
    )
  end

  def save(entity)
    record = Order.find_or_initialize_by(id: entity.id)
    record.update!(customer_email: entity.customer_email, status: entity.status)
  end
end
```

**นี่คือ Clean Architecture แบบ "เป๊ะตามตำรา"** — แต่สังเกตสิ่งที่เกิดขึ้น: ตอนนี้มี
**สอง class ที่แทน "คำสั่งซื้อ" ในระบบเดียวกัน** (`Order` กับ `OrderEntity`) ทุกครั้งที่
เพิ่ม field ใหม่ในฐานข้อมูล ต้องแก้ทั้ง migration, `Order`, `OrderEntity`, **และ**
`OrderRepository` ที่แปลงไปมา — โค้ด boilerplate เพิ่มขึ้นหลายเท่าตัวสำหรับ CRUD ธรรมดาๆ
ที่ Rails scaffold ให้ฟรีอยู่แล้วใน 1 คำสั่ง (Part 028)

### ต้นทุนที่แท้จริงเทียบกับประโยชน์ที่ได้

| | ประโยชน์ | ต้นทุน |
|---|---|---|
| **Full Clean Architecture** | ตรรกะธุรกิจทดสอบได้โดยไม่ใช้ฐานข้อมูลเลย 100%, สลับฐานข้อมูล/framework ได้ทั้งระบบในทางทฤษฎี | Boilerplate เพิ่มมหาศาล (Entity + Repository + Mapper สำหรับทุก model), ทีมใหม่งงว่า "ทำไมมี Order สองตัว", ขัดกับทุก generator/gem ของ Rails ที่คาดหวัง ActiveRecord model ตรงๆ (Devise, Pundit, FactoryBot, Kaminari ฯลฯ ต้องเขียน adapter ห่อเพิ่มเอง) |
| **Rails Way ปกติ (ActiveRecord = domain + persistence)** | เขียนเร็ว, generator/gem ทั้งระบบนิเวศใช้ได้ทันที, ทีมใหม่เข้าใจง่ายเพราะเป็น convention มาตรฐาน | Business logic ปนกับ persistence, ทดสอบ logic ซับซ้อนต้องพึ่งฐานข้อมูลเสมอ, model ใหญ่ขึ้นเรื่อยๆ ("fat model") ถ้าไม่มีวินัย |

> **ข้อสรุปตรงไปตรงมาที่ควรจำไว้ตลอด Part นี้:** สำหรับแอป Rails ทั่วไปที่เป็น CRUD-heavy
> business application (ซึ่งคือแอปส่วนใหญ่ที่ทีมพัฒนาจริงเขียนกัน) **การสู้กับ Rails Way
> เต็มรูปแบบด้วย Full Clean Architecture มักไม่คุ้มค่า** — ต้นทุนความซับซ้อนที่เพิ่มขึ้นสูง
> กว่าประโยชน์ที่ได้มาก โดยเฉพาะกับทีมขนาดเล็ก-กลางที่ต้อง ship ฟีเจอร์เร็ว บริษัทที่ประสบ
> ความสำเร็จกับ Rails อย่าง Shopify, GitHub, Basecamp **ไม่ได้** ใช้ Full Clean
> Architecture ทั่วทั้งแอป — พวกเขาใช้ **Rails Way ผสมกับวินัยในการแยกส่วนที่ซับซ้อนจริงๆ
> ออกมาเป็น PORO** ซึ่งคือสิ่งที่ Step 833 จะพูดถึง

---

## Step 833: ทางสายกลางที่ใช้งานได้จริง — ActiveRecord เป็น persistence, PORO เป็น business logic

### แนวคิด: ไม่ต้องเลือกสุดทาง เลือก "ที่คุ้มที่สุด" แทน

ทางสายกลางที่ทีม Rails มืออาชีพจำนวนมากใช้จริง (และเราจะใช้ตลอด Part นี้) คือ:

> **ปล่อยให้ ActiveRecord model ทำหน้าที่ persistence (query, association, validation
> ระดับข้อมูล) ต่อไปตามปกติแบบ Rails Way — แต่เมื่อไหร่ที่ business logic เริ่มซับซ้อน
> เกินกว่า one-liner ธรรมดา หรือเป็น logic ที่อยากทดสอบแยกจากฐานข้อมูล/อยากนำไปใช้ซ้ำใน
> บริบทอื่น ให้ดึงมันออกมาเป็น PORO (service object, domain object) ที่ไม่รู้จัก Rails
> เลย**

นี่**ไม่ใช่** Clean Architecture แบบเป๊ะ (ไม่มี Repository, ไม่มี Entity คู่ขนานกับ
ActiveRecord model) แต่มัน**นำหลักการที่สำคัญที่สุด**ของ Clean Architecture มาใช้แบบเลือก
เฉพาะจุดที่คุ้มค่า:

1. **แยกตรรกะทางธุรกิจที่ซับซ้อนออกจากรายละเอียด framework** (แม้จะไม่แยกทุกจุดก็ตาม)
2. **inject dependency แทนการ hardcode** เมื่อ logic นั้นต้องคุยกับบริการภายนอก
3. **PORO ทดสอบได้เร็วโดยไม่ต้องพึ่งฐานข้อมูล** สำหรับส่วนที่ไม่จำเป็นต้องพึ่งจริงๆ

### เกณฑ์ตัดสินใจ: เมื่อไหร่ควรดึง logic ออกมาเป็น PORO

| สัญญาณ | ตัวอย่าง | ควรดึงออกไหม |
|---|---|---|
| Logic เป็นสูตรคำนวณล้วนๆ ไม่แตะฐานข้อมูล | คำนวณยอดรวม, ส่วนลด, ภาษี, ค่าคอมมิชชัน | ควรดึง — ได้ unit test ที่เร็วมากทันที |
| Logic ต้องคุยกับบริการภายนอก (payment, email, SMS) | เรียก Stripe API, ส่ง SMS ยืนยัน OTP | ควรดึง — inject เป็น dependency เพื่อสลับ/mock ได้ |
| Logic เกี่ยวข้องกับหลาย model พร้อมกัน (orchestration) | "การ checkout" ที่ต้องแตะ Order, Inventory, Payment, Notification | ควรดึงเป็น Service Object (Part 082) ที่เรียก PORO ข้างในอีกที |
| Logic เป็นแค่ query/filter ข้อมูล | `Order.where(status: "pending")` | ไม่ต้องดึง — ใช้ scope ปกติ หรือ Query Object (Part 083) ถ้าซับซ้อนมาก |
| Logic เกี่ยวกับ format การแสดงผล | จัดรูปแบบวันที่, สกุลเงิน | ไม่ต้องดึงเป็น PORO ธุรกิจ — ใช้ Decorator/Presenter (Part 083) แทน |

### ภาพรวมสถาปัตยกรรมที่จะสร้างใน Part นี้

```
Controller (adapter — รู้จัก Rails/HTTP)
     │
     ▼
CheckoutService (service object — "รู้จัก" ทั้ง Rails/ActiveRecord และ domain)
     │                                    │
     ▼                                    ▼
OrderTotal (PORO — ไม่รู้จัก Rails เลย)   PaymentGateway (port — duck-typed interface)
                                            │              │
                                     StripeGateway   FakePaymentGateway
                                     (adapter จริง)   (adapter ปลอมไว้เทส)
```

สังเกตว่า `CheckoutService` คือจุดที่ "เชื่อม" สองโลกเข้าด้วยกัน — มันรู้จัก ActiveRecord
(`Order`, `order.line_items`) แต่**มอบหมาย**การคำนวณให้ `OrderTotal` (ไม่รู้จัก Rails)
และ**มอบหมาย**การชำระเงินให้ `PaymentGateway` (port ที่ inject เข้ามา) — นี่คือ "Ports and
Adapters ในเวอร์ชันที่คุ้มค่าที่สุดสำหรับ Rails" ที่ Step 834 เป็นต้นไปจะสร้างจริงทีละชิ้น

---

## Step 834: ตัวอย่างจริง — `OrderTotal` domain object ที่ไม่รู้จัก Rails เลยแม้แต่น้อย

### เตรียมโปรเจกต์ทดลอง

สร้างแอป Rails ใหม่เพื่อทดลองแนวคิดทั้งหมดใน Part นี้ (ตัวอย่างทั้งหมดด้านล่างรันจริงและ
ผ่านการทดสอบด้วย RSpec แล้ว):

```bash
rails new clean_arch_demo --minimal -d sqlite3
cd clean_arch_demo
bundle add rspec-rails --group "development,test"
bundle add stripe
bin/rails g rspec:install

bin/rails generate model Order customer_email:string status:string \
  discount_percentage:integer total_cents:integer payment_transaction_id:string
bin/rails generate model LineItem order:references product_name:string \
  unit_price_cents:integer quantity:integer
bin/rails db:migrate
```

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  has_many :line_items, dependent: :destroy
end
```

```ruby
# app/models/line_item.rb
class LineItem < ApplicationRecord
  belongs_to :order
end
```

> **หมายเหตุเรื่องหน่วยเงิน:** ตัวอย่างทั้งหมดใน Part นี้เก็บเงินเป็น **สตางค์ (cents)**
> แบบ `Integer` ไม่ใช่ `Float`/`Decimal` บาท — เป็นแนวปฏิบัติมาตรฐานของวงการที่ต้องจัดการ
> เงินจริง (เหมือนที่ Stripe เองก็ใช้หน่วยสตางค์/cents ในทุก API) เพื่อหลีกเลี่ยงปัญหาความ
> คลาดเคลื่อนของ floating point ที่เกิดกับทศนิยมเงินโดยตรง

### เขียน `OrderTotal` — domain object ล้วนๆ

```ruby
# frozen_string_literal: true
# app/domain/order_total.rb

# OrderTotal คือ "domain object" ล้วนๆ ไม่มีการ require Rails/ActiveRecord เลย
# แม้แต่บรรทัดเดียว รับ line_items เป็น object ใดๆ ก็ได้ (duck typing) ที่ตอบสนอง
# #unit_price_cents และ #quantity เท่านั้น ทำให้เทสได้ด้วย Plain Old Ruby Object
# โดยไม่ต้องพึ่งฐานข้อมูลหรือบูต Rails environment เลย
class OrderTotal
  Result = Struct.new(:subtotal_cents, :discount_cents, :tax_cents, :total_cents, keyword_init: true)

  def initialize(line_items:, discount_percentage: 0, tax_rate: 0.07)
    @line_items = line_items
    @discount_percentage = discount_percentage
    @tax_rate = tax_rate
  end

  def calculate
    subtotal = subtotal_cents
    discount = (subtotal * @discount_percentage / 100.0).round
    taxable_amount = subtotal - discount
    tax = (taxable_amount * @tax_rate).round

    Result.new(
      subtotal_cents: subtotal,
      discount_cents: discount,
      tax_cents: tax,
      total_cents: taxable_amount + tax
    )
  end

  private

  def subtotal_cents
    @line_items.sum { |item| item.unit_price_cents * item.quantity }
  end
end
```

**สังเกตสิ่งสำคัญที่สุดในไฟล์นี้:**

- **ไม่มี `require "rails"` หรือ `require "active_record"` เลย** — ไฟล์นี้จะรันได้แม้ใน
  โปรแกรม Ruby ธรรมดาที่ไม่มี Rails ติดตั้งอยู่เลยด้วยซ้ำ
- `@line_items` ไม่ได้ประกาศว่าต้องเป็น `Array` หรือ `ActiveRecord::Relation` — มันแค่
  เรียก `.sum { |item| item.unit_price_cents * item.quantity }` ซึ่งใช้ได้กับ **object
  ใดๆ ที่ include `Enumerable`** (ทั้ง `Array` ธรรมดา และ `ActiveRecord::Relation` ที่
  include `Enumerable` มาให้เหมือนกัน จากที่เรียนใน Part 016 Step 151) และแต่ละสมาชิกแค่
  ต้องตอบสนอง `#unit_price_cents`/`#quantity` — นี่คือ duck typing ที่ทำให้ `OrderTotal`
  ใช้ได้ทั้งกับ `LineItem` (ActiveRecord model จริง) **และ** Struct/Hash ปลอมในเทสได้เลย
  โดยไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียว
- ไฟล์วางไว้ที่ `app/domain/` (ไม่ใช่ `app/models/`) เพื่อสื่อสารเจตนาให้ชัดว่า "นี่คือ
  business rule ไม่ใช่ persistence layer" — Rails 8 (ผ่าน Zeitwerk) **autoload ทุก
  โฟลเดอร์ย่อยใต้ `app/` โดยอัตโนมัติอยู่แล้ว** จึงสร้างโฟลเดอร์ใหม่ใต้ `app/` ได้เลยโดย
  ไม่ต้องแก้ `config/application.rb` เพิ่ม autoload path เอง

### ทดสอบ `OrderTotal` ด้วย RSpec โดยไม่ require rails_helper เลย

```ruby
# frozen_string_literal: true
# spec/domain/order_total_spec.rb

# หมายเหตุ: spec นี้ require แค่ order_total.rb ไฟล์เดียว ไม่ require "rails_helper"
# เลย เพื่อพิสูจน์ว่า OrderTotal เป็น POROแท้ๆ ที่รันได้โดยไม่ต้องบูต Rails
require_relative "../../app/domain/order_total"

LineItemDouble = Struct.new(:unit_price_cents, :quantity)

RSpec.describe OrderTotal do
  it "คำนวณ subtotal จากราคาต่อหน่วยคูณจำนวน" do
    line_items = [
      LineItemDouble.new(10_000, 2),
      LineItemDouble.new(5_000, 1)
    ]

    result = described_class.new(line_items: line_items).calculate

    expect(result.subtotal_cents).to eq(25_000)
  end

  it "คิดส่วนลดเป็นเปอร์เซ็นต์ก่อนคำนวณภาษี" do
    line_items = [LineItemDouble.new(100_00, 1)]

    result = described_class.new(
      line_items: line_items,
      discount_percentage: 10,
      tax_rate: 0.07
    ).calculate

    expect(result.discount_cents).to eq(1_000)
    expect(result.tax_cents).to eq(630)
    expect(result.total_cents).to eq(9_630)
  end

  it "ไม่มีส่วนลดและไม่มีภาษีเมื่อไม่ระบุ" do
    result = described_class.new(
      line_items: [LineItemDouble.new(1_000, 3)],
      tax_rate: 0
    ).calculate

    expect(result.total_cents).to eq(3_000)
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/domain/order_total_spec.rb
```

```
...

Finished in 0.00366 seconds (files took 0.10992 seconds to load)
3 examples, 0 failures
```

**ความเร็วคือประเด็นสำคัญที่ไม่ควรมองข้าม:** เทสทั้ง 3 ตัวนี้รันเสร็จใน **0.00366 วินาที**
เพราะไม่มีการเชื่อมต่อฐานข้อมูล ไม่มีการโหลด Rails application ใดๆ เลย ถ้าเราเขียนสูตร
คำนวณนี้ไว้ในเมธอดของ `Order` (ActiveRecord model) แทน ทุกเทสจะต้อง `Order.create!` จริง
ในฐานข้อมูลทดสอบ ซึ่งช้ากว่าหลายเท่าตัวเมื่อจำนวนเทสเพิ่มขึ้นเป็นหลักพัน — นี่คือประโยชน์
ที่จับต้องได้จริง ไม่ใช่แค่ความสวยงามทางทฤษฎี

### พิสูจน์ duck typing: `OrderTotal` ใช้กับ ActiveRecord จริงได้ทันทีโดยไม่ต้องแก้โค้ด

```bash
bin/rails runner '
order = Order.create!(customer_email: "test@example.com", status: "pending", discount_percentage: 0)
order.line_items.create!(product_name: "A", unit_price_cents: 1000, quantity: 3)
result = OrderTotal.new(line_items: order.line_items, tax_rate: 0.07).calculate
puts result.inspect
'
```

```
#<struct OrderTotal::Result subtotal_cents=3000, discount_cents=0, tax_cents=210, total_cents=3210>
```

`order.line_items` ในที่นี้คือ `ActiveRecord::Associations::CollectionProxy` (ไม่ใช่
`Array` หรือ `Struct` เหมือนในเทส) แต่ `OrderTotal` **ทำงานได้ทันทีโดยไม่ต้องแก้โค้ดแม้แต่
บรรทัดเดียว** เพราะมันไม่เคยสนใจ "ชนิด" ของ `@line_items` เลย สนใจแค่ว่ามันตอบสนอง
`#sum` และแต่ละสมาชิกตอบสนอง `#unit_price_cents`/`#quantity` ได้หรือไม่ — นี่คือพลังของ
duck typing (Part 016 Step 151) ที่ทำให้ domain object เชื่อมกับทั้งโลกทดสอบและโลกจริง
ได้อย่างไร้รอยต่อ

---

## Step 835: เชื่อม PORO เข้ากับ Rails ผ่าน Service Object — `CheckoutService`

### ทำไมต้องมี Service Object มาคั่นกลาง

`OrderTotal` คำนวณยอดเงินได้ แต่ **ไม่รู้วิธีบันทึกผลลัพธ์ลงฐานข้อมูล** และ **ไม่รู้วิธี
เรียกเก็บเงินจริงจากลูกค้า** — นี่คือเจตนา ไม่ใช่ข้อบกพร่อง: `OrderTotal` ต้องคง "ไม่รู้
จัก Rails" ต่อไปเพื่อรักษาความง่ายในการทดสอบ ดังนั้นจึงต้องมี class อีกชั้นหนึ่งที่ทำหน้าที่
**ประสานงาน (orchestrate)** ระหว่าง domain object (`OrderTotal`) กับ ActiveRecord
(`Order`) กับบริการภายนอก (payment gateway) — นี่คือหน้าที่ของ **Service Object** (ตาม
รูปแบบที่ Part 082 แนะนำ)

```ruby
# frozen_string_literal: true
# app/services/checkout_service.rb

# CheckoutService คือ Service Object ที่ "รู้จัก Rails/ActiveRecord" เต็มตัว (มันโหลดและ
# อัปเดต Order ผ่าน ActiveRecord ตรงๆ) แต่มันไม่คำนวณยอดเงินเอง — มันมอบหมายให้
# OrderTotal (POROเล็กๆ ที่ไม่รู้จัก Rails เลย) เป็นคนคำนวณแทน และรับ payment_gateway
# เข้ามาทาง constructor แทนการ new StripeGateway ขึ้นมาเองข้างใน (Dependency Inversion)
class CheckoutService
  def initialize(payment_gateway:)
    @payment_gateway = payment_gateway
  end

  def call(order:, card_token:)
    totals = OrderTotal.new(
      line_items: order.line_items,
      discount_percentage: order.discount_percentage
    ).calculate

    result = @payment_gateway.charge(
      amount_cents: totals.total_cents,
      currency: "thb",
      source: card_token
    )

    if result.success?
      order.update!(
        status: "paid",
        total_cents: totals.total_cents,
        payment_transaction_id: result.transaction_id
      )
    else
      order.update!(status: "payment_failed")
    end

    result
  end
end
```

**อธิบายการแบ่งความรับผิดชอบ (สังเกตว่านี่คือ SRP จาก Part 016 Step 152 ในระดับ
สถาปัตยกรรม):**

| Class | รู้จัก Rails/ActiveRecord ไหม | หน้าที่ |
|---|---|---|
| `OrderTotal` | ไม่รู้จักเลย | คำนวณตัวเลขล้วนๆ (subtotal/discount/tax/total) |
| `PaymentGateway` (port) | ไม่รู้จักเลย | นิยาม "สัญญา" ว่าการชำระเงินต้องมีหน้าตาแบบไหน |
| `CheckoutService` | **รู้จักเต็มตัว** | ประสานงาน: ดึงข้อมูลจาก AR → ส่งให้ domain คำนวณ → เรียก payment → บันทึกผลกลับ AR |
| `Order`/`LineItem` (ActiveRecord) | เป็น Rails/AR โดยตรง | persistence: เก็บ/อ่านข้อมูลจากฐานข้อมูล |

`CheckoutService` คือจุดที่ **"เจตนายอมให้ผูกติดกับ Rails"** อย่างมีสติ — มันเป็น
**adapter ที่เชื่อมโลก AR เข้ากับโลก domain** ไม่ใช่ domain เอง จุดสำคัญคือ **ตรรกะการ
คำนวณเงิน (ซึ่งซับซ้อนและมีค่าต่อธุรกิจที่สุด) ถูกดึงออกไปอยู่ใน `OrderTotal` ที่ทดสอบ
แยกได้เร็วมาก** ส่วน `CheckoutService` เองมีแค่ "การประสานงาน" ที่ไม่ซับซ้อน จึงทดสอบด้วย
ฐานข้อมูลจริง (ที่ช้ากว่า) แค่ไม่กี่เคสก็เพียงพอ — ทางสายกลางที่ Step 833 พูดถึงคือแบบนี้

> **ยังไม่มี `PaymentGateway` และ `payment_gateway.charge`/`.success?` จริงในตอนนี้** —
> Step 836–837 จะเขียนส่วนนี้ให้ครบ ตอนนี้ให้สังเกตแค่ว่า `CheckoutService` **รับ
> `payment_gateway:` เข้ามาทาง constructor** แทนการ `new StripeGateway.new` ขึ้นมาเอง
> ข้างในตัวมันเอง — นี่คือ Dependency Injection ที่ Step 836 จะทบทวนและขยายความต่อ

---

## Step 836: ทบทวน Dependency Injection ผ่าน constructor และขยายไปสู่การ inject "port" ทั้งก้อน

### ทบทวนสั้นๆ จาก Part 016 Step 156

Part 016 สอน Dependency Inversion Principle ผ่านตัวอย่าง `OrderProcessor` ที่ inject
`notifier:` เข้ามาทาง constructor แทนการ `new EmailSender.new` ขึ้นมาเองข้างใน:

```ruby
# ทบทวนจาก Part 016 Step 156
class OrderProcessor
  def initialize(notifier: EmailSender.new)
    @notifier = notifier
  end

  def complete_order(order)
    @notifier.send_message(order[:customer_contact], "คำสั่งซื้อของคุณเสร็จสมบูรณ์แล้ว")
  end
end
```

หลักการคือ: **`OrderProcessor` รู้แค่ว่า `@notifier` ต้องตอบสนอง `send_message` ได้ ไม่
สนใจว่ามันเป็น class อะไร** — นี่คือ **constructor injection** รูปแบบพื้นฐานที่สุด: ส่ง
dependency (object เดียว, method เดียว) เข้าไปตอนสร้าง object

### ระดับที่ลึกขึ้น: inject "port" ทั้งก้อน ไม่ใช่แค่ method เดียว

สิ่งที่ `CheckoutService` ทำใน Step 835 คือแนวคิดเดียวกัน แต่ **ยกระดับขึ้นหนึ่งขั้น**:
แทนที่จะ inject object ที่มี method เดียว (`send_message`) เรา inject **object ที่แทน
"บริการทั้งบริการ"** (payment gateway) ซึ่งอาจมีได้หลาย method ในอนาคต (`charge`,
`refund`, `verify_webhook` ฯลฯ) — นี่คือความหมายที่แท้จริงของคำว่า **"port"** ใน
Hexagonal Architecture: มันไม่ใช่แค่ dependency เดี่ยวๆ แต่คือ **สัญญาของบริการทั้งบริการ**
ที่ domain/service ต้องการจากโลกภายนอก

```ruby
class CheckoutService
  # payment_gateway: คือ "port" — ไม่ใช่แค่ dependency เดียว แต่คือสัญญาของบริการ
  # ชำระเงินทั้งบริการ ที่ CheckoutService ต้องการจากโลกภายนอก
  def initialize(payment_gateway:)
    @payment_gateway = payment_gateway
  end
  # ...
end
```

**ความแตกต่างจากตัวอย่าง Part 016 มีอยู่ 3 จุด:**

1. **ไม่มี default value** (`payment_gateway:` ไม่มี `= StripeGateway.new`) — เพราะการ
   สร้าง `StripeGateway` จริงต้องมี API key ที่มาจาก credentials (Part 074) ซึ่งไม่ควร
   ผูกอยู่ในโค้ดของ `CheckoutService` เอง จุดที่ตัดสินใจว่า "จะใช้ adapter ตัวไหน" ควรอยู่
   ที่**ขอบของระบบ** (เช่น controller หรือจุด bootstrap) ไม่ใช่ที่ service เอง — นี่คือ
   หลักการ **"compose at the edge, not in the middle"** ที่ Clean Architecture เน้นย้ำ
2. **สัญญามีหลาย method ที่เป็นไปได้** ไม่ใช่แค่ method เดียว — เป็นเหตุผลที่ Step 837 จะ
   นิยาม "รูปร่าง" ของ port นี้อย่างชัดเจนขึ้นผ่านเอกสารและตัวอย่าง adapter สองตัว
3. **ค่าที่คืนกลับมาก็เป็นส่วนหนึ่งของสัญญาด้วย** — `payment_gateway.charge(...)` ต้องคืน
   object ที่ตอบสนอง `#success?` ได้เสมอ ไม่ว่า adapter ตัวไหนจะ implement ก็ตาม (เรียก
   ว่า **"contract"** ของ port — ไม่ใช่แค่ชื่อ method ที่ต้องตรงกัน แต่รวมถึงรูปร่างของ
   argument และ return value ด้วย)

### วิธี compose ที่ขอบของระบบ (ตัวอย่างใน controller)

```ruby
# app/controllers/checkout_controller.rb
class CheckoutController < ApplicationController
  def create
    order = Order.find(params[:order_id])

    # จุดนี้คือ "ขอบของระบบ" ที่ตัดสินใจว่าจะใช้ adapter ตัวไหน — เลือกตาม environment
    gateway = Rails.env.test? ? FakePaymentGateway.new : StripeGateway.new(api_key: Rails.application.credentials.dig(:stripe, :secret_key))

    result = CheckoutService.new(payment_gateway: gateway).call(
      order: order,
      card_token: params[:card_token]
    )

    if result.success?
      redirect_to order, notice: "ชำระเงินสำเร็จ"
    else
      redirect_to order, alert: "ชำระเงินไม่สำเร็จ: #{result.error_message}"
    end
  end
end
```

> **ข้อควรระวังในทางปฏิบัติ:** การเช็ค `Rails.env.test?` ตรงๆ ใน controller แบบนี้เป็น
> ตัวอย่างง่ายๆ เพื่อสาธิตแนวคิด ในโปรเจกต์จริงมักย้าย logic การเลือก adapter นี้ไปไว้ที่
> **initializer** (เช่น `config/initializers/payment_gateway.rb` ที่ตั้งค่า
> `Rails.application.config.payment_gateway` ตาม environment ครั้งเดียวตอน boot) แทนที่
> จะเช็คซ้ำทุกครั้งใน action — Step 839 จะแนะนำวิธีจัดการเรื่องนี้อย่างเป็นระบบมากขึ้นด้วย
> lightweight DI container

---

## Step 837: `PaymentGateway` — duck-typed port, `StripeGateway` adapter, `FakePaymentGateway` test double

### นิยาม "รูปร่าง" ของ port ด้วยค่าที่คืนกลับร่วมกัน

ก่อนเขียน adapter ทั้งสองตัว ต้องนิยามก่อนว่า `.charge(...)` **ต้องคืนอะไรกลับมาเหมือนกัน
ทุกประการ** ไม่ว่าจะเป็น adapter ตัวไหน — นี่คือ **"PaymentResult"** ที่ทำหน้าที่เป็นส่วน
หนึ่งของสัญญา (port) เดียวกัน:

```ruby
# frozen_string_literal: true
# app/domain/payment_result.rb

# PaymentResult คือค่าที่ adapter ทุกตัว (StripeGateway, FakePaymentGateway, ...) ต้องคืน
# กลับมาเหมือนกันทุกประการ นี่คือส่วนหนึ่งของ "สัญญา" (port) ที่ CheckoutService พึ่งพา
class PaymentResult < Struct.new(:success, :transaction_id, :error_message, keyword_init: true)
  def success?
    success
  end
end
```

### Adapter ตัวที่ 1: `StripeGateway` — adapter ฝั่ง production จริง

```ruby
# frozen_string_literal: true
# app/domain/stripe_gateway.rb

# StripeGateway คือ "adapter" ฝั่ง production ที่ implement port เดียวกับ
# FakePaymentGateway ทุกประการ (#charge รับ keyword arguments ชุดเดียวกัน คืน
# PaymentResult เหมือนกัน) ตัวมันเองเป็นจุดเดียวในระบบที่รู้จักรายละเอียดของ Stripe SDK
class StripeGateway
  def initialize(api_key:)
    @api_key = api_key
  end

  def charge(amount_cents:, currency:, source:)
    stripe_charge = Stripe::Charge.create(
      { amount: amount_cents, currency: currency, source: source },
      { api_key: @api_key }
    )

    PaymentResult.new(success: true, transaction_id: stripe_charge.id, error_message: nil)
  rescue Stripe::CardError => e
    PaymentResult.new(success: false, transaction_id: nil, error_message: e.message)
  end
end
```

**สังเกต:** `StripeGateway` คือ**จุดเดียวในระบบทั้งหมด**ที่รู้จักว่า `Stripe::Charge.create`
เขียนยังไง, argument รูปแบบไหน, exception ชื่ออะไร (`Stripe::CardError`) — ถ้าวันหนึ่ง
Stripe เปลี่ยน API หรือธุรกิจเปลี่ยนไปใช้ payment provider เจ้าอื่น (เช่น Omise, 2C2P ที่
นิยมในไทย) **จุดที่ต้องแก้มีที่เดียวคือไฟล์นี้** — `CheckoutService` ไม่ต้องถูกแตะเลย
แม้แต่บรรทัดเดียว (Open/Closed Principle จาก Part 016 Step 153 อีกครั้ง)

### Adapter ตัวที่ 2: `FakePaymentGateway` — test double สำหรับเทส

```ruby
# frozen_string_literal: true
# app/domain/fake_payment_gateway.rb

# FakePaymentGateway คือ "test double" ที่ implement port เดียวกับ StripeGateway ทุก
# ประการ แต่ไม่ยิง network request ออกไปเลย ใช้แทน StripeGateway ในเทสของ CheckoutService
# เพื่อให้เทสเร็ว, ไม่พึ่งอินเทอร์เน็ต, และควบคุมผลลัพธ์ได้ตามที่ต้องการทดสอบ
class FakePaymentGateway
  attr_reader :charges

  def initialize(succeed: true)
    @succeed = succeed
    @charges = []
  end

  def charge(amount_cents:, currency:, source:)
    @charges << { amount_cents: amount_cents, currency: currency, source: source }

    if @succeed
      PaymentResult.new(success: true, transaction_id: "fake_txn_#{@charges.size}", error_message: nil)
    else
      PaymentResult.new(success: false, transaction_id: nil, error_message: "การ์ดถูกปฏิเสธ (จำลอง)")
    end
  end
end
```

`FakePaymentGateway` มีความสามารถพิเศษที่ `StripeGateway` ไม่มี (และไม่ควรมี): มัน**จำ
ทุกการเรียก `charge`** ไว้ใน `@charges` เพื่อให้เทสตรวจสอบย้อนหลังได้ว่า `CheckoutService`
เรียกมันด้วยค่าอะไรบ้าง — เทคนิคนี้เรียกว่า **spy** (test double ชนิดหนึ่งที่บันทึกการเรียก
ไว้ ต่างจาก **stub** ที่แค่คืนค่าตายตัว) ซึ่งจะเรียนอย่างเป็นระบบใน Part 019 (RSpec)

### ทดสอบว่าทั้งสอง adapter "สลับกันได้จริง" ผ่าน `CheckoutService` เดียวกัน

```ruby
# frozen_string_literal: true
# spec/services/checkout_service_spec.rb

require "rails_helper"

RSpec.describe CheckoutService do
  let(:order) do
    order = Order.create!(customer_email: "manee@example.com", status: "pending", discount_percentage: 10)
    order.line_items.create!(product_name: "หนังสือ Ruby", unit_price_cents: 35_000, quantity: 2)
    order.line_items.create!(product_name: "ปากกา", unit_price_cents: 1_500, quantity: 5)
    order
  end

  context "เมื่อชำระเงินสำเร็จ" do
    it "อัปเดตสถานะ order เป็น paid พร้อมบันทึก transaction_id" do
      fake_gateway = FakePaymentGateway.new(succeed: true)
      service = described_class.new(payment_gateway: fake_gateway)

      result = service.call(order: order, card_token: "tok_test_123")

      expect(result).to be_success
      order.reload
      expect(order.status).to eq("paid")
      expect(order.payment_transaction_id).to eq("fake_txn_1")
      # subtotal = 350*2 + 15*5 = 700+75 = 775 บาท = 77500 สตางค์
      # discount 10% = 7750, taxable = 69750, tax 7% = 4883 (ปัดเศษ), total = 74633
      expect(order.total_cents).to eq(74_633)
    end

    it "ส่งจำนวนเงินที่คำนวณจาก OrderTotal ไปให้ payment gateway อย่างถูกต้อง" do
      fake_gateway = FakePaymentGateway.new(succeed: true)
      service = described_class.new(payment_gateway: fake_gateway)

      service.call(order: order, card_token: "tok_test_123")

      expect(fake_gateway.charges.first).to include(amount_cents: 74_633, currency: "thb", source: "tok_test_123")
    end
  end

  context "เมื่อชำระเงินไม่สำเร็จ" do
    it "อัปเดตสถานะ order เป็น payment_failed และไม่บันทึก transaction_id" do
      fake_gateway = FakePaymentGateway.new(succeed: false)
      service = described_class.new(payment_gateway: fake_gateway)

      result = service.call(order: order, card_token: "tok_declined")

      expect(result).not_to be_success
      order.reload
      expect(order.status).to eq("payment_failed")
      expect(order.payment_transaction_id).to be_nil
    end
  end

  context "เมื่อสลับไปใช้ StripeGateway จริง (stub เฉพาะชั้น Stripe SDK เท่านั้น)" do
    it "ยัง implement port เดียวกันและทำงานร่วมกับ CheckoutService ได้โดยไม่ต้องแก้โค้ด" do
      fake_stripe_charge = instance_double(Stripe::Charge, id: "ch_abc123")
      allow(Stripe::Charge).to receive(:create).and_return(fake_stripe_charge)

      stripe_gateway = StripeGateway.new(api_key: "sk_test_dummy")
      service = described_class.new(payment_gateway: stripe_gateway)

      result = service.call(order: order, card_token: "tok_test_123")

      expect(result).to be_success
      expect(result.transaction_id).to eq("ch_abc123")
      order.reload
      expect(order.status).to eq("paid")
    end
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/services/checkout_service_spec.rb
```

```
....

Finished in 0.09766 seconds (files took 1.24 seconds to load)
4 examples, 0 failures
```

**จุดที่สำคัญที่สุดในเทสชุดนี้คือ context สุดท้าย** — มันพิสูจน์ว่า `CheckoutService`
**เขียนโค้ดเดียวกันเป๊ะ** (`service.call(order:, card_token:)`) ใช้งานได้กับทั้ง
`FakePaymentGateway` (ปลอมทั้งหมด) และ `StripeGateway` (ของจริง ที่แค่ stub ชั้น Stripe
SDK ไว้ไม่ให้ยิง network จริง) — `CheckoutService` **ไม่เคยรู้เลย**ว่ากำลังคุยกับ adapter
ตัวไหนอยู่ นี่คือเป้าหมายที่แท้จริงของ "port" ใน Hexagonal Architecture: **swappability
โดยไม่ต้องแก้โค้ดฝั่งที่เรียกใช้แม้แต่บรรทัดเดียว**

> **ทำไม spec ของ `StripeGateway` เอง (ไม่ผ่าน `CheckoutService`) จึงไม่ได้เขียนแยกไว้ใน
> ตัวอย่างนี้:** เพราะการทดสอบว่า `Stripe::Charge.create` เรียก Stripe จริงถูกต้องหรือไม่
> เป็นหน้าที่ของ **integration test** ที่มักใช้เครื่องมืออย่าง VCR (Part 049) บันทึก/เล่น
> ซ้ำ HTTP request จริงกับ Stripe sandbox แยกต่างหากจากเทส unit ของ business logic — นี่
> คือเหตุผลที่ Clean Architecture มีค่า: เทส `OrderTotal` และ `CheckoutService` (ที่มีค่า
> ทางธุรกิจมากกว่า) รันเร็วและไม่ง้อ network เลย ส่วนเทสที่ต้องง้อ Stripe จริงๆ (ที่ควรมี
> น้อยกว่ามาก) ถูกจำกัดไว้แค่ที่ขอบของระบบ (`StripeGateway`) เท่านั้น

---

## Step 838: ทำไม DI ใน Ruby ไม่ต้องมี framework/DI container หนักแบบ Java/C#

### DI container ใน Java/C# ทำหน้าที่อะไร และทำไมถึงต้องมี

ในภาษาอย่าง Java หรือ C# ที่เป็น **static typing** และมี `interface` เป็นทางการ การ
inject dependency มักต้องพึ่งพา **DI Framework/Container** ขนาดใหญ่ (เช่น Spring ใน Java,
`Microsoft.Extensions.DependencyInjection` ใน .NET) เพราะเหตุผลเชิงโครงสร้างของภาษาเอง:

1. **ต้องประกาศ type ของ parameter ตายตัว** — constructor ต้องระบุว่ารับ
   `IPaymentGateway` (interface) ไม่ใช่แค่ "object ที่มี method ชื่อ charge" เหมือน Ruby
2. **การ "ต่อสาย" ว่า interface ไหนคู่กับ implementation ไหน ต้องประกาศไว้ล่วงหน้าอย่างเป็น
   ทางการ** (เช่น `services.AddScoped<IPaymentGateway, StripeGateway>()`) เพราะ compiler
   ต้องตรวจสอบ type ให้ตรงกันตั้งแต่ compile time
3. เมื่อ dependency graph ใหญ่ขึ้น (class A ต้องการ B, B ต้องการ C, C ต้องการ D) การเขียน
   `new A(new B(new C(new D())))` มือเปล่าจะน่าเบื่อและเสี่ยงพลาดมาก ต้องมี container ที่
   "resolve" graph นี้ให้อัตโนมัติ

### ทำไม Ruby ไม่ต้องมีสิ่งนี้ (ในกรณีส่วนใหญ่)

**เหตุผลข้อ 1 และ 2 หายไปเพราะ duck typing** — จากตัวอย่างทั้งหมดใน Part นี้ สังเกตว่า
`CheckoutService#initialize(payment_gateway:)` **ไม่เคยประกาศ type ใดๆ เลย** มันรับได้
ทั้ง `StripeGateway`, `FakePaymentGateway`, หรือ object ใดๆ ในอนาคตที่ตอบสนอง `#charge`
ได้ถูกต้อง — ไม่มี "การต่อสาย" ที่ต้องประกาศไว้ล่วงหน้าอย่างเป็นทางการเลย เพราะ Ruby ไม่
ตรวจสอบ type ตอน compile (Ruby ไม่มี compile step แบบนั้นด้วยซ้ำ — เป็น interpreted
language) การ "ต่อสาย" ทำได้ง่ายๆ แค่ **ส่ง object ที่ถูกต้องเข้าไปตรงจุดที่ต้องการ**:

```ruby
# นี่คือ "การต่อสาย" ทั้งหมดที่ต้องมี — ไม่มี config พิเศษ ไม่มี container ต้องตั้งค่า
CheckoutService.new(payment_gateway: StripeGateway.new(api_key: secret_key))
```

**เหตุผลข้อ 3 (dependency graph ที่ซับซ้อน) เกิดขึ้นได้จริงในแอปขนาดใหญ่** แต่ในภาษาที่มี
**keyword arguments พร้อม default value** และ **first-class function** อย่าง Ruby การ
เขียน factory method ธรรมดาก็มักเพียงพอ โดยไม่ต้องมี container framework แยกต่างหาก:

```ruby
# factory method ธรรมดา — ไม่ต้องมี DI container ก็จัดการ dependency graph ได้
class CheckoutServiceFactory
  def self.build
    CheckoutService.new(
      payment_gateway: StripeGateway.new(api_key: Rails.application.credentials.dig(:stripe, :secret_key))
    )
  end
end

# ใช้งาน
CheckoutServiceFactory.build.call(order: order, card_token: token)
```

### ตารางเปรียบเทียบตรงประเด็น

| | Java/C# (ต้องมี DI Container) | Ruby (ไม่จำเป็นต้องมี) |
|---|---|---|
| นิยาม "สัญญา" ของ dependency | ต้องมี `interface` ประกาศไว้ก่อนเสมอ | แค่ object ตอบสนอง method ที่ต้องการ (duck typing) |
| การ "ต่อสาย" implementation จริง | ต้องลงทะเบียนไว้ล่วงหน้าใน container (`AddScoped<T>`) | ส่ง object ตรงเข้า constructor ตรงจุดที่ต้องการได้เลย |
| Compile-time safety | Compiler เช็ค type ให้อัตโนมัติ | ไม่มี — ต้องพึ่งเทส (unit test) เป็นตัวยืนยันแทน (เหตุผลหนึ่งที่ทำไมการเขียนเทสสำคัญมากใน Ruby) |
| Dependency graph ซับซ้อน | Container resolve ให้อัตโนมัติทั้ง tree | Factory method/class ธรรมดามักเพียงพอ |

> **ข้อคิดสำคัญ:** DI container ไม่ใช่ "สิ่งที่ผิด" — มันแก้ปัญหาที่**มีอยู่จริง**ในภาษา
> static typing ที่ syntax เข้มงวดกว่า Ruby มาก การที่ Ruby ไม่ต้องมี container แบบนั้น
> ไม่ได้แปลว่า Ruby "ดีกว่า" ในทุกมิติ — Ruby แลก compile-time safety (ที่ Java/C# มี) มา
> กับความยืดหยุ่นของ duck typing ซึ่งทำให้ไม่ต้องมี ceremony (พิธีกรรมทาง syntax) ของ
> interface/container แต่ก็ต้องพึ่งพา**เทสที่ครอบคลุม**เป็นตาข่ายนิรภัยแทน type checker

---

## Step 839: Lightweight DI container แบบทำเอง เทียบกับ `dry-container`/`dry-auto_inject`

### เมื่อไหร่ที่ "ส่ง object ตรงๆ" เริ่มไม่พอ

แนวทางใน Step 838 (ส่ง object เข้า constructor ตรงๆ, ใช้ factory method ธรรมดา) ใช้ได้ดี
กับแอปขนาดเล็ก-กลาง แต่เมื่อแอปโตขึ้นจนมี dependency ที่ต้อง "สลับตาม environment" หลาย
สิบตัว (payment gateway, email sender, SMS sender, feature flag client, cache client
ฯลฯ) การเขียน factory method แยกทุกตัวเริ่มซ้ำซ้อน — จุดนี้ทีมบางทีมเลือกใช้ **registry
pattern** แบบง่ายๆ ที่เรียกกันว่า **lightweight DI container**

### เขียน lightweight DI container เองด้วย Hash ธรรมดา

```ruby
# frozen_string_literal: true

# AppContainer คือ "registry" อย่างง่าย — เก็บ factory (Proc) ไว้ใน Hash แล้ว resolve
# (สร้าง instance จริง) แบบ lazy ตอนถูกเรียกใช้ครั้งแรกเท่านั้น พร้อม cache ไว้ใช้ซ้ำ
class AppContainer
  def initialize
    @factories = {}
    @instances = {}
  end

  def register(name, &factory)
    @factories[name] = factory
  end

  def resolve(name)
    @instances[name] ||= @factories.fetch(name).call(self)
  end
end
```

ทดลองใช้งานจริง:

```ruby
class FakeGateway
  def charge(**args) = "charged #{args}"
end

container = AppContainer.new
container.register(:payment_gateway) { FakeGateway.new }
container.register(:checkout_service) { |c| "CheckoutService with #{c.resolve(:payment_gateway).class}" }

puts container.resolve(:checkout_service)
# => CheckoutService with FakeGateway
```

**อธิบาย:** `register` เก็บ **วิธีสร้าง** (factory เป็น block) ไว้เฉยๆ ยังไม่สร้างจริง —
`resolve` เป็นจุดที่สร้าง instance จริงครั้งแรก แล้ว**เก็บ cache ไว้** (`@instances[name]
||= ...`) เพื่อไม่ต้องสร้างซ้ำทุกครั้งที่เรียก (รูปแบบนี้เรียกว่า **singleton scope** —
สร้างครั้งเดียวใช้ตลอดอายุของ container) และ factory แต่ละตัวรับ `container` เองเป็น
argument (`|c|`) ทำให้ dependency ที่ต้องการ dependency อื่นต่อ (`checkout_service`
ต้องการ `payment_gateway`) resolve กันเป็นทอดๆ ได้เอง — นี่คือแก่นของสิ่งที่ DI container
ทุกตัว (ไม่ว่าเล็กหรือใหญ่ ภาษาไหนก็ตาม) ทำ

ใช้งานจริงในบริบท Rails ผ่าน initializer:

```ruby
# config/initializers/app_container.rb
Rails.application.config.container = AppContainer.new.tap do |c|
  c.register(:payment_gateway) do
    if Rails.env.test?
      FakePaymentGateway.new
    else
      StripeGateway.new(api_key: Rails.application.credentials.dig(:stripe, :secret_key))
    end
  end

  c.register(:checkout_service) do |container|
    CheckoutService.new(payment_gateway: container.resolve(:payment_gateway))
  end
end
```

```ruby
# app/controllers/checkout_controller.rb
class CheckoutController < ApplicationController
  def create
    order = Order.find(params[:order_id])
    service = Rails.application.config.container.resolve(:checkout_service)

    result = service.call(order: order, card_token: params[:card_token])
    # ...
  end
end
```

ตอนนี้ **การตัดสินใจว่าจะใช้ adapter ตัวไหนถูกรวมไว้ที่จุดเดียว** (initializer) — controller
ไม่ต้องรู้เรื่อง `Rails.env.test?` เลยอีกต่อไป แค่ `resolve` ชื่อ service ที่ต้องการ

### ทางเลือกที่มีโครงสร้างมากกว่า: `dry-container` และ `dry-auto_inject`

สำหรับทีมที่ต้องการโครงสร้างที่เป็นระบบกว่า `AppContainer` ที่เขียนเอง มี gem ในตระกูล
[dry-rb](https://dry-rb.org) สองตัวที่นิยมใช้คู่กัน:

- **`dry-container`** — DI container ที่มีความสามารถใกล้เคียงกับ `AppContainer` ข้างบน
  แต่เพิ่มฟีเจอร์อย่าง namespacing, การ merge หลาย container เข้าด้วยกัน, และ integration
  กับ `dry-system` สำหรับ auto-loading component ขนาดใหญ่
- **`dry-auto_inject`** — ทำงานคู่กับ `dry-container` เพื่อ **generate constructor
  injection ให้อัตโนมัติ** ผ่าน syntax แบบ `include Import["payment_gateway"]` แทนการ
  เขียน `def initialize(payment_gateway:)` ซ้ำๆ ทุก class

```ruby
# ตัวอย่างหน้าตาคร่าวๆ ของ dry-auto_inject (ต้องติดตั้ง dry-auto_inject และตั้งค่า
# container แยกต่างหากก่อนจึงจะรันได้จริง — แสดงไว้เพื่อเปรียบเทียบ syntax เท่านั้น)
class CheckoutService
  include Import["payment_gateway"]

  def call(order:, card_token:)
    # payment_gateway พร้อมใช้งานทันทีโดยไม่ต้องเขียน initialize เอง
  end
end
```

### ตารางเปรียบเทียบเพื่อช่วยตัดสินใจ

| | `AppContainer` เขียนเอง | `dry-container` + `dry-auto_inject` |
|---|---|---|
| ต้องเพิ่ม dependency ใหม่ | ไม่ต้อง (เป็นโค้ด Ruby ธรรมดา ~15 บรรทัด) | ต้องเพิ่ม gem 2 ตัว |
| Learning curve | ต่ำมาก — ทีมอ่านโค้ดเข้าใจได้ทันที | ต้องเรียนรู้ syntax และ convention ของ dry-rb เพิ่ม |
| ความสามารถ | พื้นฐาน: register + resolve + cache | ครบเครื่องกว่า: namespace, auto-inject, merge container, lazy loading ขั้นสูง |
| เหมาะกับ | แอปขนาดเล็ก-กลาง ที่มี dependency ให้ inject ไม่กี่สิบตัว | แอปขนาดใหญ่มากที่แยกเป็นหลาย bounded context หรือใช้ `dry-system`/Hanami อยู่แล้ว |

> **คำแนะนำเชิงปฏิบัติ:** สำหรับแอป Rails ทั่วไป **`AppContainer` แบบง่ายๆ ที่เขียนเอง (หรือ
> แม้แต่แค่ factory method ธรรมดาแบบ Step 838) ก็มักเพียงพอตลอดอายุของโปรเจกต์** การเพิ่ม
> gem ตระกูล dry-rb เข้ามาคุ้มค่าก็ต่อเมื่อทีมมีปัญหาจัดการ dependency graph ที่ซับซ้อน
> มากจริงๆ อยู่แล้ว (มักเกิดกับแอปที่แยกเป็นหลาย domain module ชัดเจน หรือใช้
> [Hanami framework](https://hanamirb.org) ที่มี `dry-system` เป็นหัวใจอยู่แล้ว) — อย่า
> เพิ่มความซับซ้อนของ dependency (gem) เพื่อแก้ปัญหาที่ยังไม่เกิดขึ้นจริง

---

## Step 840: ประโยชน์ด้าน testing, คำตัดสินใจที่ตรงไปตรงมา, และแบบฝึกหัดรวบยอด

### ทำไมโครงสร้างแบบนี้ถึงทำให้เทสง่ายขึ้นอย่างเป็นรูปธรรม

สรุปประโยชน์ด้าน testing ที่เห็นได้จากตัวอย่างทั้งหมดใน Part นี้:

1. **`OrderTotal` เทสได้โดยไม่พึ่งฐานข้อมูลเลย** (Step 834) — เทส 3 เคสรันเสร็จใน
   0.00366 วินาที เทียบกับการต้อง `Order.create!` จริงทุกครั้งถ้าเขียนสูตรไว้ใน
   ActiveRecord model
2. **สลับ `FakePaymentGateway` ↔ `StripeGateway` ได้โดยไม่แก้ `CheckoutService`** (Step
   837) — พิสูจน์ว่า port ทำงานตามสัญญาจริง และเทสหลักส่วนใหญ่ไม่ต้อง stub Stripe SDK
   ที่ซับซ้อนเลย ใช้ `FakePaymentGateway` ธรรมดาพอ
3. **`FakePaymentGateway` เป็น spy ที่ตรวจสอบว่าถูกเรียกด้วยค่าอะไรบ้าง** (Step 837) —
   ทำให้เทสยืนยันได้ว่า `CheckoutService` ส่งจำนวนเงินที่ถูกต้อง (ที่มาจาก `OrderTotal`)
   ไปให้ payment gateway จริง ไม่ใช่แค่เช็คผลลัพธ์ปลายทางเฉยๆ
4. **การทดสอบ error case (`succeed: false`) ทำได้โดยไม่ต้องพยายามทำให้ Stripe จริง
   ปฏิเสธการ์ด** — เปลี่ยน flag เดียวใน `FakePaymentGateway` ก็จำลองทุกสถานการณ์ได้ทันที

ทั้งหมดนี้เป็นไปได้เพราะ **dependency ถูก inject เข้ามา ไม่ใช่ hardcode ไว้ข้างใน** — ถ้า
`CheckoutService` เขียน `Stripe::Charge.create` ตรงๆ ข้างในตัวมันเอง (ไม่ inject
`payment_gateway`) ทุกเทสของ `CheckoutService` จะต้อง mock `Stripe::Charge` เองซ้ำๆ ทุก
ที่ที่ใช้ ซึ่งเปราะบางกว่ามาก (ถ้า Stripe เปลี่ยน API ต้องไปแก้ mock ในทุกไฟล์เทส แทนที่
จะแก้ที่ `StripeGateway` ที่เดียว)

### คำตัดสินใจที่ตรงไปตรงมา (honest verdict)

หลังจากเห็นทั้งประโยชน์และต้นทุนตลอด Part นี้แล้ว ข้อสรุปที่ควรติดตัวไปใช้งานจริงคือ:

> **Full Clean Architecture (Entity + Repository แยกจาก ActiveRecord อย่างสมบูรณ์แบบ
> Step 832) แทบไม่คุ้มค่าสำหรับแอป Rails แบบ CRUD-heavy ทั่วไป** — boilerplate ที่เพิ่มขึ้น
> และการเสียความสามารถของ Rails ecosystem (generator, gem ที่คาดหวัง ActiveRecord model
> ตรงๆ) มักมากกว่าประโยชน์ที่ได้ สำหรับแอปส่วนใหญ่ที่ทีมพัฒนาจริงสร้างกัน
>
> **แต่หลักการเบื้องหลัง (Dependency Inversion + แยก business logic ที่ซับซ้อน/มีค่าออก
> จากรายละเอียด framework เมื่อคุ้มค่า) ให้ผลตอบแทนที่คุ้มค่ามาก แม้ในแอป Rails ทั่วไปที่
> ไม่ได้ทำ Clean Architecture เต็มรูปแบบเลย** — การมี `OrderTotal` ที่เป็น PORO, การ
> inject `PaymentGateway` แทนการ hardcode `StripeGateway` เป็นสิ่งที่ **ทำได้เพิ่มขึ้นทีละ
> จุด** โดยไม่ต้อง refactor ทั้งแอปให้เป็น hexagon ทั้งกระบิ้ง

**กฎการตัดสินใจที่ใช้ได้จริง:** ถามตัวเองว่า "โค้ดส่วนนี้ถ้าดึงออกมาเป็น PORO แล้วจะทดสอบ
ง่ายขึ้นชัดเจนไหม หรือมีโอกาสต้องสลับ implementation (payment provider, notification
channel) ในอนาคตไหม" ถ้าใช่ทั้งคู่หรือข้อใดข้อหนึ่งชัดเจน **ให้ดึงออกมาและ inject
dependency** ถ้าเป็นแค่ CRUD ธรรมดาที่ไม่มีความซับซ้อนทางธุรกิจจริง **ให้ใช้ Rails Way
ตามปกติ** ไม่ต้องสร้างชั้นนามธรรมเพิ่มโดยไม่จำเป็น

---

### แบบฝึกหัด: ระบบ Checkout ที่ทดสอบได้เต็มรูปแบบด้วย Domain Object + Injected Port

### โจทย์

จากทุกไฟล์ที่สร้างไว้ตลอด Part นี้ (`OrderTotal`, `PaymentResult`, `StripeGateway`,
`FakePaymentGateway`, `CheckoutService`) ให้ทำสิ่งต่อไปนี้เพิ่มเติมเพื่อพิสูจน์ความเข้าใจ:

1. เพิ่ม method `#refund` เข้าไปใน port ของ payment gateway — ทั้ง `StripeGateway` และ
   `FakePaymentGateway` ต้อง implement `refund(transaction_id:)` ที่คืนค่าเป็น
   `PaymentResult` เหมือนกัน (`success: true/false`)
2. เขียน `RefundService` (Service Object ใหม่) ที่รับ `payment_gateway:` ทาง constructor
   เหมือน `CheckoutService` และมี method `call(order:)` ที่เรียก `refund` ด้วย
   `order.payment_transaction_id` แล้วอัปเดต `order.status` เป็น `"refunded"` ถ้าสำเร็จ
3. เขียนเทสให้ `RefundService` ด้วย `FakePaymentGateway` ครบทั้งกรณีสำเร็จและล้มเหลว

### เฉลย

**ขั้นที่ 1 — เพิ่ม `#refund` เข้า port ทั้งสอง adapter:**

```ruby
# frozen_string_literal: true
# app/domain/stripe_gateway.rb (เพิ่ม method ใหม่ต่อจากเดิม)
class StripeGateway
  def initialize(api_key:)
    @api_key = api_key
  end

  def charge(amount_cents:, currency:, source:)
    stripe_charge = Stripe::Charge.create(
      { amount: amount_cents, currency: currency, source: source },
      { api_key: @api_key }
    )
    PaymentResult.new(success: true, transaction_id: stripe_charge.id, error_message: nil)
  rescue Stripe::CardError => e
    PaymentResult.new(success: false, transaction_id: nil, error_message: e.message)
  end

  def refund(transaction_id:)
    refund = Stripe::Refund.create({ charge: transaction_id }, { api_key: @api_key })
    PaymentResult.new(success: true, transaction_id: refund.id, error_message: nil)
  rescue Stripe::InvalidRequestError => e
    PaymentResult.new(success: false, transaction_id: nil, error_message: e.message)
  end
end
```

```ruby
# frozen_string_literal: true
# app/domain/fake_payment_gateway.rb (เพิ่ม method ใหม่ต่อจากเดิม)
class FakePaymentGateway
  attr_reader :charges, :refunds

  def initialize(succeed: true)
    @succeed = succeed
    @charges = []
    @refunds = []
  end

  def charge(amount_cents:, currency:, source:)
    @charges << { amount_cents: amount_cents, currency: currency, source: source }

    if @succeed
      PaymentResult.new(success: true, transaction_id: "fake_txn_#{@charges.size}", error_message: nil)
    else
      PaymentResult.new(success: false, transaction_id: nil, error_message: "การ์ดถูกปฏิเสธ (จำลอง)")
    end
  end

  def refund(transaction_id:)
    @refunds << { transaction_id: transaction_id }

    if @succeed
      PaymentResult.new(success: true, transaction_id: "fake_refund_#{@refunds.size}", error_message: nil)
    else
      PaymentResult.new(success: false, transaction_id: nil, error_message: "คืนเงินไม่สำเร็จ (จำลอง)")
    end
  end
end
```

**ขั้นที่ 2 — `RefundService`:**

```ruby
# frozen_string_literal: true
# app/services/refund_service.rb

# RefundService รับ payment_gateway ทาง constructor เหมือน CheckoutService ทุกประการ —
# ใช้ port เดียวกัน จึงสลับ adapter ได้แบบเดียวกันโดยไม่ต้องเขียนโครงสร้าง DI ใหม่เลย
class RefundService
  def initialize(payment_gateway:)
    @payment_gateway = payment_gateway
  end

  def call(order:)
    result = @payment_gateway.refund(transaction_id: order.payment_transaction_id)

    order.update!(status: "refunded") if result.success?

    result
  end
end
```

**ขั้นที่ 3 — เทส:**

```ruby
# frozen_string_literal: true
# spec/services/refund_service_spec.rb

require "rails_helper"

RSpec.describe RefundService do
  let(:order) do
    Order.create!(
      customer_email: "manee@example.com",
      status: "paid",
      total_cents: 74_633,
      payment_transaction_id: "fake_txn_1"
    )
  end

  context "เมื่อคืนเงินสำเร็จ" do
    it "อัปเดตสถานะ order เป็น refunded" do
      fake_gateway = FakePaymentGateway.new(succeed: true)
      service = described_class.new(payment_gateway: fake_gateway)

      result = service.call(order: order)

      expect(result).to be_success
      order.reload
      expect(order.status).to eq("refunded")
      expect(fake_gateway.refunds).to eq([{ transaction_id: "fake_txn_1" }])
    end
  end

  context "เมื่อคืนเงินไม่สำเร็จ" do
    it "ไม่เปลี่ยนสถานะ order" do
      fake_gateway = FakePaymentGateway.new(succeed: false)
      service = described_class.new(payment_gateway: fake_gateway)

      result = service.call(order: order)

      expect(result).not_to be_success
      order.reload
      expect(order.status).to eq("paid")
    end
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/services/refund_service_spec.rb
```

```
..

Finished in 0.05213 seconds (files took 1.21 seconds to load)
2 examples, 0 failures
```

**สิ่งที่เฉลยนี้พิสูจน์:** การขยาย port ที่มีอยู่แล้ว (`PaymentGateway`) ด้วย method ใหม่
(`refund`) และการเพิ่ม service ใหม่ (`RefundService`) ที่ใช้ port เดิม **ไม่ต้องเขียน
โครงสร้าง DI ใหม่เลยแม้แต่นิดเดียว** — เพราะ `FakePaymentGateway`/`StripeGateway` ถูก
ออกแบบมาให้เป็น adapter ของ "บริการชำระเงิน" ทั้งบริการ ไม่ใช่แค่ของ action เดียว การเพิ่ม
ความสามารถใหม่จึงทำได้โดยขยาย adapter เดิม แทนที่จะสร้างระบบ injection คู่ขนานใหม่ทุกครั้ง

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เขียน `NullPaymentGateway` เป็น adapter ตัวที่สาม (นอกเหนือจาก `StripeGateway` และ
   `FakePaymentGateway`) ที่ implement `charge`/`refund` เหมือนกันทุกประการ แต่คืน
   `PaymentResult.new(success: true, transaction_id: "N/A", error_message: nil)` เสมอ
   โดยไม่บันทึกอะไรเลย ใช้สำหรับ environment `development` ที่นักพัฒนาอยากทดลองหน้า UI
   checkout โดยไม่ต้องพึ่ง Stripe test mode หรือ `FakePaymentGateway` ที่มีไว้เพื่อเทส
   โดยเฉพาะ (ใบ้: นี่คือรูปแบบ **Null Object Pattern** ที่ผสมกับแนวคิด port/adapter
   ของ Part นี้)

2. ปรับ `AppContainer` จาก Step 839 ให้รองรับการ "override" การลงทะเบียนใน spec
   (เช่น เพิ่ม method `#register!` ที่บังคับเขียนทับ factory เดิมได้ แม้จะ `resolve`
   ไปแล้วครั้งหนึ่ง) แล้วเขียนเทสยืนยันว่าหลัง `register!` ใหม่ การเรียก `resolve` ครั้ง
   ถัดไปต้องได้ instance ใหม่ ไม่ใช่ instance เก่าที่ cache ไว้

3. ลองเขียน `Order` model จริงให้มี business logic เพิ่มเติมที่ **ควร** อยู่ใน
   ActiveRecord model ตามปกติ (ไม่ต้องดึงออกเป็น PORO) เช่น scope `Order.paid`,
   validation ว่า `discount_percentage` ต้องอยู่ระหว่าง 0-100 — แล้วเขียนสั้นๆ อธิบายว่า
   ทำไม logic เหล่านี้ **ไม่จำเป็นต้อง** ดึงออกมาเป็น PORO ตามเกณฑ์ตัดสินใจใน Step 833
   (ฝึกแยกแยะว่าอะไรควรอยู่ที่ไหน ไม่ใช่ทุกอย่างต้องถูกดึงออกมาเสมอไป)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจแนวคิดหลักของ **Clean Architecture / Hexagonal Architecture** ผ่าน "Ports and
  Adapters" — business logic อยู่ตรงกลาง, framework/ฐานข้อมูล/บริการภายนอกอยู่ที่ขอบ, และ
  **Dependency Rule** ที่ dependency ต้องชี้เข้าหาศูนย์กลางเสมอ ไม่ใช่ชี้ออก
- เข้าใจความตึงเครียดที่ตรงไปตรงมากับ Rails Way — ActiveRecord model รวม domain object
  กับ persistence layer ไว้ในตัวเดียวโดยธรรมชาติ ทำให้ Full Clean Architecture (แยก
  Entity/Repository จาก ActiveRecord อย่างสมบูรณ์) มักไม่คุ้มค่าสำหรับแอป CRUD-heavy ทั่วไป
- รู้จักและใช้ **ทางสายกลาง**: ปล่อยให้ ActiveRecord ทำหน้าที่ persistence ตามปกติ แต่ดึง
  business logic ที่ซับซ้อน/มีค่า ออกมาเป็น PORO เมื่อคุ้มค่า พร้อมเกณฑ์ตัดสินใจที่ใช้ได้จริง
- สร้าง **`OrderTotal`** จริง เป็น domain object ที่ไม่ require Rails/ActiveRecord เลย
  ทดสอบได้เร็วมาก (0.00366 วินาทีสำหรับ 3 เทส) และใช้งานร่วมกับทั้ง Struct ปลอมในเทสและ
  `ActiveRecord::Associations::CollectionProxy` จริงได้ทันทีผ่าน duck typing
- สร้าง **`CheckoutService`** เป็น Service Object ที่เชื่อมโลก ActiveRecord เข้ากับโลก
  domain — รู้จัก Rails เต็มตัวแต่มอบหมายการคำนวณให้ `OrderTotal` และการชำระเงินให้ port
  ที่ inject เข้ามา
- ทบทวน constructor injection จาก Part 016 และขยายไปสู่การ inject **"port" ทั้งก้อน**
  (`PaymentGateway`) ที่มีได้หลาย method และหลาย adapter — สร้าง **`StripeGateway`**
  (adapter จริง) กับ **`FakePaymentGateway`** (test double/spy) ที่ implement สัญญา
  เดียวกันทุกประการ พิสูจน์ด้วยเทสจริงว่าสลับกันได้โดยไม่แก้ `CheckoutService` เลย
- เข้าใจว่าทำไม Ruby ไม่ต้องมี DI framework/container หนักแบบ Java/C# Spring —
  duck typing ทำให้ "การต่อสาย" ทำได้แค่ส่ง object ตรงเข้า constructor โดยไม่ต้องประกาศ
  `interface` หรือลงทะเบียน container ล่วงหน้า
- เขียน **lightweight DI container** เองด้วย Hash registry ธรรมดา (`register`/`resolve`
  พร้อม lazy caching) และรู้จัก `dry-container`/`dry-auto_inject` เป็นทางเลือกที่มี
  โครงสร้างมากกว่าสำหรับทีมที่ต้องการ พร้อมคำแนะนำว่าเมื่อไหร่ควรใช้แบบไหน
- ได้ verdict ที่ตรงไปตรงมา: **Full Clean Architecture มักไม่คุ้มค่าสำหรับแอป Rails
  ทั่วไป แต่หลักการเบื้องหลัง (Dependency Inversion, แยก business logic ที่คุ้มค่าออกจาก
  framework) ให้ผลตอบแทนที่คุ้มค่ามากแม้ในแอปที่ไม่ได้ทำ Clean Architecture เต็มรูปแบบเลย**
- ฝึกขยายระบบจริงด้วยการเพิ่ม `RefundService` และ method `#refund` เข้าไปใน port เดิม
  โดยไม่ต้องสร้างโครงสร้าง DI ใหม่เลย เพราะ adapter ถูกออกแบบให้แทน "บริการทั้งบริการ"
  ตั้งแต่แรก

**ต่อไป (Part 085):** เราจะเรียนเรื่อง **Multi-tenancy** — การออกแบบแอปที่รองรับลูกค้า
หลายราย (tenant) ในระบบเดียวกัน เปรียบเทียบสองแนวทางหลัก **row-based** (ทุก tenant ใช้
ตารางเดียวกัน แยกด้วยคอลัมน์ `tenant_id`) กับ **schema-based** (แต่ละ tenant มี schema
ของตัวเองในฐานข้อมูลเดียวกัน) พร้อมลงมือใช้ gem **Apartment** เพื่อทำ schema-based
multi-tenancy ใน Rails จริง — และจะเห็นว่าหลักการ Dependency Injection ที่เพิ่งเรียนใน
Part นี้ (โดยเฉพาะการ inject "context" ของ tenant ปัจจุบันเข้าไปแทนการ hardcode) มีบทบาท
สำคัญกับการออกแบบระบบ multi-tenant ที่ทดสอบและดูแลรักษาได้ง่ายเช่นกัน
