# Part 067: Active Storage กับ Cloud Storage (S3-compatible) และ Image Processing เชิงลึก

> **Step ครอบคลุมใน Part นี้:** Step 661–670
> **ระดับ:** กลาง (ต้องผ่าน Part 066 มาก่อน — `has_one_attached`, local disk service, direct upload พื้นฐาน, variant พื้นฐาน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6 / Rails 8.1.4 / gem `image_processing` ~> 1.2 (ตัว backend คือ **libvips**) /
> gem `aws-sdk-s3` ~> 1.146

> **หมายเหตุการตรวจสอบ (verification note):** ตัวอย่างโค้ดเกือบทั้งหมดใน Part นี้รันจริงบน
> Rails app ทดลอง (Ruby 3.3.6 / Rails 8.1.4) ที่สร้างขึ้นเฉพาะสำหรับเขียนบทเรียนนี้ ส่วนที่เป็น
> "cloud storage" นั้น เราคอมไพล์และรัน **MinIO** (S3-compatible object storage server ตัวจริง,
> พูดโปรโตคอล AWS Signature V4 เดียวกับ Amazon S3 เป๊ะๆ) ขึ้นมาบน sandbox แล้วทดสอบ
> upload/download/presigned URL/variant processing กับมันจริงๆ ทุกจุดที่บอกว่า "ทดสอบแล้ว"
> คือรันจริงและเห็นผลลัพธ์จริงตามที่แสดงในเอกสารนี้ ส่วนที่ sandbox ไม่มีทางทดสอบได้จริง
> (เช่น พฤติกรรม ephemeral filesystem ข้าม server instance จริงบน Heroku/Render, การตั้งค่า
> Cloudflare R2/DigitalOcean Spaces กับบัญชีจริง, การวัด benchmark libvips เทียบ ImageMagick ที่ไม่ได้
> ติดตั้งใน sandbox นี้) จะระบุไว้ชัดเจนว่าเป็น "เหตุผลเชิงสถาปัตยกรรม/อ้างอิงจากเอกสารทางการ"
> ไม่ใช่ผลทดสอบของเราเอง

## สารบัญของ Part นี้

- Step 661: ทำไม local disk storage ใช้กับ production จริงไม่ได้ (ephemeral filesystem)
- Step 662: ตั้งค่า service `:amazon` (Amazon S3 จริง) ใน `config/storage.yml` และ credentials
- Step 663: ทางเลือกที่พูดโปรโตคอล S3 ได้เหมือนกัน — MinIO, Cloudflare R2, DigitalOcean Spaces
- Step 664: สลับ service ตาม environment (`:local` ตอน dev, `:amazon` ตอน production)
- Step 665: Direct upload ไป S3 โดยตรง — เจาะลึก presigned URL flow
- Step 666: Image processing เชิงลึกด้วย libvips — resize, crop, format, quality/compression
- Step 667: Named variant (Rails 7+) — ลงทะเบียน variant ไว้ที่โมเดล ใช้ซ้ำได้ไม่ต้องพิมพ์ option ซ้ำ
- Step 668: `analyze` และ `identify` — ดึง metadata ของไฟล์ (ขนาดภาพ, การตรวจ content type จริง)
- Step 669: Background variant processing — ไม่ให้ผู้ใช้คนแรกต้องรอ process รูป
- Step 670: CDN, proxy vs redirect mode, และเรื่อง cost/security ที่ต้องรู้ก่อนขึ้น production
- แบบฝึกหัด: ตั้งค่าอัปโหลดรูปสินค้าให้ทำงานได้ทั้ง local dev และ S3-compatible service
- สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

---

## Step 661: ทำไม local disk storage ใช้กับ production จริงไม่ได้

ใน Part 066 เราตั้งค่า Active Storage ให้เก็บไฟล์ลง **local disk** (`service: Disk` ใน
`config/storage.yml`) ซึ่งใช้งานได้ดีมากตอน development เพราะเรียบง่าย ไม่ต้องมี account
คลาวด์ ไม่มีค่าใช้จ่าย และไฟล์อยู่ตรงหน้าเรา เปิดดูด้วย `ls` ได้เลย

แต่พอจะขึ้น production จริง local disk มีปัญหาใหญ่ที่มาจากธรรมชาติของ hosting platform
สมัยใหม่แทบทุกเจ้า (Heroku, Render, Fly.io, Railway, Kubernetes, หรือแม้แต่ container ที่ deploy
ด้วย Kamal โดยไม่ได้ mount persistent volume ไว้อย่างตั้งใจ) นั่นคือ **ephemeral filesystem**

### Ephemeral filesystem คืออะไร

"Ephemeral" แปลว่า "ชั่วครู่ชั่วยาม" — filesystem ของ container/dyno ที่แอปรันอยู่ **ไม่ได้ถูก
การันตีว่าจะอยู่ถาวร** เหตุการณ์ต่อไปนี้ทำให้ไฟล์ที่เขียนลง disk ของ container หายไปทันที:

1. **Deploy ใหม่** — ทุกครั้งที่ deploy โค้ดเวอร์ชันใหม่ แพลตฟอร์มส่วนใหญ่จะสร้าง container
   ตัวใหม่ขึ้นมาแทนตัวเก่าทั้งหมด (ไม่ใช่แก้ไฟล์บนตัวเดิม) ตัวเก่าที่มีไฟล์ที่ผู้ใช้อัปโหลดไว้
   จะถูกทำลายทิ้ง
2. **Restart / crash** — dyno restart อัตโนมัติ (Heroku restart ทุก dyno อย่างน้อยวันละครั้งเป็น
   ปกติ) หรือ container crash แล้วถูก orchestrator สร้างใหม่ ก็ล้าง disk เหมือนกัน
3. **Autoscaling / หลาย server instance** — นี่คือปัญหาที่ร้ายแรงกว่านั้นอีก แม้ไม่มี deploy/restart
   เลย แอปที่รันมากกว่า 1 instance พร้อมกัน (เพื่อรองรับโหลด หรือเพื่อ high availability)
   แต่ละ instance มี disk เป็นของตัวเอง **แยกกันโดยสิ้นเชิง**

### ทำไมข้อ 3 ถึงเป็นปัญหาที่คนเจอบ่อยที่สุด

สมมติแอปมี 3 web server instance (A, B, C) อยู่หลัง load balancer:

```
ผู้ใช้ X อัปโหลดรูปโปรไฟล์
         │
         ▼
   Load Balancer ─────► ส่ง request ไปที่ instance A (สุ่มหรือ round-robin)
         │
         ▼
   instance A บันทึกไฟล์ลง /app/storage/xx/yy/xxxxxxxx (disk ของ A เท่านั้น)


ผู้ใช้ Y เปิดหน้าโปรไฟล์ของผู้ใช้ X (คนละ request ใหม่)
         │
         ▼
   Load Balancer ─────► ส่ง request ไปที่ instance B (คนละตัวกับตอนอัปโหลด!)
         │
         ▼
   instance B หาไฟล์ที่ /app/storage/xx/yy/xxxxxxxx แต่ไม่มี
   เพราะไฟล์นั้นอยู่บน disk ของ A เท่านั้น
         │
         ▼
   404 Not Found — รูปหาย ทั้งที่เพิ่งอัปโหลดไปเมื่อกี้นี้เอง
```

นี่ไม่ใช่ edge case ที่นานๆ เกิดที — มันเกิด**ทุกครั้ง**ที่ request ที่สองถูก route ไปคนละ
instance กับตอนอัปโหลด ซึ่งเป็นเรื่องปกติมากเมื่อแอปมีมากกว่า 1 instance (ซึ่งแทบทุกแอป
production ที่จริงจังจะมี อย่างน้อยก็เพื่อ zero-downtime deploy)

> **ตรงนี้คือเหตุผลเชิงสถาปัตยกรรม ไม่ใช่ผลทดสอบใน sandbox:** sandbox ที่ใช้เขียนบทเรียนนี้
> เป็น container เดี่ยว ไม่มีทางจำลอง "หลาย server instance พร้อมกัน" ให้เห็น 404 จริงๆ ได้
> แต่พฤติกรรมนี้เป็นสิ่งที่ทีมวิศวกรรมของ Heroku, Render, Fly.io ระบุไว้ชัดเจนในเอกสารของ
> ตัวเองว่า filesystem ของแต่ละ dyno/instance เป็น local และ ephemeral โดยเจตนา (เพื่อให้
> scale แนวนอนได้ง่าย) และเป็นสาเหตุอันดับต้นๆ ที่ทำให้ทีมที่เพิ่งย้ายจาก local disk ไป
> production เจอบั๊ก "รูปหายเป็นพักๆ" แบบสุ่มที่ debug ยากมาก เพราะมันไม่ fail ทุกครั้ง
> (fail เฉพาะตอน route ไปคนละ instance)

### ทางออก: Object storage แบบ centralized

Cloud object storage (S3, Google Cloud Storage, Azure Blob, หรือบริการ S3-compatible อย่าง
MinIO/R2/Spaces) แก้ปัญหานี้เพราะเป็น**บริการแยกต่างหาก**ที่อยู่นอก container ของแอป
ทุก instance ไม่ว่าจะเป็น A, B หรือ C ก็เชื่อมต่อไปยัง object storage ตัวเดียวกันผ่าน network
ไม่ใช่ disk ของตัวเอง

```
   instance A ──┐
   instance B ──┼──► S3 Bucket เดียวกัน (อยู่ตลอด ไม่หายตอน deploy/restart/scale)
   instance C ──┘
```

ไม่ว่า request จะถูก route ไปที่ instance ไหน ก็เห็นไฟล์เดียวกันเสมอ เพราะไฟล์ไม่ได้อยู่บน
disk ของ instance ใดๆ เลย มันอยู่บน S3

Active Storage ถูกออกแบบมาให้ **สลับจาก local disk ไปเป็น S3 (หรือเทียบเท่า) ได้โดยไม่ต้อง
แก้โค้ดแอปสักบรรทัดเดียว** — แก้แค่ config เท่านั้น นี่คือสิ่งที่เราจะทำใน Step ถัดไป

---

## Step 662: ตั้งค่า service `:amazon` ใน `config/storage.yml`

`config/storage.yml` คือไฟล์เดียวที่กำหนดว่า Active Storage มี "service" (ปลายทางเก็บไฟล์)
อะไรบ้าง — แต่ละ service คือ block ใน YAML ที่มี key `service:` บอกว่าใช้ adapter ไหน

ไฟล์เริ่มต้นที่ Rails generate ให้ (ที่เราใช้มาตั้งแต่ Part 066) หน้าตาประมาณนี้:

```yaml
# config/storage.yml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# ตัวอย่าง service สำหรับ Amazon S3 (comment ไว้เป็น default)
# amazon:
#   service: S3
#   access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
#   secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
#   region: us-east-1
#   bucket: your_own_bucket-<%= Rails.env %>
```

### เพิ่ม gem ที่จำเป็น

Active Storage ไม่ได้ผูก S3 SDK มาให้ตั้งแต่ต้น (เพื่อไม่บังคับให้ทุกแอปต้องมี gem นี้)
ต้องเพิ่มเองใน `Gemfile`:

```ruby
# Gemfile
gem "aws-sdk-s3", "~> 1.146", require: false
```

```bash
bundle install
```

`require: false` เพราะ Active Storage จะ `require "aws-sdk-s3"` เองตอนที่รู้ว่ามี service
ไหนใช้ `service: S3` บ้าง ไม่จำเป็นต้อง require ไว้ล่วงหน้าตอนบูตแอปทุกครั้ง

### เปิดใช้งาน service `:amazon`

```yaml
# config/storage.yml
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1
  bucket: myapp-production-uploads
```

**อธิบายแต่ละ key:**

- `service: S3` — บอก Active Storage ว่า service นี้ใช้ adapter สำหรับ AWS S3
  (`ActiveStorage::Service::S3Service`) ซึ่งภายในใช้ gem `aws-sdk-s3` ที่เพิ่งเพิ่มไป
- `access_key_id` / `secret_access_key` — credential ของ IAM user ที่มีสิทธิ์เขียนอ่าน bucket นี้
  **ห้ามเขียนค่าตรงๆ ในไฟล์นี้เด็ดขาด** เพราะไฟล์นี้ถูก commit เข้า git ทุกครั้ง เราจึงดึงจาก
  Rails encrypted credentials แทน (recap ด้านล่าง)
- `region` — ภูมิภาคของ AWS ที่ bucket นี้อยู่ (ต้องตรงกับตอนสร้าง bucket จริง)
- `bucket` — ชื่อ bucket ที่จะเก็บไฟล์ (ชื่อ bucket ต้อง unique ทั่วทั้ง AWS ไม่ใช่แค่ในบัญชีเรา)

### Recap: Rails encrypted credentials

Rails ตั้งแต่ 5.2 มีระบบเก็บ secret แบบเข้ารหัสในตัว ไฟล์ `config/credentials.yml.enc` ถูก
เข้ารหัสด้วย master key ที่อยู่ใน `config/master.key` (ไฟล์นี้ **ไม่** ถูก commit เข้า git —
`.gitignore` กันไว้ให้อัตโนมัติตั้งแต่ `rails new`)

```bash
# เปิดแก้ credentials ด้วย editor ที่ตั้งไว้ใน $EDITOR
EDITOR="code --wait" bin/rails credentials:edit

# หรือแก้เฉพาะ environment เดียว (Rails 6+ รองรับ credentials แยกต่อ environment)
EDITOR="code --wait" bin/rails credentials:edit --environment production
```

คำสั่งนี้จะถอดรหัสไฟล์ลง tmp file ชั่วคราว เปิด editor ให้แก้ แล้วเข้ารหัสกลับเมื่อปิด editor
เนื้อหาข้างในเป็น YAML ธรรมดา:

```yaml
# config/credentials.yml.enc (เนื้อหาหลังถอดรหัส - ห้าม commit เวอร์ชันถอดรหัส)
aws:
  access_key_id: AKIAxxxxxxxxxxxxxxxx
  secret_access_key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

จากนั้นในโค้ด (เช่นใน `storage.yml`) เรียกอ่านค่าด้วย `Rails.application.credentials.dig(...)`
ตามที่เห็นด้านบน — `dig` ปลอดภัยกว่า `credentials.aws.access_key_id` ตรงที่ไม่ raise error
ถ้า key ไม่มี (คืน `nil` เฉยๆ) ซึ่งมีประโยชน์มากตอน dev ที่ยังไม่ได้ตั้งค่า credential จริง

เราทดสอบ flow นี้จริงในแอปทดลอง โดยเขียนไฟล์ credentials ที่เข้ารหัสด้วย master key
แล้วอ่านค่ากลับผ่าน `Rails.application.credentials.dig`:

```irb
$ bin/rails runner 'puts Rails.application.credentials.dig(:aws, :access_key_id)'
AKIA_FAKE_EXAMPLE
```

ยืนยันได้ว่ากลไก encrypted credentials → `storage.yml` → `ActiveStorage::Service::S3Service`
ทำงานถูกต้องตามที่ตั้งไว้ (ในการทดสอบนี้ใช้ access key ปลอมเพราะไม่มีบัญชี AWS จริงใน
sandbox แต่กลไกการอ่าน/ถอดรหัส credential เป็นของจริงทั้งหมด)

### สิทธิ์ของ IAM user ควรมีแค่ไหน

**อย่าใช้ root account หรือ IAM user ที่มีสิทธิ์ `AdministratorAccess`** สำหรับงานนี้ ควรสร้าง
IAM user ใหม่ที่มี policy จำกัดเฉพาะ bucket เดียวและ action ที่จำเป็นเท่านั้น:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::myapp-production-uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::myapp-production-uploads"
    }
  ]
}
```

ถ้า key นี้หลุดไปสู่มือคนร้าย ความเสียหายจะจำกัดอยู่แค่ bucket นี้ bucket เดียว ไม่กระทบ
ทรัพยากร AWS อื่นในบัญชีทั้งหมด (รายละเอียดด้าน security เพิ่มเติมอยู่ใน Step 670)

---

## Step 663: ทางเลือกที่พูดโปรโตคอล S3 ได้เหมือนกัน (S3-compatible storage)

จุดที่หลายคนไม่รู้คือ **`service: S3` ใน `storage.yml` ไม่ได้ผูกกับ Amazon เท่านั้น** —
`ActiveStorage::Service::S3Service` ใช้ gem `aws-sdk-s3` ซึ่งจริงๆ แล้วเป็น HTTP client ที่พูด
"S3 API" (REST API มาตรฐานที่ประกอบด้วยการเซ็น request แบบ AWS Signature V4, การเรียก
`PutObject`, `GetObject`, `ListObjects` ฯลฯ) — และ S3 API แบบนี้กลายเป็น **มาตรฐานอุตสาหกรรม
โดยพฤตินัย** ที่ผู้ให้บริการ object storage เจ้าอื่นๆ ก็ implement ตามเพื่อให้ลูกค้าย้ายมาใช้ได้
โดยไม่ต้องเขียนโค้ดใหม่

นั่นแปลว่า service เดียวกันนี้ (`service: S3` + gem `aws-sdk-s3`) ใช้ได้กับ:

| ผู้ให้บริการ | จุดเด่น | ใช้ต่างจาก AWS ตรงไหนใน config |
|---|---|---|
| **Amazon S3** | มาตรฐานอ้างอิง, ระบบนิเวศใหญ่สุด | ไม่ต้องระบุ `endpoint` (ใช้ default ของ AWS) |
| **MinIO** | self-hosted, รันบนเครื่องตัวเอง/on-prem ได้, ฟรี, เหมาะกับ dev/staging/on-prem จริงจัง | ต้องระบุ `endpoint` เป็น URL ของ MinIO server + `force_path_style: true` |
| **Cloudflare R2** | **ไม่คิดค่า egress (ค่าดึงข้อมูลออก) เลย** ต่างจาก S3 ที่คิดค่า egress แพง | ต้องระบุ `endpoint` เป็น `https://<account_id>.r2.cloudflarestorage.com`, `region: auto` |
| **DigitalOcean Spaces** | ราคาคงที่ ผูกกับ Droplet ได้ง่าย เหมาะทีมเล็ก | ต้องระบุ `endpoint` เป็น `https://<region>.digitaloceanspaces.com` |

การเปลี่ยนตัวจริงๆ แล้ว**ต่างกันแค่ 2 บรรทัด**ใน `storage.yml`: `endpoint` และ
`force_path_style` (บางเจ้าไม่ต้องใช้) โค้ดแอปทั้งหมด (`has_one_attached`, `.attach`,
`.variant`, direct upload) **ไม่ต้องแก้อะไรเลยแม้แต่บรรทัดเดียว**

### ทำไมเรื่องนี้ถึงสำคัญกับ cost และความยืดหยุ่น

1. **หนี vendor lock-in ได้จริง** — ถ้าวันหนึ่งค่า egress ของ AWS แพงเกินไป (ทีมที่ให้บริการ
   สตรีมวิดีโอ/serve รูปจำนวนมากมักเจอบิล egress หลักหมื่นถึงหลักแสนบาทต่อเดือน) การย้ายไป
   Cloudflare R2 (egress ฟรี 100%) ทำได้แค่เปลี่ยน 5 บรรทัดใน `storage.yml` + copy ข้อมูลเก่า
   ไม่ต้องเขียน service layer ใหม่ ไม่ต้องแตะโค้ด model หรือ controller เลย
2. **ทดสอบ cloud storage แบบไม่มีค่าใช้จ่ายและไม่ต้องมีเน็ต** — นี่คือสิ่งที่เราใช้ตลอดบทเรียนนี้
   MinIO รันเป็น process บนเครื่อง dev/CI ได้ ทำให้ทดสอบ direct upload, presigned URL,
   permission model ของ S3 ได้เหมือนของจริงทุกกระเบียดนิ้ว โดยไม่ต้องมี AWS account
   และไม่มีความเสี่ยงเรื่องเงินหลุดจากการทดสอบผิดพลาด
3. **Data residency / on-prem requirement** — บางองค์กร (ธนาคาร, หน่วยงานรัฐ) มีข้อกำหนดว่า
   ข้อมูลต้องอยู่ใน data center ของตัวเองเท่านั้น MinIO ตอบโจทย์นี้ได้เพราะ deploy เองได้
   100% โดยยังใช้โค้ด Active Storage เดิมทุกบรรทัด

### ทดสอบจริง: รัน MinIO แล้วต่อจาก Rails

เราคอมไพล์ MinIO server จาก source (`go install github.com/minio/minio@latest`) แล้วรันขึ้นมา
บน sandbox จริงในพอร์ต 9000:

```bash
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin123
minio server ./minio-data --address ":9000" --console-address ":9001"
```

```
API: http://127.0.0.1:9000
WebUI: http://127.0.0.1:9001
```

เพิ่ม service ใน `storage.yml`:

```yaml
# config/storage.yml
minio:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:minio, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:minio, :secret_access_key) %>
  region: us-east-1
  bucket: dev-uploads
  endpoint: <%= ENV.fetch("MINIO_ENDPOINT", "http://localhost:9000") %>
  force_path_style: true   # สำคัญมาก อธิบายด้านล่าง
```

> **`force_path_style: true` คืออะไร:** AWS S3 จริงใช้ "virtual-hosted style" URL เป็น default
> คือ `https://<bucket>.s3.amazonaws.com/<key>` (ชื่อ bucket อยู่ใน subdomain) แต่ MinIO
> (และหลายเจ้าที่ self-host) ไม่รองรับ DNS wildcard subdomain per-bucket แบบนั้น จึงต้อง
> บังคับให้ SDK ใช้ "path style" แทน คือ `http://<endpoint>/<bucket>/<key>` (ชื่อ bucket อยู่ใน
> path) การลืมตั้ง flag นี้เป็นสาเหตุอันดับหนึ่งที่คนตั้งค่า MinIO/S3-compatible แล้วต่อไม่ติด

สร้าง bucket แล้วทดสอบ upload/download จริงผ่าน `aws-sdk-s3`:

```ruby
require "aws-sdk-s3"

client = Aws::S3::Client.new(
  access_key_id: "minioadmin",
  secret_access_key: "minioadmin123",
  region: "us-east-1",
  endpoint: "http://localhost:9000",
  force_path_style: true
)
client.create_bucket(bucket: "dev-uploads")
```

จากนั้นสลับ Active Storage ไปใช้ service นี้แล้วอัปโหลดไฟล์จริง:

```ruby
ActiveStorage::Blob.service = ActiveStorage::Blob.services.fetch(:minio)

blob = ActiveStorage::Blob.create_and_upload!(
  io: File.open("test_photo.jpg"),
  filename: "s3-chair.jpg",
  content_type: "image/jpeg",
  service_name: "minio"
)
```

**ผลลัพธ์จริงจากการรันในแอปทดลอง:**

```
blob key = 8tpg36xbjrvc659o0xbcfgu9cuic
blob service_name = minio
blob byte_size = 1202082
download กลับมาได้ 1202082 bytes (ตรงกับต้นฉบับ: true)
```

ไฟล์ 1,202,082 ไบต์ถูกส่งผ่าน HTTP ไปเก็บที่ MinIO จริง แล้วดาวน์โหลดกลับมาได้ขนาดตรงกัน
เป๊ะ — พิสูจน์ว่า flow "Rails → aws-sdk-s3 → S3-compatible server" ทำงานถูกต้องโดยไม่ต้อง
มี AWS account จริงเลยแม้แต่นิดเดียว

### Cloudflare R2 และ DigitalOcean Spaces (ตั้งค่าตามหลักการเดียวกัน)

```yaml
# config/storage.yml
cloudflare_r2:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:r2, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:r2, :secret_access_key) %>
  region: auto
  bucket: myapp-uploads
  endpoint: <%= Rails.application.credentials.dig(:r2, :endpoint) %>
  # endpoint ตัวอย่าง: https://<account_id>.r2.cloudflarestorage.com

do_spaces:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:spaces, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:spaces, :secret_access_key) %>
  region: sgp1
  bucket: myapp-uploads
  endpoint: https://sgp1.digitaloceanspaces.com
```

> **สิ่งที่ตรวจสอบได้จริง vs สิ่งที่อ้างอิงจากเอกสาร:** เรายืนยันกลไก "S3 API + endpoint ที่
> เปลี่ยนได้ + force_path_style" ด้วยการทดสอบจริงกับ MinIO ซึ่งพิสูจน์ตัวกลไกทางเทคนิคทั้งหมด
> ที่ Active Storage ใช้ได้ 100% (การเซ็น request แบบ Signature V4, การเรียก REST API,
> การอัปโหลด/ดาวน์โหลด) ส่วนค่า `endpoint`/`region` ที่ตรงกับ Cloudflare R2 และ DigitalOcean
> Spaces จริงๆ นั้นเราคัดลอกมาจากเอกสารทางการของแต่ละเจ้า (เนื่องจาก sandbox นี้ไม่มีบัญชี
> R2/Spaces จริงให้ทดสอบ) รูปแบบ config เหมือนกันทุกประการกับที่ทดสอบผ่านกับ MinIO
> เพียงแค่เปลี่ยนค่า endpoint เท่านั้น จึงมั่นใจได้ว่าโครงสร้างถูกต้อง

---

## Step 664: สลับ service ตาม environment

ตอน dev เราไม่อยากพึ่งเน็ตหรือ MinIO server ที่ต้องรันแยก (เว้นแต่กำลังทดสอบเรื่อง cloud
storage โดยเฉพาะแบบ Step ก่อนหน้า) — Active Storage แก้ปัญหานี้ได้ง่ายมากด้วยการตั้งค่า
**คนละ service ในแต่ละ environment file**

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ...
  config.active_storage.service = :local
end
```

```ruby
# config/environments/test.rb
Rails.application.configure do
  # ...
  config.active_storage.service = :test
end
```

```ruby
# config/environments/production.rb
Rails.application.configure do
  # ...
  config.active_storage.service = :amazon
end
```

โค้ดในโมเดล/controller/view **ไม่ต้องรู้เลยว่าตอนนี้ใช้ service ไหน** — เขียน
`product.photo.attach(...)` เหมือนกันทุก environment ส่วน Active Storage จะไปดูค่า
`config.active_storage.service` ของ environment ปัจจุบันแล้วเลือก service ที่ตรงกับ key นั้น
ใน `storage.yml` ให้เอง

เราตรวจสอบพฤติกรรมนี้จริงในแอปทดลองโดยดู `ActiveStorage::Blob.service.name` หลังบูตแอปใน
environment ต่างๆ และยืนยันว่าตรงกับ config ที่ตั้งไว้ทุกครั้ง

### กรณีต้องการ service เฉพาะ attachment (ไม่ใช้ default ทั้ง environment)

บางครั้งอยากให้ attachment บางตัวใช้ service คนละตัวกับ default (เช่น รูปโปรไฟล์เก็บที่
bucket แบบ public-read แต่เอกสาร invoice เก็บที่ bucket แบบ private ล้วน) — ระบุ `service:`
ตรงๆ ที่ `has_one_attached`/`has_many_attached` ได้เลย ไม่ต้องพึ่ง environment config:

```ruby
class Product < ApplicationRecord
  has_one_attached :photo               # ใช้ default service ของ environment
end

class User < ApplicationRecord
  has_one_attached :avatar, service: :public_bucket   # บังคับใช้ service นี้เสมอ ไม่ว่า environment ไหน
end
```

### สลับ service ตอน runtime (เทคนิคขั้นสูง ใช้ตอนเทสต์/สคริปต์ migration)

ในการทดสอบของบทเรียนนี้เราใช้เทคนิคหนึ่งที่มีประโยชน์มากตอนเขียน migration script ย้ายไฟล์
เก่าไป cloud (หรือตอนเขียนเทสต์ที่ต้องสลับ service ชั่วคราว) นั่นคือการกำหนด
`ActiveStorage::Blob.service =` ตรงๆ:

```ruby
# สลับ default service ของ process ปัจจุบันแบบ runtime (ไม่ต้องแก้ environment file)
ActiveStorage::Blob.service = ActiveStorage::Blob.services.fetch(:minio)

# หรือระบุ service ต่อการอัปโหลดครั้งเดียว โดยไม่แตะ default เลย
blob = ActiveStorage::Blob.create_and_upload!(
  io: file, filename: "x.jpg", content_type: "image/jpeg",
  service_name: "minio"   # override เฉพาะครั้งนี้
)
```

เทคนิคนี้คือสิ่งที่เราใช้ตลอด Part นี้เพื่อทดสอบกับ MinIO โดยไม่ต้องปิด/เปิดแอปใหม่ทุกครั้งที่
เปลี่ยน service — เหมาะกับ script/console/เทสต์ ส่วนในโค้ด production จริงให้ใช้
`config.active_storage.service` ต่อ environment ตามปกติ (ชัดเจนกว่า ไม่มี state แปลกๆ หลงเหลือ
ข้าม request)

---

## Step 665: Direct upload ไป S3 โดยเฉพาะ — เจาะลึก presigned URL flow

Part 066 สอน direct upload ไว้แบบทั่วไป (ใช้ `direct_upload: true` บน `form.file_field`
และ JS `@rails/activestorage` จัดการให้) หลักการคือ **ไฟล์ถูกอัปโหลดตรงจาก browser ไปยัง
storage service เลย โดยไม่ผ่าน Rails server** — Rails server ทำหน้าที่แค่ "ออกตั๋วอนุญาต"
(presigned URL) ให้ browser เอาไปใช้

คำถามคือ: ตอนใช้ local disk service ตั๋วนั้นชี้ไปที่ endpoint ของ Rails เอง (เพราะ local disk
ไม่มี URL สาธารณะ ต้องผ่าน controller ของ Rails เสมอ) แต่พอเปลี่ยนไปใช้ **S3 หรือ S3-compatible
service** ตั๋วนั้นจะชี้ **ตรงไปที่ S3 เลย ไม่ผ่าน Rails server แม้แต่นิดเดียว** — นี่คือจุดที่
ทำให้ direct upload มีประโยชน์จริงจังตอนใช้กับ cloud storage: ไฟล์ขนาดใหญ่ (วิดีโอ, PDF
หลายสิบ MB) ไม่ต้องวิ่งผ่าน Ruby process ของเราเลย ประหยัด bandwidth, memory, และ worker
thread ของ Rails server ไปได้มหาศาล

### พิสูจน์ด้วยการทดสอบจริงกับ MinIO

ขั้นแรก สร้าง blob record แบบ "ประกาศไว้ล่วงหน้า" (เหมือนที่ direct upload controller ของ
Rails ทำเบื้องหลังตอน browser เรียก `POST /rails/active_storage/direct_uploads`):

```ruby
checksum = Digest::MD5.file("test_photo.jpg").base64digest

direct_blob = ActiveStorage::Blob.create_before_direct_upload!(
  filename: "direct-upload-chair.jpg",
  byte_size: File.size("test_photo.jpg"),
  checksum: checksum,
  content_type: "image/jpeg",
  service_name: "minio"
)

presigned_url = direct_blob.service_url_for_direct_upload
puts presigned_url
```

**ผลลัพธ์จริงที่ได้ (คัดลอกมาไม่ได้แก้ไข):**

```
http://localhost:9000/dev-uploads/79orjkpydt6pir19m32msy1gv9gd?
X-Amz-Algorithm=AWS4-HMAC-SHA256&
X-Amz-Credential=minioadmin%2F20260926%2Fus-east-1%2Fs3%2Faws4_request&
X-Amz-Date=20260926T085719Z&
X-Amz-Expires=300&
X-Amz-SignedHeaders=content-length%3Bcontent-md5%3Bcontent-type%3Bhost&
X-Amz-Signature=e364b022a0eb544206b9ae46197a13a3d0124bddd4482764df50141f4c31be89
```

(ตัดบรรทัดเพื่อให้อ่านง่าย ของจริงเป็น URL บรรทัดเดียว)

สังเกตสิ่งสำคัญ 3 อย่าง:

1. **Host คือ `localhost:9000` (MinIO ตรงๆ) ไม่ใช่ Rails server ของเรา** — พิสูจน์ว่า browser
   จะยิง request ไปที่ storage service โดยตรงจริงๆ
2. **`X-Amz-Signature`** คือลายเซ็น HMAC-SHA256 ที่คำนวณจาก secret key ของเรา — ทำให้ MinIO/S3
   เชื่อได้ว่า URL นี้ถูกสร้างโดยฝ่ายที่มี credential ที่ถูกต้องจริงๆ โดยที่ credential
   **ไม่เคยถูกส่งไปให้ browser เลย** (browser ได้แค่ URL ที่เซ็นแล้ว ไม่ได้ access key/secret)
3. **`X-Amz-Expires=300`** — ตั๋วนี้ใช้ได้แค่ 300 วินาที (5 นาที) หลังจากนั้นหมดอายุทันที
   ค่านี้มาจาก `ActiveStorage.service_urls_expire_in` ซึ่งเราตรวจสอบแล้วว่า default คือ
   `300` วินาทีจริงๆ

จากนั้นจำลองสิ่งที่ browser (ผ่าน `@rails/activestorage` JS) จะทำ คือยิง `PUT` ตรงไปยัง
presigned URL นั้นพร้อมเนื้อไฟล์:

```ruby
require "net/http"

uri = URI.parse(presigned_url)
http = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Put.new(uri)
request.body = File.binread("test_photo.jpg")
request["Content-Type"] = "image/jpeg"
request["Content-MD5"] = checksum
response = http.request(request)

puts response.code   # => "200"
```

**ผลลัพธ์จริง: `PUT ตรงไป MinIO -> HTTP 200`** — ไฟล์ไปถึง MinIO สำเร็จโดยที่ Rails server
ไม่เกี่ยวข้องกับการส่งไบต์เลยแม้แต่ไบต์เดียว (Rails ทำแค่สร้าง `direct_blob` record กับออก
presigned URL เท่านั้น) และเมื่อดาวน์โหลดกลับมาตรวจสอบ ขนาดไฟล์ตรงกับต้นฉบับเป๊ะ
(`1202082` bytes ทั้งสองฝั่ง)

### ทำไม `Content-MD5` (checksum) สำคัญ

`checksum` ที่ส่งไปตอนสร้าง `direct_blob` และ header `Content-MD5` ตอน `PUT` ทำหน้าที่เป็น
**integrity check** — S3/MinIO จะคำนวณ MD5 ของไบต์ที่ได้รับจริง แล้วเทียบกับค่าที่ client
ประกาศไว้ล่วงหน้า ถ้าไม่ตรงกัน (เช่น การส่งข้อมูลเสียหายระหว่างทาง หรือมีคนพยายามสลับไฟล์)
S3 จะปฏิเสธ upload ทันที (`400 Bad Request` พร้อม error `BadDigest`) — ป้องกันไม่ให้ blob
record ใน database กับไฟล์จริงบน S3 ไม่ตรงกัน

### เรื่องที่ต้องรู้เพิ่มตอนใช้จริงกับ browser: CORS

การทดสอบด้านบนยิง `PUT` ด้วย `Net::HTTP` ซึ่งไม่มีข้อจำกัดเรื่อง CORS (Cross-Origin Resource
Sharing) เพราะไม่ใช่ browser แต่พอเป็น browser จริงที่หน้าเว็บของเราอยู่คนละ origin กับ
S3/MinIO (แน่นอนว่าคนละ origin เสมอ เพราะเว็บเราอยู่ `https://myapp.com` แต่ S3 อยู่
`https://bucket.s3.amazonaws.com`) browser จะบล็อก request นี้ด้วย CORS policy ถ้า bucket
ไม่ได้ตั้งค่า CORS ไว้ให้อนุญาต

ต้องตั้งค่า CORS ที่ตัว bucket (ไม่ใช่ที่ Rails) เช่นบน AWS S3:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["PUT", "POST"],
    "AllowedOrigins": ["https://myapp.com"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

> **หมายเหตุ:** เราไม่ได้ทดสอบ CORS จริงในบทเรียนนี้เพราะการทดสอบ CORS ต้องใช้ browser จริง
> (CORS เป็นกลไกที่ browser บังคับใช้ ไม่ใช่ server) แต่กลไก presigned URL ที่พิสูจน์แล้วด้านบน
> เป็นกลไกเดียวกันทุกประการกับที่ browser จะเรียกผ่าน `@rails/activestorage` เพียงแต่ต้อง
> เพิ่มการตั้งค่า CORS ที่ bucket ให้ครบก่อนถึงจะใช้งานจาก browser จริงได้

---

## Step 666: Image processing เชิงลึกด้วย libvips

Part 066 สอน variant พื้นฐานไปแล้ว (`resize_to_limit` ง่ายๆ) ใน Step นี้เราจะลงลึกเรื่อง
**processor เบื้องหลัง** และตัวเลือกการ resize/crop/format/compression แบบครบถ้วน

### libvips คือ default ของ Rails 8

```irb
$ bin/rails runner 'puts ActiveStorage.variant_processor'
vips
```

เราตรวจสอบแล้วว่าเมื่อใช้ `config.load_defaults 8.1` (ค่า default ของ `rails new` ปัจจุบัน)
`ActiveStorage.variant_processor` ถูกตั้งเป็น `:vips` โดยอัตโนมัติ (Rails เปลี่ยน default จาก
`:mini_magick` มาเป็น `:vips` ตั้งแต่ `load_defaults 7.1` เป็นต้นมา) หมายความว่าแอป Rails 8
ใหม่ **ไม่ต้องตั้งค่าอะไรเพิ่มเลย** ก็ได้ libvips เป็น processor ทันที แค่ต้องติดตั้งตัว
libvips เองไว้ในระบบ (บน Ubuntu/Debian: `sudo apt install libvips`, บน macOS:
`brew install vips`) กับเพิ่ม gem `image_processing` ใน Gemfile:

```ruby
# Gemfile
gem "image_processing", "~> 1.2"
```

### libvips vs ImageMagick — ทำไมถึงเปลี่ยน default

ImageMagick (และ MiniMagick ที่เป็น Ruby wrapper ของมัน) เป็นมาตรฐานเดิมของวงการ Rails
มานาน แต่มีจุดอ่อนสำคัญเรื่อง**การใช้หน่วยความจำ**:

- **ImageMagick** โดย default จะ decode รูปทั้งภาพเป็น bitmap ที่ไม่ได้บีบอัด (raw pixel
  data) เก็บไว้ใน memory ทั้งหมดก่อนเริ่มประมวลผล แล้วแต่ละ operation (resize, crop, ฯลฯ)
  มักสร้าง buffer ใหม่ทับซ้อนกันไปเรื่อยๆ ยิ่งรูปต้นฉบับใหญ่ (เช่นภาพถ่ายจากกล้องมือถือยุค
  ใหม่ที่ 4000×3000 พิกเซลขึ้นไป หรือภาพสแกนเอกสารความละเอียดสูง) หน่วยความจำที่ต้องใช้ก็
  ยิ่งพุ่งสูงแบบไม่เป็นเชิงเส้น
- **libvips** ถูกออกแบบมาด้วยสถาปัตยกรรมที่ต่างออกไปตั้งแต่ต้น คือเป็น **demand-driven,
  streaming pipeline** — มันแบ่งภาพเป็นแถบเล็กๆ (tile/scanline) แล้วประมวลผลทีละส่วนไหลผ่าน
  operation ต่างๆ โดยไม่จำเป็นต้องถือทั้งภาพไว้ใน memory พร้อมกันทั้งหมด สำหรับงานที่พบบ่อย
  ที่สุดของ Active Storage นั่นคือ "thumbnail รูปใหญ่ให้เล็กลง" (resize + crop) libvips
  ยังฉลาดพอที่จะ**อ่านเฉพาะข้อมูลเท่าที่จำเป็นสำหรับขนาดผลลัพธ์** แทนที่จะ decode เต็มความ
  ละเอียดต้นฉบับก่อนแล้วค่อย resize ทีหลัง (เทคนิคที่เรียกว่า shrink-on-load)

ผลลัพธ์ที่ทีมพัฒนา libvips และผู้ใช้งานจำนวนมากรายงานตรงกันคือ libvips ใช้ peak memory
น้อยกว่า ImageMagick/GraphicsMagick **หลายเท่าตัว** (บ่อยครั้งอยู่ในระดับ 4-8 เท่า) และเร็ว
กว่าอย่างมีนัยสำคัญสำหรับงาน thumbnail ทั่วไป ซึ่งมีผลจริงในระดับ production เมื่อ Sidekiq
worker หลายตัวประมวลผลรูปพร้อมกัน — ถ้าใช้ ImageMagick worker แต่ละตัวอาจกิน RAM หลักร้อย MB
ต่อรูปหนึ่งใบขณะประมวลผล ทำให้ต้อง provision เครื่องที่ RAM สูงกว่าที่ควรจะเป็นมาก
ในขณะที่ libvips ทำงานเดียวกันด้วย memory footprint ที่เล็กกว่ามาก ทำให้รันหลาย worker
พร้อมกันบนเครื่องเดียวกันได้มากขึ้นโดยไม่ OOM

> **ความซื่อสัตย์เรื่องการตรวจสอบ:** sandbox ที่ใช้เขียนบทเรียนนี้**ไม่มี ImageMagick ติดตั้ง
> ไว้** (`which convert` ไม่พบ) จึงไม่สามารถรัน benchmark เทียบ peak memory/เวลาระหว่าง libvips
> กับ ImageMagick แบบวัดจริงในเครื่องนี้ได้ ตัวเลข "4-8 เท่า" ด้านบนมาจากเอกสารและ benchmark
> ที่ทีมพัฒนา libvips เผยแพร่เอง (เทียบกับ ImageMagick, GraphicsMagick, OpenCV, PIL สำหรับงาน
> thumbnail ภาพขนาดต่างๆ) ซึ่งเป็นเหตุผลที่ Rails core team อ้างอิงตอนเปลี่ยน default
> processor เป็น vips ตั้งแต่ Rails 7.1 สิ่งที่เรา**ทดสอบจริง**ในบทเรียนนี้คือยืนยันว่า
> `ActiveStorage.variant_processor` เป็น `:vips` โดย default จริง และ libvips สามารถ
> resize/crop/แปลง format ไฟล์ JPEG ขนาด 1600×1200 (1.2MB) ให้เสร็จได้อย่างถูกต้องและรวดเร็ว
> (ทุก variant ในบทเรียนนี้ประมวลผลเสร็จภายในเสี้ยววินาที) แต่ไม่ได้วัดตัวเลขเปรียบเทียบกับ
> ImageMagick โดยตรง

### ตัวเลือก resize สามแบบหลักที่ต้องแยกให้ออก

`image_processing` gem (ที่ Active Storage เรียกใช้ผ่าน `.variant`) มีตัวเลือก resize 3 แบบ
ที่พฤติกรรมต่างกันชัดเจน:

```ruby
# 1) resize_to_limit — ย่อให้ "ไม่เกิน" ขนาดที่กำหนด แต่ไม่ขยายถ้าต้นฉบับเล็กกว่า
#    รักษา aspect ratio เดิมเสมอ ไม่ครอบตัดอะไรเลย (ผลลัพธ์อาจไม่เป็นสี่เหลี่ยมจัตุรัสพอดี)
product.photo.variant(resize_to_limit: [ 500, 500 ])

# 2) resize_to_fit — เหมือน resize_to_limit แต่ "ขยายได้" ถ้าต้นฉบับเล็กกว่าขนาดเป้าหมาย
product.photo.variant(resize_to_fit: [ 500, 500 ])

# 3) resize_to_fill — ย่อ/ขยายแล้ว "ครอบตัด (crop)" ส่วนเกินออกให้ได้ขนาดที่ระบุ "เป๊ะ"
#    ใช้บ่อยที่สุดสำหรับ thumbnail ที่ต้องการสี่เหลี่ยมจัตุรัสขนาดคงที่ (เช่น avatar, thumbnail กริดสินค้า)
product.photo.variant(resize_to_fill: [ 100, 100 ])
```

เราทดสอบทั้งสามแบบจริงกับภาพ JPEG ทดสอบ (1600×1200, 1,202,082 ไบต์):

| variant option | ขนาดผลลัพธ์ | byte size จริงที่วัดได้ |
|---|---|---|
| `resize_to_fill: [100, 100]` | 100×100 พอดี (ครอบตัด) | 1,318 ไบต์ |
| `resize_to_fill: [200, 200]` | 200×200 พอดี (ครอบตัด) | 6,442 ไบต์ |
| `resize_to_limit: [500, 500]` | ไม่เกิน 500×500 รักษาสัดส่วน | 48,137 ไบต์ |

(ภาพต้นฉบับที่ใช้ทดสอบเป็นภาพ noise สุ่มความละเอียดสูงเพื่อจำลอง "ภาพถ่ายจริง" ที่ไม่ใช่
สีพื้นล้วน — ตัวเลขไบต์จึงสะท้อนพฤติกรรมการบีบอัดของ JPEG กับข้อมูลที่มีรายละเอียดเยอะจริง
ไม่ใช่ค่าคงที่ตายตัว แต่แนวโน้ม "ยิ่งภาพใหญ่ ยิ่งกินพื้นที่มากขึ้น" นั้นถูกต้องเสมอ)

### แปลง format และปรับ quality/compression

```ruby
# แปลงเป็น WebP (ไฟล์เล็กกว่า JPEG ที่คุณภาพใกล้เคียงกันสำหรับภาพถ่ายทั่วไป
# แม้ในกรณีภาพ noise แบบทดสอบนี้ที่ไม่ได้มี structure ให้บีบอัดง่ายเหมือนภาพถ่ายจริง)
product.photo.variant(resize_to_limit: [ 800, 800 ], format: :webp, saver: { quality: 75 })

# ควบคุมคุณภาพ JPEG โดยตรง (ค่ายิ่งต่ำ ไฟล์ยิ่งเล็ก แต่คุณภาพยิ่งลด)
product.photo.variant(resize_to_limit: [ 1200, 1200 ], saver: { quality: 80 })
```

เราทดสอบและยืนยันว่า `format: :webp` ทำให้ blob ที่ได้มี `content_type` เป็น
`"image/webp"` จริง (ตรวจสอบด้วย `variant.processed.image.blob.content_type`) และ
`saver: { quality: N }` ถูกส่งต่อไปยัง libvips ที่ตอน encode ไฟล์ผลลัพธ์จริง (ควบคุมระดับ
lossy compression) — ค่า `quality` ยิ่งต่ำ ไฟล์ยิ่งเล็กแต่รายละเอียดยิ่งหาย ค่าที่นิยมใช้กัน
ในงานจริงคือ 75-85 สำหรับภาพสินค้า/ภาพทั่วไป (จุดสมดุลระหว่างขนาดไฟล์กับคุณภาพที่ตาเปล่า
มองแทบไม่ออกว่าถูกบีบอัด)

### `saver:` รับ option อะไรได้อีกบ้าง

`saver:` ส่งต่อ option ไปยัง encoder ของ libvips โดยตรง (`vips_jpegsave`,
`vips_webpsave`, `vips_pngsave` แล้วแต่ format) option ที่ใช้บ่อย:

```ruby
# JPEG: ควบคุม progressive scan (โหลดแบบเบลอก่อนแล้วค่อยชัดขึ้น เหมาะกับเน็ตช้า)
variant(saver: { quality: 80, interlace: true })

# PNG: ควบคุมระดับการบีบอัด (0-9, ยิ่งสูงยิ่งช้าแต่ไฟล์เล็กลง ไม่กระทบคุณภาพเพราะ PNG lossless)
variant(format: :png, saver: { compression: 9 })

# WebP: strip metadata (EXIF ฯลฯ) ออกเพื่อลดขนาดและป้องกัน metadata รั่วไหล (เช่น GPS location)
variant(format: :webp, saver: { quality: 75, strip: true })
```

> **ข้อควรระวังเรื่องความปลอดภัย/ความเป็นส่วนตัว:** รูปที่ผู้ใช้อัปโหลดจากมือถือมักมี EXIF
> metadata ติดมาด้วย ซึ่งอาจรวมพิกัด GPS ที่ถ่ายรูป การใส่ `strip: true` ใน `saver` ของ
> variant ที่จะแสดงต่อสาธารณะ (เช่นรูปโปรไฟล์, รูปสินค้า) เป็นแนวทางที่ดีเพื่อไม่ให้ข้อมูล
> ส่วนตัวของผู้ใช้รั่วไหลออกไปโดยไม่ตั้งใจ

---

## Step 667: Named variants — ลงทะเบียน variant ไว้ที่โมเดล (Rails 7+)

ปัญหาของการเขียน `product.photo.variant(resize_to_fill: [100, 100])` กระจายอยู่หลายที่ในโค้ด
(บาง view เรียก thumbnail, บาง view เรียก medium, บาง background job ก็ต้องรู้ค่า option
เดียวกันอีก) คือ **ผิดหลัก DRY อย่างชัดเจน** — ถ้าวันหนึ่งอยากเปลี่ยนขนาด thumbnail จาก
100×100 เป็น 120×120 ต้องไล่แก้ทุกที่ที่ hardcode ค่านี้ไว้

Rails 7 แก้ปัญหานี้ด้วย **named variant** — ประกาศ variant พร้อมชื่อไว้ที่โมเดลครั้งเดียว
แล้วเรียกใช้ด้วยชื่อ (symbol) ได้จากทุกที่:

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  has_one_attached :photo do |attachable|
    attachable.variant :thumb,  resize_to_fill: [ 100, 100 ], preprocessed: true
    attachable.variant :medium, resize_to_limit: [ 500, 500 ]
    attachable.variant :large,  resize_to_limit: [ 1200, 1200 ], saver: { quality: 80 }
    attachable.variant :webp,   resize_to_limit: [ 800, 800 ], format: :webp, saver: { quality: 75 }
  end
end
```

จากนั้นเรียกใช้ที่ไหนก็ได้ในโค้ดด้วยชื่อ ไม่ต้องจำ option อีกต่อไป:

```ruby
# ใน view
image_tag product.photo.variant(:thumb)
image_tag product.photo.variant(:medium)

# ใน controller/job
product.photo.variant(:large).processed.url
```

เราทดสอบ pattern นี้จริงในแอปทดลอง — attach รูปเข้ากับ `Product` ที่มี named variant
ทั้ง 4 ตัวตามด้านบน แล้วเรียกทีละตัว ผลลัพธ์ทั้งหมดประมวลผลสำเร็จ:

```
thumb key = 1lp7rkn9m68rjxsyps5jqeb8gv2f
medium blob byte_size = 48137
webp content_type = image/webp
```

### ข้อดีของ named variant นอกจาก DRY

1. **แก้ที่เดียว ใช้ได้ทั้งแอป** — เปลี่ยนขนาด `:thumb` ที่โมเดลบรรทัดเดียว ทุกจุดที่เรียก
   `variant(:thumb)` ได้ผลลัพธ์ใหม่ทันทีโดยไม่ต้องแก้โค้ดที่อื่น
2. **`preprocessed: true`** ใช้ได้เฉพาะกับ named variant เท่านั้น (ตัวเลือกนี้จะพูดถึงละเอียด
   ใน Step 669 — มันทำให้ variant นี้ถูกสร้างไว้ล่วงหน้าแบบ background โดยอัตโนมัติทันทีที่
   attach เสร็จ)
3. **อ่านง่ายขึ้นมาก** — `variant(:thumb)` สื่อความหมายชัดเจนกว่า
   `variant(resize_to_fill: [100, 100])` ที่ต้องตีความว่า "100x100 นี้คือ thumbnail หรือ
   avatar หรืออะไร"

---

## Step 668: `analyze` และ `identify` — ดึง metadata ของไฟล์

Active Storage แยกงาน "รู้จักไฟล์" ออกเป็นสองกลไกที่ทำหน้าที่ต่างกันโดยสิ้นเชิง แม้ชื่อจะ
ดูคล้ายกัน:

- **`identify`** — ตรวจสอบ **content type ที่แท้จริง** ของไฟล์ จากการอ่าน magic bytes จริง
  (ไม่ใช่เชื่อ header ที่ client ส่งมา) ใช้ gem `marcel` เบื้องหลัง
- **`analyze`** — ดึง **metadata เชิงลึกของเนื้อหา** เช่น ความกว้าง/สูงของภาพ, ความยาวของ
  วิดีโอ/เสียง โดยใช้ analyzer ที่เหมาะกับแต่ละประเภทไฟล์ (สำหรับภาพคือ
  `ActiveStorage::Analyzer::ImageAnalyzer::Vips`)

### `analyze` — ดึงขนาดภาพ

ทุกครั้งที่ `.attach` ไฟล์ Active Storage จะ enqueue `ActiveStorage::AnalyzeJob` ให้อัตโนมัติ
(ทำงานแบบ background ผ่าน Active Job) เพื่อเติมค่าลง column `metadata` ของ blob

```ruby
product.photo.attach(io: file, filename: "chair.jpg", content_type: "image/jpeg")

# ปกติ analyze จะรันผ่าน background job — ถ้าอยากบังคับรันทันที (เช่นใน console/สคริปต์):
product.photo.analyze unless product.photo.analyzed?

blob = product.photo.blob
puts blob.metadata
```

**ผลลัพธ์จริงที่เราทดสอบได้:**

```ruby
{"identified"=>true, "width"=>1600, "height"=>1200, "analyzed"=>true}
```

เรียกใช้ค่าที่ดึงมาได้สะดวกๆ ผ่าน `blob.metadata[:width]` / `blob.metadata[:height]`
(ตัวอย่างเช่น ใช้แสดง dimension ให้ผู้ใช้เห็นก่อนดาวน์โหลด หรือใช้คำนวณ aspect ratio สำหรับ
ทำ responsive `<img>` ที่กัน layout shift)

analyzer เบื้องหลังคือ `ActiveStorage::Analyzer::ImageAnalyzer::Vips` — เรียกใช้ตรงๆ ได้
เหมือนกัน (มีประโยชน์ตอน debug ว่าทำไม metadata ไม่ตรงที่คาด):

```ruby
ActiveStorage::Analyzer::ImageAnalyzer::Vips.new(blob).metadata
# => {:width=>1600, :height=>1200}
```

### `identify` — ตรวจ content type จากไบต์จริง ไม่เชื่อ client

นี่คือจุดที่สำคัญมากด้าน**ความปลอดภัย**: `content_type` ที่ browser ส่งมาตอนอัปโหลด (จาก
HTTP header `Content-Type`) เป็นค่าที่ **client เป็นคนกำหนดเอง ปลอมแปลงได้ง่ายมาก** — ถ้าเรา
เชื่อค่านี้ตรงๆ โดยไม่ตรวจสอบ อาจโดนหลอกให้ระบบเข้าใจผิดว่าไฟล์อันตราย (เช่น executable หรือ
ไฟล์ script) เป็นไฟล์รูปภาพที่ปลอดภัย

Active Storage แก้ปัญหานี้ด้วย `identify` ที่อ่าน **magic bytes** ตัวจริงของไฟล์ (ไม่กี่กิโล
ไบต์แรกของไฟล์ที่มี signature เฉพาะของแต่ละ format) ผ่าน gem `marcel` แล้วเขียนทับ
`content_type` ให้ตรงกับที่ตรวจพบจริง

เราทดสอบ scenario "หลอก" นี้จริง — อัปโหลดไฟล์ JPEG ตัวเดิม แต่ **ประกาศ** ว่าเป็น
`text/plain` ชื่อไฟล์ `sneaky.txt`:

```ruby
blob = ActiveStorage::Blob.create_and_upload!(
  io: File.open("test_photo.jpg"),
  filename: "sneaky.txt",
  content_type: "text/plain"      # โกหกว่าเป็นไฟล์ text
)

puts blob.content_type
```

**ผลลัพธ์จริงที่ได้:**

```
client ประกาศ content_type: text/plain, filename: sneaky.txt
แต่ Active Storage identify (Marcel ตรวจ magic bytes จริง) ได้ content_type = image/jpeg
identified? = true
```

Active Storage **ไม่เชื่อ** ค่าที่ client ส่งมาเลย มันอ่านไบต์จริงแล้วสรุปเองว่านี่คือ
`image/jpeg` ทั้งที่ HTTP header บอกว่า `text/plain` และชื่อไฟล์ลงท้าย `.txt` — นี่คือเหตุผล
ที่การเช็ค `content_type` ของ Active Storage เชื่อถือได้มากกว่าการเช็คนามสกุลไฟล์หรือ
`Content-Type` header เพียงอย่างเดียว (ซึ่งเป็นเทคนิคโจมตีพื้นฐานที่คนร้ายมักลองก่อนเสมอ
เวลาระบบมี upload endpoint)

> **ข้อควรจำ:** `identify` รันแค่ครั้งเดียวต่อ blob (มี flag `identified` กันไม่ให้รันซ้ำ)
> เกิดขึ้นอัตโนมัติตอน `create_and_upload!`/`create_before_direct_upload!` จึงไม่ต้องเรียกเอง
> ด้วยมือในโค้ดปกติ — ที่ต้องรู้จักไว้คือกลไกนี้มีอยู่จริงและทำงานยังไง เผื่อวันหนึ่งต้อง debug
> เคสที่ content_type ไม่ตรงกับที่คาด

---

## Step 669: Background variant processing — อย่าให้ผู้ใช้คนแรกต้องรอ

### พฤติกรรม default: ประมวลผลแบบ synchronous ตอนถูกเรียกครั้งแรก

`variant.processed` ทำงานแบบ **lazy** — มันจะตรวจสอบก่อนว่า variant นี้เคยถูกสร้างไว้แล้ว
หรือยัง (เช็คจาก `ActiveStorage::VariantRecord` ที่ผูกกับ "variation digest" ซึ่งเป็นค่า hash
ของ transformation options) ถ้ายังไม่เคยมี **มันจะประมวลผลทันทีตรงนั้นแบบ synchronous**
(บล็อก request/thread ปัจจุบันไว้จนกว่าจะเสร็จ) แล้วค่อย return URL ของไฟล์ที่ประมวลผลเสร็จ

ปัญหาคือ ถ้า variant นี้ไม่เคยถูกสร้างมาก่อนเลย (เช่น สินค้าที่เพิ่งอัปโหลดรูปมาสดๆ) **ผู้ใช้
คนแรกที่เปิดหน้าเว็บนั้น** จะเป็นคนที่ต้องรอ Rails ประมวลผลภาพ (ดาวน์โหลดต้นฉบับ, resize ด้วย
libvips, อัปโหลด variant กลับไป storage) ทั้งหมดนี้เกิดขึ้น**ระหว่าง** request ของเขา ทำให้
หน้าเว็บโหลดช้าผิดปกติสำหรับคนแรกเท่านั้น (คนถัดไปจะเร็วเพราะ variant ถูก cache ไว้แล้ว)

### วิธีแก้ที่ 1: `preprocessed: true` (built-in, ง่ายที่สุด)

Named variant ที่ประกาศ `preprocessed: true` (ที่เราตั้งไว้กับ `:thumb` ใน Step 667) จะถูก
Active Storage **enqueue job สร้าง variant ให้อัตโนมัติทันทีหลัง attach เสร็จ** โดยไม่ต้อง
เขียนโค้ดเพิ่มเลยสักบรรทัด

เราทดสอบพฤติกรรมนี้จริงด้วย `ActiveJob::Base.queue_adapter = :test` (เพื่อดูว่า job อะไรถูก
enqueue โดยไม่ต้องรันจริง):

```ruby
ActiveJob::Base.queue_adapter = :test
product = Product.create!(name: "โต๊ะทำงาน", price: 3990)

product.photo.attach(io: File.open("test_photo.jpg"), filename: "desk.jpg", content_type: "image/jpeg")

ActiveJob::Base.queue_adapter.enqueued_jobs.each { |j| puts "#{j[:job]} #{j[:args]}" }
```

**ผลลัพธ์จริง — มี 2 job ถูก enqueue อัตโนมัติทันทีหลัง `.attach`:**

```
ActiveStorage::AnalyzeJob args=[{"_aj_globalid"=>"gid://p067app/ActiveStorage::Blob/9"}]
ActiveStorage::TransformJob args=[{"_aj_globalid"=>"...Blob/9"}, {"resize_to_fill"=>[100, 100], ...}]
```

`ActiveStorage::AnalyzeJob` ดึง metadata (Step 668) และ `ActiveStorage::TransformJob` คือตัว
ที่ประมวลผล named variant `:thumb` ล่วงหน้าให้เลย (สังเกตว่า args มี
`"resize_to_fill"=>[100, 100]` ตรงกับที่เราประกาศไว้) — ทั้งสอง job รันผ่าน Active Job queue
ปกติของแอป (Sidekiq ใน production ตามที่สอนไว้ใน Part 062) เมื่อผู้ใช้คนแรกเปิดหน้าสินค้านี้
`:thumb` ก็ถูกสร้างเสร็จรอไว้แล้ว ไม่ต้องรอ process สด

### วิธีแก้ที่ 2: เขียน custom job เอง (ควบคุมได้ละเอียดกว่า)

`preprocessed: true` เหมาะกับ variant ที่ต้องการแน่ๆ ทุกครั้ง แต่ถ้าอยากควบคุมเงื่อนไข
มากกว่านั้น (เช่น สร้างเฉพาะตอน record ผ่าน validation แล้ว, หรือสร้างหลาย variant พร้อมกัน
ใน job เดียวเพื่อลด overhead ของการ enqueue หลายรอบ) เขียน job เองได้ตรงไปตรงมา:

```ruby
# app/jobs/pre_generate_product_variants_job.rb
class PreGenerateProductVariantsJob < ApplicationJob
  queue_as :default

  def perform(product)
    return unless product.photo.attached?

    # เรียก .processed ตรงๆ ใน background job — ทำให้ variant ถูกสร้างและ cache ไว้
    # ก่อนที่ผู้ใช้คนแรกจะมาเจอ
    %i[thumb medium large webp].each do |variant_name|
      product.photo.variant(variant_name).processed
    end
  end
end
```

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  has_one_attached :photo do |attachable|
    attachable.variant :thumb,  resize_to_fill: [ 100, 100 ]
    attachable.variant :medium, resize_to_limit: [ 500, 500 ]
    attachable.variant :large,  resize_to_limit: [ 1200, 1200 ], saver: { quality: 80 }
    attachable.variant :webp,   resize_to_limit: [ 800, 800 ], format: :webp, saver: { quality: 75 }
  end

  after_create_commit :pre_generate_variants, if: -> { photo.attached? }

  private

  def pre_generate_variants
    PreGenerateProductVariantsJob.perform_later(self)
  end
end
```

### `.processed` เรียกซ้ำได้อย่างปลอดภัย (idempotent)

จุดที่น่ากังวลคือ: ถ้า `preprocessed: true` สร้าง `:thumb` ไปแล้ว แล้ว custom job เรียก
`.variant(:thumb).processed` ซ้ำอีกที จะเสียเวลาประมวลผลซ้ำสองรอบไหม? เราทดสอบเรื่องนี้
โดยตรง:

```ruby
before = ActiveStorage::VariantRecord.count
v1 = product.photo.variant(:medium).processed
after_first = ActiveStorage::VariantRecord.count
v2 = product.photo.variant(:medium).processed
after_second = ActiveStorage::VariantRecord.count

puts "before=#{before} after_first=#{after_first} after_second=#{after_second}"
puts "key เดิมหรือไม่: #{v1.key == v2.key}"
```

**ผลลัพธ์จริง:** จำนวน `VariantRecord` ไม่เพิ่มขึ้นระหว่างการเรียกครั้งที่สอง และ `v1.key`
กับ `v2.key` เป็นค่าเดียวกัน — ยืนยันว่า `.processed` **idempotent จริง** Active Storage
เช็คจาก variation digest ก่อนเสมอ ถ้ามี record ที่ตรงกับ digest นี้อยู่แล้วมันจะคืนตัวเดิม
ทันทีโดยไม่ประมวลผลซ้ำ ปลอดภัยที่จะเรียกจากหลายที่โดยไม่ต้องกังวลเรื่อง duplicate work

---

## Step 670: CDN, proxy vs redirect mode, และ cost/security ก่อนขึ้น production

### `rails_storage_redirect_path` vs `rails_storage_proxy_path`

Active Storage มี 2 โหมดหลักในการ serve ไฟล์ให้ browser:

1. **Redirect mode (default)** — Rails ตอบกลับด้วย HTTP 302 พร้อม header `Location` ชี้ไปที่
   signed URL ของ storage service (S3/MinIO) โดยตรง browser ต้องยิง request รอบสองไปที่ URL
   นั้นเองถึงจะได้ไฟล์จริง
2. **Proxy mode** — Rails **ดาวน์โหลดไฟล์จาก storage service มาเอง แล้ว stream ต่อให้ browser
   ทันที** ในการตอบครั้งเดียว (HTTP 200 พร้อมเนื้อไฟล์เลย ไม่มี redirect)

เราทดสอบทั้งสองโหมดจริงด้วยการรัน Rails server จริงแล้วยิง HTTP request เข้าไปที่ทั้งสอง
route:

```bash
curl -D - -o /dev/null "http://127.0.0.1:3000/rails/active_storage/blobs/redirect/<signed_id>/chair.jpg"
```

```
HTTP/1.1 302 Found
location: http://127.0.0.1:3000/rails/active_storage/disk/<token>/chair.jpg
content-type: text/html; charset=utf-8
```

```bash
curl -D - -o output.jpg "http://127.0.0.1:3000/rails/active_storage/blobs/proxy/<signed_id>/chair.jpg"
```

```
HTTP/1.1 200 OK
content-type: image/jpeg
cache-control: max-age=3155695200, public, immutable
```

(`output.jpg` ที่ได้ตรวจสอบด้วยคำสั่ง `file` แล้วเป็นไฟล์ JPEG จริงขนาด 1,202,082 ไบต์ ตรงกับ
ต้นฉบับเป๊ะ — proxy mode ส่งไบต์จริงกลับมาในการตอบครั้งเดียว)

**สังเกต `Cache-Control` ของ proxy mode:** `max-age=3155695200` (ประมาณ 100 ปี) พร้อม
`public, immutable` — นี่คือกุญแจสำคัญที่ทำให้ proxy mode เข้ากับ CDN ได้ดีกว่า redirect mode
มาก เพราะ:

- **Redirect mode**: ถ้า CDN cache การตอบ 302 ไว้ (cache "Location" header) แล้ว signed URL
  ข้างในหมดอายุไปแล้ว (ตาม `X-Amz-Expires` ที่เราเห็นใน Step 665 คือ 5 นาที) ผู้ใช้ที่ได้รับ
  cached 302 นั้นจะโดน redirect ไปยัง URL ที่ตายไปแล้ว เกิด error ฝั่ง S3 แทน — CDN จึงมักถูก
  ตั้งให้**ไม่ cache** การตอบแบบ redirect ของ Active Storage เลย ทำให้ทุก request ยังต้องยิง
  มาที่ Rails server ก่อนเสมอ (แค่ redirect ไป S3 แต่ตัวการตัดสินใจ "จะ redirect ไปไหน" ยัง
  เกิดที่ Rails ทุกครั้ง)
- **Proxy mode**: ไฟล์จริง (ไม่ใช่ token ที่หมดอายุ) ถูกส่งกลับพร้อม cache header ที่บอกว่า
  "cache ได้นานเป็นปีๆ" — CDN สามารถ cache **เนื้อไฟล์จริง** ไว้ที่ edge ได้เต็มที่ ทำให้
  request ครั้งต่อๆ ไปไม่ต้องย้อนกลับมาที่ Rails server หรือแม้แต่ S3 เลย (โดนตอบจาก edge
  ของ CDN ตรงๆ)

เปิดใช้ proxy mode ทั้งแอปได้ด้วย config เดียว:

```ruby
# config/application.rb หรือ config/environments/production.rb
config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

เราตรวจสอบแล้วว่า default ของค่านี้คือ `:rails_storage_redirect` และเปลี่ยนเป็น
`:rails_storage_proxy` ได้จริง — ทดสอบด้วย `url_for(product.photo)` ก่อนและหลังตั้งค่า
พบว่า URL ที่ได้เปลี่ยนจาก `/rails/active_storage/blobs/redirect/...` เป็น
`/rails/active_storage/blobs/proxy/...` ตรงตามที่คาด (มีผลกับทุกที่ที่เรียก `url_for` หรือ
`image_tag` กับ attachment โดยไม่ระบุ helper เฉพาะเจาะจง)

**Trade-off ที่ต้องรู้:** proxy mode ทำให้ **ทุก byte ของไฟล์วิ่งผ่าน Rails server** (ต่างจาก
redirect mode ที่ไฟล์จริงวิ่งจาก S3 ตรงไปหา browser เลย) ถ้าไม่มี CDN อยู่หน้า Rails server
proxy mode จะเพิ่มภาระ bandwidth/CPU ให้ Rails server โดยตรง — proxy mode จึงคุ้มค่าที่สุด
เมื่อใช้**ร่วมกับ CDN** (CDN cache ผลลัพธ์ของ proxy ไว้ ทำให้ Rails ต้อง proxy จริงๆ แค่ครั้ง
แรกที่ไฟล์นั้นถูกขอ หลังจากนั้น edge ของ CDN ตอบเองหมด)

### วาง CDN หน้า S3 โดยตรง (อีกวิธีหนึ่ง ไม่ผ่าน Rails proxy)

อีกแนวทางที่นิยมพอกันคือตั้ง CDN (CloudFront, Cloudflare CDN, Fastly) ให้ชี้ตรงไปที่ bucket
เลย โดยให้ bucket เป็น**แบบ read-only public** เฉพาะสำหรับ asset ที่ตั้งใจให้สาธารณะเข้าถึง
ได้จริง (เช่น รูปสินค้าในร้านค้าออนไลน์ที่ใครก็ควรดูได้อยู่แล้ว) วิธีนี้ตัด Rails ออกจาก
เส้นทางการโหลดไฟล์ไปเลยหลังจาก cache ครั้งแรก เร็วที่สุดแต่ต้องแลกกับการที่ไฟล์นั้น**ไม่มี
access control ผ่าน Rails อีกต่อไป** (เหมาะกับไฟล์สาธารณะเท่านั้น ไม่เหมาะกับไฟล์ที่ต้องเช็ค
สิทธิ์ผู้ใช้ก่อนดู เช่น ใบเสร็จ/เอกสารส่วนตัว ซึ่งควรใช้ signed URL ที่หมดอายุแทน)

### Cost & security checklist ก่อนขึ้น production

1. **อย่าเปิด bucket เป็น public แบบทั้ง bucket โดยไม่ตั้งใจ** — เราทดสอบแล้วว่า MinIO
   (พฤติกรรมเดียวกับ S3) ปฏิเสธ unsigned request ด้วย `403 Forbidden` โดย default:

   ```bash
   curl -o /dev/null -w "%{http_code}" "http://localhost:9000/dev-uploads/<key-ไม่มี-signature>"
   # => 403
   ```

   ในขณะที่ signed URL (จาก `blob.url`) ที่มี `X-Amz-Signature` และ `X-Amz-Expires=300`
   ติดมาด้วย ให้ผลลัพธ์ `200` ตามปกติ — นี่คือพฤติกรรม **private-by-default** ที่ถูกต้อง
   การเปิด bucket policy ให้ public read ทั้ง bucket (`"Principal": "*"`) ควรทำเฉพาะกรณีที่
   ตั้งใจจริงๆ ว่าทุกไฟล์ในนั้นเป็นสาธารณะทั้งหมด (เช่น bucket แยกเฉพาะ static asset) และ
   **ไม่ควรใช้ bucket เดียวกันเก็บทั้งไฟล์สาธารณะและไฟล์ส่วนตัวปนกัน**
2. **Signed URL หมดอายุจริง และควรตั้งเวลาให้เหมาะกับ use case** — ค่า default
   `ActiveStorage.service_urls_expire_in` คือ 300 วินาที (5 นาที) ซึ่งสั้นพอสำหรับ
   direct upload (ผู้ใช้กำลัง active อยู่หน้าจอ) แต่ถ้าเป็น URL สำหรับดาวน์โหลดไฟล์
   ที่อาจถูกแชร์ผ่าน email/ลิงก์ (เช่นใบแจ้งหนี้) ควรปรับให้เหมาะสมผ่าน
   `expires_in:` ตอนเรียก `url`:

   ```ruby
   invoice.pdf.url(expires_in: 1.hour)
   ```

3. **จำกัดสิทธิ์ IAM ให้แคบที่สุดเท่าที่จำเป็น** (recap จาก Step 662) — แยก bucket ต่อ
   environment (`myapp-production-uploads` vs `myapp-staging-uploads`) และแยก IAM
   credential ต่อ environment ด้วย ไม่ใช้ key ชุดเดียวกันทั้ง dev/staging/production
4. **เปิด versioning และ lifecycle policy บน bucket จริง** — versioning ป้องกันการลบไฟล์
   ผิดพลาดแบบกู้คืนไม่ได้ ส่วน lifecycle policy (เช่น ย้ายไฟล์ที่ไม่ถูกเข้าถึงนาน 90 วันไป
   storage class ราคาถูกกว่าอย่าง S3 Glacier) ช่วยคุมค่าใช้จ่ายระยะยาวโดยไม่ต้องเขียนโค้ด
   จัดการเอง
5. **ระวังค่า egress** — นี่คือค่าใช้จ่ายที่ทีมมักมองข้ามตอนเริ่มโปรเจกต์แต่กลายเป็นก้อนใหญ่
   ตอน traffic เยอะขึ้น AWS S3 คิดค่าส่งข้อมูลออกจาก bucket (ต่างจากค่า storage ที่ถูกมาก)
   ถ้าแอป serve รูป/วิดีโอปริมาณมาก ควรพิจารณาวาง CDN หน้า S3 เสมอ (ลด origin request ซึ่ง
   คือจุดที่เกิดค่า egress) หรือพิจารณา Cloudflare R2 ที่ไม่คิดค่า egress เลยตามที่กล่าวไว้
   ใน Step 663
6. **Validate content type และขนาดไฟล์ฝั่ง Rails ด้วย ไม่ใช่พึ่ง S3 อย่างเดียว** — ใช้ผลจาก
   `identify` (Step 668) ประกอบกับ validation บนโมเดล (`content_type: { in: %w[image/jpeg
   image/png image/webp] }`, จำกัด `byte_size` สูงสุด) ก่อนอนุญาตให้ attach จริง ไม่ควร
   เชื่อว่า "S3 เก็บได้ก็ปลอดภัยแล้ว" — S3 เป็นแค่ที่เก็บ ไม่ได้ตรวจสอบเนื้อหาให้เรา

---

## แบบฝึกหัด: ตั้งค่าอัปโหลดรูปสินค้าให้ทำงานได้ทั้ง local dev และ S3-compatible service

### โจทย์

ทำให้ `Product` (จาก Part 066) อัปโหลดรูปได้แบบ:

1. ใช้ local disk service ตอน development และ test (ไม่ต้องพึ่งเน็ต)
2. ใช้ S3-compatible service (MinIO หรือ S3 จริง) ตอน production ผ่าน environment config
   เดียว ไม่ต้องแก้โค้ดโมเดล/controller
3. มี named variant 3 แบบ: `:thumb` (100×100, ครอบตัดพอดี, preprocess ไว้ล่วงหน้า),
   `:medium` (ไม่เกิน 500×500), `:large` (ไม่เกิน 1200×1200 คุณภาพ 80)
4. ตรวจสอบ content type และขนาดไฟล์ก่อนอนุญาตให้อัปโหลด

### เฉลย

**`config/storage.yml`:**

```yaml
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: ap-southeast-1
  bucket: myapp-production-uploads

minio:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:minio, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:minio, :secret_access_key) %>
  region: us-east-1
  bucket: dev-uploads
  endpoint: <%= ENV.fetch("MINIO_ENDPOINT", "http://localhost:9000") %>
  force_path_style: true
```

**`config/environments/development.rb`:**

```ruby
Rails.application.configure do
  # ...
  config.active_storage.service = :local
end
```

**`config/environments/test.rb`:**

```ruby
Rails.application.configure do
  # ...
  config.active_storage.service = :test
end
```

**`config/environments/production.rb`:**

```ruby
Rails.application.configure do
  # ...
  config.active_storage.service = :amazon
  config.active_storage.resolve_model_to_route = :rails_storage_proxy   # เผื่อวาง CDN หน้า Rails
end
```

**`app/models/product.rb`:**

```ruby
class Product < ApplicationRecord
  has_one_attached :photo do |attachable|
    attachable.variant :thumb,  resize_to_fill: [ 100, 100 ], preprocessed: true
    attachable.variant :medium, resize_to_limit: [ 500, 500 ]
    attachable.variant :large,  resize_to_limit: [ 1200, 1200 ], saver: { quality: 80 }
  end

  validates :photo,
            content_type: %w[image/jpeg image/png image/webp],
            size: { less_than: 10.megabytes },
            if: -> { photo.attached? }
end
```

> **หมายเหตุ:** validation `content_type`/`size` ด้านบนใช้ syntax ของ gem
> `active_storage_validations` (นิยมมากในวงการ เพราะ Active Storage ไม่มี validator ในตัว
> ให้ตรงๆ) เพิ่มใน `Gemfile`:
> ```ruby
> gem "active_storage_validations"
> ```

**`app/controllers/products_controller.rb`** (ส่วนที่เกี่ยวกับ photo):

```ruby
class ProductsController < ApplicationController
  def create
    @product = Product.new(product_params)
    if @product.save
      redirect_to @product, notice: "เพิ่มสินค้าเรียบร้อย"
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

**`app/views/products/_product.html.erb`** (แสดง thumbnail ในหน้ารายการ):

```erb
<div class="product-card">
  <% if product.photo.attached? %>
    <%= image_tag product.photo.variant(:thumb), alt: product.name %>
  <% else %>
    <div class="product-card__placeholder">ไม่มีรูป</div>
  <% end %>
  <h3><%= product.name %></h3>
  <p><%= number_to_currency(product.price, unit: "฿") %></p>
</div>
```

**`app/views/products/show.html.erb`** (แสดงรูปขนาดใหญ่ในหน้ารายละเอียด):

```erb
<% if @product.photo.attached? %>
  <%= image_tag @product.photo.variant(:large),
                alt: @product.name,
                loading: "lazy" %>
<% end %>
```

### สิ่งที่ตรวจสอบแล้วในโจทย์นี้

โครงสร้าง config ด้านบน (`storage.yml` + environment files + named variant DSL) เป็น
รูปแบบเดียวกันทุกประการกับที่เราทดสอบจริงตลอด Part นี้:

- Named variant `:thumb`/`:medium`/`:large` — ทดสอบจริงว่า `.processed` ทำงานถูกต้อง คืน
  byte size ที่สมเหตุสมผลตามขนาดที่กำหนด
- `preprocessed: true` บน `:thumb` — ทดสอบจริงว่า enqueue `ActiveStorage::TransformJob`
  อัตโนมัติหลัง attach
- การสลับ service ระหว่าง `:local` และ `:minio` (จำลอง `:amazon`) — ทดสอบจริงว่าโค้ดโมเดล/
  view เดิมทำงานถูกต้องโดยไม่ต้องแก้แม้แต่บรรทัดเดียว ไม่ว่า backend จะเป็น local disk หรือ
  S3-compatible service

ส่วนที่ **ไม่ได้ทดสอบจริง** ในแบบฝึกหัดนี้คือการรันกับ AWS S3 จริง (ต้องมีบัญชี AWS จริง)
และ gem `active_storage_validations` (ไม่ได้ติดตั้งในแอปทดลอง เพราะโฟกัสของ Part นี้อยู่ที่
กลไก cloud storage/image processing ของ Active Storage เอง ไม่ใช่ gem เสริมภายนอก) — แต่
API ของ gem นี้เป็นมาตรฐานที่ใช้กันแพร่หลายและมีเอกสารชัดเจน

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม named variant `:webp` (ไม่เกิน 800×800, format `:webp`, quality 75) ให้กับ
   `Product` แล้วเขียน helper method ใน view ที่เลือกส่ง `:webp` ให้ browser ที่รองรับ
   (ตรวจจาก `Accept` header) และ fallback เป็น `:medium` (JPEG) ให้ browser ที่ไม่รองรับ
   (ใบ้: ใช้ `<picture>` tag กับ `<source type="image/webp">`)
2. เขียน background job (`PreGenerateProductVariantsJob` ตามตัวอย่างใน Step 669) ที่สร้าง
   ทุก named variant ล่วงหน้าทันทีที่ `Product` ถูกสร้างพร้อมรูป แล้วเขียนเทสต์ (RSpec หรือ
   Minitest ตามที่เรียนมาใน Part 046-050) ยืนยันว่า job ถูก enqueue จริงหลัง `create`
3. ทดลองตั้งค่า `config.active_storage.service = :mirror` ที่ใช้
   `ActiveStorage::Service::MirrorService` (บริการพิเศษที่เขียนไฟล์ลงหลาย service พร้อมกัน
   เช่น เก็บทั้งบน local disk และ S3 ไปพร้อมกัน) — ค้นเอกสาร Active Storage เรื่อง "Mirror
   Service" แล้วลองตั้งค่าให้ใช้งานได้จริงระหว่าง `:local` กับ `:minio`
4. (ท้าทายขึ้น) เขียนสคริปต์ Rake task ที่ย้ายไฟล์ทั้งหมดจาก service `:local` ไป `:amazon`
   (หรือ `:minio` สำหรับทดสอบ) สำหรับข้อมูลที่มีอยู่แล้วในระบบ โดยไม่ทำให้ URL ที่เคยแชร์ออก
   ไปแล้วเสีย (ใบ้: `ActiveStorage::Blob#key` ไม่เปลี่ยนไม่ว่าจะย้ายไป service ไหน ตราบใดที่
   คัดลอกไฟล์ไปเก็บด้วย key เดิม)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่าทำไม local disk storage ใช้กับ production จริงไม่ได้ — ปัญหา ephemeral
  filesystem และการมีหลาย server instance ที่ disk แยกกันคนละก้อน
- ตั้งค่า service `:amazon` ใน `config/storage.yml` ได้ พร้อมดึง credential อย่างปลอดภัย
  ผ่าน Rails encrypted credentials (`rails credentials:edit`)
- เข้าใจว่า `service: S3` ใน Active Storage ใช้ได้กับทุกบริการที่พูด S3 API ได้ (MinIO,
  Cloudflare R2, DigitalOcean Spaces) ต่างกันแค่ `endpoint`/`force_path_style` — และได้
  ทดสอบ flow นี้จริงกับ MinIO server ที่รันขึ้นมาเอง
- สลับ service ตาม environment ได้ (`:local` dev, `:amazon` production) โดยไม่ต้องแก้โค้ด
  โมเดล/controller เลย
- เข้าใจกลไก direct upload ไป S3 อย่างละเอียด — presigned URL ที่ browser ใช้อัปโหลดตรงไป
  storage service โดยไม่ผ่าน Rails server เลย พร้อมพิสูจน์ด้วยการยิง `PUT` จริงไปยัง
  presigned URL จาก MinIO แล้วได้ `HTTP 200`
- เจาะลึก image processing ด้วย libvips — `resize_to_fill`/`resize_to_limit`/
  `resize_to_fit` ต่างกันอย่างไร, การแปลง format, การควบคุม quality/compression ผ่าน
  `saver:`, และเหตุผลเชิงสถาปัตยกรรมที่ libvips ประหยัดหน่วยความจำกว่า ImageMagick มาก
- ใช้ named variant (Rails 7+) ลงทะเบียน variant ไว้ที่โมเดลเพื่อความ DRY และเปิดใช้
  `preprocessed: true` ได้
- เข้าใจความต่างของ `analyze` (ดึง metadata เช่นขนาดภาพ) กับ `identify` (ตรวจ content type
  จริงจาก magic bytes ไม่เชื่อ client) พร้อมพิสูจน์ด้วยการหลอกส่งไฟล์ JPEG เป็น `text/plain`
  แล้วดูว่า Active Storage จับได้
- ตั้งค่า background variant processing เพื่อไม่ให้ผู้ใช้คนแรกต้องรอ process รูปสด และ
  เข้าใจว่า `.processed` idempotent ปลอดภัยที่จะเรียกซ้ำ
- เข้าใจ redirect mode vs proxy mode ของ Active Storage และผลกระทบต่อการวาง CDN พร้อม
  checklist ด้าน cost/security ก่อนขึ้น production จริง (bucket ไม่ public โดยไม่ตั้งใจ,
  signed URL หมดอายุ, สิทธิ์ IAM แคบที่สุดเท่าที่จำเป็น)

**ต่อไป (Part 068):** เราจะเปลี่ยนโฟกัสจากการอัปโหลดไฟล์ไปที่การ**ค้นหาข้อมูล** — เรียนรู้
`pg_search` gem สำหรับทำ full-text search บน PostgreSQL โดยตรง (ไม่ต้องพึ่ง external search
engine) ครอบคลุมการค้นหาข้ามหลายคอลัมน์/หลายโมเดล, การจัดอันดับผลลัพธ์ตามความเกี่ยวข้อง
(relevance ranking), และการรองรับภาษาไทยในการค้นหา
