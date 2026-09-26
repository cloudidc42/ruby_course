# Part 069: Elasticsearch/OpenSearch Integration เบื้องต้น — เมื่อ pg_search ไม่พอ

> **Step ครอบคลุมใน Part นี้:** Step 681–690
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 068 เรื่อง Full-text search ด้วย pg_search, Part 061
> เรื่อง ActiveJob เบื้องต้น, และควรผ่าน Part 025 เรื่อง ActiveRecord มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4,
> `opensearch-ruby` 3.4.0, OpenSearch 2.19.6 และ Elasticsearch 8.15.0)

> **หมายเหตุเรื่องเครื่องมือ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> Part นี้พูดถึงซอฟต์แวร์ที่ "หนัก" กว่า PostgreSQL/Redis มาก (ต้องมี JVM, ใช้ RAM หลักร้อย MB
> ถึงหลาย GB) ทีมผู้เขียนหลักสูตรจึงพยายามรันของจริงในแซนด์บ็อกซ์ที่ใช้เขียน Part นี้ให้มาก
> ที่สุดเท่าที่จะทำได้ แทนที่จะเดาเอาจากเอกสาร ผลที่ได้คือ:
>
> - **รันสำเร็จจริง:** ทั้ง **OpenSearch 2.19.6** และ **Elasticsearch 8.15.0** ถูกดึงมาเป็น
>   Docker image จริงจาก Docker Hub (`opensearchproject/opensearch:2` และ `elasticsearch:8.15.0`)
>   และรันเป็น container จริงในแซนด์บ็อกซ์ ยืนยันด้วย `_cluster/health` ที่ตอบ `"status":"green"`
>   จริง ทุกคำสั่ง Ruby/curl ที่แสดงในบทความนี้ที่ระบุว่า "ทดสอบจริง" ("verified live") คือรันจริง
>   กับเซิร์ฟเวอร์ทั้งสองตัวนี้ ไม่ใช่โค้ดที่เขียนขึ้นลอยๆ
> - **ดาวน์โหลดไม่สำเร็จ (ติด network policy):** การดาวน์โหลด tarball ตรงจาก
>   `artifacts.elastic.co` และ `artifacts.opensearch.org` (วิธีติดตั้งแบบไม่ใช้ Docker) ถูก
>   ปฏิเสธด้วย `403 Forbidden` จาก network policy ของแซนด์บ็อกซ์อย่างชัดเจน (ไม่ใช่ปัญหาเน็ตหลุด)
>   เช่นเดียวกับ `docker.elastic.co` (registry ทางการของ Elastic) แต่ image เดียวกันที่ mirror
>   ไว้บน **Docker Hub** (`docker.io`) กลับดึงได้ปกติ — บทความนี้จึงใช้เส้นทาง Docker Hub
> - **ลองแล้วไม่สำเร็จ:** การติดตั้ง plugin `analysis-icu` เพิ่มเข้าไปใน container ที่รันอยู่
>   (`opensearch-plugin install analysis-icu`) ล้มเหลวเพราะ container ไม่มี CA certificate ของ
>   proxy ที่แซนด์บ็อกซ์ใช้ (TLS certificate validation error) ส่วนที่พูดถึง ICU plugin ใน
>   Step 686 จึงมาจากเอกสารทางการ ไม่ได้ทดสอบจริง — จะระบุไว้ชัดเจนตรงจุดนั้น
> - ทุกอย่างที่เกี่ยวกับ **Thai analyzer ในตัว (built-in)**, **relevance scoring**,
>   **aggregation/facet**, **fuzzy search**, **highlighting**, และ **การเชื่อมต่อจาก Ruby**
>   (ทั้ง gem `opensearch-ruby` และ gem `elasticsearch` อย่างเป็นทางการ) **ทดสอบจริงทั้งหมด**
>   รวมถึงแอป Rails สาธิตที่สร้างขึ้นใน scratch directory แยกจาก repository ของหลักสูตรโดย
>   สิ้นเชิง (ลบทิ้งหลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือใน repo)

## สารบัญของ Part นี้

- Step 681: Elasticsearch/OpenSearch คืออะไร — ทำไม pg_search อย่างเดียวไม่พอ
- Step 682: Elasticsearch vs OpenSearch — ประวัติการแยกโปรเจกต์ปี 2021 และความเข้ากันได้
- Step 683: รันอินสแตนซ์ local ด้วย Docker — คำสั่งจริง ปัญหาจริงที่เจอ และวิธีแก้
- Step 684: เชื่อมต่อจาก Ruby/Rails — เลือก gem ให้ถูกกับเอนจิน (`elasticsearch-model`/
  `elasticsearch-rails` vs `opensearch-ruby`)
- Step 685: กำหนด Index Mapping สำหรับ Model — field type, analyzer, และทำไม category
  ต้องเป็น `keyword` ไม่ใช่ `text`
- Step 686: Thai-language Analyzer — วิธีแก้ปัญหาการตัดคำภาษาไทยที่ Part 068 เจอ
- Step 687: Indexing ข้อมูลจาก Model — `Post.import` และ callback แบบ `after_commit`
- Step 688: การค้นหา (Search) พร้อม Relevance Scoring, Fuzzy Matching, Highlighting
- Step 689: Faceted Search / Aggregations — นับจำนวนโพสต์ต่อหมวดหมู่พร้อมผลค้นหา
- Step 690: การ sync Search Index กับฐานข้อมูล — Index Drift, กลยุทธ์ reindex, Background
  Job, และต้นทุนด้านปฏิบัติการที่ต้องคิดก่อนตัดสินใจใช้

---

## Step 681: Elasticsearch/OpenSearch คืออะไร — ทำไม pg_search อย่างเดียวไม่พอ

### ทบทวนสั้นๆ จาก Part 068

ใน Part 068 เราใช้ **pg_search** ทำ full-text search ภายใน PostgreSQL โดยอาศัย
`tsvector`/`tsquery` ของ PostgreSQL เอง ข้อดีคือไม่ต้องมีระบบเพิ่มเติม ข้อมูลอยู่ที่เดียวกับ
ฐานข้อมูลหลัก แต่ Part 068 ก็ระบุตรงไปตรงมาไว้ว่า **PostgreSQL text search config มาตรฐาน
ไม่มีตัวตัดคำภาษาไทยที่ดี** — ภาษาไทยไม่มีช่องว่างคั่นระหว่างคำ (unlike English ที่ตัดคำด้วย
whitespace ได้ง่ายๆ) การ config แบบ `simple`/`pg_catalog.english` ที่ PostgreSQL มีให้จึงมองข้อความ
ภาษาไทยทั้งประโยคเป็น "ก้อนเดียว" หรือตัดคำผิดตำแหน่ง ทำให้ full-text search กับภาษาไทยได้ผล
ลัพธ์ไม่แม่นยำเท่าภาษาอังกฤษ

**Part นี้ (069) ไม่ได้มาสอน pg_search ซ้ำ** แต่มาแนะนำเครื่องมือที่ถูกออกแบบมาเพื่อแก้ปัญหานี้
โดยเฉพาะ (รวมถึงปัญหาอื่นๆ ที่ pg_search แก้ไม่ได้เลย) นั่นคือ **Elasticsearch** และ
**OpenSearch**

### Elasticsearch/OpenSearch คืออะไร — มันไม่ใช่ฐานข้อมูลเชิงสัมพันธ์

สิ่งแรกที่ต้องทำความเข้าใจให้ชัด: **Elasticsearch และ OpenSearch ไม่ใช่ relational database**
มันคือ **distributed search engine** ที่ออกแบบมาเพื่อค้นหาข้อความจำนวนมหาศาลให้เร็วและ
เกี่ยวข้อง (relevant) ที่สุด สถาปัตยกรรมภายในต่างจาก PostgreSQL โดยพื้นฐาน:

| แนวคิด | PostgreSQL (Relational DB) | Elasticsearch/OpenSearch |
|--------|------------------------------|--------------------------|
| หน่วยเก็บข้อมูล | **Table** (ตาราง) + **Row** (แถว) | **Index** + **Document** (เอกสาร JSON) |
| โครงสร้าง | Schema ตายตัว, column ชัดเจน | Schema ยืดหยุ่นกว่า (แต่เรากำหนด mapping ได้และควรทำ) |
| วิธีค้นหาข้อความ | B-tree index, GIN index (`tsvector`) | **Inverted index** ออกแบบมาเพื่อ full-text โดยเฉพาะ |
| ผลลัพธ์การค้นหา | ตรง/ไม่ตรงเงื่อนไข (WHERE) | มี **relevance score** บอกว่า "เกี่ยวข้องแค่ไหน" |
| การกระจายข้อมูล | ปกติเป็นเครื่องเดียว (หรือ replica อ่านอย่างเดียว) | ออกแบบมาให้กระจาย (shard) ข้าม node ตั้งแต่แรก |
| Aggregation | `GROUP BY` + `COUNT`/`SUM` | `aggregations` — ทำได้ซับซ้อนกว่าและเร็วกว่าบนข้อมูลค้นหา |

**Inverted index** คือหัวใจของเรื่องนี้ทั้งหมด: แทนที่จะเก็บ "เอกสาร → คำที่มีอยู่" (แบบที่
ตารางเก็บแถว → ค่าคอลัมน์) มันเก็บกลับด้าน คือ **"คำ → รายการเอกสารที่มีคำนั้น"** เหมือนดัชนี
ท้ายเล่มหนังสือ ทำให้การค้นหาคำหนึ่งคำในเอกสารนับล้านชิ้นเร็วมาก เพราะไม่ต้องไล่สแกนทีละ
เอกสาร (คล้าย GIN index ของ PostgreSQL ที่ pg_search ใช้ แต่ Elasticsearch/OpenSearch สร้าง
มาบนพื้นฐานนี้ทั้งระบบ ไม่ใช่ฟีเจอร์เสริม)

### ทำไมทีมถึงยอมเพิ่มความซับซ้อนเพื่อมาใช้เครื่องมือนี้

pg_search ใน Part 068 แก้ปัญหา "ค้นหาคำในข้อความ" ได้ในระดับหนึ่ง แต่มีข้อจำกัดที่ชนเพดาน
เร็วเมื่อความต้องการซับซ้อนขึ้น:

1. **การตัดคำภาษาไทย (แรงจูงใจหลักของ Part นี้)** — Elasticsearch/OpenSearch มาพร้อม
   **Thai analyzer ในตัว** (built-in ไม่ต้องติดตั้ง plugin เพิ่ม) ที่ตัดคำภาษาไทยได้แม่นยำกว่า
   PostgreSQL text search config มาก — จะพิสูจน์ด้วยผลลัพธ์จริงใน Step 686
2. **Relevance tuning ที่ละเอียดกว่า** — ปรับน้ำหนัก field ได้ (เช่น title สำคัญกว่า body),
   ปรับสูตรคำนวณคะแนน (BM25 และปรับ parameter ได้), boost ตามความใหม่ของข้อมูล ฯลฯ
3. **Faceted search / Aggregations** — เช่น หน้าค้นหาสินค้าที่โชว์ "อาหาร (12) เทคโนโลยี (5)
   ท่องเที่ยว (8)" ข้าง filter พร้อมกับผลค้นหา — ทำได้ในคำสั่งเดียวกับการค้นหา และเร็ว
   pg_search ไม่ได้ออกแบบมาทำสิ่งนี้ ต้องเขียน SQL `GROUP BY` แยกต่างหาก (คนละ query,
   คนละรอบ)
4. **Multi-language tokenization ที่กว้างกว่า** — ไม่ใช่แค่ไทย มี analyzer สำหรับญี่ปุ่น, จีน,
   เกาหลี, อารบิก และภาษาอื่นๆ อีกมาก ผ่าน built-in analyzer หรือ ICU plugin
5. **Scale แนวนอน (horizontal scale)** — เมื่อข้อมูลใหญ่ระดับสิบล้าน–พันล้านเอกสาร
   Elasticsearch/OpenSearch กระจาย (shard) ข้ามหลาย node ได้ตั้งแต่การออกแบบ ในขณะที่
   PostgreSQL full-text search อยู่บนเครื่องเดียว (scale แนวตั้งเป็นหลัก)
6. **แยก workload ออกจาก database หลัก** — งานค้นหาหนักๆ ไม่ไปแย่ง CPU/IO กับ
   transactional query ปกติของแอป

> **ข้อควรระวังที่ต้องจำไว้ตลอด Part นี้:** ข้อดีเหล่านี้มาพร้อม **ต้นทุนด้านปฏิบัติการ
> (operational cost)** ที่ไม่เล็กเลย — ต้องรัน ดูแล มอนิเตอร์ระบบแยกต่างหาก, ต้องแก้ปัญหา
> "ข้อมูลใน search index กับฐานข้อมูลไม่ตรงกัน" (index drift), ใช้ RAM หลัก GB ต่อ node
> เราจะพูดเรื่องนี้อย่างตรงไปตรงมาใน Step 690 ว่าเมื่อไหร่ควรใช้ เมื่อไหร่ pg_search ก็พอแล้ว

---

## Step 682: Elasticsearch vs OpenSearch — ประวัติการแยกโปรเจกต์ปี 2021 และความเข้ากันได้

### ทำไมมีสองชื่อ

**Elasticsearch** เปิดตัวปี 2010 โดยบริษัท Elastic (เดิมชื่อ Elasticsearch BV) เติบโตจนกลายเป็น
มาตรฐานของวงการ search engine แบบ open source มายาวนาน สร้างบนพื้นฐานของ **Apache Lucene**
(library ค้นหาข้อความระดับ low-level ที่เขียนด้วย Java)

ปัญหาเริ่มต้นในเดือนมกราคม 2021 เมื่อ Elastic เปลี่ยน license ของ Elasticsearch (และ Kibana)
จาก Apache License 2.0 (open source แท้ๆ) เป็น **Elastic License 2.0 / SSPL** (source-available
แต่ไม่ใช่ open source ตามนิยามของ OSI) เหตุผลที่ Elastic ให้คือต้องการป้องกันไม่ให้ผู้ให้บริการ
คลาวด์รายใหญ่ (พาดพิงถึง AWS โดยตรง) นำโค้ดไปทำเป็นบริการ managed service ขายแข่งโดยไม่ได้
สนับสนุนโปรเจกต์กลับ

**AWS** (ซึ่งมีบริการ "Amazon Elasticsearch Service" อยู่ก่อนแล้ว) ตอบโต้ด้วยการ **fork**
Elasticsearch เวอร์ชันสุดท้ายที่ยังเป็น Apache 2.0 (คือ 7.10.2) ประกาศเมื่อเมษายน 2021 ตั้งชื่อ
โปรเจกต์ใหม่ว่า **OpenSearch** ออกเวอร์ชันเสถียรตัวแรกกลางปี 2021 ยังคงเป็น Apache License 2.0
เต็มรูปแบบ และบริจาคให้มูลนิธิ (ปัจจุบันอยู่ภายใต้ Linux Foundation) เพื่อรับประกันว่าจะยังเป็น
open source จริงต่อไป

> **อัปเดตเรื่อง license ที่ควรรู้:** ในเดือนสิงหาคม 2024 Elastic ได้เพิ่มตัวเลือก license เป็น
> **AGPL v3** ให้เลือกใช้ได้อีกทางหนึ่ง นอกเหนือจาก Elastic License 2.0 และ SSPL เดิม (ผู้ใช้
> เลือกได้ว่าจะรันภายใต้ license ไหน) แต่ไม่ว่าจะเลือกแบบไหน Elasticsearch ตัวปัจจุบันก็ยังไม่ใช่
> Apache 2.0 เหมือนเดิม — นี่คือเหตุผลที่ OpenSearch ยังคงมีที่ยืนในตลาดต่อไป สำหรับทีมที่ต้องการ
> license แบบ permissive ล้วนๆ

### ความแตกต่างที่ต้องรู้สำหรับงานระดับเบื้องต้น

| ประเด็น | Elasticsearch | OpenSearch |
|---------|---------------|------------|
| ผู้ดูแลหลัก | Elastic NV | AWS + ชุมชน (ภายใต้ Linux Foundation) |
| License | Elastic License 2.0 / SSPL / AGPL v3 (เลือกได้) | Apache License 2.0 |
| จุดแยกสาย | — | Fork จาก Elasticsearch 7.10.2 (เมษายน 2021) |
| REST API พื้นฐาน | มาตรฐานเดิม | เกือบเหมือนกันทุกประการสำหรับงานพื้นฐาน (index, search, aggs) |
| เวอร์ชันที่ทดสอบใน Part นี้ | 8.15.0 | 2.19.6 |
| Client ทางการสำหรับ Ruby | gem `elasticsearch` | gem `opensearch-ruby` |

สำหรับงานระดับเบื้องต้นถึงปานกลางที่ Part นี้ครอบคลุม (สร้าง index, กำหนด mapping, indexing,
ค้นหา, aggregation) **REST API ของทั้งสองตัวแทบจะเหมือนกันทุกประการ** เพราะสายพันธุกรรม
ร่วมกันจาก Lucene และ Elasticsearch 7.10.2 เดิม เราทดสอบยืนยันเรื่องนี้จริงในแซนด์บ็อกซ์แล้ว:
**คำสั่ง `_analyze` เดียวกันทุกตัวอักษร ให้ผลลัพธ์เหมือนกันทุกประการ** ทั้งบน OpenSearch
2.19.6 และ Elasticsearch 8.15.0 (ดูผลจริงใน Step 686) ความแตกต่างเริ่มชัดเจนขึ้นเมื่อไปถึง
ฟีเจอร์ขั้นสูงกว่านี้ (machine learning plugin, ข้อ enterprise เฉพาะของแต่ละค่าย) ซึ่งอยู่นอก
ขอบเขตของ Part เบื้องต้นนี้

### แล้วควรเลือกตัวไหน

- **โปรเจกต์ใหม่ ไม่มีข้อผูกมัดเดิม** → OpenSearch มักเป็นตัวเลือกที่ปลอดภัยกว่าในแง่ license
  (Apache 2.0 ชัดเจน ไม่ต้องกังวลเรื่องเงื่อนไขการใช้เชิงพาณิชย์) และถ้าใช้ AWS อยู่แล้วก็มี
  Amazon OpenSearch Service เป็น managed service ให้ใช้ตรงๆ
- **ทีมที่คุ้นเคยกับ Elastic ecosystem อยู่แล้ว** (ใช้ Kibana, Logstash, หรือ Elastic Cloud) →
  Elasticsearch อาจสมเหตุสมผลกว่าเพราะ ecosystem ทั้งชุดออกแบบมาด้วยกัน
- Part นี้จะสาธิตทั้งสองตัว แต่ใช้ **OpenSearch เป็นหลักสำหรับตัวอย่าง Rails integration**
  (เหตุผลทางเทคนิคจะอธิบายใน Step 684) และใช้ Elasticsearch ประกอบเพื่อเทียบให้เห็นภาพ

---

## Step 683: รันอินสแตนซ์ Local ด้วย Docker — คำสั่งจริง ปัญหาจริงที่เจอ และวิธีแก้

### ทำไม Docker คือทางเลือกที่สมเหตุสมผลที่สุดในโลกจริง

Elasticsearch/OpenSearch เป็นโปรแกรม Java (รันบน JVM) การติดตั้งแบบ manual (ดาวน์โหลด
tarball, ตั้งค่า `JAVA_HOME`, ปรับ `ulimit`, ตั้งค่าไฟล์ config หลายไฟล์) ทำได้แต่ยุ่งยากและ
เสี่ยง config ผิดสูง ในทางปฏิบัติ **แทบทุกทีมใช้ Docker (หรือ managed service บนคลาวด์) รัน
Elasticsearch/OpenSearch** แม้แต่ในเครื่อง dev ของตัวเอง

```bash
# Elasticsearch (single-node, สำหรับ dev เท่านั้น — ปิด security ชั่วคราวเพื่อความง่าย)
docker run -d --name es-dev \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.15.0

# OpenSearch (single-node เช่นกัน)
docker run -d --name opensearch-dev \
  -p 9200:9200 -p 9600:9600 \
  -e "discovery.type=single-node" \
  -e "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e "DISABLE_SECURITY_PLUGIN=true" \
  opensearchproject/opensearch:2
```

จุดสำคัญของ flag ที่ใช้:

- `discovery.type=single-node` — บอกให้รันเป็น node เดียวโดยไม่ต้องรอหา node อื่นมาต่อ cluster
  (ปกติ Elasticsearch/OpenSearch ออกแบบมาให้รันหลาย node จึงต้องมีขั้นตอน "ค้นหากันเจอ"
  ถ้าไม่ปิดโหมดนี้ node เดียวจะรอ node อื่นไม่มีวันจบ)
- `ES_JAVA_OPTS`/`OPENSEARCH_JAVA_OPTS` กำหนดขนาด JVM heap — ค่า default มักตั้งเป็นสัดส่วน
  ของ RAM เครื่อง ซึ่งอาจมากเกินไปสำหรับเครื่อง dev; `-Xms512m -Xmx512m` (heap เริ่มต้นและ
  สูงสุด 512MB เท่ากัน ตามคำแนะนำที่ไม่ให้ heap ขยาย/หดระหว่างทำงาน) เพียงพอสำหรับทดลองกับ
  ข้อมูลจำนวนน้อยถึงปานกลาง
- `xpack.security.enabled=false` / `DISABLE_SECURITY_PLUGIN=true` — ปิดระบบ auth/TLS
  ภายในสำหรับ dev เท่านั้น (ห้ามทำแบบนี้ใน production เด็ดขาด — Step 690 จะพูดถึงเรื่องความ
  ปลอดภัยของ production เพิ่มเติม)

### สิ่งที่เกิดขึ้นจริงตอนทดสอบในแซนด์บ็อกซ์นี้ (บันทึกตามจริง)

ตอนเขียน Part นี้ ทีมผู้เขียนรัน `docker pull opensearchproject/opensearch:2` และ
`docker pull elasticsearch:8.15.0` จริงในแซนด์บ็อกซ์ที่มี Docker daemon อยู่แล้ว ผลที่ได้:

```
$ docker pull opensearchproject/opensearch:2
...
Digest: sha256:e321cb03c643874c42240458a618db56bc1e4fc143b10c41bec653542bc6851b
Status: Downloaded newer image for opensearchproject/opensearch:2

$ docker run -d --name opensearch-demo -p 9200:9200 -p 9600:9600 \
    -e "discovery.type=single-node" \
    -e "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "DISABLE_SECURITY_PLUGIN=true" \
    opensearchproject/opensearch:2

$ curl http://localhost:9200/_cluster/health
{"cluster_name":"docker-cluster","status":"green","timed_out":false,
 "number_of_nodes":1,"number_of_data_nodes":1,"discovered_master":true,
 "active_primary_shards":2,"active_shards":2,...}
```

`"status":"green"` แปลว่า cluster (node เดียว) ทำงานปกติสมบูรณ์ — นี่คือของจริง ไม่ใช่ mockup

แต่การพยายามดาวน์โหลด **tarball ตรงๆ** (วิธี manual ที่ไม่ผ่าน Docker) กลับถูกปฏิเสธ:

```
$ curl -I https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.15.0-linux-x86_64.tar.gz
curl: (56) CONNECT tunnel failed, response 403
```

network policy ของแซนด์บ็อกซ์นี้บล็อกโฮสต์ `artifacts.elastic.co` และ `artifacts.opensearch.org`
โดยตรง (ตรวจสอบผ่านสถานะ proxy แล้วว่าเป็นการบล็อกตามนโยบายจริง ไม่ใช่ network ล่ม) แต่
image เดียวกันที่ถูก mirror ไว้บน Docker Hub (`docker.io`) กลับดึงได้ตามปกติ — เป็นตัวอย่างจริง
ของสถานการณ์ที่พบได้บ่อยในองค์กรที่มี firewall/proxy เข้มงวด: **เส้นทางที่ผ่าน registry
กลาง (Docker Hub, หรือ private registry ขององค์กร) มักเปิดผ่านง่ายกว่าการดึงไฟล์ตรงจากเว็บของ
ผู้ผลิตซอฟต์แวร์แต่ละราย**

### ปัญหาที่เจอจริงระหว่างทดสอบ: Disk Watermark บล็อกการสร้าง Index

ระหว่างทดสอบ พบข้อผิดพลาดนี้จริงตอนพยายามสร้าง index แรก:

```
[403] {"error":{"root_cause":[{"type":"index_create_block_exception",
"reason":"blocked by: [FORBIDDEN/10/cluster create-index blocked (api)];"}]},
"status":403}
```

สาเหตุคือ Elasticsearch/OpenSearch มีกลไกป้องกันตัวเองที่เรียกว่า **disk-based shard
allocation watermark** — ถ้าดิสก์ที่ node ใช้เก็บข้อมูลเต็มเกิน threshold ที่กำหนด (ปกติ
"flood stage" อยู่ที่ 95% แต่ค่านี้ปรับได้และขึ้นกับวิธีที่ container มองเห็นพื้นที่ดิสก์ของ host)
ระบบจะตั้ง block ป้องกันการเขียนข้อมูลเพิ่มโดยอัตโนมัติ เพื่อไม่ให้ดิสก์เต็มจนระบบพัง วิธีแก้ที่ใช้
ได้จริงตอนทดสอบ (เหมาะสำหรับเครื่อง dev ที่รู้ว่าดิสก์ยังมีพอ แต่ threshold คำนวณผิดเพราะบริบท
ของ container/disk quota ที่ซ้อนกันหลายชั้น):

```bash
curl -X PUT "http://localhost:9200/_cluster/settings" -H "Content-Type: application/json" -d '{
  "persistent": {
    "cluster.routing.allocation.disk.threshold_enabled": false
  }
}'
```

> **ข้อคิดสำหรับ production:** การปิด disk threshold check แบบนี้ **เหมาะสำหรับ dev/demo
> เท่านั้น** ใน production ควรแก้ที่ต้นเหตุ (เพิ่มพื้นที่ดิสก์จริง หรือปรับ threshold ให้ตรงกับ
> ความจุจริง) เพราะ watermark นี้มีไว้ป้องกันเหตุการณ์ร้ายแรง (ดิสก์เต็มจนเขียนข้อมูลไม่ได้เลย)
> การปิดทิ้งใน production คือการปิดระบบเตือนภัยไฟไหม้ ไม่ใช่การดับไฟ

### ทดสอบว่า plugin เพิ่มเติมติดตั้งอย่างไร (และทำไมทดสอบไม่สำเร็จในแซนด์บ็อกซ์นี้)

Elasticsearch/OpenSearch รองรับการติดตั้ง plugin เพิ่ม (เช่น `analysis-icu` ที่จะพูดถึงใน
Step 686) ด้วยคำสั่ง:

```bash
docker exec opensearch-demo /usr/share/opensearch/bin/opensearch-plugin install analysis-icu
```

ทีมผู้เขียนลองรันคำสั่งนี้จริงกับ container ที่รันอยู่ ผลคือ **ล้มเหลว** ด้วย
`PKIXValidator`/`ValidatorException: unable to find valid certification path` — เพราะ container
ของ OpenSearch ไม่มี CA certificate ของ proxy ที่แซนด์บ็อกซ์นี้ใช้ (JVM ภายใน container มองไม่
เห็น TLS certificate chain ที่ถูกต้องเมื่อพยายามต่อออกไปดาวน์โหลด plugin) นี่คือข้อจำกัดของ
สภาพแวดล้อมทดสอบเฉพาะนี้ ไม่ใช่ปัญหาของ OpenSearch เอง — ในเครื่องทั่วไปที่ต่อเน็ตปกติ
คำสั่งนี้ใช้งานได้ตามเอกสารทางการ รายละเอียดเรื่อง ICU plugin ใน Step 686 จึงมาจากเอกสาร
ทางการเท่านั้น จะระบุไว้ชัดเจนตรงจุดนั้นอีกครั้ง

### ตรวจสอบทรัพยากรที่ใช้จริง

```bash
$ docker stats --no-stream
CONTAINER ID   NAME              CPU %   MEM USAGE / LIMIT     MEM %
3f64b1546533   es-demo           3.11%   1008MiB / 15.72GiB    6.27%
9d15d9f750a5   opensearch-demo   0.93%   924.3MiB / 15.72GiB   5.74%
```

แม้ตั้ง JVM heap ไว้แค่ 512MB แต่ container ใช้ RAM จริงเกือบ **1GB ต่อ node** (heap +
overhead ของ JVM เอง + native memory ของ Lucene) ตัวเลขนี้สำคัญมากสำหรับ Step 690 ตอนคุยเรื่อง
ต้นทุนด้านปฏิบัติการ — นี่คือ node เดียว ไม่มี replica ไม่มีข้อมูลจริง ระบบ production ที่มี
หลาย node พร้อม replica จะใช้ทรัพยากรมากกว่านี้อีกหลายเท่า

---

## Step 684: เชื่อมต่อจาก Ruby/Rails — เลือก gem ให้ถูกกับเอนจิน

### สอง gem ecosystem ที่แยกกันจริง

ฝั่ง Ruby มี gem หลักสองสายที่เกี่ยวข้องกับ Part นี้:

1. **`elasticsearch`** (gem client อย่างเป็นทางการของ Elastic) และ gem ระดับสูงกว่าที่ต่อยอด
   จากมัน คือ **`elasticsearch-model`** + **`elasticsearch-rails`** (ให้ DSL สไตล์ Rails เช่น
   `Model.search(query).records`, `Model.import`)
2. **`opensearch-ruby`** (gem client อย่างเป็นทางการของ OpenSearch Project) — เป็น client
   ระดับ low-level ไม่มี wrapper สำเร็จรูปสไตล์ Rails มาให้ในตัว (ไม่มี `opensearch-model`
   หรือ `opensearch-rails` อย่างเป็นทางการที่ตรงกับชื่อ ตรวจสอบจาก RubyGems แล้วไม่มี gem
   ชื่อนี้ถูก publish ไว้) ต้องเขียน integration layer เอง (ซึ่ง Part นี้จะสาธิตให้ดูทั้งหมด)

### การทดสอบจริงที่เปิดเผยเหตุผลว่าทำไมต้องแยกให้ถูก

ทีมผู้เขียนทดสอบเอา gem `elasticsearch` (เวอร์ชันล่าสุด ณ ตอนเขียน คือ 9.5.0) ไปต่อกับ
**OpenSearch** ที่รันอยู่จริงตรงๆ:

```ruby
require "elasticsearch"
client = Elasticsearch::Client.new(host: "http://localhost:9200", log: false)
client.info
```

ผลลัพธ์จริงที่ได้คือ **error** ไม่ใช่ข้อมูล cluster:

```
ERROR CLASS: Elasticsearch::UnsupportedProductError
ERROR MESSAGE: The client noticed that the server is not Elasticsearch
and we do not support this unknown product.
```

นี่คือของจริงที่ยืนยันแล้ว: gem `elasticsearch` ตั้งแต่เวอร์ชัน 7.14 เป็นต้นมา มีกลไก
**"product check"** ตรวจสอบ response header จากเซิร์ฟเวอร์ว่าเป็น Elasticsearch ของแท้จริง
หรือไม่ (Elastic เพิ่มกลไกนี้เข้ามาหลังการแยกโปรเจกต์กับ AWS พอดี) เมื่อเจอ OpenSearch (ซึ่งไม่
ส่ง header ที่ยืนยันตัวเองว่าเป็น "Elasticsearch") client จะปฏิเสธไม่ทำงานต่อทันที

ตรวจสอบ gemspec ของ `elasticsearch-model` (เวอร์ชันล่าสุด 8.0.1) ยืนยันว่ามัน **ผูก
dependency กับ `elasticsearch ~> 8`** โดยตรง ซึ่งหมายความว่า **`elasticsearch-model`/
`elasticsearch-rails` ก็ได้รับผลกระทบจากกลไก product check นี้ไปด้วยโดยอัตโนมัติ** (เป็นการ
สรุปจาก dependency ที่ตรวจสอบได้จริง ไม่ได้ทดสอบแยกทุกเวอร์ชันของ `elasticsearch-model`
เอง แต่สอดคล้องกับปัญหาที่มีการพูดถึงในชุมชน Ruby อย่างกว้างขวาง)

**ข้อสรุปเชิงปฏิบัติ:** ถ้าฐานข้อมูล search ของคุณคือ **OpenSearch** ให้ใช้ **`opensearch-ruby`**
เท่านั้น อย่าพยายามฝืนใช้ `elasticsearch-model`/`elasticsearch-rails` กับมัน (แม้จะมีวิธี
bypass product check ผ่าน custom transport ได้ทางเทคนิค แต่ไม่คุ้มกับความซับซ้อนที่เพิ่มขึ้น
ในเมื่อมี client ทางการที่ใช้ได้ตรงๆ อยู่แล้ว)

### กับดักที่สองที่พบจริง: เวอร์ชัน client ต้องเข้ากันได้กับเวอร์ชันเซิร์ฟเวอร์

ทดสอบต่อด้วยการเอา gem `elasticsearch` 9.5.0 (client เวอร์ชัน major 9) ไปต่อกับ
**Elasticsearch เซิร์ฟเวอร์จริง 8.15.0** (ของแท้ ไม่ใช่ OpenSearch) คราวนี้ผ่าน product check
แต่พังอีกจุดหนึ่ง:

```
Elastic::Transport::Transport::Errors::BadRequest ([400])
"reason":"Accept version must be either version 8 or 7, but found 9.
Accept=application/vnd.elasticsearch+json; compatible-with=9"
```

client เวอร์ชัน 9 ส่ง HTTP header ขอ API compatibility แบบ "version 9" แต่เซิร์ฟเวอร์เป็น
8.15.0 ยอมรับได้แค่ header ที่ขอ compatibility เป็น version 7 หรือ 8 เท่านั้น พอ**ลดเวอร์ชัน
gem ให้ตรงกับเซิร์ฟเวอร์** (`gem "elasticsearch", "8.15.0"`) ปัญหาหายทันที เชื่อมต่อสำเร็จ
ค้นหาได้ผลลัพธ์ปกติ:

```
Connected OK to real Elasticsearch: 8.15.0
Hits: ["ร้านอาหารไทยรสชาติดี"]
Score: 0.5753642
```

**บทเรียนที่ยืนยันได้จริง:** เมื่อใช้ตระกูล gem ของ Elastic ให้ **pin เวอร์ชัน major ของ gem
`elasticsearch` ให้ตรง (หรือใกล้เคียง) กับเวอร์ชัน major ของเซิร์ฟเวอร์จริงเสมอ** ใน `Gemfile`:

```ruby
# Gemfile — ถ้าใช้ Elasticsearch 8.x บนเซิร์ฟเวอร์
gem "elasticsearch-model", "~> 8.0"
gem "elasticsearch-rails", "~> 8.0"
# (elasticsearch-model จะดึง gem "elasticsearch" ~> 8 มาเป็น dependency ให้อัตโนมัติ)
```

### สรุปตารางการเลือก gem

| เซิร์ฟเวอร์ | Gem ที่ใช้ | หมายเหตุ |
|-------------|-----------|----------|
| Elasticsearch | `elasticsearch-model` + `elasticsearch-rails` | ต้อง pin major version ให้ตรงกับเซิร์ฟเวอร์ |
| Elasticsearch (แบบ low-level เอง) | `elasticsearch` client ตรงๆ | เหมาะถ้าต้องการควบคุมทุกอย่างเอง |
| OpenSearch | `opensearch-ruby` | client ทางการ ไม่มี product check กวนใจ |
| ทั้งสองแบบ (อยากเขียนโค้ดเดียวสลับได้) | gem `searchkick` (เวอร์ชันล่าสุด 6.1.2) | wrapper ระดับสูงที่รองรับทั้งสองฝั่งในตัว เหมาะกับทีมที่อยากได้ DSL สำเร็จรูปโดยไม่สนใจว่าเบื้องหลังเป็นเอนจินไหน (Part นี้ไม่ได้ลงลึก `searchkick` แต่ควรรู้ว่ามีตัวเลือกนี้อยู่) |

Part นี้ที่เหลือจะใช้ **`opensearch-ruby`** เป็นหลักสำหรับสาธิต Rails integration แบบเต็มรูปแบบ
(ตั้งแต่ mapping จนถึง background job) เพราะทดสอบได้ครบวงจรจริงในแซนด์บ็อกซ์นี้ และ code
pattern ที่ได้สามารถนำไปปรับใช้กับ `elasticsearch-model`/`elasticsearch-rails` ได้ไม่ยาก
(concept เหมือนกัน ต่างกันแค่ชื่อ method/gem)

ติดตั้ง gem:

```ruby
# Gemfile
gem "opensearch-ruby", "~> 3.4"
```

```bash
bundle install
```

---

## Step 685: กำหนด Index Mapping สำหรับ Model

### ทำไมต้องกำหนด Mapping เอง (อย่าปล่อยให้ dynamic mapping เดามั่ว)

ถ้าไม่กำหนด mapping ไว้ล่วงหน้าแล้วยัดเอกสารเข้าไปตรงๆ Elasticsearch/OpenSearch จะเดา field
type ให้เองจากค่าตัวอย่างแรกที่เห็น (เรียกว่า **dynamic mapping**) ปัญหาคือมันมักเดาผิดสำหรับ
ความต้องการของเรา ตัวอย่างจริงที่พบระหว่างทดสอบ Part นี้:

สร้าง record แรกโดยยังไม่ได้สร้าง index พร้อม mapping ไว้ก่อน → ระบบ auto-create index ให้
โดยเดาว่า field `category` (ค่าเช่น `"อาหาร"`) เป็น `text` (เอาไว้ full-text search) พอลอง
ทำ aggregation (`terms` aggregation สำหรับ facet) บน field นี้ ได้ error ทันที:

```
"reason":"Text fields are not optimised for operations that require
per-document field data like aggregations and sorting, so these operations
are disabled by default. Please use a keyword field instead."
```

field `category` ควรเป็น **`keyword`** (ค่าที่ตรงเป๊ะ ไม่ตัดคำ ใช้สำหรับ filter/aggregation/
sort) ไม่ใช่ `text` (ตัดคำเพื่อ full-text search) — ปัญหานี้แก้ไม่ได้ด้วยการ "แก้ mapping ทีหลัง"
เพราะ Elasticsearch/OpenSearch **ไม่ให้เปลี่ยน type ของ field ที่มีอยู่แล้วโดยตรง** ต้องสร้าง
index ใหม่แล้ว reindex ข้อมูลทั้งหมด — บทเรียนสำคัญ: **กำหนด mapping ให้ถูกต้อง "ก่อน"
ยัดข้อมูลเข้าไปจริงจังเสมอ**

### Field type หลักที่ต้องรู้จัก

| Type | ใช้เมื่อไหร่ | ตัวอย่าง field |
|------|--------------|----------------|
| `text` | ต้องการ full-text search (ตัดคำ, วิเคราะห์ด้วย analyzer) | title, body, description |
| `keyword` | ค่าที่ต้อง exact match, filter, aggregation, sort | category, status, sku, tag |
| `date` | วันที่/เวลา | published_at, created_at |
| `integer`/`long`/`float` | ตัวเลข | price, view_count, rating |
| `boolean` | จริง/เท็จ | published, featured |
| `nested`/`object` | โครงสร้างซ้อน (นอกขอบเขต Part เบื้องต้นนี้) | comments, variants |

> **เทคนิคที่ใช้บ่อย:** field เดียวกำหนดได้ทั้งสอง type พร้อมกันผ่าน "multi-field" เช่น
> `title` เป็น `text` สำหรับค้นหา และมี sub-field `title.keyword` (type `keyword`) ไว้สำหรับ
> sort ตามตัวอักษรหรือ exact match ในคำสั่งเดียว — dynamic mapping ของ OpenSearch/
> Elasticsearch จริงๆ สร้าง multi-field แบบนี้ให้อัตโนมัติสำหรับ string ที่ไม่ได้กำหนด mapping
> เอง (นี่คือเหตุผลที่ error ข้างบนแนะนำให้ "ใช้ `category.keyword`" แทนได้เช่นกัน ถ้าไม่อยาก
> กำหนด mapping เอง — แต่ Part นี้แนะนำให้กำหนด mapping ชัดเจนตั้งแต่ต้นดีกว่า อ่านง่ายกว่า)

### เขียน Searchable concern — ส่วนที่ 1: การสร้าง Index พร้อม Mapping

เราจะสร้าง `ActiveSupport::Concern` ชื่อ `Searchable` ที่ `include` เข้าไปใน Model ไหนก็ได้ที่
ต้องการ full-text search โครงเริ่มต้น:

```ruby
# app/models/concerns/searchable.rb
# frozen_string_literal: true

require "opensearch"

module Searchable
  extend ActiveSupport::Concern

  class_methods do
    def search_client
      @search_client ||= OpenSearch::Client.new(
        host: ENV.fetch("OPENSEARCH_URL", "http://localhost:9200"),
        log: false
      )
    end

    # ตั้งชื่อ index แยกตาม environment กันข้อมูล dev/test/production ปนกัน
    def search_index_name
      "#{name.underscore.pluralize}_#{Rails.env}"
    end

    def create_search_index!
      return if search_client.indices.exists(index: search_index_name)

      search_client.indices.create(
        index: search_index_name,
        body: {
          settings: { number_of_shards: 1, number_of_replicas: 0 },
          mappings: { properties: search_mapping }
        }
      )
    end
  end

  # ให้แต่ละ Model override ได้ว่า field ไหนเป็น type อะไร
  module ClassMethods
    def search_mapping
      {
        title: { type: "text", analyzer: "thai" },
        body: { type: "text", analyzer: "thai" },
        category: { type: "keyword" },
        created_at: { type: "date" }
      }
    end
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Searchable
end
```

### ทดสอบจริง

```
$ bin/rails runner 'Post.create_search_index!; puts "OK"'
OK

$ curl -s http://localhost:9200/posts_development/_mapping | python3 -m json.tool
{
    "posts_development": {
        "mappings": {
            "properties": {
                "title": { "type": "text", "analyzer": "thai" },
                "body": { "type": "text", "analyzer": "thai" },
                "category": { "type": "keyword" },
                "created_at": { "type": "date" }
            }
        }
    }
}
```

Mapping ถูกสร้างตามที่กำหนดไว้เป๊ะ — ยืนยันจากการ query `_mapping` API จริง (ไม่ใช่การเดา
ว่ามันน่าจะออกมาแบบนี้)

---

## Step 686: Thai-language Analyzer — วิธีแก้ปัญหาการตัดคำภาษาไทยที่ Part 068 เจอ

นี่คือหัวใจของเหตุผลที่ Part นี้มีอยู่ มาพิสูจน์กันด้วยผลลัพธ์จริง ไม่ใช่คำอธิบายลอยๆ

### พิสูจน์ปัญหาก่อน: `standard` analyzer กับข้อความภาษาไทย

`standard` analyzer คือ analyzer เริ่มต้นของ Elasticsearch/OpenSearch (ใช้หลักการตัดคำแบบ
Unicode text segmentation ทั่วไป เหมาะกับภาษาที่มีช่องว่างคั่นคำอย่างอังกฤษ) ทดสอบยิงประโยค
ภาษาไทยที่ไม่มีช่องว่างคั่นคำเลย (ซึ่งเป็นธรรมชาติของภาษาไทยจริงๆ) เข้า `_analyze` API:

```bash
curl -X POST "http://localhost:9200/_analyze" -H "Content-Type: application/json" -d '{
  "analyzer": "standard",
  "text": "ร้านอาหารไทยรสชาติดีที่สุดในกรุงเทพมหานคร"
}'
```

**ผลลัพธ์จริง:**

```json
{
  "tokens": [
    {
      "token": "ร้านอาหารไทยรสชาติดีที่สุดในกรุงเทพมหานคร",
      "start_offset": 0,
      "end_offset": 41,
      "type": "<SOUTHEAST_ASIAN>",
      "position": 0
    }
  ]
}
```

ทั้งประโยค 41 ตัวอักษรกลายเป็น **token เดียว** — เหมือนกับปัญหาที่ PostgreSQL config แบบ
`simple` เจอใน Part 068 เป๊ะ ถ้า index ข้อมูลด้วย analyzer นี้ การค้นหาคำว่า "ร้านอาหาร"
เดี่ยวๆ จะ**ไม่เจอ**เอกสารนี้เลย เพราะระบบมองว่ามันเป็นคำคนละคำกับ token ยักษ์ที่เก็บไว้

### วิธีแก้: analyzer ชื่อ `thai`

ทดสอบข้อความเดียวกันเป๊ะ เปลี่ยนแค่ `analyzer` เป็น `thai`:

```bash
curl -X POST "http://localhost:9200/_analyze" -H "Content-Type: application/json" -d '{
  "analyzer": "thai",
  "text": "ร้านอาหารไทยรสชาติดีที่สุดในกรุงเทพมหานคร"
}'
```

**ผลลัพธ์จริง:**

```json
{
  "tokens": [
    { "token": "ร้าน",         "position": 0 },
    { "token": "อาหาร",        "position": 1 },
    { "token": "ไทย",          "position": 2 },
    { "token": "รสชาติ",       "position": 3 },
    { "token": "ดี",           "position": 4 },
    { "token": "กรุงเทพมหานคร", "position": 7 }
  ]
}
```

ประโยคถูกตัดคำแยกออกมาถูกต้องเกือบทั้งหมด (สังเกตว่า position กระโดดจาก 4 ไป 7 — คำว่า
"ที่", "สุด", "ใน" ถูกกรองออกเพราะเป็น **Thai stopword** ที่ analyzer นี้กรองทิ้งโดยปริยาย
เหมือนที่ `english` analyzer กรอง "the", "a", "is" ทิ้ง) ทดสอบยืนยันแล้วว่า **ผลลัพธ์เหมือนกัน
เป๊ะทั้งบน OpenSearch 2.19.6 และ Elasticsearch 8.15.0** — เพราะ analyzer `thai` นี้มาจาก
**Apache Lucene core โดยตรง** (คลาส `ThaiAnalyzer`) ไม่ใช่ plugin แยกต่างหาก จึงติดตัวมาให้
ทั้งสองเอนจินโดยไม่ต้องติดตั้งอะไรเพิ่มเลย — นี่คือคำตอบตรงๆ ของ "ปัญหา Thai tokenization"
ที่ Part 068 ทิ้งไว้

### พิสูจน์ต่อว่ามันมีประโยชน์จริงตอนค้นหา (ไม่ใช่แค่ตัดคำสวยๆ)

ทดสอบ index เอกสารสองชิ้นที่มีคำว่า "ต้มยำกุ้ง" (คำเดี่ยว ไม่มีช่องว่าง) ฝังอยู่ในประโยคยาว
แล้วค้นหาด้วยแค่คำว่า "ต้มยำ" (เป็นส่วนหนึ่งของ "ต้มยำกุ้ง"):

```json
// query: { "match": { "body": "ต้มยำ" } }
```

**ผลลัพธ์จริง:**

```
score=0.975  สอนทำต้มยำกุ้งฉบับร้านอาหารไทย
score=0.902  รีวิวร้านอาหารไทยรสชาติดั้งเดิม
```

เจอทั้งสองเอกสารที่มีคำว่า "ต้มยำกุ้ง" ฝังอยู่ แม้ query จะสั้นกว่าคำเต็มในประโยค เพราะ Thai
analyzer ตัด "ต้มยำกุ้ง" ออกเป็น token ย่อยๆ ("ต้มยำ", "กุ้ง") ตั้งแต่ตอน index แล้ว การค้นหา
"ต้มยำ" เดี่ยวๆ จึง match กับ token "ต้มยำ" ที่ถูกตัดเก็บไว้ได้โดยตรง — สิ่งนี้เป็นไปไม่ได้เลย
กับ `standard` analyzer ที่เก็บทั้งประโยคเป็นก้อนเดียว

### ตั้งค่า mapping ให้ใช้ analyzer นี้ (ทวนจาก Step 685)

```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "thai" },
      "body":  { "type": "text", "analyzer": "thai" }
    }
  }
}
```

ซึ่งตรงกับที่เขียนไว้ใน `search_mapping` ของ `Searchable` concern ใน Step 685 อยู่แล้ว —
เพียงใส่ `analyzer: "thai"` ให้ทุก field ที่เป็นภาษาไทยและต้องการ full-text search

### ทางเลือกเพิ่มเติม: ICU Analysis Plugin (เอกสารทางการ — ไม่ได้ทดสอบจริงใน Part นี้)

> **ย้ำอีกครั้งตามที่แจ้งไว้ต้น Part:** ส่วนนี้มาจากเอกสารทางการของ Elasticsearch/OpenSearch
> เท่านั้น การติดตั้ง plugin จริงในแซนด์บ็อกซ์นี้ล้มเหลวเพราะข้อจำกัดเรื่อง TLS certificate ของ
> container (ดู Step 683) จึงไม่มีผลลัพธ์จริงมายืนยันในหัวข้อนี้

Plugin ชื่อ **`analysis-icu`** (มาจาก International Components for Unicode) เพิ่มความสามารถ
ด้าน text processing ที่กว้างกว่า built-in analyzer ปกติ เช่น:

- `icu_tokenizer` — ตัดคำโดยใช้กฎ Unicode text segmentation ที่ซับซ้อนกว่า `standard`
  รองรับหลายภาษารวมถึงภาษาที่ไม่มีช่องว่างคั่นคำ (ไทย, จีน, ญี่ปุ่น) ได้ดีขึ้นในบางกรณี
  โดยเฉพาะเมื่อผสมกับ dictionary เพิ่มเติม
- `icu_normalizer`/`icu_folding` — ทำให้ตัวอักษรที่หน้าตาต่างกันแต่ความหมายเดียวกัน
  (เช่น สระ/วรรณยุกต์ที่เขียนคนละรูปแบบ Unicode normalization form) ถูกมองว่าเป็นตัวเดียวกัน
  ตอนค้นหา

ติดตั้งด้วยคำสั่ง (ตามเอกสาร):

```bash
bin/opensearch-plugin install analysis-icu   # หรือ bin/elasticsearch-plugin สำหรับ Elasticsearch
```

สำหรับภาษาไทยโดยเฉพาะ **built-in `thai` analyzer ที่พิสูจน์แล้วข้างบนก็เพียงพอสำหรับงาน
ส่วนใหญ่** — `analysis-icu` มีประโยชน์มากกว่าเมื่อระบบต้องรองรับ**หลายภาษาพร้อมกัน**ใน field
เดียว หรือต้องการ normalize Unicode variant ที่ซับซ้อน ถ้าแอปพลิเคชันมีแค่ภาษาไทย+อังกฤษ
built-in analyzer ที่ทดสอบไปแล้วก็ตอบโจทย์ได้โดยไม่ต้องเพิ่มความซับซ้อนของการติดตั้ง plugin

---

## Step 687: Indexing ข้อมูลจาก Model — `Post.import` และ Callback แบบ `after_commit`

### รูปแบบที่ 1: Sync ทันทีตอน save ผ่าน callback

เพิ่มเข้าไปใน `Searchable` concern:

```ruby
# app/models/concerns/searchable.rb (เพิ่มเติมจาก Step 685)
module Searchable
  extend ActiveSupport::Concern

  included do
    after_commit :index_to_search_engine, on: [:create, :update]
    after_commit :remove_from_search_engine, on: :destroy
  end

  # ... (search_client, search_index_name, create_search_index! เหมือน Step 685) ...

  def as_indexed_json
    { title: title, body: body, category: category, created_at: created_at }
  end

  def index_to_search_engine
    self.class.create_search_index! # กันไม่ให้ dynamic mapping เดาผิด (ดู Step 685)
    self.class.search_client.index(
      index: self.class.search_index_name,
      id: id,
      body: as_indexed_json,
      refresh: true
    )
  end

  def remove_from_search_engine
    self.class.search_client.delete(index: self.class.search_index_name, id: id, refresh: true)
  rescue OpenSearch::Transport::Transport::Errors::NotFound
    nil # ลบไปแล้วก็ไม่เป็นไร ไม่ต้อง error
  end
end
```

> **ทำไมใช้ `after_commit` ไม่ใช่ `after_save`:** `after_save` รันอยู่ **ภายใน transaction**
> ที่ยังไม่ commit ถ้า transaction ถูก rollback ทีหลัง (เช่นเพราะ validation อื่นในเดียวกันล้ม)
> ข้อมูลจะถูกส่งไป index ทั้งที่จริงๆ ไม่เคยถูกบันทึกจริงในฐานข้อมูลเลย `after_commit` รันหลัง
> transaction สำเร็จแล้วเท่านั้น การันตีว่า record ที่เอาไป index มีอยู่จริงในฐานข้อมูล

### ทดสอบจริง

```ruby
Post.create!(title: "รีวิวร้านอาหารไทยรสชาติดั้งเดิม", body: "...", category: "อาหาร")
```

```
$ bin/rails runner test_integration.rb
=== 1) สร้างโพสต์ผ่าน ActiveRecord (after_commit จะ index เข้า OpenSearch อัตโนมัติ) ===
สร้างแล้ว 6 โพสต์

=== 2) ตรวจสอบว่า index เข้า OpenSearch จริงหรือไม่ (นับเอกสารตรงๆ) ===
จำนวนเอกสารใน index posts_development: 6
```

ยืนยันแล้วว่า record ที่สร้างผ่าน ActiveRecord ปกติ ถูกส่งเข้า OpenSearch จริงโดยอัตโนมัติ
ผ่าน callback โดยไม่ต้องเรียกอะไรเพิ่มเอง

### รูปแบบที่ 2: `Post.import` — bulk reindex ทั้งหมดจากฐานข้อมูล

Callback ข้างบนใช้ได้ดีสำหรับ record ที่สร้าง/แก้ทีละตัว แต่เวลาต้อง **reindex ข้อมูลทั้งหมด**
(เช่น หลังเปลี่ยน mapping, หรือครั้งแรกที่เปิดใช้ search feature กับข้อมูลเก่าที่มีอยู่แล้ว) การ
วิ่งทีละ record ช้าเกินไป ต้องใช้ **Bulk API**:

```ruby
class_methods do
  def import
    create_search_index!
    find_in_batches do |batch| # แบ่งเป็นชุดละ 1000 record (ค่า default ของ find_in_batches)
      bulk_body = batch.flat_map do |record|
        [
          { index: { _index: search_index_name, _id: record.id } },
          record.as_indexed_json
        ]
      end
      search_client.bulk(body: bulk_body) if bulk_body.any?
    end
    search_client.indices.refresh(index: search_index_name)
  end
end
```

`find_in_batches` มาจาก Part 034 (Query interface ขั้นสูง) — สำคัญมากเมื่อ table มีข้อมูลหลัก
แสนหลักล้าน record เพราะโหลดทีละ batch แทนที่จะดึงทั้งหมดเข้าหน่วยความจำทีเดียว **Bulk API**
ของ Elasticsearch/OpenSearch ก็ออกแบบมาเพื่อการนี้โดยเฉพาะ — ส่งหลายคำสั่ง index/update/delete
ใน HTTP request เดียว เร็วกว่าการยิงทีละ request มาก

**ทดสอบจริง:**

```
=== 6) ทดสอบ Post.import (reindex ทั้งหมดใหม่จากฐานข้อมูล) ===
หลัง Post.import: 6 เอกสาร
```

(ทดสอบโดยลบ index ทิ้งก่อนแล้วเรียก `Post.import` ใหม่ทั้งหมด ยืนยันว่าจำนวนเอกสารกลับมา
ครบตามจำนวน record ในฐานข้อมูล)

---

## Step 688: การค้นหา (Search) พร้อม Relevance Scoring, Fuzzy Matching, Highlighting

### `Post.search("query").records` — ให้หน้าตาเหมือน ActiveRecord query

```ruby
class_methods do
  def search(query, category: nil)
    filter = category.present? ? [{ term: { category: category } }] : []

    response = search_client.search(
      index: search_index_name,
      body: {
        query: {
          bool: {
            must: { multi_match: { query: query, fields: ["title^2", "body"] } },
            filter: filter
          }
        }
      }
    )
    SearchResult.new(self, response)
  end
end

class SearchResult
  include Enumerable

  def initialize(model_class, response)
    @model_class = model_class
    @response = response
  end

  def hits = @response.dig("hits", "hits") || []

  def records
    ids = hits.map { |h| h["_id"] }
    scores = hits.each_with_object({}) { |h, acc| acc[h["_id"].to_i] = h["_score"] }
    @model_class.where(id: ids)
                .sort_by { |r| ids.index(r.id.to_s) } # รักษาลำดับตาม relevance score
                .each { |r| r.define_singleton_method(:search_score) { scores[r.id] } }
  end

  def each(&block) = records.each(&block)
end
```

จุดที่ควรสังเกต:

- `multi_match` ค้นหา query เดียวกันในหลาย field พร้อมกัน — `fields: ["title^2", "body"]`
  หมายความว่า **ถ้า query เจอใน `title` ให้คะแนนหนักเป็น 2 เท่า** เทียบกับเจอใน `body`
  (field boosting) — ปรับตัวเลขนี้ได้ตามความสำคัญจริงของแต่ละ field ในแอปพลิเคชัน
- `.where(id: ids)` ดึง record จริงจาก PostgreSQL กลับมา (Elasticsearch/OpenSearch เก็บแค่
  สำเนาข้อมูลไว้ค้นหา ไม่ใช่แหล่งความจริงหลักของข้อมูล — **ฐานข้อมูลหลักยังคงเป็น
  PostgreSQL เสมอ** นี่คือรูปแบบมาตรฐานที่เกือบทุกระบบใช้)
- `sort_by { ids.index(...) }` จำเป็นเพราะ `.where(id: ids)` ของ ActiveRecord **ไม่รับประกัน
  ลำดับผลลัพธ์**ตาม `ids` ที่ส่งเข้าไป (SQL `WHERE id IN (...)` ไม่มีลำดับในตัวมันเอง) ต้อง
  เรียงกลับตามลำดับ relevance score จาก Elasticsearch/OpenSearch เอง

### ทดสอบจริง — Relevance Scoring

```
=== 3) Post.search('ร้านอาหารไทย').records — full text + relevance ===
score=4.659  "รีวิวร้านอาหารไทยรสชาติดั้งเดิม" (id=8)
score=4.659  "สอนทำต้มยำกุ้งฉบับร้านอาหารไทย" (id=9)
score=1.498  "เทคโนโลยี AI เปลี่ยนวงการอาหารอย่างไร" (id=11)
score=1.268  "รีวิวร้านกาแฟบรรยากาศดีใจกลางกรุงเทพ" (id=12)
```

สังเกตว่าโพสต์ที่มีคำว่า "ร้าน", "อาหาร", "ไทย" ครบทั้งสามคำใน title ได้คะแนนสูงสุดเท่ากัน
(4.659) ส่วนโพสต์ที่มีแค่บางคำ ("อาหาร" อย่างเดียว หรือ "ร้าน" อย่างเดียว) ได้คะแนนต่ำกว่าตาม
สัดส่วนคำที่ match — นี่คือ **BM25** (อัลกอริทึมคำนวณ relevance เริ่มต้นของทั้ง Elasticsearch
และ OpenSearch ตั้งแต่เวอร์ชัน 5 เป็นต้นมา) ที่คิดจากทั้งความถี่ของคำในเอกสาร (term frequency)
และความหายากของคำในทุกเอกสาร (inverse document frequency) รวมถึงความยาวเอกสาร — เรื่องที่
pg_search ทำได้จำกัดกว่ามาก (PostgreSQL ใช้สูตรคำนวณ `ts_rank` ที่ง่ายกว่าและปรับแต่งได้น้อย
กว่า BM25)

### Bonus ที่ทดสอบจริงเพิ่มเติม: Fuzzy Matching (ทนต่อคำพิมพ์ผิด)

```json
{
  "query": {
    "match": { "body": { "query": "ตมยำ", "fuzziness": "AUTO" } }
  },
  "highlight": { "fields": { "body": {} } }
}
```

query คำว่า "ตมยำ" (พิมพ์ตกวรรณยุกต์ไม้โทหาย ที่ถูกคือ "ต้มยำ") **ทดสอบจริง** — โดยไม่ใส่
`fuzziness` เลยได้ผลลัพธ์ 0 รายการ (ไม่เจออะไรเลย เพราะสะกดไม่ตรง) แต่พอใส่
`"fuzziness": "AUTO"` (ให้ระบบยอมรับความต่างของตัวอักษรได้ตาม Levenshtein distance ที่คำนวณ
อัตโนมัติตามความยาวคำ) **เจอผลลัพธ์ 2 รายการทันที**:

```json
"highlight": {
  "body": ["สูตร<em>ต้มยำ</em>กุ้งน้ำข้นที่ร้านอาหารไทยหลายแห่งใช้ทำขาย"]
}
```

`highlight` ยังห่อคำที่ match ด้วยแท็ก `<em>` ให้อัตโนมัติ (นำไปแสดงในหน้าเว็บเป็นตัวหนา/
ไฮไลต์สีได้ทันที) — ความสามารถทั้งสองนี้ (fuzzy + highlight พร้อมกันในคำสั่งเดียว) เป็นสิ่งที่
pg_search ต้องพึ่ง extension เพิ่มเติมอย่าง `pg_trgm` สำหรับ fuzzy และเขียน logic ห่อคำ match
เองสำหรับ highlight — ในขณะที่นี่คือฟีเจอร์ในตัวที่ใช้ได้ทันที

---

## Step 689: Faceted Search / Aggregations — นับจำนวนโพสต์ต่อหมวดหมู่พร้อมผลค้นหา

### Facet คืออะไร และทำไม pg_search ทำได้ยาก

ลองนึกภาพหน้าค้นหาสินค้าอีคอมเมิร์ซทั่วไป: พิมพ์คำค้นหา แล้วข้างๆ ผลลัพธ์มีกล่อง filter
โชว์ "อาหาร (3) เทคโนโลยี (2) ท่องเที่ยว (1)" ให้กดกรองต่อได้ — นี่คือ **faceted search**
สิ่งที่ทำให้มันยากถ้าใช้ SQL ธรรมดา คือต้องรัน query สองรอบแยกกัน (หนึ่งรอบดึงผลค้นหา อีกรอบ
`GROUP BY category` นับจำนวน) และการนับต้องคำนึงถึงเงื่อนไขค้นหาเดียวกันด้วย ไม่ใช่นับทั้งตาราง

Elasticsearch/OpenSearch ทำสองอย่างนี้ **ในคำสั่งเดียว** ผ่าน `aggregations` (เขียนย่อว่า
`aggs`) ที่แนบไปกับ query ค้นหาปกติ:

```ruby
def search(query, category: nil)
  filter = category.present? ? [{ term: { category: category } }] : []

  response = search_client.search(
    index: search_index_name,
    body: {
      query: {
        bool: {
          must: { multi_match: { query: query, fields: ["title^2", "body"] } },
          filter: filter
        }
      },
      aggs: {
        posts_per_category: { terms: { field: "category" } } # <-- เพิ่มบรรทัดนี้
      }
    }
  )
  SearchResult.new(self, response)
end
```

เพิ่ม method อ่านผล aggregation ใน `SearchResult`:

```ruby
class SearchResult
  # ... (เหมือนเดิม) ...

  def category_facets
    buckets = @response.dig("aggregations", "posts_per_category", "buckets") || []
    buckets.each_with_object({}) { |b, acc| acc[b["key"]] = b["doc_count"] }
  end
end
```

`terms` aggregation ทำงานบน field type `keyword` เท่านั้น (ตามที่อธิบายไว้ใน Step 685 —
นี่คือเหตุผลที่ `category` ต้องเป็น `keyword` ไม่ใช่ `text`) มันจับกลุ่มเอกสารตามค่าที่ต่างกัน
ของ field นั้น แล้วนับจำนวนในแต่ละกลุ่มให้อัตโนมัติ

### ทดสอบจริง

```
=== 4) Facet: category_facets จาก aggregation เดียวกับ search ===
อาหาร: 3
เทคโนโลยี: 1
```

(ผลนี้มาจากการค้นหาคำว่า "ร้านกาแฟ" ในชุดข้อมูลทดสอบ 8 โพสต์ — สังเกตว่า **facet นับเฉพาะ
เอกสารที่ match กับ query ค้นหาด้วย** ไม่ใช่นับทั้ง index ทั้งหมด เพราะ `aggs` ในตัวอย่างนี้
คำนวณอยู่ใน scope เดียวกับ `query` — นี่คือพฤติกรรมที่ตรวจสอบแล้วจริง: หมวด "ท่องเที่ยว" ที่มี
อยู่ใน index ไม่ปรากฏใน facet เลย เพราะไม่มีโพสต์หมวดนั้นที่ match คำค้นหานี้)

> **ข้อควรคิดสำหรับหน้า UI จริง (จะลงรายละเอียดเพิ่มใน Part 070):** ระบบ UI จริงหลายระบบ
> อยากให้ facet โชว์ **ตัวเลือกที่เหลือทั้งหมด**แม้ตอนกรองอยู่แล้ว (เช่น กรอง "อาหาร" อยู่
> แต่ยังอยากเห็นว่า "เทคโนโลยี" มีกี่ชิ้นถ้าจะสลับไปดู) วิธีทำคือแยก query สำหรับ aggregation
> ออกจาก filter หลัก (aggregation ที่ไม่รวม filter ของมิติที่กำลังแสดงอยู่) เป็นเทคนิคขั้นสูงกว่า
> ที่ Part นี้ไม่ลงลึก แต่ควรรู้ไว้ว่ามีทางเลือกออกแบบได้มากกว่าตัวอย่างพื้นฐานข้างบน

---

## Step 690: การ Sync Search Index กับฐานข้อมูล — Index Drift, Reindex, Background Job, และต้นทุนที่ต้องคิด

### ปัญหา "Index Drift" — พิสูจน์ให้เห็นจริง

ระบบที่มีฐานข้อมูลหลัก (PostgreSQL) กับ search index (Elasticsearch/OpenSearch) แยกกันสองที่
มีความเสี่ยงถาวรที่ **ข้อมูลทั้งสองฝั่งไม่ตรงกัน** เรียกปัญหานี้ว่า **index drift** ทดสอบให้เห็น
จริงด้วยสถานการณ์ที่เกิดขึ้นได้จริงในระบบ production (เช่น migration script, batch job,
console ที่ engineer รันตรงๆ โดยไม่ผ่าน ActiveRecord callback):

```ruby
# ลบ record ผ่าน raw SQL ตรงๆ (ข้าม after_commit callback ของ ActiveRecord ไปเลย)
Post.connection.execute("DELETE FROM posts WHERE id = #{post.id}")
```

**ผลลัพธ์จริง:**

```
ลบ post id=8 ออกจาก DB ตรงๆ (bypass callback) — DB เหลือ 5 แถว
แต่ยังอยู่ใน OpenSearch index หรือไม่: true <= นี่คือ 'index drift'
```

Record หายไปจากฐานข้อมูลจริง แต่ **ยังค้นหาเจอใน OpenSearch** เพราะ callback ที่ลบออกจาก
index ไม่เคยถูกเรียก — ผู้ใช้จะเห็นผลค้นหาที่ชี้ไปยัง record ที่ไม่มีอยู่จริงแล้ว (คลิกเข้าไปเจอ
404) สถานการณ์แบบนี้เกิดขึ้นได้จากหลายสาเหตุ: bulk update ผ่าน SQL ตรงๆ, callback error ที่
ถูก rescue เงียบๆ, worker ที่ crash กลางทาง, หรือแค่ปิด callback ชั่วคราวตอน migrate ข้อมูล
จำนวนมาก

### กลยุทธ์รับมือ Index Drift

1. **ป้องกันไม่ให้เกิดตั้งแต่ต้น** — หลีกเลี่ยงการแก้ข้อมูลผ่าน raw SQL/`update_all`/
   `delete_all` กับ Model ที่มี Searchable โดยไม่ตั้งใจ sync index ตามไปด้วย (หรือถ้าจำเป็นต้อง
   ทำ ให้ตามด้วยการ reindex เฉพาะส่วนที่กระทบ)
2. **Full reindex เป็นระยะ (periodic reindex)** — ตั้ง cron/scheduled job (จาก Part 062 เรื่อง
   Sidekiq scheduled job) ให้รัน `Post.import` ใหม่ทั้งหมดเป็นระยะ (เช่น ทุกคืน) เพื่อ "รีเซ็ต"
   ความไม่ตรงกันที่สะสมมา เหมาะกับข้อมูลที่ไม่ใหญ่มากจนรัน full reindex ช้าเกินไป
3. **Incremental reindex ด้วย `updated_at` cursor** — สำหรับข้อมูลใหญ่ ไม่ต้อง reindex ทั้งหมด
   ทุกครั้ง แค่ query record ที่ `updated_at > last_reindexed_at` มา sync เพิ่ม
4. **Index แบบ asynchronous ผ่าน background job** — วิธีที่แนะนำที่สุดสำหรับ production
   (รายละเอียดถัดไป)

### ทำไมควร Index แบบ Asynchronous (เชื่อมกับ Part 061)

ตัวอย่างใน Step 687 ใช้ `after_commit` เรียก `search_client.index(...)` **แบบ synchronous**
(รอผลจบก่อน request/console ถึงจะทำงานต่อ) วิธีนี้ใช้งานง่ายและเพียงพอสำหรับตัวอย่างเรียนรู้
แต่ใน production มีปัญหา: **ถ้า Elasticsearch/OpenSearch ตอบช้าหรือล่มชั่วคราว
request ของผู้ใช้ที่กำลัง save ข้อมูลจะค้างรอไปด้วย** — ตรงกับหลักการที่ Part 061 สอนไว้ตั้งแต่
ต้น: **งานที่ไม่จำเป็นต้องรู้ผลทันทีควรส่งไปทำเบื้องหลัง**

```ruby
# app/jobs/search_index_job.rb
class SearchIndexJob < ApplicationJob
  queue_as :search
  discard_on ActiveJob::DeserializationError # record ถูกลบไปแล้วก่อน job จะรัน ก็แค่ข้าม

  def perform(record)
    if record.destroyed?
      record.class.search_client.delete(index: record.class.search_index_name, id: record.id)
    else
      record.class.create_search_index!
      record.class.search_client.index(
        index: record.class.search_index_name,
        id: record.id,
        body: record.as_indexed_json
      )
    end
  end
end
```

```ruby
# app/models/concerns/searchable.rb — เปลี่ยน callback ให้ enqueue job แทนรันตรงๆ
included do
  after_commit -> { SearchIndexJob.perform_later(self) }, on: [:create, :update]
  after_commit -> { SearchIndexJob.perform_later(self) }, on: :destroy
end
```

สังเกตว่าเราส่ง **record object เข้าไปตรงๆ** (`perform_later(self)`) ไม่ใช่แค่ `id` — เหมือนที่
Part 061 Step 604 อธิบายไว้ว่า ActiveJob ใช้ **GlobalID** แปลง ActiveRecord object เป็น
reference แล้วดึงข้อมูลสดใหม่จากฐานข้อมูลตอน job รันจริง (ไม่ใช่ snapshot ค่าตอน enqueue)
ซึ่งสำคัญมากสำหรับ use case นี้: ถ้า record ถูกแก้ไขอีกครั้งก่อน job จะรันจริง job จะได้ข้อมูล
ล่าสุดเสมอ ไม่ใช่ข้อมูลเก่าตอนที่ enqueue

**ทดสอบจริง:**

```
ทดสอบ SearchIndexJob.perform_later สำหรับ post id=9
enqueued! (adapter ปัจจุบัน: async)
หลัง job รัน: title ใน index = สอนทำต้มยำกุ้งฉบับร้านอาหารไทย
```

ใน production ควรใช้ **Solid Queue** (ค่า default ของ Rails 8 จาก Part 061) หรือ **Sidekiq**
(Part 062) เป็น adapter จริง แทน `:async` ที่ใช้แค่ตอนทดสอบ (งานหายเมื่อ process ปิด — ทบทวน
จาก Part 061 Step 605) และควรแยก queue ต่างหาก (`queue_as :search`) เพื่อไม่ให้งาน indexing
ไปแย่งคิวกับงานสำคัญกว่า เช่น ส่งอีเมล

### ต้นทุนด้านปฏิบัติการที่ต้องคิดก่อนตัดสินใจใช้ — พูดตรงๆ ไม่ต้องอ้อม

หลังผ่านทั้ง 10 Step มาถึงตรงนี้ ควรสรุปให้ตรงไปตรงมา เพราะ Elasticsearch/OpenSearch
**ไม่ใช่ของฟรี** ในแง่ความซับซ้อนที่เพิ่มเข้าระบบ:

**ภาระที่เพิ่มขึ้นจริง เมื่อเทียบกับใช้แค่ pg_search:**

1. **ต้องรันและดูแลระบบแยกอีกชุด** — ต้อง monitor สุขภาพ cluster, จัดการ disk space (จำได้
   ไหมที่เจอ error `index_create_block_exception` ตอนดิสก์ตึงมือใน Step 683 — เรื่องแบบนี้
   เกิดขึ้นจริงใน production ได้เช่นกัน), วางแผน backup/snapshot ของ index (Elasticsearch/
   OpenSearch มี snapshot API ของตัวเอง แยกจาก `pg_dump`)
2. **ใช้ RAM มาก** — ยืนยันจากตัวเลขจริงใน Step 683: node เดียว heap แค่ 512MB ก็ยังกิน RAM
   จริงเกือบ 1GB ระบบ production ที่ต้องการ high availability (หลาย node, มี replica กันข้อมูล
   หาย) ใช้ทรัพยากรมากกว่านี้หลายเท่า
3. **Index drift ต้องจัดการเชิงรุก** — พิสูจน์แล้วข้างบนว่าปัญหานี้เกิดขึ้นได้ง่ายกว่าที่คิด
   ต้องมีกลยุทธ์ reindex ที่ชัดเจน ไม่ใช่แค่ "เขียน callback แล้วจบ"
4. **Deployment ซับซ้อนขึ้น** — ต้องมี Elasticsearch/OpenSearch instance ในทุก environment
   (dev, staging, production), ต้องคิดเรื่อง security (auth, TLS ที่ปิดไว้ตอน dev ต้องเปิด
   จริงจังใน production), ต้องอัปเกรดเวอร์ชันเป็นระยะ (ซึ่งมีเรื่อง breaking change ระหว่าง
   major version อย่างที่เจอจริงใน Step 684)
5. **ทีมต้องเรียนรู้ mental model ใหม่** — query DSL, mapping, analyzer, aggregation
   เป็นแนวคิดที่ต่างจาก SQL โดยสิ้นเชิง เพิ่ม cognitive load ให้ทีม

**เมื่อไหร่ pg_search/pg_trgm "พอแล้ว" ไม่ต้องใช้ Elasticsearch/OpenSearch:**

- ข้อมูลที่ต้องค้นหามีขนาดเล็กถึงปานกลาง (หลักหมื่นถึงหลักแสน record) — PostgreSQL GIN index
  จัดการได้เร็วพอในระดับนี้
- ไม่ต้องการ faceted search/aggregation ที่ซับซ้อน (แค่ "หาคำ เจอ/ไม่เจอ" ก็พอ)
- ทีมมี PostgreSQL อยู่แล้วและไม่อยากเพิ่มระบบใหม่เข้ามาดูแล (โดยเฉพาะทีมเล็กที่ไม่มี
  DevOps เฉพาะทาง)
- ความต้องการด้านภาษา ถ้าเป็นภาษาอังกฤษล้วนหรือภาษาที่ PostgreSQL text search config
  รองรับดีอยู่แล้ว ปัญหาการตัดคำจะไม่รุนแรงเท่าภาษาไทย

**เมื่อไหร่ควรลงทุนกับ Elasticsearch/OpenSearch:**

- ข้อมูลระดับล้าน record ขึ้นไป และการค้นหาเป็น core feature ของผลิตภัณฑ์ (เช่น เว็บอีคอมเมิร์ซ
  ขนาดใหญ่, marketplace, job board)
- ต้องการ faceted search/aggregation ที่ผู้ใช้โต้ตอบด้วยจริงจัง (filter หลายมิติพร้อมกัน)
- รองรับหลายภาษาที่การตัดคำซับซ้อน (ไทย, จีน, ญี่ปุ่น, อารบิก) และคุณภาพผลค้นหาส่งผลต่อธุรกิจ
  โดยตรง (เช่น การหาสินค้าไม่เจอ = เสียยอดขาย)
- มีทีม/งบประมาณพอจะดูแลระบบเพิ่มเติมได้จริง (หรือใช้ managed service อย่าง Amazon
  OpenSearch Service / Elastic Cloud เพื่อลดภาระ ops บางส่วน)

Part 070 (โปรเจกต์ปิด Phase 10) จะให้เห็นภาพระบบค้นหาสินค้าที่ใช้งานจริงแบบครบวงจร ซึ่งจะ
ช่วยตัดสินใจได้ชัดเจนขึ้นว่ากรณีใช้งานแบบไหนเหมาะกับเครื่องมือแบบไหน

---

## แบบฝึกหัด: สร้างระบบค้นหาโพสต์ภาษาไทยด้วย OpenSearch พร้อม Facet

> **สถานะการทดสอบ:** เฉลยทั้งหมดด้านล่างนี้ **ทดสอบจริง** กับ OpenSearch 2.19.6 ที่รันอยู่ใน
> แซนด์บ็อกซ์ผ่าน Docker (ตามที่อธิบายไว้ต้น Part) และแอป Rails 8.1.4 สาธิตที่สร้างแยกไว้ใน
> scratch directory ผลลัพธ์ทุกก้อนที่แสดงคือ output จริงจากการรัน ไม่ใช่ค่าที่คาดเดา

### โจทย์

สร้างระบบค้นหาบทความ (Post) ที่มี `title`, `body`, `category` เป็นภาษาไทย โดยต้อง:

1. Index ข้อมูลเข้า OpenSearch อัตโนมัติเมื่อสร้าง/แก้ไข Post (ผ่าน callback)
2. รองรับการ bulk reindex ทั้งหมดด้วย `Post.import`
3. ค้นหาได้ด้วย `Post.search("คำค้นหา").records` พร้อม relevance score และตัดคำภาษาไทยถูกต้อง
4. คืนค่า facet จำนวนโพสต์ต่อหมวดหมู่มาพร้อมกับผลค้นหาในคำสั่งเดียว

### เฉลย

**1. Migration**

```ruby
# db/migrate/xxxxxx_create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body
      t.string :category
      t.datetime :published_at
      t.timestamps
    end
  end
end
```

**2. Gemfile**

```ruby
gem "opensearch-ruby", "~> 3.4"
```

**3. Searchable concern (เวอร์ชันสมบูรณ์ รวมทุก Step ที่สอนมา)**

```ruby
# app/models/concerns/searchable.rb
# frozen_string_literal: true

require "opensearch"

module Searchable
  extend ActiveSupport::Concern

  included do
    after_commit :index_to_search_engine, on: [:create, :update]
    after_commit :remove_from_search_engine, on: :destroy
  end

  class_methods do
    def search_client
      @search_client ||= OpenSearch::Client.new(
        host: ENV.fetch("OPENSEARCH_URL", "http://localhost:9200"),
        log: false
      )
    end

    def search_index_name
      "#{name.underscore.pluralize}_#{Rails.env}"
    end

    def create_search_index!
      return if search_client.indices.exists(index: search_index_name)

      search_client.indices.create(
        index: search_index_name,
        body: {
          settings: { number_of_shards: 1, number_of_replicas: 0 },
          mappings: { properties: search_mapping }
        }
      )
    end

    def import
      create_search_index!
      find_in_batches do |batch|
        bulk_body = batch.flat_map do |record|
          [{ index: { _index: search_index_name, _id: record.id } }, record.as_indexed_json]
        end
        search_client.bulk(body: bulk_body) if bulk_body.any?
      end
      search_client.indices.refresh(index: search_index_name)
    end

    def search(query, category: nil)
      filter = category.present? ? [{ term: { category: category } }] : []

      response = search_client.search(
        index: search_index_name,
        body: {
          query: {
            bool: {
              must: { multi_match: { query: query, fields: ["title^2", "body"] } },
              filter: filter
            }
          },
          aggs: { posts_per_category: { terms: { field: "category" } } }
        }
      )
      SearchResult.new(self, response)
    end
  end

  module ClassMethods
    def search_mapping
      {
        title: { type: "text", analyzer: "thai" },
        body: { type: "text", analyzer: "thai" },
        category: { type: "keyword" },
        created_at: { type: "date" }
      }
    end
  end

  def as_indexed_json
    { title: title, body: body, category: category, created_at: created_at }
  end

  def index_to_search_engine
    self.class.create_search_index!
    self.class.search_client.index(
      index: self.class.search_index_name, id: id, body: as_indexed_json, refresh: true
    )
  end

  def remove_from_search_engine
    self.class.search_client.delete(index: self.class.search_index_name, id: id, refresh: true)
  rescue OpenSearch::Transport::Transport::Errors::NotFound
    nil
  end

  class SearchResult
    include Enumerable

    def initialize(model_class, response)
      @model_class = model_class
      @response = response
    end

    def hits = @response.dig("hits", "hits") || []

    def records
      ids = hits.map { |h| h["_id"] }
      scores = hits.each_with_object({}) { |h, acc| acc[h["_id"].to_i] = h["_score"] }
      @model_class.where(id: ids)
                  .sort_by { |r| ids.index(r.id.to_s) }
                  .each { |r| r.define_singleton_method(:search_score) { scores[r.id] } }
    end

    def category_facets
      buckets = @response.dig("aggregations", "posts_per_category", "buckets") || []
      buckets.each_with_object({}) { |b, acc| acc[b["key"]] = b["doc_count"] }
    end

    def each(&block) = records.each(&block)
  end
end
```

**4. Model**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Searchable
end
```

**5. ทดสอบด้วยข้อมูลตัวอย่างจริง**

```ruby
# bin/rails runner
seed = [
  { title: "รีวิวร้านกาแฟใจกลางเมืองบรรยากาศดี", body: "ร้านกาแฟแห่งนี้ตกแต่งสไตล์มินิมอล เหมาะกับการนั่งทำงานทั้งวัน", category: "อาหาร" },
  { title: "สูตรลับต้มยำกุ้งน้ำข้นแบบร้านดัง", body: "เคล็ดลับความเข้มข้นอยู่ที่การเคี่ยวน้ำซุปกุ้งนานเป็นชั่วโมง", category: "อาหาร" },
  { title: "10 คาเฟ่ต้องไปในกรุงเทพมหานคร", body: "รวมร้านกาแฟที่คนกรุงเทพต้องมาลองสักครั้งในชีวิต", category: "อาหาร" },
  { title: "เจาะลึกการทำงานของ Elasticsearch", body: "อธิบายสถาปัตยกรรมแบบ distributed search engine", category: "เทคโนโลยี" },
  { title: "OpenSearch ต่างจาก Elasticsearch อย่างไร", body: "ประวัติการแยกโปรเจกต์ในปี 2021", category: "เทคโนโลยี" },
  { title: "เที่ยวเชียงใหม่หน้าหนาวที่ไหนดี", body: "รวมสถานที่เที่ยวธรรมชาติและคาเฟ่วิวดอย", category: "ท่องเที่ยว" },
  { title: "แนะนำที่พักริมทะเลเดินทางง่ายจากกรุงเทพ", body: "ที่พักติดหาดเดินทางจากกรุงเทพไม่เกินสามชั่วโมง", category: "ท่องเที่ยว" },
  { title: "AI ช่วยแนะนำเมนูอาหารตามอากาศได้อย่างไร", body: "ระบบแนะนำเมนูอัตโนมัติที่ร้านอาหารเริ่มนำมาใช้จริง", category: "เทคโนโลยี" }
]
seed.each { |a| Post.create!(a) }

result = Post.search("ร้านกาแฟ")
result.records.each { |p| puts "score=%.3f  %s" % [p.search_score, p.title] }
result.category_facets.each { |k, v| puts "#{k}: #{v}" }
```

**ผลลัพธ์จริงที่ได้ (verified live):**

```
--- ค้นหา 'ร้านกาแฟ' ---
score=5.102  รีวิวร้านกาแฟใจกลางเมืองบรรยากาศดี
score=2.505  10 คาเฟ่ต้องไปในกรุงเทพมหานคร
score=2.398  สูตรลับต้มยำกุ้งน้ำข้นแบบร้านดัง
score=0.892  AI ช่วยแนะนำเมนูอาหารตามอากาศได้อย่างไร

--- facet หมวดหมู่จากผลค้นหาข้างต้น ---
อาหาร: 3
เทคโนโลยี: 1
```

**คำอธิบายผลลัพธ์:** โพสต์แรกได้คะแนนสูงสุดเพราะมีคำว่า "ร้าน" และ "กาแฟ" อยู่ใน title
(น้ำหนัก `^2`) ส่วน "10 คาเฟ่ต้องไปในกรุงเทพมหานคร" ติดอันดับสองแม้ title จะไม่มีคำว่า
"ร้านกาแฟ" ตรงๆ เพราะ **body** ของมันมีประโยค "รวม**ร้านกาแฟ**ที่คนกรุงเทพ..." อยู่จริง — ส่วน
"สูตรลับต้มยำกุ้ง..." กับ "AI ช่วยแนะนำเมนู..." ติดมาด้วยคะแนนต่ำกว่าเพราะมีแค่คำว่า "ร้าน"
คำเดียวที่ match (multi_match แบบ default จะ match ได้แม้เจอแค่บางคำใน query ไม่จำเป็นต้อง
ครบทุกคำ) นี่คือพฤติกรรม relevance-based ranking ที่แตกต่างจาก SQL `LIKE` ที่ตอบแค่ "เจอ/ไม่
เจอ" โดยไม่มีลำดับความเกี่ยวข้อง

facet แสดง "อาหาร: 3, เทคโนโลยี: 1" เฉพาะจากโพสต์ที่ปรากฏในผลค้นหานี้เท่านั้น (ตามที่อธิบาย
ไว้ใน Step 689) หมวด "ท่องเที่ยว" ไม่ปรากฏเพราะไม่มีโพสต์หมวดนั้น match คำค้นหา "ร้านกาแฟ"
เลย

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม pagination ให้ `Post.search` โดยรับ parameter `page:` และ `per_page:` แล้วแปลงเป็น
   `from`/`size` ใน query body ของ OpenSearch (ใบ้: `from = (page - 1) * per_page`) พร้อมคืน
   `total_pages` จาก `response.dig("hits", "total", "value")`
2. เพิ่ม highlight ให้ผลการค้นหา โดยแก้ `search` method ให้ใส่ `highlight: { fields: { title:
   {}, body: {} } }` เข้าไปใน query body แล้วเพิ่ม method `highlighted_snippet` ใน
   `SearchResult` ที่ดึงข้อความจาก `hit["highlight"]` ออกมาแสดงแทน `body` เต็มๆ (อ้างอิงผลลัพธ์
   จริงที่เห็นใน Step 688 เป็นแนวทาง)
3. เขียน Rake task ชื่อ `search:reindex_all` ที่วิ่ง `Post.import` (และโมเดลอื่นที่ include
   `Searchable` ถ้ามี) พร้อม progress output (เช่น `puts "Reindexing #{model}... done
   (#{count} records)"`) สำหรับรันตอน deploy ครั้งแรกหรือหลังเปลี่ยน mapping

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า Elasticsearch/OpenSearch เป็น **distributed search engine** ที่สร้างบน inverted
  index ต่างจากฐานข้อมูลเชิงสัมพันธ์โดยพื้นฐาน (document/index แทน table/row)
- รู้ประวัติและความแตกต่างของ **Elasticsearch vs OpenSearch** (การแยกโปรเจกต์ปี 2021 จาก
  ประเด็น license) และรู้ว่า REST API พื้นฐานของทั้งคู่เข้ากันได้เกือบสมบูรณ์
- รันอินสแตนซ์ OpenSearch/Elasticsearch จริงด้วย Docker ได้ และรู้จักปัญหาที่พบได้จริง
  (disk watermark block, ปัญหาการดาวน์โหลดผ่าน network policy ที่เข้มงวด)
- เลือก gem Ruby ให้ถูกกับเอนจิน (`opensearch-ruby` สำหรับ OpenSearch,
  `elasticsearch-model`/`elasticsearch-rails` ที่ต้อง pin เวอร์ชันให้ตรงสำหรับ Elasticsearch)
  และเข้าใจกลไก product check ที่ทำให้ผสมข้ามกันไม่ได้
- กำหนด **index mapping** ให้ถูกต้องตั้งแต่ต้น โดยเฉพาะการเลือกระหว่าง `text` กับ `keyword`
- แก้ปัญหา **การตัดคำภาษาไทย** ที่ Part 068 ทิ้งไว้ได้จริงด้วย built-in **Thai analyzer**
  ของ Lucene core (ไม่ต้องติดตั้ง plugin เพิ่ม) พิสูจน์ด้วยผลลัพธ์จริงทั้งการตัดคำและการค้นหา
- Index ข้อมูลจาก ActiveRecord model ได้ทั้งแบบ callback (`after_commit`) และ bulk
  (`Post.import`) พร้อมค้นหาแบบมี **relevance score**, **fuzzy matching**, **highlighting**
- ทำ **faceted search / aggregations** นับจำนวนต่อหมวดหมู่พร้อมผลค้นหาในคำสั่งเดียว —
  สิ่งที่ pg_search ทำได้ยาก
- เข้าใจปัญหา **index drift** จริง วิธีป้องกันด้วย reindex strategy และการทำ indexing
  แบบ asynchronous ผ่าน ActiveJob (เชื่อมกับ Part 061)
- ประเมินได้อย่างตรงไปตรงมาว่าเมื่อไหร่ควรลงทุนกับ Elasticsearch/OpenSearch และเมื่อไหร่
  pg_search ยังเพียงพอ โดยพิจารณาจากต้นทุนด้านปฏิบัติการที่เพิ่มขึ้นจริง

**ต่อไป (Part 070 — โปรเจกต์ปิด Phase 10):** เราจะรวมทุกอย่างที่เรียนมาตลอด Phase 10
(Active Storage จาก Part 066–067, pg_search จาก Part 068, และ Elasticsearch/OpenSearch จาก
Part นี้) มาสร้าง **ระบบอัปโหลดรูปสินค้า + ค้นหาสินค้า** แบบครบวงจรเป็นโปรเจกต์จริง ตั้งแต่
อัปโหลดรูปหลายรูปต่อสินค้า, ปรับขนาดรูปอัตโนมัติ, ไปจนถึงหน้าค้นหาสินค้าที่มีทั้ง filter,
facet ตามหมวดหมู่/ช่วงราคา, และการเรียงตาม relevance — เป็นจุดที่จะเห็นภาพว่าเครื่องมือ
ทั้งหมดที่เรียนมาใน Phase นี้ทำงานร่วมกันอย่างไรในระบบจริงชิ้นเดียว
