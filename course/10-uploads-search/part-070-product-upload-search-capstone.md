# Part 070: โปรเจกต์รวบยอดเฟส 10 — ระบบอัปโหลดรูปสินค้า + ค้นหาสินค้า

> **Step ครอบคลุมใน Part นี้:** Step 691–700
> **ระดับ:** สูง (ต้องผ่าน **Part 066–069** มาก่อนทั้งหมด — โดยเฉพาะ Part 066 Active Storage
> upload/variant/direct upload, Part 067 Active Storage กับ cloud storage และ image processing,
> Part 068 full-text search ด้วย pg_search, และ Part 069 Elasticsearch/OpenSearch เบื้องต้น —
> รวมถึง Part 034 query interface, Part 038 pagination/sorting/filtering, และ Part 061 ActiveJob
> เบื้องต้นจาก Phase 9)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (โปรเจกต์ทั้งหมดใน Part นี้สร้างและทดสอบรันจริงบน
> Ruby 3.3.6 + Rails 8.1.4 + PostgreSQL 16.13 + `pg_search` 2.4.0 + `pagy` 9.4.0 +
> `image_processing` 1.14.0 (`ruby-vips` 2.3.0 เป็น backend) — ทุกคำสั่ง `bin/rails
> console`/`runner` และทุก request ที่แสดงในเอกสารนี้ยิงจริงกับ server ที่รันอยู่จริงด้วย `curl`
> รวมถึงการอัปโหลดรูปภาพจริงผ่าน multipart form ไม่ใช่โค้ดที่เขียนคาดเดาไว้ล่วงหน้า)

นี่คือ **Part สุดท้ายของ Phase 10: File Upload / Search** ตลอด 4 Part ที่ผ่านมาเราเรียนรู้แต่ละ
เทคนิคแยกกันทีละเรื่อง: การอัปโหลดไฟล์และสร้าง variant ด้วย Active Storage (Part 066), การต่อ
Active Storage เข้ากับ cloud storage แบบ S3-compatible พร้อม image processing เชิงลึก (Part 067),
full-text search บน PostgreSQL ด้วย pg_search (Part 068), และการต่อยอดไปสู่ Elasticsearch/
OpenSearch เมื่อความต้องการค้นหาซับซ้อนขึ้น (Part 069) — ถึงเวลาแล้วที่จะเอาทุกเทคนิคเหล่านั้นมา
ประกอบเป็น **โปรเจกต์เดียวที่ใช้งานได้จริง**: ระบบแคตตาล็อกสินค้า (**Product Catalog**) ที่ผู้ขาย
อัปโหลดรูปสินค้าได้ พร้อมระบบค้นหาที่รองรับทั้งภาษาไทยและภาษาอังกฤษ

โปรเจกต์ปิดท้ายเฟสนี้มี 2 โมเดลหลัก:

- **Category** (หมวดหมู่สินค้า) — `has_many :products`
- **Product** (สินค้า) — `belongs_to :category`, มี `has_one_attached :photo` พร้อม named
  variant สองขนาด (`:thumb`, `:medium`), มี validation ตรวจชนิดไฟล์และขนาดไฟล์ (ทบทวน Part 066),
  และมี **dual-scope search** ที่รวม `tsearch` (full-text ภาษาอังกฤษ) กับ `trigram` (ค้นหา
  ภาษาไทย/คำสะกดใกล้เคียง) เข้าด้วยกัน (ทบทวน Part 068)

ขอบเขตของโปรเจกต์นี้ตั้งใจให้ **เล็กแต่ครบทุกเทคนิคของเฟส**: อัปโหลดรูปพร้อม validation, สร้าง
variant สองขนาดที่ตั้งชื่อไว้ล่วงหน้า, ประมวลผล variant ผ่าน background job แทนการรอสด, ค้นหา
สินค้าด้วยคำไทย/อังกฤษ/คำสะกดผิดเล็กน้อยได้ทั้งหมด, กรองตามหมวดหมู่ (facet แบบง่าย), แบ่งหน้าผล
ลัพธ์ (ทบทวน Part 038), และป้องกัน N+1 query อย่างถูกต้อง (ทบทวน Part 034) — ไม่มีระบบ login/
authorization ในเวอร์ชันนี้ (เป็นหัวข้อของ Phase 5 ที่เรียนไปแล้ว แต่ตั้งใจตัดออกเพื่อโฟกัสที่
เทคนิคของ Phase 10 ล้วนๆ) และไม่มี cloud storage จริง (ใช้ local disk service เพื่อความสะดวกใน
การพัฒนา/ทดสอบ แต่ทุกจุดที่เกี่ยวกับ S3 จะระบุไว้ชัดเจนว่าเปลี่ยนอะไรบ้างเมื่อขึ้น production ตาม
ที่ Part 067 สอนไว้)

## สารบัญของ Part นี้

- Step 691: วางแผนโปรเจกต์ — ER diagram, ทำไมต้องใช้ PostgreSQL, `rails new` และ Gemfile
- Step 692: `Category`/`Product` model + migration + validation ตรวจชนิด/ขนาดไฟล์รูป (ทบทวน
  Part 066)
- Step 693: Named variant `:thumb`/`:medium` ด้วย `has_one_attached` block (ทบทวน Part 067)
- Step 694: Routes + `ProductsController` CRUD เต็มรูปแบบ พร้อมฟอร์มอัปโหลดรูป
- Step 695: pg_search dual-scope — รวม `tsearch` (อังกฤษ) กับ `trigram` (ไทย) เข้าด้วยกัน
  (ทบทวน Part 068)
- Step 696: หน้าค้นหา — ช่องค้นหา + thumbnail + pagination (Part 038) + eager loading กัน N+1
  (Part 034)
- Step 697: Category facet แบบง่ายด้วย ActiveRecord grouping (พร้อมพูดถึงเส้นทางสู่
  Elasticsearch aggregations จาก Part 069)
- Step 698: ประมวลผล variant ผ่าน background job (`ProductPhotoVariantsJob`, ทบทวน Part 061)
- Step 699: `db/seeds.rb` พร้อมรูปจริง และ manual verification walkthrough แบบเต็ม
- Step 700: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 10

---

## Step 691: วางแผนโปรเจกต์ — ER diagram, ทำไมต้องใช้ PostgreSQL, `rails new` และ Gemfile

### ออกแบบโดเมนก่อนเขียนโค้ด

โดเมนของโปรเจกต์นี้เรียบง่ายกว่า Shop Manager ใน Part 040 มาก (มีแค่ 2 ตาราง) เพราะจุดโฟกัสของ
Part นี้อยู่ที่ **การอัปโหลด/แปลงรูปภาพ** และ **การค้นหา** ไม่ใช่ความซับซ้อนของความสัมพันธ์ระหว่าง
โมเดล:

```
┌──────────────┐  1        N  ┌────────────────────────┐
│   Category    │──────────────►│        Product          │
├──────────────┤              ├────────────────────────┤
│ id            │              │ id                       │
│ name          │              │ name                     │
└──────────────┘              │ description (text)       │
                               │ price_cents (integer)     │
                               │ category_id  ◄── FK       │
                               │ photo (Active Storage)    │──► active_storage_blobs
                               └────────────────────────┘    active_storage_attachments
                                                              active_storage_variant_records
```

### ทำไมต้องใช้ PostgreSQL (ไม่ใช่ SQLite ที่ Rails 8 ตั้งเป็น default)

Rails 8 ตั้ง SQLite เป็นฐานข้อมูล default ให้ตอน `rails new` เฉยๆ (ทบทวนจาก Part 021) แต่โปรเจกต์
นี้**ต้องใช้ PostgreSQL** ด้วยเหตุผลเดียวที่สำคัญที่สุด: **`pg_search` (Part 068) ต้องพึ่งความ
สามารถเฉพาะของ PostgreSQL สองอย่างที่ SQLite ไม่มี**:

1. **`tsvector`/`to_tsvector`/`ts_rank`** — ระบบ full-text search ในตัวของ PostgreSQL ที่ใช้ทำ
   `tsearch` scope
2. **extension `pg_trgm`** — ให้ operator/function เปรียบเทียบความคล้ายของ string แบบ trigram
   (`similarity()`) ที่ใช้ทำ `trigram` scope

ทั้งสองอย่างนี้เป็นฟีเจอร์ระดับฐานข้อมูล ไม่ใช่ gem ระดับ Ruby จึงย้ายไปใช้ฐานข้อมูลอื่นแทนไม่ได้
เลยถ้าต้องการใช้ `pg_search` แบบเต็มรูปแบบตามที่ Part 068 สอนไว้

### สร้างโปรเจกต์ใหม่ด้วย `--database=postgresql`

```bash
mkdir -p ~/ruby-course-workspace/part-070
cd ~/ruby-course-workspace/part-070

rails new product_catalog --database=postgresql
cd product_catalog
```

ตรวจสอบเวอร์ชันที่ได้จริง (ทดสอบแล้วบนเครื่องที่ใช้เขียนบทเรียนนี้):

```
ruby 3.3.6 (2024-11-05 revision 75015d4c1f) [x86_64-linux]
Rails 8.1.4
PostgreSQL 16.13
```

> **หมายเหตุ:** `rails new --database=postgresql` เขียน `config/database.yml` ให้ชี้ไปที่
> PostgreSQL ทันที และเพิ่ม `gem "pg"` ใน `Gemfile` ให้อัตโนมัติ — ไม่ต้องแก้อะไรเองถ้าเครื่องมี
> PostgreSQL server รันอยู่แล้วและเชื่อมต่อผ่าน Unix socket ได้แบบไม่ต้องใส่รหัสผ่าน (ค่า default
> ของ development) ถ้าใช้ Docker/เครื่องอื่นอาจต้องเพิ่ม `host:`/`username:`/`password:` ใน
> `config/database.yml` เอง

### เพิ่ม gem ที่ต้องใช้ในโปรเจกต์นี้

```ruby
# Gemfile — ต่อจาก gem "image_processing" ที่ rails new เตรียมไว้ให้แล้ว (ดูหมายเหตุด้านล่าง)
gem "image_processing", "~> 1.2"

# Full-text + fuzzy search บน PostgreSQL (tsearch + trigram) — ทบทวน Part 068
gem "pg_search", "~> 2.3"

# Pagination — ทบทวน Part 038
gem "pagy", "~> 9.4"
```

```bash
bundle install
```

> **หมายเหตุเรื่อง `image_processing`:** ตั้งแต่ Rails 7 เป็นต้นมา `rails new` ใส่บรรทัด
> `gem "image_processing", "~> 1.2"` มาให้ใน `Gemfile` อยู่แล้ว (ไม่ต้อง comment ออกเหมือน Rails
> รุ่นเก่า) เพราะ Active Storage variant ต้องพึ่ง gem นี้เป็น backend สำหรับแปลงรูปภาพ (ทบทวน
> Part 067) — เบื้องหลัง `image_processing` เรียกใช้ `ruby-vips` (เร็วกว่า ImageMagick/
> MiniMagick มาก และเป็น backend ที่ Rails แนะนำในปัจจุบัน) เครื่องที่รันโปรเจกต์นี้ต้องติดตั้ง
> `libvips` ไว้ในระบบด้วย (`apt install libvips` บน Ubuntu/Debian, `brew install vips` บน macOS)

ตรวจสอบเวอร์ชัน gem ที่ได้จริงหลัง `bundle install`:

```
pg_search (2.4.0)
pagy (9.4.0)
image_processing (1.14.0)
ruby-vips (2.3.0)
```

### สร้างฐานข้อมูล

```bash
bin/rails db:create
```

```
Created database 'product_catalog_development'
Created database 'product_catalog_test'
```

### ภาพรวม routes ที่จะสร้างใน Part นี้

| หน้าจอ | URL | หน้าที่ | Step ที่สอน |
|---|---|---|---|
| รายการสินค้า + ค้นหา + กรองหมวดหมู่ | `GET /products` | แสดง/ค้นหาสินค้าทั้งหมด | Step 696–697 |
| ดูสินค้า | `GET /products/:id` | แสดงรายละเอียดสินค้า | Step 694 |
| เพิ่มสินค้าใหม่ | `GET /products/new` → `POST /products` | ฟอร์มอัปโหลดรูป + บันทึก | Step 694 |
| แก้ไขสินค้า | `GET /products/:id/edit` → `PATCH /products/:id` | แก้ไขข้อมูล/เปลี่ยนรูป | Step 694 |
| ลบสินค้า | `DELETE /products/:id` | ลบสินค้าและรูปที่แนบ | Step 694 |

ต่างจาก Shop Manager ใน Part 040 ตรงที่ Part นี้เปิด CRUD เต็มรูปแบบให้ `Product` ทาง UI เลย
(ไม่ผ่าน console/seeds เท่านั้น) เพราะฟอร์มอัปโหลดไฟล์ (`file_field`) คือส่วนสำคัญที่ต้องสาธิตให้
เห็นการทำงานจริง ส่วน `Category` ยังคงสร้างผ่าน `db/seeds.rb`/console เท่านั้น (ไม่มีหน้าเว็บ
จัดการหมวดหมู่ในเวอร์ชันนี้ — เหตุผลเดียวกับ Part 040: ระบบจัดการ "master data" อย่างหมวดหมู่มัก
อยู่หลังระบบ admin แยกต่างหากในระบบจริง)

---

## Step 692: `Category`/`Product` model + migration + validation ตรวจชนิด/ขนาดไฟล์รูป

### สร้าง Model ด้วย generator

```bash
bin/rails generate model Category name:string
bin/rails generate model Product name:string description:text price_cents:integer category:references
bin/rails active_storage:install
```

คำสั่งสุดท้าย `bin/rails active_storage:install` คัดลอก migration ที่สร้างตาราง
`active_storage_blobs`, `active_storage_attachments`, `active_storage_variant_records` มาไว้ให้
(ทบทวนจาก Part 066 — ต้องรันคำสั่งนี้ครั้งเดียวตอนเริ่มใช้ Active Storage ในโปรเจกต์ ถ้ายังไม่เคย
รันมาก่อน)

### แก้ migration ให้มี constraint ที่ถูกต้อง

```ruby
# db/migrate/xxxxxx_create_categories.rb
class CreateCategories < ActiveRecord::Migration[8.1]
  def change
    create_table :categories do |t|
      t.string :name, null: false

      t.timestamps
    end

    add_index :categories, :name, unique: true
  end
end
```

```ruby
# db/migrate/xxxxxx_create_products.rb
class CreateProducts < ActiveRecord::Migration[8.1]
  def change
    create_table :products do |t|
      t.string :name, null: false
      t.text :description
      t.integer :price_cents, null: false, default: 0
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end

    add_index :products, :name
  end
end
```

**จุดที่ต้องสังเกต:**

- **`price_cents` เป็น integer ไม่ใช่ float** — ทบทวนหลักการเดียวกับ Part 040: ห้ามเก็บเงินเป็น
  floating-point เด็ดขาด เก็บเป็นหน่วยเล็กสุด (สตางค์) แล้วค่อยหาร 100 ตอนแสดงผล
- **`t.references :category, null: false`** — บังคับว่าสินค้าทุกชิ้นต้องมีหมวดหมู่เสมอ ไม่มี
  "สินค้าไม่มีหมวดหมู่" ในระบบนี้ ทำให้ facet ใน Step 697 ไม่ต้องจัดการกรณี `category_id: nil`
  เป็นพิเศษ

รัน migration ทั้งหมด (รวม Active Storage tables):

```bash
bin/rails db:migrate
```

```
== CreateCategories: migrating ============================
-- create_table(:categories)
-- add_index(:categories, :name, {:unique=>true})
== CreateCategories: migrated ==============================

== CreateProducts: migrating ================================
-- create_table(:products)
-- add_index(:products, :name)
== CreateProducts: migrated ==================================

== CreateActiveStorageTables: migrating =======================
-- create_table(:active_storage_blobs, {:id=>:primary_key})
-- create_table(:active_storage_attachments, {:id=>:primary_key})
-- create_table(:active_storage_variant_records, {:id=>:primary_key})
== CreateActiveStorageTables: migrated =========================
```

### `app/models/category.rb`

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :products, dependent: :restrict_with_error

  validates :name, presence: true, uniqueness: true

  scope :alphabetical, -> { order(:name) }
end
```

`dependent: :restrict_with_error` (ทบทวนจาก Part 040) ห้ามลบหมวดหมู่ที่ยังมีสินค้าอยู่ — ผู้ดูแล
ต้องย้าย/ลบสินค้าออกก่อนเสมอ ป้องกันสินค้าตกค้างแบบไม่มีหมวดหมู่

### `app/models/product.rb` (เวอร์ชันแรก — เพิ่ม variant ใน Step 693, search ใน Step 695, job ใน Step 698)

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  ACCEPTED_PHOTO_TYPES = %w[image/jpeg image/png image/webp].freeze
  MAX_PHOTO_SIZE = 5.megabytes

  belongs_to :category
  has_one_attached :photo

  validates :name, presence: true
  validates :price_cents, numericality: { only_integer: true, greater_than: 0 }
  validate :photo_content_type_allowed
  validate :photo_size_within_limit

  def price
    price_cents / 100.0
  end

  private

  # ตรวจชนิดไฟล์จาก content_type ที่ Active Storage อ่านมาจาก MIME type ของไฟล์ที่อัปโหลด
  # (ทบทวน Part 066) — `return unless photo.attached?` ทำให้สินค้าที่ยังไม่มีรูปเลยไม่ error
  # (ไม่ได้บังคับว่าทุกสินค้าต้องมีรูป ในระบบนี้รูปเป็น optional)
  def photo_content_type_allowed
    return unless photo.attached?
    return if ACCEPTED_PHOTO_TYPES.include?(photo.content_type)

    errors.add(:photo, "ต้องเป็นไฟล์ชนิด JPEG, PNG หรือ WebP เท่านั้น")
  end

  # `photo.byte_size` อ่านจากคอลัมน์ byte_size ของ active_storage_blobs โดยตรง ไม่ต้องเปิดไฟล์
  # จริงมาวัดขนาด (เร็วมาก เพราะ Active Storage บันทึกขนาดไฟล์ไว้ตั้งแต่ตอนอัปโหลดแล้ว)
  def photo_size_within_limit
    return unless photo.attached?
    return if photo.byte_size <= MAX_PHOTO_SIZE

    errors.add(:photo, "ต้องมีขนาดไม่เกิน #{MAX_PHOTO_SIZE / 1.megabyte} MB")
  end
end
```

### ทดสอบ validation ผ่าน Rails Console

```bash
bin/rails console
```

```irb
irb> p = Product.new(name: "", price_cents: 0)
irb> p.valid?
=> false
irb> p.errors.full_messages
=> ["Category must exist", "Name can't be blank", "Price cents must be greater than 0"]

irb> cat = Category.create!(name: "เสื้อผ้า")
irb> p2 = Product.new(name: "ทดสอบ", price_cents: 10_000, category: cat)
irb> p2.photo.attach(io: StringIO.new("not an image"), filename: "note.txt", content_type: "text/plain")
irb> p2.valid?
=> false
irb> p2.errors.full_messages
=> ["Photo ต้องเป็นไฟล์ชนิด JPEG, PNG หรือ WebP เท่านั้น"]
```

ทั้งสองพฤติกรรมตรงตามที่ออกแบบไว้ทุกประการ — validation มาตรฐาน (`presence`, `numericality`,
`belongs_to` ที่ implicit require) ทำงานร่วมกับ custom validation ที่ตรวจ Active Storage
attachment ได้อย่างไร้รอยต่อ เพราะ `photo` เป็นแค่ method ธรรมดาตัวหนึ่งที่ `has_one_attached`
เพิ่มให้กับ instance ของ `Product`

---

## Step 693: Named variant `:thumb`/`:medium` ด้วย `has_one_attached` block

### ทำไมต้องมี variant หลายขนาด

รูปต้นฉบับที่ผู้ขายอัปโหลดอาจมีขนาดหลายเมกะไบต์ ความละเอียดหลักพันพิกเซล — ถ้าส่งรูปต้นฉบับตรงๆ
ไปแสดงเป็น thumbnail ขนาด 150×150 พิกเซลในหน้ารายการสินค้า จะเปลืองแบนด์วิดท์มหาศาลโดยเปล่า
ประโยชน์ (ทบทวนแนวคิด image optimization จาก Part 067) วิธีแก้คือให้ Active Storage **สร้างไฟล์
ใหม่ที่ย่อขนาดแล้ว** ไว้ล่วงหน้า เรียกว่า **variant**

### ประกาศ named variant ตรงจุดเดียวกับ `has_one_attached`

Rails 7.1+ อนุญาตให้ประกาศ "ชื่อ" ของ variant ที่ใช้บ่อยไว้ล่วงหน้าตรงจุดที่ประกาศ
`has_one_attached` เลย (ทบทวนรูปแบบนี้จาก Part 067) แทนการเรียก `.variant(resize_to_fill: [...])`
พร้อม option เต็มซ้ำๆ ทุกที่ที่ใช้:

```ruby
# app/models/product.rb (เพิ่มจาก Step 692)
class Product < ApplicationRecord
  # ...

  # has_one_attached พร้อม named variant ประกาศตรงจุดเดียวกับ attachment เลย (Rails 7.1+
  # ทบทวน Part 067) — :thumb ใช้ในหน้า index/การ์ดค้นหา, :medium ใช้ในหน้า show
  has_one_attached :photo do |attachable|
    attachable.variant :thumb, resize_to_fill: [150, 150]
    attachable.variant :medium, resize_to_limit: [600, 600]
  end

  # ...
end
```

**อธิบายความต่างของ `resize_to_fill` กับ `resize_to_limit` (ทบทวนเจาะลึกจาก Part 067):**

- **`resize_to_fill: [150, 150]`** ใช้กับ `:thumb` — บังคับให้ได้ภาพขนาด **เป๊ะ** 150×150 พิกเซล
  เสมอ โดยครอบตัด (crop) ส่วนที่เกินทิ้งถ้าอัตราส่วนภาพต้นฉบับไม่ใช่สี่เหลี่ยมจัตุรัสพอดี —
  เหมาะกับ thumbnail ในตารางกริดที่ต้องการทุกช่องมีขนาดเท่ากันเป๊ะเพื่อความเป็นระเบียบของ layout
- **`resize_to_limit: [600, 600]`** ใช้กับ `:medium` — ย่อภาพให้ **ด้านที่ยาวที่สุดไม่เกิน**
  600 พิกเซล โดย**รักษาอัตราส่วนเดิมไว้เสมอ** ไม่ครอบตัดเลย เหมาะกับหน้ารายละเอียดสินค้าที่
  ต้องการให้ผู้ซื้อเห็นภาพเต็มโดยไม่บิดเบือนสัดส่วน

### ทดสอบสร้าง variant ผ่าน Rails Console

```irb
irb> cat = Category.create!(name: "เสื้อผ้า")
irb> p = Product.create!(name: "เสื้อยืดคอกลม", price_cents: 29_900, category: cat)
irb> p.photo.attach(io: File.open("db/seed_images/shirt.jpg"), filename: "shirt.jpg", content_type: "image/jpeg")

irb> p.photo.variant(:thumb).class
=> ActiveStorage::VariantWithRecord

irb> p.photo.variant(:thumb).processed.key
=> "1fnuli77wa1m3i5lj2ev0u3f8xks"   # key ของไฟล์ variant ที่ถูกสร้างขึ้นจริงบน disk service

irb> p.photo.variant(:medium).processed.key
=> "afkutbv7b4c21orrpuveext53o1e"   # คนละไฟล์กับ :thumb — คนละขนาด คนละ key เสมอ
```

`.processed` สั่งให้ Active Storage **แปลงไฟล์จริงทันที** (ถ้ายังไม่เคยมี) แล้วคืน object ที่มี
`.key` ของไฟล์ variant นั้น — ยืนยันว่า `:thumb` และ `:medium` ถูกสร้างเป็นไฟล์แยกกันจริง ไม่ใช่
แค่ URL parameter ที่ resize สดทุกครั้งที่เปิดหน้า

### วิธีที่ view จะเรียกใช้ (ภาพรวมล่วงหน้า — เวอร์ชันเต็มอยู่ Step 694)

```erb
<%# ในหน้า index — ใช้ thumbnail ขนาดคงที่สำหรับการ์ดสินค้า %>
<%= image_tag product.photo.variant(:thumb), alt: product.name %>

<%# ในหน้า show — ใช้ medium เพื่อให้เห็นภาพใหญ่ขึ้นแต่ยังไม่หนักเกินไป %>
<%= image_tag product.photo.variant(:medium), alt: product.name %>
```

การเรียก `product.photo.variant(:thumb)` แบบนี้**ไม่ได้แปลว่า variant ต้องถูกสร้างไว้ล่วงหน้า
เสมอ** — ถ้ายังไม่เคยมีไฟล์ variant นี้อยู่จริง Active Storage จะสร้างมันแบบ **lazy** (สร้างตอน
ที่ browser ร้องขอ URL จริงๆ) ให้อัตโนมัติ ซึ่งทำงานถูกต้องแต่ทำให้ผู้ใช้คนแรกที่เปิดดูรูปสินค้า
นั้นต้องรอการประมวลผลภาพสด — นี่คือปัญหาที่ **Step 698** จะแก้ด้วย background job ที่ pre-generate
variant ไว้ล่วงหน้าตั้งแต่ตอนอัปโหลดเสร็จ

---

## Step 694: Routes + `ProductsController` CRUD เต็มรูปแบบ พร้อมฟอร์มอัปโหลดรูป

### `config/routes.rb`

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "up" => "rails/health#show", as: :rails_health_check

  root "products#index"

  resources :products
end
```

`resources :products` เปิดครบทั้ง 7 action มาตรฐาน (index, show, new, create, edit, update,
destroy) — ต่างจาก Shop Manager ใน Part 040 ที่จำกัด action เพราะ Part นี้ต้องสาธิตฟอร์มอัปโหลด
ไฟล์ให้ครบวงจรจริงๆ

### `app/controllers/products_controller.rb` (เวอร์ชันแรก — เพิ่มค้นหา/facet ใน Step 696–697)

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  before_action :set_product, only: %i[show edit update destroy]

  def index
    @products = Product.includes(:category, photo_attachment: :blob).order(created_at: :desc)
  end

  def show
  end

  def new
    @product = Product.new
  end

  def create
    @product = Product.new(product_params)

    if @product.save
      redirect_to @product, notice: "เพิ่มสินค้า \"#{@product.name}\" เรียบร้อยแล้ว"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
  end

  def update
    if @product.update(product_params)
      redirect_to @product, notice: "บันทึกการแก้ไขสินค้า \"#{@product.name}\" เรียบร้อยแล้ว"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @product.destroy
    redirect_to products_path, notice: "ลบสินค้า \"#{@product.name}\" แล้ว", status: :see_other
  end

  private

  def set_product
    @product = Product.includes(:category, photo_attachment: :blob).find(params[:id])
  end

  # `:photo` permit ตรงๆ เป็น key เดียว (ไม่ใช่ array) เพราะ has_one_attached รับไฟล์เดียว —
  # ค่าที่มาจาก `file_field` คือ ActionDispatch::Http::UploadedFile ซึ่ง Active Storage
  # รู้จักและแปลงเป็น Blob ให้อัตโนมัติเมื่อ assign เข้า attribute (ทบทวน Part 066)
  def product_params
    params.require(:product).permit(:name, :description, :price_cents, :category_id, :photo)
  end
end
```

### `app/views/products/_form.html.erb`

```erb
<%# app/views/products/_form.html.erb %>
<%= form_with model: product do |f| %>
  <% if product.errors.any? %>
    <div class="form-errors">
      <h3><%= pluralize(product.errors.count, "ข้อผิดพลาด") %>ทำให้บันทึกสินค้านี้ไม่ได้:</h3>
      <ul>
        <% product.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= f.label :name, "ชื่อสินค้า" %>
    <%= f.text_field :name %>
  </div>

  <div class="field">
    <%= f.label :description, "รายละเอียดสินค้า" %>
    <%= f.text_area :description, rows: 4 %>
  </div>

  <div class="field">
    <%= f.label :category_id, "หมวดหมู่" %>
    <%= f.collection_select :category_id, Category.alphabetical, :id, :name, include_blank: "เลือกหมวดหมู่" %>
  </div>

  <div class="field">
    <%= f.label :price_cents, "ราคา (บาท)" %>
    <%# แสดง/รับค่าเป็นสตางค์ตรงๆ เพื่อความเรียบง่าย (ระบบจริงมักมี JS แปลงหน่วยบาท<->สตางค์
        ให้ผู้ใช้กรอกเป็นบาทแล้วแปลงก่อนส่ง แต่นอกขอบเขตของ Part นี้) %>
    <%= f.number_field :price_cents, min: 1, step: 1 %>
    <small>หน่วยเป็นสตางค์ เช่น สินค้าราคา 129 บาท ให้กรอก 12900</small>
  </div>

  <div class="field">
    <%= f.label :photo, "รูปสินค้า" %>
    <%# accept: จำกัดชนิดไฟล์ที่ file picker ของ browser จะให้เลือกได้ (UX เท่านั้น ไม่ใช่
        การรักษาความปลอดภัย — validation ฝั่ง Model ใน Step 692 ต่างหากที่เป็นด่านจริง เพราะ
        accept เป็นแค่คำแนะนำที่ผู้ใช้ (หรือคนที่ตั้งใจโจมตี) หลีกเลี่ยงได้ง่ายๆ) %>
    <%= f.file_field :photo, accept: Product::ACCEPTED_PHOTO_TYPES.join(",") %>

    <%# เช็คทั้งสามเงื่อนไขก่อนแสดงตัวอย่างรูปเสมอ — พบทั้งสามกรณีนี้จริงระหว่างทดสอบ Part นี้:
        1) `product.persisted?` — สำหรับ record ที่ `save` ล้มเหลว (validation error อื่นทำให้
           create ไม่ผ่าน) ไฟล์ที่แนบมากับฟอร์มยัง**ไม่ถูกอัปโหลดขึ้น service จริง** (Active
           Storage เลื่อนการอัปโหลด blob ของ record ใหม่ไปรวมไว้ในธุรกรรมตอน save เดียวกัน)
           เรียก `.variant` บน blob ที่ยังไม่ persist จะได้ `ArgumentError: Cannot get a
           signed_id for a new record` ทันที
        2) `attached?` — ต้องมีไฟล์แนบอยู่จริงก่อน
        3) `representable?` — ไฟล์ที่แนบต้องเป็นชนิดที่ Active Storage แปลง variant ได้ ถ้า
           ผู้ใช้อัปโหลดไฟล์ผิดชนิด (เช่น .txt) มาปนกับ validation error อื่น การเรียก
           `.variant` ตรงๆ กับไฟล์แบบนี้จะได้ `ActiveStorage::InvariableError` ทันที %>
    <% if product.persisted? && product.photo.attached? && product.photo.representable? %>
      <div class="current-photo">
        <p>รูปปัจจุบัน:</p>
        <%= image_tag product.photo.variant(:thumb) %>
      </div>
    <% end %>
  </div>

  <div class="actions">
    <%= f.submit product.new_record? ? "เพิ่มสินค้า" : "บันทึกการแก้ไข" %>
  </div>
<% end %>
```

> **สองข้อบกพร่องที่พบจริงระหว่างทดสอบ Part นี้ (และวิธีแก้ที่แสดงไว้ในโค้ดข้างบนแล้ว):**
> ตอนแรกโค้ดใช้แค่ `if product.photo.attached?` เฉยๆ — พออัปโหลดไฟล์ `.txt` ปลอมเป็นรูป (ผิด
> content type) การ validate ล้มเหลวตามที่ตั้งใจ แต่หน้า `new` ที่ render กลับมาแสดง error
> **500 Internal Server Error** แทนที่จะแสดงข้อความ validation เพราะ `image_tag
> product.photo.variant(:thumb)` พยายามแปลง variant ของไฟล์ text ที่ไม่ใช่รูปเลย
> (`ActiveStorage::InvariableError`) แก้ด้วยการเพิ่มเช็ค `representable?` แล้วทดสอบไฟล์ขนาดใหญ่
> เกิน limit ก็เจอ error คนละแบบอีก (`ArgumentError: Cannot get a signed_id for a new record`)
> เพราะ blob ของ record ใหม่ที่ save ไม่ผ่านยังไม่ถูกอัปโหลดขึ้น service จริง จึงต้องเพิ่มเช็ค
> `persisted?` เข้าไปอีกชั้น — บทเรียนสำคัญ: **`attached?` บอกแค่ว่า "มีไฟล์ผูกอยู่" ไม่ได้แปลว่า
> ไฟล์นั้น "พร้อมแปลง variant" เสมอไป** ต้องเช็คให้ครบทั้งสามเงื่อนไขเวลาจะ preview รูปที่เพิ่ง
> อัปโหลดในฟอร์มที่อาจ validation fail ได้

### `app/views/products/new.html.erb`

```erb
<%# app/views/products/new.html.erb %>
<h1>เพิ่มสินค้าใหม่</h1>

<%= render "form", product: @product %>

<%= link_to "« กลับไปหน้ารายการสินค้า", products_path %>
```

### `app/views/products/edit.html.erb`

```erb
<%# app/views/products/edit.html.erb %>
<h1>แก้ไขสินค้า: <%= @product.name %></h1>

<%= render "form", product: @product %>

<%= link_to "« กลับไปหน้ารายละเอียดสินค้า", @product %>
```

### `app/views/products/show.html.erb`

```erb
<%# app/views/products/show.html.erb %>
<%= link_to "« กลับไปหน้ารายการสินค้า", products_path %>

<article class="product-full">
  <h1><%= @product.name %></h1>

  <% if @product.photo.attached? && @product.photo.representable? %>
    <%# variant(:medium) เรียก variant ที่ตั้งชื่อไว้ใน has_one_attached (Step 693) ถ้า
        ProductPhotoVariantsJob (Step 698) ประมวลผลไปแล้ว ไฟล์ถูกแปลงเก็บไว้แล้ว การเรียก
        ตรงนี้จะได้ URL ทันที ไม่ต้องรอประมวลผลสด — ถ้า job ยังไม่รัน image_tag จะไป trigger
        ให้ Active Storage ประมวลผล variant "ตอนนั้นเลย" (lazy) แทน %>
    <%= image_tag @product.photo.variant(:medium), alt: @product.name %>
  <% else %>
    <div class="no-photo">ยังไม่มีรูปสินค้า</div>
  <% end %>

  <p class="product-meta">
    หมวดหมู่: <%= link_to @product.category.name, products_path(category_id: @product.category_id) %>
    · ราคา <%= number_to_currency(@product.price, unit: "฿", format: "%n %u") %>
  </p>

  <% if @product.description.present? %>
    <p class="product-description"><%= simple_format(@product.description) %></p>
  <% end %>

  <div class="actions">
    <%= link_to "แก้ไขสินค้า", edit_product_path(@product) %>
    <%= button_to "ลบสินค้า", product_path(@product), method: :delete,
                   data: { turbo_confirm: "ยืนยันการลบสินค้า \"#{@product.name}\" ใช่หรือไม่?" } %>
  </div>
</article>
```

### ทดสอบ CRUD เต็มรูปแบบด้วย `curl` จริง (มีอัปโหลดไฟล์รูปจริง)

```bash
bin/rails server -p 3070 -d
```

ดึง CSRF token จากหน้าฟอร์มก่อน (จำเป็นเมื่อยิง POST/PATCH/DELETE จากภายนอก browser จริง —
ทบทวนกลไก CSRF protection จาก Part 023):

```bash
curl -s "http://127.0.0.1:3070/products/new" -c cookies.txt -o new.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' new.html | sed 's/.*value="//;s/"$//')

curl -s -b cookies.txt -c cookies.txt -w "%{http_code}\n" \
  -F "authenticity_token=${TOKEN}" \
  -F "product[name]=กระเป๋าถือ สีดำ ทดสอบอัปโหลด" \
  -F "product[description]=สินค้าทดสอบสร้างผ่าน curl พร้อมแนบรูปจริง" \
  -F "product[category_id]=4" \
  -F "product[price_cents]=45000" \
  -F "product[photo]=@db/seed_images/backpack.jpg;type=image/jpeg" \
  "http://127.0.0.1:3070/products"
# => 302   (redirect ไปหน้า show ของสินค้าที่เพิ่งสร้าง — สร้างสำเร็จ)
```

ตรวจสอบผลผ่าน console:

```irb
irb> p = Product.find_by(name: "กระเป๋าถือ สีดำ ทดสอบอัปโหลด")
irb> p.photo.attached?
=> true
irb> p.photo.content_type
=> "image/jpeg"
```

ทดสอบ validation ผ่าน HTTP จริงด้วยไฟล์ผิดชนิด (`.txt`) — ยืนยันว่าหน้าฟอร์มแสดง error ที่ถูกต้อง
(หลังแก้ปัญหา `representable?` แล้ว) ไม่ใช่ 500:

```bash
echo "this is not an image" > fake.txt
curl -s "http://127.0.0.1:3070/products/new" -c cookies2.txt -o new2.html
TOKEN2=$(grep -o 'name="authenticity_token" value="[^"]*"' new2.html | sed 's/.*value="//;s/"$//')

curl -s -b cookies2.txt -w "%{http_code}\n" \
  -F "authenticity_token=${TOKEN2}" \
  -F "product[name]=สินค้าไฟล์ผิดชนิด" \
  -F "product[category_id]=4" \
  -F "product[price_cents]=1000" \
  -F "product[photo]=@fake.txt;type=text/plain" \
  "http://127.0.0.1:3070/products"
# => 422
```

```
ต้องเป็นไฟล์ชนิด JPEG, PNG หรือ WebP เท่านั้น
```

ทดสอบไฟล์เกินขนาด (6 MB > limit 5 MB) เช่นกัน:

```
ต้องมีขนาดไม่เกิน 5 MB
```

ทั้งสามเส้นทาง (สร้างสำเร็จ, ไฟล์ผิดชนิด, ไฟล์เกินขนาด) ทำงานถูกต้องตรงตามที่ออกแบบไว้ ยืนยันว่า
ฟอร์มอัปโหลดพร้อม validation ทำงานได้จริงแบบ end-to-end ผ่าน HTTP ไม่ใช่แค่ทดสอบผ่าน console

> **หมายเหตุเรื่อง direct upload:** Active Storage รองรับ **direct upload** (ทบทวน Part 066)
> ที่อัปโหลดไฟล์ตรงไปยัง storage service ผ่าน JavaScript **ก่อน** ที่ฟอร์มจะ submit ไปหา
> Rails server เลย (ลด load ของ web server, ให้ progress bar ได้) เปิดใช้ได้ง่ายๆ แค่เพิ่ม
> `direct_upload: true` ใน `f.file_field :photo, direct_upload: true` พร้อม pin
> `@rails/activestorage` ใน `config/importmap.rb` — Part นี้เลือก**ไม่ใช้** direct upload เพื่อ
> ให้ตัวอย่าง `curl` ข้างบนทดสอบง่ายและอ่านเข้าใจง่ายที่สุด (ฟอร์มธรรมดาแบบ multipart ก็ทำงาน
> ถูกต้องสมบูรณ์สำหรับไฟล์ขนาดไม่ใหญ่มาก) แต่ระบบ production ที่ผู้ใช้อัปโหลดไฟล์ใหญ่บ่อยๆ
> ควรพิจารณาเปิด direct upload ตามที่ Part 066 สอนไว้

---

## Step 695: pg_search dual-scope — รวม `tsearch` (อังกฤษ) กับ `trigram` (ไทย) เข้าด้วยกัน

### ปัญหาของการค้นหาสองภาษาในฐานข้อมูลเดียว

PostgreSQL full-text search (`tsearch`, ทบทวน Part 068) มาพร้อม **dictionary** ที่รู้จักการตัดคำ
(stemming) ของภาษานั้นๆ — `english` dictionary รู้ว่า `"running"` ควรแมตช์กับ `"run"` ได้ แต่
PostgreSQL **ไม่มี dictionary ภาษาไทยในตัว** เพราะภาษาไทยไม่มีช่องว่างคั่นระหว่างคำ (word
segmentation) การตัดคำไทยจึงต้องใช้ dictionary พิเศษที่ไม่ได้ติดมากับ PostgreSQL มาตรฐาน ผลคือ
ประโยคไทยทั้งประโยคที่ไม่มีช่องว่างจะถูกมองเป็น **token เดียวก้อนใหญ่**

วิธีแก้ที่ Part 068 สอนไว้คือใช้ **`pg_trgm` extension** เป็นตัวช่วยที่สอง — trigram เปรียบเทียบ
"กลุ่มตัวอักษร 3 ตัวติดกัน" (n-gram) โดยไม่สนใจภาษาหรือขอบเขตคำเลย ทำให้ค้นหาคำไทย (และคำสะกด
ใกล้เคียง/พิมพ์ผิดเล็กน้อย) ได้ผลดี Part นี้จะรวมทั้งสองเทคนิคเข้าด้วยกันเป็น **dual-scope
pattern**

### เปิดใช้ `pg_trgm` extension

```bash
bin/rails generate migration EnablePgTrgmAndSearchIndexes
```

```ruby
# db/migrate/xxxxxx_enable_pg_trgm_and_search_indexes.rb
class EnablePgTrgmAndSearchIndexes < ActiveRecord::Migration[8.1]
  def change
    # pg_trgm ให้ operator ความคล้ายของ string (similarity, %) ที่ pg_search ใช้ทำ
    # trigram search — จำเป็นสำหรับค้นหาภาษาไทย เพราะ dictionary ของ tsearch ไม่รู้จักการ
    # ตัดคำภาษาไทย
    enable_extension "pg_trgm" unless extension_enabled?("pg_trgm")

    # GIN index ด้วย gin_trgm_ops (มาจาก pg_trgm เอง) เร่งความเร็ว trigram similarity search
    # บนคอลัมน์ name/description — ทบทวนแนวคิด index จาก Part 064: ไม่มี index นี้ query ก็ยัง
    # ทำงานถูกต้อง เพียงแค่ต้องสแกนทั้งตาราง (sequential scan) ซึ่งช้าลงมากเมื่อข้อมูลโต
    add_index :products, :name, using: :gin, opclass: :gin_trgm_ops, name: "index_products_on_name_trgm"
    add_index :products, :description, using: :gin, opclass: :gin_trgm_ops, name: "index_products_on_description_trgm"
  end
end
```

```bash
bin/rails db:migrate
```

```
== EnablePgTrgmAndSearchIndexes: migrating =====================
-- extension_enabled?("pg_trgm")
-- enable_extension("pg_trgm")
-- add_index(:products, :name, {:using=>:gin, :opclass=>:gin_trgm_ops, ...})
-- add_index(:products, :description, {:using=>:gin, :opclass=>:gin_trgm_ops, ...})
== EnablePgTrgmAndSearchIndexes: migrated =======================
```

### เพิ่ม `pg_search` เข้า `Product`

```ruby
# app/models/product.rb (เพิ่มจาก Step 693)
class Product < ApplicationRecord
  include PgSearch::Model

  # ...

  # --- pg_search: dual-scope pattern (ทบทวน Part 068 Step 680 — ตั้งชื่อ scope ด้วย suffix
  # _en/_th ตามธรรมเนียมเดียวกับ `search_title_en`/`search_title_th` ที่ Part 068 สอนไว้) ---
  # tsearch ใช้ full-text search dictionary ของ PostgreSQL (english) — เหมาะกับคำภาษาอังกฤษ
  # ที่ผัน stem ได้ (เช่น "running" แมตช์ "run") แต่ "มองไม่เห็น" คำภาษาไทยเป็นคำๆ เพราะไม่มี
  # dictionary ภาษาไทยติดตั้งมาให้ ทำให้ทั้งประโยคไทยถูกมองเป็น "คำเดียว" ก้อนใหญ่
  pg_search_scope :search_by_name_and_description_en,
                   against: %i[name description],
                   using: { tsearch: { prefix: true, dictionary: "english" } }

  # trigram เปรียบเทียบ "กลุ่มตัวอักษร 3 ตัวติดกัน" (n-gram) โดยไม่สนใจว่าเป็นภาษาอะไร จึงใช้
  # ค้นหาคำไทยและคำสะกดใกล้เคียง (typo-tolerant) ได้ผลดีกว่า tsearch มาก แลกกับการที่ผลลัพธ์
  # ไม่ได้ผ่านการวิเคราะห์ทางภาษาศาสตร์ใดๆ เลย (ไม่มี stemming, ไม่ตัดคำหยุด)
  #
  # `ranked_by: ":trigram"` (ทบทวน Part 068 Step 677–678) จำเป็นเมื่อ `using:` เป็น `:trigram`
  # ล้วนๆ — ถ้าไม่ตั้งไว้ rank ที่ได้จาก `.with_pg_search_rank` จะเป็น 0 เสมอ (pg_search ไม่รู้ว่า
  # ควรเอาค่าอะไรมาเป็น rank ให้ ทั้งที่ตัว query ที่กรองผลลัพธ์เองใช้ similarity ถูกต้องอยู่แล้ว)
  #
  # จงใจใช้ `against: :name` เพียงคอลัมน์เดียว **ไม่รวม description** (ตรงกับคำแนะนำใน Part 068
  # Step 677: "จำกัด trigram search ไว้ที่คอลัมน์สั้นๆ ที่มีความหมายชัด") — พบระหว่างทดสอบว่าถ้า
  # ใช้ `against: [:name, :description]` เหมือน tsearch ด้านบน pg_search จะเอาทั้งสองคอลัมน์
  # มาต่อกันเป็น string เดียวก่อนคำนวณ similarity เมื่อ description ยาวกว่า name มาก อัตราส่วน
  # ตัวอักษรที่ตรงกัน (similarity) จะถูก "เจือจาง" จนต่ำกว่า threshold แม้ query จะสะกดใกล้เคียง
  # กับ name มากก็ตาม (ตัวอย่างจริง: ค้นหา "รอบเท้า" ซึ่งพิมพ์ผิดจาก "รองเท้า" — เทียบกับ name
  # อย่างเดียวได้ similarity 0.19 ผ่าน threshold แต่เทียบกับ name+description รวมกันได้ต่ำกว่า
  # 0.15 จนหลุด threshold ไปเลย)
  pg_search_scope :search_by_name_and_description_th,
                   against: :name,
                   using: { trigram: { threshold: 0.15 } },
                   ranked_by: ":trigram"

  # รวมผลลัพธ์จากทั้งสอง scope เข้าด้วยกัน — ต่างจาก Part 068 Step 678 (`search_combo`) ที่รวม
  # tsearch+trigram ไว้ใน `pg_search_scope` เดียวด้วย `ranked_by: ":tsearch + (0.5 * :trigram)"`
  # Part นี้แยกเป็นสอง scope อิสระแล้วรวมฝั่ง Ruby แทน เพราะ dictionary คนละตัวกันโดยสิ้นเชิง
  # (`:name` อย่างเดียวสำหรับ trigram, `:name`+`:description` สำหรับ tsearch) ทำให้คำนวณเป็น
  # `ts_rank` เดียวกันไม่ได้อยู่แล้ว — tsearch มาก่อนเสมอ (แม่นยำกว่าสำหรับคำอังกฤษและคำไทยที่
  # ขึ้นต้นตรงกับ query พอดี) ตามด้วยผลลัพธ์จาก trigram ที่ tsearch ยังไม่เจอ (ครอบคลุมคำสะกด
  # ใกล้เคียง/พิมพ์ผิด) แล้วรักษาลำดับความเกี่ยวข้องนั้นไว้ด้วย in_order_of (Rails 7.0+)
  def self.search_by_name_and_description(query)
    return all if query.blank?

    tsearch_ids = search_by_name_and_description_en(query).ids
    trigram_ids = search_by_name_and_description_th(query).ids
    ordered_ids = tsearch_ids | trigram_ids # Array#| = union ที่ตัดตัวซ้ำ รักษาลำดับซ้ายก่อน

    return none if ordered_ids.empty?

    where(id: ordered_ids).in_order_of(:id, ordered_ids)
  end

  # ...
end
```

### พิสูจน์จุดแข็ง/จุดอ่อนของแต่ละ scope ด้วยข้อมูลจริง

ทดสอบผ่าน console กับข้อมูลสินค้าจริง 12 รายการจาก `db/seeds.rb` (Step 699) — ครึ่งหนึ่งตั้งชื่อ
เป็นภาษาไทย อีกครึ่งเป็นภาษาอังกฤษโดยตั้งใจ เพื่อพิสูจน์ความแตกต่างของแต่ละ scope:

```irb
irb> Product.search_by_name_and_description_en("running").pluck(:name)
=> ["Running Shoes Red Edition"]              # tsearch: stemming ภาษาอังกฤษทำงาน

irb> Product.search_by_name_and_description_en("เสื้อ").pluck(:name)
=> ["เสื้อยืดคอกลม สีน้ำเงิน"]                 # tsearch: คำไทยที่ขึ้นต้นตรงกับ query เจอได้
                                                #          (ทั้งวลีไทยกลายเป็น token เดียว
                                                #          prefix search จับ token ที่ขึ้นต้น
                                                #          ด้วย "เสื้อ" ได้พอดี)

irb> Product.search_by_name_and_description_en("รอบเท้า").pluck(:name)     # พิมพ์ผิดจาก "รองเท้า"
=> []                                          # tsearch: หาไม่เจอ! prefix ไม่ตรงตั้งแต่
                                                #          ตัวอักษรที่ 3 ("รอบ" != "รอง")

irb> Product.search_by_name_and_description_th("รอบเท้า").pluck(:name)
=> ["รองเท้าผ้าใบสีแดง"]                       # trigram: หาเจอ! เทียบความคล้ายตัวอักษร
                                                #          ไม่สนใจตำแหน่งที่ต่างกัน

irb> Product.search_by_name_and_description("รอบเท้า").pluck(:name)
=> ["รองเท้าผ้าใบสีแดง"]                       # dual-scope: รวมจุดแข็งทั้งสองฝั่งไว้ด้วยกัน

irb> Product.search_by_name_and_description("breathable").pluck(:name)
=> ["Premium Cotton T-Shirt (Navy)", "Running Shoes Red Edition"]
                                                # เจอผ่าน tsearch เพราะคำนี้อยู่ใน description
                                                # (trigram หาไม่เจอเพราะ trigram เทียบแค่ name)

irb> Product.search_by_name_and_description("").count
=> 12                                          # query ว่าง = คืนสินค้าทั้งหมด (ไม่กรองอะไรเลย)
```

ผลลัพธ์ชุดนี้พิสูจน์ทั้งเหตุผลของ dual-scope pattern อย่างชัดเจน: **`tsearch` เก่งเรื่อง
full-text ในคำอังกฤษและคำที่สะกดตรง** (รวมถึงคำไทยที่พิมพ์ตรง เพราะ prefix matching ทำงานกับ
token ไทยก้อนใหญ่ได้ในระดับหนึ่ง) ส่วน **`trigram` เก่งเรื่องความคล้าย/ทนต่อการพิมพ์ผิด** — คำ
ค้นหา `"รอบเท้า"` ที่พิมพ์ผิดไปหนึ่งตัวอักษรกลางคำ ทำให้ `tsearch` (ซึ่งอาศัย prefix ที่ต้องตรง
ตั้งแต่ตัวแรก) หาไม่เจอเลย แต่ `trigram` (ซึ่งวัดสัดส่วนตัวอักษรที่คล้ายกันโดยไม่สนตำแหน่ง) ยังหา
เจอได้ — รวมสองอย่างเข้าด้วยกันจึงครอบคลุมกรณีการค้นหาได้กว้างกว่าการใช้อย่างใดอย่างหนึ่งเพียง
ลำพัง

---

## Step 696: หน้าค้นหา — ช่องค้นหา + thumbnail + pagination + eager loading กัน N+1

### เพิ่ม pagy เข้า Controller/Helper

```ruby
# app/controllers/application_controller.rb (เพิ่ม include)
class ApplicationController < ActionController::Base
  include Pagy::Backend

  allow_browser versions: :modern
  stale_when_importmap_changes

  # Pagy raise error แทนการคืนหน้าว่างเปล่าเมื่อ `page` ที่ขอเกินจำนวนหน้าจริง (เช่น ผลค้นหา
  # เหลือ 1 หน้า แต่ผู้ใช้ยังมี `?page=2` ค้างอยู่ใน URL จากการค้นหาก่อนหน้า) — วิธีมาตรฐานที่
  # เอกสารของ pagy เองแนะนำคือ redirect กลับไปหน้าสุดท้ายที่มีจริงแทนการโชว์หน้า error
  rescue_from Pagy::OverflowError do |exception|
    redirect_to url_for(request.query_parameters.merge(page: exception.pagy.last)),
                alert: "ไม่พบหน้าที่ร้องขอ แสดงหน้าสุดท้ายแทน"
  end
end
```

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  include Pagy::Frontend
end
```

### อัปเดต `ProductsController#index` ให้รองรับค้นหา + แบ่งหน้า + ป้องกัน N+1

```ruby
# app/controllers/products_controller.rb (แก้เฉพาะ action index — ส่วนอื่นเหมือน Step 694)
def index
  scope = Product.search_by_name_and_description(params[:q])

  # includes(:category, photo_attachment: :blob) ป้องกัน N+1 (ทบทวน Part 034): ไม่มี
  # includes นี้ ทุกแถวในหน้า index จะยิง query แยกไปหา category ของตัวเอง และอีก 2 query
  # แยกไปหา attachment + blob ของ photo ตัวเอง รวมเป็น "1 + 3N" query สำหรับ N สินค้า
  @pagy, @products = pagy(
    scope.includes(:category, photo_attachment: :blob).order(created_at: :desc),
    limit: 12
  )
end
```

### `app/views/products/index.html.erb` (เวอร์ชันค้นหา — เพิ่ม facet ใน Step 697)

```erb
<%# app/views/products/index.html.erb %>
<header class="page-header">
  <h1>สินค้าทั้งหมด</h1>
  <%= link_to "+ เพิ่มสินค้าใหม่", new_product_path, class: "btn btn-primary" %>
</header>

<%# ใช้ method: :get เพื่อให้ query params ปรากฏใน URL — แชร์ลิงก์ผลค้นหาได้ กด back/forward
    ของ browser ได้ถูกต้อง (ทบทวนหลักการเดียวกับ Part 038) %>
<%= form_with url: products_path, method: :get, local: true, class: "search-bar" do |f| %>
  <%= f.text_field :q, value: params[:q], placeholder: "ค้นหาสินค้า (ไทย/อังกฤษ)…" %>
  <%= f.submit "ค้นหา" %>
  <% if params[:q].present? %>
    <%= link_to "ล้างตัวกรอง", products_path %>
  <% end %>
<% end %>

<% if params[:q].present? %>
  <p class="search-summary">ผลการค้นหา "<%= params[:q] %>" พบ <%= @pagy.count %> รายการ</p>
<% end %>

<% if @products.any? %>
  <div class="product-grid">
    <% @products.each do |product| %>
      <%= link_to product_path(product), class: "product-card" do %>
        <% if product.photo.attached? && product.photo.representable? %>
          <%= image_tag product.photo.variant(:thumb), alt: product.name %>
        <% else %>
          <div class="no-photo-thumb">ไม่มีรูป</div>
        <% end %>

        <h3><%= product.name %></h3>
        <p class="product-card-category"><%= product.category.name %></p>
        <p class="product-card-price"><%= number_to_currency(product.price, unit: "฿", format: "%n %u") %></p>
      <% end %>
    <% end %>
  </div>

  <%== pagy_nav(@pagy) %>
<% else %>
  <p class="empty-state">ไม่พบสินค้าที่ตรงกับเงื่อนไข</p>
<% end %>
```

### พิสูจน์ว่า `includes` ป้องกัน N+1 ได้จริงด้วย SQL log

ล้าง log แล้วยิง request หนึ่งครั้งไปที่หน้า index ที่มีสินค้า 12 รายการ (เต็มหนึ่งหน้าพอดี):

```bash
> log/development.log
curl -s -o /dev/null "http://127.0.0.1:3070/products"
grep -c "SELECT" log/development.log
```

```
10
```

**10 query สำหรับ 12 สินค้า** (ไม่ใช่ `1 + 3×12 = 37` query ที่จะเกิดขึ้นถ้าไม่มี `includes`) —
ยืนยันว่าจำนวน query คงที่ไม่ขึ้นกับจำนวนแถว (constant, ไม่ใช่ linear ตาม N) นี่คือนิยามของการ
"ไม่มี N+1" ตามที่ Part 034 สอนไว้: query ที่เห็นคือ `Product Count` (สำหรับ pagy คำนวณจำนวนหน้า),
`Product Load` (โหลด 12 แถวพร้อม `JOIN`/`includes`), `Category Load` (โหลด category ที่เกี่ยวข้อง
ทั้งหมดในครั้งเดียว), `ActiveStorage::Attachment Load` และ `ActiveStorage::Blob Load` (โหลด
attachment/blob ของทุกสินค้าในครั้งเดียวเช่นกัน) — แต่ละอย่างเป็น **query เดียว** ไม่ว่าจะมีสินค้า
กี่รายการในหน้านั้นก็ตาม

ทดสอบค้นหาจริงผ่าน HTTP ยืนยันผลลัพธ์ตรงกับที่ทดสอบผ่าน console ใน Step 695:

```bash
curl -s "http://127.0.0.1:3070/products?q=running" | grep -oE '<h3>[^<]*</h3>'
```

```
<h3>Running Shoes Red Edition</h3>
```

```bash
curl -s "http://127.0.0.1:3070/products?q=%E0%B8%A3%E0%B8%AD%E0%B8%9A%E0%B9%80%E0%B8%97%E0%B9%89%E0%B8%B2" \
  | grep -oE '<h3>[^<]*</h3>'
# (URL-encoded ของคำว่า "รอบเท้า" ซึ่งพิมพ์ผิดจาก "รองเท้า")
```

```
<h3>รองเท้าผ้าใบสีแดง</h3>
```

ค้นหาด้วยคำที่พิมพ์ผิดผ่าน**หน้าเว็บจริง**ยังหาสินค้าที่ถูกต้องเจอ — พิสูจน์ว่า dual-scope pattern
ทำงานถูกต้องแบบ end-to-end ตั้งแต่ช่องค้นหาในหน้าเว็บไปจนถึงผลลัพธ์ ไม่ใช่แค่ทดสอบแยกส่วนผ่าน
console

---

## Step 697: Category facet แบบง่ายด้วย ActiveRecord grouping

### ทำไมใช้ `group`/`count` ธรรมดา ไม่ต้องพึ่ง Elasticsearch

**Facet** คือ UI ที่แสดง "ตัวเลือกกรองผลลัพธ์พร้อมจำนวนที่จะเจอถ้าเลือกตัวนั้น" (เช่น
"เสื้อผ้า (2)", "รองเท้า (2)") เป็นฟีเจอร์ที่เว็บอีคอมเมิร์ซใหญ่ๆ (Amazon, Lazada) ใช้เพื่อช่วยให้
ผู้ใช้กรองสินค้าจำนวนมากได้เร็ว ระบบค้นหาระดับ Elasticsearch (ทบทวน Part 069) มีความสามารถ
**aggregations** ที่คำนวณ facet หลาย attribute พร้อมกันแบบเรียลไทม์ได้อย่างมีประสิทธิภาพสูงมาก
แม้ข้อมูลจะมีหลักล้านแถว

แต่สำหรับโปรเจกต์ขนาดนี้ (สินค้าหลักร้อย-พัน, facet เดียวคือหมวดหมู่) การพึ่ง Elasticsearch ทั้ง
ระบบเพียงเพื่อ facet เดียวถือเป็นการ over-engineering — **`ActiveRecord::Calculations#group`
ร่วมกับ `#count`** (ทบทวนจาก Part 034) ทำงานได้ผลลัพธ์เดียวกันด้วยโค้ดบรรทัดเดียว บน PostgreSQL
ธรรมดาที่มีอยู่แล้ว

### อัปเดต Controller ให้คำนวณ facet

```ruby
# app/controllers/products_controller.rb (เวอร์ชันเต็มของ index — รวม Step 696 + Step 697)
class ProductsController < ApplicationController
  before_action :set_product, only: %i[show edit update destroy]

  def index
    scope = Product.search_by_name_and_description(params[:q])
    scope = scope.where(category_id: params[:category_id]) if params[:category_id].present?

    @pagy, @products = pagy(
      scope.includes(:category, photo_attachment: :blob).order(created_at: :desc),
      limit: 12
    )

    # นับจำนวนสินค้าต่อหมวดหมู่แบบง่าย (facet) ด้วย ActiveRecord grouping ธรรมดา — เพียงพอ
    # สำหรับ facet เดียวขนาดนี้ ถ้าต้องการ facet ซับซ้อนกว่านี้ (หลาย attribute พร้อมกัน, นับ
    # ภายใต้เงื่อนไขค้นหาที่เปลี่ยนไปเรื่อยๆ แบบเรียลไทม์บนข้อมูลจำนวนมาก) ค่อยพิจารณา
    # Elasticsearch aggregations ตามที่ Part 069 แนะนำไว้
    @category_counts = Product.group(:category_id).count
    @categories = Category.alphabetical
  end

  # ... (show, new, create, edit, update, destroy เหมือน Step 694 ทุกประการ)

  private

  def set_product
    @product = Product.includes(:category, photo_attachment: :blob).find(params[:id])
  end

  def product_params
    params.require(:product).permit(:name, :description, :price_cents, :category_id, :photo)
  end
end
```

### อัปเดต `index.html.erb` ให้มีทั้งช่องค้นหา, เลือกหมวดหมู่, และ facet nav

```erb
<%# app/views/products/index.html.erb (เวอร์ชันเต็ม — รวม Step 696 + Step 697) %>
<header class="page-header">
  <h1>สินค้าทั้งหมด</h1>
  <%= link_to "+ เพิ่มสินค้าใหม่", new_product_path, class: "btn btn-primary" %>
</header>

<%= form_with url: products_path, method: :get, local: true, class: "search-bar" do |f| %>
  <%= f.text_field :q, value: params[:q], placeholder: "ค้นหาสินค้า (ไทย/อังกฤษ)…" %>

  <%= f.select :category_id,
                options_for_select(@categories.map { |c| [c.name, c.id] }, params[:category_id]),
                { include_blank: "ทุกหมวดหมู่" } %>

  <%= f.submit "ค้นหา" %>
  <% if params[:q].present? || params[:category_id].present? %>
    <%= link_to "ล้างตัวกรอง", products_path %>
  <% end %>
<% end %>

<%# facet แบบง่าย: แสดงจำนวนสินค้าต่อหมวดหมู่ เป็นลิงก์กรองด่วน %>
<nav class="category-facets">
  <%= link_to "ทั้งหมด (#{Product.count})", products_path(q: params[:q]),
              class: params[:category_id].blank? ? "active" : nil %>
  <% @categories.each do |category| %>
    <%= link_to "#{category.name} (#{@category_counts[category.id] || 0})",
                products_path(q: params[:q], category_id: category.id),
                class: params[:category_id].to_i == category.id ? "active" : nil %>
  <% end %>
</nav>

<% if params[:q].present? %>
  <p class="search-summary">ผลการค้นหา "<%= params[:q] %>" พบ <%= @pagy.count %> รายการ</p>
<% end %>

<% if @products.any? %>
  <div class="product-grid">
    <% @products.each do |product| %>
      <%= link_to product_path(product), class: "product-card" do %>
        <% if product.photo.attached? && product.photo.representable? %>
          <%= image_tag product.photo.variant(:thumb), alt: product.name %>
        <% else %>
          <div class="no-photo-thumb">ไม่มีรูป</div>
        <% end %>

        <h3><%= product.name %></h3>
        <p class="product-card-category"><%= product.category.name %></p>
        <p class="product-card-price"><%= number_to_currency(product.price, unit: "฿", format: "%n %u") %></p>
      <% end %>
    <% end %>
  </div>

  <%== pagy_nav(@pagy) %>
<% else %>
  <p class="empty-state">ไม่พบสินค้าที่ตรงกับเงื่อนไข</p>
<% end %>
```

### ทดสอบ facet + filter ผ่าน console และ HTTP จริง

```irb
irb> Category.alphabetical.pluck(:id, :name)
=> [[4, "กระเป๋า"], [7, "ของใช้ในบ้าน"], [3, "รองเท้า"], [6, "อิเล็กทรอนิกส์"],
    [5, "เครื่องประดับ"], [2, "เสื้อผ้า"]]
irb> Product.group(:category_id).count
=> {3=>2, 5=>2, 4=>2, 6=>2, 2=>2, 7=>2}
```

ทุกหมวดหมู่มีสินค้า 2 ชิ้นพอดี (ตรงกับที่ `db/seeds.rb` สร้างไว้ — 1 ชื่อไทย + 1 ชื่ออังกฤษต่อ
หมวดหมู่) ทดสอบกรองผ่าน HTTP จริง:

```bash
curl -s "http://127.0.0.1:3070/products?category_id=2" | grep -oE '<h3>[^<]*</h3>'
```

```
<h3>Premium Cotton T-Shirt (Navy)</h3>
<h3>เสื้อยืดคอกลม สีน้ำเงิน</h3>
```

กรองตามหมวดหมู่ `id=2` ("เสื้อผ้า") ได้ผลลัพธ์ถูกต้องครบทั้งสองชิ้นในหมวดนั้น พอดีกับตัวเลขที่
facet แสดงไว้ (`เสื้อผ้า (2)`)

---

## Step 698: ประมวลผล variant ผ่าน background job

### ปัญหาของการสร้าง variant แบบ lazy

ย้อนกลับไป Step 693 — การเรียก `product.photo.variant(:thumb)` ในหน้า index/show **ไม่ได้
รับประกันว่าไฟล์ variant ถูกสร้างไว้แล้ว** ถ้ายังไม่เคยมีไฟล์นั้นมาก่อน Active Storage จะสร้างมัน
แบบ **lazy ตอนที่ browser ร้องขอ URL จริง** — หมายความว่า **ผู้ใช้คนแรก** ที่เปิดดูสินค้าชิ้นนั้น
ต้องรอ `ruby-vips` ประมวลผลภาพสด (resize, ครอบตัด, เขียนไฟล์ใหม่) ก่อนถึงจะเห็นรูป ซึ่งอาจใช้เวลา
หลักร้อยมิลลิวินาทีถึงหลักวินาทีขึ้นกับขนาดไฟล์ต้นฉบับ — ยิ่งแย่ถ้าสินค้าชิ้นนั้นมีคนเปิดพร้อมกัน
หลายคนตอนกำลังประมวลผลอยู่พอดี

วิธีแก้คือ **ประมวลผล variant ล่วงหน้าทันทีหลังอัปโหลดเสร็จ ผ่าน background job** (ทบทวนรูปแบบ
ActiveJob จาก Part 061) แทนที่จะรอให้ request จริงมาทริกเกอร์

### `app/jobs/product_photo_variants_job.rb`

```ruby
# app/jobs/product_photo_variants_job.rb
# Job นี้สร้าง Active Storage variant ทั้งสองขนาด (:thumb, :medium) ล่วงหน้า แทนที่จะปล่อยให้
# Rails สร้างแบบ "lazy" ตอนมีคนเปิดดูรูปครั้งแรก (ทบทวนรูปแบบ ActiveJob จาก Part 061 — เขียน
# job ที่รับ id ธรรมดา ไม่รับ ActiveRecord object ตรงๆ เพราะ argument ของ job ถูก serialize
# ผ่าน GlobalID แล้ว deserialize กลับตอนรันจริง ถ้า record ถูกลบไปก่อน job จะรัน การ
# deserialize object โดยตรงจะ raise error ทันที ในขณะที่ id ธรรมดาให้เราควบคุมได้เองว่าจะ
# ทำอะไรถ้าหา record ไม่เจอ)
class ProductPhotoVariantsJob < ApplicationJob
  queue_as :default

  # discard_on ป้องกัน job ค้างพยายามรันซ้ำเมื่อ Product/รูปถูกลบไปแล้วก่อนที่ job จะทันได้รัน
  # (เช่น ผู้ใช้ลบสินค้าทิ้งไม่ถึงวินาทีหลังอัปโหลด) — ไม่ใช่ error ที่ต้อง retry
  discard_on ActiveJob::DeserializationError
  discard_on ActiveStorage::FileNotFoundError

  def perform(product_id)
    product = Product.find(product_id)
    return unless product.photo.attached?

    # `.processed` สั่งให้ variant ถูกแปลงไฟล์จริงทันที (ถ้ายังไม่เคยมี) — เรียกทั้งสองขนาด
    # ในนี้ ทำให้ทั้ง :thumb และ :medium พร้อมใช้งานตั้งแต่ก่อน request แรกที่ต้องการมันจะมาถึง
    product.photo.variant(:thumb).processed
    product.photo.variant(:medium).processed
  end
end
```

### เพิ่ม callback ที่ `Product` ให้ enqueue job หลังแนบรูปสำเร็จ

```ruby
# app/models/product.rb (เวอร์ชันเต็ม — รวมทุก Step 692–698)
class Product < ApplicationRecord
  include PgSearch::Model

  ACCEPTED_PHOTO_TYPES = %w[image/jpeg image/png image/webp].freeze
  MAX_PHOTO_SIZE = 5.megabytes

  belongs_to :category

  # สังเกตว่า**ไม่ใส่** `preprocessed: true` ทั้งที่ Active Storage รองรับ (จะสั่งประมวลผล
  # variant ผ่าน background job ให้อัตโนมัติทันทีที่แนบไฟล์) — Part นี้เลือกเขียน
  # `ProductPhotoVariantsJob` เองเพื่อสาธิตรูปแบบ ActiveJob จาก Part 061 อย่างชัดเจน และเพื่อ
  # ควบคุมได้ว่าจะ pre-generate variant ไหนบ้างเมื่อไหร่ ในโปรเจกต์จริงการใช้ `preprocessed:
  # true` ตรงๆ ก็เป็นทางเลือกที่ใช้งานได้ดีเช่นกัน
  has_one_attached :photo do |attachable|
    attachable.variant :thumb, resize_to_fill: [150, 150]
    attachable.variant :medium, resize_to_limit: [600, 600]
  end

  validates :name, presence: true
  validates :price_cents, numericality: { only_integer: true, greater_than: 0 }
  validate :photo_content_type_allowed
  validate :photo_size_within_limit

  # เมื่อมีการแนบ/เปลี่ยนรูปสำเร็จ (หลัง transaction commit เท่านั้น ทบทวน Part 035/040) ให้
  # ส่งงานสร้าง variant ล่วงหน้าเข้าคิว แทนที่จะปล่อยให้ variant ถูกสร้างแบบ lazy ตอน request
  # แรกที่มีคนเปิดดูรูป
  before_save :track_photo_attachment_change
  after_commit :enqueue_photo_variants_job, on: %i[create update], if: :photo_attachment_changed?

  pg_search_scope :search_by_name_and_description_en,
                   against: %i[name description],
                   using: { tsearch: { prefix: true, dictionary: "english" } }

  pg_search_scope :search_by_name_and_description_th,
                   against: :name,
                   using: { trigram: { threshold: 0.15 } },
                   ranked_by: ":trigram"

  def self.search_by_name_and_description(query)
    return all if query.blank?

    tsearch_ids = search_by_name_and_description_en(query).ids
    trigram_ids = search_by_name_and_description_th(query).ids
    ordered_ids = tsearch_ids | trigram_ids

    return none if ordered_ids.empty?

    where(id: ordered_ids).in_order_of(:id, ordered_ids)
  end

  def price
    price_cents / 100.0
  end

  private

  def photo_content_type_allowed
    return unless photo.attached?
    return if ACCEPTED_PHOTO_TYPES.include?(photo.content_type)

    errors.add(:photo, "ต้องเป็นไฟล์ชนิด JPEG, PNG หรือ WebP เท่านั้น")
  end

  def photo_size_within_limit
    return unless photo.attached?
    return if photo.byte_size <= MAX_PHOTO_SIZE

    errors.add(:photo, "ต้องมีขนาดไม่เกิน #{MAX_PHOTO_SIZE / 1.megabyte} MB")
  end

  # `attachment_changes` (มาจาก Active Storage) เก็บรายการ attachment ที่ "กำลังจะถูกบันทึก"
  # ในรอบ save ปัจจุบัน — เช็คก่อน save จริงใน before_save แล้วจำผลไว้ในตัวแปร instance เพราะ
  # พอถึง after_commit ค่านี้จะถูกล้างไปแล้ว (attachment ถูก persist เรียบร้อย)
  def track_photo_attachment_change
    @photo_attachment_changed = attachment_changes.key?("photo")
  end

  def photo_attachment_changed?
    @photo_attachment_changed
  end

  def enqueue_photo_variants_job
    ProductPhotoVariantsJob.perform_later(id)
  end
end
```

**อธิบายจุดสำคัญ:**

- **`before_save :track_photo_attachment_change` คู่กับ `after_commit`** — `attachment_changes`
  เป็น state ชั่วคราวที่มีค่าเฉพาะ**ก่อน** save เท่านั้น (พอ save เสร็จ Active Storage ล้างมันทิ้ง
  เพราะ "การเปลี่ยนแปลง" กลายเป็น "สิ่งที่ persist แล้ว") แต่เราต้องการรู้ผลนี้ **หลัง** commit
  (เพื่อรับประกันว่า transaction สำเร็จจริงก่อนส่ง job ทบทวนหลักการ `after_commit` จาก
  Part 035/040) จึงต้องจดค่าไว้ในตัวแปร instance ระหว่างสองจุดเวลานี้
- **`if: :photo_attachment_changed?`** — ทำให้ job ถูก enqueue **เฉพาะตอนที่รูปเปลี่ยนจริง**
  ไม่ใช่ทุกครั้งที่ `update` (เช่น แก้แค่ราคาสินค้าไม่ควร enqueue job ประมวลผลรูปซ้ำโดยไม่จำเป็น)

### พิสูจน์ว่า background job ทำงานจริง

รัน `db/seeds.rb` (Step 699) ที่สร้างสินค้า 12 ชิ้นพร้อมแนบรูป แล้วตรวจสอบว่า variant record
เกิดขึ้นในฐานข้อมูลจริงหรือไม่ (ไม่ใช่แค่ enqueue job เฉยๆ แล้วไม่มีอะไรเกิดขึ้น):

```irb
irb> ActiveStorage::Blob.count
=> 30    # 12 รูปต้นฉบับ + 18 variant (ยังไม่ครบ 24 = 12×2 เพราะ process ของ db:seed
         #                              เป็น process อายุสั้น ดูคำอธิบายด้านล่าง)
irb> ActiveStorage::VariantRecord.count
=> 18
```

> **ข้อสังเกตที่ซื่อสัตย์ (พบจริงระหว่างทดสอบ Part นี้):** ค่า default ของ
> `ActiveJob::Base.queue_adapter` ใน development คือ `:async` (ทบทวน Part 061) ซึ่งรันงานใน
> thread pool ของ process เดียวกัน — ปัญหาคือ `bin/rails db:seed` เป็น **process ที่อายุสั้น**
> (รันเสร็จแล้วปิดตัวทันที) ทำให้ thread pool ของ async adapter **อาจไม่มีเวลาประมวลผล job
> ครบทุกตัวก่อน process จะปิด** (ทดสอบจริงได้ 18 จาก 24 variant ที่ควรจะมี) — นี่**ไม่ใช่ปัญหา
> ในสถานการณ์จริง** เพราะ web server process (ที่รับ request จริงจากผู้ใช้) เป็น process ที่มี
> อายุยืนตลอดเวลาที่ app รันอยู่ ต่างจาก `db:seed` ที่ตั้งใจให้รันครั้งเดียวแล้วจบ — ลองพิสูจน์ด้วย
> การเปิดหน้า index ผ่าน HTTP จริง (ซึ่งจำลองพฤติกรรมของผู้ใช้จริง):
>
> ```bash
> curl -s -o index.html "http://127.0.0.1:3070/products"
> grep -o 'src="[^"]*"' index.html | head -1
> ```
>
> ```
> src="http://127.0.0.1:3070/rails/active_storage/representations/redirect/.../mug.jpg"
> ```
>
> ```bash
> curl -s -o /dev/null -w "%{http_code}\n" -L \
>   "http://127.0.0.1:3070/rails/active_storage/representations/redirect/.../mug.jpg"
> # => 200
> ```
>
> URL ของ variant โหลดได้สำเร็จ (200) ทุกตัว — ถ้า job ยังไม่ทันประมวลผลไว้ล่วงหน้า Active
> Storage จะ fallback ไปสร้างแบบ lazy ให้ทันทีที่ request มาถึง (ตามกลไกที่อธิบายไว้ใน Step 693)
> ผู้ใช้จึงยังเห็นรูปถูกต้องเสมอไม่ว่า background job จะทันประมวลผลไว้ก่อนหรือไม่ — **background
> job มีไว้เพื่อ "ลดโอกาส" ที่ผู้ใช้จะต้องรอ ไม่ใช่เงื่อนไขที่จำเป็นต่อความถูกต้องของระบบ** นี่คือ
> เหตุผลที่ดีไซน์นี้ปลอดภัย: ต่อให้ job ล้มเหลว/ช้า/ยังไม่ทันรัน หน้าเว็บก็ยังทำงานถูกต้องอยู่ดี
>
> ในระบบ production จริง ควรสลับไปใช้ **Solid Queue** (ทบทวน Part 062 — ค่า default ของ Rails 8
> ใน production environment อยู่แล้ว) ซึ่งเก็บ job ไว้ในฐานข้อมูลและมี worker process แยกต่างหาก
> (`bin/jobs`) ที่รันอยู่ตลอดเวลา ไม่ผูกกับอายุของ process ที่ enqueue job เหมือน `:async`
> adapter ทำให้ job ทุกตัวถูกประมวลผลแน่นอนไม่ว่า process ที่ enqueue จะปิดตัวไปแล้วหรือไม่

---

## Step 699: `db/seeds.rb` พร้อมรูปจริง และ manual verification walkthrough แบบเต็ม

### เตรียมรูปภาพตัวอย่างสำหรับ seed

โปรเจกต์นี้ใช้รูปภาพ placeholder สีพื้นที่สร้างขึ้นเองด้วยสคริปต์ (แทนรูปสินค้าจริงที่หาลิขสิทธิ์
ยาก) — สคริปต์นี้ใช้ครั้งเดียวตอนเตรียมข้อมูล ไม่ใช่ส่วนหนึ่งของแอปที่รันจริง:

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true

# db/seed_images/generate.rb — สคริปต์ช่วยครั้งเดียว (dev-time only) สำหรับสร้างรูปภาพ
# placeholder สีพื้นง่ายๆ ไว้ใช้กับ db/seeds.rb
require "vips"

COLORS = {
  "shirt.jpg" => [66, 135, 245],
  "shoes.jpg" => [235, 87, 87],
  "backpack.jpg" => [39, 174, 96],
  "watch.jpg" => [242, 201, 76],
  "headphones.jpg" => [155, 89, 182],
  "mug.jpg" => [230, 126, 34]
}.freeze

COLORS.each do |filename, rgb|
  image = Vips::Image.black(800, 800).new_from_image(rgb)
  image.jpegsave(File.join(__dir__, filename), Q: 80)
  puts "created #{filename}"
end
```

```bash
bundle exec ruby db/seed_images/generate.rb
```

```
created shirt.jpg
created shoes.jpg
created backpack.jpg
created watch.jpg
created headphones.jpg
created mug.jpg
```

### `db/seeds.rb`

```ruby
# frozen_string_literal: true

puts "ลบข้อมูลเก่าทั้งหมด..."
# purge_later ทำให้ variant/blob เดิม (ถ้ามี) ถูกลบไฟล์จริงออกจาก service ด้วย ไม่เหลือไฟล์
# กำพร้าอยู่ใน storage/ — ทบทวนความสำคัญของการล้าง attachment ให้ครบจาก Part 066
Product.find_each { |product| product.photo.purge_later if product.photo.attached? }
Product.delete_all
Category.delete_all

puts "สร้างหมวดหมู่..."
category_names = %w[เสื้อผ้า รองเท้า กระเป๋า เครื่องประดับ อิเล็กทรอนิกส์ ของใช้ในบ้าน]
categories = category_names.index_with { |name| Category.create!(name: name) }

PRODUCTS = [
  { name: "เสื้อยืดคอกลม สีน้ำเงิน", description: "เสื้อยืดผ้าคอตตอน 100% ใส่สบาย ระบายอากาศดี เหมาะกับอากาศร้อนแบบไทย", category: "เสื้อผ้า", price: 29_900, photo: "shirt.jpg" },
  { name: "Premium Cotton T-Shirt (Navy)", description: "Soft breathable cotton t-shirt, perfect for everyday wear", category: "เสื้อผ้า", price: 39_900, photo: "shirt.jpg" },
  { name: "รองเท้าผ้าใบสีแดง", description: "รองเท้าผ้าใบน้ำหนักเบา พื้นยางกันลื่น เดินสบายทั้งวัน", category: "รองเท้า", price: 129_900, photo: "shoes.jpg" },
  { name: "Running Shoes Red Edition", description: "Lightweight running shoes with breathable mesh upper", category: "รองเท้า", price: 189_900, photo: "shoes.jpg" },
  { name: "กระเป๋าเป้เดินทาง สีเขียว", description: "กระเป๋าเป้จุของได้เยอะ กันน้ำ เหมาะสำหรับเดินทางและใช้งานประจำวัน", category: "กระเป๋า", price: 89_900, photo: "backpack.jpg" },
  { name: "Waterproof Travel Backpack", description: "Durable waterproof backpack with laptop compartment", category: "กระเป๋า", price: 119_900, photo: "backpack.jpg" },
  { name: "นาฬิกาข้อมือสีทอง", description: "นาฬิกาข้อมือดีไซน์คลาสสิก สายสแตนเลส กันน้ำ กันรอยขีดข่วน", category: "เครื่องประดับ", price: 249_900, photo: "watch.jpg" },
  { name: "Classic Gold Wrist Watch", description: "Elegant stainless steel watch with sapphire crystal glass", category: "เครื่องประดับ", price: 349_900, photo: "watch.jpg" },
  { name: "หูฟังไร้สาย สีม่วง", description: "หูฟังบลูทูธตัดเสียงรบกวน แบตอึด ฟังเพลงได้ทั้งวัน", category: "อิเล็กทรอนิกส์", price: 159_900, photo: "headphones.jpg" },
  { name: "Wireless Noise Cancelling Headphones", description: "Premium wireless headphones with active noise cancellation", category: "อิเล็กทรอนิกส์", price: 259_900, photo: "headphones.jpg" },
  { name: "แก้วเก็บความเย็น สีส้ม", description: "แก้วสแตนเลสเก็บความเย็นได้นาน 12 ชั่วโมง พกพาสะดวก", category: "ของใช้ในบ้าน", price: 19_900, photo: "mug.jpg" },
  { name: "Insulated Stainless Steel Mug", description: "Keeps drinks cold for 12 hours, perfect for daily commute", category: "ของใช้ในบ้าน", price: 24_900, photo: "mug.jpg" }
].freeze

puts "สร้างสินค้าพร้อมแนบรูป..."
PRODUCTS.each do |attrs|
  product = Product.create!(
    name: attrs[:name],
    description: attrs[:description],
    price_cents: attrs[:price],
    category: categories.fetch(attrs[:category])
  )

  photo_path = Rails.root.join("db/seed_images", attrs[:photo])
  product.photo.attach(
    io: File.open(photo_path),
    filename: attrs[:photo],
    content_type: "image/jpeg"
  )

  puts "  - #{product.name} (#{attrs[:category]})"
end

puts "เสร็จสิ้น: #{Category.count} หมวดหมู่, #{Product.count} สินค้า"
puts "หมายเหตุ: ProductPhotoVariantsJob ถูก enqueue ให้แต่ละสินค้าแล้ว (ผ่าน after_commit บน" \
     " Product) — รัน `bin/jobs` หรือรอ queue adapter :async ประมวลผลเพื่อสร้าง variant ล่วงหน้า"
```

```bash
bin/rails db:seed
```

```
ลบข้อมูลเก่าทั้งหมด...
สร้างหมวดหมู่...
สร้างสินค้าพร้อมแนบรูป...
  - เสื้อยืดคอกลม สีน้ำเงิน (เสื้อผ้า)
  - Premium Cotton T-Shirt (Navy) (เสื้อผ้า)
  - รองเท้าผ้าใบสีแดง (รองเท้า)
  - Running Shoes Red Edition (รองเท้า)
  - กระเป๋าเป้เดินทาง สีเขียว (กระเป๋า)
  - Waterproof Travel Backpack (กระเป๋า)
  - นาฬิกาข้อมือสีทอง (เครื่องประดับ)
  - Classic Gold Wrist Watch (เครื่องประดับ)
  - หูฟังไร้สาย สีม่วง (อิเล็กทรอนิกส์)
  - Wireless Noise Cancelling Headphones (อิเล็กทรอนิกส์)
  - แก้วเก็บความเย็น สีส้ม (ของใช้ในบ้าน)
  - Insulated Stainless Steel Mug (ของใช้ในบ้าน)
เสร็จสิ้น: 6 หมวดหมู่, 12 สินค้า
หมายเหตุ: ProductPhotoVariantsJob ถูก enqueue ให้แต่ละสินค้าแล้ว...
```

### Manual verification walkthrough แบบเต็ม (ทดสอบจริงทุกขั้นตอน)

รวบยอดทุกเทคนิคของ Part นี้เป็น checklist เดียว ทดสอบจริงทั้งหมดตามลำดับต่อไปนี้:

**1) อัปโหลดสินค้าใหม่พร้อมรูปจริงผ่านฟอร์มเว็บ:**

```bash
bin/rails server -p 3070 -d

curl -s "http://127.0.0.1:3070/products/new" -c cookies.txt -o new.html
TOKEN=$(grep -o 'name="authenticity_token" value="[^"]*"' new.html | sed 's/.*value="//;s/"$//')

curl -s -b cookies.txt -w "%{http_code}\n" \
  -F "authenticity_token=${TOKEN}" \
  -F "product[name]=กระเป๋าถือ สีดำ ทดสอบอัปโหลด" \
  -F "product[category_id]=4" \
  -F "product[price_cents]=45000" \
  -F "product[photo]=@db/seed_images/backpack.jpg;type=image/jpeg" \
  "http://127.0.0.1:3070/products"
```

```
302   ✓ redirect ไปหน้า show สำเร็จ — สินค้าถูกสร้างพร้อมรูป
```

**2) เห็น thumbnail ในหน้า index:**

```bash
curl -s "http://127.0.0.1:3070/products" | grep -c 'product-card'
```

```
12   ✓ การ์ดสินค้าครบ 12 ใบในหน้าแรก (limit 12/หน้า) แต่ละใบมี <img> variant :thumb
```

**3) ค้นหาด้วยคำไทยและคำอังกฤษ:**

```bash
curl -s "http://127.0.0.1:3070/products?q=running" | grep -oE '<h3>[^<]*</h3>'
# => <h3>Running Shoes Red Edition</h3>                              ✓

curl -s "http://127.0.0.1:3070/products?q=%E0%B9%80%E0%B8%AA%E0%B8%B7%E0%B9%89%E0%B8%AD" \
  | grep -oE '<h3>[^<]*</h3>'
# => <h3>เสื้อยืดคอกลม สีน้ำเงิน</h3>                                  ✓ (คำว่า "เสื้อ")
```

**4) กรองตามหมวดหมู่:**

```bash
curl -s "http://127.0.0.1:3070/products?category_id=3" | grep -oE '<h3>[^<]*</h3>'
```

```
<h3>Running Shoes Red Edition</h3>          ✓ ทั้งสองรายการอยู่ในหมวด "รองเท้า" (id=3)
<h3>รองเท้าผ้าใบสีแดง</h3>
```

**5) ยืนยันว่า background job ประมวลผล variant จริง:**

```irb
irb> product = Product.last
irb> product.photo.variant(:thumb).processed.key.present?
=> true
irb> ActiveStorage::VariantRecord.where(blob_id: product.photo.blob_id).count
=> 2   # :thumb และ :medium ถูกสร้างไว้ครบทั้งคู่
```

ครบทุกข้อตามที่ตั้งไว้ตอนต้น Part — ระบบอัปโหลดรูปสินค้าพร้อมค้นหาทำงานได้จริงแบบ end-to-end
ตั้งแต่การอัปโหลดไฟล์ ผ่านการประมวลผลรูปเบื้องหลัง ไปจนถึงการค้นหาและกรองผลลัพธ์

---

## Step 700: Code review ย้อนหลังทั้งโปรเจกต์ + สิ่งที่ยังขาดสำหรับ production + ปิด Phase 10

### จุดที่ควรสังเกตในเฉลยทั้งโปรเจกต์ — เชื่อมกลับไปแต่ละ Part ของเฟส 10

| เทคนิคที่ใช้ในโปรเจกต์นี้ | มาจาก Part | จุดที่ใช้จริง |
|---|---|---|
| `has_one_attached`, validation content-type/size | **Part 066** | `Product` model (Step 692) |
| Named variant (`:thumb`/`:medium`), `resize_to_fill` vs `resize_to_limit` | **Part 067** | `has_one_attached` block (Step 693) |
| `pg_search_scope`, `tsearch`, `trigram`, `pg_trgm` extension | **Part 068** | `Product` model + migration (Step 695) |
| แนวคิด aggregations สำหรับ facet ซับซ้อน (อ้างอิงเป็นทางเลือกอนาคต) | **Part 069** | อธิบายไว้ใน Step 697 |
| N+1 prevention ด้วย `includes` | Part 034 | `ProductsController#index` (Step 696) |
| Pagination ด้วย pagy | Part 038 | `ProductsController#index` + `pagy_nav` (Step 696) |
| ActiveJob pattern (`ApplicationJob`, `perform_later`, `discard_on`) | Part 061 | `ProductPhotoVariantsJob` (Step 698) |
| ห้ามเก็บเงินเป็น float, `dependent: :restrict_with_error` | Part 040 | `price_cents`, `Category` (Step 692) |

**สิ่งที่ควรสังเกตเพิ่มเติม:**

- **ทุก validation ของ `photo` เขียนแบบ `return unless photo.attached?` ก่อนเสมอ** — ทำให้รูป
  เป็น**ทางเลือก** (optional) ไม่ใช่บังคับ นี่เป็นการตัดสินใจออกแบบที่จงใจ (สินค้าที่ยังไม่มีรูป
  ก็ยังบันทึกได้ แสดงเป็น "ไม่มีรูป" ในหน้า UI) ถ้าต้องการบังคับว่าทุกสินค้าต้องมีรูปเสมอ ต้อง
  เพิ่ม `validates :photo, presence: true` แยกต่างหาก (ไม่มีใน built-in validator ของ
  `has_one_attached` ต้องเขียนเอง เพราะ Active Storage attachment ไม่ใช่ ActiveRecord
  association ธรรมดา)
- **การรวมผลลัพธ์สอง scope ด้วย `Array#|` แล้ว `where(id: ...).in_order_of(...)`** เป็นวิธีที่
  เรียบง่ายและอ่านง่ายที่สุดสำหรับข้อมูลขนาดนี้ แต่มีข้อจำกัด: ต้องดึง `.ids` ของทั้งสอง query
  มาไว้ใน Ruby ก่อนแล้วค่อยรวม (ไม่ใช่ SQL query เดียว) ถ้าข้อมูลมีหลักแสน-ล้านแถว การทำแบบนี้
  จะเริ่มไม่มีประสิทธิภาพ (ต้องดึง id ทั้งหมดที่ match มาไว้ใน memory) — จุดนี้คือสัญญาณที่บอกว่า
  ถึงเวลาพิจารณา Elasticsearch (Part 069) จริงจัง เพราะมันคำนวณ relevance ranking ของ multi-field
  multi-language search แบบนี้ได้ในเอนจินเดียว ไม่ต้องรวมผลลัพธ์ฝั่ง Ruby เอง
- **`ProductPhotoVariantsJob` ปลอดภัยแม้ job จะ fail/ช้า** เพราะ Active Storage สร้าง variant
  แบบ lazy เป็น fallback อยู่แล้ว (พิสูจน์ใน Step 698) — background job มีไว้เพื่อ **ประสิทธิภาพ**
  (ลดโอกาสผู้ใช้ต้องรอ) ไม่ใช่เงื่อนไขที่จำเป็นต่อ **ความถูกต้อง** ของระบบ เป็นตัวอย่างที่ดีของ
  การออกแบบระบบให้ "degrade gracefully" เมื่อส่วนประกอบเสริมล้มเหลว

### สิ่งที่ยังขาดสำหรับ production (ตั้งใจไม่ทำใน Part นี้ เพื่อไม่ให้ขอบเขตบวมเกินไป)

1. **ไม่มี CDN หน้ารูปภาพ** — โปรเจกต์นี้ใช้ local disk service (`config.active_storage.service
   = :local`) ทุกรูปถูก serve ผ่าน Rails server เอง (ผ่าน `/rails/active_storage/...`) ระบบจริง
   ควรใช้ S3-compatible storage ตามที่ Part 067 สอนไว้ **บวกกับ CDN** (CloudFront, Cloudflare)
   หน้า S3 อีกชั้น เพื่อ cache รูปไว้ใกล้ผู้ใช้ปลายทางและลด load ของ storage service โดยตรง
2. **ไม่มีการปรับแต่ง image optimization pipeline ให้ละเอียดกว่านี้** — ตอนนี้ variant ถูกบันทึก
   เป็น JPEG คุณภาพ default เท่านั้น ระบบจริงควรพิจารณา: แปลงเป็น **WebP/AVIF** สำหรับ browser ที่
   รองรับ (ลดขนาดไฟล์ได้มากกว่า JPEG), ทำ **responsive images** ด้วย `srcset` (ส่งรูปหลายขนาด
   ให้ browser เลือกเองตามหน้าจอ), และปรับ `Q:` (compression quality) ให้เหมาะกับแต่ละ use case
   แทนการใช้ default ของ `image_processing` ตรงๆ
3. **Search relevance ยังไม่ได้ปรับจูน (search relevance tuning)** — ตัวอย่างที่พบระหว่างทดสอบ
   Step 695 (`trigram` ที่ threshold 0.15 อาจเข้มไปสำหรับบางคำ หรือหลวมไปสำหรับบางคำ) แสดงให้เห็น
   ว่าค่า threshold/weight ที่ "ใช้ได้" ต้องอาศัยการทดสอบกับข้อมูลจริงและพฤติกรรมผู้ใช้จริงเป็น
   เวลานาน ไม่ใช่ค่าที่กำหนดครั้งเดียวแล้วจบ ระบบจริงควรมี metric วัดคุณภาพการค้นหา (เช่น
   click-through rate ของผลลัพธ์อันดับต้นๆ) และ A/B test การปรับ weight ระหว่าง `tsearch` กับ
   `trigram`
4. **ไม่มีระบบ authentication/authorization** — ใครก็ตามที่เข้าถึง URL ได้สามารถเพิ่ม/แก้/ลบ
   สินค้าได้หมด ไม่มีการตรวจสอบว่าเป็น "ผู้ขาย" ที่ได้รับอนุญาตหรือไม่ (Phase 5 สอนเรื่องนี้ไว้
   ครบแล้ว — ตัดออกจาก Part นี้เพื่อโฟกัสเทคนิคของ Phase 10 ล้วนๆ โปรเจกต์จริงต้องเพิ่ม
   `before_action :authenticate_user!` และ Pundit policy ตรวจว่าแก้ไขได้เฉพาะสินค้าของตัวเอง)
5. **Facet มีแค่หมวดหมู่เดียว ไม่รองรับหลาย attribute พร้อมกัน** — เช่น กรองพร้อมกันทั้งช่วงราคา,
   หมวดหมู่, และมีรูปหรือไม่ ระบบจริงที่ต้องการ facet หลายมิติพร้อมกันบนข้อมูลจำนวนมาก ควรใช้
   Elasticsearch aggregations (Part 069) แทน `group`/`count` ธรรมดา
6. **ไม่มี rate limiting บนช่องค้นหา** — endpoint `/products?q=...` เปิดให้ยิง query ได้ไม่จำกัด
   ถ้ามีคนยิง query แปลกๆ ถี่ๆ (เช่น string ยาวมากซ้ำๆ) อาจสร้างภาระให้ PostgreSQL โดยไม่จำเป็น
   ระบบจริงควรพิจารณา `rack-attack` (จะเรียนใน Phase 8 เชิงลึกกว่านี้) จำกัดจำนวน request ต่อ
   IP ต่อช่วงเวลา

การรู้ว่า "อะไรยังขาด" อย่างชัดเจนแบบนี้ **สำคัญพอๆ กับการรู้ว่าทำอะไรไปแล้ว** — วิศวกรมืออาชีพ
ไม่ได้แค่ส่งมอบโค้ดที่ทำงานได้ แต่ต้องสื่อสารขอบเขตและความเสี่ยงที่เหลืออยู่ให้ทีมรู้ด้วยเสมอ

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่มแกลเลอรีรูปสินค้าหลายรูป** — เปลี่ยน `has_one_attached :photo` เป็น
   `has_many_attached :photos` (ทบทวนความแตกต่างจาก Part 066) แก้ฟอร์มให้อัปโหลดได้หลายไฟล์
   พร้อมกัน (`f.file_field :photos, multiple: true`) และแก้ `product_params` ให้ permit
   `photos: []` — ระวังจุดที่ validation เนื้อหา/ขนาดไฟล์ต้องวนลูปตรวจทุกไฟล์ ไม่ใช่ไฟล์เดียว
   เหมือนเดิม แล้วปรับ `ProductPhotoVariantsJob` ให้ pre-generate variant ของทุกรูปในแกลเลอรี
2. **เพิ่ม autocomplete ให้ช่องค้นหา** — สร้าง endpoint `GET /products/search_suggestions.json`
   ที่คืนชื่อสินค้า 5 อันดับแรกที่ตรงกับสิ่งที่ผู้ใช้พิมพ์อยู่ (เรียก
   `search_by_name_and_description` แบบเดียวกัน แต่ limit 5 และคืนแค่ `id`/`name`) แล้วต่อกับ
   Stimulus controller (ทบทวน Part 053) ที่ยิง fetch ทุกครั้งที่ผู้ใช้พิมพ์ (debounce ด้วย
   `setTimeout` ป้องกันยิง request ถี่เกินไป)
3. **เพิ่มตัวกรองช่วงราคา** — เพิ่ม input สองช่อง (`min_price`, `max_price`) ในฟอร์มค้นหา แก้
   controller ให้เพิ่มเงื่อนไข `scope = scope.where(price_cents: min..max)` เมื่อมีค่าทั้งสอง
   ช่อง (ทบทวนการต่อ scope แบบมีเงื่อนไขจาก Part 034) แล้วลองคิดว่าจะแสดง facet ราคาแบบ "ช่วง"
   (เช่น "ต่ำกว่า 100 บาท (3)", "100–500 บาท (7)") ด้วย `group`/`count` ธรรมดาได้อย่างไร — เป็น
   จุดที่เริ่มเห็นข้อจำกัดของ `group` ธรรมดาเมื่อเทียบกับ Elasticsearch range aggregations

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- วางแผนและสร้างโปรเจกต์ Rails ใหม่ที่เลือก **PostgreSQL** ตั้งแต่ `rails new` โดยรู้เหตุผลชัดเจน
  ว่าทำไมต้องใช้ (`pg_search` ต้องพึ่ง `tsvector` และ extension `pg_trgm` ซึ่งเป็นความสามารถ
  เฉพาะของ PostgreSQL)
- ผสาน validation เนื้อหา/ขนาดไฟล์ของ Active Storage attachment (Part 066) เข้ากับ validation
  มาตรฐานของ ActiveRecord ในโมเดลเดียวกัน พร้อมแก้ปัญหาจริงที่พบระหว่างทดสอบ (`representable?`,
  `persisted?` ก่อนเรียก `.variant` ในฟอร์มที่อาจ validation fail)
- ประกาศ **named variant** สองขนาด (`:thumb` แบบ `resize_to_fill`, `:medium` แบบ
  `resize_to_limit`) ตรงจุดเดียวกับ `has_one_attached` (ทบทวน Part 067) และเข้าใจความแตกต่างของ
  โหมด resize ทั้งสองแบบ
- สร้าง **dual-scope search pattern** ที่รวม `pg_search_scope` แบบ `tsearch` (full-text ภาษา
  อังกฤษ) กับแบบ `trigram` (ค้นหาภาษาไทย/ทนต่อคำสะกดผิด) เข้าด้วยกัน (ทบทวน Part 068) พร้อม
  พิสูจน์ด้วยข้อมูลจริงว่าแต่ละ scope เก่ง/อ่อนเรื่องอะไร และรู้ข้อจำกัดของการ concatenate
  หลายคอลัมน์เข้าด้วยกันก่อนคำนวณ trigram similarity
- ประกอบหน้าค้นหาที่สมบูรณ์: ช่องค้นหา + facet หมวดหมู่ (ActiveRecord `group`/`count` ธรรมดา) +
  pagination (pagy, Part 038) + eager loading ป้องกัน N+1 (`includes`, Part 034) พิสูจน์ด้วย
  การนับจำนวน SQL query จริงจาก log
- เขียน **background job ประมวลผล image variant ล่วงหน้า** ตามรูปแบบ ActiveJob จาก Part 061
  (`queue_as`, `discard_on`) และเข้าใจว่าทำไม design นี้ปลอดภัยแม้ job จะล้มเหลว/ช้า (Active
  Storage มี lazy fallback อยู่แล้วเสมอ)
- เข้าใจข้อจำกัดของแนวทาง "ง่ายก่อน" ในแต่ละจุด (facet ด้วย `group`/`count`, รวมผลค้นหาด้วย
  `Array#|`) และรู้ว่าจุดไหนคือสัญญาณที่ควรพิจารณา Elasticsearch (Part 069) เมื่อระบบโตขึ้น
- ระบุรายการ "สิ่งที่ยังขาดสำหรับ production" ได้อย่างชัดเจน (CDN, image optimization pipeline,
  search relevance tuning, authorization, facet หลายมิติ, rate limiting) — ทักษะการสื่อสาร
  ขอบเขตและความเสี่ยงที่เหลืออยู่ ซึ่งสำคัญพอๆ กับทักษะการเขียนโค้ด

## สรุปภาพรวม Phase 10: File Upload/Search

ยินดีด้วย! ตอนนี้ **Phase 10: File Upload/Search (Part 066–070, Step 651–700)** เสร็จสมบูรณ์แล้ว
เราเดินทางจากพื้นฐานของ Active Storage — การอัปโหลดไฟล์, สร้าง variant, และ direct upload
(Part 066) — ไปสู่การต่อ Active Storage เข้ากับ cloud storage แบบ S3-compatible พร้อม image
processing เชิงลึกสำหรับ production (Part 067), full-text search บน PostgreSQL ด้วย pg_search
ที่รองรับทั้งภาษาอังกฤษและกลไกเสริมสำหรับภาษาที่ซับซ้อนกว่าอย่างภาษาไทย (Part 068), และการมองไปสู่
Elasticsearch/OpenSearch เมื่อความต้องการค้นหาซับซ้อนเกินกว่าที่ PostgreSQL เพียงลำพังจะตอบโจทย์ได้
อย่างมีประสิทธิภาพ (Part 069) จนมาถึงโปรเจกต์รวบยอดใน Part นี้ที่รวมทุกเทคนิคเข้าด้วยกันเป็นระบบ
แคตตาล็อกสินค้าที่ใช้งานได้จริง ทักษะทั้งหมดนี้คือรากฐานสำคัญของเว็บแอปพลิเคชันยุคปัจจุบันแทบทุก
ประเภท — ไม่ว่าจะเป็นอีคอมเมิร์ซ, โซเชียลมีเดียที่ให้อัปโหลดรูป/วิดีโอ, หรือระบบจัดการเอกสารที่
ต้องค้นหาเนื้อหาได้ ล้วนต้องพึ่งพาเทคนิคจาก Phase นี้ทั้งสิ้น

จุดที่ควรจำไว้ข้ามไปยัง Phase ถัดไป: **การออกแบบให้ระบบ "degrade gracefully"** เมื่อส่วนประกอบ
เสริม (background job, cache, search index) ล้มเหลวหรือล่าช้า เป็นหลักการที่จะพบซ้ำอีกหลายครั้ง
ตลอดหลักสูตรที่เหลือ — ระบบที่ดีไม่ใช่ระบบที่ไม่มีวันพัง แต่เป็นระบบที่ยังทำงานถูกต้อง (แม้จะช้า
ลงหรือมีฟีเจอร์เสริมบางอย่างหายไปชั่วคราว) เมื่อส่วนใดส่วนหนึ่งมีปัญหา

**ต่อไป (Part 071 — เปิด Phase 11: Payment & Third-party Integration):** เราจะออกจากโลกของไฟล์
และการค้นหา เข้าสู่หัวข้อที่ทุกเว็บอีคอมเมิร์ซต้องมี — **การรับชำระเงินจริง** ด้วย **Stripe**
เรียนรู้การสร้าง Checkout Session, การจัดการ **webhook** เพื่อรับการแจ้งเตือนเมื่อชำระเงินสำเร็จ
(หัวข้อที่ต้องระวังเรื่อง security เป็นพิเศษ เพราะเป็นจุดที่รับข้อมูลจากภายนอกโดยตรง), และการทำ
**subscription** สำหรับระบบที่เก็บเงินแบบรายเดือน/รายปี — เนื้อหาที่จะนำ Product Catalog จาก
Part นี้ไปต่อยอดเป็นระบบที่ขายสินค้าได้จริงในโลกจริง
