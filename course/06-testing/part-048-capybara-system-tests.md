# Part 048: Feature Test / System Test ด้วย Capybara

> **Step ครอบคลุมใน Part นี้:** Step 471–480
> **ระดับ:** ปานกลาง–สูง (ต่อจาก Part 046 เรื่อง RSpec model/request spec และ Part 047 เรื่อง FactoryBot/Faker)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / rspec-rails 7.1.x / capybara 3.40.x /
> selenium-webdriver 4.49.x (ตัวอย่างทั้งหมดทดสอบจริงบน Rails 8.1.4)

จาก Part 046 เราเขียน **model spec** (ทดสอบ validation/business logic ของ `ActiveRecord`
โดยตรง ไม่ผ่าน HTTP เลย) และ **request spec** (ยิง HTTP request จำลองเข้า controller
ตรวจสอบ status code, response body, redirect) มาแล้ว ส่วน Part 047 สอนให้ใช้ **FactoryBot**
สร้างข้อมูลทดสอบแทนการเขียน `User.create!(...)` ยาวๆ ซ้ำทุกที่

Part นี้ปิดช่องว่างสุดท้ายของพีระมิดการทดสอบ (Testing Pyramid) ที่ทบทวนไว้ใน Part 046: ชั้น
บนสุด — **System Test** (หรือชื่ออื่นที่ความหมายเดียวกันคือ **Feature Test** ในวงการ
Cucumber/Capybara) คือเทสที่ทำงาน "เหมือนผู้ใช้จริง" ที่สุด: เปิดหน้าเว็บ กรอกฟอร์ม กดปุ่ม
คลิกลิงก์ ผ่าน **เบราว์เซอร์** (จริงหรือจำลอง) แล้วตรวจสอบว่าสิ่งที่ปรากฏบนหน้าจอถูกต้อง
ไม่ได้เรียก method บน object ตรงๆ เหมือน spec ประเภทอื่นเลย

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างโค้ดและผลลัพธ์เทอร์มินัลใน Part นี้มาจากการสร้าง
> แอป Rails ทดลองจริงชื่อ `capybara_demo` (มี `User` ที่ login ได้ด้วย `has_secure_password`
> ต่อยอดจาก Part 041, และ `Post`/`Comment` ต่อยอดจากเฟส 3–4) ติดตั้ง `rspec-rails`,
> `factory_bot_rails`, `capybara`, `selenium-webdriver` แล้วรัน `bundle exec rspec` จริงทุก
> คำสั่ง ไม่มีผลลัพธ์ใดที่เขียนขึ้นลอยๆ — รวมถึง error message ที่เกิดจากการพิมพ์ผิดโดยตั้งใจ
> เพื่อสาธิตการ debug ก็เป็นข้อความ error จริงที่ RSpec/Capybara พ่นออกมา

## สารบัญของ Part นี้

- Step 471: System Test คืออะไร และทำไมอยู่บนสุดของ Testing Pyramid
- Step 472: สร้าง System Test แรกด้วย `rails generate rspec:system`
- Step 473: `rack_test` (เร็ว ไม่รัน JS) เทียบกับ `selenium_chrome_headless` (เบราว์เซอร์จริง รัน JS ได้)
- Step 474: Capybara DSL พื้นฐาน — `visit`, `click_link`, `click_button`, `fill_in`
- Step 475: ฟอร์มที่ซับซ้อนขึ้น — `select`, `check`/`uncheck`, `choose`
- Step 476: Assertion ของ Capybara — `have_content`, `have_selector`, `have_css`
- Step 477: กลไก auto-waiting ของ Capybara กับ anti-pattern การใช้ `sleep`
- Step 478: จำกัดขอบเขตการค้นหาด้วย `within`
- Step 479: Debug system test ที่ fail — `save_and_open_page`, `save_screenshot`, `binding.pry`
- Step 480: แบบฝึกหัด — User Journey เต็มรูปแบบ และควรเขียน System Test กี่ตัวถึงจะพอดี

---

## Step 471: System Test คืออะไร และทำไมอยู่บนสุดของ Testing Pyramid

### ทบทวน Testing Pyramid จาก Part 046

Part 046 แนะนำแนวคิด **Testing Pyramid** ไว้แบบสั้นๆ ตอนแนะนำ model spec กับ request spec —
Part นี้ขอทบทวนให้ครบภาพก่อนเริ่ม เพราะ System Test คือชั้นบนสุดที่พีระมิดพูดถึง

| ชั้น | ตัวอย่างที่เรียนมาแล้ว | ความเร็ว (โดยประมาณ) | ทดสอบผ่านชั้นอะไรบ้าง | ควรมีจำนวน |
|---|---|---|---|---|
| **Model spec** | Part 046 | เร็วที่สุด (มิลลิวินาที/ตัว) | เรียก method บน `ActiveRecord` object ตรงๆ ไม่ผ่าน router/controller/view เลย | เยอะที่สุด |
| **Request spec** | Part 046 | เร็ว (ต้อง boot Rack request/response) | Router → Controller → View render — แต่ **ไม่มี browser**, ไม่คลิกลิงก์/กรอกฟอร์มเอง, เรียก `get`/`post` ตรงๆ | ปานกลาง |
| **System spec** (Part นี้) | Part 048 | ช้าที่สุด (เป็นวินาทีต่อตัว) | Router → Controller → View → **ผ่านเบราว์เซอร์จริงหรือจำลอง** → session/cookie → (ถ้าใช้ browser จริง) JavaScript ด้วย | น้อยที่สุด |

จุดที่ทำให้ System spec ช้ากว่า request spec แม้จะวิ่งผ่าน stack เดียวกันคือ:

1. **ต้องเรนเดอร์ HTML เต็มหน้าจริง** แล้วให้ Capybara แปลง (parse) กลับมาเป็นโครงสร้างที่
   ค้นหา element ได้ (request spec ก็ render view เหมือนกัน แต่ไม่มีขั้นตอน parse/ค้นหา
   element ต่อ)
2. **จำลองพฤติกรรมผู้ใช้ทีละขั้น** — กว่าจะถึงหน้าที่ต้องการทดสอบ อาจต้อง `visit` หลายหน้า,
   กรอกฟอร์ม, กด submit ตามลำดับเดียวกับที่คนจริงทำ ไม่ใช่ยิง request เดียวจบแบบ request spec
3. **ถ้าใช้ browser จริง (Selenium)** ยังมีต้นทุนการบูตเบราว์เซอร์ทั้งตัว (process แยก)
   และรอ JavaScript ทำงานเพิ่มเข้าไปอีก (รายละเอียดเรื่องนี้ดูใน Step 473)

### แล้วทำไมยังต้องมี System Test — request spec ยังไม่พอเหรอ

Request spec ตอบคำถามได้แค่ "controller คืน status/response ที่ถูกต้องไหม" แต่ **ตอบไม่ได้**
ว่า:

- ผู้ใช้จริง **คลิกปุ่มที่ถูกต้อง** แล้วจะไปหน้าที่ตั้งใจไหม (ชื่อปุ่ม/ลิงก์ผิด แต่ route
  ยังถูกอยู่ — request spec ตรวจไม่เจอ เพราะไม่ได้ "คลิก" อะไรจริง)
- ฟอร์มที่ประกอบด้วยหลาย field (`select`, checkbox, radio) เชื่อมกับ `name=""` ที่ Rails
  form helper generate มาให้ **ถูกต้องตรงกับที่ view ตั้งใจ** ไหม
- ถ้าเป็นหน้าที่มี JavaScript (Turbo Frame, Stimulus — เฟส 7) การ interact กับหน้าเว็บ
  **หลังจาก JS ทำงานเสร็จ** แล้วให้ผลลัพธ์ตามที่ควรเป็นไหม (request spec ไม่รัน JS เลย)
- **ลำดับขั้นตอนข้าม request หลายครั้งของ user คนเดียว** (signup → login → สร้างโพสต์ →
  เห็นในรายการ) ทำงานต่อเนื่องกันได้จริงไหม เพราะแต่ละ request spec ทดสอบแยกอิสระจากกัน
  ปกติจะไม่เขียนให้ต่อเนื่องแบบนี้

พูดสั้นๆ: **System test คือเทสระดับเดียวที่ตอบคำถาม "ผู้ใช้จริงใช้งานฟีเจอร์นี้ได้จริงไหม"**
ส่วนเทสชั้นล่างๆ ตอบคำถาม "แต่ละส่วนย่อยทำงานถูกต้องไหม" — ทั้งสองแบบจำเป็นทั้งคู่ แต่ด้วย
สัดส่วนที่ต่างกันมาก (รายละเอียดเรื่องสัดส่วนที่เหมาะสมอยู่ใน Step 480)

> **คำศัพท์ที่ใช้แทนกันได้:** ในเอกสารของ Rails เอง (`ActionDispatch::SystemTestCase`) และ
> `rspec-rails` (`type: :system`) เรียกว่า **System Test** ส่วนในวงการ Ruby ที่มาจาก
> Cucumber/RSpec แต่เดิมจะเรียกว่า **Feature Test** (`type: :feature`, เขียนในโฟลเดอร์
> `spec/features/`) ทั้งสองคำหมายถึงสิ่งเดียวกัน — เทสที่ขับเคลื่อนด้วย Capybara ผ่าน
> "หน้าเว็บ" ไม่ใช่เรียก method ตรงๆ Part นี้ใช้คำว่า System Test เป็นหลักตามธรรมเนียมของ
> Rails 8 + rspec-rails รุ่นปัจจุบัน

---

## Step 472: สร้าง System Test แรกด้วย `rails generate rspec:system`

### ติดตั้ง gem ที่จำเป็น

นอกจาก `rspec-rails` และ `factory_bot_rails` ที่ติดตั้งไว้แล้วจาก Part 046–047 ต้องเพิ่ม
`capybara` และ `selenium-webdriver` ในกลุ่ม `:test` ของ `Gemfile` (แอป Rails 8 ที่สร้างด้วย
`rails new` ปกติจะมี 2 gem นี้ comment ไว้ให้อยู่แล้วในกลุ่ม `:test` เพราะเตรียมไว้สำหรับ
ระบบ Minitest system test ที่ Rails สร้างให้โดย default แต่ในหลักสูตรนี้ที่ใช้ RSpec ต้อง
ประกาศเองให้ชัดเจน):

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails", "~> 7.1"
  gem "factory_bot_rails"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
end
```

```bash
bundle install
```

### ตั้งค่า `spec/rails_helper.rb` ให้รู้จัก Capybara

หลังรัน `rails generate rspec:install` (ตามที่ทำไปแล้วใน Part 046) ไฟล์
`spec/rails_helper.rb` ที่ได้มา **ยังไม่มี** การเชื่อม Capybara เข้ากับ RSpec ต้องเพิ่มเอง —
นี่คือจุดที่มือใหม่มักงงว่าทำไม `visit`/`fill_in` ใช้ไม่ได้ทั้งที่ generate `rspec:system`
ไฟล์ spec มาแล้ว:

```ruby
# spec/rails_helper.rb
require 'rspec/rails'
require 'capybara/rspec'   # <- ต้องเพิ่มบรรทัดนี้เอง ไม่ได้มาให้อัตโนมัติ
# ...

RSpec.configure do |config|
  # ...
  # config.infer_spec_type_from_file_location!   # ปล่อยไว้แบบ comment เหมือน Part 046
  config.include FactoryBot::Syntax::Methods      # ใช้ create(...)/build(...) แทน FactoryBot.create(...)
  # ...
end
```

Part 046 เลือกปล่อย `config.infer_spec_type_from_file_location!` ไว้เป็น comment
(ไม่เปิดใช้) แล้วเขียน `type: :model`/`type: :request` กำกับให้เห็นชัดเจนในทุก
`RSpec.describe` แทนการพึ่งพาตำแหน่งไฟล์ — Part นี้ทำตามธรรมเนียมเดียวกัน: ทุกตัวอย่างจะ
เขียน `type: :system` กำกับไว้เสมอ แม้ไฟล์จะอยู่ใต้ `spec/system/` อยู่แล้วก็ตาม เพื่อให้อ่าน
โค้ดคนเดียวเข้าใจได้ทันทีว่ากำลังทดสอบอะไรอยู่โดยไม่ต้องดูว่าไฟล์อยู่โฟลเดอร์ไหน

### รัน generator

```bash
bin/rails generate rspec:system Post
```

ผลลัพธ์จริง:

```
      create  spec/system/posts_spec.rb
```

เนื้อหาที่ generator สร้างให้ (สังเกตว่าตั้งชื่อไฟล์เป็นพหูพจน์ `posts_spec.rb` ตามชื่อ
model แบบเดียวกับ request spec ใน Part 046):

```ruby
require 'rails_helper'

RSpec.describe "Posts", type: :system do
  before do
    driven_by(:rack_test)
  end

  pending "add some scenarios (or delete) #{__FILE__}"
end
```

**สังเกต 2 จุดที่สำคัญ:**

1. `before { driven_by(:rack_test) }` — generator ตั้งค่า **driver** ของ Capybara ให้เป็น
   `:rack_test` เป็นค่าเริ่มต้นให้อัตโนมัติ (รายละเอียดเรื่อง driver ทั้งหมดอยู่ใน Step 473)
2. `pending "add some scenarios..."` — generator ไม่สร้าง scenario ให้เดา สร้างแค่โครง
   แล้วทิ้ง `pending` (ทำให้รัน `rspec` แล้วเห็นเป็นสีเหลือง "pending" ไม่ใช่สีเขียว/แดง)
   ไว้เตือนว่ายังไม่มี test จริง ต้องลบบรรทัดนี้แล้วเขียน `it` เอง

### กายวิภาคของ system spec หนึ่งตัว

โค้ดใน Step 474–478 ทั้งหมดจะอยู่ในรูปแบบเดียวกันนี้เสมอ:

```ruby
require 'rails_helper'

RSpec.describe "ชื่อ feature", type: :system do
  before { driven_by(:rack_test) }

  it "คำอธิบาย scenario" do
    visit some_path        # 1) ไปที่หน้าเริ่มต้น
    fill_in "...", with: "..."  # 2) จำลองการกระทำของผู้ใช้ทีละขั้น
    click_button "..."
    expect(page).to have_content("...")  # 3) ตรวจสอบผลลัพธ์บนหน้าจอ
  end
end
```

`page` คือ object ที่ Capybara เตรียมไว้ให้แทน "หน้าเว็บปัจจุบันในเบราว์เซอร์จำลอง" — ทุก
method อย่าง `visit`, `click_link`, `fill_in` จริงๆ แล้วเป็น shortcut ของ
`page.visit`/`page.click_link`/`page.fill_in` แต่ RSpec + Capybara mixin เข้ามาให้เรียกลอยๆ
ได้เลยในบล็อก `it` ของ system spec

---

## Step 473: `rack_test` (เร็ว ไม่รัน JS) เทียบกับ `selenium_chrome_headless` (เบราว์เซอร์จริง)

Capybara ไม่ได้ผูกติดกับเบราว์เซอร์ตัวใดตัวหนึ่ง แต่ทำงานผ่านสถาปัตยกรรมแบบ **pluggable
driver** — สลับ driver ได้โดยไม่ต้องแก้โค้ด `visit`/`fill_in`/`click_button` แม้แต่บรรทัด
เดียว (นี่คือจุดออกแบบที่ฉลาดมาก และเป็นเหตุผลที่ Capybara ครองตลาดเรื่องนี้มานานกว่า 15 ปี)

### `rack_test` — driver เริ่มต้น

```ruby
before { driven_by(:rack_test) }
```

`rack_test` **ไม่ได้เปิดเบราว์เซอร์ใดๆ เลย** — มันส่ง request เข้า Rack app ของเราโดยตรงใน
process เดียวกับที่ test กำลังรันอยู่ (คล้ายกับที่ request spec ทำ) แล้วเอา HTML ที่ได้กลับมา
มา parse ด้วย Nokogiri เพื่อให้ `fill_in`/`click_button` ค้นหา element ได้

**ข้อจำกัดสำคัญของ `rack_test`:** ไม่รัน JavaScript เลย เพราะไม่มี JS engine ใดๆ อยู่เบื้อง
หลัง — ถ้าหน้าเว็บต้องพึ่ง JS ในการแสดงผล (เช่น Turbo Frame ที่โหลดเนื้อหาผ่าน `fetch`,
Stimulus controller ที่ toggle class ตอนคลิก) `rack_test` จะ "เห็น" หน้าเว็บ ณ สถานะก่อน JS
ทำงานเท่านั้น ทดสอบพิสูจน์ได้ง่ายๆ ด้วยการลอง `save_screenshot` (method สำหรับถ่ายภาพหน้าจอ)
ภายใต้ driver นี้:

```ruby
it "ถ่ายภาพหน้าจอด้วย rack_test" do
  visit posts_path
  page.save_screenshot("tmp/capybara/demo.png")
end
```

ผลลัพธ์จริงเมื่อรัน:

```
Capybara::NotSupportedByDriverError:
  Capybara::Driver::Base#save_screenshot
```

Error นี้พิสูจน์ตรงตัวว่า `rack_test` ไม่ใช่ "เบราว์เซอร์" จริง — มันไม่มีภาพหน้าจอให้ถ่าย
เพราะไม่เคยเรนเดอร์ pixel ใดๆ เลยตั้งแต่ต้น (เทียบกับ `save_and_open_page` ใน Step 479 ที่
ใช้ได้กับทุก driver เพราะแค่ dump HTML ดิบ ไม่ใช่ภาพ)

### `selenium_chrome_headless` — เบราว์เซอร์จริงแบบไม่มีหน้าต่าง (headless)

เมื่อไหร่ก็ตามที่ต้องทดสอบพฤติกรรมที่พึ่ง JavaScript จริง ต้องสลับไปใช้ driver ที่ควบคุม
เบราว์เซอร์จริงผ่าน [Selenium WebDriver](https://www.selenium.dev/) — Chrome/Chromium แบบ
**headless** (รันแบบไม่เปิดหน้าต่าง UI ให้เห็น เหมาะกับรันบน CI server ที่ไม่มีจอ) เป็นตัว
เลือกที่นิยมที่สุดในปัจจุบัน

ลงทะเบียน driver นี้ใน `spec/rails_helper.rb`:

```ruby
# spec/rails_helper.rb
Capybara.default_driver = :rack_test
Capybara.javascript_driver = :selenium_chrome_headless

Capybara.register_driver :selenium_chrome_headless do |app|
  options = Selenium::WebDriver::Chrome::Options.new
  options.add_argument("--headless=new")
  options.add_argument("--no-sandbox")
  options.add_argument("--disable-dev-shm-usage")
  options.add_argument("--disable-gpu")

  Capybara::Selenium::Driver.new(app, browser: :chrome, options: options)
end
```

แล้วสลับใช้ต่อ spec ด้วย `driven_by`:

```ruby
RSpec.describe "หน้าที่มี Stimulus controller", type: :system do
  before { driven_by(:selenium_chrome_headless) }

  it "โหลดหน้าผ่านเบราว์เซอร์จริงและรัน JS ได้" do
    visit posts_path
    expect(page).to have_content("รายการโพสต์ทั้งหมด")
  end
end
```

> **เกร็ดจากการทดสอบจริงที่ควรรู้ไว้ล่วงหน้า:** ตอนเตรียมตัวอย่างสำหรับ Part นี้ ผู้เขียนรัน
> spec ข้างบนบนเครื่องทดสอบจริงแล้วเจอ error:
> ```
> Selenium::WebDriver::Error::SessionNotCreatedError:
>   session not created: This version of ChromeDriver only supports Chrome version 147
>   Current browser version is 141.0.7390.37 with binary path ...
> ```
> นี่คือปัญหาที่พบบ่อยมากในทางปฏิบัติ: **เวอร์ชันของ `chromedriver` กับเวอร์ชันของ
> Chrome/Chromium ที่ติดตั้งอยู่บนเครื่องต้องตรงกัน (หรืออย่างน้อยรองรับกัน)** ตั้งแต่
> `selenium-webdriver` เวอร์ชัน 4.6 เป็นต้นมา ตัว gem มีกลไกชื่อ **Selenium Manager** ที่
> พยายามดาวน์โหลด `chromedriver` เวอร์ชันที่ตรงกับ Chrome บนเครื่องให้อัตโนมัติเวลารันครั้ง
> แรก (ต้องมีอินเทอร์เน็ตออกไปยัง `googlechromelabs.github.io`) ถ้าเครื่อง CI/sandbox ไม่มี
> อินเทอร์เน็ตออกนอกได้ หรือมี `chromedriver` เวอร์ชันเก่า/ใหม่เกินไปค้างอยู่ใน `PATH` ก่อน
> จะเจอ error แบบนี้ **วิธีแก้ในทางปฏิบัติ:** (1) ลบ `chromedriver` ที่ค้างอยู่ใน `PATH` ทิ้ง
> แล้วปล่อยให้ Selenium Manager จัดการเอง, (2) ใน CI ใช้ Docker image ที่ pin เวอร์ชัน Chrome
> กับ chromedriver ให้ตรงกันไว้แล้ว (เช่น image ทางการของ `selenium/standalone-chrome`) หรือ
> (3) ระบุ `options.binary` ให้ชี้ไปยัง Chrome/Chromium build ที่มากับ `chromedriver` เวอร์ชัน
> เดียวกันเป๊ะ — เป็นเรื่องที่ต้องเจอเมื่อไหร่ก็ได้เวลาตั้งค่า CI ใหม่ ไม่ใช่บั๊กของโค้ดเรา

### เปรียบเทียบสองตัวนี้แบบสรุป

| หัวข้อ | `rack_test` | `selenium_chrome_headless` |
|---|---|---|
| ความเร็ว | เร็วมาก (เทียบเท่า request spec) | ช้ากว่ามาก (ต้องบูตเบราว์เซอร์จริง) |
| รัน JavaScript | ❌ ไม่รันเลย | ✅ รันจริง เหมือนผู้ใช้เปิด Chrome |
| Render CSS จริง | ❌ | ✅ |
| `save_screenshot` | ❌ (`NotSupportedByDriverError`) | ✅ ได้ไฟล์ `.png` จริง |
| ต้องมีเบราว์เซอร์ติดตั้งบนเครื่อง | ไม่ต้อง | ต้องมี Chrome/Chromium + chromedriver ที่เข้ากันได้ |
| เหมาะกับ | ทดสอบ flow ทั่วไปที่ไม่พึ่ง JS (ฟอร์ม, CRUD, auth) | ทดสอบ Turbo Frame/Stream, Stimulus controller (**เฟส 7**), หรือ JS library ใดๆ ที่ผูกกับหน้าเว็บ |

**แนวทางปฏิบัติที่แนะนำ:** ใช้ `rack_test` เป็นค่าเริ่มต้นเสมอ (เร็วกว่ามาก และครอบคลุม
system test ส่วนใหญ่ในหลักสูตรนี้ได้พอดี เพราะยังไม่เข้าเฟส 7) แล้วสลับไปใช้
`selenium_chrome_headless` เฉพาะ spec ที่ทดสอบพฤติกรรมที่พึ่ง JavaScript จริงๆ เท่านั้น
(ทำเครื่องหมายด้วย metadata `js: true` แล้วตั้งให้ RSpec เลือก driver อัตโนมัติตาม metadata
นี้ก็ได้ — ธรรมเนียมนี้จะเห็นบ่อยขึ้นมากตอนเข้า **เฟส 7: Frontend / Hotwire / Stimulus**)

---

## Step 474: Capybara DSL พื้นฐาน — `visit`, `click_link`, `click_button`, `fill_in`

จาก Step นี้เป็นต้นไปจะใช้แอปตัวอย่างที่มี:

- `User` (มี `has_secure_password` ต่อยอดจาก Part 041) — signup/login ได้
- `Post belongs_to :user` (มี `title`, `body`, `category`, `post_type`, `published`)
- `Comment belongs_to :post`

### `visit` — เปิดหน้าเว็บ

```ruby
visit posts_path       # ใช้ route helper เหมือนที่เคยใช้ใน request spec (Part 046)
visit "/posts"          # หรือใช้ path ดิบก็ได้ แต่ไม่แนะนำ (เปราะบางถ้า route เปลี่ยน)
```

`visit` เทียบเท่ากับผู้ใช้พิมพ์ URL ในแถบที่อยู่แล้วกด Enter — เป็นจุดเริ่มต้นของแทบทุก
system spec

### `click_link` และ `click_button`

```ruby
click_link "เขียนโพสต์ใหม่"     # หา <a> ที่มีข้อความนี้ (หรือ id/title/alt ของรูปภาพข้างใน)
click_button "เข้าสู่ระบบ"      # หา <button>, <input type="submit">, <input type="button">
```

ทั้งสอง method นี้ **ไม่ต้องระบุ selector CSS เลย** — ค้นหาจาก **ข้อความที่ผู้ใช้เห็นจริง
บนหน้าจอ** เป็นหลัก (เรียกว่า **locator แบบ semantic**) ซึ่งเป็นปรัชญาหลักของ Capybara: เขียน
เทสให้ใกล้เคียงกับสิ่งที่ "คนจริง" มองเห็นและกดที่สุด ไม่ใช่ไปผูกกับ implementation detail
อย่าง CSS class ที่เปลี่ยนบ่อย

> **ข้อควรระวังที่เจอจริงระหว่างทดสอบ:** ค่าเริ่มต้นของ Capybara (`Capybara.exact = false`)
> ทำให้ locator แบบข้อความ **match แบบ substring ไม่ใช่ exact match** ระหว่างเตรียมตัวอย่าง
> ผู้เขียนลองเขียน `click_button "บันทึก"` ทั้งที่ปุ่มจริงในฟอร์มชื่อ **"บันทึกโพสต์"** —
> ปรากฏว่า **test ผ่านเฉยๆ** เพราะ `"บันทึก"` เป็น substring ของ `"บันทึกโพสต์"` พอดี ถ้าหน้า
> เว็บมีปุ่ม 2 ปุ่มที่ข้อความคาบเกี่ยวกัน เช่น `"บันทึก"` กับ `"บันทึกและเผยแพร่"`
> `click_button "บันทึก"` อาจกดปุ่มผิดตัวโดยไม่มี error ใดๆ เตือนเลย ถ้าต้องการบังคับ exact
> match ให้ส่ง `exact: true`:
> ```ruby
> click_button "บันทึก", exact: true
> ```
> หรือตั้ง `Capybara.exact = true` เป็นค่า default ทั้งโปรเจกต์ก็ได้ถ้าทีมต้องการความเข้มงวด
> แบบนี้เสมอ

### `fill_in` — กรอกข้อมูลลงฟอร์ม

```ruby
fill_in "Email", with: "author@example.com"
fill_in "Password", with: "secret123"
```

`fill_in(locator, with:)` หา `<input>`/`<textarea>` จาก **label ที่ผูกกับมัน** (ผ่าน
`for="..."` กับ `id="..."` ที่ตรงกัน — Rails form helper อย่าง `form.label`/`form.email_field`
ผูกให้อัตโนมัติอยู่แล้ว) หรือจาก `name`, `id`, `placeholder` ก็ได้ถ้าไม่มี label

### ตัวอย่างเต็ม: helper สำหรับ login ที่ใช้ซ้ำได้ทุก spec

```ruby
# spec/system/posts_spec.rb
require 'rails_helper'

RSpec.describe "Posts", type: :system do
  before { driven_by(:rack_test) }

  let(:user) { create(:user, email: "author@example.com", password: "secret123") }

  def login_as(user)
    visit login_path
    fill_in "Email", with: user.email
    fill_in "Password", with: "secret123"
    click_button "เข้าสู่ระบบ"
  end
end
```

`login_as` เป็นแค่ Ruby method ธรรมดาที่นิยามในตัว `describe` block — ไม่มีอะไรพิเศษ ใช้
หลักการเดียวกับ helper method ทั่วไปที่เรียนมาตั้งแต่เฟส 1–2 จุดประสงค์คือลดโค้ดซ้ำ (DRY)
เวลามี spec หลายตัวที่ต้อง login ก่อนเริ่ม scenario จริง — เขียนครั้งเดียว เรียกใช้ได้ทุก
`it` block ในไฟล์เดียวกัน

รันจริง:

```bash
bundle exec rspec spec/system/posts_spec.rb
```

```
.

Finished in 0.13 seconds (files took 0.7 seconds to load)
1 example, 0 failures
```

---

## Step 475: ฟอร์มที่ซับซ้อนขึ้น — `select`, `check`/`uncheck`, `choose`

ฟอร์มจริงในเว็บแอปแทบไม่มีแค่ text field เสมอไป — Post ในตัวอย่างนี้มี dropdown
(`category`), radio button (`post_type`), และ checkbox (`published`) ทั้ง 3 แบบนี้ Capybara
มี method เฉพาะให้ครบ

### View ที่ใช้ทดสอบ

```erb
<%# app/views/posts/new.html.erb %>
<%= form_with model: @post, url: posts_path do |form| %>
  <div>
    <%= form.label :category, "หมวดหมู่" %>
    <%= form.select :category, ["general", "tech", "life"] %>
  </div>

  <div>
    <%= form.label :post_type, "ประเภท" %>
    <%= form.radio_button :post_type, "article" %>
    <%= form.label :post_type_article, "บทความ" %>
    <%= form.radio_button :post_type, "note" %>
    <%= form.label :post_type_note, "โน้ตสั้นๆ" %>
  </div>

  <div>
    <%= form.check_box :published %>
    <%= form.label :published, "เผยแพร่ทันที" %>
  </div>

  <%= form.submit "บันทึกโพสต์" %>
<% end %>
```

### `select` — เลือกค่าจาก dropdown

```ruby
select "tech", from: "หมวดหมู่"
```

`select(value, from:)` หา `<select>` จาก label (`"หมวดหมู่"`) แล้วเลือก `<option>` ที่มี
ข้อความตรงกับ `value` ที่ระบุ

### `choose` — เลือก radio button

```ruby
choose "โน้ตสั้นๆ"
```

ใช้ **ข้อความของ label** ไม่ใช่ id หรือชื่อ attribute — จุดนี้เป็นอีกจุดที่เจอปัญหาจริง
ระหว่างเตรียมตัวอย่าง: ลองเขียน `choose "post_type_note"` ก่อน (คิดว่าคือ id ของ
`form.label :post_type_note, ...`) แล้วได้ error:

```
Capybara::ElementNotFound:
  Unable to find radio button "post_type_note" that is not disabled
```

สาเหตุคือ **`post_type_note` เป็นแค่ argument แรกที่ส่งให้ `form.label`** (ใช้กำหนดว่า
label ผูกกับ input ตัวไหนผ่าน `for` attribute) ไม่ใช่ข้อความที่แสดงบนหน้าจอ (ข้อความจริงคือ
`"โน้ตสั้นๆ"` ที่เป็น argument ตัวที่สอง) พอตรวจสอบ HTML ที่ Rails generate จริงด้วย
`ActionView::Base` จะเห็นชัดว่า `for` ของ label ชี้ไปที่ `id="post_post_type_note"` ซึ่งตรง
กับ `id` ของ radio button พอดี — Capybara หา element ที่ถูกจาก **ข้อความที่มองเห็น** เสมอ
(`"โน้ตสั้นๆ"`) ไม่ใช่จาก argument ภายในของ helper ที่ใช้ generate label:

```html
<input type="radio" value="note" name="post[post_type]" id="post_post_type_note" />
<label for="post_post_type_note">โน้ตสั้นๆ</label>
```

### `check` / `uncheck` — ติ๊ก/ยกเลิกติ๊ก checkbox

```ruby
check "เผยแพร่ทันที"     # ติ๊ก checkbox
uncheck "เผยแพร่ทันที"   # ยกเลิกการติ๊ก (ถ้าติ๊กอยู่ก่อน)
```

ทั้งสอง method นี้ **idempotent อย่างปลอดภัย**: `check` บน checkbox ที่ติ๊กอยู่แล้วจะไม่ทำ
อะไร (ไม่ toggle ให้กลายเป็นไม่ติ๊ก) เช่นเดียวกับ `uncheck` บน checkbox ที่ไม่ได้ติ๊กอยู่แล้ว
— ต่างจากการ `click` checkbox ตรงๆ ที่จะ toggle ทุกครั้งไม่ว่าสถานะเดิมเป็นอย่างไร

### รวมเข้าด้วยกัน: spec ที่ใช้ทั้ง 3 แบบ

```ruby
it "ผู้ใช้ที่ login แล้วเขียนโพสต์และเห็นโพสต์ในรายการ" do
  login_as(user)
  click_link "เขียนโพสต์ใหม่"

  fill_in "หัวข้อ", with: "ทดสอบ Capybara"
  fill_in "เนื้อหา", with: "เนื้อหาของโพสต์ทดสอบ"
  select "tech", from: "หมวดหมู่"
  choose "โน้ตสั้นๆ"
  check "เผยแพร่ทันที"

  click_button "บันทึกโพสต์"

  expect(page).to have_content("สร้างโพสต์สำเร็จ")
  expect(page).to have_content("ทดสอบ Capybara")
end
```

รันจริง — ผ่านทั้งหมด:

```
.

Finished in 0.1 seconds (files took 0.66 seconds to load)
1 example, 0 failures
```

และตรวจสอบเพิ่มเติมด้วย `uncheck` ในอีก scenario หนึ่ง (สร้างโพสต์แบบไม่เผยแพร่):

```ruby
check "เผยแพร่ทันที"
uncheck "เผยแพร่ทันที"
click_button "บันทึกโพสต์"

post = Post.find_by(title: "ฉบับร่าง")
expect(post.published).to eq(false)
```

```
.

Finished in 0.1 seconds (files took 0.67 seconds to load)
1 example, 0 failures
```

---

## Step 476: Assertion ของ Capybara — `have_content`, `have_selector`, `have_css`

RSpec matcher ปกติอย่าง `eq`, `include` ใช้ตรวจสอบ **ค่า Ruby object** — แต่ system spec
ต้องตรวจสอบ **สิ่งที่ปรากฏบนหน้าเว็บ** ซึ่งเป็น HTML structure ไม่ใช่ string ธรรมดา Capybara
จึงมี matcher ชุดของตัวเองที่ใช้คู่กับ `expect(page).to`

### `have_content` — ตรวจสอบว่ามีข้อความนี้ปรากฏอยู่ที่ไหนก็ได้บนหน้า

```ruby
expect(page).to have_content("สร้างโพสต์สำเร็จ")
```

`have_content` มองแค่ **ข้อความที่มองเห็นได้จริง** (visible text) ไม่สนใจว่าอยู่ใน tag ไหน
โครงสร้าง HTML เป็นอย่างไร — เหมาะกับตรวจสอบข้อความทั่วไปอย่าง flash message

### `have_selector` — ตรวจสอบว่ามี element ที่ตรงกับ CSS/XPath selector

```ruby
expect(page).to have_selector("li", text: "ทดสอบ Capybara")
```

ต่างจาก `have_content` ตรงที่ **ระบุ tag/selector ได้ชัดเจน** — ตัวอย่างข้างบนตรวจว่ามี
`<li>` ที่มีข้อความ `"ทดสอบ Capybara"` อยู่ข้างใน ไม่ใช่แค่ข้อความนั้นปรากฏที่ไหนก็ได้บนหน้า
(เช่นถ้ามันดันไปโผล่ใน `<h1>` แทน `have_selector("li", ...)` จะ fail แต่ `have_content` จะ
ผ่าน) — ใช้เมื่อโครงสร้าง HTML มีผลต่อความถูกต้องของฟีเจอร์ เช่น "โพสต์ต้องอยู่ใน `<li>` ของ
รายการ ไม่ใช่แค่มีข้อความอยู่ตรงไหนก็ได้บนหน้า"

### `have_css` — เหมือน `have_selector` แต่ระบุว่าเป็น CSS selector เสมอ

```ruby
expect(page).to have_css("h1", text: "ฉบับร่าง")
expect(page).to have_css(".comments")
```

`have_css(sel)` เทียบเท่า `have_selector(:css, sel)` — ใช้เมื่อมั่นใจว่าจะเขียนด้วย CSS
selector เท่านั้น (ไม่สลับไป XPath) ทำให้โค้ดสั้นและอ่านง่ายกว่าเล็กน้อย

ทดสอบจริงทั้งสองแบบพร้อมกัน:

```ruby
visit post_path(post)
expect(page).to have_css("h1", text: "ฉบับร่าง")
expect(page).to have_css(".comments")
```

```
.

Finished in 0.1 seconds (files took 0.67 seconds to load)
1 example, 0 failures
```

### matcher อื่นๆ ที่ควรรู้จักไว้ (ใช้หลักการเดียวกัน)

| Matcher | ใช้ตรวจสอบ |
|---|---|
| `have_link("ข้อความ")` | มี `<a>` ที่ข้อความ/href ตรงกัน |
| `have_button("ข้อความ")` | มีปุ่มที่กดได้ |
| `have_field("ชื่อ field")` | มี input/textarea/select อยู่ (เช็คแค่ว่ามี ไม่เช็คค่า) |
| `have_checked_field("ชื่อ")` | checkbox/radio ที่ติ๊กอยู่ ตรงกับชื่อที่ระบุ |
| `have_no_content("...")` | **ไม่มี** ข้อความนี้ (ดูรายละเอียดสำคัญเรื่อง negation ใน Step 477) |

**กฎการเลือกใช้ในทางปฏิบัติ:** ใช้ `have_content` เป็นค่าเริ่มต้นเสมอเพราะอ่านง่ายและพอเพียง
สำหรับ 80% ของสถานการณ์ สลับไปใช้ `have_selector`/`have_css` เฉพาะตอนที่ **โครงสร้าง HTML
เป็นส่วนหนึ่งของสิ่งที่กำลังทดสอบจริงๆ** (เช่น ต้องอยู่ใน element ที่ถูกต้อง ไม่ใช่แค่มี
ข้อความอยู่ที่ไหนสักแห่ง)

---

## Step 477: กลไก auto-waiting ของ Capybara กับ anti-pattern การใช้ `sleep`

### ปัญหาของหน้าเว็บที่อัปเดตแบบ asynchronous

พิจารณาหน้าเว็บที่ใช้ Turbo Stream หรือ JavaScript ยิง `fetch` ไปเซิร์ฟเวอร์แล้วค่อยเติม
เนื้อหาเข้า DOM (เฟส 7 จะเจอสถานการณ์นี้เต็มๆ) — ถ้าเขียน assertion ทันทีหลังคลิกปุ่ม โดยที่
เนื้อหายังโหลดไม่เสร็จ test จะ fail ทั้งที่โค้ดจริงถูกต้อง (แค่ "ช้าไปนิดเดียว")

### Anti-pattern: การใช้ `sleep`

```ruby
# อย่าทำแบบนี้
click_button "โหลดข้อมูล"
sleep 2   # ❌ รอแบบเดา — อาจสั้นไป (test flaky) หรือยาวเกิน (test ช้าโดยไม่จำเป็น)
expect(page).to have_content("โหลดเสร็จแล้ว")
```

ปัญหาของ `sleep(n)`:

1. **สั้นเกินไป** → บนเครื่อง CI ที่ทรัพยากรจำกัดกว่าเครื่อง dev อาจโหลดไม่ทันใน 2 วินาที →
   test fail แบบสุ่ม (**flaky test** — เทสที่บางครั้งผ่าน บางครั้ง fail โดยไม่มีอะไรเปลี่ยน
   ในโค้ด เป็นสิ่งที่ทีมพัฒนาเกลียดที่สุดในบรรดาปัญหาเรื่อง testing)
2. **ยาวเกินไป** → ถ้าตั้ง `sleep 5` ให้ชัวร์ไว้ก่อน ทุก test ที่ผ่าน `sleep` นี้จะช้าลง 5
   วินาทีเสมอ **แม้ในกรณีที่เนื้อหาโหลดเสร็จภายใน 0.1 วินาทีจริงๆ** พอมี test แบบนี้หลายสิบตัว
   เวลารัน test suite ทั้งหมดจะบวมขึ้นมหาศาลโดยไม่จำเป็น

### วิธีที่ถูกต้อง: ใช้ `have_content`/`have_selector` แล้วปล่อยให้ Capybara "รอ" ให้เอง

```ruby
click_button "โหลดข้อมูล"
expect(page).to have_content("โหลดเสร็จแล้ว")   # ✅ Capybara รอให้อัตโนมัติ ไม่ fail ทันที
```

**กลไกเบื้องหลัง:** matcher ทุกตัวที่ขึ้นต้นด้วย `have_` ใน Capybara ไม่ได้ตรวจสอบครั้งเดียว
แล้วจบ แต่จะ **poll (ตรวจซ้ำเป็นระยะ) จนกว่าเงื่อนไขเป็นจริง หรือจนกว่าจะครบเวลาที่กำหนดใน
`Capybara.default_max_wait_time`** (ค่า default คือ **2 วินาที**) ถ้าเนื้อหาปรากฏภายใน 2
วินาทีนั้น — ไม่ว่าจะใช้เวลา 0.05 วินาทีหรือ 1.9 วินาทีก็ตาม — assertion จะผ่านทันทีที่เจอ
โดยไม่ต้องรอจนครบ 2 วินาที (ต่างจาก `sleep` ที่รอครบเวลาเสมอไม่ว่าเนื้อหาจะพร้อมเร็วแค่ไหน)

ปรับเวลารอได้ถ้าจำเป็น (เช่น หน้าที่รู้ว่าช้ากว่าปกติ):

```ruby
Capybara.default_max_wait_time = 5   # ตั้งค่า global ใน rails_helper.rb

# หรือรอเฉพาะจุดเดียว
expect(page).to have_content("โหลดเสร็จแล้ว", wait: 10)
```

> **ทำไมตัวอย่างที่ใช้ `rack_test` ใน Part นี้ถึงไม่เคยต้องรอจริง:** `rack_test` เป็น driver
> แบบ synchronous ล้วนๆ (ไม่มี JS, ไม่มี network request แบบ async) — พอ `visit`/`click_button`
> คืนค่ากลับมา หมายความว่าหน้าเว็บ "เสร็จสมบูรณ์" แล้วเสมอ กลไก auto-wait จึงไม่มีผลกระทบที่
> สังเกตเห็นได้ (เจอเนื้อหาตั้งแต่รอบแรกทุกครั้ง) **แต่โค้ด `expect(page).to have_content`
> เขียนเหมือนกันเป๊ะไม่ว่าจะสลับไปใช้ `selenium_chrome_headless` ที่มี JS แบบ async จริง** —
> นี่คือเหตุผลที่ควรเขียน assertion ด้วย `have_content` เป็นนิสัยตั้งแต่แรก แม้จะทดสอบด้วย
> `rack_test` อยู่ก็ตาม เพราะโค้ดเดียวกันจะยังใช้งานได้ถูกต้องทันทีที่ต้องสลับไปทดสอบหน้าที่มี
> JavaScript จริงในอนาคต (เฟส 7) โดยไม่ต้องแก้อะไรเลย

### สรุปกฎ

**ห้ามใช้ `sleep` ใน system test เพื่อรอ UI อัปเดตเด็ดขาด** — ใช้ `have_content`,
`have_selector`, `have_css`, `have_no_content` หรือ matcher อื่นที่ขึ้นต้นด้วย `have_`
เท่านั้น เพราะมันมาพร้อม auto-wait ในตัวอยู่แล้วเสมอ ครอบคลุมทั้งกรณี sync (`rack_test`) และ
async (`selenium_*`) โดยไม่ต้องเขียนโค้ดต่างกัน

---

## Step 478: จำกัดขอบเขตการค้นหาด้วย `within`

### ปัญหา: ข้อความ/element เดียวกันปรากฏซ้ำหลายจุดบนหน้าเดียวกัน

หน้า `posts/show` ในตัวอย่างนี้มีทั้ง **เนื้อหาของโพสต์** และ **กล่องแสดงความคิดเห็น** อยู่ใน
หน้าเดียวกัน ถ้าทั้งสองส่วนมีฟอร์ม/ปุ่ม/ข้อความคล้ายกัน (เช่น ปุ่ม "ส่ง" ปรากฏทั้งในฟอร์มแก้ไข
โพสต์และฟอร์มคอมเมนต์) การเขียน `click_button "ส่ง"` ตรงๆ จะกำกวมว่าจะกดปุ่มไหน

`within(selector) { ... }` แก้ปัญหานี้โดย **จำกัดขอบเขตการค้นหาของทุก Capybara method ใน
block ให้อยู่แค่ภายใน element ที่ตรงกับ selector เท่านั้น**

### View ที่ใช้ทดสอบ

```erb
<%# app/views/posts/show.html.erb %>
<div class="comments">
  <h2>ความคิดเห็น (<%= @post.comments.count %>)</h2>
  <ul>
    <% @post.comments.each do |comment| %>
      <li><%= comment.body %></li>
    <% end %>
  </ul>

  <%= form_with model: [@post, Comment.new], url: post_comments_path(@post) do |form| %>
    <%= form.text_area :body, placeholder: "แสดงความคิดเห็น..." %>
    <%= form.submit "ส่งความคิดเห็น" %>
  <% end %>
</div>
```

### ใช้ `within` เพื่อกรอกและตรวจสอบเฉพาะในกล่องคอมเมนต์

```ruby
it "แสดงความคิดเห็นเฉพาะในกล่อง comments เท่านั้น" do
  post = create(:post, title: "โพสต์หลัก", user: user)
  create(:comment, post: post, body: "ความเห็นเก่า")

  login_as(user)
  visit post_path(post)

  within(".comments") do
    expect(page).to have_content("ความเห็นเก่า")
    fill_in "comment_body", with: "ความเห็นใหม่จาก within"
    click_button "ส่งความคิดเห็น"
  end

  expect(page).to have_content("ความเห็นใหม่จาก within")
end
```

รันจริง:

```
.

Finished in 0.14 seconds (files took 0.7 seconds to load)
1 example, 0 failures
```

**จุดสำคัญ:** `expect(page).to have_content(...)` และ `fill_in`/`click_button` ที่อยู่
**ภายใน** block ของ `within(".comments")` จะค้นหาแค่ภายใน `<div class="comments">` เท่านั้น
— ถ้าข้อความ `"ความเห็นเก่า"` ดันไปปรากฏซ้ำที่อื่นบนหน้า (เช่น ใน sidebar แสดง "คอมเมนต์
ล่าสุด") test ตัวนี้จะไม่สนใจส่วนนั้นเลย ส่วน `expect(page).to have_content(...)` บรรทัด
สุดท้ายที่อยู่ **นอก** block `within` กลับไปค้นหาทั้งหน้าตามปกติ — แสดงให้เห็นว่า scope ของ
`within` จำกัดอยู่แค่ใน block เท่านั้น ไม่ส่งผลกับโค้ดหลัง `end`

### ตัวช่วยที่คล้ายกัน

Capybara ยังมี `within_fieldset(legend)` และ `within_table(caption)` สำหรับกรณีเฉพาะเจาะจง
มากขึ้น (ฟอร์มที่แบ่งเป็น `<fieldset>` หรือข้อมูลในตาราง) หลักการเดียวกับ `within` ทั้งหมด —
จำกัดขอบเขตการค้นหาก่อนแล้วค่อย assert/interact ข้างใน

---

## Step 479: Debug system test ที่ fail — `save_and_open_page`, `save_screenshot`, `binding.pry`

System test เป็นเทสประเภทที่ **อ่าน error message อย่างเดียวแล้วเดาสาเหตุยากที่สุด** เพราะ
ข้อความ error บอกแค่ "หา element ไม่เจอ" แต่ไม่บอกว่า **หน้าเว็บ ณ ตอนนั้นหน้าตาเป็นอย่างไร
จริงๆ** Capybara จึงมีเครื่องมือช่วย debug 3 ตัวที่ควรรู้จักไว้เสมอ

### สถานการณ์จำลอง: พิมพ์ชื่อปุ่มผิดโดยไม่ตั้งใจ

```ruby
it "เขียนโพสต์ใหม่ (มี typo ตั้งใจใส่ไว้เพื่อสาธิตการ debug)" do
  visit login_path
  fill_in "Email", with: user.email
  fill_in "Password", with: "secret123"
  click_button "เข้าสู่ระบบ"

  click_link "เขียนโพสต์ใหม่"
  fill_in "หัวข้อ", with: "ทดสอบ debug"
  fill_in "เนื้อหา", with: "เนื้อหา"
  click_button "Save Post"   # <- typo: ปุ่มจริงชื่อ "บันทึกโพสต์"

  expect(page).to have_content("สร้างโพสต์สำเร็จ")
end
```

รันจริงแล้วได้ error:

```
Capybara::ElementNotFound:
  Unable to find button "Save Post" that is not disabled
```

Error บอกแค่ว่า "หาปุ่มชื่อนี้ไม่เจอ" แต่ **ไม่บอกว่าปุ่มที่มีอยู่จริงบนหน้านั้นชื่ออะไร** —
ถ้าไม่มีเครื่องมือ debug ก็ต้องเดาไปเรื่อยๆ หรือเปิดเบราว์เซอร์จริงมาคลิกเองทีละขั้นเพื่อ
เทียบ

### เครื่องมือที่ 1: `save_and_open_page` — dump HTML ปัจจุบันเก็บไว้ดู

```ruby
click_link "เขียนโพสต์ใหม่"
fill_in "หัวข้อ", with: "ทดสอบ debug"
fill_in "เนื้อหา", with: "เนื้อหา"

save_and_open_page   # <- เพิ่มบรรทัดนี้ก่อนบรรทัดที่ fail

click_button "Save Post"
```

รันจริง:

```
File saved to /path/to/app/tmp/capybara/capybara-20260926065206592690395.html.
Please install the launchy gem to open the file automatically.
```

`save_and_open_page` ใช้ได้กับ **ทุก driver รวมถึง `rack_test`** เพราะมันแค่บันทึก HTML
สถานะปัจจุบันของ `page` ลงไฟล์ `.html` ในโฟลเดอร์ `tmp/capybara/` ไม่ต้องพึ่งความสามารถของ
เบราว์เซอร์แต่อย่างใด (ถ้าติดตั้ง gem [`launchy`](https://github.com/copiousfreetime/launchy)
เพิ่ม มันจะเปิดไฟล์ในเบราว์เซอร์ให้อัตโนมัติทันที ถ้าไม่มีก็ต้องเปิดไฟล์เองตาม path ที่พิมพ์
มาให้)

เปิดไฟล์ที่ dump ออกมาดูจริง จะเจอปุ่มตัวจริงอยู่ในนั้น:

```html
<input type="submit" name="commit" value="บันทึกโพสต์" data-disable-with="บันทึกโพสต์" />
```

เห็นทันทีว่าปุ่มจริงชื่อ `"บันทึกโพสต์"` ไม่ใช่ `"Save Post"` — แก้โค้ด spec ให้ตรง:

```ruby
click_button "บันทึกโพสต์"   # แก้ไขแล้ว
```

รันซ้ำ:

```
.

Finished in 0.13 seconds (files took 0.63 seconds to load)
1 example, 0 failures
```

### เครื่องมือที่ 2: `page.save_screenshot` — ถ่ายภาพหน้าจอจริง (ต้องใช้ browser driver)

```ruby
page.save_screenshot("tmp/capybara/debug.png")
```

ต่างจาก `save_and_open_page` ตรงที่ `save_screenshot` **ต้องใช้ driver ที่เป็นเบราว์เซอร์
จริงเท่านั้น** (เช่น `selenium_chrome_headless`) เพราะต้องมีการ "เรนเดอร์" ภาพจริงถึงจะถ่าย
ได้ — อย่างที่พิสูจน์ไปแล้วใน Step 473 ว่า `rack_test` เรียก method นี้แล้วได้
`Capybara::NotSupportedByDriverError` ทันที เพราะมันไม่เคยเรนเดอร์ภาพอะไรเลยตั้งแต่ต้น
เหมาะกับกรณีที่ต้องดูว่า CSS/JS เรนเดอร์ผลลัพธ์หน้าตาถูกต้องหรือไม่ (ไม่ใช่แค่โครงสร้าง HTML
ดิบแบบที่ `save_and_open_page` ให้)

**เกร็ดที่มีประโยชน์มาก:** หลายทีมตั้งค่า RSpec ให้ **ถ่ายภาพหน้าจออัตโนมัติทุกครั้งที่
system test fail** (ไม่ต้องเขียน `save_screenshot` เองในทุก `it`) ด้วย hook แบบนี้ใน
`rails_helper.rb`:

```ruby
RSpec.configure do |config|
  config.after(:each, type: :system) do |example|
    if example.exception && Capybara.current_driver != :rack_test
      page.save_screenshot("tmp/capybara/failure_#{example.description.parameterize}.png")
    end
  end
end
```

### เครื่องมือที่ 3: `binding.pry` — หยุดกลางคันแล้วสำรวจแบบ interactive

ทบทวนจาก Part 001 (Step 5) — `binding.pry` แทรกลงตรงไหนก็ได้ในโค้ด Ruby เพื่อหยุดโปรแกรม ณ
จุดนั้นแล้วเข้าสู่ session แบบ interactive ใน system spec ก็ใช้หลักการเดียวกันทุกประการ:

```ruby
click_link "เขียนโพสต์ใหม่"
fill_in "หัวข้อ", with: "ทดสอบ debug"

binding.pry   # <- โปรแกรมจะหยุดตรงนี้ รอคำสั่งจาก terminal

click_button "บันทึกโพสต์"
```

พอรันด้วย `bundle exec rspec` แล้วเจอ `binding.pry` โปรแกรมจะหยุดรอ แล้วเปิด prompt ให้พิมพ์
โค้ด Ruby ทดลองได้ทันทีในบริบทของ test ที่กำลังรันอยู่ — สิ่งที่มีประโยชน์ที่สุดตอน debug
system test คือลองเรียก method ของ Capybara ตรงๆ ผ่าน `page`:

```irb
[1] pry(#<RSpec::ExampleGroups::Posts>)> page.text
=> "เขียนโพสต์ใหม่\nหัวข้อ\nเนื้อหา\nหมวดหมู่ ..."
[2] pry(#<RSpec::ExampleGroups::Posts>)> page.has_button?("บันทึกโพสต์")
=> true
[3] pry(#<RSpec::ExampleGroups::Posts>)> page.has_button?("Save Post")
=> false
[4] pry(#<RSpec::ExampleGroups::Posts>)> current_user
=> nil   # (ถ้าเรียกใน controller context — ใน system spec ต้องเรียกผ่าน object อื่น)
```

`page.text` คือวิธีเร็วที่สุดในการดูข้อความทั้งหมดที่ปรากฏบนหน้า ณ ตอนนั้นแบบไม่ต้องเปิดไฟล์
ใดๆ ส่วน `page.has_button?("...")`/`page.has_content?("...")` คือรูปแบบ **boolean** ของ
matcher `have_button`/`have_content` (ใช้ `?` แทน `expect().to`) — เหมาะมากเวลาอยากลองเดา
ข้อความที่ถูกต้องแบบ interactive ก่อนแก้โค้ดจริง

### สรุปว่าใช้เครื่องมือไหนตอนไหน

| เครื่องมือ | ใช้เมื่อ | ต้องมี browser driver ไหม |
|---|---|---|
| `save_and_open_page` | อยากดูโครงสร้าง HTML/ข้อความจริงบนหน้า ณ จุดที่ fail | ไม่ต้อง (ใช้ได้กับ `rack_test`) |
| `page.save_screenshot` | อยากดูว่าหน้าตา **ที่เรนเดอร์จริง** (CSS/JS ทำงานแล้ว) เป็นอย่างไร | ต้องมี (เช่น `selenium_chrome_headless`) |
| `binding.pry` | อยากสำรวจแบบ interactive ทีละคำสั่ง ไม่ใช่แค่ดู snapshot นิ่งๆ ครั้งเดียว | ไม่ต้อง |

---

## Step 480: แบบฝึกหัด — User Journey เต็มรูปแบบ และควรเขียน System Test กี่ตัวถึงจะพอดี

### ก่อนลงมือ: system test ควรมีกี่ตัว

คำถามที่ทีมพัฒนาที่เพิ่งเริ่มเขียน system test มักถามคือ "ควรเขียน system test ครอบคลุม
ทุก scenario เหมือน model spec ไหม" — คำตอบคือ **ไม่ควรเด็ดขาด** และนี่คือเหตุผล

### Ice Cream Cone Anti-pattern

ทบทวนรูปพีระมิดจาก Step 471: **model spec เยอะที่สุดที่ฐาน, request spec ปานกลาง, system
spec น้อยที่สุดที่ยอด** ทีมที่ทำผิดพลาดบ่อยคือ **กลับหัวพีระมิด** — เขียน system test
ครอบคลุมทุก edge case, ทุก validation message, ทุกเงื่อนไข if/else เหมือนที่ควรทำกับ model
spec แทน สุดท้ายได้โครงสร้างที่เรียกว่า **"Ice Cream Cone Anti-pattern"** (พีระมิดหัวกลับ
ฐานแคบยอดกว้าง เหมือนไอศกรีมโคน): system test เพียบ, request/model spec น้อย

ผลเสียของ ice cream cone:

1. **CI ช้ามาก** — system test แต่ละตัวใช้เวลาเป็นวินาที ถ้ามีหลายร้อยตัว suite ทั้งหมดอาจ
   ใช้เวลาเป็นชั่วโมง เทียบกับ model spec หลักพันตัวที่รันจบใน 1–2 นาที
2. **Flaky บ่อยกว่ามาก** — ยิ่งเทสผ่านหลายชั้น (routing, view, session, DOM) ยิ่งมีจุดที่
   ไม่เสถียรได้มากขึ้น (timing, การ render ที่ไม่ deterministic)
3. **debug ยาก** — system test ที่ fail บอกแค่ "หน้าเว็บไม่เป็นไปตามที่คาด" แต่สาเหตุอาจอยู่
   ที่ model validation, controller logic, view template, หรือ routing ก็ได้ทั้งนั้น ต้อง
   ไล่ดูทีละชั้นเอง (ต่างจาก model spec ที่ fail แล้วรู้ทันทีว่าปัญหาอยู่ตรงไหน)
4. **แก้ไข UI เล็กน้อยแล้วเทสพังเป็นแถบ** — ถ้าเปลี่ยนข้อความปุ่มจาก "บันทึก" เป็น "บันทึกข้อมูล"
   system test ทุกตัวที่อ้างอิงข้อความนั้นพังหมด ทั้งที่ business logic ไม่ได้เปลี่ยนเลย

### กฎปฏิบัติที่แนะนำ

**เขียน system test แค่สำหรับ "happy path" ของแต่ละ user journey ที่สำคัญต่อธุรกิจเท่านั้น**
— หนึ่ง flow หลักต่อหนึ่ง system test ก็เพียงพอ ตัวอย่างเช่น:

- Signup → Login → สร้างโพสต์ → เห็นในรายการ (สิ่งที่กำลังจะเขียนในแบบฝึกหัดนี้)
- ค้นหาสินค้า → เพิ่มลงตะกร้า → checkout (ตัวอย่างในเฟสหลังๆ)

ส่วน **edge case ทั้งหมด** (validation ผิดพลาด, unauthorized access, ข้อความ error แต่ละ
แบบ, การคำนวณตัวเลขที่ซับซ้อน) ให้ผลักลงไปทดสอบที่ **model spec** (เร็วที่สุด) หรือ
**request spec** (ถ้าต้องเช็ค HTTP status/routing ร่วมด้วย) แทน — สองชั้นนี้ตอบคำถามเดียวกัน
ได้เร็วกว่าและเสถียรกว่ามาก ไม่จำเป็นต้องพิสูจน์ผ่านการ "คลิกจริงในเบราว์เซอร์" ทุกกรณี

พูดให้จำง่าย: **system test พิสูจน์ว่า "ชิ้นส่วนทั้งหมดประกอบเข้าด้วยกันแล้วทำงานได้จริง"
ไม่ใช่พิสูจน์ว่า "แต่ละชิ้นส่วนถูกต้องในทุกกรณี"** — หน้าที่หลังเป็นของ model/request spec

### โจทย์

เขียน system spec ชื่อ `spec/system/user_journey_spec.rb` ที่จำลอง user คนหนึ่งทำครบวงจร:

1. สมัครสมาชิกใหม่ (signup)
2. Logout แล้ว login ใหม่ด้วยบัญชีเดียวกัน (พิสูจน์ว่า credential ที่บันทึกไว้ใช้ login
   ได้จริง ไม่ใช่แค่ auto-login ตอน signup)
3. เขียนโพสต์ใหม่ 1 โพสต์ (กรอกฟอร์มที่มีทั้ง text field และ `select`)
4. ตรวจสอบว่าโพสต์ปรากฏในหน้ารายการ พร้อมชื่อผู้เขียนที่ถูกต้อง

ใช้ driver `rack_test` (เร็วที่สุด และ flow นี้ไม่มี JavaScript เกี่ยวข้อง)

### เฉลย

```ruby
# spec/system/user_journey_spec.rb
require 'rails_helper'

RSpec.describe "User journey: สมัครสมาชิก -> login -> สร้างโพสต์ -> เห็นในรายการ", type: :system do
  before { driven_by(:rack_test) }

  it "ผู้ใช้ใหม่ทำครบวงจรได้สำเร็จ" do
    # 1) สมัครสมาชิก
    visit new_registration_path
    fill_in "Email", with: "newbie@example.com"
    fill_in "Password", with: "secret123"
    fill_in "ยืนยันรหัสผ่าน", with: "secret123"
    click_button "สมัครสมาชิก"

    expect(page).to have_content("สมัครสมาชิกสำเร็จ")

    # 2) ระบบ login ให้อัตโนมัติหลังสมัคร -> logout ก่อน เพื่อทดสอบ login แยกต่างหาก
    click_button "ออกจากระบบ"
    expect(page).to have_content("ออกจากระบบเรียบร้อยแล้ว")

    # 3) login ด้วยบัญชีที่เพิ่งสมัคร
    visit login_path
    fill_in "Email", with: "newbie@example.com"
    fill_in "Password", with: "secret123"
    click_button "เข้าสู่ระบบ"

    expect(page).to have_content("เข้าสู่ระบบสำเร็จ")

    # 4) สร้างโพสต์ใหม่
    click_link "เขียนโพสต์ใหม่"
    fill_in "หัวข้อ", with: "โพสต์แรกของฉัน"
    fill_in "เนื้อหา", with: "สวัสดีชาว Capybara"
    select "life", from: "หมวดหมู่"
    click_button "บันทึกโพสต์"

    # 5) ยืนยันว่าเห็นโพสต์ในรายการจริง
    expect(page).to have_content("สร้างโพสต์สำเร็จ")
    expect(page).to have_selector("li", text: "โพสต์แรกของฉัน")
    within("#posts") do
      expect(page).to have_content("newbie@example.com")
    end
  end
end
```

รันจริงด้วย `bundle exec rspec spec/system/user_journey_spec.rb --format documentation`:

```
User journey: สมัครสมาชิก -> login -> สร้างโพสต์ -> เห็นในรายการ
  ผู้ใช้ใหม่ทำครบวงจรได้สำเร็จ

Finished in 0.14 seconds (files took 0.7 seconds to load)
1 example, 0 failures
```

รวมกับ system spec อื่นๆ ที่เขียนไว้ก่อนหน้าใน Part นี้ (Step 474–478) แล้วรันทั้งโฟลเดอร์
พร้อมกัน:

```bash
bundle exec rspec spec/system --format documentation
```

```
Posts
  การเขียนโพสต์ใหม่
    ผู้ใช้ที่ login แล้วเขียนโพสต์และเห็นโพสต์ในรายการ
  การแสดงความคิดเห็น scoped ด้วย within
    แสดงความคิดเห็นเฉพาะในกล่อง comments เท่านั้น
  แบบฟอร์มที่กรอกไม่ครบ
    แสดง error และไม่สร้างโพสต์

User journey: สมัครสมาชิก -> login -> สร้างโพสต์ -> เห็นในรายการ
  ผู้ใช้ใหม่ทำครบวงจรได้สำเร็จ

Finished in 0.35 seconds (files took 0.73 seconds to load)
4 examples, 0 failures
```

**สังเกตความเร็ว:** ทั้ง 4 system spec ที่จำลอง user journey เต็มรูปแบบ (หลาย `visit`,
`fill_in`, `click_button` ต่อกันหลายขั้นตอน) รวมกันใช้เวลาแค่ **0.35 วินาที** เพราะใช้
`rack_test` ล้วน — นี่คือเหตุผลที่ Step 473 แนะนำให้ใช้ `rack_test` เป็นค่าเริ่มต้นเสมอ
ยกเว้นจำเป็นต้องพึ่ง JavaScript จริงๆ เท่านั้นถึงจะสลับไป `selenium_chrome_headless`
(ซึ่งจะช้ากว่านี้หลายเท่าตัวต่อ 1 spec)

**สิ่งที่เฉลยนี้ตั้งใจสอนเพิ่มเติมนอกเหนือจาก DSL พื้นฐาน:**

- การเขียน system test ที่ครอบคลุม **หลาย request ต่อเนื่องกันของ user คนเดียว**
  (signup → logout → login → create → verify) ในเทสเดียว ซึ่งเป็นสิ่งที่ request spec ทำได้
  ยากกว่ามาก (ต้องจัดการ session/cookie เองระหว่าง request หลายครั้ง) แต่ system spec ทำให้
  เป็นเรื่องธรรมชาติเพราะ Capybara จัดการ session ให้อัตโนมัติเหมือนเบราว์เซอร์จริง
- การผสาน **ความรู้จาก Phase 5 (Authentication — Part 041)** กับ **Phase 3–4 (CRUD/Forms)**
  เข้าด้วยกันเป็น flow เดียว ซึ่งสะท้อนการทำงานจริงที่ฟีเจอร์ต่างๆ ไม่ได้แยกทดสอบอย่างโดดเดี่ยว
  เสมอไป
- การใช้ `within("#posts")` ปิดท้ายเพื่อยืนยันว่าอีเมลของผู้เขียนที่ถูกต้องปรากฏ **เฉพาะใน
  ส่วนรายการโพสต์** ไม่ใช่แค่มีข้อความ `"newbie@example.com"` อยู่ที่ไหนก็ได้บนหน้า (ซึ่งอาจ
  ปรากฏใน header ข้อความต้อนรับด้วยเช่นกัน — การใช้ `within` ทำให้ assertion เจาะจงและ
  น่าเชื่อถือกว่า)

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เขียน system spec ทดสอบกรณี **login ด้วยรหัสผ่านผิด**: ยืนยันว่าหน้ายังคงอยู่ที่ฟอร์ม
   login เดิม (ไม่ redirect ไปไหน), มีข้อความ `"อีเมลหรือรหัสผ่านไม่ถูกต้อง"` ปรากฏ, และ
   `current_user` ยังคงเป็น `nil` (ตรวจทางอ้อมได้จากการที่ลิงก์ "เขียนโพสต์ใหม่" ไม่ปรากฏบน
   หน้ารายการโพสต์)
2. เขียน system spec ทดสอบว่า **ผู้ใช้ที่ยังไม่ login พยายามเข้าหน้า `/posts/new` โดยตรง**
   (ใช้ `visit new_post_path` ข้ามการคลิกลิงก์) จะถูก redirect ไปหน้า login พร้อมข้อความ
   แจ้งเตือน (ทดสอบว่า `before_action :require_authentication` ที่เรียนจาก Part 041 ยังคง
   ทำงานถูกต้องแม้เข้าถึงผ่านเบราว์เซอร์จริง ไม่ใช่แค่ request spec)
3. เพิ่ม comment ให้กับโพสต์เดียวกัน 2 ครั้งจาก 2 การทดสอบที่แยกกัน (คนละ `it` block) แล้ว
   ยืนยันด้วย `within(".comments")` ว่าแต่ละ scenario เห็นเฉพาะข้อมูลของตัวเอง (ใบ้: ถ้าใช้
   `config.use_transactional_fixtures = true` ที่ตั้งไว้ตั้งแต่ Part 046 ฐานข้อมูลจะถูก
   rollback ให้อัตโนมัติหลังแต่ละ `it` อยู่แล้ว — เขียน spec เพื่อ **พิสูจน์** ว่าเป็นจริง
   ด้วยตัวเอง)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจตำแหน่งของ **System Test / Feature Test** บนสุดของ Testing Pyramid — ช้าที่สุดแต่
  ทดสอบ "ประสบการณ์ผู้ใช้จริง" ผ่านเบราว์เซอร์ได้ครบวงจรที่สุด ต่างจาก model/request spec
  ที่เร็วกว่าแต่ทดสอบแค่ทีละชั้น
- สร้าง system spec ด้วย `rails generate rspec:system` และตั้งค่า `spec/rails_helper.rb`
  ให้เชื่อม Capybara เข้ากับ RSpec ได้ถูกต้อง
- เข้าใจความต่างของ driver `rack_test` (เร็ว ไม่รัน JS) กับ `selenium_chrome_headless`
  (เบราว์เซอร์จริง รัน JS ได้ ใช้เมื่อจำเป็นเท่านั้น) รวมถึงปัญหาเรื่องเวอร์ชัน
  Chrome/chromedriver ที่ต้องเข้ากันได้ ซึ่งเป็นปัญหาที่พบได้จริงเวลาตั้งค่า CI
- ใช้ Capybara DSL ครบชุด: `visit`, `click_link`/`click_button`, `fill_in`, `select`,
  `check`/`uncheck`, `choose`, และรู้จักพฤติกรรม substring-match ที่ต้องระวัง
- เลือกใช้ assertion ที่เหมาะสม: `have_content` สำหรับข้อความทั่วไป, `have_selector`/
  `have_css` เมื่อโครงสร้าง HTML มีผล
- เข้าใจกลไก **auto-waiting** ของ matcher `have_*` และรู้ว่าทำไมห้ามใช้ `sleep` รอ UI
  อัปเดตเด็ดขาด
- ใช้ `within` จำกัดขอบเขตการค้นหา element เมื่อหน้าเว็บมีส่วนที่ข้อความ/โครงสร้างซ้ำกัน
- Debug system test ที่ fail ได้อย่างเป็นระบบด้วย `save_and_open_page`, `page.save_screenshot`,
  และ `binding.pry`
- เข้าใจ **Ice Cream Cone Anti-pattern** และกฎปฏิบัติ "เขียน system test แค่ happy path ของ
  user journey หลักๆ เท่านั้น" ส่วน edge case ผลักลงไปที่ model/request spec แทน
- เขียน system test ที่ผสานความรู้ authentication (Phase 5) เข้ากับ CRUD (Phase 3–4) เป็น
  user journey เดียวที่สมบูรณ์ได้

**ต่อไป (Part 049):** เราจะเรียนรู้เรื่อง **Mocking/Stubbing** (จำลองพฤติกรรมของ object อื่น
แทนการเรียกของจริง เช่นตอนทดสอบโค้ดที่เรียก external API), **VCR** สำหรับบันทึกและเล่นซ้ำ
HTTP response ของ external API ในเทส (ไม่ต้องยิง request จริงทุกครั้งที่รัน test), และ
**test coverage** ด้วย **SimpleCov** เพื่อวัดว่าโค้ดส่วนไหนของแอปยังไม่มีเทสครอบคลุมเลย —
ปิดท้ายเครื่องมือสำคัญของเฟส 6 ก่อนเข้าสู่ Part 050 ที่จะรวบยอด TDD workflow เต็มรูปแบบ
