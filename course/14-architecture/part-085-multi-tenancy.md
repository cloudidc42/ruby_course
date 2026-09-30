# Part 085: Multi-tenancy — Row-based vs Schema-based, Apartment gem

> **Step ครอบคลุมใน Part นี้:** Step 841–850
> **ระดับ:** ขั้นสูง (ต้องผ่าน Part 034 เรื่อง Query Interface/`default_scope`, Part 041 เรื่อง
> Authentication, และ Part 082–084 เรื่อง Service Object/Clean Architecture มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6, Rails 8.1.4, PostgreSQL 16 — ทุกตัวอย่างในเลกเชอร์นี้ทดสอบ
> จริงบนสภาพแวดล้อมนี้ในโปรเจกต์ scratch แยกต่างหาก (ไม่ใช่การเดา)

## สารบัญของ Part นี้

- Step 841: Multi-tenancy คืออะไร และทำไมเป็นรากฐานของ SaaS ทุกตัว
- Step 842: Row-based multi-tenancy — เพิ่ม `organization_id` ในทุกตาราง
- Step 843: อันตรายตัวจริงของ row-based — สาธิตการรั่วไหลของข้อมูลข้ามลูกค้าจริงๆ
- Step 844: แก้ปัญหาด้วย `ActiveSupport::CurrentAttributes` และ automatic scoping ที่ปลอดภัย
- Step 845: Schema-based multi-tenancy — แยก PostgreSQL schema ต่อ tenant
- Step 846: Apartment gem คืออะไร — ตรวจสอบสถานะการดูแลรักษาจริง เทียบกับ `acts_as_tenant`
- Step 847: Database-per-tenant — ระดับการแยกที่สูงสุด และต้นทุนที่ต้องจ่าย
- Step 848: เลือกกลยุทธ์ tenancy จากความต้องการทางธุรกิจจริง ไม่ใช่ทฤษฎี
- Step 849: Subdomain-based tenant resolution — `acme.myapp.com` หา tenant ให้อัตโนมัติ
- Step 850: การเขียนชุดทดสอบ isolation — พิสูจน์ว่า tenant A เห็นข้อมูล tenant B ไม่ได้เด็ดขาด

---

## เตรียมโดเมนสำหรับ Part นี้: `Organization`, `Project`

Part นี้จะสร้างระบบ SaaS ง่ายๆ ชื่อ "Projects" ที่ลูกค้าแต่ละราย (`Organization`) มีชุด
`Project` เป็นของตัวเอง เหมือนเครื่องมือ project-management จริงๆ (Trello, Asana, Linear ฯลฯ)
ที่ทุกบริษัทที่สมัครใช้งานต้อง **แชร์แอปพลิเคชันตัวเดียวกัน** แต่ **ห้ามเห็นข้อมูลของกันและกัน
เด็ดขาด**

ถ้าจะลองทำตามในเครื่องตัวเอง ให้สร้างโปรเจกต์ scratch แยกต่างหาก:

```bash
rails new tenancy_demo --minimal --database=postgresql
cd tenancy_demo
bin/rails db:create

bin/rails generate model Organization name:string subdomain:string:uniq
bin/rails generate model Project organization:references name:string status:string
bin/rails db:migrate
```

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :projects, dependent: :destroy
end
```

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  belongs_to :organization
end
```

---

## Step 841: Multi-tenancy คืออะไร และทำไมเป็นรากฐานของ SaaS ทุกตัว

**Multi-tenancy** คือสถาปัตยกรรมที่แอปพลิเคชัน **หนึ่ง instance เดียว** (โค้ดชุดเดียว, deploy
ครั้งเดียว, server ชุดเดียว) ให้บริการลูกค้าหลายราย (**tenant**) พร้อมกัน โดยแต่ละ tenant
มองเห็นแค่ข้อมูลของตัวเอง เสมือนแต่ละรายมีแอปเป็นของตัวเองส่วนตัว

คำว่า "tenant" (ผู้เช่า) มาจากการเปรียบเทียบกับอาคารอพาร์ตเมนต์ — **อาคารเดียวกัน** (แอปพลิเคชัน)
มี**ผู้เช่าหลายราย** (ลูกค้า/บริษัท) แต่ละรายมีห้องของตัวเอง (ข้อมูลของตัวเอง) ที่ผู้เช่ารายอื่น
เข้าไม่ได้

ลองเทียบกับทางเลือกตรงข้าม — **single-tenancy** (แยก instance/database ต่อลูกค้าหนึ่งราย
โดยสมบูรณ์ ตั้งแต่ server ไปจนถึง codebase ที่ deploy แยกกัน):

| | Single-tenancy | Multi-tenancy |
|---|---|---|
| จำนวน deploy ที่ต้องดูแล | 1 ต่อลูกค้า 1 ราย (ลูกค้า 1,000 ราย = 1,000 deploy) | 1 ชุดสำหรับลูกค้าทั้งหมด |
| ต้นทุน infrastructure | สูงมาก เพิ่มเป็นเส้นตรงตามจำนวนลูกค้า | ต่ำ ใช้ทรัพยากรร่วมกันอย่างมีประสิทธิภาพ |
| การ deploy ฟีเจอร์ใหม่ | ต้อง deploy ซ้ำทุก instance | deploy ครั้งเดียวได้ทุกลูกค้า |
| ระดับ isolation ของข้อมูล | สมบูรณ์แบบ (แยกกันจริงๆ ตั้งแต่ต้น) | ต้อง**ออกแบบและพิสูจน์**ว่า isolation ทำงานถูกต้อง |

บริษัท SaaS แทบทุกราย (Slack, Notion, Shopify, Basecamp, GitHub เวอร์ชัน organization) เลือก
**multi-tenancy** เพราะต้นทุนการดำเนินงานต่ำกว่ามหาศาล และการออกฟีเจอร์ใหม่ทำได้เร็วกว่าเยอะ —
แต่สิ่งที่ต้องแลกมาคือ **ทีมพัฒนาต้องรับผิดชอบเต็มๆ ว่าข้อมูลของลูกค้าแต่ละรายจะไม่มีทางรั่วไหล
ไปหาอีกรายได้** ซึ่งเป็นโจทย์ทางวิศวกรรมที่จริงจังกว่าที่คิด — และเป็นหัวใจของ Part นี้ทั้งหมด

มีวิธีทำ multi-tenancy หลักๆ 3 ระดับ เรียงจาก isolation ต่ำสุด (แต่ทำง่ายและถูกที่สุด) ไปสูงสุด
(แต่ซับซ้อนและแพงที่สุด):

1. **Row-based (shared schema)** — ทุก tenant ใช้ตารางเดียวกันในฐานข้อมูลเดียวกัน แยกกันด้วย
   คอลัมน์ `organization_id` (Step 842–844)
2. **Schema-based** — แต่ละ tenant มี PostgreSQL schema ของตัวเองในฐานข้อมูลเดียวกัน
   (Step 845–846)
3. **Database-per-tenant** — แต่ละ tenant มีฐานข้อมูลจริงแยกกันโดยสมบูรณ์ (Step 847)

---

## Step 842: Row-based multi-tenancy — เพิ่ม `organization_id` ในทุกตาราง

**Row-based multi-tenancy** (เรียกอีกชื่อว่า **shared schema**) คือแนวทางที่ง่ายและนิยมที่สุด
สำหรับเริ่มต้น SaaS: ทุกตารางที่มีข้อมูลเฉพาะของ tenant จะมีคอลัมน์ `organization_id` (หรือ
`tenant_id`) และทุก query ต้อง `WHERE organization_id = ?` เพื่อกรองให้เห็นเฉพาะข้อมูลของ
tenant ตัวเอง

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  belongs_to :organization
end
```

```
Table "public.projects"
    Column     |  Type
----------------+---------
 id             | bigint
 organization_id | bigint   <- ทุก row บอกว่าเป็นของ tenant ไหน
 name           | varchar
 status         | varchar
```

ข้อดีของ row-based:

- **implement ง่ายที่สุด** — แค่เพิ่มคอลัมน์และเขียน `where(organization_id: ...)`
- **migration ใช้ร่วมกันได้ทุก tenant** — รัน `rails db:migrate` ครั้งเดียว ทุก tenant ได้
  schema ใหม่พร้อมกัน (ต่างจาก schema-based ที่ต้องรันซ้ำทุก schema — ดู Step 845)
- **connection pool เดียวพอ** — เชื่อมต่อฐานข้อมูลเดียว ไม่ต้องจัดการ connection แยกตาม tenant
- **query ข้าม tenant ทำได้ง่าย** — เช่น analytics ระดับแพลตฟอร์ม (`Project.count` รวมทุก
  tenant สำหรับ admin dashboard) ทำได้ในคำสั่งเดียว ไม่ต้องวน loop ยิงหลายฐานข้อมูล

ลองสาธิตแบบตรงไปตรงมาที่สุดก่อน (**ยังไม่ปลอดภัย** — จะเห็นปัญหาใน Step 843):

```ruby
acme   = Organization.create!(name: "Acme Corp",  subdomain: "acme")
globex = Organization.create!(name: "Globex Inc", subdomain: "globex")

acme.projects.create!(name: "Acme Secret Launch Plan", status: "active")
acme.projects.create!(name: "Acme Q4 Budget",          status: "active")
globex.projects.create!(name: "Globex M&A Dossier",    status: "active")

# ทุก query ที่จะแสดงผลให้ผู้ใช้เห็น ต้องกรองด้วย organization_id เสมอ
Project.where(organization_id: acme.id).pluck(:name)
# => ["Acme Secret Launch Plan", "Acme Q4 Budget"]
```

ดูเผินๆ ก็ใช้งานได้ปกติ — **แต่คำว่า "ทุก query ที่จะแสดงผลให้ผู้ใช้เห็น ต้องกรอง" นี่แหละคือ
จุดอ่อนที่อันตรายที่สุดของ row-based multi-tenancy** ซึ่งจะสาธิตให้เห็นจริงใน Step ถัดไป

---

## Step 843: อันตรายตัวจริงของ row-based — สาธิตการรั่วไหลของข้อมูลข้ามลูกค้าจริงๆ

Row-based multi-tenancy มีจุดอ่อนที่ร้ายแรงระดับ **security vulnerability class** ของตัวเอง:
**query ที่ลืมใส่ `where(organization_id: ...)` เพียงจุดเดียวในทั้งโปรเจกต์ ก็เพียงพอที่จะทำให้
ข้อมูลลูกค้ารายหนึ่งรั่วไหลไปให้ลูกค้าอีกรายเห็นได้ทันที** — ไม่มี error ไม่มี exception เกิดขึ้น
เลย เพราะในทางเทคนิคนี่คือ query ที่ "ถูกต้อง" ทุกประการ มันแค่ไม่ได้กรองเงื่อนไขที่ควรกรอง

มาสาธิตสถานการณ์ที่เกิดขึ้นจริงในโค้ด production บ่อยมาก: engineer คนแรกเขียนหน้า dashboard
ถูกต้องตามหลัก แต่ **engineer อีกคนในอีกหลายเดือนต่อมา** มาเพิ่มฟีเจอร์ widget "โปรเจกต์ที่เพิ่ง
active ล่าสุด" แล้วลืมใส่เงื่อนไข tenant (เป็นความผิดพลาดที่สมเหตุสมผลมาก เพราะโค้ดยัง compile
ผ่าน, test อื่นยัง pass, และหน้าตาผลลัพธ์ก็ดูเหมือนทำงานถูกต้องถ้า test data มีแค่ tenant เดียว):

```ruby
class NaiveTenantContext
  class << self
    attr_accessor :current_organization
  end
end

# Request จาก Acme: controller เก็บ "tenant ปัจจุบัน" ไว้ในตัวแปรธรรมดา
NaiveTenantContext.current_organization = acme

# Action แรก เขียนถูกต้อง มีการกรอง tenant
dashboard_projects = Project.where(organization_id: NaiveTenantContext.current_organization.id)
dashboard_projects.pluck(:name)
# => ["Acme Secret Launch Plan", "Acme Q4 Budget"]

# Action "recent projects widget" ที่ engineer อีกคนเขียนเพิ่มทีหลัง -- ลืมกรอง organization_id
def recent_projects_widget
  Project.order(created_at: :desc).limit(5)
end

leaked = recent_projects_widget
leaked.pluck(:name)
```

ผลลัพธ์จริงที่ได้เมื่อรันโค้ดนี้ในโปรเจกต์ scratch:

```
Acme dashboard (correctly scoped): ["Acme Secret Launch Plan", "Acme Q4 Budget"]
'Recent projects' widget shown to Acme user (BUG): ["Globex M&A Dossier", "Acme Q4 Budget", "Acme Secret Launch Plan"]
Globex data present in Acme's view? true
```

**นี่คือบั๊กจริงจังระดับ data breach** — ผู้ใช้ของ Acme เห็นชื่อโปรเจกต์ลับของ Globex (คู่แข่ง
ที่อาจอยู่ในตลาดเดียวกันด้วยซ้ำ) ปรากฏอยู่ในหน้า widget ของตัวเอง โดยไม่มีทาง error ใดๆ เตือนเลย
สังเกตว่าโค้ดของ `recent_projects_widget` ไม่ผิด syntax ไม่ผิด logic ในแง่ Ruby แม้แต่นิดเดียว —
มันแค่ **"ลืม" เงื่อนไขทางธุรกิจที่สำคัญที่สุดข้อหนึ่งของทั้งระบบ**

### ทำไมปัญหานี้ถึงหลีกเลี่ยงยากในทางปฏิบัติ

- **ไม่มีสัญญาณเตือนใดๆ ในระดับภาษา** — Ruby/Rails ไม่รู้จักแนวคิด "tenant" เอง มันเป็นแค่
  `WHERE` clause ธรรมดาที่โปรแกรมเมอร์ต้องเขียนเอง ทุกจุด ด้วยมือ
- **โค้ดที่ผิดกับโค้ดที่ถูกหน้าตาเหมือนกันเป๊ะ** — `Project.order(created_at: :desc).limit(5)`
  กับ `Project.where(organization_id: current_org.id).order(created_at: :desc).limit(5)` ต่างกัน
  แค่ประโยคเดียว ไม่มี type error ไม่มี lint warning มาตรฐานที่จับได้อัตโนมัติ
  (Brakeman ที่เรียนใน Part 080 ก็จับ pattern นี้ไม่ได้โดยตรง เพราะมันเป็นปัญหาเชิง business
  logic ไม่ใช่ปัญหา syntax ที่รู้จักกันทั่วไป)
- **จุดที่ลืมอาจไม่ใช่ controller เท่านั้น** — background job, admin panel, export CSV,
  API endpoint, console script ที่ engineer รันมือ ฯลฯ ล้วนเป็นจุดเสี่ยงที่ต้องกรอง
  `organization_id` เหมือนกันหมด และยิ่งจำนวนจุดมากเท่าไหร่ โอกาสลืมยิ่งสูงขึ้นเรื่อยๆ
- **Test มักไม่จับปัญหานี้** — ถ้า test suite สร้างข้อมูลของ tenant เดียวเสมอ (ซึ่งเป็นเรื่อง
  ปกติที่ทำกันเพื่อความง่าย) query ที่ลืมกรองจะให้ผลลัพธ์ "ถูกต้องบังเอิญ" ในทุก test เพราะไม่มี
  tenant ที่สองให้รั่วไหลไปหา (Step 850 จะแก้ปัญหานี้โดยตรงด้วยชุดทดสอบ isolation)

> **สรุปความอันตราย:** row-based multi-tenancy ผลักภาระความถูกต้องทั้งหมดไปไว้ที่ **วินัยของ
> โปรแกรมเมอร์ทุกคนในทุก query ตลอดอายุของโปรเจกต์** ซึ่งในทางปฏิบัติเป็นสมมติฐานที่พังได้ง่าย
> มาก นี่คือเหตุผลที่ multi-tenancy leak ติดอันดับต้นๆ ของช่องโหว่ร้ายแรงที่พบใน SaaS จริง
> (คล้ายกับ IDOR — Insecure Direct Object Reference — ที่เจาะลึกใน Part 079 แต่ scope กว้างกว่า
> เพราะกระทบได้ทั้งตาราง ไม่ใช่แค่ record เดียว)

---

## Step 844: แก้ปัญหาด้วย `ActiveSupport::CurrentAttributes` และ automatic scoping ที่ปลอดภัย

ทางแก้ที่ถูกต้องไม่ใช่ "จำให้ได้ทุกจุด" (ซึ่งพิสูจน์แล้วว่าพังได้) แต่คือ **ทำให้ระบบกรอง
`organization_id` ให้อัตโนมัติที่ระดับ query layer** เพื่อให้ query ที่ "ลืม" เขียน `where`
ยังคงปลอดภัยอยู่ดี — โดยใช้กลไกที่คุ้นเคยจาก Part 034: **`default_scope`**

แต่ Part 034 (Step 337) ก็สอนไปแล้วว่า `default_scope` แบบเขียนเองมีกับดักอันตรายมาก: มันชน
กับ `find`, ซ่อนพฤติกรรมไว้ไม่ให้เห็น, และหน้า admin ที่ควรเห็นข้อมูลทั้งหมดจะถูกกรองไปเงียบๆ
ด้วย — คำถามคือ แล้วจะใช้ automatic scoping แบบปลอดภัยได้อย่างไร?

คำตอบคือ **ไม่เขียน `default_scope` มือเปล่าเอง** แต่ใช้ไลบรารีที่ผ่านการทดสอบและแก้ปัญหาพวกนี้
มาแล้วอย่างละเอียด อย่าง gem [`acts_as_tenant`](https://github.com/ErwinM/acts_as_tenant)
ซึ่งภายในก็ใช้กลไก `default_scope` เหมือนกัน **แต่เพิ่ม safety net ที่การเขียนมือมักลืมใส่:**

1. **raise error แทนการคืนค่าว่างเงียบๆ** เมื่อไม่มีการตั้ง tenant ปัจจุบันไว้ (ป้องกันปัญหา
   ตรงข้ามกับ Step 843 — คือป้องกันไม่ให้ query ที่ควรมี tenant กลับไม่มี)
2. **มี escape hatch ที่ตั้งใจเรียกเท่านั้น** (`ActsAsTenant.without_tenant`) แทนที่จะให้ query
   ธรรมดาข้าม scope ไปได้โดยไม่มีการประกาศเจตนา
3. **validate ว่า record จะถูก assign ให้ tenant อื่นไม่ได้** แม้จะพยายามตั้ง `organization_id`
   ตรงๆ ก็ตาม

### ติดตั้งและตั้งค่า `acts_as_tenant`

```ruby
# Gemfile
gem "acts_as_tenant"
```

```bash
bundle install
```

```ruby
# config/initializers/acts_as_tenant.rb
ActsAsTenant.configure do |config|
  # บังคับให้ทุก query ของ model ที่ acts_as_tenant ต้องมี current_tenant เสมอ
  # ถ้าไม่มีจะ raise ActsAsTenant::Errors::NoTenantSet แทนที่จะคืนค่าว่างๆ อย่างเงียบๆ
  config.require_tenant = true
end
```

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  acts_as_tenant :organization
end
```

หนึ่งบรรทัด `acts_as_tenant :organization` ทำสิ่งเหล่านี้ให้อัตโนมัติ:
- ประกาศ `belongs_to :organization` ให้ (ไม่ต้องเขียนเองซ้ำ)
- เพิ่ม `default_scope` ที่กรอง `organization_id` ตาม **tenant ปัจจุบัน** ให้ทุก query
- เพิ่ม validation ป้องกันการ assign record ให้ tenant ที่ไม่ใช่ tenant ปัจจุบัน
- ตอนสร้าง record ใหม่ เติม `organization_id` ให้อัตโนมัติจาก tenant ปัจจุบัน ถ้ายังไม่ได้ระบุ

### `current_tenant` เก็บไว้ที่ไหน — `ActiveSupport::CurrentAttributes`

หัวใจสำคัญที่สุดคือ **`current_tenant` ต้องเป็นค่าที่ผูกกับ request ปัจจุบันเท่านั้น** ไม่ใช่
ตัวแปร global ธรรมดาแบบ `NaiveTenantContext` ใน Step 843 (ซึ่งถ้าเป็น production server ที่มี
หลาย thread ประมวลผลพร้อมกัน ตัวแปร class-level ธรรมดาจะถูก request อื่นเขียนทับกลางคัน เกิด
เป็น race condition ที่ร้ายแรงยิ่งกว่าเดิม — request ของ Acme อาจได้ tenant ของ Globex ไปเพราะ
thread อื่นเปลี่ยนค่าตัวแปร global แทรกเข้ามา)

`acts_as_tenant` แก้ปัญหานี้ด้วย **`ActiveSupport::CurrentAttributes`** — กลไกมาตรฐานของ Rails
เองสำหรับเก็บ "ค่าที่ผูกกับ request/job ปัจจุบัน" อย่างปลอดภัยทั้งในแง่ thread และ fiber (ดู
source code จริงของ gem):

```ruby
# ภายในซอร์สของ acts_as_tenant (lib/acts_as_tenant.rb)
class Current < ActiveSupport::CurrentAttributes
  attribute :current_tenant, :acts_as_tenant_unscoped, :acts_as_tenant_mutable

  resets { ActsAsTenant.configuration.tenant_change_hook&.call(nil) }
end
```

`ActiveSupport::CurrentAttributes` ถูกออกแบบมาให้ **reset ค่าอัตโนมัติทุกครั้งที่ request หรือ
job จบลง** (Rails เรียก `.reset_all` ให้เองตอนท้าย request/job) จึงไม่มีทางที่ค่าจาก request
ก่อนหน้าจะ "รั่ว" ข้ามไปยัง request ถัดไปที่ใช้ thread เดียวกัน (ปัญหาคลาสสิกของตัวแปร global
แบบ Step 843) — นี่คือเหตุผลที่โจทย์บอกว่าใช้กลไก "thread-local/request-scoped context" แทนที่
จะเป็นตัวแปรธรรมดา

### พิสูจน์ว่าบั๊กเดิมใน Step 843 ถูกแก้แล้วจริง

รันโค้ดชุดเดียวกับ Step 843 เป๊ะๆ (ฟังก์ชัน `recent_projects_widget` ที่ "ลืม" กรอง
`organization_id` เหมือนเดิมทุกตัวอักษร) แต่เปลี่ยนวิธีตั้ง tenant:

```ruby
ActsAsTenant.with_tenant(acme) do
  Project.all.pluck(:name)
  # => ["Acme Secret Launch Plan", "Acme Q4 Budget"]

  # ฟังก์ชันเดิมจาก Step 843 เป๊ะๆ -- "ลืม" กรอง organization_id เหมือนเดิม
  def recent_projects_widget
    Project.order(created_at: :desc).limit(5)
  end

  leaked = recent_projects_widget
  leaked.pluck(:name)
end
```

ผลลัพธ์จริงที่ได้:

```
Acme dashboard: ["Acme Secret Launch Plan", "Acme Q4 Budget"]
'Recent projects' widget shown to Acme user: ["Acme Q4 Budget", "Acme Secret Launch Plan"]
Globex data present in Acme's view? false
```

**โค้ดของ widget ไม่ได้ถูกแก้ไขแม้แต่ตัวอักษรเดียว** — มันยัง "ลืม" กรอง `organization_id`
เหมือนเดิม แต่เพราะ `default_scope` ที่ `acts_as_tenant` เติมให้ **ทำงานที่ชั้น query layer
โดยอัตโนมัติ ไม่ขึ้นกับว่าโปรแกรมเมอร์เขียน `where` ครบหรือไม่** บั๊กประเภทนี้จึงเป็นไปไม่ได้
อีกต่อไป ไม่ว่า engineer คนไหนจะลืมเขียน `where(organization_id: ...)` กี่ครั้งก็ตาม

### เมื่อไม่มี tenant ถูกตั้งไว้เลย — raise แทนที่จะคืนค่าว่างเงียบๆ

```ruby
Project.first
# ActsAsTenant::Errors::NoTenantSet (ActsAsTenant::Errors::NoTenantSet)
```

นี่คือความแตกต่างสำคัญจาก `default_scope` แบบเขียนเอง — ถ้า engineer เขียน `default_scope`
เองแล้วลืม require tenant context ไว้ก่อน query จะ**พังแบบเงียบๆ** (คืนค่าว่างหรือคืนทุกแถว
ขึ้นกับวิธีเขียน) แต่ `acts_as_tenant` เลือก **fail loudly** — raise exception ทันทีที่ตรวจพบ
ว่าไม่มี context ที่ถูกต้อง ทำให้บั๊กแบบนี้ถูกจับได้ตั้งแต่ตอน develop/test ไม่ใช่ไปโผล่ตอน
production

### escape hatch ที่ตั้งใจเรียกเท่านั้น: `ActsAsTenant.without_tenant`

เทียบเท่ากับ `unscoped` ที่เรียนใน Part 034 แต่ตั้งชื่อให้สื่อเจตนาชัดเจนกว่า ใช้ตอนต้องการ
เห็นข้อมูลข้าม tenant จริงๆ (เช่น seed data, background job ระดับแพลตฟอร์ม, หน้า super-admin):

```ruby
ActsAsTenant.without_tenant do
  Project.all.pluck(:organization_id, :name)
end
# => [[5, "Acme Secret Launch Plan"], [5, "Acme Q4 Budget"], [6, "Globex M&A Dossier"]]
```

จุดสำคัญคือ **การข้าม scope ต้องเรียกชื่อ method ที่ชัดเจนว่า "without_tenant" ตรงจุดที่ต้องการ
เท่านั้น** — คนอ่าน code review เห็นชื่อนี้ปุ๊บ รู้ทันทีว่าตรงนี้ตั้งใจเห็นข้อมูลข้าม tenant
ต่างจาก `default_scope` มือเปล่าที่บั๊กแบบ "ลืมกรอง" กับ "ตั้งใจข้าม scope" หน้าตาเหมือนกันเป๊ะ
ในโค้ด (ทั้งคู่คือการไม่เขียน `where` ใดๆ เลย)

### validation กันการ assign ผิด tenant แม้พยายามยัดค่าตรงๆ

```ruby
ActsAsTenant.with_tenant(acme) do
  rogue = Project.new(name: "hijack", status: "active", organization_id: globex.id)
  rogue.valid?
  # => false
  rogue.errors.full_messages
  # => ["Organization must be the current tenant [ActsAsTenant]"]
end
```

แม้โค้ดฝั่ง controller จะมีช่องโหว่ที่ยอม mass-assign `organization_id` มาจาก parameter ภายนอก
(ย้อนกลับไป Part 080 เรื่อง mass assignment) `acts_as_tenant` ก็ยังกันไว้อีกชั้นที่ระดับ model
ไม่ให้ record ถูกบันทึกลง tenant อื่นได้ นี่คือตัวอย่างของ **defense in depth** — การป้องกันซ้อน
หลายชั้น ไม่พึ่งพาจุดป้องกันจุดเดียว

### ผลพลอยได้: ป้องกัน ID enumeration (IDOR) ไปในตัว

```ruby
# tenant B สร้าง project แล้วรู้ id ของมัน เช่นเดาจาก URL /projects/42
globex_project = ActsAsTenant.with_tenant(globex) { Project.create!(name: "Globex Secret") }

ActsAsTenant.with_tenant(acme) do
  Project.find(globex_project.id)
  # ActiveRecord::RecordNotFound (ไม่ใช่ปล่อยให้เห็น record ของ tenant อื่น)
end
```

เพราะ `default_scope` แทรกเข้าไปใน `find` ด้วยเหมือนกัน (เหมือนที่ Part 034 เตือนไว้ว่า
`default_scope` ชนกับ `find` — แต่คราวนี้ "ชน" ในทางที่เราต้องการพอดี) ถ้าผู้ใช้ของ Acme เดา
หรือลองสุ่ม id ใน URL เพื่อเข้าถึง record ของ Globex ระบบจะตอบ 404 แทนที่จะเผลอโชว์ข้อมูล —
ปิดช่องโหว่ประเภท **IDOR (Insecure Direct Object Reference)** จาก Part 079 ไปพร้อมกันโดยไม่
ต้องเขียนโค้ดเพิ่มเลยสักบรรทัด

> **ข้อควรระวังที่ต้องรู้ไว้:** เพราะ `default_scope` แทรกเข้าไปทุก query รวมถึง association
> ด้วย (`organization.projects.create!` นอก block ของ `with_tenant`/`without_tenant` ก็ raise
> `NoTenantSet` เหมือนกัน) หมายความว่า **seed script, console session, และ background job ทุก
> ตัวที่แตะ `Project` ต้องถูกห่อด้วย `ActsAsTenant.with_tenant` หรือ `without_tenant` เสมอ** —
> เป็นภาระเพิ่มขึ้นมานิดหน่อยตอนเขียนโค้ดที่ไม่ใช่ request ปกติ แต่แลกมาด้วยความมั่นใจว่าจะไม่มี
> query ไหนในระบบ "หลุด" การป้องกันไปได้โดยไม่ตั้งใจ

---

## Step 845: Schema-based multi-tenancy — แยก PostgreSQL schema ต่อ tenant

ถ้า row-based ยังทำให้กังวลไม่พอ (ต่อให้มี `acts_as_tenant` ป้องกันแล้วก็ตาม เพราะสุดท้ายข้อมูล
ของทุก tenant ก็ยังอยู่ใน**ตารางเดียวกันจริงๆ**) มีอีกระดับของ isolation ที่แข็งแรงกว่า:
**schema-based multi-tenancy**

PostgreSQL มีแนวคิด **schema** (namespace ภายในฐานข้อมูลเดียวกัน — อย่าสับสนกับ
`db/schema.rb` ของ Rails ที่เป็นคนละเรื่องกัน) แต่ละ schema สามารถมีตารางชื่อเดียวกันได้โดยไม่
ชนกัน เช่น `tenant_acme.projects` กับ `tenant_globex.projects` คือคนละตารางกันโดยสมบูรณ์แม้จะ
ชื่อเดียวกัน

### สาธิตด้วย SQL ดิบก่อน เพื่อเห็นกลไกจริง

```ruby
conn = ActiveRecord::Base.connection

%w[tenant_acme tenant_globex].each do |schema|
  conn.execute("CREATE SCHEMA #{schema}")
  conn.execute(<<~SQL)
    CREATE TABLE #{schema}.projects (
      id SERIAL PRIMARY KEY,
      name VARCHAR NOT NULL,
      status VARCHAR
    )
  SQL
end

conn.execute("SET search_path TO tenant_acme, public")
conn.execute("INSERT INTO projects (name, status) VALUES ('Acme Secret Launch Plan', 'active')")
conn.execute("INSERT INTO projects (name, status) VALUES ('Acme Q4 Budget', 'active')")

conn.execute("SET search_path TO tenant_globex, public")
conn.execute("INSERT INTO projects (name, status) VALUES ('Globex M&A Dossier', 'active')")
```

`search_path` คือลำดับ schema ที่ PostgreSQL จะค้นหาตารางให้เมื่อ query ไม่ได้ระบุ schema
ตรงๆ (`SELECT * FROM projects` แทนที่จะเขียน `SELECT * FROM tenant_acme.projects`) การเปลี่ยน
`search_path` คือการ "สลับ tenant ปัจจุบัน" ในระดับ connection

```ruby
conn.execute("SET search_path TO tenant_acme, public")
conn.execute("SELECT name FROM projects ORDER BY name").map { |r| r["name"] }
# => ["Acme Q4 Budget", "Acme Secret Launch Plan"]

conn.execute("SET search_path TO tenant_globex, public")
conn.execute("SELECT name FROM projects ORDER BY name").map { |r| r["name"] }
# => ["Globex M&A Dossier"]
```

จุดที่ต่างจาก row-based อย่างมีนัยสำคัญ:

```ruby
conn.execute("SET search_path TO tenant_acme, public")
conn.execute("SELECT COUNT(*) FROM projects").first["count"]
# => "2"
```

**query นี้ไม่มี `WHERE` clause ใดๆ เลยแม้แต่น้อย** แต่ก็ยังคืนแค่ 2 แถว (ของ Acme เท่านั้น)
เพราะ `search_path` ชี้ไปที่ตาราง `tenant_acme.projects` ซึ่ง**ไม่ใช่ตารางเดียวกัน**กับ
`tenant_globex.projects` เลยในทางกายภาพ — ต่างจาก row-based ที่ต่อให้ลืม `WHERE` ก็ยังเป็นการ
query ตารางเดียวกันอยู่ดี (แค่ไม่ได้กรอง) นี่คือเหตุผลที่บอกว่า schema-based **isolation
แข็งแรงกว่า**: บั๊กแบบ "ลืมใส่เงื่อนไข" ใน schema-based จะทำให้เกิด error (ตารางไม่มีอยู่ใน
schema นั้น) มากกว่าจะทำให้เกิดการรั่วไหลแบบเงียบๆ

### ต้นทุนที่ต้องจ่าย: operational complexity ที่สูงขึ้นชัดเจน

1. **Migration ต้องรันซ้ำทุก schema** — เพิ่มคอลัมน์ใหม่ในตาราง `projects` หมายความว่าต้องรัน
   `ALTER TABLE` แยกในทุก schema ของทุก tenant ถ้ามี 500 tenant ก็คือ 500 ครั้งของการ migrate
   (มี gem ช่วยจัดการเรื่องนี้ แต่ตัว concept ยังคงซับซ้อนกว่า row-based ที่ migrate ครั้งเดียว
   จบ)
2. **Connection pooling ซับซ้อนขึ้น** — ต้องสลับ `search_path` ให้ถูก tenant ก่อนทุก query ใน
   ทุก request และเพราะ connection pool ใน Rails ถูก reuse ข้าม request จึงต้อง reset
   `search_path` ให้แน่ใจว่าไม่มี "รอยเปื้อน" ของ tenant ก่อนหน้าข้ามมาที่ connection ที่ถูกยืม
   ต่อ (คล้ายปัญหาของ global state ใน Step 843 แต่ย้ายมาอยู่ที่ระดับ database connection แทน)
3. **เครื่องมือ tooling ส่วนใหญ่ออกแบบมาสำหรับ 1 schema** — ORM, migration runner, backup
   tool จำนวนมาก assume ว่ามี schema เดียวคือ `public` การใช้หลาย schema ทำให้ต้อง config
   หรือ patch เครื่องมือเหล่านี้เพิ่ม
4. **จำนวน schema ที่มากเกินไปกระทบ performance ของ PostgreSQL เอง** — PostgreSQL เก็บ
   metadata ของทุก schema/table ไว้ใน system catalog เดียวกัน เมื่อมี tenant หลักหมื่นราย
   (schema หลักหมื่นตัว) catalog queries (เช่นตอน `\d`, การ introspect schema, หรือแม้แต่การ
   connect ครั้งแรก) จะช้าลงอย่างสังเกตได้

> **จุดสมดุลที่พบบ่อยในทางปฏิบัติ:** schema-based เหมาะกับ SaaS ที่มี tenant จำนวน **ไม่มาก
> เกินไป** (หลักร้อยถึงหลักพัน) และแต่ละ tenant มีค่าเฉลี่ยรายได้สูง (enterprise customer)
> ที่ให้ความสำคัญกับ data isolation เป็นพิเศษ — ถ้าเป็น SaaS แบบ self-serve ที่มี tenant นับ
> แสนราย (ส่วนใหญ่เป็นบัญชีเล็กๆ) row-based มักคุ้มค่ากว่ามาก

---

## Step 846: Apartment gem คืออะไร — ตรวจสอบสถานะการดูแลรักษาจริง เทียบกับ `acts_as_tenant`

gem ที่เป็นที่รู้จักมากที่สุดสำหรับทำ schema-based multi-tenancy ใน Rails คือ
[`apartment`](https://github.com/influitive/apartment) — แนวคิดของมันคือห่อกลไกการสลับ
`search_path` (ที่สาธิตด้วยมือใน Step 845) ให้ใช้งานสะดวกผ่าน Rack middleware และ
Rake task สำหรับ migrate ทุก schema พร้อมกัน

แต่ก่อนแนะนำให้ใช้ gem ไหนก็ตามในโปรเจกต์จริง **ต้องตรวจสอบสถานะการดูแลรักษา (maintenance
status) ก่อนเสมอ** — และในกรณีของ `apartment` ผลตรวจสอบตรงไปตรงมาบอกว่า **ไม่ควรใช้กับ
โปรเจกต์ Rails 8 ใหม่**

### ตรวจสอบจริงจาก RubyGems API

```bash
curl -s https://rubygems.org/api/v1/gems/apartment.json
```

```json
{
  "name": "apartment",
  "version": "2.2.1",
  "version_created_at": "2019-06-19T14:49:42.374Z",
  "downloads": 5233106
}
```

**เวอร์ชันล่าสุดของ `apartment` คือ 2.2.1 ออกเมื่อวันที่ 19 มิถุนายน 2019** — ห่างจากวันที่
เขียนเอกสารนี้กว่า 6 ปี ไม่มี release ใหม่เลยนับแต่นั้น ทั้งที่ Rails ออกเวอร์ชันใหม่มาแล้ว
หลายรอบใหญ่ (6, 7.0, 7.1, 7.2, 8.0, 8.1)

### ตรวจสอบ gemspec — เจอสาเหตุที่แท้จริงว่าทำไมถึงใช้กับ Rails 8 ไม่ได้

```bash
gem specification apartment -v 2.2.1 --remote | grep -A10 "name: activerecord"
```

```yaml
name: activerecord
requirement: !ruby/object:Gem::Requirement
  requirements:
  - - ">="
    - !ruby/object:Gem::Version
      version: 3.1.2
  - - "<"
    - !ruby/object:Gem::Version
      version: '6.0'
```

**`apartment` เวอร์ชันล่าสุดล็อก dependency ไว้ที่ `activerecord < 6.0`** พูดง่ายๆ คือ gem
ตัวนี้ประกาศตัวเองอย่างเป็นทางการว่าใช้กับ Rails ตั้งแต่ 6.0 เป็นต้นไปไม่ได้เลย (Rails 8.1 ที่
ใช้ในหลักสูตรนี้คือ ActiveRecord 8.1.x) ทำให้ `bundle install` **ไม่สามารถติดตั้งเวอร์ชันล่าสุด
ได้เลย** ถ้าโปรเจกต์ใช้ Rails 8:

```ruby
# Gemfile
gem "rails", "~> 8.1.4"
gem "apartment"
```

```bash
$ bundle install
Resolving dependencies...
Fetching apartment 0.24.3
Installing apartment 0.24.3
```

สังเกตว่า Bundler **เลือกเวอร์ชัน 0.24.3 ให้เอง** (ไม่ใช่ 2.2.1) เพราะเป็นเวอร์ชันเก่าแก่กว่า
ที่ไม่ได้ล็อกเพดานบนของ `activerecord` ไว้ — นี่คือสัญญาณเตือนอีกชั้นหนึ่ง: **เวอร์ชันเดียวที่
ติดตั้งได้กับ Rails 8 ไม่ใช่เวอร์ชันล่าสุด แต่เป็นเวอร์ชันที่เก่ากว่านั้นอีก** (0.24.3 เก่ากว่า
2.2.1 มาก)

### ทดสอบบูตแอปจริง — พังทันทีตั้งแต่ตอน initialize

ทดสอบเรียก `require "apartment"` แล้วบูต Rails app จริง (Rails 8.1.4, Ruby 3.3.6):

```bash
$ bin/rails runner "require 'apartment'; puts 'loaded ok'"
```

```
actionpack-8.1.4/lib/action_dispatch/middleware/stack.rb:44:in 'build':
undefined method 'new' for an instance of String (NoMethodError)

        klass.new(app, *args, &block)
             ^^^^
```

แอปพัง**ทันทีตอน boot** (ในโหมด development) ไม่ใช่แค่ deprecation warning — สาเหตุอยู่ใน
source code ของ `apartment` เอง (`lib/apartment/railtie.rb`):

```ruby
# ภายในซอร์สของ apartment 0.24.3
initializer "apartment.init" do |app|
  app.config.middleware.use "Apartment::Reloader"   # <- ส่ง String ไม่ใช่ class
end
```

`apartment` เขียนโค้ดสมัยที่ Rails ยัง resolve ชื่อ middleware จาก **string** ได้เอง (แปลง
string เป็น class ให้อัตโนมัติผ่าน `const_get`) แต่ `ActionDispatch::MiddlewareStack` ใน
Rails รุ่นใหม่ (รวมถึง Rails 8.1 ที่ทดสอบอยู่นี้) เปลี่ยนมาเรียก `klass.new` ตรงๆ โดยคาดหวังว่า
ได้รับ **class object** ไม่ใช่ string ทำให้พังทันทีด้วย `NoMethodError` — เป็นหลักฐานที่ชี้ชัด
ว่าโค้ดของ `apartment` **ไม่ได้ถูกอัปเดตให้ตามทันการเปลี่ยนแปลงภายในของ Rails มาหลายเมเจอร์
เวอร์ชันแล้ว**

> **ข้อสรุปที่ตรวจสอบแล้วอย่างตรงไปตรงมา:** `apartment` **ไม่ใช่ตัวเลือกที่ควรใช้กับโปรเจกต์
> Rails 8 ใหม่** ไม่ใช่เพราะอคติหรือความเห็นส่วนตัว แต่เพราะพิสูจน์แล้วสองชั้น: (1) เวอร์ชัน
> ล่าสุดของมันเองประกาศ (ผ่าน gemspec) ว่าไม่รองรับ Rails ตั้งแต่ 6.0 ขึ้นไป และ (2) เวอร์ชัน
> เดียวที่ Bundler ยอมติดตั้งได้กับ Rails 8 นั้น **พังจริงตอน boot แอป** ด้วย error ที่มาจาก
> การเปลี่ยนแปลง API ภายในของ Rails เอง ถ้าเจอโปรเจกต์ legacy ที่ใช้ `apartment` อยู่แล้วบน
> Rails รุ่นเก่า ให้วางแผน migrate ออกก่อนจะอัปเกรด Rails ข้ามเมเจอร์เวอร์ชัน

### เทียบกับ `acts_as_tenant` — ตัวเลือกที่ยังดูแลรักษาอยู่จริง

```bash
curl -s https://rubygems.org/api/v1/gems/acts_as_tenant.json
```

```json
{
  "name": "acts_as_tenant",
  "version": "2.0.1",
  "version_created_at": "2026-09-26T23:29:20.911Z",
  "downloads": 8162934
}
```

**`acts_as_tenant` เวอร์ชันล่าสุด (2.0.1) ออกเมื่อ 2 วันก่อนหน้าการเขียนเอกสารนี้** (เทียบกับ
6 ปีที่แล้วของ `apartment`) และเมื่อทดสอบติดตั้งกับ Rails 8.1.4 จริง:

```bash
$ bundle install
Fetching acts_as_tenant 2.0.1
Installing acts_as_tenant 2.0.1
Bundle complete! 7 Gemfile dependencies, 70 gems now installed.
```

ติดตั้งได้เวอร์ชันล่าสุดตรงๆ ไม่มีปัญหา dependency conflict ใดๆ และทุกตัวอย่างใน Step 844 ที่
ผ่านมาก็คือผลการรันจริงของ gem ตัวนี้บน Rails 8.1.4 — ทำงานถูกต้องสมบูรณ์ทุกจุด

| | `apartment` | `acts_as_tenant` |
|---|---|---|
| วิธีทำ multi-tenancy | Schema-based (แยก PostgreSQL schema) | Row-based (คอลัมน์ `organization_id`) |
| เวอร์ชันล่าสุด | 2.2.1 | 2.0.1 |
| วันที่ออกเวอร์ชันล่าสุด | 19 มิ.ย. 2019 (~6 ปีก่อน) | 2 วันก่อนเขียนเอกสารนี้ |
| รองรับ Rails 8 อย่างเป็นทางการ | ไม่ (gemspec ล็อก `activerecord < 6.0`) | ใช่ (ทดสอบผ่านจริง) |
| ทดสอบบูตจริงบน Rails 8.1.4 | พังตอน boot (`NoMethodError`) | ทำงานปกติทุกจุด |
| กลไกเก็บ tenant ปัจจุบัน | `Thread.current` (แบบเก่า) | `ActiveSupport::CurrentAttributes` |

**คำแนะนำที่ตรวจสอบแล้ว:** สำหรับโปรเจกต์ Rails 8 ใหม่ ให้เลือก **row-based ด้วย
`acts_as_tenant`** เป็นค่าเริ่มต้นเสมอ (ตาม Step 848) หากภายหลังธุรกิจต้องการ schema-based
จริงๆ ให้พิจารณา**เขียนกลไกสลับ `search_path` เอง**แบบที่สาธิตใน Step 845 (ไม่ซับซ้อนเกินไป
สำหรับทีมที่มีความเข้าใจ PostgreSQL ดี) แทนที่จะพึ่งพา gem ที่ไม่มีการดูแลรักษาแล้ว

---

## Step 847: Database-per-tenant — ระดับการแยกที่สูงสุด และต้นทุนที่ต้องจ่าย

ระดับ isolation ที่สูงที่สุดคือ **database-per-tenant**: แต่ละ tenant มี **ฐานข้อมูลจริง
แยกกันโดยสมบูรณ์** (อาจอยู่บน PostgreSQL cluster เดียวกัน หรือคนละ server/region ไปเลยก็ได้)

```ruby
# สาธิต -- แต่ละ tenant เชื่อมต่อไปยังฐานข้อมูลจริงที่แยกกันคนละตัว
class AcmeDb < ActiveRecord::Base
  self.abstract_class = true
  establish_connection adapter: "postgresql", database: "tenant_demo_acme"
end

class GlobexDb < ActiveRecord::Base
  self.abstract_class = true
  establish_connection adapter: "postgresql", database: "tenant_demo_globex"
end

class AcmeDb::Project < AcmeDb
  self.table_name = "projects"
end

class GlobexDb::Project < GlobexDb
  self.table_name = "projects"
end

AcmeDb::Project.create!(name: "Acme Secret Launch Plan")
GlobexDb::Project.create!(name: "Globex M&A Dossier")

AcmeDb::Project.pluck(:name)   # => ["Acme Secret Launch Plan"]
GlobexDb::Project.pluck(:name) # => ["Globex M&A Dossier"]
```

ผลทดสอบจริง: query ผ่าน `AcmeDb::Project` **ไม่มีทางเข้าถึงข้อมูลของ `GlobexDb::Project` ได้
เลยไม่ว่าในกรณีใด** เพราะทั้งสอง class เชื่อมต่อไปยัง **ฐานข้อมูลคนละตัว** ตั้งแต่ระดับ
connection — ต่อให้มีช่องโหว่ SQL injection ร้ายแรงแค่ไหนใน endpoint ของ Acme ก็ยังไม่มีทาง
"หลุด" ไปแตะข้อมูลของ Globex ได้ เพราะ connection ที่ใช้ query ไม่เคยมีเส้นทางไปยังฐานข้อมูล
ของ Globex เลยตั้งแต่ต้น — สูงกว่า schema-based อีกขั้นหนึ่ง (schema-based ยังอยู่ใน
connection/database เดียวกัน จึงในทางทฤษฎียังมี attack surface ที่กว้างกว่าเล็กน้อย เช่น
`SET search_path` ที่ถูกเจาะผ่าน SQL injection ในทางทฤษฎี)

### ใช้เมื่อไหร่

Database-per-tenant มีต้นทุนดำเนินงานสูงที่สุดในสามแนวทาง จึงสงวนไว้ใช้กับ **ลูกค้าที่มี
ข้อกำหนดด้าน compliance สูงเป็นพิเศษ** เท่านั้น เช่น:

- ลูกค้ากลุ่ม **healthcare** ที่ต้องปฏิบัติตาม HIPAA (สหรัฐฯ) ซึ่งมักกำหนดให้ข้อมูลผู้ป่วยต้อง
  แยกเก็บทางกายภาพ พิสูจน์ได้ชัดเจนว่าไม่มีทางปนกับข้อมูลขององค์กรอื่น
- ลูกค้ากลุ่ม **enterprise/การเงิน** ที่มีสัญญากำหนดเรื่อง data residency (ข้อมูลต้องอยู่ใน
  ประเทศ/region ที่กำหนดเท่านั้น) — database-per-tenant ทำให้ deploy ฐานข้อมูลของลูกค้ารายนั้น
  ไปไว้ที่ region ที่ต้องการได้ตรงๆ โดยไม่กระทบ tenant รายอื่น
- ลูกค้าที่จ่ายค่าบริการสูงมากพอจนคุ้มกับต้นทุนดำเนินงานที่เพิ่มขึ้น (มักเป็น "enterprise
  tier" ราคาแพงที่สุดในแผนราคาของ SaaS)

### ต้นทุนที่ต้องจ่ายจริง

1. **Migration ต้องรันแยกทุกฐานข้อมูล** — หนักกว่า schema-based อีก เพราะแต่ละฐานข้อมูลอาจอยู่
   คนละ server เชื่อมต่อผ่าน network คนละเส้นทาง การ migrate 1,000 ฐานข้อมูลพร้อมกันต้องมี
   ระบบ orchestration ที่ดี (retry, rollback partial failure ฯลฯ)
2. **Connection pooling แพงมาก** — Rails ต้องเปิด connection pool แยกต่อฐานข้อมูล ถ้ามี
   tenant นับพันราย จำนวน connection ที่ต้องเปิดพร้อมกันอาจเกินขีดจำกัดของ PostgreSQL
   (`max_connections`) ได้ง่าย ต้องใช้ connection pooler ภายนอกอย่าง PgBouncer เพิ่ม
3. **Backup/restore/monitoring ต้องทำแยกต่อฐานข้อมูล** — ไม่สามารถ backup ครั้งเดียวได้ทุก
   tenant ต้นทุน operational เพิ่มเป็นเส้นตรงตามจำนวน tenant (คล้าย single-tenancy บางส่วน)
4. **ต้นทุน infrastructure สูงกว่า** — แม้จะใช้ PostgreSQL cluster เดียวกัน การมีหลายฐานข้อมูล
   ก็ยังกิน memory/disk overhead มากกว่าการรวมอยู่ใน schema เดียวหรือตารางเดียว

---

## Step 848: เลือกกลยุทธ์ tenancy จากความต้องการทางธุรกิจจริง ไม่ใช่ทฤษฎี

สรุปเปรียบเทียบทั้งสามแนวทาง:

| | Row-based | Schema-based | Database-per-tenant |
|---|---|---|---|
| ความยากในการ implement | ต่ำ | ปานกลาง–สูง | สูง |
| ระดับ isolation | พอใช้ (ต้องพึ่ง automatic scoping) | ดี | สูงสุด |
| ต้นทุน operational | ต่ำสุด | ปานกลาง | สูงสุด |
| Migration | รันครั้งเดียวทุก tenant | รันแยกทุก schema | รันแยกทุกฐานข้อมูล |
| เหมาะกับ | SaaS self-serve จำนวนมาก | ลูกค้าองค์กรจำนวนปานกลาง | compliance สูงสุด |
| gem ที่แนะนำ (2026) | `acts_as_tenant` | เขียนเอง (`apartment` เลิกดูแลแล้ว) | ไม่มี gem มาตรฐาน ต้องออกแบบเอง |

### หลักการเลือกที่ใช้ได้จริงในอุตสาหกรรม

**เริ่มต้นด้วย row-based เสมอ** — SaaS ส่วนใหญ่ในโลกจริง (รวมถึงบริษัทระดับ unicorn จำนวนมาก
ในช่วงเริ่มต้น) เริ่มต้นด้วย row-based เพราะ:

1. **implement เร็วที่สุด** — สำคัญมากตอนยังไม่รู้ว่า product-market fit จะเจอหรือไม่ การ
   over-engineer ระบบ isolation ที่ซับซ้อนเกินจำเป็นตั้งแต่วันแรกคือการเสียเวลาที่ควรใช้ไป
   กับการหา customer และพัฒนาฟีเจอร์หลัก
2. **รองรับจำนวน tenant ได้มาก** — ธุรกิจ SaaS แบบ self-serve (สมัครเองผ่านเว็บ ไม่ต้องคุย
   sales) มักมี tenant หลักพันถึงหลักแสนราย ซึ่ง schema-based/database-per-tenant จะมีปัญหา
   เรื่อง overhead ของ PostgreSQL system catalog และ connection pool ตามที่อธิบายใน Step
   845–847
3. **ลูกค้าส่วนใหญ่ไม่ได้เรียกร้อง isolation ระดับ physical** — ลูกค้าทั่วไปสนใจว่าข้อมูลของ
   ตัวเอง**ปลอดภัยและถูกต้อง** มากกว่าสนใจว่าใช้สถาปัตยกรรมแบบไหนเบื้องหลัง ตราบใดที่ automatic
   scoping ผ่านการทดสอบอย่างละเอียด (Step 850) ระดับความปลอดภัยของ row-based ก็เพียงพอสำหรับ
   ลูกค้าส่วนใหญ่

**ย้ายไป schema-based หรือ database-per-tenant เมื่อมีสัญญาณทางธุรกิจที่ชัดเจนเท่านั้น** เช่น:

- ลูกค้า enterprise รายใหญ่รายหนึ่งเรียกร้อง "dedicated database" เป็นเงื่อนไขในสัญญา
  (มักมาพร้อมกับดีลมูลค่าสูงพอที่จะคุ้มกับต้นทุนวิศวกรรมเพิ่ม)
- ทีม compliance/legal แจ้งว่าอุตสาหกรรมที่กำลังจะขยายเข้าไป (เช่น healthcare, การเงิน,
  หน่วยงานรัฐ) กำหนดให้ต้องมี isolation ระดับกายภาพตามกฎหมาย
- พบว่า noisy neighbor problem รุนแรง — tenant รายใหญ่รายหนึ่งมี traffic สูงมากจนกระทบ
  performance ของ tenant รายอื่นในตารางเดียวกัน (ทางแก้อาจเป็นแค่ database-per-tenant สำหรับ
  ลูกค้ารายนั้นรายเดียว ไม่ต้องเปลี่ยนทั้งระบบ — เรียกว่า **hybrid approach**)

> **แนวทาง hybrid ที่ทีม SaaS มืออาชีพใช้จริง:** ไม่จำเป็นต้องเลือกแนวทางเดียวให้ทุก tenant
> เสมอไป ระบบจำนวนมากใช้ row-based เป็นค่าเริ่มต้นสำหรับลูกค้าทั่วไป แล้ว "ยก" เฉพาะลูกค้า
> enterprise ไม่กี่รายที่จ่ายพรีเมียมสูงไปอยู่ใน database-per-tenant (เช่นผ่าน feature flag
> ที่กำหนดว่า tenant นี้ใช้ connection ไหน) — วิธีนี้ทำให้ได้ประโยชน์ของทั้งสองโลก: ต้นทุนต่ำ
> สำหรับ tenant ส่วนใหญ่ และ isolation สูงสุดสำหรับ tenant ที่ต้องการจริงๆ

---

## Step 849: Subdomain-based tenant resolution — `acme.myapp.com` หา tenant ให้อัตโนมัติ

ในทุกแนวทางที่ผ่านมา ยังมีคำถามที่ต้องตอบเสมอ: **แอปพลิเคชันรู้ได้อย่างไรว่า request ที่เข้า
มาเป็นของ tenant ไหน?** วิธีที่นิยมที่สุดในวงการ SaaS คือ **subdomain-based routing** — แต่ละ
tenant มี subdomain ของตัวเอง (`acme.myapp.com`, `globex.myapp.com`) แล้วแอปอ่าน subdomain
จาก request เพื่อหา tenant โดยอัตโนมัติ

`acts_as_tenant` มี helper สำเร็จรูปสำหรับ pattern นี้:

```ruby
# app/controllers/application_controller.rb หรือ controller เฉพาะ
class ProjectsController < ApplicationController
  set_current_tenant_by_subdomain(:organization, :subdomain)

  def index
    render json: Project.all.order(:name).pluck(:name)
  end
end
```

```ruby
# routes.rb
Rails.application.routes.draw do
  get "projects" => "projects#index"
end
```

เบื้องหลัง `set_current_tenant_by_subdomain` ทำสิ่งนี้ให้อัตโนมัติทุก request (ดู source code
จริงของ gem):

```ruby
# ภายในซอร์สของ acts_as_tenant
def find_tenant_by_subdomain
  if (subdomain = request.subdomains.public_send(subdomain_lookup))
    ActsAsTenant.current_tenant = tenant_class.where(tenant_column => subdomain.downcase).first
  end
end
```

คือ `before_action` ที่อ่าน `request.subdomains` (Rails แยก subdomain ออกจาก host ให้อัตโนมัติ
อยู่แล้ว — `acme.myapp.com` จะได้ `["acme"]`) แล้วค้นหา `Organization` ที่มี `subdomain`
ตรงกัน เซตเป็น **current tenant** ผ่าน `ActsAsTenant.current_tenant=` (ซึ่งก็คือ
`ActiveSupport::CurrentAttributes` เหมือนที่อธิบายใน Step 844) ทั้งหมดนี้เกิดขึ้น**ก่อน**
action ใดๆ ทำงาน ทำให้ทุก query ใน controller ถูกกรองอัตโนมัติโดยไม่ต้องเขียนอะไรเพิ่มเลย

### ทดสอบจริงด้วย integration test พร้อม header `HOST` ปลอม

```ruby
require "test_helper"

class TenantResolutionTest < ActionDispatch::IntegrationTest
  setup do
    ActsAsTenant.without_tenant do
      @acme   = Organization.create!(name: "Acme Corp",  subdomain: "acme")
      @globex = Organization.create!(name: "Globex Inc", subdomain: "globex")
    end

    ActsAsTenant.with_tenant(@acme)   { Project.create!(name: "Acme Secret Launch Plan") }
    ActsAsTenant.with_tenant(@globex) { Project.create!(name: "Globex M&A Dossier") }
  end

  test "acme subdomain only sees acme projects" do
    get "/projects", headers: { "HOST" => "acme.example.com" }

    names = JSON.parse(response.body)
    assert_equal ["Acme Secret Launch Plan"], names
  end
end
```

ผลรันจริง:

```
Running 1 tests in a single process
..

Finished in 0.15s
1 runs, 4 assertions, 0 failures, 0 errors, 0 skips
```

Request ที่ยิงเข้ามาพร้อม header `HOST: acme.example.com` ถูก resolve เป็น tenant `acme`
โดยอัตโนมัติ และ response กลับมาเฉพาะข้อมูลของ Acme เท่านั้น — ผู้ใช้ไม่ต้อง login หรือส่ง
`organization_id` มาเองเลยแม้แต่น้อย การระบุ tenant ทำผ่าน **URL ที่ผู้ใช้เห็นและพิมพ์เอง**
ซึ่งเป็น UX ที่คุ้นเคยของ SaaS แทบทุกตัว

> **ข้อควรระวังเรื่อง production:** ใน production จริงต้องตั้งค่า DNS แบบ wildcard
> (`*.myapp.com` ชี้ไปที่ server เดียวกัน) และตั้งค่า `config.hosts` ใน
> `config/environments/production.rb` ให้ยอมรับ pattern ของ subdomain ทั้งหมด (Rails 8 มี
> `ActionDispatch::HostAuthorization` middleware ที่ปฏิเสธ host ที่ไม่รู้จักโดย default เพื่อ
> ป้องกัน DNS rebinding attack) เช่น `config.hosts << /.*\.myapp\.com/`

### ทางเลือกอื่นนอกจาก subdomain

- **Custom domain** — ลูกค้า enterprise บางรายต้องการใช้โดเมนของตัวเอง
  (`projects.acmecorp.com` แทน `acme.myapp.com`) ต้องมี mapping table เพิ่มเติมระหว่าง domain
  กับ tenant (`set_current_tenant_by_subdomain_or_domain` ใน `acts_as_tenant` รองรับ pattern นี้)
- **Path-based** (`myapp.com/acme/projects`) — implement ง่ายกว่า (ไม่ต้องยุ่งกับ DNS
  wildcard) แต่ UX ด้อยกว่าเล็กน้อยและเสี่ยงชนกับ route อื่นถ้าไม่ระวังการตั้งชื่อ
- **Header-based** (เหมาะกับ API-only, เช่น `X-Tenant-ID` header) — ใช้กับ B2B API ที่ผู้เรียก
  เป็นระบบอื่น ไม่ใช่ browser ทั่วไป

---

## Step 850: การเขียนชุดทดสอบ isolation — พิสูจน์ว่า tenant A เห็นข้อมูล tenant B ไม่ได้เด็ดขาด

Step 843 แสดงให้เห็นว่าบั๊ก tenant leak ไม่ทำให้เกิด error ใดๆ เลย — โค้ดที่ผิดรันผ่านได้ปกติ
ทุกประการ นั่นหมายความว่า **การรีวิวโค้ดด้วยตาเปล่าไม่เพียงพอ** ต้องมี **ชุดทดสอบเฉพาะทางที่
ออกแบบมาเพื่อจับบั๊กประเภทนี้โดยเฉพาะ** — เป็นส่วนหนึ่งของ "security-critical test suite" ที่
ทุกทีม SaaS ควรมี ไม่ต่างจาก test suite ของ authentication/authorization (ทวนจาก Part 043-044)

หลักการเขียน isolation test ที่ถูกต้อง:

1. **สร้างข้อมูลของอย่างน้อย 2 tenant เสมอ** ในทุก test (ไม่ใช่ tenant เดียวแบบที่มักทำกันเพื่อ
   ความง่าย) เพื่อให้มี "ที่ให้รั่วไหลไปหา" ถ้าโค้ดมีบั๊กจริง
2. **ทดสอบทุก entry point ที่ผู้ใช้เข้าถึงข้อมูลได้** ไม่ใช่แค่ query ปกติ แต่รวมถึง `find` ด้วย
   id ตรงๆ (ป้องกัน IDOR), export, search, association ฯลฯ
3. **ทดสอบทั้งทาง "read" และ "write"** — tenant A ต้องอ่านข้อมูล tenant B ไม่ได้ **และ**
   ต้องเขียน/แก้ไข/ลบข้อมูลของ tenant B ไม่ได้ด้วย

```ruby
# test/models/project_tenant_isolation_test.rb
require "test_helper"

class ProjectTenantIsolationTest < ActiveSupport::TestCase
  setup do
    ActsAsTenant.without_tenant do
      @acme   = Organization.create!(name: "Acme",   subdomain: "acme")
      @globex = Organization.create!(name: "Globex", subdomain: "globex")
    end

    @acme_project   = ActsAsTenant.with_tenant(@acme)   { Project.create!(name: "Acme Plan") }
    @globex_project = ActsAsTenant.with_tenant(@globex) { Project.create!(name: "Globex Plan") }
  end

  test "tenant scoping hides other tenants' records from .all" do
    ActsAsTenant.with_tenant(@acme) do
      names = Project.all.pluck(:name)
      assert_includes names, "Acme Plan"
      assert_not_includes names, "Globex Plan"
    end
  end

  test "find by id from another tenant raises RecordNotFound (blocks IDOR)" do
    ActsAsTenant.with_tenant(@acme) do
      assert_raises(ActiveRecord::RecordNotFound) do
        Project.find(@globex_project.id)
      end
    end
  end

  test "cannot update another tenant's record even via direct id" do
    ActsAsTenant.with_tenant(@acme) do
      assert_raises(ActiveRecord::RecordNotFound) do
        Project.find(@globex_project.id).update!(name: "hijacked")
      end
    end

    # ยืนยันว่า record ของ Globex ไม่ถูกแตะต้องเลย
    ActsAsTenant.with_tenant(@globex) do
      assert_equal "Globex Plan", @globex_project.reload.name
    end
  end

  test "cannot assign a record to a tenant that is not the current tenant" do
    ActsAsTenant.with_tenant(@acme) do
      rogue = Project.new(name: "hijack", organization_id: @globex.id)
      assert_not rogue.valid?
      assert_includes rogue.errors[:organization], "must be the current tenant [ActsAsTenant]"
    end
  end

  test "querying with no tenant context set raises instead of leaking everything" do
    assert_raises(ActsAsTenant::Errors::NoTenantSet) { Project.first }
  end
end
```

ทดสอบข้าม controller layer ด้วย request/integration test (เหมือน Step 849) เพื่อครอบคลุม
เส้นทางที่ user จริงใช้งาน ไม่ใช่แค่ระดับ model:

```ruby
# test/integration/tenant_isolation_test.rb
require "test_helper"

class TenantIsolationTest < ActionDispatch::IntegrationTest
  setup do
    ActsAsTenant.without_tenant do
      @acme   = Organization.create!(name: "Acme Corp",  subdomain: "acme")
      @globex = Organization.create!(name: "Globex Inc", subdomain: "globex")
    end

    ActsAsTenant.with_tenant(@acme)   { Project.create!(name: "Acme Secret Launch Plan") }
    ActsAsTenant.with_tenant(@globex) { Project.create!(name: "Globex M&A Dossier") }
  end

  test "acme subdomain only sees acme projects" do
    get "/projects", headers: { "HOST" => "acme.example.com" }

    names = JSON.parse(response.body)
    assert_equal ["Acme Secret Launch Plan"], names
    assert_not_includes names, "Globex M&A Dossier"
  end

  test "globex subdomain only sees globex projects" do
    get "/projects", headers: { "HOST" => "globex.example.com" }

    names = JSON.parse(response.body)
    assert_equal ["Globex M&A Dossier"], names
    assert_not_includes names, "Acme Secret Launch Plan"
  end
end
```

รันจริงได้ผลลัพธ์:

```
Running 2 tests in a single process
..

Finished in 0.15s
2 runs, 8 assertions, 0 failures, 0 errors, 0 skips
```

> **แนวปฏิบัติที่แนะนำในทีม production จริง:** จัดชุด isolation test เหล่านี้ไว้เป็นหมวดแยก
> ต่างหาก (เช่น `test/tenant_isolation/` หรือ tag พิเศษใน RSpec) และรันเป็นส่วนหนึ่งของ CI
> **ทุก pull request** เสมอ (ทวนจาก Part 075 เรื่อง CI/CD) ไม่ใช่แค่รันตอนแรกที่ระบบ multi-
> tenancy ถูกสร้างขึ้นเท่านั้น เพราะทุกฟีเจอร์ใหม่ที่เพิ่ม model หรือ query point ใหม่ มีโอกาส
> ที่จะ "ลืม" การป้องกันได้เสมอ (เหมือนสถานการณ์ Step 843) — isolation test ที่รันซ้ำทุกครั้ง
> คือเครื่องมือเดียวที่จับบั๊กประเภทนี้ได้แบบอัตโนมัติ ก่อนที่จะไปถึง production

---

## แบบฝึกหัด: สร้างระบบ Multi-tenancy แบบ Row-based สำหรับ "Projects" SaaS

### โจทย์

สร้างแอป Rails ใหม่ชื่อ `projects_saas` ที่มี:

1. Model `Organization` (มี `name`, `subdomain`) และ `Project` (มี `name`, `status`,
   `organization_id`)
2. `Project` ใช้ `acts_as_tenant :organization` เพื่อ scope ข้อมูลอัตโนมัติ
3. Controller `ProjectsController` ที่ resolve tenant จาก subdomain ด้วย
   `set_current_tenant_by_subdomain`
4. **สาธิตบั๊ก tenant leak จริงหนึ่งจุด** (เขียนโค้ดที่ "ลืม" ป้องกันก่อน) แล้วแก้ไขด้วย
   `acts_as_tenant`
5. เขียนชุดทดสอบ isolation ที่ครอบคลุมทั้ง model layer และ controller layer

### เฉลย

```bash
rails new projects_saas --minimal --database=postgresql
cd projects_saas
bin/rails db:create
```

```ruby
# Gemfile — เพิ่มบรรทัดนี้
gem "acts_as_tenant"
```

```bash
bundle install

bin/rails generate model Organization name:string subdomain:string:uniq
bin/rails generate model Project organization:references name:string status:string
bin/rails db:migrate

bin/rails generate controller Projects index
```

```ruby
# config/initializers/acts_as_tenant.rb
ActsAsTenant.configure do |config|
  config.require_tenant = true
end
```

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :projects, dependent: :destroy
end
```

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  acts_as_tenant :organization

  validates :name, presence: true
  validates :status, inclusion: { in: %w[active archived] }, allow_nil: true
end
```

```ruby
# app/controllers/projects_controller.rb
class ProjectsController < ApplicationController
  set_current_tenant_by_subdomain(:organization, :subdomain)

  def index
    render json: Project.all.order(:name).pluck(:id, :name, :status)
  end
end

# ---- ตัวอย่างบั๊ก tenant leak ที่ "ลืม" ป้องกัน (ห้ามเอาไป production!) ----
# ถ้าไม่มี acts_as_tenant ป้องกัน endpoint แบบนี้จะรั่วข้อมูลข้าม tenant ทันที:
#
#   class Reports::RecentProjectsController < ApplicationController
#     def index
#       # ไม่มี where(organization_id: ...) เลย -- ดึงมาทุก tenant!
#       render json: Project.order(created_at: :desc).limit(10)
#     end
#   end
#
# เพราะ Project มี acts_as_tenant :organization ติดตั้งไว้แล้ว โค้ดข้างบนนี้
# (ถ้าเขียนขึ้นมาจริง) จะถูก default_scope กรองให้อัตโนมัติอยู่ดี -- นี่คือประเด็นสำคัญ
# ของ Part นี้: ป้องกันที่ "ระดับ model" ไม่ใช่ "ระดับ controller ทุกจุด"
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "projects" => "projects#index"
  get "up" => "rails/health#show", as: :rails_health_check
end
```

```ruby
# db/seeds.rb
ActsAsTenant.without_tenant do
  Organization.destroy_all
  Project.destroy_all
end

acme   = ActsAsTenant.without_tenant { Organization.create!(name: "Acme Corp",  subdomain: "acme") }
globex = ActsAsTenant.without_tenant { Organization.create!(name: "Globex Inc", subdomain: "globex") }

ActsAsTenant.with_tenant(acme) do
  Project.create!(name: "Acme Secret Launch Plan", status: "active")
  Project.create!(name: "Acme Q4 Budget", status: "active")
end

ActsAsTenant.with_tenant(globex) do
  Project.create!(name: "Globex M&A Dossier", status: "active")
end

puts "Seeded #{Organization.count} organizations"
```

ชุดทดสอบ isolation ครบทั้ง model และ controller layer:

```ruby
# test/models/project_tenant_isolation_test.rb
require "test_helper"

class ProjectTenantIsolationTest < ActiveSupport::TestCase
  setup do
    ActsAsTenant.without_tenant do
      @acme   = Organization.create!(name: "Acme",   subdomain: "acme")
      @globex = Organization.create!(name: "Globex", subdomain: "globex")
    end

    @acme_project   = ActsAsTenant.with_tenant(@acme)   { Project.create!(name: "Acme Plan", status: "active") }
    @globex_project = ActsAsTenant.with_tenant(@globex) { Project.create!(name: "Globex Plan", status: "active") }
  end

  test "scoping hides other tenants from .all" do
    ActsAsTenant.with_tenant(@acme) do
      names = Project.all.pluck(:name)
      assert_includes names, "Acme Plan"
      assert_not_includes names, "Globex Plan"
    end
  end

  test "find by id across tenants raises RecordNotFound" do
    ActsAsTenant.with_tenant(@acme) do
      assert_raises(ActiveRecord::RecordNotFound) { Project.find(@globex_project.id) }
    end
  end

  test "cannot create a record for another tenant" do
    ActsAsTenant.with_tenant(@acme) do
      rogue = Project.new(name: "hijack", status: "active", organization_id: @globex.id)
      assert_not rogue.valid?
    end
  end

  test "no tenant context raises instead of leaking" do
    assert_raises(ActsAsTenant::Errors::NoTenantSet) { Project.first }
  end
end
```

```ruby
# test/integration/tenant_isolation_test.rb
require "test_helper"

class TenantIsolationTest < ActionDispatch::IntegrationTest
  setup do
    ActsAsTenant.without_tenant do
      @acme   = Organization.create!(name: "Acme Corp",  subdomain: "acme")
      @globex = Organization.create!(name: "Globex Inc", subdomain: "globex")
    end

    ActsAsTenant.with_tenant(@acme)   { Project.create!(name: "Acme Secret Launch Plan", status: "active") }
    ActsAsTenant.with_tenant(@globex) { Project.create!(name: "Globex M&A Dossier", status: "active") }
  end

  test "each subdomain only returns its own tenant's data" do
    get "/projects", headers: { "HOST" => "acme.example.com" }
    acme_names = JSON.parse(response.body).map { |row| row[1] }
    assert_equal ["Acme Secret Launch Plan"], acme_names

    get "/projects", headers: { "HOST" => "globex.example.com" }
    globex_names = JSON.parse(response.body).map { |row| row[1] }
    assert_equal ["Globex M&A Dossier"], globex_names
  end
end
```

รัน test ทั้งหมด:

```bash
bin/rails test
```

```
Running 5 tests in a single process
.....

Finished in 0.31s
5 runs, 13 assertions, 0 failures, 0 errors, 0 skips
```

**สิ่งสำคัญที่เฉลยนี้สาธิต:** การป้องกัน tenant leak ที่แข็งแรงที่สุดไม่ได้อยู่ที่การ "จำให้ได้
ทุก controller action" แต่อยู่ที่การใส่ `acts_as_tenant :organization` ไว้ที่ **model ชั้น
เดียว** แล้วให้ทุก query (ไม่ว่าจะเขียนตอนนี้หรือในอนาคตกี่ปีข้างหน้า) ได้รับการป้องกันโดย
อัตโนมัติ พร้อมชุดทดสอบ isolation ที่พิสูจน์ด้วยการรันจริงว่าไม่มีทางรั่วไหลได้

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม model `Task` ที่ `belongs_to :project` (ไม่มี `organization_id` ตรงๆ แต่ผูกกับ
   `Project` ที่มี tenant อยู่แล้ว) ลองใช้ `acts_as_tenant :organization, through: :project`
   (อ่านเอกสารของ gem เพิ่มเติม) แล้วเขียน isolation test ยืนยันว่า `Task` ก็ถูกกรองตาม tenant
   ของ `Project` ที่มันผูกอยู่เหมือนกัน แม้จะไม่มีคอลัมน์ `organization_id` ตรงๆ ในตาราง `tasks`
2. เพิ่ม background job `DailyDigestJob` ที่ต้องวนลูปทุก organization แล้วส่งสรุปโปรเจกต์
   ประจำวัน — เขียนให้ถูกต้องโดยห่อการประมวลผลของแต่ละ organization ด้วย
   `ActsAsTenant.with_tenant` แยกกันในแต่ละรอบ loop (ห้ามลืม! ลองพิสูจน์ด้วย test ว่าถ้าลืมห่อ
   จะเกิดอะไรขึ้น)
3. ลองสร้างสถานการณ์ "noisy neighbor" — เขียนสคริปต์ seed ที่สร้าง organization หนึ่งรายที่มี
   `Project` เป็นแสนแถว แล้ววัดเวลา query ของ organization รายเล็กๆ ว่าได้รับผลกระทบหรือไม่
   (ใบ้: ปัญหานี้คือเหตุผลหนึ่งที่ธุรกิจตัดสินใจย้ายลูกค้ารายใหญ่ไป database-per-tenant ตาม
   Step 848 — ลองคิดว่าจะเพิ่ม index อะไรที่ช่วยลดผลกระทบได้บ้างก่อนที่จะต้องแยกฐานข้อมูล)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Multi-tenancy** คือการให้แอปพลิเคชัน instance เดียวรองรับลูกค้า (tenant) หลายราย พร้อม
  รับประกันว่าข้อมูลของแต่ละรายแยกจากกันสมบูรณ์ — เป็นรากฐานของ SaaS แทบทุกตัว เพราะลดต้นทุน
  operational เทียบกับ single-tenancy อย่างมหาศาล
- **Row-based (shared schema)** เพิ่มคอลัมน์ `organization_id`/`tenant_id` ในทุกตาราง
  implement ง่ายและต้นทุนต่ำที่สุด แต่มีความเสี่ยงร้ายแรง: **การลืม `where(organization_id:
  ...)` แม้แต่จุดเดียวในทั้งโปรเจกต์ก็ทำให้ข้อมูลรั่วไหลข้าม tenant ได้ทันทีโดยไม่มี error ใดๆ
  เตือน** — สาธิตให้เห็นการรั่วไหลจริงในสถานการณ์ที่สมจริง (widget ที่ engineer คนหลังลืมกรอง)
- ทางแก้ที่ถูกต้องคือ **automatic scoping ที่ระดับ model** ผ่าน gem `acts_as_tenant` ซึ่งใช้
  `default_scope` เหมือนที่ Part 034 เตือนไว้ว่าอันตราย **แต่เพิ่ม safety net ที่จำเป็น**: raise
  `NoTenantSet` แทนคืนค่าว่างเงียบๆ, มี escape hatch ที่ตั้งใจเรียก
  (`ActsAsTenant.without_tenant`), validate ป้องกัน cross-tenant assignment, และเก็บ
  current tenant ด้วย **`ActiveSupport::CurrentAttributes`** ซึ่งปลอดภัยต่อ thread/request
  มากกว่าตัวแปร global ธรรมดา
- **Schema-based** แยก PostgreSQL schema ต่อ tenant ให้ isolation แข็งแรงกว่า (บั๊ก "ลืมกรอง"
  กลายเป็น error แทนที่จะรั่วไหลเงียบๆ เพราะเป็นคนละตารางจริงๆ ในทางกายภาพ) แต่แลกมาด้วย
  operational complexity ที่สูงขึ้นชัดเจน: migration ต้องรันแยกทุก schema, connection
  pooling ซับซ้อนขึ้น, และ PostgreSQL catalog รับภาระมากขึ้นเมื่อจำนวน schema สูง
- **ตรวจสอบสถานะ gem `apartment` อย่างตรงไปตรงมา:** เวอร์ชันล่าสุด (2.2.1) ออกเมื่อ 19 มิ.ย.
  2019 และ gemspec ล็อก `activerecord < 6.0` ไว้ ทำให้ **ไม่รองรับ Rails 8 อย่างเป็นทางการ**
  เมื่อทดสอบบูตแอปจริงด้วยเวอร์ชันเดียวที่ Bundler ยอมติดตั้ง (0.24.3) แอป **พังทันทีตอน boot**
  ด้วย `NoMethodError` เพราะโค้ดใช้วิธี register middleware แบบเก่าที่ Rails รุ่นใหม่ไม่รองรับ
  แล้ว — สรุปได้ชัดเจนว่า **`apartment` ไม่ใช่ตัวเลือกที่เหมาะสำหรับโปรเจกต์ Rails 8 ใหม่**
  ควรใช้ **`acts_as_tenant`** แทน (เวอร์ชันล่าสุดออกเมื่อไม่กี่วันก่อน ทดสอบติดตั้งและใช้งาน
  กับ Rails 8.1.4 ได้จริงไม่มีปัญหา)
- **Database-per-tenant** ให้ isolation สูงสุด (แยกฐานข้อมูลจริงทางกายภาพ) เหมาะกับลูกค้าที่มี
  ข้อกำหนด compliance สูง (healthcare, การเงิน, data residency) แต่ต้นทุนดำเนินงานสูงที่สุด
  ทั้ง migration, connection pooling, และ backup/monitoring
- **หลักการเลือกกลยุทธ์จากความต้องการธุรกิจจริง:** SaaS ส่วนใหญ่ควรเริ่มด้วย row-based เพื่อ
  ความเร็วในการพัฒนา แล้วย้ายไป schema-based/database-per-tenant เฉพาะเมื่อมีสัญญาณทางธุรกิจ
  ชัดเจน (ลูกค้า enterprise เรียกร้อง, ข้อกำหนดกฎหมาย, noisy neighbor) — แนวทาง hybrid ที่ยก
  เฉพาะลูกค้าบางรายไปอยู่ระดับ isolation สูงกว่าก็เป็นทางเลือกที่ใช้ได้จริง
- **Subdomain-based tenant resolution** (`acme.myapp.com`) เป็น UX pattern มาตรฐานของ SaaS
  ที่ให้ผู้ใช้ระบุ tenant ผ่าน URL โดยไม่ต้อง login ก่อน — `acts_as_tenant` มี
  `set_current_tenant_by_subdomain` สำเร็จรูปที่ตั้ง current tenant ให้อัตโนมัติก่อนทุก action
- **การทดสอบ isolation เป็นสิ่งจำเป็น ไม่ใช่ทางเลือก** เพราะบั๊ก tenant leak ไม่ทำให้เกิด error
  ที่รีวิวโค้ดจับได้ง่ายๆ ต้องมีชุดทดสอบเฉพาะที่สร้างข้อมูลอย่างน้อย 2 tenant เสมอ และทดสอบทั้ง
  read (`.all`, `.find`) และ write (การ assign ผิด tenant) — ควรรันเป็นส่วนหนึ่งของ CI ทุก
  pull request เหมือน security test อื่นๆ

**ต่อไป (Part 086):** เมื่อแอปพลิเคชันเดียวเริ่มรองรับผู้ใช้จำนวนมากขึ้นเรื่อยๆ จนถึงจุดที่ทีม
ต้องแยกส่วนต่างๆ ออกเป็นระบบย่อยที่ deploy อิสระจากกัน Part หน้าจะพาไปสำรวจ **Microservices
กับ Rails** — วิธีให้ Rails application หลายตัวสื่อสารกัน (service-to-service communication
ผ่าน HTTP/gRPC) และการใช้ **message queue** อย่าง RabbitMQ/Kafka เบื้องต้น เพื่อให้ระบบ
ทำงานร่วมกันแบบ asynchronous โดยไม่ต้องรอกันโดยตรง — ปิดท้าย **Phase 14: Architecture &
Scaling** ทั้งเฟส
