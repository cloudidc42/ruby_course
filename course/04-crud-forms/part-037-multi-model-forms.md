# Part 037: Multi-model forms — nested attributes ขั้นสูงกับหลายความสัมพันธ์พร้อมกัน, fields_for

> **Step ครอบคลุมใน Part นี้:** Step 361–370
> **ระดับ:** สูง (ต้องผ่าน **Part 031** มาก่อน — Part นี้สร้างต่อจากพื้นฐาน `accepts_nested_attributes_for`/`fields_for` ที่วางไว้ที่นั่นโดยตรง ไม่ทบทวนพื้นฐานซ้ำ)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6 + Rails 8.1.4 — ทุก HTML,
> ทุกผลลัพธ์ `curl`, ทุกผลลัพธ์ console ที่ปรากฏในเอกสารนี้คือของจริงที่ได้จากการรัน
> `bin/rails server` แล้วยิงคำขอเข้าไปจริง ไม่ใช่โค้ดที่เขียนคาดเดาไว้ล่วงหน้า)

**Part 031** สอนพื้นฐานของ nested attributes ไว้ครบแล้วด้วยความสัมพันธ์**เดียว** (`Post` ↔
`Comment`): `accepts_nested_attributes_for`, `fields_for`, `allow_destroy`, `_destroy`,
`reject_if: :all_blank`, และการ permit nested params ด้วย `params.expect` — ท้าย Part นั้นเขียนไว้
ชัดเจนว่า **Part 037 จะกลับมาขยายเป็นฟอร์มที่จัดการหลายความสัมพันธ์พร้อมกัน**และโครงสร้างข้อมูลที่
ซับซ้อนกว่าเดิม นี่คือ Part นั้น — เนื้อหาต่อไปนี้**ไม่ทวนพื้นฐานที่ Part 031 สอนไปแล้วซ้ำอีก** แต่จะ
สมมติว่าคุณคล่อง `fields_for`/`accepts_nested_attributes_for`/`_destroy`/`allow_destroy` แล้ว
และพาไปไกลกว่านั้น

โจทย์ที่ใช้ตลอด Part นี้คือระบบ **สั่งซื้อสินค้า (Order)** ที่จำลองสถานการณ์งานจริงได้ตรงกว่า
ฟอร์ม comment เดิมมาก:

- `Order` **`has_many :line_items`** — คำสั่งซื้อหนึ่งใบมีได้หลายรายการสินค้า
- `LineItem` **`belongs_to :product`** — แต่ละรายการ**อ้างอิง**สินค้าที่มีอยู่แล้วในระบบ (ฟอร์มนี้
  **ไม่ได้สร้าง `Product` ใหม่** เพียงแค่เลือกว่าจะซื้อสินค้าตัวไหนกี่ชิ้น — นี่คือความต่างสำคัญจาก
  Part 031 ที่ nested attributes สร้าง record ลูกขึ้นมาใหม่ทั้งหมด)
- `Order` **`has_one :address`** — ที่อยู่จัดส่ง 1 รายการต่อคำสั่งซื้อ 1 ใบ ที่นี่ Address **ถูก
  สร้างขึ้นจริง** ผ่าน nested attributes (ตรงข้ามกับ Product ที่แค่อ้างอิง) และ validation ของมันมี
  เงื่อนไข ทำงานเฉพาะเมื่อลูกค้าเลือกวิธีจัดส่งเป็น "ส่งถึงบ้าน" เท่านั้น

หนึ่งฟอร์มเดียวของ `Order` จึงต้องจัดการพร้อมกันถึง **3 โมเดล 2 ความสัมพันธ์ที่ต่างชนิดกัน**
(`has_many` ที่อ้างอิง record ภายนอก + `has_one` ที่สร้าง record ใหม่จริง) — นี่คือความซับซ้อนที่
Part 031 ยังไม่ได้แตะ และเป็นแก่นของ Part นี้ทั้งหมด

## สารบัญของ Part นี้

- Step 361: ภาพรวมโจทย์ Order/LineItem/Product/Address และเตรียมสนามทดลอง
- Step 362: Controller เตรียม `@order.line_items.build` หลายแถวว่าง + `build_address` สำหรับฟอร์ม
  `new`/`edit`
- Step 363: `fields_for :line_items` (แสดง select อ้างอิง Product) และ `fields_for :address`
  (has_one) พร้อม strong parameters ที่ต่างกันของทั้งสองแบบ
- Step 364: เพิ่มแถว nested form แบบไดนามิกด้วย vanilla JavaScript — `<template>` +
  `insertAdjacentHTML` + เทคนิค `child_index`
- Step 365: ลบแถว nested form แบบไดนามิกด้วย JavaScript — ความต่างระหว่างแถวที่ persisted แล้วกับ
  แถวที่ยังไม่เคยบันทึก
- Step 366: Nested-nested attributes สามระดับ — Order → LineItem → Product (อ้างอิงอย่างเดียว)
  เทียบกับ Order → Address (สร้างจริง) ที่ validate แบบมีเงื่อนไข `required_for_order?`
- Step 367: `reject_if` แบบ custom lambda ที่ซับซ้อนกว่า `:all_blank`
- Step 368: Validate parent จากข้อมูล nested โดยรวม — `Order` ต้องมีอย่างน้อย 1 line item
  (`reject(&:marked_for_destruction?)`)
- Step 369: แสดง validation error ของ nested record แยกเป็นรายแถวใน view + หมายเหตุความปลอดภัย
  `accepts_nested_attributes_for limit:`
- Step 370: แบบฝึกหัด — ฟอร์ม Order/LineItem/Product/Address แบบเต็ม พร้อม dynamic add/remove
  และการทดสอบด้วย `curl` จริงทั้งกรณีสำเร็จและล้มเหลว + แบบฝึกหัดเพิ่มเติม + สรุป

---

## Step 361: ภาพรวมโจทย์ Order/LineItem/Product/Address และเตรียมสนามทดลอง

### ทำไมโจทย์นี้ถึงซับซ้อนกว่า Post/Comment ของ Part 031

ฟอร์ม Post/Comment ใน Part 031 มีมิติเดียวที่ต้องจัดการ: `Post has_many :comments` และ
`Comment` ทุกฟิลด์ถูก**สร้างขึ้นใหม่จริง**ทั้งหมดผ่าน nested attributes ไม่มีการอ้างอิง record ที่มี
อยู่แล้วเลย

โจทย์ Order ของ Part นี้มี 2 มิติที่ต่างกันโดยพื้นฐานซ้อนอยู่ในฟอร์มเดียว:

1. **Order → LineItem → Product** — `LineItem` เป็น record ใหม่ที่สร้างผ่าน nested attributes
   ก็จริง แต่ field สำคัญของมัน (`product_id`) ไม่ได้ให้ผู้ใช้พิมพ์ข้อความอิสระ แต่ต้อง**เลือกจาก
   Product ที่มีอยู่แล้ว**เท่านั้น (ผ่าน `collection_select`) — ฟอร์มนี้ไม่มีทางสร้าง `Product`
   ใหม่ได้เลยแม้แต่ทางเดียว
2. **Order → Address** — `Address` เป็น record ใหม่ที่สร้างผ่าน nested attributes เหมือนกัน แต่
   field ของมัน**ทุกช่องเป็นข้อความอิสระที่ผู้ใช้พิมพ์เอง** ไม่มีการอ้างอิงอะไรจากภายนอกเลย และมี
   validation ที่ **เปิด/ปิดตามเงื่อนไข** (เลือกจัดส่งแบบไหน)

ทั้งสองมิตินี้ต้องอยู่ในฟอร์มเดียวกัน บันทึกพร้อมกันเป็นทรานแซกชันเดียว — นี่คือสิ่งที่ทำให้ต้องเข้าใจ
`fields_for`/`accepts_nested_attributes_for` ลึกกว่าระดับที่ Part 031 พาไปมาก

### เตรียมสนามทดลอง

```bash
mkdir -p ~/ruby-course-workspace/part-037
cd ~/ruby-course-workspace/part-037

rails new shop_orders --minimal
cd shop_orders

bin/rails generate model Product name:string sku:string stock:integer price_cents:integer
bin/rails generate model Order status:string customer_name:string
bin/rails generate model LineItem order:references product:references quantity:integer
bin/rails generate model Address order:references recipient_name:string line1:string \
  city:string postal_code:string delivery_method:string
```

แก้ migration ทั้ง 4 ไฟล์ให้บังคับ `null: false`/ค่า default ที่เหมาะสม (ทบทวนหลักการ "validate
สองชั้น" จาก **Part 026/030/031**):

```ruby
# db/migrate/..._create_products.rb
class CreateProducts < ActiveRecord::Migration[8.1]
  def change
    create_table :products do |t|
      t.string :name, null: false
      t.string :sku, null: false
      t.integer :stock, null: false, default: 0
      t.integer :price_cents, null: false, default: 0

      t.timestamps
    end

    add_index :products, :sku, unique: true
  end
end
```

```ruby
# db/migrate/..._create_orders.rb
class CreateOrders < ActiveRecord::Migration[8.1]
  def change
    create_table :orders do |t|
      t.string :status, null: false, default: "pending"
      t.string :customer_name, null: false

      t.timestamps
    end
  end
end
```

```ruby
# db/migrate/..._create_line_items.rb
class CreateLineItems < ActiveRecord::Migration[8.1]
  def change
    create_table :line_items do |t|
      t.references :order, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :quantity, null: false, default: 1

      t.timestamps
    end
  end
end
```

```ruby
# db/migrate/..._create_addresses.rb
class CreateAddresses < ActiveRecord::Migration[8.1]
  def change
    create_table :addresses do |t|
      # index: { unique: true } บังคับว่า order หนึ่งใบมี address ได้แค่ 1 แถวเท่านั้น
      # (ตรงกับความหมายของ has_one ฝั่ง Model — เป็นการบังคับกฎนี้ซ้ำอีกชั้นที่ระดับฐานข้อมูล)
      t.references :order, null: false, foreign_key: true, index: { unique: true }
      t.string :recipient_name
      t.string :line1
      t.string :city
      t.string :postal_code
      t.string :delivery_method, null: false, default: "pickup"

      t.timestamps
    end
  end
end
```

```bash
bin/rails db:migrate
```

```
== 20260926055351 CreateProducts: migrating ===================================
-- create_table(:products)
-- add_index(:products, :sku, {:unique=>true})
== 20260926055351 CreateProducts: migrated (0.0027s) ==========================

== 20260926055352 CreateOrders: migrating =====================================
-- create_table(:orders)
== 20260926055352 CreateOrders: migrated (0.0014s) ============================

== 20260926055353 CreateLineItems: migrating ==================================
-- create_table(:line_items)
== 20260926055353 CreateLineItems: migrated (0.0036s) =========================

== 20260926055354 CreateAddresses: migrating ==================================
-- create_table(:addresses)
== 20260926055354 CreateAddresses: migrated (0.0027s) =========================
```

### Model — `app/models/product.rb`, `app/models/line_item.rb`

```ruby
class Product < ApplicationRecord
  has_many :line_items

  validates :name, presence: true
  validates :sku, presence: true, uniqueness: true
  validates :stock, numericality: { greater_than_or_equal_to: 0 }
  validates :price_cents, numericality: { greater_than: 0 }
end
```

```ruby
class LineItem < ApplicationRecord
  belongs_to :order
  belongs_to :product

  validates :product_id, presence: true
  validates :quantity, numericality: { only_integer: true, greater_than: 0 }
end
```

`belongs_to :product` ใน Rails 5+ เป็น **required by default** อยู่แล้ว (ทบทวนจาก **Part 027
Step 261**) — ถ้า `product_id` ชี้ไปยัง Product ที่ไม่มีอยู่จริงในฐานข้อมูล `LineItem` จะ validate
ไม่ผ่านทันทีด้วย error message `"Product must exist"` (จะใช้ประโยชน์จากพฤติกรรมนี้ใน Step 369)

### Model — `app/models/address.rb` และ `app/models/order.rb` (โครงร่างเริ่มต้น)

```ruby
class Address < ApplicationRecord
  belongs_to :order

  DELIVERY_METHODS = %w[pickup ship].freeze

  validates :delivery_method, inclusion: { in: DELIVERY_METHODS }
end
```

```ruby
class Order < ApplicationRecord
  STATUSES = %w[pending paid shipped cancelled].freeze

  has_many :line_items, dependent: :destroy
  has_many :products, through: :line_items
  has_one :address, dependent: :destroy

  validates :status, inclusion: { in: STATUSES }
  validates :customer_name, presence: true

  after_initialize { self.status ||= "pending" }
end
```

โครงร่างนี้ยังไม่มี `accepts_nested_attributes_for` เลย — จะเพิ่มทีละส่วนตลอด Step 362–368 พร้อม
อธิบายเหตุผลของแต่ละ option ที่เพิ่มเข้าไป (ต่างจาก Part 031 ที่ใส่มาให้ครบตั้งแต่ต้น Part นี้จะสร้าง
ทีละชั้นให้เห็นว่าทำไมต้องมีแต่ละ option)

### Routes และ Controller โครงร่าง

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "orders#index"
  resources :orders
end
```

`OrdersController` จะค่อยๆ ขยายทีละ Step จนสมบูรณ์ที่ Step 370 — เริ่มจาก `index`/`show` ธรรมดา
ก่อน (โครงสร้างเดียวกับที่เรียนมาตั้งแต่ Part 028/030) แล้วค่อยเติม logic สำหรับ nested form ใน
Step ถัดไป

---

## Step 362: Controller เตรียมแถวว่างสำหรับ `new`/`edit`

### ทำไมต้องเตรียมแถวว่างที่ Controller ไม่ใช่ที่ View

ทบทวนหลักการจาก **Part 023 Step 227 / Part 031 Step 308**: controller มีหน้าที่เตรียมทุกอย่างที่
view ต้องใช้ไว้ล่วงหน้าเสมอ ("fat controller, thin view" ในความหมายที่ว่า controller ตัดสินใจ
เรื่อง data ทั้งหมด view แค่แสดงผล) — จำนวนแถวว่างที่จะแสดงในฟอร์ม `new` เป็นการตัดสินใจ (business
decision) ไม่ใช่เรื่องการแสดงผล จึงควรอยู่ที่ controller ในรูปแบบค่าคงที่ที่แก้ไขง่าย:

```ruby
class OrdersController < ApplicationController
  before_action :set_products, only: %i[new create edit update]
  before_action :set_order, only: %i[show edit update destroy]

  BLANK_LINE_ITEM_ROWS = 3

  def index
    @orders = Order.order(created_at: :desc)
  end

  def show
  end

  def new
    @order = Order.new
    @order.build_address
    BLANK_LINE_ITEM_ROWS.times { @order.line_items.build }
  end

  def edit
    @order.build_address if @order.address.nil?
    @order.line_items.build
  end

  private

  def set_order
    @order = Order.find(params[:id])
  end

  def set_products
    @products = Product.order(:name)
  end
end
```

**อธิบายทีละจุดที่ใหม่:**

- `@order.line_items.build` (ไม่มี argument) เรียกซ้ำ `BLANK_LINE_ITEM_ROWS` ครั้ง (3 ครั้ง) ใน
  action `new` — เหมือนหลักการจาก Part 031 Step 308 ทุกประการ เพียงแต่คราวนี้เตรียมไว้เป็นค่าคงที่
  ตั้งชื่อไว้ชัดเจนแทนการ hardcode ตัวเลขลอยๆ ในโค้ด (magic number)
- `@order.build_address` — เมธอด `build_<association>` (ไม่มี `s`) นี้เป็นสิ่งที่
  `has_one :address` เพิ่มให้อัตโนมัติ ต่างจาก `has_many` ที่ใช้ `line_items.build` (เรียกผ่าน
  collection) `has_one` เรียกตรงที่ตัว object แม่เลย (`@order.build_address` ไม่ใช่
  `@order.address.build`) เพราะ `has_one` ไม่มี "collection" ให้ build ใส่ — มีได้แค่ 1 record
  เท่านั้น ผลลัพธ์คือ `@order.address` จะเป็น `Address.new(order: @order)` พร้อมให้ `fields_for`
  ไปอ่านค่าเรนเดอร์ฟอร์มว่างต่อได้ทันที (จะเห็น HTML จริงใน Step 363)
- `edit` เรียก `build_address` **เฉพาะกรณีที่ `@order.address` เป็น `nil`** (เผื่อกรณี order เก่าที่
  สร้างไว้ก่อนจะมี Address เลย — ยังไม่เคยกรอกที่อยู่) ถ้ามี address อยู่แล้ว `fields_for` จะอ่าน
  ค่าจาก record เดิมมาเติมในฟอร์มให้อัตโนมัติ (เหมือนพฤติกรรม `form_with(model:)` ที่เรียนมา)
- `edit` เรียก `@order.line_items.build` เพิ่มอีก **1 แถวว่าง** ต่อจาก line item เดิมทั้งหมด —
  หลักการเดียวกับ Part 031 Step 308 (เผื่อไว้ให้เพิ่มสินค้าใหม่ระหว่างแก้ไข)

`create`/`update` (รวม `order_params`) จะเติมเต็มใน Step 363 หลังจากอธิบาย strong parameters
ของทั้งสองความสัมพันธ์ก่อน

---

## Step 363: `fields_for :line_items`/`fields_for :address` และ strong parameters ที่ต่างกัน

### `fields_for :line_items` — ผูกกับ Product ที่มีอยู่แล้วผ่าน `collection_select`

```erb
<%# app/views/orders/_line_item_fields.html.erb %>
<div class="line-item-row">
  <% if f.object.persisted? %>
    <%= f.hidden_field :id %>
  <% end %>

  <div class="field">
    <%= f.label :product_id, "สินค้า" %>
    <%= f.collection_select :product_id, @products, :id, :name,
          include_blank: "— เลือกสินค้า —" %>
  </div>

  <div class="field">
    <%= f.label :quantity, "จำนวน" %>
    <%= f.number_field :quantity, min: 1 %>
  </div>
</div>
```

```erb
<%# ส่วนหนึ่งของ app/views/orders/_form.html.erb %>
<%= f.fields_for :line_items do |lif| %>
  <%= render "line_item_fields", f: lif %>
<% end %>
```

HTML จริงที่ render ออกมา (หน้า `new` ที่มี 3 แถวว่างจาก `BLANK_LINE_ITEM_ROWS`):

```html
<div class="line-item-row">
  <div class="field">
    <label for="order_line_items_attributes_0_product_id">สินค้า</label>
    <select name="order[line_items_attributes][0][product_id]"
      id="order_line_items_attributes_0_product_id">
      <option value="">— เลือกสินค้า —</option>
      <option value="2">Gadget</option>
      <option value="3">Gizmo</option>
      <option value="1">Widget</option>
    </select>
  </div>
  <div class="field">
    <label for="order_line_items_attributes_0_quantity">จำนวน</label>
    <input min="1" type="number" value="1" name="order[line_items_attributes][0][quantity]"
      id="order_line_items_attributes_0_quantity" />
  </div>
</div>
<!-- แถวที่ 1 และ 2 เหมือนกัน เปลี่ยนแค่เลขดัชนี [1], [2] -->
```

**สังเกตจุดสำคัญที่ต่างจาก Part 031:** `f.collection_select :product_id, @products, :id, :name`
— field ที่ nested attributes นี้เขียนกลับไม่ใช่ field ข้อความอิสระเหมือน `commenter`/`body` ของ
Comment เดิม แต่เป็น `product_id` ที่**ค่าต้องมาจาก `@products` ที่มีอยู่แล้วในฐานข้อมูลเท่านั้น** —
`LineItem` ที่ถูกสร้างผ่าน nested attributes ตัวนี้จึงเป็นแค่ "สะพานเชื่อม" ระหว่าง `Order` กับ
`Product` ที่มีอยู่ก่อนแล้ว ไม่ได้สร้างข้อมูลใหม่ทั้งหมดเหมือนกรณี Comment — นี่คือรูปแบบการใช้
nested attributes ที่พบบ่อยมากในระบบจริง (ตะกร้าสินค้า, ใบสั่งซื้อ, ใบเสนอราคา ล้วนมีโครงสร้างแบบนี้)

### `fields_for :address` — has_one ไม่มีเลขดัชนี array

```erb
<%= f.fields_for :address do |af| %>
  <div class="field">
    <%= af.label :delivery_method, "วิธีจัดส่ง" %>
    <%= af.select :delivery_method,
          [["รับสินค้าเอง (pickup)", "pickup"], ["จัดส่งถึงบ้าน (ship)", "ship"]] %>
  </div>

  <div class="field">
    <%= af.label :recipient_name, "ชื่อผู้รับ" %>
    <%= af.text_field :recipient_name %>
  </div>

  <div class="field">
    <%= af.label :line1, "ที่อยู่" %>
    <%= af.text_field :line1 %>
  </div>

  <div class="field">
    <%= af.label :city, "เมือง/จังหวัด" %>
    <%= af.text_field :city %>
  </div>

  <div class="field">
    <%= af.label :postal_code, "รหัสไปรษณีย์" %>
    <%= af.text_field :postal_code %>
  </div>
<% end %>
```

HTML จริงที่ได้:

```html
<div class="field">
  <label for="order_address_attributes_delivery_method">วิธีจัดส่ง</label>
  <select name="order[address_attributes][delivery_method]"
    id="order_address_attributes_delivery_method">
    <option selected="selected" value="pickup">รับสินค้าเอง (pickup)</option>
    <option value="ship">จัดส่งถึงบ้าน (ship)</option>
  </select>
</div>
<div class="field">
  <label for="order_address_attributes_recipient_name">ชื่อผู้รับ</label>
  <input type="text" name="order[address_attributes][recipient_name]"
    id="order_address_attributes_recipient_name" />
</div>
<!-- line1, city, postal_code เหมือนกัน -->
```

**เปรียบเทียบชื่อ field ทั้งสองแบบให้เห็นชัด:**

| ความสัมพันธ์ | ชื่อ field ที่ได้ | มีเลขดัชนีไหม |
|---|---|---|
| `has_many :line_items` | `order[line_items_attributes][0][product_id]` | **มี** (`[0]`, `[1]`, `[2]`, ...) เพราะมีได้หลาย record |
| `has_one :address` | `order[address_attributes][recipient_name]` | **ไม่มี** เพราะมีได้แค่ 1 record เท่านั้น ไม่ต้องแยกแยะว่าเป็นตัวไหน |

ความต่างนี้สำคัญมากตอนเขียน strong parameters — จะเห็นในหัวข้อถัดไปว่า syntax ของทั้งสองแบบต่างกัน
ตามรูปร่างข้อมูลนี้พอดี

### Strong parameters — double-bracket สำหรับ `has_many`, ไม่ต้องมีสำหรับ `has_one`

ทบทวนกับดักจาก **Part 031 Step 306**: `params.expect` ต้องใช้ **double-bracket**
(`{ key: [[...]] }`) เมื่อ nested key เป็น **array ของ record** (กรณี `has_many`) — แต่คำถามที่
Part 031 ยังไม่ได้ตอบคือ "แล้ว `has_one` ล่ะ ต้องมี double-bracket ไหม" คำตอบคือ **ไม่ต้อง**
เพราะ `address_attributes` ไม่ใช่ array ของ record หลายตัว มันเป็นแค่ **Hash เดียว** (record เดียว
ไม่มีดัชนี) จึงเขียนแบบ single-level permit list ธรรมดาเหมือน field อื่นๆ ทั่วไป:

```ruby
def order_params
  params.expect(
    order: [
      :customer_name, :status,
      { line_items_attributes: [[:id, :product_id, :quantity, :_destroy]] },  # has_many → double-bracket
      { address_attributes: [:id, :recipient_name, :line1, :city, :postal_code,
                              :delivery_method, :_destroy] }                   # has_one → ไม่ต้องมี
    ]
  )
end
```

ทดสอบจริงเพื่อยืนยันว่าทั้งสองแบบทำงานถูกต้องพร้อมกันได้ในคำขอเดียว — เติม `create` ให้
`OrdersController` ก่อน:

```ruby
def create
  @order = Order.new(order_params)

  if @order.save
    redirect_to @order, notice: "สร้างคำสั่งซื้อสำเร็จ"
  else
    @order.build_address if @order.address.nil?
    @order.line_items.build if @order.line_items.empty?
    render :new, status: :unprocessable_entity
  end
end
```

(ยังไม่เพิ่ม `accepts_nested_attributes_for` ที่ Model เลยตอนนี้ — ทดสอบ request นี้จริงจะได้
`NoMethodError: undefined method 'line_items_attributes=' for an instance of Order` เพราะเมธอดนี้
เกิดจาก `accepts_nested_attributes_for` เท่านั้น จะเพิ่มให้ครบใน Step ถัดไปพร้อมอธิบาย option
แต่ละตัว)

---

## Step 364: เพิ่มแถวไดนามิกด้วย vanilla JavaScript — `<template>` + `child_index`

### ปัญหาที่ต้องแก้: ฟอร์มที่เตรียมแถวตายตัวไว้ล่วงหน้าไม่พอในงานจริง

`BLANK_LINE_ITEM_ROWS = 3` ที่เตรียมไว้ใน Step 362 ใช้ได้กับกรณีง่ายๆ แต่ในระบบสั่งซื้อจริง ลูกค้า
อาจต้องการซื้อสินค้ามากกว่า 3 รายการ หรือน้อยกว่านั้น — ต้องมีปุ่ม **"เพิ่มรายการสินค้า"** ที่เพิ่ม
แถวใหม่ได้แบบไม่จำกัด (ในทางปฏิบัติจะจำกัดด้วย `limit:` ที่ Step 369) โดยไม่ต้อง reload หน้า

> **หมายเหตุเรื่องเทคโนโลยีที่ใช้:** หลักสูตรนี้สอน Stimulus.js (ตัวจัดการ JavaScript แบบเป็น
> ระบบของ Hotwire) อย่างละเอียดใน **Phase 7 (Part 053)** ซึ่งยังไม่ถึงตอนนี้ — Part นี้จึงใช้
> **vanilla JavaScript ล้วนๆ** (ไม่มี library ใดๆ) แบบสั้นที่สุดเท่าที่จะทำได้ เพื่อโฟกัสที่หลักการ
> ของ nested attributes/Rails ล้วนๆ ไม่ปนกับการเรียนรู้ framework JS ใหม่ — เมื่อถึง Phase 7 จะ
> กลับมาเขียนฟีเจอร์เดียวกันนี้ใหม่ด้วย Stimulus controller ที่ดูแลรักษาง่ายกว่ามาก

### เทคนิคหลัก: `<template>` element ของ HTML + `child_index`

**ปัญหาแรก:** `fields_for :line_items` ที่เห็นใน Step 363 render field มาจาก record ที่มีอยู่
จริงใน `@order.line_items` เท่านั้น (ไม่ว่าจะ persisted หรือ `.build` ไว้ก่อน) — ถ้าต้องการ "แม่แบบ"
HTML ของแถวว่างสัก 1 แถวเพื่อเอาไป clone ด้วย JavaScript ภายหลัง ต้องมีวิธีสั่งให้ `fields_for`
render ให้ **โดยไม่เพิ่ม record จริงเข้าไปใน collection ของ `@order`**

คำตอบคือส่ง object เปล่าเป็นอาร์กิวเมนต์ที่ 2 ของ `fields_for` ตรงๆ พร้อม option **`child_index:`**
เพื่อกำหนดว่าตัวเลขดัชนีของแถวนี้ (ที่ปกติ Rails จะนับให้อัตโนมัติจากตำแหน่งใน array) ให้เป็นค่า
พิเศษที่เรากำหนดเอง:

```erb
<template id="line-item-template">
  <%= f.fields_for :line_items, LineItem.new, child_index: "NEW_RECORD" do |lif| %>
    <%= render "line_item_fields", f: lif %>
  <% end %>
</template>
```

HTML จริงที่ได้ (สังเกต `NEW_RECORD` ที่แทรกอยู่ในทั้ง `name` และ `id` ทุกตำแหน่งที่ปกติจะเป็นเลข):

```html
<template id="line-item-template">
  <div class="line-item-row" data-line-item-row data-persisted="false">
    <input value="0" type="hidden" name="order[line_items_attributes][NEW_RECORD][_destroy]"
      id="order_line_items_attributes_NEW_RECORD__destroy" />
    <div class="field">
      <label for="order_line_items_attributes_NEW_RECORD_product_id">สินค้า</label>
      <select name="order[line_items_attributes][NEW_RECORD][product_id]"
        id="order_line_items_attributes_NEW_RECORD_product_id">
        <option value="">— เลือกสินค้า —</option>
        <option value="2">Gadget</option>
        <option value="3">Gizmo</option>
        <option value="1">Widget</option>
      </select>
    </div>
    <div class="field">
      <label for="order_line_items_attributes_NEW_RECORD_quantity">จำนวน</label>
      <input min="1" type="number" value="1"
        name="order[line_items_attributes][NEW_RECORD][quantity]"
        id="order_line_items_attributes_NEW_RECORD_quantity" />
    </div>
    <button type="button" class="remove-line-item" onclick="removeLineItemRow(this)">
      ลบรายการนี้
    </button>
  </div>
</template>
```

**ทำไมต้องเป็น `<template>` ไม่ใช่ `<div style="display: none">`:** `<template>` เป็น HTML element
มาตรฐาน (ไม่ใช่ของ Rails) ที่เบราว์เซอร์**ไม่ render เนื้อหาข้างในเป็น DOM จริงเลย** (ไม่นับเป็นส่วน
หนึ่งของหน้าเว็บที่มองเห็นหรือถูก query ได้จนกว่าจะถูก clone ออกไปด้วย JavaScript) ต่างจาก
`display: none` ที่ยัง render เป็น DOM จริงอยู่ (แค่ไม่แสดงผลทางสายตา) — ข้อดีของ `<template>` คือ
`<select>` ข้างในจะไม่ถูกเบราว์เซอร์เริ่ม initialize หรือรวมอยู่ใน form submission โดยไม่ตั้งใจ
เพราะมันไม่ใช่ DOM element ที่ "จริง" จนกว่าจะถูกย้ายออกมา

### JavaScript ที่ใช้ clone แม่แบบ — อธิบายทีละบรรทัด

```html
<script>
  function addLineItemRow() {
    const template = document.getElementById("line-item-template");
    const container = document.getElementById("line-items-container");

    // ใช้ timestamp เป็น unique index แทนการนับเลข 0,1,2 เอง เพื่อไม่ให้ index ชนกับแถวเดิม
    // ที่ Rails render มาจาก server (โดยเฉพาะถ้าผู้ใช้ลบแถวกลางๆ ไปก่อนแล้วค่อยเพิ่มแถวใหม่)
    const uniqueIndex = new Date().getTime();
    const html = template.innerHTML.replace(/NEW_RECORD/g, uniqueIndex);

    container.insertAdjacentHTML("beforeend", html);
  }

  document.addEventListener("DOMContentLoaded", () => {
    document.getElementById("add-line-item").addEventListener("click", addLineItemRow);
  });
</script>
```

**อธิบายทีละบรรทัด:**

1. `document.getElementById("line-item-template")` — หา `<template>` ที่เตรียมไว้จาก HTML ด้านบน
2. `document.getElementById("line-items-container")` — หา `<div id="line-items-container">` ที่
   ครอบแถว line item ทั้งหมดอยู่ (ต้องห่อ `f.fields_for :line_items` เดิมด้วย `<div>` นี้เพื่อให้มี
   ที่ให้ `insertAdjacentHTML` เพิ่มแถวใหม่ต่อท้ายได้)
3. `template.innerHTML` — ดึง HTML string ที่อยู่ข้างใน `<template>` ออกมาเป็นข้อความธรรมดา (นี่
   คือคุณสมบัติพิเศษของ `<template>` — `.innerHTML` อ่านเนื้อหาข้างในได้ตรงๆ ทั้งที่มันไม่ใช่ DOM
   ที่ active)
4. `.replace(/NEW_RECORD/g, uniqueIndex)` — แทนที่ทุกจุดที่มีคำว่า `NEW_RECORD` (ทั้งใน `name` และ
   `id` attribute) ด้วยตัวเลขที่ไม่ซ้ำใคร — **นี่คือหัวใจของเทคนิค `child_index`**: เพราะเราบอก
   Rails ให้ฝัง string คงที่ `"NEW_RECORD"` ไว้แทนตัวเลขดัชนีปกติตั้งแต่ตอน render ฝั่ง server
   ทำให้ JavaScript หาและแทนที่มันได้ง่ายด้วย regex ตัวเดียว ไม่ต้องแยกวิเคราะห์โครงสร้าง HTML เอง
5. `new Date().getTime()` — ทำไมใช้ **timestamp** แทนการนับ 0, 1, 2 ไล่ไปเรื่อยๆ เอง: ถ้านับเอง
   แล้วผู้ใช้เผลอกดเพิ่มแถว 2 ครั้งเร็วๆ (หรือมีการลบแถวกลางทางแล้วเพิ่มใหม่) อาจได้เลขดัชนีที่ชน
   กับแถวที่มีอยู่แล้วในหน้า (ทำให้ field ของสองแถวปนกันตอน submit) — `Date.now()` ให้ค่าที่ไม่ซ้ำ
   กันแทบจะแน่นอน (หน่วยมิลลิวินาที) โดยไม่ต้องเก็บ state ของ "ดัชนีถัดไปคือเท่าไหร่" ไว้ที่ไหนเลย
   — **ข้อควรรู้:** ค่าตัวเลขดัชนีของ nested attributes **ไม่จำเป็นต้องเรียงต่อกันหรือเริ่มจาก 0**
   Rails สนใจแค่ว่าแต่ละดัชนีไม่ซ้ำกันเท่านั้น ไม่ได้ใช้ค่าตัวเลขนั้นไปทำอะไรอย่างอื่น (ไม่ใช่ index
   ของ array ในความหมายจริงจัง เป็นแค่ "ป้ายชื่อแยกกลุ่ม" ให้ Rails รู้ว่า field ไหนอยู่กลุ่มเดียวกัน)
6. `container.insertAdjacentHTML("beforeend", html)` — แทรก HTML string ที่ได้เข้าไปเป็นลูกตัว
   สุดท้ายของ container ทันที `insertAdjacentHTML` เร็วกว่าและปลอดภัยกว่าการตั้ง
   `container.innerHTML += html` ตรงๆ (การใช้ `+=` จะทำให้เบราว์เซอร์ parse HTML ทั้งก้อนใหม่หมด
   รวมถึงแถวเดิมที่มีอยู่แล้ว ซึ่งทำให้ event listener ที่ผูกกับ element เดิมหลุดหายไปด้วย —
   `insertAdjacentHTML` แทรกเฉพาะส่วนใหม่โดยไม่แตะ DOM เดิมเลย)

ทดสอบผลลัพธ์จริงจากเบราว์เซอร์ (จำลองด้วยมือ): กดปุ่ม "+ เพิ่มรายการสินค้า" 1 ครั้ง จะได้แถวใหม่ที่
`name="order[line_items_attributes][1758864..NNN][product_id]"` (เลขยาวๆ จาก timestamp) ต่อท้าย
แถวเดิม 3 แถว — เมื่อ submit ฟอร์ม Rails จะเห็น `line_items_attributes` เป็น Hash ที่มี 4 key (3
ดัชนีเดิมจาก server + 1 ดัชนีใหม่จาก JS) และสร้าง `LineItem` ให้ครบทั้ง 4 แถวโดยไม่สนใจเลยว่าดัชนี
จะไม่เรียงต่อกัน (`0, 1, 2, 1758864...`) — พิสูจน์จริงด้วย `curl` (จำลองสิ่งที่ JS จะ submit) ใน
Step 365

---

## Step 365: ลบแถวไดนามิกด้วย JavaScript — แถว persisted กับแถวใหม่ต้องจัดการต่างกัน

### ทำไมลบ DOM element ทิ้งเฉยๆ ใช้ได้แค่บางกรณี

ปุ่ม "ลบรายการนี้" ที่เห็นใน HTML ของ Step 363–364 ต้องแยก 2 กรณีให้ถูกต้อง ไม่เช่นนั้นจะลบข้อมูล
ผิดจากที่ตั้งใจ:

1. **แถวที่ยังไม่เคยบันทึกลงฐานข้อมูล** (แถวว่างที่เตรียมไว้ตอนสร้าง หรือแถวที่เพิ่งกดเพิ่มด้วย JS)
   — ไม่มี record จริงอยู่เบื้องหลังเลย **ลบ DOM element ทิ้งตรงๆ ได้เลย** ไม่ต้อง submit อะไร
   เกี่ยวกับแถวนี้ทั้งสิ้น ฟอร์มจะไม่มี field ของแถวนี้ส่งไปเลย เหมือนไม่เคยมีแถวนี้อยู่
2. **แถวที่ persisted แล้ว** (มี `LineItem` จริงอยู่ในฐานข้อมูล มี `id`) — **ห้ามลบ DOM element
   ทิ้งเด็ดขาด** เพราะถ้าลบทิ้ง ฟอร์มจะไม่ส่ง field ของแถวนั้นไปเลย (รวมถึง hidden field `id`)
   ทำให้ server **ไม่รู้เลยว่ามี record นี้อยู่** ไม่ต้องพูดถึงการสั่งลบมันด้วยซ้ำ — record นั้นจะ
   ยังอยู่ในฐานข้อมูลเหมือนเดิมทุกประการ ทั้งที่ผู้ใช้เห็นว่าแถวหายไปจากหน้าจอแล้ว (บั๊กที่ดูเหมือน
   ทำงานถูกแต่จริงๆ ไม่ได้ลบอะไรเลย) วิธีที่ถูกต้องคือต้อง**เก็บแถวนั้นไว้ใน DOM** (ซ่อนด้วย CSS)
   พร้อมตั้งค่า hidden field `_destroy` เป็น `"1"` เพื่อให้ยัง submit ค่านี้ไปให้
   `accepts_nested_attributes_for` เห็นและสั่ง `.destroy` ให้จริงที่ฝั่ง server (ทบทวนกลไก
   `_destroy` เต็มรูปแบบจาก **Part 031 Step 309**)

### แก้ partial ให้ระบุว่าแถวไหน persisted แล้วบ้าง

```erb
<%# app/views/orders/_line_item_fields.html.erb (เวอร์ชันเต็ม) %>
<div class="line-item-row" data-line-item-row data-persisted="<%= f.object.persisted? %>">
  <% if f.object.persisted? %>
    <%= f.hidden_field :id %>
  <% end %>
  <%= f.hidden_field :_destroy, value: "0" %>

  <div class="field">
    <%= f.label :product_id, "สินค้า" %>
    <%= f.collection_select :product_id, @products, :id, :name,
          include_blank: "— เลือกสินค้า —" %>
  </div>

  <div class="field">
    <%= f.label :quantity, "จำนวน" %>
    <%= f.number_field :quantity, min: 1 %>
  </div>

  <button type="button" class="remove-line-item" onclick="removeLineItemRow(this)">
    ลบรายการนี้
  </button>
</div>
```

**สิ่งที่เพิ่มจาก Step 363:**

- `data-persisted="<%= f.object.persisted? %>"` — ฝังสถานะ persisted ไว้ใน `data-*` attribute
  ของ HTML ตรงๆ ให้ JavaScript อ่านได้โดยไม่ต้อง query อย่างอื่นเพิ่ม (ค่าที่ได้จะเป็น string
  `"true"`/`"false"` เพราะ HTML attribute เป็น string เสมอ)
- `f.hidden_field :_destroy, value: "0"` — เขียนเอง**ตรงๆ ไม่ใช่ผ่าน `f.check_box :_destroy`**
  แบบ Part 031 เพราะที่นี่ไม่ต้องการ checkbox ที่มองเห็น (การลบทำผ่านปุ่มเดียวที่ควบคุมด้วย
  JavaScript) แต่ยังต้องมี hidden field นี้อยู่เสมอ (ค่าเริ่มต้น `"0"`) เพื่อให้ JavaScript มีที่ให้
  ตั้งค่าเป็น `"1"` ทีหลังได้เมื่อกดปุ่มลบ

### JavaScript ที่ใช้ลบแถว

```html
<script>
  function removeLineItemRow(button) {
    const row = button.closest("[data-line-item-row]");

    if (row.dataset.persisted === "true") {
      // แถวนี้มีอยู่แล้วในฐานข้อมูลจริง — ต้องส่ง _destroy=1 ไปให้
      // accepts_nested_attributes_for ลบให้ที่ฝั่ง server ห้ามลบ DOM element ทิ้งเฉยๆ
      row.querySelector("input[name*='[_destroy]']").value = "1";
      row.style.display = "none";
    } else {
      // แถวนี้ยังไม่เคยถูกบันทึกลงฐานข้อมูล — ลบออกจาก DOM ตรงๆ ได้เลย
      row.remove();
    }
  }
</script>
```

**อธิบายทีละบรรทัด:**

- `button.closest("[data-line-item-row]")` — `closest()` ไต่ขึ้นไปจาก element ปุ่มที่ถูกคลิก หา
  ancestor ที่ใกล้ที่สุดที่มี attribute `data-line-item-row` (คือ `<div class="line-item-row">`
  ที่ครอบทั้งแถว) — ใช้ `closest` แทนการจำ id ของแต่ละแถวเอง เพราะแถวที่เพิ่มด้วย JS ไม่มี id ที่
  แน่นอนตายตัว (ไม่ได้ตั้ง `id` ให้ตัว `<div>` เอง มีแค่ attribute เฉยๆ)
- `row.dataset.persisted` — อ่านค่าจาก `data-persisted="..."` ที่ฝังไว้ (Web API `.dataset` แปลง
  `data-persisted` เป็น property `persisted` ให้อัตโนมัติ ตามกฎการแปลง kebab-case → camelCase)
- `row.querySelector("input[name*='[_destroy]']")` — หา `<input>` ที่ `name` attribute มีข้อความ
  `[_destroy]` อยู่ตรงไหนก็ได้ (CSS attribute selector `*=` คือ "contains") ไม่ต้องรู้เลขดัชนีที่
  แน่นอนของแถวนั้นเลย เพราะ scope การค้นหาถูกจำกัดอยู่แค่ภายใน `row` (แถวนั้นแถวเดียว) แล้วด้วย
  `querySelector` (ตัวเดียว ไม่ใช่ `querySelectorAll`) จึงได้ element แรกที่ตรงเงื่อนไขซึ่งมีแค่ตัว
  เดียวอยู่แล้วในแถวนั้น
- `row.style.display = "none"` — ซ่อนแถวด้วย CSS (ไม่ลบออกจาก DOM) เพื่อให้ field ทั้งหมดของแถวนี้
  (รวม `id` และ `_destroy` ที่เพิ่งตั้งเป็น `"1"`) **ยังคง submit ไปพร้อมฟอร์มตามปกติ**

### ทดสอบจริง — จำลองสิ่งที่ JS จะ submit ด้วย `curl`

เพิ่ม `accepts_nested_attributes_for` ที่ฝั่ง Model ให้ครบก่อน (จะอธิบาย option แต่ละตัวเต็มๆ ใน
Step 366–369 ตอนนี้ใส่แบบพื้นฐานที่สุดไปก่อนเพื่อให้ทดสอบได้):

```ruby
# app/models/order.rb (เพิ่มจากโครงร่างเดิม)
accepts_nested_attributes_for :line_items, allow_destroy: true
accepts_nested_attributes_for :address, allow_destroy: true
```

สร้างคำสั่งซื้อที่มี 2 line items ก่อน (id=1 อ้างอิง Widget จำนวน 2, id=2 อ้างอิง Gadget จำนวน 1)
แล้วจำลอง **การแก้ไข quantity ของแถวแรก + ลบแถวที่สอง + เพิ่มแถวใหม่ด้วย "JS"** (ใช้เลขดัชนี
`99999` แทนค่า timestamp จริงเพื่อให้อ่านง่าย — หลักการเหมือนกันทุกประการ) ในคำขอ `PATCH` เดียว:

```bash
curl -s -c cookies.txt -b cookies.txt -X PATCH http://localhost:3000/orders/1 \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=สมชาย ทดสอบ" \
  --data-urlencode "order[status]=paid" \
  --data-urlencode "order[line_items_attributes][0][id]=1" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=5" \
  --data-urlencode "order[line_items_attributes][0][_destroy]=0" \
  --data-urlencode "order[line_items_attributes][1][id]=2" \
  --data-urlencode "order[line_items_attributes][1][product_id]=2" \
  --data-urlencode "order[line_items_attributes][1][quantity]=1" \
  --data-urlencode "order[line_items_attributes][1][_destroy]=1" \
  --data-urlencode "order[line_items_attributes][99999][product_id]=3" \
  --data-urlencode "order[line_items_attributes][99999][quantity]=1" \
  --data-urlencode "order[address_attributes][delivery_method]=pickup" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 302
```

ตรวจสอบผลจริง:

```irb
irb> o = Order.find(1)
irb> o.line_items.pluck(:id, :product_id, :quantity)
=> [[1, 1, 5], [3, 3, 1]]
irb> o.status
=> "paid"
```

ผลลัพธ์ยืนยันครบทั้ง 3 การกระทำจากคำขอเดียว: **id=1 quantity เปลี่ยนเป็น 5** (แก้ไข), **id=2
หายไป** (ลบผ่าน `_destroy=1`), และมี **id=3 ตัวใหม่** (สร้างจาก key `99999` ที่ไม่มี `id` มาด้วย —
พิสูจน์ว่าเลขดัชนีไม่จำเป็นต้องเรียงต่อกันตามที่อธิบายไว้ท้าย Step 364) — `99999` ในตัวอย่างนี้คือ
ค่าที่ `Date.now()` จะสร้างให้จริงในเบราว์เซอร์จริง (แค่ยาวกว่านี้มาก) ไม่มีอะไรพิเศษกว่ากันเลย

---

## Step 366: Nested-nested attributes สามระดับ — อ้างอิงอย่างเดียว vs สร้างจริง พร้อม validate แบบมีเงื่อนไข

### โครงสร้างสามระดับของโจทย์นี้

```
Order (ระดับ 1)
 ├─ has_many :line_items (ระดับ 2) — สร้างผ่าน nested attributes จริง
 │    └─ belongs_to :product (ระดับ 3) — แค่ "อ้างอิง" ผ่าน product_id เท่านั้น
 │                                        ไม่มี nested attributes สำหรับ Product เลย
 └─ has_one :address (ระดับ 2) — สร้างผ่าน nested attributes จริงเช่นกัน
      (ไม่มีระดับ 3 ต่อจาก Address ในโจทย์นี้)
```

จุดที่ต้องเข้าใจให้ชัด: **`accepts_nested_attributes_for` ใช้แค่กับ `line_items` และ `address`
เท่านั้น ไม่มีการประกาศให้ `Product` เลยแม้แต่นิดเดียว** — เพราะ `Product` ไม่ใช่ "ลูก" ที่ถูกสร้าง/
แก้ไข/ลบผ่านฟอร์มของ Order เลย มันเป็นแค่ตัวเลือกที่มีอยู่แล้วให้ `LineItem` ชี้ไปหา (คล้ายกับที่
`collection_select :category_id` ใน Part 031 Step 303 ให้ Post เลือก Category ที่มีอยู่แล้ว
เพียงแต่คราวนี้การอ้างอิงนั้นเกิดขึ้น**ภายใน nested record** ไม่ใช่ที่ตัว parent object ตรงๆ)

**กฎการตัดสินใจ:** ใช้ `accepts_nested_attributes_for` เฉพาะกับความสัมพันธ์ที่ฟอร์มนี้มีสิทธิ์
สร้าง/แก้ไข/ลบ record นั้นได้จริงๆ เท่านั้น ถ้าฟอร์มแค่ต้องการให้เลือกจาก record ที่มีอยู่แล้ว (ไม่
สร้าง ไม่แก้ ไม่ลบ) ให้ใช้ field ธรรมดาอย่าง `collection_select`/`select` ชี้ไปที่ foreign key
column ตรงๆ พอ ไม่ต้องพึ่ง nested attributes เลย

### `Address` validate แบบมีเงื่อนไข — `required_for_order?`

ที่อยู่จัดส่งไม่จำเป็นต้องกรอกครบทุกช่องเสมอไป — ถ้าลูกค้าเลือก **"รับสินค้าเอง" (`pickup`)** ไม่
ต้องกรอกที่อยู่เลยก็ valid ได้ แต่ถ้าเลือก **"จัดส่งถึงบ้าน" (`ship`)** ต้องกรอกที่อยู่ให้ครบทุกช่อง
— นี่คือ **conditional validation** (ทบทวนแนวคิด `if:`/`unless:` เบื้องต้นจาก **Part 026** เตรียม
ไว้ Part 032 จะเจาะลึกเรื่องนี้อีกครั้ง):

```ruby
class Address < ApplicationRecord
  belongs_to :order

  DELIVERY_METHODS = %w[pickup ship].freeze

  validates :delivery_method, inclusion: { in: DELIVERY_METHODS }

  with_options if: :required_for_order? do
    validates :recipient_name, presence: true
    validates :line1, presence: true
    validates :city, presence: true
    validates :postal_code, presence: true
  end

  # ที่อยู่จำเป็นต้องกรอกครบก็ต่อเมื่อลูกค้าเลือกวิธีจัดส่งเป็น "ship" เท่านั้น
  # ถ้าเลือก "pickup" (มารับเอง) ไม่ต้องกรอกที่อยู่เลยก็ valid ได้
  def required_for_order?
    delivery_method == "ship"
  end
end
```

`with_options if: :required_for_order? do ... end` เป็น syntax ของ Rails ที่ห่อ validation หลาย
ตัวให้ใช้ option เดียวกัน (`if: :required_for_order?`) ร่วมกันโดยไม่ต้องเขียน `if:` ซ้ำทุกบรรทัด —
เทียบเท่ากับเขียน `validates :recipient_name, presence: true, if: :required_for_order?` วนซ้ำ 4
ครั้ง แต่กระชับกว่า

### ทดสอบจริง — `pickup` ไม่ต้องกรอกที่อยู่เลยก็บันทึกสำเร็จ

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=สมชาย ทดสอบ" \
  --data-urlencode "order[status]=pending" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=2" \
  --data-urlencode "order[line_items_attributes][1][product_id]=2" \
  --data-urlencode "order[line_items_attributes][1][quantity]=1" \
  --data-urlencode "order[address_attributes][delivery_method]=pickup" \
  -D - -o resp.html -w "HTTP %{http_code}\n" | grep -Ei "HTTP|location"
```

```
HTTP/1.1 302 Found
location: http://localhost:3000/orders/1
HTTP 302
```

ตรวจสอบผลจริง — Address ถูกสร้างขึ้นจริงแม้จะไม่มีข้อมูลที่อยู่เลยสักช่อง เพราะ
`required_for_order?` คืน `false` ตอนที่ `delivery_method == "pickup"`:

```irb
irb> Order.find(1).address
=> #<Address id: 1, order_id: 1, recipient_name: nil, line1: nil, city: nil,
    postal_code: nil, delivery_method: "pickup", ...>
```

### ทดสอบจริง — `ship` แต่ไม่กรอกที่อยู่ ต้อง validate ไม่ผ่าน

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=ต้องจัดส่ง" \
  --data-urlencode "order[status]=pending" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=1" \
  --data-urlencode "order[address_attributes][delivery_method]=ship" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

HTML error ที่ได้จริง:

```html
<div class="form-errors">
  <h3>4 ข้อผิดพลาดทำให้บันทึกคำสั่งซื้อนี้ไม่ได้:</h3>
  <ul>
    <li>Address recipient name can&#39;t be blank</li>
    <li>Address line1 can&#39;t be blank</li>
    <li>Address city can&#39;t be blank</li>
    <li>Address postal code can&#39;t be blank</li>
  </ul>
</div>
```

**4 ข้อผิดพลาดตรงตามที่คาด** เกิดจาก `required_for_order?` คืน `true` เพราะ `delivery_method`
เป็น `"ship"` — สังเกตว่า error message มี prefix `"Address"` นำหน้าทุกข้อความ (Rails เติม prefix
ชื่อ association ให้อัตโนมัติเวลารวม error ของ nested record เข้ามาใน `order.errors` เหมือนที่เห็น
กับ Comment ใน Part 031) เปลี่ยนแค่ `delivery_method` จาก `pickup` เป็น `ship` โดยไม่แก้อะไรอย่าง
อื่นเลย ผลลัพธ์เปลี่ยนจาก **สำเร็จสมบูรณ์** เป็น **validate ไม่ผ่าน 4 จุด** ทันที — พิสูจน์ว่า
conditional validation ทำงานถูกต้องตามเงื่อนไขทางธุรกิจที่ตั้งใจไว้

---

## Step 367: `reject_if` แบบ custom lambda — ซับซ้อนกว่า `:all_blank`

### ทำไม `:all_blank` ไม่พอสำหรับ LineItem

ทบทวนจาก **Part 031 Step 307**: `reject_if: :all_blank` ข้ามแถวที่**ทุก attribute เป็นค่าว่าง
ทั้งหมด** — แต่ปัญหาคือแถว LineItem ที่ผู้ใช้**เลือกสินค้าแล้วแต่ลืมใส่จำนวน** (หรือใส่จำนวนเป็น
`0`) จะ**ไม่ถูก** `:all_blank` ข้าม เพราะ `product_id` ไม่ว่างเปล่า (`all_blank` มองว่ามีข้อมูลอยู่
แล้ว จึงพยายามสร้าง record ต่อ) แล้วก็จะไปติด validation `quantity` ที่ Model แทน กลายเป็นข้อความ
error ที่งงว่า "ทำไมแถวที่ไม่ได้ตั้งใจจะใส่อะไรถึงมา error"

**พิสูจน์จริง** ด้วยการทดสอบ `reject_if: :all_blank` กับแถวที่ `product_id` มีค่าแต่ `quantity`
เป็น `"0"`:

```irb
irb> class TempOrder < ApplicationRecord
irb>   self.table_name = "orders"
irb>   has_many :line_items, foreign_key: :order_id
irb>   accepts_nested_attributes_for :line_items, reject_if: :all_blank
irb> end
irb>
irb> o = TempOrder.new(customer_name: "x", status: "pending")
irb> o.line_items_attributes = { "0" => { "product_id" => "1", "quantity" => "0" } }
irb> o.line_items.size
=> 1
```

`o.line_items.size` เป็น **1** — record ถูกสร้างขึ้นจริงทั้งที่ `quantity: "0"` ไม่มีความหมายทาง
ธุรกิจเลย (ไม่มีใครสั่งซื้อสินค้า "0 ชิ้น") เพราะ `:all_blank` เห็นว่า `product_id` ไม่ blank จึงไม่
ข้ามให้

### เขียน custom lambda ที่ครอบคลุมกรณีนี้ด้วย

```ruby
class Order < ApplicationRecord
  # ...

  accepts_nested_attributes_for :line_items,
    allow_destroy: true,
    reject_if: :blank_line_item?

  private

  # custom reject_if: ข้ามแถวที่ยังไม่ได้เลือกสินค้า "หรือ" จำนวนเป็นค่าว่าง/ศูนย์/ติดลบ
  # ต่างจาก :all_blank เพราะแถวที่มี product_id แต่ quantity เป็น "0" ก็ควรถูกข้ามด้วย
  # (all_blank จะไม่ข้ามให้ เพราะ "0" ไม่ใช่ blank string)
  def blank_line_item?(attrs)
    product_id = attrs["product_id"]
    quantity   = attrs["quantity"].to_i

    product_id.blank? || quantity <= 0
  end
end
```

**อธิบาย:** `reject_if` เมื่อรับเป็น **symbol** (`:blank_line_item?`) จะเรียกเป็นเมธอด**ของ
instance `Order` เอง** (ไม่ใช่ class method อิสระ) พร้อมส่ง `attrs` (Hash ของ attribute ชุดนั้น
ที่ key เป็น **string** เสมอ ไม่ใช่ symbol — สังเกตว่าเขียน `attrs["product_id"]` ไม่ใช่
`attrs[:product_id]`) เป็นอาร์กิวเมนต์เดียว — ต้อง**นิยามเป็น private method** (หรือ public ก็ได้
แต่ไม่ควรเปิดเป็น public API ของ Model โดยไม่จำเป็น) เมธอดต้องคืนค่า truthy/falsy: `true` = ข้าม
record นี้ไปเลย, `false` = ดำเนินการสร้าง/แก้ไขตามปกติ

`quantity.to_i` แปลง string เป็น Integer ก่อนเทียบ (`"0".to_i #=> 0`, `"".to_i #=> 0`,
`nil.to_i` จะ error เพราะ `nil` ไม่มี `.to_i` — แต่ในทางปฏิบัติ `attrs["quantity"]` จะเป็น string
เสมอเพราะมาจาก HTTP params ไม่มีทาง `nil` ตรงๆ นอกจากไม่มี key นั้นส่งมาเลย ซึ่งกรณีนั้น
`attrs["quantity"]` จะเป็น `nil` จริง — ถ้าต้องการรองรับ `nil` ด้วยให้เขียน
`attrs["quantity"].to_i` เป็น `attrs["quantity"].to_s.to_i` แทนเพื่อความปลอดภัย)

**ทดสอบจริงว่า custom lambda นี้ข้ามแถวที่มีปัญหาให้ถูกต้อง:**

```irb
irb> o = Order.new(customer_name: "Test reject_if", status: "pending")
irb> o.line_items_attributes = {
irb>   "0" => { "product_id" => "1", "quantity" => "0" },   # product เลือกแล้ว แต่ quantity=0
irb>   "1" => { "product_id" => "2", "quantity" => "3" }
irb> }
irb> o.line_items.size
=> 1
irb> o.build_address(delivery_method: "pickup")
irb> o.save
irb> o.line_items.count
=> 1
```

**แถวที่ 2 (product_id="2", quantity=3) ถูกสร้างจริง แถวแรก (quantity="0") ถูกข้ามตั้งแต่ก่อน
validate เลย** — `o.line_items.size` เป็น `1` ทันทีตั้งแต่ assign attributes เสร็จ (ก่อนแม้แต่จะ
เรียก `.save` ด้วยซ้ำ) ต่างจากตอนใช้ `:all_blank` ที่ได้ `2` (สร้างทั้งสองแถวรวมแถวที่ไม่มีความหมาย
ด้วย)

> **กฎการเลือกใช้:** ใช้ `:all_blank` เมื่อ logic การข้ามแถวมีแค่ "ทุกช่องว่างเปล่าหมด" พอ (กรณี
> ส่วนใหญ่ของฟอร์มทั่วไปอย่าง Comment ใน Part 031) แต่เมื่อ nested record มี field ที่ "มีค่า
> default ที่ดูเหมือนไม่ว่างเปล่าแต่จริงๆ ไม่มีความหมาย" (เช่น ตัวเลข `0`, select ที่เลือก
> "ไม่ระบุ" ไว้เป็นค่าเริ่มต้น) ต้องเขียน custom lambda/method เองเสมอ — `:all_blank` เช็คแค่
> `.blank?` ของแต่ละ attribute เท่านั้น ไม่มีทางรู้ context ทางธุรกิจว่าค่าไหน "นับว่างเปล่า" ตาม
> ความหมายจริงของระบบนั้นๆ

---

## Step 368: Validate parent จากข้อมูล nested โดยรวม — ต้องมีอย่างน้อย 1 line item

### ทำไม `line_items.empty?` เฉยๆ ไม่พอ

โจทย์ทางธุรกิจ: **Order ต้องมีอย่างน้อย 1 line item เสมอ** (สั่งซื้อ 0 รายการไม่มีความหมาย) วิธี
เขียนที่ดูเหมือนถูกต้องแต่จริงๆ มีบั๊กแฝงคือ:

```ruby
# วิธีที่ผิด (มีบั๊กแฝง)
validate :must_have_line_items

def must_have_line_items
  errors.add(:base, "ต้องมีสินค้าอย่างน้อย 1 รายการ") if line_items.empty?
end
```

**ปัญหา:** ตอนแก้ไขคำสั่งซื้อ (`edit`/`update`) ถ้าผู้ใช้กด "ลบ" line item **ทุกแถวที่มีอยู่ผ่าน
`_destroy`** (ไม่ได้ลบผ่าน `LineItem.destroy` ตรงๆ) `line_items` collection **ยังคงมี record
เหล่านั้นอยู่ในหน่วยความจำ** เพียงแค่ถูก "ทำเครื่องหมายไว้ว่าจะลบ" (`marked_for_destruction?`
เป็น `true`) เท่านั้น **ยังไม่ถูกลบจริงจนกว่า `.save` จะเสร็จสมบูรณ์** — พิสูจน์จริงด้วยการ assign
`_destroy` ให้ทุกแถว:

```irb
irb> o = Order.find(1)   # มี line_items 2 แถว
irb> o.line_items_attributes = {
irb>   "0" => { "id" => "1", "_destroy" => "1" },
irb>   "1" => { "id" => "3", "_destroy" => "1" }
irb> }
irb> o.line_items.empty?
=> false
irb> o.line_items.reject(&:marked_for_destruction?).empty?
=> true
```

**`o.line_items.empty?` คืน `false`** ทั้งที่ line item ทุกแถวถูกทำเครื่องหมายให้ลบหมดแล้ว — ถ้า
`must_have_line_items` เช็คด้วย `.empty?` ตรงๆ จะ**ไม่จับบั๊กนี้เลย** ปล่อยให้บันทึกคำสั่งซื้อที่
กำลังจะเหลือ line item 0 แถวได้สำเร็จ (เพราะตอนที่ validate ทำงาน `_destroy` ยังไม่ถูกดำเนินการจริง)

### วิธีที่ถูกต้อง — กรอง `marked_for_destruction?` ออกก่อนนับ

```ruby
class Order < ApplicationRecord
  # ...
  validate :must_have_line_items

  private

  def must_have_line_items
    active_line_items = line_items.reject(&:marked_for_destruction?)

    errors.add(:base, "ต้องมีสินค้าอย่างน้อย 1 รายการในคำสั่งซื้อ") if active_line_items.empty?
  end
end
```

`line_items.reject(&:marked_for_destruction?)` กรองเอาเฉพาะแถวที่ **ไม่ได้** ถูกทำเครื่องหมายให้
ลบออกมาก่อน แล้วค่อยเช็ค `.empty?` กับผลลัพธ์ที่กรองแล้ว — `marked_for_destruction?` เป็นเมธอกของ
`ActiveRecord::AutosaveAssociation` ที่ Rails เพิ่มให้กับทุก record ที่อยู่ใน nested attributes
association อัตโนมัติ คืน `true` ก็ต่อเมื่อ record นั้นมี `_destroy` เป็นค่า truthy **และ**
`allow_destroy: true` ถูกเปิดใช้งานที่ association นั้น

### ทดสอบจริง — สร้าง Order ไม่มี line item เลย

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=ไม่มีสินค้า" \
  --data-urlencode "order[status]=pending" \
  --data-urlencode "order[line_items_attributes][0][product_id]=" \
  --data-urlencode "order[line_items_attributes][0][quantity]=1" \
  --data-urlencode "order[address_attributes][delivery_method]=pickup" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

```html
<div class="form-errors">
  <h3>1 ข้อผิดพลาดทำให้บันทึกคำสั่งซื้อนี้ไม่ได้:</h3>
  <ul>
    <li>ต้องมีสินค้าอย่างน้อย 1 รายการในคำสั่งซื้อ</li>
  </ul>
</div>
```

(แถวเดียวที่ส่งมามี `product_id` ว่างเปล่า ถูก `reject_if: :blank_line_item?` จาก Step 367 ข้าม
ไปตั้งแต่ต้น เหลือ line item 0 แถวจริงๆ ก่อนที่ `must_have_line_items` จะตรวจพบ)

### ทดสอบจริง — แก้ไข Order ที่มีอยู่แล้วโดยลบ line item ทุกแถวพร้อมกัน

สร้างสถานการณ์ตรงกับบั๊กที่อธิบายไว้ข้างต้นเป๊ะๆ — Order ที่มี 2 line items อยู่แล้ว (id=1, id=3)
ส่ง `PATCH` ที่ลบทั้งคู่พร้อมกันโดยไม่เพิ่มแถวใหม่มาแทนเลย:

```bash
curl -s -c cookies.txt -b cookies.txt -X PATCH http://localhost:3000/orders/1 \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=สมชาย ทดสอบ" \
  --data-urlencode "order[status]=paid" \
  --data-urlencode "order[line_items_attributes][0][id]=1" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=5" \
  --data-urlencode "order[line_items_attributes][0][_destroy]=1" \
  --data-urlencode "order[line_items_attributes][1][id]=3" \
  --data-urlencode "order[line_items_attributes][1][product_id]=3" \
  --data-urlencode "order[line_items_attributes][1][quantity]=1" \
  --data-urlencode "order[line_items_attributes][1][_destroy]=1" \
  --data-urlencode "order[address_attributes][delivery_method]=pickup" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

```html
<div class="form-errors">
  <h3>1 ข้อผิดพลาดทำให้บันทึกคำสั่งซื้อนี้ไม่ได้:</h3>
  <ul>
    <li>ต้องมีสินค้าอย่างน้อย 1 รายการในคำสั่งซื้อ</li>
  </ul>
</div>
```

**ได้ผลตามที่ต้องการทันที** — ถ้าใช้ `line_items.empty?` เฉยๆ (ไม่กรอง
`marked_for_destruction?`) คำขอนี้จะ**ผ่าน validation ไปได้** (เพราะ `line_items` ยังมี 2 record
อยู่ในหน่วยความจำตอนนั้น) ทั้งที่ผลลัพธ์สุดท้ายจะเป็น Order ที่ไม่มี line item เหลือเลยสักแถว
ตรวจสอบยืนยันว่าฐานข้อมูลไม่มีอะไรเปลี่ยนแปลง (ทรานแซกชัน rollback ทั้งหมด):

```irb
irb> Order.find(1).line_items.count
=> 2   # ยังเป็น 2 เท่าเดิม ไม่มีอะไรถูกลบจริงเพราะ validation ไม่ผ่านทั้งฟอร์ม
```

---

## Step 369: แสดง error ของ nested record แยกรายแถว + หมายเหตุความปลอดภัย `limit:`

### ปัญหาของการแสดง error แบบรวมที่ Part 031 ทิ้งไว้

ท้าย Part 031 Step 310 ตั้งข้อสังเกตไว้ว่า error ของ nested record ที่แสดงรวมกับ error ของ parent
(`order.errors.full_messages`) มีปัญหา: ข้อความอย่าง `"Line items product must exist"` **บอกแค่ว่า
"มี line item บางแถวมีปัญหา" แต่ไม่บอกว่าแถวไหน** — ถ้า Order มี line item 5 แถว ผู้ใช้จะไม่รู้เลย
ว่าต้องแก้แถวไหน

### วิธีแก้ — อ่าน error จาก nested object โดยตรงในแต่ละแถวของ `fields_for`

```erb
<%# app/views/orders/_line_item_fields.html.erb (เพิ่ม error ต่อแถว) %>
<div class="line-item-row" data-line-item-row data-persisted="<%= f.object.persisted? %>">
  <%# ... field เดิมทั้งหมด ... %>

  <% f.object.errors.each do |error| %>
    <p class="row-error"><%= error.full_message %></p>
  <% end %>

  <button type="button" class="remove-line-item" onclick="removeLineItemRow(this)">
    ลบรายการนี้
  </button>
</div>
```

**หัวใจสำคัญ:** `f.object` ภายใน `fields_for` block คือ **instance ของ `LineItem` ตัวนั้นแถวนั้น
เป๊ะๆ** ไม่ใช่ `order` — `f.object.errors` จึงมีแค่ error ของ `LineItem` แถวนั้นตัวเดียว ไม่ปนกับ
แถวอื่นหรือกับ error ของ `Order` เอง เพราะ ActiveRecord เก็บ `errors` แยกเป็นของแต่ละ instance
object เสมอ (ทบทวนหลักการนี้จาก **Part 026**) — วิธีนี้ใช้ได้กับ `fields_for :address` เหมือนกัน
ทุกประการ (`af.object.errors`) ตามที่เขียนไว้แล้วใน partial ของ Step 366

### ทดสอบจริง — ให้ error เกิดที่แถวที่ 2 (ดัชนี `[1]`) เท่านั้น ไม่ใช่แถวแรก

ใช้ประโยชน์จากพฤติกรรม `belongs_to :product` ที่บังคับ record ต้องมีอยู่จริง (Step 361) — ส่ง
`product_id` ปลอมที่ไม่มีอยู่จริงไปในแถวที่สอง ส่วนแถวแรกให้ถูกต้องปกติ:

```bash
curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/orders \
  -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=ทดสอบ error รายแถว" \
  --data-urlencode "order[status]=pending" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=1" \
  --data-urlencode "order[line_items_attributes][1][product_id]=9999" \
  --data-urlencode "order[line_items_attributes][1][quantity]=2" \
  --data-urlencode "order[address_attributes][delivery_method]=pickup" \
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 422
```

HTML จริงที่ render กลับมา (ตัดมาเฉพาะส่วนของ 2 แถว):

```html
<!-- แถวที่ 0 (product_id=1 ถูกต้อง) — ไม่มี error เลย -->
<div class="line-item-row" data-line-item-row data-persisted="false">
  <div class="field">
    <label for="order_line_items_attributes_0_product_id">สินค้า</label>
    <select name="order[line_items_attributes][0][product_id]" ...>...</select>
  </div>
  <!-- ไม่มี <p class="row-error"> เลย -->
</div>

<!-- แถวที่ 1 (product_id=9999 ไม่มีอยู่จริง) — มี error เฉพาะแถวนี้ -->
<div class="line-item-row" data-line-item-row data-persisted="false">
  <div class="field">
    <label for="order_line_items_attributes_1_product_id">สินค้า</label>
    <select name="order[line_items_attributes][1][product_id]" ...>...</select>
  </div>
  <p class="row-error">Product must exist</p>
</div>
```

**พิสูจน์ชัดเจน:** row-error `"Product must exist"` ปรากฏ**เฉพาะที่แถวดัชนี `[1]` เท่านั้น** แถว
`[0]` ที่ถูกต้องไม่มี error ใดๆ ติดมาเลย — ต่างจากกล่อง error รวมบนสุดของฟอร์มที่ยังคงแสดงแค่
ข้อความรวมๆ `"Line items product must exist"` (ไม่บอกแถว) เหมือนเดิม:

```html
<div class="form-errors">
  <h3>1 ข้อผิดพลาดทำให้บันทึกคำสั่งซื้อนี้ไม่ได้:</h3>
  <ul>
    <li>Line items product must exist</li>
  </ul>
</div>
```

**สรุปการใช้งานจริง:** เก็บกล่อง error รวมบนสุดไว้เป็น "ภาพรวม" (บอกจำนวนข้อผิดพลาดทั้งหมด) แต่
เพิ่ม error ต่อแถวด้วย `f.object.errors`/`af.object.errors` เพื่อให้ผู้ใช้เห็นตำแหน่งที่ต้องแก้ไข
จริงแบบไม่ต้องเดา — ทั้งสองอย่างไม่ขัดแย้งกัน แสดงพร้อมกันได้ในฟอร์มเดียว

### หมายเหตุความปลอดภัย — `accepts_nested_attributes_for limit:` ป้องกัน mass-assignment DoS

`params.expect`/`permit` ควบคุมแค่ว่า **field ไหน**อนุญาตให้ mass-assign ได้ (ทบทวนจาก Part 031)
แต่**ไม่ได้จำกัดว่าส่ง record ในอาร์เรย์ได้กี่ตัว** — ถ้าไม่มีอะไรป้องกันไว้เลย ผู้ไม่หวังดีสามารถ
เขียน script ส่ง `order[line_items_attributes]` ที่มี **หลักพัน–หลักหมื่นแถว** มาในคำขอเดียว
ทำให้ server ต้อง `build`/validate record จำนวนมหาศาลในคำขอเดียว (สิ้นเปลือง CPU/memory ต่อคำขอ
อย่างไม่สมเหตุสมผล) — นี่คือรูปแบบหนึ่งของ **mass-assignment / resource-exhaustion DoS** ที่
`accepts_nested_attributes_for` มี option **`limit:`** ไว้ป้องกันโดยเฉพาะ:

```ruby
accepts_nested_attributes_for :line_items,
  allow_destroy: true,
  limit: 10,
  reject_if: :blank_line_item?
```

**ทดสอบจริง — ส่ง 12 แถวเข้าไปเกิน limit ที่ตั้งไว้ (10):**

```irb
irb> order = Order.new(customer_name: "Limit test", status: "pending")
irb> attrs = {}
irb> 12.times { |i| attrs[i.to_s] = { "product_id" => "1", "quantity" => "1" } }
irb> order.line_items_attributes = attrs
ActiveRecord::NestedAttributes::TooManyRecords: Maximum 10 records are allowed. Got 12 records instead.
```

`accepts_nested_attributes_for` เช็คจำนวนแถว**ก่อน**แม้แต่จะเริ่ม `reject_if`/validate อะไรเลย
และ **raise exception ทันทีถ้าเกิน limit** — แปลว่าถ้า controller ไม่ได้เตรียมรับมือไว้
exception นี้จะหลุดเป็น **HTTP 500** (unhandled exception) ไม่ใช่ 4xx ที่ควรเป็นเมื่อ client ส่ง
ข้อมูลผิดปกติมา ยืนยันด้วย HTTP request จริง (controller ที่ยังไม่ได้ `rescue_from`):

```bash
curl -s -X POST http://localhost:3000/orders -H "X-CSRF-Token: $TOKEN" \
  --data-urlencode "order[customer_name]=Too many" --data-urlencode "order[status]=pending" \
  --data-urlencode "order[line_items_attributes][0][product_id]=1" \
  --data-urlencode "order[line_items_attributes][0][quantity]=1" \
  # ... (ส่งซ้ำแบบนี้ทั้งหมด 12 แถว)
  -o resp.html -w "HTTP %{http_code}\n"
```

```
HTTP 500
```

log ฝั่ง server ยืนยันสาเหตุตรงๆ:

```
ActiveRecord::NestedAttributes::TooManyRecords (Maximum 10 records are allowed. Got 12 records instead.):
app/controllers/orders_controller.rb:21:in `create'
```

**แก้ให้ตอบกลับสุภาพขึ้นด้วย `rescue_from` ที่ controller:**

```ruby
class OrdersController < ApplicationController
  # ...

  rescue_from ActiveRecord::NestedAttributes::TooManyRecords, with: :reject_too_many_line_items

  private

  def reject_too_many_line_items
    render plain: "รายการสินค้าต่อคำสั่งซื้อเกินจำนวนที่อนุญาต (สูงสุด 10 รายการ)",
           status: :bad_request
  end
end
```

ทดสอบซ้ำด้วยคำขอเดียวกันเป๊ะหลังเพิ่ม `rescue_from`:

```
HTTP 400
รายการสินค้าต่อคำสั่งซื้อเกินจำนวนที่อนุญาต (สูงสุด 10 รายการ)
```

**ผลลัพธ์เปลี่ยนจาก 500 (สื่อว่า "server มีบัค", อาจแสดง stack trace ถ้า
`consider_all_requests_local` เป็น true ใน production ที่ config ผิดพลาด) เป็น 400 (สื่อว่า
"client ส่งข้อมูลผิด" อย่างถูกต้อง) โดยไม่ต้องแก้ logic การตรวจสอบเองเลยแม้แต่บรรทัดเดียว** —
`limit:` ทำหน้าที่ตรวจสอบให้ทั้งหมด สิ่งที่ต้องทำเพิ่มมีแค่ `rescue_from` เพื่อจัดการผลลัพธ์เมื่อ
เกิน limit ให้เหมาะสมเท่านั้น

> **ทำไมไม่ใช้แค่ validation ธรรมดาแทน `limit:`:** ถ้าใช้ validation (เช่น
> `validates :line_items, length: { maximum: 10 }`) การตรวจสอบจะเกิด **หลังจาก** Rails สร้าง
> `LineItem` object ในหน่วยความจำไปแล้วทั้ง 12 (หรือ 12,000) ตัว เสียเวลา CPU ไปกับการ `build`
> object จำนวนมากก่อนจะรู้ว่าต้อง reject ทั้งหมด — `limit:` เช็คจำนวนแถว**ก่อน**สร้าง object ใดๆ
> เลยด้วยซ้ำ (เช็คจากขนาดของ Hash ที่ส่งเข้ามาโดยตรง) จึงประหยัดทรัพยากรกว่ามากในกรณีที่มีคนพยายาม
> ส่งข้อมูลจำนวนมหาศาลเข้ามาโจมตี — ใส่ `limit:` ไว้ที่ทุก `accepts_nested_attributes_for` ที่รับ
> ข้อมูลจากผู้ใช้ภายนอกเป็นแนวปฏิบัติที่ควรทำเป็นมาตรฐาน ไม่ใช่แค่ทางเลือก (เรื่องนี้จะกลับมาเจาะลึก
> อีกครั้งพร้อมช่องโหว่ mass-assignment แบบอื่นๆ ใน **Phase 13 Part 079–080**)

---

## Step 370: แบบฝึกหัด — ฟอร์ม Order/LineItem/Product/Address แบบเต็ม

### โจทย์

ประกอบทุก Step ก่อนหน้าให้เป็นระบบสั่งซื้อสินค้าที่สมบูรณ์ โดยมีเงื่อนไขครบดังนี้:

1. หน้า `orders#new` มีฟอร์มเดียวที่สร้าง `Order` พร้อม `LineItem` (อ้างอิง `Product` ที่มีอยู่
   แล้วเท่านั้น) และ `Address` ได้พร้อมกัน โดยมีปุ่ม "เพิ่มรายการสินค้า" ที่เพิ่มแถวได้แบบไดนามิก
   ด้วย JavaScript (ไม่ต้อง reload หน้า)
2. หน้า `orders#edit` แสดง line item เดิมทั้งหมดพร้อมปุ่ม "ลบรายการนี้" ที่ทำงานถูกต้องทั้งกรณีแถว
   persisted แล้วและแถวที่เพิ่งเพิ่มด้วย JS
3. `Order` ต้อง validate ว่ามี line item ที่ยังไม่ถูกทำเครื่องหมายลบเหลืออยู่อย่างน้อย 1 แถวเสมอ
4. `Address` validate แบบมีเงื่อนไข — บังคับกรอกครบเฉพาะเมื่อเลือกวิธีจัดส่งเป็น `ship`
5. `accepts_nested_attributes_for :line_items` ต้องมี `limit:` ป้องกัน mass-assignment DoS และ
   controller ต้อง `rescue_from` ให้ตอบกลับเป็น 4xx ไม่ใช่ 500 เมื่อเกิน limit
6. Error ของ nested record ต้องแสดงแยกเป็นรายแถวในฟอร์ม ไม่ใช่แค่รวมกันที่กล่อง error บนสุด

### เฉลยแบบเต็ม

โค้ดเฉลยคือชุดเดียวกับที่ประกอบขึ้นทีละส่วนตลอด Step 361–369 — สรุปไฟล์ทั้งหมดอีกครั้งแบบสมบูรณ์
เพื่อความชัดเจน (ไม่มีอะไรใหม่เพิ่มนอกเหนือจากที่อธิบายไปแล้ว)

**`app/models/order.rb`:**

```ruby
class Order < ApplicationRecord
  STATUSES = %w[pending paid shipped cancelled].freeze

  has_many :line_items, dependent: :destroy
  has_many :products, through: :line_items
  has_one :address, dependent: :destroy

  accepts_nested_attributes_for :line_items,
    allow_destroy: true,
    limit: 10,
    reject_if: :blank_line_item?

  accepts_nested_attributes_for :address, allow_destroy: true

  validates :status, inclusion: { in: STATUSES }
  validates :customer_name, presence: true

  validate :must_have_line_items

  after_initialize { self.status ||= "pending" }

  private

  def blank_line_item?(attrs)
    product_id = attrs["product_id"]
    quantity   = attrs["quantity"].to_i

    product_id.blank? || quantity <= 0
  end

  def must_have_line_items
    active_line_items = line_items.reject(&:marked_for_destruction?)

    errors.add(:base, "ต้องมีสินค้าอย่างน้อย 1 รายการในคำสั่งซื้อ") if active_line_items.empty?
  end
end
```

**`app/models/line_item.rb`:**

```ruby
class LineItem < ApplicationRecord
  belongs_to :order
  belongs_to :product

  validates :product_id, presence: true
  validates :quantity, numericality: { only_integer: true, greater_than: 0 }
end
```

**`app/models/address.rb`:**

```ruby
class Address < ApplicationRecord
  belongs_to :order

  DELIVERY_METHODS = %w[pickup ship].freeze

  validates :delivery_method, inclusion: { in: DELIVERY_METHODS }

  with_options if: :required_for_order? do
    validates :recipient_name, presence: true
    validates :line1, presence: true
    validates :city, presence: true
    validates :postal_code, presence: true
  end

  def required_for_order?
    delivery_method == "ship"
  end
end
```

**`app/models/product.rb`:**

```ruby
class Product < ApplicationRecord
  has_many :line_items

  validates :name, presence: true
  validates :sku, presence: true, uniqueness: true
  validates :stock, numericality: { greater_than_or_equal_to: 0 }
  validates :price_cents, numericality: { greater_than: 0 }
end
```

**`app/controllers/orders_controller.rb`:**

```ruby
class OrdersController < ApplicationController
  before_action :set_products, only: %i[new create edit update]
  before_action :set_order, only: %i[show edit update destroy]

  BLANK_LINE_ITEM_ROWS = 3

  rescue_from ActiveRecord::NestedAttributes::TooManyRecords, with: :reject_too_many_line_items

  def index
    @orders = Order.order(created_at: :desc)
  end

  def show
  end

  def new
    @order = Order.new
    @order.build_address
    BLANK_LINE_ITEM_ROWS.times { @order.line_items.build }
  end

  def create
    @order = Order.new(order_params)

    if @order.save
      redirect_to @order, notice: "สร้างคำสั่งซื้อสำเร็จ"
    else
      @order.build_address if @order.address.nil?
      @order.line_items.build if @order.line_items.empty?
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    @order.build_address if @order.address.nil?
    @order.line_items.build
  end

  def update
    if @order.update(order_params)
      redirect_to @order, notice: "แก้ไขคำสั่งซื้อสำเร็จ"
    else
      @order.line_items.build if @order.line_items.empty?
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @order.destroy
    redirect_to orders_path, notice: "ลบคำสั่งซื้อแล้ว"
  end

  private

  def set_order
    @order = Order.find(params[:id])
  end

  def set_products
    @products = Product.order(:name)
  end

  def order_params
    params.expect(
      order: [
        :customer_name, :status,
        { line_items_attributes: [[:id, :product_id, :quantity, :_destroy]] },
        { address_attributes: [:id, :recipient_name, :line1, :city, :postal_code,
                                :delivery_method, :_destroy] }
      ]
    )
  end

  def reject_too_many_line_items
    render plain: "รายการสินค้าต่อคำสั่งซื้อเกินจำนวนที่อนุญาต (สูงสุด 10 รายการ)",
           status: :bad_request
  end
end
```

**`app/views/orders/_line_item_fields.html.erb`, `_form.html.erb`** — ตามที่ประกอบไว้เต็มรูปแบบ
ใน Step 363–365 และ 369 (รวม `<template>`, JS `addLineItemRow`/`removeLineItemRow`, error ต่อแถว)
— **`new.html.erb`/`edit.html.erb`** เรียก `render "form", order: @order` เหมือนกันทั้งคู่ ตาม
หลักการ partial ร่วมที่เรียนมาตั้งแต่ Part 024/030/031

### ทดสอบเงื่อนไขข้อ 1–2 — สร้างและแก้ไขพร้อม dynamic add/remove

ทดสอบไว้ครบแล้วใน Step 364 (เพิ่มแถวด้วย timestamp index) และ Step 365 (`PATCH` เดียวที่แก้ไข
+ ลบ + เพิ่มพร้อมกัน ได้ผลลัพธ์ `[[1, 1, 5], [3, 3, 1]]`) — ทั้งสองกรณีจำลองพฤติกรรมของ JavaScript
ด้วย `curl` ตรงๆ (ส่ง field ชุดเดียวกับที่ JS จะสร้างให้เบราว์เซอร์ submit จริง)

### ทดสอบเงื่อนไขข้อ 3 — validate line items ขั้นต่ำ 1 แถว

ทดสอบไว้ครบแล้วใน Step 368 ทั้งกรณีสร้างใหม่ (แถวเดียวที่ส่งมาว่างเปล่า ถูก `reject_if` กรองจนเหลือ
0 แถว) และกรณีแก้ไขที่ลบทุกแถวพร้อมกัน (`_destroy=1` ทุกแถว ตรวจจับได้ด้วย
`reject(&:marked_for_destruction?)`) — ทั้งสองกรณีได้ `HTTP 422` พร้อมข้อความ
`"ต้องมีสินค้าอย่างน้อย 1 รายการในคำสั่งซื้อ"` ถูกต้องตรงกัน

### ทดสอบเงื่อนไขข้อ 4 — conditional validation ของ Address

ทดสอบไว้ครบแล้วใน Step 366 — `pickup` ไม่ต้องกรอกที่อยู่เลยก็บันทึกสำเร็จ (`HTTP 302`) ส่วน `ship`
ที่ไม่กรอกที่อยู่ได้ `HTTP 422` พร้อม 4 ข้อผิดพลาดตรงกับ field ที่ validate แบบมีเงื่อนไข

### ทดสอบเงื่อนไขข้อ 5 — `limit:` และ `rescue_from`

ทดสอบไว้ครบแล้วใน Step 369 — ส่ง 12 แถวไม่มี `rescue_from` ได้ `HTTP 500`
(`ActiveRecord::NestedAttributes::TooManyRecords` หลุดเป็น unhandled exception) เพิ่ม
`rescue_from` แล้วส่งคำขอเดียวกันเป๊ะได้ `HTTP 400` พร้อมข้อความที่อ่านเข้าใจได้แทน

### ทดสอบเงื่อนไขข้อ 6 — error รายแถว

ทดสอบไว้ครบแล้วใน Step 369 — ส่ง `product_id=9999` (ไม่มีอยู่จริง) ที่แถวดัชนี `[1]` เท่านั้น
HTML ที่ render กลับมามี `<p class="row-error">Product must exist</p>` ปรากฏเฉพาะที่แถว `[1]`
แถว `[0]` ที่ถูกต้องไม่มี error ติดมาเลย — ตรงตามเงื่อนไขข้อ 6 ทุกประการ

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม stock check ตอนสั่งซื้อ** — เพิ่ม custom validation ที่ `LineItem` (หรือ `Order`) ที่
   ตรวจสอบว่า `quantity` ที่สั่งไม่เกิน `product.stock` ที่มีอยู่จริง (ใบ้: ต้องเขียนเป็น
   `validate` แบบ custom method ไม่ใช่ `validates ... numericality` เพราะเงื่อนไขต้องเทียบกับค่า
   ของ record อื่น (`product.stock`) ไม่ใช่ค่าคงที่ — ทบทวนวิธีเขียน custom validator ให้ลึกซึ้ง
   กว่านี้ใน **Part 032**) ทดสอบให้เห็นว่า error message ที่ได้ระบุชัดว่า "สินค้าตัวไหนสั่งเกิน
   สต๊อก" ผ่านการแสดง error รายแถวตามที่เรียนใน Step 369
2. **ทำให้ปุ่ม "เพิ่มรายการสินค้า" หายไปเมื่อถึง limit** — ปัจจุบันปุ่มกดเพิ่มแถวได้เรื่อยๆ แม้ผู้ใช้
   จะกดจนเกิน `limit: 10` ที่ตั้งไว้ (ซึ่งพอ submit จริงจะโดน `rescue_from` ปัดกลับมาเป็น 400) ลอง
   แก้ JavaScript ให้นับจำนวนแถวที่ยังไม่ถูกซ่อน (`display: none`) อยู่ในหน้าปัจจุบัน แล้วปิดการใช้
   งานปุ่ม (`disabled = true`) เมื่อถึง 10 แถวพอดี ป้องกันไม่ให้ผู้ใช้เจอ error ตอน submit เลยตั้งแต่
   แรก (ใบ้: `document.querySelectorAll('[data-line-item-row]:not([style*="display: none"])')`)
3. **เพิ่ม `Category` ให้ `Product` แล้วขยายเป็น 4 ระดับ** — เพิ่มโมเดล `Category has_many :products`
   แล้วให้ฟอร์ม `Order` filter รายการ `Product` ใน `collection_select` ตาม Category ที่เลือกไว้
   (ต้องใช้ JavaScript เพิ่มเพื่อกรอง `<option>` ที่แสดงตาม Category ที่เลือก โดยไม่ reload หน้า —
   ลองคิดว่าจะฝังข้อมูล "Product ตัวไหนอยู่ Category ไหน" ลงใน HTML แบบ `data-*` attribute เพื่อให้
   JavaScript อ่านได้โดยไม่ต้องยิง request ใหม่ไปหา server ทุกครั้งที่เปลี่ยน Category ได้อย่างไร)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ต่อยอด nested attributes จาก Part 031 (ความสัมพันธ์เดียว) มาเป็นฟอร์มที่จัดการ **หลายความสัมพันธ์
  พร้อมกัน** (`has_many :line_items` + `has_one :address`) ในฟอร์มเดียวของ `Order`
- แยกความต่างระหว่าง nested attributes ที่ **อ้างอิง record ที่มีอยู่แล้ว** (`LineItem` ชี้ไปหา
  `Product` ผ่าน `collection_select`) กับที่ **สร้าง record ใหม่จริง** (`Address`) — และรู้ว่า
  `accepts_nested_attributes_for` ใช้เฉพาะกับความสัมพันธ์ที่ฟอร์มมีสิทธิ์สร้าง/แก้ไข/ลบได้จริง
  เท่านั้น
- เตรียมแถวว่างเริ่มต้นที่ controller ด้วย `.build` (`has_many`) และ `build_<association>`
  (`has_one`) พร้อมเข้าใจว่าทำไมการตัดสินใจจำนวนแถวควรอยู่ที่ controller ไม่ใช่ view
- เขียน strong parameters ที่ถูกต้องสำหรับทั้ง `has_many` (double-bracket) และ `has_one`
  (ไม่ต้องมี bracket ซ้อน) ด้วย `params.expect` พร้อมกันในฟอร์มเดียว
- เพิ่ม/ลบแถว nested form แบบไดนามิกด้วย **vanilla JavaScript** ผ่านเทคนิค `<template>` +
  `insertAdjacentHTML` + `fields_for(..., child_index:)` และเข้าใจว่าทำไมต้องแยกวิธีจัดการแถวที่
  persisted แล้ว (ตั้ง `_destroy` + ซ่อน) กับแถวใหม่ (ลบ DOM ตรงๆ) ให้ต่างกัน
- เขียน conditional validation ข้าม nested record ด้วย `with_options if:` (`Address` ต้องกรอก
  ครบเฉพาะเมื่อ `delivery_method == "ship"`)
- เขียน `reject_if` แบบ custom lambda/method ที่ซับซ้อนกว่า `:all_blank` เมื่อ field มีค่า default
  ที่ดูเหมือนไม่ว่างเปล่าแต่ไม่มีความหมายทางธุรกิจ (เช่น `quantity: "0"`)
- Validate parent จากข้อมูล nested โดยรวมได้อย่างถูกต้อง ด้วย
  `line_items.reject(&:marked_for_destruction?)` แทน `.empty?` ตรงๆ (ป้องกันบั๊กที่ record ถูก
  ทำเครื่องหมายลบหมดแล้วแต่ validation ยังไม่จับได้)
- แสดง validation error ของ nested record แยกเป็นรายแถวด้วย `f.object.errors` แทนการรวมไว้ที่กล่อง
  error บนสุดของฟอร์มเพียงอย่างเดียว
- เข้าใจ `accepts_nested_attributes_for limit:` ในฐานะเครื่องมือป้องกัน mass-assignment/DoS ที่
  ระดับ Model และรู้ว่าต้อง `rescue_from ActiveRecord::NestedAttributes::TooManyRecords` เองเสมอ
  ไม่เช่นนั้นจะได้ HTTP 500 แทนที่จะเป็น 4xx ที่เหมาะสม

**ต่อไป (Part 038):** เราจะออกจากเรื่องฟอร์มชั่วคราวไปที่การ**แสดงข้อมูลจำนวนมาก**อย่างมืออาชีพ —
**Pagination** ด้วย gem ยอดนิยมสองตัว **Kaminari** และ **Pagy** (เปรียบเทียบข้อดีข้อเสียของทั้งคู่
พร้อมเหตุผลว่าทำไมทีมใหญ่จำนวนมากเลือก Pagy เพราะประสิทธิภาพที่เหนือกว่า), **sorting** (เรียงลำดับ
ผลลัพธ์ตามคอลัมน์ที่ผู้ใช้เลือกได้แบบปลอดภัย ป้องกัน SQL injection จากชื่อคอลัมน์ที่รับมาจาก
query string), และ **filtering** (กรองข้อมูลตามเงื่อนไขหลายแบบพร้อมกันด้วย pattern ที่ทีมมืออาชีพ
ใช้จริง เช่น Search Object) — เนื้อหานี้จะนำ Order/LineItem/Product ที่สร้างไว้ใน Part นี้มาต่อยอด
เป็นหน้ารายการคำสั่งซื้อที่แบ่งหน้า เรียงลำดับ และกรองได้ครบเครื่องมากขึ้นไปอีกขั้น
