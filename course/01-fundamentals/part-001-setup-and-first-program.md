# Part 001: เริ่มต้นกับ Ruby — ติดตั้งสภาพแวดล้อม และโปรแกรมแรก

> **Step ครอบคลุมใน Part นี้:** Step 1–10
> **ระดับ:** เริ่มต้น (ไม่ต้องมีพื้นฐานเขียนโปรแกรมมาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 1: Ruby คืออะไร และทำไมต้องเรียน Ruby on Rails
- Step 2: ติดตั้ง Ruby ด้วย `rbenv` (macOS/Linux)
- Step 3: ติดตั้ง Ruby ด้วย `asdf` และทางเลือกบน Windows (WSL)
- Step 4: ตรวจสอบการติดตั้ง และทำความรู้จัก `gem`, `bundler`
- Step 5: `irb` และ `pry` — เครื่องมือทดลองโค้ดแบบ interactive
- Step 6: เขียนโปรแกรม Hello World แบบไฟล์ `.rb`
- Step 7: การรันไฟล์ Ruby และการรับ command line arguments
- Step 8: Comment, `puts`, `print`, `p` ต่างกันอย่างไร
- Step 9: ตั้งค่า Editor (VS Code) และ RuboCop เบื้องต้น
- Step 10: แบบฝึกหัดโปรเจกต์แรก — โปรแกรมทักทายและคำนวณอายุ

---

## Step 1: Ruby คืออะไร และทำไมต้องเรียน Ruby on Rails

**Ruby** เป็นภาษาโปรแกรมมิ่งที่สร้างโดย Yukihiro "Matz" Matsumoto ชาวญี่ปุ่น เปิดตัวปี 1995
ปรัชญาหลักของ Ruby คือ **"Optimize for programmer happiness"** — ออกแบบให้เขียนโค้ดได้
อ่านง่าย เป็นธรรมชาติเหมือนภาษาอังกฤษ และสนุกกับการเขียนโปรแกรม

**Ruby on Rails** (หรือเรียกสั้นๆ ว่า Rails หรือ RoR) คือ Web Application Framework ที่เขียนด้วย
Ruby เปิดตัวปี 2004 โดย David Heinemeier Hansson (DHH) มีจุดเด่นคือ:

1. **Convention over Configuration (CoC)** — Rails มีข้อกำหนดมาตรฐานที่ทำให้ไม่ต้อง config
   ทุกอย่างเอง ถ้าตั้งชื่อไฟล์/class ตามข้อกำหนด ระบบจะทำงานได้ทันที
2. **Don't Repeat Yourself (DRY)** — หลีกเลี่ยงการเขียนโค้ดซ้ำซ้อน
3. **MVC Architecture** — แยกส่วน Model, View, Controller อย่างชัดเจน
4. **Rich Ecosystem** — มี gem (library) ให้ใช้งานจำนวนมหาศาล
5. **Rapid Development** — สร้างเว็บแอปพลิเคชันได้เร็วกว่า framework อื่นๆ มาก

บริษัทระดับโลกที่ใช้ Rails ในการสร้างผลิตภัณฑ์หลัก: GitHub, Shopify, Airbnb (ช่วงแรก),
Basecamp, Hey.com, Cookpad, Zendesk

### ทำไมต้องเรียนหลักสูตรนี้

หลักสูตรนี้ออกแบบมาให้พาไปตั้งแต่ไม่รู้อะไรเลย จนถึงระดับที่สามารถ:

- ออกแบบและสร้างเว็บแอปพลิเคชันระดับ production ได้ด้วยตัวเอง
- เข้าใจสถาปัตยกรรมซอฟต์แวร์ระดับองค์กร (enterprise-grade)
- ทำงานร่วมกับทีมพัฒนาในระดับมืออาชีพ (code review, testing, CI/CD)
- เตรียมพร้อมสำหรับตำแหน่งงานระดับ Senior/Staff Engineer

---

## Step 2: ติดตั้ง Ruby ด้วย `rbenv` (macOS/Linux)

เราไม่ติดตั้ง Ruby ที่มากับระบบปฏิบัติการ (system Ruby) โดยตรง เพราะโปรเจกต์ต่างๆ อาจต้องการ
Ruby คนละเวอร์ชันกัน จึงต้องใช้ **version manager** เพื่อสลับเวอร์ชัน Ruby ได้ตามต้องการ

`rbenv` เป็น version manager ที่เบาและนิยมมากที่สุดตัวหนึ่ง

### ติดตั้งบน macOS (ด้วย Homebrew)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง rbenv และ ruby-build plugin
brew install rbenv ruby-build

# เพิ่ม rbenv เข้า shell (สมมติใช้ zsh ซึ่งเป็น default shell ของ macOS)
echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
source ~/.zshrc

# ตรวจสอบว่าติดตั้งถูกต้อง
rbenv -v
```

### ติดตั้งบน Ubuntu/Debian Linux

```bash
# ติดตั้ง dependencies ที่จำเป็นสำหรับ compile Ruby
sudo apt update
sudo apt install -y git curl libssl-dev libreadline-dev zlib1g-dev \
  autoconf bison build-essential libyaml-dev libreadline-dev \
  libncurses5-dev libffi-dev libgdbm-dev

# ติดตั้ง rbenv ผ่าน git
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# ติดตั้ง ruby-build plugin
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

rbenv -v
```

### ติดตั้ง Ruby เวอร์ชันล่าสุดด้วย rbenv

```bash
# ดู list เวอร์ชันที่ติดตั้งได้
rbenv install -l

# ติดตั้ง Ruby 3.3.5 (หรือเวอร์ชันล่าสุดที่ rbenv install -l แสดง)
rbenv install 3.3.5

# ตั้งเป็นเวอร์ชัน global (ใช้ทั่วทั้งเครื่อง)
rbenv global 3.3.5

# ตรวจสอบ
ruby -v
# => ruby 3.3.5 (2024-09-03 revision ef084cc8f4) [x86_64-linux]
```

> **หมายเหตุ:** การ compile Ruby จาก source ใช้เวลาสักครู่ (5–15 นาที ขึ้นอยู่กับสเปคเครื่อง)
> เป็นเรื่องปกติ ไม่ต้องตกใจ

---

## Step 3: ติดตั้ง Ruby ด้วย `asdf` และทางเลือกบน Windows (WSL)

### ทางเลือก: `asdf` (version manager แบบ universal ใช้ได้กับหลายภาษา)

`asdf` เหมาะกับคนที่ทำงานหลายภาษา (Ruby, Node.js, Python, ฯลฯ) เพราะจัดการทุกภาษาผ่าน
เครื่องมือเดียว

```bash
# macOS
brew install asdf
echo -e "\n. $(brew --prefix asdf)/libexec/asdf.sh" >> ~/.zshrc
source ~/.zshrc

# เพิ่ม ruby plugin
asdf plugin add ruby https://github.com/asdf-vm/asdf-ruby.git

# ติดตั้ง Ruby
asdf install ruby 3.3.5
asdf global ruby 3.3.5

ruby -v
```

แต่ละโปรเจกต์สามารถกำหนดเวอร์ชันของตัวเองได้ด้วยไฟล์ `.tool-versions` ที่ root ของโปรเจกต์:

```
# .tool-versions
ruby 3.3.5
```

เมื่อ `cd` เข้าโฟลเดอร์ที่มีไฟล์นี้ `asdf` จะสลับเวอร์ชัน Ruby ให้อัตโนมัติ (ถ้าใช้ `rbenv`
ไฟล์ที่เทียบเท่ากันคือ `.ruby-version`)

### ติดตั้งบน Windows

Rails แนะนำให้ใช้ **WSL (Windows Subsystem for Linux)** แทนการติดตั้ง Ruby บน Windows
โดยตรง เพราะ native gem บางตัว (เช่น ที่ต้อง compile C extension) มีปัญหาบน Windows บ่อย

```powershell
# เปิด PowerShell แบบ Administrator แล้วรัน
wsl --install -d Ubuntu

# รีสตาร์ทเครื่อง แล้วเปิด Ubuntu (WSL) ขึ้นมา
# จากนั้นทำตามขั้นตอน "ติดตั้งบน Ubuntu/Debian Linux" ด้านบนได้เลย
```

---

## Step 4: ตรวจสอบการติดตั้ง และทำความรู้จัก `gem`, `bundler`

หลังติดตั้ง Ruby เสร็จแล้ว ให้ตรวจสอบเครื่องมือที่มาพร้อมกัน

```bash
ruby -v      # เวอร์ชันของ Ruby interpreter
gem -v       # เวอร์ชันของ RubyGems (package manager ของ Ruby)
irb -v       # เวอร์ชันของ irb (interactive Ruby)
```

### RubyGems คืออะไร

**Gem** คือหน่วยของ library/package ใน Ruby (เทียบเท่า npm package ใน Node.js หรือ
pip package ใน Python) `gem` คือคำสั่ง command line สำหรับติดตั้ง/จัดการ gem

```bash
# ติดตั้ง gem ตัวหนึ่ง
gem install bundler

# ดู gem ที่ติดตั้งแล้วทั้งหมด
gem list

# ดูข้อมูล gem ตัวหนึ่ง
gem info bundler

# ค้นหา gem จาก rubygems.org
gem search rails
```

### Bundler คืออะไร

**Bundler** เป็นเครื่องมือจัดการ dependency ของโปรเจกต์ Ruby โดยอ่านจากไฟล์ `Gemfile`
และล็อกเวอร์ชันที่แน่นอนไว้ในไฟล์ `Gemfile.lock` (คล้าย `package-lock.json` หรือ
`requirements.txt`)

```bash
gem install bundler
bundler -v
```

ตัวอย่างไฟล์ `Gemfile` ง่ายๆ:

```ruby
# Gemfile
source "https://rubygems.org"

gem "rails", "~> 7.1.0"
gem "pg"
gem "puma"

group :development, :test do
  gem "rspec-rails"
  gem "pry"
end
```

คำสั่งที่ใช้บ่อย:

```bash
bundle install     # ติดตั้ง gem ทั้งหมดตาม Gemfile
bundle update       # อัปเดต gem ให้เป็นเวอร์ชันล่าสุดที่ constraint อนุญาต
bundle exec <cmd>   # รันคำสั่งภายใต้ environment ของ Bundler (ป้องกันปัญหาเวอร์ชันชนกัน)
```

> **แนวคิดสำคัญ:** ทุกครั้งที่รันคำสั่งเกี่ยวกับ Rails ในโปรเจกต์จริง ควรใช้ `bundle exec`
> นำหน้าเสมอ เช่น `bundle exec rails server` เพื่อให้แน่ใจว่าใช้เวอร์ชัน gem ตรงกับที่ระบุ
> ใน `Gemfile.lock` ไม่ใช่เวอร์ชันอื่นที่ติดตั้งแบบ global

---

## Step 5: `irb` และ `pry` — เครื่องมือทดลองโค้ดแบบ interactive

**irb** (Interactive Ruby) เป็น REPL (Read-Eval-Print Loop) ที่มาพร้อมกับ Ruby ใช้สำหรับ
ทดลองรันโค้ด Ruby ทีละบรรทัดได้ทันทีโดยไม่ต้องสร้างไฟล์

```bash
irb
```

```irb
irb(main):001> 1 + 1
=> 2
irb(main):002> "hello".upcase
=> "HELLO"
irb(main):003> [1, 2, 3].sum
=> 6
irb(main):004> exit
```

### pry — REPL ที่ทรงพลังกว่า

`pry` เป็น gem ทางเลือกที่ดีกว่า irb มาก มี syntax highlighting, การดู source code ของ method,
และใช้เป็น debugger ได้ (เดี๋ยวเราจะใช้บ่อยมากตอนดีบัก Rails app)

```bash
gem install pry
pry
```

```irb
[1] pry(main)> def greet(name)
[1] pry(main)*   "Hello, #{name}!"
[1] pry(main)* end
=> :greet
[2] pry(main)> greet("Ruby")
=> "Hello, Ruby!"
[3] pry(main)> show-method greet
```

จุดเด่นของ pry ที่จะใช้บ่อยในหลักสูตรนี้คือคำสั่ง `binding.pry` ที่แทรกลงในโค้ดเพื่อหยุด
โปรแกรม ณ จุดนั้นแล้วเข้าสู่ interactive session (คล้าย breakpoint) — จะสอนละเอียดใน
Part ที่พูดถึงการ debug

---

## Step 6: เขียนโปรแกรม Hello World แบบไฟล์ `.rb`

สร้างโฟลเดอร์สำหรับเก็บโค้ดฝึกหัดของหลักสูตรนี้:

```bash
mkdir -p ~/ruby-course-workspace/part-001
cd ~/ruby-course-workspace/part-001
```

สร้างไฟล์ `hello.rb`:

```ruby
# hello.rb
puts "Hello, Ruby on Rails!"
```

รันด้วยคำสั่ง:

```bash
ruby hello.rb
# => Hello, Ruby on Rails!
```

### กายวิภาคของโปรแกรมนี้

- `puts` เป็น method ที่มากับ Ruby (ย่อมาจาก "put string") ใช้พิมพ์ข้อความออกทาง
  standard output พร้อมขึ้นบรรทัดใหม่ต่อท้ายอัตโนมัติ
- `"Hello, Ruby on Rails!"` คือ String literal
- Ruby ไม่ต้องมี semicolon `;` ปิดท้ายบรรทัด (แต่ใส่ได้ถ้าต้องการเขียนหลายคำสั่งในบรรทัด
  เดียวกัน)
- Ruby ไม่ต้องมี `main` function เหมือน C/Java — โค้ดระดับบนสุด (top-level) จะถูกรันทันที
  จากบนลงล่าง

---

## Step 7: การรันไฟล์ Ruby และการรับ command line arguments

### รันไฟล์แบบระบุ argument

สร้างไฟล์ `greet.rb`:

```ruby
# greet.rb
name = ARGV[0] || "World"
puts "Hello, #{name}!"
```

```bash
ruby greet.rb
# => Hello, World!

ruby greet.rb Ruby
# => Hello, Ruby!
```

**อธิบาย:**

- `ARGV` เป็น Array ที่เก็บ command line arguments ทั้งหมดที่ส่งเข้ามาตอนรันโปรแกรม
- `ARGV[0]` คือ argument ตัวแรก ถ้าไม่มีจะได้ `nil`
- `nil || "World"` ใช้ operator `||` (or) เพื่อกำหนดค่า default เมื่อ `ARGV[0]` เป็น `nil`
  (หรือ falsy)
- `"Hello, #{name}!"` คือ string interpolation — แทรกค่าตัวแปรลงใน string ด้วย `#{}`

### รับ input จากผู้ใช้ระหว่างรันโปรแกรม

```ruby
# ask_name.rb
print "กรุณาใส่ชื่อของคุณ: "
name = gets.chomp
puts "ยินดีต้อนรับ, #{name}!"
```

```bash
ruby ask_name.rb
# กรุณาใส่ชื่อของคุณ: สมชาย
# ยินดีต้อนรับ, สมชาย!
```

**อธิบาย:**

- `gets` อ่านข้อมูล 1 บรรทัดจาก standard input (รวม newline character ต่อท้าย)
- `.chomp` ตัด newline character ที่ท้ายสุดออก (ถ้าไม่ตัด `name` จะมี `\n` ติดมาด้วย ทำให้
  ผลลัพธ์ผิดเพี้ยน)
- `print` ต่างจาก `puts` ตรงที่ไม่ขึ้นบรรทัดใหม่ให้อัตโนมัติ

---

## Step 8: Comment, `puts`, `print`, `p` ต่างกันอย่างไร

### Comment ใน Ruby

```ruby
# นี่คือ comment บรรทัดเดียว

=begin
นี่คือ comment
แบบหลายบรรทัด
(ไม่นิยมใช้ในทางปฏิบัติ ส่วนใหญ่ใช้ # ซ้ำหลายบรรทัดแทน)
=end

# แนวปฏิบัติที่ดี: ใช้ # สำหรับทุก comment เพื่อความสม่ำเสมอ
# และ comment ควรอธิบาย "ทำไม" มากกว่า "ทำอะไร" (โค้ดที่ดีอธิบายตัวเองได้ว่าทำอะไร)
```

### ความแตกต่างของ `puts`, `print`, `p`

```ruby
puts "hello"     # => hello   (ขึ้นบรรทัดใหม่ให้อัตโนมัติ, คืนค่า nil)
print "hello"    # => hello   (ไม่ขึ้นบรรทัดใหม่, คืนค่า nil)
p "hello"        # => "hello" (แสดงผลแบบ inspect คือมี "" ครอบ, คืนค่าคือ object ที่ print)

puts nil         # แสดงบรรทัดว่างเปล่า
print nil        # ไม่แสดงอะไรเลย
p nil            # => nil

puts [1, 2, 3]   # แสดงทีละบรรทัด: 1 / 2 / 3
p [1, 2, 3]      # แสดง: [1, 2, 3]  (เห็นโครงสร้าง Array ชัดเจน)
```

**กฎการเลือกใช้ในทางปฏิบัติ:**

- ใช้ `puts` เมื่อต้องการแสดงข้อความให้ผู้ใช้อ่าน (human-readable output)
- ใช้ `p` เมื่อกำลัง debug และต้องการเห็นโครงสร้างข้อมูลจริงๆ (เช่น แยกแยะ `nil` กับ
  string ว่าง `""`, หรือดูว่า string มี whitespace แฝงอยู่หรือไม่)
- ใช้ `print` เมื่อต้องการควบคุมการขึ้นบรรทัดใหม่เอง เช่น การแสดง progress bar

```ruby
# ตัวอย่างที่แสดงให้เห็นความสำคัญของ p ในการ debug
value1 = nil
value2 = ""
value3 = " "

puts value1  # (บรรทัดว่าง) - ดูเหมือนกันหมด แยกไม่ออก
puts value2  # (บรรทัดว่าง)
puts value3  # (บรรทัดว่างที่มีช่องว่าง)

p value1  # => nil            - เห็นชัดว่าเป็น nil
p value2  # => ""             - เห็นชัดว่าเป็น string ว่าง
p value3  # => " "            - เห็นชัดว่ามี whitespace
```

---

## Step 9: ตั้งค่า Editor (VS Code) และ RuboCop เบื้องต้น

### ติดตั้ง VS Code Extension สำหรับ Ruby

แนะนำ extension ต่อไปนี้ใน VS Code:

1. **Ruby LSP** (โดย Shopify) — ให้ autocomplete, go-to-definition, syntax checking
2. **Ruby** (โดย Peng Lv) — syntax highlighting พื้นฐาน
3. **endwise** — เติม `end` ให้อัตโนมัติเมื่อเปิด block

ติดตั้งผ่าน command line:

```bash
code --install-extension Shopify.ruby-lsp
```

### RuboCop — Linter/Formatter มาตรฐานของวงการ Ruby

RuboCop คือเครื่องมือตรวจสอบ code style ให้ตรงตาม Ruby community style guide
เป็นเครื่องมือที่ทีมพัฒนา Ruby/Rails มืออาชีพเกือบทุกทีมใช้

```bash
gem install rubocop

# สร้างไฟล์ config เริ่มต้น
touch .rubocop.yml
```

ตัวอย่างไฟล์ `.rubocop.yml` เบื้องต้น:

```yaml
AllCops:
  NewCops: enable
  TargetRubyVersion: 3.3

Style/Documentation:
  Enabled: false

Layout/LineLength:
  Max: 120

Metrics/MethodLength:
  Max: 15
```

รันตรวจสอบโค้ด:

```bash
rubocop hello.rb

# แก้ไขปัญหาที่แก้ได้อัตโนมัติ
rubocop -A hello.rb
```

ตัวอย่างผลลัพธ์:

```
Inspecting 1 file
W

Offenses:

hello.rb:1:1: C: [Correctable] Style/FrozenStringLiteralComment: Missing
frozen string literal comment.
puts "Hello, Ruby on Rails!"
^

1 file inspected, 1 offense detected, 1 offense auto-correctable
```

> **แนวคิด `frozen_string_literal`:** การเพิ่มคอมเมนต์ `# frozen_string_literal: true`
> ที่บรรทัดแรกสุดของไฟล์ ทำให้ string literal ทุกตัวในไฟล์นั้นเป็น immutable (แก้ไขค่าไม่ได้)
> ช่วยเพิ่มประสิทธิภาพและป้องกันบัคจากการแก้ไข string โดยไม่ตั้งใจ — เป็นธรรมเนียมปฏิบัติ
> มาตรฐานในโค้ด Ruby ยุคปัจจุบัน เราจะใส่บรรทัดนี้ไว้บนสุดของทุกไฟล์ Ruby ตั้งแต่นี้ไป

```ruby
# frozen_string_literal: true

puts "Hello, Ruby on Rails!"
```

---

## Step 10: แบบฝึกหัดโปรเจกต์แรก — โปรแกรมทักทายและคำนวณอายุ

### โจทย์

เขียนโปรแกรม `profile.rb` ที่ทำงานดังนี้:

1. ถามชื่อผู้ใช้
2. ถามปีเกิด (พ.ศ. หรือ ค.ศ. ก็ได้ แต่ในตัวอย่างนี้ใช้ ค.ศ.)
3. คำนวณอายุปัจจุบัน (ปีปัจจุบัน - ปีเกิด)
4. แสดงข้อความทักทายพร้อมอายุ

### เฉลย

```ruby
# frozen_string_literal: true

# profile.rb
require "date"

print "กรุณาใส่ชื่อของคุณ: "
name = gets.chomp

print "กรุณาใส่ปีเกิดของคุณ (ค.ศ.): "
birth_year = gets.chomp.to_i

current_year = Date.today.year
age = current_year - birth_year

puts "-" * 40
puts "สวัสดี, #{name}!"
puts "ปีนี้คุณอายุประมาณ #{age} ปี"
puts "-" * 40
```

ทดสอบรัน:

```bash
ruby profile.rb
# กรุณาใส่ชื่อของคุณ: มานี
# กรุณาใส่ปีเกิดของคุณ (ค.ศ.): 1998
# ----------------------------------------
# สวัสดี, มานี!
# ปีนี้คุณอายุประมาณ 27 ปี
# ----------------------------------------
```

**สิ่งใหม่ที่ใช้ในเฉลยนี้:**

- `require "date"` — โหลด standard library `Date` เข้ามาใช้งาน (Ruby มาพร้อม standard
  library จำนวนมากที่ต้อง `require` ก่อนใช้)
- `.to_i` — แปลง String เป็น Integer (ถ้าแปลงไม่ได้จะได้ `0` ไม่ error)
- `"-" * 40` — String repetition operator ทำซ้ำ string 40 ครั้ง

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. แก้โปรแกรมให้รองรับกรณีผู้ใช้กรอกปีเกิดเป็นตัวอักษร (เช่น "abc") โดยแสดงข้อความแจ้งเตือน
   แทนที่จะคำนวณอายุผิดพลาด (ใบ้: ตรวจสอบด้วย regex `birth_year_input.match?(/\A\d+\z/)`)
2. เพิ่มการถามเดือนเกิดและวันเกิด แล้วคำนวณว่าปีนี้วันเกิดผ่านไปแล้วหรือยัง ถ้ายังไม่ถึง
   ให้อายุคำนวณ -1 จากการคำนวณแบบเดิม
3. เขียนโปรแกรมแยกต่างหากชื่อ `bmi.rb` ที่รับน้ำหนัก (กก.) และส่วนสูง (ม.) แล้วคำนวณค่า BMI
   พร้อมแสดงผลว่าอยู่ในเกณฑ์ใด (ผอม/ปกติ/น้ำหนักเกิน/อ้วน) — สูตร BMI = น้ำหนัก / (ส่วนสูง²)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ติดตั้ง Ruby ผ่าน `rbenv` หรือ `asdf` ได้ และเข้าใจว่าทำไมต้องใช้ version manager
- เข้าใจความสัมพันธ์ระหว่าง `gem`, `bundler`, `Gemfile`, `Gemfile.lock`
- ใช้ `irb`/`pry` ทดลองรันโค้ด Ruby แบบ interactive ได้
- เขียนและรันไฟล์ `.rb` ได้ พร้อมรับ input ทั้งจาก `ARGV` และ `gets`
- เข้าใจความแตกต่างของ `puts`/`print`/`p` และรู้ว่าเมื่อไหร่ควรใช้ตัวไหน
- ตั้งค่า VS Code และ RuboCop สำหรับเขียนโค้ด Ruby อย่างมีมาตรฐาน
- เขียนโปรแกรม Ruby ขนาดเล็กที่รับ input และประมวลผลได้ครบวงจร

**ต่อไป (Part 002):** เราจะเจาะลึกเรื่องตัวแปรและชนิดข้อมูลพื้นฐานของ Ruby ทั้งหมด
(Integer, Float, String, Symbol, nil, true/false) และการแปลงชนิดข้อมูลไปมา
