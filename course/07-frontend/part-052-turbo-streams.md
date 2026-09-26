# Part 052: Turbo Streams — Real-time Update แบบไม่ใช้ JavaScript เยอะ

> **Step ครอบคลุมใน Part นี้:** Step 511–520
> **ระดับ:** ปานกลาง–สูง (ควรผ่าน Part 051 เรื่อง Turbo Drive และ Turbo Frame มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, `turbo-rails` 2.0.x, ActionCable (adapter `async`/`solid_cable`/Redis)

Part ที่แล้วเราเรียนเรื่อง **Turbo Drive** (เร่งความเร็วการนำทางทั้งหน้า) และ **Turbo Frame**
(อัปเดตเฉพาะบางส่วนของหน้าโดยไม่โหลดทั้งหน้า แต่ยังต้องเป็น "คนเดียวกันที่กด" ถึงจะเห็นผล)
Part นี้เราจะไปอีกขั้นด้วย **Turbo Stream** — กลไกที่ทำให้ Rails ส่ง "คำสั่งแก้ไข DOM" ไปยัง
browser ได้ ทั้งแบบตอบกลับ request ปกติ และแบบ **push แบบ real-time ผ่าน ActionCable** ไปยัง
ผู้ใช้คนอื่นที่กำลังเปิดหน้าเดียวกันอยู่ — โดยที่เราแทบไม่ต้องเขียน JavaScript เองเลย

## สารบัญของ Part นี้

- Step 511: Turbo Stream คืออะไร — กายวิภาคของ `<turbo-stream>` และ 7 actions
- Step 512: รูปแบบคลาสสิก — Controller เดียวตอบทั้ง `format.html` และ `format.turbo_stream`
- Step 513: การเขียน `.turbo_stream.erb` view template แยกไฟล์
- Step 514: `turbo_stream.append`/`.replace` helper และการส่งหลาย stream พร้อมกัน
- Step 515: ActionCable เบื้องหลัง Turbo Streams — `turbo_stream_from` เพื่อ subscribe
- Step 516: Broadcast ข้ามหน้าเว็บแบบ real-time — `broadcast_append_to`/`broadcast_replace_to`/`broadcast_remove_to`
- Step 517: `broadcasts_to` บรรทัดเดียว + `after_create_commit`/`after_update_commit`/`after_destroy_commit`
- Step 518: Multiple targets และ `turbo_stream.action` สำหรับ custom action
- Step 519: ผสาน Turbo Streams กับ Background Job + ข้อควรระวังด้านความปลอดภัย
- Step 520: แบบฝึกหัด — ระบบคอมเมนต์ real-time บน Post

> **หมายเหตุเรื่องสภาพแวดล้อมที่ใช้ทดสอบ:** ทุกตัวอย่างโค้ดใน Part นี้ทดสอบจริงบน Rails app
> เปล่าที่สร้างด้วย `rails new` (Ruby 3.3.6 / Rails 8.1.4 / turbo-rails 2.0.23) มี scaffold ของ
> `Post` และ `Comment` (มี `belongs_to :post`) แล้วยิง request ด้วย `curl` จริงพร้อมดู
> `development.log` เพื่อยืนยันว่า HTML/Content-Type/ActionCable broadcast ที่ได้ตรงกับที่อธิบาย
> ทุกตัวอย่าง `curl` และ log ในเอกสารนี้คือผลลัพธ์จริงที่รันได้ ไม่ใช่การจำลอง

---

## Step 511: Turbo Stream คืออะไร — กายวิภาคของ `<turbo-stream>` และ 7 actions

**Turbo Frame** (Part 051) ทำให้เราอัปเดต "กรอบ" หนึ่งกรอบในหน้าได้โดยไม่โหลดทั้งหน้า แต่มี
ข้อจำกัดคือ ต้องมี `id` ของ frame ตรงกันระหว่าง request กับ response และแก้ไขได้ทีละกรอบ
เดียวเท่านั้นต่อหนึ่ง response

**Turbo Stream** แก้ปัญหานี้ด้วยแนวคิดใหม่: แทนที่จะส่ง HTML เต็มหน้าหรือ HTML ของ frame เดียว
กลับมา เราส่ง **คำสั่งแก้ไข DOM** กลับมาเป็นชุด (จะกี่คำสั่งก็ได้ในหนึ่ง response) โดยห่อด้วย
custom element ชื่อ `<turbo-stream>`

### กายวิภาคของ `<turbo-stream>`

```html
<turbo-stream action="replace" target="comment_5">
  <template>
    <div id="comment_5" class="comment">
      <p><strong>สมชาย</strong>: แก้ไขข้อความคอมเมนต์แล้ว</p>
    </div>
  </template>
</turbo-stream>
```

ส่วนประกอบสำคัญ:

- **`action`** — บอกว่าจะทำอะไรกับ DOM ปลายทาง มีทั้งหมด 7 แบบ (อธิบายด้านล่าง)
- **`target`** — `id` ของ element ปลายทางใน DOM ปัจจุบันของ browser (หรือ `targets` เป็น
  CSS selector ถ้าต้องการแก้หลาย element พร้อมกัน — ดู Step 518)
- **`<template>`** — ห่อ HTML fragment ที่จะถูกใช้ Turbo จะ "ดึง" เนื้อหาข้างในออกมาแล้วเอาไป
  แทรก/แทนที่ตาม `action` ที่ระบุ (เนื้อหาใน `<template>` จะไม่ถูก render โดย browser ตรงๆ
  จนกว่า Turbo JavaScript จะย้ายมันออกมา — เป็นพฤติกรรมมาตรฐานของ HTML `<template>` tag)

จุดสำคัญที่ต้องเข้าใจ: **Turbo Stream ไม่ใช่การ replace ทั้งหน้า** เหมือน AJAX ทั่วไปที่ต้องเขียน
`fetch()` แล้วเอาผลลัพธ์ไป manipulate DOM เอง — ฝั่ง client มี JavaScript ของ Turbo (มาพร้อมกับ
`turbo-rails` gem ผ่าน `@hotwired/turbo-rails`) คอยดัก response ที่มี Content-Type
`text/vnd.turbo-stream.html` แล้วประมวลผล `<turbo-stream>` element ให้อัตโนมัติทั้งหมด

### 7 actions ของ Turbo Stream

| Action | ทำอะไร | ต้องมี `<template>` หรือไม่ |
|--------|--------|---------------------------|
| `append` | แทรกเนื้อหาต่อท้าย (ลูกคนสุดท้าย) ของ target | ต้องมี |
| `prepend` | แทรกเนื้อหาไว้ก่อนลูกคนแรกของ target | ต้องมี |
| `replace` | แทนที่ target element ทั้งตัว (เหมือน `outerHTML =`) | ต้องมี |
| `update` | แทนที่แค่เนื้อหาข้างใน target (เหมือน `innerHTML =`) target element เดิมยังอยู่ | ต้องมี |
| `remove` | ลบ target element ออกจาก DOM ทั้งตัว | ไม่ต้องมี |
| `before` | แทรกเนื้อหาเป็น sibling ก่อนหน้า target | ต้องมี |
| `after` | แทรกเนื้อหาเป็น sibling ถัดจาก target | ต้องมี |

เพื่อให้เห็นภาพจริง เราลองสร้าง action ชื่อ `demo#all_actions` ที่ยิงทั้ง 7 actions พร้อมกันใน
หนึ่ง response (โค้ดควบคุมฝั่ง controller):

```ruby
# app/controllers/demo_controller.rb (โค้ดทดลองเพื่อดูผลลัพธ์ทั้ง 7 actions)
class DemoController < ApplicationController
  def all_actions
    comment = Comment.first
    render turbo_stream: [
      turbo_stream.append("comments", "<div>append</div>"),
      turbo_stream.prepend("comments", "<div>prepend</div>"),
      turbo_stream.replace(comment, "<div>replace</div>"),
      turbo_stream.update("comments", "<div>update</div>"),
      turbo_stream.before(comment, "<div>before</div>"),
      turbo_stream.after(comment, "<div>after</div>"),
      turbo_stream.remove("some_id"),
    ]
  end
end
```

ยิง `curl` เข้า endpoint นี้จริง (พร้อม header `Accept: text/vnd.turbo-stream.html`) ได้ผลลัพธ์
ตรงตามที่คาด:

```bash
curl -s -H "Accept: text/vnd.turbo-stream.html" http://127.0.0.1:3000/demo/all_actions
```

```html
<turbo-stream action="append" target="comments"><template><div>append</div></template></turbo-stream>
<turbo-stream action="prepend" target="comments"><template><div>prepend</div></template></turbo-stream>
<turbo-stream action="replace" target="comment_3"><template><div>replace</div></template></turbo-stream>
<turbo-stream action="update" target="comments"><template><div>update</div></template></turbo-stream>
<turbo-stream action="before" target="comment_3"><template><div>before</div></template></turbo-stream>
<turbo-stream action="after" target="comment_3"><template><div>after</div></template></turbo-stream>
<turbo-stream action="remove" target="some_id"></turbo-stream>
```

สังเกตว่า `turbo_stream.replace(comment, ...)` และ `turbo_stream.before(comment, ...)` แปลง
object `comment` เป็น `target="comment_3"` ให้อัตโนมัติ — นี่คือกลไก `dom_id(comment)` ของ Rails
(มาจาก `ActionView::RecordIdentifier`) ที่แปลง ActiveRecord object เป็น string อย่าง
`"comment_3"` ให้เอง ไม่ต้องเขียน `dom_id(comment)` มือ

> **แนวคิดสำคัญ:** เพื่อให้ `turbo_stream.replace(comment, ...)`/`turbo_stream.update(comment, ...)`
> ทำงานได้ HTML fragment ปลายทางต้อง**มี `id="comment_3"` ตรงกับ `dom_id(comment)`ด้วยเสมอ** —
> นี่คือเหตุผลที่ partial `_comment.html.erb` ของเราต้องมี `<div id="<%= dom_id(comment) %>">`
> ครอบไว้ ถ้า id ไม่ตรงกัน Turbo จะหา target ไม่เจอและไม่ทำอะไรเลย (เงียบๆ ไม่มี error ให้เห็นด้วย
> ซึ่งเป็นบั๊กที่พบบ่อยที่สุดตอนเริ่มใช้ Turbo Stream)

---

## Step 512: รูปแบบคลาสสิก — Controller เดียวตอบทั้ง `format.html` และ `format.turbo_stream`

รูปแบบที่ใช้บ่อยที่สุดในโปรเจกต์จริงคือ controller action เดียวตอบได้ทั้งสองแบบ:

1. ถ้า request มาจาก Turbo (ฟอร์ม/ลิงก์ปกติที่ไม่ได้ปิด Turbo) → ตอบ **Turbo Stream** อัปเดต
   เฉพาะจุดที่จำเป็น
2. ถ้า request มาจาก client ที่ไม่รองรับ Turbo (เช่น JavaScript ปิดอยู่, เป็น API client, หรือ
   fallback เมื่อ JS โหลดไม่ทัน) → ตอบ **HTML เต็มหน้าแบบ redirect** เหมือนเดิม

Rails ตรวจจับ format จาก HTTP header `Accept` ที่ Turbo ใส่มาให้อัตโนมัติเวลา submit ฟอร์ม
(`text/vnd.turbo-stream.html, text/html, application/xhtml+xml`) โดยเราไม่ต้องเขียน JavaScript
เพิ่มเลย — แค่ฟอร์มเป็น `form_with` ปกติ (ซึ่ง default คือ Turbo-enabled อยู่แล้ว)

### ตัวอย่างจริง: `CommentsController`

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_post

  def create
    @comment = @post.comments.build(comment_params)

    respond_to do |format|
      if @comment.save
        format.turbo_stream
        format.html { redirect_to @post, notice: "เพิ่มคอมเมนต์แล้ว" }
      else
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_comment",
            partial: "comments/form",
            locals: { post: @post, comment: @comment }
          )
        end
        format.html { redirect_to @post, alert: "เพิ่มคอมเมนต์ไม่สำเร็จ" }
      end
    end
  end

  def destroy
    @comment = @post.comments.find(params[:id])
    @comment.destroy

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @post, notice: "ลบคอมเมนต์แล้ว" }
    end
  end

  private

  def set_post
    @post = Post.find(params[:post_id])
  end

  def comment_params
    params.require(:comment).permit(:body, :author_name)
  end
end
```

สังเกตว่า `format.turbo_stream` (บรรทัดกรณีสำเร็จ) **ไม่มี block** — เมื่อไม่มี block Rails จะ
มองหา view template ชื่อ `create.turbo_stream.erb` มา render ให้อัตโนมัติ (implicit render
เหมือนที่ `format.html` มองหา `create.html.erb` ถ้าไม่ redirect) เราจะเขียนไฟล์นั้นใน Step 513

### พิสูจน์ว่า controller เดียวตอบสองแบบได้จริง

ยิง request แบบ `Accept: text/html` (จำลอง client ที่ไม่ใช้ Turbo):

```bash
curl -s -D - -o /dev/null http://127.0.0.1:3000/posts/1/comments -X POST \
  -H "Accept: text/html" \
  -b cookies.txt -c cookies.txt \
  -H "X-CSRF-Token: $TOKEN" \
  -F "comment[author_name]=Dave" -F "comment[body]=html fallback test"
```

ผลลัพธ์จริง — ได้ full-page redirect ตามปกติ:

```
HTTP/1.1 302 Found
content-type: text/html; charset=utf-8
location: http://127.0.0.1:3000/posts/1
```

ยิง request เดิม แต่เปลี่ยนเป็น `Accept: text/vnd.turbo-stream.html` (จำลองสิ่งที่ Turbo ทำให้
อัตโนมัติเวลา submit ฟอร์มจริงในเบราว์เซอร์):

```bash
curl -s -D - http://127.0.0.1:3000/posts/1/comments -X POST \
  -H "Accept: text/vnd.turbo-stream.html, text/html, application/xhtml+xml" \
  -b cookies.txt -c cookies.txt \
  -H "X-CSRF-Token: $TOKEN" \
  -F "comment[author_name]=Alice" -F "comment[body]=สวัสดีครับ นี่คือคอมเมนต์แรก"
```

ผลลัพธ์จริง — ได้ `200 OK` พร้อม Content-Type พิเศษและ body เป็น `<turbo-stream>`:

```
HTTP/1.1 200 OK
content-type: text/vnd.turbo-stream.html; charset=utf-8
vary: Accept
```

และใน `log/development.log` เราเห็นบรรทัดยืนยันว่า Rails ตรวจจับ format ถูกต้อง:

```
Processing by CommentsController#create as TURBO_STREAM
  Parameters: {"comment"=>{"author_name"=>"Alice", "body"=>"..."}, "post_id"=>"1"}
```

`as TURBO_STREAM` คือคำตอบว่า `respond_to` เลือกกิ่ง `format.turbo_stream` เพราะ header
`Accept` ที่ส่งมา — controller เดียว โค้ดเดียว ตอบได้สองแบบ ขึ้นอยู่กับว่าใครเป็นคนเรียก

---

## Step 513: การเขียน `.turbo_stream.erb` view template แยกไฟล์

เมื่อ `format.turbo_stream` ไม่มี block เราต้องสร้างไฟล์ view ให้ตรงชื่อ action โดยใช้
นามสกุล `.turbo_stream.erb` (คู่ขนานกับ `.html.erb`)

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", partial: "comments/comment", locals: { comment: @comment } %>
<%= turbo_stream.replace "new_comment", partial: "comments/form", locals: { post: @post, comment: Comment.new } %>
```

ไฟล์นี้ทำสองอย่างพร้อมกันในหนึ่ง response:

1. **`append` เข้า `#comments`** — เอา partial ของคอมเมนต์ใหม่ไปต่อท้ายรายการคอมเมนต์เดิม
2. **`replace` ที่ `#new_comment`** — เอาฟอร์มเปล่า (`Comment.new`) ไปแทนที่ฟอร์มเดิม ทำให้ฟอร์ม
   เคลียร์ค่าที่กรอกไปแล้วโดยอัตโนมัติ (ไม่ต้องเขียน JavaScript ล้างฟอร์มเอง)

ไฟล์สำหรับ action `destroy`:

```erb
<%# app/views/comments/destroy.turbo_stream.erb %>
<%= turbo_stream.remove @comment %>
```

### พิสูจน์ด้วย curl จริง

ยิง POST สร้างคอมเมนต์ด้วย `Accept: text/vnd.turbo-stream.html` ได้ body ดังนี้ (ตัดบางส่วน
เพื่อความกระชับ แต่เป็นผลลัพธ์จริงทั้งหมด):

```html
<turbo-stream action="append" target="comments"><template><!-- BEGIN app/views/comments/_comment.html.erb
--><turbo-frame id="comment_1">
  <div id="comment_1" class="comment">
    <p><strong>Alice</strong>: สวัสดีครับ นี่คือคอมเมนต์แรก</p>
    <form data-turbo-confirm="แน่ใจนะ?" class="button_to" method="post" action="/posts/1/comments/1">...</form>
  </div>
</turbo-frame><!-- END app/views/comments/_comment.html.erb --></template></turbo-stream>
<turbo-stream action="replace" target="new_comment"><template><!-- BEGIN app/views/comments/_form.html.erb
--><turbo-frame id="new_comment">
  <form action="/posts/1/comments" accept-charset="UTF-8" method="post">...
    <textarea name="comment[body]" id="comment_body">
</textarea>
    ...
</form></turbo-frame><!-- END app/views/comments/_form.html.erb --></template></turbo-stream>
```

และยิง `DELETE /posts/1/comments/1` ด้วย Accept header เดียวกัน:

```html
<turbo-stream action="remove" target="comment_1"></turbo-stream>
```

ตรงตามที่ view template ทั้งสองไฟล์กำหนดไว้ทุกประการ — Content-Type ของ response ทั้งสองคือ
`text/vnd.turbo-stream.html; charset=utf-8` เหมือนกัน

---

## Step 514: `turbo_stream.append`/`.replace` helper และการส่งหลาย stream พร้อมกัน

`turbo_stream` เป็น **builder object** (มาจาก `Turbo::Streams::TagBuilder`) ที่ใช้งานได้ทั้งใน
view template (อย่างที่เห็นใน Step 513) และใน controller โดยตรงผ่าน `render turbo_stream:`

method หลักๆ ที่มีให้ใช้ (คู่กับ 7 actions ใน Step 511):

```ruby
turbo_stream.append(target, content = nil, **rendering, &block)
turbo_stream.prepend(target, content = nil, **rendering, &block)
turbo_stream.replace(target, content = nil, **rendering, &block)
turbo_stream.update(target, content = nil, **rendering, &block)
turbo_stream.remove(target)
turbo_stream.before(target, content = nil, **rendering, &block)
turbo_stream.after(target, content = nil, **rendering, &block)
```

`target` รับได้ทั้ง string id (`"comments"`) หรือ ActiveRecord object (`comment` → แปลงเป็น
`dom_id(comment)` ให้อัตโนมัติ) ส่วน `content`/`partial`/`locals` ก็ยืดหยุ่นเหมือน `render` ปกติ
— จะส่ง string HTML ตรงๆ, `partial:`, หรือ block ก็ได้

### ส่งหลาย stream ในหนึ่ง response จาก controller โดยตรง

ไม่จำเป็นต้องพึ่งไฟล์ `.turbo_stream.erb` เสมอไป — ถ้า logic ไม่ซับซ้อน เขียนตรงใน controller
ด้วย array ได้เลย:

```ruby
def multi_target
  render turbo_stream: turbo_stream.replace_all(".comment", partial: "comments/comment", locals: { comment: Comment.first })
end
```

ทดสอบจริง:

```bash
curl -s -H "Accept: text/vnd.turbo-stream.html" http://127.0.0.1:3000/demo/multi_target
```

```html
<turbo-stream action="replace" targets=".comment"><template>...</template></turbo-stream>
```

(รายละเอียดเรื่อง `targets` แบบ CSS selector อธิบายเต็มๆ ใน Step 518)

> **เทคนิค:** เวลาต้อง render partial เดียวกันหลายจุด (เช่น อัปเดตทั้ง `#comments` list และ
> ป้ายนับจำนวน `#comment_count`) การส่งเป็น array `render turbo_stream: [stream1, stream2]`
> อ่านง่ายกว่าการสร้างไฟล์ `.turbo_stream.erb` ที่มีแค่ 2 บรรทัด แต่ถ้า logic การ render ซับซ้อน
> ขึ้น (มี conditional หลายจุด) การแยกเป็นไฟล์ `.turbo_stream.erb` จะดูแลรักษาง่ายกว่า

---

## Step 515: ActionCable เบื้องหลัง Turbo Streams — `turbo_stream_from` เพื่อ subscribe

ทุกอย่างที่ทำมาใน Step 512–514 เป็น "**Turbo Stream แบบตอบกลับ request**" — คนที่กด submit
ฟอร์มเท่านั้นที่เห็นผลลัพธ์ ถ้ามีคนอื่นเปิดหน้าเดียวกันอยู่ในเบราว์เซอร์อีกเครื่อง เขาจะไม่เห็น
คอมเมนต์ใหม่จนกว่าจะกด refresh เอง

นี่คือจุดที่ **ActionCable** (ระบบ WebSocket ของ Rails) เข้ามาเสริม `turbo-rails` มี channel
สำเร็จรูปชื่อ `Turbo::StreamsChannel` ให้ใช้งานได้ทันทีโดยไม่ต้องเขียน channel เอง

### `turbo_stream_from` — เปิดการ subscribe จากฝั่ง view

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_stream_from @post %>
```

เท่านี้พอ! บรรทัดนี้ render ออกมาเป็น custom element:

```html
<turbo-cable-stream-source channel="Turbo::StreamsChannel"
  signed-stream-name="IloybGtPaTh2ZEhOeE4zZ3laR1Z0YnpBMU1pOVFiM04wTHpFIg==--579cb36f19f6...">
</turbo-cable-stream-source>
```

(นี่คือ HTML จริงที่ยืนยันด้วย `curl` จริงกับหน้า `/posts/1`) เมื่อ browser render element นี้
JavaScript ของ Turbo จะเปิด WebSocket connection ไปยัง ActionCable และ subscribe เข้า channel
`Turbo::StreamsChannel` โดยอัตโนมัติ — ไม่ต้องเขียน `consumer.js` หรือ Stimulus controller เอง
เลยสักบรรทัด

### `signed-stream-name` คืออะไร

ค่านี้ไม่ใช่ `"post_1"` ตรงๆ แต่เป็นค่าที่ **เซ็นชื่อ (signed)** ด้วย Rails' message verifier
เพื่อไม่ให้ client ปลอมแปลงหรือเดา stream name ของ record อื่นได้ พิสูจน์ได้จาก `rails console`:

```ruby
post = Post.find(1)
Turbo::StreamsChannel.signed_stream_name(post)
# => "IloybGtPaTh2ZEhOeE4zZ3laR1Z0YnpBMU1pOVFiM04wTHpFIg==--579cb36f19f6fcd834dd9479af0..."

# ใส่ array เพื่อสร้าง stream ที่ scope แคบลงกว่าแค่ post เฉยๆ ได้ เช่น [post, "posts"]
Turbo::StreamsChannel.signed_stream_name([post, "posts"])
# => "IloybGtPaTh2ZEhOeE4zZ3laR1Z0YnpBMU1pOVFiM04wTHpFOnBvc3RzIg==--17ce8d8e2d6e740ab6ec..."
```

สองสาย signature นี้ต่างกัน เพราะ **stream ที่ scope ต่างกันคือ "ช่องสัญญาณ" คนละช่องกัน** —
ใครสมัคร (subscribe) รับได้เฉพาะข้อความที่ broadcast เข้าตรงช่องของตัวเองเท่านั้น ค่า signed
string นี้เปลี่ยนทุกครั้งที่ `secret_key_base` เปลี่ยน (เช่นข้าม environment) จึงปลอมแปลงข้าม
แอปหรือข้าม record ไม่ได้

> **ทำไมต้อง sign:** ถ้าใช้ string ตรงๆ อย่าง `"post_1"` เป็นชื่อ stream ผู้ใช้ที่เปิด dev tools
> จะเดาได้ทันทีว่าถ้าเปลี่ยนเป็น `"post_2"` จะได้ยิน broadcast ของโพสต์อื่น (ที่อาจเป็นข้อมูล
> private) การ sign ทำให้ต่อให้เห็นค่าที่ render ออกมาในหน้า HTML ก็เอาไปดัดแปลงเป็นของ record
> อื่นไม่ได้ (แต่ก็ยังมีเรื่องต้องระวังเพิ่มอยู่ดี จะอธิบายลึกใน Step 519)

---

## Step 516: Broadcast ข้ามหน้าเว็บแบบ Real-time — ฟีเจอร์เด่นของบทนี้

นี่คือหัวใจของ Turbo Stream: การ **broadcast** ทำให้ทุกคนที่ subscribe อยู่ (ผ่าน
`turbo_stream_from`) ได้รับ `<turbo-stream>` เดียวกัน "โดยที่ไม่ต้องกด submit อะไรเลย" — เพียงแค่
เปิดหน้าทิ้งไว้เฉยๆ

สถานการณ์: **User A** โพสต์คอมเมนต์ใหม่ → **User B** ที่เปิดหน้า post เดียวกันอยู่ (คนละ
browser/tab/เครื่อง) ต้องเห็นคอมเมนต์นั้นปรากฏขึ้นทันทีโดยไม่ต้อง refresh

### เรียก broadcast จาก Model callback

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post

  after_create_commit -> {
    broadcast_append_to post, target: "comments", partial: "comments/comment", locals: { comment: self }
  }
  after_update_commit -> {
    broadcast_replace_to post, target: self, partial: "comments/comment", locals: { comment: self }
  }
  after_destroy_commit -> {
    broadcast_remove_to post, target: self
  }
end
```

จุดสำคัญ:

- `broadcast_append_to post, ...` — คำว่า `post` ตัวแรกคือ **stream ปลายทาง** ต้องตรงกับสิ่งที่
  ใช้ตอน subscribe (`turbo_stream_from @post` ใน Step 515) มิฉะนั้นจะ broadcast ไปยัง "ช่อง" ที่
  ไม่มีใคร subscribe อยู่เลย (ส่งไปแต่ไม่มีใครได้ยิน ไม่ error แต่ก็ไม่มีอะไรเกิดขึ้น)
- ใช้ `after_*_commit` (ไม่ใช่ `after_create`/`after_update` เฉยๆ) เพราะเราต้องให้แน่ใจว่า
  transaction ของฐานข้อมูล commit สำเร็จแล้วจริงๆ ก่อนจะ broadcast — ถ้า broadcast ใน
  `after_create` ธรรมดาแล้ว transaction ถูก rollback ทีหลัง (เช่นเพราะ callback อื่นล้มเหลว)
  ผู้ใช้จะเห็นคอมเมนต์ที่ไม่มีอยู่จริงในฐานข้อมูล
- `partial:`/`locals:` เหมือนกับที่ใช้ใน `turbo_stream.append` ทุกประการ — ต่างกันตรงที่
  `broadcast_append_to` **render และส่งออกไปทันที** ไม่ต้องรอ controller ตอบ response ก่อน

### พิสูจน์ว่า broadcast เกิดขึ้นจริง (โดยไม่ต้องเปิดสองแท็บ)

ในสภาพแวดล้อมนี้เราตรวจสอบผ่าน `rails console` + `development.log` แทนการเปิด 2 browser tab
(ซึ่งให้ผลลัพธ์เดียวกันในเชิงเทคนิค เพราะ Turbo ฝั่ง client แค่ดัก message จาก WebSocket แล้ว
ประมวลผล `<turbo-stream>` — ทำงานเหมือนกันไม่ว่าจะมาจาก HTTP response หรือจาก ActionCable):

```ruby
# รันผ่าน `rails runner` หรือ `rails console`
post = Post.first
comment = post.comments.create!(author_name: "Bob", body: "คอมเมนต์จาก console")
```

ผลลัพธ์จริงใน `log/development.log` (ไม่ได้มาจากการยิง HTTP request ใดๆ เลย — มาจากการสร้าง
record ตรงๆ ใน console คนละ process กับ web server ที่รันอยู่):

```
[ActionCable] Broadcasting to Z2lkOi8vdHNxN3gyZGVtbzA1Mi9Qb3N0LzE:
  "<turbo-stream action=\"append\" target=\"comments\"><template><!-- BEGIN app/views/comments/_comment.html.erb
  --><turbo-frame id=\"comment_2\">
    <div id=\"comment_2\" class=\"comment\">
      <p><strong>Bob</strong>: คอมเมนต์จาก console</p>
      ...
```

`Z2lkOi8vdHNxN3gyZGVtbzA1Mi9Qb3N0LzE` คือ Base64 ของ `signed_stream_name(post)` ตัวเดียวกับที่
`turbo_stream_from @post` ใช้ subscribe ไว้ — พิสูจน์ว่าถ้ามี browser จริงเปิดหน้า `/posts/1`
ค้างอยู่ (เรียก subscribe ผ่าน channel นี้ไปแล้ว) ข้อความนี้จะถูกส่งไปหาและ Turbo JS ฝั่ง client
จะ append `<div id="comment_2">...</div>` เข้า `#comments` ให้อัตโนมัติ **โดยที่ browser นั้นไม่ได้
เป็นคนกด submit ฟอร์มเลย**

ทดสอบต่อด้วยการ update และ destroy comment เดียวกัน ก็เห็น broadcast คู่กับ action ที่ถูกต้อง:

```
[ActionCable] Broadcasting to Z2lkOi8v...: "<turbo-stream action=\"replace\" target=\"comment_2\">...
[ActionCable] Broadcasting to Z2lkOi8v...: "<turbo-stream action=\"remove\" target=\"comment_2\"></turbo-stream>"
```

> **ทำไม controller ยังคง render `format.turbo_stream` เองด้วย (Step 512–513) ทั้งที่มี
> broadcast แล้ว:** ผู้ใช้ที่กด submit ฟอร์มควรเห็นผลลัพธ์ **ทันที** โดยไม่ต้องรอ WebSocket
> roundtrip (แม้จะเร็วมากแต่ก็มี latency เพิ่มเล็กน้อยเทียบกับ response ของ request ตัวเอง) จึง
> เป็นธรรมเนียมปฏิบัติทั่วไปที่จะ **ตอบ Turbo Stream ให้คนกดเองผ่าน response ปกติ (Step 512-514)
> และ broadcast แยกไปให้ "คนอื่นๆ" ผ่าน ActionCable (Step 516)** — สองกลไกนี้ทำงานคู่กัน ไม่ใช่
> แทนที่กัน

---

## Step 517: `broadcasts_to` บรรทัดเดียว + `after_*_commit` อัตโนมัติ

การเขียน `after_create_commit`/`after_update_commit`/`after_destroy_commit` ครบ 3 บรรทัดแบบ
Step 516 ใช้บ่อยมากจน Rails มี macro สำเร็จรูปให้เขียนบรรทัดเดียวจบ สำหรับกรณีที่ต้องการ
"sync สถานะของ record หนึ่งตัวให้ทุกคนที่ดูอยู่เห็นล่าสุดเสมอ" (ต่างจาก Step 516 ที่เป็นการ
เพิ่ม/ลบ item ในลิสต์ของอีก record หนึ่ง)

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy

  broadcasts_to ->(post) { [post, "posts"] }
end
```

`broadcasts_to` เขียนแทน:

- `after_create_commit` → broadcast `prepend`/`append` เข้า stream ที่ระบุ
- `after_update_commit` → broadcast `replace` element ที่มี `id="post_1"` (dom_id ของ post เอง)
- `after_destroy_commit` → broadcast `remove` element `id="post_1"`

โดยจะ render partial `posts/_post` (ตาม convention เดียวกับที่ `render @post` ใช้) ให้อัตโนมัติ
ไม่ต้องระบุ `partial:`/`locals:` เอง

### ข้อแตกต่างสำคัญที่ยืนยันได้จริง: `broadcasts_to` ทำงานผ่าน ActiveJob (async)

ทดสอบ update post ผ่าน console:

```ruby
Post.first.update!(title: "Hello (updated)")
```

log ที่ได้ **ไม่ใช่** `[ActionCable] Broadcasting to ...` ทันทีแบบ Step 516 แต่เป็น:

```
[ActiveJob] Enqueued Turbo::Streams::ActionBroadcastJob (Job ID: b4e32da9-...) to Async(default)
  with arguments: "Z2lkOi8vdHNxN3gyZGVtbzA1Mi9Qb3N0LzE:posts",
  {:action=>:replace, :target=>"post_1", :targets=>nil, :attributes=>{}, :locals=>{...}, :partial=>"posts/post"}
```

นี่คือความแตกต่างเชิงพฤติกรรมที่สำคัญมาก:

| | `broadcast_append_to`/`broadcast_replace_to`/`broadcast_remove_to` (Step 516) | `broadcasts_to` macro (Step 517) |
|---|---|---|
| การทำงาน | **synchronous** — render + ส่งออก ActionCable ทันทีใน request/callback thread เดียวกัน | **asynchronous** — enqueue เป็น `Turbo::Streams::ActionBroadcastJob` ผ่าน ActiveJob แล้วค่อย broadcast ตอนjob รัน |
| ผลกระทบ | ถ้า partial render ช้า จะหน่วง request/callback ที่เรียกมัน | ไม่บล็อก request หลัก เหมาะกับ record ที่มีคน subscribe เยอะหรือ partial render หนัก |
| เหมาะกับ | อยากควบคุม target/partial เอง (เช่น append เข้า collection ของอีก record หนึ่ง อย่าง comment เข้า post) | อยาก sync สถานะของ record ตัวมันเองแบบ "set-and-forget" บรรทัดเดียว |

เมธอดที่ใช้ async ภายในยังมีให้เรียกตรงๆ ได้เช่นกัน (ชื่อลงท้ายด้วย `_later_to`):
`broadcast_append_later_to`, `broadcast_prepend_later_to`, `broadcast_replace_later_to`,
`broadcast_update_later_to` — เลือกใช้เมื่ออยากได้พฤติกรรม non-blocking แบบ `broadcasts_to`
แต่ยังอยากคุม target/partial เองแบบ Step 516

> **เกร็ดความรู้เพิ่มเติม:** Rails/Turbo เวอร์ชันใหม่ยังมีกลไกทางเลือกชื่อ `broadcasts_refreshes`
> ซึ่งใช้แนวคิด "Page Refresh" ของ Turbo 8 (สั่งให้ browser fetch หน้าปัจจุบันซ้ำแล้ว morph DOM
> แทนการ append/replace fragment เจาะจง) เหมาะกับหน้าที่ผูก state ซับซ้อนหลายจุดในหน้าเดียว
> Part นี้ไม่ได้ลงลึกกลไกนี้เพราะโฟกัสที่ Turbo Stream แบบ fragment-based เป็นหลัก แต่ควรรู้ว่า
> มีทางเลือกนี้อยู่เมื่อไปอ่านเอกสารของ Hotwire เพิ่มเติม

---

## Step 518: Multiple targets และ `turbo_stream.action` สำหรับ custom action

### อัปเดตหลาย element พร้อมกันด้วย `targets` (CSS selector)

ทุก action หลัก (`append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`) มีคู่
"_all" ที่รับ CSS selector แทน id เดี่ยว:

```ruby
turbo_stream.append_all(".comment", partial: "comments/comment", locals: { comment: comment })
turbo_stream.replace_all(".comment.unread", partial: "comments/comment", locals: { comment: comment })
turbo_stream.remove_all(".flash-message")
```

ทดสอบจริงด้วย `turbo_stream.replace_all(".comment", ...)`:

```html
<turbo-stream action="replace" targets=".comment"><template>...</template></turbo-stream>
```

สังเกตว่า attribute เปลี่ยนจาก `target="..."` เอกพจน์ เป็น `targets="..."` พหูพจน์ (สะกดกันคนละ
แบบจริงๆ ในสเปกของ Turbo) — ฝั่ง client จะใช้ `document.querySelectorAll` แทน
`document.getElementById` ทำให้อัปเดตได้หลาย element ที่ match selector เดียวกันในคราวเดียว
เหมาะกับกรณีเช่น "badge แสดงสถานะ" ที่ปรากฏซ้ำหลายจุดในหน้าเดียว

### Custom action ด้วย `turbo_stream.action`

นอกจาก 7 actions มาตรฐาน เรายังนิยาม action ของตัวเองได้ ใช้เมื่อ 7 แบบไม่พอ เช่น ต้องการ
"highlight" element ที่เพิ่งอัปเดตด้วย animation โดยไม่ต้องแทนที่ DOM ทั้งก้อน:

```ruby
render turbo_stream: turbo_stream.action(:highlight, "comment_2")
```

ทดสอบจริง:

```html
<turbo-stream action="highlight" target="comment_2"><template></template></turbo-stream>
```

แต่ browser จะไม่รู้ว่า action `highlight` ต้องทำอะไร จนกว่าเราจะไปลงทะเบียนมันฝั่ง JavaScript:

```javascript
// app/javascript/custom/turbo_stream_actions.js
import { Turbo } from "@hotwired/turbo-rails"

Turbo.StreamActions.highlight = function () {
  // `this` คือ turbo-stream element เอง, `this.targetElements` คือ array ของ
  // element ปลายทางทั้งหมด (รองรับทั้ง target เดี่ยวและ targets พหูพจน์)
  this.targetElements.forEach((element) => {
    element.classList.add("highlight")
    setTimeout(() => element.classList.remove("highlight"), 2000)
  })
}
```

```javascript
// app/javascript/application.js
import "./custom/turbo_stream_actions"
```

```css
.highlight {
  transition: background-color 2s ease;
  background-color: #fff3cd;
}
```

วิธีนี้ทำให้ backend สั่ง effect ฝั่ง UI แบบง่ายๆ (ไฮไลต์แถวที่เพิ่งอัปเดต, เล่นเสียงแจ้งเตือน,
เลื่อนสกอลล์ไปยัง element ใหม่) ได้โดยไม่ต้องพึ่ง Stimulus controller เต็มรูปแบบ — Stimulus
(Part 053) จะเหมาะกับ interaction ที่ซับซ้อนกว่านี้ ที่ต้องจัดการ state ฝั่ง client เอง

---

## Step 519: ผสาน Turbo Streams กับ Background Job + ข้อควรระวังด้านความปลอดภัย

### ผสานกับงานที่ใช้เวลานาน (เกริ่นนำ — รายละเอียดเต็มอยู่ Phase 9: Part 061 เป็นต้นไป)

บางงานใช้เวลานานเกินกว่าจะ render กลับใน request เดียว (เช่น ประมวลผลรูปภาพ, เรียก API
ภายนอก, สร้างรายงาน PDF) รูปแบบที่นิยมคือ: ตอบ Turbo Stream ทันทีด้วย placeholder/spinner แล้ว
ให้ background job เป็นคนยิง broadcast จริงเมื่องานเสร็จ ผ่าน stream เดิมที่หน้านั้น subscribe
อยู่แล้ว:

```ruby
# app/controllers/reports_controller.rb
def create
  @report = current_user.reports.create!(report_params)
  GenerateReportJob.perform_later(@report)

  render turbo_stream: turbo_stream.append(
    "reports",
    partial: "reports/pending", locals: { report: @report }
  )
end
```

```ruby
# app/jobs/generate_report_job.rb
class GenerateReportJob < ApplicationJob
  def perform(report)
    pdf = SlowPdfGenerator.call(report)
    report.update!(file: pdf, status: :done)
    # after_update_commit บน Report model (หรือเรียกตรงนี้เลย) จะ broadcast_replace_to
    # แทนที่ partial "pending" ด้วย partial "done" ที่มีลิงก์ดาวน์โหลด
  end
end
```

```ruby
# app/models/report.rb
class Report < ApplicationRecord
  after_update_commit -> {
    broadcast_replace_to owner, target: self, partial: "reports/report", locals: { report: self }
  }
end
```

ผู้ใช้เห็น "กำลังสร้างรายงาน..." ทันที แล้วเห็นมันเปลี่ยนเป็นลิงก์ดาวน์โหลดโดยอัตโนมัติเมื่อ job
เสร็จ — ไม่ต้อง refresh หน้า ไม่ต้องเขียน polling JavaScript เอง (รายละเอียดเรื่อง ActiveJob,
adapter, retry, ฯลฯ จะเรียนเต็มรูปแบบใน Part 061 "ActiveJob เบื้องต้น")

### ความปลอดภัย — Broadcast ต้อง scope ให้ถูกคนเสมอ

Turbo Stream ทำให้ real-time UI ทำได้ง่ายมาก แต่ก็เปิดช่องให้เกิดข้อมูลรั่วไหลได้ง่ายพอกัน
ถ้าไม่ระวังเรื่อง scope ของ stream ข้อควรระวังที่สำคัญ:

1. **`signed_stream_name` ป้องกันการปลอมแปลง แต่ไม่ป้องกันการดักฟังของแท้** — ถ้าเรา
   `turbo_stream_from @post` ในหน้าที่ **ไม่ควรให้ผู้ใช้คนนี้เข้าถึงได้** (เช่น post เป็น draft
   ส่วนตัวของคนอื่น) แล้วเผลอ render หน้านั้นให้เขาเห็น (บั๊กเรื่อง authorization ใน controller)
   ตัว signed stream name ที่หลุดออกมาในหน้า HTML นั้นก็ใช้ subscribe รับข้อมูลจริงได้ทันที —
   **การป้องกันต้องอยู่ที่ชั้น controller/policy ก่อนจะ render `turbo_stream_from` เสมอ**
   ไม่ใช่หวังพึ่งการ sign อย่างเดียว

   ```erb
   <%# ผิด: render turbo_stream_from โดยไม่เช็คสิทธิ์ก่อน %>
   <%= turbo_stream_from @post %>

   <%# ถูก: เช็คให้แน่ใจว่า action `show` ผ่าน authorization (เช่น Pundit) มาก่อนแล้ว
       ก่อนหน้านี้ ไม่ใช่แค่ไม่ error ตอน render %>
   ```

2. **อย่าใช้ stream แบบ "รวม" (shared/global) สำหรับข้อมูลส่วนตัว** — เช่นถ้าจะ broadcast
   คำแจ้งเตือนของผู้ใช้แต่ละคน อย่า broadcast เข้า stream ที่ทุกคน subscribe ร่วมกัน
   (`"notifications"` เฉยๆ) เพราะทุกคนที่เปิดหน้าจะเห็นการแจ้งเตือนของทุกคน ให้ scope ด้วย
   ตัวตนผู้ใช้เสมอ:

   ```erb
   <%# scope stream ด้วย current_user เพื่อให้เป็น "ช่องส่วนตัว" ของแต่ละคน %>
   <%= turbo_stream_from [current_user, :notifications] %>
   ```

   ```ruby
   # broadcast ก็ต้องระบุ scope ให้ตรงกันเป๊ะ ไม่ใช่ broadcast เข้า current_user ผิดคน
   broadcast_append_to [notification.recipient, :notifications],
                        target: "notifications", partial: "notifications/notification"
   ```

3. **ตรวจสอบว่า ActionCable connection รู้จักตัวตนผู้ใช้** — ถ้า `identified_by` ใน
   `ApplicationCable::Connection` ไม่ผูกกับผู้ใช้ที่ login อยู่ การ scope stream ด้วย
   `current_user` ในข้อ 2 ก็ไม่มีความหมาย เพราะใครก็เชื่อมต่อ ActionCable ได้แล้วสุ่ม
   subscribe เข้า stream ของคนอื่นถ้าเดาค่า signed name ได้ (ซึ่งถูกออกแบบมาให้เดาไม่ได้ แต่การ
   reject connection ที่ไม่ได้ login ตั้งแต่ต้นทางเป็นเกราะป้องกันอีกชั้นที่ควรมี):

   ```ruby
   # app/channels/application_cable/connection.rb
   module ApplicationCable
     class Connection < ActionCable::Connection::Base
       identified_by :current_user

       def connect
         self.current_user = find_verified_user
       end

       private

       def find_verified_user
         if (user = User.find_by(id: cookies.encrypted[:user_id]))
           user
         else
           reject_unauthorized_connection
         end
       end
     end
   end
   ```

4. **ระวัง callback ที่ broadcast กว้างเกินไปโดยไม่ตั้งใจ** — `broadcasts_to` ที่ระบุ stream
   เป็นค่าคงที่ (ไม่ผูกกับเจ้าของ record) เช่น `broadcasts_to ->(order) { "orders" }` (ไม่ scope
   ด้วยผู้ใช้) จะทำให้ order ของทุกคนไปโผล่ใน stream เดียวกันหมด ต้อง scope ด้วยเจ้าของเสมอ
   อย่าง `broadcasts_to ->(order) { [order.user, "orders"] }`

สรุปหลักคิดสั้นๆ: **"stream name ที่ sign แล้ว" ป้องกันการปลอมแปลง แต่ "การ scope stream ให้แคบ
พอ + authorization ที่ controller/channel" คือสิ่งที่ป้องกันข้อมูลรั่วไหลจริง**

---

## Step 520: แบบฝึกหัด — ระบบคอมเมนต์ real-time บน Post

### โจทย์

สร้างระบบคอมเมนต์บน `Post` ที่:

1. ผู้ใช้เพิ่มคอมเมนต์ผ่านฟอร์มได้ตามปกติ (ตอบทั้ง `format.html` และ `format.turbo_stream`)
2. เมื่อมีคอมเมนต์ใหม่ถูกสร้าง **ทุกคนที่กำลังเปิดหน้า post นั้นอยู่ต้องเห็นคอมเมนต์ใหม่ทันที
   โดยไม่ต้อง refresh** (ผ่าน ActionCable broadcast)
3. ลบคอมเมนต์ได้ และการลบก็ต้อง real-time เช่นกัน

### เฉลย

**Migration/Model:**

```ruby
# db/migrate/xxxx_create_comments.rb
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :post, null: false, foreign_key: true
      t.string :author_name
      t.text :body
      t.timestamps
    end
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  validates :body, presence: true

  after_create_commit -> {
    broadcast_append_to post, target: "comments", partial: "comments/comment", locals: { comment: self }
  }
  after_update_commit -> {
    broadcast_replace_to post, target: self, partial: "comments/comment", locals: { comment: self }
  }
  after_destroy_commit -> {
    broadcast_remove_to post, target: self
  }
end
```

**Routes:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts do
    resources :comments, only: [:create, :destroy]
  end
end
```

**Controller:**

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_post

  def create
    @comment = @post.comments.build(comment_params)

    respond_to do |format|
      if @comment.save
        format.turbo_stream
        format.html { redirect_to @post, notice: "เพิ่มคอมเมนต์แล้ว" }
      else
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_comment", partial: "comments/form", locals: { post: @post, comment: @comment }
          )
        end
        format.html { redirect_to @post, alert: "เพิ่มคอมเมนต์ไม่สำเร็จ" }
      end
    end
  end

  def destroy
    @comment = @post.comments.find(params[:id])
    @comment.destroy

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @post, notice: "ลบคอมเมนต์แล้ว" }
    end
  end

  private

  def set_post
    @post = Post.find(params[:post_id])
  end

  def comment_params
    params.require(:comment).permit(:body, :author_name)
  end
end
```

**Views:**

```erb
<%# app/views/comments/_comment.html.erb %>
<%= turbo_frame_tag comment do %>
  <div id="<%= dom_id(comment) %>" class="comment">
    <p><strong><%= comment.author_name.presence || "ไม่ระบุชื่อ" %></strong>: <%= comment.body %></p>
    <%= button_to "ลบ", [comment.post, comment], method: :delete,
          form: { data: { turbo_confirm: "แน่ใจนะ?" } } %>
  </div>
<% end %>
```

```erb
<%# app/views/comments/_form.html.erb %>
<%= turbo_frame_tag "new_comment" do %>
  <%= form_with model: comment, url: post_comments_path(post) do |f| %>
    <% if comment.errors.any? %>
      <div class="error"><%= comment.errors.full_messages.to_sentence %></div>
    <% end %>
    <p>
      <%= f.label :author_name, "ชื่อผู้เขียน" %>
      <%= f.text_field :author_name %>
    </p>
    <p>
      <%= f.label :body, "ข้อความ" %>
      <%= f.text_area :body %>
    </p>
    <%= f.submit "ส่งคอมเมนต์" %>
  <% end %>
<% end %>
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", partial: "comments/comment", locals: { comment: @comment } %>
<%= turbo_stream.replace "new_comment", partial: "comments/form", locals: { post: @post, comment: Comment.new } %>
```

```erb
<%# app/views/comments/destroy.turbo_stream.erb %>
<%= turbo_stream.remove @comment %>
```

```erb
<%# app/views/posts/show.html.erb — ส่วนที่เพิ่มเข้าไปจาก scaffold เดิม %>
<%= turbo_stream_from @post %>

<h2>คอมเมนต์</h2>
<div id="comments">
  <%= render @post.comments %>
</div>

<%= render "comments/form", post: @post, comment: Comment.new %>
```

### วิธีตรวจสอบว่า real-time ทำงานจริง

**1) ตรวจ contract ของ Turbo Stream response ด้วย curl:**

```bash
curl -s -D - -c cookies.txt -b cookies.txt \
  -H "Accept: text/vnd.turbo-stream.html" \
  -H "X-CSRF-Token: $TOKEN" \
  -F "comment[author_name]=Alice" -F "comment[body]=สวัสดีครับ" \
  http://127.0.0.1:3000/posts/1/comments
```

ต้องได้ `content-type: text/vnd.turbo-stream.html; charset=utf-8` และ body มี
`<turbo-stream action="append" target="comments">` ห่อ partial ของคอมเมนต์ไว้ (ผลลัพธ์จริงตรง
กับที่แสดงใน Step 513)

**2) ตรวจว่า broadcast ยิงออก ActionCable จริงด้วย `rails console` + log:**

```ruby
post = Post.first
post.comments.create!(author_name: "Bob", body: "ทดสอบ real-time")
```

เปิด `log/development.log` (หรือดู console output ถ้ารันผ่าน `rails runner`) ต้องเห็นบรรทัด:

```
[ActionCable] Broadcasting to <signed-stream-name>: "<turbo-stream action=\"append\" target=\"comments\">..."
```

`<signed-stream-name>` นี้ต้องตรงกับค่าที่ `Turbo::StreamsChannel.signed_stream_name(post)` คืน
มา (เทียบกับค่าที่ปรากฏใน `<turbo-cable-stream-source signed-stream-name="...">` ของหน้า
`/posts/1` จริง) — ถ้าตรงกัน แปลว่า browser ใดก็ตามที่เปิดหน้านั้นค้างไว้และ subscribe สำเร็จ
จะได้รับ HTML fragment นี้ผ่าน WebSocket และ Turbo JS จะ append เข้า `#comments` ให้อัตโนมัติ
โดยไม่ต้อง refresh — **นี่คือ mechanism เดียวกับที่ Step 516 พิสูจน์ไว้แบบละเอียด**

**3) ตรวจการลบ:**

```bash
curl -s -H "Accept: text/vnd.turbo-stream.html" -X DELETE \
  -H "X-CSRF-Token: $TOKEN" -b cookies.txt \
  http://127.0.0.1:3000/posts/1/comments/1
# => <turbo-stream action="remove" target="comment_1"></turbo-stream>
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่มตัวนับจำนวนคอมเมนต์ (`<span id="comment_count">`) ที่อัปเดตแบบ real-time ทุกครั้งที่มี
   คอมเมนต์ใหม่/ถูกลบ (ใบ้: ต้องส่ง `<turbo-stream>` เพิ่มอีกก้อนหนึ่งคู่กับ append/remove เดิม
   ทั้งใน `create.turbo_stream.erb`/`destroy.turbo_stream.erb` และในทุก callback ที่ broadcast)
2. ปรับให้ `turbo_stream_from` และ broadcast ทั้งหมด scope ตาม `current_user` แบบ Step 519
   สมมติว่า post มีสถานะ private ที่มองเห็นได้เฉพาะเจ้าของ — ต้องมั่นใจว่าคนอื่นไม่สามารถ
   subscribe ไปเห็นคอมเมนต์ real-time ของ post ส่วนตัวคนอื่นได้ แม้จะเดา URL ถูก
3. ทำ pagination ให้แสดงคอมเมนต์ล่าสุดแค่ 20 รายการ และเมื่อมีคอมเมนต์ใหม่เกิน 20
   ให้คอมเมนต์เก่าสุดถูกลบออกจาก DOM อัตโนมัติผ่าน broadcast (ใบ้: broadcast สอง action
   พร้อมกันคือ `append` ตัวใหม่ และ `remove` ตัวที่เกินโควตา)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจกายวิภาคของ `<turbo-stream action="..." target="...">` และครบทั้ง 7 actions
  (`append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`) พร้อมพิสูจน์ผลลัพธ์จริง
  ด้วย curl
- เขียน controller เดียวตอบได้ทั้ง `format.html` (fallback เต็มหน้า) และ `format.turbo_stream`
  (อัปเดตเฉพาะจุด) ตามรูปแบบคลาสสิกที่ใช้ทั่วไปในโปรเจกต์ Rails จริง
- สร้างและใช้ `.turbo_stream.erb` template แยกไฟล์ และใช้ `turbo_stream` builder ทั้งในไฟล์
  view และตรงใน controller ผ่าน `render turbo_stream: [...]`
- เข้าใจว่า `turbo_stream_from` เปิดการ subscribe ผ่าน `Turbo::StreamsChannel` ของ ActionCable
  ได้อย่างไร และ `signed-stream-name` ทำหน้าที่ป้องกันการปลอมแปลง stream อย่างไร
- ใช้ `broadcast_append_to`/`broadcast_replace_to`/`broadcast_remove_to` จาก model callback
  ทำให้เกิด **real-time update ข้ามผู้ใช้** — พิสูจน์แล้วว่า User A สร้างคอมเมนต์แล้ว User B
  ที่เปิดหน้าเดียวกันจะเห็นทันทีผ่าน ActionCable โดยไม่ต้อง refresh
- รู้จัก `broadcasts_to` บรรทัดเดียวสำหรับ CRUD broadcasting อัตโนมัติ และพิสูจน์แล้วว่ามันทำงาน
  ผ่าน ActiveJob (asynchronous) ต่างจาก `broadcast_*_to` ที่ทำงานแบบ synchronous
- ใช้ `targets` (CSS selector) อัปเดตหลาย element พร้อมกัน และสร้าง custom action ของตัวเองด้วย
  `turbo_stream.action` คู่กับ `Turbo.StreamActions` ฝั่ง JavaScript
- รู้วิธีผสาน Turbo Stream กับ background job สำหรับงานที่ใช้เวลานาน และเข้าใจหลักการ scope
  stream ให้ปลอดภัย ไม่ให้ผู้ใช้เห็นข้อมูล real-time ของคนอื่นโดยไม่ได้รับอนุญาต
- สร้างระบบคอมเมนต์ real-time เต็มรูปแบบบน `Post` model ได้ด้วยตัวเอง ตั้งแต่ migration,
  model callback, controller, ไปจนถึง view — โดยไม่ต้องเขียน JavaScript เองสักบรรทัดเดียว

**ต่อไป (Part 053):** เราจะเรียน **Stimulus.js** เฟรมเวิร์ก JavaScript เบาๆ ที่มากับ Hotwire
สำหรับกรณีที่ Turbo Drive/Frame/Stream ยังไม่พอ (เช่น toggle เมนู, validate ฟอร์มฝั่ง client,
drag-and-drop) จะเจาะลึกโครงสร้างหลักของ Stimulus: **controller**, **action**, **target**, และ
**value** พร้อมตัวอย่างการเขียน controller ตัวแรกที่เชื่อมกับ HTML ผ่าน `data-controller`,
`data-action`, และ `data-*-target` attributes
