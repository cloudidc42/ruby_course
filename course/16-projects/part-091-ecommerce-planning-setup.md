# Part 091: Capstone 1 — E-Commerce Platform: Planning, ER Diagram, Setup

> **Step ครอบคลุมใน Part นี้:** Step 901–910

**ระดับ:** ขั้นสูง (ต้องผ่าน Phase 1–15 ทั้งหมด)
**Ruby on Rails เวอร์ชัน:** 7.1+
**Ruby เวอร์ชัน:** 3.2+
**เครื่องมือหลัก:** PostgreSQL, Devise, ActiveAdmin, Pagy, Stimulus, Active Storage, Tailwind CSS

---

## บทนำ

ยินดีต้อนรับสู่ **Capstone Project 1** — โปรเจกต์สุดยอดที่จะทดสอบทักษะทั้งหมดที่คุณได้เรียนมาตลอด Phase 1–15 ในคอร์สนี้ เราจะสร้าง **E-Commerce Platform** จริงตั้งแต่ต้นจนจบ ครอบคลุมตั้งแต่การวางแผนระบบ การออกแบบฐานข้อมูล ไปจนถึงการสร้าง UI ที่ใช้งานได้จริง

E-Commerce platform ที่เราจะสร้างในโปรเจกต์นี้จะรองรับ:
- **ผู้ซื้อ (Buyer)** — ค้นหาสินค้า เพิ่มลงตะกร้า ชำระเงิน ติดตามคำสั่งซื้อ เขียนรีวิว
- **ผู้ขาย (Seller)** — ลงสินค้า จัดการสต็อก ดูรายงานยอดขาย
- **ผู้ดูแลระบบ (Admin)** — จัดการ user ทั้งหมด อนุมัติสินค้า ดูภาพรวมระบบ

Part นี้จะครอบคลุมการวางแผน การออกแบบ ER Diagram และการ setup โปรเจกต์ให้พร้อมสำหรับการพัฒนาต่อใน Part 092

---

## สารบัญ

- [Step 901: Product Spec — Feature List และ User Stories](#step-901)
- [Step 902: ER Diagram — ออกแบบโครงสร้างฐานข้อมูล](#step-902)
- [Step 903: สร้างโปรเจกต์ Rails ใหม่](#step-903)
- [Step 904: Authentication ด้วย Devise และ User Roles](#step-904)
- [Step 905: Product Model พร้อม Active Storage](#step-905)
- [Step 906: Category Model แบบ Self-Referential](#step-906)
- [Step 907: Seed Data จริงด้วยข้อมูลภาษาไทย](#step-907)
- [Step 908: Admin Dashboard](#step-908)
- [Step 909: Product Listing Page พร้อม Filter และ Pagination](#step-909)
- [Step 910: Product Detail Page พร้อม Variants และ Stock](#step-910)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุป)
- [ตัวอย่างใน Part ถัดไป](#ถัดไป)

---

## Step 901: Product Spec ของ E-Commerce — Feature List และ User Stories {#step-901}

ก่อนที่จะเริ่มเขียนโค้ดแม้แต่บรรทัดเดียว ขั้นตอนที่สำคัญที่สุดคือการวางแผนและเขียน **Product Specification** ให้ชัดเจน

### ทำไม Product Spec ถึงสำคัญ

Product Spec ช่วยให้เราเข้าใจ "สิ่งที่ต้องสร้าง" ก่อนที่จะตัดสินใจว่า "จะสร้างอย่างไร" มันช่วยลดความเสี่ยงที่จะสร้างฟีเจอร์ผิด หรือลืมฟีเจอร์สำคัญที่จำเป็น

### Feature List ของ E-Commerce Platform

```
=== CORE FEATURES ===

[BUYER]
- ลงทะเบียน/เข้าสู่ระบบ (Devise)
- ค้นหาและกรองสินค้า (keyword, category, price range, rating)
- ดูรายละเอียดสินค้า (รูปภาพ, ตัวเลือก variant, stock)
- เพิ่มสินค้าลงตะกร้า (session + database cart)
- Checkout หลายขั้นตอน (ที่อยู่ → ชำระเงิน → ยืนยัน)
- ชำระเงินผ่าน Stripe
- ดูประวัติคำสั่งซื้อ
- ติดตามสถานะคำสั่งซื้อ
- เขียนรีวิวสินค้า (หลัง order สำเร็จ)

[SELLER]
- สมัครเป็น seller (upgrade role)
- เพิ่ม/แก้ไข/ลบสินค้า
- อัปโหลดรูปสินค้าหลายรูป
- จัดการ variants (size, color, stock)
- ดู orders ที่เกี่ยวข้องกับสินค้าของตัวเอง
- ดูรายงานยอดขาย (revenue, ยอดขายรายเดือน)

[ADMIN]
- จัดการ users ทั้งหมด (ban, promote)
- อนุมัติ/ปฏิเสธสินค้าใหม่
- จัดการ categories
- ดู dashboard ภาพรวม (GMV, จำนวน users, orders)
- จัดการ orders ทั้งหมด

=== TECHNICAL FEATURES ===
- Pagination (Pagy)
- Real-time cart update (Turbo Frames)
- Image optimization (Active Storage + variants)
- Background jobs (Sidekiq)
- Email notifications (Action Mailer)
- Search (PgSearch หรือ ransack)
```

### User Stories

User stories ช่วยให้เราคิดจากมุมมองของผู้ใช้ ตัวอย่าง:

```
=== BUYER USER STORIES ===

US-B001: ในฐานะผู้ซื้อ ฉันต้องการค้นหาสินค้าด้วยชื่อ
         เพื่อให้ฉันหาสินค้าที่ต้องการได้รวดเร็ว

US-B002: ในฐานะผู้ซื้อ ฉันต้องการกรองสินค้าตาม category
         เพื่อให้ฉันดูสินค้าในหมวดหมู่ที่สนใจได้

US-B003: ในฐานะผู้ซื้อ ฉันต้องการเพิ่มสินค้าลงตะกร้า
         โดยไม่ต้องล็อกอิน (guest cart)

US-B004: ในฐานะผู้ซื้อ ฉันต้องการชำระเงินด้วย credit card
         ผ่าน Stripe อย่างปลอดภัย

US-B005: ในฐานะผู้ซื้อ ฉันต้องการรับ email ยืนยันคำสั่งซื้อ
         ทันทีหลังจาก checkout สำเร็จ

=== SELLER USER STORIES ===

US-S001: ในฐานะผู้ขาย ฉันต้องการเพิ่มสินค้าพร้อมรูปหลายรูป
         เพื่อให้ผู้ซื้อเห็นสินค้าจากหลายมุม

US-S002: ในฐานะผู้ขาย ฉันต้องการตั้งค่า variants ของสินค้า (size/color)
         พร้อมราคาและ stock แยกต่างหาก

US-S003: ในฐานะผู้ขาย ฉันต้องการดูรายงานยอดขายรายเดือน
         เพื่อติดตามประสิทธิภาพธุรกิจ

=== ADMIN USER STORIES ===

US-A001: ในฐานะ admin ฉันต้องการดู dashboard
         แสดงสถิติสำคัญ เช่น GMV วันนี้ users ใหม่ orders รอดำเนินการ
```

### Priority Matrix

```
=== MoSCoW Priority ===

MUST HAVE (MVP):
✓ User auth (buyer/seller/admin)
✓ Product CRUD
✓ Category management
✓ Shopping cart
✓ Checkout + Stripe payment
✓ Order management
✓ Email notifications

SHOULD HAVE:
○ Product reviews
○ Seller dashboard with analytics
○ Advanced search/filter
○ Image optimization

COULD HAVE:
○ Wishlist
○ Coupon/discount system
○ Multi-currency
○ Product recommendations

WON'T HAVE (v1):
✗ Mobile app
✗ Live chat
✗ Affiliate system
```

---

## Step 902: ER Diagram — ออกแบบโครงสร้างฐานข้อมูล {#step-902}

ER Diagram (Entity Relationship Diagram) คือแผนผังของฐานข้อมูลที่แสดงตาราง (entities) และความสัมพันธ์ระหว่างตาราง (relationships)

### Entities หลักของระบบ

```
┌─────────────────────────────────────────────────────────────┐
│                    E-COMMERCE ER DIAGRAM                     │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│    users     │     │    categories    │     │    products     │
├──────────────┤     ├──────────────────┤     ├─────────────────┤
│ id           │     │ id               │     │ id              │
│ email        │     │ name             │     │ name            │
│ encrypted_pw │     │ slug             │     │ description     │
│ first_name   │     │ parent_id (FK)   │     │ price           │
│ last_name    │     │ position         │     │ stock_quantity  │
│ role (enum)  │     │ created_at       │     │ status (enum)   │
│ created_at   │     │ updated_at       │     │ user_id (FK)    │
│ updated_at   │     └──────────────────┘     │ category_id(FK) │
└──────────────┘              │               │ created_at      │
       │           self-ref   │               │ updated_at      │
       │           (parent)   │               └─────────────────┘
       │                      │                       │
       │             ┌────────┘                       │
       │             ▼                                │
       │      ┌──────────────┐              ┌─────────────────┐
       │      │  categories  │              │  product_images │
       │      │  (children)  │              ├─────────────────┤
       │      └──────────────┘              │ id              │
       │                                    │ product_id (FK) │
       │                                    │ (Active Storage)│
       │                                    └─────────────────┘
       │
       ├──────────────────────────────────────────────────────┐
       │                                                      │
       ▼                                                      ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│   orders     │     │  order_items │     │   cart_items     │
├──────────────┤     ├──────────────┤     ├──────────────────┤
│ id           │◄────│ id           │     │ id               │
│ user_id (FK) │     │ order_id(FK) │     │ user_id (FK)     │
│ status(enum) │     │ product_id   │     │ product_id (FK)  │
│ total_price  │     │ quantity     │     │ quantity         │
│ address_snap │     │ unit_price   │     │ created_at       │
│ created_at   │     │ created_at   │     │ updated_at       │
│ updated_at   │     └──────────────┘     └──────────────────┘
└──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   payments   │     │   reviews    │
├──────────────┤     ├──────────────┤
│ id           │     │ id           │
│ order_id(FK) │     │ user_id (FK) │
│ amount       │     │ product_id   │
│ stripe_id    │     │ rating (1-5) │
│ status(enum) │     │ body         │
│ paid_at      │     │ created_at   │
│ created_at   │     └──────────────┘
└──────────────┘
```

### Relationships สรุป

```
User         has_many :products (as seller)
User         has_many :orders (as buyer)
User         has_many :reviews
User         has_one  :cart (via cart_items)

Product      belongs_to :user (seller)
Product      belongs_to :category
Product      has_many_attached :images
Product      has_many :order_items
Product      has_many :reviews
Product      has_many :cart_items

Category     belongs_to :parent, class_name: "Category", optional: true
Category     has_many :children, class_name: "Category", foreign_key: "parent_id"
Category     has_many :products

Order        belongs_to :user
Order        has_many :order_items
Order        has_one  :payment

OrderItem    belongs_to :order
OrderItem    belongs_to :product

Payment      belongs_to :order

Review       belongs_to :user
Review       belongs_to :product

CartItem     belongs_to :user
CartItem     belongs_to :product
```

---

## Step 903: `rails new ecommerce` — Setup โปรเจกต์ {#step-903}

ได้เวลาลงมือสร้างโปรเจกต์จริงแล้ว!

### สร้างโปรเจกต์ Rails ใหม่

```bash
# สร้างโปรเจกต์พร้อม PostgreSQL และ Tailwind CSS
rails new ecommerce \
  --database=postgresql \
  --css=tailwind \
  --javascript=importmap

cd ecommerce
```

### ติดตั้ง Gems ที่จำเป็น

เปิดไฟล์ `Gemfile` และเพิ่ม gems ต่อไปนี้:

```ruby
# Gemfile

# Authentication
gem "devise"

# Pagination
gem "pagy"

# Image processing
gem "image_processing", "~> 1.2"

# Search
gem "ransack"

# Admin
gem "activeadmin"

# State machine
gem "aasm"

# Stripe payment
gem "stripe"

# Background jobs
gem "sidekiq"

group :development, :test do
  gem "factory_bot_rails"
  gem "faker"
  gem "rspec-rails"
end
```

```bash
bundle install
```

### ตั้งค่า Database

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: ecommerce_development

test:
  <<: *default
  database: ecommerce_test

production:
  <<: *default
  url: <%= ENV["DATABASE_URL"] %>
```

```bash
# สร้าง database
rails db:create
```

### ตั้งค่า Storage สำหรับรูปภาพ

```ruby
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# สำหรับ production ใช้ S3 หรือ GCS
amazon:
  service: S3
  access_key_id: <%= ENV["AWS_ACCESS_KEY_ID"] %>
  secret_access_key: <%= ENV["AWS_SECRET_ACCESS_KEY"] %>
  region: ap-southeast-1
  bucket: <%= ENV["AWS_BUCKET"] %>
```

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :amazon
```

### ตั้งค่า Tailwind

```bash
# สร้างไฟล์ layout หลัก
rails generate controller pages home
```

```html
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html lang="th">
  <head>
    <title>ร้านค้าออนไลน์</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "tailwind", "inter-font", "data-turbo-track": "reload" %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body class="bg-gray-50 min-h-screen">
    <%= render "shared/navbar" %>

    <main class="container mx-auto px-4 py-8">
      <%= render "shared/flash" %>
      <%= yield %>
    </main>

    <%= render "shared/footer" %>
  </body>
</html>
```

### ตั้งค่า Routes หลัก

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users

  root "pages#home"

  resources :products, only: [:index, :show]
  resources :categories, only: [:show]

  # Cart
  resource :cart, only: [:show]
  resources :cart_items, only: [:create, :update, :destroy]

  # Checkout และ Orders
  resources :orders, only: [:index, :show, :create]
  resources :checkouts, only: [:new, :create]

  # Stripe webhooks
  post "/webhooks/stripe", to: "webhooks#stripe"

  # Seller namespace
  namespace :seller do
    root "dashboard#index"
    resources :products
    resources :orders, only: [:index, :show]
    resources :reports, only: [:index]
  end

  # Admin namespace
  namespace :admin do
    root "dashboard#index"
    resources :users
    resources :products
    resources :categories
    resources :orders
  end
end
```

---

## Step 904: Authentication — Devise Setup และ User Roles {#step-904}

### ติดตั้ง Devise

```bash
rails generate devise:install
rails generate devise User
rails generate devise:views
```

### เพิ่ม Fields ให้กับ User

```bash
rails generate migration AddFieldsToUsers \
  first_name:string \
  last_name:string \
  role:integer \
  phone:string \
  avatar:string
```

```ruby
# db/migrate/XXXXXX_add_fields_to_users.rb
class AddFieldsToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :first_name, :string, null: false, default: ""
    add_column :users, :last_name, :string, null: false, default: ""
    add_column :users, :role, :integer, null: false, default: 0
    add_column :users, :phone, :string
    add_column :users, :bio, :text
    add_column :users, :shop_name, :string  # สำหรับ seller

    add_index :users, :role
  end
end
```

```bash
rails db:migrate
```

### User Model พร้อม Role Enum

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  # Role enum: 0=buyer, 1=seller, 2=admin
  enum :role, { buyer: 0, seller: 1, admin: 2 }

  # Associations
  has_many :products, foreign_key: :seller_id, dependent: :destroy
  has_many :orders, dependent: :nullify
  has_many :reviews, dependent: :destroy
  has_many :cart_items, dependent: :destroy

  has_one_attached :avatar

  # Validations
  validates :first_name, presence: true, length: { maximum: 50 }
  validates :last_name, presence: true, length: { maximum: 50 }
  validates :phone, format: { with: /\A0[0-9]{8,9}\z/, message: "ต้องเป็นเบอร์โทรศัพท์ไทย" },
            allow_blank: true

  # Scopes
  scope :active_sellers, -> { seller.where(approved_seller: true) }

  # Helper methods
  def full_name
    "#{first_name} #{last_name}"
  end

  def display_name
    shop_name.presence || full_name
  end

  def can_sell?
    seller? || admin?
  end
end
```

### ApplicationController พร้อม Authorization Helpers

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?

  # Helper methods สำหรับ authorization
  helper_method :current_buyer?, :current_seller?, :current_admin?

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:first_name, :last_name, :phone])
    devise_parameter_sanitizer.permit(:account_update, keys: [:first_name, :last_name, :phone, :bio, :shop_name, :avatar])
  end

  def require_buyer!
    unless current_user&.buyer? || current_user&.admin?
      redirect_to root_path, alert: "คุณไม่มีสิทธิ์เข้าถึงหน้านี้"
    end
  end

  def require_seller!
    unless current_user&.seller? || current_user&.admin?
      redirect_to root_path, alert: "กรุณาสมัครเป็นผู้ขายก่อน"
    end
  end

  def require_admin!
    unless current_user&.admin?
      redirect_to root_path, alert: "เฉพาะ Admin เท่านั้น"
    end
  end

  def current_buyer?
    current_user&.buyer?
  end

  def current_seller?
    current_user&.seller?
  end

  def current_admin?
    current_user&.admin?
  end
end
```

### Devise Views ที่ปรับแต่งเป็นภาษาไทย

```html
<!-- app/views/devise/sessions/new.html.erb -->
<div class="max-w-md mx-auto mt-10">
  <div class="bg-white rounded-lg shadow-md p-8">
    <h2 class="text-2xl font-bold text-center mb-6">เข้าสู่ระบบ</h2>

    <%= form_for(resource, as: resource_name, url: session_path(resource_name)) do |f| %>
      <div class="mb-4">
        <%= f.label :email, "อีเมล", class: "block text-sm font-medium text-gray-700 mb-1" %>
        <%= f.email_field :email,
              autofocus: true,
              autocomplete: "email",
              class: "w-full border rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" %>
      </div>

      <div class="mb-6">
        <%= f.label :password, "รหัสผ่าน", class: "block text-sm font-medium text-gray-700 mb-1" %>
        <%= f.password_field :password,
              autocomplete: "current-password",
              class: "w-full border rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" %>
      </div>

      <div class="mb-4">
        <%= f.submit "เข้าสู่ระบบ", class: "w-full bg-blue-600 text-white py-2 px-4 rounded-lg hover:bg-blue-700 transition" %>
      </div>
    <% end %>

    <p class="text-center text-sm text-gray-600">
      ยังไม่มีบัญชี?
      <%= link_to "สมัครสมาชิก", new_registration_path(resource_name), class: "text-blue-600 hover:underline" %>
    </p>
  </div>
</div>
```

---

## Step 905: Product Model — Active Storage และ Scopes {#step-905}

### สร้าง Product Model

```bash
rails generate model Product \
  name:string \
  description:text \
  price:decimal \
  stock_quantity:integer \
  status:integer \
  seller:references \
  category:references \
  slug:string
```

```ruby
# db/migrate/XXXXXX_create_products.rb
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string :name, null: false
      t.text :description
      t.decimal :price, precision: 10, scale: 2, null: false
      t.integer :stock_quantity, default: 0, null: false
      t.integer :status, default: 0, null: false  # 0=draft, 1=published, 2=archived
      t.references :seller, null: false, foreign_key: { to_table: :users }
      t.references :category, null: true, foreign_key: true
      t.string :slug, null: false
      t.integer :views_count, default: 0
      t.decimal :average_rating, precision: 3, scale: 2, default: 0.0

      t.timestamps
    end

    add_index :products, :slug, unique: true
    add_index :products, :status
    add_index :products, :price
    add_index :products, [:status, :category_id]
  end
end
```

```bash
rails db:migrate
```

### Product Model พร้อม Active Storage

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  # Associations
  belongs_to :seller, class_name: "User"
  belongs_to :category, optional: true
  has_many :order_items, dependent: :restrict_with_error
  has_many :reviews, dependent: :destroy
  has_many :cart_items, dependent: :destroy

  # Active Storage — รองรับรูปหลายรูป
  has_many_attached :images

  # Validations
  validates :name, presence: true, length: { maximum: 200 }
  validates :price, presence: true, numericality: { greater_than: 0 }
  validates :stock_quantity, numericality: { greater_than_or_equal_to: 0 }
  validates :slug, presence: true, uniqueness: true
  validate :images_presence_for_published

  # Enum
  enum :status, { draft: 0, published: 1, archived: 2 }

  # Callbacks
  before_validation :generate_slug, if: -> { slug.blank? && name.present? }

  # Scopes ที่จำเป็น
  scope :published, -> { where(status: :published) }
  scope :draft, -> { where(status: :draft) }
  scope :in_stock, -> { where("stock_quantity > 0") }
  scope :out_of_stock, -> { where(stock_quantity: 0) }
  scope :by_category, ->(category_id) { where(category_id: category_id) }
  scope :by_seller, ->(seller_id) { where(seller_id: seller_id) }
  scope :price_between, ->(min, max) { where(price: min..max) }
  scope :recently_added, -> { order(created_at: :desc) }
  scope :popular, -> { order(views_count: :desc) }
  scope :top_rated, -> { order(average_rating: :desc) }
  scope :search_by_name, ->(query) {
    where("name ILIKE :q OR description ILIKE :q", q: "%#{query}%")
  }

  # Instance methods
  def in_stock?
    stock_quantity > 0
  end

  def primary_image
    images.first
  end

  def update_average_rating!
    avg = reviews.average(:rating)&.round(2) || 0.0
    update_column(:average_rating, avg)
  end

  private

  def generate_slug
    base = name.parameterize
    slug_candidate = base
    counter = 1
    while Product.exists?(slug: slug_candidate)
      slug_candidate = "#{base}-#{counter}"
      counter += 1
    end
    self.slug = slug_candidate
  end

  def images_presence_for_published
    if published? && images.none?
      errors.add(:images, "ต้องอัปโหลดรูปสินค้าอย่างน้อย 1 รูป ก่อน publish")
    end
  end
end
```

### Image Variants สำหรับขนาดต่างๆ

```ruby
# config/initializers/active_storage_variants.rb
# กำหนด preset variants สำหรับใช้ซ้ำ

# ใช้ใน view:
# image_tag product.primary_image.variant(:thumb)
# image_tag product.primary_image.variant(:medium)
# image_tag product.primary_image.variant(:large)
```

```ruby
# app/models/product.rb - เพิ่ม method สำหรับ variant
def image_variant(image, size)
  dimensions = {
    thumb:  { resize_to_fill: [150, 150] },
    medium: { resize_to_fill: [400, 400] },
    large:  { resize_to_limit: [800, 800] }
  }
  image.variant(dimensions[size] || dimensions[:medium]).processed
end
```

---

## Step 906: Category Model — Self-Referential Association {#step-906}

Category ของ e-commerce มักมีโครงสร้างแบบ hierarchical เช่น:
- เสื้อผ้า → เสื้อผู้ชาย → เสื้อยืด
- อิเล็กทรอนิกส์ → โทรศัพท์ → iPhone

### สร้าง Category Model

```bash
rails generate model Category \
  name:string \
  slug:string \
  description:text \
  parent_id:integer \
  position:integer \
  icon:string
```

```ruby
# db/migrate/XXXXXX_create_categories.rb
class CreateCategories < ActiveRecord::Migration[7.1]
  def change
    create_table :categories do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      t.integer :parent_id
      t.integer :position, default: 0
      t.string :icon
      t.boolean :active, default: true

      t.timestamps
    end

    add_index :categories, :slug, unique: true
    add_index :categories, :parent_id
    add_index :categories, [:parent_id, :position]

    # Foreign key สำหรับ self-referential
    add_foreign_key :categories, :categories, column: :parent_id
  end
end
```

### Category Model

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  # Self-referential association
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children,
           class_name: "Category",
           foreign_key: :parent_id,
           dependent: :nullify,
           inverse_of: :parent

  has_many :products, dependent: :nullify

  # Validations
  validates :name, presence: true, uniqueness: { scope: :parent_id }
  validates :slug, presence: true, uniqueness: true

  # Callbacks
  before_validation :generate_slug

  # Scopes
  scope :root_categories, -> { where(parent_id: nil) }
  scope :subcategories_of, ->(parent) { where(parent_id: parent.id) }
  scope :active, -> { where(active: true) }
  scope :ordered, -> { order(:position, :name) }

  # Instance methods

  # คืน array ของ ancestors ตั้งแต่ root ถึง parent
  def ancestors
    result = []
    current = self.parent
    while current.present?
      result.unshift(current)
      current = current.parent
    end
    result
  end

  # คืน array [root, ..., parent, self]
  def breadcrumb
    ancestors + [self]
  end

  # ตรวจว่าเป็น root category หรือไม่
  def root?
    parent_id.nil?
  end

  # ตรวจว่ามี children หรือไม่
  def leaf?
    children.none?
  end

  # หา products ทั้งหมดรวม subcategories
  def all_products
    category_ids = [id] + all_descendant_ids
    Product.where(category_id: category_ids)
  end

  def all_descendant_ids
    children.flat_map { |child| [child.id] + child.all_descendant_ids }
  end

  def to_s
    name
  end

  private

  def generate_slug
    return if slug.present?
    self.slug = name.parameterize
  end
end
```

### Breadcrumb Helper

```ruby
# app/helpers/categories_helper.rb
module CategoriesHelper
  def breadcrumb_for(category)
    items = category.breadcrumb
    content_tag(:nav, aria: { label: "breadcrumb" }) do
      content_tag(:ol, class: "flex items-center space-x-2 text-sm") do
        items.map.with_index do |cat, i|
          content_tag(:li, class: "flex items-center") do
            if i < items.length - 1
              link_to(cat.name, category_path(cat), class: "text-blue-600 hover:underline") +
              content_tag(:span, " / ", class: "mx-2 text-gray-400")
            else
              content_tag(:span, cat.name, class: "text-gray-600 font-medium")
            end
          end
        end.join.html_safe
      end
    end
  end
end
```

---

## Step 907: Seed Data จริงด้วยข้อมูลภาษาไทย {#step-907}

Seed data ที่ดีช่วยให้เราทดสอบแอปพลิเคชันได้อย่างสมจริง

### ไฟล์ Seeds หลัก

```ruby
# db/seeds.rb
puts "🌱 เริ่ม Seed data..."

# สร้าง Admin user
admin = User.find_or_create_by!(email: "admin@ecommerce.th") do |u|
  u.password = "password123"
  u.first_name = "สมชาย"
  u.last_name = "ผู้ดูแล"
  u.role = :admin
end
puts "✓ Admin: #{admin.email}"

# สร้าง Seller users
sellers = [
  { email: "seller1@ecommerce.th", first_name: "สมหญิง", last_name: "ร้านค้า", shop_name: "ร้านผ้าไทย" },
  { email: "seller2@ecommerce.th", first_name: "วิชัย", last_name: "พ่อค้า", shop_name: "ร้านอิเล็กทรอนิกส์วิชัย" },
  { email: "seller3@ecommerce.th", first_name: "มาลี", last_name: "ขนม", shop_name: "ขนมไทยมาลี" }
]

sellers.each do |s|
  user = User.find_or_create_by!(email: s[:email]) do |u|
    u.password = "password123"
    u.first_name = s[:first_name]
    u.last_name = s[:last_name]
    u.shop_name = s[:shop_name]
    u.role = :seller
  end
  puts "✓ Seller: #{user.email}"
end

# สร้าง Buyer users
5.times do |i|
  User.find_or_create_by!(email: "buyer#{i+1}@ecommerce.th") do |u|
    u.password = "password123"
    u.first_name = ["สมศรี", "นิรันดร์", "อรุณ", "พรทิพย์", "ธนกร"][i]
    u.last_name = ["ใจดี", "มีสุข", "สว่าง", "งดงาม", "รวยทรัพย์"][i]
    u.role = :buyer
  end
end
puts "✓ Buyers: 5 คน"

# สร้าง Categories
categories_data = {
  "เสื้อผ้าและแฟชั่น" => ["เสื้อผู้ชาย", "เสื้อผู้หญิง", "กางเกง", "กระโปรง", "ชุดไทย"],
  "อิเล็กทรอนิกส์" => ["โทรศัพท์มือถือ", "แล็ปท็อป", "หูฟัง", "กล้องถ่ายรูป"],
  "อาหารและของกิน" => ["ขนมไทย", "อาหารแห้ง", "เครื่องดื่ม", "ผลไม้อบแห้ง"],
  "บ้านและสวน" => ["เฟอร์นิเจอร์", "ของตกแต่งบ้าน", "เครื่องครัว"],
  "กีฬาและกลางแจ้ง" => ["อุปกรณ์ฟิตเนส", "เสื้อผ้ากีฬา", "รองเท้ากีฬา"]
}

categories_map = {}
categories_data.each do |parent_name, children|
  parent = Category.find_or_create_by!(name: parent_name) do |c|
    c.position = 0
    c.active = true
  end
  categories_map[parent_name] = parent

  children.each_with_index do |child_name, i|
    child = Category.find_or_create_by!(name: child_name, parent_id: parent.id) do |c|
      c.position = i
      c.active = true
    end
    categories_map[child_name] = child
  end
end
puts "✓ Categories: #{Category.count} หมวดหมู่"

# สร้าง Products จริง
products_data = [
  {
    name: "เสื้อยืดลายช้างไทย พรีเมียม",
    description: "เสื้อยืดผ้า cotton 100% ลายช้างไทยดีไซน์สวยงาม เหมาะสำหรับของที่ระลึก มีให้เลือกหลายขนาด",
    price: 299.00,
    stock_quantity: 50,
    category: categories_map["เสื้อผู้ชาย"],
    status: :published
  },
  {
    name: "ผ้าไหมไทยแท้ ลายดอกไม้",
    description: "ผ้าไหมไทยทอมือจากเชียงใหม่ ลายดอกไม้ประดับประดา เหมาะสำหรับตัดชุดพิเศษ",
    price: 1_200.00,
    stock_quantity: 15,
    category: categories_map["ชุดไทย"],
    status: :published
  },
  {
    name: "หูฟังไร้สาย Hi-Fi รุ่น Pro",
    description: "หูฟังไร้สาย Bluetooth 5.0 เสียงคุณภาพสูง กันน้ำ IPX5 แบตเตอรี่ใช้งานได้ 30 ชั่วโมง",
    price: 2_990.00,
    stock_quantity: 30,
    category: categories_map["หูฟัง"],
    status: :published
  },
  {
    name: "ขนมทองหยิบโบราณ แม่เฮง",
    description: "ขนมทองหยิบทำมือตำรับโบราณ ใช้ไข่ไก่อินทรีย์ ไม่ใส่วัตถุกันเสีย บรรจุกล่องพรีเมียม",
    price: 189.00,
    stock_quantity: 100,
    category: categories_map["ขนมไทย"],
    status: :published
  },
  {
    name: "กระบะใส่ของตกแต่งบ้าน ไม้สัก",
    description: "กระบะไม้สักแท้ งานแฮนด์เมด ขัดเงา ขนาด 30x20 ซม. เหมาะตกแต่งโต๊ะและชั้นวาง",
    price: 450.00,
    stock_quantity: 25,
    category: categories_map["ของตกแต่งบ้าน"],
    status: :published
  }
]

seller = User.find_by!(email: "seller1@ecommerce.th")
products_data.each do |pd|
  Product.find_or_create_by!(name: pd[:name]) do |p|
    p.description = pd[:description]
    p.price = pd[:price]
    p.stock_quantity = pd[:stock_quantity]
    p.category = pd[:category]
    p.status = pd[:status]
    p.seller = seller
  end
end
puts "✓ Products: #{Product.count} รายการ"

puts "✅ Seed data เสร็จสิ้น!"
```

```bash
rails db:seed
```

---

## Step 908: Admin Dashboard {#step-908}

### Custom Admin Namespace

เราจะสร้าง admin namespace เองแทน ActiveAdmin เพื่อให้ customizable มากขึ้น:

```bash
mkdir -p app/controllers/admin
mkdir -p app/views/admin
rails generate controller Admin::Dashboard index
```

```ruby
# app/controllers/admin/base_controller.rb
class Admin::BaseController < ApplicationController
  before_action :authenticate_user!
  before_action :require_admin!

  layout "admin"
end
```

```ruby
# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < Admin::BaseController
  def index
    @stats = {
      total_users: User.count,
      new_users_today: User.where(created_at: Time.zone.today.all_day).count,
      total_products: Product.count,
      published_products: Product.published.count,
      total_orders: Order.count,
      pending_orders: Order.where(status: :pending).count,
      total_revenue: Order.where(status: :paid).sum(:total_price),
      today_revenue: Order.where(status: :paid, created_at: Time.zone.today.all_day).sum(:total_price)
    }

    @recent_orders = Order.includes(:user).order(created_at: :desc).limit(10)
    @top_products = Product.published.order(views_count: :desc).limit(5)
  end
end
```

```html
<!-- app/views/admin/dashboard/index.html.erb -->
<div class="space-y-6">
  <h1 class="text-2xl font-bold text-gray-800">Admin Dashboard</h1>

  <!-- Stats Grid -->
  <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">ผู้ใช้ทั้งหมด</p>
      <p class="text-3xl font-bold text-blue-600"><%= @stats[:total_users] %></p>
    </div>
    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">สินค้า published</p>
      <p class="text-3xl font-bold text-green-600"><%= @stats[:published_products] %></p>
    </div>
    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">Orders รอดำเนินการ</p>
      <p class="text-3xl font-bold text-yellow-600"><%= @stats[:pending_orders] %></p>
    </div>
    <div class="bg-white rounded-lg shadow p-4">
      <p class="text-sm text-gray-500">รายรับวันนี้</p>
      <p class="text-3xl font-bold text-purple-600">
        ฿<%= number_with_delimiter(@stats[:today_revenue].to_i) %>
      </p>
    </div>
  </div>

  <!-- Recent Orders -->
  <div class="bg-white rounded-lg shadow">
    <div class="p-4 border-b">
      <h2 class="font-semibold">Orders ล่าสุด</h2>
    </div>
    <table class="w-full">
      <thead class="bg-gray-50">
        <tr>
          <th class="px-4 py-2 text-left text-sm">Order #</th>
          <th class="px-4 py-2 text-left text-sm">ลูกค้า</th>
          <th class="px-4 py-2 text-left text-sm">ยอดรวม</th>
          <th class="px-4 py-2 text-left text-sm">สถานะ</th>
        </tr>
      </thead>
      <tbody>
        <% @recent_orders.each do |order| %>
          <tr class="border-t hover:bg-gray-50">
            <td class="px-4 py-2 text-sm">#<%= order.id %></td>
            <td class="px-4 py-2 text-sm"><%= order.user.full_name %></td>
            <td class="px-4 py-2 text-sm">฿<%= number_with_delimiter(order.total_price) %></td>
            <td class="px-4 py-2">
              <span class="px-2 py-1 rounded-full text-xs
                <%= order.paid? ? 'bg-green-100 text-green-800' : 'bg-yellow-100 text-yellow-800' %>">
                <%= order.status %>
              </span>
            </td>
          </tr>
        <% end %>
      </tbody>
    </table>
  </div>
</div>
```

---

## Step 909: Product Listing Page — Filter, Sort, Pagination {#step-909}

### ติดตั้ง Pagy

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Pagy::Backend
  # ...
end

# app/helpers/application_helper.rb
module ApplicationHelper
  include Pagy::Frontend
end
```

### Products Controller

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  def index
    @q = Product.published.ransack(params[:q])
    products = @q.result(distinct: true).includes(:seller, :category, images_attachments: :blob)

    # Apply additional filters
    products = products.by_category(params[:category_id]) if params[:category_id].present?
    products = products.price_between(params[:min_price], params[:max_price]) if params[:min_price].present? && params[:max_price].present?
    products = products.in_stock if params[:in_stock] == "1"

    # Sorting
    products = case params[:sort]
               when "price_asc"  then products.order(price: :asc)
               when "price_desc" then products.order(price: :desc)
               when "newest"     then products.order(created_at: :desc)
               when "popular"    then products.popular
               when "top_rated"  then products.top_rated
               else products.recently_added
               end

    @pagy, @products = pagy(products, limit: 20)
    @categories = Category.root_categories.active.ordered.includes(:children)
    @total_count = @pagy.count
  end

  def show
    @product = Product.published.find_by!(slug: params[:id])
    @product.increment!(:views_count)
    @reviews = @product.reviews.includes(:user).order(created_at: :desc)
    @related_products = Product.published
                                .by_category(@product.category_id)
                                .where.not(id: @product.id)
                                .limit(4)
  end
end
```

### Product Listing View พร้อม Stimulus Filter

```javascript
// app/javascript/controllers/product_filter_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["form", "results"]

  connect() {
    // Auto-submit on filter change
    this.formTarget.querySelectorAll("select, input[type=checkbox]").forEach(el => {
      el.addEventListener("change", () => this.submit())
    })
  }

  submit() {
    this.formTarget.requestSubmit()
  }

  clearFilters() {
    this.formTarget.reset()
    this.submit()
  }
}
```

```html
<!-- app/views/products/index.html.erb -->
<div class="flex gap-6" data-controller="product-filter">

  <!-- Sidebar Filter -->
  <aside class="w-64 shrink-0">
    <%= form_with url: products_path,
                  method: :get,
                  data: { product_filter_target: "form", turbo_frame: "products_list" },
                  class: "space-y-6" do |f| %>

      <!-- Category Filter -->
      <div class="bg-white rounded-lg shadow p-4">
        <h3 class="font-semibold mb-3">หมวดหมู่</h3>
        <% @categories.each do |cat| %>
          <label class="flex items-center mb-2 cursor-pointer">
            <%= radio_button_tag :category_id, cat.id,
                params[:category_id] == cat.id.to_s,
                class: "mr-2" %>
            <span class="text-sm"><%= cat.name %></span>
          </label>
        <% end %>
      </div>

      <!-- Price Range -->
      <div class="bg-white rounded-lg shadow p-4">
        <h3 class="font-semibold mb-3">ช่วงราคา</h3>
        <div class="flex gap-2">
          <%= number_field_tag :min_price, params[:min_price],
                placeholder: "ต่ำสุด",
                class: "w-full border rounded px-2 py-1 text-sm" %>
          <span class="text-gray-400">—</span>
          <%= number_field_tag :max_price, params[:max_price],
                placeholder: "สูงสุด",
                class: "w-full border rounded px-2 py-1 text-sm" %>
        </div>
        <%= submit_tag "ค้นหา", class: "mt-2 w-full bg-blue-600 text-white py-1 rounded text-sm" %>
      </div>

      <!-- In Stock Only -->
      <div class="bg-white rounded-lg shadow p-4">
        <label class="flex items-center cursor-pointer">
          <%= check_box_tag :in_stock, "1", params[:in_stock] == "1", class: "mr-2" %>
          <span class="text-sm">มีสินค้าในสต็อก</span>
        </label>
      </div>
    <% end %>
  </aside>

  <!-- Product Grid -->
  <div class="flex-1">
    <!-- Sort & Count -->
    <div class="flex justify-between items-center mb-4">
      <p class="text-sm text-gray-600">พบ <%= @total_count %> รายการ</p>
      <%= select_tag :sort,
            options_for_select([
              ["ล่าสุด", "newest"],
              ["ราคาต่ำ-สูง", "price_asc"],
              ["ราคาสูง-ต่ำ", "price_desc"],
              ["ยอดนิยม", "popular"],
              ["คะแนนสูง", "top_rated"]
            ], params[:sort]),
            class: "border rounded px-3 py-1 text-sm" %>
    </div>

    <%= turbo_frame_tag "products_list" do %>
      <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        <% @products.each do |product| %>
          <%= render "products/card", product: product %>
        <% end %>
      </div>

      <!-- Pagination -->
      <div class="mt-6 flex justify-center">
        <%== pagy_nav(@pagy) %>
      </div>
    <% end %>
  </div>
</div>
```

```html
<!-- app/views/products/_card.html.erb -->
<div class="bg-white rounded-lg shadow hover:shadow-md transition overflow-hidden">
  <%= link_to product_path(product) do %>
    <% if product.images.attached? %>
      <%= image_tag product.primary_image.variant(resize_to_fill: [300, 300]),
            class: "w-full h-48 object-cover",
            loading: "lazy" %>
    <% else %>
      <div class="w-full h-48 bg-gray-200 flex items-center justify-center">
        <span class="text-gray-400 text-sm">ไม่มีรูปภาพ</span>
      </div>
    <% end %>
  <% end %>

  <div class="p-3">
    <h3 class="text-sm font-medium line-clamp-2 mb-1">
      <%= link_to product.name, product_path(product), class: "hover:text-blue-600" %>
    </h3>

    <div class="flex items-center justify-between">
      <span class="text-lg font-bold text-red-600">
        ฿<%= number_with_delimiter(product.price.to_i) %>
      </span>
      <% unless product.in_stock? %>
        <span class="text-xs text-gray-400 bg-gray-100 px-2 py-1 rounded">หมด</span>
      <% end %>
    </div>

    <div class="text-xs text-gray-500 mt-1">
      ⭐ <%= product.average_rating %> · ร้าน <%= product.seller.display_name %>
    </div>
  </div>
</div>
```

---

## Step 910: Product Detail Page — รูปหลายรูป, Variants, Stock {#step-910}

### Product Detail View

```html
<!-- app/views/products/show.html.erb -->
<div class="max-w-6xl mx-auto">
  <!-- Breadcrumb -->
  <% if @product.category %>
    <nav class="mb-4 text-sm text-gray-500">
      <%= link_to "หน้าแรก", root_path, class: "hover:text-blue-600" %>
      <% @product.category.breadcrumb.each do |cat| %>
        <span class="mx-1">/</span>
        <%= link_to cat.name, category_path(cat), class: "hover:text-blue-600" %>
      <% end %>
    </nav>
  <% end %>

  <div class="grid grid-cols-1 md:grid-cols-2 gap-8 bg-white rounded-lg shadow p-6">

    <!-- รูปภาพ -->
    <div class="space-y-3">
      <!-- รูปหลัก -->
      <div class="aspect-square bg-gray-100 rounded-lg overflow-hidden" id="main-image">
        <% if @product.images.attached? %>
          <%= image_tag @product.primary_image.variant(resize_to_limit: [600, 600]),
                id: "main-product-image",
                class: "w-full h-full object-contain" %>
        <% end %>
      </div>

      <!-- Thumbnails -->
      <% if @product.images.count > 1 %>
        <div class="flex gap-2 overflow-x-auto">
          <% @product.images.each_with_index do |img, i| %>
            <%= image_tag img.variant(resize_to_fill: [100, 100]),
                  class: "w-20 h-20 object-cover rounded cursor-pointer border-2 border-transparent hover:border-blue-500 #{i == 0 ? 'border-blue-500' : ''}",
                  onclick: "document.getElementById('main-product-image').src = '#{url_for(img.variant(resize_to_limit: [600, 600]))}'",
                  loading: "lazy" %>
          <% end %>
        </div>
      <% end %>
    </div>

    <!-- รายละเอียดสินค้า -->
    <div class="space-y-4">
      <h1 class="text-2xl font-bold text-gray-900"><%= @product.name %></h1>

      <!-- Rating -->
      <div class="flex items-center gap-2">
        <div class="flex text-yellow-400">
          <% 5.times do |i| %>
            <span><%= i < @product.average_rating ? "★" : "☆" %></span>
          <% end %>
        </div>
        <span class="text-sm text-gray-500">(<%= @reviews.count %> รีวิว)</span>
        <span class="text-sm text-gray-400">|</span>
        <span class="text-sm text-gray-500">ขายแล้ว <%= @product.order_items.sum(:quantity) %> ชิ้น</span>
      </div>

      <!-- ราคา -->
      <div class="text-3xl font-bold text-red-600">
        ฿<%= number_with_delimiter(@product.price)%>
      </div>

      <!-- Stock Indicator -->
      <div class="flex items-center gap-2">
        <% if @product.in_stock? %>
          <span class="inline-flex items-center px-3 py-1 rounded-full text-sm bg-green-100 text-green-800">
            ✓ มีสินค้า (<%= @product.stock_quantity %> ชิ้น)
          </span>
        <% else %>
          <span class="inline-flex items-center px-3 py-1 rounded-full text-sm bg-red-100 text-red-800">
            ✗ สินค้าหมด
          </span>
        <% end %>
      </div>

      <!-- Add to Cart -->
      <% if @product.in_stock? %>
        <%= form_with url: cart_items_path, method: :post, class: "flex gap-3" do |f| %>
          <%= f.hidden_field :product_id, value: @product.id %>
          <div class="flex items-center border rounded-lg overflow-hidden">
            <button type="button" onclick="this.nextElementSibling.value = Math.max(1, parseInt(this.nextElementSibling.value) - 1)"
                    class="px-3 py-2 bg-gray-100 hover:bg-gray-200">−</button>
            <%= f.number_field :quantity, value: 1, min: 1, max: @product.stock_quantity,
                  class: "w-16 text-center py-2 border-x" %>
            <button type="button" onclick="this.previousElementSibling.value = Math.min(<%= @product.stock_quantity %>, parseInt(this.previousElementSibling.value) + 1)"
                    class="px-3 py-2 bg-gray-100 hover:bg-gray-200">+</button>
          </div>
          <%= f.submit "เพิ่มลงตะกร้า", class: "flex-1 bg-orange-500 text-white py-2 rounded-lg hover:bg-orange-600 font-semibold cursor-pointer" %>
        <% end %>
      <% end %>

      <!-- Product Description -->
      <div class="border-t pt-4">
        <h3 class="font-semibold mb-2">รายละเอียดสินค้า</h3>
        <p class="text-gray-600 leading-relaxed"><%= simple_format(@product.description) %></p>
      </div>

      <!-- Seller Info -->
      <div class="border-t pt-4">
        <h3 class="font-semibold mb-2">ร้านค้า</h3>
        <div class="flex items-center gap-3">
          <div class="w-10 h-10 bg-blue-100 rounded-full flex items-center justify-center">
            <span class="text-blue-600 font-semibold"><%= @product.seller.display_name.first %></span>
          </div>
          <div>
            <p class="font-medium"><%= @product.seller.display_name %></p>
            <p class="text-sm text-gray-500">สินค้า <%= @product.seller.products.published.count %> รายการ</p>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Reviews Section -->
  <div class="mt-8 bg-white rounded-lg shadow p-6">
    <h2 class="text-xl font-bold mb-4">รีวิวจากลูกค้า (<%= @reviews.count %>)</h2>
    <% if @reviews.any? %>
      <% @reviews.each do |review| %>
        <div class="border-b py-4 last:border-b-0">
          <div class="flex items-center gap-2 mb-2">
            <span class="font-medium"><%= review.user.full_name %></span>
            <div class="flex text-yellow-400 text-sm">
              <%= "★" * review.rating %><%= "☆" * (5 - review.rating) %>
            </div>
            <span class="text-sm text-gray-400"><%= time_ago_in_words(review.created_at) %> ที่ผ่านมา</span>
          </div>
          <p class="text-gray-600"><%= review.body %></p>
        </div>
      <% end %>
    <% else %>
      <p class="text-gray-400 text-center py-8">ยังไม่มีรีวิว</p>
    <% end %>
  </div>

  <!-- Related Products -->
  <% if @related_products.any? %>
    <div class="mt-8">
      <h2 class="text-xl font-bold mb-4">สินค้าที่เกี่ยวข้อง</h2>
      <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
        <% @related_products.each do |product| %>
          <%= render "products/card", product: product %>
        <% end %>
      </div>
    </div>
  <% end %>
</div>
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: เพิ่ม ProductVariant Model
สร้าง model `ProductVariant` ที่ belongs_to Product และมี fields: `sku`, `size`, `color`, `price_modifier`, `stock_quantity` พร้อม validation และ scope ที่เหมาะสม

### แบบฝึกหัดที่ 2: สร้าง Wish List Feature
เพิ่ม `Wishlist` model ที่ผูก User กับ Product แบบ many-to-many ผ่าน join table และสร้าง toggle action ด้วย Turbo Stream

### แบบฝึกหัดที่ 3: ปรับปรุง Category Tree
เพิ่ม method `depth` ใน Category model ที่คืนค่า depth ของ category (root = 0, child = 1, grandchild = 2) และใช้มันใน seed data เพื่อ generate breadcrumbs

### แบบฝึกหัดที่ 4: Admin User Management
สร้าง `Admin::UsersController` ที่มี actions: `index` (list + search), `show`, `ban` (toggle `banned_at`), และ `promote` (change role) พร้อม view ที่สมบูรณ์

### แบบฝึกหัดที่ 5: Ransack Search Form
ปรับปรุง product listing ให้ใช้ Ransack อย่างเต็มที่ โดยเพิ่ม search field ที่ค้นหาทั้ง name, description, และ seller.shop_name พร้อม highlight คำค้นหาใน result

---

## สรุปสิ่งที่ได้เรียนรู้ {#สรุป}

ใน Part 091 นี้ เราได้วางรากฐานของ Capstone Project 1 ครอบคลุม:

| สิ่งที่เรียน | เนื้อหา |
|---|---|
| **Product Planning** | Feature list, User stories, MoSCoW priority |
| **ER Diagram** | ออกแบบ 8 entities และ relationships ทั้งหมด |
| **Rails Setup** | `rails new` พร้อม PostgreSQL + Tailwind |
| **Devise + Roles** | Authentication + enum roles (buyer/seller/admin) |
| **Product Model** | Active Storage สำหรับรูปหลายรูป + scopes |
| **Category Model** | Self-referential association + breadcrumb |
| **Seed Data** | ข้อมูลจริงภาษาไทย 5 categories, 5 products |
| **Admin Dashboard** | Custom admin namespace + stats |
| **Product Listing** | Filter + sort + Pagy pagination + Stimulus |
| **Product Detail** | Image gallery + stock indicator + reviews |

### Key Concepts ที่ควรจำ

1. **Self-referential association** ใช้ `belongs_to :parent, class_name: "Category"` และ `has_many :children, foreign_key: :parent_id`
2. **Active Storage many-to-many** ใช้ `has_many_attached :images` สำหรับรูปหลายรูป
3. **Enum roles** ใช้ `enum :role, { buyer: 0, seller: 1, admin: 2 }` และ before_action guards
4. **Pagy** ใช้ `include Pagy::Backend` ใน controller และ `include Pagy::Frontend` ใน helper
5. **Ransack** ใช้ `@q = Model.ransack(params[:q])` และ `@q.result` เพื่อ filter

---

## ตัวอย่างใน Part ถัดไป {#ถัดไป}

**Part 092: Capstone 1 — E-Commerce Platform: Cart, Checkout, Payment, Order**

ใน Part ถัดไปเราจะสร้าง:
- **Shopping Cart** ด้วย session + database cart แบบ hybrid
- **Turbo Frames Cart** ที่อัปเดต realtime ไม่ reload หน้า
- **Checkout Flow** แบบหลายขั้นตอนด้วย Form Object
- **Stripe Payment** พร้อม webhook สำหรับ confirm order
- **Order Lifecycle** ด้วย state machine (pending → paid → shipped → delivered)
- **Email Notifications** ด้วย Action Mailer + Sidekiq
- **Deploy** ด้วย Kamal

```ruby
# ตัวอย่างโค้ดที่จะเรียนใน Part 092
class CartsController < ApplicationController
  def show
    @cart_items = current_cart_items
    @subtotal = @cart_items.sum { |item| item.product.price * item.quantity }
  end

  private

  def current_cart_items
    if current_user
      current_user.cart_items.includes(:product)
    else
      session[:cart] ||= {}
      # session-based cart logic
    end
  end
end
```

---

*Part 091 สิ้นสุดแล้ว — ไปต่อที่ Part 092 เพื่อสร้าง Cart, Checkout, และ Payment!*
