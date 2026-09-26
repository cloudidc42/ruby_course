# Part 043: Authorization เชิงลึกด้วย Pundit — Policy และ Scope

> **Step ครอบคลุมใน Part นี้:** Step 421–430
> **ระดับ:** กลาง-สูง (ควรผ่าน Part 041 "Authentication เบื้องต้น" และ Part 042 "Devise" มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, `pundit` ~> 2.5 (ทดสอบจริงกับ pundit 2.5.2)

## สารบัญของ Part นี้

- Step 421: Authentication vs Authorization — ทบทวนความแตกต่างก่อนเริ่ม
- Step 422: ทำไมต้อง Pundit — ปรัชญา "Plain Old Ruby Objects" ไม่มี DSL เวทมนตร์
- Step 423: ติดตั้ง Pundit และทำความรู้จัก `ApplicationPolicy`
- Step 424: เขียน `PostPolicy` ตัวแรก — `initialize`, `update?`/`destroy?`/`edit?`
- Step 425: ใช้ `authorize` ใน Controller และจัดการ `Pundit::NotAuthorizedError`
- Step 426: ใช้ `policy` ใน View — ซ่อน/แสดงปุ่มตามสิทธิ์จริง
- Step 427: `Pundit::Scope` — กรองรายการด้วย `policy_scope` ใน action `index`
- Step 428: Safety net กันลืม authorize — `verify_authorized` และ `verify_policy_scoped`
- Step 429: Role-based condition ใน policy และการจัดระเบียบ policy สำหรับแอปที่กำลังโต
- Step 430: Testing Pundit policy ด้วย RSpec — `permissions` DSL และ matcher `permit`

---

## Step 421: Authentication vs Authorization — ทบทวนความแตกต่างก่อนเริ่ม

ใน Part 041 เราสร้างระบบ login/logout ด้วย `has_secure_password` และ session-based
authentication เอง ส่วนใน Part 042 เราติดตั้ง Devise เพื่อให้ได้ฟีเจอร์ authentication
ระดับ production (confirmable, lockable, remember me ฯลฯ) แบบไม่ต้องเขียนเองทั้งหมด

ทั้งสอง Part นั้นตอบคำถามเดียวกัน คือ **"คุณเป็นใคร?" (Who are you?)** — นี่คือ
**Authentication (การพิสูจน์ตัวตน)**

แต่มีคำถามที่สำคัญไม่แพ้กันซึ่ง authentication ตอบไม่ได้เลย คือ
**"คุณทำสิ่งนี้ได้หรือไม่?" (What can you do?)** — นี่คือ **Authorization (การกำหนดสิทธิ์)**

### ตัวอย่างที่ทำให้เห็นความต่างชัดเจน

สมมติแอปบล็อกของเรามีผู้ใช้ 3 คน: มานี (เจ้าของโพสต์ A), สมชาย (สมาชิกทั่วไป),
และแอดมิน

| สถานการณ์ | Authentication ตอบได้ไหม | Authorization ตอบได้ไหม |
|---|---|---|
| สมชาย login เข้าระบบสำเร็จหรือไม่ | ตอบได้ (ใช่ เพราะ password ถูกต้อง) | ไม่เกี่ยวข้อง |
| สมชาย **ควรแก้ไข** โพสต์ A ของมานีได้หรือไม่ | ตอบไม่ได้ (authentication ไม่รู้เรื่องสิทธิ์) | ตอบได้ (ไม่ได้ เพราะไม่ใช่เจ้าของ) |
| แอดมินลบโพสต์ A ของมานีได้หรือไม่ | ตอบไม่ได้ | ตอบได้ (ได้ เพราะเป็นแอดมิน) |
| ผู้ใช้ที่ยังไม่ login เห็นหน้ารายการโพสต์ได้ไหม | ไม่เกี่ยวข้อง (ไม่มี user เลยด้วยซ้ำ) | ตอบได้ (เห็นเฉพาะโพสต์ที่เผยแพร่แล้ว) |

สังเกตว่า **authentication เกิดขึ้นครั้งเดียวตอน login** (หรือไม่เกิดเลยถ้าเป็น guest)
แต่ **authorization ต้องถูกตรวจสอบซ้ำทุกครั้งที่มีการกระทำ (action)** เพราะสิทธิ์ขึ้นกับ
ทั้ง "ใครทำ" (`user`) และ "ทำกับอะไร" (`record`) พร้อมกันเสมอ

### ทำไมไม่เขียน authorization logic ไว้ใน controller ตรงๆ

หลายคนที่เพิ่งเริ่มมักเขียนแบบนี้:

```ruby
# app/controllers/posts_controller.rb — วิธีที่ไม่แนะนำ
def update
  @post = Post.find(params[:id])

  unless @post.user == current_user || current_user.admin?
    redirect_to root_path, alert: "ไม่มีสิทธิ์"
    return
  end

  @post.update(post_params)
  redirect_to @post
end

def destroy
  @post = Post.find(params[:id])

  unless @post.user == current_user || current_user.admin?
    redirect_to root_path, alert: "ไม่มีสิทธิ์"
    return
  end

  @post.destroy
  redirect_to posts_path
end
```

ปัญหาของวิธีนี้คือ:

1. **โค้ดซ้ำซ้อน (violates DRY)** — เงื่อนไข `@post.user == current_user || current_user.admin?`
   ถูกก็อปวางในทุก action ที่ต้องการ authorization
2. **ลืมง่าย** — ถ้ามี action ใหม่ (เช่น `publish`, `archive`) แล้วลืมเขียนเงื่อนไขนี้ ก็จะเกิด
   security hole ทันทีโดยไม่มีใครรู้จนกว่าจะถูกโจมตี
3. **ทดสอบยาก** — logic การให้สิทธิ์ปนอยู่กับ logic การจัดการ HTTP request ทำให้เขียน unit
   test แยกเฉพาะส่วน authorization ไม่ได้ ต้องยิง request ผ่าน controller ทุกครั้ง
4. **กระจัดกระจาย** — ถ้า business rule เปลี่ยน (เช่น เพิ่มบทบาท "editor" ที่แก้ไขโพสต์คนอื่น
   ได้แต่ลบไม่ได้) ต้องไล่หาทุกจุดในโค้ดเบสที่มีเงื่อนไขคล้ายกันนี้

สิ่งที่เราต้องการคือ **จุดศูนย์กลางเดียว** ที่เก็บกฎว่า "ใครทำอะไรกับอะไรได้บ้าง" แยกออกจาก
controller อย่างชัดเจน ทดสอบได้อิสระ และบังคับให้ทุก action ต้องเรียกใช้ (ไม่ให้ลืม) — นี่คือ
สิ่งที่ **Pundit** ถูกออกแบบมาให้ทำ

---

## Step 422: ทำไมต้อง Pundit — ปรัชญา "Plain Old Ruby Objects" ไม่มี DSL เวทมนตร์

ในวงการ Rails มี authorization gem หลักๆ อยู่ 2 ตัวที่นิยมที่สุดคือ **Pundit** และ
**CanCanCan** (จะเรียนใน Part 044) ทั้งสองแก้ปัญหาเดียวกัน แต่ปรัชญาการออกแบบต่างกันมาก

### ปรัชญาของ Pundit

Pundit ยึดหลักการที่ตรงไปตรงมาที่สุดในบรรดา authorization gem ทั้งหมด:

> **"Authorization policy คือ Ruby class ธรรมดา ที่มี method คืนค่า `true`/`false`"**

ไม่มี DSL พิเศษ ไม่มี syntax ใหม่ต้องท่องจำ ไม่มี configuration file รวมศูนย์ที่นิยามกฎ
ทั้งหมดของทั้งแอป (แบบที่ CanCanCan ทำผ่าน `Ability` class เดียว) — ทุก policy คือ class ที่
เราคุ้นเคยอยู่แล้ว มี `initialize`, มี instance method, ทดสอบด้วย RSpec แบบเดียวกับที่ทดสอบ
class อื่นๆ ทุกประการ (ตามที่เรียนไปแล้วใน Part 019)

ตัวอย่างรูปร่างของ Pundit policy (ยังไม่ต้องรันตอนนี้ แค่ดูโครงสร้าง):

```ruby
class PostPolicy
  def initialize(user, post)
    @user = user
    @post = post
  end

  def update?
    user == post.user || user.admin?
  end

  private

  attr_reader :user, :post
end
```

นี่คือ Ruby class ล้วนๆ ไม่มีอะไรพิเศษเลย — เรียก `PostPolicy.new(current_user, @post).update?`
ที่ไหนก็ได้ในแอป ก็ได้คำตอบ `true`/`false` กลับมา

### เปรียบเทียบปรัชญากับ CanCanCan (ดูรายละเอียดเต็มใน Part 044)

CanCanCan รวมกฎทั้งหมดของทั้งแอปไว้ใน class เดียวชื่อ `Ability`:

```ruby
# แนวทางของ CanCanCan (ตัวอย่างคร่าวๆ เพื่อเปรียบเทียบ — รายละเอียดอยู่ Part 044)
class Ability
  include CanCan::Ability

  def initialize(user)
    can :read, Post, published: true
    can :manage, Post, user_id: user.id if user
    can :manage, :all if user&.admin?
  end
end
```

ข้อดีของ CanCanCan คือเห็นภาพรวมสิทธิ์ทั้งแอปในที่เดียว แต่พอแอปโตขึ้น class `Ability`
นี้จะยาวขึ้นเรื่อยๆ จนกลายเป็น "god object" ที่แก้ยาก และ DSL อย่าง `can`/`cannot` ก็มี
เงื่อนไขพิเศษ (hash conditions, block conditions) ที่ต้องเรียนรู้เพิ่ม

Pundit เลือกทางตรงข้าม: **หนึ่ง policy class ต่อหนึ่ง model** ทำให้ไฟล์เล็ก อ่านง่าย
แก้ policy ของ `Post` โดยไม่กระทบ policy ของ `Comment` เลย และเพราะเป็น Ruby class
ธรรมดา จึงสามารถใช้ inheritance, module, หรือแม้แต่ dependency injection ได้ตามปกติ
โดยไม่ต้องเรียนรู้อะไรใหม่นอกเหนือจาก Ruby ที่เรารู้อยู่แล้ว

### ทำไมปรัชญานี้ถึง "เข้ากับ" Convention over Configuration ของ Rails

Rails เองก็ยึดหลัก "ถ้าตั้งชื่อถูกตามข้อกำหนด ระบบทำงานให้อัตโนมัติ" (ทบทวนได้จาก Part 001)
Pundit ใช้หลักการเดียวกันทุกประการ:

- Model ชื่อ `Post` → Pundit คาดหวัง policy ชื่อ `PostPolicy` โดยอัตโนมัติ
- Controller action ชื่อ `update` → Pundit คาดหวัง policy method ชื่อ `update?` โดยอัตโนมัติ
- ต้องการกรอง collection สำหรับ `Post` → Pundit คาดหวัง class ชื่อ `PostPolicy::Scope`
  ที่มี method `resolve`

เพราะทุกอย่างอิงจาก **ชื่อคลาส** (`record.class.name + "Policy"`) ล้วนๆ ผ่าน method
`Pundit.policy` ที่เดินเรื่อง reflection ธรรมดา ไม่มีความมหัศจรรย์ซ่อนอยู่เบื้องหลังเลย — ถ้า
เข้าใจ Ruby class และ Module ตาม Part 009–010 ก็อ่านซอร์สโค้ดของ Pundit ทั้งหมดได้ในเวลา
ไม่ถึงชั่วโมง (ตัว gem มีโค้ดจริงไม่ถึง 300 บรรทัด)

---

## Step 423: ติดตั้ง Pundit และทำความรู้จัก `ApplicationPolicy`

### เพิ่ม gem ลง Gemfile

```ruby
# Gemfile
gem "pundit", "~> 2.5"
```

```bash
bundle install
```

> **ตรวจสอบเวอร์ชัน:** บทเรียนนี้ทดสอบจริงกับ `pundit (2.5.2)` บน Ruby 3.3.6 / Rails 8.1.4
> รันคำสั่ง `bundle exec gem list pundit` เพื่อดูเวอร์ชันที่ติดตั้งจริงในโปรเจกต์ของคุณ

### รวม `Pundit::Authorization` เข้า `ApplicationController`

Pundit ให้ method อย่าง `authorize`, `policy`, `policy_scope` แก่ controller ผ่าน module
`Pundit::Authorization` ต้อง `include` เองใน `ApplicationController` (ไม่ได้ทำอัตโนมัติ
เพราะ Pundit ไม่อยากยัดเยียดพฤติกรรมใดๆ ให้แอปโดยที่เราไม่รู้ตัว — สอดคล้องกับปรัชญา
"ไม่มีเวทมนตร์" ที่พูดถึงใน Step 422):

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  # current_user ควรมีอยู่แล้วจาก Part 041 (session-based) หรือ Part 042 (Devise
  # จะมี current_user ให้อัตโนมัติอยู่แล้ว)
end
```

### รัน generator `pundit:install`

Pundit มี generator ที่สร้างไฟล์ base policy ให้อัตโนมัติ:

```bash
bin/rails generate pundit:install
```

ผลลัพธ์ที่ได้:

```
create  app/policies/application_policy.rb
```

เปิดไฟล์ที่ได้มาดู (นี่คือโค้ดจริงที่ generator สร้างให้ ไม่มีการแก้ไข):

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?
    false
  end

  def show?
    false
  end

  def create?
    false
  end

  def new?
    create?
  end

  def update?
    false
  end

  def edit?
    update?
  end

  def destroy?
    false
  end

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      raise NoMethodError, "You must define #resolve in #{self.class}"
    end

    private

    attr_reader :user, :scope
  end
end
```

**อธิบายทีละส่วน:**

- `attr_reader :user, :record` — ทุก policy เก็บ "ใคร" (`user`) และ "ทำกับอะไร" (`record`)
  ไว้เป็น instance variable แล้วเปิดให้อ่านผ่าน method (Pundit เรียก positional argument
  ตัวที่สองว่า `record` เสมอ ไม่ว่าจริงๆ แล้วจะเป็น `Post`, `Comment` หรือ model ใดก็ตาม)
- ทุก `*?` method ใน `ApplicationPolicy` **default เป็น `false` ทั้งหมด** (ยกเว้น `new?`
  และ `edit?` ที่ alias ไปยัง `create?`/`update?`) — นี่คือหลักการ **"deny by default"**
  หรือ **fail-safe default**: ถ้า policy ลูก (เช่น `PostPolicy`) ลืม override method ไหน
  ระบบจะ **ปฏิเสธสิทธิ์โดยอัตโนมัติ** แทนที่จะอนุญาตโดยไม่ตั้งใจ — เป็นหลักการความ
  ปลอดภัยที่สำคัญมาก: "พลาดแล้วบล็อกไว้ก่อน" ดีกว่า "พลาดแล้วเปิดช่องโหว่"
- `class Scope` ที่ซ้อนอยู่ข้างในคือ base class สำหรับกรอง collection (จะอธิบายละเอียด
  ใน Step 427) — ถ้าไม่ override `resolve` จะ raise `NoMethodError` ทันที ยึดหลัก
  fail-safe เดียวกัน

จากนี้ไป **ทุก policy ที่เราเขียน จะสืบทอด (`< ApplicationPolicy`) จากไฟล์นี้** เหมือนที่
`ApplicationController` และ `ApplicationRecord` เป็นจุดศูนย์กลางของ controller/model
ทั้งหมด — Pundit ตั้งชื่อไฟล์และ pattern นี้เพื่อให้เข้ากับธรรมเนียมของ Rails โดยตรง

---

## Step 424: เขียน `PostPolicy` ตัวแรก — `initialize`, `update?`/`destroy?`/`edit?`

สมมติเรามีโมเดล `Post` ที่ `belongs_to :user` (เจ้าของโพสต์) และมี column `published`
(boolean, บอกว่าเผยแพร่แล้วหรือยัง) กับโมเดล `User` ที่มี column `role` (enum: `member`
หรือ `admin`):

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :posts, dependent: :destroy

  enum :role, { member: 0, admin: 1 }
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  validates :title, presence: true
end
```

สร้าง policy ด้วย generator ของ Pundit เอง (มี `rspec:policy` generator แถมมาด้วย
จะใช้ตอน Step 430):

```bash
bin/rails generate pundit:policy post
```

```
create  app/policies/post_policy.rb
create  spec/policies/post_policy_spec.rb
```

แก้ไข `app/policies/post_policy.rb` ให้เป็นกฎจริงของแอปเรา:

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  # ทุกคนเห็นหน้ารายการโพสต์ได้ (แต่จะเห็น "รายการไหนบ้าง" ควบคุมด้วย Scope ใน Step 427)
  def index?
    true
  end

  # เห็นโพสต์เดี่ยวได้ ถ้า: เผยแพร่แล้ว หรือเป็นเจ้าของ หรือเป็นแอดมิน
  def show?
    record.published? || owner? || admin?
  end

  # สร้างโพสต์ได้ต้อง login ก่อน (guest สร้างไม่ได้)
  def create?
    user.present?
  end

  # แก้ไขได้เฉพาะเจ้าของหรือแอดมิน
  def update?
    owner? || admin?
  end

  def edit?
    update?
  end

  # ลบได้เฉพาะเจ้าของหรือแอดมิน
  def destroy?
    owner? || admin?
  end

  private

  def owner?
    user.present? && record.user_id == user.id
  end

  def admin?
    user&.admin?
  end
end
```

**อธิบายทีละจุด:**

- `record` ในที่นี้คือ instance ของ `Post` ที่ถูกส่งเข้ามาตอน `PostPolicy.new(user, post)`
  — ใน `ApplicationPolicy` เรามี `attr_reader :user, :record` อยู่แล้ว จึงเรียก `record`
  ตรงๆ ได้เลยโดยไม่ต้องประกาศซ้ำ
- `owner?` และ `admin?` เป็น **private helper method** ภายใน policy เอง — จุดนี้สำคัญมาก:
  Pundit ไม่ได้บังคับให้ policy มีแค่ method ที่ลงท้ายด้วย `?` เท่านั้น เราสามารถเขียน
  helper method อะไรก็ได้เพื่อจัดระเบียบโค้ดให้อ่านง่ายขึ้น (สังเกตว่า `update?` และ
  `destroy?` เรียก `owner? || admin?` เหมือนกัน แทนที่จะเขียนเงื่อนไขซ้ำ)
- `user&.admin?` ใช้ safe navigation operator (เรียนใน Part 002) เพราะ `user` อาจเป็น
  `nil` ได้ (กรณี guest ที่ไม่ได้ login) — ถ้าเขียน `user.admin?` เฉยๆ จะ raise
  `NoMethodError` ทันทีเมื่อไม่มี user login
- `show?` คือตัวอย่างที่ดีของ business rule ที่ผสมเงื่อนไขหลายอย่าง: "เห็นได้ถ้าเผยแพร่แล้ว
  **หรือ** เป็นเจ้าของ **หรือ** เป็นแอดมิน" — logic ระดับนี้ถ้าไปเขียนกระจายอยู่ใน
  controller หรือ view หลายจุด จะดูแลรักษายากกว่าการรวมไว้ที่เดียวมาก

### ทดลองเรียกใช้ policy ตรงๆ ใน console (ยังไม่ผ่าน controller)

เพราะ Pundit policy เป็น Ruby class ธรรมดา เราเรียกใช้ตรงๆ ได้เลยโดยไม่ต้องมี HTTP
request เลยด้วยซ้ำ ลองเปิด `bin/rails console`:

```ruby
alice = User.create!(email: "alice@example.com", password: "password123", role: :member)
bob   = User.create!(email: "bob@example.com",   password: "password123", role: :member)
carol = User.create!(email: "carol@example.com", password: "password123", role: :admin)

draft = Post.create!(title: "ร่างของ Alice", body: "...", user: alice, published: false)

PostPolicy.new(alice, draft).update?  # => true  (เจ้าของ)
PostPolicy.new(bob, draft).update?    # => false (ไม่ใช่เจ้าของ ไม่ใช่แอดมิน)
PostPolicy.new(carol, draft).update?  # => true  (แอดมิน)

PostPolicy.new(bob, draft).show?      # => false (draft ที่ยังไม่เผยแพร่ ไม่ใช่เจ้าของ)
draft.update!(published: true)
PostPolicy.new(bob, draft).show?      # => true  (เผยแพร่แล้ว ใครก็เห็นได้)
```

ผลลัพธ์ข้างต้นคือค่าที่ตรวจสอบจริงจากการรันในสภาพแวดล้อมทดสอบ (scratch Rails app บน
Ruby 3.3.6 / Rails 8.1.4 / pundit 2.5.2) — สังเกตว่าเราทดสอบ authorization logic ทั้งหมด
ได้โดยไม่ต้องยิง HTTP request แม้แต่ครั้งเดียว นี่คือข้อดีสำคัญของการแยก policy ออกมาเป็น
class ต่างหาก

---

## Step 425: ใช้ `authorize` ใน Controller และจัดการ `Pundit::NotAuthorizedError`

การเรียก `PostPolicy.new(current_user, @post).update?` ตรงๆ ทุกครั้งใน controller ก็ยัง
ซ้ำซ้อนอยู่ดี Pundit จึงมี method ช่วย `authorize` ที่ทำ 2 อย่างพร้อมกัน:

1. หา policy class ที่ถูกต้องให้อัตโนมัติ (จากชื่อ class ของ record)
2. เรียก method ที่ตรงกับชื่อ action ปัจจุบันให้อัตโนมัติ (ผ่าน `action_name` ของ
   controller) และถ้าได้ `false` กลับมา จะ **raise `Pundit::NotAuthorizedError` ทันที**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :set_post, only: %i[show edit update destroy]

  def show
    authorize @post
  end

  def new
    @post = Post.new
    authorize @post
  end

  def create
    @post = current_user.posts.build(post_params)
    authorize @post
    if @post.save
      redirect_to @post, notice: "สร้างโพสต์สำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    authorize @post
  end

  def update
    authorize @post
    if @post.update(post_params)
      redirect_to @post, notice: "แก้ไขโพสต์สำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    authorize @post
    @post.destroy
    redirect_to posts_path, notice: "ลบโพสต์แล้ว", status: :see_other
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```

**สิ่งสำคัญที่ต้องเข้าใจเรื่อง `authorize @post`:**

- ใน action `update` คำสั่ง `authorize @post` เทียบเท่ากับการเขียน
  `raise Pundit::NotAuthorizedError unless PostPolicy.new(current_user, @post).update?`
  — Pundit หา `update?` มาจากชื่อ action `update` โดยอัตโนมัติ (mapping ตรงไปตรงมา:
  action `show` → method `show?`, action `edit` → method `edit?` และเช่นกันกับ
  `create?`, `destroy?`)
- `authorize` เรียก `current_user` ให้อัตโนมัติ (Pundit มองหา method ชื่อ `current_user`
  ใน controller โดย default — ถ้า authentication system ของเราใช้ชื่ออื่น เช่น
  `logged_in_account` ต้อง override `pundit_user` ใน `ApplicationController`)
- ถ้า policy method คืนค่า `false` จะ **raise exception ทันที และหยุดการทำงานของ
  action** (โค้ดหลังจาก `authorize` ในบรรทัดนั้นจะไม่ถูกรันเลย) — ต่างจากการเขียน
  `if`/`unless` เองที่ต้อง `return` เองทุกครั้ง

### จัดการ `Pundit::NotAuthorizedError` ด้วย `rescue_from`

ถ้าไม่จัดการ exception นี้เลย ผู้ใช้จะเห็นหน้า error 500 (หรือหน้า debug เต็มรูปแบบใน
development) ซึ่งไม่เป็นมิตรกับผู้ใช้และรั่วไหลข้อมูลภายในระบบ วิธีมาตรฐานคือดักด้วย
`rescue_from` ใน `ApplicationController`:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

  def user_not_authorized
    flash[:alert] = "คุณไม่มีสิทธิ์ทำรายการนี้"
    redirect_back fallback_location: root_path, status: :see_other
  end
end
```

**อธิบาย:**

- `rescue_from` เป็นกลไกมาตรฐานของ Rails controller (ไม่เกี่ยวกับ Pundit โดยตรง)
  สำหรับดัก exception ทุกชนิดที่เกิดขึ้นใน action ไหนก็ได้ของ controller (และ
  controller ลูกทั้งหมด เพราะประกาศไว้ที่ `ApplicationController`)
- `redirect_back fallback_location: root_path` พาผู้ใช้กลับไปหน้าก่อนหน้า (อ่านจาก
  HTTP header `Referer`) ถ้าหาไม่เจอ (เช่น เข้าผ่าน URL ตรงๆ) จะ fallback ไปหน้าแรก
- `status: :see_other` (HTTP 303) เป็นแนวปฏิบัติที่ถูกต้องสำหรับ redirect หลังการ
  กระทำที่ล้มเหลว/สำเร็จแบบไม่ idempotent (Rails 7+ แนะนำใช้ 303 แทน 302 default สำหรับ
  redirect หลัง POST/PATCH/DELETE)

### ทดสอบพฤติกรรมจริง

ทดสอบผ่าน integration test (จำลอง HTTP request จริง) ยืนยันผลลัพธ์ที่ verify มาแล้ว:

```ruby
# บ็อบพยายามแก้ไขโพสต์ของอลิซที่เขาไม่ใช่เจ้าของ
post session_path, params: { email: bob.email, password: "password123" }
get edit_post_path(alice_draft)
# => status: 303 (See Other), redirect ไปที่ "/"

follow_redirect!
# หน้าที่ redirect ไปแสดงข้อความ "คุณไม่มีสิทธิ์ทำรายการนี้" จาก flash[:alert]
```

ผลลัพธ์จริงจากการรันทดสอบ: `status=303`, `location=http://localhost/`, และหน้าที่
redirect ไปมีข้อความ "ไม่มีสิทธิ์" ปรากฏอยู่จริง — ตรงตามที่ตั้งใจออกแบบไว้ทุกประการ

> **ข้อควรระวัง:** `authorize` ต้องถูกเรียก **หลังจาก** `current_user` และ record (เช่น
> `@post`) พร้อมใช้งานแล้วเสมอ ถ้าเรียก `authorize @post` ก่อนที่ `@post` จะถูก set
> (เช่น สลับลำดับ `before_action` ผิด) จะได้ `NoMethodError` เพราะ `@post` เป็น `nil`

---

## Step 426: ใช้ `policy` ใน View — ซ่อน/แสดงปุ่มตามสิทธิ์จริง

การตรวจสอบสิทธิ์ใน controller ป้องกันไม่ให้ action ทำงานโดยไม่ได้รับอนุญาต แต่ถ้า view
ยังคงแสดงลิงก์ "แก้ไข"/"ลบ" ให้ผู้ใช้ที่ไม่มีสิทธิ์เห็น ก็จะสร้างประสบการณ์ที่แย่ (ผู้ใช้
กดลิงก์แล้วโดนเด้งกลับพร้อม flash error ทุกครั้ง) Pundit จึงมี view helper `policy`
ที่ทำงานเหมือนกับ `authorize` ทุกประการ ต่างกันแค่ **ไม่ raise exception** แค่คืนค่า
`true`/`false` ตรงๆ:

```erb
<%# app/views/posts/index.html.erb %>
<h1>โพสต์ทั้งหมด</h1>
<ul>
  <% @posts.each do |post| %>
    <li>
      <%= link_to post.title, post %>
      (<%= post.published? ? "เผยแพร่แล้ว" : "ฉบับร่าง" %>, โดย <%= post.user.email %>)

      <% if policy(post).update? %>
        <%= link_to "แก้ไข", edit_post_path(post) %>
      <% end %>

      <% if policy(post).destroy? %>
        <%= link_to "ลบ", post,
              data: { turbo_method: :delete, turbo_confirm: "แน่ใจหรือไม่?" } %>
      <% end %>
    </li>
  <% end %>
</ul>
```

```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<% if policy(@post).update? %>
  <%= link_to "แก้ไข", edit_post_path(@post) %>
<% end %>
```

**อธิบาย:**

- `policy(post)` คืนค่า instance ของ `PostPolicy.new(current_user, post)` ให้อัตโนมัติ
  (เหมือน `authorize` แต่ไม่ raise error) — เรียกต่อด้วย method ใดก็ได้ที่มีอยู่ใน
  policy เช่น `.update?`, `.destroy?`, `.show?`
- **สำคัญมาก:** `policy` ใน view ใช้เพื่อ **UI ที่ดีขึ้นเท่านั้น** ไม่ใช่ security layer
  ที่แท้จริง — ผู้ใช้ที่รู้ URL ของ `edit_post_path` ยังสามารถพิมพ์ URL เข้าไปตรงๆ ได้
  ถ้า controller ไม่มี `authorize @post` ป้องกันไว้ด้วย ระบบจะยังคงถูกเจาะได้อยู่ดี
  จำหลักนี้ไว้เสมอ: **"authorize ใน controller คือด่านป้องกันจริง, policy ใน view คือ
  แค่การซ่อนปุ่มเพื่อ UX ที่ดี"**
- ถ้าลืม `authorize` ใน controller แต่ใส่ `policy(@post).update?` ใน view ไว้ถูกต้อง
  ผู้ใช้จะไม่เห็นปุ่ม "แก้ไข" ในหน้าเว็บ แต่ยังคงยิง `PATCH /posts/1` ตรงๆ ผ่าน curl
  หรือ Postman ได้สำเร็จอยู่ดี — Step 428 จะแนะนำ safety net ที่ป้องกันความผิดพลาด
  แบบนี้โดยอัตโนมัติ

---

## Step 427: `Pundit::Scope` — กรองรายการด้วย `policy_scope` ใน action `index`

จนถึงตอนนี้เราตรวจสอบสิทธิ์กับ record **ทีละตัว** เท่านั้น (`authorize @post` ตรวจสอบ
โพสต์ตัวเดียว) แต่ action `index` ต้องจัดการกับ **รายการ (collection)** ทั้งหมด — คำถาม
คือ "ผู้ใช้คนนี้ **เห็นโพสต์ไหนได้บ้าง**" ซึ่งเป็นคำถามที่ต่างจาก "ผู้ใช้คนนี้เห็นโพสต์
ตัวนี้ได้ไหม" (นั่นคือ `show?`)

ถ้าเขียนแบบไร้เดียงสา อาจทำแบบนี้:

```ruby
# วิธีที่ไม่แนะนำ — ดึงทุกโพสต์มาก่อน แล้วค่อยกรองด้วย Ruby (ไม่ efficient และ policy
# logic กระจายออกจาก policy class)
def index
  @posts = Post.all.select { |post| PostPolicy.new(current_user, post).show? }
end
```

ปัญหา: ดึงข้อมูล **ทุกแถวจากฐานข้อมูล** มาไว้ใน memory ก่อน แล้วค่อยกรองทีหลังด้วย Ruby
— performance แย่มากถ้ามีข้อมูลเป็นหมื่นเป็นแสนแถว (ต้องโหลดทุก record ขึ้นมาเป็น Ruby
object ทั้งหมดก่อน ทั้งที่สุดท้ายอาจจะเก็บไว้ใช้แค่ไม่กี่สิบตัว)

### วิธีที่ถูกต้อง: `PostPolicy::Scope`

Pundit แก้ปัญหานี้ด้วยการให้ policy แต่ละตัวมี nested class ชื่อ `Scope` ที่รับหน้าที่
สร้าง **ActiveRecord::Relation ที่กรองไว้แล้ว** (ยังไม่ query จริงจนกว่าจะถูกเรียกใช้ —
lazy evaluation ตามปกติของ ActiveRecord ที่เรียนใน Part 034):

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  # ... (index?, show?, create?, update?, destroy? เหมือนเดิมจาก Step 424 ...)

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(user_id: user.id))
      else
        scope.where(published: true)
      end
    end
  end

  # ... (private owner?, admin? เหมือนเดิม) ...
end
```

**อธิบายทีละบรรทัด:**

- `class Scope < ApplicationPolicy::Scope` สืบทอดจาก base `Scope` ใน
  `ApplicationPolicy` ที่ generator สร้างให้ตั้งแต่ Step 423 (มี `attr_reader :user,
  :scope` ให้แล้วผ่าน `initialize(user, scope)`)
- `scope` ในที่นี้คือ argument ตัวที่สองที่ส่งเข้ามา (โดยทั่วไปคือ `Post` class เอง หรือ
  `ActiveRecord::Relation` เช่น `Post.where(...)` ถ้าต้องการกรองซ้อนกัน)
- ตรรกะ 3 กรณี:
  1. ถ้าเป็นแอดมิน → เห็น **ทุกโพสต์** ไม่ว่าจะเผยแพร่แล้วหรือยัง
  2. ถ้า login แล้วแต่ไม่ใช่แอดมิน → เห็นโพสต์ที่เผยแพร่แล้ว **หรือ** โพสต์ของตัวเอง
     (แม้จะยังเป็น draft ก็ตาม) — ใช้ `.or()` ของ ActiveRecord (เรียนใน Part 034)
  3. ถ้าไม่ได้ login (guest, `user` เป็น `nil`) → เห็นเฉพาะโพสต์ที่เผยแพร่แล้วเท่านั้น
- ทั้งหมดนี้ยังคงเป็น **SQL query เดียว** ไม่ใช่การโหลดทุกแถวมากรองด้วย Ruby

### เรียกใช้ผ่าน `policy_scope` ใน Controller

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = policy_scope(Post)
end
```

`policy_scope(Post)` เทียบเท่ากับการเขียน `PostPolicy::Scope.new(current_user,
Post).resolve` — Pundit หา class `PostPolicy::Scope` ให้อัตโนมัติจากชื่อ `Post` (มี
convention เดียวกับ `authorize` ทุกประการ)

### ตรวจสอบผลลัพธ์จริง

ทดสอบด้วยการสร้างข้อมูลจริงใน console: อลิซ (member) มีโพสต์ draft 1 อัน + published
1 อัน, บ็อบ (member) มีโพสต์ draft 1 อัน, แครอล เป็น admin:

```ruby
PostPolicy::Scope.new(alice, Post.all).resolve.pluck(:title)
# => ["Alice draft", "Alice published"]   (เห็นของตัวเองทั้งสองแบบ)

PostPolicy::Scope.new(bob, Post.all).resolve.pluck(:title)
# => ["Alice published", "Bob draft"]     (เห็นของ Alice ที่เผยแพร่แล้ว + ของตัวเอง)

PostPolicy::Scope.new(carol, Post.all).resolve.pluck(:title)
# => ["Alice draft", "Alice published", "Bob draft"]   (แอดมินเห็นหมด)

PostPolicy::Scope.new(nil, Post.all).resolve.pluck(:title)
# => ["Alice published"]                  (guest เห็นแค่ที่เผยแพร่แล้ว)
```

ผลลัพธ์ทั้ง 4 กรณีนี้คือค่าที่ตรวจสอบจริงจากการรันในสภาพแวดล้อมทดสอบ ตรงตามตรรกะที่
ออกแบบไว้ทั้งหมด — สังเกตว่าเราทดสอบ `Scope` แยกจาก `PostPolicy` หลักได้อย่างอิสระ
เพราะมันเป็น class คนละตัวกัน (แม้จะซ้อนอยู่ข้างในก็ตาม)

### View ที่ใช้ผลลัพธ์จาก policy_scope

```erb
<%# app/views/posts/index.html.erb — @posts มาจาก policy_scope(Post) แล้ว %>
<h1>โพสต์ทั้งหมด (ที่คุณมีสิทธิ์เห็น)</h1>
<ul>
  <% @posts.each do |post| %>
    <li><%= link_to post.title, post %></li>
  <% end %>
</ul>
```

ไม่ต้องมี `if` เงื่อนไขใดๆ ใน view เลย เพราะ `@posts` ถูกกรองมาให้ถูกต้องตั้งแต่ต้นทาง
แล้ว — นี่คือประโยชน์ของการแยกกฎ authorization ออกจาก view logic โดยสิ้นเชิง

---

## Step 428: Safety Net กันลืม authorize — `verify_authorized` และ `verify_policy_scoped`

ปัญหาที่ใหญ่ที่สุดของระบบ authorization ทุกแบบคือ **"ลืมเรียก" มากกว่า "เรียกผิด"** —
ถ้าเราลืมใส่ `authorize @post` ใน action ใหม่ที่เพิ่งเขียน (เช่น `def publish` ที่ลืม
authorize ไปเลย) action นั้นจะทำงานสำเร็จเงียบๆ **โดยไม่มีการตรวจสอบสิทธิ์ใดๆ เลย** —
เป็นช่องโหว่ด้านความปลอดภัยที่อันตรายที่สุดแบบหนึ่ง เพราะมันไม่ error ให้เห็นเลยตอน
พัฒนา จะรู้ตัวอีกทีก็ตอนถูกโจมตีจริงแล้ว

Pundit แก้ปัญหานี้ด้วย `after_action` สอง callback ที่ทำหน้าที่เป็น **safety net**
ตรวจสอบว่า *ทุก action* ได้เรียก `authorize`/`policy_scope` แล้วจริงหรือไม่:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  after_action :verify_authorized, except: %i[index]
  after_action :verify_policy_scoped, only: %i[index]

  # ... actions เดิมทั้งหมด ...
end
```

**อธิบาย:**

- `verify_authorized` ตรวจสอบว่า action ที่เพิ่งรันจบไปได้เรียก `authorize` (หรือ
  `skip_authorization`) แล้วหรือยัง ถ้ายัง จะ **raise `Pundit::AuthorizationNotPerformedError`**
- `verify_policy_scoped` ตรวจสอบแบบเดียวกันแต่สำหรับ `policy_scope` (หรือ
  `skip_policy_scope`) ใช้กับ action ที่ทำงานกับ collection เช่น `index`
- ใส่ `except: %i[index]` ใน `verify_authorized` เพราะ action `index` ใช้
  `policy_scope` แทน (ไม่ได้ authorize record เดี่ยว) แล้วสลับไปตรวจด้วย
  `verify_policy_scoped` แทนใน action นั้นแทน

### พิสูจน์ว่า safety net ทำงานจริง — จำลองการ "ลืม" authorize

เพื่อพิสูจน์ว่ากลไกนี้ทำงานจริงตามที่อธิบาย ลองคอมเมนต์บรรทัด `authorize @post` ออกจาก
action `show` ชั่วคราว (สาธิตความผิดพลาดที่เกิดขึ้นได้ในชีวิตจริง):

```ruby
def show
  # authorize @post  # <- ลืมเรียก authorize โดยตั้งใจ เพื่อสาธิต
end
```

ยิง request จริงไปที่ `GET /posts/1`:

```
GET /posts/1 (forgot authorize in #show) status=500
```

และใน log ของ Rails เห็นข้อความ exception ชัดเจน:

```
Pundit::AuthorizationNotPerformedError (PostsController):
```

นี่คือผลลัพธ์ที่ตรวจสอบจริงจากการรันโค้ดนี้ในสภาพแวดล้อมทดสอบ — สังเกตว่าระบบแจ้ง
error **ทันทีตอนพัฒนา/ทดสอบ** แทนที่จะปล่อยให้ action ทำงานสำเร็จแบบไม่มีการป้องกันใดๆ
เลย ซึ่งต่างกันมากในแง่ความปลอดภัย เมื่อใส่ `authorize @post` กลับเข้าไปตามเดิม
action ก็กลับมาทำงานปกติ (status 200)

> **แนวปฏิบัติที่แนะนำ:** ใส่ `after_action :verify_authorized` และ
> `after_action :verify_policy_scoped` ไว้ที่ `ApplicationController` เลย (ไม่ใช่แค่
> `PostsController`) เพื่อบังคับใช้กับทุก controller ในแอปโดยอัตโนมัติ ยกเว้น controller
> ที่ตั้งใจไม่ใช้ authorization เลย (เช่น `SessionsController` สำหรับ login) ให้ปิดเฉพาะ
> จุดด้วย `skip_after_action :verify_authorized` หรือเรียก `skip_authorization` ใน action
> นั้นๆ

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  after_action :verify_authorized, unless: :skip_pundit?
  after_action :verify_policy_scoped, if: :index_action?

  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

  def skip_pundit?
    devise_controller? || index_action?
  end

  def index_action?
    action_name == "index"
  end

  def user_not_authorized
    flash[:alert] = "คุณไม่มีสิทธิ์ทำรายการนี้"
    redirect_back fallback_location: root_path, status: :see_other
  end
end
```

> **หมายเหตุ:** `Pundit::PolicyScopingNotPerformedError` (ที่ `verify_policy_scoped`
> raise เมื่อลืมเรียก `policy_scope`) เป็น subclass ของ
> `Pundit::AuthorizationNotPerformedError` ดังนั้นถ้า `rescue_from` ดัก
> `AuthorizationNotPerformedError` ไว้ (แทนที่จะดักแค่ `NotAuthorizedError`) จะครอบคลุม
> ทั้งสองกรณีลืมพร้อมกัน — แต่โดยทั่วไปแนะนำให้ปล่อยให้ error พวกนี้ crash ตอน
> development/test เพื่อบังคับให้แก้ทันที ไม่ควร `rescue_from` มันแบบเงียบๆ (ต่างจาก
> `Pundit::NotAuthorizedError` ที่ควรจับแล้วแสดงหน้า 403 ที่เป็นมิตรกับผู้ใช้จริง)

---

## Step 429: Role-based Condition ใน Policy และการจัดระเบียบสำหรับแอปที่กำลังโต

### เก็บ business rule ไว้ใน policy เท่านั้น ไม่กระจายไปที่อื่น

หลักการสำคัญที่สุดของ Pundit (และของ authorization ที่ดีทั่วไป) คือ **กฎทุกข้อต้องอยู่
ในที่เดียว** — ห้ามมีเงื่อนไข `user.admin? || record.user == user` ปรากฏซ้ำใน
controller, view, service object, background job หรือที่ไหนก็ตามนอกจาก policy class

ตัวอย่างการเพิ่ม role ใหม่ "editor" ที่แก้ไขโพสต์คนอื่นได้แต่ลบไม่ได้ — เพราะ logic ทั้งหมด
รวมอยู่ใน `PostPolicy` เดียว การเพิ่ม role ใหม่แก้ที่เดียวจบ:

```ruby
class User < ApplicationRecord
  enum :role, { member: 0, editor: 1, admin: 2 }
end
```

```ruby
class PostPolicy < ApplicationPolicy
  def update?
    owner? || admin? || editor?
  end

  def destroy?
    owner? || admin?  # editor ไม่ให้สิทธิ์ลบ
  end

  private

  def owner?
    user.present? && record.user_id == user.id
  end

  def admin?
    user&.admin?
  end

  def editor?
    user&.editor?
  end
end
```

สังเกตว่า **controller และ view ไม่ต้องแก้อะไรเลย** เพราะทั้งคู่เรียก `authorize @post`
และ `policy(@post).update?` แบบเดิม — ผลลัพธ์ของ method เปลี่ยนไปตามกฎใหม่โดยอัตโนมัติ
นี่คือประโยชน์ที่จับต้องได้จริงของการรวมศูนย์กฎไว้ที่เดียว

### Base Policy สำหรับ Logic ที่ใช้ร่วมกันหลาย Policy

เมื่อแอปมีหลาย model ที่ต้องการกฎคล้ายกัน (เช่น "เจ้าของหรือแอดมินเท่านั้นที่แก้ไข/ลบได้")
ควรดึง logic ที่ใช้ร่วมกันไปไว้ที่ `ApplicationPolicy` แทนที่จะก็อปวางในทุก policy:

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  # ... index?/show?/create?/update?/destroy? default false เหมือนเดิม ...

  private

  # helper ที่ policy ลูกทุกตัวเรียกใช้ร่วมกันได้ ถ้า record มี column ชื่อ user_id
  def owner?
    user.present? && record.respond_to?(:user_id) && record.user_id == user.id
  end

  def admin?
    user&.admin?
  end

  class Scope
    # ... เหมือนเดิม ...
  end
end
```

ตอนนี้ `PostPolicy` และ `CommentPolicy` เรียก `owner?`/`admin?` ที่สืบทอดมาได้เลย ไม่ต้อง
ประกาศซ้ำ:

```ruby
class PostPolicy < ApplicationPolicy
  def update?
    owner? || admin?
  end
  # ไม่ต้องประกาศ owner?/admin? อีก เพราะสืบทอดมาจาก ApplicationPolicy แล้ว
end
```

### โครงสร้างไฟล์ policy สำหรับแอปที่กำลังโต

เมื่อจำนวน model เพิ่มขึ้น (Post, Comment, Category, Tag, ...) โฟลเดอร์ `app/policies/`
จะมีไฟล์เพิ่มตามจำนวน model แบบ 1 ต่อ 1 ซึ่งเป็นเรื่องปกติและเป็นข้อดี ไม่ใช่ข้อเสีย:

```
app/policies/
├── application_policy.rb   # base class + shared helper (owner?, admin?)
├── post_policy.rb          # PostPolicy + PostPolicy::Scope
├── comment_policy.rb       # CommentPolicy + CommentPolicy::Scope
└── category_policy.rb      # CategoryPolicy + CategoryPolicy::Scope
```

ถ้ามี logic ที่ใช้ร่วมกันเฉพาะบาง policy (ไม่ใช่ทุก policy) ให้ดึงเป็น Ruby module แล้ว
`include` เข้าไปเฉพาะที่ต้องการ (ใช้หลัก Mixin ที่เรียนใน Part 010):

```ruby
# app/policies/concerns/commentable_policy.rb
module CommentablePolicy
  def can_comment?
    user.present?
  end
end
```

```ruby
class PostPolicy < ApplicationPolicy
  include CommentablePolicy
end
```

รูปแบบนี้คือเหตุผลที่ Pundit "scale ได้ดี" กับแอปขนาดใหญ่: แทนที่จะมี `Ability` class
เดียวที่ยาวขึ้นเรื่อยๆ (ปัญหาของ CanCanCan ตามที่พูดถึงใน Step 422) เรามีไฟล์เล็กๆ จำนวน
มากที่แต่ละไฟล์รับผิดชอบ model เดียว แก้ไขอิสระจากกัน conflict ตอน merge code น้อยกว่ามาก
เมื่อทำงานเป็นทีม

---

## Step 430: Testing Pundit Policy ด้วย RSpec — `permissions` DSL และ Matcher `permit`

ใน Part 019 เราเรียน RSpec เบื้องต้น (`describe`/`context`/`it`, matcher, `let`,
`before`/`after`) ไปแล้ว ตอนนี้เราจะเอาความรู้นั้นมาทดสอบ policy โดยเฉพาะ ซึ่ง Pundit
มี integration กับ RSpec มาให้ในตัว gem เลย (ไม่ต้องติดตั้ง gem เพิ่ม)

### เปิดใช้งาน `pundit/rspec`

```ruby
# spec/rails_helper.rb
require "rspec/rails"
require "pundit/rspec"   # เพิ่มบรรทัดนี้
```

`pundit/rspec` เพิ่ม 2 อย่างให้ RSpec:

1. **DSL `permissions`** — ห่อกลุ่ม example ที่ทดสอบ method เดียวกัน (คล้าย `describe`
   แต่มี syntax เฉพาะสำหรับ policy)
2. **Matcher `permit(user, record)`** — ตรวจสอบว่า policy อนุญาต (`true`) สำหรับ
   `user`/`record` คู่นั้นหรือไม่ ในทุก permission ที่ระบุไว้ใน `permissions` ครอบอยู่

### Generator สร้างโครง policy spec ให้อัตโนมัติ

ตอนรัน `bin/rails generate pundit:policy post` ใน Step 424 นั้น Pundit สร้างไฟล์
`spec/policies/post_policy_spec.rb` ให้พร้อม pending example ไว้เป็นโครงตั้งต้น:

```ruby
require "rails_helper"

RSpec.describe PostPolicy, type: :policy do
  let(:user) { User.new }

  subject { described_class }

  permissions ".scope" do
    pending "add some examples to (or delete) #{__FILE__}"
  end

  permissions :show? do
    pending "add some examples to (or delete) #{__FILE__}"
  end

  # ... create?, update?, destroy? ...
end
```

นี่คือไฟล์จริงที่ generator สร้างให้ (ไม่มีการแก้ไข) — สังเกตว่า `type: :policy` ทำให้
RSpec รู้ว่า spec นี้เป็น policy spec โดยเฉพาะ (ถ้าไฟล์อยู่ใต้ `spec/policies/` Pundit จะ
กำหนด `type: :policy` ให้อัตโนมัติโดยไม่ต้องเขียนก็ได้)

### เขียน spec จริงสำหรับ `PostPolicy`

แทนที่ pending block ด้วย example จริง โดยใช้ FactoryBot (Part 047 จะเจาะลึกเรื่องนี้
อีกครั้ง — ตอนนี้ขอใช้แบบพื้นฐานก่อน):

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
    role { :member }

    trait :admin do
      role { :admin }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    body { "เนื้อหาโพสต์ตัวอย่าง" }
    published { false }
    association :user
  end
end
```

```ruby
# spec/policies/post_policy_spec.rb
require "rails_helper"

RSpec.describe PostPolicy, type: :policy do
  subject { described_class }

  let(:owner)         { create(:user) }
  let(:other_member)  { create(:user) }
  let(:admin)         { create(:user, :admin) }
  let(:guest)         { nil }

  let(:draft)          { create(:post, user: owner, published: false) }
  let(:published_post) { create(:post, user: owner, published: true) }

  permissions :show? do
    it "permits the owner to see their own draft" do
      expect(subject).to permit(owner, draft)
    end

    it "permits an admin to see anyone's draft" do
      expect(subject).to permit(admin, draft)
    end

    it "permits anyone to see a published post" do
      expect(subject).to permit(other_member, published_post)
      expect(subject).to permit(guest, published_post)
    end

    it "does not permit another member to see someone else's draft" do
      expect(subject).not_to permit(other_member, draft)
    end
  end

  permissions :update?, :destroy? do
    it "permits the owner" do
      expect(subject).to permit(owner, draft)
    end

    it "permits an admin" do
      expect(subject).to permit(admin, draft)
    end

    it "does not permit another member" do
      expect(subject).not_to permit(other_member, draft)
    end

    it "does not permit a guest" do
      expect(subject).not_to permit(guest, draft)
    end
  end
end
```

**อธิบาย:**

- `permissions :update?, :destroy?` รับได้หลาย symbol พร้อมกัน — ทุก `it` ใน block นี้
  จะถูกทดสอบกับ **ทั้งสอง method** (คือ `update?` และ `destroy?`) โดยอัตโนมัติ เหมาะ
  มากสำหรับ policy ของเราที่ `update?`/`destroy?` มีตรรกะเดียวกันเป๊ะ (`owner? ||
  admin?`) เขียนครั้งเดียวทดสอบครอบทั้งคู่
- `expect(subject).to permit(owner, draft)` อ่านว่า "คาดหวังว่า `PostPolicy` จะอนุญาต
  `owner` กับ `draft` ใน**ทุก** permission ที่ `permissions` บล็อกครอบไว้" — ถ้า
  permission ไหนคืนค่า `false` matcher จะ fail ทันที
- `subject { described_class }` คือ `PostPolicy` เอง (ไม่ใช่ instance) เพราะ matcher
  `permit` จะสร้าง instance เองภายในด้วย `policy.new(user, record)`

### รันทดสอบจริงและดูผลลัพธ์

```bash
bundle exec rspec spec/policies/post_policy_spec.rb
```

```
...................

Finished in 0.22272 seconds (files took 2.35 seconds to load)
19 examples, 0 failures
```

ผลลัพธ์นี้คือผลการรันจริงจากสภาพแวดล้อมทดสอบ (รวม spec ของ `CommentPolicy` ที่จะเขียน
ในแบบฝึกหัดด้วย) — ทั้ง 19 example ผ่านหมด ยืนยันว่ากฎทุกข้อของ `PostPolicy` ทำงานตรง
ตามที่ออกแบบไว้

### ดูข้อความ error ตอนทดสอบล้มเหลว (เพื่อความเข้าใจ)

ลองเขียน expectation ที่ผิดโดยตั้งใจ เพื่อดูว่า matcher `permit` แสดง error message
อย่างไร:

```ruby
it "demo failing expectation" do
  expect(subject).to permit(other_member, draft)  # ผิด: other_member ไม่ใช่เจ้าของ
end
```

```
Failure/Error: expect(subject).to permit(other_member, draft)
  Expected PostPolicy to grant update? on #<Post:0x00007f5b841473a0> but update? was not granted
```

นี่คือข้อความ error จริงที่ pundit/rspec สร้างให้ — บอกชัดเจนว่า policy ตัวไหน (`PostPolicy`),
permission ตัวไหน (`update?`), และ record ตัวไหนที่ทดสอบไม่ผ่าน ทำให้ debug ได้เร็วเมื่อ
spec แดง

### ทดสอบ `PostPolicy::Scope` แยกต่างหาก

`Scope` เป็น class คนละตัวกับ policy หลัก จึงทดสอบแยกเป็น `describe` block ของตัวเองได้
(ไม่ต้องใช้ `permissions` DSL เพราะไม่ได้ตรวจ `true`/`false` แต่ตรวจว่า relation ที่ได้
มี record ไหนอยู่บ้าง):

```ruby
describe PostPolicy::Scope do
  subject { described_class.new(user, Post).resolve }

  let!(:owner_draft)     { create(:post, user: owner, published: false) }
  let!(:owner_published) { create(:post, user: owner, published: true) }
  let!(:other_draft)     { create(:post, user: other_member, published: false) }

  context "when the user is an admin" do
    let(:user) { admin }

    it "returns every post" do
      expect(subject).to include(owner_draft, owner_published, other_draft)
    end
  end

  context "when the user is a regular member" do
    let(:user) { other_member }

    it "returns published posts plus the user's own posts" do
      expect(subject).to include(owner_published, other_draft)
      expect(subject).not_to include(owner_draft)
    end
  end

  context "when there is no user (guest)" do
    let(:user) { nil }

    it "returns only published posts" do
      expect(subject).to include(owner_published)
      expect(subject).not_to include(owner_draft, other_draft)
    end
  end
end
```

สังเกตว่าใช้ `let!` (มีเครื่องหมาย `!`) แทน `let` เฉยๆ เพื่อบังคับให้สร้าง record ใน
ฐานข้อมูล**ก่อน**ที่ `subject` จะถูกเรียก (ถ้าใช้ `let` เฉยๆ record จะยังไม่ถูกสร้างจนกว่า
จะถูกอ้างถึงครั้งแรก ซึ่งอาจช้ากว่าตอนที่ `subject` รัน query ไปแล้ว) — รายละเอียดของ
`let` vs `let!` อยู่ใน Part 019 และ Part 046

---

## แบบฝึกหัด: สร้างระบบ Authorization เต็มรูปแบบสำหรับ Post + Comment

### โจทย์

ขยายระบบบล็อกที่มีอยู่ (Post + Comment, `belongs_to :user` ทั้งคู่) ให้มี authorization
ครบวงจรตามกฎต่อไปนี้:

**สำหรับ Post:**
1. ทุกคน (รวม guest) เห็นรายการโพสต์ที่เผยแพร่แล้วได้
2. เจ้าของเห็นโพสต์ของตัวเองได้ทุกสถานะ (draft และ published)
3. แอดมินเห็นโพสต์ทุกอันของทุกคน
4. เฉพาะผู้ใช้ที่ login แล้วเท่านั้นที่สร้างโพสต์ได้
5. แก้ไข/ลบโพสต์ได้เฉพาะเจ้าของหรือแอดมิน

**สำหรับ Comment:**
1. ผู้ใช้ที่ login แล้วเท่านั้นที่คอมเมนต์ได้
2. ลบคอมเมนต์ได้ถ้า: เป็นคนเขียนคอมเมนต์เอง, หรือเป็นเจ้าของโพสต์ที่คอมเมนต์นั้นอยู่, หรือ
   เป็นแอดมิน

**เงื่อนไขทางเทคนิค:**
- ต้องมี `after_action :verify_authorized` และ `verify_policy_scoped` ป้องกันการลืม
  authorize ในทุก controller
- ต้องมี RSpec policy spec ครอบคลุมทุกกฎข้างต้น

### เฉลย

**Model (สมมติว่ามีอยู่แล้วจาก Part ก่อนหน้า):**

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy

  enum :role, { member: 0, admin: 1 }
  validates :email, presence: true, uniqueness: true
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  validates :title, presence: true
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
  validates :body, presence: true
end
```

**Base Policy พร้อม shared helper:**

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?  = false
  def show?   = false
  def create? = false
  def new?    = create?
  def update? = false
  def edit?   = update?
  def destroy? = false

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      raise NoMethodError, "You must define #resolve in #{self.class}"
    end

    private

    attr_reader :user, :scope
  end

  private

  def admin?
    user&.admin?
  end
end
```

> **หมายเหตุ:** ใช้ endless method definition (`def index? = false`) ซึ่งเป็น syntax
> ที่รองรับตั้งแต่ Ruby 3.0 (ทบทวนได้จาก Part 002) — ทางเลือกนี้อ่านง่ายสำหรับ method
> ที่มีแค่ expression เดียว แต่ generator ของ Pundit เองยังสร้างแบบ `def ... end` เต็ม
> รูปแบบ (ตามที่เห็นใน Step 423) ทั้งสองแบบใช้แทนกันได้ทุกประการ

**PostPolicy:**

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def index?
    true
  end

  def show?
    record.published? || owner? || admin?
  end

  def create?
    user.present?
  end

  def update?
    owner? || admin?
  end

  def edit?
    update?
  end

  def destroy?
    owner? || admin?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(user_id: user.id))
      else
        scope.where(published: true)
      end
    end
  end

  private

  def owner?
    user.present? && record.user_id == user.id
  end
end
```

**CommentPolicy:**

```ruby
# app/policies/comment_policy.rb
class CommentPolicy < ApplicationPolicy
  def create?
    user.present?
  end

  def destroy?
    user.present? && (record.user_id == user.id || record.post.user_id == user.id || admin?)
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.all
    end
  end
end
```

**PostsController:**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  after_action :verify_authorized, except: %i[index]
  after_action :verify_policy_scoped, only: %i[index]

  before_action :set_post, only: %i[show edit update destroy]

  def index
    @posts = policy_scope(Post)
  end

  def show
    authorize @post
  end

  def new
    @post = Post.new
    authorize @post
  end

  def create
    @post = current_user.posts.build(post_params)
    authorize @post
    if @post.save
      redirect_to @post, notice: "สร้างโพสต์สำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    authorize @post
  end

  def update
    authorize @post
    if @post.update(post_params)
      redirect_to @post, notice: "แก้ไขโพสต์สำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    authorize @post
    @post.destroy
    redirect_to posts_path, notice: "ลบโพสต์แล้ว", status: :see_other
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```

**CommentsController:**

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  after_action :verify_authorized

  def create
    @post = Post.find(params[:post_id])
    @comment = @post.comments.build(comment_params)
    @comment.user = current_user
    authorize @comment
    @comment.save
    redirect_to @post
  end

  def destroy
    @comment = Comment.find(params[:id])
    authorize @comment
    @comment.destroy
    redirect_to @comment.post, status: :see_other
  end

  private

  def comment_params
    params.require(:comment).permit(:body)
  end
end
```

**Routes:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts do
    resources :comments, only: %i[create destroy]
  end

  root "posts#index"
end
```

**Factories:**

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
    role { :member }

    trait :admin do
      role { :admin }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    body { "เนื้อหาโพสต์ตัวอย่าง" }
    published { false }
    association :user
  end
end
```

```ruby
# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    body { "ความคิดเห็นตัวอย่าง" }
    association :post
    association :user
  end
end
```

**RSpec policy spec — PostPolicy:**

```ruby
# spec/policies/post_policy_spec.rb
require "rails_helper"

RSpec.describe PostPolicy, type: :policy do
  subject { described_class }

  let(:owner)        { create(:user) }
  let(:other_member) { create(:user) }
  let(:admin)        { create(:user, :admin) }
  let(:guest)        { nil }

  let(:draft)          { create(:post, user: owner, published: false) }
  let(:published_post) { create(:post, user: owner, published: true) }

  permissions :show? do
    it "permits the owner to see their own draft" do
      expect(subject).to permit(owner, draft)
    end

    it "permits an admin to see anyone's draft" do
      expect(subject).to permit(admin, draft)
    end

    it "permits anyone to see a published post" do
      expect(subject).to permit(other_member, published_post)
      expect(subject).to permit(guest, published_post)
    end

    it "does not permit another member to see someone else's draft" do
      expect(subject).not_to permit(other_member, draft)
    end
  end

  permissions :update?, :destroy? do
    it "permits the owner" do
      expect(subject).to permit(owner, draft)
    end

    it "permits an admin" do
      expect(subject).to permit(admin, draft)
    end

    it "does not permit another member" do
      expect(subject).not_to permit(other_member, draft)
    end

    it "does not permit a guest" do
      expect(subject).not_to permit(guest, draft)
    end
  end

  permissions :create? do
    it "permits any signed-in user" do
      expect(subject).to permit(owner, Post.new)
    end

    it "does not permit a guest" do
      expect(subject).not_to permit(guest, Post.new)
    end
  end

  describe PostPolicy::Scope do
    subject { described_class.new(user, Post).resolve }

    let!(:owner_draft)     { create(:post, user: owner, published: false) }
    let!(:owner_published) { create(:post, user: owner, published: true) }
    let!(:other_draft)     { create(:post, user: other_member, published: false) }

    context "when the user is an admin" do
      let(:user) { admin }

      it "returns every post" do
        expect(subject).to include(owner_draft, owner_published, other_draft)
      end
    end

    context "when the user is a regular member" do
      let(:user) { other_member }

      it "returns published posts plus the user's own posts" do
        expect(subject).to include(owner_published, other_draft)
        expect(subject).not_to include(owner_draft)
      end
    end

    context "when there is no user (guest)" do
      let(:user) { nil }

      it "returns only published posts" do
        expect(subject).to include(owner_published)
        expect(subject).not_to include(owner_draft, other_draft)
      end
    end
  end
end
```

**RSpec policy spec — CommentPolicy:**

```ruby
# spec/policies/comment_policy_spec.rb
require "rails_helper"

RSpec.describe CommentPolicy, type: :policy do
  subject { described_class }

  let(:post_owner)     { create(:user) }
  let(:comment_author) { create(:user) }
  let(:other_member)   { create(:user) }
  let(:admin)          { create(:user, :admin) }

  let(:post_record) { create(:post, user: post_owner) }
  let(:comment)     { create(:comment, post: post_record, user: comment_author) }

  permissions :create? do
    it "permits any signed-in user" do
      expect(subject).to permit(other_member, Comment.new)
    end

    it "does not permit a guest" do
      expect(subject).not_to permit(nil, Comment.new)
    end
  end

  permissions :destroy? do
    it "permits the comment's author" do
      expect(subject).to permit(comment_author, comment)
    end

    it "permits the owner of the post the comment belongs to" do
      expect(subject).to permit(post_owner, comment)
    end

    it "permits an admin" do
      expect(subject).to permit(admin, comment)
    end

    it "does not permit an unrelated member" do
      expect(subject).not_to permit(other_member, comment)
    end
  end
end
```

รันทดสอบทั้งหมด:

```bash
bundle exec rspec spec/policies
```

```
...................

Finished in 0.22272 seconds (files took 2.35 seconds to load)
19 examples, 0 failures
```

ผลลัพธ์ทั้ง 19 example (11 จาก `PostPolicy` + 8 จาก `CommentPolicy`) ผ่านหมด ยืนยันว่า
กฎ authorization ทั้งระบบทำงานถูกต้องตามที่โจทย์กำหนด — ทดสอบจริงในสภาพแวดล้อม Ruby
3.3.6 / Rails 8.1.4 / pundit 2.5.2

**สิ่งที่ทำให้เฉลยนี้สมบูรณ์:**

- ทุก action ที่แก้ไขข้อมูลมี `authorize` ครบ และมี `verify_authorized`/
  `verify_policy_scoped` เป็น safety net คอยตรวจจับความผิดพลาดตอนพัฒนา
- Business rule (ใครทำอะไรได้บ้าง) อยู่ใน policy class เท่านั้น ไม่มีการเช็ค
  `current_user.admin?` หลุดเข้าไปใน controller หรือ view เลยแม้แต่จุดเดียว
- `PostPolicy::Scope` แยก query logic ของ "เห็นได้ไหม" ออกจาก policy หลักอย่างชัดเจน
  ทำงานเป็น SQL query เดียว ไม่ loop กรองด้วย Ruby
- RSpec spec ทดสอบทุกกฎแยกจาก HTTP layer โดยสิ้นเชิง รันเร็ว (ไม่ต้องผ่าน routing/
  controller เลย) และอ่านง่ายด้วย `permissions` DSL

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม role `moderator` ที่แก้ไข/ลบคอมเมนต์ของใครก็ได้ (เหมือนแอดมิน) แต่**ไม่มีสิทธิ์**
   แก้ไข/ลบโพสต์ของคนอื่น (ต่างจากแอดมินที่ทำได้ทุกอย่าง) — ต้องแก้เฉพาะ policy เท่านั้น
   โดยไม่แตะ controller หรือ view เลยแม้แต่บรรทัดเดียว แล้วเขียน RSpec spec พิสูจน์ว่า
   `moderator` ผ่าน `CommentPolicy#destroy?` แต่ไม่ผ่าน `PostPolicy#update?`

2. เพิ่ม action `PATCH /posts/:id/publish` ที่เปลี่ยนสถานะ `published` จาก `false` เป็น
   `true` (เฉพาะเจ้าของหรือแอดมิน) โดยต้องมี `publish?` เป็น method ใหม่ใน `PostPolicy`
   (ไม่ใช้ `update?` ซ้ำ) พร้อม route `member do patch :publish end` และเขียน controller
   action `publish` ที่ไม่ลืมเรียก `authorize @post` — ลองคอมเมนต์ `authorize` ออกชั่วคราว
   เพื่อยืนยันว่า `verify_authorized` จับความผิดพลาดได้จริงตามที่เรียนใน Step 428

3. เขียน `CategoryPolicy` ใหม่ทั้งหมดสำหรับโมเดล `Category` ที่ `has_many :posts` โดยกำหนด
   กฎว่า: ทุกคนดู `Category` ได้ (`index?`/`show?` เป็น `true` เสมอ) แต่สร้าง/แก้ไข/ลบได้
   เฉพาะแอดมินเท่านั้น จากนั้นสังเกตว่าโครงสร้าง policy นี้ **ไม่มีแนวคิด `owner?` เลย**
   (เพราะ `Category` ไม่มี `user_id`) — นี่คือตัวอย่างที่แสดงให้เห็นว่า policy แต่ละตัว
   อิสระจากกันโดยสมบูรณ์ และไม่จำเป็นต้องมีโครงสร้างเดียวกันทุกตัว

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจความแตกต่างระหว่าง **Authentication** ("คุณเป็นใคร") ที่เรียนไปใน Part 041–042
  กับ **Authorization** ("คุณทำสิ่งนี้ได้ไหม") ที่เป็นหัวข้อของ Part นี้ และทำไมการเขียน
  เงื่อนไขสิทธิ์กระจัดกระจายอยู่ใน controller โดยตรงถึงเป็นแนวทางที่มีปัญหา
- เข้าใจปรัชญาของ Pundit ว่าเป็น "Plain Old Ruby Objects" ไม่มี DSL พิเศษ ต่างจาก
  CanCanCan ที่รวมกฎทั้งหมดไว้ใน `Ability` class เดียว
- ติดตั้ง Pundit จริง (`include Pundit::Authorization`, `rails generate pundit:install`)
  และเข้าใจโครงสร้างของ `ApplicationPolicy` ที่ generator สร้างให้ รวมถึงหลักการ
  "deny by default" ที่ทุก method คืนค่า `false` เป็นค่าเริ่มต้น
- เขียน `PostPolicy` จริงพร้อม `initialize(user, post)`, `update?`/`destroy?`/`edit?`,
  และ private helper method (`owner?`, `admin?`) เพื่อจัดระเบียบเงื่อนไข
- ใช้ `authorize @post` ใน controller และจัดการ `Pundit::NotAuthorizedError` ด้วย
  `rescue_from` ให้แสดงหน้าที่เป็นมิตรกับผู้ใช้แทนหน้า error 500
- ใช้ `policy(@post).update?` ใน view เพื่อซ่อน/แสดงปุ่มตามสิทธิ์จริง พร้อมเข้าใจว่านี่คือ
  แค่ UX ที่ดีขึ้น ไม่ใช่ security layer ที่แท้จริง (ต้องมี `authorize` ใน controller เสมอ)
- ใช้ `Pundit::Scope` (`PostPolicy::Scope`) และ `policy_scope(Post)` เพื่อกรอง collection
  ทั้งชุดด้วย SQL query เดียว แทนที่จะโหลดทุก record มากรองด้วย Ruby
- ใช้ `verify_authorized` และ `verify_policy_scoped` เป็น safety net จับการ "ลืม"
  authorize อัตโนมัติ และพิสูจน์แล้วว่ามันจับ `Pundit::AuthorizationNotPerformedError`
  ได้จริงเมื่อมี action ที่ลืมเรียก `authorize`
- เก็บ role-based condition (`user.admin?`, `record.user == user`) ไว้ใน policy เท่านั้น
  และรู้วิธีจัดระเบียบ policy สำหรับแอปที่กำลังโต ด้วย shared helper ใน `ApplicationPolicy`
  และ module แบบ Mixin สำหรับ logic ที่ใช้ร่วมกันเฉพาะบาง policy
- ทดสอบ policy แยกจาก HTTP layer โดยสิ้นเชิงด้วย RSpec ผ่าน `pundit/rspec`, DSL
  `permissions`, และ matcher `permit` — เขียน spec ครอบคลุมทั้ง policy หลักและ
  `Scope` แยกกัน ต่อยอดจากพื้นฐาน RSpec ที่เรียนไปใน Part 019

**ต่อไป (Part 044):** เราจะดู **authorization ทางเลือกอื่น** นอกจาก Pundit คือ
**CanCanCan** ที่ใช้ปรัชญาต่างออกไป (รวมกฎไว้ใน `Ability` class เดียวผ่าน DSL `can`/
`cannot`) พร้อมเปรียบเทียบข้อดีข้อเสียของทั้งสองแนวทางอย่างละเอียด และเจาะลึกการออกแบบ
**Role-Based Access Control (RBAC)** ในระดับที่ซับซ้อนขึ้น เช่น การมีหลาย role ต่อผู้ใช้
หนึ่งคน และการจัดการสิทธิ์แบบ hierarchy (บทบาทที่สืบทอดสิทธิ์จากบทบาทอื่น)
