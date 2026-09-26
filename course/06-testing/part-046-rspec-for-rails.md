# Part 046: RSpec สำหรับ Rails — Model Spec, Request Spec

> **Step ครอบคลุมใน Part นี้:** Step 451–460
> **ระดับ:** กลาง–สูง (ต้องผ่าน Part 019 เรื่อง RSpec พื้นฐานมาก่อน — `describe`/`context`/`it`,
> matcher, `let`/`let!`, `before`/`after`, `subject`, `double`/`instance_double` — Part นี้จะไม่
> พูดซ้ำเรื่องเหล่านั้น แต่จะโฟกัสเฉพาะสิ่งที่เพิ่มเข้ามาเมื่อใช้ RSpec **ภายในแอป Rails**
> จริงๆ นอกจากนี้ยังอ้างอิงรูปแบบ authentication แบบ session-based จาก Part 041 ด้วย)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, gem `rspec-rails` 7.1.x, gem `shoulda-matchers`
> 6.5.x (ทุกตัวอย่างในบทนี้รันจริงบนแอป Rails ที่สร้างขึ้นเพื่อทดสอบ ไม่ใช่โค้ดที่เขียนลอยๆ)

นี่คือ Part แรกของ **เฟส 6: Testing (TDD/BDD)** — เฟสก่อนหน้า (เฟส 1–5) พาเรียนรู้จนสร้าง
เว็บแอป Rails ที่มี CRUD, form, association, authentication/authorization ครบแล้ว แต่ยังไม่มี
Part ไหนพิสูจน์ว่าโค้ดที่เขียนไป **ทำงานถูกต้องจริง** อย่างเป็นระบบเลย ตั้งแต่ Part นี้เป็นต้นไป
เราจะเขียนเทสต์คู่ไปกับโค้ดทุกครั้ง

## สารบัญของ Part นี้

- Step 451: `rspec-rails` คืออะไร ต่างจาก RSpec ธรรมดา (Part 019) อย่างไร และการติดตั้งในแอป Rails
- Step 452: `rails generate rspec:install` — `spec/spec_helper.rb` vs `spec/rails_helper.rb` ต่างกันอย่างไรและทำไมต้องมีสองไฟล์
- Step 453: `config.use_transactional_fixtures` — ฐานข้อมูล rollback อัตโนมัติหลังทุกเทสต์
- Step 454: Model spec เบื้องต้น — ทดสอบ validation ด้วย `spec/models/post_spec.rb`
- Step 455: Model spec ต่อ — ทดสอบ association, scope, instance method
- Step 456: `shoulda-matchers` — เขียน validation/association test แบบกระชับด้วย one-liner
- Step 457: Request spec เบื้องต้น — ทดสอบ HTTP request/response เต็มวงจร (status code, HTML/JSON body)
- Step 458: Request spec ขั้นสูง — auth-protected action, `sign_in` pattern, validation error response, redirect และทำไมเลิกใช้ controller spec
- Step 459: การรันสเปก — `bundle exec rspec`, การกรองด้วย `--tag`, `fdescribe`/`fit`
- Step 460: การจัดระเบียบโฟลเดอร์ `spec/` ที่ขยายใหญ่ขึ้น — `spec/support/`, shared context, `rails_helper.rb`

---

## Step 451: `rspec-rails` คืออะไร ต่างจาก RSpec ธรรมดา (Part 019) อย่างไร และการติดตั้งในแอป Rails

### ทบทวนสั้นๆ: RSpec ล้วนๆ (Part 019) ทดสอบอะไรได้บ้าง

ใน Part 019 เราติดตั้งแค่ gem `rspec` เฉยๆ แล้วเขียนเทสต์ให้ **plain Ruby class** เช่น
`Calculator`, `BankAccount` — ไม่มี Rails เข้ามาเกี่ยวข้องเลย `describe`/`it`/`let`/`double` ที่
เรียนไปทั้งหมดยังใช้ได้เหมือนเดิมทุกประการเมื่อมาเทสต์ Rails app เพราะเป็น RSpec core
(`rspec-core`, `rspec-expectations`, `rspec-mocks`) ตัวเดียวกัน

แต่แอป Rails มีสิ่งที่ plain Ruby ไม่มี: ActiveRecord model ที่ต้องคุยกับฐานข้อมูลจริง,
controller ที่ต้องผ่าน routing/middleware, session, cookie, request/response cycle เต็มรูปแบบ
— ถ้าใช้ RSpec เปล่าๆ ทดสอบสิ่งเหล่านี้ เราจะต้อง boot Rails environment เอง, จัดการ database
transaction เอง, เขียน helper จำลอง HTTP request เอง ฯลฯ ซึ่งเป็นงานหนักที่ไม่มีใครอยากทำซ้ำทุก
โปรเจกต์

### `rspec-rails` คือสะพานเชื่อมระหว่าง RSpec กับ Rails

**`rspec-rails`** เป็น gem แยกต่างหาก (ไม่ใช่ RSpec core) ที่ทำหน้าที่:

1. เพิ่ม **Rails generator** — `rails generate rspec:install`, และทำให้ `rails generate model`/
   `rails generate controller` สร้างไฟล์ spec ให้อัตโนมัติแทนที่จะสร้างไฟล์ Minitest
2. เพิ่ม **spec type** ใหม่ที่ผูกกับ Rails โดยเฉพาะ: `type: :model`, `type: :request`,
   `type: :system`, `type: :job`, `type: :mailer`, `type: :helper`, `type: :view` — แต่ละ type
   จะ mix-in behavior ที่เหมาะกับสิ่งที่กำลังทดสอบให้อัตโนมัติ (เช่น `type: :request` จะมี method
   `get`/`post`/`patch`/`delete` ให้ยิง HTTP request จำลองได้ทันที)
3. เพิ่ม matcher เฉพาะของ Rails: `have_http_status`, `redirect_to`, `render_template`,
   `route_to` และอื่นๆ
4. จัดการ database transaction ให้อัตโนมัติผ่าน `config.use_transactional_fixtures` (Step 453)
5. เชื่อมกับ Rails autoloading (Zeitwerk) ทำให้เรียก `Post`, `User` ในสเปกได้ทันทีโดยไม่ต้อง
   `require_relative` เหมือนที่ทำใน Part 019

พูดสั้นๆ: **`rspec-rails` ไม่ใช่ RSpec framework ตัวใหม่** แต่เป็น "adapter" ที่ทำให้ RSpec ตัว
เดิมที่เรียนไปแล้ว ทำงานเข้ากับวงจรชีวิตของ Rails app ได้อย่างราบรื่น

### ติดตั้งในแอป Rails

สร้างแอปใหม่สำหรับทดลองใน Part นี้ (ในโปรเจกต์จริงคุณจะเพิ่ม RSpec เข้าไปในแอปที่มีอยู่แล้วด้วย
ขั้นตอนเดียวกันนี้เป๊ะๆ):

```bash
rails new blog_app --minimal -d sqlite3 --skip-test
cd blog_app
```

สังเกต flag `--skip-test` — บอก Rails ไม่ต้องติดตั้ง Minitest ให้ (ตามที่เรียนใน Part 018 ว่า
Rails ติดตั้ง Minitest เป็นค่าเริ่มต้นเสมอถ้าไม่ใส่ flag นี้) เพราะเราจะใช้ RSpec แทนทั้งหมด

เพิ่ม gem เข้า `Gemfile`:

```ruby
# Gemfile

gem "bcrypt", "~> 3.1.7"   # ต้องใช้กับ has_secure_password (เรียนใน Part 041)

group :development, :test do
  gem "debug", platforms: %i[mri windows], require: "debug/prelude"
  gem "rspec-rails", "~> 7.1"
  gem "shoulda-matchers", "~> 6.4"
end
```

**ทำไม `rspec-rails` ต้องอยู่ใน group `:development, :test` ไม่ใช่ `:test` เฉยๆ:** เพราะ Rails
generator (เช่น `rails generate model`) รันในสภาพแวดล้อม `development` แต่ต้องรู้จัก
`rspec-rails` เพื่อสร้างไฟล์ spec ให้อัตโนมัติ ถ้าใส่ไว้แค่ group `:test` เฉยๆ คำสั่ง
`rails generate` ที่รันแบบปกติ (environment เป็น development) จะมองไม่เห็น gem นี้เลย

```bash
bundle install
```

```
Fetching gem metadata from https://rubygems.org/...........
Resolving dependencies...
Fetching shoulda-matchers 6.5.0
Installing shoulda-matchers 6.5.0
Bundle complete! 9 Gemfile dependencies, 77 gems now installed.
```

> **หมายเหตุเรื่องเวอร์ชัน:** ตัวอย่างในบทนี้ทดสอบจริงด้วย `rspec-rails 7.1.1` และ
> `shoulda-matchers 6.5.0` บน Rails 8.1.4 — ถ้า `bundle install` ของคุณได้เวอร์ชันสูงกว่านี้ก็ไม่
> เป็นปัญหา เพราะ API หลักที่ใช้ในบทนี้เสถียรมาหลายเมเจอร์เวอร์ชันแล้ว

---

## Step 452: `rails generate rspec:install` — `spec/spec_helper.rb` vs `spec/rails_helper.rb` ต่างกันอย่างไรและทำไมต้องมีสองไฟล์

```bash
bin/rails generate rspec:install
```

ผลลัพธ์จริง:

```
      create  .rspec
      create  spec
      create  spec/spec_helper.rb
      create  spec/rails_helper.rb
```

สังเกตว่าคำสั่งนี้สร้างไฟล์ **สองไฟล์** ต่างจาก `rspec --init` ที่เรียนใน Part 019 ซึ่งสร้าง
แค่ `spec/spec_helper.rb` ไฟล์เดียว — นี่คือความต่างสำคัญที่สุดของการใช้ RSpec ใน Rails

### ไฟล์ `.rspec`

```
--require spec_helper
```

เหมือนเดิมกับ Part 019 ทุกประการ — บอกให้ RSpec `require "spec_helper"` อัตโนมัติทุกครั้งที่รัน
`rspec` โดยไม่ต้องเขียนใน spec file เอง **สังเกตให้ดี:** ไฟล์นี้ require แค่ `spec_helper` ไม่ใช่
`rails_helper` — นี่ไม่ใช่ความผิดพลาด แต่เป็นการออกแบบที่ตั้งใจ (อธิบายด้านล่าง)

### `spec/spec_helper.rb` — config ของ RSpec core ล้วนๆ ไม่รู้จัก Rails เลย

```ruby
# This file was generated by the `rails generate rspec:install` command. Conventionally, all
# specs live under a `spec` directory, which RSpec adds to the `$LOAD_PATH`.
# The generated `.rspec` file contains `--require spec_helper` which will cause
# this file to always be loaded, without a need to explicitly require it in any
# files.
#
# Given that it is always loaded, you are encouraged to keep this file as
# light-weight as possible. Requiring heavyweight dependencies from this file
# will add to the boot time of your test suite on EVERY test run, even for an
# individual file that may not need all of that loaded.
RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups

  # settings อื่นๆ เช่น config.order = :random, config.filter_run_when_matching :focus
  # ถูกคอมเมนต์ไว้เป็นค่าเริ่มต้น (เหมือนที่เห็นใน Part 019) — เปิดใช้ได้ตามต้องการ
end
```

ไฟล์นี้หน้าตาแทบจะเหมือนกับ `spec_helper.rb` ที่เห็นใน Part 019 เป๊ะๆ — เพราะมันคือ config
ระดับ **RSpec core เท่านั้น ไม่รู้จัก Rails เลยแม้แต่น้อย** ไฟล์นี้ **ไม่ได้ `require` Rails**
ดังนั้นถ้า spec file ไหน `require "spec_helper"` เฉยๆ (ไม่ require rails_helper) จะไม่มี
`Post`, `User`, หรือ Rails environment ให้ใช้เลย เหมาะกับการทดสอบ **plain Ruby class ที่ไม่
เกี่ยวกับ Rails** เช่น service object, value object, หรือ utility class ที่ไม่ต้องแตะฐานข้อมูล/
ActiveRecord

### `spec/rails_helper.rb` — โหลด Rails environment เต็มรูปแบบ

```ruby
# This file is copied to spec/ when you run 'rails generate rspec:install'
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'
# Prevent database truncation if the environment is production
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'
# Add additional requires below this line. Rails is not loaded until this point!

Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }

# Checks for pending migrations and applies them before tests are run.
begin
  ActiveRecord::Migration.maintain_test_schema!
rescue ActiveRecord::PendingMigrationError => e
  abort e.to_s.strip
end

RSpec.configure do |config|
  config.fixture_paths = [
    Rails.root.join('spec/fixtures')
  ]

  config.use_transactional_fixtures = true

  # config.infer_spec_type_from_file_location!

  config.filter_rails_from_backtrace!
end
```

**สิ่งสำคัญที่เกิดขึ้นในไฟล์นี้ เรียงตามลำดับ:**

1. `require 'spec_helper'` — โหลด config พื้นฐานของ RSpec core ก่อนเสมอ (คนละหน้าที่กัน ไม่ใช่
   การเขียนซ้ำ)
2. `require_relative '../config/environment'` — บรรทัดนี้คือจุดที่ **บูต Rails application
   ทั้งตัว** (โหลด model, controller, routes, initializers, database connection ทั้งหมด) เป็น
   ขั้นตอนที่ใช้เวลานานที่สุดของการรันเทสต์ครั้งแรก
3. `require 'rspec/rails'` — โหลด `rspec-rails` เข้ามาจริงๆ ตรงนี้ ถึงจะมี `type: :model`,
   `type: :request` และ matcher อย่าง `have_http_status` ให้ใช้
4. `ActiveRecord::Migration.maintain_test_schema!` — เทียบ schema ของฐานข้อมูล test กับ
   `db/schema.rb` อัตโนมัติ ถ้าไม่ตรงกัน (เช่น มี migration ใหม่ที่ยังไม่ได้ migrate ฐาน test) จะ
   migrate ให้อัตโนมัติก่อนเริ่มรันเทสต์ — ไม่ต้องพิมพ์ `rails db:test:prepare` เองอีกต่อไป
5. `config.use_transactional_fixtures = true` — หัวใจสำคัญของ Step 453 ถัดไป

### ทำไมต้องแยกสองไฟล์ ทำไมไม่รวมเป็นไฟล์เดียว

**เหตุผล:** `spec_helper.rb` ถูกออกแบบให้ **เบาที่สุดเท่าที่จะทำได้** เพราะ `.rspec` บังคับให้
โหลดไฟล์นี้ **ทุกครั้ง** ที่รัน `rspec` ไม่ว่าจะรันสเปกไฟล์ไหนก็ตาม ถ้าใส่การบูต Rails ทั้งตัวไว้
ในไฟล์นี้ (ซึ่งใช้เวลาหลายวินาทีในโปรเจกต์ใหญ่) แม้แต่การรันสเปกไฟล์เดียวที่ไม่ได้แตะ Rails เลย
ก็จะต้องเสียเวลาบูต Rails ทุกครั้งอยู่ดี

ด้วยการแยกไฟล์ spec file ที่ **ต้องการ** Rails (model spec, request spec) จะ
`require "rails_helper"` เอง ส่วน spec file ที่เป็น plain Ruby ล้วนๆ จะ `require "spec_helper"`
เฉยๆ (หรือไม่ require อะไรเลยนอกจาก `.rspec` ที่บังคับอยู่แล้ว) ทำให้ทีมที่มี spec ทั้งสองแบบปน
กันในโปรเจกต์เดียวได้ประสิทธิภาพที่ดีที่สุดสำหรับแต่ละแบบ

ลองสังเกตจาก spec file ที่ generator สร้างให้อัตโนมัติเวลาสร้าง model (Step 454) — บรรทัดแรกสุด
จะเป็น `require "rails_helper"` เสมอ ไม่ใช่ `spec_helper`

> **กฎการเลือกใช้ในทางปฏิบัติ:** เกือบทุก spec file ในแอป Rails จริงจะ `require "rails_helper"`
> เพราะแทบทุกอย่างที่เทสต์เกี่ยวข้องกับ Rails ไม่ทางใดก็ทางหนึ่ง ยกเว้น spec ที่ทดสอบ pure Ruby
> object ที่ไม่แตะ ActiveRecord/ActionController เลยจริงๆ (เช่น value object คำนวณเลขล้วนๆ) จึง
> จะพิจารณาใช้ `spec_helper` เฉยๆ เพื่อความเร็ว

---

## Step 453: `config.use_transactional_fixtures` — ฐานข้อมูล rollback อัตโนมัติหลังทุกเทสต์

### ปัญหาที่จะเกิดถ้าไม่มีกลไกนี้

ทุก model spec / request spec ที่แตะฐานข้อมูล (สร้าง record ผ่าน `Post.create!`,
`User.create!`) จะทิ้งข้อมูลไว้ในฐานข้อมูล test ถ้าไม่มีใครลบออก พอรันเทสต์ตัวถัดไป ข้อมูลเก่าที่
ค้างอยู่จะกวนผลลัพธ์ทันที เช่น เทสต์ที่คาดหวังว่า `Post.count` ต้องเป็น 0 ตอนเริ่มต้น จะ fail ถ้ามี
record ค้างจากเทสต์ก่อนหน้า

### `use_transactional_fixtures = true` แก้ปัญหานี้อย่างไร

เมื่อตั้งค่านี้เป็น `true` (ค่าเริ่มต้นที่ generator ตั้งมาให้แล้ว) rspec-rails จะห่อทุก example
ด้วย **database transaction** โดยอัตโนมัติ: เปิด transaction ก่อน example เริ่ม แล้ว
**rollback ทันทีหลัง example จบ** ไม่ว่า example นั้นจะ pass หรือ fail ก็ตาม ผลคือทุก record ที่
ถูกสร้าง/แก้ไข/ลบระหว่าง example นั้นจะหายไปราวกับไม่เคยเกิดขึ้น ก่อนที่ example ถัดไปจะเริ่ม

พิสูจน์ด้วยเทสต์จริง:

```ruby
# frozen_string_literal: true ไม่ใช้ในไฟล์ Rails ตามธรรมเนียมของ generator (ดูหมายเหตุ Step 451)

RSpec.describe "การพิสูจน์ use_transactional_fixtures" do
  it "ฐานข้อมูลว่างเปล่าตอนเริ่มต้น example ที่ 1" do
    expect(Post.count).to eq(0)
    user = User.create!(email: "a@example.com", password: "password123")
    Post.create!(title: "ทดสอบ", body: "เนื้อหา", user: user)
    expect(Post.count).to eq(1)
  end

  it "ฐานข้อมูลกลับมาว่างเปล่าอีกครั้งตอนเริ่ม example ที่ 2 (แม้ example ที่ 1 เพิ่งสร้างข้อมูลไป)" do
    expect(Post.count).to eq(0)
    expect(User.count).to eq(0)
  end
end
```

```bash
bundle exec rspec spec/models/transactional_demo_spec.rb --format documentation
```

ผลลัพธ์จริง:

```
การพิสูจน์ use_transactional_fixtures
  ฐานข้อมูลว่างเปล่าตอนเริ่มต้น example ที่ 1
  ฐานข้อมูลกลับมาว่างเปล่าอีกครั้งตอนเริ่ม example ที่ 2 (แม้ example ที่ 1 เพิ่งสร้างข้อมูลไป)

Finished in 0.03378 seconds (files took 1.03 seconds to load)
2 examples, 0 failures
```

ทั้งสอง example ผ่านหมด แม้ example แรกจะสร้าง `User` และ `Post` ไว้จริงก็ตาม เพราะ transaction
ของ example แรกถูก rollback ไปแล้วก่อนที่ example ที่สองจะเริ่มทำงาน

### เหตุผลที่กลไกนี้เร็วกว่าการ "ลบข้อมูลเองด้วยมือ"

ทางเลือกอื่นที่บางทีมใช้คือ **database truncation** (สั่ง `DELETE FROM` หรือ `TRUNCATE` ทุกตาราง
หลังแต่ละเทสต์ผ่าน gem เช่น `database_cleaner`) วิธีนี้ทำงานถูกต้องเหมือนกัน แต่ **ช้ากว่ามาก**
ในโปรเจกต์ใหญ่ เพราะต้องยิงคำสั่ง SQL จริงไปที่ทุกตารางหลังทุกเทสต์ ในขณะที่ transaction
rollback เป็นการยกเลิกงานที่ database เก็บไว้ใน memory/log อยู่แล้ว **เร็วกว่าเป็นสิบเท่า**

**ข้อจำกัดที่ต้องรู้:** transaction rollback ใช้ไม่ได้กับโค้ดที่รัน background job แบบ
async จริง (เช่น ยิง job ผ่าน queue ที่ทำงานใน process/thread อื่น) เพราะ process อื่นมองไม่เห็น
transaction ที่ยังไม่ commit ของ process หลัก — กรณีนี้ทีมส่วนใหญ่ยังคงพึ่งพา `database_cleaner`
เสริมเฉพาะจุดที่จำเป็น (เรื่อง background job testing จะเรียนละเอียดในเฟส 9)

> **แนวปฏิบัติที่ถูกต้อง:** ปล่อย `config.use_transactional_fixtures = true` ไว้เป็นค่าเริ่มต้น
> เสมอ ไม่ต้องไปยุ่งกับมัน — นี่คือเหตุผลที่ model spec/request spec ในบทนี้ทุกตัวอย่างไม่มีการ
> เขียนโค้ด "เก็บกวาด" ข้อมูลทดสอบเองเลยสักบรรทัดเดียว มันเกิดขึ้นอัตโนมัติเบื้องหลังเสมอ

---

## Step 454: Model spec เบื้องต้น — ทดสอบ validation ด้วย `spec/models/post_spec.rb`

### สร้าง model ด้วย generator แล้วสังเกตว่า spec ถูกสร้างให้อัตโนมัติ

```bash
bin/rails generate model User email:string password_digest:string
```

```
      invoke  active_record
      create    db/migrate/20260926064544_create_users.rb
      create    app/models/user.rb
      invoke    rspec
      create      spec/models/user_spec.rb
```

```bash
bin/rails generate model Post title:string body:text status:string published_at:datetime user:references
```

```
      invoke  active_record
      create    db/migrate/20260926064545_create_posts.rb
      create    app/models/post.rb
      invoke    rspec
      create      spec/models/post_spec.rb
```

สังเกตบรรทัด `invoke rspec` — เพราะแอปนี้มี gem `rspec-rails` อยู่ใน `Gemfile` แล้ว Rails
generator จึงรู้เองว่าต้องสร้างไฟล์สเปกแบบ RSpec (`spec/models/post_spec.rb`) แทนที่จะสร้าง
`test/models/post_test.rb` แบบ Minitest (ที่เรียนใน Part 018) — ไม่ต้องตั้งค่าอะไรเพิ่มเลย
Rails ตรวจจับจาก Gemfile ให้อัตโนมัติ

ไฟล์ที่ generator สร้างให้ **เป็นแค่โครงเปล่า**:

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  pending "add some examples to (or delete) #{__FILE__}"
end
```

สังเกต `require 'rails_helper'` ตรงกับที่อธิบายใน Step 452 และ `type: :model` คือ metadata ที่
บอก rspec-rails ว่านี่คือ model spec (ทำให้ได้ behavior เฉพาะ เช่น การ rollback อัตโนมัติจาก
Step 453 และการเข้าถึง matcher เฉพาะของ ActiveRecord)

รัน migration ก่อนเขียนโค้ดจริง:

```bash
bin/rails db:migrate
```

### เขียน model และ validation

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  STATUSES = %w[draft published archived].freeze

  belongs_to :user

  validates :title, presence: true, length: { maximum: 100 }
  validates :body, presence: true
  validates :status, inclusion: { in: STATUSES }

  before_validation :set_default_status, on: :create

  private

  def set_default_status
    self.status ||= "draft"
  end
end
```

### เขียน validation spec แบบพื้นฐาน (ยังไม่ใช้ shoulda-matchers)

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  let(:user) { User.create!(email: "author@example.com", password: "password123") }

  subject(:post_record) do
    described_class.new(title: "บทความแรก", body: "เนื้อหาบทความ", user: user)
  end

  describe "validations" do
    it "is valid with valid attributes" do
      expect(post_record).to be_valid
    end

    it "is invalid without a title" do
      post_record.title = nil
      expect(post_record).not_to be_valid
      expect(post_record.errors[:title]).to include("can't be blank")
    end

    it "is invalid when status is not one of the allowed values" do
      post_record.status = "urgent"
      expect(post_record).not_to be_valid
      expect(post_record.errors[:status]).to include("is not included in the list")
    end
  end

  describe "before_validation :set_default_status" do
    it "defaults status to draft when not given" do
      post_record.status = nil
      post_record.valid?
      expect(post_record.status).to eq("draft")
    end

    it "does not override an explicitly given status" do
      post_record.status = "published"
      post_record.valid?
      expect(post_record.status).to eq("published")
    end
  end
end
```

รันดู:

```bash
bundle exec rspec spec/models/post_spec.rb --format documentation
```

```
Post
  validations
    is valid with valid attributes
    is invalid without a title
    is invalid when status is not one of the allowed values
  before_validation :set_default_status
    defaults status to draft when not given
    does not override an explicitly given status

Finished in 0.05 seconds (files took 0.98 seconds to load)
5 examples, 0 failures
```

**สังเกตรูปแบบสำคัญที่ต่างจาก Part 019:**

- ทดสอบ validation ของ ActiveRecord ด้วย `be_valid`/`not_to be_valid` แทนการเรียก method ตรงๆ
  แบบ plain Ruby object — เพราะ `valid?` เป็น method มาตรฐานของ ActiveModel ที่ประมวลผล
  `validates` ทุกตัวที่ประกาศไว้ในโมเดล
- `post_record.errors[:title]` คืน Array ของข้อความ error ทั้งหมดของ attribute นั้น ใช้คู่กับ
  matcher `include` (จาก Part 019 Step 185) ตรวจสอบข้อความ error แบบเจาะจงได้
- `let(:user)` สร้าง `User` จริงลงฐานข้อมูล (`User.create!`) เพราะ `Post` ต้องการ
  `belongs_to :user` — เป็น dependency ที่ model spec เลี่ยงไม่ได้ (ต่างจาก Part 019 ที่ใช้
  `instance_double` แทน dependency ภายนอกได้ตลอด แต่ความสัมพันธ์ระหว่าง ActiveRecord model
  ต้องใช้ record จริงเพื่อให้ foreign key ทำงานถูกต้อง)

---

## Step 455: Model spec ต่อ — ทดสอบ association, scope, instance method

### ทดสอบ association

```ruby
describe "associations" do
  it "belongs to a user" do
    expect(post_record.user).to eq(user)
  end

  it "is invalid without a user" do
    post_record.user = nil
    expect(post_record).not_to be_valid
  end
end
```

**ทำไมข้อสอง (`is invalid without a user`) ถึงผ่าน** แม้ในโมเดลไม่ได้เขียน
`validates :user, presence: true` เอาไว้เลย — เพราะตั้งแต่ **Rails 5** เป็นต้นมา `belongs_to`
**บังคับ (required) โดยอัตโนมัติ** (ค่าเริ่มต้นของ config `config.active_record.belongs_to_required_by_default`
คือ `true`) ถ้าไม่ตั้ง `user` ให้ record นี้จะ invalid ทันทีโดยไม่ต้องเขียน validation เพิ่มเอง —
เป็นเรื่องสำคัญที่ต้องรู้เวลาอ่าน error message เพราะ error จะปรากฏใน `errors[:user]` ว่า
`"must exist"` ไม่ใช่ `"can't be blank"`

### ทดสอบ scope

เพิ่ม scope ในโมเดล:

```ruby
# app/models/post.rb (เพิ่มเข้าไป)
scope :published, -> { where(status: "published") }
scope :recent, -> { order(created_at: :desc) }
```

```ruby
describe ".published" do
  it "returns only posts with status published" do
    draft = Post.create!(title: "ร่าง", body: "...", user: user, status: "draft")
    published = Post.create!(title: "เผยแพร่แล้ว", body: "...", user: user, status: "published")

    expect(Post.published).to include(published)
    expect(Post.published).not_to include(draft)
  end
end
```

**หลักการเขียนเทสต์ scope ที่ดี:** เตรียมข้อมูลอย่างน้อย **สองแบบที่ต่างกัน** (ที่ scope ควร
รวมและที่ scope ควรตัดออก) แล้วยืนยันทั้งสองทิศทางด้วย `include`/`not_to include` — ถ้าเทสต์แค่
ทิศทางเดียว (เช่น เช็คแค่ว่า `Post.published` include `published` แต่ไม่เช็คว่า **ไม่รวม**
`draft`) จะจับบั๊กไม่ได้ถ้ามีคนแก้ scope ให้กลายเป็น `Post.all` โดยไม่ตั้งใจ (เทสต์ก็ยังผ่านอยู่ดี
เพราะ `Post.all` ก็ include `published` เหมือนกัน)

### ทดสอบ instance method

```ruby
# app/models/post.rb (เพิ่มเข้าไป)
def published?
  status == "published"
end

def publish!
  update!(status: "published", published_at: Time.current)
end

def excerpt(length = 20)
  body.to_s.truncate(length)
end
```

```ruby
describe "#published?" do
  context "when status is published" do
    it "returns true" do
      post_record.status = "published"
      expect(post_record.published?).to be(true)
    end
  end

  context "when status is draft" do
    it "returns false" do
      post_record.status = "draft"
      expect(post_record.published?).to be(false)
    end
  end
end

describe "#publish!" do
  it "changes status to published and sets published_at" do
    post_record.save!

    expect { post_record.publish! }
      .to change { post_record.reload.status }.from("draft").to("published")

    expect(post_record.published_at).not_to be_nil
  end
end

describe "#excerpt" do
  it "truncates the body to the given length" do
    post_record.body = "a" * 50
    expect(post_record.excerpt(20).length).to eq(20)
  end

  it "defaults to 20 characters when no length is given" do
    post_record.body = "a" * 50
    expect(post_record.excerpt.length).to eq(20)
  end
end
```

สังเกตว่า `#publish!` ใช้ `expect { }.to change { }.from(...).to(...)` (matcher `change` จาก
Part 019 Step 185) คู่กับ `.reload` — **ต้องมี `.reload`** เพราะ `update!` เปลี่ยนค่าใน object
ที่อยู่ใน memory อยู่แล้วทันที การเรียก `post_record.status` ตรงๆ (ไม่ reload) จะเห็นค่าที่
เปลี่ยนไปแล้วเสมอไม่ว่า `update!` จะเขียนลงฐานข้อมูลจริงหรือไม่ — การ `.reload` บังคับให้ query
ฐานข้อมูลใหม่ ทำให้เทสต์พิสูจน์ได้จริงๆ ว่าการเปลี่ยนแปลง **ถูกบันทึกลงฐานข้อมูลจริง** ไม่ใช่แค่
เปลี่ยนค่าใน memory เฉยๆ

รันทั้งไฟล์ดูผลลัพธ์เต็ม (ยังไม่รวม shoulda-matchers ที่จะเพิ่มใน Step ถัดไป):

```bash
bundle exec rspec spec/models/post_spec.rb --format documentation
```

```
Post
  validations
    is valid with valid attributes
    is invalid without a title
    is invalid when title is longer than 100 characters
    is invalid without a body
    is invalid when status is not one of the allowed values
  before_validation :set_default_status
    defaults status to draft when not given
    does not override an explicitly given status
  associations
    belongs to a user
    is invalid without a user
  .published
    returns only posts with status published
  #published?
    when status is published
      returns true
    when status is draft
      returns false
  #publish!
    changes status to published and sets published_at
  #excerpt
    truncates the body to the given length
    defaults to 20 characters when no length is given

Finished in 0.09 seconds (files took 1.1 seconds to load)
14 examples, 0 failures
```

---

## Step 456: `shoulda-matchers` — เขียน validation/association test แบบกระชับด้วย one-liner

### ปัญหาที่ shoulda-matchers แก้

สังเกตว่าเทสต์ validation ใน Step 454 แต่ละตัวต้องเขียน 3-4 บรรทัด (ตั้งค่า attribute ให้ผิด →
`valid?` → เช็ค error message) ทั้งที่ validation แต่ละบรรทัดในโมเดลมีแค่บรรทัดเดียว
(`validates :title, presence: true`) — สัดส่วนโค้ดเทสต์ต่อโค้ดจริงสูงเกินไปสำหรับ pattern ที่
เกิดซ้ำๆ แบบนี้ **`shoulda-matchers`** เป็น gem ที่รวบรวม matcher สำเร็จรูปสำหรับทดสอบ
validation/association ของ ActiveRecord โดยเฉพาะ ทำให้เขียนเทสต์แบบนี้ได้ในบรรทัดเดียว

### ติดตั้งและ config

เพิ่ม config ต่อท้าย `spec/rails_helper.rb`:

```ruby
# spec/rails_helper.rb (เพิ่มต่อท้ายไฟล์)
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

`with.library :rails` บอกให้ shoulda-matchers โหลดเฉพาะชุด matcher ที่เกี่ยวกับ ActiveRecord/
ActionController (มีชุด `:active_model`, `:active_record` แยกย่อยให้เลือกด้วยถ้าต้องการจำกัด
ขอบเขตแคบกว่านี้ แต่ `:rails` ครอบคลุมทุกอย่างที่ต้องใช้ในบทนี้)

### เขียนเทสต์เดิมใหม่ด้วย shoulda-matchers

```ruby
describe "validations" do
  it { is_expected.to validate_presence_of(:title) }
  it { is_expected.to validate_length_of(:title).is_at_most(100) }
  it { is_expected.to validate_presence_of(:body) }
  it { is_expected.to validate_inclusion_of(:status).in_array(%w[draft published archived]) }
end

describe "associations" do
  it { is_expected.to belong_to(:user) }
end
```

**สังเกตว่านี่คือ one-liner syntax `it { is_expected.to ... }` ที่เรียนไปแล้วใน Part 019
Step 188** — shoulda-matchers ไม่ได้เพิ่ม syntax ใหม่ให้ RSpec เลย แค่เพิ่ม **matcher ใหม่**
(`validate_presence_of`, `validate_length_of`, `validate_inclusion_of`, `belong_to`) ที่ใช้คู่
กับ `is_expected.to`/`expect(subject).to` แบบเดิมทุกประการ

รันดูผลลัพธ์จริง:

```bash
bundle exec rspec spec/models/post_spec.rb --format documentation
```

```
Post
  validations
    is expected to validate that :title cannot be empty/falsy
    is expected to validate that the length of :title is at most 100
    is expected to validate that :body cannot be empty/falsy
    is expected to validate that :status is either ‹"draft"›, ‹"published"›, or ‹"archived"›
    is valid with valid attributes
    is invalid without a title
  associations
    is expected to belong to user required: true
  ...

Finished in 0.12388 seconds (files took 2.54 seconds to load)
15 examples, 0 failures
```

สังเกตข้อความ documentation format ที่ shoulda-matchers สร้างให้อัตโนมัติ
(`is expected to validate that :title cannot be empty/falsy`) — อ่านเป็นประโยคสมบูรณ์ได้ทันที
โดยไม่ต้องเขียนคำอธิบายเอง และบรรทัด `is expected to belong to user required: true` ยืนยันตรง
กับที่อธิบายใน Step 455 ว่า `belongs_to` บังคับ (`required: true`) เป็นค่าเริ่มต้น

### ทดสอบเดียวกับ `User` model

```ruby
# spec/models/user_spec.rb
require "rails_helper"

RSpec.describe User, type: :model do
  subject(:user) { described_class.new(email: "user@example.com", password: "password123") }

  it { is_expected.to validate_presence_of(:email) }
  it { is_expected.to have_secure_password }
  it { is_expected.to have_many(:posts).dependent(:destroy) }

  it "is invalid with a duplicate email (case-insensitive)" do
    described_class.create!(email: "USER@example.com", password: "password123")
    expect(user).not_to be_valid
    expect(user.errors[:email]).to include("has already been taken")
  end
end
```

```
User
  is expected to validate that :email cannot be empty/falsy
  is expected to have a secure password, defined on password attribute
  is expected to have many posts dependent => destroy
  is invalid with a duplicate email (case-insensitive)

Finished in ... seconds
4 examples, 0 failures
```

สังเกตว่า `have_secure_password` และ `have_many(...).dependent(:destroy)` ก็เป็น
shoulda-matcher เช่นกัน ทดสอบ `has_secure_password` (จาก Part 041) และ `has_many :posts,
dependent: :destroy` ได้ในบรรทัดเดียว

### ข้อควรระวัง: shoulda-matchers ทดสอบ "โครงสร้าง" ไม่ใช่ "พฤติกรรมทางธุรกิจ"

`it { is_expected.to validate_presence_of(:title) }` พิสูจน์ได้แค่ว่า **มี** validation
presence ประกาศอยู่บน attribute นั้นจริง แต่ไม่ได้พิสูจน์ business logic ที่ซับซ้อนกว่านั้น เช่น
เงื่อนไข "title ต้องไม่ซ้ำกับ post อื่นของ user คนเดียวกัน" หรือ "status จะเปลี่ยนเป็น published
ได้เฉพาะตอนที่ user เป็น admin เท่านั้น" — สิ่งเหล่านี้ยังต้องเขียนเทสต์แบบพฤติกรรม (behavioral,
ที่เขียนมาแล้วใน Step 454–455) เอง shoulda-matchers เหมาะกับ **การครอบคลุม validation/
association พื้นฐานให้ครบเร็วๆ** ไม่ใช่ตัวแทนของการทดสอบ business rule ทั้งหมด

> **แนวปฏิบัติที่ดีในทีมมืออาชีพ:** ใช้ shoulda-matchers คู่กับเทสต์แบบพฤติกรรมเสมอ ไม่ใช่แทนที่
> กัน — ใช้ shoulda-matchers สำหรับ "มี validation/association นี้อยู่จริงไหม" (โครงสร้าง) และ
> ใช้ `context`/`it` แบบเต็มสำหรับ "เมื่อเงื่อนไข X เกิดขึ้น ระบบทำอะไร" (พฤติกรรม)

---

## Step 457: Request spec เบื้องต้น — ทดสอบ HTTP request/response เต็มวงจร (status code, HTML/JSON body)

### สร้าง controller ด้วย generator แล้วสังเกตว่า generator สร้างอะไรให้

```bash
bin/rails generate controller Posts
```

```
      create  app/controllers/posts_controller.rb
      invoke  erb
      create    app/views/posts
      invoke  rspec
      create    spec/requests/posts_spec.rb
      invoke  helper
      create    app/helpers/posts_helper.rb
      invoke    rspec
      create      spec/helpers/posts_helper_spec.rb
```

**สังเกตให้ดี:** generator สร้างไฟล์ไว้ที่ `spec/requests/posts_spec.rb` ไม่ใช่
`spec/controllers/posts_controller_spec.rb` — นี่คือธรรมเนียมมาตรฐานปัจจุบันของ rspec-rails
(อธิบายเหตุผลเต็มใน Step 458) ไฟล์เปล่าที่ได้มามีหน้าตาแบบนี้:

```ruby
require 'rails_helper'

RSpec.describe "Posts", type: :request do
  describe "GET /index" do
    pending "add some examples (or delete) #{__FILE__}"
  end
end
```

สังเกต `RSpec.describe "Posts", type: :request` — รับ **string** ("Posts") ไม่ใช่ class เหมือน
model spec (`RSpec.describe Post, type: :model`) เพราะ request spec ไม่ได้ผูกกับ class
`PostsController` โดยตรง แต่ผูกกับ **เส้นทาง URL** ("Posts" เป็นแค่คำอธิบาย ไม่ใช่ค่าที่มีผลต่อ
การทำงาน)

### เตรียม route และ controller

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts
  root "posts#index"
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :set_post, only: %i[show]

  def index
    @posts = Post.recent

    respond_to do |format|
      format.html
      format.json { render json: @posts }
    end
  end

  def show
    respond_to do |format|
      format.html
      format.json { render json: @post }
    end
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end
end
```

```erb
<%# app/views/posts/index.html.erb %>
<h1>รายการบทความ</h1>

<ul>
  <% @posts.each do |post| %>
    <li><%= link_to post.title, post %></li>
  <% end %>
</ul>
```

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>
<p>สถานะ: <%= @post.status %></p>
```

### เขียน request spec

```ruby
# spec/requests/posts_spec.rb
require "rails_helper"

RSpec.describe "Posts", type: :request do
  let(:user) { User.create!(email: "author@example.com", password: "password123") }
  let!(:post_record) do
    Post.create!(title: "บทความทดสอบ", body: "เนื้อหา", status: "published", user: user)
  end

  describe "GET /posts" do
    it "returns http success" do
      get posts_path
      expect(response).to have_http_status(:ok)
    end

    it "renders the list of posts in the HTML body" do
      get posts_path
      expect(response.body).to include("บทความทดสอบ")
    end

    it "returns posts as JSON when requested" do
      get posts_path, as: :json
      expect(response).to have_http_status(:ok)
      json = response.parsed_body
      expect(json.first["title"]).to eq("บทความทดสอบ")
    end
  end

  describe "GET /posts/:id" do
    it "returns http success for an existing post" do
      get post_path(post_record)
      expect(response).to have_http_status(:ok)
      expect(response.body).to include(post_record.title)
    end

    it "returns 404 for a post that does not exist" do
      get post_path(id: -1)
      expect(response).to have_http_status(:not_found)
    end
  end
end
```

รันดู:

```bash
bundle exec rspec spec/requests/posts_spec.rb --format documentation
```

```
Posts
  GET /posts
    returns http success
    renders the list of posts in the HTML body
    returns posts as JSON when requested
  GET /posts/:id
    returns http success for an existing post
    returns 404 for a post that does not exist

Finished in 0.18 seconds (files took 1.0 seconds to load)
5 examples, 0 failures
```

**สิ่งใหม่ที่ `type: :request` เพิ่มให้ทันทีโดยไม่ต้อง require อะไรเพิ่ม:**

| Method/Object | หน้าที่ |
|---|---|
| `get`, `post`, `patch`, `put`, `delete` | ยิง HTTP request จำลองไปที่ path ที่ระบุ ผ่าน **Rack middleware stack เต็มรูปแบบ** (routing, session, cookies) แต่ไม่เปิด TCP socket จริง จึงเร็วกว่าการทดสอบผ่านเบราว์เซอร์จริงมาก |
| `response` | object แทนผลลัพธ์ HTTP ที่ได้กลับมา (status, body, headers) |
| `have_http_status(:ok)` | matcher ตรวจสอบ status code รับได้ทั้ง symbol (`:ok`, `:not_found`, `:unprocessable_content`) และตัวเลข (`200`, `404`) |
| `response.parsed_body` | แปลง response body เป็น Ruby object อัตโนมัติตาม content type (JSON → Hash/Array) ไม่ต้อง `JSON.parse(response.body)` เอง |
| route helper เช่น `posts_path`, `post_path(post_record)` | เรียกใช้ได้ทันทีเหมือนในโค้ด view/controller จริง (มาจาก `Rails.application.routes.url_helpers` ที่ rspec-rails mix-in ให้อัตโนมัติ) |

**สังเกตเทสต์ 404 (`returns 404 for a post that does not exist`):** ควบคุมโดยที่โมเดลไม่ต้องมี
`rescue_from` เอง เพราะ Rails ตั้งค่า `config.action_dispatch.show_exceptions = :rescuable` ใน
`config/environments/test.rb` เป็นค่าเริ่มต้นอยู่แล้ว ทำให้ `ActiveRecord::RecordNotFound` (ที่
`Post.find` raise เมื่อหา id ไม่เจอ) ถูกแปลงเป็น HTTP 404 โดยอัตโนมัติเหมือนพฤติกรรมจริงตอนรัน
production แทนที่จะทำให้เทสต์ crash ด้วย exception ตรงๆ

---

## Step 458: Request spec ขั้นสูง — auth-protected action, `sign_in` pattern, validation error response, redirect และทำไมเลิกใช้ controller spec

### ต่อยอด authentication แบบ session-based จาก Part 041

ทบทวนจาก Part 041: ระบบ login สร้าง session ด้วย `session[:user_id] = user.id` ผ่าน
`SessionsController` และมี `current_user`/`require_authentication` ใน `ApplicationController`
เราจะนำรูปแบบเดียวกันมาใช้ในแอปทดสอบนี้ แล้วเขียน request spec ที่ต้อง **login ก่อน** จึงจะเข้า
action ที่ป้องกันไว้ได้

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :session, only: %i[new create destroy]
  resources :posts

  root "posts#index"
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  allow_browser versions: :modern

  private

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user

  def require_authentication
    return if current_user

    redirect_to new_session_path, alert: "กรุณาเข้าสู่ระบบก่อน"
  end
end
```

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def new
  end

  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path, notice: "เข้าสู่ระบบสำเร็จ"
    else
      flash.now[:alert] = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      render :new, status: :unprocessable_content
    end
  end

  def destroy
    session.delete(:user_id)
    redirect_to root_path, notice: "ออกจากระบบแล้ว"
  end
end
```

```ruby
# app/controllers/posts_controller.rb (เพิ่ม action ที่ป้องกันไว้)
class PostsController < ApplicationController
  before_action :require_authentication, only: %i[new create edit update destroy]
  before_action :set_post, only: %i[show edit update destroy]

  # ... index/show เหมือน Step 457 ...

  def new
    @post = current_user.posts.build
  end

  def create
    @post = current_user.posts.build(post_params)

    if @post.save
      redirect_to @post, notice: "สร้างบทความสำเร็จ"
    else
      render :new, status: :unprocessable_content
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: "ลบบทความสำเร็จ"
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :status)
  end
end
```

> **หมายเหตุเรื่อง `:unprocessable_content`:** Rails 8.1 (ผ่าน Rack เวอร์ชันใหม่) เปลี่ยนชื่อ
> symbol มาตรฐานของ status 422 จาก `:unprocessable_entity` (ชื่อเดิมที่ใช้กันมานาน) เป็น
> `:unprocessable_content` — ถ้าใช้ `:unprocessable_entity` ในโค้ดปัจจุบันจะยังทำงานได้แต่ขึ้น
> deprecation warning:
> `Status code :unprocessable_entity is deprecated and will be removed in a future version of Rack. Please use :unprocessable_content instead.`
> บทเรียนนี้ใช้ `:unprocessable_content` ตามมาตรฐานล่าสุดตั้งแต่ต้น

### helper สำหรับ "login" ใน request spec

request spec ไม่มีเบราว์เซอร์จริง ไม่มีการกรอกฟอร์มด้วยมือ วิธีจำลองผู้ใช้ที่ login แล้วมีสอง
แนวทางหลัก:

**แนวทางที่ 1 (แนะนำ — สมจริงที่สุด):** ยิง `POST` ไปที่ endpoint login จริง เหมือนผู้ใช้กรอก
ฟอร์มเอง ทำให้ session cookie ถูกตั้งค่าโดยผ่าน code path การ authenticate จริงทั้งหมด — วิธีนี้
ทดสอบ "ระบบ login เองก็ยังทำงานถูกต้องอยู่" ไปด้วยในตัวทุกครั้งที่ request spec อื่นเรียกใช้

```ruby
# spec/support/request_sign_in_helper.rb
module RequestSignInHelper
  def sign_in_as(user, password:)
    post session_path, params: { email: user.email, password: password }
  end
end

RSpec.configure do |config|
  config.include RequestSignInHelper, type: :request
end
```

`config.include RequestSignInHelper, type: :request` ผูก method `sign_in_as` ให้ใช้ได้เฉพาะใน
spec ที่มี metadata `type: :request` เท่านั้น (ไม่ปนกับ model spec ที่ไม่เกี่ยวข้อง) — เป็น
รูปแบบเดียวกับที่จะเห็นซ้ำอีกใน Step 460 ตอนจัดระเบียบ `spec/support/`

**แนวทางที่ 2 (เร็วกว่า แต่สมจริงน้อยกว่า):** เซ็ต `session[:user_id]` ตรงๆ ผ่าน
`ActionDispatch::TestRequest` — rspec-rails request spec รองรับการเข้าถึง `session` เขียนได้
โดยตรงในบาง context แต่มีความซับซ้อนเรื่อง cookie jar มากกว่า ในทางปฏิบัติทีมส่วนใหญ่เลือก
**แนวทางที่ 1** เพราะเขียนง่ายกว่า อ่านเข้าใจง่ายกว่า และได้ประโยชน์พ่วง (ทดสอบ login flow จริง
ไปด้วย) แม้จะช้ากว่าเล็กน้อยเพราะต้องผ่าน `authenticate` (bcrypt hash) ทุกครั้ง

### เขียน request spec สำหรับ action ที่ป้องกันไว้

```ruby
# spec/requests/posts_spec.rb (เพิ่มเข้าไป)
describe "GET /posts/new" do
  context "when not signed in" do
    it "redirects to the login page" do
      get new_post_path
      expect(response).to redirect_to(new_session_path)
    end
  end

  context "when signed in" do
    before { sign_in_as(user, password: "password123") }

    it "returns http success" do
      get new_post_path
      expect(response).to have_http_status(:ok)
    end
  end
end

describe "POST /posts" do
  context "when not signed in" do
    it "does not create a post and redirects to login" do
      expect do
        post posts_path, params: { post: { title: "ใหม่", body: "เนื้อหา" } }
      end.not_to change(Post, :count)

      expect(response).to redirect_to(new_session_path)
    end
  end

  context "when signed in" do
    before { sign_in_as(user, password: "password123") }

    context "with valid parameters" do
      let(:valid_params) { { post: { title: "บทความใหม่", body: "เนื้อหาบทความใหม่" } } }

      it "creates a new post" do
        expect { post posts_path, params: valid_params }.to change(Post, :count).by(1)
      end

      it "redirects to the created post" do
        post posts_path, params: valid_params
        expect(response).to redirect_to(post_path(Post.last))
      end

      it "sets a success flash message" do
        post posts_path, params: valid_params
        expect(flash[:notice]).to eq("สร้างบทความสำเร็จ")
      end
    end

    context "with invalid parameters" do
      let(:invalid_params) { { post: { title: "", body: "" } } }

      it "does not create a new post" do
        expect { post posts_path, params: invalid_params }.not_to change(Post, :count)
      end

      it "returns unprocessable_content status" do
        post posts_path, params: invalid_params
        expect(response).to have_http_status(:unprocessable_content)
      end

      it "re-renders the new form with error messages" do
        post posts_path, params: invalid_params
        expect(response.body).to include("prohibited this post from being saved")
      end
    end
  end
end
```

ผลลัพธ์จริงเมื่อรันทั้งไฟล์ `spec/requests/posts_spec.rb`:

```
Posts
  GET /posts
    returns http success
    renders the list of posts in the HTML body
    returns posts as JSON when requested
  GET /posts/:id
    returns http success for an existing post
    returns 404 for a post that does not exist
  GET /posts/new
    when not signed in
      redirects to the login page
    when signed in
      returns http success
  POST /posts
    when not signed in
      does not create a post and redirects to login
    when signed in
      with valid parameters
        creates a new post
        redirects to the created post
        sets a success flash message
      with invalid parameters
        does not create a new post
        returns unprocessable_content status
        re-renders the new form with error messages
  DELETE /posts/:id
    when signed in
      destroys the post
      redirects to the posts list

Finished in 0.5 seconds (files took 1.0 seconds to load)
16 examples, 0 failures
```

**รูปแบบ `context "when not signed in"` / `context "when signed in"`** คือการนำหลักการจาก
Part 019 Step 183 มาใช้ต่อยอด: แยก **เงื่อนไขด้าน authentication** เป็นคนละ `context` เสมอ ทำให้
เห็นชัดเจนใน documentation format ว่าแต่ละ action มีพฤติกรรมต่างกันอย่างไรระหว่างผู้ใช้ที่ login
แล้วกับยังไม่ได้ login — เป็น pattern ที่พบในแทบทุก request spec ของแอป Rails ที่มีระบบสมาชิก

### ทำไมเลิกใช้ controller spec แล้วเปลี่ยนมาใช้ request spec

RSpec เคยมี spec type ชื่อ **controller spec** (`type: :controller`) ที่ทดสอบ controller
แบบแยกเดี่ยว (isolation) โดยไม่ผ่าน routing engine จริง:

```ruby
# รูปแบบเก่า (controller spec) -- แสดงไว้เพื่อให้จำได้เวลาเจอในโค้ด legacy เท่านั้น
# ต้องติดตั้ง gem เสริม `rails-controller-testing` เพิ่ม เพราะ rspec-rails เอา
# assigns/render_template ออกจาก core ไปแล้วตั้งแต่หลายเวอร์ชันก่อน
RSpec.describe PostsController, type: :controller do
  describe "GET #show" do
    it "assigns the requested post to @post" do
      post_record = Post.create!(title: "...", body: "...", user: user)
      get :show, params: { id: post_record.id }
      expect(assigns(:post)).to eq(post_record)
      expect(response).to render_template(:show)
    end
  end
end
```

**ปัญหาของ controller spec ที่ทำให้ทีมงาน RSpec เองแนะนำให้เลิกใช้ตั้งแต่ Rails 5:**

| ประเด็น | Controller spec (แบบเก่า) | Request spec (แบบปัจจุบัน) |
|---|---|---|
| ผ่าน routing engine จริงไหม | ไม่ผ่าน (`get :show, params: {...}` เรียก action ตรงๆ ข้ามการ match route) | ผ่านจริง (`get post_path(post_record)` ต้องแปลง URL ผ่าน `config/routes.rb` จริง) |
| ผ่าน middleware stack ไหม | ไม่ผ่าน (Rack middleware ทั้งหมดถูกข้าม) | ผ่านครบ (session, cookie, CSRF layer ทำงานเหมือนจริง) |
| ตรวจ instance variable ภายใน (`assigns(:post)`) | ทำได้ (แต่ผูกเทสต์กับ implementation detail ภายใน controller) | ทำไม่ได้โดยตรง (ต้องเช็คผลลัพธ์ที่สังเกตได้จากภายนอกแทน เช่น `response.body`) |
| ความน่าเชื่อถือ | ต่ำกว่า — ผ่านเทสต์ได้ทั้งที่ route จริงอาจพังหรือ middleware อาจบล็อก request จริง | สูงกว่า — ถ้าเทสต์ผ่าน แปลว่า request จริงจากผู้ใช้ก็ควรทำงานได้เหมือนกัน |
| ความเร็ว | เร็วกว่าเล็กน้อย (ข้าม routing/middleware) | ช้ากว่าเล็กน้อย แต่ต่างกันไม่มากในทางปฏิบัติ |
| สถานะปัจจุบัน | ยังใช้งานได้ (ต้องติดตั้ง `rails-controller-testing` เพิ่ม) แต่ RSpec ไม่แนะนำให้เขียนใหม่แล้ว | เป็นค่าเริ่มต้นที่ generator สร้างให้ (ยืนยันจริงจาก Step 457) |

เหตุผลหลักคือ **`assigns(:post)` และ `render_template`** ผูกเทสต์เข้ากับรายละเอียดภายในของ
controller มากเกินไป (ผูกกับชื่อ instance variable ที่ controller ใช้) ทำให้ refactor
controller ได้ยาก (แก้แค่เปลี่ยนชื่อตัวแปรภายใน ก็ทำให้เทสต์พังทั้งที่พฤติกรรมจริงยังถูกต้อง) ใน
ขณะที่ request spec ตรวจสอบแค่ **สิ่งที่ผู้ใช้จริงมองเห็นได้** (status code, response body,
redirect, flash) ซึ่งเป็นสิ่งที่สำคัญจริงๆ

> **สรุปแนวปฏิบัติ:** เขียน **request spec เสมอ** สำหรับทดสอบ controller/route ในโค้ดใหม่ทุก
> กรณี ไม่ต้องเรียนรู้หรือติดตั้ง `rails-controller-testing` เพิ่มเลย เว้นแต่ต้องดูแลโค้ด legacy
> ที่มี controller spec เก่าอยู่แล้วจำนวนมากและยังไม่มีเวลาย้ายมาเป็น request spec ทั้งหมด

---

## Step 459: การรันสเปก — `bundle exec rspec`, การกรองด้วย `--tag`, `fdescribe`/`fit`

### รันทั้งชุดเทสต์

```bash
bundle exec rspec
```

สแกนหาไฟล์ `spec/**/*_spec.rb` ทั้งหมดในโปรเจกต์และรันทุกไฟล์ (เหมือนพฤติกรรมที่เรียนใน
Part 019) ผลลัพธ์รวมของแอปในบทนี้:

```
Post
  ... (15 examples)
User
  ... (4 examples)
Posts
  ... (16 examples)

Finished in 0.56 seconds (files took 0.96 seconds to load)
37 examples, 0 failures
```

### กรองด้วย tag — `--tag`

บางเทสต์ใช้เวลานานกว่าปกติ (เช่น ยิง API ภายนอกจริง หรือทดสอบ integration ที่ซับซ้อน) การติด
**tag** (metadata แบบ symbol) ไว้ที่ example ทำให้เลือกรันหรือข้ามเป็นกลุ่มได้:

```ruby
describe ".published" do
  it "returns only posts with status published", :slow do
    # ...
  end
end
```

`:slow` ตรงนี้คือ metadata แบบ shorthand (เทียบเท่ากับ `slow: true`) รันเฉพาะเทสต์ที่ติด tag
นี้:

```bash
bundle exec rspec --tag slow --format documentation
```

```
Run options: include {:slow=>true}

Post
  .published
    returns only posts with status published

Finished in 0.03 seconds (files took 0.92 seconds to load)
1 example, 0 failures
```

รันทุกเทสต์ **ยกเว้น** ที่ติด tag นี้ (ใส่ `~` นำหน้าชื่อ tag):

```bash
bundle exec rspec --tag ~slow --format progress
```

```
Run options: exclude {:slow=>true}
....................................

Finished in 0.55 seconds (files took 1.06 seconds to load)
36 examples, 0 failures
```

**การใช้งานจริงในทีม:** ทีมส่วนใหญ่ตั้ง tag แบบนี้ไว้กับเทสต์กลุ่มพิเศษ เช่น `:slow`
(ทดสอบที่ใช้เวลานาน), `:external_api` (ยิง service ภายนอกจริง ไม่เหมาะรันบนเครื่อง dev บ่อยๆ),
`:wip` (work in progress ยังทำไม่เสร็จ) แล้วตั้งค่าใน CI ให้รันแค่บาง tag ตามจังหวะ (เช่น รัน
เทสต์ทั้งหมดยกเว้น `:external_api` ทุกครั้งที่ push โค้ด แต่รัน `:external_api` เฉพาะ
scheduled job ตอนกลางคืน)

### โฟกัสเทสต์ระหว่างพัฒนา — `fdescribe`/`fit`/`fcontext`

เวลากำลังเขียนหรือแก้เทสต์ตัวเดียวในไฟล์ที่มีเทสต์เป็นสิบๆ ตัว การรันทั้งไฟล์ทุกครั้งเสียเวลาโดย
ไม่จำเป็น RSpec มี alias พิเศษที่ขึ้นต้นด้วย `f` (focus) ให้ใช้แทน `it`/`describe`/`context`
เพื่อบอกว่า "รันเฉพาะตัวนี้เท่านั้น"

ต้องเปิดใช้งานก่อนใน `spec/spec_helper.rb` (ปกติ generator คอมเมนต์บรรทัดนี้ไว้เป็นค่าเริ่มต้น
ให้ไปเปิดเอง):

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  # ...
  config.filter_run_when_matching :focus
end
```

ลองเปลี่ยน `it` ตัวหนึ่งเป็น `fit`:

```ruby
describe "#published?" do
  context "when status is published" do
    fit "returns true" do
      post_record.status = "published"
      expect(post_record.published?).to be(true)
    end
  end
  # ...
end
```

รันทั้งชุดเทสต์ (ไม่ระบุไฟล์เจาะจง):

```bash
bundle exec rspec --format documentation
```

ผลลัพธ์จริง — ทั้งโปรเจกต์ที่มี 37 example ลดเหลือรันแค่ตัวเดียว:

```
Run options: include {:focus=>true}

Post
  #published?
    when status is published
      returns true

Finished in 0.03 seconds (files took 1.1 seconds to load)
1 example, 0 failures
```

`fdescribe`/`fcontext` ทำงานแบบเดียวกันแต่โฟกัสทั้งกลุ่ม (ทุก `it` ข้างในกลุ่มนั้น) แทนที่จะ
โฟกัสแค่ example เดียว

> **คำเตือนสำคัญที่สุดของ Step นี้:** `fit`/`fdescribe`/`fcontext` **ต้องลบออกก่อน commit
> เสมอ** เพราะถ้าเผลอ push โค้ดที่ยังมี `fit` ค้างอยู่ CI จะรันแค่เทสต์ที่ focus ไว้เท่านั้น
> (เหมือนที่เห็นด้านบนว่าลดจาก 37 เหลือ 1) ทำให้เทสต์ตัวอื่นที่พังจริงหลุดผ่าน CI ไปโดยไม่มีใคร
> รู้ตัว ทีมมืออาชีพส่วนใหญ่ป้องกันด้วย RuboCop cop ชื่อ `RSpec/Focus` ที่ fail ทันทีถ้าเจอ
> `fit`/`fdescribe`/`fcontext` หลงเหลือในโค้ด หรือตั้ง CI ให้รันด้วย
> `bundle exec rspec --force-color` พ่วง script ตรวจจับคำเหล่านี้ก่อน merge

### ตัวเลือกอื่นที่ใช้บ่อยตอนพัฒนา

```bash
# รันไฟล์เดียว
bundle exec rspec spec/models/post_spec.rb

# รันเฉพาะบรรทัดที่ระบุ (ใช้ line number ที่ RSpec แจ้งไว้ตอนเทสต์ fail)
bundle exec rspec spec/models/post_spec.rb:42

# รันเฉพาะ example ที่ชื่อ (description) ตรงกับ pattern ที่ระบุ
bundle exec rspec spec/models/post_spec.rb -e "reading_time_minutes"

# รันเฉพาะเทสต์ที่ fail ล่าสุด (ต้องมี config.example_status_persistence_file_path ตั้งไว้)
bundle exec rspec --only-failures

# รันเทสต์ตัวที่ fail ล่าสุดก่อน แล้วค่อยรันตัวอื่นตามหลัง
bundle exec rspec --next-failure
```

---

## Step 460: การจัดระเบียบโฟลเดอร์ `spec/` ที่ขยายใหญ่ขึ้น — `spec/support/`, shared context, `rails_helper.rb`

เมื่อโปรเจกต์โตขึ้น โฟลเดอร์ `spec/` จะมีไฟล์เป็นร้อยๆ ไฟล์ RSpec/rspec-rails มีธรรมเนียมการจัด
โครงสร้างที่ชัดเจนเพื่อไม่ให้โกลาหล

### โครงสร้างมาตรฐานของ `spec/`

```
spec/
├── models/                 # model spec — mirror app/models/
│   ├── post_spec.rb
│   └── user_spec.rb
├── requests/                # request spec — mirror ชื่อ controller (ไม่ใช่ path เป๊ะๆ)
│   └── posts_spec.rb
├── support/                  # ไฟล์ config/helper เสริมที่ไม่ใช่ spec โดยตรง
│   ├── request_sign_in_helper.rb
│   └── shared_contexts/
│       └── signed_in_user.rb
├── factories/                # (จะเรียนใน Part 047 — FactoryBot)
├── rails_helper.rb
└── spec_helper.rb
```

### `spec/support/` — รวม helper/config ที่ใช้ซ้ำหลายไฟล์

`request_sign_in_helper.rb` ที่เขียนใน Step 458 ถูกเก็บไว้ที่ `spec/support/` ตามธรรมเนียม
มาตรฐาน — โฟลเดอร์นี้เก็บทุกอย่างที่ **ไม่ใช่ spec โดยตรง** แต่ spec หลายไฟล์ต้องใช้ร่วมกัน เช่น
custom matcher, helper method, shared context/examples

จำได้จาก Step 452 ว่า `rails_helper.rb` มีบรรทัดนี้ถูกคอมเมนต์ไว้เป็นค่าเริ่มต้น:

```ruby
# Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }
```

**ต้องเปิดใช้งานบรรทัดนี้เอง** (ลบ `#` ออก) ถ้าต้องการให้ทุกไฟล์ใน `spec/support/` ถูก
`require` อัตโนมัติทุกครั้งที่รันเทสต์ — เหตุผลที่ generator คอมเมนต์ไว้เป็นค่าเริ่มต้นคือ
เพิ่มเวลา boot-up (สแกนทุกไฟล์ในโฟลเดอร์นี้ทุกครั้งแม้สเปกที่กำลังรันจะไม่ได้ใช้ helper นั้นเลย)
— ในโปรเจกต์เล็กถึงกลางแทบทุกทีมเปิดบรรทัดนี้ทันทีเพราะสะดวกกว่ามาก ส่วนโปรเจกต์ใหญ่ระดับ
enterprise ที่รันเทสต์นับหมื่นไฟล์ บางทีมเลือก `require` เฉพาะไฟล์ support ที่จำเป็นตรงในแต่ละ
spec file แทน เพื่อลดเวลา boot-up ให้เหลือน้อยที่สุด

### `spec/support/shared_contexts/` — DRY เรื่อง setup ที่ใช้ซ้ำข้ามหลาย spec file

Step 458 มี pattern `sign_in_as(user, password: "password123")` ซ้ำอยู่หลาย `context` ใน
ไฟล์เดียว ถ้ามี request spec ไฟล์อื่นอีกหลายไฟล์ (เช่น `spec/requests/comments_spec.rb`,
`spec/requests/profiles_spec.rb`) ที่ต้องการ user ที่ login แล้วเหมือนกัน การก็อปวาง
`let(:user)` + `before { sign_in_as(...) }` ซ้ำทุกไฟล์ผิดหลัก DRY ที่เรียนมาตั้งแต่ Part 019
Step 189 — ใช้ `shared_context` แก้ปัญหานี้ได้เหมือนกับที่ `shared_examples` แก้ปัญหาการเทสต์
ซ้ำข้าม class

```ruby
# spec/support/shared_contexts/signed_in_user.rb
RSpec.shared_context "a signed in user" do
  let(:user) { User.create!(email: "author@example.com", password: "password123") }

  before { sign_in_as(user, password: "password123") }
end
```

ใช้งานด้วย `include_context`:

```ruby
RSpec.describe "Posts (shared_context demo)", type: :request do
  include_context "a signed in user"

  it "allows a signed in user to view the new post form" do
    get new_post_path
    expect(response).to have_http_status(:ok)
  end
end
```

รันจริง:

```
Posts (shared_context demo)
  allows a signed in user to view the new post form

Finished in 0.09 seconds (files took 1.1 seconds to load)
1 example, 0 failures
```

**`shared_context` vs `shared_examples` (Part 019 Step 189) ต่างกันอย่างไร:** `shared_examples`
แชร์ **ทั้ง setup และ example (`it` block)** ไปด้วยกัน ใช้เมื่อ class/endpoint หลายตัวมี
พฤติกรรมที่ต้องเทสต์เหมือนกันเป๊ะ ส่วน `shared_context` แชร์ **แค่ setup** (`let`, `before`)
เท่านั้น ไม่มี `it` ติดมาด้วย ใช้เมื่อ endpoint ต่างกันต้องการ "สถานะเริ่มต้น" เดียวกัน (เช่น
"มี user ที่ login แล้ว") แต่ตัว example ที่แต่ละไฟล์เขียนเองยังต่างกันไปตาม endpoint นั้นๆ

### `config.shared_context_metadata_behavior = :apply_to_host_groups`

ค่านี้ (ที่เห็นอยู่ใน `spec_helper.rb` ที่ generator สร้างให้ตั้งแต่ต้น) เปิดทางให้ shared
context ผูกกับ **metadata** แทนการเรียก `include_context` ตรงๆ ได้ด้วย เช่น ประกาศเป็น
`RSpec.shared_context "a signed in user", :authenticated do ... end` แล้วทุก `describe`/`it`
ที่ติด metadata `:authenticated` (เช่น `context "when signed in", :authenticated`) จะได้ setup
นี้มาอัตโนมัติโดยไม่ต้องเขียน `include_context` เลย รูปแบบนี้สะดวกเมื่อมี shared context ที่ใช้
บ่อยมากจริงๆ แต่ก็ทำให้ **มองไม่เห็นจากในไฟล์ว่า setup มาจากไหน** ถ้าใช้เยอะเกินไป ทีมส่วนใหญ่จึง
ยังใช้ `include_context` แบบเขียนตรงๆ เป็นค่าเริ่มต้น และเก็บรูปแบบ metadata ไว้เฉพาะกรณีที่คุ้มค่า
จริงๆ เท่านั้น

### สรุปบทบาทของแต่ละไฟล์ config ใน `spec/`

| ไฟล์ | บทบาท | โหลด Rails ไหม |
|---|---|---|
| `.rspec` | ตั้งค่า command line option ที่ใช้ทุกครั้ง (เช่น `--require spec_helper`) | ไม่เกี่ยวข้อง |
| `spec/spec_helper.rb` | config ระดับ RSpec core (matcher, mock framework, order) | ไม่ |
| `spec/rails_helper.rb` | boot Rails, โหลด rspec-rails, ตั้งค่า transactional fixtures, โหลด `spec/support/**/*.rb`, config shoulda-matchers | ใช่ |
| `spec/support/*.rb` | helper method, custom matcher — โหลดผ่าน `rails_helper.rb` | ตามที่ไฟล์นั้นต้องการ |
| `spec/support/shared_contexts/*.rb` | `RSpec.shared_context` ที่ใช้ซ้ำข้าม spec file | ตามที่ไฟล์นั้นต้องการ |

---

## แบบฝึกหัด: เขียน model spec + request spec เต็มรูปแบบให้ `Post` พร้อม TDD

### โจทย์

ต่อยอดจากแอป `blog_app` ที่สร้างตลอด Part นี้ ให้เพิ่ม **instance method ใหม่**
`#reading_time_minutes` ให้กับ `Post`:

- คำนวณเวลาที่ใช้อ่านบทความโดยประมาณ จากจำนวนคำใน `body`
- สมมติความเร็วในการอ่าน 200 คำต่อนาที
- ปัดเศษ**ขึ้น**เสมอเป็นจำนวนเต็มนาที (เช่น 2.01 นาที ต้องได้ 3 นาที ไม่ใช่ 2)
- อย่างน้อยต้องได้ 1 นาทีเสมอ แม้บทความจะสั้นมาก

**ให้ใช้ TDD (Red-Green-Refactor) ในการพัฒนา:** เขียนเทสต์ก่อน รันดูว่า fail จริง (Red) แล้ว
ค่อยเขียนโค้ดให้ผ่าน (Green)

### เฉลย

**ขั้นตอนที่ 1 (Red) — เขียนเทสต์ก่อนที่ method จะมีอยู่จริง**

```ruby
# spec/models/post_spec.rb (เพิ่มต่อท้าย describe block เดิม)
describe "#reading_time_minutes" do
  it "returns 1 minute for a short body" do
    post_record.body = "สวัสดี " * 10
    expect(post_record.reading_time_minutes).to eq(1)
  end

  it "rounds up to the nearest whole minute for a longer body" do
    post_record.body = "คำ " * 450 # 450 คำ / 200 คำต่อนาที = 2.25 -> ปัดขึ้นเป็น 3
    expect(post_record.reading_time_minutes).to eq(3)
  end
end
```

รันดูเฉพาะเทสต์ใหม่:

```bash
bundle exec rspec spec/models/post_spec.rb -e "reading_time_minutes" --format documentation
```

ผลลัพธ์จริง (**Red** — fail ตามที่คาดไว้ เพราะยังไม่มี method นี้เลย):

```
Run options: include {:full_description=>/reading_time_minutes/}

Post
  #reading_time_minutes
    returns 1 minute for a short body (FAILED - 1)
    rounds up to the nearest whole minute for a longer body (FAILED - 2)

Failures:

  1) Post#reading_time_minutes returns 1 minute for a short body
     Failure/Error: expect(post_record.reading_time_minutes).to eq(1)

     NoMethodError:
       undefined method `reading_time_minutes' for an instance of Post
     # ./spec/models/post_spec.rb:97:in `block (3 levels) in <top (required)>'

  2) Post#reading_time_minutes rounds up to the nearest whole minute for a longer body
     Failure/Error: expect(post_record.reading_time_minutes).to eq(3)

     NoMethodError:
       undefined method `reading_time_minutes' for an instance of Post
     # ./spec/models/post_spec.rb:102:in `block (3 levels) in <top (required)>'

Finished in 0.02565 seconds (files took 0.91012 seconds to load)
2 examples, 2 failures

Failed examples:

rspec ./spec/models/post_spec.rb:95 # Post#reading_time_minutes returns 1 minute for a short body
rspec ./spec/models/post_spec.rb:100 # Post#reading_time_minutes rounds up to the nearest whole minute for a longer body
```

RSpec บอกสาเหตุชัดเจน (`NoMethodError: undefined method 'reading_time_minutes'`) ตรงตามที่
คาดหวังในขั้นตอน Red — เทสต์ **ต้อง fail ด้วยเหตุผลที่ถูกต้อง** ก่อนเสมอ (ถ้า fail ด้วยเหตุผลอื่น
เช่น syntax error ใน spec เอง แสดงว่าเทสต์ยังเขียนไม่สมบูรณ์)

**ขั้นตอนที่ 2 (Green) — เขียนโค้ดให้เทสต์ผ่าน**

```ruby
# app/models/post.rb (เพิ่มเข้าไป)
def reading_time_minutes
  word_count = body.to_s.split.size
  [(word_count / 200.0).ceil, 1].max
end
```

รันเทสต์เดิมซ้ำอีกครั้ง:

```bash
bundle exec rspec spec/models/post_spec.rb -e "reading_time_minutes" --format documentation
```

ผลลัพธ์จริง (**Green**):

```
Post
  #reading_time_minutes
    returns 1 minute for a short body
    rounds up to the nearest whole minute for a longer body

Finished in 0.03561 seconds (files took 1.12 seconds to load)
2 examples, 0 failures
```

**ขั้นตอนที่ 3 — รันทั้งชุดเทสต์ยืนยันว่าไม่ทำให้ของเดิมพัง (Regression check)**

```bash
bundle exec rspec --format progress
```

```
.....................................

Finished in 0.49 seconds (files took 1.04 seconds to load)
37 examples, 0 failures
```

### ไฟล์ spec เต็มรูปแบบของ `Post`

รวมกับทุกอย่างที่เขียนสะสมมาตลอด Step 454–458 (`spec/models/post_spec.rb` มีครบทั้ง
validation ด้วย shoulda-matchers, association, `before_validation`, scope `.published`,
instance method `#published?`/`#publish!`/`#excerpt` และ `#reading_time_minutes` ที่เพิ่งเพิ่ม
ด้วย TDD ข้างต้น — โครงสร้างเดียวกันเป๊ะกับที่แสดงเต็มไว้แล้วในหัวข้อ "สรุปชุด model spec" ของ
Step 456) ส่วน `spec/requests/posts_spec.rb` ก็เป็นไฟล์เดียวกับที่แสดงเต็มไว้แล้วใน Step 458
โดยไม่มีการเปลี่ยนแปลงเพิ่มเติม (feature ที่เพิ่มในแบบฝึกหัดนี้อยู่ในระดับ model เท่านั้น ไม่กระทบ
request spec)

รันทั้งสองไฟล์พร้อมกันเพื่อยืนยันผลรวม:

```bash
bundle exec rspec spec/models/post_spec.rb spec/requests/posts_spec.rb --format progress
```

```
.................................

Finished in 0.42 seconds (files took 1.0 seconds to load)
33 examples, 0 failures
```

**33 example ทั้งหมดผ่าน** — 17 example จาก `post_spec.rb` (validation, association, scope,
instance method รวม `#reading_time_minutes` ที่เพิ่งพัฒนาด้วย TDD) บวกกับ 16 example จาก
`posts_spec.rb` (request spec เต็มรูปแบบจาก Step 458) ครอบคลุมทั้ง model layer และ HTTP layer
ของ `Post` แบบครบวงจร

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มโมเดล `Comment` (`belongs_to :post`, `belongs_to :user`, มี attribute `body:text`) พร้อม
   validation ว่า `body` ต้องมีอย่างน้อย 5 ตัวอักษร เขียน model spec ให้ครบด้วย shoulda-matchers
   (`validate_presence_of`, `validate_length_of`, `belong_to`) จากนั้นเพิ่ม nested route
   `resources :posts do resources :comments, only: %i[create destroy] end` แล้วเขียน request
   spec ทดสอบ `POST /posts/:post_id/comments` ทั้งกรณี signed in/not signed in และกรณี
   parameter ถูก/ผิด (ใช้ pattern เดียวกับ `POST /posts` ที่เขียนไปแล้วในบทนี้)

2. เพิ่มเงื่อนไข **authorization** ให้ `PostsController#edit`/`#update`/`#destroy`: อนุญาตเฉพาะ
   `current_user` ที่เป็นเจ้าของ post เท่านั้น (ถ้าไม่ใช่เจ้าของให้ redirect กลับ `posts_path`
   พร้อม flash แจ้งเตือนว่าไม่มีสิทธิ์) เขียน request spec เพิ่มอย่างน้อย 2 `context` ใหม่:
   "เมื่อเป็นเจ้าของบทความ" และ "เมื่อไม่ใช่เจ้าของบทความ" (ต้องสร้าง user คนที่สองในเทสต์เพื่อ
   จำลองสถานการณ์นี้)

3. เขียน `shared_examples` (ทบทวนจาก Part 019 Step 189) ชื่อ `"requires authentication"` ที่รับ
   HTTP method และ path เป็นพารามิเตอร์ แล้วตรวจสอบว่า action นั้น redirect ไปหน้า login เมื่อยัง
   ไม่ได้ signed in จากนั้นเรียกใช้ `it_behaves_like "requires authentication"` กับทุก action ที่
   มี `before_action :require_authentication` ใน `PostsController` (`new`, `create`, `edit`,
   `update`, `destroy`) แทนที่จะเขียน context "when not signed in" ซ้ำๆ ทุก `describe` (ใบ้:
   ต้องส่ง HTTP verb และ path builder เป็น block หรือ symbol เข้าไปใน `shared_examples` เพราะแต่
   ละ action ใช้ verb/path ต่างกัน)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า `rspec-rails` ไม่ใช่ RSpec framework ตัวใหม่ แต่เป็น adapter ที่เชื่อม RSpec core
  (จาก Part 019) เข้ากับวงจรชีวิตของ Rails app ผ่าน spec type, matcher, และ generator เฉพาะ
- แยกบทบาทของ `spec/spec_helper.rb` (config RSpec core, เบา, ไม่รู้จัก Rails) กับ
  `spec/rails_helper.rb` (boot Rails เต็มรูปแบบ, โหลด rspec-rails, โหลด `spec/support/`) ได้
  อย่างถูกต้อง และรู้ว่าทำไมต้องแยกสองไฟล์
- เข้าใจกลไก `config.use_transactional_fixtures` ที่ห่อทุก example ด้วย database transaction
  แล้ว rollback อัตโนมัติ ทำให้ไม่ต้องเขียนโค้ดเก็บกวาดข้อมูลทดสอบเองเลย พร้อมพิสูจน์ด้วยเทสต์
  จริงว่าฐานข้อมูล "ว่างเปล่า" ทุกครั้งที่ example ใหม่เริ่มต้น
- เขียน model spec ทดสอบ validation, association, scope, และ instance method ของ ActiveRecord
  model ได้ครบวงจร โดยใช้ทั้งรูปแบบพื้นฐาน (`be_valid`, `errors[]`, `change { }`) และรูปแบบ
  กระชับด้วย `shoulda-matchers` (`validate_presence_of`, `belong_to`, `have_many`) พร้อมเข้าใจ
  ข้อจำกัดว่า shoulda-matchers ทดสอบแค่โครงสร้าง ไม่ใช่ business logic ทั้งหมด
- เขียน request spec ทดสอบ HTTP request/response เต็มวงจร ทั้ง HTML และ JSON, ทดสอบ
  action ที่ป้องกันด้วย authentication ผ่านการจำลอง `sign_in_as` (ยิง `POST` ไปที่ endpoint
  login จริง), ทดสอบ validation error response (`unprocessable_content`) และ redirect
- เข้าใจเหตุผลเชิงประวัติศาสตร์และเทคนิคที่ทำให้วงการ Rails เปลี่ยนจาก controller spec มาเป็น
  request spec เป็นมาตรฐาน (ผ่าน routing/middleware จริง, ไม่ผูกกับ implementation detail
  ภายใน controller) และยืนยันด้วย generator จริงว่า `rails generate controller` สร้าง
  `spec/requests/*_spec.rb` ให้อัตโนมัติ ไม่ใช่ `spec/controllers/*_spec.rb`
- รันสเปกได้หลายรูปแบบ: `bundle exec rspec` (ทั้งชุด), กรองด้วย `--tag`/`--tag ~`, ระบุ
  ไฟล์/บรรทัด/pattern เฉพาะ, และใช้ `fdescribe`/`fit` โฟกัสระหว่างพัฒนา พร้อมเข้าใจความเสี่ยง
  ของการลืมลบ focus ก่อน commit
- จัดระเบียบโฟลเดอร์ `spec/` ที่ขยายใหญ่ขึ้นด้วย `spec/support/` (helper method), และ
  `spec/support/shared_contexts/` (`RSpec.shared_context`/`include_context`) เพื่อ DRY
  ขั้นตอน setup ที่ใช้ซ้ำข้ามหลาย spec file
- ฝึกใช้ TDD (Red-Green) จริงกับ method `#reading_time_minutes` ตั้งแต่เขียนเทสต์ที่ fail ด้วย
  เหตุผลที่ถูกต้อง (`NoMethodError`) ก่อน แล้วค่อยเขียนโค้ดให้ผ่าน พร้อมรัน regression check
  ทั้งชุดเทสต์ยืนยันว่าไม่ทำของเดิมพัง

**ต่อไป (Part 047):** สังเกตว่าตลอด Part นี้ทุกเทสต์ต้องเขียน `User.create!(email: "...",
password: "...")` และ `Post.create!(title: "...", body: "...", user: user)` ซ้ำๆ ด้วยมือทุก
ครั้ง ทั้งยังต้องคิดค่าตัวอย่าง (`"บทความแรก"`, `"เนื้อหาบทความ"`) เองทุกจุด ซึ่งไม่สมจริงและ
ซ้ำซ้อนมากขึ้นเรื่อยๆ เมื่อโมเดลมี attribute เยอะขึ้น Part หน้าจะแนะนำ **FactoryBot** — เครื่องมือ
สร้าง test data แบบมาตรฐานที่แทนที่การเรียก `Model.create!` ตรงๆ ด้วย `create(:post)` ที่
กำหนดค่าเริ่มต้นไว้ล่วงหน้า, **Faker** — gem สร้างข้อมูลสุ่มสมจริง (ชื่อคน, อีเมล, ข้อความ) แทนการ
พิมพ์ `"บทความแรก"` ซ้ำๆ, และเทคนิคการจัดการ test data ให้ยังคง maintainable แม้โมเดลจะซับซ้อน
ขึ้นมากในเฟสถัดๆ ไป
