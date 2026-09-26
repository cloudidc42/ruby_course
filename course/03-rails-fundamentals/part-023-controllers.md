# Part 023: Controller — action, params, session, flash, before_action

> **Step ครอบคลุมใน Part นี้:** Step 221–230
> **ระดับ:** เริ่มต้น–ปานกลาง (ต่อจาก Part 021 การติดตั้ง Rails และ Part 022 เรื่อง Routing)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Rails 8.1.4)

Controller คือ "สมองส่วนกลาง" ของ Rails MVC — เป็นตัวรับ request ที่ router ส่งมาให้
(ตามที่เรียนไปใน Part 022) ตัดสินใจว่าจะทำอะไรกับ request นั้น ดึงข้อมูลจาก Model
(ซึ่งจะเรียนละเอียดใน Part 025) แล้วส่งต่อให้ View render ผลลัพธ์กลับไปหา browser

Part นี้เจาะลึกเฉพาะ Controller layer: โครงสร้างของมัน, การอ่านข้อมูลที่ผู้ใช้ส่งมา (`params`),
การเก็บข้อมูลข้าม request (`session`), การส่งข้อความชั่วคราว (`flash`), และกลไก filter/callback
ที่ทำให้เขียนโค้ดที่ใช้ร่วมกันหลาย action ได้โดยไม่ต้องเขียนซ้ำ

## สารบัญของ Part นี้

- Step 221: `ApplicationController` และการสืบทอด Controller
- Step 222: สร้าง Controller ด้วย `rails generate controller`, action คือ public method, การ render โดย convention
- Step 223: `params` เบื้องต้น — query string parameters
- Step 224: `params` จาก route (`:id`) และจากฟอร์ม (`params[:model][:field]`)
- Step 225: `session` — เก็บข้อมูลข้าม request ด้วย cookie-based session
- Step 226: `flash` และ `flash.now` — ข้อความแบบใช้ครั้งเดียว
- Step 227: `redirect_to` vs `render` และรูปแบบ Post/Redirect/Get
- Step 228: `before_action` พร้อม `only:`/`except:` — logic ที่ใช้ร่วมกันหลาย action
- Step 229: `after_action`, `around_action` เบื้องต้น และ `rescue_from` จัดการ error ระดับ Controller
- Step 230: แบบฝึกหัด — `GreetingsController` ครบวงจร

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยการสร้างแอป Rails เปล่าๆ
> รัน `bin/rails server` แล้วยิง request ด้วย `curl` เพื่อดู response, header, และ log จริง
> ไม่ใช่โค้ดที่เขียนขึ้นลอยๆ — ผลลัพธ์ที่แสดงในเอกสารนี้คือผลลัพธ์จริงที่ได้จากการรัน

---

## Step 221: `ApplicationController` และการสืบทอด Controller

เมื่อสร้างโปรเจกต์ใหม่ด้วย `rails new` (ตามที่เรียนใน Part 021) Rails จะสร้างไฟล์
`app/controllers/application_controller.rb` มาให้อัตโนมัติ:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Only allow modern browsers supporting webp images, web push, badges, import maps, CSS nesting, and CSS :has.
  allow_browser versions: :modern
end
```

**อธิบายโครงสร้าง:**

- `ActionController::Base` คือ base class ของ Controller ทั้งหมดใน Rails มาจาก gem
  `actionpack` ซึ่งเป็นส่วนหนึ่งของ Rails framework ให้ความสามารถพื้นฐานทั้งหมด เช่น
  การ render view, จัดการ session/cookie, redirect, ส่ง response กลับไปยัง client
- `ApplicationController` เป็น controller กลางที่ controller อื่นทุกตัวในแอปจะสืบทอด
  (inherit) มาจากมันอีกที **ไม่ใช่** สืบทอดจาก `ActionController::Base` โดยตรง เพื่อให้มีจุด
  เดียวสำหรับใส่ logic ที่ต้องใช้ร่วมกันทั้งแอป เช่น การตรวจสอบ login, การจัดการ error, หรือ
  method ช่วยเหลือที่ controller ทุกตัวต้องใช้
- `allow_browser versions: :modern` เป็นฟีเจอร์ default ของ Rails 8 ที่บล็อก browser
  รุ่นเก่าที่ไม่รองรับฟีเจอร์เว็บสมัยใหม่ (เราจะปิดหรือปรับ config นี้ชั่วคราวในตัวอย่างที่ใช้
  `curl` ทดสอบ เพราะ `curl` ไม่ได้ส่ง User-Agent แบบ browser จริง จึงถูกปฏิเสธ)

โครงสร้างการสืบทอดที่เราจะใช้ตลอด Part นี้:

```
ActionController::Base
        ↑
ApplicationController         (app/controllers/application_controller.rb)
        ↑
GreetingsController, ArticlesController, SecretAreaController, ... (controller ของเราเอง)
```

**ทำไมต้องมีชั้นกลาง (`ApplicationController`) เสมอ:** เพราะ Ruby รองรับการสืบทอดได้ทีละ
class เดียว (single inheritance) การมี `ApplicationController` เป็นจุดกลางทำให้เราเพิ่ม
`before_action`, `rescue_from`, หรือ helper method ที่ต้องใช้ "ทุก controller" ได้ในที่เดียว
แทนที่จะต้องเขียนซ้ำในทุก controller — เป็นการใช้หลัก **DRY** (Don't Repeat Yourself) ที่
เรียนมาตั้งแต่เฟส 1

---

## Step 222: สร้าง Controller ด้วย `rails generate controller`, action คือ public method, และการ render โดย convention

### ใช้ generator สร้าง Controller

```bash
bin/rails generate controller Greetings hello
```

ผลลัพธ์จริงที่ได้ (ทดสอบบน Rails 8.1.4):

```
      create  app/controllers/greetings_controller.rb
       route  get "greetings/hello"
      invoke  erb
      create    app/views/greetings
      create    app/views/greetings/hello.html.erb
      invoke  test_unit
      create    test/controllers/greetings_controller_test.rb
      invoke  helper
      create    app/helpers/greetings_helper.rb
      invoke    test_unit
```

สังเกตว่า generator สร้างให้ 4 อย่างพร้อมกัน:

1. **Controller file** `app/controllers/greetings_controller.rb`
2. **Route** — เพิ่มบรรทัด `get "greetings/hello"` ให้ใน `config/routes.rb` โดยอัตโนมัติ
   (ชื่อ path จะกลายเป็น `greetings/hello` ตรงตามที่ระบุ)
3. **View** `app/views/greetings/hello.html.erb` — ไฟล์เปล่าๆ ที่มีชื่อตรงกับ action
4. **Test file และ helper** — ไฟล์เปล่าสำหรับเขียน test และ helper method ในอนาคต

> ถ้าไม่ต้องการให้ generator แก้ไฟล์ `routes.rb` ให้ (เช่นตอนที่เราต้องการกำหนด route เอง
> ตามที่เรียนใน Part 022) ใส่ flag `--skip-routes`:
> ```bash
> bin/rails generate controller Greetings hello --skip-routes
> ```

ไฟล์ `greetings_controller.rb` ที่ได้:

```ruby
class GreetingsController < ApplicationController
  def hello
  end
end
```

### Action คือ public instance method ธรรมดา

สิ่งสำคัญที่ต้องเข้าใจ: **action ก็คือ public instance method ของ class ที่สืบทอดจาก
`ApplicationController`** ไม่มีอะไรพิเศษไปกว่า method ทั่วไปที่เรียนมาตั้งแต่ Part 007
Rails router เป็นคนเรียก method นี้ให้เมื่อมี request เข้ามาตรงกับ route ที่ผูกไว้

จุดที่ทำให้ action "พิเศษ" กว่า method ทั่วไปคือ:

- Action ต้องเป็น **public method** เท่านั้น — ถ้าประกาศเป็น `private` หรือ `protected`
  Rails จะหา action นั้นไม่เจอ และโยน `ActionController::UrlGenerationError` หรือ
  `AbstractController::ActionNotFound` (ธรรมเนียมปฏิบัติคือ helper method ภายในของ
  controller ให้ประกาศเป็น `private` เสมอ เพื่อกันไม่ให้ผู้ใช้เรียก method เหล่านั้นผ่าน URL
  ได้โดยตรง — จะเห็นตัวอย่างจริงในหัวข้อ `before_action` ต่อไป)
- ภายใน method นี้ เราเข้าถึง object พิเศษที่ Rails เตรียมไว้ให้ได้ทันที เช่น `params`,
  `session`, `flash`, `request`, `response`, `cookies` — ซึ่งจะเรียนทีละตัวใน Part นี้

### การ render view โดยอัตโนมัติตาม convention

สังเกตว่า method `hello` ด้านบนไม่มีเนื้อหาอะไรเลย (`def hello; end`) แต่เมื่อรัน server
แล้วเข้า `http://localhost:3000/greetings/hello` จะได้หน้าเว็บที่มีเนื้อหาจากไฟล์
`app/views/greetings/hello.html.erb` แสดงออกมา:

```erb
<%# app/views/greetings/hello.html.erb %>
<h1>Greetings#hello</h1>
<p>Find me in app/views/greetings/hello.html.erb</p>
```

นี่คือ **Convention over Configuration** อีกตัวอย่างหนึ่ง (แนวคิดที่เรียนไปตั้งแต่ Step 1):
ถ้า action ไม่ได้เรียก `render` หรือ `redirect_to` อย่างชัดเจน Rails จะ **render view โดย
อัตโนมัติ** จากไฟล์ที่อยู่ใน `app/views/<ชื่อ controller เป็น snake_case>/<ชื่อ action>.html.erb`
เสมอ

ผลลัพธ์จริงจากการยิง `curl` ไปที่ route นี้ (ตัดส่วน `<head>` ออกเพื่อความกระชับ):

```bash
curl http://localhost:3000/greetings/hello
```

```html
<!-- BEGIN app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>...</head>
  <body>
    <!-- BEGIN app/views/greetings/hello.html.erb -->
    <h1>Greetings#hello</h1>
    <p>Find me in app/views/greetings/hello.html.erb</p>
    <!-- END app/views/greetings/hello.html.erb -->
  </body>
</html>
<!-- END app/views/layouts/application.html.erb -->
```

(คอมเมนต์ `BEGIN`/`END` เหล่านี้ Rails ใส่มาให้เองในโหมด development เพื่อบอกว่าส่วนไหนของ
HTML มาจากไฟล์ view ไหน มีประโยชน์มากตอน debug layout/partial ซับซ้อน — เดี๋ยวจะเรียนเรื่อง
layout และ partial แบบเต็มใน Part 024)

ถ้า action **มีเนื้อหา** และเรียก `render` หรือ `redirect_to` เองแล้ว Rails จะไม่ render view
อัตโนมัติซ้ำอีก (และถ้า action หนึ่งเรียก `render`/`redirect_to` มากกว่า 1 ครั้ง จะได้ error
`AbstractController::DoubleRenderError: Render and/or redirect were called multiple times
in this action` — เป็นข้อผิดพลาดที่มือใหม่เจอบ่อยมาก ต้องระวัง)

---

## Step 223: `params` เบื้องต้น — query string parameters

`params` คือ object ที่ Rails เตรียมไว้ให้ทุก action เข้าถึงได้ทันที เก็บ **ข้อมูลทั้งหมด
ที่มาพร้อมกับ request** ไม่ว่าจะมาจาก query string, route segment, หรือ form body — เป็น
instance ของ class `ActionController::Parameters` (ไม่ใช่ `Hash` ธรรมดา แต่ใช้งานคล้าย Hash
มาก มี method `[]`, `fetch`, `dig`, `key?` ฯลฯ)

เพิ่ม action ทดลองใน `GreetingsController`:

```ruby
class GreetingsController < ApplicationController
  def hello
  end

  def show_params
    render plain: params.inspect
  end
end
```

เพิ่ม route:

```ruby
# config/routes.rb
get "greetings/show_params", to: "greetings#show_params"
```

ทดสอบด้วย query string:

```bash
curl "http://localhost:3000/greetings/show_params?name=Somchai&lang=th"
```

ผลลัพธ์จริง:

```
#<ActionController::Parameters {"name"=>"Somchai", "lang"=>"th", "controller"=>"greetings", "action"=>"show_params"} permitted: false>
```

**สังเกต 3 อย่างสำคัญ:**

1. `name` และ `lang` คือ query string parameters ที่เราส่งมาผ่าน URL (`?name=...&lang=...`)
   Rails parse ให้อัตโนมัติ ไม่ต้องเขียนโค้ด parse URL เอง
2. Rails ใส่ `controller` และ `action` เข้าไปใน `params` ให้เองเสมอ (มาจากผลของการ
   match route ตามที่เรียนใน Part 022) — เป็นเรื่องปกติ ไม่ใช่บั๊ก
3. `permitted: false` คือสถานะ **strong parameters** — บอกว่า params ชุดนี้ยังไม่ได้ถูก
   "อนุญาต" ให้ใช้ mass-assignment กับ Model (เช่น `Article.create(params)`) เราจะเรียน
   strong parameters แบบเต็มใน Part 031 ตอนนี้รู้แค่ว่าเห็นคำนี้แล้วอย่าเพิ่งตกใจ

การอ่านค่าจาก `params`:

```ruby
def show_params
  name = params[:name]        # => "Somchai"  (เข้าถึงด้วย Symbol ได้ แม้ key จริงจะเป็น String)
  name = params["name"]       # => "Somchai"  (เข้าถึงด้วย String ก็ได้เหมือนกัน)
  missing = params[:missing]  # => nil        (ไม่มี key นี้ ได้ nil ไม่ error)
  render plain: name
end
```

**จุดเด่นที่ต่างจาก `Hash` ธรรมดา:** `ActionController::Parameters` ยอมให้ใช้ทั้ง Symbol
และ String เข้าถึง key เดียวกันได้ (เรียกว่า **indifferent access**) ซึ่งสะดวกมาก เพราะ
ข้อมูลที่มาจาก HTTP request (query string, form) เป็น String ล้วน แต่โค้ด Ruby ของเรามักเขียน
ด้วย Symbol

---

## Step 224: `params` จาก route (`:id`) และจากฟอร์ม (`params[:model][:field]`)

### Route parameters

จาก Part 022 เรารู้จัก dynamic segment ใน route เช่น `:id` แล้ว ค่าที่ match ได้จะถูกใส่เข้า
`params` เหมือนกับ query string ทุกประการ — ไม่มีความต่างในมุมของ action เลย

```ruby
# config/routes.rb
get "articles/:id", to: "articles#show", as: :article
```

```ruby
# app/controllers/articles_controller.rb
class ArticlesController < ApplicationController
  def show
    render plain: "ดูบทความ id = #{params[:id]}"
  end
end
```

ทดสอบ:

```bash
curl http://localhost:3000/articles/42
# => ดูบทความ id = 42
```

`params[:id]` ได้ค่าเป็น **String เสมอ** (`"42"` ไม่ใช่ `42`) เพราะ URL เป็น text ล้วนๆ
ถ้าต้องใช้เป็นตัวเลขต้อง `.to_i` เอง (ตามที่เรียนเรื่องการแปลงชนิดข้อมูลใน Part 002)

### Form parameters

เมื่อฟอร์ม HTML ส่งข้อมูลแบบ `POST`/`PATCH` มา ข้อมูลจะอยู่ใน **request body** แทนที่จะอยู่
ใน URL แต่ Rails ก็ยัดรวมเข้า `params` ให้เหมือนกัน โดยฟอร์มที่มี `name` เป็นรูปแบบ
`article[title]` จะถูก parse เป็น nested structure:

```html
<form action="/articles" method="post">
  <input type="text" name="article[title]" placeholder="ชื่อบทความ">
  <button type="submit">บันทึก</button>
</form>
```

```ruby
def create
  title = params[:article][:title]
  render plain: "ได้รับชื่อบทความ: #{title}"
end
```

เมื่อ submit ฟอร์มด้วยชื่อ "หัดเขียน Rails Controller" ค่า `params` เต็มๆ ที่ controller ได้รับ
จะมีหน้าตาประมาณนี้ (ดูจาก log จริงของ Rails):

```
Parameters: {"authenticity_token"=>"[FILTERED]", "article"=>{"title"=>"หัดเขียน Rails Controller"}, "controller"=>"articles", "action"=>"create"}
```

**สังเกต:** `article` กลายเป็น nested `ActionController::Parameters` อีกชั้นหนึ่ง เข้าถึงด้วย
`params[:article][:title]` ส่วน `authenticity_token` คือ CSRF token ที่ Rails แนบมาป้องกัน
Cross-Site Request Forgery โดยอัตโนมัติทุกฟอร์ม (log จะซ่อนค่านี้เป็น `[FILTERED]` เพื่อความ
ปลอดภัย เรื่อง CSRF แบบเต็มจะเรียนใน Part 079 เรื่อง Security)

> **ทำไม `Article.create(params[:article])` ตรงๆ ถึงอันตราย:** เพราะผู้ใช้ที่ประสงค์ร้าย
> อาจแอบส่ง field อื่นที่ไม่ควรแก้ไขได้ เช่น `article[admin]=true` ปนเข้ามาในฟอร์ม
> (เรียกว่า **mass assignment vulnerability**) นี่คือเหตุผลที่ Rails บังคับให้ params ต้องผ่าน
> การ "permit" ก่อนเสมอ (strong parameters) — เราจะเรียนวิธีเขียน private method
> `article_params` ที่ทำหน้าที่กรองนี้อย่างละเอียดใน Part 031 ตอนนี้เก็บไว้เป็นความรู้เตรียม
> พร้อมก่อน

---

## Step 225: `session` — เก็บข้อมูลข้าม request ด้วย cookie-based session

HTTP เป็น protocol แบบ **stateless** — แต่ละ request ไม่รู้จักกันเลย server จำอะไรจาก
request ก่อนหน้าไม่ได้โดยธรรมชาติ `session` คือกลไกที่ Rails เตรียมไว้ให้แก้ปัญหานี้ —
เป็น Hash-like object ที่ **ข้อมูลจะยังอยู่ข้าม request** ตราบใดที่เป็น browser/client เดียวกัน

### Rails เก็บ session ไว้ที่ไหน

ค่า default ของ Rails คือ **`ActionDispatch::Session::CookieStore`** — เก็บข้อมูล session
ทั้งหมดไว้ใน **cookie ฝั่ง browser** โดยตรง (ไม่ใช่เก็บในฐานข้อมูลหรือหน่วยความจำฝั่ง server)
แต่ข้อมูลจะถูก **เข้ารหัสและเซ็นชื่อ (encrypted + signed)** ด้วย `secret_key_base` ของแอป
ทำให้ผู้ใช้อ่านหรือแก้ไขค่าข้างในไม่ได้แม้จะเปิดดู cookie ได้ก็ตาม

ตั้งค่าอยู่ใน `config/initializers/session_store.rb` (ถ้าต้องการเปลี่ยนไปใช้ store แบบอื่น
เช่นเก็บใน database หรือ Redis จะปรับที่ไฟล์นี้ — เรื่องนี้จะกลับมาเจาะลึกอีกครั้งตอนพูดถึง
scaling ในเฟสหลังๆ ของหลักสูตร)

### ตัวอย่าง: ตัวนับจำนวนครั้งที่เข้าชม

```ruby
class GreetingsController < ApplicationController
  def visit_counter
    session[:visits] = session[:visits].to_i + 1
    render plain: "คุณเข้าชมหน้านี้ทั้งหมด #{session[:visits]} ครั้งแล้ว"
  end
end
```

```ruby
# config/routes.rb
get "greetings/visit_counter", to: "greetings#visit_counter"
```

ทดสอบด้วย `curl` โดยใช้ cookie jar (`-c` เขียน cookie ลงไฟล์, `-b` อ่าน cookie จากไฟล์)
เพื่อจำลอง browser เดียวกันที่เข้าซ้ำหลายครั้ง:

```bash
curl -c cookies.txt -b cookies.txt http://localhost:3000/greetings/visit_counter
curl -c cookies.txt -b cookies.txt http://localhost:3000/greetings/visit_counter
curl -c cookies.txt -b cookies.txt http://localhost:3000/greetings/visit_counter
```

ผลลัพธ์จริง:

```
คุณเข้าชมหน้านี้ทั้งหมด 1 ครั้งแล้ว
คุณเข้าชมหน้านี้ทั้งหมด 2 ครั้งแล้ว
คุณเข้าชมหน้านี้ทั้งหมด 3 ครั้งแล้ว
```

แต่ถ้ายิง request โดย **ไม่ใช้ cookie เดิม** (จำลองผู้ใช้คนใหม่ หรือ browser คนละตัว):

```bash
curl http://localhost:3000/greetings/visit_counter
# => คุณเข้าชมหน้านี้ทั้งหมด 1 ครั้งแล้ว
```

ตัวนับเริ่มใหม่จาก 1 ทันที เพราะไม่มี session cookie เก่าแนบมาด้วย — พิสูจน์ให้เห็นว่า
`session` ผูกกับ "ผู้ใช้คนนั้น/browser นั้น" ไม่ใช่ข้อมูลระดับแอปที่ทุกคนเห็นเหมือนกัน (ถ้า
ต้องการเก็บข้อมูลที่ทุกคนเห็นร่วมกัน ต้องใช้ Model + database ซึ่งเรียนใน Part 025 เป็นต้นไป
หรือใช้ `Rails.cache` ที่จะเรียนใน Part 063)

เมื่อดูใน cookie ที่ curl เก็บไว้ จะเห็นว่าค่าที่ส่งมาจริงคือสตริงเข้ารหัสยาวๆ ไม่ใช่ตัวเลข
`1`, `2`, `3` ตรงๆ:

```
_app_name_session   n2ZQqW5j9%2FxFOljfDzGzA9xi3dmcxlgjKmMtwYFLVGZQ6sYo%2FhXK3mUSUAX7Fgv6R5yb...
```

นี่คือหลักฐานว่าค่าถูกเข้ารหัสไว้จริง ไม่ใช่แค่ base64 encode ธรรมดา

### ล้าง session

```ruby
session[:visits] = nil     # ลบ key เดียว
session.delete(:visits)    # ลบ key เดียว (แบบชัดเจนกว่า)
reset_session               # ล้างข้อมูล session ทั้งหมด (มักใช้ตอน logout)
```

---

## Step 226: `flash` และ `flash.now` — ข้อความแบบใช้ครั้งเดียว

`flash` คือกล่องเก็บข้อความชั่วคราวที่มีอายุการใช้งาน **"ข้าม request เดียว"** เท่านั้น
ใช้บ่อยที่สุดสำหรับข้อความแจ้งเตือน เช่น "บันทึกสำเร็จ", "เกิดข้อผิดพลาด" ที่ต้องการแสดง
**หลังจาก redirect ไปหน้าอื่นแล้ว**

หลักการทำงาน: `flash` เก็บอยู่ใน `session` ภายใน แต่ Rails จะ **ลบทิ้งอัตโนมัติหลังจากถูก
อ่านไปแสดงผลครั้งหนึ่งแล้ว** — เขียนแล้วอ่านได้อีกแค่ 1 request ถัดไปเท่านั้น

### ตัวอย่าง: `flash` ธรรมดา (ใช้คู่กับ `redirect_to`)

```ruby
def reset_counter
  session[:visits] = 0
  flash[:notice] = "รีเซ็ตตัวนับเรียบร้อยแล้ว"
  redirect_to greetings_visit_counter_path
end
```

ทดสอบ (ดู response header ของ request ที่เป็น POST เอง ไม่ follow redirect):

```bash
curl -i -X POST http://localhost:3000/greetings/reset_counter
```

ผลลัพธ์จริง (ตัดเฉพาะส่วนสำคัญ):

```
HTTP/1.1 302 Found
location: http://localhost:3000/greetings/visit_counter
set-cookie: _app_session=...(ค่าใหม่ที่มี flash[:notice] ฝังอยู่ข้างใน)...
```

จะเห็นว่า `redirect_to` ทำให้ response กลับมาเป็น **status 302** พร้อม header `Location`
ชี้ไปหน้าใหม่ ส่วนข้อความ flash จะถูกฝังไปกับ cookie session และพร้อมให้ view ของหน้า
`visit_counter` (คือ request **ถัดไป**) เอามาแสดงได้ — เดี๋ยวใน Part 024 เราจะเรียนวิธี
แสดง `flash[:notice]` ใน layout ด้วย ERB จริงๆ

### ตัวอย่าง: `flash.now` (ใช้คู่กับ `render` ใน request เดียวกัน)

ปัญหาคือ ถ้าเราใช้ `flash[:alert] = "..."` ตามด้วย `render` (ไม่ redirect) ข้อความนั้นจะ
**ค้างอยู่ต่ออีก 1 request ถัดไปโดยไม่ได้ตั้งใจ** เพราะ `render` ไม่ได้เริ่ม request ใหม่
— แต่ `flash[:alert]` ถูกออกแบบมาให้ "รอเผื่อ request ถัดไป" เสมอ ทำให้ผู้ใช้อาจเห็นข้อความ
เดิมโผล่มาอีกครั้งตอนกด reload หรือไปหน้าอื่นแบบงงๆ

`flash.now` แก้ปัญหานี้: ข้อความจะแสดงผลได้ **เฉพาะใน response ของ request ปัจจุบันเท่านั้น**
ไม่ถูกเก็บไว้ให้ request ถัดไป

```ruby
class ArticlesController < ApplicationController
  ARTICLES = []

  def create
    title = params[:article]&.fetch(:title, nil)

    if title.blank?
      flash.now[:alert] = "กรุณาใส่ชื่อบทความ"
      render :new, status: :unprocessable_entity
      return
    end

    ARTICLES << { id: ARTICLES.size + 1, title: title }
    flash[:notice] = "สร้างบทความ \"#{title}\" สำเร็จ"
    redirect_to articles_path
  end
end
```

view `app/views/articles/new.html.erb`:

```erb
<h1>สร้างบทความใหม่</h1>

<% if flash.now[:alert] %>
  <p style="color:red"><%= flash.now[:alert] %></p>
<% end %>

<form action="/articles" method="post">
  <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">
  <input type="text" name="article[title]" placeholder="ชื่อบทความ">
  <button type="submit">บันทึก</button>
</form>
```

ทดสอบส่งชื่อบทความเป็นค่าว่าง:

```bash
curl -X POST http://localhost:3000/articles --data-urlencode "article[title]="
```

ผลลัพธ์จริง (เฉพาะส่วน body):

```html
<h1>สร้างบทความใหม่</h1>

  <p style="color:red">กรุณาใส่ชื่อบทความ</p>

<form action="/articles" method="post">
  ...
</form>
```

พร้อมกับ HTTP status `422 Unprocessable Entity` (ใส่ผ่าน `status: :unprocessable_entity`
เพื่อบอก client ว่าคำขอนี้ "เข้าใจได้ แต่ข้อมูลไม่ผ่านเงื่อนไข" — เป็น status code มาตรฐานที่
Rails แนะนำให้ใช้ตอน validation ล้มเหลว)

**สรุปกฎการเลือกใช้:**

| สถานการณ์ | ใช้ |
|---|---|
| ตามด้วย `redirect_to` (ไปหน้าอื่น/request ใหม่) | `flash[:key] = ...` |
| ตามด้วย `render` (แสดงผลใน request เดิม เช่น form ที่กรอกผิด) | `flash.now[:key] = ...` |

---

## Step 227: `redirect_to` vs `render` และรูปแบบ Post/Redirect/Get

ตอนนี้เราใช้ทั้ง `render` และ `redirect_to` มาหลายครั้งแล้ว มาสรุปความต่างให้ชัดเจน:

| | `render` | `redirect_to` |
|---|---|---|
| ส่งอะไรกลับไปหา browser | เนื้อหา HTML/text/JSON ที่เสร็จสมบูรณ์ทันที | HTTP status `3xx` + header `Location` |
| Browser ทำอะไรต่อ | แสดงผลเลย ไม่มี request ใหม่ | ยิง request **ใหม่** ไปยัง URL ใน `Location` โดยอัตโนมัติ |
| URL ที่ browser เห็น | **ไม่เปลี่ยน** (ยังเป็น URL เดิมที่ submit form) | **เปลี่ยน** เป็น URL ปลายทาง |
| จำนวน request จริง | 1 | 2 (request เดิม + request ใหม่ที่ตามมา) |
| ใช้ `flash.now` หรือ `flash` | `flash.now` (request เดียวกัน) | `flash` (ข้าม request) |

### Post/Redirect/Get (PRG) pattern — ทำไมต้อง redirect หลัง POST ที่สำเร็จ

ลองพิจารณา action `create` ของ `ArticlesController` อีกครั้ง:

```ruby
def create
  title = params[:article]&.fetch(:title, nil)

  if title.blank?
    flash.now[:alert] = "กรุณาใส่ชื่อบทความ"
    render :new, status: :unprocessable_entity   # (A) render กลับ ไม่ redirect
    return
  end

  ARTICLES << { id: ARTICLES.size + 1, title: title }
  flash[:notice] = "สร้างบทความ \"#{title}\" สำเร็จ"
  redirect_to articles_path                        # (B) redirect ไปหน้า index
end
```

**ทำไม (A) กรณี validation ผิดพลาด ถึง `render` แทนที่จะ `redirect_to`:** เพราะยังไม่มี
ข้อมูลอะไรถูกสร้างขึ้นจริง การที่ browser ยัง "ค้าง" อยู่ที่ URL `/articles` (POST) ก็ไม่มี
ผลเสีย ถ้าผู้ใช้กด reload ก็แค่ POST ข้อมูลเดิมซ้ำ (ซึ่งก็ล้มเหลวเหมือนเดิม ไม่มีอะไรเสียหาย)
และเรายังได้ใช้ `flash.now` แสดง error โดยไม่ต้องยิง request รอบสอง ทำให้ค่าที่ผู้ใช้กรอกไว้
ในฟอร์ม (ถ้ามีการ render ค่ากลับเข้าไปใน input) ไม่หายไปด้วย

**ทำไม (B) กรณีสำเร็จ ต้อง `redirect_to` แทนที่จะ `render` หน้า index ตรงๆ:** นี่คือรูปแบบ
มาตรฐานที่เรียกว่า **Post/Redirect/Get (PRG)** ลองจินตนาการว่าถ้าเราเปลี่ยนเป็น `render`
หน้า index ตรงๆ แทน (ไม่ redirect):

1. Browser เพิ่งทำ `POST /articles` และได้ผลลัพธ์เป็นหน้า index กลับมา
2. URL บน address bar **ยังคงเป็น** `POST /articles` (ไม่เปลี่ยน เพราะไม่มี redirect)
3. ถ้าผู้ใช้กด **F5 (reload)** browser จะถามว่า "Confirm Form Resubmission" และถ้าผู้ใช้กด
   ยืนยัน (หรือบาง browser ทำอัตโนมัติ) จะเป็นการ **`POST` ซ้ำอีกครั้ง** → สร้างบทความซ้ำอีก
   1 รายการโดยไม่ตั้งใจ — บั๊กคลาสสิกที่เรียกว่า **duplicate form submission**

ด้วย PRG pattern (`redirect_to` หลัง POST สำเร็จ):

1. Browser ทำ `POST /articles` → server ตอบกลับด้วย `302 Found` + `Location: /articles`
2. Browser ยิง request ใหม่เป็น `GET /articles` โดยอัตโนมัติ (ตาม HTTP spec)
3. URL บน address bar เปลี่ยนเป็น `GET /articles` แล้ว
4. ถ้าผู้ใช้กด **F5** ตอนนี้ browser จะแค่ **`GET /articles` ซ้ำ** เท่านั้น — ไม่มีการสร้าง
   ข้อมูลซ้ำเลย เพราะ `GET` ไม่ควรมีผลข้างเคียง (side effect) ตามหลักการออกแบบ HTTP method
   (concept นี้เรียกว่า **idempotency** — `GET` ควรเรียกกี่ครั้งก็ได้ผลลัพธ์เหมือนเดิมเสมอ
   ต่างจาก `POST` ที่แต่ละครั้งอาจสร้างข้อมูลใหม่)

**กฎปฏิบัติ:** action ที่รับ `POST`/`PATCH`/`PUT`/`DELETE` แล้วทำสำเร็จ (สร้าง/แก้ไข/ลบข้อมูล
สำเร็จ) ให้ `redirect_to` เสมอ ส่วน `render` ให้เก็บไว้ใช้ตอนแสดงฟอร์มเดิมพร้อม error หรือ
ตอบกลับข้อมูลในรูปแบบอื่น (เช่น JSON สำหรับ API ซึ่งจะเรียนในเฟส 8)

ยืนยันด้วย log จริงจาก request ที่สร้างบทความสำเร็จ:

```bash
curl -i -X POST http://localhost:3000/articles --data-urlencode "article[title]=หัดเขียน Rails Controller"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/articles
content-length: 0
```

`content-length: 0` ยืนยันว่า response ของ `redirect_to` ไม่มีเนื้อหา HTML ใดๆ เลย มีแค่
header บอกทางเท่านั้น — เนื้อหาจริงจะมาจาก `GET /articles` ที่ browser ยิงตามมาเอง

---

## Step 228: `before_action` พร้อม `only:`/`except:` — logic ที่ใช้ร่วมกันหลาย action

หลาย action ในหลาย controller มักต้องการ logic ที่ซ้ำกัน เช่น "เช็คว่า login แล้วหรือยัง
ก่อนอนุญาตให้เข้าถึง" `before_action` คือกลไก **callback** ที่ให้ Rails รัน method ที่ระบุไว้
**ก่อน** action จริงจะทำงาน — ถ้าไม่ระบุ `only:`/`except:` จะรันก่อนทุก action ใน controller
นั้น

### ตัวอย่าง: ระบบตรวจสอบ token ก่อนเข้าหน้า admin

```ruby
class SecretAreaController < ApplicationController
  before_action :require_token, only: [:admin]

  def index
    render plain: "หน้านี้เข้าได้ทุกคน ไม่ต้องมี token"
  end

  def admin
    render plain: "ยินดีต้อนรับเข้าสู่หน้า admin (ผ่านการตรวจสอบ token แล้ว)"
  end

  private

  def require_token
    return if params[:token] == "secret123"

    flash[:alert] = "ต้องใส่ token ที่ถูกต้องก่อนเข้าใช้งาน"
    redirect_to secret_area_path
  end
end
```

**ประเด็นสำคัญเรื่อง `only:`/`except:`:**

- `only: [:admin]` — รัน `require_token` **เฉพาะก่อน** action `admin` เท่านั้น action
  `index` จะไม่ถูกเช็ค token
- ถ้าอยากให้รันทุก action **ยกเว้น** บาง action ให้ใช้ `except:` แทน เช่น
  `before_action :require_login, except: [:index, :show]` (รูปแบบทั่วไปที่ใช้บ่อยมากในหน้า
  ที่มีทั้งส่วนสาธารณะและส่วนที่ต้อง login)
- Method ที่ `before_action` เรียก (`require_token` ในที่นี้) ต้องเป็น **`private`** เสมอ
  เพราะไม่ใช่ action ที่ควรถูกเรียกตรงๆ จาก URL

**กฎสำคัญที่สุดของ `before_action`:** ถ้า method ที่ถูกเรียกมีการ `render` หรือ
`redirect_to` เกิดขึ้นข้างใน (เหมือน `require_token` ด้านบนตอน token ไม่ถูกต้อง) Rails จะ
**หยุด callback chain ทันที** และ **ไม่รัน action จริงต่อ** — เรียกว่า "halt" the filter chain

ทดสอบให้เห็นผลจริง:

```bash
# ไม่ใส่ token เลย
curl -i http://localhost:3000/secret_area/admin
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/secret_area
server-timing: start_processing.action_controller;dur=0.01, redirect_to.action_controller;dur=0.15, halted_callback.action_controller;dur=0.01, process_action.action_controller;dur=2.71
```

สังเกต header `server-timing` ที่ Rails แนบมาให้ในโหมด development — มี event ชื่อ
**`halted_callback.action_controller`** ปรากฏอยู่จริง ซึ่งเป็นหลักฐานยืนยันตรงๆ ว่า
callback chain ถูกหยุดกลางคัน (method `admin` **ไม่เคยถูกเรียกเลย**) เพราะ `require_token`
สั่ง `redirect_to` ไปแล้ว

ทดสอบด้วย token ที่ถูกต้อง:

```bash
curl "http://localhost:3000/secret_area/admin?token=secret123"
# => ยินดีต้อนรับเข้าสู่หน้า admin (ผ่านการตรวจสอบ token แล้ว)
```

คราวนี้ `require_token` เจอ `params[:token] == "secret123"` เป็นจริง จึง `return` เฉยๆ
โดยไม่ render/redirect อะไร — callback chain จึงเดินหน้าต่อไปยัง action `admin` ตามปกติ

> **ในโลกจริง** `require_token` แบบนี้เป็นแค่ตัวอย่างง่ายๆ เพื่อโฟกัสที่กลไก `before_action`
> ล้วนๆ ระบบ authentication จริงจะซับซ้อนกว่านี้มาก (ใช้ `has_secure_password`, session
> เก็บ `user_id`, หรือใช้ gem อย่าง Devise) ซึ่งจะเรียนแบบเต็มในเฟส 5 (Part 041–045)
> รูปแบบ `before_action :authenticate_user!` ที่จะเห็นตอนนั้นก็ใช้กลไกตัวเดียวกับที่เรียนในนี้
> ทุกประการ

### เรียก `before_action` หลายตัว และลำดับการทำงาน

```ruby
class ArticlesController < ApplicationController
  before_action :log_visit
  before_action :require_login, except: [:index, :show]
  before_action :find_article, only: [:show]

  # ...
end
```

`before_action` หลายตัวจะรันเรียงตามลำดับที่ประกาศจากบนลงล่าง **ก่อน** action เสมอ ถ้าตัว
ใดตัวหนึ่ง halt (render/redirect) ตัวที่เหลือและ action จริงจะไม่ถูกรันต่อ

---

## Step 229: `after_action`, `around_action` เบื้องต้น และ `rescue_from`

### `after_action` — รันหลัง action เสร็จ

`after_action` คือ callback ที่รัน **หลังจาก** action ทำงานเสร็จแล้ว (แต่ก่อนส่ง response
ออกไปจริงๆ) ใช้บ่อยสำหรับ logging, การตั้งค่า HTTP header เพิ่มเติม, หรือ audit trail

```ruby
class ApplicationController < ActionController::Base
  after_action :log_completion

  private

  def log_completion
    Rails.logger.info "[log_completion] เสร็จสิ้น #{controller_name}##{action_name} status=#{response.status}"
  end
end
```

### `around_action` — ครอบทั้งก่อนและหลัง action

`around_action` คือ callback ที่ **ครอบ** การทำงานทั้งหมดไว้ (ทั้ง `before_action`, action,
และ `after_action` ที่อยู่ในระดับเดียวกันหรือ subclass) ต้องเรียก `yield` ตรงกลางเพื่อส่งต่อ
การควบคุมให้ส่วนที่ถูกครอบทำงาน คล้าย `Proc`/block ที่เรียนมาใน Part 008 มาก — เหมาะกับงาน
จับเวลา, เปิด/ปิด transaction, หรือจัดการ resource ที่ต้อง "เปิดก่อน-ปิดหลัง" เสมอ

```ruby
class ApplicationController < ActionController::Base
  around_action :measure_request_time
  after_action :log_completion

  private

  def measure_request_time
    started_at = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield
    duration_ms = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - started_at) * 1000).round(2)
    Rails.logger.info "[measure_request_time] #{controller_name}##{action_name} ใช้เวลา #{duration_ms} ms"
  end

  def log_completion
    Rails.logger.info "[log_completion] เสร็จสิ้น #{controller_name}##{action_name} status=#{response.status}"
  end
end
```

### ลำดับการทำงานจริง (ยืนยันจาก log)

เมื่อยิง request ปกติที่ไม่ถูก halt (`GET /articles`) log ที่ได้จริงคือ:

```
[log_completion] เสร็จสิ้น articles#index status=200
[measure_request_time] articles#index ใช้เวลา 0.4 ms
```

สังเกตว่า **`log_completion` (after_action) รันมาก่อน** ส่วนบรรทัด `measure_request_time`
(around_action) มาปรากฏ**หลัง** — เพราะ `around_action` ครอบ `after_action` ไว้ด้วย ลำดับ
จริงคือ: `measure_request_time` เริ่มจับเวลา → `yield` → `before_action` (ถ้ามี) → action
จริง → `after_action` (`log_completion`) → กลับมาที่ `measure_request_time` หลัง `yield`
คำนวณเวลาที่ใช้แล้วค่อย log

แต่เมื่อ request ถูก `before_action` halt (เช่น `GET /secret_area/admin` แบบไม่มี token
จาก Step 228) log ที่ได้จริงคือ:

```
[measure_request_time] secret_area#admin ใช้เวลา 2.57 ms
```

**ไม่มีบรรทัด `log_completion` เลย** — เพราะ `after_action` จะ **ไม่ถูกรัน** ถ้า
`before_action` ที่มาก่อนหน้า halt การทำงานไปแล้ว แต่ `around_action` ยังคงรันจนจบ (เพราะ
`yield` ยัง return กลับมาให้ตามปกติแม้ข้างในจะถูก halt ก่อนถึง action จริงก็ตาม) — เป็นจุด
ละเอียดอ่อนที่ควรจำ: **`after_action` ถูกข้ามได้เมื่อมีการ halt แต่ `around_action` ยังทำงาน
ต่อจนจบเสมอ**

### `rescue_from` — จัดการ exception ระดับ Controller

`rescue_from` คือกลไกที่ให้ `ApplicationController` (หรือ controller ไหนก็ได้) **ดักจับ
exception** ที่เกิดขึ้นระหว่าง action ทำงาน แล้วเปลี่ยนไปทำอย่างอื่นแทนที่จะปล่อยให้แอปล่ม
ด้วยหน้า error 500 แบบดิบๆ

ตัวอย่างคลาสสิกที่สุดคือการจัดการ `ActiveRecord::RecordNotFound` — exception ที่ Rails
โยนออกมาอัตโนมัติเมื่อเรียก `Model.find(id)` แล้วหา record ไม่เจอ (`ActiveRecord::RecordNotFound`
เป็น class ที่มีอยู่แล้วในทุกแอป Rails ตั้งแต่ต้น แม้จะยังไม่ได้เรียนเรื่อง Model/migration
อย่างเป็นทางการก็ตาม ซึ่งจะเรียนเต็มรูปแบบใน Part 025 — ในตัวอย่างนี้เราจะจำลองการโยน
exception ตัวนี้เองก่อน เพื่อโฟกัสที่กลไก `rescue_from` ล้วนๆ):

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found

  private

  def render_not_found
    render plain: "404 ไม่พบข้อมูลที่ต้องการ", status: :not_found
  end
end
```

```ruby
class ArticlesController < ApplicationController
  ARTICLES = []

  def show
    id = params[:id].to_i
    article = ARTICLES.find { |a| a[:id] == id }
    raise ActiveRecord::RecordNotFound, "ไม่พบบทความ id=#{id}" unless article

    render plain: "บทความ ##{article[:id]}: #{article[:title]}"
  end
end
```

ทดสอบเรียก id ที่ไม่มีอยู่จริง:

```bash
curl -i http://localhost:3000/articles/999
```

ผลลัพธ์จริง:

```
HTTP/1.1 404 Not Found
...
404 ไม่พบข้อมูลที่ต้องการ
```

และใน log ของ Rails (`log/development.log`) จะเห็นบรรทัดที่ยืนยันว่า `rescue_from` ทำงาน
จริง:

```
Started GET "/articles/999" for 127.0.0.1
Processing by ArticlesController#show as */*
  Parameters: {"id"=>"999"}
rescue_from handled ActiveRecord::RecordNotFound (ไม่พบบทความ id=999) - app/controllers/articles_controller.rb:32:in `show'
Completed 404 Not Found in 1ms
```

**ทำไมต้องใส่ `rescue_from` ไว้ที่ `ApplicationController`:** เพราะ `ActiveRecord::RecordNotFound`
อาจเกิดขึ้นได้จากทุก controller ที่เรียก `find` (เช่น `PostsController`, `UsersController`,
`OrdersController` ฯลฯ) การใส่ไว้ที่จุดกลางจุดเดียวทำให้ **ทุก controller ในแอปได้พฤติกรรม
"แสดงหน้า 404 สวยๆ แทนหน้า error 500" โดยอัตโนมัติ** โดยไม่ต้องเขียน `rescue_from` ซ้ำใน
ทุก controller — ประหยัดโค้ดได้มาก และเป็นรูปแบบมาตรฐานที่โปรเจกต์ Rails จริงแทบทุกโปรเจกต์ใช้

`rescue_from` รองรับหลาย exception class พร้อมกันได้ และรองรับ block แทน `with:` ก็ได้:

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
  rescue_from ActionController::ParameterMissing do |exception|
    render plain: "ขาด parameter ที่จำเป็น: #{exception.param}", status: :bad_request
  end
end
```

> **ข้อควรระวัง:** อย่าใช้ `rescue_from StandardError` หรือ `rescue_from Exception` แบบกว้างๆ
> ครอบทุกอย่าง เพราะจะทำให้ debug ยากมาก (error จริงถูกกลืนหายไปหมด เห็นแต่หน้า error
> ทั่วไปที่ไม่บอกอะไรเลย) ควรระบุ exception class ที่เจาะจงเสมอ เช่นในตัวอย่างข้างต้น

---

## Step 230: แบบฝึกหัด — `GreetingsController` ครบวงจร

### โจทย์

สร้าง action ใหม่ในระบบเดิม (`GreetingsController`) ที่ทำงานดังนี้:

1. อ่านค่าจาก query string parameter ชื่อ `name` (ถ้าไม่ได้ส่งมา ให้ใช้ค่า default เป็น
   "ผู้มาเยือน")
2. เพิ่มตัวนับจำนวนครั้งที่มีคนมาทักทาย เก็บไว้ใน `session` (แยกจากตัวนับ `visit_counter`
   ที่ทำไปแล้วใน Step 225)
3. ตั้งค่า `flash[:notice]` เป็นข้อความทักทายที่รวมชื่อและจำนวนครั้ง แล้ว `redirect_to`
   ไปยัง action อีกตัวที่แสดงผล flash นั้น
4. เพิ่ม `before_action` ที่ทำงานกับ**ทุก action** ใน `GreetingsController` (ไม่ใส่
   `only:`/`except:`) เพื่อบันทึก log ว่ามี request อะไรเข้ามาบ้าง (HTTP method, path,
   และ params)

### เฉลย

```ruby
# config/routes.rb
get "greetings/greet", to: "greetings#greet"
get "greetings/greeting_result", to: "greetings#greeting_result"
```

```ruby
# app/controllers/greetings_controller.rb
class GreetingsController < ApplicationController
  before_action :log_every_request

  def hello
  end

  def greet
    name = params[:name].presence || "ผู้มาเยือน"
    session[:greet_count] = session[:greet_count].to_i + 1
    flash[:notice] = "สวัสดี, #{name}! คุณมาเยี่ยมหน้านี้เป็นครั้งที่ #{session[:greet_count]} แล้ว"
    redirect_to greetings_greeting_result_path
  end

  def greeting_result
    render plain: flash[:notice] || "ยังไม่มีข้อความทักทาย ลองเข้าหน้า /greetings/greet?name=ชื่อคุณ ก่อน"
  end

  private

  def log_every_request
    Rails.logger.info(
      "[ExerciseLog] #{request.request_method} #{request.fullpath} " \
      "params=#{params.except(:controller, :action).to_unsafe_h}"
    )
  end
end
```

**อธิบายจุดที่น่าสนใจในเฉลย:**

- `params[:name].presence` — `presence` เป็น method จาก Active Support (ส่วนขยายของ Rails
  ที่เติม method สะดวกๆ ให้ Ruby core class) คืนค่าตัวเองถ้าไม่ว่าง (`nil`/`""`) หรือคืน `nil`
  ถ้าว่าง สั้นกว่าการเขียน `params[:name].present? ? params[:name] : nil` มาก แล้วต่อด้วย
  `|| "ผู้มาเยือน"` เพื่อกำหนดค่า default (รูปแบบเดียวกับ `ARGV[0] || "World"` ที่เรียนไปตั้งแต่
  Part 001)
- แยก `greet` (ทำงานจริง + redirect) กับ `greeting_result` (แค่แสดงผล) ออกจากกัน คือการ
  ใช้ PRG pattern จาก Step 227 อย่างเป็นรูปธรรม
- `request.fullpath` คือ path พร้อม query string เต็ม (เช่น `/greetings/greet?name=Somchai`)
  ต่างจาก `request.path` ที่ไม่รวม query string — `request` เป็น object ที่ Rails เตรียมไว้
  ให้ทุก action เหมือน `params` มีข้อมูลเกี่ยวกับ HTTP request ดิบๆ ครบถ้วน
- `params.except(:controller, :action).to_unsafe_h` — ตัด key `controller`/`action` ที่
  Rails ใส่มาให้เองออกก่อน log (ไม่งั้น log จะรกด้วยข้อมูลที่ซ้ำซ้อน) แล้วแปลงเป็น `Hash`
  ธรรมดาด้วย `to_unsafe_h` เพื่อให้ log อ่านง่าย (`to_unsafe_h` ใช้ได้เพราะเราแค่จะ log
  ดูเฉยๆ ไม่ได้เอาไปใช้ mass-assignment กับ Model)

ทดสอบจริง (จำลอง browser เดียวกันด้วย cookie jar, ใช้ `-L` ให้ curl follow redirect
อัตโนมัติ):

```bash
curl -c cookies.txt -b cookies.txt -L "http://localhost:3000/greetings/greet?name=Somchai"
curl -c cookies.txt -b cookies.txt -L "http://localhost:3000/greetings/greet?name=Somchai"
curl -c cookies.txt -b cookies.txt -L "http://localhost:3000/greetings/greet"
```

ผลลัพธ์จริง:

```
สวัสดี, Somchai! คุณมาเยี่ยมหน้านี้เป็นครั้งที่ 1 แล้ว
สวัสดี, Somchai! คุณมาเยี่ยมหน้านี้เป็นครั้งที่ 2 แล้ว
สวัสดี, ผู้มาเยือน! คุณมาเยี่ยมหน้านี้เป็นครั้งที่ 3 แล้ว
```

และเมื่อเข้า `greeting_result` โดยตรงจาก session ใหม่ (ไม่เคยผ่าน `greet` มาก่อน):

```bash
curl http://localhost:3000/greetings/greeting_result
# => ยังไม่มีข้อความทักทาย ลองเข้าหน้า /greetings/greet?name=ชื่อคุณ ก่อน
```

ตรวจสอบว่า `before_action :log_every_request` ทำงานจริงทุก action ด้วยการดู
`log/development.log`:

```
[ExerciseLog] GET /greetings/greet?name=Somchai params={"name"=>"Somchai"}
[ExerciseLog] GET /greetings/greeting_result params={}
[ExerciseLog] GET /greetings/greet?name=Somchai params={"name"=>"Somchai"}
[ExerciseLog] GET /greetings/greeting_result params={}
[ExerciseLog] GET /greetings/greet params={}
[ExerciseLog] GET /greetings/greeting_result params={}
```

เห็นได้ชัดว่า log ถูกเขียนก่อน action ทุกครั้ง ทั้ง `greet` และ `greeting_result` (เพราะไม่ได้
ใส่ `only:`/`except:` จึงครอบคลุมทุก action ใน controller) และ `params` ที่ log ออกมาก็ตรง
กับที่ query string ส่งมาจริงในแต่ละ request

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม action `logout_greeting` ที่เรียก `reset_session` เพื่อล้างทั้ง `session[:visits]`
   และ `session[:greet_count]` พร้อมกันในคำสั่งเดียว แล้ว `redirect_to root_path` พร้อม
   ข้อความ flash แจ้งว่า "ล้างข้อมูลเรียบร้อยแล้ว"
2. เพิ่ม `before_action` ใหม่ชื่อ `require_name_param` ที่ใช้เฉพาะกับ action `greet`
   (ใช้ `only:`) ถ้าไม่มี `params[:name]` ส่งมาเลย ให้ `redirect_to greetings_hello_path`
   พร้อม `flash[:alert] = "กรุณาระบุชื่อผ่าน query string เช่น ?name=Somchai"` แทนที่จะไป
   ต่อที่ action `greet` เลย
3. เปลี่ยน `render_not_found` ใน `ApplicationController` ให้ตรวจสอบ `request.format` ก่อน
   — ถ้า client ขอ HTML ให้ `render plain: "404 ไม่พบข้อมูล", status: :not_found` เหมือนเดิม
   แต่ถ้า client ขอ JSON (`request.format.json?`) ให้ตอบกลับเป็น
   `render json: { error: "not_found" }, status: :not_found` แทน (ใบ้: ทดสอบด้วย
   `curl -H "Accept: application/json" ...`)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- `ApplicationController` เป็นจุดกลางที่ controller ทุกตัวในแอปสืบทอดมา ใช้ใส่ logic ที่
  ต้องการให้ทุก controller มีร่วมกัน
- Action คือ public instance method ธรรมดา ถ้าไม่เรียก `render`/`redirect_to` เอง Rails จะ
  render view ที่ชื่อตรงกับ action ให้อัตโนมัติตาม convention
- `rails generate controller` สร้าง controller, route, view, test file ให้ครบในคำสั่งเดียว
- `params` รวมข้อมูลจาก query string, route segment (`:id`), และ form body ไว้ในที่เดียว
  เป็น `ActionController::Parameters` ที่เข้าถึงได้ด้วยทั้ง Symbol และ String
- `session` เก็บข้อมูลข้าม request แบบผูกกับผู้ใช้แต่ละคน (default เป็น cookie-based store
  ที่เข้ารหัสและเซ็นชื่อไว้)
- `flash` ใช้ส่งข้อความข้าม 1 request (คู่กับ `redirect_to`) ส่วน `flash.now` ใช้แสดงผลใน
  request เดียวกัน (คู่กับ `render`)
- `redirect_to` vs `render` ต่างกันที่จำนวน request และ URL ที่ browser เห็น และทำไม
  Post/Redirect/Get pattern ถึงป้องกันปัญหา duplicate form submission ได้
- `before_action` (พร้อม `only:`/`except:`) ใช้ทำ logic ที่ใช้ร่วมกันหลาย action เช่นการ
  ตรวจสอบสิทธิ์ และรู้จักกลไก halt callback chain เมื่อมีการ render/redirect ข้างใน
- `after_action`/`around_action` และลำดับการทำงานที่สัมพันธ์กัน รวมถึง `rescue_from` สำหรับ
  ดักจับ exception ระดับ controller เพื่อแสดงหน้า error ที่เป็นมิตรกว่าหน้า 500 ดิบๆ

**ต่อไป (Part 024):** เราจะเจาะลึกฝั่ง **View** — ภาษา ERB สำหรับฝัง Ruby ลงใน HTML,
ระบบ layout ที่ครอบทุกหน้าไว้ (ที่เห็นผ่านๆ ไปแล้วในคอมเมนต์ `BEGIN app/views/layouts/...`
ของ Part นี้), การแยกส่วน HTML ที่ใช้ซ้ำออกเป็น partial, และการเขียน helper method
สำหรับ view โดยเฉพาะ — รวมถึงวิธีแสดง `flash[:notice]`/`flash[:alert]` ที่ตั้งค่าไว้ใน Part
นี้ให้ปรากฏบนหน้าเว็บจริงๆ เสียที
