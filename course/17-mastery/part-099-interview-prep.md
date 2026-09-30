# Part 099: การเตรียมสัมภาษณ์งาน Ruby on Rails ระดับ Senior/Staff, Live Coding Practice

> **Step ครอบคลุมใน Part นี้:** Step 981–990

**ระดับ:** Senior/Mastery | **Rails version:** 7.x / 8.x | **Ruby version:** 3.2+

การสัมภาษณ์งาน Senior Rails Engineer แตกต่างจากระดับ Junior อย่างมาก ไม่ใช่แค่คำถามยากขึ้น แต่ต้องการให้คุณแสดงให้เห็นว่า "คิดอย่างไร" ไม่ใช่แค่ "รู้อะไร" Part นี้จะเตรียมคุณอย่างครอบคลุมตั้งแต่ technical questions ไปจนถึงการต่อรอง offer และการเริ่มต้นงานใหม่อย่างถูกต้อง

---

## สารบัญ

- [Step 981: สิ่งที่ Senior Rails Engineer ควรรู้](#step-981)
- [Step 982: Technical Screen ทั่วไป](#step-982)
- [Step 983: System Design Interview สำหรับ Rails](#step-983)
- [Step 984: Live Coding Practice](#step-984)
- [Step 985: Behavioral Questions](#step-985)
- [Step 986: Code Review Exercise](#step-986)
- [Step 987: Take-home Assignment Strategies](#step-987)
- [Step 988: Portfolio ที่โดดเด่น](#step-988)
- [Step 989: การต่อรอง Salary และ Offer Evaluation](#step-989)
- [Step 990: First 90 Days ในงานใหม่](#step-990)
- [แบบฝึกหัด](#exercises)
- [สรุปสิ่งที่ได้เรียนรู้](#summary)

---

## Step 981: สิ่งที่ Senior Rails Engineer ควรรู้ — ครอบคลุมหัวข้อจากหลักสูตรนี้ทั้งหมด {#step-981}

Senior Rails Engineer ไม่ใช่แค่คนที่ใช้ Rails มานาน แต่คือคนที่เข้าใจ "ทำไม" ไม่ใช่แค่ "อย่างไร"

### Competency Map สำหรับ Senior Rails Engineer

**Ruby Language Mastery**
```
□ ทำความเข้าใจ Ruby object model (modules, mixins, method lookup)
□ Metaprogramming (define_method, method_missing, respond_to_missing?)
□ Blocks, Procs, Lambda — ความแตกต่างและ use cases
□ Garbage collection และ memory management
□ Concurrency: Thread, Fiber, Ractors (Ruby 3+)
□ Performance profiling ด้วย ruby-prof, stackprof
```

**Rails Framework Deep Dive**
```
□ ActiveRecord: query optimization, N+1, eager loading
□ ActiveRecord: transactions, optimistic/pessimistic locking
□ ActionController: middleware stack, before/after actions
□ Routing: constraints, concerns, nested resources
□ ActiveJob: adapters, retries, dead letter queues
□ ActionCable: WebSocket, broadcasting, subscriptions
□ ActiveStorage: variants, direct upload, signed URLs
□ Hotwire: Turbo Frames, Turbo Streams, Stimulus
```

**Database**
```
□ SQL: complex JOINs, window functions, CTEs
□ PostgreSQL: indexes (B-tree, GIN, GiST), EXPLAIN ANALYZE
□ Database design: normalization, trade-offs
□ Migrations: zero-downtime migrations, reversible
□ Scaling: read replicas, connection pooling, partitioning
```

**Testing**
```
□ TDD/BDD: RSpec, Capybara, FactoryBot
□ Test doubles: mocks, stubs, spies
□ Performance testing: load testing (k6, Artillery)
□ Security testing: Brakeman, bundler-audit
```

**Security**
```
□ OWASP Top 10: SQL injection, XSS, CSRF
□ Authentication: Devise, JWT, OAuth2, WebAuthn
□ Authorization: Pundit, CanCan
□ Secret management: Rails credentials, Vault
```

**Ops & Deployment**
```
□ Docker + Docker Compose
□ Kamal (Rails native deployment)
□ CI/CD: GitHub Actions, CircleCI
□ Monitoring: New Relic, Datadog, Prometheus
□ Cloud: AWS, GCP, Heroku fundamentals
```

**Architecture & Design**
```
□ SOLID principles, Design patterns (GoF)
□ Service Objects, Form Objects, Query Objects
□ Event-driven architecture
□ API design: REST, GraphQL
□ Caching strategies
□ System design: scaling, trade-offs
```

---

## Step 982: Technical Screen ทั่วไป — Ruby trivia, Rails magic, database questions {#step-982}

Technical screen มักเป็น 60 นาทีกับ Engineer หรือ Engineering Manager เพื่อ filter candidates ก่อนเข้า onsite

### Ruby Questions ที่มักถูกถาม

**Q1: อธิบายความแตกต่างระหว่าง `include`, `extend`, และ `prepend`**

```ruby
module Greetable
  def greet
    "Hello, I am #{name}"
  end
end

# include: เพิ่ม instance methods
class Person
  include Greetable
  attr_reader :name
  def initialize(name); @name = name; end
end
Person.new("Alice").greet  # ✅ "Hello, I am Alice"

# extend: เพิ่ม class methods (หรือ singleton methods)
class Robot
  extend Greetable
  def self.name; "R2-D2"; end
end
Robot.greet  # ✅ "Hello, I am R2-D2"

# prepend: เพิ่มเข้าไปใน method lookup chain ก่อน class เอง
# ใช้สำหรับ monkey-patching ที่ยังเรียก original method ได้
module TimedGreeting
  def greet
    start = Time.now
    result = super  # เรียก Greetable#greet
    puts "Took #{Time.now - start}s"
    result
  end
end

class SmartPerson < Person
  prepend TimedGreeting  # TimedGreeting#greet เรียกก่อน Person#greet
end
```

**Q2: อธิบาย Frozen String Literal**

```ruby
# frozen_string_literal: true

# Ruby 3.x: string literals เป็น frozen โดย default
str = "hello"
str << " world"  # FrozenError: can't modify frozen String

# ทำไม? Performance — frozen strings share memory
# ไม่ต้อง allocate object ใหม่ทุกครั้ง

# ถ้าต้องการ mutable string
str = +"hello"  # String#+@ สร้าง mutable copy
str << " world"  # ✅
```

**Q3: Explain `yield_self` / `then` และ `tap`**

```ruby
# tap: ส่ง object เข้า block แล้ว return object เดิม
# ใช้สำหรับ debugging chain
User.new(name: "Alice")
    .tap { |u| puts "Before save: #{u.inspect}" }
    .save!
    .tap { |result| puts "Save result: #{result}" }

# then / yield_self: ส่ง object เข้า block แล้ว return ค่าจาก block
# ใช้สำหรับ pipeline
"alice@example.com"
  .then { |email| User.find_by(email: email) }
  .then { |user| user&.profile || GuestProfile.new }
  .then { |profile| ProfileSerializer.new(profile).as_json }
```

### Rails Questions ที่มักถูกถาม

**Q4: อธิบาย N+1 Query และวิธีแก้**

```ruby
# N+1 Problem
posts = Post.all          # Query 1: SELECT * FROM posts
posts.each do |post|
  puts post.author.name   # Query N: SELECT * FROM users WHERE id = ?
end
# ถ้ามี 100 posts = 101 queries!

# แก้ด้วย eager loading
posts = Post.includes(:author).all  # 2 queries เท่านั้น
posts.each { |post| puts post.author.name }

# หรือ joins (1 query)
posts = Post.joins(:author).select("posts.*, users.name as author_name")

# ตรวจจับ N+1 ด้วย Bullet gem
```

**Q5: Transaction ใน Rails**

```ruby
# Basic transaction
ActiveRecord::Base.transaction do
  order.update!(status: :processing)
  payment.create!(amount: order.total)
  inventory.decrement!(order.items)
  # ถ้า exception เกิดใน block นี้ → rollback ทั้งหมด
end

# Savepoints (nested transactions)
Order.transaction do
  order.save!
  
  Order.transaction(requires_new: true) do  # Savepoint
    send_notification(order)
    # ถ้า fail → rollback แค่ notification, ไม่ rollback order
  end
end

# ระวัง: ActiveRecord::Rollback ไม่ bubble up
Order.transaction do
  raise ActiveRecord::Rollback  # rollback โดยไม่ raise ออกไป
end
# ไม่มี exception ออกมา!

# ถ้าต้องการ handle:
begin
  Order.transaction do
    order.save! or raise ActiveRecord::Rollback
  end
rescue ActiveRecord::RecordInvalid => e
  # handle validation error
end
```

### Database Questions

**Q6: อธิบาย EXPLAIN ANALYZE**

```sql
-- ดูว่า PostgreSQL ทำอะไรกับ query นี้
EXPLAIN ANALYZE
SELECT orders.*, users.email
FROM orders
JOIN users ON users.id = orders.user_id
WHERE orders.status = 'pending'
  AND orders.created_at > '2024-01-01'
ORDER BY orders.created_at DESC
LIMIT 20;

-- อ่าน output:
-- Seq Scan: อ่านทั้งตาราง (ช้า ถ้าตารางใหญ่)
-- Index Scan: ใช้ index (เร็ว)
-- cost=0.00..8.27: estimated cost
-- rows=1: estimated rows
-- actual time=0.015..0.015: actual time
-- Rows Removed by Filter: rows ที่ถูก filter ออก
```

---

## Step 983: System Design Interview สำหรับ Rails — design URL shortener, design job board ด้วย Rails {#step-983}

System design interview วัดความสามารถในการคิดและออกแบบระบบ ไม่ใช่แค่ตอบ "ถูก/ผิด"

### Framework สำหรับ System Design

```
1. Clarify requirements (5 นาที)
   - Functional requirements: ระบบทำอะไรได้บ้าง?
   - Non-functional: scale เท่าไหร่? latency? availability?

2. Back-of-envelope estimation (5 นาที)
   - Users: 1M DAU
   - Read/Write ratio: 100:1
   - Storage: ?
   - Bandwidth: ?

3. High-level design (10 นาที)
   - Components หลัก
   - Data flow

4. Detailed design (20 นาที)
   - Database schema
   - API design
   - Key algorithms

5. Trade-offs discussion (10 นาที)
   - ทำไมถึงเลือก approach นี้?
   - อะไรคือ weaknesses?
   - ถ้า scale 10x จะทำอะไร?
```

### Design: URL Shortener ด้วย Rails

**Clarify Requirements:**
```
Functional:
- สร้าง short URL จาก long URL
- redirect short URL ไป long URL
- Custom alias (optional)
- Analytics: click count, geographic data

Non-functional:
- 100M URLs ที่สร้าง
- 10B redirects/day (~115,000 req/sec)
- Availability 99.9%
- Latency < 10ms สำหรับ redirect
```

**Database Schema:**
```ruby
# Migration
create_table :short_urls do |t|
  t.string :short_code, null: false, index: { unique: true }
  t.string :original_url, null: false
  t.bigint :user_id, index: true
  t.integer :click_count, default: 0
  t.timestamp :expires_at
  t.timestamps
end

# Index สำหรับ lookup
add_index :short_urls, :short_code, unique: true
```

**Short Code Generation:**
```ruby
class ShortUrl < ApplicationRecord
  CHARS = ("a".."z").to_a + ("A".."Z").to_a + ("0".."9").to_a
  
  before_create :generate_short_code
  
  private
  
  def generate_short_code
    loop do
      code = Array.new(7) { CHARS.sample }.join
      # Base62 encoding ของ sequential ID
      # 7 characters → 62^7 = 3.5 trillion URLs
      self.short_code = code
      break unless ShortUrl.exists?(short_code: code)
    end
  end
  
  # หรือใช้ bijection-based approach (deterministic, ไม่ต้อง DB check)
  def self.encode(id)
    result = ""
    while id > 0
      result = CHARS[id % 62] + result
      id /= 62
    end
    result.rjust(7, CHARS[0])
  end
end
```

**Redirect Controller ที่ optimized:**
```ruby
class RedirectsController < ApplicationController
  def show
    short_code = params[:short_code]
    
    # Cache ใน Redis เพื่อ skip DB
    original_url = Rails.cache.fetch("url:#{short_code}", expires_in: 24.hours) do
      ShortUrl.find_by!(short_code: short_code).original_url
    end
    
    # Track click แบบ async (ไม่ block redirect)
    TrackClickJob.perform_later(short_code, request.remote_ip)
    
    redirect_to original_url, status: :moved_permanently, allow_other_host: true
  rescue ActiveRecord::RecordNotFound
    render plain: "URL not found", status: :not_found
  end
end
```

### Design: Job Board ด้วย Rails

**Schema:**
```ruby
create_table :companies do |t|
  t.string :name, null: false
  t.string :logo_url
  t.text :description
  t.timestamps
end

create_table :job_postings do |t|
  t.references :company, null: false, foreign_key: true
  t.string :title, null: false
  t.text :description, null: false
  t.string :location
  t.boolean :remote_ok, default: false
  t.string :employment_type  # full_time, part_time, contract
  t.integer :salary_min
  t.integer :salary_max
  t.string :currency, default: "THB"
  t.string :experience_level  # junior, mid, senior, staff
  t.string :status, default: "draft"  # draft, published, closed
  t.timestamp :published_at
  t.timestamp :closes_at
  t.timestamps
  
  t.index [:status, :published_at]
  t.index [:company_id, :status]
end

create_table :tags do |t|
  t.string :name, null: false, index: { unique: true }
end

create_table :job_posting_tags, id: false do |t|
  t.references :job_posting, null: false
  t.references :tag, null: false
  t.index [:job_posting_id, :tag_id], unique: true
end

create_table :applications do |t|
  t.references :job_posting, null: false, foreign_key: true
  t.references :user, null: false, foreign_key: true
  t.string :status, default: "submitted"
  t.text :cover_letter
  t.string :resume_url
  t.timestamps
  
  t.index [:job_posting_id, :user_id], unique: true  # ป้องกัน duplicate
end
```

---

## Step 984: Live Coding Practice — FizzBuzz → Two Sum → ออกแบบ ActiveRecord model ที่ซับซ้อน {#step-984}

Live coding ต้องการทั้งความรู้และความสงบ ฝึกพูดความคิดออกมาดังๆ ขณะเขียนโค้ด

### Level 1: FizzBuzz (ทุกคนต้องทำได้)

```ruby
# โจทย์: print 1 ถึง 100
# หาร 3 ลงตัว → "Fizz"
# หาร 5 ลงตัว → "Buzz"  
# หาร 15 ลงตัว → "FizzBuzz"

# Basic solution
(1..100).each do |n|
  if n % 15 == 0
    puts "FizzBuzz"
  elsif n % 3 == 0
    puts "Fizz"
  elsif n % 5 == 0
    puts "Buzz"
  else
    puts n
  end
end

# Ruby-idiomatic solution
(1..100).map do |n|
  result = ""
  result += "Fizz" if n % 3 == 0
  result += "Buzz" if n % 5 == 0
  result.empty? ? n.to_s : result
end.each { |s| puts s }

# Elegant one-liner (อาจ over-engineer ใน interview)
(1..100).each { |n| puts ["FizzBuzz", nil, nil, "Fizz", nil, "Buzz", "Fizz", nil, nil, "Fizz", "Buzz", nil, "Fizz", nil, "FizzBuzz"][(n-1)%15] || n }
```

### Level 2: Two Sum

```ruby
# โจทย์: ให้ array ของตัวเลขและ target
# return indices ของสองตัวที่บวกกันได้เท่ากับ target

# Brute force O(n²) — พูดก่อนว่าจะปรับปรุง
def two_sum_naive(nums, target)
  nums.each_with_index do |a, i|
    nums.each_with_index do |b, j|
      next if i == j
      return [i, j] if a + b == target
    end
  end
end

# Optimal O(n) — ใช้ Hash
def two_sum(nums, target)
  seen = {}  # value → index
  
  nums.each_with_index do |num, i|
    complement = target - num
    
    if seen.key?(complement)
      return [seen[complement], i]
    end
    
    seen[num] = i
  end
  
  nil  # ไม่พบคำตอบ
end

# Test
two_sum([2, 7, 11, 15], 9)  # [0, 1] เพราะ 2+7=9
two_sum([3, 2, 4], 6)        # [1, 2] เพราะ 2+4=6
```

### Level 3: ออกแบบ ActiveRecord Model ที่ซับซ้อน

```ruby
# โจทย์: ออกแบบ model สำหรับ multi-level referral system
# User สามารถ refer user อื่น
# Track commission ผ่านหลายระดับ (up to 3 levels)

class User < ApplicationRecord
  belongs_to :referrer, class_name: "User", optional: true
  has_many :referrals, class_name: "User", foreign_key: :referrer_id
  
  # ทุก commission ที่ user นี้ได้รับ
  has_many :earned_commissions, class_name: "Commission", 
           foreign_key: :recipient_id
  
  # ทุก commission ที่ generate จากการ refer ของ user นี้
  has_many :generated_commissions, class_name: "Commission",
           foreign_key: :source_user_id
  
  def referral_chain
    chain = []
    current = self
    while current.referrer && chain.size < 3
      chain << current.referrer
      current = current.referrer
    end
    chain
  end
  
  def total_earned_commission
    earned_commissions.sum(:amount)
  end
end

class Commission < ApplicationRecord
  belongs_to :source_user, class_name: "User"
  belongs_to :recipient, class_name: "User"
  belongs_to :order
  
  enum level: { direct: 1, indirect_1: 2, indirect_2: 3 }
  
  COMMISSION_RATES = {
    direct: 0.05,      # 5%
    indirect_1: 0.02,  # 2%
    indirect_2: 0.01   # 1%
  }.freeze
  
  validates :amount, numericality: { greater_than: 0 }
end

# Service สำหรับ distribute commissions
class ReferralCommissionService
  def initialize(order)
    @order = order
    @buyer = order.user
  end
  
  def call
    referral_chain = @buyer.referral_chain
    
    ActiveRecord::Base.transaction do
      referral_chain.each_with_index do |referrer, index|
        level = Commission.levels.keys[index]
        rate = Commission::COMMISSION_RATES[level.to_sym]
        amount = @order.total * rate
        
        Commission.create!(
          source_user: @buyer,
          recipient: referrer,
          order: @order,
          level: level,
          amount: amount
        )
      end
    end
  end
end
```

### เทคนิคการ Live Code ที่ดี

```
1. ฟัง โจทย์ให้เข้าใจก่อน อย่าเริ่ม code ทันที
2. ถามคำถาม: "ถ้า array ว่างจะทำอย่างไร? ค่าซ้ำ?"
3. พูดดังๆ: "ผมจะเริ่มด้วย brute force ก่อน แล้วค่อย optimize"
4. เขียน test cases ก่อน code
5. ถ้าติด: "ผมติดตรงนี้ ขอเวลาคิดสักครู่" ดีกว่า silent นาน
6. อธิบาย time/space complexity ของ solution
```

---

## Step 985: Behavioral Questions — STAR method, leadership principle, conflict resolution {#step-985}

Senior ต้องตอบ behavioral questions ให้ดีเท่า technical questions

### STAR Method

```
S — Situation: บริบทของสถานการณ์
T — Task: หน้าที่หรือเป้าหมายของคุณ
A — Action: สิ่งที่คุณทำ (ส่วนนี้สำคัญที่สุด)
R — Result: ผลลัพธ์ที่วัดได้
```

**ตัวอย่าง: "Tell me about a time you had a technical disagreement"**

```
S: ทีมของเราต้องตัดสินใจว่าจะใช้ GraphQL หรือ REST API 
   สำหรับ mobile app ใหม่ ผมและ Tech Lead มีความเห็นต่างกัน

T: ผมเป็นคนเสนอ GraphQL และต้องทำให้ทีมเข้าใจว่าทำไม

A: ผมสร้าง proof of concept ทั้งสองแบบในสัปดาห์เดียว
   วัด latency, payload size, และ developer experience
   นำเสนอข้อมูลแบบ data-driven ให้ทีมตัดสินใจ
   ไม่ใช่แค่ "ผมคิดว่า..."
   
   Tech Lead ยังไม่เชื่อในเรื่อง GraphQL complexity
   ผมเสนอ compromise: ใช้ REST สำหรับ MVP
   แต่ออกแบบ API ให้ migration ไป GraphQL ในอนาคตได้ง่าย

R: ทีมตกลงตามแนวทางนี้ ส่ง MVP ตรงเวลา
   3 เดือนต่อมาเราเริ่ม migrate เป็น GraphQL ได้ราบรื่น
   เพราะ API ถูกออกแบบไว้ล่วงหน้า
```

### คำถาม Behavioral ที่ต้องเตรียม

```
1. "Tell me about a time you delivered a project under tight deadline"
2. "Describe a situation where you had to learn something new quickly"
3. "Tell me about a time you mentored a junior developer"
4. "How did you handle a production incident?"
5. "Describe a time you had to push back on a PM's request"
6. "What's the most technically complex thing you've built?"
7. "Tell me about a technical decision you made that you'd do differently"
```

**ข้อที่ 7 — ตอบอย่างไร?**

```
อย่ากลัวที่จะพูดถึงความผิดพลาด! Interviewer ต้องการเห็นว่า:
- คุณเรียนรู้จากประสบการณ์
- คุณ humble พอที่จะยอมรับว่าเคยผิดพลาด
- คุณมี judgment ที่ดีขึ้นตามเวลา

"ตอนนั้นผมเลือกใช้ MongoDB เพราะคิดว่า 'flexible schema' จะช่วยได้
แต่สุดท้ายพบว่า data มีความสัมพันธ์ซับซ้อน ซึ่ง relational DB จัดการได้ดีกว่า
ถ้าทำใหม่จะเริ่มด้วย PostgreSQL และ validate ว่า schema มีความ dynamic จริงหรือเปล่า"
```

---

## Step 986: Code Review Exercise ในการสัมภาษณ์ — อ่าน PR ที่มีบัก/ปัญหา แล้ว comment {#step-986}

บางบริษัทให้ review โค้ดในการสัมภาษณ์แทน live coding

### ตัวอย่าง Code Review Exercise

```ruby
# โค้ดที่ต้อง review:
class UsersController < ApplicationController
  def create
    @user = User.new(params[:user])
    
    if @user.save
      UserMailer.welcome_email(@user).deliver_now
      redirect_to root_path, notice: "Welcome!"
    else
      render :new
    end
  end
  
  def search
    @users = User.where("name LIKE '%#{params[:q]}%'")
    render json: @users
  end
  
  def update_role
    @user = User.find(params[:id])
    @user.update(role: params[:role])
    redirect_to admin_users_path
  end
  
  def export
    users = User.all
    csv = users.map { |u| [u.id, u.email, u.created_at].join(",") }.join("\n")
    send_data csv, filename: "users.csv"
  end
end
```

**Model Review Comments ที่ดี:**

```
🔴 [BLOCKER] Line 3: SQL Injection Vulnerability
`params[:user]` โดยตรงเป็น mass assignment vulnerability
ต้องใช้ strong parameters:

def user_params
  params.require(:user).permit(:name, :email, :password)
end

@user = User.new(user_params)

🔴 [BLOCKER] Line 14: SQL Injection ที่ชัดเจน
User.where("name LIKE '%#{params[:q]}%'")
ต้องใช้ parameterized query:
User.where("name LIKE ?", "%#{params[:q]}%")
หรือ User.where("name ILIKE :q", q: "%#{params[:q]}%")

🔴 [BLOCKER] Line 19: ไม่มี Authorization
ทุกคน (แม้ไม่ใช่ admin) สามารถเปลี่ยน role ได้
ต้องเพิ่ม before_action :require_admin! หรือ authorize @user

🟡 [SUGGESTION] Line 6-7: Email ใน transaction
deliver_now ทำใน HTTP request thread ซึ่งอาจ timeout
แนะนำใช้ deliver_later เพื่อส่งผ่าน background job

🟡 [SUGGESTION] Line 23-26: Memory issue
User.all โหลด users ทุกคนเข้า memory
ถ้ามี users มาก อาจ OOM
ควรใช้ find_each หรือ in_batches:

CSV.generate do |csv|
  User.find_each { |u| csv << [u.id, u.email, u.created_at] }
end

🔵 [NIT] Line 19: ไม่มี error handling
@user.update(role:...) อาจ fail
ควร check return value หรือใช้ update!
```

---

## Step 987: Take-home Assignment Strategies — การบริหารเวลา, README ที่ดี, trade-offs ที่ยอมรับได้ {#step-987}

Take-home assignment วัดหลายอย่างพร้อมกัน: คุณภาพโค้ด, การตัดสินใจ, และการสื่อสาร

### บริหารเวลาให้ดี

```
หลักการ: ทำให้เสร็จใน 60-70% ของเวลาที่กำหนด
เหลือเวลาสำหรับ:
- เขียน README
- Clean up code
- Self-review
- Tests เพิ่มเติม

ถ้า assignment 8 ชั่วโมง:
- 5-6 ชั่วโมง: implementation
- 1 ชั่วโมง: tests
- 1 ชั่วโมง: README + polish
```

### README ที่ดีสำหรับ Take-home

```markdown
# Job Board API

## Getting Started

\`\`\`bash
git clone ...
cd job-board
bundle install
cp .env.example .env
rails db:create db:migrate db:seed
rails server
\`\`\`

## Assumptions

Assignment ไม่ได้ระบุบางส่วน ผมตัดสินใจดังนี้:
- Authentication: ใช้ JWT แทน session เพราะ API-first
- Pagination: default 20 items/page ตาม common convention
- Search: PostgreSQL full-text search เพียงพอสำหรับ scope นี้

## What I Built

- [x] User registration & authentication (JWT)
- [x] Job posting CRUD (companies only)
- [x] Job application flow
- [x] Search jobs by keyword, location, remote
- [x] Application status tracking

## What I Would Add With More Time

- [ ] Elasticsearch สำหรับ search ที่ดีกว่า
- [ ] Email notifications ด้วย Action Mailer
- [ ] Rate limiting บน API endpoints
- [ ] Comprehensive error handling middleware

## Technical Decisions

### ทำไมใช้ Form Objects สำหรับ Job Posting?
โค้ด validation ซับซ้อนเกินกว่า model ธรรมดา จึงแยกออกเป็น
JobPostingForm เพื่อ separation of concerns และ testability

### ทำไมไม่ใช้ Elasticsearch?
Scope ของ assignment ไม่ได้กำหนด scale เป้าหมาย
PostgreSQL full-text search เพียงพอและลด complexity
ถ้า production จริงจะเพิ่ม Elasticsearch ทีหลัง

## API Documentation

### POST /api/auth/register
\`\`\`json
Request: { "user": { "email": "...", "password": "..." } }
Response 201: { "token": "...", "user": { "id": 1, "email": "..." } }
\`\`\`
```

### Trade-offs ที่ยอมรับได้

```
ระบุ trade-offs อย่างโปร่งใส ไม่ซ่อน

"ผมเลือก simple caching ด้วย Rails.cache แทน Redis cluster
เพราะ assignment ไม่ได้กำหนด scale requirement
ถ้า production จริง จะ migrate ไป Redis ได้ง่าย"

"Tests ครอบคลุม happy path และ critical edge cases
แต่ไม่ได้ test every single error case
เพราะ priority คือ demonstrate ว่าเข้าใจ TDD patterns
มากกว่าจะ test ทุกอย่าง 100%"
```

---

## Step 988: Portfolio ที่โดดเด่น — GitHub profile, live demo, technical blog, open source contribution {#step-988}

Portfolio แยกผู้สมัครที่เท่ากันออกจากกัน และบางครั้ง bypass technical screen ได้เลย

### GitHub Profile ที่ดี

```
✅ GitHub profile README ที่ informative
   - แนะนำตัวสั้นๆ
   - Stack ที่เชี่ยวชาญ
   - Projects ที่น่าสนใจ

✅ Pinned repositories: 4-6 projects ที่ดีที่สุด
✅ Contribution graph สม่ำเสมอ (ไม่ต้อง commit ทุกวัน)
✅ README ที่ดีทุก project
✅ Stars received ในงาน open source
```

### 3-5 Projects ที่ควรมีใน Portfolio

```
1. Full-stack Rails App ที่ deploy จริง
   - User authentication
   - Database with relationships
   - Background jobs
   - Tests
   - Live demo URL

2. API project ที่แสดง expertise
   - Authentication (JWT/OAuth)
   - Proper error handling
   - Documentation (Swagger/OpenAPI)
   - Postman collection

3. Open source contribution
   - PR ที่ merged ในโปรเจกต์ known
   - Ruby gem ของตัวเอง

4. Interesting side project
   - แสดง creativity
   - แก้ปัญหาจริง
   - ใช้ technology ที่น่าสนใจ

5. Technical blog
   - อธิบาย something complex ที่ทำ
   - Link จาก project README
```

### Live Demo สำคัญมาก

```bash
# Deploy ง่ายๆ ด้วย Render.com (free tier)
# หรือ Fly.io (free hobby plan)

# Fly.io deployment
fly launch
fly deploy
fly open  # เปิด browser

# config/environments/production.rb
config.force_ssl = true
config.log_level = :info
config.active_storage.service = :amazon  # S3 สำหรับ files
```

```markdown
# ใน README ต้องมี:
## Live Demo

🚀 **[View Live Demo](https://your-project.fly.dev)**

Test credentials:
- Email: demo@example.com
- Password: password123

Note: demo data resets every 24 hours
```

### Technical Blog

```
ไม่ต้องเขียนทุกวัน แค่สม่ำเสมอ

หัวข้อที่ดี:
- "How I debugged a mysterious N+1 query in production"
- "Building a multi-tenant Rails app: lessons learned"
- "Why I switched from Sidekiq to Solid Queue"
- "Real-world WebSocket with ActionCable and React"

Platform แนะนำ:
- dev.to (ฟรี, community ใหญ่, SEO ดี)
- personal site ด้วย Jekyll/Hugo (ควบคุมได้เต็ม)
- Medium (เข้าถึงได้กว้าง)
```

---

## Step 989: การต่อรอง Salary และ Offer Evaluation — market rate, total compensation, growth opportunity {#step-989}

หลายคนยอมรับ offer แรกโดยไม่ต่อรอง ซึ่งเป็นความผิดพลาดที่ส่งผลระยะยาว

### รู้ Market Rate ก่อน

```
แหล่งข้อมูล:
- levels.fyi: เปรียบเทียบ TC (Total Compensation) ของ tech companies
- Glassdoor: salary range บน job title
- LinkedIn Salary Insights
- ถาม peers ในชุมชน (Ruby Thailand, Discord groups)
- Recruiter ที่ติดต่อมา (ถามตรงๆ ได้)

ตัวอย่าง Range สำหรับ Senior Rails Engineer (2024, Thailand):
Junior (0-2 ปี): 40,000 - 70,000 THB/เดือน
Mid (2-5 ปี): 70,000 - 120,000 THB/เดือน
Senior (5+ ปี): 120,000 - 200,000 THB/เดือน
Staff/Principal: 200,000+ THB/เดือน

Remote positions (USD): Senior ~$80,000-$150,000/ปี
```

### Total Compensation ≠ Base Salary

```
Total Compensation รวม:
- Base Salary
- Annual Bonus (5-20% of base)
- Stock/RSU (สำหรับบางบริษัท)
- Health Insurance
- Provident Fund (บริษัทจ่ายสมทบ)
- Learning Budget
- Equipment
- Remote/Flexible work (value ที่ประเมินยาก)
- ประกันชีวิต

ตัวอย่าง: Offer A vs Offer B
A: 120,000 base + 10% bonus + equipment budget 30,000
= ~142,000 effective monthly value

B: 110,000 base + 20% bonus + stock options
= ~132,000 base cash + upside จาก stock
```

### วิธีต่อรอง

```
1. อย่าเป็นคนบอก number ก่อน
"ผมสนใจตำแหน่งนี้มาก ช่วย share range ของตำแหน่งนี้ได้ไหม?"

2. ถ้าถูกถามก่อน ให้ range ที่ bottom คือสิ่งที่ยอมรับได้
"Based on market rate และ experience ผมมอง 130,000-160,000 ครับ"

3. เมื่อได้ offer
"ขอบคุณสำหรับ offer นะครับ ผมสนใจมาก 
แต่ขอเวลาพิจารณา 48 ชั่วโมงได้ไหม?"
(อย่า accept ทันที เว้นแต่ perfect)

4. Counter offer
"ผมตื่นเต้นมากกับโอกาสนี้ และพร้อม join
แต่ขอ counter เล็กน้อย — ถ้าเป็น 140,000 
ผมพร้อม accept ทันทีเลยครับ"

5. Leverage multiple offers
"ผมมี offer อีกที่ที่ 145,000 แต่ prefer ที่นี่มากกว่า
ถ้าปรับได้ใกล้เคียง ผมตัดสินใจได้ทันที"
```

### Red Flags ใน Offer/Company

```
🚩 "เราทำงาน 60+ ชั่วโมง/สัปดาห์เป็นปกติ"
🚩 Trial period แต่จ่าย 50% เท่านั้น
🚩 ไม่มี equity/bonus แต่ยัง startup stage
🚩 Vague title เช่น "We're all flat here, no titles"
   (อาจหมายถึงไม่มี career progression)
🚩 "Family culture" บน job posting
   (อาจหมายถึงขาด work-life boundary)
🚩 Negative Glassdoor reviews จำนวนมากเกี่ยวกับ management
🚩 Turnover สูง — ถามว่า "ทีมนี้มีกี่คนลาออกในปีที่แล้ว?"
```

---

## Step 990: First 90 Days ในงานใหม่ — onboarding, build credibility, avoid common new-hire mistakes {#step-990}

วิธีที่คุณเริ่มต้นงานใหม่กำหนด reputation คุณในทีมสำหรับปีต่อๆ ไป

### Month 1: Listen and Learn (อย่าแก้อะไรทั้งนั้น!)

```
Week 1-2: Setup และ Orientation
- Setup development environment
- อ่าน CONTRIBUTING.md, ARCHITECTURE.md, ADRs
- พูดคุยกับทุกคนในทีม (1-on-1 coffee chat)
- เข้าใจ business domain ก่อน code

Week 3-4: First Contribution
- เลือก ticket เล็กๆ "good first issue"
- ทำความเข้าใจ code review process ของทีม
- เรียนรู้ deployment process
- ถามคำถาม บันทึกสิ่งที่ไม่ชัดเจน
```

### Month 2: Build Credibility

```
ทำสิ่งเหล่านี้ได้คะแนน:
✅ Fix bugs ที่ทีมหลีกเลี่ยงมานาน (แสดงความกล้า)
✅ เพิ่ม tests ให้โค้ดที่ไม่มี test
✅ Document สิ่งที่เรียนรู้ระหว่าง onboarding
✅ ช่วย review PR ของคนอื่น
✅ เสนอ improvement เล็กๆ ใน retrospective

สิ่งที่ต้องระวัง:
⚠️ อย่า compare กับงานเก่า ("เราทำแบบนี้ที่เก่า..."  เป็น toxic)
⚠️ อย่า refactor ใหญ่โดยไม่ขอ buy-in ก่อน
⚠️ อย่า suggest ว่า "ทุกอย่างต้องเปลี่ยน" — สร้างความไม่ไว้วางใจ
```

### Month 3: Start to Lead

```
เป้าหมาย Month 3:
- Own feature หนึ่งอย่างเต็มที่ตั้งแต่ design ถึง deploy
- Mentor developer คนอื่นในทีม
- มีส่วนร่วมใน technical decisions
- Set quarterly goals กับ manager

Framework: 30-60-90 Day Plan
เขียนเอกสารนี้ให้ manager ดูก่อนเริ่มงาน:
- 30 วัน: สิ่งที่จะ learn
- 60 วัน: สิ่งที่จะ contribute
- 90 วัน: สิ่งที่จะ lead/own
```

### Common New-hire Mistakes

```
❌ Over-promise ในสัปดาห์แรก
"ผมสามารถ refactor ระบบ authentication ทั้งหมดได้ใน 2 สัปดาห์!"
→ Under-deliver แล้วเสีย credibility

❌ Avoid asking questions เพราะกลัวดู "ไม่รู้"
→ Senior คนที่ถามคำถามฉลาดๆ ดูดีกว่าคนที่ assume แล้วทำผิด

❌ Skip 1-on-1s กับ manager
→ 1-on-1 คือโอกาสที่ดีที่สุดในการ align expectations

❌ ทำงานโดดเดี่ยว นานเกินไป
→ ถ้าติดเกิน 30 นาที ให้ ask for help
   ใน new job ให้ลดเป็น 15 นาที

❌ ไม่บันทึก onboarding experience
→ Notes ของคุณจะช่วย improve onboarding สำหรับ hire คนต่อไป
```

---

## แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Mock Technical Screen

ให้ partner ถามคำถาม 5 ข้อต่อไปนี้ และตอบออกมาดังๆ:
1. อธิบาย Ruby method lookup chain
2. อธิบายความแตกต่างระหว่าง `let` และ `let!` ใน RSpec
3. ทำไม `before_action` ใน ApplicationController ถึงอันตราย?
4. อธิบาย database deadlock และวิธีป้องกัน
5. อธิบาย `race condition` ใน background jobs และวิธีแก้

### แบบฝึกหัดที่ 2: System Design ใน 45 นาที

ออกแบบ "Twitter-like newsfeed" ด้วย Rails:
- Users follow other users
- Users post tweets
- Newsfeed แสดง tweets จากคนที่ follow
- ออกแบบ database schema
- ออกแบบ API endpoints
- พูดถึง scaling considerations

### แบบฝึกหัดที่ 3: Live Coding

แก้โจทย์ต่อไปนี้ใน 20 นาที พร้อม tests:
```
โจทย์: ให้ string ของ brackets "()[]{}"
return true ถ้า brackets ทั้งหมด valid (properly nested)

valid_brackets("()") → true
valid_brackets("()[]{}") → true
valid_brackets("(]") → false
valid_brackets("([)]") → false
valid_brackets("{[]}") → true
```

### แบบฝึกหัดที่ 4: สร้าง 90-Day Plan

สร้าง 30-60-90 Day Plan สำหรับ Rails Engineer ที่เพิ่ง join บริษัท e-commerce ขนาดกลาง

### แบบฝึกหัดที่ 5: Portfolio Review

Review GitHub profile ของตัวเอง:
- README ของทุก pinned repo ครบถ้วนไหม?
- มี live demo ไหม?
- Code ใน repo เก่าๆ ควร clean up ไหม?
- มี technical blog post ไหม?

---

## สรุปสิ่งที่ได้เรียนรู้ {#summary}

ใน Part 099 นี้เราได้เตรียมความพร้อมสำหรับการสัมภาษณ์งาน:

| หัวข้อ | สิ่งสำคัญที่ได้เรียน |
|--------|---------------------|
| Competency Map | Senior Rails Engineer ต้องรู้ Ruby, Rails, DB, Testing, Security, Ops, Architecture |
| Technical Screen | Ruby/Rails/DB questions ที่มักถูกถาม พร้อมคำตอบ |
| System Design | Framework 5 ขั้น: Clarify → Estimate → High-level → Detail → Trade-offs |
| Live Coding | พูดดังๆ ขณะเขียน, เริ่ม brute force แล้ว optimize, เขียน tests |
| Behavioral | STAR method, เตรียมตัวอย่างจาก experience จริง |
| Code Review Exercise | ระบุ security issues, performance, การใช้ severity labels |
| Take-home Assignment | README ที่ดี, document trade-offs, สมดุลระหว่าง completeness กับ quality |
| Portfolio | 3-5 projects ที่ diverse, live demo, technical blog |
| Salary Negotiation | รู้ market rate, negotiate บน total comp ไม่ใช่แค่ base |
| First 90 Days | Listen first, build credibility, avoid over-promising |

**Key Takeaway:** การสัมภาษณ์ Senior position ต้องการให้คุณแสดง "engineering judgment" ไม่ใช่แค่ความจำ โจทย์ที่ยากที่สุดในการสัมภาษณ์มักไม่มีคำตอบเดียวที่ถูก แต่ดูว่าคุณ navigate ความไม่แน่นอนได้อย่างไร

**ต่อไป:** Part 100 คือ Final Part ของหลักสูตร! เราจะสรุปทุกสิ่งที่ได้เรียนรู้ใน 1000 Steps พร้อม Roadmap ต่อไป และ Community Resources
