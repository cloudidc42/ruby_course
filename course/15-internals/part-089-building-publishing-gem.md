# Part 089: การสร้างและ Publish Gem ของตัวเองไปยัง RubyGems

> **Step ครอบคลุมใน Part นี้:** Step 881–890
> **ระดับ:** Intermediate–Advanced
> **Ruby Version:** 3.3.6
> **Bundler Version:** 2.5.x

ใน Part 088 เราได้เรียนรู้เรื่อง Rails Engine ซึ่งเป็นการแยกโมดูลของแอปพลิเคชัน Rails ออกเป็นส่วนย่อยที่ mount ได้ ใน Part นี้เราจะก้าวไปอีกขั้น โดยเรียนรู้การสร้าง **Ruby Gem** ของตัวเองตั้งแต่ต้น ตั้งแต่การออกแบบโครงสร้าง การเขียนโค้ด การทดสอบ ไปจนถึงการ publish ไปยัง RubyGems.org หรือ private gem server สำหรับใช้ภายใน organization Gem เป็นหน่วยที่เล็กกว่า Engine แต่มีความยืดหยุ่นสูง ใช้ได้กับโปรเจกต์ Ruby ทุกประเภท ไม่จำกัดแค่ Rails

---

## สารบัญ

- [Step 881: ทำไมต้องสร้าง Gem ของตัวเอง](#step-881)
- [Step 882: สร้าง Gem Skeleton ด้วย Bundler](#step-882)
- [Step 883: เขียนโค้ดหลักของ Gem](#step-883)
- [Step 884: .gemspec เชิงลึก](#step-884)
- [Step 885: Test Gem ด้วย RSpec](#step-885)
- [Step 886: Rake Tasks สำหรับ Gem](#step-886)
- [Step 887: Publish ไปยัง RubyGems.org](#step-887)
- [Step 888: Versioning Strategy](#step-888)
- [Step 889: Private Gem Server](#step-889)
- [Step 890: Maintain Gem อย่างมืออาชีพ](#step-890)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## Setup

ก่อนเริ่มต้น ตรวจสอบว่าเครื่องมือพร้อมใช้งาน:

```bash
# ตรวจสอบ Ruby version
ruby -v
# ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]

# ตรวจสอบ Bundler version
bundler -v
# Bundler version 2.5.23

# ตรวจสอบ RubyGems version
gem -v
# 3.5.23

# ติดตั้ง bundler ถ้ายังไม่มี
gem install bundler

# ติดตั้ง rake สำหรับ gem tasks
gem install rake
```

ตัวอย่างใน Part นี้จะสร้าง gem ชื่อ `thai_formatter` ซึ่งเป็น utility สำหรับจัดรูปแบบข้อความภาษาไทย เช่น ตัวเลขเป็นคำอ่าน และการจัดรูปแบบวันที่แบบไทย

---

## Step 881: ทำไมต้องสร้าง Gem ของตัวเอง {#step-881}

### แนวคิด

Ruby Gem คือหน่วยของโค้ดที่บรรจุ library, utility, หรือ tool ที่สามารถแจกจ่ายและติดตั้งซ้ำได้ด้วย `gem install` หรือเพิ่มใน `Gemfile` เหตุผลที่นักพัฒนาสร้าง gem ของตัวเองมีหลายกรณี

### เหตุผลหลัก 3 ข้อ

**1. Code Reuse ข้ามโปรเจกต์**

เมื่อมีโค้ดที่ต้องการใช้ซ้ำในหลายโปรเจกต์ การแปลงเป็น gem ช่วยให้:
- จัดการ dependency ได้ในที่เดียว
- อัปเดตครั้งเดียว ทุกโปรเจกต์ได้รับการแก้ไข
- มี version control ที่ชัดเจน

```ruby
# แทนที่จะ copy-paste โค้ดนี้ทุกโปรเจกต์
module ThaiFormatter
  def self.number_to_thai_baht(amount)
    "฿#{format('%.2f', amount)}"
  end
end

# แค่เพิ่ม gem ใน Gemfile
# gem 'thai_formatter', '~> 1.0'
```

**2. Open Source และ Community**

การ publish gem เป็น open source ช่วยให้:
- ชุมชน Ruby ได้รับประโยชน์จากงานของคุณ
- รับ feedback และ contribution จากคนอื่น
- สร้าง portfolio และชื่อเสียงในชุมชน

**3. Private Gem สำหรับ Organization**

บริษัทหลายแห่งสร้าง private gem สำหรับ:
- Shared business logic ที่ใช้ร่วมกันระหว่างทีม
- Internal tools ที่ไม่ควรเปิดสาธารณะ
- Standard library ของ organization เช่น authentication, logging

```
โครงสร้างทั่วไปของ Organization ที่ใช้ Private Gems:

organization-auth-gem    ← authentication logic
    ↓
organization-api-gem     ← API client
    ↓
app-backend              ← แอปหลักที่ใช้ทั้งสอง gem
app-frontend-api         ← อีกแอปที่ใช้ organization-api-gem
```

### เปรียบเทียบ: Gem vs Concerns vs Engine

| วิธีการ | ใช้เมื่อ | ขอบเขต |
|--------|---------|--------|
| Concerns | โค้ดซ้ำใน model/controller เดียวกัน | ภายในโปรเจกต์เดียว |
| Rails Engine | mount ทั้ง Rails sub-app | เฉพาะโปรเจกต์ Rails |
| Ruby Gem | code library ทั่วไป | ทุก Ruby project |

---

## Step 882: สร้าง Gem Skeleton ด้วย Bundler {#step-882}

### คำสั่ง `bundle gem`

Bundler มีคำสั่งที่สร้างโครงสร้าง gem ให้อัตโนมัติ:

```bash
# สร้าง gem ใหม่
bundle gem thai_formatter

# Bundler จะถามตัวเลือกต่างๆ:
# Do you want to generate tests with your gem?
# Enter a test framework. rspec/minitest/test-unit/(none): rspec
#
# Do you want to set up continuous integration for your gem?
# Enter a CI service. github/gitlab/circle/(none): github
#
# Do you want to license your gem permissively under the MIT license?
# y/n: y
#
# Do you want to include a code of conduct in gems you generate?
# y/n: y
#
# Do you want to include a changelog?
# y/n: y
```

### โครงสร้างไฟล์ที่สร้างขึ้น

```
thai_formatter/
├── .github/
│   └── workflows/
│       └── main.yml          ← CI configuration
├── lib/
│   ├── thai_formatter/
│   │   └── version.rb        ← VERSION constant
│   └── thai_formatter.rb     ← entry point หลัก
├── spec/
│   ├── spec_helper.rb        ← RSpec configuration
│   └── thai_formatter_spec.rb
├── .gitignore
├── .rspec
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── Gemfile                   ← สำหรับ development
├── LICENSE.txt
├── README.md
├── Rakefile
└── thai_formatter.gemspec    ← metadata และ configuration หลัก
```

### ดูไฟล์แต่ละส่วน

**`lib/thai_formatter/version.rb`** — VERSION constant:

```ruby
# lib/thai_formatter/version.rb
# frozen_string_literal: true

module ThaiFormatter
  VERSION = "0.1.0"
end
```

**`lib/thai_formatter.rb`** — entry point หลัก:

```ruby
# lib/thai_formatter.rb
# frozen_string_literal: true

require_relative "thai_formatter/version"

module ThaiFormatter
  class Error < StandardError; end
  # Your code goes here...
end
```

**`Gemfile`** — ใช้ระหว่าง development:

```ruby
# Gemfile
# frozen_string_literal: true

source "https://rubygems.org"

# Specify your gem's dependencies in thai_formatter.gemspec
gemspec
```

ข้อสังเกต: `Gemfile` ของ gem อ้างอิงไปยัง `.gemspec` แทนที่จะระบุ dependencies โดยตรง เพราะ dependencies จริงอยู่ใน `.gemspec`

### ติดตั้ง dependencies

```bash
cd thai_formatter
bundle install
```

---

## Step 883: เขียนโค้ดหลักของ Gem {#step-883}

### วางแผนโครงสร้าง Module

ก่อนเขียนโค้ด วางแผน namespace และ class structure:

```
ThaiFormatter                    ← main module
├── ThaiFormatter::Number        ← จัดการตัวเลข
│   ├── .to_baht(amount)
│   └── .to_thai_words(number)
├── ThaiFormatter::Date          ← จัดการวันที่
│   ├── .to_thai_date(date)
│   └── .thai_era(year)
└── ThaiFormatter::Text          ← จัดการข้อความ
    └── .normalize(text)
```

### สร้างไฟล์โครงสร้าง

```bash
mkdir -p lib/thai_formatter
touch lib/thai_formatter/number.rb
touch lib/thai_formatter/date.rb
touch lib/thai_formatter/text.rb
```

### เขียน `VERSION` constant

```ruby
# lib/thai_formatter/version.rb
# frozen_string_literal: true

module ThaiFormatter
  # Version ตาม Semantic Versioning (SemVer)
  # Major.Minor.Patch
  VERSION = "0.1.0"
end
```

### เขียน main entry point

```ruby
# lib/thai_formatter.rb
# frozen_string_literal: true

require_relative "thai_formatter/version"
require_relative "thai_formatter/number"
require_relative "thai_formatter/date"
require_relative "thai_formatter/text"

# ThaiFormatter เป็น gem สำหรับจัดรูปแบบข้อมูลภาษาไทย
# รองรับการแปลงตัวเลข วันที่ และข้อความ
#
# @example การใช้งานพื้นฐาน
#   ThaiFormatter::Number.to_baht(1500.50)
#   #=> "฿1,500.50"
#
#   ThaiFormatter::Date.to_thai_date(Date.today)
#   #=> "30 กันยายน 2569"
module ThaiFormatter
  # Error พื้นฐานของ gem นี้
  class Error < StandardError; end

  # InvalidInputError เกิดขึ้นเมื่อรับ input ที่ไม่ถูกต้อง
  class InvalidInputError < Error; end
end
```

### เขียน Number module

```ruby
# lib/thai_formatter/number.rb
# frozen_string_literal: true

module ThaiFormatter
  # จัดการการแปลงและจัดรูปแบบตัวเลขภาษาไทย
  module Number
    # หน่วยของตัวเลขภาษาไทย
    THAI_DIGITS = %w[ศูนย์ หนึ่ง สอง สาม สี่ ห้า หก เจ็ด แปด เก้า].freeze

    THAI_UNITS = {
      1 => "",
      10 => "สิบ",
      100 => "ร้อย",
      1_000 => "พัน",
      10_000 => "หมื่น",
      100_000 => "แสน",
      1_000_000 => "ล้าน"
    }.freeze

    # แปลงจำนวนเงินเป็นรูปแบบสกุลเงินบาท
    #
    # @param amount [Numeric] จำนวนเงิน
    # @param symbol [String] สัญลักษณ์สกุลเงิน (default: "฿")
    # @return [String] ตัวเลขในรูปแบบสกุลเงิน
    # @raise [InvalidInputError] ถ้า amount ไม่ใช่ตัวเลข
    #
    # @example
    #   ThaiFormatter::Number.to_baht(1500)     #=> "฿1,500.00"
    #   ThaiFormatter::Number.to_baht(1500.5)   #=> "฿1,500.50"
    #   ThaiFormatter::Number.to_baht(-500)     #=> "-฿500.00"
    def self.to_baht(amount, symbol: "฿")
      raise InvalidInputError, "amount ต้องเป็นตัวเลข" unless amount.is_a?(Numeric)

      negative = amount.negative?
      abs_amount = amount.abs
      formatted = format("%.2f", abs_amount)

      # เพิ่ม comma separator
      integer_part, decimal_part = formatted.split(".")
      integer_with_commas = integer_part.gsub(/(\d)(?=(\d{3})+\z)/, "\\1,")

      result = "#{symbol}#{integer_with_commas}.#{decimal_part}"
      negative ? "-#{result}" : result
    end

    # แปลงตัวเลขเป็นคำอ่านภาษาไทย
    #
    # @param number [Integer] ตัวเลขที่ต้องการแปลง (0 ถึง 999,999,999)
    # @return [String] คำอ่านภาษาไทย
    #
    # @example
    #   ThaiFormatter::Number.to_thai_words(0)      #=> "ศูนย์"
    #   ThaiFormatter::Number.to_thai_words(21)     #=> "ยี่สิบเอ็ด"
    #   ThaiFormatter::Number.to_thai_words(1000)   #=> "หนึ่งพัน"
    def self.to_thai_words(number)
      raise InvalidInputError, "รองรับเฉพาะ Integer" unless number.is_a?(Integer)
      raise InvalidInputError, "รองรับเฉพาะเลขไม่เกิน 999,999,999" if number.abs > 999_999_999

      return "ศูนย์" if number.zero?

      negative = number.negative?
      result = convert_to_thai_words(number.abs)
      negative ? "ลบ#{result}" : result
    end

    private_class_method def self.convert_to_thai_words(number)
      return THAI_DIGITS[number] if number < 10

      result = ""
      [1_000_000, 100_000, 10_000, 1_000, 100, 10, 1].each do |unit|
        digit = number / unit
        next if digit.zero?

        number %= unit

        if unit == 10 && digit == 2
          result += "ยี่สิบ"
        elsif unit == 10 && digit == 1
          result += "สิบ"
        elsif unit == 1 && digit == 1 && result.length > 0
          result += "เอ็ด"
        else
          result += "#{THAI_DIGITS[digit]}#{THAI_UNITS[unit]}"
        end
      end

      result
    end
  end
end
```

### เขียน Date module

```ruby
# lib/thai_formatter/date.rb
# frozen_string_literal: true

require "date"

module ThaiFormatter
  # จัดการการแปลงและจัดรูปแบบวันที่แบบไทย
  module Date
    # ชื่อเดือนภาษาไทย
    THAI_MONTHS = %w[
      มกราคม กุมภาพันธ์ มีนาคม เมษายน พฤษภาคม มิถุนายน
      กรกฎาคม สิงหาคม กันยายน ตุลาคม พฤศจิกายน ธันวาคม
    ].freeze

    # ชื่อวันในสัปดาห์ภาษาไทย
    THAI_WEEKDAYS = %w[อาทิตย์ จันทร์ อังคาร พุธ พฤหัสบดี ศุกร์ เสาร์].freeze

    # พุทธศักราชต่างจาก ค.ศ. 543 ปี
    BUDDHIST_ERA_OFFSET = 543

    # แปลงวันที่เป็นรูปแบบไทย
    #
    # @param date [Date, Time, DateTime] วันที่ที่ต้องการแปลง
    # @param era [Symbol] :be สำหรับ พ.ศ., :ce สำหรับ ค.ศ. (default: :be)
    # @return [String] วันที่ในรูปแบบไทย
    #
    # @example
    #   ThaiFormatter::Date.to_thai_date(Date.new(2024, 9, 30))
    #   #=> "30 กันยายน 2567"
    def self.to_thai_date(date, era: :be)
      validate_date!(date)

      day = date.day
      month = THAI_MONTHS[date.month - 1]
      year = era == :be ? date.year + BUDDHIST_ERA_OFFSET : date.year

      "#{day} #{month} #{year}"
    end

    # แปลงวันที่พร้อมวันในสัปดาห์
    #
    # @param date [Date] วันที่
    # @return [String] วันที่พร้อมชื่อวัน
    #
    # @example
    #   ThaiFormatter::Date.to_full_thai_date(Date.new(2024, 9, 30))
    #   #=> "วันจันทร์ที่ 30 กันยายน พ.ศ. 2567"
    def self.to_full_thai_date(date)
      validate_date!(date)

      weekday = THAI_WEEKDAYS[date.wday]
      day = date.day
      month = THAI_MONTHS[date.month - 1]
      year = date.year + BUDDHIST_ERA_OFFSET

      "วัน#{weekday}ที่ #{day} #{month} พ.ศ. #{year}"
    end

    private_class_method def self.validate_date!(date)
      unless date.respond_to?(:day) && date.respond_to?(:month) && date.respond_to?(:year)
        raise InvalidInputError, "ต้องส่ง Date, Time หรือ DateTime object"
      end
    end
  end
end
```

### เขียน Text module

```ruby
# lib/thai_formatter/text.rb
# frozen_string_literal: true

module ThaiFormatter
  # จัดการและ normalize ข้อความภาษาไทย
  module Text
    # ลบช่องว่างซ้ำซ้อนและ normalize unicode
    #
    # @param text [String] ข้อความที่ต้องการ normalize
    # @return [String] ข้อความที่ผ่านการ normalize แล้ว
    def self.normalize(text)
      raise InvalidInputError, "text ต้องเป็น String" unless text.is_a?(String)

      text
        .unicode_normalize(:nfc)
        .gsub(/[[:space:]]+/, " ")
        .strip
    end

    # ตรวจสอบว่าข้อความมีอักขระภาษาไทยหรือไม่
    #
    # @param text [String] ข้อความที่ต้องการตรวจสอบ
    # @return [Boolean]
    def self.contains_thai?(text)
      raise InvalidInputError, "text ต้องเป็น String" unless text.is_a?(String)

      text.match?(/[฀-๿]/)
    end

    # นับจำนวนคำในข้อความภาษาไทย (ประมาณการ)
    # ภาษาไทยไม่มีช่องว่างระหว่างคำ วิธีนี้นับจำนวน token คร่าวๆ
    #
    # @param text [String] ข้อความภาษาไทย
    # @return [Integer] จำนวนคำโดยประมาณ
    def self.approximate_word_count(text)
      raise InvalidInputError, "text ต้องเป็น String" unless text.is_a?(String)

      # ประมาณการ: เฉลี่ย 2-3 ตัวอักษรต่อคำในภาษาไทย
      thai_chars = text.scan(/[฀-๿]/).length
      (thai_chars / 2.5).ceil
    end
  end
end
```

---

## Step 884: .gemspec เชิงลึก {#step-884}

### ไฟล์ `.gemspec` คืออะไร

`.gemspec` คือไฟล์ metadata หลักของ gem ทุก gem ต้องมีไฟล์นี้ มันบอก RubyGems ว่า gem นี้คืออะไร ต้องการ dependency อะไร และประกอบด้วยไฟล์อะไรบ้าง

### โครงสร้างและฟิลด์สำคัญ

```ruby
# thai_formatter.gemspec
# frozen_string_literal: true

require_relative "lib/thai_formatter/version"

Gem::Specification.new do |spec|
  # ── ข้อมูลพื้นฐาน ──────────────────────────────────────────
  spec.name    = "thai_formatter"
  spec.version = ThaiFormatter::VERSION  # ดึงจาก VERSION constant
  spec.authors = ["Your Name"]
  spec.email   = ["your.email@example.com"]

  # สรุปสั้นๆ (ปรากฏใน gem list) — ไม่เกิน 80 ตัวอักษร
  spec.summary = "Ruby gem สำหรับจัดรูปแบบข้อมูลภาษาไทย"

  # คำอธิบายละเอียด (ปรากฏใน gem info)
  spec.description = <<~DESC
    ThaiFormatter เป็น Ruby gem ที่ช่วยจัดรูปแบบ
    ข้อมูลภาษาไทย ได้แก่ ตัวเลขเป็นบาท, ตัวเลขเป็นคำอ่าน,
    วันที่แบบไทย (พุทธศักราช), และ normalize ข้อความ
  DESC

  spec.homepage = "https://github.com/yourusername/thai_formatter"
  spec.license  = "MIT"

  # ── Version ที่รองรับ ───────────────────────────────────────
  spec.required_ruby_version = ">= 3.0.0"

  # ── Links สำคัญ ─────────────────────────────────────────────
  spec.metadata["allowed_push_host"] = "https://rubygems.org"
  spec.metadata["homepage_uri"]      = spec.homepage
  spec.metadata["source_code_uri"]   = spec.homepage
  spec.metadata["changelog_uri"]     = "#{spec.homepage}/blob/main/CHANGELOG.md"
  spec.metadata["documentation_uri"] = "https://rubydoc.info/gems/thai_formatter"
  spec.metadata["rubygems_mfa_required"] = "true"  # บังคับ MFA สำหรับการ push

  # ── ไฟล์ที่รวมใน gem ────────────────────────────────────────
  # spec.files บอกว่าไฟล์ใดจะถูกบรรจุใน gem
  spec.files = Dir.chdir(__dir__) do
    `git ls-files -z`.split("\x0").reject do |f|
      (File.expand_path(f, __dir__) == __FILE__) ||
        f.start_with?(*%w[bin/ test/ spec/ features/ .git .github appveyor Gemfile])
    end
  end

  spec.bindir        = "exe"
  spec.executables   = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]  # บอกว่า require จะหาไฟล์จากที่ไหน

  # ── Runtime Dependencies ──────────────────────────────────────
  # dependency ที่ผู้ใช้ gem ต้องการด้วย
  # spec.add_dependency "some_gem", "~> 2.0"
  # spec.add_dependency "another_gem", ">= 1.0", "< 3.0"

  # ── Development Dependencies ─────────────────────────────────
  # dependency ที่ใช้เฉพาะตอน develop gem (ไม่ถูกติดตั้งเมื่อใช้ gem)
  spec.add_development_dependency "rspec", "~> 3.13"
  spec.add_development_dependency "rubocop", "~> 1.65"
  spec.add_development_dependency "rubocop-rspec", "~> 3.0"
  spec.add_development_dependency "yard", "~> 0.9"
  spec.add_development_dependency "simplecov", "~> 0.22"
end
```

### ความแตกต่างระหว่าง `add_dependency` และ `add_development_dependency`

```ruby
# add_dependency — ผู้ใช้ gem ต้องติดตั้งด้วย
spec.add_dependency "activerecord", "~> 7.0"
# เมื่อใครทำ gem install thai_formatter
# activerecord จะถูกติดตั้งอัตโนมัติ

# add_development_dependency — เฉพาะ developer ของ gem เท่านั้น
spec.add_development_dependency "rspec", "~> 3.13"
# เมื่อใครทำ gem install thai_formatter
# rspec จะ NOT ถูกติดตั้ง
```

### Version Constraints ที่ใช้บ่อย

```ruby
# Pessimistic (~>) — ใช้บ่อยที่สุด
spec.add_dependency "rails", "~> 7.1"
# หมายความว่า >= 7.1 และ < 8.0
# ป้องกัน breaking change จาก major version

spec.add_dependency "activesupport", "~> 7.1.0"
# หมายความว่า >= 7.1.0 และ < 7.2
# เข้มงวดกว่า

# Exact version
spec.add_dependency "some_gem", "= 1.2.3"

# Greater than or equal
spec.add_dependency "some_gem", ">= 1.0"

# Multiple constraints
spec.add_dependency "some_gem", ">= 1.0", "< 3.0"
```

### ตรวจสอบ gemspec

```bash
# validate gemspec
gem build thai_formatter.gemspec

# หรือ
bundle exec rake build

# ดู metadata ของ gem ที่สร้าง
gem specification thai_formatter-0.1.0.gem
```

---

## Step 885: Test Gem ด้วย RSpec {#step-885}

### Setup RSpec

Bundler สร้างไฟล์ RSpec พื้นฐานให้แล้ว แต่เราจะปรับแต่งเพิ่ม:

```ruby
# spec/spec_helper.rb
# frozen_string_literal: true

require "simplecov"

# เริ่ม code coverage ก่อน require gem
SimpleCov.start do
  add_filter "/spec/"
  minimum_coverage 80  # ต้องมี coverage อย่างน้อย 80%
end

require "thai_formatter"
require "date"

RSpec.configure do |config|
  # ปิด monkey-patching (use expect syntax เท่านั้น)
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
  config.filter_run_when_matching :focus
  config.example_status_persistence_file_path = ".rspec_status"
  config.disable_monkey_patching!
  config.warnings = true

  if config.files_to_run.one?
    config.default_formatter = "doc"
  end

  config.profile_examples = 10
  config.order = :random
  Kernel.srand config.seed
end
```

### เขียน Test สำหรับ Number module

```ruby
# spec/thai_formatter/number_spec.rb
# frozen_string_literal: true

RSpec.describe ThaiFormatter::Number do
  describe ".to_baht" do
    context "เมื่อรับจำนวนเต็ม" do
      it "แปลงเป็นรูปแบบสกุลเงินบาท" do
        expect(described_class.to_baht(1000)).to eq("฿1,000.00")
      end

      it "แสดง 2 ตำแหน่งทศนิยม" do
        expect(described_class.to_baht(42)).to eq("฿42.00")
      end

      it "ใส่ comma separator สำหรับตัวเลขใหญ่" do
        expect(described_class.to_baht(1_000_000)).to eq("฿1,000,000.00")
      end
    end

    context "เมื่อรับทศนิยม" do
      it "แปลงทศนิยมได้ถูกต้อง" do
        expect(described_class.to_baht(1500.50)).to eq("฿1,500.50")
      end

      it "ปัดทศนิยมเป็น 2 ตำแหน่ง" do
        expect(described_class.to_baht(99.999)).to eq("฿100.00")
      end
    end

    context "เมื่อรับจำนวนลบ" do
      it "แสดงเครื่องหมายลบหน้าสัญลักษณ์" do
        expect(described_class.to_baht(-500)).to eq("-฿500.00")
      end
    end

    context "เมื่อรับ symbol ที่กำหนดเอง" do
      it "ใช้ symbol ที่กำหนด" do
        expect(described_class.to_baht(100, symbol: "THB ")).to eq("THB 100.00")
      end
    end

    context "เมื่อรับ input ที่ไม่ถูกต้อง" do
      it "raise InvalidInputError สำหรับ string" do
        expect { described_class.to_baht("abc") }
          .to raise_error(ThaiFormatter::InvalidInputError, /ตัวเลข/)
      end

      it "raise InvalidInputError สำหรับ nil" do
        expect { described_class.to_baht(nil) }
          .to raise_error(ThaiFormatter::InvalidInputError)
      end
    end
  end

  describe ".to_thai_words" do
    it "แปลง 0 เป็น ศูนย์" do
      expect(described_class.to_thai_words(0)).to eq("ศูนย์")
    end

    it "แปลงเลขหลักเดียว" do
      expect(described_class.to_thai_words(5)).to eq("ห้า")
    end

    it "แปลง 21 เป็น ยี่สิบเอ็ด" do
      expect(described_class.to_thai_words(21)).to eq("ยี่สิบเอ็ด")
    end

    it "แปลง 10 เป็น สิบ" do
      expect(described_class.to_thai_words(10)).to eq("สิบ")
    end

    it "แปลง 20 เป็น ยี่สิบ" do
      expect(described_class.to_thai_words(20)).to eq("ยี่สิบ")
    end

    it "แปลง 1000 เป็น หนึ่งพัน" do
      expect(described_class.to_thai_words(1000)).to eq("หนึ่งพัน")
    end

    it "แปลงจำนวนลบ" do
      expect(described_class.to_thai_words(-5)).to eq("ลบห้า")
    end

    context "input ไม่ถูกต้อง" do
      it "raise error สำหรับ Float" do
        expect { described_class.to_thai_words(1.5) }
          .to raise_error(ThaiFormatter::InvalidInputError)
      end

      it "raise error สำหรับตัวเลขเกิน limit" do
        expect { described_class.to_thai_words(1_000_000_000) }
          .to raise_error(ThaiFormatter::InvalidInputError)
      end
    end
  end
end
```

### เขียน Test สำหรับ Date module

```ruby
# spec/thai_formatter/date_spec.rb
# frozen_string_literal: true

RSpec.describe ThaiFormatter::Date do
  let(:sample_date) { ::Date.new(2024, 9, 30) }

  describe ".to_thai_date" do
    it "แปลงวันที่เป็นรูปแบบไทย พ.ศ." do
      expect(described_class.to_thai_date(sample_date)).to eq("30 กันยายน 2567")
    end

    it "รองรับ era: :ce สำหรับ ค.ศ." do
      expect(described_class.to_thai_date(sample_date, era: :ce)).to eq("30 กันยายน 2024")
    end

    it "แปลงเดือนมกราคมถูกต้อง" do
      date = ::Date.new(2024, 1, 1)
      expect(described_class.to_thai_date(date)).to eq("1 มกราคม 2567")
    end

    it "แปลงเดือนธันวาคมถูกต้อง" do
      date = ::Date.new(2024, 12, 31)
      expect(described_class.to_thai_date(date)).to eq("31 ธันวาคม 2567")
    end

    it "รับ Time object ได้" do
      time = Time.new(2024, 9, 30)
      expect(described_class.to_thai_date(time)).to eq("30 กันยายน 2567")
    end

    it "raise error สำหรับ string" do
      expect { described_class.to_thai_date("2024-09-30") }
        .to raise_error(ThaiFormatter::InvalidInputError)
    end
  end

  describe ".to_full_thai_date" do
    it "แสดงวันที่พร้อมชื่อวัน" do
      # 30 กันยายน 2024 ตรงกับวันจันทร์
      expect(described_class.to_full_thai_date(sample_date))
        .to eq("วันจันทร์ที่ 30 กันยายน พ.ศ. 2567")
    end
  end
end
```

### รัน Tests

```bash
# รันทุก test
bundle exec rspec

# รัน test เฉพาะไฟล์
bundle exec rspec spec/thai_formatter/number_spec.rb

# รันพร้อม format แบบ documentation
bundle exec rspec --format documentation

# รัน test พร้อม coverage report
bundle exec rspec

# ดู coverage report
open coverage/index.html
```

### ผลลัพธ์ที่ควรได้

```
ThaiFormatter::Number
  .to_baht
    เมื่อรับจำนวนเต็ม
      แปลงเป็นรูปแบบสกุลเงินบาท
      แสดง 2 ตำแหน่งทศนิยม
      ใส่ comma separator สำหรับตัวเลขใหญ่
    เมื่อรับทศนิยม
      แปลงทศนิยมได้ถูกต้อง
      ปัดทศนิยมเป็น 2 ตำแหน่ง
    ...

Finished in 0.05 seconds (files took 0.3 seconds to load)
18 examples, 0 failures

Coverage report generated. 94.32% covered.
```

---

## Step 886: Rake Tasks สำหรับ Gem {#step-886}

### Rakefile พื้นฐาน

Bundler สร้าง `Rakefile` พื้นฐานให้แล้ว มาดูและปรับแต่ง:

```ruby
# Rakefile
# frozen_string_literal: true

require "bundler/gem_tasks"
require "rspec/core/rake_task"
require "rubocop/rake_task"

# Task สำหรับรัน RSpec
RSpec::RakeTask.new(:spec)

# Task สำหรับรัน RuboCop (linter)
RuboCop::RakeTask.new

# Task default — รันเมื่อพิมพ์แค่ `rake`
task default: %i[spec rubocop]

# Task สำหรับ generate documentation
begin
  require "yard"
  YARD::Rake::YardocTask.new(:doc) do |t|
    t.files = ["lib/**/*.rb"]
    t.options = ["--output-dir", "doc", "--markup", "markdown"]
  end
rescue LoadError
  task(:doc) { puts "YARD not available" }
end
```

### Task จาก `bundler/gem_tasks`

การ `require "bundler/gem_tasks"` เพิ่ม tasks สำคัญดังนี้:

```bash
# ดู rake tasks ทั้งหมด
bundle exec rake --tasks

# Output:
# rake build         # Build thai_formatter-0.1.0.gem into the pkg directory
# rake clean         # Remove any temporary products
# rake clobber       # Remove any generated files
# rake install       # Build and install thai_formatter-0.1.0.gem into system gems
# rake install:local # Build and install thai_formatter-0.1.0.gem into system gems without network access
# rake release[remote]  # Create tag v0.1.0 and build and push thai_formatter-0.1.0.gem to rubygems.org
# rake spec          # Run RSpec code examples
# rake rubocop       # Run RuboCop
```

### การใช้ Rake Tasks

```bash
# Build gem ไว้ใน pkg/ directory
bundle exec rake build
# pkg/thai_formatter-0.1.0.gem ถูกสร้าง

# ติดตั้ง gem เข้าระบบ local (สำหรับทดสอบ)
bundle exec rake install

# ทดสอบ gem ที่ติดตั้ง
irb
# > require 'thai_formatter'
# > ThaiFormatter::Number.to_baht(1500)

# Build + push ไป RubyGems.org (ต้อง login ก่อน)
bundle exec rake release
```

### สร้าง Custom Rake Task สำหรับ Version Bump

```ruby
# Rakefile (เพิ่มเติม)
namespace :version do
  desc "Bump patch version (0.1.0 -> 0.1.1)"
  task :patch do
    bump_version(:patch)
  end

  desc "Bump minor version (0.1.0 -> 0.2.0)"
  task :minor do
    bump_version(:minor)
  end

  desc "Bump major version (0.1.0 -> 1.0.0)"
  task :major do
    bump_version(:major)
  end

  def bump_version(type)
    version_file = "lib/thai_formatter/version.rb"
    content = File.read(version_file)

    # ดึง version ปัจจุบัน
    current = content.match(/VERSION = "(\d+)\.(\d+)\.(\d+)"/)[1..3].map(&:to_i)
    major, minor, patch = current

    new_version = case type
                  when :patch then "#{major}.#{minor}.#{patch + 1}"
                  when :minor then "#{major}.#{minor + 1}.0"
                  when :major then "#{major + 1}.0.0"
                  end

    # อัปเดตไฟล์
    new_content = content.gsub(/VERSION = "\d+\.\d+\.\d+"/, %(VERSION = "#{new_version}"))
    File.write(version_file, new_content)

    puts "Version bumped: #{current.join('.')} -> #{new_version}"
    puts "Don't forget to update CHANGELOG.md!"
  end
end
```

```bash
# ใช้งาน
bundle exec rake version:patch
# Version bumped: 0.1.0 -> 0.1.1
# Don't forget to update CHANGELOG.md!
```

---

## Step 887: Publish ไปยัง RubyGems.org {#step-887}

### ขั้นตอนการ Publish

**1. สมัคร Account บน RubyGems.org**

```bash
# ไปที่ https://rubygems.org/sign_up และสมัคร account
# หลังจากสมัครแล้ว ให้ enable MFA (แนะนำอย่างยิ่ง)
```

**2. Setup credentials บนเครื่อง**

```bash
# Login และบันทึก API key
gem signin

# จะถามอีเมลและรหัสผ่าน
# Enter your RubyGems.org credentials.
# Don't have an account yet? Create one at https://rubygems.org/sign_up
#
#    Email:   your.email@example.com
#  Password:
#
# Signed in with API key: rubygems.org-XXXXXXXX.

# credentials จะถูกบันทึกที่
cat ~/.gem/credentials
# ---
# :rubygems_api_key: rubygems_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

**3. ตรวจสอบ gem ก่อน publish**

```bash
# Build gem
bundle exec rake build

# ดูไฟล์ใน gem ที่จะ publish
gem contents pkg/thai_formatter-0.1.0.gem

# ตรวจสอบ metadata
gem specification pkg/thai_formatter-0.1.0.gem

# ทดสอบ install จาก local
gem install pkg/thai_formatter-0.1.0.gem

# ทดสอบว่าทำงานได้
ruby -e "require 'thai_formatter'; puts ThaiFormatter::Number.to_baht(1000)"
```

**4. Publish ไปยัง RubyGems.org**

```bash
# วิธีที่ 1: ใช้ rake release (แนะนำ)
# rake release จะ: build gem, create git tag, push ไป RubyGems.org
bundle exec rake release

# วิธีที่ 2: push โดยตรง
gem push pkg/thai_formatter-0.1.0.gem
```

### ป้องกัน Accidental Push

เพิ่ม metadata ใน gemspec เพื่อกำหนด allowed push host:

```ruby
# thai_formatter.gemspec
spec.metadata["allowed_push_host"] = "https://rubygems.org"

# ถ้าเป็น private gem ให้ใส่ URL ของ private server แทน
# spec.metadata["allowed_push_host"] = "https://gems.yourdomain.com"
```

ถ้าพยายาม push ไปยัง host อื่น จะได้ error:

```
ERROR: "gem push" was called with arguments ["pkg/thai_formatter-0.1.0.gem"]
but allowed push host is "https://rubygems.org"
```

### การจัดการ API Key

```bash
# ดู credentials ที่มี
gem signin --list
# Credentials stored at /root/.gem/credentials

# สร้าง API key ใหม่ (บน RubyGems.org → Settings → API Keys)
# กำหนดสิทธิ์เฉพาะที่จำเป็น:
# - index: อ่าน gem list
# - push: publish gem ใหม่
# - yank: ลบ gem version

# ใช้ API key เฉพาะ
gem push pkg/thai_formatter-0.1.0.gem --key rubygems.org

# บันทึก key เพิ่มเติม
gem signin --key my-deployment-key
```

### หลัง Publish

```bash
# ตรวจสอบว่า gem พร้อมใช้งาน
gem list -r thai_formatter

# ดูหน้า gem บน RubyGems.org
# https://rubygems.org/gems/thai_formatter

# ทดสอบ install
gem install thai_formatter
ruby -e "require 'thai_formatter'; puts ThaiFormatter::VERSION"
```

---

## Step 888: Versioning Strategy {#step-888}

### Semantic Versioning (SemVer)

SemVer คือมาตรฐานการตั้งเลข version ในรูปแบบ `MAJOR.MINOR.PATCH`:

```
Version: 2.4.1
         │ │ └── PATCH: แก้ bug ที่ backward compatible
         │ └──── MINOR: เพิ่ม feature ที่ backward compatible
         └────── MAJOR: เปลี่ยน API ที่ไม่ backward compatible (breaking change)
```

### กฎของ SemVer

| เมื่อไหร่ | เพิ่ม | ตัวอย่าง |
|---------|------|---------|
| แก้ bug ที่ไม่กระทบ API | PATCH | 1.0.0 → 1.0.1 |
| เพิ่ม method/feature ใหม่ | MINOR | 1.0.1 → 1.1.0 |
| ลบ/เปลี่ยน method ที่มีอยู่ | MAJOR | 1.1.0 → 2.0.0 |
| เวอร์ชัน pre-release | suffix | 2.0.0-alpha.1 |

### ตัวอย่างการ evolve VERSION

```ruby
# lib/thai_formatter/version.rb

# Initial release
VERSION = "0.1.0"

# แก้ bug ใน to_baht ที่ handle nil ไม่ถูกต้อง
VERSION = "0.1.1"

# เพิ่ม ThaiFormatter::Text module
VERSION = "0.2.0"

# เพิ่ม ThaiFormatter::Date.to_full_thai_date
VERSION = "0.2.1"  # หรือ 0.3.0 ถ้าถือว่า feature ใหม่

# เปลี่ยน API ของ to_thai_date (parameter เปลี่ยนจาก :buddhist เป็น :be)
VERSION = "1.0.0"  # MAJOR เพราะ breaking change
```

### CHANGELOG.md

ทุก release ควรอัปเดต `CHANGELOG.md`:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- รองรับการแปลงตัวเลขเกิน 100 ล้าน

## [0.2.0] - 2024-09-30

### Added
- เพิ่ม `ThaiFormatter::Text` module สำหรับ normalize ข้อความ
- เพิ่ม `ThaiFormatter::Text.contains_thai?`
- เพิ่ม `ThaiFormatter::Text.approximate_word_count`

### Changed
- ปรับปรุง error messages ให้เข้าใจง่ายขึ้น

## [0.1.1] - 2024-09-20

### Fixed
- แก้ bug ที่ `to_baht` ไม่ throw error เมื่อรับ nil

## [0.1.0] - 2024-09-15

### Added
- Initial release
- `ThaiFormatter::Number.to_baht`
- `ThaiFormatter::Number.to_thai_words`
- `ThaiFormatter::Date.to_thai_date`
- `ThaiFormatter::Date.to_full_thai_date`
```

### Git Tags

ทุก release ควรสร้าง git tag:

```bash
# สร้าง annotated tag
git tag -a v0.2.0 -m "Release version 0.2.0

- เพิ่ม ThaiFormatter::Text module
- แก้ error messages"

# Push tag ไป remote
git push origin v0.2.0

# ดู tags ทั้งหมด
git tag -l

# ดู tag ที่ระบุ
git show v0.2.0
```

### Deprecation Warning

ก่อน breaking change ควรมี deprecation period:

```ruby
# lib/thai_formatter/number.rb

# ฟังก์ชัน old API — deprecated ตั้งแต่ v0.2.0
# จะถูกลบใน v1.0.0
def self.format_baht(amount)
  warn "[DEPRECATION] `ThaiFormatter::Number.format_baht` is deprecated " \
       "and will be removed in v1.0.0. Use `to_baht` instead."
  to_baht(amount)
end
```

---

## Step 889: Private Gem Server {#step-889}

### ทำไมต้องใช้ Private Gem Server

บางครั้งเราไม่ต้องการ publish gem เป็น public บน RubyGems.org เช่น:
- gem มี business logic ที่เป็นความลับ
- gem มี credential หรือ configuration ขององค์กร
- ต้องการควบคุมว่าใครเข้าถึงได้บ้าง

### ตัวเลือก Private Gem Server

#### 1. GitHub Packages

```ruby
# Gemfile ของโปรเจกต์ที่ใช้ private gem
source "https://rubygems.pkg.github.com/your-org" do
  gem "internal_gem", "~> 1.0"
end

# สร้าง .netrc หรือ credentials
# ~/.netrc
machine rubygems.pkg.github.com
  login YOUR_GITHUB_USERNAME
  password YOUR_GITHUB_PERSONAL_ACCESS_TOKEN
```

```bash
# Publish ไปยัง GitHub Packages
gem push --key github \
  --host https://rubygems.pkg.github.com/your-org \
  pkg/internal_gem-1.0.0.gem
```

#### 2. Gemfury

Gemfury เป็น hosted private gem server ที่ง่ายที่สุด:

```bash
# ติดตั้ง gemfury CLI
gem install gemfury

# Push gem
fury push pkg/internal_gem-1.0.0.gem --as=your-org

# ใช้ใน Gemfile
source "https://gem.fury.io/your-org-token/" do
  gem "internal_gem"
end
```

#### 3. Nexus Repository Manager

สำหรับองค์กรที่ต้องการ on-premise solution:

```ruby
# gemspec
spec.metadata["allowed_push_host"] = "https://nexus.yourcompany.com/repository/rubygems-hosted/"

# Gemfile
source "https://nexus.yourcompany.com/repository/rubygems-proxy/"
source "https://nexus.yourcompany.com/repository/rubygems-hosted/"

gem "internal_gem", "~> 1.0"

# Push ด้วย credentials
gem push pkg/internal_gem-1.0.0.gem \
  --host https://nexus.yourcompany.com/repository/rubygems-hosted/ \
  --key nexus
```

#### 4. Self-hosted ด้วย Geminabox

```bash
# ติดตั้ง geminabox
gem install geminabox

# สร้าง config.ru
cat > config.ru << 'EOF'
require "geminabox"
Geminabox.data = "/var/geminabox-data"
run Geminabox::Server
EOF

# รัน server
rackup config.ru

# Push gem
gem inabox pkg/internal_gem-1.0.0.gem --host http://localhost:9292
```

### การตั้งค่า Credentials ใน CI/CD

```yaml
# .github/workflows/publish.yml
name: Publish Gem

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3.6'
          bundler-cache: true

      - name: Setup credentials
        run: |
          mkdir -p ~/.gem
          cat > ~/.gem/credentials << EOF
          ---
          :rubygems_api_key: ${{ secrets.RUBYGEMS_API_KEY }}
          EOF
          chmod 0600 ~/.gem/credentials

      - name: Build and push gem
        run: |
          bundle exec rake build
          gem push pkg/*.gem
```

### ใช้ Gemfile.lock อย่างถูกต้อง

สำหรับ gem ที่ใช้ใน production:

```bash
# commit Gemfile.lock สำหรับ application
git add Gemfile.lock

# แต่สำหรับ gem library — KHÔNG commit Gemfile.lock
# เพิ่ม Gemfile.lock ใน .gitignore ของ gem
echo "Gemfile.lock" >> .gitignore
```

เหตุผลที่ไม่ commit `Gemfile.lock` ใน gem:
- ผู้ใช้ gem ใช้ version ต่างๆ ของ Ruby และ gems
- การ lock version ใน gem อาจทำให้เกิด conflict กับ Gemfile.lock ของแอปพลิเคชัน

---

## Step 890: Maintain Gem อย่างมืออาชีพ {#step-890}

### Issue Triage

การจัดการ issues บน GitHub อย่างเป็นระบบ:

```markdown
# Issue Templates (.github/ISSUE_TEMPLATE/bug_report.md)

---
name: Bug Report
about: Create a report to help us improve
title: '[BUG] '
labels: 'bug'
---

## Environment
- Ruby version: 
- thai_formatter version: 
- OS: 

## Describe the bug
A clear description of what the bug is.

## To Reproduce
```ruby
# Minimal reproduction code
require 'thai_formatter'
ThaiFormatter::Number.to_baht(...)
```

## Expected behavior
What you expected to happen.

## Actual behavior
What actually happened (include error messages).
```

### Label Strategy

```
Labels สำหรับ issue triage:
bug         → code ทำงานผิดพลาด
enhancement → feature request ใหม่
question    → ถามข้อสงสัย
documentation → เกี่ยวกับ docs
good first issue → เหมาะสำหรับ new contributor
help wanted → ต้องการความช่วยเหลือ
wontfix     → ไม่แก้
duplicate   → ซ้ำกับ issue อื่น
```

### รับ Pull Requests อย่างมีคุณภาพ

```markdown
# .github/PULL_REQUEST_TEMPLATE.md

## Summary
อธิบายสิ่งที่เปลี่ยนแปลงและเหตุผล

## Type of change
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change

## Testing
- [ ] เพิ่ม/อัปเดต tests สำหรับ changes เหล่านี้
- [ ] รัน `bundle exec rspec` แล้วผ่านทั้งหมด
- [ ] รัน `bundle exec rubocop` แล้วไม่มี offense

## Checklist
- [ ] อัปเดต CHANGELOG.md แล้ว
- [ ] อัปเดต README ถ้ามีการเปลี่ยน public API
```

### Security Updates

```ruby
# ใช้ bundler-audit เพื่อตรวจหา vulnerability
# Gemfile (development dependencies)

# gemspec
spec.add_development_dependency "bundler-audit", "~> 0.9"

# Rakefile
require "bundler/audit/task"
Bundler::Audit::Task.new

# รัน security check
bundle exec rake bundle:audit
```

```yaml
# .github/workflows/security.yml
name: Security Check

on:
  schedule:
    - cron: '0 8 * * 1'  # ทุกวันจันทร์ 8 โมงเช้า
  push:
    branches: [main]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3.6'
          bundler-cache: true
      - run: bundle exec bundle-audit check --update
```

### Deprecation อย่างถูกต้อง

```ruby
# lib/thai_formatter/number.rb

module ThaiFormatter
  module Number
    # @deprecated Use {.to_baht} instead. Will be removed in v2.0.0.
    def self.format_as_currency(amount)
      ActiveSupport::Deprecation.warn(
        "`ThaiFormatter::Number.format_as_currency` is deprecated " \
        "and will be removed in v2.0.0. Use `to_baht` instead.",
        caller
      )
      to_baht(amount)
    end
  end
end
```

### Continuous Integration

```yaml
# .github/workflows/main.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        ruby-version: ['3.1', '3.2', '3.3']

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby ${{ matrix.ruby-version }}
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ matrix.ruby-version }}
          bundler-cache: true

      - name: Run tests
        run: bundle exec rspec

      - name: Run linter
        run: bundle exec rubocop

      - name: Security audit
        run: bundle exec bundle-audit check --update

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.ruby-version == '3.3'
```

### README ที่ดี

```markdown
# README.md

# ThaiFormatter

[![Gem Version](https://badge.fury.io/rb/thai_formatter.svg)](https://badge.fury.io/rb/thai_formatter)
[![CI](https://github.com/yourusername/thai_formatter/actions/workflows/main.yml/badge.svg)](...)
[![Coverage Status](https://codecov.io/gh/yourusername/thai_formatter/branch/main/graph/badge.svg)](...)

Ruby gem สำหรับจัดรูปแบบข้อมูลภาษาไทย

## Installation

เพิ่มใน `Gemfile`:

```ruby
gem 'thai_formatter', '~> 0.1'
```

หรือติดตั้งโดยตรง:

```bash
gem install thai_formatter
```

## Usage

### จัดรูปแบบตัวเลขเป็นสกุลเงินบาท

```ruby
ThaiFormatter::Number.to_baht(1500)     #=> "฿1,500.00"
ThaiFormatter::Number.to_baht(1500.50)  #=> "฿1,500.50"
```

### แปลงตัวเลขเป็นคำอ่าน

```ruby
ThaiFormatter::Number.to_thai_words(21)    #=> "ยี่สิบเอ็ด"
ThaiFormatter::Number.to_thai_words(1000)  #=> "หนึ่งพัน"
```

### จัดรูปแบบวันที่แบบไทย

```ruby
ThaiFormatter::Date.to_thai_date(Date.today)
#=> "30 กันยายน 2567"
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes (`git commit -am 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/my-feature`)
5. Create a new Pull Request

กรุณาอ่าน [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) ก่อน contribute

## License

MIT License — ดูรายละเอียดที่ [LICENSE.txt](LICENSE.txt)
```

### Checklist ก่อน Release ทุกครั้ง

```bash
# 1. รัน full test suite
bundle exec rspec

# 2. รัน linter
bundle exec rubocop

# 3. รัน security audit
bundle exec bundle-audit check --update

# 4. อัปเดต CHANGELOG.md

# 5. Bump version
bundle exec rake version:patch  # หรือ minor/major

# 6. Commit changes
git add .
git commit -m "Release v0.1.1"

# 7. Build และ test gem locally
bundle exec rake build
gem install pkg/thai_formatter-0.1.1.gem
ruby -e "require 'thai_formatter'; puts ThaiFormatter::VERSION"

# 8. Push และ release
bundle exec rake release
# (สร้าง git tag, build, และ push ไป RubyGems.org)

# 9. สร้าง GitHub Release
# ไปที่ Releases page บน GitHub และสร้าง release note
```

---

## แบบฝึกหัด

### ระดับพื้นฐาน

**แบบฝึกหัดที่ 1: สร้าง Gem แรก**

สร้าง gem ชื่อ `string_calculator` ที่มี method:
- `StringCalculator.add(string)` — รับ string ที่มีตัวเลขคั่นด้วย comma หรือ newline เช่น "1,2,3" หรือ "1\n2,3" และคืนผลบวก
- รับ string ว่างคืน 0
- รับ negative number ให้ raise error บอกว่า negative number ไม่ได้รับ

```ruby
# ตัวอย่างผลลัพธ์ที่คาดหวัง
StringCalculator.add("")        # => 0
StringCalculator.add("1")       # => 1
StringCalculator.add("1,2")     # => 3
StringCalculator.add("1\n2,3")  # => 6
StringCalculator.add("-1,2")    # raise "negatives not allowed: -1"
```

**แบบฝึกหัดที่ 2: เขียน Tests ให้ครบ**

เขียน RSpec tests ให้ครอบคลุมทุก case ของ `StringCalculator.add` รวมถึง edge cases และ test สำหรับ error

### ระดับกลาง

**แบบฝึกหัดที่ 3: เพิ่ม Feature และ Bump Version**

เพิ่ม features ให้ `string_calculator`:
- รองรับ custom delimiter ในรูปแบบ `"//[delimiter]\n numbers"` เช่น `"//;\n1;2"` คืน 3
- รองรับ delimiter ยาวกว่า 1 ตัวอักษร เช่น `"//[***]\n1***2***3"` คืน 6

หลังจากนั้น:
1. อัปเดต CHANGELOG.md
2. Bump minor version (เพราะเพิ่ม feature ใหม่)
3. Build gem และทดสอบ

**แบบฝึกหัดที่ 4: Publish ไปยัง GitHub Packages**

1. สร้าง GitHub repository สำหรับ gem
2. ตั้งค่า GitHub Actions สำหรับ CI
3. ตั้งค่า publish ไปยัง GitHub Packages
4. ทดสอบติดตั้ง gem จาก GitHub Packages ในโปรเจกต์อื่น

### ระดับสูง

**แบบฝึกหัดที่ 5: Rails Integration**

สร้าง gem ที่รวม Railtie เพื่อ:
- เพิ่ม helper methods ให้ทุก Rails View
- เพิ่ม configuration initializer
- ใส่ locale file สำหรับ error messages ภาษาไทย

```ruby
# เมื่อเพิ่ม gem ใน Rails app
# app/views/invoices/show.html.erb
<%= thai_baht(invoice.amount) %>    <%# helper จาก gem %>
<%= thai_date(invoice.created_at) %> <%# helper จาก gem %>
```

**แบบฝึกหัดที่ 6: Performance Benchmark**

เพิ่ม benchmark ใน `spec/benchmarks/` เพื่อวัด performance ของ:
- `to_thai_words` กับตัวเลขหลากหลายขนาด
- `to_thai_date` กับ volume สูง

และเพิ่ม rake task สำหรับรัน benchmarks

---

## สรุปสิ่งที่ได้เรียนรู้

ใน Part นี้ครอบคลุม 3 ด้านหลักของการสร้างและดูแล Ruby Gem:

### ด้านโครงสร้างและการพัฒนา

| สิ่งที่เรียนรู้ | คำสั่ง/ไฟล์หลัก |
|--------------|---------------|
| สร้าง gem skeleton | `bundle gem gem_name` |
| โครงสร้างไฟล์มาตรฐาน | `lib/`, `spec/`, `.gemspec` |
| VERSION constant | `lib/gem_name/version.rb` |
| require structure | `require_relative` hierarchy |
| `.gemspec` configuration | `add_dependency`, `add_development_dependency` |

### ด้านคุณภาพและ Testing

| เครื่องมือ | วัตถุประสงค์ |
|----------|------------|
| RSpec | Unit testing |
| SimpleCov | Code coverage |
| RuboCop | Code style linter |
| bundler-audit | Security vulnerability check |
| YARD | Documentation generation |

### ด้านการ Publish และ Maintain

| กระบวนการ | เครื่องมือ |
|----------|----------|
| Build gem | `rake build` |
| Publish public | `gem push` / `rake release` |
| Publish private | GitHub Packages, Gemfury, Nexus |
| Versioning | SemVer + CHANGELOG.md + git tags |
| CI/CD | GitHub Actions |

### Best Practices สำคัญ

1. **เริ่มด้วยการวาง API ที่ดี** — คิดว่าผู้ใช้ gem จะใช้อย่างไร ก่อนเขียนโค้ด
2. **Test coverage สูง** — gem ที่ดีควรมี coverage อย่างน้อย 90%
3. **Version อย่าง SemVer** — ช่วยให้ผู้ใช้ pin version ได้อย่างปลอดภัย
4. **ไม่ commit Gemfile.lock** — gem library ควรยืดหยุ่นเรื่อง dependency version
5. **Deprecate ก่อน remove** — ให้เวลาผู้ใช้ปรับตัวก่อน breaking change
6. **Security first** — ใช้ MFA บน RubyGems.org และ run bundle-audit เป็นประจำ
7. **Documentation ครบถ้วน** — README ที่ดีและ YARD docs ช่วยลด support burden

---

## ตัวอย่าง Workflow สมบูรณ์

```bash
# วันที่ 1: เริ่มโปรเจกต์
bundle gem thai_formatter --test=rspec --ci=github --mit --coc --changelog

# พัฒนา features
cd thai_formatter
# ... เขียนโค้ดใน lib/ ...

# รัน tests ตลอดการพัฒนา
bundle exec rspec

# เตรียม release
bundle exec rubocop -a        # auto-fix style issues
bundle exec bundle-audit check
# อัปเดต CHANGELOG.md
bundle exec rake version:patch # หรือ minor/major ตาม change
git add .
git commit -m "Release v0.1.0: initial implementation"

# Release
bundle exec rake release
# gem pushed → https://rubygems.org/gems/thai_formatter
```

---

## Preview: Part 090 — Metaprogramming ขั้นสูง

ใน Part ต่อไป **Part 090: Metaprogramming ขั้นสูง** เราจะเจาะลึกเรื่อง Ruby Metaprogramming ซึ่งเป็นหัวใจของ magic ที่อยู่เบื้องหลัง Rails และ Gem หลายตัว:

- **`method_missing` และ `respond_to_missing?`** — intercepting unknown method calls
- **`define_method` และ Dynamic Method Generation** — สร้าง method ในเวลา runtime
- **`class_eval` และ `instance_eval`** — เปิด class และ instance เพื่อเพิ่มโค้ด
- **`included`, `extended`, `prepended` hooks** — lifecycle ของ module
- **`ObjectSpace`** — introspect ทุก object ในระบบ
- **DSL Construction** — สร้าง Domain Specific Language ด้วย metaprogramming
- **`Proc`, `Lambda`, และ `Method` objects** — callable objects ใน Ruby

Metaprogramming คือความสามารถที่ทำให้ Ruby เป็น Ruby และเข้าใจมันจะทำให้คุณอ่านและเขียนโค้ด Rails รวมถึง gem ได้อย่างมีประสิทธิภาพมากขึ้น
