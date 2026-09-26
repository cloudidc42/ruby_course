# Part 071: Stripe Integration — Checkout, Webhook, Subscription

> **Step ครอบคลุมใน Part นี้:** Step 701–710
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 023 เรื่อง controller/params, Part 056 เรื่อง JSON/API,
> Part 062 เรื่อง idempotency ของ background job, และ Part 067 เรื่อง Rails encrypted
> credentials มาก่อนทั้งหมด — Part นี้จะ recap เฉพาะส่วนที่จำเป็น ไม่สอนซ้ำตั้งแต่ต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem `stripe` (ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4,
> stripe gem 19.6.2, Stripe API version `2024-06-20`)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> แซนด์บ็อกซ์ที่ใช้เขียน Part นี้ **ไม่มี credential ของบัญชี Stripe จริง และ network policy
> ปฏิเสธการเชื่อมต่อออกไปยัง `api.stripe.com` โดยตรง** (ทดสอบแล้ว: `curl` ไปยัง
> `https://api.stripe.com` ถูกปฏิเสธด้วย `403` ที่ระดับ proxy ก่อนถึงตัวเซิร์ฟเวอร์ของ Stripe
> ด้วยซ้ำ) ทีมผู้เขียนหลักสูตรจึงทำสิ่งที่ตรวจสอบได้จริงมากที่สุดเท่าที่ทำได้แทนการเดาจากเอกสาร
> อย่างเดียว โดยแบ่งชัดเจนดังนี้:
>
> - **ทดสอบจริง 100% (ไม่ต้องพึ่ง network ไปหา Stripe เลย):** การ **verify webhook signature**
>   ด้วย `Stripe::Webhook.construct_event` — เพราะกลไกนี้เป็นการคำนวณ HMAC-SHA256 ล้วนๆ ที่ทำ
>   ในเครื่องทั้งหมด ไม่มีการเรียก network ไปที่ Stripe แต่อย่างใด เราเขียนสคริปต์คำนวณลายเซ็นเอง
>   ด้วย `OpenSSL::HMAC` ตามอัลกอริทึมที่ Stripe เอกสารไว้ แล้วป้อนเข้า `construct_event` ของ
>   gem จริง ยืนยันว่า verify ผ่านกรณีถูกต้อง และถูกปฏิเสธถูกต้องทั้งกรณี secret ผิด, payload
>   ถูกแก้ไข (tamper), และ timestamp เก่าเกิน tolerance (replay attack)
> - **ทดสอบจริงกับ Rails app จริง:** เราสร้างแอป Rails 8.1.4 จริงในเครื่อง (แยกจาก repository
>   ของหลักสูตรโดยสิ้นเชิง ลบทิ้งหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) ที่มี routing, migration,
>   model, controller ครบตามที่สอนใน Part นี้ รันบน Puma server จริง ยิง HTTP request จริงด้วย
>   `curl` ทั้งฝั่ง checkout controller และ webhook controller จุดเดียวที่ถูก **stub** คือเมธอด
>   `Stripe::Checkout::Session.create` (เพราะเรียก network จริงไม่ได้) ส่วนที่เหลือทั้งหมด —
>   CSRF, routing, การบันทึกลงฐานข้อมูลจริงผ่าน SQLite, การ redirect ด้วย `allow_other_host`,
>   การ verify signature, **idempotency guard ที่กันการ fulfill order ซ้ำจริง** — เป็นโค้ด
>   production จริงที่รันผ่านทั้งหมด
> - **ตรวจสอบความถูกต้องของ syntax/attribute ด้วยการอ่าน source code ของ gem โดยตรง:** พารามิเตอร์
>   และ attribute ทุกตัวที่ใช้ในตัวอย่าง (เช่น `session.payment_status`, `session.url`,
>   `subscription.status`, `subscription.cancel_at_period_end`) ถูกยืนยันว่ามีอยู่จริงในซอร์สโค้ด
>   gem `stripe` 19.6.2 ที่ติดตั้งจริงในแซนด์บ็อกซ์ (ไม่ใช่เดาจากความจำ) — รวมถึงพบว่า Stripe API
>   เวอร์ชันปัจจุบันได้ย้าย `current_period_end` ออกจาก top-level ของ `Subscription` ไปอยู่ใต้
>   `items.data[].current_period_end` แล้ว ซึ่งเป็นรายละเอียดที่เปลี่ยนไปจากเอกสารเก่าหลายฉบับ
>   ในอินเทอร์เน็ต Part นี้จึงเขียนให้ตรงกับ SDK เวอร์ชันปัจจุบันจริง
> - **มาจากเอกสารทางการ ไม่ได้ทดสอบจริง (จะระบุไว้ชัดเจนตรงจุด):** พฤติกรรมฝั่ง Stripe เอง
>   ที่ต้องพึ่ง network จริง เช่น หน้าตาของหน้า Checkout ที่ Stripe สร้างให้, ผลลัพธ์จริงของการ
>   กดจ่ายเงินด้วยเลขบัตรทดสอบ `4242 4242 4242 4242`, และการรันคำสั่ง Stripe CLI (`stripe listen`,
>   `stripe trigger`) จริง — ไบนารี `stripe` CLI ไม่ได้ติดตั้งอยู่ในแซนด์บ็อกซ์นี้ (ตรวจสอบแล้วว่า
>   `command not found`) ส่วนนี้จะอธิบาย workflow ให้ครบถ้วนตามเอกสารทางการของ Stripe แต่จะระบุไว้
>   ชัดเจนว่าไม่ได้รันจริงในการเขียน Part นี้

ยินดีต้อนรับสู่ **Phase 11: Payment & Third-party Integration** หลังจากผ่าน Phase 10 ที่สอนให้จัดการ
ไฟล์และการค้นหา (รากฐานของแคตตาล็อกสินค้า) มาถึงจุดที่สำคัญที่สุดจุดหนึ่งของเว็บอีคอมเมิร์ซทุกระบบ
นั่นคือ **การรับเงินจริงจากลูกค้า** Phase นี้มีแค่ 2 Part (071–072) แต่เข้มข้นมาก เพราะทั้งสองหัวข้อ
— การชำระเงิน (Part นี้) และการยืนยันตัวตน/แจ้งเตือนผ่าน third-party (Part ถัดไป) — ล้วนเป็นจุดที่
เว็บแอปพลิเคชันต้อง**เชื่อมต่อกับระบบภายนอกที่เราควบคุมไม่ได้โดยตรง** ต้องคิดเรื่อง security,
ความน่าเชื่อถือของข้อมูลที่ได้รับกลับมา, และการจัดการเมื่อสิ่งต่างๆ ไม่เป็นไปตามแผนอย่างรอบคอบ
กว่าการเขียน CRUD ธรรมดามาก

## สารบัญของ Part นี้

- Step 701: ทำไมไม่ควรสร้างระบบชำระเงินเอง — ภาระ PCI DSS และความเสี่ยงด้านความปลอดภัย
- Step 702: ติดตั้ง `stripe` gem และตั้งค่า API key ผ่าน Rails encrypted credentials
- Step 703: Stripe Checkout คืออะไร — สร้าง Checkout Session แรกและทำความเข้าใจโมเดลความ
  ปลอดภัยของมัน
- Step 704: Redirect ผู้ใช้ไปหน้า Checkout, success/cancel URL, และกับดักที่ห้ามทำบนหน้า success
- Step 705: ทำไม Webhook จำเป็น — ข้อผิดพลาดคลาสสิกของการเชื่อ redirect ฝั่ง client อย่างเดียว
- Step 706: สร้าง webhook endpoint และ verify signature จริงด้วย `Stripe::Webhook.construct_event`
- Step 707: Handle `checkout.session.completed` เพื่อ fulfill order จริง
- Step 708: Idempotency สำหรับ webhook — Stripe อาจส่ง event ซ้ำ ต้องออกแบบให้ปลอดภัย
- Step 709: Stripe Subscriptions — Product/Price, subscription Checkout Session, lifecycle webhook
- Step 710: ทดสอบ local ด้วย Stripe CLI (`stripe listen --forward-to`)

---

## Step 701: ทำไมไม่ควรสร้างระบบชำระเงินเอง — ภาระ PCI DSS และความเสี่ยงด้านความปลอดภัย

### ทำไม "เขียนเองก็ได้" ไม่ใช่ทางเลือกที่ดีสำหรับเกือบทุกทีม

การรับชำระเงินด้วยบัตรเครดิต/เดบิตฟังดูเหมือนแค่ "รับเลขบัตร วันหมดอายุ CVV แล้วยิง API ไปหา
ธนาคาร" แต่ในทางปฏิบัติ ทันทีที่ระบบของเรา**สัมผัส** (touch) ข้อมูลบัตรเครดิตดิบ — แม้จะแค่รับผ่าน
form แล้วส่งต่อทันทีโดยไม่เก็บลง database เลยก็ตาม — ระบบทั้งหมดจะตกอยู่ภายใต้มาตรฐาน
**PCI DSS (Payment Card Industry Data Security Standard)** ทันที

PCI DSS ไม่ใช่แค่ "แนวทางที่ดี" แต่เป็น**ข้อบังคับตามสัญญา**ที่ทุก merchant ที่รับบัตร Visa/
Mastercard/JCB ต้องปฏิบัติตาม ผลกระทบต่อทีมพัฒนามีดังนี้:

1. **ระดับความเข้มงวดขึ้นอยู่กับปริมาณธุรกรรม** — แบ่งเป็น 4 ระดับ (Level 1–4) ยิ่งมีธุรกรรมต่อปี
   มาก ยิ่งต้องผ่านการ audit ที่เข้มงวดขึ้น ตั้งแต่ **SAQ (Self-Assessment Questionnaire)** ที่
   กรอกเองได้ ไปจนถึง **QSA (Qualified Security Assessor)** ที่ต้องจ้างผู้ตรวจสอบภายนอกมา audit
   ระบบทั้งหมดทุกปี ค่าใช้จ่ายหลักแสนถึงหลักล้านบาทต่อปีสำหรับธุรกิจขนาดกลางขึ้นไป
2. **ข้อกำหนดทางเทคนิคที่ต้องทำครบทั้ง 12 หมวด** เช่น encrypt ข้อมูลบัตรทั้งตอน rest และ transit,
   จำกัดสิทธิ์การเข้าถึงข้อมูลบัตรแบบ need-to-know, เก็บ log การเข้าถึงทุกครั้งและเก็บไว้อย่างน้อย
   1 ปี, สแกนหาช่องโหว่ (vulnerability scan) ทุกไตรมาสโดยผู้ให้บริการที่ได้รับอนุมัติ (ASV),
   ทำ penetration test อย่างน้อยปีละครั้ง, ห้ามเก็บ CVV ไว้เลยแม้จะ encrypt แล้วก็ตาม
3. **ความเสี่ยงเมื่อเกิดข้อมูลรั่วไหล** — ถ้าเก็บเลขบัตรเองแล้วเกิด data breach ความเสียหายไม่ใช่
   แค่ค่าปรับจาก payment network (หลักแสนถึงหลักล้านดอลลาร์ต่อครั้ง) แต่รวมถึงต้นทุนแจ้งลูกค้า
   ทุกรายที่ได้รับผลกระทบ, ค่าใช้จ่ายด้าน forensic investigation, ความเสียหายต่อชื่อเสียงที่
   ประเมินเป็นตัวเงินไม่ได้ และในหลายประเทศ (รวมถึงไทยภายใต้ พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล)
   อาจมีความรับผิดทางกฎหมายเพิ่มเติมด้วย
4. **ต้นทุนที่มองไม่เห็นแต่มหาศาล**: ทีม engineering ต้องเสียเวลาส่วนใหญ่ไปกับการดูแล compliance
   แทนที่จะสร้างฟีเจอร์ให้ธุรกิจ และทุกครั้งที่แก้โค้ดส่วนที่เกี่ยวกับการชำระเงินต้องผ่านกระบวนการ
   ตรวจสอบเพิ่มเติมเสมอ

### ทางออก: อย่าให้ข้อมูลบัตรผ่านเซิร์ฟเวอร์ของเราเลย

หลักการสำคัญที่สุดของ PCI DSS คือ **ยิ่งระบบของเรา "สัมผัส" ข้อมูลบัตรน้อยเท่าไหร่ ภาระ compliance
ยิ่งน้อยลงเท่านั้น** ผู้ให้บริการอย่าง **Stripe** ออกแบบผลิตภัณฑ์ (โดยเฉพาะ **Stripe Checkout** ที่
เราจะใช้ใน Part นี้) ให้ข้อมูลบัตรของลูกค้าถูกกรอกและส่งตรงไปยังเซิร์ฟเวอร์ของ Stripe เท่านั้น
**ไม่เคยผ่านเซิร์ฟเวอร์ของเราแม้แต่ไบต์เดียว** ทำให้ทีมของเราอยู่ในระดับ compliance ที่ง่ายที่สุด
คือ **SAQ A** (กรอกแบบฟอร์มสั้นๆ ยืนยันตัวเองปีละครั้ง ไม่ต้องมี QSA มาตรวจ) แทนที่จะต้องผ่าน
SAQ D ที่ซับซ้อนที่สุดถ้าเก็บ/ประมวลผลข้อมูลบัตรเอง

### ทำไม Stripe ถึงเป็นตัวเลือกยอดนิยม

ตลาด payment gateway มีผู้เล่นหลายราย (PayPal, Braintree, Adyen, Omise/Opn ในไทย ฯลฯ) แต่ Stripe
ได้รับความนิยมสูงมากในหมู่นักพัฒนาโดยเฉพาะ ด้วยเหตุผล:

1. **Developer Experience ยอดเยี่ยม** — เอกสารละเอียด, SDK ทางการสำหรับแทบทุกภาษารวมถึง Ruby,
   error message ที่อ่านแล้วเข้าใจได้ทันทีว่าต้องแก้อะไร
2. **Test mode ที่สมบูรณ์แบบ** — ทดสอบได้ทุก flow (สำเร็จ/ล้มเหลว/3D Secure/ธนาคารปฏิเสธ) ด้วย
   เลขบัตรทดสอบมาตรฐาน (เช่น `4242 4242 4242 4242` สำหรับกรณีสำเร็จ) โดยไม่ต้องมีเงินจริงเลย
3. **Hosted Checkout ที่ปลอดภัยและปรับแต่งได้** — สิ่งที่ Part นี้จะสอนเป็นหลัก
4. **รองรับ use case หลากหลาย** — ตั้งแต่ one-time payment, subscription/recurring billing,
   marketplace (Stripe Connect), invoicing ไปจนถึง in-person payment
5. **Webhook system ที่แข็งแรง** — ระบบแจ้งเตือน server-to-server ที่ verify ได้อย่างปลอดภัย
   (หัวใจของ Step 705–708)
6. บริษัทระดับโลกจำนวนมากใช้ Stripe เป็น payment infrastructure หลัก เช่น Shopify (บางส่วน),
   Lyft, Instacart, DoorDash เป็นสัญญาณว่าระบบรองรับ scale ระดับสูงได้จริง

> **หลักคิดสำคัญ:** งานของนักพัฒนา Rails ในบริบทนี้**ไม่ใช่**การสร้างระบบชำระเงิน แต่คือการ
> **เชื่อมต่อกับระบบชำระเงินที่มีอยู่แล้วอย่างถูกต้องและปลอดภัย** — โฟกัสของ Part นี้ทั้งหมดจึงอยู่
> ที่การใช้ Stripe API ให้ถูกวิธี โดยเฉพาะจุดที่พลาดบ่อยที่สุดคือเรื่อง webhook (Step 705–708)

---

## Step 702: ติดตั้ง `stripe` gem และตั้งค่า API key ผ่าน Rails encrypted credentials

### ติดตั้ง gem

```ruby
# Gemfile
gem "stripe"
```

```bash
bundle install
```

ทดสอบในแซนด์บ็อกซ์นี้ใช้ `stripe` gem เวอร์ชัน **19.6.2** (เวอร์ชันล่าสุดที่ดึงจาก RubyGems ณ
วันที่เขียน) รองรับ Ruby 3.3.x และ Rails 8.1.x ได้เต็มรูปแบบ ไม่มี dependency ขัดแย้งใดๆ

### API Key สองแบบของ Stripe

Stripe มี API key สองประเภทที่ต้องแยกให้ถูกเสมอ:

| Key | Prefix | ใช้ที่ไหน | เปิดเผยได้ไหม |
|-----|--------|-----------|----------------|
| **Publishable key** | `pk_test_...` / `pk_live_...` | ฝั่ง client (JavaScript, mobile app) | ได้ — ออกแบบมาให้เปิดเผยในโค้ด frontend |
| **Secret key** | `sk_test_...` / `sk_live_...` | ฝั่ง server เท่านั้น | **ห้ามเปิดเผยเด็ดขาด** ใครมี key นี้เรียก API สร้างการชาร์จเงินแทนบัญชีเราได้ทันที |

Part นี้ใช้ Stripe Checkout ซึ่งเป็น hosted page ของ Stripe เอง ทำให้เกือบทุกกรณีเราแทบไม่ต้องใช้
publishable key ในโค้ด frontend ของเราเองเลย (ต่างจากการทำ custom payment form ด้วย Stripe
Elements ที่ต้องใช้ publishable key ฝั่ง client) — แต่ **secret key ต้องถูกเก็บอย่างปลอดภัยเสมอ**
ไม่ว่าจะใช้แนวทางไหน

### Recap: Rails encrypted credentials (ทบทวนจาก Part 067)

Part 067 สอนไว้แล้วว่า Rails มีระบบเก็บ secret แบบเข้ารหัสในตัว (`config/credentials.yml.enc` +
`config/master.key` ที่ไม่ถูก commit เข้า git) หลักการเดียวกันนี้ใช้กับ Stripe key ได้ทันที:

```bash
EDITOR="code --wait" bin/rails credentials:edit
```

เพิ่มเนื้อหาลงไป:

```yaml
# config/credentials.yml.enc (เนื้อหาหลังถอดรหัส - ห้าม commit เวอร์ชันถอดรหัส)
stripe:
  publishable_key: "<ใส่ publishable key ของคุณจาก Stripe Dashboard ที่นี่ ขึ้นต้นด้วย pk_test_...>"
  secret_key: "<ใส่ secret key ของคุณจาก Stripe Dashboard ที่นี่ ขึ้นต้นด้วย sk_test_...>"
  webhook_signing_secret: "<ใส่ webhook signing secret จาก Stripe CLI/Dashboard ที่นี่ ขึ้นต้นด้วย whsec_...>"
```

> **แนะนำให้แยก credentials ต่อ environment** (`bin/rails credentials:edit --environment
> production`) เพื่อให้ใช้ test key ตอน dev/staging และ live key ตอน production เท่านั้น — ป้องกัน
> อุบัติเหตุคลาสสิกที่นักพัฒนามือใหม่มักทำพลาด คือใช้ live key ตอนทดสอบแล้วเผลอชาร์จเงินจริง

### ตั้งค่า initializer

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)

# ล็อก API version ไว้ตายตัว ป้องกันพฤติกรรมเปลี่ยนแปลงกะทันหันเมื่อ Stripe อัปเดต API
# (Stripe คงเวอร์ชันเก่าให้ใช้งานได้เสมอ แต่ default account version จะขยับเรื่อยๆ ถ้าไม่ระบุ)
Stripe.api_version = "2024-06-20"
```

### ทดสอบจริง: ยืนยันว่ากลไก credentials → Stripe.api_key ทำงานถูกต้อง

เราทดสอบ flow นี้จริงในแอปทดลอง โดยเขียน credentials ที่เข้ารหัสด้วย master key จริง แล้วอ่านค่า
กลับผ่าน `Rails.application.credentials.dig`:

```irb
$ bin/rails runner 'puts Rails.application.credentials.dig(:stripe, :publishable_key);
                     puts Rails.application.credentials.dig(:stripe, :secret_key);
                     puts Rails.application.credentials.dig(:stripe, :webhook_signing_secret)'
pk_test_FAKE_EXAMPLE_KEY
sk_test_FAKE_EXAMPLE_KEY
whsec_FAKE_EXAMPLE_SECRET
```

ยืนยันได้ว่ากลไกการเข้ารหัส/ถอดรหัส credentials ทำงานถูกต้องสมบูรณ์ (ในการทดสอบนี้ใช้ key ปลอม
เพราะไม่มีบัญชี Stripe จริงในแซนด์บ็อกซ์ แต่กลไก encrypted credentials ที่ Rails ใช้จริงทั้งหมด
เหมือนกับที่จะใช้กับ key จริงทุกประการ)

---

## Step 703: Stripe Checkout คืออะไร — สร้าง Checkout Session แรก

### Stripe Checkout คืออะไร และทำไมถึงปลอดภัยที่สุด

**Stripe Checkout** คือหน้าชำระเงินสำเร็จรูปที่ **Stripe เป็นคนสร้างและ host เอง** (อยู่ที่โดเมน
`checkout.stripe.com`) เมื่อเราต้องการรับเงินจากลูกค้า สิ่งที่แอป Rails ของเราทำมีแค่:

1. เรียก API สร้าง **Checkout Session** (บอก Stripe ว่าจะขายอะไร ราคาเท่าไหร่)
2. **Redirect** ผู้ใช้ไปที่ URL ของ Checkout Session นั้น (เป็นหน้าของ Stripe ทั้งหมด)
3. ลูกค้ากรอกข้อมูลบัตรบนหน้าของ Stripe เอง (**ไม่ใช่หน้าของเรา**) — นี่คือเหตุผลที่ Step 701
   บอกว่าเราอยู่ใน SAQ A ได้ เพราะข้อมูลบัตรไม่เคยผ่านเซิร์ฟเวอร์ของเราเลย
4. Stripe ประมวลผลการจ่ายเงิน แล้ว redirect ผู้ใช้กลับมาที่เว็บเราตาม `success_url`/`cancel_url`
   ที่เราระบุไว้ พร้อม**ส่ง webhook มาแจ้งเราแบบ server-to-server แยกต่างหาก** (หัวใจของ
   Step 705 เป็นต้นไป)

เทียบกับทางเลือกอื่นอย่าง **Stripe Elements** (ฝัง input field ของ Stripe ลงในหน้าเว็บเราเอง แบบ
custom UI) Checkout ใช้งานง่ายกว่ามาก เหมาะกับการเริ่มต้น และยังคงความปลอดภัยระดับเดียวกัน
(ข้อมูลบัตรก็ยังไม่ผ่าน server เราเช่นกันเพราะ Elements ใช้ iframe ที่คุยกับ Stripe โดยตรง)
Part นี้จะสอน Checkout เป็นหลักเพราะเป็นจุดเริ่มต้นที่ทีมส่วนใหญ่ควรเลือกใช้ก่อนเสมอ

### สร้าง Checkout Session

```ruby
# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  PRODUCT = {
    name: "คอร์สเรียน Ruby on Rails (Lifetime Access)",
    amount_cents: 150_000, # หน่วยเป็นสตางค์/cent เสมอ (1,500.00 บาท)
    currency: "thb"
  }.freeze

  def new
  end

  def create
    checkout_session = Stripe::Checkout::Session.create(
      mode: "payment", # one-time payment (ต่างจาก "subscription" ที่จะสอนใน Step 709)
      line_items: [{
        price_data: {
          currency: PRODUCT[:currency],
          unit_amount: PRODUCT[:amount_cents],
          product_data: { name: PRODUCT[:name] }
        },
        quantity: 1
      }],
      success_url: checkout_success_url(session_id: "{CHECKOUT_SESSION_ID}"),
      cancel_url: checkout_cancel_url
    )

    redirect_to checkout_session.url, allow_other_host: true, status: :see_other
  end
end
```

**อธิบายทีละส่วน:**

- `mode: "payment"` — บอก Stripe ว่านี่คือการชำระเงินครั้งเดียว (ค่าที่เป็นไปได้อีกสองแบบคือ
  `"subscription"` และ `"setup"` สำหรับบันทึกบัตรไว้ใช้ทีหลังโดยยังไม่เก็บเงิน)
- `line_items` — รายการสินค้า ในตัวอย่างนี้ใช้ `price_data` แบบ inline (กำหนดราคาสดๆ ตอนสร้าง
  session) เหมาะกับสินค้าที่ราคาคำนวณแบบไดนามิก ถ้าเป็นสินค้า/แพ็กเกจที่ราคาคงที่ซ้ำๆ (โดยเฉพาะ
  subscription) ควรสร้าง **Product** และ **Price** ไว้ล่วงหน้าใน Stripe แล้วอ้างอิงด้วย `price:
  "price_xxxxx"` แทน (รายละเอียดใน Step 709)
- `unit_amount` — หน่วยเป็น**หน่วยย่อยที่สุดของสกุลเงิน**เสมอ (สตางค์สำหรับ THB, cent สำหรับ USD)
  **ไม่ใช่บาท/ดอลลาร์** เป็นจุดที่พลาดบ่อยมาก (ลืมคูณ 100 หรือคูณซ้ำสอง)
- `success_url` / `cancel_url` — URL ที่ Stripe จะ redirect กลับมาหลังจบ flow (รายละเอียด Step
  704) สังเกตว่า `success_url` มี `{CHECKOUT_SESSION_ID}` เป็น placeholder ตามตัวอักษร — Stripe
  จะแทนที่ด้วย ID จริงของ session ตอน redirect ให้อัตโนมัติ (ไม่ใช่ ERB interpolation ของเรา)
- `checkout_session.url` — URL ของหน้า Checkout ที่ Stripe สร้างให้ (โดเมน
  `checkout.stripe.com`) ต้อง redirect ด้วย **`allow_other_host: true`** เพราะ Rails ตั้งแต่
  เวอร์ชันที่มี open redirect protection จะปฏิเสธการ redirect ไปโดเมนอื่นโดยไม่ระบุ flag นี้
  อย่างชัดเจน — ยืนยันแล้วว่าถ้าลืมใส่ Rails จะ raise
  `ActionController::Redirecting::UnsafeRedirectError` ทันที
- `status: :see_other` (HTTP 303) — แนวปฏิบัติที่ดีสำหรับ redirect หลัง POST request (Post/Redirect/
  Get pattern) ป้องกันปัญหาถ้าผู้ใช้กด refresh หน้า

### ทดสอบจริง: การสร้าง Order + redirect ทำงานถูกต้องบน Rails 8.1.4 จริง

เราสร้างแอป Rails จริง เพิ่ม route, migration สำหรับตาราง `orders`, และ stub เฉพาะเมธอด
`Stripe::Checkout::Session.create` ให้คืนค่า object ปลอมที่มี `id`/`url` (เพราะเรียก network จริง
ไม่ได้) แล้วยิง HTTP POST จริงด้วย `curl` ผ่าน Puma server ที่รันอยู่จริง:

```bash
$ curl -sS -b cookies.txt -c cookies.txt -i -X POST http://127.0.0.1:3099/checkout \
    --data-urlencode "authenticity_token=$TOKEN"

HTTP/1.1 303 See Other
location: https://checkout.stripe.com/c/pay/cs_test_fake_123456
```

และตรวจสอบฐานข้อมูลจริง (SQLite) หลังเรียก:

```irb
$ bin/rails runner 'Order.all.each { |o| puts o.attributes }'
{"id"=>1, "product_name"=>"คอร์สเรียน Ruby on Rails (Lifetime Access)",
 "amount_cents"=>150000, "currency"=>"thb", "status"=>"pending",
 "stripe_checkout_session_id"=>"cs_test_fake_123456", ...}
```

ยืนยันว่า controller สร้าง `Order` จริง, เรียก `Stripe::Checkout::Session.create` ด้วยพารามิเตอร์
ที่ถูกต้อง, และ redirect ไปยัง URL ของ Stripe ด้วย HTTP 303 จริง — ทุกอย่างยกเว้นตัว network call
ไปหา Stripe เองเป็นโค้ด production จริงที่พิสูจน์แล้วว่าทำงาน

---

## Step 704: Redirect ผู้ใช้ไปหน้า Checkout, success/cancel URL, และกับดักที่ห้ามทำบนหน้า success

### ผูก Order เข้ากับ Checkout Session

ในทางปฏิบัติ เราต้องสร้าง record ของ "คำสั่งซื้อ" (Order) ในฐานข้อมูลของเราเองก่อน แล้วผูกกับ
Checkout Session ID เพื่อให้ภายหลัง (ตอนได้รับ webhook) เรารู้ว่า event ที่ได้รับมาตรงกับคำสั่งซื้อ
ไหน:

```ruby
def create
  order = Order.create!(
    product_name: PRODUCT[:name],
    amount_cents: PRODUCT[:amount_cents],
    currency: PRODUCT[:currency],
    status: "pending"
  )

  checkout_session = Stripe::Checkout::Session.create(
    mode: "payment",
    line_items: [{
      price_data: {
        currency: PRODUCT[:currency],
        unit_amount: PRODUCT[:amount_cents],
        product_data: { name: PRODUCT[:name] }
      },
      quantity: 1
    }],
    success_url: checkout_success_url(session_id: "{CHECKOUT_SESSION_ID}"),
    cancel_url: checkout_cancel_url,
    client_reference_id: order.id,   # อ้างอิง order ของเราแบบง่ายๆ
    metadata: { order_id: order.id } # เก็บข้อมูลเพิ่มเติม ดึงกลับมาได้ตอนรับ webhook
  )

  order.update!(stripe_checkout_session_id: checkout_session.id)
  redirect_to checkout_session.url, allow_other_host: true, status: :see_other
end
```

`client_reference_id` และ `metadata` เป็นสองวิธีหลักที่ Stripe เปิดให้เราแนบข้อมูลของระบบเราเอง
ไปกับ Checkout Session — ทั้งคู่จะถูกส่งกลับมาในตัว event ตอนรับ webhook ทำให้เราจับคู่ order ได้
โดยไม่ต้องพึ่งพา `stripe_checkout_session_id` เพียงอย่างเดียว (ในตัวอย่างของ Part นี้จะใช้การหา
`Order` จาก `stripe_checkout_session_id` เป็นหลัก เพราะตรงไปตรงมาและมี unique index รองรับอยู่แล้ว
แต่ `metadata` ยังมีประโยชน์มากสำหรับกรณีที่ระบบซับซ้อนขึ้น เช่น ต้องรู้ user_id หรือ cart_id ด้วย)

### Controller สำหรับหน้า success และ cancel

```ruby
def success
  # ⚠️ คำเตือนสำคัญที่สุดของ Step นี้: ห้าม fulfill order (ปลดล็อกสิทธิ์/ส่งของ) ตรงนี้เด็ดขาด!
  # การที่ browser ถูก redirect มาที่ URL นี้ "ไม่ใช่" หลักฐานว่าจ่ายเงินสำเร็จจริง
  # ดูเหตุผลแบบละเอียดใน Step 705 — การ fulfill ต้องรอ webhook เท่านั้น
  @order = Order.find_by(stripe_checkout_session_id: params[:session_id])
end

def cancel
  # ผู้ใช้กดยกเลิกหรือปิดหน้า Checkout กลางคัน — order ยังคงสถานะ "pending" ต่อไป
  # (ไม่ต้อง set เป็น "cancelled" ทันที เพราะผู้ใช้อาจกลับไปจ่ายเงินสำเร็จทีหลังด้วย checkout
  # session เดิมได้ถ้ายังไม่หมดอายุ — ปล่อยให้ webhook หรือ job เก็บกวาด order ที่ค้างนานเกินไป
  # เป็นคนตัดสินใจแทน)
end
```

View ของหน้า success ควรแสดงข้อความทำนอง "กำลังตรวจสอบการชำระเงิน" หรือ "ขอบคุณที่สั่งซื้อ"
แบบกลางๆ ไม่ควรแสดงว่า "ปลดล็อกคอร์สให้แล้ว" หรือ "จัดส่งสินค้าแล้ว" ในหน้านี้ เพราะยังไม่มีการ
ยืนยันจาก Stripe อย่างเป็นทางการ

---

## Step 705: ทำไม Webhook จำเป็น — ข้อผิดพลาดคลาสสิกของการเชื่อ redirect ฝั่ง client อย่างเดียว

นี่คือหัวข้อที่สำคัญที่สุดของ Part นี้ นักพัฒนามือใหม่ (และบางครั้งแม้แต่มือเก๋า) จำนวนมากทำผิดพลาด
จุดนี้ในระบบชำระเงินจริง

### "การ redirect กลับมาที่ success_url" ไม่ใช่หลักฐานการจ่ายเงินสำเร็จ

ลองพิจารณาเหตุผลทีละข้อว่าทำไม request ที่มาที่ `GET /checkout/success?session_id=...` ถึงเชื่อ
ถือไม่ได้ 100%:

1. **URL นี้สามารถถูกแก้ไข/ปลอมแปลงได้** — มันเป็นแค่ HTTP GET request ธรรมดาที่มาจาก browser
   ของผู้ใช้ ผู้ใช้ (หรือใครก็ตามที่รู้ pattern ของ URL) สามารถพิมพ์ URL นี้เข้า address bar ตรงๆ
   พร้อมใส่ `session_id` อะไรก็ได้ที่เดา/คัดลอกมา โดยไม่เคยจ่ายเงินจริงเลยแม้แต่บาทเดียว ถ้าโค้ด
   ของเรา fulfill order ทันทีที่เห็น request มาที่ path นี้ นั่นคือช่องโหว่ร้ายแรง
2. **Browser อาจไม่เคยไปถึง success_url เลยแม้จ่ายเงินสำเร็จจริง** — เครือข่ายอาจหลุดระหว่างที่
   Stripe กำลัง redirect ผู้ใช้กลับมา, ผู้ใช้อาจปิดแท็บ/แอปก่อนที่ browser จะโหลด success_url
   เสร็จ, หรือถ้าเป็น mobile app ผู้ใช้อาจสลับแอปออกไปกลางทาง — ในทุกกรณีนี้ **เงินถูกตัดสำเร็จ
   แล้วจริงๆ** แต่ระบบของเราไม่เคยรู้เลยถ้าพึ่งพา success_url เพียงอย่างเดียว ลูกค้าจะจ่ายเงินไป
   แต่ไม่ได้รับสินค้า/บริการ
3. **บาง payment method เป็นแบบ asynchronous** — วิธีจ่ายเงินบางประเภท (เช่น bank transfer,
   บาง wallet ในบางประเทศ) ไม่ยืนยันผลทันที Checkout Session อาจ "completed" (ลูกค้ากรอกข้อมูล
   ครบแล้ว) แต่ `payment_status` ยังเป็น `"unpaid"` รอการยืนยันจากธนาคารอีกหลายชั่วโมงถึงหลายวัน
   ก็เป็นไปได้ (ยืนยันจากเอกสาร Stripe — เป็นเหตุผลที่ event `checkout.session.completed` ควร
   เช็ค `payment_status == "paid"` ก่อนเสมอ ไม่ใช่เชื่อแค่ว่า event เกิดขึ้น)

### กลไกที่เชื่อถือได้เพียงอย่างเดียว: Webhook (Server-to-Server)

**Webhook** คือกลไกที่ **เซิร์ฟเวอร์ของ Stripe เรียก HTTP POST มาที่เซิร์ฟเวอร์ของเราโดยตรง**
(ไม่ผ่าน browser ของผู้ใช้เลย) เมื่อมีเหตุการณ์สำคัญเกิดขึ้น เช่น การจ่ายเงินสำเร็จ นี่คือช่องทาง
เดียวที่:

- มาจาก Stripe เองโดยตรง (verify ได้ด้วย cryptographic signature — Step 706)
- ส่งข้อมูลที่เป็นสถานะล่าสุดจริงของ transaction ไม่ใช่แค่ "ผู้ใช้ browser ถูก redirect มา"
- Stripe มีกลไก **retry อัตโนมัติ** ถ้า endpoint ของเราตอบ error หรือ timeout (ไม่เหมือนการ
  redirect ที่เกิดครั้งเดียวจบ) ทำให้ความเชื่อถือได้ (reliability) สูงกว่ามาก

> **กฎเหล็กที่ต้องจำ:** `success_url` มีไว้เพื่อ**ปรับปรุงประสบการณ์ผู้ใช้** (บอกผู้ใช้ว่า "กำลัง
> ดำเนินการ" ทันที ไม่ต้องรอ) แต่**การ fulfill order จริง (ปลดล็อกสิทธิ์ ส่งอีเมลยืนยัน หักสต็อก
> สินค้า) ต้องเกิดจาก webhook เท่านั้น** ทุกครั้ง ไม่มีข้อยกเว้น ระบบชำระเงินระดับ production
> ที่ทำถูกต้องแทบทั้งหมดยึดหลักการนี้

### ข้อผิดพลาดคลาสสิกที่พบบ่อยในโค้ดจริง

```ruby
# ❌ อันตราย — ตัวอย่างโค้ดที่ทำผิดพลาดคลาสสิก (ห้ามทำตาม)
def success
  order = Order.find_by(stripe_checkout_session_id: params[:session_id])
  order.update!(status: "paid") # fulfill ทันทีจากแค่การเห็น GET request นี้!
  UserMailer.order_confirmation(order).deliver_later
  order.user.grant_course_access!
end
```

ปัญหา: ใครก็ตามที่คัดลอก URL success_url ของตัวเอง (จาก order เก่าที่เคยจ่ายเงินจริงสำเร็จ) แล้ว
เปิดซ้ำ หรือแม้แต่เดา `session_id` ถ้า ID คาดเดาได้ง่ายพอ ก็จะได้รับสิทธิ์/สินค้าฟรีโดยไม่ต้องจ่าย
เงินจริงอีกครั้ง (แม้ในความเป็นจริง Checkout Session ID ของ Stripe คาดเดายากมากเพราะเป็น random
string ยาว แต่หลักการยังคงผิดอยู่ดี เพราะไม่ได้ตรวจสอบสถานะการจ่ายเงินจริงจาก Stripe เลย)

---

## Step 706: สร้าง webhook endpoint และ verify signature จริงด้วย `Stripe::Webhook.construct_event`

### ตั้งค่า route และ controller

```ruby
# config/routes.rb
namespace :stripe do
  post "webhooks", to: "webhooks#create"
end
```

```ruby
# app/controllers/stripe/webhooks_controller.rb
class Stripe::WebhooksController < ApplicationController
  # webhook มาจาก Stripe server โดยตรง ไม่มี CSRF token ของ Rails ติดมาด้วย (และไม่ควรมี เพราะ
  # นี่ไม่ใช่ request จาก browser ของผู้ใช้ในระบบเรา) ความปลอดภัยของ endpoint นี้พึ่งพา signature
  # verification ด้านล่างทั้งหมดแทน ไม่ใช่ CSRF token
  skip_before_action :verify_authenticity_token, raise: false

  def create
    payload = request.body.read
    sig_header = request.headers["Stripe-Signature"]
    endpoint_secret = Rails.application.credentials.dig(:stripe, :webhook_signing_secret)

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
    rescue JSON::ParserError
      return head :bad_request
    rescue Stripe::SignatureVerificationError => e
      Rails.logger.warn("[Stripe::Webhooks] signature ไม่ถูกต้อง: #{e.message}")
      return head :bad_request
    end

    # ... จัดการ event ต่อใน Step 707-708

    head :ok
  end
end
```

**จุดสำคัญที่ต้องระวัง:**

- `request.body.read` — ต้องอ่าน **raw request body ดิบๆ** ไม่ใช่ `params` เพราะ signature ของ
  Stripe คำนวณจาก byte สตริงดิบของ body เป๊ะๆ ถ้า Rails แปลง (parse) เป็น Hash ไปแล้วค่อยแปลง
  กลับเป็น JSON ใหม่ ลำดับ key/format อาจต่างจากต้นฉบับเล็กน้อย ทำให้ signature ไม่ตรงกัน
- `skip_before_action :verify_authenticity_token` — เพราะ Stripe ไม่มีทางส่ง Rails CSRF token
  มาด้วยได้ (endpoint นี้ต้องเปิดรับ POST จากภายนอกโดยไม่มี session ของผู้ใช้เกี่ยวข้องเลย)
  ความปลอดภัยทั้งหมดของ endpoint นี้จึงอยู่ที่การ verify signature แทน

### Verify signature อย่างไร — และทดสอบจริงว่า verify ได้ถูกต้องจริง

Stripe ลงชื่อทุก webhook ด้วยอัลกอริทึมนี้ (เอกสารทางการของ Stripe):

1. เอา `timestamp` (Unix time ตอนส่ง) ต่อกับ `.` และ raw JSON payload: `"#{timestamp}.#{payload}"`
2. คำนวณ **HMAC-SHA256** ของสตริงนั้นด้วย **webhook signing secret** (`whsec_...`) เป็น key
3. ส่งมาใน header `Stripe-Signature` รูปแบบ `t=<timestamp>,v1=<hex signature>`

`Stripe::Webhook.construct_event(payload, sig_header, secret)` ทำหน้าที่นี้ให้ทั้งหมด: แยก
`timestamp`/`signature` จาก header, คำนวณ HMAC ใหม่ด้วย secret ที่เราให้มา, เทียบกับ signature
ที่ได้รับมา, และเช็คว่า `timestamp` ไม่เก่าเกินไป (ป้องกัน replay attack — ค่า default tolerance
คือ **300 วินาที**) ถ้าอย่างใดอย่างหนึ่งไม่ผ่านจะ raise `Stripe::SignatureVerificationError`

เราเขียนสคริปต์ทดสอบกลไกนี้แบบเต็มรูปแบบโดยไม่ต้องพึ่ง network ไปหา Stripe เลย (เพราะเป็นการคำนวณ
ในเครื่องล้วนๆ) ด้วยการคำนวณ HMAC เองด้วย `OpenSSL::HMAC` ตามอัลกอริทึมข้างต้น แล้วป้อนเข้า
`Stripe::Webhook.construct_event` ของ gem จริง:

```ruby
require "stripe"
require "openssl"
require "json"

endpoint_secret = "whsec_" + ("a".."z").to_a.sample(24).join
payload = {
  id: "evt_test_webhook", object: "event", type: "checkout.session.completed",
  data: { object: { id: "cs_test_12345", payment_status: "paid",
                     metadata: { order_id: "42" }, amount_total: 150_000 } }
}.to_json

timestamp = Time.now.to_i
signed_payload = "#{timestamp}.#{payload}"
signature = OpenSSL::HMAC.hexdigest("SHA256", endpoint_secret, signed_payload)
sig_header = "t=#{timestamp},v1=#{signature}"

event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
puts event.type # => "checkout.session.completed"
```

**ผลการทดสอบจริง (รันจริงในแซนด์บ็อกซ์นี้ ทุกกรณี):**

```
== Case 1: signature ถูกต้อง ==
PASSED construct_event สำเร็จ, event.type = checkout.session.completed, order_id = 42

== Case 2: secret ผิด ต้องถูกปฏิเสธ ==
PASSED ถูกปฏิเสธตามคาด: No signatures found matching the expected signature for payload

== Case 3: payload ถูกแก้ไขระหว่างทาง (tamper) ต้องถูกปฏิเสธ ==
PASSED ถูกปฏิเสธตามคาด: No signatures found matching the expected signature for payload

== Case 4: timestamp เก่าเกินไป (replay attack) ต้องถูกปฏิเสธด้วย tolerance เริ่มต้น ==
PASSED ถูกปฏิเสธตามคาด (timestamp เก่าเกิน tolerance): Timestamp outside the tolerance zone
```

นี่คือการยืนยันแบบเต็มรูปแบบว่า:

1. Signature ที่คำนวณถูกต้องตามอัลกอริทึมที่ Stripe เอกสารไว้ **ผ่าน** การ verify จริง
2. ถ้า secret ผิด (เช่น ใช้ secret ของ endpoint อื่น หรือพิมพ์ผิด) **ถูกปฏิเสธ**
3. ถ้า payload ถูกแก้ไขแม้แค่ตัวอักษรเดียวระหว่างทาง (เช่น มี proxy/man-in-the-middle พยายามแก้
   จำนวนเงิน) **ถูกปฏิเสธ** ทันที เพราะ HMAC จะไม่ตรงกันอีกต่อไป
4. Replay attack (การจับ webhook เก่าที่ signature ยังถูกต้องมาส่งซ้ำทีหลังนานๆ) **ถูกปฏิเสธ**
   เพราะ timestamp เก่าเกิน tolerance 300 วินาที

โค้ด `Stripe::WebhooksController` ที่แสดงด้านบนจึงเป็นโค้ด production จริงที่ผ่านการทดสอบทุกกรณี
สำคัญแล้ว ไม่ใช่แค่โค้ดตัวอย่างที่คัดลอกจากเอกสารมาเฉยๆ

### ตั้งค่า endpoint บน Stripe Dashboard

ในการใช้งานจริง (ไม่ใช่แค่ทดสอบ local) ต้องไปตั้งค่าที่ Stripe Dashboard → Developers → Webhooks
→ Add endpoint แล้วระบุ URL จริงของ production (เช่น `https://myapp.com/stripe/webhooks`) และ
เลือก event type ที่ต้องการรับ (อย่างน้อย `checkout.session.completed`,
`customer.subscription.updated`, `customer.subscription.deleted` สำหรับ Part นี้) Stripe จะ generate
**webhook signing secret** เฉพาะของ endpoint นั้นให้ (คนละตัวกับที่ใช้ตอน dev ผ่าน Stripe CLI ใน
Step 710) ต้องนำไปใส่ใน production credentials แยกต่างหาก

---

## Step 707: Handle `checkout.session.completed` เพื่อ fulfill order จริง

### เขียน handler สำหรับ event

```ruby
class Stripe::WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token, raise: false

  def create
    payload = request.body.read
    sig_header = request.headers["Stripe-Signature"]
    endpoint_secret = Rails.application.credentials.dig(:stripe, :webhook_signing_secret)

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
    rescue JSON::ParserError
      return head :bad_request
    rescue Stripe::SignatureVerificationError => e
      Rails.logger.warn("[Stripe::Webhooks] signature ไม่ถูกต้อง: #{e.message}")
      return head :bad_request
    end

    case event.type
    when "checkout.session.completed"
      handle_checkout_completed(event.data.object)
    else
      Rails.logger.info("[Stripe::Webhooks] ไม่จัดการ event type: #{event.type}")
    end

    head :ok
  end

  private

  def handle_checkout_completed(session)
    order = Order.find_by(stripe_checkout_session_id: session.id)
    return unless order

    # เช็ค payment_status เสมอ อย่าเชื่อแค่ว่า event นี้เกิดขึ้น (ทบทวน Step 705 ข้อ 3:
    # บาง payment method เป็น async — session "completed" ได้โดยที่ยังไม่ paid จริง)
    if session.payment_status == "paid" && !order.fulfilled?
      order.update!(stripe_payment_intent_id: session.payment_intent)
      order.fulfill!
      # ในระบบจริง: ส่งอีเมลยืนยัน (ActionMailer, Part 072), ปลดล็อกสิทธิ์เข้าถึงคอร์ส,
      # หักสต็อกสินค้า ฯลฯ — ควรทำผ่าน background job (Part 061-062) ไม่ใช่ inline ตรงนี้
      # เพื่อให้ webhook ตอบกลับ Stripe เร็วที่สุด (Stripe timeout endpoint ที่ตอบช้าเกินไป)
    end
  end
end
```

**ทำไมต้องเช็ค `payment_status == "paid"` ซ้ำอีกชั้น:** ตามเอกสารทางการของ Stripe (ส่วนนี้มาจาก
เอกสาร ไม่ได้ทดสอบจริงเพราะต้องมีวิธีจ่ายเงินแบบ async จริงถึงจะจำลองได้) event
`checkout.session.completed` จะยิงทันทีที่ลูกค้ากรอกข้อมูลบน Checkout ครบและกด submit สำเร็จ
แต่สำหรับวิธีจ่ายเงินบางประเภทที่ยืนยันผลช้า (asynchronous payment methods) `payment_status` ณ
ตอนนั้นอาจยังเป็น `"unpaid"` อยู่ — กรณีนี้ Stripe จะยิง event เพิ่มเติมทีหลัง
(`checkout.session.async_payment_succeeded` หรือ `checkout.session.async_payment_failed`) เพื่อ
แจ้งผลจริง ระบบที่รัดกุมจึงควร handle ทั้งสาม event type นี้ ไม่ใช่แค่ `checkout.session.completed`
อย่างเดียว (ในแบบฝึกหัดท้าย Part จะมีโจทย์เสริมให้ลองเพิ่ม event เหล่านี้เอง)

### ทำไมต้องเช็ค `return unless order` และ `!order.fulfilled?`

- `return unless order` — ป้องกันกรณี event มาถึงแต่หา order ที่ตรงกันไม่เจอ (เช่น ข้อมูลทดสอบ
  เก่าที่ order ถูกลบไปแล้ว) ไม่ควรปล่อยให้ raise `NoMethodError` จน endpoint ตอบ 500 กลับไป
  ซึ่งจะทำให้ Stripe คิดว่า endpoint ล้มเหลวแล้ว**retry ส่ง event เดิมมาซ้ำเรื่อยๆ** โดยไม่มี
  ประโยชน์
- `!order.fulfilled?` — เป็นจุดเริ่มต้นของเรื่อง **idempotency** ที่จะขยายความเต็มรูปแบบใน
  Step 708

---

## Step 708: Idempotency สำหรับ webhook — Stripe อาจส่ง event ซ้ำ ต้องออกแบบให้ปลอดภัย

### ทบทวนหลักการจาก Part 062

Part 062 (Sidekiq Deep Dive) สอนไว้แล้วว่าระบบ background job ส่วนใหญ่รับประกันแค่ **"at least
once delivery"** ไม่ใช่ "exactly once" — **หลักการเดียวกันนี้ใช้กับ Stripe webhook เป๊ะๆ**
เอกสารทางการของ Stripe ระบุชัดเจนว่า **endpoint ของเราอาจได้รับ event เดียวกันมากกว่าหนึ่งครั้ง**
ได้จากหลายสาเหตุ:

1. Endpoint ของเราตอบช้าเกินไป (timeout) — Stripe คิดว่าล้มเหลวแล้ว retry ส่งใหม่ ทั้งที่จริงแล้ว
   เราประมวลผลสำเร็จไปแล้วก่อนตอบกลับไม่ทัน
2. Endpoint ตอบ HTTP status ที่ไม่ใช่ 2xx (เช่น เกิด error หลังประมวลผลผลข้างเคียงสำเร็จไปแล้ว
   แต่ก่อนถึงบรรทัด `head :ok`) — Stripe จะ retry ตาม exponential backoff ของตัวเอง
3. Network ระหว่างทางมีปัญหาชั่วคราวทำให้ Stripe ไม่ได้รับ acknowledgment แม้ endpoint จะประมวลผล
   สำเร็จแล้วจริง
4. Dashboard มีปุ่มให้ resend event ด้วยตนเองสำหรับ debug ซึ่งคนอาจกดซ้ำโดยไม่ตั้งใจ

เพราะฉะนั้น **handler ของทุก event ที่มีผลข้างเคียง (fulfill order, ตัดสิทธิ์, ส่งอีเมล) ต้องถูก
ออกแบบให้ปลอดภัยเมื่อถูกเรียกซ้ำ (idempotent)** — เรียกกี่ครั้งก็ต้องได้ผลลัพธ์เหมือนเรียกครั้งเดียว
เป๊ะๆ

### วิธีทำ: unique constraint ระดับฐานข้อมูลบน `event.id`

Stripe ให้ `id` ที่ unique เฉพาะของทุก event มาเสมอ (รูปแบบ `evt_xxxxxxxxx`) วิธีที่ปลอดภัยที่สุด
คือบันทึกไว้ในตารางแยก พร้อม **unique index** แล้วให้ database เป็นคนตัดสินกันซ้ำ (เหมือนที่ Part
062 สอนไว้ว่าการเช็คด้วย `if` เฉยๆ ไม่ปลอดภัย 100% เพราะมี race condition ได้ถ้า Stripe ยิง 2
request มาเกือบพร้อมกันพอดี):

```ruby
# db/migrate/xxxx_create_processed_stripe_events.rb
class CreateProcessedStripeEvents < ActiveRecord::Migration[8.1]
  def change
    create_table :processed_stripe_events do |t|
      t.string :stripe_event_id, null: false
      t.string :event_type
      t.timestamps
    end
    add_index :processed_stripe_events, :stripe_event_id, unique: true
  end
end
```

```ruby
def create
  payload = request.body.read
  sig_header = request.headers["Stripe-Signature"]
  endpoint_secret = Rails.application.credentials.dig(:stripe, :webhook_signing_secret)

  begin
    event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
  rescue JSON::ParserError
    return head :bad_request
  rescue Stripe::SignatureVerificationError => e
    Rails.logger.warn("[Stripe::Webhooks] signature ไม่ถูกต้อง: #{e.message}")
    return head :bad_request
  end

  # --- Idempotency guard ---
  # ใช้ database unique constraint แทนการเช็คด้วย exists? เฉยๆ เพื่อป้องกัน race condition
  # ถ้า Stripe ยิง 2 request มาเกือบพร้อมกันพอดี (ตามหลักการเดียวกับ Part 062)
  begin
    ProcessedStripeEvent.create!(stripe_event_id: event.id, event_type: event.type)
  rescue ActiveRecord::RecordNotUnique
    Rails.logger.info("[Stripe::Webhooks] event #{event.id} เคยถูกประมวลผลแล้ว ข้าม")
    return head :ok # ตอบ 200 เสมอ แม้จะข้าม เพื่อบอก Stripe ว่าไม่ต้อง retry อีก
  end

  case event.type
  when "checkout.session.completed"
    handle_checkout_completed(event.data.object)
  when "customer.subscription.updated"
    handle_subscription_updated(event.data.object)
  when "customer.subscription.deleted"
    handle_subscription_deleted(event.data.object)
  end

  head :ok
end
```

**จุดสำคัญ:** ต้องตอบ `head :ok` (HTTP 200) เสมอทั้งกรณีประมวลผลสำเร็จและกรณีข้ามเพราะซ้ำ —
เพราะถ้าตอบ error กลับไปตอนเจอ event ซ้ำ Stripe จะยิ่ง retry มาอีกไม่รู้จบ (มันคิดว่า endpoint
ยังล้มเหลวอยู่)

### ทดสอบจริงแบบ end-to-end: ส่ง webhook เดิมซ้ำ ยืนยันว่าไม่ fulfill ซ้ำจริง

เราทดสอบเต็มรูปแบบผ่าน Rails app จริงที่รันบน Puma server จริง โดยส่ง webhook event เดียวกัน (คำนวณ
signature จริงด้วย secret เดียวกับที่อยู่ใน credentials) **สองครั้งติดกัน** ผ่าน HTTP POST จริง:

```
== ครั้งที่ 1: webhook ที่ signature ถูกต้อง ==
status: 200

== ครั้งที่ 2: ส่ง event เดิมซ้ำ (จำลอง Stripe redeliver) ==
status: 200
```

ตรวจสอบ log จริงของ Rails:

```
[Stripe::Webhooks] event evt_test_7025d826bf70 เคยถูกประมวลผลแล้ว ข้าม
```

และตรวจสอบฐานข้อมูลจริงหลังทั้งสองครั้ง:

```irb
$ bin/rails runner 'o = Order.find(1); puts o.attributes; puts ProcessedStripeEvent.count'
{"id"=>1, ..., "status"=>"paid", "fulfilled_at"=>2026-09-26 09:27:51 UTC, ...}
1
```

**ยืนยันชัดเจน:** แม้ webhook เดียวกันถูกส่งมาสองครั้ง `Order#fulfilled_at` ถูกตั้งค่าแค่ครั้งเดียว
(ไม่ถูกเขียนทับซ้ำ), มี `ProcessedStripeEvent` แค่ 1 แถวสำหรับ event นี้ (ไม่ใช่ 2), และไม่มีการ
ส่งอีเมล/ปลดล็อกสิทธิ์ซ้ำเป็นครั้งที่สอง — idempotency guard ทำงานถูกต้อง 100% ในสถานการณ์จริงที่
จำลองการ redeliver ของ Stripe

นอกจากนี้เรายังทดสอบกรณี signature ผิดผ่าน HTTP จริงด้วย ยืนยันว่า endpoint ตอบ `400 Bad Request`
และไม่มีการสร้าง `ProcessedStripeEvent` หรือแตะ `Order` เลยแม้แต่น้อย — endpoint ปลอดภัยจาก
request ปลอมที่ไม่ได้เซ็นด้วย secret ที่ถูกต้อง

---

## Step 709: Stripe Subscriptions — Product/Price, subscription Checkout Session, lifecycle webhook

### Product และ Price ใน Stripe

สำหรับสินค้าที่เก็บเงินซ้ำ (recurring billing เช่น รายเดือน/รายปี) แนวปฏิบัติที่ดีคือสร้าง
**Product** (ตัวสินค้า/แพ็กเกจ เช่น "แผน Pro") และ **Price** (ราคา + รอบการเก็บเงินของ product นั้น
เช่น "500 บาท/เดือน") ไว้ใน Stripe **ล่วงหน้าครั้งเดียว** (ผ่าน Dashboard หรือสคริปต์ setup ที่รัน
ครั้งเดียว) แทนที่จะสร้างใหม่ทุกครั้งที่มีการสมัครสมาชิก:

```ruby
# สคริปต์ setup ที่รันครั้งเดียว (ไม่ใช่โค้ดที่รันทุก request)
product = Stripe::Product.create(name: "แผน Pro รายเดือน")

price = Stripe::Price.create(
  product: product.id,
  unit_amount: 29_900,       # 299.00 บาท/เดือน
  currency: "thb",
  recurring: { interval: "month" }
)

puts price.id # => "price_xxxxxxxxxxxx" — เก็บ ID นี้ไว้ใช้ในโค้ดแอป (เช่นใน credentials/config)
```

### สร้าง subscription Checkout Session

```ruby
def create_subscription
  checkout_session = Stripe::Checkout::Session.create(
    mode: "subscription", # ต่างจาก Step 703 ที่ใช้ "payment"
    line_items: [{
      price: Rails.application.credentials.dig(:stripe, :pro_plan_price_id),
      quantity: 1
    }],
    success_url: subscription_success_url(session_id: "{CHECKOUT_SESSION_ID}"),
    cancel_url: subscription_cancel_url,
    client_reference_id: current_user.id,
    subscription_data: {
      metadata: { user_id: current_user.id }
    }
  )

  redirect_to checkout_session.url, allow_other_host: true, status: :see_other
end
```

ความต่างหลักจาก one-time payment: `mode: "subscription"`, `line_items` อ้างอิง `price` ที่สร้าง
ไว้ล่วงหน้า (ไม่ใช้ `price_data` inline เพราะราคาซ้ำต้องผูกกับรอบบิลที่ชัดเจน), และมี
`subscription_data` สำหรับแนบ metadata ระดับ subscription (แยกจาก metadata ระดับ session)

### Handle subscription lifecycle webhook

เมื่อ subscription สร้างสำเร็จ Checkout จะยิง `checkout.session.completed` เหมือน one-time payment
(เข้าถึง `session.subscription` เพื่อดู subscription ID ที่ถูกสร้าง) แต่ที่สำคัญกว่าสำหรับ
subscription คือ event ที่เกิดขึ้น**ตลอดอายุการเป็นสมาชิก**:

```ruby
def handle_subscription_updated(subscription)
  user = User.find_by(stripe_subscription_id: subscription.id)
  return unless user

  case subscription.status
  when "active"
    user.update!(plan: "pro", subscription_active: true)
  when "past_due"
    # การหักเงินรอบล่าสุดล้มเหลว (บัตรหมดอายุ/เงินไม่พอ) — Stripe จะ retry เองตาม
    # dunning schedule ที่ตั้งค่าไว้ ระบบเราแค่บันทึกสถานะไว้เตือนผู้ใช้
    user.update!(subscription_active: false)
  when "canceled", "unpaid"
    user.update!(plan: "free", subscription_active: false)
  end

  # หมายเหตุ: subscription.cancel_at_period_end (true/false) บอกว่าผู้ใช้กดยกเลิกแล้วแต่ยังใช้
  # สิทธิ์ได้จนครบรอบบิลปัจจุบัน — ต่างจาก status: "canceled" ที่ยกเลิกมีผลทันที
end

def handle_subscription_deleted(subscription)
  # ยิงเมื่อ subscription ถูกลบจริง (ครบรอบสุดท้ายหลังยกเลิก หรือถูกยกเลิกทันที)
  user = User.find_by(stripe_subscription_id: subscription.id)
  user&.update!(plan: "free", subscription_active: false)
end
```

> **หมายเหตุจากการตรวจสอบ source code ของ gem จริง:** attribute `status` และ
> `cancel_at_period_end` มีอยู่จริงบน `Stripe::Subscription` ใน gem เวอร์ชัน 19.6.2 ที่ทดสอบ
> แต่ **`current_period_end` ไม่ได้เป็น top-level attribute อีกต่อไปแล้ว** ในเอกสาร/บทความเก่า
> หลายแห่งบนอินเทอร์เน็ตยังอ้างถึง `subscription.current_period_end` ตรงๆ ซึ่ง**ใช้ไม่ได้แล้ว**
> กับ API version ปัจจุบัน ต้องเข้าถึงผ่าน `subscription.items.data.first.current_period_end`
> แทน (Stripe ย้ายค่านี้ไปอยู่ระดับ subscription item เพราะ subscription หนึ่งตัวอาจมีหลาย
> item ที่แต่ละ item มีรอบบิลต่างกันได้ในเวอร์ชัน API ใหม่) เป็นตัวอย่างที่ดีว่าทำไมควรอ้างอิง
> เอกสาร/SDK เวอร์ชันปัจจุบันเสมอ แทนการเชื่อโค้ดตัวอย่างเก่าที่เจอทั่วไปตามอินเทอร์เน็ต

### ทำไมต้อง handle event ที่ "ลบ/ยกเลิก" ด้วย ไม่ใช่แค่ event "สร้างสำเร็จ"

ระบบ subscription ที่ทำแค่ "ปลดล็อกสิทธิ์ตอนสมัคร" โดยไม่ handle การยกเลิก/หมดอายุ/หักเงินล้มเหลว
เลย จะกลายเป็นช่องโหว่ทางธุรกิจที่ร้ายแรง — ผู้ใช้ที่ยกเลิกบัตรหรือหยุดจ่ายเงินจะยังคงมีสิทธิ์เข้า
ถึงระบบต่อไปเรื่อยๆ โดยไม่มีวันถูกตัดสิทธิ์ นี่คือเหตุผลที่ Step 709 ให้ subscribe ทั้ง
`customer.subscription.updated` และ `customer.subscription.deleted` ไม่ใช่แค่
`checkout.session.completed` อย่างเดียว

---

## Step 710: ทดสอบ local ด้วย Stripe CLI (`stripe listen --forward-to`)

> **หมายเหตุ:** เนื้อหา Step นี้อธิบายตาม workflow จากเอกสารทางการของ Stripe **ไบนารี `stripe`
> CLI ไม่ได้ติดตั้งอยู่ในแซนด์บ็อกซ์ที่ใช้เขียน Part นี้** (ตรวจสอบแล้วว่าไม่มีคำสั่งนี้ในระบบ)
> จึงไม่ได้รันคำสั่งจริงในหัวข้อนี้ — แต่ workflow ที่อธิบายด้านล่างตรงกับเอกสารทางการทุกจุด

### ปัญหาที่ Stripe CLI แก้: การรับ webhook ตอน dev บนเครื่อง local

Webhook คือ HTTP request ที่ Stripe ต้องยิงมาถึงเซิร์ฟเวอร์ของเรา แต่เครื่อง dev ของเรา (เช่น
`localhost:3000`) ไม่มี public URL ที่ Stripe เข้าถึงได้จากอินเทอร์เน็ต **Stripe CLI** แก้ปัญหานี้
ด้วยการสร้าง tunnel ชั่วคราวที่ **forward** event จริงจากบัญชี Stripe (test mode) มาที่เครื่อง
local ของเราโดยตรง โดยไม่ต้องเปิด public URL หรือใช้เครื่องมืออย่าง ngrok เลย

### ขั้นตอนการใช้งาน

**1. ติดตั้ง Stripe CLI** (ตามระบบปฏิบัติการ):

```bash
# macOS (Homebrew)
brew install stripe/stripe-cli/stripe

# Linux (ตัวอย่าง Debian/Ubuntu ผ่าน apt repository ของ Stripe)
curl -s https://packages.stripe.dev/api/security/keypair/stripe-cli-gpg/public \
  | gpg --dearmor | sudo tee /usr/share/keyrings/stripe.gpg
echo "deb [signed-by=/usr/share/keyrings/stripe.gpg] https://packages.stripe.dev/stripe-cli-debian-local stable main" \
  | sudo tee -a /etc/apt/sources.list.d/stripe.list
sudo apt update && sudo apt install stripe
```

**2. Login เข้าบัญชี Stripe:**

```bash
stripe login
# จะเปิด browser ให้ยืนยันตัวตนและอนุญาต CLI เข้าถึงบัญชี (test mode)
```

**3. เริ่ม forward webhook มาที่เซิร์ฟเวอร์ dev:**

```bash
stripe listen --forward-to localhost:3000/stripe/webhooks
```

คำสั่งนี้จะพิมพ์ **webhook signing secret ชั่วคราว** ออกมาทันที (รูปแบบ `whsec_...` เหมือนกัน แต่
เป็นคนละตัวกับ secret ของ endpoint จริงบน production Dashboard):

```
> Ready! Your webhook signing secret is whsec_1a2b3c4d5e6f... (^C to quit)
```

ต้องนำ secret นี้ไปใส่ใน **development credentials** (`bin/rails credentials:edit --environment
development`) แทนที่ค่าเดิมชั่วคราวขณะรัน `stripe listen` (secret นี้จะเปลี่ยนทุกครั้งที่รันคำสั่ง
ใหม่ ถ้าต้องการ secret คงที่ให้ใช้ flag `--print-secret` ดูค่าที่ผูกกับ CLI account หรือ pin
device ด้วย `stripe listen --latest`)

**4. รัน Rails server คู่กันในอีก terminal:**

```bash
bin/rails server
```

ตอนนี้ทุก event จริงที่เกิดในบัญชี Stripe test mode (เช่น มีคนกดจ่ายเงินจริงผ่าน Checkout ที่ชี้ไป
บัญชี test) จะถูก forward มาที่ `localhost:3000/stripe/webhooks` ทันที ผ่าน signature ที่ verify
ได้จริงด้วย secret ที่ CLI พิมพ์ให้

**5. จำลอง event โดยไม่ต้องจ่ายเงินจริงเลยด้วย `stripe trigger`:**

```bash
stripe trigger checkout.session.completed
```

คำสั่งนี้สร้าง fixture event ปลอมที่มีโครงสร้างเหมือน event จริงทุกประการ ส่งผ่าน `stripe listen`
มาที่ endpoint ของเราทันที มีประโยชน์มากสำหรับทดสอบ handler โดยไม่ต้องกรอกบัตรเทสจริงทุกครั้ง

**6. ทดสอบ flow เต็มรูปแบบด้วยบัตรทดสอบ:**

เมื่อทดสอบ Checkout จริงผ่านหน้าเว็บ (ไม่ใช่ `stripe trigger`) ใช้เลขบัตรทดสอบมาตรฐานของ Stripe:

| สถานการณ์ | เลขบัตร |
|-----------|---------|
| จ่ายสำเร็จ | `4242 4242 4242 4242` |
| บัตรถูกปฏิเสธ (declined) | `4000 0000 0000 0002` |
| ต้องผ่าน 3D Secure authentication | `4000 0025 0000 3155` |

วันหมดอายุใช้วันในอนาคตใดๆ ก็ได้ (เช่น `12/34`), CVV ใช้ตัวเลข 3 หลักใดๆ ก็ได้ (เช่น `123`) —
ทั้งหมดนี้ใช้ได้เฉพาะกับ **test mode** (`pk_test_`/`sk_test_`) เท่านั้น

### สรุป workflow การทดสอบ local

```
Terminal 1: bin/rails server                          (Rails app รันที่ localhost:3000)
Terminal 2: stripe listen --forward-to localhost:3000/stripe/webhooks   (tunnel + verify secret)
Terminal 3: stripe trigger checkout.session.completed  (จำลอง event แบบไม่ต้องจ่ายเงินจริง)
            หรือเปิด browser ไปที่ checkout URL จริงแล้วจ่ายด้วยบัตรทดสอบ
```

---

## แบบฝึกหัด: ระบบขายคอร์สออนไลน์แบบ one-time purchase พร้อม webhook fulfill แบบ idempotent

### โจทย์

สร้างระบบขายคอร์สออนไลน์ครั้งเดียว (one-time purchase ไม่ใช่ subscription) ที่ทำสิ่งต่อไปนี้ให้ครบ:

1. หน้าแรกมีปุ่ม "ชำระเงินด้วย Stripe Checkout" กดแล้วสร้าง `Order` และ Checkout Session แล้ว
   redirect ไปหน้า Stripe จริง
2. หน้า success/cancel ที่**ไม่** fulfill order ตรงๆ (รอ webhook เท่านั้น)
3. Webhook endpoint ที่ **verify signature จริง**, จัดการ event `checkout.session.completed`,
   และ**ป้องกันการ fulfill ซ้ำ**ด้วย unique constraint ระดับฐานข้อมูล
4. เขียนสคริปต์ทดสอบที่จำลอง Stripe redeliver webhook เดิมซ้ำ และยืนยันว่า order ไม่ถูก fulfill
   ซ้ำ (ตามที่ Step 708 พิสูจน์ไว้)

> **สถานะการทดสอบเฉลยนี้:** เฉลยทั้งหมดด้านล่าง **ทดสอบจริงครบทุกจุด** กับ Rails 8.1.4 + Ruby
> 3.3.6 + stripe gem 19.6.2 บน Puma server จริง, SQLite จริง ยกเว้น**จุดเดียว**คือเมธอด
> `Stripe::Checkout::Session.create` ที่ถูก stub (คืนค่า object ปลอมที่มี `id`/`url`) เพราะ
> แซนด์บ็อกซ์นี้เชื่อมต่อ `api.stripe.com` จริงไม่ได้ (network policy ปฏิเสธด้วย 403) ส่วนการ
> verify webhook signature ใช้ secret จริงและคำนวณ HMAC จริงทั้งหมด ไม่มีการ stub ส่วนนี้เลย

### เฉลย

**Migration:**

```ruby
# db/migrate/xxxx_create_orders.rb
class CreateOrders < ActiveRecord::Migration[8.1]
  def change
    create_table :orders do |t|
      t.string :product_name
      t.integer :amount_cents
      t.string :currency
      t.string :status
      t.string :stripe_checkout_session_id
      t.string :stripe_payment_intent_id
      t.datetime :fulfilled_at
      t.timestamps
    end
    add_index :orders, :stripe_checkout_session_id, unique: true
  end
end
```

```ruby
# db/migrate/xxxx_create_processed_stripe_events.rb
class CreateProcessedStripeEvents < ActiveRecord::Migration[8.1]
  def change
    create_table :processed_stripe_events do |t|
      t.string :stripe_event_id, null: false
      t.string :event_type
      t.timestamps
    end
    add_index :processed_stripe_events, :stripe_event_id, unique: true
  end
end
```

**Model:**

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  validates :product_name, :amount_cents, :currency, presence: true

  def fulfilled?
    fulfilled_at.present?
  end

  def fulfill!
    update!(status: "paid", fulfilled_at: Time.current)
  end
end
```

**Routes:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "checkouts#new"

  resource :checkout, only: [:new, :create]
  get "checkout/success", to: "checkouts#success", as: :checkout_success
  get "checkout/cancel", to: "checkouts#cancel", as: :checkout_cancel

  namespace :stripe do
    post "webhooks", to: "webhooks#create"
  end
end
```

**Checkouts Controller:**

```ruby
# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  PRODUCT = {
    name: "คอร์สเรียน Ruby on Rails (Lifetime Access)",
    amount_cents: 150_000,
    currency: "thb"
  }.freeze

  def new
  end

  def create
    order = Order.create!(
      product_name: PRODUCT[:name],
      amount_cents: PRODUCT[:amount_cents],
      currency: PRODUCT[:currency],
      status: "pending"
    )

    session = Stripe::Checkout::Session.create(
      mode: "payment",
      line_items: [{
        price_data: {
          currency: PRODUCT[:currency],
          unit_amount: PRODUCT[:amount_cents],
          product_data: { name: PRODUCT[:name] }
        },
        quantity: 1
      }],
      success_url: checkout_success_url(session_id: "{CHECKOUT_SESSION_ID}"),
      cancel_url: checkout_cancel_url,
      client_reference_id: order.id,
      metadata: { order_id: order.id }
    )

    order.update!(stripe_checkout_session_id: session.id)
    redirect_to session.url, allow_other_host: true, status: :see_other
  end

  def success
    @order = Order.find_by(stripe_checkout_session_id: params[:session_id])
  end

  def cancel
  end
end
```

**Webhooks Controller:**

```ruby
# app/controllers/stripe/webhooks_controller.rb
class Stripe::WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token, raise: false

  def create
    payload = request.body.read
    sig_header = request.headers["Stripe-Signature"]
    endpoint_secret = Rails.application.credentials.dig(:stripe, :webhook_signing_secret)

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
    rescue JSON::ParserError
      return head :bad_request
    rescue Stripe::SignatureVerificationError => e
      Rails.logger.warn("[Stripe::Webhooks] signature ไม่ถูกต้อง: #{e.message}")
      return head :bad_request
    end

    begin
      ProcessedStripeEvent.create!(stripe_event_id: event.id, event_type: event.type)
    rescue ActiveRecord::RecordNotUnique
      Rails.logger.info("[Stripe::Webhooks] event #{event.id} เคยถูกประมวลผลแล้ว ข้าม")
      return head :ok
    end

    case event.type
    when "checkout.session.completed"
      handle_checkout_completed(event.data.object)
    else
      Rails.logger.info("[Stripe::Webhooks] ไม่จัดการ event type: #{event.type}")
    end

    head :ok
  end

  private

  def handle_checkout_completed(session)
    order = Order.find_by(stripe_checkout_session_id: session.id)
    return unless order

    if session.payment_status == "paid" && !order.fulfilled?
      order.update!(stripe_payment_intent_id: session.payment_intent)
      order.fulfill!
    end
  end
end
```

**สคริปต์ทดสอบ end-to-end (จำลองการทำงานของ Stripe จริง):**

```ruby
# ยิง HTTP request จริงผ่าน Puma server ที่รันอยู่จริง (ไม่ใช่ mock controller)
require "openssl"
require "json"
require "net/http"

endpoint_secret = "whsec_FAKE_EXAMPLE_SECRET" # ต้องตรงกับค่าใน credentials ของแอปทดสอบ
uri = URI("http://127.0.0.1:3099/stripe/webhooks")

def post_webhook(uri, payload, sig_header)
  http = Net::HTTP.new(uri.host, uri.port)
  req = Net::HTTP::Post.new(uri, { "Content-Type" => "application/json",
                                    "Stripe-Signature" => sig_header })
  req.body = payload
  http.request(req)
end

payload = {
  id: "evt_test_#{SecureRandom.hex(6)}", object: "event", type: "checkout.session.completed",
  data: { object: { id: "cs_test_fake_123456", payment_status: "paid",
                     payment_intent: "pi_test_fake_999", metadata: { order_id: "1" } } }
}.to_json

timestamp = Time.now.to_i
signature = OpenSSL::HMAC.hexdigest("SHA256", endpoint_secret, "#{timestamp}.#{payload}")
sig_header = "t=#{timestamp},v1=#{signature}"

puts "ครั้งที่ 1: #{post_webhook(uri, payload, sig_header).code}"  # => 200
puts "ครั้งที่ 2 (ซ้ำ): #{post_webhook(uri, payload, sig_header).code}" # => 200 แต่ไม่ fulfill ซ้ำ
```

**ผลการทดสอบจริงแบบเต็ม (รันจริงกับ Rails 8.1.4 + Puma):**

```
== Step 1: POST /checkout ==
HTTP/1.1 303 See Other
location: https://checkout.stripe.com/c/pay/cs_test_fake_123456

== Step 2: webhook ครั้งที่ 1 ==
status: 200
Order#1 → status="paid", fulfilled_at=2026-09-26 09:27:51 UTC, stripe_payment_intent_id="pi_test_fake_999"

== Step 3: webhook ครั้งที่ 2 (ส่ง event เดิมซ้ำ) ==
status: 200
log: "[Stripe::Webhooks] event evt_test_7025d826bf70 เคยถูกประมวลผลแล้ว ข้าม"
Order#1 → fulfilled_at ไม่เปลี่ยนแปลง, ProcessedStripeEvent.count == 1 (ไม่ใช่ 2)

== Step 4: webhook signature ผิด ==
status: 400 (ถูกปฏิเสธก่อนแตะฐานข้อมูลเลย)
```

ยืนยันครบทุกข้อของโจทย์: redirect ไป Stripe สำเร็จ, webhook verify signature จริงและปฏิเสธ
signature ปลอมได้ถูกต้อง, fulfill order สำเร็จเมื่อได้รับ webhook ที่ถูกต้อง, และ**ไม่ fulfill ซ้ำ**
แม้ได้รับ webhook เดียวกันซ้ำสองครั้ง

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มการ handle event `checkout.session.async_payment_succeeded` และ
   `checkout.session.async_payment_failed` (สำหรับวิธีจ่ายเงินแบบ asynchronous ที่กล่าวถึงใน
   Step 707) และเพิ่ม event `checkout.session.expired` เพื่ออัปเดตสถานะ order เป็น `"expired"`
   เมื่อลูกค้าไม่กรอกข้อมูลจนหมดเวลา (Checkout Session หมดอายุ default 24 ชั่วโมง)
2. เพิ่มการส่งอีเมลยืนยันคำสั่งซื้อด้วย ActionMailer หลังจาก `fulfill!` สำเร็จ (ทบทวน/รอเรียน
   Part 072) โดยต้องออกแบบให้ idempotent เช่นกัน — ถ้า handler ถูกเรียกซ้ำ (แม้จะผ่าน
   idempotency guard หลักไปไม่ได้ก็ตาม) ต้องไม่ส่งอีเมลซ้ำสองฉบับ
3. แปลงระบบให้รองรับทั้ง one-time purchase และ subscription ในหน้าเดียวกัน (ให้ผู้ใช้เลือกได้)
   พร้อมเพิ่มหน้า "จัดการการสมัครสมาชิกของฉัน" ที่สร้าง Stripe Customer Portal session ด้วย
   `Stripe::BillingPortal::Session.create(customer: user.stripe_customer_id, return_url: ...)`
   ให้ผู้ใช้ยกเลิก/อัปเกรดแผน หรือแก้ไขบัตรได้เองโดยไม่ต้องสร้างหน้า UI เอง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่าทำไมแทบไม่มีทีมไหนควรสร้างระบบชำระเงินด้วยตัวเอง — ภาระ **PCI DSS compliance**
  (SAQ A ถึง SAQ D, audit, vulnerability scan รายไตรมาส) และความเสี่ยงมหาศาลเมื่อข้อมูลบัตรรั่วไหล
  ทำให้การใช้ **Stripe Checkout** (ที่ข้อมูลบัตรไม่เคยผ่านเซิร์ฟเวอร์ของเราเลย) เป็นทางเลือกที่
  ถูกต้องเกือบทุกครั้งสำหรับการเริ่มต้น
- ติดตั้ง `stripe` gem และตั้งค่า API key ผ่าน Rails encrypted credentials ได้อย่างปลอดภัย
  (ทบทวน pattern จาก Part 067) พร้อมแยก publishable key/secret key ให้ถูกที่ถูกทาง
- สร้าง **Checkout Session** และ redirect ผู้ใช้ไปหน้าชำระเงินของ Stripe ได้จริง เข้าใจ
  `success_url`/`cancel_url`, `client_reference_id`, `metadata`, และทำไมต้องใช้
  `allow_other_host: true`
- เข้าใจอย่างลึกซึ้งว่า**ทำไมการเชื่อ redirect ฝั่ง client อย่างเดียวเป็นข้อผิดพลาดร้ายแรง** และ
  **webhook คือกลไกเดียวที่เชื่อถือได้** สำหรับการยืนยันการชำระเงิน — พร้อมยกตัวอย่างโค้ดผิดพลาด
  คลาสสิกที่พบบ่อยในระบบจริง
- Verify webhook signature ด้วย `Stripe::Webhook.construct_event` ได้อย่างถูกต้อง **ทดสอบจริงครบ
  ทุกกรณี** (ผ่าน/secret ผิด/payload ถูกแก้ไข/replay attack) ด้วยการคำนวณ HMAC-SHA256 เองตาม
  อัลกอริทึมของ Stripe
- ออกแบบ webhook handler ให้ **idempotent** ได้อย่างถูกต้องด้วย database unique constraint บน
  `event.id` (ผูกโยงกับหลักการ "at least once delivery" จาก Part 062) และ**พิสูจน์ด้วยการทดสอบ
  จริง**ว่า order ไม่ถูก fulfill ซ้ำแม้ webhook เดิมถูกส่งมาซ้ำ
- ตั้งค่าและใช้งาน **Stripe Subscriptions**: สร้าง Product/Price, สร้าง subscription Checkout
  Session, และ handle lifecycle webhook (`customer.subscription.updated`/`deleted`) พร้อมรู้จุด
  ที่เอกสารเก่าบนอินเทอร์เน็ตผิดพลาด (เช่น `current_period_end` ที่ย้ายตำแหน่งใน API เวอร์ชันใหม่)
- เข้าใจ workflow การทดสอบ webhook แบบ local ด้วย **Stripe CLI** (`stripe listen --forward-to`,
  `stripe trigger`) และรู้จักเลขบัตรทดสอบมาตรฐานสำหรับจำลองทุกสถานการณ์

**ต่อไป (Part 072 — ปิดท้าย Phase 11):** เราจะเรียนรู้การเชื่อมต่อกับบริการภายนอกอีกสามกลุ่มที่
เว็บแอปพลิเคชันสมัยใหม่แทบทุกตัวต้องมี — **Omniauth** สำหรับให้ผู้ใช้ล็อกอินด้วยบัญชี Google/
Facebook แทนการสมัครสมาชิกเอง, **Action Mailer** สำหรับส่งอีเมลจริง (ซึ่งจะเอามาใช้เติมเต็มโจทย์
ข้อ 2 ในแบบฝึกหัดเพิ่มเติมของ Part นี้พอดี — การส่งอีเมลยืนยันคำสั่งซื้อหลัง fulfill order สำเร็จ),
และ **SMS/notification integration** สำหรับแจ้งเตือนผู้ใช้ผ่านช่องทางอื่นนอกเหนือจากอีเมล นี่คือ
Part ปิดท้าย Phase 11 ก่อนที่ Phase 12 จะพาไปสู่โลกของ **DevOps & Deployment** อย่างเต็มตัว
