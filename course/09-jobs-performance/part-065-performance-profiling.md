# Part 065: Performance Profiling — rack-mini-profiler, Memory Profiling และปิด Phase 9 ด้วยชุดเครื่องมือวัดประสิทธิภาพครบวงจร

> **Step ครอบคลุมใน Part นี้:** Step 641–650
> **ระดับ:** สูง (ต้องผ่าน Part 061 ActiveJob, Part 062 Sidekiq, Part 063 Caching, และ Part 064
> Database Performance/N+1 มาก่อน — Part นี้ไม่สอนเทคนิคแก้ปัญหาซ้ำ แต่จะสอน**เครื่องมือวัดผล**
> ที่ทำให้รู้ว่าควรหยิบเทคนิคไหนจาก 4 Part ที่ผ่านมาไปใช้ ณ จุดไหน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6, Rails 8.1.4, `rack-mini-profiler` 3.3.1, `stackprof` 0.2.28,
> `memory_profiler` 1.1.0, `benchmark-ips` 2.15.1, `derailed_benchmarks` 2.2.1 — ทุกคำสั่งและ
> ตัวเลขในเอกสารนี้รันจริงบนแอป Rails ทดลองที่สร้างขึ้นเฉพาะสำหรับ Part นี้ ไม่มีตัวเลขที่แต่งขึ้นเอง

Part นี้คือ **Part สุดท้ายของ Phase 9: Background Jobs & Performance** ตลอด 4 Part ที่ผ่านมา
เราเรียนรู้ *เทคนิคการแก้ปัญหา* ประสิทธิภาพไปทีละด้าน — ย้ายงานหนักไปทำเบื้องหลังด้วย ActiveJob
(Part 061) และ Sidekiq (Part 062), ลดภาระฐานข้อมูลด้วย caching (Part 063), และแก้ query ที่ช้า
ด้วย index กับการกำจัด N+1 (Part 064) — แต่คำถามที่ยังไม่มีคำตอบตลอดมาคือ **"แล้วเรารู้ได้อย่างไร
ว่าต้องใช้เทคนิคไหน ที่จุดไหนของโค้ด"** Part นี้คือคำตอบ: เราจะเรียนรู้ชุดเครื่องมือ **profiling**
ที่ทำให้เห็นตัวเลขจริงว่าเวลาและหน่วยความจำของ request หนึ่งครั้งถูกใช้ไปกับอะไรบ้าง ก่อนตัดสินใจ
ลงมือแก้ไขอะไรเลยสักบรรทัด — `rack-mini-profiler` สำหรับดู breakdown ของแต่ละ request แบบสด,
`memory_profiler` และ `derailed_benchmarks` สำหรับตามรอยการใช้หน่วยความจำ, `benchmark-ips`
สำหรับเปรียบเทียบโค้ดสองแบบอย่างมีนัยสำคัญทางสถิติ และ `ActiveSupport::Notifications` ที่เป็น
กลไกเบื้องหลังที่เครื่องมือเหล่านี้เกือบทั้งหมดใช้ร่วมกัน ปิดท้ายด้วยการนำทุกเครื่องมือมาประกอบกัน
ในแบบฝึกหัดจริง: เจอหน้าเว็บช้า → profile → หาสาเหตุ → แก้ → profile ซ้ำเพื่อยืนยันว่าดีขึ้นจริง

## สารบัญของ Part นี้

- Step 641: แนวคิด "วัดก่อน อย่าเดา" — Premature Optimization และภาพรวมเครื่องมือใน Part นี้
- Step 642: ติดตั้ง `rack-mini-profiler` + `stackprof` และทำความรู้จัก badge วัดความเร็ว
- Step 643: อ่านผล mini-profiler แบบละเอียด — นับ query, จับเวลา view, ขุดหา query ที่ซ้ำ
- Step 644: Flamegraph mode (`?pp=flamegraph`) — CPU profiling หา method ที่กินเวลาที่สุด
- Step 645: แก้ N+1 ที่เจอ แล้ว profile ซ้ำ — วัดผลต่างก่อน/หลังด้วยตัวเลขจริง
- Step 646: `memory_profiler` — เทียบการจอง object ระหว่างโค้ด naive กับโค้ดที่ให้ DB ทำงานแทน
- Step 647: `ActiveSupport::Notifications` — กลไกเบื้องหลังที่ทำให้ profiler ทุกตัวทำงานได้
- Step 648: `benchmark-ips` — เปรียบเทียบความเร็วโค้ดอย่างมีนัยสำคัญทางสถิติ ไม่ใช่แค่ `Time.now`
- Step 649: `derailed_benchmarks` — หา memory bloat ตอน boot และ memory leak ระหว่างรัน + checklist
  ปัญหาประสิทธิภาพที่พบบ่อยที่สุดใน Rails
- Step 650: เมื่อไหร่ต้องใช้ APM ระดับ production + แบบฝึกหัด: profile หน้าเว็บช้าจริง แก้ไข
  แล้ววัดผลต่าง + ปิด Phase 9

---

## Step 641: แนวคิด "วัดก่อน อย่าเดา" — Premature Optimization และภาพรวมเครื่องมือใน Part นี้

### "Premature optimization is the root of all evil"

ประโยคนี้มาจาก Donald Knuth หนึ่งในบิดาแห่งวิทยาการคอมพิวเตอร์ ความหมายที่แท้จริงของประโยคนี้
**ไม่ใช่** "อย่าสนใจประสิทธิภาพเลย" แต่คือ **"อย่าเสียเวลาไปปรับแต่งจุดที่คุณ*เดา*ว่าช้า ก่อนที่จะมี
ข้อมูลจริงยืนยันว่ามันช้าจริง"** — เพราะสัญชาตญาณของโปรแกรมเมอร์เรื่อง "อะไรช้า" มักผิดพลาดบ่อยมาก
กว่าที่คิด

ตัวอย่างที่จะเห็นตลอด Part นี้: ถ้าคุณเปิดหน้าเว็บหนึ่งแล้วรู้สึกว่าช้า สัญชาตญาณทั่วไปมักชี้ไปที่
"ERB template คงเขียนวนลูปแย่ๆ" หรือ "คงต้องแต่ง CSS/JS ให้เบาลง" แต่พอวัดจริงด้วยเครื่องมือใน
Part นี้ กลับพบว่าเวลาเกือบทั้งหมดหายไปกับการยิง SQL query ซ้ำๆ กันหลักพันครั้งโดยที่ template
เขียนถูกต้องตามหลักการทุกอย่าง — ถ้าไม่มีเครื่องมือวัด คุณอาจเสียเวลาไปปรับแต่ง template อยู่นาน
โดยไม่แตะจุดที่แท้จริงเลยสักครั้ง

**หลักการทำงานที่ถูกต้องเสมอ คือวงจร 4 ขั้นตอนนี้:**

1. **วัด (Measure)** — ใช้เครื่องมือ profiling หาตัวเลขจริงว่าเวลา/หน่วยความจำหายไปกับอะไร
2. **ระบุจุดที่แพงที่สุด (Identify the bottleneck)** — จากตัวเลข ไม่ใช่จากความรู้สึก
3. **แก้เฉพาะจุดนั้น (Fix)** — เลือกเทคนิคที่ตรงกับปัญหา (index? cache? background job? eager
   loading?) จากคลังเทคนิคที่เรียนมาใน Part 061–064
4. **วัดซ้ำ (Re-measure)** — ยืนยันว่าตัวเลขดีขึ้นจริง ไม่ใช่แค่ "รู้สึกว่าเร็วขึ้น"

Part นี้ทั้ง Part คือการฝึกวงจรนี้ให้เป็นนิสัย

### ภาพรวมเครื่องมือที่จะได้ใช้ใน Part นี้

| เครื่องมือ | ใช้ตอบคำถาม | ระดับ |
|---|---|---|
| `rack-mini-profiler` | request หนึ่งครั้งใช้เวลาไปกับอะไรบ้าง (SQL/view/ทั้งหมด) | ระดับ request |
| `rack-mini-profiler` flamegraph | ภายใน request นั้น เวลา CPU ไปกระจุกอยู่ที่ method ไหน | ระดับ CPU/method |
| `memory_profiler` | โค้ดช่วงหนึ่งจอง object ใน RAM กี่ตัว กี่ byte | ระดับ code block |
| `ActiveSupport::Notifications` | กลไกที่ tool ข้างบนใช้วัดผลจริงๆ เบื้องหลัง | ระดับ instrumentation |
| `benchmark-ips` | โค้ด A กับ B แบบไหนเร็วกว่ากันจริง (ไม่ใช่แค่รันครั้งเดียว) | ระดับเปรียบเทียบ |
| `derailed_benchmarks` | แอปทั้งแอปกิน RAM ตอน boot เท่าไหร่ / โตขึ้นระหว่างรันไหม | ระดับทั้งแอป |
| APM (Skylight ฯลฯ) | traffic จริงจากผู้ใช้จริงใน production เป็นอย่างไร | ระดับ production |

### แอปทดลองที่จะใช้ตลอด Part นี้

เพื่อให้ทุกตัวเลขในเอกสารนี้เป็นของจริงไม่ใช่ค่าสมมติ เราสร้างแอป Rails ทดลองขึ้นมาหนึ่งตัว (ไม่รวม
อยู่ในโค้ดหลักสูตร เป็นแค่ sandbox สำหรับสาธิต) มีโครงสร้างง่ายๆ:

```ruby
# app/models/author.rb
class Author < ApplicationRecord
  has_many :books
end

# app/models/book.rb
class Book < ApplicationRecord
  belongs_to :author
end
```

พร้อมข้อมูลจำลอง **300 authors และ 2,234 books** (เฉลี่ยคนละ 5–10 เล่ม) — ปริมาณข้อมูลระดับนี้
เพียงพอที่จะทำให้ปัญหา N+1 แสดงผลกระทบด้านเวลาให้เห็นชัดเจนเมื่อ profile จริง (สังเกตว่านี่คือ
สถานการณ์เดียวกับที่ Part 064 ใช้สอนเรื่อง Bullet gem — Part นี้จะใช้ `rack-mini-profiler` มอง
ปัญหาเดียวกันจากอีกมุมหนึ่ง คือมุม "เวลาที่ผู้ใช้จริงต้องรอ" แทนมุม "จำนวน query ที่ Bullet เตือน")

---

## Step 642: ติดตั้ง `rack-mini-profiler` + `stackprof` และทำความรู้จัก badge วัดความเร็ว

### ติดตั้ง

`rack-mini-profiler` เป็น Rack middleware ที่แทรกตัวเองเข้าไปในทุก request ของแอป แล้ว "แปะ"
ตัวเลขเวลาที่ใช้ไว้เป็น badge ลอยอยู่มุมซ้ายบนของหน้าเว็บทุกหน้า **ควรติดตั้งไว้ใน group
`:development` เท่านั้น** — ไม่ใช่เครื่องมือที่ควรเปิดทิ้งไว้ใน production เพราะแอบเพิ่ม overhead
ให้ทุก request จริง (การวัดผล production ที่ปลอดภัยกว่าคือ APM ใน Step 650)

```ruby
# Gemfile
group :development do
  gem "rack-mini-profiler", "~> 3.3"
  gem "stackprof" # จำเป็นสำหรับโหมด flamegraph (Step 644) — ไม่มีตัวนี้ flamegraph จะใช้ไม่ได้
end
```

```bash
bundle install
```

แค่นี้เสร็จแล้ว — `rack-mini-profiler` มาพร้อม Railtie ที่ hook ตัวเองเข้ากับ middleware stack
ของ Rails โดยอัตโนมัติทันทีที่ gem อยู่ใน Gemfile ของ environment ปัจจุบัน **ไม่ต้องเขียน
initializer เพิ่มเพื่อให้มันเริ่มทำงาน** (initializer จะใช้ก็ต่อเมื่อต้องการปรับแต่งพฤติกรรม
ขั้นสูง เช่น การเปลี่ยน storage backend สำหรับเก็บผล profiling)

### รันเซิร์ฟเวอร์แล้วดูสิ่งที่เกิดขึ้น

```bash
bin/rails server
```

เปิดเบราว์เซอร์ไปที่หน้าที่มีปัญหา (ในตัวอย่างนี้คือ `/books` ซึ่งจงใจเขียนให้มี N+1):

```ruby
# app/controllers/books_controller.rb — เวอร์ชันแรกที่ "ดูปกติ" แต่มีปัญหาซ่อนอยู่
class BooksController < ApplicationController
  def index
    @books = Book.all
  end
end
```

```erb
<%# app/views/books/index.html.erb %>
<h1>รายการหนังสือทั้งหมด</h1>
<ul>
  <% @books.each do |book| %>
    <li><%= book.title %> โดย <%= book.author.name %> (<%= book.published_year %>)</li>
  <% end %>
</ul>
```

เมื่อเปิดหน้านี้ในเบราว์เซอร์จริง จะเห็น **badge สีเหลี่ยมเล็กๆ ลอยอยู่มุมซ้ายบน** แสดงตัวเลข
มิลลิวินาทีรวมของ request นั้น (เช่น `2765.2ms (2236 sql)`) กดที่ badge จะขยายเป็นรายละเอียด
เต็มรูปแบบ — เนื้อหาที่เห็นบน badge นี้ไม่ใช่เวทมนตร์ มันคือข้อมูลเดียวกันเป๊ะๆ กับที่เราจะดึงออกมา
ตรวจสอบด้วย `curl` ต่อไปนี้ (มีประโยชน์มากตอน debug ผ่าน terminal หรือใน CI ที่ไม่มีเบราว์เซอร์)

### ดูข้อมูลเบื้องหลัง badge ผ่าน HTTP header

ทุก response ที่ผ่าน `rack-mini-profiler` จะแนบ header พิเศษมาด้วยเสมอ:

```bash
curl -s -i http://127.0.0.1:3000/books | head -20
```

ผลลัพธ์จริงที่ได้ (ตัดเฉพาะ header ที่เกี่ยวข้อง):

```
HTTP/1.1 200 OK
x-runtime: 2.591674
server-timing: sql.active_record;dur=967.27, start_processing.action_controller;dur=0.08,
  instantiation.active_record;dur=99.62, render_template.action_view;dur=2480.03,
  render_layout.action_view;dur=2483.66, process_action.action_controller;dur=2513.86
x-miniprofiler-original-cache-control: max-age=0, private, must-revalidate
x-miniprofiler-ids: 8hep4o4yid328csm7qyz
content-length: 384491
```

**อธิบาย:**

- `x-runtime: 2.591674` คือเวลาทั้งหมดที่ Rails ใช้ตอบ request นี้ (หน่วยวินาที) — **2.59 วินาที
  สำหรับหน้าที่แสดงหนังสือแค่ 2,234 เล่ม ถือว่าช้ามาก** นี่คือสัญญาณแรกที่บอกว่าต้อง profile ต่อ
- `server-timing` เป็น header มาตรฐานของเว็บ (ไม่ใช่ของ mini-profiler โดยเฉพาะ — DevTools ของ
  เบราว์เซอร์ทุกตัวอ่าน header นี้ได้และแสดงเป็นกราฟใน tab Network ให้อัตโนมัติ) สังเกตว่า
  `render_template.action_view;dur=2480.03` คือเกือบทั้งหมดของเวลาทั้ง request — **ดูเผินๆ
  เหมือนปัญหาอยู่ที่การ render view** แต่ Step 643 จะแสดงให้เห็นว่าจริงๆ แล้วปัญหาซ่อนอยู่ *ข้างใน*
  ขั้นตอน render นั้นเอง ไม่ใช่ที่ตัว template
- `x-miniprofiler-ids: 8hep4o4yid328csm7qyz` คือ **id ของผลการวัดครั้งนี้** — เก็บไว้ในหน่วยความจำ
  ของ mini-profiler เอง (หรือ Redis/Memcached ถ้าตั้งค่า storage แบบนั้น) ใช้ id นี้ดึงรายละเอียด
  แบบเต็มออกมาดูได้ในขั้นตอนถัดไป

> **หมายเหตุเรื่องความปลอดภัย:** โดย default `rack-mini-profiler` เปิดทำงานให้ทุก request ใน
> development เท่านั้น การตั้งค่าให้เปิดใน production (`config.enabled = true` แบบไม่มีเงื่อนไข)
> ต้องระวังมาก เพราะ endpoint `/mini-profiler-resources/*` ที่จะเห็นใน Step 643 เปิดเผยรายละเอียด
> ของ query และ path ไฟล์ในเครื่องออกมา ควรจำกัดด้วย `config.authorization_mode = :whitelist`
> ร่วมกับ IP whitelist เสมอถ้าจำเป็นต้องเปิดใน production

---

## Step 643: อ่านผล mini-profiler แบบละเอียด — นับ query, จับเวลา view, ขุดหา query ที่ซ้ำ

### ดึงผลแบบเต็มออกมาเป็น JSON

`rack-mini-profiler` เก็บผลการวัดของทุก request ไว้ และมี endpoint สำหรับดึงออกมาดูแบบเต็ม —
เวลาที่เบราว์เซอร์คลิกขยาย badge มันก็เรียก endpoint เดียวกันนี้ผ่าน JavaScript อยู่เบื้องหลัง เรา
เรียกตรงๆ ด้วย `curl` ได้เช่นกัน โดยใช้ `id` จาก header `x-miniprofiler-ids` ใน Step 642 และแนบ
header `X-Requested-With: XMLHttpRequest` เพื่อบอกว่าอยากได้ JSON แทน HTML:

```bash
curl -s -H "X-Requested-With: XMLHttpRequest" \
  "http://127.0.0.1:3000/mini-profiler-resources/results?id=8hep4o4yid328csm7qyz" \
  | python3 -m json.tool | head -30
```

ผลลัพธ์จริง (ตัดส่วนสำคัญ):

```json
{
  "name": "/books",
  "duration_milliseconds": 2765.195555999526,
  "sql_count": 2236,
  "duration_milliseconds_in_sql": 137.87681188841816,
  "request_path": "/books"
}
```

**นี่คือตัวเลขที่บอกทุกอย่าง:**

- **`sql_count: 2236`** — request เดียวยิง SQL query ไปทั้งหมด **2,236 ครั้ง** สำหรับหน้าที่ควรมี
  แค่ 2 query (ดึงหนังสือ 1 ครั้ง + ดึงคนเขียน 1 ครั้งถ้าทำถูกวิธี) เลข 2,236 นี้เกือบเท่ากับ
  จำนวนหนังสือ 2,234 เล่มในระบบพอดี — **เป็นลายเซ็นคลาสสิกของปัญหา N+1** (ทบทวนจาก Part 064)
- **`duration_milliseconds_in_sql: 137.9`** — เวลาที่ SQL ทั้งหมดรวมกันใช้จริงๆ มีแค่ ~138ms
  จาก 2,765ms ทั้งหมด (แค่ 5%) — **นี่คือจุดที่คนมักเข้าใจผิด** ถ้าดูแค่ "เวลารวมของ SQL" จะคิดว่า
  ฐานข้อมูลไม่ใช่ปัญหา แต่ปัญหาจริงคือ**จำนวนครั้ง**ของการ round-trip ไปกลับระหว่าง Ruby process
  กับ database ที่มากถึง 2,236 ครั้ง แต่ละครั้งมี overhead ของตัวเอง (เปิด/ปิด statement, TCP/socket
  round-trip แม้เป็น SQLite ในเครื่องเดียวกันก็ยังมี overhead ของ context switch) ที่สะสมกันจนกิน
  เวลาไปมาก — ตัวเลข "เวลารวม" ของ query จึงบอกได้ไม่ครบ ต้องดู **จำนวนครั้ง** ควบคู่ไปด้วยเสมอ

### ขุดลึกลงไปดู query ที่ซ้ำกันจริงๆ

`page_struct` ที่ได้จาก endpoint นี้มีโครงสร้างเป็น timing tree ซ้อนกันตามลำดับที่โค้ดทำงาน
(controller action → render layout → render template) แต่ละ node เก็บ `sql_timings` ของ query
ที่เกิดขึ้น "ในช่วงเวลานั้น" ไว้ ลองดึงเฉพาะ node ของการ render view ออกมานับความถี่ของแต่ละ query:

```python
# วิเคราะห์ JSON ที่ได้จาก mini-profiler ด้วย Python (หรือจะเขียนเป็น Ruby ก็ได้เช่นกัน)
import json
from collections import Counter

d = json.load(open("mini_profiler_result.json"))
render_node = d["root"]["children"][0]["children"][0]  # node ของ "Rendering: books/index.html.erb"
sql = render_node["sql_timings"]

counts = Counter(s["formatted_command_string"][:60] for s in sql)
for query, count in counts.most_common(3):
    print(count, query)
```

ผลลัพธ์จริง:

```
2234  SELECT "authors".* FROM "authors" WHERE "authors"."id" = ?
   1  SELECT "books".* FROM "books" /*action='index'...*/
```

**ยืนยันชัดเจน 100%:** ระหว่างการ render view เพียงครั้งเดียว โค้ดยิง
`SELECT "authors".* FROM "authors" WHERE "authors"."id" = ?` ไปทั้งหมด **2,234 ครั้ง** — ตรงกับ
จำนวนหนังสือเป๊ะ เพราะทุกครั้งที่ ERB วนลูปเรียก `book.author.name` โดยที่ `@books` ไม่ได้ preload
ความสัมพันธ์ `author` ไว้ Rails จะยิง query ใหม่แยกไปทีละ `book` — **นี่คือหลักฐานที่จับต้องได้ของ
N+1 problem** ที่ Part 064 สอนวิธีแก้ไว้แล้ว (จะนำมาใช้จริงใน Step 645)

> **preview:** สังเกตว่าวิธีนี้ให้ข้อมูลแบบเดียวกับที่ **Bullet gem** (Part 064) แจ้งเตือนอัตโนมัติ
> ตอน request ทำงาน แต่ `rack-mini-profiler` ให้บริบทเพิ่มเติมว่า "เวลาที่เสียไปจริงๆ เท่าไหร่"
> ควบคู่กับ "SQL ซ้ำกี่ครั้ง" ทำให้ประเมินความรุนแรงของปัญหาได้แม่นยำกว่าการรู้แค่ว่า "มี N+1"

### สรุปฟิลด์สำคัญของผล mini-profiler

| ฟิลด์ | ความหมาย |
|---|---|
| `duration_milliseconds` | เวลารวมทั้ง request ตั้งแต่เข้า middleware จนถึงส่ง response กลับ |
| `sql_count` | จำนวนครั้งที่ยิง SQL query ทั้งหมดใน request นี้ |
| `duration_milliseconds_in_sql` | เวลารวมที่ SQL ทั้งหมดใช้ (ไม่รวม overhead อื่น) |
| `has_duplicate_sql_timings` | `true` ถ้ามี query ที่มี SQL text เดียวกันเกิดซ้ำ (สัญญาณ N+1) |
| `root.children[].sql_timings[]` | รายการ query แต่ละครั้ง พร้อม `formatted_command_string` และ `duration_milliseconds` |

---

## Step 644: Flamegraph mode (`?pp=flamegraph`) — CPU profiling หา method ที่กินเวลาที่สุด

`rack-mini-profiler` ทำได้มากกว่าการนับ SQL — มันมีโหมด **flamegraph** ที่ใช้ `stackprof`
(ที่ติดตั้งไว้ตั้งแต่ Step 642) sampling call stack ของโปรแกรมทุกๆ ช่วงเวลาสั้นๆ ระหว่างที่ request
กำลังทำงาน แล้วนำมาสร้างเป็นภาพ **flame graph** ที่แสดงว่า CPU ใช้เวลาไปกับ method ไหนมากที่สุด
— มีประโยชน์มากเมื่อปัญหา**ไม่ใช่** SQL แต่เป็นการคำนวณหนักฝั่ง Ruby เอง (เช่น loop ประมวลผลข้อมูล
ขนาดใหญ่, serialize JSON ก้อนโต, หรือ regex ที่ backtrack หนัก)

### เรียกใช้งาน

เพิ่ม query parameter `?pp=flamegraph` ต่อท้าย URL ที่ต้องการวิเคราะห์:

```bash
curl -s "http://127.0.0.1:3000/books?pp=flamegraph" -o flamegraph.html
```

ในเบราว์เซอร์จริง คำสั่งนี้จะเปิดหน้า **speedscope** (เครื่องมือ visualize flame graph แบบ
interactive ที่ฝังมาด้วย) ให้เห็นเป็นแท่งสีซ้อนกันเป็นชั้นๆ — ยิ่งแท่งกว้าง ยิ่งกิน CPU เวลามาก
ยิ่งแท่งลึก ยิ่งเป็น method ที่ถูกเรียกจากข้างใน method อื่นอีกที ข้อมูลดิบที่ฝังอยู่ในหน้า HTML
นี้คือ JSON รูปแบบ stackprof ธรรมดา ดึงออกมาวิเคราะห์ตรงๆ ได้เช่นกัน:

```python
import re, json
txt = open("flamegraph.html", encoding="utf-8").read()
graph = json.loads(re.search(r"var graph = (\{.*?\});", txt).group(1))

print("โหมด:", graph["mode"], "| จำนวน sample:", graph["samples"])

frames = graph["frames"]
# กรองเฉพาะ frame ที่มาจากโค้ดแอปเราเองหรือ ActiveRecord/ActionView โดยตรง
interesting = [f for f in frames.values()
               if "/app/" in (f.get("file") or "") or "activerecord" in (f.get("file") or "")]
interesting.sort(key=lambda f: f["total_samples"], reverse=True)
for f in interesting[:6]:
    print(f["total_samples"], f["name"])
```

ผลลัพธ์จริงจากการรัน flamegraph บนหน้า `/books` เวอร์ชันที่ยังมี N+1 (รวม 3,116 samples):

```
2657  #<Class:...>#_app_views_books_index_html_erb__...   (ตัว view template เอง)
2564  Book::GeneratedAssociationMethods#author
2516  ActiveRecord::Associations::SingularAssociation#reader
2512  ActiveRecord::Associations::Association#reload
```

**อ่านผลได้ว่า:** จาก 3,116 samples ที่ stackprof เก็บระหว่าง request ทั้งหมด มีถึง **2,657 samples
(~85%)** ที่ call stack ยังอยู่ภายในไฟล์ `app/views/books/index.html.erb` และในจำนวนนั้น **2,564
samples** กำลังอยู่ในเมธอด `#author` (accessor ของความสัมพันธ์ `belongs_to :author` ที่ Rails
สร้างให้อัตโนมัติ) ซึ่งลึกลงไปอีกคือ `Association#reload` — คือขั้นตอนการยิง query ใหม่เมื่อ
ความสัมพันธ์ยังไม่ถูกโหลดไว้ก่อน **flamegraph ยืนยันเรื่องเดียวกับที่ SQL count บอกใน Step 643
จากอีกมุมหนึ่ง: เวลาแทบทั้งหมดของ request นี้ไม่ได้เสียไปกับ "การ render HTML" แต่เสียไปกับ
"การเรียก query ซ้ำๆ ที่ซ่อนอยู่ข้างในลูป render" ต่างหาก**

> **หมายเหตุ:** พารามิเตอร์เสริมที่ปรับได้ผ่าน query string เช่นกัน:
> `flamegraph_sample_rate` (ความถี่ในการ sample, ค่า default คือทุก 500 ไมโครวินาที — ยิ่งถี่ยิ่ง
> ละเอียดแต่ overhead สูงขึ้น) และ `flamegraph_mode=cpu` (เปลี่ยนจาก wall-clock time เป็น CPU time
> ล้วนๆ ไม่นับเวลาที่ thread ว่างรอ I/O เช่นรอฐานข้อมูลตอบกลับ — มีประโยชน์เวลาต้องการแยกว่าเวลา
> ที่หายไปเป็น "รอ I/O" หรือ "คำนวณจริง")

---

## Step 645: แก้ N+1 ที่เจอ แล้ว profile ซ้ำ — วัดผลต่างก่อน/หลังด้วยตัวเลขจริง

ทั้ง Step 643 (SQL count) และ Step 644 (flamegraph) ชี้ไปที่จุดเดียวกัน: `@books = Book.all`
ไม่ได้ preload ความสัมพันธ์ `author` มาด้วย แก้ไขด้วยเทคนิค **eager loading** ที่เรียนมาใน
Part 064:

```ruby
# app/controllers/books_controller.rb
class BooksController < ApplicationController
  def index
    @books = Book.includes(:author) # เปลี่ยนจาก Book.all บรรทัดเดียว
  end
end
```

view ไม่ต้องแก้อะไรเลย — `book.author.name` โค้ดหน้าตาเหมือนเดิมทุกตัวอักษร เพราะ `.includes`
แค่บอก Rails ให้ preload ข้อมูลมาล่วงหน้าด้วย query เดียว (`SELECT * FROM authors WHERE id IN
(...)`) แล้วจับคู่ให้อัตโนมัติ ไม่ต้องเปลี่ยนวิธีเขียนโค้ดที่ใช้ข้อมูลเลย

### Profile ซ้ำเพื่อยืนยันผล

```bash
curl -s -i http://127.0.0.1:3000/books | grep -iE "x-runtime|x-miniprofiler-ids"
```

```
x-runtime: 0.141100
x-miniprofiler-ids: 5z45ci7redovuekyfvs5
```

```bash
curl -s -H "X-Requested-With: XMLHttpRequest" \
  "http://127.0.0.1:3000/mini-profiler-resources/results?id=5z45ci7redovuekyfvs5" \
  | python3 -m json.tool
```

```json
{
  "duration_milliseconds": 142.2968269998819,
  "sql_count": 2,
  "duration_milliseconds_in_sql": 74.657933000708
}
```

### ตารางเปรียบเทียบก่อน/หลัง (ตัวเลขจริงทั้งคู่)

| ตัวชี้วัด | ก่อนแก้ (`Book.all`) | หลังแก้ (`Book.includes(:author)`) | ดีขึ้น |
|---|---|---|---|
| เวลารวมทั้ง request | 2,765.2 ms | 142.3 ms | **เร็วขึ้น ~19.4 เท่า** |
| จำนวน SQL query | 2,236 ครั้ง | 2 ครั้ง | **ลดลง 1,118 เท่า** |
| `x-runtime` (Rails รายงานเอง) | 2.59 วินาที | 0.14 วินาที | **เร็วขึ้น ~18.4 เท่า** |

การเปลี่ยนโค้ดแค่ **1 คำ** (`.all` → `.includes(:author)`) ทำให้หน้าเว็บเร็วขึ้นเกือบ 20 เท่า —
นี่คือเหตุผลที่บทนี้ยืนยันหนักแน่นว่า **ต้อง profile ก่อนแก้เสมอ**: ถ้าไม่มีเครื่องมือชี้เป้า
แม่นยำขนาดนี้ นักพัฒนาจำนวนมากจะเสียเวลาไปกับการปรับแต่งจุดอื่นที่ไม่ใช่ปัญหาจริง

> **preview สู่ Step 650:** ขั้นตอน "profile → เจอปัญหา → แก้ → profile ซ้ำ" ที่เพิ่งทำเป็นแบบฝึกหัด
> ไปแล้วรอบหนึ่งใน Step นี้ คือ**แบบฝึกหัดหลัก**ของ Part นี้ทั้งหมด Step 650 จะให้ลองทำซ้ำ
> กระบวนการเดียวกันนี้ด้วยตัวเอง กับหน้าเว็บอีกหน้าหนึ่งที่มีปัญหาคล้ายกันแต่ไม่เหมือนกันเป๊ะ

---

## Step 646: `memory_profiler` — เทียบการจอง object ระหว่างโค้ด naive กับโค้ดที่ให้ DB ทำงานแทน

`rack-mini-profiler` เก่งเรื่อง**เวลา**และ**จำนวน query** แต่ไม่ได้บอกอะไรเกี่ยวกับ**หน่วยความจำ**
เลย — โค้ดสองแบบอาจใช้เวลาพอๆ กัน แต่แบบหนึ่งอาจจอง object ใน RAM มากกว่าอีกแบบสิบเท่า ซึ่งจะ
ส่งผลเป็นการทำงานหนักขึ้นของ Garbage Collector (GC) และหน่วยความจำของ process โตขึ้นเรื่อยๆ
เมื่อมี concurrent request จำนวนมาก — gem `memory_profiler` มีไว้ตอบคำถามนี้โดยเฉพาะ

### ติดตั้งและใช้งานพื้นฐาน

```ruby
# Gemfile (group :development)
gem "memory_profiler", "~> 1.1"
```

รูปแบบการใช้งานหลักคือ `MemoryProfiler.report { ... }.pretty_print`

### ตัวอย่างจริง: รวมยอดจำนวนหน้าหนังสือของแต่ละนักเขียน 2 แบบ

**แบบที่ 1 — naive:** โหลด `Book` ทุกแถวเป็น ActiveRecord object เต็มรูปแบบ แล้ววนลูปรวมเลขเอง
ด้วย Ruby (รูปแบบที่เขียนกันทั่วไปโดยไม่ทันคิดว่ากำลังสร้าง object จำนวนมากเกินจำเป็น):

```ruby
require "memory_profiler"

report_naive = MemoryProfiler.report do
  totals = Hash.new(0)
  Book.includes(:author).find_each do |book|
    totals[book.author.name] += book.pages
  end
  totals
end
report_naive.pretty_print(scale_bytes: true)
```

**แบบที่ 2 — ให้ฐานข้อมูลรวมเลขให้:** ใช้ `.group(...).sum(...)` ของ ActiveRecord ซึ่งแปลงเป็น
SQL `GROUP BY` + `SUM()` แล้วให้ SQLite/PostgreSQL คำนวณให้เสร็จ ส่งกลับมาแค่ Hash ผลลัพธ์เล็กๆ
โดยไม่ต้องสร้าง `Book`/`Author` object แม้แต่ตัวเดียว:

```ruby
report_grouped = MemoryProfiler.report do
  Book.joins(:author).group("authors.name").sum(:pages)
end
report_grouped.pretty_print(scale_bytes: true)
```

### ผลลัพธ์จริงที่ได้ (รันบนข้อมูลชุดเดียวกัน 2,234 books / 300 authors)

```
# แบบที่ 1: naive
Total allocated: 7.90 MB (67605 objects)
Total retained:  493.51 kB (4156 objects)

allocated objects by class
-----------------------------------
     35840  String
     11259  Array
      8447  Hash
      2534  ActiveModel::LazyAttributeSet
      2534  ActiveRecord::Result::IndexedRow
      2234  ActiveRecord::Associations::BelongsToAssociation
      2234  Book
       300  Author
```

```
# แบบที่ 2: group(...).sum(...)
Total allocated: 486.54 kB (5305 objects)
Total retained:  54.59 kB (769 objects)

allocated objects by class
-----------------------------------
      2457  String
      2023  Array
       344  Hash
       301  Enumerator
```

### สรุปเปรียบเทียบ

| | naive (`find_each` + Ruby loop) | `group(...).sum(...)` | ต่างกัน |
|---|---|---|---|
| จำนวน object ที่จอง | 67,605 | 5,305 | **น้อยกว่า ~12.7 เท่า** |
| หน่วยความจำที่จอง | 7.90 MB | 486.5 kB | **น้อยกว่า ~16.2 เท่า** |
| `Book`/`Author` object ที่สร้าง | 2,234 + 300 = 2,534 ตัว | **0 ตัว** | ไม่ต้องสร้างเลย |

**อธิบาย:** ในแบบที่ 1 การเรียก `Book.includes(:author).find_each` แม้จะแก้ปัญหา N+1 เรื่อง
*จำนวน query* ไปแล้ว (มีแค่ 2 query จริงๆ) แต่ยังต้องแปลงทุกแถวในผลลัพธ์ให้เป็น **ActiveRecord
object เต็มรูปแบบ** (มี attribute methods, dirty tracking, association cache ฯลฯ ติดมาด้วย) ทั้งที่
สุดท้ายเราต้องการแค่ตัวเลขสองตัว (ชื่อกับผลรวม) — แบบที่ 2 ไม่ต้องสร้าง object พวกนี้เลยเพราะให้
ฐานข้อมูลคำนวณให้เสร็จตั้งแต่ต้นทาง นี่คือบทเรียนสำคัญ: **การแก้ N+1 ด้วย `.includes` ทำให้จำนวน
query ถูกต้อง แต่ถ้าโจทย์จริงคือ "ต้องการแค่ตัวเลขสรุป" การให้ฐานข้อมูล aggregate ให้เลยยังคง
ประหยัดกว่ามาก ทั้งเวลาและหน่วยความจำ**

### อ่านหมวดอื่นๆ ของ `pretty_print`

`memory_profiler` ยังแบ่งรายงานออกเป็นหมวดที่มีประโยชน์อีกหลายหมวด:

```
allocated memory by gem
-----------------------------------
   4.51 MB  activerecord-8.1.4
   1.23 MB  activemodel-8.1.4
   1.01 MB  sqlite3-2.9.6-x86_64-linux-gnu
 971.76 kB  activesupport-8.1.4
```

หมวด **"by gem"** และ **"by file"** มีประโยชน์มากเมื่อสงสัยว่า gem ตัวไหนใน dependency chain
ของเรากำลังจอง memory มากผิดปกติ (ทบทวนแนวคิดเดียวกันนี้อีกครั้งใน Step 649 ตอนดู
`derailed_benchmarks`) ส่วน **`retained` vs `allocated`**: `allocated` คือ object ทั้งหมดที่ถูก
สร้างขึ้นระหว่าง block ทำงาน ในขณะที่ `retained` คือ object ที่ยังไม่ถูก GC เก็บกวาดทิ้งหลัง block
จบ (ยังมีใครถืออ้างอิงอยู่) — ตัวเลข `retained` ที่สูงผิดปกติเมื่อเทียบกับ `allocated` เป็นสัญญาณ
เตือนของ **memory leak** ที่ควรสืบสวนต่อ

---

## Step 647: `ActiveSupport::Notifications` — กลไกเบื้องหลังที่ทำให้ profiler ทุกตัวทำงานได้

ถึงจุดนี้อาจสงสัยว่า `rack-mini-profiler` รู้ได้อย่างไรว่า SQL query แต่ละครั้งเกิดขึ้นตอนไหน
ใช้เวลาเท่าไหร่ โดยที่ไม่ต้องแก้โค้ดใน ActiveRecord เอง — คำตอบคือ **`ActiveSupport::Notifications`**
ระบบ pub/sub (publish/subscribe) ที่ฝังอยู่ในแทบทุกจุดสำคัญของ Rails framework ตั้งแต่ ActiveRecord
ไปจนถึง ActionView และ ActionController ทุกครั้งที่มีเหตุการณ์สำคัญเกิดขึ้น (ยิง SQL, render
template, เรียก controller action) Rails จะ "ประกาศ" (instrument) เหตุการณ์นั้นออกไป และใครก็ตาม
ที่ "subscribe" ไว้ล่วงหน้าจะได้รับแจ้งพร้อมรายละเอียดครบถ้วน (ชื่อเหตุการณ์, เวลาเริ่ม-จบ,
payload ข้อมูลประกอบ) — `rack-mini-profiler`, `memory_profiler` (บางส่วน), Bullet gem, New Relic,
Skylight **ทุกตัวใช้กลไกนี้เป็นฐานเดียวกันทั้งหมด**

### เขียน instrumentation เองเพื่อทำความเข้าใจกลไกนี้

```ruby
queries = []

subscriber = ActiveSupport::Notifications.subscribe("sql.active_record") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  queries << { sql: event.payload[:sql], duration_ms: event.duration.round(2) }
end

Book.includes(:author).limit(5).each { |b| b.author.name }
Book.joins(:author).group("authors.name").sum(:pages)

ActiveSupport::Notifications.unsubscribe(subscriber)

puts "จับ query ได้ทั้งหมด #{queries.size} ครั้ง"
queries.each { |q| puts "  [#{q[:duration_ms]} ms] #{q[:sql][0, 70]}" }
```

รันจริงได้ผลลัพธ์ (ตัดบางบรรทัดที่เป็น query ตรวจสอบ schema ภายในของ SQLite ออกเพื่อความกระชับ):

```
จับ query ได้ทั้งหมด 13 ครั้ง
  [1.64 ms] SELECT "books".* FROM "books" LIMIT 5 /*application='...'*/
  [0.10 ms] SELECT "authors".* FROM "authors" WHERE "authors"."id" = 1
  [0.80 ms] SELECT SUM("books"."pages") AS "sum_pages", "authors"."name" ...
```

**อธิบายกลไก:**

- `ActiveSupport::Notifications.subscribe("ชื่อ event")` ลงทะเบียน block ที่จะถูกเรียกทุกครั้งที่
  event นั้นเกิดขึ้นที่ไหนก็ตามในแอป — `"sql.active_record"` คือชื่อ event ที่ ActiveRecord
  instrument ทุกครั้งที่ query ถูกส่งไปยังฐานข้อมูลจริง (ไม่ว่าจะเรียกผ่าน `.find`, `.where`,
  `.includes` หรือ raw SQL ก็ตาม)
- `ActiveSupport::Notifications::Event.new(*args)` แปลง argument ดิบ (มี 5 ตัว: ชื่อ event, เวลา
  เริ่ม, เวลาจบ, id, payload) ให้เป็น object ที่ใช้งานสะดวก มี `.duration` (มิลลิวินาที) และ
  `.payload` (Hash ข้อมูลประกอบ เช่น `:sql`, `:name`, `:binds`) ให้เรียกใช้ตรงๆ
- `queries` เป็นแค่ Array ธรรมดาที่เรานิยามเอง — เราสามารถทำอะไรก็ได้กับข้อมูลที่ subscribe มา เช่น
  บวกรวมเวลา, กรองเฉพาะ query ที่ช้ากว่า 100ms, หรือเก็บลง log พิเศษ **นี่คือสิ่งเดียวกับที่
  `rack-mini-profiler` ทำในเวอร์ชันซับซ้อนกว่านี้มาก**: มัน subscribe `"sql.active_record"` และ
  `"render_template.action_view"` (พร้อม event อื่นๆ อีกหลายสิบตัว) เก็บผลไว้เป็น tree ตามลำดับ
  เวลา แล้วนำมาแสดงเป็น badge ที่เห็นใน Step 642 — ไม่มีเวทมนตร์ซ่อนอยู่เลย
- อย่าลืม `unsubscribe` เมื่อเลิกใช้ ถ้าลืมไว้ subscriber จะยังทำงานต่อไปตลอดอายุของ process
  (รวมถึงตอนรัน production จริง ถ้าโค้ดนี้หลุดเข้าไปโดยไม่ตั้งใจ) ซึ่งเป็นการรั่วไหลของ resource
  ที่ตรวจจับได้ยาก

> **preview:** Event ที่มีประโยชน์อื่นๆ ที่ควรรู้จักไว้: `"process_action.action_controller"`
> (ครอบคลุมทั้ง controller action), `"render_template.action_view"` /
> `"render_partial.action_view"` (การ render แต่ละ view/partial แยกกัน — มีประโยชน์มากเวลาหา
> partial ตัวไหนช้าที่สุดในหน้าที่มี partial ซ้อนกันหลายชั้นแบบ Russian Doll Caching จาก
> Part 063), และ `"enqueue.active_job"` / `"perform.active_job"` (ทบทวนจาก Part 061 — ใช้ตรวจสอบ
> ว่า job ถูก enqueue และ perform จริงหรือไม่ในเชิง instrumentation)

---

## Step 648: `benchmark-ips` — เปรียบเทียบความเร็วโค้ดอย่างมีนัยสำคัญทางสถิติ ไม่ใช่แค่ `Time.now`

วิธี "เปรียบเทียบความเร็ว" ที่มือใหม่มักเขียนกันคือแบบนี้:

```ruby
# วิธีที่ไม่ควรใช้ — วัดผลรอบเดียว ไม่มีนัยสำคัญทางสถิติ
start = Time.now
do_something_a
puts Time.now - start

start = Time.now
do_something_b
puts Time.now - start
```

ปัญหาของวิธีนี้คือ **การวัดครั้งเดียวมี noise สูงมาก** — เครื่องอาจกำลังยุ่งกับ process อื่น, GC
อาจเกิดขึ้นพอดีตอนวัด, disk cache อาจยังไม่ warm — ผลที่ได้อาจต่างกันทุกครั้งที่รันซ้ำ จนสรุปผิดได้
ง่ายๆ ว่าโค้ดไหนเร็วกว่ากัน gem **`benchmark-ips`** (iterations per second) แก้ปัญหานี้ด้วยการ
**รันโค้ดซ้ำเป็นพันๆ ครั้งในเวลาที่กำหนด** พร้อมช่วง warmup ก่อนเริ่มนับจริง แล้วรายงานผลเป็น
"จำนวนครั้งต่อวินาที" พร้อม**ค่าความคลาดเคลื่อนทางสถิติ** (standard deviation) ทำให้เชื่อถือได้ว่า
ความต่างที่เห็นเป็นความต่างจริง ไม่ใช่ noise

### ติดตั้งและใช้งาน

```ruby
# Gemfile (group :development)
gem "benchmark-ips", "~> 2.14"
```

เปรียบเทียบสองแนวทางเดียวกับที่ใช้ใน Step 646 (naive loop กับ `group.sum`) แต่คราวนี้วัด**ความเร็ว**
แทนหน่วยความจำ:

```ruby
require "benchmark/ips"

Benchmark.ips do |x|
  x.config(time: 3, warmup: 1) # รันจริง 3 วินาที หลัง warmup 1 วินาที (ค่า default คือ 5 วิ)

  x.report("naive: โหลด AR object แล้วรวมเลขใน Ruby") do
    totals = Hash.new(0)
    Book.includes(:author).find_each { |book| totals[book.author.name] += book.pages }
  end

  x.report("group(...).sum(...) ให้ DB รวมให้") do
    Book.joins(:author).group("authors.name").sum(:pages)
  end

  x.compare! # สรุปว่าตัวไหนเร็วกว่ากี่เท่า
end
```

### ผลลัพธ์จริง

```
Warming up --------------------------------------
naive: โหลด AR object แล้วรวมเลขใน Ruby     3.000 i/100ms
      group(...).sum(...) ให้ DB รวมให้    95.000 i/100ms
Calculating -------------------------------------
naive: โหลด AR object แล้วรวมเลขใน Ruby     37.228 (±13.4%) i/s   (26.86 ms/i) -    114 in   3.06s
      group(...).sum(...) ให้ DB รวมให้    780.502 (±12.8%) i/s    (1.28 ms/i) -  2.375k in   3.04s

Comparison:
      group(...).sum(...) ให้ DB รวมให้:      780.5 i/s
naive: โหลด AR object แล้วรวมเลขใน Ruby:       37.2 i/s - 20.97x  slower
```

**อธิบาย:**

- **`i/s`** (iterations per second) คือจำนวนครั้งที่ block ทำงานสำเร็จต่อวินาที — ยิ่งมากยิ่งเร็ว
  แบบ `group.sum` ทำได้ **780.5 ครั้ง/วินาที** ในขณะที่ naive ทำได้แค่ **37.2 ครั้ง/วินาที**
- **`(±13.4%)`** คือค่าความคลาดเคลื่อนทางสถิติจากการรันหลายรอบ — ตัวเลขนี้ยิ่งเล็กยิ่งน่าเชื่อถือ
  ถ้าค่า `±` ของทั้งสองฝั่งสูงมากจนช่วงค่าซ้อนทับกัน แปลว่า**ยังสรุปไม่ได้ชัดเจน**ว่าอันไหนเร็วกว่า
  จริง (ต้องรันนานขึ้นหรือลด noise ของเครื่องลง) — กรณีนี้ตัวเลขต่างกันมากจนชัดเจนไม่ต้องสงสัย
- **`x.compare!`** สรุปให้อัตโนมัติว่า **`group.sum` เร็วกว่า naive ถึง 20.97 เท่า** ตัวเลขนี้
  สอดคล้องไปในทิศทางเดียวกับผลของ `memory_profiler` ใน Step 646 (จองหน่วยความจำน้อยกว่า ~16 เท่า)
  — เมื่อ**ทั้งเวลาและหน่วยความจำ**ชี้ไปทางเดียวกัน ยิ่งมั่นใจได้ว่าเป็นทางเลือกที่ดีกว่าจริง ไม่ใช่
  การ trade-off ที่ต้องเลือกอย่างใดอย่างหนึ่ง

> **ข้อควรระวัง:** `benchmark-ips` เหมาะกับการเทียบ**โค้ดช่วงสั้นๆ ที่แยกเดี่ยวได้** (micro-benchmark)
> เช่น เทียบ 2 วิธีเขียน query, เทียบ algorithm 2 แบบ — ไม่เหมาะกับการวัดพฤติกรรมของ**ทั้ง
> request** ที่มีปัจจัยแวดล้อมซับซ้อน (เช่น caching, connection pool) ซึ่งกรณีนั้นควรกลับไปใช้
> `rack-mini-profiler` (Step 642–645) แทน

---

## Step 649: `derailed_benchmarks` — หา memory bloat ตอน boot และ memory leak ระหว่างรัน + checklist ปัญหาที่พบบ่อย

เครื่องมือทั้งหมดที่ผ่านมาโฟกัสที่ **1 request หรือ 1 ช่วงโค้ด** แต่บางปัญหาเกิดในระดับ
**ทั้งแอปพลิเคชัน** เช่น "ทำไม process ของเรากิน RAM 500MB ตั้งแต่ยังไม่มีใครเข้าใช้งานเลย"
หรือ "ทำไม RAM ค่อยๆ โตขึ้นเรื่อยๆ จนต้อง restart server ทุกคืน" — คำถามแบบนี้คืองานของ
**`derailed_benchmarks`**

```ruby
# Gemfile (group :development)
gem "derailed_benchmarks", "~> 2.1"
```

### หา memory bloat ตอน boot ด้วย `derailed bundle:mem`

```bash
bundle exec derailed bundle:mem
```

คำสั่งนี้ boot แอปขึ้นมาแบบเต็มรูปแบบ (require ทุก framework ของ Rails) แล้ววัดว่าการ `require`
แต่ละไฟล์/แต่ละ gem กิน RAM ไปเท่าไหร่ พร้อมแสดงเป็น**โครงสร้างต้นไม้**ตามลำดับการพึ่งพา ผลลัพธ์
จริงจากการรัน (Rails 8.1.4):

```
TOP: 52.0313 MiB
  rails/all: 50.0391 MiB
    action_mailbox/engine: 16.9219 MiB
      action_mailbox: 16.9102 MiB
        action_mailbox/mail_ext: 16.7852 MiB
          action_mailbox/mail_ext/address_equality.rb: 14.1836 MiB
            mail/elements/address: 14.1836 MiB
              mail/parsers/address_lists_parser: 14.0117 MiB
```

**อธิบาย:** ตัวเลข **TOP: 52.03 MiB** คือหน่วยความจำรวมที่การ `require` framework ทั้งหมดใช้ไป
ก่อนที่จะมี request แรกเข้ามาด้วยซ้ำ ที่น่าสนใจคือรายการย่อย: การ require `action_mailbox`
(feature รับอีเมลเข้า ที่แอปทดลองนี้**ไม่ได้ใช้งานเลย**) กิน RAM ไปถึง **16.9 MiB** โดยตัวการหลัก
คือการ require ไฟล์ `mail/parsers/address_lists_parser` ของ gem `mail` ที่กินไปเดี่ยวๆ ถึง
**14 MiB** — นี่คือตัวอย่างจริงของ **"memory bloat"**: gem ที่ไม่ได้ตั้งใจใช้แต่ถูก require ติดมา
โดยไม่รู้ตัว (เพราะเป็นส่วนหนึ่งของ `rails/all` หรือ Gemfile ที่โตขึ้นเรื่อยๆ ตามอายุโปรเจกต์) กิน
RAM ไปฟรีๆ ทุก process ที่ boot ขึ้นมา ยิ่งมี worker หลาย process (เช่น Puma แบบ cluster mode,
Sidekiq หลาย process) ยิ่งคูณ RAM ที่เสียไปฟรีๆ นี้ตามจำนวน process

> **แนวทางแก้เมื่อเจอ bloat แบบนี้ในโปรเจกต์จริง:** ถ้าไม่ได้ใช้ Action Mailbox จริงๆ ให้เอา
> `require "action_mailbox/engine"` ออกจาก `config/application.rb` (แอปที่สร้างด้วย
> `rails new --minimal` จะไม่มีบรรทัดนี้ตั้งแต่ต้นอยู่แล้ว) — ทบทวนโครงสร้าง
> `config/application.rb` จาก Part 021

### หา memory leak ระหว่างรันด้วย `derailed exec perf:mem_over_time`

```bash
PATH_TO_HIT=/books TEST_COUNT=600 bundle exec derailed exec perf:mem_over_time
```

คำสั่งนี้เรียก endpoint ที่ระบุซ้ำๆ ตามจำนวน `TEST_COUNT` ครั้ง พร้อมพิมพ์ค่า **RSS** (Resident
Set Size — หน่วยความจำจริงที่ process ใช้อยู่ในตอนนั้น หน่วย MB) ออกมาทุก 5 วินาที ผลลัพธ์จริงจาก
การรัน 600 ครั้งติดกัน:

```
PID: 3053
113.6484375
122.53125
122.53515625
122.5390625
```

**อธิบาย:** RSS เริ่มที่ **113.6 MB** แล้วขยับขึ้นไปที่ **~122.5 MB** ในตัวอย่างที่ 2 จากนั้น
**คงที่**ตลอดตัวอย่างที่ 3 และ 4 — รูปแบบนี้คือ **สุขภาพดี ไม่ใช่ memory leak** การขยับขึ้นครั้งแรก
เป็นเรื่องปกติ (Ruby จองหน่วยความจำเพิ่มสำหรับ object pool, heap page ใหม่, และ cache ภายในต่างๆ
ที่ "อุ่นเครื่อง" ในช่วงแรก) สัญญาณของ **memory leak ตัวจริง** คือกราฟที่ไต่ขึ้นเรื่อยๆ **ไม่มี
วันคงที่** ตลอดการทดสอบ (เช่น 113 → 130 → 150 → 175 → ... ไม่หยุด) ซึ่งมักเกิดจากสาเหตุอย่าง
global cache ที่ไม่มีการจำกัดขนาด, subscriber ของ `ActiveSupport::Notifications` ที่ไม่เคย
`unsubscribe` (ทบทวนคำเตือนจาก Step 647), หรือ closure ที่ถืออ้างอิงถึง object ก้อนใหญ่ไว้โดยไม่
จำเป็น

### Checklist: จุดที่ควรตรวจสอบก่อนเสมอเมื่อ Rails app ช้า

รวบยอดเทคนิคจาก Part 061–064 เป็น checklist ที่ใช้ไล่ตรวจทุกครั้งที่เจอหน้าเว็บช้า **เรียงจาก
พบบ่อยที่สุดไปหาน้อยที่สุดตามประสบการณ์จริงของทีม Rails ทั่วไป**:

| # | อาการที่พบ | เครื่องมือตรวจจับ | ทางแก้ (เรียนใน Part ไหน) |
|---|---|---|---|
| 1 | `sql_count` สูงผิดปกติ, ตัวเลขใกล้เคียงจำนวนแถว | `rack-mini-profiler` (Step 643), Bullet | `.includes`/`.preload`/`.eager_load` (Part 064) |
| 2 | query เดี่ยวช้าแม้เรียกครั้งเดียว, ช้าขึ้นเมื่อข้อมูลเยอะขึ้น | `EXPLAIN ANALYZE` (Part 064) | เพิ่ม `add_index` ให้คอลัมน์ที่ใช้ `WHERE`/`ORDER BY`/`JOIN` (Part 064) |
| 3 | view/partial เดิมถูก render ซ้ำด้วยข้อมูลเดิมทุก request | `server-timing` header (Step 642) | fragment cache / Russian doll caching / `Rails.cache` (Part 063) |
| 4 | response payload ใหญ่ผิดปกติ, serialize ข้อมูลที่ view ไม่ได้ใช้ | ขนาด `content-length`, `memory_profiler` (Step 646) | เลือกเฉพาะคอลัมน์ที่ต้องใช้ (`select`/`pluck`), ปรับ serializer (Part 056–057) |
| 5 | request ค้างรอ งานที่ไม่จำเป็นต้องเสร็จก่อนตอบ user (ส่งอีเมล, เรียก API ภายนอก, export ไฟล์) | `x-runtime` สูงแต่ `sql_count`/CPU ปกติ | ย้ายไปทำใน background job ด้วย ActiveJob + Sidekiq (Part 061–062) |
| 6 | RAM ของ process โตขึ้นเรื่อยๆ ไม่หยุด | `derailed exec perf:mem_over_time` (Step 649) | หา subscriber/cache ที่ไม่มีขอบเขต, ตรวจ `unsubscribe`/`expires_in` |
| 7 | RAM ตอน boot สูงผิดปกติตั้งแต่ยังไม่มี request | `derailed bundle:mem` (Step 649) | ตัด gem/framework ที่ไม่ได้ใช้ออกจาก `config/application.rb`/Gemfile |

การไล่ checklist นี้ทุกครั้งก่อนลงมือแก้ คือการนำหลักการ "วัดก่อน อย่าเดา" จาก Step 641 มาใช้
เป็นรูปธรรม — แทนที่จะเดาว่าเป็นข้อไหน ให้เปิดเครื่องมือที่ตรงกับแต่ละแถวมาเช็คให้แน่ใจก่อนเสมอ

---

## Step 650: เมื่อไหร่ต้องใช้ APM ระดับ production + แบบฝึกหัด + ปิด Phase 9

### ทำไม profiling บนเครื่อง dev ไม่พอ

เครื่องมือทั้งหมดใน Part นี้ทำงานได้ดีเยี่ยม**บนเครื่อง development** ที่เรากดเปิดหน้าเว็บเอง
ทีละ request แต่มีข้อจำกัดสำคัญ: มันแสดงผลแค่ **สถานการณ์ที่เราจำลองขึ้นเอง บนข้อมูลชุดที่เรา
seed เอง** — ในระบบจริงที่มีผู้ใช้จริงหลายพันคนพร้อมกัน มีรูปแบบการใช้งานที่คาดเดาไม่ได้ (บาง
account มีข้อมูลเยอะกว่าที่ทดสอบไว้ 100 เท่า, บาง endpoint ถูกเรียกพร้อมกันหลายพัน request ต่อ
วินาทีจนเกิด lock contention ที่ไม่มีทางจำลองบนเครื่องเดียวได้) — คำถามอย่าง "endpoint ไหนช้าที่สุด
ในสัปดาห์นี้" หรือ "performance แย่ลงหลัง deploy เวอร์ชันล่าสุดหรือไม่" ต้องอาศัยเครื่องมือที่เก็บ
ข้อมูลจาก **traffic จริงต่อเนื่องตลอดเวลา** ซึ่งคือหน้าที่ของ **APM (Application Performance
Monitoring)**

### ภาพรวมสั้นๆ ของตัวเลือก APM ยอดนิยมสำหรับ Rails

| เครื่องมือ | จุดเด่น | เหมาะกับ |
|---|---|---|
| **Skylight** | เน้น Rails โดยเฉพาะ ติดตั้งง่ายมาก UI เรียบง่าย เน้นตัวชี้วัดที่สำคัญจริงๆ ไม่ท่วมท้น | ทีมเล็ก-กลางที่อยากได้ข้อมูลไว เร็ว ไม่ซับซ้อน |
| **New Relic** | ครอบคลุมกว้างมาก (ไม่ใช่แค่ Rails) มี feature ระดับ enterprise เยอะ (distributed tracing ข้าม microservice, log correlation) | องค์กรใหญ่ที่มีหลายภาษา/หลาย service ต้องมองภาพรวมทั้งระบบ |
| **Scout APM** | ราคาเข้าถึงง่าย เน้น Rails/Ruby เช่นกัน มี memory bloat detection ในตัว (แนวคิดคล้าย `derailed_benchmarks` แต่รันต่อเนื่องบน production) | ทีมขนาดกลางที่อยากได้ฟีเจอร์ใกล้เคียง New Relic แต่งบจำกัดกว่า |

ทั้งสามตัวทำงานคล้ายกันในหลักการ: ติดตั้ง gem/agent ที่ subscribe เข้ากับ
`ActiveSupport::Notifications` แบบเดียวกับที่ Step 647 สาธิตให้ดู แต่แทนที่จะพิมพ์ผลออก
terminal มันจะส่งข้อมูลไปเก็บที่ server กลาง แล้วรวบรวมเป็นกราฟ percentile (p50/p95/p99 — เวลา
ตอบสนองที่ผู้ใช้ 50%/95%/99% ได้รับจริง ไม่ใช่แค่ค่าเฉลี่ยที่บิดเบือนได้ง่ายจาก outlier), เตือน
อัตโนมัติเมื่อ endpoint ใดช้าลงผิดปกติ, และเชื่อมโยงกลับมาที่ commit/deploy ที่ทำให้เกิดการ
เปลี่ยนแปลงนั้น — ความสามารถเหล่านี้คือสิ่งที่ `rack-mini-profiler` (ออกแบบมาสำหรับ 1 request
ที่กำลังดูอยู่ตรงหน้า) ไม่ได้ออกแบบมาให้ทำ

> **แนวทางปฏิบัติที่แนะนำ:** ใช้ `rack-mini-profiler` + `memory_profiler` + `benchmark-ips`
> ระหว่างพัฒนา (dev-time) เพื่อวินิจฉัยและแก้ปัญหาที่รู้อยู่แล้วว่ามีอยู่ตรงไหน แล้วติดตั้ง APM
> ตัวใดตัวหนึ่งใน production เพื่อ**ตรวจจับ**ปัญหาที่ยังไม่รู้ว่ามีอยู่ — สองระดับนี้เสริมกัน
> ไม่ได้แทนที่กัน

---

### แบบฝึกหัด: Profile หน้าเว็บช้าจริง หาสาเหตุ แก้ไข แล้ววัดผลต่าง

#### โจทย์

ทีมของคุณได้รับแจ้งว่าหน้า **`/authors`** (สรุปรายชื่อนักเขียนทั้งหมด พร้อมจำนวนหนังสือและยอด
จำนวนหน้ารวมของแต่ละคน) โหลดช้ามาก โค้ดปัจจุบันมีดังนี้:

```ruby
# app/controllers/authors_controller.rb (เวอร์ชันที่มีปัญหา)
class AuthorsController < ApplicationController
  def index
    @authors = Author.all
  end
end
```

```erb
<%# app/views/authors/index.html.erb %>
<h1>สรุปนักเขียนทั้งหมด</h1>
<table>
  <tr><th>ชื่อ</th><th>จำนวนเล่ม</th><th>รวมหน้า</th></tr>
  <% @authors.each do |author| %>
    <tr>
      <td><%= author.name %></td>
      <td><%= author.books.count %></td>
      <td><%= author.books.sum(:pages) %></td>
    </tr>
  <% end %>
</table>
```

ให้ใช้ `rack-mini-profiler` วินิจฉัยปัญหา แก้ไขโค้ด แล้ว profile ซ้ำเพื่อยืนยันว่าดีขึ้นจริงด้วย
ตัวเลข

#### เฉลย

**ขั้นที่ 1 — วัดสภาพก่อนแก้**

```bash
curl -s -i http://127.0.0.1:3000/authors | grep -iE "x-runtime|x-miniprofiler-ids"
```

```
x-runtime: 0.782068
x-miniprofiler-ids: ojb4nxtex78xe7owjcwz
```

```bash
curl -s -H "X-Requested-With: XMLHttpRequest" \
  "http://127.0.0.1:3000/mini-profiler-resources/results?id=ojb4nxtex78xe7owjcwz" \
  | python3 -m json.tool
```

```json
{
  "duration_milliseconds": 941.3112210004329,
  "sql_count": 602,
  "duration_milliseconds_in_sql": 51.235799999631126
}
```

**ขั้นที่ 2 — วิเคราะห์สาเหตุ**

`sql_count: 602` บนระบบที่มี **300 authors** ไม่ใช่เรื่องบังเอิญ: 602 ≈ (300 × 2) + 2 — ทุกครั้งที่
loop ผ่าน `author` หนึ่งคน โค้ด view เรียก `author.books.count` (ยิง query `SELECT COUNT(*)`)
**และ** `author.books.sum(:pages)` (ยิง query `SELECT SUM(pages)`) แยกกันคนละ query ทั้งคู่ —
เป็น **N+1 สองชั้นซ้อนกันในลูปเดียว** (ต่างจาก Step 643 ที่มี N+1 แค่ชั้นเดียว)

**ขั้นที่ 3 — แก้ไข**

```ruby
# app/controllers/authors_controller.rb (แก้แล้ว)
class AuthorsController < ApplicationController
  def index
    # preload ความสัมพันธ์ books ของทุก author มาด้วย query เดียว
    @authors = Author.includes(:books)
  end
end
```

```erb
<%# app/views/authors/index.html.erb (แก้แล้ว) %>
<h1>สรุปนักเขียนทั้งหมด</h1>
<table>
  <tr><th>ชื่อ</th><th>จำนวนเล่ม</th><th>รวมหน้า</th></tr>
  <% @authors.each do |author| %>
    <tr>
      <td><%= author.name %></td>
      <td><%= author.books.size %></td>
      <td><%= author.books.sum(&:pages) %></td>
    </tr>
  <% end %>
</table>
```

**จุดสำคัญที่พลาดไม่ได้:** แค่เปลี่ยน controller เป็น `.includes(:books)` อย่างเดียว **ไม่พอ** —
ถ้า view ยังเรียก `author.books.count` และ `author.books.sum(:pages)` แบบเดิม ActiveRecord จะยัง
ยิง SQL `COUNT`/`SUM` ใหม่ทุกครั้งอยู่ดี (เพราะ `.count`/`.sum(:column)` แบบไม่มี block เป็นเมธอด
คำนวณที่ไว้ใจ **ฐานข้อมูล** เสมอ ไม่สนใจว่า association ถูก preload ไว้แล้วหรือไม่) ต้องเปลี่ยนเป็น
`.size` (ใช้ข้อมูลที่โหลดมาแล้วในหน่วยความจำถ้า preload ไว้แล้ว) และ `.sum(&:pages)` แบบมี block
(บังคับให้ใช้ `Enumerable#sum` วนลูปในหน่วยความจำแทนการยิง SQL ใหม่) — รายละเอียดเล็กๆ แบบนี้คือ
สิ่งที่ profiler ช่วยจับได้ทันทีถ้าลืมแก้ (จะยังเห็น `sql_count` สูงผิดปกติอยู่แม้แก้ controller
ไปแล้ว)

**ขั้นที่ 4 — Profile ซ้ำเพื่อยืนยันผล**

```bash
curl -s -i http://127.0.0.1:3000/authors | grep -iE "x-runtime|x-miniprofiler-ids"
```

```
x-runtime: 0.069087
x-miniprofiler-ids: 4s7uqish403mhu387wfj
```

```json
{
  "duration_milliseconds": 70.01147799928731,
  "sql_count": 2,
  "duration_milliseconds_in_sql": 29.413599000690738
}
```

**ตารางสรุปผล (ตัวเลขจริงทั้งคู่):**

| ตัวชี้วัด | ก่อนแก้ | หลังแก้ | ดีขึ้น |
|---|---|---|---|
| เวลารวมทั้ง request | 941.3 ms | 70.0 ms | **เร็วขึ้น ~13.4 เท่า** |
| จำนวน SQL query | 602 ครั้ง | 2 ครั้ง | **ลดลง 301 เท่า** |
| `x-runtime` | 0.78 วินาที | 0.07 วินาที | **เร็วขึ้น ~11.3 เท่า** |

แบบฝึกหัดนี้จบวงจร **วัด → วินิจฉัย → แก้ → วัดซ้ำ** ที่เป็นแก่นของ Part นี้ทั้ง Part ครบถ้วน
ด้วยตัวเองเป็นครั้งที่สอง (ครั้งแรกคือ Step 643–645 ที่ทำให้ดูเป็นตัวอย่าง)

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **ต่อยอดด้วย caching:** หน้า `/authors` ที่แก้แล้วยังคงคำนวณ `.size`/`.sum(&:pages)` ใหม่ทุก
   request แม้ข้อมูลนักเขียนจะไม่ได้เปลี่ยนบ่อย ลองห่อผลลัพธ์ทั้งหน้าด้วย low-level caching ผ่าน
   `Rails.cache.fetch("authors_summary", expires_in: 1.hour) { ... }` (ทบทวน Part 063) แล้วใช้
   `rack-mini-profiler` วัดว่า request ที่สองเป็นต้นไป (cache hit) เร็วขึ้นจาก 70ms อีกเท่าไหร่
   — ระวังเรื่อง cache invalidation ด้วย: ต้อง expire cache นี้เมื่อไหร่ ถ้ามีการเพิ่ม/ลบหนังสือ
2. **วัด memory ของหน้าที่เพิ่งแก้เอง:** ใช้ `memory_profiler` (Step 646) เปรียบเทียบการจอง
   object ของโค้ด `/authors` เวอร์ชันก่อนแก้กับหลังแก้ด้วยตัวเอง (ทำ `MemoryProfiler.report`
   ครอบ logic เดียวกับที่ controller ทำ) แล้วเขียนสรุปว่าต่างกันกี่เท่า เทียบกับผลด้านเวลาที่
   ได้จาก mini-profiler ไปในทิศทางเดียวกันหรือไม่
3. **ลองใช้ flamegraph กับหน้าที่ไม่ใช่ N+1:** เขียน action ทดลองใหม่ที่มีการคำนวณหนักฝั่ง Ruby
   ล้วนๆ ไม่เกี่ยวกับฐานข้อมูลเลย (เช่น sort array ขนาดใหญ่ด้วย algorithm ที่ไม่มีประสิทธิภาพ หรือ
   ประมวลผล regex ที่ backtrack หนัก) แล้วใช้ `?pp=flamegraph` (Step 644) หา method ที่กิน CPU
   มากที่สุด เปรียบเทียบกับ flamegraph ของหน้าที่มี N+1 ว่าหน้าตาของ "ปัญหา I/O-bound" กับ
   "ปัญหา CPU-bound" ต่างกันอย่างไรเมื่อดูผ่าน flamegraph

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจหลักการ **"วัดก่อน อย่าเดา"** และความหมายที่แท้จริงของ premature optimization — เครื่องมือ
  profiling มีไว้ป้องกันการเสียเวลาไปแก้จุดที่ไม่ใช่ปัญหาจริง
- ติดตั้งและใช้ **`rack-mini-profiler`** อ่าน badge/HTTP header เพื่อดูเวลารวม, จำนวน SQL query,
  และเวลาที่ query ทั้งหมดใช้ไปของแต่ละ request แบบสด พร้อมขุดลึกหา query ที่ซ้ำกัน (สัญญาณ N+1)
- ใช้โหมด **flamegraph** (`?pp=flamegraph`) ร่วมกับ `stackprof` ทำ CPU profiling หา method ที่
  กินเวลาที่สุดใน call stack จริง แยกแยะปัญหา I/O-bound (รอ query) ออกจาก CPU-bound (คำนวณหนัก)
- แก้ปัญหา N+1 ที่ตรวจพบด้วย eager loading (`.includes`) แล้วยืนยันผลด้วยตัวเลขจริง (เร็วขึ้น
  ~19 เท่าในตัวอย่างหลัก, ~13 เท่าในแบบฝึกหัด)
- ใช้ **`memory_profiler`** เปรียบเทียบจำนวน object และหน่วยความจำที่โค้ดสองแบบจองใช้ เข้าใจว่า
  การแก้ N+1 เรื่องจำนวน query ไม่ได้แปลว่าประหยัดหน่วยความจำเสมอไป
- เข้าใจ **`ActiveSupport::Notifications`** กลไก pub/sub เบื้องหลังที่ทำให้ profiler, Bullet, และ
  APM ทุกตัวทำงานได้ พร้อมเขียน instrumentation ง่ายๆ ด้วยตัวเอง
- ใช้ **`benchmark-ips`** เปรียบเทียบความเร็วโค้ดสองแบบอย่างมีนัยสำคัญทางสถิติ แทนการวัดด้วย
  `Time.now` ครั้งเดียวที่ไม่น่าเชื่อถือ
- ใช้ **`derailed_benchmarks`** หา memory bloat ตอน boot แอป (`bundle:mem`) และตรวจสอบว่า RAM
  โตขึ้นเรื่อยๆ ระหว่างรันหรือไม่ (`perf:mem_over_time`)
- มี **checklist ปัญหาประสิทธิภาพที่พบบ่อยที่สุดใน Rails** ที่รวบยอดเทคนิคจาก Part 061–064 ไว้
  เป็นลำดับการตรวจสอบเดียว
- เข้าใจขอบเขตของ dev-time profiling และรู้ว่าเมื่อไหร่ต้องเสริมด้วย **APM** (Skylight, New Relic,
  Scout) สำหรับตรวจจับปัญหาจาก traffic จริงใน production
- ฝึกวงจร **วัด → วินิจฉัย → แก้ → วัดซ้ำ** ด้วยมือทั้งในบทเรียนและแบบฝึกหัด จนกลายเป็นขั้นตอนที่
  ทำได้เป็นธรรมชาติ

## สรุปภาพรวม Phase 9: Background Jobs & Performance

ยินดีด้วย! ตอนนี้ **Phase 9: Background Jobs & Performance (Part 061–065, Step 601–650)** เสร็จ
สมบูรณ์แล้ว เราเดินทางผ่านชุดทักษะที่แยก "แอปที่ทำงานได้" ออกจาก "แอปที่ทำงานได้ดีในระดับ
production" อย่างชัดเจน — เริ่มจาก **ActiveJob** (Part 061) ที่สอนวิธีย้ายงานที่ไม่จำเป็นต้องเสร็จ
ก่อนตอบ user ไปทำเบื้องหลัง, ต่อด้วย **Sidekiq** (Part 062) ที่เจาะลึก queue/retry/scheduled job
ระดับที่ใช้งานจริงในองค์กร, **Caching** (Part 063) ที่สอน fragment cache, Russian doll caching,
และ low-level caching ผ่าน `Rails.cache` เพื่อลดภาระการคำนวณซ้ำซ้อน, **Database Performance**
(Part 064) ที่สอนอ่าน `EXPLAIN ANALYZE`, ออกแบบ index ให้ถูกจุด, และตรวจจับ N+1 ด้วย Bullet
จนมาถึง **Performance Profiling** ใน Part นี้ที่ผูกทุกอย่างเข้าด้วยกันด้วยเครื่องมือวัดผลที่
บอกได้อย่างแม่นยำว่า**เมื่อไหร่**ควรหยิบเทคนิคไหนจาก 4 Part ก่อนหน้ามาใช้

ทักษะทั้ง 5 Part นี้รวมกันคือสิ่งที่แยกนักพัฒนา Rails ระดับ mid-level ออกจากระดับ senior อย่าง
ชัดเจนที่สุดจุดหนึ่ง — โค้ดที่ "ทำงานถูกต้อง" นั้นเขียนได้ไม่ยาก แต่โค้ดที่ "ทำงานถูกต้อง **และ**
รับ traffic จริงได้โดยไม่ล่ม ไม่ทำให้ผู้ใช้รอนาน ไม่ทำให้ server bill พุ่งจากการใช้ RAM/CPU เกิน
ความจำเป็น" ต้องอาศัยทั้งความรู้เชิงเทคนิค (index, cache, background job) **และ** วินัยในการวัดผล
ก่อนตัดสินใจเสมอ ซึ่งคือแก่นของ Part นี้ทั้ง Part

**ต่อไป (Part 066 — เปิด Phase 10: File Upload / Search):** เราจะเปลี่ยนโฟกัสไปยังความสามารถ
ที่แทบทุกแอปพลิเคชันระดับ production ต้องมี — การจัดการไฟล์ที่ผู้ใช้อัปโหลด ด้วย **Active
Storage** ระบบจัดการไฟล์แนบมาตรฐานของ Rails: เริ่มจากการอัปโหลดไฟล์พื้นฐาน (`has_one_attached`,
`has_many_attached`), การสร้าง **variant** ของรูปภาพ (ย่อขนาด, crop, แปลง format) แบบ on-the-fly,
ไปจนถึง **direct upload** ที่ให้เบราว์เซอร์ของผู้ใช้อัปโหลดไฟล์ตรงไปยัง storage โดยไม่ต้องผ่าน
server ของเราก่อน (ลดภาระ server และเวลารอของผู้ใช้ไปพร้อมกัน) — ทักษะการวัดประสิทธิภาพที่เพิ่ง
เรียนจบใน Phase นี้จะยังมีประโยชน์ต่อเนื่อง เพราะการอัปโหลด/ประมวลผลไฟล์คือหนึ่งในจุดที่ทำให้
request ช้าและกิน memory ได้ง่ายที่สุดจุดหนึ่งเช่นกัน
