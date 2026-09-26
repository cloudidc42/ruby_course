# Part 022: Routing — config/routes.rb, resources, RESTful routes, route helpers

> **Step ครอบคลุมใน Part นี้:** Step 211–220
> **ระดับ:** เริ่มต้น–ปานกลาง (ต้องมีพื้นฐาน Ruby ครบเฟส 1–2 และเคยรัน `rails new`/`rails server` มาก่อนจาก Part 021)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x

## สารบัญของ Part นี้

- Step 211: Routing คืออะไร — router ทำหน้าที่อะไรใน request lifecycle ของ Rails
- Step 212: RESTful convention — 7 action มาตรฐาน และทำไมควรยึดตามนี้
- Step 213: `resources :posts` — ประกาศบรรทัดเดียว ได้ 7 routes; อ่านผลลัพธ์ `rails routes`
- Step 214: Route helper — `_path` vs `_url`, การใช้งานใน view/controller/redirect
- Step 215: Custom action บน resource — `member do...end` และ `collection do...end`
- Step 216: Singular resource — `resource :profile` (ไม่มี `:id`)
- Step 217: Nested resources — `resources :posts do resources :comments end` และ `shallow: true`
- Step 218: Namespace vs Scope — `namespace :admin` เทียบกับ `scope module:`
- Step 219: Constraints — จำกัด route ด้วย format, subdomain, และ custom constraint class
- Step 220: Root route, redirect route, และการจัดการ 404 แบบ catch-all

---

## Step 211: Routing คืออะไร — router ทำหน้าที่อะไรใน request lifecycle ของ Rails

ใน Part 021 เราสร้างแอป Rails ได้แล้ว และรู้ว่าเมื่อรัน `bin/rails server` เบราว์เซอร์จะยิง
HTTP request เข้ามาที่แอป แต่คำถามคือ: เมื่อ request มาถึง Rails แล้ว **ใครเป็นคนตัดสินใจว่า
request นี้ควรไปทำงานที่ไฟล์/method ไหน?**

คำตอบคือ **Router** ซึ่งกำหนดไว้ที่ไฟล์ `config/routes.rb` — นี่คือจุดแรกสุดที่ request ทุก
ตัวต้องผ่านก่อนจะไปถึง Controller

### Request lifecycle แบบย่อ (ที่เกี่ยวกับ routing)

```
1. Browser ส่ง HTTP request: GET /posts/5
2. Rails router (config/routes.rb) จับคู่ verb + path กับกฎที่ประกาศไว้
3. ถ้าจับคู่ได้ -> router รู้ว่าต้องเรียก PostsController#show พร้อม params[:id] = "5"
4. ถ้าจับคู่ไม่ได้ -> Rails ตอบกลับ 404 Not Found
5. Controller ทำงาน -> เตรียมข้อมูล -> render View กลับไปเป็น HTML response
```

Router จึงเปรียบเสมือน "สมุดหน้าเหลือง" ที่แมป **(HTTP verb, URL pattern)** ไปยัง
**(controller, action)** หนึ่งคู่เสมอ โดย HTTP verb ที่ใช้บ่อยในเว็บแอปทั่วไปมี 4 ตัว:

| Verb | ความหมายทั่วไป |
|------|----------------|
| `GET` | ขอดูข้อมูล (ไม่ควรมีผลข้างเคียงต่อข้อมูลในระบบ) |
| `POST` | สร้างข้อมูลใหม่ |
| `PATCH` / `PUT` | แก้ไขข้อมูลที่มีอยู่ |
| `DELETE` | ลบข้อมูล |

จุดสำคัญคือ **URL เดียวกัน แต่ verb ต่างกัน สามารถไปคนละ action ได้** เช่น
`GET /posts/5` กับ `DELETE /posts/5` เป็น URL pattern เดียวกัน (`/posts/:id`) แต่ไปคนละที่
(`show` กับ `destroy`) — นี่คือเหตุผลที่ HTML form ปกติทำได้แค่ `GET`/`POST` แต่ Rails ใช้
เทคนิค `_method` hidden field (ผ่าน `form_with`) เพื่อจำลอง `PATCH`/`PUT`/`DELETE` ผ่าน
JavaScript (Rails UJS/Turbo) — รายละเอียดจะพูดถึงเมื่อถึง Part เรื่อง form

### ประกาศ route แบบพื้นฐานที่สุด (ยังไม่ใช้ `resources`)

Syntax พื้นฐานของการประกาศ route คือ:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "welcome/index"
end
```

บรรทัดนี้บอกว่า: เมื่อมี request แบบ `GET /welcome/index` เข้ามา ให้ไปที่
`WelcomeController#index` — Rails เดา controller/action จากชื่อ path เอง
(`welcome/index` -> controller `Welcome`, action `index`)

แต่ถ้าต้องการระบุ controller/action ให้ชัดเจนไม่พึ่งการเดา ให้เขียนแบบนี้แทน:

```ruby
get "welcome/index", to: "welcome#index"

# หรือ syntax แบบ hash (เก่ากว่า แต่ยังพบได้บ่อยในโค้ด legacy)
get "welcome/index" => "welcome#index"
```

### คำสั่งที่ต้องใช้บ่อยที่สุดตลอด Part นี้: `bin/rails routes`

คำสั่งนี้แสดง route ทั้งหมดที่ประกาศไว้ในแอป พร้อมคอลัมน์สำคัญ 4 คอลัมน์:

```bash
bin/rails routes
```

```
            Prefix Verb   URI Pattern               Controller#Action
rails_health_check GET    /up(.:format)             rails/health#show
```

- **Prefix** — ชื่อ route helper (จะพูดถึงใน Step 214) เช่น `rails_health_check` ใช้เรียกผ่าน
  `rails_health_check_path`
- **Verb** — HTTP method ที่ route นี้ตอบสนอง
- **URI Pattern** — รูปแบบ URL ส่วน `(.:format)` หมายถึง "ต่อท้ายด้วย `.json`, `.xml` ฯลฯ
  ได้ (optional)" — เดี๋ยวจะอธิบายเรื่อง format ใน Step 219
- **Controller#Action** — ปลายทางที่ request จะถูกส่งไป

> **หมายเหตุ:** `(.:format)` เป็น optional segment ปกติจะไม่ระบุก็ได้ (Rails จะตอบกลับเป็น HTML
> โดย default) แต่ถ้าอยากได้ JSON กลับมาก็ต่อท้ายด้วย `.json` เช่น `GET /posts/5.json`

ในทุก Step ต่อจากนี้ เราจะเขียน route ใน `config/routes.rb` แล้วรัน `bin/rails routes` เพื่อ
ดูผลลัพธ์จริงเสมอ — เป็นทักษะสำคัญที่สุดของการทำงานกับ routing ใน Rails เพราะช่วยตรวจสอบว่า
สิ่งที่เราคิดว่า route จะออกมาเป็นแบบนั้นจริงหรือไม่ ก่อนจะไปเขียน view/controller ต่อ

---

## Step 212: RESTful convention — 7 action มาตรฐาน และทำไมควรยึดตามนี้

เว็บแอปส่วนใหญ่ทำงานกับ "ทรัพยากร" (resource) เช่น บทความ (post), สินค้า (product),
ผู้ใช้ (user) และการกระทำกับทรัพยากรเหล่านี้ก็ซ้ำแบบเดิมเสมอ: ดูรายการทั้งหมด, ดูรายละเอียด
ทีละอัน, สร้างใหม่, แก้ไข, ลบ

Rails (ตาม REST — Representational State Transfer ที่เสนอโดย Roy Fielding) กำหนด
**convention มาตรฐาน 7 action** สำหรับแต่ละ resource ไว้ล่วงหน้า:

| Action | Verb | URL | ความหมาย |
|--------|------|-----|----------|
| `index` | GET | `/posts` | แสดงรายการ post ทั้งหมด |
| `show` | GET | `/posts/:id` | แสดงรายละเอียด post เดียว |
| `new` | GET | `/posts/new` | แสดงฟอร์มสำหรับสร้าง post ใหม่ |
| `create` | POST | `/posts` | รับข้อมูลจากฟอร์ม แล้วบันทึก post ใหม่ลงฐานข้อมูล |
| `edit` | GET | `/posts/:id/edit` | แสดงฟอร์มสำหรับแก้ไข post ที่มีอยู่ |
| `update` | PATCH/PUT | `/posts/:id` | รับข้อมูลจากฟอร์มแก้ไข แล้วบันทึกการเปลี่ยนแปลง |
| `destroy` | DELETE | `/posts/:id` | ลบ post |

สังเกตว่า:

- `index`/`show`/`new`/`edit` เป็น GET ทั้งหมด เพราะมีหน้าที่ **แสดงผล** อย่างเดียว
  ไม่ควรมีผลข้างเคียงเปลี่ยนแปลงข้อมูล (เรียกว่า "safe" ตามหลัก HTTP)
- `create`/`update`/`destroy` คือ action ที่ **เปลี่ยนแปลงข้อมูลจริง** และมักจะ redirect
  ไปหน้าอื่นหลังทำงานเสร็จ (ไม่ render view ตรงๆ) — รูปแบบนี้เรียกว่า
  **Post/Redirect/Get pattern** ป้องกันปัญหาการ submit form ซ้ำเวลากด refresh
- `new` กับ `create` คู่กัน (แสดงฟอร์ม → รับข้อมูล) เช่นเดียวกับ `edit` กับ `update`

### ทำไมต้องยึด convention นี้ (ทั้งที่จริงๆ จะตั้งชื่อ action อะไรก็ได้)

นี่คือหัวใจของหลักการ **Convention over Configuration** ที่พูดถึงใน Part 001:

1. **นักพัฒนาคนอื่นเข้าใจโค้ดทันที** — เห็น `PostsController` มี action `index/show/new/
   create/edit/update/destroy` ก็รู้ทันทีว่าแต่ละอันทำอะไร โดยไม่ต้องอ่าน implementation
2. **Rails มี helper และ generator ที่ผูกกับ convention นี้โดยตรง** — `resources :posts`
   (Step 213), `form_with model: @post` (จะรู้เองว่าต้อง POST ไป `create` หรือ PATCH ไป
   `update`), scaffold generator (Part 028) — ทั้งหมดนี้ทำงานได้ "ฟรี" เพราะเรายึด convention
3. **ลดการตัดสินใจที่ไม่จำเป็น** — ไม่ต้องมานั่งคิดเองว่าจะตั้งชื่อ action ว่า `list` หรือ
   `all` หรือ `getAll` ดี เพราะมีคำตอบมาตรฐานอยู่แล้ว
4. **Routing file สั้นและอ่านง่าย** — `resources :posts` บรรทัดเดียวแทนที่จะเขียน route
   7 บรรทัดเอง (ดู Step 213)

> **ข้อควรระวัง:** การมี 7 action มาตรฐานไม่ได้แปลว่า **ทุก resource ต้องมีครบทั้ง 7**
> เช่น log entry ที่สร้างจากระบบอัตโนมัติอาจมีแค่ `index` กับ `show` (ดูอย่างเดียว แก้ไข/ลบ
> ไม่ได้) — เดี๋ยว Step 213 จะโชว์วิธีจำกัด action ด้วย `only:`/`except:`

---

## Step 213: `resources :posts` — ประกาศบรรทัดเดียว ได้ 7 routes; อ่านผลลัพธ์ `rails routes`

### ปัญหาถ้าประกาศ route ทีละบรรทัดแบบ manual

ถ้าไม่ใช้ `resources` เราจะต้องเขียนแบบนี้เพื่อให้ได้ครบ 7 action ของ post:

```ruby
# config/routes.rb — วิธีที่ยาวและซ้ำซ้อน (ไม่แนะนำ)
Rails.application.routes.draw do
  get    "posts",          to: "posts#index"
  get    "posts/new",      to: "posts#new"
  post   "posts",          to: "posts#create"
  get    "posts/:id",      to: "posts#show"
  get    "posts/:id/edit", to: "posts#edit"
  patch  "posts/:id",      to: "posts#update"
  put    "posts/:id",      to: "posts#update"
  delete "posts/:id",      to: "posts#destroy"
end
```

ซ้ำซ้อน อ่านยาก และเสี่ยงพิมพ์ผิด (เช่น ลืม `put` คู่กับ `patch`) — ยิ่งมีหลาย resource
ในแอป (posts, comments, users, categories, ...) ก็ยิ่งเขียนซ้ำเยอะขึ้นเรื่อยๆ

### ทางออก: `resources :posts`

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts
end
```

บรรทัดเดียวนี้สร้าง route ครบทั้ง 7 action ให้อัตโนมัติ ตรวจสอบด้วย `bin/rails routes`
(ผลลัพธ์จริงจากการรันบน Rails 8.1.4):

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

สังเกตสิ่งที่เกิดขึ้น:

- ได้ครบ 7 บรรทัด action (นับ `PATCH`/`PUT` สำหรับ `update` เป็น 2 บรรทัดแต่จริงๆ คือ
  action เดียวกัน — Rails รองรับทั้งสอง verb เพราะ HTTP spec เดิมตั้งใจให้ `PUT` ใช้แทนที่
  ทั้ง resource ส่วน `PATCH` ใช้แก้บางส่วน แต่ Rails ปฏิบัติเหมือนกันทั้งคู่)
- ชื่อ path (`posts`, `posts/new`, `posts/:id`, ...) ถูกสร้างจากชื่อ resource
  (`:posts`) โดยอัตโนมัติ เป็นรูปพหูพจน์เสมอ
- คอลัมน์ **Prefix** (`posts`, `new_post`, `edit_post`, `post`) คือชื่อ route helper —
  รายละเอียดเต็มอยู่ Step 214

### ต้องมี controller ที่ตรงกับชื่อ resource เสมอ

`resources :posts` แค่ประกาศ route แต่ไม่ได้สร้าง controller ให้ — เราต้องมี
`app/controllers/posts_controller.rb` ที่มี method รองรับแต่ละ action ที่ route ชี้ไป
มิฉะนั้นจะได้ error `ActionController::RoutingError` หรือ `AbstractController::ActionNotFound`

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
  end

  def show
  end

  def new
  end

  def create
  end

  def edit
  end

  def update
  end

  def destroy
  end
end
```

(รายละเอียดเรื่อง controller/action ทั้งหมดจะสอนใน Part 023 — ตอนนี้โฟกัสที่ routing ก่อน)

### จำกัดจำนวน action ด้วย `only:` / `except:`

ไม่ใช่ทุก resource ต้องการครบ 7 action เสมอไป — บาง resource อาจดูได้อย่างเดียว ไม่มีหน้า
สร้าง/แก้ไข (เช่น log, audit trail) ใช้ `only:` เพื่อระบุเฉพาะ action ที่ต้องการ หรือ
`except:` เพื่อระบุ action ที่ **ไม่** ต้องการ (ทางเลือกทั้งสองแบบให้ผลตรงข้ามกัน
เลือกใช้แบบไหนก็ได้ตามความสั้นกระชับ):

```ruby
resources :posts, only: %i[index show]
# ได้แค่ index, show — ไม่มี new/create/edit/update/destroy

resources :posts, except: %i[destroy]
# ได้ทุก action ยกเว้น destroy
```

ตรวจสอบด้วย `bin/rails routes`:

```
Prefix Verb URI Pattern          Controller#Action
 posts GET  /posts(.:format)     posts#index
  post GET  /posts/:id(.:format) posts#show
```

> **แนวปฏิบัติที่ดี:** ใช้ `only:`/`except:` เสมอเมื่อรู้ล่วงหน้าว่า resource นั้นไม่ต้องการ
> action ครบ 7 ตัว เพราะนอกจากจะทำให้ `rails routes` อ่านง่ายขึ้นแล้ว ยังป้องกันไม่ให้มีใคร
> เข้าถึง URL ที่ไม่ควรมีอยู่ (เช่น ยิง `DELETE /posts/5` ทั้งที่ไม่ได้ตั้งใจเปิดสิทธิ์ลบ)

---

## Step 214: Route helper — `_path` vs `_url`, การใช้งานใน view/controller/redirect

### Route helper คืออะไร

จากผลลัพธ์ `bin/rails routes` ใน Step 213 คอลัมน์ **Prefix** คือชื่อฐานของ **route helper
method** ที่ Rails สร้างให้อัตโนมัติสองตัวต่อหนึ่ง prefix:

- `<prefix>_path` — คืนค่า URL แบบ **relative path** เช่น `/posts/1`
- `<prefix>_url` — คืนค่า URL แบบ **absolute URL** เช่น `http://example.com/posts/1`

ตัวอย่างการรัน `rails runner` เพื่อดูค่าจริงที่ helper เหล่านี้คืนมา (ผลลัพธ์จริงที่ทดสอบ
บน Rails 8.1.4):

```bash
bin/rails runner '
include Rails.application.routes.url_helpers
Rails.application.routes.default_url_options[:host] = "example.com"

puts posts_path        # => /posts
puts posts_url         # => http://example.com/posts
puts post_path(1)      # => /posts/1
puts post_url(1)       # => http://example.com/posts/1
puts new_post_path     # => /posts/new
puts edit_post_path(1) # => /posts/1/edit
'
```

ผลลัพธ์:

```
/posts
http://example.com/posts
/posts/1
http://example.com/posts/1
/posts/new
/posts/1/edit
```

### ทำไมต้องใช้ helper แทนการเขียน string เอง

```ruby
# ไม่ควรทำ — hardcode string URL เอง
link_to "ดูโพสต์", "/posts/#{post.id}"

# ควรทำ — ใช้ route helper
link_to "ดูโพสต์", post_path(post)
```

เหตุผล:

1. **ทนต่อการเปลี่ยนแปลง URL structure** — ถ้าวันหนึ่งเปลี่ยน URL จาก `/posts/:id` เป็น
   `/articles/:id` แค่แก้ที่ `config/routes.rb` ที่เดียว โค้ดที่เรียก `post_path(post)`
   ทุกจุดยังทำงานถูกต้อง (แค่เปลี่ยนชื่อ resource ใน routes.rb เท่านั้น) แต่ถ้า hardcode
   string ไว้ ต้องไล่แก้ทุกที่ที่เขียน `/posts/...`
2. **พิมพ์ผิดจะ error ทันทีตอน boot/test** — ถ้าเขียน `psot_path` (พิมพ์ผิด) Ruby จะโยน
   `NoMethodError` ทันที ต่างจาก string ที่พิมพ์ผิดแล้วรันผ่านแต่ลิงก์พัง (เงียบๆ)
3. **`post_path(post)` รับ object ได้เลย** — ไม่ต้องเขียน `post.id` เอง เพราะ Rails เรียก
   `post.to_param` ให้อัตโนมัติ (ปกติคืนค่า `id.to_s`)

### เมื่อไหร่ใช้ `_path` เมื่อไหร่ใช้ `_url`

กฎง่ายๆ ที่ยึดได้เสมอ:

- **ใช้ `_path` เป็นค่าเริ่มต้นเสมอ** สำหรับ `link_to`, `redirect_to`, `form_with url:`
  ภายในแอปเดียวกัน — เพราะ relative path สั้นกว่า และไม่มีปัญหาเรื่อง http/https หรือ
  domain ผิดพลาด
- **ใช้ `_url` เฉพาะตอนที่ต้องการ absolute URL จริงๆ** เช่น
  - ส่งลิงก์ในอีเมล (Action Mailer) — อีเมลไม่มี "หน้าปัจจุบัน" ให้ relative จาก
  - Redirect ไปยัง URL ภายนอกแอป หรือส่งไปยัง client ผ่าน API/JSON
  - `sitemap.xml`, Open Graph meta tag (`og:url`) ที่ต้องเป็น absolute URL เสมอ

```ruby
# ใน controller — redirect ปกติใช้ _path
redirect_to post_path(@post)
# หรือสั้นกว่า (Rails แปลง object เป็น path helper ให้อัตโนมัติ)
redirect_to @post

# ใน mailer — ต้องใช้ _url เพราะอีเมลไม่มี "โดเมนปัจจุบัน"
class PostMailer < ApplicationMailer
  def notify(post)
    @post_url = post_url(post) # ต้องตั้งค่า default_url_options[:host] ใน config ก่อน
    mail(to: post.author_email, subject: "โพสต์ใหม่ถูกเผยแพร่แล้ว")
  end
end
```

> **หมายเหตุ:** `redirect_to @post` ใช้งานได้เพราะ Rails เรียก polymorphic routing
> (`url_for(@post)`) ซึ่งจะเดา helper ที่ถูกต้องจาก class ของ object (`Post` ->
> `post_path`) ให้อัตโนมัติ — สะดวกมากเมื่อ redirect ไปหน้า show ของ record ที่เพิ่งสร้าง/
> แก้ไข

### กรอง route ด้วย `-g` (grep) และ `-c` (controller) เวลา route เยอะขึ้น

เมื่อแอปโตขึ้น `bin/rails routes` จะแสดง route หลายสิบ/หลายร้อยบรรทัด หาของที่ต้องการยาก
ใช้ flag เหล่านี้ช่วยกรอง (ทดสอบผลลัพธ์จริงแล้ว):

```bash
# กรองเฉพาะ route ที่มีคำว่า "posts" ปรากฏใน prefix, path หรือ controller#action
bin/rails routes -g posts

# กรองเฉพาะ route ของ controller ชื่อ comments
bin/rails routes -c comments
```

---

## Step 215: Custom action บน resource — `member do...end` และ `collection do...end`

7 action มาตรฐานไม่พอเสมอไป เช่น ต้องการปุ่ม "เผยแพร่โพสต์" (`publish`) หรือ "เก็บเข้า
คลัง" (`archive`) ซึ่งเป็นการกระทำกับ post **หนึ่งอัน** ที่เจาะจงด้วย `:id` หรือต้องการหน้า
"ค้นหาโพสต์" (`search`) ที่ทำงานกับ **collection ทั้งหมด** ไม่เจาะจง `:id` ใดๆ

Rails มีสอง block สำหรับกรณีนี้คือ `member` และ `collection`:

```ruby
# config/routes.rb
resources :posts do
  member do
    patch :publish
    patch :archive
  end

  collection do
    get :search
  end
end
```

ตรวจสอบผลลัพธ์จริงด้วย `bin/rails routes`:

```
        Prefix Verb  URI Pattern                  Controller#Action
  publish_post PATCH /posts/:id/publish(.:format) posts#publish
  archive_post PATCH /posts/:id/archive(.:format) posts#archive
  search_posts GET   /posts/search(.:format)      posts#search
         posts GET   /posts(.:format)             posts#index
               ...
```

### วิธีแยกความแตกต่างของ `member` กับ `collection`

| | `member` | `collection` |
|---|----------|--------------|
| ทำงานกับ | record หนึ่งตัว (ต้องมี `:id`) | ทั้ง collection (ไม่มี `:id`) |
| URL pattern ที่ได้ | `/posts/:id/<action>` | `/posts/<action>` |
| Route helper ที่ได้ | `<action>_post_path(post)` | `<action>_posts_path` |
| ตัวอย่างการใช้งานจริง | publish, archive, like, duplicate | search, export, bulk_destroy |

จำง่ายๆ: ถ้า action นั้นต้องรู้ว่า "โพสต์**ไหน**" ถึงจะทำงานได้ (ต้องมี `:id`) ให้ใช้
`member` แต่ถ้า action ทำงานกับโพสต์ทั้งชุดพร้อมกัน (ไม่เจาะจงตัวใดตัวหนึ่ง) ให้ใช้
`collection`

### ประกาศแบบบรรทัดเดียวเมื่อมีแค่ 1 custom action

ถ้ามี custom action แค่ตัวเดียว ไม่จำเป็นต้องเปิด block ก็ได้ ใช้ `on:` แทน:

```ruby
resources :posts do
  patch :publish, on: :member
  get   :search,  on: :collection
end
```

ให้ผลลัพธ์เหมือนกับการใช้ `member do...end` / `collection do...end` ทุกประการ — เลือกใช้
รูปแบบไหนก็ได้ แต่ถ้ามี custom action มากกว่า 1 ตัว การเปิด block จะอ่านง่ายกว่า

### การเรียกใช้ route helper ที่ได้จาก member/collection

```erb
<%# app/views/posts/show.html.erb %>
<%= button_to "เผยแพร่", publish_post_path(@post), method: :patch %>

<%# app/views/posts/index.html.erb %>
<%= form_with url: search_posts_path, method: :get do |f| %>
  <%= f.text_field :q %>
  <%= f.submit "ค้นหา" %>
<% end %>
```

> **ข้อควรระวัง:** custom action ควรใช้อย่างประหยัด — ถ้าพบว่า controller หนึ่งเริ่มมี
> custom action เยอะขึ้นเรื่อยๆ (publish, archive, feature, pin, ...) มักเป็นสัญญาณว่า
> ควรแยกเป็น resource ใหม่แทน เช่น `resources :posts do resource :publication end`
> (แนวคิดนี้เรียกว่า "verb เป็น noun" — จะพูดถึงเพิ่มเติมเมื่อถึงเรื่อง Service Object ใน
> เฟส 14)

---

## Step 216: Singular resource — `resource :profile` (ไม่มี `:id`)

บาง resource ในระบบมีอยู่แค่ "หนึ่งเดียว" ต่อผู้ใช้หนึ่งคน (หรือต่อ context หนึ่งอัน) เช่น
โปรไฟล์ของผู้ใช้ที่ login อยู่ (`/profile` ไม่ใช่ `/profiles/5`) — ในกรณีนี้ **ไม่ต้องมี
`:id` ใน URL เลย** เพราะ Rails รู้อยู่แล้วว่า "โปรไฟล์" หมายถึงของ `current_user` เสมอ

ใช้ `resource` (เอกพจน์ ไม่มี `s`) แทน `resources`:

```ruby
# config/routes.rb
resource :profile, only: %i[show edit update]
```

ตรวจสอบผลลัพธ์จริง:

```
      Prefix Verb  URI Pattern            Controller#Action
edit_profile GET   /profile/edit(.:format) profiles#edit
     profile GET   /profile(.:format)      profiles#show
             PATCH /profile(.:format)      profiles#update
             PUT   /profile(.:format)      profiles#update
```

สังเกตความแตกต่างจาก `resources :posts` อย่างชัดเจน:

1. **ไม่มี `:id` ใน URL pattern เลย** — `/profile` ไม่ใช่ `/profile/:id`
2. **ไม่มี index** — เพราะมีแค่หนึ่งเดียว ไม่มี "รายการทั้งหมด" ให้แสดง
3. **route helper ไม่มี `s` ต่อท้าย และไม่ต้องส่ง object/id เข้าไป** — เป็น
   `profile_path` (ไม่ใช่ `profiles_path` หรือ `profile_path(id)`)

```ruby
# ใช้งาน route helper ของ singular resource
redirect_to profile_path       # ถูกต้อง — ไม่ต้องส่ง argument
redirect_to edit_profile_path  # ถูกต้อง
```

### ข้อควรระวัง: ชื่อ controller ยังคงเป็น **พหูพจน์** ตาม convention เดิม

แม้ URL และ route helper จะเป็นเอกพจน์ (`/profile`, `profile_path`) แต่ **controller ที่
ต้องมีจริงยังคงเป็นชื่อพหูพจน์** คือ `ProfilesController` (ไม่ใช่ `ProfileController`) —
นี่คือ convention มาตรฐานของ Rails ที่มักทำให้มือใหม่สับสน (เจอ `uninitialized constant
ProfileController` บ่อยเวลาลืมจุดนี้):

```ruby
# app/controllers/profiles_controller.rb  <- ชื่อพหูพจน์เสมอ แม้ resource จะเป็นเอกพจน์
class ProfilesController < ApplicationController
  def show
    @profile = current_user.profile
  end

  def edit
    @profile = current_user.profile
  end

  def update
    @profile = current_user.profile
    if @profile.update(profile_params)
      redirect_to profile_path, notice: "อัปเดตโปรไฟล์สำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  def profile_params
    params.require(:profile).permit(:display_name, :bio)
  end
end
```

### เมื่อไหร่ควรใช้ `resource` (เอกพจน์) แทน `resources`

ใช้เมื่อ resource นั้น "มีอยู่แค่ชิ้นเดียวในบริบทนั้น" เสมอ ตัวอย่างที่พบบ่อย:

- โปรไฟล์ของ user ที่ login อยู่ (`resource :profile`)
- การตั้งค่าบัญชี (`resource :settings`)
- session การ login/logout (`resource :session, only: %i[new create destroy]`
  ใช้แทนระบบ login เพราะไม่มีแนวคิด "id ของ session ไหน" — มีแค่ "session ปัจจุบัน")
- ตะกร้าสินค้าของผู้ใช้ปัจจุบัน (`resource :cart`)

---

## Step 217: Nested resources — `resources :posts do resources :comments end` และ `shallow: true`

comment (ความคิดเห็น) เป็นตัวอย่างคลาสสิกของความสัมพันธ์แบบ "เป็นของ" (belongs to)
comment หนึ่งอันต้อง**เป็นของ post หนึ่งอันเสมอ** ไม่มี comment ที่ลอยอยู่เดี่ยวๆ โดยไม่มี
post — ความสัมพันธ์แบบนี้ควรสะท้อนออกมาใน URL ด้วย เพื่อให้ URL สื่อความหมายชัดเจนว่า
comment นี้อยู่ภายใต้ post ไหน

### ประกาศ nested resources แบบตรงไปตรงมา

```ruby
# config/routes.rb
resources :posts do
  resources :comments
end
```

ตรวจสอบผลลัพธ์จริง (เฉพาะส่วนของ comments):

```
        Prefix Verb   URI Pattern                             Controller#Action
 post_comments GET    /posts/:post_id/comments(.:format)      comments#index
               POST   /posts/:post_id/comments(.:format)      comments#create
new_post_comment GET  /posts/:post_id/comments/new(.:format)  comments#new
edit_post_comment GET /posts/:post_id/comments/:id/edit(.:format) comments#edit
  post_comment GET    /posts/:post_id/comments/:id(.:format)  comments#show
               PATCH  /posts/:post_id/comments/:id(.:format)  comments#update
               PUT    /posts/:post_id/comments/:id(.:format)  comments#update
               DELETE /posts/:post_id/comments/:id(.:format)  comments#destroy
```

สิ่งที่เปลี่ยนไป:

- ทุก URL ของ comment มี `/posts/:post_id/` นำหน้าเสมอ — ใน controller เข้าถึงได้ผ่าน
  `params[:post_id]`
- route helper กลายเป็น `post_comments_path(post)` (ต้องส่ง post เข้าไปด้วยเสมอ เพราะ
  URL ต้องมี `post_id`)

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  before_action :set_post

  def create
    @comment = @post.comments.build(comment_params)
    if @comment.save
      redirect_to post_path(@post), notice: "เพิ่มความคิดเห็นสำเร็จ"
    else
      redirect_to post_path(@post), alert: "เพิ่มความคิดเห็นไม่สำเร็จ"
    end
  end

  private

  def set_post
    @post = Post.find(params[:post_id]) # comment ทุกตัวต้องมี post_id เสมอ
  end

  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

```erb
<%# ใน view ต้องส่งทั้ง post และ comment เข้า path helper %>
<%= link_to "แก้ไข", edit_post_comment_path(@post, comment) %>
<%= link_to "ลบ", post_comment_path(@post, comment), data: { turbo_method: :delete } %>
```

### ปัญหาของ nested resources ที่ลึกเกินไป (nesting ที่ลึกกว่า 1 ชั้น)

Rails ทำให้ nest ซ้อนกันได้กี่ชั้นก็ได้ในทางเทคนิค แต่ **ควรหลีกเลี่ยงการ nest เกิน 1 ชั้น**
เพราะ URL จะยาวและอ่านยากขึ้นเรื่อยๆ ตัวอย่างที่ไม่ควรทำ:

```ruby
# ไม่ควรทำ — nest ลึกเกินไป (2 ชั้น)
resources :posts do
  resources :comments do
    resources :replies
  end
end
```

จะได้ URL แบบ `/posts/:post_id/comments/:comment_id/replies/:id` ซึ่งยาวเกินความจำเป็น
และ controller ของ `replies` ต้อง `find` ทั้ง post และ comment ก่อนถึงจะเข้าถึง reply ได้
จริง (สร้าง coupling ที่ไม่จำเป็นระหว่าง 3 resource)

กฎที่ใช้ได้ในทางปฏิบัติ (มาจากบทความ "The Rails Way" ของ Jamis Buck ที่เป็นที่ยอมรับใน
วงการ Rails มานาน): **"Resources should never be nested more than 1 level deep"**

### ทางออก: `shallow: true`

เมื่อ nest 1 ชั้นแต่ต้องการ URL ที่สั้นลงสำหรับ action ที่ **ไม่จำเป็นต้องรู้ parent**
(คือ `show`, `edit`, `update`, `destroy` — เพราะรู้ `comment.id` ก็หา comment เจอได้เลย
ไม่ต้องพึ่ง `post_id`) ให้เติม `shallow: true`:

```ruby
# config/routes.rb
resources :posts do
  resources :comments, shallow: true
end
```

ตรวจสอบผลลัพธ์จริง — เทียบให้เห็นความแตกต่างชัดๆ:

```
     Prefix Verb   URI Pattern                             Controller#Action
post_comments GET   /posts/:post_id/comments(.:format)     comments#index
              POST  /posts/:post_id/comments(.:format)     comments#create
new_post_comment GET /posts/:post_id/comments/new(.:format) comments#new
edit_comment GET    /comments/:id/edit(.:format)            comments#edit
     comment GET    /comments/:id(.:format)                 comments#show
             PATCH  /comments/:id(.:format)                 comments#update
             PUT    /comments/:id(.:format)                 comments#update
             DELETE /comments/:id(.:format)                 comments#destroy
```

สิ่งที่ `shallow: true` ทำ:

| Action | ก่อน shallow | หลัง shallow |
|--------|-------------|--------------|
| `index` | `/posts/:post_id/comments` | `/posts/:post_id/comments` (เหมือนเดิม — ต้องรู้ post) |
| `new` | `/posts/:post_id/comments/new` | `/posts/:post_id/comments/new` (เหมือนเดิม) |
| `create` | `/posts/:post_id/comments` | `/posts/:post_id/comments` (เหมือนเดิม) |
| `show` | `/posts/:post_id/comments/:id` | `/comments/:id` (สั้นลง!) |
| `edit` | `/posts/:post_id/comments/:id/edit` | `/comments/:id/edit` (สั้นลง!) |
| `update` | `/posts/:post_id/comments/:id` | `/comments/:id` (สั้นลง!) |
| `destroy` | `/posts/:post_id/comments/:id` | `/comments/:id` (สั้นลง!) |

หลักการคือ: action ที่ **ต้องสร้างใหม่** (`new`, `create`) หรือ **ต้องแสดงเป็นรายการภายใต้
parent** (`index`) ยังต้องมี `post_id` เพราะยังไม่รู้ comment เป็นตัวไหน (หรือกำลังจะสร้าง)
แต่ action ที่ **มี comment `:id` อยู่แล้ว** (`show`, `edit`, `update`, `destroy`) ไม่จำเป็น
ต้องรู้ post เลย เพราะ `Comment.find(id)` ก็หาเจอได้โดยตรง — `shallow: true` จึงตัด
`post_id` ที่ไม่จำเป็นออกจาก URL เหล่านั้น

> **แนวปฏิบัติที่แนะนำ:** เมื่อ nest resources ให้ใส่ `shallow: true` เป็นค่าเริ่มต้นแทบทุก
> ครั้ง ยกเว้นมีเหตุผลเฉพาะที่ต้องการ `post_id` ติดอยู่ใน URL ของทุก action จริงๆ (เช่น
> ต้องการ URL ที่สื่อบริบทชัดเจนสำหรับ SEO)

ถ้ามีหลาย resource ที่ nest กันในไฟล์เดียว และอยากให้ทุกอันเป็น shallow โดยไม่ต้องเขียน
`shallow: true` ซ้ำทุกที่ ใช้ `shallow do...end` ครอบได้:

```ruby
shallow do
  resources :posts do
    resources :comments
    resources :likes
  end
end
```

---

## Step 218: Namespace vs Scope — `namespace :admin` เทียบกับ `scope module:`

เว็บแอปขนาดกลาง-ใหญ่มักมีส่วน "แอดมิน" แยกออกจากส่วนผู้ใช้ทั่วไปอย่างชัดเจน ทั้ง URL
(`/admin/posts` ต่างจาก `/posts`) และ controller (`Admin::PostsController` ต่างจาก
`PostsController`) Rails มีสองเครื่องมือที่ทำเรื่องนี้ได้ ซึ่งมือใหม่มักสับสนกัน:
`namespace` และ `scope module:`

### `namespace :admin` — เปลี่ยนทั้ง URL, module และ route helper prefix

```ruby
# config/routes.rb
namespace :admin do
  resources :posts
end
```

ตรวจสอบผลลัพธ์จริง:

```
      Prefix Verb   URI Pattern                     Controller#Action
 admin_posts GET    /admin/posts(.:format)          admin/posts#index
             POST   /admin/posts(.:format)          admin/posts#create
new_admin_post GET  /admin/posts/new(.:format)      admin/posts#new
edit_admin_post GET /admin/posts/:id/edit(.:format) admin/posts#edit
  admin_post GET    /admin/posts/:id(.:format)      admin/posts#show
             PATCH  /admin/posts/:id(.:format)      admin/posts#update
             PUT    /admin/posts/:id(.:format)      admin/posts#update
             DELETE /admin/posts/:id(.:format)      admin/posts#destroy
```

`namespace :admin` เปลี่ยนพร้อมกันทั้ง 3 สิ่ง:

1. **URL** มี `/admin` นำหน้า: `/admin/posts`
2. **Controller ต้องอยู่ใน module `Admin`**: ต้องสร้างไฟล์ที่
   `app/controllers/admin/posts_controller.rb` โดยมี class `Admin::PostsController`
3. **Route helper มี prefix `admin_`**: `admin_posts_path`, `admin_post_path(post)`

```ruby
# app/controllers/admin/posts_controller.rb
module Admin
  class PostsController < ApplicationController
    def index
      @posts = Post.all
    end
    # ...
  end
end
```

> **ข้อควรระวังเรื่อง scaffold/generator:** เวลาใช้ `rails g controller` สำหรับ
> controller ที่อยู่ใน namespace ให้เขียนเป็น `rails g controller Admin::Posts index show`
> (ใช้ `::`) Rails จะสร้างโฟลเดอร์ `app/controllers/admin/` และไฟล์ view ที่
> `app/views/admin/posts/` ให้อัตโนมัติ ตรงกับที่ route คาดหวังพอดี

### `scope module:` — เปลี่ยนแค่ module ที่ไปหา controller โดย URL และ helper เหมือนเดิม

บางครั้งต้องการแค่ **จัดกลุ่ม controller ในโค้ดให้เป็นระเบียบ** (แยกโฟลเดอร์ตาม domain
เช่น `storefront/`) แต่ **ไม่ต้องการให้ URL หรือ route helper เปลี่ยน** กรณีนี้ใช้
`scope module:` แทน:

```ruby
# config/routes.rb
scope module: "storefront" do
  resources :orders
end
```

ตรวจสอบผลลัพธ์จริง:

```
Prefix Verb   URI Pattern                Controller#Action
orders GET    /orders(.:format)          storefront/orders#index
       POST   /orders(.:format)          storefront/orders#create
new_order GET /orders/new(.:format)      storefront/orders#new
...
 order GET    /orders/:id(.:format)      storefront/orders#show
```

สังเกตความต่างจาก `namespace` อย่างชัดเจน:

- **URL ไม่มี `/storefront` นำหน้า** — ยังคงเป็น `/orders` เหมือนเดิม
- **Route helper ไม่มี prefix `storefront_`** — ยังคงเป็น `orders_path`, `order_path`
- **แต่ controller ต้องอยู่ที่ `app/controllers/storefront/orders_controller.rb`**
  (module `Storefront::OrdersController`) เหมือนกับ namespace

### ตารางเปรียบเทียบสรุป

| | `namespace :admin` | `scope module: "storefront"` |
|---|---|---|
| URL path | เปลี่ยน (`/admin/...`) | ไม่เปลี่ยน |
| Route helper prefix | เปลี่ยน (`admin_...`) | ไม่เปลี่ยน |
| Controller module | เปลี่ยน (`Admin::...`) | เปลี่ยน (`Storefront::...`) |
| ใช้เมื่อ | ต้องการแยกทั้ง URL และโค้ดออกจากกันชัดเจน (เช่น พื้นที่แอดมิน, พื้นที่ API) | ต้องการจัดระเบียบโค้ดเป็นกลุ่มเท่านั้น โดย URL ที่ผู้ใช้เห็นยังเหมือนเดิม |

พูดง่ายๆ คือ **`namespace :admin` = `scope module: "admin", path: "admin", as: "admin"`
รวมกันในคำสั่งเดียว** — ถ้าต้องการ custom เพิ่มเติม เช่น เปลี่ยน module แต่ตั้ง path เป็น
คำอื่น ก็ประกาศ `scope` เองแบบละเอียดได้ เช่น:

```ruby
# module เปลี่ยนเป็น admin, path เปลี่ยนเป็น "backend", helper prefix เปลี่ยนเป็น "admin_"
scope module: "admin", path: "backend", as: "admin" do
  resources :posts
end
# ได้ URL /backend/posts แต่ helper ยังชื่อ admin_posts_path
# และ controller ยังเป็น Admin::PostsController
```

---

## Step 219: Constraints — จำกัด route ด้วย format, subdomain, และ custom constraint class

บางครั้งต้องการให้ route หนึ่งทำงาน **เฉพาะเมื่อเงื่อนไขบางอย่างเป็นจริง** เท่านั้น เช่น
รับเฉพาะ format ที่กำหนด, รับเฉพาะ subdomain ที่กำหนด หรือเงื่อนไข custom ที่ซับซ้อนกว่านั้น
Rails มี `constraints` ให้ใช้สำหรับกรณีเหล่านี้

### Constraint แบบ format (regex)

```ruby
# config/routes.rb
get "posts.:format" => "posts#index", constraints: { format: /json|xml/ }
```

route นี้จะ match เฉพาะเมื่อ format เป็น `json` หรือ `xml` เท่านั้น (`GET /posts.json` หรือ
`GET /posts.xml` — ไม่ match `GET /posts.csv`) ตรวจสอบด้วย `bin/rails routes`:

```
Prefix Verb URI Pattern      Controller#Action
      GET   /posts.:format   posts#index {:format=>/json|xml/}
```

สังเกตว่าคอลัมน์ Controller#Action มีเงื่อนไข constraint ต่อท้ายให้เห็นด้วย — มีประโยชน์
มากตอน debug ว่าทำไม route ถึง (ไม่) match

### Constraint แบบ subdomain

ใช้บ่อยเมื่อแอปมีหลาย subdomain (เช่น `api.example.com` แยกจาก `www.example.com`):

```ruby
constraints subdomain: "api" do
  get "status" => "posts#index"
end
```

route `/status` นี้จะทำงานเฉพาะเมื่อ request มาจาก `api.example.com/status` เท่านั้น ถ้ามา
จาก `www.example.com/status` หรือไม่มี subdomain เลย จะได้ 404 (ตรวจสอบแล้วว่า Rails มองว่า
route ไม่ match เมื่อ subdomain ไม่ตรง)

```
Prefix Verb URI Pattern    Controller#Action
status GET  /status(.:format) posts#index {:subdomain=>"api"}
```

### Constraint แบบ custom class — เมื่อเงื่อนไขซับซ้อนกว่า regex/subdomain ธรรมดา

สำหรับเงื่อนไขที่ซับซ้อนกว่านั้น (เช่น ต้องเช็คว่า user login อยู่หรือไม่ ต้องเช็ค
feature flag ฯลฯ) เขียนเป็น class ที่มี method `matches?(request)` คืนค่า `true`/`false`:

```ruby
# app/constraints/admin_subdomain_constraint.rb
class AdminSubdomainConstraint
  def matches?(request)
    request.subdomain == "admin"
  end
end
```

```ruby
# config/routes.rb
constraints AdminSubdomainConstraint.new do
  get "dashboard" => "admin/dashboard#index"
end
```

ตรวจสอบด้วย `bin/rails routes` (ผลลัพธ์จริง — route แบบ custom class จะแสดงเหมือน route
ปกติ ไม่มีข้อความ constraint ต่อท้ายให้เห็นเหมือนกรณี hash constraint):

```
   Prefix Verb URI Pattern          Controller#Action
dashboard GET  /dashboard(.:format) admin/dashboard#index
```

ทดสอบ behavior จริงด้วยการยิง request (`matches?` คืน `false` เมื่อไม่ใช่ admin subdomain):

```bash
curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/dashboard
# => 404  (เพราะไม่ได้มาจาก admin subdomain เลย constraint ไม่ match)
```

`request` object ที่ส่งเข้า `matches?` คือ `ActionDispatch::Request` เต็มรูปแบบ เข้าถึงได้
ทั้ง `request.params`, `request.session`, `request.cookies`, `request.subdomain` ฯลฯ
ทำให้เขียนเงื่อนไขซับซ้อนแค่ไหนก็ได้ เช่น:

```ruby
# app/constraints/beta_tester_constraint.rb
class BetaTesterConstraint
  def matches?(request)
    request.cookies["beta_tester"] == "true"
  end
end
```

> **ข้อควรระวัง:** อย่าใช้ constraint class เพื่อทำ **authorization** (การเช็คสิทธิ์เข้าถึง
> เช่น "เฉพาะ admin เท่านั้นเข้าได้") เพราะถ้า constraint ไม่ match จะได้ 404 เฉยๆ
> (เหมือนไม่มีหน้านี้อยู่เลย) ซึ่งอาจไม่ใช่ UX ที่ต้องการ (บางทีอยากให้ user เห็นข้อความ
> "ไม่มีสิทธิ์เข้าถึง" ไม่ใช่ "ไม่พบหน้านี้") งาน authorization ที่แท้จริงควรทำใน
> `before_action` ของ controller (Part 023) หรือใช้ gem อย่าง Pundit (Part 043) —
> constraint เหมาะกับการ "แยกกลุ่ม route ตามคุณสมบัติของ request" มากกว่า

---

## Step 220: Root route, redirect route, และการจัดการ 404 แบบ catch-all

### Root route — กำหนดว่า `/` ไปที่ไหน

ทุกเว็บไซต์ต้องมีหน้าแรก (`GET /`) Rails ใช้คำสั่ง `root` เพื่อกำหนด:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts
end
```

`root "posts#index"` คือ shorthand ของ `root to: "posts#index"` ตรวจสอบผลลัพธ์จริง:

```
Prefix Verb URI Pattern Controller#Action
 root GET   /           posts#index
```

route helper ที่ได้คือ `root_path` / `root_url` (ไม่ใช่ `posts_path` แม้จะชี้ไปที่ action
เดียวกัน) ใช้ในลิงก์โลโก้/หน้าแรกได้เลย:

```erb
<%= link_to "หน้าแรก", root_path %>
```

> **แนวปฏิบัติที่ดี:** ใส่ `root` **ไว้บนสุดของไฟล์** `config/routes.rb` เสมอ (หลัง route
> ของระบบอย่าง health check) แม้จะไม่มีผลต่อการทำงาน (Rails หา root ได้ไม่ว่าจะประกาศไว้
> ตรงไหนในไฟล์) แต่ทำให้คนอ่านไฟล์เห็นทันทีว่าแอปนี้หน้าแรกคือหน้าไหน โดยไม่ต้องไล่หาทั้ง
> ไฟล์

### Redirect route — เปลี่ยน URL เก่าให้ชี้ไป URL ใหม่ (ไม่ต้องพึ่ง controller)

เมื่อเปลี่ยนโครงสร้าง URL (เช่น เปลี่ยนจาก `/old-posts` เป็น `/posts`) แต่ต้องการให้ลิงก์เก่า
ที่คนแชร์ไว้ (หรือ search engine index ไว้) ยังใช้งานได้ ใช้ `redirect` แทนการเขียน
controller action ใหม่:

```ruby
# config/routes.rb
get "old-posts" => redirect("/posts")
get "old-posts/:id" => redirect("/posts/%{id}")
```

ทดสอบจริงด้วย `curl` (ยืนยันว่าได้ HTTP 301 Moved Permanently พร้อม header `Location`
ที่ถูกต้อง):

```bash
curl -I http://localhost:3000/old-posts
```

```
HTTP/1.1 301 Moved Permanently
Location: http://localhost:3000/posts
Content-Type: text/html; charset=UTF-8
```

`redirect` รับได้ทั้ง string ธรรมดา และ string ที่มี placeholder แบบ `%{param_name}`
สำหรับดึงค่าจาก URL เดิมมาใส่ใน URL ใหม่ (ดังตัวอย่าง `old-posts/:id`) หรือจะใช้ block
รับ `params`/`request` เพื่อคำนวณ URL ปลายทางแบบซับซ้อนกว่านั้นก็ได้:

```ruby
get "old-posts/:id" => redirect { |params, req| "/posts/#{params[:id]}?ref=old" }
```

> **หมายเหตุ:** redirect ที่ได้เป็น **301 (Permanent Redirect)** โดย default ซึ่งเบราว์เซอร์
> และ search engine จะจำไว้และไม่เรียก URL เก่าซ้ำอีกในอนาคต ถ้าต้องการ redirect แบบ
> ชั่วคราว (302) ต้องระบุ `status: 302` เพิ่ม เช่น
> `get "maintenance-notice" => redirect("/", status: 302)`

### จัดการ 404 แบบ catch-all — เมื่อไม่มี route ไหน match เลย

ถ้า URL ที่ผู้ใช้พิมพ์ไม่ตรงกับ route ใดๆ ที่ประกาศไว้เลย Rails จะตอบกลับหน้า 404 มาตรฐาน
(`public/404.html`) ให้อัตโนมัติอยู่แล้วโดยไม่ต้องทำอะไรเพิ่ม แต่ถ้าต้องการควบคุมหน้า 404
เอง (เช่น ให้มี layout เดียวกับเว็บ ไม่ใช่หน้า static html เปล่าๆ) ให้ประกาศ route
**catch-all ไว้ล่างสุดของไฟล์เสมอ**:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "posts#index"

  resources :posts

  # ต้องอยู่ล่างสุดของไฟล์เสมอ — เพราะ Rails จับคู่ route จากบนลงล่าง
  # ถ้าวางไว้บนสุด จะ "ดักจับ" ทุก request ก่อน route อื่นๆ ที่ประกาศไว้ด้านล่าง
  match "*unmatched", to: "application#route_not_found", via: :all
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def route_not_found
    render plain: "404 Not Found", status: :not_found
  end
end
```

ทดสอบจริง:

```bash
curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/nonexistent-thing
# => 404
```

**อธิบาย syntax:**

- `"*unmatched"` — เครื่องหมาย `*` (glob route) หมายถึง "จับคู่กับอะไรก็ได้ที่เหลือ"
  รวมทุก segment ของ path เก็บไว้ใน `params[:unmatched]`
- `via: :all` — จับคู่ทุก HTTP verb (`GET`, `POST`, `PATCH`, ฯลฯ) ไม่ใช่แค่ `GET`
- **ลำดับสำคัญมาก** — Rails จับคู่ route จากบนลงล่างไฟล์ตามลำดับที่ประกาศ (route แรกที่
  match จะถูกใช้ทันที ไม่ตรวจ route ถัดไปอีก) route แบบ catch-all จึงต้องอยู่**ล่างสุด
  ของไฟล์เสมอ** ไม่เช่นนั้นจะ "กิน" ทุก request ก่อน route จริงๆ ด้านล่าง ทำให้ route อื่น
  ไม่มีวันถูกเรียกถึงเลย

> **แนวคิดสำคัญที่ควรจำ:** ปัญหา routing ที่พบบ่อยที่สุดของมือใหม่คือ "ทำไม route นี้ไม่
> ทำงาน ทั้งที่ประกาศถูกต้องแล้ว" ซึ่งส่วนใหญ่เกิดจาก **ลำดับ route ผิด** — มี route ที่
> กว้างกว่า (เช่น catch-all หรือ dynamic segment ที่ครอบคลุมกว้าง) ถูกประกาศไว้**ก่อน**
> route ที่เจาะจงกว่า ทำให้ route ที่เจาะจงกว่าไม่มีวันถูกเรียกถึง วิธีตรวจสอบเสมอคือรัน
> `bin/rails routes -g <keyword>` และดูว่า route ที่คาดหวังปรากฏอยู่จริงหรือไม่ ในลำดับที่
> ถูกต้องหรือไม่

---

## แบบฝึกหัด: ออกแบบ routes.rb สำหรับระบบบล็อกเล็กๆ

### โจทย์

ออกแบบและเขียน `config/routes.rb` สำหรับระบบบล็อกที่มีข้อกำหนดดังนี้:

1. มี resource `posts` ครบ 7 action มาตรฐาน
2. แต่ละ post มี `comments` ที่ nested อยู่ภายใต้ post แต่ให้ใช้ **shallow nesting**
   (จำกัดเฉพาะ action `create` และ `destroy` เท่านั้น — ระบบนี้ยังไม่ต้องการหน้าแก้ไข
   comment แยกต่างหาก)
3. มีส่วนแอดมิน (`namespace :admin`) ที่มี `posts` ครบ 7 action และ `comments` ที่ดูได้
   (`index`) และลบได้ (`destroy`) เท่านั้น
4. หน้าแรกของเว็บ (`root`) ให้ไปที่หน้ารายการ post (`posts#index`)

### เฉลย

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  # ข้อ 4: root route ไปหน้ารายการ post — วางไว้บนสุดตามแนวปฏิบัติที่ดี
  root "posts#index"

  # ข้อ 1 + ข้อ 2: posts พร้อม comments แบบ nested + shallow
  resources :posts do
    resources :comments, only: %i[create destroy], shallow: true
  end

  # ข้อ 3: ส่วนแอดมิน แยก namespace ชัดเจนทั้ง URL, controller module, route helper
  namespace :admin do
    resources :posts
    resources :comments, only: %i[index destroy]
  end
end
```

ต้องมี controller รองรับครบตามที่ route ต้องการ:

```bash
bin/rails g controller Posts index show new create edit update destroy
bin/rails g controller Comments create destroy
bin/rails g controller Admin::Posts index show new create edit update destroy
bin/rails g controller Admin::Comments index destroy
```

ตรวจสอบด้วย `bin/rails routes` — ผลลัพธ์จริงที่ทดสอบแล้วบน Rails 8.1.4:

```
            Prefix Verb   URI Pattern                        Controller#Action
rails_health_check GET    /up(.:format)                      rails/health#show
              root GET    /                                  posts#index
     post_comments POST   /posts/:post_id/comments(.:format) comments#create
           comment DELETE /comments/:id(.:format)            comments#destroy
             posts GET    /posts(.:format)                   posts#index
                   POST   /posts(.:format)                   posts#create
          new_post GET    /posts/new(.:format)               posts#new
         edit_post GET    /posts/:id/edit(.:format)          posts#edit
              post GET    /posts/:id(.:format)               posts#show
                   PATCH  /posts/:id(.:format)               posts#update
                   PUT    /posts/:id(.:format)               posts#update
                   DELETE /posts/:id(.:format)               posts#destroy
       admin_posts GET    /admin/posts(.:format)             admin/posts#index
                   POST   /admin/posts(.:format)             admin/posts#create
    new_admin_post GET    /admin/posts/new(.:format)         admin/posts#new
   edit_admin_post GET    /admin/posts/:id/edit(.:format)    admin/posts#edit
        admin_post GET    /admin/posts/:id(.:format)         admin/posts#show
                   PATCH  /admin/posts/:id(.:format)         admin/posts#update
                   PUT    /admin/posts/:id(.:format)         admin/posts#update
                   DELETE /admin/posts/:id(.:format)         admin/posts#destroy
    admin_comments GET    /admin/comments(.:format)          admin/comments#index
     admin_comment DELETE /admin/comments/:id(.:format)      admin/comments#destroy
```

**ตรวจทานผลลัพธ์ตามข้อกำหนดทีละข้อ:**

1. `posts` มี route ครบ 7 action ✓ (`index`, `create`, `new`, `edit`, `show`, `update`
   x2 method, `destroy`)
2. `comments` มีแค่ `post_comments` (create, มี `post_id` เพราะต้องรู้ว่าสร้างให้ post ไหน)
   กับ `comment` (destroy, ไม่มี `post_id` เพราะ `shallow: true` ตัดออกให้แล้ว) ✓ ตรงตาม
   `only: %i[create destroy]` ที่ระบุไว้พอดี
3. `admin_posts` มีครบ 7 action พร้อม prefix/module `admin` ✓ ส่วน `admin_comments` มีแค่
   `index` กับ `destroy` ✓
4. `root GET /` ชี้ไปที่ `posts#index` ✓

**จุดที่ควรสังเกตเพิ่มเติมจากผลลัพธ์:**

- `comment_path(comment)` (ไม่ใช่ `post_comment_path`) ใช้สำหรับปุ่มลบ comment เพราะ
  `shallow: true` ทำให้ helper ของ `destroy` ไม่ต้องมี `post_id` — ในหน้า
  `posts/show.html.erb` ที่แสดงลิสต์ comment จึงเขียนปุ่มลบได้ตรงไปตรงมาแบบนี้:

  ```erb
  <% @post.comments.each do |comment| %>
    <p><%= comment.body %></p>
    <%= button_to "ลบ", comment_path(comment), method: :delete %>
  <% end %>
  ```

- ฟอร์ม create comment ต้องใช้ `post_comments_path(@post)` เพราะต้องระบุว่า comment ใหม่
  นี้เป็นของ post ไหน:

  ```erb
  <%= form_with model: [@post, Comment.new] do |f| %>
    <%= f.text_area :body %>
    <%= f.submit "แสดงความคิดเห็น" %>
  <% end %>
  ```

  (`form_with model: [@post, Comment.new]` คือการใช้ nested route ผ่าน array — Rails จะ
  ประกอบ URL เป็น `post_comments_path(@post)` ให้อัตโนมัติ รายละเอียดเรื่อง `form_with`
  เต็มรูปแบบจะสอนใน Part 031)

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม custom action `feature`/`unfeature` (ปักหมุด/เลิกปักหมุดโพสต์เด่น) เป็น
   `member` action ของ `posts` แล้วรัน `bin/rails routes -g feature` เพื่อตรวจสอบว่า
   URL และ route helper ที่ได้คือ `feature_post_path(post)`/`unfeature_post_path(post)`
   ตามที่คาดหวังหรือไม่

2. เพิ่ม resource ใหม่ชื่อ `tags` (แท็กของบทความ) แบบ `resources :tags, only: %i[index
   show]` เท่านั้น (ผู้ใช้ดูรายการแท็กและดูบทความในแต่ละแท็กได้ แต่สร้าง/แก้ไข/ลบแท็กได้
   จากฝั่งแอดมินเท่านั้น) แล้วเพิ่ม `resources :tags` (ครบ 7 action) เข้าไปใน
   `namespace :admin` เพื่อให้แอดมินจัดการแท็กได้ครบ — ตรวจสอบว่า `admin_tags_path` และ
   `tags_path` ไม่ชนกัน (ควรเป็นคนละ URL คนละ helper กันอย่างชัดเจน)

3. เขียน constraint class ชื่อ `JsonRequestConstraint` ที่ตรวจสอบว่า request header
   `Accept` มีคำว่า `application/json` หรือไม่ (`request.headers["Accept"]`) แล้วใช้
   constraint นี้ครอบ route กลุ่มหนึ่งที่ควรตอบกลับเฉพาะ JSON เท่านั้น ทดสอบด้วย
   `curl -H "Accept: application/json"` เทียบกับการยิงแบบไม่ใส่ header ว่าผลลัพธ์ต่างกัน
   ตามที่ออกแบบไว้หรือไม่

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- Router (`config/routes.rb`) คือจุดแรกที่ request ทุกตัวต้องผ่าน ทำหน้าที่จับคู่
  **(HTTP verb, URL pattern)** กับ **(controller, action)**
- RESTful convention กำหนด 7 action มาตรฐาน (`index`, `show`, `new`, `create`, `edit`,
  `update`, `destroy`) ที่ทุกทีม Rails ทั่วโลกยึดร่วมกัน ทำให้โค้ดเข้าใจตรงกันได้ทันที
- `resources :posts` สร้าง route ครบ 7 action ให้ในบรรทัดเดียว จำกัดจำนวนได้ด้วย
  `only:`/`except:`
- Route helper (`_path`/`_url`) ควรใช้แทนการ hardcode URL เสมอ เพื่อความทนทานต่อการ
  เปลี่ยนแปลงโครงสร้าง URL ในอนาคต
- `member`/`collection` เพิ่ม custom action ให้ resource ได้ตามต้องการ นอกเหนือจาก 7
  action มาตรฐาน
- `resource :profile` (เอกพจน์) ใช้กับ resource ที่มีอยู่แค่หนึ่งเดียวในบริบทนั้น
  (ไม่มี `:id` ใน URL) แต่ controller ยังคงตั้งชื่อพหูพจน์เสมอ
- Nested resources สื่อความสัมพันธ์ parent-child ผ่าน URL ได้ แต่ควรจำกัดความลึกไว้แค่
  1 ชั้น และใช้ `shallow: true` เพื่อตัด URL ที่ไม่จำเป็นออก
- `namespace :admin` เปลี่ยนทั้ง URL, controller module, และ route helper prefix พร้อมกัน
  ส่วน `scope module:` เปลี่ยนแค่ controller module โดย URL/helper เหมือนเดิม
- `constraints` จำกัด route ด้วยเงื่อนไข format, subdomain หรือเงื่อนไข custom ผ่าน class
  ที่มี method `matches?(request)` — แต่ไม่ควรใช้แทนการทำ authorization จริง
- `root` กำหนดหน้าแรกของเว็บ, `redirect` เปลี่ยนเส้นทาง URL เก่าไปใหม่โดยไม่ต้องเขียน
  controller, และ catch-all route (`match "*unmatched"`) ต้องอยู่ล่างสุดของไฟล์เสมอ
  เพราะ Rails จับคู่ route จากบนลงล่างตามลำดับที่ประกาศไว้

**ต่อไป (Part 023):** ตอนนี้เรารู้แล้วว่า request จะถูกส่งไปที่ controller#action ไหน
Part ถัดไปจะเจาะลึกฝั่ง **Controller** เอง — action ทำงานอย่างไรจริงๆ, การอ่านค่าจาก
`params` (ทั้งจาก URL segment, query string, และ form body), การเก็บข้อมูลข้าม request
ด้วย `session`, การส่งข้อความแจ้งเตือนชั่วคราวด้วย `flash`, และการใช้ `before_action`
เพื่อรันโค้ดร่วมกันก่อนทุก action (เช่น ตรวจสอบ login, โหลด record ที่ใช้ร่วมกัน)
