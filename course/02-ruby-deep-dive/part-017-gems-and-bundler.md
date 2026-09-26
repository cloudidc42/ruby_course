# Part 017: Gem, Bundler เชิงลึก และการสร้าง Gem ของตัวเอง

> **Step ครอบคลุมใน Part นี้:** Step 161–170
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 001–016 มาก่อน โดยเฉพาะ Part 001 Step 4 ที่แนะนำ `gem`/`bundler` เบื้องต้นไว้แล้ว)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, RubyGems 3.5.x, Bundler 2.5.x (ติดตั้งมาพร้อม Ruby 3.3 อยู่แล้ว)

ใน Part 001 Step 4 เราเกริ่นไว้แล้วว่า `gem` คือหน่วยของ library ใน Ruby และ `bundler` คือ
เครื่องมือจัดการ dependency ผ่านไฟล์ `Gemfile`/`Gemfile.lock` — Part นี้จะไม่พูดซ้ำเรื่องพื้นฐาน
เหล่านั้นอีก แต่จะเจาะลึกลงไปว่า **gem คืออะไรจริงๆ ในระดับไฟล์**, **version constraint ทำงาน
อย่างไรกันแน่**, **ทำไม `bundle exec` ถึงสำคัญมากในโปรเจกต์จริง**, และปิดท้ายด้วยการ
**ลงมือสร้าง gem ของตัวเองตั้งแต่ศูนย์** จนถึงขั้นติดตั้งทดสอบบนเครื่องตัวเอง (ยังไม่ publish
ขึ้น RubyGems.org จริง — ส่วนนั้นจะสอนละเอียดใน Part 089 ของเฟส 15)

## สารบัญของ Part นี้

- Step 161: Gem คืออะไรในระดับไฟล์ และโครงสร้างไดเรกทอรีมาตรฐานของ gem
- Step 162: Version constraint ใน Gemfile เจาะลึก (`~>`, `>=`, `=`, การ pin เวอร์ชัน)
- Step 163: Gemfile groups (`:development`, `:test`, `:production`) และ `require: false`
- Step 164: `Gemfile.lock` คืออะไร และทำไมต้อง commit เข้า version control เสมอ
- Step 165: `bundle exec` เจาะลึก — ทำไมรันคำสั่งตรงๆ อาจได้ gem ผิดเวอร์ชัน
- Step 166: การระบุแหล่งที่มาของ gem: `path:`, `git:`, และ `github:` shorthand
- Step 167: สร้าง gem ใหม่ด้วย `bundle gem` และสำรวจโครงสร้างที่ถูก generate
- Step 168: เขียน metadata ใน `.gemspec` (name, version, summary, dependencies)
- Step 169: implement ฟังก์ชันการทำงานจริงในไฟล์ library ของ gem
- Step 170: ทดสอบ gem บนเครื่องตัวเองก่อน publish (`bundle exec rspec`, `gem build`, `gem install`)

---

## Step 161: Gem คืออะไรในระดับไฟล์ และโครงสร้างไดเรกทอรีมาตรฐานของ gem

ใน Part 001 เราบอกว่า "gem คือหน่วยของ library/package ใน Ruby" ในระดับ concept
แต่ในระดับไฟล์จริงๆ **gem หนึ่งตัวคือไฟล์ archive นามสกุล `.gem`** ซึ่งภายในเป็นไฟล์ `tar`
ที่บรรจุ 3 ส่วนหลัก:

1. `metadata.gz` — ข้อมูล metadata ของ gem (ชื่อ, เวอร์ชัน, dependency, author ฯลฯ) ที่ถูก
   compress ด้วย gzip เก็บในรูปแบบ YAML ที่ serialize มาจากออบเจกต์ `Gem::Specification`
2. `data.tar.gz` — ไฟล์ source code จริงทั้งหมดของ gem (โฟลเดอร์ `lib/`, `bin/` ฯลฯ)
3. `checksums.yaml.gz` — SHA checksum ของไฟล์ทั้งสองด้านบน ใช้ตรวจสอบความถูกต้อง

เราสามารถพิสูจน์ได้ด้วยการแกะ gem ที่ติดตั้งอยู่แล้วออกมาดู:

```bash
# ดูว่า gem ถูกติดตั้งอยู่ที่ไหนบนเครื่อง
gem which rspec
# => /home/user/.rbenv/versions/3.3.5/lib/ruby/gems/3.3.0/gems/rspec-3.13.0/lib/rspec.rb

# ดูรายการไฟล์ทั้งหมดที่มากับ gem โดยไม่ต้อง unpack
gem contents rspec-core | head -20

# แกะ .gem file ออกมาดูโครงสร้างจริง (ดาวน์โหลด gem ใหม่มาแกะในโฟลเดอร์ปัจจุบัน)
gem fetch rspec-core --version 3.13.0
gem unpack rspec-core-3.13.0.gem
ls rspec-core-3.13.0/
```

ผลลัพธ์จาก `ls` จะเห็นโครงสร้างที่เกือบทุก gem ใน ecosystem ของ Ruby ใช้ร่วมกัน เพราะเป็น
**convention มาตรฐาน** ที่ RubyGems และ Bundler คาดหวัง:

```
my_gem/
├── my_gem.gemspec        # ไฟล์ metadata หลัก — "พิมพ์เขียว" ของ gem
├── Gemfile                # ใช้ตอน develop gem ตัวเอง (มักจะ source จาก gemspec)
├── Rakefile                # งานอัตโนมัติ เช่น รัน test, build gem
├── README.md
├── LICENSE.txt
├── CHANGELOG.md
├── lib/
│   ├── my_gem.rb          # entry point — ไฟล์ที่ถูกโหลดเมื่อมีคน require "my_gem"
│   └── my_gem/
│       ├── version.rb     # เก็บค่าคงที่ VERSION ไว้ที่เดียว ใช้ทั้งใน gemspec และโค้ด
│       └── some_class.rb  # ไฟล์ implementation อื่นๆ ที่แตกออกมาตาม class/module
├── spec/                   # เทสต์ (ถ้าเลือก RSpec ตอนสร้าง)
│   ├── spec_helper.rb
│   └── my_gem_spec.rb
├── bin/
│   ├── console             # สคริปต์เปิด irb/pry แบบโหลด gem ไว้ล่วงหน้า สำหรับทดลองโค้ด
│   └── setup                # สคริปต์ setup โปรเจกต์ให้คนอื่นที่ clone มาใหม่
└── sig/                     # (ทางเลือก) RBS type signature สำหรับ static type checking
    └── my_gem.rbs
```

**จุดสำคัญที่ต้องเข้าใจ:**

- ชื่อไฟล์ entry point ใน `lib/` **ต้องตรงกับชื่อ gem เป๊ะๆ** (ขีดกลาง `-` ในชื่อ gem จะ
  กลายเป็น `/` ในชื่อไฟล์ เช่น gem ชื่อ `active-model-serializers` จะมี entry point ที่
  `lib/active/model/serializers.rb`) นี่คือเหตุผลที่ gem ส่วนใหญ่นิยมตั้งชื่อด้วย underscore
  `_` แทน `-` เพื่อให้ mapping ตรงไปตรงมา (`my_gem` → `lib/my_gem.rb`)
- เมื่อมีคนเขียน `require "my_gem"` RubyGems จะค้นหาไฟล์ `my_gem.rb` ใน `$LOAD_PATH`
  ที่ gem ได้ลงทะเบียนไว้ (ซึ่งก็คือโฟลเดอร์ `lib/` ของ gem นั้น) — เราจะเห็นเรื่องนี้ชัดเจน
  เมื่อเขียน gem เองใน Step 167 เป็นต้นไป
- `version.rb` แยกเก็บเลขเวอร์ชันไว้ต่างหาก เพื่อให้ทั้ง `.gemspec` และโค้ดในตัว gem
  อ้างอิงค่าเดียวกันได้โดยไม่ต้องแก้เลขเวอร์ชันหลายที่ (Single Source of Truth)

```ruby
# ตัวอย่างเนื้อหาไฟล์ lib/my_gem/version.rb ทั่วไป
# frozen_string_literal: true

module MyGem
  VERSION = "0.1.0"
end
```

---

## Step 162: Version constraint ใน Gemfile เจาะลึก

Ruby ecosystem ใช้มาตรฐาน **Semantic Versioning (SemVer)** คือเลขเวอร์ชันรูปแบบ
`MAJOR.MINOR.PATCH` เช่น `7.1.3`:

- **MAJOR** (`7`) เพิ่มขึ้นเมื่อมีการเปลี่ยนแปลงที่ breaking change (โค้ดเดิมอาจพังถ้าอัปเดต)
- **MINOR** (`1`) เพิ่มขึ้นเมื่อเพิ่มฟีเจอร์ใหม่แบบ backward-compatible (ไม่พังของเดิม)
- **PATCH** (`3`) เพิ่มขึ้นเมื่อแก้บั๊กเล็กๆ น้อยๆ แบบ backward-compatible

ใน `Gemfile` เราระบุ "ช่วง" ของเวอร์ชันที่ยอมรับได้ผ่าน operator ต่างๆ ต่อไปนี้:

```ruby
# frozen_string_literal: true
source "https://rubygems.org"

# 1) ไม่ระบุเวอร์ชันเลย — ยอมรับเวอร์ชันล่าสุดที่มีตอน bundle install/update
gem "rails"

# 2) exact pin — ต้องเป็นเวอร์ชันนี้เท่านั้น เป๊ะๆ ห้ามมากกว่าหรือน้อยกว่า
gem "rails", "7.1.3"

# 3) >= (greater than or equal) — เวอร์ชันนี้ขึ้นไป ไม่มีเพดานบน (อันตราย เพราะอาจได้
#    เวอร์ชัน major ใหม่ที่ breaking change แบบไม่รู้ตัว)
gem "rails", ">= 7.0"

# 4) ช่วงแบบระบุทั้งขอบล่างและขอบบน
gem "rails", ">= 7.0", "< 8.0"

# 5) ~> (pessimistic operator / "twiddle-wakka") — นิยมใช้มากที่สุดในทางปฏิบัติ
gem "rails", "~> 7.1.0"   # หมายถึง >= 7.1.0 และ < 7.2.0 (ล็อกที่ PATCH เท่านั้นให้อัปเดตได้)
gem "rails", "~> 7.1"     # หมายถึง >= 7.1.0 และ < 8.0   (ล็อกที่ MINOR ให้ major เดียวกัน)
```

**กฎการอ่าน `~>` ให้จำง่ายๆ:** ตัวเลขตัวสุดท้ายที่ระบุคือตัวที่ "อนุญาตให้ขยับขึ้นได้"
ส่วนตัวที่อยู่ก่อนหน้าจะถูกล็อกไว้ไม่ให้เปลี่ยน

| เขียนใน Gemfile | แปลว่า | ตัวอย่างเวอร์ชันที่ยอมรับ |
|---|---|---|
| `"~> 7.1.3"` | `>= 7.1.3, < 7.2.0` | 7.1.3, 7.1.4, 7.1.99 (ไม่รวม 7.2.0) |
| `"~> 7.1"`   | `>= 7.1.0, < 8.0.0` | 7.1.0, 7.5.0, 7.99.0 (ไม่รวม 8.0.0) |
| `"~> 7"`     | `>= 7.0.0, < 8.0.0` | เหมือนบรรทัดบน (ระบุแค่ major) |

เราสามารถทดลองดูว่า constraint แต่ละแบบ resolve ออกมาเป็นเวอร์ชันไหนได้ด้วยคำสั่ง:

```bash
# ดูเวอร์ชันทั้งหมดของ gem ที่มีบน rubygems.org
gem list rails --remote --all | head -5

# ทดสอบด้วย Gem::Requirement โดยตรงใน irb — มีประโยชน์มากตอน debug constraint ซับซ้อน
irb -r rubygems
```

```irb
irb> req = Gem::Requirement.new("~> 7.1.0")
=> #<Gem::Requirement:0x00... @requirements=[[">=", #<Gem::Version "7.1.0">], ["<", #<Gem::Version "7.2.0">]]>

irb> req.satisfied_by?(Gem::Version.new("7.1.5"))
=> true
irb> req.satisfied_by?(Gem::Version.new("7.2.0"))
=> false
irb> req.satisfied_by?(Gem::Version.new("7.0.9"))
=> false
```

**แนวปฏิบัติในทีมมืออาชีพ:**

- ใช้ `~>` เป็นค่าเริ่มต้นแทบทุกกรณี เพราะยอมให้อัปเดต patch/minor version (bug fix,
  security patch) ได้อัตโนมัติตอน `bundle update` แต่กันไม่ให้ major version ใหม่ที่
  breaking change หลุดเข้ามาแบบไม่ตั้งใจ
- หลีกเลี่ยง `>=` แบบไม่มีเพดานบนใน production Gemfile เพราะควบคุมไม่ได้ว่าจะได้
  เวอร์ชันไหนในอนาคต
- exact pin (`"7.1.3"`) ใช้เฉพาะกรณีพิเศษ เช่น gem เวอร์ชันใหม่มีบั๊กที่กระทบระบบเรา
  โดยตรง และต้องการล็อกไว้ชัดเจนจนกว่าจะแก้บั๊กเสร็จ

---

## Step 163: Gemfile groups และ `require: false`

โปรเจกต์จริงมี gem ที่ใช้เฉพาะบางสถานการณ์ เช่น gem สำหรับ debug ไม่ควรถูกโหลดขึ้น
production server, gem สำหรับ test ไม่จำเป็นต้องติดตั้งบนเครื่อง production เลย
Bundler แก้ปัญหานี้ด้วย **group**

```ruby
# frozen_string_literal: true
source "https://rubygems.org"

gem "rails", "~> 7.1.0"
gem "pg", "~> 1.5"
gem "puma", "~> 6.4"

group :development do
  gem "web-console"          # console แบบ interactive ในหน้า error ของ Rails (dev เท่านั้น)
  gem "listen"                 # ใช้ auto-reload code เวลาไฟล์เปลี่ยน
end

group :test do
  gem "capybara"               # จำลอง browser สำหรับ system test
  gem "selenium-webdriver"
end

group :development, :test do
  gem "rspec-rails"            # ใช้ทั้งตอน dev (เขียนเทสต์) และตอน test (รันเทสต์บน CI)
  gem "pry-byebug"
  gem "rubocop", require: false
  gem "brakeman", require: false
end

group :production do
  gem "pg_stat_statements"     # เครื่องมือ monitor query เฉพาะ production
end
```

**เรื่อง `require: false` สำคัญมาก:** ปกติเมื่อ Bundler โหลด Gemfile ผ่าน
`Bundler.require` มันจะ `require` ทุก gem ที่ระบุไว้ให้อัตโนมัติตามชื่อ gem (แปลง `-`
เป็น `_`) แต่บาง gem อย่าง `rubocop` หรือ `brakeman` **ไม่ได้ถูกออกแบบมาให้ require
เข้า process ของแอปตอนรัน** — มันเป็นเครื่องมือที่เรียกผ่าน command line เท่านั้น
(`bundle exec rubocop`) การใส่ `require: false` บอก Bundler ว่า "ติดตั้ง gem นี้ไว้
ในกลุ่มที่ระบุ แต่อย่า require มันตอน `Bundler.require` อัตโนมัติ"

```ruby
# ตัวอย่างจากไฟล์ config/application.rb ของ Rails (ประมาณของจริง)
require_relative "boot"
require "rails/all"

Bundler.require(*Rails.groups)
# บรรทัดนี้จะ require เฉพาะ gem ในกลุ่มที่ตรงกับ Rails.env ปัจจุบัน
# (เช่นตอนรันด้วย RAILS_ENV=test จะโหลดกลุ่ม :default กับ :test เท่านั้น
#  ไม่โหลดกลุ่ม :development หรือ :production)
```

**คำสั่งที่เกี่ยวข้องกับ group:**

```bash
# ติดตั้งทุก gem ยกเว้นกลุ่ม production (ทำตอน setup เครื่อง dev)
bundle install --without production

# ตั้งค่านี้ให้จำไว้ถาวรในเครื่อง (เก็บไว้ที่ .bundle/config)
bundle config set --local without 'production'

# เมื่อ deploy ขึ้น production server ค่อยตั้งค่าตรงข้าม
bundle config set --local without 'development test'
bundle install
```

> **ข้อควรระวัง:** `bundle install --without` แค่ "ไม่ติดตั้ง" gem กลุ่มนั้น
> แต่ `Gemfile.lock` ยังคง**บันทึกไว้**ว่ากลุ่มไหนมี gem อะไรบ้าง (เพื่อให้เครื่องอื่นที่ไม่ได้
> ตั้ง `--without` ยัง install ครบได้) — Bundler ฉลาดพอที่จะจำ exclusion นี้ไว้ที่ไฟล์
> `.bundle/config` ของเครื่องนั้นๆ ไม่ใช่ใน `Gemfile.lock` เอง

---

## Step 164: `Gemfile.lock` คืออะไร และทำไมต้อง commit เข้า version control เสมอ

เมื่อรัน `bundle install` ครั้งแรก Bundler จะทำสิ่งที่เรียกว่า **dependency resolution**:
อ่าน constraint ทุกบรรทัดใน `Gemfile` (รวมถึง dependency ของ dependency แบบ transitive)
แล้วคำนวณหาชุดเวอร์ชันที่เข้ากันได้ทั้งหมดพอดี จากนั้นเขียนผลลัพธ์นั้นลงไฟล์ `Gemfile.lock`

```
# ตัวอย่างเนื้อหาบางส่วนของ Gemfile.lock
GEM
  remote: https://rubygems.org/
  specs:
    actioncable (7.1.3)
      actionpack (= 7.1.3)
      activesupport (= 7.1.3)
      nio4r (~> 2.0)
      websocket-driver (>= 0.6.1)
    actionpack (7.1.3)
      actionview (= 7.1.3)
      activesupport (= 7.1.3)
      rack (>= 2.2.4)
    ...

PLATFORMS
  x86_64-linux
  arm64-darwin-23

DEPENDENCIES
  pg (~> 1.5)
  puma (~> 6.4)
  rails (~> 7.1.0)
  rspec-rails

BUNDLED WITH
   2.5.6
```

สังเกตว่า `Gemfile.lock` เก็บ **เลขเวอร์ชันที่แน่นอนเป๊ะๆ** (`actioncable (7.1.3)` ไม่ใช่
`~> 7.1`) ของทุก gem ทั้งที่เราระบุเองใน `Gemfile` และที่เป็น transitive dependency
(gem ที่ gem อื่นต้องการ) รวมถึง platform ที่ resolve ไว้ (`x86_64-linux`,
`arm64-darwin-23`) และเวอร์ชัน Bundler ที่ใช้ตอน lock ครั้งล่าสุด

### ทำไม `Gemfile.lock` ต้อง commit เข้า git เสมอ

นี่คือหนึ่งในกฎที่**สำคัญที่สุด**ของการทำงานกับ Rails/Ruby ในทีม:

1. **Reproducible builds** — ถ้าไม่ commit lock file ทุกครั้งที่มีคน clone โปรเจกต์แล้วรัน
   `bundle install` Bundler จะต้อง resolve dependency ใหม่ทั้งหมดจาก constraint ใน
   `Gemfile` เพียวๆ ซึ่งอาจได้เวอร์ชันที่ต่างจากที่คนอื่นในทีมใช้อยู่ (เพราะเวลาผ่านไป
   มี gem เวอร์ชันใหม่ออกมาเรื่อยๆ) นำไปสู่บั๊กแบบ "โค้ดเดียวกัน แต่รันคนละพฤติกรรม"
2. **Dev/CI/Production ต้องเหมือนกัน 100%** — ถ้า production ใช้ gem เวอร์ชันต่างจากที่
   ทดสอบไว้ใน CI อาจเจอบั๊กที่ไม่เคยเจอตอน test เลยเมื่อ deploy จริง
3. **Security & audit** — เมื่อมี CVE (ช่องโหว่ความปลอดภัย) ประกาศออกมาสำหรับ gem
   เวอร์ชันใดเวอร์ชันหนึ่ง เราต้องรู้แน่ชัดว่าระบบเราใช้เวอร์ชันไหนอยู่ `Gemfile.lock`
   คือแหล่งความจริงเดียว (single source of truth) สำหรับเรื่องนี้
4. **Code review ที่มีความหมาย** — เมื่อมี pull request ที่แก้ `Gemfile.lock` เพื่อนร่วมทีม
   จะเห็น diff ชัดเจนว่า gem ตัวไหนเปลี่ยนจากเวอร์ชันอะไรเป็นอะไร ช่วยตรวจสอบ
   breaking change ได้ก่อน merge

```bash
# คำสั่งที่แก้ Gemfile.lock (ต้อง commit หลังรันเสมอ)
bundle install              # เพิ่ม gem ใหม่ที่เพิ่งใส่ใน Gemfile / อัปเดต lock ถ้าจำเป็น
bundle update               # อัปเดตทุก gem ให้เป็นเวอร์ชันล่าสุดที่ constraint อนุญาต
bundle update rails         # อัปเดตเฉพาะ rails (และ dependency ของมัน) ตัวเดียว
                             # ไม่กระทบ gem อื่นที่ไม่เกี่ยวข้อง — ควรใช้แบบนี้เจาะจง
                             # มากกว่า bundle update เปล่าๆ ในโปรเจกต์จริงที่โตแล้ว

# ตรวจสอบว่า Gemfile.lock ตรงกับ Gemfile หรือไม่ โดยไม่แก้ไขอะไร (ใช้บน CI)
bundle check

# ติดตั้งแบบ "ห้าม resolve ใหม่เด็ดขาด" ใช้เวอร์ชันใน lock file เป๊ะๆ เท่านั้น
# (เป็นค่า default ของ bundle install อยู่แล้วถ้า lock file ตรงกับ Gemfile)
bundle install --deployment  # (deprecated ใน bundler ใหม่ ใช้ bundle config set deployment true แทน)
```

> **กฎเหล็ก:** `.gitignore` ของโปรเจกต์ Ruby/Rails **ต้องไม่มี** `Gemfile.lock` อยู่ในนั้น
> เด็ดขาด (ต่างจาก `node_modules/` ที่ ignore ได้ แต่ `package-lock.json` ก็ควร commit
> เช่นกันด้วยเหตุผลเดียวกัน) มีเพียงกรณีเดียวที่ gem ตัว library (ไม่ใช่ Rails app)
> อาจเลือกไม่ commit lock file เพราะต้องการให้ทดสอบกับหลายเวอร์ชันของ dependency
> (เดี๋ยวเราจะเห็นว่า `bundle gem` ที่สร้าง gem ใหม่ default จะ .gitignore
> `Gemfile.lock` ไว้ให้ด้วยเหตุผลนี้ — แต่กฎนี้ใช้กับ**gem library** เท่านั้น
> ไม่ใช้กับ **Rails application** ที่เป็น end product)

---

## Step 165: `bundle exec` เจาะลึก — ทำไมรันคำสั่งตรงๆ อาจได้ gem ผิดเวอร์ชัน

Part 001 บอกไว้สั้นๆ ว่าควรใช้ `bundle exec` นำหน้าคำสั่งเสมอ Step นี้จะอธิบายว่า
**ทำไม** ถึงจำเป็นจริงๆ ในระดับกลไก

### ปัญหาที่เกิดขึ้นจริง

RubyGems อนุญาตให้ติดตั้ง gem **หลายเวอร์ชันพร้อมกัน**บนเครื่องเดียว (ต่างจาก
`pip` ของ Python ที่ปกติมีแค่เวอร์ชันเดียว active ต่อ environment) ลองดูตัวอย่าง:

```bash
# สมมติเครื่องเรามี rspec หลายเวอร์ชันติดตั้งอยู่ (เกิดขึ้นได้ง่ายมากตามอายุการใช้งาน)
gem list rspec-core
# *** LOCAL GEMS ***
# rspec-core (3.13.0, 3.12.2, 3.10.1)
```

ถ้าโปรเจกต์ A มี `Gemfile.lock` ล็อก `rspec-core (3.10.1)` ไว้ (เพราะเขียนไว้นานแล้ว
ยังไม่ได้อัปเดต) แต่เรารันคำสั่งตรงๆ โดยไม่ผ่าน bundler:

```bash
rspec spec/   # ไม่ใช้ bundle exec นำหน้า
```

`rspec` ในที่นี้คือ **executable ที่ RubyGems ติดตั้งไว้ใน PATH ของระบบ** (ไม่ใช่คำสั่ง
ของ Bundler) มันจะไปโหลด `rspec-core` เวอร์ชัน**ล่าสุดที่ติดตั้งอยู่บนเครื่อง**
(3.13.0 ในตัวอย่างนี้) ซึ่ง**ไม่ตรงกับที่ `Gemfile.lock` ล็อกไว้** ถ้า RSpec 3.13
มี syntax หรือ matcher ที่เปลี่ยนไปจาก 3.10 การรันเทสต์อาจ error แบบงงๆ, ได้ผลลัพธ์
ต่างจากที่ CI รัน (เพราะ CI ใช้ `bundle exec` เสมอ), หรือแย่กว่านั้นคือ**ผ่านบนเครื่องเรา
แต่ล้มเหลวบน CI** (หรือกลับกัน)

### `bundle exec` แก้ปัญหานี้อย่างไร

```bash
bundle exec rspec spec/
```

เมื่อสั่งแบบนี้ Bundler จะทำงานดังนี้ก่อนคำสั่งจริงจะรัน:

1. อ่าน `Gemfile.lock` ในโปรเจกต์ปัจจุบัน เพื่อรู้ว่าแต่ละ gem ต้องเป็นเวอร์ชันไหนเป๊ะๆ
2. เรียก `Bundler.setup` ซึ่งจะ**เขียนทับ** `$LOAD_PATH` ของ Ruby process ใหม่ทั้งหมด
   ให้ชี้ไปที่ path ของ gem เวอร์ชันที่ระบุใน lock file **เท่านั้น** (ตัด path ของ
   เวอร์ชันอื่นที่ติดตั้งอยู่บนเครื่องออกไปจากการค้นหา)
3. ถ้าโค้ดใดพยายาม `require` gem ที่ไม่ได้อยู่ใน `Gemfile` เลย (หรือพยายามโหลดเวอร์ชัน
   ที่ไม่ตรงกับ lock file) Bundler จะ raise `Gem::LoadError` ทันทีแทนที่จะปล่อยให้
   โหลดเวอร์ชันผิดแบบเงียบๆ
4. จากนั้นจึงรัน executable ที่ระบุ (`rspec`) ภายใต้ environment ที่ถูกควบคุมแล้วนี้

พูดง่ายๆ คือ **`bundle exec` = "รันคำสั่งนี้ ภายใต้ชุดเวอร์ชัน gem ที่ Gemfile.lock
กำหนดไว้เป๊ะๆ เท่านั้น ห้ามหลุดไปใช้เวอร์ชันอื่นที่ติดตั้งอยู่บนเครื่องเด็ดขาด"**

เราพิสูจน์ความแตกต่างนี้ได้ด้วยสคริปต์ทดลอง:

```ruby
# frozen_string_literal: true
# check_load_path.rb — รันทั้งแบบมีและไม่มี bundle exec เพื่อดูความต่าง

puts "Ruby load path entries ที่เกี่ยวกับ gem:"
$LOAD_PATH.grep(/gems/).first(5).each { |p| puts "  #{p}" }

begin
  gem "rspec-core", "= 3.10.1"
  puts "โหลด rspec-core 3.10.1 ได้สำเร็จ"
rescue Gem::LoadError => e
  puts "โหลดไม่ได้: #{e.message}"
end
```

```bash
ruby check_load_path.rb              # ไม่ผ่าน Bundler — เห็น path เวอร์ชันล่าสุดที่ระบบมี
bundle exec ruby check_load_path.rb  # ผ่าน Bundler — เห็น path เฉพาะเวอร์ชันใน lock file
```

### Binstub — ทางเลือกที่ไม่ต้องพิมพ์ `bundle exec` ทุกครั้ง

โปรเจกต์ Rails ที่สร้างด้วย `rails new` จะมีโฟลเดอร์ `bin/` พร้อม script อย่าง
`bin/rails`, `bin/rspec` ที่เรียกว่า **binstub** — เป็นสคริปต์ห่อ (wrapper) ที่ตั้งค่า
Bundler environment ให้อัตโนมัติอยู่แล้วในตัว ทำให้เรียก `bin/rails server` ได้ผลลัพธ์
เหมือน `bundle exec rails server` โดยไม่ต้องพิมพ์ `bundle exec` เอง

```bash
# สร้าง binstub เพิ่มเติมให้ gem ตัวอื่นที่ยังไม่มี (เช่น rspec ถ้ายังไม่มี bin/rspec)
bundle binstub rspec-core

# ตอนนี้เรียกได้ทั้งสองแบบ ผลลัพธ์เหมือนกัน
bin/rspec spec/
bundle exec rspec spec/
```

> **สรุปแนวปฏิบัติ:** ในโปรเจกต์ Rails ใช้ `bin/rails`, `bin/rspec` ที่มีมาให้อยู่แล้ว
> ส่วนคำสั่งอื่นที่ไม่มี binstub (เช่น `rubocop`, `rake`) ให้ใช้ `bundle exec` นำหน้า
> เป็นนิสัยเสมอ — การลืมใส่ `bundle exec` เป็นสาเหตุของบั๊ก "งานอยู่บนเครื่องฉัน" ที่พบ
> บ่อยที่สุดอันดับต้นๆ ในทีม Ruby/Rails

---

## Step 166: การระบุแหล่งที่มาของ gem — `path:`, `git:`, และ `github:` shorthand

ปกติ Bundler ดึง gem จาก RubyGems.org (ระบุด้วย `source "https://rubygems.org"`
บนสุดของ Gemfile) แต่บางสถานการณ์เราต้องการดึง gem จากแหล่งอื่น:

### `path:` — ใช้ gem ที่อยู่บนเครื่องตัวเอง (local development)

เหมาะกับตอนกำลังพัฒนา gem ของตัวเองไปพร้อมกับแอปที่ใช้ gem นั้น (monorepo หรือ
โฟลเดอร์ข้างๆ กัน) เพื่อทดสอบว่าการแก้ไข gem กระทบแอปอย่างไรแบบ real-time
โดยไม่ต้อง build/install gem ใหม่ทุกครั้ง

```ruby
# Gemfile ของแอป ที่กำลังพัฒนา gem thai_baht_formatter คู่กันอยู่
gem "thai_baht_formatter", path: "../thai_baht_formatter"
```

เมื่อใช้ `path:` Bundler จะโหลดโค้ดจากโฟลเดอร์นั้นตรงๆ ทุกครั้งที่รัน ไม่มีการ cache/copy
ไฟล์ ดังนั้นแก้โค้ดใน gem แล้ว restart แอป (หรือ reload console) ก็เห็นผลทันที

### `git:` — ดึง gem จาก git repository โดยตรง

เหมาะกับกรณี gem ยังไม่ได้ publish ขึ้น RubyGems.org หรือเราต้องการใช้โค้ด branch
ล่าสุดที่ยังไม่ออก release เป็นทางการ (หรือใช้ fork ของเราเองที่แก้บั๊กที่ยังไม่ได้ merge
เข้า upstream)

```ruby
# ใช้ branch ล่าสุดจาก main
gem "some_gem", git: "https://github.com/someone/some_gem.git", branch: "main"

# ใช้ tag ที่ระบุเจาะจง
gem "some_gem", git: "https://github.com/someone/some_gem.git", tag: "v2.0.0"

# ใช้ commit SHA ที่ระบุเจาะจงเป๊ะๆ (ใช้เมื่อต้องการความแน่นอนสูงสุด)
gem "some_gem", git: "https://github.com/someone/some_gem.git", ref: "a1b2c3d"
```

### `github:` — shorthand สำหรับ GitHub โดยเฉพาะ

Bundler มี syntax ย่อสำหรับ GitHub เพราะเป็นแหล่งที่ใช้บ่อยที่สุด:

```ruby
# เทียบเท่ากับ git: "https://github.com/rails/rails.git"
gem "rails", github: "rails/rails", branch: "main"

# ใช้ fork ของตัวเองที่แก้บั๊กเฉพาะทาง ระหว่างรอ PR ถูก merge เข้า upstream
gem "some_gem", github: "myusername/some_gem", branch: "fix-thai-locale-bug"
```

**Use case ที่พบบ่อยในทีมจริง:**

1. เจอบั๊กใน gem ที่ใช้อยู่ → fork gem นั้นบน GitHub → แก้บั๊ก → เปลี่ยน Gemfile ให้ชี้
   ไปที่ fork ของตัวเองชั่วคราว (`github: "ourteam/gem_name", branch: "our-fix"`) →
   ส่ง Pull Request กลับไปที่ upstream → พอ upstream merge และ release เวอร์ชันใหม่
   แล้ว ค่อยเปลี่ยน Gemfile กลับไปใช้ `gem "gem_name", "~> x.y"` แบบปกติ
2. ทีมกำลังพัฒนา internal gem (เช่น gem รวม logic ที่ใช้ร่วมกันหลายโปรเจกต์ในบริษัท)
   แต่ยังไม่พร้อม publish ขึ้น RubyGems.org (หรือตั้งใจให้เป็น private gem เท่านั้น)
   → ใช้ `git:` ชี้ไปที่ private repository ขององค์กร

```bash
# หลังแก้ Gemfile ให้ชี้ไปที่ path/git/github source ใหม่ ต้องรัน bundle install ใหม่เสมอ
bundle install
# Gemfile.lock จะบันทึก revision/commit SHA ที่ resolve ได้ไว้ด้วย ทำให้ทีมอื่นที่ bundle
# install ตามได้โค้ดจาก commit เดียวกันเป๊ะๆ
```

---

## Step 167: สร้าง gem ใหม่ด้วย `bundle gem` และสำรวจโครงสร้างที่ถูก generate

ถึงเวลาลงมือสร้าง gem จริงแล้ว Bundler มีคำสั่งสร้าง scaffold ของ gem ใหม่ให้อัตโนมัติ

```bash
mkdir -p ~/ruby-course-workspace/part-017
cd ~/ruby-course-workspace/part-017

bundle gem thai_baht_formatter
```

คำสั่งนี้จะถามคำถามแบบ interactive (ครั้งแรกที่รันบนเครื่องเท่านั้น หลังจากนั้น Bundler
จะจำค่าที่เลือกไว้ใน `~/.bundle/config`):

```
Do you want to generate tests with your gem?
Enter no, minitest, rspec or test-unit. rspec is the default: rspec

Do you want to license your code permissively under the MIT license?
This should match the licensing of the gems you depend on and the
purpose of your gem. See https://choosealicense.com/licenses/ for more info.
(default: no): yes

Do you want to include a code of conduct in gems you generate?
Contributor Covenant is the default. See https://www.contributor-covenant.org
for more info. (default: no): no

Do you want to include a changelog? (default: yes): yes

Do you want to add a linter and formatter to your gem's generated
Rakefile? no: none, rubocop: RuboCop, standard: Standard (default: rubocop): rubocop

Do you want to set up Continuous Integration (CI) for your gem?
Supported CI Templates: none, github, gitlab, circle (default: none): none
```

(ในตัวอย่างนี้ตอบ: rspec, yes สำหรับ MIT license, no สำหรับ code of conduct,
yes สำหรับ changelog, rubocop, none สำหรับ CI — เพื่อให้ output ใกล้เคียงกับด้านล่าง
ที่สุด สามารถตอบต่างจากนี้ได้ ผลลัพธ์จะต่างกันเล็กน้อยตามตัวเลือก)

ผลลัพธ์ที่ได้:

```bash
find thai_baht_formatter -type f | sort
```

```
thai_baht_formatter/.rspec
thai_baht_formatter/.rubocop.yml
thai_baht_formatter/CHANGELOG.md
thai_baht_formatter/Gemfile
thai_baht_formatter/LICENSE.txt
thai_baht_formatter/README.md
thai_baht_formatter/Rakefile
thai_baht_formatter/bin/console
thai_baht_formatter/bin/setup
thai_baht_formatter/lib/thai_baht_formatter.rb
thai_baht_formatter/lib/thai_baht_formatter/version.rb
thai_baht_formatter/sig/thai_baht_formatter.rbs
thai_baht_formatter/spec/spec_helper.rb
thai_baht_formatter/spec/thai_baht_formatter_spec.rb
thai_baht_formatter/thai_baht_formatter.gemspec
```

**อธิบายไฟล์สำคัญที่ generate มาให้:**

```ruby
# lib/thai_baht_formatter/version.rb (generate มาให้อัตโนมัติ)
# frozen_string_literal: true

module ThaiBahtFormatter
  VERSION = "0.1.0"
end
```

```ruby
# lib/thai_baht_formatter.rb (generate มาให้อัตโนมัติ — entry point)
# frozen_string_literal: true

require_relative "thai_baht_formatter/version"

module ThaiBahtFormatter
  class Error < StandardError; end
  # โค้ด implementation จริงของเราจะเพิ่มตรงนี้ใน Step 169
end
```

```ruby
# Gemfile ภายในตัวโปรเจกต์ gem เอง (คนละไฟล์กับ Gemfile ของแอปที่จะมาใช้ gem นี้)
# frozen_string_literal: true

source "https://rubygems.org"

# Specify your gem's dependencies in thai_baht_formatter.gemspec
gemspec

group :development do
  gem "rake", "~> 13.0"
  gem "rspec", "~> 3.13"
  gem "rubocop", "~> 1.21"
end
```

สังเกตว่า Gemfile ของตัว gem เองใช้คำสั่ง **`gemspec`** แทนที่จะระบุ `gem "..."`
ทีละบรรทัด — คำสั่งนี้บอกให้ Bundler อ่าน dependency ทั้งหมดจากไฟล์ `.gemspec` แทน
(ซึ่งจะเห็นรายละเอียดใน Step ถัดไป) นี่คือรูปแบบมาตรฐานสำหรับ Gemfile ของ gem ทุกตัว
เพราะ dependency ของ "ตัว gem เอง" (สิ่งที่คนติดตั้ง gem นี้ต้องมี) ควรประกาศไว้ใน
`.gemspec` เพียงที่เดียว ไม่ใช่ทั้งใน Gemfile และ gemspec ซ้ำซ้อนกัน

```ruby
# bin/console (generate มาให้อัตโนมัติ) — สคริปต์เปิด irb แบบโหลด gem ไว้ล่วงหน้า
#!/usr/bin/env ruby
# frozen_string_literal: true

require "bundler/setup"
require "thai_baht_formatter"

# You can add fixtures and/or initialization code here to make experimenting
# with your gem easier. You can also use a different console, if you like.

require "irb"
IRB.start(__FILE__)
```

```bash
# ทดลองใช้ bin/console เพื่อเข้า irb ที่ require gem ไว้ให้แล้ว
cd thai_baht_formatter
bin/console
```

```irb
irb(main):001> ThaiBahtFormatter::VERSION
=> "0.1.0"
```

---

## Step 168: เขียน metadata ใน `.gemspec` (name, version, summary, dependencies)

ไฟล์ `thai_baht_formatter.gemspec` คือ**พิมพ์เขียว**ของ gem ทั้งหมด มาดูเนื้อหาที่
generate มาให้ (บางส่วนถูก comment ไว้ให้เติมเอง) แล้วเติมให้สมบูรณ์:

```ruby
# thai_baht_formatter.gemspec
# frozen_string_literal: true

require_relative "lib/thai_baht_formatter/version"

Gem::Specification.new do |spec|
  spec.name = "thai_baht_formatter"
  spec.version = ThaiBahtFormatter::VERSION
  spec.authors = ["สมชาย ใจดี"]
  spec.email = ["somchai@example.com"]

  spec.summary = "แปลงตัวเลข Integer/Float ให้เป็นข้อความสกุลเงินบาทไทยที่อ่านง่าย"
  spec.description = <<~DESC
    ThaiBahtFormatter เป็น gem ขนาดเล็กสำหรับแปลงตัวเลข (Integer หรือ Float)
    ให้กลายเป็น string รูปแบบสกุลเงินบาทไทย เช่น 1234.5 -> "1,234.50 บาท"
    เหมาะสำหรับใช้แสดงราคาสินค้า, ยอดคำสั่งซื้อ, หรือรายงานทางการเงินในแอป
    ที่ใช้สกุลเงินบาทเป็นหลัก
  DESC
  spec.homepage = "https://github.com/somchai/thai_baht_formatter"
  spec.license = "MIT"
  spec.required_ruby_version = ">= 3.1.0"

  spec.metadata["homepage_uri"] = spec.homepage
  spec.metadata["source_code_uri"] = "#{spec.homepage}"
  spec.metadata["changelog_uri"] = "#{spec.homepage}/blob/main/CHANGELOG.md"

  # ระบุว่าไฟล์ไหนบ้างที่จะถูกรวมเข้าไปใน .gem package ตอน build
  # ใช้ `git ls-files` เพื่อดึงเฉพาะไฟล์ที่ commit ไว้ใน git (ไม่รวมไฟล์ temp/ignore)
  spec.files = Dir.chdir(__dir__) do
    `git ls-files -z`.split("\x0").reject do |f|
      (File.expand_path(f) == __FILE__) ||
        f.start_with?(*%w[bin/ test/ spec/ features/ .git .github appveyor Gemfile])
    end
  end

  spec.bindir = "exe"
  spec.executables = spec.files.grep(%r{\Aexe/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  # === Runtime dependency ===
  # gem ที่ "จำเป็นต้องมี" เพื่อให้ thai_baht_formatter ทำงานได้ตอนถูกใช้งานจริง
  # (คนที่ `gem install thai_baht_formatter` จะถูกติดตั้ง dependency เหล่านี้ตามมาด้วย)
  spec.add_dependency "bigdecimal", "~> 3.1"

  # === Development dependency ===
  # gem ที่ใช้เฉพาะตอน "พัฒนา gem นี้เอง" (เขียนเทสต์, lint) ไม่ถูกติดตั้งให้คนที่แค่
  # `gem install` มาใช้งาน — เดี๋ยวนี้นิยมประกาศไว้ใน Gemfile ของตัวโปรเจกต์ gem แทน
  # (ตามที่เห็นใน Step 167) แต่บาง gem รุ่นเก่ายังใช้ add_development_dependency ตรงนี้ก็ได้
  # spec.add_development_dependency "rspec", "~> 3.13"

  spec.metadata["rubygems_mfa_required"] = "true"
end
```

**อธิบายฟิลด์สำคัญทีละตัว:**

| ฟิลด์ | ความหมาย |
|---|---|
| `spec.name` | ชื่อ gem — ต้องตรงกับชื่อไฟล์ entry point ใน `lib/` |
| `spec.version` | เลขเวอร์ชัน ดึงมาจาก `VERSION` constant ใน `version.rb` (Single Source of Truth) |
| `spec.summary` | คำอธิบายสั้นบรรทัดเดียว โชว์ในผลการค้นหาบน RubyGems.org |
| `spec.description` | คำอธิบายละเอียดกว่า summary แสดงในหน้า gem บน RubyGems.org |
| `spec.homepage` | URL ของโปรเจกต์ (มักเป็น GitHub repo) |
| `spec.license` | ชื่อ license (ต้องเป็นชื่อมาตรฐานที่ SPDX รู้จัก เช่น `"MIT"`, `"Apache-2.0"`) |
| `spec.required_ruby_version` | เวอร์ชัน Ruby ขั้นต่ำที่ gem นี้รองรับ — RubyGems จะปฏิเสธการติดตั้งถ้า Ruby บนเครื่องต่ำกว่านี้ |
| `spec.files` | รายชื่อไฟล์ที่จะถูกรวมเข้า `.gem` package ตอน build (ปกติดึงจาก `git ls-files`) |
| `spec.require_paths` | โฟลเดอร์ที่จะถูกเพิ่มเข้า `$LOAD_PATH` เมื่อ gem ถูก activate — เกือบทุก gem ใช้ `["lib"]` |
| `spec.add_dependency` | ประกาศ **runtime dependency** — gem อื่นที่ gem เราต้องใช้ตอนทำงานจริง |
| `spec.add_development_dependency` | ประกาศ dependency ที่ใช้แค่ตอนพัฒนา/เทสต์ gem เอง |

**runtime dependency กับ development dependency ต่างกันอย่างไร ในทางปฏิบัติ:**

- `spec.add_dependency "bigdecimal", "~> 3.1"` → เมื่อมีคน `gem install
  thai_baht_formatter` หรือใส่ใน Gemfile ของแอปอื่น RubyGems/Bundler จะติดตั้ง
  `bigdecimal` ให้อัตโนมัติด้วย เพราะโค้ดจริงของเราต้องใช้มันตอนรัน
- `spec.add_development_dependency "rspec"` → dependency นี้จะถูกติดตั้งเฉพาะตอนที่
  เรา clone source code ของ **ตัว gem thai_baht_formatter เอง** มาพัฒนาต่อ (เช่น
  รัน `bundle install` ในโฟลเดอร์ gem) แต่จะ**ไม่**ถูกติดตั้งให้คนที่แค่เอา gem
  ไปใช้งานในโปรเจกต์อื่น (เพราะเขาไม่ได้ต้องการรันเทสต์ของตัว gem)

---

## Step 169: implement ฟังก์ชันการทำงานจริงในไฟล์ library ของ gem

ตอนนี้มาเขียน logic จริงของ `ThaiBahtFormatter` กัน โครงสร้างที่ generate มาให้มี
`lib/thai_baht_formatter.rb` เป็น entry point และ `lib/thai_baht_formatter/version.rb`
เก็บเลขเวอร์ชัน — เราจะเพิ่มไฟล์ `lib/thai_baht_formatter/formatter.rb` แยกต่างหาก
ตาม convention ที่ดี (แยก class/module ออกเป็นไฟล์ของตัวเอง แล้วค่อย `require` เข้ามา
รวมกันที่ entry point)

```ruby
# lib/thai_baht_formatter/formatter.rb
# frozen_string_literal: true

require "bigdecimal"

module ThaiBahtFormatter
  # แปลงตัวเลข (Integer หรือ Float) ให้เป็น string รูปแบบสกุลเงินบาทไทย
  # เช่น 1234.5 -> "1,234.50 บาท", -50 -> "-50.00 บาท"
  module Formatter
    module_function

    # @param amount [Integer, Float, BigDecimal] จำนวนเงินที่ต้องการ format
    # @param unit [String] หน่วยเงินที่จะต่อท้าย (default: "บาท")
    # @return [String] ข้อความที่ format แล้ว
    def to_baht(amount, unit: "บาท")
      unless amount.is_a?(Numeric)
        raise ArgumentError, "amount ต้องเป็นตัวเลข (Integer/Float/BigDecimal) เท่านั้น " \
                              "ได้รับ #{amount.class} มาแทน"
      end

      sign = amount.negative? ? "-" : ""
      whole, fraction = split_into_whole_and_fraction(amount.abs)

      "#{sign}#{group_thousands(whole)}.#{fraction} #{unit}"
    end

    def split_into_whole_and_fraction(positive_amount)
      # ปัดเศษให้เหลือ 2 ตำแหน่งเสมอ (สกุลเงินบาทมีหน่วยย่อยคือสตางค์ 2 หลัก)
      rounded = BigDecimal(positive_amount.to_s).round(2)
      whole_part = rounded.to_i
      fraction_part = ((rounded - whole_part) * 100).round.to_i
      [whole_part.to_s, format("%02d", fraction_part)]
    end
    private_class_method :split_into_whole_and_fraction

    # เติม comma คั่นหลักพัน เช่น "1234567" -> "1,234,567"
    def group_thousands(digits_string)
      digits_string.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
    end
    private_class_method :group_thousands
  end
end
```

```ruby
# lib/thai_baht_formatter.rb (แก้ไข entry point ให้ require ไฟล์ใหม่เข้ามา)
# frozen_string_literal: true

require_relative "thai_baht_formatter/version"
require_relative "thai_baht_formatter/formatter"

module ThaiBahtFormatter
  class Error < StandardError; end
end
```

ทดลองใช้งานผ่าน `bin/console`:

```bash
bin/console
```

```irb
irb(main):001> ThaiBahtFormatter::Formatter.to_baht(1234.5)
=> "1,234.50 บาท"
irb(main):002> ThaiBahtFormatter::Formatter.to_baht(1_000_000)
=> "1,000,000.00 บาท"
irb(main):003> ThaiBahtFormatter::Formatter.to_baht(-50)
=> "-50.00 บาท"
irb(main):004> ThaiBahtFormatter::Formatter.to_baht(99.999)
=> "100.00 บาท"
irb(main):005> ThaiBahtFormatter::Formatter.to_baht(1234.5, unit: "THB")
=> "1,234.50 THB"
irb(main):006> ThaiBahtFormatter::Formatter.to_baht("abc")
# ArgumentError: amount ต้องเป็นตัวเลข (Integer/Float/BigDecimal) เท่านั้น ได้รับ String มาแทน
```

**สิ่งที่ควรสังเกตในโค้ดนี้:**

- ใช้ `module_function` เพื่อให้ทุก method ในโมดูลเรียกใช้แบบ `Formatter.to_baht(...)`
  ได้ตรงๆ โดยไม่ต้อง `include` เข้า class ไหน (เหมาะกับ utility module ที่ไม่มี state)
- ใช้ `BigDecimal` แทน `Float` ตอนคำนวณเพื่อหลีกเลี่ยงปัญหา floating-point precision
  ของเงิน (เช่น `0.1 + 0.2` ใน Float ปกติจะได้ `0.30000000000000004` ซึ่งอันตรายมาก
  กับการคำนวณเงิน) — เหตุผลเดียวกับที่เราประกาศ `spec.add_dependency "bigdecimal"`
  ใน gemspec ก่อนหน้านี้
- validate input ด้วย `raise ArgumentError` ที่ข้อความชัดเจน เป็นแนวปฏิบัติที่ดีสำหรับ
  public API ของ gem ที่คนอื่นจะเอาไปใช้ (fail fast พร้อมข้อความบอกสาเหตุ)
- แยก private helper method ด้วย `private_class_method` เพื่อไม่ให้เป็นส่วนหนึ่งของ
  public API ที่คนภายนอกเรียกใช้ได้ (เผื่อในอนาคตอยากเปลี่ยน implementation ภายใน
  โดยไม่กระทบคนที่ใช้ gem นี้อยู่)

---

## Step 170: ทดสอบ gem บนเครื่องตัวเองก่อน publish

ก่อนจะปล่อย gem ออกสู่โลกภายนอก (ซึ่งเราจะเรียนวิธี publish ขึ้น RubyGems.org จริงใน
**Part 089**) ต้องทดสอบให้แน่ใจว่า gem ทำงานถูกต้องทั้งใน 2 ระดับ: (1) รันเทสต์ผ่าน
และ (2) ติดตั้งจากไฟล์ `.gem` ที่ build ออกมาแล้วใช้งานได้จริงเหมือนคนทั่วไปที่จะ
`gem install` ในอนาคต

### 1) เขียนและรันเทสต์ด้วย RSpec ภายในตัว gem

```ruby
# spec/spec_helper.rb (ส่วนใหญ่มากับ generate อยู่แล้ว)
# frozen_string_literal: true

require "thai_baht_formatter"

RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.example_status_persistence_file_path = ".rspec_status"
  config.disable_monkey_patching!

  config.default_formatter = "doc" if config.files_to_run.one?

  config.order = :random
  Kernel.srand config.seed
end
```

```ruby
# spec/thai_baht_formatter/formatter_spec.rb
# frozen_string_literal: true

RSpec.describe ThaiBahtFormatter::Formatter do
  describe ".to_baht" do
    it "แปลง Float ธรรมดาให้เป็นรูปแบบเงินบาทถูกต้อง" do
      expect(described_class.to_baht(1234.5)).to eq("1,234.50 บาท")
    end

    it "แปลง Integer ให้เป็นรูปแบบเงินบาทถูกต้อง" do
      expect(described_class.to_baht(500)).to eq("500.00 บาท")
    end

    it "เติม comma คั่นหลักพันเมื่อจำนวนเงินมีหลายหลัก" do
      expect(described_class.to_baht(1_000_000)).to eq("1,000,000.00 บาท")
    end

    it "จัดการจำนวนติดลบได้ถูกต้อง" do
      expect(described_class.to_baht(-50)).to eq("-50.00 บาท")
    end

    it "ปัดเศษทศนิยมตำแหน่งที่ 3 ขึ้นไปให้เหลือ 2 ตำแหน่ง" do
      expect(described_class.to_baht(99.999)).to eq("100.00 บาท")
    end

    it "รองรับการเปลี่ยนหน่วยเงินผ่าน keyword argument unit" do
      expect(described_class.to_baht(1234.5, unit: "THB")).to eq("1,234.50 THB")
    end

    it "raise ArgumentError เมื่อได้รับค่าที่ไม่ใช่ตัวเลข" do
      expect { described_class.to_baht("abc") }.to raise_error(
        ArgumentError, /amount ต้องเป็นตัวเลข/
      )
    end

    it "จัดการค่า 0 ได้ถูกต้อง" do
      expect(described_class.to_baht(0)).to eq("0.00 บาท")
    end
  end
end
```

รันเทสต์ **ต้องใช้ `bundle exec` เสมอ** (ตามที่เรียนใน Step 165 — gem ตัวโปรเจกต์เองก็
ไม่มีข้อยกเว้น):

```bash
cd thai_baht_formatter
bundle install          # ติดตั้ง dependency ตาม Gemfile (rspec, rubocop, rake)
bundle exec rspec
```

```
ThaiBahtFormatter::Formatter
  .to_baht
    แปลง Float ธรรมดาให้เป็นรูปแบบเงินบาทถูกต้อง
    แปลง Integer ให้เป็นรูปแบบเงินบาทถูกต้อง
    เติม comma คั่นหลักพันเมื่อจำนวนเงินมีหลายหลัก
    จัดการจำนวนติดลบได้ถูกต้อง
    ปัดเศษทศนิยมตำแหน่งที่ 3 ขึ้นไปให้เหลือ 2 ตำแหน่ง
    รองรับการเปลี่ยนหน่วยเงินผ่าน keyword argument unit
    raise ArgumentError เมื่อได้รับค่าที่ไม่ใช่ตัวเลข
    จัดการค่า 0 ได้ถูกต้อง

Finished in 0.01234 seconds (files took 0.15 seconds to load)
8 examples, 0 failures
```

`Rakefile` ที่ generate มาให้ตั้งค่า default task ให้รัน rspec (และ rubocop ถ้าเลือกไว้
ตอน `bundle gem`) อยู่แล้ว จึงเรียกสั้นๆ ได้ด้วย:

```bash
bundle exec rake        # เทียบเท่ากับรันทั้ง rspec และ rubocop ต่อกัน
```

### 2) Build ไฟล์ `.gem` และติดตั้งทดสอบบนเครื่องจริง

การรันผ่าน `bin/console` หรือ RSpec ยังเป็นการรันจาก **source code ตรงๆ**
เท่านั้น ยังไม่ได้พิสูจน์ว่า `.gemspec` ถูกต้องสมบูรณ์พอที่จะ package และติดตั้งจริงได้
ขั้นตอนสุดท้ายคือ build เป็นไฟล์ `.gem` แล้วลองติดตั้งเหมือนผู้ใช้งานทั่วไป:

```bash
# build .gem file จาก .gemspec — เทียบเท่ากับ bundle exec rake build
gem build thai_baht_formatter.gemspec
```

```
Successfully built RubyGem
  Name: thai_baht_formatter
  Version: 0.1.0
  File: thai_baht_formatter-0.1.0.gem
```

```bash
# ติดตั้ง .gem ไฟล์นี้แบบ local (ไม่ผ่าน RubyGems.org เลย) เหมือนดาวน์โหลดมาติดตั้งเอง
gem install ./thai_baht_formatter-0.1.0.gem

# ตรวจสอบว่าติดตั้งสำเร็จ และเป็นเวอร์ชันที่ถูกต้อง
gem list thai_baht_formatter
# *** LOCAL GEMS ***
# thai_baht_formatter (0.1.0)
```

ทดสอบใช้งานจริงจากนอกโฟลเดอร์โปรเจกต์ gem (เพื่อพิสูจน์ว่าไม่ได้พึ่งพา relative path
ใดๆ ของ source code เดิม แต่โหลดจาก gem ที่ติดตั้งในระบบจริงๆ):

```bash
cd /tmp
ruby -e 'require "thai_baht_formatter"; puts ThaiBahtFormatter::Formatter.to_baht(2_500.75)'
# => 2,500.75 บาท
```

ถ้าต้องการทดสอบว่าแอปอื่น (เช่นแอป Rails ที่กำลังพัฒนา) ใช้ gem นี้ได้จริงก่อน publish
สามารถใช้ `path:` ตามที่เรียนใน Step 166 ชี้ไปยังโฟลเดอร์ gem ได้เลยโดยไม่ต้อง build/
install ให้ยุ่งยาก:

```ruby
# Gemfile ของแอปอื่นที่ต้องการทดสอบ thai_baht_formatter ก่อน publish จริง
gem "thai_baht_formatter", path: "../thai_baht_formatter"
```

เมื่อทดสอบเสร็จหมดแล้วและอยากล้าง gem ที่ install ไว้ทดสอบออกจากเครื่อง:

```bash
gem uninstall thai_baht_formatter
```

> **หยุดตรงนี้ก่อน:** ขั้นตอนที่เหลือคือการ `gem push thai_baht_formatter-0.1.0.gem`
> เพื่อ publish ขึ้น RubyGems.org ให้คนทั้งโลกติดตั้งผ่าน `gem install
> thai_baht_formatter` ได้จริง — ส่วนนี้เกี่ยวข้องกับการสมัคร account บน
> RubyGems.org, การตั้งค่า API key, MFA (multi-factor authentication ที่บังคับ
> สำหรับ gem owner ทุกคนในปัจจุบัน), และแนวปฏิบัติเรื่อง semantic versioning ตอน
> release เวอร์ชันใหม่ๆ ซึ่งจะสอนละเอียดใน **Part 089 (เฟส 15: Rails Internals &
> Gem Building)** เมื่อเราพร้อมสร้าง gem ที่ซับซ้อนและเป็นประโยชน์ในระดับที่ควร
> แชร์ให้คนอื่นใช้จริง

---

## แบบฝึกหัดจบ Part: สร้าง gem `thai_baht_formatter` ให้สมบูรณ์

### โจทย์

จากทุก Step ด้านบน ให้รวบรวมเป็นโปรเจกต์ gem ที่สมบูรณ์ครบวงจร โดยมีข้อกำหนดเพิ่มเติม
จาก Step 169 ดังนี้:

1. เพิ่ม method `ThaiBahtFormatter::Formatter.to_baht_short` ที่แสดงจำนวนเงินแบบย่อ
   เมื่อมากกว่าหรือเท่ากับ 1 ล้าน โดยใช้หน่วย "ล้าน" (เช่น `2_500_000` →
   `"2.50 ล้านบาท"`) และถ้าน้อยกว่า 1 ล้านให้ fallback ไปเรียก `to_baht` ตามปกติ
2. เขียนเทสต์ครอบคลุม method ใหม่นี้ด้วย
3. Build เป็น `.gem` และติดตั้งทดสอบบนเครื่องจริงให้ผ่านสมบูรณ์

### เฉลย

```ruby
# lib/thai_baht_formatter/formatter.rb (ฉบับสมบูรณ์ พร้อม to_baht_short)
# frozen_string_literal: true

require "bigdecimal"

module ThaiBahtFormatter
  module Formatter
    module_function

    ONE_MILLION = 1_000_000

    def to_baht(amount, unit: "บาท")
      unless amount.is_a?(Numeric)
        raise ArgumentError, "amount ต้องเป็นตัวเลข (Integer/Float/BigDecimal) เท่านั้น " \
                              "ได้รับ #{amount.class} มาแทน"
      end

      sign = amount.negative? ? "-" : ""
      whole, fraction = split_into_whole_and_fraction(amount.abs)

      "#{sign}#{group_thousands(whole)}.#{fraction} #{unit}"
    end

    # แสดงจำนวนเงินแบบย่อเป็นหน่วย "ล้านบาท" เมื่อจำนวนมากกว่าหรือเท่ากับ 1,000,000
    # เช่น 2_500_000 -> "2.50 ล้านบาท", 999_999 -> "999,999.00 บาท" (fallback)
    def to_baht_short(amount)
      unless amount.is_a?(Numeric)
        raise ArgumentError, "amount ต้องเป็นตัวเลข (Integer/Float/BigDecimal) เท่านั้น " \
                              "ได้รับ #{amount.class} มาแทน"
      end

      return to_baht(amount) if amount.abs < ONE_MILLION

      sign = amount.negative? ? "-" : ""
      millions = BigDecimal(amount.abs.to_s) / ONE_MILLION
      "#{sign}#{format('%.2f', millions)} ล้านบาท"
    end

    def split_into_whole_and_fraction(positive_amount)
      rounded = BigDecimal(positive_amount.to_s).round(2)
      whole_part = rounded.to_i
      fraction_part = ((rounded - whole_part) * 100).round.to_i
      [whole_part.to_s, format("%02d", fraction_part)]
    end
    private_class_method :split_into_whole_and_fraction

    def group_thousands(digits_string)
      digits_string.reverse.gsub(/(\d{3})(?=\d)/, '\1,').reverse
    end
    private_class_method :group_thousands
  end
end
```

```ruby
# spec/thai_baht_formatter/formatter_spec.rb (เพิ่มส่วนทดสอบ to_baht_short)
# frozen_string_literal: true

RSpec.describe ThaiBahtFormatter::Formatter do
  describe ".to_baht" do
    it "แปลง Float ธรรมดาให้เป็นรูปแบบเงินบาทถูกต้อง" do
      expect(described_class.to_baht(1234.5)).to eq("1,234.50 บาท")
    end

    it "raise ArgumentError เมื่อได้รับค่าที่ไม่ใช่ตัวเลข" do
      expect { described_class.to_baht("abc") }.to raise_error(ArgumentError)
    end
  end

  describe ".to_baht_short" do
    it "แสดงเป็นหน่วยล้านบาทเมื่อจำนวนมากกว่าหรือเท่ากับ 1,000,000" do
      expect(described_class.to_baht_short(2_500_000)).to eq("2.50 ล้านบาท")
    end

    it "แสดงหน่วยล้านบาทพอดี 1 ล้านได้ถูกต้อง" do
      expect(described_class.to_baht_short(1_000_000)).to eq("1.00 ล้านบาท")
    end

    it "fallback ไปใช้ to_baht ปกติเมื่อจำนวนน้อยกว่า 1 ล้าน" do
      expect(described_class.to_baht_short(999_999)).to eq("999,999.00 บาท")
    end

    it "จัดการจำนวนติดลบระดับล้านได้ถูกต้อง" do
      expect(described_class.to_baht_short(-3_200_000)).to eq("-3.20 ล้านบาท")
    end

    it "raise ArgumentError เมื่อได้รับค่าที่ไม่ใช่ตัวเลข" do
      expect { described_class.to_baht_short(nil) }.to raise_error(ArgumentError)
    end
  end
end
```

รันเทสต์และ build เพื่อยืนยันว่าทุกอย่างทำงานถูกต้องครบวงจร:

```bash
cd thai_baht_formatter
bundle exec rspec
# 13 examples, 0 failures

gem build thai_baht_formatter.gemspec
# Successfully built RubyGem
#   Name: thai_baht_formatter
#   Version: 0.1.0
#   File: thai_baht_formatter-0.1.0.gem

gem install ./thai_baht_formatter-0.1.0.gem
ruby -e 'require "thai_baht_formatter"; puts ThaiBahtFormatter::Formatter.to_baht_short(15_750_000)'
# => 15.75 ล้านบาท
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม method `to_baht_text` ที่แปลงตัวเลขเป็น**คำอ่านภาษาไทยเต็มรูปแบบ**แบบที่ใช้
   เขียนเช็ค เช่น `1250` → `"หนึ่งพันสองร้อยห้าสิบบาทถ้วน"` (ใบ้: ต้องเขียนตารางแปลง
   หลักหน่วย-สิบ-ร้อย-พัน-หมื่น-แสน-ล้าน เป็นคำอ่านแยกกัน และจัดการกรณีพิเศษของภาษาไทย
   เช่น "ยี่สิบ" แทน "สองสิบ", "เอ็ด" แทน "หนึ่ง" ในหลักหน่วยเมื่อมีหลักสิบนำหน้า)
2. เพิ่ม support ให้ gem นี้ format สกุลเงินอื่นได้ด้วยผ่าน parameter `locale:`
   เช่น `to_baht(1234.5, locale: :en)` ให้ผลลัพธ์เป็น `"THB 1,234.50"` แบบ
   international format แทนที่จะเป็นภาษาไทยเสมอ
3. เพิ่ม executable command line ให้ gem นี้ (ไฟล์ใน `exe/thai_baht_formatter`
   ตามที่ `spec.bindir = "exe"` เตรียมไว้ใน gemspec แล้ว) ให้เรียกจาก terminal
   ตรงๆ ได้ เช่น `thai_baht_formatter 1234.5` แล้วพิมพ์ `1,234.50 บาท` ออกทาง
   stdout — ทดสอบด้วยการ `gem build` + `gem install` แล้วเรียก command นั้นจาก
   terminal จริง (ใบ้: ต้องเพิ่ม shebang `#!/usr/bin/env ruby` บนสุดของไฟล์ และ
   `chmod +x exe/thai_baht_formatter`)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า gem คือไฟล์ archive `.gem` ที่บรรจุ metadata + source code และรู้จัก
  โครงสร้างไดเรกทอรีมาตรฐาน (`lib/`, `.gemspec`, `bin/`, `spec/`) ที่ทุก gem ใช้ร่วมกัน
- อ่านและเขียน version constraint ใน Gemfile ได้อย่างแม่นยำ ทั้ง `~>`, `>=`, และ
  exact pin พร้อมรู้ว่าควรเลือกใช้แบบไหนในสถานการณ์จริง
- ใช้ Gemfile groups (`:development`, `:test`, `:production`) และ `require: false`
  แยก dependency ตามสภาพแวดล้อมได้ถูกต้อง
- เข้าใจกลไกเบื้องหลัง `Gemfile.lock` และเหตุผลที่**ต้อง**commit เข้า version control
  เสมอเพื่อ reproducible build
- เข้าใจกลไกเบื้องหลัง `bundle exec` ในระดับ `$LOAD_PATH` และรู้ว่าทำไมการรันคำสั่ง
  โดยไม่ผ่าน Bundler อาจได้ gem ผิดเวอร์ชันแบบเงียบๆ
- ระบุแหล่งที่มาของ gem แบบ `path:`, `git:`, `github:` ได้ สำหรับ workflow การพัฒนา
  gem คู่ขนานกับแอป หรือใช้ fork ระหว่างรอ PR merge
- สร้าง gem ใหม่ตั้งแต่ศูนย์ด้วย `bundle gem`, เขียน `.gemspec` ที่มี metadata และ
  dependency ครบถ้วน, implement logic จริง, เขียนเทสต์ด้วย RSpec, และ build +
  install `.gem` ไฟล์เพื่อทดสอบบนเครื่องจริงก่อน publish

**ต่อไป (Part 018):** เราจะเรียนรู้การเขียนเทสต์ด้วย **Minitest** ซึ่งเป็น testing
framework มาตรฐานที่มากับ Ruby เอง (ต่างจาก RSpec ที่เป็น gem แยกต่างหาก) ครอบคลุม
`unit test`, `assertion` แบบต่างๆ, และการทำ `test doubles` (mock/stub) เบื้องต้น —
ความรู้เรื่อง gem/Bundler จาก Part นี้จะเป็นพื้นฐานสำคัญ เพราะทั้ง Minitest และ RSpec
(Part 019) ก็เป็น gem ที่ถูกจัดการผ่าน Bundler เหมือนกันทั้งคู่
