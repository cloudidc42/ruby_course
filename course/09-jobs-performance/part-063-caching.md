# Part 063: Caching — Fragment Cache, Russian Doll Caching, Low-level Caching

> **Step ครอบคลุมใน Part นี้:** Step 621–630
> **ระดับ:** กลาง (ต่อจาก Part 061–062 เรื่อง ActiveJob และ Sidekiq)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x — cache store หลักที่ใช้สาธิตคือ **solid_cache**
> (ค่า default ของ Rails 8 บน production) ทุกตัวอย่างทดสอบรันจริงด้วยแอป Rails ที่ generate ใหม่
> บน Rails 8.1.4

ใน Part 034 เราเรียนเรื่อง N+1 query และการ `includes` เพื่อลดจำนวน query ไปแล้ว ใน Part 035
เราเจอ `touch: true` และถูกทิ้งท้ายไว้ว่า "เรื่อง caching แบบเต็มรูปแบบจะสอนลึกกว่านี้มากใน
**Part 063**" — นี่คือ Part นั้น เราจะมาดูว่าเมื่อ query เร็วขึ้นแล้ว แต่ถ้าหน้าเว็บเดิมถูกเรียก
ซ้ำๆ ด้วยข้อมูลที่ไม่เปลี่ยนบ่อย เราจะ**ข้ามการคำนวณซ้ำไปเลย**ได้อย่างไร ตั้งแต่ caching ระดับ
ชิ้นส่วนของหน้าเว็บ (fragment caching), การซ้อน cache หลายชั้นแบบตุ๊กตารัสเซีย (Russian Doll
Caching), การ cache ผลลัพธ์การคำนวณ/เรียก API ภายนอกด้วย `Rails.cache` (low-level caching),
ไปจนถึง caching อีกชั้นที่อยู่นอก Rails process เลยคือ HTTP caching ผ่าน ETag/`Cache-Control`

## สารบัญของ Part นี้

- Step 621: ทำไมต้อง Caching และ Cache Store ที่ Rails รองรับ (solid_cache, memory, file, redis, memcached)
- Step 622: เปิดใช้ Caching ใน Development ด้วย `bin/rails dev:cache` และเรื่องเก่าของ Page/Action Caching
- Step 623: Fragment Caching เบื้องต้น — `<% cache @post do %>` และที่มาของ Cache Key
- Step 624: Russian Doll Caching — ซ้อน Cache หลายชั้นด้วย `touch: true`
- Step 625: พิสูจน์ Selective Invalidation ด้วย Log จริง
- Step 626: Low-level Caching — `Rails.cache.fetch` สำหรับงานที่ไม่ใช่ View
- Step 627: ออกแบบ Cache Key ให้ดี — Versioned Key, พารามิเตอร์, และการเลี่ยง Collision
- Step 628: เคลียร์ Cache ด้วยมือ — `Rails.cache.delete`, `Rails.cache.clear` และข้อควรระวัง
- Step 629: Collection Caching — `render collection:, cached: true`
- Step 630: HTTP Caching — `fresh_when`, `stale?`, ETag, `Cache-Control`
- แบบฝึกหัด: Russian Doll Caching เต็มรูปแบบสำหรับหน้า Post + Comments พร้อมพิสูจน์ด้วย log จริง

---

## Step 621: ทำไมต้อง Caching และ Cache Store ที่ Rails รองรับ

### ปัญหาที่ caching แก้

ลองนึกภาพหน้า Dashboard ที่แสดงยอดขายรวมของทั้งเดือน คำนวณจากการ join ตารางออเดอร์นับแสนแถว
ใช้เวลา query+คำนวณ 800ms ทุกครั้ง ถ้าหน้านี้มีคนเข้าดู 1,000 ครั้งต่อนาที (และตัวเลขเปลี่ยน
จริงแค่ตอนมีออเดอร์ใหม่เข้ามา ไม่ใช่ทุกวินาที) แปลว่าเราคำนวณ**ค่าเดิมซ้ำๆ 1,000 ครั้ง**โดย
เปล่าประโยชน์ — นี่คือแก่นของปัญหาที่ caching แก้: **เก็บผลลัพธ์ของงานที่ทำไปแล้วไว้ใช้ซ้ำ
แทนที่จะคำนวณ/render ใหม่ทุกครั้ง** ตราบใดที่ข้อมูลต้นทางยังไม่เปลี่ยน

Caching ใน Rails มีหลายระดับ ตั้งแต่ระดับเล็กสุด (ชิ้นส่วน HTML หนึ่งก้อนในหน้าเว็บ) ไปจนถึง
ระดับใหญ่สุด (ทั้ง response ถูก cache ไว้ที่ CDN โดย Rails process ไม่ต้องทำงานเลยด้วยซ้ำ)
Part นี้เน้นที่ **fragment caching**, **low-level caching** และแตะ **HTTP caching** เบาๆ
เป็นการเปิดประตูไปสู่แนวคิดนั้น

### Cache Store — ที่เก็บข้อมูล cache จริงๆ อยู่ที่ไหน

`Rails.cache` คือ object กลางที่ interface เดียวกันหมด (`.fetch`, `.write`, `.read`,
`.delete`, `.clear`) แต่ **เก็บข้อมูลจริงที่ไหนขึ้นอยู่กับ cache store ที่ตั้งค่าไว้**
ปรับได้ผ่าน `config.cache_store` ในแต่ละ environment file:

| Cache Store | เก็บข้อมูลที่ไหน | ต้องมี service แยกไหม | เหมาะกับ | ข้อควรระวัง |
|---|---|---|---|---|
| `:solid_cache_store` | ตาราง `solid_cache_entries` ในฐานข้อมูล SQL (แยก database ก็ได้ หรือใช้ร่วมกับ primary db ก็ได้) | **ไม่ต้อง** | ค่า default ของ Rails 8 บน production, ทีมเล็ก-กลางที่ไม่อยากเพิ่ม infra | อ่าน/เขียนผ่าน DB round-trip ช้ากว่า in-memory เล็กน้อย ต้อง migrate schema ก่อนใช้ |
| `:memory_store` | Heap ของ process Ruby เอง (RAM) | ไม่ต้อง | ค่า default ของ **development** | ไม่ share ข้ามหลาย process/worker, ข้อมูลหายทันทีที่ restart server, ไม่เหมาะ production ที่รันหลาย process |
| `:file_store` | ไฟล์บน disk (default `tmp/cache/`) | ไม่ต้อง | เซิร์ฟเวอร์เดี่ยวเล็กๆ ที่ไม่อยากพึ่ง DB/Redis เลย | I/O ช้ากว่า memory, ไม่ share ข้ามหลายเครื่อง (multi-server) |
| `:redis_cache_store` | Redis server แยกต่างหาก | **ต้อง** (Redis) | ระบบใหญ่/หลาย process ที่ต้องการความเร็วสูง, TTL ละเอียด, และมักมี Redis อยู่แล้วสำหรับ Sidekiq | ต้องดูแล infra เพิ่ม (เหมือนที่เจอตอนตั้งค่า Sidekiq ใน Part 062) |
| `:mem_cache_store` | Memcached server แยกต่างหาก (ผ่าน gem `dalli`) | **ต้อง** (Memcached) | ระบบเก่าที่มี Memcached อยู่แล้ว | ฟีเจอร์น้อยกว่า Redis (ไม่มี persistence, ไม่มี data structure ซับซ้อน) |
| `:null_store` | ไม่เก็บอะไรเลย (ทุก `.fetch` = miss เสมอ) | ไม่ต้อง | ค่า default ของ **test environment** เพื่อไม่ให้ cache ปนกันข้าม test case | ไม่ใช่ cache จริง — ใช้เพื่อ "ปิด" caching เท่านั้น |

### จุดเด่นของ Rails 8: solid_cache เป็น default บน production แล้ว

ก่อนหน้านี้ (Rails ≤ 7) ถ้าอยากมี cache store ที่ share ข้ามหลาย process/server บน production
ได้จริง แทบจะบังคับต้องติดตั้ง Redis หรือ Memcached เพิ่ม — เป็น infrastructure อีกชิ้นที่ต้อง
ดูแล ตั้งแต่ Rails 8 เป็นต้นมา **solid_cache** (gem ที่ทีม Rails core เขียนเอง คู่กับ
solid_queue และ solid_cable) เข้ามาเป็นทางเลือก default แทน โดยใช้ฐานข้อมูล SQL ธรรมดา
(เช่น SQLite/PostgreSQL/MySQL ตัวเดียวกับที่แอปใช้อยู่แล้ว) เป็นที่เก็บ cache — **ไม่ต้องมี
Redis/Memcached แยกต่างหากเลยก็ cache ได้ในระดับ production จริง**

ทดสอบจริงด้วยการ generate แอปใหม่บน Rails 8.1.4 (`rails new myapp` แบบไม่ใส่ `--minimal`)
พบว่า `Gemfile` มี `gem "solid_cache"` มาให้อัตโนมัติ และ `config/environments/production.rb`
ตั้งค่าให้เลยว่า:

```ruby
# config/environments/production.rb (ส่วนที่ generator ใส่ให้อัตโนมัติ)
config.cache_store = :solid_cache_store
```

พร้อมไฟล์ `config/cache.yml` ที่กำหนด retention policy:

```yaml
# config/cache.yml
default: &default
  store_options:
    # Cap age of oldest cache entry to fulfill retention policies
    # max_age: <%= 60.days.to_i %>
    max_size: <%= 256.megabytes %>
    namespace: <%= Rails.env %>

development:
  <<: *default

test:
  <<: *default

production:
  database: cache          # แยก database ต่างหากจาก primary บน production
  <<: *default
```

> **ข้อสังเกตสำคัญที่ทดสอบจริงแล้ว:** ค่า default ของ **development** ที่ Rails 8.1.4 generate
> ให้ยังคงเป็น `config.cache_store = :memory_store` (ไม่ใช่ solid_cache) — solid_cache ถูกตั้ง
> เป็น default อัตโนมัติเฉพาะ**production** เท่านั้น ส่วน development ยังใช้ memory_store แบบ
> เดิมเพื่อความเร็วสูงสุดตอนพัฒนา (ไม่ต้องมี DB round-trip) ในบทนี้เราจะ**สลับ development ให้
> ใช้ solid_cache_store ด้วย** เพื่อสาธิตพฤติกรรมจริงของมันแบบเห็นในฐานข้อมูลได้เลย และเพราะ
> การเห็น cache entry อยู่ในตาราง SQL จริงๆ ช่วยให้เข้าใจกลไกเบื้องหลังชัดกว่า memory_store ที่
> มองไม่เห็นจากภายนอก process

```ruby
# config/environments/development.rb
# เปลี่ยนจาก config.cache_store = :memory_store เป็น
config.cache_store = :solid_cache_store
```

เพราะ development ไม่ได้แยก database สำหรับ cache แบบ production (ไม่มี key `database:` ใน
`config/cache.yml` สำหรับ development) solid_cache จะใช้ connection เดียวกับฐานข้อมูลหลัก
(“unmanaged” mode) จึงต้องมีตาราง `solid_cache_entries` อยู่ในฐานข้อมูล development ด้วย
ถ้า generate แอปแบบเต็ม (ไม่ `--minimal`) ตัว installer จะสร้าง `db/cache_schema.rb` ให้แล้ว
แต่เมื่อไม่ได้แยก database ในบทนี้ เราเพิ่มตารางนี้เข้าไปในฐานข้อมูลหลักด้วย migration ปกติ
(รายละเอียดเต็มอยู่ในแบบฝึกหัดท้าย Part):

```ruby
# db/migrate/xxxx_create_solid_cache_entries_for_dev.rb
class CreateSolidCacheEntriesForDev < ActiveRecord::Migration[8.1]
  def change
    create_table :solid_cache_entries, force: :cascade do |t|
      t.binary :key, limit: 1024, null: false
      t.binary :value, limit: 536870912, null: false
      t.datetime :created_at, null: false
      t.integer :key_hash, limit: 8, null: false
      t.integer :byte_size, limit: 4, null: false
      t.index [:byte_size], name: "index_solid_cache_entries_on_byte_size"
      t.index [:key_hash, :byte_size], name: "index_solid_cache_entries_on_key_hash_and_byte_size"
      t.index [:key_hash], name: "index_solid_cache_entries_on_key_hash", unique: true
    end
  end
end
```

ทดสอบจริงใน `bin/rails console` หลัง migrate และตั้งค่าเสร็จ:

```irb
irb> Rails.cache.class
=> SolidCache::Store

irb> Rails.cache.write("hello", "world")
=> true

irb> Rails.cache.read("hello")
=> "world"

irb> SolidCache::Entry.count
=> 1
```

ยืนยันว่า `Rails.cache` เขียน/อ่านผ่าน solid_cache จริง และข้อมูลถูกเก็บเป็นแถวในตาราง SQL
จริงๆ — ไม่ใช่แค่ใน memory ของ process อีกต่อไป (แปลว่าข้อมูล cache **รอดจากการ restart
server** ด้วย ต่างจาก `:memory_store` ที่หายทันทีที่ process ตาย)

---

## Step 622: เปิดใช้ Caching ใน Development ด้วย `bin/rails dev:cache`

### ทำไม caching ถึง "ปิด" อยู่ใน development โดย default

ถ้าเปิด `config/environments/development.rb` จะเห็น comment ที่ Rails generate ให้ชัดเจน:

```ruby
# config/environments/development.rb
# Enable/disable Action Controller caching. By default Action Controller caching is disabled.
# Run rails dev:cache to toggle Action Controller caching.
if Rails.root.join("tmp/caching-dev.txt").exist?
  config.action_controller.perform_caching = true
  config.action_controller.enable_fragment_cache_logging = true
  config.public_file_server.headers = { "cache-control" => "public, max-age=#{2.days.to_i}" }
else
  config.action_controller.perform_caching = false
end
```

เหตุผลตรงไปตรงมา: ตอนพัฒนาเราต้องการเห็นผลของทุกการแก้โค้ด/ข้อมูลทันที ถ้า fragment cache
เปิดอยู่โดย default นักพัฒนาจะงงว่า "ทำไมแก้ view แล้วหน้าเว็บไม่เปลี่ยน" (ทั้งที่จริงๆ มัน
แค่ยัง render จาก cache เก่า) Rails จึงปิด `perform_caching` ไว้เป็นค่า default ใน development
เสมอ — โค้ด `<% cache @post do %>...<% end %>` ในบทถัดๆ ไปจะยังทำงานโดยไม่ error แต่จะ**ไม่
cache จริง** (render ใหม่ทุกครั้ง) จนกว่าจะเปิดสวิตช์นี้

### เปิดสวิตช์ด้วย `bin/rails dev:cache`

```bash
bin/rails dev:cache
```

ทดสอบจริง — รันครั้งแรก:

```
Action Controller caching enabled for development mode.
```

รันซ้ำอีกครั้ง (toggle กลับ):

```
Action Controller caching disabled for development mode.
```

คำสั่งนี้แค่สร้าง/ลบไฟล์ `tmp/caching-dev.txt` (empty marker file) ที่ตัว
`config/environments/development.rb` เช็คอยู่ตามโค้ดด้านบน **ต้อง restart server หลังสลับ**
เพราะค่านี้ถูกอ่านตอน boot process เท่านั้น ไม่ใช่ config ที่ reload กลางอากาศได้เหมือนโค้ด
แอปทั่วไป — ตลอดบทนี้เราเปิด dev:cache ไว้ตลอดเพื่อให้เห็นพฤติกรรม cache จริงในทุกตัวอย่าง

> **ข้อควรระวัง:** อย่าลืมปิด caching กลับ (`bin/rails dev:cache` อีกครั้ง) เวลาสลับกลับไปทำงาน
> ปกติที่ไม่ได้ทดสอบเรื่อง cache เพราะมิฉะนั้นจะงงว่าทำไมแก้ view/data แล้วหน้าเว็บไม่อัปเดต

### Page Caching และ Action Caching — ของเก่าที่ถูกถอดออกจาก Rails core แล้ว

ถ้าอ่านบทความ Rails เก่าๆ (สมัย Rails 2–3) อาจเจอคำว่า **Page Caching** (cache ทั้ง HTML
response เป็นไฟล์ static แล้วให้ web server เสิร์ฟไฟล์นั้นตรงๆ โดยไม่ผ่าน Rails process เลย)
และ **Action Caching** (คล้าย page caching แต่ยังผ่าน filter/before_action ของ controller
ก่อน) — ทั้งสองแบบนี้ **ถูกถอดออกจาก Rails core ตั้งแต่ Rails 4** (ย้ายไปเป็น gem แยกต่างหาก
ชื่อ `actionpack-page_caching` และ `actionpack-action_caching` ซึ่งปัจจุบันแทบไม่มีใครใช้แล้ว)

เหตุผลที่ถูกเลิกใช้: มันเหมาะกับเว็บที่เนื้อหา "เหมือนกันทุกคนเป๊ะๆ" เท่านั้น (ไม่มี login,
ไม่มีเนื้อหาต่างกันตาม user) ซึ่งเว็บแอปยุคปัจจุบันแทบทุกตัวมี personalization บางอย่างเสมอ
(ปุ่ม login/logout, ชื่อผู้ใช้, CSRF token ที่ต้องต่างกันทุก request) ทำให้ cache ทั้งหน้าแบบ
เพจเดียวใช้ไม่ได้จริงในทางปฏิบัติ **แนวทางที่มาแทนที่และเป็นมาตรฐานของ Rails ตั้งแต่นั้นมาคือ
"Fragment Caching"** — cache แค่บาง**ส่วน**ของหน้าเว็บที่เหมือนกันทุกคน (เช่น เนื้อหาบทความ)
ในขณะที่ส่วนที่ต่างกันตาม user (เช่น navbar ที่มีชื่อผู้ใช้) render สดทุกครั้ง — นี่คือหัวข้อ
หลักของ Part นี้ตั้งแต่ Step ถัดไป

---

## Step 623: Fragment Caching เบื้องต้น — `<% cache @post do %>` และที่มาของ Cache Key

### Syntax พื้นฐาน

สมมติแอป Blog เดิมจาก Part 030 ที่มี `Post` และ `Comment` — หน้า `posts/show.html.erb` แบบ
ไม่มี cache:

```erb
<%# app/views/posts/show.html.erb (ไม่มี cache) %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>
```

ห่อด้วย `cache` helper:

```erb
<%# app/views/posts/show.html.erb (มี fragment cache) %>
<% cache @post do %>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
<% end %>
```

แค่นี้ Rails จะ**เก็บ HTML ที่ render ได้จากบล็อกนี้ไว้ใน cache store** ครั้งต่อไปที่มีคน
เข้าหน้านี้ (ด้วย `@post` ตัวเดียวกัน และข้อมูลยังไม่เปลี่ยน) Rails จะดึง HTML ที่เก็บไว้มาใช้
ตรงๆ โดย**ไม่รันโค้ด Ruby ข้างในบล็อกซ้ำเลย** (ไม่ query DB ซ้ำ ไม่ประมวลผล logic ซ้ำ)

### Cache Key มาจากไหน — `cache_key_with_version`

หัวใจสำคัญของ fragment caching คือ**คำถามที่ว่า cache นี้ควรจะยังใช้ได้อยู่หรือเปล่า** — Rails
ตอบคำถามนี้ด้วยการผูก cache แต่ละก้อนเข้ากับ **cache key** ที่คำนวณจาก 2 ส่วนหลัก:

1. **Template digest** — hash ที่คำนวณจากเนื้อหาไฟล์ `.html.erb` เอง (รวมถึง partial ที่มัน
   เรียกใช้) ถ้าแก้โค้ดในไฟล์ template นี้ (เช่น เปลี่ยน CSS class, เพิ่ม field ใหม่) digest
   จะเปลี่ยนทันที ทำให้ cache เก่าถูกมองว่า "ไม่ตรงกัน" อัตโนมัติโดยไม่ต้องไปลบ cache เก่าเอง
2. **`cache_key_with_version` ของ object** — ผูกกับข้อมูลจริงของ record นั้น โดย default
   Active Record จะสร้างจาก `<table_name>/<id>-<updated_at ที่แปลงเป็นตัวเลข>`

ทดสอบจริงใน console:

```irb
irb> post = Post.first
irb> post.cache_key
=> "posts/1"

irb> post.cache_key_with_version
=> "posts/1-20260926081716956590"

irb> post.updated_at
=> Sat, 26 Sep 2026 08:17:16 UTC +00:00
```

สังเกตว่า `cache_key_with_version` มีตัวเลขต่อท้ายที่มาจาก `updated_at` (แปลงเป็น timestamp
แบบละเอียดระดับ microsecond) — **นี่คือกลไกทั้งหมดของการ invalidate cache แบบอัตโนมัติ**:
ทุกครั้งที่ record ถูก `update` (แม้แต่ field เดียว) `updated_at` จะขยับ ทำให้
`cache_key_with_version` เปลี่ยนค่า ทำให้ key ที่ใช้ค้นหาใน cache store เปลี่ยนไปด้วย —
เท่ากับว่า cache เก่าที่ผูกกับ key เดิม**ไม่มีใครมาอ่านอีกแล้ว** (มันไม่ได้ถูก "ลบ" แต่ถูก
"ทิ้งไว้เฉยๆ" กลายเป็นขยะที่ค่อยๆ ถูกเก็บกวาดทีหลังตาม retention policy ของ cache store)

`cache @post do` (โดยไม่ระบุอะไรเพิ่ม) เทียบเท่ากับการเขียนเต็มๆ ว่า:

```erb
<% cache "views/posts/show:#{template_digest}/#{@post.cache_key_with_version}" do %>
```

### ดู Cache Key เต็มรูปแบบจาก log จริง

เปิด `bin/rails dev:cache` แล้วเข้าหน้า `/posts/1` ครั้งแรก (cache ยังว่าง) log ที่ได้จริง:

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (8.1ms)
Write fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (1.5ms)
```

สังเกต key เต็ม: `views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401`
แยกเป็น 2 ส่วนตามที่อธิบายไว้:

- `views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee` — virtual path ของ template
  บวก template digest (hash 32 ตัวอักษร)
- `posts/1-20260926081656502401` — `cache_key_with_version` ของ `@post`

**Rails จะ log บรรทัด `Read fragment` เสมอทุกครั้งที่เจอ `cache do...end`** ไม่ว่าจะ hit
หรือ miss (มันคือความพยายาม "อ่าน" cache ก่อนเสมอ) แต่จะมีบรรทัด `Write fragment` ตามมา
**เฉพาะตอน miss เท่านั้น** (เพราะต้องรันโค้ดในบล็อกแล้วเขียนผลลัพธ์กลับเข้า cache) ทดสอบเข้า
หน้าเดิมซ้ำอีกครั้ง (ข้อมูลไม่เปลี่ยน):

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (5.6ms)
```

มีแค่ `Read fragment` บรรทัดเดียว **ไม่มี `Write fragment`** — นี่คือ **cache hit** และ
สังเกตด้วยว่า `Rendered posts/show.html.erb` มี Duration ลดจาก 33.5ms เหลือ 5.9ms เพราะ
Rails ข้ามการ query และ render เนื้อหาข้างในไปเลย

---

## Step 624: Russian Doll Caching — ซ้อน Cache หลายชั้นด้วย `touch: true`

### แนวคิด

หน้า `posts/show` จริงๆ ไม่ได้มีแค่เนื้อหาโพสต์ ยังมีลิสต์ comment ต่อท้ายด้วย ถ้า cache
ทั้งหน้าเป็นก้อนเดียวรวม comment ด้วย จะเกิดปัญหา: **แค่มีคนพิมพ์ comment ใหม่ 1 อัน หรือแก้ไข
comment เก่า 1 อัน ก็ต้อง invalidate cache ทั้งหน้า** ทั้งที่เนื้อหาโพสต์เองไม่ได้เปลี่ยนเลย
และถ้าโพสต์นั้นมี comment เป็นร้อยๆ อัน การ re-render ใหม่ทั้งหมดทุกครั้งที่มี comment ใหม่
1 อันก็สิ้นเปลืองมาก

**Russian Doll Caching** (ตั้งชื่อตามตุ๊กตาแม่ลูกดกของรัสเซียที่ซ้อนกันเป็นชั้นๆ) คือการ
**ซ้อน fragment cache หลายชั้น** โดยชั้นในสุด cache แต่ละ comment แยกกัน แล้วชั้นนอกครอบด้วย
cache ของทั้งโพสต์อีกที — เมื่อ comment หนึ่งอันเปลี่ยน จะ invalidate แค่ 2 จุด: fragment ของ
comment นั้นเอง กับ fragment ชั้นนอกสุดของโพสต์ (เพราะโครงสร้างหน้าเปลี่ยน) แต่ **fragment
ของ comment ตัวอื่นๆ ที่ไม่เกี่ยวข้องยังคง valid อยู่เหมือนเดิม ไม่ต้อง render ใหม่**

### การเชื่อม comment เข้ากับ `updated_at` ของ post ด้วย `touch: true`

ปัญหาคือ: ถ้า comment ถูกแก้ไข ก้อน cache ชั้นในของ comment นั้นจะ invalidate เองอัตโนมัติอยู่
แล้ว (เพราะ `comment.updated_at` เปลี่ยน) แต่ก้อน cache ชั้นนอกที่ผูกกับ **`post.updated_at`**
จะยัง**ไม่รู้เรื่อง**ด้วย เพราะการแก้ comment ไม่ได้ไปแตะ column ใดๆ ของตาราง `posts` เลย —
นี่คือจุดที่ `touch: true` (ที่เกริ่นไว้ใน **Part 035 Step 348**) เข้ามาแก้ปัญหาพอดี:

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end

# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

`touch: true` ทำให้ทุกครั้งที่ comment ถูก create/update/destroy Rails จะยิง
`UPDATE posts SET updated_at = ... WHERE id = ?` ให้อัตโนมัติในธุรกรรมเดียวกัน — เท่ากับว่า
`post.updated_at` "ขยับ" ตามทุกครั้งที่มีอะไรเปลี่ยนใน comment ของมัน ทำให้
`post.cache_key_with_version` เปลี่ยนตาม และ cache ชั้นนอกของโพสต์ invalidate โดยอัตโนมัติ
พอดีกับที่เนื้อหาจริงๆ เปลี่ยนไป (มี comment ใหม่/แก้ไข/ถูกลบ)

### เขียน View แบบ Russian Doll

รูปแบบที่ Rails แนะนำและใช้กันเป็นมาตรฐาน คือห่อ **ชั้นนอก** ด้วย `cache @post do` แล้วข้างใน
ใช้ **collection caching** (จะอธิบายละเอียดใน Step 629) สำหรับ comment แต่ละอัน:

```erb
<%# app/views/posts/show.html.erb %>
<% cache @post do %>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
  <p>Post updated_at: <%= @post.updated_at.to_fs(:inspect) %></p>

  <h2>Comments (<%= @post.comments.size %>)</h2>

  <%= render partial: "comments/comment", collection: @post.comments.order(:id), cached: true %>
<% end %>
```

```erb
<%# app/views/comments/_comment.html.erb %>
<div class="comment" id="comment_<%= comment.id %>">
  <p><%= comment.body %></p>
  <small>comment updated_at: <%= comment.updated_at.to_fs(:inspect) %></small>
</div>
```

สังเกตโครงสร้าง: `cache @post do ... end` คือ**ตุ๊กตาชั้นนอก** ข้างในมี
`render ..., cached: true` ที่สร้าง**ตุ๊กตาชั้นในหลายตัว** (หนึ่งตัวต่อหนึ่ง comment) —
นี่คือ "Russian Doll" ตามชื่อ: cache ก้อนเล็กซ้อนอยู่ข้างในของ cache ก้อนใหญ่

---

## Step 625: พิสูจน์ Selective Invalidation ด้วย Log จริง

นี่คือส่วนที่สำคัญที่สุดของ Part นี้ — เราจะพิสูจน์ด้วย log จริงว่า **การแก้ comment แค่ 1 อัน
ไม่ทำให้ comment อื่นๆ ที่ไม่เกี่ยวข้องต้อง render ใหม่**

### รอบที่ 1: เข้าหน้าครั้งแรก (cache ว่างทั้งหมด)

สมมติโพสต์ 1 มี comment 3 อัน เข้า `/posts/1` ครั้งแรก:

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (8.1ms)
  Rendered collection of comments/_comment.html.erb [0 / 3 cache hits] (Duration: 11.2ms | GC: 1.3ms)
Write fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (1.5ms)
```

อ่านออกมาเป็นขั้นตอน:

1. ลอง `Read fragment` ชั้นนอก (posts/1) — **miss**
2. เพราะ miss เลยต้องรันโค้ดข้างใน block ซึ่งรวมถึงการ render collection ของ comment —
   ลอง read cache ของ comment ทั้ง 3 อัน ได้ `[0 / 3 cache hits]` (miss หมดทั้ง 3 อัน
   เพราะยังไม่เคย cache มาก่อน)
3. `Write fragment` ชั้นนอก — เขียน HTML ที่ render เสร็จแล้ว (รวม comment ทั้ง 3 ที่ฝังอยู่
   ข้างใน) กลับเข้า cache

### รอบที่ 2: เข้าหน้าซ้ำ (ข้อมูลยังไม่เปลี่ยน)

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (5.6ms)
```

**มีแค่บรรทัดเดียว!** เพราะชั้นนอก hit ทันที Rails เลยไม่ต้อง execute โค้ดข้างใน block อีกเลย —
ไม่มีการ query comment ใหม่ ไม่มีการเช็ค cache ของ comment แต่ละอันด้วยซ้ำ (เพราะมันไม่ได้
รันโค้ดถึงจุดนั้นเลย) นี่คือประสิทธิภาพสูงสุดของ caching: **ข้ามการทำงานทั้งหมดไปเลยเมื่อรู้ว่า
ผลลัพธ์เดิมยังใช้ได้**

### รอบที่ 3: แก้ไข comment 1 อัน แล้วเข้าหน้าอีกครั้ง

```irb
irb> c = Comment.find(1)
irb> c.update!(body: "แก้ไขคอมเมนต์นี้ใหม่ เพื่อทดสอบ Russian Doll Caching")
irb> c.post.reload.updated_at
=> Sat, 26 Sep 2026 08:17:16 UTC +00:00   # เปลี่ยนจากเดิมทันที เพราะ touch: true
```

เข้า `/posts/1` อีกครั้ง log ที่ได้จริง:

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (1.1ms)
  Rendered collection of comments/_comment.html.erb [2 / 3 cache hits] (Duration: 3.0ms | GC: 0.0ms)
Write fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (2.2ms)
```

**นี่คือหลักฐานที่ต้องการ:**

- Key ของชั้นนอกเปลี่ยนจาก `...posts/1-20260926081656502401` เป็น
  `...posts/1-20260926081716956590` (เพราะ `post.updated_at` ขยับตาม `touch: true`) →
  **miss** (สมเหตุสมผล เพราะโครงสร้างข้อมูลของหน้าเปลี่ยนจริง มี comment ที่แก้ไขไปแล้ว)
- แต่ collection ของ comment แสดง **`[2 / 3 cache hits]`** — แปลว่ามีแค่ **1 อัน** (comment
  ที่เพิ่งแก้ไข) ที่ต้อง render ใหม่ ส่วน **2 อันที่เหลือยังคง cache hit** ไม่ถูกแตะต้องเลย

ดู SQL query เบื้องหลังก็ยืนยันเรื่องเดียวกัน — Rails อ่าน cache key ของ comment ทั้ง 3 ตัว
ด้วย query เดียว (multi-get แบบ batch):

```sql
SELECT "solid_cache_entries"."key", "solid_cache_entries"."value"
FROM "solid_cache_entries"
WHERE "solid_cache_entries"."key_hash" IN (253991913665818901, -8579237593648541179, -125786583592899438)
```

แล้วมีแค่ **1 คำสั่ง `INSERT ... ON CONFLICT DO UPDATE`** (upsert) ตามมา — เขียนกลับแค่ก้อน
เดียวที่ miss จริงๆ ไม่ใช่เขียนทั้ง 3 ก้อนใหม่หมด **สรุปสั้นๆ:** การแก้ comment 1 อัน จาก
comment ทั้งหมด 3 อันในโพสต์ ทำให้เกิดงาน render+write cache ใหม่แค่ 1 comment เท่านั้น
(บวกกับ fragment ชั้นนอกของ post เอง) — ไม่ใช่ทั้ง 3 comment เหมือนที่คนไม่เข้าใจ Russian
Doll Caching อาจกลัวว่าจะเกิดขึ้น

### รอบที่ 4: เข้าหน้าซ้ำอีกครั้งหลังแก้ไข (cache อุ่นใหม่แล้ว)

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (1.1ms)
```

กลับมาเป็น full hit อีกครั้ง — cache "settle" ตัวใหม่เรียบร้อย พร้อมรับ traffic ต่อไปโดยไม่ต้อง
คำนวณซ้ำ จนกว่าจะมีการเปลี่ยนแปลงข้อมูลครั้งถัดไป

---

## Step 626: Low-level Caching — `Rails.cache.fetch` สำหรับงานที่ไม่ใช่ View

Fragment caching เหมาะกับการ cache **ผลลัพธ์การ render HTML** แต่บางงานไม่เกี่ยวกับ view
เลย เช่น ผลการคำนวณสถิติที่ query หนัก, ผลลัพธ์จากการเรียก external API ที่ช้าและมี rate
limit, หรือค่าที่ใช้ร่วมกันหลายที่ในแอป (ทั้ง controller, background job, mailer) — สำหรับ
งานพวกนี้เราใช้ **low-level caching** ผ่าน `Rails.cache.fetch` โดยตรง

### รูปแบบพื้นฐาน

```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  def expensive
    result = Rails.cache.fetch("reports/expensive_total", expires_in: 10.seconds) do
      Rails.logger.info "[EXPENSIVE] กำลังคำนวณจริง (ไม่ได้อ่านจาก cache)..."
      sleep 1 # จำลอง query หนักหรือ HTTP call ไปหา external service ที่ช้า
      Post.joins(:comments).count
    end

    render plain: "total comments across posts = #{result}"
  end
end
```

`Rails.cache.fetch(key, options) { block }` ทำงานดังนี้:

1. ลองหา `key` ใน cache store — **ถ้าเจอ (hit)** คืนค่าที่เก็บไว้ทันที **ไม่รัน block เลย**
2. **ถ้าไม่เจอ (miss)** รัน `block` เพื่อคำนวณค่าจริง แล้วเขียนผลลัพธ์เก็บเข้า cache ด้วย
   `key` เดียวกัน ก่อนคืนค่ากลับ
3. `expires_in:` กำหนดว่า entry นี้จะ "หมดอายุ" (ถูกมองว่าเป็น miss) หลังจากเวลาที่กำหนด แม้
   ไม่มีใครไปลบมันออกเองก็ตาม

ทดสอบจริงด้วยการวัดเวลา:

```bash
$ time curl -s http://localhost:3000/reports/expensive
total comments across posts = 5
real    0m1.031s   # ช้า เพราะ miss แล้วต้องรัน sleep 1 จริง

$ time curl -s http://localhost:3000/reports/expensive
total comments across posts = 5
real    0m0.014s   # เร็วขึ้นมาก เพราะอ่านจาก cache ตรงๆ ไม่รัน block เลย
```

และใน log เห็นชัดว่า `[EXPENSIVE] กำลังคำนวณจริง...` ปรากฏแค่ครั้งเดียว (ตอน request แรก
ที่ miss) ไม่ปรากฏเลยในครั้งที่สอง — พิสูจน์ว่า block ไม่ได้ถูกรันซ้ำจริงๆ

ทดสอบต่อว่าหลังผ่านไปเกิน `expires_in: 10.seconds` แล้ว entry หมดอายุจริง:

```bash
$ sleep 11
$ time curl -s http://localhost:3000/reports/expensive
total comments across posts = 5
real    0m1.019s   # กลับมาช้าอีกครั้ง เพราะ entry หมดอายุแล้ว ต้องคำนวณใหม่
```

log ยืนยันว่า `[EXPENSIVE] กำลังคำนวณจริง...` กลับมาปรากฏอีกครั้งหลังผ่าน 11 วินาที — ตรงตาม
`expires_in: 10.seconds` ที่ตั้งไว้พอดี

### ใช้ low-level caching เพื่อ cache ผลจาก external API

Use case ที่พบบ่อยมากในงานจริง: เรียก API ของบุคคลที่สาม (เช่น อัตราแลกเปลี่ยนเงินตรา, ข้อมูล
สภาพอากาศ, ราคาหุ้น) ที่มี rate limit และช้า:

```ruby
class ExchangeRateService
  def self.usd_to_thb
    Rails.cache.fetch("exchange_rate/usd_to_thb", expires_in: 1.hour) do
      response = Faraday.get("https://api.example.com/rates/USD_THB")
      JSON.parse(response.body)["rate"]
    end
  end
end
```

โค้ดฝั่งที่เรียกใช้ (`ExchangeRateService.usd_to_thb`) ไม่ต้องรู้เลยว่าข้างในมี caching อยู่
— เรียกกี่ครั้งก็ได้ผลเร็วเท่ากันเสมอ (ยกเว้นครั้งแรกสุดหรือหลังหมดอายุ) และที่สำคัญคือ **ลด
จำนวนครั้งที่ยิงไปหา API จริงลงมหาศาล** ซึ่งมักมี rate limit หรือค่าใช้จ่ายต่อ request ด้วย

---

## Step 627: ออกแบบ Cache Key ให้ดี — Versioned Key, พารามิเตอร์, และการเลี่ยง Collision

Cache key ที่ออกแบบไม่ดีคือสาเหตุอันดับต้นๆ ของบั๊ก **"ข้อมูลเก่าค้าง" (stale cache)** หรือ
**"cache ของคนนึงไปโผล่ให้อีกคนเห็น"** (cache collision) หลักการออกแบบ cache key ที่ดี:

### 1. ใส่ทุกอย่างที่ทำให้ผลลัพธ์ต่างกันลงใน key

ถ้าผลลัพธ์ขึ้นกับ user, พารามิเตอร์, หรือ locale ต้องใส่สิ่งเหล่านั้นลงใน key ด้วย ไม่ใช่ใส่
แค่ชื่อ resource เฉยๆ:

```ruby
# ผิด — ทุก user เห็นค่าเดียวกันหมด ทั้งที่ควรต่างกันตาม user
Rails.cache.fetch("dashboard_summary") { build_summary_for(current_user) }

# ถูก — แยก cache ต่อ user
Rails.cache.fetch("dashboard_summary/user/#{current_user.id}") { build_summary_for(current_user) }

# ถูก และรองรับ invalidate อัตโนมัติเมื่อ user เปลี่ยนด้วย (ผูกกับ updated_at ของ user)
Rails.cache.fetch(["dashboard_summary", current_user]) { build_summary_for(current_user) }
```

ข้อสุดท้ายใช้ **Array เป็น cache key** ได้เลย — Rails จะ normalize แต่ละ element ให้เอง (ถ้า
เป็น Active Record object จะเรียก `cache_key_with_version` ให้อัตโนมัติ) ทดสอบจริง:

```irb
irb> key = ["report", 42, :monthly, "2026-09"]
irb> Rails.cache.write(key, "cached report")
irb> ActiveSupport::Cache.expand_cache_key(key)
=> "report/42/monthly/2026-09"

irb> Rails.cache.read(key)
=> "cached report"

irb> Rails.cache.read("report/42/monthly/2026-09")   # อ่านด้วย string key ที่ normalize แล้วก็ได้ค่าเดียวกัน
=> "cached report"
```

### 2. Versioned Key — เผื่อ logic การคำนวณเปลี่ยนแต่ข้อมูลต้นทางไม่เปลี่ยน

Fragment caching แก้ปัญหานี้ให้อัตโนมัติด้วย template digest (Step 623) แต่ low-level
caching **ไม่มีกลไกแบบนั้นให้ฟรี** — ถ้าเราแก้ logic การคำนวณใน block (เช่น เปลี่ยนสูตร
คำนวณ, แก้บั๊ก) แต่ key เดิม ผลลัพธ์เก่าที่ผิดพลาดจะยังถูกอ่านออกมาใช้ต่อจนกว่าจะหมดอายุเอง
วิธีแก้คือใส่ **version number** ต่อท้าย key แล้ว bump เลขทุกครั้งที่แก้ logic:

```ruby
CACHE_VERSION = "v2" # bump เป็น v3 ทุกครั้งที่แก้สูตรคำนวณ เพื่อบังคับ invalidate cache เก่าทั้งหมดทันที

Rails.cache.fetch("#{CACHE_VERSION}/reports/expensive_total", expires_in: 1.hour) do
  # ...
end
```

การ bump version ทำให้ key เก่าทั้งหมด "กลายเป็นขยะ" ทันทีโดยไม่ต้องไปไล่ลบทีละอัน (เหมือน
หลักการเดียวกับที่ template digest ทำให้ fragment cache อัตโนมัติ)

### 3. หลีกเลี่ยง Collision — อย่าใช้ key ที่กว้างเกินไป

```ruby
# อันตราย — ถ้ามีหลาย service/feature ใช้ key ชื่อ "count" เหมือนกัน จะทับกันโดยไม่รู้ตัว
Rails.cache.fetch("count") { Post.count }
Rails.cache.fetch("count") { User.count } # ทับ key เดิม! คนละความหมายแต่ key ชนกัน

# ปลอดภัยกว่า — namespace ให้ชัดเจนตามลำดับชั้นของ concept
Rails.cache.fetch("posts/total_count") { Post.count }
Rails.cache.fetch("users/total_count") { User.count }
```

แนวทางที่นิยมคือตั้งชื่อ key แบบ path (`resource/sub_resource/parameter`) เหมือนตัวอย่าง
ด้านบน อ่านง่าย ไม่ชนกันง่ายๆ และ debug ง่ายเวลาต้อง list cache key ที่เกี่ยวกับ resource
หนึ่งๆ ออกมาดู

---

## Step 628: เคลียร์ Cache ด้วยมือ — `Rails.cache.delete`, `Rails.cache.clear` และข้อควรระวัง

บางสถานการณ์เราต้องการบังคับล้าง cache ทันทีโดยไม่รอให้หมดอายุเอง เช่น admin กดปุ่ม
"รีเฟรชรายงาน" หรือมีการแก้ไขข้อมูลผ่านช่องทางที่ไม่ได้ผ่าน callback ปกติ (เช่น
`update_all`, raw SQL)

### `Rails.cache.delete(key)` — ลบ entry เดียว

```irb
irb> Rails.cache.write("stale_entry", 999)
irb> Rails.cache.read("stale_entry")
=> 999

irb> Rails.cache.delete("stale_entry")
=> true

irb> Rails.cache.read("stale_entry")
=> nil
```

ทดสอบจริงยืนยันว่าหลัง `delete` ค่าที่อ่านกลับมาเป็น `nil` ทันที — request ถัดไปที่เรียก
`Rails.cache.fetch` ด้วย key เดียวกันจะถือว่า miss และรัน block คำนวณใหม่ทันที (ไม่ต้องรอ
`expires_in` หมดอายุ)

### `Rails.cache.clear` — ลบ**ทุก** entry ในทั้ง cache store

```irb
irb> SolidCache::Entry.count
=> 7

irb> Rails.cache.clear
=> true

irb> SolidCache::Entry.count
=> 0
```

ทดสอบจริงยืนยันว่า `Rails.cache.clear` ลบทุกแถวในตาราง `solid_cache_entries` ทันที ไม่ว่า
entry นั้นจะเกี่ยวกับ feature ไหนก็ตาม

> **ทำไม `Rails.cache.clear` บน production ถึงมักเป็นความคิดที่แย่:**
>
> 1. **กระทบทุก feature พร้อมกัน** — cache ของทุกหน้า ทุก resource ทุก low-level cache ทั่ว
>    ทั้งแอปหายไปพร้อมกันหมดในคำสั่งเดียว ไม่ใช่แค่ส่วนที่เรามีปัญหา
> 2. **"Thundering herd" ตอนหลัง clear** — ทันทีที่ cache ว่างเปล่า ทุก request ที่เข้ามาถัดไป
>    (ซึ่งอาจมีจำนวนมากพร้อมกันบนแอปที่มี traffic สูง) จะกลายเป็น cache miss พร้อมกันหมด ทำให้
>    ฐานข้อมูล/service ต้นทางถูกยิง query/คำนวณหนักพร้อมกันในทันที ซึ่งอาจทำให้ระบบล่มได้จริง
>    (ปัญหานี้เรียกว่า **cache stampede**)
> 3. **ถ้า cache store share connection เดียวกับ DB หลัก** (เช่น solid_cache ที่ไม่ได้แยก
>    database) การลบข้อมูลจำนวนมากพร้อมกันอาจไปแย่ง lock/IO กับ query ปกติของแอป ทำให้ทั้งแอป
>    ช้าลงชั่วขณะ
>
> **แนวทางที่ปลอดภัยกว่า:** ใช้ `Rails.cache.delete(key)` หรือ `Rails.cache.delete_matched`
> (ลบเฉพาะ key ที่ match pattern — รองรับเฉพาะบาง cache store เช่น `:file_store`,
> `:redis_cache_store`; **solid_cache ไม่รองรับ** `delete_matched` เพราะการ scan pattern
> บน SQL table ขนาดใหญ่จะช้ามาก) หรือดีที่สุดคือออกแบบให้ cache invalidate ตัวเองอัตโนมัติผ่าน
> versioned key/`cache_key_with_version` (Step 623, 627) เพื่อไม่ต้องมานั่ง `clear` ด้วยมือ
> เลยตั้งแต่แรก — เก็บ `Rails.cache.clear` ไว้ใช้เฉพาะตอน debug ใน development หรือกรณีฉุกเฉิน
> จริงๆ บน production เท่านั้น

---

## Step 629: Collection Caching — `render collection:, cached: true`

Step 236 (Part 024) สอนเรื่อง `render partial:, collection:` ที่เร็วกว่าเขียน `each` เอง —
ทีนี้เพิ่ม option `cached: true` เข้าไป จะได้ **collection caching**: Rails cache แต่ละ
partial ในคอลเลกชันแยกกัน**และอ่านทั้งหมดด้วย query เดียว** (multi-get แบบ batch) แทนที่จะ
เช็ค cache ทีละตัว

### ตัวอย่าง: หน้ารายชื่อโพสต์

```erb
<%# app/views/posts/index.html.erb %>
<h1>All posts</h1>

<%= render partial: "post", collection: @posts, cached: true %>
```

```erb
<%# app/views/posts/_post.html.erb %>
<div class="post-card" id="post_<%= post.id %>">
  <h3><%= post.title %></h3>
  <p><%= post.body.to_s.truncate(80) %></p>
  <small>updated_at: <%= post.updated_at.to_fs(:inspect) %></small>
</div>
```

ทดสอบจริง — เข้าหน้า `/posts` (มี 2 โพสต์) ครั้งแรก:

```
Rendered collection of posts/_post.html.erb [0 / 2 cache hits] (Duration: 5.2ms | GC: 0.7ms)
```

เข้าซ้ำอีกครั้ง (ข้อมูลไม่เปลี่ยน):

```
Rendered collection of posts/_post.html.erb [2 / 2 cache hits] (Duration: 2.1ms | GC: 0.4ms)
```

log บรรทัดเดียวสรุปให้เห็นชัดว่ากี่ตัว hit จากทั้งหมดกี่ตัว — สะดวกมากตอน debug ว่า cache
ทำงานได้ผลจริงแค่ไหนในหน้าที่มีลิสต์ยาวๆ ดู SQL เบื้องหลังก็ยืนยันว่าเป็น**การอ่านแบบ
batch เดียว** ไม่ใช่ query แยกทีละ record:

```sql
-- อ่าน cache key ของทั้ง 2 posts ด้วย query เดียว
SELECT "solid_cache_entries"."key", "solid_cache_entries"."value"
FROM "solid_cache_entries"
WHERE "solid_cache_entries"."key_hash" IN (-6822810485970492659, -8867162529269117615)
```

ยิ่ง collection มีจำนวนรายการเยอะ (หลักร้อย-พัน) ความต่างของ `cached: true` กับการ render
ปกติจะยิ่งเห็นชัด เพราะ record ที่ไม่เปลี่ยนแปลง (ส่วนใหญ่ในลิสต์ยาวๆ มักเป็นแบบนี้) จะข้ามการ
render ไปเลย เหลือแค่ record ที่เปลี่ยนจริงๆ ไม่กี่ตัวเท่านั้นที่ต้อง render ใหม่

> **ข้อสังเกต:** collection caching ใช้หลักการเดียวกับ Russian Doll Caching ทุกประการ (แค่ละ
> element ในคอลเลกชัน = ตุ๊กตาหนึ่งชั้น) เพียงแต่ไม่มีตุ๊กตาชั้นนอกครอบทั้งลิสต์ ถ้าอยากเพิ่ม
> ชั้นนอกครอบทั้งหน้า index ด้วย (เช่น cache ทั้งหน้ารวม header/footer) ก็ทำได้โดยห่อด้วย
> `cache` block อีกชั้นตามรูปแบบ Step 624 ได้เหมือนกัน

---

## Step 630: HTTP Caching — `fresh_when`, `stale?`, ETag, `Cache-Control`

ทุกเทคนิคที่ผ่านมาเป็น caching **ฝั่งใน Rails process** (ลด query/การคำนวณ/การ render) แต่ยัง
มี caching อีกชั้นที่อยู่**นอก Rails process ไปเลย** คือ **HTTP caching** — บอก browser หรือ
CDN ที่อยู่หน้า Rails ว่า "เนื้อหานี้ยังไม่เปลี่ยน ไม่ต้องขอข้อมูลใหม่จากฉันเลยก็ได้" ทำให้
บาง request **ไม่ไปถึง Rails process ด้วยซ้ำ** (เร็วที่สุดเท่าที่จะเป็นไปได้ เพราะไม่มีการ
ประมวลผลใดๆ ฝั่ง server เลย)

### `fresh_when` — วิธีที่ง่ายที่สุดในการเปิด Conditional GET

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])

    fresh_when @post, public: true
  end
end
```

`fresh_when @post` ทำ 2 อย่างให้อัตโนมัติ:

1. คำนวณ **ETag** (ลายเซ็นของเนื้อหา) จาก `@post` (ใช้ `cache_key_with_version` เบื้องหลัง
   เช่นเดียวกับ fragment caching) และแนบ header `ETag` กับ `Last-Modified` ไปกับ response
2. ถ้า request ที่เข้ามามี header `If-None-Match` ตรงกับ ETag ปัจจุบัน (แปลว่า browser/CDN
   มีสำเนาที่ up-to-date อยู่แล้ว) **ตอบกลับ `304 Not Modified` ทันทีโดยไม่ render view เลย**

ทดสอบจริง — request แรก (ไม่มี conditional header):

```bash
$ curl -sD - -o /dev/null http://localhost:3000/posts/1 | grep -iE "etag|last-modified|cache-control"
etag: W/"40a21505bb0640da22adbd2deb849470"
last-modified: Sat, 26 Sep 2026 08:17:16 GMT
cache-control: public
```

ส่ง request ครั้งที่สองพร้อม `If-None-Match` ที่ตรงกับ ETag ที่ได้มา:

```bash
$ curl -sD - -o /dev/null -H 'If-None-Match: W/"40a21505bb0640da22adbd2deb849470"' http://localhost:3000/posts/1
HTTP/1.1 304 Not Modified
etag: W/"40a21505bb0640da22adbd2deb849470"
```

log ฝั่ง server ยืนยันว่า**ไม่มีการ render view เลย**:

```
Completed 304 Not Modified in 2ms (ActiveRecord: 0.2ms (1 query, 0 cached) | GC: 0.0ms)
```

เทียบกับ request ที่ส่ง ETag ผิด (จำลองว่า browser มีสำเนาเก่าที่ไม่ตรงแล้ว):

```bash
$ curl -sD - -o /dev/null -H 'If-None-Match: W/"stale"' http://localhost:3000/posts/1
HTTP/1.1 200 OK
etag: W/"40a21505bb0640da22adbd2deb849470"
```

ได้ `200 OK` พร้อม render เนื้อหาปกติ (และ ETag ใหม่ที่ถูกต้องกลับไปให้ client เก็บไว้ใช้ครั้ง
ถัดไป)

### `stale?` — เวอร์ชันที่ยืดหยุ่นกว่าสำหรับ conditional render

`fresh_when` เหมาะกับ action ที่ render เนื้อหาแบบไม่มีเงื่อนไข ส่วน `stale?` ให้ควบคุมได้ว่า
"ถ้ายังไม่ stale ให้หยุดทำงานเลย" แบบ early return:

```ruby
def show
  @post = Post.find(params[:id])

  if stale?(@post, public: true)
    # โค้ดส่วนนี้ (เช่น query ข้อมูลเพิ่มเติมที่หนัก) จะรันก็ต่อเมื่อข้อมูลจริงๆ เปลี่ยนแล้วเท่านั้น
    @related_posts = Post.where.not(id: @post.id).limit(3)
    render :show
  end
  # ถ้า stale?(@post) เป็น false Rails จะตอบ 304 ให้อัตโนมัติ โดยไม่รันโค้ดหลังจากนี้เลย
end
```

ต่างจาก `fresh_when` ตรงที่ `stale?` คืนค่า `true`/`false` ให้เราเขียน logic ต่อเองได้ —
เหมาะกับกรณีที่มี query เพิ่มเติมที่หนักและอยากข้ามไปเลยถ้ารู้อยู่แล้วว่าจะตอบ 304

### `Cache-Control` — ความแตกต่างระหว่าง `public` และ `private`

```ruby
fresh_when @post, public: true   # อนุญาตให้ CDN/proxy กลาง (เช่น Cloudflare) cache แทนคนอื่นได้ด้วย
fresh_when @post                 # default คือ private — cache ได้เฉพาะใน browser ของผู้ใช้คนนั้นเท่านั้น
```

`public: true` เหมาะกับเนื้อหาที่**ทุกคนเห็นเหมือนกัน** (เช่น หน้าบทความสาธารณะ) ส่วนหน้าที่มี
เนื้อหาเฉพาะ user (เช่น "ตะกร้าสินค้าของฉัน") **ห้ามตั้ง `public: true` เด็ดขาด** เพราะ shared
proxy/CDN อาจนำ response ของ user คนหนึ่งไปเสิร์ฟให้ user อีกคนโดยไม่ตั้งใจ — เป็นช่องโหว่ความ
ปลอดภัยร้ายแรง (ข้อมูลรั่วไหลข้าม user) รายละเอียดเรื่อง cache-related security issue แบบเต็ม
รูปแบบจะกลับมาพูดถึงอีกครั้งใน **Part 079 (OWASP Top 10)**

> **สรุปตำแหน่งของ HTTP caching เทียบกับสิ่งที่เรียนมาทั้งบท:** fragment/low-level caching
> (Step 623–629) ลดงานที่ **Rails process** ต้องทำ (query, คำนวณ, render) แต่ request ยังคง
> วิ่งมาถึง Rails เสมอ ส่วน HTTP caching ทำให้ request **บางส่วนไม่ต้องวิ่งมาถึง Rails เลย** —
> ทั้งสองชั้นทำงานร่วมกันได้ดี ไม่ใช่ทางเลือกที่ต้องเลือกอย่างใดอย่างหนึ่ง ในทางปฏิบัติทีมส่วน
> ใหญ่จะ hand-tune fragment/low-level caching มากกว่า เพราะควบคุม granularity ได้ละเอียดกว่า
> ส่วน HTTP caching มักตั้งค่าไว้กว้างๆ (เช่น asset ที่มี fingerprint ตั้ง cache ยาวเป็นปี ตามที่
> เห็นใน `config/environments/production.rb` ตั้งแต่ Part 024) แล้วปล่อยผ่านมากกว่าจะมาไล่จูน
> ทีละหน้า

---

## แบบฝึกหัด: Russian Doll Caching เต็มรูปแบบสำหรับหน้า Post + Comments

### โจทย์

สร้างหน้า Blog แบบเดียวกับที่ใช้สาธิตตลอด Part นี้ (ต่อยอดจากแอป Blog ใน Part 030) โดยกำหนด
ให้:

1. ตั้งค่า `config.cache_store = :solid_cache_store` ทั้งใน development และ production
   (production มีให้อยู่แล้วจาก generator) แล้วเพิ่มตาราง `solid_cache_entries` เข้าไปใน
   ฐานข้อมูล development ด้วย migration
2. เปิด caching ใน development ด้วย `bin/rails dev:cache`
3. `Comment belongs_to :post, touch: true` เพื่อให้แก้ comment กระทบ `post.updated_at`
4. หน้า `posts#show` ต้อง cache ด้วย Russian Doll pattern: ชั้นนอก cache ทั้งโพสต์
   (`cache @post do`) ชั้นในใช้ collection caching สำหรับ comment
   (`render partial:, collection:, cached: true`)
5. พิสูจน์ด้วย log จริงว่า: (ก) รอบแรกทุกอย่าง miss (ข) รอบสองทุกอย่าง hit (ค) หลังแก้ไข
   comment 1 อันจากทั้งหมด 3 อัน มีแค่ comment นั้นกับ fragment ชั้นนอกที่ miss ส่วน comment
   อีก 2 อันยังคง hit

### เฉลย

**Model:**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end
```

**Migration เพิ่มตาราง solid_cache เข้า development database:**

```ruby
# db/migrate/xxxx_create_solid_cache_entries_for_dev.rb
class CreateSolidCacheEntriesForDev < ActiveRecord::Migration[8.1]
  def change
    create_table :solid_cache_entries, force: :cascade do |t|
      t.binary :key, limit: 1024, null: false
      t.binary :value, limit: 536870912, null: false
      t.datetime :created_at, null: false
      t.integer :key_hash, limit: 8, null: false
      t.integer :byte_size, limit: 4, null: false
      t.index [:byte_size], name: "index_solid_cache_entries_on_byte_size"
      t.index [:key_hash, :byte_size], name: "index_solid_cache_entries_on_key_hash_and_byte_size"
      t.index [:key_hash], name: "index_solid_cache_entries_on_key_hash", unique: true
    end
  end
end
```

**Config:**

```ruby
# config/environments/development.rb
config.cache_store = :solid_cache_store
```

```bash
bin/rails db:migrate
bin/rails dev:cache
```

**Controller:**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    fresh_when @post, public: true
  end
end
```

**Views:**

```erb
<%# app/views/posts/show.html.erb %>
<% cache @post do %>
  <h1><%= @post.title %></h1>
  <p><%= @post.body %></p>
  <p>Post updated_at: <%= @post.updated_at.to_fs(:inspect) %></p>

  <h2>Comments (<%= @post.comments.size %>)</h2>

  <%= render partial: "comments/comment", collection: @post.comments.order(:id), cached: true %>
<% end %>
```

```erb
<%# app/views/comments/_comment.html.erb %>
<div class="comment" id="comment_<%= comment.id %>">
  <p><%= comment.body %></p>
  <small>comment updated_at: <%= comment.updated_at.to_fs(:inspect) %></small>
</div>
```

**Seed ข้อมูลทดสอบ (โพสต์ 1 อัน มี comment 3 อัน):**

```ruby
# db/seeds.rb
post = Post.create!(title: "เริ่มต้นกับ Rails Caching", body: "...")
post.comments.create!(body: "บทความดีมากครับ อ่านเข้าใจง่าย")
post.comments.create!(body: "รอตอนต่อไปเรื่อง Russian Doll Caching เลย")
post.comments.create!(body: "ขอบคุณสำหรับตัวอย่าง log จริงด้วยครับ")
```

### ผลลัพธ์ที่ทดสอบจริงบน Rails 8.1.4 — พิสูจน์ Selective Invalidation

**(ก) รอบแรก เข้า `/posts/1` (cache ว่างทั้งหมด):**

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (8.1ms)
  Rendered collection of comments/_comment.html.erb [0 / 3 cache hits] (Duration: 11.2ms | GC: 1.3ms)
Write fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (1.5ms)
```

miss ทั้งชั้นนอกและ comment ทั้ง 3 อัน (สมเหตุสมผล — ยังไม่เคย cache มาก่อน)

**(ข) รอบสอง เข้าซ้ำ (ข้อมูลไม่เปลี่ยน):**

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081656502401 (5.6ms)
```

hit ทันทีตั้งแต่ชั้นนอก ไม่มีการแตะ comment เลยแม้แต่ query เดียว

**(ค) แก้ไข comment 1 อันด้วย console:**

```irb
irb> c = Comment.find(1)
irb> c.update!(body: "แก้ไขคอมเมนต์นี้ใหม่ เพื่อทดสอบ Russian Doll Caching")
irb> c.post.reload.updated_at
=> Sat, 26 Sep 2026 08:17:16 UTC +00:00   # เปลี่ยนแล้ว เพราะ touch: true cascade ขึ้นมา
```

เข้า `/posts/1` อีกครั้ง:

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (1.1ms)
  Rendered collection of comments/_comment.html.erb [2 / 3 cache hits] (Duration: 3.0ms | GC: 0.0ms)
Write fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (2.2ms)
```

**หลักฐานยืนยันครบทุกข้อที่โจทย์ต้องการ:**

- Key ของ fragment ชั้นนอกเปลี่ยนไปตาม `post.updated_at` ใหม่ → miss ตามคาด (โครงสร้างหน้า
  เปลี่ยนจริง)
- Collection ของ comment แสดง **`[2 / 3 cache hits]`** — มีแค่ comment ที่เพิ่งแก้ไข **1 อัน**
  ที่ต้อง render ใหม่ ส่วนอีก **2 อันที่ไม่เกี่ยวข้องยังคง cache hit** ไม่ถูกแตะต้องเลย
- ยืนยันเพิ่มเติมด้วย SQL log: มีคำสั่ง `SELECT ... WHERE key_hash IN (...)` แค่ครั้งเดียวที่
  อ่าน cache key ของ comment ทั้ง 3 อันพร้อมกัน (multi-get แบบ batch) ตามด้วย
  `INSERT ... ON CONFLICT DO UPDATE` แค่ **1 แถว** เท่านั้น (ไม่ใช่ 3 แถว)

เข้าซ้ำอีกครั้งหลังจากนี้ กลับมาเป็น full hit ปกติ:

```
Read fragment views/posts/show:0838ce4e9fd25c9477f29e4d2bacbcee/posts/1-20260926081716956590 (1.1ms)
```

**สรุปแบบฝึกหัด:** Russian Doll Caching ทำงานตามที่ออกแบบไว้ทุกประการ — การแก้ไขข้อมูลเล็กๆ
1 จุด (comment เดียว) ทำให้เกิดงาน re-render/re-write cache แค่ที่จุดนั้นกับ fragment ชั้นนอก
ที่ครอบมันอยู่เท่านั้น ไม่กระทบ fragment อื่นๆ ที่ไม่เกี่ยวข้องเลยแม้แต่น้อย — นี่คือเหตุผลที่
เทคนิคนี้ scale ได้ดีกับหน้าที่มี comment/รายการย่อยจำนวนมาก

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มชั้นบนสุดอีกชั้นให้กับหน้า `posts#index`: ห่อทั้งหน้าด้วย `cache` block ที่ผูกกับ
   "โพสต์ล่าสุดที่ถูกแก้ไข" (เช่น `cache ["posts_index", Post.maximum(:updated_at)] do`)
   ครอบ collection caching ของ post card ที่มีอยู่แล้วอีกที ทดสอบว่าการแก้ไขโพสต์ไหนก็ตาม
   invalidate ชั้นบนสุดนี้ แต่ post card อื่นๆ ที่ไม่เกี่ยวข้องยังคง cache hit เหมือนเดิมหรือไม่
   (ใบ้: สังเกตว่าตอนนี้มี cache ซ้อนกัน 2 ชั้นในหน้าเดียว — ชั้นบนสุดควบคุมว่าจะ render
   collection ใหม่ทั้งหมดหรือเปล่า ส่วนชั้น collection caching ควบคุมว่าแต่ละ card ไหนต้อง
   render ใหม่จริงๆ)
2. ลองเปลี่ยน low-level cache ในตัวอย่าง `ReportsController#expensive` ให้ cache แยกตาม
   `current_user` (สมมติว่ารายงานต่างกันไปตาม user ที่ login) ออกแบบ cache key ให้ปลอดภัย
   ไม่ให้ user คนหนึ่งเห็นรายงานของอีกคนโดยไม่ตั้งใจ (ทบทวนหลักการ Step 627)
3. ลองสลับ `config.cache_store` เป็น `:memory_store` ชั่วคราว แล้วรัน server 2 instance
   พร้อมกันที่คนละ port (`bin/rails server -p 3000` กับ `-p 3001`) เขียน cache จาก instance
   หนึ่ง แล้วลองอ่านจากอีก instance หนึ่งดูว่าเห็นค่าเดียวกันหรือไม่ — เทียบกับการสลับกลับไปใช้
   `:solid_cache_store` แล้วทำซ้ำแบบเดียวกัน อธิบายว่าทำไมผลลัพธ์ถึงต่างกัน และเชื่อมโยงกับ
   เหตุผลที่ production ที่รันหลาย process/server ควรใช้ cache store ที่ share ข้อมูลได้จริง
4. (ท้าทายขึ้น) ลองเขียน request spec (ทบทวนจาก Part 046) ที่ตรวจสอบว่า `PostsController#show`
   ตอบ `304 Not Modified` เมื่อส่ง `If-None-Match` ที่ตรงกับ ETag ของ response ก่อนหน้า และตอบ
   `200 OK` เมื่อโพสต์ถูกแก้ไขหลังจากนั้น (ใบ้: ต้องเปิด `config.action_controller.perform_caching`
   หรือ mock ให้เหมาะสมกับ test environment ซึ่ง default เป็น `:null_store`)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปัญหาที่ caching แก้: การคำนวณ/render **ค่าเดิมซ้ำๆ** ทั้งที่ข้อมูลต้นทางยังไม่เปลี่ยน
  เป็นการสิ้นเปลือง resource โดยเปล่าประโยชน์
- รู้จัก cache store ทั้งหมดที่ Rails รองรับ (`:solid_cache_store`, `:memory_store`,
  `:file_store`, `:redis_cache_store`, `:mem_cache_store`, `:null_store`) พร้อมจุดเด่น/
  ข้อจำกัดของแต่ละตัว และเข้าใจว่า **solid_cache คือ default ใหม่ของ Rails 8 บน production**
  (DB-backed, ไม่ต้องมี Redis/Memcached แยก) ในขณะที่ development ยัง default เป็น
  `:memory_store` อยู่
- เปิด caching ใน development ด้วย `bin/rails dev:cache` (ปิดอยู่โดย default เพื่อไม่ให้
  รบกวนการเห็นผลของโค้ดที่แก้ระหว่างพัฒนา) และรู้จักประวัติของ Page/Action Caching ที่ถูก
  ถอดออกจาก Rails core ไปแล้ว
- ใช้ **Fragment Caching** (`<% cache @post do %>`) และเข้าใจที่มาของ cache key แบบเต็ม
  (template digest + `cache_key_with_version` ที่ผูกกับ `updated_at`)
- ทำ **Russian Doll Caching** ได้จริง — ซ้อน cache หลายชั้นสำหรับ post + comments โดยใช้
  `touch: true` (ต่อยอดจาก Part 035) ให้การแก้ไข child cascade ไปทำให้ fragment ของ parent
  invalidate ตามโดยอัตโนมัติ
- **พิสูจน์ด้วย log จริง** ว่าการแก้ไขข้อมูลชิ้นเดียวทำให้เกิดการ invalidate แบบเฉพาะจุด
  ไม่กระทบ fragment อื่นที่ไม่เกี่ยวข้อง (`[2 / 3 cache hits]` เป็นหลักฐานสำคัญของบทนี้)
- ใช้ **Low-level Caching** (`Rails.cache.fetch(key, expires_in:) { ... }`) สำหรับงานที่ไม่
  เกี่ยวกับ view เช่น การคำนวณหนักหรือผลลัพธ์จาก external API
- ออกแบบ **cache key ที่ดี**: ใส่ทุกพารามิเตอร์ที่มีผลต่อผลลัพธ์, ใช้ versioned key เมื่อแก้
  logic การคำนวณ, และตั้งชื่อ key แบบ namespace ชัดเจนเพื่อเลี่ยง collision
- ล้าง cache ด้วยมือด้วย `Rails.cache.delete`/`Rails.cache.clear` และเข้าใจอันตรายของ
  `clear` บน production (กระทบทุก feature พร้อมกัน + เสี่ยงเกิด cache stampede)
- ใช้ **Collection Caching** (`render collection:, cached: true`) เพื่ออ่าน cache ของทั้ง
  collection ด้วย query เดียว แทนที่จะเช็คทีละ record
- รู้จัก **HTTP Caching** อีกชั้นหนึ่งที่แยกจาก Rails process ด้วย `fresh_when`/`stale?`,
  ETag และ `Cache-Control` — ทำให้บาง request ไม่ต้องวิ่งมาถึง Rails เลยด้วยซ้ำ

**ต่อไป (Part 064):** เราจะเจาะลึกเรื่อง **Database Performance** — การอ่าน execution plan
ด้วย `EXPLAIN ANALYZE` เพื่อดูว่า query หนึ่งๆ ใช้ index จริงหรือไม่, วิธีออกแบบ index ให้ตรงกับ
pattern การ query ของแอป, และการตรวจจับปัญหา N+1 query แบบอัตโนมัติด้วย gem **Bullet** ที่จะ
เตือนทันทีตอน development เมื่อโค้ดมีความเสี่ยงจะยิง query เกินความจำเป็น — เป็นอีกเครื่องมือ
สำคัญที่ทำงานคู่กับ caching ที่เรียนไปใน Part นี้ เพราะ caching ช่วยซ่อนปัญหา query ช้าได้
ชั่วคราว แต่การแก้ที่ต้นเหตุ (query/index ที่ดี) ยังจำเป็นเสมอสำหรับ request ที่เป็น cache miss
