# Part 056: Rails API-only Mode, JSON Response และ Serializer (Jbuilder / Blueprinter) — เปิด Phase 8

> **Step ครอบคลุมใน Part นี้:** Step 551–560
> **ระดับ:** กลาง-สูง (ควรผ่าน **Part 034** เรื่อง Query interface/N+1, **Part 023** เรื่อง
> `rescue_from`, และ **Part 045** เรื่อง JWT/API auth มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6,
> Rails 8.1.4 โหมด `--api`, gem `jbuilder` 2.15.x ซึ่งเป็นเวอร์ชัน default ที่มากับ Rails,
> gem `blueprinter` 1.3.x)

ยินดีต้อนรับสู่ **Phase 8: APIs & GraphQL** — เฟสนี้จะเปลี่ยนมุมมองของเราจาก "เขียนเว็บแอปที่
render HTML ให้ browser" (Phase 3–7 ทั้งหมด) ไปสู่ "เขียน backend ที่พูดภาษา JSON ล้วนๆ ให้ client
ประเภทไหนก็ได้เชื่อมต่อ" — mobile app, SPA, บริการอื่น, หรือแม้แต่ Rails app ตัวอื่นเรียกหากันเอง

จำ **Part 045** ได้ไหม — ตอนนั้นเราสร้างแอป `--api` ขึ้นมาแบบเร่งด่วนเพื่อสาธิต JWT authentication
โดยข้าม (glossed over) รายละเอียดของโหมด `--api` เองไปพอสมควร แค่พอให้ `render json:` ทำงานได้
Part นี้จะย้อนกลับไปเจาะทุกรายละเอียดที่ Part 045 ข้ามไป: **`--api` mode ตัดอะไรออกไปจริงๆ และ
เพราะอะไร**, **Rails serialize ActiveRecord object เป็น JSON อย่างไรเบื้องหลัง**, และที่สำคัญ
ที่สุดคือ **จะจัดการ JSON output ของ API ขนาดกลาง-ใหญ่ให้เป็นระเบียบ ทดสอบได้ และไม่ผูกติดกับ
model อย่างไร** — คำถามนี้เป็นคำถามที่ทีม Rails จริงเถียงกันมากที่สุดคำถามหนึ่ง เพราะระบบนิเวศ
(ecosystem) ของเครื่องมือ serialize JSON ใน Ruby เปลี่ยนไปเยอะมากในช่วงหลายปีที่ผ่านมา — Part นี้
จะพาไปดูภาพที่เป็นปัจจุบันจริง ไม่ใช่คำแนะนำที่ล้าสมัยจาก tutorial เก่า

## สารบัญของ Part นี้

- Step 551: `rails new --api` คืออะไรจริงๆ — สิ่งที่ Rails ตัดออกให้ และเหตุผลเบื้องหลัง
- Step 552: `render json:` กับกลไก `as_json`/`to_json` เบื้องหลัง — Rails serialize
  ActiveRecord object เป็น JSON ได้อย่างไร
- Step 553: ควบคุม JSON output เบื้องต้นด้วย `only:`/`except:` และทำไมการ override `as_json`
  บน model ถึงเป็น anti-pattern
- Step 554: Jbuilder — เครื่องมือสร้าง JSON แบบ view ที่มากับ Rails
- Step 555: เปรียบเทียบแนวทาง Serialization ที่ใช้จริงในปัจจุบัน — Jbuilder vs
  Blueprinter/Alba vs `as_json` ล้วน vs ActiveModel::Serializer (สถานะปัจจุบันที่ต้องรู้)
- Step 556: เลือกแนวทางของหลักสูตรนี้ และสร้าง Blueprint แรกด้วย Blueprinter
- Step 557: Serializer สำหรับ `Post` ที่มี `Comment` ซ้อนกัน (nested) พร้อมหลาย view
- Step 558: หลีกเลี่ยง N+1 ตอน serialize ข้อมูลที่มี association (ทบทวน `includes` จาก Part 034)
- Step 559: HTTP Status Code ที่ถูกต้องสำหรับ API response — ตารางอ้างอิงที่ต้องจำ
- Step 560: รูปแบบ Error Response ที่สม่ำเสมอทั้ง API ด้วย `rescue_from` + แบบฝึกหัดปิดท้าย

---

## Step 551: `rails new --api` คืออะไรจริงๆ — สิ่งที่ Rails ตัดออกให้ และเหตุผลเบื้องหลัง

### ทบทวนสั้นๆ จาก Part 045

Part 045 สั่ง `rails new jwt_demo --api -d sqlite3` แล้วใช้งานได้เลยโดยไม่อธิบายว่าธงนี้ทำอะไรบ้าง
Step นี้จะสร้างแอปใหม่อีกครั้งเพื่อ**ไล่ดูไฟล์ที่ถูกสร้าง/ตัดออกทีละจุด** เทียบกับแอปแบบเต็ม
(`rails new` ธรรมดา จาก Part 021)

```bash
rails new posts_api --api -d sqlite3
cd posts_api
```

ตอนสร้างเสร็จ ลองเทียบสิ่งที่ **ไม่มี** ในโปรเจกต์นี้กับแอปแบบเต็ม:

```bash
ls app/
# controllers  jobs  mailers  models  views
```

สังเกตว่า **ไม่มี** `app/helpers/` และ **ไม่มี** `app/assets/` เลย (แอปแบบเต็มจาก Part 021 มีทั้งคู่)
ส่วน `app/views/` ยังอยู่ แต่มีแค่ template ของ Action Mailer (`layouts/mailer.text.erb`,
`layouts/mailer.html.erb`) ไม่มี `layouts/application.html.erb` — ทดสอบจริงแล้วได้ผลตรงตามนี้

### `ApplicationController < ActionController::API` แทนที่ `ActionController::Base`

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
end
```

นี่คือหัวใจของโหมด `--api` — `ActionController::API` เป็น class คนละตัวกับ
`ActionController::Base` ที่ใช้มาตลอด Phase 3–7 มันสืบทอด (`include`) เฉพาะ module ของ
`ActionController` ที่ **จำเป็นสำหรับ API เท่านั้น** เช่น `Renderers` (สำหรับ `render json:`),
`StrongParameters`, `Instrumentation` — และ**ตัดโมดูลที่เกี่ยวกับ browser/HTML ทั้งหมดออกไป**
เช่น `Helpers` (view helper), `Flash`, `Cookies` บางส่วน (session ยังพอเรียกได้ถ้าตั้งค่าเอง แต่
ไม่ enable โดย default), และ `ActionView::Layouts` แบบเต็มรูปแบบสำหรับ HTML

### Middleware Stack ที่บางลงจริง

ลองเทียบ middleware stack จริงระหว่างแอปแบบเต็มกับแอป `--api`:

```bash
bin/rails middleware
```

ผลลัพธ์ที่ทดสอบรันจริงบนแอป `--api`:

```
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use ActionDispatch::ServerTiming
use ActiveSupport::Cache::Strategy::LocalCache::Middleware
use Rack::Runtime
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use ActionDispatch::DebugExceptions
use ActionDispatch::ActionableExceptions
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
run PostsApi::Application.routes
```

สิ่งที่**ไม่มี**เลยในรายการนี้ (แต่มีในแอปแบบเต็ม): `ActionDispatch::Session::CookieStore`,
`ActionDispatch::Cookies`, `ActionDispatch::Flash`, `Rack::MethodOverride` — พูดง่ายๆ คือ**ไม่มี
กลไก session/flash/CSRF protection ติดมาให้เลย**

**ทำไมถึงตัดออก?** ทบทวนเหตุผลจาก **Part 045 Step 441** — API ที่เปิดให้ mobile app, SPA
คนละโดเมน, หรือ machine-to-machine เรียกใช้งาน **ควรเป็น stateless** (ไม่มี session ฝั่ง server)
เพราะ client เหล่านี้ไม่มี "cookie jar" ให้พึ่งพาอยู่แล้ว การใช้ **JWT/token-based auth** (ที่
เรียนเต็มรูปแบบใน Part 045) แทน session-based auth จึงทำให้ CSRF protection **ไม่จำเป็นอีกต่อไป**
(CSRF โจมตีผ่านกลไกที่ browser แนบ cookie ให้อัตโนมัติ — ถ้าไม่มี cookie ไม่มีช่องโหว่นี้เลย) —
Rails จึงออกแบบให้ `--api` mode **ไม่แบก middleware ที่ไม่มีวันได้ใช้เหล่านี้ไปโดยเปล่าประโยชน์**
ทั้งเรื่องประสิทธิภาพ (แต่ละ request ผ่าน middleware น้อยชั้นกว่า เร็วกว่าเล็กน้อย) และเรื่อง
ความชัดเจนของโค้ด (ไม่มี API ตัวไหนพยายามอ่าน `session[:xxx]` แล้วงงว่าทำไมไม่มีอะไรอยู่ในนั้นเลย)

### `config.api_only = true` — สวิตช์ตัวเดียวที่ควบคุมทั้งหมด

```ruby
# config/application.rb
module PostsApi
  class Application < Rails::Application
    config.load_defaults 8.1
    config.autoload_lib(ignore: %w[assets tasks])

    # Only loads a smaller set of middleware suitable for API only apps.
    # Middleware like session, flash, cookies can be added back manually.
    # Skip views, helpers and assets when generating a new resource.
    config.api_only = true
  end
end
```

คอมเมนต์ที่ Rails generate มาให้บอกตรงๆ ว่า **`config.api_only = true` เป็นแค่ค่าตั้งต้นที่
เปลี่ยนกลับได้** — ถ้าโปรเจกต์เริ่มจาก `--api` แล้วภายหลังต้องการ session/cookie คืนมาบางส่วน
(เช่น ต้องการ browsable HTML debug page สำหรับ admin) สามารถ `include` module ที่ต้องการกลับเข้า
ไปใน `ApplicationController` เองได้ (`include ActionController::Cookies`,
`include ActionController::RequestForgeryProtection` เป็นต้น) — **ไม่ใช่กำแพงที่ทะลุไม่ได้**
เป็นเพียง sensible default เท่านั้น

### `Gemfile` ที่บางลง และจุดที่ต้อง "เอา `#` ออกเอง"

```bash
grep -v '^#' Gemfile | grep -v '^\s*$'
```

```ruby
source "https://rubygems.org"
gem "rails", "~> 8.1.4"
gem "sqlite3", ">= 2.1"
gem "puma", ">= 5.0"
gem "tzinfo-data", platforms: %i[ windows jruby ]
gem "solid_cache"
gem "solid_queue"
gem "solid_cable"
gem "bootsnap", require: false
gem "kamal", require: false
gem "thruster", require: false
gem "image_processing", "~> 1.2"
group :development, :test do
  gem "debug", platforms: %i[ mri windows ], require: "debug/prelude"
  gem "bundler-audit", require: false
  gem "brakeman", require: false
  gem "rubocop-rails-omakase", require: false
end
```

สังเกตว่า**ไม่มี** `gem "jbuilder"` โผล่มาให้ใช้งานทันที — เปิดไฟล์ `Gemfile` ดูตรงๆ จะเจอบรรทัด
ที่ comment ไว้:

```ruby
# Build JSON APIs with ease [https://github.com/rails/jbuilder]
# gem "jbuilder"
```

**นี่คือจุดที่หลาย tutorial เข้าใจผิด** — บาง blog เก่าบอกว่า "Jbuilder มากับ Rails API mode
โดย default" แต่ทดสอบจริงกับ Rails 8.1.4 แล้วพบว่า **`gem "jbuilder"` ถูก comment ไว้เหมือนกับ
โหมดเต็มทุกประการ** ต้องเอา `#` ออกเองเสมอถ้าจะใช้ (จะกลับมาใช้จริงใน Step 554) เหตุผลที่ Rails
ยัง comment ไว้ให้แม้ในโหมด `--api` คือ **Rails ไม่ต้องการ "ตัดสินใจแทน" ว่าโปรเจกต์นี้จะเลือกใช้
Jbuilder, serializer gem อื่น, หรือแค่ `render json:` ธรรมดาก็พอ** — Step 555 จะพาไปดูภาพรวมของ
ทางเลือกทั้งหมดก่อนตัดสินใจ

### ตารางสรุป: `--api` mode ตัดอะไรออก และทำไม

| องค์ประกอบ | โหมดเต็ม (`rails new`) | โหมด `--api` | เหตุผลที่ตัดออก |
|---|---|---|
| Controller base class | `ActionController::Base` | `ActionController::API` | ไม่ต้องมี HTML rendering, helper, layout เต็มรูปแบบ |
| `app/views/` | มี layout, partial, template ครบ | เหลือแค่ mailer template | Endpoint ส่วนใหญ่ตอบ JSON ไม่ต้องมี view HTML |
| `app/helpers/`, `app/assets/` | มี | **ไม่มี** | View helper/asset pipeline ไม่เกี่ยวกับการตอบ JSON |
| Session/Cookie middleware | มีเต็ม (`CookieStore`, `Flash`) | **ไม่มี** | API ควรเป็น stateless (ดู Part 045 Step 441) |
| CSRF Protection | เปิดอัตโนมัติ | **ปิด** (ไม่มี middleware เลย) | ไม่มี cookie ให้ browser แนบอัตโนมัติ = ไม่มีช่องโหว่ CSRF |
| `gem "jbuilder"` ใน Gemfile | uncomment ให้พร้อมใช้ | **comment ไว้เหมือนกัน** ต้องเปิดเอง | ให้ผู้พัฒนาเลือก serialization strategy เอง |
| Generator (`scaffold`) | สร้าง view HTML ให้ด้วย | สร้างแค่ controller/model, ข้าม view/helper | ไม่มี HTML ให้ generate |

> **ข้อควรจำ:** `--api` ไม่ใช่ Rails "อีกเวอร์ชันหนึ่ง" — มันคือ Rails ตัวเดียวกันเป๊ะ เพียงแค่
> ปิดสวิตช์บางอันที่ตั้งไว้เผื่อ web app แบบดั้งเดิม โมเดล, migration, routing, validation,
> association, ActiveJob — ทุกอย่างที่เรียนมาตลอด Phase 1–7 **ทำงานเหมือนเดิมทุกประการ**
> เปลี่ยนแค่ชั้น controller/view ที่คุยกับ client เท่านั้น

---

## Step 552: `render json:` กับกลไก `as_json`/`to_json` เบื้องหลัง

### วิธีที่ง่ายที่สุดในการตอบ JSON

```ruby
# app/controllers/demo_controller.rb
class DemoController < ApplicationController
  def basic
    post = Post.first
    render json: post
  end
end
```

```bash
curl -s http://127.0.0.1:3098/demo/basic
```

ผลลัพธ์ที่ทดสอบรันจริง:

```json
{"id":1,"title":"Rails API mode คืออะไร","body":"เนื้อหาเกี่ยวกับ Rails API only mode","user_id":1,"published":true,"created_at":"2026-09-26T07:37:43.523Z","updated_at":"2026-09-26T07:37:43.523Z"}
```

แค่ `render json: post` ก็ได้ JSON ที่มีทุก column ของ database row นั้นออกมาแล้ว — มันเกิดขึ้นได้
อย่างไร?

### เบื้องหลัง: `render json:` เรียก `to_json` ให้อัตโนมัติ

`render json: obj` คือ syntax sugar ของ Rails ที่ทำสิ่งเดียวกับ `obj.to_json` (ถ้า `obj` ยังไม่ใช่
String อยู่แล้ว) โดยตั้งค่า `Content-Type: application/json` ให้ด้วย ทดลองเรียกตรงๆ ใน
`bin/rails console` (ทดสอบรันจริงแล้ว):

```ruby
post = Post.first
post.to_json
# => "{\"id\":1,\"title\":\"Rails API mode คืออะไร\",\"body\":\"เนื้อหาเกี่ยวกับ Rails API only
#     mode\",\"user_id\":1,\"published\":true,\"created_at\":\"2026-09-26T07:37:43.523Z\",
#     \"updated_at\":\"2026-09-26T07:37:43.523Z\"}"
```

`to_json` ที่ ActiveSupport เพิ่มให้กับทุก object (ไม่ใช่แค่ ActiveRecord) มีขั้นตอนภายในคือ **เรียก
`as_json` ก่อนเสมอ แล้วค่อยแปลง Hash/Array ที่ได้เป็น JSON string** — พูดอีกแบบ: `as_json` ทำหน้าที่
"แปลง Ruby object เป็น Hash/Array/primitive ที่ปลอดภัยสำหรับ JSON" ส่วน `to_json` ทำหน้าที่ "แปลง
Hash/Array นั้นเป็น string สุดท้าย" (ใช้ `JSON.generate` ภายใน)

```ruby
post.as_json
# => {"id"=>1, "title"=>"Rails API mode คืออะไร", "body"=>"เนื้อหาเกี่ยวกับ Rails API only mode",
#     "user_id"=>1, "published"=>true, "created_at"=>"2026-09-26T07:37:43.523Z",
#     "updated_at"=>"2026-09-26T07:37:43.523Z"}
# ตัวแปรผลลัพธ์เป็น Hash ธรรมดา (key เป็น String) ไม่ใช่ JSON string
```

### `ActiveRecord::Base#as_json` ทำอะไรบ้าง โดย default

`as_json` ของ ActiveRecord (มาจาก `ActiveModel::Serializers::JSON`) โดย default จะ:

1. เอา **column ทุกตัวของ database table นั้น** มาใส่เป็น key-value ใน Hash (อ่านจาก
   `attributes` ของ record)
2. แปลงชนิดข้อมูลให้เหมาะกับ JSON โดยอัตโนมัติ (`Time`/`DateTime` → ISO 8601 string,
   `BigDecimal` → string หรือ number ตาม config, `nil` → `null`)
3. **ไม่รวม association ใดๆ เลย** (`user_id` เป็นแค่ foreign key column ธรรมดา ไม่ใช่ object
   `user` เต็มตัว) เว้นแต่จะสั่งเพิ่มเองด้วย option `include:`
4. **ไม่รวม method อื่นที่ไม่ใช่ column** เว้นแต่จะสั่งเพิ่มเองด้วย option `methods:`

### `Array` ของ ActiveRecord ก็ `as_json`/`to_json` ได้เหมือนกัน

```ruby
Post.all.as_json
# => [{"id"=>1, "title"=>"...", ...}, {"id"=>2, "title"=>"...", ...}]
```

`render json: Post.all` จึงตอบ JSON array ของทุก post ได้ทันทีโดยไม่ต้องเขียนอะไรเพิ่มเลย —
นี่คือเหตุผลที่ **ตัวอย่างสอน Rails API เบื้องต้นแทบทุกที่ใช้ `render json: @posts` ได้โดยไม่มี
serializer อะไรเลย** เพราะกลไก `as_json` เริ่มต้นนี้ครอบคลุมกรณีง่ายๆ ได้ดีอยู่แล้ว — ปัญหาจะเริ่ม
ปรากฏก็ต่อเมื่อ API ต้องการควบคุมรูปร่างของ JSON ให้ต่างจาก "ทุก column ของตาราง" ซึ่งเป็นหัวข้อของ
Step ถัดไป

> **สิ่งที่ต้องระวังจากพฤติกรรม default นี้:** ทุก column ของตารางถูกส่งออกไปหมด **รวมถึง column
> ที่อาจไม่ควรเปิดเผยให้ client เห็น** เช่น `password_digest`, `internal_notes`,
> `stripe_customer_id` — ถ้า model ไหนมี column แบบนี้อยู่ การ `render json: model_instance`
> ตรงๆ โดยไม่กรองอะไรเลยคือความเสี่ยงด้าน security ที่พบบ่อยมากในโค้ด production จริง (จัดอยู่ใน
> หมวด **mass assignment / over-exposure** ที่จะเจาะลึกเต็มรูปแบบใน **Part 080**)

---

## Step 553: ควบคุม JSON output ด้วย `only:`/`except:` และทำไม override `as_json` ถึงเป็น anti-pattern

### `only:`/`except:` บน `render json:` — วิธีที่เร็วที่สุดในการกรอง column

```ruby
def only
  post = Post.first
  render json: post, only: [:id, :title]
end
```

```bash
curl -s http://127.0.0.1:3098/demo/only
```

ผลลัพธ์ที่ทดสอบรันจริง:

```json
{"id":1,"title":"Rails API mode คืออะไร"}
```

หรือใช้ `except:` เมื่อต้องการ "เอาทุก column ยกเว้นบางตัว" (สะดวกกว่าตอน column มีเยอะและอยากซ่อน
แค่ 1–2 ตัว):

```ruby
post.as_json(except: [:created_at, :updated_at])
# => {"id"=>1, "title"=>"...", "body"=>"...", "user_id"=>1, "published"=>true}
```

(ทดสอบรันจริงแล้วทั้งสองแบบ ให้ผลตรงตามคาด) `only:`/`except:` ใช้ได้ทั้งกับ `render json:`
โดยตรง และกับ `.as_json(...)` ที่เรียกเอง เพราะทั้งคู่ pass option ไปที่ method เดียวกันข้างใน

### `include:` — ดึง association ติดมาด้วยแบบง่ายที่สุด

```ruby
post.as_json(include: :comments)
```

ผลลัพธ์ที่ทดสอบรันจริง (ตัดให้สั้นลง):

```json
{
  "id": 1, "title": "...", "body": "...", "user_id": 1, "published": true,
  "created_at": "...", "updated_at": "...",
  "comments": [
    {"id": 1, "post_id": 1, "author_name": "สมชาย", "body": "บทความดีมากครับ", "created_at": "...", "updated_at": "..."},
    {"id": 2, "post_id": 1, "author_name": "สมหญิง", "body": "รอตอนต่อไปอยู่เลย", "created_at": "...", "updated_at": "..."}
  ]
}
```

ใช้งานได้จริง แต่สังเกตปัญหาที่ตามมาทันที: **ควบคุม field ของ nested object ไม่ได้ง่ายๆ** ถ้า
ต้องการให้ `comments` แสดงแค่ `id`/`body` โดยไม่มี `post_id`/`created_at`/`updated_at` ต้องเขียน
ซ้อน hash แบบ `include: { comments: { only: [:id, :body] } }` ซึ่งอ่านยากขึ้นเรื่อยๆ เมื่อโครงสร้าง
ซับซ้อนขึ้น — นี่คือสัญญาณแรกที่บอกว่า **`only:`/`except:`/`include:` แบบ inline เหมาะกับ
prototype หรือ endpoint ง่ายๆ เท่านั้น** ไม่เหมาะกับ API ที่มีรูปร่าง JSON ซับซ้อนหรือต้องใช้ซ้ำ
หลายที่ (Step 555 จะเทียบทางเลือกที่ดีกว่าสำหรับกรณีนี้)

### ทางเลือกที่ดูน่าสนใจแต่เป็นกับดัก: Override `as_json` บน Model

หลาย tutorial เก่าแนะนำให้แก้ปัญหาข้างบนด้วยการ override `as_json` ตรงๆ ใน model:

```ruby
# app/models/post.rb — วิธีที่ "ดูเหมือนดี" แต่มีปัญหาแฝงอยู่
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments

  def as_json(options = {})
    { id: id, title: title, published: published }
  end
end
```

ทดสอบดูว่าเกิดอะไรขึ้น (รันจริงใน `bin/rails console`):

```ruby
post = Post.first
post.as_json
# => {:id=>1, :title=>"Rails API mode คืออะไร", :published=>true}

post.to_json
# => "{\"id\":1,\"title\":\"Rails API mode คืออะไร\",\"published\":true}"

# ลองส่ง option only: เข้าไปเหมือนเดิม...
post.as_json(only: [:id])
# => {:id=>1, :title=>"Rails API mode คืออะไร", :published=>true}   <-- ยังคืนครบ 3 field เหมือนเดิม!
```

**นี่คือปัญหาที่ทดสอบรันจริงแล้วเห็นชัดเจน:** เมธอด `as_json` ที่ override ทับไปใหม่ **รับ
`options` เข้ามาแต่ไม่ได้ใช้งานมันเลย** — controller ไหนก็ตามที่เคยเรียก
`render json: post, only: [:id]` โดยคาดหวังพฤติกรรม default ของ Rails (กรอง column ตามที่ขอ)
จะได้ผลลัพธ์ที่ผิดไปจากที่ตั้งใจทันที **โดยไม่มี error หรือ warning ใดๆ เตือนเลย** — เป็น
silent bug ที่ตรวจจับยากมากเพราะโค้ด controller ดูถูกต้องทุกอย่าง ปัญหาซ่อนอยู่ใน model ไฟล์
ที่ไกลออกไป

### ทำไมการ Override `as_json` ถึงถือเป็น Anti-pattern ในปัจจุบัน

1. **ผูก concern ผิดชั้น (mixing concerns):** Model ควรรับผิดชอบแค่ **business logic และ
   data integrity** (validation, association, scope) — "JSON ควรมีหน้าตาอย่างไรตอนตอบ API"
   เป็นเรื่องของ **presentation layer** คนละชั้นกันโดยสิ้นเชิง (แนวคิดเดียวกับที่ Part 024 สอน
   เรื่องแยก View ออกจาก Controller/Model) การยัดทั้งสองเรื่องไว้ใน method เดียวทำให้ model
   ใหญ่ขึ้นเรื่อยๆ และแก้ยากขึ้นเมื่อ requirement เปลี่ยน
2. **ใช้ได้แค่รูปร่างเดียว:** ถ้า endpoint `GET /posts` (list) ต้องการ JSON แบบสั้น (`id`,
   `title` เท่านั้น) แต่ `GET /posts/:id` (detail) ต้องการ JSON แบบเต็ม (มี `comments`,
   `author` ด้วย) — override `as_json` แบบตายตัวแบบนี้**ทำได้แค่รูปร่างเดียวเท่านั้น** จะต้องเขียน
   `if`/`case` เช็ค context ข้างในตัว method เอง (เช่น เช็ค `options[:view]`) ซึ่งพา code กลับไปสู่
   ปัญหาเดิมคือ options ไม่ทำงานตามที่คาดถ้าไม่ตั้งใจ implement เองให้ครบทุกกรณี
3. **ทดสอบยากขึ้น:** การทดสอบ "JSON ที่ API คืนกลับมาถูกต้องไหม" ควรเป็นความรับผิดชอบของ
   request spec/serializer spec (เรียนใน **Part 046**) แต่พอ logic นี้ไปอยู่ใน model method
   ที่ชื่อ `as_json` (ชื่อเดียวกับที่ Rails framework ใช้ภายใน) การเขียน test แยกเฉพาะส่วน
   serialization จาก business logic อื่นของ model ทำได้ไม่สะดวก
4. **ชนกับ internal use ของ Rails เอง:** Rails framework เองก็เรียก `as_json`/`to_json`
   ในหลายจุดภายใน (เช่น logging, cache key generation บางกรณี) การ override method ที่มีชื่อ
   นี้ตรงๆ เสี่ยงชนกับพฤติกรรมที่ Rails คาดหวังในจุดที่ไม่ได้ตั้งใจ

> **สรุปหลักการสำคัญของ Step นี้:** `only:`/`except:`/`include:` แบบ inline **ใช้ได้ดีสำหรับ
> กรณีง่ายๆ ชั่วคราว** (prototype, debug endpoint) แต่**ไม่ควรเป็นกลยุทธ์หลักของ API ที่มีขนาด
> กลาง-ใหญ่** และการ **override `as_json` บน model ไม่ใช่คำตอบที่ถูกต้อง** เพราะผูก concern
> ผิดชั้นและเปิดช่องให้เกิด silent bug ได้ง่าย — คำตอบที่ถูกต้องคือแยก "การตัดสินใจว่า JSON
> หน้าตาเป็นอย่างไร" ออกมาเป็น**object ต่างหาก** ซึ่งมีสองแนวทางหลักที่ระบบนิเวศ Rails ใช้จริงใน
> ปัจจุบัน: **Jbuilder** (Step 554) และ **serializer gem แบบ plain Ruby class** เช่น Blueprinter
> (Step 555–556)

---

## Step 554: Jbuilder — เครื่องมือสร้าง JSON แบบ view ที่มากับ Rails

### เปิดใช้งาน Jbuilder

เปิด `Gemfile` แล้วเอา `#` ออกจากบรรทัดที่ Rails generate ไว้ให้ตั้งแต่ Step 551:

```ruby
# Gemfile
gem "jbuilder"
```

```bash
bundle install
```

### แนวคิด: JSON เป็น "View" เหมือน ERB เป็น "View" ของ HTML

ตลอด Phase 3–7 เราคุ้นเคยกับการที่ controller action หนึ่งตัวจะ render **ไฟล์ view** ที่ชื่อ
ตรงกับ action นั้นโดยอัตโนมัติ (`app/views/posts/show.html.erb` สำหรับ `PostsController#show`)
— **Jbuilder ใช้หลักการเดียวกันเป๊ะ** เพียงแค่เปลี่ยนนามสกุลไฟล์เป็น `.json.jbuilder` แทน
`.html.erb`:

```ruby
# app/controllers/demo_controller.rb
class DemoController < ApplicationController
  def jbuilder_show
    @post = Post.find(params[:id])
    # ไม่มี render ตรงๆ — Rails จะหา app/views/demo/jbuilder_show.json.jbuilder ให้อัตโนมัติ
    # ตามชื่อ controller + action เหมือนกับ .html.erb ทุกประการ
  end
end
```

```ruby
# app/views/demo/jbuilder_show.json.jbuilder
json.id @post.id
json.title @post.title
json.body @post.body
json.published @post.published

json.author do
  json.id @post.user.id
  json.email @post.user.email
end

json.comments @post.comments do |comment|
  json.id comment.id
  json.author_name comment.author_name
  json.body comment.body
end

json.comments_count @post.comments.size
```

```bash
curl -s http://127.0.0.1:3098/demo/jbuilder_show/1
```

ผลลัพธ์ที่ทดสอบรันจริง:

```json
{"id":1,"title":"Rails API mode คืออะไร","body":"เนื้อหาเกี่ยวกับ Rails API only mode","published":true,"author":{"id":1,"email":"author@example.com"},"comments":[{"id":1,"author_name":"สมชาย","body":"บทความดีมากครับ"},{"id":2,"author_name":"สมหญิง","body":"รอตอนต่อไปอยู่เลย"}],"comments_count":2}
```

ทดสอบรันจริงใน Rails 8.1.4 โหมด `--api` ยืนยันว่า **Jbuilder ทำงานได้ทันทีโดยไม่ต้องปิด
`config.api_only` หรือเปิด module อะไรเพิ่มเติมเลย** — แค่เปิด gem แล้วสร้างไฟล์ view ตามชื่อที่
ถูกต้องก็พอ (Jbuilder เชื่อมตัวเองเข้ากับ template handler ของ `ActionView` ซึ่งยังทำงานอยู่
เบื้องหลังแม้ใน `ActionController::API` ก็ตาม)

### DSL ของ `json.*` อธิบายทีละบรรทัด

- `json.id @post.id` — กำหนด key `"id"` ให้มีค่าตามที่ระบุ (รูปแบบ `json.<key> <value>`)
- `json.author do ... end` — สร้าง **nested object** โดยทุกอย่างข้างในบล็อกกลายเป็น field ของ
  key `"author"`
- `json.comments @post.comments do |comment| ... end` — เมื่อ argument ตัวที่สองเป็น
  **collection** (Array/ActiveRecord::Relation) Jbuilder จะ **loop ให้อัตโนมัติ** แล้วสร้าง
  JSON array โดยเนื้อหาข้างในบล็อกกำหนดรูปร่างของแต่ละ element — เทียบเท่ากับ `.map` ใน Ruby
  ธรรมดา แต่เขียนกระชับกว่า
- ลำดับการเขียนใน template **คือลำดับ key ที่ปรากฏใน JSON output จริง** (ต่างจาก Hash ธรรมดาที่
  ไม่รับประกันลำดับเสมอไปในบาง context) เหมาะกับตอนต้องการ JSON ที่มี field เรียงตามที่ต้องการ
  ชัดเจน

### Partial ใน Jbuilder — DRY เหมือน ERB partial

ปัญหาที่ตามมาเร็วมากคือ **โครงสร้าง comment เดียวกันนี้อาจต้องใช้ซ้ำในหลาย endpoint** (เช่นทั้ง
`show` และ `index`) — Jbuilder รองรับ partial แบบเดียวกับ ERB partial (ทบทวนแนวคิด partial จาก
**Part 024**):

```ruby
# app/views/comments/_comment.json.jbuilder
json.id comment.id
json.author_name comment.author_name
json.body comment.body
```

```ruby
# app/views/demo/jbuilder_show.json.jbuilder (แก้ให้เรียก partial แทน)
json.id @post.id
json.title @post.title
json.comments @post.comments, partial: "comments/comment", as: :comment
```

`json.comments @post.comments, partial: "...", as: :comment` เทียบเท่ากับ
`render partial: "comments/comment", collection: @post.comments, as: :comment` ใน ERB — Rails
loop ให้เองและ render partial ให้กับแต่ละ comment

### ข้อดี/ข้อจำกัดของ Jbuilder ที่ต้องรู้ก่อนตัดสินใจใช้จริง

**ข้อดี:**

- **มากับ Rails อยู่แล้ว** (แค่เอา `#` ออก) ไม่ต้องประเมิน gem ภายนอกเพิ่ม
- **สอดคล้องกับแนวคิด MVC เดิมของ Rails** — คนที่คุ้นกับ ERB view จะเรียนรู้ Jbuilder ได้เร็วมาก
  เพราะ mental model เหมือนกันเป๊ะ (controller ไม่ต้อง render เอง, ไฟล์ view ตามชื่อ action,
  partial, layout — ทุกอย่างที่เคยเรียนมาใช้ได้เลย)
- เหมาะมากกับ endpoint ที่ **ต้องปรับแต่งรูปร่าง JSON เฉพาะจุดเยอะ** (field ที่คำนวณเฉพาะ, logic
  แสดงผลที่ซับซ้อนแบบ conditional) เพราะเป็น Ruby DSL ธรรมดา เขียน `if`/`unless` ปนได้อิสระ

**ข้อจำกัดที่ต้องรู้:**

- **Template อยู่แยกไฟล์จาก controller และ model** ทำให้ตอนอ่านโค้ดต้องเปิดไฟล์เพิ่มอีกไฟล์เสมอ
  เพื่อรู้ว่า endpoint หนึ่งตอบอะไรกลับไปบ้าง (ต่างจาก serializer class ที่มักอยู่ในไฟล์เดียวจบ)
- **ทดสอบยากกว่า plain Ruby object เล็กน้อย** เพราะต้อง render ผ่าน view context
  (`ActionView::Base`) จริงๆ ถึงจะทดสอบได้เต็มรูปแบบ ต่างจาก serializer class ที่เป็น Ruby
  object ธรรมดา เรียก `.new(post).serialize` ใน unit test ได้ตรงๆ โดยไม่ต้องพึ่ง Rails view
  layer เลย
- **Performance** — Jbuilder ต้องผ่าน ActionView template rendering pipeline เต็มรูปแบบ (compile
  template, evaluate ใน view context) ซึ่งมี overhead มากกว่า serializer แบบ plain Ruby class
  ที่ทำแค่ loop สร้าง Hash ตรงๆ (ความต่างชัดเจนขึ้นเมื่อ endpoint ต้อง serialize ข้อมูลจำนวนมาก
  ต่อ request)

---

## Step 555: เปรียบเทียบแนวทาง Serialization ที่ใช้จริงในปัจจุบัน

Step นี้คือหัวใจสำคัญที่สุดของ Part นี้ — เพราะระบบนิเวศเครื่องมือ serialize JSON ใน Ruby
เปลี่ยนไปมากในช่วงหลายปีที่ผ่านมา คำแนะนำจาก tutorial ปี 2015–2018 จำนวนมากที่บอกว่า
"ใช้ `active_model_serializers`" **ไม่ใช่คำแนะนำที่ถูกต้องอีกต่อไปสำหรับโปรเจกต์ใหม่ในปี 2026**

### ทางเลือกที่ 1: `active_model_serializers` (AMS) — เข้าใจสถานะปัจจุบันให้ถูกต้อง

`active_model_serializers` เคยเป็น **มาตรฐานโดยพฤตินัย (de facto standard)** ของวงการ Rails API
ในช่วงปี 2014–2018 มี syntax คล้าย Rails model เดิม:

```ruby
# ตัวอย่าง syntax ของ active_model_serializers (แสดงเพื่อการอ้างอิงเท่านั้น
# หลักสูตรนี้ไม่แนะนำให้เริ่มโปรเจกต์ใหม่ด้วย gem นี้ ดูเหตุผลด้านล่าง)
class PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :body
  has_many :comments
end
```

**สถานะที่แท้จริงในปัจจุบัน (ตรวจสอบข้อมูลล่าสุดแล้ว):** repository
`rails-api/active_model_serializers` อยู่ในสถานะ **maintenance mode** เท่านั้น — ยังมีการออก
เวอร์ชันแก้ bug เป็นระยะบน branch `0-10-stable` (เวอร์ชันล่าสุด `0.10.16`) แต่**แทบไม่มีการพัฒนา
ฟีเจอร์ใหม่แล้ว** และ **ผู้ดูแลชุดเดิมส่วนใหญ่ (ทั้งจากยุค 0.8, 0.9 และ 0.10 ตอนต้น) ไม่ได้ทำงาน
กับ gem นี้อีกต่อไป** พูดตรงๆ คือ **gem ยังไม่ตาย แต่ก็ไม่ใช่ทางเลือกที่ทีมควรเลือกสำหรับ
โปรเจกต์ใหม่ในปี 2026** เพราะระบบนิเวศได้ย้ายไปยังเครื่องมือรุ่นใหม่ที่เบากว่า เร็วกว่า และมีคนดูแล
เชิงรุกมากกว่าแล้ว — **หลักสูตรนี้จะไม่ใช้ AMS เป็นตัวอย่างหลักด้วยเหตุผลนี้** (ถ้าเจอโค้ด
production เก่าที่ใช้ AMS อยู่ สามารถอ่านเข้าใจได้ไม่ยากเพราะ syntax คล้าย ActiveRecord แต่ไม่ควร
เลือกมันสำหรับโค้ดใหม่)

### ทางเลือกที่ 2: Jbuilder — ที่เรียนไปแล้วใน Step 554

มากับ Rails, สอดคล้องกับแนวคิด MVC, เหมาะกับทีมที่คุ้นกับ ERB view อยู่แล้ว — ยังคงเป็นทางเลือกที่
**สมเหตุสมผลอย่างยิ่ง** สำหรับหลายทีมโดยเฉพาะทีมที่มาจากพื้นฐาน Rails แบบดั้งเดิม

### ทางเลือกที่ 3: Serializer แบบ Plain Ruby Class (Blueprinter / Alba)

แนวทางใหม่ที่ได้รับความนิยมมากขึ้นเรื่อยๆ ในช่วงหลายปีที่ผ่านมาคือ **serializer เป็น Ruby class
ธรรมดา** (ไม่ใช่ view template) ที่ประกาศ field ที่ต้องการแบบ declarative — ตัวที่ได้รับความนิยม
มากที่สุดสองตัวในปัจจุบันคือ **Blueprinter** และ **Alba**

**Blueprinter** (โดยทีม Procore):

```ruby
class PostBlueprint < Blueprinter::Base
  identifier :id
  fields :title, :body
end

PostBlueprint.render(post)   # => JSON string
```

**Alba** (โดย okuramasafumi) — เน้นความเร็วและ syntax ที่กระชับมาก:

```ruby
class PostResource
  include Alba::Resource
  attributes :id, :title, :body
end

PostResource.new(post).serialize   # => JSON string
```

ทั้งสอง gem นี้เป็น**ผู้เล่นหลักในสาย "serializer แบบ plain Ruby object" ของปัจจุบัน** จากการ
ตรวจสอบข้อมูลเปรียบเทียบล่าสุด ทั้งคู่ถูกจัดเป็น **modern high-performance serializer** ที่เร็วกว่า
แนวทางแบบเก่าอย่างมีนัยสำคัญ (ผลทดสอบ benchmark หนึ่งชุดพบว่า Alba เร็วกว่า Blueprinter เล็กน้อย
แต่ทั้งคู่เร็วกว่า AMS แบบเก่าหลายเท่าตัว) และทั้งคู่ยังคง **active development** อยู่จริง (ไม่ใช่
แค่ maintenance mode เหมือน AMS)

### ตารางเปรียบเทียบแบบตรงไปตรงมา (สถานะที่ตรวจสอบแล้ว)

| ประเด็น | `render json:` + `as_json` ล้วน | Jbuilder | Blueprinter / Alba | `active_model_serializers` |
|---|---|---|---|---|
| ต้องติดตั้ง gem เพิ่มไหม | ไม่ต้อง | ต้องเปิด (มากับ Rails แล้ว) | ต้องติดตั้งเอง | ต้องติดตั้งเอง |
| สถานะการดูแล (2026) | เป็นส่วนหนึ่งของ Rails core | Active (ทีม Rails core ดูแล) | **Active** ทั้งคู่ | **Maintenance mode เท่านั้น** ไม่แนะนำสำหรับโปรเจกต์ใหม่ |
| รูปแบบไฟล์ | inline ใน controller/model | View template แยกไฟล์ (`.json.jbuilder`) | Plain Ruby class แยกไฟล์ | Plain Ruby class แยกไฟล์ (คล้าย model) |
| ทดสอบแบบ unit test ง่ายแค่ไหน | ง่าย (เป็น Hash ธรรมดา) | ต้องผ่าน view context | **ง่ายมาก** (เรียก `.render`/`.serialize` ตรงๆ) | ง่าย (แต่ gem ไม่แนะนำแล้ว) |
| Performance | เร็ว (built-in) | ปานกลาง (ผ่าน template pipeline) | **เร็วมาก** (ออกแบบมาเพื่อความเร็วโดยเฉพาะ) | ช้ากว่าทางเลือกสมัยใหม่ |
| เหมาะกับ | prototype, endpoint ง่ายๆ | ทีมที่คุ้น Rails view แบบดั้งเดิม, logic การแสดงผลซับซ้อน | API ขนาดกลาง-ใหญ่ที่ต้องการ structure ชัดเจน, หลาย view ต่อ 1 model | โค้ดเก่าที่มีอยู่แล้ว (ไม่แนะนำเริ่มใหม่) |
| Nested/association | `include:` (จำกัด ควบคุมยากขึ้นเมื่อซับซ้อน) | `json.xxx do...end` (ยืดหยุ่นมาก) | `association`/`many`/`one` (declarative ชัดเจน) | `has_many`/`belongs_to` (คล้าย AR) |

> **ข้อสรุปที่ตรงไปตรงมาที่สุดของ Step นี้:** ไม่มีคำตอบเดียวที่ "ถูกต้องเสมอ" — **Jbuilder** และ
> **serializer แบบ plain Ruby class (Blueprinter/Alba)** ต่างก็เป็นทางเลือกที่ทีมมืออาชีพเลือกใช้
> จริงในปี 2026 ขึ้นอยู่กับสไตล์ทีมและลักษณะ API สิ่งเดียวที่ชัดเจนคือ **`active_model_serializers`
> ไม่ใช่คำแนะนำที่เหมาะสมสำหรับโปรเจกต์ใหม่อีกต่อไป** เพราะอยู่ในสถานะ maintenance mode มานาน

---

## Step 556: เลือกแนวทางของหลักสูตรนี้ และสร้าง Blueprint แรกด้วย Blueprinter

### เหตุผลที่หลักสูตรนี้เลือก Blueprinter เป็นตัวอย่างหลักของ Part นี้

ทั้ง Jbuilder และ Blueprinter/Alba ล้วนเป็นคำตอบที่ยอมรับได้ — **แบบฝึกหัดปิดท้าย Part นี้ให้
ทดลองเขียนด้วย Jbuilder เป็นทางเลือกเสริมด้วย** แต่เนื้อหาหลักของ Step 556–560 เลือกใช้
**Blueprinter** เป็นตัวอย่างหลัก ด้วยเหตุผล 3 ข้อ:

1. **ทดสอบเป็น unit test ได้ตรงไปตรงมาที่สุด** — Blueprint เป็น Ruby class ธรรมดา ไม่ต้องพึ่ง
   Rails view layer เลย เขียน spec เรียก `PostBlueprint.render(post)` ตรงๆ ได้ทันที (สำคัญมาก
   เพราะ **Phase 6: Testing** ที่เพิ่งเรียนจบไปเน้นย้ำเรื่องนี้ตลอด — code ที่ทดสอบง่ายคือ code
   ที่ดี)
2. **แยก concern ชัดเจนที่สุด** — ไฟล์ blueprint หนึ่งไฟล์ตอบคำถามเดียวคือ "model นี้แปลงเป็น
   JSON แบบไหนได้บ้าง (มีกี่ view)" ไม่ปนกับ logic การ render แบบ template ทั่วไป
3. **รองรับหลาย "view" ของ model เดียวกันได้ในไฟล์เดียว** ตรงตามที่ Step 553 ชี้ปัญหาไว้ว่า
   endpoint `index` กับ `show` มักต้องการรูปร่าง JSON ต่างกัน — ฟีเจอร์นี้ของ Blueprinter ตอบโจทย์
   นี้ได้ตรงจุดมาก (จะเห็นเต็มรูปแบบใน Step 557)

> **ข้อควรจำ:** นี่คือ**การเลือกของหลักสูตรเพื่อความสม่ำเสมอของตัวอย่างที่เหลือใน Part นี้**
> ไม่ใช่คำตัดสินว่า Jbuilder ด้อยกว่า — ทีมจริงจำนวนมากเลือก Jbuilder ด้วยเหตุผลที่สมเหตุสมผลไม่
> แพ้กัน (ดูตารางเปรียบเทียบ Step 555 อีกครั้งประกอบการตัดสินใจในโปรเจกต์ของตัวเอง)

### ติดตั้ง Blueprinter

```ruby
# Gemfile
gem "blueprinter", "~> 1.1"
```

```bash
bundle install
```

(ทดสอบรันจริงแล้ว constraint `~> 1.1` resolve เป็นเวอร์ชัน `1.3.0` ซึ่งเป็นเวอร์ชันล่าสุดที่
ใช้งานได้ ณ ตอนเขียน Part นี้)

### Blueprint แรก — `PostBlueprint` แบบพื้นฐาน

```bash
mkdir -p app/blueprints
```

```ruby
# app/blueprints/post_blueprint.rb
class PostBlueprint < Blueprinter::Base
  identifier :id

  fields :title, :body, :published, :created_at
end
```

**อธิบาย:**

- `Blueprinter::Base` คือ base class ที่ทุก blueprint ต้อง inherit — เทียบเท่ากับที่
  `ApplicationRecord` เป็น base class ของทุก model
- `identifier :id` ประกาศว่า field ไหนคือ "ตัวระบุตัวตน" ของ object นี้ — Blueprinter ใส่
  field นี้ไว้เป็นอันดับแรกสุดของ output เสมอ (คล้ายกับที่ตาราง database มักมี `id` เป็นคอลัมน์
  แรก)
- `fields :title, :body, :published, :created_at` ประกาศ field อื่นๆ ที่ต้องการรวมใน JSON —
  **ต่างจาก `as_json` default ตรงที่ Blueprinter ไม่รวมทุก column ให้อัตโนมัติ ต้องประกาศเอง
  ทีละตัว** ซึ่งเป็นข้อดีด้าน security เพราะ**ไม่มีทางลืมซ่อน column ที่ไม่ควรเปิดเผยโดยไม่ตั้งใจ**
  (ตรงข้ามกับความเสี่ยงที่พูดถึงท้าย Step 552)

### เรียกใช้ Blueprint จาก Controller

```ruby
class PostsController < ApplicationController
  def show
    post = Post.find(params[:id])
    render json: PostBlueprint.render(post)
  end
end
```

`PostBlueprint.render(post)` คืนค่าเป็น **JSON string ที่พร้อมส่งออกแล้ว** (ต่างจาก
`PostBlueprint.render_as_hash(post)` ที่คืนเป็น Hash ธรรมดาถ้าต้องการนำไปประมวลผลต่อก่อน) —
`render json: <json_string>` รู้จักรับ String ที่เป็น valid JSON อยู่แล้วได้โดยตรง ไม่ต้อง
`to_json` ซ้ำอีกรอบ

---

## Step 557: Serializer สำหรับ `Post` ที่มี `Comment` ซ้อนกัน (nested) พร้อมหลาย view

### ปัญหาที่ Step 553 ทิ้งไว้: `index` กับ `show` ต้องการ JSON คนละแบบ

Endpoint แบบ `GET /posts` (รายการ) มักต้องการ JSON **แบบสั้น** เพื่อประหยัด bandwidth (ไม่ต้องส่ง
comment ทั้งหมดของทุกโพสต์มาด้วยตอนแสดงแค่รายการ) ในขณะที่ `GET /posts/:id` (รายละเอียด) ต้องการ
JSON **แบบเต็ม** พร้อม comment ทั้งหมดฝังมาด้วย — Blueprinter มีฟีเจอร์ **`view`** ที่ตอบโจทย์นี้
ได้ตรงจุด:

```ruby
# app/blueprints/comment_blueprint.rb
class CommentBlueprint < Blueprinter::Base
  identifier :id

  fields :author_name, :body, :created_at
end
```

```ruby
# app/blueprints/post_blueprint.rb
class PostBlueprint < Blueprinter::Base
  identifier :id

  fields :title, :body, :published, :created_at

  view :list do
    field :comments_count do |post, _options|
      # ใช้ post.comments.size ไม่ใช่ .count เพราะถ้า comments ถูก preload ไว้แล้วด้วย
      # includes/preload, .size จะนับจาก array ที่โหลดมาแล้วในหน่วยความจำ ไม่ยิง query
      # ซ้ำอีกรอบ ต่างจาก .count ที่ยิง SQL COUNT(*) ใหม่ทุกครั้งเสมอ (ทบทวนจาก Part 034)
      post.comments.size
    end
  end

  view :detail do
    association :comments, blueprint: CommentBlueprint

    field :author do |post, _options|
      { id: post.user.id, email: post.user.email }
    end
  end
end
```

### ทดสอบทั้งสอง view จริง

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    posts = Post.includes(:comments, :user).order(created_at: :desc)
    render json: PostBlueprint.render(posts, view: :list)
  end

  def show
    post = Post.find(params[:id])
    render json: PostBlueprint.render(post, view: :detail)
  end
end
```

```bash
curl -s http://127.0.0.1:3098/posts
```

ผลลัพธ์ที่ทดสอบรันจริง (`view: :list`):

```json
[{"id":1,"body":"เนื้อหาเกี่ยวกับ Rails API only mode","comments_count":2,"created_at":"2026-09-26 07:37:43 UTC","published":true,"title":"Rails API mode คืออะไร"}]
```

```bash
curl -s http://127.0.0.1:3098/posts/1
```

ผลลัพธ์ที่ทดสอบรันจริง (`view: :detail`):

```json
{"id":1,"author":{"id":1,"email":"author@example.com"},"body":"เนื้อหาเกี่ยวกับ Rails API only mode","comments":[{"id":1,"author_name":"สมชาย","body":"บทความดีมากครับ","created_at":"2026-09-26 07:37:43 UTC"},{"id":2,"author_name":"สมหญิง","body":"รอตอนต่อไปอยู่เลย","created_at":"2026-09-26 07:37:43 UTC"}],"created_at":"2026-09-26 07:37:43 UTC","published":true,"title":"Rails API mode คืออะไร"}
```

ทดสอบรันจริงยืนยันว่า **view เดียวกันของ model เดียวกัน (`Post`) ให้ JSON คนละรูปร่างได้ตามที่
ต้องการ** — `:list` ไม่มี `comments`/`author` เลย มีแค่ `comments_count` แบบตัวเลขสรุป ในขณะที่
`:detail` มี `comments` เป็น array ซ้อนเต็มรูปแบบผ่าน `CommentBlueprint` และมี `author` เป็น field
ที่คำนวณเอง (custom field block)

**อธิบาย syntax ใหม่ที่ใช้:**

- `view :list do ... end` / `view :detail do ... end` — ประกาศ "มุมมอง" ของ blueprint เดียวกัน
  ที่มี field ต่างกันได้ โดย field ที่ประกาศไว้นอก `view` block (เช่น `fields :title, :body, ...`
  ด้านบนสุด) จะถูกรวมเข้าไปใน**ทุก view โดยอัตโนมัติ** (เป็น field พื้นฐานร่วมกัน) ส่วน field ที่
  ประกาศไว้ข้างใน `view` block เฉพาะจะมีแค่ตอนเรียก view นั้นเท่านั้น
- `association :comments, blueprint: CommentBlueprint` — บอก Blueprinter ว่า field `comments`
  ควรถูก serialize ด้วย `CommentBlueprint` (ไม่ใช่แค่ dump ทุก column แบบ `as_json` เดิม) ทำให้
  ควบคุมรูปร่างของ nested object ได้แม่นยำเท่ากับ object ระดับบนสุด — แก้ปัญหาที่ Step 553 เจอกับ
  `include: { comments: { only: [...] } }` ที่ซับซ้อนขึ้นเรื่อยๆ ได้อย่างสวยงาม
- `field :comments_count do |post, _options| ... end` / `field :author do |post, _options| ... end`
  — ประกาศ **custom field** ที่คำนวณจาก logic เองแทนที่จะดึงตรงจาก attribute ของ model —
  block รับ 2 argument: object ที่กำลัง serialize (`post`) และ `options` ที่ส่งเข้ามาตอนเรียก
  `.render(post, options_ที่ส่งมาด้วย)` (เผื่อกรณีต้องการ context เพิ่มเติม เช่น current_user
  สำหรับคำนวณ field ที่ขึ้นกับสิทธิ์ผู้ใช้)

---

## Step 558: หลีกเลี่ยง N+1 ตอน Serialize ข้อมูลที่มี Association

### ปัญหาคลาสสิกที่ทบทวนจาก Part 034: Serializer ที่ดูสวยงามแต่ซ่อน N+1 ไว้

ตัวอย่าง `PostBlueprint` ด้านบนดูสวยงามและอ่านง่าย แต่มีกับดักซ่อนอยู่ — ถ้า controller เขียนแบบนี้
โดยลืม `includes`:

```ruby
def index
  posts = Post.order(created_at: :desc)   # ลืม .includes(:comments, :user)!
  render json: PostBlueprint.render(posts, view: :list)
end
```

`view :list` เรียก `post.comments.size` สำหรับทุกโพสต์ — ถ้าไม่ preload ไว้ก่อน แต่ละครั้งที่เรียก
`post.comments.size` (สำหรับโพสต์ที่ยังไม่เคยโหลด `comments` มาก่อน) จะยิง SQL แยกออกไปทันที **นี่
คือ N+1 query แบบเดียวกับที่ Part 034 สอนไว้ทุกประการ** เพียงแค่คราวนี้จุดที่เรียก association
ซ่อนอยู่ใน serializer แทนที่จะอยู่ใน view ERB ตรงๆ

### พิสูจน์ด้วยการนับ query จริง

```ruby
query_count = 0
ActiveSupport::Notifications.subscribe("sql.active_record") { query_count += 1 }

Post.first  # warmup query กันไม่ให้นับ query แรกที่โหลด schema/cache ปนเข้ามา

# ===== ไม่ใช้ includes =====
query_count = 0
posts = Post.order(:id).to_a
posts.each { |p| p.comments.size }
puts "ไม่ใช้ includes: #{query_count} queries"

# ===== ใช้ includes(:comments) =====
query_count = 0
posts2 = Post.includes(:comments).order(:id).to_a
posts2.each { |p| p.comments.size }
puts "ใช้ includes(:comments): #{query_count} queries"
```

ผลลัพธ์ที่ทดสอบรันจริง (มี 3 โพสต์ในข้อมูลตัวอย่าง แต่ละโพสต์มี comment ไม่เท่ากัน):

```
ไม่ใช้ includes: 4 queries
ใช้ includes(:comments): 2 queries
```

**อธิบายตัวเลข:** แบบไม่ใช้ `includes` เกิด **1 query สำหรับดึง posts ทั้งหมด + 3 query แยก**
(หนึ่ง query ต่อโพสต์หนึ่งอัน เพื่อนับ comment ของโพสต์นั้น) รวมเป็น 4 — นี่คือรูปแบบ N+1 ตรงตัว
(N = จำนวนโพสต์) ส่วนแบบใช้ `includes(:comments)` เกิดแค่ **2 query คงที่เสมอ** (1 สำหรับ posts,
1 สำหรับ comments ทั้งหมดของทุกโพสต์ในครั้งเดียวด้วย `WHERE post_id IN (...)`) **ไม่ว่าจะมีโพสต์
กี่ร้อยกี่พันอันก็ตาม** ตัวเลขนี้จะยิ่งต่างกันมากขึ้นเรื่อยๆ เมื่อจำนวนโพสต์เพิ่มขึ้น (ทบทวนหลักการ
`includes`/`preload`/`eager_load` แบบเต็มรูปแบบได้ที่ **Part 034 Step 334–335**)

### กฎการปฏิบัติจริงเมื่อเขียน Controller คู่กับ Serializer ที่มี Association

```ruby
class PostsController < ApplicationController
  def index
    # .includes(:comments) ป้องกัน N+1 ตอนเรียก post.comments.size ใน PostBlueprint view :list
    # .includes(:user) ป้องกัน N+1 ถ้า view ไหนต้องใช้ผู้เขียนด้วย (เผื่ออนาคต)
    posts = Post.includes(:comments, :user).order(created_at: :desc)
    render json: PostBlueprint.render(posts, view: :list)
  end

  def show
    post = Post.includes(:comments, :user).find(params[:id])
    render json: PostBlueprint.render(post, view: :detail)
  end
end
```

> **หลักการสำคัญที่ต้องยึดถือเสมอ:** **ทุกครั้งที่ serializer/blueprint/jbuilder template เรียก
> association ของ model** (ไม่ว่าจะเป็น `.comments`, `.user`, `.category` หรืออะไรก็ตาม) **ต้อง
> ตรวจสอบว่า controller `includes` association นั้นไว้ก่อนแล้วเสมอ** — วิธีที่ปลอดภัยที่สุดในการ
> ตรวจสอบระหว่างพัฒนาคือติดตั้ง gem **`bullet`** ที่จะแจ้งเตือนอัตโนมัติทันทีที่ตรวจพบ N+1 ระหว่าง
> รัน development server (จะเรียนเต็มรูปแบบใน **Part 064: Database Performance**) แต่ในระหว่างที่
> ยังไม่ได้ติดตั้ง `bullet` การนับ query ด้วย `ActiveSupport::Notifications` แบบข้างบนคือวิธี
> ตรวจสอบด้วยมือที่ตรงไปตรงมาที่สุด

---

## Step 559: HTTP Status Code ที่ถูกต้องสำหรับ API Response

### ทำไม Status Code ถึงสำคัญมากสำหรับ API (มากกว่าเว็บที่ render HTML)

เว็บแอปที่ render HTML ให้ browser ผู้ใช้มักไม่สนใจ HTTP status code มากนัก เพราะผู้ใช้ "เห็น"
หน้าเว็บที่ render ออกมาโดยตรงอยู่แล้วไม่ว่า status code จะเป็นอะไร แต่สำหรับ **API ที่ client เป็น
โปรแกรม** (mobile app, SPA, service อื่น) — **status code คือสัญญาณแรกที่ client ใช้ตัดสินใจว่า
จะประมวลผล response อย่างไร** ก่อนจะแตะเนื้อหา JSON ด้วยซ้ำ (เช่น library อย่าง `fetch`/`axios`
ฝั่ง JavaScript จะ throw error อัตโนมัติถ้า status code อยู่ในช่วง 4xx/5xx) — ใช้ status code
ผิดแม้ body จะถูกต้อง ก็ทำให้ client ประมวลผลผิดพลาดได้ทันที

### ตารางอ้างอิง HTTP Status Code สำหรับ RESTful API

| Status Code | Symbol ใน Rails | ใช้เมื่อไหร่ | ตัวอย่าง action |
|---|---|---|---|
| **200 OK** | `:ok` (default ของ `render json:`) | คำขอสำเร็จ มีข้อมูลส่งกลับ | `GET /posts`, `GET /posts/:id`, `PATCH /posts/:id` ที่สำเร็จ |
| **201 Created** | `:created` | สร้าง resource ใหม่สำเร็จ | `POST /posts` ที่สำเร็จ |
| **204 No Content** | `:no_content` | คำขอสำเร็จ แต่**ไม่มี body ส่งกลับ** | `DELETE /posts/:id` ที่สำเร็จ |
| **400 Bad Request** | `:bad_request` | Request ผิดรูปแบบตั้งแต่ต้น (JSON parse ไม่ได้, ขาด parameter ที่จำเป็น) | ขาด key `post` ใน body ของ `POST /posts` |
| **401 Unauthorized** | `:unauthorized` | ไม่ได้ยืนยันตัวตน หรือยืนยันตัวตนไม่ผ่าน | ไม่มี/ผิด JWT (ทบทวนจาก **Part 045**) |
| **403 Forbidden** | `:forbidden` | ยืนยันตัวตนผ่านแล้ว แต่**ไม่มีสิทธิ์**ทำสิ่งนี้ | ผู้ใช้ทั่วไปพยายามลบโพสต์ของคนอื่น (ทบทวน Pundit จาก **Part 043**) |
| **404 Not Found** | `:not_found` | ไม่พบ resource ที่ระบุ | `GET /posts/999` ที่ไม่มีอยู่จริง |
| **422 Unprocessable Entity** | `:unprocessable_entity` | Request เข้าใจได้ แต่ **ข้อมูลไม่ผ่าน validation** | `POST /posts` ที่ `title` ว่างเปล่า |
| **429 Too Many Requests** | `:too_many_requests` | โดน rate limit | เรียก API ถี่เกินที่กำหนด (จะเรียนเต็มรูปแบบใน **Part 060** ด้วย `rack-attack`) |
| **500 Internal Server Error** | `:internal_server_error` | Error ที่ไม่คาดคิดฝั่ง server (bug จริงๆ) | Exception ที่ไม่ถูก `rescue` ไว้ |

### สิ่งที่พบบ่อยว่าเขียนผิด: 200 กับทุกกรณี

รูปแบบที่พบบ่อยในโค้ดของมือใหม่ (และเป็นความผิดพลาดที่ต้องแก้ให้เป็นนิสัยตั้งแต่ต้น):

```ruby
# ผิด — ใช้ 200 (default) กับทุกกรณีแม้จะ error
def create
  post = Post.new(post_params)
  if post.save
    render json: post
  else
    render json: { errors: post.errors }   # <-- ลืมใส่ status! ได้ 200 OK ทั้งที่ล้มเหลว
  end
end
```

```ruby
# ถูก — ระบุ status ให้ตรงกับผลลัพธ์จริงเสมอ
def create
  post = Post.new(post_params)
  if post.save
    render json: PostBlueprint.render(post, view: :detail), status: :created   # 201
  else
    render json: { errors: post.errors.full_messages }, status: :unprocessable_entity   # 422
  end
end
```

Client ที่เขียนถูกต้องจะเช็ค status code ก่อนเสมอ (`if response.status == 201`) — ถ้า server ส่ง
`200 OK` กลับมาพร้อม body ที่จริงๆ แล้วเป็น error message client ฝั่งนั้นจะเข้าใจผิดว่าคำขอสำเร็จ
ทันที เป็นบั๊กที่ตรวจจับยากมากเพราะทั้งสองฝั่งดู "ทำงานได้" แต่ตีความข้อมูลผิดกัน

---

## Step 560: รูปแบบ Error Response ที่สม่ำเสมอทั้ง API ด้วย `rescue_from`

### ปัญหา: Error แต่ละจุดตอบ JSON คนละรูปแบบ

ถ้าปล่อยให้แต่ละ controller เขียน error response เอง มักจบลงด้วยรูปแบบไม่ตรงกันทั่วทั้ง API:

```ruby
# controller A
render json: { error: "not found" }, status: :not_found

# controller B
render json: { message: "record not found" }, status: :not_found

# controller C
render json: { errors: ["ไม่พบข้อมูล"] }, status: :not_found
```

Client ที่ต้องเขียนโค้ดจัดการ error จากทั้ง 3 รูปแบบนี้ต้องเขียน branching logic แยกกันสำหรับแต่ละ
endpoint — ขัดกับหลักการพื้นฐานของ API ที่ดีคือ **ต้องคาดเดารูปแบบ response ได้อย่างสม่ำเสมอ**

### ออกแบบรูปแบบ Error เดียวที่ใช้ทั้ง API

```json
{
  "error": {
    "code": "validation_failed",
    "message": "ข้อมูลไม่ผ่านการตรวจสอบ",
    "details": ["Title can't be blank", "Body can't be blank"]
  }
}
```

- `code` — string คงที่ที่ client ใช้เขียน logic แยกกรณีได้ (ไม่ควรเปลี่ยนบ่อย ต่างจาก `message`
  ที่อาจแปลภาษาได้)
- `message` — ข้อความสรุปสั้นๆ อ่านง่าย (แสดงให้ผู้ใช้ปลายทางเห็นได้ตรงๆ)
- `details` — ข้อมูลเสริม (เช่น list ของ validation error ทีละข้อ) — เป็น optional ไม่ใช่ทุก error
  จะมี field นี้

### รวม Logic การสร้าง Error Response ไว้ที่ `ApplicationController` ด้วย `rescue_from`

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :render_unprocessable
  rescue_from ActionController::ParameterMissing, with: :render_bad_request

  private

  # รูปแบบ error ที่คงที่แบบเดียวกันทั้ง API ทุก endpoint
  def render_error(status:, code:, message:, details: nil)
    body = { error: { code: code, message: message } }
    body[:error][:details] = details if details.present?
    render json: body, status: status
  end

  def render_not_found(exception)
    render_error(status: :not_found, code: "not_found", message: exception.message)
  end

  def render_unprocessable(exception)
    render_error(
      status: :unprocessable_entity,
      code: "validation_failed",
      message: "ข้อมูลไม่ผ่านการตรวจสอบ",
      details: exception.record.errors.full_messages
    )
  end

  def render_bad_request(exception)
    render_error(status: :bad_request, code: "bad_request", message: exception.message)
  end
end
```

**อธิบาย:**

- `rescue_from ExceptionClass, with: :method_name` — ทบทวนจาก **Part 023** — ทุกครั้งที่
  exception ประเภทนี้เกิดขึ้นใน controller ไหนก็ตามที่สืบทอดจาก `ApplicationController`
  (คือทุก controller ในแอป) Rails จะเรียก method ที่ระบุแทนที่จะปล่อยให้ error หลุดออกไปเป็น
  `500 Internal Server Error` แบบดิบๆ ที่ leak รายละเอียด implementation ให้ client เห็น
- `ActiveRecord::RecordNotFound` — exception มาตรฐานที่ `Model.find(id)` throw เองอัตโนมัติเมื่อ
  หา record ไม่เจอ (ทบทวนจาก **Part 025**) — **ข้อดีของการดัก exception นี้แทนการเช็ค
  `find_by` + `if nil?` เองทุกที่คือโค้ด controller สั้นลงมาก** เขียน `Post.find(params[:id])`
  ตรงๆ ได้เลยโดยไม่ต้องกัง วลเรื่อง 404 ในทุก action (ดูตัวอย่างเต็มในแบบฝึกหัดด้านล่าง)
- `ActiveRecord::RecordInvalid` — throw โดย method ที่ลงท้ายด้วย `!` เช่น `save!`/`update!`
  เมื่อ validation ไม่ผ่าน (ต่างจาก `save`/`update` ที่คืน `false` เฉยๆ ไม่ throw) — การใช้
  `save!`/`update!` คู่กับ `rescue_from` แบบนี้ทำให้ controller เขียนแบบ **happy path เดียว**
  โดยไม่ต้องมี `if/else` แยกกรณีสำเร็จ/ล้มเหลวในทุก action (pattern นี้เรียกว่า "let it crash"
  แล้วดักที่ชั้นบนสุด — สะอาดกว่าการ `if/else` ซ้ำๆ ทุก action มาก)
- `exception.record.errors.full_messages` — `RecordInvalid#record` คืน object ที่ validation
  ไม่ผ่าน ทำให้ดึง `.errors.full_messages` มาใส่ใน `details` ได้ตรงๆ (ทบทวน `errors` object จาก
  **Part 026**)

### ทดสอบทั้ง 3 กรณี Error ด้วย `curl` จริง

**404 — หา resource ไม่เจอ:**

```bash
curl -s -w "\nHTTP:%{http_code}\n" http://127.0.0.1:3098/posts/999
```

```json
{"error":{"code":"not_found","message":"Couldn't find Post with 'id'=\"999\""}}
HTTP:404
```

**422 — validation ไม่ผ่าน:**

```bash
curl -s -w "\nHTTP:%{http_code}\n" -X POST http://127.0.0.1:3098/posts \
  -H "Content-Type: application/json" \
  -d '{"post":{"title":"","body":"","user_id":1}}'
```

```json
{"error":{"code":"validation_failed","message":"ข้อมูลไม่ผ่านการตรวจสอบ","details":["Title can't be blank","Body can't be blank"]}}
HTTP:422
```

**400 — ขาด parameter ที่จำเป็น (ลืมส่ง key `post` มาทั้งหมด):**

```bash
curl -s -w "\nHTTP:%{http_code}\n" -X POST http://127.0.0.1:3098/posts \
  -H "Content-Type: application/json" \
  -d '{}'
```

```json
{"error":{"code":"bad_request","message":"param is missing or the value is empty or invalid: post"}}
HTTP:400
```

ทดสอบรันจริงทั้ง 3 กรณีแล้ว ให้ผลลัพธ์ตรงตามรูปแบบ error ที่ออกแบบไว้ทุกประการ — **ไม่ว่า error
จะเกิดจาก controller ไหนหรือ exception ประเภทไหนใน 3 แบบนี้ รูปแบบ JSON ที่ client ได้รับจะเป็น
`{"error": {"code": ..., "message": ..., "details": ...}}` เสมอ** เพราะทุก controller สืบทอด
logic การจัดการ error มาจาก `ApplicationController` จุดเดียว

---

## แบบฝึกหัด: Posts API ครบวงจร (index/show/create/update/destroy) พร้อม Serializer และ Error Response ที่สม่ำเสมอ

### โจทย์

สร้าง Rails API-only application ชื่อ `posts_api` ที่มีคุณสมบัติดังนี้:

1. Model `User` (มี `email`), `Post` (มี `title`, `body`, `published`, `belongs_to :user`,
   `has_many :comments`), `Comment` (มี `author_name`, `body`, `belongs_to :post`)
2. `PostBlueprint` ที่มี 2 view: `:list` (สำหรับ index — มี `comments_count` แต่ไม่มี comment
   เต็มรูปแบบ) และ `:detail` (สำหรับ show/create/update — มี `comments` และ `author` ฝังมาเต็ม)
3. `CommentBlueprint` แยกต่างหาก ใช้ทั้งใน `:detail` view ของ post และตอบกลับตอนสร้าง comment ใหม่
4. Endpoint ครบ: `GET /posts`, `GET /posts/:id`, `POST /posts`, `PATCH /posts/:id`,
   `DELETE /posts/:id`, และ `POST /posts/:post_id/comments`
5. Status code ถูกต้องตามตาราง Step 559 ทุก action
6. Error response รูปแบบเดียวกันทั้ง API ตาม Step 560 (404, 422, 400)
7. `.includes` ป้องกัน N+1 ใน `index` ตาม Step 558
8. ทดสอบด้วย `curl` จริงให้ครบทุก endpoint ทั้งกรณีสำเร็จและกรณี error

### เฉลย

```bash
rails new posts_api --api -d sqlite3
cd posts_api
```

```ruby
# Gemfile — เอา # ออกจากบรรทัด jbuilder ที่มีอยู่แล้ว แล้วเพิ่ม blueprinter
gem "jbuilder"
gem "blueprinter", "~> 1.1"
```

```bash
bundle install
bin/rails g model User email:string:uniq
bin/rails g model Post title:string body:text user:references published:boolean
bin/rails g model Comment post:references author_name:string body:text
bin/rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :posts

  validates :email, presence: true, uniqueness: true
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 120 }
  validates :body, presence: true
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post

  validates :author_name, presence: true
  validates :body, presence: true
end
```

```ruby
# app/blueprints/comment_blueprint.rb
class CommentBlueprint < Blueprinter::Base
  identifier :id

  fields :author_name, :body, :created_at
end
```

```ruby
# app/blueprints/post_blueprint.rb
class PostBlueprint < Blueprinter::Base
  identifier :id

  fields :title, :body, :published, :created_at

  view :list do
    field :comments_count do |post, _options|
      post.comments.size
    end
  end

  view :detail do
    association :comments, blueprint: CommentBlueprint

    field :author do |post, _options|
      { id: post.user.id, email: post.user.email }
    end
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :render_unprocessable
  rescue_from ActionController::ParameterMissing, with: :render_bad_request

  private

  def render_error(status:, code:, message:, details: nil)
    body = { error: { code: code, message: message } }
    body[:error][:details] = details if details.present?
    render json: body, status: status
  end

  def render_not_found(exception)
    render_error(status: :not_found, code: "not_found", message: exception.message)
  end

  def render_unprocessable(exception)
    render_error(
      status: :unprocessable_entity,
      code: "validation_failed",
      message: "ข้อมูลไม่ผ่านการตรวจสอบ",
      details: exception.record.errors.full_messages
    )
  end

  def render_bad_request(exception)
    render_error(status: :bad_request, code: "bad_request", message: exception.message)
  end
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :update, :destroy]

  def index
    posts = Post.includes(:comments, :user).order(created_at: :desc)
    render json: PostBlueprint.render(posts, view: :list)
  end

  def show
    render json: PostBlueprint.render(@post, view: :detail)
  end

  def create
    post = Post.new(post_params)
    post.save!
    render json: PostBlueprint.render(post, view: :detail), status: :created
  end

  def update
    @post.update!(post_params)
    render json: PostBlueprint.render(@post, view: :detail)
  end

  def destroy
    @post.destroy!
    head :no_content
  end

  private

  def set_post
    @post = Post.includes(:comments, :user).find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :published, :user_id)
  end
end
```

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def create
    post = Post.find(params[:post_id])
    comment = post.comments.new(comment_params)
    comment.save!
    render json: CommentBlueprint.render(comment), status: :created
  end

  private

  def comment_params
    params.require(:comment).permit(:author_name, :body)
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  resources :posts, only: [:index, :show, :create, :update, :destroy] do
    resources :comments, only: [:create], shallow: true
  end
end
```

สร้างข้อมูลตัวอย่างและรัน server:

```bash
bin/rails runner '
user = User.find_or_create_by!(email: "author@example.com")
post = Post.find_or_create_by!(title: "Rails API mode คืออะไร", body: "เนื้อหาเกี่ยวกับ Rails API only mode", user: user, published: true)
post.comments.find_or_create_by!(author_name: "สมชาย", body: "บทความดีมากครับ")
post.comments.find_or_create_by!(author_name: "สมหญิง", body: "รอตอนต่อไปอยู่เลย")
'
bin/rails server -p 3098
```

### ทดสอบครบทุก endpoint ด้วย `curl` (คำสั่งจริงที่รันแล้วได้ผลตามนี้)

```bash
BASE=http://127.0.0.1:3098

# 1) GET /posts — list view (สั้น มี comments_count ไม่มี comments เต็ม)
curl -s "$BASE/posts"
# => [{"id":1,"body":"...","comments_count":2,"created_at":"...","published":true,"title":"Rails API mode คืออะไร"}]

# 2) GET /posts/1 — detail view (มี comments + author ฝังมาเต็ม)
curl -s "$BASE/posts/1"
# => {"id":1,"author":{"id":1,"email":"author@example.com"},"body":"...","comments":[...2 comments...],
#     "created_at":"...","published":true,"title":"Rails API mode คืออะไร"}

# 3) GET /posts/999 — 404
curl -s -w "\nHTTP:%{http_code}\n" "$BASE/posts/999"
# => {"error":{"code":"not_found","message":"Couldn't find Post with 'id'=\"999\""}}
#    HTTP:404

# 4) POST /posts — สร้างสำเร็จ -> 201
curl -s -w "\nHTTP:%{http_code}\n" -X POST "$BASE/posts" \
  -H "Content-Type: application/json" \
  -d '{"post":{"title":"บทความใหม่","body":"เนื้อหาบทความใหม่","user_id":1,"published":false}}'
# => {"id":2,"author":{...},"body":"เนื้อหาบทความใหม่","comments":[],"created_at":"...",
#     "published":false,"title":"บทความใหม่"}
#    HTTP:201

# 5) POST /posts ข้อมูลไม่ผ่าน validation -> 422
curl -s -w "\nHTTP:%{http_code}\n" -X POST "$BASE/posts" \
  -H "Content-Type: application/json" \
  -d '{"post":{"title":"","body":"","user_id":1}}'
# => {"error":{"code":"validation_failed","message":"ข้อมูลไม่ผ่านการตรวจสอบ",
#     "details":["Title can't be blank","Body can't be blank"]}}
#    HTTP:422

# 6) POST /posts ลืมส่ง key "post" -> 400
curl -s -w "\nHTTP:%{http_code}\n" -X POST "$BASE/posts" -H "Content-Type: application/json" -d '{}'
# => {"error":{"code":"bad_request","message":"param is missing or the value is empty or invalid: post"}}
#    HTTP:400

# 7) PATCH /posts/2 — แก้ไขสำเร็จ -> 200
curl -s -w "\nHTTP:%{http_code}\n" -X PATCH "$BASE/posts/2" \
  -H "Content-Type: application/json" \
  -d '{"post":{"published":true}}'
# => {"id":2, ..., "published":true, ...}
#    HTTP:200

# 8) POST /posts/1/comments — เพิ่ม comment ใหม่ -> 201
curl -s -w "\nHTTP:%{http_code}\n" -X POST "$BASE/posts/1/comments" \
  -H "Content-Type: application/json" \
  -d '{"comment":{"author_name":"วิชัย","body":"เห็นด้วยครับ"}}'
# => {"id":3,"author_name":"วิชัย","body":"เห็นด้วยครับ","created_at":"..."}
#    HTTP:201

# 9) DELETE /posts/2 — ลบสำเร็จ -> 204 (ไม่มี body)
curl -s -w "\nHTTP:%{http_code}\n" -X DELETE "$BASE/posts/2"
# => (ไม่มี body)
#    HTTP:204

# 10) GET /posts/2 หลังลบไปแล้ว -> 404
curl -s -w "\nHTTP:%{http_code}\n" "$BASE/posts/2"
# => {"error":{"code":"not_found","message":"Couldn't find Post with 'id'=\"2\""}}
#    HTTP:404
```

ทดสอบรันจริงทั้ง 10 กรณีข้างต้นยืนยันแล้วว่าให้ผลลัพธ์ตรงตามที่คาดหวังทุกประการ บน Ruby 3.3.6 /
Rails 8.1.4 / Blueprinter 1.3.0 — **นี่คือ Posts API ที่สมบูรณ์ครบวงจร: CRUD เต็มรูปแบบ, JSON
output ที่ควบคุมรูปร่างได้แม่นยำผ่าน serializer แยกจาก model, ป้องกัน N+1 ด้วย `includes`,
status code ถูกต้องตามมาตรฐาน REST ทุก action, และ error response รูปแบบเดียวกันสม่ำเสมอทั้ง API**

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เขียน endpoint เดิมทั้งหมดใหม่ด้วย Jbuilder แทน Blueprinter** — สร้าง
   `app/views/posts/index.json.jbuilder` และ `app/views/posts/show.json.jbuilder` ที่ให้ผลลัพธ์
   JSON **เหมือนเป๊ะ** กับที่ `PostBlueprint` (view `:list`/`:detail`) ให้ไว้ (ใช้ `json.partial!`
   แยก partial สำหรับ comment ออกมาต่างหากเพื่อไม่ให้เขียนโครงสร้าง comment ซ้ำสองที่) แล้วเทียบ
   ว่าไฟล์ไหนอ่านง่ายกว่าในความเห็นของตัวเอง — นี่คือวิธีที่ดีที่สุดในการรู้สึกถึงความต่างระหว่าง
   สองแนวทางจริงๆ ด้วยตัวเอง ไม่ใช่แค่อ่านตารางเปรียบเทียบ
2. **เพิ่ม `403 Forbidden` เข้าไปในระบบ** — สมมติว่า `Post` มี column `user_id` ที่บอกว่าใครเป็น
   เจ้าของ เพิ่ม `before_action` ใน `PostsController` ที่เช็คว่า "ผู้ใช้ปัจจุบัน" (ใช้ JWT auth
   จาก **Part 045** ผูกเข้ามาด้วยก็ได้ หรือจะ hardcode `current_user` ไว้ก่อนเพื่อโฟกัสที่
   status code ก็ได้) เป็นเจ้าของโพสต์นั้นจริงก่อนจะยอมให้ `update`/`destroy` ถ้าไม่ใช่เจ้าของ
   ให้ตอบ `403` ด้วยรูปแบบ error เดียวกับ Step 560 (เพิ่ม `rescue_from` หรือ `render_error`
   เรียกตรงๆ ก็ได้)
3. **เพิ่ม field ที่คำนวณจาก options ใน Blueprint** — แก้ `PostBlueprint` ให้มี field
   `can_edit` ที่คืน `true`/`false` โดยรับ `current_user` ผ่าน options ตอนเรียก
   `PostBlueprint.render(post, view: :detail, current_user: current_user)` แล้วเทียบ
   `post.user_id == current_user&.id` ข้างใน field block — ฝึกความเข้าใจเรื่อง `options` ที่
   ส่งผ่านเข้าไปใน field block ของ Blueprinter (ที่ Step 557 แนะนำไว้แต่ยังไม่ได้ใช้จริง)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจ **`rails new --api`** อย่างละเอียด — `ActionController::API` แทน `ActionController::Base`,
  middleware stack ที่บางลง (ไม่มี session/cookie/CSRF), และเหตุผลที่เชื่อมโยงกลับไปที่หลักการ
  **stateless authentication** ของ Part 045
- เข้าใจกลไกเบื้องหลัง `render json:` — `to_json` เรียก `as_json` ก่อนเสมอ และรู้ว่า
  `as_json` ของ ActiveRecord default รวมทุก column แต่ไม่รวม association/method อื่น
- ควบคุม JSON output เบื้องต้นด้วย `only:`/`except:`/`include:` และเข้าใจว่าทำไม **การ override
  `as_json` บน model เป็น anti-pattern** — ผูก concern ผิดชั้น ใช้ได้แค่รูปร่างเดียว และเปิดช่องให้
  เกิด silent bug ได้ (เห็นจากการทดสอบจริงที่ `only:` ถูกเมินไปเฉยๆ)
- ใช้ **Jbuilder** สร้าง JSON แบบ view template เหมือน ERB พร้อม partial สำหรับ reuse โครงสร้าง
- เปรียบเทียบแนวทาง serialization ที่ใช้จริงในปัจจุบันอย่างตรงไปตรงมา: **`active_model_serializers`
  อยู่ในสถานะ maintenance mode แล้ว ไม่แนะนำสำหรับโปรเจกต์ใหม่** ในขณะที่ **Jbuilder** และ
  **Blueprinter/Alba** ยังคง active และเป็นทางเลือกที่สมเหตุสมผลทั้งคู่ ขึ้นอยู่กับสไตล์ทีม
- สร้าง **Blueprint** ด้วย Blueprinter ที่มีหลาย `view` (`:list`/`:detail`) สำหรับ `Post` ที่มี
  `Comment` ซ้อนกัน ผ่าน `association`/`field` block
- ป้องกัน **N+1 query** ตอน serialize ข้อมูลที่มี association ด้วย `includes` — พิสูจน์ด้วยการนับ
  query จริง (4 queries ไม่ใช้ includes vs 2 queries ใช้ includes) ทบทวนหลักการเต็มจาก Part 034
- จำ **ตาราง HTTP status code** สำหรับ API ได้ครบ: 200/201/204/400/401/403/404/422/429/500 และ
  ใช้ถูกต้องในทุก action
- ออกแบบ **รูปแบบ error response ที่สม่ำเสมอทั้ง API** ด้วย `rescue_from` ที่
  `ApplicationController` ครอบคลุม `RecordNotFound`/`RecordInvalid`/`ParameterMissing`
- สร้าง **Posts API ครบวงจร** (index/show/create/update/destroy + nested comments) ที่รวมทุก
  หลักการข้างต้นเข้าด้วยกัน และทดสอบ end-to-end ด้วย `curl` จริงครบทุก endpoint ทั้งกรณีสำเร็จและ
  error

**ต่อไป (Part 057):** Part นี้สร้าง Posts API เวอร์ชันแรกที่ทำงานได้ครบถ้วนแล้ว แต่ยังมีคำถามที่
ทุก API ระดับ production ต้องตอบให้ได้ — ถ้า client เก่าคาดหวังรูปร่าง JSON แบบหนึ่ง แต่ทีมต้องการ
เปลี่ยนโครงสร้าง response โดยไม่ทำ client เดิมพัง จะทำอย่างไร (**API Versioning**), มีมาตรฐานเปิด
ที่กำหนดรูปร่าง JSON ของ API ไว้ให้เป็นระบบมากขึ้นไหม (**JSON:API spec**), และถ้า `GET /posts` มี
ข้อมูลนับหมื่นนับแสนแถว จะส่งกลับมาทีเดียวหมดไม่ได้แน่ๆ ต้องแบ่งหน้าอย่างไร (**Pagination สำหรับ
API** ด้วย `Kaminari`/`Pagy` ที่เคยเห็นแวบหนึ่งใน **Part 038** สำหรับหน้าเว็บ แต่ API ต้องการ
รูปแบบ metadata ที่ต่างออกไป) — Part 057 จะตอบทั้ง 3 คำถามนี้ พร้อมยกระดับ Posts API จาก Part นี้
ให้พร้อมใช้งานจริงในระดับ production มากขึ้นไปอีกขั้น
