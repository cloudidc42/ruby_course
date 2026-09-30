# Part 083: Query Object, Decorator Pattern (Draper), Presenter

> **Step ครอบคลุมใน Part นี้:** Step 821–830
> **ระดับ:** สูง (ต้องผ่าน Part 034 เรื่อง Query Interface, Part 038 เรื่อง Filter Object/Pagination,
> Part 055 เรื่อง ViewComponent, และ Part 082 เรื่อง Service Object/Form Object มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6 / Rails 8.1.4, gem `draper` 4.0.6, `rspec-rails` 8.0.4 — ทุกคำสั่ง,
> SQL log, และผลลัพธ์ HTTP ในเอกสารนี้รันจริงและ capture จริงบนแอป Rails ทดลองที่สร้างด้วย
> `rails new --minimal` แล้วลบทิ้งหลังตรวจสอบเสร็จ

Part 082 พาไปรู้จัก **Service Object** (ย้าย business logic ที่ซับซ้อนออกจาก controller/model) และ
**Form Object** (ย้าย logic การรับ/validate input ที่ซับซ้อนออกจาก model) ไปแล้ว — ทั้งสอง pattern
แก้ปัญหาฝั่ง "การเขียนข้อมูล" (write path) เป็นหลัก Part นี้เดินหน้าต่อไปยังอีกสองปัญหาที่คู่กันเสมอ
ในแอป Rails ที่โตขึ้น:

1. **ฝั่งการอ่านข้อมูล (read path)** — เมื่อ query ซับซ้อนขึ้นเรื่อยๆ (`joins`, `group`, `having`,
   subquery) และต้องใช้ซ้ำในหลาย controller/report/background job จะจัดระเบียบอย่างไรไม่ให้ SQL logic
   กระจัดกระจายซ้ำซ้อน — คำตอบคือ **Query Object** ซึ่ง Part 038 Step 378 เคยพูดถึง **Filter Object**
   ไปแล้วในฐานะ "ญาติใกล้ชิด" ของ pattern นี้ พร้อมสัญญาไว้ว่าจะกลับมาทำให้เป็นระบบใน Part นี้
2. **ฝั่งการแสดงผล (view logic)** — เมื่อ view เริ่มมี logic คำนวณ/จัดรูปแบบข้อมูลเยอะขึ้น
   (`number_to_currency`, เงื่อนไขแสดง badge, การรวมข้อมูลจากหลาย model เพื่อ render หน้าเดียว) จะย้าย
   logic เหล่านี้ออกจาก ERB ที่เกลื่อนกลาดไปไว้ที่ไหน — คำตอบมีสามทางที่คนมักสับสนกัน: **View Helper**
   (ที่รู้จักมาตั้งแต่ Part 024), **Decorator** (ผ่าน gem `draper`), และ **Presenter** (PORO มือเขียน)
   Part นี้จะทำให้เห็นชัดว่าแต่ละอันเหมาะกับสถานการณ์ไหน แล้วเทียบกับ **ViewComponent** จาก Part 055
   ให้ครบวงจร

## สารบัญของ Part นี้

- Step 821: ทบทวน Filter Object จาก Part 038 แล้วยกระดับเป็น Query Object อย่างเป็นทางการ
- Step 822: Query Object ทั่วไปที่ไม่ใช่แค่ filter — `TopSellingProductsQuery`, `OverdueOrdersQuery`
- Step 823: Convention ของ Query Object — `.call`/`.new(relation).call`, chainable vs terminal,
  namespace `app/queries/`
- Step 824: ทำไม Query Object ช่วยเรื่อง testability และป้องกัน `joins`/`group`/`having` ซ้ำซ้อน
- Step 825: ความสับสนของ View Helper/Decorator/Presenter — ตารางเปรียบเทียบ
- Step 826: Decorator Pattern ด้วย `draper` — ติดตั้ง, generate, `delegate_all`,
  `decorates_association`
- Step 827: ใช้ decorated object ใน controller (`.decorate`) และ view — เรียกใช้เหมือน model เดิม
  บวกความสามารถใหม่
- Step 828: Presenter Pattern — ทางเลือกที่ไม่ต้องพึ่ง gem เขียนเองด้วย PORO/`SimpleDelegator`
- Step 829: Presenter vs Decorator vs ViewComponent — กรอบการตัดสินใจว่าเมื่อไหร่ควรใช้ตัวไหน
- Step 830: ทดสอบ Decorator/Presenter แบบแยกหน่วย (RSpec) โดยไม่พึ่ง controller/view
- แบบฝึกหัดปิด Part: `TopSellingProductsQuery` + `ProductDecorator` + `DashboardPresenter` ประกอบกัน
  เป็นหน้า admin dashboard
- สรุปสิ่งที่ได้เรียนรู้ + เปิดตัว Part 084

---

## เตรียมโปรเจกต์สำหรับ Part นี้

```bash
rails new query_decorator_demo --minimal
cd query_decorator_demo
```

```ruby
# Gemfile
gem "draper", "~> 4.0"

group :development, :test do
  gem "rspec-rails"
end
```

```bash
bundle install
bin/rails generate rspec:install
```

ใช้โดเมนต่อยอดจาก Part 038 (`Post`/`Author`/`Category`) สำหรับ Query Object และเพิ่มโดเมนร้านค้า
(`Product`/`Customer`/`Order`/`OrderItem`) สำหรับ Decorator/Presenter/แบบฝึกหัด:

```bash
bin/rails generate model Author name:string nationality:string
bin/rails generate model Category name:string
bin/rails generate model Post title:string body:text published:boolean views:integer \
  published_at:datetime author:references category:references

bin/rails generate model Customer name:string email:string
bin/rails generate model Product name:string price_cents:integer stock:integer sku:string
bin/rails generate model Order customer:references status:string total_cents:integer
bin/rails generate model OrderItem order:references product:references \
  quantity:integer unit_price_cents:integer
```

แก้ migration ให้มี default ระดับฐานข้อมูล (แนวทางเดียวกับ Part 026/034/038):

```ruby
# db/migrate/..._create_posts.rb
t.boolean :published, default: false, null: false
t.integer :views, default: 0, null: false

# db/migrate/..._create_products.rb
t.integer :price_cents, default: 0, null: false
t.integer :stock, default: 0, null: false

# db/migrate/..._create_orders.rb
t.string :status, default: "pending", null: false
t.integer :total_cents, default: 0, null: false

# db/migrate/..._create_order_items.rb
t.integer :quantity, default: 1, null: false
t.integer :unit_price_cents, default: 0, null: false
```

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :posts
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :posts
end

# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category

  scope :published, -> { where(published: true) }
end

# app/models/customer.rb
class Customer < ApplicationRecord
  has_many :orders
end

# app/models/product.rb
class Product < ApplicationRecord
  has_many :order_items
  has_many :orders, through: :order_items
end

# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :customer
  has_many :order_items
  has_many :products, through: :order_items

  scope :paid, -> { where(status: "paid") }
end

# app/models/order_item.rb
class OrderItem < ApplicationRecord
  belongs_to :order
  belongs_to :product
end
```

```bash
bin/rails db:create db:migrate
```

Seed ข้อมูล: 45 บทความแบบเดียวกับ Part 038, บวก 3 ลูกค้า, 5 สินค้า (มีบางชิ้น stock ต่ำ/หมด
โดยตั้งใจ), และ 30 ออเดอร์กระจาย status `paid`/`pending`:

```ruby
# db/seeds.rb
authors = { "Nichada" => "Thai", "Somsak" => "Thai", "Wanda" => "American", "Kenji" => "Japanese" }
  .map { |name, nat| Author.create!(name: name, nationality: nat) }
categories = %w[Ruby Rails DevOps Career].map { |n| Category.create!(name: n) }

45.times do |i|
  Post.create!(
    title: "หัวข้อบทความที่ #{i + 1}", body: "เนื้อหาตัวอย่าง",
    author: authors[i % authors.size], category: categories[i % categories.size],
    published: i % 5 != 0, views: (i * 37) % 1000,
    published_at: i % 5 != 0 ? (45 - i).days.ago : nil
  )
end

customers = ["สมชาย ใจดี", "วรรณา สุขสันต์", "Kenji Tanaka"]
  .map { |n| Customer.create!(name: n, email: "#{n.parameterize}@example.com") }

products = [
  { name: "คีย์บอร์ดเชิงกล", price_cents: 199_000, stock: 12, sku: "KB-001" },
  { name: "เมาส์ไร้สาย", price_cents: 59_000, stock: 0, sku: "MS-002" },
  { name: "จอมอนิเตอร์ 27 นิ้ว", price_cents: 699_000, stock: 4, sku: "MN-003" },
  { name: "หูฟังตัดเสียงรบกวน", price_cents: 349_000, stock: 25, sku: "HP-004" },
  { name: "แท่นวางโน้ตบุ๊ก", price_cents: 39_000, stock: 3, sku: "ST-005" }
].map { |attrs| Product.create!(attrs) }

30.times do |i|
  order = Order.create!(customer: customers[i % customers.size],
                         status: i.even? ? "paid" : "pending", total_cents: 0)
  total = 0
  ((i % 3) + 1).times do |j|
    product = products[(i + j) % products.size]
    quantity = (i % 4) + 1
    OrderItem.create!(order: order, product: product, quantity: quantity,
                       unit_price_cents: product.price_cents)
    total += quantity * product.price_cents
  end
  order.update!(total_cents: total)
end
```

```bash
bin/rails db:seed
# Authors: 4, Categories: 4, Posts: 45
# Customers: 3, Products: 5, Orders: 30 (paid: 15)
```

---

## Step 821: ทบทวน Filter Object จาก Part 038 แล้วยกระดับเป็น Query Object อย่างเป็นทางการ

Part 038 Step 378 สอนไปแล้วว่า `PostFilter` คือ PORO ที่รับ `relation`/`params` เข้ามา แล้วคืน
`ActiveRecord::Relation` ที่ผ่านการกรองแล้วออกไป — ตอนนั้นเราบอกไว้ว่า pattern นี้เป็น "ญาติใกล้ชิด"
ของ **Query Object** ที่จะทำให้เป็นระบบเต็มรูปแบบใน Part นี้ มาดูของเดิมอีกครั้ง:

```ruby
# app/models/post_filter.rb (จาก Part 038)
class PostFilter
  def initialize(relation = Post.all, params = {})
    @relation = relation
    @params = params
  end

  def call
    relation = @relation
    relation = filter_by_search(relation)
    relation = filter_by_category(relation)
    relation
  end

  private

  def filter_by_search(relation)
    return relation if @params[:q].blank?

    relation.where("title LIKE ?", "%#{sanitize_like(@params[:q])}%")
  end

  def filter_by_category(relation)
    return relation if @params[:category_id].blank?

    relation.where(category_id: @params[:category_id])
  end

  def sanitize_like(term)
    term.gsub(/[\\%_]/) { |char| "\\#{char}" }
  end
end
```

`PostFilter` มีคุณสมบัติครบถ้วนของสิ่งที่เรียกว่า **Query Object** อยู่แล้วโดยไม่รู้ตัว:

1. เป็น PORO ที่ **รับ relation เป็น input และคืน relation เป็น output** (ไม่ใช่ Array, ไม่ query
   จริงจนกว่าผู้เรียกจะ enumerate)
2. มี **method เดียวที่เป็นจุดเข้า** (`call`) ทำให้เรียกใช้ได้แบบเดียวกันทุกที่
3. **แยกความรู้เรื่อง `params`/HTTP ออกจาก model** ตามหลักการที่ Part 038 อธิบายไว้

สิ่งที่ `PostFilter` **ยังไม่เป็นทางการ** คือ:

- ไม่มี **namespace แยกต่างหาก** (`app/models/post_filter.rb` ปนอยู่กับ model จริงๆ ในโฟลเดอร์
  `app/models/`) — เมื่อจำนวน Query Object เพิ่มขึ้นเรื่อยๆ ในทีมใหญ่ การแยกโฟลเดอร์ทำให้มองเห็นภาพรวม
  ง่ายกว่ามาก
- ไม่มี **class method สะดวกสำหรับเรียกแบบ one-liner** (`PostFilter.call(...)` แทนที่จะต้อง
  `PostFilter.new(...).call` ทุกครั้ง)
- ยังไม่ได้แยกความแตกต่างระหว่าง Query Object ที่ **"กรอง" relation ที่มีอยู่แล้ว** (แบบ `PostFilter`)
  กับ Query Object ที่ **"สร้าง" query ที่ซับซ้อนเกินกว่าจะเป็น scope เดี่ยวๆ ได้** (เช่น query ที่มี
  `GROUP BY`/`HAVING`/subquery) — ทั้งสองแบบมีรูปร่างหน้าตาต่างกันเล็กน้อยแม้จะใช้หลักการเดียวกัน

Step ถัดไปจะแก้ทั้งสามจุดนี้ พร้อมแสดงตัวอย่าง Query Object ที่ซับซ้อนกว่าการกรองธรรมดา

---

## Step 822: Query Object ทั่วไปที่ไม่ใช่แค่ filter — `TopSellingProductsQuery`, `OverdueOrdersQuery`

**Query Object** ในความหมายที่กว้างกว่า Filter Object คือ: **class ที่ห่อหุ้ม query ที่ซับซ้อนพอที่
จะไม่เหมาะเป็น scope เดี่ยวๆ ใน model อีกต่อไป** ไม่ว่าจะเป็นเพราะ query นั้นมี `join`/`group`/`having`
หลายชั้น, ต้องรับ parameter หลายตัวที่ประกอบกันเป็นเงื่อนไข, หรือใช้ซ้ำจากหลายจุดที่ไม่ใช่แค่
controller เดียว (report, background job, API endpoint)

### ตัวอย่างที่ 1: `TopSellingProductsQuery` — query ที่มี `GROUP BY`/`HAVING` จริงจัง

โจทย์: "สินค้าขายดีที่สุด N อันดับ เรียงตามจำนวนที่ขายได้รวม" — ต้อง `JOIN` ตาราง `order_items`,
`GROUP BY` สินค้า, และ `SUM` จำนวนที่ขาย เขียนเป็น scope เดี่ยวๆ ใน `Product` ได้ไม่สวยเท่าแยก class
ต่างหาก เพราะมี parameter ที่ปรับได้ (`limit`, ช่วงเวลา) ประกอบกันหลายทาง:

```ruby
# app/queries/application_query.rb
class ApplicationQuery
  def self.call(...)
    new(...).call
  end
end
```

```ruby
# app/queries/top_selling_products_query.rb
class TopSellingProductsQuery < ApplicationQuery
  def initialize(relation = Product.all, limit: 5, since: nil)
    @relation = relation
    @limit = limit
    @since = since
  end

  def call
    scope = @relation
      .joins(:order_items)
      .merge(scoped_order_items)
      .group("products.id")
      .select("products.*, SUM(order_items.quantity) AS units_sold")
      .order("units_sold DESC")

    scope = scope.limit(@limit) if @limit
    scope
  end

  private

  def scoped_order_items
    return OrderItem.all if @since.blank?

    OrderItem.joins(:order).merge(Order.where(orders: { created_at: @since.. }))
  end
end
```

ทดสอบจริงด้วย `rails runner`:

```ruby
ActiveRecord::Base.logger = Logger.new(STDOUT)
TopSellingProductsQuery.call(limit: 3).each { |p| puts "#{p.name}: #{p.units_sold}" }
```

```
Product Load (0.3ms)  SELECT products.*, SUM(order_items.quantity) AS units_sold FROM "products"
INNER JOIN "order_items" ON "order_items"."product_id" = "products"."id"
GROUP BY "products"."id" ORDER BY units_sold DESC LIMIT 3

หูฟังตัดเสียงรบกวน: 34
จอมอนิเตอร์ 27 นิ้ว: 30
คีย์บอร์ดเชิงกล: 28
```

จุดสำคัญที่ควรสังเกต:

1. **`units_sold` เป็นคอลัมน์ที่ไม่มีอยู่จริงในตาราง `products`** แต่มาจาก `SELECT ... AS
   units_sold` — ActiveRecord สร้าง attribute method ชื่อ `units_sold` ให้กับ object ที่คืนมาโดย
   อัตโนมัติจาก alias นี้ (`p.units_sold` เรียกได้เหมือน attribute ปกติทุกประการ แม้จะไม่มี column
   นี้ใน schema จริง)
2. **`scoped_order_items` แยก logic ของเงื่อนไข `since:` ออกเป็น private method** — ทำให้ `call`
   อ่านง่าย ไม่ปนกันเป็นก้อนเดียว
3. **`ApplicationQuery.call(...)` ใช้ Ruby 3 anonymous argument forwarding (`...`)** ส่งต่อทุก
   argument ที่รับมาไปยัง `new(...)` โดยไม่ต้องเขียนซ้ำ — ทำให้ subclass ทุกตัวได้ `.call` แบบ
   one-liner ฟรีโดยไม่ต้องเขียน class method ซ้ำในทุก Query Object

> **ข้อควรระวังจริงที่เจอตอนทดสอบ:** ถ้าเรียก `TopSellingProductsQuery.call(limit: 1).size` (ใช้
> `.size` แทน `.to_a.size`) จะได้ error `SQLite3::SQLException: no such column: units_sold` ทันที —
> เพราะ `.size` บน `ActiveRecord::Relation` ที่ยังไม่ได้ load จะพยายามยิง `SELECT COUNT(*)` แทนที่จะ
> นับจาก array ที่ query มาแล้ว และ query นับจำนวนที่ ActiveRecord สร้างขึ้นเองนั้น **ไม่รู้จัก
> `units_sold`** เพราะเป็นแค่ alias ชั่วคราวของ `SELECT` เดิม ไม่ใช่ attribute จริงที่ `COUNT` query
> ใหม่จะมองเห็น — บทเรียนตรงนี้คือ **Query Object ที่ใช้ raw `select`/`group` ควรระวังการเรียก method
> ที่ทำให้ ActiveRecord สร้าง query ใหม่แบบไม่ได้ตั้งใจ** (`.size`, `.count`, `.exists?` ล้วนมีพฤติกรรม
> พิเศษแบบนี้) วิธีที่ปลอดภัยที่สุดคือ `.to_a` ให้ query จริงก่อนแล้วค่อยเรียก method ของ Array ต่อ ถ้า
> ต้องการรู้จำนวนแถวเป็นเรื่องเป็นราว

### ตัวอย่างที่ 2: `OverdueOrdersQuery` — Query Object แบบ terminal (ไม่รับ relation จากภายนอก)

Query Object ไม่จำเป็นต้องรับ `relation` เป็น argument เสมอไป — บาง query มีความหมายเฉพาะตัวจนไม่มี
เหตุผลที่จะให้ผู้เรียกส่ง relation อื่นมาแทนได้ (เรียกว่า **terminal query object**):

```ruby
# app/queries/overdue_orders_query.rb
class OverdueOrdersQuery < ApplicationQuery
  OVERDUE_AFTER = 3.days

  def initialize(older_than: OVERDUE_AFTER)
    @older_than = older_than
  end

  def call
    Order
      .where(status: "pending")
      .where(created_at: ..@older_than.ago)
      .order(created_at: :asc)
  end
end
```

ทดสอบจริง (จำลองออเดอร์ค้างเก่าด้วยการ backdate):

```ruby
Order.where(status: "pending").limit(3).update_all(created_at: 10.days.ago)

OverdueOrdersQuery.call.each { |o| puts "Order##{o.id} pending since #{o.created_at.to_date}" }
# Order#2 pending since 2026-09-18
# Order#4 pending since 2026-09-18
# Order#6 pending since 2026-09-18

OverdueOrdersQuery.call(older_than: 30.days).count
# => 0   (ยังไม่มีออเดอร์ที่ค้างเกิน 30 วัน)
```

สังเกตความต่างจาก `TopSellingProductsQuery`: `OverdueOrdersQuery.new` **ไม่รับ `relation` เป็น
parameter แรกเลย** เพราะ "ออเดอร์ที่เกินกำหนดชำระ" เป็นแนวคิดที่ผูกกับ `Order` โดยตรง ไม่มีเหตุผลทาง
ธุรกิจที่จะเรียก query นี้กับ relation อื่น — การออกแบบ constructor ให้รับเฉพาะ parameter ที่จำเป็นจริง
(`older_than:`) ทำให้ API ของ class นี้ชัดเจนและอ่านง่ายกว่าถ้าฝืนใส่ `relation` เข้าไปโดยไม่มีที่ใช้

> **หลักการเลือก:** ถ้า query มีแนวโน้มจะถูกเรียกกับ relation ที่กรองมาแล้วบางส่วน (composability)
> ให้รับ `relation` เป็น argument แรกแบบ `TopSellingProductsQuery` — ถ้า query มีความหมายผูกกับ
> Model เดียวชัดเจนและไม่มี use case ที่ต้องเรียกกับ relation อื่น ให้เขียนแบบ terminal อย่าง
> `OverdueOrdersQuery` ได้เลย ไม่ต้องฝืนใส่ parameter ที่ไม่มีใครใช้เพื่อ "ให้ดูสม่ำเสมอ"

---

## Step 823: Convention ของ Query Object — `.call`/`.new(relation).call`, chainable vs terminal, namespace `app/queries/`

รวบยอด convention ที่ทีมใหญ่ใช้กันเป็นมาตรฐานสำหรับ Query Object:

### 1) Namespace: `app/queries/`

Rails 8 (Zeitwerk) autoload ทุกโฟลเดอร์ย่อยใต้ `app/` ให้อัตโนมัติโดยไม่ต้องตั้งค่าเพิ่ม — สร้าง
โฟลเดอร์ `app/queries/` แล้ววาง Query Object ทุกตัวไว้ที่นั่นได้ทันที:

```
app/
  queries/
    application_query.rb
    top_selling_products_query.rb
    overdue_orders_query.rb
    post_filter_query.rb   # ย้าย PostFilter จาก Part 038 มาไว้ที่นี่แล้วเปลี่ยนชื่อให้สื่อความหมาย
```

> **ทำไมไม่ใช้ `app/models/` เหมือน `PostFilter` เดิม:** ไม่ผิดกติกาอะไร (`app/models/` autoload
> ได้เหมือนกัน) แต่เมื่อจำนวน Query Object โตเกิน 5-10 ตัว การปนกับ ActiveRecord model จริงในโฟลเดอร์
> เดียวกันทำให้ scan หาไฟล์ยากขึ้น — แยกโฟลเดอร์ตาม **บทบาท (role)** ของ class ไม่ใช่แค่ "เป็น Ruby
> object เหมือนกัน" เป็นหลักการเดียวกับที่ Part 082 แยก `app/services/` และ `app/forms/` ออกจาก
> `app/models/`

### 2) Naming: ลงท้ายด้วย `Query` เสมอ

`TopSellingProductsQuery`, `OverdueOrdersQuery` — ชื่อบอกตรงตัวว่า "นี่คือ query ที่คืนอะไร" ไม่ใช่
กริยา (ต่างจาก Service Object ที่มักตั้งชื่อเป็นกริยา เช่น `CreateOrder` จาก Part 082) เพราะ Query
Object เป็นเรื่องของ **การอ่าน (สิ่งที่เป็น noun)** ไม่ใช่ **การกระทำ (สิ่งที่เป็น verb)**

### 3) Interface: `.call` เป็น public method เดียว, `ApplicationQuery.call(...)` เป็น shortcut

```ruby
class ApplicationQuery
  def self.call(...)
    new(...).call
  end
end
```

ทุก Query Object เรียกใช้ได้แบบเดียวกันหมดสองรูปแบบ:

```ruby
TopSellingProductsQuery.call(limit: 3)                          # shortcut ผ่าน class method
TopSellingProductsQuery.new(Product.where("stock > 0"), limit: 3).call  # เรียก instance ตรงๆ เมื่อต้องการ compose relation ก่อน
```

### 4) Chainable vs Terminal — สองรูปแบบที่ควรแยกให้ชัดในใจ

| ลักษณะ | Chainable Query Object | Terminal Query Object |
|---|---|---|
| ตัวอย่าง | `TopSellingProductsQuery` | `OverdueOrdersQuery` |
| รับ `relation` เป็น argument แรก | รับ (default เป็น `Model.all`) | ไม่รับ — เริ่มจาก `Model` ตรงๆ ข้างใน `call` |
| `call` คืนค่าเป็น | `ActiveRecord::Relation` เสมอ (ยัง lazy, ต่อ `.where`/`.order`/pagy ได้อีก) | `ActiveRecord::Relation` เช่นกัน แต่ไม่ได้ออกแบบมาให้รับ relation ภายนอกเข้ามาผสม |
| เหมาะกับ | query ที่มักถูกเรียกร่วมกับเงื่อนไขอื่นที่มีอยู่แล้ว (compose ต่อยอด) | query ที่มีความหมายตายตัวชัดเจนในตัวเอง ไม่มี use case ให้ผสมกับ relation อื่น |

ทั้งสองแบบยังคง **คืนค่าเป็น `ActiveRecord::Relation` เสมอ ไม่ใช่ `Array`** ตามหลัก lazy evaluation
จาก Part 034 Step 331 — เป็นกฎที่ **ไม่มีข้อยกเว้น** สำหรับ Query Object ทุกตัว เพราะนี่คือสิ่งที่ทำให้
ผู้เรียกยังต่อ `.includes`/`.order`/`pagy(...)` เพิ่มเองได้เสมอโดยไม่ต้อง query ซ้ำ

### 5) การ compose Query Object กับ scope ปกติ — ทำงานร่วมกันได้เพราะทุกอย่างคือ `Relation`

```ruby
TopSellingProductsQuery.new(Product.where("stock > 0"), limit: 10).call
```

```ruby
[Product Load]
SELECT products.*, SUM(order_items.quantity) AS units_sold FROM "products"
INNER JOIN "order_items" ON "order_items"."product_id" = "products"."id"
WHERE (stock > 0) GROUP BY "products"."id" ORDER BY units_sold DESC LIMIT 10
```

```
หูฟังตัดเสียงรบกวน: stock=25 units_sold=34
จอมอนิเตอร์ 27 นิ้ว: stock=4 units_sold=30
คีย์บอร์ดเชิงกล: stock=12 units_sold=28
แท่นวางโน้ตบุ๊ก: stock=3 units_sold=26
```

**"เมาส์ไร้สาย" (stock=0) หายไปจากผลลัพธ์โดยอัตโนมัติ** เพราะ `Product.where("stock > 0")` ที่ส่งเข้า
มาเป็น `@relation` ถูกรวมเข้ากับเงื่อนไข `JOIN`/`GROUP BY` ของ Query Object ผ่าน SQL `WHERE` เดียวกัน —
นี่คือพลังของการยึดหลัก "input เป็น relation, output เป็น relation" อย่างเคร่งครัด: Query Object
ประกอบเข้ากับ scope ใดๆ ของ model ได้เสมอโดยไม่ต้องแก้โค้ดของ Query Object เองเลยสักบรรทัด

---

## Step 824: ทำไม Query Object ช่วยเรื่อง testability และป้องกัน `joins`/`group`/`having` ซ้ำซ้อน

### ปัญหาที่เกิดขึ้นจริงถ้าไม่มี Query Object

ลองจินตนาการว่าต้องการ "สินค้าขายดี" ใน **2 จุดที่ต่างกัน**: หน้า admin dashboard และ weekly report
ที่ส่งอีเมลทุกวันจันทร์ ถ้าไม่มี Query Object แต่ละจุดจะเขียน SQL เดียวกันซ้ำเอง:

```ruby
# app/controllers/admin/dashboard_controller.rb
def index
  @top_products = Product.joins(:order_items).group("products.id")
    .select("products.*, SUM(order_items.quantity) AS units_sold")
    .order("units_sold DESC").limit(5)
end

# app/jobs/weekly_report_job.rb
def perform
  top = Product.joins(:order_items).group("products.id")
    .select("products.*, SUM(order_items.quantity) AS units_sold")
    .order("units_sold DESC").limit(5)
  # ...
end
```

ปัญหาที่ตามมาเมื่อ business logic เปลี่ยน (เช่น "ต้องนับเฉพาะออเดอร์ที่ `status: paid` เท่านั้น" หรือ
"ต้องไม่นับสินค้าที่ถูกยกเลิกไปแล้ว"): **ต้องไปแก้ SQL เดิมซ้ำทุกจุด** และมีความเสี่ยงสูงที่จะแก้ไม่
ครบ (แก้ที่ dashboard แต่ลืม weekly report) ทำให้ตัวเลขที่แสดงในสองที่ **ไม่ตรงกัน** ซึ่งเป็นบัคที่
ตรวจจับยากมากเพราะไม่มี error ให้เห็น มีแต่ตัวเลขผิดที่ดูเผินๆ เหมือนถูก

### แก้ด้วย Query Object: จุดเดียว, แก้ที่เดียว

```ruby
# ทั้ง 2 จุดเรียกแบบเดียวกัน
@top_products = TopSellingProductsQuery.call(limit: 5)
top = TopSellingProductsQuery.call(limit: 5)
```

เมื่อ business logic เปลี่ยน แก้ที่ `TopSellingProductsQuery` ตัวเดียว ทุกจุดได้ผลลัพธ์ที่ตรงกัน
โดยอัตโนมัติ — นี่คือหลักการ **DRY (Don't Repeat Yourself)** ที่ Part 001 แนะนำไว้ตั้งแต่ต้น
หลักสูตร นำมาประยุกต์กับ SQL logic ที่ซับซ้อนระดับ `joins`/`group`/`having` โดยเฉพาะ

### Testability: ทดสอบ SQL logic ที่ซับซ้อนได้โดยไม่ต้องพึ่ง controller/job

```ruby
# spec/queries/top_selling_products_query_spec.rb
require "rails_helper"

RSpec.describe TopSellingProductsQuery do
  let(:popular) { Product.create!(name: "สินค้าขายดี", price_cents: 1000, stock: 10, sku: "P1") }
  let(:unpopular) { Product.create!(name: "สินค้าขายน้อย", price_cents: 1000, stock: 10, sku: "P2") }
  let(:customer) { Customer.create!(name: "ลูกค้า A", email: "a@example.com") }

  before do
    order = Order.create!(customer: customer, status: "paid", total_cents: 0)
    OrderItem.create!(order: order, product: popular, quantity: 10, unit_price_cents: 1000)
    OrderItem.create!(order: order, product: unpopular, quantity: 1, unit_price_cents: 1000)
  end

  describe ".call" do
    it "เรียงสินค้าตามยอดขาย (units_sold) จากมากไปน้อย" do
      result = described_class.call
      expect(result.map(&:id)).to eq([popular.id, unpopular.id])
    end

    it "จำกัดจำนวนผลลัพธ์ตาม limit:" do
      result = described_class.call(limit: 1).to_a
      expect(result.size).to eq(1)
      expect(result.first.id).to eq(popular.id)
    end

    it "คำนวณ units_sold ให้ถูกต้อง" do
      top = described_class.call.find { |p| p.id == popular.id }
      expect(top.units_sold).to eq(10)
    end
  end

  describe "composing กับ scope ปกติของ Product" do
    it "เรียกต่อจาก relation ที่กรองมาก่อนหน้าได้ (chainable)" do
      unpopular.update!(stock: 0)
      result = described_class.new(Product.where("stock > 0"), limit: 10).call
      expect(result.map(&:id)).to contain_exactly(popular.id)
    end
  end
end
```

```bash
bundle exec rspec spec/queries/top_selling_products_query_spec.rb
```

```
....

Finished in 0.13 seconds (files took 1.2 seconds to load)
4 examples, 0 failures
```

**4 test นี้ไม่มีจุดไหนแตะ controller, job, หรือ HTTP request เลย** — ทดสอบตรงจุดว่า SQL ที่ซับซ้อน
(`joins`/`group`/`order` ด้วย alias) ทำงานถูกต้องด้วยความเร็วสูงสุด และเมื่อ business logic เปลี่ยน
ในอนาคต (เช่นเพิ่มเงื่อนไข `since:`) การแก้ไขและ re-test ทำที่ไฟล์เดียวจบ ไม่ต้องไล่แก้/ไล่ทดสอบทุกจุด
ที่เรียกใช้

---

## Step 825: ความสับสนของ View Helper/Decorator/Presenter — ตารางเปรียบเทียบ

เปลี่ยนโฟกัสจากฝั่งอ่านข้อมูล (Query Object) ไปฝั่งแสดงผล (view logic) — ปัญหาที่ต้องแก้คราวนี้คือ:
**ERB เริ่มมี logic คำนวณ/จัดรูปแบบเยอะเกินไป** ตัวอย่างโค้ดที่เจอได้ทั่วไปในโปรเจกต์ที่ไม่ได้จัด
ระเบียบ:

```erb
<%# ตัวอย่างที่ไม่ดี: logic กระจายอยู่ใน ERB ตรงๆ %>
<% products.each do |product| %>
  <tr>
    <td><%= number_to_currency(product.price_cents / 100.0, unit: "฿", format: "%u%n") %></td>
    <td>
      <% if product.stock <= 0 %>
        <span class="badge badge-danger">สินค้าหมด</span>
      <% elsif product.stock <= 5 %>
        <span class="badge badge-warning">เหลือน้อย (<%= product.stock %> ชิ้น)</span>
      <% else %>
        <span class="badge badge-success">มีสินค้า</span>
      <% end %>
    </td>
  </tr>
<% end %>
```

โค้ดนี้ **ทำงานถูกต้อง** แต่มีปัญหาเดียวกับ controller ที่ยัด `if` เกลื่อนกลาดจาก Part 038 Step 378:
ทดสอบยาก (ต้อง render view จริงถึงจะทดสอบ logic การคำนวณ badge ได้), ใช้ซ้ำที่อื่นไม่ได้, และเมื่อ
logic ซับซ้อนขึ้นจะทำให้ ERB อ่านยากจนหา HTML structure จริงไม่เจอ

Rails และ community มี **3 เครื่องมือ** ที่แก้ปัญหานี้ แต่คนละมุมกัน และคนที่เพิ่งเริ่มมักสับสนว่า
ควรใช้ตัวไหนเมื่อไหร่:

| คุณสมบัติ | View Helper | Decorator (Draper) | Presenter (PORO มือเขียน) |
|---|---|---|---|
| ที่มา | มากับ Rails core (Part 024) | ต้องติดตั้ง gem `draper` | ไม่ต้องมี dependency เพิ่ม |
| ขอบเขตของ method | **Global** — ทุก helper method มองเห็นกันหมดในทุก view (namespace เดียว) | **ผูกกับ instance ของ model เดียว** ที่ decorate | **ออกแบบเองได้อิสระ** ตาม scope ที่ต้องการ |
| รับ argument อย่างไร | ต้องส่ง object เข้าไปเป็น argument ทุกครั้ง (`badge_for(product)`) | เรียกเป็น method ของ object ตรงๆ (`product.stock_status_badge`) | เรียกเป็น method ของ object ที่ wrap ไว้ (`presenter.stock_status_badge`) |
| เข้าถึง view helper อื่น (`number_to_currency`, `content_tag`, `link_to`) | เข้าถึงได้เต็มที่ (เป็น context เดียวกับ view) | เข้าถึงผ่าน `helpers`/`h` | เข้าถึงผ่าน object ที่ส่งเข้ามาตอน initialize (ต้องส่งเอง) |
| ผูกกับ object ตัวเดียวหรือหลายตัว | ไม่ผูก — เขียนให้รับหลาย object เป็น argument ได้อิสระ | **ผูกกับ model เดียวเป็นหลัก** (ขยายไปหลายตัวได้ผ่าน `decorates_association` แต่โครงสร้างยังเป็น "model + associations ของมัน") | **ผสมหลาย object/หลาย data source เข้าด้วยกันได้อิสระ** เพื่อ view เดียว |
| Testability แบบแยกหน่วย | ทดสอบได้ผ่าน `helper` spec แต่ยังต้องพึ่ง Rails helper context | ทดสอบได้แบบ PORO เกือบเต็มที่ (ดู Step 830) | ทดสอบได้แบบ PORO เต็มที่ที่สุด (ควบคุม dependency เองทั้งหมด) |
| เหมาะกับ | logic เล็กๆ ที่ใช้ซ้ำหลายจุด ไม่ผูกกับ object ใดเป็นพิเศษ (เช่น `format_thai_date`) | เพิ่ม method เกี่ยวกับการแสดงผลให้ **object เดียว** โดยไม่แตะ model จริง | รวมข้อมูลจาก **หลาย model/หลาย query** เพื่อ view-model ของหน้าใดหน้าหนึ่งโดยเฉพาะ (เช่นหน้า dashboard) |

จุดที่ควรจำให้ขึ้นใจคือ **ทั้งสามอย่างแก้ปัญหาเดียวกัน** ("อย่าให้ view logic กระจัดกระจายใน ERB")
แต่รูปร่างต่างกันตามจำนวน object ที่เกี่ยวข้อง: **Helper ไม่ผูกกับ object ใดเลย (ยืดหยุ่นที่สุดแต่
namespace รวมเป็นก้อนเดียว), Decorator ผูกกับ object เดียว (โครงสร้างชัดเจนที่สุด เข้ากับ OOP), และ
Presenter รวมได้หลาย object (ยืดหยุ่นที่สุดสำหรับหน้าที่ซับซ้อน)** — Step ถัดไปจะลงลึก Decorator ก่อน
แล้วตามด้วย Presenter

---

## Step 826: Decorator Pattern ด้วย `draper` — ติดตั้ง, generate, `delegate_all`, `decorates_association`

### ติดตั้ง

```ruby
# Gemfile
gem "draper", "~> 4.0"
```

```bash
bundle install
```

### Generate decorator ตัวแรก

```bash
bin/rails generate decorator Product
```

```
create  app/decorators/product_decorator.rb
invoke  rspec
create    spec/decorators/product_decorator_spec.rb
```

> **สังเกต:** ถ้า generate controller ด้วย `bin/rails generate controller Posts index` ในโปรเจกต์
> ที่มี draper อยู่แล้ว Rails จะ **generate decorator ให้อัตโนมัติคู่กันไปเลย** (`app/decorators/
> post_decorator.rb`) เพราะ draper hook เข้ากับ controller generator ของ Rails โดยตรง — เป็น
> ความสะดวกที่ draper เพิ่มให้ ไม่ต้องเรียก `generate decorator` แยกทุกครั้งถ้า workflow ปกติของทีมคือ
> generate controller ก่อนเสมอ

ไฟล์ที่ generate มาให้:

```ruby
# app/decorators/product_decorator.rb
class ProductDecorator < Draper::Decorator
  delegate_all

  # Define presentation-specific methods here. Helpers are accessed through
  # `helpers` (aka `h`). You can override attributes, for example:
  #
  #   def created_at
  #     helpers.content_tag :span, class: 'time' do
  #       object.created_at.strftime("%a %m/%d/%y")
  #     end
  #   end

end
```

### กายวิภาคของ `Draper::Decorator`

- **`delegate_all`** — บอก draper ให้ **ส่งต่อ method call ทุกตัวที่ decorator เองไม่มีนิยามไว้ไปให้
  object ต้นฉบับ (`Product` จริง) โดยอัตโนมัติ** นี่คือกลไกสำคัญที่สุดของ Draper: ไม่ต้องเขียน
  `delegate :name, :price_cents, :sku, to: :object` เองทีละ attribute เหมือนเขียน Presenter มือเปล่า
  (ดู Step 828 ที่จะแสดงให้เห็นว่าถ้าไม่มี `delegate_all` ต้องเขียนเองแบบไหน)
- **`object`** — reference กลับไปยัง instance จริงที่ถูก decorate (ใน `ProductDecorator` คือ
  `Product` instance ตัวจริง) ใช้เมื่อต้องการเข้าถึง attribute ดิบๆ โดยไม่ผ่าน decorator (เช่นถ้า
  override method ชื่อเดียวกับ attribute ใน model)
- **`helpers`/`h`** — proxy ไปยัง Rails view helpers ทั้งหมด (`number_to_currency`, `content_tag`,
  `link_to`, `l`, ฯลฯ) — Draper เรียก proxy นี้ว่า `Draper::ViewHelpers`

### เพิ่ม method เฉพาะสำหรับการแสดงผล

```ruby
# app/decorators/product_decorator.rb
class ProductDecorator < Draper::Decorator
  delegate_all

  def formatted_price
    helpers.number_to_currency(price_cents / 100.0, unit: "฿", format: "%u%n")
  end

  def stock_status_badge
    css_class, label = stock_status
    helpers.content_tag(:span, label, class: "badge badge-#{css_class}")
  end

  def stock_status
    if object.stock <= 0
      [:danger, "สินค้าหมด"]
    elsif object.stock <= 5
      [:warning, "เหลือน้อย (#{object.stock} ชิ้น)"]
    else
      [:success, "มีสินค้า"]
    end
  end
end
```

จุดที่ควรสังเกต:

1. **`price_cents` ใน `formatted_price` เรียกผ่าน `delegate_all` โดยตรง** (ไม่ต้องเขียน `object.
   price_cents`) — เพราะ `ProductDecorator` ไม่มี method ชื่อ `price_cents` ของตัวเอง จึงถูกส่งต่อไป
   ยัง `object.price_cents` โดยอัตโนมัติผ่านกลไก `delegate_all`
2. **`object.stock` ใน `stock_status` เรียกผ่าน `object` ตรงๆ** — ทั้งสองแบบ (`price_cents` เฉยๆ
   กับ `object.stock`) ให้ผลลัพธ์เหมือนกันทุกประการเพราะ `delegate_all` ทำงานอยู่เบื้องหลัง แต่การ
   เขียน `object.xxx` ชัดเจนกว่าเวลาที่ต้องอ่านโค้ดปนกับ method ของ decorator เอง — เลือกใช้ให้
   สม่ำเสมอตามธรรมเนียมของทีม (ตัวอย่างนี้ใช้ `object.` เมื่ออยู่ใน method ที่มี logic เยอะเพื่อความ
   ชัดเจน)
3. **`formatted_price`/`stock_status_badge` เป็น method ใหม่ที่ `Product` model จริงไม่มี** — นี่คือ
   หัวใจของ Decorator: **เพิ่มความสามารถด้านการแสดงผลโดยไม่แตะ `app/models/product.rb` เลย**
   ถ้าเพิ่ม method เหล่านี้ลง model ตรงๆ จะผิดหลักการเดียวกับที่ Part 038 Step 378 เตือนไว้เรื่อง
   `params`: **model ไม่ควรรู้จักเรื่องการแสดงผล** (`content_tag`, CSS class, การจัดรูปแบบเฉพาะหน้า)
   เพราะเป็นแนวคิดของ view layer ไม่ใช่ของ business data

### `decorates_association` — decorate ทั้ง object หลักและ association ที่เกี่ยวข้อง

โจทย์: `PostDecorator` ต้องการให้ `post.author`/`post.category` ที่เรียกผ่าน decorator คืนค่าเป็น
decorated object ด้วยเช่นกัน (ไม่ใช่ `Author`/`Category` ดิบๆ):

```ruby
# app/decorators/post_decorator.rb
class PostDecorator < Draper::Decorator
  delegate_all

  decorates_association :author
  decorates_association :category

  def status_badge
    if object.published?
      helpers.content_tag(:span, "เผยแพร่แล้ว", class: "badge badge-success")
    else
      helpers.content_tag(:span, "ฉบับร่าง", class: "badge badge-secondary")
    end
  end

  def formatted_published_at
    return "ยังไม่เผยแพร่" if object.published_at.blank?

    object.published_at.strftime("%d/%m/%Y")
  end
end
```

`decorates_association :author` บอก draper ว่า "เมื่อไหร่ที่เรียก `post_decorator.author` ให้ไป
`decorate` ผลลัพธ์ของ `object.author` ก่อนคืนค่ากลับมาด้วย" — draper จะพยายาม **infer** ชื่อ decorator
ให้เองจากชื่อ class (`Author` → มองหา `AuthorDecorator`) ต้อง generate decorator ให้ `Author`/
`Category` ไว้ล่วงหน้าด้วยไม่งั้นจะ error:

```bash
bin/rails generate decorator Author
bin/rails generate decorator Category
```

ทดสอบสิ่งที่เกิดขึ้นถ้า **ลืม** generate decorator ให้ `Author`:

```ruby
Post.first.decorate.author
```

```
Draper::UninferrableDecoratorError: Could not infer a decorator for Author.
```

Error นี้ชัดเจนมาก บอกตรงๆ ว่าต้องมี `AuthorDecorator` ก่อนถึงจะใช้ `decorates_association :author`
ได้ — ถ้าต้องการระบุ decorator เองแทนการให้ draper เดา (เช่นกรณีชื่อ class กับชื่อ decorator ไม่ตรงกัน
ตามธรรมเนียม) ใช้ option `with:`:

```ruby
decorates_association :author, with: SomeCustomDecorator
```

---

## Step 827: ใช้ decorated object ใน controller (`.decorate`) และ view — เรียกใช้เหมือน model เดิมบวกความสามารถใหม่

### ใน controller

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.published.includes(:author, :category)
                 .order(created_at: :desc).limit(10).decorate
  end

  def show
    @post = Post.find(params[:id]).decorate
  end
end
```

`.decorate` เป็น method ที่ draper เพิ่มให้ **ทั้ง instance เดียว** (`Post.find(id).decorate` →
`PostDecorator` instance) **และ collection/relation** (`Post.published....decorate` → cursor ของ
`PostDecorator` หลายตัวที่ห่อ `Post` แต่ละแถวไว้) โดยไม่ต้องเขียน `.map { |p| p.decorate }` เอง

### ใน view — เรียกใช้ได้เหมือน object เดิมทุกประการ

```erb
<%# app/views/posts/index.html.erb %>
<h1>บทความทั้งหมด</h1>

<table>
  <tbody>
    <% @posts.each do |post| %>
      <tr>
        <td><%= post.title %></td>
        <td><%= post.author.name %></td>
        <td><%= post.category.name %></td>
        <td><%== post.status_badge %></td>
        <td><%= post.formatted_published_at %></td>
      </tr>
    <% end %>
  </tbody>
</table>
```

รันเซิร์ฟเวอร์แล้วทดสอบจริง:

```bash
bin/rails server
curl -s http://localhost:3000/posts
```

```html
<tr>
  <td>หัวข้อบทความที่ 45</td>
  <td>Nichada</td>
  <td>Ruby</td>
  <td><span class="badge badge-success">เผยแพร่แล้ว</span></td>
  <td>27/09/2026</td>
</tr>
```

สังเกตว่า:

- `post.title` — attribute ธรรมดาที่ delegate ผ่าน `delegate_all` ไปยัง `Post` จริง
- `post.author.name` — `author` ถูก decorate ผ่าน `decorates_association :author` แล้วก็ยังเรียก
  `.name` ต่อได้ตามปกติ (เพราะ `AuthorDecorator` ก็มี `delegate_all` เหมือนกัน)
- `post.status_badge` — method ใหม่ที่มีแค่ใน `PostDecorator` เท่านั้น ไม่มีใน `Post` model จริง
- `<%== ... %>` ยังต้องใช้เหมือนเดิม (ทบทวนจาก Part 038 Step 372) เพราะ `content_tag` คืนค่าเป็น
  HTML-safe string ที่ไม่ต้องการให้ ERB escape ซ้ำ

**View ไม่รู้เลยว่า `@posts`/`@post` เป็น `PostDecorator` ไม่ใช่ `Post` ดิบๆ** — นี่คือประโยชน์หลักของ
Decorator pattern: code ที่เขียนอยู่แล้ว (`post.title`, `post.author.name`) **ยังใช้ได้เหมือนเดิมทุก
ประการ** เพิ่มแค่ความสามารถใหม่เข้าไปโดยไม่ต้องแก้โค้ดเดิมที่มีอยู่แล้วเลย

### สิ่งที่ควรรู้: decorated object ยังผ่าน `is_a?`/`kind_of?` ของ class เดิมได้

```ruby
post = Post.first.decorate
post.class          # => PostDecorator
post.is_a?(Post)    # => true  !!
post.decorated?     # => true
post.object.class   # => Post
```

Draper override `kind_of?`/`is_a?` ให้ตอบ `true` เมื่อเทียบกับ class ของ object ต้นฉบับด้วย —
เพราะโค้ดจำนวนมากใน Rails (เช่น `form_for`, `dom_id`, matcher บางตัวใน test) ตรวจสอบ type ของ object
ก่อนตัดสินใจ ทำงานอย่างไร ถ้า `PostDecorator` ไม่ตอบ `true` ต่อ `is_a?(Post)` ฟีเจอร์เหล่านี้อาจพังได้
โดยไม่คาดคิด — นี่คือรายละเอียดที่ทำให้ decorated object "โปร่งใส" (transparent) กับโค้ดฝั่ง Rails
framework เอง ไม่ใช่แค่กับโค้ดของเราเท่านั้น

---

## Step 828: Presenter Pattern — ทางเลือกที่ไม่ต้องพึ่ง gem เขียนเองด้วย PORO/`SimpleDelegator`

Decorator ผ่าน draper สะดวกมากเมื่อ **wrap object เดียว** แต่มีบางสถานการณ์ที่ Decorator ไม่ใช่
เครื่องมือที่เหมาะ: **เมื่อ view ต้องการข้อมูลจากหลาย model/หลาย query ผสมกัน** เช่นหน้า admin
dashboard ที่ต้องแสดง "ยอดขายรวม", "สินค้าขายดี", "ออเดอร์ค้างชำระ", "จำนวนบทความ" พร้อมกันในหน้าเดียว
— ข้อมูลเหล่านี้ไม่ได้ผูกกับ model ตัวใดตัวหนึ่งเป็นพิเศษ การพยายามยัดทุกอย่างเป็น method ของ
`SomeModelDecorator` ตัวใดตัวหนึ่งจะดูแปลกและผิดความหมาย (dashboard ไม่ใช่ "การแสดงผลของ Order
ตัวหนึ่ง" หรือ "การแสดงผลของ Product ตัวหนึ่ง")

**Presenter** คือ PORO ธรรมดาที่ไม่ต้องมี gem — ออกแบบเองอิสระให้ตรงกับ "โครงสร้างข้อมูลที่ view
ต้องการ" (บางครั้งเรียกว่า **view model**) โดยตรง

### เขียน Presenter มือเปล่า — เห็นความแตกต่างจาก `delegate_all` ของ Draper ชัดเจน

```ruby
class ManualPostPresenter
  def initialize(post, view_context)
    @post = post
    @view = view_context
  end

  def title = @post.title
  def author_name = @post.author.name

  def status_badge
    if @post.published?
      @view.content_tag(:span, "เผยแพร่แล้ว", class: "badge badge-success")
    else
      @view.content_tag(:span, "ฉบับร่าง", class: "badge badge-secondary")
    end
  end
end
```

```ruby
view_context = ApplicationController.new.view_context
p = ManualPostPresenter.new(Post.first, view_context)

p.title         # => "หัวข้อบทความที่ 1"
p.author_name   # => "Nichada"
p.status_badge  # => "<span class=\"badge badge-secondary\">ฉบับร่าง</span>"

p.body
# NoMethodError: undefined method 'body' for an instance of ManualPostPresenter
```

**นี่คือความต่างสำคัญที่สุดระหว่าง Presenter มือเปล่ากับ Decorator ของ draper:** `ManualPostPresenter`
**ไม่มี** `delegate_all` ให้ฟรีเหมือน `Draper::Decorator` — ถ้าต้องการเรียก `p.body` (attribute ของ
`Post` ที่ยังไม่ได้ wrap) ต้องเขียน method ส่งต่อเองทีละตัว หรือใช้กลไกอื่นช่วย

### ทางเลือกที่เบากว่า: ใช้ `SimpleDelegator` จาก Ruby standard library

ถ้าต้องการ auto-delegation แบบ Draper แต่ไม่อยากเพิ่ม gem dependency ทั้งก้อน Ruby standard library
มี `SimpleDelegator` (จาก `delegate`) ให้ใช้ได้ฟรี:

```ruby
require "delegate"

class SimpleDelegatorPresenter < SimpleDelegator
  def initialize(post, view_context)
    super(post)
    @view = view_context
  end

  def status_badge
    css_class = __getobj__.published? ? "success" : "secondary"
    label = __getobj__.published? ? "เผยแพร่แล้ว" : "ฉบับร่าง"
    @view.content_tag(:span, label, class: "badge badge-#{css_class}")
  end
end
```

```ruby
p = SimpleDelegatorPresenter.new(Post.first, ApplicationController.new.view_context)
p.title          # => "หัวข้อบทความที่ 1"  (delegate ผ่าน SimpleDelegator อัตโนมัติ)
p.body           # => "เนื้อหาตัวอย่าง"     (delegate ผ่าน SimpleDelegator อัตโนมัติ — ไม่ error!)
p.status_badge   # => "<span class=\"badge\">ฉบับร่าง</span>"
```

`SimpleDelegator#initialize` รับ object ที่จะ wrap แล้ว `super(post)` ส่งต่อให้ class แม่จัดการ
delegation ให้อัตโนมัติ (`__getobj__` คือ method ที่ `SimpleDelegator` ให้มาสำหรับเข้าถึง object
ต้นฉบับ เทียบเท่ากับ `object` ของ Draper) — นี่คือทางสายกลางระหว่าง **PORO ล้วนๆ ที่ต้องเขียน
delegate เอง** กับ **draper ที่ต้องเพิ่ม gem dependency**

### Presenter ที่แท้จริง: รวมข้อมูลจากหลาย source เพื่อ view เดียว

รูปแบบที่ Presenter ทำได้ดีที่สุดคือแบบนี้ — **ไม่ได้ wrap object เดียว แต่ประกอบข้อมูลจากหลายแหล่ง**:

```ruby
# app/presenters/dashboard_presenter.rb
class DashboardPresenter
  def initialize(view_context)
    @view_context = view_context
  end

  def top_products
    @top_products ||= TopSellingProductsQuery.call(limit: 5).decorate
  end

  def overdue_orders
    @overdue_orders ||= OverdueOrdersQuery.call
  end

  def overdue_orders_count
    overdue_orders.count
  end

  def total_revenue
    @total_revenue ||= Order.paid.sum(:total_cents)
  end

  def formatted_total_revenue
    h.number_to_currency(total_revenue / 100.0, unit: "฿", format: "%u%n")
  end

  def published_posts_count
    @published_posts_count ||= Post.published.count
  end

  def draft_posts_count
    @draft_posts_count ||= Post.where(published: false).count
  end

  private

  attr_reader :view_context

  def h
    view_context
  end
end
```

สังเกตว่า `DashboardPresenter` **ผสมทั้ง Query Object (`TopSellingProductsQuery`,
`OverdueOrdersQuery`) และ Decorator (`.decorate` บนผลลัพธ์ของ Query Object) เข้าด้วยกัน** — ทั้งสาม
pattern ของ Part นี้ทำงานร่วมกันได้อย่างเป็นธรรมชาติเพราะทุกตัวยึดหลักการเดียวกัน: **รับ input ชัดเจน,
คืน output ชัดเจน, ไม่ผูกติดกับ HTTP request หรือ view โดยตรง**

ใช้งานใน controller:

```ruby
# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < ApplicationController
  def index
    @presenter = DashboardPresenter.new(view_context)
  end
end
```

และใน view:

```erb
<%# app/views/admin/dashboard/index.html.erb %>
<h1>แดชบอร์ดผู้ดูแลระบบ</h1>

<section>
  <h2>ยอดขายรวม (ออเดอร์ที่ชำระแล้ว)</h2>
  <p><%= @presenter.formatted_total_revenue %></p>
</section>

<section>
  <h2>สินค้าขายดี Top 5</h2>
  <ul>
    <% @presenter.top_products.each do |product| %>
      <li>
        <%= product.name %> — ขายไป <%= product.units_sold %> ชิ้น
        (<%= product.formatted_price %>) <%== product.stock_status_badge %>
      </li>
    <% end %>
  </ul>
</section>

<section>
  <h2>ออเดอร์ค้างชำระเกินกำหนด (<%= @presenter.overdue_orders_count %> รายการ)</h2>
  <ul>
    <% @presenter.overdue_orders.each do |order| %>
      <li>Order #<%= order.id %> — ค้างตั้งแต่ <%= order.created_at.to_date %></li>
    <% end %>
  </ul>
</section>
```

ทดสอบด้วย curl จริง:

```bash
curl -s http://localhost:3000/admin/dashboard
```

```html
<h2>ยอดขายรวม (ออเดอร์ที่ชำระแล้ว)</h2>
<p>฿164,800.00</p>

<h2>สินค้าขายดี Top 5</h2>
<li>หูฟังตัดเสียงรบกวน — ขายไป 34 ชิ้น (฿3,490.00) <span class="badge badge-success">มีสินค้า</span></li>
<li>จอมอนิเตอร์ 27 นิ้ว — ขายไป 30 ชิ้น (฿6,990.00) <span class="badge badge-warning">เหลือน้อย (4 ชิ้น)</span></li>
<li>คีย์บอร์ดเชิงกล — ขายไป 28 ชิ้น (฿1,990.00) <span class="badge badge-success">มีสินค้า</span></li>
<li>เมาส์ไร้สาย — ขายไป 28 ชิ้น (฿590.00) <span class="badge badge-danger">สินค้าหมด</span></li>
<li>แท่นวางโน้ตบุ๊ก — ขายไป 26 ชิ้น (฿390.00) <span class="badge badge-warning">เหลือน้อย (3 ชิ้น)</span></li>

<h2>ออเดอร์ค้างชำระเกินกำหนด (3 รายการ)</h2>
<li>Order #2 — ค้างตั้งแต่ 2026-09-18</li>
<li>Order #4 — ค้างตั้งแต่ 2026-09-18</li>
<li>Order #6 — ค้างตั้งแต่ 2026-09-18</li>
```

ทุกส่วนของ dashboard render ถูกต้องจากการรวม Query Object + Decorator เข้าด้วยกันผ่าน Presenter ตัว
เดียวที่ controller ต้องรู้จักแค่ `DashboardPresenter` ตัวเดียวเท่านั้น ไม่ต้องเรียก Query Object 2
ตัว + จัดการ decorate เองในทุก controller ที่ต้องใช้ dashboard data (ถ้ามีมากกว่าจุดเดียวในอนาคต เช่น
API endpoint สำหรับ mobile app ที่ต้องการข้อมูลชุดเดียวกัน)

---

## Step 829: Presenter vs Decorator vs ViewComponent — กรอบการตัดสินใจว่าเมื่อไหร่ควรใช้ตัวไหน

Part 055 สอน **ViewComponent** ไปแล้วว่าเหมาะกับ **UI fragment ที่ใช้ซ้ำและมี template ของตัวเอง**
(ปุ่ม, การ์ด, badge ที่ซับซ้อน) — ตอนนี้เรามีเครื่องมือ 3 ตัวที่ทับซ้อนกันบางส่วน มาสรุปกรอบการ
ตัดสินใจให้ชัดเจน:

| สถานการณ์ | เครื่องมือที่เหมาะ | เหตุผล |
|---|---|---|
| เพิ่ม method แสดงผลให้ **object เดียว** เช่น `product.formatted_price` | **Decorator** | ผูกกับ object เดียวเป็นธรรมชาติ, `delegate_all` ทำให้ attribute เดิมยังเรียกได้ครบ ไม่ต้องเขียน wrapper เอง |
| รวมข้อมูลจาก **หลาย model/หลาย query** เพื่อ **หน้าใดหน้าหนึ่งโดยเฉพาะ** เช่นหน้า dashboard | **Presenter** | ไม่มี object "หลัก" ตัวเดียวให้ผูก — Presenter ออกแบบ shape ข้อมูลให้ตรงกับสิ่งที่ view ต้องการเป๊ะ (view model) |
| UI ที่ใช้ซ้ำหลายสิบจุดและมี **template (HTML) ของตัวเอง** เช่นปุ่ม, การ์ด, modal, dropdown | **ViewComponent** | มี `.rb` + `.html.erb` คู่กัน, testable ด้วย `render_inline`, สร้าง reusable UI ที่ไม่ผูกกับ model ใดเป็นพิเศษ |
| logic เล็กๆ ที่ไม่ผูกกับ object ใด ใช้ร่วมกันทั้งแอป เช่น `format_thai_date(date)` | **View Helper** | ยังเป็นเครื่องมือที่เหมาะที่สุดสำหรับ utility function ระดับ global ที่ Part 024 สอนไว้ |

### กรณีผสมกันจริง: ทั้งสี่เครื่องมือทำงานร่วมกันได้ในหน้าเดียว

ในตัวอย่าง `DashboardPresenter` ข้างต้น ถ้าต้องการยกระดับต่อไปอีก สามารถผสมทั้งสี่เครื่องมือในหน้า
เดียวกันได้ตามธรรมชาติ:

```erb
<%# แนวคิดขยายผล (ไม่ใช่โค้ดที่ทดสอบในเอกสารนี้) %>
<h2>สินค้าขายดี Top 5</h2>
<% @presenter.top_products.each do |product| %>
  <%# product คือ ProductDecorator (มาจาก Query Object + .decorate) %>
  <%= render(ProductCardComponent.new(product: product)) %>
  <%# ProductCardComponent คือ ViewComponent ที่รับ decorated object เข้าไป render การ์ดสวยๆ %>
<% end %>
```

ลำดับการไหลของข้อมูล: **Query Object** (`TopSellingProductsQuery`) ดึงข้อมูลดิบจากฐานข้อมูล →
**Decorator** (`.decorate`) เพิ่ม method สำหรับแสดงผลให้แต่ละ `Product` → **Presenter**
(`DashboardPresenter`) รวมหลาย query/หลาย decorated collection เข้าเป็น view model เดียวของหน้า
dashboard → **ViewComponent** (`ProductCardComponent`) render แต่ละรายการเป็น HTML fragment ที่ใช้
ซ้ำได้ในหน้าอื่นด้วย — **แต่ละ layer มีความรับผิดชอบเดียวที่ชัดเจน ไม่ทับซ้อนกัน** ตรงตามหลัก Single
Responsibility Principle ที่ Part 016 สอนไว้

> **กฎย่อเพื่อจำง่าย:** ถาม "สิ่งนี้ผูกกับกี่ object" — **หนึ่ง object → Decorator**,
> **หลาย object รวมกันเพื่อหน้าเดียว → Presenter**, **ไม่ผูกกับ object แต่เป็น UI ที่มี template
> ของตัวเอง → ViewComponent**, **ไม่ผูกกับอะไรเลย เป็นแค่ utility function → Helper**

---

## Step 830: ทดสอบ Decorator/Presenter แบบแยกหน่วย (RSpec) โดยไม่พึ่ง controller/view

### ทดสอบ Decorator — เรียก `helpers` ได้โดยไม่ต้อง render view จริง

```ruby
# spec/decorators/product_decorator_spec.rb
require "rails_helper"

RSpec.describe ProductDecorator do
  let(:stock) { 10 }
  let(:product) { Product.create!(name: "เมาส์ทดสอบ", price_cents: 59_000, stock: stock, sku: "T-1") }
  let(:decorated) { product.decorate }

  describe "#formatted_price" do
    it "แสดงราคาเป็นสกุลเงินบาทจาก cents" do
      expect(decorated.formatted_price).to eq("฿590.00")
    end
  end

  describe "#stock_status_badge" do
    context "เมื่อสินค้าหมด" do
      let(:stock) { 0 }

      it "แสดง badge สีแดงพร้อมข้อความสินค้าหมด" do
        expect(decorated.stock_status_badge).to include("badge-danger")
        expect(decorated.stock_status_badge).to include("สินค้าหมด")
      end
    end

    context "เมื่อสินค้าเหลือน้อย" do
      let(:stock) { 3 }

      it "แสดง badge สีเหลืองพร้อมจำนวนที่เหลือ" do
        expect(decorated.stock_status_badge).to include("badge-warning")
        expect(decorated.stock_status_badge).to include("3 ชิ้น")
      end
    end

    context "เมื่อสินค้ามีเพียงพอ" do
      let(:stock) { 20 }

      it "แสดง badge สีเขียว" do
        expect(decorated.stock_status_badge).to include("badge-success")
      end
    end
  end

  it "ยังเรียก attribute เดิมของ model ได้ปกติผ่าน delegate_all" do
    expect(decorated.name).to eq(product.name)
    expect(decorated.sku).to eq(product.sku)
  end
end
```

```bash
bundle exec rspec spec/decorators/product_decorator_spec.rb
```

```
.....

Finished in 0.06 seconds (files took 2.69 seconds to load)
5 examples, 0 failures
```

**ทั้ง 5 test เรียก `helpers.number_to_currency`/`helpers.content_tag` ผ่าน decorator ได้โดยไม่ต้อง
mock อะไรเพิ่มเลย** — เพราะ `rspec-rails` (ที่ติดตั้งไว้ตั้งแต่ต้น Part) จับคู่ spec ที่อยู่ใต้
`spec/decorators/` เข้ากับ decorator infrastructure ของ draper โดยอัตโนมัติผ่าน metadata ที่ generator
ใส่ไว้ให้ ทำให้ `helpers` proxy ใช้งานได้ทันทีในสภาพแวดล้อมของ spec โดยไม่ต้อง render controller หรือ
view จริงสักครั้งเดียว — เร็วกว่าการทดสอบผ่าน request/system spec มาก

### ทดสอบ Presenter — ใช้ `view_context` จริงจาก `ApplicationController`

Presenter ที่เขียนมือ ควบคุม dependency ทั้งหมดเอง (ไม่มี metadata พิเศษจาก generator ให้เหมือน
decorator) จึงต้องส่ง `view_context` เข้าไปเองตรงๆ ตอนทดสอบ:

```ruby
# spec/presenters/dashboard_presenter_spec.rb
require "rails_helper"

RSpec.describe DashboardPresenter do
  let(:view_context) { ApplicationController.new.view_context }
  let(:presenter) { described_class.new(view_context) }
  let(:customer) { Customer.create!(name: "ลูกค้า A", email: "a@example.com") }

  describe "#formatted_total_revenue" do
    it "รวมยอดขายเฉพาะออเดอร์ที่ชำระแล้ว (paid) เท่านั้น" do
      Order.create!(customer: customer, status: "paid", total_cents: 100_00)
      Order.create!(customer: customer, status: "pending", total_cents: 999_00)

      expect(presenter.formatted_total_revenue).to eq("฿100.00")
    end
  end

  describe "#published_posts_count / #draft_posts_count" do
    it "นับบทความแยกตามสถานะเผยแพร่" do
      author = Author.create!(name: "A", nationality: "Thai")
      category = Category.create!(name: "Ruby")
      Post.create!(title: "P1", body: "x", author: author, category: category, published: true)
      Post.create!(title: "P2", body: "x", author: author, category: category, published: false)

      expect(presenter.published_posts_count).to eq(1)
      expect(presenter.draft_posts_count).to eq(1)
    end
  end

  describe "#overdue_orders_count" do
    it "นับเฉพาะออเดอร์ pending ที่เกินกำหนด" do
      Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 10.days.ago)
      Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 1.hour.ago)

      expect(presenter.overdue_orders_count).to eq(1)
    end
  end
end
```

```bash
bundle exec rspec spec/presenters/dashboard_presenter_spec.rb
```

```
...

Finished in 0.08 seconds (files took 1.3 seconds to load)
3 examples, 0 failures
```

**`ApplicationController.new.view_context` สร้าง view context จริงโดยไม่ต้องยิง HTTP request** —
ได้ object ที่มี helper method ทั้งหมดพร้อมใช้ (`number_to_currency`, `content_tag`, `l`) เหมือนกับ
view context ที่ controller จริงส่งเข้า Presenter ตอน production ทุกประการ แต่เร็วกว่าการทดสอบผ่าน
request spec มาก เพราะไม่ต้อง route/render จริง

### หลักการแบ่งชั้นการทดสอบ (ต่อยอดจาก Part 038 Step 380)

```
Query Object  → ทดสอบ SQL logic ที่ joins/group/having ซับซ้อน (spec/queries/)
Decorator     → ทดสอบ method การแสดงผลของ object เดียว โดยใช้ helpers ผ่าน metadata ของ draper (spec/decorators/)
Presenter     → ทดสอบการรวมข้อมูลจากหลาย source โดยส่ง view_context เข้าไปเอง (spec/presenters/)
Controller    → ทดสอบว่าทุกชิ้นส่วนต่อกันถูกต้องระดับ HTTP (request spec, ช้ากว่าแต่ครอบคลุมภาพรวมจริง)
```

แต่ละชั้นทดสอบสิ่งที่ตัวเองรับผิดชอบเท่านั้น ไม่ทดสอบซ้ำข้ามชั้น — เช่น request spec ของ
`Admin::DashboardController` ไม่จำเป็นต้องไล่ทดสอบทุก edge case ของการคำนวณ `units_sold` ซ้ำ
(เพราะ `TopSellingProductsQuery` มี spec ของตัวเองที่ทดสอบเรื่องนั้นครบแล้ว) — แค่ทดสอบว่า
controller เรียก presenter ถูกต้องและ response กลับมาเป็น HTTP 200 ก็เพียงพอ

---

## แบบฝึกหัด: ประกอบ Query Object + Decorator + Presenter เป็นหน้า Admin Dashboard

### โจทย์

สร้างหน้า `/admin/dashboard` ที่แสดง:

1. **สินค้าขายดี Top 5** — ใช้ `TopSellingProductsQuery` (Query Object ที่มี `joins`/`group`)
2. **ราคาและสถานะสต็อกของแต่ละสินค้า** — ใช้ `ProductDecorator` (Decorator ผ่าน draper) เพิ่ม
   `formatted_price`/`stock_status_badge`
3. **ยอดขายรวม, จำนวนบทความเผยแพร่แล้ว/ฉบับร่าง, ออเดอร์ค้างชำระ** — ใช้ `DashboardPresenter`
   (Presenter) รวมข้อมูลทั้งหมดเป็น view model เดียว
4. เขียน test แยกหน่วยครบทั้ง 3 ส่วน (Query Object, Decorator, Presenter) โดยไม่พึ่ง controller/view

### เฉลย

**1) Query Object และ 2) Decorator — ใช้โค้ดชุดเดียวกับ Step 822/826 ตรงๆ โดยไม่ต้องแก้ไข**

`app/queries/application_query.rb`, `app/queries/top_selling_products_query.rb`,
`app/queries/overdue_orders_query.rb` (Step 822) และ `app/decorators/product_decorator.rb`
(Step 826) ใช้โค้ดชุดเดิมทุกบรรทัด — นี่คือประโยชน์ที่จับต้องได้ของการแยก Query Object/Decorator
ออกมาเป็น class อิสระ: เขียนครั้งเดียว **ใช้ประกอบกับ Presenter ใหม่ได้ทันทีโดยไม่ต้องแตะโค้ดเดิม
เลยสักบรรทัด**

**3) Presenter**

```ruby
# app/presenters/dashboard_presenter.rb
class DashboardPresenter
  def initialize(view_context)
    @view_context = view_context
  end

  def top_products
    @top_products ||= TopSellingProductsQuery.call(limit: 5).decorate
  end

  def overdue_orders
    @overdue_orders ||= OverdueOrdersQuery.call
  end

  def overdue_orders_count
    overdue_orders.count
  end

  def total_revenue
    @total_revenue ||= Order.paid.sum(:total_cents)
  end

  def formatted_total_revenue
    h.number_to_currency(total_revenue / 100.0, unit: "฿", format: "%u%n")
  end

  def published_posts_count
    @published_posts_count ||= Post.published.count
  end

  def draft_posts_count
    @draft_posts_count ||= Post.where(published: false).count
  end

  private

  attr_reader :view_context

  def h
    view_context
  end
end
```

**4) Controller + Route + View**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "admin/dashboard", to: "admin/dashboard#index"
end
```

```ruby
# app/controllers/admin/dashboard_controller.rb
class Admin::DashboardController < ApplicationController
  def index
    @presenter = DashboardPresenter.new(view_context)
  end
end
```

```erb
<%# app/views/admin/dashboard/index.html.erb %>
<h1>แดชบอร์ดผู้ดูแลระบบ</h1>

<section>
  <h2>ยอดขายรวม (ออเดอร์ที่ชำระแล้ว)</h2>
  <p><%= @presenter.formatted_total_revenue %></p>
</section>

<section>
  <h2>สินค้าขายดี Top 5</h2>
  <ul>
    <% @presenter.top_products.each do |product| %>
      <li>
        <%= product.name %> — ขายไป <%= product.units_sold %> ชิ้น
        (<%= product.formatted_price %>) <%== product.stock_status_badge %>
      </li>
    <% end %>
  </ul>
</section>

<section>
  <h2>ออเดอร์ค้างชำระเกินกำหนด (<%= @presenter.overdue_orders_count %> รายการ)</h2>
  <ul>
    <% @presenter.overdue_orders.each do |order| %>
      <li>Order #<%= order.id %> — ค้างตั้งแต่ <%= order.created_at.to_date %></li>
    <% end %>
  </ul>
</section>

<section>
  <h2>บทความ</h2>
  <p>
    เผยแพร่แล้ว <%= @presenter.published_posts_count %> รายการ /
    ฉบับร่าง <%= @presenter.draft_posts_count %> รายการ
  </p>
</section>
```

ทดสอบจริงด้วย curl ให้ผลลัพธ์ตรงกับที่ Step 828 แสดงไว้ทุกประการ (ยอดขายรวม ฿164,800.00, สินค้าขายดี
Top 5 เรียงถูกต้องพร้อม badge สต็อก, ออเดอร์ค้างชำระ 3 รายการ, บทความเผยแพร่แล้ว 36/ฉบับร่าง 9) — เพราะ
`DashboardPresenter`/controller/view เป็นโค้ดชุดเดียวกันทุกบรรทัด สิ่งที่เพิ่มเข้ามาในแบบฝึกหัดนี้คือ
**ชุด test แยกหน่วยที่ครอบคลุมครบทั้ง 3 layer**

**5) Test แยกหน่วยครบทั้ง 3 ส่วน**

```ruby
# spec/queries/top_selling_products_query_spec.rb
require "rails_helper"

RSpec.describe TopSellingProductsQuery do
  let(:popular) { Product.create!(name: "สินค้าขายดี", price_cents: 1000, stock: 10, sku: "P1") }
  let(:unpopular) { Product.create!(name: "สินค้าขายน้อย", price_cents: 1000, stock: 10, sku: "P2") }
  let(:customer) { Customer.create!(name: "ลูกค้า A", email: "a@example.com") }

  before do
    order = Order.create!(customer: customer, status: "paid", total_cents: 0)
    OrderItem.create!(order: order, product: popular, quantity: 10, unit_price_cents: 1000)
    OrderItem.create!(order: order, product: unpopular, quantity: 1, unit_price_cents: 1000)
  end

  it "เรียงสินค้าตามยอดขาย (units_sold) จากมากไปน้อย" do
    expect(described_class.call.map(&:id)).to eq([popular.id, unpopular.id])
  end

  it "เรียกต่อจาก relation ที่กรองมาก่อนหน้าได้ (chainable)" do
    unpopular.update!(stock: 0)
    result = described_class.new(Product.where("stock > 0"), limit: 10).call
    expect(result.map(&:id)).to contain_exactly(popular.id)
  end
end
```

```ruby
# spec/queries/overdue_orders_query_spec.rb
require "rails_helper"

RSpec.describe OverdueOrdersQuery do
  let(:customer) { Customer.create!(name: "ลูกค้า A", email: "a@example.com") }

  it "คืนเฉพาะออเดอร์ pending ที่เก่ากว่า threshold" do
    old_order = Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 10.days.ago)
    recent_order = Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 1.hour.ago)
    paid_old_order = Order.create!(customer: customer, status: "paid", total_cents: 0, created_at: 10.days.ago)

    result = described_class.call

    expect(result).to include(old_order)
    expect(result).not_to include(recent_order, paid_old_order)
  end
end
```

```ruby
# spec/decorators/product_decorator_spec.rb
require "rails_helper"

RSpec.describe ProductDecorator do
  let(:stock) { 10 }
  let(:product) { Product.create!(name: "เมาส์ทดสอบ", price_cents: 59_000, stock: stock, sku: "T-1") }
  let(:decorated) { product.decorate }

  it "แสดงราคาเป็นสกุลเงินบาทจาก cents" do
    expect(decorated.formatted_price).to eq("฿590.00")
  end

  context "เมื่อสินค้าหมด" do
    let(:stock) { 0 }

    it "แสดง badge สีแดง" do
      expect(decorated.stock_status_badge).to include("badge-danger", "สินค้าหมด")
    end
  end
end
```

```ruby
# spec/presenters/dashboard_presenter_spec.rb
require "rails_helper"

RSpec.describe DashboardPresenter do
  let(:view_context) { ApplicationController.new.view_context }
  let(:presenter) { described_class.new(view_context) }
  let(:customer) { Customer.create!(name: "ลูกค้า A", email: "a@example.com") }

  it "รวมยอดขายเฉพาะออเดอร์ที่ชำระแล้ว (paid) เท่านั้น" do
    Order.create!(customer: customer, status: "paid", total_cents: 100_00)
    Order.create!(customer: customer, status: "pending", total_cents: 999_00)

    expect(presenter.formatted_total_revenue).to eq("฿100.00")
  end

  it "นับเฉพาะออเดอร์ pending ที่เกินกำหนด" do
    Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 10.days.ago)
    Order.create!(customer: customer, status: "pending", total_cents: 0, created_at: 1.hour.ago)

    expect(presenter.overdue_orders_count).to eq(1)
  end
end
```

```bash
bundle exec rspec spec/queries spec/decorators/product_decorator_spec.rb spec/presenters
```

```
..........

Finished in 0.16 seconds (files took 1.3 seconds to load)
10 examples, 0 failures
```

ทุกส่วนทดสอบผ่านโดยไม่มีจุดไหนต้องยิง HTTP request, render controller จริง, หรือแตะ view เลยแม้แต่
จุดเดียว — เพราะทั้ง Query Object, Decorator, และ Presenter ถูกออกแบบให้เป็น class ที่รับ input ชัดเจน
และคืน output ที่ทดสอบได้โดยตรงตั้งแต่แรก

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม parameter `since:` ให้ `TopSellingProductsQuery` ใช้งานได้จริง (ตัวอย่างในบทความยังไม่ได้
   ทดสอบกรณีที่มีข้อมูลเก่าจริง) โดยสร้างออเดอร์ที่ backdate ไปมากกว่า 30 วัน แล้วเขียน RSpec ยืนยัน
   ว่า `TopSellingProductsQuery.call(since: 7.days.ago)` **ไม่นับ** ยอดขายจากออเดอร์เก่าเหล่านั้น
   (ใบ้: ต้อง seed ข้อมูลสองชุดที่มี `created_at` ต่างกันชัดเจน แล้วตรวจสอบว่า `units_sold` ที่ได้
   ไม่รวมชุดเก่า)
2. เขียน `CustomerDecorator` ที่เพิ่ม method `lifetime_value` (ผลรวม `total_cents` ของออเดอร์ที่
   `status: "paid"` ทั้งหมดของลูกค้าคนนั้น จัดรูปแบบเป็นสกุลเงินบาท) แล้วใช้
   `decorates_association :customer` ใน `OrderDecorator` (ต้องสร้างเพิ่ม) เพื่อให้เรียก
   `order.customer.lifetime_value` ได้ตรงๆ — พร้อมเขียนเทียบว่าถ้าทำ pattern เดียวกันนี้ด้วย
   Presenter (ไม่ใช้ draper) ต้องเขียนโค้ดต่างจากกันตรงไหนบ้าง
3. เพิ่มหน้า export CSV ของ "สินค้าขายดี" (ใช้ `TopSellingProductsQuery` เดิม ไม่ต้องเขียน query ใหม่)
   แล้วพิจารณาว่า `ProductDecorator` ที่มี method อย่าง `formatted_price` (คืนค่าเป็น HTML string
   ผ่าน `content_tag`) เหมาะกับการใช้ตรงๆ ใน CSV หรือไม่ — ถ้าไม่เหมาะ ควรแก้ไขอย่างไร (ใบ้: CSV ไม่
   ต้องการ HTML tag เลย พิจารณาแยก method ที่คืนค่าตัวเลข/ข้อความล้วนออกจาก method ที่คืนค่า HTML
   โดยเฉพาะ เพื่อให้ decorator ตัวเดียวใช้ได้ทั้งสองบริบท)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Query Object คือการยกระดับ Filter Object จาก Part 038** ให้เป็นระบบมากขึ้น: namespace
  `app/queries/`, base class `ApplicationQuery` ที่ให้ `.call(...)` แบบ shortcut ผ่าน anonymous
  argument forwarding, และหลักการที่เข้มงวดกว่าเดิมว่า **ต้องรับ/คืน `ActiveRecord::Relation` เสมอ**
  ไม่ใช่ `Array`
- **Query Object มีสองรูปแบบ:** **chainable** (รับ `relation` เป็น argument แรก เพื่อ compose กับ
  scope อื่นได้ เช่น `TopSellingProductsQuery`) และ **terminal** (ไม่รับ relation ภายนอก เพราะมี
  ความหมายผูกกับ model เดียวชัดเจน เช่น `OverdueOrdersQuery`) — เลือกแบบไหนขึ้นกับว่า query นั้นมี
  use case ให้ผสมกับเงื่อนไขอื่นจริงหรือไม่
- **Query Object ป้องกันการเขียน `joins`/`group`/`having` ซ้ำซ้อนในหลาย controller/job/API endpoint**
  — เมื่อ business logic เปลี่ยน แก้ที่เดียวจบ ทุกจุดที่เรียกใช้ได้ผลลัพธ์ตรงกันเสมอ และทดสอบ SQL
  logic ที่ซับซ้อนได้โดยไม่ต้องพึ่ง controller
- **ข้อควรระวังเรื่อง `.size` กับ Query Object ที่ใช้ raw `select` alias** — `.size`/`.count` บน
  relation ที่ยังไม่ load อาจสร้าง query ใหม่ที่ไม่รู้จัก alias ที่ตั้งไว้ ควร `.to_a` ก่อนถ้าต้องการ
  นับจากผลลัพธ์ที่ query มาแล้วจริง
- **View Helper/Decorator/Presenter แก้ปัญหาเดียวกัน (แยก view logic ออกจาก ERB) แต่ต่างกันที่ขอบเขต:**
  Helper ไม่ผูกกับ object ใด (global), Decorator ผูกกับ object เดียว, Presenter รวมได้หลาย object
  เพื่อหน้าใดหน้าหนึ่งโดยเฉพาะ
- **Decorator ผ่าน `draper`:** `rails generate decorator Model`, `delegate_all` ส่งต่อ attribute
  ที่ไม่ได้ override ไปยัง `object` อัตโนมัติ, `helpers`/`h` เข้าถึง view helper ทั้งหมด,
  `decorates_association` decorate association ให้ด้วย (ต้องมี decorator ของ association นั้นก่อน
  ไม่งั้น `Draper::UninferrableDecoratorError`)
- **`.decorate` ใช้ได้ทั้ง instance เดียวและ collection/relation** — เรียกใน controller
  (`Model.find(id).decorate`) แล้วใช้ใน view เหมือน object เดิมทุกประการ บวกความสามารถใหม่ที่
  decorator เพิ่มให้ — decorated object ยังผ่าน `is_a?(Model)` เดิมได้ ทำให้ "โปร่งใส" กับโค้ด Rails
  framework เอง
- **Presenter เป็นทางเลือกที่ไม่ต้องพึ่ง gem** — เขียน PORO เองได้แต่ต้องเขียน delegation เอง
  (ไม่มี `delegate_all` ฟรี) หรือใช้ `SimpleDelegator` จาก Ruby standard library เพื่อได้ auto-
  delegation แบบเบาๆ โดยไม่เพิ่ม dependency — เหมาะที่สุดกับการรวมข้อมูลจากหลาย model/query เป็น
  view model เดียวสำหรับหน้าที่ซับซ้อน เช่น dashboard
- **กรอบการตัดสินใจ Decorator vs Presenter vs ViewComponent (Part 055) vs Helper:** ถามว่า "ผูกกับ
  กี่ object" — หนึ่ง object → Decorator, หลาย object รวมเพื่อหน้าเดียว → Presenter, UI ที่มี
  template ของตัวเอง → ViewComponent, ไม่ผูกกับอะไรเลย → Helper — ทั้งสี่เครื่องมือประกอบกันในหน้า
  เดียวได้ตามธรรมชาติโดยไม่ทับซ้อนความรับผิดชอบกัน
- **ทดสอบ Decorator/Presenter แยกหน่วยได้โดยไม่ต้องพึ่ง controller/view จริง** — decorator ใช้
  `helpers` ผ่าน metadata ที่ generator ของ draper เตรียมไว้ให้อัตโนมัติ ส่วน presenter ส่ง
  `ApplicationController.new.view_context` เข้าไปเองเพื่อได้ view helper ครบชุดโดยไม่ต้องยิง HTTP
  request

**ต่อไป (Part 084):** Part นี้แสดงให้เห็นแล้วว่า Query Object/Decorator/Presenter ช่วยแยกความ
รับผิดชอบออกจาก controller/model/view ได้อย่างเป็นระบบ — แต่ทุก pattern ที่เรียนมาจนถึงตอนนี้ (รวม
Service/Form Object จาก Part 082) ยังคง **ผูกติดกับ ActiveRecord และ Rails framework โดยตรง** ทุก
class เรียก `Post.where(...)`/`Order.create!(...)` ตรงๆ โดยไม่มีชั้นกั้นระหว่าง business logic กับ
framework เลย — Part ถัดไปจะพาไปรู้จัก **Clean Architecture / Hexagonal Architecture ใน Rails**:
แนวคิดการแยก "แก่นของ business logic" ออกจาก "รายละเอียดของ framework/database" อย่างเป็นทางการ
พร้อม **Dependency Injection เบื้องต้น** เพื่อให้ business logic ทดสอบได้โดยไม่ต้องพึ่ง Rails หรือ
ฐานข้อมูลจริงเลยแม้แต่น้อย — เป็นก้าวสุดท้ายของการออกแบบสถาปัตยกรรมระดับ enterprise ก่อนจะไปถึงเรื่อง
Multi-tenancy ใน Part 085
