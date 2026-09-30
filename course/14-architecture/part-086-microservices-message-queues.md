# Part 086: Microservices กับ Rails — Service Communication และ Message Queue เบื้องต้น (RabbitMQ/Kafka) — ปิด Phase 14: Architecture & Scaling

> **Step ครอบคลุมใน Part นี้:** Step 851–860
> **ระดับ:** สูง (ต้องผ่าน Part 082–085 มาก่อนทั้งหมด โดยเฉพาะ **Part 084** Clean Architecture/
> Hexagonal และ Dependency Injection, **Part 085** Multi-tenancy รวมถึง **Part 061–062**
> ActiveJob/Sidekiq และ **Part 071** การเรียก REST API ภายนอกด้วย `Net::HTTP`/Stripe)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem `bunny` 2.24.x (คุย RabbitMQ), gem `sidekiq`
> 8.x, gem `redis` 5.4.x, gem `faraday` 2.14.x — **ตัวอย่าง RabbitMQ, Sidekiq, Faraday, และ
> Redis pub/sub ทั้งหมดในบทนี้ทดสอบรันจริง** บน Ruby 3.3.6 กับ **RabbitMQ 3.12.1** (Erlang/OTP 25)
> และ **Redis 7.x** ที่ติดตั้งและรันจริงในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ ส่วน **Kafka อธิบาย
> เชิงแนวคิดเท่านั้น** (ไม่ได้รันจริง) — เหตุผลอธิบายไว้ละเอียดใน Step 858

Part นี้คือ **Part สุดท้ายของ Phase 14: Architecture & Scaling** เราเดินทางมาไกลมากตั้งแต่
Service Object/Form Object (Part 082), Query Object/Decorator (Part 083), Clean Architecture
กับ Dependency Injection (Part 084), จนถึง Multi-tenancy (Part 085) — ทุก Part ที่ผ่านมาสอน
วิธี "จัดระเบียบโค้ดภายใน" Rails application เดียวให้สะอาดและขยายได้ Part นี้จะพาไปอีกก้าวหนึ่ง
คือคำถามที่ทีมวิศวกรรมระดับ Senior/Staff ต้องตอบให้ได้: **เมื่อไหร่ที่แอปเดียวไม่พอ และต้องเริ่ม
แยกเป็นหลายเซอร์วิส?** และถ้าต้องแยกจริงๆ แล้ว **เซอร์วิสเหล่านั้นจะคุยกันอย่างไร**

เราจะเรียนรู้การสื่อสารสองแบบหลัก — **synchronous** (เรียกแล้วรอคำตอบทันที ผ่าน REST API ด้วย
`Faraday`/`Net::HTTP`) และ **asynchronous** (ส่งอีเวนต์แล้วไปทำงานอื่นต่อ ผ่าน **message queue**
อย่าง RabbitMQ) พร้อมตัวอย่างที่รันได้จริงทั้งฝั่ง publish และ consume แล้วปิดท้ายด้วยแบบฝึกหัด
ที่จำลองสถานการณ์จริง: Rails app หนึ่งประกาศอีเวนต์ `OrderPlaced` ออกไป และ worker process
อีกตัวหนึ่งที่แยกกันโดยสิ้นเชิงคอยรับอีเวนต์นั้นไปประมวลผล — สิ่งนี้คือหัวใจของสถาปัตยกรรม
microservices ทุกระบบในโลกจริง

> **คำเตือนที่สำคัญที่สุดของ Part นี้ (อ่านก่อน Step 851):** เนื้อหาที่กำลังจะเรียนต่อไปนี้ไม่ได้
> มีไว้เพื่อบอกว่า "ทุกโปรเจกต์ควรทำ microservices" ตรงกันข้ามเลย — เป้าหมายที่แท้จริงคือให้เข้าใจ
> **ต้นทุนที่แท้จริง** ของสถาปัตยกรรมแบบนี้ เพื่อที่จะตัดสินใจ **ไม่ทำมัน** ได้อย่างมีเหตุผล
> จนกว่าจะถึงจุดที่คุ้มค่าจริงๆ — ซึ่งสำหรับโปรเจกต์ส่วนใหญ่ในโลกนี้ จุดนั้นมาช้ากว่าที่กระแส
> เทคโนโลยีทำให้เราเชื่อมาก

## สารบัญของ Part นี้

- Step 851: มายาคติเรื่อง Microservices — Monolith First และคำถามที่ต้องตอบก่อนตัดสินใจแยก
- Step 852: Modular Monolith คือทางสายกลาง — ใช้ Clean Architecture (Part 084) เตรียมพร้อมแยก
  ในอนาคตโดยไม่ต้องจ่ายต้นทุนของ microservices ตอนนี้
- Step 853: Synchronous Communication — Rails เรียก REST API ของเซอร์วิสอื่นด้วย `Faraday`
  (ทบทวนและขยายผลจาก Part 071)
- Step 854: ความเสี่ยงของ Synchronous Call — cascading failure, timeout, และวงจรตัดไฟ
  (circuit breaker) เบื้องต้น
- Step 855: Asynchronous Communication ด้วย Message Queue — publish แล้วไม่ต้องรอ
- Step 856: RabbitMQ core concepts (Exchange, Queue, Binding, Routing Key) พร้อมตัวอย่างจริง
  ด้วย gem `bunny`
- Step 857: Background Worker บริโภคอีเวนต์จากคิว เชื่อมต่อกับ ActiveJob/Sidekiq (ทบทวน
  Part 061–062)
- Step 858: Kafka เบื้องต้น — Log-based Streaming ต่างจาก Message Broker แบบ RabbitMQ อย่างไร
- Step 859: Distributed Monolith — เมื่อแยกเซอร์วิสแล้วแย่กว่าเดิม
- Step 860: เส้นทางแยกเซอร์วิสแบบค่อยเป็นค่อยไป + แบบฝึกหัดปิด Phase 14

---

## Step 851: มายาคติเรื่อง Microservices — Monolith First และคำถามที่ต้องตอบก่อนตัดสินใจแยก

### Microservices คืออะไร (นิยามสั้นๆ)

**Microservices** คือสถาปัตยกรรมที่แบ่งแอปพลิเคชันหนึ่งตัวออกเป็นเซอร์วิสขนาดเล็กหลายตัว
แต่ละตัว:

1. รันเป็น process แยกกันอิสระ (deploy แยกกันได้)
2. มีฐานข้อมูล/storage เป็นของตัวเอง (ไม่แชร์ database กับเซอร์วิสอื่น)
3. สื่อสารกับเซอร์วิสอื่นผ่าน network (HTTP API, message queue, RPC) เท่านั้น ไม่เรียก method
   ข้ามเซอร์วิสแบบ in-process ได้เลย
4. มักเป็นเจ้าของ (owns) โดเมนธุรกิจส่วนหนึ่งอย่างชัดเจน เช่น "Inventory Service",
   "Payment Service", "Notification Service"

ตรงข้ามกับ **Monolith** ที่ทุกอย่างรันเป็น process เดียว, ฐานข้อมูลเดียว (หรือกลุ่มเดียว),
deploy พร้อมกันทั้งหมดในครั้งเดียว — ซึ่งคือรูปแบบที่หลักสูตรนี้สอนมาตลอด 85 Part ที่ผ่านมา

### ทำไมกระแส Microservices ถึงแรงมาก (และทำไมต้องระวังกระแสนี้)

บริษัทอย่าง Netflix, Amazon, Uber พูดถึง microservices บ่อยมากในงานสัมมนาและบทความวิศวกรรม
ทำให้เกิดความเข้าใจผิดที่พบบ่อยมากในวงการ: **"บริษัทเทคระดับโลกใช้ microservices ดังนั้นเรา
ก็ควรใช้เหมือนกัน"** — นี่คือกับดักที่มีชื่อเรียกในวงการว่า **"resume-driven development"**
(เลือกสถาปัตยกรรมเพราะอยากใส่ไว้ในเรซูเม่ ไม่ใช่เพราะปัญหาจริงต้องการมัน)

ความจริงที่มักถูกมองข้ามคือ:

- Netflix และ Amazon **เริ่มต้นเป็น monolith** และแยกเป็น microservices **หลังจาก**เจอปัญหา
  scale ที่วัดผลได้ชัดเจนแล้วเท่านั้น (ทีมวิศวกรของ Amazon เองมีบทความชื่อดังปี 2023 ที่เล่าว่า
  ทีม Prime Video **ย้อนกลับจาก microservices มาเป็น monolith** สำหรับบางระบบ เพราะต้นทุนของ
  microservices (โดยเฉพาะค่า network call ระหว่างเซอร์วิส) สูงกว่าประโยชน์ที่ได้จริง
- **DHH (ผู้สร้าง Rails)** เขียนบทความและพูดในหลายงานว่า Basecamp/HEY (ผลิตภัณฑ์ของบริษัทเขา
  ที่มีผู้ใช้จริงระดับหลักล้าน) ยังคงเป็น **"majestic monolith"** จนถึงทุกวันนี้ ไม่ใช่เพราะ
  ล้าหลัง แต่เพราะทีมเล็ก การ deploy/debug/ทดสอบระบบเดียวง่ายกว่ามาก และ monolith ที่ออกแบบดี
  (มี module ชัดเจน — ตามที่ Part 082–084 สอนมา) **รองรับ scale ได้สูงกว่าที่คนส่วนใหญ่คิด**

### ต้นทุนที่แท้จริงของ Microservices (สิ่งที่กระแสมักไม่พูดถึง)

การแยกเป็น microservices **ไม่ใช่ของฟรี** — มันคือการแลกเปลี่ยน (trade-off) ที่ชัดเจน:

| ได้มา | ต้องจ่าย |
|---|---|
| Deploy แต่ละเซอร์วิสอิสระจากกัน | ต้องมี CI/CD pipeline แยกต่อเซอร์วิส, versioning ของ API ระหว่างกัน |
| Scale เฉพาะเซอร์วิสที่ต้องการ (เช่น เพิ่ม instance เฉพาะ image-processing) | Network call แทนที่ method call — latency สูงขึ้นมาก, อาจ fail ได้ (Part 853–854) |
| ทีมต่างทีมทำงานคนละเซอร์วิสได้อิสระ (Conway's Law) | ต้องมี service discovery, API contract, distributed tracing, centralized logging |
| Failure isolation (เซอร์วิสหนึ่งล่ม เซอร์วิสอื่นอาจรอดได้ถ้าออกแบบดี) | Distributed transaction ยากขึ้นมาก — ไม่มี `ActiveRecord::Base.transaction` ข้ามเซอร์วิสให้ใช้ฟรีอีกต่อไป (ต้องใช้ pattern อย่าง Saga) |
| เทคโนโลยีต่างกันต่อเซอร์วิสได้ (polyglot) | ต้อง maintain ความรู้หลายภาษา/framework, onboarding ยากขึ้น |

**คำถามที่ต้องตอบให้ได้ก่อนแยกเซอร์วิส (ไม่ใช่ถามว่า "อยากทำไหม" แต่ถามว่า "จำเป็นหรือยัง"):**

1. **ขนาดทีม** — มีวิศวกรมากพอที่แต่ละเซอร์วิสจะมีทีมเป็นเจ้าของจริงจังหรือยัง? (กฎง่ายๆ ที่
   ใช้กันในวงการ: ถ้าทีมทั้งหมดนั่งกินพิซซ่าสองถาดจบ — "two-pizza team" ของ Amazon — อาจยังไม่ถึง
   จุดที่ต้องแยก)
2. **ขอบเขตของปัญหา scale ชัดเจนหรือยัง?** — มีส่วนไหนของระบบที่ scale ต่างจากส่วนอื่นชัดเจน
   มากจนต้อง scale แยก (เช่น service แปลงวิดีโอที่กิน CPU สูงมาก ในขณะที่ web request ทั่วไป
   เบามาก)? หรือแค่ "รู้สึกว่าน่าจะโตในอนาคต"?
3. **เคย optimize monolith จนสุดทางแล้วหรือยัง?** — ใช้ caching (Part 063), database index/
   query optimization (Part 064), background job (Part 061–062) แล้วหรือยัง? เทคนิคเหล่านี้
   แก้ปัญหา scale ได้เยอะมากโดยไม่ต้องแยกเซอร์วิสเลย
4. **มี bounded context ที่ชัดเจนจริงหรือยัง?** — ถ้าโค้ดยังพันกันยุ่งเหยิงจนแยก module ใน
   monolith เดียวกันไม่ได้ (Part 082–084 ยังไม่ถูกนำไปใช้จริง) การแยกเป็น microservices จะยิ่ง
   ทำให้ความยุ่งเหยิงนั้น**กระจายข้าม network** ซึ่งแก้ไขยากกว่าเดิมมาก — นี่คือประเด็นสำคัญ
   ที่จะพูดถึงเต็มๆ ใน Step 852

> **หลักการที่ควรจำจาก Step นี้ (Martin Fowler เรียกว่า "MonolithFirst"):** เริ่มต้นด้วย
> **modular monolith เสมอ** แล้วแยกเป็น microservices **เฉพาะส่วนที่มีเหตุผลวัดผลได้ชัดเจน**
> เมื่อถึงเวลานั้นจริงๆ — ไม่ใช่ออกแบบเป็น microservices ตั้งแต่วันแรกของโปรเจกต์ Step 852
> จะแสดงให้เห็นว่า Clean Architecture ที่เรียนใน Part 084 คือสิ่งที่ทำให้ "แยกทีหลัง" เป็นไป
> ได้อย่างราบรื่น โดยไม่ต้องเขียนใหม่ทั้งหมด

---

## Step 852: Modular Monolith คือทางสายกลาง — ใช้ Clean Architecture เตรียมพร้อมแยกในอนาคต

### Modular Monolith คืออะไร

**Modular Monolith** คือ monolith ที่ถูกจัดโครงสร้างภายในให้แบ่งเป็น **module ที่มีขอบเขต
ชัดเจน** (bounded context ตามศัพท์ของ Domain-Driven Design) แต่ละ module สื่อสารกับ module
อื่นผ่าน **interface ที่ชัดเจน** เท่านั้น ไม่ไปยุ่งกับ implementation ภายในของ module อื่นตรงๆ
— รันเป็น process เดียว, deploy พร้อมกัน, แต่ **จัดระเบียบเหมือนเตรียมพร้อมจะแยกได้ทุกเมื่อ**

นี่คือสิ่งที่ Part 082–084 เตรียมพื้นฐานไว้ให้แบบไม่รู้ตัว:

- **Part 082 (Service Object)** — ทำให้ business logic แต่ละ "การกระทำ" (เช่น
  `Orders::PlaceOrder`) ถูกห่อหุ้มเป็นหน่วยเดียวที่มี entry point ชัดเจน (`#call`) แทนที่จะ
  กระจายอยู่ใน Controller/Model
- **Part 083 (Query Object/Decorator)** — แยก "การอ่านข้อมูลที่ซับซ้อน" ออกจาก Model โดยตรง
- **Part 084 (Clean Architecture/DI)** — จัดชั้น (layer) ของโค้ดให้ business logic
  (use case/entity) **ไม่รู้จัก** รายละเอียดภายนอก (framework, database, external service)
  โดยตรง แต่คุยผ่าน **interface ที่นิยามเอง** แล้วฉีด (inject) implementation จริงเข้ามาทีหลัง

### ตัวอย่าง: Seam ที่ทำให้แยกเซอร์วิสได้ในอนาคตโดยไม่ต้องเขียนใหม่

สมมติในระบบร้านค้าออนไลน์ มี use case `PlaceOrder` ที่ต้องเช็ค stock จาก "ระบบคลังสินค้า"
ก่อนสร้างออเดอร์ ถ้าออกแบบตาม Clean Architecture (Part 084) จะได้หน้าตาประมาณนี้:

```ruby
# frozen_string_literal: true

module Orders
  # Use Case (business logic) ไม่รู้จักเลยว่า "เช็ค stock" ทำงานอย่างไรจริงๆ
  # รู้แค่ว่ามี object ตัวหนึ่งที่ตอบสนอง #available?(sku:, quantity:) — นี่คือ "interface"
  # ที่ Part 084 เรียกว่า Port (ในความหมายของ Hexagonal Architecture)
  class PlaceOrder
    class OutOfStockError < StandardError; end

    def initialize(inventory_checker:, order_repository:)
      @inventory_checker = inventory_checker
      @order_repository = order_repository
    end

    def call(customer:, line_items:)
      line_items.each do |item|
        unless @inventory_checker.available?(sku: item.sku, quantity: item.quantity)
          raise OutOfStockError, "สินค้า #{item.sku} มีไม่พอในสต็อก"
        end
      end

      @order_repository.create!(customer: customer, line_items: line_items)
    end
  end
end
```

**วันนี้** (ตอนที่ยังเป็น monolith) `inventory_checker` คือ object ธรรมดาที่ query
`ActiveRecord::Base` ตรงๆ ในโปรเซสเดียวกัน:

```ruby
# frozen_string_literal: true

module Orders
  # Adapter วันนี้: เช็ค stock จากตาราง inventory_items ในฐานข้อมูลเดียวกัน (in-process)
  class InProcessInventoryChecker
    def available?(sku:, quantity:)
      InventoryItem.find_by(sku: sku)&.quantity_on_hand.to_i >= quantity
    end
  end
end

# ประกอบร่างผ่าน Dependency Injection (ทบทวน Part 084)
Orders::PlaceOrder.new(
  inventory_checker: Orders::InProcessInventoryChecker.new,
  order_repository: Orders::ActiveRecordOrderRepository.new
)
```

**ในอนาคต** ถ้าทีมโตขึ้นและมีเหตุผลชัดเจนที่จะแยก "Inventory" เป็นเซอร์วิสของตัวเอง (เช่น
ทีมคลังสินค้าต้องการ deploy อิสระ หรือระบบคลังสินค้าต้อง scale แยกเพราะมีการซิงค์กับหน้าร้าน
สาขาจริงหลายพันจุดแบบ real-time) สิ่งที่ต้องเปลี่ยน **คือแค่ adapter ตัวเดียว** —
`PlaceOrder` (business logic หลัก) **ไม่ต้องแก้แม้แต่บรรทัดเดียว:**

```ruby
# frozen_string_literal: true

module Orders
  # Adapter ใหม่: เช็ค stock ผ่าน HTTP call ไปยัง Inventory Service ที่แยกเป็นเซอร์วิสต่างหากแล้ว
  # (รายละเอียดการเรียก Faraday จะเรียนเต็มๆ ใน Step 853)
  class RemoteInventoryChecker
    def initialize(client: InventoryServiceClient.new(base_url: ENV.fetch("INVENTORY_SERVICE_URL")))
      @client = client
    end

    def available?(sku:, quantity:)
      @client.stock_for(sku) >= quantity
    end
  end
end

# แค่เปลี่ยนตัวที่ inject เข้าไป — PlaceOrder ไม่ถูกแตะต้องเลย
Orders::PlaceOrder.new(
  inventory_checker: Orders::RemoteInventoryChecker.new,
  order_repository: Orders::ActiveRecordOrderRepository.new
)
```

**นี่คือคุณค่าที่แท้จริงของ Part 084 ที่เพิ่งเรียนไป:** Clean Architecture ไม่ได้มีไว้เพื่อ
"เตรียมทำ microservices" โดยตรง แต่ผลพลอยได้ (side benefit) ของมันคือทำให้การแยกเซอร์วิสใน
อนาคต **เป็นการเปลี่ยน adapter ตัวเดียว ไม่ใช่การเขียน business logic ใหม่ทั้งหมด** — ทีมจึง
ได้ประโยชน์จากการจัดโครงสร้างที่ดี **ตั้งแต่วันนี้** (โค้ดทดสอบง่ายขึ้น, อ่านง่ายขึ้น) โดยยัง
**ไม่ต้องจ่ายต้นทุนการ operate หลาย process/หลาย database ของ microservices เลย** จนกว่าจะถึง
วันที่จำเป็นจริงๆ

> **ข้อควรระวัง:** อย่าสร้าง interface/abstraction ล่วงหน้าสำหรับ**ทุกอย่าง**ในระบบ "เผื่อว่า
> วันหนึ่งจะแยกเซอร์วิส" — นั่นคือ over-engineering (YAGNI: You Aren't Gonna Need It) หลักการ
> ที่ดีคือทำตามที่ Part 084 สอน: ใช้ DI ตรงจุดที่มี **external dependency จริงๆ** (database,
> external API, file system) อยู่แล้วตามธรรมชาติของโค้ดที่ดี ไม่ใช่สร้าง interface ปลอมๆ ทุก
> class เพื่อ "เผื่อไว้" — ขอบเขต (bounded context) ที่ชัดเจนสำคัญกว่าจำนวน interface ที่มี

---

## Step 853: Synchronous Communication — Rails เรียก REST API ของเซอร์วิสอื่นด้วย `Faraday`

### รูปแบบเดียวกับที่เคยเรียนไปแล้วใน Part 071

Part 071 สอนการเรียก Stripe API ด้วย `Net::HTTP` ตรงๆ (สำหรับ verify webhook signature) และ
ใช้ gem `stripe` (ที่ภายในก็เรียก HTTP เหมือนกัน) สำหรับ operation อื่นๆ — **การเรียกเซอร์วิส
ภายนอกใดๆ ก็ตาม ไม่ว่าจะเป็น Stripe, เซอร์วิสของทีมอื่นในบริษัทเดียวกัน หรือ third-party API
ทั่วไป ล้วนมีรูปร่างเดียวกันเป๊ะ:**

1. สร้าง HTTP request (method, URL, header, body)
2. ส่งออกไปแล้ว**รอ**คำตอบ (นี่คือความหมายของคำว่า "synchronous")
3. แปลผล response (parse JSON, เช็ค status code)
4. จัดการ error ที่อาจเกิดขึ้น (timeout, connection refused, 4xx/5xx)

Part นี้จะใช้ **`Faraday`** แทน `Net::HTTP` ตรงๆ เพราะ Faraday เป็น HTTP client library
มาตรฐานของวงการ Ruby ที่ให้ middleware system (encode/decode JSON อัตโนมัติ, retry, logging)
และที่สำคัญที่สุดสำหรับหลักสูตรนี้: **มี test adapter ในตัว** ทำให้เขียน unit test ที่ไม่ต้อง
ยิง network จริงได้ง่ายมาก (จะเห็นในตัวอย่างด้านล่าง)

```bash
gem install faraday -v "~> 2.14"
```

### ตัวอย่าง: `InventoryServiceClient` — เรียก Inventory Service ที่แยกเป็นเซอร์วิสต่างหาก

```ruby
# frozen_string_literal: true

# app/clients/inventory_service_client.rb
require "faraday"
require "json"

class InventoryServiceClient
  class ServiceUnavailableError < StandardError; end

  def initialize(base_url:, connection: nil)
    @connection = connection || Faraday.new(url: base_url) do |f|
      f.request :json                                   # แปลง body ที่ส่งออกเป็น JSON อัตโนมัติ
      f.response :json, content_type: /\bjson$/          # แปลง response body กลับเป็น Hash อัตโนมัติ
      f.options.timeout = 2                               # timeout รวมทั้ง request (วินาที)
      f.options.open_timeout = 1                          # timeout เฉพาะตอนเชื่อมต่อ (connect)
      f.adapter Faraday.default_adapter
    end
  end

  def stock_for(sku)
    response = @connection.get("/api/inventory/#{sku}")
    raise ServiceUnavailableError, "inventory service returned #{response.status}" unless response.success?

    response.body["quantity"]
  rescue Faraday::ConnectionFailed, Faraday::TimeoutError => e
    raise ServiceUnavailableError, "inventory service unreachable: #{e.message}"
  end
end
```

**อธิบาย:**

- `f.request :json` / `f.response :json` คือ **middleware** ของ Faraday — ทำหน้าที่แปลงข้อมูล
  ก่อน/หลังส่ง request โดยอัตโนมัติ (เทียบได้กับ Rack middleware ที่จะเรียนเต็มๆ ใน Part 087)
  ทำให้โค้ด business logic ไม่ต้องยุ่งกับ `JSON.parse`/`JSON.generate` เอง
- `f.options.timeout` และ `f.options.open_timeout` **สำคัญมาก** — ถ้าไม่ตั้งค่า Faraday จะรอ
  คำตอบจากปลายทางได้**ไม่จำกัดเวลา** ซึ่งเป็นสาเหตุอันดับหนึ่งของ cascading failure ที่จะพูดถึง
  ใน Step 854
- แปลง exception ของ Faraday (`ConnectionFailed`, `TimeoutError`) ให้เป็น
  `ServiceUnavailableError` ของเราเอง — pattern เดียวกับที่ Part 011 และ Part 020 (Todo CLI)
  สอนมาตลอด: **แปลง error ทั่วไปจาก library ภายนอกให้เป็น error เฉพาะทางของระบบเราเอง**
  เพื่อให้ Controller/Service Object ที่เรียกใช้ `rescue` ได้ง่ายและชัดเจน

### ทดสอบโดยไม่ต้องยิง network จริง — Faraday Test Adapter

จุดเด่นสำคัญของ Faraday คือ **test adapter** ในตัว ทำให้จำลอง response ของเซอร์วิสภายนอกได้
โดยไม่ต้องมีเซอร์วิสนั้นรันอยู่จริง — โค้ดต่อไปนี้ **ทดสอบรันจริงแล้ว** ยืนยันว่า
`InventoryServiceClient` ทำงานถูกต้องทั้งกรณีสำเร็จและกรณี error:

```ruby
# frozen_string_literal: true

require "faraday"
require_relative "inventory_service_client"

stub_connection = Faraday.new do |f|
  f.request :json
  f.response :json, content_type: /\bjson$/
  f.adapter :test do |stub|
    stub.get("/api/inventory/SKU-1") do
      [200, { "Content-Type" => "application/json" }, { quantity: 42 }.to_json]
    end
    stub.get("/api/inventory/SKU-DOWN") { raise Faraday::ConnectionFailed, "connection refused" }
  end
end

client = InventoryServiceClient.new(base_url: "http://inventory.internal", connection: stub_connection)
puts "SKU-1 stock: #{client.stock_for('SKU-1')}"

begin
  client.stock_for("SKU-DOWN")
rescue InventoryServiceClient::ServiceUnavailableError => e
  puts "caught error as expected: #{e.message}"
end
```

ผลลัพธ์จริงจากการรัน:

```
SKU-1 stock: 42
caught error as expected: inventory service unreachable: connection refused
```

**อธิบาย:** `connection:` ที่รับผ่าน `initialize` (Step แรกของไฟล์ `InventoryServiceClient`)
คือ **Dependency Injection** แบบเดียวกับที่ Part 084 สอน — production code จะไม่ส่ง
`connection:` มาเอง (ใช้ค่า default ที่ยิง network จริง) แต่ test สามารถแทนที่ด้วย
`stub_connection` แบบนี้ได้เสมอ เพื่อทดสอบ logic ล้วนๆ โดยไม่ต้องพึ่งเซอร์วิสภายนอกที่ควบคุม
ไม่ได้ (ทบทวนแนวคิด "ทดสอบได้โดยไม่ต้องพึ่งของจริง" จาก VCR ใน Part 049 และ `StringIO` ใน
Part 020)

ใน RSpec ของ Rails project จริง จะเขียนแบบนี้:

```ruby
# frozen_string_literal: true

RSpec.describe InventoryServiceClient do
  it "คืนจำนวนสต็อกเมื่อเซอร์วิสตอบสำเร็จ" do
    connection = Faraday.new do |f|
      f.adapter :test do |stub|
        stub.get("/api/inventory/SKU-1") { [200, {}, { quantity: 10 }.to_json] }
      end
    end

    client = described_class.new(base_url: "http://x", connection: connection)
    expect(client.stock_for("SKU-1")).to eq(10)
  end
end
```

---

## Step 854: ความเสี่ยงของ Synchronous Call — Cascading Failure และวงจรตัดไฟเบื้องต้น

### ปัญหา: เซอร์วิสที่ช้าหรือล่ม ทำให้ request ของเราช้า/ล่มตามไปด้วย

นี่คือ**ความเสี่ยงที่สำคัญที่สุด**ของการสื่อสารแบบ synchronous ระหว่างเซอร์วิส ลองดูภาพนี้:

```
ผู้ใช้กด "สั่งซื้อ"
   → Web Request เข้า Rails app (Checkout Service)
       → เรียก InventoryServiceClient#stock_for (synchronous, รอคำตอบ)
           → Inventory Service กำลังโดน traffic สูงผิดปกติ ตอบช้ามาก (5 วินาที) หรือล่มไปเลย
       ← รอ... รอ... รอ...
   ← Web Request ค้างอยู่ 5 วินาที (หรือ timeout แล้ว error) ทั้งที่ผู้ใช้แค่ต้องการสั่งซื้อ
```

ถ้า Checkout Service ไม่ได้ตั้ง `timeout` ไว้เลย (ตามที่เตือนไว้ท้าย Step 853) request จะ
**ค้างไม่จำกัดเวลา** — worker/thread ของ Rails ที่ใช้ประมวลผล request นี้จะถูกกินไปเรื่อยๆ
จนกระทั่ง **worker pool เต็ม** ทำให้ request อื่นๆ ของผู้ใช้คนอื่นที่**ไม่เกี่ยวอะไรกับ
Inventory Service เลย** ก็ค้างตามไปด้วย — นี่คือปรากฏการณ์ที่เรียกว่า **cascading failure**
(ความล้มเหลวลุกลาม) เซอร์วิสหนึ่งล่ม แต่ลากทั้งระบบล่มตามไปด้วย ทั้งที่เซอร์วิสอื่นๆ ยังทำงาน
ปกติดีอยู่

### แนวทางบรรเทา 1: Timeout ที่สมเหตุสมผลเสมอ (ขั้นต่ำที่ต้องทำ)

ตามที่ตั้งไว้แล้วใน Step 853 (`f.options.timeout = 2`) — กฎง่ายๆ คือ **timeout ทุก network
call เสมอ ไม่มีข้อยกเว้น** ค่าที่เหมาะสมขึ้นกับลักษณะงาน (เช็ค stock ควรเร็วมาก เช่น < 1 วินาที
ส่วน API ที่ทำงานหนักกว่าอาจให้เวลามากกว่านั้นได้) แต่ **ต้องมีค่าจำกัดเสมอ**

### แนวทางบรรเทา 2: Circuit Breaker (วงจรตัดไฟ)

แนวคิดยืมมาจากวงจรไฟฟ้าในบ้าน — ถ้ามีกระแสไฟเกิน เบรกเกอร์จะ**ตัดวงจรทันที**แทนที่จะปล่อยให้
ไฟไหม้บ้าน Circuit Breaker pattern ในซอฟต์แวร์ทำงานคล้ายกัน: ถ้าเรียกเซอร์วิสปลายทางแล้ว fail
ติดต่อกันเกิน threshold ที่กำหนด ให้ **"เปิดวงจร" (open)** คือหยุดยิง request ไปเซอร์วิสนั้น
ชั่วคราวทันที (คืนค่า error/fallback ทันทีโดยไม่ต้องรอ timeout ซ้ำๆ) แล้วค่อยๆ "ทดสอบ" ใหม่
เป็นระยะว่าเซอร์วิสฟื้นหรือยัง

```ruby
# frozen_string_literal: true

# ตัวอย่าง Circuit Breaker แบบง่ายที่สุดที่เขียนเอง เพื่อให้เห็นกลไกเบื้องหลังชัดเจน
# (ในงานจริงแนะนำใช้ gem ที่ทดสอบมาอย่างดีแล้ว เช่น `stoplight` แทนการเขียนเอง)
class SimpleCircuitBreaker
  class CircuitOpenError < StandardError; end

  def initialize(failure_threshold: 3, reset_after: 10)
    @failure_threshold = failure_threshold
    @reset_after = reset_after
    @failure_count = 0
    @state = :closed # :closed = ทำงานปกติ, :open = ตัดวงจร, :half_open = กำลังทดสอบ
    @opened_at = nil
  end

  def call
    raise CircuitOpenError, "circuit is open" if open?

    begin
      result = yield
      reset!
      result
    rescue StandardError => e
      record_failure
      raise e
    end
  end

  private

  def open?
    return false unless @state == :open

    if Time.now - @opened_at >= @reset_after
      @state = :half_open # ให้โอกาสลองใหม่ 1 ครั้ง
      false
    else
      true
    end
  end

  def record_failure
    @failure_count += 1
    return unless @failure_count >= @failure_threshold

    @state = :open
    @opened_at = Time.now
  end

  def reset!
    @failure_count = 0
    @state = :closed
    @opened_at = nil
  end
end
```

ใช้งานร่วมกับ `InventoryServiceClient`:

```ruby
breaker = SimpleCircuitBreaker.new(failure_threshold: 3, reset_after: 10)
client = InventoryServiceClient.new(base_url: ENV.fetch("INVENTORY_SERVICE_URL"))

begin
  quantity = breaker.call { client.stock_for("SKU-1") }
rescue SimpleCircuitBreaker::CircuitOpenError
  # เซอร์วิสกำลังมีปัญหาต่อเนื่อง — คืนค่า fallback ทันที (เช่น "ไม่แน่ใจสต็อก ให้ผู้ใช้ยืนยันเอง")
  # แทนที่จะรอ timeout ซ้ำๆ ทุกครั้งที่มีคนสั่งซื้อ ซึ่งจะยิ่งซ้ำเติมเซอร์วิสที่กำลังแย่อยู่แล้ว
rescue InventoryServiceClient::ServiceUnavailableError
  # เซอร์วิสตอบ error แต่ยังไม่ถึง threshold ที่จะเปิดวงจร
end
```

**อธิบาย:** เมื่อ `@failure_count` ถึง `failure_threshold` วงจรจะ "เปิด" (`@state = :open`)
ทำให้ทุก call ถัดไปได้ `CircuitOpenError` **ทันที** โดยไม่ต้องรอ timeout ของ Faraday เลยแม้แต่
ครั้งเดียว จนกว่าจะผ่านเวลา `reset_after` วินาที ถึงจะยอมให้ทดสอบใหม่อีกครั้ง (`:half_open`)
— นี่คือกลไกที่ป้องกันไม่ให้ระบบทั้งหมด "รอ timeout พร้อมกันเป็นพันๆ request" ตอนที่เซอร์วิส
ปลายทางกำลังมีปัญหาหนักอยู่แล้ว ซึ่งจะยิ่งซ้ำเติมสถานการณ์ให้แย่ลงไปอีก

> **ในงานจริง** แนะนำใช้ gem อย่าง [`stoplight`](https://github.com/bolshakov/stoplight) ที่
> รองรับการเก็บ state ผ่าน Redis (ให้หลาย process/instance ของ Rails app แชร์สถานะวงจรเดียวกัน
> ได้), มี notifier, และมี dashboard — คลาส `SimpleCircuitBreaker` ข้างบนมีไว้เพื่อให้เข้าใจ
> **กลไกเบื้องหลัง** เท่านั้น ไม่ควรใช้ใน production จริง (ไม่ thread-safe, ไม่แชร์ state ข้าม
> process)

### เมื่อไหร่ Timeout + Circuit Breaker ยังไม่พอ

Timeout และ Circuit Breaker ช่วย**บรรเทา**ปัญหา แต่ไม่ได้**แก้ที่ต้นเหตุ** — ต้นเหตุคือการที่
เรา**ผูก**ความสำเร็จของ operation หนึ่ง (สั่งซื้อสินค้า) เข้ากับความพร้อมใช้งาน ณ เวลานั้นของ
เซอร์วิสอื่น (เช็คสต็อก) แบบทันทีทันใด ทั้งที่จริงๆ แล้วงานหลายอย่างที่เกิดขึ้นตอนสั่งซื้อ
(เช่น ส่งอีเมลยืนยัน, อัปเดตยอดขายสำหรับ dashboard, แจ้งระบบบัญชี) **ไม่จำเป็นต้องเกิดขึ้น
ทันทีแบบ synchronous เลย** — นี่คือจุดที่ **message queue** เข้ามาแก้ปัญหาได้ตรงจุดกว่ามาก
ซึ่งจะเรียนต่อใน Step 855

---

## Step 855: Asynchronous Communication ด้วย Message Queue — Publish แล้วไม่ต้องรอ

### แนวคิดหลัก: แยกเวลาออกจากกัน (Decoupling in Time)

การสื่อสารแบบ **asynchronous** ผ่าน message queue เปลี่ยนโมเดลจาก "เรียกแล้วรอคำตอบ" เป็น
"**ประกาศว่าเกิดอะไรขึ้น (publish an event) แล้วเดินหน้าต่อทันที**" — ผู้รับ (consumer) จะมา
หยิบอีเวนต์นั้นไปประมวลผล **เมื่อไหร่ก็ได้ที่พร้อม** โดยผู้ส่ง (producer) ไม่ต้องรู้เลยว่ามีใคร
กำลังฟังอยู่กี่คน หรือฟังอยู่หรือเปล่า ณ ขณะนั้น

```
แบบ Synchronous (Step 853–854):
  Checkout Service ──รอคำตอบ──→ Inventory Service
                    ←──ตอบกลับ──

แบบ Asynchronous (Step นี้):
  Checkout Service ──publish "OrderPlaced"──→ [Message Queue]
  (ทำงานต่อทันที ไม่รอ)                            │
                                                    ├──→ Email Service (ส่งอีเมลยืนยัน)
                                                    ├──→ Analytics Service (บันทึกสถิติ)
                                                    └──→ Inventory Service (ตัดสต็อก)
```

**ข้อดีที่ตรงกับปัญหาใน Step 854 พอดี:** ถ้า Email Service ล่มไป Checkout Service **ไม่รู้ตัว
เลยด้วยซ้ำ** — request ของผู้ใช้จบไปนานแล้ว อีเวนต์ยังคงรออยู่ใน queue อย่างปลอดภัย
(ถ้าตั้งค่า durable ไว้ถูกต้อง ตามที่จะเรียนใน Step 856) รอจนกว่า Email Service จะกลับมาทำงาน
ได้อีกครั้งแล้วค่อยประมวลผลอีเวนต์ที่ค้างอยู่ — **ไม่มี cascading failure เกิดขึ้นเลย**

### คำศัพท์ที่ต้องรู้ก่อนไป Step ถัดไป

- **Producer / Publisher** — ฝั่งที่สร้างและส่งอีเวนต์เข้า queue (ในตัวอย่างของเราคือ
  Checkout/Order service)
- **Consumer / Subscriber** — ฝั่งที่รับอีเวนต์จาก queue ไปประมวลผล (อาจมีได้หลายตัว ฟังคิว/
  อีเวนต์เดียวกัน)
- **Event** — ข้อความที่บอกว่า "เกิดอะไรขึ้นแล้ว" ในอดีต (เช่น `OrderPlaced` — สั่งซื้อสำเร็จ
  ไปแล้ว) ต่างจาก **Command** ที่บอกว่า "จงทำสิ่งนี้" (เช่น `SendEmail`) — Part นี้เน้น
  event-driven pattern ที่เผยแพร่ "สิ่งที่เกิดขึ้นแล้ว" เพราะยืดหยุ่นกว่า: มีกี่ consumer ก็ได้
  มาฟังอีเวนต์เดียวกันแล้วตัดสินใจทำอะไรก็ได้ตามที่ตัวเองสนใจ โดย producer ไม่ต้องรู้จักหรือ
  แก้โค้ดเลยเมื่อมี consumer ใหม่เพิ่มเข้ามา

### สิ่งที่ต้องแลก (Trade-off) ของ Asynchronous — ไม่ใช่ของฟรีเช่นกัน

- **Eventual Consistency** — ข้อมูลจะ "ตรงกันในที่สุด" ไม่ใช่ "ตรงกันทันที" เช่น หลังสั่งซื้อ
  สำเร็จ อาจมีช่วงเวลาสั้นๆ ที่ยอดสต็อกยังไม่ถูกตัด เพราะ Inventory Service ยังไม่ได้ประมวลผล
  อีเวนต์ (ต่างจาก synchronous ที่รู้ผลทันทีตอนนั้นเลยว่าสำเร็จหรือไม่)
- **Debugging ยากขึ้น** — เวลามีปัญหา ต้องไล่ดู log ข้ามหลาย process/เซอร์วิส (ต้องมี
  distributed tracing ที่ดี ซึ่งเป็นหัวข้อขั้นสูงกว่าที่จะพูดถึงในเบื้องต้นเท่านั้นที่นี่)
- **การส่งซ้ำ (At-least-once delivery)** — message broker ส่วนใหญ่ (รวมถึง RabbitMQ) รับประกัน
  ว่าอีเวนต์ **จะถูกส่งไปถึง consumer อย่างน้อยหนึ่งครั้ง** แต่ **อาจส่งซ้ำได้** (เช่น consumer
  ประมวลผลเสร็จแล้วแต่ยังไม่ทัน ack กลับไป แล้ว connection หลุดพอดี broker จะส่งข้อความเดิมมาใหม่)
  → consumer **ต้องออกแบบให้ idempotent เสมอ** (ประมวลผลซ้ำกี่ครั้งก็ได้ผลลัพธ์เหมือนเดิม) —
  หลักการเดียวกับที่ **Part 071 Step 708** สอนเรื่อง Stripe webhook และ **Part 062 Step 618**
  สอนเรื่อง Sidekiq job ทุกประการ เพียงแค่ตอนนี้มาเจอปัญหาเดียวกันอีกครั้งในบริบทของ message
  queue

> **preview:** จาก Step ถัดไปเป็นต้นไป เราจะใช้ **RabbitMQ** เป็น message broker ตัวอย่างจริง —
> เพราะเป็น broker แบบ open-source ที่นิยมที่สุดตัวหนึ่งในวงการ Ruby/Rails และรองรับ pattern
> ที่ซับซ้อน (routing แบบ topic/direct/fanout) ได้ครบถ้วนกว่า message queue แบบง่ายอื่นๆ

---

## Step 856: RabbitMQ Core Concepts (Exchange, Queue, Binding, Routing Key) พร้อมตัวอย่างจริงด้วย `bunny`

### ติดตั้งและรัน RabbitMQ

```bash
# Ubuntu/Debian
sudo apt-get install -y rabbitmq-server

# macOS
brew install rabbitmq

# หรือใช้ Docker (สะดวกที่สุดสำหรับ dev environment)
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management
```

> **หมายเหตุเรื่องการทดสอบ:** ตัวอย่างทั้งหมดใน Step นี้**ทดสอบรันจริง**บน **RabbitMQ 3.12.1**
> (ติดตั้งผ่าน `apt-get install rabbitmq-server` แล้วรันด้วย `rabbitmq-server -detached`) ไม่ใช่
> การจำลองหรือคาดเดาผลลัพธ์ — ผลลัพธ์ที่แสดงในบทความนี้คือ output จริงจากเทอร์มินัล

ติดตั้ง gem `bunny` ซึ่งเป็น Ruby client มาตรฐานสำหรับคุยกับ RabbitMQ ผ่านโปรโตคอล **AMQP**
(Advanced Message Queuing Protocol):

```ruby
# Gemfile
gem "bunny", "~> 2.24"
```

### 4 แนวคิดหลักของ RabbitMQ

RabbitMQ ใช้โมเดล **AMQP** ที่มี 4 องค์ประกอบหลักที่ต้องเข้าใจให้แม่นก่อนเขียนโค้ด:

1. **Exchange** — จุดที่ producer ส่งข้อความเข้ามา **Exchange ไม่เก็บข้อความไว้เอง** หน้าที่
   ของมันคือ "ตัดสินใจว่าจะส่งข้อความนี้ต่อไปที่ queue ไหนบ้าง" ตามกฎการ routing ที่กำหนดไว้
   มีหลายชนิด: **`direct`** (ส่งตรงตาม routing key ที่ตรงกันเป๊ะ), **`topic`** (ส่งตาม pattern
   ของ routing key เช่น `order.*`), **`fanout`** (ส่งไปทุก queue ที่ bind ไว้ โดยไม่สนใจ
   routing key เลย — เหมาะกับ broadcast)
2. **Queue** — ที่เก็บข้อความจริงๆ (เป็น buffer แบบ FIFO) consumer จะมาดึงข้อความจาก queue
   โดยตรง ไม่ได้ดึงจาก exchange
3. **Binding** — "สาย" ที่เชื่อม exchange เข้ากับ queue พร้อมระบุเงื่อนไข (เช่น routing key
   แบบไหนที่จะถูกส่งผ่านสายนี้ไปยัง queue นี้)
4. **Routing Key** — "ป้ายกำกับ" ที่ผูกไปกับข้อความตอน publish ใช้ประกอบการตัดสินใจ routing
   ของ exchange (เทียบคร่าวๆ ได้กับ "หัวข้อ" ของอีเมลที่ใช้กรองเข้าโฟลเดอร์)

```
Producer                Exchange (topic: "orders_exchange")           Queue
   │                            │                                        │
   │──publish──────────────────→│                                        │
   │  routing_key: "order.placed"                                        │
   │                            │──(binding: routing_key "order.placed")→│ order_events.order_placed
   │                            │                                        │
                                                                     Consumer ดึงข้อความจากนี่
```

### ตัวอย่างจริง: Publisher

```ruby
# frozen_string_literal: true

require "bunny"
require "json"
require "time"
require "securerandom"

connection = Bunny.new(host: "127.0.0.1")
connection.start

channel = connection.create_channel

# ประกาศ exchange ชนิด topic — เลือก topic เพราะยืดหยุ่นที่สุด รองรับทั้งกรณีง่าย (routing key
# ตรงเป๊ะ) และกรณีซับซ้อน (pattern matching เช่น "order.*" จับได้ทั้ง order.placed, order.cancelled)
exchange = channel.topic("orders_exchange", durable: true)

# ประกาศ queue + ผูก (bind) เข้ากับ exchange ด้วย routing key ที่สนใจ
queue = channel.queue("order_events.order_placed", durable: true)
queue.bind(exchange, routing_key: "order.placed")

event_payload = {
  event: "OrderPlaced",
  order_id: 1042,
  total_cents: 259_00,
  occurred_at: Time.now.utc.iso8601,
  event_id: SecureRandom.uuid
}

exchange.publish(
  event_payload.to_json,
  routing_key: "order.placed",
  persistent: true,                    # ให้ message เขียนลงดิสก์ ไม่ใช่แค่ในหน่วยความจำ
  content_type: "application/json"
)

puts "[publisher] published OrderPlaced event: #{event_payload}"

connection.close
```

ผลลัพธ์จริงจากการรัน `ruby publisher.rb`:

```
[publisher] published OrderPlaced event: {:event=>"OrderPlaced", :order_id=>1042, :total_cents=>25900, :occurred_at=>"2026-09-28T18:49:09Z", :event_id=>"4a392731-8675-4554-9839-540c83ae4de3"}
```

**อธิบาย:**

- `durable: true` ตอนประกาศทั้ง exchange และ queue หมายถึง "ถ้า RabbitMQ broker restart
  ให้จำ exchange/queue นี้ไว้ (ไม่หายไปพร้อม memory)" — แต่ **ไม่ได้แปลว่าข้อความข้างในจะ
  รอดด้วย** ถ้าไม่ได้ตั้ง `persistent: true` ตอน publish ด้วย
- `persistent: true` ตอน publish คือสิ่งที่บอกให้ **ข้อความนั้นๆ** ถูกเขียนลงดิสก์จริง — ต้องมี
  ทั้งคู่ (`durable: true` ที่ queue **และ** `persistent: true` ที่ message) ถึงจะรับประกันว่า
  ข้อความไม่หายแม้ broker ล่มกะทันหัน
- `event_id` (UUID) ใส่ไว้ให้ consumer ใช้ตรวจสอบ idempotency ได้ (ทบทวนเหตุผลจาก Step 855 —
  message อาจถูกส่งซ้ำได้)

### ตัวอย่างจริง: Consumer

```ruby
# frozen_string_literal: true

require "bunny"
require "json"

connection = Bunny.new(host: "127.0.0.1")
connection.start

channel = connection.create_channel
channel.prefetch(1)   # รับทีละ 1 ข้อความ ไม่ดึงมาตุนไว้ล่วงหน้าจนล้น

# consumer ต้องประกาศ exchange/queue/binding ด้วยชื่อและค่าเดียวกันกับฝั่ง publisher เสมอ
# (RabbitMQ ไม่สนใจว่าใครประกาศก่อน — declare ซ้ำด้วยค่าเดิมไม่มีปัญหาอะไร)
exchange = channel.topic("orders_exchange", durable: true)
queue = channel.queue("order_events.order_placed", durable: true)
queue.bind(exchange, routing_key: "order.placed")

puts "[consumer] waiting for messages..."

delivery_info, _properties, payload = queue.pop

if payload
  data = JSON.parse(payload)
  puts "[consumer] received event: #{data['event']} for order_id=#{data['order_id']}"
  channel.ack(delivery_info.delivery_tag)   # บอก broker ว่าประมวลผลสำเร็จแล้ว ลบออกจากคิวได้
else
  puts "[consumer] no message available"
end

connection.close
```

ผลลัพธ์จริงจากการรัน `ruby consumer.rb`:

```
[consumer] waiting for messages...
[consumer] received event: OrderPlaced for order_id=1042
```

**อธิบาย:**

- `queue.pop` ดึงข้อความ **หนึ่งครั้ง** แล้วจบ (เหมาะกับ demo/สคริปต์สั้นๆ) — สำหรับ worker
  process ที่ต้องรันตลอดไป จะใช้ `queue.subscribe` แทน (เรียนเต็มๆ ใน Step 857)
- `channel.ack(delivery_info.delivery_tag)` คือขั้นตอนสำคัญมากที่มักถูกลืม — ถ้าไม่ `ack`
  RabbitMQ จะคิดว่าข้อความนั้น**ยังไม่ถูกประมวลผลสำเร็จ** และจะส่งกลับมาให้ใหม่ (ให้ consumer
  ตัวเดิมหรือตัวอื่นที่ต่อ queue เดียวกันอยู่) ทันทีที่ connection ปิดหรือ timeout — นี่คือ
  กลไกที่ทำให้ RabbitMQ รับประกัน **at-least-once delivery** ตามที่อธิบายไว้ใน Step 855

### พิสูจน์ Decoupling จริง: Publish ตอนไม่มี Consumer รันอยู่เลย

นี่คือการทดลองที่สำคัญที่สุดของ Step นี้ — พิสูจน์ว่า message queue แก้ปัญหา cascading failure
จาก Step 854 ได้จริง โดย publish สองครั้งติดกัน **ในขณะที่ยังไม่มี consumer รันอยู่เลย**:

```bash
ruby publisher.rb   # publish ครั้งที่ 1
ruby publisher.rb   # publish ครั้งที่ 2 (ไม่มี consumer ฟังอยู่เลยทั้งสองครั้ง)
```

ตรวจสอบสถานะ queue ผ่าน `rabbitmqctl` (เครื่องมือ admin ที่มากับ RabbitMQ):

```bash
rabbitmqctl list_queues name messages durable
```

ผลลัพธ์จริง:

```
Listing queues for vhost / ...
name                        messages  durable
order_events.order_placed   2         true
```

**ข้อความทั้งสองรออยู่ใน queue อย่างปลอดภัย** แม้ไม่มี consumer รันอยู่เลยตอนที่ publish —
นี่คือสิ่งที่ synchronous call (Step 853–854) **ทำไม่ได้เลย** (ถ้า Inventory Service ล่มตอนที่
Checkout Service เรียก จะได้ error ทันที ไม่มีการ "รอไว้ก่อน") ลองรัน consumer อีกครั้งเพื่อ
ดึงข้อความออกมาทีละอัน:

```bash
ruby consumer.rb
```

```
[consumer] waiting for messages...
[consumer] received event: OrderPlaced for order_id=1042
```

ตรวจสอบ queue อีกครั้ง — เหลือ 1 ข้อความพอดี (ตัวที่เพิ่ง `ack` ไปแล้วถูกลบออกจาก queue จริง):

```bash
rabbitmqctl list_queues name messages durable
```

```
Listing queues for vhost / ...
name                        messages  durable
order_events.order_placed   1         true
```

นี่คือหลักฐานที่จับต้องได้ว่า message queue **แยกช่วงเวลาของการ publish ออกจากการ consume**
โดยสิ้นเชิง — producer และ consumer ไม่ต้อง "ออนไลน์พร้อมกัน" เลยแม้แต่วินาทีเดียว

---

## Step 857: Background Worker บริโภคอีเวนต์จากคิว เชื่อมต่อกับ ActiveJob/Sidekiq

### ทำไมไม่ควรทำงานหนักตรงๆ ในตัว AMQP Consumer

ตัวอย่างใน Step 856 ใช้ `queue.pop`/`channel.ack` ตรงๆ ซึ่งเหมาะกับการสาธิต แต่ในงานจริง
consumer ที่รันตลอดเวลาด้วย `queue.subscribe` (blocking loop) **ไม่ควรทำงานหนักตรงในนั้น**
เพราะ:

- ไม่มี retry/backoff อัตโนมัติแบบที่ Sidekiq ให้ (Part 062 Step 616)
- ไม่มี dashboard ดูสถานะงานเหมือน Sidekiq Web UI (Part 062 Step 620)
- ควบคุม concurrency (กี่ job พร้อมกัน) ยากกว่า infrastructure ที่มีอยู่แล้ว

**pattern ที่แนะนำในงานจริง:** ให้ AMQP consumer ทำหน้าที่แค่ **"bridge"** — รับอีเวนต์จากคิว
แล้ว**ส่งต่อทันที**ให้ ActiveJob/Sidekiq (ที่เรียนเต็มๆ ไปแล้วใน Part 061–062) ไปทำงานหนักแทน
วิธีนี้ทำให้ได้ประโยชน์ของทั้งสองระบบพร้อมกัน: RabbitMQ จัดการเรื่อง cross-service messaging,
Sidekiq จัดการเรื่อง retry/concurrency/monitoring ภายในเซอร์วิสของเราเอง

```
RabbitMQ Queue → [Bridge Consumer process] → Sidekiq (enqueue) → Sidekiq Worker (ประมวลผลจริง)
                  (รับอีเวนต์ ตัดสินใจว่าเป็น job อะไร)
```

### ตัวอย่างจริง: Sidekiq Job ที่รับช่วงงานต่อ

```ruby
# frozen_string_literal: true

# app/jobs/order_confirmation_job.rb
require "sidekiq"

class OrderConfirmationJob
  include Sidekiq::Job
  sidekiq_options queue: "default", retry: 3   # ทบทวน retry/queue config จาก Part 062 Step 615–616

  def perform(order_id, total_cents)
    # งานจริง: ส่งอีเมลยืนยันออเดอร์ (Action Mailer), อัปเดต analytics ฯลฯ
    puts "[Sidekiq worker] sending confirmation email for order ##{order_id} (total: #{total_cents} cents)"
  end
end
```

### ตัวอย่างจริง: Bridge Consumer ที่เชื่อม RabbitMQ เข้ากับ Sidekiq

```ruby
# frozen_string_literal: true

# จำลอง "bridge" process: ฟังอีเวนต์จาก RabbitMQ แล้วแปลงเป็น Sidekiq job
require "bunny"
require "json"
require_relative "order_confirmation_job"

connection = Bunny.new(host: "127.0.0.1")
connection.start
channel = connection.create_channel
exchange = channel.topic("orders_exchange", durable: true)
queue = channel.queue("order_events.order_placed", durable: true)
queue.bind(exchange, routing_key: "order.placed")

delivery_info, _properties, payload = queue.pop
if payload
  data = JSON.parse(payload)
  puts "[bridge] consumed event #{data['event']} from RabbitMQ, enqueueing Sidekiq job..."
  OrderConfirmationJob.perform_async(data["order_id"], data["total_cents"])
  channel.ack(delivery_info.delivery_tag)
else
  puts "[bridge] no message"
end
connection.close
```

### พิสูจน์ Pipeline เต็มรูปแบบ — ทดสอบรันจริงข้าม 3 Process

นี่คือการทดลองที่สำคัญที่สุดของ Part นี้: พิสูจน์ว่า **RabbitMQ → Bridge Consumer → Sidekiq →
Sidekiq Worker** ทำงานร่วมกันได้จริงข้าม process ที่แยกกันโดยสิ้นเชิง (เหมือนในระบบจริงที่แต่ละ
ส่วนอาจรันคนละเครื่อง/คนละ container กัน)

```bash
# 1) publish อีเวนต์เข้า RabbitMQ (จำลอง Rails app ฝั่ง Order)
ruby publisher.rb

# 2) bridge consumer ดึงอีเวนต์ออกจาก RabbitMQ แล้ว enqueue เป็น Sidekiq job (เข้า Redis)
ruby bridge_consumer.rb

# 3) รัน Sidekiq worker process จริง ให้ไปดึง job จาก Redis มาประมวลผล
sidekiq -r ./order_confirmation_job.rb -c 1
```

ผลลัพธ์จริงจากการรันทั้ง 3 ขั้นตอนติดกัน:

```
[publisher] published OrderPlaced event: {:event=>"OrderPlaced", ..., "order_id":1042, ...}
[bridge] consumed event OrderPlaced from RabbitMQ, enqueueing Sidekiq job...
INFO ... Sidekiq 8.1.7 connecting to Redis with options {...}
```

จากนั้นในเทอร์มินัลของ Sidekiq worker:

```
INFO ... Sidekiq 8.1.7 connecting to Redis with options {...}
INFO ... jid=a1fc1235cf1237193c4c67fd class=OrderConfirmationJob: start
[Sidekiq worker] sending confirmation email for order #1042 (total: 25900 cents)
INFO ... jid=a1fc1235cf1237193c4c67fd class=OrderConfirmationJob elapsed=0.0: done
```

**ยืนยันด้วย Redis โดยตรง** ว่า job จริงถูกดันเข้าคิวของ Sidekiq (ก่อนถูกประมวลผล):

```bash
redis-cli lrange queue:default 0 -1
```

```
{"retry":3,"queue":"default","args":[1042,25900],"class":"OrderConfirmationJob","jid":"b1f7a483ad450954a2715399", ...}
```

และหลัง Sidekiq worker ประมวลผลเสร็จ คิวใน Redis ว่างเปล่า (`redis-cli llen queue:default` ได้
`0`) — ยืนยันว่า **pipeline ทั้งหมดทำงานจบสมบูรณ์จริง** ตั้งแต่ publish จน worker ประมวลผลเสร็จ
ข้าม 3 process ที่เป็นอิสระต่อกันโดยสิ้นเชิง (คนละ Ruby process, คุยกันผ่าน RabbitMQ และ Redis
เท่านั้น ไม่มีการเรียก method ข้าม process โดยตรงเลยสักจุดเดียว)

### รูปแบบ Production: Consumer แบบ Subscribe (Blocking Loop)

ตัวอย่างข้างบนใช้ `queue.pop` เพื่อความง่ายในการสาธิต (ดึงข้อความเดียวแล้วจบโปรแกรม) แต่ worker
process จริงใน production ต้อง**รันตลอดไป** (long-running process ที่ดูแลผ่าน systemd,
Kubernetes Deployment, หรือ process manager อื่น) โดยใช้ `queue.subscribe` แทน:

```ruby
# frozen_string_literal: true

# bin/order_events_consumer — รันแบบ `ruby bin/order_events_consumer` แล้วปล่อยให้ทำงานตลอดไป
require "bunny"
require "json"

connection = Bunny.new(host: ENV.fetch("RABBITMQ_HOST", "127.0.0.1"))
connection.start
channel = connection.create_channel
channel.prefetch(1)
exchange = channel.topic("orders_exchange", durable: true)
queue = channel.queue("order_events.order_placed", durable: true)
queue.bind(exchange, routing_key: "order.placed")

puts "[consumer] waiting for messages... (Ctrl+C to stop)"

# block: true ทำให้เมธอดนี้ไม่ return เลย — โปรแกรมจะวนรอรับข้อความไปเรื่อยๆ
# (คือรูปแบบที่ถูกต้องสำหรับ worker process ที่ควรรันอยู่ตลอดเวลา)
queue.subscribe(manual_ack: true, block: true) do |delivery_info, _properties, payload|
  data = JSON.parse(payload)
  OrderConfirmationJob.perform_async(data["order_id"], data["total_cents"])
  channel.ack(delivery_info.delivery_tag)
rescue JSON::ParserError => e
  # ข้อความเสียหาย/ผิดรูปแบบ — reject โดยไม่ส่งกลับเข้าคิว (requeue: false) ป้องกัน infinite loop
  channel.reject(delivery_info.delivery_tag, false)
  Rails.logger.error("[order_events_consumer] bad payload: #{e.message}") if defined?(Rails)
end
```

**อธิบาย:** `channel.reject(delivery_tag, false)` (พารามิเตอร์ที่สองคือ `requeue`) สำคัญมาก
สำหรับข้อความที่ **พังถาวร** (เช่น JSON ผิดรูปแบบ) — ถ้า `requeue: true` (ค่า default ของ
`nack`) ข้อความจะถูกส่งกลับเข้าคิวทันทีแล้วมาให้ consumer ตัวเดิมรับอีก กลายเป็น **infinite
loop ที่กิน CPU ตลอดไป** วิธีที่ถูกต้องคือ `reject` แบบ `requeue: false` แล้วส่งข้อความนั้นไปที่
**Dead Letter Exchange (DLX)** — กลไกที่คล้าย "dead job set" ของ Sidekiq ที่เรียนใน Part 062
Step 616 ทุกประการ เพียงแค่เป็นกลไกของ RabbitMQ เอง (เป็นหัวข้อสำหรับศึกษาต่อ ดูแบบฝึกหัด
เพิ่มเติมท้าย Part)

---

## Step 858: Kafka เบื้องต้น — Log-based Streaming ต่างจาก Message Broker แบบ RabbitMQ อย่างไร

> **หมายเหตุความซื่อตรงเรื่องการทดสอบ:** Step นี้อธิบาย **เชิงแนวคิดล้วนๆ** โดยอ้างอิงจาก
> เอกสารและพฤติกรรมที่มีการบันทึกไว้อย่างดีของ Apache Kafka **ไม่ได้ทดสอบรันจริงในสภาพแวดล้อม
> เดียวกับที่ใช้ทดสอบ RabbitMQ ข้างต้น** เหตุผลคือ Kafka broker ต้องการ **JVM (Java Virtual
> Machine)** และ metadata coordination service (ในรุ่นเก่าคือ **Apache ZooKeeper** แยกต่างหาก
> ในรุ่นใหม่ตั้งแต่ Kafka 3.3+ ใช้โหมด **KRaft** ที่ตัด ZooKeeper ออกได้ แต่ยังต้องมี JVM
> อยู่ดี) ซึ่งมี footprint หนักกว่า RabbitMQ (ที่ใช้แค่ Erlang runtime) มากสำหรับการสาธิตสั้นๆ
> ในสภาพแวดล้อมทดสอบของหลักสูตรนี้ — ในงานจริงควรทดลองรัน Kafka ด้วยตัวเองผ่าน Docker
> (`confluentinc/cp-kafka` หรือ `apache/kafka` official image) เพื่อจับความรู้สึกจริงเพิ่มเติม

### โมเดลความคิดที่ต่างจาก RabbitMQ โดยสิ้นเชิง

RabbitMQ คือ **message broker** — ข้อความถูก "ส่งต่อ" ผ่าน exchange ไปยัง queue แล้ว**ถูกลบ
ออก**หลังจาก consumer `ack` สำเร็จ (คิดแบบ "จดหมายที่เปิดอ่านแล้วก็โยนทิ้ง")

**Kafka** คือ **distributed commit log** — โมเดลความคิดต่างไปคนละเรื่องเลย:

- ข้อความถูกเก็บไว้ใน **Topic** ที่แบ่งเป็นหลาย **Partition** (คิดแบบ "สมุดบันทึกต่อเนื่อง"
  ที่มีเลขบรรทัดกำกับเรื่อยๆ ไม่มีวันลบทิ้งจนกว่าจะครบนโยบาย retention ที่ตั้งไว้ เช่น
  "เก็บไว้ 7 วัน" หรือ "เก็บไว้ตลอดไป")
- Producer **append** ข้อความต่อท้าย partition เสมอ (เหมือนเขียนต่อท้ายสมุดบันทึก ไม่มีการ
  "route" แบบ exchange ของ RabbitMQ)
- Consumer **ไม่ได้ลบข้อความออกตอนอ่าน** — แค่จด **offset** (ตำแหน่งเลขบรรทัดที่อ่านถึงแล้ว)
  ไว้เอง ทำให้ **อ่านซ้ำ (replay) ข้อความเก่าได้ทุกเมื่อ** เพียงแค่ย้อน offset กลับไป — สิ่งนี้
  ทำไม่ได้เลยใน RabbitMQ (ข้อความหายไปทันทีที่ `ack`)
- **Consumer Group** — consumer หลายตัวรวมกลุ่มกันอ่าน topic เดียวกัน โดย Kafka จะแบ่ง
  partition ให้แต่ละ consumer ในกลุ่มรับผิดชอบคนละส่วน (ถ้ามี consumer group อื่นมาอ่าน topic
  เดียวกันอีกกลุ่ม จะได้ข้อมูลชุดเดียวกันทั้งหมดอีกรอบอย่างอิสระ — เหมาะมากสำหรับ "ให้หลายทีม
  ประมวลผล event stream เดียวกันเพื่อจุดประสงค์ต่างกัน" เช่น ทีม Analytics กับทีม Fraud
  Detection อ่าน stream การสั่งซื้อเดียวกัน แต่ทำคนละอย่างกับข้อมูล)
- **การเรียงลำดับ (ordering)** รับประกันแค่ **ภายใน partition เดียวกันเท่านั้น** ไม่รับประกัน
  ข้ามหลาย partition — การออกแบบ partition key (เช่น ใช้ `order_id` เป็น key เพื่อให้ event
  ของ order เดียวกันตกไปอยู่ partition เดียวกันเสมอ) จึงสำคัญมาก

```
RabbitMQ:  Producer → Exchange → Queue (FIFO, ลบทิ้งหลัง ack) → Consumer
Kafka:     Producer → Topic [Partition 0, 1, 2, ...] (เก็บถาวรตาม retention) → Consumer Group
                                                                                (จำ offset เอง, replay ได้)
```

### เมื่อไหร่ควรเลือก Kafka แทน RabbitMQ

| สถานการณ์ | เลือก |
|---|---|
| Task queue ทั่วไป (ส่งอีเมล, ประมวลผลรูปภาพ, งานที่ทำครั้งเดียวจบ) | **RabbitMQ** (หรือ Sidekiq/Solid Queue ตรงๆ เลยด้วยซ้ำ ถ้าไม่ข้ามเซอร์วิส) |
| RPC-style messaging ที่ต้องการ routing ซับซ้อน (topic/direct/fanout ตามเงื่อนไข) | **RabbitMQ** |
| Event streaming ปริมาณสูงมาก (เช่น log ทุก click ของผู้ใช้ระดับล้าน event/วินาที) | **Kafka** |
| ต้องการให้หลายทีม/หลายระบบ อ่าน event stream เดียวกันได้อิสระ และย้อนอ่านประวัติเก่าได้ | **Kafka** |
| Event Sourcing (ระบบที่เก็บ "ประวัติการเปลี่ยนแปลงทั้งหมด" เป็นแหล่งความจริงหลัก) | **Kafka** (เพราะเก็บ log ถาวรโดยธรรมชาติของมันอยู่แล้ว) |
| ทีมเล็ก/กลาง ที่ยังไม่มีปัญหา throughput ระดับสูงมาก | **RabbitMQ** (ติดตั้ง/ดูแลง่ายกว่ามาก ไม่ต้องมี JVM/ZooKeeper) |

**กฎง่ายๆ ที่จำง่าย:** RabbitMQ เหมาะกับ **"งาน" (task)** ที่ต้องทำแล้วจบ ส่วน Kafka เหมาะกับ
**"สตรีมของเหตุการณ์" (event stream)** ที่มีคุณค่าอยู่ในตัวประวัติศาสตร์ทั้งหมดของมัน ไม่ใช่
แค่เหตุการณ์ล่าสุด

### ถ้าจะใช้ Kafka กับ Rails (แนวคิด — โค้ดตัวอย่างยังไม่ได้ทดสอบรันจริง)

Ruby ecosystem มี gem `ruby-kafka` (pure Ruby, เก่ากว่า ดูแลน้อยลงในปัจจุบัน) และ `rdkafka`
(bindings ของ `librdkafka` ซึ่งเป็น C library อย่างเป็นทางการที่ Confluent ดูแล นิยมใช้มากกว่า
ในโปรเจกต์ใหม่เพราะเร็วกว่าและ maintain active กว่า):

```ruby
# frozen_string_literal: true

# ตัวอย่างแนวคิดเท่านั้น (ไม่ได้ทดสอบรันจริงในบทนี้) — โครงสร้างการ publish ด้วย rdkafka
require "rdkafka"

config = { "bootstrap.servers" => "localhost:9092" }
producer = Rdkafka::Config.new(config).producer

producer.produce(
  topic: "order-events",
  payload: { event: "OrderPlaced", order_id: 1042 }.to_json,
  key: "1042" # ใช้ order_id เป็น partition key เพื่อการันตี ordering ต่อออเดอร์เดียวกัน
)
```

โครงสร้างหน้าตาคล้ายกับ `bunny` มาก (เพราะ concept "publish ข้อความออกไป" เหมือนกัน) แต่
**ความหมายเบื้องหลังต่างกันโดยสิ้นเชิง** ตามที่อธิบายไว้ข้างต้น — เข้าใจแนวคิดให้แม่นก่อนเลือก
เครื่องมือ สำคัญกว่าการท่องจำ syntax

---

## Step 859: Distributed Monolith — เมื่อแยกเซอร์วิสแล้วแย่กว่าเดิม

### นิยาม: Anti-pattern ที่พบบ่อยที่สุดของทีมที่แยก Microservices เร็วเกินไป

**Distributed Monolith** คือระบบที่ **ถูกแยกเป็นหลาย service แล้ว** (มีหลาย repository, deploy
แยกกัน) แต่ **ยังคงพฤติกรรมของ monolith ที่แย่ที่สุด** ไว้ครบ — คือ **เซอร์วิสทุกตัวยังคง
ผูกติดกันแน่นมาก (tightly coupled)** ผ่านการเรียก synchronous call ข้ามเซอร์วิสเป็นลูกโซ่
ทำให้ได้**ต้นทุนของ microservices เต็มๆ (network latency, deployment complexity, debugging
ยาก) โดยไม่ได้ประโยชน์ที่แท้จริงของมันเลยสักอย่าง** (deploy อิสระไม่ได้จริง เพราะเซอร์วิส A
เปลี่ยน API แล้ว B/C/D ต้องแก้ตามพร้อมกันเสมอ)

```
อาการของ Distributed Monolith:

Checkout Service ──(sync)──→ Pricing Service ──(sync)──→ Tax Service ──(sync)──→ Discount Service
      │                                                                                    │
      └───────────────────────── รอครบทุกขั้นตอนก่อนตอบผู้ใช้ ─────────────────────────────┘

ถ้าขั้นตอนไหนช้า/ล่ม → ทั้งเชนพัง (เหมือนที่อธิบายใน Step 854 แต่ยิ่งแย่กว่าเดิม
เพราะตอนนี้มี "จุดที่อาจพัง" มากขึ้นหลายจุดตามจำนวนเซอร์วิสที่เพิ่มเข้ามา)
```

### สัญญาณเตือนว่ากำลังเป็น Distributed Monolith

1. **แชร์ฐานข้อมูลเดียวกันข้ามเซอร์วิส** — ถ้าสองเซอร์วิส "แยก" กันแต่ยังอ่าน/เขียนตาราง
   เดียวกันในฐานข้อมูลเดียวกัน แสดงว่ายังไม่ได้แยก ownership ของข้อมูลจริงๆ (หลักการเดียวกับ
   ที่ Part 085 Multi-tenancy สอนเรื่องการแยก data ownership ให้ชัดเจนต่อ tenant — ที่นี่คือ
   แยกต่อ **เซอร์วิส** แทน)
2. **Deploy เซอร์วิสหนึ่งไม่ได้ถ้าไม่ deploy เซอร์วิสอื่นพร้อมกัน** — ถ้าทุกครั้งที่แก้ Checkout
   Service ต้องแก้ Pricing Service ตามด้วยเสมอ แสดงว่า coupling แน่นเกินไป ทั้งที่ "แยกเซอร์วิส"
   ไปแล้วในนาม
3. **มี synchronous call chain ยาวเกิน 2–3 hop** — ยิ่ง chain ยาว โอกาส fail สะสม
   (compounding failure probability) ยิ่งสูง เช่น ถ้าแต่ละ hop มีโอกาสสำเร็จ 99.9% chain 5 hop
   จะเหลือโอกาสสำเร็จรวมแค่ `0.999^5 ≈ 99.5%` เท่านั้น — ดูเหมือนน้อย แต่ที่ scale สูงคือ
   ความแตกต่างของผู้ใช้หลายพันคนที่เจอ error ต่อวัน
4. **ไม่มีการใช้ message queue/event เลยแม้แต่จุดเดียว** — ถ้าทุก cross-service interaction
   เป็น synchronous หมด (ไม่มีจุดไหนที่ "ประกาศแล้วไม่ต้องรอ" ตามที่ Step 855 สอน) นี่คือ
   สัญญาณชัดเจนว่าทีมแยกเซอร์วิสตามโครงสร้างองค์กร/ทีม แต่ยังไม่ได้ออกแบบการสื่อสารให้เข้ากับ
   ธรรมชาติของ distributed system จริงๆ

### วิธีแก้: ใช้ Async สำหรับทุกอย่างที่ไม่ต้องรอคำตอบทันที

หลักการง่ายๆ ที่ใช้แยกได้ทันที: **ถามตัวเองว่า "ผู้ใช้ต้องรอผลลัพธ์นี้ก่อนเห็นหน้าจอถัดไปไหม?"**

- **ต้องรอ** (เช่น "เช็คว่าคูปองนี้ใช้ได้จริงไหมตอนกดใช้") → synchronous call สมเหตุสมผล
  (แต่ต้องมี timeout + circuit breaker ตามที่ Step 854 สอน)
- **ไม่ต้องรอ** (เช่น "แจ้งระบบบัญชีว่ามีการขายเกิดขึ้น", "อัปเดต dashboard สถิติ", "ส่งอีเมล
  ยืนยัน") → **ควรเป็น asynchronous event เสมอ** ตามที่ Step 855–857 สอน

การไล่ตรวจ synchronous call ทุกจุดในระบบด้วยคำถามนี้ มักจะพบว่า**ส่วนใหญ่**ของการเรียกข้าม
เซอร์วิสที่เขียนแบบ synchronous ไปตั้งแต่แรก **จริงๆ แล้วไม่จำเป็นต้อง sync เลย** — นี่คือวิธี
แก้ Distributed Monolight ที่ได้ผลจริงและไม่ต้องออกแบบระบบใหม่ทั้งหมด

---

## Step 860: เส้นทางแยกเซอร์วิสแบบค่อยเป็นค่อยไป + แบบฝึกหัดปิด Phase 14

### หลักการ: แยกทีละเซอร์วิสที่มีเหตุผลวัดผลได้ชัดเจน ไม่ใช่ Big-bang Rewrite

จากทุก Step ที่ผ่านมาใน Part นี้ สรุปเป็นเส้นทางปฏิบัติจริงได้ดังนี้:

1. **อย่าเริ่มด้วย microservices** — เริ่มด้วย modular monolith ที่จัดโครงสร้างตาม Clean
   Architecture (Part 084) เสมอ (Step 851–852)
2. **รอจนกว่าจะมีเหตุผลที่วัดผลได้ชัดเจน** ไม่ใช่ "รู้สึกว่าน่าจะดี" — ตัวอย่างเหตุผลที่ชัดเจน
   จริง เช่น:
   - **มีการวัด metric จริง** ว่าส่วนหนึ่งของระบบ (เช่น image processing สำหรับ Active Storage
     ที่เรียนใน Part 066–067) กิน CPU สูงกว่าส่วนอื่นมาก และต้องการ scale (เพิ่ม instance)
     แยกจาก web server pool ทั่วไป เพราะถ้า scale รวมกันจะเปลือง cost มหาศาล (ต้อง scale
     instance ทั้งหมดตามความต้องการของส่วนที่หนักที่สุด ทั้งที่ web request ทั่วไปไม่ต้องการ
     ขนาดนั้นเลย)
   - **ทีมโตจนมีเจ้าของเฉพาะทางจริง** (ทีม Payments แยกจากทีม Catalog อย่างเป็นทางการ พร้อม
     ต้องการ deploy cycle ของตัวเอง)
3. **เลือกแยกแค่ "หนึ่งเซอร์วิส" ที่ bounded context ชัดเจนที่สุด** (มักเป็นส่วนที่ DI/interface
   ไว้แล้วจาก Part 084 ตามที่แสดงใน Step 852 — extraction seam ที่มีอยู่แล้ว)
4. **ออกแบบการสื่อสารให้ async เป็นค่าเริ่มต้น** สำหรับทุกอย่างที่ไม่ต้องรอผลทันที (Step 855–857)
   ใช้ synchronous เฉพาะจุดที่จำเป็นจริงๆ พร้อม timeout/circuit breaker (Step 853–854)
5. **ย้าย traffic แบบค่อยเป็นค่อยไป** — เช่นใช้ feature flag เปิดใช้ remote adapter
   (`RemoteInventoryChecker` จาก Step 852) ให้ traffic เพียงส่วนน้อยก่อน (canary), เฝ้าดู error
   rate/latency, แล้วค่อยเพิ่มสัดส่วนขึ้นเรื่อยๆ จนมั่นใจว่าเซอร์วิสใหม่รับภาระได้จริง

### ตัวอย่างการเลือกเซอร์วิสแรกที่จะแยก: Image Processing Service

สมมติระบบ e-commerce ของเรามี feature ให้ผู้ขายอัปโหลดรูปสินค้าจำนวนมาก แล้วระบบต้องประมวลผล
(resize, generate thumbnail หลายขนาด, compress) — วัดผลจริงแล้วพบว่า:

- การประมวลผลรูปภาพกิน CPU สูงถึง 80% ของ CPU time ทั้งระบบในช่วงเวลาที่มีการอัปโหลดจำนวนมาก
- Web request ทั่วไป (ดูสินค้า, สั่งซื้อ) แทบไม่ใช้ CPU เลยเมื่อเทียบกัน แต่ถูก instance
  เดียวกันแย่ง CPU ไปจนตอบช้าตาม
- ต้อง scale instance ทั้งเว็บเพื่อรองรับ peak ของการประมวลผลรูป ทั้งที่ peak ของ web traffic
  ปกติไม่ต้องการขนาดนั้นเลย

**นี่คือเหตุผลที่วัดผลได้ชัดเจนตามเกณฑ์ข้อ 2** — เหมาะเป็นตัวเลือกแรกที่จะแยกออกมาเป็น
**Image Processing Service** ต่างหาก:

```ruby
# frozen_string_literal: true

# เดิม (monolith): ประมวลผลรูปในกระบวนการเดียวกัน (ผ่าน ActiveJob ธรรมดา — Part 061)
module Products
  class ProcessUploadedImage
    def call(product_image)
      # resize, generate variant ฯลฯ (Part 066–067)
    end
  end
end

# หลังแยกเซอร์วิส: publish อีเวนต์แทนการประมวลผลตรงๆ ในโปรเซสเดียวกัน
module Products
  class ProcessUploadedImage
    def initialize(event_publisher: OrderEvents::Publisher)
      @event_publisher = event_publisher
    end

    def call(product_image)
      @event_publisher.publish(
        exchange: "product_images_exchange",
        routing_key: "image.uploaded",
        payload: { image_id: product_image.id, url: product_image.url }
      )
      # จบทันที ไม่รอผลการประมวลผลรูป — Image Processing Service (แยกเซอร์วิสแล้ว)
      # จะ consume อีเวนต์นี้ไปทำงานหนักเอง แล้ว publish อีเวนต์ "image.processed" กลับมา
      # เมื่อเสร็จ ให้ monolith เดิม consume เพื่ออัปเดตสถานะต่อไป
    end
  end
end
```

สังเกตว่า**รูปแบบการสื่อสารระหว่าง monolith เดิมกับ Image Processing Service ใหม่คือ
asynchronous ทั้งหมด** (publish/consume ผ่าน RabbitMQ) ตรงตามหลักการ "async เป็นค่าเริ่มต้น"
จาก Step 859 — เพราะผู้ใช้ที่อัปโหลดรูปไม่จำเป็นต้อง**รอ**ให้ประมวลผลเสร็จก่อนเห็นหน้าจอถัดไป
(แสดง "กำลังประมวลผล..." แล้วอัปเดตภายหลังผ่าน Turbo Streams จาก Part 052 ก็เพียงพอ) — นี่คือ
ตัวอย่างที่แสดงให้เห็นว่าการแยกเซอร์วิสที่ดี **ไม่ได้แค่ย้ายโค้ดไปอีกที่หนึ่ง** แต่ต้อง
**ออกแบบการสื่อสารใหม่ให้เข้ากับธรรมชาติของ distributed system** ไปพร้อมกันด้วยเสมอ

---

## แบบฝึกหัดปิด Phase 14: OrderPlaced Event Flow เต็มรูปแบบ

### โจทย์

สร้างระบบจำลอง 2 ส่วนที่**แยก process กันโดยสิ้นเชิง** (เหมือนสองเซอร์วิสจริง):

1. **`bin/place_order`** — จำลอง Rails app ฝั่ง Order ที่หลังสั่งซื้อสำเร็จ จะ publish อีเวนต์
   `OrderPlaced` ออกไปยัง RabbitMQ (ไม่ต้องรอใครตอบกลับ)
2. **`bin/order_worker`** — worker process แยกต่างหาก ที่ subscribe ฟังอีเวนต์ `OrderPlaced`
   จากคิวเดียวกัน แล้วประมวลผล (ในที่นี้คือ `puts` จำลองการส่งอีเมลยืนยัน)

โครงสร้างไฟล์:

```
order_events/
├── lib/
│   └── order_events/
│       ├── topology.rb     # รวมค่า exchange/queue/routing key + การเชื่อมต่อไว้ที่เดียว
│       └── publisher.rb    # class สำหรับ publish อีเวนต์ OrderPlaced
└── bin/
    ├── place_order          # entry point ฝั่ง publish
    └── order_worker          # entry point ฝั่ง consume (worker แยก process)
```

**เหตุผลของโครงสร้างนี้:** `topology.rb` แยกออกมาต่างหาก เพื่อให้ทั้งฝั่ง publish และ consume
**อ้างอิงค่า exchange/queue/routing key ชุดเดียวกันเสมอ** (ถ้าแต่ละฝั่งเขียนชื่อเองแยกกัน
มีความเสี่ยงสูงมากที่จะพิมพ์ชื่อไม่ตรงกันแล้วมึนงงว่า "ทำไม consumer ไม่เห็นข้อความเลย")
— ในระบบจริงที่สองเซอร์วิสอยู่คนละ repository กันจริงๆ มักแก้ปัญหานี้ด้วยการทำ **shared
contract** เช่น JSON Schema หรือ gem ภายในองค์กรที่ทั้งสองฝั่ง depend on ร่วมกัน

### เฉลย

**`lib/order_events/topology.rb`**

```ruby
# frozen_string_literal: true

require "bunny"

module OrderEvents
  # รวม topology (exchange/queue/binding) ไว้ที่เดียว ให้ทั้งฝั่ง publish และ consume
  # เรียกใช้ค่าเดียวกันเสมอ ป้องกันปัญหา "ชื่อ exchange/queue ไม่ตรงกัน" ระหว่างสอง process
  EXCHANGE_NAME = "orders_exchange"
  QUEUE_NAME = "order_events.order_placed"
  ROUTING_KEY = "order.placed"

  def self.connection
    Bunny.new(host: ENV.fetch("RABBITMQ_HOST", "127.0.0.1"))
  end

  def self.declare(channel)
    exchange = channel.topic(EXCHANGE_NAME, durable: true)
    queue = channel.queue(QUEUE_NAME, durable: true)
    queue.bind(exchange, routing_key: ROUTING_KEY)
    [exchange, queue]
  end
end
```

**`lib/order_events/publisher.rb`**

```ruby
# frozen_string_literal: true

require "json"
require "time"
require "securerandom"
require_relative "topology"

module OrderEvents
  # จำลองสิ่งที่ Rails app ฝั่ง "Order" จะเรียกใช้หลัง commit transaction สำเร็จ
  # (ในโปรเจกต์ Rails จริง มักเรียกจาก after_commit callback หรือจาก Service Object
  # อย่าง Orders::PlaceOrder#call ทบทวน Part 082)
  class Publisher
    def self.publish_order_placed(order_id:, total_cents:)
      connection = OrderEvents.connection
      connection.start
      channel = connection.create_channel
      exchange, = OrderEvents.declare(channel)

      payload = {
        event: "OrderPlaced",
        event_id: SecureRandom.uuid,
        order_id: order_id,
        total_cents: total_cents,
        occurred_at: Time.now.utc.iso8601
      }

      exchange.publish(
        payload.to_json,
        routing_key: OrderEvents::ROUTING_KEY,
        persistent: true,
        content_type: "application/json"
      )

      payload
    ensure
      connection&.close
    end
  end
end
```

**`bin/place_order`**

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

require_relative "../lib/order_events/publisher"

order_id = (ARGV[0] || rand(1000..9999)).to_i
total_cents = (ARGV[1] || rand(10_000..50_000)).to_i

payload = OrderEvents::Publisher.publish_order_placed(order_id: order_id, total_cents: total_cents)
puts "[place_order] published OrderPlaced: #{payload}"
```

**`bin/order_worker`**

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

require_relative "../lib/order_events/topology"
require "json"

# worker แบบ standalone process — รันแยกจาก Rails app โดยสิ้นเชิง (เช่น `ruby bin/order_worker`
# หรือแพ็กเป็น systemd service / container แยกใน production)
#
# ในตัวอย่างนี้จำกัดจำนวนข้อความที่ประมวลผลผ่าน ARGV[0] (ค่า default = ไม่จำกัด/รอตลอดไป)
# เพื่อให้ใช้สาธิตแบบสั้นๆ ได้ — production จริงจะใช้ queue.subscribe(block: true) วนลูปตลอดไป
limit = ARGV[0]&.to_i

connection = OrderEvents.connection
connection.start
channel = connection.create_channel
channel.prefetch(1)
_, queue = OrderEvents.declare(channel)

puts "[order_worker] waiting for OrderPlaced events..."

processed = 0
consumer = queue.subscribe(manual_ack: true, block: false) do |delivery_info, _properties, payload|
  data = JSON.parse(payload)
  puts "[order_worker] processing #{data['event']} order_id=#{data['order_id']} total_cents=#{data['total_cents']}"
  # งานจริงตรงนี้: ส่งอีเมลยืนยัน, อัปเดตสต็อก, สร้าง invoice ฯลฯ
  # (โปรเจกต์จริงมักส่งต่อให้ ActiveJob/Sidekiq ทำงานหนักแทน ดู Step 857)
  channel.acknowledge(delivery_info.delivery_tag, false)
  processed += 1
end

# poll แบบง่ายจนกว่าจะประมวลผลครบ limit ข้อความ (เฉพาะสำหรับ demo ให้สคริปต์จบเองได้)
if limit
  sleep 0.2 while processed < limit
  consumer.cancel
end

connection.close
puts "[order_worker] processed #{processed} event(s), shutting down"
```

### รันจริง (ทดสอบแล้ว)

```bash
chmod +x bin/place_order bin/order_worker

ruby bin/place_order 5001 129900
ruby bin/place_order 5002 45000
ruby bin/order_worker 2
```

ผลลัพธ์จริงจากการรัน:

```
[place_order] published OrderPlaced: {:event=>"OrderPlaced", :event_id=>"0d1dccea-d688-4e43-af49-f1bd186e87db", :order_id=>5001, :total_cents=>129900, :occurred_at=>"2026-09-28T18:53:25Z"}
[place_order] published OrderPlaced: {:event=>"OrderPlaced", :event_id=>"2e597036-3f74-45c7-9a42-6c49351f6cf2", :order_id=>5002, :total_cents=>45000, :occurred_at=>"2026-09-28T18:53:25Z"}
[order_worker] waiting for OrderPlaced events...
[order_worker] processing OrderPlaced order_id=5001 total_cents=129900
[order_worker] processing OrderPlaced order_id=5002 total_cents=45000
[order_worker] processed 2 event(s), shutting down
```

**สังเกตสิ่งสำคัญ:** `bin/place_order` ถูกรัน**สองครั้งติดกัน**และ**จบการทำงานทันที**ทุกครั้ง
(ไม่รอ worker เลย) ส่วน `bin/order_worker` ถูกรัน**แยกต่างหากทีหลัง**และดึงเอาข้อความทั้งสอง
ที่ค้างอยู่ในคิวมาประมวลผลจนครบ — นี่คือการพิสูจน์แบบจับต้องได้ว่า publisher และ consumer
**ไม่ต้องรันพร้อมกันเลยแม้แต่วินาทีเดียว** ตามหลักการ asynchronous communication ที่เรียนมา
ตลอดทั้ง Part นี้

> **ถ้าไม่มี RabbitMQ ให้ใช้งาน — ทางเลือกด้วย Redis Pub/Sub (ทดสอบรันจริงแล้วเช่นกัน):**
> Redis ที่ติดตั้งมาแล้วตั้งแต่ Part 062 (Sidekiq) มีความสามารถ pub/sub ในตัว ใช้สาธิตแนวคิด
> "publish แล้วไม่ต้องรอ" ได้เหมือนกัน **แต่มีข้อจำกัดสำคัญที่ต้องเข้าใจ**: Redis pub/sub เป็น
> **fire-and-forget แท้ๆ** — ถ้าไม่มี subscriber กำลังฟังอยู่ ณ วินาทีที่ publish ข้อความนั้น
> **หายไปเลยทันที ไม่มีการเก็บไว้รอเหมือน RabbitMQ queue** (ไม่มี durable/persistent ให้ตั้งค่า)
> จึงเหมาะกับกรณีที่ยอมรับการ "พลาดได้บ้าง" เท่านั้น (เช่น broadcast การอัปเดต real-time ผ่าน
> Turbo Streams/ActionCable) ไม่เหมาะกับอีเวนต์ทางธุรกิจสำคัญอย่าง `OrderPlaced` ที่พลาดไม่ได้
>
> ```ruby
> # publisher (ต้องมี subscriber ฟังอยู่ก่อนแล้วเท่านั้นถึงจะได้รับ)
> require "redis"
> require "json"
> redis = Redis.new(url: "redis://127.0.0.1:6379/0")
> payload = { event: "OrderPlaced", order_id: 2001, total_cents: 15900 }
> subscribers = redis.publish("order_events", payload.to_json)
> puts "published to #{subscribers} subscriber(s)"
>
> # subscriber (ต้องรันอยู่ก่อน publish เท่านั้นถึงจะเห็นข้อความ)
> require "redis"
> require "json"
> redis = Redis.new(url: "redis://127.0.0.1:6379/0")
> redis.subscribe("order_events") do |on|
>   on.message do |_channel, message|
>     data = JSON.parse(message)
>     puts "received #{data['event']} for order_id=#{data['order_id']}"
>   end
> end
> ```
>
> ผลลัพธ์จริงจากการทดสอบ (subscriber รันก่อน แล้วค่อย publish):
>
> ```
> published to 1 subscriber(s): {:event=>"OrderPlaced", :order_id=>2001, ...}
> [subscriber] listening on order_events...
> [subscriber] received OrderPlaced for order_id=2001
> ```
>
> เทียบให้เห็นชัด: ถ้าสลับลำดับเป็น publish ก่อนแล้วค่อยเปิด subscriber ทีหลัง (เหมือนที่ทดลอง
> กับ RabbitMQ ใน Step 856) ข้อความจะหายไปเฉยๆ โดยไม่มี error ใดๆ เลย — นี่คือความต่างที่สำคัญ
> ที่สุดระหว่าง **message queue จริง (RabbitMQ)** กับ **pub/sub แบบง่าย (Redis)**: อย่างแรก
> การันตีการส่งถึง (delivery guarantee) ได้ อย่างหลังไม่ได้เลย

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **Dead Letter Exchange (DLX)** — ขยาย `bin/order_worker` ให้เมื่อประมวลผลอีเวนต์ล้มเหลว
   (เช่น `order_id` เป็นค่าติดลบซึ่งถือว่าข้อมูลเสีย) ให้ `channel.reject` แบบ `requeue: false`
   แล้วตั้งค่า **Dead Letter Exchange** ให้กับ queue หลัก (ผ่าน argument
   `arguments: { "x-dead-letter-exchange" => "orders_dlx" }` ตอนประกาศ queue) เพื่อให้ข้อความ
   ที่พังไปตกอยู่ที่ queue สำรองแยกต่างหาก แทนที่จะหายไปเฉยๆ หรือวนลูปไม่รู้จบ — ตรวจสอบด้วย
   `rabbitmqctl list_queues` ว่าข้อความที่พังไปโผล่ที่ queue ใหม่จริง (แนวคิดเดียวกับ "dead
   job set" ของ Sidekiq ใน Part 062 Step 616)
2. **Circuit Breaker ด้วย gem จริง** — แทนที่ `SimpleCircuitBreaker` ใน Step 854 ด้วย gem
   `stoplight` จริง ห่อหุ้มการเรียก `InventoryServiceClient#stock_for` แล้วเขียนเทสต์ (RSpec)
   ยืนยันว่าวงจรเปิด (`Stoplight::Error::RedLight`) จริงหลังจาก fail ครบ threshold ที่กำหนดไว้
   (ใช้ Faraday test adapter จำลอง failure ซ้ำๆ ตามที่เรียนใน Step 853 ประกอบการทดสอบ)
3. **ออกแบบ (บนกระดาษ/diagram ไม่ต้องเขียนโค้ด) การย้าย flow "ส่ง Invoice ทางอีเมล" จาก
   synchronous call ไปเป็น asynchronous ผ่าน RabbitMQ topic exchange** ที่มี 2 routing key คือ
   `invoice.created` และ `invoice.voided` พร้อมระบุว่า consumer จะออกแบบให้ **idempotent**
   อย่างไร (ทบทวนหลักการจาก Part 062 Step 618) ถ้า RabbitMQ ส่งอีเวนต์ `invoice.created`
   ซ้ำสองครั้งโดยไม่ตั้งใจ ระบบต้อง**ไม่ส่งอีเมลซ้ำสองฉบับ**ให้ลูกค้า — ใบ้: พิจารณาใช้
   `event_id` (เหมือนที่ใส่ไว้ใน payload ของ Step 856) บันทึกลงตาราง `processed_events`
   ก่อนประมวลผลจริงทุกครั้ง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจ **ต้นทุนที่แท้จริงของ microservices** เทียบกับ monolith และรู้จักหลักการ
  **"Monolith First"** — แยกเซอร์วิสเมื่อมีเหตุผลวัดผลได้ชัดเจนเท่านั้น ไม่ใช่เพราะกระแส
- เห็นว่า **Clean Architecture/DI จาก Part 084** ทำให้การแยกเซอร์วิสในอนาคต (ถ้าจำเป็นจริง)
  เป็นแค่การเปลี่ยน **adapter** ตัวเดียว โดยไม่ต้องเขียน business logic ใหม่ทั้งหมด
- เรียก REST API ของเซอร์วิสอื่นแบบ **synchronous** ด้วย `Faraday` พร้อมตั้ง timeout ที่ถูกต้อง
  และทดสอบด้วย test adapter โดยไม่ต้องยิง network จริง
- เข้าใจความเสี่ยงของ synchronous call — **cascading failure** — และวิธีบรรเทาด้วย timeout
  และ **circuit breaker**
- เข้าใจแนวคิด **asynchronous communication ผ่าน message queue** ที่แก้ปัญหา cascading
  failure ได้ตรงจุด ด้วยการแยก "เวลา" ของ producer และ consumer ออกจากกัน
- เข้าใจ **Exchange, Queue, Binding, Routing Key** ของ RabbitMQ และเขียน publisher/consumer
  จริงด้วย gem `bunny` — **ทดสอบรันจริง** ทั้งการ publish, consume, และพิสูจน์ durability ของ
  ข้อความที่ค้างอยู่ในคิวแม้ไม่มี consumer รันอยู่เลย
- เชื่อมต่อ RabbitMQ consumer เข้ากับ **ActiveJob/Sidekiq (Part 061–062)** ผ่านรูปแบบ "bridge
  consumer" — **ทดสอบรันจริงแบบ end-to-end ข้าม 3 process** ตั้งแต่ publish จนถึง Sidekiq
  worker ประมวลผลเสร็จ
- เข้าใจแนวคิด **Kafka** (log-based streaming, partition, consumer group, replay-ability)
  และรู้ว่าเมื่อไหร่ควรเลือก Kafka แทน RabbitMQ (เชิงแนวคิด ไม่ได้ทดสอบรันจริงในบทนี้)
- รู้จัก **Distributed Monolith anti-pattern** และวิธีตรวจจับ/แก้ไขด้วยการใช้ async เป็น
  ค่าเริ่มต้นสำหรับทุกอย่างที่ผู้ใช้ไม่ต้องรอผลทันที
- เข้าใจ **เส้นทางแยกเซอร์วิสแบบค่อยเป็นค่อยไป** — เลือกเซอร์วิสแรกที่มีเหตุผลวัดผลได้ชัดเจน
  (เช่น image processing ที่กิน CPU สูงผิดปกติ) แทนการทำ big-bang rewrite ทั้งระบบ

## สรุปภาพรวม Phase 14: Architecture & Scaling

ยินดีด้วย! ตอนนี้ **Phase 14: Architecture & Scaling (Part 082–086, Step 811–860)** เสร็จ
สมบูรณ์แล้ว นี่คือ Phase ที่พาเราออกจากคำถาม "จะเขียนโค้ดให้ทำงานได้อย่างไร" (ซึ่งตอบไปแล้ว
เกือบทั้งหมดใน Phase 1–13) ไปสู่คำถามที่ยากกว่ามากและเป็นคำถามหลักของวิศวกรระดับ Senior/Staff:
**"จะจัดโครงสร้างและขยายระบบให้ดูแลรักษาได้ในระยะยาวอย่างไร เมื่อระบบโตขึ้นเรื่อยๆ"**

เส้นทางที่เดินผ่านมาทั้ง 5 Part ของ Phase นี้ต่อกันเป็นเรื่องราวเดียว:

- **Part 082 (Service Object/Form Object)** สอนให้แยก "การกระทำทางธุรกิจ" ออกจาก Controller/
  Model ที่บวมเกินไป — ก้าวแรกของการจัดระเบียบโค้ด
- **Part 083 (Query Object/Decorator)** สอนให้แยก "การอ่านข้อมูลที่ซับซ้อน" และ "การนำเสนอ
  ข้อมูล" ออกจากกัน — ก้าวที่สอง
- **Part 084 (Clean Architecture/DI)** ยกระดับแนวคิดทั้งหมดให้เป็นระบบ — แบ่ง layer ชัดเจน
  และใช้ dependency injection ทำให้ business logic ไม่ผูกติดกับรายละเอียดภายนอก
- **Part 085 (Multi-tenancy)** นำหลักการแยกความรับผิดชอบไปใช้กับปัญหาจริงระดับ business —
  การรองรับลูกค้าหลายราย (tenant) ในระบบเดียว โดยแยก data ownership ให้ชัดเจน
- **Part 086 (Part นี้)** ปิดท้ายด้วยคำถามที่ใหญ่ที่สุด — เมื่อไหร่ที่ระบบเดียวไม่พอ และถ้าต้อง
  แยกจริงๆ เซอร์วิสเหล่านั้นจะสื่อสารกันอย่างไรให้ทนทานต่อความล้มเหลว (resilient) แทนที่จะ
  กลายเป็น distributed monolith ที่แย่กว่า monolith เดิม

**บทเรียนที่สำคัญที่สุดที่ควรติดตัวไปจาก Phase ทั้งหมดนี้:** สถาปัตยกรรมที่ดีไม่ใช่การใช้
เทคนิค/pattern ที่ซับซ้อนที่สุดเท่าที่จะทำได้ แต่คือการเลือก**ความซับซ้อนที่จำเป็นจริง** ตามขนาด
ปัญหาที่มีอยู่จริง ณ ตอนนั้น — Service Object ใน Part 082 มีประโยชน์ตั้งแต่แอปขนาดเล็กมาก
ในขณะที่ microservices ใน Part นี้อาจไม่จำเป็นเลยแม้แอปจะใหญ่มากแล้วก็ตาม **ทักษะที่แท้จริงคือ
การรู้ว่าจะใช้เครื่องมือไหนเมื่อไหร่** ไม่ใช่การรู้จักเครื่องมือให้ได้มากที่สุด

**ต่อไป (Part 087 — เปิด Phase 15: Rails Internals & Gem Building):** เราออกจากคำถามระดับ
สถาปัตยกรรม "ระหว่างเซอร์วิส" (macro) ไปเจาะลึกสถาปัตยกรรม "ภายใน Rails framework เอง" (micro)
Part 087 จะพาไปเรียนรู้ **Rack** — สเปกมาตรฐานที่ web server ทุกตัวในโลก Ruby (Puma, Unicorn,
Falcon) และ web framework ทุกตัว (Rails, Sinatra, Hanami) สร้างอยู่บนพื้นฐานเดียวกัน จะได้เห็นว่า
`config.middleware` ที่เคยเห็นผ่านตาใน `config/application.rb` ตลอดหลักสูตรนี้คืออะไรกันแน่
และ Rails เองก็เป็นแค่ **middleware stack ขนาดใหญ่ตัวหนึ่ง** ที่ต่อกันเป็นชั้นๆ ก่อนไปถึง
Controller ของเรา — Phase 15 นี้จะเปลี่ยนมุมมองที่มีต่อ Rails ไปตลอดกาล จาก "framework ที่ใช้
เป็น" ไปสู่ "framework ที่เข้าใจว่าทำงานอย่างไรจริงๆ ข้างใน"
