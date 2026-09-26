# Part 028: Scaffold, Generator ของ Rails, และการอ่านโค้ดที่ Generate มา

> **Step ครอบคลุมใน Part นี้:** Step 271–280
> **ระดับ:** กลาง (ต้องผ่าน Part 021–025 มาก่อนทั้งหมด — โดยเฉพาะ Routing (Part 022),
> Controller (Part 023), View (Part 024) และ Model/ActiveRecord/Migration (Part 025) — เพราะ
> Part นี้ไม่ได้สอนแนวคิดใหม่เพิ่ม แต่สอนวิธี "ให้ Rails เขียนโค้ดที่เราเพิ่งเรียนมาทั้งหมดให้
> อัตโนมัติในคำสั่งเดียว")
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบรันจริงบน Ruby 3.3.6 +
> Rails 8.1.4 — สร้างแอปทดลองจริง รัน generator จริง อ่านไฟล์ที่เกิดขึ้นจริงทุกไฟล์ ไม่มีโค้ด
> ที่แต่งขึ้นมาลอยๆ)

จาก Part 022–025 (Routing, Controller, View, Model/ActiveRecord/Migration) เราเขียนทุกส่วน
ของ MVC ด้วยมือทีละไฟล์ — สร้าง route เอง เขียน controller เอง สร้าง view เอง เขียน migration
เอง เพื่อให้ **เข้าใจว่าแต่ละไฟล์ทำหน้าที่อะไรและทำไมต้องมี** มาถึง Part นี้ เราจะเห็นว่า Rails
มีเครื่องมือที่เขียนไฟล์เหล่านี้ **ให้เราทั้งหมดในคำสั่งเดียว** เรียกว่า **generator** — และตัวที่
ทรงพลังที่สุดในกลุ่มนี้คือ **scaffold**

สิ่งสำคัญที่ต้องเข้าใจให้ชัดตั้งแต่ต้น Part: **scaffold ไม่ได้สร้างโค้ดวิเศษที่ไม่เคยเห็นมาก่อน**
— ทุกบรรทัดที่ scaffold เขียนให้คือสิ่งที่เราเรียนมาแล้วทั้งหมดใน Part 022–025 (`resources`,
`before_action`, `params`, ERB, partial, migration) เพียงแค่ Rails รู้ pattern มาตรฐานที่คน
ส่วนใหญ่เขียนซ้ำกันทุกโปรเจกต์ (CRUD ครบ 7 action) จึงเขียนให้อัตโนมัติแทนที่จะให้เราพิมพ์เอง
ทีละบรรทัด — พูดอีกแบบ: **ถ้า Part 022–025 สอนให้อ่านออกเขียนได้ Part นี้จะสอนให้ "พิมพ์เร็วขึ้น"
โดยไม่เสียความเข้าใจไปเลยแม้แต่นิดเดียว** เพราะเราจะแกะโค้ดที่ generate มาดูทุกบรรทัดเหมือนเดิม
(validation และ association ที่ scaffold ยังไม่ใส่ให้ จะเป็นหัวข้อของ Part ถัดๆ ไป — Part นี้
จะแตะทั้งสองเรื่องแบบผิวเผินเท่าที่จำเป็นต่อการใช้ scaffold เท่านั้น)

## สารบัญของ Part นี้

- Step 271: Generator คืออะไร — ระบบ code generation ของ Rails และรายชื่อ generator ทั้งหมด
- Step 272: `rails generate scaffold` — รันจริงครั้งแรก และไล่ดู output แบบเต็ม
- Step 273: อ่าน Migration และ Model ที่ scaffold สร้าง — เทียบกับที่เขียนเองใน Part 025
- Step 274: อ่าน Controller ที่ scaffold สร้าง — `params.expect`, `respond_to`, เทียบกับ Part 023
- Step 275: อ่าน View: `index`, `show`, partial ของ record — Turbo Drive กับ Rails 8 สแคฟโฟลด์
- Step 276: อ่าน View: `new`, `edit`, `_form.html.erb` และ routes ที่ scaffold เพิ่มให้
- Step 277: Test file ที่ scaffold สร้างให้ และทำไมไม่มี system test มาให้อัตโนมัติอีกต่อไป
- Step 278: Generator ย่อยที่มีประโยชน์ — `generate model`, `generate controller`,
  `generate migration Add...To...`
- Step 279: `rails destroy` — ยกเลิก generator, ข้อควรระวังเรื่อง migration ที่ migrate ไปแล้ว,
  และการปรับแต่ง scaffold template ของทีม
- Step 280: เมื่อไหร่ควรใช้ scaffold ในงานจริง + แบบฝึกหัด — เพิ่ม `Comment` ที่ผูกกับ `Post`

---

## Step 271: Generator คืออะไร — ระบบ code generation ของ Rails และรายชื่อ generator ทั้งหมด

### ปัญหาที่ generator แก้

ย้อนกลับไปดู Part 022–025 อีกครั้ง: การสร้าง resource หนึ่งตัวให้ครบ (เช่น `Post`) ต้องทำ
อย่างน้อย 4 อย่างแยกกัน:

1. เขียน migration สร้างตาราง (Part 025)
2. เขียน Model class เปล่าๆ ที่สืบทอดจาก `ApplicationRecord` (Part 025)
3. เพิ่ม `resources :posts` ใน `config/routes.rb` (Part 022)
4. เขียน Controller ที่มีครบ 7 action พร้อม view คู่กันทุก action (Part 023–024)

งานทั้งหมดนี้เป็น **boilerplate** — โค้ดที่มีโครงสร้างซ้ำเดิมแทบทุกโปรเจกต์ Rails ต่างกันแค่ชื่อ
resource กับชื่อ field เท่านั้นเอง ถ้าต้องพิมพ์เองทุกไฟล์ทุกครั้งที่ทำ resource ใหม่ (ซึ่งเกิดขึ้น
บ่อยมากตอนเริ่มโปรเจกต์ หรือตอนทำ prototype) จะเสียเวลาไปกับงานที่ไม่ต้องใช้ความคิดสร้างสรรค์เลย

**Generator** คือระบบของ Rails (มาจาก gem `railties`) ที่เขียนไฟล์ boilerplate เหล่านี้ให้อัตโนมัติ
จาก template ที่ทีม Rails core (และ gem อื่นๆ) เตรียมไว้ล่วงหน้า โดยรับแค่ชื่อ resource กับชื่อ
field เป็น argument — เป็นกลไกเดียวกับที่เราใช้ไปแล้วโดยไม่รู้ตัวตั้งแต่ `rails new` (Part 021,
generate โครงสร้างทั้งโปรเจกต์), `rails generate model` (Part 025), และ
`rails generate controller` (Part 023) — Part นี้แค่เรียนรู้ระบบนี้อย่างเป็นทางการ พร้อมรู้จัก
ตัวที่ทรงพลังที่สุดคือ **scaffold**

### ดูรายชื่อ generator ทั้งหมดที่มีในแอป

```bash
bin/rails generate
```

ผลลัพธ์จริง (ทดสอบบน Rails 8.1.4 ในแอปที่สร้างด้วย `rails new` แบบเต็ม):

```
Please choose a generator below.

Rails:
  application_record
  authentication
  benchmark
  channel
  controller
  generator
  helper
  integration_test
  jbuilder
  job
  mailbox
  mailer
  migration
  model
  resource
  scaffold
  scaffold_controller
  script
  system_test
  task

ActiveRecord:
  active_record:application_record
  active_record:multi_db

SolidCable:
  solid_cable:install
  solid_cable:update

SolidCache:
  solid_cache:install

SolidQueue:
  solid_queue:install
  solid_queue:update

Stimulus:
  stimulus

TestUnit:
  test_unit:authentication
  test_unit:channel
  test_unit:generator
  test_unit:install
  test_unit:mailbox
```

สังเกตสิ่งสำคัญ: generator ไม่ได้มีแค่ของ Rails core (หมวด `Rails:`) เท่านั้น — **gem อื่นที่
ติดตั้งอยู่ในแอปก็เพิ่ม generator ของตัวเองเข้ามาในรายการนี้ได้** (หมวด `SolidCable:`,
`SolidQueue:`, `Stimulus:`, `TestUnit:` ล้วนมาจาก gem ที่ Rails 8 ติดตั้งให้ตั้งแต่ `rails new`)
— นี่คือเหตุผลที่ generator list ของแต่ละโปรเจกต์อาจไม่เหมือนกัน ขึ้นอยู่กับว่าติดตั้ง gem อะไรไว้
บ้าง (เช่นถ้าติดตั้ง `rspec-rails` ตาม Part 019 จะมีหมวด `Rspec:` เพิ่มเข้ามาแทนที่บางส่วนของ
`TestUnit:`)

### เรียกดูรายละเอียดของ generator ตัวใดตัวหนึ่ง

ทุก generator รองรับ `--help` เพื่อดู argument และ option ที่รองรับทั้งหมด:

```bash
bin/rails generate model --help
bin/rails generate controller --help
bin/rails generate scaffold --help
```

จะใช้คำสั่งนี้บ่อยตลอด Part นี้ เพราะเป็นวิธีที่เชื่อถือได้ที่สุดในการดูว่า generator ตัวหนึ่งรับ
option อะไรบ้าง — เร็วกว่าและตรงกับเวอร์ชัน Rails ที่ติดตั้งจริงมากกว่าการค้นหาจากบทความเก่าบน
อินเทอร์เน็ตที่อาจอ้างอิง Rails เวอร์ชันก่อนหน้า

### Generator ทั้งหมดที่เราเจอมาแล้วโดยไม่รู้ตัว

| คำสั่งที่เคยใช้ | เจอครั้งแรกใน | สร้างอะไร |
|---|---|---|
| `rails new` | Part 021 | สร้างทั้งโปรเจกต์ (จริงๆ คือ generator ตัวหนึ่งเหมือนกัน แค่เรียกผ่าน `rails` ไม่ใช่ `rails generate`) |
| `rails generate controller` | Part 023 | Controller + route + view เปล่า + helper + test |
| `rails generate model` | Part 025 | Model + migration + test + fixture |
| `rails generate migration` | Part 025 | migration file เปล่า (หรือเดาเนื้อหาให้จากชื่อ) |
| `rails generate channel` | Part 021 (Step 205) | Channel สำหรับ Action Cable |

สังเกตว่าทุกตัวข้างต้นคือ generator **ย่อย** ที่สร้างแค่ "ส่วนหนึ่ง" ของ resource เท่านั้น — Part
นี้จะแนะนำ **`scaffold`** ซึ่งเป็น generator ที่**เรียก generator ย่อยเหล่านี้ต่อกันหมดในคำสั่ง
เดียว** (จะเห็นชัดเจนจาก output ใน Step 272 ที่มีบรรทัด `invoke active_record`,
`invoke scaffold_controller`, `invoke erb` ต่อกันเป็นทอดๆ)

---

## Step 272: `rails generate scaffold` — รันจริงครั้งแรก และไล่ดู output แบบเต็ม

### เตรียมแอปทดลอง

```bash
mkdir -p ~/ruby-course-workspace/part-028
cd ~/ruby-course-workspace/part-028
rails new blog_app
cd blog_app
```

สังเกตว่ารอบนี้เรา**ไม่ใส่ `--minimal`** (ต่างจากที่ Part 021 บอกไว้ว่าเฟส 3 จะใช้ `--minimal`
เป็นหลัก) เพราะ Part นี้ต้องการเห็น scaffold ทำงานแบบเต็มรูปแบบ รวมถึงส่วนที่เกี่ยวกับ JSON
response (ผ่าน gem `jbuilder`) ซึ่งเป็นสิ่งที่ `--minimal` ตัดออกไป (ทบทวนตาราง Part 021 Step
203) — ท้าย Step 274 จะมีหมายเหตุเปรียบเทียบให้เห็นว่าถ้าใช้ `--minimal` output จะสั้นกว่านี้
อย่างไร

### รัน scaffold ครั้งแรก

```bash
bin/rails generate scaffold Post title:string body:text
```

Output จริงทั้งหมด (ทดสอบบน Rails 8.1.4):

```
      invoke  active_record
      create    db/migrate/20260926030346_create_posts.rb
      create    app/models/post.rb
      invoke    test_unit
      create      test/models/post_test.rb
      create      test/fixtures/posts.yml
      invoke  resource_route
       route    resources :posts
      invoke  scaffold_controller
      create    app/controllers/posts_controller.rb
      invoke    erb
      create      app/views/posts
      create      app/views/posts/index.html.erb
      create      app/views/posts/edit.html.erb
      create      app/views/posts/show.html.erb
      create      app/views/posts/new.html.erb
      create      app/views/posts/_form.html.erb
      create      app/views/posts/_post.html.erb
      invoke    resource_route
      invoke    test_unit
      create      test/controllers/posts_controller_test.rb
      invoke    helper
      create      app/helpers/posts_helper.rb
      invoke      test_unit
      invoke    jbuilder
      create      app/views/posts/index.json.jbuilder
      create      app/views/posts/show.json.jbuilder
      create      app/views/posts/_post.json.jbuilder
```

### วิเคราะห์ syntax ของคำสั่ง

```
rails generate scaffold <ชื่อ Model เอกพจน์> <field>:<type> <field>:<type> ...
```

รูปแบบเดียวกันเป๊ะกับ `rails generate model` ที่เรียนไปใน Part 025 Step 242 — เพราะ scaffold
**เรียก** `generate model` เป็นขั้นตอนแรกจริงๆ (สังเกตบรรทัด `invoke active_record` ที่ขึ้นก่อน
เป็นอันดับแรก)

### แกะ output ทีละบรรทัด — scaffold คือ generator ที่เรียก generator ย่อยต่อกัน

อ่าน indentation และคำว่า `invoke`/`create` ในผลลัพธ์ให้ดี มันบอกลำดับการทำงานทั้งหมด:

| ลำดับ | บรรทัด `invoke` | หน้าที่ | เทียบเท่า generator ที่เคยใช้แยก |
|---|---|---|---|
| 1 | `invoke active_record` | สร้าง migration + Model + model test + fixture | `rails generate model` (Part 025) |
| 2 | `invoke resource_route` | เพิ่ม `resources :posts` ใน `config/routes.rb` | เขียนเองใน Part 022 |
| 3 | `invoke scaffold_controller` | สร้าง Controller ที่มีครบ 7 action | เขียนเองใน Part 023 |
| 4 | `invoke erb` | สร้าง view ทั้ง 6 ไฟล์ (`.html.erb`) | เขียนเองใน Part 024 |
| 5 | `invoke test_unit` (ซ้อนใน scaffold_controller) | สร้าง controller test | — |
| 6 | `invoke helper` | สร้าง `app/helpers/posts_helper.rb` เปล่า | เหมือนที่ `generate controller` ทำใน Part 023 |
| 7 | `invoke jbuilder` | สร้าง view สำหรับตอบ JSON (`.json.jbuilder`) | จะเรียนเต็มรูปแบบใน Phase 8 (Part 056) |

**สรุปเป็นภาพเดียว:** `scaffold` = `model` + `resources` ใน routes + `scaffold_controller`
(controller ที่ฉลาดกว่า controller เปล่าของ `generate controller`) + `erb` (view 6 ไฟล์) +
`jbuilder` (view JSON 3 ไฟล์ ถ้ามี gem `jbuilder`) + test file ของทุกชั้น — **ทุกอย่างที่ Part
022–025 สอนแยกกันคนละ Part มารวมกันอยู่ในคำสั่งเดียวนี้ทั้งหมด**

### รายการไฟล์ทั้งหมดที่ถูกสร้าง (สรุปให้เห็นภาพรวมก่อนแกะทีละไฟล์)

เรียงจาก output ใน Step 272 ให้เป็นรายการไฟล์ล้วนๆ (ไม่รวมโฟลเดอร์):

```
db/migrate/20260926030346_create_posts.rb
app/models/post.rb
test/models/post_test.rb
test/fixtures/posts.yml
config/routes.rb                          (แก้ไข ไม่ใช่ไฟล์ใหม่)
app/controllers/posts_controller.rb
app/views/posts/index.html.erb
app/views/posts/edit.html.erb
app/views/posts/show.html.erb
app/views/posts/new.html.erb
app/views/posts/_form.html.erb
app/views/posts/_post.html.erb
test/controllers/posts_controller_test.rb
app/helpers/posts_helper.rb
app/views/posts/index.json.jbuilder
app/views/posts/show.json.jbuilder
app/views/posts/_post.json.jbuilder
```

รวมทั้งหมด **16 ไฟล์ใหม่ + แก้ไข 1 ไฟล์เดิม** จากคำสั่งเดียว — Step 273–277 จะไล่เปิดอ่านทุกไฟล์
เหล่านี้ทีละกลุ่ม เพื่อยืนยันว่าไม่มี "เวทมนตร์" อะไรซ่อนอยู่เลย ทุกบรรทัดอธิบายได้ด้วยความรู้จาก
Part 022–025

---

## Step 273: อ่าน Migration และ Model ที่ scaffold สร้าง — เทียบกับที่เขียนเองใน Part 025

### Migration

```ruby
# db/migrate/20260926030346_create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

เทียบกับ migration ที่เขียนเองใน **Part 025 Step 243** (`rails generate model Post
title:string body:text published:boolean`) — **โครงสร้างเหมือนกันทุกประการ** เพราะ scaffold
เรียก generator `active_record` ตัวเดียวกับที่ `generate model` ใช้ ต่างกันแค่ field ที่เราสั่ง
ต่อท้ายชื่อ Model เท่านั้น รัน migrate ตามปกติ:

```bash
bin/rails db:create db:migrate
```

```
Created database 'storage/development.sqlite3'
Created database 'storage/test.sqlite3'
== 20260926030346 CreatePosts: migrating ======================================
-- create_table(:posts)
   -> 0.0015s
== 20260926030346 CreatePosts: migrated (0.0016s) =============================
```

### Model

```ruby
# app/models/post.rb
class Post < ApplicationRecord
end
```

**ว่างเปล่าเหมือนที่เห็นใน Part 025 ทุกประการ** — ข้อสังเกตสำคัญที่ต้องเน้นย้ำ: **scaffold ไม่ใส่
validation หรือ association ให้เองเลยแม้แต่บรรทัดเดียว** ทั้งที่ `title`/`body` ดูเหมือนควรมี
`validates :title, presence: true` เป็นอย่างน้อย — นี่คือขอบเขตที่ตั้งใจของ scaffold: **มันสร้าง
"โครงร่างที่ทำงานได้" ให้เท่านั้น ไม่ใช่ "แอปที่พร้อมใช้งานจริง"** การเพิ่ม validation (สอนใน
Part 026) และ association (สอนใน Part 027) ยังคงเป็นหน้าที่ของนักพัฒนาที่ต้องเติมเข้าไปเองเสมอ
เป็นประเด็นที่จะเจาะลึกอีกครั้งใน Step 280

> **ทดสอบจริงแล้วยืนยัน:** ลองสร้าง Post ด้วย `title` เป็น `nil` ผ่าน `rails console` ทันทีหลัง
> scaffold (ยังไม่ทันเพิ่ม validation เอง) — `Post.create(title: nil, body: "x")` จะสำเร็จ
> (`persisted?` เป็น `true`) เหมือนที่ Part 025 Step 249 เคยเตือนไว้แล้วว่า ActiveRecord ไม่
> ป้องกันอะไรให้โดย default

### เปรียบเทียบสรุป: สร้างเอง (Part 025) vs scaffold (Part นี้)

| | `rails generate model` (Part 025) | `rails generate scaffold` (Part นี้) |
|---|---|---|
| Migration | ✅ เหมือนกัน | ✅ เหมือนกัน |
| Model | ✅ เหมือนกัน (ว่างเปล่า) | ✅ เหมือนกัน (ว่างเปล่า) |
| Route | ❌ ไม่เพิ่มให้ | ✅ เพิ่ม `resources :posts` ให้ |
| Controller | ❌ ไม่สร้างให้ | ✅ สร้างครบ 7 action |
| View | ❌ ไม่สร้างให้ | ✅ สร้างครบ 6 ไฟล์ + JSON 3 ไฟล์ |
| Helper | ❌ ไม่สร้างให้ | ✅ สร้างให้ (ว่างเปล่า) |
| Test | Model test เท่านั้น | Model test + Controller test |

---

## Step 274: อ่าน Controller ที่ scaffold สร้าง — `params.expect`, `respond_to`, เทียบกับ Part 023

นี่คือไฟล์ที่น่าสนใจที่สุดของ scaffold เพราะรวมทุกอย่างที่เรียนใน Part 023 ไว้ในที่เดียว:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :set_post, only: %i[ show edit update destroy ]

  # GET /posts or /posts.json
  def index
    @posts = Post.all
  end

  # GET /posts/1 or /posts/1.json
  def show
  end

  # GET /posts/new
  def new
    @post = Post.new
  end

  # GET /posts/1/edit
  def edit
  end

  # POST /posts or /posts.json
  def create
    @post = Post.new(post_params)

    respond_to do |format|
      if @post.save
        format.html { redirect_to @post, notice: "Post was successfully created." }
        format.json { render :show, status: :created, location: @post }
      else
        format.html { render :new, status: :unprocessable_content }
        format.json { render json: @post.errors, status: :unprocessable_content }
      end
    end
  end

  # PATCH/PUT /posts/1 or /posts/1.json
  def update
    respond_to do |format|
      if @post.update(post_params)
        format.html { redirect_to @post, notice: "Post was successfully updated.", status: :see_other }
        format.json { render :show, status: :ok, location: @post }
      else
        format.html { render :edit, status: :unprocessable_content }
        format.json { render json: @post.errors, status: :unprocessable_content }
      end
    end
  end

  # DELETE /posts/1 or /posts/1.json
  def destroy
    @post.destroy!

    respond_to do |format|
      format.html { redirect_to posts_path, notice: "Post was successfully destroyed.", status: :see_other }
      format.json { head :no_content }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_post
      @post = Post.find(params.expect(:id))
    end

    # Only allow a list of trusted parameters through.
    def post_params
      params.expect(post: [ :title, :body ])
    end
end
```

ไล่อ่านทีละส่วน เทียบกับสิ่งที่เรียนมาแล้วใน Part 023:

### 1) `before_action :set_post, only: %i[ show edit update destroy ]`

ตรงกับที่เรียน Step 228 เป๊ะ — action ที่ต้องใช้ `@post` ตัวเดียวกัน (4 action: show, edit,
update, destroy) ถูกดึงมารวมไว้ที่ `before_action` เดียวแทนที่จะเขียน `Post.find(...)` ซ้ำ 4 ที่
— นี่คือตัวอย่างจริงของ **DRY** ที่ Step 228 อธิบายไว้ ปรากฏเป็นรูปธรรมทันทีที่ scaffold สร้างให้

`%i[ show edit update destroy ]` คือ `%i[]` literal สำหรับ array ของ symbol (`[:show, :edit,
:update, :destroy]`) — เขียนสั้นกว่า ทีม Rails core เลือกใช้ syntax นี้เป็นมาตรฐานในทุก
generator template

### 2) `params.expect` — strong parameters เวอร์ชันใหม่ของ Rails (แทนที่ `require`/`permit`)

```ruby
def set_post
  @post = Post.find(params.expect(:id))
end

def post_params
  params.expect(post: [ :title, :body ])
end
```

นี่คือจุดที่**ต่างจากบทความ/หลักสูตร Rails รุ่นเก่าที่ยังสอน syntax แบบ**
`params.require(:post).permit(:title, :body)` **อย่างชัดเจน** — `params.expect` เป็น API ใหม่
(เริ่มมีตั้งแต่ Rails 7.1 และเป็นค่า default ที่ scaffold generate ให้ตั้งแต่ Rails 8) ที่ทำงาน
คล้ายกันมาก (กรอง key ที่ไม่ได้อนุญาตออก ป้องกัน mass assignment vulnerability ตามที่ Part 023
Step 224 เกริ่นไว้) แต่มีข้อดีเพิ่มขึ้น 2 อย่าง:

1. **ตรวจรูปร่างของ params เข้มงวดกว่า** — `params.expect(post: [:title, :body])` บอกชัดเจนว่า
   ต้องมี key `post` เป็น Hash ที่มี `title`/`body` ข้างใน ถ้า client ส่งรูปร่างผิด (เช่นส่ง
   `post` มาเป็น Array แทน Hash) จะโยน `ActionController::ParameterMissing` ทันที ป้องกันช่อง
   โหว่บางแบบที่ `require`/`permit` แบบเก่าพลาดได้
2. **`params.expect(:id)` แทน `params.require(:id)`** — ใช้แม้กับ scalar parameter เดี่ยวๆ
   (ไม่ใช่แค่ nested hash) ทำให้ codebase ใช้ API เดียวสม่ำเสมอทั้งไฟล์

> **หมายเหตุสำหรับการอ่านโค้ดในโลกจริง:** โปรเจกต์ Rails ที่มีอยู่ก่อนหน้า Rails 7.1 (และ
> ตำรา/บทความ/คอร์สจำนวนมากในอินเทอร์เน็ต) ยังคงใช้ `params.require(:post).permit(:title,
> :body)` อยู่ — ทั้งสองแบบทำงานได้ผลลัพธ์เทียบเท่ากันในกรณีทั่วไป และ `require`/`permit` ยังไม่
> ถูกถอดออกจาก Rails (ใช้ได้ต่อไปตามปกติ) เพียงแต่ **scaffold ของ Rails 8.1 เลือก generate
> แบบใหม่ (`expect`) ให้เป็นค่าเริ่มต้นแล้ว** — เจาะลึกเรื่อง strong parameters ทั้งสองรูปแบบแบบ
> เต็มรูปแบบใน **Part 031**

### 3) `respond_to` + `format.html`/`format.json`

```ruby
respond_to do |format|
  if @post.save
    format.html { redirect_to @post, notice: "Post was successfully created." }
    format.json { render :show, status: :created, location: @post }
  else
    format.html { render :new, status: :unprocessable_content }
    format.json { render json: @post.errors, status: :unprocessable_content }
  end
end
```

`respond_to` คือกลไกที่ให้ 1 action ตอบกลับได้**หลายรูปแบบ (format)** ตามที่ client ร้องขอ —
ทบทวนจาก Part 024 Step 231: ชื่อไฟล์ view อย่าง `index.html.erb` มีส่วน `.html` เป็น **format**
`respond_to` คือจุดที่ controller ใช้ตัดสินใจว่าจะ render format ไหนตาม `Accept` header หรือ
นามสกุลท้าย URL (`.json`) ที่ client ส่งมา:

- `format.html { ... }` — ทำงานเมื่อ client ขอ HTML (browser ทั่วไปที่เข้าผ่าน `/posts`)
- `format.json { ... }` — ทำงานเมื่อ client ขอ JSON (เช่นเข้า `/posts.json` หรือส่ง header
  `Accept: application/json`) — ใช้ `render :show` ซึ่งจะไป render ไฟล์ `show.json.jbuilder`
  (ดู Step 275) แทนที่จะ render `show.html.erb`

ทดสอบจริงให้เห็นทั้งสองแบบพร้อมกัน (หลัง `bin/rails server` และมี post `id: 1` อยู่แล้ว):

```bash
curl -s http://localhost:3000/posts/1 | grep -o '<h1>.*</h1>\|<div>.*Title.*'
curl -s http://localhost:3000/posts/1.json
```

```
{"id":1,"title":"Hello Rails","body":"...","created_at":"...","updated_at":"...","url":"http://localhost:3000/posts/1.json"}
```

**นี่คือแอปเดียวกัน คนละ URL แค่ต่อ `.json` ท้าย path ก็ได้ API response ทันทีโดยไม่ต้องเขียน
controller เพิ่มอีกตัว** — เป็นตัวอย่างที่ทำให้เห็นภาพชัดว่า Rails ออกแบบมาให้ 1 controller
ให้บริการได้ทั้งเว็บหน้าเต็มและ API แบบง่ายๆ ไปพร้อมกัน (การทำ API แบบเต็มรูปแบบ พร้อม
versioning และ serializer ที่เหมาะสมกว่า jbuilder ตรงๆ จะเรียนใน **Phase 8 ตั้งแต่ Part 056**)

### 4) `status: :unprocessable_content` และ `status: :see_other` — รายละเอียดที่เปลี่ยนใน Rails 8

สองจุดนี้ต่างจากโค้ดตัวอย่างเก่าที่อาจเคยเห็นในบทความ/หลักสูตรอื่น:

- **`status: :unprocessable_content`** (แทนที่ `:unprocessable_entity` แบบเดิม) — HTTP status
  code 422 ถูกเปลี่ยนชื่ออย่างเป็นทางการใน RFC 9110 จาก "Unprocessable Entity" เป็น
  "Unprocessable Content" Rails 7.1 เพิ่ม symbol ใหม่นี้เป็น alias และ Rails 8.1 เปลี่ยนมาใช้
  ชื่อใหม่เป็นค่า default ในทุก generator template แล้ว — `:unprocessable_entity` เดิมยังใช้
  ได้ปกติ (backward compatible) ความหมายเลข status code เหมือนกันทุกประการ (422)
- **`status: :see_other`** (303) — ต่อจาก Part 023 Step 227 เรื่อง Post/Redirect/Get: การ
  redirect ปกติใช้ 302 Found แต่ Turbo (ที่ผูกมากับทุกแอป Rails ตั้งแต่ Rails 7 ตามที่ Part 024
  Step 237 เคยพูดถึง) ต้องการให้ redirect หลัง `PATCH`/`DELETE` เป็น **303 See Other** โดยเฉพาะ
  เพราะตาม HTTP spec แล้ว 303 คือ status เดียวที่บอก client อย่างชัดเจนว่า **"ให้ตามไปด้วย
  `GET` เท่านั้น ไม่ว่า request เดิมจะเป็น method อะไรก็ตาม"** (302 มีความหมายกำกวมกว่า —
  client บางตัวอาจตามไปด้วย method เดิม) เพราะ Turbo ทำงานผ่าน `fetch()` ของ JavaScript ที่ยึด
  ตาม spec นี้เป๊ะ ถ้า redirect หลัง `PATCH`/`DELETE` เป็น 302 ธรรมดา Turbo อาจพยายามยิง
  `PATCH`/`DELETE` ซ้ำไปที่หน้าใหม่โดยไม่ตั้งใจ — นี่คือรายละเอียดเชิงลึกของ Turbo ที่ scaffold
  จัดการให้อัตโนมัติแล้วโดยที่เราไม่ต้องรู้ก็ใช้งานได้ (จะเจาะลึก Turbo เต็มรูปแบบใน Part 051–052)

### เทียบโครงสร้างกับ Controller ที่เขียนเองใน Part 023

| แนวคิดจาก Part 023 | ปรากฏใน scaffold controller ตรงไหน |
|---|---|
| `before_action` + `only:` (Step 228) | `before_action :set_post, only: %i[ show edit update destroy ]` |
| private helper method (Step 228) | `set_post`, `post_params` อยู่ใต้ `private` |
| `redirect_to` vs `render` + PRG (Step 227) | `create`/`update` สำเร็จ → `redirect_to`, ล้มเหลว → `render :new`/`:edit` |
| `flash`/`notice:` shorthand (Step 226) | `redirect_to @post, notice: "..."` — `notice:` เป็น shortcut ของการตั้ง `flash[:notice]` ก่อน redirect |
| mass assignment (เกริ่นไว้ Step 224) | `post_params` กรอง key ก่อนส่งเข้า `Post.new`/`.update` เสมอ |

**ไม่มีอะไรใน controller นี้ที่ Part 023 ไม่ได้ปูพื้นไว้แล้ว** — scaffold แค่ประกอบทุกอย่างเข้า
ด้วยกันตาม pattern ที่ทีม Rails core ตกผลึกมาจากประสบการณ์สร้างเว็บแอปนับพันโปรเจกต์

### ถ้าใช้ `--minimal` (ไม่มี gem `jbuilder`) จะได้ controller แบบไหน

ทดสอบเปรียบเทียบจริงด้วยแอปที่สร้างด้วย `rails new blog_app_minimal --minimal` แล้วรัน scaffold
เดียวกัน — ได้ controller ที่**สั้นกว่ามาก**เพราะไม่มี gem `jbuilder` ให้ตอบ JSON:

```ruby
# posts_controller.rb เมื่อสร้างจากแอปแบบ --minimal (ไม่มี respond_to/format.json เลย)
def create
  @post = Post.new(post_params)

  if @post.save
    redirect_to @post, notice: "Post was successfully created."
  else
    render :new, status: :unprocessable_content
  end
end
```

**บทเรียนสำคัญ:** generator ไม่ได้ generate โค้ดแบบตายตัวเสมอไป — มันตรวจสอบ **gem ที่ติดตั้งอยู่
ในแอปจริง** (`Gemfile`/`Gemfile.lock`) ก่อนตัดสินใจว่าจะสร้างอะไรให้บ้าง ถ้าไม่มี `jbuilder`
ก็จะไม่มี `respond_to`/`format.json`/ไฟล์ `.json.jbuilder` เกิดขึ้นเลย เพราะ generate โค้ดที่
เรียก gem ที่ไม่มีอยู่จริงไปก็ใช้งานไม่ได้อยู่ดี

---

## Step 275: อ่าน View — `index`, `show`, partial ของ record — Turbo Drive กับ Rails 8 สแคฟโฟลด์

### `app/views/posts/index.html.erb`

```erb
<p style="color: green"><%= notice %></p>

<% content_for :title, "Posts" %>

<h1>Posts</h1>

<div id="posts">
  <% @posts.each do |post| %>
    <%= render post %>
    <p>
      <%= link_to "Show this post", post %>
    </p>
  <% end %>
</div>

<%= link_to "New post", new_post_path %>
```

ไล่ดูทีละบรรทัด เทียบกับ Part 024:

- `<%= notice %>` — ทบทวน Part 023 Step 226: `redirect_to @post, notice: "..."` ตั้งค่า
  `flash[:notice]` ให้ Rails มี **helper method ชื่อ `notice`** ที่เป็นทางลัดของ
  `flash[:notice]` ให้เรียกตรงๆ ในทุก view โดยไม่ต้องพิมพ์ `flash[:notice]` เต็มๆ (มี `alert`
  เป็น shortcut คู่กันสำหรับ `flash[:alert]` เช่นกัน)
- `content_for :title, "Posts"` — ตรงกับ Part 024 Step 239 เป๊ะ ใช้ตั้งชื่อ `<title>` เฉพาะหน้า
  ผ่าน layout ที่มี `<title><%= content_for(:title) || "..." %></title>` (ทบทวน Part 021
  Step 204 เรื่อง layout ที่ Rails 8 generate ให้)
- `<%= render post %>` — ทบทวน Part 024 Step 236 (รูปย่อสุดของการ render collection): Rails
  เดา partial จาก class ของ `post` เอง (`Post` → `posts/_post`) เพราะ `Post` เป็น
  ActiveRecord model ที่ implement `to_partial_path` ให้อยู่แล้ว
- `<div id="posts">` — id ธรรมดาที่ครอบ list ไว้ ยังไม่ใช่ Turbo Frame ที่แท้จริง (อธิบายเพิ่ม
  ด้านล่าง)

### `app/views/posts/_post.html.erb` (partial ที่ index/show ใช้ร่วมกัน)

```erb
<div id="<%= dom_id post %>">
  <div>
    <strong>Title:</strong>
    <%= post.title %>
  </div>

  <div>
    <strong>Body:</strong>
    <%= post.body %>
  </div>

</div>
```

**`dom_id post`** คือ helper ใหม่ที่ Part 024 ยังไม่ได้พูดถึง — คืนค่า string ที่ไม่ซ้ำกันสำหรับ
แต่ละ record เพื่อใช้เป็น HTML `id` attribute เช่น `dom_id(Post.find(1))` ได้ `"post_1"`
ทดสอบจริง:

```irb
irb(main):001> dom_id(Post.find(1))
=> "post_1"
irb(main):002> dom_id(Post.new)
=> "new_post"
```

**เหตุผลที่ scaffold ใส่ `dom_id` มาให้ตั้งแต่ต้น (จุดที่เชื่อมกับ Turbo Frames/Streams):**
`id="post_1"` นี้คือ**จุดเกาะ (anchor)** มาตรฐานที่ Turbo Streams (Part 052) ใช้อ้างอิงเวลาต้อง
อัปเดต/ลบ element หนึ่งตัวบนหน้าเว็บแบบ real-time โดยไม่ต้อง reload ทั้งหน้า (เช่น ลบ post
ตัวหนึ่งแล้วให้ `<div id="post_1">` หายไปจากหน้าจอทันทีโดยไม่ต้อง JavaScript เอง) — scaffold
เตรียม "จุดเกาะ" นี้ไว้ล่วงหน้าให้แล้ว แม้ว่า ณ Part นี้เรายังไม่ได้ต่อยอดไปใช้ Turbo Streams จริงๆ
ก็ตาม

> **เรื่อง Turbo Frames/Streams ใน scaffold ของ Rails 8 — สิ่งที่มีจริงกับสิ่งที่ยังไม่มี:**
> ทดสอบจริงแล้วยืนยันว่า scaffold ของ Rails 8.1 **ไม่ได้ห่อ view ด้วย `turbo_frame_tag` ให้
> อัตโนมัติ** (ต่างจากที่บางคนอาจเข้าใจผิด) สิ่งที่มีให้จริงๆ คือ 2 อย่าง: (1) **Turbo Drive**
> ทำงานอยู่เบื้องหลังทั้งแอปตั้งแต่ `rails new` (เพราะ gem `turbo-rails` อยู่ใน `Gemfile`
> เริ่มต้น) ทำให้การคลิก link/submit form **ไม่ reload หน้าทั้งหน้า** แต่ยิง request แบบ
> AJAX แล้วสลับแค่ `<body>` ให้ความรู้สึกเหมือน SPA (Single Page App) โดยที่ scaffold ไม่ต้อง
> เขียนโค้ดอะไรเพิ่มเลยแม้แต่บรรทัดเดียว และ (2) `dom_id` ที่เตรียมไว้ให้ตามที่อธิบายข้างต้น
> การห่อ `turbo_frame_tag`/เขียน `*.turbo_stream.erb` เพื่ออัปเดตบางส่วนของหน้าแบบเจาะจง
> (ไม่ reload ทั้ง `<body>`) เป็นสิ่งที่นักพัฒนาต้องเพิ่มเข้าไปเอง — เจาะลึกเต็มรูปแบบใน
> **Part 051 (Turbo Frames)** และ **Part 052 (Turbo Streams)**

### `app/views/posts/show.html.erb`

```erb
<p style="color: green"><%= notice %></p>

<%= render @post %>

<div>
  <%= link_to "Edit this post", edit_post_path(@post) %> |
  <%= link_to "Back to posts", posts_path %>

  <%= button_to "Destroy this post", @post, method: :delete %>
</div>
```

`button_to` เป็น helper ที่ยังไม่เคยเห็นมาก่อน — ต่างจาก `link_to` ตรงที่ **render ออกมาเป็น
`<form>` พร้อมปุ่ม submit** แทนที่จะเป็น `<a>` ธรรมดา:

```html
<form class="button_to" method="post" action="/posts/1">
  <input type="hidden" name="_method" value="delete">
  <button type="submit">Destroy this post</button>
  <input type="hidden" name="authenticity_token" value="...">
</form>
```

**ทำไมต้องใช้ `button_to` แทน `link_to` + `data: { turbo_method: :delete }`** (ที่ Part 024
Step 237 สอนไว้): ทั้งสองวิธีทำงานได้ผลลัพธ์เดียวกันในทางปฏิบัติ (ส่ง `DELETE` request) แต่
`button_to` สร้าง `<form>` จริงที่ทำงานได้แม้ **ปิด JavaScript** (เพราะเป็น HTML form ธรรมดาที่
ส่ง `_method=delete` ให้ Rails middleware แปลงเป็น DELETE ให้ ไม่ต้องพึ่ง Turbo intercept
click event เลย) ส่วน `link_to ... data: { turbo_method: :delete }` ต้องพึ่ง JavaScript
(Turbo) ทำงานอยู่เสมอถึงจะส่ง request ถูก method — scaffold เลือก `button_to` เพราะทนทานกว่า
เป็นค่า default ที่ปลอดภัยกว่าสำหรับปุ่มที่มีผลทำลายข้อมูล (destroy)

---

## Step 276: อ่าน View — `new`, `edit`, `_form.html.erb` และ routes ที่ scaffold เพิ่มให้

### `app/views/posts/new.html.erb` และ `edit.html.erb`

```erb
<%# app/views/posts/new.html.erb %>
<% content_for :title, "New post" %>

<h1>New post</h1>

<%= render "form", post: @post %>

<br>

<div>
  <%= link_to "Back to posts", posts_path %>
</div>
```

```erb
<%# app/views/posts/edit.html.erb %>
<% content_for :title, "Editing post" %>

<h1>Editing post</h1>

<%= render "form", post: @post %>

<br>

<div>
  <%= link_to "Show this post", @post %> |
  <%= link_to "Back to posts", posts_path %>
</div>
```

สังเกตว่าทั้งสองไฟล์เหมือนกันเกือบทั้งหมด ต่างแค่หัวข้อกับลิงก์ท้ายหน้า — เพราะฟอร์มจริงๆ (การรับ
`title`/`body`) ถูกแยกออกไปเป็น partial ต่างหากแล้วเรียกใช้ร่วมกันทั้งคู่ ตรงตามหลัก DRY ที่
Part 024 Step 235 สอนไว้เป๊ะ

### `app/views/posts/_form.html.erb` — หัวใจของทั้งสองหน้า

```erb
<%= form_with(model: post) do |form| %>
  <% if post.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(post.errors.count, "error") %> prohibited this post from being saved:</h2>

      <ul>
        <% post.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :title, style: "display: block" %>
    <%= form.text_field :title %>
  </div>

  <div>
    <%= form.label :body, style: "display: block" %>
    <%= form.textarea :body %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

### `form_with(model: post)` — ทำไมฟอร์มเดียวใช้ได้ทั้ง new และ edit

นี่คือกลไกที่ยังไม่เคยเจอมาก่อนใน Part 022–024 (Part 024 สอนแค่ `render`/partial แต่ไม่ได้ลง
ลึกเรื่อง form) `form_with(model: post)` **ฉลาดพอที่จะสร้าง URL และ HTTP method ให้เองโดย
เดาจากสถานะของ `post`**:

```irb
irb(main):001> new_post = Post.new
irb(main):002> new_post.persisted?
=> false

irb(main):003> existing_post = Post.find(1)
irb(main):004> existing_post.persisted?
=> true
```

- ถ้า `post.persisted?` เป็น `false` (มาจากหน้า `new` ที่ `@post = Post.new`) →
  `form_with` render form ที่ `action="/posts"` และ `method="post"` (ไป action `create`)
- ถ้า `post.persisted?` เป็น `true` (มาจากหน้า `edit` ที่ `@post = Post.find(params[:id])`) →
  `form_with` render form ที่ `action="/posts/1"` พร้อม hidden field `_method=patch`
  (ไป action `update`)

ยืนยันด้วย HTML จริงที่ render ออกมาสองกรณี:

```html
<!-- render จากหน้า new (post ยังไม่ persisted) -->
<form action="/posts" accept-charset="UTF-8" method="post">
  ...
</form>

<!-- render จากหน้า edit (post persisted แล้ว, id=1) -->
<form action="/posts/1" accept-charset="UTF-8" method="post">
  <input type="hidden" name="_method" value="patch" autocomplete="off">
  ...
</form>
```

นี่คือเหตุผลที่ partial `_form.html.erb` ไฟล์เดียวใช้ได้กับทั้งสองสถานการณ์โดยไม่ต้องเขียน
เงื่อนไข if/else เอง — `form_with` ตัดสินใจให้หมดจากแค่ `post.persisted?` ตัวเดียว
(รายละเอียดเชิงลึกของ `form_with` ทั้งหมด รวมถึง `fields_for`, nested attributes จะเรียนเต็ม
รูปแบบใน **Part 031**)

### `post.errors` — จุดที่โยงกับ Part 026 (แม้ Part นี้จะมาก่อน)

```erb
<% if post.errors.any? %>
  ...
  <% post.errors.each do |error| %>
    <li><%= error.full_message %></li>
  <% end %>
<% end %>
```

ตอนนี้ (ก่อนเรียน Part 026 เรื่อง validation) `post.errors` จะ**ว่างเปล่าเสมอ** เพราะ Model
`Post` ยังไม่มี `validates` ใดๆ เลย (ตามที่สังเกตไว้ใน Step 273) — ส่วนนี้ของ view ถูกเตรียมไว้
ล่วงหน้าโดย scaffold แล้ว รอแค่วันที่เราเพิ่ม `validates :title, presence: true` เข้าไปใน
Model เท่านั้น มันก็จะเริ่มแสดงข้อความ error ให้ทันทีโดยไม่ต้องแก้ view แม้แต่บรรทัดเดียว — เป็น
ตัวอย่างที่ดีว่าทำไม MVC ที่แยกชั้นกันชัดเจนถึงทำให้เพิ่มฟีเจอร์ทีหลังได้ง่าย

### Routes ที่ scaffold เพิ่มให้

```ruby
# config/routes.rb (บรรทัดที่ scaffold เพิ่มเข้าไปให้อัตโนมัติ)
Rails.application.routes.draw do
  resources :posts
  # ...
end
```

ตรวจสอบด้วย `bin/rails routes -g post` (ทบทวนคำสั่งจาก Part 022 Step 214):

```
      Prefix Verb   URI Pattern               Controller#Action
       posts GET    /posts(.:format)          posts#index
             POST   /posts(.:format)          posts#create
    new_post GET    /posts/new(.:format)      posts#new
   edit_post GET    /posts/:id/edit(.:format) posts#edit
        post GET    /posts/:id(.:format)      posts#show
             PATCH  /posts/:id(.:format)      posts#update
             PUT    /posts/:id(.:format)      posts#update
             DELETE /posts/:id(.:format)      posts#destroy
```

**บรรทัดเดียวจาก Part 022 Step 213 (`resources :posts`) นี่เอง** — scaffold เพิ่มให้อัตโนมัติที่
บรรทัดบนสุดของ `Rails.application.routes.draw do ... end` เสมอ ถ้ามี route อื่นอยู่ก่อนแล้ว
(เช่น `root`) scaffold จะแทรกไว้เหนือ route เดิมโดยไม่รบกวนบรรทัดอื่น (ถ้าไม่ต้องการให้แก้
`routes.rb` ให้ ใช้ flag `--skip-routes` ที่เรียนไปแล้วกับ `generate controller` ใน Part 023
Step 222 ได้เหมือนกัน)

---

## Step 277: Test file ที่ scaffold สร้างให้ และทำไมไม่มี system test มาให้อัตโนมัติอีกต่อไป

### Model test และ Fixture

```ruby
# test/models/post_test.rb
require "test_helper"

class PostTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```

```yaml
# test/fixtures/posts.yml
# Read about fixtures at https://api.rubyonrails.org/classes/ActiveRecord/FixtureSet.html

one:
  title: MyString
  body: MyText

two:
  title: MyString
  body: MyText
```

ไฟล์เปล่าที่รอให้เราเขียนต่อ (เหมือนที่ Part 025 Step 242 เคยเห็นตอน `generate model`) —
**fixture** คือชุดข้อมูลตัวอย่างที่ Rails โหลดเข้าฐานข้อมูล `test` อัตโนมัติก่อนรัน test แต่ละครั้ง
(`MyString`/`MyText` เป็นค่า placeholder ที่ generator เดามาจากชนิดข้อมูลของ column เฉยๆ ไม่มี
ความหมายพิเศษ ควรแก้เป็นข้อมูลที่สื่อความหมายกว่านี้เอง) — จะเจาะลึกเรื่อง fixture เทียบกับ
FactoryBot ใน **Part 047**

### Controller test — ไฟล์ที่มีประโยชน์ทันทีโดยไม่ต้องแก้อะไรเลย

```ruby
# test/controllers/posts_controller_test.rb
require "test_helper"

class PostsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @post = posts(:one)
  end

  test "should get index" do
    get posts_url
    assert_response :success
  end

  test "should get new" do
    get new_post_url
    assert_response :success
  end

  test "should create post" do
    assert_difference("Post.count") do
      post posts_url, params: { post: { body: @post.body, title: @post.title } }
    end

    assert_redirected_to post_url(Post.last)
  end

  test "should show post" do
    get post_url(@post)
    assert_response :success
  end

  test "should get edit" do
    get edit_post_url(@post)
    assert_response :success
  end

  test "should update post" do
    patch post_url(@post), params: { post: { body: @post.body, title: @post.title } }
    assert_redirected_to post_url(@post)
  end

  test "should destroy post" do
    assert_difference("Post.count", -1) do
      delete post_url(@post)
    end

    assert_redirected_to posts_url
  end
end
```

ต่างจาก model test ที่ปล่อยว่างไว้ — **controller test ไฟล์นี้ใช้งานได้ทันที** (ครอบคลุมทั้ง 7
action) เพราะ scaffold รู้ล่วงหน้าอยู่แล้วว่า controller ของมันหน้าตาเป็นยังไง (`posts(:one)`
ดึงข้อมูลจาก fixture ด้านบนมาใช้) ลองรันดูจริง:

```bash
bin/rails test test/controllers/posts_controller_test.rb
```

```
Running 7 tests in a single process (parallelization threshold is 50)
Run options: --seed 12345

# Running:

.......

Finished in 0.847361s, 8.2604 runs/s, 15.3266 assertions/s.
7 runs, 13 assertions, 0 failures, 0 errors, 0 skips
```

7 test ผ่านหมดตั้งแต่แรกโดยไม่ต้องแก้อะไรเลย — เป็น**ตาข่ายนิรภัย (safety net) ฟรี**ที่ scaffold
มอบให้ทันที ถ้าใครมาแก้ controller ทีหลังแล้วพลาดจนพัง test ชุดนี้จะจับได้ทันที (เจาะลึกการเขียน
request/controller test ด้วย RSpec แบบมืออาชีพใน **Part 046**)

### ทำไมไม่มี System Test มาให้อัตโนมัติ (ต่างจากที่บางคนอาจเคยได้ยิน)

**ทดสอบจริงแล้วยืนยัน:** scaffold ของ Rails 8.1 **ไม่ได้สร้าง** `test/system/posts_test.rb`
ให้อัตโนมัติ แม้ว่าแอปที่สร้างด้วย `rails new` แบบเต็มจะมี gem `capybara` และ
`selenium-webdriver` อยู่ใน `Gemfile` (กลุ่ม `:test`) พร้อมใช้งานอยู่แล้วก็ตาม — เช็คได้จาก
`--help` ของ scaffold generator (ทบทวน Step 271):

```
TestUnit options:
      [--system-tests=SYSTEM_TESTS]  # Generate system test files (set to 'true' to enable)
```

Option นี้เป็น **opt-in** (ต้องเปิดเองด้วยมือ) ไม่ใช่ default อีกต่อไปแล้ว (ต่างจาก Rails
เวอร์ชันเก่าก่อนหน้านี้ที่เคยสร้างให้อัตโนมัติ) เหตุผลคือ **system test ช้ากว่า controller test
มาก** (ต้องเปิด browser จริงผ่าน Selenium/Capybara) ทีม Rails core จึงเปลี่ยนให้เป็นทางเลือกที่
นักพัฒนาต้องตัดสินใจเปิดเองตามความเหมาะสมของแต่ละ resource ไม่ใช่สร้างมาให้ครบทุกตัวโดยอัตโนมัติ

เปิดใช้งานได้ด้วย:

```bash
bin/rails generate scaffold Post title:string body:text --system-tests=true
```

หรือสร้างแยกทีหลังด้วย `bin/rails generate system_test Posts` ก็ได้เช่นกัน — เจาะลึกการเขียน
system test ด้วย Capybara แบบเต็มรูปแบบใน **Part 048**

---

## Step 278: Generator ย่อยที่มีประโยชน์ — `generate model`, `generate controller`, `generate migration Add...To...`

scaffold เหมาะกับตอนสร้าง resource ใหม่ทั้งชุด แต่ในงานจริงส่วนใหญ่เราต้องการแค่**บางส่วน**เท่านั้น
มาทบทวนและเจาะลึก generator ย่อยที่ scaffold เรียกใช้ข้างใน ซึ่งเรียกแยกเดี่ยวๆ ได้เหมือนกัน

### `rails generate model` — เมื่อต้องการแค่ Model + Migration (ไม่มี route/controller/view)

ใช้บ่อยที่สุดสำหรับ **model ที่ไม่มีหน้าเว็บของตัวเอง** เช่น model กลางที่ใช้เก็บข้อมูล join
table, หรือ model ที่ผู้ใช้ไม่เข้าถึงตรงๆ ผ่าน controller แต่ถูกจัดการผ่าน association ของ model
อื่นแทน:

```bash
bin/rails generate model Category name:string
```

```
      invoke  active_record
      create    db/migrate/20260926030431_create_categories.rb
      create    app/models/category.rb
      invoke    test_unit
      create      test/models/category_test.rb
      create      test/fixtures/categories.yml
```

เทียบกับ scaffold: **ไม่มีบรรทัด `resource_route`, `scaffold_controller`, `erb`, `helper`,
`jbuilder` เลย** — ตรงตามที่คาดหวัง เพราะ `generate model` ทำแค่ส่วน `active_record` ส่วนเดียว
(ทบทวนเต็มรูปแบบจาก Part 025 Step 242)

### `rails generate controller` — เมื่อต้องการหน้าเว็บที่ไม่ผูกกับ Model โดยตรง

ทบทวนจาก Part 023 Step 222 — ใช้เมื่อต้องการ controller/view ธรรมดาที่ไม่ใช่ CRUD ของ
resource ไหนเป็นพิเศษ เช่นหน้า static, หน้า dashboard, หน้า report:

```bash
bin/rails generate controller Reports index summary
```

```
      create  app/controllers/reports_controller.rb
       route  get "reports/index"
              get "reports/summary"
      invoke  erb
      create    app/views/reports
      create    app/views/reports/index.html.erb
      create    app/views/reports/summary.html.erb
      invoke  test_unit
      create    test/controllers/reports_controller_test.rb
      invoke  helper
      create    app/helpers/reports_helper.rb
      invoke      test_unit
```

```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  def index
  end

  def summary
  end
end
```

สังเกตว่า route ที่เพิ่มให้ (`get "reports/index"`, `get "reports/summary"`) เป็น route ตรงๆ
ตามชื่อ action **ไม่ใช่** `resources :reports` — เพราะ `generate controller` ไม่รู้ (และไม่ควร
สันนิษฐาน) ว่า controller นี้จะทำ CRUD แบบ RESTful หรือเปล่า ต่างจาก scaffold ที่มั่นใจได้ 100%
ว่าต้องเป็น RESTful เต็มรูปแบบ (7 action) เสมอ

### `rails generate migration Add<Field>To<Table>` — เมื่อต้องการแก้ตารางที่มีอยู่แล้ว

ทบทวนเต็มรูปแบบจาก Part 025 Step 245 — ใช้เมื่อ Model/ตารางมีอยู่แล้ว แค่ต้องการเพิ่ม/ลบ/แก้
column โดยไม่แตะ migration เดิม (**กฎเหล็ก:** ห้ามแก้ migration ที่ migrate ไปแล้วเด็ดขาด):

```bash
bin/rails generate migration AddPublishedToPosts published:boolean
```

```
      invoke  active_record
      create    db/migrate/20260926030432_add_published_to_posts.rb
```

```ruby
class AddPublishedToPosts < ActiveRecord::Migration[8.1]
  def change
    add_column :posts, :published, :boolean
  end
end
```

Rails เดาเนื้อหาให้เกือบสมบูรณ์จากแค่ **ชื่อ migration** ที่ตั้งตามรูปแบบ
`Add<Column>To<Table>` (หรือกลับกัน `Remove<Column>From<Table>`) พร้อม `column:type` ต่อท้าย
— ถ้าตั้งชื่อ migration ไม่ตรงรูปแบบนี้ (เช่นตั้งชื่อเอง) จะได้ไฟล์เปล่าที่มีแค่ `def change; end`
ให้เติมเอง

### ตารางสรุป generator ที่ใช้บ่อยที่สุดในงานจริง

| Generator | สร้างอะไร | ใช้เมื่อ |
|---|---|---|
| `generate scaffold` | ครบทุกชั้นของ resource | เริ่ม prototype/admin CRUD ใหม่ทั้งหมด |
| `generate model` | Model + Migration | เพิ่ม model ที่ไม่มีหน้าเว็บของตัวเอง (join table, model ภายใน) |
| `generate controller` | Controller + View (ไม่ RESTful) | หน้า static, dashboard, report ที่ไม่ผูก resource ใดเป็นพิเศษ |
| `generate migration` | Migration เปล่า/เดาเนื้อหาจากชื่อ | แก้ตารางที่มีอยู่แล้ว (เพิ่ม/ลบ/แก้ column, index) |
| `generate resource` | Model + Migration + route (ไม่มี controller action/view) | ต้องการ route/model แต่จะเขียน controller action เองทั้งหมด |

---

## Step 279: `rails destroy` — ยกเลิก generator, ข้อควรระวังเรื่อง migration, และการปรับแต่ง template

### `rails destroy` — ตัวย้อนกลับของ `rails generate`

ทุก generator ที่รันไปแล้วสามารถ **ยกเลิก (undo)** ได้ด้วยคำสั่ง `destroy` ที่รับ argument ชุด
เดียวกันเป๊ะกับตอน generate — เปลี่ยนแค่คำว่า `generate`/`g` เป็น `destroy`/`d`:

```bash
bin/rails destroy scaffold Post
```

```
      invoke  active_record
      remove    db/migrate/20260926030346_create_posts.rb
      remove    app/models/post.rb
      invoke    test_unit
      remove      test/models/post_test.rb
      remove      test/fixtures/posts.yml
      invoke  resource_route
       route    resources :posts
      invoke  scaffold_controller
      remove    app/controllers/posts_controller.rb
      invoke    erb
      remove      app/views/posts
      remove      app/views/posts/index.html.erb
      remove      app/views/posts/edit.html.erb
      remove      app/views/posts/show.html.erb
      remove      app/views/posts/new.html.erb
      remove      app/views/posts/_form.html.erb
      remove      app/views/posts/_post.html.erb
      invoke    resource_route
      invoke    test_unit
      remove      test/controllers/posts_controller_test.rb
      invoke    helper
      remove      app/helpers/posts_helper.rb
      invoke      test_unit
      invoke    jbuilder
      remove      app/views/posts/index.json.jbuilder
      remove      app/views/posts/show.json.jbuilder
      remove      app/views/posts/_post.json.jbuilder
```

`destroy` ลบไฟล์ทุกไฟล์ที่ `generate` เคยสร้าง (และลบบรรทัด `resources :posts` ออกจาก
`routes.rb` ให้ด้วย) — เหมือนกดปุ่ม Undo ให้กับคำสั่ง generate ทั้งชุด มีประโยชน์มากเวลาสร้าง
scaffold ผิด (เช่น สะกดชื่อ Model ผิด หรือใส่ field ผิด) แล้วอยากเริ่มใหม่แบบสะอาด แทนที่จะไล่ลบ
ไฟล์ทีละไฟล์เอง

### ข้อควรระวังสำคัญที่สุด: `destroy` ไม่ได้ rollback ฐานข้อมูลให้!

นี่คือจุดที่มือใหม่พลาดบ่อยที่สุด — ทดสอบจริงให้เห็นปัญหา: สมมติเรา migrate ไปแล้ว (มีตาราง
`posts` อยู่ในฐานข้อมูลจริง) แล้วสั่ง `destroy scaffold Post` **โดยไม่ rollback ก่อน**:

```bash
bin/rails db:migrate   # migrate ไปแล้ว ตาราง posts มีอยู่จริงในฐานข้อมูล
bin/rails destroy scaffold Post   # ลบไฟล์ migration ทิ้งไปเลย โดยไม่ได้ drop table ก่อน
```

ตรวจสอบฐานข้อมูลหลังจากนั้น:

```irb
irb(main):001> ActiveRecord::Base.connection.table_exists?("posts")
=> true
```

**ตาราง `posts` ยังอยู่ในฐานข้อมูลจริง!** เพราะ `destroy` แค่ **ลบไฟล์** เท่านั้น มันไม่รู้และ
ไม่สนใจว่าไฟล์ migration ที่มันลบไปนั้น "เคยถูกรันไปแล้วหรือยัง" — ยิ่งไปกว่านั้น ลองรันคำสั่ง
ตรวจสอบสถานะ migration ดู จะเจอปัญหาที่ชัดเจนกว่านั้นอีก:

```bash
bin/rails db:migrate:status
```

```
database: storage/development.sqlite3

 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20260926030346  ********** NO FILE **********
```

**`********** NO FILE **********`** คือข้อความเตือนที่ Rails แสดงเมื่อพบว่ามี version
migration ถูกบันทึกไว้ในตาราง `schema_migrations` ของฐานข้อมูล (แปลว่าเคย "รัน" ไปแล้ว) แต่หา
ไฟล์ migration ต้นฉบับที่ตรงกับ version นั้นไม่เจอ (เพราะเราลบทิ้งไปแล้ว) — สถานะนี้เป็นปัญหาจริง
เพราะทำให้ `db:rollback` ในอนาคตอาจสับสน (ไม่รู้จะ "ย้อนกลับ" การเปลี่ยนแปลงอะไร เพราะไม่มีไฟล์
`change`/`down` ให้อ้างอิงแล้ว)

### วิธีที่ถูกต้อง: `db:rollback` ก่อนเสมอ แล้วค่อย `destroy`

```bash
# ลำดับที่ถูกต้อง
bin/rails db:rollback           # (1) คืนโครงสร้างฐานข้อมูลกลับก่อน (ทบทวน Part 025 Step 245)
bin/rails destroy scaffold Post # (2) ค่อยลบไฟล์ทั้งหมดทีหลัง
```

ตรวจสอบอีกครั้งหลังทำตามลำดับที่ถูกต้อง:

```irb
irb(main):001> ActiveRecord::Base.connection.table_exists?("posts")
=> false
```

```bash
bin/rails db:migrate:status
```

```
database: storage/development.sqlite3

 Status   Migration ID    Migration Name
--------------------------------------------------
```

สะอาดหมดจด ไม่มีร่องรอยเหลืออยู่เลย — **กฎปฏิบัติที่ต้องจำ: `db:rollback` มาก่อนเสมอ ก่อน
`destroy` migration/model/scaffold ที่เคย migrate ไปแล้ว** ถ้าลืมและเกิดปัญหา `NO FILE` ขึ้นมา
แล้ว วิธีแก้คือสร้างไฟล์ migration เปล่าๆ ที่มีชื่อ/timestamp ตรงกับ version ที่ error กลับมาใหม่
ชั่วคราว (แค่พอให้ `db:rollback` มี `down`/`change` ให้เรียก) แล้วค่อยลบทิ้งให้ถูกลำดับ

### `rails app:template` และการปรับแต่ง scaffold template ของทีม (เบื้องต้น)

โปรเจกต์ทีมขนาดใหญ่ที่มี convention เฉพาะของตัวเอง (เช่น อยากให้ scaffold generate view ที่มี
CSS class ของ design system บริษัทติดมาด้วยทุกครั้ง โดยไม่ต้องมาแก้ทีละไฟล์หลัง generate) ทำได้
โดยการ **override template ที่ scaffold ใช้** — Rails มองหา custom template ที่โฟลเดอร์
`lib/templates/` ของแอปก่อนเสมอ ก่อนจะย้อนไปใช้ template เริ่มต้นจาก gem `railties`

ตัวอย่างจริง: copy template ต้นฉบับของ index view มาไว้ในแอป แล้วปรับแต่ง

```bash
mkdir -p lib/templates/erb/scaffold
cp $(bundle show railties)/lib/rails/generators/erb/scaffold/templates/index.html.erb.tt \
   lib/templates/erb/scaffold/index.html.erb.tt
```

เพิ่มบรรทัด comment มาตรฐานของทีมเข้าไปบนสุดของไฟล์:

```erb
<%# lib/templates/erb/scaffold/index.html.erb.tt %>
<%# ปรับแต่งโดยทีมของเรา: เพิ่ม comment มาตรฐานทุกหน้า index %>
<p style="color: green"><%= notice %></p>
<%# ... เนื้อหาเดิมที่เหลือจาก template ต้นฉบับ ... %>
```

หลังจากนั้น **scaffold generator ใดๆ ที่รันในแอปนี้ต่อจากนี้จะใช้ template ที่ปรับแต่งแล้วแทน
ของเดิมทันที** (ทดสอบจริงแล้วยืนยันผลลัพธ์นี้ — generate scaffold ตัวใหม่จะได้ comment ที่เราใส่
เข้าไปติดมาด้วยทุกครั้งโดยอัตโนมัติ) ระบบเดียวกันนี้ใช้ปรับแต่ง controller template ได้ด้วย
(อยู่ที่ `lib/templates/rails/scaffold_controller/controller.rb.tt`) และใช้กับ generator อื่น
ที่ไม่ใช่ scaffold ได้เช่นกัน (`lib/templates/active_record/model/model.rb.tt` เป็นต้น)

> **ขอบเขตของ Part นี้:** การปรับแต่ง generator template แบบเต็มรูปแบบ (เขียน custom
> generator class ของตัวเอง ไม่ใช่แค่ override template) เป็นหัวข้อที่ลึกกว่านี้มาก และไม่ค่อย
> จำเป็นสำหรับโปรเจกต์ทั่วไป — ในทางปฏิบัติ ทีมส่วนใหญ่ปรับแต่งแค่ 2–3 ไฟล์ที่ใช้บ่อยที่สุด (เช่น
> `_form.html.erb` ให้มี CSS class มาตรฐาน) เท่าที่จำเป็นจริงๆ เท่านั้น ไม่ต้องเขียน custom
> generator เต็มรูปแบบ เว้นแต่ทำงานในทีมที่มี resource คล้ายกันจำนวนมากและต้องการความสม่ำเสมอ
> ระดับสูงจริงๆ

---

## Step 280: เมื่อไหร่ควรใช้ scaffold ในงานจริง + แบบฝึกหัด

### scaffold ดีจริงตอนไหน

Scaffold ไม่ใช่ของเล่นสำหรับผู้เริ่มต้นเท่านั้น — วิศวกร Rails มืออาชีพก็ใช้มันจริงในสถานการณ์
เหล่านี้:

1. **Prototype/proof-of-concept ที่ต้องทำเร็ว** — ลูกค้าอยากเห็นหน้าตาระบบคร่าวๆ ก่อนตัดสินใจ
   scaffold ทำให้มี CRUD ที่คลิกได้จริงภายในไม่กี่นาที
2. **หน้า admin ภายในที่ไม่ต้องสวย ไม่ต้องซับซ้อน** — เช่น หน้าให้ทีม operation จัดการข้อมูล
   ตั้งค่าง่ายๆ (categories, tags, feature flags) ที่ business logic แทบไม่มีเลย ใช้ scaffold
   ตรงๆ แล้วปรับ styling นิดหน่อยก็เพียงพอในหลายกรณี
3. **จุดเริ่มต้นให้อ่าน ไม่ใช่จุดจบ** — แม้แต่ resource ที่ซับซ้อนก็ยัง scaffold ก่อนได้ แล้ว
   ค่อยลบ/แก้ทับส่วนที่ไม่ต้องการทีหลัง เร็วกว่าพิมพ์ controller/view จากศูนย์เสมอ (ทดสอบเวลา
   จริง: `rails generate scaffold` ใช้เวลาต่ำกว่า 1 วินาที เทียบกับการพิมพ์ไฟล์ทั้ง 16 ไฟล์เอง)
4. **เรียนรู้ pattern มาตรฐานของ Rails** — เหตุผลที่แท้จริงที่หลักสูตรนี้เก็บ Part นี้ไว้หลัง
   Part 022–027 (ไม่ใช่สอนตั้งแต่ต้น): scaffold code ที่ generate มาคือ **"Rails idiomatic
   code" ตัวอย่างที่ทีม core ยืนยันแล้วว่าถูกต้องตาม convention** เหมาะเป็นข้อมูลอ้างอิงเทียบกับ
   โค้ดที่เขียนเอง

### scaffold ไม่พอสำหรับงานจริงตรงไหน (ต้องพูดตรงๆ)

ในขณะเดียวกัน โปรเจกต์ production ส่วนใหญ่ **ไม่ได้ใช้โค้ดจาก scaffold ตรงๆ แบบไม่แก้เลย** —
เหตุผลหลักๆ ที่พบเจอบ่อยที่สุดในงานจริง:

| ปัญหา | รายละเอียด | แก้ด้วย Part ไหน |
|---|---|---|
| ไม่มี validation | `title` เป็น `nil` ก็ save ผ่านตามที่เห็นใน Step 273 | Part 026 |
| ไม่มี authorization | ใครก็เข้า `/posts/1/edit` แล้วแก้ของคนอื่นได้หมด | Part 043 (Pundit) |
| Controller โตเร็วเกินไป | resource จริงมักมี business logic เพิ่ม (ส่งอีเมล, คำนวณ, เรียก API ภายนอก) ที่ไม่ควรยัดใส่ controller action ตรงๆ | Part 082 (Service Object) |
| Query ไม่มีประสิทธิภาพ | `Post.all` โหลดทุกแถวไม่มี pagination — พังทันทีเมื่อข้อมูลเป็นหมื่นแถว | Part 038 (Pagination) |
| N+1 query | ถ้า index แสดง association เพิ่ม (เช่นชื่อผู้เขียนของแต่ละ post) โค้ดจาก scaffold ไม่มี `includes` ป้องกันไว้เลย | Part 034 |
| ข้อความเป็นภาษาอังกฤษล้วน | `"Post was successfully created."` ไม่ใช่ข้อความภาษาไทยที่ผู้ใช้จริงอ่าน | Part 039 (I18n) |
| Form ธรรมดาเกินไปสำหรับ UX จริง | ไม่มี rich text editor, ไม่มี image upload, ไม่มี autocomplete | Part 054, 066 |

**สรุปเป็นหลักการที่ควรจำไปใช้ตลอดสายอาชีพ:** scaffold คือ**จุดเริ่มต้นให้แก้ไข** ไม่ใช่
**ผลิตภัณฑ์สุดท้ายที่พร้อม deploy** — วิศวกรมือใหม่ที่ deploy โค้ดจาก scaffold ตรงๆ ขึ้น
production โดยไม่เพิ่ม validation/authorization/pagination เลย คือสัญญาณอันตรายที่ reviewer
มืออาชีพจะทักทันที ในทางกลับกัน วิศวกรที่ปฏิเสธใช้ scaffold เลยเพราะ "มันไม่ใช่โค้ดที่สมบูรณ์แบบ"
ก็เสียเวลาไปกับการพิมพ์ boilerplate ที่ generator ทำให้ได้ในเสี้ยววินาทีเช่นกัน — ทักษะที่สำคัญ
จริงๆ คือ **รู้ว่าจุดไหนของโค้ดที่ scaffold ให้มา "พอใช้ได้" และจุดไหนที่ "ต้องแก้ก่อน deploy
เสมอ"** ซึ่งคือสิ่งที่ Step 273–276 เพิ่งไล่วิเคราะห์ให้ดูมาทั้งหมด

---

### แบบฝึกหัด: เพิ่ม `Comment` ที่ผูกกับ `Post` ด้วย scaffold แบบ nested

### โจทย์

ต่อยอดแอป `blog_app` จาก Part นี้ ให้มีระบบแสดงความคิดเห็นใต้โพสต์:

1. Scaffold โมเดล `Comment` ที่ผูกกับ `Post` (ใช้ `post:references` เพื่อให้ได้ `belongs_to
   :post` และ foreign key มาโดยอัตโนมัติ) พร้อม field `body:text`
2. เพิ่ม `has_many :comments, dependent: :destroy` เข้าไปใน `Post` model เอง (scaffold ใส่
   ให้แค่ฝั่ง `belongs_to` เท่านั้น)
3. แก้ `config/routes.rb` ให้ `comments` เป็น **nested resource** ใต้ `posts` แบบ `shallow:
   true` (ทบทวน Part 022 Step 217) แทนที่ route แบบเรียบๆ ที่ scaffold generate ให้ตอนแรก
4. แก้ไข controller/view ที่ scaffold สร้างให้ทำงานถูกต้องกับ route แบบ nested (นี่คือ
   "one small custom behavior" ตามที่โจทย์กำหนด)
5. เพิ่มส่วนแสดงรายการ comment ในหน้า `posts#show` (ซึ่ง scaffold ไม่ได้เตรียมให้)
6. รันเซิร์ฟเวอร์แล้วทดสอบ flow ทั้งหมดจริงผ่าน browser/`curl`

### เฉลย

**1) Scaffold `Comment`**

```bash
bin/rails generate scaffold Comment post:references body:text
```

```
      invoke  active_record
      create    db/migrate/20260926030539_create_comments.rb
      create    app/models/comment.rb
      invoke    test_unit
      create      test/models/comment_test.rb
      create      test/fixtures/comments.yml
      invoke  resource_route
       route    resources :comments
      invoke  scaffold_controller
      create    app/controllers/comments_controller.rb
      invoke    erb
      create      app/views/comments
      create      app/views/comments/index.html.erb
      create      app/views/comments/edit.html.erb
      create      app/views/comments/show.html.erb
      create      app/views/comments/new.html.erb
      create      app/views/comments/_form.html.erb
      create      app/views/comments/_comment.html.erb
      invoke    resource_route
      invoke    test_unit
      create      test/controllers/comments_controller_test.rb
      invoke    helper
      create      app/helpers/comments_helper.rb
      invoke      test_unit
```

field ชนิด `references` ทำให้ migration และ Model ต่างจาก scaffold ปกติ:

```ruby
# db/migrate/..._create_comments.rb
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :post, null: false, foreign_key: true
      t.text :body

      t.timestamps
    end
  end
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

`t.references :post` สร้าง column `post_id` พร้อม index และ `foreign_key: true` ที่ผูก
constraint กับตาราง `posts` ในระดับฐานข้อมูลจริง (ทบทวนตาราง type ใน Part 025 Step 243 แถว
`:references`) — **และที่สำคัญที่สุด: `generate scaffold Comment post:references ...`
เดาได้เองว่าต้องใส่ `belongs_to :post` ให้ใน Model ทันที** เพราะเห็นชื่อ field ลงท้ายด้วย
`:references` จึงรู้ว่าเป็น association ไม่ใช่ column ธรรมดา — migrate ให้เรียบร้อยก่อน:

```bash
bin/rails db:migrate
```

**2) เพิ่ม `has_many :comments` ฝั่ง `Post` (ส่วนที่ scaffold ไม่ทำให้)**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

`dependent: :destroy` ทำให้เวลาลบ post หนึ่งอัน comment ที่ผูกกับมันถูกลบตามไปด้วยทั้งหมด
(ทบทวนเหตุผลจาก Part 025 Step 249 — ป้องกัน "ข้อมูลกำพร้า" ที่ `post_id` ชี้ไปยัง post ที่ไม่
มีอยู่แล้ว) — ทดสอบยืนยันพฤติกรรมนี้จริงในขั้นตอนสุดท้ายด้านล่าง

**3) แก้ routes ให้เป็น nested + shallow**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts do
    resources :comments, shallow: true
  end

  # ...
end
```

ตรวจสอบด้วย `bin/rails routes -g comment`:

```
          Prefix Verb   URI Pattern                            Controller#Action
   post_comments GET    /posts/:post_id/comments(.:format)     comments#index
                 POST   /posts/:post_id/comments(.:format)     comments#create
new_post_comment GET    /posts/:post_id/comments/new(.:format) comments#new
    edit_comment GET    /comments/:id/edit(.:format)           comments#edit
         comment GET    /comments/:id(.:format)                comments#show
                 PATCH  /comments/:id(.:format)                comments#update
                 PUT    /comments/:id(.:format)                comments#update
                 DELETE /comments/:id(.:format)                comments#destroy
```

ตรงตามที่ Part 022 Step 217 อธิบายไว้เป๊ะ: `index`/`new`/`create` ยังต้องมี `post_id` (เพราะ
ยังไม่รู้ comment เป็นตัวไหน หรือกำลังจะสร้างใหม่) ส่วน `show`/`edit`/`update`/`destroy` สั้นลง
เพราะ `shallow: true` ตัด `post_id` ที่ไม่จำเป็นออกแล้ว (`Comment.find(id)` หาเจอได้เลยไม่ต้อง
พึ่ง post)

**4) แก้ `CommentsController` ให้รู้จัก post ที่ตัวเองสังกัดอยู่**

โค้ดต้นฉบับที่ scaffold generate ให้ **ใช้ไม่ได้กับ route แบบ nested** ทันที เพราะไม่มี logic
อ่าน `params[:post_id]` เลย — นี่คือจุดที่ต้อง "แก้โค้ดที่ generate มา" ตามโจทย์:

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_post, only: %i[ index new create ]
  before_action :set_comment, only: %i[ show edit update destroy ]

  # GET /posts/:post_id/comments
  def index
    @comments = @post.comments
  end

  # GET /comments/1
  def show
  end

  # GET /posts/:post_id/comments/new
  def new
    @comment = @post.comments.build
  end

  # GET /comments/1/edit
  def edit
  end

  # POST /posts/:post_id/comments
  def create
    @comment = @post.comments.build(comment_params)

    if @comment.save
      redirect_to @comment.post, notice: "Comment was successfully created."
    else
      render :new, status: :unprocessable_content
    end
  end

  # PATCH/PUT /comments/1
  def update
    if @comment.update(comment_params)
      redirect_to @comment.post, notice: "Comment was successfully updated.", status: :see_other
    else
      render :edit, status: :unprocessable_content
    end
  end

  # DELETE /comments/1
  def destroy
    post = @comment.post
    @comment.destroy!
    redirect_to post, notice: "Comment was successfully destroyed.", status: :see_other
  end

  private
    # scaffold เดิมไม่มี method นี้ให้ — เพิ่มเองเพราะ comment ต้อง "เป็นของ" post เสมอ
    def set_post
      @post = Post.find(params[:post_id])
    end

    def set_comment
      @comment = Comment.find(params.expect(:id))
    end

    # ตัด :post_id ออกจากรายการที่ scaffold generate ให้ตอนแรก — post ถูกกำหนดจาก
    # nested route (params[:post_id]) เสมอ ไม่ควรให้ผู้ใช้ยัด post_id เองผ่านฟอร์ม
    def comment_params
      params.expect(comment: [ :body ])
    end
end
```

จุดที่เปลี่ยนจากต้นฉบับ (เทียบกับ Step 274) มี 4 อย่าง: (1) เพิ่ม `set_post` เป็น
`before_action` ใหม่ทั้งหมด (2) `index`/`new`/`create` ใช้ `@post.comments` แทน
`Comment.all`/`Comment.new` ตรงๆ — ทบทวนรูปแบบนี้จาก Part 022 Step 217 ที่เคยเขียน
`@post.comments.build(comment_params)` ไว้แล้ว (3) redirect ไปที่ `@comment.post` แทน
`@comment` เพราะต้องการกลับไปหน้าโพสต์ ไม่ใช่หน้า comment เดี่ยวๆ (4) ตัด `:post_id` ออกจาก
`comment_params` เพราะ post ถูกกำหนดจาก route อยู่แล้ว ไม่ควรเปิดช่องให้ผู้ใช้ยัดค่าเองผ่านฟอร์ม
(ความเสี่ยง mass assignment เดียวกับที่ Part 023 Step 224 เตือนไว้)

**5) แก้ view ของ comment ให้ตรงกับ route ใหม่**

Form partial เดิมมี field `post_id` เป็น text field ธรรมดา (เห็นได้จาก Step 276 style) ต้องตัด
ออกเพราะตอนนี้ post ถูกกำหนดจาก nested route แล้ว ไม่ต้องให้ผู้ใช้พิมพ์เอง และต้องแก้ `url:`
ของ `form_with` เองเพราะ scaffold ไม่รู้จัก nested route:

```erb
<%# app/views/comments/_form.html.erb %>
<%#
  scaffold generate ให้แค่ form_with(model: comment) เฉยๆ ซึ่งจะพยายาม POST/PATCH ไปที่
  comment_path/comments_path แบบ non-nested (เพราะไม่รู้จัก post) — ต้องระบุ url: เอง
  ให้ตรงกับ route แบบ nested + shallow ที่แก้ไว้ในข้อ 3
%>
<%= form_with model: comment,
      url: (comment.new_record? ? post_comments_path(comment.post) : comment_path(comment)) do |form| %>
  <% if comment.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(comment.errors.count, "error") %> prohibited this comment from being saved:</h2>
      <ul>
        <% comment.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :body, style: "display: block" %>
    <%= form.textarea :body %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

(ตัด field `post_id` ออกไปทั้ง `<div>` เพราะไม่จำเป็นอีกต่อไป) ปรับ `_comment.html.erb` ให้แสดง
เวลาแบบอ่านง่ายแทนตัวเลข `post_id` ดิบๆ ที่ scaffold ใส่มาให้ (ใช้ `time_ago_in_words` ที่เรียน
ไปแล้วใน Part 024 Step 237):

```erb
<%# app/views/comments/_comment.html.erb %>
<%#
  ปรับจากต้นฉบับที่ scaffold generate ให้ (ซึ่งโชว์ comment.post_id เป็นตัวเลขดิบๆ) —
  เปลี่ยนมาโชว์เวลาแบบอ่านง่ายแทน
%>
<div id="<%= dom_id comment %>">
  <div>
    <strong>แสดงความเห็นเมื่อ:</strong>
    <%= time_ago_in_words(comment.created_at) %>ที่แล้ว
  </div>

  <div>
    <strong>Body:</strong>
    <%= comment.body %>
  </div>
</div>
```

**6) เพิ่มส่วนแสดง comment ในหน้า `posts#show` (ฟีเจอร์ที่ scaffold ไม่มีให้)**

```erb
<%# app/views/posts/show.html.erb — เพิ่มท้ายไฟล์เดิมที่ scaffold generate ให้ %>
<p style="color: green"><%= notice %></p>

<%= render @post %>

<div>
  <%= link_to "Edit this post", edit_post_path(@post) %> |
  <%= link_to "Back to posts", posts_path %>

  <%= button_to "Destroy this post", @post, method: :delete %>
</div>

<%# ส่วนที่เพิ่มเอง (ไม่ได้มาจาก scaffold): แสดงความคิดเห็นทั้งหมดของโพสต์นี้ %>
<h2>ความคิดเห็น (<%= @post.comments.count %>)</h2>

<div id="comments">
  <%= render partial: "comments/comment", collection: @post.comments %>
</div>

<%= link_to "แสดงความเห็น", new_post_comment_path(@post) %>
```

ใช้ `render partial:, collection:` ข้าม controller ที่ทบทวนจาก Part 024 Step 236 พอดี — ดึง
partial `comments/_comment` (ที่ scaffold ของ `Comment` สร้างไว้ให้แล้ว) มาใช้ซ้ำจากหน้า `Post`
โดยไม่ต้องเขียน HTML แสดง comment ใหม่เองเลย

**7) ทดสอบจริงทั้ง flow**

```bash
bin/rails server
```

เปิด `http://localhost:3000/posts/new` สร้างโพสต์ใหม่ → ถูก redirect ไปหน้า
`http://localhost:3000/posts/1` (เห็นข้อความ "Post was successfully created." สีเขียว และ
ส่วน "ความคิดเห็น (0)" ที่เพิ่มเองในข้อ 6) จากนั้นกดลิงก์ "แสดงความเห็น" → กรอกข้อความ → submit
ผลลัพธ์จริงที่ทดสอบแล้ว (ตัด header ที่ไม่สำคัญออก):

```html
<h2>ความคิดเห็น (1)</h2>

<div id="comments">
  <div id="comment_1">
    <div>
      <strong>แสดงความเห็นเมื่อ:</strong>
      less than a minuteที่แล้ว
    </div>

    <div>
      <strong>Body:</strong>
      เขียนดีมากครับ
    </div>
  </div>
</div>
```

ทดสอบ shallow route ด้วย `curl` ตรง: `GET /comments/1` และ `GET /comments/1/edit` ตอบ
`200 OK` ทั้งคู่ (ไม่ต้องมี `post_id` ใน URL เลย ตรงตามที่ nested+shallow ออกแบบไว้) ทดสอบแก้ไข
ด้วย `PATCH /comments/1` แล้วเปิดหน้าโพสต์ใหม่ เห็นข้อความ comment เปลี่ยนตามจริง สุดท้ายทดสอบ
`dependent: :destroy` โดยลบ post ทั้งอัน (`DELETE /posts/1`) แล้วตรวจสอบผ่าน console:

```irb
irb(main):001> Post.count
=> 0
irb(main):002> Comment.count
=> 0
```

**Comment ที่เคยผูกกับ post ถูกลบตามไปด้วยอัตโนมัติ** ยืนยันว่า `has_many :comments,
dependent: :destroy` ที่เพิ่มเองในข้อ 2 ทำงานถูกต้องตามที่ตั้งใจ — ครบทุกขั้นตอนของโจทย์

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. Scaffold resource ใหม่ชื่อ `Category` (มีแค่ `name:string`) แล้วลองรัน
   `bin/rails destroy scaffold Category` **ก่อน** ที่จะ `db:migrate` เลย (ยังไม่เคย migrate)
   สังเกตว่า `db:migrate:status` มีปัญหา `NO FILE` เกิดขึ้นหรือไม่ เทียบกับกรณีที่ migrate ไป
   แล้วค่อย destroy ตามที่ Step 279 อธิบาย พร้อมอธิบายว่าทำไมผลลัพธ์ต่างกัน
2. เพิ่ม option `--api` ตอน generate scaffold (`bin/rails generate scaffold Tag name:string
   --api`) แล้วเปรียบเทียบไฟล์ที่ได้กับ scaffold ปกติ — controller ต่างกันตรงไหน (ใบ้: ลองเปิด
   `app/controllers/tags_controller.rb` เทียบบรรทัดต่อบรรทัด และสังเกตว่ามีการสร้าง
   `app/views/tags/` หรือไม่)
3. ลองสร้าง custom scaffold controller template ของตัวเองที่
   `lib/templates/rails/scaffold_controller/controller.rb.tt` ให้เปลี่ยนข้อความ
   `notice: "..."` ทั้งหมดเป็นภาษาไทย (เช่น "สร้างข้อมูลสำเร็จ") แล้ว scaffold resource ใหม่
   ทดสอบว่า controller ที่ได้มีข้อความภาษาไทยติดมาด้วยทันทีโดยไม่ต้องแก้ทีหลังหรือไม่

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Generator** คือระบบ code generation ของ Rails ที่ gem ต่างๆ (รวมถึง Rails core เอง) เพิ่ม
  เข้ามาได้ ดูรายการทั้งหมดในแอปด้วย `bin/rails generate` เปล่าๆ และดู option ของแต่ละตัวด้วย
  `--help`
- **`rails generate scaffold`** คือ generator ที่เรียก generator ย่อยต่อกันเป็นทอด (`active_record`
  → `resource_route` → `scaffold_controller` → `erb` → `helper` → `jbuilder`) สร้างไฟล์ครบ
  ทุกชั้นของ resource ในคำสั่งเดียว
- Migration และ Model ที่ scaffold สร้างให้**เหมือนกับที่ `generate model` สร้าง (Part 025)
  ทุกประการ** — ไม่มี validation/association ให้อัตโนมัติ ต้องเติมเอง
- Controller ที่ scaffold สร้างใช้ `before_action` + `only:`, private helper method,
  `params.expect` (strong parameters เวอร์ชันใหม่แทน `require`/`permit`), `respond_to` +
  `format.html`/`format.json` (เมื่อมี gem `jbuilder`), และ `status: :see_other` (303) ที่
  จำเป็นสำหรับให้ Turbo ทำงานถูกต้องหลัง `PATCH`/`DELETE` — ทุกอย่างสร้างต่อยอดจากแนวคิด Part 023
- View ที่ scaffold สร้างใช้ `dom_id` เตรียมจุดเกาะไว้สำหรับ Turbo Streams ในอนาคต (Part 052)
  และใช้ `form_with(model: ...)` ตัดสินใจ URL/method จาก `persisted?` ให้อัตโนมัติ — Turbo
  Drive ทำงานอยู่เบื้องหลังทั้งแอปโดยไม่ต้องเขียนโค้ดเพิ่ม แต่ scaffold ไม่ได้ห่อ Turbo Frame
  ให้อัตโนมัติ
- Test file ที่ scaffold สร้างให้ (model test เปล่า, controller test ที่ใช้งานได้ทันที) และรู้
  ว่า system test ไม่ใช่ default อีกต่อไปในเวอร์ชันปัจจุบัน ต้องเปิดเองด้วย `--system-tests=true`
- Generator ย่อยที่มีประโยชน์แยกตามสถานการณ์: `generate model` (แค่ข้อมูล), `generate
  controller` (แค่หน้าเว็บที่ไม่ RESTful), `generate migration Add...To...` (แก้ตารางเดิม)
- `rails destroy` ยกเลิก generator ได้ แต่ **ไม่ rollback ฐานข้อมูลให้** — ต้อง `db:rollback`
  ก่อนเสมอเมื่อ resource นั้น migrate ไปแล้ว ไม่งั้นจะเจอปัญหา `NO FILE` ในตาราง migration
- Custom scaffold template ทำได้ผ่าน `lib/templates/` เพื่อให้ทีมมี convention ของตัวเองติดมา
  กับทุก resource ที่ scaffold ใหม่โดยอัตโนมัติ
- **scaffold คือจุดเริ่มต้นให้แก้ไข ไม่ใช่ผลิตภัณฑ์สุดท้าย** — เหมาะกับ prototype และหน้า admin
  ง่ายๆ แต่ resource ระดับ production ส่วนใหญ่ต้องเพิ่ม validation, authorization, pagination,
  N+1 prevention, และ I18n เองเสมอ ซึ่งเป็นหัวข้อของ Part ถัดๆ ไปในหลักสูตรนี้ทั้งหมด

**ต่อไป (Part 029):** ตอนนี้เรามีหน้าเว็บที่ใช้งานได้ครบ CRUD แล้ว แต่หน้าตายังโล่งๆ ไม่มีรูปภาพ
ไม่มี CSS ที่เป็นระเบียบ — Part 029 จะพาไปเจาะลึก **Asset Pipeline** ของ Rails 8 (**Propshaft**
ที่เกริ่นไว้สั้นๆ ตั้งแต่ Part 021 Step 205) วิธีจัดการไฟล์ static (รูปภาพ, CSS, JavaScript),
การใช้ `image_tag`, `link_to` กับ asset, และวิธีที่ Propshaft ทำ fingerprint ไฟล์เพื่อจัดการ
เรื่อง cache — ก่อนที่ Part 030 จะรวบยอดทุกอย่างในเฟส 3 เป็นโปรเจกต์ Blog แบบครบวงจร
