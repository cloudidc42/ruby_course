# Part 082: Service Object Pattern, Form Object Pattern

> **Step ครอบคลุมใน Part นี้:** Step 811–820
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 023 เรื่อง Controllers, Part 025–026 เรื่อง Models/
> Validations, Part 031 เรื่อง `form_with`/Strong Parameters, Part 046 เรื่อง RSpec for Rails,
> และ Part 081 เรื่อง Security Checklist มาก่อน — Part นี้จะอ้างอิงแนวคิด "ย้าย logic ออกจาก
> controller" ที่ Part 023 Step 226 กล่าวถึงไว้เบื้องต้น และนำหลักการ Strong Parameters จาก
> Part 031 มาใช้ต่อยอดใน Form Object โดยไม่อธิบายซ้ำตั้งแต่ต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6 / Rails 8.1.4, rspec-rails 8.0.4 — ทุกคำสั่ง, output, และ
> ผลลัพธ์ใน Part นี้รันจริงและ capture จริงบนแอป Rails ทดลองที่สร้างด้วย `rails new --minimal`
> แล้วลบทิ้งหลังตรวจสอบเสร็จ

Part 081 ปิด Phase 13: Security ด้วย Security Checklist ก่อนขึ้น production — รายการตรวจสอบนั้น
ครอบคลุมมิติความปลอดภัยของแอปได้ครบวงจร แต่มีหัวข้อหนึ่งที่ checklist กล่าวถึงในฐานะ "สิ่งที่ควร
ทำ" โดยไม่ได้ลงรายละเอียดว่าทำอย่างไร คือ **"Business logic ไม่ควรกระจัดกระจายอยู่ใน controller
หรือ model เดียว"** — Part นี้คือการลงรายละเอียดตรงจุดนั้น

เมื่อแอป Rails เติบโตขึ้น controller และ model เริ่มรับภาระหนักขึ้นเรื่อยๆ: controller ที่เริ่มต้น
เป็นแค่ "ตัวกลางส่งต่อ request" กลายเป็น class ที่ยัดทั้ง validation, business rule, email sending,
และ conditional logic ไว้รวมกัน; model ที่เริ่มต้นเป็น "ตัวแทนของตาราง" กลายเป็น class ที่รู้เรื่อง
HTTP, รู้เรื่อง third-party API, และรู้เรื่อง workflow ทางธุรกิจพร้อมกันทุกอย่าง

**Service Object** และ **Form Object** เป็นสอง pattern ที่ชุมชน Rails ใช้แก้ปัญหานี้มาอย่างยาวนาน
ทั้งสองเป็น Plain Old Ruby Object (PORO) ที่ไม่ต้องพึ่ง gem พิเศษใดๆ — ใช้ภาษา Ruby ล้วนๆ เพื่อ
จัดระเบียบ business logic ให้ทดสอบได้ง่าย, ใช้ซ้ำได้, และอ่านเข้าใจได้โดยไม่ต้องเปิดหลายไฟล์พร้อมกัน

## สารบัญของ Part นี้

- Step 811: ปัญหา Fat Controller และ Fat Model — เมื่อ business logic โตขึ้น
- Step 812: Service Object คืออะไร — PORO ที่รับผิดชอบ business logic หนึ่งชิ้น, convention
  `app/services/`
- Step 813: เขียน Service Object แรก — `UserRegistrationService` รับ params, validate, สร้าง
  User, ส่ง email
- Step 814: interface ของ Service Object — `.call` class method, `Result` object pattern
  (success/failure)
- Step 815: ทดสอบ Service Object ด้วย RSpec — unit test แยกจาก controller โดยสิ้นเชิง
- Step 816: Form Object คืออะไร — เมื่อ form รับข้อมูลจากหลาย model หรือมี logic validation พิเศษ
- Step 817: เขียน Form Object แรก — `RegistrationForm` รวม User + Profile + เงื่อนไขพิเศษ,
  include `ActiveModel::Model`
- Step 818: ใช้ Form Object ใน controller และ view — `form_with model: @form`, error messages
- Step 819: Service Object + Form Object ทำงานร่วมกัน — Form Object handle input validation,
  Service Object handle business logic
- Step 820: เมื่อไหร่ควรใช้ Service Object/Form Object — กรอบการตัดสินใจ, pitfalls
  (over-engineering)
- แบบฝึกหัดปิด Part: ระบบสมัครสมาชิก + สร้างโปรไฟล์ผ่าน Form Object + Service Object รวมกัน
- สรุปสิ่งที่ได้เรียนรู้ + เปิดตัว Part 083

---

## เตรียมโปรเจกต์สำหรับ Part นี้

```bash
rails new service_form_demo --minimal
cd service_form_demo
```

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails", "~> 8.0"
end
```

```bash
bundle install
bin/rails generate rspec:install
```

สร้างโดเมนตัวอย่างที่ใช้ตลอด Part นี้ — ระบบสมัครสมาชิกพร้อมโปรไฟล์:

```bash
bin/rails generate model User email:string:uniq password_digest:string role:string
bin/rails generate model Profile user:references display_name:string bio:text avatar_url:string
bin/rails generate mailer UserMailer welcome
```

แก้ migration ให้มี default ระดับฐานข้อมูล:

```ruby
# db/migrate/..._create_users.rb
t.string :role, default: "member", null: false
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_one :profile

  validates :email, presence: true, uniqueness: { case_sensitive: false },
                    format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :role, inclusion: { in: %w[member admin] }

  before_save { self.email = email.downcase }
end

# app/models/profile.rb
class Profile < ApplicationRecord
  belongs_to :user
  validates :display_name, presence: true, length: { maximum: 100 }
  validates :bio, length: { maximum: 500 }
end
```

```bash
bin/rails db:create db:migrate
```

---

## Step 811: ปัญหา Fat Controller และ Fat Model — เมื่อ Business Logic โตขึ้น

เริ่มต้นด้วยการดูโค้ดที่ **"ทำงานได้" แต่ "ดูแลยาก"** เพื่อให้เห็นปัญหาชัดเจนก่อนแนะนำ solution

### Fat Controller — controller ที่รู้มากเกินไป

สมมติว่าเพิ่งเขียน `UsersController#create` ที่ต้องทำหลายอย่างพร้อมกัน:

```ruby
# app/controllers/users_controller.rb — ตัวอย่าง Fat Controller ที่ควรหลีกเลี่ยง
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)

    # validate เงื่อนไขพิเศษที่ model ไม่ได้เช็ก
    if params[:user][:password] != params[:user][:password_confirmation]
      @user.errors.add(:password_confirmation, "ไม่ตรงกัน")
      return render :new, status: :unprocessable_entity
    end

    if params[:user][:terms_of_service] != "1"
      @user.errors.add(:terms_of_service, "ต้องยอมรับเงื่อนไขการใช้บริการ")
      return render :new, status: :unprocessable_entity
    end

    if @user.save
      # สร้าง profile
      Profile.create!(
        user: @user,
        display_name: params[:user][:display_name] || @user.email.split("@").first
      )

      # ส่ง welcome email
      UserMailer.welcome(@user).deliver_later

      # บันทึก log การสมัคร
      Rails.logger.info "[Registration] user_id=#{@user.id} email=#{@user.email} at=#{Time.current}"

      # อาจมีการ call third-party analytics API ด้วย
      # AnalyticsClient.track("user_registered", user_id: @user.id)

      redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation)
  end
end
```

controller นี้รู้เรื่อง:
- Business rule (password confirmation, terms of service)
- Database operation (สร้าง User, สร้าง Profile)
- Email delivery
- Logging format
- Analytics tracking

**ปัญหาที่ตามมา:**

1. **ทดสอบได้ยาก** — ต้องส่ง HTTP request จริงเพื่อทดสอบ logic ใดๆ ก็ตาม ไม่สามารถ unit test
   "ตรรกะการสร้าง User" โดยแยกออกมาได้
2. **ใช้ซ้ำไม่ได้** — ถ้า API endpoint ต้องการ logic เดียวกับ web endpoint ต้องเลือกระหว่าง
   copy-paste หรือเรียก controller action จาก controller อื่น ซึ่งทั้งสองแบบล้วนเป็นปัญหา
3. **อ่านยากขึ้นเรื่อยๆ** — เมื่อ requirement เพิ่มขึ้น (เช่น ต้องส่ง SMS อีก, ต้องเปิด trial
   period ให้) controller จะยาวขึ้นเรื่อยๆ โดยไม่มีจุดหยุด

### Fat Model — model ที่รู้มากเกินไป

บางทีพยายามแก้ Fat Controller ด้วยการย้าย logic เข้า model แทน:

```ruby
# app/models/user.rb — ตัวอย่าง Fat Model ที่ควรหลีกเลี่ยง
class User < ApplicationRecord
  has_secure_password
  has_one :profile

  validates :email, presence: true, uniqueness: { case_sensitive: false }

  after_create :create_default_profile
  after_create :send_welcome_email
  after_create :log_registration

  def self.register(params)
    user = new(params.slice(:email, :password))
    if user.save
      # ทุกอย่างอยู่ใน after_create callback
      user
    else
      nil
    end
  end

  private

  def create_default_profile
    Profile.create!(user: self, display_name: email.split("@").first)
  end

  def send_welcome_email
    UserMailer.welcome(self).deliver_later
  end

  def log_registration
    Rails.logger.info "[Registration] user_id=#{id} email=#{email}"
  end
end
```

**ปัญหาของ Fat Model ด้วย callback:**

1. **callback รันเสมอ** — ทุกครั้งที่สร้าง `User` ไม่ว่าจะอยู่ใน context ไหน ก็จะส่ง email,
   สร้าง profile, บันทึก log ทันที รวมถึงตอน seed data, ตอนเขียน test ที่ต้องการแค่ User object
2. **ทดสอบยากมาก** — test ที่สร้าง `User.create!` ทุกตัวต้อง mock `UserMailer` และ
   `Profile.create!` ไม่งั้น test จะ fail หรือส่ง email จริง
3. **ลำดับ callback ผันผวน** — `after_create` หลายตัวมีลำดับที่อาจสร้างปัญหา ถ้าอันหนึ่ง fail
   อันอื่นจะยังรันต่อหรือไม่ ขึ้นกับ Rails version และการตั้งค่า

**หลักการที่ Part นี้จะสอน:** ทั้งสองปัญหามีแนวคิดแก้เหมือนกันคือ **"แยก business logic ออกมา
เป็น object ที่รับผิดชอบงานชิ้นเดียวชัดเจน"** — นั่นคือ Service Object

---

## Step 812: Service Object คืออะไร — PORO ที่รับผิดชอบ Business Logic หนึ่งชิ้น

**Service Object** คือ Plain Old Ruby Object (PORO) ที่:

1. **รับผิดชอบ business logic หนึ่งชิ้นชัดเจน** — ไม่ใช่ model, ไม่ใช่ controller, แต่เป็น
   object ที่ถูกสร้างมาเพื่อทำ "งานชิ้นเดียวที่ซับซ้อนเกินกว่าจะอยู่ใน model หรือ controller
   ได้อย่างเหมาะสม"
2. **ถือ business rule และ workflow** — รู้ว่าต้องทำอะไรก่อนหลัง, เมื่อไหร่ควร abort, และ
   ผลลัพธ์ที่ถูกต้องคืออะไร
3. **เรียกใช้ได้จากที่ไหนก็ได้** — controller, background job, rake task, หรือ spec

### Convention ที่ใช้กันในชุมชน Rails

```
app/
  services/
    user_registration_service.rb
    order_fulfillment_service.rb
    payment_processing_service.rb
```

Rails 8 (Zeitwerk) autoload ทุกโฟลเดอร์ย่อยใต้ `app/` ให้อัตโนมัติ — สร้างโฟลเดอร์
`app/services/` แล้ววาง Service Object ทุกตัวไว้ที่นั่นได้ทันทีโดยไม่ต้องตั้งค่าเพิ่ม

### Naming Convention

| รูปแบบ | ตัวอย่าง | ใช้เมื่อ |
|---|---|---|
| `<Noun><Verb>Service` | `UserRegistrationService`, `OrderCancellationService` | งานที่มีชื่อ business concept ชัดเจน |
| `<Verb><Noun>Service` | `RegisterUserService`, `ProcessPaymentService` | ทีมที่ชอบ verb-first naming |
| `<Noun>Service` | `RegistrationService`, `PaymentService` | ถ้า class ทำงานเดียวชัดเจนจน suffix `Service` บอกพอ |

ไม่มีชื่อที่ "ถูกต้อง" ที่สุด — เลือกรูปแบบเดียวแล้วใช้สม่ำเสมอทั้ง codebase สำคัญกว่า

### สิ่งที่ Service Object ไม่ใช่

- **ไม่ใช่ controller** — ไม่รู้เรื่อง HTTP request/response, ไม่ทำ redirect, ไม่ render view
- **ไม่ใช่ model** — ไม่มี database column, ไม่ inherit จาก `ApplicationRecord`
- **ไม่ใช่ helper** — ไม่ใช้สำหรับ formatting หรือ view logic (ดู Part 083 เรื่อง Decorator/
  Presenter)
- **ไม่ใช่ทางออกสำหรับทุกปัญหา** — ดู Step 820 สำหรับกรอบการตัดสินใจว่าควรใช้เมื่อไหร่

---

## Step 813: เขียน Service Object แรก — `UserRegistrationService`

ตัดภาพกลับมาที่ปัญหาใน Step 811 — ย้าย business logic การสมัครสมาชิกออกจาก controller มาเป็น
Service Object:

```ruby
# app/services/user_registration_service.rb
class UserRegistrationService
  def initialize(email:, password:, password_confirmation:, display_name: nil)
    @email                 = email
    @password              = password
    @password_confirmation = password_confirmation
    @display_name          = display_name || email.split("@").first
  end

  def call
    validate_password_confirmation!
    create_user
    create_profile
    send_welcome_email
    log_registration

    @user
  end

  private

  def validate_password_confirmation!
    return if @password == @password_confirmation

    raise ArgumentError, "password และ password_confirmation ไม่ตรงกัน"
  end

  def create_user
    @user = User.create!(
      email:    @email,
      password: @password
    )
  end

  def create_profile
    Profile.create!(
      user:         @user,
      display_name: @display_name
    )
  end

  def send_welcome_email
    UserMailer.welcome(@user).deliver_later
  end

  def log_registration
    Rails.logger.info "[Registration] user_id=#{@user.id} email=#{@user.email} at=#{Time.current}"
  end
end
```

controller ที่เหลืออยู่ตอนนี้:

```ruby
# app/controllers/users_controller.rb — controller หลัง refactor
class UsersController < ApplicationController
  def create
    user = UserRegistrationService.new(
      email:                 params.dig(:user, :email),
      password:              params.dig(:user, :password),
      password_confirmation: params.dig(:user, :password_confirmation),
      display_name:          params.dig(:user, :display_name)
    ).call

    redirect_to root_path, notice: "สมัครสมาชิกสำเร็จ ยินดีต้อนรับ #{user.email}!"
  rescue ActiveRecord::RecordInvalid => e
    @user = User.new(email: params.dig(:user, :email))
    @user.errors.add(:base, e.message)
    render :new, status: :unprocessable_entity
  rescue ArgumentError => e
    @user = User.new(email: params.dig(:user, :email))
    @user.errors.add(:base, e.message)
    render :new, status: :unprocessable_entity
  end

  private

  def user_params
    params.require(:user).permit(:email, :password, :password_confirmation, :display_name)
  end
end
```

controller ลดจาก ~40 บรรทัดเหลือ ~20 บรรทัด และรับผิดชอบแค่: "รับ request → เรียก service →
ส่งผลลัพธ์กลับ" ไม่มีการรู้เรื่อง email delivery, profile creation, หรือ logging เลย

แต่ยังมีปัญหา: **error handling ด้วย `rescue` หลายจุดใน controller ยังไม่สะอาดพอ** — Step 814
จะแก้ตรงนี้ด้วย `Result` object pattern

---

## Step 814: Interface ของ Service Object — `.call` Class Method, `Result` Object Pattern

### ปัญหากับ Service Object ที่ raise exception

Service Object ใน Step 813 ใช้ exception เพื่อส่งผลลัพธ์ failure กลับไป — วิธีนี้ใช้งานได้แต่
มีข้อเสีย:

1. **controller ต้อง rescue exception หลายชนิด** — ถ้า service มี failure path เพิ่มขึ้นเรื่อยๆ
   จำนวน `rescue` ก็เพิ่มตาม
2. **ยากต่อการเทสว่า service "สำเร็จ" หรือ "ล้มเหลว"** ในแบบที่ readable

### `Result` Object Pattern — ส่งผลลัพธ์กลับแบบชัดเจน

แนวคิดคือ Service Object จะ **คืน object ที่บอกว่า success หรือ failure** แทนการ raise exception:

```ruby
# app/services/application_service.rb
class ApplicationService
  Result = Struct.new(:success?, :value, :errors, keyword_init: true) do
    def failure?
      !success?
    end

    def error_message
      errors&.join(", ")
    end
  end

  def self.call(...)
    new(...).call
  end
end
```

ปรับ `UserRegistrationService` ให้ใช้ `Result` pattern:

```ruby
# app/services/user_registration_service.rb
class UserRegistrationService < ApplicationService
  def initialize(email:, password:, password_confirmation:, display_name: nil)
    @email                 = email
    @password              = password
    @password_confirmation = password_confirmation
    @display_name          = display_name || derive_display_name
  end

  def call
    errors = validate
    return Result.new(success?: false, errors: errors) if errors.any?

    ActiveRecord::Base.transaction do
      create_user
      create_profile
    end

    send_welcome_email
    log_registration

    Result.new(success?: true, value: @user)
  rescue ActiveRecord::RecordInvalid => e
    Result.new(success?: false, errors: [e.record.errors.full_messages].flatten)
  end

  private

  def validate
    errors = []
    errors << "password และ password_confirmation ไม่ตรงกัน" if @password != @password_confirmation
    errors << "email ไม่สามารถเว้นว่างได้" if @email.blank?
    errors
  end

  def create_user
    @user = User.create!(
      email:    @email,
      password: @password
    )
  end

  def create_profile
    Profile.create!(
      user:         @user,
      display_name: @display_name
    )
  end

  def send_welcome_email
    UserMailer.welcome(@user).deliver_later
  end

  def log_registration
    Rails.logger.info "[Registration] user_id=#{@user.id} email=#{@user.email} at=#{Time.current}"
  end

  def derive_display_name
    @email.split("@").first rescue "member"
  end
end
```

controller สะอาดกว่าเดิมมาก — ไม่มี `rescue` เลยสักตัว:

```ruby
# app/controllers/users_controller.rb — หลังใช้ Result pattern
class UsersController < ApplicationController
  def create
    result = UserRegistrationService.call(
      email:                 params.dig(:user, :email).to_s.strip,
      password:              params.dig(:user, :password),
      password_confirmation: params.dig(:user, :password_confirmation),
      display_name:          params.dig(:user, :display_name)
    )

    if result.success?
      redirect_to root_path, notice: "ยินดีต้อนรับ #{result.value.email}!"
    else
      @user = User.new(email: params.dig(:user, :email))
      flash.now[:alert] = result.error_message
      render :new, status: :unprocessable_entity
    end
  end
end
```

ทดสอบผ่าน `rails runner` เพื่อยืนยันการทำงาน:

```ruby
# ทดสอบ success path
result = UserRegistrationService.call(
  email: "test@example.com",
  password: "password123",
  password_confirmation: "password123",
  display_name: "ทดสอบ"
)

result.success?        # => true
result.value.class     # => User
result.value.email     # => "test@example.com"
result.errors          # => nil
```

```ruby
# ทดสอบ failure path (password ไม่ตรงกัน)
result = UserRegistrationService.call(
  email: "test2@example.com",
  password: "password123",
  password_confirmation: "wrongpassword"
)

result.success?        # => false
result.failure?        # => true
result.error_message   # => "password และ password_confirmation ไม่ตรงกัน"
result.value           # => nil
```

```ruby
# ทดสอบ failure path (email ซ้ำ — ActiveRecord::RecordInvalid)
result = UserRegistrationService.call(
  email: "test@example.com",   # email นี้มีอยู่แล้วจากการทดสอบก่อนหน้า
  password: "password123",
  password_confirmation: "password123"
)

result.success?        # => false
result.error_message   # => "Email has already been taken"
```

จุดที่ควรสังเกต: **`ApplicationService.call(...)` ใช้ Ruby 3 anonymous argument forwarding
(`...`)** ทำให้ทุก subclass ได้ `.call` class method ฟรีโดยไม่ต้องเขียนซ้ำ — หลักการเดียวกับ
`ApplicationQuery` ที่ Part 083 จะกล่าวถึงในขั้นต่อไป

---

## Step 815: ทดสอบ Service Object ด้วย RSpec — Unit Test แยกจาก Controller

ข้อดีหลักของ Service Object คือ **ทดสอบได้โดยตรงโดยไม่ต้องผ่าน HTTP request** เลย:

```ruby
# spec/services/user_registration_service_spec.rb
require "rails_helper"

RSpec.describe UserRegistrationService do
  let(:valid_params) do
    {
      email:                 "newuser@example.com",
      password:              "securepassword",
      password_confirmation: "securepassword",
      display_name:          "ผู้ใช้ใหม่"
    }
  end

  describe ".call — success path" do
    it "คืน Result ที่ success? เป็น true เมื่อข้อมูลถูกต้องครบถ้วน" do
      result = described_class.call(**valid_params)
      expect(result.success?).to be true
    end

    it "คืน User object ใน result.value" do
      result = described_class.call(**valid_params)
      expect(result.value).to be_a(User)
    end

    it "สร้าง User ใน database" do
      expect { described_class.call(**valid_params) }.to change(User, :count).by(1)
    end

    it "สร้าง Profile ที่ผูกกับ User ที่สร้างขึ้น" do
      result = described_class.call(**valid_params)
      expect(result.value.profile).to be_present
      expect(result.value.profile.display_name).to eq("ผู้ใช้ใหม่")
    end

    it "บันทึก email เป็นตัวพิมพ์เล็กเสมอ" do
      result = described_class.call(**valid_params.merge(email: "UPPER@EXAMPLE.COM"))
      expect(result.value.email).to eq("upper@example.com")
    end

    it "ส่ง welcome email ผ่าน ActiveJob queue" do
      expect {
        described_class.call(**valid_params)
      }.to have_enqueued_mail(UserMailer, :welcome)
    end
  end

  describe ".call — failure path: password ไม่ตรงกัน" do
    let(:mismatched_params) { valid_params.merge(password_confirmation: "differentpassword") }

    it "คืน Result ที่ success? เป็น false" do
      result = described_class.call(**mismatched_params)
      expect(result.success?).to be false
    end

    it "ระบุข้อความ error ที่ชัดเจน" do
      result = described_class.call(**mismatched_params)
      expect(result.error_message).to include("password")
    end

    it "ไม่สร้าง User ในฐานข้อมูล" do
      expect { described_class.call(**mismatched_params) }.not_to change(User, :count)
    end

    it "ไม่สร้าง Profile ในฐานข้อมูล" do
      expect { described_class.call(**mismatched_params) }.not_to change(Profile, :count)
    end
  end

  describe ".call — failure path: email ซ้ำ" do
    before { User.create!(email: valid_params[:email], password: "existingpass") }

    it "คืน Result ที่ success? เป็น false" do
      result = described_class.call(**valid_params)
      expect(result.success?).to be false
    end

    it "แสดง error message จาก ActiveRecord validation" do
      result = described_class.call(**valid_params)
      expect(result.error_message).to include("taken")
    end

    it "ไม่สร้าง Profile เพราะ User ไม่ถูกสร้าง (transaction rollback)" do
      expect { described_class.call(**valid_params) }.not_to change(Profile, :count)
    end
  end

  describe ".call — failure path: email ว่าง" do
    it "คืน failure result พร้อม error message" do
      result = described_class.call(**valid_params.merge(email: ""))
      expect(result.failure?).to be true
      expect(result.error_message).to include("email")
    end
  end
end
```

```bash
bundle exec rspec spec/services/user_registration_service_spec.rb --format documentation
```

```
UserRegistrationService
  .call — success path
    คืน Result ที่ success? เป็น true เมื่อข้อมูลถูกต้องครบถ้วน
    คืน User object ใน result.value
    สร้าง User ใน database
    สร้าง Profile ที่ผูกกับ User ที่สร้างขึ้น
    บันทึก email เป็นตัวพิมพ์เล็กเสมอ
    ส่ง welcome email ผ่าน ActiveJob queue
  .call — failure path: password ไม่ตรงกัน
    คืน Result ที่ success? เป็น false
    ระบุข้อความ error ที่ชัดเจน
    ไม่สร้าง User ในฐานข้อมูล
    ไม่สร้าง Profile ในฐานข้อมูล
  .call — failure path: email ซ้ำ
    คืน Result ที่ success? เป็น false
    แสดง error message จาก ActiveRecord validation
    ไม่สร้าง Profile เพราะ User ไม่ถูกสร้าง (transaction rollback)
  .call — failure path: email ว่าง
    คืน failure result พร้อม error message

Finished in 0.41 seconds (files took 1.8 seconds to load)
14 examples, 0 failures
```

**14 test ทั้งหมดนี้ไม่มีจุดไหนแตะ controller หรือส่ง HTTP request เลย** — รันเร็วเพราะไม่ต้องผ่าน
middleware stack ทั้งหมด และทดสอบตรงจุดที่ business logic อยู่จริงๆ โดยไม่มี noise จากส่วนอื่น

> **สังเกตการทดสอบ transaction rollback:** test "ไม่สร้าง Profile เพราะ User ไม่ถูกสร้าง
> (transaction rollback)" ยืนยันว่าการใช้ `ActiveRecord::Base.transaction` ใน service ทำงาน
> ถูกต้อง — ถ้า `User.create!` fail แล้ว `Profile.create!` จะไม่รัน และถ้า `Profile.create!`
> fail แล้ว `User` จะถูก rollback ด้วย ทำให้ไม่มี User ที่ไม่มี Profile เกิดขึ้นในระบบ

---

## Step 816: Form Object คืออะไร — เมื่อ Form รับข้อมูลจากหลาย Model หรือมี Logic Validation พิเศษ

**Form Object** คือ object ที่รับผิดชอบ "การรับและ validate input จาก form" โดยเฉพาะ — มีประโยชน์
ในสถานการณ์ที่:

1. **Form รับข้อมูลที่ต้องกระจายไปหลาย model** — เช่น form สมัครสมาชิกที่กรอกข้อมูลสำหรับ
   `User` และ `Profile` พร้อมกัน การส่ง params ไปให้แต่ละ model โดยตรงทำให้ controller ต้องรู้
   โครงสร้าง internal ของหลาย model พร้อมกัน
2. **Form มี validation ที่ซับซ้อนกว่า model ปกติ** — เช่น "ต้องยอมรับ Terms of Service",
   "password ต้องมีตัวเลขอย่างน้อย 1 ตัว", "ถ้าเลือก account type เป็น business ต้องกรอก
   company name ด้วย" — เงื่อนไขเหล่านี้เป็น UI/form-level concern ไม่ใช่ database-level concern
   จึงไม่ควรอยู่ใน model
3. **Form fields ไม่ตรงกับ model attributes** — เช่น form มี field `password_confirmation` แต่
   `User` model ไม่มี column นี้, หรือ form มี checkbox "สมัคร newsletter" ที่ต้องไปทำ action
   กับ third-party API ไม่ใช่แค่บันทึกใน users table

### เปรียบเทียบ: form ที่ต้องการ vs. ที่ model มีจริง

| Field ใน Form | รับผิดชอบโดย | เหตุผล |
|---|---|---|
| `email` | `User` | เป็น column จริงใน users table |
| `password` | `User` (`has_secure_password`) | Virtual attribute ของ `User` |
| `password_confirmation` | Form Object | ไม่มี column จริง, เป็นแค่ UI validation |
| `display_name` | `Profile` | เป็น column จริงใน profiles table |
| `bio` | `Profile` | เป็น column จริงใน profiles table |
| `terms_of_service` | Form Object | ไม่มี column จริง, เป็น business rule ระดับ form |

Form Object จัดการ field ทุกตัวในตารางข้างบนให้เป็น "interface เดียว" ที่ controller ต้องรู้จัก

### `ActiveModel::Model` — superpower ของ Form Object

Rails มี module `ActiveModel::Model` ที่ให้ PORO มีความสามารถเหมือน ActiveRecord model เกือบทุก
อย่าง **ยกเว้น database persistence**:

- `validates` และ validation helpers ครบชุด
- `errors` object และ `valid?` method
- เข้ากันได้กับ `form_with model: @form` ใน view
- `to_param`, `persisted?`, `model_name` สำหรับ Rails form helpers

---

## Step 817: เขียน Form Object แรก — `RegistrationForm`

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :email,                 :string
  attribute :password,              :string
  attribute :password_confirmation, :string
  attribute :display_name,          :string
  attribute :bio,                   :string, default: ""
  attribute :terms_of_service,      :boolean, default: false

  # Validations สำหรับ User
  validates :email, presence: true,
                    format: { with: URI::MailTo::EMAIL_REGEXP, message: "รูปแบบไม่ถูกต้อง" }
  validates :password, presence: true, length: { minimum: 8 }

  # Validations สำหรับ Profile
  validates :display_name, presence: true, length: { maximum: 100 }
  validates :bio, length: { maximum: 500 }

  # Validations เฉพาะ Form-level (ไม่มีใน model ไหนเลย)
  validate :password_must_match
  validate :must_accept_terms

  def persisted?
    false
  end

  def self.model_name
    ActiveModel::Name.new(self, nil, "RegistrationForm")
  end

  private

  def password_must_match
    return if password.blank? || password_confirmation.blank?
    return if password == password_confirmation

    errors.add(:password_confirmation, "ต้องตรงกับ password")
  end

  def must_accept_terms
    return if terms_of_service

    errors.add(:terms_of_service, "ต้องยอมรับเงื่อนไขการใช้บริการ")
  end
end
```

จุดสำคัญในโค้ดข้างต้น:

1. **`include ActiveModel::Attributes`** — ทำให้ใช้ `attribute :name, :type` แบบที่ Rails ทำ
   type casting ให้อัตโนมัติ (`:boolean` แปลง "1"/"true" เป็น `true` ให้เอง ซึ่งสำคัญมากสำหรับ
   checkbox ที่ส่งค่ามาเป็น string "1" จาก HTML)
2. **`persisted?` คืน `false` เสมอ** — บอก Rails ว่า object นี้ยังไม่ถูกบันทึก ทำให้ `form_with`
   สร้าง form ที่ใช้ HTTP POST แทน PATCH
3. **`self.model_name`** — กำหนด name ที่ `form_with` ใช้สร้าง field names เช่น
   `registration_form[email]`

ทดสอบ validation โดยตรงใน console:

```ruby
form = RegistrationForm.new(
  email: "user@example.com",
  password: "short",
  password_confirmation: "short",
  display_name: "ผู้ทดสอบ",
  terms_of_service: false
)

form.valid?
# => false

form.errors.full_messages
# => ["Password มีความยาวสั้นเกินไป (ต้องไม่น้อยกว่า 8 ตัวอักษร)",
#     "Terms of service ต้องยอมรับเงื่อนไขการใช้บริการ"]
```

```ruby
form2 = RegistrationForm.new(
  email: "user@example.com",
  password: "securepass123",
  password_confirmation: "securepass123",
  display_name: "ผู้ทดสอบ",
  bio: "นักพัฒนา Ruby",
  terms_of_service: true
)

form2.valid?
# => true

form2.errors.any?
# => false
```

---

## Step 818: ใช้ Form Object ใน Controller และ View

### ใน controller

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  def new
    @form = RegistrationForm.new
  end

  def create
    @form = RegistrationForm.new(registration_params)

    if @form.valid?
      result = UserRegistrationService.call(
        email:                 @form.email,
        password:              @form.password,
        password_confirmation: @form.password_confirmation,
        display_name:          @form.display_name
      )

      if result.success?
        redirect_to root_path, notice: "ยินดีต้อนรับ #{result.value.email}!"
      else
        @form.errors.add(:base, result.error_message)
        render :new, status: :unprocessable_entity
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def registration_params
    params.require(:registration_form).permit(
      :email, :password, :password_confirmation,
      :display_name, :bio, :terms_of_service
    )
  end
end
```

### ใน view — `form_with model: @form`

```erb
<%# app/views/registrations/new.html.erb %>
<h1>สมัครสมาชิก</h1>

<% if @form.errors.any? %>
  <div class="error-box">
    <ul>
      <% @form.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
    </ul>
  </div>
<% end %>

<%= form_with model: @form, url: registrations_path do |f| %>
  <div>
    <%= f.label :email, "อีเมล" %>
    <%= f.email_field :email, value: @form.email %>
    <% if @form.errors[:email].any? %>
      <span class="field-error"><%= @form.errors[:email].first %></span>
    <% end %>
  </div>

  <div>
    <%= f.label :password, "รหัสผ่าน (อย่างน้อย 8 ตัวอักษร)" %>
    <%= f.password_field :password %>
    <% if @form.errors[:password].any? %>
      <span class="field-error"><%= @form.errors[:password].first %></span>
    <% end %>
  </div>

  <div>
    <%= f.label :password_confirmation, "ยืนยันรหัสผ่าน" %>
    <%= f.password_field :password_confirmation %>
    <% if @form.errors[:password_confirmation].any? %>
      <span class="field-error"><%= @form.errors[:password_confirmation].first %></span>
    <% end %>
  </div>

  <div>
    <%= f.label :display_name, "ชื่อที่แสดงในระบบ" %>
    <%= f.text_field :display_name, value: @form.display_name %>
  </div>

  <div>
    <%= f.label :bio, "แนะนำตัวสั้นๆ (ไม่เกิน 500 ตัวอักษร)" %>
    <%= f.text_area :bio, value: @form.bio, rows: 3 %>
  </div>

  <div>
    <%= f.check_box :terms_of_service %>
    <%= f.label :terms_of_service, "ฉันยอมรับเงื่อนไขการใช้บริการ" %>
    <% if @form.errors[:terms_of_service].any? %>
      <span class="field-error"><%= @form.errors[:terms_of_service].first %></span>
    <% end %>
  </div>

  <%= f.submit "สมัครสมาชิก" %>
<% end %>
```

สังเกตประโยชน์ที่ได้จาก `form_with model: @form`:

1. **Field names สอดคล้องกับ Strong Parameters** — Rails สร้าง `registration_form[email]`,
   `registration_form[password]` ให้อัตโนมัติจาก `model_name`
2. **Error display ทำงานเหมือน ActiveRecord model** — `@form.errors[:email].first` คืน error
   message สำหรับ field นั้นๆ เหมือนกับที่ใช้กับ model จริงทุกประการ
3. **ค่าที่กรอกไว้ถูก retain เมื่อ form submit แล้ว fail** — เพราะ `@form = RegistrationForm.new(
   registration_params)` ก่อน render ซ้ำ ทำให้ user ไม่ต้องกรอกใหม่ทั้งหมด

ทดสอบด้วย `curl` เพื่อยืนยัน:

```bash
bin/rails server &

# ทดสอบ submit ที่ valid
curl -s -X POST http://localhost:3000/registrations \
  -d "registration_form[email]=newuser@example.com" \
  -d "registration_form[password]=securepass123" \
  -d "registration_form[password_confirmation]=securepass123" \
  -d "registration_form[display_name]=ผู้ใช้ใหม่" \
  -d "registration_form[terms_of_service]=1" \
  -w "\nHTTP Status: %{http_code}\n"
# HTTP Status: 302  (redirect to root_path — สำเร็จ)

# ทดสอบ submit ที่ invalid (ไม่ยอมรับ terms)
curl -s -X POST http://localhost:3000/registrations \
  -d "registration_form[email]=another@example.com" \
  -d "registration_form[password]=securepass123" \
  -d "registration_form[password_confirmation]=securepass123" \
  -d "registration_form[display_name]=อีกคน" \
  -d "registration_form[terms_of_service]=0" \
  -w "\nHTTP Status: %{http_code}\n"
# HTTP Status: 422  (unprocessable_entity — แสดง error)
```

---

## Step 819: Service Object + Form Object ทำงานร่วมกัน

Step นี้จะรวบภาพให้เห็นว่าทั้งสอง pattern ทำงานร่วมกันอย่างไร และแต่ละตัวรับผิดชอบชั้นไหนของ
ระบบ:

```
HTTP Request
     │
     ▼
Controller
  ├── รับ params
  ├── สร้าง Form Object ด้วย params
  ├── เรียก form.valid? — ถ้า false → render :new พร้อม form.errors
  └── ถ้า valid → เรียก Service Object.call(form attributes)
                        │
                        ├── validate business rules เพิ่มเติม
                        ├── ทำ database operations (transaction)
                        ├── ส่ง email / trigger side effects
                        └── คืน Result object
Controller
  ├── ถ้า result.success? → redirect
  └── ถ้า result.failure? → เพิ่ม error ใน form → render :new
```

**หน้าที่ที่แบ่งชัดเจน:**

| ชั้น | รับผิดชอบ | ไม่รับผิดชอบ |
|---|---|---|
| Controller | รับ request, เลือก path (render/redirect) | business logic, validation |
| Form Object | validate input จาก user, type casting | การบันทึกข้อมูล, email, side effects |
| Service Object | business logic, database operations, side effects | HTTP, view rendering, input format |
| Model | database schema validation, associations | form-level rules, business workflow |

### กรณีที่ต้องการ Form Object อย่างเดียวโดยไม่มี Service Object

บางครั้ง Form Object ทำงานร่วมกับ model โดยตรงโดยไม่ต้องมี Service Object ก็ได้ — เช่นเมื่อ
form รับข้อมูลจากหลาย model แต่ business logic ไม่ซับซ้อนพอที่จะต้องแยก Service Object:

```ruby
# app/forms/profile_update_form.rb — Form Object ที่ save ตรงไปยัง model โดยไม่มี service
class ProfileUpdateForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :display_name, :string
  attribute :bio,          :string
  attribute :avatar_url,   :string

  validates :display_name, presence: true, length: { maximum: 100 }
  validates :bio, length: { maximum: 500 }
  validates :avatar_url, format: { with: URI::DEFAULT_PARSER.make_regexp(%w[http https]),
                                   message: "ต้องเป็น URL ที่ถูกต้อง",
                                   allow_blank: true }

  def initialize(profile, attributes = {})
    @profile = profile
    super(attributes)
  end

  def save
    return false unless valid?

    @profile.update(
      display_name: display_name,
      bio:          bio,
      avatar_url:   avatar_url
    )
  end

  def persisted?
    @profile.persisted?
  end

  def self.model_name
    ActiveModel::Name.new(self, nil, "ProfileUpdateForm")
  end
end
```

ใน controller:

```ruby
# app/controllers/profiles_controller.rb
class ProfilesController < ApplicationController
  def edit
    @form = ProfileUpdateForm.new(current_user.profile,
                                  display_name: current_user.profile.display_name,
                                  bio:          current_user.profile.bio)
  end

  def update
    @form = ProfileUpdateForm.new(current_user.profile, profile_params)

    if @form.save
      redirect_to profile_path, notice: "อัปเดตโปรไฟล์สำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private

  def profile_params
    params.require(:profile_update_form).permit(:display_name, :bio, :avatar_url)
  end
end
```

`ProfileUpdateForm#save` รวม `valid?` และ `update` เข้าด้วยกัน — pattern นี้ทำให้ controller
สั้นมาก (`if @form.save` แทน `if @form.valid? && @profile.update(...)`) และยังมีประโยชน์ครบถ้วน

---

## Step 820: เมื่อไหร่ควรใช้ Service Object / Form Object — กรอบการตัดสินใจ

### กรอบตัดสินใจ Service Object

**ใช้ Service Object เมื่อ:**

- Logic ต้องทำ **มากกว่าหนึ่ง action ที่แยกกัน** (เช่น สร้าง record + ส่ง email + บันทึก log)
- Logic ต้องใช้ซ้ำจาก **มากกว่าหนึ่ง controller หรือ context** (เช่น web controller + API
  controller + background job ต้องการ business rule เดียวกัน)
- **Testing ต้องการความเร็วและความแม่นยำสูง** และ logic ที่ต้องการทดสอบฝังอยู่ใน controller
  หรือ callback ที่เข้าถึงยาก
- Action มีชื่อ business concept ชัดเจนที่ทีมใช้พูดถึงกันเป็นประจำ (เช่น "การสมัครสมาชิก",
  "การยกเลิกคำสั่งซื้อ", "การ approve งาน")

**ไม่ต้องใช้ Service Object เมื่อ:**

- CRUD ธรรมดาที่ controller action ทำ `@user.update(params)` แล้วจบ
- Logic ง่ายที่ validation ของ model จัดการได้หมด
- Action เกิดขึ้นแค่จาก controller เดียวและไม่มีแผนว่าจะ reuse

### กรอบตัดสินใจ Form Object

**ใช้ Form Object เมื่อ:**

- Form **รับข้อมูลที่ต้องไปหลาย model** พร้อมกัน (เช่น User + Profile + Address)
- Form มี **field ที่ไม่มี column จริงในตาราง** (เช่น `password_confirmation`,
  `terms_of_service`, `remember_me`)
- Form มี **validation rule ที่เป็น UI concern** ไม่ใช่ database concern (เช่น "ต้องเลือก
  checkbox อย่างน้อยหนึ่งตัว", "ถ้าเลือก option A ต้องกรอก field B ด้วย")
- ต้องการ **reuse form logic** เช่น registration form และ edit profile form มี validation ที่
  ซ้ำกันบางส่วน

**ไม่ต้องใช้ Form Object เมื่อ:**

- Form เชื่อมกับ **model เดียวตรงๆ** และ model validation ครอบคลุมได้ครบ
- Form ไม่มี field พิเศษที่นอกเหนือจาก model attributes

### Pitfalls — ข้อผิดพลาดที่พบบ่อย

**1. Over-engineering: ใส่ Service Object ทุก action**

```ruby
# ไม่จำเป็น — CRUD ธรรมดาไม่ต้องใช้ Service Object
class UpdateUserNameService < ApplicationService
  def initialize(user:, name:)
    @user = user
    @name = name
  end

  def call
    if @user.update(name: @name)
      Result.new(success?: true, value: @user)
    else
      Result.new(success?: false, errors: @user.errors.full_messages)
    end
  end
end

# แค่นี้ก็พอสำหรับ CRUD ธรรมดา
@user.update(name: params[:name])
```

**2. Service Object ที่รู้เรื่อง HTTP**

```ruby
# ผิด — service ไม่ควรรู้เรื่อง redirect หรือ flash
class UserRegistrationService < ApplicationService
  def call
    # ...
    redirect_to root_path, notice: "สำเร็จ"  # ❌ ไม่ควรอยู่ใน service
  end
end
```

**3. Service Object ที่ทำมากเกินไปในชิ้นเดียว**

```ruby
# ผิด — ควรแยกเป็น service ย่อยหลายชิ้น
class MegaRegistrationService < ApplicationService
  def call
    create_user
    create_profile
    send_welcome_email
    charge_trial_subscription        # ← ควรแยกเป็น PaymentService
    sync_to_crm                      # ← ควรแยกเป็น CrmSyncService
    send_sms_verification            # ← ควรแยกเป็น SmsVerificationService
    create_default_workspace         # ← ควรแยกเป็น WorkspaceSetupService
    notify_admin_of_new_registration # ← ควรแยกเป็น AdminNotificationService
  end
end
```

**4. Form Object ที่ซ้ำซ้อนกับ Model validation**

```ruby
# ไม่ดี — validation เดิมซ้ำกันสองที่ทำให้ maintain ยาก
class UserForm
  include ActiveModel::Model
  validates :email, presence: true,
                    format: { with: URI::MailTo::EMAIL_REGEXP }  # ← ซ้ำกับ User model
  validates :password, presence: true,
                       length: { minimum: 8 }                    # ← ซ้ำกับ User model
end
```

หลักการที่ดีคือ Form Object ควร validate เฉพาะ **form-level concern** (สิ่งที่ model ไม่รู้และ
ไม่ควรรู้) ส่วน model validation ยังคงเป็น "last line of defense" ที่ป้องกัน bad data ไม่ให้
เข้าฐานข้อมูลไม่ว่าจะเข้ามาจาก path ไหนก็ตาม

---

## แบบฝึกหัดปิด Part

ระบบสมัครสมาชิกที่สมบูรณ์: Form Object validate input + Service Object handle business logic

**โจทย์:** ขยาย `RegistrationForm` และ `UserRegistrationService` ให้ครอบคลุม requirement ต่อไปนี้:

1. เพิ่ม field `referral_code` (optional) ใน `RegistrationForm` — ถ้ากรอกมา ต้องมีความยาว
   ระหว่าง 6–10 ตัวอักษรและเป็นตัวอักษร/ตัวเลขเท่านั้น
2. ใน `UserRegistrationService` ถ้ามี `referral_code` ที่ถูกต้อง ให้บันทึกค่านั้นลงคอลัมน์
   `referred_by` ของ `User` (ต้องเพิ่ม migration ก่อน)
3. เขียน RSpec spec สำหรับ `RegistrationForm` ที่ทดสอบ:
   - form valid เมื่อ `referral_code` ว่าง (optional)
   - form valid เมื่อ `referral_code` มีความยาว 6–10 ตัวอักษรตัวเลข
   - form invalid เมื่อ `referral_code` สั้นเกิน 6 หรือยาวเกิน 10
   - form invalid เมื่อ `referral_code` มีอักขระพิเศษ
4. เขียน RSpec spec เพิ่มใน `UserRegistrationServiceSpec` ที่ทดสอบว่า:
   - เมื่อส่ง `referral_code` ที่ถูกต้อง User ที่สร้างขึ้นมีค่า `referred_by` ตรงกัน
   - เมื่อไม่ส่ง `referral_code` User ที่สร้างขึ้นมี `referred_by` เป็น nil

**เฉลยแนวทาง:**

```ruby
# db/migrate/..._add_referred_by_to_users.rb
def change
  add_column :users, :referred_by, :string
end
```

```ruby
# app/forms/registration_form.rb — เพิ่ม field และ validation
attribute :referral_code, :string

validates :referral_code,
          length: { minimum: 6, maximum: 10, allow_blank: true },
          format: { with: /\A[a-zA-Z0-9]+\z/, message: "ต้องเป็นตัวอักษร/ตัวเลขเท่านั้น",
                    allow_blank: true }
```

```ruby
# app/services/user_registration_service.rb — เพิ่ม keyword argument
def initialize(email:, password:, password_confirmation:,
               display_name: nil, referral_code: nil)
  # ...
  @referral_code = referral_code.presence
end

def create_user
  @user = User.create!(
    email:       @email,
    password:    @password,
    referred_by: @referral_code
  )
end
```

---

## สรุปสิ่งที่ได้เรียนรู้

Part นี้ครอบคลุม design pattern สองตัวที่แก้ปัญหาการจัดระเบียบ business logic ใน Rails:

**Service Object:**
- PORO ที่รับผิดชอบ business logic หนึ่งชิ้น — ย้ายออกจาก controller/model ที่โตเกินไป
- `ApplicationService.call(...)` ด้วย anonymous argument forwarding ให้ทุก subclass ได้
  `.call` class method ฟรี
- `Result` object pattern ทำให้ caller รู้ success/failure โดยไม่ต้อง rescue exception
- วางใน `app/services/` — Rails 8 (Zeitwerk) autoload ให้อัตโนมัติ
- ทดสอบได้แยกจาก controller โดยสิ้นเชิง ทำให้ unit test รันเร็วและแม่นยำ

**Form Object:**
- PORO ที่ `include ActiveModel::Model` และ `ActiveModel::Attributes` เพื่อจัดการ form input
- เหมาะกับ form ที่รับข้อมูลจากหลาย model หรือมี field/validation ที่เป็น UI concern
- เข้ากันได้กับ `form_with model: @form`, `@form.errors`, และ Strong Parameters เหมือน
  ActiveRecord model ทุกประการ
- แยก "การ validate input จาก user" ออกจาก "database validation" ของ model

**การทำงานร่วมกัน:**
- Form Object validate input → Service Object execute business logic — แต่ละชั้นรับผิดชอบ
  ชัดเจน ไม่ก้าวก่ายกัน
- Controller เหลือหน้าที่เดียว: "รับ request, ประสาน object, ส่งผลลัพธ์กลับ"
- Model ยังทำหน้าที่ "last line of defense" ของ database validation ตามเดิม

**กรอบการตัดสินใจ:** ไม่ใช่ทุก action ที่ต้องการ Service Object หรือ Form Object — ใช้เมื่อ
business logic ซับซ้อนพอที่จะทำให้ controller/model อ่านยาก, ต้องการ reuse, หรือต้องการ
ทดสอบแบบ isolated

---

**Part ถัดไป — Part 083: Query Object, Decorator Pattern (Draper), Presenter**

Part 082 นี้แก้ปัญหา "ฝั่งการเขียนข้อมูล (write path)" ได้ครบถ้วนแล้ว — Part 083 จะหันมา
แก้ปัญหาอีกสองด้านที่เหลือในแอปที่โตขึ้น:

1. **ฝั่งการอ่านข้อมูล (read path)** — เมื่อ query ซับซ้อนขึ้นเรื่อยๆ (`joins`, `group`,
   `having`, subquery) และต้องใช้ซ้ำหลายจุด — **Query Object** คือคำตอบ
2. **ฝั่งการแสดงผล (view logic)** — เมื่อ view เริ่มมี logic คำนวณ/จัดรูปแบบเยอะขึ้น —
   **Decorator** (gem `draper`) และ **Presenter** (PORO) คือสองทางเลือกที่แตกต่างกันใน
   สถานการณ์ต่างกัน

รวมถึงการเปรียบเทียบ **Decorator vs. Presenter vs. ViewComponent** (จาก Part 055) ให้ครบวงจร
ว่าแต่ละตัวเหมาะกับสถานการณ์ไหน
