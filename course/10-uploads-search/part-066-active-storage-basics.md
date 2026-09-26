# Part 066: Active Storage — Upload, Variant, Direct Upload — เปิด Phase 10: File Upload/Search

> **Step ครอบคลุมใน Part นี้:** Step 651–660
> **ระดับ:** ปานกลาง (ต้องผ่าน Part 025 เรื่อง ActiveRecord/migration เบื้องต้น, Part 031 เรื่อง
> `form_with`/strong parameters, และ Part 023 เรื่อง controller/params มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6, Rails 8.1.4, `image_processing` 1.14.0 (พร้อม `ruby-vips` 2.3.0
> และ `mini_magick` 5.4.0 เป็น dependency), libvips 8.15.1 (ตัวประมวลผลภาพจริงที่ติดตั้งไว้ในเครื่อง),
> `importmap-rails` 2.2.3 — ทุกคำสั่งและผลลัพธ์ในเอกสารนี้รันจริงบนแอป Rails ทดลองชื่อ `storage_demo`
> ที่สร้างขึ้นเฉพาะสำหรับ Part นี้ (ติดตั้ง Active Storage จริง, แนบไฟล์ PNG/PDF จริงที่สร้างขึ้นเอง,
> generate variant และ preview จริงด้วย libvips/poppler, ทดสอบ purge และ direct upload endpoint
> ด้วย integration test จริง) แล้วลบทิ้งหลังทดสอบเสร็จ ไม่มีตัวเลขหรือผลลัพธ์ใดที่แต่งขึ้นเอง

ยินดีต้อนรับสู่ **Phase 10: File Upload & Search** หลังจาก Phase 9 พาไปเรียนรู้การทำให้แอปเร็วขึ้น
และทนต่อโหลดมากขึ้น (background job, caching, database performance, profiling) Phase นี้จะพาไป
แก้ปัญหาที่แทบทุกเว็บแอปต้องเจอ: **ผู้ใช้ต้องการอัปโหลดไฟล์** (รูปโปรไฟล์, รูปสินค้า, เอกสาร PDF)
และ **ผู้ใช้ต้องการค้นหาข้อมูล** จากไฟล์และข้อความจำนวนมากได้อย่างรวดเร็ว Part นี้เริ่มจากครึ่งแรก
ของ Phase — **Active Storage** เฟรมเวิร์กอัปโหลดไฟล์ที่มากับ Rails เอง

ก่อนมี Active Storage (เปิดตัวพร้อม Rails 5.2 ปี 2018) นักพัฒนา Rails ต้องพึ่งพา gem ภายนอกอย่าง
**Paperclip** หรือ **CarrierWave** เพื่อจัดการการอัปโหลดไฟล์ ซึ่งแต่ละตัวมีวิธีตั้งค่าและ API ของ
ตัวเอง ต้องเขียน migration เพิ่มคอลัมน์ `avatar_file_name`, `avatar_content_type`,
`avatar_file_size` ลงในทุกตารางที่ต้องการแนบไฟล์ ทำให้ schema ของแต่ละโมเดลบวมขึ้นเรื่อยๆ ตามจำนวน
ไฟล์ที่ต้องแนบ — **Active Storage แก้ปัญหานี้ด้วยการรวมศูนย์**: ไม่ว่าจะแนบไฟล์กี่ประเภทกับกี่โมเดล
ก็ใช้แค่ 3 ตารางเดียวกันทั้งแอป และที่สำคัญที่สุดคือ **เป็นส่วนหนึ่งของ Rails framework เอง** ไม่ต้อง
ติดตั้ง gem แยกต่างหากสำหรับความสามารถหลัก (ต้องเพิ่มแค่ `image_processing` gem เพื่อ generate
thumbnail เท่านั้น)

## สารบัญของ Part นี้

- Step 651: Active Storage คืออะไร และสถาปัตยกรรม 3 ตาราง (blobs, attachments, variant_records)
- Step 652: ติดตั้ง Active Storage — `rails active_storage:install`, migrate, `config/storage.yml`
- Step 653: `has_one_attached` / `has_many_attached` และการแนบไฟล์ (ผ่านโค้ดและผ่านฟอร์ม)
- Step 654: Disk service เชิงลึก — ไฟล์เก็บอยู่ตรงไหนจริงๆ บนดิสก์ และ blob เก็บ metadata อะไรบ้าง
- Step 655: แสดงผลไฟล์ที่แนบ — `url_for`, `rails_blob_path`, `image_tag`
- Step 656: Image Variant — สร้าง thumbnail ด้วย `variant(resize_to_limit: ...)` และ
  `image_processing` + libvips/ImageMagick backend
- Step 657: `variant` vs `preview` — สร้างภาพตัวอย่างของไฟล์ที่ไม่ใช่รูปภาพ (PDF, วิดีโอ)
- Step 658: Validate attachment — เพราะ Active Storage ไม่มี validator ในตัว ต้องเขียนเอง
- Step 659: ลบไฟล์ที่แนบ — `purge` (ลบทันที) vs `purge_later` (ลบแบบ background)
- Step 660: Direct Upload — อัปโหลดตรงจากเบราว์เซอร์ไปยัง storage ก่อน submit ฟอร์ม +
  แบบฝึกหัด: ระบบอัปโหลดรูปสินค้าพร้อม validation และ thumbnail

---

## Step 651: Active Storage คืออะไร และสถาปัตยกรรม 3 ตาราง

### Active Storage คืออะไร

**Active Storage** คือเฟรมเวิร์กสำหรับแนบไฟล์ (ไปยัง cloud storage หรือ disk) เข้ากับ Active
Record model ที่ **มาพร้อมกับ Rails ตั้งแต่เวอร์ชัน 5.2** ไม่ต้องติดตั้ง gem ภายนอกเพื่อความสามารถ
หลัก (attach/detach/download ไฟล์) รองรับ backend หลายแบบในตัว — Disk (เก็บบนเครื่องเซิร์ฟเวอร์เอง,
ค่าเริ่มต้นตอน dev), Amazon S3, Google Cloud Storage, Microsoft Azure Storage, และยังรองรับการ
**mirror** ไฟล์ไปหลาย service พร้อมกันได้ในตัว (Part 067 จะเจาะลึกเรื่อง cloud storage)

### สถาปัตยกรรม: 3 ตารางที่ใช้ร่วมกันทั้งแอป

จุดที่ต่างจาก Paperclip/CarrierWave อย่างชัดเจนที่สุดคือ Active Storage **ไม่เพิ่มคอลัมน์ในตาราง
โมเดลของเราเลยสักตัว** ไม่ว่าจะแนบไฟล์กี่แบบกับกี่โมเดล ทุกอย่างถูกเก็บแยกไว้ใน 3 ตารางกลาง:

| ตาราง | หน้าที่ | คอลัมน์สำคัญ |
|---|---|---|
| `active_storage_blobs` | เก็บ**ข้อมูลอ้างอิงถึงไฟล์จริง** (metadata) — ไม่ใช่เนื้อไฟล์เอง | `key` (ชื่อไฟล์แบบสุ่มที่ใช้อ้างอิงใน storage จริง), `filename`, `content_type`, `byte_size`, `checksum`, `service_name`, `metadata` (JSON เก็บ width/height ฯลฯ) |
| `active_storage_attachments` | ตารางเชื่อม (join table) ระหว่าง blob กับโมเดลใดก็ได้ผ่าน **polymorphic association** | `record_type`, `record_id`, `name` (ชื่อ attachment เช่น "avatar"), `blob_id` |
| `active_storage_variant_records` | เก็บ record ของ **variant ที่เคย generate ไปแล้ว** เพื่อไม่ต้องประมวลผลภาพซ้ำทุกครั้ง | `blob_id` (blob ต้นฉบับ), `variation_digest` (hash ของพารามิเตอร์ resize ที่ใช้ระบุตัวตน variant นั้น) |

**หัวใจของสถาปัตยกรรมนี้คือ separation ระหว่าง "blob" กับ "attachment":**

- **Blob** คือข้อมูลเกี่ยวกับไฟล์หนึ่งไฟล์ ไม่ผูกกับโมเดลใดโมเดลหนึ่ง
- **Attachment** คือความสัมพันธ์ที่บอกว่า "โมเดล X ตัวนี้ มี attachment ชื่อ Y ที่ชี้ไปที่ blob ตัวนี้"

เพราะแยกกันแบบนี้ ไฟล์เดียวกัน (blob เดียวกัน) จึงสามารถถูกแนบเข้ากับหลาย record ได้โดยไม่ต้อง
ก็อปปี้ไฟล์ซ้ำ (เช่น ใช้รูป logo เดียวกันเป็น "cover" ของหลายสินค้า)

### ข้อมูลเชิงลึกที่ยืนยันจากซอร์สโค้ดจริง: variant ก็คือ attachment เหมือนกัน

เวลา generate variant (thumbnail) ครั้งแรก Rails ไม่ได้แค่เขียนไฟล์ภาพใหม่ลงดิสก์เฉยๆ แต่จะสร้าง
record ใน `active_storage_variant_records` ด้วย และ record นั้นเอง **ก็มี attachment ของตัวเอง
อีกชั้นหนึ่ง** — ดูจากซอร์สโค้ดจริงของ `ActiveStorage::VariantRecord`:

```ruby
# activestorage-8.1.4/app/models/active_storage/variant_record.rb (โค้ดจริงจาก gem)
class ActiveStorage::VariantRecord < ActiveStorage::Record
  belongs_to :blob
  has_one_attached :image
end
```

นั่นคือ Active Storage ใช้ฟีเจอร์ `has_one_attached` ของตัวเอง **สร้าง blob ใหม่สำหรับ variant
แล้วผูกกับ `VariantRecord` ผ่าน `has_one_attached :image`** เมื่อทดลองจริงจะเห็นข้อมูลแบบนี้ในตาราง
`active_storage_attachments`:

```
id  name    record_type              record_id  blob_id
1   avatar  User                     1          1
2   image   ActiveStorage::VariantRecord  1      2
```

Row แรกคือ attachment ของ `user.avatar` (ต้นฉบับ) ส่วน row ที่สองคือ attachment ของ variant ที่
generate ขึ้นมา (blob_id 2 คือไฟล์ thumbnail ที่ resize แล้ว) — เข้าใจสถาปัตยกรรมนี้แล้วจะเห็นภาพ
ชัดเจนว่าทำไม Step 656 ถึง generate variant ครั้งแรกช้ากว่าครั้งถัดไป (ครั้งแรกต้องประมวลผลภาพและ
สร้าง blob+attachment ใหม่ ครั้งถัดไปแค่ query หา `VariantRecord` ที่มีอยู่แล้วจาก `variation_digest`)

---

## Step 652: ติดตั้ง Active Storage — `rails active_storage:install`, migrate, `config/storage.yml`

### เพิ่ม gem ที่จำเป็น

ความสามารถหลัก (attach/detach) ไม่ต้องเพิ่ม gem อะไรเลยเพราะมากับ Rails อยู่แล้ว แต่ถ้าต้องการ
generate **image variant** (Step 656) ต้องเพิ่ม gem `image_processing` — ข่าวดีคือถ้าสร้างแอปด้วย
`rails new` แบบปกติ (ไม่ใช้ `--minimal`) ตั้งแต่ Rails 7 ขึ้นไป **บรรทัดนี้ถูกใส่ไว้ให้อัตโนมัติแล้ว**
ใน `Gemfile`:

```ruby
# Gemfile (ค่าเริ่มต้นที่ rails new ใส่ให้ ไม่ต้องเพิ่มเอง ถ้าไม่ได้ใช้ --minimal)
# Use Active Storage variants [https://guides.rubyonrails.org/active_storage_overview.html#transforming-images]
gem "image_processing", "~> 1.2"
```

ถ้าโปรเจกต์เก่าหรือสร้างด้วย `--minimal` แล้วไม่มีบรรทัดนี้ ให้เพิ่มเองแล้ว `bundle install`

### รัน generator ติดตั้ง

```bash
bin/rails active_storage:install
```

ผลลัพธ์จริงที่ได้ (ชื่อไฟล์ migration จะมี timestamp ต่างกันไปตามเวลาที่รัน):

```
Copied migration 20260926085342_create_active_storage_tables.active_storage.rb from active_storage
```

คำสั่งนี้แค่คัดลอก migration จาก gem `activestorage` มาไว้ใน `db/migrate/` ของแอปเรา ยังไม่ได้
สร้างตารางจริง — เปิดดูเนื้อหา migration ที่ได้จะเห็นการสร้างครบทั้ง 3 ตารางตามที่อธิบายใน Step 651:

```ruby
# db/migrate/xxxxxxxxxxxxxx_create_active_storage_tables.active_storage.rb
class CreateActiveStorageTables < ActiveRecord::Migration[7.0]
  def change
    create_table :active_storage_blobs, id: primary_key_type do |t|
      t.string   :key,          null: false
      t.string   :filename,     null: false
      t.string   :content_type
      t.text     :metadata
      t.string   :service_name, null: false
      t.bigint   :byte_size,    null: false
      t.string   :checksum
      t.datetime :created_at, precision: 6, null: false
      t.index [ :key ], unique: true
    end

    create_table :active_storage_attachments, id: primary_key_type do |t|
      t.string     :name,     null: false
      t.references :record,   null: false, polymorphic: true, index: false
      t.references :blob,     null: false
      t.datetime :created_at, precision: 6, null: false
      t.index [ :record_type, :record_id, :name, :blob_id ],
              name: :index_active_storage_attachments_uniqueness, unique: true
      t.foreign_key :active_storage_blobs, column: :blob_id
    end

    create_table :active_storage_variant_records, id: primary_key_type do |t|
      t.belongs_to :blob, null: false, index: false
      t.string :variation_digest, null: false
      t.index [ :blob_id, :variation_digest ],
              name: :index_active_storage_variant_records_uniqueness, unique: true
      t.foreign_key :active_storage_blobs, column: :blob_id
    end
  end
end
```

รัน migrate จริง:

```bash
bin/rails db:migrate
```

```
== 20260926085342 CreateActiveStorageTables: migrating ========================
-- create_table(:active_storage_blobs, {:id=>:primary_key})
   -> 0.0031s
-- create_table(:active_storage_attachments, {:id=>:primary_key})
   -> 0.0065s
-- create_table(:active_storage_variant_records, {:id=>:primary_key})
   -> 0.0127s
== 20260926085342 CreateActiveStorageTables: migrated (0.0225s) ===============
```

### `config/storage.yml` — ประกาศ service ที่ใช้เก็บไฟล์

`rails new` ปกติจะสร้างไฟล์นี้ให้อัตโนมัติ (เนื้อหาจริงที่ generator สร้าง):

```yaml
# config/storage.yml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# Use bin/rails credentials:edit to set the AWS secrets (as aws:access_key_id|secret_access_key)
# amazon:
#   service: S3
#   access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
#   secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
#   region: us-east-1
#   bucket: your_own_bucket-<%= Rails.env %>

# mirror:
#   service: Mirror
#   primary: local
#   mirrors: [ amazon, google, microsoft ]
```

สังเกตว่า `storage.yml` แค่ **ประกาศ** ว่า service ไหนชื่ออะไร ใช้ backend แบบไหน (`Disk`, `S3`,
`GCS`, `AzureStorage`, `Mirror`) — ต้องบอก Rails อีกทีว่า environment ไหนใช้ service ชื่ออะไร ผ่าน
`config.active_storage.service` ในแต่ละไฟล์ environment:

```ruby
# config/environments/development.rb (ค่าเริ่มต้นที่ rails new ใส่ให้)
config.active_storage.service = :local

# config/environments/test.rb (ค่าเริ่มต้นที่ rails new ใส่ให้)
config.active_storage.service = :test
```

สังเกตว่า **test environment ใช้ service ชื่อ `test` แยกจาก `local` ที่ dev ใช้** — และ root ของ
`test` ชี้ไปที่ `tmp/storage` ไม่ใช่ `storage/` เพื่อไม่ให้ไฟล์ที่สร้างขึ้นตอนรัน test ปนกับไฟล์จริง
ตอน dev (และ `tmp/` มักถูกล้างทิ้งเป็นประจำโดยไม่กระทบข้อมูลจริง) ค่าเริ่มต้นทั้งสองนี้ใช้ backend
`Disk` เหมือนกัน ต่างกันแค่ path — Production ควรเปลี่ยนไปใช้ service อย่าง `amazon` (S3) แทน ซึ่ง
Part 067 จะเจาะลึกเรื่องนี้

---

## Step 653: `has_one_attached` / `has_many_attached` และการแนบไฟล์

### ประกาศความสามารถในการแนบไฟล์บนโมเดล

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar        # แนบได้ไฟล์เดียว (เช่น รูปโปรไฟล์)
  has_many_attached :documents    # แนบได้หลายไฟล์ (เช่น เอกสารแนบ)
end
```

`has_one_attached`/`has_many_attached` ไม่ได้เพิ่มคอลัมน์ในตาราง `users` เลยแม้แต่ตัวเดียว — มันแค่
ประกาศ association ไปยัง `active_storage_attachments` โดยใช้ `name: "avatar"` (หรือ `"documents"`)
เป็นตัวระบุ ทำให้ 1 โมเดลแนบไฟล์ได้หลายชนิดพร้อมกันโดยไม่ชนกัน

### แนบไฟล์ผ่านโค้ด (เช่นใน console, seed, background job)

```ruby
user = User.create!(name: "Somchai")

user.avatar.attach(
  io: File.open("avatar.png"),
  filename: "avatar.png",
  content_type: "image/png"
)

user.avatar.attached?     # => true
user.avatar.filename      # => "avatar.png" (เป็น ActiveStorage::Filename object)
user.avatar.byte_size     # => 3002
user.avatar.content_type  # => "image/png"
```

ผลลัพธ์จริงจากการทดสอบบนแอป `storage_demo` (สร้างไฟล์ PNG ทดสอบขนาด 800x600 ด้วย libvips แล้ว
แนบเข้ากับ user):

```
attached? true
filename: avatar.png
byte_size: 3002
content_type: image/png
key: q7ucjy5v6kglrfb1jkkfdpxd6047
blob service_name: local
```

### แนบไฟล์ผ่านฟอร์ม (วิธีที่ใช้จริงในเว็บแอป)

```erb
<%# app/views/products/new.html.erb %>
<%= form_with model: @product, url: products_path do |f| %>
  <%= f.text_field :name %>
  <%= f.file_field :photo %>
  <%= f.submit "บันทึก" %>
<% end %>
```

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  def create
    @product = Product.new(product_params)
    if @product.save
      redirect_to products_path
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def product_params
    # ต้อง permit :photo เหมือน parameter ตัวอื่นๆ — Active Storage จัดการที่เหลือให้เอง
    params.require(:product).permit(:name, :price, :photo)
  end
end
```

จุดสำคัญคือ **ไม่ต้องเขียนโค้ดจัดการไฟล์เพิ่มเองเลย** — แค่ `permit :photo` เหมือน parameter
ปกติ แล้วเรียก `@product.save` ครั้งเดียว Active Storage จะสร้าง blob, อัปโหลดไฟล์ไปยัง service
ที่ตั้งไว้ และสร้าง attachment ให้ครบในทีเดียว ยืนยันด้วย integration test จริงที่ POST ไฟล์แบบ
multipart form:

```ruby
test "creates product with uploaded photo via multipart form" do
  file = fixture_file_upload("test_avatar.png", "image/png")

  assert_difference -> { Product.count }, 1 do
    post products_url, params: { product: { name: "รองเท้า", photo: file } }
  end

  product = Product.last
  assert product.photo.attached?
  assert_equal "test_avatar.png", product.photo.filename.to_s
end
```

รันแล้วผ่านจริง: `1 runs, 7 assertions, 0 failures, 0 errors`

---

## Step 654: Disk service เชิงลึก — ไฟล์เก็บอยู่ตรงไหนจริงๆ บนดิสก์

### โครงสร้าง path บนดิสก์

หลังแนบไฟล์เข้ากับ user ในตัวอย่าง Step 653 ลอง `find` ดูจริงในโฟลเดอร์ `storage/` ของแอป:

```bash
find storage -type f
```

```
storage/46/qn/46qnt2r0n8xd7xoqxcenrwvj32oz
storage/q7/uc/q7ucjy5v6kglrfb1jkkfdpxd6047
```

สังเกต pattern: **`storage/<key 2 ตัวแรก>/<key ตัวที่ 3-4>/<key เต็ม>`** — `key` คือ token สุ่ม
(ไม่ใช่ชื่อไฟล์เดิมของผู้ใช้) ที่ generate ตอนสร้าง blob เอาไว้ใช้อ้างอิงไฟล์จริง เหตุผลที่ต้องแยก
เป็นโฟลเดอร์ย่อยแบบนี้คือ **ป้องกันปัญหา filesystem ช้าเมื่อมีไฟล์นับล้านไฟล์อยู่ในโฟลเดอร์เดียวกัน**
(หลาย filesystem เริ่มมีปัญหา performance เมื่อโฟลเดอร์เดียวมีไฟล์เกินหลักหมื่น-แสน) การแบ่งเป็น
โฟลเดอร์ย่อยตามตัวอักษรของ key ช่วยกระจายไฟล์ออกไปหลายพันโฟลเดอร์ย่อยโดยอัตโนมัติ

**ชื่อไฟล์เดิมของผู้ใช้ไม่ได้หายไปไหน** — มันถูกเก็บไว้ในคอลัมน์ `filename` ของตาราง
`active_storage_blobs` ต่างหาก แล้วค่อยเอามาใช้ตอน generate URL สำหรับดาวน์โหลด (ดู Step 655) ไฟล์
บนดิสก์เองไม่มีนามสกุลติดอยู่เลยด้วยซ้ำ — Active Storage ใช้ `content_type` ที่เก็บในฐานข้อมูล
กำหนด `Content-Type` header ตอน serve ไฟล์แทน

### ข้อมูลที่ blob เก็บไว้จริง (metadata)

```ruby
blob = ActiveStorage::Blob.first
blob.attributes
```

```ruby
{
  "id"           => 1,
  "key"          => "q7ucjy5v6kglrfb1jkkfdpxd6047",
  "filename"     => "avatar.png",
  "content_type" => "image/png",
  "metadata"     => { "identified" => true, "width" => 800, "height" => 600, "analyzed" => true },
  "service_name" => "local",
  "byte_size"    => 3002,
  "checksum"     => "hkxzgbQ913Z/p/OFb7yxqQ==",
  "created_at"   => 2026-09-26 08:54:50 UTC
}
```

จุดที่ควรสังเกต:

- **`content_type` ไม่ได้เชื่อค่าจากฝั่ง client แบบดิบๆ 100%** — ตอนแนบไฟล์ Active Storage เรียก
  gem `marcel` (ตัวตรวจสอบ MIME type จากเนื้อหาไฟล์จริง ไม่ใช่แค่ดูนามสกุล) ให้ช่วย "ยืนยัน"
  ประเภทไฟล์ผ่าน `identify_without_saving` — `metadata["identified"] => true` คือหลักฐานว่าผ่าน
  ขั้นตอนนี้แล้ว (แต่ไม่ใช่การป้องกันการปลอมแปลงแบบสมบูรณ์ 100% — รายละเอียดเชิงความปลอดภัยเจาะลึก
  กว่านี้จะอยู่ใน Phase 13: Security)
- **`metadata["width"]`/`["height"]`** ถูกเติมโดยอัตโนมัติสำหรับไฟล์รูปภาพ ผ่านขั้นตอน "analyze"
  (`metadata["analyzed"] => true`) มีประโยชน์มากตอนต้องแสดงผล `<img>` พร้อม `width`/`height`
  attribute ล่วงหน้าโดยไม่ต้องเปิดไฟล์จริง (ช่วยลด Cumulative Layout Shift ฝั่ง frontend)
- **`service_name`** บอกว่า blob นี้เก็บอยู่ใน service ไหนตาม `storage.yml` — ฟิลด์นี้สำคัญมากตอน
  ทำ Part 067 ที่มีหลาย service พร้อมกัน (local + S3) เพราะ Active Storage รู้ได้จากฟิลด์นี้เองว่า
  ต้องไปดึงไฟล์จากที่ไหน โดยไม่ต้องเขียนโค้ดแยกเงื่อนไขเอง

---

## Step 655: แสดงผลไฟล์ที่แนบ — `url_for`, `rails_blob_path`, `image_tag`

### สร้าง URL ไปยังไฟล์

```ruby
# ใน view หรือ controller (มี request context ให้ Rails รู้ host อัตโนมัติ)
url_for(user.avatar)                       # URL แบบเต็ม (มี host)
rails_blob_path(user.avatar, only_path: true)  # path อย่างเดียว (ไม่มี host)
```

ผลลัพธ์จริงที่ได้ (ทดสอบผ่าน `Rails.application.routes.url_helpers`):

```
/rails/active_storage/blobs/redirect/eyJfcmFpbHMiOns...--30e912c9.../avatar.png
```

สังเกตว่า URL ที่ได้ **ไม่ใช่ path ตรงไปยังไฟล์บนดิสก์เลย** (ไม่ใช่ `/storage/q7/uc/q7uc...`) แต่เป็น
route พิเศษ `/rails/active_storage/blobs/redirect/<signed_id>/<filename>` — ส่วน
`eyJfcmFpbHMiOns...` คือ **signed ID** (เข้ารหัสด้วย `Rails.application.message_verifier` แบบเดียว
กับ session cookie) ที่ห่อ blob id เอาไว้ ป้องกันไม่ให้ใครเดา URL ไฟล์คนอื่นได้จากการนับเลข id ไป
เรื่อยๆ เมื่อ browser เรียก URL นี้ Active Storage จะตรวจสอบ signed ID แล้ว **redirect** ไปยัง URL
จริงของไฟล์ (ถ้าเป็น disk service ในตัวอย่างนี้จะ redirect ไปที่ route ภายในอีกทีเพื่อ stream ไฟล์
ออกมา, ถ้าเป็น S3 จะ redirect ไปที่ presigned URL ของ S3 โดยตรง)

### ใช้ใน view จริงด้วย `image_tag`

```erb
<%= image_tag user.avatar %>
```

Rails ฉลาดพอที่จะรับ attachment object ส่งเข้า `image_tag` ได้โดยตรง ไม่ต้องเรียก `url_for` เอง
ก่อน — ยืนยันจาก integration test จริงที่ render view แล้วตรวจ HTML ที่ได้:

```erb
<%# app/views/products/index.html.erb %>
<% if product.photo.attached? %>
  <%= image_tag product.photo %>
<% end %>
```

**ข้อควรระวังที่พลาดบ่อย:** ต้องเช็ก `product.photo.attached?` ก่อนเสมอ ถ้าไม่มีไฟล์แนบแล้วเรียก
`image_tag product.photo` ตรงๆ จะได้ URL ที่ไม่มีความหมาย (ไม่ error แต่ได้ `<img>` ที่ src ว่าง/ผิด)
— แนวปฏิบัติที่ดีคือเตรียม placeholder image ไว้แสดงกรณีไม่มีไฟล์แนบเสมอ

---

## Step 656: Image Variant — สร้าง thumbnail ด้วย `variant`

### ทำไมต้องมี variant

ไฟล์ต้นฉบับที่ผู้ใช้อัปโหลดมักมีขนาดใหญ่เกินความจำเป็น (รูปจากกล้องมือถือสมัยนี้ 4000×3000 พิกเซล
ขึ้นไป) การแสดงรูปขนาดนั้นเต็มๆ ในหน้า listing ที่ต้องการแค่ thumbnail 150×150 พิกเซลเป็นการสิ้นเปลือง
bandwidth มหาศาล **Variant** คือกลไกของ Active Storage สำหรับสร้างรูปเวอร์ชันย่อ/แปลงรูปแบบ
"ตามต้องการ" (on-the-fly) โดยไม่ต้องเขียนไฟล์ resize ไว้ล่วงหน้าเอง

### ใช้งานจริง

```ruby
user.avatar.variant(resize_to_limit: [200, 200])
```

```erb
<%= image_tag user.avatar.variant(resize_to_limit: [200, 200]) %>
```

ทดสอบจริงบนแอป `storage_demo`:

```ruby
variant = user.avatar.variant(resize_to_limit: [200, 200])
processed = variant.processed
processed.key      # => "46qnt2r0n8xd7xoqxcenrwvj32oz" (blob ใหม่ที่สร้างขึ้นสำหรับ variant นี้)
```

`resize_to_limit: [200, 200]` หมายถึง **ย่อรูปให้พอดีกับกรอบ 200×200 โดยรักษาสัดส่วนเดิมไว้** (ไม่
บิดเบี้ยว ไม่ครอบตัด) — ถ้ารูปต้นฉบับมีสัดส่วนไม่เป็นสี่เหลี่ยมจัตุรัส ด้านที่สั้นกว่าจะเล็กกว่า 200
ตัวเลือก resize อื่นที่ใช้บ่อย ได้แก่ `resize_to_fill` (ครอบตัดให้เต็มกรอบพอดี เหมาะกับ avatar
วงกลม), `resize_to_fit` (เหมือน `resize_to_limit` แต่ขยายรูปเล็กให้ใหญ่ขึ้นได้ด้วยถ้าจำเป็น)

### `image_processing` gem และ backend libvips/ImageMagick

Active Storage เองไม่มีความสามารถประมวลผลภาพในตัว — มันมอบหมายงานนี้ให้ gem **`image_processing`**
ซึ่งเป็น wrapper บาง (thin wrapper) ที่คุยกับโปรแกรมประมวลผลภาพจริงที่ติดตั้งไว้ในเครื่อง (ไม่ใช่
ประมวลผลด้วย Ruby ล้วนๆ) รองรับ 2 backend:

| Backend | โปรแกรมที่ต้องติดตั้งในเครื่อง | gem ที่ image_processing ใช้คุยด้วย |
|---|---|---|
| **libvips** (แนะนำ, ค่าเริ่มต้นของ Rails ตั้งแต่ 7.1) | `vips` | `ruby-vips` |
| ImageMagick | `convert`/`magick` | `mini_magick` |

ตรวจสอบว่า Rails กำหนด backend ไหนเป็นค่าเริ่มต้นด้วยคำสั่งนี้:

```ruby
Rails.application.config.active_storage.variant_processor
# => :vips
```

ยืนยันจริงบนแอปทดสอบ ได้ผลลัพธ์ `:vips` ตรงกับที่คาดไว้ (Rails 7.1 ขึ้นไปเปลี่ยนค่าเริ่มต้นจาก
`:mini_magick` มาเป็น `:vips` เพราะ libvips **เร็วกว่าและใช้ RAM น้อยกว่า ImageMagick อย่างมาก**
โดยเฉพาะกับไฟล์ขนาดใหญ่)

**ข้อสังเกตที่ทำให้หลายคนสับสน:** เปิด `Gemfile.lock` จะเห็นทั้ง `ruby-vips` และ `mini_magick` ถูก
bundle มาด้วยกันเสมอ (เพราะเป็น dependency ของ `image_processing` gem ทั้งคู่) แต่ **เครื่องจริง
(server) ต้องติดตั้งโปรแกรมของ backend ที่เลือกใช้จริงเพียงตัวเดียว** — ถ้าตั้ง
`variant_processor = :vips` ก็ต้องติดตั้งโปรแกรม `vips` ในเครื่อง (เช่น `apt install libvips-tools`
บน Ubuntu หรือ `brew install vips` บน macOS) โดยไม่จำเป็นต้องมี ImageMagick เลย — gem ทั้งสองแค่
"พร้อมใช้" แต่จะทำงานได้จริงก็ต่อเมื่อโปรแกรมเบื้องหลังมีอยู่ในเครื่องเท่านั้น (ในสภาพแวดล้อมที่ใช้
ทดสอบเอกสารนี้มีแค่ `vips` ติดตั้งไว้ ไม่มี ImageMagick เลย และทุก variant ก็ generate สำเร็จโดยใช้
`:vips` ตามค่าเริ่มต้น)

> **หมายเหตุ Docker/Production:** ต้องเพิ่ม `libvips` (หรือ `imagemagick`) ลงใน Dockerfile หรือ
> package ของ server เสมอ — ลืมขั้นตอนนี้เป็นสาเหตุอันดับต้นๆ ที่ variant พังใน production ทั้งที่
> รันได้ปกติตอน dev (เพราะเครื่อง dev มักติดตั้ง libvips ไว้อยู่แล้วโดยไม่รู้ตัว ผ่าน Homebrew หรือ
> package อื่น)

---

## Step 657: `variant` vs `preview` — ภาพตัวอย่างของไฟล์ที่ไม่ใช่รูปภาพ

`variant` ใช้ได้กับไฟล์ที่**เป็นรูปภาพอยู่แล้ว**เท่านั้น (ย่อ/ขยาย/แปลง format รูปเดิม) แต่ถ้าไฟล์ที่
แนบเป็น **PDF หรือวิดีโอ** ซึ่งไม่ใช่รูปภาพโดยตรง ต้องใช้ **`preview`** แทน — `preview` จะสร้าง
"ภาพตัวแทน" ของไฟล์นั้นขึ้นมาก่อน (เช่น หน้าแรกของ PDF, เฟรมแรกของวิดีโอ) แล้วค่อย resize ภาพตัวแทน
นั้นเหมือน variant ปกติ

### ตัวอย่างจริงกับไฟล์ PDF

```ruby
user.documents.attach(
  io: File.open("report.pdf"),
  filename: "report.pdf",
  content_type: "application/pdf"
)

doc = user.documents.last
preview = doc.preview(resize_to_limit: [200, 200])
processed = preview.processed
processed.image.content_type  # => "image/png"
```

ทดสอบจริงด้วยไฟล์ PDF ขนาดเล็กที่สร้างขึ้นเอง ได้ผลลัพธ์:

```
preview blob key: 9wcefpgylq9fkeh5bzw6lg62feid
preview content_type: image/png
```

Active Storage เลือก **previewer** ที่เหมาะสมให้อัตโนมัติจากรายการที่มีอยู่ ตรวจสอบได้ด้วย:

```ruby
ActiveStorage.previewers
# => [ActiveStorage::Previewer::PopplerPDFPreviewer,
#     ActiveStorage::Previewer::MuPDFPreviewer,
#     ActiveStorage::Previewer::VideoPreviewer]
```

| Previewer | ใช้กับไฟล์ | ต้องติดตั้งโปรแกรม |
|---|---|---|
| `PopplerPDFPreviewer` | PDF | `poppler-utils` (คำสั่ง `pdftoppm`) |
| `MuPDFPreviewer` | PDF (ทางเลือกสำรอง) | `mupdf` (คำสั่ง `mutool`) |
| `VideoPreviewer` | วิดีโอ (mp4, mov ฯลฯ) | `ffmpeg` |

Active Storage จะลองใช้ previewer ตัวแรกที่ `accept?` คืนค่า true (คือมีไฟล์ต้องตรง content_type
และมีโปรแกรมที่จำเป็นติดตั้งอยู่จริงในเครื่อง) — ในการทดสอบเอกสารนี้เครื่องมี `poppler-utils`
ติดตั้งไว้ (`pdftoppm` ใช้งานได้) ทำให้ `PopplerPDFPreviewer` ถูกเลือกใช้และสร้าง preview สำเร็จ

### ใช้ preview ใน view พร้อม fallback

```erb
<% if doc.content_type.in?(%w[image/png image/jpeg image/webp]) %>
  <%= image_tag doc.variant(resize_to_limit: [200, 200]) %>
<% elsif doc.previewable? %>
  <%= image_tag doc.preview(resize_to_limit: [200, 200]) %>
<% else %>
  <%= link_to doc.filename, rails_blob_path(doc) %>
<% end %>
```

`previewable?` เช็กว่ามี previewer ตัวไหนรองรับไฟล์นี้ไหมก่อนเรียก `.preview` ป้องกัน error กรณี
ไฟล์เป็นประเภทที่ไม่มี previewer รองรับเลย (เช่นไฟล์ `.zip` หรือ `.docx`) — กรณีนั้นควรแสดงแค่ link
ดาวน์โหลดธรรมดาแทน

---

## Step 658: Validate attachment — ต้องเขียนเอง เพราะ Active Storage ไม่มี validator ในตัว

### ทำไมต้องเขียนเอง

ต่างจาก column ปกติของ ActiveRecord ที่มี `validates :name, presence: true` ใช้ได้ทันที **Active
Storage ไม่มี validator สำเร็จรูปสำหรับตรวจสอบ attachment เลย** (ไม่มี `validates :photo,
attached: true` หรือ `content_type: { in: [...] }` ให้ใช้ตรงๆ) เพราะ attachment ไม่ใช่ attribute
ปกติของโมเดล (ไม่ได้เป็นคอลัมน์ในตาราง) จึงต้องเขียน custom validation method เอง

### ตัวอย่างที่ทดสอบจริงและใช้งานได้

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  has_one_attached :photo

  validates :name, presence: true
  validate :photo_must_be_present
  validate :photo_content_type
  validate :photo_size

  private

  def photo_must_be_present
    errors.add(:photo, "ต้องแนบรูปสินค้า") unless photo.attached?
  end

  def photo_content_type
    return unless photo.attached?

    acceptable_types = %w[image/png image/jpeg image/webp]
    unless photo.content_type.in?(acceptable_types)
      errors.add(:photo, "ต้องเป็นไฟล์ PNG, JPEG หรือ WEBP เท่านั้น")
    end
  end

  def photo_size
    return unless photo.attached?

    if photo.blob.byte_size > 5.megabytes
      errors.add(:photo, "ต้องมีขนาดไม่เกิน 5MB")
    end
  end
end
```

**จุดสำคัญที่พลาดไม่ได้:** ทุก custom validation ที่เช็ก property ของไฟล์ (`content_type`,
`byte_size`) ต้องมี guard clause `return unless photo.attached?` ก่อนเสมอ — ถ้าไม่มี attachment
เลย การเรียก `photo.content_type` จะได้ `nil` แล้วเช็ก `.in?(...)` ก็ได้แค่ `false` เฉยๆ (ไม่ error
แต่จะซ้ำซ้อนกับ error message ของ `photo_must_be_present`)

ทดสอบจริง 3 กรณี:

```ruby
# กรณี 1: แนบไฟล์ .txt โดยตั้ง content_type ปลอมเป็น text/plain
p1 = Product.new(name: "เสื้อยืด")
p1.photo.attach(io: File.open("bad.txt"), filename: "bad.txt", content_type: "text/plain")
p1.valid?  # => false
p1.errors.full_messages  # => ["Photo ต้องเป็นไฟล์ PNG, JPEG หรือ WEBP เท่านั้น"]

# กรณี 2: ไม่แนบไฟล์เลย
p2 = Product.new(name: "กางเกงยีนส์")
p2.valid?  # => false
p2.errors.full_messages  # => ["Photo ต้องแนบรูปสินค้า"]

# กรณี 3: แนบไฟล์ PNG ที่ถูกต้อง
p3 = Product.new(name: "หมวก")
p3.photo.attach(io: File.open("hat.png"), filename: "hat.png", content_type: "image/png")
p3.valid?  # => true
p3.save    # => true
```

ทั้ง 3 กรณีให้ผลตรงตามที่ออกแบบไว้ทุกกรณี (ยืนยันด้วย `rails runner` จริง)

> **หมายเหตุเรื่อง i18n:** ข้อความ error ด้านบนแสดงชื่อ attribute เป็น "Photo" (humanize จากชื่อ
> attribute ภาษาอังกฤษ) ถ้าต้องการให้ขึ้นเป็นภาษาไทยทั้งหมด (เช่น "รูปสินค้า ต้องแนบรูปสินค้า") ให้
> เพิ่มคำแปลใน `config/locales/th.yml` ภายใต้ `activerecord.attributes.product.photo` ตามหลักการที่
> เคยเรียนไปแล้วใน Part 039 (Internationalization)

---

## Step 659: ลบไฟล์ที่แนบ — `purge` vs `purge_later`

### `purge` — ลบทันที (synchronous)

```ruby
user.avatar.purge
```

`purge` จะ**ลบทั้งไฟล์จริงบนดิสก์/cloud storage และ record ของ blob ในฐานข้อมูลทันที** แบบ
synchronous (บล็อก request ที่เรียกจนกว่าจะลบเสร็จ) ทดสอบจริง:

```ruby
key = user.avatar.blob.key
path = user.avatar.blob.service.path_for(key)

File.exist?(path)   # => true (ก่อน purge)
user.avatar.purge
user.avatar.attached?  # => false
File.exist?(path)      # => false (หลัง purge — ไฟล์บนดิสก์หายไปจริง)
```

### `purge_later` — ลบแบบ background (asynchronous)

```ruby
user.avatar.purge_later
```

`purge_later` **ตัดความสัมพันธ์ (ลบ attachment record) ทันที** แต่ผลักภาระการลบไฟล์จริง (I/O ที่
อาจช้า โดยเฉพาะกับ cloud storage ที่ต้องยิง API call ออกไปข้างนอก) ไปทำใน background ผ่าน
`ActiveStorage::PurgeJob` (ActiveJob) ทดสอบจริง:

```ruby
puts Rails.application.config.active_job.queue_adapter  # => :async

user.avatar.attach(io: File.open("avatar2.png"), filename: "avatar2.png", content_type: "image/png")
user.avatar.attached?  # => true

user.avatar.purge_later
user.avatar.attached?  # => false  (แยกออกจาก record ทันที แม้ไฟล์จริงบนดิสก์ยังไม่ถูกลบ)
```

**เมื่อไหร่ควรใช้แบบไหน:** ใช้ `purge_later` เป็นค่าเริ่มต้นในโค้ดที่ตอบสนอง user request โดยตรง
(เช่น action `destroy` ของ controller ที่ user กดปุ่ม "ลบสินค้า") เพื่อไม่ให้ user ต้องรอ I/O การลบ
ไฟล์ทำงานเสร็จก่อน server จะตอบกลับ — หลักการเดียวกับที่เรียนไปใน Phase 9 เรื่องการย้ายงานที่ไม่
จำเป็นต้องเสร็จก่อนตอบ user ไปทำเบื้องหลัง (Part 061) ส่วน `purge` แบบ synchronous เหมาะกับสถานการณ์
ที่ต้องมั่นใจว่าไฟล์ถูกลบเสร็จแล้วจริงๆ ก่อนจะทำขั้นตอนถัดไป เช่น script migration ข้อมูล หรือ
เทส

### จุดที่ควรรู้: `dependent: :purge_later` บน association

ถ้าลบ record ทั้งตัว (เช่น `user.destroy`) โดยไม่ได้สั่ง purge attachment เองก่อน ไฟล์ที่เคยแนบไว้
จะ**กลายเป็น orphan** (blob ยังอยู่ในฐานข้อมูล ไฟล์ยังอยู่บน storage แต่ไม่มี record ไหนอ้างอิงถึง
แล้ว) เพราะ `has_one_attached`/`has_many_attached` ไม่ผูก `dependent: :destroy` ให้อัตโนมัติ
ถ้าต้องการให้ลบไฟล์ตามไปด้วยเมื่อลบ record ต้นทาง ให้ระบุ:

```ruby
class User < ApplicationRecord
  has_one_attached :avatar, dependent: :purge_later
end
```

---

## Step 660: Direct Upload — อัปโหลดตรงจากเบราว์เซอร์ไปยัง storage

### ปัญหาของการอัปโหลดแบบปกติ

เวลาใช้ `f.file_field :photo` ธรรมดาแล้ว submit ฟอร์ม ไฟล์จะถูกส่งผ่าน **Rails server ก่อนเสมอ** —
เบราว์เซอร์อัปโหลดไฟล์มาที่ Puma worker, worker ต้องรับข้อมูลไฟล์ทั้งหมดเข้ามาในหน่วยความจำ/ดิสก์
ชั่วคราว ก่อนจะส่งต่อไปเก็บที่ storage จริง (ไม่ว่าจะเป็น disk หรือ S3) ถ้าไฟล์ใหญ่หรือ connection
ผู้ใช้ช้า **worker process นั้นจะถูกจองไว้ทั้งหมดตลอดเวลาที่รอรับไฟล์** ไม่สามารถไปตอบ request อื่น
ได้ — เป็นปัญหาด้าน scalability ตรงกับสิ่งที่เรียนมาใน Phase 9 เรื่อง performance/concurrency

**Direct Upload** แก้ปัญหานี้ด้วยการให้ **เบราว์เซอร์อัปโหลดไฟล์ตรงไปยัง storage เอง** โดยไม่ผ่าน
Rails server เลย (Rails server รับแค่ metadata เล็กๆ ก่อน/หลัง ไม่ต้องรับไฟล์จริง)

### เปิดใช้งานในฟอร์ม

```erb
<%= form_with model: @product, url: products_path do |f| %>
  <%= f.file_field :photo, direct_upload: true %>
<% end %>
```

HTML ที่ได้จริงจากการ render (ยืนยันด้วย integration test):

```html
<input data-direct-upload-url="http://www.example.com/rails/active_storage/direct_uploads"
       type="file" name="product[photo]" id="product_photo" />
```

แค่เพิ่ม attribute `data-direct-upload-url` แต่ตัว `<input>` ยังเป็น `type="file"` ธรรมดา — ที่ทำให้
มันทำงานแบบ "direct" ได้จริงคือ **JavaScript** ที่ต้องติดตั้งแยกต่างหาก

### ติดตั้ง JavaScript ที่จำเป็น (`@rails/activestorage`)

ถ้าโปรเจกต์ใช้ **importmap** (ค่าเริ่มต้นของ Rails 8 เมื่อสร้างแอปด้วย `rails new` ธรรมดา):

```bash
bin/importmap pin @rails/activestorage
```

```ruby
# app/javascript/application.js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

ถ้าใช้ jsbundling (esbuild/webpack) แทน ให้ติดตั้งผ่าน npm/yarn ตามปกติ:

```bash
yarn add @rails/activestorage
```

```js
// app/javascript/application.js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

`ActiveStorage.start()` จะไป "ดัก" ทุก `<input type="file" data-direct-upload-url="...">` ที่มีอยู่
ในหน้าเว็บ แล้วเปลี่ยนพฤติกรรมตอน submit ฟอร์มที่มี input นั้นอยู่ให้ทำงานตามขั้นตอนถัดไป

### ขั้นตอนที่เกิดขึ้นจริงเบื้องหลัง (ยืนยันด้วยการยิง request จริงตรงไปที่ endpoint)

1. ผู้ใช้เลือกไฟล์ผ่าน `<input type="file">` ตามปกติ
2. เมื่อกดปุ่ม submit, JavaScript ดักไว้ก่อน แล้ว **POST ข้อมูล metadata ของไฟล์** (ไม่ใช่ตัวไฟล์)
   ไปที่ endpoint `/rails/active_storage/direct_uploads`:

   ```json
   {
     "blob": {
       "filename": "avatar.png",
       "content_type": "image/png",
       "byte_size": 3002,
       "checksum": "hkxzgbQ913Z/p/OFb7yxqQ=="
     }
   }
   ```

3. Rails server สร้าง blob record ในฐานข้อมูล (ยังไม่มีไฟล์จริง) แล้วตอบกลับด้วย `signed_id` และ
   **presigned URL** สำหรับอัปโหลดไฟล์จริง — ผลลัพธ์จริงที่ได้จากการทดสอบ endpoint นี้ตรงๆ:

   ```json
   {
     "id": 1,
     "key": "u7h0ka30ggwi0tnwwspf5d1dmne5",
     "filename": "avatar.png",
     "content_type": "image/png",
     "byte_size": 3002,
     "checksum": "hkxzgbQ913Z/p/OFb7yxqQ==",
     "signed_id": "eyJfcmFpbHMiOns...",
     "direct_upload": {
       "url": "http://www.example.com/rails/active_storage/disk/eyJfcmFpbHMiOns.../...",
       "headers": { "Content-Type": "image/png" }
     }
   }
   ```

   (ในตัวอย่างนี้ backend เป็น local disk service `direct_upload.url` จึงชี้กลับมาที่ route ภายใน
   แอปเอง — ถ้าเป็น S3 ค่านี้จะเป็น **presigned S3 URL** ที่ชี้ตรงไปยัง AWS โดยไม่ผ่าน Rails
   เลยแม้แต่ขั้นตอนเดียว รายละเอียดจะอยู่ใน Part 067)
4. JavaScript ใช้ `direct_upload.url` และ `headers` ที่ได้มา **PUT ไฟล์จริงตรงไปยัง storage** (ไม่
   ผ่าน Rails controller action ใดๆ เลย)
5. ระหว่างขั้นตอนที่ 4 นี้ JavaScript จะยิง custom event ให้ front-end ใช้แสดง **progress bar** ได้:
   `direct-uploads:start`, `direct-upload:progress` (มี `event.detail.progress` เป็นเปอร์เซ็นต์),
   `direct-upload:error`, `direct-uploads:end`
6. อัปโหลดเสร็จ JavaScript จะแทนที่ `<input type="file">` เดิมด้วย **hidden field ที่เก็บแค่
   `signed_id`** (string สั้นๆ) แล้วปล่อยให้ฟอร์ม submit ต่อไปตามปกติ
7. Controller ฝั่ง Rails ได้รับแค่ `signed_id` ใน params (ไม่ใช่ไฟล์ดิบอีกต่อไป) แล้ว
   `product.photo.attach(params[:product][:photo])` ก็ทำงานได้ปกติ — โค้ด controller **ไม่ต้องแก้
   อะไรเลย** ทั้งที่ flow เบื้องหลังเปลี่ยนไปทั้งหมด

### ทำไม Direct Upload ถึงคุ้มค่า

| ด้าน | อัปโหลดแบบปกติ | Direct Upload |
|---|---|---|
| ไฟล์ผ่าน Puma worker ไหม | ผ่าน (worker ถูกจองไว้ตลอดเวลาที่รับไฟล์) | ไม่ผ่านเลย (worker รับแค่ metadata เล็กๆ) |
| Progress bar | ทำเองยาก (ต้องเขียน AJAX upload เอง) | มีให้พร้อมผ่าน custom event |
| เหมาะกับไฟล์ขนาดใหญ่ | ไม่เหมาะ (เสี่ยง timeout, กิน memory server) | เหมาะมาก โดยเฉพาะกับ S3 |
| ต้องแก้โค้ด controller | - | ไม่ต้องแก้เลย (ยังคง `attach` แบบเดิม) |

จุดที่เชื่อมกับ Phase 9 ที่เพิ่งเรียนจบไป: Direct Upload คือหลักการเดียวกับ background job — **ย้าย
งานที่ไม่จำเป็นต้องให้ Rails process ทำเอง ออกไปให้ระบบอื่นทำแทน** เพียงแต่ที่นี่ระบบอื่นคือ
"เบราว์เซอร์ของผู้ใช้เอง" ที่คุยตรงกับ storage แทนที่จะให้ Rails เป็นตัวกลางที่ต้องแบกภาระ I/O นั้น

---

## แบบฝึกหัด: ระบบอัปโหลดรูปสินค้าพร้อม validation และ thumbnail

### โจทย์

สร้างโมเดล `Product` ที่:

1. มี `has_one_attached :photo`
2. Validate ว่าต้องมีรูปแนบเสมอ, ต้องเป็นไฟล์ PNG/JPEG/WEBP เท่านั้น, และขนาดไม่เกิน 5MB
3. หน้า index แสดงรายการสินค้าพร้อม **thumbnail ขนาด 150×150** ของแต่ละสินค้า

### เฉลย

**Model** (เหมือนที่ทดสอบจริงใน Step 658):

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  has_one_attached :photo

  validates :name, presence: true
  validate :photo_must_be_present
  validate :photo_content_type
  validate :photo_size

  private

  def photo_must_be_present
    errors.add(:photo, "ต้องแนบรูปสินค้า") unless photo.attached?
  end

  def photo_content_type
    return unless photo.attached?

    acceptable_types = %w[image/png image/jpeg image/webp]
    unless photo.content_type.in?(acceptable_types)
      errors.add(:photo, "ต้องเป็นไฟล์ PNG, JPEG หรือ WEBP เท่านั้น")
    end
  end

  def photo_size
    return unless photo.attached?

    if photo.blob.byte_size > 5.megabytes
      errors.add(:photo, "ต้องมีขนาดไม่เกิน 5MB")
    end
  end
end
```

**Migration** (คอลัมน์ธรรมดา ไม่มีอะไรเกี่ยวกับไฟล์เลย เพราะ Active Storage จัดการแยกต่างหาก):

```ruby
class CreateProducts < ActiveRecord::Migration[8.1]
  def change
    create_table :products do |t|
      t.string  :name
      t.integer :price
      t.timestamps
    end
  end
end
```

**Controller:**

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  def index
    @products = Product.all
  end

  def new
    @product = Product.new
  end

  def create
    @product = Product.new(product_params)
    if @product.save
      redirect_to products_path, notice: "เพิ่มสินค้าเรียบร้อย"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def product_params
    params.require(:product).permit(:name, :price, :photo)
  end
end
```

**View — ฟอร์มเพิ่มสินค้า (พร้อม direct upload):**

```erb
<%# app/views/products/new.html.erb %>
<h1>เพิ่มสินค้าใหม่</h1>

<%= form_with model: @product, url: products_path do |f| %>
  <% if @product.errors.any? %>
    <ul>
      <% @product.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  <% end %>

  <%= f.label :name, "ชื่อสินค้า" %>
  <%= f.text_field :name %>

  <%= f.label :price, "ราคา" %>
  <%= f.number_field :price %>

  <%= f.label :photo, "รูปสินค้า" %>
  <%= f.file_field :photo, direct_upload: true, accept: "image/png,image/jpeg,image/webp" %>

  <%= f.submit "บันทึก" %>
<% end %>
```

**View — หน้า index พร้อม thumbnail:**

```erb
<%# app/views/products/index.html.erb %>
<h1>สินค้าทั้งหมด</h1>
<ul>
  <% @products.each do |product| %>
    <li>
      <% if product.photo.attached? %>
        <%= image_tag product.photo.variant(resize_to_limit: [150, 150]) %>
      <% else %>
        <span>ไม่มีรูป</span>
      <% end %>
      <%= product.name %> — <%= number_to_currency(product.price, unit: "฿") %>
    </li>
  <% end %>
</ul>
```

### ยืนยันผลด้วย test จริง

สร้างสินค้าพร้อมแนบไฟล์ PNG ทดสอบ แล้วเปิดหน้า index จริง ตรวจสอบ HTML ที่ render ออกมา:

```ruby
test "index renders thumbnail variant for product photo" do
  product = Product.new(name: "เสื้อยืดสีขาว")
  product.photo.attach(
    io: File.open(Rails.root.join("test/fixtures/files/test_avatar.png")),
    filename: "shirt.png", content_type: "image/png"
  )
  product.save!

  get products_url
  assert_response :success
  assert_match(/representations/, response.body)
end
```

ผลลัพธ์จริง (ทั้ง 4 test ในแอปทดสอบผ่านหมด): `4 runs, 16 assertions, 0 failures, 0 errors` และ HTML
ที่ได้จริงมี `<img>` tag ชี้ไปที่ representation URL ของ variant ที่ resize เหลือ 150×150 ถูกต้อง:

```html
<img src="http://www.example.com/rails/active_storage/representations/redirect/...&variation.../shirt.png" />
```

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **แกลเลอรีรูปหลายรูป:** เพิ่ม `has_many_attached :gallery_images` ให้ `Product` แล้วทำหน้า
   `show` แสดงรูปทั้งหมดในแกลเลอรีเป็น thumbnail แถวเดียว (ใบ้: `product.gallery_images.each do
   |image| ... end` วน loop เหมือน association ปกติ) พร้อม validate ว่าห้ามแนบเกิน 5 รูปต่อสินค้า
2. **ตรวจสอบมิติภาพ:** เพิ่ม validation ใหม่ที่ปฏิเสธรูปที่มีความกว้างหรือความสูงต่ำกว่า 300 พิกเซล
   (ใบ้: ใช้ `photo.metadata[:width]`/`[:height]` — แต่ระวังว่าค่านี้อาจยังเป็น `nil` ถ้าขั้นตอน
   analyze ยังไม่เสร็จตอนที่ validate ทำงาน ลองหาวิธีบังคับให้ analyze เสร็จก่อนด้วย
   `photo.analyze` แล้วดูว่าพฤติกรรมเปลี่ยนไปอย่างไร)
3. **รองรับไฟล์ PDF เป็นใบเสร็จแนบ:** เพิ่ม `has_one_attached :invoice_pdf` ให้โมเดลอื่น (เช่น
   `Order` ถ้ามีจาก Part ก่อนๆ) แล้วแสดง thumbnail หน้าแรกของ PDF ด้วย `.preview` ในหน้า index
   พร้อมจัดการกรณีเครื่อง production ไม่มี `poppler-utils`/`mupdf` ติดตั้งไว้ (ลอง rescue
   `ActiveStorage::UnpreviewableError` แล้ว fallback ไปแสดง link ดาวน์โหลดธรรมดาแทน)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Active Storage เป็นส่วนหนึ่งของ Rails เอง** ไม่ต้องพึ่ง gem ภายนอกอย่าง
  Paperclip/CarrierWave อีกต่อไป และรู้จักสถาปัตยกรรม 3 ตารางกลาง (`active_storage_blobs`,
  `active_storage_attachments`, `active_storage_variant_records`) ที่ใช้ร่วมกันได้ทุกโมเดลในแอป
  โดยไม่เพิ่มคอลัมน์ในตารางโมเดลเลยแม้แต่ตัวเดียว
- ติดตั้ง Active Storage ด้วย `rails active_storage:install` + `db:migrate` และเข้าใจว่า
  `config/storage.yml` แค่ประกาศ service ส่วน `config.active_storage.service` ต่างหากที่บอกว่า
  environment ไหนใช้ service ไหน
- ใช้ `has_one_attached`/`has_many_attached` ประกาศความสามารถแนบไฟล์บนโมเดล และแนบไฟล์ได้ทั้งผ่าน
  โค้ด (`attach(io:, filename:, content_type:)`) และผ่านฟอร์มเว็บปกติ (`permit :photo`)
- รู้ว่าไฟล์บน disk service เก็บอยู่ที่ path `storage/xx/yy/<key>` โดยใช้ key แบบสุ่ม ไม่ใช่ชื่อไฟล์
  เดิม และรู้ว่า blob เก็บ metadata สำคัญอะไรบ้าง (content_type ที่ผ่านการยืนยันด้วย `marcel`,
  width/height ของรูปที่ analyze อัตโนมัติ)
- แสดงผลไฟล์ที่แนบด้วย `url_for`/`rails_blob_path`/`image_tag` และเข้าใจว่า URL ที่ได้เป็น signed
  URL ที่ปลอดภัย ไม่ใช่ path ตรงไปยังไฟล์บนดิสก์
- สร้าง **image variant** (thumbnail) ด้วย `variant(resize_to_limit: ...)` ผ่าน gem
  `image_processing` ที่ต้องพึ่งโปรแกรมประมวลผลภาพจริงในเครื่อง (libvips เป็นค่าเริ่มต้นของ Rails
  8, เร็วกว่า ImageMagick) และเข้าใจว่า variant ที่ generate แล้วถูกเก็บเป็น blob+attachment ใหม่
  อีกชุดหนึ่งเพื่อไม่ต้องประมวลผลซ้ำ
- แยกความต่างระหว่าง `variant` (ใช้กับไฟล์ที่เป็นรูปภาพอยู่แล้ว) กับ `preview` (สร้างภาพตัวแทนของ
  PDF/วิดีโอที่ไม่ใช่รูปภาพโดยตรง ต้องพึ่ง poppler/mupdf/ffmpeg)
- เขียน **custom validation** สำหรับตรวจสอบ attachment เอง (presence, content_type, byte_size)
  เพราะ Active Storage ไม่มี validator สำเร็จรูปให้ใช้เหมือน column ปกติ
- ใช้ `purge` (ลบทันทีแบบ synchronous) และ `purge_later` (ตัดความสัมพันธ์ทันที แต่ลบไฟล์จริงแบบ
  background ผ่าน `ActiveStorage::PurgeJob`) อย่างถูกจังหวะ
- เข้าใจกลไกและประโยชน์ของ **Direct Upload** — เบราว์เซอร์คุยตรงกับ storage ผ่าน
  `/rails/active_storage/direct_uploads` โดยไม่ต้องให้ไฟล์จริงผ่าน Rails server เลย ช่วยเรื่อง
  progress bar และไม่จองพื้นที่ worker process ไว้นานเกินจำเป็น
- สร้างระบบอัปโหลดรูปสินค้าที่สมบูรณ์ครบวงจร ตั้งแต่ validate ไปจนถึงแสดง thumbnail จริง

**ต่อไป (Part 067):** เราจะย้ายจาก local disk service ไปใช้ **cloud storage** จริงแบบ
S3-compatible (Amazon S3 หรือทางเลือกที่ถูกกว่าอย่าง Cloudflare R2/MinIO) ตั้งค่าให้ production ใช้
cloud storage โดยที่ development ยังใช้ disk เหมือนเดิม เจาะลึกเรื่อง credentials, CORS สำหรับ
direct upload ข้าม domain, และเทคนิค image processing ขั้นสูงกว่านี้ (crop, format conversion,
quality control) เพื่อเตรียมพร้อมสำหรับระบบอัปโหลดระดับ production จริง
