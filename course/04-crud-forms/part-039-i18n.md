# Part 039: Internationalization (I18n) — จัดการ Locale และ Error Message ภาษาไทยแบบเต็มรูปแบบ

> **Step ครอบคลุมใน Part นี้:** Step 381–390
> **ระดับ:** กลาง (ต้องผ่าน Part 026 และ Part 032 มาก่อน โดยเฉพาะเรื่อง `validates`,
> `errors.add`, และ I18n เบื้องต้นที่แตะไว้ใน Step 320 — Part นี้จะขยายความเรื่องนั้นให้ครบ)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3 — ทั้งผ่าน `bin/rails runner` และผ่าน `bin/rails server` จริงด้วย `curl`)

Part 032 Step 320 เคยแตะเรื่อง I18n สั้นๆ ตอนย้ายข้อความ error ของ validation ไปไว้ที่
`config/locales/th.yml` แล้วทิ้งท้ายไว้ว่า "เรื่อง I18n และการจัดการ locale file แบบเต็มรูปแบบ
(การสลับภาษา, pluralization, การจัดโครงสร้างไฟล์ locale ขนาดใหญ่) จะสอนแบบละเอียดทั้ง Part ใน
Part 039" — นี่คือ Part นั้น

หลักสูตรนี้เป็นภาษาไทย และผู้เรียนจำนวนมากต้องสร้างแอปที่ต้อง**แสดงผลเป็นภาษาไทยทั้งระบบ** ไม่ใช่
แค่ error message แต่รวมถึงเมนู, ป้ายกำกับฟอร์ม, ข้อความแจ้งเตือน, รูปแบบวันที่/เวลา ไปจนถึง
การสลับภาษาให้ผู้ใช้เลือกเองได้ — I18n (**I**nternationalizatio**n**, มีตัวอักษร 18 ตัวระหว่าง I
กับ n จึงย่อแบบนี้) คือกลไกที่ Rails สร้างมาให้ตั้งแต่แกนกลางของ framework เพื่อรองรับเรื่องนี้
ทั้งหมด ไม่ใช่ gem เสริมที่ต้องติดตั้งเพิ่ม

## สารบัญของ Part นี้

- Step 381: I18n คืออะไร ทำไมอยู่ใน Rails Core, และเมธอด `I18n.t`/`t`/`I18n.l`/`l` เบื้องต้น
- Step 382: โครงสร้างไฟล์ `config/locales/*.yml` และธรรมเนียมการเขียน YAML แบบซ้อนชั้น
- Step 383: ตั้งค่า `config.i18n.default_locale = :th` และสร้าง `th.yml` ที่ใช้งานได้จริง
- Step 384: ใช้ `t("nested.key")` ใน View/Controller และ Interpolation (`%{name}`)
- Step 385: Pluralization (`one:`/`other:`) และความจริงที่ว่าภาษาไทยไม่มีพหูพจน์ทางไวยากรณ์
- Step 386: Lazy Lookup `t(".title")` — ความสะดวกที่แลกมาด้วยความเสี่ยง
- Step 387: ธรรมเนียม I18n ของ ActiveRecord — `activerecord.models`, `.attributes`, `.errors`
- Step 388: `available_locales` และกลไกสลับภาษา (URL param, subdomain, `Accept-Language`)
- Step 389: `I18n.l` จัดรูปแบบวันที่/ตัวเลขตาม locale และข้อควรระวังเรื่องปี พ.ศ.
- Step 390: จัดการ Missing Translation — `I18n.exception_handler` และ `raise_on_missing_translations`

---

## Step 381: I18n คืออะไร ทำไมอยู่ใน Rails Core, และเมธอด `I18n.t`/`t`/`I18n.l`/`l` เบื้องต้น

### ทำไม I18n ต้องอยู่ใน Rails Core ไม่ใช่ gem เสริม

ลองจินตนาการว่าแอปทุกตัวที่ DHH และทีม Rails สร้างต้องแปลเป็นหลายภาษาได้ตั้งแต่วันแรก (Rails
เองก็ต้องรองรับ error message, validation message เริ่มต้นเป็นภาษาต่างๆ) — Rails จึงออกแบบให้
**ทุกข้อความที่โปรแกรมแสดงต่อผู้ใช้เป็น "key" ที่ไปค้นหาคำแปลจริง แทนที่จะ hardcode string ตรงๆ**
ตั้งแต่ระดับ framework เอง ไม่ใช่ทางเลือกที่เพิ่มทีหลัง

เบื้องหลัง Rails ใช้ gem ชื่อ **`i18n`** (จาก [ruby-i18n](https://github.com/ruby-i18n/i18n))
ซึ่งถูกดึงเข้ามาเป็น dependency พื้นฐานของ Rails เองอยู่แล้วตั้งแต่ `rails new` (ไม่ต้อง
`gem "i18n"` เพิ่มใน `Gemfile` เอง) เห็นได้จากการที่ทุกครั้งที่เรียก `Post.new.errors.full_messages`
ที่ผ่านมาตั้งแต่ Part 026 ข้อความ `"can't be blank"` ที่เห็นนั้น **มาจากไฟล์ locale ภาษาอังกฤษที่
ติดมากับ Rails เองอยู่แล้ว** ไม่ใช่ string ที่ Rails hardcode ไว้ตรงๆ ในโค้ด validator

### `I18n.t` — เมธอดหลักในการแปลข้อความ

`I18n.t(key)` (ย่อมาจาก **translate**) รับ key เป็น string หรือ symbol แล้วคืนข้อความที่แปลไว้
ใน locale ปัจจุบัน:

```irb
irb(main):001> I18n.available_locales
=> [:en]
irb(main):002> I18n.default_locale
=> :en
irb(main):003> I18n.t("hello")
=> "Hello world"
```

`"hello"` เป็น key ตัวอย่างที่ Rails generate มาให้ในไฟล์ `config/locales/en.yml` ตั้งแต่
`rails new` (จะดูโครงสร้างไฟล์นี้ละเอียดใน Step 382)

### `t` ใน View/Controller — alias ของ `I18n.t`

ใน View และ Controller ไม่ต้องพิมพ์ `I18n.t` เต็มๆ เพราะ Rails มี helper method ชื่อ `t`
(alias ของ `I18n.t` ที่ผูกกับ View context) ให้เรียกสั้นๆ ได้เลย:

```erb
<%# app/views/anywhere.html.erb %>
<p><%= t("hello") %></p>
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ **ต่างกันแค่บริบทที่เรียก**: `I18n.t` ใช้ได้ทุกที่ (Model,
`bin/rails runner`, Rake task) ส่วน `t` เป็น convenience method ที่ผูกกับ View/Controller เท่านั้น
และมีความสามารถพิเศษเพิ่มเติมที่ `I18n.t` ธรรมดาไม่มี (Lazy Lookup ที่จะเรียนใน Step 386)

### `I18n.l` — จัดรูปแบบวันที่/เวลาตาม locale

`I18n.l(object)` (ย่อมาจาก **localize**) ต่างจาก `t` ตรงที่ไม่ได้แปล string ธรรมดา แต่รับ
`Date`/`Time`/`DateTime`/`ActiveSupport::TimeWithZone` แล้ว**จัดรูปแบบการแสดงผลตาม locale
ปัจจุบัน**:

```irb
irb(main):001> I18n.l(Date.new(2026, 9, 26))
=> "2026-09-26"
irb(main):002> I18n.l(Time.new(2026, 9, 26, 14, 30, 0))
=> "Sat, 26 Sep 2026 14:30:00 +0000"
```

ผลลัพธ์ตอนนี้ยังเป็นรูปแบบ default ของ Rails (ยังไม่มี locale ภาษาไทยกำหนดรูปแบบไว้) — ใน
Step 383 เมื่อสร้าง `th.yml` แล้ว รูปแบบวันที่จะเปลี่ยนเป็น "26 กันยายน 2026" ทันทีโดยไม่ต้อง
แก้โค้ดที่เรียก `I18n.l` แม้แต่บรรทัดเดียว — นี่คือจุดแข็งของ I18n: **โค้ดเขียนครั้งเดียว
เปลี่ยนภาษา/รูปแบบได้จากไฟล์ locale ล้วนๆ**

ใน View ใช้ตัวย่อ `l` (alias ของ `I18n.l`) เหมือนกับ `t`:

```erb
<p><%= l(@post.created_at) %></p>
```

> **สรุปสั้นๆ ก่อนไป Step ถัดไป:** `t`/`I18n.t` สำหรับ**แปลข้อความ** (string ที่ hardcode
> ไม่ได้เพราะต้องเปลี่ยนภาษา) ส่วน `l`/`I18n.l` สำหรับ**จัดรูปแบบวันที่/เวลา/ตัวเลข**ตาม
> ธรรมเนียมของแต่ละภาษา (บางภาษาเขียนวันที่แบบ วัน/เดือน/ปี บางภาษาเขียนแบบ เดือน/วัน/ปี)

---

## Step 382: โครงสร้างไฟล์ `config/locales/*.yml` และธรรมเนียมการเขียน YAML แบบซ้อนชั้น

### Rails โหลดไฟล์ locale ยังไง

Rails **autoload ทุกไฟล์ `.yml`/`.rb` ในโฟลเดอร์ `config/locales/`** (รวมโฟลเดอร์ย่อยด้วย ถ้า
config `I18n.load_path` ถูกขยายเพิ่ม) โดยไม่ต้อง `require` เอง — ไฟล์ที่ Rails generate มาให้ตั้งแต่
`rails new` คือ `config/locales/en.yml`:

```yaml
# config/locales/en.yml
en:
  hello: "Hello world"
```

### โครงสร้าง YAML: key บนสุดต้องเป็นชื่อ locale เสมอ

สังเกตว่าทุกอย่างอยู่ใต้ key `en:` — นี่คือกฎที่**บังคับเสมอ**: ทุกไฟล์ locale ต้องมี key
ระดับบนสุดเป็นชื่อ locale (`en`, `th`, `ja`, ฯลฯ) ครอบเนื้อหาทั้งหมดไว้ ถ้าลืมใส่ locale
ครอบไว้ Rails จะไม่รู้ว่า key ที่เหลือเป็นของภาษาไหน

### Nested Key — จัดกลุ่มด้วยการซ้อนชั้น YAML

YAML รองรับการซ้อนชั้น (nesting) แบบ Hash ธรรมดา ทำให้จัดกลุ่ม key ที่เกี่ยวข้องกันไว้ด้วยกันได้
เป็นธรรมชาติ:

```yaml
en:
  posts:
    index:
      title: "All Posts"
    new:
      title: "New Post"
```

เข้าถึง nested key ด้วย **dot notation** (จุดคั่นแต่ละระดับ):

```irb
irb(main):001> I18n.t("posts.index.title")
=> "All Posts"
```

หรือส่ง `scope:` แยกออกมาก็ได้ (มีประโยชน์เมื่อ key เดียวกันถูกใช้ซ้ำในหลาย scope):

```irb
irb(main):002> I18n.t("title", scope: "posts.index")
=> "All Posts"
irb(main):003> I18n.t(:title, scope: [:posts, :index])
=> "All Posts"
```

### ธรรมเนียมการจัดโครงสร้างไฟล์ในโปรเจกต์จริง

ไม่มีกฎบังคับว่าต้องมีไฟล์เดียวต่อ locale เสมอไป — Rails โหลดทุกไฟล์ `.yml` ใน
`config/locales/` มารวมกัน (ถ้า key ชนกันไฟล์ที่โหลดทีหลังจะทับไฟล์ก่อนหน้า ขึ้นกับลำดับ
alphabetical ของชื่อไฟล์) โปรเจกต์ขนาดใหญ่มักแยกไฟล์ตามโดเมน/ฟีเจอร์เพื่อไม่ให้ไฟล์เดียวยาวเกินไป
เช่น:

```
config/locales/
  th.yml               # ข้อความทั่วไปของแอป (nav, layout, flash)
  th.posts.yml         # ข้อความเฉพาะฟีเจอร์ posts
  th.activerecord.yml  # ชื่อ Model/attribute/error message ทั้งหมด
  en.yml
```

ทุกไฟล์ในกลุ่มนี้ยังต้องขึ้นต้นด้วย `th:` เหมือนเดิม (Rails จะ merge เนื้อหาใต้ `th:` จากทุกไฟล์
เข้าด้วยกันเป็น Hash เดียว) — Part นี้เพื่อความง่ายในการติดตาม จะรวมทุกอย่างไว้ในไฟล์
`config/locales/th.yml` ไฟล์เดียวก่อน แล้วค่อยแยกไฟล์เมื่อแอปโตขึ้นในบทเรียนหลังๆ

> **ข้อควรระวังเรื่อง YAML และค่าที่ตีความเป็น boolean:** YAML ตีความ string บางคำเป็น
> boolean/nil โดยอัตโนมัติแบบไม่สนตัวพิมพ์เล็กใหญ่ เช่น `yes`, `no`, `true`, `false`, `on`,
> `off`, `null`, `~` — ถ้าต้องการให้ค่าพวกนี้เป็น **string จริงๆ** (เช่น ข้อความปุ่มที่บังเอิญ
> เป็นคำว่า "Yes") ต้องใส่เครื่องหมายคำพูดครอบไว้เสมอ: `"yes": "ใช่"` ไม่ใช่ `yes: "ใช่"`
> (แบบหลังนี้ YAML จะตีความ `yes` เป็น key ที่มีค่า boolean `true` แทน ไม่ใช่ string `"yes"`)

---

## Step 383: ตั้งค่า `config.i18n.default_locale = :th` และสร้าง `th.yml` ที่ใช้งานได้จริง

### ทำไมต้องตั้ง default locale เอง

ค่า default ของ `I18n.default_locale` คือ `:en` เสมอ ไม่ว่าแอปจะสร้างมาเพื่อผู้ใช้ภาษาอะไรก็ตาม
— สำหรับแอปที่ผู้ใช้ส่วนใหญ่เป็นคนไทยและ UI หลักเป็นภาษาไทย ควรเปลี่ยนค่านี้ตั้งแต่ต้นโปรเจกต์
เพื่อให้ไม่ต้องส่ง `locale: :th` กำกับทุกครั้งที่เรียก `t`/`l`

แก้ที่ `config/application.rb`:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.load_defaults 8.1

    # I18n
    config.i18n.available_locales = [:th, :en]
    config.i18n.default_locale = :th
  end
end
```

`available_locales` คือรายชื่อ locale ทั้งหมดที่แอปนี้**อนุญาตให้สลับไปใช้ได้** (จะใช้เต็มรูปแบบ
ใน Step 388 ตอนทำระบบสลับภาษา) ส่วน `default_locale` คือ locale ที่ใช้เมื่อไม่มีการระบุอย่างอื่น
เจาะจง

### สร้าง `config/locales/th.yml` ที่มีข้อความ UI จริง

```yaml
# config/locales/th.yml
th:
  app:
    name: "ระบบจัดการบทความ"
  nav:
    home: "หน้าแรก"
    posts: "บทความ"
    categories: "หมวดหมู่"
  posts:
    index:
      title: "รายการบทความ"
      empty: "ยังไม่มีบทความ"
    new:
      title: "เขียนบทความใหม่"
    edit:
      title: "แก้ไขบทความ"
    form:
      submit: "บันทึกบทความ"
    flash:
      created: "สร้างบทความเรียบร้อยแล้ว"
      updated: "แก้ไขบทความเรียบร้อยแล้ว"
      destroyed: "ลบบทความเรียบร้อยแล้ว"
```

รีสตาร์ท `bin/rails console`/`bin/rails runner` แล้วทดสอบ:

```irb
irb(main):001> I18n.available_locales
=> [:th, :en]
irb(main):002> I18n.default_locale
=> :th
irb(main):003> I18n.locale
=> :th
irb(main):004> I18n.t("app.name")
=> "ระบบจัดการบทความ"
irb(main):005> I18n.t("posts.index.title")
=> "รายการบทความ"
```

**สังเกต:** `I18n.locale` (locale ที่กำลังใช้งาน ณ ตอนนี้จริงๆ) แยกจาก `I18n.default_locale`
(locale ที่จะ**กลับไปใช้เมื่อไม่มีใครระบุ**) — ตอนนี้ทั้งคู่เป็น `:th` เหมือนกันเพราะยังไม่มีอะไร
เปลี่ยน `I18n.locale` เอง ความแตกต่างนี้จะสำคัญมากขึ้นตอนทำระบบสลับภาษาใน Step 388

> **ทำไมยังต้องมี `en.yml` เก็บไว้ทั้งที่ default เป็นภาษาไทย:** Rails เองมี error message,
> validation message เริ่มต้นเป็นภาษาอังกฤษฝังอยู่ใน gem `i18n`/`activerecord` อยู่แล้ว
> (ไฟล์เหล่านี้ไม่ได้อยู่ใน `config/locales/` ของโปรเจกต์เรา แต่อยู่ใน gem เอง และ Rails โหลด
> มาให้อัตโนมัติเป็น fallback สุดท้ายเสมอ) การเก็บ `en.yml` ของโปรเจกต์ไว้เผื่อวันหนึ่งต้องรองรับ
> ผู้ใช้ภาษาอังกฤษเพิ่ม (ตามที่จะฝึกใน "แบบฝึกหัดเพิ่มเติม" ท้าย Part นี้) เป็นการเตรียมพร้อมที่ดี
> แต่ไม่ใช่ข้อบังคับ — แอปที่ต้องการแค่ภาษาไทยอย่างเดียวไม่มีปัญหาอะไรถ้าจะลบ/ไม่แตะ `en.yml` เลย

---

## Step 384: ใช้ `t("nested.key")` ใน View/Controller และ Interpolation (`%{name}`)

### ใช้ใน View

```erb
<%# app/views/posts/index.html.erb %>
<h1><%= t("posts.index.title") %></h1>

<% if @posts.empty? %>
  <p><%= t("posts.index.empty") %></p>
<% end %>
```

### ใช้ใน Controller

`t`/`I18n.t` เรียกใช้ในฝั่ง Controller ได้เหมือนกัน มีประโยชน์มากตอนตั้งข้อความ `flash`:

```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.new(post_params)

  if @post.save
    redirect_to @post, notice: t("posts.flash.created")
  else
    render :new, status: :unprocessable_entity
  end
end
```

### Interpolation — แทรกค่าตัวแปรลงในข้อความที่แปลไว้

ข้อความ UI จำนวนมากต้องมีค่าที่เปลี่ยนไปตามข้อมูลจริง (ชื่อผู้ใช้, จำนวน, วันที่) — I18n รองรับ
**interpolation** ด้วย syntax `%{key}` ในไฟล์ locale แล้วส่งค่าจริงเข้าไปตอนเรียก `t`:

```yaml
# config/locales/th.yml
th:
  greeting: "สวัสดี, %{name}!"
```

```irb
irb(main):001> I18n.t("greeting", name: "สมชาย")
=> "สวัสดี, สมชาย!"
```

ใช้ในหน้า show ของบทความ (แสดงชื่อผู้เขียน):

```yaml
th:
  posts:
    show:
      by_author: "เขียนโดย %{author}"
```

```erb
<%# app/views/posts/show.html.erb %>
<p><%= t("posts.show.by_author", author: @post.email.presence || "ไม่ระบุผู้เขียน") %></p>
```

ทดสอบผ่านหน้าเว็บจริง (`curl` ไปยัง `bin/rails server` ที่รันอยู่):

```bash
curl -s http://127.0.0.1:3000/posts/1 | grep by_author -A1
# <p>เขียนโดย ไม่ระบุผู้เขียน</p>
```

**ข้อควรระวังสำคัญ:** ถ้าไฟล์ locale ระบุ `%{name}` ไว้ แต่ตอนเรียก `t` **ลืมส่งค่า** `name:`
เข้าไป Rails จะ raise `I18n::MissingInterpolationArgument` ทันที (ไม่ใช่แสดงข้อความเปล่าๆ
เงียบๆ) — เป็นพฤติกรรมที่ตั้งใจให้ error ชัดเจนตั้งแต่ development แทนที่จะปล่อยให้ผู้ใช้เห็น
UI ที่ดูแปลกๆ ใน production:

```irb
irb(main):002> I18n.t("greeting")
Traceback (most recent call last):
I18n::MissingInterpolationArgument (
  Missing interpolation argument :name in "สวัสดี, %{name}!" (I18n::MissingInterpolationArgument)
)
```

---

## Step 385: Pluralization (`one:`/`other:`) และความจริงที่ว่าภาษาไทยไม่มีพหูพจน์ทางไวยากรณ์

### กลไก pluralization ของ I18n

ภาษาอังกฤษมีรูปเอกพจน์/พหูพจน์ต่างกัน ("1 item" vs "2 items") — I18n จัดการเรื่องนี้ด้วยการให้
key เดียวมีลูกหลายตัวตาม **CLDR pluralization category** (`zero`, `one`, `two`, `few`, `many`,
`other` — แต่ละภาษาใช้ category ไม่เท่ากัน ภาษาอังกฤษใช้แค่ `one`/`other`) แล้วส่ง `count:`
เข้าไปตอนเรียก `t` เพื่อให้ I18n เลือก category ที่ตรงกับตัวเลขนั้นให้อัตโนมัติ:

```yaml
# config/locales/en.yml
en:
  items:
    zero: "No items"
    one: "1 item"
    other: "%{count} items"
```

```irb
irb(main):001> I18n.t("items", count: 0, locale: :en)
=> "No items"
irb(main):002> I18n.t("items", count: 1, locale: :en)
=> "1 item"
irb(main):003> I18n.t("items", count: 5, locale: :en)
=> "5 items"
```

สังเกตว่า **ไม่ต้องเขียน logic if/else เลือกคำเอง** — แค่ส่ง `count:` เข้าไป I18n จะเลือก
key ย่อยที่ตรงกับกฎ pluralization ของ locale นั้นให้เอง และ `%{count}` ใน string ก็ interpolate
ค่าตัวเลขให้อัตโนมัติด้วยในตัว (ไม่ต้องส่ง `count:` ซ้ำสองรอบ)

### ความจริงที่ต้องเข้าใจให้ชัด: ภาษาไทยไม่มีพหูพจน์ทางไวยากรณ์

ภาษาไทย**ไม่ผันคำตามจำนวน** ("สินค้า 1 ชิ้น" กับ "สินค้า 5 ชิ้น" ใช้คำว่า "สินค้า" เหมือนกันเป๊ะ
ไม่มีการเติม -s หรือเปลี่ยนรูปคำแบบภาษาอังกฤษ) ดังนั้นในทางเทคนิค **locale `th` ต้องการแค่ key
เดียวคือ `other:`** ก็เพียงพอสำหรับทุกจำนวนแล้ว:

```yaml
# config/locales/th.yml
th:
  items:
    zero: "ไม่มีสินค้า"
    one: "มีสินค้า 1 ชิ้น"
    other: "มีสินค้า %{count} ชิ้น"
```

```irb
irb(main):001> [0, 1, 2, 5].each { |n| puts I18n.t("items", count: n) }
ไม่มีสินค้า
มีสินค้า 1 ชิ้น
มีสินค้า 2 ชิ้น
มีสินค้า 5 ชิ้น
```

ในตัวอย่างข้างบนใส่ `zero:`/`one:` แยกไว้เพื่อความสละสลวยของประโยคเท่านั้น (ข้อความเช่น "มีสินค้า
0 ชิ้น" ฟังดูแปลกกว่า "ไม่มีสินค้า") **ไม่ใช่เพราะกฎไวยากรณ์บังคับ** — ถ้าขี้เกียจเขียนแยก ใส่แค่
`other: "มีสินค้า %{count} ชิ้น"` key เดียวก็ถูกต้องตามหลักภาษาไทย 100% เพราะ I18n จะ fallback
ไปใช้ `other:` เสมอเมื่อไม่มี category ที่ตรงกว่า

> **แล้วทำไม Step นี้ยังต้องสอน pluralization ถ้าภาษาไทยไม่ต้องใช้จริงจัง?** เหตุผลสำคัญคือ
> แอปภาษาไทยแทบทุกตัวยังมีจุดที่ต้องแสดงข้อความภาษาอังกฤษปนอยู่เสมอ เช่น: (1) ข้อความในอีเมล
> ที่ต้องรองรับทั้งลูกค้าไทยและต่างชาติ, (2) log/error message ภายในสำหรับทีม dev, (3) แอปที่
> วางแผนขยายไปตลาดต่างประเทศในอนาคต (อย่างที่จะฝึกใน "แบบฝึกหัดเพิ่มเติม" ท้าย Part — เพิ่ม
> locale `en` เข้าไปอีกภาษา) — เมื่อถึงจุดนั้น key ภาษาอังกฤษที่ไม่ได้แยก `one:`/`other:` ให้
> ถูกต้องจะแสดงผลแปลกๆ ทันที ("1 items" แทนที่จะเป็น "1 item") จึงควรฝึกเขียน pluralization
> ให้ถูกหลักไว้ตั้งแต่ต้น แม้ locale หลักของแอปจะเป็นภาษาไทยที่ไม่จำเป็นต้องใช้ก็ตาม

---

## Step 386: Lazy Lookup `t(".title")` — ความสะดวกที่แลกมาด้วยความเสี่ยง

### ปัญหา: พิมพ์ scope เต็มซ้ำๆ ทุกบรรทัดในไฟล์เดียวกัน

จาก Step 384 ถ้า View หนึ่งไฟล์ต้องเรียก `t` หลายครั้ง จะเห็น scope ซ้ำกันเต็มไปหมด:

```erb
<%# app/views/posts/index.html.erb %>
<h1><%= t("posts.index.title") %></h1>
<p><%= t("posts.index.empty") %></p>
```

### Lazy Lookup — ใส่แค่ `.` นำหน้า key แล้ว Rails เติม scope ให้อัตโนมัติ

`t` (เฉพาะตอนเรียกจาก View — **ไม่ใช้ได้กับ `I18n.t` ตรงๆ หรือเรียกจาก Controller/Model**)
มีความสามารถพิเศษเรียกว่า **Lazy Lookup**: ถ้า key ที่ส่งเข้ามาขึ้นต้นด้วย `.` Rails จะเติม
scope ที่มาจาก**ที่อยู่ของไฟล์ View ปัจจุบัน** นำหน้าให้อัตโนมัติ:

```erb
<%# app/views/posts/index.html.erb %>
<h1><%= t(".title") %></h1>
<p><%= t(".empty") %></p>
```

โค้ดสองบรรทัดนี้เทียบเท่ากับ `t("posts.index.title")` และ `t("posts.index.empty")` เป๊ะๆ
— Rails คำนวณ scope จาก path ของไฟล์ (`app/views/posts/index.html.erb` → ตัด `app/views/`
กับนามสกุลไฟล์ออก → ได้ `posts.index`) แล้วต่อ key ที่เหลือ (`.title` → `title`) เข้าไปเป็น
`posts.index.title`

ทดสอบยืนยันผลลัพธ์ตรงกันจริง:

```irb
irb(main):001> ActionView::Base.empty.t("posts.index.title")
NoMethodError    # (Lazy Lookup ต้องมี view context จริง เรียกจาก console ตรงๆ ไม่ได้)
```

(บรรทัดข้างบนแค่แสดงให้เห็นว่า Lazy Lookup ผูกกับ view context เท่านั้น — วิธีทดสอบจริงคือเปิด
หน้าเว็บแล้วดูผลลัพธ์ที่ render ออกมา ซึ่งตรงกับที่คาดไว้ทุกประการเมื่อรันจริงผ่าน
`bin/rails server`)

### ข้อดี: กระชับ, ย้ายไฟล์/เปลี่ยนชื่อ View ได้โดยไม่ต้องแก้ key

ถ้าเปลี่ยนชื่อ action จาก `index` เป็น `list` ในอนาคต (เช่น ย้าย view ไปโฟลเดอร์อื่น) แค่ย้าย
key ใน locale file ตามไปด้วย ไม่ต้องไล่แก้ทุกจุดที่ hardcode `"posts.index.title"` ไว้ตรงๆ ใน
โค้ด View

### ข้อเสีย: มองจาก key เดียวไม่รู้ว่า key เต็มคืออะไร

จุดอ่อนใหญ่ที่สุดคือ **อ่านโค้ดแล้วไม่รู้ทันทีว่า `t(".title")` แปลว่า key อะไรกันแน่** ต้องไป
เปิดดูว่าไฟล์นี้อยู่ path ไหนก่อนถึงจะเดา scope ได้ถูก ต่างจาก `t("posts.index.title")` ที่อ่าน
คำเดียวก็รู้ทันที — นอกจากนี้ยังมีข้อจำกัดทางเทคนิคอีก 2 ข้อ:

1. **ใช้ไม่ได้กับ partial ที่ render จากหลายที่**: ถ้า `_form.html.erb` ถูก render จากทั้ง
   `new.html.erb` และ `edit.html.erb` การเรียก `t(".title")` ใน partial จะได้ scope ที่คำนวณจาก
   **path ของ partial เอง** (`posts._form`) ไม่ใช่ scope ของหน้าที่เรียกมัน — ทำให้ต้องระวังเรื่อง
   scope ที่ partial มองเห็นเป็นพิเศษ
2. **ใช้ไม่ได้นอก View**: เรียกจาก Controller, Model, background job ไม่ได้เลย ต้องพิมพ์ key
   เต็มเสมอในบริบทเหล่านั้น

> **แนวทางที่ทีมส่วนใหญ่เลือกใช้:** ใช้ Lazy Lookup (`t(".key")`) เฉพาะข้อความที่**ผูกกับ View
> เดียวจริงๆ ไม่มีวันใช้ซ้ำที่อื่น** (เช่น หัวข้อของหน้านั้นๆ) ส่วนข้อความที่ต้องใช้ข้าม View
> (เช่น ข้อความ flash ที่ต้องเรียกจาก Controller ด้วย อย่าง `posts.flash.created` ใน Step 384)
> ให้เขียน key เต็มเสมอ เพราะยังไงก็ต้องเรียกจาก Controller ซึ่ง Lazy Lookup ใช้ไม่ได้อยู่แล้ว

---

## Step 387: ธรรมเนียม I18n ของ ActiveRecord — `activerecord.models`, `.attributes`, `.errors`

### ทบทวนจาก Part 032 Step 320 แล้วขยายให้ครบ

Step 320 แนะนำโครงสร้าง `activerecord.errors.models.<model>.attributes.<attribute>.<type>`
ไปแล้วสำหรับ error message — Step นี้จะอธิบายธรรมเนียม I18n ทั้งหมดของ ActiveRecord ให้ครบทุก
namespace ที่มี ไม่ใช่แค่ส่วน errors

### `activerecord.models.<model>` — ชื่อของ Model เอง

```yaml
th:
  activerecord:
    models:
      post: "บทความ"
      category: "หมวดหมู่"
```

```irb
irb(main):001> Post.model_name.human
=> "บทความ"
irb(main):002> Category.model_name.human
=> "หมวดหมู่"
```

`model_name.human` ถูก Rails เรียกใช้เองในหลายที่ เช่น scaffold ที่ generate มา หรือหน้า error
`ActiveRecord::RecordNotFound` บางเวอร์ชัน — การแปล key นี้ไว้ทำให้ข้อความพวกนั้นเป็นภาษาไทยไป
โดยอัตโนมัติด้วย

### `activerecord.attributes.<model>.<attribute>` — ชื่อ field

```yaml
th:
  activerecord:
    attributes:
      post:
        title: "หัวข้อ"
        body: "เนื้อหา"
        slug: "สลัก URL"
        category: "หมวดหมู่"
        email: "อีเมลผู้เขียน"
      category:
        name: "ชื่อหมวดหมู่"
```

```irb
irb(main):001> Post.human_attribute_name(:title)
=> "หัวข้อ"
irb(main):002> Post.human_attribute_name(:slug)
=> "สลัก URL"
```

`human_attribute_name` มีผลกับ**สองที่**ที่สำคัญมาก:

1. `form.label :title` ใน View (จาก `form_with`/`form.label`) จะแสดงข้อความนี้แทนชื่อ column
   แบบ Title Case ที่ Rails เดาเอง (`"Title"`)
2. `errors.full_messages` เติมชื่อ field ที่แปลไว้นี้นำหน้าข้อความ error เสมอ (ตามที่เห็นใน
   Part 032 Step 320)

### `activerecord.errors` — ทบทวนโครงสร้างจาก Step 320 แบบเต็ม

```yaml
th:
  activerecord:
    errors:
      messages:
        blank: "ต้องระบุข้อมูลนี้"
        taken: "ถูกใช้ไปแล้ว"
        invalid: "ไม่ถูกต้อง"
      models:
        post:
          attributes:
            title:
              blank: "กรุณาใส่หัวข้อของบทความ"
```

ทดสอบด้วย Model `Post` จาก Part 032 (มี `belongs_to :category`, `validates :slug, presence:
true, uniqueness: { scope: :category_id }`):

```irb
irb(main):001> tech = Category.create!(name: "เทคโนโลยี")
irb(main):002> Post.create!(title: "แนะนำ Ruby", body: "b", slug: "intro", category: tech)

irb(main):003> dup = Post.new(title: "t2", body: "b", slug: "intro", category: tech)
irb(main):004> dup.valid?
=> false
irb(main):005> dup.errors.full_messages
=> ["สลัก URL ถูกใช้ไปแล้ว"]
irb(main):006> dup.errors[:slug]
=> ["ถูกใช้ไปแล้ว"]

irb(main):007> no_title = Post.new(body: "b", slug: "s2", category: tech)
irb(main):008> no_title.valid?
=> false
irb(main):009> no_title.errors.full_messages
=> ["หัวข้อ กรุณาใส่หัวข้อของบทความ"]
irb(main):010> no_title.errors[:title]
=> ["กรุณาใส่หัวข้อของบทความ"]
```

ผลลัพธ์ตรงกับที่ Step 320 อธิบายไว้ทุกประการ: `slug` ใช้ข้อความ generic (`taken`) ที่ไม่มีชื่อ
field ในตัว ส่วน `title` ใช้ข้อความเฉพาะ (`post.title.blank`) ที่มีชื่อ field พูดซ้ำอยู่แล้ว —
ทบทวนกฎจาก Step 320: ข้อความเฉพาะแบบเต็มประโยคควรแสดงด้วย `errors[:field].first` ไม่ใช่
`full_messages` เพื่อไม่ให้ชื่อ field ซ้ำสองรอบ

> **Namespace ที่ยังไม่ได้พูดถึง: `activerecord.errors.models.<model>.<error_type>`** (ไม่มี
> `.attributes` คั่นกลาง) ใช้เมื่อต้องการ override ข้อความ error type หนึ่งให้ทั้ง Model โดยไม่
> เจาะจง attribute เดียว เช่น ถ้าอยากให้ `blank` ของทุก field ใน `Post` (ไม่ใช่ทั้งแอป) ใช้
> ข้อความเดียวกันหมด: `activerecord.errors.models.post.blank: "ข้อมูลของบทความต้องระบุให้ครบ"`
> — Rails ค้นหาตามลำดับความเจาะจงเหมือนที่ Step 320 อธิบายไว้ทุกประการ (attribute เฉพาะ → model
> ทั่วไป → ทั่วทั้งแอป → default ของ Rails)

---

## Step 388: `available_locales` และกลไกสลับภาษา (URL param, subdomain, `Accept-Language`)

### ทำไมต้องมีระบบสลับภาษา

แอปจริงจำนวนมากต้องรองรับผู้ใช้หลายกลุ่มภาษา (ลูกค้าไทยกับลูกค้าต่างชาติในระบบเดียวกัน) หรือ
อย่างน้อยก็ต้องมีทางให้นักพัฒนาสลับไปดู UI ภาษาอังกฤษได้ตอน debug — วิธีที่ใช้กันทั่วไปมี 3 แบบ
หลัก แต่ละแบบมีข้อดี/ข้อเสียต่างกัน

| วิธี | ตัวอย่าง URL | ข้อดี | ข้อเสีย |
|------|-------------|-------|---------|
| URL param | `example.com/posts?locale=en` | ทำง่ายที่สุด, แชร์ลิงก์พร้อมภาษาได้ | ต้องแนบ param ทุก URL ถ้าไม่เก็บ session |
| Subdomain | `en.example.com/posts` | SEO ดี (แยก URL ตายตัวต่อภาษา ให้ search engine index แยก) | ต้อง setup DNS/SSL รองรับ subdomain |
| `Accept-Language` header | (ไม่เห็นใน URL) | ไม่ต้องให้ผู้ใช้ทำอะไร เดาภาษาจาก browser ให้อัตโนมัติ | เดาผิดได้ (ผู้ใช้ตั้ง browser เป็นภาษาอื่นแต่อยากอ่านภาษาไทย) |

แนวทางที่ทีมพัฒนาส่วนใหญ่เลือกในทางปฏิบัติคือ**ผสมทั้งสามแบบ**: ใช้ `Accept-Language` เป็นค่าเดา
เริ่มต้น (ถ้าผู้ใช้ยังไม่เคยเลือกอะไรเลย) ใช้ URL param เป็นทางที่ผู้ใช้เลือกเปลี่ยนเองได้ชัดเจน
และจำค่าที่เลือกไว้ใน session เพื่อไม่ต้องแนบ param ซ้ำทุกหน้า

### เขียน `around_action` ที่รวมทั้ง 3 แหล่งเข้าด้วยกัน

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = locale_from_params || locale_from_session || locale_from_header || I18n.default_locale
    session[:locale] = locale if locale_from_params
    I18n.with_locale(locale, &action)
  end

  def locale_from_params
    requested = params[:locale]
    requested&.to_sym if requested.present? && I18n.available_locales.include?(requested.to_sym)
  end

  def locale_from_session
    stored = session[:locale]
    stored&.to_sym if stored.present? && I18n.available_locales.include?(stored.to_sym)
  end

  def locale_from_header
    header = request.headers["Accept-Language"]
    return nil if header.blank?

    preferred = header.scan(/[a-z]{2}/i).first&.downcase&.to_sym
    preferred if I18n.available_locales.include?(preferred)
  end
end
```

**อธิบายทีละส่วน:**

- `around_action` (ไม่ใช่ `before_action`) เพราะต้องการให้ `I18n.locale` ที่ตั้งไว้มีผลแค่
  **ระหว่างที่ request นี้กำลังประมวลผลอยู่เท่านั้น** แล้วคืนค่ากลับเป็นเดิมทันทีที่ action
  จบลง — `I18n.with_locale(locale) { ... }` ทำสิ่งนี้ให้อัตโนมัติ (คล้าย `Thread.current` แบบมี
  ขอบเขตชัดเจน) ถ้าใช้ `I18n.locale = locale` ตรงๆ ใน `before_action` แทน จะเป็นการ**เปลี่ยนค่า
  แบบ global ทิ้งไว้ข้ามไปถึง request อื่น** ในสภาพแวดล้อมที่ thread ถูกใช้ซ้ำ (ธรรมดามากใน
  Puma ที่รันหลาย request บน thread เดียวกันสลับกันไป) — ถ้าไม่ล้างค่าคืนหลัง action จบ
  request ถัดไปที่ใช้ thread เดียวกันอาจได้ locale ผิดจาก request ก่อนหน้าที่ค้างอยู่
- ลำดับความสำคัญจากมากไปน้อย: **param → session → header → default** — ผู้ใช้ที่เพิ่งกด
  เปลี่ยนภาษาชัดเจน (มี param) ต้องชนะค่าที่เคยจำไว้ใน session เสมอ
- `session[:locale] = locale if locale_from_params` เก็บค่าไว้เฉพาะตอนที่ผู้ใช้ระบุ param
  มาเองเท่านั้น (ไม่เก็บค่าที่เดาจาก header ลง session เพราะนั่นอาจไม่ใช่ภาษาที่ผู้ใช้ต้องการ
  จริงๆ)
- ทุก helper method เช็ค `I18n.available_locales.include?(...)` เสมอก่อนใช้ค่า — ป้องกันไม่ให้
  ผู้ใช้ส่ง `?locale=hack_string_ใดๆ` เข้ามาแล้วทำให้ `I18n.locale` เป็นค่าที่ไม่มีไฟล์รองรับ
  (ถ้าไม่เช็คตรงนี้ Rails จะไม่ error ทันที แต่ทุก `t` หลังจากนั้นจะเจอ key ที่ไม่มีคำแปลไปหมด)

### เพิ่มลิงก์สลับภาษาใน Layout

```erb
<%# app/views/layouts/application.html.erb %>
<nav>
  <% I18n.available_locales.each do |locale| %>
    <%= link_to locale.to_s.upcase, url_for(request.query_parameters.merge(locale: locale)) %>
  <% end %>
</nav>
```

### ทดสอบผ่าน `curl` จริง (ไม่ใช่แค่ `rails runner`)

```bash
bin/rails server -p 3000 &

curl -s "http://127.0.0.1:3000/posts" | grep -A1 "<h1>"
# <h1>รายการบทความ</h1>

curl -s "http://127.0.0.1:3000/posts?locale=en" | grep -A1 "<h1>"
# <h1><span class="translation_missing" title="translation missing: en.posts.index.title">Title</span></h1>
```

ผลลัพธ์ที่สองแสดง `translation_missing` เพราะยังไม่ได้เพิ่ม key `posts.index.title` ใน
`en.yml` เลย (สาธิตพฤติกรรมนี้ตั้งใจ — จะแก้ไขในหัวข้อ Missing Translation ที่ Step 390
และในแบบฝึกหัดเพิ่มเติมท้าย Part) ทดสอบการจำค่าใน session ต่อด้วย cookie jar:

```bash
curl -s -c /tmp/cookies.txt "http://127.0.0.1:3000/posts?locale=en" -o /dev/null
curl -s -b /tmp/cookies.txt "http://127.0.0.1:3000/posts" | grep -A1 "<h1>"
# ยังเป็น en อยู่ ทั้งที่รอบนี้ไม่มี ?locale=en ใน URL แล้ว เพราะ session จำค่าไว้ให้
```

---

## Step 389: `I18n.l` จัดรูปแบบวันที่/ตัวเลขตาม locale และข้อควรระวังเรื่องปี พ.ศ.

### กำหนดรูปแบบวันที่/เวลา/ตัวเลขของภาษาไทยใน `th.yml`

I18n มี namespace มาตรฐานสำหรับกำหนดรูปแบบวันที่ ชื่อวัน ชื่อเดือน และตัวเลข — key เหล่านี้ถูก
`I18n.l`/`number_to_currency` และ helper อื่นๆ ของ Rails อ่านโดยอัตโนมัติ:

```yaml
# config/locales/th.yml
th:
  date:
    abbr_day_names: [อา, จ, อ, พ, พฤ, ศ, ส]
    day_names:
      - วันอาทิตย์
      - วันจันทร์
      - วันอังคาร
      - วันพุธ
      - วันพฤหัสบดี
      - วันศุกร์
      - วันเสาร์
    abbr_month_names:
      [~, ม.ค., ก.พ., มี.ค., เม.ย., พ.ค., มิ.ย., ก.ค., ส.ค., ก.ย., ต.ค., พ.ย., ธ.ค.]
    month_names:
      - ~
      - มกราคม
      - กุมภาพันธ์
      - มีนาคม
      - เมษายน
      - พฤษภาคม
      - มิถุนายน
      - กรกฎาคม
      - สิงหาคม
      - กันยายน
      - ตุลาคม
      - พฤศจิกายน
      - ธันวาคม
    formats:
      default: "%d %B %Y"
      short: "%d %b %Y"
      long: "%A ที่ %d %B %Y"

  time:
    am: "ก่อนเที่ยง"
    pm: "หลังเที่ยง"
    formats:
      default: "%d %B %Y เวลา %H:%M น."
      short: "%d %b %H:%M น."
      long: "%A ที่ %d %B %Y เวลา %H:%M น."

  number:
    format:
      separator: "."
      delimiter: ","
      precision: 2
    currency:
      format:
        unit: "฿"
        format: "%u%n"
        separator: "."
        delimiter: ","
        precision: 2
```

**สิ่งที่ต้องสังเกต:** `month_names`/`abbr_month_names` ต้องมี **13 สมาชิก** (ตัวแรกเป็น `~`
คือ `nil` ใน YAML) เพราะ Ruby's `Date#month` นับเดือนตั้งแต่ 1–12 ไม่ใช่ 0–11 การใส่ `nil`
เป็นสมาชิกตัวที่ 0 ไว้ทำให้ index ตรงกับหมายเลขเดือนจริงพอดี (index 1 = มกราคม, ไม่ใช่ index 0)

### ทดสอบ `I18n.l`

```irb
irb(main):001> I18n.l(Date.new(2026, 9, 26))
=> "26 กันยายน 2026"
irb(main):002> I18n.l(Date.new(2026, 9, 26), format: :long)
=> "วันเสาร์ ที่ 26 กันยายน 2026"
irb(main):003> I18n.l(Time.new(2026, 9, 26, 14, 30, 0))
=> "26 กันยายน 2026 เวลา 14:30 น."
```

เปรียบเทียบกับ Step 381 ที่ยังไม่มี `th.yml` กำหนดรูปแบบไว้ (`"2026-09-26"`) — โค้ดที่เรียก
`I18n.l` **ไม่ได้เปลี่ยนแม้แต่ตัวอักษรเดียว** สิ่งที่เปลี่ยนคือไฟล์ locale เท่านั้น นี่คือ
ประโยชน์ที่แท้จริงของการแยกการจัดรูปแบบออกจากโค้ด

### ทดสอบ `number_to_currency` (ผ่าน view helper)

```irb
irb(main):001> include ActionView::Helpers::NumberHelper
irb(main):002> number_to_currency(1500.5)
=> "฿1,500.50"
irb(main):003> number_with_delimiter(1234567)
=> "1,234,567"
```

### ข้อควรระวังที่ต้องพูดตรงๆ: `I18n.l` ไม่แปลงเป็นปี พ.ศ. ให้อัตโนมัติ

นี่คือกับดักที่ทีมพัฒนาไทยจำนวนมากเข้าใจผิด: **`I18n.l` จัดการแค่ "รูปแบบการแสดงผล" (ชื่อเดือน
เป็นภาษาไทย, ลำดับวัน/เดือน/ปี) แต่ไม่แปลง "ตัวเลขปี" จากค.ศ. เป็นพ.ศ. ให้เองเลย** แม้จะตั้ง
locale เป็น `:th` แล้วก็ตาม:

```irb
irb(main):001> I18n.l(Date.new(2026, 9, 26))
=> "26 กันยายน 2026"    # ยังเป็น ค.ศ. 2026 ไม่ใช่ พ.ศ. 2569
```

เหตุผลเชิงเทคนิคคือ `I18n.l` ใช้กลไกเดียวกับ `strftime` ของ Ruby ล้วนๆ (`%Y` คือปีค.ศ.ตรงๆ จาก
object `Date`/`Time` เอง) ไม่มี placeholder หรือ config ใดในระบบ I18n มาตรฐานที่บวกเลข 543
ให้อัตโนมัติ — เรื่องปีพุทธศักราชเป็นเรื่องเฉพาะของปฏิทินไทยที่ standard I18n (ซึ่งออกแบบตาม
มาตรฐานสากลอย่าง CLDR) ไม่ได้รองรับให้โดยตรง

**ทางแก้ที่ทีมจริงใช้กัน:** เขียน helper method ของตัวเองที่เรียก `I18n.l` แล้วแทนที่ปีค.ศ.
ด้วยปีพ.ศ.ทีหลัง:

```ruby
# app/helpers/buddhist_era_helper.rb
module BuddhistEraHelper
  def l_be(date_or_time, format: :default)
    formatted = I18n.l(date_or_time, format: format)
    ce_year = date_or_time.year
    be_year = ce_year + 543

    formatted.sub(ce_year.to_s, be_year.to_s)
  end
end
```

```irb
irb(main):001> include BuddhistEraHelper
irb(main):002> l_be(Date.new(2026, 9, 26))
=> "26 กันยายน 2569"
irb(main):003> l_be(Date.new(2026, 9, 26), format: :long)
=> "วันเสาร์ ที่ 26 กันยายน 2569"
```

> **ทำไมไม่ทำให้ `l_be` เป็นค่า default ของทั้งแอปไปเลย:** เจตนาที่ต้องแยก method ต่างหาก
> (ไม่ overrides `l`/`I18n.l` ตรงๆ) คือ **ระบบหลังบ้าน (ฐานข้อมูล, log, API, การเปรียบเทียบ
> วันที่ทางคณิตศาสตร์) ควรใช้ปีค.ศ. เสมอ** เพราะเป็นมาตรฐานสากลที่ library ส่วนใหญ่ (รวมถึง
> `Date`/`Time` ของ Ruby เอง) ยึดตาม — ปีพ.ศ.ควรเป็นแค่ **ชั้นการแสดงผล (presentation layer)**
> สุดท้ายที่ผู้ใช้เห็นบนหน้าจอเท่านั้น การแยก `l_be` ออกมาชัดเจนทำให้เห็นตรงจุดที่โค้ดเรียกใช้ว่า
> "จุดนี้ตั้งใจแปลงเป็นพ.ศ.เพื่อแสดงผล" ต่างจาก `l` ธรรมดาที่ยังคงเป็นค.ศ. — ถ้าทีมต้องการ
> ใช้ปีพ.ศ.เป็นมาตรฐานทั้งแอปจริงจัง (เช่น ระบบราชการที่บังคับใช้พ.ศ.) ควรพิจารณา gem เฉพาะทาง
> อย่าง `thai_date_time` หรือเขียน custom `Calendar` class ที่ครอบคลุมกว่านี้ ซึ่งเกินขอบเขตของ
> Part นี้

---

## Step 390: จัดการ Missing Translation — `I18n.exception_handler` และ `raise_on_missing_translations`

### พฤติกรรม default เมื่อหา key ไม่เจอ

ที่เห็นใน Step 388 ตอนสลับไป `?locale=en` แล้วเจอ `translation_missing` คือพฤติกรรม default
ของ Rails: **ไม่ raise exception แต่คืนข้อความ placeholder ที่มองเห็นได้ชัดใน HTML แทน**

```irb
irb(main):001> I18n.t("this.key.does.not.exist")
=> "Translation missing: th.this.key.does.not.exist"
```

ใน View ข้อความนี้จะถูกห่อด้วย `<span class="translation_missing" title="...">` ทำให้เห็น
ชัดเจนตอน inspect element แต่**ถ้าไม่มีใครสังเกต หน้าเว็บจริงจะหลุดออกไป production พร้อมข้อความ
ภาษาอังกฤษกึ่งเทคนิคแบบนี้ให้ผู้ใช้เห็นตรงๆ** ซึ่งไม่ควรเกิดขึ้น

### บังคับให้ raise ทันทีด้วย `raise: true`

เรียกแบบเจาะจงจุดเดียวได้ด้วย option `raise: true`:

```irb
irb(main):001> I18n.t("this.key.does.not.exist", raise: true)
Traceback (most recent call last):
I18n::MissingTranslationData (Translation missing: th.this.key.does.not.exist)
```

### `config.i18n.raise_on_missing_translations` — เปิดพฤติกรรมนี้ทั้งแอปในบาง environment

Rails มี config สำเร็จรูปที่ generate มาให้ (แต่ comment ปิดไว้เป็น default) ทั้งใน
`config/environments/test.rb` และ `development.rb`:

```ruby
# config/environments/test.rb
Rails.application.configure do
  # ...
  config.i18n.raise_on_missing_translations = true
end
```

เมื่อเปิดค่านี้ **ทุกครั้งที่ helper `t`/`l` ใน View หา key ไม่เจอ จะ raise
`I18n::MissingTranslationData` ทันที** แทนที่จะคืน placeholder เงียบๆ — เหมาะมากกับ**test
environment** เพราะทำให้ test suite **fail ทันทีถ้ามีใคร merge โค้ดที่ลืมเพิ่ม key ใน locale
file** แทนที่จะปล่อยให้หลุดไปจนเจอตอน production (ซึ่งกว่าจะรู้ตัวอาจสายไปแล้ว)

> **ทำไมไม่เปิดใน production ด้วย:** ถ้า key หายไปจริงๆ ใน production การให้ทั้งหน้า crash
> ด้วย `I18n::MissingTranslationData` **แย่กว่า**การแสดง placeholder ที่ดูแปลกๆ แต่หน้ายังใช้
> งานได้ปกติมาก — production ควรใช้พฤติกรรม default (ไม่ raise) เสมอ ส่วน `raise_on_missing_
> translations` ควรเปิดเฉพาะ **test** (ให้ CI จับได้ก่อน deploy) และอาจเปิดใน **development**
> ด้วยถ้าทีมต้องการความเข้มงวดตั้งแต่ตอนเขียนโค้ด แต่ต้องแลกกับความน่ารำคาญที่ทุก key ที่ยังไม่
> เขียนจะทำให้หน้าเว็บใช้งานไม่ได้เลยระหว่างพัฒนา — ส่วนใหญ่ทีมจึงเปิดแค่ `test` เป็นมาตรฐาน

### `I18n.exception_handler` — ปรับพฤติกรรมแบบละเอียดกว่า on/off

ถ้าต้องการพฤติกรรมที่ละเอียดกว่าการ raise/ไม่ raise แบบ all-or-nothing (เช่น ต้องการ **log**
ทุกครั้งที่เจอ key หาย เพื่อเก็บสถิติว่าทีมแปลยังขาด key ไหนบ้าง แต่ไม่อยากให้แอป crash) เขียน
custom exception handler เองได้:

```ruby
# config/initializers/i18n.rb
I18n.exception_handler = lambda do |exception, locale, key, options|
  if exception.is_a?(I18n::MissingTranslationData)
    Rails.logger.warn("[I18n] Missing translation: #{locale}.#{key}")
  end

  # เรียก handler เดิมของ I18n ต่อ เพื่อให้พฤติกรรมที่เหลือ (คืน placeholder) ยังทำงานปกติ
  I18n::ExceptionHandler.new.call(exception, locale, key, options)
end
```

วิธีนี้ทำให้ production ยังคงแสดง placeholder ตามปกติ (ไม่ crash) แต่ทีมจะเห็น log ทุกครั้งที่
เกิดเหตุการณ์นี้ ทำให้ตามไปแก้ locale file ที่ขาดได้โดยไม่ต้องรอผู้ใช้แจ้งปัญหาเข้ามาเอง

### ทดสอบ `raise: true` ผ่าน `bin/rails runner` ให้เห็นภาพรวมทั้งหมด

```irb
irb(main):001> begin
irb(main):002>   I18n.t("this.key.does.not.exist", raise: true)
irb(main):003> rescue I18n::MissingTranslationData => e
irb(main):004>   puts "raised: #{e.class}: #{e.message}"
irb(main):005> end
raised: I18n::MissingTranslationData: Translation missing: th.this.key.does.not.exist

irb(main):006> I18n.t("this.key.does.not.exist")
=> "Translation missing: th.this.key.does.not.exist"

irb(main):007> I18n.t("this.key.does.not.exist", default: "ค่า default")
=> "ค่า default"
```

`default:` เป็นอีกทางเลือกที่มีประโยชน์มาก — ให้ค่าสำรองที่ใช้แทนได้ทันทีถ้า key หลักไม่มี
โดยไม่ต้อง raise หรือโชว์ placeholder เลย (เห็นการใช้แล้วใน Step 384 ตอนเขียน
`t(".created", default: t("posts.flash.created"))` เผื่อกรณี action บาง controller ยังไม่มี
key เฉพาะของตัวเอง)

---

## แบบฝึกหัด: แปลระบบบทความทั้งหมดเป็นภาษาไทย พร้อมระบบสลับภาษา

### โจทย์

จากโมเดล `Post`/`Category` ที่ใช้มาตลอด Part นี้ (สืบทอดมาจาก Part 032) ให้ทำสิ่งต่อไปนี้ให้
ครบ:

1. เขียน `config/locales/th.yml` ให้ `Post`/`Category` มีข้อความ error ภาษาไทยครบทุก field
   (`title`, `body`, `slug`, `category`, `email` ของ `Post`; `name` ของ `Category`) ตาม
   ธรรมเนียม `activerecord.attributes`/`activerecord.errors` ที่เรียนใน Step 387
2. เขียน View ของ `posts#index`, `#new`, `#edit`, `#show` ให้ใช้ Lazy Lookup `t(".key")`
   สำหรับข้อความเฉพาะหน้า และ key เต็มสำหรับข้อความที่ใช้ข้าม Controller/View (เช่น flash)
3. เพิ่ม `switch_locale` ใน `ApplicationController` ตามแบบ Step 388 พร้อมลิงก์สลับภาษาใน layout
4. ยืนยันด้วย `bin/rails runner` และ `curl` ต่อ `bin/rails server` จริง ว่า validation error
   แสดงเป็นภาษาไทยครบถ้วน ทั้งตอนเรียกผ่าน console และตอน submit ฟอร์มจริงผ่าน HTTP

### เฉลย

**1) `config/locales/th.yml` ฉบับสมบูรณ์**

```yaml
# config/locales/th.yml
th:
  app:
    name: "ระบบจัดการบทความ"
  nav:
    home: "หน้าแรก"
    posts: "บทความ"
  posts:
    index:
      title: "รายการบทความ"
      empty: "ยังไม่มีบทความ"
      new_post: "เขียนบทความใหม่"
    new:
      title: "เขียนบทความใหม่"
    edit:
      title: "แก้ไขบทความ"
    show:
      by_author: "เขียนโดย %{author}"
    form:
      submit: "บันทึกบทความ"
    flash:
      created: "สร้างบทความเรียบร้อยแล้ว"
      updated: "แก้ไขบทความเรียบร้อยแล้ว"
      destroyed: "ลบบทความเรียบร้อยแล้ว"

  activerecord:
    models:
      post: "บทความ"
      category: "หมวดหมู่"
    attributes:
      post:
        title: "หัวข้อ"
        body: "เนื้อหา"
        slug: "สลัก URL"
        category: "หมวดหมู่"
        email: "อีเมลผู้เขียน"
      category:
        name: "ชื่อหมวดหมู่"
    errors:
      messages:
        blank: "ต้องระบุข้อมูลนี้"
        taken: "ถูกใช้ไปแล้ว"
        invalid: "ไม่ถูกต้อง"
      models:
        post:
          attributes:
            title:
              blank: "กรุณาใส่หัวข้อของบทความ"
        category:
          attributes:
            name:
              blank: "กรุณาใส่ชื่อหมวดหมู่"

  date:
    abbr_day_names: [อา, จ, อ, พ, พฤ, ศ, ส]
    day_names: [วันอาทิตย์, วันจันทร์, วันอังคาร, วันพุธ, วันพฤหัสบดี, วันศุกร์, วันเสาร์]
    abbr_month_names:
      [~, ม.ค., ก.พ., มี.ค., เม.ย., พ.ค., มิ.ย., ก.ค., ส.ค., ก.ย., ต.ค., พ.ย., ธ.ค.]
    month_names:
      [~, มกราคม, กุมภาพันธ์, มีนาคม, เมษายน, พฤษภาคม, มิถุนายน, กรกฎาคม, สิงหาคม,
       กันยายน, ตุลาคม, พฤศจิกายน, ธันวาคม]
    formats:
      default: "%d %B %Y"
      long: "%A ที่ %d %B %Y"

  time:
    am: "ก่อนเที่ยง"
    pm: "หลังเที่ยง"
    formats:
      default: "%d %B %Y เวลา %H:%M น."
      long: "%A ที่ %d %B %Y เวลา %H:%M น."
```

**2) `app/controllers/application_controller.rb`**

```ruby
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = locale_from_params || locale_from_session || I18n.default_locale
    session[:locale] = locale if locale_from_params
    I18n.with_locale(locale, &action)
  end

  def locale_from_params
    requested = params[:locale]
    requested&.to_sym if requested.present? && I18n.available_locales.include?(requested.to_sym)
  end

  def locale_from_session
    stored = session[:locale]
    stored&.to_sym if stored.present? && I18n.available_locales.include?(stored.to_sym)
  end
end
```

**3) `app/controllers/posts_controller.rb`**

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]

  def index
    @posts = Post.includes(:category).order(created_at: :desc)
  end

  def show; end

  def new
    @post = Post.new
  end

  def create
    @post = Post.new(post_params)

    if @post.save
      redirect_to @post, notice: t("posts.flash.created")
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit; end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: t("posts.flash.updated")
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: t("posts.flash.destroyed")
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :slug, :category_id, :email)
  end
end
```

**4) View ที่ใช้ Lazy Lookup**

```erb
<%# app/views/posts/index.html.erb %>
<h1><%= t(".title") %></h1>
<%= link_to t(".new_post"), new_post_path %>

<% if @posts.empty? %>
  <p><%= t(".empty") %></p>
<% else %>
  <ul>
    <% @posts.each do |post| %>
      <li>
        <%= link_to post.title, post %>
        <small>(<%= post.category.name %> &middot; <%= l(post.created_at.to_date) %>)</small>
      </li>
    <% end %>
  </ul>
<% end %>
```

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= t(".by_author", author: @post.email.presence || "ไม่ระบุผู้เขียน") %></p>
<p><%= l(@post.created_at, format: :long) %></p>
<p><%= @post.body %></p>
<%= link_to t("posts.edit.title"), edit_post_path(@post) %>
```

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with model: post do |form| %>
  <% if post.errors.any? %>
    <div class="errors">
      <h2><%= pluralize(post.errors.count, "ข้อผิดพลาด") %> ทำให้บันทึกไม่สำเร็จ:</h2>
      <ul>
        <% post.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :category_id, Post.human_attribute_name(:category) %>
    <%= form.collection_select :category_id, Category.all, :id, :name %>
  </div>
  <div>
    <%= form.label :title, Post.human_attribute_name(:title) %>
    <%= form.text_field :title %>
  </div>
  <div>
    <%= form.label :slug, Post.human_attribute_name(:slug) %>
    <%= form.text_field :slug %>
  </div>
  <div>
    <%= form.label :body, Post.human_attribute_name(:body) %>
    <%= form.text_area :body %>
  </div>

  <%= form.submit t("posts.form.submit") %>
<% end %>
```

```erb
<%# app/views/layouts/application.html.erb (เพิ่มในส่วน <body>) %>
<nav>
  <% I18n.available_locales.each do |locale| %>
    <%= link_to locale.to_s.upcase, url_for(request.query_parameters.merge(locale: locale)) %>
  <% end %>
</nav>
<% if notice %>
  <p class="notice"><%= notice %></p>
<% end %>
```

**5) ยืนยันผลผ่าน `bin/rails runner`**

```irb
irb(main):001> tech = Category.create!(name: "เทคโนโลยี")
irb(main):002> post = Post.new(body: "b", slug: "intro", category: tech)  # ไม่มี title
irb(main):003> post.valid?
=> false
irb(main):004> post.errors.full_messages
=> ["หัวข้อ กรุณาใส่หัวข้อของบทความ"]
irb(main):005> Category.new.tap(&:valid?).errors.full_messages
=> ["ชื่อหมวดหมู่ กรุณาใส่ชื่อหมวดหมู่"]
```

**6) ยืนยันผลผ่าน `curl` กับ `bin/rails server` จริง**

```bash
bin/rails server -p 3000 &
sleep 2

# หน้า index เป็นภาษาไทยตาม default_locale
curl -s http://127.0.0.1:3000/posts | grep -oE '<h1>[^<]*</h1>'
# <h1>รายการบทความ</h1>

# ดึง CSRF token แล้ว submit ฟอร์มที่จงใจส่ง title ว่าง
TOKEN=$(curl -s -c /tmp/c.txt "http://127.0.0.1:3000/posts/new" \
  | grep -oE 'name="authenticity_token" value="[^"]*"' | sed -E 's/.*value="([^"]*)"/\1/')

curl -s -b /tmp/c.txt -X POST "http://127.0.0.1:3000/posts" \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "post[category_id]=1" \
  --data-urlencode "post[title]=" \
  --data-urlencode "post[slug]=intro" \
  --data-urlencode "post[body]=เนื้อหา" \
  | grep -A2 'class="errors"'
```

ผลลัพธ์ที่ได้จริง (ทดสอบยืนยันแล้วตอนเขียน Part นี้):

```
<div class="errors">
  <h2>2 ข้อผิดพลาด ทำให้บันทึกไม่สำเร็จ:</h2>
  <ul>
      <li>หัวข้อ กรุณาใส่หัวข้อของบทความ</li>
      <li>สลัก URL ถูกใช้ไปแล้ว</li>
  </ul>
</div>
```

**ยืนยันแล้วว่า error message ภาษาไทยไหลผ่านมาถูกต้องตลอดทาง: Model validation (ภาษาไทย) →
Controller (`render :new, status: :unprocessable_entity` โดยไม่แปล error เอง) → View (`error.
full_message` แสดงข้อความที่ Model ให้มาตรงๆ) → HTML response ที่ผู้ใช้เห็นจริงผ่าน HTTP** — ไม่มี
จุดไหนใน pipeline นี้ที่ hardcode ข้อความภาษาอังกฤษทิ้งไว้เลย

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม locale `:en` เต็มรูปแบบให้แอปนี้ (ไฟล์ `config/locales/en.yml` ที่มี key ครบทุกตัวที่
   `th.yml` มี รวมถึง `activerecord.attributes`/`activerecord.errors`) แล้วทดสอบว่าสลับไป
   `?locale=en` แล้วไม่มี `translation_missing` โผล่ที่ไหนเลยในทุกหน้า (`index`, `new`, `edit`,
   `show`) — ใบ้: เปิด `config.i18n.raise_on_missing_translations = true` ใน
   `config/environments/development.rb` ชั่วคราวระหว่างทำแบบฝึกหัดนี้ เพื่อให้เจอ key ที่ขาด
   ทันทีแทนที่จะต้องไล่ดูทีละหน้าเอง (อย่าลืมปิดกลับหลังทำเสร็จ)
2. สร้าง UI สลับภาษาที่ดีขึ้นกว่าลิงก์ธรรมดา: ใช้ dropdown (`<select>` + JavaScript ที่
   `submit` ฟอร์มทันทีที่เลือก หรือ Stimulus controller ถ้าคุ้นเคยจาก Part ในเฟส 7) และเพิ่ม
   flag/ป้ายบอกว่าภาษาไหนกำลังถูกเลือกอยู่ (`current: true` ใน `link_to` เมื่อ `I18n.locale ==
   locale`)
3. เขียน spec ง่ายๆ ด้วย `bin/rails runner` ที่ตรวจสอบว่า**ทุก key ใน `en.yml` มี key คู่กันใน
   `th.yml` เป๊ะ และกลับกัน** (ใช้ `I18n.backend.send(:translations)` ดึงโครงสร้าง Hash ทั้งหมด
   ของแต่ละ locale ออกมาเปรียบเทียบ key แบบ recursive) เพื่อจับกรณีที่มีคนเพิ่ม key ใหม่ให้
   ภาษาหนึ่งแต่ลืมเพิ่มอีกภาษาหนึ่ง — เทคนิคนี้ทีมพัฒนาจริงมักเขียนเป็น Rake task หรือ CI check
   แยกต่างหาก ไม่ใช่แค่ทดลองเล่นเฉยๆ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- I18n เป็นกลไกที่อยู่ใน **Rails Core** ตั้งแต่ต้น ไม่ใช่ gem เสริม — ทุกข้อความที่ Rails
  แสดงต่อผู้ใช้ (รวมถึง validation message เริ่มต้นที่เจอมาตั้งแต่ Part 026) ผ่านระบบนี้ทั้งหมด
- `I18n.t`/`t` แปลข้อความจาก key, `I18n.l`/`l` จัดรูปแบบวันที่/เวลาตาม locale — ทั้งคู่เขียน
  โค้ดครั้งเดียว เปลี่ยนภาษา/รูปแบบได้จากไฟล์ locale ล้วนๆ โดยไม่ต้องแก้โค้ดที่เรียกใช้
- โครงสร้างไฟล์ `config/locales/*.yml` ต้องขึ้นต้นด้วยชื่อ locale เสมอ ใช้ nested key จัดกลุ่ม
  และ dot notation/`scope:` เข้าถึง key ที่ซ้อนกันหลายชั้น
- ตั้ง `config.i18n.default_locale = :th` และ `available_locales` ให้ตรงกับภาษาที่แอปรองรับจริง
- Interpolation (`%{name}`) แทรกค่าตัวแปรลงในข้อความที่แปลไว้ และ raise
  `MissingInterpolationArgument` ทันทีถ้าลืมส่งค่าที่ประกาศไว้ในเทมเพลต
- Pluralization ด้วย `one:`/`other:`/`count:` — และรู้ว่า**ภาษาไทยไม่มีพหูพจน์ทางไวยากรณ์** จึงใช้
  แค่ `other:` เพียงพอ แต่ยังต้องเขียนให้ถูกหลักเผื่อวันที่แอปต้องรองรับภาษาอังกฤษเพิ่ม
- Lazy Lookup `t(".key")` สะดวกและกระชับ แต่ใช้ได้เฉพาะใน View เท่านั้น อ่านโค้ดแล้วไม่เห็น key
  เต็มทันที และมีพฤติกรรมพิเศษที่ต้องระวังตอนใช้กับ partial ที่ render จากหลายที่
- ธรรมเนียม I18n ของ ActiveRecord ครบทั้ง 3 namespace: `activerecord.models` (ชื่อ Model),
  `activerecord.attributes` (ชื่อ field ที่มีผลกับทั้ง form label และ error message), และ
  `activerecord.errors` (ลำดับความเจาะจงจาก attribute เฉพาะ → model ทั่วไป → ทั่วทั้งแอป)
- ระบบสลับภาษาที่ผสม URL param (ผู้ใช้เลือกเอง) + session (จำค่าไว้) ผ่าน `around_action` และ
  `I18n.with_locale` (ไม่ใช้ `I18n.locale =` ตรงๆ เพราะเสี่ยงรั่วไหลข้าม request บน thread เดียวกัน)
- `I18n.l` **ไม่แปลงปีเป็นพุทธศักราชให้อัตโนมัติ** — ต้องเขียน helper เสริมเองถ้าต้องการแสดงผล
  เป็นปีพ.ศ. โดยเก็บปีค.ศ.ไว้เป็นมาตรฐานในชั้นข้อมูล/ตรรกะเสมอ
- จัดการ Missing Translation ได้หลายระดับ: `raise: true` เฉพาะจุด,
  `config.i18n.raise_on_missing_translations = true` ทั้ง environment (แนะนำเปิดใน `test` เพื่อ
  ให้ CI จับได้ก่อน production), และ `I18n.exception_handler` แบบ custom สำหรับ log โดยไม่ crash

**ต่อไป (Part 040):** ตอนนี้เรามีเครื่องมือครบมือแล้วสำหรับเฟส 4 ทั้งเฟส — form ขั้นสูง,
validation ขั้นสูง, association ขั้นสูง, query interface, callback, concern, nested form,
pagination, และ I18n Part 040 จะเป็น**โปรเจกต์รวบยอดปิดท้ายเฟส 4**: สร้าง**ระบบจัดการร้านค้า
เล็ก** ที่มี `Product`, `Category`, `Order` ทำงานร่วมกันเป็นระบบเดียว ใช้เทคนิคที่เรียนมาทั้งเฟส
ผสมกันในโปรเจกต์จริงชิ้นเดียว รวมถึง**ทำ UI ภาษาไทยเต็มรูปแบบตั้งแต่ต้น**ตามที่เรียนใน Part นี้
ไม่ใช่แค่แปะ I18n ทีหลังเป็นของแถม — ก่อนจะไปเรียนรู้เรื่อง Authentication & Authorization ในเฟส 5
ต่อไป
