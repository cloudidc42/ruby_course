# Part 020: Rake, Rakefile และการเขียน CLI Tool ด้วย Ruby ล้วน — ปิด Phase 2 ด้วยโปรเจกต์ Todo CLI พร้อม Test

> **Step ครอบคลุมใน Part นี้:** Step 191–200
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 011–019 มาก่อน โดยเฉพาะ Part 011 Exception, Part 012
> File I/O/JSON, Part 013 Enumerable ขั้นสูง, Part 014 Data.define, และ Part 019 RSpec)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, rake 13.x, rspec 3.13 —
> `Data.define` ที่ใช้ในโปรเจกต์ปิดท้ายต้องการ **Ruby >= 3.2**)

Part นี้คือ **Part สุดท้ายของ Phase 2: Ruby Deep Dive** เราจะเรียนรู้เครื่องมือสำคัญตัวสุดท้าย
ก่อนกระโดดเข้า Rails นั่นคือ **Rake** — เครื่องมือรันงานอัตโนมัติที่ Rails ใช้เป็นฐานรากของ
คำสั่งอย่าง `rails db:migrate` แทบทุกคำสั่งที่ไม่ใช่การรัน server แล้วปิดท้ายด้วยโปรเจกต์รวบยอด
ขนาดใหญ่ที่สุดของ Phase นี้ — **Todo CLI** แอปพลิเคชัน command line ที่มีโครงสร้างไฟล์หลายไฟล์
เป็นระเบียบแบบมืออาชีพ พร้อม custom exception (Part 011), persist ข้อมูลเป็น JSON (Part 012),
ใช้ Enumerable กรอง/เรียงข้อมูล (Part 013), ใช้ `Data.define` เป็น value object (Part 014),
และมี **RSpec test ชุดเต็ม** (Part 019) ที่รันผ่าน **Rake task** — สรุปทุกอย่างที่เรียนมาตลอด
10 Part ของ Phase 2 ให้มาอยู่ในโปรเจกต์เดียวที่ใช้งานได้จริง

## สารบัญของ Part นี้

- Step 191: Rake คืออะไร และทำไมต้องมี — ทางเลือกที่เป็น Ruby ล้วนแทน Makefile
- Step 192: ติดตั้งและนิยาม task พื้นฐานตัวแรกด้วย `task :name do ... end`
- Step 193: `desc` — ใส่คำอธิบาย task และดูรายการ task ทั้งหมดด้วย `rake -T`
- Step 194: Task dependencies — `task build: [:clean, :compile]`
- Step 195: Namespace — จัดกลุ่ม task ด้วย `namespace :db do ... end` เรียกผ่าน `rake db:migrate`
- Step 196: ส่ง argument เข้า task ด้วย `task :greet, [:name] do |t, args| ... end`
- Step 197: วางแผนโปรเจกต์ปิด Phase 2 — Todo CLI: โจทย์และการออกแบบโครงสร้างไฟล์
- Step 198: เฉลย Layer ข้อมูล — `Task` (Data.define), Custom Exception, `TaskList` (Enumerable + JSON)
- Step 199: เฉลย CLI Loop และ Rakefile — เชื่อมทุกส่วนเข้าด้วยกันเป็นโปรแกรมที่รันได้จริง
- Step 200: เฉลย RSpec Test Suite รันผ่าน Rake + แบบฝึกหัดเพิ่มเติม + ปิด Phase 2

---

## Step 191: Rake คืออะไร และทำไมต้องมี — ทางเลือกที่เป็น Ruby ล้วนแทน Makefile

ในโลกของ C/C++ มีเครื่องมือชื่อ **`make`** ที่อ่านไฟล์ `Makefile` เพื่อกำหนดว่า "งาน" ไหน
ขึ้นกับ "งาน" ไหนบ้าง แล้วรันตามลำดับที่ถูกต้องให้อัตโนมัติ (เช่น compile ไฟล์ A ก่อน แล้วค่อย
link เป็นโปรแกรม) ปัญหาคือ syntax ของ `Makefile` เป็นภาษาเฉพาะตัวที่เคร่งครัดเรื่อง tab/space
มาก และไม่ใช่ภาษาโปรแกรมมิ่งทั่วไปที่เขียน logic ซับซ้อนได้สะดวก

**Rake** (**R**uby M**ake**) คือเครื่องมือแบบเดียวกัน แต่เขียนด้วย **Ruby ล้วนๆ** — ไฟล์ config
ของ Rake (ชื่อ `Rakefile`) ก็คือไฟล์ Ruby ธรรมดา จึงใช้ syntax, variable, method, class, loop,
condition ของ Ruby ได้ทุกอย่างโดยไม่ต้องเรียนภาษาใหม่เลย Rake เขียนโดย Jim Weirich และกลายเป็น
เครื่องมือมาตรฐานของวงการ Ruby อย่างรวดเร็ว จนถูกดึงเข้ามาเป็น **default gem** ที่ติดมากับ Ruby
ทุกเวอร์ชันตั้งแต่ Ruby 1.9 เป็นต้นมา (ไม่ต้อง `gem install` เพิ่มในเครื่องที่ติดตั้ง Ruby ปกติ)

**หน้าที่หลักของ Rake:**

1. **Automation** — รวมคำสั่งที่ต้องพิมพ์ซ้ำๆ (รัน test, ล้างไฟล์ temp, build asset) ไว้เป็น
   คำสั่งสั้นๆ คำสั่งเดียว
2. **Dependency management** — กำหนดว่างานไหนต้องรันก่อนงานไหน แล้วปล่อยให้ Rake จัดลำดับเอง
3. **Namespace** — จัดกลุ่มงานที่เกี่ยวข้องกันไว้ด้วยกัน (เช่น งานทั้งหมดเกี่ยวกับฐานข้อมูล)

```bash
# ตรวจสอบว่ามี rake ติดมากับ Ruby อยู่แล้วหรือยัง (ปกติจะมีอยู่แล้วเสมอ)
rake --version
# => rake, version 13.4.2 (เลขเวอร์ชันอาจต่างกันไปตามที่ติดตั้ง)

# ถ้าไม่มีจริงๆ (พบได้ยากมาก) ติดตั้งเพิ่มได้ด้วย
gem install rake
```

> **preview Rails:** ทุกครั้งที่พิมพ์ `rails db:migrate`, `rails db:seed`, `rails routes`,
> `rails assets:precompile` — คำสั่งเหล่านี้ **คือ Rake task ทั้งหมด** ที่ Rails framework
> นิยามไว้ให้ตั้งแต่สร้างโปรเจกต์ (`bin/rails` เป็นเพียง wrapper ที่เรียก Rake ต่ออีกที) การเข้าใจ
> Rake ให้ทะลุปรุโปร่งใน Part นี้จึงเป็นกุญแจสำคัญที่ทำให้เราไม่ "ท่องจำคำสั่ง" ตอนเข้า Rails
> แต่ **เข้าใจว่ามันทำงานอย่างไรจริงๆ**

---

## Step 192: ติดตั้งและนิยาม task พื้นฐานตัวแรกด้วย `task :name do ... end`

Rake มองหาไฟล์ชื่อ **`Rakefile`** (ตัว R ใหญ่ ไม่มีนามสกุล — เหมือนธรรมเนียมของ `Makefile`)
ที่ตำแหน่ง directory ปัจจุบันเสมอเมื่อถูกเรียกใช้งาน หน่วยพื้นฐานที่สุดในไฟล์นี้คือ **task**

```bash
mkdir -p ~/ruby-course-workspace/part-020/rake-basics
cd ~/ruby-course-workspace/part-020/rake-basics
```

```ruby
# frozen_string_literal: true

# Rakefile
task :hello do
  puts "สวัสดี จาก Rake!"
end

task :bye do
  puts "ลาก่อน!"
end
```

```bash
rake hello
# => สวัสดี จาก Rake!

rake bye
# => ลาก่อน!
```

**อธิบาย:**

- `task :name do ... end` นิยาม task ชื่อ `:name` — syntax หน้าตาเหมือน block ของ method ทั่วไป
  ใน Ruby ทุกประการ เพราะ `task` ก็คือ **method** ตัวหนึ่งที่ Rake ประกาศไว้ให้เรียกใช้ระดับ
  top-level ของไฟล์ (รับ Symbol เป็น argument แล้วรับ block เป็นเนื้อหางาน)
- รันด้วยคำสั่ง `rake <ชื่อ task>` จาก directory เดียวกับที่มีไฟล์ `Rakefile` อยู่ — ถ้ารันจาก
  directory อื่นที่ไม่มี `Rakefile` จะได้ error `No Rakefile found`
- ตอนนี้เรายังไม่ได้ใส่คำอธิบายให้ task เลย ลองรัน `rake -T` (list tasks) ดูจะพบว่า **ไม่มี
  อะไรแสดงออกมาเลย** ทั้งที่มี 2 task อยู่จริง — เดี๋ยวเราจะแก้ปัญหานี้ใน Step 193 ด้วย `desc`

---

## Step 193: `desc` — ใส่คำอธิบาย task และดูรายการ task ทั้งหมดด้วย `rake -T`

```ruby
# frozen_string_literal: true

desc "ทักทายผู้ใช้งาน"
task :hello do
  puts "สวัสดี จาก Rake!"
end

desc "กล่าวคำอำลา"
task :bye do
  puts "ลาก่อน!"
end
```

```bash
rake -T
# rake bye     # กล่าวคำอำลา
# rake hello   # ทักทายผู้ใช้งาน
```

**อธิบาย:**

- `desc "..."` เป็น method ที่เรียก**ก่อน**การนิยาม `task` เสมอ — มันจะ "ผูก" คำอธิบายเข้ากับ
  task ตัวถัดไปที่ถูกนิยามทันที (ทำงานคล้ายกับ annotation/decorator)
- `rake -T` (ย่อมาจาก `--tasks`) แสดงรายการ task **เฉพาะตัวที่มี `desc` กำกับไว้เท่านั้น** และ
  เรียงตามลำดับตัวอักษร — นี่คือเหตุผลที่ Step 192 ไม่เห็นอะไรเลย เพราะยังไม่มี `desc`
- ถ้าต้องการดู task **ทั้งหมด** รวมถึงตัวที่ไม่มี `desc` (เรียกว่า "undocumented task") ใช้
  `rake -AT` (`--all --tasks`) แทน — task ที่ไม่มี desc ยังคง**รันได้ปกติ** เพียงแค่ไม่ถูก
  แสดงใน `rake -T` เฉยๆ

> **แนวปฏิบัติที่ดี:** ใส่ `desc` ให้ทุก task ที่ตั้งใจให้คนอื่น (หรือตัวเองในอนาคต) เรียกใช้
> โดยตรง เพราะ `rake -T` คือเอกสารประกอบการใช้งานแบบสดๆ ที่ไม่มีวันตกยุค (ไม่เหมือน comment
> ที่อาจลืมอัปเดต) — โปรเจกต์ Rails ทุกโปรเจกต์ใช้หลักการนี้ ลองรัน `rails -T` ในโปรเจกต์ Rails
> ใดๆ จะเห็น task นับสิบตัวพร้อมคำอธิบายครบถ้วน

---

## Step 194: Task dependencies — `task build: [:clean, :compile]`

Task หนึ่งสามารถ "ขึ้นกับ" (depend on) task อื่นได้ — Rake จะรัน dependency ให้ครบตามลำดับ
**ก่อน** ที่จะรัน task หลัก

```ruby
# frozen_string_literal: true

desc "ล้างไฟล์ build เก่า"
task :clean do
  puts "กำลังล้างไฟล์เก่า..."
end

desc "compile ซอร์สโค้ด"
task :compile do
  puts "กำลัง compile..."
end

desc "build โปรเจกต์ทั้งหมด (clean แล้ว compile)"
task build: %i[clean compile] do
  puts "build เสร็จสมบูรณ์!"
end
```

```bash
rake build
# กำลังล้างไฟล์เก่า...
# กำลัง compile...
# build เสร็จสมบูรณ์!
```

**อธิบาย:**

- `task build: %i[clean compile]` เทียบเท่ากับการเขียนแบบเก่า `task :build => [:clean, :compile]`
  — ทั้งสองแบบคือการส่ง **Hash** ที่มี key เป็นชื่อ task และ value เป็น Array ของ dependency
  (`%i[clean compile]` คือ literal ของ Array of Symbol `[:clean, :compile]` ทบทวนจาก Part 003)
- Rake จะรัน `:clean` ก่อน แล้วรัน `:compile` ตามลำดับที่ระบุใน Array แล้วจึงรันเนื้อหาของ
  `:build` เอง เสมอ
- ถ้า `:build` ถูกเรียกซ้ำอีกครั้งในการรัน `rake` ครั้งเดียวกัน (เช่นมี task อื่นที่ก็ขึ้นกับ
  `:clean` เหมือนกัน) Rake จะ **จำไว้ว่า `:clean` ถูกรันไปแล้ว และจะไม่รันซ้ำ** — นี่คือกลไก
  ป้องกันงานซ้ำซ้อนที่ Rake จัดการให้อัตโนมัติ (คล้ายกับที่ `require` ป้องกันการโหลดไฟล์ซ้ำ)

---

## Step 195: Namespace — จัดกลุ่ม task ด้วย `namespace :db do ... end` เรียกผ่าน `rake db:migrate`

เมื่อโปรเจกต์มี task จำนวนมาก การจัดกลุ่ม task ที่เกี่ยวข้องกันไว้ด้วยกันภายใต้ **namespace**
ช่วยให้เป็นระเบียบและป้องกันชื่อชนกัน (แนวคิดเดียวกับ Module ที่ใช้เป็น namespace ใน Part 010
Step 95)

```ruby
# frozen_string_literal: true

namespace :db do
  desc "รัน migration ทั้งหมด"
  task :migrate do
    puts "กำลังรัน migration..."
  end

  desc "ล้างข้อมูลทั้งหมดในฐานข้อมูล"
  task :reset do
    puts "กำลังล้างฐานข้อมูล..."
  end
end
```

```bash
rake -T
# rake db:migrate   # รัน migration ทั้งหมด
# rake db:reset     # ล้างข้อมูลทั้งหมดในฐานข้อมูล

rake db:migrate
# => กำลังรัน migration...
```

**อธิบาย:**

- `namespace :db do ... end` ครอบ task ทั้งหมดภายในด้วย prefix `db:` — task ชื่อ `:migrate`
  ที่อยู่ข้างในจะถูกเรียกจริงผ่านชื่อเต็ม **`db:migrate`** ไม่ใช่ `migrate` เฉยๆ
- เครื่องหมาย `:` คือตัวคั่นระหว่าง namespace กับชื่อ task — ซ้อนกันหลายชั้นได้ เช่น
  `namespace :db do namespace :backup do task :run ... end end end` จะเรียกผ่าน
  `db:backup:run`
- ถ้าต้องการเรียก task หนึ่งจากภายใน task อื่นในไฟล์เดียวกัน (ข้าม namespace) ใช้
  `Rake::Task["db:migrate"].invoke`

```ruby
# เรียก task อื่นจากภายใน task โดยใช้ Rake::Task[...].invoke
namespace :db do
  task :migrate do
    puts "กำลังรัน migration..."
  end
end

desc "เตรียมระบบทั้งหมดสำหรับ deploy: migrate แล้วเคลียร์ cache"
task :deploy_prep do
  Rake::Task["db:migrate"].invoke
  puts "เคลียร์ cache..."
end
```

> **preview Rails:** namespace `db:` ที่เพิ่งเรียนนี้คือ namespace เดียวกันเป๊ะๆ กับที่ Rails
> ใช้จริง — `db:migrate`, `db:rollback`, `db:seed`, `db:reset` ทั้งหมดเป็น task ที่ Rails/
> ActiveRecord นิยามไว้ในโครงสร้าง namespace แบบนี้ตั้งแต่แรก

---

## Step 196: ส่ง argument เข้า task ด้วย `task :greet, [:name] do |t, args| ... end`

Task รับ argument จาก command line ได้ ด้วย syntax พิเศษที่ระบุชื่อ parameter ไว้ในวงเล็บ
เหลี่ยมต่อจากชื่อ task

```ruby
# frozen_string_literal: true

desc "ทักทายคนที่ระบุชื่อมา"
task :greet, [:name] do |_t, args|
  name = args[:name] || "เพื่อน"
  puts "สวัสดี, #{name}!"
end
```

```bash
rake "greet[Somchai]"
# => สวัสดี, Somchai!

rake greet
# => สวัสดี, เพื่อน!
```

**อธิบาย:**

- `task :greet, [:name] do |t, args| ... end` — ส่วน `[:name]` ประกาศว่า task นี้รับ argument
  ชื่อ `:name` ได้ 1 ตัว ส่วน block รับ 2 parameter เสมอ: **`t`** คือตัว `Rake::Task` object
  ของ task นี้เอง (ปกติไม่ค่อยได้ใช้ จึงตั้งชื่อ `_t` นำหน้าด้วย `_` บอกว่าตั้งใจไม่ใช้ ทบทวน
  แนวคิดนี้จาก Part 007) และ **`args`** คือ `Rake::TaskArguments` — object คล้าย Hash ที่ดึงค่า
  ออกมาได้ด้วย `args[:name]`
- **สำคัญมากเรื่อง shell quoting:** เครื่องหมาย `[` และ `]` เป็นอักขระพิเศษของ shell บางตัว
  (โดยเฉพาะ `zsh` ที่ใช้เป็น default shell ของ macOS) ถ้าพิมพ์ `rake greet[Somchai]` ตรงๆ
  โดยไม่มีเครื่องหมายคำพูดครอบ อาจได้ error `zsh: no matches found` — จึงต้องใส่เครื่องหมาย
  คำพูดครอบทั้งคำสั่งเสมอ: `rake "greet[Somchai]"`
- `args[:name] || "เพื่อน"` ใช้ `||` กำหนดค่า default เมื่อไม่ส่ง argument มา (ทบทวนรูปแบบนี้
  จาก Part 001 Step 7 ตอนเรียน `ARGV[0] || "World"`)

### รับ argument หลายตัว พร้อมค่า default ผ่าน `with_defaults`

```ruby
desc "แนะนำตัวพร้อมอายุ (มีค่า default ให้ age)"
task :introduce, [:name, :age] do |_t, args|
  args.with_defaults(age: "ไม่ทราบ")
  puts "ชื่อ #{args[:name]}, อายุ #{args[:age]}"
end
```

```bash
rake "introduce[Somchai,30]"
# => ชื่อ Somchai, อายุ 30

rake "introduce[Somchai]"
# => ชื่อ Somchai, อายุ ไม่ทราบ
```

`args.with_defaults(age: "ไม่ทราบ")` กำหนดค่า default ให้ทุก key ที่ยังไม่มีค่า (สะดวกกว่าการ
เขียน `||` ทีละตัวเมื่อมีหลาย argument) — argument ตัวที่สองในคำสั่ง `rake "introduce[Somchai,30]"`
คั่นด้วยเครื่องหมายจุลภาค (`,`) ตามลำดับที่ประกาศไว้ใน `[:name, :age]`

### สรุปคำสั่ง Rake ที่ใช้บ่อยที่สุด

| คำสั่ง/Syntax | ความหมาย |
|---|---|
| `task :name do ... end` | นิยาม task พื้นฐาน |
| `desc "..."` | ใส่คำอธิบาย task ตัวถัดไป (ให้ปรากฏใน `rake -T`) |
| `task build: [:a, :b] do ... end` | นิยาม task ที่ขึ้นกับ task อื่นก่อนเสมอ |
| `namespace :name do ... end` | จัดกลุ่ม task ภายใต้ prefix `name:` |
| `task :name, [:arg] do \|t, args\| ... end` | นิยาม task ที่รับ argument จาก command line |
| `rake -T` / `rake --tasks` | แสดงรายการ task ทั้งหมดที่มี `desc` |
| `rake -AT` | แสดงรายการ task ทั้งหมด รวมตัวที่ไม่มี `desc` |
| `Rake::Task["name"].invoke` | เรียก task อื่นจากภายในโค้ด Ruby |
| `task default: :spec` | กำหนดว่าพิมพ์ `rake` เฉยๆ (ไม่ระบุชื่อ) ให้รัน task ไหน |

> **preview Rails — สิ่งที่จะเจอเต็มรูปแบบใน Phase 3–4:** เมื่อสร้างโปรเจกต์ Rails ด้วย
> `rails new` แล้วรัน `rails db:migrate -T` (หรือ `bin/rails -T`) จะเห็น task นับร้อยตัวที่
> Rails สร้างให้อัตโนมัติ เช่น `db:migrate`, `db:seed`, `db:rollback`, `routes`,
> `assets:precompile`, `stats`, `tmp:clear` ฯลฯ — ทั้งหมดนี้คือ `task`/`namespace`/`desc`
> ที่เขียนด้วยหลักการเป๊ะๆ เหมือนที่เพิ่งเรียนใน 6 Step ที่ผ่านมา เพียงแค่มันถูกนิยามไว้ใน
> source code ของ Rails/ActiveRecord ให้เราแล้วเท่านั้นเอง เราจะเจาะลึกเรื่องนี้อีกครั้งตอน
> เรียน Migration ใน Phase 3

---

## Step 197: วางแผนโปรเจกต์ปิด Phase 2 — Todo CLI: โจทย์และการออกแบบโครงสร้างไฟล์

### โจทย์

สร้างแอปพลิเคชัน **Todo CLI** เป็นโปรแกรม command line จัดการรายการสิ่งที่ต้องทำ โดยต้องมี
คุณสมบัติครบถ้วนดังนี้:

1. **เมนูหลัก** ให้ผู้ใช้เลือก: เพิ่มงาน, แสดงรายการทั้งหมด, ทำเครื่องหมายว่าเสร็จแล้ว, ลบงาน,
   บันทึกลงไฟล์, แสดงสถิติ, ออกจากโปรแกรม
2. **Persist ข้อมูล** เป็นไฟล์ JSON เพื่อให้ข้อมูลไม่หายเมื่อปิดโปรแกรม (ทบทวน Part 012)
3. **Value Object** สำหรับเก็บข้อมูลงานแต่ละชิ้นด้วย `Data.define` (ทบทวน Part 014)
4. **Custom Exception** อย่างน้อย 3 ชนิด สำหรับ error ที่คาดการณ์ได้ (ทบทวน Part 011)
5. **Enumerable methods** สำหรับกรอง/เรียงงาน เช่น ดูเฉพาะงานที่ค้าง, เรียงตามวันที่สร้าง
   (ทบทวน Part 013)
6. **RSpec test suite** ครอบคลุม business logic ทั้งหมด และรันผ่าน **Rake task** (เชื่อม
   Part 019 เข้ากับ Part 020 นี้)
7. **โครงสร้างไฟล์แบบมืออาชีพ** — แยกไฟล์ตามหน้าที่ ไม่ยัดทุกอย่างไว้ไฟล์เดียวเหมือน
   `library_cli.rb` ใน Part 010

### การออกแบบโครงสร้างไฟล์ (Design Decisions)

Part 010 (`library_cli.rb`) เขียนทุกอย่างไว้ในไฟล์เดียว ซึ่งเหมาะกับโปรเจกต์เล็ก แต่เมื่อ
โปรเจกต์โตขึ้น — มี custom exception, มี persistence logic, มี CLI, มี test — การยัดทุกอย่าง
ไว้ไฟล์เดียวจะเริ่มอ่านยากและทดสอบยาก โปรเจกต์นี้จึงแยกเป็นหลายไฟล์ตาม **หน้าที่ความรับผิดชอบ**
(Single Responsibility) ดังนี้:

```
todo_cli/
├── Gemfile                    # ระบุ gem ที่ต้องใช้ (rake, rspec)
├── Rakefile                   # รวม task อัตโนมัติ: รัน test, รันโปรแกรม, เคลียร์ข้อมูล
├── bin/
│   └── todo                   # executable entry point ของโปรแกรม
├── lib/
│   ├── todo_cli.rb             # ไฟล์กลางที่ require ทุกไฟล์ย่อยเข้าด้วยกัน
│   └── todo_cli/
│       ├── errors.rb           # custom exception class ทั้งหมด
│       ├── task.rb             # Value Object: Task (ข้อมูล ไม่มี logic ซับซ้อน)
│       ├── task_list.rb        # Business logic: จัดการ collection ของ Task + persistence
│       └── cli.rb              # ส่วนติดต่อผู้ใช้: รับ input, เรียก TaskList, แสดงผล
└── spec/
    ├── spec_helper.rb          # ตั้งค่า RSpec กลาง
    ├── task_spec.rb            # test ของ Task
    ├── task_list_spec.rb       # test ของ TaskList
    └── cli_spec.rb             # test ของ CLI (ใช้ StringIO จำลอง input/output)
```

**เหตุผลเบื้องหลังการออกแบบแต่ละจุด — สิ่งเหล่านี้คือสะพานตรงไปสู่โครงสร้างของ Rails ที่จะเรียน
ตั้งแต่ Phase 3:**

- **`lib/todo_cli/` แยกไฟล์ตามหน้าที่ (`task.rb`, `task_list.rb`, `cli.rb`)** — เหมือนที่ Rails
  แยก `app/models/`, `app/controllers/`, `app/views/` ตามหน้าที่ ไม่ใช่ตามความสะดวกของผู้เขียน
  `task.rb` (ข้อมูล) จะกลายเป็นแนวคิดเดียวกับ **Model** ใน Rails, `cli.rb` (รับ input/แสดงผล)
  จะกลายเป็นแนวคิดเดียวกับ **Controller + View** รวมกัน
- **`errors.rb` แยกออกมาต่างหาก** — เพื่อให้เห็นภาพรวมของ "สิ่งที่อาจผิดพลาดได้" ทั้งหมดใน
  โปรเจกต์ในที่เดียว แทนที่จะกระจายการนิยาม exception ปนไปกับ business logic
- **`Task` เป็น `Data.define` ไม่ใช่ `class` ธรรมดา** — เพราะ `Task` มีหน้าที่แค่ "เก็บข้อมูล"
  ของงานหนึ่งชิ้น ไม่มี state ที่ซับซ้อนหรือพฤติกรรมเยอะ และเราต้องการให้มันเป็น **immutable**
  (แก้ไขค่าเดิมไม่ได้ ต้องสร้างตัวใหม่เสมอ) เพื่อป้องกันบัคจากการแก้ไขข้อมูลโดยไม่ตั้งใจ — ตรงตาม
  เหตุผลที่อธิบายไว้ใน Part 014 Step 138 เป๊ะๆ
- **`TaskList` เป็น class แยกที่ `include Enumerable`** — เพื่อห่อหุ้ม (encapsulate) collection
  ของ `Task` ทั้งหมดไว้ในที่เดียว ไม่ให้ `cli.rb` ไปยุ่งกับ Array ตรงๆ (ทบทวนเหตุผลเดียวกับที่
  `Library` class ทำใน Part 010) และการ `include Enumerable` ทำให้ได้ `select`, `sort_by`,
  `partition`, `map` มาใช้ฟรีทันที (ทบทวน Part 013)
- **`CLI` รับ `input`/`output` ผ่าน dependency injection แทนการผูกกับ `$stdin`/`$stdout` ตรงๆ**
  — เพื่อให้ทดสอบได้โดยไม่ต้องพิมพ์โต้ตอบจริงๆ ตอนรัน test (จะเห็นชัดเจนใน Step 200 ที่ทดสอบ
  `CLI` ด้วย `StringIO`) — นี่คือหลักการ **Dependency Injection** เบื้องต้น ซึ่งเป็นรากฐานเดียว
  กับที่ Rails Controller test ไม่ต้องเปิด browser จริงเพื่อทดสอบ (จะเรียนเต็มรูปแบบใน Phase 6)
- **`bin/todo` แยกจาก `lib/`** — ไฟล์ใน `bin/` มีหน้าที่แค่ "เริ่มโปรแกรม" (entry point) ส่วน
  logic ทั้งหมดอยู่ใน `lib/` — โครงสร้างนี้คือแบบเดียวกับที่ Rails มี `bin/rails` เป็นแค่ตัวเรียก
  ใช้โค้ดจริงที่อยู่ในที่อื่น
- **`Rakefile` อยู่ที่ root ของโปรเจกต์** — และนี่คือจุดที่น่าสนใจที่สุด: **โปรเจกต์ Rails ทุก
  โปรเจกต์ก็มีไฟล์ `Rakefile` อยู่ที่ root เหมือนกันเป๊ะๆ** (ลองเปิดดูได้ในโปรเจกต์ Rails ใดๆ)
  สิ่งที่เรียนใน Step 191–196 จึงไม่ใช่แค่ทฤษฎีเตรียมไปเข้าใจ Rails — มันคือไฟล์เดียวกัน
  ตำแหน่งเดียวกัน กลไกเดียวกันกับที่จะเจอจริงตั้งแต่ Part 021 เป็นต้นไป

> **หมายเหตุเรื่อง autoloading:** ไฟล์ `lib/todo_cli.rb` ของเราต้อง `require_relative` ไฟล์
> ย่อยทุกไฟล์ด้วยมือตามลำดับ (errors ก่อน เพราะ task.rb อ้างถึง error class, แล้วค่อย task,
> task_list, cli ตามลำดับการพึ่งพา) เพราะ Ruby ธรรมดาไม่มีระบบโหลดไฟล์อัตโนมัติ — **Rails ใช้
> ระบบชื่อ Zeitwerk ที่โหลด class ทุกไฟล์ใน `app/` ให้อัตโนมัติโดยไม่ต้อง `require` เองเลยสัก
> บรรทัดเดียว** (แค่ตั้งชื่อไฟล์/class ให้ตรงกัน) จะได้เรียนเรื่องนี้ตั้งแต่ Phase 3 — ตอนนี้
> การ `require_relative` ด้วยมือทำให้เห็นภาพว่า "ใครโหลดอะไรก่อนหลัง" ชัดเจนกว่าตอนเริ่มต้น

---

## Step 198: เฉลย Layer ข้อมูล — `Task` (Data.define), Custom Exception, `TaskList` (Enumerable + JSON)

### `lib/todo_cli/errors.rb` — Custom Exception ทั้งหมดของระบบ

```ruby
# frozen_string_literal: true

module TodoCli
  # Error เป็น base class กลางของทุก exception ในระบบ Todo CLI
  # ทำให้ CLI (Step 199) rescue ได้ด้วย `rescue TodoCli::Error` ตัวเดียว ครอบคลุมทุกกรณี
  # ที่ "คาดการณ์ไว้แล้ว" โดยไม่ไปดัก StandardError กว้างเกินไป (ทบทวน Part 011 Step 103)
  class Error < StandardError; end

  # เกิดขึ้นเมื่อข้อมูลงานไม่ถูกต้อง เช่น ชื่องานว่างเปล่า
  class InvalidTaskError < Error; end

  # เกิดขึ้นเมื่อค้นหางานด้วย id ที่ไม่มีอยู่จริง
  class TaskNotFoundError < Error; end

  # เกิดขึ้นเมื่อบันทึก/โหลดไฟล์ล้มเหลว (เขียนไฟล์ไม่ได้, ไฟล์ JSON เสียหาย)
  class StorageError < Error; end
end
```

**อธิบาย:** โครงสร้าง `Error < StandardError` แล้วให้ error เฉพาะทางทุกตัวสืบทอดจาก `Error`
(ไม่ใช่ `StandardError` ตรงๆ) คือ pattern เดียวกับที่ Part 011 Step 106 แนะนำไว้ — ทำให้
สามารถ `rescue TodoCli::Error` ครั้งเดียวเพื่อดักทุก error ที่ระบบนี้ออกแบบไว้ หรือจะดักเฉพาะ
ชนิดย่อย เช่น `rescue TodoCli::TaskNotFoundError` เพื่อจัดการเฉพาะกรณีนั้นก็ได้เช่นกัน

### `lib/todo_cli/task.rb` — Value Object ด้วย `Data.define`

```ruby
# frozen_string_literal: true

module TodoCli
  # Task คือ Value Object แบบ immutable สำหรับเก็บข้อมูลงานหนึ่งชิ้น
  # ใช้ Data.define (Ruby 3.2+) แทนการเขียน class ธรรมดา เพราะ Task มีแค่ข้อมูล 4 ฟิลด์
  # ที่ไม่ควรถูกแก้ไขตรงๆ หลังสร้างแล้ว (ทบทวน Part 014 Step 138)
  Task = Data.define(:id, :title, :done, :created_at) do
    # นิยาม method เพิ่มเติมได้ภายใน block นี้ เหมือนที่ทำกับ Struct.new ใน Part 014 Step 134
    def done?
      done
    end

    # "แก้ไข" สถานะเป็นเสร็จแล้ว โดยสร้าง Task ตัวใหม่ผ่าน `with`
    # (ของเดิมไม่เปลี่ยนแปลง เพราะทุก instance ของ Data ถูก freeze ให้อัตโนมัติ)
    def mark_done
      with(done: true)
    end

    # แปลงเป็น Hash ที่พร้อม serialize เป็น JSON — ต้องแปลง Time เป็น String ก่อนเสมอ
    # เพราะ JSON ไม่มีชนิดข้อมูล Date/Time ในตัว (ทบทวน Part 012 Step 118)
    def to_h_for_json
      {
        id: id,
        title: title,
        done: done,
        created_at: created_at.iso8601
      }
    end

    # class method ของ Data subclass เอง นิยามผ่าน `def self.xxx` ภายใน block ได้ตามปกติ
    def self.from_h(hash)
      new(
        id: hash.fetch(:id),
        title: hash.fetch(:title),
        done: hash.fetch(:done),
        created_at: Time.parse(hash.fetch(:created_at))
      )
    rescue KeyError => e
      raise InvalidTaskError, "ข้อมูลงานไม่ครบ: #{e.message}"
    end
  end
end
```

**อธิบาย:**

- `Data.define(:id, :title, :done, :created_at)` สร้าง class ใหม่ที่มี 4 ฟิลด์ พร้อม
  constructor ที่รับได้ทั้งแบบ positional และ keyword, `attr_reader` ให้ทุกฟิลด์, และ `==`
  ที่เทียบตามค่าภายในให้อัตโนมัติทั้งหมด — โดยไม่ต้องเขียนเองสักบรรทัด (ทบทวน Part 014 Step 138)
- `with(done: true)` คือวิธีมาตรฐานในการ "เปลี่ยนค่า" ของ `Data` object — เพราะแก้ไขตรงๆ ไม่ได้
  (`task.done = true` จะได้ `NoMethodError` ทันที) `with` จะคืน **instance ใหม่** ที่มีค่าฟิลด์
  ที่ระบุเปลี่ยนไป ส่วนฟิลด์อื่นคงค่าเดิมทั้งหมด
- `hash.fetch(:id)` ใช้ `fetch` แทน `hash[:id]` เพื่อให้ raise `KeyError` ทันทีถ้า key ไม่มีอยู่
  จริง (แทนที่จะได้ `nil` เงียบๆ แล้วพังทีหลังตอนเอาไปใช้) แล้ว `rescue KeyError` ครอบไว้แปลงเป็น
  `InvalidTaskError` ของเราเอง — เป็นตัวอย่างการ "แปลง error ทั่วไปให้เป็น error เฉพาะทางของ
  ระบบ" ตามที่แนะนำใน Part 011
- `InvalidTaskError` ใน `def self.from_h` เรียกใช้ได้โดยไม่ต้องเขียน `TodoCli::` นำหน้า เพราะ
  method นี้ถูกนิยาม (เขียน) อยู่ภายใน `module TodoCli` ตาม **lexical scope** ทาง Ruby จะค้นหา
  constant จากขอบเขตที่โค้ด**ถูกเขียน**ไว้ก่อนเสมอ ไม่ใช่จากตำแหน่งที่ method ถูกเรียกใช้งาน

### `lib/todo_cli/task_list.rb` — Business Logic + Persistence

```ruby
# frozen_string_literal: true

module TodoCli
  # TaskList ห่อหุ้ม collection ของ Task ทั้งหมดไว้ในที่เดียว (เทียบได้กับ Model/Repository
  # ใน Rails ที่จะเรียนใน Phase 3 — CLI ไม่ควรไปยุ่งกับ Array ของ Task ตรงๆ เลย)
  class TaskList
    include Enumerable

    def initialize(tasks = [])
      @tasks = tasks
      @next_id = compute_next_id(tasks)
    end

    # ต้องนิยาม each เพียง method เดียว แล้ว Enumerable จะปลดล็อก select, sort_by, partition,
    # count, map, find, any? ฯลฯ ให้ทั้งหมด (ทบทวน Part 010 Step 99 และ Part 013)
    def each
      return enum_for(:each) unless block_given?

      @tasks.each { |task| yield task }
    end

    def add(title)
      cleaned = title.to_s.strip
      raise InvalidTaskError, "ชื่องานต้องไม่ว่างเปล่า" if cleaned.empty?

      task = Task.new(id: @next_id, title: cleaned, done: false, created_at: Time.now)
      @tasks << task
      @next_id += 1
      task
    end

    def find_by_id(id)
      find { |task| task.id == id } || raise(TaskNotFoundError, "ไม่พบงาน ID #{id}")
    end

    def complete(id)
      task = find_by_id(id)
      replace_task(task, task.mark_done)
    end

    def delete(id)
      task = find_by_id(id)
      @tasks.delete(task)
      task
    end

    # --- ตัวอย่างการใช้ Enumerable method สำหรับกรอง/เรียงข้อมูล (ทบทวน Part 013) ---

    # reject(&:done?) คือ Enumerable method ที่คืนเฉพาะ element ที่ block คืนค่า falsy
    # (ตรงข้ามกับ select) — ใช้แทน select { |t| !t.done? } อ่านง่ายกว่า
    def pending
      reject(&:done?)
    end

    def completed_tasks
      select(&:done?)
    end

    def sorted_by_created
      sort_by(&:created_at)
    end

    # partition แบ่ง collection เป็น 2 กลุ่มตาม block เดียว: [กลุ่มที่ true, กลุ่มที่ false]
    def stats
      done_tasks, pending_tasks = partition(&:done?)
      { total: count, done: done_tasks.size, pending: pending_tasks.size }
    end

    # --- Persistence: บันทึก/โหลดเป็นไฟล์ JSON (ทบทวน Part 012) ---

    def save_to(path)
      File.write(path, JSON.pretty_generate(map(&:to_h_for_json)))
    rescue SystemCallError => e
      raise StorageError, "บันทึกไฟล์ไม่สำเร็จ: #{e.message}"
    end

    def self.load_from(path)
      return new unless File.exist?(path)

      raw_data = JSON.parse(File.read(path), symbolize_names: true)
      new(raw_data.map { |hash| Task.from_h(hash) })
    rescue JSON::ParserError => e
      raise StorageError, "ไฟล์ข้อมูล #{path} เสียหาย: #{e.message}"
    end

    private

    def replace_task(old_task, new_task)
      index = @tasks.index(old_task)
      @tasks[index] = new_task
      new_task
    end

    def compute_next_id(tasks)
      return 1 if tasks.empty?

      tasks.map(&:id).max + 1
    end
  end
end
```

**อธิบาย:**

- `find_by_id` ใช้ `find { ... } || raise(...)` — `Enumerable#find` คืน `nil` ถ้าหาไม่เจอ (ค่า
  falsy) จึงทำให้ `||` ไปประเมินฝั่งขวาคือ `raise(...)` ทันที ถ้าเจอ (ค่า truthy) จะ short-circuit
  ข้าม `raise` ไปเลย — เป็นสำนวน Ruby ที่กระชับกว่าการเขียน `if`/`unless` แยกบรรทัด
- `SystemCallError` ที่ `rescue` ใน `save_to` เป็น superclass ของ error ที่เกี่ยวกับระบบไฟล์
  ทั้งหมด เช่น `Errno::ENOENT` (ไม่พบไฟล์/โฟลเดอร์), `Errno::EACCES` (ไม่มีสิทธิ์เขียน) —
  การ rescue ที่ระดับนี้ครอบคลุมปัญหาระบบไฟล์ทุกแบบโดยไม่ต้องแจกแจงทีละชนิด
- `JSON.parse(..., symbolize_names: true)` แปลง key ทั้งหมดของ Hash ที่ได้จาก JSON ให้เป็น
  Symbol แทน String (`"title"` → `:title`) ทำให้เขียน `hash.fetch(:title)` ใน `Task.from_h`
  ได้ตรงกัน (ทบทวน Part 012 Step 118)
- `compute_next_id` คำนวณ id ตัวถัดไปจาก id สูงสุดที่มีอยู่ + 1 แทนการนับจากจำนวนงาน — สำคัญ
  มากเพราะถ้าลบงานกลางลิสต์ไปแล้ว การนับจากจำนวนงานที่เหลือจะทำให้ id ซ้ำกับงานเก่าที่ยังอยู่ได้

---

## Step 199: เฉลย CLI Loop และ Rakefile — เชื่อมทุกส่วนเข้าด้วยกันเป็นโปรแกรมที่รันได้จริง

### `lib/todo_cli/cli.rb` — ส่วนติดต่อผู้ใช้

```ruby
# frozen_string_literal: true

module TodoCli
  # CLI แยก concern "รับ input / แสดงผล" ออกจาก business logic ทั้งหมด (Task, TaskList)
  # โดยรับ input/output ผ่าน dependency injection แทนการผูกกับ $stdin/$stdout ตรงๆ
  # เพื่อให้ทดสอบได้ง่ายด้วย StringIO (ดู spec/cli_spec.rb ใน Step 200)
  class CLI
    DEFAULT_STORAGE_PATH = File.join(Dir.pwd, "todos.json")

    def initialize(storage_path: DEFAULT_STORAGE_PATH, input: $stdin, output: $stdout)
      @storage_path = storage_path
      @input = input
      @output = output
      @list = TaskList.load_from(@storage_path)
    end

    def run
      loop do
        print_menu
        choice = read_line

        if choice == "0" || choice.empty?
          handle_exit
          break
        end

        dispatch(choice)
      end
    end

    private

    # gets คืน nil ถ้า input stream ถูกปิด (เช่น กด Ctrl+D) — แปลงเป็น "" แทนเสมอ
    # เพื่อไม่ให้โปรแกรม error ตอนเรียก .chomp/.to_i ต่อจาก nil (ทบทวนแนวคิดจาก Part 010 Step 100)
    def read_line
      @input.gets&.chomp || ""
    end

    def dispatch(choice)
      case choice
      when "1" then handle_add
      when "2" then handle_list
      when "3" then handle_complete
      when "4" then handle_delete
      when "5" then handle_save
      when "6" then handle_stats
      else @output.puts "กรุณาเลือกเมนูให้ถูกต้อง (0-6)"
      end
    end

    def print_menu
      @output.puts "\n#{'=' * 40}"
      @output.puts "  Todo CLI"
      @output.puts "=" * 40
      @output.puts "1) เพิ่มงานใหม่"
      @output.puts "2) แสดงรายการทั้งหมด"
      @output.puts "3) ทำเครื่องหมายว่าเสร็จแล้ว"
      @output.puts "4) ลบงาน"
      @output.puts "5) บันทึกลงไฟล์"
      @output.puts "6) แสดงสถิติ"
      @output.puts "0) บันทึกและออกจากโปรแกรม"
      @output.print "เลือกเมนู: "
    end

    def handle_add
      @output.print "ชื่องาน: "
      task = @list.add(read_line)
      @output.puts "เพิ่มงาน \"#{task.title}\" แล้ว (ID: #{task.id})"
    rescue Error => e
      @output.puts "เกิดข้อผิดพลาด: #{e.message}"
    end

    def handle_list
      if @list.count.zero?
        @output.puts "ยังไม่มีงานในระบบ"
        return
      end

      @list.sorted_by_created.each_with_index do |task, index|
        mark = task.done? ? "[x]" : "[ ]"
        @output.puts "#{index + 1}. #{mark} ##{task.id} #{task.title}"
      end
    end

    def handle_complete
      @output.print "ID ของงานที่ทำเสร็จแล้ว: "
      task = @list.complete(read_line.to_i)
      @output.puts "ทำเครื่องหมาย \"#{task.title}\" ว่าเสร็จแล้ว"
    rescue Error => e
      @output.puts "เกิดข้อผิดพลาด: #{e.message}"
    end

    def handle_delete
      @output.print "ID ของงานที่จะลบ: "
      task = @list.delete(read_line.to_i)
      @output.puts "ลบงาน \"#{task.title}\" แล้ว"
    rescue Error => e
      @output.puts "เกิดข้อผิดพลาด: #{e.message}"
    end

    def handle_save
      @list.save_to(@storage_path)
      @output.puts "บันทึกข้อมูลลงไฟล์ #{@storage_path} เรียบร้อยแล้ว"
    rescue Error => e
      @output.puts "เกิดข้อผิดพลาด: #{e.message}"
    end

    def handle_stats
      data = @list.stats
      @output.puts "งานทั้งหมด: #{data[:total]}"
      @output.puts "เสร็จแล้ว: #{data[:done]}"
      @output.puts "ค้างอยู่: #{data[:pending]}"
    end

    def handle_exit
      handle_save
      @output.puts "ขอบคุณที่ใช้งาน Todo CLI ลาก่อน!"
    end
  end
end
```

### `lib/todo_cli.rb` — ไฟล์กลางที่ประกอบทุกส่วนเข้าด้วยกัน

```ruby
# frozen_string_literal: true

require "json"
require "time"

require_relative "todo_cli/errors"
require_relative "todo_cli/task"
require_relative "todo_cli/task_list"
require_relative "todo_cli/cli"
```

**อธิบาย:** ลำดับการ `require_relative` สำคัญมาก — `errors.rb` ต้องโหลดก่อนเพราะ `task.rb`
อ้างถึง `InvalidTaskError`, ส่วน `task_list.rb` อ้างถึง `Task` และ error class ต่างๆ จึงต้องโหลด
หลังทั้งสองไฟล์แรก, และ `cli.rb` อ้างถึง `TaskList` จึงโหลดเป็นลำดับสุดท้าย — การ require ตาม
ลำดับการพึ่งพา (dependency order) แบบนี้ด้วยมือ คือสิ่งที่ Zeitwerk ของ Rails จะทำให้อัตโนมัติ

### `bin/todo` — Entry Point ของโปรแกรม

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

require_relative "../lib/todo_cli"

TodoCli::CLI.new.run
```

```bash
chmod +x bin/todo   # ให้สิทธิ์รันเป็นโปรแกรมได้โดยตรง (ทำครั้งเดียว)
./bin/todo
# หรือรันผ่าน ruby ตรงๆ โดยไม่ต้อง chmod ก็ได้เช่นกัน
ruby bin/todo
```

บรรทัดแรก `#!/usr/bin/env ruby` เรียกว่า **shebang** บอกระบบปฏิบัติการว่าไฟล์นี้ต้องรันด้วย
`ruby` interpreter ที่หาเจอใน `PATH` — ทำให้เรียก `./bin/todo` ตรงๆ ได้โดยไม่ต้องพิมพ์ `ruby`
นำหน้า (หลังจาก `chmod +x` แล้ว) นี่คือรูปแบบเดียวกับไฟล์ `bin/rails`, `bin/rake` ที่จะเจอใน
ทุกโปรเจกต์ Rails

### `Gemfile` — ระบุ dependency ของโปรเจกต์

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

gem "rake", "~> 13.2"
gem "rspec", "~> 3.13"
```

```bash
bundle install
```

### `Rakefile` — รวม task อัตโนมัติทั้งหมดของโปรเจกต์

```ruby
# frozen_string_literal: true

require "rspec/core/rake_task"
require_relative "lib/todo_cli"

# กำหนด task ชื่อ :spec โดยใช้ RSpec::Core::RakeTask ซึ่งมากับ gem rspec-core
# เพื่อให้รัน RSpec ผ่านคำสั่ง rake ได้ (แทนการพิมพ์ `rspec` ตรงๆ) — มี desc "Run RSpec code
# examples" ติดมาให้อัตโนมัติอยู่แล้ว จึงไม่ต้องเขียน desc ซ้ำเอง
RSpec::Core::RakeTask.new(:spec) do |t|
  t.pattern = "spec/**/*_spec.rb"
  t.rspec_opts = "--format documentation --color"
end

# พิมพ์ `rake` เฉยๆ (ไม่ระบุชื่อ task) ให้เท่ากับ `rake spec` โดยอัตโนมัติ
task default: :spec

desc "รันโปรแกรม Todo CLI แบบ interactive"
task :run do
  ruby "bin/todo"
end

desc "ลบไฟล์ข้อมูล todos.json ทิ้ง (เริ่มต้นใหม่หมด)"
task :clean do
  rm_f "todos.json"
  puts "ลบไฟล์ todos.json เรียบร้อยแล้ว (ถ้ามีอยู่)"
end

desc "ตรวจสอบทั้งหมดก่อน commit"
task check: [:spec] do
  puts "ผ่านการตรวจสอบทั้งหมดแล้ว พร้อม commit!"
end

namespace :todo do
  desc 'เพิ่มงานใหม่แบบรวดเร็วผ่าน command line เช่น rake "todo:add[ซื้อของ]"'
  task :add, [:title] do |_task, args|
    list = TodoCli::TaskList.load_from("todos.json")
    task_obj = list.add(args[:title])
    list.save_to("todos.json")
    puts "เพิ่มงาน \"#{task_obj.title}\" แล้ว (ID: #{task_obj.id})"
  end

  desc "แสดงรายการงานทั้งหมดแบบรวดเร็วผ่าน command line"
  task :list do
    list = TodoCli::TaskList.load_from("todos.json")
    if list.count.zero?
      puts "ยังไม่มีงานในระบบ"
    else
      list.sorted_by_created.each do |t|
        mark = t.done? ? "[x]" : "[ ]"
        puts "#{mark} ##{t.id} #{t.title}"
      end
    end
  end

  desc "แสดงสถิติงานทั้งหมดแบบรวดเร็วผ่าน command line"
  task :stats do
    list = TodoCli::TaskList.load_from("todos.json")
    stats = list.stats
    puts "ทั้งหมด: #{stats[:total]} | เสร็จแล้ว: #{stats[:done]} | ค้างอยู่: #{stats[:pending]}"
  end
end
```

```bash
bundle exec rake -T
# rake check            # ตรวจสอบทั้งหมดก่อน commit
# rake clean            # ลบไฟล์ข้อมูล todos.json ทิ้ง (เริ่มต้นใหม่หมด)
# rake run              # รันโปรแกรม Todo CLI แบบ interactive
# rake spec             # Run RSpec code examples
# rake todo:add[title]  # เพิ่มงานใหม่แบบรวดเร็วผ่าน command line เช่น rake "todo:add[ซื้อของ]"
# rake todo:list        # แสดงรายการงานทั้งหมดแบบรวดเร็วผ่าน command line
# rake todo:stats       # แสดงสถิติงานทั้งหมดแบบรวดเร็วผ่าน command line

bundle exec rake "todo:add[ซื้อของ]"
# => เพิ่มงาน "ซื้อของ" แล้ว (ID: 1)

bundle exec rake todo:list
# => [ ] #1 ซื้อของ
```

**อธิบาย:**

- ใช้ `require "rspec/core/rake_task"` แล้วสร้าง `RSpec::Core::RakeTask.new(:spec)` — วิธีนี้
  คือวิธีมาตรฐานที่โปรเจกต์ Ruby (รวมถึง Rails เอง) ใช้เชื่อม RSpec เข้ากับ Rake ทุกโปรเจกต์
  แทนที่จะเขียน `task :spec do sh "rspec" end` เอง เพราะ `RSpec::Core::RakeTask` จัดการเรื่อง
  exit code (ทำให้ CI/CD ตรวจจับได้ว่า test พังหรือไม่) และ option ต่างๆ ให้ครบถ้วนกว่า
- `ruby "bin/todo"` ใน task `:run` ไม่ใช่ `Kernel#system` — แต่เป็น method `ruby` ที่ Rake ผสม
  (mix in) เข้ามาให้ผ่าน `Rake::FileUtilsExt` ซึ่งจะเรียก Ruby interpreter ตัวเดียวกับที่กำลัง
  รัน Rake อยู่ตอนนี้พอดี (สำคัญเวลามี Ruby หลายเวอร์ชันในเครื่องผ่าน rbenv/asdf — การันตีว่าใช้
  เวอร์ชันที่ถูกต้องเสมอ)
- `rm_f`, `sh` ก็เป็น method จาก `FileUtils` ที่ Rake ผสมเข้ามาให้ใช้ได้ตรงๆ ในไฟล์ `Rakefile`
  เช่นกัน (`rm_f` คือ `rm -f` ที่ไม่ error แม้ไฟล์จะไม่มีอยู่จริง)
- `task check: [:spec]` สาธิตการใช้ dependency (Step 194) ในบริบทจริง — ก่อนจะ "ตรวจสอบพร้อม
  commit" ต้องรัน test ให้ผ่านก่อนเสมอ
- namespace `todo:` (Step 195) และ argument `[:title]` (Step 196) ถูกนำมาใช้จริงในโปรเจกต์
  นี้ — แสดงให้เห็นว่า Rake ไม่ได้มีไว้แค่รัน test แต่ยังใช้เป็น **CLI ทางเลือก** สำหรับจัดการ
  ข้อมูลแบบรวดเร็ว โดยไม่ต้องเปิดเมนู interactive เต็มรูปแบบ (สะดวกมากเวลาเขียน script
  อัตโนมัติหรือเรียกจาก CI/CD)
- **ต้องรันผ่าน `bundle exec rake`** ไม่ใช่ `rake` เฉยๆ — เพื่อให้ Ruby หา gem `rspec` เจอตาม
  เวอร์ชันที่ระบุใน `Gemfile.lock` พอดิบพอดี ตรงตามหลักการที่ Part 001 Step 4 อธิบายไว้ตั้งแต่ต้น
  หลักสูตร

### ทดสอบรันโปรแกรมแบบ interactive

```bash
ruby bin/todo
```

```
========================================
  Todo CLI
========================================
1) เพิ่มงานใหม่
2) แสดงรายการทั้งหมด
3) ทำเครื่องหมายว่าเสร็จแล้ว
4) ลบงาน
5) บันทึกลงไฟล์
6) แสดงสถิติ
0) บันทึกและออกจากโปรแกรม
เลือกเมนู: 1
ชื่องาน: ซื้อของเข้าบ้าน
เพิ่มงาน "ซื้อของเข้าบ้าน" แล้ว (ID: 1)

เลือกเมนู: 1
ชื่องาน: อ่านหนังสือ Ruby
เพิ่มงาน "อ่านหนังสือ Ruby" แล้ว (ID: 2)

เลือกเมนู: 2
1. [ ] #1 ซื้อของเข้าบ้าน
2. [ ] #2 อ่านหนังสือ Ruby

เลือกเมนู: 3
ID ของงานที่ทำเสร็จแล้ว: 1
ทำเครื่องหมาย "ซื้อของเข้าบ้าน" ว่าเสร็จแล้ว

เลือกเมนู: 6
งานทั้งหมด: 2
เสร็จแล้ว: 1
ค้างอยู่: 1

เลือกเมนู: 0
บันทึกข้อมูลลงไฟล์ /home/user/todo_cli/todos.json เรียบร้อยแล้ว
ขอบคุณที่ใช้งาน Todo CLI ลาก่อน!
```

โค้ดทั้งหมดข้างต้นนี้ทดสอบรันจริงแล้วบน Ruby 3.3.6 ทั้งแบบ interactive เต็มรูปแบบ (เพิ่มงาน,
แสดงรายการ, ทำเครื่องหมายเสร็จ, ดูสถิติ, ลบงาน, บันทึกไฟล์) และแบบผ่าน `rake todo:add`/
`rake todo:list` — ไฟล์ `todos.json` ที่บันทึกออกมามีรูปแบบดังนี้:

```json
[
  {
    "id": 1,
    "title": "ซื้อของเข้าบ้าน",
    "done": true,
    "created_at": "2026-09-26T02:36:30+00:00"
  },
  {
    "id": 2,
    "title": "อ่านหนังสือ Ruby",
    "done": false,
    "created_at": "2026-09-26T02:36:45+00:00"
  }
]
```

---

## Step 200: เฉลย RSpec Test Suite รันผ่าน Rake + แบบฝึกหัดเพิ่มเติม + ปิด Phase 2

ปิดท้ายโปรเจกต์ด้วยชุดทดสอบ RSpec ที่ครอบคลุม `Task`, `TaskList`, และ `CLI` ทั้งหมด — ทบทวน
รูปแบบ `describe`/`context`/`it`/`let`/`before` จาก Part 019

### `spec/spec_helper.rb`

```ruby
# frozen_string_literal: true

require_relative "../lib/todo_cli"

RSpec.configure do |config|
  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.disable_monkey_patching!
  config.order = :random
  Kernel.srand config.seed
end
```

### `spec/task_spec.rb`

```ruby
# frozen_string_literal: true

require "spec_helper"

RSpec.describe TodoCli::Task do
  let(:task) do
    described_class.new(id: 1, title: "ซื้อของ", done: false,
                         created_at: Time.new(2026, 1, 1, 0, 0, 0, "+00:00"))
  end

  describe "#done?" do
    it "คืนค่า false เมื่อ done เป็น false" do
      expect(task.done?).to be false
    end

    it "คืนค่า true เมื่อ done เป็น true" do
      expect(task.with(done: true).done?).to be true
    end
  end

  describe "#mark_done" do
    it "คืน Task ตัวใหม่ที่ done เป็น true โดยไม่แก้ตัวเดิม" do
      done_task = task.mark_done

      expect(done_task.done?).to be true
      expect(task.done?).to be false # ตัวเดิมไม่เปลี่ยน เพราะ Data เป็น immutable
    end
  end

  describe "#to_h_for_json" do
    it "แปลงเป็น Hash พร้อม created_at แบบ ISO8601 string" do
      hash = task.to_h_for_json

      expect(hash).to include(id: 1, title: "ซื้อของ", done: false)
      expect(hash[:created_at]).to eq("2026-01-01T00:00:00+00:00")
    end
  end

  describe ".from_h" do
    it "สร้าง Task กลับจาก Hash ที่มาจาก JSON" do
      hash = { id: 5, title: "อ่านหนังสือ", done: true, created_at: "2026-01-01T00:00:00+00:00" }
      rebuilt = described_class.from_h(hash)

      expect(rebuilt.id).to eq(5)
      expect(rebuilt.done?).to be true
      expect(rebuilt.created_at).to be_a(Time)
    end

    it "raise InvalidTaskError เมื่อข้อมูลไม่ครบ" do
      expect { described_class.from_h(id: 1, title: "x") }
        .to raise_error(TodoCli::InvalidTaskError)
    end
  end
end
```

### `spec/task_list_spec.rb`

```ruby
# frozen_string_literal: true

require "spec_helper"
require "tmpdir"

RSpec.describe TodoCli::TaskList do
  subject(:list) { described_class.new }

  describe "#add" do
    it "เพิ่มงานใหม่และคืน Task ที่สร้างสำเร็จ" do
      task = list.add("ซื้อของ")

      expect(task.title).to eq("ซื้อของ")
      expect(task.done?).to be false
      expect(list.count).to eq(1)
    end

    it "ตัด whitespace หัวท้ายของชื่องานให้อัตโนมัติ" do
      task = list.add("   อ่านหนังสือ   ")
      expect(task.title).to eq("อ่านหนังสือ")
    end

    it "raise InvalidTaskError เมื่อชื่องานว่างเปล่า" do
      expect { list.add("   ") }.to raise_error(TodoCli::InvalidTaskError, /ต้องไม่ว่างเปล่า/)
    end

    it "กำหนด id เรียงต่อกันโดยอัตโนมัติ" do
      first = list.add("งานที่ 1")
      second = list.add("งานที่ 2")

      expect(first.id).to eq(1)
      expect(second.id).to eq(2)
    end
  end

  describe "#find_by_id" do
    it "คืนงานที่ id ตรงกัน" do
      task = list.add("ซื้อของ")
      expect(list.find_by_id(task.id)).to eq(task)
    end

    it "raise TaskNotFoundError เมื่อไม่พบ id" do
      expect { list.find_by_id(999) }.to raise_error(TodoCli::TaskNotFoundError, /999/)
    end
  end

  describe "#complete" do
    it "เปลี่ยนสถานะงานเป็นเสร็จแล้ว" do
      task = list.add("ซื้อของ")
      completed = list.complete(task.id)

      expect(completed.done?).to be true
      expect(list.find_by_id(task.id).done?).to be true
    end
  end

  describe "#delete" do
    it "ลบงานออกจากรายการ" do
      task = list.add("ซื้อของ")
      list.delete(task.id)

      expect(list.count).to eq(0)
      expect { list.find_by_id(task.id) }.to raise_error(TodoCli::TaskNotFoundError)
    end
  end

  describe "#pending และ #completed_tasks" do
    before do
      list.add("งาน A")
      b = list.add("งาน B")
      list.complete(b.id)
    end

    it "#pending คืนเฉพาะงานที่ยังไม่เสร็จ" do
      expect(list.pending.map(&:title)).to eq(["งาน A"])
    end

    it "#completed_tasks คืนเฉพาะงานที่เสร็จแล้ว" do
      expect(list.completed_tasks.map(&:title)).to eq(["งาน B"])
    end
  end

  describe "#stats" do
    it "สรุปจำนวนงานทั้งหมด/เสร็จแล้ว/ค้างอยู่" do
      list.add("งาน A")
      done_task = list.add("งาน B")
      list.complete(done_task.id)

      expect(list.stats).to eq(total: 2, done: 1, pending: 1)
    end
  end

  describe "Enumerable" do
    it "ใช้ method จาก Enumerable ได้ทันทีเพราะนิยาม each ไว้" do
      list.add("แอปเปิ้ล")
      list.add("กล้วย")

      expect(list.map(&:title)).to eq(["แอปเปิ้ล", "กล้วย"])
      expect(list.count).to eq(2)
      expect(list.any? { |task| task.title == "กล้วย" }).to be true
    end
  end

  describe "#save_to และ .load_from" do
    it "บันทึกและโหลดข้อมูลกลับมาได้ครบถ้วน (round-trip ผ่าน JSON)" do
      Dir.mktmpdir do |dir|
        path = File.join(dir, "todos.json")

        list.add("ซื้อของ")
        done_task = list.add("อ่านหนังสือ")
        list.complete(done_task.id)
        list.save_to(path)

        loaded = described_class.load_from(path)

        expect(loaded.count).to eq(2)
        expect(loaded.map(&:title)).to match_array(["ซื้อของ", "อ่านหนังสือ"])
        expect(loaded.find_by_id(done_task.id).done?).to be true
      end
    end

    it "คืน TaskList ว่างเปล่าถ้าไฟล์ยังไม่มีอยู่จริง" do
      loaded = described_class.load_from("/tmp/ไม่มีไฟล์นี้แน่นอน.json")
      expect(loaded.count).to eq(0)
    end

    it "raise StorageError เมื่อไฟล์ข้อมูลเสียหาย (ไม่ใช่ JSON ที่ถูกต้อง)" do
      Dir.mktmpdir do |dir|
        path = File.join(dir, "broken.json")
        File.write(path, "{ invalid json ]")

        expect { described_class.load_from(path) }.to raise_error(TodoCli::StorageError)
      end
    end
  end
end
```

### `spec/cli_spec.rb` — ทดสอบ CLI ด้วย `StringIO` แทน `$stdin`/`$stdout` จริง

```ruby
# frozen_string_literal: true

require "spec_helper"
require "stringio"
require "tmpdir"
require "json"

RSpec.describe TodoCli::CLI do
  let(:storage_path) { File.join(Dir.mktmpdir, "todos_test.json") }

  def run_cli(input_text)
    input = StringIO.new(input_text)
    output = StringIO.new
    cli = described_class.new(storage_path: storage_path, input: input, output: output)
    cli.run
    output.string
  end

  it "เพิ่มงานใหม่และแสดงในรายการได้" do
    transcript = run_cli("1\nซื้อของ\n2\n0\n")

    expect(transcript).to include("เพิ่มงาน \"ซื้อของ\" แล้ว")
    expect(transcript).to include("[ ] #1 ซื้อของ")
  end

  it "ทำเครื่องหมายงานว่าเสร็จแล้วได้" do
    transcript = run_cli("1\nซื้อของ\n3\n1\n2\n0\n")

    expect(transcript).to include("ทำเครื่องหมาย \"ซื้อของ\" ว่าเสร็จแล้ว")
    expect(transcript).to include("[x] #1 ซื้อของ")
  end

  it "แจ้งข้อผิดพลาดเมื่อเพิ่มงานชื่อว่างเปล่า" do
    transcript = run_cli("1\n\n0\n")

    expect(transcript).to include("เกิดข้อผิดพลาด: ชื่องานต้องไม่ว่างเปล่า")
  end

  it "บันทึกข้อมูลลงไฟล์จริงเมื่อออกจากโปรแกรม" do
    run_cli("1\nซื้อของ\n0\n")

    expect(File.exist?(storage_path)).to be true
    saved = JSON.parse(File.read(storage_path))
    expect(saved.first["title"]).to eq("ซื้อของ")
  end
end
```

**อธิบาย:**

- `StringIO.new("1\nซื้อของ\n2\n0\n")` สร้าง object ที่ตอบสนอง `#gets` เหมือน `$stdin` ทุก
  ประการ แต่อ่านจาก String ที่เตรียมไว้ล่วงหน้าแทนการรอผู้ใช้พิมพ์จริง — เพราะ `CLI#initialize`
  รับ `input:`/`output:` เป็น parameter (ทบทวน dependency injection จาก Step 197) การทดสอบจึง
  ทำได้โดยไม่ต้องพิมพ์โต้ตอบเลยแม้แต่ครั้งเดียว และรันได้เร็วมาก (ทั้งไฟล์ใช้เวลาต่ำกว่า
  0.02 วินาที)
- `output.string` ดึงข้อความทั้งหมดที่ถูกเขียนผ่าน `StringIO` ออกมาเป็น String เดียว แล้วใช้
  matcher `include` ตรวจสอบว่ามีข้อความที่คาดหวังปรากฏอยู่หรือไม่ — เป็นวิธีทดสอบ "transcript"
  ของโปรแกรม CLI ทั้งหมดโดยไม่ต้อง mock ทีละ method
- `Dir.mktmpdir` (จาก Part 012) สร้างโฟลเดอร์ชั่วคราวสำหรับเก็บไฟล์ทดสอบ ป้องกันไม่ให้ test
  ไปทับไฟล์ `todos.json` ของโปรแกรมจริงโดยไม่ตั้งใจ

### รันชุดทดสอบทั้งหมดผ่าน Rake

```bash
bundle exec rake spec
```

```
Randomized with seed 57990

TodoCli::Task
  #to_h_for_json
    แปลงเป็น Hash พร้อม created_at แบบ ISO8601 string
  #mark_done
    คืน Task ตัวใหม่ที่ done เป็น true โดยไม่แก้ตัวเดิม
  #done?
    คืนค่า false เมื่อ done เป็น false
    คืนค่า true เมื่อ done เป็น true
  .from_h
    สร้าง Task กลับจาก Hash ที่มาจาก JSON
    raise InvalidTaskError เมื่อข้อมูลไม่ครบ

TodoCli::CLI
  บันทึกข้อมูลลงไฟล์จริงเมื่อออกจากโปรแกรม
  แจ้งข้อผิดพลาดเมื่อเพิ่มงานชื่อว่างเปล่า
  ทำเครื่องหมายงานว่าเสร็จแล้วได้
  เพิ่มงานใหม่และแสดงในรายการได้

TodoCli::TaskList
  #pending และ #completed_tasks
    #completed_tasks คืนเฉพาะงานที่เสร็จแล้ว
    #pending คืนเฉพาะงานที่ยังไม่เสร็จ
  #complete
    เปลี่ยนสถานะงานเป็นเสร็จแล้ว
  Enumerable
    ใช้ method จาก Enumerable ได้ทันทีเพราะนิยาม each ไว้
  #save_to และ .load_from
    บันทึกและโหลดข้อมูลกลับมาได้ครบถ้วน (round-trip ผ่าน JSON)
    raise StorageError เมื่อไฟล์ข้อมูลเสียหาย (ไม่ใช่ JSON ที่ถูกต้อง)
    คืน TaskList ว่างเปล่าถ้าไฟล์ยังไม่มีอยู่จริง
  #add
    ตัด whitespace หัวท้ายของชื่องานให้อัตโนมัติ
    กำหนด id เรียงต่อกันโดยอัตโนมัติ
    raise InvalidTaskError เมื่อชื่องานว่างเปล่า
    เพิ่มงานใหม่และคืน Task ที่สร้างสำเร็จ
  #find_by_id
    คืนงานที่ id ตรงกัน
    raise TaskNotFoundError เมื่อไม่พบ id
  #stats
    สรุปจำนวนงานทั้งหมด/เสร็จแล้ว/ค้างอยู่
  #delete
    ลบงานออกจากรายการ

Finished in 0.00938 seconds (files took 0.08191 seconds to load)
25 examples, 0 failures

Randomized with seed 57990
```

ชุดทดสอบทั้งหมด **25 examples ผ่านหมดทุกตัว (0 failures)** — ทดสอบรันจริงแล้วทั้งผ่านคำสั่ง
`rspec` ตรงๆ และผ่าน `bundle exec rake spec` ให้ผลลัพธ์เดียวกัน ยืนยันว่า `RSpec::Core::RakeTask`
เชื่อมต่อกับชุดทดสอบได้ถูกต้องสมบูรณ์ และ `bundle exec rake check` (ที่ขึ้นกับ `:spec`) ก็รัน
ทดสอบให้ผ่านก่อนแล้วค่อยพิมพ์ "ผ่านการตรวจสอบทั้งหมดแล้ว พร้อม commit!" ตามที่ออกแบบไว้ใน
Step 199

### จุดที่ควรสังเกตในเฉลยทั้งโปรเจกต์

- **`Data.define` ทำให้ `Task` เขียนสั้นกว่า `class` ธรรมดามาก** แต่ก็ต้องแลกกับการที่ทุกการ
  "แก้ไข" ต้องผ่าน `with` เสมอ (ดู `mark_done`, `replace_task` ใน `TaskList`) — นี่คือข้อดี
  ของ immutability: ป้องกัน bug จากการที่หลายส่วนของโค้ดถือ reference เดียวกันแล้วมีใครบางคน
  แก้ไขค่าโดยไม่มีใครรู้ (state ที่แชร์กันแบบ mutable คือแหล่งบัคที่พบบ่อยที่สุดอย่างหนึ่งใน
  โปรแกรมขนาดใหญ่)
- **`TaskList` เป็นตัวอย่างที่ดีของการรวม 3 บทเรียนเข้าด้วยกัน**: `include Enumerable` (Part
  010/013) ทำให้กรอง/เรียงข้อมูลง่าย, custom exception (Part 011) ทำให้ error message ชัดเจน
  และจับแยกประเภทได้, และ `File`/`JSON` (Part 012) ทำให้ persist ข้อมูลได้จริง — ตัวอย่างชัดเจน
  ว่าโค้ดระดับมืออาชีพมักไม่ได้ใช้ "หนึ่งเทคนิค" แต่ผสมผสานหลายเทคนิคเข้าด้วยกันในไฟล์เดียว
- **`CLI` ไม่รู้จัก `Task` โดยตรงเลย** — มันคุยกับ `TaskList` เท่านั้น (เรียก `@list.add`,
  `@list.complete` ฯลฯ) ไม่เคยสร้าง `Task.new` เอง — นี่คือการรักษา **layer ของความรับผิดชอบ**
  ให้ชัดเจน (`CLI` ไม่ควรรู้วิธีสร้าง `Task` ที่ถูกต้อง แค่รู้ว่าจะขอให้ `TaskList` ทำให้)
- **การทดสอบ `CLI` ด้วย `StringIO` แทนการพิมพ์จริง** คือทักษะสำคัญที่จะใช้ซ้ำตลอดหลักสูตร —
  แนวคิดเดียวกันนี้จะกลับมาอีกครั้งตอนทดสอบ Rails Controller (`post`, `get` แทนการเปิด browser)
  ใน Phase 6

> **สะพานเชื่อมสู่ Rails:** โครงสร้างทั้งหมดของโปรเจกต์นี้ — `Task`/`TaskList` (ข้อมูล +
> business logic), `CLI` (รับ input, เรียก logic, แสดงผล), `Rakefile` (task อัตโนมัติ), `spec/`
> (test) — คือโครงร่างเดียวกับที่ Rails ใช้จริง เพียงแค่เปลี่ยนวิธีรับ input/แสดงผลจาก
> terminal เป็น HTTP request/response เท่านั้น: `TaskList` จะกลายเป็น **ActiveRecord Model**,
> `CLI` จะแยกเป็น **Controller** (รับ request, เรียก Model) และ **View** (render HTML แทนการ
> `puts`), ส่วน `Rakefile` และ `spec/` จะยังคงอยู่ในตำแหน่งเดิมเป๊ะๆ ไม่มีอะไรเปลี่ยน — เท่ากับ
> ว่า **สองในสี่ส่วนของโปรเจกต์นี้ (Rakefile, spec/) จะย้ายไป Rails ได้โดยแทบไม่ต้องเรียนรู้
> อะไรใหม่เลย**

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มวันครบกำหนด (due date) และการเรียงงานใกล้ครบกำหนดไว้บนสุด** — เพิ่มฟิลด์
   `due_date` ให้ `Task` (ใช้ `Date` จาก Part 001), แก้เมนู "เพิ่มงานใหม่" ให้ถามวันครบกำหนด
   (จะปล่อยว่างก็ได้โดยใช้ `nil`), แล้วเพิ่ม method `overdue?` ใน `Task` และ method
   `sorted_by_due_date` ใน `TaskList` ที่ใช้ `sort_by` โดยจัดการกรณี `due_date` เป็น `nil`
   ให้ถูกต้อง (ใบ้: ใช้ `sort_by { |t| t.due_date || Date::Infinity.new }` เพื่อให้งานที่ไม่มี
   กำหนดวันไปอยู่ท้ายสุดเสมอ)
2. **เพิ่มระดับความสำคัญ (priority)** — เพิ่มฟิลด์ `priority` ให้ `Task` (เช่น `:low`,
   `:medium`, `:high` เป็น Symbol) พร้อม validation ใน `TaskList#add` ว่าต้องเป็นค่าที่กำหนด
   ไว้เท่านั้น (raise `InvalidTaskError` ถ้าไม่ใช่) แล้วเพิ่มเมนูใหม่ "แสดงรายการเรียงตาม
   ความสำคัญ" ที่ใช้ `sort_by` ร่วมกับการแปลง Symbol เป็นตัวเลขเพื่อกำหนดลำดับ (high ก่อน
   medium ก่อน low)
3. **เพิ่ม tag และการกรองงานตาม tag** — เปลี่ยน `Task` ให้มีฟิลด์ `tags` เป็น Array ของ String
   (เช่น `["งานบ้าน", "ด่วน"]`), แก้ `to_h_for_json`/`from_h` ให้ serialize/deserialize Array
   นี้ได้ถูกต้อง, แล้วเพิ่มเมนู "กรองงานตาม tag" ที่รับ tag จากผู้ใช้แล้วใช้
   `select { |t| t.tags.include?(tag) }` — ลองต่อยอดเพิ่มเติมด้วย `group_by(&:tags)` (ทบทวน
   Part 013) เพื่อแสดงสรุปจำนวนงานแยกตาม tag ทั้งหมดในระบบ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Rake** คืออะไร ทำไมถึงมีอยู่ (ทางเลือกที่เป็น Ruby ล้วนแทน `Makefile`) และรู้ว่า
  มันติดตั้งมาพร้อม Ruby อยู่แล้วในฐานะ default gem
- นิยาม task พื้นฐานด้วย `task :name do ... end`, ใส่คำอธิบายด้วย `desc`, และใช้ `rake -T`/
  `rake -AT` ดูรายการ task ทั้งหมดในโปรเจกต์
- กำหนด **dependency ระหว่าง task** ด้วย `task build: [:clean, :compile]` และเข้าใจว่า Rake
  ป้องกันการรัน dependency ซ้ำซ้อนให้อัตโนมัติ
- จัดกลุ่ม task ด้วย **namespace** (`namespace :db do ... end`) เรียกผ่านชื่อเต็มแบบ
  `rake db:migrate` และเรียก task ข้าม namespace ด้วย `Rake::Task["..."].invoke`
- ส่ง **argument** เข้า task ได้ด้วย `task :name, [:arg] do |t, args| ... end` พร้อมเข้าใจ
  เรื่อง shell quoting และการตั้งค่า default ด้วย `args.with_defaults`
- ออกแบบและสร้างโปรเจกต์ **Todo CLI** ที่มีโครงสร้างไฟล์แบบมืออาชีพ แยกตามหน้าที่ความ
  รับผิดชอบ (`lib/todo_cli/{errors,task,task_list,cli}.rb`, `bin/todo`, `spec/`, `Rakefile`)
- ผสานทุกเทคนิคจาก Part 011–014 เข้าด้วยกันในโปรเจกต์เดียว: custom exception hierarchy,
  file I/O + JSON persistence, Enumerable สำหรับกรอง/เรียงข้อมูล, และ `Data.define` เป็น
  value object แบบ immutable
- เขียน **RSpec test suite ครบถ้วน** สำหรับ Model layer และ CLI (ทดสอบด้วย `StringIO`) แล้ว
  เชื่อมเข้ากับ Rake ผ่าน `RSpec::Core::RakeTask` เพื่อรันด้วยคำสั่งเดียว `rake spec`
- เข้าใจว่าโครงสร้างโปรเจกต์นี้ (`Rakefile`, `spec/`, การแยก data/logic/UI) จะเป็นรากฐานตรง
  ไปสู่โครงสร้างของ Rails application ในทุก Phase ถัดจากนี้

## สรุปภาพรวม Phase 2: Ruby Deep Dive

ยินดีด้วย! ตอนนี้ **Phase 2: Ruby Deep Dive (Part 011–020, Step 101–200)** เสร็จสมบูรณ์แล้ว
เราเดินทางจากการจัดการข้อผิดพลาดอย่างเป็นระบบด้วย Exception Handling (Part 011), การอ่าน/เขียน
ไฟล์และ JSON/CSV/YAML (Part 012), Enumerable ขั้นสูงสำหรับประมวลผลข้อมูล (Part 013),
Comparable/Struct/Data สำหรับออกแบบโครงสร้างข้อมูล (Part 014), Metaprogramming เบื้องต้น
(Part 015), Duck Typing และ SOLID principles (Part 016), Gem/Bundler และการสร้าง gem ของ
ตัวเอง (Part 017), การทดสอบด้วย Minitest (Part 018), RSpec แบบมืออาชีพ (Part 019) จนมาถึง
Rake และโปรเจกต์รวบยอด Todo CLI ใน Part นี้ — ครบทั้งเทคนิคขั้นสูงของภาษา Ruby เอง, วินัยการ
เขียนโค้ดที่ทดสอบได้ (testable) และดูแลรักษาง่าย (maintainable), และเครื่องมือ automation ที่
มืออาชีพทุกคนใช้จริงในงานประจำวัน ทักษะทั้งหมดนี้ไม่ใช่แค่ "ความรู้ Ruby เพิ่มเติม" แต่เป็น
**รากฐานที่ Rails framework ทั้งเฟรมเวิร์กถูกสร้างขึ้นมาอยู่บนนั้นโดยตรง** — Exception
hierarchy ที่เรียนใน Part 011 คือแบบเดียวกับที่ ActiveRecord ใช้, Enumerable ที่เรียนใน
Part 013 คือสิ่งที่ทำให้ `Post.where(...).map { ... }` ทำงานได้, และ Rakefile ที่เพิ่งเขียน
ใน Part นี้คือไฟล์เดียวกับที่จะเปิดเจอในทุกโปรเจกต์ Rails ตั้งแต่วันแรก

**ต่อไป (Part 021 — เปิด Phase 3: Rails Fundamentals — MVC):** ถึงเวลาที่รอคอยมาตลอด 20 Part
เราจะติดตั้ง **Ruby on Rails** เป็นครั้งแรก ด้วยคำสั่ง `gem install rails` และสร้างโปรเจกต์แรก
ด้วย `rails new` จากนั้นจะพาสำรวจ**โครงสร้างโฟลเดอร์ทั้งหมด**ที่ Rails สร้างให้อัตโนมัติ
(`app/`, `config/`, `db/`, `lib/`, และแน่นอนว่ารวมถึง `Rakefile` ที่เพิ่งเรียนจบไปหมาดๆ) พร้อม
รัน **Rails server** เป็นครั้งแรกเพื่อดูหน้าเว็บ "Yay! You're on Rails!" — จุดเริ่มต้นอย่างเป็น
ทางการของการเป็นนักพัฒนา Ruby on Rails
