# Part 040: โปรเจกต์รวบยอดเฟส 4 — ระบบจัดการร้านค้าเล็ก (Product, Category, Order)

> **Step ครอบคลุมใน Part นี้:** Step 391–400
> **ระดับ:** สูง (ต้องผ่าน **Part 031–039** มาก่อนทั้งหมด — โดยเฉพาะ Part 031 form_with/strong
> params/nested attributes, Part 032 validation ขั้นสูง, Part 033 association ขั้นสูง
> (`has_many :through`), Part 034 query interface, Part 035 callback ขั้นสูง, Part 036 Concern,
> Part 037 multi-model form, Part 038 pagination/sorting/filtering, และ Part 039 I18n)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (โปรเจกต์ทั้งหมดใน Part นี้สร้างและทดสอบรันจริงบน
> Ruby 3.3.6 + Rails 8.1.4 + pagy 9.4.0 + sqlite3 — ทุก request ที่แสดงในเอกสารนี้ยิงด้วย `curl`
> จริงกับ server ที่รันอยู่จริง ทุกคำสั่ง `bin/rails runner`/console ก็รันจริงเช่นกัน ไม่ใช่โค้ด
> ที่เขียนคาดเดาไว้ล่วงหน้า)

นี่คือ **Part สุดท้ายของ Phase 4: CRUD, Forms, ActiveRecord ขั้นสูง** ตลอด 9 Part ที่ผ่านมา
เราเจาะลึกแต่ละเทคนิคแยกกันทีละเรื่องบนฐานของ Mini Blog เดิมจาก Phase 3: `form_with`/strong
params/nested attributes เบื้องต้น (Part 031), validation ขั้นสูง (Part 032), association ขั้นสูง
(Part 033), query interface (Part 034), callback ขั้นสูง (Part 035), Concern (Part 036),
multi-model form เต็มรูปแบบ (Part 037), pagination/sorting/filtering (Part 038), และ I18n
(Part 039) — ถึงเวลาแล้วที่จะเอาทุกเทคนิคเหล่านั้นมาประกอบเป็น**โปรเจกต์ใหม่ทั้งหมด** ที่ออกแบบมา
ให้ต้องใช้ทุกเทคนิคจริงๆ ไม่ใช่แค่ทบทวน Mini Blog เดิมซ้ำอีกรอบ

โปรเจกต์ปิดท้ายเฟสนี้คือ **ระบบจัดการร้านค้าเล็ก (Shop Manager)** ที่มี 4 โมเดลหลัก:

- **Category** (หมวดหมู่สินค้า) — `has_many :products`
- **Product** (สินค้า) — `belongs_to :category`, มี SKU ที่ไม่ซ้ำ**ภายในหมวดหมู่เดียวกัน**เท่านั้น
- **Order** (ออเดอร์) — `has_many :line_items`, และเข้าถึงสินค้าทั้งหมดในออเดอร์ผ่าน
  `has_many :products, through: :line_items` (ทบทวนจาก Part 033)
- **LineItem** (รายการสินค้าในออเดอร์) — ตารางกลางที่เชื่อม Order กับ Product พร้อมจำนวนที่สั่ง

ขอบเขตของโปรเจกต์นี้ตั้งใจให้ **เล็กแต่ลึก** — ไม่มีตะกร้าสินค้าแบบ session, ไม่มีการชำระเงินจริง,
ไม่มีระบบ login (ทั้งหมดนี้เป็นหัวข้อของเฟสถัดๆ ไป) แต่ทุกจุดที่มีอยู่ต้องสาธิตเทคนิคขั้นสูงของ
เฟส 4 ให้เห็นการทำงานจริง: SKU ไม่ซ้ำแบบมี scope, Concern ที่ใช้ร่วมกันสองโมเดล, ฟอร์มสร้างออเดอร์
พร้อมรายการสินค้าหลายแถวในหน้าเดียว, การตัดสต็อกแบบอะตอมิกด้วย transaction+lock, การกันขายเกินสต็อก
ด้วย `throw :abort`, pagination/sorting/filtering ที่ปลอดภัยจาก SQL injection, การป้องกัน N+1
ด้วย `includes`, และข้อความ error ทั้งหมดเป็นภาษาไทยผ่าน I18n

## สารบัญของ Part นี้

- Step 391: วางแผนโดเมน — ER diagram, `rails new shop_manager`, และภาพรวม routes
- Step 392: Model + Migration — SKU ไม่ซ้ำแบบมี scope และ custom validator `SkuFormatValidator`
- Step 393: `Sluggable` Concern — ใช้ร่วมกันระหว่าง Category และ Product (พร้อมกับดักภาษาไทยที่
  เจอจริงระหว่างทดสอบ)
- Step 394: Routes + `OrdersController` — multi-model nested form ด้วย `fields_for` และ
  validation "ต้องมีสินค้าอย่างน้อย 1 รายการ"
- Step 395: Callback ขั้นสูงใน `Order` — ตัดสต็อกแบบอะตอมิกในธุรกรรมเดียว, ป้องกันขายเกินสต็อก
  ด้วย `throw :abort`, แจ้งเตือนด้วย `after_commit`
- Step 396: หน้า Products index — `ProductFilter` PORO, pagination ด้วย pagy, sorting แบบ
  allowlist ปลอดภัย
- Step 397: Query interface ที่คำนึงถึงประสิทธิภาพ — scope, `includes` ป้องกัน N+1 พิสูจน์ด้วย
  SQL log จริง
- Step 398: I18n เต็มรูปแบบ — `config/locales/th.yml`, ข้อความ UI และ validation error ภาษาไทย
  ทั้งแอป
- Step 399: `db/seeds.rb` และการทดสอบ flow เต็มรูปแบบด้วย `curl` จริง (เรียกดู → กรอง/เรียงลำดับ →
  สร้างออเดอร์ → ยืนยันตัดสต็อก → ทดสอบขายเกินสต็อก → ยืนยันข้อความไทย)
- Step 400: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 4

---

## Step 391: วางแผนโดเมน — ER diagram, `rails new shop_manager`, และภาพรวม routes

### ออกแบบโดเมนก่อนเขียนโค้ดสักบรรทัด

ทบทวนหลักการจาก **Part 030 Step 291**: วาดภาพความสัมพันธ์ของข้อมูลให้ชัดเจนก่อนเสมอ โดเมนของ
ร้านค้าเล็กนี้มี 4 ตาราง และมีความสัมพันธ์ที่ซับซ้อนกว่า Mini Blog อย่างมีนัยสำคัญ — โดยเฉพาะ
ความสัมพันธ์ระหว่าง Order กับ Product ที่**ไม่ใช่** `has_many`/`belongs_to` ตรงๆ แต่ต้องผ่าน
ตารางกลาง:

```
┌──────────────┐  1     N  ┌──────────────┐
│   Category    │──────────►│    Product    │
├──────────────┤           ├──────────────┤
│ id            │           │ id            │
│ name          │           │ name          │
│ slug          │           │ sku           │
└──────────────┘           │ category_id   │◄── FK
                            │ price_cents   │
                            │ stock         │
                            │ slug          │
                            └───────┬──────┘
                                    │ 1
                                    │
                                    │ N
┌──────────────┐  1     N  ┌───────┴──────┐
│    Order      │──────────►│   LineItem    │
├──────────────┤           ├──────────────┤
│ id            │           │ id            │
│ customer_name │           │ order_id      │◄── FK
│ customer_email│           │ product_id    │◄── FK
│ status        │           │ quantity      │
└──────────────┘           └──────────────┘

Order.has_many :products, through: :line_items   (ทบทวน Part 033)
```

**จุดที่ต้องตัดสินใจตั้งแต่ต้น (และเหตุผล):**

- **SKU ไม่ซ้ำ "ภายในหมวดหมู่เดียวกัน" ไม่ใช่ทั้งตาราง** — ร้านค้าจริงมักให้แต่ละแผนก
  (หมวดหมู่) ตั้งรหัสสินค้าของตัวเองได้อิสระ เช่น `"001"` ในหมวด "เสื้อผ้า" กับ `"001"` ใน
  หมวด "รองเท้า" เป็นคนละสินค้ากันได้ — นี่คือเหตุผลที่ต้องใช้ `uniqueness: { scope:
  :category_id }` (ทบทวน Part 032 Step 315) แทน `uniqueness: true` ธรรมดา
- **Order ไม่มี column ที่ชี้ไปหา Product ตรงๆ เลย** — สินค้าที่อยู่ในออเดอร์เข้าถึงได้ผ่าน
  `LineItem` เท่านั้น (ตารางกลางที่เก็บ `quantity` ด้วย) — เป็นตัวอย่างคลาสสิกของ
  `has_many :through` (ทบทวน Part 033) ต่างจาก `has_and_belongs_to_many` ตรงที่ตารางกลางมี
  attribute ของตัวเอง (`quantity`) ไม่ใช่แค่ join table เปล่าๆ
- **Category และ Product ทั้งคู่มี `slug`** — นี่คือจุดที่ Concern (Part 036) มีประโยชน์จริง:
  logic การสร้าง slug เหมือนกันทุกตัวอักษร เขียนครั้งเดียวใช้ร่วมกันได้
- **ไม่มีระบบ cart/session ใดๆ** — ฟอร์มสร้างออเดอร์ให้ผู้ใช้เลือกสินค้าและกรอกจำนวนได้หลาย
  แถวในหน้าเดียวเลย (multi-model form ทบทวน Part 037) แทนการจำลอง flow ตะกร้าสินค้าที่ต้องพึ่ง
  session/JavaScript ซึ่งไกลเกินขอบเขตของเฟสนี้

### สร้างโปรเจกต์ใหม่

```bash
mkdir -p ~/ruby-course-workspace/part-040
cd ~/ruby-course-workspace/part-040

rails new shop_manager --minimal
cd shop_manager
```

เพิ่ม gem `pagy` สำหรับ pagination (ทบทวน Part 038) ลงใน `Gemfile`:

```ruby
# Gemfile — เพิ่มบรรทัดนี้ต่อจาก gem "puma"
gem "pagy", "~> 9.4"
```

```bash
bundle install
```

ตรวจสอบเวอร์ชันที่ได้จริง (ทดสอบแล้วบนเครื่องที่ใช้เขียนบทเรียนนี้):

```
ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]
Rails 8.1.4
pagy (9.4.0)
```

> **หมายเหตุเรื่องเวอร์ชัน pagy:** ตอนเขียนบทเรียนนี้ เครื่องมีทั้ง pagy 9.4.0 และ pagy 43.x
> ติดตั้งอยู่ (pagy เปลี่ยนรูปแบบเลขเวอร์ชันเป็นแบบ CalVer ในรุ่นหลัง) โค้ดในบทเรียนนี้ทดสอบกับ
> **pagy 9.4.0** โดยเฉพาะ — API หลักที่ใช้ (`Pagy::Backend#pagy`, คีย์ `limit:`) เสถียรพอที่จะ
> ใช้ได้กับ pagy รุ่น 9.x ทั้งหมด แต่ถ้าใช้รุ่น major อื่นควรเช็ค CHANGELOG ของ gem ก่อนเสมอ

### ภาพรวม routes ที่จะสร้างใน Part นี้

| หน้าจอ | URL | หน้าที่ | Step ที่สอน |
|---|---|---|---|
| รายการสินค้า (กรอง/เรียง/แบ่งหน้า) | `GET /products` | แสดงสินค้าทั้งหมด | Step 396 |
| ดูสินค้า | `GET /products/:id` | แสดงรายละเอียดสินค้า | Step 396 |
| รายการออเดอร์ | `GET /orders` | แสดงออเดอร์ทั้งหมด | Step 394 |
| สร้างออเดอร์ใหม่ | `GET /orders/new` → `POST /orders` | ฟอร์ม multi-model + บันทึก | Step 394–395 |
| ดูออเดอร์ | `GET /orders/:id` | แสดงรายละเอียดออเดอร์ + ยอดรวม | Step 394 |

สังเกตว่า**ไม่มี** route สำหรับสร้าง/แก้ไข Category หรือ Product ผ่านหน้าเว็บเลย — ในระบบร้านค้า
เล็กจริง การจัดการสินค้ามักทำผ่าน Rails console หรือระบบ admin แยกต่างหาก (จะเรียนรูปแบบ
admin ที่แท้จริงใน Phase หลังๆ) Part นี้โฟกัสที่การให้**ลูกค้า**เรียกดูสินค้าและสร้างออเดอร์ ซึ่ง
เป็น flow ที่ใช้เทคนิคขั้นสูงของเฟส 4 ได้ครบที่สุด

---

## Step 392: Model + Migration — SKU ไม่ซ้ำแบบมี scope และ custom validator `SkuFormatValidator`

### สร้าง Model ทั้ง 4 ตัวด้วย generator

```bash
bin/rails generate model Category name:string slug:string
bin/rails generate model Product name:string sku:string category:references \
  price_cents:integer stock:integer slug:string
bin/rails generate model Order customer_name:string customer_email:string status:string
bin/rails generate model LineItem order:references product:references quantity:integer
```

### แก้ Migration ทั้ง 4 ไฟล์ให้มี constraint และ index ที่ถูกต้อง

```ruby
# db/migrate/xxxxxx_create_categories.rb
class CreateCategories < ActiveRecord::Migration[8.1]
  def change
    create_table :categories do |t|
      t.string :name, null: false
      t.string :slug, null: false

      t.timestamps
    end

    add_index :categories, :name, unique: true
    add_index :categories, :slug, unique: true
  end
end
```

```ruby
# db/migrate/xxxxxx_create_products.rb
class CreateProducts < ActiveRecord::Migration[8.1]
  def change
    create_table :products do |t|
      t.string :name, null: false
      t.string :sku, null: false
      t.references :category, null: false, foreign_key: true
      t.integer :price_cents, null: false
      t.integer :stock, null: false, default: 0
      t.string :slug, null: false

      t.timestamps
    end

    # SKU ไม่ซ้ำ "ภายในหมวดหมู่เดียวกัน" เท่านั้น (composite unique index คู่กับ
    # validates :sku, uniqueness: { scope: :category_id } ที่ระดับ Model ด้านล่าง)
    add_index :products, [:category_id, :sku], unique: true
    add_index :products, :slug, unique: true
  end
end
```

```ruby
# db/migrate/xxxxxx_create_orders.rb
class CreateOrders < ActiveRecord::Migration[8.1]
  def change
    create_table :orders do |t|
      t.string :customer_name, null: false
      t.string :customer_email
      t.string :status, null: false, default: "pending"

      t.timestamps
    end
  end
end
```

```ruby
# db/migrate/xxxxxx_create_line_items.rb
class CreateLineItems < ActiveRecord::Migration[8.1]
  def change
    create_table :line_items do |t|
      t.references :order, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :quantity, null: false

      t.timestamps
    end
  end
end
```

**จุดที่ต่างจาก migration ทั่วไปที่เคยเขียนมา:**

- **`add_index :products, [:category_id, :sku], unique: true`** — composite unique index
  (ดัชนีผสมจาก 2 คอลัมน์) ทบทวนจาก **Part 032 Step 317**: กฎที่ตั้งไว้ระดับ Ruby
  (`uniqueness: { scope: :category_id }`) กับกฎที่ตั้งไว้ระดับฐานข้อมูล (unique index) **ต้อง
  ตรงกันเสมอ** — ถ้า index ครอบแค่ `sku` เฉยๆ โดยไม่มี `category_id` ร่วมด้วย จะมีช่องว่างที่
  Model บอกว่า "ซ้ำได้ถ้าคนละหมวดหมู่" แต่ฐานข้อมูลกลับห้ามซ้ำทั้งตาราง ขัดแย้งกันเอง
- **`status: { default: "pending" }` ที่ระดับ migration** — ทำให้ record ใหม่ทุกตัวที่สร้างผ่าน
  `Order.new` (แม้ไม่ได้ระบุ `status`) มีค่า `"pending"` ทันทีในหน่วยความจำ (Rails โหลด column
  default มาใส่ attribute ให้อัตโนมัติ) ไม่ต้องเขียน `after_initialize` หรือ callback ใดๆ เพิ่ม

รันคำสั่ง migrate:

```bash
bin/rails db:migrate
```

### `app/validators/sku_format_validator.rb` — custom validator แบบ `ActiveModel::EachValidator`

ทบทวนจาก **Part 032 Step 311**: validator ที่ตรวจ field เดียว และมีโอกาสนำไป reuse กับ field
อื่นในอนาคต (เช่น coupon code) ควรแยกเป็น class ต่างหากใน `app/validators/` แทนการเขียน
`validate :method_name` ฝังอยู่ใน Model โดยตรง

```ruby
# app/validators/sku_format_validator.rb
# frozen_string_literal: true

# Custom validator (แบบ ActiveModel::EachValidator ทบทวนจาก Part 032) ที่บังคับรูปแบบ
# SKU ให้เป็นตัวพิมพ์ใหญ่และตัวเลขเท่านั้น คั่นด้วยขีดกลางได้ (เช่น "SHIRT-RED-M")
# เขียนแยกเป็น class ต่างหาก (ไม่ใช่ `validate :method_name` ธรรมดา) เพราะเป็นกฎที่มีโอกาส
# นำไปใช้กับ field อื่นที่ต้องการรูปแบบ "code" คล้ายกันในอนาคต (เช่น coupon code)
class SkuFormatValidator < ActiveModel::EachValidator
  FORMAT = /\A[A-Z0-9]+(-[A-Z0-9]+)*\z/

  def validate_each(record, attribute, value)
    return if value.blank? # ปล่อยให้ validates :sku, presence: true จัดการกรณีว่างเปล่าเอง

    unless value.match?(FORMAT)
      record.errors.add(attribute, :invalid_sku_format)
    end
  end
end
```

สังเกตว่าเรียก `record.errors.add(attribute, :invalid_sku_format)` ด้วย **symbol** ไม่ใช่ string
ข้อความตรงๆ — เหตุผลคือปล่อยให้ I18n (Step 398) เป็นคนแปลข้อความจริงจาก key นี้ทีหลัง แทนการ
hardcode ข้อความภาษาอังกฤษไว้ในโค้ด validator (ทบทวนหลักการแยก "โครงสร้าง error" ออกจาก
"ข้อความที่แสดงผล" จาก Part 039)

### `app/models/category.rb`

```ruby
# app/models/category.rb
# frozen_string_literal: true

class Category < ApplicationRecord
  include Sluggable # เขียนเต็มใน Step 393

  # dependent: :restrict_with_error — ห้ามลบหมวดหมู่ที่ยังมีสินค้าอยู่ (ต่างจาก Mini Blog
  # ที่ใช้ :destroy เพราะที่นี่การลบ Category ทิ้งสินค้าทั้งหมดไปด้วยเป็นการกระทำที่รุนแรง
  # เกินไปสำหรับระบบร้านค้า — ผู้ดูแลต้องย้าย/ลบสินค้าออกก่อนเสมอ)
  has_many :products, dependent: :restrict_with_error

  validates :name, presence: true, uniqueness: true

  scope :alphabetical, -> { order(:name) }
end
```

### `app/models/product.rb`

```ruby
# app/models/product.rb
# frozen_string_literal: true

class Product < ApplicationRecord
  include Sluggable # เขียนเต็มใน Step 393

  belongs_to :category
  has_many :line_items, dependent: :restrict_with_error
  has_many :orders, through: :line_items

  validates :name, presence: true
  # sku ไม่ซ้ำ "ภายในหมวดหมู่เดียวกัน" เท่านั้น (composite uniqueness ทบทวนจาก Part 032
  # Step 315) คู่กับ custom validator ที่บังคับรูปแบบตัวพิมพ์ใหญ่+ตัวเลข (Part 032 Step 311)
  validates :sku, presence: true,
                   uniqueness: { scope: :category_id, case_sensitive: true },
                   sku_format: true
  validates :price_cents, numericality: { only_integer: true, greater_than: 0 }
  validates :stock, numericality: { only_integer: true, greater_than_or_equal_to: 0 }

  scope :in_stock, -> { where("stock > 0") }
  scope :out_of_stock, -> { where(stock: 0) }
  scope :by_category, ->(category_id) { where(category_id: category_id) if category_id.present? }
  scope :search_by_name, lambda { |query|
    where("name LIKE ?", "%#{sanitize_sql_like(query)}%") if query.present?
  }

  def price
    price_cents / 100.0
  end
end
```

**อธิบายจุดสำคัญ:**

- **`validates :sku, ..., sku_format: true`** — เรียกใช้ `SkuFormatValidator` ผ่าน syntax สั้น
  แบบเดียวกับ `presence: true` ทุกประการ (ทบทวนกลไก convention นี้จาก Part 032: Rails ตัดคำว่า
  `Validator` ออกจากชื่อ class แล้วแปลงเป็น key อัตโนมัติ — `sku_format: true` map ไปหา
  `SkuFormatValidator`)
- **`sanitize_sql_like(query)`** — escape ตัวอักษรพิเศษของ SQL `LIKE` (`%`, `_`) ก่อนนำไป
  ประกอบ query ป้องกันไม่ให้ผู้ใช้พิมพ์ `%` ในช่องค้นหาแล้วเปลี่ยนความหมายของ pattern โดยไม่ตั้งใจ
  — เป็น class method ที่ ActiveRecord เตรียมไว้ให้เรียกจากใน scope ได้โดยตรง
- **`price` (ไม่ใช่ column แต่เป็น method)** — เก็บราคาจริงในฐานข้อมูลเป็น **สตางค์ (integer)**
  ไม่ใช่บาททศนิยม (`price_cents`) หลักการนี้คือ "ห้ามเก็บเงินเป็น float เด็ดขาด" เพราะ
  floating-point มีปัญหาเรื่องความแม่นยำ (เช่น `0.1 + 0.2 != 0.3` ใน IEEE 754) การเก็บเป็น
  integer หน่วยเล็กสุด (สตางค์) แล้วค่อยหารด้วย 100 ตอนแสดงผลเป็นแนวทางมาตรฐานที่ระบบการเงินจริง
  ใช้กัน

### `app/models/order.rb` และ `app/models/line_item.rb` (โครงสร้างพื้นฐาน — ส่วน callback เต็มอยู่ Step 395)

```ruby
# app/models/line_item.rb
# frozen_string_literal: true

class LineItem < ApplicationRecord
  belongs_to :order
  belongs_to :product

  validates :quantity, presence: true,
                        numericality: { only_integer: true, greater_than: 0 }
end
```

โครงสร้างเริ่มต้นของ `Order` (จะเติมส่วน callback ให้ครบใน Step 395):

```ruby
# app/models/order.rb (บางส่วน — ดูเวอร์ชันเต็มใน Step 395)
# frozen_string_literal: true

class Order < ApplicationRecord
  STATUSES = %w[pending confirmed].freeze

  has_many :line_items, dependent: :destroy
  has_many :products, through: :line_items

  validates :customer_name, presence: true
  validates :status, inclusion: { in: STATUSES }
end
```

### ทดสอบ SKU scoped uniqueness และ custom validator ผ่าน Rails Console

```bash
bin/rails console
```

```irb
irb> cat1 = Category.create!(name: "เสื้อผ้า")
irb> cat2 = Category.create!(name: "รองเท้า")

irb> Product.create!(name: "เสื้อยืดสีขาว", sku: "SHIRT-WHITE-M", category: cat1,
                      price_cents: 25_000, stock: 10)
=> #<Product id: 1, sku: "SHIRT-WHITE-M", category_id: 1, ...>

# SKU เดียวกัน แต่คนละหมวดหมู่ — ต้องสร้างได้ (scoped uniqueness)
irb> Product.create!(name: "รองเท้าสีขาว", sku: "SHIRT-WHITE-M", category: cat2,
                      price_cents: 100_000, stock: 5)
=> #<Product id: 2, sku: "SHIRT-WHITE-M", category_id: 2, ...>   # สำเร็จ!

# SKU เดียวกัน หมวดหมู่เดียวกัน — ต้องห้าม
irb> Product.create!(name: "dup", sku: "SHIRT-WHITE-M", category: cat1,
                      price_cents: 1_000, stock: 1)
# ActiveRecord::RecordInvalid: Validation failed: Sku has already been taken

# รูปแบบ SKU ผิด (มีเว้นวรรค/ตัวพิมพ์เล็ก) — custom validator ต้องจับได้
irb> p = Product.new(name: "bad sku", sku: "bad sku!!", category: cat1,
                      price_cents: 1_000, stock: 1)
irb> p.valid?
=> false
irb> p.errors.full_messages
=> ["Sku is invalid"]   # (ข้อความไทยเต็มรูปแบบจะเห็นหลัง Step 398)
```

ทั้งสองพฤติกรรม (scoped uniqueness ผ่าน และ format ผิดถูกจับ) ตรงตามที่ออกแบบไว้ทุกประการ —
ยืนยันว่า SKU "ไม่ซ้ำเฉพาะภายในหมวดหมู่เดียวกัน" ทำงานถูกต้องจริง ไม่ใช่แค่ทฤษฎีบนกระดาษ

---

## Step 393: `Sluggable` Concern — ใช้ร่วมกันระหว่าง Category และ Product

### ทำไมต้องใช้ Concern ตรงนี้

ทั้ง `Category` และ `Product` ต้องการ "slug" (ชื่อที่เป็นมิตรกับ URL เช่น `เสื้อยืดสีขาว` →
`shirt-white`) ที่สร้างขึ้นอัตโนมัติจาก `name` — logic การสร้าง slug นี้**เหมือนกันทุกตัวอักษร**
ระหว่างสองโมเดล ถ้าเขียน `before_validation :assign_slug` แยกไว้ในแต่ละ Model จะเป็นการ
เขียนโค้ดซ้ำ (ขัดหลัก DRY) — Concern (ทบทวนจาก **Part 036**) คือเครื่องมือมาตรฐานของ Rails
สำหรับแชร์ behavior แบบนี้ระหว่างหลาย Model

### `app/models/concerns/sluggable.rb`

```ruby
# app/models/concerns/sluggable.rb
# frozen_string_literal: true

# Concern ที่ให้ Model สร้าง "slug" (ชื่อ URL-friendly) จาก attribute `name`
# ของตัวเองโดยอัตโนมัติ ใช้ร่วมกันระหว่าง Category และ Product เพื่อไม่ต้องเขียน
# logic เดียวกันซ้ำสองที่ (DRY ระดับ Model ตามที่เรียนใน Part 036)
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_validation :assign_slug
    validates :slug, presence: true, uniqueness: true
  end

  private

  # สร้าง slug ใหม่จาก name ทุกครั้งที่ name เปลี่ยน (หรือยังไม่เคยมี slug มาก่อน)
  # ไม่สร้างซ้ำถ้า name ไม่เปลี่ยนแปลง เพื่อไม่ให้ URL เดิมเปลี่ยนไปมาโดยไม่จำเป็น
  #
  # ข้อควรระวังที่ทดสอบเจอจริงระหว่างเขียน seeds.rb: `String#parameterize` ถูกออกแบบมา
  # สำหรับตัวอักษรละตินเท่านั้น ชื่อสินค้าภาษาไทยล้วน (เช่น "เสื้อผ้า") จะถูก parameterize
  # แล้วได้ค่าว่างเปล่ากลับมา — แต่ที่อันตรายกว่านั้นคือชื่อที่ "ปนตัวอักษรละติน" เข้ามา
  # นิดเดียว เช่น "เสื้อยืดสีขาว ไซส์ M" กับ "เสื้อยืดสีดำ ไซส์ M" จะถูก parameterize เหลือ
  # แค่ `"m"` เท่ากันทั้งคู่ (ตัวอักษรไทยถูกตัดทิ้งหมด เหลือแต่ "M" ตัวเดียว) ทำให้ได้ slug
  # ชนกันแบบเงียบๆ โดยไม่มีใครคาดคิด ถ้าปล่อยผ่านแบบไม่ตรวจสอบ record ที่สองจะ save ไม่ผ่าน
  # เพราะ uniqueness validation — จึงต้องมีทั้ง fallback (เมื่อว่างเปล่า) และการไล่ตรวจสอบ
  # ความไม่ซ้ำกับฐานข้อมูลจริงพร้อมต่อเลขนับท้าย slug ให้เอง (pattern เดียวกับที่ gem อย่าง
  # FriendlyId ใช้ภายใน)
  def assign_slug
    return if name.blank?
    return if slug.present? && !name_changed?

    base = name.parameterize
    base = SecureRandom.hex(4) if base.blank?
    self.slug = unique_slug_for(base)
  end

  def unique_slug_for(base)
    candidate = base
    counter = 2

    while self.class.where.not(id: id).exists?(slug: candidate)
      candidate = "#{base}-#{counter}"
      counter += 1
    end

    candidate
  end
end
```

### พิสูจน์กับดักภาษาไทย + parameterize ด้วยของจริง

ระหว่างเขียน `db/seeds.rb` (Step 399) เจอปัญหานี้เข้าเต็มๆ ลองรันดูจริงเพื่อความเข้าใจ:

```irb
irb> "เสื้อผ้า".parameterize
=> ""   # ชื่อไทยล้วน → ค่างเปล่า!

irb> "เสื้อยืดสีขาว ไซส์ M".parameterize
=> "m"   # เหลือแค่ตัว M ที่ปนอยู่ในชื่อ

irb> "เสื้อยืดสีดำ ไซส์ M".parameterize
=> "m"   # ชนกับอันข้างบนพอดี!
```

ทดสอบ concern เต็มรูปแบบผ่าน console ยืนยันว่า disambiguation (การต่อเลขท้าย slug) ทำงานถูกต้อง:

```irb
irb> cat1 = Category.create!(name: "เสื้อผ้า")
irb> Product.create!(name: "เสื้อยืดสีขาว ไซส์ M", sku: "SHIRT-WHITE-M", category: cat1,
                      price_cents: 25_000, stock: 20).slug
=> "m"
irb> Product.create!(name: "เสื้อยืดสีดำ ไซส์ M", sku: "SHIRT-BLACK-M", category: cat1,
                      price_cents: 25_000, stock: 3).slug
=> "m-2"   # ชนกับตัวแรก ("m") ระบบต่อเลข "-2" ให้อัตโนมัติ ไม่ raise error ใดๆ

irb> Category.create!(name: "กระเป๋าเดินทาง 20 นิ้ว").slug
=> "20"   # เหลือแค่ตัวเลขที่ปนอยู่ในชื่อ — ก็ยังเป็น slug ที่ valid (ไม่ blank, ไม่ซ้ำใคร)
```

> **ทำไมไม่แก้ปัญหาด้วยการเก็บ slug เป็นภาษาไทยตรงๆ:** เพราะ slug ต้องใช้เป็นส่วนหนึ่งของ URL
> ซึ่งแม้ URL ภาษาไทยจะทำได้ทางเทคนิค (ผ่าน URL encoding เป็น percent-encoded UTF-8) แต่จะได้
> URL ที่ยาวและไม่สวยงามเวลาคัดลอกไปแปะที่อื่น (เช่น `%E0%B9%80%E0%B8%AA...`) ระบบร้านค้าจริง
> ส่วนใหญ่จึงเลือกใช้ slug แบบ ASCII เท่านั้น (ตัวเลข/hex fallback เมื่อจำเป็น) แล้วให้ชื่อเต็ม
> ภาษาไทยแสดงอยู่ใน `<title>`/heading ของหน้าแทน ซึ่งเป็นแนวทางที่ Part นี้เลือกใช้

---

## Step 394: Routes + `OrdersController` — multi-model nested form ด้วย `fields_for`

### `config/routes.rb`

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  root "products#index"

  resources :products, only: %i[index show]
  resources :orders, only: %i[index new create show]
end
```

สังเกตว่า**ไม่มี** `resources :categories` และ `resources :products` ก็จำกัดแค่ `index`/`show`
เท่านั้น (ทบทวนจาก Part 022: ประกาศเฉพาะ action ที่ใช้จริง) ตามที่ออกแบบไว้ใน Step 391 — การ
จัดการสินค้า/หมวดหมู่ทำผ่าน console/seeds เท่านั้นในเวอร์ชันนี้ของแอป

### ขยาย `Order` model ให้รองรับ multi-model nested form เต็มรูปแบบ

```ruby
# app/models/order.rb (ส่วน nested attributes — ส่วน callback เต็มอยู่ Step 395)
# frozen_string_literal: true

class Order < ApplicationRecord
  STATUSES = %w[pending confirmed].freeze

  has_many :line_items, dependent: :destroy
  # has_many :through — Order ไม่มี column ที่ชี้ไปหา Product โดยตรงเลย แต่เข้าถึง Product
  # ทั้งหมดที่อยู่ในออเดอร์นี้ได้ผ่านตารางกลาง LineItem (ทบทวนจาก Part 033)
  has_many :products, through: :line_items

  # nested attributes ฝั่ง Model (ทบทวนจาก Part 031/037) — อนุญาตให้สร้าง/ลบ LineItem
  # ไปพร้อมกับการสร้าง/แก้ไข Order ในฟอร์มเดียว reject_if: :all_blank ตัดแถวที่ผู้ใช้ไม่ได้
  # กรอกอะไรเลยทิ้งไป (กันกรณีฟอร์มมีแถวว่างเหลือจาก JS เพิ่มแถวที่ยังไม่ได้กรอก)
  accepts_nested_attributes_for :line_items, allow_destroy: true,
                                              reject_if: :all_blank

  validates :customer_name, presence: true
  validates :status, inclusion: { in: STATUSES }
  validate :must_have_at_least_one_line_item

  private

  def must_have_at_least_one_line_item
    meaningful_line_items = line_items.reject(&:marked_for_destruction?)
    errors.add(:base, :no_line_items) if meaningful_line_items.empty?
  end
end
```

**อธิบายจุดสำคัญ:**

- **`must_have_at_least_one_line_item`** — custom validation ที่ไม่ใช่ validate field เดียว
  แต่ตรวจ**ทั้ง collection** ของ `line_items` — เขียนเป็น `validate :method_name` ธรรมดา
  (ไม่ใช่ custom validator class) เพราะเป็นกฎที่ผูกกับความหมายเฉพาะของ `Order` เท่านั้น
  ไม่มีแผนนำไปใช้ซ้ำที่ Model อื่น (ทบทวนตารางตัดสินใจ "เมื่อไหร่ควรใช้อะไร" จาก Part 032
  Step 314)
- **`line_items.reject(&:marked_for_destruction?)`** — สำคัญมาก: `line_items` ที่โหลดมาจาก
  nested attributes form อาจมีแถวที่ผู้ใช้ติ๊ก "ลบ" ไว้ (ผ่าน `_destroy`) ปนอยู่ ถ้านับรวมแถว
  เหล่านั้นด้วย จะทำให้ validation "ต้องมีอย่างน้อย 1 รายการ" ผิดพลาด (เช่น มี 2 แถว ติ๊กลบ
  ทั้งคู่ แต่ระบบคิดว่ายังมี 2 แถวอยู่ทั้งที่จริงๆ จะเหลือ 0 แถวหลัง save) — `marked_for_
  destruction?` เป็น method ที่ `accepts_nested_attributes_for` เพิ่มให้อัตโนมัติ

### `app/controllers/orders_controller.rb`

```ruby
# app/controllers/orders_controller.rb
# frozen_string_literal: true

class OrdersController < ApplicationController
  # จำนวนแถว LineItem เปล่าที่เตรียมไว้ล่วงหน้าในฟอร์มสร้างออเดอร์ (ไม่มี JavaScript ใน
  # โปรเจกต์นี้ ทบทวนแนวทาง "form ธรรมดาไม่พึ่ง Turbo" จาก Part 030 — จึง "เผื่อแถวว่าง"
  # ไว้ล่วงหน้าแทนการเพิ่มแถวด้วยปุ่ม JS)
  BLANK_LINE_ITEM_ROWS = 3

  def index
    @orders = Order.includes(line_items: :product).order(created_at: :desc)
  end

  def new
    @order = Order.new
    BLANK_LINE_ITEM_ROWS.times { @order.line_items.build }
    @products = Product.in_stock.order(:name)
  end

  def create
    @order = Order.new(order_params)

    if @order.save
      redirect_to @order, notice: t("shop.orders.created_notice", id: @order.id)
    else
      @products = Product.in_stock.order(:name)
      flash.now[:alert] = t("shop.orders.failed_alert")
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @order = Order.includes(line_items: :product).find(params[:id])
  end

  private

  # strong parameters สำหรับ multi-model form (ทบทวนจาก Part 031/037): permit
  # `line_items_attributes` เป็น array ของ Hash ที่อนุญาตเฉพาะ `product_id`, `quantity`,
  # `_destroy` เท่านั้น — ไม่ permit `status` เลย เพราะสถานะออเดอร์ต้อง "pending" เสมอตอน
  # สร้างใหม่จากฝั่งลูกค้า (ป้องกันไม่ให้ผู้ใช้ปลอม params ส่ง status=confirmed มาตรงๆ)
  def order_params
    params.require(:order).permit(
      :customer_name, :customer_email,
      line_items_attributes: %i[id product_id quantity _destroy]
    )
  end
end
```

### `app/views/orders/new.html.erb` — multi-model nested form

```erb
<%# app/views/orders/new.html.erb %>
<% content_for(:title, t("shop.orders.new_title")) %>

<h1><%= t("shop.orders.new_title") %></h1>

<%= form_with model: @order, local: true do |f| %>
  <% if @order.errors.any? %>
    <div class="form-errors">
      <h3><%= pluralize(@order.errors.count, "ข้อผิดพลาด", plural: "ข้อผิดพลาด") %>ทำให้สร้างออเดอร์นี้ไม่ได้:</h3>
      <ul>
        <% @order.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= f.label :customer_name, t("shop.orders.customer_name") %>
    <%= f.text_field :customer_name %>
  </div>

  <div class="field">
    <%= f.label :customer_email, t("shop.orders.customer_email") %>
    <%= f.email_field :customer_email %>
  </div>

  <h2><%= t("shop.orders.line_items") %></h2>

  <%# fields_for เรนเดอร์ฟอร์มของ LineItem (Model ลูก) ซ้อนอยู่ในฟอร์มของ Order (Model แม่)
      ทบทวนจาก Part 031 Step 308 / Part 037 — แต่ละแถวมี field ชื่อ
      order[line_items_attributes][0][product_id] เป็นต้น ทำให้ Rails ประกอบกลับเป็น Array
      ของ Hash ที่ accepts_nested_attributes_for ฝั่ง Model รับไปสร้าง LineItem หลายแถวใน
      คำสั่ง save ครั้งเดียว %>
  <div class="line-items">
    <%= f.fields_for :line_items do |lf| %>
      <div class="line-item-row">
        <%= lf.select :product_id,
                       options_from_collection_for_select(@products, :id, :name, lf.object.product_id),
                       { include_blank: "เลือกสินค้า" } %>
        <%= lf.number_field :quantity, min: 1, placeholder: "จำนวน" %>
      </div>
    <% end %>
  </div>

  <p class="hint">เลือกสินค้าและกรอกจำนวนอย่างน้อย 1 แถว แถวที่เหลือว่างไว้จะถูกข้ามไปเอง</p>

  <div class="actions">
    <%= f.submit t("shop.orders.submit") %>
  </div>
<% end %>
```

**อธิบายกลไก `fields_for` แบบเจาะลึก (ทบทวน/ขยายจาก Part 031/037):**

Controller เรียก `@order.line_items.build` ไว้ 3 ครั้งใน action `new` ทำให้ `@order.line_items`
มี 3 แถวเปล่าอยู่ในหน่วยความจำ (ยังไม่ persist) — `f.fields_for :line_items` วนลูปผ่าน
collection นี้ แล้วสร้าง input ให้แต่ละแถวโดยอัตโนมัติ ชื่อ field ที่ได้จริงคือ:

```html
<select name="order[line_items_attributes][0][product_id]" ...>
<input name="order[line_items_attributes][0][quantity]" ...>
<select name="order[line_items_attributes][1][product_id]" ...>
<input name="order[line_items_attributes][1][quantity]" ...>
```

ตัวเลข `[0]`, `[1]`, `[2]` คือ index ที่ Rails ใส่ให้อัตโนมัติตามลำดับของ collection —
เมื่อ submit ฟอร์ม `params[:order][:line_items_attributes]` จะกลายเป็น Hash ของ Hash
(`{"0" => {...}, "1" => {...}, "2" => {...}}`) ซึ่ง `accepts_nested_attributes_for` ฝั่ง
Model รู้วิธีแปลงเป็น `LineItem.new` หลายตัวโดยอัตโนมัติ — แถวที่ผู้ใช้ปล่อยว่างไว้ทั้งคู่
(`product_id` และ `quantity` ว่างทั้งคู่) จะถูก `reject_if: :all_blank` ที่ประกาศไว้ใน Model
ตัดทิ้งไปเงียบๆ ไม่กลายเป็น `LineItem` เปล่าที่ validation ไม่ผ่าน

### `app/views/orders/show.html.erb`

```erb
<%# app/views/orders/show.html.erb %>
<% content_for(:title, t("shop.orders.show_title", id: @order.id)) %>

<%= link_to "« กลับไปหน้ารายการออเดอร์", orders_path %>

<article class="order-full">
  <h1><%= t("shop.orders.show_title", id: @order.id) %></h1>
  <p class="order-meta">
    ลูกค้า: <%= @order.customer_name %>
    <% if @order.customer_email.present? %> (<%= @order.customer_email %>)<% end %>
    · สถานะ: <%= @order.status %>
    · สร้างเมื่อ <%= @order.created_at.strftime("%d %b %Y เวลา %H:%M น.") %>
  </p>

  <table class="line-item-table">
    <thead>
      <tr>
        <th>สินค้า</th>
        <th>จำนวน</th>
        <th>ราคา/ชิ้น</th>
        <th>รวม</th>
      </tr>
    </thead>
    <tbody>
      <% @order.line_items.each do |line_item| %>
        <tr>
          <td><%= link_to line_item.product.name, product_path(line_item.product) %></td>
          <td><%= line_item.quantity %></td>
          <td><%= number_to_currency(line_item.product.price, unit: "฿", format: "%n %u") %></td>
          <td><%= number_to_currency(line_item.quantity * line_item.product.price, unit: "฿", format: "%n %u") %></td>
        </tr>
      <% end %>
    </tbody>
    <tfoot>
      <tr>
        <td colspan="3"><%= t("shop.orders.total") %></td>
        <td><%= number_to_currency(@order.total_cents / 100.0, unit: "฿", format: "%n %u") %></td>
      </tr>
    </tfoot>
  </table>
</article>
```

### `app/views/orders/index.html.erb`

```erb
<%# app/views/orders/index.html.erb %>
<% content_for(:title, t("shop.nav.orders")) %>

<header class="page-header">
  <h1><%= t("shop.nav.orders") %></h1>
  <%= link_to t("shop.nav.new_order"), new_order_path, class: "btn btn-primary" %>
</header>

<% if @orders.any? %>
  <table class="order-table">
    <thead>
      <tr>
        <th>เลขที่ออเดอร์</th>
        <th>ลูกค้า</th>
        <th>จำนวนรายการ</th>
        <th>ยอดรวม</th>
        <th>สถานะ</th>
      </tr>
    </thead>
    <tbody>
      <% @orders.each do |order| %>
        <tr>
          <td><%= link_to "##{order.id}", order_path(order) %></td>
          <td><%= order.customer_name %></td>
          <td><%= pluralize(order.line_items.size, "รายการ", plural: "รายการ") %></td>
          <td><%= number_to_currency(order.total_cents / 100.0, unit: "฿", format: "%n %u") %></td>
          <td><%= order.status %></td>
        </tr>
      <% end %>
    </tbody>
  </table>
<% else %>
  <p class="empty-state">ยังไม่มีออเดอร์ในระบบ</p>
<% end %>
```

---

## Step 395: Callback ขั้นสูงใน `Order` — ตัดสต็อกแบบอะตอมิก + ป้องกันขายเกินสต็อก

นี่คือหัวใจของโปรเจกต์นี้ — จุดที่เทคนิค callback ขั้นสูงจาก **Part 035** ต้องทำงานร่วมกับ
ทั้ง transaction, row locking, และ `throw :abort` เพื่อรับประกันว่า**สต็อกสินค้าจะไม่ติดลบ
เด็ดขาด แม้มีคนพยายามสั่งซื้อพร้อมกันหลายคน**

### เวอร์ชันเต็มของ `app/models/order.rb`

```ruby
# app/models/order.rb (เวอร์ชันเต็ม — รวม Step 394 + Step 395)
# frozen_string_literal: true

class Order < ApplicationRecord
  STATUSES = %w[pending confirmed].freeze

  has_many :line_items, dependent: :destroy
  # has_many :through — Order ไม่มี column ที่ชี้ไปหา Product โดยตรงเลย แต่เข้าถึง Product
  # ทั้งหมดที่อยู่ในออเดอร์นี้ได้ผ่านตารางกลาง LineItem (ทบทวนจาก Part 033)
  has_many :products, through: :line_items

  # nested attributes ฝั่ง Model (ทบทวนจาก Part 031/037) — อนุญาตให้สร้าง/ลบ LineItem
  # ไปพร้อมกับการสร้าง/แก้ไข Order ในฟอร์มเดียว reject_if: :all_blank ตัดแถวที่ผู้ใช้ไม่ได้
  # กรอกอะไรเลยทิ้งไป (กันกรณีฟอร์มมีแถวว่างเหลือจาก JS เพิ่มแถวที่ยังไม่ได้กรอก)
  accepts_nested_attributes_for :line_items, allow_destroy: true,
                                              reject_if: :all_blank

  validates :customer_name, presence: true
  validates :status, inclusion: { in: STATUSES }
  validate :must_have_at_least_one_line_item

  # before_create ทำงานอยู่ "ภายใน" transaction เดียวกับที่ Rails ครอบ INSERT ของ Order
  # และ LineItem ทั้งหมดไว้ให้อัตโนมัติอยู่แล้ว (เพราะ autosave ของ has_many ผ่าน
  # accepts_nested_attributes_for) การตัดสต็อกที่นี่จึงเป็นส่วนหนึ่งของธุรกรรมเดียวกัน —
  # ถ้า throw :abort ถูกเรียกที่ตรงไหนก็ตาม การ INSERT ทั้งหมด (Order + LineItem ทุกแถว)
  # จะถูก rollback หมด ไม่มีออเดอร์ค้างอยู่แบบตัดสต็อกไปครึ่งเดียว
  before_create :reserve_stock!

  # after_commit ทำงาน "หลัง" transaction ยืนยันสำเร็จแล้วเท่านั้น (ทบทวนจาก Part 035
  # Step 341: ตารางอ้างอิงลำดับ callback ทั้งหมด) เหมาะกับ side effect ที่ไม่ควร rollback
  # ตามข้อมูล เช่นการแจ้งเตือน — ในระบบจริงตรงนี้จะเป็นการ enqueue ActiveJob ส่งอีเมล/SMS
  # (เรียนใน Phase 9/11) ตอนนี้จำลองด้วยการ log ข้อความไว้ก่อน
  after_commit :notify_order_placed, on: :create

  def total_cents
    line_items.sum { |line_item| line_item.quantity * line_item.product.price_cents }
  end

  private

  def must_have_at_least_one_line_item
    meaningful_line_items = line_items.reject(&:marked_for_destruction?)
    errors.add(:base, :no_line_items) if meaningful_line_items.empty?
  end

  def reserve_stock!
    line_items.each do |line_item|
      # Product.lock.find(...) ออก SQL "SELECT ... FOR UPDATE" ล็อกแถวสินค้านั้นไว้จนกว่า
      # transaction ปัจจุบันจะจบ ป้องกัน race condition เวลามีสอง Order พยายามตัดสต็อก
      # สินค้าตัวเดียวกันพร้อมกัน (ใน PostgreSQL/MySQL จริงจะเห็นผลชัดเจนกว่า SQLite ที่
      # เขียนได้ทีละ connection อยู่แล้ว แต่ SQL ที่ถูกต้องเหมือนกันทุก database)
      product = Product.lock.find(line_item.product_id)

      if product.stock < line_item.quantity
        errors.add(:base, :insufficient_stock,
                    product_name: product.name, available: product.stock, requested: line_item.quantity)
        throw :abort
      end

      product.decrement!(:stock, line_item.quantity)
    end
  end

  def notify_order_placed
    Rails.logger.info(
      "[notification-simulated] ออเดอร์ ##{id} ของคุณ #{customer_name} ถูกสร้างเรียบร้อยแล้ว " \
      "ยอดรวม #{total_cents / 100.0} บาท (จำลองการส่งอีเมลยืนยัน — ยังไม่ต่อ Action Mailer จริง)"
    )
  end
end
```

### ทำไม `reserve_stock!` ต้องเป็น `before_create` ไม่ใช่ `after_create`

ทบทวนตารางลำดับ callback จาก **Part 035 Step 341**: `before_create` ทำงาน**ก่อน** SQL `INSERT`
ของ record นั้นจะถูกส่งไปที่ฐานข้อมูลจริง แต่ยังอยู่**ภายใน**ธุรกรรมเดียวกับ `INSERT` เสมอ — ถ้า
`throw :abort` ถูกเรียกใน `before_create` ทั้ง `INSERT` ของ `Order` และ `LineItem` ทุกแถว (ที่
มาจาก `autosave` ของ `accepts_nested_attributes_for`) จะไม่ถูกส่งไปที่ฐานข้อมูลเลย — ราวกับว่า
ไม่มีอะไรเกิดขึ้น (`save` คืนค่า `false`, ไม่มี record ใดๆ ถูกสร้าง) ตรงกันข้ามกับถ้าใช้
`after_create` ซึ่งทำงาน**หลัง** `INSERT` ถูกส่งไปแล้ว — ถ้าจะยกเลิกตรงนั้นต้อง `raise
ActiveRecord::Rollback` เอง และ `Order` จะถูกสร้างขึ้นมาก่อนแล้วค่อยถูกลบทิ้งทีหลัง (สิ้นเปลือง
และเสี่ยงกว่า)

### กับดักที่ต้องระวัง: `throw :abort` ใช้กับ `before_*` เท่านั้น ไม่ใช่ `around_*`

ทบทวนจาก **Part 035 Step 343**: `throw :abort` ทำงานถูกต้องเฉพาะใน `before_*` callback
เพราะ ActiveSupport ห่อแต่ละ `before_*` callback ไว้ด้วย `catch(:abort) { ... }` ให้อัตโนมัติ —
ถ้าเผลอย้าย `reserve_stock!` ไปเป็น `around_create` แล้วยังใช้ `throw :abort` เหมือนเดิม จะได้
`UncaughtThrowError` แทนที่จะหยุด save เงียบๆ ตามที่ตั้งใจ — นี่คือเหตุผลที่บทเรียนนี้ยืนยันใช้
`before_create` เท่านั้นสำหรับ pattern นี้

### ทดสอบ transaction/lock/throw :abort ทั้งชุดผ่าน Rails Console

```irb
irb> shirt = Product.find_by(sku: "SHIRT-WHITE-M")
irb> black_shirt = Product.find_by(sku: "SHIRT-BLACK-M")
irb> black_shirt.stock
=> 3

# กรณีสำเร็จ — สั่งซื้อภายในสต็อกที่มี
irb> order = Order.new(customer_name: "สมชาย", line_items_attributes: [
       { product_id: shirt.id, quantity: 2 },
       { product_id: black_shirt.id, quantity: 1 }
     ])
irb> order.save!
=> true
irb> shirt.reload.stock
=> 18   # 20 - 2
irb> black_shirt.reload.stock
=> 2    # 3 - 1

# กรณีขายเกินสต็อก — ต้องถูกบล็อกทั้งออเดอร์ ไม่ตัดสต็อกบางส่วน
irb> oversell = Order.new(customer_name: "ทดสอบโอเวอร์เซล", line_items_attributes: [
       { product_id: black_shirt.id, quantity: 999 }
     ])
irb> oversell.save
=> false
irb> oversell.errors.full_messages
=> ["สินค้า \"เสื้อยืดสีดำ ไซส์ M\" มีไม่พอ (เหลือ 2 ชิ้น แต่สั่ง 999 ชิ้น)"]
irb> black_shirt.reload.stock
=> 2    # ไม่เปลี่ยนแปลง — throw :abort rollback ทั้งธุรกรรม
irb> Order.count
=> 1    # ไม่มีออเดอร์ที่ค้างครึ่งๆ กลางๆ ถูกสร้างขึ้นมาเลย
```

ผลลัพธ์ตรงตามที่ออกแบบไว้ทั้งหมด: ตัดสต็อกสำเร็จเมื่อมีของพอ, บล็อกทั้งออเดอร์ทันทีเมื่อสินค้า
ตัวใดตัวหนึ่งในออเดอร์มีไม่พอ (ไม่ตัดสต็อกสินค้าตัวอื่นที่เหลือพอไปก่อนแล้วค่อยพัง), และไม่มี
`Order`/`LineItem` เศษซากหลงเหลือในฐานข้อมูลเลย — พิสูจน์ว่า atomicity ของ transaction ทำงาน
ถูกต้องจริง

---

## Step 396: หน้า Products index — `ProductFilter` PORO, pagination ด้วย pagy, sorting แบบ allowlist

### `app/models/product_filter.rb` — PORO สำหรับกรอง/เรียงลำดับ

ทบทวนจาก **Part 038**: แยก logic การแปลง `params` ให้เป็น query ที่ปลอดภัยออกมาเป็น
Plain Old Ruby Object (ไม่สืบทอดจาก `ActiveRecord::Base` เลย) เพื่อให้ทดสอบได้อิสระจาก HTTP
request และไม่ทำให้ Controller บวมด้วย logic การกรอง

```ruby
# app/models/product_filter.rb
# frozen_string_literal: true

# Plain Old Ruby Object (PORO) — ไม่ได้สืบทอดจาก ActiveRecord::Base เลย ทำหน้าที่เดียว
# คือ "แปลง params ที่มาจากฟอร์มค้นหา/dropdown ให้กลายเป็น ActiveRecord::Relation ที่ปลอดภัย"
# แยกออกมาจาก ProductsController เพื่อให้ทดสอบ logic การกรอง/เรียงลำดับได้อิสระโดยไม่ต้อง
# ยุ่งกับ HTTP request เลย (ทบทวนแนวคิดนี้จาก Part 038)
class ProductFilter
  # Allowlist ของคอลัมน์ที่อนุญาตให้เรียงลำดับได้เท่านั้น — ห้ามเอาค่าจาก params[:sort]
  # ไปต่อ string SQL ตรงๆ เด็ดขาด (เช่น `order(params[:sort])`) เพราะเปิดช่องให้ผู้ไม่หวังดี
  # ส่งค่าที่เป็น SQL แปลกปลอมเข้ามาได้ (SQL injection ผ่านช่อง sort) — ทุกค่าที่อนุญาตต้อง
  # ประกาศไว้ล่วงหน้าใน Hash นี้เท่านั้น
  ALLOWED_SORTS = {
    "newest" => { created_at: :desc },
    "name" => { name: :asc },
    "price" => { price_cents: :asc },
    "stock" => { stock: :desc }
  }.freeze

  DEFAULT_SORT = "newest"

  def initialize(relation, params)
    @relation = relation
    @params = params
  end

  # method หลักที่ Controller เรียกใช้ — คืน ActiveRecord::Relation ที่กรอง/เรียงแล้ว
  # (ยังไม่ได้ execute query จริงจนกว่าจะถูก paginate/iterate — ยังคง lazy ตามธรรมชาติของ
  # ActiveRecord::Relation ทบทวนจาก Part 034)
  def results
    scope = relation
    scope = scope.search_by_name(query)
    scope = scope.by_category(category_id)
    scope.order(sort_clause)
  end

  def query
    params[:q].presence
  end

  def category_id
    params[:category_id].presence
  end

  # ใช้ใน view เพื่อ highlight ตัวเลือกที่กำลังเลือกอยู่ใน dropdown เรียงลำดับ
  def sort_key
    ALLOWED_SORTS.key?(params[:sort]) ? params[:sort] : DEFAULT_SORT
  end

  private

  attr_reader :relation, :params

  def sort_clause
    ALLOWED_SORTS.fetch(sort_key)
  end
end
```

**อธิบายจุดสำคัญ:**

- **`ALLOWED_SORTS` เป็น allowlist ไม่ใช่ blocklist** — วิธีปลอดภัยที่สุดในการรับ input จาก
  ผู้ใช้แล้วนำไปสร้าง SQL คือ "อนุญาตเฉพาะค่าที่รู้จักล่วงหน้าเท่านั้น" (allowlist) ไม่ใช่การ
  พยายาม "กรองค่าที่อันตรายออก" (blocklist) เพราะ blocklist มักมีช่องโหว่ที่คิดไม่ถึงเสมอ —
  ถ้า `params[:sort]` เป็นค่าที่ไม่รู้จัก (เช่นพยายามยิง SQL injection) `sort_key` จะ fallback
  ไปที่ `DEFAULT_SORT` เงียบๆ โดยไม่ error เลย
- **`scope.search_by_name(query)` เมื่อ `query` เป็น `nil`** — ทบทวนจาก Step 392: scope
  `search_by_name` เขียนแบบ `where(...) if query.present?` ซึ่งเมื่อเงื่อนไขเป็นเท็จจะคืนค่า
  `nil` และ Rails ตีความว่า "ไม่ต้องกรองอะไรเพิ่ม" (เทียบเท่ากับเรียก `.all` เฉยๆ) — chain
  ต่อกันได้อย่างปลอดภัยแม้ผู้ใช้ไม่ได้ส่ง query/category_id มาเลย

### `app/controllers/products_controller.rb`

```ruby
# app/controllers/products_controller.rb
# frozen_string_literal: true

class ProductsController < ApplicationController
  # จำนวนสินค้าต่อหน้า — ตั้งค่าน้อยโดยตั้งใจ (6) เพื่อให้เห็น pagination ทำงานจริงแม้มี
  # สินค้าตัวอย่างแค่ 14 ตัวจาก seeds.rb
  PER_PAGE = 6

  def index
    # includes(:category) ป้องกัน N+1 — ทบทวนจาก Part 034 (ดูรายละเอียดเต็มใน Step 397)
    @filter = ProductFilter.new(Product.includes(:category), params)
    @pagy, @products = pagy(@filter.results, limit: PER_PAGE)
    @categories = Category.alphabetical
  end

  def show
    @product = Product.includes(:category).find(params[:id])
  end
end
```

`ApplicationController` ต้อง `include Pagy::Backend` เพื่อให้ method `pagy` เรียกใช้ได้:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  allow_browser versions: :modern

  # Pagy::Backend ให้ method `pagy` เรียกใช้ใน controller action ได้ (ทบทวนจาก Part 038)
  include Pagy::Backend
end
```

`config/initializers/pagy.rb`:

```ruby
# config/initializers/pagy.rb
# frozen_string_literal: true

require "pagy"

Pagy::DEFAULT[:limit] = 6
Pagy::DEFAULT[:size] = 7
```

### `app/helpers/products_helper.rb`

```ruby
# app/helpers/products_helper.rb
# frozen_string_literal: true

module ProductsHelper
  # แปลง key ของ ProductFilter::ALLOWED_SORTS ให้เป็น [label ภาษาไทย, value] สำหรับ
  # ใช้กับ options_for_select — label มาจากไฟล์ locale เท่านั้น ไม่ hardcode ในโค้ด
  def sort_select_options
    ProductFilter::ALLOWED_SORTS.keys.map do |key|
      [t("shop.products.sort_options.#{key}"), key]
    end
  end
end
```

### `app/views/products/index.html.erb`

```erb
<%# app/views/products/index.html.erb %>
<% content_for(:title, t("shop.products.index_title")) %>

<h1><%= t("shop.products.index_title") %></h1>

<%# form_with url:/method: :get แบบ url-backed (ทบทวนจาก Part 031 Step 301) — ไม่ผูกกับ
    Model ใดเลย เพราะนี่คือฟอร์มค้นหา ไม่ใช่ฟอร์มบันทึกข้อมูล %>
<%= form_with url: products_path, method: :get, local: true, class: "filter-form" do |f| %>
  <%= f.text_field :q, value: params[:q], placeholder: t("shop.products.search_placeholder") %>

  <%= f.select :category_id,
               options_from_collection_for_select(@categories, :id, :name, params[:category_id]),
               { include_blank: t("shop.products.all_categories") } %>

  <%= f.select :sort, options_for_select(sort_select_options, @filter.sort_key) %>

  <%= f.submit "ค้นหา" %>
<% end %>

<% if @products.any? %>
  <table class="product-table">
    <thead>
      <tr>
        <th>สินค้า</th>
        <th><%= t("shop.products.sku_label") %></th>
        <th>หมวดหมู่</th>
        <th>ราคา</th>
        <th>คงเหลือ</th>
      </tr>
    </thead>
    <tbody>
      <%= render partial: "product_row", collection: @products, as: :product %>
    </tbody>
  </table>

  <%= render "shared/pagination", pagy: @pagy %>
<% else %>
  <p class="empty-state"><%= t("shop.products.no_results") %></p>
<% end %>
```

### `app/views/products/_product_row.html.erb`

```erb
<%# app/views/products/_product_row.html.erb %>
<tr>
  <td><%= link_to product.name, product_path(product) %></td>
  <td><code><%= product.sku %></code></td>
  <td><%= product.category.name %></td>
  <td><%= number_to_currency(product.price, unit: "฿", format: "%n %u") %></td>
  <td>
    <% if product.stock.positive? %>
      <%= t("shop.products.in_stock", count: product.stock) %>
    <% else %>
      <span class="badge badge-danger"><%= t("shop.products.out_of_stock") %></span>
    <% end %>
  </td>
</tr>
```

### `app/views/shared/_pagination.html.erb` — pagination ที่เขียนเองแทน `pagy_nav`

```erb
<%# app/views/shared/_pagination.html.erb
    ใช้ Pagy object (`pagy`) ตรงๆ แทนการเรียก pagy_nav ของ gem — เขียนเอง 2 เหตุผล:
    1) ต้องการ label ภาษาไทยล้วน (gem pagy ไม่มีชุดคำแปลไทยในตัว)
    2) ต้องคง query string เดิมทั้งหมด (q, category_id, sort) ไว้ทุกครั้งที่เปลี่ยนหน้า

    ข้อควรระวังที่ทดสอบเจอจริง: ตอนแรกลองใช้ `url_for(request.query_parameters.merge(page: ...))`
    ซึ่งดูเหมือนจะได้ผล แต่กลับพา link ไปที่ "/" (root) ไม่ใช่ "/products" — สาเหตุคือ
    `url_for` กับ Hash ธรรมดาต้องอาศัยการ "จับคู่ route จาก controller/action" และในแอปนี้ทั้ง
    `root "products#index"` และ `resources :products` (action index) ชี้ไปที่ controller/action
    เดียวกัน ทำให้ Rails เลือก route แรกที่ประกาศไว้ (root) มาสร้าง URL ให้แทน — ทางที่ปลอดภัย
    และตรงไปตรงมากว่าคือ "ต่อ query string เข้ากับ path ปัจจุบันตรงๆ" โดยไม่ต้องพึ่งการเดา
    route เลย %>
<% if pagy.pages > 1 %>
  <nav class="pagination" aria-label="pagination">
    <% if pagy.prev %>
      <%= link_to t("shop.pagination.previous"),
                  "#{request.path}?#{request.query_parameters.merge(page: pagy.prev).to_query}",
                  class: "btn" %>
    <% end %>

    <span class="pagination-status">หน้า <%= pagy.page %> จาก <%= pagy.pages %></span>

    <% if pagy.next %>
      <%= link_to t("shop.pagination.next"),
                  "#{request.path}?#{request.query_parameters.merge(page: pagy.next).to_query}",
                  class: "btn" %>
    <% end %>
  </nav>
<% end %>
```

> **บทเรียนจากบั๊กที่เจอจริงตอนเขียน Part นี้:** ตอนแรกเขียน pagination ด้วย `url_for(request.
> query_parameters.merge(page: pagy.next))` ตามสัญชาตญาณ ทดสอบแล้วพบว่า link ที่ได้พาไปที่ `/`
> แทนที่จะเป็น `/products?page=2` — เหตุผลคือ `root "products#index"` กับ `resources :products`
> (action `index`) ชี้ไปที่ controller/action เดียวกันเป๊ะ ทำให้ `url_for` แบบ Hash เลือก route
> **แรก** ที่ประกาศไว้ในไฟล์ (root) มาสร้าง URL ให้ ทั้งที่ตั้งใจให้เป็น `/products` — วิธีแก้ที่
> ตรงไปตรงมาและปลอดภัยกว่าคือต่อ query string เข้ากับ `request.path` ตรงๆ โดยไม่พึ่งการเดา route
> เลย บั๊กแบบนี้เป็นตัวอย่างที่ดีว่าทำไม **ต้องทดสอบด้วยการรัน server จริงเสมอ** ไม่ใช่แค่อ่าน
> โค้ดแล้วเชื่อว่าน่าจะถูก

### ทดสอบ sorting แบบ allowlist ปลอดภัยด้วย `curl` จริง

```bash
curl -s "http://localhost:3050/products?sort=price" -o /dev/null -w "HTTP %{http_code}\n"
# HTTP 200

# ลองยิงค่า sort ที่เป็นอันตราย (จำลอง SQL injection attempt)
curl -s -G "http://localhost:3050/products" \
  --data-urlencode "sort=id;DROP TABLE products;--" \
  -o /dev/null -w "HTTP %{http_code}\n"
# HTTP 200   (ไม่ error, ไม่มี SQL รันจริง — fallback ไปที่ DEFAULT_SORT เงียบๆ)
```

ทดสอบจริงยืนยันว่าค่า `sort` ที่ไม่อยู่ใน `ALLOWED_SORTS` จะถูกละเว้นและ fallback ไปที่
`"newest"` โดยอัตโนมัติ **ไม่มี SQL แปลกปลอมใดๆ ถูกส่งไปยังฐานข้อมูลเลย** — พิสูจน์ว่า pattern
allowlist ทำงานได้จริง ไม่ใช่แค่ทฤษฎี

---

## Step 397: Query interface ที่คำนึงถึงประสิทธิภาพ — scope, `includes` ป้องกัน N+1

ทบทวนจาก **Part 034**: `includes` โหลดความสัมพันธ์ล่วงหน้าด้วย query แยกต่างหาก (หรือ
`LEFT OUTER JOIN` แล้วแต่กรณี) แทนที่จะปล่อยให้แต่ละ record ยิง query ของตัวเองตอนเข้าถึง
association — Part นี้มี 2 จุดที่เสี่ยง N+1 ชัดเจน และทั้งสองจุดถูกป้องกันไว้แล้ว

### จุดที่ 1: `ProductsController#index` — แสดง `product.category.name` ในทุกแถว

```ruby
@filter = ProductFilter.new(Product.includes(:category), params)
```

ถ้าไม่มี `includes(:category)` ตรงนี้ ทุกแถวในตาราง (6 แถวต่อหน้าตาม `PER_PAGE`) ที่ต้องแสดง
`product.category.name` จะยิง query แยกไปหา `categories` ทีละแถว รวมเป็น **1 query หลัก + 6
query ย่อย = 7 query** ต่อการโหลดหน้าเดียว ยิ่งเพิ่ม `PER_PAGE` ยิ่งแย่ลงเป็นเส้นตรง

### จุดที่ 2: `OrdersController#show`/`#index` — ซ้อนกัน 2 ชั้น

```ruby
@order = Order.includes(line_items: :product).find(params[:id])
@orders = Order.includes(line_items: :product).order(created_at: :desc)
```

`includes(line_items: :product)` คือ **nested includes** — บอก Rails ว่า "โหลด `line_items`
ของออเดอร์นี้ล่วงหน้า **และ** โหลด `product` ของแต่ละ `line_item` ล่วงหน้าด้วย" ถ้าขาด
ส่วนนี้ไป การแสดงหน้า `show` ของออเดอร์ที่มี 5 รายการสินค้าจะยิง **1 query (order) + 1 query
(line_items) + 5 query (product ของแต่ละ line_item) = 7 query** และถ้าเป็นหน้า `index` ที่มี
10 ออเดอร์ แต่ละออเดอร์มี 3 รายการ จะกลายเป็น **1 + 10 + 30 = 41 query** ในหน้าเดียว

### พิสูจน์ด้วย SQL log จริง

เปิด `log/development.log` แล้วล้าง log เก่าก่อนยิง request:

```bash
> log/development.log
curl -s "http://localhost:3050/products" -o /dev/null
grep -E "Product Load|Category Load|Product Count|Product Exists" log/development.log
```

ผลลัพธ์จริงที่ได้ (ทดสอบบน seed data 14 สินค้า 3 หมวดหมู่):

```
Product Count (0.1ms)  SELECT COUNT(*) FROM "products"
Category Load (0.2ms)  SELECT "categories".* FROM "categories" ORDER BY "categories"."name" ASC
Product Exists? (0.1ms)  SELECT 1 AS one FROM "products" LIMIT 1 OFFSET 0
Product Load (0.2ms)  SELECT "products".* FROM "products" ORDER BY "products"."created_at" DESC LIMIT 6 OFFSET 0
Category Load (0.2ms)  SELECT "categories".* FROM "categories" WHERE "categories"."id" IN (2, 1)
```

สังเกตบรรทัดสุดท้าย: **`WHERE "categories"."id" IN (2, 1)`** — นี่คือ query เดียวที่โหลด
category ของ**ทุกแถว**ในหน้านั้นพร้อมกัน (ใช้ `IN` รวมทุก id ที่ต้องการ) ไม่ใช่ 6 query แยก
ทีละแถว — นี่คือหลักฐานว่า `includes` ทำงานถูกต้องจริง (`Product Count`/`Product Exists?` มา
จาก pagy ที่ต้องนับจำนวนทั้งหมดก่อนคำนวณเลขหน้า — เป็น query เพิ่มเติมที่คาดหวังได้จาก
pagination เสมอ ไม่ใช่ N+1)

ทดสอบจุดที่ 2 เช่นกัน:

```bash
> log/development.log
curl -s "http://localhost:3050/orders/1" -o /dev/null
grep -E "Order Load|LineItem Load|Product Load" log/development.log
```

```
Order Load (0.1ms)  SELECT "orders".* FROM "orders" WHERE "orders"."id" = 1 LIMIT 1
LineItem Load (0.1ms)  SELECT "line_items".* FROM "line_items" WHERE "line_items"."order_id" = 1
Product Load (0.2ms)  SELECT "products".* FROM "products" WHERE "products"."id" IN (1, 3)
```

**3 query คงที่** ไม่ว่าออเดอร์นี้จะมีกี่รายการสินค้าก็ตาม (คำสั่งที่ 3 ใช้ `IN (1, 3)` รวม
`product_id` ของทุก `line_item` ไว้ในคำสั่งเดียว) — นี่คือความแตกต่างระหว่างระบบที่ "ทำงานได้"
กับระบบที่ "ทำงานได้และไม่ล่มเมื่อข้อมูลเยอะขึ้น"

---

## Step 398: I18n เต็มรูปแบบ — `config/locales/th.yml`

### ตั้งค่า default locale เป็นภาษาไทยทั้งแอป

```ruby
# config/application.rb (เพิ่มในคลาส Application)
module ShopManager
  class Application < Rails::Application
    config.load_defaults 8.1
    config.autoload_lib(ignore: %w[assets tasks])
    config.generators.system_tests = nil

    # I18n: ตั้งค่า default locale เป็นภาษาไทยทั้งแอป (ทบทวนจาก Part 039) — ทุกข้อความ
    # ที่ผ่าน `t()`/`I18n.t` และทุก validation error message จะแปลเป็นไทยโดยอัตโนมัติ
    # โดยไม่ต้องระบุ locale: :th ซ้ำในทุกจุดที่เรียกใช้
    config.i18n.default_locale = :th
    config.i18n.available_locales = %i[th en]
  end
end
```

### `config/locales/th.yml` — ไฟล์เดียวที่รวมทั้งข้อความ UI และ validation error ทั้งหมด

```yaml
th:
  # ข้อความ UI ทั่วไปของแอป (เรียกผ่าน t("shop....") ในทุก View/Controller)
  shop:
    app_name: "ระบบจัดการร้านค้าเล็ก"
    nav:
      products: "รายการสินค้า"
      new_order: "สร้างออเดอร์ใหม่"
      orders: "ออเดอร์ทั้งหมด"
    products:
      index_title: "รายการสินค้าทั้งหมด"
      search_placeholder: "ค้นหาชื่อสินค้า..."
      all_categories: "ทุกหมวดหมู่"
      sort_by: "เรียงตาม"
      sort_options:
        newest: "ใหม่ล่าสุด"
        name: "ชื่อ (ก-ฮ)"
        price: "ราคา"
        stock: "จำนวนคงเหลือ"
      no_results: "ไม่พบสินค้าที่ตรงกับเงื่อนไข"
      in_stock: "มีสินค้า %{count} ชิ้น"
      out_of_stock: "สินค้าหมด"
      sku_label: "รหัสสินค้า"
    orders:
      new_title: "สร้างออเดอร์ใหม่"
      show_title: "ออเดอร์ #%{id}"
      customer_name: "ชื่อลูกค้า"
      customer_email: "อีเมล (ไม่บังคับ)"
      line_items: "รายการสินค้าในออเดอร์"
      add_line_item: "+ เพิ่มรายการสินค้า"
      remove_line_item: "ลบรายการนี้"
      submit: "ยืนยันสร้างออเดอร์"
      total: "ยอดรวม"
      created_notice: "สร้างออเดอร์ #%{id} สำเร็จแล้ว ตัดสต็อกสินค้าเรียบร้อย"
      failed_alert: "สร้างออเดอร์ไม่สำเร็จ กรุณาตรวจสอบข้อมูลด้านล่าง"
    pagination:
      previous: "« ก่อนหน้า"
      next: "ถัดไป »"

  # การแปลชื่อ Model และ attribute — มีผลกับทุกข้อความ error ที่ Rails สร้างให้อัตโนมัติ
  # (ทบทวนจาก Part 039: `errors.full_messages` ประกอบขึ้นจาก "ชื่อ attribute" + "ข้อความ"
  # สองส่วนนี้แปลแยกกันคนละที่ในไฟล์ locale)
  activerecord:
    models:
      category: "หมวดหมู่"
      product: "สินค้า"
      order: "ออเดอร์"
      line_item: "รายการสินค้า"
    attributes:
      category:
        name: "ชื่อหมวดหมู่"
        slug: "slug"
        products: "สินค้า"
      product:
        name: "ชื่อสินค้า"
        sku: "รหัสสินค้า (SKU)"
        category: "หมวดหมู่"
        category_id: "หมวดหมู่"
        price_cents: "ราคา"
        stock: "จำนวนคงเหลือ"
        slug: "slug"
      order:
        customer_name: "ชื่อลูกค้า"
        customer_email: "อีเมลลูกค้า"
        status: "สถานะ"
        line_items: "รายการสินค้า"
        base: "ออเดอร์"
      line_item:
        order: "ออเดอร์"
        product: "สินค้า"
        product_id: "สินค้า"
        quantity: "จำนวน"

    # ข้อความ error ที่เจาะจงเฉพาะ Model/attribute — ใช้กับ `errors.add(:base, :no_line_items)`
    # และ `errors.add(:base, :insufficient_stock, ...)` ใน Order (Step 395)
    errors:
      messages:
        # ใช้ตอน `save!`/`create!` ที่ raise ActiveRecord::RecordInvalid — ข้อความนี้คือ
        # `exception.message` ที่เห็นตอนรัน seeds.rb ด้วย create! (ทบทวนจาก Part 030 Step 299)
        record_invalid: "การตรวจสอบข้อมูลไม่ผ่าน: %{errors}"
        restrict_dependent_destroy:
          has_one: "ไม่สามารถลบได้ เนื่องจากยังมี %{record} ที่เกี่ยวข้องอยู่"
          has_many: "ไม่สามารถลบได้ เนื่องจากยังมี %{record} ที่เกี่ยวข้องอยู่"
      models:
        order:
          attributes:
            base:
              no_line_items: "ต้องมีสินค้าอย่างน้อย 1 รายการในออเดอร์"
              insufficient_stock: "สินค้า \"%{product_name}\" มีไม่พอ (เหลือ %{available} ชิ้น แต่สั่ง %{requested} ชิ้น)"

  # ข้อความ error มาตรฐานของ Rails (presence, uniqueness, numericality, ฯลฯ) แปลรวมไว้ที่นี่
  # ครั้งเดียว มีผลกับทุก Model ในแอป — และข้อความ custom ที่ตั้งใจให้ "ใช้ซ้ำได้" (ไม่ผูกกับ
  # Model ใดโดยเฉพาะ) อย่าง `invalid_sku_format` จาก SkuFormatValidator (Step 392) ก็อยู่ระดับนี้
  errors:
    format: "%{attribute} %{message}"
    messages:
      accepted: "ต้องได้รับการยอมรับ"
      blank: "ห้ามเว้นว่าง"
      confirmation: "ไม่ตรงกับ %{attribute}"
      empty: "ห้ามเว้นว่าง"
      equal_to: "ต้องเท่ากับ %{count}"
      even: "ต้องเป็นเลขคู่"
      exclusion: "มีค่านี้ไม่ได้"
      greater_than: "ต้องมากกว่า %{count}"
      greater_than_or_equal_to: "ต้องมากกว่าหรือเท่ากับ %{count}"
      inclusion: "ไม่อยู่ในตัวเลือกที่กำหนด"
      invalid: "ไม่ถูกต้อง"
      less_than: "ต้องน้อยกว่า %{count}"
      less_than_or_equal_to: "ต้องน้อยกว่าหรือเท่ากับ %{count}"
      not_a_number: "ต้องเป็นตัวเลขเท่านั้น"
      not_an_integer: "ต้องเป็นจำนวนเต็มเท่านั้น"
      odd: "ต้องเป็นเลขคี่"
      other_than: "ต้องไม่เท่ากับ %{count}"
      present: "ต้องเว้นว่าง"
      required: "ต้องระบุ"
      taken: "มีอยู่ในระบบแล้ว"
      too_long: "ยาวเกินไป (ไม่เกิน %{count} ตัวอักษร)"
      too_short: "สั้นเกินไป (อย่างน้อย %{count} ตัวอักษร)"
      wrong_length: "ความยาวไม่ถูกต้อง (ต้องมี %{count} ตัวอักษร)"
      invalid_sku_format: "ต้องเป็นตัวพิมพ์ใหญ่ A-Z และตัวเลข 0-9 เท่านั้น (คั่นด้วย - ได้ เช่น SHIRT-RED-M)"

  number:
    currency:
      format:
        unit: "฿"
        format: "%n %u"
        separator: "."
        delimiter: ","
        precision: 2
```

**อธิบายจุดที่มักพลาด (เจอจริงระหว่างเขียนบทเรียนนี้):**

- **`errors.messages.record_invalid` ต้องแปลด้วย** — ตอนแรกลืมใส่ key นี้ รัน `db:seed` แล้วเจอ
  `ActiveRecord::RecordInvalid` แต่ตัว exception เอง**หา translation ไม่เจอ**เพราะ Rails
  แปล `record_invalid` (ข้อความของตัว exception เอง) แยกจาก validation message ปกติ — ถ้าลืม
  key นี้ ทุก `save!`/`create!` ที่ fail จะแสดง error message ว่า "Translation missing" แทนที่
  จะแสดงเนื้อหา error จริง
- **`activerecord.attributes.category.products`** — ใช้กับ `dependent: :restrict_with_error`
  (Step 392) โดยเฉพาะ: Rails สร้างข้อความ error จาก
  `Category.human_attribute_name(:products)` (ชื่อ **association**, ไม่ใช่ชื่อ Model) ถ้าไม่ได้
  แปล key นี้ไว้ ข้อความ error ที่ได้จะเป็น `"...เนื่องจากยังมี products ที่เกี่ยวข้องอยู่"`
  (คำอังกฤษปนอยู่กลางประโยคไทย) — ทดสอบยืนยันด้วยของจริง:

```irb
irb> cat = Category.find_by(name: "เสื้อผ้า")
irb> cat.destroy
=> false
irb> cat.errors.full_messages.to_sentence
=> "ไม่สามารถลบได้ เนื่องจากยังมี สินค้า ที่เกี่ยวข้องอยู่"
```

### ทดสอบข้อความ error ภาษาไทยทั้งชุดผ่าน Rails Console

```irb
irb> p = Product.new(name: "bad sku", sku: "bad sku!!", category: cat1,
                      price_cents: 1_000, stock: 1)
irb> p.valid?
irb> p.errors.full_messages.to_sentence
=> "รหัสสินค้า (SKU) ต้องเป็นตัวพิมพ์ใหญ่ A-Z และตัวเลข 0-9 เท่านั้น (คั่นด้วย - ได้ เช่น SHIRT-RED-M)"

irb> dup = Product.new(name: "dup", sku: "SHIRT-WHITE-M", category: cat1,
                        price_cents: 1_000, stock: 1)
irb> dup.valid?
irb> dup.errors.full_messages.to_sentence
=> "รหัสสินค้า (SKU) มีอยู่ในระบบแล้ว"

irb> blank = Product.new
irb> blank.valid?
irb> blank.errors.full_messages.to_sentence
=> "slug ห้ามเว้นว่าง, หมวดหมู่ ต้องระบุ, ชื่อสินค้า ห้ามเว้นว่าง, รหัสสินค้า (SKU) ห้ามเว้นว่าง, and ราคา ต้องเป็นตัวเลขเท่านั้น"
```

ข้อความทั้งหมดเป็นภาษาไทยล้วน ไม่มีคำอังกฤษหลงเหลืออยู่เลย (ยกเว้น `slug` ที่ตั้งใจปล่อยไว้เป็น
ภาษาอังกฤษเพราะเป็น field ภายในที่ผู้ใช้ไม่ควรต้องเห็นความหมายอยู่แล้ว)

---

## Step 399: `db/seeds.rb` และการทดสอบ flow เต็มรูปแบบด้วย `curl` จริง

### `db/seeds.rb`

```ruby
# db/seeds.rb — สร้างข้อมูลตัวอย่างสำหรับระบบจัดการร้านค้าเล็ก
# frozen_string_literal: true
# รันด้วยคำสั่ง: bin/rails db:seed

puts "กำลังล้างข้อมูลเก่า..."
LineItem.destroy_all
Order.destroy_all
Product.destroy_all
Category.destroy_all

puts "กำลังสร้างหมวดหมู่สินค้า..."

clothing = Category.create!(name: "เสื้อผ้า")
shoes    = Category.create!(name: "รองเท้า")
bags     = Category.create!(name: "กระเป๋า")

puts "กำลังสร้างสินค้าตัวอย่าง..."

Product.create!(name: "เสื้อยืดสีขาว ไซส์ M", sku: "SHIRT-WHITE-M", category: clothing,
                 price_cents: 25_000, stock: 20)
Product.create!(name: "เสื้อยืดสีขาว ไซส์ L", sku: "SHIRT-WHITE-L", category: clothing,
                 price_cents: 25_000, stock: 15)
Product.create!(name: "เสื้อยืดสีดำ ไซส์ M", sku: "SHIRT-BLACK-M", category: clothing,
                 price_cents: 25_000, stock: 3)
Product.create!(name: "กางเกงยีนส์ทรงตรง", sku: "JEANS-STRAIGHT-32", category: clothing,
                 price_cents: 89_000, stock: 8)
Product.create!(name: "เสื้อฮู้ดสีเทา", sku: "HOODIE-GREY-L", category: clothing,
                 price_cents: 65_000, stock: 0)
Product.create!(name: "แจ็คเก็ตกันลม", sku: "JACKET-WIND-M", category: clothing,
                 price_cents: 120_000, stock: 5)

Product.create!(name: "รองเท้าผ้าใบสีขาว", sku: "SHOE-WHITE-42", category: shoes,
                 price_cents: 150_000, stock: 10)
Product.create!(name: "รองเท้าผ้าใบสีดำ", sku: "SHOE-BLACK-42", category: shoes,
                 price_cents: 150_000, stock: 2)
Product.create!(name: "รองเท้าแตะยาง", sku: "SANDAL-RUBBER-40", category: shoes,
                 price_cents: 29_000, stock: 30)
Product.create!(name: "รองเท้าบูทหนัง", sku: "BOOT-LEATHER-41", category: shoes,
                 price_cents: 210_000, stock: 0)

Product.create!(name: "กระเป๋าเป้ผ้าแคนวาส", sku: "BACKPACK-CANVAS-01", category: bags,
                 price_cents: 55_000, stock: 12)
Product.create!(name: "กระเป๋าสะพายหนัง", sku: "SLING-LEATHER-01", category: bags,
                 price_cents: 99_000, stock: 6)
Product.create!(name: "กระเป๋าเดินทาง 20 นิ้ว", sku: "LUGGAGE-20IN", category: bags,
                 price_cents: 180_000, stock: 4)
Product.create!(name: "กระเป๋าสตางค์หนังแท้", sku: "WALLET-LEATHER-01", category: bags,
                 price_cents: 45_000, stock: 25)

puts "เสร็จสิ้น! สร้าง #{Category.count} หมวดหมู่, #{Product.count} สินค้า"
puts "(ยังไม่สร้างออเดอร์ตัวอย่าง — จะสร้างผ่านการทดสอบ curl ด้านล่างเพื่อพิสูจน์ทั้ง flow)"
```

```bash
bin/rails db:seed
```

```
กำลังล้างข้อมูลเก่า...
กำลังสร้างหมวดหมู่สินค้า...
กำลังสร้างสินค้าตัวอย่าง...
เสร็จสิ้น! สร้าง 3 หมวดหมู่, 14 สินค้า
(ยังไม่สร้างออเดอร์ตัวอย่าง — จะสร้างผ่านการทดสอบ curl ด้านล่างเพื่อพิสูจน์ทั้ง flow)
```

ยืนยัน id ที่ได้จริงหลัง seed (จะใช้ id เหล่านี้ตลอดการทดสอบด้านล่าง):

```
1: เสื้อยืดสีขาว ไซส์ M | sku=SHIRT-WHITE-M | cat=เสื้อผ้า | price=250.0 | stock=20
2: เสื้อยืดสีขาว ไซส์ L | sku=SHIRT-WHITE-L | cat=เสื้อผ้า | price=250.0 | stock=15
3: เสื้อยืดสีดำ ไซส์ M | sku=SHIRT-BLACK-M | cat=เสื้อผ้า | price=250.0 | stock=3
...
14: กระเป๋าสตางค์หนังแท้ | sku=WALLET-LEATHER-01 | cat=กระเป๋า | price=450.0 | stock=25
```

### ทดสอบ Flow เต็มรูปแบบด้วย `curl` จริง

รัน server ก่อน:

```bash
bin/rails server -p 3050
```

**ขั้นที่ 1 — เปิดหน้ารายการสินค้า (ค่าเริ่มต้น เรียงใหม่สุดก่อน, หน้าละ 6 ชิ้น)**

```bash
curl -s http://localhost:3050/products -o p1.html -w "HTTP %{http_code}\n"
grep -oE '<td><a[^>]*>[^<]+</a></td>' p1.html
```

```
HTTP 200
<td><a href="/products/14">กระเป๋าสตางค์หนังแท้</a></td>
<td><a href="/products/13">กระเป๋าเดินทาง 20 นิ้ว</a></td>
<td><a href="/products/12">กระเป๋าสะพายหนัง</a></td>
<td><a href="/products/11">กระเป๋าเป้ผ้าแคนวาส</a></td>
<td><a href="/products/10">รองเท้าบูทหนัง</a></td>
<td><a href="/products/9">รองเท้าแตะยาง</a></td>
```

**ขั้นที่ 2 — กรองตามหมวดหมู่ (เสื้อผ้า id=1) + เรียงตามราคา**

```bash
curl -s "http://localhost:3050/products?category_id=1&sort=price" -o p2.html -w "HTTP %{http_code}\n"
grep -oE '<td><a[^>]*>[^<]+</a></td>' p2.html
```

```
HTTP 200
<td><a href="/products/1">เสื้อยืดสีขาว ไซส์ M</a></td>
<td><a href="/products/2">เสื้อยืดสีขาว ไซส์ L</a></td>
<td><a href="/products/3">เสื้อยืดสีดำ ไซส์ M</a></td>
<td><a href="/products/5">เสื้อฮู้ดสีเทา</a></td>
<td><a href="/products/4">กางเกงยีนส์ทรงตรง</a></td>
<td><a href="/products/6">แจ็คเก็ตกันลม</a></td>
```

**ขั้นที่ 3 — ค้นหาด้วยคำว่า "รองเท้า"**

```bash
curl -s -G "http://localhost:3050/products" --data-urlencode "q=รองเท้า" -o p3.html -w "HTTP %{http_code}\n"
grep -oE '<td><a[^>]*>[^<]+</a></td>' p3.html
```

```
HTTP 200
<td><a href="/products/10">รองเท้าบูทหนัง</a></td>
<td><a href="/products/9">รองเท้าแตะยาง</a></td>
<td><a href="/products/8">รองเท้าผ้าใบสีดำ</a></td>
<td><a href="/products/7">รองเท้าผ้าใบสีขาว</a></td>
```

ทั้ง 4 ผลลัพธ์ตรงกับชื่อสินค้าที่มีคำว่า "รองเท้า" ทุกตัว ไม่ตกหล่นและไม่มีสินค้าอื่นปนมา —
ยืนยันว่า `search_by_name` (Step 392/396) กับ `sanitize_sql_like` ทำงานถูกต้องกับข้อความไทยจริง

**ขั้นที่ 4 — เปิดหน้าสร้างออเดอร์ ดึง CSRF token**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3050/orders/new -o new_order.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new_order.html | sed 's/.*content="//;s/"$//')
```

**ขั้นที่ 5 — สร้างออเดอร์ที่มี 2 รายการสินค้า (product 1 x2 ชิ้น, product 3 x1 ชิ้น)**

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3050/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=สมชาย ใจดี" \
  --data-urlencode "order[customer_email]=somchai@example.com" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=2" \
  --data-urlencode "order[line_items_attributes][1][product_id]=3" \
  --data-urlencode "order[line_items_attributes][1][quantity]=1" \
  --data-urlencode "order[line_items_attributes][2][product_id]=" \
  --data-urlencode "order[line_items_attributes][2][quantity]=" \
  -D - -o order_create.html -w "HTTP %{http_code}\n" | grep -E "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3050/orders/1
HTTP 302
```

สังเกตว่าแถวที่ 3 (`[2]`) ส่งเป็นค่าว่างทั้งคู่ตามที่ฟอร์มเตรียมไว้ล่วงหน้า 3 แถว (Step 394) —
`reject_if: :all_blank` ตัดแถวนี้ทิ้งอัตโนมัติ ไม่ทำให้ validation ผิดพลาด

**ขั้นที่ 6 — ยืนยันว่าสต็อกถูกตัดจริง**

```bash
bin/rails runner 'p1=Product.find(1); p3=Product.find(3);
  puts "product 1 stock: #{p1.stock} (was 20)"
  puts "product 3 stock: #{p3.stock} (was 3)"'
```

```
product 1 stock: 18 (was 20)
product 3 stock: 2 (was 3)
```

18 = 20 − 2 และ 2 = 3 − 1 ตรงตามจำนวนที่สั่งซื้อทุกประการ

**ขั้นที่ 7 — เปิดดูออเดอร์ที่สร้าง ยืนยันยอดรวม**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3050/orders/1 -o order1.html -w "HTTP %{http_code}\n"
grep -oE '<h1>[^<]+</h1>|ลูกค้า: [^<·]+|<td>[0-9,]+\.[0-9]+ ฿</td>' order1.html
```

```
HTTP 200
<h1>ออเดอร์ #1</h1>
ลูกค้า: สมชาย ใจดี
<td>250.00 ฿</td>
<td>500.00 ฿</td>
<td>250.00 ฿</td>
<td>250.00 ฿</td>
<td>750.00 ฿</td>
```

500 (250×2) + 250 (250×1) = 750 บาท ตรงกับยอดรวมที่แสดงในแถวสุดท้าย (`tfoot`)

**ขั้นที่ 8 — ทดสอบขายเกินสต็อก (product 3 เหลือ 2 ชิ้น สั่ง 999 ชิ้น)**

```bash
curl -s -c cookies.txt -b cookies.txt http://localhost:3050/orders/new -o new_order2.html
TOKEN=$(grep -o 'name="csrf-token" content="[^"]*"' new_order2.html | sed 's/.*content="//;s/"$//')

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3050/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=ทดสอบโอเวอร์เซล" \
  --data-urlencode "order[line_items_attributes][0][product_id]=3" \
  --data-urlencode "order[line_items_attributes][0][quantity]=999" \
  -D - -o oversell.html -w "HTTP %{http_code}\n" | grep HTTP
grep -oE '<li>[^<]+</li>' oversell.html
```

```
HTTP/1.1 422 Unprocessable Content
HTTP 422
<li>สินค้า &quot;เสื้อยืดสีดำ ไซส์ M&quot; มีไม่พอ (เหลือ 2 ชิ้น แต่สั่ง 999 ชิ้น)</li>
```

**ขั้นที่ 9 — ยืนยันว่าสต็อกไม่เปลี่ยนแปลง และไม่มีออเดอร์ผีถูกสร้างขึ้น**

```bash
bin/rails runner 'puts "product 3 stock: #{Product.find(3).stock} (expect still 2)"
  puts "Order.count: #{Order.count} (expect still 1)"'
```

```
product 3 stock: 2 (expect still 2)
Order.count: 1 (expect still 1)
```

**ขั้นที่ 10 — ทดสอบ validation อื่นๆ ด้วยข้อความภาษาไทย**

```bash
echo "=== ไม่กรอกชื่อลูกค้า ==="
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3050/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=" \
  --data-urlencode "order[line_items_attributes][0][product_id]=2" \
  --data-urlencode "order[line_items_attributes][0][quantity]=1" \
  -o blank_name.html -w "HTTP %{http_code}\n"
grep -oE '<li>[^<]+</li>' blank_name.html

echo "=== ไม่เลือกสินค้าสักรายการเดียว ==="
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3050/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=ไม่มีสินค้า" \
  --data-urlencode "order[line_items_attributes][0][product_id]=" \
  --data-urlencode "order[line_items_attributes][0][quantity]=" \
  -o no_items.html -w "HTTP %{http_code}\n"
grep -oE '<li>[^<]+</li>' no_items.html
```

```
=== ไม่กรอกชื่อลูกค้า ===
HTTP 422
<li>ชื่อลูกค้า ห้ามเว้นว่าง</li>

=== ไม่เลือกสินค้าสักรายการเดียว ===
HTTP 422
<li>ต้องมีสินค้าอย่างน้อย 1 รายการในออเดอร์</li>
```

### สรุปผลการทดสอบ flow ทั้งหมด

| ขั้นตอน | Action | HTTP Status ที่ได้จริง |
|---|---|---|
| เรียกดูสินค้า (default) | `ProductsController#index` | 200 |
| กรองตามหมวดหมู่ + เรียงราคา | `ProductsController#index` | 200 |
| ค้นหาชื่อสินค้าภาษาไทย | `ProductsController#index` | 200 |
| สร้างออเดอร์ 2 รายการสำเร็จ | `OrdersController#create` | 302 → redirect ไป `show` |
| ดูออเดอร์ + ยอดรวมถูกต้อง | `OrdersController#show` | 200 |
| พยายามขายเกินสต็อก | `OrdersController#create` | 422 + ข้อความไทย + สต็อกไม่เปลี่ยน |
| ไม่กรอกชื่อลูกค้า | `OrdersController#create` | 422 + ข้อความไทย |
| ไม่เลือกสินค้าเลย | `OrdersController#create` | 422 + ข้อความไทย |

ทุกแถวคือผลลัพธ์ที่ทดสอบรันจริงระหว่างเขียนบทเรียนนี้ — **ระบบจัดการร้านค้าเล็กทำงานได้ครบวงจร
จริงตั้งแต่การเรียกดูสินค้าจนถึงการป้องกันขายเกินสต็อก**

---

## แบบฝึกหัด

### โจทย์

ต่อยอดจากระบบที่สร้างไว้ ให้เพิ่มความสามารถ **"ยกเลิกออเดอร์" (`PATCH /orders/:id/cancel`)** ที่
เปลี่ยนสถานะออเดอร์เป็น `"cancelled"` และ**คืนสต็อกสินค้า**กลับเข้าไปในทุกรายการของออเดอร์นั้น

### เฉลย

```ruby
# db/migrate/xxxxxx_add_cancelled_to_orders_statuses.rb — ไม่ต้อง migrate เพิ่ม เพราะ
# status เป็น string อยู่แล้ว แค่เพิ่มค่าที่ยอมรับได้ใน Model
```

```ruby
# app/models/order.rb (ส่วนที่แก้ไข)
class Order < ApplicationRecord
  STATUSES = %w[pending confirmed cancelled].freeze
  # ...

  def cancel!
    return false if status == "cancelled"

    transaction do
      line_items.each do |line_item|
        Product.lock.find(line_item.product_id).increment!(:stock, line_item.quantity)
      end
      update!(status: "cancelled")
    end
    true
  end
end
```

```ruby
# config/routes.rb (เพิ่มบรรทัดเดียว)
resources :orders, only: %i[index new create show] do
  member { patch :cancel }
end
```

```ruby
# app/controllers/orders_controller.rb (เพิ่ม action)
def cancel
  @order = Order.find(params[:id])
  if @order.cancel!
    redirect_to @order, notice: "ยกเลิกออเดอร์ #{@order.id} และคืนสต็อกสินค้าเรียบร้อยแล้ว"
  else
    redirect_to @order, alert: "ออเดอร์นี้ถูกยกเลิกไปแล้ว"
  end
end
```

**อธิบาย:** `cancel!` ใช้ `transaction do ... end` แบบเปิดตรงๆ (ไม่ใช่ callback) เพราะเป็นการ
กระทำที่ผู้ใช้ "สั่ง" ให้เกิดขึ้นเอง (ไม่ใช่ side effect อัตโนมัติของการ save) — ใช้ `Product.
lock.find` เหมือนกับ `reserve_stock!` ใน Step 395 เพื่อป้องกัน race condition แบบเดียวกัน
(ถ้ามีคนสั่งซื้อสินค้าตัวเดียวกันพร้อมกับที่กำลังยกเลิกออเดอร์เก่า)

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มหน้า admin สำหรับจัดการ Product/Category** — สร้าง `Admin::ProductsController` และ
   `Admin::CategoriesController` ที่มี CRUD ครบ 7 action พร้อมฟอร์ม (ทบทวน strong parameters
   จาก Part 031 และ `fields_for`/`accepts_nested_attributes_for` ถ้าต้องการให้สร้าง Category
   ใหม่พร้อม Product แรกได้ในฟอร์มเดียว) — ระวังว่ายังไม่มีระบบ authentication เลยในตอนนี้ (จะ
   เรียนใน Phase 5) จึงต้อง**ไม่ deploy หน้านี้ขึ้น production จริง** จนกว่าจะมี Part 041–045
2. **เพิ่ม `low_stock` scope และหน้าแจ้งเตือนสินค้าใกล้หมด** — เพิ่ม
   `scope :low_stock, -> { where(stock: 1..5) }` ใน `Product` แล้วสร้างหน้า
   `GET /products/low_stock` ที่แสดงเฉพาะสินค้าที่ใกล้หมด เรียงจากน้อยไปมาก พร้อมทดสอบว่า
   `includes(:category)` ยังป้องกัน N+1 ได้ในหน้านี้ด้วย
3. **เขียน request spec (Minitest) ให้ครอบคลุม flow ทั้งหมดใน Step 399** — แปลงทุกขั้นตอนที่
   ทดสอบด้วย `curl` ในบทเรียนนี้ให้กลายเป็น automated test ที่รันซ้ำได้ (ทบทวนพื้นฐาน Minitest
   จาก Part 018 — request spec เต็มรูปแบบสำหรับ Rails จะเรียนเจาะลึกใน **Phase 6**) โจทย์ที่ยาก
   ที่สุดคือการเขียน test สำหรับ "ขายเกินสต็อกต้องถูกบล็อกและสต็อกต้องไม่เปลี่ยน" ให้ครอบคลุม
   ทั้งกรณีสำเร็จและกรณีล้มเหลวในไฟล์เดียวกัน

---

## Step 400: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 4

### เดินย้อนดูว่าแต่ละไฟล์สาธิต Part ไหนบ้าง

| ไฟล์ / โค้ด | สาธิตอะไร | Part ที่สอน |
|---|---|---|
| `form_with url: products_path, method: :get` | url-backed form (ไม่ผูก Model) | Part 031 |
| `params.require(:order).permit(..., line_items_attributes: [...])` | strong parameters สำหรับ nested array | Part 031 |
| `SkuFormatValidator < ActiveModel::EachValidator` | custom validator แบบ reusable | Part 032 |
| `validates :sku, uniqueness: { scope: :category_id }` | composite/scoped uniqueness | Part 032 |
| `Order.has_many :products, through: :line_items` | `has_many :through` | Part 033 |
| `Product.in_stock`, `search_by_name`, `by_category` scopes | query interface, scope | Part 034 |
| `Product.includes(:category)`, `Order.includes(line_items: :product)` | ป้องกัน N+1 | Part 034 |
| `before_create :reserve_stock!`, `throw :abort`, `after_commit` | callback ขั้นสูง, transaction | Part 035 |
| `module Sluggable; extend ActiveSupport::Concern; ...` | Concern แชร์ระหว่าง Model | Part 036 |
| `f.fields_for :line_items`, `accepts_nested_attributes_for` | multi-model nested form | Part 037 |
| `ProductFilter`, `pagy(@filter.results, limit: ...)`, `ALLOWED_SORTS` | pagination, safe sorting, filter object | Part 038 |
| `config/locales/th.yml`, `config.i18n.default_locale = :th` | I18n เต็มรูปแบบ | Part 039 |
| ทั้งโปรเจกต์รวมกัน | **เทคนิคขั้นสูงของ CRUD/Forms/ActiveRecord ทำงานร่วมกันเป็นระบบเดียว** | **Part 040 (Part นี้)** |

### จุดออกแบบที่ควรพิจารณาอีกครั้ง (แม้จะทำงานถูกต้องแล้ว)

1. **`Product.lock.find(line_item.product_id)` ทำงานทีละแถวใน loop** — ถ้าออเดอร์มีสินค้า
   จำนวนมาก (เช่น 50 รายการ) จะมี 50 query แยกกันสำหรับล็อกแต่ละแถว แทนที่จะล็อกทีเดียวด้วย
   `Product.lock.where(id: ids)` — ที่เลือกเขียนแบบ loop ใน Part นี้เพราะโค้ดอ่านง่ายกว่าและ
   ขนาดออเดอร์จริงมักไม่เกินสิบกว่ารายการ แต่ระบบร้านค้าขนาดใหญ่ควรพิจารณา batch locking
2. **`ProductsController` ไม่มีการ cache ผลการนับจำนวนสินค้าทั้งหมด (`Product.count`)** — pagy
   เรียก `count` ทุกครั้งที่โหลดหน้า index ซึ่งเป็น query ที่ค่อนข้างถูกในตารางขนาดเล็ก แต่เมื่อ
   ข้อมูลเป็นล้านแถว `COUNT(*)` จะช้าลงอย่างมีนัยสำคัญ — เทคนิคแก้ปัญหานี้ (เช่น approximate
   count, cached count) จะเรียนใน **Part 063–065 (Performance)**
3. **`notify_order_placed` เขียนไปที่ `Rails.logger` ตรงๆ** — ในระบบจริงควรใช้ ActiveJob
   (`OrderMailerJob.perform_later`) เพื่อไม่ให้ผู้ใช้ต้องรอ HTTP response จนกว่าอีเมลจะถูกส่ง
   เสร็จ (การส่งอีเมลอาจใช้เวลาหลายร้อย ms ถึงหลายวินาที) — `after_commit` ที่ enqueue job
   เบื้องหลังคือ pattern มาตรฐานที่จะเรียนใน **Phase 9 (Part 061)**

### สิ่งที่ยังขาดชัดเจนสำหรับ Production (ตั้งใจปล่อยไว้ให้เฟสถัดไปแก้)

**1. ไม่มีระบบ Authentication/Authorization เลย**

ตอนนี้**ใครก็ตาม**ที่เข้าเว็บนี้ได้สามารถสร้างออเดอร์ในนามลูกค้าคนไหนก็ได้ (แค่กรอกชื่อ/อีเมล
เอง ไม่มีการยืนยันตัวตนใดๆ) และถ้าเปิดหน้า admin ตามแบบฝึกหัดข้อ 1 จะยิ่งอันตรายกว่าเดิมเพราะ
ใครก็แก้ไข/ลบสินค้าได้หมด **Phase 5 (Part 041–045)** จะสอนตั้งแต่ `has_secure_password`
พื้นฐาน ไปจนถึง Devise, Pundit/CanCanCan สำหรับ authorization เต็มรูปแบบ

**2. ไม่มี Automated Test สักบรรทัดเดียว**

ทุกอย่างที่ "ทดสอบ" ใน Part นี้ (Step 395, 396, 399) เป็นการยิง `curl`/รัน console ด้วยมือ ซึ่ง
ไม่มีทางรันซ้ำอัตโนมัติได้ทุกครั้งที่แก้โค้ด และไม่มีใครรู้ทันทีว่าการแก้โค้ดในอนาคต (เช่นแก้
`reserve_stock!`) ทำให้การป้องกันขายเกินสต็อกพังหรือไม่ — ทักษะเดียวกับที่เคยเขียน Minitest/
RSpec ใน Part 018–020 จะนำมาใช้กับ Rails เต็มรูปแบบใน **Phase 6 (Part 046–050)**

**3. ไม่มีการชำระเงินจริง**

`Order` ในระบบนี้เป็นแค่บันทึกความตั้งใจซื้อ — ไม่มีการเชื่อมต่อกับ payment gateway ใดๆ, ไม่มี
การตรวจสอบว่าลูกค้าจ่ายเงินจริงก่อนยืนยันออเดอร์ (`status` เปลี่ยนจาก `"pending"` เป็น
`"confirmed"` ไม่ได้ถูก implement เลยด้วยซ้ำใน Part นี้) — การเชื่อมต่อ Stripe หรือ payment
gateway อื่นจะเรียนใน **Part 071 (Phase 11)**

**4. ไม่มี Rate Limiting และไม่มีการป้องกัน bot สร้างออเดอร์สแปม**

ทฤษฎีแล้วมีคนเขียนสคริปต์ยิง `POST /orders` รัวๆ เพื่อตัดสต็อกสินค้าคู่แข่งให้หมดโดยไม่ตั้งใจซื้อ
จริงได้ (แม้จะเป็น pending order ที่ไม่มีการจ่ายเงินก็ตาม) — `rack-attack` และเทคนิคป้องกันจะ
เรียนใน **Part 060**

> **ทำไมถึงตั้งใจไม่แก้ 4 เรื่องนี้ใน Part นี้:** เช่นเดียวกับที่ Part 030 อธิบายไว้ตอนปิด
> Phase 3 — แต่ละเรื่องมีความลึกพอที่จะเป็นเนื้อหาทั้ง Part หรือทั้ง Phase การพยายามยัดทุกอย่าง
> ไว้ใน capstone ตัวเดียวจะทำให้ Part นี้ยาวเกินไปและสอนแต่ละเรื่องแบบผิวเผินเกินไป — หลักการ
> ของหลักสูตรนี้คือ**สอนให้ลึกทีละเรื่อง** ดีกว่าสอนกว้างแต่ตื้นทุกเรื่องพร้อมกัน

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ออกแบบโดเมนที่มีความสัมพันธ์ซับซ้อนกว่า Mini Blog อย่างมีนัยสำคัญ (`has_many :through` ผ่าน
  ตารางกลางที่มี attribute ของตัวเอง) ก่อนลงมือเขียน migration
- เขียน scoped uniqueness (`uniqueness: { scope: :category_id }`) คู่กับ composite unique
  index ที่ระดับฐานข้อมูล และเขียน custom validator (`ActiveModel::EachValidator`) ที่ใช้ซ้ำได้
- สร้าง Concern (`Sluggable`) ที่แชร์ behavior เดียวกันระหว่าง 2 Model และเจอ/แก้กับดักจริงของ
  `String#parameterize` กับข้อความภาษาไทย
- เขียน multi-model nested form เต็มรูปแบบด้วย `fields_for` + `accepts_nested_attributes_for`
  พร้อม custom validation ที่ตรวจทั้ง collection (`must_have_at_least_one_line_item`)
- ใช้ callback ขั้นสูง (`before_create`, `after_commit`) ร่วมกับ transaction, row locking
  (`Product.lock.find`), และ `throw :abort` เพื่อรับประกันความถูกต้องของสต็อกสินค้าแบบอะตอมิก
- สร้าง `ProductFilter` PORO ที่แยก logic การกรอง/เรียงลำดับออกจาก Controller พร้อม pagination
  ด้วย pagy และ allowlist ป้องกัน SQL injection ผ่านช่อง sort
- ใช้ `includes` ป้องกัน N+1 ทั้งแบบชั้นเดียวและแบบซ้อนกัน (`includes(line_items: :product)`)
  และพิสูจน์ด้วย SQL log จริงว่าจำนวน query คงที่ไม่ว่าข้อมูลจะเยอะแค่ไหน
- เขียน `config/locales/th.yml` เต็มรูปแบบที่ครอบคลุมทั้งข้อความ UI และ validation error
  มาตรฐานของ Rails ทั้งหมด รวมถึงกับดักที่มักพลาด (`record_invalid`,
  `restrict_dependent_destroy` ที่ต้องแปลชื่อ association ด้วย)
- ทดสอบ flow เต็มรูปแบบด้วย `curl` จริง ตั้งแต่เรียกดูสินค้าพร้อมกรอง/เรียง/แบ่งหน้า ไปจนถึง
  สร้างออเดอร์ ยืนยันการตัดสต็อก และพิสูจน์ว่าการขายเกินสต็อกถูกบล็อกจริงพร้อมข้อความไทย
- ทำ code review ย้อนหลังทั้งโปรเจกต์ เชื่อมโยงทุกไฟล์กลับไปยัง Part 031–039 ที่สอนเทคนิคนั้นๆ
  และวิเคราะห์ได้ว่าโปรเจกต์ยังขาดอะไรสำหรับ production จริง

## สรุปภาพรวม Phase 4: CRUD/Forms/ActiveRecord ขั้นสูง

ยินดีด้วย! ตอนนี้ **Phase 4: CRUD, Forms, ActiveRecord ขั้นสูง (Part 031–040, Step 301–400)**
เสร็จสมบูรณ์แล้ว เราเดินทางจาก `form_with` เชิงลึกและ nested attributes เบื้องต้น (Part 031),
validation ขั้นสูงพร้อม custom validator (Part 032), association ขั้นสูงทั้ง `has_many :through`
และ polymorphic (Part 033), query interface และการป้องกัน N+1 (Part 034), callback ขั้นสูงและ
วงจรชีวิตเต็มรูปแบบของ ActiveRecord object (Part 035), Concern สำหรับจัดโครงสร้างโมเดลขนาดใหญ่
(Part 036), multi-model form เต็มรูปแบบ (Part 037), pagination/sorting/filtering แบบมืออาชีพ
(Part 038), และ I18n เต็มรูปแบบ (Part 039) จนมาถึงโปรเจกต์รวบยอด **Shop Manager** ใน Part นี้ที่
บังคับให้ใช้ทุกเทคนิคเหล่านั้นร่วมกันจริง ไม่ใช่แค่เรียงต่อกันแบบแยกส่วน

สิ่งที่สำคัญที่สุดที่ควรพาติดตัวไปจากเฟสนี้คือ **ความเข้าใจว่า "ความถูกต้องของข้อมูล" ต้องได้รับ
การป้องกันหลายชั้นพร้อมกันเสมอ** — ตั้งแต่ระดับฐานข้อมูล (foreign key, unique index, `NOT
NULL`), ระดับ Model (validation, custom validator, callback ที่ทำงานภายใน transaction), ไปจนถึง
ระดับ Controller (strong parameters, allowlist สำหรับ sorting) — ไม่มีชั้นไหนที่ "เพียงพอเอง
ตามลำพัง" การขายเกินสต็อกใน Step 395 คือตัวอย่างที่ชัดเจนที่สุดของ Part นี้: ถ้าขาดชั้นใดชั้น
หนึ่งไป (transaction, row lock, หรือ `throw :abort`) ระบบจะมีช่องโหว่ให้สต็อกติดลบได้ทันทีเมื่อมี
การใช้งานพร้อมกันหลายคน

Phase 4 ยังทิ้งช่องว่างไว้อย่างตั้งใจ 4 เรื่องใหญ่ (authentication, automated test, การชำระเงิน,
rate limiting) ตามที่วิเคราะห์ไว้ใน Step 400 — เช่นเดียวกับ Phase 3 ที่ปิดท้ายด้วยช่องว่าง 3 เรื่อง
ที่ Phase 4 เพิ่งแก้ไปหนึ่งเรื่อง (pagination/performance ผ่าน Part 038) ช่องว่างที่เหลือคือแผนที่
ที่ชี้ทางไปสู่เฟสถัดไปของหลักสูตรอย่างชัดเจน

**ต่อไป (Part 041 — เปิด Phase 5: Authentication & Authorization):** เราจะแก้ช่องโหว่ที่ใหญ่ที่สุด
ของทั้ง Mini Blog (Phase 3) และ Shop Manager (Phase 4) นั่นคือการที่**ใครก็ทำอะไรก็ได้โดยไม่มีการ
ยืนยันตัวตนเลย** — เริ่มจาก **Authentication เบื้องต้น**: `has_secure_password` (ใช้ `bcrypt`
เข้ารหัสรหัสผ่าน), การสร้างระบบ session-based login/logout ด้วยมือทั้งหมด (ยังไม่ใช้ gem สำเร็จรูป
อย่าง Devise ในตอนนี้ เพื่อให้เข้าใจกลไกเบื้องหลังก่อน — Devise จะเรียนใน Part 042), และการป้องกัน
route ด้วย `before_action :require_login` — เทคนิคเหล่านี้จะนำไปใช้อุดช่องโหว่ authentication ที่
Part 030 และ Part 040 ทิ้งไว้ทั้งคู่โดยตรง
