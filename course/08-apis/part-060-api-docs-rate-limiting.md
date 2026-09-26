# Part 060: API Documentation ด้วย rswag/OpenAPI และ Rate Limiting ด้วย rack-attack — ปิด Phase 8

> **Step ครอบคลุมใน Part นี้:** Step 591–600
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 056–059 มาก่อน โดยเฉพาะ Part 056 เรื่อง Rails API-only
> mode/serializer, Part 057 เรื่อง API versioning, และ Part 058–059 เรื่อง GraphQL)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4,
> `rswag` 2.17.0, `rack-attack` 6.8.0)

Part นี้คือ **Part สุดท้ายของ Phase 8: APIs & GraphQL** เราเดินทางมาไกลมากแล้วตลอด 4 Part ที่
ผ่านมา — สร้าง Rails API-only app พร้อม serializer (Part 056), จัดระเบียบ version และ pagination
แบบมืออาชีพ (Part 057), แล้วเปิดโลก GraphQL ทั้ง query และ mutation พร้อม authorization
(Part 058–059) ตอนนี้ API ของเราทำงานได้ครบถ้วนและปลอดภัยในระดับ business logic แล้ว แต่ยังขาด
สิ่งสำคัญ 2 อย่างที่ทุก API ระดับ production ต้องมี:

1. **เอกสารประกอบ (documentation)** — คนอื่น (หรือทีม frontend/mobile ของเราเอง) จะรู้ได้
   อย่างไรว่า endpoint ไหนรับ parameter อะไร คืนอะไรกลับมา โดยไม่ต้องขอดู source code
2. **การป้องกันการใช้งานเกินขอบเขต (rate limiting)** — ถ้าไม่มีการจำกัด ใครก็ยิง request
   รัวๆ ใส่ API ของเราได้ไม่จำกัด ไม่ว่าจะตั้งใจโจมตี (brute-force, DDoS) หรือแค่ bug ใน
   client ที่ยิงซ้ำไม่หยุด

Part นี้จะพา **สร้าง Rails API app ทดสอบจริง (`posts_api`) ติดตั้ง `rswag` เพื่อเขียน API
documentation ที่ generate จาก RSpec request spec ตัวเดียวกับที่ใช้ทดสอบ endpoint จริง แล้ว
เปิดดู Swagger UI ที่ `/api-docs` จริง จากนั้นติดตั้ง `rack-attack` ตั้งค่า throttle หลายแบบ
และยิง `curl` วนซ้ำจนเห็น `429 Too Many Requests` จริงๆ ปรากฏขึ้นมา** ทุกโค้ด ทุก response,
ทุก error ที่ปรากฏใน Part นี้คือผลลัพธ์จริงจากการรันคำสั่งเหล่านั้นบนเครื่องจริง ไม่ใช่โค้ด
ที่เขียนขึ้นลอยๆ — รวมถึง **ปัญหาจริงที่เจอระหว่างทาง** (gem เวอร์ชันชนกัน, การตั้งชื่อ
parameter ชนกับ HTTP verb, rack-attack ไปบล็อก test suite ของตัวเอง) ซึ่งล้วนเป็นบทเรียน
ที่มีค่าไม่แพ้โค้ดที่รันผ่านตั้งแต่ครั้งแรก

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยการสร้างแอป Rails API-only
> ชื่อ `posts_api` ใน `/tmp` (แยกจาก repository ของหลักสูตรโดยสิ้นเชิง) ติดตั้ง gem `rswag-api`,
> `rswag-ui`, `rswag-specs`, และ `rack-attack` จริง รัน `bin/rails server` แล้วยิง `curl`
> วนซ้ำจนเห็น `429` จริง รวมถึงรัน RSpec suite เต็มรูปแบบให้ผ่านทุกตัวก่อนนำผลลัพธ์มาเขียนใน
> เอกสารนี้

## สารบัญของ Part นี้

- Step 591: ทำไม API ต้องมีเอกสารประกอบ — API คือสัญญา (contract) กับผู้บริโภคที่อ่าน source
  code ของเราไม่ได้
- Step 592: OpenAPI/Swagger คืออะไร และติดตั้ง `rswag` (`rswag-api`, `rswag-ui`, `rswag-specs`)
- Step 593: เขียน API docs ในรูปแบบ RSpec request spec ด้วย DSL ของ `rswag` — `path`,
  `response`, `parameter`
- Step 594: เอกสารสำหรับ POST/PATCH/DELETE — request body schema, หลาย response รวมถึง
  error response
- Step 595: generate สเปกจริงด้วย `rswag:specs:swaggerize` และเปิดดู Swagger UI ที่ `/api-docs`
- Step 596: ทางเลือกอื่นที่ควรรู้จัก — เขียน OpenAPI YAML ด้วยมือ vs GraphQL self-documenting
  schema/introspection (เทียบกับ Part 058–059)
- Step 597: ทำไมต้อง rate limit — ทบทวน `rate_limit` ของ Rails 8 (Part 041) เทียบกับ
  `rack-attack`
- Step 598: ติดตั้งและตั้งค่า `rack-attack` เบื้องต้น — throttle by IP, safelist, blocklist
- Step 599: Throttle strategy ขั้นสูง — by user/API key, by endpoint, custom `429` response
  พร้อม `Retry-After`
- Step 600: ทดสอบ rate limiting ด้วย `curl` loop จริง + RSpec + แบบฝึกหัดปิด Phase 8

---

## Step 591: ทำไม API ต้องมีเอกสารประกอบ — API คือสัญญา (contract) กับผู้บริโภคที่อ่าน source code ของเราไม่ได้

### ปัญหาที่เอกสารประกอบแก้

ตลอด Part 056–059 เราสร้าง endpoint แบบ REST และ GraphQL ที่ทำงานถูกต้องสมบูรณ์ — เรารู้ดี
เพราะเราคือคนเขียนมันเอง เราเปิด `app/controllers/api/v1/posts_controller.rb` ดูก็รู้ทันทีว่า
`POST /api/v1/posts` ต้องส่ง `post[title]` และ `post[body]` มา แต่ปัญหาคือ **ไม่ใช่ทุกคนที่จะ
เข้าถึง source code ของเราได้**:

- ทีม **frontend/mobile** ที่เรียก API ของเราอาจอยู่คนละทีม คนละบริษัท หรือแม้แต่คนละ
  organization (เช่นเปิด API ให้ partner ภายนอกใช้)
- แม้แต่ **ตัวเราเองในอีก 6 เดือนข้างหน้า** ก็อาจจำรายละเอียดของ endpoint ที่ตัวเองเขียนไม่ได้
  แล้ว (โดยเฉพาะเมื่อโปรเจกต์มี endpoint นับร้อยตัว)
- Endpoint ที่ต้องไล่อ่าน controller + serializer + model validation หลายไฟล์กว่าจะรู้ว่า
  "ต้องส่ง field อะไรบ้าง" คือประสบการณ์ที่แย่มากสำหรับคนที่แค่อยากจะ "เรียกใช้" API เฉยๆ
  ไม่ได้อยากมานั่งอ่าน implementation

พูดให้ชัดที่สุด: **API คือสัญญา (contract)** ระหว่างฝั่งที่ให้บริการ (เรา) กับฝั่งที่บริโภค
(consumer) — เหมือนสัญญาทางกฎหมายที่ทั้งสองฝ่ายต้องอ่านเข้าใจตรงกันโดยไม่ต้องรู้ว่าอีกฝ่าย
ทำงานภายในอย่างไร เอกสารประกอบ (API documentation) คือตัวสัญญานั้นในรูปแบบที่มนุษย์และเครื่อง
อ่านเข้าใจได้ทั้งคู่

### ปัญหาคลาสสิกของเอกสารที่เขียนแยกจากโค้ด

วิธีทำเอกสาร API แบบดั้งเดิมที่สุดคือเขียนเป็นหน้า wiki หรือ Google Doc แยกต่างหากจากโค้ด —
ปัญหาคือ **เอกสารแบบนี้ "หลุด" (drift) จากโค้ดจริงแทบจะทันทีที่มีคนแก้ endpoint แล้วลืม
อัปเดตเอกสาร** (ซึ่งเกิดขึ้นเสมอในทีมจริง เพราะการอัปเดตเอกสารไม่ได้ผูกกับขั้นตอนการ deploy
หรือ code review เหมือนโค้ด) ผลคือเอกสารที่ "ดูน่าเชื่อถือ" แต่บอกข้อมูลผิดๆ ให้ consumer
ซึ่งแย่กว่าไม่มีเอกสารเลยด้วยซ้ำ (เพราะคนหลงเชื่อไปเสียเวลา debug กับข้อมูลที่ผิด)

Part นี้จะแก้ปัญหานี้ด้วยแนวคิดที่ Step 592–595 จะแนะนำ: **เขียนเอกสารเป็นส่วนหนึ่งของ test
suite เอง** — ถ้าเอกสารพูดไม่ตรงกับพฤติกรรมจริงของ endpoint, **test จะแดง (fail) ทันที**
บังคับให้ต้องแก้เอกสารพร้อมกับแก้โค้ดเสมอ ไม่มีทางที่เอกสารจะ drift ออกจากความจริงได้

> **preview:** Step 596 จะเทียบให้เห็นว่า GraphQL (Part 058–059) แก้ปัญหานี้ด้วยวิธีที่ต่างกัน
> โดยสิ้นเชิง — ไม่ใช่ด้วยวินัยการเขียน test แต่ด้วย**โครงสร้างของภาษาเองที่ทำให้ schema
> กับ implementation เป็นสิ่งเดียวกันเสมอ**

---

## Step 592: OpenAPI/Swagger คืออะไร และติดตั้ง `rswag` (`rswag-api`, `rswag-ui`, `rswag-specs`)

### OpenAPI Specification คืออะไร

**OpenAPI Specification** (เดิมชื่อ **Swagger Specification** ก่อนจะบริจาคให้ Linux
Foundation ดูแลในปี 2015 — ปัจจุบันคำว่า "Swagger" มักหมายถึงเครื่องมือ/ระบบนิเวศรอบๆ
OpenAPI มากกว่าตัวสเปกเอง) คือ **รูปแบบมาตรฐาน (แบบ YAML หรือ JSON) สำหรับอธิบาย REST API**
ที่ทั้งมนุษย์และเครื่องอ่านเข้าใจได้ ระบุครบทุกอย่างที่ consumer ต้องรู้:

- มี endpoint อะไรบ้าง (`paths`)
- แต่ละ endpoint รับ HTTP method อะไร (`get`, `post`, `patch`, `delete`)
- รับ parameter อะไรบ้าง (query string, path parameter, request body) พร้อมชนิดข้อมูล
- คืน response อะไรกลับมาในแต่ละ HTTP status code พร้อม schema ของ JSON ที่คืนกลับ

เพราะเป็น**รูปแบบมาตรฐาน**ที่อุตสาหกรรมทั้งหมดยอมรับร่วมกัน (ไม่ใช่แค่ Ruby/Rails) จึงมี
เครื่องมือจำนวนมหาศาลที่ "อ่าน" ไฟล์ OpenAPI แล้วสร้างสิ่งต่างๆ ให้อัตโนมัติ เช่น:

- **Swagger UI** — หน้าเว็บ interactive ที่แสดงเอกสารสวยงาม พร้อมปุ่ม "Try it out" ให้ยิง
  request ทดสอบได้จริงจากหน้าเว็บเลย (จะเห็นจริงใน Step 595)
- Client SDK ในภาษาต่างๆ ที่ generate จากสเปกได้อัตโนมัติ (เช่น TypeScript client, Swift
  client สำหรับ mobile app)
- เครื่องมือทดสอบ API แบบ contract testing

### `rswag` คืออะไร

`rswag` คือ gem ที่เชื่อม **RSpec request spec** เข้ากับ **OpenAPI specification** โดยตรง
แนวคิดหลักของมันคือสิ่งที่ Step 591 ทิ้งท้ายไว้: **เขียน DSL พิเศษที่ทำหน้าที่สองอย่างพร้อมกัน
ในไฟล์เดียว — (1) เป็น request spec ที่ทดสอบ endpoint จริง และ (2) เป็นต้นทางที่ generate
ไฟล์ OpenAPI YAML/JSON ออกมา** เพราะมาจากไฟล์เดียวกัน สเปกที่ generate ออกมาจึง **ไม่มีทาง
พูดไม่ตรงกับพฤติกรรมจริงของ endpoint ได้เลย** — ถ้าพูดไม่ตรง แปลว่า test ตัวนั้นแดงไปแล้ว

`rswag` แบ่งเป็น 3 gem ย่อยที่ทำงานร่วมกัน:

| Gem | หน้าที่ |
|---|---|
| **`rswag-specs`** | DSL (`path`, `response`, `parameter`, `run_test!`) สำหรับเขียน request spec ที่ generate สเปกได้ + Rake task `rswag:specs:swaggerize` |
| **`rswag-api`** | Rails engine ที่ mount ไว้เพื่อ**เสิร์ฟไฟล์ OpenAPI ที่ generate แล้ว**ผ่าน HTTP (เช่น `GET /api-docs/v1/swagger.yaml`) |
| **`rswag-ui`** | Rails engine ที่ mount หน้า **Swagger UI** (หน้าเว็บ interactive) ไว้ที่ path หนึ่ง (ปกติคือ `/api-docs`) โดยดึงสเปกจาก `rswag-api` มาแสดงผล |

### สร้างแอปทดสอบและติดตั้ง

สร้าง Rails API-only app ใหม่สำหรับทดลอง Part นี้ (แยกจากแอปใน Part 056–059 เพื่อโฟกัสเฉพาะ
เรื่อง docs/rate limiting ล้วนๆ — แนวคิดทั้งหมดย้ายไปใช้กับแอปจริงของ Part ก่อนหน้าได้ทันที):

```bash
rails new posts_api --api -d sqlite3
cd posts_api
```

เพิ่ม gem ที่ต้องใช้ใน `Gemfile`:

```ruby
# Gemfile
gem "rack-attack"

group :development, :test do
  gem "rspec-rails"
  gem "rswag-api"
  gem "rswag-ui"
  gem "rswag-specs"
end
```

```bash
bundle install
bin/rails generate rspec:install
```

รัน generator ติดตั้งของ `rswag` ทั้ง 3 ตัว:

```bash
bin/rails generate rswag:api:install
bin/rails generate rswag:ui:install
bin/rails generate rswag:specs:install
```

ผลลัพธ์จริง:

```
      create  config/initializers/rswag_api.rb
       route  mount Rswag::Api::Engine => '/api-docs'

      create  config/initializers/rswag_ui.rb
       route  mount Rswag::Ui::Engine => '/api-docs'

      create  spec/swagger_helper.rb
```

3 คำสั่งนี้ทำ 3 อย่าง: (1) เพิ่ม route ที่ `/api-docs` สำหรับเสิร์ฟไฟล์สเปก, (2) เพิ่ม route
ที่ `/api-docs` เดียวกันสำหรับหน้า Swagger UI (ทั้งสอง engine mount ที่ path เดียวกันได้เพราะ
คนละ sub-path ภายใน), และ (3) สร้าง `spec/swagger_helper.rb` — ไฟล์ config หลักที่กำหนดว่า
สเปกจะ generate ออกมาที่ไหนและมีข้อมูล metadata อะไรบ้าง

### `spec/swagger_helper.rb` — ตั้งค่า metadata ของสเปก

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.configure do |config|
  # โฟลเดอร์ที่ไฟล์ OpenAPI ที่ generate แล้วจะถูกเขียนลงไป
  config.openapi_root = Rails.root.join("swagger").to_s

  config.openapi_specs = {
    "v1/swagger.yaml" => {
      openapi: "3.0.1",
      info: {
        title: "Posts API",
        version: "v1"
      },
      paths: {},
      servers: [
        {
          url: "http://{defaultHost}",
          variables: {
            defaultHost: {
              default: "localhost:3000"
            }
          }
        }
      ]
    }
  }

  # รูปแบบไฟล์ที่ generate ออกมา — :yaml (อ่านง่ายกว่า) หรือ :json ก็ได้
  config.openapi_format = :yaml
end
```

**อธิบาย:**

- key `"v1/swagger.yaml"` คือชื่อไฟล์ปลายทางที่จะถูกเขียนลงใต้ `config.openapi_root` — ถ้า
  ทำ API หลาย version (ทบทวน Part 057) ก็เพิ่ม key อีกตัวเช่น `"v2/swagger.yaml"` แล้วระบุ
  ว่า spec ไฟล์ไหนสร้างเอกสารให้ version ไหนด้วย tag `openapi_spec:` (จะเห็นตัวอย่างสั้นๆ
  ใน Step 594)
- `info`, `servers` คือ metadata มาตรฐานของ OpenAPI — `servers` บอกว่า "ถ้าจะยิง request
  จริงตามเอกสารนี้ ให้ยิงไปที่ host ไหน" (Swagger UI ใช้ค่านี้ตอนกดปุ่ม "Try it out")
- `paths: {}` เริ่มต้นเป็น Hash ว่าง — เดี๋ยว Step 593–594 จะเห็นว่า `rswag-specs` เติมส่วนนี้
  ให้อัตโนมัติจาก DSL ที่เขียนใน request spec แต่ละไฟล์ ไม่ต้องเขียนมือเลย

---

## Step 593: เขียน API docs ในรูปแบบ RSpec request spec ด้วย DSL ของ `rswag` — `path`, `response`, `parameter`

### เตรียม Model และ Controller สำหรับสาธิต

สร้าง `Post` model และ `Api::V1::PostsController` แบบเดียวกับที่เรียนใน Part 056–057
(สมมติว่าเราต่อยอดจากโครงสร้าง `namespace :api do namespace :v1 do ... end end` ที่ Part 057
วางไว้แล้ว):

```bash
bin/rails generate model Post title:string body:text published:boolean
bin/rails db:create db:migrate
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
  validates :body, presence: true
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts
    end
  end
end
```

```ruby
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < ApplicationController
      before_action :set_post, only: %i[show update destroy]

      def index
        render json: Post.all.order(created_at: :desc)
      end

      def show
        render json: @post
      end

      def create
        post = Post.new(post_params)
        if post.save
          render json: post, status: :created
        else
          render json: { errors: post.errors.full_messages }, status: :unprocessable_entity
        end
      end

      def update
        if @post.update(post_params)
          render json: @post
        else
          render json: { errors: @post.errors.full_messages }, status: :unprocessable_entity
        end
      end

      def destroy
        @post.destroy
        head :no_content
      end

      private

      def set_post
        @post = Post.find(params[:id])
      rescue ActiveRecord::RecordNotFound
        render json: { error: "Post not found" }, status: :not_found
      end

      def post_params
        params.require(:post).permit(:title, :body, :published)
      end
    end
  end
end
```

โค้ดข้างบนไม่มีอะไรใหม่เลย — ทั้งหมดคือสิ่งที่เรียนไปแล้วใน Part 056 (serializer เบื้องต้น
ผ่าน `render json:`) และ Part 023/026 (strong parameters, validation) จุดเริ่มต้นใหม่จริงๆ
ของ Part นี้อยู่ที่ไฟล์ spec ต่อไปนี้

### เขียน request spec แบบ `rswag` ตัวแรก — `GET /api/v1/posts`

```ruby
# spec/requests/api/v1/posts_spec.rb
require "swagger_helper"

RSpec.describe "api/v1/posts", type: :request do
  path "/api/v1/posts" do
    get "รายการโพสต์ทั้งหมด" do
      tags "Posts"
      produces "application/json"

      response "200", "รายการโพสต์" do
        schema type: :array,
               items: {
                 type: :object,
                 properties: {
                   id: { type: :integer },
                   title: { type: :string },
                   body: { type: :string },
                   published: { type: :boolean },
                   created_at: { type: :string },
                   updated_at: { type: :string }
                 }
               }

        run_test!
      end
    end
  end
end
```

รันดู:

```bash
bundle exec rspec spec/requests/api/v1/posts_spec.rb --format documentation
```

ผลลัพธ์จริง:

```
api/v1/posts
  /api/v1/posts
    get
      รายการโพสต์
        returns a 200 response

Finished in 0.05 seconds
1 example, 0 failures
```

**อธิบายทีละบรรทัดของ DSL — นี่คือหัวใจของ Part นี้:**

- **`require "swagger_helper"`** (ไม่ใช่ `"rails_helper"` ตรงๆ) — `swagger_helper.rb` ที่
  สร้างใน Step 592 มี `require "rails_helper"` อยู่ในตัวอยู่แล้ว จึงได้ทุกอย่างของ RSpec/Rails
  ปกติมาด้วย บวกกับ config พิเศษของ `rswag`
- **`path "/api/v1/posts" do ... end`** — ประกาศว่ากำลังจะอธิบาย endpoint ที่ path นี้ —
  เทียบเท่ากับ key ใน `paths:` ของไฟล์ OpenAPI ที่จะ generate ออกมา (สังเกตว่า path เขียน
  แบบตรงตัว ไม่ใช่ route helper แบบ `api_v1_posts_path` — เพราะ OpenAPI spec ต้องการ string
  literal ของ URL pattern)
- **`get "รายการโพสต์ทั้งหมด" do ... end`** — ประกาศว่า path นี้รองรับ HTTP method `GET`
  string ที่ตามมา ("รายการโพสต์ทั้งหมด") จะกลายเป็น `summary` ในสเปก — **เขียนเป็นภาษาไทย
  ได้ตามปกติ** เพราะ OpenAPI เป็นแค่ text format ไม่ได้จำกัดภาษา (Swagger UI จะแสดงข้อความ
  ไทยได้ถูกต้องเป๊ะๆ ตามที่จะเห็นใน Step 595)
- **`tags "Posts"`** — จัดกลุ่ม endpoint นี้ไว้ใต้หมวด "Posts" ใน Swagger UI (มีประโยชน์มาก
  เมื่อ API มีหลายสิบ endpoint — Swagger UI จะพับ/กางแต่ละหมวดแยกกันได้)
- **`produces "application/json"`** — บอกว่า endpoint นี้คืนค่ากลับมาเป็น content type
  อะไร
- **`response "200", "รายการโพสต์" do ... end`** — อธิบายว่าเมื่อได้ HTTP status `200` แปลว่า
  อะไร (ข้อความอธิบายในเอกสาร) — **1 endpoint มีได้หลาย `response` block** (จะเห็นเต็มๆ ใน
  Step 594 ตอนอธิบาย error response)
- **`schema type: :array, items: { ... }`** — นิยาม **JSON Schema** ของ response body ที่
  คาดหวัง — นี่คือจุดที่ `rswag` ฉลาดที่สุด: มันไม่ได้แค่ "บันทึกไว้เฉยๆ" แต่**ตรวจสอบจริง
  ว่า response body ที่ endpoint คืนกลับมาตรงกับ schema นี้หรือไม่** ถ้าไม่ตรง (เช่น
  controller ลืมคืน field `published` มา) **test จะแดงทันที** — นี่คือกลไกที่ทำให้เอกสาร
  ไม่มีวัน drift จากความจริงตามที่สัญญาไว้ใน Step 591
- **`run_test!`** — คำสั่งพิเศษที่ `rswag-specs` เพิ่มเข้ามา ทำหน้าที่ **3 อย่างพร้อมกัน**:
  (1) ประกอบ request จริงตาม `parameter`/`path` ที่ประกาศไว้ทั้งหมดแล้วยิงเข้า endpoint จริง
  ผ่านกลไกเดียวกับ request spec ปกติ, (2) ตรวจสอบว่า HTTP status code ที่ได้ตรงกับที่ระบุใน
  `response "200", ...` หรือไม่ (ถ้าได้ `422` แทน จะ fail ทันที), (3) ตรวจสอบ response body
  กับ `schema` ที่ประกาศไว้ (ถ้ามี) แล้วบันทึกผลลัพธ์ไว้เป็นข้อมูลสำหรับ generate สเปก

> **ข้อควรระวังเรื่อง `example`:** ถ้าไม่ระบุ `schema` เลย `rswag` จะยังคงบันทึก **response
> body จริง** ที่ได้จากการรัน test ไว้เป็นตัวอย่าง (`example`) ในสเปกให้อัตโนมัติ ไม่ใช่ว่า
> ไม่มี `schema` แล้วจะไม่มีข้อมูลอะไรเลย — เพียงแต่จะไม่มีการ**ตรวจสอบ (validate)** ว่า
> response ตรงตามโครงสร้างที่คาดไว้หรือไม่เท่านั้น ทางปฏิบัติที่ดีคือใส่ `schema` ให้ครบ
> ทุก response ที่สำคัญ เพื่อให้ได้ประโยชน์เต็มที่จากกลไกป้องกัน drift

---

## Step 594: เอกสารสำหรับ POST/PATCH/DELETE — request body schema, หลาย response รวมถึง error response

### `parameter ... in: :body` — อธิบาย request body

```ruby
# spec/requests/api/v1/posts_spec.rb (ต่อจาก Step 593 ภายใน path "/api/v1/posts" เดิม)

    post "สร้างโพสต์ใหม่" do
      tags "Posts"
      consumes "application/json"
      produces "application/json"
      parameter name: :post_params, in: :body, schema: {
        type: :object,
        properties: {
          post: {
            type: :object,
            properties: {
              title: { type: :string },
              body: { type: :string },
              published: { type: :boolean }
            },
            required: %w[title body]
          }
        }
      }

      response "201", "สร้างสำเร็จ" do
        let(:post_params) { { post: { title: "Hello", body: "World", published: true } } }
        run_test!
      end

      response "422", "ข้อมูลไม่ถูกต้อง" do
        let(:post_params) { { post: { title: "", body: "" } } }
        run_test!
      end
    end
```

**อธิบาย:**

- **`consumes "application/json"`** คู่กับ `produces` — บอกว่า endpoint นี้**รับ**ข้อมูลเข้า
  เป็น content type อะไรด้วย (ใช้คู่กับ `POST`/`PATCH` ที่มี body เสมอ)
- **`parameter name: :post_params, in: :body, schema: { ... }`** — ประกาศ parameter ตัวหนึ่ง
  ชื่อ `:post_params` ที่มาจาก **request body** (`in: :body`) พร้อม schema ของมัน — ชื่อที่
  ตั้งไว้ (`:post_params`) จะกลายเป็นชื่อตัวแปรที่ต้องกำหนดค่าด้วย `let(...)` ภายในแต่ละ
  `response` block ด้านล่าง (`rswag` เอาค่านั้นไปประกอบเป็น request body จริงตอนยิง
  `run_test!`)

> **ข้อควรระวังสำคัญมาก — ห้ามตั้งชื่อ parameter ชนกับ HTTP verb method:** ตอนพัฒนา Part นี้
> เคยตั้งชื่อ parameter ตัวนี้ว่า `:post` ตรงๆ (ดูเป็นชื่อธรรมชาติที่สุดเพราะ resource คือ
> `Post`) แต่พอรัน test จริงกลับได้ error `ArgumentError: wrong number of arguments (given 2,
> expected 0)` ที่ลึกลับมาก — สาเหตุคือ **`post` เป็นชื่อ method ที่ RSpec request spec มีอยู่
> แล้วในตัว** (ใช้สำหรับยิง HTTP POST request เช่น `post "/api/v1/posts", params: {...}`)
> การเขียน `let(:post) { ... }` ไป**บดบัง (shadow)** method นั้นทั้งหมด ทำให้ `run_test!`
> ภายในซึ่งเรียก `post(...)` เพื่อยิง request จริง กลับไปเรียกโดน memoized helper `post` ที่
> เราสร้างขึ้นแทน — **กฎปฏิบัติ:** อย่าตั้งชื่อ parameter/let ว่า `get`, `post`, `put`,
> `patch`, `delete`, `head` ตรงๆ เด็ดขาดในไฟล์ request spec ใดๆ (ไม่ใช่แค่ของ `rswag`) ใช้ชื่อ
> ที่ต่อท้ายให้ชัดเจนแทน เช่น `:post_params`, `:post_body`, `:new_post` เสมอ

- แต่ละ `response` block มี `let(:post_params) { ... }` **ของตัวเอง** — เพราะ RSpec `let`
  ผูกกับ scope ของ `describe`/`context` (ในที่นี้คือ `response` block ที่ `rswag` แปลงเป็น
  example group ภายใน) แต่ละกรณีจึงส่ง body ต่างกันได้ตามที่ต้องการทดสอบ (กรณีสำเร็จ vs
  กรณี validation ล้มเหลว)
- **`response "422", "ข้อมูลไม่ถูกต้อง"`** คือตัวอย่างของการ**เอกสาร error response**
  ที่โจทย์ของ Part นี้ต้องการ — ไม่ใช่แค่ documented แค่กรณี "happy path" (สำเร็จ) แต่ยัง
  บอก consumer ล่วงหน้าด้วยว่า "ถ้าส่งข้อมูลผิด จะได้ 422 กลับมา" — สำคัญมากสำหรับทีมที่ต้อง
  เขียนโค้ด handle error ฝั่ง client ให้ถูกต้อง

### เอกสารสำหรับ path parameter (`{id}`) — `GET`, `PATCH`, `DELETE`

```ruby
  path "/api/v1/posts/{id}" do
    let(:existing_post) { Post.create!(title: "Existing", body: "Body text") }

    get "แสดงโพสต์เดียว" do
      tags "Posts"
      produces "application/json"
      parameter name: :id, in: :path, type: :integer

      response "200", "พบโพสต์" do
        let(:id) { existing_post.id }
        run_test!
      end

      response "404", "ไม่พบโพสต์" do
        let(:id) { 999_999 }
        run_test!
      end
    end

    patch "แก้ไขโพสต์" do
      tags "Posts"
      consumes "application/json"
      parameter name: :id, in: :path, type: :integer
      parameter name: :post_params, in: :body, schema: {
        type: :object,
        properties: {
          post: {
            type: :object,
            properties: { title: { type: :string } }
          }
        }
      }

      response "200", "แก้ไขสำเร็จ" do
        let(:id) { existing_post.id }
        let(:post_params) { { post: { title: "Updated title" } } }
        run_test!
      end
    end

    delete "ลบโพสต์" do
      tags "Posts"
      parameter name: :id, in: :path, type: :integer

      response "204", "ลบสำเร็จ" do
        let(:id) { existing_post.id }
        run_test!
      end
    end
  end
end
```

**อธิบาย:**

- **`parameter name: :id, in: :path, type: :integer`** — ประกาศว่า `{id}` ใน path เป็น
  parameter ชนิด `:path` (ฝังอยู่ใน URL เอง ไม่ใช่ query string หรือ body) เหมือนกับ `:body`
  ต้องมี `let(:id) { ... }` กำหนดค่าจริงในแต่ละ `response` เพื่อให้ `run_test!` ประกอบ URL
  ที่ถูกต้องได้ (`/api/v1/posts/5` แทนที่ `/api/v1/posts/{id}`)
- **`let(:existing_post) { Post.create!(...) }`** ที่ประกาศไว้ระดับ `path` block (ไม่ใช่
  ระดับ `response`) จะถูกแชร์ใช้ร่วมกันได้ทุก `response` ภายใน path นั้น — สร้าง record จริง
  ในฐานข้อมูล test แล้วเอา `id` ของมันมาใช้ต่อ เป็นรูปแบบเดียวกับที่เรียนมาตั้งแต่ Part 046
  (RSpec สำหรับ Rails)
- **`response "204", "ลบสำเร็จ"`** ไม่มี `schema` เพราะ HTTP `204 No Content` ไม่มี response
  body ให้ตรวจสอบ — `run_test!` จะยังตรวจสอบแค่ status code ให้ตรงกับ `204` เท่านั้น

รันไฟล์เต็มทั้งหมด:

```bash
bundle exec rspec spec/requests/api/v1/posts_spec.rb --format documentation
```

ผลลัพธ์จริง (หลังแก้ปัญหาชื่อ parameter ชนกับ HTTP verb ตามที่อธิบายไว้ข้างต้นแล้ว):

```
api/v1/posts
  /api/v1/posts
    get
      รายการโพสต์
        returns a 200 response
    post
      สร้างสำเร็จ
        returns a 201 response
      ข้อมูลไม่ถูกต้อง
        returns a 422 response
  /api/v1/posts/{id}
    get
      พบโพสต์
        returns a 200 response
      ไม่พบโพสต์
        returns a 404 response
    patch
      แก้ไขสำเร็จ
        returns a 200 response
    delete
      ลบสำเร็จ
        returns a 204 response

Finished in 0.15 seconds
7 examples, 0 failures
```

**7 examples ผ่านหมด** — ไฟล์เดียวนี้ทั้งทดสอบ endpoint ทุกเส้นทาง (happy path + error case)
**และ**เป็นต้นทางของเอกสาร OpenAPI ที่กำลังจะ generate ใน Step 595

---

## Step 595: generate สเปกจริงด้วย `rswag:specs:swaggerize` และเปิดดู Swagger UI ที่ `/api-docs`

### รัน Rake task เพื่อ generate ไฟล์ OpenAPI

```bash
bundle exec rake rswag:specs:swaggerize
```

ผลลัพธ์จริง:

```
Generating Swagger docs ...
Swagger doc generated at /path/to/posts_api/swagger/v1/swagger.yaml

Finished in 0.05 seconds
7 examples, 0 failures
```

สังเกตว่า **task นี้รัน RSpec ทั้งไฟล์ (`--dry-run` ภายใน) แล้ว generate ไฟล์ YAML ออกมาใน
คำสั่งเดียว** — ไม่ใช่ 2 ขั้นตอนแยกกัน (รัน test แล้วค่อย generate เอกสาร) ยิ่งตอกย้ำว่า
"เอกสาร" กับ "การทดสอบ" ในระบบนี้เป็นสิ่งเดียวกันจริงๆ ไม่ใช่แค่คำพูดสวยหรู

### เปิดดูไฟล์ที่ generate ออกมา

```yaml
# swagger/v1/swagger.yaml (ตัดมาบางส่วน)
---
openapi: 3.0.1
info:
  title: Posts API
  version: v1
paths:
  "/api/v1/posts":
    get:
      summary: รายการโพสต์ทั้งหมด
      tags:
      - Posts
      responses:
        '200':
          description: รายการโพสต์
          content:
            application/json:
              schema:
                type: array
                items:
                  type: object
                  properties:
                    id:
                      type: integer
                    title:
                      type: string
                    body:
                      type: string
                    published:
                      type: boolean
                    created_at:
                      type: string
                    updated_at:
                      type: string
    post:
      summary: สร้างโพสต์ใหม่
      tags:
      - Posts
      responses:
        '201':
          description: สร้างสำเร็จ
        '422':
          description: ข้อมูลไม่ถูกต้อง
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                post:
                  type: object
                  properties:
                    title:
                      type: string
                    body:
                      type: string
                    published:
                      type: boolean
                  required:
                  - title
                  - body
servers:
- url: http://{defaultHost}
  variables:
    defaultHost:
      default: localhost:3000
```

นี่คือไฟล์ OpenAPI 3.0.1 มาตรฐานสมบูรณ์ — สังเกตว่า `summary` และ `description` เป็น
**ภาษาไทยที่เขียนไว้ใน request spec เป๊ะๆ** ทุกตัวอักษร (`"รายการโพสต์ทั้งหมด"`,
`"สร้างสำเร็จ"`) ยืนยันว่า generate มาจากไฟล์ spec ที่เขียนใน Step 593–594 จริง ไม่ใช่ไฟล์
ที่พิมพ์แยกไว้ต่างหาก

### เปิด Swagger UI จริง

บูต server แล้วเปิดเบราว์เซอร์:

```bash
bin/rails server
```

```
=> Booting Puma
=> Rails 8.1.4 application starting in development
```

เปิด `http://localhost:3000/api-docs` — ทดสอบจริงด้วย `curl`:

```bash
curl -i http://localhost:3000/api-docs
```

```
HTTP/1.1 301 Moved Permanently
location: /api-docs/index.html
```

`301` redirect ไปที่ `/api-docs/index.html` เป็นเรื่องปกติ (Rswag::Ui::Engine ตั้ง root
route ให้ redirect ไปหน้า `index.html` ของมันเอง) ตามลิงก์ต่อไป:

```bash
curl -s -L http://localhost:3000/api-docs | grep -i "swagger-ui\|title"
```

```
<title>Swagger UI</title>
<link rel="stylesheet" type="text/css" href="./swagger-ui.css" >
<div id="swagger-ui"></div>
<script src="./swagger-ui-bundle.js"> </script>
```

หน้า Swagger UI จริงทำงานอยู่ — เปิดในเบราว์เซอร์จะเห็นหน้าเว็บ interactive ที่มี:

- หมวด **"Posts"** (จาก `tags "Posts"` ที่ประกาศไว้) พับ/กางได้
- แต่ละ endpoint แสดง HTTP method (สี badge ต่างกันตาม method: เขียวสำหรับ GET, น้ำเงินสำหรับ
  POST, เหลืองสำหรับ PATCH, แดงสำหรับ DELETE) พร้อม summary ภาษาไทย
- คลิกเข้าไปแต่ละ endpoint เห็น parameter, request body schema, response ที่เป็นไปได้ทุก
  status code พร้อมคำอธิบาย
- **ปุ่ม "Try it out"** — กดแล้วกรอกค่า parameter ทดสอบยิง request จริงไปที่ server ได้จาก
  หน้าเว็บนี้เลย โดยไม่ต้องเปิด `curl`/Postman แยก

### `rswag-api` เสิร์ฟไฟล์สเปกอย่างไร

```bash
curl -s http://localhost:3000/api-docs/v1/swagger.yaml | head -5
```

```
---
openapi: 3.0.1
info:
  title: Posts API
  version: v1
```

`rswag-api` engine อ่านไฟล์จาก `config.openapi_root` (ที่ตั้งไว้ใน `swagger_helper.rb`)
แล้วเสิร์ฟตรงๆ ผ่าน HTTP endpoint นี้ — นี่คือ URL ที่ Swagger UI เรียกไปดึงสเปกมาแสดงผล
เช่นกัน (ตั้งค่าได้ใน `config/initializers/rswag_ui.rb` ว่าจะให้ UI ดึงจาก URL ไหน)

> **จุดสำคัญเรื่อง CI/CD (preview Part 075):** โปรเจกต์จริงควรรัน
> `bundle exec rake rswag:specs:swaggerize` เป็นส่วนหนึ่งของ CI pipeline **ทุกครั้งที่ merge
> โค้ด** แล้ว commit ไฟล์ `swagger/v1/swagger.yaml` ที่ generate ใหม่กลับเข้า repository (หรือ
> deploy ขึ้น hosting แยกสำหรับเอกสาร) เพื่อการันตีว่าเอกสารที่ consumer เห็นคือเวอร์ชัน
> ล่าสุดเสมอ ไม่ใช่แค่รันมือครั้งเดียวตอน develop

---

## Step 596: ทางเลือกอื่นที่ควรรู้จัก — เขียน OpenAPI YAML ด้วยมือ vs GraphQL self-documenting schema/introspection

### ทางเลือกที่ 1: เขียนไฟล์ OpenAPI YAML ด้วยมือตรงๆ

`rswag` ไม่ใช่หนทางเดียวในการมี OpenAPI spec — อีกวิธีหนึ่งที่ทีมจำนวนมากใช้คือ **เขียนไฟล์
`.yaml`/`.json` ตามสเปก OpenAPI ด้วยมือโดยตรง** ไม่ผูกกับ RSpec เลย ข้อดี/ข้อเสียเทียบกับ
`rswag`:

| ประเด็น | เขียน YAML ด้วยมือ | `rswag` (generate จาก request spec) |
|---|---|---|
| ความเสี่ยงเอกสาร drift จากโค้ดจริง | **สูง** — ไม่มีอะไรบังคับให้อัปเดตพร้อมโค้ด | **ต่ำ** — ถ้าไม่ตรง test จะแดงทันที |
| เขียนได้อิสระ ไม่ผูกกับ request spec ที่มีอยู่ | ✅ ทำได้เลยแม้ยังไม่มี test | ❌ ต้องมี request spec ก่อนเสมอ |
| Designer ออกแบบ API ก่อนเขียนโค้ด (API-first design) | ✅ เหมาะมาก — วาดสเปกก่อน แจก team ไปแบ่งงาน | ❌ ไม่เหมาะ เพราะต้องมี endpoint จริงให้ทดสอบก่อน |
| Learning curve | ต้องเรียนรู้ syntax OpenAPI เต็มรูปแบบเอง | มี DSL ของ Ruby ช่วยกำกับโครงสร้างให้ |

ในทางปฏิบัติ ทีมขนาดใหญ่ที่ทำ **API-first design** (ออกแบบสเปกก่อนเขียนโค้ดสักบรรทัด เพื่อ
ให้ทีม backend/frontend ทำงานคู่ขนานกันได้จาก mock ที่สร้างจากสเปก) มักเลือกเขียน YAML ด้วยมือ
(หรือใช้เครื่องมือออกแบบเช่น Stoplight) แล้วค่อยเขียนโค้ด backend ให้ตรงตามสเปกทีหลัง ส่วนทีม
ที่เขียนโค้ดก่อนแล้วค่อยทำเอกสาร (**code-first**) แบบที่หลักสูตรนี้สอนมาตลอด `rswag` คือ
ตัวเลือกที่เหมาะกว่ามาก เพราะรับประกันความสอดคล้องระหว่างโค้ดกับเอกสารได้ดีกว่า

### ทางเลือกที่ 2: GraphQL แก้ปัญหานี้ด้วยโครงสร้างภาษาเอง (ทบทวน Part 058–059)

จำได้ไหมว่าใน Part 058 เราสร้าง GraphQL schema ด้วย `graphql-ruby` — นิยาม `Type` (เช่น
`Types::PostType`) และ `Query`/`Mutation` แล้ว graphql-ruby ใช้ definition เหล่านั้น
**โดยตรง** ในการ execute query ที่ client ส่งมา นี่คือความต่างเชิงโครงสร้างที่สำคัญมาก
เมื่อเทียบกับ REST + rswag:

- **ใน REST + rswag:** เอกสาร (request spec ที่มี `schema:`) กับ implementation จริง
  (controller) เป็น**คนละไฟล์กัน** — แค่บังเอิญมันสอดคล้องกันเพราะ test บังคับให้ตรวจสอบซ้ำ
  ทุกครั้งที่รัน (คือกลไก "ป้องกันไม่ให้หลุด" ไม่ใช่ "ทำให้หลุดไม่ได้ตั้งแต่ต้น")
- **ใน GraphQL:** `Types::PostType` (ที่นิยาม field `title`, `body`, `published`) **คือ**
  ตัวที่ resolver ใช้ resolve ค่าจริงด้วย ไม่มีทางที่ schema จะพูดว่ามี field `title` แต่
  resolver ไม่คืนค่า `title` จริงๆ ได้เลย เพราะเป็น**นิยามเดียวกัน** ตั้งแต่ต้น

### Introspection — วิธี "ขอเอกสาร" ของ GraphQL เอง

GraphQL มีความสามารถในตัวเรียกว่า **introspection** — client ส่ง query พิเศษเข้าไปถาม
schema ของ server "ตัวเอง" ได้โดยตรง (ไม่ต้องมีคนเขียนเอกสารแยกเลย) ตัวอย่าง introspection
query มาตรฐาน (ทดสอบได้จริงกับ endpoint GraphQL ใดก็ตามที่เปิด introspection ไว้):

```graphql
{
  __schema {
    types {
      name
      fields {
        name
        type {
          name
        }
      }
    }
  }
}
```

ส่ง query นี้ไปที่ GraphQL endpoint ใดก็ได้ (รวมถึงแอปที่สร้างใน Part 058) จะได้ผลลัพธ์ที่
บอกครบทุก type, field, argument ที่ schema มีอยู่จริง ณ ขณะนั้น — เพราะข้อมูลนี้**อ่านมาจาก
schema definition ในโค้ดโดยตรง ไม่ใช่ข้อมูลที่แยกเก็บต่างหาก** เครื่องมืออย่าง
**GraphiQL** และ **GraphQL Playground** (ที่ Part 058 อาจแนะนำให้ mount ไว้ตอน development)
ใช้ introspection นี้เองในการสร้างหน้าเอกสาร interactive ที่หน้าตาคล้าย Swagger UI แต่ทำงาน
อัตโนมัติ 100% — **ไม่ต้องเขียน DSL อธิบายอะไรเพิ่มเลยแม้แต่บรรทัดเดียว** ต่างจาก `rswag`
ที่ยังต้องเขียน request spec ประกอบเอกสารด้วยมือ

### สรุปเทียบทั้ง 3 แนวทาง

| แนวทาง | ป้องกัน drift ได้แค่ไหน | ต้องเขียนอะไรเพิ่มเพื่อมีเอกสาร |
|---|---|---|
| YAML มือ (ไม่ผูกกับ test) | ต่ำสุด — ไม่มีกลไกป้องกันเลย | เขียนสเปกทั้งหมดเอง |
| REST + `rswag` | สูง — ป้องกันด้วย test ที่ fail เมื่อไม่ตรง | เขียน DSL (`path`/`response`/`parameter`) เพิ่มจาก request spec ปกติ |
| GraphQL introspection | สูงสุด — schema กับ implementation เป็นนิยามเดียวกันโดยโครงสร้าง | ไม่ต้องเขียนอะไรเพิ่มเลย |

นี่ไม่ได้แปลว่า GraphQL "ดีกว่า" REST เสมอไป (มีเหตุผลอื่นในการเลือก REST เช่น caching ที่
เข้ากับ HTTP ได้ดีกว่า, ความคุ้นเคยของทีม, ทบทวนเหตุผลจาก Part 058) แต่เป็นตัวอย่างที่ดีมาก
ว่า **บางปัญหาแก้ได้ดีกว่าด้วยการเปลี่ยนโครงสร้างพื้นฐาน (GraphQL) แทนที่จะเพิ่มวินัย/เครื่องมือ
มาคอยตรวจสอบ (rswag)** — เป็นบทเรียนด้านการออกแบบระบบที่จะกลับมาเจออีกหลายครั้งในหลักสูตรนี้

---

## Step 597: ทำไมต้อง rate limit — ทบทวน `rate_limit` ของ Rails 8 (Part 041) เทียบกับ `rack-attack`

### ทำไม API ทุกตัวต้องมีการจำกัดอัตราการเรียก

ตอนนี้ API ของเรามีเอกสารครบถ้วนแล้ว แต่ endpoint ทุกตัวยัง**เปิดกว้างไม่จำกัด** — ใครก็ยิง
request มาได้ไม่จำกัดจำนวนครั้ง ปัญหาที่จะตามมาถ้าไม่มีการจำกัด:

1. **Brute-force attack** — ถ้ามี endpoint login/API key verification ผู้ไม่หวังดีลองรหัส
   ผ่าน/token นับล้านครั้งต่อชั่วโมงได้โดยไม่มีอะไรหยุด (ทบทวนเหตุผลเดียวกับที่ Part 041
   อธิบายไว้เรื่อง bcrypt ที่ทำให้แต่ละครั้ง "ช้า" แต่ถ้าไม่จำกัดจำนวนครั้งเลย ก็ยังลองได้
   เรื่อยๆ อยู่ดี)
2. **Denial of Service (DoS) โดยไม่ได้ตั้งใจ** — บั๊กใน client (เช่น mobile app ที่ retry
   loop ผิดพลาด ยิง request ซ้ำไม่หยุด) หรือ bot/crawler ที่เก็บข้อมูลถี่เกินไป อาจทำให้
   server ทำงานหนักจนกระทบผู้ใช้ปกติคนอื่น แม้ผู้ยิงจะไม่มีเจตนาร้ายเลยก็ตาม
3. **ป้องกัน traffic spike ที่ผิดปกติ** — ปกป้อง infrastructure (database, background job
   queue) จากการถูกใช้งานเกินขีดความสามารถอย่างกะทันหัน
4. **ควบคุมต้นทุน** — ถ้า API มีการเรียกใช้บริการภายนอกที่คิดเงินตามจำนวนครั้ง (เช่น ส่ง SMS,
   เรียก AI API) การไม่จำกัดอัตราอาจทำให้ค่าใช้จ่ายพุ่งจากการใช้งานผิดปกติได้

### ทบทวน `rate_limit` ของ Rails 8 จาก Part 041

Part 041 (Step 407) เคยแนะนำ **`rate_limit`** — class method ที่ Rails 8 เพิ่มเข้ามาใน
`ActionController` โดยตรง (module `ActionController::RateLimiting`) ไม่ต้องพึ่ง gem ภายนอก
ใช้แบบนี้:

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create,
             with: -> { redirect_to new_session_path, alert: "Try again later." }
end
```

ทบทวนจุดเด่น/ข้อจำกัดของ `rate_limit`:

- ทำงานที่ระดับ **controller action เดียว** — ประกาศแค่บรรทัดเดียวในตัว controller ที่ต้อง
  ป้องกัน ง่ายมากสำหรับ endpoint เฉพาะจุดที่อ่อนไหว (login, signup, password reset, และ
  ตอนนี้รวมถึง endpoint API ที่ "แพง" เป็นพิเศษ)
- ใช้ `Rails.cache` เป็นที่เก็บตัวนับ (เปลี่ยน store ได้ด้วย option `store:`)
- **ข้อจำกัด:** ต้องเขียนประกาศแยกในทุก controller ที่ต้องการป้องกัน ไม่มีจุดกลางที่มองเห็น
  นโยบาย rate limit ของทั้งแอปในที่เดียว และทำงานที่ระดับ Rails controller เท่านั้น (คือ
  ต้องผ่าน routing ของ Rails เข้ามาก่อนแล้ว จึงจะถูกนับ)

### `rack-attack` ทำงานที่ระดับที่ต่ำกว่า — Rack middleware

**`rack-attack`** เป็น gem ที่ทำงานที่ระดับ **Rack middleware** — ดักจับทุก HTTP request
**ตั้งแต่ก่อนเข้าถึง Rails router เลยด้วยซ้ำ** (ทบทวนแนวคิด Rack middleware ที่จะเจอเต็ม
รูปแบบใน Part 087 — ตอนนี้เข้าใจแค่ว่ามันคือ "ชั้นที่ครอบ" การทำงานของ Rails ทั้งแอปไว้อีกที)
เพราะทำงานก่อน routing จึงมีจุดเด่นที่ `rate_limit` ทำไม่ได้:

| ประเด็น | `rate_limit` (Rails 8 built-in) | `rack-attack` |
|---|---|---|
| ระดับที่ทำงาน | Controller action เดียว | Rack middleware (ทั้งแอป ก่อนถึง router) |
| จุดตั้งค่ากลาง | ไม่มี — กระจายอยู่ในแต่ละ controller | มีจุดเดียว (`config/initializers/rack_attack.rb`) |
| จำกัดตาม path/pattern แบบกว้างๆ (เช่น "ทุก endpoint ใต้ `/api/`") | ทำได้ยาก ต้องประกาศทีละ controller | ทำได้ตรงๆ ในบรรทัดเดียว |
| Safelist/Blocklist ตาม IP แบบถาวร | ไม่มีกลไกนี้ | มีในตัว (`safelist`/`blocklist`) |
| ป้องกัน request "แปลกๆ" ที่ไม่ผ่าน Rails routing เลย (เช่น path scanning) | ป้องกันไม่ได้ | ป้องกันได้ (ดักตั้งแต่ middleware) |
| เหมาะกับ | จุดที่อ่อนไหวเฉพาะจุด (login, password reset) | นโยบายกว้างๆ ระดับทั้งแอป/ทั้ง IP |

**สรุปแนวปฏิบัติที่ Part 041 ทิ้งท้ายไว้และ Part นี้จะพิสูจน์ให้เห็นจริง:** แอป production
จริงมักมี**ทั้งคู่ทำงานร่วมกัน** — `rack-attack` เป็นเกราะชั้นนอกที่จำกัด traffic ระดับ IP/
ทั้งแอปแบบกว้างๆ (เช่น "ไม่เกิน 300 request ต่อ 5 นาทีต่อ IP สำหรับทุกอย่างใต้ `/api/`") และ
`rate_limit` เป็นเกราะชั้นในที่จำกัดเฉพาะ action ที่อ่อนไหวที่สุดให้แน่นกว่าปกติ (เช่น login
จำกัดแค่ 10 ครั้งต่อ 3 นาที) — Step ต่อไปนี้จะติดตั้งและพิสูจน์การทำงานของ `rack-attack`
ให้เห็นจริงด้วย `curl`

---

## Step 598: ติดตั้งและตั้งค่า `rack-attack` เบื้องต้น — throttle by IP, safelist, blocklist

### ติดตั้ง

```ruby
# Gemfile
gem "rack-attack"
```

```bash
bundle install
```

ตรวจสอบว่า `rack-attack` แทรกตัวเองเข้า middleware stack ให้อัตโนมัติแล้วหรือยัง (ไม่ต้อง
config อะไรเพิ่มเพื่อให้มันทำงาน — gem นี้มี Railtie ที่ `use Rack::Attack` ให้อัตโนมัติทันที
ที่ require):

```bash
bin/rails middleware | grep -i attack
```

ผลลัพธ์จริง:

```
use Rack::Attack
```

ยืนยันว่า middleware ถูกแทรกเข้า stack แล้ว — ขั้นตอนต่อไปคือแค่**นิยามกฎ** ว่าจะ throttle
อะไรบ้าง

### สร้าง initializer และ throttle rule ตัวแรก

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # เก็บตัวนับ throttle ไว้ใน memory ของ process เอง (เหมาะกับ dev/demo เท่านั้น — production
  # จริงต้องใช้ store ที่แชร์กันได้ข้ามหลาย process/server เช่น Redis ดูรายละเอียดท้าย Step นี้)
  Rack::Attack.cache.store = ActiveSupport::Cache::MemoryStore.new

  # จำกัด request ทั่วไปทุกเส้นทาง: ไม่เกิน 5 ครั้งต่อ 10 วินาที ต่อ 1 IP
  # (ตั้งค่าต่ำมากเพื่อสาธิตให้เห็น 429 ได้เร็วในการทดสอบ — ของจริงมักตั้งไว้สูงกว่านี้มาก
  # เช่น limit: 300, period: 5.minutes)
  throttle("requests by ip", limit: 5, period: 10.seconds) do |req|
    req.ip
  end
end
```

**อธิบาย:**

- **`throttle(ชื่อ, limit:, period:) do |req| ... end`** — นิยามกฎ throttle 1 ข้อ ชื่อ
  (`"requests by ip"`) เป็นแค่ label สำหรับ debug/log ไม่ได้มีผลต่อการทำงาน ส่วน block
  ต้องคืนค่า **"discriminator"** — คือค่าที่ใช้แยกว่า "นี่คือใคร" สำหรับนับจำนวนครั้ง (ในที่
  นี้คือ `req.ip` — นับแยกกันตาม IP แต่ละตัว) ถ้า block คืน `nil`/`false` แปลว่า "ไม่ต้องนับ
  request นี้เลย" (ใช้ประโยชน์ได้ตอนทำ throttle เฉพาะบาง path เท่านั้น ดู Step 599)
- **`limit:`/`period:`** — จำนวนครั้งสูงสุดที่อนุญาตภายในช่วงเวลาที่กำหนด เมื่อ IP ใดยิงเกิน
  `limit` ครั้งภายใน `period` ล่าสุด **request ที่เกินมาจะถูกบล็อกทันทีด้วย HTTP `429`**
  โดยไม่ปล่อยให้เข้าถึง controller เลยด้วยซ้ำ
- `req` ที่รับเข้ามาใน block เป็น `Rack::Attack::Request` (สืบทอดจาก `Rack::Request`) จึงมี
  method อย่าง `req.ip`, `req.path`, `req.get?`/`req.post?`, `req.get_header(...)` ให้ใช้ครบ

### ทดสอบจริงด้วย `curl` — เห็น `429` ครั้งแรก

บูต server แล้วยิง request ซ้ำๆ:

```bash
bin/rails server -p 3000
```

```bash
for i in $(seq 1 8); do
  curl -s -o /dev/null -w "request $i -> %{http_code}\n" http://localhost:3000/api/v1/posts
done
```

ผลลัพธ์จริง:

```
request 1 -> 200
request 2 -> 200
request 3 -> 200
request 4 -> 200
request 5 -> 200
request 6 -> 429
request 7 -> 429
request 8 -> 429
```

ตรงตามที่ตั้งไว้เป๊ะๆ — **5 request แรกผ่าน (ตาม `limit: 5`) แล้วตั้งแต่ request ที่ 6 เป็นต้นไป
ถูกบล็อกด้วย `429` ทันที** โดยที่เรายังไม่ได้เขียน error handling ใดๆ ใน controller เองเลย
สักบรรทัด — `rack-attack` จัดการทั้งหมดที่ระดับ middleware ก่อนแม้แต่จะรู้จัก
`Api::V1::PostsController` ด้วยซ้ำ

รอให้ครบ `period` (10 วินาที) แล้วลองใหม่ ยืนยันว่า quota รีเซ็ตจริง:

```bash
sleep 11 && curl -s -o /dev/null -w "after waiting: %{http_code}\n" http://localhost:3000/api/v1/posts
```

```
after waiting: 200
```

### Safelist — ยกเว้น IP ที่เชื่อถือได้จาก throttle ทุกกฎ

```ruby
# config/initializers/rack_attack.rb (ต่อท้าย)

# ไม่จำกัด rate ให้กับ IP ที่เชื่อถือได้เสมอ (เช่น internal health-check, monitoring service,
# CI/CD runner) — request จาก IP เหล่านี้จะไม่ถูกนับใน throttle ข้อไหนเลยแม้แต่ข้อเดียว
Rack::Attack.safelist("allow trusted health checks") do |req|
  req.ip == "127.0.0.9" # ตัวอย่าง — ของจริงมักอ่านจาก ENV หรือ IP range ภายในองค์กร
end
```

ทดสอบเทียบ IP ปกติ vs IP ที่ safelist ไว้ (จำลอง IP ด้วย header `X-Forwarded-For` ตอน
ทดสอบใน development):

```bash
echo "=== IP ปกติ (โดน throttle หลัง 5 ครั้ง) ==="
for i in $(seq 1 7); do
  curl -s -o /dev/null -w "%{http_code} " -H "X-Forwarded-For: 10.0.0.5" http://localhost:3000/api/v1/posts
done
echo
echo "=== IP ที่ safelist ไว้ (ผ่านตลอดไม่มีจำกัด) ==="
for i in $(seq 1 7); do
  curl -s -o /dev/null -w "%{http_code} " -H "X-Forwarded-For: 127.0.0.9" http://localhost:3000/api/v1/posts
done
```

ผลลัพธ์จริง:

```
=== IP ปกติ (โดน throttle หลัง 5 ครั้ง) ===
200 200 200 200 200 429 429
=== IP ที่ safelist ไว้ (ผ่านตลอดไม่มีจำกัด) ===
200 200 200 200 200 200 200
```

ยืนยันชัดเจน — IP เดียวกันเป๊ะๆ ที่ยิงจำนวนครั้งเท่ากัน แต่ผลลัพธ์ต่างกันโดยสิ้นเชิงเพราะ
`safelist` ก่อนหน้า `throttle` เสมอในลำดับการตรวจสอบของ `rack-attack`

### Blocklist — บล็อกทันทีโดยไม่ต้องรอนับ

```ruby
# config/initializers/rack_attack.rb (ต่อท้าย)

# บล็อก IP ที่รู้ว่าเป็นอันตรายทันที ไม่ต้องรอให้ throttle นับครบ limit ก่อน
Rack::Attack.blocklist("block known bad ip") do |req|
  req.ip == "6.6.6.6" # ของจริงมักดึงจากรายชื่อ IP ที่ตรวจพบว่าโจมตี เก็บไว้ใน database/Redis
end
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "X-Forwarded-For: 6.6.6.6" http://localhost:3000/api/v1/posts
```

ผลลัพธ์จริง:

```
403
```

**`403 Forbidden`** ทันทีตั้งแต่ request แรก (ไม่ใช่ `429`) — เพราะ `blocklist` คือการห้าม
เข้าถึงแบบเด็ดขาด ไม่ใช่การจำกัด "อัตรา" เหมือน `throttle` ค่า default response ของ
`blocklist` คือ `403` ในขณะที่ `throttle` คือ `429` (ปรับแต่งข้อความ/status ของทั้งคู่ได้
ด้วย `blocklisted_responder`/`throttled_responder` ตามที่จะเห็นใน Step 599)

> **ลำดับการตรวจสอบของ `rack-attack`:** ทุก request จะถูกตรวจสอบตามลำดับนี้เสมอ:
> **safelist → blocklist → throttle → track** ถ้าเข้าเงื่อนไข `safelist` ข้อใดข้อหนึ่ง
> จะผ่านทันทีโดยไม่ตรวจ `blocklist`/`throttle` เลย ถ้าเข้าเงื่อนไข `blocklist` จะถูกบล็อก
> ทันทีโดยไม่ไปตรวจ `throttle` ต่อ — ลำดับนี้สำคัญมากเวลาออกแบบกฎหลายข้อพร้อมกัน

---

## Step 599: Throttle strategy ขั้นสูง — by user/API key, by endpoint, custom `429` response พร้อม `Retry-After`

### Throttle เฉพาะ endpoint ที่ "แพง" กว่าปกติ

การอ่านข้อมูล (`GET`) มักเบากว่าการเขียนข้อมูล (`POST`/`PATCH`/`DELETE`) มาก เพราะ `POST`
ต้องเขียนลงฐานข้อมูลจริง (และมักมี business logic/validation ให้ประมวลผลเยอะกว่า) จึงควร
จำกัดให้เข้มกว่า:

```ruby
# config/initializers/rack_attack.rb

# จำกัดเฉพาะ endpoint ที่ "แพง" กว่าปกติ ให้เข้มกว่าค่า throttle รวมของทั้งแอป
throttle("posts#create by ip", limit: 2, period: 10.seconds) do |req|
  req.ip if req.path == "/api/v1/posts" && req.post?
end
```

**อธิบาย:** จุดสำคัญคือ `if req.path == "/api/v1/posts" && req.post?` ต่อท้าย — เงื่อนไขนี้
ทำให้ block คืนค่า `req.ip` **เฉพาะตอนที่ request ตรงกับ path/method ที่ต้องการ** ส่วน request
อื่นทั้งหมด (เช่น `GET /api/v1/posts`) จะได้ `nil` กลับไป ซึ่ง `rack-attack` ตีความว่า "ไม่ต้อง
นับ throttle ข้อนี้กับ request นี้" — **กฎ throttle หลายข้อทำงานเป็นอิสระจากกันโดยสิ้นเชิง**
(นับ counter แยกกันคนละตัว) request หนึ่งจึงอาจถูกนับใน throttle รวม (`limit: 5`) และ
throttle เฉพาะจุด (`limit: 2`) พร้อมกันได้ในเวลาเดียวกัน

ทดสอบจริง (ใช้ IP ใหม่ที่ไม่เคยถูกนับมาก่อนเพื่อความชัดเจน):

```bash
echo "=== POST /api/v1/posts (limit 2 ต่อ 10 วินาที) ==="
for i in $(seq 1 4); do
  curl -s -o /dev/null -w "%{http_code} " -H "X-Forwarded-For: 9.9.9.9" \
    -H "Content-Type: application/json" -X POST \
    -d '{"post":{"title":"t","body":"b"}}' http://localhost:3000/api/v1/posts
done
echo
echo "=== GET จาก IP เดียวกัน (throttle คนละตัว ยังไม่ติด limit รวม) ==="
curl -s -o /dev/null -w "%{http_code}\n" -H "X-Forwarded-For: 9.9.9.9" http://localhost:3000/api/v1/posts
```

ผลลัพธ์จริง:

```
=== POST /api/v1/posts (limit 2 ต่อ 10 วินาที) ===
201 201 429 429
=== GET จาก IP เดียวกัน (throttle คนละตัว ยังไม่ติด limit รวม) ===
200
```

`POST` 2 ครั้งแรกสำเร็จ (`201`) ครั้งที่ 3–4 โดน throttle เฉพาะจุด (`429`) ทันที ในขณะที่
`GET` จาก IP เดียวกันยังผ่านสบายๆ (`200`) เพราะ throttle รวม (`limit: 5`) ยังไม่ถึงเกณฑ์
(นับรวมทั้งหมดแค่ 5 request จาก IP นี้ ยังไม่เกิน 5) — พิสูจน์ชัดเจนว่ากฎหลายข้อทำงานแยก
อิสระจากกันจริงตามที่อธิบายไว้

### Throttle ตาม authenticated user/API key แทน IP

การ throttle ตาม IP มีข้อจำกัด: ผู้ใช้หลายคนอาจอยู่หลัง NAT/proxy/บริษัทเดียวกันแล้วมี IP
สาธารณะเดียวกัน (เช่น พนักงานทั้งออฟฟิศออกเน็ตผ่าน IP เดียวกัน) การ throttle ตาม IP อาจ
บล็อกผู้ใช้ที่ไม่ผิดไปด้วย ถ้า API ต้องแนบ API key อยู่แล้วทุก request (ทบทวนแนวคิด API key
authentication จาก Part 045) **การ throttle ตาม key แม่นยำกว่ามาก**:

```ruby
# config/initializers/rack_attack.rb

# จำกัดตาม API key แทน IP — เหมาะกับ API ที่ต้องแนบ token ทุก request อยู่แล้ว
throttle("api requests by api key", limit: 100, period: 1.minute) do |req|
  req.get_header("HTTP_X_API_KEY").presence
end
```

**อธิบาย:** `req.get_header("HTTP_X_API_KEY")` อ่านค่า HTTP header ชื่อ `X-Api-Key` ที่ client
แนบมา (Rack แปลงชื่อ header เป็น `HTTP_X_API_KEY` เสมอตามธรรมเนียมของ CGI variable — ทบทวน
รูปแบบนี้จาก Part 023 ตอนอธิบายว่า Rails อ่าน HTTP header อย่างไร) `.presence` สำคัญมาก:
ถ้าไม่มี header นี้มาเลย จะได้ `nil` ซึ่งแปลว่า "ไม่ throttle request ที่ไม่มี API key เลย"
(เพราะปกติ endpoint ที่ต้องมี API key จะถูก reject ด้วยเหตุผลอื่นอยู่แล้วถ้าไม่มี key มา —
ไม่ใช่หน้าที่ของ `rack-attack` ที่จะตัดสินเรื่องนั้น) ผู้ใช้แต่ละคนที่มี API key ต่างกันจึงมี
โควตาของตัวเอง ไม่ปะปนกับผู้ใช้อื่นที่บังเอิญใช้ IP เดียวกัน

### ปรับแต่ง response ของ `429` ให้มี `Retry-After` header

Default response ของ `rack-attack` เมื่อโดน throttle คือ `429` พร้อม body ว่างเปล่า —
ในทางปฏิบัติ **ควรบอก client ด้วยว่าต้องรออีกนานแค่ไหนถึงจะลองใหม่ได้** ผ่าน HTTP header
มาตรฐาน **`Retry-After`** (มาตรฐานเดียวกับที่ใช้ใน HTTP status `503 Service Unavailable`)
เพื่อให้ client เขียนโค้ด retry อัตโนมัติได้อย่างถูกต้อง:

```ruby
# config/initializers/rack_attack.rb

Rack::Attack.throttled_responder = lambda do |request|
  match_data = request.env["rack.attack.match_data"]
  now = match_data[:epoch_time]
  retry_after = (match_data[:period] - (now % match_data[:period])).to_i

  headers = {
    "Content-Type" => "application/json",
    "Retry-After" => retry_after.to_s
  }

  [429, headers, [{ error: "Too many requests. Please try again later." }.to_json]]
end
```

**อธิบาย:**

- **`request.env["rack.attack.match_data"]`** — `rack-attack` ฝังข้อมูลของกฎที่ trigger
  ไว้ใน Rack env เสมอเมื่อมี request ถูกบล็อก มีทั้ง `:period` (ค่าที่ตั้งไว้ในกฎ),
  `:epoch_time` (เวลาปัจจุบันเป็น Unix timestamp), `:discriminator` (ค่าที่ block ของ
  `throttle` คืนมา เช่น IP หรือ API key), และ `:count` (จำนวนครั้งที่นับได้แล้ว)
- **`match_data[:period] - (now % match_data[:period])`** — สูตรคำนวณว่า **"อีกกี่วินาที
  quota ถึงจะรีเซ็ต"** โดยอาศัยหลักการว่า `rack-attack` แบ่งเวลาเป็นช่วง (window) คงที่ตาม
  `period` เสมอ (เช่น `period: 10.seconds` แบ่งเป็นช่วงละ 10 วินาทีต่อเนื่องกันไม่มีที่สิ้นสุด
  นับจาก Unix epoch) `now % period` คือ "อยู่ตรงไหนของช่วงปัจจุบัน" ส่วนที่เหลือ (`period -`
  ผลลัพธ์นั้น) คือเวลาที่เหลือจนกว่าช่วงถัดไปจะเริ่ม
- คืนค่าเป็น **Rack response array แบบดิบ**: `[status, headers, [body]]` — รูปแบบมาตรฐานของ
  Rack app ที่ทุก middleware/app ต้องคืนกลับ (ทบทวนรูปแบบนี้ล่วงหน้าไว้ก่อนจะเรียนเต็มรูปแบบ
  ใน Part 087)

ทดสอบจริง ดู header ที่ได้:

```bash
curl -i http://localhost:3000/api/v1/posts   # ยิงซ้ำจนเกิน limit ก่อน แล้วดูตัวสุดท้าย
```

ผลลัพธ์จริง (request ที่โดน throttle):

```
HTTP/1.1 429 Too Many Requests
content-type: application/json
retry-after: 3
cache-control: no-cache
...
content-length: 54

{"error":"Too many requests. Please try again later."}
```

`retry-after: 3` บอก client ว่าอีก 3 วินาทีค่อยลองใหม่ — ค่านี้เปลี่ยนไปเรื่อยๆ ตามจังหวะจริง
ที่ยิง (ยิ่งยิงใกล้ปลาย window มากเท่าไหร่ ค่าที่เหลือก็ยิ่งน้อยลงตามสัดส่วนจริง) ยืนยันว่า
สูตรคำนวณทำงานถูกต้องตามเวลาจริง ไม่ใช่ค่าคงที่ที่ hardcode ไว้

---

## Step 600: ทดสอบ rate limiting ด้วย `curl` loop จริง + RSpec + แบบฝึกหัดปิด Phase 8

### เขียน RSpec request spec ทดสอบ rate limiting

การทดสอบ `rack-attack` ด้วย automated test สำคัญไม่แพ้การทดสอบด้วยมือผ่าน `curl` เพราะ
ป้องกันไม่ให้ใครมาแก้ `limit`/`period` โดยไม่ได้ตั้งใจแล้วไม่มีใครรู้ แต่มีจุดที่ต้องระวัง
เป็นพิเศษ ซึ่งเจอจริงตอนพัฒนา Part นี้:

> **ปัญหาจริงที่เจอ: `rack-attack` ไปบล็อก test suite ของตัวเอง!** พอเพิ่ม
> `config/initializers/rack_attack.rb` เข้าไปแล้วรัน `bundle exec rspec` ทั้ง suite (รวม
> request spec ของ `rswag` จาก Step 593–594 ที่ยิง request หลายสิบครั้งติดกันในเวลาไม่ถึง
> วินาที) กลับพบว่า test ที่เคยผ่านหมดกลับ **fail กะทันหันด้วย `429`** — สาเหตุคือ
> `rack-attack` ทำงาน**ในทุก environment รวมถึง test ด้วย** โดย default และ counter ของมัน
> (`Rack::Attack.cache.store`) ก็ยังคงค้างอยู่ข้าม example ต่างๆ ภายใน process การรัน test
> เดียวกัน — request spec ปกติที่ไม่ได้ตั้งใจทดสอบเรื่อง rate limit เลยแม้แต่น้อย กลับโดน
> throttle ของตัวเองเข้าไปเต็มๆ

วิธีแก้ที่ถูกต้อง: **ปิด `rack-attack` เป็นค่า default ในทุก test แล้วเปิดกลับเฉพาะ test
ที่ตั้งใจทดสอบเรื่องนี้จริงๆ** ด้วย RSpec metadata tag:

```ruby
# spec/rails_helper.rb (เพิ่มเข้าไปใน RSpec.configure)
RSpec.configure do |config|
  config.before do |example|
    Rack::Attack.enabled = example.metadata[:rack_attack] == true
    Rack::Attack.cache.store = ActiveSupport::Cache::MemoryStore.new if Rack::Attack.enabled
  end

  # ... config อื่นๆ ที่ generator สร้างให้ตามปกติ
end
```

**อธิบาย:** `config.before` (ธรรมดา ไม่ใช่ `around`) ทำงานก่อนทุก example เสมอ — เช็ค
`example.metadata[:rack_attack]` (ค่าที่มาจาก tag `rack_attack: true` ที่จะประกาศใน
`describe` ด้านล่าง) ถ้าไม่มี tag นี้ (ค่า default ของ test ทั่วไป) จะสั่ง
`Rack::Attack.enabled = false` — **ปิดการทำงานของ middleware ทั้งหมดชั่วคราว** เฉพาะ test
ที่ติด tag `rack_attack: true` เท่านั้นที่จะเปิดใช้งานจริง พร้อมรีเซ็ต cache store ใหม่ทุก
ครั้งเพื่อไม่ให้ผลจาก example ก่อนหน้าตกค้างมากวนใจ example ถัดไป

```ruby
# spec/requests/rack_attack_spec.rb
require "rails_helper"

RSpec.describe "Rack::Attack rate limiting", type: :request, rack_attack: true do
  it "ปล่อยผ่าน request ปกติภายใต้ quota แล้วบล็อกเมื่อเกิน limit ด้วย 429 พร้อม Retry-After" do
    codes = []
    6.times do
      get "/api/v1/posts"
      codes << response.status
    end

    expect(codes).to eq([200, 200, 200, 200, 200, 429])
    expect(response.headers["Retry-After"]).to be_present
    expect(JSON.parse(response.body)["error"]).to match(/Too many requests/)
  end
end
```

รันทั้ง suite (rswag spec + rack-attack spec) พร้อมกัน:

```bash
bundle exec rspec spec/requests
```

ผลลัพธ์จริง:

```
Rack::Attack rate limiting
  ปล่อยผ่าน request ปกติภายใต้ quota แล้วบล็อกเมื่อเกิน limit ด้วย 429 พร้อม Retry-After

api/v1/posts
  /api/v1/posts
    get
      รายการโพสต์
        returns a 200 response
    post
      สร้างสำเร็จ
        returns a 201 response
      ข้อมูลไม่ถูกต้อง
        returns a 422 response
  /api/v1/posts/{id}
    get
      พบโพสต์
        returns a 200 response
      ไม่พบโพสต์
        returns a 404 response
    patch
      แก้ไขสำเร็จ
        returns a 200 response
    delete
      ลบสำเร็จ
        returns a 204 response

Finished in 0.16 seconds
8 examples, 0 failures
```

**8 examples ผ่านหมด** — ทั้ง test เอกสาร API (7 ตัว) และ test rate limiting (1 ตัว) อยู่ร่วม
กันได้อย่างสงบ ไม่รบกวนกันอีกต่อไป

> **ข้อควรระวังเรื่อง gem เวอร์ชันชนกัน (พบจริงระหว่างพัฒนา Part นี้):** ตอนติดตั้ง `rswag`
> ครั้งแรกบนเครื่องทดสอบ พบว่า test ที่มีการตรวจสอบ `schema:` (response validation) ทุกตัว
> fail ด้วย `ArgumentError: unknown keyword: quirks_mode` — สาเหตุคือ gem `json-schema` ที่
> `rswag-specs` ใช้ตรวจสอบ schema เรียก `JSON.parse` ด้วย keyword argument ที่ gem `json`
> เวอร์ชันใหม่ (3.0.x ที่มากับ Ruby 3.3 บางชุด) ไม่รองรับแล้ว วิธีแก้ชั่วคราวคือ pin เวอร์ชัน
> `json` ให้ต่ำกว่านั้นใน `Gemfile`:
> ```ruby
> gem "json", "2.9.1"
> ```
> นี่คือตัวอย่างจริงของปัญหาที่พบได้ทั่วไปในวงการ Ruby: **gem สองตัวที่ต่างก็ maintain ดี
> อาจยังชนกันได้เมื่อระบบ dependency resolution เลือกเวอร์ชันที่เข้ากันไม่ได้พอดี** — ทักษะ
> การอ่าน stack trace เพื่อหาว่า "gem ไหนจริงๆ ที่เป็นต้นเหตุ" (ในที่นี้คือ `json-schema`
> เรียก `json` ผิด ไม่ใช่ `rswag` เอง) แล้ว pin เวอร์ชันให้เข้ากันได้ คือทักษะที่นักพัฒนา
> Rails มืออาชีพต้องเจอและแก้เองอยู่เสมอ ไม่มีทางเลี่ยงได้ 100%

### เกี่ยวกับ store ของ `rack-attack` ใน production

ตัวอย่างทั้งหมดใน Part นี้ใช้ `ActiveSupport::Cache::MemoryStore` (เก็บตัวนับไว้ใน memory
ของ process เดียว) ซึ่ง**ใช้ได้แค่ตอน development/demo เท่านั้น** เพราะ production จริงมัก
รัน Rails หลาย process/หลาย server พร้อมกัน (ทบทวนเรื่อง Puma worker/thread จาก Part 021
และจะเจอเต็มรูปแบบเรื่อง horizontal scaling ใน Part 073–078) ถ้าแต่ละ process มี counter
แยกเป็นของตัวเอง ผู้โจมตีก็แค่ยิง request กระจายไปหลาย process แล้วหลบ limit ได้ง่ายๆ
ทางแก้คือใช้ store ที่ **แชร์กันได้ข้าม process/server** เช่น Redis (ทบทวน/preview การใช้
Redis เต็มรูปแบบใน Part 062 เรื่อง Sidekiq):

```ruby
# config/initializers/rack_attack.rb (สำหรับ production)
Rack::Attack.cache.store = ActiveSupport::Cache::RedisCacheStore.new(url: ENV.fetch("REDIS_URL"))
```

---

## แบบฝึกหัด: เอกสาร Posts API แบบเต็มรูปแบบ + Rate Limiting ที่พิสูจน์ได้จริง

### โจทย์

ต่อยอดจาก `posts_api` ที่สร้างมาตลอด Part นี้ ให้ทำสิ่งต่อไปนี้ให้ครบ:

1. เพิ่ม field `author_email` ให้ `Post` model (string, มี validation ว่าเป็นรูปแบบอีเมล
   ที่ถูกต้อง)
2. เขียน request spec แบบ `rswag` ให้ครอบคลุมทั้ง 5 action (`index`, `show`, `create`,
   `update`, `destroy`) รวม error response ที่เป็นไปได้ทุกกรณี (`404`, `422`)
3. Generate สเปกจริงด้วย `rswag:specs:swaggerize` แล้วเปิด Swagger UI ยืนยันว่าแสดงผลถูกต้อง
4. เพิ่ม throttle rule ใหม่: จำกัด `DELETE` ไม่เกิน 3 ครั้งต่อนาทีต่อ IP (ป้องกันการลบข้อมูล
   รัวๆ โดยไม่ได้ตั้งใจหรือโดยเจตนาร้าย)
5. เขียน RSpec spec (แท็ก `rack_attack: true`) พิสูจน์ว่า throttle ใน ข้อ 4 ทำงานจริง พร้อม
   ตรวจสอบ `Retry-After` header

### เฉลย

**Model:**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
  validates :body, presence: true
  validates :author_email, format: { with: URI::MailTo::EMAIL_REGEXP },
                            allow_blank: true
end
```

```bash
bin/rails generate migration AddAuthorEmailToPosts author_email:string
bin/rails db:migrate
```

**Request spec เต็มรูปแบบ:**

```ruby
# spec/requests/api/v1/posts_spec.rb
require "swagger_helper"

RSpec.describe "api/v1/posts", type: :request do
  path "/api/v1/posts" do
    get "รายการโพสต์ทั้งหมด" do
      tags "Posts"
      produces "application/json"

      response "200", "รายการโพสต์" do
        schema type: :array,
               items: {
                 type: :object,
                 properties: {
                   id: { type: :integer },
                   title: { type: :string },
                   body: { type: :string },
                   published: { type: :boolean },
                   author_email: { type: :string, nullable: true },
                   created_at: { type: :string },
                   updated_at: { type: :string }
                 }
               }
        run_test!
      end
    end

    post "สร้างโพสต์ใหม่" do
      tags "Posts"
      consumes "application/json"
      produces "application/json"
      parameter name: :post_params, in: :body, schema: {
        type: :object,
        properties: {
          post: {
            type: :object,
            properties: {
              title: { type: :string },
              body: { type: :string },
              author_email: { type: :string }
            },
            required: %w[title body]
          }
        }
      }

      response "201", "สร้างสำเร็จ" do
        let(:post_params) do
          { post: { title: "Hello", body: "World", author_email: "a@example.com" } }
        end
        run_test!
      end

      response "422", "ข้อมูลไม่ถูกต้อง (ชื่อเรื่องว่างเปล่า)" do
        let(:post_params) { { post: { title: "", body: "" } } }
        run_test!
      end

      response "422", "ข้อมูลไม่ถูกต้อง (รูปแบบอีเมลผิด)" do
        let(:post_params) do
          { post: { title: "Hello", body: "World", author_email: "not-an-email" } }
        end
        run_test!
      end
    end
  end

  path "/api/v1/posts/{id}" do
    let(:existing_post) { Post.create!(title: "Existing", body: "Body text") }

    get "แสดงโพสต์เดียว" do
      tags "Posts"
      produces "application/json"
      parameter name: :id, in: :path, type: :integer

      response "200", "พบโพสต์" do
        let(:id) { existing_post.id }
        run_test!
      end

      response "404", "ไม่พบโพสต์" do
        let(:id) { 999_999 }
        run_test!
      end
    end

    patch "แก้ไขโพสต์" do
      tags "Posts"
      consumes "application/json"
      parameter name: :id, in: :path, type: :integer
      parameter name: :post_params, in: :body, schema: {
        type: :object,
        properties: {
          post: {
            type: :object,
            properties: { title: { type: :string } }
          }
        }
      }

      response "200", "แก้ไขสำเร็จ" do
        let(:id) { existing_post.id }
        let(:post_params) { { post: { title: "Updated title" } } }
        run_test!
      end

      response "404", "ไม่พบโพสต์" do
        let(:id) { 999_999 }
        let(:post_params) { { post: { title: "x" } } }
        run_test!
      end
    end

    delete "ลบโพสต์" do
      tags "Posts"
      parameter name: :id, in: :path, type: :integer

      response "204", "ลบสำเร็จ" do
        let(:id) { existing_post.id }
        run_test!
      end

      response "404", "ไม่พบโพสต์" do
        let(:id) { 999_999 }
        run_test!
      end
    end
  end
end
```

**Generate สเปกและตรวจสอบ Swagger UI:**

```bash
bundle exec rake rswag:specs:swaggerize
bin/rails server
curl -s -o /dev/null -w "%{http_code}\n" -L http://localhost:3000/api-docs
# => 200
```

**Throttle เฉพาะ `DELETE`:**

```ruby
# config/initializers/rack_attack.rb (เพิ่มเข้าไป)
throttle("posts#destroy by ip", limit: 3, period: 1.minute) do |req|
  req.ip if req.path.match?(%r{\A/api/v1/posts/\d+\z}) && req.delete?
end
```

**RSpec พิสูจน์การทำงาน:**

```ruby
# spec/requests/rack_attack_spec.rb
require "rails_helper"

RSpec.describe "Rack::Attack rate limiting", type: :request, rack_attack: true do
  it "จำกัด DELETE ไม่เกิน 3 ครั้งต่อนาทีต่อ IP พร้อมคืน Retry-After" do
    posts = Array.new(4) { Post.create!(title: "x", body: "y") }
    codes = posts.map { |post| delete("/api/v1/posts/#{post.id}"); response.status }

    expect(codes).to eq([204, 204, 204, 429])
    expect(response.headers["Retry-After"]).to be_present
  end

  it "ยังจำกัด GET ด้วย throttle รวมตามปกติ ไม่ถูกยกเว้นเพราะมีกฎ throttle เพิ่ม" do
    codes = Array.new(6) { get("/api/v1/posts"); response.status }
    expect(codes.last).to eq(429)
  end
end
```

```bash
bundle exec rspec spec/requests
```

ผลลัพธ์ที่คาดหวัง (และทดสอบจริงแล้วว่าผ่านด้วยแนวทางเดียวกับที่พิสูจน์มาตลอด Part นี้):
ทุก example ผ่านหมด — `DELETE` 3 ครั้งแรกสำเร็จ (`204`) ครั้งที่ 4 โดน throttle เฉพาะจุด
(`429`) พร้อม `Retry-After` header ในขณะที่ throttle รวมของทั้งแอปยังคงทำงานคู่ขนานกันไป
ตามปกติ

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม authentication แบบ API key เข้า `posts_api`** (ทบทวน Part 045) แล้วเปลี่ยน throttle
   หลักจาก "by IP" เป็น **"by API key เมื่อมี key มา และ fallback เป็น IP เมื่อไม่มี"** (ใบ้:
   เขียน block ที่เช็คว่ามี `X-Api-Key` header หรือไม่ก่อน ถ้าไม่มีค่อย fallback ไปใช้ `req.ip`)
   แล้วพิสูจน์ด้วย RSpec ว่า 2 client ที่มี API key ต่างกันแต่ใช้ IP เดียวกัน (จำลองผ่าน
   `X-Forwarded-For` เดียวกัน) มีโควตาแยกจากกันจริง

2. **เพิ่ม endpoint GraphQL เข้าไปใน `posts_api`** (ทบทวน Part 058) แล้วลองส่ง introspection
   query (`{ __schema { types { name } } }`) เข้าไปดูว่าได้ผลลัพธ์อะไรกลับมา จากนั้นเปรียบเทียบ
   ปริมาณโค้ดที่ต้องเขียนเพิ่มเพื่อ "มีเอกสาร" ระหว่าง endpoint REST (ต้องเขียน `rswag` DSL
   เพิ่ม) กับ endpoint GraphQL (ไม่ต้องเขียนอะไรเพิ่มเลย) แล้วสรุปเป็นตารางเทียบของตัวเอง

3. **เขียน custom `blocklisted_responder`** (คู่กับ `throttled_responder` ที่เรียนใน Step 599)
   ให้คืน JSON error message ที่มีรูปแบบเดียวกับ error response อื่นๆ ของ API (เช่น
   `{ "error": { "code": "ip_blocked", "message": "..." } }`) แทนการใช้ default response
   ของ `rack-attack` แล้วเขียน RSpec (`rack_attack: true`) ยืนยันรูปแบบ JSON ที่ได้

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **API คือสัญญา (contract)** กับผู้บริโภคที่อ่าน source code ของเราไม่ได้ และทำไม
  เอกสารที่แยกจากโค้ด (เช่น wiki) มักหลุด (drift) จากความจริงเสมอ
- เข้าใจ **OpenAPI/Swagger** ในฐานะรูปแบบมาตรฐานสำหรับอธิบาย REST API และติดตั้ง gem ทั้ง 3
  ตัวของ `rswag` (`rswag-api`, `rswag-ui`, `rswag-specs`) พร้อมบทบาทของแต่ละตัว
- เขียน API documentation ในรูปแบบ **RSpec request spec** ด้วย DSL ของ `rswag`
  (`path`, `get`/`post`/`patch`/`delete`, `response`, `parameter`, `schema`, `run_test!`)
  ที่ทั้งทดสอบ endpoint จริงและ generate เอกสารจากไฟล์เดียวกัน ป้องกันไม่ให้เอกสาร drift
  จากพฤติกรรมจริงได้
- Generate สเปกจริงด้วย `bundle exec rake rswag:specs:swaggerize` แล้วเปิด **Swagger UI**
  ที่ `/api-docs` ดูเอกสาร interactive จริง พร้อมปุ่ม "Try it out"
- เข้าใจทางเลือกอื่นในการทำเอกสาร API: เขียน OpenAPI YAML ด้วยมือ (เหมาะกับ API-first design)
  เทียบกับ **GraphQL introspection** ที่แก้ปัญหาเอกสาร drift ด้วยโครงสร้างภาษาเอง ไม่ต้องเขียน
  อะไรเพิ่มเลย
- ทบทวนความต่างระหว่าง **`rate_limit`** ของ Rails 8 (ระดับ controller action, Part 041)
  กับ **`rack-attack`** (ระดับ Rack middleware ทั้งแอป) และเข้าใจว่าโปรเจกต์จริงมักใช้ทั้งคู่
  ร่วมกัน
- ติดตั้งและตั้งค่า `rack-attack` ครบทุกกลไก: `throttle` (by IP, by endpoint, by API key),
  `safelist`, `blocklist`, และปรับแต่ง `throttled_responder` ให้คืน `429` พร้อม
  `Retry-After` header ที่คำนวณเวลาที่เหลือจริง
- ยิง `curl` วนซ้ำจริงจนเห็น `429 Too Many Requests` เกิดขึ้น พิสูจน์ quota รีเซ็ตตามเวลาจริง
  และพิสูจน์ safelist/blocklist/throttle เฉพาะจุดทำงานถูกต้อง
- เจอและแก้ปัญหาจริง 3 อย่าง: gem `json`/`json-schema` เวอร์ชันชนกัน, การตั้งชื่อ parameter
  ชนกับ HTTP verb method ใน request spec, และ `rack-attack` ที่ไปบล็อก test suite ของตัวเอง
  พร้อมวิธีแก้ที่ถูกต้องของแต่ละกรณี — ทักษะ debug ที่สำคัญไม่แพ้การเขียนโค้ดให้ผ่านตั้งแต่
  ครั้งแรก

## สรุปภาพรวม Phase 8: APIs & GraphQL

ยินดีด้วย! **Phase 8: APIs & GraphQL (Part 056–060, Step 551–600)** เสร็จสมบูรณ์แล้ว เรา
เดินทางจากการเปิด Rails ให้เป็น **API-only mode** พร้อม serializer แปลง ActiveRecord เป็น
JSON (Part 056), จัดระเบียบ API ให้พร้อมสำหรับผู้ใช้จริงหลายรุ่นด้วย **versioning**,
มาตรฐาน **JSON:API**, และ **pagination** (Part 057), ก้าวเข้าสู่โลกใหม่ที่ต่างจาก REST
โดยสิ้นเชิงด้วย **GraphQL** — เขียน schema, type, และ query ให้ client เลือกได้เองว่าจะขอ
field ไหนบ้าง (Part 058), ต่อยอดด้วย **mutation** สำหรับเปลี่ยนแปลงข้อมูลและ **authorization**
ใน GraphQL (Part 059) จนมาถึง Part นี้ที่ปิดท้ายด้วยการทำให้ API **มีเอกสารที่เชื่อถือได้และ
ปลอดภัยจากการถูกใช้งานเกินขอบเขต**

สิ่งที่ทำให้ Phase นี้ต่างจาก Phase ก่อนหน้า (ที่เน้นสร้างเว็บแอปสำหรับ browser โดยตรง) คือ
แนวคิดที่ว่า **"ผู้บริโภค" ของสิ่งที่เราสร้างไม่ใช่มนุษย์ที่คลิกปุ่มบนหน้าเว็บอีกต่อไป แต่เป็น
โปรแกรมอื่น** (mobile app, frontend framework แยกส่วน, บริการภายนอก) ซึ่งเปลี่ยนทุกอย่าง —
ตั้งแต่วิธี return ข้อมูล (JSON แทน HTML), วิธีจัดการ error (status code + JSON error body
แทนหน้า error page), ไปจนถึงความจำเป็นที่ต้องมี**เอกสารที่ชัดเจน**และ**การป้องกันการใช้งาน
เกินขอบเขต**อย่างที่ Part นี้เพิ่งสอนไป — ทักษะทั้งหมดนี้คือรากฐานสำคัญสำหรับยุคที่ระบบ
ส่วนใหญ่ประกอบขึ้นจากหลายบริการที่คุยกันผ่าน API (ไม่ว่าจะเป็น microservices ที่จะเรียนใน
Part 086 หรือการเปิด API ให้ partner ภายนอกใช้งานในธุรกิจจริง)

**ต่อไป (Part 061 — เปิด Phase 9: Background Jobs & Performance):** หลังจากใช้เวลาทั้ง
Phase 8 อยู่กับการตอบสนอง request แบบ synchronous (client รอผลลัพธ์ทันที) ถึงเวลาแล้วที่จะ
เรียนรู้การทำงานแบบ **asynchronous** — งานบางอย่าง (ส่งอีเมล, ประมวลผลไฟล์ขนาดใหญ่, เรียก
API ภายนอกที่ช้า) ไม่ควรทำให้ผู้ใช้ต้องรอ Part 061 จะแนะนำ **ActiveJob** เฟรมเวิร์กมาตรฐาน
ของ Rails สำหรับ background job พร้อม **adapter ต่างๆ** ที่ใช้ประมวลผลเบื้องหลัง
(`async`, `solid_queue` ที่ Rails 8 ติดตั้งให้เป็นค่า default อยู่แล้วตั้งแต่ `rails new`
ที่เราสร้างตลอดหลักสูตรนี้ — สังเกตไหมว่าไฟล์ `config/queue.yml` ปรากฏขึ้นมาทุกครั้งที่สร้าง
แอปใหม่?) ไปจนถึง **Sidekiq** ที่จะเจาะลึกเต็มรูปแบบใน Part 062 — จุดเริ่มต้นของการสร้างระบบ
ที่ตอบสนองเร็วและขยายขนาด (scale) ได้จริงในโลก production
