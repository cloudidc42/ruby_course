# Part 031: form_with เชิงลึก, strong parameters, nested attributes

> **Step ครอบคลุมใน Part นี้:** Step 301–310
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน **Part 030** มาก่อน — โปรเจกต์ Mini Blog เดิมคือฐานที่ Part
> นี้ต่อยอดโดยตรง)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6 + Rails 8.1.4 — ทุกตัวอย่าง
> HTML ที่ปรากฏในเอกสารนี้คือผลลัพธ์จริงที่ได้จากการรัน `bin/rails server` แล้วยิง `curl` เข้าไป
> จริง ไม่ใช่โค้ดที่เขียนคาดเดาไว้ล่วงหน้า)

นี่คือ **Part แรกของ Phase 4: CRUD, Forms, ActiveRecord ขั้นสูง** ต่อจาก Phase 3 ที่ปิดท้ายด้วย
โปรเจกต์รวบยอด Mini Blog (Part 030) ที่ตอนนั้นเราใช้ `form_with` และ strong parameters แบบ
"ผิวเผินพอให้ใช้งานได้" เท่านั้น — Part นี้จะกลับไปเจาะทุกซอกทุกมุมของทั้งสองเรื่องนั้นให้ลึกที่สุด
พร้อมเพิ่มเทคนิคใหม่ที่ Part 030 ยังไม่ได้แตะเลย: **nested attributes** ที่ทำให้สร้าง/แก้ไข Post
พร้อม Comment ที่แนบมาด้วยได้ในฟอร์มเดียว ไม่ต้องแยกหน้าเหมือนเดิม

โปรเจกต์ที่ใช้ตลอด Part นี้คือ **Mini Blog เวอร์ชันขยาย** — ยังเป็น Post/Comment/Category ตัว
เดิมจากแนวคิด Part 030 แต่เพิ่มฟิลด์ใหม่เข้าไปใน `Post` เพื่อให้มีที่ให้สาธิต form builder helper
ได้ครบทุกแบบ: `category` (ความสัมพันธ์ใหม่), `status`, `visibility`, `featured`, `published_on`,
`tags` — และเพิ่มความสามารถให้ Post สร้าง/แก้ไข Comment ที่แนบมาพร้อมกันได้ในฟอร์มเดียวผ่าน
`accepts_nested_attributes_for`

> **หมายเหตุเรื่องแผนที่หลักสูตร:** ใน **outline** (`course/00-outline.md`) Part 037 มีหัวข้อ
> "Multiple models form (nested_attributes, accepts_nested_attributes_for), fields_for" ปรากฏอยู่
> ด้วย — ไม่ใช่ความซ้ำซ้อนโดยไม่ตั้งใจ Part นี้ (031) สอน**พื้นฐานของ nested attributes กับ
> ความสัมพันธ์เดียว** (Post ↔ Comment ที่มีอยู่แล้ว) เพื่อให้ปูพื้นเรื่อง form/params ทั้งหมดให้
> แน่นก่อน ส่วน **Part 037** จะกลับมาขยายเป็นฟอร์มที่จัดการ**หลายความสัมพันธ์พร้อมกัน**และโครงสร้าง
> ข้อมูลที่ซับซ้อนกว่านี้ (เช่น Order ที่มีทั้ง LineItem และ Address ซ้อนอยู่พร้อมกัน) — เนื้อหาใน
> Part นี้คือพื้นฐานที่ Part 037 จะสร้างต่อยอด ไม่ใช่เนื้อหาเดียวกันเขียนซ้ำ

## สารบัญของ Part นี้

- Step 301: `form_with` เชิงลึก — model-backed vs url-backed, `scope:`, `local:` และความเข้าใจผิด
  เรื่อง "remote by default" ใน Rails ยุค Turbo
- Step 302: `form_with(model:)` เดา URL/HTTP verb จาก `persisted?` ได้อย่างไร + เตรียมสนามทดลอง
  (ขยายสคีมา Mini Blog เดิม)
- Step 303: Form builder helper ครบชุด — `text_field`, `text_area`, `select`,
  `collection_select`, `check_box`, `radio_button`, `date_select`
- Step 304: การแสดง error — `errors.full_messages`, `field_error_proc`, การห่อฟิลด์ผิดด้วย
  `field_with_errors`
- Step 305: Strong Parameters เชิงลึก — `require` + `permit` แบบดั้งเดิม เทียบกับ `params.expect`
  ของ Rails 8
- Step 306: Permit nested/array params — `tags: []` และ `comments_attributes: [...]`
- Step 307: `accepts_nested_attributes_for` ฝั่ง Model — `allow_destroy`, `reject_if`
- Step 308: `fields_for` — เรนเดอร์ฟอร์มของโมเดลลูกซ้อนอยู่ในฟอร์มของโมเดลแม่
- Step 309: `_destroy` และ `allow_destroy: true` — ลบ record ลูกผ่านฟอร์มของ record แม่
- Step 310: แบบฝึกหัด — ฟอร์ม Post ที่สร้าง/แก้ไข/ลบ Comment แบบซ้อนได้ในหน้าเดียว (เฉลยเต็ม
  ทดสอบด้วย `curl` จริง) + แบบฝึกหัดเพิ่มเติม + สรุป

---

## Step 301: `form_with` เชิงลึก — model-backed vs url-backed, `scope:`, `local:`

### ทบทวนสิ่งที่รู้แล้ว แล้วขยายให้ลึกขึ้น

ใน Part 024 และ Part 030 เราใช้ `form_with model: post do |f| ... end` มาตลอดโดยไม่ได้อธิบายว่า
`form_with` มีโหมดการทำงาน **2 แบบที่ต่างกันโดยพื้นฐาน**:

**1. Model-backed form** — ส่ง `model:` เข้าไป

```erb
<%= form_with model: @post do |f| %>
  <%= f.text_field :title %>
<% end %>
```

`form_with` จะ**อ่านค่าจาก `@post` เอง**ทั้งหมด: ชื่อ field (`post[title]`), ค่าเริ่มต้นในฟิลด์
(`@post.title`), URL ปลายทาง และ HTTP verb (ดูละเอียดใน Step 302) — Controller/View ไม่ต้องบอก
อะไรเพิ่มเลยนอกจาก object ตัวเดียว

**2. URL-backed form** — ส่ง `url:` เข้าไปแทน (ไม่มี `model:` หรือส่ง `model: false`)

```erb
<%= form_with url: search_posts_path, method: :get do |f| %>
  <%= f.text_field :q %>
<% end %>
```

ฟอร์มแบบนี้ **ไม่ผูกกับ ActiveRecord object ใดๆ เลย** เหมาะกับฟอร์มที่ไม่ได้ไว้บันทึกข้อมูลลง
โมเดลตรงๆ เช่น ฟอร์มค้นหา, ฟอร์ม login, หรือฟอร์มที่ส่งไป action พิเศษที่ไม่ตรงกับ resource
มาตรฐาน — สังเกตว่าไม่มี object ให้ดึงชื่อฟิลด์อัตโนมัติ ชื่อ input ที่ได้จะเป็นชื่อ attribute
เปล่าๆ ไม่มี prefix (`q` ไม่ใช่ `post[q]`)

ทดสอบจริงด้วย `ActionView::Base` (ไม่ต้องพึ่งหน้าเว็บเต็ม ก็ตรวจสอบ HTML ที่ได้ได้ทันที):

```
$ bin/rails runner '
view = ActionView::Base.with_empty_template_cache.new(ActionView::LookupContext.new([]), {}, nil)
puts view.form_with(url: "/search", method: :get) { |f| f.text_field(:q) }
'
```

```html
<form action="/search" accept-charset="UTF-8" method="get">
  <input type="text" name="q" id="q" />
</form>
```

สังเกต 2 จุด: field name เป็น `q` เฉยๆ (ไม่มี prefix) และ**ไม่มี** hidden field
`authenticity_token` เลย — เพราะ Rails รู้ว่า `GET` request ไม่มี body ที่จะโดน CSRF attack ผ่าน
form submission ได้ (CSRF token มีไว้ป้องกัน state-changing request อย่าง POST/PATCH/DELETE
เท่านั้น ทบทวนกลไก CSRF จาก **Part 023**)

### `scope:` — ตั้งชื่อ prefix ของ field เองแบบไม่ต้องมี model

ถ้าอยากได้ field name แบบมี prefix (`search[q]`) โดยไม่ต้องผูกกับ model จริง ใช้ `scope:`:

```
$ bin/rails runner '
view = ActionView::Base.with_empty_template_cache.new(ActionView::LookupContext.new([]), {}, nil)
puts view.form_with(url: "/search", scope: :search, method: :get) { |f| f.text_field(:q) }
'
```

```html
<form action="/search" accept-charset="UTF-8" method="get">
  <input type="text" name="search[q]" id="search_q" />
</form>
```

`scope: :search` ทำให้ field ชื่อ `search[q]` และ id เป็น `search_q` — ฝั่ง Controller อ่านค่า
ด้วย `params[:search][:q]` หรือ `params.expect(search: [:q])` (Step 305) `scope:` มีประโยชน์เวลา
ต้องการฟอร์มที่รวมหลาย input ไว้เป็นกลุ่มเดียวกันโดยไม่ผูกกับ ActiveRecord model ใดๆ

`scope:` ยังใช้ **override** ชื่อ prefix ของ model-backed form ได้ด้วย ถ้าอยากได้ prefix ที่ไม่
ตรงกับชื่อคลาสของ model (เช่น ฟอร์มเดียวใช้ได้กับหลาย controller ที่คาดหวัง key คนละชื่อ):

```
$ bin/rails runner '
view = ActionView::Base.with_empty_template_cache.new(ActionView::LookupContext.new([]), {}, nil)
post = Post.new
puts view.form_with(model: post, scope: :article, url: "/posts") { |f| f.text_field(:title) }
'
```

```html
<form action="/posts" accept-charset="UTF-8" method="post">
  <input type="text" name="article[title]" id="article_title" />
</form>
```

ถึง `post` จะเป็น instance ของคลาส `Post` แต่ field ที่ได้กลายเป็น `article[title]` ตาม `scope:`
ที่ระบุไว้ — ใช้กรณีนี้ไม่บ่อยนัก แต่ควรรู้จักไว้เพราะเจอในโค้ด legacy อยู่บ้าง

### `local:` — ความเข้าใจผิดที่พบบ่อยที่สุดเรื่อง `form_with`

นี่คือจุดที่บทเรียนออนไลน์จำนวนมาก (รวมถึงเอกสารบางเวอร์ชันเก่า) อธิบายผิดพลาด ขอเริ่มจาก
**ข้อเท็จจริงที่ตรวจสอบจริงแล้ว** ก่อนอธิบายที่มา

**ทดลองที่ 1: `local:` ไม่ได้ส่งผลอะไรเลยในทางปฏิบัติปัจจุบัน**

สร้างหน้าเดียวที่มี `form_with` 3 แบบ (default, `local: true`, `remote: true`) แล้วดู HTML ที่
render ออกมาจริง:

```erb
<%= form_with url: "/foo" do |f| %>            <%# default — ไม่ระบุ local:/remote: %>
  <%= f.text_field :q %>
<% end %>

<%= form_with url: "/foo", local: true do |f| %>
  <%= f.text_field :q %>
<% end %>

<%= form_with url: "/foo", remote: true do |f| %>
  <%= f.text_field :q %>
<% end %>
```

ผลลัพธ์จริงที่ทดสอบบน Rails 8.1.4 (แอปที่มี `turbo-rails` ติดตั้งครบตามค่าเริ่มต้นของ
`rails new`):

```html
<!-- default -->
<form action="/foo" accept-charset="UTF-8" method="post">...</form>
<!-- local: true -->
<form action="/foo" accept-charset="UTF-8" method="post">...</form>
<!-- remote: true -->
<form action="/foo" accept-charset="UTF-8" method="post">...</form>
```

**ทั้งสามแบบ render ออกมาเหมือนกันทุกตัวอักษร** ไม่มี `data-remote` ปรากฏเลยแม้แต่กรณี
`remote: true` — เหตุผลคือ `local:`/`remote:` ใน `form_with` เป็นกลไกที่หลงเหลือมาจากยุค
**Rails UJS** (rails-ujs, สมัย Rails ≤ 6) ที่ใช้ `data-remote="true"` ให้ jQuery-UJS ดักจับแล้วยิง
เป็น AJAX request แทน — ตั้งแต่ Rails 6.1 เป็นต้นมา ค่า config
`config.action_view.form_with_generates_remote_forms` เปลี่ยน default เป็น `false` ซึ่งเทียบเท่า
กับตั้ง `local: true` เป็นค่าเริ่มต้นอยู่แล้ว การพิมพ์ `local: true` เองซ้ำในปัจจุบันจึง
**ไม่มีผลอะไรเพิ่มเติม เพราะเป็นค่า default อยู่แล้ว** และ rails-ujs เองก็ไม่ได้ติดตั้งมาใน
Rails 8 app ใหม่ด้วยซ้ำ (ถูกแทนที่ด้วย Turbo ทั้งหมด)

**ทดลองที่ 2: ตัวที่ควบคุมพฤติกรรม "remote" ตัวจริงในยุค Rails 8 คือ Turbo Drive ไม่ใช่ `local:`**

Rails 8 app ที่สร้างด้วย `rails new` แบบปกติ (ไม่ใส่ `--minimal`) จะมี gem `turbo-rails` ติดมา
เสมอ — **Turbo Drive** (ส่วนหนึ่งของ Turbo ที่โหลดผ่าน JavaScript) จะ**ดักจับทุกฟอร์มและทุกลิงก์
ในหน้าเว็บโดยอัตโนมัติ** แล้วส่ง request ผ่าน `fetch()` แทนการโหลดหน้าใหม่ทั้งหน้า (คล้าย SPA แต่
ทำที่ระดับ browser navigation ไม่ต้องเขียน JavaScript เอง) — พฤติกรรมนี้เปิดใช้งาน**เสมอ** ไม่ว่า
form_with จะเขียน `local:` อย่างไรก็ตาม เพราะมันเป็นกลไกฝั่ง JavaScript ที่แยกจาก HTML attribute
`data-remote` เดิมโดยสิ้นเชิง

วิธีที่ถูกต้องในการ**ปิด** Turbo Drive สำหรับฟอร์มใดฟอร์มหนึ่งโดยเฉพาะคือใส่
`data: { turbo: false }` (ไม่ใช่ `local: true`):

```erb
<%= form_with url: "/foo", data: { turbo: false } do |f| %>
  <%= f.text_field :q %>
<% end %>
```

```html
<form data-turbo="false" action="/foo" accept-charset="UTF-8" method="post">...</form>
```

นี่คือ HTML จริงที่ทดสอบได้ — สังเกตแอตทริบิวต์ `data-turbo="false"` ที่ปรากฏเฉพาะกรณีนี้เท่านั้น
เมื่อ Turbo (ฝั่ง JavaScript) เห็นแอตทริบิวต์นี้ มันจะปล่อยให้ browser submit ฟอร์มแบบมาตรฐาน
(full-page reload) โดยไม่เข้าไปแทรกแซง

**สรุปให้ชัดสำหรับหลักสูตรนี้:**

| สิ่งที่ต้องการ | วิธีที่ถูกต้องในปัจจุบัน (Rails 8) |
|---|---|
| ฟอร์ม submit แบบ full-page reload มาตรฐาน (ไม่มี Turbo/AJAX เกี่ยวข้องเลย) | ไม่ต้องเขียนอะไรเลยถ้าไม่มี `turbo-rails` ในแอป (เหมือนโปรเจกต์ Mini Blog เดิมที่ใช้ `--minimal`) หรือใส่ `data: { turbo: false }` ถ้าแอปมี Turbo ติดตั้งอยู่ |
| อยากเขียน `local: true` ไว้เป็นเอกสารในโค้ด (documentation-as-code) | เขียนได้ ไม่ผิด แต่ไม่มีผลจริงในทางเทคนิคอีกต่อไป (เป็นค่า default อยู่แล้วตั้งแต่ Rails 6.1) |
| ฟอร์มแบบ AJAX จริงๆ (rails-ujs แบบเก่า) | ไม่แนะนำในโปรเจกต์ใหม่ — ใช้ Turbo Frames/Streams แทน (เรียนเต็มรูปแบบใน **Phase 7 Part 051–052**) |

โปรเจกต์ Mini Blog ที่ใช้ตลอด Part นี้ยังคงสร้างด้วย `rails new mini_blog --minimal` เหมือน
Part 030 (ไม่มี `turbo-rails`) ดังนั้นทุกฟอร์มในเอกสารนี้ยังคง submit แบบ full-page reload
มาตรฐาน 100% เราจะยังคงเขียน `local: true` ต่อไปในทุกฟอร์มของโปรเจกต์นี้**ตามธรรมเนียมที่วางไว้
ตั้งแต่ Part 030** (สื่อความตั้งใจให้คนอ่านโค้ดเข้าใจง่าย แม้จะไม่มีผลทางเทคนิคใน environment
ที่ไม่มี Turbo ก็ตาม) แต่ตอนนี้เข้าใจแล้วว่า**ทำไม**ถึงไม่มีผลกระทบทางเทคนิคจริง และรู้ว่าถ้าวัน
หนึ่งเพิ่ม Turbo เข้ามา (Phase 7) ต้องใช้ `data: { turbo: false }` แทนถ้าต้องการปิด Turbo เฉพาะจุด

---

## Step 302: `form_with(model:)` เดา URL/HTTP verb จาก `persisted?` ได้อย่างไร + เตรียมสนามทดลอง

### กลไกการเดา URL และ verb

ทบทวนสั้นๆ จาก Part 024/030: `form_with model: post` ที่ไม่ระบุ `url:` เอง จะให้ Rails เดา
ปลายทางให้อัตโนมัติ โดยอาศัย **polymorphic routing** (ทบทวนจาก **Part 022**) ร่วมกับเมธอด
`persisted?` ของ object:

- `post.persisted?` เป็น `false` (record ใหม่ ยังไม่มี `id`) → `form_with` เดาว่าต้อง `POST` ไปที่
  `posts_path`
- `post.persisted?` เป็น `true` (record มีอยู่แล้วในฐานข้อมูล มี `id`) → `form_with` เดาว่าต้อง
  `PATCH` ไปที่ `post_path(post)`

`persisted?` เป็นเมธอดของ `ActiveRecord::Base` ที่คืนค่า `true`/`false` ตามว่า record นั้นเคยถูก
บันทึก (`save`) สำเร็จแล้วหรือยัง — `Post.new` จะได้ `persisted? # => false` เสมอ ส่วน
`Post.find(1)` หรือ record ที่เพิ่ง `save` สำเร็จจะได้ `persisted? # => true`

เนื่องจากเบราว์เซอร์ยังไม่รองรับ `<form method="patch">` โดยตรง (HTML form spec รองรับแค่ `GET`/
`POST`) Rails จึงใช้เทคนิค **`_method` hidden field** (ทบทวนจาก **Part 022 Step 211**): ส่ง
`method="post"` จริงๆ ทาง HTTP แต่แนบ `<input type="hidden" name="_method" value="patch">` ไว้
ข้างใน แล้ว Rails middleware (`Rack::MethodOverride`) จะอ่านค่านี้แล้วสลับ verb ให้เป็น `PATCH`
ก่อนส่งเข้า router จริง

### พิสูจน์ด้วย HTML จริงที่ render ออกมา

**ฟอร์มของ record ใหม่ (`Post.new`, `persisted? == false`):**

```html
<form action="/posts" accept-charset="UTF-8" method="post">
  <input type="hidden" name="authenticity_token" value="..." />
  ...
</form>
```

**ฟอร์มของ record ที่มีอยู่แล้ว (`Post.find(1)`, `persisted? == true`):**

```html
<form action="/posts/1" accept-charset="UTF-8" method="post">
  <input type="hidden" name="_method" value="patch" />
  <input type="hidden" name="authenticity_token" value="..." />
  ...
</form>
```

สังเกตว่า `action` เปลี่ยนจาก `/posts` เป็น `/posts/1` และมี hidden field `_method=patch` เพิ่มขึ้น
มา — ทั้งหมดนี้เกิดจาก `form_with model: post` **บรรทัดเดียวกันเป๊ะ** ในไฟล์ `_form.html.erb`
ไม่ได้เขียนโค้ดแยกกันระหว่างหน้า `new` กับ `edit` เลย (นี่คือเหตุผลที่ partial `_form` ใช้ร่วมกัน
ได้ระหว่างสองหน้า ตามที่เรียนใน Part 024/030)

### เตรียมสนามทดลอง — ขยายสคีมา Mini Blog เดิม

ก่อนจะสาธิต form builder helper ครบทุกแบบใน Step 303 ต้องมีฟิลด์ให้สาธิตก่อน ต่อยอดจาก Mini Blog
Part 030 โดยเพิ่มโมเดล `Category` ใหม่ และเพิ่มฟิลด์ให้ `Post`:

```bash
mkdir -p ~/ruby-course-workspace/part-031
cd ~/ruby-course-workspace/part-031

rails new mini_blog --minimal
cd mini_blog

bin/rails generate model Category name:string

bin/rails generate model Post title:string body:text category:references \
  status:string visibility:string featured:boolean published_on:date tags:text

bin/rails generate model Comment post:references commenter:string body:text
```

แก้ migration ทั้งสามไฟล์ให้บังคับ `null: false`/ค่า default ที่เหมาะสม (ทบทวนแนวคิด "validate
สองชั้น" จาก **Part 026/030**):

```ruby
# db/migrate/..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title, null: false
      t.text :body, null: false
      t.references :category, null: true, foreign_key: true
      t.string :status, null: false, default: "draft"
      t.string :visibility, null: false, default: "public"
      t.boolean :featured, null: false, default: false
      t.date :published_on
      t.text :tags

      t.timestamps
    end
  end
end
```

```ruby
# db/migrate/..._create_comments.rb
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :post, null: false, foreign_key: true
      t.string :commenter, null: false
      t.text :body, null: false

      t.timestamps
    end
  end
end
```

**ทำไม `category` ถึง `null: true`** (ต่างจาก `Comment#post` ที่บังคับ `null: false`) — เพราะ
หมวดหมู่เป็นข้อมูลเสริม บทความยังมีความหมายสมบูรณ์แม้ไม่มีหมวดหมู่ ต่างจาก comment ที่ไม่มี
ความหมายเลยถ้าไม่มี post เจ้าของ (ทบทวนหลักการนี้จาก **Part 027**)

รัน migration:

```bash
bin/rails db:migrate
```

```
== CreateCategories: migrating ================================================
-- create_table(:categories)
== CreateCategories: migrated ==================================================

== CreatePosts: migrating ======================================================
-- create_table(:posts)
== CreatePosts: migrated =======================================================

== CreateComments: migrating ===================================================
-- create_table(:comments)
== CreateComments: migrated =====================================================
```

### Model — `app/models/post.rb`

```ruby
class Post < ApplicationRecord
  STATUSES = %w[draft published archived].freeze
  VISIBILITIES = %w[public private].freeze

  belongs_to :category, optional: true
  has_many :comments, dependent: :destroy

  accepts_nested_attributes_for :comments, allow_destroy: true, reject_if: :all_blank

  serialize :tags, coder: JSON, type: Array

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true, length: { minimum: 10 }
  validates :status, inclusion: { in: STATUSES }
  validates :visibility, inclusion: { in: VISIBILITIES }

  after_initialize { self.tags ||= [] }
  before_validation { self.tags = Array(tags).reject(&:blank?) }
end
```

จุดที่ยังใหม่ (`accepts_nested_attributes_for`, `serialize`) จะอธิบายละเอียดใน Step 306–307 —
ตอนนี้ขอให้สังเกตแค่โครงสร้างไฟล์ก่อน

`belongs_to :category, optional: true` — ทบทวนจาก **Part 027 Step 261 / Part 030 Step 293**:
Rails 5+ ทำให้ `belongs_to` เป็น **required by default** เสมอ ถ้าไม่ต้องการบังคับ (เช่นกรณีนี้ที่
หมวดหมู่เป็นข้อมูลเสริม) ต้องระบุ `optional: true` เองเสมอ มิฉะนั้น `Post.new(category_id: nil)`
จะ validate ไม่ผ่านทันทีโดยไม่มี error message ที่ชัดเจนว่าทำไม

`app/models/comment.rb` และ `app/models/category.rb` เหมือน Part 030 ทุกประการ (เพิ่มแค่
`has_many :posts` ใน `Category` และ `validates :name, presence: true, uniqueness: true`)

### Routes และ Controller โครงร่าง

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts do
    resources :comments, only: %i[create destroy], shallow: true
  end
end
```

เหมือน Part 030 ทุกประการ — โครงสร้าง routing ไม่มีอะไรเปลี่ยนเลยเพราะ Part นี้ไม่ได้เพิ่ม
resource ใหม่ (Category ยังไม่ต้องมี controller ของตัวเอง เพราะสร้างผ่าน `db/seeds.rb` และ
`bin/rails console` เท่านั้นในเอกสารนี้ — CRUD เต็มรูปแบบของ Category ปล่อยเป็นแบบฝึกหัด)

`PostsController`/`CommentsController` จะค่อยๆ ขยายทีละ Step ในเอกสารนี้ (`before_action
:set_post`/`set_categories` แบบเดียวกับ Part 030 เตรียม `@post`/`@categories` ให้ทุก action ที่
ต้องใช้) จนได้เวอร์ชันสมบูรณ์ครบทุก action ที่ Step 310

---

## Step 303: Form builder helper ครบชุด

Rails มี form builder helper ให้ใช้เยอะมาก แต่ที่ใช้บ่อยที่สุดในงานจริงมี 7 ตัว ครอบคลุมชนิด
input แทบทุกแบบที่ HTML form มี ต่อไปนี้คือทั้งหมด**พร้อม HTML จริงที่ render ออกมา**เมื่อทดสอบ
กับหน้า `posts#new`

### 1. `text_field` และ `text_area` (ทบทวนแล้วจาก Part 024/030)

```erb
<%= f.text_field :title %>
<%= f.text_area :body, rows: 6 %>
```

```html
<input type="text" name="post[title]" id="post_title" />
<textarea rows="6" name="post[body]" id="post_body">
</textarea>
```

### 2. `select` — ตัวเลือกแบบกำหนด array เอง (ไม่ผูกกับ ActiveRecord collection)

```erb
<%= f.select :status,
      [["ฉบับร่าง", "draft"], ["เผยแพร่แล้ว", "published"], ["เก็บถาวร", "archived"]] %>
```

```html
<select name="post[status]" id="post_status">
  <option selected="selected" value="draft">ฉบับร่าง</option>
  <option value="published">เผยแพร่แล้ว</option>
  <option value="archived">เก็บถาวร</option>
</select>
```

รูปแบบ array คือ `[[label, value], [label, value], ...]` — สังเกตว่า `<option>` ของ `"draft"`
มี `selected="selected"` ติดมาอัตโนมัติ เพราะ `post.status` มีค่า default เป็น `"draft"` จาก
migration (`default: "draft"`) `f.select` ฉลาดพอที่จะเทียบค่าปัจจุบันของ attribute กับแต่ละ
`value` ใน array แล้วใส่ `selected` ให้ตัวที่ตรงกันเองโดยอัตโนมัติ ไม่ต้องเขียนเงื่อนไขเอง

`select` เหมาะกับตัวเลือกที่**ไม่ได้มาจากตาราง**ในฐานข้อมูล (เช่น enum-like values ที่กำหนดตายตัว
ในโค้ด อย่าง `Post::STATUSES` ที่ประกาศไว้ใน Step 302)

### 3. `collection_select` — ตัวเลือกที่มาจาก ActiveRecord collection จริง

```erb
<%= f.collection_select :category_id, @categories, :id, :name,
      { include_blank: "— ไม่ระบุหมวดหมู่ —" } %>
```

```html
<select name="post[category_id]" id="post_category_id">
  <option value="">— ไม่ระบุหมวดหมู่ —</option>
  <option value="1">เทคโนโลยี</option>
  <option value="2">ไลฟ์สไตล์</option>
</select>
```

อาร์กิวเมนต์ของ `collection_select` เรียงเป็น `method, collection, value_method, text_method,
options`:

- `:category_id` — attribute ของ `post` ที่จะถูกตั้งค่า (ไม่ใช่ `:category` object)
- `@categories` — ActiveRecord relation หรือ Array ของ object ใดๆ ก็ได้ที่ต้องการให้เป็นตัวเลือก
- `:id` — เมธอดที่เรียกกับแต่ละ object ใน collection เพื่อได้ `value` ของ `<option>`
- `:name` — เมธอดที่เรียกกับแต่ละ object ใน collection เพื่อได้ label ที่แสดงให้ผู้ใช้เห็น
- `{ include_blank: "..." }` — ตัวเลือกว่างที่ให้เลือก "ไม่ระบุ" ได้ (จำเป็นเพราะ `category`
  เป็น `optional: true`)

**ความต่างสำคัญระหว่าง `select` กับ `collection_select`:** `select` ให้เขียน array ตัวเลือกเอง
ตรงๆ ใน view (เหมาะกับตัวเลือกคงที่ไม่กี่ตัว) ส่วน `collection_select` ดึงตัวเลือกจากฐานข้อมูล
จริงแบบไดนามิก (เหมาะกับข้อมูลที่เพิ่ม/แก้ไขได้ เช่น หมวดหมู่ที่ผู้ดูแลระบบเพิ่มเองได้ภายหลัง) —
ถ้าใช้ `select` กับข้อมูลแบบหลังจะต้องคอย hardcode รายการใหม่ทุกครั้งที่มีหมวดหมู่เพิ่ม ซึ่งผิด
หลักการออกแบบ

### 4. `radio_button` — เลือกได้ตัวเดียวจากหลายตัวเลือกที่เห็นพร้อมกันหมด

```erb
<label class="inline"><%= f.radio_button :visibility, "public" %> สาธารณะ</label>
<label class="inline"><%= f.radio_button :visibility, "private" %> ส่วนตัว</label>
```

```html
<label class="inline">
  <input type="radio" value="public" checked="checked" name="post[visibility]"
    id="post_visibility_public" /> สาธารณะ
</label>
<label class="inline">
  <input type="radio" value="private" name="post[visibility]"
    id="post_visibility_private" /> ส่วนตัว
</label>
```

สังเกตว่า `f.radio_button :visibility, "public"` เรียกซ้ำ 2 ครั้งด้วย value คนละตัว (`"public"`,
`"private"`) แต่ทั้งคู่ได้ `name="post[visibility]"` เหมือนกัน — เบราว์เซอร์จะรู้เองว่า radio
button ที่มี `name` เดียวกันคือกลุ่มเดียวกัน เลือกได้ทีละอันเท่านั้น และ id ที่ Rails สร้างให้จะ
ต่อท้ายด้วยค่า value (`post_visibility_public`, `post_visibility_private`) เพื่อไม่ให้ id ชนกัน
— `checked="checked"` ติดมาที่ `"public"` อัตโนมัติเพราะเป็นค่า default จาก migration เหมือนกับ
`select`

**ต่างจาก `select` ตรงไหน:** ทั้งคู่ใช้เลือก 1 จากหลายตัวเลือกเหมือนกัน แต่ `radio_button` แสดง
ทุกตัวเลือกพร้อมกันบนหน้าจอตลอดเวลา (เหมาะกับตัวเลือกน้อยๆ ที่อยากให้ผู้ใช้เห็นครบทุกตัวโดยไม่ต้อง
คลิกเปิด dropdown) ส่วน `select` เหมาะกับตัวเลือกที่มีเยอะ (ประหยัดพื้นที่หน้าจอ)

### 5. `check_box` — ค่า boolean ตัวเดียว (checked/unchecked)

```erb
<label class="inline"><%= f.check_box :featured %> บทความแนะนำ (featured)</label>
```

```html
<label class="inline">
  <input name="post[featured]" type="hidden" value="0" />
  <input type="checkbox" value="1" name="post[featured]" id="post_featured" /> บทความแนะนำ (featured)
</label>
```

จุดที่มักทำให้มือใหม่งงคือ **มี `<input>` ซ่อนกัน 2 ตัว** ไม่ใช่ตัวเดียว: hidden field
`value="0"` วางไว้**ก่อน** checkbox จริงเสมอ เหตุผลคือ **เบราว์เซอร์จะไม่ส่ง field ของ checkbox
ที่ไม่ได้ติ๊กเลือกไปกับ form submission** (นี่คือพฤติกรรมมาตรฐานของ HTML ไม่ใช่ของ Rails) ถ้าไม่มี
hidden field สำรองไว้ การ "เอาติ๊กออก" จาก checkbox ที่เคย checked อยู่แล้วจะทำให้ params ไม่มี
key `post[featured]` เลย (ไม่ใช่ `false` แต่คือ**หายไป**) ซึ่งทำให้ `@post.update(post_params)`
มองว่าไม่มีการเปลี่ยนแปลงค่า `featured` เลย (ค่าเก่ายังคงอยู่) — hidden field ที่มี `value="0"`
เขียนก่อนจึงประกันว่า**เสมอ**จะมี key `post[featured]` ส่งมาด้วย ถ้า checkbox ไม่ได้ติ๊ก ค่าที่ได้
คือ `"0"` (จาก hidden field) ถ้าติ๊กไว้ เบราว์เซอร์จะส่งค่าจาก checkbox จริง (`"1"`) มาทับค่าจาก
hidden field ตามลำดับที่ field ปรากฏใน HTML (browser ส่งค่าตามลำดับ ค่าหลังทับค่าก่อนเมื่อ key
ซ้ำกัน)

### 6. `date_select` — เลือกวันที่ผ่าน 3 dropdown (ปี/เดือน/วัน)

```erb
<%= f.date_select :published_on, include_blank: true %>
```

```html
<select id="post_published_on_1i" name="post[published_on(1i)]">
  <option value="" label=" "></option>
  <option value="2021">2021</option>
  ...
  <option value="2031">2031</option>
</select>
<select id="post_published_on_2i" name="post[published_on(2i)]">
  <option value="" label=" "></option>
  <option value="1">January</option>
  ...
  <option value="12">December</option>
</select>
<select id="post_published_on_3i" name="post[published_on(3i)]">
  <option value="" label=" "></option>
  <option value="1">1</option>
  ...
  <option value="31">31</option>
</select>
```

`date_select` สร้าง `<select>` **3 ตัวแยกกัน** ไม่ใช่ 1 ตัว — สังเกตชื่อ field ที่แปลกตา:
`published_on(1i)`, `published_on(2i)`, `published_on(3i)` เลข `1i`/`2i`/`3i` หมายถึง "ส่วนที่ 1/
2/3 ของการประกอบเป็น multi-parameter attribute" (`i` ย่อมาจาก integer) — เมื่อ params เหล่านี้
มาถึง controller, ActiveRecord (ผ่านกลไก **multiparameter attributes**) จะรวมทั้ง 3 ค่ากลับเป็น
`Date` object เดียวให้อัตโนมัติก่อน assign เข้า `published_on` โดยไม่ต้องเขียนโค้ดแปลงเอง —
ปีที่แสดงเป็น dropdown default จะเป็นช่วง ±5 ปีจากปีปัจจุบัน (ปรับได้ด้วย option `start_year:`/
`end_year:`) `include_blank: true` เพิ่มตัวเลือกว่างในทั้ง 3 dropdown เผื่อผู้ใช้ไม่ต้องการระบุ
วันที่เผยแพร่ตอนนี้ (เช่น เก็บเป็นฉบับร่างไว้ก่อน)

> **ทางเลือกสมัยใหม่กว่า:** ถ้าต้องการ UX ที่ดีกว่า (date picker เดียวจากปฏิทิน ไม่ใช่ 3
> dropdown) ใช้ `f.date_field :published_on` แทนได้ (render เป็น `<input type="date">` ตัวเดียว
> ที่เบราว์เซอร์สมัยใหม่แสดงเป็น date picker ให้อัตโนมัติ) — `date_select` ยังคงมีประโยชน์ในกรณีที่
> ต้องการ fallback ที่ทำงานได้แน่นอนแม้เบราว์เซอร์เก่าไม่รองรับ `<input type="date">` หรือ
> ต้องการควบคุมช่วงปีที่เลือกได้อย่างละเอียด

---

## Step 304: การแสดง error — `full_messages`, `field_with_errors`, `field_error_proc`

### `errors.full_messages` และ `errors.each` (ทบทวนจาก Part 026/030)

```erb
<% if post.errors.any? %>
  <div class="form-errors">
    <h3><%= pluralize(post.errors.count, "ข้อผิดพลาด", plural: "ข้อผิดพลาด") %>ทำให้บันทึกบทความนี้ไม่ได้:</h3>
    <ul>
      <% post.errors.each do |error| %>
        <li><%= error.full_message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

ทดสอบจริงด้วยการส่ง `title` ว่างเปล่าและ `body` สั้นเกินไป:

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=" \
  --data-urlencode "post[body]=สั้น" \
  --data-urlencode "post[status]=draft" \
  --data-urlencode "post[visibility]=public" \
  -o invalid_resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

HTML ที่ได้จริง:

```html
<div class="form-errors">
  <h3>2 ข้อผิดพลาดทำให้บันทึกบทความนี้ไม่ได้:</h3>
  <ul>
    <li>Title can&#39;t be blank</li>
    <li>Body is too short (minimum is 10 characters)</li>
  </ul>
</div>
```

### `ActionView::Base.field_error_proc` — Rails ห่อฟิลด์ที่ผิดให้อัตโนมัติโดยไม่ต้องขอ

สิ่งที่หลายคนไม่รู้ (เพราะไม่เคยเปิดดู HTML จริงของฟิลด์ที่ error) คือ Rails **ห่อฟิลด์ทุกตัวที่
attribute นั้นมี error อยู่ด้วย `<div class="field_with_errors">` ให้อัตโนมัติ** โดยไม่ต้องเขียน
โค้ดเพิ่มเลยแม้แต่บรรทัดเดียว ทดสอบจริงจาก request เดียวกันข้างบน (ที่ `title` และ `body` ผิดทั้ง
คู่):

```html
<div class="field_with_errors"><label for="post_title">ชื่อบทความ</label></div>
<div class="field_with_errors"><input type="text" value="" name="post[title]" id="post_title" /></div>

<div class="field_with_errors"><label for="post_body">เนื้อหา</label></div>
<div class="field_with_errors"><textarea rows="6" name="post[body]" id="post_body">
</textarea></div>
```

สังเกตว่า**ทั้ง `label` และ `input`/`textarea` ถูกห่อแยกกันคนละ `<div>`** (ไม่ใช่ห่อรวมกันเป็น
`<div>` เดียว) — กลไกนี้มาจาก `ActionView::Base.field_error_proc` ซึ่งเป็นค่า config ที่นิยามไว้
ในซอร์สโค้ดของ Rails เองประมาณนี้:

```ruby
# actionview/lib/action_view/base.rb (ซอร์สโค้ดจริงของ Rails 8.1.4)
cattr_accessor :field_error_proc, default: Proc.new { |html_tag, instance|
  content_tag :div, html_tag, class: "field_with_errors"
}
```

ทุกครั้งที่ form builder helper (`f.text_field`, `f.label`, ฯลฯ) สร้าง tag ให้ attribute ที่มี
error อยู่ Rails จะเรียก proc ตัวนี้ห่อ tag นั้นด้วย `<div class="field_with_errors">` เสมอ — นี่
คือเหตุผลที่หลายโปรเจกต์ต้องเขียน CSS rule สำหรับ class นี้เพื่อให้ layout ไม่พังตอนมี error (เช่น
`.field_with_errors { display: contents; }` เพื่อไม่ให้ `<div>` แทรกตัวมามีผลต่อ layout ที่ใช้
flexbox/grid)

**ปรับแต่ง/ปิดพฤติกรรมนี้ได้** โดยแก้ค่า `field_error_proc` ใน initializer (มักทำเมื่อใช้ CSS
framework ที่มีคลาสของตัวเองสำหรับ field ที่ error เช่น Bootstrap's `is-invalid`):

```ruby
# config/initializers/field_error_proc.rb
ActionView::Base.field_error_proc = Proc.new { |html_tag, instance| html_tag }
```

โค้ดข้างบนทำให้ Rails **หยุดห่อ** field ที่ error ด้วย `<div>` เลย (คืน tag เดิมกลับไปเฉยๆ) เหมาะ
กับทีมที่อยากจัดการ error styling เองทั้งหมดผ่าน CSS class ที่ระบุเองในฟอร์ม ไม่ต้องพึ่งพฤติกรรม
default ของ Rails

> **ข้อควรระวัง:** `field_with_errors` เป็นพฤติกรรม default ของ **Rails ทั้งเฟรมเวิร์ก** ไม่ใช่
> อะไรที่ Mini Blog เขียนเอง ถ้าเจอ `<div class="field_with_errors">` โผล่มาในหน้าเว็บโดยไม่คาด
> คิดตอนทำโปรเจกต์จริง อย่าเพิ่งตกใจว่าโค้ดมี bug — นี่คือพฤติกรรมที่ถูกต้องแล้ว เพียงแต่ต้องเตรียม
> CSS รองรับไว้เท่านั้น

---

## Step 305: Strong Parameters เชิงลึก — `require`/`permit` เทียบกับ `params.expect`

### ทบทวน `require` + `permit` (รูปแบบดั้งเดิม)

```ruby
def post_params
  params.require(:post).permit(:title, :body, :category_id, :status, :visibility)
end
```

- `require(:post)` — บังคับว่า params ต้องมี key `"post"` ถ้าไม่มีจะ raise
  `ActionController::ParameterMissing` (Rails จับ exception นี้ให้อัตโนมัติแล้วตอบ
  **HTTP 400 Bad Request**)
- `.permit(...)` — whitelist ฟิลด์ที่อนุญาตให้ mass-assign ได้ ฟิลด์อื่นที่ไม่ได้ permit จะถูก
  ตัดทิ้งเงียบๆ (ทบทวนจาก **Part 023/030**)

### ทำไม Rails 8 ถึงแนะนำ `params.expect` แทน

Rails 7.1 เริ่มเพิ่ม `params.expect` เข้ามา และตั้งแต่ Rails 8 เอกสารทางการของ Rails
(`ActionController::Parameters#expect`) ระบุตรงๆ ว่า **"`expect` is the preferred way to require
and permit parameters. It is safer than the previous recommendation to call `permit` and
`require` in sequence, which could allow user triggered 500 errors."**

ปัญหาของ `require.permit` คือมัน**สมมติว่าค่าที่ `params[:post]` ชี้ไปต้องเป็น Hash เสมอ** ถ้ามี
คนจงใจ (หรือ client ที่เขียนโค้ดผิด) ส่งค่า `post` เป็น string ธรรมดาแทนที่จะเป็น hash ที่ซ้อนอยู่
`.require(:post)` จะคืนค่า string นั้นกลับมาตรง ๆ (เพราะ key `"post"` มีอยู่จริง ไม่ได้ raise
`ParameterMissing`) แล้วพอเรียก `.permit` ต่อบน string จะเกิด **`NoMethodError`** ทันที เพราะ
`String` ไม่มีเมธอด `permit` — exception นี้ **ไม่ใช่** `ActionController::ParameterMissing` ที่
Rails จับให้อัตโนมัติ จึงหลุดเป็น **HTTP 500 Internal Server Error** แทนที่จะเป็น 400 ตามที่ควร
เป็น (client ส่งข้อมูลผิดรูปแบบ ควรได้ 4xx ไม่ใช่ 5xx ที่สื่อว่า "server มีบัค")

**ทดสอบจริงเพื่อยืนยัน** ด้วย `bin/rails runner`:

```ruby
params = ActionController::Parameters.new(post: "hack")

params.require(:post).permit(:title, :body)
# => NoMethodError: undefined method `permit' for an instance of String

params.expect(post: [:title, :body])
# => ActionController::ParameterMissing: param is missing or the value is empty or invalid: post
```

ผลลัพธ์จริงตรงตามที่คาดไว้ทุกประการ — `expect` ตรวจสอบ**รูปร่าง (shape)** ของ params อย่าง
เข้มงวดกว่า ถ้ารูปร่างไม่ตรงกับที่คาดไว้ (ควรเป็น Hash แต่ได้ String มา) จะ raise
`ActionController::ParameterMissing` แบบเดียวกับกรณี "ไม่มี key เลย" ทำให้ error path เดียวกันครอบ
คลุมทั้งสองกรณี (key หาย + รูปร่างผิด) และ Rails จับ handle ให้เป็น 400 เสมอ

**ยืนยันผ่าน HTTP จริงด้วย** — สร้าง route/controller เดียวกันแต่สลับ implementation ของ
`post_params` แล้วยิง request จริงที่มี `post=hack` (ไม่ใช่ nested hash):

```bash
# เมื่อ post_params ใช้ require(:post).permit(...)
curl -s -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" --data-urlencode "post=hack" \
  -o resp.html -w "HTTP %{http_code}\n"
```
```
HTTP 500
```

```bash
# เมื่อ post_params ใช้ params.expect(post: [...])
curl -s -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" --data-urlencode "post=hack" \
  -o resp.html -w "HTTP %{http_code}\n"
```
```
HTTP 400
```

**request เดียวกันเป๊ะ ต่างกันแค่ implementation ของ `post_params` ฝั่ง controller** — ผลลัพธ์
เปลี่ยนจาก 500 (unhandled exception, exposes stack trace ถ้า `consider_all_requests_local` เป็น
true) เป็น 400 (handled gracefully) — นี่คือเหตุผลตัวจริงที่ Rails 8 เปลี่ยนคำแนะนำมาใช้ `expect`
เป็นค่าเริ่มต้น ไม่ใช่แค่เรื่อง syntax สวยกว่า

### Syntax พื้นฐานของ `params.expect`

```ruby
# require(:post).permit(:title, :body) แบบเดิม
params.require(:post).permit(:title, :body)

# เทียบเท่ากันด้วย params.expect
params.expect(post: [:title, :body])
```

ทั้งสองแบบได้ผลลัพธ์เหมือนกันทุกกรณีปกติ (เมื่อ params ถูกรูปแบบ) ต่างกันแค่พฤติกรรมตอน params
**ผิดรูปแบบ** ตามที่พิสูจน์ไว้ข้างบน — Rails 8 แนะนำให้ใช้ `expect` เป็นค่าเริ่มต้นสำหรับโค้ด
ใหม่ทั้งหมด ส่วน `require.permit` ยังใช้งานได้ปกติ (ไม่ได้ deprecated) เพียงแต่ไม่ใช่ "วิธีที่
แนะนำที่สุด" อีกต่อไป

หลักสูตรนี้จะใช้ `params.expect` เป็นหลักตั้งแต่ Part นี้เป็นต้นไป — `PostsController` เวอร์ชัน
เต็มจะปรากฏใน Step 310

### หมายเหตุด้านความปลอดภัย — ทำไมต้องมี Strong Parameters เลยตั้งแต่แรก

ก่อนจะมี strong parameters (นำมาใช้เป็นค่าเริ่มต้นตั้งแต่ Rails 4) Rails รุ่นเก่ายอมให้เขียน
`Post.new(params[:post])` ตรงๆ โดยไม่มี whitelist ใดๆ เลย — ช่องโหว่นี้เรียกว่า **mass assignment
vulnerability**: ถ้าโมเดลมี attribute ที่ไม่ควรให้ผู้ใช้แก้เอง (เช่น `admin: boolean`,
`account_balance: integer`, หรือในกรณีของ Mini Blog คือ `created_at`) ผู้ไม่หวังดีสามารถแนบ field
พิเศษนั้นเข้าไปใน form submission ตรงๆ (แม้ฟอร์ม HTML จริงจะไม่มี input ให้กรอกฟิลด์นั้นเลยก็ตาม —
เพราะ HTTP request แก้ไข/ปลอมแปลงเองได้อย่างอิสระโดยไม่ผ่านฟอร์มที่เห็นในเบราว์เซอร์) แล้วเปลี่ยน
ค่าที่ไม่ควรเปลี่ยนได้ทันที เช่น ส่ง `post[admin]=true` แนบไปกับฟอร์มสมัครสมาชิกทั้งที่ฟอร์มจริงมี
แค่ช่องกรอก email/password

Strong parameters (`permit`/`expect`) แก้ปัญหานี้โดยบังคับให้ **ต้องระบุ whitelist อย่างชัดเจน**
ว่า attribute ไหนอนุญาตให้ mass-assign ได้ — attribute ที่ไม่ได้ระบุไว้จะถูกตัดทิ้งเสมอไม่ว่า
client จะพยายามส่งมาอย่างไรก็ตาม เรื่องนี้จะเจาะลึกอีกครั้งพร้อมตัวอย่าง exploit จริงและเทคนิค
ป้องกันขั้นสูงกว่าใน **Phase 13 (Part 079: OWASP Top 10 ใน Rails, Part 080: Mass assignment
เจาะลึก)** — ตอนนี้แค่เข้าใจหลักการและใช้ `permit`/`expect` ให้ถูกต้องเสมอก็เพียงพอสำหรับ Phase 4

---

## Step 306: Permit nested/array params — `tags: []` และ `comments_attributes: [...]`

### Array ของ scalar values — `tags: []`

Mini Blog เวอร์ชันขยายมีฟิลด์ `tags` เป็น array ของ string (เช่น `["ruby", "rails"]`) — ฝั่ง HTML
ใช้ checkbox หลายตัวที่มีชื่อ (`name`) เดียวกันลงท้ายด้วย `[]`:

```erb
<%= hidden_field_tag "post[tags][]", "" %>
<% %w[ruby rails tutorial beginner].each do |tag| %>
  <label class="inline">
    <%= check_box_tag "post[tags][]", tag, post.tags.include?(tag), id: "post_tags_#{tag}" %>
    <%= tag %>
  </label>
<% end %>
```

```html
<input type="hidden" name="post[tags][]" id="post_tags_" value="" />
<label class="inline">
  <input type="checkbox" name="post[tags][]" id="post_tags_ruby" value="ruby" /> ruby
</label>
<label class="inline">
  <input type="checkbox" name="post[tags][]" id="post_tags_rails" value="rails" /> rails
</label>
<!-- ... tutorial, beginner เหมือนกัน -->
```

**สังเกต 2 จุดสำคัญ:**

1. ใช้ `check_box_tag` (ไม่ใช่ `f.check_box`) — เพราะ `f.check_box` ผูกกับ **1 attribute boolean
   เดียว** เสมอ (เห็นแล้วใน Step 303) แต่ที่นี่ต้องการ checkbox **หลายตัวที่ผลลัพธ์รวมกันเป็น
   array เดียว** ซึ่งเป็นคนละแบบกัน `check_box_tag` เป็น helper ระดับต่ำกว่า (ไม่ผูกกับ form
   builder object) ที่ยอมให้กำหนด `name` เองตรงๆ รวมถึงตั้งชื่อซ้ำกันได้หลายตัว
2. `hidden_field_tag "post[tags][]", ""` ที่วางไว้ก่อน — เหตุผลเดียวกับ hidden field ของ
   `check_box` ใน Step 303: ถ้าผู้ใช้ไม่ติ๊กอะไรเลยสักตัว เบราว์เซอร์จะไม่ส่ง key
   `post[tags][]` มาเลย ทำให้ `params[:post][:tags]` เป็น `nil` แทนที่จะเป็น `[]` — hidden field
   ว่างเปล่าตัวนี้ประกันว่าอย่างน้อย key `post[tags][]` จะมีค่าหนึ่งค่าเสมอ (string ว่าง) ทำให้
   `params[:post][:tags]` เป็น array เสมอ (ถึงจะมี `""` ปนอยู่ก็ตาม — filter ออกฝั่ง Model ตาม
   ที่เห็นใน Step 302: `before_validation { self.tags = Array(tags).reject(&:blank?) }`)

**Strong parameters สำหรับ array ของ scalar:**

```ruby
# แบบ permit
params.require(:post).permit(tags: [])

# แบบ expect
params.expect(post: [{ tags: [] }])
```

`tags: []` (array เปล่า) เป็น syntax พิเศษที่บอก Rails ว่า `"tags"` คาดหวังเป็น **array ของค่า
เดี่ยวๆ (scalar)** ไม่ใช่ array ของ hash — ถ้าเขียนแค่ `permit(:tags)` เฉยๆ (ไม่มี `: []`)
Rails จะปฏิเสธค่าที่เป็น array ทันที (มองว่าเป็นรูปแบบที่ไม่คาดคิด แล้วตัดทิ้งเงียบๆ)

ทดสอบจริงด้วยการสร้างโพสต์ที่ติ๊ก tag `ruby` และ `rails`:

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบฟอร์มขั้นสูง" \
  --data-urlencode "post[body]=เนื้อหาทดสอบฟอร์มที่มีความยาวมากกว่าสิบตัวอักษรแน่นอน" \
  --data-urlencode "post[tags][]=" \
  --data-urlencode "post[tags][]=ruby" \
  --data-urlencode "post[tags][]=rails" \
  ... (ฟิลด์อื่นตามปกติ)
```

ตรวจสอบผลจริงผ่าน console:

```irb
irb> Post.last.tags
=> ["ruby", "rails"]
```

`""` ที่มาจาก hidden field ถูก `reject(&:blank?)` กรองออกไปแล้วก่อนบันทึก เหลือแค่ tag ที่ผู้ใช้
ติ๊กเลือกจริง

### Array ของ Hash (nested attributes) — `comments_attributes: [...]` และกับดักของ `expect`

`comments_attributes` (จาก `accepts_nested_attributes_for`, อธิบายเต็มใน Step 307) มีรูปร่าง
ต่างจาก `tags` — มันคือ **Hash ที่ key เป็นตัวเลขดัชนี** ("0", "1", "2", ...) แต่ละ key ชี้ไปยัง
Hash ของ attribute หนึ่ง record (ไม่ใช่ Array ตรงๆ แบบที่ JSON API ทั่วไปส่งกัน) — รูปแบบนี้มาจาก
วิธีที่ `fields_for` render field ออกมา (ดู Step 308):

```
post[comments_attributes][0][commenter] = "สมชาย"
post[comments_attributes][0][body]      = "ความคิดเห็นแรก"
post[comments_attributes][1][commenter] = "สมหญิง"
post[comments_attributes][1][body]      = "ความคิดเห็นที่สอง"
```

**แบบ `permit` แบบดั้งเดิม** เขียนตรงไปตรงมา:

```ruby
params.require(:post).permit(
  :title, :body,
  comments_attributes: [:id, :commenter, :body, :_destroy]
)
```

**แบบ `expect` มีกับดักที่ต้องระวังมาก** — ทดสอบจริงเพื่อพิสูจน์:

```ruby
raw = {
  post: {
    title: "Hello", body: "Body text here long enough",
    comments_attributes: { "0" => { commenter: "A", body: "hi" } }
  }
}
params = ActionController::Parameters.new(raw)

# ผิด — ใช้ syntax แบบ permit เดิม (single bracket) กับ expect
permitted = params.expect(post: [:title, :body, { comments_attributes: [:commenter, :body] }])
permitted[:comments_attributes]
# => {} (ว่างเปล่า! ข้อมูลหายไปเงียบๆ โดยไม่มี error ใดๆ เตือนเลย)

# ถูก — ต้องใช้ double-bracket (array ซ้อน array) แม้ค่าจริงจะเป็น Hash ก็ตาม
permitted = params.expect(post: [:title, :body, { comments_attributes: [[:commenter, :body]] }])
permitted[:comments_attributes]
# => {"0"=>{"commenter"=>"A", "body"=>"hi"}}   (ถูกต้อง ได้ข้อมูลครบ)
```

ผลลัพธ์ทั้งสองกรณีนี้ทดสอบจริงแล้วตรงตามที่แสดง — **นี่คือกับดักที่อันตรายที่สุดของการย้ายจาก
`permit` มาใช้ `expect`**: syntax `{ comments_attributes: [:id, :body] }` (single bracket) ที่
เคยใช้ได้กับ `.permit` **ใช้ไม่ได้กับ `.expect`** สำหรับ nested attributes — ต้องเปลี่ยนเป็น
double-bracket `{ comments_attributes: [[:id, :body]] }` เสมอ (สังเกต `[[ ... ]]` ที่มีวงเล็บ
array ซ้อนกัน 2 ชั้น) ที่แย่กว่านั้นคือ **syntax ผิดแบบนี้ไม่ raise error ใดๆ เลย** — มันแค่คืนค่า
`{}` (Hash ว่าง) เงียบๆ ทำให้ nested record ที่ผู้ใช้กรอกมาหายไปโดยไม่มีใครสังเกตเห็นจนกว่าจะมีคน
บ่นว่า "กรอกความคิดเห็นแล้วทำไมไม่บันทึก"

> **กฎจำง่ายสำหรับ `params.expect`:** ทุกครั้งที่ nested key เป็น **array ของ record จริงๆ** (ไม่
> ว่าจะมาในรูป JSON array หรือ Hash แบบมีดัชนีจาก `fields_for`) ให้ใช้ **double-bracket เสมอ**
> (`{ key: [[:field1, :field2]] }`) ส่วนถ้า nested key เป็น **array ของ scalar ธรรมดา** (string/
> number เดี่ยวๆ ไม่มีโครงสร้างย่อย) ให้ใช้ **array เปล่า** (`{ key: [] }`) ไม่ต้องมี bracket ซ้อน
> — ทั้งสองกรณีต่างกันตรงที่ว่า "ข้างในแต่ละช่องคือ scalar หรือ hash" ไม่เกี่ยวกับว่าข้อมูลจริง
> เป็น Array หรือ Hash ที่ฝั่ง Ruby

`post_params` เวอร์ชันเต็มที่ใช้ทั้ง `tags: []` และ `comments_attributes: [[...]]` พร้อมกัน (จาก
`PostsController` ที่จะใช้จริงตลอด Part นี้):

```ruby
def post_params
  params.expect(
    post: [
      :title, :body, :category_id, :status, :visibility, :featured, :published_on,
      { tags: [] },
      { comments_attributes: [[:id, :commenter, :body, :_destroy]] }
    ]
  )
end
```

---

## Step 307: `accepts_nested_attributes_for` ฝั่ง Model

### ประกาศความสามารถ "สร้าง/แก้ไข/ลบ record ลูกผ่าน record แม่" ที่ฝั่ง Model

```ruby
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy

  accepts_nested_attributes_for :comments, allow_destroy: true, reject_if: :all_blank

  # ...
end
```

`accepts_nested_attributes_for :comments` เพิ่มเมธอด **`comments_attributes=`** เข้าไปใน `Post`
โดยอัตโนมัติ — เมธอดนี้รับ Hash (หรือ Array) ของ attribute หลายชุด แล้วทำสิ่งเหล่านี้ให้ทั้งหมด
ในการเรียกครั้งเดียว:

- ถ้า attribute ชุดนั้น**ไม่มี** `:id` → **สร้าง** `Comment` ใหม่ (ผูกกับ `post_id` ให้อัตโนมัติ
  ผ่าน association เหมือน `@post.comments.build` ที่เรียนใน Part 030)
- ถ้า attribute ชุดนั้น**มี** `:id` ที่ตรงกับ comment ที่มีอยู่แล้ว → **อัปเดต** comment ตัวนั้น
- ถ้า attribute ชุดนั้น**มี** `:id` **และ** `_destroy` เป็นค่า truthy (`"1"`, `true`) → **ลบ**
  comment ตัวนั้น (ต้องมี `allow_destroy: true` เท่านั้นถึงจะทำงาน ดูละเอียดใน Step 309)

ทั้งหมดนี้เกิดขึ้น**ในทรานแซกชันเดียวกัน**กับการ `save`/`update` ของ `Post` เอง — ถ้า record ลูก
ตัวใดตัวหนึ่ง validate ไม่ผ่าน ทั้งการบันทึก `Post` และ `Comment` ทั้งหมดจะ**ยกเลิกทั้งหมด**ไม่มี
อะไรถูกบันทึกเลยแม้แต่ตัวเดียว (ความสมบูรณ์ของทรานแซกชัน — ทบทวนแนวคิดนี้จาก **Part 026**)

### `reject_if: :all_blank` — ไม่สร้าง record เปล่าจากแถวฟอร์มที่ไม่ได้กรอกอะไรเลย

ฟอร์มที่มี `fields_for` มักเตรียมแถวเปล่าไว้ล่วงหน้าให้ผู้ใช้กรอกเพิ่มได้ (ดู Step 308 — เตรียม
comment เปล่าไว้ 3 แถวในหน้า `new`) แต่ถ้าผู้ใช้ไม่กรอกอะไรเลยในบางแถว ไม่ควรสร้าง `Comment`
เปล่าๆ ขึ้นมาในฐานข้อมูล — `reject_if: :all_blank` บอก Rails ว่า **ถ้าทุก attribute ใน record
ลูกชุดนั้นเป็นค่าว่างทั้งหมด (ไม่นับ `id` และ `_destroy`) ให้ข้าม record นั้นไปเลย** ไม่ต้องพยายาม
สร้างหรือ validate มัน

ทดสอบจริง — ส่งฟอร์มที่มี comment 3 แถว แต่กรอกจริงแค่แถวเดียว อีก 2 แถวเว้นว่างไว้ทั้งหมด:

```irb
irb> post = Post.last
irb> post.comments.count
=> 1
```

จากทั้งหมด 3 แถวที่ส่งมา มีแค่ **1 record** ที่ถูกสร้างจริง — อีก 2 แถวที่ว่างเปล่าถูก
`reject_if: :all_blank` กรองทิ้งไปตั้งแต่ก่อน validate เลย ไม่มี `Comment` เปล่าๆ หลงเหลือใน
ฐานข้อมูล

`reject_if` รับได้ทั้ง symbol (ชื่อเมธอดที่นิยามเองก็ได้ ไม่จำเป็นต้องเป็น `:all_blank` ที่ Rails
เตรียมไว้ให้) หรือ `Proc`:

```ruby
# ตัวอย่าง custom logic — ข้าม record ถ้า commenter ว่างเปล่า (ไม่สนใจ body)
accepts_nested_attributes_for :comments,
  allow_destroy: true,
  reject_if: proc { |attrs| attrs["commenter"].blank? }
```

---

## Step 308: `fields_for` — เรนเดอร์ฟอร์มของโมเดลลูกซ้อนอยู่ในฟอร์มของโมเดลแม่

### Controller เตรียม record ลูกเปล่าไว้ให้ view

```ruby
def new
  @post = Post.new
  3.times { @post.comments.build }
end

def edit
  @post.comments.build   # เผื่อไว้ให้มีช่องความคิดเห็นใหม่ว่างเสมอ 1 แถวสำหรับพิมพ์เพิ่มตอนแก้ไข
end
```

ทบทวนหลักการจาก **Part 023 Step 227**: controller ต้องเตรียมทุกอย่างที่ view ต้องใช้ไว้ล่วงหน้า
เสมอ — `@post.comments.build` (ไม่มี argument) สร้าง `Comment` เปล่าที่ยังไม่ persisted แต่ผูกกับ
`@post` ผ่าน association แล้ว `fields_for` จะอ่านค่าจาก object เปล่านี้เพื่อ render field ว่างๆ
ให้กรอก — หน้า `new` เตรียมไว้ 3 แถว (ให้กรอก comment แรกๆ พร้อมกับสร้างบทความได้เลย) ส่วนหน้า
`edit` เตรียมเพิ่มอีก 1 แถวว่างต่อจาก comment เดิมที่มีอยู่แล้ว (สำหรับ "เพิ่มความคิดเห็นใหม่ระหว่าง
แก้ไข")

### `fields_for` ใน `_form.html.erb`

```erb
<%= form_with model: post, local: true do |f| %>
  <%# ... field อื่นๆ ของ post เอง (title, body, category ฯลฯ) ... %>

  <fieldset class="nested-comments">
    <legend>ความคิดเห็นแนบมาพร้อมบทความ (ไม่บังคับ)</legend>

    <%= f.fields_for :comments do |cf| %>
      <div class="nested-comment-row">
        <% if cf.object.persisted? %>
          <%= cf.hidden_field :id %>
        <% end %>

        <div class="field">
          <%= cf.label :commenter, "ผู้แสดงความคิดเห็น" %>
          <%= cf.text_field :commenter %>
        </div>

        <div class="field">
          <%= cf.label :body, "ข้อความ" %>
          <%= cf.text_area :body, rows: 2 %>
        </div>

        <% if cf.object.persisted? %>
          <div class="field">
            <label class="inline">
              <%= cf.check_box :_destroy %> ลบความคิดเห็นนี้
            </label>
          </div>
        <% end %>
      </div>
    <% end %>
  </fieldset>

  <div class="actions">
    <%= f.submit "บันทึกบทความ" %>
  </div>
<% end %>
```

**อธิบายทีละจุด:**

- `f.fields_for :comments do |cf| ... end` — วนลูปผ่าน `post.comments` **ทุกตัว** (ทั้งที่
  persisted อยู่แล้วและที่เพิ่ง `build` ไว้เปล่าๆ) แล้ว yield form builder ตัวใหม่ (`cf`) ให้ต่อ
  ฟอร์มละ 1 record — `cf` ทำงานเหมือน `f` ทุกประการ (มี `text_field`, `label`, `check_box`
  ครบ) เพียงแต่ **scope ของชื่อ field เปลี่ยนไป**ให้ตรงกับโครงสร้าง nested attributes
  โดยอัตโนมัติ
- `cf.object.persisted?` — เช็คว่า comment แถวนี้เป็น record เก่าที่มีอยู่แล้วในฐานข้อมูล หรือ
  เป็นแถวเปล่าที่เพิ่ง `build` ไว้รอกรอกใหม่ — ใช้แยกว่าจะแสดง hidden field `id` และ checkbox
  "ลบ" หรือไม่ (ไม่มีประโยชน์ที่จะให้ "ลบ" record ที่ยังไม่เคยถูกสร้างขึ้นมาจริงเลย)
- `cf.hidden_field :id` — **จำเป็นมาก** สำหรับแถวที่ persisted แล้ว เพราะนี่คือสิ่งเดียวที่บอก
  `accepts_nested_attributes_for` ว่า "แถวนี้คือการแก้ไข record เดิม ไม่ใช่การสร้างใหม่" ถ้าลืม
  ใส่ hidden field นี้ ทุกครั้งที่ submit ฟอร์มแก้ไข จะกลายเป็น**สร้าง comment ใหม่ซ้ำ**แทนที่จะ
  แก้ไขของเดิม (เพราะ `comments_attributes=` มองว่าไม่มี `:id` มา แปลว่าต้องสร้างใหม่เสมอ)

HTML จริงที่ render ออกมาสำหรับหน้า `new` (record ใหม่ทั้ง 3 แถว ไม่มี `persisted?`):

```html
<div class="nested-comment-row">
  <div class="field">
    <label for="post_comments_attributes_0_commenter">ผู้แสดงความคิดเห็น</label>
    <input type="text" name="post[comments_attributes][0][commenter]"
      id="post_comments_attributes_0_commenter" />
  </div>
  <div class="field">
    <label for="post_comments_attributes_0_body">ข้อความ</label>
    <textarea rows="2" name="post[comments_attributes][0][body]"
      id="post_comments_attributes_0_body"></textarea>
  </div>
</div>
<!-- แถวที่ 1 และ 2 เหมือนกัน เปลี่ยนแค่เลขดัชนี [1], [2] -->
```

และสำหรับหน้า `edit` ของ post ที่มี comment เดิมอยู่แล้ว 1 อัน (id=1, persisted):

```html
<div class="nested-comment-row">
  <input type="hidden" value="1" name="post[comments_attributes][0][id]"
    id="post_comments_attributes_0_id" />
  <div class="field">
    <label for="post_comments_attributes_0_commenter">ผู้แสดงความคิดเห็น</label>
    <input type="text" value="สมชาย" name="post[comments_attributes][0][commenter]"
      id="post_comments_attributes_0_commenter" />
  </div>
  <div class="field">
    <label for="post_comments_attributes_0_body">ข้อความ</label>
    <textarea rows="2" name="post[comments_attributes][0][body]"
      id="post_comments_attributes_0_body">ความคิดเห็นแรกที่แนบมาพร้อมบทความ</textarea>
  </div>
  <div class="field">
    <label class="inline">
      <input name="post[comments_attributes][0][_destroy]" type="hidden" value="0" />
      <input type="checkbox" value="1" name="post[comments_attributes][0][_destroy]"
        id="post_comments_attributes_0__destroy" /> ลบความคิดเห็นนี้
    </label>
  </div>
</div>
```

สังเกตว่า field `id` มี `value="1"` ติดมาด้วย (ค่าจริงจากฐานข้อมูล) และ `commenter`/`body` มีค่า
เดิมแสดงอยู่แล้ว (`value="สมชาย"`, เนื้อหาข้างในของ `<textarea>`) — เหมือนกับที่ `form_with
model: post` เติมค่าเดิมให้ field ของ `post` เอง `fields_for` ก็ทำแบบเดียวกันให้ field ของ
`comments` ที่ซ้อนอยู่ข้างในโดยอัตโนมัติทุกประการ

---

## Step 309: `_destroy` และ `allow_destroy: true` — ลบ record ลูกผ่านฟอร์มของ record แม่

### กลไกเบื้องหลัง `_destroy`

`_destroy` **ไม่ใช่ column จริงในตาราง `comments`** — มันคือ "pseudo-attribute" พิเศษที่
`accepts_nested_attributes_for` รู้จักเป็นกรณีพิเศษเท่านั้น เมื่อ Rails เห็นว่า attribute ชุดหนึ่ง
มี `_destroy` เป็นค่า truthy (`"1"`, `true`, `1`) **และ** `allow_destroy: true` ถูกตั้งไว้ที่ฝั่ง
Model มันจะเรียก `.destroy` กับ record นั้นแทนที่จะพยายามอัปเดต — ถ้า `allow_destroy` เป็น
`false` (ค่า default) การส่ง `_destroy: "1"` มาจะ**ถูกเพิกเฉยเงียบๆ** (ไม่ error แต่ก็ไม่ลบด้วย)
ซึ่งเป็นพฤติกรรมที่ตั้งใจให้ปลอดภัยไว้ก่อน — ต้องเปิดใช้งานเองอย่างชัดเจนถึงจะลบได้จริง

### ทดสอบจริง — ลบ comment ผ่านฟอร์มแก้ไขของ Post (ไม่ใช่ปุ่มลบแยกต่างหาก)

ส่ง `PATCH /posts/1` พร้อม `_destroy=1` สำหรับ comment id=1 ที่มีอยู่แล้ว:

```bash
curl -s -c cookies.txt -b cookies.txt -X PATCH http://localhost:3000/posts/1 \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบฟอร์มขั้นสูง" \
  --data-urlencode "post[body]=เนื้อหาทดสอบฟอร์มที่มีความยาวมากกว่าสิบตัวอักษรแน่นอน" \
  --data-urlencode "post[status]=published" \
  --data-urlencode "post[visibility]=private" \
  --data-urlencode "post[comments_attributes][0][id]=1" \
  --data-urlencode "post[comments_attributes][0][commenter]=สมชาย" \
  --data-urlencode "post[comments_attributes][0][body]=ความคิดเห็นแรกที่แนบมาพร้อมบทความ" \
  --data-urlencode "post[comments_attributes][0][_destroy]=1" \
  -D - -o resp.html -w "HTTP %{http_code}\n" | grep -E "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/posts/1
HTTP 302
```

ตรวจสอบผลจริง:

```irb
irb> Post.find(1).comments.count
=> 0
```

comment id=1 หายไปจากฐานข้อมูลจริง — **ทั้งหมดนี้เกิดจาก request เดียว** (`PATCH /posts/1`) ไม่ได้
ยิง `DELETE /comments/1` แยกต่างหากเลย — นี่คือความต่างสำคัญระหว่างการลบผ่าน
`CommentsController#destroy` (ปุ่มลบแยก ที่เรียนใน Part 030) กับการลบผ่าน nested attributes
(checkbox "ลบ" ที่อยู่**ในฟอร์มเดียวกัน**กับการแก้ไขข้อมูลอื่น) — วิธีหลังเหมาะกับ UX แบบ "แก้ไข
ทุกอย่างพร้อมกันในหน้าเดียว แล้วกดบันทึกทีเดียว" ซึ่งเป็นรูปแบบที่พบบ่อยมากในระบบจัดการเนื้อหา
(CMS) และฟอร์มขนาดใหญ่ที่มีรายการย่อยจำนวนมาก

### ผสมการสร้าง แก้ไข และลบพร้อมกันในการ submit เดียว

จุดที่ทรงพลังที่สุดของ nested attributes คือความสามารถทำ**ทั้งสามอย่างพร้อมกันในคำขอเดียว**:
แก้ไข comment หนึ่งอัน, ลบอีกอันหนึ่ง, และสร้างอันใหม่ — ทดสอบจริงด้วย post ที่มี comment 2 อัน
อยู่แล้ว (id=2 "A", id=3 "B"):

```bash
curl -s -c cookies.txt -b cookies.txt -X PATCH http://localhost:3000/posts/1 \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบฟอร์มขั้นสูง (แก้ไข)" \
  --data-urlencode "post[body]=เนื้อหาทดสอบฟอร์มที่มีความยาวมากกว่าสิบตัวอักษรแน่นอน" \
  --data-urlencode "post[status]=published" \
  --data-urlencode "post[visibility]=private" \
  --data-urlencode "post[comments_attributes][0][id]=2" \
  --data-urlencode "post[comments_attributes][0][commenter]=A-แก้ไขแล้ว" \
  --data-urlencode "post[comments_attributes][0][body]=Comment A (edited)" \
  --data-urlencode "post[comments_attributes][1][id]=3" \
  --data-urlencode "post[comments_attributes][1][commenter]=B" \
  --data-urlencode "post[comments_attributes][1][body]=Comment B" \
  --data-urlencode "post[comments_attributes][1][_destroy]=1" \
  --data-urlencode "post[comments_attributes][2][commenter]=คนใหม่" \
  --data-urlencode "post[comments_attributes][2][body]=ความคิดเห็นใหม่ที่เพิ่มพร้อมกันในการ submit เดียวกัน" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 302
```

ตรวจสอบผลจริง:

```irb
irb> Post.find(1).comments.pluck(:id, :commenter, :body)
=> [[2, "A-แก้ไขแล้ว", "Comment A (edited)"],
    [4, "คนใหม่", "ความคิดเห็นใหม่ที่เพิ่มพร้อมกันในการ submit เดียวกัน"]]
```

ผลลัพธ์ยืนยันครบทั้ง 3 ปฏิบัติการในคำขอเดียว: **id=2 ถูกแก้ไข** (commenter/body เปลี่ยนไป),
**id=3 หายไป** (ถูกลบเพราะ `_destroy=1`), และมี **id=4 ตัวใหม่** (สร้างขึ้นเพราะไม่มี `:id` ส่งมา
ในชุดข้อมูลที่ 3) — `accepts_nested_attributes_for` แยกแยะทั้ง 3 กรณีออกจากกันได้อัตโนมัติจาก
รูปร่างของแต่ละชุด attribute เพียงอย่างเดียว (มี `id` ไหม, มี `_destroy` truthy ไหม) โดยที่โค้ด
ฝั่ง Controller **ไม่ต้องเขียนเงื่อนไขแยกแยะเองแม้แต่บรรทัดเดียว** — นี่คือคุณค่าหลักของเทคนิคนี้

---

## Step 310: แบบฝึกหัด — ฟอร์ม Post ที่สร้าง/แก้ไข/ลบ Comment แบบซ้อนได้ในหน้าเดียว

### โจทย์

สร้างระบบ Mini Blog ที่ต่อยอดจากทุก Step ก่อนหน้าให้ครบวงจร โดยมีเงื่อนไขดังนี้:

1. หน้า `posts#new` ต้องมีฟอร์มเดียวที่สร้างทั้ง `Post` และ `Comment` แนบมาได้พร้อมกัน (อย่างน้อย
   3 แถวว่างให้กรอก โดยแถวที่ไม่ได้กรอกอะไรเลยต้องไม่ถูกบันทึกเป็น record เปล่า)
2. หน้า `posts#edit` ต้องแสดง comment เดิมทั้งหมดพร้อม checkbox "ลบ" ต่อท้ายแต่ละแถว และมีแถวว่าง
   เพิ่มอีก 1 แถวสำหรับพิมพ์ comment ใหม่ระหว่างแก้ไข
3. `post_params` ต้องใช้ `params.expect` (ไม่ใช่ `require.permit`) และ permit ให้ครบทั้ง
   `tags: []` และ `comments_attributes` แบบ nested
4. ต้องมี validation ทั้งฝั่ง `Post` และ `Comment` ทำงานถูกต้อง — ถ้า comment ที่แนบมาไม่ผ่าน
   validation (เช่น `commenter` ว่างเปล่า) ทั้งฟอร์มต้อง**ไม่บันทึกอะไรเลยแม้แต่ตัวเดียว** (ทั้ง
   post และ comment อื่นที่ถูกต้อง) — พิสูจน์ด้วยการทดสอบจริง

### เฉลยแบบเต็ม

โค้ดเฉลยของแบบฝึกหัดนี้**คือโค้ดชุดเดียวกันกับที่ประกอบขึ้นทีละชิ้นตลอด Step 302–309** ไม่มีอะไร
ต้องเขียนเพิ่มอีกเลย — สรุปตำแหน่งไฟล์ที่เกี่ยวข้องอีกครั้งเพื่อความชัดเจน:

- **`app/models/post.rb`** — เนื้อหาครบตามที่วางไว้ใน Step 302 (schema/validation) และ Step 307
  (`accepts_nested_attributes_for :comments, allow_destroy: true, reject_if: :all_blank`)
- **`app/controllers/posts_controller.rb`** — เนื้อหาครบตามโครงร่างใน Step 302 บวกกับ
  `post_params` เวอร์ชันเต็มที่ปิดท้าย Step 306 (ใช้ `params.expect` พร้อม `tags: []` และ
  `comments_attributes: [[:id, :commenter, :body, :_destroy]]`)
- **`app/views/posts/_form.html.erb`** — partial เดียวที่รวม helper ทุกตัวจาก Step 303
  (`select`/`collection_select`/`radio_button`/`check_box`/`date_select`), error display จาก
  Step 304, tags checkbox จาก Step 306, และ `fields_for` เต็มรูปแบบจาก Step 308
- **`app/views/posts/new.html.erb`** และ **`edit.html.erb`** — เหมือน Part 030 ทุกประการ (เรียก
  `<%= render "form", post: @post %>` ทั้งคู่ ต่างกันแค่หัวข้อหน้า) ไม่ต้องแก้อะไรเพิ่มเพราะ
  partial `_form` เดียวรองรับทั้งสองหน้าอยู่แล้วตามหลักการที่เรียนมาตั้งแต่ Part 024/030

สิ่งที่เหลือคือการ**ทดสอบยืนยันว่าโค้ดทั้งชุดนี้ตอบโจทย์ครบทั้ง 4 ข้อจริง** ด้วย `curl` จริง —
ไม่ใช่แค่เขียนโค้ดแล้วเชื่อว่ามันน่าจะทำงาน

### ทดสอบเงื่อนไขข้อ 1 — สร้างพร้อม comment 3 แถว มีแค่แถวเดียวที่กรอกจริง

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/new -o new.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความทดสอบฟอร์มขั้นสูง" \
  --data-urlencode "post[body]=เนื้อหาทดสอบฟอร์มที่มีความยาวมากกว่าสิบตัวอักษรแน่นอน" \
  --data-urlencode "post[category_id]=1" \
  --data-urlencode "post[status]=published" \
  --data-urlencode "post[visibility]=private" \
  --data-urlencode "post[featured]=0" \
  --data-urlencode "post[featured]=1" \
  --data-urlencode "post[published_on(1i)]=2026" \
  --data-urlencode "post[published_on(2i)]=9" \
  --data-urlencode "post[published_on(3i)]=30" \
  --data-urlencode "post[tags][]=" \
  --data-urlencode "post[tags][]=ruby" \
  --data-urlencode "post[tags][]=rails" \
  --data-urlencode "post[comments_attributes][0][commenter]=สมชาย" \
  --data-urlencode "post[comments_attributes][0][body]=ความคิดเห็นแรกที่แนบมาพร้อมบทความ" \
  --data-urlencode "post[comments_attributes][1][commenter]=" \
  --data-urlencode "post[comments_attributes][1][body]=" \
  --data-urlencode "post[comments_attributes][2][commenter]=" \
  --data-urlencode "post[comments_attributes][2][body]=" \
  -D - -o resp.html -w "HTTP %{http_code}\n" | grep -E "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/posts/1
HTTP 302
```

ตรวจสอบผลจริง:

```irb
irb> p = Post.find(1)
irb> p.category.name
=> "เทคโนโลยี"
irb> p.status
=> "published"
irb> p.visibility
=> "private"
irb> p.featured
=> true
irb> p.published_on
=> Wed, 30 Sep 2026
irb> p.tags
=> ["ruby", "rails"]
irb> p.comments.count
=> 1
irb> p.comments.first.commenter
=> "สมชาย"
```

ทุกฟิลด์บันทึกถูกต้องครบถ้วน และมี **comment แค่ 1 อัน** จากทั้งหมด 3 แถวที่ส่งมา — ตรงตามเงื่อนไข
ข้อ 1 ทุกประการ (`reject_if: :all_blank` ทำงานถูกต้อง)

### ทดสอบเงื่อนไขข้อ 2 — แก้ไขพร้อมลบ/แก้/เพิ่ม comment ในคำขอเดียว

(ทดสอบไว้ครบแล้วใน Step 309 — สร้าง comment สองอันเพิ่มก่อน แล้ว `PATCH` เดียวที่แก้ไขอันแรก
ลบอันที่สอง และเพิ่มอันใหม่พร้อมกัน ได้ผลลัพธ์ตรงตามคาด `[[2, "A-แก้ไขแล้ว", ...], [4, "คนใหม่",
...]]`)

### ทดสอบเงื่อนไขข้อ 3 — `post_params` ใช้ `params.expect`

ยืนยันแล้วใน Step 305–306 ว่า controller ใช้ `params.expect(post: [...])` จริง ไม่ใช่
`require.permit` และรองรับทั้ง `tags: []` (single bracket, array ของ scalar) กับ
`comments_attributes: [[...]]` (double bracket, array ของ hash) ถูกต้องตามกฎที่สรุปไว้ท้าย
Step 306

### ทดสอบเงื่อนไขข้อ 4 — validation ล้มเหลวที่ nested record ต้องยกเลิกการบันทึกทั้งหมด

ทดสอบด้วยการส่ง comment ที่มี `body` แต่ไม่มี `commenter` (validation
`validates :commenter, presence: true` ของ `Comment` ต้องไม่ผ่าน):

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3000/posts/new -o new2.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new2.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/posts \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "post[title]=บทความที่ควรจะพังตอนบันทึก" \
  --data-urlencode "post[body]=เนื้อหาที่ยาวพอสำหรับ validation ของ Post เอง" \
  --data-urlencode "post[status]=draft" \
  --data-urlencode "post[visibility]=public" \
  --data-urlencode "post[comments_attributes][0][commenter]=" \
  --data-urlencode "post[comments_attributes][0][body]=มี body แต่ไม่มีชื่อผู้แสดงความคิดเห็น" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

ได้ **HTTP 422** (validation ไม่ผ่าน render กลับหน้า `new` เหมือนเดิม ไม่ redirect) ตรวจสอบว่า
ไม่มีอะไรถูกบันทึกลงฐานข้อมูลเลยแม้แต่ `Post` ที่ตัวมันเองผ่าน validation:

```irb
irb> Post.exists?(title: "บทความที่ควรจะพังตอนบันทึก")
=> false
```

`Post.exists?` คืน `false` — ยืนยันว่า**ทั้งทรานแซกชันถูก rollback ทั้งหมด** ถึงแม้ `Post` เองจะ
ผ่าน validation ของตัวเองสบายๆ (title/body ถูกต้องครบ) แต่เพราะ nested `Comment` ที่แนบมาไม่ผ่าน
validation ทำให้ `@post.save` คืนค่า `false` และไม่มีอะไรถูกเขียนลงฐานข้อมูลเลยสักตาราง — ตรงตาม
เงื่อนไขข้อ 4 ที่โจทย์กำหนดไว้ทุกประการ (ทบทวนหลักการทรานแซกชันของ ActiveRecord validation จาก
**Part 026**)

`_form.html.erb` แสดง error ของ nested comment ได้ด้วยหรือไม่ — คำตอบคือ **ยังไม่ได้ในเวอร์ชัน
เฉลยนี้** เพราะ `post.errors.full_messages` แสดงเฉพาะ error message ของ `Post` เอง ส่วน error ของ
`Comment` ที่ซ้อนอยู่จะไปโผล่เป็น error แบบพิเศษที่ชื่อ `"Comments commenter can't be blank"`
(Rails รวม error ของ nested record เข้ามาใน `post.errors` ให้อัตโนมัติโดยเติม prefix ชื่อ
association ให้) ทดสอบตรวจสอบได้:

```irb
irb> post = Post.new(title: "x" * 5 + " test", body: "body ยาวพอสมควรครับ")
irb> post.comments.build(commenter: "", body: "no name")
irb> post.valid?
=> false
irb> post.errors.full_messages
=> ["Comments commenter can't be blank"]
```

ข้อความ `"Comments commenter can't be blank"` จะไปแสดงอยู่ใน `<div class="form-errors">` บนสุด
ของฟอร์มโดยอัตโนมัติ (เพราะ loop `post.errors.each` ที่เขียนไว้ครอบคลุมทุก error รวมถึงของ nested
record ด้วย) เพียงแต่ไม่ได้ไป highlight ที่ field ที่ผิดจริงๆ ในแถว nested นั้นโดยเฉพาะ — การทำ
error highlighting ที่แม่นยำระดับ per-nested-row เป็นเทคนิคที่ลึกกว่านี้อีกขั้น (ต้องเทียบ index
ของ error กับ index ของแถวใน `fields_for`) ซึ่งจะกลับมาขยายอีกครั้งใน **Part 037**

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มปุ่ม "เพิ่มความคิดเห็น" แบบ JavaScript-free** — ปัจจุบันฟอร์มเตรียมแถว comment ว่างไว้
   ตายตัว (3 แถวตอนสร้าง, 1 แถวตอนแก้ไข) ลองแก้ให้ผู้ใช้เพิ่มแถวได้เองแบบไม่จำกัดจำนวนโดยไม่ใช้
   JavaScript เลย (ใบ้: ใช้ query parameter เช่น `?comment_rows=5` ที่ controller อ่านค่าไปกำหนด
   จำนวนครั้งที่ `.build` แล้วทำปุ่ม "เพิ่มอีก 1 แถว" เป็นลิงก์ที่เพิ่มค่านี้แล้ว reload หน้าใหม่
   — วิธีนี้ไม่สวยเท่าการเพิ่มด้วย JavaScript/Stimulus แต่ทำงานได้แน่นอน 100% โดยไม่ต้องรอถึง
   Phase 7)
2. **ทำ error highlighting ระดับ per-nested-row** — แก้ partial ให้ error ของ comment แต่ละแถว
   (เช่น "Commenter can't be blank") ไปแสดงอยู่**ใต้ field ของแถวนั้นโดยตรง** แทนที่จะไปรวมอยู่ที่
   กล่อง error บนสุดของฟอร์มเหมือนตอนนี้ (ใบ้: `cf.object.errors[:commenter]` เข้าถึง error ของ
   object แต่ละตัวใน `fields_for` block ได้โดยตรง ไม่ต้องพึ่ง `post.errors` ที่รวมทุกอย่างไว้
   ด้วยกัน)
3. **เพิ่ม CRUD เต็มรูปแบบให้ `Category`** — ตอนนี้ Category สร้างผ่าน `db/seeds.rb`/console
   เท่านั้น ลองเพิ่ม `CategoriesController` พร้อม view ครบ (index/new/create/edit/update/
   destroy) ที่ใช้ `form_with`/strong parameters ตามหลักการทั้งหมดที่เรียนใน Part นี้ — ระวัง
   เรื่อง `destroy` ของ Category ที่ยังมี Post ผูกอยู่ (ต้องตัดสินใจว่าจะทำอย่างไร: ป้องกันไม่ให้
   ลบ, ตั้ง `category_id` ของ Post ที่เกี่ยวข้องเป็น `nil`, หรือ `dependent: :destroy` ลบ Post
   ตามไปด้วย — แต่ละทางเลือกมีข้อดี/ข้อเสียต่างกัน ลองวิเคราะห์เอง)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจความต่างระหว่าง model-backed form (`form_with model:`) กับ url-backed form
  (`form_with url:`) และรู้จัก `scope:` สำหรับตั้ง prefix ของ field name เอง
- แก้ความเข้าใจผิดที่พบบ่อยเรื่อง `local:`/`remote:` — รู้ว่ามันเป็นกลไกเก่าของ rails-ujs ที่ไม่มี
  ผลจริงในทางเทคนิคอีกต่อไปตั้งแต่ Rails 6.1 และรู้ว่าตัวควบคุม Turbo Drive ตัวจริงคือ
  `data: { turbo: false }` ไม่ใช่ `local:`
- เข้าใจกลไกที่ `form_with(model:)` เดา URL และ HTTP verb จาก `persisted?` และเห็น `_method`
  hidden field ที่ทำให้ HTML form ธรรมดาส่ง `PATCH` ได้
- ใช้ form builder helper ครบทั้ง 7 แบบพร้อมเห็น HTML จริงที่ render ออกมา:
  `text_field`/`text_area`, `select`, `collection_select`, `radio_button`, `check_box`,
  `date_select`
- เข้าใจกลไก `field_error_proc` ที่ Rails ห่อฟิลด์ที่ error ด้วย `<div class="field_with_errors">`
  ให้อัตโนมัติ และรู้วิธีปิด/ปรับแต่งพฤติกรรมนี้
- เข้าใจความต่างระหว่าง `require.permit` แบบดั้งเดิมกับ `params.expect` ของ Rails 8 อย่างลึกซึ้ง
  — พิสูจน์ด้วย HTTP จริงว่า `require.permit` เจอ params ผิดรูปแบบแล้วได้ 500 ส่วน `expect` ได้
  400 ที่ถูกต้องกว่า
- permit array ของ scalar (`tags: []`) และ array ของ nested hash (`comments_attributes:
  [[...]]`) ได้ถูกต้อง พร้อมรู้จักกับดักสำคัญที่ `expect` ต้องใช้ double-bracket สำหรับ nested
  attributes เสมอ (ต่างจาก `permit` ที่ใช้ single-bracket ได้)
- ประกาศ `accepts_nested_attributes_for` พร้อม `allow_destroy:`/`reject_if:` ฝั่ง Model และ
  ใช้ `fields_for` เรนเดอร์ฟอร์มของ record ลูกซ้อนอยู่ในฟอร์มของ record แม่
- ใช้ `_destroy` ลบ record ลูกผ่านฟอร์มของ record แม่ได้ รวมถึงผสมการสร้าง/แก้ไข/ลบพร้อมกันใน
  คำขอเดียว และเห็นว่า validation ที่ล้มเหลวใน nested record ทำให้ทั้งทรานแซกชัน rollback ทั้งหมด
- เข้าใจที่มาของ strong parameters ในฐานะเครื่องมือป้องกัน mass assignment vulnerability และรู้ว่า
  เรื่องนี้จะกลับมาเจาะลึกอีกครั้งใน Phase 13 (Part 079–080)

**ต่อไป (Part 032):** เราจะกลับไปที่ Model layer อีกครั้งเพื่อเจาะลึก **Validation ขั้นสูง** —
การเขียน **custom validator** ของตัวเอง (ทั้งแบบ inline ด้วย `validate :method_name` และแบบ
class แยกต่างหากด้วย `ActiveModel::Validator`), **conditional validation** (`if:`/`unless:` ที่
ควบคุมว่า validation รันเมื่อไหร่), และ **uniqueness validation แบบมี scope** (เช่น "ชื่อบทความ
ต้องไม่ซ้ำ แต่ซ้ำกันได้ถ้าอยู่คนละหมวดหมู่") — เทคนิคเหล่านี้จะทำให้ Model ของ Mini Blog
รัดกุมและสมจริงมากขึ้นไปอีกขั้น ก่อนจะไปถึง Association ขั้นสูง (`has_many :through`,
polymorphic) ใน Part 033 และเดินทางต่อไปจนถึงโปรเจกต์รวบยอดของ Phase 4 ใน Part 040
