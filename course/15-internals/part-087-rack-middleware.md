# Part 087: Rack เบื้องลึก, Middleware ของ Rails

> **Step ครอบคลุมใน Part นี้:** Step 861–870
> **ระดับ:** Advanced
> **Ruby Version:** 3.3.6 | **Rails Version:** 8.1.4

ใน Part 086 เราได้เรียนรู้เรื่อง Microservices Architecture และ Message Queues ซึ่งช่วยให้เราแยกระบบใหญ่ออกเป็นบริการย่อยๆ และสื่อสารกันแบบ asynchronous ใน Part นี้เราจะดำดิ่งลงไปสู่รากฐานที่แท้จริงของ Rails นั่นคือ **Rack Protocol** และ **Middleware Stack** ซึ่งเป็นกลไกเบื้องหลังที่ทำให้ Rails และ web framework ภาษา Ruby ทุกตัวทำงานได้ การเข้าใจ Rack จะเปิดมุมมองใหม่ให้เราเห็นว่า request แต่ละตัวเดินทางผ่านชั้นต่างๆ อย่างไรก่อนถึง controller ของเรา

---

## สารบัญ

- [Step 861: Rack protocol คืออะไร](#step-861-rack-protocol-คืออะไร)
- [Step 862: Rack application ที่เรียบง่าย](#step-862-rack-application-ที่เรียบง่าย)
- [Step 863: Middleware Stack](#step-863-middleware-stack)
- [Step 864: Rails middleware stack](#step-864-rails-middleware-stack)
- [Step 865: เขียน custom middleware แรก](#step-865-เขียน-custom-middleware-แรก)
- [Step 866: เพิ่ม middleware ใน Rails](#step-866-เพิ่ม-middleware-ใน-rails)
- [Step 867: Middleware ที่มีประโยชน์จริง](#step-867-middleware-ที่มีประโยชน์จริง)
- [Step 868: Rack::Test สำหรับ test middleware โดยตรง](#step-868-racktest-สำหรับ-test-middleware-โดยตรง)
- [Step 869: Warden/Devise ทำงานผ่าน middleware](#step-869-wardendevise-ทำงานผ่าน-middleware)
- [Step 870: Metal controller vs Full Rails stack](#step-870-metal-controller-vs-full-rails-stack)

---

## Setup สำหรับ Part นี้

ก่อนเริ่ม Part นี้ ให้ตรวจสอบว่า Rails app ของเราพร้อมใช้งาน และมี gem ที่จำเป็น:

```bash
# ตรวจสอบ Ruby และ Rails version
ruby --version   # ruby 3.3.6
rails --version  # Rails 8.1.4

# สร้าง Rails app ใหม่สำหรับทดลอง (ถ้าไม่มี)
rails new rack_demo --skip-test

cd rack_demo
```

เพิ่ม gem ใน `Gemfile`:

```ruby
# Gemfile
gem 'rack-test', '~> 2.1'

group :development, :test do
  gem 'rspec-rails', '~> 6.1'
end
```

```bash
bundle install
```

---

## Step 861: Rack protocol คืออะไร

**Rack** คือ interface มาตรฐานระหว่าง web server (เช่น Puma, Unicorn, Thin) และ web application framework ภาษา Ruby (เช่น Rails, Sinatra, Hanami) มันเป็นข้อตกลงเล็กๆ ที่ทำให้ทั้งสองฝ่ายสื่อสารกันได้โดยไม่ต้องรู้จักกันโดยตรง

### หัวใจของ Rack: Interface เดียวที่ทุกอย่างต้องปฏิบัติตาม

Rack กำหนดว่า **Rack application** ทุกตัวต้องเป็น object ที่ตอบสนองต่อเมธอด `call` โดย:

- **รับ argument เดียว** คือ `env` ซึ่งเป็น Hash ที่มีข้อมูลทั้งหมดเกี่ยวกับ HTTP request
- **คืนค่า Array 3 ตัว** คือ `[status, headers, body]`

```
[status_code, headers_hash, body_enumerable]
```

ตัวอย่าง response ที่ valid:

```ruby
[200, { "Content-Type" => "text/plain" }, ["Hello, World!"]]
```

### ส่วนประกอบของ Response

| ส่วน | ชนิดข้อมูล | ตัวอย่าง |
|------|-----------|---------|
| status | Integer | `200`, `404`, `500` |
| headers | Hash (String keys) | `{ "Content-Type" => "text/html" }` |
| body | Enumerable ที่ respond to `each` | `["<html>...</html>"]`, IO object |

### env Hash มีอะไรบ้าง

`env` คือ Hash ที่ Rack-compliant web server สร้างขึ้นจาก HTTP request โดยมี key สำคัญดังนี้:

```ruby
# key มาตรฐานจาก CGI/Rack spec
env["REQUEST_METHOD"]   # "GET", "POST", "PUT", "DELETE"
env["PATH_INFO"]        # "/users/1"
env["QUERY_STRING"]     # "page=2&per=10"
env["HTTP_HOST"]        # "localhost:3000"
env["HTTP_ACCEPT"]      # "text/html,application/xhtml+xml"
env["CONTENT_TYPE"]     # "application/json"
env["CONTENT_LENGTH"]   # "42"
env["rack.input"]       # StringIO — body ของ request
env["rack.url_scheme"]  # "http" หรือ "https"
env["rack.errors"]      # IO สำหรับ error logging

# key ที่ Rails เพิ่มเติม
env["action_dispatch.request_id"]
env["warden"]           # Warden authentication object
```

### ความเชื่อมโยงกับ Rails

เมื่อ Puma (web server) ได้รับ HTTP request มันจะแปลง request นั้นเป็น `env` Hash และส่งให้กับ Rails application ผ่านเมธอด `call` จากนั้น Rails จะประมวลผลและคืน array `[status, headers, body]` กลับไปให้ Puma ส่งต่อเป็น HTTP response

```
Browser → HTTP Request → Puma → env Hash → Rails.call(env) → [200, {...}, [...]] → Puma → HTTP Response → Browser
```

Rails เองก็เป็น Rack application! คุณสามารถดูได้จากไฟล์ `config.ru` ที่ root ของทุก Rails project:

```ruby
# config.ru
require_relative "config/environment"

run Rails.application
Rails.application.load_server
```

`run Rails.application` คือการบอกว่า "ให้ Rails.application เป็น Rack app ที่จัดการ request ทั้งหมด"

---

## Step 862: Rack application ที่เรียบง่าย

มาเขียน Rack application ขนาดเล็กเพื่อทำความเข้าใจ interface อย่างลึกซึ้ง

### Rack App แบบ Lambda

วิธีที่ง่ายที่สุดในการสร้าง Rack app คือใช้ Lambda หรือ Proc เพราะมันตอบสนองต่อ `call` อยู่แล้ว:

```ruby
# my_app.rb
require 'rack'

# สร้าง Rack app ด้วย lambda
my_app = lambda do |env|
  # ดึงข้อมูลจาก env
  method  = env["REQUEST_METHOD"]
  path    = env["PATH_INFO"]

  # สร้าง response body
  body = "สวัสดี! คุณส่ง #{method} request มาที่ #{path}\n"

  # คืน [status, headers, body]
  [
    200,
    {
      "Content-Type"  => "text/plain; charset=utf-8",
      "Content-Length" => body.bytesize.to_s
    },
    [body]
  ]
end

# รัน app ด้วย Rack::Server
Rack::Server.start(app: my_app, Port: 9292)
```

### Rack App แบบ Class

ในทางปฏิบัติเรามักเขียนเป็น class เพื่อให้จัดการได้ง่ายกว่า:

```ruby
# hello_rack.rb
require 'rack'

class HelloRack
  def call(env)
    request = Rack::Request.new(env)

    case request.path
    when "/"
      body = "<h1>หน้าแรก</h1>"
      [200, { "Content-Type" => "text/html; charset=utf-8" }, [body]]

    when "/about"
      body = "<h1>เกี่ยวกับเรา</h1>"
      [200, { "Content-Type" => "text/html; charset=utf-8" }, [body]]

    else
      body = "<h1>404 - ไม่พบหน้านี้</h1>"
      [404, { "Content-Type" => "text/html; charset=utf-8" }, [body]]
    end
  end
end

run HelloRack.new
```

### สร้างไฟล์ config.ru

`config.ru` (อ่านว่า "config dot r-u" หรือ "Rack Up") คือไฟล์ที่ `rackup` command ใช้เปิด server:

```ruby
# config.ru
require_relative 'hello_rack'

# บอก Rack ว่าจะใช้ app อะไร
run HelloRack.new
```

```bash
# รัน Rack app
rackup config.ru

# หรือระบุ port
rackup config.ru -p 9292

# ทดสอบด้วย curl
curl http://localhost:9292/
curl http://localhost:9292/about
curl http://localhost:9292/not-exist
```

### ใช้ Rack::Response เพื่อสร้าง Response ง่ายขึ้น

```ruby
class HelloRack
  def call(env)
    request  = Rack::Request.new(env)
    response = Rack::Response.new

    response["Content-Type"] = "text/html; charset=utf-8"

    case request.path
    when "/"
      response.write "<h1>หน้าแรก</h1>"
    when "/json"
      response["Content-Type"] = "application/json"
      response.write '{"message": "สวัสดี JSON"}'
    else
      response.status = 404
      response.write "<h1>ไม่พบหน้านี้</h1>"
    end

    # finish คืน [status, headers, body]
    response.finish
  end
end
```

### Rack::Request helper methods

```ruby
def call(env)
  req = Rack::Request.new(env)

  req.get?          # true ถ้า GET request
  req.post?         # true ถ้า POST request
  req.path          # "/users/1"
  req.params        # Hash ของ query params + form data
  req.params["id"]  # "1"
  req.ip            # IP address ของผู้ส่ง
  req.user_agent    # "Mozilla/5.0 ..."
  req.xhr?          # true ถ้าเป็น AJAX request
  req.ssl?          # true ถ้าใช้ HTTPS
  req.body.read     # อ่าน request body (สำหรับ POST/PUT)
end
```

---

## Step 863: Middleware Stack

**Middleware** คือ Rack application ที่ "ห่อหุ้ม" application อื่น มันรับ `env` จาก server, ทำงานบางอย่าง, จากนั้นส่ง `env` ต่อไปยัง app ถัดไป และรับ response กลับมาเพื่อแก้ไขก่อนส่งคืน

### แนวคิด Middleware Chain

จินตนาการว่า request ของคุณต้องผ่านด่านตรวจหลายๆ ด่านก่อนถึงปลายทาง:

```
Request → [Middleware A] → [Middleware B] → [Middleware C] → [Your App]
                                                               ↓
Response ← [Middleware A] ← [Middleware B] ← [Middleware C] ←
```

แต่ละ middleware สามารถ:
1. **แก้ไข request** ก่อนส่งต่อ (เช่น เพิ่ม header, parse body)
2. **หยุด request** และคืน response เองเลย (เช่น authentication failed → 401)
3. **แก้ไข response** ก่อนส่งกลับ (เช่น เพิ่ม CORS header, compress body)
4. **บันทึก log** ทั้งขาไปและขากลับ

### โครงสร้างของ Middleware

```ruby
class SimpleMiddleware
  # รับ app ถัดไปใน chain ผ่าน constructor
  def initialize(app)
    @app = app
  end

  def call(env)
    # ---- ขาไป (before app) ----
    puts "Request เข้ามาแล้ว: #{env['REQUEST_METHOD']} #{env['PATH_INFO']}"

    # ส่งต่อให้ app ถัดไป
    status, headers, body = @app.call(env)

    # ---- ขากลับ (after app) ----
    puts "Response: #{status}"

    # คืนค่าต่อไป (อาจแก้ไขก็ได้)
    [status, headers, body]
  end
end
```

### Rack::Builder — เครื่องมือสร้าง Middleware Stack

`Rack::Builder` คือ DSL สำหรับประกอบ middleware stack เข้าด้วยกัน:

```ruby
# config.ru
require 'rack'
require_relative 'my_app'
require_relative 'middlewares'

# สร้าง app ด้วย Rack::Builder
app = Rack::Builder.new do
  # use เพิ่ม middleware (ลำดับบนสุด = ชั้นนอกสุด)
  use Rack::CommonLogger    # log ทุก request
  use Rack::ShowExceptions  # แสดง error อย่างสวยงาม
  use SimpleMiddleware       # middleware ของเราเอง

  # run กำหนด app ที่อยู่ในสุด
  run MyApp.new
end

run app
```

ลำดับของ `use` มีความสำคัญมาก — middleware ที่ประกาศก่อนจะอยู่ชั้นนอกสุด หมายความว่ามันรับ request มาก่อน และส่ง response คืนทีหลังสุด

### Rack::Builder DSL ใน config.ru แบบ shorthand

```ruby
# config.ru — Rack::Builder ทำงานโดยปริยายในไฟล์นี้
use Rack::CommonLogger
use Rack::ShowExceptions
use MyAuthentication

map "/api" do
  use RateLimiter
  run ApiApp.new
end

map "/" do
  run WebApp.new
end
```

### Middleware ที่มากับ Rack

Rack มี middleware สำเร็จรูปหลายตัว:

```ruby
use Rack::CommonLogger      # บันทึก request log ในรูปแบบ Apache
use Rack::ShowExceptions    # แสดง exception ได้อย่างสวยงาม
use Rack::Lint              # ตรวจสอบว่า app ปฏิบัติตาม spec
use Rack::Static,           # serve static files
    urls: ["/images", "/js"],
    root: "public"
use Rack::Deflater          # compress response ด้วย gzip
use Rack::ConditionalGet    # รองรับ HTTP caching (ETag, Last-Modified)
use Rack::ETag              # เพิ่ม ETag header อัตโนมัติ
use Rack::Session::Cookie,  # session ผ่าน cookie
    secret: "my_secret"
```

---

## Step 864: Rails middleware stack

Rails มี middleware stack ที่ซับซ้อนกว่า Rack ธรรมดา ประกอบด้วย middleware จำนวนมากที่ทำงานร่วมกัน

### ดู middleware stack ของ Rails

```bash
bundle exec rails middleware
```

ผลลัพธ์จะประมาณนี้:

```
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use ActionDispatch::ServerTiming
use ActiveSupport::Cache::Strategy::LocalCache::Middleware
use Rack::Runtime
use Rack::MethodOverride
use ActionDispatch::RequestId
use ActionDispatch::RemoteIp
use Sprockets::Rails::QuietAssets
use Rails::Rack::Logger
use ActionDispatch::ShowExceptions
use WebConsole::Middleware
use ActionDispatch::DebugExceptions
use ActionDispatch::ActionableExceptions
use ActionDispatch::Reloader
use ActionDispatch::Callbacks
use ActiveRecord::Migration::CheckPending
use ActionDispatch::Cookies
use ActionDispatch::Session::CookieStore
use ActionDispatch::Flash
use ActionDispatch::ContentSecurityPolicy::Middleware
use ActionDispatch::PermissionsPolicy::Middleware
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
use Rack::TempfileReaper
run MyApp::Application.routes
```

### ทำความรู้จักแต่ละ Middleware

**การรักษาความปลอดภัย:**
```
ActionDispatch::HostAuthorization
```
ตรวจสอบว่า Host header ตรงกับ `config.hosts` ที่กำหนดไว้ ป้องกัน DNS rebinding attacks

```
ActionDispatch::RemoteIp
```
ระบุ IP จริงของผู้ใช้จาก `X-Forwarded-For` header (เมื่ออยู่หลัง proxy/load balancer)

```
ActionDispatch::ContentSecurityPolicy::Middleware
```
เพิ่ม `Content-Security-Policy` header ตามที่กำหนดใน `config/initializers/content_security_policy.rb`

**การจัดการ Request/Response:**
```
Rack::Sendfile
```
ส่งไฟล์ผ่าน web server โดยตรง (เร็วกว่าผ่าน Ruby) เมื่อ response มี `X-Sendfile` header

```
ActionDispatch::Static
```
Serve ไฟล์ static จาก `public/` directory โดยไม่ต้องผ่าน Rails routing

```
Rack::MethodOverride
```
แปลง POST request ที่มี `_method` parameter (เช่น `_method=DELETE`) เป็น HTTP method ที่ถูกต้อง ทำให้ HTML form ส่ง PUT/PATCH/DELETE ได้

```
ActionDispatch::RequestId
```
สร้าง unique ID สำหรับทุก request และเก็บใน `X-Request-Id` header

**Session และ Cookie:**
```
ActionDispatch::Cookies
```
Parse และ serialize cookies

```
ActionDispatch::Session::CookieStore
```
เก็บ session data ใน encrypted cookie (default ของ Rails)

```
ActionDispatch::Flash
```
จัดการ flash messages (notice/alert ที่ข้ามระหว่าง request)

**Caching:**
```
ActiveSupport::Cache::Strategy::LocalCache::Middleware
```
ใช้ in-memory cache ต่อ request เพื่อหลีกเลี่ยงการ query cache store ซ้ำซ้อน

```
Rack::ConditionalGet
```
รองรับ HTTP conditional requests (`If-None-Match`, `If-Modified-Since`) คืน 304 เมื่อไม่มีการเปลี่ยนแปลง

```
Rack::ETag
```
เพิ่ม `ETag` header อัตโนมัติสำหรับ response ที่ไม่มี

**Logging และ Error Handling:**
```
Rails::Rack::Logger
```
เริ่ม request log และเพิ่ม tag ให้ logger

```
ActionDispatch::ShowExceptions
```
จัดการ exception ที่ไม่ถูก rescue และแสดง error page (ใช้ `public/500.html` ฯลฯ)

```
ActionDispatch::DebugExceptions
```
ใน development mode: แสดง exception พร้อม stack trace และ request info อย่างละเอียด

**Development helpers:**
```
ActionDispatch::Reloader
```
Reload code เมื่อมีการเปลี่ยนแปลงไฟล์ (development mode เท่านั้น)

```
ActiveRecord::Migration::CheckPending
```
ตรวจสอบว่ามี pending migration หรือไม่ ถ้ามีจะ raise error

---

## Step 865: เขียน custom middleware แรก

มาเขียน middleware จริงๆ ที่วัดเวลาของแต่ละ request และบันทึกลง log

### RequestTimingMiddleware

```ruby
# lib/middleware/request_timing_middleware.rb

class RequestTimingMiddleware
  def initialize(app)
    @app    = app
    @logger = Rails.logger
  end

  def call(env)
    # จับเวลาเริ่มต้น
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

    # ดึงข้อมูล request
    method = env["REQUEST_METHOD"]
    path   = env["PATH_INFO"]

    # ส่งต่อให้ middleware/app ถัดไป
    status, headers, body = @app.call(env)

    # คำนวณเวลาที่ใช้
    elapsed_ms = (
      (Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time) * 1000
    ).round(2)

    # บันทึก log
    @logger.info(
      "[RequestTiming] #{method} #{path} → #{status} (#{elapsed_ms}ms)"
    )

    # เพิ่ม header บอก client ว่าใช้เวลาเท่าไหร่
    headers["X-Processing-Time"] = "#{elapsed_ms}ms"

    [status, headers, body]
  end
end
```

### ทดสอบ Middleware ใน isolation

ก่อนเพิ่มใน Rails เราสามารถทดสอบ middleware ด้วย plain Ruby:

```ruby
# test/middleware/request_timing_middleware_test.rb
require 'test_helper'
require 'rack/test'

class RequestTimingMiddlewareTest < ActiveSupport::TestCase
  include Rack::Test::Methods

  def app
    # สร้าง simple app ที่ return 200
    inner_app = lambda do |env|
      [200, { "Content-Type" => "text/plain" }, ["OK"]]
    end

    # ห่อด้วย middleware ของเรา
    RequestTimingMiddleware.new(inner_app)
  end

  test "เพิ่ม X-Processing-Time header" do
    get "/"

    assert last_response.ok?
    assert last_response.headers.key?("X-Processing-Time")

    time_header = last_response.headers["X-Processing-Time"]
    assert_match(/\d+\.\d+ms/, time_header)
  end

  test "ไม่แก้ไข status code" do
    inner_app = lambda do |_env|
      [404, { "Content-Type" => "text/plain" }, ["Not Found"]]
    end
    app_with_middleware = RequestTimingMiddleware.new(inner_app)

    get "/"

    response = app_with_middleware.call(
      Rack::MockRequest.env_for("/")
    )
    assert_equal 404, response[0]
  end

  test "ไม่แก้ไข body" do
    get "/hello"

    assert_equal "OK", last_response.body
  end
end
```

---

## Step 866: เพิ่ม middleware ใน Rails

เมื่อเขียน middleware เสร็จแล้ว เราต้องบอก Rails ให้รู้จักและใช้มัน

### config.middleware.use

วิธีพื้นฐานที่สุดคือเพิ่มที่ท้าย stack:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # เพิ่มที่ท้าย middleware stack
    config.middleware.use RequestTimingMiddleware
  end
end
```

### insert_before และ insert_after

บางครั้งเราต้องการให้ middleware อยู่ในตำแหน่งที่เจาะจง:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # เพิ่มก่อน middleware ที่ระบุ
    config.middleware.insert_before ActionDispatch::RequestId,
                                    RequestTimingMiddleware

    # เพิ่มหลัง middleware ที่ระบุ
    config.middleware.insert_after Rails::Rack::Logger,
                                   RequestTimingMiddleware

    # เพิ่มที่ตำแหน่งที่ 0 (ชั้นนอกสุด)
    config.middleware.insert_before 0, RequestTimingMiddleware
  end
end
```

### delete — ลบ middleware ออก

Rails มี middleware บางตัวที่เราอาจไม่ต้องการ:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # ลบ middleware ออก (เช่น ถ้า handle session เองที่อื่น)
    config.middleware.delete ActionDispatch::Session::CookieStore

    # ลบและแทนที่ด้วยของเราเอง
    config.middleware.delete ActionDispatch::Session::CookieStore
    config.middleware.use ActionDispatch::Session::MemCacheStore
  end
end
```

### swap — แทนที่ middleware

```ruby
config.middleware.swap ActionDispatch::Session::CookieStore,
                       ActionDispatch::Session::ActiveRecordStore
```

### ตั้งค่า middleware ต่างกันตาม environment

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.middleware.use BetterErrors::Middleware, allow_ip: "127.0.0.1"
end

# config/environments/production.rb
Rails.application.configure do
  config.middleware.use Rack::Deflater  # compress ใน production เท่านั้น
end
```

### ส่ง options ให้ middleware

```ruby
# middleware ที่รับ option
class RateLimitMiddleware
  def initialize(app, options = {})
    @app       = app
    @limit     = options.fetch(:limit, 100)
    @window    = options.fetch(:window, 60)
    @store     = options.fetch(:store, {})
  end

  def call(env)
    # ...
  end
end

# config/application.rb
config.middleware.use RateLimitMiddleware,
                      limit: 200,
                      window: 60,
                      store: Rails.cache
```

### ตรวจสอบ stack หลังเพิ่ม middleware

```bash
bundle exec rails middleware
```

หรือใน Rails console:

```ruby
Rails.application.middleware.map(&:klass)
# => [ActionDispatch::HostAuthorization, Rack::Sendfile, ..., RequestTimingMiddleware, ...]
```

---

## Step 867: Middleware ที่มีประโยชน์จริง

### RateLimitMiddleware — จำกัดจำนวน request

```ruby
# lib/middleware/rate_limit_middleware.rb
class RateLimitMiddleware
  # ค่า default: 100 requests ต่อ 60 วินาที
  def initialize(app, limit: 100, window: 60)
    @app    = app
    @limit  = limit
    @window = window
    @store  = Hash.new { |h, k| h[k] = [] }
    @mutex  = Mutex.new
  end

  def call(env)
    ip = env["REMOTE_ADDR"] || env["HTTP_X_FORWARDED_FOR"]&.split(",")&.first&.strip

    if rate_limited?(ip)
      headers = {
        "Content-Type"    => "application/json",
        "Retry-After"     => @window.to_s,
        "X-RateLimit-Limit" => @limit.to_s
      }
      body = [{ error: "Rate limit exceeded. Please try again later." }.to_json]
      return [429, headers, body]
    end

    record_request(ip)

    status, headers, body = @app.call(env)

    # เพิ่ม header บอก remaining limit
    remaining = remaining_requests(ip)
    headers["X-RateLimit-Limit"]     = @limit.to_s
    headers["X-RateLimit-Remaining"] = [remaining, 0].max.to_s
    headers["X-RateLimit-Reset"]     = reset_time(ip).to_s

    [status, headers, body]
  end

  private

  def rate_limited?(ip)
    @mutex.synchronize do
      clean_old_requests(ip)
      @store[ip].size >= @limit
    end
  end

  def record_request(ip)
    @mutex.synchronize do
      @store[ip] << Time.now.to_f
    end
  end

  def remaining_requests(ip)
    @mutex.synchronize do
      @limit - @store[ip].size
    end
  end

  def reset_time(ip)
    @mutex.synchronize do
      oldest = @store[ip].min || Time.now.to_f
      (oldest + @window).to_i
    end
  end

  def clean_old_requests(ip)
    cutoff = Time.now.to_f - @window
    @store[ip].reject! { |time| time < cutoff }
  end
end
```

หมายเหตุ: ใน production ควรใช้ Redis เป็น store แทน in-memory Hash เพื่อให้ทำงานได้ข้ามหลาย process:

```ruby
# config/application.rb
config.middleware.insert_before ActionDispatch::RequestId,
                                RateLimitMiddleware,
                                limit: 1000,
                                window: 3600
```

### MaintenanceModeMiddleware — ระงับให้บริการชั่วคราว

```ruby
# lib/middleware/maintenance_mode_middleware.rb
class MaintenanceModeMiddleware
  MAINTENANCE_FILE = Rails.root.join("tmp", "maintenance.txt")
  EXEMPT_PATHS     = ["/health", "/admin/maintenance"].freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    path = env["PATH_INFO"]

    # ข้าม maintenance mode สำหรับ path ที่ยกเว้น
    if maintenance_mode? && !exempt_path?(path)
      message = maintenance_message
      headers = {
        "Content-Type"  => "text/html; charset=utf-8",
        "Retry-After"   => "3600",
        "Cache-Control" => "no-cache, no-store"
      }
      return [503, headers, [maintenance_page(message)]]
    end

    @app.call(env)
  end

  private

  def maintenance_mode?
    MAINTENANCE_FILE.exist?
  end

  def maintenance_message
    MAINTENANCE_FILE.exist? ? MAINTENANCE_FILE.read.strip : ""
  end

  def exempt_path?(path)
    EXEMPT_PATHS.any? { |exempt| path.start_with?(exempt) }
  end

  def maintenance_page(message)
    <<~HTML
      <!DOCTYPE html>
      <html lang="th">
      <head>
        <meta charset="UTF-8">
        <title>ปิดปรับปรุงชั่วคราว</title>
        <style>
          body { font-family: sans-serif; text-align: center; padding: 100px; }
          h1 { color: #e74c3c; }
        </style>
      </head>
      <body>
        <h1>ปิดปรับปรุงชั่วคราว</h1>
        <p>ขณะนี้ระบบกำลังปิดปรับปรุง กรุณากลับมาใหม่ในภายหลัง</p>
        #{message.present? ? "<p><em>#{CGI.escapeHTML(message)}</em></p>" : ""}
        <p>ขออภัยในความไม่สะดวก</p>
      </body>
      </html>
    HTML
  end
end
```

การใช้งาน:

```bash
# เปิด maintenance mode
touch tmp/maintenance.txt
echo "กำลัง upgrade database ประมาณ 30 นาที" > tmp/maintenance.txt

# ปิด maintenance mode
rm tmp/maintenance.txt
```

```ruby
# config/application.rb
config.middleware.insert_before 0, MaintenanceModeMiddleware
```

### LocaleMiddleware — ตั้ง locale จาก subdomain

```ruby
# lib/middleware/locale_middleware.rb
class LocaleMiddleware
  LOCALE_SUBDOMAINS = {
    "th"  => :th,
    "en"  => :en,
    "ja"  => :ja,
    "www" => :en
  }.freeze

  def initialize(app)
    @app = app
  end

  def call(env)
    host      = env["HTTP_HOST"] || ""
    subdomain = host.split(".").first.to_s.downcase

    locale = LOCALE_SUBDOMAINS.fetch(subdomain, I18n.default_locale)

    # เก็บ locale ใน env เพื่อให้ Rails controller อ่านได้
    env["app.locale"] = locale

    # ตั้ง I18n.locale สำหรับ request นี้
    original_locale = I18n.locale
    I18n.locale     = locale

    status, headers, body = @app.call(env)

    # คืน locale กลับ (thread safety)
    I18n.locale = original_locale

    [status, headers, body]
  end
end
```

---

## Step 868: Rack::Test สำหรับ test middleware โดยตรง

`Rack::Test` ช่วยให้เราทดสอบ Rack app และ middleware โดยตรง โดยไม่ต้องผ่าน Rails test stack ทั้งหมด ทำให้ test เร็วและ focused กว่า

### Setup

```ruby
# Gemfile
gem 'rack-test', '~> 2.1'
```

```ruby
# test/test_helper.rb หรือสร้างไฟล์แยก
require 'rack/test'
```

### ทดสอบ Middleware โดยตรง

```ruby
# test/middleware/rate_limit_middleware_test.rb
require 'test_helper'
require 'rack/test'

class RateLimitMiddlewareTest < ActiveSupport::TestCase
  include Rack::Test::Methods

  # กำหนด app ที่จะใช้ใน test
  def app
    # inner app ที่ return 200 เสมอ
    inner = lambda do |_env|
      [200, { "Content-Type" => "text/plain" }, ["OK"]]
    end

    # ห่อด้วย middleware — limit = 3 เพื่อให้ test ได้เร็ว
    RateLimitMiddleware.new(inner, limit: 3, window: 60)
  end

  test "request ปกติผ่านได้" do
    get "/"
    assert_equal 200, last_response.status
    assert_equal "OK", last_response.body
  end

  test "เพิ่ม RateLimit headers" do
    get "/"
    assert last_response.headers["X-RateLimit-Limit"]
    assert last_response.headers["X-RateLimit-Remaining"]
    assert last_response.headers["X-RateLimit-Reset"]
  end

  test "X-RateLimit-Remaining ลดลงทุก request" do
    get "/"
    first_remaining = last_response.headers["X-RateLimit-Remaining"].to_i

    get "/"
    second_remaining = last_response.headers["X-RateLimit-Remaining"].to_i

    assert_equal first_remaining - 1, second_remaining
  end

  test "ส่ง 429 เมื่อเกิน rate limit" do
    # ส่ง request จนเกิน limit (3 ครั้ง)
    3.times { get "/" }

    # ครั้งที่ 4 ควร blocked
    get "/"
    assert_equal 429, last_response.status
    assert last_response.headers.key?("Retry-After")
  end

  test "body ตอน 429 เป็น JSON" do
    4.times { get "/" }

    assert_equal "application/json",
                 last_response.content_type.split(";").first
    parsed = JSON.parse(last_response.body)
    assert parsed["error"].present?
  end
end
```

### Rack::MockRequest สำหรับ test แบบ low-level

```ruby
# ใช้ Rack::MockRequest โดยตรงเมื่อต้องการควบคุมมากกว่า
class MaintenanceModeMiddlewareTest < ActiveSupport::TestCase
  def setup
    inner = lambda do |_env|
      [200, { "Content-Type" => "text/plain" }, ["App is running"]]
    end
    @app = MaintenanceModeMiddleware.new(inner)
  end

  test "ผ่านได้เมื่อไม่อยู่ใน maintenance mode" do
    env    = Rack::MockRequest.env_for("/")
    status, _headers, body = @app.call(env)

    assert_equal 200, status
    assert_equal "App is running", body.join
  end

  test "คืน 503 เมื่ออยู่ใน maintenance mode" do
    # สร้างไฟล์ maintenance
    maintenance_file = Rails.root.join("tmp", "maintenance.txt")
    FileUtils.touch(maintenance_file)

    begin
      env    = Rack::MockRequest.env_for("/some/page")
      status, headers, body = @app.call(env)

      assert_equal 503, status
      assert_equal "text/html; charset=utf-8", headers["Content-Type"]
      assert body.join.include?("ปิดปรับปรุงชั่วคราว")
    ensure
      # ลบไฟล์หลัง test
      maintenance_file.delete if maintenance_file.exist?
    end
  end

  test "ยกเว้น /health จาก maintenance mode" do
    maintenance_file = Rails.root.join("tmp", "maintenance.txt")
    FileUtils.touch(maintenance_file)

    begin
      env    = Rack::MockRequest.env_for("/health")
      status, _headers, _body = @app.call(env)

      assert_equal 200, status
    ensure
      maintenance_file.delete if maintenance_file.exist?
    end
  end
end
```

### ทดสอบ middleware chain

```ruby
class MiddlewareChainTest < ActiveSupport::TestCase
  include Rack::Test::Methods

  def app
    inner = lambda do |_env|
      [200, { "Content-Type" => "text/plain" }, ["Final Response"]]
    end

    # ประกอบ middleware stack ด้วย Rack::Builder
    Rack::Builder.new do
      use RequestTimingMiddleware
      use RateLimitMiddleware, limit: 100
      run inner
    end
  end

  test "ทั้ง middleware ทำงานร่วมกัน" do
    get "/"

    assert last_response.ok?
    assert last_response.headers.key?("X-Processing-Time")
    assert last_response.headers.key?("X-RateLimit-Remaining")
  end
end
```

---

## Step 869: Warden/Devise ทำงานผ่าน middleware

**Warden** คือ authentication framework สำหรับ Rack ที่ Devise ใช้เป็นรากฐาน การเข้าใจ Warden ช่วยให้เราเข้าใจว่า authentication ของ Devise ทำงานอย่างไรจริงๆ ที่ middleware layer

### Warden ทำงานอย่างไร

Warden เป็น middleware ที่เพิ่ม `warden` object เข้าไปใน `env`:

```ruby
# Warden middleware ทำสิ่งนี้โดยประมาณ
class Warden::Manager
  def initialize(app, &block)
    @app     = app
    @config  = Warden::Config.new
    block.call(@config) if block
  end

  def call(env)
    # สร้าง Warden::Proxy และเก็บใน env
    env["warden"] = Warden::Proxy.new(env, self)

    begin
      result = @app.call(env)
    rescue Warden::NotAuthenticated => e
      # จัดการเมื่อ authentication ล้มเหลว
      process_unauthenticated(env, e)
    end
  end
end
```

### Authentication Flow ใน Middleware Layer

```
Request
  ↓
[Warden::Manager] — สร้าง env["warden"] = Warden::Proxy
  ↓
[Devise::SessionsController หรือ Controller อื่นๆ]
  ↓
  controller เรียก current_user
    ↓
    Devise::Controllers::Helpers#current_user
      ↓
      warden.authenticate(scope: :user)
        ↓
        Warden::Proxy#authenticate
          ↓
          ตรวจสอบ session ["warden.user.user.key"]
          ถ้าไม่มี → ลอง Strategies ต่างๆ
            - DatabaseAuthenticatable (username+password)
            - Rememberable (remember_me cookie)
            - TokenAuthenticatable (API token)
```

### เจาะลึก Warden Strategy

```ruby
# app/strategies/jwt_strategy.rb
class JwtStrategy < Warden::Strategies::Base
  def valid?
    # strategy นี้ valid เมื่อมี Authorization header
    env["HTTP_AUTHORIZATION"].present?
  end

  def authenticate!
    token = extract_token

    begin
      payload = JWT.decode(token, Rails.application.secret_key_base).first
      user    = User.find_by(id: payload["user_id"])

      if user
        success!(user)  # authentication สำเร็จ, เก็บ user
      else
        fail!("ไม่พบผู้ใช้")  # authentication ล้มเหลว
      end
    rescue JWT::DecodeError => e
      fail!("Token ไม่ถูกต้อง: #{e.message}")
    end
  end

  private

  def extract_token
    env["HTTP_AUTHORIZATION"].to_s.gsub(/^Bearer\s+/, "")
  end
end

# ลงทะเบียน strategy
Warden::Strategies.add(:jwt, JwtStrategy)
```

### ตั้งค่า Warden ใน Devise Initializer

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.warden do |manager|
    # เพิ่ม strategy ใหม่
    manager.default_strategies(scope: :user).unshift(:jwt)

    # กำหนดว่าจะทำอะไรเมื่อ authentication ล้มเหลว
    manager.failure_app = CustomFailureApp
  end
end
```

### Custom Failure App

```ruby
# app/authentication/custom_failure_app.rb
class CustomFailureApp < Devise::FailureApp
  def respond
    if request.format == :json
      json_failure_response
    else
      super
    end
  end

  private

  def json_failure_response
    self.status        = 401
    self.content_type  = "application/json"
    self.response_body = [{
      error:   "Unauthorized",
      message: i18n_message
    }.to_json]
  end
end
```

### เข้าถึง Warden โดยตรงใน Middleware

```ruby
class ApiKeyMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    # เข้าถึง warden ที่ middleware ก่อนหน้าเพิ่มไว้
    warden = env["warden"]

    # ตรวจสอบ API key จาก header
    api_key = env["HTTP_X_API_KEY"]

    if api_key.present?
      user = ApiKey.find_by(key: api_key)&.user

      if user
        # authenticate user ใน Warden โดยตรง
        warden.set_user(user, scope: :user)
      end
    end

    @app.call(env)
  end
end
```

### ดู Warden ใน Rails Console

```ruby
# Rails console
# simulate request environment
env = Rack::MockRequest.env_for("/")

# ดู warden scope
app = Rails.application
app.call(env)  # หลัง call env["warden"] จะมี object

# ใน request จริง
class UsersController < ApplicationController
  def debug_auth
    render json: {
      warden_user:  warden.user,
      current_user: current_user,
      authenticated: warden.authenticated?(:user)
    }
  end

  private

  def warden
    request.env["warden"]
  end
end
```

---

## Step 870: Metal controller vs Full Rails stack

เมื่อต้องการ performance สูงสุด `ActionController::Metal` ให้เราเขียน controller ที่ข้ามชั้น Rails หนาๆ ไปได้ โดยยังคงความสะดวกของ routing

### Full Rails Stack vs Metal Controller

```
Full Stack:
Request → Middleware Stack → Router → AbstractController → Base
          → ActionController::Base (Views, Helpers, Cookies, Flash, ...)
          → Controller Action

Metal Stack:
Request → Middleware Stack → Router → ActionController::Metal
          → Controller Action (เร็วกว่า ~40% ต่อ request)
```

### ActionController::Metal พื้นฐาน

```ruby
# app/controllers/fast_api_controller.rb
class FastApiController < ActionController::Metal
  # include เฉพาะ module ที่ต้องการจริงๆ
  include ActionController::Rendering
  include ActionController::Renderers::All
  include ActionController::ConditionalGet
  include ActionController::StrongParameters

  # routes ใช้งานได้ตามปกติ
  def index
    # เข้าถึง request และ response โดยตรง
    self.response_body = [{ message: "สวัสดีจาก Metal!" }.to_json]
    self.content_type  = "application/json"
    self.status        = 200
  end

  def show
    user = User.find_by(id: params[:id])

    if user
      render json: { id: user.id, name: user.name }
    else
      self.status        = 404
      self.response_body = [{ error: "ไม่พบผู้ใช้" }.to_json]
    end
  end
end
```

### ใช้ routes ตามปกติ

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # routing สำหรับ Metal controller เหมือนกันทุกอย่าง
  resources :fast_api, only: [:index, :show]

  # หรือ route ตรงๆ
  get "/health", to: "health#check"
end
```

### เพิ่ม module ตามต้องการ

Metal controller เริ่มต้นจากน้อยมาก เราเลือก include เฉพาะที่ต้องการ:

```ruby
class ApiController < ActionController::Metal
  # Rendering
  include ActionController::Rendering          # render json: ...
  include ActionController::MimeResponds       # respond_to

  # Request handling
  include ActionController::StrongParameters   # params.require(...)
  include ActionController::Helpers            # helper methods

  # Authentication
  include ActionController::HttpAuthentication::Token::ControllerMethods
  include ActionController::HttpAuthentication::Basic::ControllerMethods

  # Callbacks
  include AbstractController::Callbacks        # before_action
  include ActionController::Rescue            # rescue_from

  # Caching
  include ActionController::Caching           # caches_action

  before_action :authenticate!

  def index
    render json: { data: "protected data" }
  end

  private

  def authenticate!
    token = request.headers["Authorization"]&.gsub(/^Bearer\s+/, "")
    user  = User.find_by(api_token: token)

    unless user
      render json: { error: "Unauthorized" }, status: :unauthorized
    end
  end
end
```

### เปรียบเทียบ Performance

```ruby
# benchmark_test.rb
require 'benchmark'
require 'rack/test'

# Full stack response time
n = 10_000
Benchmark.bm(20) do |x|
  x.report("ActionController::Base:") do
    # test ผ่าน full Rails stack
    # ~2-3ms per request
  end

  x.report("ActionController::Metal:") do
    # test ผ่าน Metal
    # ~1-1.5ms per request (~40% เร็วกว่า)
  end
end
```

### Rack App โดยตรงใน Routes

สำหรับ endpoint ที่ต้องการความเร็วสูงสุด เราสามารถ mount Rack app โดยตรงใน routes:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Mount Rack app โดยตรง — เร็วที่สุด
  mount HealthCheckApp, at: "/health"
  mount MetricsApp,     at: "/metrics"

  # หรือใช้ lambda
  get "/ping", to: lambda { |_env|
    [200, { "Content-Type" => "text/plain" }, ["pong"]]
  }
end
```

```ruby
# app/rack/health_check_app.rb
class HealthCheckApp
  def self.call(env)
    status = database_healthy? ? 200 : 503
    body   = { status: status == 200 ? "ok" : "degraded" }.to_json

    [status, { "Content-Type" => "application/json" }, [body]]
  end

  def self.database_healthy?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue StandardError
    false
  end
end
```

### เมื่อไหรควรใช้ Metal vs Base

| สถานการณ์ | ใช้อะไร |
|-----------|---------|
| Endpoint ที่ call บ่อยมาก (health check, ping) | Rack app โดยตรง |
| API endpoint ที่ต้องการความเร็ว | ActionController::Metal |
| Feature ครบๆ (views, helpers, flash, cookies) | ActionController::Base |
| Admin panel, user-facing pages | ActionController::Base |
| Internal microservice endpoint | ActionController::Metal |
| Webhook receiver | ActionController::Metal + rescue_from |

### ทดสอบ Metal Controller

```ruby
# test/controllers/fast_api_controller_test.rb
require 'test_helper'

class FastApiControllerTest < ActionDispatch::IntegrationTest
  test "index คืน JSON" do
    get fast_api_index_url

    assert_response :success
    assert_equal "application/json", response.content_type.split(";").first

    data = JSON.parse(response.body)
    assert data["message"]
  end

  test "show คืน user data" do
    user = users(:one)

    get fast_api_url(user)

    assert_response :success
    data = JSON.parse(response.body)
    assert_equal user.id, data["id"]
  end

  test "show คืน 404 เมื่อไม่พบ" do
    get fast_api_url(id: 99999)

    assert_response :not_found
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Rack App จากศูนย์

สร้าง standalone Rack application ที่:
- ตอบสนองต่อ `GET /` ด้วยหน้า HTML ธรรมดา
- ตอบสนองต่อ `GET /time` ด้วยเวลาปัจจุบันในรูปแบบ JSON
- ตอบสนองต่อ `GET /health` ด้วย `{ "status": "ok" }` และ status 200
- คืน 404 สำหรับ path อื่นๆ

```ruby
# solution_app.rb
require 'rack'
require 'json'

class SolutionApp
  def call(env)
    request = Rack::Request.new(env)

    case request.path
    when "/"
      # TODO: คืน HTML response
    when "/time"
      # TODO: คืน JSON response พร้อมเวลาปัจจุบัน
    when "/health"
      # TODO: คืน health check response
    else
      # TODO: คืน 404
    end
  end
end

run SolutionApp.new
```

### แบบฝึกหัดที่ 2: Logging Middleware

เขียน middleware ที่:
- บันทึก method, path, status, และเวลาที่ใช้
- บันทึก request body (ถ้าเป็น POST/PUT/PATCH) โดยตัด log ให้สั้นกว่า 500 ตัวอักษร
- เพิ่ม header `X-Request-Id` ที่เป็น UUID สำหรับทุก request (ถ้ายังไม่มี)

```ruby
# lib/middleware/enhanced_logging_middleware.rb
class EnhancedLoggingMiddleware
  def initialize(app)
    @app    = app
    @logger = Rails.logger
  end

  def call(env)
    # TODO: implement
  end

  private

  def generate_request_id
    # TODO: สร้าง UUID
  end

  def truncate_body(body_io)
    # TODO: อ่านและตัด body
  end
end
```

### แบบฝึกหัดที่ 3: Feature Flag Middleware

เขียน middleware สำหรับ Feature Flags ที่:
- อ่าน feature flags จาก `Rails.cache` หรือ environment variable
- ถ้า feature flag `"maintenance_v2"` เปิด ให้ redirect `/` ไป `/maintenance`
- ถ้า feature flag `"api_v2"` เปิด ให้ rewrite request path จาก `/api/v1/...` เป็น `/api/v2/...`
- เพิ่ม header `X-Feature-Flags` บอก flags ที่เปิดอยู่

```ruby
# lib/middleware/feature_flag_middleware.rb
class FeatureFlagMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    flags = active_flags

    # TODO: ตรวจสอบและจัดการ feature flags
  end

  private

  def active_flags
    # TODO: ดึง flags จาก cache หรือ ENV
  end
end
```

### แบบฝึกหัดที่ 4: Metal Controller สำหรับ Webhook

เขียน Metal controller สำหรับรับ GitHub webhook ที่:
- ตรวจสอบ HMAC signature จาก `X-Hub-Signature-256` header
- Parse JSON body
- Enqueue background job ตาม event type
- คืน 200 เมื่อ OK, 401 เมื่อ signature ไม่ถูกต้อง, 422 เมื่อ event ไม่รู้จัก

```ruby
# app/controllers/webhooks/github_controller.rb
module Webhooks
  class GithubController < ActionController::Metal
    include ActionController::Rescue
    include AbstractController::Callbacks

    before_action :verify_signature!

    def receive
      # TODO: implement
    end

    private

    def verify_signature!
      # TODO: ตรวจสอบ HMAC-SHA256
    end

    def payload
      @payload ||= JSON.parse(request.body.read)
    rescue JSON::ParseError
      render status: :unprocessable_entity
      nil
    end
  end
end
```

---

## สรุปสิ่งที่ได้เรียนรู้

ใน Part นี้เราได้เรียนรู้รากฐานที่สำคัญที่สุดอย่างหนึ่งของ Ruby web development:

### Rack Protocol (Step 861-862)
- Rack คือ interface มาตรฐานที่เชื่อม web server กับ framework
- ทุก Rack app ต้องตอบสนองต่อ `call(env)` และคืน `[status, headers, body]`
- `env` คือ Hash ที่มีข้อมูลทั้งหมดของ HTTP request
- Rails เองก็เป็น Rack application

### Middleware Stack (Step 863-864)
- Middleware คือ Rack app ที่ห่อหุ้ม app อื่น ประมวลผลทั้งขาเข้าและขาออก
- `Rack::Builder` ใช้ DSL `use`/`run` ประกอบ middleware stack
- Rails มี middleware stack มาให้พร้อมกว่า 20 ตัว ครอบคลุมงานทั่วไปทั้งหมด
- ลำดับของ middleware สำคัญมาก — ชั้นนอกสุดประมวลผลก่อนสุด

### Custom Middleware (Step 865-867)
- Middleware เขียนง่าย: `initialize(app)` + `call(env)` ก็พอ
- เพิ่มใน Rails ด้วย `config.middleware.use`, `insert_before`, `insert_after`, `delete`
- ใช้งานจริงได้กับ rate limiting, maintenance mode, locale detection ฯลฯ

### Testing (Step 868)
- `Rack::Test` ช่วยทดสอบ middleware โดยตรงโดยไม่ต้องผ่าน Rails stack ทั้งหมด
- `Rack::MockRequest.env_for` สร้าง env จำลองสำหรับ unit test
- ทดสอบ middleware chain ด้วย `Rack::Builder` ใน test

### Authentication Layer (Step 869)
- Warden คือ authentication middleware ที่ Devise ใช้
- `env["warden"]` มีอยู่ใน middleware ทุกตัวหลัง Warden
- สามารถเขียน custom Warden Strategy สำหรับ authentication แบบพิเศษได้
- Failure app ควบคุมพฤติกรรมเมื่อ authentication ล้มเหลว

### Performance (Step 870)
- `ActionController::Metal` เร็วกว่า `ActionController::Base` ประมาณ 40%
- เลือก include เฉพาะ module ที่ต้องการ
- Mount Rack app โดยตรงใน routes สำหรับ endpoint ที่ต้องการเร็วสุด
- ใช้ Metal สำหรับ API endpoint, webhook receiver, health check

---

## Preview: Part 088

ใน **Part 088: Rails Engine และ Mountable Apps** เราจะเรียนรู้การสร้างและใช้งาน Rails Engine ซึ่งเป็นวิธีการแบ่ง Rails application ออกเป็นส่วนย่อยๆ ที่สามารถนำกลับมาใช้ใหม่ได้ เนื้อหาจะครอบคลุม:

- Rails Engine คืออะไร และต่างจาก gem อย่างไร
- Full Engine vs Mountable Engine
- เขียน Engine แรกพร้อม models, controllers, views
- Isolate namespace เพื่อหลีกเลี่ยง naming conflict
- Share models และ helpers ระหว่าง Engine และ main app
- Test Engine อย่างถูกต้อง
- ตัวอย่าง engine จริง: Devise, ActiveAdmin, Spree
- Deploy Engine เป็น gem สำหรับใช้ข้ามโปรเจกต์

การเข้าใจ Engine จะช่วยให้เราสร้าง modular applications และ shared libraries ได้อย่างมืออาชีพ
