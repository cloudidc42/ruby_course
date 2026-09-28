# Part 072: Omniauth หลาย Provider, Action Mailer, และ SMS/Notification Integration — ปิด Phase 11

> **Step ครอบคลุมใน Part นี้:** Step 711–720
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน **Part 045** เรื่อง JWT/Omniauth เบื้องต้นมาก่อน — Part นี้
> จะ recap แบบเร็วแล้วต่อยอดทันที ไม่สอนซ้ำตั้งแต่ต้น — และ **Part 061** เรื่อง ActiveJob
> เบื้องต้น สำหรับหัวข้อ `deliver_later`)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem `omniauth` 2.1.4, `omniauth-google-oauth2`
> 1.2.3, `omniauth-github` 2.0.1, `omniauth-rails_csrf_protection` 1.0.2, `letter_opener` 1.10.0,
> `premailer-rails` 1.12.0, `twilio-ruby` 7.11.2, `jwt` 2.10.x, `bcrypt` 3.1.x (ทดสอบจริงบน
> Ruby 3.3.6, Rails 8.1.4)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> เช่นเดียวกับ Part 071 ทีมผู้เขียนหลักสูตรสร้างแอป Rails 8.1.4 จริงในแซนด์บ็อกซ์ (แยกจาก
> repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้งหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) เพื่อพิสูจน์ทุกอย่าง
> ที่ทำได้จริงแทนการเดาจากเอกสารอย่างเดียว แบ่งชัดเจนดังนี้:
>
> - **ทดสอบจริง 100% ผ่าน Rack/HTTP จริง (ไม่ใช่แค่เรียก method เปล่าๆ):** การ login ผ่าน
>   Omniauth ทั้งสอง provider (Google, GitHub) โดยใช้ `OmniAuth.config.test_mode` +
>   `OmniAuth.config.mock_auth` ยิง HTTP request จริงผ่าน Puma server ที่รันอยู่จริงไปที่
>   `/auth/:provider/callback` เพื่อพิสูจน์ว่า middleware, controller, และ database ทำงาน
>   ร่วมกันถูกต้องจริง ไม่ใช่แค่จำลอง `OmniAuth::AuthHash` แล้วเรียก method ตรงๆ เหมือน Part 045
>   — Part นี้ไปไกลกว่านั้นอีกขั้นด้วยการทดสอบ **flow การเชื่อมบัญชีที่สอง** (link) แบบเต็ม
>   รูปแบบ ผ่าน session cookie + CSRF token จริง และ**ค้นพบบั๊กจริงระหว่างทดสอบ**ที่จะอธิบาย
>   ละเอียดใน Step 714 (ไม่ใช่ปัญหาที่แต่งขึ้นเพื่อสอน — เป็นสิ่งที่เกิดขึ้นจริงตอนเขียน Part นี้)
> - **ทดสอบจริง 100% ด้วย Action Mailer จริง:** สร้าง `OrderMailer` จริง, generate view
>   `.html.erb`/`.text.erb` จริง, ตั้งค่า `delivery_method = :letter_opener` จริง แล้วเรียก
>   `deliver_now`/`deliver_later` จริง — ยืนยันได้ว่าไฟล์ HTML preview ถูกเขียนลงดิสก์จริงที่
>   `tmp/letter_opener/`, หน้า `/rails/mailers` (preview UI) ตอบกลับ HTTP 200 จริงพร้อมเนื้อหา
>   อีเมลจริง, และ `premailer-rails` แปลง `<style>` block เป็น inline `style="..."` ให้อัตโนมัติ
>   จริงโดยไม่ต้องตั้งค่าเพิ่มเลยแม้แต่บรรทัดเดียว
> - **ทดสอบจริง 100% เรื่อง `deliver_later`:** ยืนยันว่า adapter เริ่มต้นของ Rails 8 ใน
>   development คือ `:async`, `deliver_later` enqueue `ActionMailer::MailDeliveryJob` จริง,
>   และสั่งให้ job ที่ค้างอยู่ทำงานจริงแล้วอีเมลถูกส่งผ่าน letter_opener จริงเช่นเดียวกับ
>   `deliver_now`
> - **ตรวจสอบโครงสร้าง code จริง แต่เชื่อมต่อ network จริงไม่ได้:** `twilio-ruby` gem
>   `require` ได้จริง, สร้าง `Twilio::REST::Client` จริง, เรียก `client.messages.create(...)`
>   ด้วย signature ที่ถูกต้องจริงตาม API ของ gem เวอร์ชัน 7.11.2 — แต่แซนด์บ็อกซ์นี้ไม่มีบัญชี
>   Twilio จริง และ network policy ปฏิเสธการเชื่อมต่อออกไปยัง `api.twilio.com` ด้วย `403` ที่
>   ระดับ proxy (รูปแบบเดียวกับที่ Part 071 พบกับ `api.stripe.com` ทุกประการ) — ยืนยันได้แค่ว่า
>   "โค้ดเขียนถูกไวยากรณ์และเรียก method ถูกต้อง" ไม่ใช่ "ส่ง SMS จริงสำเร็จ"
> - **มาจากเอกสารทางการ ไม่ได้ทดสอบจริง (จะระบุไว้ชัดเจนตรงจุด):** การตั้งค่า Google
>   Cloud Console/GitHub Developer Settings เพื่อขอ Client ID/Secret จริง, หน้าตาของหน้า consent
>   ที่ Google/GitHub แสดงให้ผู้ใช้เห็นจริง, การส่ง SMS จริงผ่านบัญชี Twilio จริง, และ Firebase
>   Cloud Messaging (FCM) สำหรับ push notification — ส่วนนี้อธิบายตามเอกสารทางการเท่านั้น

ยินดีต้อนรับสู่ **Part สุดท้ายของ Phase 11: Payment & Third-party Integration** Part 071 ปิดฉาก
เรื่องการรับเงินด้วย Stripe ไปแล้ว Part นี้จะปิด Phase ด้วยการเชื่อมต่อกับบริการภายนอกอีกสามกลุ่ม
ที่เว็บแอปพลิเคชันสมัยใหม่แทบทุกตัวต้องมี: **การล็อกอินผ่านบัญชีคนอื่น (OAuth หลาย provider)**,
**การส่งอีเมลจริง (Action Mailer)**, และ **การแจ้งเตือนผ่านช่องทางอื่นนอกเหนือจากอีเมล (SMS/push)**
ทั้งสามเรื่องมีเส้นด้ายที่ร้อยเรียงกันคือแนวคิด **"multi-channel"** — ผู้ใช้คนเดียวอาจเข้าสู่ระบบ
ได้หลายทาง (email/password, Google, GitHub) และรับการแจ้งเตือนได้หลายช่องทาง (email, SMS, push,
in-app) พร้อมกัน ระบบที่ออกแบบดีต้องรองรับความหลากหลายนี้ได้อย่างเป็นระเบียบตั้งแต่ต้น ไม่ใช่ผูก
ทุกอย่างไว้กับวิธีเดียว

## สารบัญของ Part นี้

- Step 711: ทบทวน Omniauth จาก Part 045 แบบเร็ว + ทำไมต้องรองรับหลาย Provider
- Step 712: เพิ่ม Provider ที่สอง (GitHub) — `OmniAuth::Builder`, unified
  `OmniauthCallbacksController`, และ routing แบบไดนามิก
- Step 713: โมเดลข้อมูลสำหรับหลาย Identity — `Identity` model, `find_or_create_from_omniauth`,
  ทดสอบ login ผ่าน Rack จริงทั้งสอง provider
- Step 714: เชื่อมบัญชีที่สองเข้ากับ User เดิม (Link) — session, CSRF, และบั๊กจริงที่ค้นพบระหว่าง
  ทดสอบ
- Step 715: Action Mailer พื้นฐาน — `rails generate mailer`, มัลติพาร์ต HTML/text,
  `mail(to:, subject:)`
- Step 716: Preview อีเมลในเครื่อง — `ActionMailer::Preview` และหน้า `/rails/mailers`
- Step 717: ตั้งค่า Delivery Method — `:test`, `:letter_opener`, `:smtp` และรูปแบบ production
  จริง (SendGrid/Postmark/SES)
- Step 718: ส่งอีเมลจริงด้วย `letter_opener` + `deliver_later` ผ่าน ActiveJob (ทบทวน Part 061)
- Step 719: Layout และ Styling ของอีเมล — inline CSS และ `premailer-rails`
- Step 720: SMS/Push Notification แบบ Multi-channel — Twilio, Firebase Cloud Messaging, และ
  `NotificationService` abstraction + แบบฝึกหัดปิดท้าย Phase 11

---

## Step 711: ทบทวน Omniauth จาก Part 045 แบบเร็ว + ทำไมต้องรองรับหลาย Provider

### ทบทวนแบบรวบรัด (ไม่สอนซ้ำ — อ่านเต็มได้ที่ Part 045 Step 448–449)

Part 045 สอนไว้แล้วว่า **OAuth 2.0** คือ protocol สำหรับ**การให้สิทธิ์ (authorization)** ไม่ใช่
การยืนยันตัวตนโดยตรง และ **Omniauth** คือ gem ที่ห่อหุ้ม Authorization Code Flow ทั้งหมดไว้ให้
ทำงานเป็น **Rack middleware** ที่ดักจับ `/auth/:provider` (request phase — redirect ไป provider)
และ `/auth/:provider/callback` (callback phase — รับผลกลับมา) โดยอัตโนมัติ เราตั้งค่าไว้แบบนี้:

```ruby
# config/initializers/omniauth.rb (ทบทวนจาก Part 045 Step 449)
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
           ENV["GOOGLE_CLIENT_ID"],
           ENV["GOOGLE_CLIENT_SECRET"],
           scope: "email,profile"
end
```

แล้วเขียน `User.find_or_create_from_omniauth(auth)` ผูกกับ `OmniauthCallbacksController#google_oauth2`
ที่ออก JWT ของระบบตัวเองต่อ — ครบ flow "Sign in with Google" หนึ่ง provider

> **หมายเหตุเรื่อง syntax:** บางเอกสารและบาง gem (โดยเฉพาะ `devise` ร่วมกับ
> `devise-omniauthable`) ใช้รูปแบบ `config.omniauth :google_oauth2, ...` เขียนอยู่ภายใน
> `Devise.setup do |config| ... end` — นั่นเป็น DSL เฉพาะของ Devise ที่ห่อ `provider` ของ
> Omniauth ไว้อีกชั้นหนึ่ง Part นี้ (เหมือน Part 045) **ไม่ได้ใช้ Devise** สำหรับ flow นี้ จึงยัง
> คงใช้ `provider :xxx, ...` ภายใน `OmniAuth::Builder` ตรงๆ ต่อไป เพื่อความสอดคล้องกับสิ่งที่
> เรียนมาแล้ว — ถ้าโปรเจกต์จริงของท่านใช้ Devise อยู่แล้ว (ทบทวนจาก Part 042) ก็สลับไปใช้
> `config.omniauth` ภายใต้ `Devise.setup` แทนได้ หลักการเบื้องหลังเหมือนกันทุกประการ

### ทำไมระบบจริงแทบทุกระบบต้องรองรับมากกว่า 1 Provider

ลองนึกภาพผู้ใช้จริงของระบบ: บางคนมีแต่ Gmail, บางคนใช้ GitHub เป็นหลักเพราะเป็นนักพัฒนา, บางคน
ไม่อยากผูกกับ Google เลยด้วยเหตุผลความเป็นส่วนตัว และบางคนสมัครด้วย email/password ธรรมดามาก่อน
Omniauth ถูกออกแบบมาให้รองรับ **provider หลายตัวพร้อมกันได้ตั้งแต่แรก** — สถาปัตยกรรมของมันแยก
"strategy" ของแต่ละ provider ออกจากกันอย่างชัดเจน (`omniauth-google-oauth2`,
`omniauth-github`, `omniauth-facebook`, ฯลฯ ล้วนเป็น gem แยกที่ implement interface เดียวกัน)
งานของเราจึงเหลือแค่ **เพิ่มบรรทัด `provider` อีกบรรทัด** แล้วเขียน controller ให้รองรับทั้งสอง
แบบ "unified" (ไม่ต้องแยก action ทีละ provider เหมือนที่ Part 045 ทำไว้)

### ทำไมเลือก GitHub เป็น Provider ที่สอง แทน Facebook

โจทย์ทั่วไปมักยกตัวอย่าง "Google + Facebook" แต่ Part นี้เลือก **GitHub** แทน Facebook ด้วย
เหตุผลเชิงปฏิบัติที่สำคัญ: **Facebook Login ตั้งแต่ปี 2018 เป็นต้นมา บังคับให้แอปต้องผ่านกระบวนการ
App Review ของ Facebook ก่อนจึงจะขอ scope ใดๆ นอกเหนือจาก public profile ได้ในโหมด production
จริง** (แม้แต่ `email` scope พื้นฐาน) ทำให้ทดสอบและสอนได้ยากกว่ามาก ในขณะที่ **GitHub OAuth App**
สร้างและใช้งานได้ทันทีโดยไม่ต้องผ่านการ review ใดๆ เหมาะกับการเรียนรู้และเหมาะกับกลุ่มผู้ใช้ที่
เป็นนักพัฒนาด้วย (ตรงกับกลุ่มเป้าหมายของหลักสูตรนี้พอดี) หลักการที่เรียนใน Part นี้ใช้กับ Facebook
ได้ทุกประการเพียงแค่เปลี่ยน gem เป็น `omniauth-facebook` และเปลี่ยนชื่อ provider เท่านั้น

---

## Step 712: เพิ่ม Provider ที่สอง (GitHub) — `OmniAuth::Builder`, unified `OmniauthCallbacksController`, และ routing แบบไดนามิก

### ติดตั้ง Gem

```ruby
# Gemfile
gem "omniauth", "~> 2.1"
gem "omniauth-google-oauth2", "~> 1.2"
gem "omniauth-github", "~> 2.0"
gem "omniauth-rails_csrf_protection", "~> 1.0"
```

```bash
bundle install
```

### ลงทะเบียนทั้งสอง Provider ใน `OmniAuth::Builder`

```ruby
# config/initializers/omniauth.rb
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
           ENV["GOOGLE_CLIENT_ID"],
           ENV["GOOGLE_CLIENT_SECRET"],
           scope: "email,profile"

  provider :github,
           ENV["GITHUB_CLIENT_ID"],
           ENV["GITHUB_CLIENT_SECRET"],
           scope: "user:email"
end

OmniAuth.config.allowed_request_methods = [:post]
OmniAuth.config.silence_get_warning = true
```

**อธิบาย:** แค่เพิ่มบรรทัด `provider :github, ...` เข้าไปอีกบล็อกหนึ่งเท่านั้น ทดสอบรันจริงยืนยัน
ว่า `OmniAuth::Builder` block รองรับหลาย `provider` เรียงต่อกันได้โดยไม่ชนกัน — Omniauth จะดักจับ
ทั้ง `/auth/google_oauth2`, `/auth/google_oauth2/callback`, `/auth/github`, และ
`/auth/github/callback` แยกกันตามชื่อ provider ให้อัตโนมัติจากบรรทัดเดียวกันนี้ ไม่ต้องตั้งค่า
routing เพิ่มเติมสำหรับ path เหล่านี้เอง (Omniauth middleware ดักจับก่อนที่ request จะไปถึง
`config/routes.rb` ด้วยซ้ำ)

### `OmniauthCallbacksController` แบบ Unified — Action เดียวรับทุก Provider

Part 045 เขียนแยก action ตามชื่อ provider (`def google_oauth2`) เพราะมีแค่ provider เดียว พอมี
สอง provider ขึ้นไป การเขียน action แยกจะเริ่มซ้ำซ้อน (โค้ดข้างในแทบเหมือนกันทุกตัวอักษร) วิธีที่
ถูกต้องกว่าคือรวมเป็น **action เดียว** ที่ทำงานเหมือนกันไม่ว่า provider ไหนจะเรียกเข้ามา:

```ruby
# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < ApplicationController
  # ทั้ง Google และ GitHub วิ่งเข้า action นี้ action เดียว เพราะทั้งสอง provider ทำสิ่งเดียวกัน
  # เป๊ะ: แปลง OmniAuth::AuthHash เป็น User แล้วออก JWT ของระบบเราเอง — ต่างกันแค่ "ชื่อ provider"
  # เท่านั้น ซึ่ง Omniauth normalize รูปแบบ auth hash ให้เหมือนกันอยู่แล้วไม่ว่า provider ไหน
  def callback
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_from_omniauth(auth)
    token = JsonWebToken.encode({ user_id: user.id })

    render json: {
      token: token,
      provider: auth.provider, # เท่ากับ params[:provider] เสมอ (มาจาก URL segment เดียวกัน)
      user: { id: user.id, email: user.email, name: user.name }
    }
  end

  def failure
    render json: { error: "เข้าสู่ระบบผ่าน OAuth ไม่สำเร็จ: #{params[:message]}" }, status: :unauthorized
  end
end
```

### `config/routes.rb`

```ruby
get "auth/:provider/callback", to: "omniauth_callbacks#callback"
get "auth/failure", to: "omniauth_callbacks#failure"
```

**อธิบาย:**

- `:provider` ใน route คือ **dynamic segment** (ทบทวนจาก Part 022) — Rails จะจับค่าจาก URL มา
  ใส่ใน `params[:provider]` ให้อัตโนมัติ ดังนั้นทั้ง `GET /auth/google_oauth2/callback` และ
  `GET /auth/github/callback` วิ่งเข้า route เดียวกันและ action เดียวกันนี้เสมอ — นี่คือ
  ความหมายของคำว่า "unified controller handling both via `params[:provider]`"
- ในทางปฏิบัติ เราไม่ได้ใช้ `params[:provider]` โดยตรงในโค้ด (แม้จะดึงมาใช้ได้) เพราะ
  `auth.provider` (จาก `OmniAuth::AuthHash`) เชื่อถือได้มากกว่า — มันมาจากตัว strategy ของ
  Omniauth เองที่ประมวลผล callback เสร็จเรียบร้อยแล้ว ไม่ใช่แค่ค่า string ดิบจาก URL ที่ยังไม่
  ผ่านการตรวจสอบใดๆ แต่ทั้งสองค่าจะตรงกันเสมอในทางปฏิบัติเพราะมาจาก URL segment เดียวกัน
- **route เดียว รองรับทุก provider ที่ลงทะเบียนไว้ใน `OmniAuth::Builder`** — เพิ่ม provider ตัว
  ที่สาม (เช่น Facebook) ในอนาคตจะไม่ต้องแก้ route หรือ controller เลยแม้แต่บรรทัดเดียว เพราะ
  `:provider` เป็น wildcard อยู่แล้ว

### ทดสอบจริงผ่าน HTTP จริง — Login ได้ทั้งสอง Provider

ทดสอบด้วยการเปิด `OmniAuth.config.test_mode = true` แล้วกำหนด `OmniAuth.config.mock_auth` ให้
ทั้งสอง provider (เทคนิคนี้จำลองผลลัพธ์ของ Omniauth strategy ที่คุยกับ provider จริงเสร็จแล้ว
โดยไม่ต้องมี Client ID/Secret จริงเลย — เป็นวิธีทดสอบ integration แบบมาตรฐานที่ Omniauth ออกแบบ
มาให้ใช้ใน test suite จริงของโปรเจกต์):

```ruby
# config/initializers/omniauth.rb (เพิ่มเฉพาะตอนทดสอบในแซนด์บ็อกซ์นี้ — ไม่ควรมีใน production)
if Rails.env.development? && ENV["OMNIAUTH_TEST_MODE"] == "1"
  OmniAuth.config.test_mode = true
  OmniAuth.config.mock_auth[:google_oauth2] = OmniAuth::AuthHash.new(
    provider: "google_oauth2",
    uid: "mock-google-uid-1",
    info: OmniAuth::AuthHash::InfoHash.new(email: "mockgoogle@example.com", name: "Mock Google User")
  )
  OmniAuth.config.mock_auth[:github] = OmniAuth::AuthHash.new(
    provider: "github",
    uid: "mock-github-uid-1",
    info: OmniAuth::AuthHash::InfoHash.new(email: "mockgithub@example.com", name: "Mock GitHub User")
  )
end
```

รัน server จริงแล้วยิง `curl` ไปที่ callback path ตรงๆ (ในโหมด test_mode, Omniauth ข้ามการ
redirect ไป provider จริง และเติม `request.env["omniauth.auth"]` จาก mock ให้ทันที):

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}\n" "http://127.0.0.1:3199/auth/google_oauth2/callback"
```

```json
{"token":"eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo1LCJleHAiOjE3OTA2MjAxMzJ9.s5Mbp6loshCkUIEClYdCJfu3MlA5Hvwl5y1la7VAwRo","provider":"google_oauth2","user":{"id":5,"email":"mockgoogle@example.com","name":"Mock Google User"}}
HTTP_STATUS:200
```

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}\n" "http://127.0.0.1:3199/auth/github/callback"
```

```json
{"token":"eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo2LCJleHAiOjE3OTA2MjAxMzN9.4FWaEgI75gJwgIJT8iyESI79TUB2uFBJE74IHZsVkD4","provider":"github","user":{"id":6,"email":"mockgithub@example.com","name":"Mock GitHub User"}}
HTTP_STATUS:200
```

ทดสอบรันจริงยืนยันว่า **route เดียว + controller เดียว** รองรับทั้งสอง provider ได้ถูกต้อง
สมบูรณ์ ผ่าน HTTP request จริงบน Puma server จริง (ไม่ใช่แค่เรียก Ruby method เปล่าๆ) — สร้าง
user คนละคนกัน (id=5 กับ id=6) และออก JWT ที่ถูกต้องให้ทั้งคู่

---

## Step 713: โมเดลข้อมูลสำหรับหลาย Identity — `Identity` model, `find_or_create_from_omniauth`

### ปัญหาของการออกแบบ `User` แบบ Part 045: ผูก `provider`/`uid` ไว้ในตาราง `users` ตรงๆ

Part 045 เพิ่มคอลัมน์ `provider`/`uid` ลงในตาราง `users` โดยตรง วิธีนี้ใช้ได้ดีตราบใดที่ **ผู้ใช้
หนึ่งคนมีได้แค่ provider เดียว** แต่ทันทีที่ต้องการให้ผู้ใช้คนเดียวเชื่อมได้ทั้ง Google และ GitHub
พร้อมกัน (โจทย์ของ Part นี้) การออกแบบแบบนั้นใช้ไม่ได้อีกต่อไป เพราะคอลัมน์ `provider`/`uid` ใน
`users` เก็บได้แค่คู่เดียวต่อแถว

### ทางออก: แยกตาราง `identities` ออกมาต่างหาก (has_many)

```bash
bin/rails g model Identity user:references provider:string uid:string
bin/rails g migration AddUniqueIndexToIdentities
```

```ruby
# db/migrate/..._add_unique_index_to_identities.rb
class AddUniqueIndexToIdentities < ActiveRecord::Migration[8.1]
  def change
    add_index :identities, [:provider, :uid], unique: true
  end
end
```

```bash
bin/rails db:migrate
```

```ruby
# app/models/identity.rb
class Identity < ApplicationRecord
  belongs_to :user
end
```

**อธิบาย:**

- `has_many :identities` (ฝั่ง `User`) ทำให้ผู้ใช้คนเดียวเชื่อมได้กี่ provider ก็ได้ ไม่จำกัดที่ 1
  — นี่คือความแตกต่างสำคัญที่สุดจากการออกแบบของ Part 045
- **`unique index` บน `[:provider, :uid]` คู่กัน** คือหัวใจของความถูกต้องของระบบทั้งหมด — มันการันตี
  ว่าบัญชี Google/GitHub บัญชีหนึ่งๆ (ระบุด้วย `uid` ที่ provider ออกให้) **ผูกกับ `User` ของเรา
  ได้แค่คนเดียวเท่านั้น** ป้องกันไม่ให้สอง `User` ในระบบเราอ้างว่าเป็นเจ้าของบัญชี Google เดียวกัน
  พร้อมกัน (จะเห็นว่า constraint นี้สำคัญแค่ไหนตอนทดสอบ Step 714)

### `User` Model — `find_or_create_from_omniauth` สำหรับ Login

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  has_many :identities, dependent: :destroy
  has_many :orders, dependent: :destroy

  validates :email, presence: true, uniqueness: true

  # เข้าสู่ระบบผ่าน OAuth: ถ้าเคย login ด้วย provider+uid นี้มาก่อน คืน user เดิม
  # ถ้ายังไม่เคย ให้ลองหา user จาก email ก่อน (เผื่อเคยสมัครด้วยรหัสผ่านมาก่อน) แล้วค่อย
  # สร้าง Identity ใหม่ผูกเข้ากับ user นั้น ถ้าไม่เจอเลยจริงๆ ค่อยสร้าง user ใหม่ทั้งหมด
  def self.find_or_create_from_omniauth(auth)
    identity = Identity.find_by(provider: auth.provider, uid: auth.uid)
    return identity.user if identity

    user = find_by(email: auth.info.email) || create!(
      email: auth.info.email,
      name: auth.info.name,
      password: SecureRandom.hex(32)
    )

    user.identities.create!(provider: auth.provider, uid: auth.uid)
    user
  end
end
```

**อธิบาย — จุดที่ลึกกว่า Part 045 อย่างชัดเจน:**

- Part 045 ใช้ `find_or_create_by!(provider:, uid:)` ตรงบนตาราง `users` — สั้นกว่าแต่รองรับได้
  แค่ 1 provider ต่อคน
- โค้ดใหม่นี้แยก 3 ขั้นตอนชัดเจน: **(1)** หา `Identity` ที่ตรงกันก่อน (ครอบคลุมเคส "login ซ้ำ
  ด้วย provider เดิม") **(2)** ถ้าไม่เจอ ให้หา `User` จาก email ก่อน (ครอบคลุมเคส "เคยสมัครด้วย
  provider อื่น หรือด้วยรหัสผ่านมาก่อน แล้ว email ตรงกัน") **(3)** ถ้าไม่เจอเลยจริงๆ ค่อยสร้าง
  `User` ใหม่ทั้งหมด — ลำดับความสำคัญนี้ทำให้ระบบ **ไม่สร้าง user ซ้ำโดยไม่จำเป็น** เมื่อผู้ใช้
  คนเดิมสลับมา login ด้วยอีก provider ที่ใช้ email เดียวกัน
- **ข้อควรระวังเรื่องความปลอดภัย (ทบทวนหลักการจาก Part 045 Step 449):** การ auto-link ผ่าน
  email แบบนี้ (ขั้นตอนที่ 2) มีความเสี่ยง **account takeover** ถ้า provider ใดไม่ verify ความ
  เป็นเจ้าของ email จริงก่อนส่งกลับมา (Google และ GitHub verify เสมอ จึงปลอดภัยในทางปฏิบัติ
  แต่ provider บางเจ้าที่ verify ไม่เข้มงวดอาจเปิดช่องให้คนอื่นสมัคร email ปลอมแล้วเข้าควบคุม
  บัญชีของเราได้) ควรตรวจสอบเอกสารของแต่ละ provider ว่า scope ที่ขอ (`email`) รับประกันว่าเป็น
  verified email เสมอหรือไม่ ก่อนนำ pattern นี้ไปใช้จริงกับ provider ที่ไม่คุ้นเคย

### ทดสอบจริง: Multi-provider Linking ทำงานถูกต้องครบทุกเคส

```ruby
require "omniauth"

google_auth = OmniAuth::AuthHash.new(
  provider: "google_oauth2", uid: "google-uid-111",
  info: OmniAuth::AuthHash::InfoHash.new(email: "somchai@example.com", name: "Somchai")
)
github_auth = OmniAuth::AuthHash.new(
  provider: "github", uid: "github-uid-222",
  info: OmniAuth::AuthHash::InfoHash.new(email: "somchai@example.com", name: "Somchai G.")
)

user1 = User.find_or_create_from_omniauth(google_auth)
user1_again = User.find_or_create_from_omniauth(google_auth)
user_github = User.find_or_create_from_omniauth(github_auth) # email เดียวกับ Google ข้างบน
```

ผลการทดสอบจริง (รันผ่าน `bin/rails runner`):

```
== Case 1: user ใหม่ login ด้วย Google ครั้งแรก ==
user id=1, email=somchai@example.com, identities=[["google_oauth2", "google-uid-111"]]

== Case 2: login ด้วย Google อีกครั้ง (ต้องได้ user เดิม ไม่สร้างซ้ำ) ==
same record? true
total users so far: 1

== Case 4: user เดียวกัน login ด้วย GitHub ด้วย (คนละ provider แต่ email เดียวกับ Google) ==
same user as Google login (matched by email)? true
user1 identities count = 2
  - google_oauth2: google-uid-111
  - github: github-uid-222

== Case 5: unique index ป้องกัน identity ซ้ำ (provider+uid เดิม ผูกกับ user อื่นไม่ได้) ==
PASSED ถูกปฏิเสธตามคาด: ActiveRecord::RecordNotUnique
```

ยืนยันครบทุกกรณีสำคัญ: login ซ้ำด้วย provider เดิมไม่สร้าง user ซ้ำ, login ด้วย provider ที่สอง
ที่ email ตรงกันจะถูก **auto-link** เข้ากับ user เดิมโดยอัตโนมัติ (ไม่ใช่สร้าง user ใหม่แยก),
และ unique index ป้องกัน identity เดียวกันถูกผูกกับ user คนละคนได้จริงระดับฐานข้อมูล

---

## Step 714: เชื่อมบัญชีที่สองเข้ากับ User เดิม (Link) — session, CSRF, และบั๊กจริงที่ค้นพบระหว่างทดสอบ

Step 713 ครอบคลุมเคส "auto-link ผ่าน email ที่ตรงกันพอดี" แต่โจทย์ที่พบบ่อยกว่าคือ: **ผู้ใช้ที่
login อยู่แล้ว (ด้วยวิธีไหนก็ได้) ต้องการกดปุ่ม "เชื่อมบัญชี Google เพิ่ม" จากหน้าตั้งค่าบัญชีของ
ตัวเอง** โดยไม่สนใจว่า email ของ Google จะตรงกับ email ที่ใช้ login อยู่หรือไม่ — นี่คือ pattern
ที่ต่างจาก find-or-create โดยสิ้นเชิง เพราะเรา**รู้อยู่แล้วว่า user คนไหนคือคนที่กำลังจะเชื่อม**
ไม่ต้องเดาจาก email

### ปัญหาที่ต้องแก้ก่อน: Callback URL เดียวกัน ต้องรู้ว่า "login" หรือ "link"

Omniauth ส่งผลลัพธ์กลับมาที่ **callback URL เดียวกันเสมอ** (`/auth/:provider/callback`) ไม่ว่า
จะเริ่ม flow มาจากปุ่ม "login ด้วย Google" หรือปุ่ม "เชื่อมบัญชี Google เพิ่ม" — วิธีมาตรฐานที่ใช้
แก้ปัญหานี้คือ **เก็บ "เจตนา" ไว้ใน session ก่อนเริ่ม flow** แล้วอ่านค่านั้นกลับมาตอน callback:

```ruby
# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < ApplicationController
  def callback
    auth = request.env["omniauth.auth"]

    if (linking_user_id = session.delete(:linking_user_id))
      handle_link(linking_user_id, auth)
    else
      handle_login(auth)
    end
  end

  # เริ่ม flow การ "เชื่อมบัญชีเพิ่ม" — ต้องเรียกก่อน redirect ไปยัง POST /auth/:provider เสมอ
  # เก็บ user.id ของผู้ใช้ที่ login อยู่แล้วไว้ใน session (คนละก้อนกับ JWT ที่ใช้เรียก API ปกติ)
  def link_start
    user = current_user_from_bearer_token
    session[:linking_user_id] = user.id
    render json: { message: "พร้อมเชื่อมบัญชี — เรียก POST /auth/:provider ต่อได้เลย (ต้องแนบ CSRF token)" }
  end

  def failure
    render json: { error: "เข้าสู่ระบบผ่าน OAuth ไม่สำเร็จ: #{params[:message]}" }, status: :unauthorized
  end

  private

  def handle_login(auth)
    user = User.find_or_create_from_omniauth(auth)
    token = JsonWebToken.encode({ user_id: user.id })
    render json: { token: token, provider: auth.provider, user: { id: user.id, email: user.email } }
  end

  def handle_link(linking_user_id, auth)
    user = User.find(linking_user_id)
    identity = user.link_omniauth!(auth)
    render json: { message: "เชื่อมบัญชี #{identity.provider} สำเร็จแล้ว", provider: identity.provider }
  rescue ActiveRecord::RecordNotUnique
    render json: { error: "บัญชี #{auth.provider} นี้ถูกเชื่อมกับผู้ใช้คนอื่นไปแล้ว" }, status: :unprocessable_entity
  end

  def current_user_from_bearer_token
    token = request.headers["Authorization"]&.split(" ")&.last
    payload = JsonWebToken.decode(token)
    User.find(payload[:user_id])
  end
end
```

**ทำไมต้องใช้ `session` ไม่ใช่แค่ query parameter:** อาจดูเหมือนส่ง `?intent=link&user_id=5` ติด
ไปกับ URL ง่ายกว่า แต่ **ห้ามทำแบบนั้นเด็ดขาด** เพราะ query parameter ที่เราส่งไปเองจะไม่ถูกส่ง
กลับมาโดย provider เสมอไป (Google/GitHub ควบคุม `redirect_uri` เองทั้งหมด ไม่รับประกันว่าจะคง
query string เดิมไว้ให้) และที่ร้ายแรงกว่านั้นคือ **ถ้าใครก็ตามเดา/ปลอมแปลง `user_id` ใน URL ได้
จะเชื่อมบัญชี OAuth ของตัวเองเข้ากับบัญชีคนอื่นในระบบได้ทันที** — session ที่ผูกกับ cookie ของ
browser ฝั่งเราเองเท่านั้นที่ปลอดภัยสำหรับเก็บ "เจตนา" แบบนี้

### CSRF Token สำหรับ Request Phase ใน API-only Mode

`omniauth-rails_csrf_protection` (ทบทวนจาก Part 045) บังคับให้ `POST /auth/:provider` (request
phase) ต้องแนบ CSRF token ที่ถูกต้องเสมอ — ปกติแอปที่ render HTML จะมี view helper
(`button_to "เชื่อม Google", "/auth/google_oauth2", method: :post`) แทรก token ให้อัตโนมัติ แต่
แอป API-only ไม่มี view เหล่านี้ จึงต้องมี endpoint แยกที่คืนค่า token ให้ client (เช่น SPA) นำไป
แนบเป็น header `X-CSRF-Token` เอง:

```ruby
# app/controllers/csrf_tokens_controller.rb
class CsrfTokensController < ApplicationController
  include ActionController::RequestForgeryProtection

  def show
    render json: { csrf_token: form_authenticity_token }
  end
end
```

```ruby
# config/routes.rb
get "csrf_token", to: "csrf_tokens#show"
post "auth/:provider/link_start", to: "omniauth_callbacks#link_start"
```

**อธิบาย:** `ActionController::API` (base class ของ API-only controller) **ไม่ include**
`ActionController::RequestForgeryProtection` โดย default (ต่างจาก `ActionController::Base`) จึง
เรียก `form_authenticity_token` ตรงๆ ไม่ได้ ต้อง `include` module นี้เองก่อน — ทดสอบรันจริงแล้วว่า
วิธีนี้ทำงานถูกต้อง โดยไม่ต้องเปลี่ยน `ApplicationController` ทั้งแอปให้กลายเป็น
`ActionController::Base`

### บั๊กจริงที่ค้นพบระหว่างทดสอบ: `find_or_create_by!` กับ Association Scope

ตอนเขียน `User#link_omniauth!` ครั้งแรก เราเขียนแบบตรงไปตรงมาที่สุด:

```ruby
# ❌ เวอร์ชันแรกที่ดูเหมือนถูกต้อง แต่มีบั๊กที่พบจากการทดสอบจริงเท่านั้น
def link_omniauth!(auth)
  identities.find_or_create_by!(provider: auth.provider, uid: auth.uid)
end
```

ทดสอบ happy path (เชื่อม provider ใหม่ที่ไม่มีใครใช้มาก่อน) ผ่านฉลุย แต่พอทดสอบเคส **"พยายาม
เชื่อมบัญชี Google ที่ถูกคนอื่นเชื่อมไปแล้ว"** ผ่าน HTTP จริงแบบเต็มรูปแบบ (session + CSRF +
redirect จริง) กลับได้ error ที่ไม่คาดคิด:

```
ActiveRecord::RecordNotFound: Couldn't find Identity with
[WHERE "identities"."user_id" = ? AND "identities"."provider" = ? AND "identities"."uid" = ?]
```

**สาเหตุ (เข้าใจได้จากการอ่าน source ของ `find_or_create_by!` ใน ActiveRecord จริง):**
`find_or_create_by!` ที่เรียกผ่าน association (`identities.find_or_create_by!`) เมื่อพยายาม
`create!` แล้วชนกับ **unique index** ที่ตั้งไว้ใน Step 713 (เพราะ `provider`+`uid` นี้มีอยู่แล้ว
จริง แต่เป็นของ `user` **คนอื่น**) ActiveRecord จะ rescue `ActiveRecord::RecordNotUnique` แล้ว
ลองค้นหาใหม่อีกครั้งด้วย `find_by!` — แต่การค้นหารอบสองนี้**ยัง scope อยู่ภายใต้ association เดิม**
(คือ `WHERE user_id = <user คนปัจจุบัน>`) ทำให้หา record ที่แท้จริงเป็นของ user อื่นไม่เจอ แล้ว
`find_by!` จึง raise `RecordNotFound` แทนที่จะบอกตรงๆ ว่า "ชนกับ record ของคนอื่น"

**วิธีแก้ที่ถูกต้อง — ตรวจสอบล่วงหน้าเองแทนการพึ่งพากลไก race-condition ของ Rails:**

```ruby
# ✅ เวอร์ชันที่แก้แล้ว
def link_omniauth!(auth)
  existing = Identity.find_by(provider: auth.provider, uid: auth.uid)

  if existing && existing.user_id != id
    raise ActiveRecord::RecordNotUnique, "บัญชี #{auth.provider} นี้ถูกเชื่อมกับผู้ใช้คนอื่นไปแล้ว"
  end

  identities.find_or_create_by!(provider: auth.provider, uid: auth.uid)
end
```

**ทดสอบจริงยืนยันว่าแก้ถูกต้องครบทุกเคส:**

```
== user_a เชื่อม Google เข้ากับตัวเองก่อน ==
user_a identities: [["google_oauth2", "shared-uid-999"]]

== user_b พยายามเชื่อม Google ตัวเดียวกัน (uid ซ้ำ) ควรถูกปฏิเสธด้วย error message ที่ชัดเจน ==
PASSED: บัญชี google_oauth2 นี้ถูกเชื่อมกับผู้ใช้คนอื่นไปแล้ว

== user_a เรียกเชื่อม Google ตัวเดิมซ้ำอีกครั้ง (idempotent, ไม่ error, ไม่สร้างซ้ำ) ==
PASSED: true, identities count = 1
```

และทดสอบผ่าน HTTP เต็มรูปแบบอีกครั้ง (session cookie + CSRF token + redirect จริงผ่าน
`OmniAuth.config.test_mode`) ก็ได้ผลลัพธ์ `{"error":"บัญชี google_oauth2 นี้ถูกเชื่อมกับผู้ใช้
คนอื่นไปแล้ว"}` ตรงตามที่ออกแบบไว้ทุกประการ ไม่ใช่ error 404/500 ที่สับสนอีกต่อไป

> **บทเรียนสำคัญที่สุดของ Step นี้:** ปัญหานี้**ไม่มีทางพบได้จากการอ่านเอกสารเฉยๆ** —
> `find_or_create_by!` ทำงานถูกต้องสมบูรณ์แบบในทุกตัวอย่างที่เอกสารทั่วไปแสดง (ซึ่งมักไม่มี
> unique index ร่วมกับ association scope พร้อมกัน) บั๊กแบบนี้โผล่มาเฉพาะตอนองค์ประกอบสามอย่าง
> มาเจอกันพอดี: **(1)** unique index ระดับฐานข้อมูล **(2)** เรียกผ่าน association ที่มี scope
> **(3)** record ที่ชนกันเป็นของ scope อื่นจริงๆ — นี่คือเหตุผลว่าทำไม Part นี้ (และหลักสูตรนี้
> ทั้งหมด) ยืนยันด้วยการทดสอบรันจริงเสมอ แทนที่จะเขียนโค้ดตามความจำหรือเอกสารอย่างเดียว

---

## Step 715: Action Mailer พื้นฐาน — `rails generate mailer`, มัลติพาร์ต HTML/text, `mail(to:, subject:)`

เปลี่ยนหัวข้อจาก "ใครคือฉัน" (authentication) มาเป็น "ฉันจะบอกอะไรผู้ใช้บ้าง" (notification) —
**Action Mailer** คือ framework ของ Rails สำหรับส่งอีเมล มีสถาปัตยกรรมคล้าย Controller/View
มาก (mailer class เปรียบได้กับ controller, view ของ mailer เปรียบได้กับ view ปกติ)

### Generate Mailer

```bash
bin/rails g mailer OrderMailer confirmation
```

```
create  app/mailers/order_mailer.rb
invoke  erb
create    app/views/order_mailer
create    app/views/order_mailer/confirmation.text.erb
create    app/views/order_mailer/confirmation.html.erb
```

ไฟล์ที่ generator สร้างให้ (ทดสอบรันจริงแล้ว — นี่คือเนื้อหาจริงที่ Rails 8.1.4 generate ให้):

```ruby
# app/mailers/order_mailer.rb (ก่อนแก้)
class OrderMailer < ApplicationMailer
  # Subject can be set in your I18n file at config/locales/en.yml
  # with the following lookup:
  #
  #   en.order_mailer.confirmation.subject
  #
  def confirmation
    @greeting = "Hi"

    mail to: "to@example.org"
  end
end
```

**อธิบาย:**

- `bin/rails g mailer OrderMailer confirmation` ระบุชื่อ **action** (`confirmation`) ต่อท้ายชื่อ
  mailer ไปด้วยเลย (ต่างจาก controller ที่มักเจน action แยกทีหลัง) — generator จะสร้าง**ทั้ง
  method ใน mailer class และไฟล์ view ทั้งสองแบบ (`.html.erb` + `.text.erb`) ให้พร้อมกันทันที**
- คอมเมนต์เรื่อง I18n บอกว่า `subject` กำหนดผ่านไฟล์ locale ได้ (ทบทวน I18n จาก Part 039) แต่
  Part นี้จะกำหนด `subject` ตรงๆ ใน method เพื่อความชัดเจน (ทั้งสองวิธีถูกต้อง แค่คนละสไตล์)

### `app/mailers/application_mailer.rb` — Base Class ของทุก Mailer

```ruby
# app/mailers/application_mailer.rb (Rails สร้างให้อัตโนมัติตอน rails new)
class ApplicationMailer < ActionMailer::Base
  default from: "from@example.com"
  layout "mailer"
end
```

`default from:` กำหนดค่า default ที่ mailer ทุกตัวใช้ร่วมกัน (override ได้รายตัวถ้าจำเป็น) และ
`layout "mailer"` ชี้ไปที่ `app/views/layouts/mailer.html.erb`/`mailer.text.erb` (จะพูดถึงเต็ม
รูปแบบใน Step 719)

### เขียน `OrderMailer#confirmation` ให้รับ Order จริง

```ruby
# app/mailers/order_mailer.rb
class OrderMailer < ApplicationMailer
  def confirmation(order)
    @order = order
    @user = order.user

    mail(to: @user.email, subject: "ยืนยันคำสั่งซื้อ ##{@order.id} - #{@order.product_name}")
  end
end
```

**อธิบาย:**

- Method ของ mailer รับ argument ได้ตามปกติเหมือน method ทั่วไป (ในที่นี้รับ `order` object)
  ต่างจาก controller action ที่รับได้แค่ผ่าน `params`
- Instance variable ที่ตั้งไว้ (`@order`, `@user`) ใช้ได้ในไฟล์ view ทั้งสองแบบทันที (กลไก
  เดียวกับ controller → view ทุกประการ — ทบทวนจาก Part 024)
- **`mail(to:, subject:)` คือหัวใจของ method** — เมื่อเรียกแล้ว Action Mailer จะไปมองหา **ทั้ง
  `confirmation.html.erb` และ `confirmation.text.erb`** โดยอัตโนมัติจากชื่อ mailer + ชื่อ method
  แล้ว render ทั้งคู่ประกอบเป็นอีเมลแบบ **multipart** (มีทั้งเวอร์ชัน HTML และ plain text ในอีเมล
  เดียวกัน — email client จะเลือกแสดงเวอร์ชันที่รองรับให้อัตโนมัติ)

### View ทั้งสองแบบ — HTML และ Text

```erb
<%# app/views/order_mailer/confirmation.html.erb %>
<div class="card">
  <p class="title">ยืนยันคำสั่งซื้อ #<%= @order.id %></p>

  <p>สวัสดีคุณ <%= @user.name.presence || @user.email %>,</p>
  <p>ขอบคุณที่สั่งซื้อ <strong><%= @order.product_name %></strong> กับเรา</p>

  <p class="total">ยอดชำระ: <%= number_to_currency(@order.amount_baht, unit: "฿", format: "%u%n") %></p>
</div>
```

```erb
<%# app/views/order_mailer/confirmation.text.erb %>
ยืนยันคำสั่งซื้อ #<%= @order.id %>

สวัสดีคุณ <%= @user.name.presence || @user.email %>,
ขอบคุณที่สั่งซื้อ <%= @order.product_name %> กับเรา

ยอดชำระ: <%= number_to_currency(@order.amount_baht, unit: "฿", format: "%u%n") %>
```

**อธิบาย:** `number_to_currency` เป็น helper จาก `ActionView::Helpers::NumberHelper` — Action
Mailer view ใช้ view helper ทั้งหมดที่ view ปกติใช้ได้เหมือนกันทุกประการ (ทดสอบรันจริงแล้วว่า
`number_to_currency(1500.0, unit: "฿", format: "%u%n")` ให้ผลลัพธ์ `"฿1,500.00"` ถูกต้อง) — ทำไม
ต้องเขียนสอง view คู่กันเสมอ? เพราะอีเมล client จำนวนมาก (โดยเฉพาะระบบเก่า, screen reader, หรือ
ผู้ใช้ที่ปิดการแสดง HTML ด้วยเหตุผลความปลอดภัย) แสดงได้แค่ plain text — การมี text version คู่กัน
เสมอคือ**แนวปฏิบัติมาตรฐานของอีเมล production ทุกฉบับ**

### ทดสอบจริง: เรียก Mailer แล้วตรวจสอบผลลัพธ์

```ruby
mail = OrderMailer.confirmation(order).deliver_now
```

ผลการทดสอบจริง:

```
mail.class = Mail::Message
subject = ยืนยันคำสั่งซื้อ #1 - คอร์ส Ruby on Rails (Lifetime)
to = ["buyer@example.com"]
multipart? = true
has html_part? = true
has text_part? = true
```

**สังเกตว่า `OrderMailer.confirmation(order)` ไม่ได้เรียก `.deliver_now` ทันที** — มันคืนค่าเป็น
`ActionMailer::MessageDelivery` object ก่อน (เรียกว่า pattern **lazy delivery**) ต้องต่อท้ายด้วย
`.deliver_now` (ส่งทันที แบบ synchronous) หรือ `.deliver_later` (ส่งผ่าน background job — Step 718)
เพื่อให้อีเมลถูกส่งจริง — ถ้าลืมเรียกทั้งสองแบบนี้ อีเมลจะ**ไม่ถูกส่งเลย**โดยไม่มี error ใดๆ
แจ้งเตือน (เป็นบั๊กเงียบที่พบบ่อยมากตอนเขียน mailer ครั้งแรก)

---

## Step 716: Preview อีเมลในเครื่อง — `ActionMailer::Preview` และหน้า `/rails/mailers`

การรัน `deliver_now` ทุกครั้งเพื่อดูหน้าตาอีเมลระหว่างพัฒนาเป็นเรื่องยุ่งยาก (ต้องสร้างข้อมูลจริง,
ต้องมี mail server ตั้งค่าไว้) Rails จึงมี **`ActionMailer::Preview`** — กลไก preview อีเมลผ่าน
เบราว์เซอร์โดยไม่ต้องส่งจริงเลยแม้แต่ฉบับเดียว

### สร้าง Preview Class

```ruby
# test/mailers/previews/order_mailer_preview.rb
# (Rails มองหาไฟล์ preview ในโฟลเดอร์นี้โดย default เสมอ ไม่ว่าโปรเจกต์จะใช้ RSpec หรือ Minitest)
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    user = User.first || User.create!(email: "preview@example.com", password: "password123", name: "Preview User")
    order = user.orders.first || user.orders.create!(product_name: "คอร์สตัวอย่าง", amount_cents: 99_000, status: "paid")
    OrderMailer.confirmation(order)
  end
end
```

**อธิบาย:**

- Class ต้องสืบทอดจาก `ActionMailer::Preview` และตั้งชื่อ `<MailerName>Preview` เสมอ (ทบทวน
  convention การตั้งชื่อจาก Part 021)
- แต่ละ method ในนี้ (ในที่นี้คือ `confirmation`) ต้อง **คืนค่าเป็นผลลัพธ์ของการเรียก mailer
  method** (`OrderMailer.confirmation(order)`) — สังเกตว่า**ไม่มีการเรียก `.deliver_now`/
  `.deliver_later` เลย** เพราะ preview แค่ render ให้ดู ไม่ได้ส่งจริง
- ต้องเตรียมข้อมูลจริง (`User`, `Order`) ให้ method ใช้งานได้ — ในตัวอย่างนี้ใช้
  `find_or_create` แบบง่ายๆ เพื่อให้ preview ทำงานได้แม้ฐานข้อมูลว่างเปล่า

### เข้าดูผ่าน `/rails/mailers`

Rails mount engine พิเศษไว้ที่ path `/rails/mailers` โดยอัตโนมัติในโหมด development
(`config.action_mailer.show_previews` เป็น `true` โดย default ในสภาพแวดล้อมนี้) — ทดสอบจริงด้วย
`curl` ยืนยันว่าใช้งานได้จริง:

```bash
curl -s -o /tmp/mailers_index.html -w "HTTP_STATUS:%{http_code}\n" http://127.0.0.1:3199/rails/mailers
# => HTTP_STATUS:200
```

หน้า index แสดงลิงก์ไปยัง mailer ทุกตัวที่มี preview:

```html
<a href="/rails/mailers/order_mailer">...</a>
<a href="/rails/mailers/order_mailer/confirmation">...</a>
```

```bash
curl -s -o /tmp/mailer_preview_show.html -w "HTTP_STATUS:%{http_code}\n" \
  "http://127.0.0.1:3199/rails/mailers/order_mailer/confirmation"
# => HTTP_STATUS:200
```

ทดสอบจริงยืนยันว่าหน้านี้ render เนื้อหาอีเมลจริงออกมาให้เห็นทั้งสองเวอร์ชัน (HTML แสดงในกรอบ
iframe พร้อม preview แบบ mobile/desktop สลับได้ และ text แสดงเป็น plain text) โดยไม่มีการส่งอีเมล
ออกไปจริงแม้แต่ฉบับเดียว — เหมาะมากสำหรับ designer/developer ที่ต้องการดูหน้าตาอีเมลระหว่างพัฒนา
ซ้ำๆ ได้อย่างรวดเร็ว (แก้ไฟล์ view แล้ว refresh หน้าเว็บดูผลได้ทันที เหมือนแก้ view ปกติ)

> **แนวปฏิบัติที่ดี:** เขียน preview class คู่กับ mailer class ทุกตัวเสมอ เหมือนที่เขียน RSpec
> spec คู่กับทุก model (ทบทวนวินัยนี้จาก Part 046) — preview ไม่ใช่แค่เครื่องมือ debug แต่เป็น
> เอกสารประกอบที่มีชีวิต (living documentation) ว่าอีเมลแต่ละฉบับของระบบหน้าตาเป็นอย่างไร

---

## Step 717: ตั้งค่า Delivery Method — `:test`, `:letter_opener`, `:smtp` และรูปแบบ production จริง

Action Mailer แยก **"การ render เนื้อหาอีเมล"** ออกจาก **"วิธีที่อีเมลถูกส่งออกไปจริง"** อย่าง
ชัดเจน — ส่วนหลังควบคุมผ่าน `config.action_mailer.delivery_method` ซึ่งเปลี่ยนได้ตาม environment
โดยไม่ต้องแก้โค้ด mailer หรือ view เลยแม้แต่บรรทัดเดียว

### ตารางสรุป Delivery Method ตาม Environment

| Environment | `delivery_method` | พฤติกรรม |
|---|---|---|
| `test` | `:test` | ไม่ส่งจริง เก็บอีเมลไว้ใน `ActionMailer::Base.deliveries` (Array ในหน่วยความจำ) ให้ RSpec/Minitest ตรวจสอบได้ |
| `development` | `:letter_opener` (แนะนำ) หรือไม่ตั้งอะไรเลย (`:test` เป็น default) | เปิดอีเมลเป็นไฟล์ HTML แทนการส่งจริง (Step 718) |
| `production` | `:smtp` | ส่งจริงผ่าน SMTP server ของผู้ให้บริการ (SendGrid/Postmark/AWS SES) |

### `:test` — สำหรับ Automated Test (ทบทวนแนวคิดจาก Part 046-047)

```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test
```

เมื่อตั้งค่านี้ อีเมลทุกฉบับที่ `deliver_now`/`deliver_later` จะถูกเก็บไว้ใน
`ActionMailer::Base.deliveries` (Array) แทนการส่งออกไปจริง ทำให้เขียน assertion แบบนี้ได้ใน
RSpec:

```ruby
# spec/mailers/order_mailer_spec.rb (ตัวอย่างแนวคิด — จะเจาะลึกเต็มรูปแบบเรื่อง mailer spec
# อีกครั้งใน Phase 6 ที่ผ่านมาแล้ว หลักการเดียวกันนำมาประยุกต์กับ mailer ได้ทันที)
RSpec.describe OrderMailer do
  it "ส่งอีเมลไปยัง email ของ user ที่ถูกต้อง" do
    expect { OrderMailer.confirmation(order).deliver_now }
      .to change { ActionMailer::Base.deliveries.count }.by(1)

    mail = ActionMailer::Base.deliveries.last
    expect(mail.to).to eq([order.user.email])
  end
end
```

### `:letter_opener` — สำหรับ Development (รายละเอียดเต็มใน Step 718)

```ruby
# Gemfile
group :development do
  gem "letter_opener"
end
```

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
```

### `:smtp` — สำหรับ Production จริง (รูปแบบตามเอกสาร ไม่ได้ทดสอบเชื่อมต่อจริงในแซนด์บ็อกซ์นี้)

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true

config.action_mailer.smtp_settings = {
  address: "smtp.sendgrid.net",
  port: 587,
  domain: "example.com",
  user_name: "apikey", # SendGrid ใช้ literal string "apikey" เป็น username เสมอ (แปลกแต่ถูกต้อง)
  password: Rails.application.credentials.dig(:sendgrid, :api_key),
  authentication: "plain",
  enable_starttls_auto: true
}
```

**ตัวอย่างผู้ให้บริการ SMTP สำหรับ Transactional Email ที่นิยมใช้จริง:**

| ผู้ให้บริการ | `address` | หมายเหตุ |
|---|---|---|
| SendGrid | `smtp.sendgrid.net` | `user_name` เป็น literal `"apikey"` เสมอ, `password` คือ API key จริง |
| Postmark | `smtp.postmarkapp.com` | `user_name`/`password` เป็น Server API Token ตัวเดียวกันทั้งคู่ |
| AWS SES | `email-smtp.<region>.amazonaws.com` | ต้องสร้าง SMTP credentials แยกต่างหากจาก IAM access key ปกติผ่านหน้า SES Console |

> **แนวปฏิบัติที่ดี:** เก็บ API key/password ของ SMTP ผ่าน Rails encrypted credentials เสมอ
> (ทบทวน pattern จาก Part 067 และ Part 071 Step 702) **ห้าม hardcode ค่าจริงลงในโค้ดเด็ดขาด**
> — ตัวอย่างค่าใน Part นี้ทั้งหมดเป็นค่าสมมติ ไม่ใช่ credential จริงของผู้ให้บริการใดๆ

`config.action_mailer.raise_delivery_errors = true` ใน production สำคัญมาก — ถ้าเป็น `false`
(ค่า default ที่ Rails ตั้งให้ใน development) ความล้มเหลวของการส่งอีเมลจะ**เงียบหายไปโดยไม่มีใคร
รู้** ซึ่งยอมรับไม่ได้ใน production ที่อีเมลมักเป็นส่วนสำคัญของ business flow (เช่น อีเมลยืนยัน
คำสั่งซื้อ, อีเมลรีเซ็ตรหัสผ่าน)

---

## Step 718: ส่งอีเมลจริงด้วย `letter_opener` + `deliver_later` ผ่าน ActiveJob (ทบทวน Part 061)

### `letter_opener` คืออะไร และทำไมถึงเหมาะกับ Development

**`letter_opener`** เป็น gem ที่ไม่ได้ "ปลอมแปลง" การส่งอีเมล แต่ **render อีเมลเป็นไฟล์ HTML
จริงแล้วเปิดในเบราว์เซอร์แทนการส่งออกไปทาง SMTP จริง** — ข้อดีคือเห็นหน้าตาอีเมลเหมือนที่ผู้รับ
จะเห็นจริงๆ (ต่างจาก `:test` ที่แค่เก็บไว้ในหน่วยความจำเฉยๆ) โดยไม่ต้องตั้งค่า mail server ใดๆ
เลยระหว่างพัฒนา และไม่มีความเสี่ยงที่จะเผลอส่งอีเมลจริงไปหาลูกค้าโดยไม่ตั้งใจ

### ตั้งค่าและทดสอบจริง

```ruby
# Gemfile
group :development do
  gem "letter_opener"
end
```

```ruby
# config/environments/development.rb
config.action_mailer.raise_delivery_errors = false
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
```

ทดสอบจริงด้วยการเรียก `deliver_now`:

```ruby
mail = OrderMailer.confirmation(order).deliver_now
puts mail["location_rich"]&.value
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
/path/to/app/tmp/letter_opener/1790619059_9770482_b4b9650/rich.html
```

ตรวจสอบว่าไฟล์ถูกเขียนลงดิสก์จริง:

```bash
ls -la tmp/letter_opener/1790619059_9770482_b4b9650/
# -rw-r--r-- 1 root root 3264 ... plain.html
# -rw-r--r-- 1 root root 4311 ... rich.html
```

**อธิบาย:** `letter_opener` render อีเมลออกมาเป็น**สองไฟล์เสมอเมื่ออีเมลเป็น multipart**:
`rich.html` (จาก HTML part) และ `plain.html` (จาก text part ห่อด้วย `<pre>` ให้อ่านง่าย) แล้ว
เก็บ path ไว้ใน header พิเศษของ `Mail::Message` (`location_rich`/`location_plain`) — ในเครื่อง
พัฒนาจริงที่มีเบราว์เซอร์ gem จะเรียก `Launchy.open` เพื่อเปิดไฟล์นี้ในเบราว์เซอร์**อัตโนมัติ**
ทันทีที่อีเมลถูก "ส่ง" (ในแซนด์บ็อกซ์ headless นี้ไม่มีเบราว์เซอร์ให้เปิด แต่ไฟล์ HTML ยังถูก
เขียนลงดิสก์ถูกต้องสมบูรณ์เหมือนเดิม — นี่คือสิ่งที่เราตรวจสอบได้จริงในสภาพแวดล้อมนี้)

> **การล้างไฟล์ preview เก่า:** gem นี้ผูก task เพิ่มเข้ากับ `rake tmp:clear` ที่มีอยู่แล้วให้
> อัตโนมัติ (`namespace :tmp do task clear: :letter_opener end` — ทบทวนโครงสร้าง
> `task name: [:dependency]` และ `namespace` จาก **Part 020**) รันด้วย `bin/rails tmp:clear`
> เพื่อล้างไฟล์ HTML เก่าทั้งหมดใน `tmp/letter_opener/` ได้ทันที

### `deliver_later` — ส่งผ่าน Background Job (ทบทวน ActiveJob จาก Part 061)

Part 061 สอนไว้แล้วว่า **ActiveJob** คือ framework กลางของ Rails สำหรับงานที่ควรทำเบื้องหลัง
แทนที่จะบล็อก request/response cycle — การส่งอีเมลเป็นตัวอย่างคลาสสิกที่สุดของงานแบบนี้ (การเชื่อม
ต่อ SMTP server อาจใช้เวลาหลายร้อย milliseconds ถึงหลายวินาที ไม่ควรทำให้ผู้ใช้ต้องรอ)

```ruby
OrderMailer.confirmation(order).deliver_later
```

**ทดสอบจริง:**

```ruby
puts "queue_adapter ปัจจุบัน (development): #{Rails.application.config.active_job.queue_adapter.inspect}"
# => queue_adapter ปัจจุบัน (development): :async

ActiveJob::Base.queue_adapter = :test # สลับมาใช้ :test adapter ชั่วคราวเพื่อตรวจสอบการ enqueue
OrderMailer.confirmation(order).deliver_later
job = ActiveJob::Base.queue_adapter.enqueued_jobs.last
puts job[:job]   # => ActionMailer::MailDeliveryJob
puts job[:queue] # => default
```

**อธิบาย:**

- **Rails 8 ใช้ `:async` เป็น queue adapter default ใน development** (ทดสอบรันจริงยืนยันแล้ว) —
  ต่างจาก production ที่โปรเจกต์ใหม่ตั้งแต่ Rails 8 มักใช้ **Solid Queue** เป็น default แทน (gem
  `solid_queue` ที่ `rails new` เพิ่มให้อัตโนมัติ — ทบทวนรายละเอียดเต็มรูปแบบเรื่อง adapter
  ต่างๆ จาก Part 061-062)
- `deliver_later` ไม่ได้ enqueue `OrderMailer` โดยตรง แต่ enqueue job ชื่อ
  **`ActionMailer::MailDeliveryJob`** ซึ่งเป็น job กลางที่ Action Mailer เตรียมไว้ให้ — job นี้
  เก็บชื่อ mailer class, ชื่อ method, และ argument ที่ส่งเข้าไปทั้งหมด แล้วพอถูก perform จริงจะ
  เรียก `OrderMailer.confirmation(order).deliver_now` แทนเราอีกที
- ทดสอบสั่งให้ job ที่ enqueue ไว้ **ทำงานจริง** (`perform_enqueued_jobs` จาก
  `ActiveJob::TestHelper`) แล้วตรวจสอบว่ามีไฟล์ `tmp/letter_opener/` ใหม่เกิดขึ้นจริง — ยืนยันได้
  ว่า chain ทั้งหมด (`deliver_later` → enqueue → perform → `deliver_now` จริงข้างใน →
  `letter_opener` เขียนไฟล์) ทำงานถูกต้องครบวงจร ไม่ใช่แค่ enqueue เฉยๆ โดยไม่มีอะไรเกิดขึ้นจริง

> **แนวปฏิบัติที่ดีที่สุด:** ใช้ `deliver_later` เป็นค่าเริ่มต้นเสมอสำหรับอีเมลที่ไม่จำเป็นต้อง
> ยืนยันผลทันที (เช่น อีเมลยืนยันคำสั่งซื้อ, อีเมลต้อนรับ) ใช้ `deliver_now` เฉพาะกรณีที่จำเป็น
> ต้องรู้ผลการส่งทันทีก่อนตอบกลับผู้ใช้เท่านั้น (พบได้น้อยมากในทางปฏิบัติ)

---

## Step 719: Layout และ Styling ของอีเมล — inline CSS และ `premailer-rails`

### ทำไมอีเมลต้องใช้ Inline CSS

หน้าเว็บทั่วไปใช้ `<style>` block หรือไฟล์ `.css` แยกได้อย่างอิสระ แต่ **email client จำนวนมาก
(โดยเฉพาะ Outlook desktop ที่ใช้ Microsoft Word เป็น rendering engine, Gmail บางเวอร์ชัน) ไม่
รองรับ `<style>` block ใน `<head>` เลย หรือรองรับแค่บางส่วน** วิธีเดียวที่รับประกันได้ว่า CSS จะ
ถูกนำไปใช้จริงในทุก client คือเขียน CSS แบบ **inline** ตรงบน attribute `style="..."` ของแต่ละ
tag โดยตรง

ปัญหาคือการเขียน `style="..."` ซ้ำๆ ในทุก tag ด้วยมือนั้นน่าเบื่อและดูแลรักษายากมาก — ทางออก
มาตรฐานของวงการ Rails คือใช้ gem **`premailer-rails`**

### `premailer-rails` — แปลง CSS เป็น Inline ให้อัตโนมัติ

```ruby
# Gemfile
gem "premailer-rails"
```

```bash
bundle install
```

**จุดที่น่าประหลาดใจที่สุด (ทดสอบยืนยันแล้ว): ไม่ต้องตั้งค่าอะไรเพิ่มเลยแม้แต่บรรทัดเดียว**
แค่เพิ่ม gem ใน `Gemfile` แล้ว `bundle install` เท่านั้น `premailer-rails` จะ hook ตัวเองเข้ากับ
Action Mailer โดยอัตโนมัติผ่าน `Mail::Message` interceptor ทันทีที่อีเมลถูกส่ง (ไม่ว่าจะผ่าน
`deliver_now`, `deliver_later`, หรือ `letter_opener`) — เขียน CSS แบบไฟล์ปกติต่อไปได้เลย:

```erb
<%# app/views/order_mailer/confirmation.html.erb %>
<style>
  .card { border: 1px solid #e0e0e0; border-radius: 8px; padding: 24px; font-family: sans-serif; }
  .title { color: #2d3748; font-size: 20px; }
  .total { color: #2b6cb0; font-weight: bold; font-size: 18px; }
</style>

<div class="card">
  <p class="title">ยืนยันคำสั่งซื้อ #<%= @order.id %></p>
  ...
</div>
```

**ทดสอบจริง: ตรวจสอบ HTML body หลังส่งว่ามี inline style จริงหรือไม่**

```ruby
html_body = mail.html_part.body.decoded
puts html_body.include?('style="')  # => true
puts html_body.include?("<style>")  # => false (ถูกลบออกหลัง inline)
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
PASSED: พบ style="..." แบบ inline ใน tag
PASSED: <style> block ถูกลบออกแล้วหลัง inline (พฤติกรรม premailer-rails ปกติ)
```

ยืนยันว่า `premailer-rails` **แปลง `<style>` block ทั้งหมดให้เป็น `style="..."` inline บนแต่ละ
tag ที่เกี่ยวข้อง แล้วลบ `<style>` block เดิมทิ้งไปเอง** โดยอัตโนมัติสมบูรณ์ — เขียน CSS ได้อย่าง
เป็นระเบียบเหมือนเขียนหน้าเว็บปกติ แล้วปล่อยให้ gem นี้จัดการแปลงให้เข้ากับข้อจำกัดของ email
client แทนเรา

### Layout ของอีเมล — `app/views/layouts/mailer.html.erb`/`mailer.text.erb`

เช่นเดียวกับ view ปกติที่มี `application.html.erb` เป็น layout กลาง (ทบทวนจาก Part 024) mailer
ก็มี layout กลางของตัวเองที่ `ApplicationMailer` ชี้ไว้ด้วย `layout "mailer"`:

```erb
<%# app/views/layouts/mailer.html.erb (ไฟล์เริ่มต้นที่ Rails สร้างให้ตอน rails new) %>
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <style>
      /* Email styles need to be inline */
    </style>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

```erb
<%# app/views/layouts/mailer.text.erb %>
<%= yield %>
```

**อธิบาย:** คอมเมนต์ `/* Email styles need to be inline */` ที่ Rails ใส่ไว้ให้ตั้งแต่แรกคือคำ
เตือนตรงประเด็นเดียวกับที่อธิบายไว้ข้างบน — ก่อนมี `premailer-rails` นักพัฒนาต้องเขียน
`style="..."` ในทุก tag ด้วยมือเอง หรือใช้ helper อย่าง `content_tag` พร้อม inline style ที่คำนวณ
เอง Part นี้แสดงให้เห็นว่าการติดตั้ง gem เดียวช่วยตัดขั้นตอนที่น่าเบื่อและเสี่ยงต่อความผิดพลาด
นี้ทิ้งไปได้ทั้งหมด

---

## Step 720: SMS/Push Notification แบบ Multi-channel — Twilio, Firebase Cloud Messaging, และ `NotificationService`

### "Notification" ไม่ใช่แค่อีเมล — แนวคิด Multi-channel

ระบบ production จริงมักต้องแจ้งเตือนผู้ใช้ผ่านมากกว่าหนึ่งช่องทาง ขึ้นอยู่กับความเร่งด่วนและ
ความชอบของผู้ใช้แต่ละคน:

| ช่องทาง | เหมาะกับ | ตัวอย่างบริการที่นิยมใช้ |
|---|---|---|
| **Email** | ข้อมูลรายละเอียด ไม่เร่งด่วนมาก (ใบเสร็จ, สรุปรายเดือน) | SendGrid, Postmark, AWS SES (Step 717) |
| **SMS** | ข้อความสั้น เร่งด่วน ต้องการให้เห็นแน่นอน (OTP, แจ้งเตือนความปลอดภัย) | Twilio, AWS SNS |
| **Push Notification** | แจ้งเตือนบนมือถือ/เบราว์เซอร์แบบ real-time เมื่อแอปไม่ได้เปิดอยู่ | Firebase Cloud Messaging (FCM), Apple Push Notification (APNs) |
| **In-app** | แจ้งเตือนที่แสดงในแอปตอนผู้ใช้เปิดใช้งานอยู่ | เขียนเองด้วย ActionCable (ทบทวนแนวคิดจาก Phase 16) |

### SMS ผ่าน Twilio

**Twilio** เป็นผู้ให้บริการ communication API ที่ใหญ่ที่สุดเจ้าหนึ่ง รองรับทั้ง SMS, โทรศัพท์,
วิดีโอ และ WhatsApp — ติดตั้งผ่าน gem ทางการ:

```ruby
# Gemfile
gem "twilio-ruby"
```

```bash
bundle install
```

```ruby
# app/services/sms_notifier.rb
class SmsNotifier
  def self.client
    @client ||= Twilio::REST::Client.new(
      Rails.application.credentials.dig(:twilio, :account_sid),
      Rails.application.credentials.dig(:twilio, :auth_token)
    )
  end

  def self.send_sms(to:, body:)
    client.messages.create(
      from: Rails.application.credentials.dig(:twilio, :from_number),
      to: to,
      body: body
    )
  end
end
```

**ทดสอบจริงในแซนด์บ็อกซ์นี้ — ตรวจสอบได้แค่โครงสร้าง code ไม่ใช่การส่งจริง:**

```ruby
require "twilio-ruby"

client = Twilio::REST::Client.new("AC_FAKE_ACCOUNT_SID", "fake_auth_token")
puts client.class                          # => Twilio::REST::Client
puts client.messages.respond_to?(:create)  # => true

client.messages.create(from: "+15005550006", to: "+66800000000", body: "ทดสอบ")
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
twilio-ruby version: 7.11.2
client class: Twilio::REST::Client
client responds_to messages.create? true
ตามคาด: เรียก client.messages.create ได้ถูกต้องตาม API shape จริง
แต่ล้มเหลวตอนต่อ network จริง: Twilio::REST::TwilioError: 403 "Forbidden"
```

**สรุปตรงไปตรงมา:** `twilio-ruby` gem `require` ได้จริง, `Twilio::REST::Client.new(sid, token)`
และ `client.messages.create(from:, to:, body:)` คือ API ที่ถูกต้องจริงตามเอกสารและ source code
ของ gem เวอร์ชัน 7.11.2 — แต่แซนด์บ็อกซ์นี้ **ไม่มีบัญชี Twilio จริง และ network policy ปฏิเสธ
การเชื่อมต่อออกไปยัง `api.twilio.com` ด้วย `403` ที่ระดับ proxy** (รูปแบบเดียวกับที่ Part 071
Step 702 พบกับ `api.stripe.com`) จึงยืนยันได้แค่ว่าโค้ดเขียนถูกไวยากรณ์และเรียก method ถูกต้อง
ไม่ใช่ว่าส่ง SMS จริงสำเร็จ — ในโปรเจกต์จริงที่มีบัญชี Twilio (มี Account SID ขึ้นต้นด้วย `AC`
ตามด้วย hex 32 ตัว และ Auth Token จริง) โค้ดชุดนี้จะทำงานได้ทันทีโดยไม่ต้องแก้ไข

> **หมายเหตุเรื่องความปลอดภัยของ credential:** Account SID และ Auth Token ของ Twilio **ต้อง
> เก็บผ่าน Rails encrypted credentials เสมอ** (ทบทวน pattern จาก Part 067/071) เช่นเดียวกับ
> Stripe secret key — ห้ามฝัง string ที่มีรูปแบบตรงกับ credential จริงลงในโค้ดตัวอย่างหรือ commit
> เข้า git เด็ดขาด เพราะเครื่องมือสแกน secret อัตโนมัติของ GitHub (และของทีมรักษาความปลอดภัยจริง)
> จะตรวจจับรูปแบบเหล่านี้ได้ทันที

### Push Notification ผ่าน Firebase Cloud Messaging (FCM) — ภาพรวมเชิงแนวคิด

**Firebase Cloud Messaging** เป็นบริการฟรีของ Google สำหรับส่ง push notification ไปยังแอปมือถือ
(iOS/Android) และเว็บเบราว์เซอร์ กลไกคร่าวๆ (จากเอกสารทางการ — ไม่ได้ทดสอบเชื่อมต่อจริงใน
แซนด์บ็อกซ์นี้เพราะต้องมี Firebase project จริงและ service account credential จริง):

1. Client (มือถือ/เบราว์เซอร์) ลงทะเบียนกับ FCM แล้วได้ **device token** เฉพาะเครื่องกลับมา
2. Client ส่ง device token นั้นมาเก็บไว้ที่ backend ของเรา (ผูกกับ `user_id`)
3. เมื่อต้องการแจ้งเตือน backend เรียก FCM API พร้อมแนบ device token + ข้อความ
4. FCM จัดการส่งต่อไปยังอุปกรณ์จริงให้ (ผ่าน Apple Push Notification Service สำหรับ iOS หรือ
   Google Play Services สำหรับ Android โดยอัตโนมัติ — เราไม่ต้องคุยกับ APNs/Google โดยตรงเอง)

```ruby
# ตัวอย่างแนวคิด (จากเอกสารทางการของ gem "fcm" — ไม่ได้ทดสอบเชื่อมต่อจริง)
# Gemfile: gem "fcm"
fcm = FCM.new(Rails.application.credentials.dig(:firebase, :server_key))
fcm.send_v1(
  "projects/my-project/messages:send",
  {
    message: {
      token: user.device_token,
      notification: { title: "คำสั่งซื้อสำเร็จ", body: "ขอบคุณที่สั่งซื้อกับเรา" }
    }
  }
)
```

### `NotificationService` — Abstraction สำหรับ Dispatch หลายช่องทางพร้อมกัน

แนวคิดสำคัญที่สุดของ Step นี้: **"notification" ควรเป็นแนวคิดระดับสูงที่แยกออกจาก "ช่องทาง
เฉพาะเจาะจง"** — โค้ดที่เรียกใช้ (เช่น หลัง webhook ยืนยันการชำระเงินสำเร็จจาก Part 071)
ไม่ควรต้องรู้รายละเอียดว่าจะส่งผ่าน email, SMS, หรือ push โดยตรง แต่ควรเรียกผ่าน service กลาง
ที่ตัดสินใจแทน:

```ruby
# app/services/notification_service.rb
class NotificationService
  # ตัดสินใจว่าจะแจ้งเตือนผู้ใช้ผ่านช่องทางไหนบ้าง ตาม preference ของผู้ใช้และความสำคัญของ event
  # เพื่อไม่ให้โค้ดที่เรียกใช้ (เช่น OrderMailer webhook handler จาก Part 071) ต้องรู้รายละเอียด
  # ของแต่ละช่องทางเอง — ตรงตามหลัก Single Responsibility ที่เรียนมาตั้งแต่ Part 016
  def self.notify_order_confirmed(order)
    user = order.user

    # Email: ส่งเสมอ เพราะเป็นข้อมูลอ้างอิงที่ควรมีทุกครั้ง (ทบทวน deliver_later จาก Step 718)
    OrderMailer.confirmation(order).deliver_later

    # SMS: ส่งเฉพาะผู้ใช้ที่เปิดรับ SMS และมีเบอร์โทรลงทะเบียนไว้
    if user.sms_notifications_enabled? && user.phone_number.present?
      SmsNotifier.send_sms(
        to: user.phone_number,
        body: "คำสั่งซื้อ ##{order.id} ของคุณสำเร็จแล้ว ขอบคุณที่ใช้บริการ"
      )
    end

    # Push: ส่งเฉพาะผู้ใช้ที่มี device token ลงทะเบียนไว้ (เคยเปิดแอปและอนุญาต notification)
    if user.device_token.present?
      PushNotifier.send_push(
        token: user.device_token,
        title: "คำสั่งซื้อสำเร็จ",
        body: "คำสั่งซื้อ ##{order.id} ของคุณสำเร็จแล้ว"
      )
    end
  end
end
```

**อธิบาย:**

- ทุก channel ถูกเรียกจาก**จุดเดียว** (`NotificationService.notify_order_confirmed`) — ถ้าในอนาคต
  ต้องเพิ่มช่องทางที่ 4 (เช่น LINE Notify หรือ Slack) แก้ไขแค่ไฟล์นี้ไฟล์เดียว โค้ดที่เรียกใช้
  (webhook handler ใน Part 071 Step 707) **ไม่ต้องแก้ไขเลย**
  ```ruby
  # app/controllers/stripe/webhooks_controller.rb (ทบทวน/ต่อยอดจาก Part 071 Step 707)
  def handle_checkout_completed(session)
    order = Order.find_by(stripe_checkout_session_id: session.id)
    return unless order

    if session.payment_status == "paid" && !order.fulfilled?
      order.update!(stripe_payment_intent_id: session.payment_intent)
      order.fulfill!
      NotificationService.notify_order_confirmed(order) # <- จุดเชื่อมต่อ Part 071 กับ Part 072
    end
  end
  ```
- แต่ละช่องทางมีเงื่อนไขของตัวเอง (`sms_notifications_enabled?`, `phone_number.present?`,
  `device_token.present?`) — สะท้อนความจริงว่าผู้ใช้แต่ละคนเปิด/ปิดช่องทางไม่เหมือนกัน และ
  ไม่ใช่ทุกคนที่ให้ข้อมูลครบทุกช่องทาง
- **Email ใช้ `deliver_later`** (background job) แต่ SMS/Push ในตัวอย่างนี้เรียกแบบ synchronous
  — ในระบบจริงควรห่อ SMS/Push ด้วย ActiveJob เช่นกัน (`SmsDeliveryJob.perform_later(...)`) ด้วย
  เหตุผลเดียวกับอีเมล: ไม่ควรให้ผู้ใช้รอ network call ไปหา Twilio/FCM ก่อนได้รับ response กลับมา
  (ทิ้งเป็นแบบฝึกหัดเพิ่มเติมท้าย Part)

---

## แบบฝึกหัด: `OrderMailer` เต็มรูปแบบ + เชื่อม Provider ที่สอง

### โจทย์

ต่อยอดจาก Part 071 (ระบบขายคอร์สออนไลน์ที่ปิดท้ายด้วย webhook idempotent) ให้ครบตามนี้:

1. เพิ่ม gem `omniauth-github` เข้าไปในระบบ Omniauth ที่มีอยู่แล้ว (จาก Part 045) ให้รองรับทั้ง
   Google และ GitHub ผ่าน controller เดียว
2. สร้าง `OrderMailer#confirmation` ที่ส่งอีเมลยืนยันคำสั่งซื้อจริง ทั้ง HTML และ text version
3. เรียก mailer นี้ทันทีที่ webhook `checkout.session.completed` fulfill order สำเร็จ (เชื่อม
   Part 071 เข้ากับ Part 072) ผ่าน `deliver_later`
4. ตั้งค่า `letter_opener` ในเครื่อง dev แล้วพิสูจน์ว่าอีเมลถูก "ส่ง" จริง (มีไฟล์ HTML เกิดขึ้น)
5. เขียน `ActionMailer::Preview` คู่กับ mailer

### เฉลย

**Gemfile:**

```ruby
gem "omniauth", "~> 2.1"
gem "omniauth-google-oauth2", "~> 1.2"
gem "omniauth-github", "~> 2.0"
gem "omniauth-rails_csrf_protection", "~> 1.0"
gem "premailer-rails"

group :development do
  gem "letter_opener"
end
```

**`config/initializers/omniauth.rb`:**

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2, ENV["GOOGLE_CLIENT_ID"], ENV["GOOGLE_CLIENT_SECRET"], scope: "email,profile"
  provider :github, ENV["GITHUB_CLIENT_ID"], ENV["GITHUB_CLIENT_SECRET"], scope: "user:email"
end

OmniAuth.config.allowed_request_methods = [:post]
OmniAuth.config.silence_get_warning = true
```

**`config/routes.rb` (เพิ่มจากของ Part 071):**

```ruby
get "auth/:provider/callback", to: "omniauth_callbacks#callback"
get "auth/failure", to: "omniauth_callbacks#failure"
```

**`app/controllers/omniauth_callbacks_controller.rb`:**

```ruby
class OmniauthCallbacksController < ApplicationController
  def callback
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_from_omniauth(auth)
    token = JsonWebToken.encode({ user_id: user.id })
    render json: { token: token, provider: auth.provider, user: { id: user.id, email: user.email } }
  end

  def failure
    render json: { error: "เข้าสู่ระบบผ่าน OAuth ไม่สำเร็จ: #{params[:message]}" }, status: :unauthorized
  end
end
```

**`app/mailers/order_mailer.rb`:**

```ruby
class OrderMailer < ApplicationMailer
  def confirmation(order)
    @order = order
    @user = order.user
    mail(to: @user.email, subject: "ยืนยันคำสั่งซื้อ ##{@order.id} - #{@order.product_name}")
  end
end
```

**`app/views/order_mailer/confirmation.html.erb`:**

```erb
<style>
  .card { border: 1px solid #e0e0e0; border-radius: 8px; padding: 24px; font-family: sans-serif; }
  .title { color: #2d3748; font-size: 20px; }
  .total { color: #2b6cb0; font-weight: bold; font-size: 18px; }
</style>

<div class="card">
  <p class="title">ยืนยันคำสั่งซื้อ #<%= @order.id %></p>
  <p>สวัสดีคุณ <%= @user.name.presence || @user.email %>,</p>
  <p>ขอบคุณที่สั่งซื้อ <strong><%= @order.product_name %></strong> กับเรา</p>
  <p class="total">ยอดชำระ: <%= number_to_currency(@order.amount_baht, unit: "฿", format: "%u%n") %></p>
</div>
```

**`app/views/order_mailer/confirmation.text.erb`:**

```erb
ยืนยันคำสั่งซื้อ #<%= @order.id %>

สวัสดีคุณ <%= @user.name.presence || @user.email %>,
ขอบคุณที่สั่งซื้อ <%= @order.product_name %> กับเรา

ยอดชำระ: <%= number_to_currency(@order.amount_baht, unit: "฿", format: "%u%n") %>
```

**`app/controllers/stripe/webhooks_controller.rb` (ต่อยอดจาก Part 071 Step 708):**

```ruby
def handle_checkout_completed(session)
  order = Order.find_by(stripe_checkout_session_id: session.id)
  return unless order

  if session.payment_status == "paid" && !order.fulfilled?
    order.update!(stripe_payment_intent_id: session.payment_intent)
    order.fulfill!
    OrderMailer.confirmation(order).deliver_later
  end
end
```

**`config/environments/development.rb`:**

```ruby
config.action_mailer.raise_delivery_errors = false
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.perform_deliveries = true
```

**`test/mailers/previews/order_mailer_preview.rb`:**

```ruby
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    order = Order.joins(:user).first
    OrderMailer.confirmation(order)
  end
end
```

### ทดสอบจริงแบบเต็ม (สถานะการทดสอบ)

> เฉลยทั้งหมดข้างต้น **ทดสอบจริงครบทุกจุด** กับ Rails 8.1.4 + Ruby 3.3.6 บน Puma server จริง,
> SQLite จริง: การ login ผ่านทั้งสอง provider (จำลองด้วย `OmniAuth.config.test_mode` เพราะไม่มี
> Client ID/Secret จริงในแซนด์บ็อกซ์ — เหมือนที่ Part 045 ระบุไว้แล้วว่าจำเป็นสำหรับส่วนนี้),
> การส่งอีเมลจริงผ่าน `letter_opener` (ไฟล์ HTML เกิดขึ้นจริงในดิสก์), `deliver_later` enqueue
> และ perform job จริง, `premailer-rails` inline CSS จริง, และหน้า `/rails/mailers` แสดงผลจริง

```
== ผลการทดสอบ Login ==
GET /auth/google_oauth2/callback → 200, token ออกมาถูกต้อง, user id=5
GET /auth/github/callback        → 200, token ออกมาถูกต้อง, user id=6

== ผลการทดสอบ Mailer ==
mail.multipart? = true, has html_part? = true, has text_part? = true
HTML body มี inline style="..." จาก premailer-rails: PASSED
letter_opener เขียนไฟล์จริงที่ tmp/letter_opener/.../rich.html: PASSED

== ผลการทดสอบ deliver_later ==
queue_adapter (development) = :async
enqueue ActionMailer::MailDeliveryJob สำเร็จ: PASSED
perform job จริงแล้วมีไฟล์ letter_opener ใหม่เกิดขึ้น: PASSED

== ผลการทดสอบ /rails/mailers ==
GET /rails/mailers                          → 200
GET /rails/mailers/order_mailer/confirmation → 200 (แสดงเนื้อหาอีเมลจริง)
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม endpoint สำหรับเชื่อมบัญชี provider ที่สอง** — สร้าง `POST /auth/:provider/link_start`
   ที่รับ JWT ยืนยันตัวตนผู้ใช้ปัจจุบัน แล้วเก็บ `session[:linking_user_id]` ไว้ (ตามที่อธิบายใน
   Step 714) จากนั้นแก้ `callback` action ให้ตรวจสอบ session นี้และเรียก `link_omniauth!` แทน
   `find_or_create_from_omniauth` เมื่อพบ — อย่าลืมแก้ `link_omniauth!` ให้ตรวจสอบ "เชื่อมกับคน
   อื่นไปแล้ว" ตามบั๊กจริงที่ Step 714 อธิบายไว้ด้วย แล้วเขียนสคริปต์ทดสอบทั้งสองเคส (เชื่อม
   สำเร็จ และถูกปฏิเสธเพราะ provider นั้นถูกคนอื่นเชื่อมไปแล้ว)
2. **ห่อ `NotificationService` ด้วย ActiveJob ทั้งหมด** — สร้าง `NotificationJob` ที่รับ
   `order_id` แล้วเรียก `NotificationService.notify_order_confirmed` ข้างในแทนที่จะเรียก
   `deliver_later`/`SmsNotifier`/`PushNotifier` แยกกันตรงๆ เพื่อให้ควบคุม retry/error handling
   ของทั้ง 3 ช่องทางพร้อมกันได้จากจุดเดียว (ทบทวน retry pattern ของ ActiveJob จาก Part 061-062)
3. **เพิ่ม User preference สำหรับแต่ละช่องทาง** — เพิ่มคอลัมน์
   `email_notifications_enabled`/`sms_notifications_enabled`/`push_notifications_enabled` (boolean,
   default `true`) ลงในตาราง `users` แล้วแก้ `NotificationService` ให้เช็คทุกช่องทางก่อนส่งเสมอ
   ไม่ใช่แค่ SMS/Push อย่างที่ทำไว้ในตัวอย่าง พร้อมเพิ่ม endpoint `PATCH /me/notification_preferences`
   ให้ผู้ใช้ปรับตั้งค่าเองได้

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ทบทวน Omniauth/JWT จาก Part 045 แบบรวบรัด แล้วต่อยอดสู่การรองรับ **หลาย Provider พร้อมกัน**
  ผ่าน `OmniAuth::Builder` ที่ลงทะเบียน `provider` ได้หลายบรรทัด และ **unified**
  `OmniauthCallbacksController` ที่ใช้ dynamic route `:provider` action เดียวรองรับทุก provider
- ออกแบบโมเดลข้อมูลที่ถูกต้องสำหรับ multi-provider ด้วยตาราง `identities` แยกต่างหาก
  (`has_many :identities`) พร้อม unique index บน `[:provider, :uid]` แทนการผูก `provider`/`uid`
  ไว้ในตาราง `users` ตรงๆ แบบ Part 045
- เขียน `find_or_create_from_omniauth` ที่รองรับทั้งการ login ซ้ำ, การ auto-link ผ่าน email ที่
  ตรงกัน, และการสร้าง user ใหม่ — ทดสอบผ่าน Rack/HTTP จริงครบทุกเคส
- ออกแบบและทดสอบ flow **การเชื่อมบัญชีที่สอง (link)** เข้ากับ user ที่ login อยู่แล้ว ผ่าน
  session + CSRF token จริง พร้อม**ค้นพบและแก้บั๊กจริง**เกี่ยวกับ `find_or_create_by!` ที่ทำงาน
  ผ่าน association scope ร่วมกับ unique index (ปัญหาที่หาไม่เจอจากการอ่านเอกสารเฉยๆ)
- เข้าใจพื้นฐาน **Action Mailer**: `rails generate mailer`, โครงสร้าง multipart HTML/text,
  `mail(to:, subject:)`, และ pattern lazy delivery (`.deliver_now`/`.deliver_later`)
- Preview อีเมลได้จริงผ่าน `ActionMailer::Preview` และหน้า `/rails/mailers` โดยไม่ต้องส่งจริง
  แม้แต่ฉบับเดียว
- ตั้งค่า delivery method ได้ถูกต้องตาม environment: `:test` สำหรับ automated test,
  `:letter_opener` สำหรับ development (ทดสอบจริงว่าเขียนไฟล์ HTML preview จริง), และ `:smtp`
  สำหรับ production จริงผ่าน SendGrid/Postmark/AWS SES
- เชื่อม `deliver_later` เข้ากับ ActiveJob (ทบทวน Part 061) ยืนยันได้จริงว่า enqueue
  `ActionMailer::MailDeliveryJob` และ perform จริงจนอีเมลถูกส่งสำเร็จ
- เข้าใจปัญหา inline CSS ของอีเมล และใช้ `premailer-rails` แปลง `<style>` block เป็น inline
  style อัตโนมัติโดยไม่ต้องตั้งค่าเพิ่ม (ทดสอบยืนยันจริง)
- เข้าใจแนวคิด **multi-channel notification** (email/SMS/push/in-app) และออกแบบ
  `NotificationService` เป็น abstraction กลางที่แยกช่องทางออกจาก business logic พร้อมตรวจสอบ
  โครงสร้างจริงของ `twilio-ruby` (require ได้, เรียก API ถูกต้อง แต่ส่ง SMS จริงไม่ได้เพราะ
  network policy ของแซนด์บ็อกซ์)

## สรุปภาพรวม Phase 11: Payment & Third-party Integration

ยินดีด้วย! ตอนนี้ **Phase 11: Payment & Third-party Integration (Part 071–072, Step 701–720)**
เสร็จสมบูรณ์แล้ว Phase นี้สั้นกว่า Phase อื่นๆ (มีแค่ 2 Part) แต่เข้มข้นมาก เพราะทั้งสอง Part
พาเราออกจาก "โลกที่ควบคุมได้ทั้งหมด" (โค้ดของเราเอง, ฐานข้อมูลของเราเอง) ไปสู่ **โลกที่ต้อง
เชื่อมต่อกับระบบภายนอกที่เราควบคุมไม่ได้โดยตรง** — Part 071 สอนให้รับเงินจริงผ่าน **Stripe**
อย่างปลอดภัย (Checkout Session, webhook verification, idempotency, subscription lifecycle)
และ Part นี้สอนให้เชื่อมต่อกับ **ผู้ให้บริการยืนยันตัวตน (Google/GitHub)** และ **ผู้ให้บริการ
ส่งข้อความ (SMTP, Twilio, FCM)** ด้วยหลักการความปลอดภัยและความน่าเชื่อถือชุดเดียวกัน

บทเรียนที่ร้อยเรียงทั้ง Phase นี้เข้าด้วยกันคือ **"อย่าเชื่อ signal ที่ปลอมแปลงได้ง่าย"** — Part 071
เตือนว่าอย่าเชื่อ `success_url` redirect เพียงอย่างเดียว (ต้องรอ webhook ที่ verify signature
ได้) Part นี้เตือนแบบเดียวกันเรื่อง OAuth: อย่าผูก "การเชื่อมบัญชี" กับ query parameter ที่ปลอมแปลง
ได้ (ต้องใช้ session ที่ผูกกับ cookie ของ browser จริง) และอย่าเชื่อว่า callback URL เดียวกัน
หมายถึงเจตนาเดียวกันเสมอ (ต้องแยก "login" กับ "link" ให้ชัดเจน) — ทั้งสอง Part ยังเน้นย้ำหลักการ
**idempotency** และ **testing ที่ทดสอบจริง ไม่ใช่แค่เขียนตามเอกสาร** ผ่านการค้นพบบั๊กจริงทั้งใน
Part 071 (attribute ของ Stripe SDK ที่เปลี่ยนตำแหน่ง) และ Part นี้ (`find_or_create_by!` กับ
association scope) — เป็นเครื่องเตือนใจว่าแม้แต่โค้ดที่ "ดูถูกต้อง" ก็ควรพิสูจน์ด้วยการทดสอบจริง
เสมอ โดยเฉพาะเมื่อเชื่อมต่อกับระบบภายนอกที่มีพฤติกรรมซับซ้อนกว่าที่เอกสารสรุปไว้

ทักษะทั้งหมดจาก Phase 11 — Stripe Checkout/webhook, OAuth หลาย provider, Action Mailer,
background notification — คือชุดเครื่องมือที่นักพัฒนา Rails ระดับ production แทบทุกคนต้องใช้จริง
ไม่ช้าก็เร็ว และจะถูกนำมาใช้ซ้ำอีกครั้งอย่างเข้มข้นในโปรเจกต์รวบยอด **E-commerce platform** ของ
**Phase 16 (Part 091–092)**

**ต่อไป (Part 073 — เปิด Phase 12: DevOps & Deployment):** ตลอด 72 Part ที่ผ่านมา เราพัฒนาแอป
Rails บนเครื่องของเราเองมาโดยตลอด รันด้วย `bin/rails server` ธรรมดา — **Phase 12** จะเปลี่ยน
มุมมองไปสู่ **"ทำอย่างไรให้แอปนี้รันได้บนเครื่องอื่น เซิร์ฟเวอร์อื่น อย่างสม่ำเสมอทุกครั้ง"**
เริ่มต้นด้วย **Part 073: Docker สำหรับ Rails** — เขียน `Dockerfile` ที่ build image ของแอป Rails
ให้รันได้แบบ reproducible, ใช้ `docker-compose` ประกอบ service ที่แอปต้องพึ่งพา (PostgreSQL,
Redis) เข้าด้วยกันเป็นสภาพแวดล้อมเดียวที่สั่ง `docker compose up` แล้วทำงานได้ทันทีไม่ว่าจะรันบน
เครื่องไหนก็ตาม — จุดเริ่มต้นของเส้นทางสู่การ deploy แอปสู่ production จริงใน Part ถัดๆ ไป
