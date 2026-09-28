# Part 080: Mass Assignment, Secure HTTP Headers และ Brakeman Scan เชิงลึก — A05 Security Misconfiguration

> **Step ครอบคลุมใน Part นี้:** Step 791–800
> **ระดับ:** กลาง–สูง (ต้องผ่าน Part 034 เรื่อง Strong Parameters, Part 075 เรื่อง Brakeman เบื้องต้น
> และ Part 079 เรื่อง OWASP Top 10 มาก่อน — Part นี้จะ **อ้างอิงแนวคิดเหล่านั้น** โดยไม่สอนซ้ำตั้งแต่ต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x

ใน Part ที่แล้ว (079) เราเจาะลึกช่องโหว่ประเภท **A03 Injection** — SQL Injection, XSS, และ CSRF โดยพิสูจน์
ให้เห็นว่า "ข้อมูลที่ผู้ใช้ควบคุมได้" สามารถปนเข้าไปกับ "โครงสร้างคำสั่ง/โค้ด/ความน่าเชื่อถือของ request"
ได้อย่างไร Part นี้จะเปลี่ยนมุมมองมาที่ **A05 Security Misconfiguration** — ช่องโหว่ที่ไม่ได้เกิดจาก
"การเขียนโค้ดผิด" โดยตรง แต่เกิดจาก **การตั้งค่าที่ไม่รัดกุม** ซึ่งแยกเป็น 3 หัวข้อหลักที่เชื่อมกัน:

1. **Mass Assignment** — ปล่อยให้ `params` กำหนดว่าฟิลด์ไหนถูกอัปเดตได้บ้างโดยไม่ผ่าน allowlist ที่
   รัดกุม
2. **Secure HTTP Headers** — Headers ที่ browser ต้องการเพื่อบังคับพฤติกรรมที่ปลอดภัย แต่ Rails ยังตั้ง
   ให้เป็น default บางส่วนเท่านั้น
3. **Brakeman Scan เชิงลึก** — อ่านและจัดการผลสแกน Brakeman ในแบบที่ Part 075 ไม่ได้ครอบคลุม
   รวมถึง check ที่เกี่ยวกับ mass assignment, header, และ configuration โดยตรง

ทั้งสามเรื่องนี้มีจุดร่วมคือ: **เป็นสิ่งที่แอปทำงาน "ได้ปกติ" ถ้าไม่ได้ตั้งใจมองหา** — ไม่มี error ขึ้น
ไม่มีอะไรพัง แต่ช่องโหว่ซ่อนอยู่ในระดับ configuration ที่นักพัฒนามักข้ามไปโดยไม่รู้ตัว

## สารบัญของ Part นี้

- Step 791: Mass Assignment คืออะไร — กลไกเบื้องหลังและทำไม Strong Parameters ถึงถูกสร้างมา
- Step 792: ช่องโหว่ Mass Assignment ในทางปฏิบัติ — พิสูจน์ด้วยของจริง (escalate role และ override admin flag)
- Step 793: Strong Parameters ที่ถูกต้องจริง — pattern ที่ปลอดภัยและ pattern ที่ดูปลอดภัยแต่ยังเสี่ยง
- Step 794: Secure HTTP Headers — Rails ตั้งอะไรให้อยู่แล้วบ้าง และอะไรที่ต้องตั้งเพิ่มเอง
- Step 795: HSTS (Strict-Transport-Security) — บังคับ HTTPS ในระดับ browser ป้องกัน downgrade attack
- Step 796: Clickjacking กับ `X-Frame-Options` / `frame-ancestors` — ป้องกันการฝังแอปใน iframe
- Step 797: Brakeman เชิงลึก — check ประเภทที่เกิน SQL Injection: `MassAssignment`, `UnsafeReflection`, `HeaderDoS`, `JRuby`
- Step 798: อ่านและจัดการ Brakeman output อย่างมีระบบ — confidence, ignore file, false positive vs. real finding
- Step 799: Brakeman ใน CI Pipeline (ทบทวนจาก Part 075) + automation ที่ถูกต้อง
- Step 800: แบบฝึกหัดปิด Part — Mass Assignment พิสูจน์จริง + Headers ตรวจด้วย curl + Brakeman scan

---

## Step 791: Mass Assignment คืออะไร — กลไกเบื้องหลังและทำไม Strong Parameters ถึงถูกสร้างมา

### กลไกพื้นฐาน: `update` และ `params` ใน Rails

เวลาผู้ใช้ส่ง form มาที่ controller Rails จะรับข้อมูลผ่าน `params` object ซึ่งเป็น hash-like structure ที่
เก็บทุกค่าที่ส่งมาใน request ปัญหาเริ่มต้นตรงนี้: **`params` รับค่าทุกอย่างที่ client ส่งมาโดยไม่มีการกรอง**
ไม่ว่าจะมาจาก form ที่สร้างไว้ใน view หรือมาจาก HTTP request ที่ทำตรงๆ ด้วย `curl` หรือ `Postman`

```ruby
# ตัวอย่าง: สมมติว่า controller ทำแบบนี้ (อย่าทำในโค้ดจริง)
def update
  @user = User.find(params[:id])
  @user.update(params[:user])  # อันตราย! ส่ง params ทั้งก้อนเข้า update
  redirect_to @user
end
```

ถ้า form ของเราส่งแค่ `user[name]` กับ `user[email]` มา ทุกอย่างดูปกติ — แอปทำงานได้ตามที่ตั้งใจ แต่ถ้า
ผู้โจมตีแก้ request แล้วเพิ่ม `user[admin]=true` หรือ `user[role]=admin` เข้าไปด้วยล่ะ?

ถ้า `User` model มีฟิลด์ `admin` หรือ `role` อยู่จริง — Rails จะ **update ฟิลด์นั้นตามที่ส่งมา** โดยไม่มี
ข้อผิดพลาดใดๆ ปรากฏให้เห็น นี่คือ **Mass Assignment vulnerability**

### ประวัติ: ทำไม Strong Parameters ถูกสร้างมา

ก่อน Rails 4 (ก่อนปี 2013) Rails มีระบบที่เรียกว่า `attr_accessible` และ `attr_protected` ในระดับ Model:

```ruby
# วิธีเก่า (Rails 3 และก่อนหน้า) — อย่าใช้แล้ว
class User < ActiveRecord::Base
  attr_accessible :name, :email          # อนุญาตให้ mass assign ได้เฉพาะสองฟิลด์นี้
  # หรือ
  attr_protected :admin, :role           # ห้าม mass assign สองฟิลด์นี้ (blacklist — อันตรายกว่า)
end
```

ระบบนี้มีปัญหาสองอย่าง:
1. **`attr_protected` เป็น blacklist** — ถ้าเพิ่มฟิลด์ใหม่แล้วลืมเพิ่มใน protected รายการ ฟิลด์นั้นก็เสี่ยง
2. **logic อยู่ใน Model ทุก context เหมือนกัน** — บางครั้ง admin สามารถ set `role` ได้ แต่ user ธรรมดา
   ไม่ได้ ระบบเก่าจัดการเรื่องนี้ลำบากมาก

**เหตุการณ์ที่เปลี่ยนทุกอย่าง:** ปี 2012 นักพัฒนาชื่อ Egor Homakov ค้นพบและสาธิตช่องโหว่ Mass
Assignment ใน GitHub.com จริงๆ โดยการ push commit ที่มี SSH public key ที่เขาสุ่มขึ้นมาเอง เข้าไปใน
repository ของ Rails เอง (ที่เขาไม่มีสิทธิ์) ผ่านการ exploit ช่องโหว่นี้ในระบบ authorization ของ GitHub
เหตุการณ์นี้ทำให้ Ruby on Rails team ตัดสินใจ **ย้าย whitelist มาอยู่ใน Controller** ในรูปของ **Strong
Parameters** (เปิดตัวใน Rails 4.0) และในที่สุด **เลิกใช้และลบ `attr_accessible` / `attr_protected`
ออกจาก Rails core** ไปเลย

### Strong Parameters: แนวคิดพื้นฐาน

```ruby
# วิธีปัจจุบัน (Rails 4+) — ถูกต้อง
def update
  @user = User.find(params[:id])
  @user.update(user_params)  # ส่งเฉพาะ params ที่ผ่าน permit แล้ว
  redirect_to @user
end

private

def user_params
  params.require(:user).permit(:name, :email)
  # ฟิลด์อื่นที่ไม่ได้ permit จะถูกกรองออกอัตโนมัติ แม้จะส่งมาใน request ก็ตาม
end
```

ความต่างที่สำคัญจากระบบเก่า: **whitelist อยู่ใน Controller** ไม่ใช่ใน Model ทำให้:
- admin controller อาจ `permit` ฟิลด์ได้มากกว่า user controller
- แต่ละ action สามารถมี `permit` ของตัวเองได้
- model ไม่รู้เรื่องเลยว่าผู้ใช้มี permission แค่ไหน — model ทำหน้าที่ data layer อย่างเดียว

---

## Step 792: ช่องโหว่ Mass Assignment ในทางปฏิบัติ — พิสูจน์ด้วยของจริง

### สร้าง scenario ที่มีช่องโหว่จริง

ลองสร้างแอปทดลองขนาดเล็กที่มีช่องโหว่ mass assignment แบบที่พบในโค้ดจริง:

```ruby
# db/migrate/20240101000001_create_users.rb
class CreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.boolean :admin, default: false, null: false
      t.string :role, default: "member", null: false
      t.integer :credits, default: 0, null: false
      t.timestamps
    end
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  validates :name, presence: true
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
end
```

```ruby
# app/controllers/users_controller.rb — มีช่องโหว่!
class UsersController < ApplicationController
  def update
    @user = User.find(params[:id])

    # ช่องโหว่ที่ 1: ส่ง params ทั้งก้อนโดยไม่ permit
    if @user.update(params[:user])
      redirect_to @user, notice: "Updated!"
    else
      render :edit
    end
  end
end
```

### พิสูจน์การโจมตีด้วย curl

สมมติว่า user ธรรมดามี id = 5 ต้องการแก้แค่ชื่อตัวเอง แต่ผู้โจมตีส่ง request นี้ไป:

```bash
# request ที่ user ธรรมดาควรส่ง:
curl -X PATCH "https://example.com/users/5" \
  -d "user[name]=New+Name" \
  -H "X-CSRF-Token: ..." \
  -b "session=..."

# request ที่ผู้โจมตีส่งจริง (เพิ่มฟิลด์อื่นเข้าไป):
curl -X PATCH "https://example.com/users/5" \
  -d "user[name]=New+Name&user[admin]=true&user[role]=admin&user[credits]=99999" \
  -H "X-CSRF-Token: ..." \
  -b "session=..."
```

ผลที่เกิดขึ้นถ้า controller มีช่องโหว่: Rails จะ update ฟิลด์ `admin`, `role`, และ `credits` ตามที่ส่งมา
โดย **ไม่มี error ใดๆ** — User ธรรมดากลายเป็น admin และมี credits 99,999 ทันที

### ตรวจยืนยันผ่าน Rails console

```ruby
# ก่อนโจมตี:
User.find(5)
# => #<User id: 5, name: "Alice", email: "alice@example.com", admin: false, role: "member", credits: 0>

# หลังจาก request ที่เป็นอันตราย:
User.find(5)
# => #<User id: 5, name: "New Name", email: "alice@example.com", admin: true, role: "admin", credits: 99999>
```

### ช่องโหว่ที่พบบ่อยในโค้ดจริง — ที่ดูปลอดภัยแต่ยังเสี่ยง

**Pattern 1: `permit!` — อนุญาตทุกอย่าง (อันตรายมาก)**

```ruby
# อย่าทำ! permit! = ปิด Strong Parameters ทั้งหมด
def user_params
  params.require(:user).permit!  # เหมือนกับไม่มี Strong Parameters เลย
end
```

**Pattern 2: permit nested hash โดยไม่ระบุ keys**

```ruby
# อันตราย: อนุญาตให้ส่ง hash ใดๆ ก็ได้ภายใต้ :settings
def user_params
  params.require(:user).permit(:name, :email, settings: {})
  # settings: {} หมายความว่า "settings ใดก็ได้" — เหมือน permit! สำหรับ nested hash นั้น
end

# ที่ถูกต้อง: ระบุ key ใน nested hash ให้ชัดเจน
def user_params
  params.require(:user).permit(:name, :email, settings: [:theme, :notifications])
end
```

**Pattern 3: ใช้ `slice` แต่ลืมว่า `slice` ไม่แปลง type**

```ruby
# ดูปลอดภัยแต่ยังมีปัญหา:
def update
  allowed = params[:user].slice(:name, :email)  # ดูเหมือนกรองแล้ว
  @user.update(allowed)  # แต่ถ้า params[:user] ไม่ใช่ ActionController::Parameters?
end
# ที่ถูกต้องคือใช้ Strong Parameters ผ่าน params.require().permit() เสมอ
```

---

## Step 793: Strong Parameters ที่ถูกต้องจริง — Pattern ที่ปลอดภัยและ Pattern ที่ดูปลอดภัยแต่ยังเสี่ยง

### Pattern พื้นฐาน: require + permit

```ruby
private

def user_params
  params.require(:user).permit(:name, :email)
end
```

- `require(:user)` — บังคับว่าต้องมี key `:user` ใน params (ถ้าไม่มีจะ raise `ActionController::ParameterMissing`)
- `permit(:name, :email)` — อนุญาตเฉพาะสองฟิลด์นี้ ฟิลด์อื่นทั้งหมดจะถูกกรองออกและ Rails จะ log
  คำเตือน `Unpermitted parameter: :admin` ใน development log

### Nested attributes

```ruby
# อนุญาต nested model (เช่น has_many :addresses)
def user_params
  params.require(:user).permit(
    :name,
    :email,
    addresses_attributes: [:id, :street, :city, :country, :_destroy]
    # :id และ :_destroy จำเป็นสำหรับ accepts_nested_attributes_for
  )
end
```

### Array of scalars

```ruby
# อนุญาต array ของค่าธรรมดา (เช่น checkbox หลายช่อง)
def user_params
  params.require(:user).permit(:name, role_ids: [])
  # role_ids: [] บอกว่า "role_ids เป็น array ของค่า scalar ได้"
end
```

### Context-dependent permit: admin vs. regular user

บางครั้ง admin ควรสามารถตั้งค่า `role` ได้ แต่ user ธรรมดาไม่ได้ วิธีจัดการ:

```ruby
def user_params
  permitted = [:name, :email]
  permitted += [:role, :admin, :credits] if current_user.admin?
  params.require(:user).permit(*permitted)
end
```

หรือสร้าง private method แยก:

```ruby
def user_params
  if current_user.admin?
    admin_user_params
  else
    regular_user_params
  end
end

def admin_user_params
  params.require(:user).permit(:name, :email, :role, :admin, :credits)
end

def regular_user_params
  params.require(:user).permit(:name, :email)
end
```

### เขียน test ยืนยัน Strong Parameters

```ruby
# spec/requests/users_spec.rb
RSpec.describe "Users", type: :request do
  let(:user) { create(:user, admin: false, role: "member", credits: 0) }

  describe "PATCH /users/:id" do
    context "เมื่อ user ธรรมดา" do
      before { sign_in user }

      it "ไม่อนุญาตให้เปลี่ยน admin flag" do
        patch user_path(user), params: { user: { name: "New Name", admin: true } }
        expect(user.reload.admin).to be false
      end

      it "ไม่อนุญาตให้เปลี่ยน role" do
        patch user_path(user), params: { user: { name: "New Name", role: "admin" } }
        expect(user.reload.role).to eq "member"
      end

      it "ไม่อนุญาตให้เพิ่ม credits เอง" do
        patch user_path(user), params: { user: { name: "New Name", credits: 99999 } }
        expect(user.reload.credits).to eq 0
      end

      it "อนุญาตให้เปลี่ยน name และ email ได้ปกติ" do
        patch user_path(user), params: { user: { name: "New Name", email: "new@example.com" } }
        expect(user.reload.name).to eq "New Name"
        expect(user.reload.email).to eq "new@example.com"
      end
    end
  end
end
```

---

## Step 794: Secure HTTP Headers — Rails ตั้งอะไรให้อยู่แล้วบ้าง และอะไรที่ต้องตั้งเพิ่มเอง

### ทำไม HTTP Headers สำคัญด้านความปลอดภัย

เมื่อ server ส่ง HTTP response กลับมา นอกจาก body (HTML, JSON) แล้ว ยังมี **headers** ที่บอก browser
ว่าควรจัดการ response นั้นอย่างไร Browser สมัยใหม่ใช้ headers เหล่านี้ในการ:

- บังคับว่า page นี้โหลดได้เฉพาะผ่าน HTTPS เท่านั้น (HSTS)
- ป้องกันไม่ให้ page ถูกฝังใน iframe ของเว็บอื่น (ป้องกัน clickjacking)
- บังคับว่า browser ไม่ควรเดา content type เอง (ป้องกัน MIME-type sniffing)
- กำหนดว่า script จากแหล่งไหนได้รับอนุญาตให้รันใน page นี้ (CSP จาก Step 786)

### headers ที่ Rails ตั้งให้เป็น default อยู่แล้ว

ตั้งแต่ Rails 5+ เป็นต้นมา Rails เพิ่ม security headers หลายตัวให้อัตโนมัติผ่าน middleware
`ActionDispatch::DefaultHeaders` ตรวจสอบได้ด้วย:

```bash
# ดู default headers ที่ Rails ตั้งให้:
rails middleware | grep Header
# ActionDispatch::DefaultHeaders

# หรือ query ด้วย curl ในแอปที่รันอยู่:
curl -I http://localhost:3000/
```

**Default headers ที่ Rails 8 ตั้งให้:**

```
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 0
X-Content-Type-Options: nosniff
X-Permitted-Cross-Domain-Policies: none
Referrer-Policy: strict-origin-when-cross-origin
```

**คำอธิบายแต่ละ header:**

| Header | ค่า default | ความหมาย |
|--------|------------|----------|
| `X-Frame-Options` | `SAMEORIGIN` | อนุญาตให้ฝังใน iframe เฉพาะจาก origin เดียวกัน (ป้องกัน clickjacking จากเว็บภายนอก) |
| `X-XSS-Protection` | `0` | ปิด built-in XSS filter ของ browser (ตั้งใจ — filter เก่าๆ มีปัญหาและ deprecated แล้ว) |
| `X-Content-Type-Options` | `nosniff` | บังคับให้ browser ใช้ `Content-Type` ที่ server ระบุ ไม่ใช่เดาเอง |
| `X-Permitted-Cross-Domain-Policies` | `none` | บล็อก Flash/PDF plugins ไม่ให้เข้าถึง cross-domain data |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | ส่ง full URL ใน `Referer` เฉพาะ same-origin; ส่งแค่ origin เมื่อข้าม origin |

### Headers ที่ต้องตั้งเพิ่มเอง

**1. `Strict-Transport-Security` (HSTS)** — Rails ไม่ตั้งให้อัตโนมัติ ต้องเปิด `force_ssl`:

```ruby
# config/environments/production.rb
config.force_ssl = true
# Rails จะเพิ่ม header: Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**2. `Content-Security-Policy` (CSP)** — ครอบคลุมแล้วใน Step 786 (Part 079)

**3. `Permissions-Policy`** — ควบคุมการเข้าถึง browser features (camera, microphone, geolocation):

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.permissions_policy do |policy|
  policy.camera      :none
  policy.gyroscope   :none
  policy.microphone  :none
  policy.usb         :none
  policy.geolocation :self
end
```

### ตรวจสอบ headers ด้วย curl

```bash
# ตรวจสอบ headers ของแอปที่รันอยู่:
curl -sI http://localhost:3000/ | grep -E "(X-Frame|X-Content|Strict|Content-Security|Referrer|Permissions)"

# ผลที่ควรเห็นใน production:
# X-Frame-Options: SAMEORIGIN
# X-Content-Type-Options: nosniff
# Strict-Transport-Security: max-age=31536000; includeSubDomains
# Content-Security-Policy: default-src 'self'; ...
# Referrer-Policy: strict-origin-when-cross-origin
# Permissions-Policy: camera=(), gyroscope=(), microphone=(), usb=(), geolocation=(self)
```

### เขียน request spec ยืนยัน headers

```ruby
# spec/requests/security_headers_spec.rb
RSpec.describe "Security Headers", type: :request do
  describe "GET /" do
    before { get root_path }

    it "มี X-Frame-Options header" do
      expect(response.headers["X-Frame-Options"]).to eq "SAMEORIGIN"
    end

    it "มี X-Content-Type-Options header" do
      expect(response.headers["X-Content-Type-Options"]).to eq "nosniff"
    end

    it "มี Referrer-Policy header" do
      expect(response.headers["Referrer-Policy"]).to include "strict-origin"
    end
  end
end
```

---

## Step 795: HSTS (Strict-Transport-Security) — บังคับ HTTPS ในระดับ Browser ป้องกัน Downgrade Attack

### HTTPS ทั่วไปยังมีช่องโหว่อะไร?

แม้แอปจะรองรับ HTTPS แต่ถ้า user พิมพ์ `http://example.com` (ไม่มี s) browser จะส่ง request แบบ HTTP
ก่อน แล้วค่อย redirect ไป HTTPS — ช่วงเวลาที่ request เดินทางแบบ HTTP นั้นคือจุดอ่อน:

**SSL Stripping Attack** (downgrade attack):
1. ผู้โจมตีอยู่ใน man-in-the-middle position (เช่น Wi-Fi สาธารณะ)
2. User ส่ง `http://example.com` (HTTP)
3. ผู้โจมตี intercept request นี้ไว้ก่อน ทำตัวเป็น proxy — คุยกับ server ด้วย HTTPS แทน user
4. ส่ง HTTP response กลับให้ user (ไม่ใช่ HTTPS redirect)
5. User "เห็น" ว่าใช้งานแอปได้ปกติ แต่จริงๆ ผู้โจมตีเห็น traffic ทั้งหมด

**HSTS แก้ปัญหานี้ได้อย่างไร:**

เมื่อ browser เคยเห็น `Strict-Transport-Security` header จาก server แล้ว browser จะ **จำ** ว่า domain
นี้ต้องใช้ HTTPS เสมอ และในครั้งต่อไปแม้ user จะพิมพ์ `http://` browser จะ **upgrade เป็น HTTPS
ก่อนส่ง request ออกไป** โดยไม่ผ่านเครือข่ายเลย — ผู้โจมตีไม่มีโอกาส intercept HTTP request

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**ความหมายของแต่ละส่วน:**

| ส่วน | ความหมาย |
|------|----------|
| `max-age=31536000` | browser จำว่าต้องใช้ HTTPS นาน 1 ปี (31,536,000 วินาที) นับจากครั้งที่เห็น header ล่าสุด |
| `includeSubDomains` | ใช้กับ subdomain ทั้งหมดด้วย (api.example.com, admin.example.com ฯลฯ) |
| `preload` | ขอเข้า HSTS Preload List ของ browser — browser รู้จะใช้ HTTPS แม้ก่อนที่จะเคยเปิดเว็บนั้นเลย |

### ตั้งค่าใน Rails

```ruby
# config/environments/production.rb
config.force_ssl = true
# ค่า default คือ max-age=31536000; includeSubDomains

# ถ้าต้องการปรับแต่งค่า:
config.ssl_options = {
  hsts: {
    expires: 1.year,
    subdomains: true,
    preload: true
  }
}
```

### ข้อควรระวัง: อย่าเปิด HSTS โดยยังไม่พร้อม

HSTS เมื่อเปิดแล้ว **ยากมากที่จะถอนออก** เพราะ browser ที่เคย "จำ" ไปแล้วจะยังบังคับ HTTPS ต่อไปจนกว่า
max-age จะหมด (1 ปีถ้าตั้งตาม default) ถ้ายังไม่แน่ใจว่าทุก subdomain พร้อมรองรับ HTTPS 100% ให้
เริ่มด้วย `max-age` สั้นๆ ก่อน:

```ruby
# ระหว่าง testing (max-age สั้น = ถอนออกได้เร็ว):
config.ssl_options = { hsts: { expires: 5.minutes } }

# เมื่อมั่นใจแล้ว:
config.ssl_options = { hsts: { expires: 1.year, subdomains: true } }
```

---

## Step 796: Clickjacking กับ `X-Frame-Options` / `frame-ancestors` — ป้องกันการฝังแอปใน iframe

### Clickjacking คืออะไร

Clickjacking (หรือ UI Redressing) คือการโจมตีที่ผู้โจมตีสร้างเว็บหน้าตาหลอกๆ แล้ว **ซ่อนแอปของเราไว้
ใน invisible iframe** วางทับอยู่ด้านบน เมื่อ user คิดว่าตัวเองคลิกปุ่มบนเว็บหลอก จริงๆ แล้วกำลังคลิกปุ่ม
บนแอปของเรา (เช่น ปุ่ม "โอนเงิน" หรือ "ยืนยันการซื้อ") โดยไม่รู้ตัว

```html
<!-- เว็บของผู้โจมตี -->
<style>
  iframe {
    position: absolute;
    top: 0;
    left: 0;
    opacity: 0;  /* มองไม่เห็น แต่รับ click ได้ */
    z-index: 10;
    width: 100%;
    height: 100%;
  }
  .fake-button {
    position: absolute;
    top: 200px;
    left: 300px;
  }
</style>

<div class="fake-button">คลิกเพื่อรับของรางวัล!</div>
<iframe src="https://your-bank.com/transfer?to=attacker&amount=10000"></iframe>
```

### `X-Frame-Options` — การป้องกันแบบดั้งเดิม

```
X-Frame-Options: DENY          # ห้ามฝังใน iframe ทุกที่
X-Frame-Options: SAMEORIGIN    # อนุญาตเฉพาะจาก origin เดียวกัน (Rails default)
X-Frame-Options: ALLOW-FROM https://trusted.com  # deprecated — ไม่ควรใช้
```

Rails ตั้ง `SAMEORIGIN` ให้ by default ซึ่งดีพอสำหรับแอปส่วนใหญ่ที่ไม่จำเป็นต้องฝังใน iframe จากเว็บอื่น

### `Content-Security-Policy: frame-ancestors` — วิธีใหม่ที่ยืดหยุ่นกว่า

`frame-ancestors` ใน CSP ทำสิ่งเดียวกับ `X-Frame-Options` แต่มีความสามารถมากกว่า:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  # ห้ามทุกที่ (เข้มกว่า DENY):
  policy.frame_ancestors :none

  # หรืออนุญาตเฉพาะ origins ที่ระบุ:
  policy.frame_ancestors :self, "https://trusted-partner.com"
end
```

เมื่อมีทั้ง `X-Frame-Options` และ `frame-ancestors` ใน CSP browser สมัยใหม่จะเลือกใช้ `frame-ancestors`
แทน — ดังนั้นถ้าตั้ง CSP ให้ถูกต้องแล้ว `X-Frame-Options` ก็แค่เป็น fallback สำหรับ browser เก่า

### กรณีพิเศษ: แอปที่ต้องฝัง iframe ใน context บางอย่าง

ตัวอย่างเช่น payment widget ที่ merchant embed ในเว็บตัวเอง:

```ruby
class PaymentWidgetController < ApplicationController
  # override header เฉพาะ action นี้:
  before_action :allow_iframe_from_merchant

  private

  def allow_iframe_from_merchant
    response.headers["X-Frame-Options"] = "ALLOW-FROM #{merchant.domain}"
    # หรือใช้ CSP:
    response.set_header(
      "Content-Security-Policy",
      "frame-ancestors 'self' #{merchant.domain}"
    )
  end
end
```

---

## Step 797: Brakeman เชิงลึก — Check ประเภทที่เกิน SQL Injection

### ทบทวนสั้น: Brakeman คืออะไร (จาก Part 075)

**Brakeman** เป็น static analysis tool สำหรับ Rails โดยเฉพาะ ทำงานโดยวิเคราะห์ซอร์สโค้ดโดยตรง (ไม่รัน
แอป ไม่ต้องมี database) และค้นหา pattern ที่รู้จักว่าเป็นช่องโหว่ด้านความปลอดภัย

```bash
# รัน Brakeman:
bundle exec brakeman

# ผลลัพธ์แบบ JSON สำหรับ CI:
bundle exec brakeman -f json -o brakeman_report.json

# ดู check ทั้งหมดที่ Brakeman มี:
bundle exec brakeman --list-checks
```

### Check ที่เกี่ยวกับ Mass Assignment

**`MassAssignment` check** — Brakeman ตรวจหา:

```ruby
# Pattern ที่ Brakeman จะ flag:

# 1. update_attributes หรือ update โดยไม่ผ่าน Strong Parameters
User.find(id).update_attributes(params[:user])   # WARNING: Mass assignment

# 2. new() หรือ create() โดยไม่ผ่าน permit
User.new(params[:user])                          # WARNING: Mass assignment

# 3. assign_attributes โดยตรง
@user.assign_attributes(params[:user])           # WARNING: Mass assignment
```

ตัวอย่าง output ของ Brakeman เมื่อพบ MassAssignment:

```
+------------+-------+--------+-----------+------+----------------------+
| Confidence | Class | Method | Warning   | Line | Message              |
+------------+-------+--------+-----------+------+----------------------+
| High       | Users | update | Mass Assi | 12   | Unprotected mass    |
|            |       |        | gnment    |      | assignment           |
+------------+-------+--------+-----------+------+----------------------+
```

### Check ที่เกี่ยวกับ Configuration

**`ForceSSL` check** — ตรวจว่า production ได้เปิด `force_ssl`:

```
Warning: SSL is not forced in config/environments/production.rb
```

**`SessionSettings` check** — ตรวจ cookie settings:
- `secure: true` — ส่ง cookie เฉพาะ HTTPS
- `httponly: true` — JavaScript ใน browser ไม่สามารถอ่าน cookie ได้ (ป้องกัน XSS ขโมย session)

```ruby
# Rails ตั้ง httponly: true ให้ by default สำหรับ session cookie
# แต่ถ้าตั้ง secure: false ใน production จะถูก Brakeman flag:
Rails.application.config.session_store :cookie_store,
  key: "_my_app_session",
  secure: false   # WARNING: Session cookie without secure flag in production
```

**`DefaultRoutes` check** — ตรวจว่ายังมี `resources :debug` หรือ route ที่ไม่ควรมีใน production:

```ruby
# Brakeman จะ warn ถ้ามี:
namespace :admin do
  get "debug" => "debug#index"  # ไม่มี authentication guard
end
```

### Check ที่เกี่ยวกับ Code Patterns อันตราย

**`UnsafeReflection` check** — ตรวจหาการใช้ `constantize`, `send`, `public_send` กับ user input:

```ruby
# อันตราย — Brakeman จะ flag:
klass = params[:model].constantize           # WARNING: Dynamic class instantiation
klass.new(params[:attributes])

method_name = params[:action]
@object.send(method_name)                    # WARNING: Dynamic method dispatch
```

**`RemoteCodeExecution` / `Deserialize` check** — ตรวจหา YAML.load หรือ Marshal.load กับ user input:

```ruby
# อันตรายมาก — Remote Code Execution:
data = YAML.load(params[:data])              # WARNING: YAML.load with user input
# ใช้ YAML.safe_load แทน:
data = YAML.safe_load(params[:data])         # ปลอดภัย
```

**`Redirect` check** — ตรวจหา open redirect:

```ruby
# อันตราย:
redirect_to params[:return_to]               # WARNING: Open redirect

# ที่ถูกต้อง:
destination = URI.parse(params[:return_to]) rescue nil
if destination&.relative?
  redirect_to params[:return_to]
else
  redirect_to root_path
end
```

### Check ที่เกี่ยวกับ Template

**`CrossSiteScripting` check** — ตรวจหา `raw`, `html_safe`, `content_tag` ที่ใช้กับ user input:

```erb
<%# อันตราย: %>
<%= raw params[:message] %>                  <%# WARNING: Cross-Site Scripting %>
<%= params[:content].html_safe %>            <%# WARNING: Cross-Site Scripting %>

<%# ปลอดภัย (auto-escape): %>
<%= params[:message] %>
```

**`RenderInline` check** — ตรวจหาการ render ERB จาก user input:

```ruby
# อันตรายมาก:
render inline: params[:template]             # WARNING: Remote code execution
```

---

## Step 798: อ่านและจัดการ Brakeman Output อย่างมีระบบ — Confidence, Ignore File, False Positive vs. Real Finding

### โครงสร้าง Output ของ Brakeman

เมื่อรัน `bundle exec brakeman` จะได้ output 2 ส่วนหลัก:

```
== Warnings ==

Confidence: High
Category: Mass Assignment
Message: Unprotected mass assignment
Code: User.update(params[:user])
File: app/controllers/users_controller.rb
Line: 15

Confidence: Medium
Category: Cross-Site Scripting
Message: Possible XSS vulnerability in render
Code: render partial: params[:partial]
File: app/controllers/pages_controller.rb
Line: 8
```

**Confidence levels:**

| ระดับ | ความหมาย | วิธีจัดการ |
|-------|----------|-----------|
| **High** | Brakeman มั่นใจว่าเป็นช่องโหว่จริง (พบ pattern ที่อันตรายชัดเจน) | แก้ไขทันที |
| **Medium** | อาจเป็นช่องโหว่ ขึ้นอยู่กับ context | ตรวจสอบก่อน ถ้าจริงให้แก้ไข |
| **Weak** | Brakeman ไม่แน่ใจมาก อาจเป็น false positive สูง | ตรวจสอบ แต่ไม่ใช่ priority สูงสุด |

### การจัดการ False Positives ด้วย Ignore File

บางครั้ง Brakeman flag โค้ดที่เราตรวจสอบแล้วว่าปลอดภัยในบริบทนั้นๆ เช่น:

```ruby
# โค้ดนี้ดูเหมือน open redirect แต่จริงๆ ตรวจสอบแล้ว:
def redirect_after_login
  destination = session[:return_to] || root_path
  session.delete(:return_to)
  redirect_to destination  # Brakeman อาจ flag นี้
end
```

**สร้าง ignore file:**

```bash
# สร้าง / แก้ไข ignore file แบบ interactive:
bundle exec brakeman -I

# Brakeman จะแสดง warning ทีละอัน และถามว่า:
# (i)gnore / (a)dd note / (s)kip
```

Ignore file จะถูกสร้างที่ `config/brakeman.ignore`:

```json
{
  "ignored_warnings": [
    {
      "warning_type": "Redirect",
      "warning_code": 18,
      "fingerprint": "abc123...",
      "check_name": "Redirect",
      "message": "Possible unprotected redirect",
      "file": "app/controllers/sessions_controller.rb",
      "line": 42,
      "note": "Return URL มาจาก session ที่ server ตั้งไว้ ไม่ใช่ user input โดยตรง — verified safe"
    }
  ]
}
```

**สิ่งสำคัญ:** ignore file ต้อง **commit เข้า git** เพราะ:
1. ทีมทุกคนและ CI ต้องใช้ ignore file เดียวกัน
2. ทุก entry ควรมี `note` อธิบายว่าทำไมถึง ignore — เป็น audit trail

### กระบวนการ triage Brakeman findings

```
สำหรับแต่ละ warning ใน Brakeman output:

1. อ่าน warning type, confidence, และ message
2. เปิดไฟล์ที่ระบุ และอ่านโค้ดรอบๆ บรรทัดนั้น
3. ถามตัวเองว่า:
   - "user input ไปถึงจุดนี้ได้ไหม?"
   - "ถ้าไปถึงได้ ผลที่แย่ที่สุดคืออะไร?"
4. ถ้าเป็นช่องโหว่จริง → แก้ไขโค้ด
5. ถ้า verified safe → ignore ด้วย note ที่ชัดเจน
6. ไม่มีทางเลือก "ข้ามไปก่อน" โดยไม่มี note
```

### รัน Brakeman แล้วดู summary

```bash
bundle exec brakeman --summary

# Output:
# +---------------------+-------+
# | Scanned/Ignored     |       |
# +---------------------+-------+
# | Controllers         | 12    |
# | Models              | 8     |
# | Templates           | 34    |
# | Ignored warnings    | 3     |
# +---------------------+-------+
#
# +---------------------+-------+
# | Warnings Found      |       |
# +---------------------+-------+
# | Total               | 5     |
# | High confidence     | 2     |
# | Medium confidence   | 2     |
# | Weak confidence     | 1     |
# +---------------------+-------+
```

---

## Step 799: Brakeman ใน CI Pipeline (ทบทวนจาก Part 075) + Automation ที่ถูกต้อง

### ทบทวนจาก Part 075: Brakeman ใน GitHub Actions

Part 075 แสดงวิธีเพิ่ม Brakeman เข้า CI pipeline พื้นฐาน:

```yaml
# .github/workflows/security.yml (จาก Part 075)
name: Security

on: [push, pull_request]

jobs:
  brakeman:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Run Brakeman
        run: bundle exec brakeman --no-pager -q
```

### เพิ่ม threshold และ exit code handling

```yaml
# .github/workflows/security.yml (เวอร์ชันปรับปรุง)
- name: Run Brakeman
  run: |
    bundle exec brakeman \
      --no-pager \
      --quiet \
      --format json \
      --output brakeman_report.json \
      --exit-on-warn  # exit code 1 ถ้ามี warning (ทำให้ CI fail)

- name: Upload Brakeman report
  if: always()  # upload แม้ Brakeman จะ fail
  uses: actions/upload-artifact@v4
  with:
    name: brakeman-report
    path: brakeman_report.json
```

### สิ่งที่ต้องมีใน workflow ที่ดี

**1. ใช้ ignore file ร่วมกับ CI:**

```yaml
- name: Run Brakeman with ignore file
  run: |
    bundle exec brakeman \
      --ignore-config config/brakeman.ignore \
      --exit-on-warn
```

**2. ป้องกันไม่ให้ ignore file "เก่า" ทำให้ warning จริงหลุดออกไป:**

```yaml
- name: Check for expired ignores
  run: |
    bundle exec brakeman \
      --ignore-config config/brakeman.ignore \
      --prune-ignore-file \
      --format text
```

**3. แยก job security จาก job test:**

```yaml
# CI ควรมี 2 jobs แยกกัน:
jobs:
  test:
    # ... rspec, rubocop
  security:
    # ... brakeman, bundler-audit
    # แยกกันเพื่อให้เห็นชัดว่าส่วนไหน fail
```

### Integration กับ bundler-audit (ทบทวนจาก Part 075)

```yaml
- name: Check for vulnerable gems
  run: |
    gem install bundler-audit
    bundle audit check --update  # ดาวน์โหลด CVE database ล่าสุดก่อน
```

ทั้ง Brakeman (static analysis ของโค้ด) และ bundler-audit (ตรวจ gem versions ที่มี CVE) ทำงานคนละชั้นกัน
และควรรันทั้งคู่เสมอ — ไม่มีตัวไหนทดแทนอีกตัวได้

### ตัวอย่าง CI workflow สมบูรณ์สำหรับ Phase 13

```yaml
# .github/workflows/security.yml
name: Security Checks

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  brakeman:
    name: Static Analysis (Brakeman)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Run Brakeman
        run: |
          bundle exec brakeman \
            --no-pager \
            --quiet \
            --ignore-config config/brakeman.ignore \
            --exit-on-warn
      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: brakeman-report-${{ github.sha }}
          path: brakeman_report.json

  dependency-audit:
    name: Dependency Audit (bundler-audit)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Audit dependencies
        run: |
          gem install bundler-audit
          bundle audit check --update
```

---

## Step 800: แบบฝึกหัดปิด Part — Mass Assignment พิสูจน์จริง + Headers ตรวจด้วย curl + Brakeman Scan

### แบบฝึกหัดที่ 1: ตรวจสอบ Mass Assignment ในแอปของคุณ

เปิด controller ที่มี `update` หรือ `create` action ในแอปของคุณ (หรือแอปทดลอง) แล้วตอบคำถามเหล่านี้:

**Checklist:**

```
[ ] 1. ทุก update/create action ใช้ private method ที่เรียก params.require().permit() ไหม?
[ ] 2. ไม่มีที่ไหนใช้ params.require().permit! เลยใช่ไหม?
[ ] 3. nested attributes มีการ permit แบบ explicit keys ไหม (ไม่ใช่ {} แบบเปิดกว้าง)?
[ ] 4. มี request spec ที่ยืนยันว่า admin-only fields ถูกกรองออกเมื่อ regular user ส่งมาไหม?
```

**ทดสอบด้วย curl:**

```bash
# สมมติแอปรันอยู่ที่ localhost:3000 และ user id=1 มี admin: false
# รัน rails server ใน development แล้วลอง:

# 1. ดู CSRF token (ต้องใช้ใน development ถ้าเปิด CSRF protection):
curl -c cookies.txt -s http://localhost:3000/login | grep csrf

# 2. Login (ปรับตาม auth system ของแอป):
curl -c cookies.txt -b cookies.txt \
  -X POST http://localhost:3000/sessions \
  -d "email=user@example.com&password=password&authenticity_token=TOKEN"

# 3. ส่ง request update พร้อม field ที่ไม่ควรได้รับอนุญาต:
curl -c cookies.txt -b cookies.txt \
  -X PATCH http://localhost:3000/users/1 \
  -d "user[name]=Test&user[admin]=true&authenticity_token=TOKEN"

# 4. ตรวจสอบผลว่า admin ยังเป็น false:
# (ดูจาก response หรือ check database โดยตรง)
```

### แบบฝึกหัดที่ 2: ตรวจ HTTP Headers ด้วย curl

```bash
# รัน Rails server แล้วตรวจสอบ headers:
bundle exec rails server &

# ตรวจ headers ทั้งหมดที่ส่งมา:
curl -sI http://localhost:3000/ | sort

# Headers ที่ควรเห็นใน development:
# X-Frame-Options: SAMEORIGIN
# X-Content-Type-Options: nosniff
# Referrer-Policy: strict-origin-when-cross-origin

# ตรวจว่า X-Frame-Options ป้องกัน iframe จาก origin อื่นจริง:
# สร้างไฟล์ test_iframe.html บน domain อื่น แล้วดูใน browser console ว่า block หรือเปล่า
```

**เพิ่ม Permissions-Policy ถ้ายังไม่มี:**

```ruby
# config/initializers/permissions_policy.rb
Rails.application.config.permissions_policy do |policy|
  policy.camera      :none
  policy.microphone  :none
  policy.geolocation :none
  policy.usb         :none
end
```

แล้วตรวจยืนยัน:

```bash
curl -sI http://localhost:3000/ | grep -i permissions
# ควรเห็น: Permissions-Policy: camera=(), microphone=(), ...
```

### แบบฝึกหัดที่ 3: รัน Brakeman แล้ว triage ผลลัพธ์

```bash
# รัน Brakeman ในแอปของคุณ:
bundle exec brakeman --no-pager

# สำหรับแต่ละ warning ที่ได้:
# 1. เปิดไฟล์และบรรทัดที่ระบุ
# 2. วิเคราะห์ว่าเป็น real vulnerability หรือ false positive
# 3. ถ้า real → แก้ไข
# 4. ถ้า false positive → เพิ่มเข้า ignore file พร้อม note

# สร้าง ignore file ถ้ายังไม่มี:
bundle exec brakeman -I

# ตรวจสอบว่า ignore file ถูก commit เข้า git:
git add config/brakeman.ignore
git status
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. สร้าง User model ที่มีฟิลด์ `subscription_tier` (free/pro/enterprise) และ `monthly_fee` แล้วเขียน
   Strong Parameters ที่อนุญาตให้ admin เปลี่ยน `subscription_tier` ได้ แต่ user ธรรมดาทำไม่ได้ และ
   ไม่มีใครเปลี่ยน `monthly_fee` ผ่าน web form ได้เลย (คำนวณโดย system เท่านั้น) เขียน request spec
   ยืนยันทั้งสามกรณีนี้

2. เพิ่ม `Content-Security-Policy` ที่ป้องกัน inline script (ไม่มี `unsafe-inline`) ในแอปทดลอง แล้ว
   ตรวจสอบว่า `frame-ancestors 'self'` ทำงานถูกต้องโดยสร้าง HTML file บน origin อื่น (เช่น เปิด
   `file://` ใน browser) แล้วพยายาม embed แอปใน iframe ดูว่า browser block หรือเปล่า

3. ในโค้ดที่มีอยู่ของคุณ ลองหา pattern ที่ Brakeman อาจ flag แล้ว fix ก่อนที่ Brakeman จะหา
   (เช่น `YAML.load`, `constantize` กับ params, redirect ที่ไม่ validate destination URL) แล้วรัน
   Brakeman หลังแก้เพื่อยืนยันว่า warning หายไปจริง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจกลไกของ **Mass Assignment** จากรากเหง้า — ทำไม Rails ต้องสร้าง Strong Parameters ขึ้นมาแทน
  `attr_accessible` และอะไรคือ pattern ที่ดูปลอดภัยแต่ยังเสี่ยงอยู่ (`permit!`, nested hash แบบเปิด
  กว้าง) พร้อมพิสูจน์การโจมตี admin escalation ด้วยของจริง
- รู้จัก **Secure HTTP Headers** ที่ Rails ตั้งให้ by default และ headers ที่ต้องตั้งเพิ่มเอง โดยเฉพาะ
  `Strict-Transport-Security` (ต้องเปิด `force_ssl`), `Content-Security-Policy` (สอนแล้วใน Part 079),
  และ `Permissions-Policy` — พร้อมวิธีตรวจยืนยันด้วย `curl`
- เข้าใจ **Clickjacking** และวิธีที่ `X-Frame-Options: SAMEORIGIN` และ `frame-ancestors` ใน CSP
  ป้องกันการฝังแอปใน iframe ของเว็บอื่น
- อ่านและตีความ **Brakeman output** ได้ในระดับเชิงลึก: รู้จัก check ประเภทต่างๆ นอกเหนือจาก SQL
  Injection (MassAssignment, UnsafeReflection, RemoteCodeExecution, Redirect, CrossSiteScripting)
  เข้าใจ confidence levels และรู้วิธีจัดการ false positives ด้วย ignore file ที่มี note ชัดเจน
- ตั้งค่า **Brakeman ใน CI pipeline** อย่างถูกต้อง รวมถึงการใช้ ignore file ร่วมกับ CI และ integration
  กับ bundler-audit เพื่อครอบคลุมทั้ง static code analysis และ known CVE ใน dependencies

**ต่อไป (Part 081):** Part สุดท้ายของ Phase 13 จะเจาะลึก **Secrets Management และ ActiveRecord
Encryption** — ป้องกันข้อมูลที่ละเอียดอ่อนในฐานข้อมูลแม้แต่เมื่อมีคนเข้าถึง database โดยตรง และปิดท้าย
ด้วย **Security Checklist ก่อนขึ้น Production** ที่รวบรวมทุกหัวข้อจากทั้ง Phase 13 (และบาง Phase ก่อน
หน้า) ให้เป็นรายการตรวจสอบเดียวที่ใช้ได้จริง
