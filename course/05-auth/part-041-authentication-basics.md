# Part 041: Authentication เบื้องต้น — has_secure_password และ session-based login/logout

> **Step ครอบคลุมใน Part นี้:** Step 401–410
> **ระดับ:** ปานกลาง (ต่อจากเฟส 4 เรื่อง CRUD/Forms — เปิดเฟส 5: Authentication & Authorization)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Rails 8.1.4)

ถึงตอนนี้เราสร้างเว็บแอปที่มี Model, View, Controller, Form, Validation ครบแล้ว (เฟส 3–4)
แต่ทุกแอปที่ผ่านมายังไม่มีแนวคิดเรื่อง **"ใครเป็นใคร"** เลย — ทุกคนที่เข้าเว็บเห็นข้อมูล
เหมือนกันหมด แก้ไข/ลบข้อมูลได้เหมือนกันหมด ซึ่งไม่มีแอปจริงในโลกไหนทำงานแบบนี้

Part นี้เปิด **เฟส 5: Authentication & Authorization** โดยเจาะเฉพาะเรื่อง
**Authentication** (การพิสูจน์ตัวตน) ก่อน — เราจะสร้างระบบ signup/login/logout ด้วยมือของ
เราเองทั้งหมดก่อน 1 รอบ เพื่อให้เข้าใจกลไกเบื้องหลังแบบไม่มีอะไรซ่อน จากนั้นจะแนะนำ
เครื่องมือที่ **Rails 8 เตรียมมาให้ในตัว** คือคำสั่ง `bin/rails generate authentication`
ซึ่งสร้างระบบเดียวกันนี้ให้แบบพร้อมใช้ (production-grade) ในคำสั่งเดียว แล้วเปรียบเทียบ
โค้ดทั้งสองแบบทีละจุดว่าเหมือน/ต่างกันอย่างไร

> **หมายเหตุเรื่องเครื่องมือ:** ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยการสร้างแอป Rails
> เปล่าๆ 2 ตัว (ตัวหนึ่งสร้างระบบ auth ด้วยมือ อีกตัวรัน
> `bin/rails generate authentication` จริง) รัน `bin/rails server` แล้วยิง request ด้วย
> `curl` เพื่อดู response, cookie, database, และ log จริงทุกจุด — ไม่ใช่โค้ดที่เขียนขึ้นลอยๆ

## สารบัญของ Part นี้

- Step 401: Authentication vs Authorization — ทบทวนความต่าง และภาพรวมของ Part นี้
- Step 402: `has_secure_password` และ `bcrypt` — สร้าง User model, ทำไมห้ามเก็บรหัสผ่านแบบ plaintext
- Step 403: สร้างระบบ Signup ด้วยมือ — `RegistrationsController`, validation ที่มากับ `has_secure_password`
- Step 404: สร้างระบบ Login/Logout ด้วยมือ — `SessionsController`, `session[:user_id]`, `current_user`, `require_authentication`
- Step 405: รู้จัก `bin/rails generate authentication` — ฟีเจอร์ใหม่ของ Rails 8 ที่สร้างระบบ auth ให้แบบเต็มรูปแบบ
- Step 406: อ่านโค้ดที่ generator สร้าง (1) — `User`, `Session`, `Current`, และ `Authentication` concern
- Step 407: อ่านโค้ดที่ generator สร้าง (2) — `SessionsController`, `authenticate_by`, `rate_limit` ในตัว
- Step 408: Password Reset เต็มรูปแบบ — `generates_token_for`, `PasswordsController`, `PasswordsMailer`, `deliver_later`
- Step 409: Remember-me, persistent session, และ Security Checklist ก่อนขึ้น production
- Step 410: แบบฝึกหัด — สร้างระบบ auth เต็มวงจรด้วย generator พร้อม Registration ของตัวเอง

---

## Step 401: Authentication vs Authorization — ทบทวนความต่าง และภาพรวมของ Part นี้

ก่อนเริ่มเขียนโค้ด ต้องแยกคำสองคำนี้ให้ชัดเจนก่อน เพราะเป็นคำที่มือใหม่สับสนกันบ่อยที่สุด
ในวงการเว็บ:

| คำ | ความหมาย | คำถามที่ตอบ | ตัวอย่างในหลักสูตรนี้ |
|---|---|---|---|
| **Authentication** (การพิสูจน์ตัวตน) | ตรวจสอบว่า "คุณคือใคร" | "คุณเป็น user คนไหนในระบบ?" | Part นี้ (041): signup, login, logout |
| **Authorization** (การกำหนดสิทธิ์) | ตรวจสอบว่า "คุณทำสิ่งนี้ได้ไหม" | "user คนนี้แก้ไขบทความนี้ได้หรือไม่?" | Part 043–044: Pundit, CanCanCan, RBAC |

พูดสั้นๆ: **Authentication มาก่อนเสมอ** — ระบบต้องรู้ก่อนว่าใครกำลังใช้งานอยู่
(authentication สำเร็จ) ถึงจะไปตัดสินใจต่อได้ว่าคนนั้น "มีสิทธิ์" ทำอะไรได้บ้าง
(authorization) ตัวอย่างเทียบกับชีวิตจริง: การ์ดพนักงานที่ใช้แตะเข้าตึก (authentication —
พิสูจน์ว่าเป็นพนักงานจริง) ต่างจากการที่การ์ดใบเดียวกันเปิดได้แค่บางชั้น
(authorization — สิทธิ์เข้าถึงเฉพาะพื้นที่)

ใน Part นี้เราจะสร้างระบบตอบคำถาม **"คุณเป็นใคร"** เท่านั้น — ยังไม่มีเรื่อง role, permission,
หรือ "user คนนี้แก้ไขได้เฉพาะโพสต์ของตัวเอง" (เดี๋ยวจะเรียนเรื่องนั้นเต็มๆ ใน Part 043–044)

### ทบทวนพื้นฐานที่มีอยู่แล้วจาก Part 023

จำได้ไหมว่าใน Part 023 (เรื่อง Controller) เราเคยเห็นกลไกเหล่านี้มาแล้ว:

- `session` — เก็บข้อมูลข้าม request แบบผูกกับ browser/ผู้ใช้แต่ละคน (Step 225)
- `before_action` พร้อมกลไก **halt the filter chain** เมื่อมี `redirect_to`/`render`
  เกิดขึ้นข้างใน (Step 228)
- ตอนนั้นเราสร้าง `require_token` แบบง่ายๆ ที่เช็ค `params[:token] == "secret123"` และตั้งข้อสังเกต
  ไว้ว่า **"ระบบ authentication จริงจะซับซ้อนกว่านี้มาก ... ซึ่งจะเรียนแบบเต็มในเฟส 5
  (Part 041–045)"**

Part นี้คือการเติมเต็มคำสัญญานั้น — `before_action :require_authentication` ที่เราจะเขียนใน
Step 404 ใช้กลไกตัวเดียวกับ `require_token` ทุกประการ เพียงแต่แทนที่จะเช็ค
`params[:token]` แบบดิบๆ เราจะเช็คว่า **มี user ที่ login อยู่ใน session จริงหรือไม่**

### แผนการเรียนใน Part นี้ (ทำไมต้องสร้างมือก่อน)

หลายคนอยากกระโดดไปใช้ generator หรือ gem อย่าง Devise (Part 042) ทันที แต่หลักสูตรนี้
ยืนยันให้สร้างด้วยมือก่อนเสมอ ด้วยเหตุผลเดียวกับที่ไม่สอน scaffold ก่อนสอน MVC แยกส่วน
(Part 021–028): **ถ้าไม่เข้าใจกลไกเบื้องหลัง เวลา generator/gem ทำงานผิดปกติหรือต้อง
custom พฤติกรรม จะแก้ปัญหาไม่ถูก**

ลำดับของ Part นี้จึงเป็น:

1. **Step 402–404:** สร้างระบบ signup/login/logout ด้วยมือทั้งหมด ไม่พึ่ง gem ใดๆ
   นอกจาก `bcrypt` (ซึ่งเป็นแค่ตัวช่วย hash รหัสผ่าน ไม่ใช่ authentication framework)
2. **Step 405–408:** แนะนำ `bin/rails generate authentication` ที่ Rails 8 เตรียมไว้ให้
   ในตัว แล้วอ่านโค้ดที่มันสร้างทีละไฟล์ เทียบกับสิ่งที่เราเขียนมือใน Step 402–404
3. **Step 409:** สรุปเรื่อง security ที่ต้องรู้ก่อนขึ้น production จริง

---

## Step 402: `has_secure_password` และ `bcrypt` — สร้าง User model, ทำไมห้ามเก็บรหัสผ่านแบบ plaintext

### ทำไมห้ามเก็บรหัสผ่านตรงๆ ในฐานข้อมูล

ก่อนเขียนโค้ด ต้องเข้าใจหลักการนี้ให้แน่นก่อน: **ห้ามเก็บรหัสผ่านผู้ใช้เป็น plaintext
(ข้อความธรรมดา) ในฐานข้อมูลเด็ดขาด** ไม่ว่าจะกรณีใดก็ตาม เหตุผล:

1. ถ้าฐานข้อมูลรั่วไหล (data breach) — ซึ่งเกิดขึ้นได้เสมอไม่ว่าจะป้องกันดีแค่ไหน —
   รหัสผ่านผู้ใช้ทุกคนจะถูกเปิดเผยทันที
2. คนส่วนใหญ่ใช้รหัสผ่านเดียวกันซ้ำในหลายเว็บ — รหัสผ่านที่รั่วจากเว็บเราอาจถูกเอาไปลอง
   เข้าเว็บอื่น (banking, email) ของผู้ใช้คนเดียวกันได้ (เรียกว่า **credential stuffing**)
3. แม้แต่ทีมพัฒนา/แอดมินของเราเองก็ไม่ควรรู้รหัสผ่านจริงของผู้ใช้ได้

วิธีแก้คือ **hash function แบบทางเดียว (one-way hash)**: แปลงรหัสผ่านเป็นสตริงยาวๆ
ที่ไม่มีทางย้อนกลับไปเป็นรหัสผ่านเดิมได้ (ต่างจาก encryption ที่ decrypt กลับได้ถ้ามี key)
ตอน login เราจะไม่ "ถอดรหัส" อะไรเลย แต่จะ **hash รหัสผ่านที่ผู้ใช้พิมพ์มาใหม่อีกครั้ง
แล้วเทียบกับ hash ที่เก็บไว้** ว่าตรงกันหรือไม่

### `bcrypt` — algorithm มาตรฐานสำหรับ hash รหัสผ่าน

`bcrypt` คือ hashing algorithm ที่ออกแบบมาเฉพาะสำหรับรหัสผ่านโดยเฉพาะ (ต่างจาก
`SHA256`/`MD5` ที่ออกแบบมาสำหรับ checksum ทั่วไป **ห้ามเอามาใช้ hash รหัสผ่าน**) จุดเด่น
ของ bcrypt:

- มี **salt** แบบสุ่มฝังอยู่ในผลลัพธ์ hash เอง ทำให้รหัสผ่านที่เหมือนกันของ user 2 คน
  ได้ hash ที่ต่างกันเสมอ (ป้องกัน rainbow table attack)
- ปรับ **cost factor** ได้ (ยิ่งสูงยิ่งช้า) — ความช้านี้คือจุดประสงค์ ไม่ใช่บั๊ก
  เพราะทำให้การ brute-force เดารหัสผ่านทีละล้านครั้งต่อวินาทีเป็นไปไม่ได้ในทางปฏิบัติ

### เพิ่ม gem `bcrypt`

สร้างโปรเจกต์สาธิตใหม่ (เหมือนที่ทำใน Part 023 — เพื่อโฟกัสที่กลไก authentication ล้วนๆ):

```bash
rails new auth_manual -d sqlite3
cd auth_manual
```

เปิด `Gemfile` — จะเห็นว่า Rails **เตรียมบรรทัดนี้ไว้ให้แบบ comment ในทุกโปรเจกต์ใหม่
อยู่แล้ว** ตั้งแต่ `rails new` (เป็นนัยว่า Rails คาดหวังว่าเราจะต้องใช้มันสักวัน):

```ruby
# Use Active Model has_secure_password [https://guides.rubyonrails.org/active_model_basics.html#securepassword]
gem "bcrypt", "~> 3.1.7"
```

ลบเครื่องหมาย `#` ออกแล้ว `bundle install`:

```bash
bundle install
```

### สร้าง User model

```bash
bin/rails generate model User email:string password_digest:string
```

ผลลัพธ์จริง:

```
      invoke  active_record
      create    db/migrate/20260926062943_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml
```

สังเกตชื่อ column: **`password_digest`** ไม่ใช่ `password` — นี่คือธรรมเนียมที่
`has_secure_password` (ที่จะเพิ่มในขั้นต่อไป) บังคับใช้ชื่อนี้เป๊ะๆ (`digest` แปลว่า
"ผลลัพธ์จากการ hash") ห้ามตั้งชื่ออื่น มิฉะนั้น `has_secure_password` จะหา column ไม่เจอ

แก้ migration ให้รัดกุมขึ้นก่อน migrate (`null: false` และ unique index บน email
เพื่อกันอีเมลซ้ำระดับฐานข้อมูล ไม่ใช่แค่ระดับ Rails validation):

```ruby
# db/migrate/xxxxxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :password_digest, null: false

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

```bash
bin/rails db:create db:migrate
```

### เปิดใช้งาน `has_secure_password`

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
end
```

หนึ่งบรรทัด `has_secure_password` นี้เพิ่มความสามารถให้ `User` ทันที 4 อย่าง:

1. **virtual attribute** `password` และ `password_confirmation` — ไม่ใช่ column จริง
   ในฐานข้อมูล แต่เป็น attribute ชั่วคราวที่ใช้ตอนรับค่าจากฟอร์ม แล้วแปลงเป็น
   `password_digest` (bcrypt hash) ให้อัตโนมัติก่อนบันทึก
2. **validation อัตโนมัติ 3 ข้อ** (เปิดใช้เสมอ เว้นแต่ปิดด้วย `validations: false`):
   - `password` ต้องไม่ว่างตอนสร้าง record ใหม่
   - ถ้ามีการส่ง `password_confirmation` มา ต้องตรงกับ `password`
   - `password` ต้องยาวไม่เกิน 72 ไบต์ (ข้อจำกัดของ bcrypt เอง ถ้ายาวกว่านี้ตัวอักษร
     ส่วนเกินจะถูกตัดทิ้งเฉยๆ โดยไม่แจ้งเตือน ซึ่งอันตราย จึงบังคับ validate ไว้ก่อน)
3. **method `authenticate(password)`** — รับรหัสผ่าน plaintext มาเทียบกับ
   `password_digest` ที่เก็บไว้ คืนค่า `self` (object ของ user) ถ้าตรงกัน หรือ `false`
   ถ้าไม่ตรง
4. **password reset token แบบอัตโนมัติ** (ฟีเจอร์ใหม่ที่เพิ่งมาใน Rails 8) — จะเจาะลึกใน
   Step 408

### ทดลองใน `bin/rails console`

```bash
bin/rails console
```

```irb
irb> u = User.new(email: "test@example.com")
irb> u.valid?
=> false
irb> u.errors.full_messages
=> ["Password can't be blank"]

irb> u2 = User.new(email: "test2@example.com", password: "short", password_confirmation: "different")
irb> u2.valid?
=> false
irb> u2.errors.full_messages
=> ["Password confirmation doesn't match Password"]

irb> u3 = User.create!(email: "test3@example.com", password: "secret123", password_confirmation: "secret123")
irb> u3.password_digest
=> "$2a$12$kHf.I1l1iWaxT...(ตัดให้สั้นลง)..."

irb> u3.authenticate("secret123")
=> #<User id: 1, email: "test3@example.com", ...>   # ตรง → คืน object ของ user

irb> u3.authenticate("wrongpass")
=> false                                             # ไม่ตรง → คืน false

irb> long_pw = "a" * 80
irb> u4 = User.new(email: "test4@example.com", password: long_pw, password_confirmation: long_pw)
irb> u4.valid?
=> false
irb> u4.errors.full_messages
=> ["Password is too long"]
```

ผลลัพธ์ทั้งหมดข้างบนคือผลลัพธ์จริงที่ได้จากการรันบน Rails 8.1.4 — สังเกต 3 อย่าง:

1. **`password_digest` ขึ้นต้นด้วย `$2a$12$`** — นี่คือรูปแบบมาตรฐานของ bcrypt hash:
   `$2a$` บอก algorithm version, `12` คือ **cost factor** (ค่า default ของ Rails)
   ตามด้วย salt และ hash ที่เข้ารหัสด้วย base64 แบบพิเศษของ bcrypt — ความยาวคงที่เสมอ
   ไม่ว่ารหัสผ่านต้นฉบับจะยาวแค่ไหน
2. **`authenticate` คืนค่าที่เป็น truthy object หรือ `false`** ไม่ใช่ `true`/`false`
   ธรรมดา — ทำให้เขียน `user.authenticate(password)` เป็นเงื่อนไขใน `if` ได้ตรงๆ และยัง
   เอา object นั้นไปใช้ต่อได้เลย (จะเห็นประโยชน์ชัดเจนใน Step 404)
3. **ไม่มีทางเรียก `u3.password_digest` แล้วได้รหัสผ่านต้นฉบับ `"secret123"` กลับมา**
   เพราะ hash เป็นการเข้ารหัสทางเดียว — นี่คือหลักประกันความปลอดภัยที่พูดถึงตอนต้น Step นี้

> **ข้อควรระวัง:** `has_secure_password` ต้องมี gem `bcrypt` เท่านั้น ห้ามใช้ gem hash
> แบบอื่น (เช่น `digest`, `openssl` แบบ SHA256 ตรงๆ) มา hash รหัสผ่านแทน เพราะขาดเรื่อง
> salt อัตโนมัติและ cost factor ที่ปรับได้ ซึ่งเป็นหัวใจของความปลอดภัยรหัสผ่านยุคปัจจุบัน

---

## Step 403: สร้างระบบ Signup ด้วยมือ — `RegistrationsController` และ validation จาก `has_secure_password`

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :registrations, only: [:new, :create]
  get "login", to: "sessions#new"
  post "login", to: "sessions#create"
  delete "logout", to: "sessions#destroy"
  get "dashboard", to: "dashboard#index"
  root "sessions#new"
  # ... (route อื่นๆ ที่ Rails generate ให้ตอน rails new)
end
```

### `RegistrationsController`

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      session[:user_id] = @user.id
      redirect_to dashboard_path, notice: "สมัครสมาชิกสำเร็จ ยินดีต้อนรับ!"
    else
      flash.now[:alert] = "สมัครสมาชิกไม่สำเร็จ กรุณาตรวจสอบข้อมูล"
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

จุดที่ควรสังเกต (ทั้งหมดเป็นแนวคิดที่เรียนมาแล้วจากเฟส 3–4):

- `user_params` ใช้ **strong parameters** (`.require.permit`) ตามที่เรียนใน Part 031 —
  อนุญาตเฉพาะ `email`, `password`, `password_confirmation` เท่านั้น ห้ามมี field อื่น
  หลุดเข้ามา (เช่นสมมติว่าในอนาคตเพิ่ม column `admin:boolean` ให้ User — ถ้าไม่ระบุ
  `permit` ไว้ชัดเจน ผู้ใช้จะไม่มีทาง mass-assign ตัวเองเป็น admin ผ่านฟอร์มได้)
- `if @user.save ... else ... end` คือรูปแบบมาตรฐานเดียวกับที่เขียน `create` action
  ของ Model ไหนก็ตามตั้งแต่ Part 026 (เรื่อง validation) — **ไม่มีอะไรพิเศษสำหรับ User
  เลย** มันก็คือ record ธรรมดาที่บังเอิญมี validation จาก `has_secure_password`
  ติดมาด้วย
- ทันทีที่สมัครสำเร็จ เราตั้ง `session[:user_id] = @user.id` เลย — เป็นธรรมเนียมปฏิบัติ
  มาตรฐาน (สมัครเสร็จควร login อัตโนมัติ ไม่บังคับให้ไปกรอกฟอร์ม login ซ้ำอีกรอบ)
- `render :new, status: :unprocessable_entity` ใช้ `flash.now` (ไม่ใช่ `flash`) เพราะ
  เป็นการ `render` ในการ request เดิม ไม่ใช่ `redirect_to` — ตรงตามกฎที่เรียนใน
  Part 023 Step 226 ทุกประการ

### View

```erb
<%# app/views/registrations/new.html.erb %>
<h1>สมัครสมาชิก</h1>

<%= tag.div(flash[:alert], style: "color:red") if flash[:alert] %>

<%= form_with model: @user, url: registrations_path do |form| %>
  <% if @user.errors.any? %>
    <ul style="color:red">
      <% @user.errors.full_messages.each do |msg| %><li><%= msg %></li><% end %>
    </ul>
  <% end %>
  <%= form.email_field :email, placeholder: "email" %><br>
  <%= form.password_field :password, placeholder: "password" %><br>
  <%= form.password_field :password_confirmation, placeholder: "confirm password" %><br>
  <%= form.submit "สมัครสมาชิก" %>
<% end %>
```

### ทดสอบด้วย `curl`

บูต server แล้วยิง request จริง (ต้องดึง CSRF token จากฟอร์มมาก่อนเสมอ ไม่งั้นจะได้
`ActionController::InvalidAuthenticityToken` ตามหลักการ CSRF protection ที่พูดถึงใน
Part 023 Step 224):

```bash
# ดึง CSRF token จากหน้าฟอร์มก่อน (พร้อมเก็บ cookie ของ session ใหม่)
curl -c cookies.txt http://localhost:3000/registrations/new -o signup.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' signup.html | sed -E 's/.*value="([^"]*)"/\1/')

# สมัครสมาชิกจริง
curl -i -c cookies.txt -b cookies.txt -X POST http://localhost:3000/registrations \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[email]=manual@example.com" \
  --data-urlencode "user[password]=secret123" \
  --data-urlencode "user[password_confirmation]=secret123"
```

ผลลัพธ์จริง:

```
HTTP/1.1 302 Found
location: http://localhost:3000/dashboard
```

`302` พร้อม `Location` ชี้ไปหน้า `dashboard` — สมัครสำเร็จและ login ให้อัตโนมัติตามที่
ตั้งใจไว้ (หน้า `dashboard` เรายังไม่สร้าง จะสร้างใน Step 404 คู่กับ `require_authentication`)

ทดสอบซ้ำด้วยอีเมลเดิม (ยืนยัน uniqueness validation):

```irb
irb> u = User.new(email: "manual@example.com", password: "secret123", password_confirmation: "secret123")
irb> u.valid?
=> false
irb> u.errors.full_messages
=> ["Email has already been taken"]
```

---

## Step 404: สร้างระบบ Login/Logout ด้วยมือ — `session[:user_id]`, `current_user`, `require_authentication`

### แนวคิดหลัก: session เก็บแค่ `user.id` ไม่เก็บทั้ง object

จาก Part 023 Step 225 เรารู้แล้วว่า `session` เป็น cookie-based store ที่เข้ารหัส/เซ็นชื่อ
ไว้ แต่ยังมีข้อจำกัดเรื่องขนาด (cookie ปกติจำกัดไม่เกิน 4KB) — จึงไม่ควรเก็บข้อมูล user
ทั้ง object ลงไป **เก็บแค่ `user.id` (ตัวเลขเดียว) เป็นพอ** แล้วทุกครั้งที่ต้องใช้ข้อมูล
user เต็มๆ ค่อย query จากฐานข้อมูลด้วย id นั้นอีกที

### `current_user` และ `require_authentication` ใน `ApplicationController`

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  allow_browser versions: :modern

  private

  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user

  def require_authentication
    return if current_user

    session[:return_to_after_authenticating] = request.url
    redirect_to login_path, alert: "กรุณาเข้าสู่ระบบก่อนเข้าใช้งานหน้านี้"
  end
end
```

**อธิบายทีละจุด:**

- `@current_user ||= ...` คือรูปแบบ **memoization** ที่เรียนมาตั้งแต่เฟส 1–2 (ป้องกันไม่ให้
  query ฐานข้อมูลซ้ำหลายครั้งในหนึ่ง request ถ้ามีการเรียก `current_user` มากกว่า 1 จุด
  เช่นทั้งใน controller และใน view)
- `User.find_by(id: ...)` (ไม่ใช่ `User.find(...)`) เพราะ `find_by` คืน `nil` เมื่อไม่เจอ
  แทนที่จะ raise `ActiveRecord::RecordNotFound` — เหมาะกับกรณีนี้เพราะ "ไม่มี user login
  อยู่" เป็นสถานะปกติที่ต้องรองรับ ไม่ใช่ error
- `helper_method :current_user` ทำให้เรียก `current_user` ในไฟล์ view (`.erb`) ได้ด้วย
  ไม่ใช่แค่ใน controller (by default ไฟล์ view เข้าถึง private method ของ controller
  ไม่ได้ ต้องประกาศผ่าน `helper_method` ก่อนเสมอ)
- `require_authentication` ใช้กลไก **เดียวกับ** `require_token` จาก Part 023 Step 228
  เป๊ะๆ: ถ้าเงื่อนไขผ่าน (`current_user` มีค่า) ก็ `return` เฉยๆ ปล่อยให้ callback chain
  เดินหน้าต่อ ถ้าไม่ผ่านก็ `redirect_to` ซึ่งจะ **halt the filter chain** ทันที
- `session[:return_to_after_authenticating] = request.url` เก็บ URL ที่ผู้ใช้ตั้งใจจะเข้า
  ไว้ก่อน login เพื่อให้หลัง login สำเร็จ **เด้งกลับไปหน้าเดิม** แทนที่จะไปหน้า dashboard
  เสมอ (ประสบการณ์ผู้ใช้ที่ดีกว่า) — ใช้ตอน `create` ใน `SessionsController` ด้านล่าง

### `SessionsController`

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def new
  end

  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to (session.delete(:return_to_after_authenticating) || dashboard_path),
                  notice: "เข้าสู่ระบบสำเร็จ"
    else
      flash.now[:alert] = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      render :new, status: :unprocessable_entity
    end
  end

  def destroy
    reset_session
    redirect_to root_path, notice: "ออกจากระบบเรียบร้อยแล้ว"
  end
end
```

**อธิบายทีละจุด:**

- `user&.authenticate(params[:password])` — ใช้ **safe navigation operator `&.`**
  (เรียนมาตั้งแต่เฟส 1) เพื่อรองรับกรณี `find_by` คืน `nil` (ไม่มีอีเมลนี้ในระบบ)
  ถ้าเขียน `user.authenticate(...)` ตรงๆ โดยไม่มี `&.` จะได้ `NoMethodError` ทันทีเมื่อ
  กรอกอีเมลผิด
- **จุดสำคัญด้านความปลอดภัย:** ข้อความ error ใช้คำว่า **"อีเมลหรือรหัสผ่านไม่ถูกต้อง"**
  แบบรวมๆ ไม่บอกแยกว่า "ไม่มีอีเมลนี้ในระบบ" หรือ "รหัสผ่านผิด" เพราะถ้าบอกแยก จะทำให้
  ผู้ไม่หวังดีรู้ได้ว่าอีเมลไหน "มีอยู่จริง" ในระบบ (เรียกว่า **user enumeration
  vulnerability**) แล้วเอาไปใช้โจมตีต่อ (เช่น ลองสุ่มรหัสผ่านเฉพาะกับอีเมลที่ยืนยันแล้วว่า
  มีอยู่จริง)
- `session.delete(:return_to_after_authenticating) || dashboard_path` — `delete` ทั้งอ่าน
  ค่าและลบ key ออกจาก session ในคำสั่งเดียว (ใช้ครั้งเดียวแล้วทิ้ง เหมือนหลักการของ
  `flash` ใน Part 023 Step 226) ถ้าไม่มีค่านี้ (เช่น login จากหน้า `/login` ตรงๆ ไม่ได้
  ถูก redirect มาจากหน้าอื่น) จะได้ `nil` แล้ว fallback ไปที่ `dashboard_path`
- `destroy` เรียก **`reset_session`** ซึ่งเป็น method ของ Rails ที่ล้างข้อมูล session
  **ทั้งหมด** (ไม่ใช่แค่ `session[:user_id]`) — เคยเห็นมาแล้วใน Part 023 Step 225
  เป็นวิธีที่ถูกต้องที่สุดสำหรับ logout เพราะล้างค่าที่อาจค้างอยู่ทุกตัว ไม่ใช่แค่ค่าที่
  เรานึกออก

### `DashboardController` — ตัวอย่างหน้าที่ต้อง login ก่อนถึงเข้าได้

```ruby
# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  before_action :require_authentication

  def index
    render plain: "สวัสดี #{current_user.email}! นี่คือหน้าที่เข้าได้เฉพาะผู้ login แล้วเท่านั้น"
  end
end
```

หนึ่งบรรทัด `before_action :require_authentication` นี้เองที่ทำให้ controller นี้กลาย
เป็น "protected" — รูปแบบเดียวกับที่ Part 023 บอกไว้ล่วงหน้าว่าจะได้เห็นใน "เฟส 5"

### ทดสอบด้วย `curl` ครบวงจร

```bash
# 1) เข้าหน้า dashboard โดยยังไม่ได้ login
curl -i http://localhost:3000/dashboard
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/login
```

`require_authentication` ทำงาน — ไม่มี session ก็ถูกเด้งไปหน้า login ทันที

```bash
# 2) login ด้วยรหัสผ่านผิด (ดึง CSRF token จากหน้า login ก่อนเสมอ)
curl -c cookies.txt http://localhost:3000/login -o login.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' login.html | sed -E 's/.*value="([^"]*)"/\1/')

curl -i -c cookies.txt -b cookies.txt -X POST http://localhost:3000/login \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "email=manual@example.com" \
  --data-urlencode "password=wrongpass"
```

ผลลัพธ์จริง: **`422 Unprocessable Content`** พร้อม body ที่มีข้อความ
"อีเมลหรือรหัสผ่านไม่ถูกต้อง" ปรากฏอยู่ (มาจาก `flash.now[:alert]` + `render :new` ตามที่
เขียนไว้ — ไม่ redirect เพราะยังไม่มีอะไรสำเร็จ)

```bash
# 3) login ด้วยรหัสผ่านถูกต้อง
curl -c cookies2.txt http://localhost:3000/login -o login2.html
TOKEN2=$(grep -o 'name="authenticity_token" value="[^"]*"' login2.html | sed -E 's/.*value="([^"]*)"/\1/')

curl -i -c cookies2.txt -b cookies2.txt -X POST http://localhost:3000/login \
  --data-urlencode "authenticity_token=$TOKEN2" \
  --data-urlencode "email=manual@example.com" \
  --data-urlencode "password=secret123"
```

ผลลัพธ์จริง:

```
HTTP/1.1 302 Found
location: http://localhost:3000/dashboard
```

```bash
# 4) เข้าหน้า dashboard อีกครั้งด้วย cookie ของ session ที่ login แล้ว
curl -b cookies2.txt http://localhost:3000/dashboard
```

```
สวัสดี manual@example.com! นี่คือหน้าที่เข้าได้เฉพาะผู้ login แล้วเท่านั้น
```

```bash
# 5) logout
curl -c cookies2.txt http://localhost:3000/login -o login3.html
CSRF=$(grep -o '<meta name="csrf-token" content="[^"]*"' login3.html | sed -E 's/.*content="([^"]*)"/\1/')

curl -i -c cookies2.txt -b cookies2.txt -X DELETE http://localhost:3000/logout \
  -H "X-CSRF-Token: $CSRF"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/
```

```bash
# 6) เข้าหน้า dashboard อีกครั้งหลัง logout — ต้องถูกเด้งกลับไปหน้า login
curl -i -b cookies2.txt http://localhost:3000/dashboard
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/login
```

ครบวงจร signup → login (ผิด/ถูก) → เข้าหน้า protected → logout → เข้าหน้า protected อีก
ครั้ง (ถูกเด้ง) ตามที่ตั้งใจไว้ทุกจุด

> **หมายเหตุสำคัญเรื่อง CSRF กับ `curl -X DELETE`:** สังเกตว่าขั้นตอน logout (ข้อ 5)
> ใช้วิธีส่ง CSRF token ผ่าน **HTTP header** (`-H "X-CSRF-Token: ..."`) แทนที่จะส่งเป็น
> body parameter เหมือนตัวอย่างอื่น — เพราะทดสอบจริงแล้วพบว่าเมื่อยิง HTTP verb ที่ไม่ใช่
> `GET`/`POST` ตรงๆ ด้วย `curl -X DELETE` พร้อม body parameter `authenticity_token` จะได้
> `ActionController::InvalidAuthenticityToken` แต่พอเปลี่ยนไปส่งผ่าน header กลับผ่านได้
> ปกติ — เรื่องนี้ไม่ใช่บั๊ก แต่เป็นเพราะในความเป็นจริง browser ไม่เคยส่ง HTTP `DELETE`
> ตรงๆ จากฟอร์ม HTML เลย (ฟอร์ม HTML รองรับแค่ `GET`/`POST`) ปุ่ม logout ที่ใช้
> `button_to ..., method: :delete` หรือ Turbo ในเว็บจริงจึงส่งเป็น **`POST` พร้อม header
> `X-CSRF-Token`** ผ่าน JavaScript fetch เบื้องหลัง (Turbo) แล้วให้ Rails router แปลความ
> เป็น `DELETE` อีกที — พฤติกรรมที่เราเจอตอนทดสอบด้วย `curl` จึงเป็นภาพสะท้อนของกลไกจริง
> ที่ Rails ใช้ทำงานร่วมกับ Turbo อยู่แล้ว

ทีนี้เราก็มีระบบ authentication แบบ session-based ที่ทำงานได้ครบวงจรจริง โดยเขียนโค้ด
เองล้วนๆ ไม่ถึง 60 บรรทัด (ไม่นับ view) — เข้าใจกลไกทุกจุดแล้ว มาดูกันว่า Rails 8 มี
เครื่องมือที่ทำสิ่งเดียวกันนี้ให้แบบ production-ready ในคำสั่งเดียวได้อย่างไร

---

## Step 405: รู้จัก `bin/rails generate authentication` — ฟีเจอร์ใหม่ของ Rails 8

### ทำไมต้องมี generator ตัวนี้

ระบบที่เราสร้างมือใน Step 402–404 นั้น **ใช้งานได้จริง** แต่ยังขาดหลายอย่างที่ระบบ
production จริงควรมี เช่น:

- **Password reset** (ลืมรหัสผ่าน) — ยังไม่มีเลย
- **Rate limiting** การพยายาม login ผิดซ้ำๆ (ป้องกัน brute-force attack)
- **หลาย session ต่อ user คนเดียว** (เช่น login พร้อมกันทั้งมือถือและคอม แล้ว "logout
  จากทุกอุปกรณ์" ได้)
- **ป้องกัน timing attack** ตอนเช็คอีเมล/รหัสผ่าน (จะอธิบายละเอียดใน Step 407)

เขียนทั้งหมดนี้เองได้ แต่ใช้เวลาและความรู้เฉพาะทางด้าน security พอสมควร — Rails 8
(เปิดตัวปลายปี 2024) จึงเพิ่ม generator ชื่อ **`authentication`** เข้ามาในตัว framework
เอง (ไม่ต้องติดตั้ง gem เพิ่ม) เพื่อสร้างระบบ auth ที่ครอบคลุมเรื่องเหล่านี้ให้อัตโนมัติ
เป็นฟีเจอร์ที่ **ค่อนข้างใหม่มาก** (มาพร้อม Rails 8.0 ปลายปี 2024) ตำราหรือบทความเก่าๆ
ก่อนหน้านั้นจะไม่พูดถึงเลย

ตรวจสอบว่ามี generator นี้จริง:

```bash
bin/rails generate authentication --help
```

ผลลัพธ์จริงจาก Rails 8.1.4:

```
Usage:
  bin/rails generate authentication [options]

Options:
      [--skip-namespace]                 # Skip namespace (affects only isolated engines)
                                         # Default: false
      [--skip-collision-check]           # Skip collision check
                                         # Default: false
      [--api], [--no-api], [--skip-api]  # Generate API-only controllers and models, with no view templates
                                         # Default: false
  -e, [--template-engine=NAME]           # Template engine to be invoked
                                         # Default: erb
  -t, [--test-framework=NAME]            # Test framework to be invoked
                                         # Default: test_unit

Description:
    Generates a basic authentication system with users, sessions, and password reset.
```

บรรทัดสุดท้าย ("Generates a basic authentication system with users, sessions, and
password reset") สรุปตรงตัวว่ามันทำอะไร — ตรงกับสิ่งที่เราเพิ่งสร้างมือ **บวกกับ
password reset** ที่เรายังไม่ได้ทำ

### รัน generator จริง

```bash
rails new auth_generated -d sqlite3
cd auth_generated

# เปิด comment gem bcrypt ใน Gemfile ก่อน (generator ไม่ทำให้อัตโนมัติตอนนี้ ต้องทำเอง
# ก่อนรัน — แต่ generator จะ gsub เพิ่มบรรทัดให้ในบางเวอร์ชัน ถ้ายังไม่มีก็เพิ่มเองได้)
bundle install

bin/rails generate authentication
```

ผลลัพธ์จริง (ทดสอบบน Rails 8.1.4):

```
      invoke  erb
      create    app/views/passwords/new.html.erb
      create    app/views/passwords/edit.html.erb
      create    app/views/sessions/new.html.erb
      create  app/models/session.rb
      create  app/models/user.rb
      create  app/models/current.rb
      create  app/controllers/sessions_controller.rb
      create  app/controllers/concerns/authentication.rb
      create  app/controllers/passwords_controller.rb
      create  app/channels/application_cable/connection.rb
      create  app/mailers/passwords_mailer.rb
      create  app/views/passwords_mailer/reset.html.erb
      create  app/views/passwords_mailer/reset.text.erb
      insert  app/controllers/application_controller.rb
       route  resources :passwords, param: :token
       route  resource :session
        gsub  Gemfile
         run  bundle install --quiet
    generate  migration
       rails  generate migration CreateUsers email_address:string!:uniq password_digest:string! --force
      invoke  active_record
      create    db/migrate/20260926062330_create_users.rb
    generate  migration
       rails  generate migration CreateSessions user:references ip_address:string user_agent:string --force
      invoke  active_record
      create    db/migrate/20260926062331_create_sessions.rb
      invoke  test_unit
      create    test/fixtures/users.yml
      create    test/models/user_test.rb
      create    test/controllers/sessions_controller_test.rb
      create    test/controllers/passwords_controller_test.rb
      create    test/mailers/previews/passwords_mailer_preview.rb
      create    test/test_helpers/session_test_helper.rb
      insert    test/test_helper.rb
```

ในคำสั่งเดียว generator นี้:

1. **`gsub` Gemfile** เพื่อเปิดใช้ `bcrypt` เอง (ไม่ต้องแก้มือ) แล้วรัน `bundle install`
   ให้ทันที
2. สร้าง **3 model**: `User`, `Session`, `Current`
3. สร้าง **2 controller**: `SessionsController`, `PasswordsController` และ **1 concern**:
   `Authentication`
4. สร้าง **1 mailer** (`PasswordsMailer`) พร้อม view สำหรับอีเมล (ทั้งแบบ HTML และ text)
5. สร้าง **3 view**: `sessions/new`, `passwords/new`, `passwords/edit`
6. เพิ่ม route ให้อัตโนมัติ: `resource :session` และ
   `resources :passwords, param: :token`
7. `insert` โค้ดเพิ่มเข้าไปใน `ApplicationController` ที่มีอยู่แล้ว (ไม่ทับไฟล์เดิม)
8. สร้าง **2 migration** สำหรับตาราง `users` และ `sessions`
9. สร้าง test file ให้ครบ (model test, controller test, mailer preview)

สังเกตว่า **ไม่มี `RegistrationsController` หรือหน้า signup ให้เลย** — generator นี้ทำ
แค่ "login, logout, password reset" (ตรงกับคำอธิบายใน `--help`) ส่วนหน้าสมัครสมาชิกเป็น
เรื่องเฉพาะของแต่ละแอปที่ต้องสร้างเอง (เราจะสร้างในแบบฝึกหัด Step 410 — เอาความรู้จาก
Step 403 มาต่อยอดใส่เข้าไปในระบบที่ generator สร้างให้)

migrate ฐานข้อมูล:

```bash
bin/rails db:create db:migrate
```

```
== 20260926062330 CreateUsers: migrating ======================================
-- create_table(:users)
-- add_index(:users, :email_address, {:unique=>true})
== 20260926062330 CreateUsers: migrated (0.0074s) =============================

== 20260926062331 CreateSessions: migrating ===================================
-- create_table(:sessions)
== 20260926062331 CreateSessions: migrated (0.0137s) ==========================
```

---

## Step 406: อ่านโค้ดที่ generator สร้าง (1) — `User`, `Session`, `Current`, และ `Authentication` concern

### `app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

เทียบกับ `User` ที่เราเขียนมือใน Step 402:

| จุดต่าง | เวอร์ชันมือ (Step 402) | เวอร์ชัน generator |
|---|---|---|
| ชื่อ column อีเมล | `email` | **`email_address`** (ชัดเจนกว่าว่าเก็บ "ที่อยู่อีเมล" ไม่ใช่แค่ "อีเมล" เฉยๆ) |
| ความสัมพันธ์กับ session | ไม่มี (session เก็บแค่ `user_id` ใน cookie) | **`has_many :sessions`** — session ถูกเก็บเป็น **record ในฐานข้อมูล** ไม่ใช่แค่ค่าดิบใน cookie |
| การ normalize อีเมล | ไม่มี | **`normalizes :email_address, with: ->(e) { e.strip.downcase }`** |
| Validation | `validates :email, presence: true, uniqueness: true` เขียนเอง | **ไม่มีบรรทัด validate uniqueness ให้เห็นเลย** เพราะบังคับด้วย unique index ระดับฐานข้อมูลแทน (จะอธิบายต่อ) |

**`normalizes`** เป็น API ของ Active Record (Rails 7.1+) ที่ทำให้ค่าถูก "ปรับให้เป็น
มาตรฐานเดียวกัน" ทุกครั้งก่อนบันทึกและก่อนใช้ query — `e.strip.downcase` ตัดช่องว่างหัว
ท้ายและแปลงเป็นตัวพิมพ์เล็กทั้งหมด ทำให้ `"Test@Example.com "` กับ `"test@example.com"`
ถูกมองว่าเป็นอีเมลเดียวกันเสมอ (ทั้งตอนบันทึกและตอนค้นหาด้วย `find_by(email_address:
...)`) — เป็นจุดที่เวอร์ชันมือของเรายังไม่ได้ทำ (บั๊กที่พบบ่อยในระบบ auth จริง: ผู้ใช้
สมัครด้วย `Test@example.com` แต่ login ด้วย `test@example.com` แล้วหา user ไม่เจอ)

**`has_many :sessions, dependent: :destroy`** คือความต่างที่สำคัญที่สุด: session ใน
เวอร์ชัน generator ไม่ได้เก็บแค่ `user_id` ใน cookie เฉยๆ แต่มี **ตาราง `sessions` ใน
ฐานข้อมูลจริง** เก็บ 1 record ต่อ "การ login ครั้งหนึ่ง" — เดี๋ยวจะเห็นประโยชน์ของ
การออกแบบแบบนี้ใน Step 409 (เรื่อง remember-me และ logout ทุกอุปกรณ์)

### `app/models/session.rb`

```ruby
class Session < ApplicationRecord
  belongs_to :user
end
```

ง่ายมาก — เป็นแค่ Model ธรรมดาที่ `belongs_to :user` (association พื้นฐานจาก Part 027)
migration ของมันคือ:

```ruby
class CreateSessions < ActiveRecord::Migration[8.1]
  def change
    create_table :sessions do |t|
      t.references :user, null: false, foreign_key: true
      t.string :ip_address
      t.string :user_agent

      t.timestamps
    end
  end
end
```

เก็บ `ip_address` และ `user_agent` ของอุปกรณ์ที่ login ไว้ด้วย — เป็นข้อมูลที่มีประโยชน์
มากสำหรับหน้า "อุปกรณ์ที่ login อยู่" (แบบที่เห็นใน Gmail, Facebook) ที่ให้ผู้ใช้ดูว่า
login อยู่ที่ไหนบ้าง และเลือก logout เฉพาะอุปกรณ์ได้

### `app/models/current.rb`

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

นี่คือคลาสที่ใช้ `ActiveSupport::CurrentAttributes` — กลไกของ Rails ที่เก็บข้อมูลแบบ
**per-request, thread-safe, global-like** (ไม่ใช่ global variable ธรรมดาที่อันตราย
เพราะอาจรั่วข้าม request/thread ได้ แต่ `CurrentAttributes` ถูกออกแบบมาให้ Rails ล้างค่า
ให้อัตโนมัติทุกจบ request) แนวคิดคือ: แทนที่จะต้องพก `current_user` ผ่าน parameter ไปทุก
method/class ที่ต้องใช้ (เช่น background job, service object) เราเรียก `Current.user`
จากที่ไหนก็ได้ในระหว่าง request นั้นๆ

`delegate :user, to: :session, allow_nil: true` หมายความว่า `Current.user` จริงๆ แล้ว
เท่ากับ `Current.session&.user` — ดึง user จาก session ที่ผูกไว้อีกที (ถ้า
`Current.session` เป็น `nil` ก็ได้ `nil` กลับมาแทนที่จะ error เพราะมี `allow_nil: true`)

เทียบกับเวอร์ชันมือ: `Current.user` ในเวอร์ชัน generator ทำหน้าที่เดียวกับ `current_user`
method ที่เราเขียนไว้ใน `ApplicationController` (Step 404) เพียงแต่เก็บเป็น "session
object เต็มๆ" (ซึ่งมี `user_agent`, `ip_address` ติดมาด้วย) ไม่ใช่แค่ `user_id` ดิบๆ

### `app/controllers/concerns/authentication.rb` — หัวใจของทั้งระบบ

```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def authenticated?
      resume_session
    end

    def require_authentication
      resume_session || request_authentication
    end

    def resume_session
      Current.session ||= find_session_by_cookie
    end

    def find_session_by_cookie
      Session.find_by(id: cookies.signed[:session_id]) if cookies.signed[:session_id]
    end

    def request_authentication
      session[:return_to_after_authenticating] = request.url
      redirect_to new_session_path
    end

    def after_authentication_url
      session.delete(:return_to_after_authenticating) || root_url
    end

    def start_new_session_for(user)
      user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
        Current.session = session
        cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
      end
    end

    def terminate_session
      Current.session.destroy
      cookies.delete(:session_id)
    end
end
```

ไฟล์นี้คือหัวใจของระบบทั้งหมด แล้วมันถูกเขียนเป็น **`ActiveSupport::Concern`** (เรียนมา
แล้วใน Part 036) แทนที่จะเขียนตรงใน `ApplicationController` เพื่อให้ **แยกส่วนที่เกี่ยว
กับ authentication ออกเป็นไฟล์ของตัวเอง** — เป็นตัวอย่างการจัดโครงสร้างโค้ดแบบมืออาชีพ
ที่ Part 036 พูดถึงไว้ล่วงหน้าแบบเป๊ะๆ

ไล่เทียบกับเวอร์ชันมือทีละ method:

| Method ในเวอร์ชัน generator | เทียบเท่ากับอะไรในเวอร์ชันมือ (Step 404) | ต่างกันตรงไหน |
|---|---|---|
| `require_authentication` | `require_authentication` | โครงสร้างเหมือนกันเป๊ะ (`resume_session \|\| request_authentication` คือรูปแบบ `return if current_user; redirect...` แบบย่อ) |
| `resume_session` | ส่วนหนึ่งของ `current_user` | เวอร์ชัน generator หา session จาก **cookie ที่เซ็นชื่อ + ตาราง `sessions`** ไม่ใช่หาจาก `session[:user_id]` ตรงๆ |
| `find_session_by_cookie` | `User.find_by(id: session[:user_id])` | ค้นหาผ่าน `Session.find_by(id: cookies.signed[:session_id])` แล้วค่อยได้ `user` ผ่าน `Current.session.user` |
| `start_new_session_for(user)` | `session[:user_id] = user.id` | สร้าง **record ใหม่ในตาราง `sessions`** ก่อน แล้วเก็บแค่ `session.id` นั้นลง cookie (ไม่ใช่ `user.id` ตรงๆ) |
| `terminate_session` | `reset_session` | ลบ record ในตาราง `sessions` ออกจากฐานข้อมูลจริง ไม่ใช่แค่ล้าง cookie |
| `allow_unauthenticated_access` | (เราไม่มี เพราะใช้ `before_action ... only:`/`except:` ตรงๆ) | ใช้ `skip_before_action` เพื่อ "ยกเว้น" บาง action จาก `before_action :require_authentication` ที่ผูกไว้ใน `included do` — จะใช้เยอะมากใน Step 407 |

จุดที่ **ต่างกันเชิงสถาปัตยกรรมที่สำคัญที่สุด** คือเรื่อง **session แบบมีฐานข้อมูล
รองรับ (database-backed session)**:

- เวอร์ชันมือ: cookie เก็บ `user_id` โดยตรง → ต้องการ "logout" ก็แค่ลบค่าออกจาก cookie
  (`reset_session`) วิธีนี้ **logout ได้แค่อุปกรณ์ปัจจุบัน** ไม่มีทาง "logout จากทุก
  อุปกรณ์" ได้ เพราะไม่มีที่เก็บกลางที่รู้ว่ามีกี่ cookie ถืออยู่บ้าง
- เวอร์ชัน generator: cookie เก็บแค่ `session.id` (id ของ record ในตาราง `sessions`)
  → การ logout จริงๆ คือการ **ลบ record นั้นออกจากฐานข้อมูล** (`Current.session.destroy`)
  ทำให้ cookie เดิมที่ค้างอยู่ใน browser (ถ้ามี) กลายเป็น "อ้างอิงถึง session ที่ไม่มีอยู่
  แล้ว" ทันที และที่สำคัญกว่านั้น: เพราะ user 1 คนมีได้หลาย session record (หนึ่งต่อหนึ่ง
  อุปกรณ์/browser) การเรียก `current_user.sessions.destroy_all` (ซึ่งจะเห็นใน Step 408
  ตอน reset รหัสผ่าน) จึงเท่ากับ **"logout จากทุกอุปกรณ์พร้อมกัน"** ได้ในคำสั่งเดียว

`cookies.signed[:session_id]` และ `cookies.signed.permanent[:session_id]` ใช้กลไก
**signed cookie** (เซ็นชื่อด้วย `secret_key_base` เหมือน `session` ปกติที่เรียนใน
Part 023) เพื่อป้องกันไม่ให้ผู้ใช้ปลอมค่า `session_id` ขึ้นมาเองแล้วสวมสิทธิ์เป็นคนอื่น
— แม้แต่ถ้ารู้ id ของ session คนอื่นในฐานข้อมูล ก็ปลอม cookie ที่เซ็นชื่อถูกต้องไม่ได้ถ้า
ไม่รู้ `secret_key_base` ของแอป

---

## Step 407: อ่านโค้ดที่ generator สร้าง (2) — `SessionsController`, `authenticate_by`, `rate_limit` ในตัว

### `app/controllers/sessions_controller.rb`

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_path, alert: "Try again later." }

  def new
  end

  def create
    if user = User.authenticate_by(params.permit(:email_address, :password))
      start_new_session_for user
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Try another email address or password."
    end
  end

  def destroy
    terminate_session
    redirect_to new_session_path, status: :see_other
  end
end
```

3 บรรทัดแรกมีความรู้ใหม่ที่สำคัญมากซ่อนอยู่:

#### 1. `allow_unauthenticated_access only: %i[ new create ]`

เพราะ `Authentication` concern ผูก `before_action :require_authentication` ไว้กับ
**ทุก action ของทุก controller ที่สืบทอดจาก `ApplicationController`** โดย default
(นึกถึง Step 228 เรื่อง `before_action` ไม่ใส่ `only:`/`except:` — รันทุก action)
แปลว่าถ้าไม่ทำอะไรเลย แม้แต่หน้า **login เอง** ก็จะถูกบล็อกไม่ให้เข้า (เพราะยังไม่ login
จะเข้าหน้า login ได้ยังไง ก็ต้อง... login ก่อน — วนลูปไม่จบ!) `allow_unauthenticated_access`
คือ class method ที่ `Authentication` concern เพิ่มให้ (ดู `class_methods do ... end`
ใน Step 406) ทำหน้าที่ `skip_before_action :require_authentication` เฉพาะ action ที่
ระบุ — ในที่นี้คือ `new` (แสดงฟอร์ม) และ `create` (ทำการ login จริง) ส่วน `destroy`
(logout) ยังต้องผ่าน `require_authentication` ตามปกติ (สมเหตุสมผล — logout ได้ก็ต่อเมื่อ
login อยู่แล้วเท่านั้น)

#### 2. `rate_limit to: 10, within: 3.minutes, only: :create, with: -> { ... }`

นี่คือฟีเจอร์ **`rate_limit`** ซึ่งเป็นอีกหนึ่งความสามารถใหม่ที่ Rails 8 เพิ่มเข้ามาใน
`ActionController` โดยตรง (module `ActionController::RateLimiting`) — ไม่ต้องพึ่ง gem
ภายนอกใดๆ สำหรับ **rate limit ระดับ action** สังเคราะห์ความหมายได้ว่า: "จำกัดให้
action `create` นี้ถูกเรียกได้ไม่เกิน **10 ครั้งภายใน 3 นาที** ต่อ 1 หน่วย (ค่า default
ของหน่วยคือ **IP address** ของผู้เรียก) ถ้าเกิน ให้ทำตาม `with:` (redirect กลับไปหน้า
login พร้อม alert "Try again later.") แทนที่จะปล่อยให้ทำงานตามปกติ"

หลักการเก็บตัวนับใช้ `ActiveSupport::Cache` store เดียวกับ `Rails.cache` ของแอป
(ปรับ store อื่นได้ด้วย option `store:`) เราทดสอบจริงโดยการยิง login ด้วยรหัสผ่านผิดซ้ำๆ
เกิน 10 ครั้งภายใน 3 นาทีจาก IP เดียวกัน แล้วสังเกตว่า response เปลี่ยนจาก
`"Try another email address or password."` (ข้อความปกติตอนรหัสผ่านผิด) ไปเป็น
**`"Try again later."`** (ข้อความจาก `rate_limit`) ทันทีที่เกินโควตา — ยืนยันว่า
mechanism ทำงานจริงตามที่ประกาศไว้

> **เทียบกับ rack-attack:** gem `rack-attack` (ที่จะเรียนใน Part 060) ทำงานที่ระดับ
> **Rack middleware** คือดักจับ request ตั้งแต่ก่อนเข้าถึง Rails router เลย เหมาะกับการ
> จำกัด traffic ระดับทั้งแอป/ทั้ง IP อย่างกว้างๆ (เช่น จำกัด API ทั้งหมด, บล็อก IP ที่ยิง
> ถี่ผิดปกติ) ส่วน `rate_limit` ที่เห็นในนี้ทำงานที่ระดับ **controller action เดียว**
> เหมาะกับ "จำกัดเฉพาะ action ที่อ่อนไหว" (เช่น login, signup, password reset) ทั้งสอง
> อย่างใช้ร่วมกันได้ไม่ขัดแย้งกัน — แอป production จริงมักมีทั้งคู่: `rack-attack` เป็น
> เกราะชั้นนอกระดับ IP/ทั้งแอป และ `rate_limit` เป็นเกราะชั้นในเฉพาะจุดที่อ่อนไหวที่สุด

#### 3. `User.authenticate_by(params.permit(:email_address, :password))` — ป้องกัน timing attack

เวอร์ชันมือของเราเขียนว่า:

```ruby
user = User.find_by(email: params[:email])
user&.authenticate(params[:password])
```

ดูเผินๆ เหมือนทำงานเหมือนกับ `authenticate_by` แต่จริงๆ แล้วมีช่องโหว่ด้านความปลอดภัย
ที่ละเอียดอ่อนซ่อนอยู่: **ถ้าอีเมลไม่มีในระบบ โค้ดจะ return ทันทีโดยไม่เรียก
`.authenticate` เลย** (เพราะ `nil&.authenticate(...)` สั้นแบบไม่ทำอะไร) แต่ถ้าอีเมลมีอยู่
จริง โค้ดจะเรียก `.authenticate` ซึ่งภายในรัน bcrypt hash comparison ที่ **ใช้เวลานาน
กว่ามาก** (bcrypt ถูกออกแบบมาให้ช้าโดยตั้งใจ ตามที่อธิบายใน Step 402) ผลคือ **เวลาตอบ
สนองของ request จะต่างกันอย่างมีนัยสำคัญ** ระหว่างกรณี "อีเมลไม่มีในระบบ" กับ "อีเมลมี
แต่รหัสผ่านผิด" — ผู้ไม่หวังดีสามารถวัดเวลาตอบสนอง (timing attack) เพื่อเดาว่าอีเมลไหน
"มีอยู่จริง" ในระบบได้ แม้ข้อความ error จะเหมือนกันทุกตัวอักษรก็ตาม!

มาดู source code จริงของ `authenticate_by` (จาก gem `activerecord`) ที่แก้ปัญหานี้:

```ruby
# active_record/secure_password.rb (Rails 8.1.4)
def authenticate_by(attributes)
  passwords, identifiers = attributes.to_h.partition do |name, value|
    !has_attribute?(name) && has_attribute?("#{name}_digest")
  end.map(&:to_h)

  raise ArgumentError, "One or more password arguments are required" if passwords.empty?
  raise ArgumentError, "One or more finder arguments are required" if identifiers.empty?

  return if passwords.any? { |name, value| value.nil? || value.empty? }

  if record = find_by(identifiers)
    record if passwords.count { |name, value| record.public_send(:"authenticate_#{name}", value) } == passwords.size
  else
    new(passwords)   # <-- จุดสำคัญ: สร้าง object เปล่าแล้วรัน bcrypt เปรียบเทียบไปเปล่าๆ
    nil
  end
end
```

จุดที่ฉลาดที่สุดคือบรรทัด `new(passwords)` ใน branch `else` (กรณีหา user ไม่เจอ) —
แทนที่จะ `return nil` ทันที มันสร้าง `User.new` เปล่าๆ ขึ้นมาก่อน ซึ่งจะไป trigger การ
เซ็ต virtual attribute `password=` ซึ่งเบื้องหลังก็รัน bcrypt hash แม้จะไม่มีการเทียบผล
ลัพธ์จริงก็ตาม — เป็นการ **"เสียเวลาปลอมๆ"** ให้เท่ากับกรณีที่ user มีอยู่จริง ทำให้เวลา
ตอบสนองของทั้งสองกรณี ("อีเมลไม่มี" กับ "อีเมลมีแต่รหัสผ่านผิด") **ใกล้เคียงกันมากจนวัด
ความต่างไม่ได้ในทางปฏิบัติ**

นี่คือตัวอย่างที่ดีมากว่าทำไม **"เขียนเองได้" กับ "เขียนเองอย่างปลอดภัยเทียบเท่า
production-grade code" เป็นคนละเรื่องกัน** — รายละเอียดแบบนี้เป็นสิ่งที่ทีม Rails core
ใส่ใจและ Rails 8 เตรียมไว้ให้ฟรีๆ ผ่าน `authenticate_by`

> **ปรับเวอร์ชันมือให้ปลอดภัยขึ้น:** ถ้าจะใช้เวอร์ชัน manual จาก Step 404 ต่อใน
> production จริง ให้เปลี่ยนมาเรียก `User.authenticate_by(email: params[:email],
> password: params[:password])` แทน `find_by` + `&.authenticate` ตรงๆ — `authenticate_by`
> ใช้ได้กับทุก Active Record model ที่มี `has_secure_password` โดยไม่ต้องพึ่ง generator
> เลย เพราะเป็น method ระดับ framework

### View ที่ generator สร้างให้ (`app/views/sessions/new.html.erb`)

```erb
<%= tag.div(flash[:alert], style: "color:red") if flash[:alert] %>
<%= tag.div(flash[:notice], style: "color:green") if flash[:notice] %>

<%= form_with url: session_path do |form| %>
  <%= form.email_field :email_address, required: true, autofocus: true, autocomplete: "username", placeholder: "Enter your email address", value: params[:email_address] %><br>
  <%= form.password_field :password, required: true, autocomplete: "current-password", placeholder: "Enter your password", maxlength: 72 %><br>
  <%= form.submit "Sign in" %>
<% end %>
<br>

<%= link_to "Forgot password?", new_password_path %>
```

จุดที่น่าสังเกต: `maxlength: 72` บน password field — ตรงกับข้อจำกัดของ bcrypt ที่อธิบาย
ไว้ใน Step 402 พอดี (ป้องกันผู้ใช้พิมพ์รหัสผ่านยาวเกิน 72 ตัวอักษรตั้งแต่ระดับ HTML เลย)
และ `autocomplete: "username"`/`"current-password"` เป็นค่ามาตรฐานที่บอก browser ให้
autofill ถูกช่อง (ไม่เกี่ยวกับ Rails แต่เป็น best practice ของฟอร์ม login ทั่วไป)

---

## Step 408: Password Reset เต็มรูปแบบ — `generates_token_for`, `PasswordsController`, `PasswordsMailer`

### ฟีเจอร์ใหม่ล่าสุด: `has_secure_password` สร้าง password reset token ให้ฟรี

นี่คือรายละเอียดที่ **ใหม่มากแม้แต่สำหรับคนที่รู้จัก Rails 8 มาสักพักแล้ว** — ตรวจสอบ
source code ของ `has_secure_password` (gem `activemodel` เวอร์ชัน 8.1.4) โดยตรง:

```ruby
# active_model/secure_password.rb
DEFAULT_RESET_TOKEN_EXPIRES_IN = 15.minutes

# ... (ภายใน has_secure_password method)
if reset_token && respond_to?(:generates_token_for)
  # ...
  generates_token_for :"#{attribute}_reset", expires_in: reset_token_expires_in do
    public_send(:"#{attribute}_salt")&.last(10)
  end
end
```

พูดง่ายๆ คือ: **แค่เขียน `has_secure_password` เฉยๆ (ไม่ต้องเขียนอะไรเพิ่ม) ถ้าคลาสนั้น
เป็น Active Record** (ซึ่ง `respond_to?(:generates_token_for)` จะเป็นจริงเสมอสำหรับ
Active Record) **Rails จะเรียก `generates_token_for :password_reset, expires_in:
15.minutes` ให้อัตโนมัติ** ทำให้ได้ method เหล่านี้มาฟรีทันที:

```ruby
user.password_reset_token                        # สร้าง token ใหม่ (signed, มีวันหมดอายุฝังอยู่)
User.find_by_password_reset_token(token)          # หา user จาก token — คืน nil ถ้าหมดอายุ/ผิด
User.find_by_password_reset_token!(token)         # เหมือนกัน แต่ raise ActiveSupport::MessageVerifier::InvalidSignature ถ้าผิด/หมดอายุ
```

`generates_token_for` (API ระดับ Active Record ทั่วไป ไม่ได้ผูกกับ password โดยเฉพาะ
— ใช้กับ token วัตถุประสงค์อื่นได้ด้วย เช่น email confirmation token) ทำงานโดยเข้ารหัส
`id` ของ record + "purpose" (ในที่นี้คือ `"password_reset"`) + เวลาหมดอายุ ลงในสตริงที่
เซ็นชื่อไว้ (คล้าย signed cookie ที่เรียนมาแล้ว) และ **ผูกกับค่าบางอย่างของ record ณ
ขณะสร้าง token** (ในที่นี้คือ 10 ตัวท้ายของ `password_salt`) ทำให้ **ถ้ารหัสผ่านถูก
เปลี่ยนไปแล้ว token เก่าที่เคยส่งไปในอีเมลจะใช้ไม่ได้ทันที** แม้จะยังไม่หมดอายุ 15 นาที
ก็ตาม — เป็นการป้องกันอีกชั้นที่ละเอียดมาก

### `app/controllers/passwords_controller.rb`

```ruby
class PasswordsController < ApplicationController
  allow_unauthenticated_access
  before_action :set_user_by_token, only: %i[ edit update ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_password_path, alert: "Try again later." }

  def new
  end

  def create
    if user = User.find_by(email_address: params[:email_address])
      PasswordsMailer.reset(user).deliver_later
    end

    redirect_to new_session_path, notice: "Password reset instructions sent (if user with that email address exists)."
  end

  def edit
  end

  def update
    if @user.update(params.permit(:password, :password_confirmation))
      @user.sessions.destroy_all
      redirect_to new_session_path, notice: "Password has been reset."
    else
      redirect_to edit_password_path(params[:token]), alert: "Passwords did not match."
    end
  end

  private
    def set_user_by_token
      @user = User.find_by_password_reset_token!(params[:token])
    rescue ActiveSupport::MessageVerifier::InvalidSignature
      redirect_to new_password_path, alert: "Password reset link is invalid or has expired."
    end
end
```

**อธิบายทีละจุด:**

- `allow_unauthenticated_access` **ไม่มี `only:`** เลย — หมายความว่าทุก action ใน
  controller นี้เข้าได้โดยไม่ต้อง login (สมเหตุสมผล: คนที่ "ลืมรหัสผ่าน" คือคนที่ **ยัง
  ไม่ได้ login อยู่แล้ว** ถึงต้องมาใช้ฟีเจอร์นี้)
- `create` — **จงใจตอบข้อความเดียวกันเป๊ะ** ("Password reset instructions sent (if user
  with that email address exists).") **ไม่ว่าจะเจอ user จริงหรือไม่ก็ตาม** (สังเกตว่า
  `redirect_to` อยู่ **นอก** `if` block) นี่คือการป้องกัน **user enumeration** แบบเดียว
  กับที่อธิบายไว้ใน Step 404 — ถ้าตอบต่างกัน ("ไม่พบอีเมลนี้" vs "ส่งอีเมลแล้ว") ผู้ไม่
  หวังดีจะใช้ช่องทางนี้ไล่เช็คได้ว่าอีเมลไหนมีบัญชีอยู่ในระบบบ้าง
- `PasswordsMailer.reset(user).deliver_later` — ใช้ **`deliver_later`** ไม่ใช่
  `deliver_now` เพื่อส่งอีเมลผ่าน **background job** (ผ่าน Active Job) แทนที่จะบล็อก
  การทำงานของ request ปัจจุบันรอจนกว่าจะส่งอีเมลเสร็จ (การเชื่อมต่อ SMTP อาจใช้เวลาเป็น
  วินาที ทำให้ผู้ใช้ต้องรอหน้าเว็บค้างโดยไม่จำเป็น) เรื่อง Active Job และ background job
  แบบเต็มรูปแบบจะเรียนละเอียดใน **Part 061 (ActiveJob เบื้องต้น)** และ Action Mailer
  เต็มรูปแบบใน **Part 072** ตอนนี้จำแค่หลักการ: "งานที่ไม่จำเป็นต้องรอผลทันที เช่นการ
  ส่งอีเมล ควรส่งผ่าน `deliver_later` เสมอ ไม่ใช่ `deliver_now`"
- `set_user_by_token` ใช้ `find_by_password_reset_token!` (มี `!`) ที่ **raise
  exception** เมื่อ token ผิด/หมดอายุ แล้วจับด้วย `rescue` ในบรรทัดถัดมาทันที — รูปแบบ
  เดียวกับ `rescue_from` ที่เรียนใน Part 023 Step 229 เพียงแต่คราวนี้ `rescue` ผูกกับ
  private method เดียว ไม่ใช่ทั้ง controller
- `update` — เมื่อเปลี่ยนรหัสผ่านสำเร็จ เรียก **`@user.sessions.destroy_all`** ทันที —
  นี่คือจุดที่ใช้ประโยชน์จาก "session แบบมีฐานข้อมูลรองรับ" ที่อธิบายไว้ใน Step 406:
  **ล้าง session ของ user คนนี้ทุกอุปกรณ์พร้อมกัน** ทันทีที่รีเซ็ตรหัสผ่านสำเร็จ (เหตุผล
  ด้านความปลอดภัย: ถ้ามีคนอื่นแอบใช้บัญชีนี้อยู่ในอีกอุปกรณ์หนึ่งตอนที่เจ้าของบัญชีตัวจริง
  รีเซ็ตรหัสผ่าน คนนั้นจะถูกเตะออกจากระบบทันที)

### `app/mailers/passwords_mailer.rb`

```ruby
class PasswordsMailer < ApplicationMailer
  def reset(user)
    @user = user
    mail subject: "Reset your password", to: user.email_address
  end
end
```

```erb
<%# app/views/passwords_mailer/reset.html.erb %>
<p>
  You can reset your password on
  <%= link_to "this password reset page", edit_password_url(@user.password_reset_token) %>.

  This link will expire in <%= distance_of_time_in_words(0, @user.password_reset_token_expires_in) %>.
</p>
```

`edit_password_url(@user.password_reset_token)` เรียก `@user.password_reset_token` ที่
เราเพิ่งรู้จักว่ามาจาก `generates_token_for` อัตโนมัติ แล้วส่งเป็น `:token` param ให้กับ
route `resources :passwords, param: :token` (ที่ generator เพิ่มไว้ใน `routes.rb` ตอน
Step 405 — สังเกตว่าใช้ `param: :token` แทนที่จะเป็น `:id` แบบ resource ทั่วไป เพราะ
"id" ของการ reset รหัสผ่านคือ token ที่เซ็นชื่อไว้ ไม่ใช่ primary key ตัวเลขธรรมดา)

### ทดสอบ flow เต็มรูปแบบด้วย `curl`

```bash
# 1) ขอรีเซ็ตรหัสผ่าน
curl -c cookies.txt http://localhost:3000/passwords/new -o forgot.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' forgot.html | sed -E 's/.*value="([^"]*)"/\1/')

curl -i -c cookies.txt -b cookies.txt -X POST http://localhost:3000/passwords \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "email_address=somchai@example.com"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/session/new
```

เพราะไม่ได้ตั้งค่า SMTP จริง (development mode) อีเมลจะถูก **log ไว้ใน
`log/development.log`** แทนที่จะส่งจริง (ผ่าน Active Job adapter แบบ `:async` ที่เป็น
ค่า default ของ Rails เมื่อไม่ได้ตั้งค่าอื่น) — เนื้อหาอีเมลจริงที่ log ออกมา:

```
[ActiveJob] [ActionMailer::MailDeliveryJob] Delivered mail ...@vm.mail (39.4ms)
[ActiveJob] [ActionMailer::MailDeliveryJob] Date: ...
From: from@example.com
To: somchai@example.com
Subject: Reset your password
...
You can reset your password on
http://localhost:3000/passwords/eyJfcmFpbHMiOnsiZGF0YSI6WzEsInVDNjd4VU95aWUiXSwiZXhwIjoiMjAyNi0wOS0yNlQwNjo0MToyNC4xODNaIiwicHVyIjoiVXNlclxucGFzc3dvcmRfcmVzZXRcbjkwMCJ9fQ==--5bd5cad19b1b40cadbee56c5691dfeb344932e30/edit

This link will expire in 15 minutes.
```

("15 minutes" ในบรรทัดสุดท้ายยืนยัน `DEFAULT_RESET_TOKEN_EXPIRES_IN = 15.minutes` ที่
เห็นใน source code ตรงๆ — และถ้าถอดรหัส base64 ส่วนหนึ่งของ token จะเห็นข้อความ
`"purpose":"User\npassword_reset\n900"` ฝังอยู่ข้างใน ซึ่ง `900` วินาที ก็คือ 15 นาที
พอดี)

```bash
# 2) เปิดลิงก์ในอีเมล (คัดลอก path มาจาก log ด้านบน)
TOKEN_PATH="eyJfcmFpbHMiOnsiZGF0YSI6WzEsInVDNjd4VU95aWUiXSwiZXhwIjoiMjAyNi0wOS0yNlQwNjo0MToyNC4xODNaIiwicHVyIjoiVXNlclxucGFzc3dvcmRfcmVzZXRcbjkwMCJ9fQ==--5bd5cad19b1b40cadbee56c5691dfeb344932e30"
curl -c cookies2.txt "http://localhost:3000/passwords/$TOKEN_PATH/edit" -o reset_edit.html
CSRF=$(grep -o 'name="authenticity_token" value="[^"]*"' reset_edit.html | sed -E 's/.*value="([^"]*)"/\1/')

# 3) ตั้งรหัสผ่านใหม่
curl -i -c cookies2.txt -b cookies2.txt -X PUT "http://localhost:3000/passwords/$TOKEN_PATH" \
  --data-urlencode "authenticity_token=$CSRF" \
  --data-urlencode "password=NewSecret456" \
  --data-urlencode "password_confirmation=NewSecret456"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/session/new
```

```bash
# 4) ยืนยันว่ารหัสผ่านเก่าใช้ไม่ได้แล้ว
curl -c c3.txt http://localhost:3000/session/new -o l1.html
T=$(grep -o 'name="authenticity_token" value="[^"]*"' l1.html | sed -E 's/.*value="([^"]*)"/\1/')
curl -i -c c3.txt -b c3.txt -X POST http://localhost:3000/session \
  --data-urlencode "authenticity_token=$T" \
  --data-urlencode "email_address=somchai@example.com" \
  --data-urlencode "password=SuperSecret123"   # รหัสผ่านเดิม
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/session/new    # กลับไปหน้า login เหมือนเดิม = ล้มเหลว
```

```bash
# 5) ยืนยันว่ารหัสผ่านใหม่ใช้งานได้ และเข้าหน้า protected ได้จริง
curl -c c4.txt http://localhost:3000/session/new -o l2.html
T2=$(grep -o 'name="authenticity_token" value="[^"]*"' l2.html | sed -E 's/.*value="([^"]*)"/\1/')
curl -c c4.txt -b c4.txt -X POST http://localhost:3000/session \
  --data-urlencode "authenticity_token=$T2" \
  --data-urlencode "email_address=somchai@example.com" \
  --data-urlencode "password=NewSecret456"
curl -b c4.txt http://localhost:3000/
```

```
ยินดีต้อนรับ somchai@example.com! (หน้านี้เข้าถึงได้เฉพาะผู้ที่ login แล้ว)
```

ยืนยันครบวงจร: **ขอรีเซ็ต → รับลิงก์ทางอีเมล (log) → ตั้งรหัสผ่านใหม่ → รหัสผ่านเก่าใช้
ไม่ได้ → รหัสผ่านใหม่ใช้ได้และเข้าหน้า protected ได้จริง** และตรวจสอบฐานข้อมูลตรงๆ ก็
ยืนยันว่า `password_digest` เปลี่ยนเป็น hash ใหม่ (ขึ้นต้นด้วย `$2a$12$` เหมือนเดิม
แต่เป็นค่าที่ต่างไปจากเดิม) และ session record เก่าทั้งหมดถูกลบออกจากตาราง `sessions`
จริงตามที่ `update` action สั่งไว้

---

## Step 409: Remember-me, Persistent Session, และ Security Checklist ก่อนขึ้น production

### "Remember me" ไม่ใช่ตัวเลือก — มันเปิดอยู่แล้วโดย default

หลายคนคุ้นเคยกับ checkbox "จดจำฉันไว้" (remember me) ในฟอร์ม login ของเว็บทั่วไป ที่
ผู้ใช้ต้องเลือกเปิดเองถึงจะ login ค้างไว้ข้ามวันได้ — แต่เวอร์ชันที่ `bin/rails generate
authentication` สร้างให้นั้น **ไม่มี checkbox แบบนั้นเลย เพราะเปิด "remember me" ให้
ทุกคนโดย default อยู่แล้ว** ดูจากบรรทัดนี้ใน `start_new_session_for` (Step 406):

```ruby
cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
```

`.permanent` เป็น method ของ `ActionDispatch::Cookies` ที่ตั้งค่า **`expires: 20.years.
from_now`** ให้กับ cookie โดยอัตโนมัติ — ตรวจสอบได้จริงจาก cookie ที่ browser/curl
เก็บไว้หลัง login สำเร็จ (ถอดรหัสส่วนหนึ่งของ cookie เพื่อดู `exp`):

```
session_id cookie ...exp":"2046-09-26T06:26:43.475Z"...
```

Login วันนี้ (2026) แต่ cookie หมดอายุปี **2046** — นี่คือ "remember me" ที่เปิดอยู่แล้ว
โดยไม่ต้องทำอะไรเพิ่ม!

### ทำไมเปิด remember-me ตลอดถึงยังปลอดภัยอยู่ (เพราะ session อยู่ฝั่ง server)

ถ้าเป็นระบบแบบเวอร์ชันมือ (Step 402–404) ที่เก็บ `user_id` ตรงๆ ใน cookie การตั้ง
cookie ให้หมดอายุ 20 ปีแบบนี้จะ**อันตรายมาก** เพราะถ้า cookie หลุดไปอยู่ในมือคนอื่น
(ผ่านการขโมย, share คอมสาธารณะที่ไม่ล้าง cookie ฯลฯ) คนนั้นจะสวมสิทธิ์เป็นเจ้าของบัญชีได้
ยาวนานถึง 20 ปีโดยไม่มีทางเพิกถอน

แต่เพราะ cookie เก็บแค่ `session.id` ที่ชี้ไปยัง **record ในตาราง `sessions`** (ไม่ใช่
`user_id` ตรงๆ) การเพิกถอนสิทธิ์จึงทำได้ทุกเมื่อโดยไม่ต้องรอ cookie หมดอายุเอง:

- Logout ปกติ (`terminate_session`) → ลบ session record นั้นทิ้ง → cookie ที่เหลืออยู่
  ใน browser (ถ้ามี)ใช้งานไม่ได้อีกต่อไปทันที (`find_session_by_cookie` หาไม่เจอ →
  `resume_session` ได้ `nil` → `require_authentication` เด้งกลับไป login)
- Reset รหัสผ่าน (`@user.sessions.destroy_all`) → ลบทุก session record ของ user คนนั้น
  → ทุกอุปกรณ์ที่เคย login ไว้ถูกเตะออกพร้อมกัน
- ถ้าจะสร้างฟีเจอร์ **"ดูอุปกรณ์ที่ login อยู่ / logout อุปกรณ์อื่น"** ก็ทำได้ทันทีโดยไม่
  ต้องแก้ schema เลย เพราะข้อมูล `ip_address`/`user_agent` ต่อ session ถูกเก็บไว้แล้ว:

  ```ruby
  # ตัวอย่างแนวคิด (ยังไม่ต้องเขียนตอนนี้ แค่ให้เห็นภาพว่า schema รองรับอยู่แล้ว)
  current_user.sessions.each do |s|
    puts "#{s.ip_address} - #{s.user_agent} - login เมื่อ #{s.created_at}"
  end
  current_user.sessions.find(other_session_id).destroy  # เตะอุปกรณ์เดียวออก
  ```

นี่คือเหตุผลเชิงสถาปัตยกรรมที่แท้จริงว่าทำไม generator เลือกออกแบบให้ session อยู่ใน
ฐานข้อมูล (Step 406) แทนที่จะเก็บทุกอย่างใน cookie ตรงๆ แบบเวอร์ชันมือของเรา — **"remember
me ตลอดไป" ปลอดภัยได้ ถ้าฝั่ง server ควบคุมการเพิกถอนได้เสมอ ไม่ว่า cookie จะยังอายุ
เหลือกี่ปีก็ตาม**

### Security Checklist สำหรับระบบ Authentication

สรุปเป็นรายการตรวจสอบก่อนขึ้น production จริง (ผสมทั้งสิ่งที่เห็นจริงในโค้ดของ
generator และหลักการทั่วไป):

| หัวข้อ | สถานะในเวอร์ชัน generator | คำอธิบาย |
|---|---|---|
| **เก็บรหัสผ่านแบบ hash** | ✅ `has_secure_password` + bcrypt | ไม่มีทาง reverse กลับเป็น plaintext ได้ |
| **ป้องกัน user enumeration** | ✅ ข้อความ error เดียวกันทั้งตอน login และ password reset | ไม่บอกแยกว่า "อีเมลไม่มี" กับ "รหัสผ่านผิด" |
| **ป้องกัน timing attack** | ✅ `authenticate_by` รัน bcrypt ปลอมเมื่อไม่เจอ user | เวลาตอบสนองใกล้เคียงกันไม่ว่าอีเมลจะมีอยู่จริงหรือไม่ |
| **Rate limiting เฉพาะจุดอ่อนไหว** | ✅ `rate_limit` บน `SessionsController#create` และ `PasswordsController#create` | ป้องกัน brute-force เดารหัสผ่าน/ยิงขอ reset ถี่ๆ — เสริมด้วย `rack-attack` ระดับแอปทั้งหมดได้ใน Part 060 |
| **Secure cookie flags** | ✅ `httponly: true`, `same_site: :lax`, และเซ็นชื่อด้วย `cookies.signed` | ดูรายละเอียดด้านล่าง |
| **Session ที่เพิกถอนได้จริง** | ✅ session เก็บในฐานข้อมูล ไม่ใช่แค่ cookie | logout/reset รหัสผ่านมีผลทันที ไม่ต้องรอ cookie หมดอายุ |
| **Token หมดอายุ (password reset)** | ✅ `generates_token_for` มี `expires_in: 15.minutes` และผูกกับ `password_salt` | token เก่าใช้ไม่ได้ทั้งเมื่อหมดเวลาและเมื่อรหัสผ่านถูกเปลี่ยนไปแล้ว |
| **CSRF protection** | ✅ มาจาก Rails core (`protect_from_forgery`) ไม่เกี่ยวกับ auth โดยตรง | เรียนไปแล้วใน Part 023 |
| **Session fixation** | ✅ ป้องกันโดยอัตโนมัติ | ดูรายละเอียดด้านล่าง |
| **HTTPS ใน production** | ⚠️ ต้องตั้งค่าเอง | ดูรายละเอียดด้านล่าง |

**Secure cookie flags อธิบายเพิ่ม:**

- **`httponly: true`** — บอก browser ว่าห้ามให้ JavaScript ฝั่ง client อ่านค่า cookie
  นี้ได้เลย (`document.cookie` จะไม่เห็นค่านี้) ป้องกันไม่ให้ช่องโหว่ XSS (ที่จะเรียนใน
  Part 079) ถูกใช้ขโมย session cookie ไปได้ แม้ผู้โจมตีจะฉีด JavaScript เข้าหน้าเว็บได้
  สำเร็จก็ตาม
- **`same_site: :lax`** — บอก browser ว่าห้ามแนบ cookie นี้ไปกับ request ที่มาจาก
  เว็บไซต์อื่น (cross-site) ยกเว้นการ navigate ปกติ (คลิกลิงก์) ป้องกันการโจมตีแบบ
  **CSRF** อีกชั้นหนึ่ง (นอกเหนือจาก authenticity token ที่เรียนไปแล้วใน Part 023)
- **`cookies.signed`** (ไม่ใช่ `cookies` ธรรมดา) — เซ็นชื่อค่าด้วย `secret_key_base`
  ของแอป เหมือนกับที่ `session` ปกติทำ (Part 023 Step 225) ทำให้แก้ไขค่า `session_id`
  เองไม่ได้แม้จะเปิดดู cookie ได้ก็ตาม

**Session fixation:** คือการโจมตีที่ผู้ไม่หวังดี "ฝัง" session id ของตัวเองให้เหยื่อใช้
ไว้ล่วงหน้า (เช่น ส่งลิงก์ที่มี session id ปลอมให้เหยื่อคลิก) แล้วรอให้เหยื่อ login
ด้วย session id นั้น จากนั้นผู้โจมตีก็สวมสิทธิ์เป็นเหยื่อได้โดยใช้ session id เดียวกัน
วิธีป้องกันมาตรฐานคือ **ต้องสร้าง session id ใหม่ทุกครั้งหลัง login สำเร็จ** (ไม่ใช้
session id เดิมที่มีอยู่ก่อน login ต่อ) — สังเกตว่า `start_new_session_for` (Step 406)
เรียก `user.sessions.create!` **สร้าง record ใหม่ทุกครั้ง** ไม่เคยนำ session เดิม (ถ้ามี)
มาใช้ซ้ำเลย ทำให้ id ของ session เปลี่ยนไปทุกครั้งที่ login สำเร็จโดยอัตโนมัติ — ป้องกัน
session fixation ได้ในตัวโดยที่นักพัฒนาไม่ต้องคิดถึงเรื่องนี้เองด้วยซ้ำ (เทียบกับเวอร์ชัน
มือของเราที่ใช้ `session[:user_id] = user.id` ตรงๆ — Rails cookie-based session ของ
`ActionDispatch::Session::CookieStore` ก็มีกลไกป้องกันคล้ายกันในตัวอยู่แล้วเช่นกัน แต่
ควรรู้จักหลักการนี้ไว้เผื่อไปเจอ session store แบบอื่นที่ไม่มีการป้องกันนี้ในตัว)

**HTTPS ใน production:** ทุกอย่างข้างต้นจะไร้ความหมายทันทีถ้าแอปยังรับ traffic ผ่าน
HTTP ธรรมดา (ไม่เข้ารหัส) เพราะ cookie (ต่อให้ signed/httponly แค่ไหน) จะถูกดักอ่านได้
ง่ายๆ ระหว่างทางบนเครือข่าย (เช่น WiFi สาธารณะ) วิธีบังคับคือเพิ่มบรรทัดนี้ใน
`config/environments/production.rb`:

```ruby
config.force_ssl = true
```

`force_ssl = true` บังคับให้ทุก request ต้องผ่าน HTTPS (redirect HTTP → HTTPS อัตโนมัติ)
และเพิ่ม flag `secure: true` ให้ cookie ทุกตัวโดยอัตโนมัติด้วย (บอก browser ว่าส่ง
cookie นี้ผ่าน HTTPS เท่านั้น ห้ามส่งผ่าน HTTP) — เรื่องนี้จะกลับมาเจาะลึกอีกครั้งใน
**Part 079–081 (เฟส 13: Security)** และตอน deploy จริงใน **Part 076 (Kamal)**

---

## Step 410: แบบฝึกหัด — สร้างระบบ Auth เต็มวงจรด้วย Generator พร้อม Registration ของตัวเอง

### โจทย์

สร้างแอป Rails ใหม่ชื่อ `bookshelf_auth` แล้วทำตามขั้นตอนต่อไปนี้:

1. รัน `bin/rails generate authentication` เพื่อสร้างระบบ login/logout/password reset
2. เพิ่ม `RegistrationsController` ของตัวเอง (generator ไม่มีให้ ตามที่เรียนใน
   Step 405) ที่ใช้ `email_address`/`password`/`password_confirmation` (ให้ตรงกับชื่อ
   column ที่ generator สร้าง ไม่ใช่ `email` แบบ Step 403) และเรียก
   `start_new_session_for(@user)` (method จาก `Authentication` concern) แทนที่จะเซ็ต
   `session[:user_id]` ตรงๆ แบบเวอร์ชันมือ
3. สร้าง `HomeController#index` เป็นหน้าแรกที่ **ต้อง login ก่อนถึงเข้าได้** (ใช้กลไก
   default ของ `Authentication` concern คือไม่ต้องเขียน `before_action` เพิ่มเอง)
   แสดงข้อความทักทายด้วย `Current.user.email_address`
4. ทดสอบ flow เต็มวงจรด้วย `curl`: signup → เข้าหน้า protected → logout → login →
   ขอรีเซ็ตรหัสผ่าน → ตั้งรหัสผ่านใหม่ → login ด้วยรหัสผ่านใหม่ → เข้าหน้า protected
   อีกครั้ง

### เฉลย

```bash
rails new bookshelf_auth -d sqlite3
cd bookshelf_auth
bundle install
bin/rails generate authentication
bin/rails db:create db:migrate
```

**`app/controllers/registrations_controller.rb`** (ไฟล์ใหม่ที่ต้องสร้างเอง):

```ruby
class RegistrationsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      start_new_session_for @user
      redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ ยินดีต้อนรับ!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

สังเกตว่าไฟล์นี้แทบจะเหมือนกับ `RegistrationsController` ที่เขียนมือใน Step 403
เป๊ะๆ (เพราะหลักการ "รับฟอร์ม → validate ผ่าน `has_secure_password` → บันทึก" เป็นเรื่อง
เดียวกันเสมอ ไม่ว่าจะอยู่ในระบบมือหรือระบบที่ generator สร้าง) มีต่างแค่ 2 จุด:
`allow_unauthenticated_access` (เพราะตอนนี้ `before_action :require_authentication`
ทำงานกับทุก controller โดย default แล้ว จาก `Authentication` concern) และเรียก
`start_new_session_for @user` แทนที่จะเซ็ต `session[:user_id]` ตรงๆ (เพื่อให้ได้
record ในตาราง `sessions` ที่มี `ip_address`/`user_agent` ครบถ้วนแบบเดียวกับตอน login
ปกติ)

**`app/views/registrations/new.html.erb`:**

```erb
<h1>สมัครสมาชิก</h1>
<%= form_with model: @user, url: registrations_path do |form| %>
  <% if @user.errors.any? %>
    <ul style="color:red">
      <% @user.errors.full_messages.each do |msg| %><li><%= msg %></li><% end %>
    </ul>
  <% end %>
  <%= form.email_field :email_address, placeholder: "email" %><br>
  <%= form.password_field :password, placeholder: "password" %><br>
  <%= form.password_field :password_confirmation, placeholder: "confirm password" %><br>
  <%= form.submit "สมัครสมาชิก" %>
<% end %>
```

**`app/controllers/home_controller.rb`:**

```ruby
class HomeController < ApplicationController
  def index
    render plain: "ยินดีต้อนรับ #{Current.user.email_address}! (หน้านี้เข้าถึงได้เฉพาะผู้ที่ login แล้ว)"
  end
end
```

ไม่ต้องเขียน `before_action :require_authentication` เลย เพราะ `Authentication`
concern ผูกไว้ให้กับ **ทุก controller ที่สืบทอดจาก `ApplicationController` โดย default**
อยู่แล้ว (Step 406–407) — นี่คือ controller "ที่ต้อง login ก่อน" แบบง่ายที่สุดเท่าที่
จะเป็นไปได้ในระบบนี้: **แค่ไม่เรียก `allow_unauthenticated_access` ก็พอ**

**`config/routes.rb`:**

```ruby
Rails.application.routes.draw do
  resource :session
  resources :passwords, param: :token
  resources :registrations, only: [:new, :create]
  root "home#index"
end
```

ทดสอบด้วย `curl` ครบวงจร:

```bash
# 1) เข้าหน้าแรกโดยยังไม่ login → ต้องถูกเด้งไปหน้า login
curl -i http://localhost:3000/
# HTTP/1.1 302 Found
# location: http://localhost:3000/session/new

# 2) สมัครสมาชิก
curl -c cookies.txt http://localhost:3000/registrations/new -o signup.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' signup.html | sed -E 's/.*value="([^"]*)"/\1/')
curl -i -c cookies.txt -b cookies.txt -X POST http://localhost:3000/registrations \
  --data-urlencode "authenticity_token=$TOKEN" \
  --data-urlencode "user[email_address]=somchai@example.com" \
  --data-urlencode "user[password]=SuperSecret123" \
  --data-urlencode "user[password_confirmation]=SuperSecret123"
# HTTP/1.1 302 Found
# location: http://localhost:3000/

# 3) เข้าหน้าแรก (login อัตโนมัติจากตอนสมัคร)
curl -b cookies.txt http://localhost:3000/
# ยินดีต้อนรับ somchai@example.com! (หน้านี้เข้าถึงได้เฉพาะผู้ที่ login แล้ว)

# 4) logout (ดึง CSRF token ผ่าน meta tag แล้วส่งเป็น header ตามที่เรียนใน Step 404)
curl -c cookies.txt http://localhost:3000/session/new -o page.html
CSRF=$(grep -o '<meta name="csrf-token" content="[^"]*"' page.html | sed -E 's/.*content="([^"]*)"/\1/')
curl -i -c cookies.txt -b cookies.txt -X DELETE http://localhost:3000/session -H "X-CSRF-Token: $CSRF"
# HTTP/1.1 303 See Other
# location: http://localhost:3000/session/new

# 5) เข้าหน้าแรกอีกครั้งหลัง logout → ถูกเด้งกลับไป login
curl -i -b cookies.txt http://localhost:3000/
# HTTP/1.1 302 Found
# location: http://localhost:3000/session/new

# 6) login กลับเข้าไปใหม่
curl -c cookies2.txt http://localhost:3000/session/new -o l.html
T=$(grep -o 'name="authenticity_token" value="[^"]*"' l.html | sed -E 's/.*value="([^"]*)"/\1/')
curl -i -c cookies2.txt -b cookies2.txt -X POST http://localhost:3000/session \
  --data-urlencode "authenticity_token=$T" \
  --data-urlencode "email_address=somchai@example.com" \
  --data-urlencode "password=SuperSecret123"
# HTTP/1.1 302 Found
# location: http://localhost:3000/

# 7) ขอรีเซ็ตรหัสผ่าน
curl -c cookies3.txt http://localhost:3000/passwords/new -o forgot.html
T2=$(grep -o 'name="authenticity_token" value="[^"]*"' forgot.html | sed -E 's/.*value="([^"]*)"/\1/')
curl -i -c cookies3.txt -b cookies3.txt -X POST http://localhost:3000/passwords \
  --data-urlencode "authenticity_token=$T2" \
  --data-urlencode "email_address=somchai@example.com"
# HTTP/1.1 302 Found — ดึง reset link จริงจาก log/development.log ตามวิธีใน Step 408

# 8) ตั้งรหัสผ่านใหม่ด้วย token จาก log แล้ว login ด้วยรหัสผ่านใหม่ — ทำตามขั้นตอนใน Step 408
```

ผลลัพธ์ทุกขั้นตอนตรงตามที่คาดไว้ทั้งหมด (verify ด้วยการ query ฐานข้อมูล: มี user 1
record, `password_digest` เป็น bcrypt hash ที่ขึ้นต้นด้วย `$2a$12$`, ตาราง `sessions`
มี record ที่ถูกสร้างและลบตามจังหวะ login/logout/reset ถูกต้อง)

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม validation ใน `User` model ให้ตรวจสอบว่า `email_address` เป็นรูปแบบอีเมลที่
   ถูกต้องจริง (ใบ้: `validates :email_address, format: { with: URI::MailTo::EMAIL_REGEXP
   }`) แล้วทดสอบว่าสมัครด้วย `"not-an-email"` ถูก reject จริงหรือไม่
2. เพิ่มหน้า "อุปกรณ์ที่ login อยู่" (`SessionsController#index` หรือสร้าง
   controller ใหม่) ที่แสดงรายการ `current_user.sessions` ทั้งหมดพร้อม
   `ip_address`/`user_agent`/`created_at` และมีปุ่ม "ออกจากระบบอุปกรณ์นี้" ที่ลบ session
   record นั้นเพียงตัวเดียว (ไม่ใช่ `destroy_all`) — ต้องระวังเรื่อง authorization ง่ายๆ
   ด้วยว่าห้ามให้ user คนหนึ่งลบ session ของอีกคนได้ (ใบ้:
   `current_user.sessions.find(params[:id])` ไม่ใช่ `Session.find(params[:id])` ตรงๆ
   — ถ้าใช้ `Session.find` ตรงๆ จะเป็นช่องโหว่ authorization ที่เรียกว่า **Insecure
   Direct Object Reference (IDOR)** ซึ่งจะเรียนลึกกว่านี้ในเฟส 13)
3. ลองปิด `rate_limit` ใน `SessionsController` ชั่วคราว (comment บรรทัดนั้นออก) แล้วเขียน
   สคริปต์ (bash loop หรือ Ruby script เรียก `Net::HTTP`) ยิง login ผิดรหัสผ่านซ้ำๆ 100
   ครั้งติดกัน วัดเวลาที่ใช้ทั้งหมด แล้วเปิด `rate_limit` กลับมาทำซ้ำอีกครั้ง เปรียบเทียบ
   ว่าพฤติกรรมต่างกันอย่างไร (ทั้งเวลาที่ใช้และ response ที่ได้)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ความต่างระหว่าง **Authentication** ("คุณเป็นใคร") กับ **Authorization** ("คุณทำอะไร
  ได้บ้าง") — Part นี้ตอบแค่คำถามแรก
- `has_secure_password` + gem `bcrypt` เพิ่ม virtual attribute `password`/
  `password_confirmation`, validation อัตโนมัติ, และ method `authenticate` ให้ Model
  ที่มี column `password_digest` — และทำไมห้ามเก็บรหัสผ่านแบบ plaintext เด็ดขาด
- สร้างระบบ signup/login/logout ด้วยมือทั้งหมดได้: `RegistrationsController`,
  `SessionsController`, `session[:user_id]`, `current_user` (memoized + `helper_method`),
  และ `require_authentication` ที่ใช้กลไก `before_action` halt เดียวกับที่เรียนใน
  Part 023
- Rails 8 มี generator ในตัวชื่อ `bin/rails generate authentication` ที่สร้างระบบ
  login/logout/password reset แบบ production-grade ให้ในคำสั่งเดียว (แต่ไม่รวม signup
  — ต้องสร้าง `RegistrationsController` เอง)
- อ่านและเข้าใจโค้ดทุกไฟล์ที่ generator สร้าง: `User`/`Session`/`Current` model,
  `Authentication` concern (`resume_session`, `require_authentication`,
  `start_new_session_for`, `terminate_session`, `allow_unauthenticated_access`),
  `SessionsController`, `PasswordsController`
- `User.authenticate_by` ป้องกัน **timing attack** และ **user enumeration** ที่เวอร์ชัน
  `find_by` + `&.authenticate` ธรรมดายังทำไม่ได้
- `rate_limit` เป็นฟีเจอร์ใหม่ของ `ActionController` (Rails 8) สำหรับจำกัดจำนวนครั้งที่
  เรียก action ได้ในแต่ละช่วงเวลา ต่างจาก `rack-attack` (Part 060) ตรงที่ทำงานที่ระดับ
  action เดียว ไม่ใช่ระดับทั้งแอป
- `has_secure_password` ผูก `generates_token_for :password_reset` ให้อัตโนมัติ (หมดอายุ
  15 นาที ผูกกับ salt ของรหัสผ่าน) ทำให้ได้ `password_reset_token`/
  `find_by_password_reset_token!` มาใช้ทำ password reset flow แบบสมบูรณ์
- Session ที่เก็บเป็น record ในฐานข้อมูล (ไม่ใช่แค่ค่าดิบใน cookie) ทำให้ "remember me"
  เปิดได้ตลอด (`cookies.signed.permanent`) อย่างปลอดภัย เพราะเพิกถอนสิทธิ์ได้ทุกเมื่อจาก
  ฝั่ง server โดยไม่ต้องรอ cookie หมดอายุ
- Security checklist ของระบบ authentication: hash รหัสผ่าน, ป้องกัน user enumeration,
  ป้องกัน timing attack, rate limiting, secure cookie flags (`httponly`, `same_site`,
  signed), ป้องกัน session fixation, และ `force_ssl` สำหรับ production

**ต่อไป (Part 042):** เราจะเรียนรู้จัก **Devise** — gem authentication ที่ได้รับความ
นิยมสูงสุดในระบบนิเวศ Rails มายาวนาน ทั้งการติดตั้ง, การ customize controller/view ให้
ตรงกับดีไซน์ของแอป, และโมดูลเสริมที่ Devise มีให้แต่ `bin/rails generate authentication`
ยังไม่มี เช่น **`:confirmable`** (บังคับยืนยันอีเมลก่อนใช้งานได้) และ **`:lockable`**
(ล็อกบัญชีชั่วคราวหลัง login ผิดติดต่อกันหลายครั้ง) — พร้อมเปรียบเทียบว่าเมื่อไหร่ควร
เลือกใช้ Devise แทนระบบที่เขียนเอง/generator ของ Rails ที่เพิ่งเรียนไปใน Part นี้
