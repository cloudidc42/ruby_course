# Part 049: Mocking/Stubbing เชิงลึก, VCR สำหรับ External API, Test Coverage (SimpleCov)

> **Step ครอบคลุมใน Part นี้:** Step 481–490
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 019 เรื่อง RSpec doubles/`instance_double`/`allow`/
> `expect(...).to receive` มาก่อน และผ่าน Part 046–048 เรื่อง RSpec สำหรับ Rails,
> FactoryBot, Capybara มาก่อน — Part นี้จะไม่สอนพื้นฐาน double ซ้ำ แต่จะพาไปลึกกว่านั้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, gem `vcr` ~> 6.3 (ทดสอบจริงบน 6.4.0),
> `webmock` ~> 3.24 (ทดสอบจริงบน 3.26.4), `simplecov` (ทดสอบจริงบน 1.3.1)

ใน Part 019 เราเรียนไปแล้วว่า `double`/`instance_double`/`allow`/`expect(...).to receive`
คือเครื่องมือของ **rspec-mocks** สำหรับปลอมตัวแทน Ruby object ตัวหนึ่งด้วยอีกตัวหนึ่งที่เรา
ควบคุมพฤติกรรมได้เต็มที่ ปิดท้ายด้วยตัวอย่าง `BankAccount` ที่ปลอมตัวแทน
`NotificationService` เพื่อไม่ต้องยิงการแจ้งเตือนจริงทุกครั้งที่รันเทสต์

แต่ในโลกจริง โปรเจกต์ Rails แทบทุกตัวต้องคุยกับ **ระบบภายนอกที่ไม่ใช่ Ruby object ธรรมดา**
เช่น เรียก REST API ของผู้ให้บริการพยากรณ์อากาศ, เรียก payment gateway, เรียก
LINE Notify — สิ่งเหล่านี้คุยกันผ่าน **HTTP protocol ดิบๆ** (`Net::HTTP`, `Faraday`,
`HTTParty`) ซึ่ง `instance_double` ช่วยอะไรไม่ได้เลย เพราะไม่มี "Ruby class ของฝั่งตรงข้าม"
ให้ verify ด้วย — ตัว HTTP request วิ่งออกไปนอกเครื่องจริงๆ ทาง socket

Part นี้จะพาไปตอบคำถาม 3 เรื่องที่ต่อยอดจาก Part 019 โดยตรง:

1. เมื่อไหร่ควร mock/stub เมื่อไหร่ควรใช้ของจริง (ทบทวนให้แน่นขึ้นอีกชั้น ไม่ใช่สอนซ้ำ)
2. จะทดสอบโค้ดที่เรียก HTTP ภายนอกได้อย่างไรโดยไม่ต้องยิง request จริงทุกครั้งที่รันเทสต์
   (WebMock + VCR) และจะทดสอบโค้ดที่พึ่งพาเวลาปัจจุบันได้อย่างไร (`travel_to`)
3. เทสต์ที่เขียนมาทั้งหมดครอบคลุมโค้ดจริงแค่ไหน (SimpleCov) และตัวเลขนั้น "บอกอะไรได้บ้าง
   และบอกอะไรไม่ได้เลย"

## สารบัญของ Part นี้

- Step 481: ทบทวนและเจาะลึก — เมื่อไหร่ควร mock/stub เมื่อไหร่ควรใช้ของจริง
- Step 482: ปัญหาที่ RSpec double แก้ไม่ได้ — HTTP request ดิบและ `WebMock`
- Step 483: gem `vcr` — แนวคิด "cassette" บันทึก HTTP interaction จริงครั้งเดียว เล่นซ้ำได้ตลอดไป
- Step 484: `VCR.configure` เจาะลึก — cassette library, `hook_into`, record mode `:once`/`:new_episodes`/`:none`
- Step 485: `filter_sensitive_data` — ป้องกัน API key หลุดเข้าไปในไฟล์ cassette ที่ commit เข้า git
- Step 486: ตัวอย่างจริงเต็มรูปแบบ — `WeatherService` เรียก external API ทดสอบด้วย VCR cassette
- Step 487: Mocking เวลา — `ActiveSupport::Testing::TimeHelpers` (`travel_to`/`freeze_time`) เทียบกับ Timecop
- Step 488: ติดตั้งและตั้งค่า `SimpleCov` — `require` บนสุดของ `spec_helper.rb`, `SimpleCov.start "rails"`
- Step 489: อ่านรายงาน coverage อย่างมีวิจารณญาณ — 100% ไม่ได้แปลว่าไม่มีบั๊ก
- Step 490: `SimpleCov.minimum_coverage` ใน CI — safety net ไม่ใช่เป้าหมาย + แบบฝึกหัดรวบยอด

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยแอป Rails 8.1.4 บน
> Ruby 3.3.6 ที่สร้างขึ้นมาเฉพาะ รันคำสั่ง `bundle install` และ `bundle exec rspec` จริง
> ทุกจุด รวมถึงเปิด local HTTP server ขึ้นมาจำลอง external API เพื่อบันทึก VCR cassette
> จากการเรียก HTTP จริง (ไม่ใช่ไฟล์ cassette ที่เขียนมือ) แล้วลองปิด server นั้นทิ้งเพื่อ
> พิสูจน์ว่าเทสต์ยังผ่านได้จาก cassette ที่บันทึกไว้โดยไม่มีการเชื่อมต่อเครือข่ายจริงเกิดขึ้น
> อีกเลย — ตรงตามหลักการของ VCR ที่จะอธิบายใน Step 483

---

## Step 481: ทบทวนและเจาะลึก — เมื่อไหร่ควร mock/stub เมื่อไหร่ควรใช้ของจริง

### ทบทวนสั้นๆ จาก Part 019

Part 019 สอนไปแล้วว่า `double`/`instance_double` ใช้ "ปลอมตัวแทน dependency" ของ object
ที่กำลังทดสอบ ตัวอย่าง `BankAccount` ที่ปลอมตัวแทน `NotificationService` เพื่อไม่ต้องยิงการ
แจ้งเตือนจริง คำถามที่ Part นี้จะตอบให้ลึกขึ้นคือ **"ปลอมตัวแทนอะไรได้บ้าง และห้ามปลอมตัวแทน
อะไร"**

### กฎ 4 ข้อ: เมื่อไหร่ควร mock/stub

ใช้ mock/stub เมื่อ dependency ของโค้ดที่กำลังทดสอบมีลักษณะข้อใดข้อหนึ่งต่อไปนี้:

1. **ช้า (slow)** — เชื่อมต่อฐานข้อมูลจริงข้ามเครื่อง, เรียก API ภายนอกที่ใช้เวลาหลักวินาที
   ถ้าเทสต์ 1 ตัวใช้เวลา 2 วินาทีเพราะรอ network และมีเทสต์แบบนี้ 500 ตัว สูทเทสต์ทั้งหมด
   จะใช้เวลาเป็นชั่วโมง ทำให้ทีมไม่กล้ารันเทสต์บ่อยๆ (ขัดกับเป้าหมายพื้นฐานของการเขียนเทสต์)
2. **แพง (expensive)** — มีค่าใช้จ่ายจริงต่อการเรียก 1 ครั้ง เช่น ส่ง SMS จริง, เรียก
   payment gateway จริงที่คิดค่าธรรมเนียมทุก transaction, ส่งอีเมลจริงหาลูกค้า — รันเทสต์
   ทุกวันหลายร้อยครั้งจะกลายเป็นค่าใช้จ่ายมหาศาลหรือทำให้ลูกค้าจริงได้รับอีเมลปลอม
3. **ไม่แน่นอน (non-deterministic)** — ผลลัพธ์เปลี่ยนไปทุกครั้งที่รัน เช่น เวลาปัจจุบัน
   (`Time.now`), เลขสุ่ม, การเชื่อมต่อ network ที่อาจ timeout แบบสุ่มเพราะ latency —
   เทสต์ที่พึ่งพาสิ่งเหล่านี้ตรงๆ จะ **flaky** (บางทีผ่านบางทีไม่ผ่านโดยไม่มีอะไรในโค้ด
   เปลี่ยนเลย) ซึ่งทำลายความน่าเชื่อถือของทั้งสูทเทสต์
4. **เป็นระบบภายนอกที่เราไม่ได้เป็นเจ้าของ (external/out of our control)** — API ของบุคคล
   ที่สาม อาจล่ม, เปลี่ยน response format โดยไม่แจ้งล่วงหน้า, จำกัดจำนวนครั้งที่เรียกได้
   ต่อวัน (rate limit) — ถ้าเทสต์ของเราพึ่งพาว่า API ภายนอกต้องออนไลน์เสมอ เทสต์จะ fail
   เวลา API เขาล่ม ทั้งที่โค้ดของเราไม่มีบั๊กอะไรเลย

สังเกตว่า HTTP call ไปยัง external API เข้าเงื่อนไขได้ถึง 3 ใน 4 ข้อพร้อมกัน (ช้า + ไม่แน่นอน
+ อยู่นอกการควบคุมของเรา) นี่คือเหตุผลที่ Step 482 เป็นต้นไปจะโฟกัสเรื่องนี้เต็มๆ

### กฎเหล็กข้อเดียวที่ห้ามละเมิด: ห้าม mock ตัว object ที่กำลังทดสอบเอง

นี่คือ anti-pattern ที่มือใหม่ทำผิดบ่อยที่สุด — mock **สิ่งที่ตัวเองกำลังพยายามพิสูจน์ว่า
ทำงานถูกต้อง**

```ruby
# frozen_string_literal: true

# lib/order_total_calculator.rb
class OrderTotalCalculator
  def initialize(items)
    @items = items
  end

  def total
    @items.sum { |item| item[:price] * item[:quantity] }
  end
end
```

```ruby
# ตัวอย่างที่ผิดหลักการอย่างร้ายแรง -- ห้ามเขียนแบบนี้เด็ดขาด
RSpec.describe OrderTotalCalculator do
  it "คำนวณราคารวมได้ถูกต้อง" do
    calculator = instance_double(OrderTotalCalculator, total: 500)
    # ทดสอบว่า calculator.total เท่ากับ 500 -- แต่ 500 คือค่าที่เรา "สั่ง" ให้ stub คืนเอง!
    expect(calculator.total).to eq(500)
    # เทสต์นี้ผ่าน 100% เสมอ ไม่ว่า OrderTotalCalculator#total จะมีบั๊กร้ายแรงแค่ไหนก็ตาม
    # เพราะไม่มีการรันโค้ดจริงของ OrderTotalCalculator เลยแม้แต่บรรทัดเดียว
  end
end
```

เทสต์ด้านบน **ไม่ได้ทดสอบอะไรเลย** มันแค่ยืนยันว่า "ถ้าฉันบอกให้ double คืนค่า 500 มันจะคืน
500 จริง" ซึ่งเป็นเรื่องจริงเสมอโดยไม่เกี่ยวอะไรกับคุณภาพของ `OrderTotalCalculator` แม้แต่
น้อย วิธีที่ถูกต้องคือใช้ **object จริง** เมื่อทดสอบ object นั้นโดยตรง:

```ruby
# frozen_string_literal: true

RSpec.describe OrderTotalCalculator do
  it "คำนวณราคารวมได้ถูกต้อง" do
    calculator = described_class.new([
      { price: 100, quantity: 2 },
      { price: 50, quantity: 3 }
    ])

    expect(calculator.total).to eq(350)   # รันโค้ดจริงของ #total แล้วตรวจผลลัพธ์จริง
  end
end
```

### ตารางสรุป: mock/stub dependency VS ใช้ของจริงกับ subject

| สิ่งที่กำลังพิจารณา | ทำอย่างไร |
|---|---|
| **Subject** (class/object ที่ `describe` ครอบอยู่ ตามที่เรียนใน Part 019 Step 188) | ใช้ **ของจริงเสมอ** ห้าม mock ตัวมันเอง เพราะนั่นคือสิ่งที่เทสต์มีไว้พิสูจน์ |
| **Dependency ที่เป็น Ruby object ธรรมดา ควบคุมได้เต็มที่ ทำงานเร็ว ไม่มีผลข้างเคียงภายนอก** (เช่น `Array`, PORO ธรรมดาที่ไม่แตะ network/DB) | ใช้ของจริง ไม่ต้อง mock (mock มากเกินไปทำให้เทสต์เปราะบาง แก้ implementation แล้วเทสต์พังทั้งที่ behavior ยังถูกต้อง) |
| **Dependency ที่ช้า/แพง/ไม่แน่นอน/อยู่นอกการควบคุม** (external API, payment gateway, ระบบส่งอีเมลจริง, `Time.now`) | Mock/stub เสมอ — ตาม 4 ข้อด้านบน |
| **Dependency ที่เป็น HTTP call ดิบ ไม่ใช่ Ruby object** (`Net::HTTP.get`, Faraday, HTTParty เรียก URL ภายนอก) | `instance_double` ใช้ไม่ได้ ต้องใช้เครื่องมือระดับ HTTP โดยตรง — **WebMock**/**VCR** (Step 482 เป็นต้นไป) |

---

## Step 482: ปัญหาที่ RSpec double แก้ไม่ได้ — HTTP request ดิบและ `WebMock`

### ทำไม `instance_double` ช่วยไม่ได้กับ HTTP call

ลองสมมติ service object ที่เรียก external weather API ตรงๆ ผ่าน `Net::HTTP` (standard
library ของ Ruby ที่เรียนไปแล้วเรื่อง `require` ใน Part 001):

```ruby
# frozen_string_literal: true

# app/services/weather_service.rb
require "net/http"
require "json"

class WeatherApiError < StandardError; end

class WeatherService
  ENDPOINT = "https://api.weatherapi.com/v1/current.json"

  def initialize(api_key: Rails.application.credentials.dig(:weather_api, :key))
    @api_key = api_key
  end

  def current_temperature(city)
    uri = URI(ENDPOINT)
    uri.query = URI.encode_www_form(key: @api_key, q: city)

    response = Net::HTTP.get_response(uri)
    raise WeatherApiError, "WeatherAPI returned #{response.code}" unless response.is_a?(Net::HTTPSuccess)

    data = JSON.parse(response.body)
    data.dig("current", "temp_c")
  end
end
```

ลองมองหาว่าจะ `instance_double` อะไรได้บ้าง — คำตอบคือ **ไม่มีเลย** เพราะ:

- `Net::HTTP.get_response` เป็น class method ของ standard library ที่มากับ Ruby เอง ไม่ใช่
  dependency ที่เรา inject เข้ามาแบบ `notifier:` ใน `BankAccount` ของ Part 019
- แม้จะสร้าง `instance_double(Net::HTTP)` ได้ในทางเทคนิค แต่มันจะไม่ได้ห้าม **socket
  connection จริง** ที่เกิดขึ้นตอนเรียก `URI(ENDPOINT)` และยิง TCP request จริงออกไปนอกเครื่อง
  — `instance_double` แค่ปลอมตัวแทน method call ระดับ Ruby object เท่านั้น มันไม่รู้จัก
  และไม่เกี่ยวข้องกับ network stack เลย
- ถ้ารันเทสต์ตรงๆ โดยไม่มีอะไรกันไว้เลย โค้ดข้างบนจะยิง HTTP request จริงออกไปยัง
  `api.weatherapi.com` ทุกครั้งที่รันเทสต์ — ช้า, ต้องมี internet, ต้องมี API key จริง, และ
  ผลลัพธ์เปลี่ยนไปตามสภาพอากาศจริงในแต่ละวัน (เข้าเงื่อนไขทั้ง 3 ข้อจาก Step 481 พร้อมกัน)

### `WebMock` — สกัดกั้น HTTP request ตั้งแต่ระดับ library ก่อนจะออกจากเครื่อง

`WebMock` เป็น gem ที่ทำงานโดย **monkey-patch HTTP library ยอดนิยมทุกตัวใน Ruby**
(`Net::HTTP`, Faraday, HTTParty, Excon ฯลฯ) ให้สกัดกั้น request ทุกตัวไว้ก่อนที่จะสร้าง
socket connection จริง ต่างจาก `instance_double` ตรงที่ WebMock ทำงาน **ที่ระดับ HTTP
protocol** (method, URL, headers, body) ไม่ใช่ระดับ Ruby method call

ติดตั้งใน `Gemfile`:

```ruby
group :test do
  gem "webmock", "~> 3.24"
end
```

```bash
bundle install
```

เปิดใช้งานใน `spec/rails_helper.rb`:

```ruby
require "webmock/rspec"
```

เมื่อ require `webmock/rspec` แล้ว **WebMock จะปิดกั้นการเชื่อมต่อ network จริงทั้งหมดโดย
อัตโนมัติ** สำหรับทุกเทสต์ในสูท ถ้ามี HTTP request ใดหลุดออกไปโดยไม่ได้ stub ไว้ก่อน
เทสต์จะ fail ทันทีพร้อมข้อความบอกชัดเจนว่า request ไหนที่ไม่ได้ถูก stub — เป็นกลไกป้องกัน
ไม่ให้เทสต์แอบยิง network จริงโดยไม่ได้ตั้งใจ (fail-safe by default)

### `stub_request` — กำหนดคำตอบปลอมให้ HTTP request ที่ตรงเงื่อนไข

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe WeatherService do
  it "คืนอุณหภูมิปัจจุบันจาก response ที่ stub ไว้" do
    stub_request(:get, "https://api.weatherapi.com/v1/current.json")
      .with(query: { key: "dummy-key", q: "Bangkok" })
      .to_return(
        status: 200,
        body: { location: { name: "Bangkok" }, current: { temp_c: 32.5 } }.to_json,
        headers: { "Content-Type" => "application/json" }
      )

    service = WeatherService.new(api_key: "dummy-key")
    expect(service.current_temperature("Bangkok")).to eq(32.5)
  end

  it "raise WeatherApiError เมื่อ API ตอบกลับด้วย error status" do
    stub_request(:get, "https://api.weatherapi.com/v1/current.json")
      .with(query: { key: "dummy-key", q: "InvalidCity" })
      .to_return(status: 400, body: "Bad Request")

    service = WeatherService.new(api_key: "dummy-key")
    expect { service.current_temperature("InvalidCity") }.to raise_error(WeatherApiError, /400/)
  end
end
```

**กายวิภาคของ `stub_request`:**

- `stub_request(:get, url)` — บอกว่า request HTTP method `GET` ไปยัง `url` นี้ ให้จับ
  ไว้ (รองรับทุก method: `:get`, `:post`, `:put`, `:patch`, `:delete`)
- `.with(query: {...})` — จำกัดให้ตรงเฉพาะ request ที่มี query string ตรงตามนี้เท่านั้น
  (ถ้า service เรียกด้วย city อื่นที่ไม่ได้ stub ไว้ WebMock จะ raise error ทันที ไม่ใช่
  คืนค่าว่างเงียบๆ)
- `.to_return(status:, body:, headers:)` — กำหนด response ปลอมที่จะได้กลับมา ราวกับว่า
  server จริงตอบกลับมาแบบนี้

### ถ้ามี HTTP request ที่ไม่ได้ stub ไว้เลย

```ruby
it "ตัวอย่างที่ตั้งใจไม่ stub อะไรเลย" do
  service = WeatherService.new(api_key: "dummy-key")
  service.current_temperature("Phuket")   # ไม่มีการ stub_request ไว้เลย
end
```

ผลลัพธ์จริงเมื่อรัน (ข้อความยาวมากในทางปฏิบัติ ตัดมาเฉพาะส่วนสำคัญ):

```
WebMock::NetConnectNotAllowedError:
  Real HTTP connections are disabled. Unregistered request:

  GET: https://api.weatherapi.com/v1/current.json?key=dummy-key&q=Phuket

  You can stub this request with the following snippet:

  stub_request(:get, "https://api.weatherapi.com/v1/current.json?key=dummy-key&q=Phuket").
    to_return(status: 200, body: "", headers: {})
```

สังเกตว่า WebMock ใจดีมาก — มันบอกวิธี stub ให้เสร็จสรรพเลย (copy วางแล้วแก้ body ได้ทันที)
นี่คือประโยชน์สำคัญของการปิดกั้น network จริงเป็นค่าเริ่มต้น: **บั๊กจากการลืม stub ถูกจับ
ได้ทันทีที่รันเทสต์ ไม่ใช่ตอน CI พยายามต่อ internet แล้ว timeout อย่างงงๆ**

### ข้อจำกัดของ WebMock เพียงลำพัง

`stub_request` ใช้งานได้ดีมากกับ request ง่ายๆ 1–2 เคส แต่พอ service มี HTTP call ที่
ซับซ้อนขึ้น (response จริงมี field เป็นร้อยตัว, ต้องทดสอบหลาย city หลาย edge case) การ
เขียน `.to_return(body: {...}.to_json)` มือทุกครั้งจะเริ่มเป็นภาระ:

1. ต้องคาดเดา/จำลอง response format ของ API จริงเองทั้งหมด เสี่ยงผิดเพี้ยนจาก response
   จริง (เช่น จริงๆ API คืน field ชื่อ `temp_C` ตัวใหญ่ แต่เราจำลองเป็น `temp_c` ตัวเล็ก
   โดยไม่รู้ตัว)
2. ถ้า API เปลี่ยน response format จริง เทสต์ที่ stub เองจะไม่มีทางรู้เลย เพราะไม่เคยคุย
   กับ API จริงอีกต่อไปหลังจากเขียน stub ครั้งแรก

นี่คือช่องว่างที่ **VCR** เข้ามาเติมเต็ม — แทนที่จะ "เดา" response เราจะ "บันทึก" response
จริงจาก API ไว้ 1 ครั้ง แล้วเล่นซ้ำไฟล์ที่บันทึกไว้นั้นตลอดไป

---

## Step 483: gem `vcr` — แนวคิด "cassette" บันทึก HTTP interaction จริงครั้งเดียว เล่นซ้ำได้ตลอดไป

### แนวคิดหลักของ VCR

**VCR** (ชื่อมาจากเครื่องเล่นวิดีโอเทป Video Cassette Recorder) ทำงานตามชื่อ:

1. **ครั้งแรก** ที่รันเทสต์ พร้อม cassette ที่ยังไม่มีไฟล์อยู่จริง → VCR ปล่อยให้ HTTP
   request วิ่งออกไปหา server จริง (ผ่านเครือข่ายจริง) แล้ว **บันทึก** ทั้ง request และ
   response ทั้งหมดลงไฟล์ YAML ที่เรียกว่า **cassette**
2. **ครั้งต่อๆ ไป** ที่รันเทสต์เดิม → VCR เห็นว่ามี cassette ไฟล์นั้นอยู่แล้ว จึง **ไม่ยิง
   HTTP request จริงอีกเลย** แต่ใช้ response ที่บันทึกไว้ในไฟล์นั้นตอบกลับแทนทันที

ผลลัพธ์คือได้ทั้งสองข้อดีพร้อมกัน: **ความสมจริงของ response จริงจาก API** (เพราะบันทึก
มาจากการเรียกจริงจริงๆ) **บวกกับความเร็ว/ความแน่นอนของการ mock** (เพราะหลังบันทึกครั้งแรก
ไม่มีการต่อ network อีกเลย)

VCR ไม่ได้แทนที่ WebMock — มันทำงาน**บนหลัง** WebMock (`hook_into :webmock`) โดยเพิ่ม
ชั้นการจัดการไฟล์ cassette ให้อัตโนมัติ แทนที่จะต้องเขียน `stub_request` มือทุกเคส

### ติดตั้ง

```ruby
# Gemfile
group :test do
  gem "vcr", "~> 6.3"
  gem "webmock", "~> 3.24"
end
```

```bash
bundle install
```

เวอร์ชันที่ทดสอบจริงในบทเรียนนี้:

```bash
bundle exec gem list vcr webmock
# vcr (6.4.0)
# webmock (3.26.4)
```

### ตั้งค่าเบื้องต้นแบบง่ายที่สุดก่อน (รายละเอียดเต็มอยู่ใน Step 484)

สร้างไฟล์ `spec/support/vcr.rb`:

```ruby
# frozen_string_literal: true

require "vcr"
require "webmock/rspec"

VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
end
```

และให้ `spec/rails_helper.rb` โหลดไฟล์ทุกไฟล์ใน `spec/support/` อัตโนมัติ (บรรทัดนี้มากับ
`rails generate rspec:install` อยู่แล้ว เพียงแต่ถูก comment ไว้ ให้เปิดออก):

```ruby
Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }
```

### ทดสอบจริงแบบครบวงจร: บันทึกครั้งแรก แล้วปิด server ทิ้ง

เพื่อพิสูจน์ให้เห็นภาพชัดที่สุดว่า cassette ทำงานอย่างไรจริงๆ (ไม่ใช่แค่ทฤษฎี) บทเรียนนี้
จำลอง external API ด้วย local HTTP server ของเราเอง (ใช้ `WEBrick` ซึ่งเป็น standard
library เดิมของ Ruby ก่อนถูกแยกเป็น gem ต่างหากในเวอร์ชันหลัง) แทนที่จะพึ่งพา API จริง
บนอินเทอร์เน็ตที่ต้องมี API key และอาจไม่เสถียรเวลาทำตามในห้องเรียน — **กลไกของ VCR ที่
ได้เรียนจะเหมือนกันทุกประการไม่ว่าจะเรียก server จำลองนี้ หรือ API จริงบนอินเทอร์เน็ต**
เพียงแค่เปลี่ยนค่า `ENDPOINT` ในตัวอย่างต่อไปนี้ให้ชี้ไปยัง URL จริงของผู้ให้บริการเท่านั้น

```ruby
# frozen_string_literal: true

# tmp_test_server.rb (ใช้ประกอบการสาธิตเท่านั้น ไม่ใช่ส่วนหนึ่งของแอป Rails จริง)
require "webrick"
require "json"

server = WEBrick::HTTPServer.new(Port: 9393, Logger: WEBrick::Log.new(File::NULL), AccessLog: [])

server.mount_proc "/v1/current.json" do |req, res|
  res["Content-Type"] = "application/json"
  city = req.query["q"] || "unknown"
  res.body = { location: { name: city }, current: { temp_c: 32.5, condition: { text: "Sunny" } } }.to_json
end

trap("INT") { server.shutdown }
server.start
```

```bash
gem install webrick
ruby tmp_test_server.rb &   # รันเป็น background process จำลอง external API
```

แก้ `WeatherService::ENDPOINT` ให้ชี้มาที่ server จำลองนี้ชั่วคราว (`http://127.0.0.1:9393/v1/current.json`)
แล้วเขียนสเปคพร้อม cassette:

```ruby
# frozen_string_literal: true

# spec/services/weather_service_spec.rb
require "rails_helper"

RSpec.describe WeatherService do
  subject(:service) { described_class.new(api_key: "dummy-key") }

  describe "#current_temperature", vcr: { cassette_name: "weather_service/current_temperature" } do
    it "returns the current temperature for the given city" do
      temperature = service.current_temperature("Bangkok")
      expect(temperature).to eq(32.5)
    end
  end
end
```

`vcr: { cassette_name: "..." }` เป็น metadata ที่ทำงานได้เพราะเราเรียก
`config.configure_rspec_metadata!` ไว้แล้วใน `VCR.configure` — มันบอก RSpec ว่า "example
นี้ให้ครอบด้วย `VCR.use_cassette("...")` ให้อัตโนมัติ" โดยไม่ต้องเขียน `VCR.use_cassette`
ล้อมเองทุกจุด

รันครั้งแรก (server จำลองยังทำงานอยู่):

```bash
bundle exec rspec spec/services/weather_service_spec.rb --format documentation
```

```
WeatherService
  #current_temperature
    returns the current temperature for the given city

Finished in 0.03088 seconds (files took 1.31 seconds to load)
1 example, 0 failures
```

ตรวจดูไฟล์ที่ VCR สร้างขึ้นให้อัตโนมัติที่ `spec/cassettes/weather_service/current_temperature.yml`
(เนื้อหาจริงที่บันทึกได้ในการทดสอบนี้):

```yaml
---
http_interactions:
- request:
    method: get
    uri: http://127.0.0.1:9393/v1/current.json?key=dummy-key&q=Bangkok
    body:
      encoding: US-ASCII
      string: ''
    headers:
      Accept-Encoding:
      - gzip;q=1.0,deflate;q=0.6,identity;q=0.3
      Accept:
      - "*/*"
      User-Agent:
      - Ruby
      Host:
      - 127.0.0.1:9393
  response:
    status:
      code: 200
      message: OK
    headers:
      Content-Type:
      - application/json
      Server:
      - WEBrick/1.9.2 (Ruby/3.3.6/2024-11-05)
      Content-Length:
      - '86'
    body:
      encoding: UTF-8
      string: '{"location":{"name":"Bangkok"},"current":{"temp_c":32.5,"condition":{"text":"Sunny"}}}'
  recorded_at: Sat, 26 Sep 2026 06:46:25 GMT
recorded_with: VCR 6.4.0
```

นี่คือ **response จริง** ที่บันทึกมาจาก server จำลองจริงๆ ไม่ใช่ค่าที่เราเดาหรือพิมพ์มือ
สังเกตว่าไฟล์นี้เก็บทุกอย่างที่จำเป็นในการ "เล่นซ้ำ" HTTP interaction นี้: method, URL,
headers, body ทั้งฝั่ง request และ response

**ขั้นตอนพิสูจน์สำคัญที่สุด:** ปิด server จำลองทิ้ง (`kill` process ที่รันอยู่) แล้วรันเทสต์
เดิมซ้ำอีกครั้ง:

```bash
kill %1   # ฆ่า background process ของ WEBrick server
bundle exec rspec spec/services/weather_service_spec.rb --format documentation
```

```
WeatherService
  #current_temperature
    returns the current temperature for the given city

Finished in 0.02219 seconds (files took 1.14 seconds to load)
1 example, 0 failures
```

**เทสต์ยังผ่านเหมือนเดิมทุกประการ แม้ server ที่เป็นต้นทางของ HTTP request จะถูกปิดไป
แล้วโดยสิ้นเชิง** — เพราะ VCR อ่านคำตอบจากไฟล์ cassette ที่บันทึกไว้ในรอบก่อนหน้าแทนการยิง
HTTP request จริง นี่คือหัวใจของ VCR: **บันทึกความจริงไว้ครั้งเดียว ใช้ซ้ำได้ตลอดไปโดยไม่
ต้องพึ่งพา network หรือ API key จริงอีกเลยในการรันเทสต์ประจำวัน**

> **ทำไมต้อง commit ไฟล์ cassette เข้า git:** เพราะเพื่อนร่วมทีมและ CI server จะไม่มีทาง
> เข้าถึง server จำลอง (หรือ API จริงตอน record ครั้งแรก) ได้เหมือนเรา การ commit ไฟล์
> `.yml` เข้า version control ทำให้ทุกคนที่ clone โปรเจกต์รันเทสต์ตัวนี้ได้ทันทีโดยไม่ต้อง
> มี API key จริงเลยด้วยซ้ำ (ยกเว้นตอนต้องการ re-record ใหม่ ซึ่งจะพูดถึงใน Step 484)

---

## Step 484: `VCR.configure` เจาะลึก — cassette library, `hook_into`, record mode `:once`/`:new_episodes`/`:none`

### ค่าคอนฟิกที่ใช้บ่อยที่สุดครบชุด

```ruby
# frozen_string_literal: true

# spec/support/vcr.rb
require "vcr"
require "webmock/rspec"

VCR.configure do |config|
  # โฟลเดอร์เก็บไฟล์ cassette ทั้งหมด (ค่าเริ่มต้นคือ spec/vcr_cassettes ถ้าไม่ระบุ)
  config.cassette_library_dir = "spec/cassettes"

  # บอก VCR ว่าให้ทำงานผ่าน WebMock (ทางเลือกอื่นคือ :typhoeus, :excon แต่ :webmock
  # ครอบคลุม HTTP library ที่ใช้บ่อยที่สุดเกือบทั้งหมดแล้ว รวมถึง Net::HTTP และ Faraday)
  config.hook_into :webmock

  # เปิดใช้ metadata `vcr: { cassette_name: "..." }` ใน RSpec example (ตามที่เห็นใน Step 483)
  config.configure_rspec_metadata!

  # ค่าเริ่มต้นของทุก cassette ถ้าไม่ได้ระบุ record mode เจาะจงไว้
  config.default_cassette_options = {
    record: :once
  }

  # อนุญาตให้เทสต์ที่ไม่ได้ใช้ VCR เลยยิง localhost ได้ตามปกติ (มีประโยชน์เวลารัน Capybara
  # feature test ที่ต้องคุยกับ Rails server ของตัวเองในเทสต์เดียวกัน)
  config.ignore_localhost = true
end
```

### Record mode ทั้ง 3 แบบ

| Record mode | พฤติกรรม | ใช้เมื่อ |
|---|---|---|
| `:once` (**ค่าเริ่มต้นที่แนะนำ**) | ถ้ายังไม่มีไฟล์ cassette → บันทึกใหม่จาก network จริง ถ้ามีไฟล์อยู่แล้ว → เล่นซ้ำอย่างเดียว ห้ามต่อ network อีก | เกือบทุกกรณีในทางปฏิบัติ ปลอดภัยที่สุดเพราะบันทึกได้แค่ครั้งเดียวเท่านั้น |
| `:new_episodes` | ถ้ามี cassette อยู่แล้ว เล่นซ้ำ interaction ที่เคยบันทึกไว้ตามปกติ แต่ถ้าเจอ request ใหม่ที่ยังไม่เคยมีในไฟล์ (เช่น เพิ่ม test case ที่เรียก city ใหม่) จะยิง network จริงแล้ว **เพิ่ม** เข้าไปในไฟล์เดิม | เมื่อกำลังเพิ่ม test case ใหม่ให้ spec ที่มี cassette เดิมอยู่แล้ว และต้องการบันทึก interaction ใหม่เพิ่มโดยไม่อยากลบไฟล์เก่าทั้งหมด |
| `:none` | ห้ามยิง network จริงเด็ดขาดไม่ว่ากรณีใด ถ้า request ไม่ตรงกับที่บันทึกไว้ใน cassette → raise error ทันที | รันบน CI server ที่ไม่ควรมีการยิง network จริงเกิดขึ้นได้เลยแม้แต่ครั้งเดียว (ป้องกัน CI แอบบันทึก cassette ใหม่โดยไม่ตั้งใจ) |

ตัวอย่างการยืนยันพฤติกรรมของ `:none` (ทดสอบจริง โดยเจตนาลบ cassette ทิ้งก่อนแล้วลอง
บังคับ record mode เป็น `:none`):

```ruby
VCR.use_cassette("nonexistent_cassette", record: :none) do
  WeatherService.new.current_temperature("Bangkok")
end
```

ผลลัพธ์จริงเมื่อรัน — ไม่มีการต่อ network เกิดขึ้นเลย แต่ raise error ทันทีแทน:

```
VCR::Errors::UnhandledHTTPRequestError:

  ==================================================================
  An HTTP request has been made that VCR does not know how to handle:
    GET http://127.0.0.1:9393/v1/current.json?key=dummy-key&q=Bangkok

  There is currently a cassette in use named 'nonexistent_cassette' which
  does not contain a matching request, and is configured to not allow
  new requests to be recorded because :record => :none is being used
  (which is the default when no cassette is in use).
  ==================================================================
```

ข้อความ error ของ VCR ยาวและอธิบายละเอียดมาก (ตัดมาส่วนสำคัญ) — บอกชัดว่า request ไหน
ที่ไม่ตรง cassette และเหตุผลที่ระบบปฏิเสธ ทำให้ debug ได้เร็วโดยไม่ต้องเดา

### กำหนด record mode เฉพาะ example ตัวเดียว (override ค่า default)

```ruby
RSpec.describe WeatherService do
  # ใช้ตอนอยากบังคับ re-record cassette ใหม่ทั้งไฟล์ (เช่น API เปลี่ยน response format จริง)
  it "ตัวอย่างที่บังคับ re-record cassette ใหม่ทุกครั้งชั่วคราว", vcr: {
    cassette_name: "weather_service/current_temperature",
    record: :new_episodes
  } do
    # ...
  end
end
```

ในทางปฏิบัติ เวลาต้องการบันทึก cassette ใหม่ทั้งไฟล์เพราะ API เปลี่ยนจริง วิธีที่ปลอดภัย
กว่าการแก้ `record:` ในโค้ดคือ **ลบไฟล์ cassette เก่าทิ้งแล้วรันเทสต์ใหม่** เพราะ `:once`
จะตรวจไม่พบไฟล์เดิม แล้วบันทึกใหม่ให้เองโดยอัตโนมัติ ไม่ต้องแก้โค้ด spec เลยแม้แต่บรรทัด
เดียว — เป็นวิธีที่แนะนำมากกว่าการสลับ record mode ไปมาในโค้ด

### เมื่อมี HTTP call ที่ไม่ตรงกับ cassette (แม้ cassette จะมีไฟล์อยู่)

ถ้าเรียก city ที่ไม่เคยบันทึกไว้ (เช่น cassette มีแค่ query `q=Bangkok` แต่โค้ดพยายามเรียก
`q=Phuket`) VCR mode `:once` จะปฏิเสธทันทีเช่นกัน เพราะถือว่า cassette "มีอยู่แล้ว" จึงห้าม
บันทึกเพิ่ม (ต้องใช้ `:new_episodes` เท่านั้นถ้าต้องการพฤติกรรมแบบเพิ่มได้)

---

## Step 485: `filter_sensitive_data` — ป้องกัน API key หลุดเข้าไปในไฟล์ cassette ที่ commit เข้า git

### ปัญหา: cassette เก็บทุกอย่างของ request จริง รวมถึงความลับด้วย

จากไฟล์ cassette ใน Step 483 สังเกตว่า URL ที่บันทึกไว้มี `key=dummy-key` ติดอยู่ในไฟล์
`.yml` ตรงๆ — ในสถานการณ์จริงที่ใช้ API key จริง (ไม่ใช่ `"dummy-key"` แบบตัวอย่าง) ค่านี้
จะเป็น**ความลับจริง**ที่รั่วไหลเข้าไปในไฟล์ cassette แล้วถูก commit เข้า git repository
โดยไม่มีใครรู้ตัว — ถ้า repository เป็น public หรือแม้แต่ private ที่มีคนเข้าถึงได้หลายคน
API key นั้นก็รั่วไหลไปแล้ว

### `filter_sensitive_data` — แทนที่ข้อมูลลับด้วย placeholder ก่อนเขียนลงไฟล์

```ruby
# frozen_string_literal: true

VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!

  config.filter_sensitive_data("<WEATHER_API_KEY>") { ENV.fetch("WEATHER_API_KEY", "dummy-key") }
end
```

`filter_sensitive_data(placeholder) { block }` ทำงาน 2 ทิศทาง:

1. **ตอนบันทึก (record):** ก่อนเขียนไฟล์ VCR จะรัน block ที่ให้ไว้ (`ENV.fetch(...)`)
   ได้ค่าจริงของ API key ออกมา แล้วค้นหาค่านั้นในเนื้อหา request/response ทั้งหมด
   แทนที่ทุกจุดที่เจอด้วย placeholder (`<WEATHER_API_KEY>`) ก่อนเขียนลงไฟล์จริง
2. **ตอนเล่นซ้ำ (replay):** VCR จะทำย้อนกลับ — แทนที่ placeholder ในไฟล์ด้วยค่าจริงจาก
   block อีกครั้งชั่วคราวตอนเปรียบเทียบกับ request ที่โค้ดยิงเข้ามา เพื่อให้ match กันได้
   ถูกต้อง (คือ API key จริงไม่เคยถูกเขียนลงไฟล์เลยตลอดกระบวนการ)

ผลลัพธ์ไฟล์ cassette หลังใส่ `filter_sensitive_data` (ทดสอบจริง เทียบกับ Step 483 ที่ยัง
ไม่ได้ filter):

```yaml
---
http_interactions:
- request:
    method: get
    uri: http://127.0.0.1:9393/v1/current.json?key=<WEATHER_API_KEY>&q=Bangkok
    # ...
```

สังเกตว่า `key=dummy-key` ในไฟล์เดิมกลายเป็น `key=<WEATHER_API_KEY>` แล้ว — ไม่ว่า API key
จริงจะเป็นค่าอะไรก็ตาม จะไม่ปรากฏในไฟล์ที่ commit เข้า git เลยแม้แต่ตัวอักษรเดียว

### ใช้ `Rails.application.credentials` แทน `ENV` ได้เช่นกัน

```ruby
config.filter_sensitive_data("<WEATHER_API_KEY>") do
  Rails.application.credentials.dig(:weather_api, :key)
end
```

หลักการเดียวกันทุกประการ ไม่ว่า API key จริงจะเก็บไว้ที่ environment variable (ตามที่จะ
เรียนละเอียดเรื่อง credentials ใน Part 074) หรือ Rails encrypted credentials ก็ใช้วิธีนี้
กรองได้เหมือนกัน

### filter หลายค่าพร้อมกัน

```ruby
VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!

  config.filter_sensitive_data("<WEATHER_API_KEY>") { ENV.fetch("WEATHER_API_KEY", "dummy-key") }
  config.filter_sensitive_data("<STRIPE_SECRET_KEY>") { ENV.fetch("STRIPE_SECRET_KEY", nil) }
  config.filter_sensitive_data("<AUTHORIZATION_HEADER>") do |interaction|
    interaction.request.headers["Authorization"]&.first
  end
end
```

รูปแบบสุดท้ายที่รับ `interaction` เป็น argument มีประโยชน์เมื่อความลับไม่ได้อยู่ใน
environment variable ตรงๆ แต่อยู่ใน header ของ request เอง (เช่น `Authorization: Bearer ...`
ที่สร้างขึ้นแบบ dynamic ทุกครั้ง)

> **กฎปฏิบัติสำคัญ:** ทุกโปรเจกต์ที่ใช้ VCR ควรตั้ง `filter_sensitive_data` ให้ครบทุกจุดที่
> มีความลับ **ตั้งแต่วันแรกที่เริ่มใช้ VCR** อย่ารอจนกว่าจะมี API key จริงหลุดไปแล้วค่อยแก้
> เพราะไฟล์ cassette เก่าที่ commit ไปแล้วจะยังฝัง secret ไว้ในประวัติ git ตลอดไป
> (ต้องแก้ด้วยการ rewrite git history ซึ่งยุ่งยากกว่ามาก)

---

## Step 486: ตัวอย่างจริงเต็มรูปแบบ — `WeatherService` เรียก external API ทดสอบด้วย VCR cassette

รวมทุกเทคนิคจาก Step 482–485 เป็นตัวอย่างที่สมบูรณ์ เพื่อให้เห็นภาพรวมทั้งกระบวนการตั้งแต่
service object จริงจนถึง spec ที่ทดสอบครบทุก branch

### Service object

```ruby
# frozen_string_literal: true

# app/services/weather_service.rb
require "net/http"
require "json"

class WeatherApiError < StandardError; end

class WeatherService
  ENDPOINT = "https://api.weatherapi.com/v1/current.json"

  def initialize(api_key: Rails.application.credentials.dig(:weather_api, :key))
    @api_key = api_key
  end

  def current_temperature(city)
    uri = URI(ENDPOINT)
    uri.query = URI.encode_www_form(key: @api_key, q: city)

    response = Net::HTTP.get_response(uri)
    raise WeatherApiError, "WeatherAPI returned #{response.code}" unless response.is_a?(Net::HTTPSuccess)

    data = JSON.parse(response.body)
    data.dig("current", "temp_c")
  end
end
```

### การตั้งค่า VCR แบบเต็ม (รวมทุก Step ก่อนหน้า)

```ruby
# frozen_string_literal: true

# spec/support/vcr.rb
require "vcr"
require "webmock/rspec"

VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data("<WEATHER_API_KEY>") { ENV.fetch("WEATHER_API_KEY", "dummy-key") }

  config.default_cassette_options = { record: :once }
end
```

### Spec ที่ทดสอบทั้งเคสสำเร็จและเคส error

```ruby
# frozen_string_literal: true

# spec/services/weather_service_spec.rb
require "rails_helper"

RSpec.describe WeatherService do
  subject(:service) { described_class.new(api_key: "dummy-key") }

  describe "#current_temperature" do
    context "เมื่อ API ตอบกลับสำเร็จ", vcr: { cassette_name: "weather_service/bangkok_success" } do
      it "คืนอุณหภูมิปัจจุบันเป็นตัวเลข" do
        expect(service.current_temperature("Bangkok")).to eq(32.5)
      end
    end

    context "เมื่อ API ตอบกลับด้วย error status", vcr: { cassette_name: "weather_service/invalid_city" } do
      it "raise WeatherApiError" do
        expect { service.current_temperature("InvalidCityXYZ") }.to raise_error(WeatherApiError, /400/)
      end
    end
  end
end
```

**หมายเหตุเรื่องการบันทึก cassette ของเคส error:** ครั้งแรกที่รันสเปคนี้ ต้องมีบางอย่าง
ตอบกลับด้วย HTTP 400 จริงให้ VCR บันทึก (ในโปรเจกต์จริงมักทำโดยเรียก API จริงด้วยพารามิเตอร์
ที่รู้อยู่แล้วว่าจะทำให้ API ตอบ error เช่น ชื่อเมืองที่ไม่มีอยู่จริง) หลังบันทึกครั้งแรก
แล้ว cassette ไฟล์ที่ได้จะเล่นซ้ำ error response นั้นได้ตลอดไปโดยไม่ต้องพึ่งพา API จริงอีก
— รวมถึงเคส error ก็ไม่ต่างจากเคสสำเร็จเลยในแง่การทำงานของ VCR

### เปรียบเทียบวิธีคิดระหว่าง 3 เครื่องมือที่เรียนมา

| เครื่องมือ | ใช้กับ | ต้นทาง response | เหมาะกับ |
|---|---|---|---|
| `instance_double` (Part 019) | Ruby object ที่เรา inject เป็น dependency | เราเขียนเองล้วนๆ | Dependency ที่เป็น Ruby class ในระบบเรา/gem ที่เราควบคุมได้ |
| `stub_request` (WebMock, Step 482) | HTTP request ดิบ | เราเขียนเองล้วนๆ | Response ง่ายๆ 1–2 เคส ที่ไม่อยากตั้งค่า cassette ให้ยุ่งยาก |
| VCR cassette (Step 483–486) | HTTP request ดิบ | บันทึกจาก API จริงจริงๆ | Service ที่คุยกับ external API ที่มี response ซับซ้อน อยากให้ตรงกับของจริงเป๊ะ |

ในทางปฏิบัติ ทีมส่วนใหญ่ใช้ VCR เป็นค่าเริ่มต้นสำหรับทุก external API integration และเก็บ
`stub_request` ตรงๆ ไว้ใช้เฉพาะเทสต์เล็กๆ ที่ไม่คุ้มจะสร้าง cassette (เช่น ทดสอบว่า
WebMock ปิดกั้น network จริงถูกต้องตามที่โชว์ใน Step 482)

---

## Step 487: Mocking เวลา — `ActiveSupport::Testing::TimeHelpers` (`travel_to`/`freeze_time`) เทียบกับ Timecop

### ปัญหา: โค้ดที่พึ่งพา "เวลาปัจจุบัน" ทดสอบยากตรงไหน

`Time.current`/`Date.today` เข้าเงื่อนไข "ไม่แน่นอน (non-deterministic)" จาก Step 481
โดยตรง — ทุกครั้งที่รันเทสต์ ค่าที่ได้เปลี่ยนไปเสมอ ทำให้เทสต์ที่เขียนแบบตรงไปตรงมาพังได้
โดยไม่มีอะไรผิด:

```ruby
# frozen_string_literal: true

# app/services/subscription_expiry_checker.rb
class SubscriptionExpiryChecker
  def initialize(expires_at)
    @expires_at = expires_at
  end

  def expired?
    Time.current >= @expires_at
  end

  def days_remaining
    return 0 if expired?

    ((@expires_at - Time.current) / 1.day).ceil
  end
end
```

```ruby
# ตัวอย่างที่ผิดหลักการ -- เทสต์นี้ผลลัพธ์เปลี่ยนไปทุกวันที่รัน ขึ้นอยู่กับว่า "วันนี้" คือวันไหน
RSpec.describe SubscriptionExpiryChecker do
  it "ยังไม่หมดอายุถ้าวันหมดอายุยังไม่ถึง" do
    checker = described_class.new(3.days.from_now)   # ขึ้นกับเวลาจริงตอนรันเทสต์เสมอ
    expect(checker.expired?).to be(false)
  end
end
```

เทสต์นี้ "ดูเหมือน" ใช้งานได้ แต่จริงๆ แล้วมันทดสอบแค่ว่า "3 วันจากตอนนี้ ยังไม่ถึง 3 วัน
จากตอนนี้" ซึ่งเป็นจริงเสมอโดยไม่เกี่ยวกับ business logic เลย และถ้าจะเขียนเทสต์ที่ต้องการ
วันที่ตายตัวเป๊ะๆ (เช่น "หมดอายุพอดีวันที่ 10 มกราคม 2026 เวลาเที่ยงคืน") จะเขียนไม่ได้เลย
ถ้าไม่มีวิธีควบคุมว่า "ตอนนี้" คือเมื่อไหร่

### `ActiveSupport::Testing::TimeHelpers` — มากับ Rails ในตัว ไม่ต้องเพิ่ม gem

Rails มี module ชื่อ `ActiveSupport::Testing::TimeHelpers` ให้ใช้งานได้ทันทีโดยไม่ต้อง
เพิ่ม gem อะไรเพิ่มเลย เพียงแค่ include เข้าไปใน RSpec config:

```ruby
# frozen_string_literal: true

# spec/rails_helper.rb
RSpec.configure do |config|
  config.include ActiveSupport::Testing::TimeHelpers

  # ... config อื่นๆ ตามเดิม
end
```

จากนั้นจะมี method 3 ตัวใช้งานได้ในทุก example:

| Method | พฤติกรรม |
|---|---|
| `travel_to(time) { ... }` | ตั้งค่า "ตอนนี้" ให้เป็น `time` เฉพาะภายใน block นี้เท่านั้น ออกจาก block แล้วเวลากลับเป็นปกติอัตโนมัติ |
| `freeze_time { ... }` | เหมือน `travel_to(Time.current)` คือ "หยุดนาฬิกา" ไว้ ณ ขณะที่เรียก ป้องกันเวลาขยับแม้แต่มิลลิวินาทีเดียวระหว่าง block ทำงาน |
| `travel(duration) { ... }` | เลื่อนเวลาไปข้างหน้า/หลังตามระยะเวลาที่กำหนด (เช่น `travel(3.days)`) สัมพัทธ์กับเวลาปัจจุบันจริง |

### ทดสอบ `SubscriptionExpiryChecker` ด้วย `travel_to` (ทดสอบจริง)

```ruby
# frozen_string_literal: true

# spec/services/subscription_expiry_checker_spec.rb
require "rails_helper"

RSpec.describe SubscriptionExpiryChecker do
  describe "#expired?" do
    it "is false before the expiry date" do
      travel_to Time.zone.local(2026, 1, 1, 12, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.expired?).to be(false)
      end
    end

    it "is true after the expiry date" do
      travel_to Time.zone.local(2026, 1, 15, 12, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.expired?).to be(true)
      end
    end

    it "is true exactly at the expiry moment" do
      expiry = Time.zone.local(2026, 1, 10, 0, 0, 0)
      travel_to expiry do
        checker = described_class.new(expiry)
        expect(checker.expired?).to be(true)
      end
    end
  end

  describe "#days_remaining" do
    it "counts the number of days left, rounded up" do
      travel_to Time.zone.local(2026, 1, 1, 0, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 4, 0, 0, 0))
        expect(checker.days_remaining).to eq(3)
      end
    end

    it "returns 0 once expired" do
      travel_to Time.zone.local(2026, 2, 1) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.days_remaining).to eq(0)
      end
    end
  end
end
```

ผลลัพธ์จริงเมื่อรัน:

```bash
bundle exec rspec spec/services/subscription_expiry_checker_spec.rb --format documentation
```

```
SubscriptionExpiryChecker
  #expired?
    is false before the expiry date
    is true after the expiry date
    is true exactly at the expiry moment
  #days_remaining
    counts the number of days left, rounded up
    returns 0 once expired

Finished in 0.02589 seconds (files took 1.31 seconds to load)
5 examples, 0 failures
```

สังเกตว่าเทสต์นี้จะได้ผลลัพธ์เดียวกันทุกประการ **ไม่ว่าจะรันวันไหนก็ตาม** ไม่ว่าจะรันวันนี้
หรืออีก 5 ปีข้างหน้า เพราะ `Time.current` ภายใน block ของ `travel_to` ถูกตรึงไว้ที่ค่าที่
กำหนดเป๊ะๆ แล้ว — นี่คือความหมายของคำว่า "deterministic" ที่พูดถึงใน Step 481

> **ข้อควรระวัง:** ถ้าลืม include `ActiveSupport::Testing::TimeHelpers` จะได้ error
> `NoMethodError: undefined method 'travel_to'` ทันที (ทดสอบจริงแล้วเจอ error นี้ก่อนเพิ่ม
> `config.include` เข้าไป) เป็นสัญญาณที่ชัดเจนว่าลืมขั้นตอนนี้

### เทียบกับ Timecop gem

**Timecop** เป็น gem ยอดนิยมที่ทำหน้าที่คล้ายกันมาก และเคยเป็นตัวเลือกมาตรฐานของวงการ
ก่อนที่ Rails จะมี `ActiveSupport::Testing::TimeHelpers` ในตัว:

```ruby
# วิธีของ Timecop (ต้องเพิ่ม gem "timecop" ใน Gemfile ก่อน)
Timecop.freeze(Time.zone.local(2026, 1, 10)) do
  checker = SubscriptionExpiryChecker.new(Time.zone.local(2026, 1, 10))
  expect(checker.expired?).to be(true)
end

# หรือแบบไม่ใช้ block (ต้องเรียก Timecop.return เองเพื่อคืนค่าเวลาปกติ -- เสี่ยงลืม)
Timecop.freeze(Time.zone.local(2026, 1, 10))
# ... ทดสอบ ...
Timecop.return
```

| ประเด็น | `ActiveSupport::Testing::TimeHelpers` | Timecop |
|---|---|---|
| ต้องเพิ่ม gem หรือไม่ | ไม่ต้อง (มากับ Rails ในตัวอยู่แล้ว) | ต้องเพิ่ม `gem "timecop"` ใน Gemfile |
| Dependency เพิ่มในโปรเจกต์ | ไม่มี | มี 1 gem เพิ่ม ต้องดูแล/อัปเดตเวอร์ชันเอง |
| กลไกเบื้องหลัง | Monkey-patch `Time.now`/`Date.today` แบบเดียวกับ Timecop เป๊ะ | เหมือนกัน |
| การใช้งานพื้นฐาน | `travel_to`/`freeze_time`/`travel` | `Timecop.freeze`/`Timecop.travel` |
| Integration กับ Rails | แนบแน่นเพราะเป็นส่วนหนึ่งของ ActiveSupport อยู่แล้ว | ต้อง integrate เองเล็กน้อย (ปกติทำงานได้ดีอยู่แล้ว) |

**กฎปฏิบัติสำหรับ Rails 8:** ใช้ `ActiveSupport::Testing::TimeHelpers` (`travel_to`/
`freeze_time`) เป็นค่าเริ่มต้นเสมอ เพราะไม่ต้องเพิ่ม dependency ใหม่เข้าโปรเจกต์เลย ทำงาน
เหมือนกันทุกประการ และเป็นสิ่งที่ Rails core team ดูแลเองโดยตรง — เก็บ Timecop ไว้ในหัวแค่
เผื่อเจอโค้ด legacy ที่ใช้อยู่แล้วเท่านั้น ไม่ต้องเพิ่มใหม่ในโปรเจกต์ Rails 8 อีก

---

## Step 488: ติดตั้งและตั้งค่า `SimpleCov` — `require` บนสุดของ `spec_helper.rb`, `SimpleCov.start "rails"`

### SimpleCov คืออะไร

**SimpleCov** เป็น gem ที่วัด **test coverage** — สัดส่วนของบรรทัดโค้ดจริง (`app/`, `lib/`)
ที่ถูกรันอย่างน้อย 1 ครั้งระหว่างการรันเทสต์ทั้งหมด แล้วสร้างรายงาน HTML ให้ดูว่าไฟล์ไหน
บรรทัดไหนที่ "ไม่เคยถูกทดสอบแตะต้องเลย"

### ติดตั้ง

```ruby
# Gemfile
group :test do
  gem "simplecov", require: false
end
```

```bash
bundle install
```

> **ทำไมต้อง `require: false`:** เพราะ SimpleCov **ต้องเริ่มทำงาน (`.start`) ก่อนที่โค้ด
> แอปของเราจะถูก `require` เข้ามาแม้แต่บรรทัดเดียว** ถ้า Bundler auto-require ให้ตาม
> ลำดับปกติใน `Gemfile` อาจช้าเกินไป (โค้ดแอปบางส่วนถูกโหลดไปก่อนแล้ว) เราจึงต้อง `require`
> มันเองด้วยมือ ณ จุดที่แน่ใจว่าเป็น**จุดแรกสุด**ของกระบวนการโหลดทั้งหมด

### ตำแหน่งที่ถูกต้อง: บนสุดของ `spec_helper.rb` เท่านั้น

```ruby
# frozen_string_literal: true

# spec/spec_helper.rb
require "simplecov"
SimpleCov.start "rails"

# โค้ดเดิมทั้งหมดของ spec_helper.rb ที่ rspec --init สร้างให้ ตามด้วยบรรทัดนี้ต่อ...
RSpec.configure do |config|
  # ...
end
```

**เหตุผลที่ต้องอยู่ "บรรทัดแรกสุดของไฟล์แรกสุดที่ถูกโหลด":** SimpleCov ทำงานโดยการ hook
เข้ากับ Ruby's `Coverage` module (standard library) เพื่อนับว่าแต่ละบรรทัดของแต่ละไฟล์ถูก
`require`/execute กี่ครั้ง ถ้าไฟล์ `app/services/weather_service.rb` ถูก `require` ไปแล้ว
**ก่อน** ที่ `SimpleCov.start` จะทำงาน บรรทัดของไฟล์นั้นจะไม่ถูกนับเข้ารายงานเลย (เพราะพลาด
จังหวะเริ่มติดตาม) เนื่องจาก `spec/spec_helper.rb` ถูก require เป็นไฟล์แรกสุดเสมอ (ผ่าน
`.rspec` ที่มี `--require spec_helper`) การวาง `SimpleCov.start` ไว้บรรทัดบนสุดของไฟล์นี้
คือจุดที่เร็วที่สุดเท่าที่จะทำได้ในระบบของ RSpec

`SimpleCov.start "rails"` คือการเริ่มทำงานพร้อม **preset ของ Rails** ที่มากับ SimpleCov
เอง ซึ่งตั้งค่าที่เหมาะกับโปรเจกต์ Rails ให้อัตโนมัติ เช่น กรองไฟล์ `config/`, `db/`,
`spec/`/`test/` ออกจากรายงาน (เพราะไฟล์เหล่านี้ไม่ใช่โค้ด business logic ที่ต้องการวัด
coverage) และแบ่งกลุ่มไฟล์ในรายงานตามโครงสร้าง Rails (Models, Controllers, Services ฯลฯ)

### รันเทสต์แล้วดูรายงาน

```bash
bundle exec rspec
```

ผลลัพธ์จริง (ต่อท้ายผลการรันเทสต์ตามปกติ):

```
Finished in 0.03082 seconds (files took 1.15 seconds to load)
8 examples, 0 failures

Coverage report generated for RSpec to coverage/index.html
Line coverage: 23 / 27 (85.18%)
```

SimpleCov สร้างโฟลเดอร์ `coverage/` ขึ้นมาที่ root ของโปรเจกต์ พร้อมไฟล์ `index.html` ที่
เปิดดูด้วย browser ได้ทันที แสดงรายชื่อไฟล์ทั้งหมดพร้อมเปอร์เซ็นต์ coverage ของแต่ละไฟล์
และไฮไลต์สีที่บรรทัดโค้ดโดยตรง (สีเขียว = ถูกทดสอบแล้ว, สีแดง = ไม่เคยถูกรันเลยระหว่างเทสต์)

```bash
# .gitignore -- ต้องเพิ่มบรรทัดนี้เสมอ ห้าม commit โฟลเดอร์ coverage/ เข้า git
echo "coverage/" >> .gitignore
```

`coverage/` เป็นไฟล์ที่ generate ใหม่ได้ทุกครั้งที่รันเทสต์ ไม่ใช่ source code จึงไม่ควร
commit เข้า version control เหมือนกับที่ไม่ commit `log/` หรือ `tmp/`

### ตัวอย่างรายงานจริงเมื่อมีไฟล์ที่ไม่เคยถูกทดสอบเลย

จากการรันจริงในบทเรียนนี้ (สูทเทสต์มี `WeatherService`, `SubscriptionExpiryChecker` แต่
ยังไม่มีเทสต์ให้ controller ที่ Rails generate มาให้อัตโนมัติ):

```
Line coverage: 23 / 27 (85.18%)
```

ถ้าลอง drill-down เข้าไปดูรายไฟล์ (SimpleCov แสดงให้ในหน้า `coverage/index.html`) จะเห็น
ไฟล์ที่ 0% ชัดเจน:

```
app/controllers/application_controller.rb   0.00%
app/models/application_record.rb            0.00%
app/services/weather_service.rb           100.00%
app/services/subscription_expiry_checker.rb 100.00%
```

`application_controller.rb`/`application_record.rb` เป็นไฟล์ที่ Rails generate มาให้ตอน
`rails new` แต่ยังไม่มี logic อะไรให้เทสต์ (แค่ inherit จาก base class เฉยๆ) จึงเป็นเรื่อง
ปกติที่จะขึ้น 0% — SimpleCov แค่รายงานความจริงตามที่เป็น ไม่ได้บอกว่าไฟล์นั้น "แย่"
เสมอไป (ประเด็นนี้จะขยายความต่อใน Step 489)

---

## Step 489: อ่านรายงาน coverage อย่างมีวิจารณญาณ — 100% ไม่ได้แปลว่าไม่มีบั๊ก

### Coverage percentage วัดอะไรจริงๆ

**Line coverage** (ค่าที่ SimpleCov รายงานเป็นค่าเริ่มต้น) วัดแค่ **"บรรทัดนี้ถูกรันอย่าง
น้อย 1 ครั้งหรือไม่"** เท่านั้น — มันไม่รู้อะไรเลยเกี่ยวกับ:

- ค่าที่ได้จากการรันบรรทัดนั้นถูกต้องหรือไม่ (มี assertion ตรวจสอบจริงหรือเปล่า)
- Branch ทุกทางของ `if`/`case` ถูกทดสอบครบทุกทางหรือไม่ (line coverage นับแค่ว่าบรรทัด
  `if condition` ถูกรัน ไม่สนว่า `condition` เคยเป็น `true` และ `false` ครบทั้งคู่หรือยัง)
- Edge case ที่ควรทดสอบแต่ไม่มีใครนึกถึง (เพราะไม่มี code path ไหนรองรับ edge case นั้น
  เลยด้วยซ้ำ — coverage วัดได้แค่โค้ดที่ **มีอยู่แล้ว** เท่านั้น วัดสิ่งที่ **ควรมีแต่ไม่มี**
  ไม่ได้เลย)

### พิสูจน์ด้วยตัวอย่างจริง: 100% coverage ที่มีบั๊กร้ายแรงซ่อนอยู่

```ruby
# frozen_string_literal: true

# app/services/discount_calculator.rb
class DiscountCalculator
  def self.final_price(price, percentage)
    discount = price * percentage / 100
    price - discount + 1   # บั๊ก: ไม่ควรมี "+ 1" แต่บรรทัดนี้ยังถูก "รัน" ได้ปกติ
  end
end
```

```ruby
# frozen_string_literal: true

# ตัวอย่างเทสต์ "ไล่ตัวเลข coverage" -- ให้ 100% line coverage แต่ไม่จับบั๊กเลย
RSpec.describe DiscountCalculator do
  it "runs without error" do
    result = DiscountCalculator.final_price(200, 10)
    expect(result).not_to be_nil   # assertion อ่อนเกินไป แค่เช็คว่าไม่ใช่ nil
  end
end
```

รันจริงแล้วดูรายงาน coverage ของไฟล์นี้:

```
app/services/discount_calculator.rb   100.00%   (4/4 lines)
```

**ไฟล์นี้ได้ 100% line coverage เต็ม** ทุกบรรทัดถูกรันจริง (`class`, `def self.final_price`,
บรรทัดคำนวณ `discount`, บรรทัด `return` — SimpleCov นับ 4 บรรทัดที่ executable ครบทุกบรรทัด)
แต่บั๊ก `+ 1` ที่ทำให้ผลลัพธ์ผิดพลาดยังคงซ่อนอยู่โดยไม่มีใครจับได้ เพราะ:

- `DiscountCalculator.final_price(200, 10)` ควรได้ผลลัพธ์ `180` (200 ลด 10% = หัก 20 บาท
  เหลือ 180) แต่โค้ดจริงคืนค่า `181` เพราะบั๊ก `+ 1`
- Assertion `expect(result).not_to be_nil` **ผ่านเสมอไม่ว่าค่าที่ได้จะเป็นอะไรก็ตาม**
  ตราบใดที่ไม่ใช่ `nil` — เทสต์นี้ยืนยันแค่ว่า "method รันได้โดยไม่ crash" ไม่ได้ยืนยันว่า
  "method คำนวณถูกต้อง" เลยแม้แต่นิดเดียว

เขียน assertion ที่ตรวจสอบค่าจริงแทน จะจับบั๊กนี้ได้ทันที:

```ruby
RSpec.describe DiscountCalculator do
  it "คำนวณราคาหลังหักส่วนลดได้ถูกต้อง" do
    result = DiscountCalculator.final_price(200, 10)
    expect(result).to eq(180)   # assertion ที่ตรวจค่าจริง -- จะ FAIL ทันทีเพราะได้ 181
  end
end
```

```
Failures:

  1) DiscountCalculator คำนวณราคาหลังหักส่วนลดได้ถูกต้อง
     Failure/Error: expect(result).to eq(180)

       expected: 180
            got: 181
```

**ทั้งสองเวอร์ชันของเทสต์ได้ 100% line coverage เท่ากันเป๊ะ** (รันโค้ดชุดเดียวกันทุก
บรรทัด) แต่มีแค่เวอร์ชันที่ 2 เท่านั้นที่จับบั๊กได้ นี่คือหลักฐานที่ชัดเจนที่สุดว่า
**coverage percentage วัดแค่ "โค้ดถูกรันหรือยัง" ไม่ได้วัด "assertion ตรวจสอบถูกต้องแค่
ไหน" เลย**

### กฎสำคัญ: Coverage คือ "พื้น" (floor) ไม่ใช่ "เพดาน" (ceiling)

- **100% coverage ไม่ได้แปลว่าโค้ดไม่มีบั๊ก** — มันแค่แปลว่า "ทุกบรรทัดเคยถูกรันแล้วอย่าง
  น้อย 1 ครั้งระหว่างการทดสอบ" เท่านั้น คุณภาพของ assertion ยังต้องอาศัยวิจารณญาณของคนเขียน
  เทสต์อยู่ดี ไม่มีตัวเลขไหนวัดแทนได้
- **0% coverage แปลว่าไม่มีการทดสอบเลยแน่นอน** (ทิศทางตรงข้ามเป็นความจริงเสมอ) — จุดนี้คือ
  ประโยชน์ที่แท้จริงของ coverage: **ใช้หาไฟล์/บรรทัดที่ไม่เคยถูกแตะต้องเลย** ไม่ใช่ใช้วัด
  ว่า "เทสต์ที่มีอยู่ดีพอหรือยัง"
- **อันตรายของการ "ไล่ตัวเลข coverage"** — ถ้าทีมตั้งเป้าว่า "ต้องได้ 100% coverage" โดยไม่
  เข้าใจข้อจำกัดนี้ นักพัฒนาจะเขียนเทสต์แบบ `discount_calculator_spec.rb` เวอร์ชันแรก
  (แค่เรียก method ให้ผ่าน ไม่ตรวจ assertion จริงจัง) เพื่อไล่ตัวเลขให้ถึงเป้า ผลคือได้
  ตัวเลขสวยงามบนหน้าจอ CI แต่ **ความมั่นใจในคุณภาพจริงกลับต่ำลง** เพราะทีมเริ่มเชื่อว่า
  "100% coverage = ปลอดภัยแล้ว" ทั้งที่ไม่จริงเลย

**สรุปแนวคิดที่ถูกต้อง:** ใช้ coverage เป็นเครื่องมือ**ค้นหาพื้นที่ที่ไม่มีเทสต์เลย** (ซึ่ง
มีประโยชน์จริง — ไฟล์ที่ 0% คือสัญญาณเตือนที่ชัดเจนว่าไม่มีความมั่นใจอะไรเลยกับโค้ดส่วนนั้น)
แต่อย่าใช้ตัวเลข coverage เป็น**เป้าหมายสุดท้าย**ของคุณภาพเทสต์ — เป้าหมายที่แท้จริงคือ
เทสต์ที่ **ตรวจสอบ behavior ที่ถูกต้องจริงๆ** ด้วย assertion ที่มีความหมาย ไม่ใช่แค่เทสต์
ที่ทำให้ตัวเลขสวยขึ้น

---

## Step 490: `SimpleCov.minimum_coverage` ใน CI — safety net ไม่ใช่เป้าหมาย

### ตั้งเกณฑ์ขั้นต่ำที่ยอมรับได้

แม้ Step 489 จะเตือนว่าอย่าไล่ตัวเลข coverage แต่การตั้ง **เกณฑ์ขั้นต่ำ** ยังมีประโยชน์
มาก — ในฐานะ **safety net** ที่คอยจับกรณี "โค้ดใหม่ถูก merge เข้ามาโดยไม่มีเทสต์คุ้มครอง
เลยแม้แต่น้อย" ไม่ใช่ในฐานะเป้าหมายที่ต้องไล่ให้ถึง 100%

```ruby
# frozen_string_literal: true

# spec/spec_helper.rb
require "simplecov"
SimpleCov.start "rails" do
  minimum_coverage 80
end
```

เมื่อรันเทสต์แล้ว coverage ต่ำกว่าเกณฑ์ที่ตั้งไว้ SimpleCov จะทำให้ process จบด้วย **exit
code ที่ไม่ใช่ 0** (คือถือว่า "ล้มเหลว" ในสายตาของ CI) แม้ว่าเทสต์ทุกตัวจะ "ผ่าน" หมดก็ตาม
— ทดสอบจริงโดยตั้งเกณฑ์ไว้ที่ 80% ในขณะที่สูทเทสต์จริงได้แค่ 78.94% (ก่อนจะเพิ่มเทสต์ให้
`SubscriptionExpiryChecker` เข้ามาช่วยดันตัวเลขขึ้น):

```
Finished in 0.03088 seconds (files took 1.31 seconds to load)
1 example, 0 failures

Coverage report generated for RSpec to coverage/index.html
Line coverage: 15 / 19 (78.94%)
Line coverage (78.94%) is below the expected minimum coverage (80.00%).
  Lowest-coverage files (line):
      0.00%  app/controllers/application_controller.rb
      0.00%  app/models/application_record.rb
SimpleCov failed with exit 2 due to a coverage related error
```

สังเกตว่า RSpec เองรายงาน `1 example, 0 failures` (เทสต์ทั้งหมดผ่าน) แต่ SimpleCov ยังคง
ทำให้ exit code ไม่ใช่ 0 — นี่คือกลไกที่ทำให้ **CI pipeline หยุด build ได้** แม้ไม่มีเทสต์
ไหน fail เลยสักตัว เพราะ CI ตรวจสอบ exit code ของคำสั่งสุดท้าย ไม่ใช่แค่ข้อความ "failures"

หลังเพิ่มเทสต์ให้ `SubscriptionExpiryChecker` และ `WeatherService` ครบแล้ว รันสูทเทสต์
ทั้งหมดใหม่:

```
Finished in 0.03082 seconds (files took 1.15 seconds to load)
8 examples, 0 failures

Coverage report generated for RSpec to coverage/index.html
Line coverage: 23 / 27 (85.18%)
```

85.18% ผ่านเกณฑ์ 80% แล้ว — SimpleCov จบด้วย exit code 0 ปกติ ทำให้ CI ผ่าน build ต่อไปได้

### ตัวอย่างการเชื่อมกับ CI (เกริ่นไว้ก่อน จะเรียนเต็มใน Part 075)

```yaml
# .github/workflows/test.yml (ตัวอย่างคร่าวๆ)
- name: Run RSpec with coverage
  run: bundle exec rspec
  # ถ้า SimpleCov.minimum_coverage ไม่ผ่าน exit code จะไม่ใช่ 0
  # GitHub Actions จะ mark step นี้ว่า failed อัตโนมัติโดยไม่ต้องเขียนเช็คเพิ่มเอง
```

### `minimum_coverage_by_file` — เกณฑ์ต่อไฟล์ (เข้มงวดกว่า)

`minimum_coverage` ตรวจแค่**ค่าเฉลี่ยรวมทั้งโปรเจกต์** ซึ่งมีจุดอ่อน: ไฟล์ที่มี coverage
สูงมากไฟล์หนึ่ง (เช่น model ง่ายๆ ที่เทสต์ครบ 100%) สามารถ "กลบ" ไฟล์ที่มี coverage ต่ำมาก
อีกไฟล์หนึ่งได้ (เช่น service object ใหม่ที่ยังไม่มีเทสต์เลย 0%) ทำให้ค่าเฉลี่ยรวมยังผ่าน
เกณฑ์อยู่ทั้งที่มีจุดเสี่ยงจริงซ่อนอยู่

```ruby
SimpleCov.start "rails" do
  minimum_coverage 80
  minimum_coverage_by_file 60   # ทุกไฟล์ต้องได้อย่างน้อย 60% เป็นรายไฟล์ ไม่ใช่แค่ค่าเฉลี่ยรวม
end
```

### กฎปฏิบัติสำหรับการตั้งเกณฑ์

1. **อย่าตั้ง 100%** — ตามเหตุผลทั้งหมดใน Step 489 การไล่ 100% มักนำไปสู่เทสต์คุณภาพต่ำ
   ที่เขียนขึ้นมาเพื่อไล่ตัวเลขเท่านั้น
2. **เริ่มจากตัวเลขปัจจุบันของโปรเจกต์จริง** — ถ้าโปรเจกต์มี coverage อยู่แล้ว 65% ให้ตั้ง
   เกณฑ์ไว้ที่ใกล้เคียงตัวเลขนั้น (เช่น 60–65%) ไม่ใช่กระโดดไปตั้งที่ 90% ทันที เพราะจะทำให้
   CI แดงตลอดเวลาโดยไม่มีใครแก้จริงจัง (คนจะเริ่มมองข้าม CI ที่แดงตลอด — เสียจุดประสงค์)
3. **ปรับเกณฑ์ขึ้นทีละน้อยเมื่อ coverage ดีขึ้นตามธรรมชาติ** — ไม่ใช่ตั้งเกณฑ์สูงแล้วไล่
   เขียนเทสต์ให้ทัน แต่ให้เกณฑ์ **ตามหลัง** ความคืบหน้าจริงของทีม เพื่อป้องกันไม่ให้ตัวเลข
   ที่เคยดีขึ้นแล้วถอยหลังลงมาโดยไม่มีใครสังเกต (เช่น มี PR ใหม่เพิ่มโค้ด 200 บรรทัดโดยไม่มี
   เทสต์เลย ทำให้ค่าเฉลี่ยรวมตกจาก 85% เหลือ 78% — เกณฑ์ที่ตั้งไว้ที่ 80% จะจับความผิดปกติ
   นี้ได้ทันทีที่ CI รัน)
4. **มองว่าเกณฑ์นี้คือ "เส้นเตือนภัย" ไม่ใช่ "ใบรับรองคุณภาพ"** — ผ่านเกณฑ์แล้วไม่ได้แปลว่า
   เทสต์ดีพอแล้ว การรีวิวโค้ด (code review) และดุลยพินิจของทีมยังจำเป็นเสมอ ไม่มีตัวเลข
   อัตโนมัติตัวไหนแทนที่การอ่านเทสต์จริงด้วยสายตาคนได้

---

## แบบฝึกหัด: `WeatherService` + `SubscriptionExpiryChecker` ครบวงจร พร้อมรายงาน SimpleCov

### โจทย์

สร้างแอป Rails ใหม่ (หรือใช้แอปเดิมจาก Part ก่อนๆ) แล้วทำตามขั้นตอนต่อไปนี้ให้ครบ:

1. เพิ่ม gem `vcr`, `webmock`, `simplecov` ลง `Gemfile` กลุ่ม `:test`
2. ตั้งค่า `SimpleCov.start "rails"` พร้อม `minimum_coverage 80` ที่บรรทัดบนสุดของ
   `spec/spec_helper.rb`
3. ตั้งค่า `VCR.configure` ใน `spec/support/vcr.rb` ให้ครบ (`cassette_library_dir`,
   `hook_into :webmock`, `configure_rspec_metadata!`, `filter_sensitive_data`)
4. เขียน `WeatherService` (ตามโค้ดใน Step 486) พร้อม spec ที่ทดสอบทั้งเคสสำเร็จและเคส
   error โดยใช้ VCR cassette จริง
5. เขียน `SubscriptionExpiryChecker` (ตามโค้ดใน Step 487) พร้อม spec ที่ใช้ `travel_to`
   ครอบคลุมทั้ง `#expired?` และ `#days_remaining`
6. รัน `bundle exec rspec` ทั้งโปรเจกต์ แล้วอ่านรายงาน coverage ที่ได้

### เฉลย

**Gemfile:**

```ruby
group :test do
  gem "rspec-rails", "~> 7.0"
  gem "vcr", "~> 6.3"
  gem "webmock", "~> 3.24"
  gem "simplecov", require: false
end
```

**`spec/spec_helper.rb` (ส่วนที่เพิ่มเข้าไปบนสุด):**

```ruby
# frozen_string_literal: true

require "simplecov"
SimpleCov.start "rails" do
  minimum_coverage 80
end

# ... โค้ดเดิมของ spec_helper.rb ต่อจากนี้
```

**`spec/support/vcr.rb`:**

```ruby
# frozen_string_literal: true

require "vcr"
require "webmock/rspec"

VCR.configure do |config|
  config.cassette_library_dir = "spec/cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data("<WEATHER_API_KEY>") { ENV.fetch("WEATHER_API_KEY", "dummy-key") }
  config.default_cassette_options = { record: :once }
end
```

**`spec/rails_helper.rb` (ส่วนที่ต้องเพิ่ม/เปิด comment):**

```ruby
Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }

RSpec.configure do |config|
  config.include ActiveSupport::Testing::TimeHelpers

  # ... config อื่นๆ ตามเดิม
end
```

**`app/services/weather_service.rb`:**

```ruby
# frozen_string_literal: true

require "net/http"
require "json"

class WeatherApiError < StandardError; end

class WeatherService
  ENDPOINT = "https://api.weatherapi.com/v1/current.json"

  def initialize(api_key: Rails.application.credentials.dig(:weather_api, :key))
    @api_key = api_key
  end

  def current_temperature(city)
    uri = URI(ENDPOINT)
    uri.query = URI.encode_www_form(key: @api_key, q: city)

    response = Net::HTTP.get_response(uri)
    raise WeatherApiError, "WeatherAPI returned #{response.code}" unless response.is_a?(Net::HTTPSuccess)

    data = JSON.parse(response.body)
    data.dig("current", "temp_c")
  end
end
```

**`spec/services/weather_service_spec.rb`:**

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe WeatherService do
  subject(:service) { described_class.new(api_key: "dummy-key") }

  describe "#current_temperature" do
    context "เมื่อ API ตอบกลับสำเร็จ", vcr: { cassette_name: "weather_service/bangkok_success" } do
      it "คืนอุณหภูมิปัจจุบันเป็นตัวเลข" do
        expect(service.current_temperature("Bangkok")).to eq(32.5)
      end
    end

    context "เมื่อ API ตอบกลับด้วย error status", vcr: { cassette_name: "weather_service/invalid_city" } do
      it "raise WeatherApiError" do
        expect { service.current_temperature("InvalidCityXYZ") }.to raise_error(WeatherApiError, /400/)
      end
    end
  end
end
```

**`app/services/subscription_expiry_checker.rb`:**

```ruby
# frozen_string_literal: true

class SubscriptionExpiryChecker
  def initialize(expires_at)
    @expires_at = expires_at
  end

  def expired?
    Time.current >= @expires_at
  end

  def days_remaining
    return 0 if expired?

    ((@expires_at - Time.current) / 1.day).ceil
  end
end
```

**`spec/services/subscription_expiry_checker_spec.rb`:**

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe SubscriptionExpiryChecker do
  describe "#expired?" do
    it "is false before the expiry date" do
      travel_to Time.zone.local(2026, 1, 1, 12, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.expired?).to be(false)
      end
    end

    it "is true after the expiry date" do
      travel_to Time.zone.local(2026, 1, 15, 12, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.expired?).to be(true)
      end
    end

    it "is true exactly at the expiry moment" do
      expiry = Time.zone.local(2026, 1, 10, 0, 0, 0)
      travel_to expiry do
        checker = described_class.new(expiry)
        expect(checker.expired?).to be(true)
      end
    end
  end

  describe "#days_remaining" do
    it "counts the number of days left, rounded up" do
      travel_to Time.zone.local(2026, 1, 1, 0, 0, 0) do
        checker = described_class.new(Time.zone.local(2026, 1, 4, 0, 0, 0))
        expect(checker.days_remaining).to eq(3)
      end
    end

    it "returns 0 once expired" do
      travel_to Time.zone.local(2026, 2, 1) do
        checker = described_class.new(Time.zone.local(2026, 1, 10))
        expect(checker.days_remaining).to eq(0)
      end
    end
  end
end
```

**รันทั้งสูท:**

```bash
bundle exec rspec --format documentation
```

**ผลลัพธ์จริงที่ได้ (ทดสอบตามขั้นตอนนี้ทุกจุด):**

```
SubscriptionExpiryChecker
  #expired?
    is false before the expiry date
    is true after the expiry date
    is true exactly at the expiry moment
  #days_remaining
    counts the number of days left, rounded up
    returns 0 once expired

WeatherService
  #current_temperature
    เมื่อ API ตอบกลับสำเร็จ
      คืนอุณหภูมิปัจจุบันเป็นตัวเลข
    เมื่อ API ตอบกลับด้วย error status
      raise WeatherApiError

Finished in 0.03082 seconds (files took 1.15 seconds to load)
8 examples, 0 failures

Coverage report generated for RSpec to coverage/index.html
Line coverage: 23 / 27 (85.18%)
```

85.18% ผ่านเกณฑ์ `minimum_coverage 80` ที่ตั้งไว้ SimpleCov จบด้วย exit code 0 ปกติ

### สิ่งที่ได้ฝึกจากเฉลยนี้

- แยกได้ชัดว่าเมื่อไหร่ต้องใช้ `instance_double` (Part 019), เมื่อไหร่ต้องใช้
  `stub_request`/VCR (dependency ที่เป็น HTTP ดิบ), และเมื่อไหร่ต้องใช้ `travel_to`
  (dependency ที่เป็นเวลาปัจจุบัน) — ทั้งสามเป็นเครื่องมือคนละแบบสำหรับปัญหาคนละประเภท
  ตามหลักการ 4 ข้อของ Step 481
- เห็นวงจรเต็มของ VCR ตั้งแต่การตั้งค่า, การบันทึก cassette จริงครั้งแรก, การเล่นซ้ำโดยไม่
  ต้องพึ่งพา network อีกเลย, ไปจนถึงการป้องกัน API key รั่วไหลด้วย `filter_sensitive_data`
- ใช้ `ActiveSupport::Testing::TimeHelpers` ทดสอบ logic ที่พึ่งพาเวลาได้อย่างแน่นอน
  (deterministic) โดยไม่ต้องเพิ่ม gem ใหม่เข้าโปรเจกต์เลย
- ตั้งค่า `SimpleCov.minimum_coverage` เป็น safety net ใน CI พร้อมเข้าใจว่าตัวเลข coverage
  ที่ได้เป็นแค่ "พื้น" ไม่ใช่ "เพดาน" ของคุณภาพเทสต์

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม method `CurrencyExchangeService#convert(amount, from:, to:)` ที่เรียก external
   API ตัวอย่าง (เช่น จำลอง `https://api.exchangerate-example.com/latest`) คืนค่าจำนวนเงิน
   ที่แปลงแล้ว เขียนสเปคด้วย VCR ให้ครอบคลุมทั้งเคสสำเร็จ, เคส currency code ที่ไม่รู้จัก
   (API ตอบ 404), และเคส API ล่ม (ตอบ 500) — ฝึกตั้งชื่อ cassette แยกตามสถานการณ์ให้อ่าน
   แล้วเข้าใจทันทีว่าแต่ละไฟล์ทดสอบอะไร (เช่น `currency_exchange/success.yml`,
   `currency_exchange/unknown_currency.yml`, `currency_exchange/server_error.yml`)

2. เขียน class `TrialPeriodPolicy` ที่รับวันที่สมัครสมาชิก (`signed_up_at`) และมี method
   `#trial_active?` (คืน `true` ถ้ายังอยู่ในช่วงทดลองใช้ 14 วันนับจากวันที่สมัคร) และ
   `#grace_period_active?` (คืน `true` ถ้าพ้นช่วงทดลองใช้แล้วแต่ยังอยู่ในช่วงผ่อนผันอีก 3
   วันถัดมา) เขียนสเปคด้วย `travel_to` ให้ครอบคลุมทั้ง 4 สถานะที่เป็นไปได้: อยู่ในช่วง
   ทดลองใช้, วันสุดท้ายของช่วงทดลองใช้พอดี, อยู่ในช่วงผ่อนผัน, และพ้นช่วงผ่อนผันแล้วโดย
   สมบูรณ์

3. รันสูทเทสต์ทั้งหมดของแบบฝึกหัดข้อ 1–2 พร้อมกับ `WeatherService`/
   `SubscriptionExpiryChecker` จาก Part นี้ แล้วลองตั้ง `minimum_coverage_by_file 70`
   เพิ่มเข้าไปในการตั้งค่า SimpleCov — สังเกตว่าไฟล์ไหนที่ทำให้ CI แดงเพราะ coverage
   รายไฟล์ไม่ถึงเกณฑ์ แม้ค่าเฉลี่ยรวมทั้งโปรเจกต์จะผ่าน `minimum_coverage` หลักก็ตาม แล้ว
   เขียนเทสต์เพิ่มเฉพาะจุดนั้นให้ผ่านเกณฑ์

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ทบทวนและเจาะลึกกฎการเลือก mock/stub vs ของจริง: mock เมื่อ dependency ช้า/แพง/ไม่แน่นอน/
  อยู่นอกการควบคุม และห้าม mock ตัว subject ที่กำลังทดสอบเองเด็ดขาด (anti-pattern ที่ทำให้
  เทสต์ผ่านเสมอโดยไม่ทดสอบอะไรจริง)
- เข้าใจข้อจำกัดของ `instance_double` เมื่อเจอ HTTP request ดิบ และใช้ **WebMock**
  (`stub_request`) สกัดกั้น network request ที่ระดับ HTTP protocol ได้
- ใช้ **VCR** บันทึก HTTP interaction จริงลง "cassette" ครั้งเดียว แล้วเล่นซ้ำได้ตลอดไป
  โดยไม่ต้องพึ่งพา network จริงอีกเลย พิสูจน์แล้วว่าเทสต์ยังผ่านได้แม้ปิด server ต้นทางทิ้ง
- ตั้งค่า `VCR.configure` ครบ: `cassette_library_dir`, `hook_into :webmock`, record mode
  ทั้ง 3 แบบ (`:once`/`:new_episodes`/`:none`) และรู้ว่าแบบไหนเหมาะกับสถานการณ์ไหน
- ป้องกัน API key รั่วไหลเข้า cassette ด้วย `filter_sensitive_data` ก่อน commit เข้า git
- เขียน service object เรียก external API จริงพร้อมชุดเทสต์ VCR ครบทั้งเคสสำเร็จและ error
- ใช้ `ActiveSupport::Testing::TimeHelpers` (`travel_to`/`freeze_time`/`travel`) ทดสอบ
  logic ที่พึ่งพาเวลาได้แบบ deterministic โดยไม่ต้องเพิ่ม gem Timecop เข้าโปรเจกต์ Rails 8
- ติดตั้งและตั้งค่า `SimpleCov.start "rails"` ที่บรรทัดบนสุดของ `spec_helper.rb` และเข้าใจ
  ว่าทำไมตำแหน่งนี้สำคัญมาก
- อ่านและตีความรายงาน coverage อย่างมีวิจารณญาณ พิสูจน์ด้วยตัวอย่างจริงว่า 100% line
  coverage ไม่ได้แปลว่าไม่มีบั๊ก และเข้าใจอันตรายของการ "ไล่ตัวเลข coverage" ด้วยเทสต์ที่
  assertion อ่อนเกินไป
- ตั้ง `SimpleCov.minimum_coverage`/`minimum_coverage_by_file` เป็น safety net ใน CI
  พร้อมกฎปฏิบัติที่ถูกต้อง: เริ่มจากตัวเลขจริงของโปรเจกต์ ปรับขึ้นทีละน้อย และมองว่าเป็น
  เส้นเตือนภัยไม่ใช่ใบรับรองคุณภาพ

**ต่อไป (Part 050):** เราจะปิดท้าย **เฟส 6: Testing (TDD/BDD)** ด้วย Part ที่เป็นบทสรุป
รวบยอดของทั้งเฟส — **TDD workflow เต็มรูปแบบ: Red-Green-Refactor** โดยจะหยิบฟีเจอร์ใหม่
ขึ้นมา 1 ฟีเจอร์ แล้วเขียนโค้ดตามลำดับ **Red** (เขียนเทสต์ที่ fail ก่อนมี implementation) →
**Green** (เขียน implementation ให้น้อยที่สุดเท่าที่จะทำให้เทสต์ผ่าน) → **Refactor** (ปรับ
โครงสร้างโค้ดให้สะอาดขึ้นโดยเทสต์ยังผ่านเหมือนเดิม) แบบวนซ้ำจนฟีเจอร์สมบูรณ์ — นำทุกเทคนิค
จาก Part 046–049 ทั้งเฟส (RSpec สำหรับ Rails, FactoryBot, Capybara, mocking/VCR/coverage)
มาประกอบร่างเป็น workflow การทำงานจริงที่ทีมมืออาชีพใช้ทุกวัน ก่อนจะปิดเฟส 6 แล้วเข้าสู่
เฟส 7 เรื่อง Frontend/Hotwire/Stimulus ต่อไป
