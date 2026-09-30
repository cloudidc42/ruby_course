# Part 098: System Design สำหรับ Rails Application ขนาดใหญ่ (Scaling Checklist)

> **Step ครอบคลุมใน Part นี้:** Step 971–980

**ระดับ:** Senior/Mastery | **Rails version:** 7.x / 8.x | **Ruby version:** 3.2+

เมื่อ Rails application ของคุณเติบโต ปัญหาที่ไม่เคยเห็นในช่วง development จะเริ่มปรากฏขึ้น ตั้งแต่ database ช้า, memory เต็ม, ไปจนถึง server ล่มกลางดึก Part นี้จะให้ Scaling Checklist ที่ครอบคลุมทุกด้าน โดยอิงจากประสบการณ์จริงของ production Rails applications ที่มี traffic สูง พร้อม trade-offs ที่ต้องตัดสินใจในแต่ละขั้นตอน

---

## สารบัญ

- [Step 971: Scaling คืออะไร](#step-971)
- [Step 972: Database Scaling](#step-972)
- [Step 973: Caching Strategy ระดับ Production](#step-973)
- [Step 974: Background Job Architecture](#step-974)
- [Step 975: Search ขนาดใหญ่](#step-975)
- [Step 976: File Storage at Scale](#step-976)
- [Step 977: Multi-region Deployment](#step-977)
- [Step 978: Feature Flags สำหรับ Safe Deployment](#step-978)
- [Step 979: Observability ครบวงจร](#step-979)
- [Step 980: Incident Response](#step-980)
- [แบบฝึกหัด](#exercises)
- [สรุปสิ่งที่ได้เรียนรู้](#summary)

---

## Step 971: Scaling คืออะไร — vertical vs horizontal scaling, เมื่อไหรต้องเริ่มคิดเรื่อง scale {#step-971}

Scaling คือการทำให้ระบบรองรับ load ที่เพิ่มขึ้นได้ แต่ก่อนจะ scale ต้องรู้ว่า bottleneck อยู่ที่ไหน

### Vertical Scaling vs Horizontal Scaling

**Vertical Scaling (Scale Up):** เพิ่มทรัพยากรให้เครื่องเดิม

```
ปัจจุบัน: 1 server, 4 CPU cores, 8GB RAM
หลัง scale up: 1 server, 16 CPU cores, 64GB RAM

ข้อดี:
✅ ไม่ต้องเปลี่ยน architecture
✅ ไม่มีปัญหา distributed state
✅ เร็วที่สุดในการ implement

ข้อเสีย:
❌ มีขีดจำกัด (ไม่สามารถ scale ขึ้นไปเรื่อยๆ)
❌ Single point of failure
❌ Downtime เมื่อ upgrade hardware
❌ ราคาแพงขึ้นแบบ non-linear
```

**Horizontal Scaling (Scale Out):** เพิ่มจำนวน server

```
ปัจจุบัน: 1 server, 4 CPU cores
หลัง scale out: 5 servers, 4 CPU cores each + load balancer

ข้อดี:
✅ Scale ได้ไม่จำกัด (ทางทฤษฎี)
✅ High availability — เครื่องหนึ่งล่ม เครื่องอื่นรับแทน
✅ Cost efficient กว่าในระยะยาว

ข้อเสีย:
❌ ต้องออกแบบ stateless application
❌ Session management ซับซ้อนขึ้น
❌ Distributed systems problems (network partition, etc.)
```

### Rails และ Horizontal Scaling

Rails application scale ได้ดีในแนวนอน แต่ต้องระวัง:

```ruby
# ❌ State ที่ไม่ควรเก็บใน instance variable ของ server
class ApplicationController < ActionController::Base
  @@current_users_count = 0  # Shared state — ปัญหาใน multi-server!
  
  def track_user
    @@current_users_count += 1  # Race condition!
  end
end

# ✅ State ควรอยู่ใน shared store (Database หรือ Redis)
class ApplicationController < ActionController::Base
  def track_user
    Rails.cache.increment("active_users_count")
  end
end
```

### เมื่อไหรต้องเริ่มคิดเรื่อง Scale

```
เริ่มเมื่อพบสัญญาณเหล่านี้:
📊 Response time p95 > 500ms อย่างต่อเนื่อง
📊 CPU usage > 70% อย่างต่อเนื่อง  
📊 Memory usage > 80% อย่างต่อเนื่อง
📊 Database connection pool exhausted บ่อยครั้ง
📊 Error rate เพิ่มขึ้นเมื่อ traffic สูง

ไม่ควร scale ก่อนมีปัญหา (Premature optimization)
แต่ต้องมี capacity planning ล่วงหน้า 3-6 เดือน
```

### Amdahl's Law — ข้อจำกัดของการ Scale

```
ถ้า 80% ของงานทำ parallel ได้ และ 20% ทำ sequential
การเพิ่ม server จาก 1 เป็น ∞ จะเพิ่ม performance ได้สูงสุดแค่ 5x

→ ต้อง optimize serial bottleneck ก่อน
   (มักคือ database หรือ external API calls)
```

---

## Step 972: Database scaling — read replicas, connection pooling (PgBouncer), partitioning ตาราง {#step-972}

Database มักเป็น bottleneck แรกที่พบเมื่อ traffic เพิ่มขึ้น

### Read Replicas

Pattern พื้นฐานที่สุดสำหรับ database scaling:

```ruby
# config/database.yml
production:
  primary:
    adapter: postgresql
    database: myapp_production
    host: primary-db.example.com
    pool: 20
    
  primary_replica:
    adapter: postgresql
    database: myapp_production
    host: replica-db.example.com
    pool: 20
    replica: true  # ✅ บอก Rails ว่านี่คือ replica

# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  connects_to database: { 
    writing: :primary, 
    reading: :primary_replica 
  }
end
```

```ruby
# Controller: ใช้ replica สำหรับ read-heavy queries
class ReportsController < ApplicationController
  def index
    ActiveRecord::Base.connected_to(role: :reading) do
      @report = ExpensiveReportQuery.new.call
    end
  end
end

# หรือใช้ automatic switching
# config/application.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = 
  ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = 
  ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

**Replication Lag** คือปัญหาสำคัญ:

```ruby
# หลัง write, อย่า read จาก replica ทันที!
def create
  @order = Order.create!(order_params)
  # ❌ อาจได้ข้อมูลเก่า เพราะ replica ยังไม่ sync
  redirect_to @order
end

def show
  # ✅ Force read จาก primary หลัง recent write
  ActiveRecord::Base.connected_to(role: :writing) do
    @order = Order.find(params[:id])
  end
end
```

### Connection Pooling ด้วย PgBouncer

Rails application 10 instances × 20 connections = 200 connections ถึง PostgreSQL
PostgreSQL รองรับได้ แต่แต่ละ connection ใช้ memory (~5-10MB) → 200 × 7MB = 1.4GB เสียไปกับ connections!

PgBouncer แก้ปัญหานี้:

```
Rails App 1 (20 connections)  ─┐
Rails App 2 (20 connections)  ─┤
Rails App 3 (20 connections)  ─┤→ PgBouncer → PostgreSQL (25 connections จริงๆ)
Rails App 4 (20 connections)  ─┤
Rails App 5 (20 connections)  ─┘

200 connections ถึง PgBouncer → แค่ 25 connections ถึง PostgreSQL
```

```ini
# pgbouncer.ini
[databases]
myapp = host=primary-db.example.com dbname=myapp_production

[pgbouncer]
pool_mode = transaction   # transaction pooling: efficient สุด
max_client_conn = 1000    # Rails connections รวมกัน
default_pool_size = 25    # Connections จริงถึง PostgreSQL
server_idle_timeout = 600
```

```ruby
# config/database.yml — ชี้ไปที่ PgBouncer แทน PostgreSQL
production:
  adapter: postgresql
  host: pgbouncer.internal   # PgBouncer host
  port: 5432
  pool: 20
  
  # Transaction mode: ต้อง disable prepared statements
  prepared_statements: false
  advisory_locks: false
```

### Table Partitioning

สำหรับตารางที่มีข้อมูลมาก เช่น orders, events, logs:

```ruby
# Migration: สร้าง partitioned table
class CreatePartitionedOrders < ActiveRecord::Migration[7.1]
  def up
    # สร้าง parent table
    execute <<~SQL
      CREATE TABLE orders (
        id BIGSERIAL,
        user_id BIGINT NOT NULL,
        created_at TIMESTAMP NOT NULL,
        total_amount DECIMAL(10,2)
      ) PARTITION BY RANGE (created_at);
    SQL
    
    # สร้าง partitions ตาม quarter
    execute <<~SQL
      CREATE TABLE orders_2024_q1 
        PARTITION OF orders
        FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
        
      CREATE TABLE orders_2024_q2
        PARTITION OF orders
        FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
    SQL
    
    # Index ต้องสร้างบนแต่ละ partition
    execute <<~SQL
      CREATE INDEX ON orders_2024_q1 (user_id);
      CREATE INDEX ON orders_2024_q2 (user_id);
    SQL
  end
end
```

**ประโยชน์ของ partitioning:**
- Query ที่ filter ด้วย `created_at` จะแตะแค่ partition ที่ relevant
- Archiving เก่า: drop partition เก่าทั้งก้อนได้ทันที (เร็วกว่า DELETE มาก)
- Vacuum และ index rebuild ทำแค่ partition ที่ active

---

## Step 973: Caching strategy ระดับ production — CDN สำหรับ static asset, Cloudflare + Rails caching layer {#step-973}

Caching ที่ดีเป็นแบบ layered — แต่ละชั้นช่วยลด load ที่ชั้นถัดไป

### Caching Layers

```
Browser Cache
     ↓ (cache miss)
CDN (Cloudflare/CloudFront)
     ↓ (cache miss)
Rails HTTP Cache (ETag, Last-Modified)
     ↓ (cache miss)
Rails.cache (Redis)
     ↓ (cache miss)
Database Query Cache
     ↓ (cache miss)
Database
```

### Layer 1: CDN สำหรับ Static Assets

```ruby
# config/environments/production.rb
config.public_file_server.headers = {
  "Cache-Control" => "public, max-age=31536000, immutable"
}

# Asset pipeline: fingerprint ทุก asset เพื่อ cache busting
# image.abc123.png, application.def456.js
# → CDN cache ได้ 1 ปีเต็ม เพราะ fingerprint เปลี่ยนเมื่อ content เปลี่ยน
```

### Layer 2: Cloudflare สำหรับ Dynamic Content

```ruby
# ใช้ Surrogate-Control header สำหรับ Cloudflare
# (ซ่อนจาก browser แต่ Cloudflare อ่านได้)
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])
    response.headers["Surrogate-Control"] = "max-age=3600"
    response.headers["Surrogate-Key"] = "product-#{@product.id}"
  end
end

# Purge cache เมื่อ product อัปเดต
class Product < ApplicationRecord
  after_update :purge_cloudflare_cache
  
  private
  
  def purge_cloudflare_cache
    CloudflareCacheService.purge_by_tag("product-#{id}")
  end
end
```

### Layer 3: Rails HTTP Cache

```ruby
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])
    
    # ETag-based caching
    if stale?(@article)
      respond_to do |format|
        format.html
        format.json { render json: @article }
      end
    end
    # ถ้า stale? return false → Rails ส่ง 304 Not Modified โดยอัตโนมัติ
  end
  
  def index
    @articles = Article.published.order(created_at: :desc)
    
    # Collection cache
    if stale?(last_modified: @articles.maximum(:updated_at))
      render :index
    end
  end
end
```

### Layer 4: Rails.cache (Redis)

```ruby
# Fragment caching ใน view
<% cache @product do %>
  <%= render "product_details", product: @product %>
<% end %>

# Low-level caching สำหรับ expensive computation
def expensive_recommendation
  Rails.cache.fetch("recommendations/#{id}", expires_in: 1.hour) do
    # ML model call หรือ complex query
    RecommendationEngine.compute(self)
  end
end

# Russian Doll Caching — nested cache
<% cache ["v2", @order] do %>
  <% @order.items.each do |item| %>
    <% cache item do %>
      <%= render "order_item", item: item %>
    <% end %>
  <% end %>
<% end %>
```

### Cache Invalidation Strategy

```ruby
# Touch parent เมื่อ child เปลี่ยน
class OrderItem < ApplicationRecord
  belongs_to :order, touch: true  # อัปเดต order.updated_at เมื่อ item เปลี่ยน
end

# Cache key ที่ดี
class Product < ApplicationRecord
  def cache_key_with_version
    "products/#{id}-#{updated_at.to_i}"
    # เมื่อ product อัปเดต → updated_at เปลี่ยน → cache key เปลี่ยน → cache miss
  end
end
```

---

## Step 974: Background job architecture — queue prioritization, dead letter queue, idempotent jobs {#step-974}

Background jobs เป็นหัวใจของ Rails application ที่ต้องรองรับ load สูง

### Queue Prioritization

```ruby
# config/sidekiq.yml
:queues:
  - [critical, 10]    # Payment, authentication — ทำก่อน weight 10
  - [default, 5]      # Regular jobs — ทำปกติ
  - [mailers, 3]      # Email — รอได้หน่อย
  - [low, 1]          # Reports, analytics — รอได้นาน

# Job ที่ assign queue ที่เหมาะสม
class ChargePaymentJob < ApplicationJob
  queue_as :critical
  
  def perform(order_id)
    Order.find(order_id).charge!
  end
end

class WeeklyReportJob < ApplicationJob
  queue_as :low
  
  def perform(user_id)
    User.find(user_id).generate_weekly_report
  end
end
```

### Dead Letter Queue (DLQ)

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry 3 ครั้ง ก่อน discard
  retry_on StandardError, wait: :polynomially_longer, attempts: 3
  
  # Discard บาง error ที่ไม่ควร retry
  discard_on ActiveRecord::RecordNotFound
  
  # Job ที่ fail ทั้งหมด → ส่งไป DLQ
  around_perform do |job, block|
    block.call
  rescue => error
    # Log และส่งไป dead letter queue
    DeadLetterQueue.enqueue(
      job_class: job.class.name,
      job_id: job.job_id,
      arguments: job.arguments,
      error_class: error.class.name,
      error_message: error.message,
      failed_at: Time.current
    )
    raise  # Re-raise เพื่อให้ Sidekiq บันทึก failure
  end
end

# app/models/dead_letter_queue.rb
class DeadLetterQueue < ApplicationRecord
  # ทีม ops สามารถ review และ retry jobs ใน DLQ ได้
  def retry!
    job_class.constantize.perform_later(*JSON.parse(arguments))
    update!(retried_at: Time.current, status: "retried")
  end
end
```

### Idempotent Jobs

Job ต้องทำงานได้อย่างปลอดภัยแม้จะ run ซ้ำหลายครั้ง

```ruby
class SendOrderConfirmationJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    # ❌ ไม่ idempotent — ถ้า job run ซ้ำ จะส่ง email ซ้ำ
    OrderMailer.confirmation(order).deliver_now
  end
end

class SendOrderConfirmationJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    # ✅ Idempotent — ส่งแค่ครั้งเดียว
    return if order.confirmation_sent?
    
    OrderMailer.confirmation(order).deliver_now
    order.update!(confirmation_sent_at: Time.current)
  end
end

# ตัวอย่าง idempotent payment processing
class ProcessPaymentJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)
    
    # ใช้ idempotency key เพื่อป้องกัน double charge
    Stripe::PaymentIntent.create(
      amount: order.total_cents,
      currency: "thb",
      idempotency_key: "order-#{order.id}-#{order.updated_at.to_i}"
    )
  end
end
```

### Job Monitoring

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.on(:startup) do
    Rails.logger.info "Sidekiq started with #{Sidekiq.options[:concurrency]} workers"
  end
end

# Dashboard สำหรับ monitoring (ใน production ต้อง protect ด้วย auth)
# config/routes.rb
require "sidekiq/web"
mount Sidekiq::Web => "/sidekiq", constraints: AdminConstraint.new
```

---

## Step 975: Search ขนาดใหญ่ — Elasticsearch cluster vs managed service (Elastic Cloud / OpenSearch) {#step-975}

PostgreSQL full-text search ได้ถึงหมื่น records แต่พอข้อมูลถึงล้าน record ต้องใช้ dedicated search engine

### เมื่อไหรควรเปลี่ยนจาก PostgreSQL ไป Elasticsearch

```sql
-- PostgreSQL full-text search ยังได้สำหรับ simple use case
SELECT * FROM products 
WHERE to_tsvector('english', name || ' ' || description) 
@@ plainto_tsquery('ruby book');

-- แต่ไม่รองรับ:
-- ✗ Typo tolerance (fuzzy search)
-- ✗ Relevance scoring ที่ซับซ้อน  
-- ✗ Faceted search (filter by category + price range พร้อมกัน)
-- ✗ Auto-complete ที่เร็ว
-- ✗ Multi-language support
```

### Elasticsearch กับ Rails

```ruby
# Gemfile
gem "elasticsearch-model"
gem "elasticsearch-rails"

# app/models/product.rb
class Product < ApplicationRecord
  include Elasticsearch::Model
  include Elasticsearch::Model::Callbacks  # Auto-index เมื่อ model เปลี่ยน
  
  # กำหนด index mapping
  settings index: { number_of_shards: 3 } do
    mappings dynamic: false do
      indexes :name, type: :text, analyzer: :thai
      indexes :description, type: :text, analyzer: :thai
      indexes :price, type: :float
      indexes :category_id, type: :keyword
      indexes :in_stock, type: :boolean
    end
  end
  
  # Custom as_indexed_json
  def as_indexed_json(options = {})
    as_json(only: [:name, :description, :price, :category_id, :in_stock])
  end
end

# Reindex ทั้งหมด (ใน background job)
class ReindexProductsJob < ApplicationJob
  def perform
    Product.import(force: true, refresh: true)
  end
end
```

```ruby
# Search Service
class ProductSearchService
  def initialize(query:, filters: {}, page: 1, per_page: 20)
    @query = query
    @filters = filters
    @page = page
    @per_page = per_page
  end
  
  def call
    Product.search(build_query)
  end
  
  private
  
  def build_query
    {
      query: {
        bool: {
          must: [
            {
              multi_match: {
                query: @query,
                fields: ["name^3", "description"],  # name สำคัญกว่า 3x
                fuzziness: "AUTO"  # Typo tolerance
              }
            }
          ],
          filter: build_filters
        }
      },
      aggs: {
        categories: {
          terms: { field: "category_id" }
        },
        price_ranges: {
          range: {
            field: "price",
            ranges: [
              { to: 100 },
              { from: 100, to: 500 },
              { from: 500 }
            ]
          }
        }
      },
      from: (@page - 1) * @per_page,
      size: @per_page
    }
  end
  
  def build_filters
    filters = [{ term: { in_stock: true } }]
    filters << { term: { category_id: @filters[:category_id] } } if @filters[:category_id]
    filters
  end
end
```

### Managed Service: Elastic Cloud vs Self-hosted

```
Self-hosted Elasticsearch:
✅ ราคาถูกกว่า
✅ Control เต็มที่
❌ ต้อง manage เอง (backup, upgrade, monitoring)
❌ ต้องมี DevOps expertise

Elastic Cloud:
✅ Managed service — ไม่ต้อง manage infra
✅ Auto-scaling
✅ Integrated monitoring
❌ ราคาสูงกว่า (~$300+/เดือนสำหรับ production cluster)

AWS OpenSearch (managed):
✅ integrate กับ AWS ได้ดี
✅ ถูกกว่า Elastic Cloud เล็กน้อย
❌ Version lag จาก upstream Elasticsearch
```

---

## Step 976: File storage at scale — S3 + CloudFront, direct upload, signed URL {#step-976}

การเก็บไฟล์บน server disk ไม่ scalable และไม่ safe

### ทำไมต้องใช้ S3?

```
Local disk:
❌ ถ้า server ล่ม ไฟล์หายหมด
❌ Horizontal scaling: server ใหม่ไม่มีไฟล์
❌ ขนาดจำกัดตาม disk size
❌ Bandwidth จาก application server แพง

S3:
✅ Durability 99.999999999% (11 nines)
✅ Shared storage ข้ามทุก server
✅ ขนาดไม่จำกัดในทางปฏิบัติ
✅ Cheaper bandwidth ผ่าน CloudFront
```

### Active Storage กับ S3

```ruby
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1  # Singapore
  bucket: myapp-production-files
  upload:
    server_side_encryption: "AES256"

# config/environments/production.rb
config.active_storage.service = :amazon

# Model
class User < ApplicationRecord
  has_one_attached :avatar
  has_many_attached :documents
  
  # Variant สำหรับ image processing
  def avatar_thumbnail
    avatar.variant(resize_to_fill: [200, 200], format: :webp)
  end
end
```

### Direct Upload — ลด load บน Rails server

```
ไม่ใช้ Direct Upload:
Browser → Rails Server (อัปโหลดไฟล์) → S3
ปัญหา: Rails server ต้องรับ bandwidth ทั้งหมด
        memory spike ระหว่าง upload

Direct Upload:
Browser → Rails Server (ขอ presigned URL)
Browser → S3 (อัปโหลดตรง)  ← ไม่ผ่าน Rails เลย!
Browser → Rails Server (บอกว่าอัปโหลดเสร็จแล้ว)
```

```javascript
// app/javascript/controllers/file_upload_controller.js
import { Controller } from "@hotwired/stimulus"
import { DirectUpload } from "@rails/activestorage"

export default class extends Controller {
  static targets = ["input", "progress"]
  
  upload(event) {
    const file = event.target.files[0]
    const upload = new DirectUpload(
      file,
      "/rails/active_storage/direct_uploads",
      this  // callback object
    )
    
    upload.create((error, blob) => {
      if (error) {
        console.error(error)
      } else {
        // สร้าง hidden input ที่มี blob.signed_id
        const hiddenField = document.createElement("input")
        hiddenField.type = "hidden"
        hiddenField.name = this.inputTarget.name
        hiddenField.value = blob.signed_id
        this.element.appendChild(hiddenField)
      }
    })
  }
  
  // Progress callback
  directUploadWillStoreFileWithXHR(request) {
    request.upload.addEventListener("progress", event => {
      const progress = event.loaded / event.total * 100
      this.progressTarget.value = progress
    })
  }
}
```

### Signed URL สำหรับ Private Files

```ruby
# app/controllers/documents_controller.rb
class DocumentsController < ApplicationController
  before_action :authenticate_user!
  
  def show
    @document = Document.find(params[:id])
    authorize @document  # Pundit authorization
    
    # สร้าง signed URL ที่ expire ใน 5 นาที
    # ผู้ใช้ไม่สามารถแชร์ URL นี้ให้คนอื่นได้นาน
    redirect_to @document.file.url(expires_in: 5.minutes),
                allow_other_host: true
  end
end

# สำหรับ inline display (เช่น PDF)
def inline_url
  Rails.application.routes.url_helpers.rails_blob_url(
    document.file,
    disposition: "inline",
    expires_in: 10.minutes
  )
end
```

### CloudFront CDN สำหรับ Public Files

```ruby
# config/storage.yml — เพิ่ม CloudFront settings
amazon:
  service: S3
  region: ap-southeast-1
  bucket: myapp-production
  # ใช้ CloudFront URL แทน S3 URL โดยตรง
  public: true

# config/environments/production.rb
config.active_storage.resolve_model_to_route = :rails_storage_proxy
# หรือ ถ้าใช้ public files ผ่าน CDN
config.active_storage.url_options = { host: "https://cdn.example.com" }
```

---

## Step 977: Multi-region deployment — latency considerations, database replication lag {#step-977}

เมื่อผู้ใช้อยู่ทั่วโลก การ deploy แค่ region เดียวทำให้ latency สูงสำหรับผู้ใช้ที่อยู่ไกล

### Latency คือปัญหาหลัก

```
การส่ง request จาก Browser ถึง Server:
- ไทย → Singapore: ~20ms
- ไทย → US East: ~200ms
- ไทย → Europe: ~250ms

สำหรับ API ที่มี 5 roundtrips:
- Singapore: 5 × 20ms = 100ms total latency
- US East: 5 × 200ms = 1000ms total latency = 1 วินาที!
```

### Multi-region Architecture สำหรับ Rails

```
Global Architecture:

Users (Thailand) → CDN → App Servers (Singapore) → Primary DB (Singapore)
Users (Europe)   → CDN → App Servers (Frankfurt) ─┘ (ผ่าน DB replication)
Users (US)       → CDN → App Servers (Virginia) ──┘

Primary DB: Singapore (write)
Read Replica 1: Frankfurt (read สำหรับ European users)
Read Replica 2: Virginia (read สำหรับ US users)
```

### Database Replication Lag

```ruby
# Replication lag คือความล่าช้าระหว่าง write ที่ primary และ read ที่ replica
# ปกติอยู่ที่ 10-100ms สำหรับ cross-region

# ปัญหา: User สร้าง post → redirect to show page → ไม่เห็น post ที่เพิ่งสร้าง!
class PostsController < ApplicationController
  def create
    @post = Post.create!(post_params)
    # session บอก Rails ว่า user เพิ่ง write
    session[:last_write_at] = Time.current.to_i
    redirect_to @post
  end
  
  def show
    # ถ้า write เมื่อกี้ (<5 วินาที) → อ่านจาก primary
    if (Time.current.to_i - session[:last_write_at].to_i) < 5
      ActiveRecord::Base.connected_to(role: :writing) do
        @post = Post.find(params[:id])
      end
    else
      @post = Post.find(params[:id])  # อ่านจาก replica ได้
    end
  end
end
```

### Fly.io สำหรับ Multi-region Rails

```toml
# fly.toml
app = "myapp"
primary_region = "sin"  # Singapore

[[regions]]
  name = "sin"  # Singapore
  
[[regions]]
  name = "fra"  # Frankfurt

[env]
  PRIMARY_REGION = "sin"
```

```ruby
# config/database.yml
production:
  primary:
    adapter: postgresql
    host: <%= ENV["PRIMARY_DATABASE_URL"] %>
    
  primary_replica:
    adapter: postgresql
    # ใช้ nearest replica
    host: <%= ENV["REPLICA_DATABASE_URL"] || ENV["PRIMARY_DATABASE_URL"] %>
    replica: true
```

### CDN-first Architecture

```
ลด latency โดยไม่ต้อง multi-region app servers:
- Static assets → CloudFront (global edge)
- API responses → Cloudflare Cache (edge caching)
- App server → Singapore เพียงที่เดียว แต่ CDN ครอบ

ผล:
- Static: ทุก region ≈ 5ms (จาก CDN edge)
- Cached API: ทุก region ≈ 10ms (จาก CDN edge)
- Uncached API: Singapore 20ms, Frankfurt 250ms

เหมาะสำหรับ app ที่ 80%+ ของ requests เป็น cacheable
```

---

## Step 978: Feature flags สำหรับ safe deployment — `Flipper` gem, progressive rollout, A/B test {#step-978}

Feature flags ทำให้ deploy กับ release เป็นคนละสิ่งกัน

### ทำไมต้อง Feature Flags?

```
ปัญหาของการ deploy ทุก feature พร้อมกัน:
- ถ้า feature ใหม่มีบัก → ต้อง rollback ทั้ง deploy
- ไม่สามารถทดสอบกับ real users ได้บางส่วน
- ไม่สามารถ disable feature รวดเร็วโดยไม่ต้อง deploy

Feature Flags แก้ปัญหาด้วย:
- Deploy code ไปแล้ว แต่ยัง "ปิด" feature ไว้
- เปิด feature ให้ 1% ก่อน → 10% → 50% → 100%
- ถ้ามีปัญหา → ปิด flag ทันทีโดยไม่ต้อง deploy
```

### Flipper Gem

```ruby
# Gemfile
gem "flipper"
gem "flipper-active_record"  # เก็บ flags ใน database
gem "flipper-ui"             # Web UI สำหรับ manage flags

# db/migrate/xxx_create_flipper_tables.rb
class CreateFlipperTables < ActiveRecord::Migration[7.1]
  def change
    create_table :flipper_features do |t|
      t.string :key, null: false
      t.timestamps null: false
    end
    add_index :flipper_features, :key, unique: true
    
    create_table :flipper_gates do |t|
      t.references :feature, null: false
      t.string :key, null: false
      t.string :value
      t.timestamps null: false
    end
  end
end
```

```ruby
# config/initializers/flipper.rb
require "flipper"
require "flipper/adapters/active_record"

Flipper.configure do |config|
  config.adapter { Flipper::Adapters::ActiveRecord.new }
end

# ป้องกัน N+1 ด้วย caching
Flipper.configure do |config|
  config.adapter do
    Flipper::Adapters::ActiveRecordCached.new(
      Flipper::Adapters::ActiveRecord.new,
      expires_in: 30.seconds
    )
  end
end
```

```ruby
# การใช้งาน Feature Flags

# ✅ Enable สำหรับ specific user
Flipper.enable(:new_checkout, current_user)

# ✅ Enable สำหรับ % ของ users (progressive rollout)
Flipper.enable_percentage_of_actors(:new_dashboard, 10)  # 10%

# ✅ Enable สำหรับ group
Flipper.register(:beta_users) do |actor|
  actor.respond_to?(:beta?) && actor.beta?
end
Flipper.enable_group(:new_feature, :beta_users)

# ✅ Enable ทั้งหมด (เมื่อ rollout เสร็จ)
Flipper.enable(:new_checkout)

# ✅ Disable ทันที (emergency)
Flipper.disable(:problematic_feature)
```

```ruby
# Controller
class CheckoutsController < ApplicationController
  def new
    if Flipper.enabled?(:new_checkout_v2, current_user)
      render :new_v2
    else
      render :new
    end
  end
end

# View
<% if Flipper.enabled?(:new_dashboard_widget, current_user) %>
  <%= render "new_widget" %>
<% else %>
  <%= render "old_widget" %>
<% end %>
```

### A/B Testing ด้วย Feature Flags

```ruby
# เปิด 50% A, 50% B
Flipper.enable_percentage_of_actors(:checkout_flow_experiment, 50)

class CheckoutsController < ApplicationController
  def new
    @variant = Flipper.enabled?(:checkout_flow_experiment, current_user) ? "B" : "A"
    
    # Track variant ใน analytics
    Analytics.track(current_user, "Checkout Viewed", variant: @variant)
    
    render "new_#{@variant.downcase}"
  end
end

# วัดผล: compare conversion rate ของ A vs B
```

### Flipper UI (Admin Panel)

```ruby
# config/routes.rb
require "flipper/ui"

Rails.application.routes.draw do
  mount Flipper::UI.app(Flipper) => "/admin/features",
        constraints: AdminConstraint.new
end
```

---

## Step 979: Observability ครบวงจร — metrics (Prometheus/Datadog), traces (OpenTelemetry), logs (structured JSON) {#step-979}

Observability คือความสามารถในการเข้าใจว่า system กำลังทำอะไรอยู่ โดยดูจาก output ของ system นั้น

### สามเสาหลักของ Observability

```
Metrics: ตัวเลขที่วัดได้ตามเวลา
→ "Request rate 500/sec, p95 latency 250ms, error rate 0.1%"

Traces: การติดตาม request หนึ่งตั้งแต่ต้นจนจบ
→ "Request ใช้เวลา 250ms: 10ms routing, 5ms auth, 200ms DB query, 35ms rendering"

Logs: บันทึกเหตุการณ์แบบ text
→ "2024-01-15 10:30:00 ERROR PaymentService charge failed: Card declined"
```

### Structured JSON Logging

```ruby
# config/initializers/lograge.rb
# Lograge: เปลี่ยน Rails default log เป็น single-line JSON
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new
  
  config.lograge.custom_options = lambda do |event|
    {
      request_id: event.payload[:headers]["X-Request-Id"],
      user_id: event.payload[:user_id],
      tenant_id: event.payload[:tenant_id]
    }
  end
end

# Output:
# {"method":"POST","path":"/orders","status":201,"duration":45.2,"user_id":123}
```

```ruby
# Service ที่ log อย่างถูกต้อง
class PaymentService
  def charge(order)
    logger.info({
      event: "payment_attempted",
      order_id: order.id,
      amount: order.total_cents,
      currency: "THB"
    }.to_json)
    
    result = Stripe::Charge.create(...)
    
    logger.info({
      event: "payment_succeeded",
      order_id: order.id,
      charge_id: result.id
    }.to_json)
    
    result
  rescue Stripe::CardError => e
    logger.warn({
      event: "payment_failed",
      order_id: order.id,
      error_code: e.code,
      error_message: e.message
    }.to_json)
    raise
  end
end
```

### OpenTelemetry สำหรับ Distributed Tracing

```ruby
# Gemfile
gem "opentelemetry-sdk"
gem "opentelemetry-exporter-otlp"
gem "opentelemetry-instrumentation-rails"
gem "opentelemetry-instrumentation-active_record"

# config/initializers/opentelemetry.rb
require "opentelemetry/sdk"
require "opentelemetry/exporter/otlp"
require "opentelemetry/instrumentation/rails"
require "opentelemetry/instrumentation/active_record"

OpenTelemetry::SDK.configure do |c|
  c.service_name = "myapp"
  c.service_version = ENV["GIT_SHA"] || "unknown"
  
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV["OTEL_EXPORTER_OTLP_ENDPOINT"]
      )
    )
  )
  
  # Auto-instrument Rails, ActiveRecord, Sidekiq, etc.
  c.use_all
end
```

```ruby
# Custom spans สำหรับ business logic
class RecommendationService
  def call(user)
    tracer = OpenTelemetry.tracer_provider.tracer("recommendation-service")
    
    tracer.in_span("calculate_recommendations") do |span|
      span.set_attribute("user.id", user.id)
      span.set_attribute("user.plan", user.plan)
      
      recommendations = compute_recommendations(user)
      span.set_attribute("recommendations.count", recommendations.size)
      
      recommendations
    end
  end
end
```

### Metrics ด้วย Prometheus

```ruby
# Gemfile
gem "prometheus-client"

# config/initializers/prometheus.rb
require "prometheus/client"

METRICS = {
  http_requests: Prometheus::Client::Counter.new(
    :http_requests_total,
    docstring: "Total HTTP requests",
    labels: [:method, :path, :status]
  ),
  
  payment_processing_duration: Prometheus::Client::Histogram.new(
    :payment_processing_seconds,
    docstring: "Payment processing duration",
    buckets: [0.1, 0.5, 1.0, 2.5, 5.0, 10.0]
  )
}

Prometheus::Client.registry.register(METRICS[:http_requests])
Prometheus::Client.registry.register(METRICS[:payment_processing_duration])

# Middleware
# config/application.rb
config.middleware.use Prometheus::Client::Rack::Exporter
config.middleware.use Prometheus::Client::Rack::Collector
```

### Datadog สำหรับ APM (ถ้าใช้ managed service)

```ruby
# Gemfile
gem "ddtrace"

# config/initializers/datadog.rb
Datadog.configure do |c|
  c.service = "myapp"
  c.env = Rails.env
  c.version = ENV["GIT_SHA"]
  
  c.tracing.instrument :rails
  c.tracing.instrument :active_record, service_name: "myapp-db"
  c.tracing.instrument :sidekiq
  c.tracing.instrument :redis
  
  # Error tracking
  c.tracing.report_hostname = true
end
```

---

## Step 980: Incident Response — runbook, on-call rotation, post-mortem culture {#step-980}

สิ่งที่แยก Senior Engineer จาก Junior ในยามวิกฤตคือความสงบและกระบวนการที่ชัดเจน

### Runbook คืออะไร?

Runbook คือเอกสาร step-by-step สำหรับแก้ปัญหาที่รู้ว่าจะเกิดซ้ำ

```markdown
# Runbook: High Database CPU Usage

## ตรวจจับ
Alert: database_cpu > 80% for 5 minutes
Dashboard: https://monitoring.example.com/db-overview

## ขั้นตอนการแก้ปัญหา

### Step 1: ระบุ Query ที่ใช้ CPU สูง
\`\`\`sql
SELECT pid, query, state, wait_event_type, wait_event,
       now() - pg_stat_activity.query_start AS duration
FROM pg_stat_activity
WHERE query != '<IDLE>' 
  AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY duration DESC
LIMIT 20;
\`\`\`

### Step 2: Kill long-running query (ถ้าจำเป็น)
\`\`\`sql
-- ดู queries ที่ run > 5 นาที
SELECT pg_terminate_backend(pid), query
FROM pg_stat_activity
WHERE duration > interval '5 minutes'
  AND state = 'active';
\`\`\`

### Step 3: Check for Lock Contention
\`\`\`sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_locks blocking_locks 
  ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocked_activity 
  ON blocked_activity.pid = blocked_locks.pid
WHERE NOT blocked_locks.granted;
\`\`\`

### Step 4: Scale Up Connection Pool ชั่วคราว
\`\`\`bash
# ใน PgBouncer config
heroku config:set PGBOUNCER_POOL_SIZE=50
\`\`\`

### Step 5: เปิด read replica สำหรับ heavy queries
\`\`\`ruby
# app/models/application_record.rb - เปลี่ยน temporarily
connects_to database: { writing: :primary, reading: :primary_replica }
\`\`\`

## Escalation Path
- 0-10 นาที: on-call engineer แก้เอง
- 10-30 นาที: escalate ไป Tech Lead
- 30+ นาที: escalate ไป Engineering Manager + CEO notification

## Recovery Verification
- CPU < 50% for 5 minutes
- Error rate < 0.1%
- p95 latency < 500ms

## Root Cause Investigation
หลัง incident: ดู slow_query_log และสร้าง indexes ที่จำเป็น
```

### On-call Rotation

```
หลักการ On-call ที่ดี:
- Rotation สม่ำเสมอ — ทุกคนต้องมีส่วนร่วม
- Primary + Secondary on-call เสมอ
- Escalation path ชัดเจน
- Alert ที่ actionable — ไม่ alert เพราะเหตุผล noisy

เครื่องมือ:
- PagerDuty: ส่ง alert ไป phone call/SMS
- OpsGenie: คล้าย PagerDuty
- Incident.io: incident management
```

### Incident Severity Levels

```
SEV-1 (P0): Production down, ทุก user ได้รับผลกระทบ
→ ตอบสนองทันที, wake up on-call เลย
→ Status page อัปเดต, CEO informed

SEV-2 (P1): Partial outage, บาง feature ไม่ทำงาน  
→ ตอบสนองภายใน 15 นาที
→ Status page อัปเดต

SEV-3 (P2): Degraded performance, ไม่มี data loss
→ ตอบสนองภายใน 1 ชั่วโมง
→ Ticket สร้าง, แก้ใน business hours

SEV-4 (P3): Minor issue
→ แก้ใน sprint ถัดไป
```

### Post-mortem Culture (Blameless)

```markdown
# Post-mortem: Payment Service Outage (2024-01-15)

**Duration:** 45 minutes (14:30-15:15 UTC+7)
**Impact:** ~2,000 users ไม่สามารถ checkout ได้
**Severity:** SEV-2

## Timeline
14:30 - Alert: error_rate > 5% บน payment endpoints
14:32 - On-call (สมชาย) acknowledge alert
14:35 - ระบุว่า Stripe webhook handler throwing 500 errors
14:40 - Deploy hotfix: rescue StripeSignatureVerificationError
14:45 - Error rate ลดลงเป็น 0.1%
15:15 - Confirmed stable, post-mortem scheduled

## Root Cause
Stripe อัปเดต library version → signature verification algorithm เปลี่ยน
เราไม่ได้ pin Stripe gem version → auto-updated ใน deployment เมื่อวาน

## What Went Well
✅ Alert ทำงานได้เร็ว (< 2 นาที)
✅ On-call ตอบสนองเร็ว
✅ Hotfix deploy ได้ภายใน 10 นาที

## What Went Wrong
❌ ไม่ได้ pin gem versions ใน Gemfile.lock committed
❌ ไม่มี smoke test บน payment flow หลัง deployment
❌ Stripe changelog ไม่ได้อยู่ใน deployment checklist

## Action Items
□ [Dev Lead] Pin Stripe gem: gem "stripe", "~> 9.x"
□ [QA] เพิ่ม post-deploy smoke test สำหรับ payment flow
□ [DevOps] เพิ่ม "review Stripe changelog" ใน deployment runbook
□ [Dev Lead] Set up Dependabot กับ manual approval สำหรับ critical gems

## บทเรียน (ไม่โทษคน!)
"ปัญหานี้เกิดจาก process ที่ขาด — ไม่มีใครผิด"
การ pin gem versions เป็น best practice ที่เราควรมีตั้งนานแล้ว
```

---

## แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Database Optimization

ในโปรเจกต์ Rails ที่มีตาราง `orders` จำนวน 10 ล้านแถว ให้:
1. เพิ่ม read replica ใน `database.yml`
2. ระบุ 3 indexes ที่ควรมีสำหรับ query ทั่วไป
3. เขียน migration สำหรับ partition ตาราง orders ตาม year

### แบบฝึกหัดที่ 2: Caching Layer

ออกแบบ caching strategy สำหรับ product listing page ที่:
- แสดง 20 products พร้อม reviews count
- Filter ได้ด้วย category และ price range
- Sort ได้ด้วย popularity หรือ newest

### แบบฝึกหัดที่ 3: Feature Flag Implementation

สร้าง feature flag "new_search_ui" ที่:
- เปิดให้ beta users ดูก่อน
- Progressive rollout 5% → 25% → 100%
- สามารถ disable ได้ทันที

### แบบฝึกหัดที่ 4: เขียน Runbook

เขียน runbook สำหรับกรณี "High Memory Usage บน Rails Application Server" รวมถึง step การ diagnose, คำสั่งที่ใช้, และ escalation path

### แบบฝึกหัดที่ 5: Observability Setup

Setup structured logging สำหรับ Rails application ด้วย Lograge ที่รวม:
- Request ID
- User ID
- Duration
- SQL query count
- Memory usage

---

## สรุปสิ่งที่ได้เรียนรู้ {#summary}

ใน Part 098 นี้เราได้เรียนรู้ Scaling Checklist ที่ครอบคลุม:

| หัวข้อ | สิ่งสำคัญ | เมื่อไหรนำไปใช้ |
|--------|-----------|----------------|
| Vertical vs Horizontal Scaling | เพิ่ม resource vs เพิ่มจำนวน server | เมื่อ server metrics เกิน threshold |
| Database: Read Replicas | แยก read/write traffic | เมื่อ DB read หนักกว่า write |
| Database: PgBouncer | Connection pooling | เมื่อ connection pool exhausted |
| Database: Partitioning | แบ่ง table ขนาดใหญ่ | ตาราง > 100M rows |
| CDN | Cache static assets ที่ edge | ตั้งแต่วันแรก |
| Rails.cache | In-memory/Redis cache | เมื่อ query/computation ซ้ำบ่อย |
| Background Jobs: Priority Queues | แยก critical vs low-priority | เมื่อมี job หลายประเภท |
| Background Jobs: DLQ + Idempotency | จัดการ failed jobs ปลอดภัย | งาน payment/critical operations |
| Elasticsearch | Full-text search ขนาดใหญ่ | เมื่อ PostgreSQL search ช้า |
| S3 + Direct Upload | File storage ที่ scalable | ตั้งแต่วันแรก (ไม่ใช้ local disk) |
| Multi-region | Reduce latency ทั่วโลก | เมื่อ users อยู่หลาย region |
| Feature Flags | Safe deployment | Feature ใหญ่ + progressive rollout |
| Observability: Metrics/Traces/Logs | เข้าใจระบบในเชิงลึก | ตั้งแต่ go production |
| Incident Response | Runbook + Post-mortem | เมื่อมี incident ครั้งแรก |

**Key Takeaway:** Scale ไม่ได้เริ่มจากการซื้อ server ใหญ่ขึ้น แต่เริ่มจากการเข้าใจว่า bottleneck อยู่ที่ไหน และแก้จุดนั้นก่อน ระบบที่ดีต้องมี observability ครบถ้วน เพื่อให้รู้ว่าต้อง scale ตรงไหน

**ต่อไป:** Part 099 จะเตรียมคุณสำหรับการสัมภาษณ์งาน Senior/Staff Rails Engineer ด้วย live coding practice, system design interview, และกลยุทธ์การต่อรอง salary
