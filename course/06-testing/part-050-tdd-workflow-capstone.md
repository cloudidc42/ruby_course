# Part 050: TDD Workflow เต็มรูปแบบ — สร้างฟีเจอร์ Comment Moderation ด้วย Red-Green-Refactor (ปิด Phase 6)

> **Step ครอบคลุมใน Part นี้:** Step 491–500
> **ระดับ:** สูง (ต้องผ่าน Part 046 RSpec สำหรับ Rails, Part 047 FactoryBot/Faker, Part 048
> Capybara System Test, Part 049 Mocking/VCR/SimpleCov มาก่อน รวมถึง Part 041
> `has_secure_password` และ Part 043 Pundit)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6, Rails 8.1.4, `rspec-rails` 8.0.4, `factory_bot_rails` 6.5.1,
> `faker` 3.8.0, `capybara` 3.40.0, `pundit` 2.5.2, `simplecov` 0.22.0 — **ทุกบรรทัดโค้ดและทุก
> ผลลัพธ์ terminal ใน Part นี้รันจริงในโปรเจกต์ Rails ทดสอบจริง ไม่มีการจำลองหรือแต่งขึ้น**

Part 046–049 สอนเครื่องมือทดสอบของ Rails ทีละชิ้นแยกกัน: RSpec สำหรับเขียน model/request spec,
FactoryBot/Faker สำหรับสร้างข้อมูลทดสอบ, Capybara สำหรับจำลองพฤติกรรมผู้ใช้ในเบราว์เซอร์, และ
mocking/VCR/SimpleCov สำหรับจัดการ dependency ภายนอกและวัดความครอบคลุมของเทสต์ **Part นี้ไม่สอน
เครื่องมือใหม่แม้แต่ตัวเดียว** — งานเดียวของ Part นี้คือเอาทุกเครื่องมือที่มีอยู่แล้วมาประกอบกันเป็น
**วินัยการทำงานเดียว (workflow)** ที่มืออาชีพใช้จริงทุกวัน นั่นคือ **Test-Driven Development
(TDD)** ผ่านวงจร **Red → Green → Refactor**

โจทย์ที่ใช้สาธิตคือฟีเจอร์ที่พบได้ในระบบบล็อกแทบทุกระบบ: **Comment Moderation** — ผู้อ่านแสดง
ความเห็นใต้บทความได้ แต่ความเห็นจะไม่แสดงต่อสาธารณะทันที ต้องรอให้ **เจ้าของบทความ** อนุมัติก่อน
เจ้าของบทความอนุมัติหรือปฏิเสธความเห็นได้ ระบบนี้มีทุกอย่างที่ทำให้เป็นตัวอย่าง TDD ที่ดี:
ความสัมพันธ์ระหว่างโมเดล, state machine ที่มี logic การเปลี่ยนสถานะ, การจำกัดสิทธิ์
(authorization), และ flow การใช้งานที่ข้าม request หลายจุด — เราจะสร้างมันขึ้นมาทีละชิ้น **โดยไม่
เขียนโค้ด production แม้แต่บรรทัดเดียวก่อนที่จะมีเทสต์สีแดงเรียกร้องให้เขียนมันขึ้นมาก่อน**

> **หมายเหตุเรื่องจุดเริ่มต้น:** สมมติว่าเรากำลังทำงานต่อในแอปบล็อกเดิมจาก Part 030 (Phase 3)
> ที่มี `User` (พร้อม `has_secure_password` จาก Part 041) และ `Post` อยู่แล้ว, ติดตั้ง Pundit
> ไว้แล้วจาก Part 043, และมีชุดทดสอบ RSpec/FactoryBot/Capybara/SimpleCov ที่ตั้งค่าไว้ตาม
> Part 046–049 ครบถ้วนแล้ว — โครงสร้างพื้นฐานเหล่านี้ **ไม่ใช่สิ่งที่ TDD ใน Part นี้** เพราะมันมี
> อยู่ก่อนแล้ว งานของเราคือฟีเจอร์ **ใหม่** หนึ่งฟีเจอร์ที่จะเพิ่มเข้าไปในระบบที่มีอยู่ — นี่คือ
> สถานการณ์ที่สมจริงที่สุดของ TDD ในงานจริง (ไม่มีใครเขียนแอปทั้งระบบใหม่ทุกครั้งที่เพิ่มฟีเจอร์)

## สารบัญของ Part นี้

- Step 491: Red-Green-Refactor คืออะไร ทำไม "เขียนเทสต์ก่อน" ถึงเปลี่ยนการออกแบบโค้ดไปตลอดกาล
  — เตรียมเครื่องมือ + Kata อุ่นเครื่องแบบครบวงจรจริง
- Step 492: TDD Cycle 1 (RED) — เขียน Model Spec ของ `Comment` ก่อนที่ `Comment` จะมีตัวตนจริง
- Step 493: TDD Cycle 1 (GREEN) — สร้าง Migration และ `Comment` Model ให้เทสต์ผ่านด้วยโค้ด
  น้อยที่สุด
- Step 494: TDD Cycle 2 — สร้างความเห็นผ่าน Nested Route ด้วย Request Spec (RED → GREEN →
  REFACTOR)
- Step 495: TDD Cycle 3 — Moderation State Machine ด้วย `enum` (pending/approved/rejected)
- Step 496: TDD Cycle 4 — จำกัดสิทธิ์อนุมัติ/ปฏิเสธด้วย Pundit (เฉพาะเจ้าของบทความเท่านั้น)
- Step 497: TDD Cycle 5 — System Test เต็มรูปแบบด้วย Capybara: ผู้เขียนเห็น เข้าไปอนุมัติ
  ความเห็น
- Step 498: Refactor Pass — สกัด `CommentModerationService` ออกมา โดยชุดทดสอบทั้งหมดยังผ่าน
  เหมือนเดิมทุกตัว (พิสูจน์ safety net ของการรีแฟกเตอร์)
- Step 499: เติม FactoryBot Factories ย้อนหลัง เพื่อล้าง setup ของทุก spec ที่เขียนมาก่อนหน้านี้
- Step 500: รัน Full Suite พร้อม SimpleCov Coverage Report เต็มรูปแบบ + สรุปบทเรียนของ TDD +
  แบบฝึกหัดต่อยอด + ปิด Phase 6

---

## Step 491: Red-Green-Refactor คืออะไร ทำไม "เขียนเทสต์ก่อน" ถึงเปลี่ยนการออกแบบโค้ดไปตลอดกาล

### วงจร Red-Green-Refactor

**Test-Driven Development (TDD)** มีวินัยที่เรียบง่ายมาก แค่ 3 ขั้นตอน วนซ้ำไปเรื่อยๆ:

1. **🔴 RED** — เขียนเทสต์สำหรับพฤติกรรมที่ **ยังไม่มีอยู่จริง** แล้วรันดู มันต้อง **แดง (fail)**
   เสมอ ถ้าเขียนเทสต์แล้วผ่านตั้งแต่แรกโดยไม่ได้เขียนโค้ด production เพิ่มเลย แปลว่าเทสต์นั้น
   ไม่ได้ทดสอบอะไรจริงๆ (หรือพฤติกรรมนั้นมีอยู่แล้ว) ต้องแก้เทสต์ใหม่
2. **🟢 GREEN** — เขียนโค้ด production ที่ **น้อยที่สุดเท่าที่จะทำให้เทสต์ผ่าน** ไม่ต้องสวย ไม่ต้อง
   ครบทุกกรณี แค่ทำให้ไฟสีแดงกลายเป็นสีเขียวให้เร็วที่สุด
3. **🔵 REFACTOR** — ตอนนี้มีเทสต์ที่ผ่านคอยเป็น safety net แล้ว ปรับโครงสร้างโค้ด (ทั้ง
   production code และ/หรือ test code) ให้สะอาดขึ้น อ่านง่ายขึ้น โดย **ห้ามเปลี่ยนพฤติกรรม**
   — รันเทสต์ซ้ำหลังรีแฟกเตอร์ทุกครั้งเพื่อยืนยันว่ายังเขียวอยู่

แล้ววนกลับไปข้อ 1 ใหม่สำหรับพฤติกรรมถัดไป ทำซ้ำแบบนี้เรื่อยๆ จนฟีเจอร์สมบูรณ์

### ทำไม "เขียนเทสต์ก่อน" ถึงต่างจาก "เขียนเทสต์ทีหลัง"

หลายคนที่เขียนเทสต์เป็นอยู่แล้ว (จาก Part 046–049) อาจสงสัยว่า "เขียนโค้ดก่อนแล้วค่อยเขียนเทสต์
คลุมทีหลังก็ได้ผลลัพธ์เหมือนกันไม่ใช่เหรอ — ทั้งคู่ก็ได้ชุดเทสต์ที่ผ่านหมดในที่สุด" คำตอบคือ
**ไม่เหมือนกัน** เพราะสองแนวทางนี้ **บังคับให้คิดคนละลำดับ**:

| แง่มุม | เขียนเทสต์ทีหลัง (Test-After) | เขียนเทสต์ก่อน (Test-First / TDD) |
|---|---|---|
| สิ่งที่คิดก่อน | "โค้ดนี้ทำงานอย่างไร" (implementation) | "โค้ดนี้ควรถูกใช้งานอย่างไร" (interface/contract) |
| มุมมอง | มองจากข้างใน (คนเขียน) | มองจากข้างนอก (คนเรียกใช้) |
| ความเสี่ยง | เทสต์มักจะ "ยืนยันสิ่งที่โค้ดทำอยู่แล้ว" รวมถึง bug ที่ไม่รู้ตัว | เทสต์นิยาม "สิ่งที่โค้ดควรทำ" ก่อนโค้ดจะมีอยู่ จึงจับ bug ได้ตั้งแต่ต้น |
| ผลต่อการออกแบบ | โค้ดมักถูกออกแบบตามความสะดวกของคนเขียน แล้วค่อยดัดให้ทดสอบได้ทีหลัง (มักต้อง mock เยอะ) | โค้ดถูกบังคับให้ "ทดสอบได้ง่าย" ตั้งแต่การออกแบบครั้งแรก เพราะต้องเขียนเทสต์เรียกมันก่อนที่มันจะมีอยู่ |
| ความครอบคลุม | มักข้าม edge case ที่คนเขียนไม่ได้นึกถึงตอนเขียนโค้ด (เพราะกำลังโฟกัสที่ทำให้มันรันผ่าน ไม่ใช่คิดว่าอะไรจะพังได้) | แต่ละเทสต์บังคับให้ต้องนึกถึง 1 พฤติกรรม/1 edge case ก่อนเขียนโค้ดรองรับ |

ประเด็นที่สำคัญที่สุดคือแถวที่ 4: **"ผลต่อการออกแบบ"** — เมื่อเขียนเทสต์ก่อน เราจำเป็นต้องเขียน
โค้ดในมุมมองของ "ผู้เรียกใช้" เสมอ (เช่น `Comment.create!(post:, user:, body:)` ต้องเรียกใช้ง่าย
พอที่จะเขียนในเทสต์ได้อย่างเป็นธรรมชาติ) สิ่งนี้ผลักดันให้เกิด dependency injection, small focused
methods, และ interface ที่ชัดเจนโดยอัตโนมัติ — ไม่ใช่เพราะ "หลักการออกแบบที่ดี" บอกให้ทำ แต่เพราะ
**การเขียนเทสต์ก่อนทำให้เขียนโค้ดที่ทดสอบยากไม่ได้ตั้งแต่แรก** เราจะเห็นผลลัพธ์ของแนวคิดนี้ชัดเจน
ตลอดทั้ง Part นี้ โดยเฉพาะตอนที่ authorization (Step 496) และ service object (Step 498) ถูกดึงออกมา
เป็นหน่วยที่ทดสอบแยกจากกันได้ทันที โดยไม่ต้องคิดย้อนกลับไปออกแบบใหม่

> **ข้อควรระวัง:** TDD ไม่ใช่ "เขียนเทสต์ให้ครบ 100% coverage" และไม่ใช่ "ห้ามคิดก่อนเขียนเทสต์"
> — มันคือวินัยของ**ลำดับการทำงาน** เท่านั้น: เทสต์แดงมาก่อนเสมอ โค้ดที่ทำให้เทสต์เขียวมาทีหลัง
> เสมอ ส่วนจะออกแบบอะไรยังไงเป็นเรื่องที่ต้องคิดเหมือนเดิม เพียงแต่คิดผ่านเทสต์แทนที่จะคิดผ่าน
> การเขียน method ตรงๆ

### เตรียมเครื่องมือ: ติดตั้ง gem ที่จำเป็นทั้งหมด

ก่อนเริ่ม TDD ให้ตรวจสอบว่า `Gemfile` มี gem ครบตามที่ Part 046–049 ตั้งค่าไว้แล้ว (ถ้าเป็นโปรเจกต์
ต่อเนื่องจาก Part ก่อนหน้า ส่วนนี้มีอยู่แล้ว ข้ามไปอ่าน Step 492 ได้เลย):

```ruby
# Gemfile
gem "bcrypt", "~> 3.1.7"   # has_secure_password (Part 041)
gem "pundit", "~> 2.5"     # authorization (Part 043)

group :development, :test do
  gem "rspec-rails", "~> 8.0"          # Part 046
  gem "factory_bot_rails", "~> 6.5"    # Part 047
end

group :test do
  gem "faker", "~> 3.5"                # Part 047
  gem "capybara", "~> 3.40"            # Part 048
  gem "selenium-webdriver", "~> 4.49"  # Part 048 (JS system test เมื่อจำเป็น)
  gem "simplecov", "~> 0.22", require: false  # Part 049
end
```

```bash
bundle install
```

และ `spec/rails_helper.rb` ต้องมีสามส่วนนี้ครบ (ทบทวนจาก Part 046/047/048/049):

```ruby
# spec/rails_helper.rb (ส่วนที่เกี่ยวข้อง)

# SimpleCov ต้อง start ก่อนโค้ดแอปพลิเคชันตัวแรกจะถูก require เข้ามา (Part 049)
require "simplecov"
SimpleCov.start "rails" do
  add_filter "/spec/"
  add_group "Policies", "app/policies"
  add_group "Services", "app/services"
end

# ...
require "rspec/rails"
require "capybara/rspec"   # Part 048

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods   # Part 047 — เรียก create(:user) ตรงๆ ได้
  config.infer_spec_type_from_file_location!

  config.before(:suite) do
    Faker::Config.locale = "th"   # ให้ Faker สุ่มชื่อ/ประโยคเป็นภาษาไทย (Part 047)
  end
end

Capybara.default_driver = :rack_test
Capybara.javascript_driver = :selenium_chrome_headless
```

### Kata อุ่นเครื่อง: `PasswordStrength` — สาธิตวงจร RGR แบบเต็มๆ ก่อนลงมือกับฟีเจอร์จริง

ก่อนจะเข้าฟีเจอร์ Comment Moderation ที่มีหลายชั้น (model → request → authorization → system)
ลองสาธิตวงจร Red-Green-Refactor แบบสมบูรณ์บนปัญหาเล็กๆ ที่เห็นภาพได้ในหน้าจอเดียวก่อน: เขียน
class `PasswordStrength` ที่ให้คะแนนความปลอดภัยของรหัสผ่าน (`:weak`/`:medium`/`:strong`) — เลือก
โจทย์นี้เพราะมันเกี่ยวกับ `User#password` ที่มีอยู่แล้วในระบบ (Part 041) พอดี

**🔴 RED รอบที่ 1 — เขียนเทสต์กรณีแรกก่อนที่ class จะมีอยู่จริง:**

```ruby
# spec/services/password_strength_spec.rb
require "rails_helper"

RSpec.describe PasswordStrength do
  describe ".score" do
    it "คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร" do
      expect(described_class.score("1234")).to eq(:weak)
    end
  end
end
```

รันทันที (ไม่ต้องรอ ไม่ต้องเดา) — และนี่คือผลลัพธ์จริงจาก terminal:

```
An error occurred while loading ./spec/services/password_strength_spec.rb.
Failure/Error:
  RSpec.describe PasswordStrength do
    describe ".score" do
      it "คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร" do
        expect(described_class.score("1234")).to eq(:weak)
      end
    end
  end

NameError:
  uninitialized constant PasswordStrength
# ./spec/services/password_strength_spec.rb:5:in `<top (required)>'
No examples found.

Finished in 0.00007 seconds (files took 1.43 seconds to load)
0 examples, 0 failures, 1 error occurred outside of examples
```

แดงตามคาด — `PasswordStrength` ไม่มีอยู่จริงเลยด้วยซ้ำ (`uninitialized constant` ไม่ใช่แค่
assertion ล้มเหลว) นี่คือ RED ที่ "แดงเพราะเหตุผลที่ถูกต้อง"

**🟢 GREEN รอบที่ 1 — เขียนโค้ดน้อยที่สุดให้ผ่าน (แม้จะดู "โกง" ก็ตาม):**

```ruby
# app/services/password_strength.rb
class PasswordStrength
  def self.score(_password)
    :weak
  end
end
```

```
PasswordStrength
  .score
    คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร

Finished in 0.00778 seconds (files took 1.4 seconds to load)
1 example, 0 failures
```

เขียวจริง! ใช่ว่ามันจะดูเหมือนโกง (คืน `:weak` ตายตัวไม่สนใจ input เลย) — แต่นี่คือเทคนิค TDD ที่
เรียกว่า **"Fake it"**: เขียนคำตอบที่ง่ายที่สุดเท่าที่จะทำให้เทสต์ปัจจุบันผ่านก่อน แล้วปล่อยให้
**เทสต์ถัดไป** เป็นคนบังคับให้ implementation ต้องฉลาดขึ้น

**🔴 RED รอบที่ 2 — เพิ่มเทสต์ที่บังคับให้ "โกง" ต่อไปไม่ได้ (Triangulation):**

```ruby
    it "คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก" do
      expect(described_class.score("abcdefgh")).to eq(:medium)
    end

    it "คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ" do
      expect(described_class.score("Abcdefg1!")).to eq(:strong)
    end
```

```
  1) PasswordStrength.score คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก
     Failure/Error: expect(described_class.score("abcdefgh")).to eq(:medium)

       expected: :medium
            got: :weak

  2) PasswordStrength.score คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ
     Failure/Error: expect(described_class.score("Abcdefg1!")).to eq(:strong)

       expected: :strong
            got: :weak

Finished in 0.01818 seconds (files took 1.17 seconds to load)
3 examples, 2 failures
```

แดงอีกครั้ง เพราะ implementation ตายตัวคืน `:weak` เสมอ — เทคนิคนี้เรียกว่า **triangulation**:
ใช้เทสต์หลายกรณีค่อยๆ "บีบ" ให้ implementation ทั่วไป (general) ขึ้นทีละนิด แทนที่จะพยายามคิด
solution สมบูรณ์ตั้งแต่แรก

**🟢 GREEN รอบที่ 2 — generalize implementation ให้ผ่านทั้ง 3 เทสต์:**

```ruby
class PasswordStrength
  def self.score(password)
    return :weak if password.length < 8
    return :strong if strong?(password)

    :medium
  end

  def self.strong?(password)
    password.match?(/[a-z]/) && password.match?(/[A-Z]/) &&
      password.match?(/\d/) && password.match?(/[^a-zA-Z0-9]/)
  end
end
```

```
PasswordStrength
  .score
    คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร
    คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก
    คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ

Finished in 0.01043 seconds (files took 1.25 seconds to load)
3 examples, 0 failures
```

**🔵 REFACTOR — ตอนนี้มี safety net 3 เทสต์แล้ว ปรับโครงสร้างให้สะอาดขึ้นได้อย่างมั่นใจ:**

```ruby
# รีแฟกเตอร์: ย้าย pattern ที่ตรวจสอบออกมาเป็นค่าคงที่ที่อ่านความหมายได้ทันที
# และซ่อน strong? ไว้เป็น private เพราะเป็นรายละเอียดภายในที่ผู้ใช้ class ไม่จำเป็นต้องรู้
class PasswordStrength
  REQUIRED_STRONG_PATTERNS = [/[a-z]/, /[A-Z]/, /\d/, /[^a-zA-Z0-9]/].freeze

  class << self
    def score(password)
      return :weak if password.length < 8
      return :strong if strong?(password)

      :medium
    end

    private

    def strong?(password)
      REQUIRED_STRONG_PATTERNS.all? { |pattern| password.match?(pattern) }
    end
  end
end
```

```
PasswordStrength
  .score
    คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร
    คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก
    คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ

Finished in 0.01625 seconds (files took 1.48 seconds to load)
3 examples, 0 failures
```

ยังเขียวเหมือนเดิมทุกตัว — **นี่คือหัวใจของ "Refactor ปลอดภัย"**: เปลี่ยนโครงสร้างภายในทั้งหมด
(เพิ่ม constant, เปลี่ยน `self.` เป็น `class << self`, ทำ `strong?` เป็น private) โดยไม่ต้องมานั่ง
ทดสอบด้วยมือว่า behavior เดิมยังอยู่ครบ — แค่กด `rspec` ซ้ำ ถ้าเขียวเหมือนเดิมคือปลอดภัย ถ้ามีตัวไหน
แดงขึ้นมาคือรีแฟกเตอร์พลาดไปเปลี่ยน behavior โดยไม่ตั้งใจ

**ตั้งแต่ Step 492 เป็นต้นไป เราจะทำวงจรนี้ซ้ำกับฟีเจอร์ Comment Moderation จริง — ไม่มีบรรทัดโค้ด
production แม้แต่บรรทัดเดียวที่จะถูกเขียนขึ้นโดยไม่มีเทสต์สีแดงเรียกร้องมันก่อน** และเพื่อความ
กระชับ ตั้งแต่นี้ไปจะ**ตัดบรรทัดท้าย ๆ ของผลลัพธ์ที่เกี่ยวกับ `Coverage report generated...` และ
`Line Coverage: ...%` ออกจาก output ที่แสดง** (SimpleCov ทำงานอยู่เบื้องหลังทุกครั้งที่รัน `rspec`
อยู่แล้วโดยอัตโนมัติตามที่ตั้งค่าไว้ข้างต้น) แล้วจะกลับมาดูตัวเลข coverage เต็มรูปแบบอีกครั้งใน
Step 500

---

## Step 492: TDD Cycle 1 (RED) — เขียน Model Spec ของ `Comment` ก่อนที่ `Comment` จะมีตัวตนจริง

ฟีเจอร์ Comment Moderation เริ่มจากหน่วยที่เล็กที่สุดและมี dependency น้อยที่สุดก่อนเสมอ: **โมเดล
`Comment`** — ยังไม่สนใจ route, controller, หรือ authorization ในตอนนี้เลย สนใจแค่ "ข้อมูล
ความเห็นหนึ่งชิ้นหน้าตาเป็นอย่างไร" ก่อน

`Comment` ต้องมีคุณสมบัติพื้นฐาน 2 อย่าง (ยังไม่พูดเรื่อง moderation status — นั่นคือ Cycle ถัดไป
ใน Step 495):

1. `belongs_to :post` และ `belongs_to :user` — ความเห็นหนึ่งชิ้นต้องผูกกับบทความหนึ่งชิ้นและ
   ผู้ใช้หนึ่งคนเสมอ
2. `body` ต้องไม่ว่างเปล่า (validation พื้นฐานที่สุด)

เขียนเทสต์นี้ **ก่อน** ที่จะสร้างไฟล์ `app/models/comment.rb` หรือ migration ใดๆ เลยด้วยซ้ำ:

```ruby
# spec/models/comment_spec.rb
require "rails_helper"

RSpec.describe Comment, type: :model do
  let(:user) { User.create!(name: "ผู้แสดงความเห็น", email: "commenter@example.com", password: "password123") }
  let(:author) { User.create!(name: "ผู้เขียน", email: "author2@example.com", password: "password123") }
  let(:post_record) { Post.create!(title: "บทความ", body: "เนื้อหา", user: author) }

  describe "associations" do
    it "belongs_to :post" do
      comment = described_class.new(post: post_record, user: user, body: "ดีมาก")
      expect(comment.post).to eq(post_record)
    end

    it "belongs_to :user" do
      comment = described_class.new(post: post_record, user: user, body: "ดีมาก")
      expect(comment.user).to eq(user)
    end
  end

  describe "validations" do
    it "ไม่ valid เมื่อ body ว่างเปล่า" do
      comment = described_class.new(post: post_record, user: user, body: "")
      expect(comment).not_to be_valid
      expect(comment.errors[:body]).to include("can't be blank")
    end

    it "valid เมื่อข้อมูลครบถ้วน" do
      comment = described_class.new(post: post_record, user: user, body: "ดีมาก")
      expect(comment).to be_valid
    end
  end
end
```

รันดู (ยังไม่มีทั้ง migration และ model):

```bash
bundle exec rspec spec/models/comment_spec.rb
```

```
An error occurred while loading ./spec/models/comment_spec.rb.
Failure/Error:
  RSpec.describe Comment, type: :model do
    let(:user) { User.create!(name: "ผู้แสดงความเห็น", email: "commenter@example.com", password: "password123") }
    let(:author) { User.create!(name: "ผู้เขียน", email: "author2@example.com", password: "password123") }
    let(:post_record) { Post.create!(title: "บทความ", body: "เนื้อหา", user: author) }

    describe "associations" do
      it "belongs_to :post" do
        comment = described_class.new(post: post_record, user: user, body: "ดีมาก")
        expect(comment.post).to eq(post_record)
      end

NameError:
  uninitialized constant Comment
# ./spec/models/comment_spec.rb:5:in `<top (required)>'
No examples found.

Finished in 0.00005 seconds (files took 1.48 seconds to load)
0 examples, 0 failures, 1 error occurred outside of examples
```

🔴 **RED สมบูรณ์แบบ** — `uninitialized constant Comment` ยืนยันชัดเจนว่าเทสต์นี้กำลังทดสอบสิ่งที่
ยังไม่มีอยู่จริง ไม่ใช่ false positive ข้อสังเกตสำคัญคือ **RSpec ไม่สามารถ "load" ไฟล์ spec ได้
เลยด้วยซ้ำ** เพราะ `Comment` ถูกอ้างถึงตั้งแต่บรรทัดแรกของ `describe` — นี่เป็นเรื่องปกติมากตอน
เริ่ม TDD cycle ใหม่ที่ยังไม่มี class อยู่เลย (ต่างจากตอนเทสต์ behavior ใหม่ของ class ที่มีอยู่
แล้ว ซึ่งจะ error ที่ระดับ example แทน)

---

## Step 493: TDD Cycle 1 (GREEN) — สร้าง Migration และ `Comment` Model ให้เทสต์ผ่านด้วยโค้ดน้อยที่สุด

ตอนนี้มีเทสต์สีแดงที่บอกชัดเจนว่าต้องมีอะไรบ้าง: `Comment` ที่มี `post_id`, `user_id`, `body`,
และมี `belongs_to`/`validates` ตามนั้น — สร้าง migration ก่อน:

```bash
bin/rails generate migration CreateComments post:references user:references body:text
bin/rails db:migrate
```

```
== [timestamp] CreateComments: migrating ===================================
-- create_table(:comments)
   -> 0.0032s
== [timestamp] CreateComments: migrated (0.0032s) ==========================
```

`post:references user:references` สร้างคอลัมน์ `post_id`/`user_id` พร้อม index ให้อัตโนมัติ —
ทบทวนจาก Part 027 (association เบื้องต้น) แล้วเขียน model ให้น้อยที่สุดเท่าที่เทสต์ต้องการ:

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user

  validates :body, presence: true
end
```

อย่าลืมเพิ่ม `has_many :comments` ที่ฝั่ง `Post` ด้วย (association เป็นสองทาง):

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy   # <- เพิ่มบรรทัดนี้

  validates :title, presence: true
  validates :body, presence: true
end
```

รันเทสต์ซ้ำ:

```bash
bundle exec rspec spec/models/comment_spec.rb
```

```
Comment
  associations
    belongs_to :post
    belongs_to :user
  validations
    ไม่ valid เมื่อ body ว่างเปล่า
    valid เมื่อข้อมูลครบถ้วน

Finished in 1.09 seconds (files took 3.04 seconds to load)
4 examples, 0 failures
```

🟢 **GREEN** — ทั้ง 4 examples ผ่าน **ด้วยโค้ด production แค่ 6 บรรทัด** (ไม่นับ `end`) นี่คือ
สิ่งที่ TDD บังคับให้เกิดขึ้นโดยธรรมชาติ: เขียนแค่พอที่เทสต์ต้องการ ไม่มี validation เกินความ
จำเป็น ไม่มี method ที่ยังไม่มีใครเรียกใช้

**REFACTOR รอบนี้:** ไม่มีอะไรต้องรีแฟกเตอร์ — โค้ด 6 บรรทัดนี้อ่านง่ายและตรงไปตรงมาที่สุดแล้ว
**นี่คือผลลัพธ์ที่ถูกต้องเช่นกัน** วงจร Red-Green-Refactor ไม่ได้บังคับว่าทุกรอบต้องมีการรีแฟกเตอร์
เสมอไป — ถ้าโค้ดหลัง GREEN สะอาดอยู่แล้ว ก็เดินหน้าไป cycle ถัดไปได้เลยโดยไม่ต้องฝืนหาอะไรมาแก้

---

## Step 494: TDD Cycle 2 — สร้างความเห็นผ่าน Nested Route ด้วย Request Spec (RED → GREEN → REFACTOR)

Cycle ถัดไปคือ "ผู้ใช้ที่ล็อกอินแล้วแสดงความเห็นใต้บทความได้" — ระดับนี้ไม่ใช่ unit เดี่ยวๆ แล้ว
แต่เป็น **Request Spec** (ทบทวน Part 046) ที่ทดสอบทั้ง route → controller → model ทำงานร่วมกัน
ผ่าน HTTP request จริง

ออกแบบ endpoint ก่อน (นี่คือส่วนหนึ่งของ "คิดจากมุมมองผู้เรียกใช้" ที่ Step 491 พูดถึง):
`POST /posts/:post_id/comments` — nested route ใต้ post เพราะความเห็นไม่มีตัวตนถ้าไม่มีบทความ
เจ้าของ (ทบทวน nested resources จาก Part 022)

**🔴 RED — เขียน request spec ก่อนที่ route และ controller จะมีอยู่จริง:**

```ruby
# spec/requests/comments_spec.rb
require "rails_helper"

RSpec.describe "Comments", type: :request do
  let(:author) { User.create!(name: "ผู้เขียน", email: "author3@example.com", password: "password123") }
  let(:commenter) { User.create!(name: "ผู้อ่าน", email: "reader3@example.com", password: "password123") }
  let(:post_record) { Post.create!(title: "บทความทดสอบ", body: "เนื้อหา", user: author) }

  describe "POST /posts/:post_id/comments" do
    context "เมื่อเข้าสู่ระบบแล้ว" do
      before { login_as(commenter) }

      it "สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่" do
        expect do
          post post_comments_path(post_record), params: { comment: { body: "เยี่ยมมาก!" } }
        end.to change(Comment, :count).by(1)

        comment = Comment.last
        expect(comment.post).to eq(post_record)
        expect(comment.user).to eq(commenter)
        expect(comment.body).to eq("เยี่ยมมาก!")
      end

      it "redirect กลับไปหน้า post หลังสร้างสำเร็จ" do
        post post_comments_path(post_record), params: { comment: { body: "เยี่ยมมาก!" } }
        expect(response).to redirect_to(post_path(post_record))
      end

      it "ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา" do
        expect do
          post post_comments_path(post_record), params: { comment: { body: "" } }
        end.not_to change(Comment, :count)

        expect(response).to have_http_status(:unprocessable_entity)
      end
    end

    context "เมื่อยังไม่เข้าสู่ระบบ" do
      it "redirect ไปหน้า login แทนการสร้างความเห็น" do
        expect do
          post post_comments_path(post_record), params: { comment: { body: "เยี่ยมมาก!" } }
        end.not_to change(Comment, :count)

        expect(response).to redirect_to(login_path)
      end
    end
  end
end
```

สังเกตว่ามีการเรียก helper `login_as(user)` ที่ยังไม่มีอยู่ — เพิ่ม helper กลางนี้ไว้ใน
`spec/support/` (เขียนครั้งเดียว ใช้ได้ทุก request/system spec ที่เหลือของ Part นี้):

```ruby
# spec/support/login_helpers.rb
module LoginHelpers
  DEFAULT_PASSWORD = "password123"

  # request spec: login ผ่าน POST จริง (เหมือนผู้ใช้กรอกฟอร์ม) session ที่ได้จะถูกเก็บต่อเนื่อง
  def login_as(user, password: DEFAULT_PASSWORD)
    post login_path, params: { email: user.email, password: password }
  end

  # system spec: กรอกฟอร์มจริงผ่าน Capybara (ใช้ใน Step 497)
  def sign_in_via_ui(user, password: DEFAULT_PASSWORD)
    visit login_path
    fill_in "อีเมล", with: user.email
    fill_in "รหัสผ่าน", with: password
    click_button "เข้าสู่ระบบ"
  end
end

RSpec.configure do |config|
  config.include LoginHelpers, type: :request
  config.include LoginHelpers, type: :system
end
```

และเปิดให้ `rails_helper.rb` โหลดไฟล์ใน `spec/support/` อัตโนมัติ (บรรทัดนี้ถูก comment ไว้โดย
generator ตั้งแต่แรก — เปิดใช้งานตอนนี้):

```ruby
# spec/rails_helper.rb
Rails.root.glob('spec/support/**/*.rb').sort_by(&:to_s).each { |f| require f }
```

รันดู:

```bash
bundle exec rspec spec/requests/comments_spec.rb
```

```
Comments
  POST /posts/:post_id/comments
    เมื่อเข้าสู่ระบบแล้ว
      สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่ (FAILED - 1)
      redirect กลับไปหน้า post หลังสร้างสำเร็จ (FAILED - 2)
      ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา (FAILED - 3)
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login แทนการสร้างความเห็น (FAILED - 4)

Failures:

  1) Comments POST /posts/:post_id/comments เมื่อเข้าสู่ระบบแล้ว สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่
     Failure/Error: post post_comments_path(post_record), params: { comment: { body: "เยี่ยมมาก!" } }

     NoMethodError:
       undefined method `post_comments_path' for ...

Finished in 0.12195 seconds (files took 1.39 seconds to load)
4 examples, 4 failures
```

🔴 **RED ทั้ง 4 examples** — `post_comments_path` ไม่มีอยู่จริงเพราะ routes.rb ยังไม่ได้ประกาศ
nested resource นี้เลย นี่คือจุดที่ TDD ช่วยได้ชัดมาก: เทสต์บอกตรงๆ ว่าเราลืมอะไร แทนที่จะต้อง
เดาว่าทำไม browser ขึ้น routing error ตอนทดสอบด้วยมือ

**🟢 GREEN — เพิ่ม route, controller, และปรับ view ให้ครบ:**

```ruby
# config/routes.rb
resources :posts, only: %i[index show] do
  resources :comments, only: %i[create]
end
```

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :require_login

  def create
    @post = Post.find(params[:post_id])
    @comment = @post.comments.build(comment_params.merge(user: current_user))

    if @comment.save
      redirect_to post_path(@post), notice: "ส่งความเห็นเรียบร้อยแล้ว"
    else
      @comments = @post.comments
      render "posts/show", status: :unprocessable_entity
    end
  end

  private

  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

`require_login` เป็น private method ที่มีอยู่แล้วใน `ApplicationController` (จาก session-based
auth ของ Part 041) — `redirect_to login_path` เมื่อยังไม่ล็อกอิน ทำให้ context "เมื่อยังไม่เข้าสู่
ระบบ" ผ่านไปโดยอัตโนมัติโดยไม่ต้องเขียน logic แยก

ต้องแก้ `PostsController#show` และ view ให้มีฟอร์มส่งความเห็นด้วย (มิเช่นนั้น `render
"posts/show"` ตอน validation ล้มเหลวจะพังเพราะ `@comment` ไม่ได้ถูกกำหนดไว้):

```ruby
# app/controllers/posts_controller.rb
def show
  @post = Post.find(params[:id])
  @comment = Comment.new
  @comments = @post.comments
end
```

```erb
<%# app/views/posts/show.html.erb (ส่วนที่เพิ่ม) %>
<h2>ความเห็น</h2>
<ul id="comments">
  <% @comments.each do |comment| %>
    <li id="comment_<%= comment.id %>">
      <strong><%= comment.user.name %>:</strong> <%= comment.body %>
    </li>
  <% end %>
</ul>

<% if logged_in? %>
  <%= form_with model: @comment, url: post_comments_path(@post), local: true do |f| %>
    <div><%= f.label :body, "ความเห็นของคุณ" %><%= f.text_area :body %></div>
    <%= f.submit "ส่งความเห็น" %>
  <% end %>
<% else %>
  <p><%= link_to "เข้าสู่ระบบ", login_path %> เพื่อแสดงความเห็น</p>
<% end %>
```

รันซ้ำ:

```bash
bundle exec rspec spec/requests/comments_spec.rb
```

```
Comments
  POST /posts/:post_id/comments
    เมื่อเข้าสู่ระบบแล้ว
      สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่
      redirect กลับไปหน้า post หลังสร้างสำเร็จ
/opt/rbenv/versions/3.3.6/.../have_http_status.rb:219: warning: Status code :unprocessable_entity
is deprecated and will be removed in a future version of Rack. Please use :unprocessable_content instead.
      ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login แทนการสร้างความเห็น

Finished in 1.16 seconds (files took 1.17 seconds to load)
4 examples, 0 failures
```

🟢 **GREEN** — ทั้ง 4 examples ผ่าน แต่สังเกตบรรทัดเตือน (`warning:`) กลางผลลัพธ์ — Rails 8.1
เปลี่ยนชื่อ symbol มาตรฐานจาก `:unprocessable_entity` เป็น `:unprocessable_content` ตาม HTTP
spec ฉบับล่าสุด ตัวเก่ายังใช้ได้แต่จะถูกถอดออกในอนาคต

**🔵 REFACTOR — แก้ deprecation warning ที่ safety net เพิ่งจับได้:**

```ruby
# app/controllers/comments_controller.rb (เปลี่ยนบรรทัดเดียว)
render "posts/show", status: :unprocessable_content
```

```ruby
# spec/requests/comments_spec.rb (เปลี่ยนบรรทัดเดียวเช่นกัน)
expect(response).to have_http_status(:unprocessable_content)
```

```bash
bundle exec rspec spec/requests/comments_spec.rb
```

```
Comments
  POST /posts/:post_id/comments
    เมื่อเข้าสู่ระบบแล้ว
      สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่
      redirect กลับไปหน้า post หลังสร้างสำเร็จ
      ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login แทนการสร้างความเห็น

Finished in 1.31 seconds (files took 1.36 seconds to load)
4 examples, 0 failures
```

เขียวเหมือนเดิม ไม่มี warning แล้ว — นี่เป็นตัวอย่างที่ดีว่า **"Refactor" ไม่ได้จำกัดแค่การจัดโค้ด
ให้สวยเท่านั้น** การแก้ deprecation warning, เปลี่ยนไปใช้ API เวอร์ชันใหม่ที่แนะนำ, หรือลบโค้ดที่
ไม่ได้ใช้แล้ว ก็นับเป็น "รีแฟกเตอร์" ทั้งหมด ตราบใดที่ behavior จากมุมมองผู้ใช้ยังเหมือนเดิม —
และการมีเทสต์ที่ผ่านอยู่แล้วทำให้กล้าแก้จุดเล็กๆ แบบนี้ได้ทันทีโดยไม่ต้องกลัวว่าจะพังอะไรที่มองไม่เห็น

---

## Step 495: TDD Cycle 3 — Moderation State Machine ด้วย `enum` (pending/approved/rejected)

นี่คือหัวใจสำคัญที่สุดของฟีเจอร์: **ความเห็นทุกชิ้นต้องมีสถานะการตรวจสอบ** และสถานะนี้ควบคุมว่า
มันจะแสดงต่อสาธารณะหรือไม่ ออกแบบ business rule ก่อนเขียนโค้ด:

- ความเห็นใหม่ทุกชิ้นเริ่มที่สถานะ **`pending`** (รอตรวจสอบ) เสมอ ไม่มีทางสร้างความเห็นที่
  `approved` ตั้งแต่แรกได้
- `#approve!` เปลี่ยนสถานะเป็น **`approved`** และบันทึกทันที
- `#reject!` เปลี่ยนสถานะเป็น **`rejected`** และบันทึกทันที
- มีเฉพาะความเห็นที่ `approved` เท่านั้นที่ **"มองเห็นได้จากสาธารณะ"** (`.visible`)

**🔴 RED — เขียนเทสต์ state machine ทั้งหมดนี้ก่อนที่คอลัมน์ `status` จะมีอยู่จริงด้วยซ้ำ:**

```ruby
# spec/models/comment_spec.rb (เพิ่มเข้าไปในไฟล์เดิม)
describe "สถานะการตรวจสอบ (moderation state machine)" do
  it "มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่" do
    comment = described_class.create!(post: post_record, user: user, body: "รอตรวจสอบ")
    expect(comment).to be_pending
  end

  it "#approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที" do
    comment = described_class.create!(post: post_record, user: user, body: "รอตรวจสอบ")
    comment.approve!
    expect(comment).to be_approved
    expect(comment.reload).to be_approved
  end

  it "#reject! เปลี่ยนสถานะเป็น rejected และบันทึกลงฐานข้อมูลทันที" do
    comment = described_class.create!(post: post_record, user: user, body: "รอตรวจสอบ")
    comment.reject!
    expect(comment).to be_rejected
    expect(comment.reload).to be_rejected
  end

  describe ".visible" do
    it "คืนเฉพาะความเห็นที่ approved เท่านั้น ไม่รวม pending หรือ rejected" do
      approved = described_class.create!(post: post_record, user: user, body: "อนุมัติแล้ว").tap(&:approve!)
      pending = described_class.create!(post: post_record, user: user, body: "รอตรวจสอบ")
      rejected = described_class.create!(post: post_record, user: user, body: "ถูกปฏิเสธ").tap(&:reject!)

      expect(post_record.comments.visible).to contain_exactly(approved)
      expect(post_record.comments.visible).not_to include(pending, rejected)
    end
  end
end
```

```bash
bundle exec rspec spec/models/comment_spec.rb
```

```
Comment
  ...
  สถานะการตรวจสอบ (moderation state machine)
    มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่ (FAILED - 1)
    #approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที (FAILED - 2)
    #reject! เปลี่ยนสถานะเป็น rejected และบันทึกลงฐานข้อมูลทันที (FAILED - 3)
    .visible
      คืนเฉพาะความเห็นที่ approved เท่านั้น ไม่รวม pending หรือ rejected (FAILED - 4)

Failures:

  1) ... มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่
     Failure/Error: expect(comment).to be_pending
       expected #<Comment id: 1, ...> to respond to `pending?`

  2) ... #approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที
     Failure/Error: comment.approve!

     NoMethodError:
       undefined method `approve!' for an instance of Comment

Finished in 1.08 seconds (files took 1.32 seconds to load)
8 examples, 4 failures
```

🔴 **RED 4 examples** — ข้อความ error สองแบบต่างกันชัดเจน: `respond to 'pending?'` (ยังไม่มี
enum เลย) กับ `undefined method 'approve!'` (ยังไม่มี custom method) — เทสต์กำลังบอก spec ของ
สิ่งที่ต้องสร้างอย่างละเอียด โดยที่เรายังไม่ได้เขียน migration แม้แต่บรรทัดเดียว

**🟢 GREEN — เพิ่มคอลัมน์ `status` แล้วประกาศ `enum` พร้อม method ที่ต้องการ:**

```bash
bin/rails generate migration AddStatusToComments status:integer
```

แก้ migration ที่ generate มาให้มี `default` และ `null: false` (ทบทวนเหตุผลจาก Part 025 —
คอลัมน์ enum ไม่ควรปล่อยให้เป็น `NULL` ได้ เพราะจะทำให้ query สถานะพลาดง่าย):

```ruby
# db/migrate/xxx_add_status_to_comments.rb
class AddStatusToComments < ActiveRecord::Migration[8.1]
  def change
    add_column :comments, :status, :integer, default: 0, null: false
    add_index :comments, :status
  end
end
```

```bash
bin/rails db:migrate
```

```
== [timestamp] AddStatusToComments: migrating ==============================
-- add_column(:comments, :status, :integer, {:default=>0, :null=>false})
   -> 0.0024s
-- add_index(:comments, :status)
   -> 0.0007s
== [timestamp] AddStatusToComments: migrated (0.0032s) =====================
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user

  # enum ให้ทั้ง predicate method (pending?, approved?, rejected?) และ scope
  # (Comment.pending, Comment.approved, Comment.rejected) มาให้ฟรีทันที (ทบทวนแนวคิดจาก
  # Part 013 ที่ Rails สร้าง helper ให้อัตโนมัติจากการประกาศ metadata สั้นๆ)
  enum :status, { pending: 0, approved: 1, rejected: 2 }, default: :pending

  validates :body, presence: true

  # ความเห็นที่ "มองเห็นได้จากสาธารณะ" มีแค่สถานะ approved เท่านั้น
  scope :visible, -> { approved }

  def approve!
    update!(status: :approved)
  end

  def reject!
    update!(status: :rejected)
  end
end
```

> **ทำไมไม่ใช้ `comment.approved!`/`comment.rejected!` ที่ `enum` สร้างให้ฟรีอยู่แล้ว?**
> Rails `enum` สร้าง bang method ให้อัตโนมัติตามชื่อค่า (`comment.approved!` จะ set แล้ว save
> ให้ทันที) แต่เราเลือกห่อมันด้วย `#approve!`/`#reject!` ของเราเองแทน ด้วยเหตุผลด้าน**การออกแบบ
> โดเมน**: ชื่อ method ควรสื่อ **การกระทำ** (`approve`, `reject`) ไม่ใช่ **สถานะปลายทาง**
> (`approved`, `rejected`) — และที่สำคัญกว่านั้น มันเปิดจุดให้เพิ่ม "ผลข้างเคียง" ในอนาคต (เช่น
> ส่งอีเมลแจ้งเตือนตอนอนุมัติ — ดูแบบฝึกหัดต่อยอดท้าย Part) โดยไม่ต้องไปแก้ที่เรียกใช้ `enum`
> bang method หลายจุดทั่วโค้ด นี่คือการตัดสินใจออกแบบเล็กๆ ที่ TDD บังคับให้คิดตั้งแต่ตอนเขียน
> เทสต์ (เราเขียน `comment.approve!` ในเทสต์ตั้งแต่ต้น ไม่ใช่ `comment.approved!`)

รันซ้ำ:

```bash
bundle exec rspec spec/models/comment_spec.rb
```

```
Comment
  associations
    belongs_to :post
    belongs_to :user
  validations
    ไม่ valid เมื่อ body ว่างเปล่า
    valid เมื่อข้อมูลครบถ้วน
  สถานะการตรวจสอบ (moderation state machine)
    มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่
    #approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที
    #reject! เปลี่ยนสถานะเป็น rejected และบันทึกลงฐานข้อมูลทันที
    .visible
      คืนเฉพาะความเห็นที่ approved เท่านั้น ไม่รวม pending หรือ rejected

Finished in 1.09 seconds (files took 2.75 seconds to load)
8 examples, 0 failures
```

🟢 **GREEN ทั้ง 8 examples** — schema ที่ได้ตอนนี้:

```ruby
# db/schema.rb (ส่วนของตาราง comments)
create_table "comments", force: :cascade do |t|
  t.integer "post_id", null: false
  t.integer "user_id", null: false
  t.text "body"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.integer "status", default: 0, null: false
  t.index ["post_id"], name: "index_comments_on_post_id"
  t.index ["status"], name: "index_comments_on_status"
  t.index ["user_id"], name: "index_comments_on_user_id"
end
```

**REFACTOR รอบนี้:** โค้ดสะอาดพออยู่แล้วเช่นกัน (comment สั้นๆ อธิบายเหตุผลไว้ในโค้ดตรงจุด
ที่ตัดสินใจ ไม่ต้องแยกไฟล์หรือแยก class เพิ่ม) — จุดที่น่าสนใจกว่าคือ **`Comment.comments.visible`
ยังไม่ถูกใช้จริงในหน้าเว็บเลย** — นั่นคือ cycle ถัดไป

---

## Step 496: TDD Cycle 4 — จำกัดสิทธิ์อนุมัติ/ปฏิเสธด้วย Pundit (เฉพาะเจ้าของบทความเท่านั้น)

ตอนนี้ `Comment` มี method `approve!`/`reject!` แล้ว แต่ยังไม่มีใครเรียกมันผ่าน HTTP ได้เลย และที่
สำคัญกว่านั้น **ต้องไม่ใช่ใครก็เรียกได้** — มีแค่ **เจ้าของบทความ** (`post.user`) เท่านั้นที่มีสิทธิ์
อนุมัติหรือปฏิเสธความเห็นใต้บทความของตัวเอง (ไม่ใช่เจ้าของความเห็นเอง และไม่ใช่ผู้ใช้คนไหนก็ได้
ที่ล็อกอินอยู่) นี่คือจุดที่ **Pundit** (Part 043) เข้ามา

**🔴 RED — เขียน request spec ครอบคลุมทั้ง 3 บทบาท (เจ้าของ, คนอื่น, ยังไม่ล็อกอิน) ก่อน:**

```ruby
# spec/requests/comment_moderations_spec.rb
require "rails_helper"

RSpec.describe "Comment moderation", type: :request do
  let(:author) { User.create!(name: "ผู้เขียนบทความ", email: "author4@example.com", password: "password123") }
  let(:commenter) { User.create!(name: "ผู้แสดงความเห็น", email: "commenter4@example.com", password: "password123") }
  let(:stranger) { User.create!(name: "คนอื่น", email: "stranger4@example.com", password: "password123") }
  let(:post_record) { Post.create!(title: "บทความ", body: "เนื้อหา", user: author) }
  let!(:comment) { Comment.create!(post: post_record, user: commenter, body: "รอตรวจสอบ") }

  describe "PATCH /posts/:post_id/comments/:id/approve" do
    context "เมื่อเป็นเจ้าของบทความ" do
      before { login_as(author) }

      it "อนุมัติความเห็นสำเร็จ" do
        patch approve_post_comment_path(post_record, comment)
        expect(comment.reload).to be_approved
        expect(response).to redirect_to(post_path(post_record))
      end
    end

    context "เมื่อไม่ใช่เจ้าของบทความ" do
      before { login_as(stranger) }

      it "ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น" do
        patch approve_post_comment_path(post_record, comment)
        expect(comment.reload).to be_pending
        expect(response).to redirect_to(root_path)
      end
    end

    context "เมื่อยังไม่เข้าสู่ระบบ" do
      it "redirect ไปหน้า login" do
        patch approve_post_comment_path(post_record, comment)
        expect(comment.reload).to be_pending
        expect(response).to redirect_to(login_path)
      end
    end
  end

  describe "PATCH /posts/:post_id/comments/:id/reject" do
    context "เมื่อเป็นเจ้าของบทความ" do
      before { login_as(author) }

      it "ปฏิเสธความเห็นสำเร็จ" do
        patch reject_post_comment_path(post_record, comment)
        expect(comment.reload).to be_rejected
        expect(response).to redirect_to(post_path(post_record))
      end
    end

    context "เมื่อไม่ใช่เจ้าของบทความ (แม้จะเป็นคนโพสต์ความเห็นเองก็ตาม)" do
      before { login_as(commenter) }

      it "ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น" do
        patch reject_post_comment_path(post_record, comment)
        expect(comment.reload).to be_pending
        expect(response).to redirect_to(root_path)
      end
    end
  end
end
```

สังเกต context ที่สองของ `reject` เป็นพิเศษ: **แม้แต่คนที่โพสต์ความเห็นเองก็ยังต้องถูกปฏิเสธสิทธิ์
เช่นกัน** — นี่คือ edge case ที่มักถูกลืมถ้าเขียนโค้ดก่อนแล้วค่อยคิดเทสต์ทีหลัง (สัญชาตญาณผิดๆ
ที่พบบ่อยคือ "เจ้าของความเห็นน่าจะลบ/แก้ของตัวเองได้" ซึ่งไม่ใช่กฎของฟีเจอร์นี้)

```bash
bundle exec rspec spec/requests/comment_moderations_spec.rb
```

```
  1) Comment moderation PATCH /posts/:post_id/comments/:id/approve เมื่อเป็นเจ้าของบทความ อนุมัติความเห็นสำเร็จ
     Failure/Error: patch approve_post_comment_path(post_record, comment)

     NoMethodError:
       undefined method `approve_post_comment_path' for ...

  (... อีก 4 failures เหตุผลเดียวกัน ...)

Finished in 0.20679 seconds (files took 1.23 seconds to load)
5 examples, 5 failures
```

🔴 **RED ทั้ง 5 examples** — ยังไม่มี route `approve`/`reject` เลยด้วยซ้ำ

**🟢 GREEN — เพิ่ม route, Pundit policy, และ controller actions:**

```ruby
# config/routes.rb
resources :posts, only: %i[index show] do
  resources :comments, only: %i[create] do
    member do
      patch :approve
      patch :reject
    end
  end
end
```

```ruby
# app/policies/comment_policy.rb
# เฉพาะเจ้าของ "บทความ" (post.user) เท่านั้นที่มีสิทธิ์อนุมัติ/ปฏิเสธความเห็น —
# ไม่ใช่เจ้าของความเห็นเอง และไม่ใช่ผู้ใช้คนไหนก็ได้ที่ล็อกอินอยู่ (ทบทวน Part 043)
class CommentPolicy < ApplicationPolicy
  def approve?
    post_author?
  end

  def reject?
    post_author?
  end

  private

  def post_author?
    user.present? && record.post.user_id == user.id
  end
end
```

```ruby
# app/controllers/comments_controller.rb (เพิ่ม actions)
before_action :set_comment, only: %i[approve reject]

def approve
  authorize @comment
  @comment.approve!
  redirect_to post_path(@comment.post), notice: "อนุมัติความเห็นแล้ว"
end

def reject
  authorize @comment
  @comment.reject!
  redirect_to post_path(@comment.post), notice: "ปฏิเสธความเห็นแล้ว"
end

private

def set_comment
  @comment = Comment.find(params[:id])
end
```

`authorize @comment` จะเรียก `CommentPolicy#approve?`/`#reject?` ให้อัตโนมัติตามชื่อ action
ปัจจุบัน (ทบทวนกลไกนี้จาก Part 043) และถ้าคืน `false` จะ raise `Pundit::NotAuthorizedError`
ซึ่ง `ApplicationController` ดักไว้แล้วด้วย `rescue_from` (ตั้งค่าไว้ตอนติดตั้ง Pundit ตั้งแต่
Part 043):

```ruby
# app/controllers/application_controller.rb (ส่วนที่เกี่ยวข้อง — มีอยู่แล้วจาก Part 043)
rescue_from Pundit::NotAuthorizedError, with: :render_forbidden

private

def render_forbidden
  respond_to do |format|
    format.html { redirect_to root_path, alert: "คุณไม่มีสิทธิ์ทำรายการนี้" }
    format.any  { head :forbidden }
  end
end
```

```bash
bundle exec rspec spec/requests/comment_moderations_spec.rb
```

```
Comment moderation
  PATCH /posts/:post_id/comments/:id/approve
    เมื่อเป็นเจ้าของบทความ
      อนุมัติความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login
  PATCH /posts/:post_id/comments/:id/reject
    เมื่อเป็นเจ้าของบทความ
      ปฏิเสธความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ (แม้จะเป็นคนโพสต์ความเห็นเองก็ตาม)
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น

Finished in 0.22441 seconds (files took 1.4 seconds to load)
5 examples, 0 failures
```

🟢 **GREEN ทั้ง 5 examples** ตั้งแต่ครั้งแรกที่ลองรันหลังเขียนโค้ด — ไม่ต้องวนแก้หลายรอบ เพราะ
Pundit policy ที่เขียนตรงกับกฎที่เทสต์ต้องการเป๊ะๆ (นี่คือข้อดีของการเขียนเทสต์ที่ครอบคลุมทุกบทบาท
ไว้ก่อน — เห็นภาพครบก่อนเขียน policy จึงเขียนถูกตั้งแต่ครั้งแรก)

**REFACTOR:** ยังไม่รีแฟกเตอร์ตอนนี้ — รอให้ทั้งฟีเจอร์สมบูรณ์ (รวม system test ใน Step 497) ก่อน
แล้วค่อยมองภาพรวมทั้งหมดพร้อมกันใน Step 498 (การรีแฟกเตอร์ทีเดียวหลังเห็นโค้ดที่ซ้ำกันชัดเจนแล้ว
มักได้ผลลัพธ์ที่ดีกว่าการรีแฟกเตอร์ทุก cycle เล็กๆ ไปเรื่อยโดยยังไม่เห็น pattern ที่แท้จริง)

---

## Step 497: TDD Cycle 5 — System Test เต็มรูปแบบด้วย Capybara: ผู้เขียนเห็น เข้าไปอนุมัติความเห็น

Request spec ใน Step 494–496 ทดสอบแต่ละ endpoint แยกกัน แต่ยังไม่มีเทสต์ไหนยืนยันว่า **flow
ทั้งหมดที่ผู้ใช้จริงจะเจอ** ทำงานถูกต้องแบบ end-to-end: ผู้อ่านทั่วไปต้องไม่เห็นความเห็นที่รอ
ตรวจสอบ, ผู้เขียนบทความต้องเห็นรายการรอตรวจสอบพร้อมปุ่มจัดการ, กดอนุมัติแล้วความเห็นต้องย้ายไป
อยู่ในรายการสาธารณะทันที — นี่คืองานของ **System Test** (Part 048)

**🔴 RED — เขียน system spec ที่จำลอง flow เต็มรูปแบบ ก่อนที่ UI ส่วนนี้จะมีอยู่จริง:**

```ruby
# spec/system/comment_moderation_spec.rb
require "rails_helper"

RSpec.describe "Comment moderation flow", type: :system do
  let(:author) { User.create!(name: "ผู้เขียนบทความ", email: "sys_author@example.com", password: "password123") }
  let(:reader) { User.create!(name: "ผู้อ่าน", email: "sys_reader@example.com", password: "password123") }
  let(:post_record) { Post.create!(title: "ระบบ TDD ทำงานอย่างไร", body: "เนื้อหาบทความ", user: author) }

  it "ผู้เขียนบทความเห็นความเห็นที่รอตรวจสอบ อนุมัติได้ แล้วความเห็นนั้นแสดงต่อสาธารณะทันที" do
    pending_comment = Comment.create!(post: post_record, user: reader, body: "บทความดีมากครับ")

    # ผู้อ่านทั่วไปที่ยังไม่ล็อกอิน ต้องไม่เห็นความเห็นที่ยังไม่ผ่านการอนุมัติ
    visit post_path(post_record)
    expect(page).not_to have_content("บทความดีมากครับ")

    # เจ้าของบทความล็อกอินเข้ามา ต้องเห็นความเห็นที่รอตรวจสอบ พร้อมปุ่มจัดการ
    sign_in_via_ui(author)
    visit post_path(post_record)

    within("#pending_comments") do
      expect(page).to have_content("บทความดีมากครับ")
      click_button "อนุมัติ", match: :first
    end

    expect(page).to have_content("อนุมัติความเห็นแล้ว")
    expect(pending_comment.reload).to be_approved

    within("#comments") do
      expect(page).to have_content("บทความดีมากครับ")
    end
    expect(page).not_to have_css("#pending_comments")
  end

  it "เจ้าของบทความปฏิเสธความเห็นได้ และความเห็นจะไม่แสดงต่อสาธารณะ" do
    Comment.create!(post: post_record, user: reader, body: "ความเห็นสแปม")

    sign_in_via_ui(author)
    visit post_path(post_record)

    within("#pending_comments") { click_button "ปฏิเสธ", match: :first }

    expect(page).to have_content("ปฏิเสธความเห็นแล้ว")
    expect(page).not_to have_content("ความเห็นสแปม")
  end
end
```

```bash
bundle exec rspec spec/system/comment_moderation_spec.rb
```

รันครั้งแรกเจอปัญหาที่ไม่เกี่ยวกับ business logic เลย:

```
Selenium::WebDriver::Error::WebDriverError:
  Unsuccessful command executed: [".../selenium-manager", "--browser", "chrome", ...] - Code 65
  {"code"=>65, "message"=>"error sending request for url
  (https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json)", ...}

Finished in 1.39 seconds (files took 1.18 seconds to load)
2 examples, 2 failures
```

นี่ไม่ใช่ RED ที่เกี่ยวกับฟีเจอร์เลย — rspec-rails ตั้งค่า driver เริ่มต้นของ `type: :system`
เป็นตัวที่รองรับ JavaScript (พยายามเปิด Chrome จริงผ่าน Selenium) แต่ฟีเจอร์ moderation ของเรา
**ไม่มี JavaScript แม้แต่บรรทัดเดียว** (ปุ่มอนุมัติ/ปฏิเสธเป็นแค่ `button_to` ที่ submit form ปกติ)
จึงกำหนด driver default ให้ตรงกับความเป็นจริงของฟีเจอร์ (ทบทวน Part 048 เรื่องการเลือก driver
ให้เหมาะกับสิ่งที่ทดสอบ):

```ruby
# spec/support/capybara.rb
# ฟีเจอร์ moderation ไม่มี JavaScript เลย จึงให้ system spec ทุกตัวใช้ rack_test เป็น
# default (เร็วกว่ามาก) ยกเว้น spec ที่ติด tag `js: true` เท่านั้น
RSpec.configure do |config|
  config.before(:each, type: :system) { driven_by :rack_test }
  config.before(:each, type: :system, js: true) { driven_by :selenium_chrome_headless }
end
```

รันซ้ำ:

```bash
bundle exec rspec spec/system/comment_moderation_spec.rb
```

```
  1) Comment moderation flow ผู้เขียนบทความเห็นความเห็นที่รอตรวจสอบ อนุมัติได้ แล้วความเห็นนั้นแสดงต่อสาธารณะทันที
     Failure/Error: expect(page).not_to have_content("บทความดีมากครับ")
       expected not to find text "บทความดีมากครับ" in "หน้าแรก | เข้าสู่ระบบ
       ระบบ TDD ทำงานอย่างไร
       โดย ผู้เขียนบทความ
       เนื้อหาบทความ
       ความเห็น
       ผู้อ่าน: บทความดีมากครับ
       เข้าสู่ระบบ เพื่อแสดงความเห็น"

  2) Comment moderation flow เจ้าของบทความปฏิเสธความเห็นได้ และความเห็นจะไม่แสดงต่อสาธารณะ
     Failure/Error:
       within("#pending_comments") do
         click_button "ปฏิเสธ", match: :first
       end

     Capybara::ElementNotFound:
       Unable to find css "#pending_comments"

Finished in 1.11 seconds (files took 1.49 seconds to load)
2 examples, 2 failures
```

🔴 **ตอนนี้คือ RED "จริง" ของฟีเจอร์แล้ว** — และมันบอกตรงประเด็นสองเรื่องพอดี: (1) หน้า post ยัง
แสดงความเห็น **ทุกสถานะ** ปนกันหมด (ไม่มีการกรอง `.visible`) และ (2) ยังไม่มี UI รายการรอตรวจสอบ
เลยด้วยซ้ำ (`#pending_comments` หาไม่เจอ)

**🟢 GREEN — ใช้ `.visible` (จาก Step 495) กรองรายการสาธารณะ และเพิ่ม UI จัดการสำหรับเจ้าของ:**

```ruby
# app/controllers/posts_controller.rb
def show
  @post = Post.find(params[:id])
  @comment = Comment.new
  @comments = @post.comments.visible

  # เฉพาะเจ้าของบทความเท่านั้นที่ต้องเห็นรายการรอตรวจสอบ
  @pending_comments = @post.comments.pending if logged_in? && current_user == @post.user
end
```

```erb
<%# app/views/posts/show.html.erb (เพิ่มส่วนรายการรอตรวจสอบ) %>
<% if @pending_comments.present? %>
  <h2>ความเห็นที่รอตรวจสอบ (เห็นเฉพาะเจ้าของบทความ)</h2>
  <ul id="pending_comments">
    <% @pending_comments.each do |comment| %>
      <li id="pending_comment_<%= comment.id %>">
        <strong><%= comment.user.name %>:</strong> <%= comment.body %>
        <%= button_to "อนุมัติ", approve_post_comment_path(@post, comment), method: :patch %>
        <%= button_to "ปฏิเสธ", reject_post_comment_path(@post, comment), method: :patch %>
      </li>
    <% end %>
  </ul>
<% end %>
```

```bash
bundle exec rspec spec/system/comment_moderation_spec.rb
```

```
Comment moderation flow
  ผู้เขียนบทความเห็นความเห็นที่รอตรวจสอบ อนุมัติได้ แล้วความเห็นนั้นแสดงต่อสาธารณะทันที
  เจ้าของบทความปฏิเสธความเห็นได้ และความเห็นจะไม่แสดงต่อสาธารณะ

Finished in 0.92586 seconds (files took 1.42 seconds to load)
2 examples, 0 failures
```

🟢 **GREEN ทั้งสอง system tests** — flow เต็มรูปแบบทำงานถูกต้องตั้งแต่หน้าเว็บสาธารณะไปจนถึง
การจัดการของเจ้าของบทความ มาถึงจุดนี้ลองรัน **ชุดทดสอบทั้งหมดของฟีเจอร์นี้** พร้อมกันดูสักครั้ง
เพื่อยืนยันว่าไม่มีอะไรที่ cycle หลังไปทำ cycle ก่อนหน้าพัง:

```bash
bundle exec rspec
```

```
Comment
  associations
    belongs_to :post
    belongs_to :user
  validations
    ไม่ valid เมื่อ body ว่างเปล่า
    valid เมื่อข้อมูลครบถ้วน
  สถานะการตรวจสอบ (moderation state machine)
    มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่
    #approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที
    #reject! เปลี่ยนสถานะเป็น rejected และบันทึกลงฐานข้อมูลทันที
    .visible
      คืนเฉพาะความเห็นที่ approved เท่านั้น ไม่รวม pending หรือ rejected

Comment moderation
  PATCH /posts/:post_id/comments/:id/approve
    เมื่อเป็นเจ้าของบทความ
      อนุมัติความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login
  PATCH /posts/:post_id/comments/:id/reject
    เมื่อเป็นเจ้าของบทความ
      ปฏิเสธความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ (แม้จะเป็นคนโพสต์ความเห็นเองก็ตาม)
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น

Comments
  POST /posts/:post_id/comments
    เมื่อเข้าสู่ระบบแล้ว
      สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่
      redirect กลับไปหน้า post หลังสร้างสำเร็จ
      ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login แทนการสร้างความเห็น

PasswordStrength
  .score
    คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร
    คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก
    คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ

Comment moderation flow
  ผู้เขียนบทความเห็นความเห็นที่รอตรวจสอบ อนุมัติได้ แล้วความเห็นนั้นแสดงต่อสาธารณะทันที
  เจ้าของบทความปฏิเสธความเห็นได้ และความเห็นจะไม่แสดงต่อสาธารณะ

Finished in 1.19 seconds (files took 1.22 seconds to load)
22 examples, 0 failures
```

**22 examples, 0 failures** — ฟีเจอร์สมบูรณ์ในระดับ "ใช้งานได้จริง" แล้ว ทุกบรรทัดของ
production code ที่มีอยู่ตอนนี้ถูกเรียกร้องขึ้นมาโดยเทสต์สักตัวเสมอ ไม่มีโค้ดส่วนไหนที่เขียนขึ้น
มา "เผื่อไว้" โดยไม่มีเทสต์รองรับ

---

## Step 498: Refactor Pass — สกัด `CommentModerationService` โดยชุดทดสอบทั้งหมดยังผ่านเหมือนเดิม

ตอนนี้ทุกเทสต์เขียวหมดแล้ว และนี่คือจังหวะที่เหมาะที่สุดสำหรับการ **มองภาพรวม** แทนที่จะรีแฟกเตอร์
ทีละ cycle เล็กๆ — ดูโค้ด `CommentsController#approve`/`#reject` อีกครั้ง:

```ruby
def approve
  authorize @comment
  @comment.approve!
  redirect_to post_path(@comment.post), notice: "อนุมัติความเห็นแล้ว"
end

def reject
  authorize @comment
  @comment.reject!
  redirect_to post_path(@comment.post), notice: "ปฏิเสธความเห็นแล้ว"
end
```

สองบรรทัดนี้ (`@comment.approve!` / `@comment.reject!`) ดูเรียบง่ายมากจนอาจคิดว่า "ไม่มีอะไรต้อง
รีแฟกเตอร์" — แต่ลองมองไปข้างหน้า: แบบฝึกหัดต่อยอดท้าย Part นี้ (ดู Step 500) จะขอให้เพิ่ม "ส่ง
อีเมลแจ้งผู้แสดงความเห็นเมื่อได้รับการอนุมัติ" — ถ้า "การอนุมัติ" ยังเป็นแค่ `@comment.approve!`
บรรทัดเดียวอยู่ใน Controller เมื่อไหร่ที่ต้องเพิ่ม side effect (ส่งอีเมล, แจ้งเตือน, log กิจกรรม)
โค้ดนั้นจะเริ่มไหลเข้าไปอยู่ใน Controller ซึ่งควรมีหน้าที่แค่ "รับ request แล้วสั่งงาน" ไม่ใช่
"รู้วิธีทำงานทั้งหมด"

จึงสกัดพฤติกรรม "การตรวจสอบความเห็น" ออกมาเป็น **Service Object** — Ruby class ธรรมดา ไม่ต้องพึ่ง
Rails magic ใดๆ (แนวคิดเดียวกับ `PasswordStrength` ใน Step 491 หรือ `TaskList` จาก Todo CLI ใน
Part 020 — Service Object แบบเต็มรูปแบบพร้อม pattern อื่นๆ จะเรียนเจาะลึกใน Part 082 แต่หลักการ
พื้นฐานที่สุดคือ "ดึง logic ที่ไม่ใช่หน้าที่ของ Controller ออกมาเป็น class เดี่ยวๆ ที่ทดสอบแยกได้"
ซึ่งเราทำได้ตั้งแต่ตอนนี้โดยไม่ต้องรอเรียนบทที่ว่าด้วย pattern):

```ruby
# app/services/comment_moderation_service.rb
# รวม "การกระทำเชิงพฤติกรรม" ของการตรวจสอบความเห็นไว้ในที่เดียว แยกออกจาก Controller
# วันนี้ยังดูเหมือนเป็นแค่ wrapper บางๆ รอบ comment.approve!/reject! สองบรรทัด แต่จุดประสงค์
# คือเปิดที่ว่างสำหรับ "ผลข้างเคียง" ที่กำลังจะตามมาแน่ๆ ในอนาคตอันใกล้ โดยไม่ต้องแตะ
# Controller อีกเลยสักครั้งเดียว
class CommentModerationService
  def initialize(comment)
    @comment = comment
  end

  def approve!
    comment.approve!
  end

  def reject!
    comment.reject!
  end

  private

  attr_reader :comment
end
```

```ruby
# app/controllers/comments_controller.rb (แก้ 2 บรรทัด)
def approve
  authorize @comment
  CommentModerationService.new(@comment).approve!
  redirect_to post_path(@comment.post), notice: "อนุมัติความเห็นแล้ว"
end

def reject
  authorize @comment
  CommentModerationService.new(@comment).reject!
  redirect_to post_path(@comment.post), notice: "ปฏิเสธความเห็นแล้ว"
end
```

สังเกตว่า **ไม่มีการเขียนเทสต์ใหม่แม้แต่ตัวเดียวสำหรับการรีแฟกเตอร์รอบนี้** — นี่คือหลักการสำคัญ
ของขั้นตอน Refactor ใน RGR: **ไม่มี behavior ใหม่เกิดขึ้น จึงไม่ต้องมีเทสต์ใหม่** ชุดทดสอบเดิมทั้ง
22 examples คือ safety net ที่ต้องพิสูจน์ว่าการย้ายโค้ดครั้งนี้ไม่ได้ทำให้อะไรพัง รันทั้งชุดซ้ำ:

```bash
bundle exec rspec
```

```
...

Finished in 1.44 seconds (files took 1.57 seconds to load)
22 examples, 0 failures
```

**22 examples, 0 failures — จำนวนเท่าเดิมเป๊ะ** ก่อนรีแฟกเตอร์ก็ 22 examples ผ่านหมด หลังย้าย
โค้ดไปอยู่ใน service object ก็ยัง 22 examples ผ่านหมดเหมือนเดิม **นี่คือหลักฐานที่จับต้องได้ว่า
"safety net" ของ TDD ทำงานจริง** — ถ้าการรีแฟกเตอร์ทำให้ behavior เปลี่ยนไปแม้แต่นิดเดียว (เช่น
ลืมเรียก `save`, สลับลำดับ argument ผิด) จะมีเทสต์อย่างน้อยหนึ่งตัวในชุดนี้แดงขึ้นมาทันที

---

## Step 499: เติม FactoryBot Factories ย้อนหลัง เพื่อล้าง Setup ของทุก Spec

สังเกตว่าตลอด Step 492–497 ทุก spec เขียน setup ด้วย `User.create!(name: ..., email: ...,
password: ...)` และ `Post.create!(title: ..., body: ..., user: ...)` ซ้ำกันแทบทุกไฟล์ — ตอนที่
ยังโฟกัสที่ "ทำให้ RED กลายเป็น GREEN ให้เร็วที่สุด" การเขียนแบบตรงไปตรงมานี้ถูกต้องแล้ว (อย่าง
ที่ Step 491 บอกไว้ว่า GREEN คือ "โค้ดน้อยที่สุดเท่าที่จะทำให้ผ่าน" — และนี่ใช้กับโค้ดฝั่งเทสต์
ด้วยเช่นกัน) แต่ตอนนี้ฟีเจอร์เสร็จสมบูรณ์แล้ว เป็นจังหวะที่เหมาะจะ **รีแฟกเตอร์ฝั่งเทสต์** ด้วย
FactoryBot/Faker (Part 047) ที่มีอยู่แล้วในโปรเจกต์

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { Faker::Lorem.sentence(word_count: 4) }
    body { Faker::Lorem.paragraph(sentence_count: 3) }
    association :user
  end
end
```

```ruby
# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { Faker::Lorem.sentence }
    association :post
    association :user

    # trait ทำให้สร้างความเห็นที่ "ผ่านสถานะมาแล้ว" ได้ในบรรทัดเดียว แทนที่จะต้อง create
    # แล้วเรียก .approve!/.reject! ต่ออีกบรรทัดทุกครั้งที่ต้องใช้ในเทสต์
    trait :approved do
      status { :approved }
    end

    trait :rejected do
      status { :rejected }
    end
  end
end
```

แล้วไล่แก้ทุก spec ให้ใช้ factory แทน ตัวอย่างเช่น `spec/models/comment_spec.rb`:

```ruby
# ก่อนรีแฟกเตอร์
let(:user) { User.create!(name: "ผู้แสดงความเห็น", email: "commenter@example.com", password: "password123") }
let(:author) { User.create!(name: "ผู้เขียน", email: "author2@example.com", password: "password123") }
let(:post_record) { Post.create!(title: "บทความ", body: "เนื้อหา", user: author) }

# ...
approved = described_class.create!(post: post_record, user: user, body: "อนุมัติแล้ว").tap(&:approve!)
pending = described_class.create!(post: post_record, user: user, body: "รอตรวจสอบ")
rejected = described_class.create!(post: post_record, user: user, body: "ถูกปฏิเสธ").tap(&:reject!)
```

```ruby
# หลังรีแฟกเตอร์
let(:user) { create(:user) }
let(:author) { create(:user) }
let(:post_record) { create(:post, user: author) }

# ...
approved = create(:comment, :approved, post: post_record, user: user)
pending = create(:comment, post: post_record, user: user)
rejected = create(:comment, :rejected, post: post_record, user: user)
```

สั้นลงชัดเจน และ **สื่อเจตนาได้ตรงกว่าเดิม** — `create(:comment, :approved, ...)` อ่านออกเสียง
ได้ทันทีว่า "สร้างความเห็นที่อนุมัติแล้ว" ในขณะที่ `.create!(...).tap(&:approve!)` ต้องอ่านสอง
รอบกว่าจะเข้าใจ ทำแบบเดียวกันกับ `spec/requests/comments_spec.rb`,
`spec/requests/comment_moderations_spec.rb`, และ `spec/system/comment_moderation_spec.rb`
(เปลี่ยนทุก `User.create!(...)`/`Post.create!(...)`/`Comment.create!(...)` เป็น
`create(:user)`/`create(:post, ...)`/`create(:comment, ...)` ตามลำดับ)

รันชุดทดสอบทั้งหมดอีกครั้งหลังแก้ทุกไฟล์:

```bash
bundle exec rspec
```

```
Comment
  associations
    belongs_to :post
    belongs_to :user
  validations
    ไม่ valid เมื่อ body ว่างเปล่า
    valid เมื่อข้อมูลครบถ้วน
  สถานะการตรวจสอบ (moderation state machine)
    มีสถานะ pending เป็นค่าเริ่มต้นเสมอเมื่อสร้างใหม่
    #approve! เปลี่ยนสถานะเป็น approved และบันทึกลงฐานข้อมูลทันที
    #reject! เปลี่ยนสถานะเป็น rejected และบันทึกลงฐานข้อมูลทันที
    .visible
      คืนเฉพาะความเห็นที่ approved เท่านั้น ไม่รวม pending หรือ rejected

Comment moderation
  PATCH /posts/:post_id/comments/:id/approve
    เมื่อเป็นเจ้าของบทความ
      อนุมัติความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login
  PATCH /posts/:post_id/comments/:id/reject
    เมื่อเป็นเจ้าของบทความ
      ปฏิเสธความเห็นสำเร็จ
    เมื่อไม่ใช่เจ้าของบทความ (แม้จะเป็นคนโพสต์ความเห็นเองก็ตาม)
      ปฏิเสธการเข้าถึง และไม่เปลี่ยนสถานะความเห็น

Comments
  POST /posts/:post_id/comments
    เมื่อเข้าสู่ระบบแล้ว
      สร้างความเห็นใหม่ผูกกับ post และ user ที่ล็อกอินอยู่
      redirect กลับไปหน้า post หลังสร้างสำเร็จ
      ไม่สร้างความเห็นเมื่อ body ว่างเปล่า และแสดง error กลับมา
    เมื่อยังไม่เข้าสู่ระบบ
      redirect ไปหน้า login แทนการสร้างความเห็น

PasswordStrength
  .score
    คืนค่า :weak เมื่อรหัสผ่านสั้นกว่า 8 ตัวอักษร
    คืนค่า :medium เมื่อยาวพอแต่มีแต่ตัวพิมพ์เล็ก
    คืนค่า :strong เมื่อยาวพอและมีทั้งพิมพ์เล็ก พิมพ์ใหญ่ ตัวเลข และอักขระพิเศษ

Comment moderation flow
  ผู้เขียนบทความเห็นความเห็นที่รอตรวจสอบ อนุมัติได้ แล้วความเห็นนั้นแสดงต่อสาธารณะทันที
  เจ้าของบทความปฏิเสธความเห็นได้ และความเห็นจะไม่แสดงต่อสาธารณะ

Finished in 1.41 seconds (files took 1.39 seconds to load)
22 examples, 0 failures
```

**22 examples, 0 failures — เท่าเดิมอีกครั้ง** เปลี่ยนวิธีสร้างข้อมูลทดสอบทั้งระบบจากการเรียก
`.create!` ตรงๆ ไปเป็น FactoryBot ทั้งหมด โดยที่จำนวนเทสต์และผลลัพธ์ไม่เปลี่ยนแปลงแม้แต่ตัวเดียว
— นี่คือการรีแฟกเตอร์ **ฝั่งเทสต์เอง** ซึ่งเป็นสิ่งที่ TDD สนับสนุนเช่นกัน (โค้ดเทสต์ก็คือโค้ด
ต้องดูแลรักษาให้สะอาดเหมือนโค้ด production เช่นกัน — ทบทวนหลักการนี้จาก Part 047)

---

## Step 500: รัน Full Suite พร้อม SimpleCov Coverage Report เต็มรูปแบบ + สรุปบทเรียนของ TDD

### รันชุดทดสอบทั้งหมดครั้งสุดท้าย พร้อมดู Coverage

```bash
bundle exec rspec
```

ผลลัพธ์เหมือนเดิมทุกประการ (**22 examples, 0 failures**) แต่คราวนี้ไม่ตัดบรรทัดท้ายออกแล้ว —
มาดู SimpleCov output เต็มๆ ตามที่ตั้งค่าไว้ตั้งแต่ Step 491:

```
Coverage report generated for RSpec to .../coverage.
Line Coverage: 89.78% (123 / 137)
```

**89.78%** คือ % ของบรรทัดโค้ดใน `app/` ที่ถูกรันผ่านอย่างน้อยหนึ่งครั้งระหว่างรันชุดทดสอบ — แต่
ตัวเลขรวมตัวเดียวบอกอะไรได้ไม่มาก เปิดดูรายละเอียดต่อไฟล์จาก `coverage/index.html` (ทบทวนวิธีอ่าน
รายงานนี้จาก Part 049) จะเห็นรายละเอียดประมาณนี้:

| ไฟล์ | บรรทัดที่ทดสอบ | รวม | % |
|---|---|---|---|
| `app/controllers/application_controller.rb` | 16 | 16 | **100.00%** |
| `app/controllers/comments_controller.rb` | 23 | 23 | **100.00%** |
| `app/controllers/posts_controller.rb` | 8 | 8 | **100.00%** |
| `app/controllers/sessions_controller.rb` | 8 | 12 | 66.67% |
| `app/models/comment.rb` | 10 | 10 | **100.00%** |
| `app/models/post.rb` | 5 | 5 | **100.00%** |
| `app/models/user.rb` | 6 | 6 | **100.00%** |
| `app/policies/application_policy.rb` | 17 | 27 | 62.96% |
| `app/policies/comment_policy.rb` | 8 | 8 | **100.00%** |
| `app/services/comment_moderation_service.rb` | 9 | 9 | **100.00%** |
| `app/services/password_strength.rb` | 10 | 10 | **100.00%** |

ทุกไฟล์ที่เขียนขึ้นมา **ในฐานะส่วนหนึ่งของ TDD cycle ของ Part นี้** (`comment.rb`,
`comments_controller.rb`, `posts_controller.rb`, `comment_policy.rb`,
`comment_moderation_service.rb`) มี coverage **100%** ทั้งหมด — นี่ไม่ใช่เรื่องบังเอิญ แต่เป็นผล
ธรรมชาติของ TDD: **เพราะไม่มีบรรทัดโค้ดไหนถูกเขียนขึ้นมาโดยไม่มีเทสต์เรียกร้องมันก่อน จึงไม่มี
บรรทัดไหนที่ "ไม่ถูกทดสอบ" หลงเหลืออยู่เลย** ส่วนสองไฟล์ที่ coverage ต่ำกว่า 100% —
`sessions_controller.rb` (66.67%) และ `application_policy.rb` (62.96%) — คือไฟล์ **จาก
โครงสร้างพื้นฐานเดิม** (Part 041, Part 043) ที่ **ไม่ได้ถูก TDD ใน Part นี้**: บาง action ของ
`SessionsController` (เช่น กรณี login ผิดพลาด, logout) ไม่เคยถูกเทสต์ในฟีเจอร์นี้เพราะ
`login_as` helper สมมติว่า login สำเร็จเสมอ และ `ApplicationPolicy` (base class) มี method
default หลายตัว (`index?`, `create?`, `edit?`, ฯลฯ) ที่ไม่มี policy ตัวไหนใน Part นี้เรียกใช้เลย
— **SimpleCov เพิ่งช่วยเราค้นพบว่ามีจุดที่ควรกลับไปเติมเทสต์ในอนาคต** ซึ่งเป็นการใช้งาน coverage
ที่ถูกต้องตามที่ Part 049 สอนไว้: coverage ไม่ใช่เป้าหมาย แต่เป็น **เครื่องมือชี้จุดบอด**

### สรุปสิ่งที่ TDD ช่วยได้จริง และสิ่งที่มันไม่ได้ช่วยมากในแบบฝึกหัดนี้ (ถอดบทเรียนอย่างตรงไปตรงมา)

ตลอด 5 TDD cycle ที่ผ่านมา ควรถอดบทเรียนตามความเป็นจริง ไม่ใช่แค่พูดว่า "TDD ดีเสมอ":

**จุดที่ TDD ช่วยได้ชัดเจนมาก:**

- **State machine (Step 495)** — การเขียนเทสต์ `#approve!`/`#reject!`/`.visible` ก่อน บังคับให้
  คิดครบทุก transition (`pending → approved`, `pending → rejected`) และกรณี query
  (`.visible` ต้องไม่รวม `pending`/`rejected`) **ก่อน**ที่จะเขียน `enum` แม้แต่บรรทัดเดียว ถ้า
  เขียนโค้ดก่อนแล้วค่อยทดสอบทีหลัง มีโอกาสสูงที่จะลืมเทสต์กรณี `rejected` ไม่ถูกกรองออกจาก
  `.visible` เพราะตอนเขียนโค้ดมักโฟกัสแค่ "ทำให้ approved แสดงได้" แล้วลืมยืนยันว่า "สถานะอื่น
  ต้องไม่แสดง"
- **Authorization (Step 496)** — การไล่เขียนเทสต์ทีละบทบาท (เจ้าของ/คนอื่น/ยังไม่ล็อกอิน) **ก่อน**
  เขียน `CommentPolicy` ทำให้เขียน policy ถูกตั้งแต่ครั้งแรกโดยไม่ต้องวนแก้ และจับ edge case
  "เจ้าของความเห็นเองก็ไม่มีสิทธิ์ปฏิเสธความเห็นตัวเอง" ได้ตั้งแต่ตอนคิดเทสต์ ก่อนที่จะกลาย
  เป็นช่องโหว่ด้าน security ใน production
- **การรีแฟกเตอร์ (Step 498–499)** — พิสูจน์ได้อย่างเป็นรูปธรรมด้วยตัวเลข **22 examples ทั้ง
  ก่อนและหลัง** ทั้งตอนสกัด service object และตอนเปลี่ยนไปใช้ FactoryBot — เป็นการยืนยันด้วย
  ข้อมูลจริง ไม่ใช่ความรู้สึกว่า "น่าจะไม่พัง"

**จุดที่ TDD ช่วยได้น้อยกว่าที่คาด (บอกตามตรง):**

- **CRUD พื้นฐานอย่าง Step 492–493 (สร้าง model เปล่าๆ พร้อม `belongs_to`/`validates`)** —
  ความแตกต่างระหว่าง "เขียนเทสต์ก่อน" กับ "เขียนโค้ดก่อนแล้วเขียนเทสต์ตาม" แทบไม่มีนัยสำคัญเลย
  ในกรณีนี้ เพราะไม่มี logic ที่ซับซ้อนพอจะทำให้แนวทางไหนแนวทางหนึ่งเผยจุดบอดของอีกฝั่ง — โค้ด
  `belongs_to :post` กับ `validates :body, presence: true` เขียนถูกตั้งแต่ครั้งแรกไม่ว่าจะ
  เขียนเทสต์ก่อนหรือหลัง เพราะมันเป็น Rails convention ที่ตรงไปตรงมาอยู่แล้ว
- **View/UI ธรรมดาอย่าง form และ list (ส่วนหนึ่งของ Step 494, 497)** — Request/System spec
  ช่วยยืนยันว่า flow ทำงาน แต่ **รายละเอียดหน้าตา** (จัดวาง CSS, ข้อความ label) ไม่ใช่สิ่งที่
  TDD ควรถูกใช้บังคับ — การไล่เขียนเทสต์ทีละ pixel ไม่มีประโยชน์และเสียเวลาเปล่า จุดสมดุลที่ดี
  คือทดสอบ **behavior ที่จับต้องได้** (element ไหนมีอยู่/ไม่มีอยู่, ข้อความไหนปรากฏ/ไม่ปรากฏ)
  แต่ไม่ทดสอบรายละเอียดการจัดวางหน้าตา

ข้อสรุปที่ตรงไปตรงมาที่สุด: **TDD คุ้มค่าที่สุดเมื่อมี logic ที่ "ผิดพลาดได้ง่ายถ้าคิดไม่ครบ"**
— state machine, authorization rule, การคำนวณที่ซับซ้อน, edge case ที่ไม่ชัดเจน — ส่วนโค้ดที่
เป็น boilerplate ตรงไปตรงมา (CRUD พื้นฐาน, การจัดวาง UI) เขียนเทสต์คลุมไว้เพื่อป้องกันการพังใน
อนาคตก็ยังจำเป็น แต่ **ลำดับ** ว่าจะเขียนเทสต์ก่อนหรือหลังไม่ใช่ปัจจัยชี้ขาดคุณภาพเท่ากับกรณีแรก
มืออาชีพที่ใช้ TDD จริงมักไม่ได้ทำ 100% ทุกบรรทัดของทุกโปรเจกต์ แต่เลือกใช้อย่างเข้มข้นตรงจุดที่
ความเสี่ยงสูงและ logic ซับซ้อน แล้วผ่อนลงในจุดที่ตรงไปตรงมา — **นี่คือวิจารณญาณที่สำคัญกว่าการ
ท่องจำว่า "ต้องเขียนเทสต์ก่อนเสมอ" แบบไม่มีข้อยกเว้น**

### แบบฝึกหัดต่อยอด (ทำเอง — ไม่มีเฉลย)

ลองนำวินัย Red-Green-Refactor ที่เพิ่งฝึกไปใช้กับฟีเจอร์ใหม่ต่อไปนี้ในโปรเจกต์เดิม โดย**ต้องเขียน
เทสต์ก่อนเสมอ**ทุกครั้ง ห้ามเขียนโค้ด production ก่อนมีเทสต์สีแดง:

1. **TDD ระบบส่งอีเมลแจ้งเตือนเมื่อความเห็นได้รับการอนุมัติ** — ใช้ Action Mailer สร้าง
   `CommentMailer#approved_notification(comment)` ที่ส่งหาอีเมลของ `comment.user` เขียน
   request spec หรือ spec ของ `CommentModerationService` (ที่สกัดไว้แล้วใน Step 498) ที่ยืนยันว่า
   เมื่อ `approve!` ถูกเรียก mailer ต้องถูกเรียกด้วย (ใช้ `have_enqueued_mail` matcher หรือ
   mock/stub ตามที่เรียนใน Part 049 — คิดดูว่าจะ mock ตรงจุดไหนถึงจะไม่ทำให้เทสต์ช้าลงเพราะส่ง
   อีเมลจริงทุกครั้งที่รัน)
2. **TDD ข้อจำกัดช่วงเวลาแก้ไขความเห็น** — ผู้แสดงความเห็นแก้ไขความเห็นของตัวเองได้ แต่**ภายใน
   15 นาทีหลังสร้างเท่านั้น** (หลังจากนั้นต้องแสดง error) ออกแบบเทสต์ที่ครอบคลุมทั้งกรณี "ยังอยู่
   ในช่วงเวลา" และ "เลยเวลาไปแล้ว" — ต้องคิดวิธีทดสอบเรื่องเวลาที่ผ่านไปโดยไม่ต้อง `sleep` จริง
   ในเทสต์ (ใบ้: `travel_to`/`travel` จาก ActiveSupport::Testing::TimeHelpers)
3. **TDD การจำกัดจำนวนความเห็นต่อผู้ใช้ต่อบทความ (rate limiting เบื้องต้น)** — ผู้ใช้แต่ละคนแสดง
   ความเห็นใต้บทความเดียวกันได้ไม่เกิน 5 ครั้ง (ป้องกันสแปม) เขียน model spec ที่ทดสอบทั้งกรณี
   "ยังไม่ถึงขีดจำกัด" และ "ถึงขีดจำกัดแล้ว" ก่อน แล้วค่อยเขียน custom validation ให้ผ่าน — ลอง
   สังเกตดูว่าการเขียนเทสต์ก่อนช่วยให้คิด edge case (เช่น นับเฉพาะความเห็นที่ไม่ถูก reject
   หรือไม่?) ได้ชัดเจนกว่าการเขียนโค้ดก่อนหรือไม่

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจวงจร **Red-Green-Refactor** อย่างละเอียด และเหตุผลเชิงลึกว่าทำไม "เขียนเทสต์ก่อน" ถึง
  ผลักดันให้เกิดการออกแบบที่ทดสอบง่ายกว่า "เขียนเทสต์ทีหลัง" — ผ่านการเปรียบเทียบโดยตรงและผ่าน
  kata `PasswordStrength` ที่สาธิตวงจรเต็มรูปแบบ (RED → GREEN แบบ "Fake it" → RED แบบ
  triangulation → GREEN แบบ generalize → REFACTOR)
- สร้างฟีเจอร์ **Comment Moderation** ทั้งฟีเจอร์ตั้งแต่ศูนย์ผ่าน TDD ล้วนๆ 5 cycle: model +
  validation (Step 492–493), request spec สำหรับ nested route (Step 494), moderation state
  machine ด้วย `enum` (Step 495), authorization ด้วย Pundit (Step 496), และ system test เต็ม
  รูปแบบด้วย Capybara (Step 497) — ทุก cycle มี RED (ผลลัพธ์ terminal จริงที่แดง) ตามด้วย GREEN
  (ผลลัพธ์จริงที่เขียว) เสมอ
- ฝึกแยกแยะว่า "REFACTOR" ไม่จำเป็นต้องเกิดทุก cycle (Step 493 ไม่มีอะไรต้องแก้) และไม่ได้จำกัด
  แค่การจัดโค้ดให้สวย แต่รวมถึงการแก้ deprecation warning (Step 494) และการสกัด logic ออกมาเป็น
  class ใหม่เพื่อรองรับอนาคต (Step 498)
- พิสูจน์ด้วยตัวเลขจริง (22 examples, 0 failures คงที่) ว่า **ชุดทดสอบที่มีอยู่คือ safety net
  ที่ทำให้รีแฟกเตอร์ได้อย่างมั่นใจ** ทั้งตอนสกัด `CommentModerationService` (Step 498) และตอน
  เปลี่ยนไปใช้ FactoryBot factories ย้อนหลัง (Step 499)
- อ่านและตีความ **SimpleCov coverage report แบบละเอียดต่อไฟล์** (Step 500) พบว่าทุกไฟล์ที่ถูก
  สร้างผ่าน TDD cycle มี coverage 100% โดยธรรมชาติ ในขณะที่ไฟล์เดิมที่ไม่ได้ผ่าน TDD ใน Part นี้
  (`sessions_controller.rb`, `application_policy.rb`) มี coverage ต่ำกว่า — และเข้าใจว่าตัวเลข
  coverage มีไว้ **ชี้จุดบอด** ไม่ใช่เป้าหมายที่ต้องไล่ให้ถึง 100% ทุกไฟล์
- ถอดบทเรียนอย่างตรงไปตรงมาว่า TDD **ทรงพลังที่สุดกับ logic ที่ซับซ้อนและเสี่ยงต่อการคิดไม่ครบ**
  (state machine, authorization) แต่ **ให้ประโยชน์เพิ่มเติมไม่มากนัก** กับ CRUD/boilerplate
  ตรงไปตรงมา — เป็นวิจารณญาณสำคัญสำหรับการใช้ TDD ในงานจริงที่ไม่มีเวลาไม่จำกัด

## สรุปภาพรวม Phase 6: Testing (TDD/BDD)

ยินดีด้วย! ตอนนี้ **Phase 6: Testing (TDD/BDD) (Part 046–050, Step 451–500)** เสร็จสมบูรณ์แล้ว
เราเดินทางจากการเขียน RSpec สำหรับ Rails โดยเฉพาะ — model spec และ request spec (Part 046) —
ไปจนถึงการจัดการข้อมูลทดสอบอย่างเป็นระบบด้วย FactoryBot และ Faker แทนการสร้างข้อมูลด้วยมือทุกครั้ง
(Part 047), การจำลองพฤติกรรมผู้ใช้จริงในเบราว์เซอร์ด้วย Capybara System Test (Part 048), การ
ควบคุม dependency ภายนอกด้วย mocking/stubbing และ VCR พร้อมวัดความครอบคลุมของโค้ดด้วย SimpleCov
(Part 049) จนมาถึง Part นี้ที่ประกอบทุกเครื่องมือเหล่านั้นเข้าด้วยกันเป็น **วินัยการทำงานเดียว**
ที่ใช้ได้จริงในงานประจำวัน — Red-Green-Refactor

สิ่งที่สำคัญที่สุดที่ควรพกติดตัวไปจาก Phase นี้ไม่ใช่ "รู้จัก syntax ของ RSpec" (แม้จะสำคัญ) แต่
คือ **ความมั่นใจที่มาจากการมีชุดทดสอบที่เชื่อถือได้** — Part 050 นี้แสดงให้เห็นชัดเจนที่สุดว่า
เมื่อมีเทสต์ที่ครอบคลุมพฤติกรรมจริง การรีแฟกเตอร์ (ไม่ว่าจะสกัด service object หรือเปลี่ยนวิธี
สร้างข้อมูลทดสอบ) กลายเป็นเรื่องที่ทำได้อย่างสบายใจ ไม่ใช่เรื่องที่ต้องกลัว — และทักษะทั้งหมดใน
Phase นี้จะติดตัวไปใช้ **ทุก Phase ที่เหลือของหลักสูตร** ตั้งแต่นี้ไป ไม่ว่าจะเป็น Turbo/Stimulus
(Phase 7), API/GraphQL (Phase 8), Background Jobs (Phase 9) ไปจนถึง Capstone Project สุดท้าย
(Phase 16) — ทุกฟีเจอร์ใหม่ที่เขียนต่อจากนี้ควรถูกสร้างผ่านวินัย Red-Green-Refactor แบบเดียวกับ
ที่เพิ่งฝึกใน Part นี้

**ต่อไป (Part 051 — เปิด Phase 7: Frontend / Hotwire / Stimulus):** ถึงเวลาทำให้เว็บแอปที่เรา
สร้างมาตลอด 5 Phase รู้สึก "เร็วและลื่นไหล" แบบ Single Page Application โดยไม่ต้องเขียน
JavaScript framework หนักๆ เลยแม้แต่บรรทัดเดียว — เราจะเรียนรู้ **Turbo Drive** ที่ทำให้ทุก
`link_to`/`form_with` ในแอปกลายเป็น AJAX request โดยอัตโนมัติแบบไม่ต้องแก้โค้ดเดิมเลย และ
**Turbo Frames** ที่ทำให้อัปเดตแค่บางส่วนของหน้าเว็บได้โดยไม่ต้อง reload ทั้งหน้า — ฟีเจอร์
Comment Moderation ที่เพิ่งสร้างเสร็จใน Part นี้จะถูกหยิบกลับมาใช้เป็นตัวอย่างจริงในการทำให้ปุ่ม
"อนุมัติ"/"ปฏิเสธ" อัปเดตหน้าเว็บแบบไม่ต้อง reload อีกต่อไป
