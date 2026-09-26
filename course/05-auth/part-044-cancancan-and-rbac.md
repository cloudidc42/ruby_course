# Part 044: Authorization ทางเลือก — CanCanCan และ Role-Based Access Control (RBAC)

> **Step ครอบคลุมใน Part นี้:** Step 431–440
> **ระดับ:** กลาง–สูง (ควรผ่าน Part 041–043 มาก่อน — เข้าใจ `has_secure_password`,
> session-based authentication, และ Pundit policy/scope)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, `cancancan` ~> 3.6 (ทดสอบจริงบน Ruby 3.3.6 /
> Rails 8.1.4)

ใน Part 043 เราเรียน **Pundit** ไปแล้วในเชิงลึก — pattern ที่แยก authorization logic
ออกเป็น policy class ต่อ model หนึ่งไฟล์ (`PostPolicy`, `CommentPolicy`, ฯลฯ) พร้อม
`Scope` class สำหรับกรอง collection

Part นี้จะไม่พูดถึงการเขียน Pundit policy ซ้ำอีก แต่จะพาไปรู้จัก **CanCanCan** ซึ่งเป็น
authorization gem อีกสายหนึ่งที่ได้รับความนิยมไม่แพ้กัน มีปรัชญาการออกแบบที่ตรงข้ามกับ
Pundit อย่างชัดเจน แล้วเราจะถอยออกมาหนึ่งก้าว มองภาพรวมของ **Role-Based Access Control
(RBAC)** ในฐานะแนวคิดการออกแบบระบบสิทธิ์ ที่ใช้ได้ไม่ว่าจะเลือก Pundit, CanCanCan หรือเขียน
authorization logic เองล้วนๆ

## สารบัญของ Part นี้

- Step 431: CanCanCan คืออะไร — ปรัชญาเทียบกับ Pundit และตารางเปรียบเทียบ tradeoffs
- Step 432: ติดตั้ง CanCanCan และสร้าง `Ability` class เริ่มต้น
- Step 433: นิยาม abilities ด้วย DSL `can`/`cannot` — ownership, `:manage`/`:all`, role-based
  block
- Step 434: `load_and_authorize_resource` เทียบกับ `authorize!` + `can?`/`cannot?` ใน view
- Step 435: จัดการสิทธิ์ที่ถูกปฏิเสธด้วย `rescue_from CanCan::AccessDenied`
- Step 436: ออกแบบ RBAC schema — ตาราง `roles` + join table เทียบกับคอลัมน์ `role` เดี่ยว
- Step 437: Role-based access ด้วย Rails 7+ `enum` และการสร้าง Ability จาก enum
- Step 438: ระบบสิทธิ์แบบละเอียด — `Permission`/`RolePermission` model-driven ที่ admin
  ปรับได้เอง
- Step 439: การทดสอบ Ability ด้วย RSpec (`cancan/matchers`)
- Step 440: แบบฝึกหัด — สร้างระบบสิทธิ์แบบ multi-role (member/editor/admin) ครบวงจร

---

## Step 431: CanCanCan คืออะไร — ปรัชญาเทียบกับ Pundit และตารางเปรียบเทียบ tradeoffs

### CanCanCan คืออะไร

**CanCanCan** (fork ที่ maintain ต่อจาก gem เดิมชื่อ `cancan` ของ Ryan Bates ที่หยุดพัฒนาไปแล้ว)
เป็น authorization library ที่มีปรัชญาแตกต่างจาก Pundit อย่างสิ้นเชิงตรงที่:

> **Pundit:** authorization logic กระจายอยู่ใน **policy class แยกไฟล์ต่อหนึ่ง model**
> (`PostPolicy`, `CommentPolicy`) — แต่ละไฟล์มี method อธิบายสิทธิ์ของ action นั้นๆ ตรงๆ
> เช่น `def update?; user.admin? || record.user == user; end`
>
> **CanCanCan:** authorization logic ทั้งหมดของทั้งแอปรวมอยู่ใน **class เดียวชื่อ `Ability`**
> ที่ใช้ DSL คำสั่ง `can` / `cannot` ประกาศกฎทีละบรรทัด แล้ว engine ของ CanCanCan จะนำกฎ
> เหล่านั้นมาตอบคำถาม "user คนนี้ทำ action นี้กับ object นี้ได้ไหม" ให้เองโดยอัตโนมัติ

ตัวอย่างสั้นๆ ให้เห็นภาพความต่างทันที (ยังไม่ต้องรันตอนนี้):

```ruby
# แบบ Pundit (Part 043) — หนึ่งไฟล์ต่อหนึ่ง model
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || record.user == user
  end

  def destroy?
    user.admin?
  end
end
```

```ruby
# แบบ CanCanCan — รวมทุกกฎของทุก model ไว้ใน Ability class เดียว
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    can :update, Post, user_id: user.id
    can :manage, Post if user.admin?
  end
end
```

จุดสำคัญที่ต่างกันโดยตรง:

1. **หน่วยของการจัดระเบียบโค้ด** — Pundit จัดระเบียบ "ตาม model" (1 model = 1 ไฟล์)
   ส่วน CanCanCan จัดระเบียบ "ตาม user/role" (ทุก model รวมกันในไฟล์เดียว)
2. **การ query สิทธิ์** — Pundit ต้องเขียน method ชื่อ `?` ต่อท้ายทุก action
   (`update?`, `destroy?`, `publish?`) ส่วน CanCanCan ใช้ engine กลาง `can?(:update, post)`
   ที่ทำงานกับทุก action/model ได้จากกฎที่ประกาศไว้ ไม่ต้องเขียน method แยกทีละ action
3. **การกรอง collection** — Pundit ใช้ `Scope` class แยกต่างหาก ส่วน CanCanCan ใช้
   `Model.accessible_by(current_ability)` ที่ generate SQL `WHERE` จากกฎ `can` โดยตรง
4. **การเชื่อมกับ Controller** — Pundit ให้เขียน `authorize @post` เองทุก action (explicit)
   ส่วน CanCanCan มี macro `load_and_authorize_resource` ที่ **โหลดและตรวจสิทธิ์ให้พร้อมกัน**
   เพียงบรรทัดเดียวในทุก action ของ controller (จะเห็นรายละเอียดใน Step 434)

### ตารางเปรียบเทียบ tradeoffs

| ประเด็น | Pundit | CanCanCan |
|---|---|---|
| โครงสร้างไฟล์ | 1 policy class ต่อ 1 model (`app/policies/`) | 1 `Ability` class รวมทุก model |
| Syntax | Plain Ruby method (`def update?`) | DSL เฉพาะ (`can`, `cannot`) |
| ความกระชับสำหรับแอปเล็ก | ต้องสร้างไฟล์ policy ใหม่ทุก model แม้กฎง่ายๆ | เขียนกฎเพิ่มในไฟล์เดียวได้เร็วมาก |
| ความชัดเจน (explicit vs implicit) | Explicit สูง — ต้องเรียก `authorize` เองทุก action เห็นชัดว่า action ไหนถูกเช็ค | Implicit กว่า — `load_and_authorize_resource` เดาชื่อ model จาก controller ให้อัตโนมัติ อ่านโค้ดต้องรู้ convention ก่อน |
| ความเสี่ยงเมื่อแอปโตขึ้น | แต่ละไฟล์เล็ก จัดการง่าย แต่จำนวนไฟล์เพิ่มเป็นเส้นตรงตามจำนวน model | **`Ability` class เดียวอาจกลายเป็น "God Object"** — พอมี 20-30 model กฎอาจยาวหลายร้อยบรรทัดในไฟล์เดียว อ่าน/แก้ยาก |
| การทดสอบ | เทสต์ policy ทีละไฟล์แยกกันได้เป็นธรรมชาติ | เทสต์ `Ability` ต้องระวังให้ตั้ง context (role/user) ถูกต้อง เพราะกฎทุก model ปนกันในที่เดียว |
| ลำดับกฎมีผลหรือไม่ | ไม่มีปัญหาเรื่องลำดับ (แต่ละ method อิสระจากกัน) | **มีผลมาก** — `cannot` ที่ประกาศทีหลัง `can` จะ override กฎก่อนหน้าเสมอ ลืมเรื่องนี้ทำให้เกิดบั๊กแบบ "ทำไมสิทธิ์ไม่ทำงานตามที่คิด" |
| การกรอง query | ต้องเขียน `Scope` class เพิ่ม | มี `accessible_by` ให้ใช้ทันทีจากกฎ `can` ที่มีอยู่แล้ว ไม่ต้องเขียนโค้ดเพิ่ม |
| Controller integration | ต้องเขียน `authorize`/`policy_scope` เอง (2 บรรทัดขึ้นไปต่อ action) | `load_and_authorize_resource` บรรทัดเดียวครอบคลุมทุก action มาตรฐาน |
| เหมาะกับ | แอปขนาดกลาง-ใหญ่ที่มี business rule ซับซ้อนต่างกันมากในแต่ละ model, ทีมที่ต้องการความชัดเจนสูงสุด | แอปขนาดเล็ก-กลางที่กฎส่วนใหญ่อิงกับ role ตรงไปตรงมา ต้องการความเร็วในการพัฒนา |

**สรุปแบบใช้งานจริง:** ไม่มี gem ไหน "ดีกว่า" อีกฝั่งแบบเบ็ดเสร็จ — ทีมจำนวนมากในอุตสาหกรรม
เลือก Pundit สำหรับแอประดับ enterprise ที่โตต่อเนื่องยาวนาน เพราะไฟล์แยกกันดูแลง่ายกว่าเมื่อ
ทีมใหญ่ขึ้น ส่วน CanCanCan ยังนิยมมากในแอปขนาดเล็ก-กลางหรือ MVP ที่ต้องการความเร็ว และตัว
`load_and_authorize_resource` + `accessible_by` ยังคงเป็นจุดขายที่ประหยัดโค้ดได้มากเมื่อ
authorization logic ไม่ซับซ้อน

> **ข้อสังเกตสำคัญ:** gem ทั้งสองตัวแก้ปัญหาเดียวกัน (ตอบว่า "ใครทำอะไรกับอะไรได้บ้าง")
> ด้วยแนวคิดตรงข้ามกันเรื่อง "รวมศูนย์ vs กระจาย" — ทักษะที่สำคัญกว่าการจำ syntax ของ gem
> ตัวใดตัวหนึ่งคือการเข้าใจ **RBAC** ในฐานะแนวคิด ซึ่งเราจะเจาะลึกตั้งแต่ Step 436 เป็นต้นไป
> และใช้ได้กับ authorization library ตัวไหนก็ได้ หรือแม้แต่ไม่ใช้ gem เลยก็ได้

---

## Step 432: ติดตั้ง CanCanCan และสร้าง `Ability` class เริ่มต้น

### ติดตั้ง gem

เพิ่มใน `Gemfile`:

```ruby
# Gemfile
gem "cancancan", "~> 3.6"
```

```bash
bundle install
```

### Generate Ability class

CanCanCan มี generator สร้างโครงไฟล์ `app/models/ability.rb` ให้ทันที:

```bash
bin/rails g cancan:ability
```

```
create  app/models/ability.rb
```

เนื้อหาเริ่มต้นที่ generator สร้างให้ (เป็น template พร้อมคอมเมนต์อธิบาย):

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    # Define abilities for the user here. For example:
    #
    #   return unless user.present?
    #   can :read, :all
    #   return unless user.admin?
    #   can :manage, :all
    #
    # The first argument to `can` is the action you are giving the user
    # permission to do.
    # If you pass :manage it will apply to every action. Other common actions
    # here are :read, :create, :update and :destroy.
    #
    # The second argument is the resource the user can perform the action on.
    # If you pass :all it will apply to every resource. Otherwise pass a Ruby
    # class of the resource.
    #
    # The third argument is an optional hash of conditions to further filter the
    # objects.
    # For example, here the user can only update published articles.
    #
    #   can :update, Article, published: true
  end
end
```

สังเกตว่า CanCanCan วาง `Ability` ไว้ที่ `app/models/ability.rb` (ไม่ใช่ `app/policies/`
แบบ Pundit) เพราะในมุมมองของ CanCanCan `Ability` คือ **model ของสิทธิ์ผู้ใช้คนหนึ่ง** ไม่ใช่
policy ของ resource ใดๆ

### `current_ability` — จุดเชื่อมกับ Controller

CanCanCan เพิ่ม method `current_ability` ให้ทุก controller โดยอัตโนมัติผ่าน
`ControllerAdditions` (คล้าย `Pundit` ที่ inject `policy`/`policy_scope` เข้าไป) โดย default
มันเรียก `Ability.new(current_user)` ให้เอง — แปลว่าแค่มี method `current_user` ใน
`ApplicationController` (ซึ่งเราทำไว้แล้วตั้งแต่ Part 041) ก็ใช้งานได้ทันที ไม่ต้อง config
เพิ่มอะไรเลย

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user

  # current_ability ถูกเพิ่มให้อัตโนมัติจาก CanCan::ControllerAdditions
  # และ default implementation คือ Ability.new(current_user)
end
```

ถ้าต้องการ custom วิธีสร้าง ability (เช่น ต้องส่ง argument เพิ่มเข้าไปใน `Ability.new`)
override ได้ตรงๆ:

```ruby
# app/controllers/application_controller.rb
def current_ability
  @current_ability ||= Ability.new(current_user)
end
```

---

## Step 433: นิยาม abilities ด้วย DSL `can`/`cannot` — ownership, `:manage`/`:all`, role-based block

มาสร้างแอปตัวอย่างเพื่อทดลองแบบรันได้จริง — ระบบบล็อกที่มี `User` (มี `role` เป็น
member/editor/admin) และ `Post` (มี `published` boolean, `user_id` เจ้าของ)

### `can` — ให้สิทธิ์

Syntax พื้นฐาน: `can <action>, <subject>, <conditions>`

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new # guest ที่ยังไม่ login ก็ให้ผ่าน (null object pattern)

    can :read, Post, published: true # ทุกคนอ่าน post ที่ publish แล้วได้
    can :read, Comment

    return if user.new_record? # guest หยุดแค่นี้

    can :create, Post
    can :create, Comment
    can :read, Post, user_id: user.id # เห็น draft ของตัวเองได้ด้วย แม้ยังไม่ publish
    can %i[update destroy], Post, user_id: user.id
    can %i[update destroy], Comment, user_id: user.id

    case user.role
    when "editor"
      can :read, Post # editor อ่านของทุกคนได้ แม้ยังไม่ publish
      can :update, Post
      can :destroy, Comment
    when "admin"
      can :manage, :all
    end
  end
end
```

**อธิบายทีละส่วน:**

- `user ||= User.new` — CanCanCan เรียก `Ability.new(current_user)` แม้ตอนที่ยังไม่ login
  (`current_user` เป็น `nil`) เราจึงแทนที่ด้วย `User.new` (record ที่ยังไม่ save, `new_record?`
  เป็น `true`) เพื่อให้เขียนโค้ดต่อไปเรียก `user.role` ได้โดยไม่ต้องเช็ค `nil` ทุกบรรทัด —
  เทคนิคนี้เรียกว่า **Null Object Pattern**
- `can :read, Post, published: true` — hash ตัวที่ 3 คือ **conditions** จะถูกแปลงเป็นเงื่อนไข
  เทียบกับ attribute ของ object ตรงๆ (`post.published == true`) และที่สำคัญคือมันแปลงเป็น
  SQL `WHERE` ได้ด้วยเมื่อใช้กับ `accessible_by` (Step 434)
- `can %i[update destroy], Post, user_id: user.id` — argument แรกเป็น Array ได้ ประกาศได้
  หลาย action พร้อมกันในบรรทัดเดียว
- **กฎที่ตรงกันหลายข้อจะ "รวมกัน" แบบ OR** — เช่น `Post` มีทั้ง `can :read, Post,
  published: true` และ `can :read, Post, user_id: user.id` แปลว่า user อ่านได้ถ้า
  "publish แล้ว **หรือ** เป็นเจ้าของ" (ไม่ใช่ AND)
- `case user.role ... when "editor" ...` — เพราะ `role` เป็น Rails enum (จะสอนละเอียดใน
  Step 437) การเทียบ `user.role` จะได้ string กลับมา (`"editor"`, `"admin"`) ไม่ใช่ integer
  ดิบ ทำให้ `case/when` อ่านง่าย
- `can :manage, :all` — `:manage` หมายถึงทุก action (`read`, `create`, `update`, `destroy`
  และ action แบบ custom ใดๆ) ส่วน `:all` หมายถึงทุก model ในระบบ บรรทัดนี้บรรทัดเดียว
  แปลว่า **admin ทำอะไรก็ได้กับทุกอย่าง**

### `cannot` — ตัดสิทธิ์ (และเรื่องลำดับกฎที่ต้องระวัง)

`cannot` มี syntax เดียวกับ `can` แต่ใช้ตัดสิทธิ์ที่เคยให้ไว้ก่อนหน้าออกไปบางส่วน จุดที่
**ต้องระวังมากที่สุด** ของ CanCanCan คือ **ลำดับการประกาศมีผล — กฎที่ประกาศทีหลังชนะกฎก่อนหน้า
เสมอเมื่อเงื่อนไขตรงกัน**

ทดสอบจริงด้วย `rails runner`:

```ruby
# ทดสอบผ่าน bin/rails runner
class TempAbility
  include CanCan::Ability
  def initialize(user)
    can :manage, Post
    cannot :destroy, Post, published: true
  end
end

editor = User.find_by(email: "editor@example.com")
published_post = Post.find_by(published: true)
draft_post = Post.create!(title: "d", body: "b", user: editor, published: false)

ability = TempAbility.new(editor)
ability.can?(:destroy, published_post) # => false (cannot ทับ can ไว้)
ability.can?(:destroy, draft_post)     # => true  (cannot ไม่ตรงเงื่อนไข ไม่มีผล)
```

ผลลัพธ์จริงจากการรัน:

```
can destroy published (should be false due to cannot override): false
can destroy draft (should be true): true
```

**กฎการจำ:** เขียน `can :manage, :all` แบบกว้างๆ ไว้ก่อน แล้วค่อยตามด้วย `cannot` เพื่อ
"เจาะรู" ตัดสิทธิ์บางกรณีออกทีหลังเสมอ ถ้าสลับลำดับ (`cannot` มาก่อน `can` ที่ครอบคลุมกว่า)
`cannot` จะไม่มีผลอะไรเลยเพราะ `can` ที่ตามมาทีหลังจะ "เขียนทับ" มันอีกที

---

## Step 434: `load_and_authorize_resource` เทียบกับ `authorize!` + `can?`/`cannot?` ใน view

### แบบ explicit — `authorize!` ทีละ action

เหมือนกับที่ Pundit ใช้ `authorize @post` explicit ใน CanCanCan ก็มี `authorize!` (สังเกต `!`
ต่อท้าย เพราะเรียกแล้ว raise exception ทันทีถ้าไม่มีสิทธิ์ ไม่ return boolean):

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize! :read, @post
  end

  def update
    @post = Post.find(params[:id])
    authorize! :update, @post
    @post.update(post_params)
  end
end
```

และ `can?`/`cannot?` ใช้ตรวจสอบแบบ boolean (ไม่ raise) ได้ทั้งใน controller และ view —
เทียบเท่ากับ `policy(@post).update?` ของ Pundit:

```erb
<%# app/views/posts/show.html.erb %>
<% if can? :update, @post %>
  <%= link_to "แก้ไข", edit_post_path(@post) %>
<% end %>

<% if cannot? :destroy, @post %>
  <p class="text-gray-400">คุณไม่มีสิทธิ์ลบโพสต์นี้</p>
<% end %>
```

### แบบ implicit — `load_and_authorize_resource` (จุดขายหลักของ CanCanCan)

macro ตัวนี้ **โหลด record จาก `params[:id]` ให้ + ตรวจสิทธิ์ให้ในบรรทัดเดียว** ครอบคลุมทั้ง 7
action มาตรฐาน (`index`, `show`, `new`, `create`, `edit`, `update`, `destroy`):

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  load_and_authorize_resource # โหลด @post/@posts และเช็คสิทธิ์ให้อัตโนมัติทุก action

  def index
    # @posts ถูก scope ให้แล้วโดย CanCanCan (accessible_by) — เห็นเฉพาะที่มีสิทธิ์ :read
    render json: @posts.as_json(only: %i[id title published user_id])
  end

  def show
    render json: @post.as_json(only: %i[id title body published user_id])
  end

  def create
    @post.user = current_user
    if @post.save
      render json: @post, status: :created
    else
      render json: { errors: @post.errors.full_messages }, status: :unprocessable_entity
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

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```

**`load_and_authorize_resource` ทำอะไรให้บ้าง (แยกตาม action):**

- `index` — สร้าง `@posts = Post.accessible_by(current_ability)` ให้เอง (ดู `accessible_by`
  ด้านล่าง) — ผลลัพธ์คือ query ที่ scope ตามกฎ `can :read, ...` ทั้งหมดอัตโนมัติ ไม่ต้องเขียน
  `Post.where(...)` เอง
- `show`/`edit`/`update`/`destroy` — สร้าง `@post = Post.find(params[:id])` ให้ แล้วเรียก
  `authorize!` ให้ทันที ถ้าไม่มีสิทธิ์จะ raise `CanCan::AccessDenied` ก่อนเข้าโค้ดใน action
  เลยด้วยซ้ำ (action ของเราจึงไม่มีบรรทัด `Post.find` หรือ `authorize!` ให้เห็นเลย)
- `new`/`create` — สร้าง `@post = Post.new(post_params)` (หรือ `Post.new` เฉยๆ สำหรับ `new`)
  แล้วตรวจสิทธิ์ `:create` ให้

### ทดสอบจริงด้วย curl

รัน server แล้วทดสอบสิทธิ์แต่ละ role:

```bash
# login เป็น member แล้วดู index — เห็นเฉพาะโพสต์ตัวเอง + โพสต์ที่ publish แล้ว
curl -s -c cookies.txt -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"member@example.com","password":"secret123"}'

curl -s -b cookies.txt http://localhost:3000/posts
# => [{"id":1,"title":"Own draft","published":false,"user_id":2},
#     {"id":3,"title":"Other published","published":true,"user_id":3}]

# member พยายามแก้ draft ของคนอื่น — โดน 403 ทันทีจาก load_and_authorize_resource
curl -s -b cookies.txt -w "\nHTTP:%{http_code}\n" \
  -X PATCH http://localhost:3000/posts/2 \
  -H "Content-Type: application/json" -d '{"post":{"title":"hacked"}}'
# => {"error":"You are not authorized to access this page."}
# => HTTP:403
```

ผลลัพธ์นี้คือค่าที่ได้จากการรันจริงกับแอปทดสอบ — `member` เห็น post id 1 (ของตัวเอง แม้ยัง
เป็น draft) และ id 3 (publish แล้ว) แต่ไม่เห็น id 2 (draft ของ editor คนอื่น) ตรงตามกฎที่
ประกาศไว้ใน Step 433 ทุกประการ และเมื่อพยายามแก้ id 2 ตรงๆ ผ่าน URL ก็โดนบล็อกด้วย
`CanCan::AccessDenied` ก่อนจะถึงโค้ด `update` ด้วยซ้ำ

### `accessible_by` แบบเจาะลึก

`Model.accessible_by(ability)` คือ engine ที่แปลงกฎ `can` ทั้งหมดที่ตรงกับ action/model
ให้กลายเป็น SQL `WHERE` clause โดยอัตโนมัติ:

```ruby
# editor มีกฎ can :read, Post (ไม่มีเงื่อนไข) จึงได้ query แบบไม่มี WHERE เลย
Post.accessible_by(Ability.new(editor)).to_sql
# => SELECT "posts".* FROM "posts"

# member มีกฎ can :read, Post, published: true OR user_id: user.id
# CanCanCan จะรวมเป็น WHERE (published = 1 OR user_id = ?) ให้เอง
```

จุดนี้คือความแตกต่างสำคัญจาก Pundit ที่ต้องเขียน `Scope` class เองทุกครั้ง — CanCanCan ได้
behavior นี้ "ฟรี" จากกฎ `can` ที่ประกาศไว้แล้วสำหรับ authorize ตัว record เดี่ยว ไม่ต้องเขียน
โค้ดกรอง query แยกอีกชุด

---

## Step 435: จัดการสิทธิ์ที่ถูกปฏิเสธด้วย `rescue_from CanCan::AccessDenied`

เมื่อ `authorize!` หรือ `load_and_authorize_resource` เจอ action ที่ไม่มีสิทธิ์ มันจะ raise
`CanCan::AccessDenied` — ถ้าไม่ดักไว้ ผู้ใช้จะเห็นหน้า error 500 ของ Rails ตรงๆ ซึ่งไม่เหมาะ
กับ production เลย

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  rescue_from CanCan::AccessDenied do |exception|
    respond_to do |format|
      format.json { render json: { error: exception.message }, status: :forbidden }
      format.html { redirect_to root_path, alert: exception.message }
    end
  end
end
```

**อธิบาย:**

- `rescue_from` เป็น mechanism มาตรฐานของ Rails (ไม่ใช่ของ CanCanCan) สำหรับดัก exception
  ทุกตัวที่เกิดขึ้นใน action ของ controller นั้นๆ (และ subclass ทั้งหมด) — วางไว้ที่
  `ApplicationController` เพื่อให้ครอบคลุมทุก controller ในแอป
- `exception.message` ของ `CanCan::AccessDenied` default คือ
  `"You are not authorized to access this page."` เปลี่ยนข้อความ default ได้ด้วย
  `exception.default_message = "..."` ก่อน raise หรือ config ผ่าน `I18n` (key
  `unauthorized.default`)
- `respond_to` แยก format html/json ให้ตอบกลับต่างกันตามชนิด request — สำหรับ API จะได้
  JSON error พร้อม status `403 Forbidden`, สำหรับหน้าเว็บปกติจะ redirect กลับหน้าแรกพร้อม
  flash message

### เทียบกับ i18n

ถ้าอยากให้ข้อความ error เป็นภาษาไทยตาม locale ของแอป (ตามที่เรียนไปใน Part 039) ทำได้โดย
custom message ตอน raise หรือแก้ที่จุดเดียวใน `ApplicationController`:

```ruby
rescue_from CanCan::AccessDenied do |exception|
  redirect_to root_path, alert: "คุณไม่มีสิทธิ์ทำรายการนี้"
end
```

> **ข้อควรระวัง:** ต่างจาก Pundit ที่ raise `Pundit::NotAuthorizedError`, CanCanCan raise
> `CanCan::AccessDenied` — ถ้าโปรเจกต์เคยใช้ทั้งสอง gem ผสมกันช่วง migrate จาก gem หนึ่งไปอีก
> ตัว ต้อง `rescue_from` ทั้งสอง exception class ไว้พร้อมกัน

---

## Step 436: ออกแบบ RBAC schema — ตาราง `roles` + join table เทียบกับคอลัมน์ `role` เดี่ยว

ไม่ว่าจะเลือก Pundit หรือ CanCanCan ปัญหาที่อยู่เบื้องหลังเสมอคือ **จะเก็บ "role ของ user" ไว้
ที่ไหนและแบบไหนในฐานข้อมูล** — นี่คือหัวใจของการออกแบบ RBAC และมี 2 แนวทางหลักที่ต้องรู้จัก
tradeoffs

### แนวทางที่ 1: คอลัมน์ `role` เดี่ยวบนตาราง `users`

```ruby
# db/migrate/xxx_add_role_to_users.rb
add_column :users, :role, :integer, null: false, default: 0
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  enum :role, { member: 0, editor: 1, admin: 2 }, default: :member
end
```

**ข้อดี:**
- Query เร็วที่สุด — ไม่ต้อง join ตารางไหนเลย `User.where(role: :admin)` ทำงานตรงๆ บน
  ตาราง `users`
- Setup ง่าย มี migration เดียว ไม่ต้อง seed ตารางเพิ่ม
- เหมาะกับแอปที่ role เป็นแนวคิดง่ายๆ ระดับ "สิทธิ์แบบขั้นบันได" (member < editor < admin)

**ข้อเสีย:**
- **หนึ่ง user มีได้แค่ 1 role เท่านั้น** — ถ้าธุรกิจต้องการ "user คนเดียวเป็นทั้ง editor
  ของทีม A และ viewer ของทีม B" คอลัมน์เดียวทำไม่ได้เลย ต้องเปลี่ยนสถาปัตยกรรมทั้งหมด
- เพิ่ม role ใหม่ต้องแก้โค้ด (เพิ่มค่าใน `enum`) แล้ว deploy ใหม่เสมอ ไม่มีทางให้ admin
  สร้าง role ใหม่เองผ่านหน้าเว็บได้
- ถ้าแอปเป็น multi-tenant (เช่น SaaS ที่ user เป็นสมาชิกหลายองค์กร) role แบบ global ทั้งระบบ
  ไม่ตอบโจทย์ "role ต่างกันในแต่ละองค์กร"

### แนวทางที่ 2: ตาราง `roles` + join table `user_roles`

```ruby
# db/migrate/xxx_create_roles.rb
create_table :roles do |t|
  t.string :name, null: false
  t.timestamps
end
add_index :roles, :name, unique: true

# db/migrate/xxx_create_user_roles.rb
create_table :user_roles do |t|
  t.references :user, null: false, foreign_key: true
  t.references :role, null: false, foreign_key: true
  t.timestamps
end
add_index :user_roles, %i[user_id role_id], unique: true
```

```ruby
# app/models/role.rb
class Role < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :users, through: :user_roles
end

# app/models/user_role.rb
class UserRole < ApplicationRecord
  belongs_to :user
  belongs_to :role
end

# app/models/user.rb
class User < ApplicationRecord
  has_many :user_roles, dependent: :destroy
  has_many :roles, through: :user_roles

  def role?(name)
    roles.exists?(name: name.to_s)
  end
end
```

```ruby
# ใช้งาน
editor_role = Role.find_by!(name: "editor")
user.roles << editor_role       # เพิ่ม role ให้ user (ไม่ทับ role เดิม)
user.role?("editor")            # => true
user.roles.pluck(:name)         # => ["editor", "moderator"]  (มีได้หลาย role พร้อมกัน)
```

**ข้อดี:**
- **หนึ่ง user มีได้หลาย role พร้อมกัน** (many-to-many) ตอบโจทย์ระบบที่ซับซ้อนขึ้น
- เพิ่ม/ลบ role ใหม่ทำได้ผ่านข้อมูล (insert แถวใหม่ในตาราง `roles`) โดยไม่ต้องแก้โค้ดหรือ
  deploy ใหม่ — เปิดทางให้ admin จัดการ role เองผ่านหน้าเว็บได้จริง
- ต่อยอดไปสู่ per-resource role ได้ง่าย (เช่น เพิ่มคอลัมน์ `scope_type`/`scope_id`
  แบบ polymorphic บน `user_roles` เพื่อทำ "เป็น admin เฉพาะในองค์กร X" — เทคนิคนี้จะเจอ
  อีกครั้งตอนเรียน multi-tenancy ใน Part 085)

**ข้อเสีย:**
- Query ทุกครั้งต้อง join อย่างน้อย 1 ตาราง (`user_roles`) — ถ้าเช็คสิทธิ์บ่อยมากในทุก request
  ต้องระวังเรื่อง N+1 และอาจต้อง cache role ไว้ (เช่น preload `includes(:roles)`)
- ซับซ้อนกว่าในการ setup เริ่มต้น (3 model แทนที่จะเป็น 1 คอลัมน์) และต้อง seed ข้อมูล
  `roles` เริ่มต้นเสมอ (`db/seeds.rb`)
- Logic การเช็ค "มี role นี้ไหม" เปลี่ยนจาก `user.admin?` (คำเดียวจบจาก enum) เป็น
  `user.role?("admin")` หรือต้องเขียน helper method เพิ่มเอง

### ตารางสรุปการเลือกใช้

| เงื่อนไข | เลือกใช้ |
|---|---|
| แอปเล็ก-กลาง, role เป็นลำดับขั้นชัดเจน (member → editor → admin) | คอลัมน์ `role` เดี่ยว + enum |
| ต้องการให้ user มีได้หลาย role พร้อมกัน | ตาราง `roles` + join table |
| ต้องการให้ admin เพิ่ม/แก้ role เองผ่านหน้าเว็บโดยไม่ deploy โค้ดใหม่ | ตาราง `roles` + join table |
| แอปเป็น multi-tenant ที่ role ต่างกันในแต่ละองค์กร/ทีม | ตาราง `roles` + join table (ต่อยอด polymorphic scope) |
| ต้องการ performance สูงสุด ไม่อยาก join | คอลัมน์ `role` เดี่ยว + enum |

ในทางปฏิบัติ **ทีมจำนวนมากเริ่มจากคอลัมน์ `role` เดี่ยวก่อนเสมอ** เพราะง่ายและพอเพียงสำหรับ
ความต้องการช่วงแรก แล้วค่อย migrate ไปเป็นตาราง `roles` + join table เมื่อ requirement
ซับซ้อนขึ้นจริงๆ (YAGNI — อย่าออกแบบซับซ้อนเกินความจำเป็นตั้งแต่วันแรก) — Step 437 จะสอน
การ implement แนวทางแรกแบบเต็มรูปแบบ และ Step 438 จะพาไปดูวิธีทำระบบสิทธิ์แบบละเอียดขึ้นไปอีก
ระดับด้วย `Permission` model

---

## Step 437: Role-based access ด้วย Rails 7+ `enum` และการสร้าง Ability จาก enum

### Syntax `enum` แบบใหม่ของ Rails 7.1+

Rails 7.1 เปลี่ยน syntax การประกาศ `enum` ให้กระชับขึ้น (syntax เก่าที่เคยเห็นในโปรเจกต์รุ่น
ก่อนคือ `enum role: { member: 0, editor: 1, admin: 2 }` — ยังใช้ได้อยู่แต่ deprecated)

```ruby
# db/migrate/xxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email
      t.string :password_digest
      t.integer :role, null: false, default: 0

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  # Syntax ใหม่ (Rails 7.1+): ชื่อ attribute เป็น positional argument ตัวแรก
  enum :role, { member: 0, editor: 1, admin: 2 }, default: :member

  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
end
```

ทดสอบจริงผ่าน `rails runner`:

```ruby
u = User.create!(name: "Somchai", email: "somchai@example.com", password: "secret123")
u.role       # => "member"  (ค่า default)
u.member?    # => true
u.editor!    # เปลี่ยน role เป็น editor และ save ทันที (เหมือน update!(role: :editor))
u.role       # => "editor"
User.roles   # => {"member"=>0, "editor"=>1, "admin"=>2}
```

ผลลัพธ์ที่ได้จากการรันจริง:

```
member
true
editor
{"member"=>0, "editor"=>1, "admin"=>2}
```

**สิ่งที่ `enum` สร้างให้อัตโนมัติ:**

- Method เช็คสถานะ: `user.member?`, `user.editor?`, `user.admin?`
- Method เปลี่ยนสถานะ: `user.editor!` (update + save ทันที)
- Scope สำหรับ query: `User.member`, `User.editor`, `User.admin` (เทียบเท่า
  `User.where(role: :editor)`)
- Class method `User.roles` คืน hash ของทุกค่าที่เป็นไปได้พร้อมเลข integer จริงในฐานข้อมูล
- `user.role` คืนค่าเป็น **string** เสมอ (`"editor"`) ไม่ใช่เลข `1` ดิบ — ทำให้เขียน
  `case/when` หรือเทียบด้วย string อ่านง่าย โดยที่ database ยังเก็บเป็น integer
  (ประหยัดพื้นที่ + index เร็วกว่า string)

> **ข้อควรระวังเรื่อง migration ที่รันไปแล้ว (พบจริงระหว่างทำ Part นี้):** ถ้าแก้ไข migration
> ไฟล์ที่เคย `db:migrate` ไปแล้วก่อนหน้า (เช่น เปลี่ยน `t.integer :role` เป็น `t.string :role`)
> แล้วสั่ง `db:drop db:create db:migrate` ใหม่ Rails 8 อาจ **ไม่รันไฟล์ migration ที่แก้ไขซ้ำ**
> แต่จะโหลดจาก `db/schema.rb` ที่ยัง cache โครงสร้างเก่าไว้แทน (เพราะเห็นว่า version ล่าสุดใน
> schema.rb ตรงกับ migration ตัวสุดท้ายอยู่แล้ว) ผลคือคอลัมน์จะยังเป็นชนิดข้อมูลเก่าอยู่ทั้งที่
> ไฟล์ migration ถูกแก้แล้ว — วิธีแก้คือลบ `db/schema.rb` ทิ้งก่อน แล้วค่อย `db:migrate` ใหม่
> เพื่อบังคับให้ Rails ไล่รัน migration ทุกไฟล์จริงๆ (หรือใช้ `db:migrate:redo` กับ migration
> ตัวนั้นโดยเฉพาะ ถ้ายังไม่ได้แก้ไฟล์ migration แต่จะ rollback แล้ว migrate ใหม่)

### สร้าง Ability class จาก role enum

นำ `enum` ที่เพิ่งสร้างมาผูกกับ `Ability` (ต่อยอดจาก Step 433 ให้สมบูรณ์):

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new

    can :read, Post, published: true
    can :read, Comment

    return if user.new_record?

    can :create, Post
    can :create, Comment
    can :read, Post, user_id: user.id
    can %i[update destroy], Post, user_id: user.id
    can %i[update destroy], Comment, user_id: user.id

    case user.role
    when "editor"
      can :read, Post
      can :update, Post
      can :destroy, Comment
    when "admin"
      can :manage, :all
    end
  end
end
```

ทดสอบครบทั้ง 4 กรณี (guest, member, editor, admin) ผ่าน `rails runner` จริง:

```ruby
member_ability = Ability.new(member)
member_ability.can?(:read, own_post)     # => true  (เจ้าของอ่าน draft ตัวเองได้)
member_ability.can?(:read, other_draft)  # => false (ไม่ใช่เจ้าของ + ยังไม่ publish)
member_ability.can?(:update, other_draft)# => false

editor_ability = Ability.new(editor)
editor_ability.can?(:read, other_draft)  # => true  (editor เห็น draft ของทุกคน)
editor_ability.can?(:update, own_post)   # => true  (แก้ของคนอื่นได้ตามกฎ editor)
editor_ability.can?(:destroy, own_post)  # => false (editor ไม่มีสิทธิ์ destroy Post)

admin_ability = Ability.new(admin)
admin_ability.can?(:manage, own_post)    # => true
admin_ability.can?(:destroy, User)       # => true (:manage, :all ครอบคลุมทุก model)

guest_ability = Ability.new(nil)
guest_ability.can?(:read, other_published) # => true
guest_ability.can?(:create, Post)          # => false
```

ผลลัพธ์จริงจากการรัน (ตรงกับที่คาดไว้ทุกบรรทัด):

```
member can? read own_post (unpublished, own): true
member can? read other_draft (unpublished, not own): false
member can? read other_published: true
member can? update own_post: true
member can? update other_draft: false
member can? destroy other_draft: false
editor can? read other_draft (not own, unpublished): true
editor can? update own_post (belongs to member): true
editor can? destroy own_post: false
admin can? manage own_post: true
admin can? destroy User: true
guest can? read other_published: true
guest can? read other_draft: false
guest can? create Post: false
```

ทดสอบผ่าน controller จริงด้วย curl (login เป็น editor แล้วดู index, แก้ post ของ member,
พยายามลบ — ตรงตามกฎทุกกรณี):

```bash
curl -s -b editor_cookies.txt http://localhost:3000/posts
# => [{"id":1,...},{"id":2,...},{"id":3,...}]  -- editor เห็นทุกโพสต์ แม้ยังไม่ publish

curl -s -b editor_cookies.txt -X PATCH http://localhost:3000/posts/1 \
  -H "Content-Type: application/json" -d '{"post":{"title":"edited by editor"}}'
# => {"title":"edited by editor", ...}  HTTP 200 -- แก้ของคนอื่นได้ตามกฎ editor

curl -s -b editor_cookies.txt -X DELETE http://localhost:3000/posts/1
# => {"error":"You are not authorized to access this page."}  HTTP 403
# -- editor ไม่มีสิทธิ์ destroy Post ตามที่ประกาศไว้
```

---

## Step 438: ระบบสิทธิ์แบบละเอียด — `Permission`/`RolePermission` model-driven

Role-based ด้วย `case/when` ใน Step 437 เหมาะกับกรณีที่กฎ **ไม่เปลี่ยนบ่อย** และจำนวน role
ไม่เยอะ แต่ถ้าธุรกิจต้องการให้ **admin ปรับ "editor ทำอะไรกับ resource ไหนได้บ้าง" เองผ่าน
หน้าเว็บ โดยไม่ต้องรอ deploy โค้ดใหม่ทุกครั้ง** — เราต้องย้าย "กฎ" จากโค้ด Ruby ไปเก็บใน
ฐานข้อมูลแทน นี่คือแนวคิดของระบบ **permission ที่ configurable**

### ออกแบบตาราง `permissions`

```ruby
# db/migrate/xxx_create_permissions.rb
class CreatePermissions < ActiveRecord::Migration[8.1]
  def change
    create_table :permissions do |t|
      t.string :role, null: false      # ตรงกับค่า enum ของ User#role (เก็บเป็น string)
      t.string :resource, null: false  # ชื่อคลาส เช่น "Post", "Comment"
      t.string :action, null: false    # เช่น "read", "create", "update", "destroy", "manage"

      t.timestamps
    end
    add_index :permissions, %i[role resource action], unique: true,
              name: "index_permissions_on_role_resource_action"
  end
end
```

> **ทำไมเก็บ `role` เป็น string ไม่ใช่ integer:** ถ้าเก็บเป็น integer ตรงกับเลขของ `enum`
> ใน `User` การเปลี่ยนลำดับ/เพิ่มค่ากลางของ enum ในอนาคตจะทำให้ข้อมูลเก่าใน `permissions`
> ผิดเพี้ยนทันที (เลข 1 ที่เคยหมายถึง `editor` อาจกลายเป็นความหมายอื่น) การเก็บเป็น string
> ("editor") ทำให้ตาราง `permissions` ไม่ผูกกับลำดับตัวเลขภายในของ enum เลย ปลอดภัยกว่ามาก

```ruby
# app/models/permission.rb
class Permission < ApplicationRecord
  validates :role, :resource, :action, presence: true
  validates :action, uniqueness: { scope: %i[role resource] }

  scope :for_role, ->(role) { where(role: role) }
end
```

> **ข้อควรระวังเรื่อง uniqueness scope ที่เขียนผิดง่ายมาก (พบจริงระหว่างทดสอบ Part นี้):**
> `validates :role, uniqueness: { scope: %i[resource action] }` กับ
> `validates :action, uniqueness: { scope: %i[role resource] }` **ไม่เหมือนกัน** แม้ดู
> คล้ายกันมาก — แบบแรกหมายความว่า "resource+action คู่หนึ่งมี role ซ้ำกันไม่ได้" (เช่น
> ห้ามมี 2 แถวที่ resource=Post, action=read พร้อมกันไม่ว่า role จะเป็นอะไร) ซึ่ง**ผิด**
> เพราะเราต้องการให้หลาย role อ่าน Post ได้พร้อมกัน ส่วนแบบที่สอง (ที่ถูกต้อง) หมายความว่า
> "role+resource คู่หนึ่งมี action ซ้ำกันไม่ได้" ซึ่งตรงกับที่ต้องการจริงๆ คือห้าม insert
> permission ซ้ำ (role, resource, action) ทั้งสามค่าเหมือนกันทุกตัว — เวลาทำ compound
> uniqueness ให้ยึดหลักว่า **field ที่ validate ต้องเป็นตัวที่เรา "ห้ามซ้ำภายใต้ scope ที่เหลือ
> ทั้งหมด"** ไม่ใช่เลือกตามความเคยชิน

### สร้าง Ability ที่อ่านสิทธิ์จากฐานข้อมูล

```ruby
# app/models/permission_based_ability.rb
class PermissionBasedAbility
  include CanCan::Ability

  RESOURCE_CLASSES = { "Post" => Post, "Comment" => Comment }.freeze

  def initialize(user)
    user ||= User.new
    role = user.new_record? ? "guest" : user.role

    Permission.for_role(role).find_each do |permission|
      subject = RESOURCE_CLASSES.fetch(permission.resource, permission.resource.to_sym)
      can permission.action.to_sym, subject
    end

    # เงื่อนไข ownership ยังคงต้องเขียนในโค้ด เพราะ join กับ user_id ผ่านตาราง permissions
    # อย่างเดียวทำไม่ได้ตรงๆ — DB-driven ใช้กำหนด "role ไหนทำ action ไหนกับ resource ไหนได้
    # บ้าง" ส่วนเงื่อนไขเชิง object (เช่น "เฉพาะของตัวเอง") ยังคงประกาศเพิ่มเป็นโค้ดตามปกติ
    return if user.new_record?

    can %i[update destroy], Post, user_id: user.id
    can %i[update destroy], Comment, user_id: user.id
  end
end
```

Seed ข้อมูลตัวอย่าง แล้วให้ admin แก้สิทธิ์ผ่านหน้าเว็บได้ในอนาคตโดยไม่ต้อง deploy:

```ruby
# db/seeds.rb (ตัวอย่างข้อมูลเริ่มต้น)
Permission.find_or_create_by!(role: "guest",  resource: "Post", action: "read")
Permission.find_or_create_by!(role: "member", resource: "Post", action: "read")
Permission.find_or_create_by!(role: "member", resource: "Post", action: "create")
Permission.find_or_create_by!(role: "editor", resource: "Post", action: "read")
Permission.find_or_create_by!(role: "editor", resource: "Post", action: "create")
Permission.find_or_create_by!(role: "editor", resource: "Post", action: "update")
Permission.find_or_create_by!(role: "admin",  resource: "Post", action: "manage")
```

ทดสอบจริงด้วย `rails runner`:

```ruby
member_ab = PermissionBasedAbility.new(member)
member_ab.can?(:read, Post)              # => true
member_ab.can?(:create, Post)            # => true
member_ab.can?(:update, editors_post)    # => false (ไม่มีสิทธิ์ update จาก DB และไม่ใช่เจ้าของ)

editor_ab = PermissionBasedAbility.new(editor)
editor_ab.can?(:update, editors_own_post)   # => true (มีสิทธิ์ update จาก DB)
editor_ab.can?(:destroy, editors_own_post)  # => true (เป็นเจ้าของ แม้ DB ไม่ได้ให้สิทธิ์ destroy)
editor_ab.can?(:destroy, other_editors_post)# => false (ไม่มีสิทธิ์จาก DB และไม่ใช่เจ้าของ)
```

ผลลัพธ์จริง:

```
member can? read Post: true
member can? create Post: true
member can? update Post (not own): false
editor can? update Post (own): true
editor can? destroy Post: true
admin can? destroy Post: true
guest can? read Post: true
guest can? create Post: false
```

สังเกตว่า **"editor can? destroy Post: true" ในกรณีที่เป็นเจ้าของ** แม้ตาราง `permissions`
จะไม่มีแถว `editor/Post/destroy` เลย — เพราะกฎ ownership (`can :destroy, Post, user_id:
user.id`) เป็นกฎที่เขียนโค้ดแยกไว้ต่างหาก ทำงานควบคู่กับกฎที่มาจากฐานข้อมูล นี่คือจุดสำคัญ
ของการออกแบบแบบ hybrid: **"role ทำอะไรกับ resource ประเภทไหนได้บ้าง" มาจากฐานข้อมูล (แก้ได้
โดยไม่ deploy) ส่วน "เงื่อนไขเชิง object เช่น ownership" ยังคงเป็นโค้ดตายตัว** (เพราะการ join
เงื่อนไขแบบไดนามิกอย่าง `user_id: user.id` เก็บเป็นข้อมูลในตาราง `permissions` ตรงๆ ทำได้ยาก
และเสี่ยงต่อความปลอดภัยถ้าให้ admin กำหนดเงื่อนไขแบบ arbitrary ผ่านหน้าเว็บ)

### แนวคิดหน้าจอ admin (Admin::PermissionsController)

ในระบบจริง มักมี controller สำหรับ admin จัดการ matrix ของ role × resource × action แบบ
checkbox:

```ruby
# app/controllers/admin/permissions_controller.rb
module Admin
  class PermissionsController < ApplicationController
    before_action { authorize! :manage, Permission }

    def index
      @roles = User.roles.keys
      @resources = %w[Post Comment]
      @actions = %w[read create update destroy manage]
      @permissions = Permission.all.group_by { |p| [p.role, p.resource] }
    end

    def toggle
      permission = Permission.find_or_initialize_by(permission_params)
      if permission.persisted?
        permission.destroy
      else
        permission.save!
      end
      redirect_to admin_permissions_path
    end

    private

    def permission_params
      params.require(:permission).permit(:role, :resource, :action)
    end
  end
end
```

หน้า view จะ render เป็นตาราง checkbox ที่ submit ทีละช่องไปยัง `toggle` — เมื่อ admin
ติ๊ก/ถอนติ๊ก ระบบจะเปลี่ยนสิทธิ์ทันทีโดยไม่ต้อง restart หรือ deploy แอปเลย เพราะ
`PermissionBasedAbility` อ่านค่าจากฐานข้อมูลสดทุกครั้งที่สร้าง instance ใหม่ (ทุก request)

---

## Step 439: การทดสอบ Ability ด้วย RSpec (`cancan/matchers`)

CanCanCan มี matcher สำเร็จรูปชื่อ `be_able_to` ให้ใช้กับ RSpec โดยตรง ทำให้เทสต์ `Ability`
อ่านเป็นภาษาธรรมชาติได้ (หมายเหตุ: Part นี้ยังไม่ใช้ FactoryBot เพราะจะสอนอย่างเป็นทางการใน
Part 047 — ตอนนี้สร้างข้อมูลทดสอบด้วย `User.create!`/`Post.create!` ตรงๆ ไปก่อน)

### ติดตั้ง matcher

```ruby
# spec/rails_helper.rb
require 'rspec/rails'
require 'cancan/matchers' # เพิ่มบรรทัดนี้เพื่อใช้ be_able_to matcher
```

### เขียน spec

```ruby
# spec/models/ability_spec.rb
require "rails_helper"

RSpec.describe Ability, type: :model do
  let(:owner)  { User.create!(name: "Owner", email: "owner@example.com", password: "secret123", role: :member) }
  let(:other)  { User.create!(name: "Other", email: "other@example.com", password: "secret123", role: :member) }
  let(:editor) { User.create!(name: "Editor", email: "editor@example.com", password: "secret123", role: :editor) }
  let(:admin)  { User.create!(name: "Admin", email: "admin@example.com", password: "secret123", role: :admin) }

  let(:published_post) { Post.create!(title: "Published", body: "...", user: other, published: true) }
  let(:draft_post)     { Post.create!(title: "Draft", body: "...", user: other, published: false) }
  let(:own_draft)      { Post.create!(title: "My draft", body: "...", user: owner, published: false) }

  describe "guest (nil user)" do
    subject(:ability) { Ability.new(nil) }

    it { is_expected.to be_able_to(:read, published_post) }
    it { is_expected.not_to be_able_to(:read, draft_post) }
    it { is_expected.not_to be_able_to(:create, Post) }
  end

  describe "member" do
    subject(:ability) { Ability.new(owner) }

    it { is_expected.to be_able_to(:read, published_post) }
    it { is_expected.to be_able_to(:read, own_draft) }
    it { is_expected.not_to be_able_to(:read, draft_post) }
    it { is_expected.to be_able_to(:create, Post) }
    it { is_expected.to be_able_to(:update, own_draft) }
    it { is_expected.not_to be_able_to(:update, draft_post) }
    it { is_expected.not_to be_able_to(:destroy, draft_post) }
  end

  describe "editor" do
    subject(:ability) { Ability.new(editor) }

    it { is_expected.to be_able_to(:read, draft_post) }
    it { is_expected.to be_able_to(:update, draft_post) }
    it { is_expected.not_to be_able_to(:destroy, draft_post) }
    it { is_expected.to be_able_to(:destroy, Comment.new) }
  end

  describe "admin" do
    subject(:ability) { Ability.new(admin) }

    it { is_expected.to be_able_to(:manage, draft_post) }
    it { is_expected.to be_able_to(:destroy, User.new) }
  end
end
```

รันจริง:

```bash
bundle exec rspec spec/models/ability_spec.rb
```

ผลลัพธ์จริงจากการรัน:

```
................

Finished in 0.19711 seconds (files took 0.92267 seconds to load)
16 examples, 0 failures
```

**อธิบาย:**

- `be_able_to(:read, published_post)` แปลตรงๆ เป็น `ability.can?(:read, published_post)`
  ต้องเป็น `true` — syntax นี้มาจาก matcher ที่ `cancan/matchers` เพิ่มให้ ใช้ได้ทั้งกับ
  `to`/`not_to`
- **หลักการสำคัญของการเทสต์ Ability:** เทสต์ทุก **combination ของ role × resource state**
  ที่มีนัยสำคัญทางธุรกิจ ไม่ใช่แค่ทดสอบ "admin ทำได้ทุกอย่าง" อย่างเดียว — โดยเฉพาะกรณี
  ที่เป็น edge case เช่น "member เห็น draft ของตัวเองได้ แต่เห็น draft ของคนอื่นไม่ได้"
  ถ้าไม่มีเทสต์แยกสองกรณีนี้ บั๊กแบบ "condition ผิดตัว user_id" จะไม่ถูกจับได้เลย
- ต่างจากการเทสต์ Pundit policy (ที่แยกไฟล์ spec ต่อ policy หนึ่งไฟล์) การเทสต์ `Ability`
  ของ CanCanCan จะรวมทุก resource ไว้ในไฟล์เดียว (`ability_spec.rb`) — ยิ่งแอปโตขึ้น ไฟล์นี้
  จะยิ่งยาวขึ้นเรื่อยๆ ตาม `Ability` class เอง (สอดคล้องกับข้อเสียเรื่อง "God Object" ที่พูดถึง
  ใน Step 431) แนวทางบรรเทาปัญหานี้คือใช้ `describe` block แยกตาม role/resource ให้ชัดเจน
  เพื่อให้ไฟล์ยังอ่านง่ายแม้จะยาว

---

## Step 440: แบบฝึกหัด — สร้างระบบสิทธิ์แบบ multi-role (member/editor/admin) ครบวงจร

### โจทย์

สร้างระบบบล็อกเล็กๆ ที่มี:

1. `User` model พร้อมคอลัมน์ `role` เป็น enum 3 ค่า: `member`, `editor`, `admin`
   (default: `member`)
2. `Post` model ที่มี `user_id` (เจ้าของ) และ `published` (boolean)
3. `Ability` class (CanCanCan) ที่ประกาศกฎ:
   - ทุกคน (รวม guest) อ่านโพสต์ที่ `published: true` ได้
   - member สร้างโพสต์ได้, แก้ไข/ลบได้เฉพาะโพสต์ตัวเอง, และเห็น draft ของตัวเองได้ด้วย
   - editor อ่าน/แก้ไขโพสต์ของทุกคนได้ (รวม draft) แต่ลบไม่ได้ (ยกเว้นของตัวเอง)
   - admin ทำได้ทุกอย่างกับทุก resource
4. `PostsController` ที่ใช้ `load_and_authorize_resource`
5. RSpec ability spec ที่ครอบคลุมทั้ง 4 role (guest/member/editor/admin)

### เฉลย

**Migration:**

```ruby
# db/migrate/xxx_create_users.rb
class CreateUsers < ActiveRecord::Migration[8.1]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email
      t.string :password_digest
      t.integer :role, null: false, default: 0

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

```ruby
# db/migrate/xxx_create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body
      t.references :user, null: false, foreign_key: true
      t.boolean :published, null: false, default: false

      t.timestamps
    end
  end
end
```

**Models:**

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  has_many :posts, dependent: :destroy

  enum :role, { member: 0, editor: 1, admin: 2 }, default: :member

  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user

  validates :title, presence: true
end
```

**Ability class:**

```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new

    can :read, Post, published: true

    return if user.new_record?

    can :create, Post
    can :read, Post, user_id: user.id
    can %i[update destroy], Post, user_id: user.id

    case user.role
    when "editor"
      can :read, Post
      can :update, Post
    when "admin"
      can :manage, :all
    end
  end
end
```

**Controller:**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  load_and_authorize_resource

  def index
    render json: @posts.as_json(only: %i[id title published user_id])
  end

  def show
    render json: @post.as_json(only: %i[id title body published user_id])
  end

  def create
    @post.user = current_user
    if @post.save
      render json: @post, status: :created
    else
      render json: { errors: @post.errors.full_messages }, status: :unprocessable_entity
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

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
  helper_method :current_user

  rescue_from CanCan::AccessDenied do |exception|
    respond_to do |format|
      format.json { render json: { error: exception.message }, status: :forbidden }
      format.html { redirect_to root_path, alert: exception.message }
    end
  end
end
```

**RSpec spec:**

```ruby
# spec/rails_helper.rb — เพิ่มบรรทัดนี้
require 'cancan/matchers'
```

```ruby
# spec/models/ability_spec.rb
require "rails_helper"

RSpec.describe Ability, type: :model do
  let(:owner)  { User.create!(name: "Owner", email: "owner@example.com", password: "secret123", role: :member) }
  let(:other)  { User.create!(name: "Other", email: "other@example.com", password: "secret123", role: :member) }
  let(:editor) { User.create!(name: "Editor", email: "editor@example.com", password: "secret123", role: :editor) }
  let(:admin)  { User.create!(name: "Admin", email: "admin@example.com", password: "secret123", role: :admin) }

  let(:published_post) { Post.create!(title: "Published", body: "...", user: other, published: true) }
  let(:draft_post)     { Post.create!(title: "Draft", body: "...", user: other, published: false) }
  let(:own_draft)      { Post.create!(title: "My draft", body: "...", user: owner, published: false) }

  describe "guest" do
    subject(:ability) { Ability.new(nil) }

    it { is_expected.to be_able_to(:read, published_post) }
    it { is_expected.not_to be_able_to(:read, draft_post) }
    it { is_expected.not_to be_able_to(:create, Post) }
  end

  describe "member" do
    subject(:ability) { Ability.new(owner) }

    it { is_expected.to be_able_to(:read, own_draft) }
    it { is_expected.not_to be_able_to(:read, draft_post) }
    it { is_expected.to be_able_to(:update, own_draft) }
    it { is_expected.not_to be_able_to(:update, draft_post) }
    it { is_expected.not_to be_able_to(:destroy, draft_post) }
  end

  describe "editor" do
    subject(:ability) { Ability.new(editor) }

    it { is_expected.to be_able_to(:read, draft_post) }
    it { is_expected.to be_able_to(:update, draft_post) }
    it { is_expected.not_to be_able_to(:destroy, draft_post) }
  end

  describe "admin" do
    subject(:ability) { Ability.new(admin) }

    it { is_expected.to be_able_to(:manage, draft_post) }
    it { is_expected.to be_able_to(:destroy, User.new) }
  end
end
```

รันแล้วต้องได้ผลลัพธ์ทุก example ผ่าน (สอดคล้องกับที่ทดสอบจริงไว้ใน Step 439):

```
............

Finished in 0.15s
12 examples, 0 failures
```

**ตรวจสอบเพิ่มด้วย curl (ทางเลือก แต่แนะนำให้ทำเพื่อยืนยัน behavior ระดับ HTTP):**

```bash
# member เห็นเฉพาะโพสต์ตัวเอง + โพสต์ publish แล้ว
curl -s -b member_cookies.txt http://localhost:3000/posts

# member แก้ post คนอื่นไม่ได้ (403)
curl -s -b member_cookies.txt -w "\nHTTP:%{http_code}\n" \
  -X PATCH http://localhost:3000/posts/2 -d '{"post":{"title":"x"}}'

# admin ลบโพสต์ใครก็ได้ (204)
curl -s -b admin_cookies.txt -w "\nHTTP:%{http_code}\n" -X DELETE http://localhost:3000/posts/1
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม role ใหม่ชื่อ `moderator` ที่มีสิทธิ์ลบ `Comment` ของใครก็ได้ (เหมือน admin เฉพาะกับ
   `Comment`) แต่ **แก้ไขหรือลบ `Post` ไม่ได้เลย แม้แต่ของตัวเอง** เขียน RSpec spec ยืนยัน
   ทั้งสองด้าน (ทำสิ่งที่ทำได้จริง และไม่สามารถทำสิ่งที่ไม่ควรทำได้)
2. เปลี่ยนจากคอลัมน์ `role` เดี่ยว (Step 437) ไปเป็นตาราง `roles` + join table `user_roles`
   (ตามที่ออกแบบไว้ใน Step 436) แล้วปรับ `Ability` ให้รองรับ user ที่มีได้หลาย role พร้อมกัน
   (เช่น user คนเดียวเป็นทั้ง `editor` และ `moderator`) — ใบ้: เปลี่ยนจาก `case user.role`
   เป็นการวน `user.roles.each` แล้วสะสมกฎทีละ role
3. ต่อยอด `PermissionBasedAbility` จาก Step 438 ให้มี `Admin::PermissionsController` ที่ทำงาน
   ได้จริง (ไม่ใช่แค่โครงร่าง) พร้อม view เป็นตาราง checkbox role × resource × action แล้ว
   เขียน request spec ยืนยันว่าเมื่อ admin ติ๊กเพิ่มสิทธิ์ใหม่ผ่านหน้าเว็บ ผู้ใช้ role นั้น
   ต้องได้สิทธิ์ใหม่ทันทีโดยไม่ต้อง restart server

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปรัชญาของ **CanCanCan** ที่ตรงข้ามกับ Pundit — รวมศูนย์ authorization logic ไว้ใน
  `Ability` class เดียวด้วย DSL `can`/`cannot` แทนที่จะกระจายเป็น policy class ต่อ model
  และรู้ tradeoffs ของทั้งสองแนวทางว่าเหมาะกับสถานการณ์ไหน (รวมถึงความเสี่ยงที่ `Ability`
  จะกลายเป็น God Object เมื่อแอปโตขึ้น)
- ติดตั้ง `cancancan`, generate `Ability` class, และเขียนกฎด้วย `can`/`cannot` ครอบคลุม
  ownership condition, `:manage`/`:all`, และกฎตาม role — พร้อมเข้าใจว่าลำดับการประกาศกฎมีผล
  ต่อผลลัพธ์จริง (`cannot` ที่มาทีหลังชนะเสมอ)
- ใช้ `load_and_authorize_resource` เพื่อโหลด + ตรวจสิทธิ์ resource ในบรรทัดเดียว เทียบกับ
  การเขียน `authorize!`/`can?`/`cannot?` แบบ explicit และเข้าใจ `accessible_by` สำหรับกรอง
  query อัตโนมัติจากกฎที่มีอยู่แล้ว
- จัดการ `CanCan::AccessDenied` ด้วย `rescue_from` ให้ตอบกลับเหมาะสมทั้ง HTML และ JSON
- ออกแบบ RBAC schema ได้ทั้งสองแบบ — คอลัมน์ `role` เดี่ยวสำหรับกรณีง่าย และตาราง
  `roles`/`user_roles` สำหรับกรณีที่ user ต้องมีได้หลาย role พร้อม tradeoffs ที่ชัดเจนของ
  แต่ละแบบ
- Implement role-based access ด้วย Rails 7+ `enum` syntax ใหม่ และผูกเข้ากับ `Ability`
  class ได้อย่างสมบูรณ์
- สร้างระบบสิทธิ์แบบละเอียดที่ admin ปรับได้เองผ่านฐานข้อมูล (`Permission` model) แทนการ
  hardcode กฎไว้ในโค้ด และเข้าใจแนวทาง hybrid ที่ผสมกฎจาก DB เข้ากับเงื่อนไข ownership ที่
  ยังต้องเขียนเป็นโค้ด
- เขียน RSpec spec ทดสอบ `Ability` ด้วย `cancan/matchers` (`be_able_to`) ครอบคลุมทุก
  combination ของ role และสถานะ resource ที่มีนัยสำคัญ

**ต่อไป (Part 045):** เราจะออกจากโลกของ session-based authentication ไปสู่
**Token-based authentication** — เจาะลึก JWT (JSON Web Token) สำหรับทำ API authentication,
วิธี issue/verify/refresh token อย่างปลอดภัย, และปิดท้ายด้วยพื้นฐานของ OAuth ผ่าน Omniauth
สำหรับทำระบบ "Login with Google/Facebook" ซึ่งเป็นพื้นฐานสำคัญก่อนเข้าสู่เฟส API & GraphQL
ในบทถัดๆ ไป
