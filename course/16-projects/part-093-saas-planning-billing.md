# Part 093: Capstone 2 — SaaS Multi-Tenant App: Planning, Billing (Subscription)

> **Step ครอบคลุมใน Part นี้:** Step 921–930

**ระดับ:** Advanced | **Ruby:** 3.2+ | **Rails:** 7.1+ | **PostgreSQL:** 15+

ยินดีต้อนรับสู่ Capstone Project ที่ 2 — การสร้าง SaaS Application แบบ Multi-Tenant ที่สมบูรณ์แบบ! ใน Part นี้เราจะวางแผนสถาปัตยกรรม ออกแบบระบบ Billing ด้วย Stripe และสร้างรากฐานที่แข็งแกร่งสำหรับแอปพลิเคชัน SaaS ระดับ Production เราจะครอบคลุมตั้งแต่การกำหนด Feature Set และ Pricing Tier ไปจนถึงการ implement Multi-tenancy ด้วย Row-Level Security และการจัดการ Subscription ผ่าน Stripe Billing API อย่างครบถ้วน

---

## สารบัญ

- [Step 921: SaaS Product Spec](#step-921-saas-product-spec)
- [Step 922: Multi-tenancy Architecture Decision](#step-922-multi-tenancy-architecture-decision)
- [Step 923: Rails New SaaS App Setup](#step-923-rails-new-saas-app-setup)
- [Step 924: Current Tenant Pattern](#step-924-current-tenant-pattern)
- [Step 925: Row-Level Security](#step-925-row-level-security)
- [Step 926: Stripe Billing Integration](#step-926-stripe-billing-integration)
- [Step 927: Subscription Model](#step-927-subscription-model)
- [Step 928: Webhook Handler สำหรับ Stripe Events](#step-928-webhook-handler-สำหรับ-stripe-events)
- [Step 929: Billing Portal](#step-929-billing-portal)
- [Step 930: Feature Gating ด้วย Subscription Plan](#step-930-feature-gating-ด้วย-subscription-plan)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## Step 921: SaaS Product Spec

ก่อนเริ่มเขียนโค้ดบรรทัดแรก เราจำเป็นต้องมี Product Specification ที่ชัดเจน ขั้นตอนนี้จะพาคุณผ่านกระบวนการกำหนด Feature List, Pricing Tier และ User Stories สำหรับ SaaS Application ของเรา

ในตัวอย่างนี้เราจะสร้าง **ProjectFlow** — แอปพลิเคชัน Project Management สำหรับทีมงาน

### Feature List

```
ProjectFlow — SaaS Project Management Tool

Core Features:
├── Projects & Tasks
│   ├── สร้าง/แก้ไข/ลบ Projects
│   ├── Task management (CRUD)
│   ├── Task assignment ให้ team members
│   ├── Due dates & priorities
│   └── Task comments & attachments
│
├── Team Collaboration
│   ├── Invite team members ด้วย email
│   ├── Role-based access (Owner/Admin/Member)
│   ├── Real-time notifications
│   └── Activity feed
│
├── Reporting & Analytics
│   ├── Project progress dashboard
│   ├── Team productivity metrics
│   └── Time tracking
│
└── Integrations
    ├── Slack notifications
    ├── GitHub issues sync
    └── API access
```

### Pricing Tiers

```yaml
# pricing_tiers.yml

free:
  name: "Free"
  price_monthly: 0
  price_yearly: 0
  limits:
    users: 3
    projects: 5
    storage_gb: 1
    api_calls_per_month: 1000
  features:
    - basic_tasks
    - team_collaboration
    - email_notifications
  stripe_price_id: null

pro:
  name: "Pro"
  price_monthly: 29
  price_yearly: 290  # ~17% discount
  limits:
    users: 25
    projects: unlimited
    storage_gb: 50
    api_calls_per_month: 50000
  features:
    - basic_tasks
    - team_collaboration
    - email_notifications
    - advanced_reporting
    - time_tracking
    - api_access
    - priority_support
  stripe_price_id: "price_pro_monthly"

enterprise:
  name: "Enterprise"
  price_monthly: 99
  price_yearly: 990
  limits:
    users: unlimited
    projects: unlimited
    storage_gb: 500
    api_calls_per_month: unlimited
  features:
    - all_pro_features
    - sso_integration
    - audit_logs
    - custom_roles
    - dedicated_support
    - sla_guarantee
  stripe_price_id: "price_enterprise_monthly"
```

### User Stories

```markdown
# User Stories — ProjectFlow

## Epic 1: Account Management
- US-001: ในฐานะ Owner, ฉันต้องการสร้าง account ใหม่ เพื่อเริ่มใช้งาน ProjectFlow
- US-002: ในฐานะ Owner, ฉันต้องการเลือก pricing plan ที่เหมาะกับทีม
- US-003: ในฐานะ Owner, ฉันต้องการ invite สมาชิกเข้าร่วม account ของฉัน
- US-004: ในฐานะ Admin, ฉันต้องการจัดการ roles ของ members

## Epic 2: Project & Task Management
- US-005: ในฐานะ Member, ฉันต้องการสร้าง project ใหม่
- US-006: ในฐานะ Member, ฉันต้องการ assign task ให้ตัวเองหรือ members คนอื่น
- US-007: ในฐานะ Member, ฉันต้องการดู dashboard ที่แสดง tasks ของฉัน

## Epic 3: Billing & Subscription
- US-008: ในฐานะ Owner, ฉันต้องการ upgrade plan ผ่าน Stripe Checkout
- US-009: ในฐานะ Owner, ฉันต้องการดู billing history
- US-010: ในฐานะ Owner, ฉันต้องการ cancel subscription ได้ทุกเมื่อ
```

Product Spec ที่ดีช่วยให้เราตัดสินใจเรื่อง Architecture ได้ถูกต้อง ก่อนเริ่ม code ควรใช้เวลา review spec กับ stakeholders ให้ครบถ้วน

---

## Step 922: Multi-tenancy Architecture Decision

การตัดสินใจเรื่อง Multi-tenancy เป็นหนึ่งในการตัดสินใจที่สำคัญที่สุดสำหรับ SaaS Application มีหลายแนวทางให้เลือก แต่เราจะวิเคราะห์และเลือกแนวทางที่เหมาะสมที่สุด

### เปรียบเทียบ Multi-tenancy Approaches

```
Approach 1: Separate Databases per Tenant
─────────────────────────────────────────
장점:
  ✓ Data isolation สูงสุด
  ✓ ง่ายต่อการ backup/restore per tenant
  ✓ Performance isolation

ข้อเสีย:
  ✗ จัดการฐานข้อมูลหลาย instance ยาก
  ✗ Schema migration ซับซ้อน
  ✗ ค่าใช้จ่ายสูง (เหมาะกับ Enterprise-grade)

Approach 2: Separate Schemas per Tenant (PostgreSQL)
────────────────────────────────────────────────────
ข้อดี:
  ✓ Isolation ในระดับ schema
  ✓ ใช้ database เดียว
  ✓ ง่ายต่อ per-tenant customization

ข้อเสีย:
  ✗ Migration ซับซ้อน (ต้อง migrate ทุก schema)
  ✗ Connection pooling ยุ่งยาก

Approach 3: Row-Based Isolation (เราเลือกแนวนี้)
─────────────────────────────────────────────────
ข้อดี:
  ✓ ง่ายที่สุดในการ implement
  ✓ Migration ทำครั้งเดียว
  ✓ เหมาะกับ startup/early stage
  ✓ Gems รองรับดี (ActsAsTenant, etc.)

ข้อเสีย:
  ✗ ต้องระวัง data leakage ระหว่าง tenants
  ✗ Query performance ต้องมี index ที่ดี
```

### ออกแบบ Data Model สำหรับ Row-Based Isolation

```ruby
# ทุก table ที่เป็น tenant-specific จะมี account_id column

# accounts table — root tenant
accounts
  id, name, subdomain, plan, created_at

# users table — global (user อยู่ได้หลาย account)
users
  id, email, password_digest, name, created_at

# memberships table — join table ระหว่าง user กับ account
memberships
  id, user_id, account_id, role, created_at

# projects table — scoped by account_id
projects
  id, account_id, name, description, created_at

# tasks table — scoped by account_id (ผ่าน project)
tasks
  id, account_id, project_id, title, assignee_id, status, due_date
```

### Index Strategy สำหรับ Row-Based Isolation

```sql
-- ทุก tenant-scoped table ต้องมี composite index นี้
CREATE INDEX idx_projects_account ON projects(account_id);
CREATE INDEX idx_tasks_account ON tasks(account_id);
CREATE INDEX idx_tasks_account_project ON tasks(account_id, project_id);

-- Unique constraints ต้องรวม account_id ด้วย
CREATE UNIQUE INDEX idx_projects_name_per_account
  ON projects(account_id, name);
```

Row-based isolation เป็นแนวทางที่ได้รับความนิยมมากที่สุดสำหรับ SaaS startup เพราะ implement ได้เร็ว และสามารถ migrate ไป schema-based ได้ในภายหลังหากต้องการ

---

## Step 923: Rails New SaaS App Setup

เริ่มสร้าง Rails application ของเราด้วย PostgreSQL และ configure dependencies พื้นฐาน

### สร้าง Project ใหม่

```bash
# สร้าง Rails app ใหม่ด้วย PostgreSQL
rails new saas_app \
  --database postgresql \
  --css tailwind \
  --javascript importmap \
  --skip-test  # จะใช้ RSpec แทน

cd saas_app

# สร้าง database
rails db:create
```

### Gemfile Setup

```ruby
# Gemfile

source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.2.2"

gem "rails", "~> 7.1.0"
gem "pg", "~> 1.1"
gem "puma", ">= 5.0"
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "tailwindcss-rails"
gem "jbuilder"

# Authentication
gem "devise"
gem "devise-jwt"  # สำหรับ API authentication

# Multi-tenancy
gem "acts_as_tenant"

# Authorization
gem "pundit"

# Billing
gem "stripe", "~> 10.0"
gem "pay", "~> 7.0"  # Rails Stripe wrapper

# Background Jobs
gem "sidekiq"
gem "redis", ">= 4.0.1"

# Email
gem "mailgun-ruby"  # หรือ sendgrid

# Utilities
gem "pagy"           # Pagination
gem "ransack"        # Search & filter
gem "money-rails"    # Currency handling
gem "friendly_id"    # URL-friendly slugs

group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
  gem "pry-rails"
  gem "rubocop-rails"
end

group :development do
  gem "web-console"
  gem "letter_opener"  # Preview emails in browser
  gem "annotate"
end

group :test do
  gem "capybara"
  gem "selenium-webdriver"
  gem "webmock"
  gem "vcr"
  gem "shoulda-matchers"
  gem "stripe-ruby-mock", require: "stripe_mock"
end
```

### Account Model — Tenant Root

```bash
# Generate Account model
rails generate model Account \
  name:string \
  subdomain:string \
  plan:string \
  stripe_customer_id:string \
  created_at:datetime \
  updated_at:datetime
```

```ruby
# app/models/account.rb
class Account < ApplicationRecord
  # Associations
  has_many :memberships, dependent: :destroy
  has_many :users, through: :memberships
  has_many :projects, dependent: :destroy
  has_one :subscription, dependent: :destroy

  # Validations
  validates :name, presence: true, length: { minimum: 2, maximum: 100 }
  validates :subdomain,
    presence: true,
    uniqueness: { case_sensitive: false },
    format: { with: /\A[a-z0-9\-]+\z/, message: "ใช้ได้เฉพาะ lowercase, ตัวเลข และขีดกลาง" },
    length: { minimum: 3, maximum: 63 }

  # Slugify subdomain
  extend FriendlyId
  friendly_id :name, use: :slugged

  # Scopes
  scope :active, -> { joins(:subscription).where(subscriptions: { status: "active" }) }

  # Plan helpers
  PLANS = %w[free pro enterprise].freeze

  def free?
    plan == "free" || plan.nil?
  end

  def pro?
    plan == "pro"
  end

  def enterprise?
    plan == "enterprise"
  end

  def on_plan?(plan_name)
    plan == plan_name.to_s
  end

  def plan_limit(feature)
    PlanLimits.for(plan)[feature]
  end
end
```

```bash
# Generate migration
rails db:migrate

# Generate seed data
rails generate migration AddIndexesToAccounts
```

```ruby
# db/migrate/xxx_add_indexes_to_accounts.rb
class AddIndexesToAccounts < ActiveRecord::Migration[7.1]
  def change
    add_index :accounts, :subdomain, unique: true
    add_index :accounts, :stripe_customer_id
    add_index :accounts, :plan
  end
end
```

---

## Step 924: Current Tenant Pattern

`Current.account` pattern ช่วยให้เราเข้าถึง tenant ปัจจุบันได้จากทุกที่ใน request cycle โดยไม่ต้องส่ง account object ผ่านทุก method

### ActiveSupport::CurrentAttributes

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  # Attributes ที่จะถูก reset ทุก request
  attribute :account     # Account ปัจจุบัน (tenant)
  attribute :user        # User ที่ล็อกอินอยู่
  attribute :membership  # Membership ของ user ใน account นั้น

  # Delegate ไปยัง membership
  delegate :role, to: :membership, prefix: false, allow_nil: true

  def user=(user)
    super
    # Reset account เมื่อ user เปลี่ยน
    self.account = nil if user.nil?
  end

  def account=(account)
    super
    # ตั้งค่า ActsAsTenant ด้วย
    ActsAsTenant.current_tenant = account
  end

  # Helper methods
  def owner?
    membership&.owner?
  end

  def admin?
    membership&.admin? || owner?
  end

  def member?
    membership.present?
  end
end
```

### ApplicationController Setup

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  # ตั้งค่า tenant ก่อนทุก request
  before_action :authenticate_user!
  before_action :set_current_tenant
  before_action :set_current_user

  private

  def set_current_user
    Current.user = current_user
  end

  def set_current_tenant
    # Subdomain-based tenant detection
    account = find_account_from_request

    if account.nil?
      redirect_to root_url(subdomain: false), alert: "ไม่พบ account นี้"
      return
    end

    Current.account = account
    Current.membership = current_user&.membership_for(account)

    # ตรวจสอบว่า user เป็นสมาชิกของ account นี้
    unless Current.member? || skip_tenant_check?
      redirect_to dashboard_url, alert: "คุณไม่มีสิทธิ์เข้าถึง account นี้"
    end
  end

  def find_account_from_request
    if request.subdomain.present? && request.subdomain != "www"
      Account.find_by(subdomain: request.subdomain)
    elsif params[:account_id]
      Account.find_by(id: params[:account_id])
    end
  end

  def skip_tenant_check?
    false  # Override in controllers ที่ไม่ต้องการ tenant
  end

  # Pundit after_action
  after_action :verify_authorized, except: :index, unless: :devise_controller?
  after_action :verify_policy_scoped, only: :index, unless: :devise_controller?

  def pundit_user
    Current  # ส่ง Current object ให้ Pundit ใช้แทน user
  end
end
```

### Middleware สำหรับ Tenant Detection

```ruby
# app/middleware/tenant_detection_middleware.rb
class TenantDetectionMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)
    subdomain = request.subdomain

    if subdomain.present? && subdomain != "www"
      account = Account.find_by(subdomain: subdomain)
      env["current_account"] = account
    end

    @app.call(env)
  end
end

# config/application.rb
config.middleware.use TenantDetectionMiddleware
```

---

## Step 925: Row-Level Security

Row-Level Security ด้วย default scope ป้องกันการ query ข้ามข้อมูลระหว่าง tenants

### ActsAsTenant Gem Setup

```ruby
# config/initializers/acts_as_tenant.rb
ActsAsTenant.configure do |config|
  config.require_tenant = true  # Raise error ถ้าไม่มี tenant set
  config.pkey = :id
end
```

### Base Model ที่ใช้ ActsAsTenant

```ruby
# app/models/concerns/tenant_scoped.rb
module TenantScoped
  extend ActiveSupport::Concern

  included do
    # ผูก model เข้ากับ Account tenant
    acts_as_tenant :account

    # Validate ว่า account_id ตรงกับ Current.account
    validate :account_matches_current_tenant

    private

    def account_matches_current_tenant
      if ActsAsTenant.current_tenant && account_id != ActsAsTenant.current_tenant.id
        errors.add(:account, "ไม่ตรงกับ tenant ปัจจุบัน")
      end
    end
  end
end
```

### Project Model ที่ใช้ TenantScoped

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  include TenantScoped

  # Associations
  belongs_to :account
  has_many :tasks, dependent: :destroy
  has_many :memberships, through: :account

  # Validations
  validates :name, presence: true, uniqueness: { scope: :account_id }
  validates :name, length: { minimum: 2, maximum: 100 }

  # Scopes
  scope :active, -> { where(archived: false) }
  scope :archived, -> { where(archived: true) }
  scope :recent, -> { order(updated_at: :desc) }

  # ActsAsTenant จะ inject default_scope อัตโนมัติ
  # SELECT * FROM projects WHERE account_id = [current_tenant_id]
end
```

### ทดสอบ Row-Level Security

```ruby
# spec/models/project_spec.rb
RSpec.describe Project, type: :model do
  let(:account_a) { create(:account) }
  let(:account_b) { create(:account) }

  before do
    # สร้าง projects ใน 2 accounts
    ActsAsTenant.with_tenant(account_a) do
      create_list(:project, 3)
    end
    ActsAsTenant.with_tenant(account_b) do
      create_list(:project, 2)
    end
  end

  it "scopes queries to current tenant" do
    ActsAsTenant.with_tenant(account_a) do
      expect(Project.count).to eq(3)  # เห็นแค่ projects ของตัวเอง
    end

    ActsAsTenant.with_tenant(account_b) do
      expect(Project.count).to eq(2)  # เห็นแค่ projects ของตัวเอง
    end
  end

  it "prevents cross-tenant data access" do
    project_a = nil
    ActsAsTenant.with_tenant(account_a) do
      project_a = Project.first
    end

    ActsAsTenant.with_tenant(account_b) do
      expect { Project.find(project_a.id) }.to raise_error(ActiveRecord::RecordNotFound)
    end
  end
end
```

---

## Step 926: Stripe Billing Integration

การ integrate Stripe เพื่อจัดการ Payment, Subscription และ Billing สำหรับ SaaS Application

### Stripe Setup

```bash
# ติดตั้ง Stripe CLI สำหรับ development
brew install stripe/stripe-cli/stripe

# Login
stripe login

# Listen for webhooks ใน development
stripe listen --forward-to localhost:3000/webhooks/stripe
```

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.stripe[:secret_key]
Stripe.api_version = "2023-10-16"

# config/credentials.yml.enc (แก้ไขด้วย rails credentials:edit)
stripe:
  publishable_key: pk_test_xxx
  secret_key: sk_test_xxx
  webhook_secret: whsec_xxx
  product_ids:
    pro: prod_xxx
    enterprise: prod_yyy
  price_ids:
    pro_monthly: price_xxx
    pro_yearly: price_yyy
    enterprise_monthly: price_zzz
    enterprise_yearly: price_www
```

### Stripe Products และ Prices Setup

```ruby
# lib/tasks/stripe.rake
namespace :stripe do
  desc "Setup Stripe Products and Prices"
  task setup: :environment do
    # สร้าง Pro Product
    pro_product = Stripe::Product.create(
      name: "ProjectFlow Pro",
      description: "สำหรับทีมที่ต้องการ features ครบครัน",
      metadata: { plan: "pro" }
    )

    # Pro Monthly Price
    Stripe::Price.create(
      product: pro_product.id,
      nickname: "Pro Monthly",
      unit_amount: 2900,  # $29.00 ใน cents
      currency: "usd",
      recurring: {
        interval: "month",
        interval_count: 1
      },
      metadata: { plan: "pro", interval: "monthly" }
    )

    # Pro Yearly Price (17% discount)
    Stripe::Price.create(
      product: pro_product.id,
      nickname: "Pro Yearly",
      unit_amount: 29000,  # $290.00
      currency: "usd",
      recurring: {
        interval: "year",
        interval_count: 1
      },
      metadata: { plan: "pro", interval: "yearly" }
    )

    puts "Stripe setup เสร็จสิ้น!"
  end
end
```

### Stripe Checkout Session

```ruby
# app/services/stripe/checkout_session_creator.rb
module Stripe
  class CheckoutSessionCreator
    def initialize(account:, price_id:, user:)
      @account = account
      @price_id = price_id
      @user = user
    end

    def call
      ensure_stripe_customer!

      ::Stripe::Checkout::Session.create(
        customer: @account.stripe_customer_id,
        payment_method_types: ["card"],
        line_items: [{
          price: @price_id,
          quantity: 1
        }],
        mode: "subscription",
        allow_promotion_codes: true,
        subscription_data: {
          trial_period_days: 14,  # 14-day free trial
          metadata: {
            account_id: @account.id,
            plan: plan_from_price_id
          }
        },
        success_url: success_billing_url(@account),
        cancel_url: billing_url(@account),
        metadata: {
          account_id: @account.id
        }
      )
    rescue ::Stripe::StripeError => e
      Rails.logger.error "Stripe Checkout Error: #{e.message}"
      raise
    end

    private

    def ensure_stripe_customer!
      return if @account.stripe_customer_id.present?

      customer = ::Stripe::Customer.create(
        email: @user.email,
        name: @account.name,
        metadata: { account_id: @account.id }
      )

      @account.update!(stripe_customer_id: customer.id)
    end

    def plan_from_price_id
      price = ::Stripe::Price.retrieve(@price_id)
      price.metadata["plan"]
    end

    def success_billing_url(account)
      Rails.application.routes.url_helpers.success_billing_url(
        host: "#{account.subdomain}.#{Rails.application.config.app_domain}",
        session_id: "{CHECKOUT_SESSION_ID}"
      )
    end

    def billing_url(account)
      Rails.application.routes.url_helpers.billing_url(
        host: "#{account.subdomain}.#{Rails.application.config.app_domain}"
      )
    end
  end
end
```

---

## Step 927: Subscription Model

Subscription model จัดการ state ของ subscription ของแต่ละ account รวมถึง plan, status, trial period

```bash
rails generate model Subscription \
  account:references \
  stripe_subscription_id:string \
  stripe_customer_id:string \
  plan:string \
  status:string \
  current_period_start:datetime \
  current_period_end:datetime \
  trial_ends_at:datetime \
  cancel_at_period_end:boolean \
  canceled_at:datetime
```

```ruby
# app/models/subscription.rb
class Subscription < ApplicationRecord
  belongs_to :account

  # Stripe subscription statuses
  STATUSES = %w[
    trialing
    active
    past_due
    canceled
    unpaid
    incomplete
    incomplete_expired
    paused
  ].freeze

  validates :status, inclusion: { in: STATUSES }
  validates :stripe_subscription_id, presence: true, uniqueness: true

  # Scopes
  scope :active, -> { where(status: %w[active trialing]) }
  scope :trialing, -> { where(status: "trialing") }
  scope :past_due, -> { where(status: "past_due") }

  # Status helpers
  def active?
    status.in?(%w[active trialing])
  end

  def trialing?
    status == "trialing"
  end

  def past_due?
    status == "past_due"
  end

  def canceled?
    status == "canceled"
  end

  def trial_active?
    trialing? && trial_ends_at&.future?
  end

  def trial_days_remaining
    return 0 unless trial_active?
    (trial_ends_at.to_date - Date.today).to_i
  end

  def days_until_renewal
    return nil unless current_period_end
    (current_period_end.to_date - Date.today).to_i
  end

  # Sync จาก Stripe subscription object
  def self.sync_from_stripe(stripe_sub)
    subscription = find_or_initialize_by(stripe_subscription_id: stripe_sub.id)

    subscription.assign_attributes(
      stripe_customer_id: stripe_sub.customer,
      plan: stripe_sub.metadata["plan"] || extract_plan(stripe_sub),
      status: stripe_sub.status,
      current_period_start: Time.at(stripe_sub.current_period_start),
      current_period_end: Time.at(stripe_sub.current_period_end),
      trial_ends_at: stripe_sub.trial_end ? Time.at(stripe_sub.trial_end) : nil,
      cancel_at_period_end: stripe_sub.cancel_at_period_end,
      canceled_at: stripe_sub.canceled_at ? Time.at(stripe_sub.canceled_at) : nil
    )

    subscription.save!
    subscription
  end

  private

  def self.extract_plan(stripe_sub)
    # ดึง plan จาก price metadata
    price_id = stripe_sub.items.data.first&.price&.id
    return "free" unless price_id

    price = Stripe::Price.retrieve(price_id)
    price.metadata["plan"] || "unknown"
  rescue Stripe::StripeError
    "unknown"
  end
end
```

### Migration สำหรับ Subscription

```ruby
# db/migrate/xxx_create_subscriptions.rb
class CreateSubscriptions < ActiveRecord::Migration[7.1]
  def change
    create_table :subscriptions do |t|
      t.references :account, null: false, foreign_key: true
      t.string :stripe_subscription_id, null: false
      t.string :stripe_customer_id
      t.string :plan, default: "free"
      t.string :status, default: "trialing"
      t.datetime :current_period_start
      t.datetime :current_period_end
      t.datetime :trial_ends_at
      t.boolean :cancel_at_period_end, default: false
      t.datetime :canceled_at

      t.timestamps
    end

    add_index :subscriptions, :stripe_subscription_id, unique: true
    add_index :subscriptions, :stripe_customer_id
    add_index :subscriptions, [:account_id, :status]
    add_index :subscriptions, :current_period_end
  end
end
```

---

## Step 928: Webhook Handler สำหรับ Stripe Events

Webhook handler รับ events จาก Stripe และ update subscription status ใน database

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Stripe webhooks — ไม่ต้องการ CSRF protection
  post "/webhooks/stripe", to: "webhooks/stripe#create"

  # ... routes อื่นๆ
end
```

### Webhook Controller

```ruby
# app/controllers/webhooks/stripe_controller.rb
module Webhooks
  class StripeController < ActionController::Base
    # ปิด CSRF protection สำหรับ webhooks
    skip_before_action :verify_authenticity_token

    def create
      payload = request.body.read
      sig_header = request.env["HTTP_STRIPE_SIGNATURE"]
      webhook_secret = Rails.application.credentials.stripe[:webhook_secret]

      # Verify webhook signature
      event = Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)

      # Process event
      StripeWebhookProcessor.new(event).process

      render json: { received: true }, status: :ok

    rescue JSON::ParserError => e
      render json: { error: "Invalid payload" }, status: :bad_request
    rescue Stripe::SignatureVerificationError => e
      Rails.logger.error "Stripe webhook signature verification failed: #{e.message}"
      render json: { error: "Invalid signature" }, status: :bad_request
    rescue => e
      Rails.logger.error "Stripe webhook processing error: #{e.message}"
      render json: { error: "Processing failed" }, status: :internal_server_error
    end
  end
end
```

### Webhook Processor

```ruby
# app/services/stripe_webhook_processor.rb
class StripeWebhookProcessor
  HANDLED_EVENTS = %w[
    customer.subscription.created
    customer.subscription.updated
    customer.subscription.deleted
    customer.subscription.trial_will_end
    invoice.payment_succeeded
    invoice.payment_failed
    invoice.payment_action_required
    checkout.session.completed
  ].freeze

  def initialize(event)
    @event = event
  end

  def process
    return unless HANDLED_EVENTS.include?(@event.type)

    Rails.logger.info "Processing Stripe event: #{@event.type} (#{@event.id})"

    case @event.type
    when "customer.subscription.created", "customer.subscription.updated"
      handle_subscription_updated
    when "customer.subscription.deleted"
      handle_subscription_canceled
    when "customer.subscription.trial_will_end"
      handle_trial_ending
    when "invoice.payment_succeeded"
      handle_payment_succeeded
    when "invoice.payment_failed"
      handle_payment_failed
    when "invoice.payment_action_required"
      handle_payment_action_required
    when "checkout.session.completed"
      handle_checkout_completed
    end
  end

  private

  def handle_subscription_updated
    stripe_sub = @event.data.object
    account = find_account_by_stripe_customer(stripe_sub.customer)
    return unless account

    subscription = Subscription.sync_from_stripe(stripe_sub)
    account.update!(plan: subscription.plan)

    # ส่ง notification ถ้า plan เปลี่ยน
    AccountMailer.plan_changed(account, subscription).deliver_later if plan_changed?(subscription)

    Rails.logger.info "Subscription updated for account #{account.id}: #{subscription.plan} (#{subscription.status})"
  end

  def handle_subscription_canceled
    stripe_sub = @event.data.object
    account = find_account_by_stripe_customer(stripe_sub.customer)
    return unless account

    subscription = Subscription.find_by(stripe_subscription_id: stripe_sub.id)
    subscription&.update!(status: "canceled", canceled_at: Time.current)
    account.update!(plan: "free")

    AccountMailer.subscription_canceled(account).deliver_later
  end

  def handle_trial_ending
    stripe_sub = @event.data.object
    account = find_account_by_stripe_customer(stripe_sub.customer)
    return unless account

    trial_end = Time.at(stripe_sub.trial_end)
    AccountMailer.trial_ending_soon(account, trial_end).deliver_later
  end

  def handle_payment_failed
    invoice = @event.data.object
    account = find_account_by_stripe_customer(invoice.customer)
    return unless account

    subscription = Subscription.find_by(stripe_subscription_id: invoice.subscription)
    subscription&.update!(status: "past_due")

    AccountMailer.payment_failed(account, invoice).deliver_later
    Rails.logger.warn "Payment failed for account #{account.id}"
  end

  def handle_payment_succeeded
    invoice = @event.data.object
    account = find_account_by_stripe_customer(invoice.customer)
    return unless account

    subscription = Subscription.find_by(stripe_subscription_id: invoice.subscription)
    subscription&.update!(status: "active") if subscription&.past_due?

    Rails.logger.info "Payment succeeded for account #{account.id}"
  end

  def handle_checkout_completed
    session = @event.data.object
    account = Account.find_by(id: session.metadata["account_id"])
    return unless account

    # Subscription จะถูก create ผ่าน customer.subscription.created event
    Rails.logger.info "Checkout completed for account #{account.id}"
  end

  def find_account_by_stripe_customer(customer_id)
    Account.find_by(stripe_customer_id: customer_id)
  end

  def plan_changed?(subscription)
    subscription.saved_change_to_plan?
  end
end
```

---

## Step 929: Billing Portal

Stripe Customer Portal ช่วยให้ลูกค้าจัดการ subscription ของตัวเองได้โดยไม่ต้องสร้าง UI เอง

```ruby
# app/controllers/billing_controller.rb
class BillingController < ApplicationController
  before_action :require_owner!

  def show
    @subscription = Current.account.subscription
    @plans = PlanCatalog.all
    @current_usage = AccountUsageCalculator.new(Current.account).calculate
  end

  def new_checkout
    price_id = params[:price_id]

    unless valid_price_id?(price_id)
      redirect_to billing_path, alert: "Invalid plan selected"
      return
    end

    session = Stripe::CheckoutSessionCreator.new(
      account: Current.account,
      price_id: price_id,
      user: current_user
    ).call

    redirect_to session.url, allow_other_host: true
  rescue Stripe::StripeError => e
    redirect_to billing_path, alert: "เกิดข้อผิดพลาด: #{e.message}"
  end

  def portal
    # สร้าง Stripe Customer Portal session
    portal_session = Stripe::BillingPortal::Session.create(
      customer: Current.account.stripe_customer_id,
      return_url: billing_url
    )

    redirect_to portal_session.url, allow_other_host: true
  rescue Stripe::StripeError => e
    redirect_to billing_path, alert: "ไม่สามารถเปิด Billing Portal ได้: #{e.message}"
  end

  def success
    @session_id = params[:session_id]
    flash.now[:notice] = "ยินดีด้วย! Subscription ของคุณเริ่มต้นแล้ว"
  end

  private

  def require_owner!
    unless Current.owner?
      redirect_to root_path, alert: "เฉพาะ Owner เท่านั้นที่จัดการ Billing ได้"
    end
  end

  def valid_price_id?(price_id)
    valid_price_ids = Rails.application.credentials.stripe[:price_ids].values
    valid_price_ids.include?(price_id)
  end
end
```

### Billing View

```erb
<!-- app/views/billing/show.html.erb -->
<div class="max-w-4xl mx-auto px-4 py-8">
  <h1 class="text-2xl font-bold mb-6">Billing & Subscription</h1>

  <!-- Current Subscription Status -->
  <div class="bg-white rounded-lg shadow p-6 mb-6">
    <h2 class="text-lg font-semibold mb-4">แผนปัจจุบัน</h2>

    <% if @subscription&.active? %>
      <div class="flex items-center gap-4">
        <span class="px-3 py-1 bg-green-100 text-green-800 rounded-full text-sm font-medium">
          <%= @subscription.plan.capitalize %>
        </span>

        <% if @subscription.trialing? %>
          <span class="text-sm text-gray-600">
            Trial หมดอายุใน <%= @subscription.trial_days_remaining %> วัน
          </span>
        <% else %>
          <span class="text-sm text-gray-600">
            ต่ออายุในวันที่ <%= @subscription.current_period_end&.strftime("%d %B %Y") %>
          </span>
        <% end %>

        <%= link_to "จัดการ Subscription", portal_billing_path,
          method: :post,
          class: "ml-auto px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700" %>
      </div>

    <% else %>
      <p class="text-gray-600">คุณยังไม่มี subscription ที่ active</p>
    <% end %>
  </div>

  <!-- Pricing Plans -->
  <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
    <% @plans.each do |plan| %>
      <div class="bg-white rounded-lg shadow p-6 <%= 'ring-2 ring-blue-500' if plan.current?(Current.account) %>">
        <h3 class="text-lg font-semibold"><%= plan.name %></h3>
        <p class="text-3xl font-bold mt-2">
          $<%= plan.price_monthly %>
          <span class="text-sm text-gray-500 font-normal">/เดือน</span>
        </p>

        <ul class="mt-4 space-y-2">
          <% plan.features.each do |feature| %>
            <li class="flex items-center gap-2 text-sm">
              <svg class="w-4 h-4 text-green-500"><!-- checkmark --></svg>
              <%= feature %>
            </li>
          <% end %>
        </ul>

        <% unless plan.current?(Current.account) %>
          <%= button_to "เลือกแผนนี้",
            new_checkout_billing_path,
            params: { price_id: plan.stripe_price_id },
            class: "mt-6 w-full px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700" %>
        <% end %>
      </div>
    <% end %>
  </div>
</div>
```

---

## Step 930: Feature Gating ด้วย Subscription Plan

Feature Gating ป้องกันไม่ให้ผู้ใช้ที่มี plan ต่ำกว่าเข้าถึง features ที่ต้องการ plan สูงกว่า

### PlanLimits Configuration

```ruby
# config/plan_limits.yml
free:
  max_users: 3
  max_projects: 5
  storage_gb: 1
  api_calls_per_month: 1000
  features:
    - basic_tasks
    - team_collaboration

pro:
  max_users: 25
  max_projects: -1  # unlimited
  storage_gb: 50
  api_calls_per_month: 50000
  features:
    - basic_tasks
    - team_collaboration
    - advanced_reporting
    - time_tracking
    - api_access

enterprise:
  max_users: -1  # unlimited
  max_projects: -1
  storage_gb: 500
  api_calls_per_month: -1  # unlimited
  features:
    - basic_tasks
    - team_collaboration
    - advanced_reporting
    - time_tracking
    - api_access
    - sso_integration
    - audit_logs
    - custom_roles
```

### PlanFeature Concern

```ruby
# app/controllers/concerns/plan_feature.rb
module PlanFeature
  extend ActiveSupport::Concern

  included do
    helper_method :feature_available?, :plan_limit
  end

  # ตรวจสอบว่า feature นี้ available สำหรับ plan ปัจจุบัน
  def feature_available?(feature_name)
    PlanLimits.feature_available?(Current.account.plan, feature_name)
  end

  # ดึง limit สำหรับ feature
  def plan_limit(limit_name)
    PlanLimits.limit_for(Current.account.plan, limit_name)
  end

  # before_action helper — redirect ถ้า feature ไม่ available
  def require_feature!(feature_name, redirect_path: billing_path)
    unless feature_available?(feature_name)
      respond_to do |format|
        format.html do
          redirect_to redirect_path,
            alert: "Feature นี้ต้องการ plan ที่สูงกว่า กรุณา upgrade"
        end
        format.json do
          render json: {
            error: "Feature not available",
            required_plan: PlanLimits.minimum_plan_for(feature_name),
            upgrade_url: billing_url
          }, status: :payment_required
        end
      end
    end
  end

  # before_action helper — ตรวจสอบ quota
  def require_quota!(resource_name, count: nil)
    limit = plan_limit(resource_name)
    return if limit == -1  # unlimited

    current_count = count || current_resource_count(resource_name)

    if current_count >= limit
      respond_to do |format|
        format.html do
          redirect_to billing_path,
            alert: "คุณถึงขีดจำกัดแล้ว (#{current_count}/#{limit}) กรุณา upgrade เพื่อเพิ่มขีดจำกัด"
        end
        format.json do
          render json: {
            error: "Quota exceeded",
            limit: limit,
            current: current_count,
            upgrade_url: billing_url
          }, status: :payment_required
        end
      end
    end
  end
end
```

### ใช้งานใน Controllers

```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  include PlanFeature

  before_action -> { require_feature!(:advanced_reporting) }

  def index
    @reports = Report.for_account(Current.account).recent
  end
end

# app/controllers/projects_controller.rb
class ProjectsController < ApplicationController
  include PlanFeature

  before_action :check_project_quota, only: :create

  def create
    @project = Project.new(project_params)
    @project.account = Current.account

    if @project.save
      redirect_to @project, notice: "สร้าง Project สำเร็จ!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def check_project_quota
    project_count = Current.account.projects.count
    require_quota!(:max_projects, count: project_count)
  end
end
```

### PlanLimits Service Object

```ruby
# app/services/plan_limits.rb
class PlanLimits
  LIMITS = YAML.load_file(Rails.root.join("config/plan_limits.yml")).freeze

  def self.for(plan)
    LIMITS[plan.to_s] || LIMITS["free"]
  end

  def self.feature_available?(plan, feature)
    plan_config = for(plan)
    plan_config["features"].include?(feature.to_s)
  end

  def self.limit_for(plan, limit_name)
    plan_config = for(plan)
    plan_config[limit_name.to_s] || 0
  end

  def self.minimum_plan_for(feature)
    %w[free pro enterprise].find do |plan|
      feature_available?(plan, feature)
    end
  end

  def self.unlimited?(plan, limit_name)
    limit_for(plan, limit_name) == -1
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: เพิ่ม Usage Dashboard

สร้าง dashboard ที่แสดงการใช้งานปัจจุบันเทียบกับ quota ของ plan:

```ruby
# สร้าง AccountUsageCalculator service
class AccountUsageCalculator
  def initialize(account)
    @account = account
  end

  def calculate
    {
      users: {
        current: @account.memberships.active.count,
        limit: PlanLimits.limit_for(@account.plan, :max_users),
        percentage: calculate_percentage(:max_users, @account.memberships.active.count)
      },
      projects: {
        current: @account.projects.active.count,
        limit: PlanLimits.limit_for(@account.plan, :max_projects),
        percentage: calculate_percentage(:max_projects, @account.projects.active.count)
      }
    }
  end

  private

  def calculate_percentage(limit_name, current)
    limit = PlanLimits.limit_for(@account.plan, limit_name)
    return 0 if limit == -1  # unlimited
    [(current.to_f / limit * 100).round, 100].min
  end
end
```

### แบบฝึกหัดที่ 2: Implement Stripe Coupon

เพิ่ม support สำหรับ promotion code ใน checkout flow:

```ruby
# ใน CheckoutSessionCreator
discounts: params[:coupon_code].present? ? [{
  promotion_code: params[:coupon_code]
}] : []
```

### แบบฝึกหัดที่ 3: Dunning Management

สร้าง background job สำหรับจัดการ accounts ที่ payment failed:

```ruby
class DunningJob < ApplicationJob
  queue_as :billing

  def perform
    past_due_accounts = Account.joins(:subscription).where(subscriptions: { status: "past_due" })

    past_due_accounts.find_each do |account|
      subscription = account.subscription
      days_overdue = (Date.today - subscription.current_period_end.to_date).to_i

      case days_overdue
      when 1..3
        AccountMailer.payment_reminder(account, days_overdue).deliver_now
      when 4..7
        AccountMailer.payment_urgent(account, days_overdue).deliver_now
      when 8..
        # Suspend account
        account.update!(suspended: true)
        AccountMailer.account_suspended(account).deliver_now
      end
    end
  end
end
```

---

## สรุปสิ่งที่ได้เรียนรู้

ใน Part 093 นี้ เราได้เรียนรู้:

1. **SaaS Product Planning** — กระบวนการกำหนด Feature List, Pricing Tiers และ User Stories อย่างเป็นระบบก่อนเริ่มพัฒนา

2. **Multi-tenancy Architecture** — การเปรียบเทียบ 3 แนวทาง (Separate DB, Schema-based, Row-based) และเลือก Row-based isolation ที่เหมาะกับ startup

3. **Rails SaaS Setup** — การ configure Rails project สำหรับ multi-tenant app ด้วย gems ที่จำเป็น

4. **CurrentAttributes Pattern** — ใช้ `ActiveSupport::CurrentAttributes` สร้าง `Current.account` และ `Current.user` ที่เข้าถึงได้ทั้ง request cycle

5. **ActsAsTenant** — Row-level security อัตโนมัติด้วย default scope ที่ filter ทุก query ตาม tenant

6. **Stripe Billing** — Integration ครบวงจรตั้งแต่ Products, Prices ไปจนถึง Checkout Session

7. **Subscription Model** — การออกแบบ model ที่รองรับ Stripe subscription lifecycle รวมถึง trial, renewal และ cancellation

8. **Stripe Webhooks** — การรับและ process events จาก Stripe อย่างปลอดภัยด้วย signature verification

9. **Billing Portal** — ใช้ Stripe Customer Portal ให้ลูกค้าจัดการ subscription เองโดยไม่ต้องสร้าง UI

10. **Feature Gating** — ระบบควบคุมการเข้าถึง features ตาม subscription plan ด้วย `PlanFeature` concern

---

## ตัวอย่างโค้ดสำคัญที่ควรจำ

```ruby
# 1. Set current tenant ต้องทำก่อนทุก request
Current.account = Account.find_by(subdomain: request.subdomain)

# 2. ActsAsTenant inject default scope อัตโนมัติ
acts_as_tenant :account  # ใน model

# 3. Feature gating
before_action -> { require_feature!(:api_access) }

# 4. Quota checking
before_action -> { require_quota!(:max_projects) }, only: :create

# 5. Webhook signature verification
Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)
```

---

## Preview of Next Part

**Part 094: Capstone 2 — Team, Organization, Permission, Invite Flow**

ใน Part ถัดไป เราจะสร้างระบบ Team Management ที่ครบถ้วน:

- **Team Model** — Membership join table ด้วย roles: owner/admin/member
- **Pundit Policies** — Authorization แบบ multi-tenant ที่ปลอดภัย
- **Invite Flow** — ระบบ Invitation ด้วย secure token และ email delivery
- **Organization Settings** — หน้าจัดการสมาชิก roles และ permissions
- **Audit Log** — ติดตาม sensitive actions ทุกอย่าง
- **Account Switching** — UI สำหรับ user ที่อยู่หลาย accounts
- **API Keys** — API access per account พร้อม rate limiting
- **Onboarding Flow** — Welcome wizard สำหรับ new accounts
- **SaaS Metrics** — Dashboard แสดง MRR, churn rate และ active users

เตรียมพร้อมสำหรับการสร้าง Team Management ที่สมบูรณ์แบบใน Part 094!
