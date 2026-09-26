# Part 051: Turbo Drive, Turbo Frames — ก้าวแรกสู่ Hotwire

> **Step ครอบคลุมใน Part นี้:** Step 501–510
> **ระดับ:** กลาง (ต่อจาก Part 023–024 เรื่อง Controller/View และ Part 048 เรื่อง System Test ด้วย Capybara)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, `turbo-rails` 2.0.x (Hotwire) — ติดตั้งมาให้อัตโนมัติ
> เมื่อรัน `rails new` แบบเต็ม (ถ้าใช้ `--minimal` อย่างที่เฟส 3 ทำไปตลอด จะ**ไม่มี** gem นี้)

ใน Part 021 ตอนเปรียบเทียบ `rails new` แบบเต็มกับ `--minimal` เราเห็นตารางที่บอกว่า
`importmap-rails`, `turbo-rails`, `stimulus-rails` (Hotwire) จะ **ไม่ถูกติดตั้ง** ถ้าใช้
`--minimal` — และใน Part 048 ตอนเรียนเรื่อง System Test เราก็เกริ่นไว้ว่าถ้าหน้าเว็บต้องพึ่ง
Turbo Frame/Stream ในการแสดงผล การทดสอบด้วย Rack::Test ธรรมดาจะไม่พอ ต้องใช้ระบบที่รัน
JavaScript จริง (Selenium/Cuprite) และบอกว่ารายละเอียดเชิงลึกของ Turbo จะพูดถึงในเฟส 7

**ถึงเวลานั้นแล้ว** — Part นี้เปิดเฟส 7 (Frontend/Hotwire) อย่างเป็นทางการ เราจะเจาะลึก
**Turbo Drive** (กลไกที่ทำให้ Rails 8 app ของเรา "เร็ว" แบบ SPA โดยที่เรายังไม่ได้เขียนโค้ด
อะไรเพิ่มเลยด้วยซ้ำ) และ **Turbo Frames** (การจำกัดขอบเขตการนำทางให้เหลือแค่ส่วนเล็กๆ ของหน้า)
ส่วน **Turbo Streams** (การอัปเดตหน้าเว็บแบบ real-time จากฝั่ง server) จะพูดถึงเต็มรูปแบบใน
Part 052 ถัดไป

ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วยแอป Rails 8.1.4 ที่สร้างด้วย `rails new` (ไม่ใส่
`--minimal`) จึงมี `turbo-rails` เวอร์ชัน **2.0.23** ติดมาให้แล้วตั้งแต่ต้น และตรวจสอบพฤติกรรม
จริงทั้งฝั่ง HTTP (ด้วย `curl` ดู request/response header ตรงๆ) และฝั่ง client-side JavaScript
(ด้วย headless Chromium ผ่าน Playwright เพื่อดูว่า Turbo Drive/Frame "นำทาง" โดยไม่รีโหลดหน้าจริง
หรือไม่ ซึ่งเป็นสิ่งที่ดูจาก HTML/HTTP อย่างเดียวไม่เห็น)

## สารบัญของ Part นี้

- Step 501: Hotwire คืออะไร — ปรัชญา HTML-over-the-wire และทำไม Rails 8 เลือกใช้เป็นค่าเริ่มต้น
- Step 502: Turbo Drive — ดักจับการคลิกลิงก์/ส่งฟอร์ม แล้ว fetch + สลับหน้าแทนการโหลดใหม่ทั้งหมด
- Step 503: `data-turbo="false"` — เมื่อไหร่ที่ต้องปิด Turbo Drive
- Step 504: พฤติกรรม Caching และ Preview ของ Turbo Drive
- Step 505: Post/Redirect/Get กับ Turbo — ทำไมต้อง `redirect_to` และสถานะ 303 See Other
- Step 506: Turbo Frames เบื้องต้น — ขอบเขตการนำทางที่เล็กกว่าทั้งหน้า
- Step 507: Inline Edit ด้วย Turbo Frame แบบเต็มรูปแบบ
- Step 508: Lazy Loading เนื้อหาใน Turbo Frame
- Step 509: Frame ที่ชี้เป้าไปยัง Frame อื่น (`_top`, named target)
- Step 510: Debug Turbo — Network tab, header `Turbo-Frame`, และปัญหา "Content missing"
- แบบฝึกหัด: หน้า Posts ที่แก้ไขได้แบบ inline ผ่าน Turbo Frame

---

## Step 501: Hotwire คืออะไร — ปรัชญา HTML-over-the-wire และทำไม Rails 8 เลือกใช้เป็นค่าเริ่มต้น

**Hotwire** (ย่อมาจาก **H**TML **O**ver **T**he **Wire**) คือชุดเทคนิคที่ทีม 37signals/Basecamp
(ทีมเดียวกับที่สร้าง Rails) พัฒนาขึ้นเพื่อสร้างเว็บแอปที่ตอบสนองเร็วแบบ SPA (Single Page
Application) **โดยไม่ต้องเขียน JavaScript เยอะ** และ**ไม่ต้องแยกฝั่ง frontend ออกจาก backend**
เป็นคนละโปรเจกต์แบบที่สาย React/Vue + JSON API มักทำกัน

Hotwire ประกอบด้วย 4 ส่วนหลัก (2 ส่วนแรกคือหัวใจของ Part นี้):

| ส่วนประกอบ | หน้าที่ | Part ที่สอน |
|---|---|---|
| **Turbo Drive** | เร่งความเร็วการนำทางระหว่างหน้าทั้งหมดในแอป (ทำงานอัตโนมัติ ไม่ต้องเขียนโค้ด) | Part นี้ (Step 502–505) |
| **Turbo Frames** | จำกัดขอบเขตการนำทาง/อัปเดตให้เหลือแค่ส่วนหนึ่งของหน้า | Part นี้ (Step 506–510) |
| **Turbo Streams** | อัปเดตหน้าเว็บจาก server แบบ real-time (ผ่าน WebSocket หรือหลัง form submit) | Part 052 |
| **Stimulus** | เขียน JavaScript เพิ่มเติมแบบมีโครงสร้าง เมื่อ Turbo อย่างเดียวไม่พอ | Part 053 |

### ปรัชญาเบื้องหลัง: ทำไมต้องส่ง HTML แทนที่จะส่ง JSON

แนวทางที่นิยมกันมากในวงการเว็บช่วงหลังคือ **สถาปัตยกรรม SPA เต็มรูปแบบ**: server เป็น JSON API
ล้วนๆ ส่วน browser โหลด JavaScript bundle (React/Vue/Angular) มาทั้งก้อน แล้วให้ JavaScript
เป็นคนสร้าง HTML เองทั้งหมดฝั่ง client วิธีนี้ทรงพลังมาก แต่ก็มีต้นทุนที่มองข้ามไม่ได้:

- ต้องเขียน **logic การแสดงผลซ้ำสองที่** — server รู้อยู่แล้วว่าโพสต์นี้หน้าตายังไง (จาก
  Model + View ที่เราเรียนมาตลอดเฟส 3–6) แต่พอทำ SPA ต้องมาเขียน component ฝั่ง JavaScript
  อธิบายหน้าตาเดิมซ้ำอีกรอบ
- ต้องมี **client-side state management** (Redux, Vuex, Zustand, ...) เพื่อ sync ข้อมูลระหว่าง
  component ต่างๆ — ความซับซ้อนที่เพิ่มขึ้นแบบก้าวกระโดด
- ต้องมี **build pipeline** แยกต่างหาก (webpack/vite ฝั่ง frontend) พร้อมทีมที่ต้องเชี่ยวชาญ
  ทั้ง Ruby และ JavaScript ecosystem คู่ขนานกัน

Hotwire เสนอแนวทางตรงข้าม: **ให้ server ยังคงเป็นคนตัดสินใจเรื่อง HTML เหมือนเดิมทุกประการ**
(ใช้ ERB, partial, helper ที่เราเรียนมาแล้วทั้งหมดใน Part 024) เพียงแต่เปลี่ยนวิธีที่ browser
รับ HTML ก้อนนั้นไปแสดงผล — แทนที่จะโหลดทั้งหน้าใหม่ (full page reload, ขาว-กระพริบ,
เสีย state ของ JS ทั้งหมด) ก็ให้ JavaScript เล็กๆ ที่ Hotwire เตรียมไว้ให้แล้ว (ไม่ต้องเขียนเอง)
ไป **fetch HTML มาแล้วสลับใส่หน้าที่มีอยู่แทน** — ผลลัพธ์คือความรู้สึกเร็วแบบ SPA
แต่โค้ดฝั่งเรายังคงเป็น Rails MVC ธรรมดาแทบทั้งหมด

> **แนวคิดสำคัญที่ควรจำ:** Hotwire ไม่ได้ "แทนที่" JavaScript แต่เป็นการ**ลดปริมาณ JavaScript
> ที่ต้องเขียนเอง**ให้เหลือน้อยที่สุด — งานส่วนใหญ่ (นำทางหน้า, อัปเดตบางส่วนของหน้า) ทำผ่าน
> HTML attribute ธรรมดา (`data-turbo-*`) และ custom element (`<turbo-frame>`) ส่วนที่เหลือ
> ที่จำเป็นต้องมี interactivity จริงๆ (เช่น toggle เมนู, drag-and-drop) ค่อยเขียนด้วย Stimulus
> (Part 053) ซึ่งก็ยังเบากว่า React/Vue มาก

### ทำไม Rails 8 ถึงติดตั้ง Hotwire มาให้เป็นค่าเริ่มต้น

ตรวจสอบ `Gemfile` ที่ `rails new` แบบเต็มสร้างให้ (ทดสอบจริงบน Rails 8.1.4):

```ruby
# Gemfile (ตัดมาเฉพาะส่วนที่เกี่ยวข้อง)

# Use JavaScript with ESM import maps [https://github.com/rails/importmap-rails]
gem "importmap-rails"
# Hotwire's SPA-like page accelerator [https://turbo.hotwired.dev]
gem "turbo-rails"
# Hotwire's modest JavaScript framework [https://stimulus.hotwired.dev]
gem "stimulus-rails"
```

และใน `Gemfile.lock`:

```
turbo-rails (2.0.23)
```

สังเกตคอมเมนต์ที่ Rails ใส่กำกับ `turbo-rails` ไว้เองว่า **"Hotwire's SPA-like page
accelerator"** — ทีม Rails มองว่านี่คือฟีเจอร์พื้นฐานที่ทุกแอปควรมี ไม่ใช่ของเสริมที่ต้องเลือก
ติดตั้งเองอีกต่อไป (ต่างจากยุค Rails 6 ที่ต้องติดตั้ง Turbolinks เองหรือใช้ webpacker ทำ SPA)

ไฟล์ `app/javascript/application.js` ที่ generate มาก็ import ไว้ให้เสร็จสรรพ:

```javascript
// app/javascript/application.js
// Configure your import map in config/importmap.rb. Read more: https://github.com/rails/importmap-rails
import "@hotwired/turbo-rails"
import "controllers"
```

บรรทัด `import "@hotwired/turbo-rails"` นี้เองที่ทำให้ **ทุกหน้าในแอป Rails 8 (ที่ไม่ได้ใช้
`--minimal`) มี Turbo Drive ทำงานอยู่แล้วตั้งแต่วันแรก โดยที่เรายังไม่ได้เขียนโค้ดอะไรเพิ่มเลย**
— นี่คือเหตุผลที่ Part 021/048 บอกว่า Turbo "มีอยู่แล้ว" ในทุกแอปที่เราสร้างมาตลอดเฟส 3–6
เพียงแต่เรายังไม่ได้ลงรายละเอียดว่ามันทำอะไรให้บ้าง

---

## Step 502: Turbo Drive — ดักจับการคลิกลิงก์/ส่งฟอร์ม แล้ว fetch + สลับหน้าแทนการโหลดใหม่ทั้งหมด

### พฤติกรรมแบบเดิม (ไม่มี Turbo) เทียบกับ Turbo Drive

ปกติเมื่อคลิกลิงก์หรือ submit ฟอร์มใน HTML ธรรมดา browser จะ:

1. ยกเลิกทุกอย่างในหน้าปัจจุบันทิ้ง (JS state, scroll position, การเชื่อมต่อ WebSocket)
2. ยิง request ไปหา server
3. รอ response แล้ว **โหลดหน้าใหม่ทั้งหมด** (parse HTML ใหม่, โหลด CSS/JS ใหม่ทั้งชุด,
   วาดหน้าจอใหม่ทั้งจอ — สังเกตได้จากหน้าจอกระพริบขาวสั้นๆ ระหว่างเปลี่ยนหน้า)

**Turbo Drive** ดักพฤติกรรมนี้ตั้งแต่ก่อน browser จะทำงานตามปกติ โดยการ:

1. ดักฟัง event `click` บนทุก `<a>` และ event `submit` บนทุก `<form>` ในหน้า (ผ่าน JavaScript
   ที่ import มาจาก `@hotwired/turbo-rails` อัตโนมัติ)
2. เรียก `preventDefault()` ไม่ให้ browser ทำ navigation ตามปกติ
3. ยิง `fetch()` ไปหา URL เดียวกันนั้นเอง ด้วย `Accept: text/html, application/xhtml+xml`
4. เมื่อได้ HTML กลับมา จะ**แทนที่ทั้ง `<body>`** ของหน้าปัจจุบันด้วย `<body>` ใหม่ที่ parse
   จาก response (และอัปเดต `<head>` เท่าที่จำเป็น เช่น `<title>`) แล้วอัปเดต URL บน address bar
   ด้วย History API (`pushState`) — ทั้งหมดนี้**โดยไม่มีการโหลดหน้าใหม่จริงในระดับ browser เลย**

ผลลัพธ์ที่ผู้ใช้เห็นคือการเปลี่ยนหน้าที่**เร็วขึ้นอย่างชัดเจน** เพราะ:

- ไม่ต้อง parse/ประมวลผล `<head>` ใหม่ทั้งหมด — Turbo เทียบ `<link>`/`<script>` ที่มี
  `data-turbo-track="reload"` (ค่า default ของ `stylesheet_link_tag`/`javascript_importmap_tags`
  ที่ Rails generate ให้) กับของเดิม ถ้าเหมือนกันทุกไฟล์ก็**ข้ามการโหลด asset ซ้ำ**ไปเลย
- JavaScript runtime ไม่ต้อง reset ใหม่ทั้งหมด (ตัวแปร global, WebSocket connection ที่เปิดค้าง
  ไว้ยังคงอยู่ได้ ถ้าโค้ดเขียนรองรับ)
- ไม่มีจอขาวกระพริบระหว่างเปลี่ยนหน้า (perceived performance ดีขึ้นมาก แม้เวลาจริงที่ server
  ใช้ประมวลผลจะเท่าเดิม)

### ทดสอบจริงด้วย headless Chromium (Playwright) — พิสูจน์ว่าไม่มีการโหลดหน้าใหม่จริง

การดูจาก curl อย่างเดียวพิสูจน์ไม่ได้ว่า browser "ไม่โหลดหน้าใหม่" เพราะ curl ไม่รัน
JavaScript เลย จึงทดสอบด้วยเบราว์เซอร์ Chromium จริงผ่าน Playwright แทน — วิธีคือตั้งตัวแปร
`window.__marker` ไว้ก่อนคลิกลิงก์ ถ้า browser โหลดหน้าใหม่จริง ตัวแปรนี้จะหายไป (เพราะ
JavaScript context ถูกรีเซ็ตทั้งหมด) แต่ถ้า Turbo Drive ทำงาน (ไม่โหลดหน้าใหม่) ตัวแปรจะยังอยู่:

```javascript
// verify_drive.js (สคริปต์ทดสอบ ไม่ใช่ส่วนหนึ่งของแอป)
await page.goto('http://localhost:3287/articles');
await page.evaluate(() => { window.__marker = 'alive-before-nav'; });

await page.click('a[href="/articles/new"]');
await page.waitForTimeout(400);

console.log('URL:', page.url());
console.log('marker survived:', await page.evaluate(() => window.__marker));
```

ผลลัพธ์ที่ได้จริง:

```
URL after clicking "เขียนบทความใหม่": http://localhost:3287/articles/new
window.__marker survived (Turbo Drive morph, same JS context): alive-before-nav
```

**URL เปลี่ยนไปเป็น `/articles/new` จริง แต่ `window.__marker` ยังอยู่ครบ** — ยืนยันว่า
ทั้งหน้าไม่ได้ถูกโหลดใหม่จากศูนย์เลย เป็นการดักคลิกแล้วสลับ `<body>` ให้เท่านั้น ตรงตามที่
Turbo Drive ออกแบบไว้

### เบื้องหลังทางเทคนิค: `renderMethod` และการรักษาตำแหน่ง scroll

อ่าน source code ของ `turbo.js` (เวอร์ชัน 2.0.23 ที่มากับ `turbo-rails` ที่ใช้ในหลักสูตรนี้)
พบว่าการนำทางแบบปกติ (คลิกลิงก์ไปหน้าใหม่) ใช้ `PageRenderer` ซึ่งมีค่า `renderMethod` เป็น
`"replace"` เสมอ (ไม่ใช่การ diff/morph แบบละเอียดทีละ element เหมือนที่บาง framework ทำ) —
พูดง่ายๆ คือ Turbo Drive **แทนที่ `<body>` ทั้งก้อนด้วยของใหม่** ความเร็วที่ได้มาจากการ
ข้ามขั้นตอนโหลด asset ซ้ำและไม่ reset JS runtime เป็นหลัก ไม่ใช่จากการ diff DOM แบบละเอียด
(เทคนิค morph แบบละเอียดนั้น Turbo สงวนไว้ใช้กับ Turbo Frame และฟีเจอร์ "Page Refresh" ของ
Turbo Streams ซึ่งจะพูดถึงใน Part 052)

เรื่อง **scroll position**: ถ้าเป็นการนำทางไป "หน้าใหม่" ปกติ (action แบบ `advance`) Turbo Drive
จะเลื่อนกลับไปด้านบนสุดให้ (เหมือน browser ทำเองปกติ) แต่ถ้าเป็นการกด **ปุ่ม back/forward**
ของ browser (action แบบ `restore`) Turbo Drive จะดึง**ตำแหน่ง scroll ที่จำไว้ตอนออกจากหน้านั้น**
กลับมาให้อัตโนมัติ — ต่างจากเว็บที่ทำ full page reload ทุกครั้งซึ่งมักจะรีเซ็ต scroll กลับ
บนสุดเสมอไม่ว่าจะกด back หรือไม่

---

## Step 503: `data-turbo="false"` — เมื่อไหร่ที่ต้องปิด Turbo Drive

Turbo Drive เปิดทำงานกับ **ทุกลิงก์และทุกฟอร์มในหน้าโดยอัตโนมัติ** โดยไม่ต้องตั้งค่าอะไร แต่
บางสถานการณ์เราต้องการให้ browser ทำ full page navigation แบบเดิมจริงๆ — ใช้ attribute
`data-turbo="false"` เพื่อสั่งปิด Turbo Drive เฉพาะจุดนั้น:

```erb
<%# ลิงก์นี้จะทำ full page reload แบบเดิม ไม่ผ่าน Turbo Drive %>
<%= link_to "ดาวน์โหลดไฟล์ PDF", report_path(format: :pdf), data: { turbo: false } %>

<%# ปิดทั้ง form %>
<%= form_with url: external_payment_gateway_path, data: { turbo: false } do |form| %>
  ...
<% end %>
```

`data: { turbo: false }` ใน `link_to`/`form_with` ของ Rails จะ render ออกมาเป็น
`data-turbo="false"` ใน HTML ให้อัตโนมัติ

### ทดสอบจริง: full reload เกิดขึ้นจริงเมื่อใส่ `data-turbo="false"`

ใช้ marker เดียวกับ Step 502 แต่คราวนี้แนบ `data-turbo="false"` เข้าไปที่ลิงก์:

```javascript
await page.evaluate(() => { window.__marker2 = 'alive-before-full-reload'; });
// ... คลิกลิงก์ที่มี data-turbo="false" ...
console.log(await page.evaluate(() => window.__marker2).catch(() => 'undefined (context reset)'));
```

ผลลัพธ์ที่ได้จริง:

```
window.__marker2 after full reload (expect undefined -> JS context reset): undefined
```

`window.__marker2` **หายไปจริง** — ยืนยันว่า `data-turbo="false"` ทำให้เกิด full page
navigation ตามปกติของ browser ไม่ผ่านกลไก fetch+สลับ `<body>` ของ Turbo Drive เลย

### กรณีที่ควรใช้ `data-turbo="false"` ในงานจริง

1. **ลิงก์ดาวน์โหลดไฟล์** (PDF, CSV export, รูปภาพ) — ถ้าปล่อยให้ Turbo Drive ดักไว้ มันจะพยายาม
   `fetch()` ไฟล์นั้นมาเป็น HTML แล้วจะพังเพราะ response ไม่ใช่ HTML (Turbo Drive จะเช็ค
   `Content-Type` และปล่อยผ่านให้ browser จัดการเองถ้าไม่ใช่ HTML อยู่แล้วก็จริง แต่การใส่
   `data-turbo="false"` ชัดเจนกว่าและตัดปัญหา edge case ออกไปตั้งแต่ต้น)
2. **ฟอร์มที่ยิงไปยังโดเมนภายนอก** (เช่น payment gateway ที่ต้อง redirect ไปหน้าของผู้ให้บริการ
   จริงๆ) — Turbo Drive ทำงานได้ดีที่สุดกับ same-origin navigation เท่านั้น
3. **หน้าที่มี third-party JavaScript widget ที่ไม่รองรับการถูก "ย้าย DOM"** (เช่น chat widget
   บางตัวที่ผูก event listener ไว้กับ DOM element ตรงๆ แล้วพังถ้าโดน Turbo Drive สลับ `<body>`
   ทับ) — ปิด Turbo เฉพาะหน้านั้นเป็นทางออกที่เร็วที่สุดโดยไม่ต้องแก้ widget

> **ข้อควรระวัง:** `data-turbo="false"` ปิดเฉพาะ**การนำทาง** (link/form) ไม่ได้ปิด Turbo Drive
> ทั้งระบบ ถ้าต้องการปิด Turbo Drive ทั้งแอป (ไม่แนะนำ เพราะเสียประโยชน์ไปเกือบหมด) ต้อง
> ลบการ `import "@hotwired/turbo-rails"` ออกจาก `app/javascript/application.js` แทน

---

## Step 504: พฤติกรรม Caching และ Preview ของ Turbo Drive

Turbo Drive เก็บ **snapshot ของ DOM ทุกหน้าที่เคยเข้าชม** ไว้ใน memory ของ browser (คนละเรื่อง
กับ HTTP cache ของ server) เพื่อใช้ 2 จุดประสงค์:

### 1. Instant back/forward navigation ด้วย "preview"

เมื่อกดปุ่ม **back** ของ browser กลับไปหน้าที่เคยดูมาก่อน Turbo Drive จะ**แสดง snapshot ที่
เก็บไว้ทันที** (เรียกว่า preview — เห็นหน้าเดิมโผล่ขึ้นมาเลยแบบไม่มีการหน่วงรอ network) ในขณะ
เดียวกันก็ยิง request ไปหา server เพื่อขอข้อมูลล่าสุดควบคู่กันไปเบื้องหลัง แล้วค่อยแทนที่ preview
ด้วยข้อมูลสดเมื่อได้ response กลับมา — นี่คือเหตุผลที่การกด back ใน Rails 8 app รู้สึก
"ทันทีทันใด" กว่าเว็บที่ต้องโหลดใหม่ทุกครั้ง

จาก source code ของ `turbo.js` มี property ชื่อ `isPreview` ที่ใช้แยกแยะว่ากำลังแสดง snapshot
ที่ cache ไว้ (preview) หรือแสดงข้อมูลสดจาก network แล้ว — ยืนยันว่ากลไกนี้มีอยู่จริงในระดับ
implementation ไม่ใช่แค่ทฤษฎี

### 2. หลีกเลี่ยงการยิง request ซ้ำโดยไม่จำเป็น

ถ้า Turbo Drive มี snapshot ของหน้าที่กำลังจะไปอยู่แล้วในหน่วยความจำ (เพิ่งออกจากหน้านั้นมา
ไม่นาน) บางกรณีมันจะใช้ snapshot เดิมแสดงผลได้เลยโดยไม่ต้องรอ network เลยด้วยซ้ำ — เหมาะกับ
การสลับไปมาระหว่างหน้า index กับ show บ่อยๆ

### ควบคุมพฤติกรรม cache ด้วย meta tag `turbo-cache-control`

บางหน้าไม่ควรถูก cache/preview เลย เช่น หน้าที่มีข้อมูลอ่อนไหวเฉพาะ session นั้น (เลขบัตร
เครดิตที่กรอกไว้ชั่วคราว, หน้า checkout) Turbo รองรับ meta tag พิเศษให้ควบคุมได้:

```erb
<%# ปิด cache ทั้งหมดสำหรับหน้านี้ — ทุกครั้งที่กลับมาหน้านี้ (รวมกด back) จะต้องยิง
    request ใหม่เสมอ ไม่แสดง preview จาก snapshot เก่าเลย %>
<% content_for :head do %>
  <meta name="turbo-cache-control" content="no-cache">
<% end %>
```

หรือถ้าต้องการแค่ปิด "preview" ตอนกด back (ยังใช้ cache ปกติได้ตอน navigate ไปข้างหน้า)
ใช้ค่า `no-preview` แทน — จากการอ่าน source พบว่า Turbo เช็คค่านี้ผ่าน 2 property คือ
`isPreviewable` (เทียบกับ `"no-preview"`) และ `isCacheable` (เทียบกับ `"no-cache"`) แยกจากกัน
ชัดเจน ให้เลือกใช้ตามความเหมาะสม:

| ค่า `turbo-cache-control` | ผลลัพธ์ |
|---|---|
| (ไม่ใส่ — ค่า default) | Cache และแสดง preview ได้ตามปกติ |
| `no-preview` | ไม่แสดง preview ตอนกด back/forward แต่ยังเก็บ snapshot ไว้ใช้กรณีอื่น |
| `no-cache` | ไม่เก็บ snapshot ของหน้านี้เลย ทุกครั้งที่มาหน้านี้ต้องยิง request ใหม่เสมอ |

> **แนวปฏิบัติจริง:** ส่วนใหญ่ไม่ต้องยุ่งกับ meta tag นี้เลยเพราะ Turbo Drive cache แค่ใน
> หน่วยความจำของ browser session นั้น (หายไปเมื่อปิดแท็บ ไม่ใช่ persistent storage) ความเสี่ยง
> ด้านความปลอดภัยจึงต่ำกว่าที่คิด แต่สำหรับหน้าที่มีข้อมูลอ่อนไหวจริงๆ (เช่นหลัง logout ไม่อยาก
> ให้กด back แล้วเห็นหน้า dashboard เดิมโผล่มาแม้แค่เสี้ยววินาที) การใส่ `no-cache` ไว้ที่หน้า
> นั้นๆ เป็นเรื่องที่ควรทำ

---

## Step 505: Post/Redirect/Get กับ Turbo — ทำไมต้อง `redirect_to` และสถานะ 303 See Other

ใน Part 023 (Step 227) เราเรียนหลักการ **Post/Redirect/Get (PRG)** ไปแล้วว่า action ที่รับ
`POST`/`PATCH`/`PUT`/`DELETE` แล้วทำสำเร็จ ต้อง `redirect_to` เสมอ (ไม่ใช่ `render`) เพื่อป้องกัน
ปัญหา duplicate form submission ตอนผู้ใช้กด reload — หลักการนี้**ยังคงเป็นจริงทั้งหมด** เมื่อใช้
Turbo แต่มีรายละเอียดเพิ่มเติมอีกชั้นที่ควรรู้

### ทำไม Rails 8 scaffold ใช้ `status: :see_other` (303) แทน 302 ธรรมดา

ลองดู controller ที่ `rails generate scaffold` สร้างให้ (ทดสอบจริงบน Rails 8.1.4):

```ruby
# app/controllers/articles_controller.rb (ส่วนที่เกี่ยวข้อง)
def update
  respond_to do |format|
    if @article.update(article_params)
      format.html { redirect_to @article, notice: "Article was successfully updated.", status: :see_other }
      # ...
```

สังเกตว่า `update` และ `destroy` ที่ scaffold generate ให้มี `status: :see_other` ต่อท้าย
`redirect_to` เสมอ (ส่วน `create` ใช้ redirect ธรรมดาแบบไม่ระบุ status ซึ่งจะได้ค่า default
เป็น 302 Found) — นี่ไม่ใช่เรื่องบังเอิญ แต่เป็นการแก้ปัญหาที่มีมาตั้งแต่ Turbo เริ่มถูกใช้
อย่างแพร่หลาย

**ปัญหาเดิม (redirect ธรรมดา 302 กับ Turbo):** ตาม HTTP spec เดิม เมื่อ browser (หรือ Turbo
Drive) เจอ `302 Found` มันจะ**คงเดิม HTTP method** ของ request ต้นทางไว้ตอน follow redirect
นั่นแปลว่าถ้า `PATCH /articles/1` ตอบกลับ `302` ไปที่ `/articles/1` Turbo อาจจะพยายามยิง
`PATCH /articles/1` **ซ้ำอีกครั้ง** ไปยัง URL ปลายทาง (ตาม RFC เดิม 302 ไม่ได้บังคับเปลี่ยน
เป็น GET เหมือนที่ browser ทั่วไปทำกันมาตามธรรมเนียม) ซึ่งจะกลายเป็นวนซ้ำหรือ error ไม่ตรงกับ
ที่ตั้งใจ

**ทางแก้:** ใช้สถานะ **303 See Other** แทน — 303 **บังคับ**ตาม spec ว่าผู้รับต้องเปลี่ยนไปใช้
`GET` เสมอไม่ว่า method เดิมจะเป็นอะไร ทำให้ Turbo (และ browser ทุกตัว) รู้แน่ชัดว่าต้องยิง
`GET` ไปที่ URL ปลายทาง ไม่มีทางตีความผิดเป็นอย่างอื่น — Rails 8 จึงเปลี่ยนให้ scaffold generate
`update`/`destroy` ด้วย `status: :see_other` เป็นค่าเริ่มต้น เพื่อให้ทำงานถูกต้อง 100% กับ Turbo

### ทดสอบจริงด้วย curl — เห็นสถานะ 303 ตรงๆ

```bash
curl -i -X PATCH http://localhost:3287/articles/1 \
  -H "X-CSRF-Token: ..." \
  --data-urlencode "article[title]=แก้ไขแล้ว"
```

```
HTTP/1.1 303 See Other
location: http://localhost:3287/articles/1
content-length: 0
```

และสำหรับ `destroy`:

```bash
curl -i -X DELETE http://localhost:3287/articles/3 -H "X-CSRF-Token: ..."
```

```
HTTP/1.1 303 See Other
location: http://localhost:3287/articles
```

ยืนยันด้วยการทดสอบจริงว่าทั้ง `update` และ `destroy` ตอบกลับด้วย **303** เสมอ พร้อม
`content-length: 0` (ไม่มี HTML ติดมาด้วย เหมือนหลักการ PRG เดิมทุกประการ — ข้อมูลจริงจะมาจาก
`GET` ที่ตามมาทีหลัง)

### สรุปกฎที่ต้องจำ

| Action | สถานะ redirect ที่ควรใช้ | เหตุผล |
|---|---|---|
| `create` สำเร็จ | `302` (default ของ `redirect_to`) | Browser แปลง `POST` → `GET` ให้เองตามธรรมเนียมมาตั้งแต่แรกอยู่แล้ว ไม่มีปัญหา |
| `update` สำเร็จ | **`303 :see_other`** | ต้องบังคับให้เปลี่ยนเป็น `GET` เพราะ method เดิมคือ `PATCH`/`PUT` |
| `destroy` สำเร็จ | **`303 :see_other`** | เหตุผลเดียวกัน เพราะ method เดิมคือ `DELETE` |
| validation ล้มเหลว (ทุก action) | ไม่ redirect — ใช้ `render` พร้อม `status: :unprocessable_content` | ยังไม่มีอะไรเปลี่ยนแปลงจริง ตาม PRG เดิมที่เรียนใน Part 023 |

> **ทำไมต้องรู้เรื่องนี้ก่อนเรียน Turbo Frame:** ใน Step 507 เราจะเห็นว่าเมื่อฟอร์มถูก submit
> ภายใน Turbo Frame แล้ว controller `redirect_to` กลับไปด้วย 303 พฤติกรรมของ Turbo Frame
> คือจะ **follow redirect นั้นด้วย `GET`** แล้วไปหา `<turbo-frame>` ที่มี `id` ตรงกันในหน้า
> ปลายทางมาแทนที่เนื้อหาเดิม — ถ้า redirect ผิดสถานะ (เช่นลืมใส่ `:see_other`) หรือหน้า
> ปลายทางไม่มี frame ที่ตรงกัน (Step 510) การอัปเดตแบบ inline จะพังทันที

---

## Step 506: Turbo Frames เบื้องต้น — ขอบเขตการนำทางที่เล็กกว่าทั้งหน้า

Turbo Drive (Step 502–505) ทำงานกับ**ทั้งหน้า** เสมอ — คลิกลิงก์ไหนก็ตาม สุดท้าย `<body>`
ทั้งก้อนถูกแทนที่ **Turbo Frame** คือขั้นถัดไป: มันให้เรา**ตีกรอบ**ส่วนหนึ่งของหน้าเป็น
"หน่วยนำทางของตัวเอง" — ลิงก์หรือฟอร์มที่อยู่ **ภายใน** กรอบนั้น เมื่อถูกคลิก/submit จะไม่กระทบ
ส่วนอื่นของหน้าเลย มีแค่เนื้อหาในกรอบเท่านั้นที่เปลี่ยน

### Custom element `<turbo-frame>`

```erb
<%= turbo_frame_tag "my_frame" do %>
  <p>เนื้อหาเริ่มต้น</p>
  <%= link_to "โหลดเนื้อหาใหม่", some_path %>
<% end %>
```

render ออกมาเป็น:

```html
<turbo-frame id="my_frame">
  <p>เนื้อหาเริ่มต้น</p>
  <a href="/some_path">โหลดเนื้อหาใหม่</a>
</turbo-frame>
```

`turbo_frame_tag` เป็น helper จาก `turbo-rails` (module `Turbo::FramesHelper`) กฎการทำงาน
ที่สำคัญที่สุดมี 2 ข้อ:

1. **ลิงก์/ฟอร์มที่อยู่ภายใน `<turbo-frame>` จะถูก "จับ" โดย frame นั้นโดยอัตโนมัติ** — ไม่ต้อง
   เขียนอะไรเพิ่ม เมื่อคลิก มันจะ fetch response แล้ว**มองหา `<turbo-frame>` ที่มี `id` เดียวกัน**
   ในหน้าที่ fetch มา แล้วเอาแค่เนื้อหาข้างในของ frame นั้นมาแทนที่ frame เดิม — **ส่วนอื่นของ
   หน้าไม่ถูกแตะต้องเลย** (URL บน address bar ก็ไม่เปลี่ยนด้วย เพราะนี่ไม่ใช่การนำทางทั้งหน้า)
2. **`turbo_frame_tag(record)`** เมื่อส่ง ActiveRecord object เข้าไป จะแปลงเป็น `id` ให้อัตโนมัติ
   ด้วย `dom_id` (concept เดียวกับที่ใช้ตอน generate scaffold) เช่น
   `turbo_frame_tag(Article.find(1))` จะได้ `<turbo-frame id="article_1">`

### ทดสอบจริง: helper แปลง record เป็น id อย่างไร

```ruby
# turbo-rails source: app/helpers/turbo/frames_helper.rb (คอมเมนต์ในซอร์สเอง)
#   turbo_frame_tag(Article.find(1))
#   # => <turbo-frame id="article_1"></turbo-frame>
#
#   turbo_frame_tag(Article.find(1), "comments")
#   # => <turbo-frame id="comments_article_1"></turbo-frame>
```

### ตัวอย่างที่ทดสอบจริง: list บทความ ที่แต่ละแถวเป็น frame ของตัวเอง

```erb
<%# app/views/articles/_article.html.erb %>
<%= turbo_frame_tag article do %>
  <div>
    <strong>ชื่อเรื่อง:</strong> <%= article.title %>
  </div>
  <div>
    <strong>เนื้อหา:</strong> <%= article.body %>
  </div>
  <div>
    <%= link_to "แก้ไข", edit_article_path(article) %>
  </div>
<% end %>
```

```erb
<%# app/views/articles/index.html.erb %>
<div id="articles">
  <%= render @articles %>
</div>
```

render จริงออกมาได้ (ทดสอบจริงด้วย `curl` แล้วดึงเฉพาะ tag `<turbo-frame>`):

```bash
curl -s http://localhost:3287/articles | grep -oP '<turbo-frame[^>]*>'
```

```
<turbo-frame id="article_1">
<turbo-frame id="article_2">
<turbo-frame id="article_3">
```

แต่ละบทความอยู่ในกรอบของตัวเองแล้ว — ขั้นถัดไป (Step 507) คือทำให้ลิงก์ "แก้ไข" ที่อยู่ข้างใน
กรอบเปลี่ยนเนื้อหาในกรอบนั้นให้กลายเป็นฟอร์ม โดยไม่แตะกรอบอื่นเลย

### พิสูจน์ด้วย Playwright ว่า "คลิกในกรอบ" ไม่กระทบส่วนอื่นของหน้าจริง

```javascript
await page.click('#article_1 a:has-text("แก้ไข")');
await page.waitForTimeout(500);
console.log('URL หลังคลิก (ควรไม่เปลี่ยน):', page.url());
```

ผลลัพธ์จริง:

```
Main frame URL after click (should be UNCHANGED, still /articles): http://localhost:3287/articles
Request for /articles/1/edit found: true
Turbo-Frame header sent: article_1
Form now rendered inside #article_1 frame: true
```

**URL บน address bar ไม่เปลี่ยนเลย** (ยังคงเป็น `/articles`) แม้ว่า Turbo จะยิง request จริงไปหา
`/articles/1/edit` เบื้องหลัง (สังเกต request header พิเศษ `Turbo-Frame: article_1` ที่ถูกส่ง
มาด้วย — จะอธิบายละเอียดใน Step 510) และผลลัพธ์คือฟอร์มแก้ไขไปโผล่อยู่ **เฉพาะภายใน** frame
`article_1` เท่านั้น

---

## Step 507: Inline Edit ด้วย Turbo Frame แบบเต็มรูปแบบ

ตอนนี้มาประกอบร่างทุกอย่างให้เป็น flow การแก้ไขแบบ inline ที่สมบูรณ์: คลิก "แก้ไข" → ฟอร์ม
โผล่มาแทนที่ในกรอบเดิม → กด submit → กรอบนั้นอัปเดตเป็นข้อมูลใหม่ **โดยไม่มีการนำทางหน้าเต็ม
เกิดขึ้นเลยแม้แต่ครั้งเดียว**

### หลักการสำคัญที่มักพลาด: หน้า `show` ก็ต้องมี frame เดียวกันด้วย

Controller `update` ของเรา (Step 505) ใช้ `redirect_to @article, status: :see_other` เหมือน
เดิมทุกประการ — **ไม่ต้องแก้โค้ด controller เลยแม้แต่บรรทัดเดียว** สิ่งที่ต้องทำคือฝั่ง **view**
เท่านั้น: ทั้งหน้า `edit` และหน้า `show` ต้องห่อเนื้อหาด้วย `turbo_frame_tag` ที่มี **id เดียวกัน**
(`article_1`) เพื่อให้ Turbo Frame หาเจอตอน follow redirect ไปหน้า `show`

```erb
<%# app/views/articles/edit.html.erb %>
<% content_for :title, "แก้ไขบทความ" %>

<%= turbo_frame_tag @article do %>
  <%= render "form", article: @article %>
  <%= link_to "ยกเลิก", article_path(@article) %>
<% end %>
```

```erb
<%# app/views/articles/show.html.erb %>
<p style="color: green"><%= notice %></p>

<%= render @article %>
<%# _article.html.erb (Step 506) มี turbo_frame_tag article ห่ออยู่แล้ว
    ทำให้ show ก็มี <turbo-frame id="article_1"> เหมือนกับที่ index ใช้ %>

<p><%= link_to "กลับไปหน้ารายการบทความ", articles_path %></p>
```

### ทดสอบจริงด้วย curl — ไล่ทีละขั้นตอนของ request/response

**ขั้น 1: คลิก "แก้ไข" → GET `/articles/1/edit` พร้อม header `Turbo-Frame: article_1`**

```bash
curl -i http://localhost:3287/articles/1/edit -H "Turbo-Frame: article_1"
```

```
HTTP/1.1 200 OK
content-length: 1505
...
<!-- BEGIN .../turbo-rails-2.0.23/app/views/layouts/turbo_rails/frame.html.erb -->
<html>
  <head>...</head>
  <body>
    <!-- BEGIN app/views/articles/edit.html.erb -->
    <turbo-frame id="article_1">
      <form action="/articles/1" ...>
        ...
```

สังเกต 2 อย่าง: (1) **layout ที่ใช้เปลี่ยนไปเป็น `turbo_rails/frame.html.erb`** ซึ่งเป็น layout
แบบเบาที่มากับ gem เอง (ไม่มี CSS/navbar ของแอปติดมา) — Rails **สลับ layout ให้อัตโนมัติ**
เมื่อตรวจพบ header `Turbo-Frame` เพราะเนื้อหานอกกรอบไม่ถูกใช้อยู่ดี ไม่จำเป็นต้อง render ให้เสีย
เวลา (2) response เล็กลงมาก (1,505 bytes เทียบกับเกือบ 3,000 bytes ตอนโหลดทั้งหน้าแบบปกติ)

**ขั้น 2: กด submit → PATCH `/articles/1` พร้อม header เดิม**

```bash
curl -i -X PATCH http://localhost:3287/articles/1 \
  -H "Turbo-Frame: article_1" -H "X-CSRF-Token: ..." \
  --data-urlencode "article[title]=แก้ไขแล้ว" \
  --data-urlencode "article[body]=เนื้อหาใหม่"
```

```
HTTP/1.1 303 See Other
location: http://localhost:3287/articles/1
content-length: 0
```

controller ทำงานเหมือนเดิมทุกประการกับตอนไม่มี Turbo Frame เลย (redirect 303 ตาม Step 505)

**ขั้น 3: Turbo follow redirect ด้วย GET พร้อม header `Turbo-Frame` เดิม**

```bash
curl -s http://localhost:3287/articles/1 -H "Turbo-Frame: article_1"
```

```html
<turbo-frame id="article_1">
  <div>
    <strong>ชื่อเรื่อง:</strong> แก้ไขแล้ว
  </div>
  <div>
    <strong>เนื้อหา:</strong> เนื้อหาใหม่
  </div>
  ...
```

**หน้า `show` มี `<turbo-frame id="article_1">` ที่ตรงกันพอดี** — Turbo หยิบเนื้อหาข้างในนี้
มาแทนที่กรอบเดิมบนหน้า `index` ได้สำเร็จ

### ยืนยันด้วย Playwright ว่า end-to-end flow ทำงานจริงในเบราว์เซอร์

```javascript
await page.click('#article_1 a:has-text("แก้ไข")');
await page.fill('#article_1 input[name="article[title]"]', 'แก้ไขผ่าน Playwright');
await page.click('#article_1 input[type="submit"]');
await page.waitForTimeout(700);

console.log('URL after submit:', page.url());
console.log('#article_1 shows updated title:',
  (await page.locator('#article_1').innerText()).includes('แก้ไขผ่าน Playwright'));
```

ผลลัพธ์จริง:

```
URL after submit (should STILL be /articles, no full navigation): http://localhost:3287/articles
#article_1 frame content now shows updated title: true
```

**ทั้งหมดนี้เกิดขึ้นโดยที่ผู้ใช้ไม่เคยออกจากหน้า `/articles` เลยแม้แต่วินาทีเดียว** — ไม่มี
full page reload, ไม่มีจอกระพริบ, บทความอื่นๆ ในหน้าเดียวกันก็ไม่ถูกแตะต้องเลยสักนิด นี่คือ
พลังที่แท้จริงของ Turbo Frame: ได้ UX แบบ inline-edit ของ SPA เต็มรูปแบบ โดยไม่มี JavaScript
สักบรรทัดที่เราต้องเขียนเอง

### กรณี validation ล้มเหลว — error แสดงในกรอบเดิมเป๊ะ

ถ้ากรอกข้อมูลไม่ผ่าน validation (เช่น `title` ว่างเปล่า พร้อม `validates :title, presence: true`
ใน model) controller จะ `render :edit, status: :unprocessable_content` ตามปกติ (ไม่ redirect
เพราะยังไม่มีอะไรถูกบันทึกจริง — หลักการ PRG เดิม) ทดสอบจริงได้ผลลัพธ์:

```bash
curl -i -X PATCH http://localhost:3287/articles/2 \
  -H "Turbo-Frame: article_2" -H "X-CSRF-Token: ..." \
  --data-urlencode "article[title]=" --data-urlencode "article[body]=x"
```

```
HTTP status: 422
<turbo-frame id="article_2">
  <form action="/articles/2" ...>
    <div style="color: red">
      <h2>1 error prohibited this article from being saved:</h2>
      <ul>
        <li>Title can't be blank</li>
      </ul>
    </div>
    ...
```

ข้อความ error ไปโผล่**อยู่ในกรอบเดิม** พร้อมฟอร์มที่ยังกรอกอยู่ (ค่าที่พิมพ์ไว้ไม่หาย) — ผู้ใช้
เห็น error แก้แล้วกด submit ใหม่ได้ทันทีโดยไม่ต้องออกจากหน้า index เลย

---

## Step 508: Lazy Loading เนื้อหาใน Turbo Frame

Turbo Frame ไม่จำเป็นต้องมีเนื้อหาตั้งแต่แรกก็ได้ — ใส่ attribute `src` ให้ frame แล้วมันจะ
**ยิง request ไปโหลดเนื้อหาเองทันทีที่ frame นั้นถูกแนบเข้า DOM** (คล้าย `<iframe src="...">`
แต่ผลลัพธ์ถูกดึงมา "แปะ" เป็นส่วนหนึ่งของหน้าจริงๆ ไม่ใช่ iframe แยก document)

### ตัวอย่าง: โหลดสถิติของบทความแบบ lazy

```ruby
# config/routes.rb
resources :articles do
  member do
    get :stats
  end
end
```

```ruby
# app/controllers/articles_controller.rb
def stats
  @word_count = @article.body.to_s.split.size
end
```

```erb
<%# app/views/articles/stats.html.erb %>
<%= turbo_frame_tag "article_#{@article.id}_stats" do %>
  <p>บทความนี้มี <strong><%= @word_count %></strong> คำ</p>
<% end %>
```

```erb
<%# app/views/articles/show.html.erb — เพิ่มเข้าไป %>
<%= turbo_frame_tag "article_#{@article.id}_stats",
      src: stats_article_path(@article), loading: "lazy" do %>
  <p>กำลังโหลดสถิติ...</p>
<% end %>
```

`loading: "lazy"` (ทางเลือกอื่นคือไม่ใส่เลย ซึ่งจะโหลดทันทีที่หน้าโหลดเสร็จ ไม่รอ scroll)
บอก Turbo ว่า**ให้รอจนกว่า frame นี้จะเลื่อนเข้ามาอยู่ใน viewport ก่อน** ถึงค่อยยิง request จริง
— ใช้ [Intersection Observer](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
ของ browser เป็นกลไกเบื้องหลัง เหมาะกับเนื้อหาที่ต้องคำนวณหนักหรืออยู่ด้านล่างของหน้ายาวๆ ที่
ผู้ใช้อาจไม่ได้เลื่อนไปถึง

### ทดสอบจริงด้วย Playwright — พิสูจน์ว่า request ไม่ถูกยิงจนกว่าจะเลื่อนถึง

ทดสอบโดยดันเนื้อหาให้ frame อยู่พ้น viewport ตอนโหลดหน้าแรก (เพิ่ม `<div style="height:
2000px">` คั่นไว้ก่อนหน้าชั่วคราวเพื่อการทดสอบ) แล้วดักฟัง network request ที่มี `/stats`:

```javascript
await page.goto('http://localhost:3287/articles/1');
await page.waitForTimeout(600);
console.log('Stats request count BEFORE scrolling into view:', statsRequests.length);

await page.evaluate(() => {
  document.querySelector('turbo-frame[loading="lazy"]').scrollIntoView();
});
await page.waitForTimeout(800);
console.log('Stats request count AFTER scrolling into view:', statsRequests.length);
```

ผลลัพธ์จริง:

```
Stats request count BEFORE scrolling into view (expect 0): 0
Stats request count AFTER scrolling into view (expect >= 1): 1
Lazy frame content after scroll: บทความนี้มี 8 คำ
```

**ยืนยันชัดเจน:** ไม่มี request ไปหา `/articles/1/stats` เลยจนกว่าจะเรียก `scrollIntoView()`
— ก่อนหน้านั้น frame แสดงข้อความ "กำลังโหลดสถิติ..." ที่เป็น placeholder เฉยๆ ค้างไว้ตาม
เนื้อหาเริ่มต้นที่เขียนไว้ในบล็อกของ `turbo_frame_tag`

> **ข้อควรระวัง:** ทดสอบจริงพบว่าถ้า frame ที่มี `loading: "lazy"` อยู่ **ใกล้ด้านบนของหน้า**
> (อยู่ใน viewport อยู่แล้วตั้งแต่โหลดเสร็จ) มันจะโหลดทันทีเหมือนไม่มี `lazy` เลย เพราะเงื่อนไข
> คือ "อยู่ใน viewport หรือยัง" ไม่ใช่ "รอ N วินาที" — ถ้าอยากให้เห็นผลของ lazy loading ชัดเจน
> ต้องวางเนื้อหาไว้พ้นสายตาจริงๆ ก่อน (ต้องเลื่อนถึงจะเห็น)

---

## Step 509: Frame ที่ชี้เป้าไปยัง Frame อื่น (`_top`, named target)

ปกติลิงก์/ฟอร์มภายใน `<turbo-frame>` จะอัปเดต**เฉพาะกรอบที่ตัวเองอยู่**เท่านั้น (Step 506)
แต่บางสถานการณ์เราต้องการให้การกระทำข้างในกรอบ ส่งผลกระทบไปยังที่อื่นแทน — ทำได้ด้วย
attribute `data-turbo-frame`

### ทะลุออกจากกรอบทั้งหมดด้วย `data-turbo-frame="_top"`

กรณีคลาสสิกที่สุดคือปุ่ม **ลบ** ที่อยู่ในกรอบของแต่ละแถว — ถ้าปล่อยให้พฤติกรรม default ทำงาน
(อัปเดตแค่กรอบตัวเอง) หลังลบสำเร็จ Turbo จะพยายามหา `<turbo-frame>` ที่ id ตรงกันในหน้า
redirect ปลายทาง (`/articles`) ซึ่งไม่มีทางมีอยู่แล้ว (เพราะบทความนั้นถูกลบไปแล้ว!) ทำให้เกิด
ปัญหา "Content missing" (Step 510) ทันที

ทางแก้คือสั่งให้การลบ**ทะลุกรอบออกไปทำ Turbo Drive แบบเต็มหน้า**แทน ด้วย `_top`:

```erb
<%= button_to "ลบ", article, method: :delete,
      data: { turbo_frame: "_top", turbo_confirm: "ยืนยันการลบ?" } %>
```

render ออกมาเป็น:

```html
<form class="button_to" method="post" action="/articles/1">
  <input type="hidden" name="_method" value="delete" />
  <button data-turbo-frame="_top" data-turbo-confirm="ยืนยันการลบ?" type="submit">ลบ</button>
  ...
</form>
```

จาก source code ของ `turbo.js` เมื่อเจอ `data-turbo-frame="_top"` มันจะตั้งค่า target frame
เป็น `null` (แทนที่จะพยายามหา element ด้วย `id="_top"` ซึ่งไม่มีจริง) ซึ่งมีผลให้ Turbo Frame
**ส่งต่อการนำทางนี้ให้ Turbo Drive จัดการแบบเต็มหน้าปกติ** — ผลคือหลังลบสำเร็จ ทั้งหน้าจะ
นำทางไปที่ `/articles` (แสดง flash message "ลบสำเร็จ" ที่ด้านบนสุดของหน้าด้วย ซึ่งจะไม่โผล่เลย
ถ้าอัปเดตแค่ในกรอบเดิม เพราะกรอบไม่ครอบคลุมพื้นที่ flash message)

### ชี้เป้าไปยัง frame ชื่ออื่นจาก "นอก" frame ใดๆ เลย

`data-turbo-frame` ใช้ได้แม้ element นั้นไม่ได้อยู่ใน `<turbo-frame>` เลยด้วยซ้ำ — เหมาะกับ
กรณีฟอร์มค้นหาที่อยู่นอกกรอบ แต่ต้องการให้ผลลัพธ์ไปแสดงในกรอบที่อยู่คนละจุดของหน้า:

```erb
<%# ฟอร์มนี้อยู่ "นอก" turbo-frame ใดๆ เลย แต่สั่ง target ไปที่ turbo-frame
    ชื่อ "search_results" ที่อยู่คนละจุดในหน้าเดียวกันได้ %>
<%= form_with url: articles_path, method: :get, data: { turbo_frame: "search_results" } do |form| %>
  <%= form.text_field :query, placeholder: "ค้นหาชื่อบทความ..." %>
  <%= form.submit "ค้นหา" %>
<% end %>

<%= turbo_frame_tag "search_results" do %>
  <div id="articles">
    <%= render @articles %>
  </div>
<% end %>
```

```ruby
# app/controllers/articles_controller.rb
def index
  @articles = params[:query].present? ? Article.where("title LIKE ?", "%#{params[:query]}%") : Article.all
end
```

ทดสอบจริงด้วย curl (จำลอง header ที่ Turbo จะส่งตอนฟอร์มถูก submit):

```bash
curl -s "http://localhost:3287/articles?query=Hello" -H "Turbo-Frame: search_results"
```

```html
<form data-turbo-frame="search_results" action="/articles" accept-charset="UTF-8" method="get">
  <input placeholder="ค้นหาชื่อบทความ..." value="Hello" type="text" name="query" id="query" />
  <input type="submit" name="commit" value="ค้นหา" data-disable-with="ค้นหา" />
</form>
<turbo-frame id="search_results">
  <div id="articles">
    ...
```

สังเกตว่า `data-turbo-frame="search_results"` ถูก render ลงบน `<form>` โดยตรง (Rails ส่ง
`data:` hash ที่ให้ไว้ใน `form_with` ต่อไปยัง tag `<form>` เอง) และแม้ฟอร์มจะอยู่คนละตำแหน่งกับ
`<turbo-frame id="search_results">` เลย ผลการค้นหาก็ยังไปอัปเดตตรงจุดที่ frame นั้นอยู่ได้ถูกต้อง
— หน้าเว็บทั้งหน้า (รวม input ค้นหาเอง) ไม่ถูกโหลดใหม่เลยระหว่างพิมพ์ค้นหา

---

## Step 510: Debug Turbo — Network tab, header `Turbo-Frame`, และปัญหา "Content missing"

### อ่าน Network tab ใน DevTools เพื่อดูว่า Turbo กำลังทำอะไรอยู่

เมื่อเปิด DevTools → Network tab แล้วคลิกลิงก์ที่อยู่ใน Turbo Frame จะเห็น request ที่มี
**Request Header ชื่อ `Turbo-Frame`** ติดไปด้วยเสมอ ค่าของมันคือ `id` ของ frame ที่กำลัง
นำทางอยู่ — นี่คือวิธีหลักในการแยกแยะว่า request หนึ่งๆ เป็นการนำทางแบบ Turbo Frame
(scope แคบ) หรือ Turbo Drive ปกติ (ทั้งหน้า, ไม่มี header นี้) หรือเป็นการโหลดหน้าแบบเต็ม
ครั้งแรกของ browser (ไม่มี header นี้เช่นกัน)

จาก source code ของ `turbo-rails` gem (`turbo/frames/frame_request.rb`) ฝั่ง server เอง
ก็ใช้ header ตัวนี้ในการตัดสินใจ:

```ruby
# turbo-rails gem source: app/controllers/turbo/frames/frame_request.rb
module Turbo::Frames::FrameRequest
  included do
    layout -> { "turbo_rails/frame" if turbo_frame_request? }
    etag { :frame if turbo_frame_request? }
    helper_method :turbo_frame_request?, :turbo_frame_request_id
  end

  private
    def turbo_frame_request?
      turbo_frame_request_id.present?
    end

    def turbo_frame_request_id
      request.headers["Turbo-Frame"]
    end
end
```

module นี้ถูก include เข้า `ActionController::Base` ให้อัตโนมัติทุก controller — เห็นได้ชัดว่า
Rails **สลับ layout เป็น `turbo_rails/frame` โดยอัตโนมัติทันทีที่เจอ header `Turbo-Frame`**
(ไม่ต้องเขียนโค้ดอะไรเองเลย ตามที่สาธิตไปแล้วใน Step 507) และ `turbo_frame_request?` ก็เป็น
helper method ที่เรียกใช้ได้ทั้งใน controller และ view เผื่อกรณีที่ต้องการเช็คเองว่า request
นี้มาจาก Turbo Frame หรือไม่ (เช่น ต้องการ render เนื้อหาต่างกันเล็กน้อยระหว่างเปิดผ่าน frame
กับเปิดแบบเต็มหน้า)

### เครื่องมือ curl สำหรับ debug โดยไม่ต้องเปิดเบราว์เซอร์

จำลอง header `Turbo-Frame` ด้วย curl ตรงๆ ได้เลย ไม่ต้องพึ่งเบราว์เซอร์ — เป็นวิธีเร็วที่สุด
ในการตรวจสอบว่า response มี frame ที่ต้องการหรือไม่ก่อนจะเสียเวลาเปิด browser debug:

```bash
curl -s http://localhost:3287/articles/1 -H "Turbo-Frame: article_1" \
  | grep -c 'turbo-frame id="article_1"'
```

ถ้าได้ `1` (หรือมากกว่า) แปลว่า response มี frame ที่ตรงกันแน่นอน ถ้าได้ `0` คือสัญญาณเตือน
ล่วงหน้าของปัญหาที่กำลังจะเจอในหัวข้อถัดไป

### Gotcha ที่พบบ่อยที่สุด: "Content missing" เพราะลืมห่อ `<turbo-frame>` ที่ปลายทาง

นี่คือข้อผิดพลาดที่มือใหม่ Turbo Frame แทบทุกคนต้องเจออย่างน้อยหนึ่งครั้ง: ลืมใส่
`turbo_frame_tag` ที่มี **id ตรงกัน** ในหน้าปลายทางที่ frame กำลังจะนำทางไป

**ทดสอบจริง (จงใจทำผิดเพื่อดูอาการ):** ลบ `turbo_frame_tag` ออกจาก `show.html.erb` ชั่วคราว
ให้เหลือแค่ HTML ธรรมดา:

```erb
<%# show.html.erb แบบที่ "ลืม" ห่อ turbo_frame_tag — ตัวอย่างของบั๊ก %>
<div class="post">
  <strong><%= @post.title %></strong>
  <p><%= @post.body %></p>
</div>
```

แล้วยิง request แบบเดียวกับที่ Turbo จะทำหลัง redirect:

```bash
curl -s http://localhost:3287/posts/1 -H "Turbo-Frame: post_1" \
  | grep -c 'turbo-frame id="post_1"'
```

```
0
```

**server ตอบกลับ HTTP 200 ปกติทุกประการ ไม่มี error ใดๆ ฝั่ง server เลย** — แต่ในนั้น**ไม่มี**
`<turbo-frame id="post_1">` อยู่เลย ฝั่ง server จึงไม่มีทางรู้ตัวว่าเกิดปัญหา ปัญหานี้เป็น
**client-side error ล้วนๆ**

### สิ่งที่เกิดขึ้นจริงในเบราว์เซอร์เมื่อ frame หาไม่เจอ

ทดสอบด้วย headless Chromium จริง โดยสร้าง frame ที่ชี้ไปหา URL ที่ไม่มี frame id ตรงกัน:

```javascript
await page.evaluate(() => {
  const frame = document.createElement('turbo-frame');
  frame.id = 'this_id_does_not_exist_anywhere';
  frame.setAttribute('src', '/articles/1');
  document.body.appendChild(frame);
});
await page.waitForTimeout(800);
console.log(await page.evaluate(() =>
  document.getElementById('this_id_does_not_exist_anywhere').innerHTML));
```

ผลลัพธ์จริง:

```html
<strong class="turbo-frame-error">Content missing</strong>
```

**ข้อความ "Content missing" ที่หลายคนเคยเจอโผล่ในหน้าเว็บมาจากตรงนี้เอง** — Turbo แทนที่เนื้อหา
ใน frame ด้วยข้อความ error นี้แทนที่จะปล่อยว่างเงียบๆ เพื่อให้สังเกตเห็นได้ง่ายว่ามีอะไรผิดปกติ

นอกจากนี้ อ่านจาก source code ของ `turbo.js` พบว่านอกจากแสดงข้อความนี้แล้ว Turbo ยังโยน
JavaScript error ออกมาด้วย ซึ่งจะเห็นเป็นข้อความสีแดงใน Console tab ของ DevTools:

```
The response (200) did not contain the expected <turbo-frame id="this_id_does_not_exist_anywhere">
and will be ignored. To perform a full page visit instead, set turbo-visit-control to reload.
```

ข้อความนี้ตรงไปตรงมามาก: **ระบุ id ของ frame ที่หาไม่เจอ** และ**แนะนำทางแก้ไว้ในตัวเลย**
(ตั้ง `turbo-visit-control` เป็น `reload` ถ้าต้องการให้กรณีแบบนี้ fallback เป็นการโหลดหน้าเต็ม
แทนที่จะพัง) — ก่อนจะโยน error นี้ Turbo จะยิง custom event ชื่อ `turbo:frame-missing` ออกมา
ก่อนเสมอ (เป็น event ที่ยกเลิกได้ ด้วย `event.preventDefault()`) เผื่อแอปต้องการดักจับสถานการณ์
นี้เองแล้วจัดการแบบกำหนดเอง (เช่น fallback ไปโหลดหน้าเต็มแทน) แทนที่จะปล่อยให้ error เกิดขึ้น

### ทำไมหน้า error ของ Rails เอง (500/404) ถึงไม่โดนปัญหานี้

ลองสังเกตหน้า error มาตรฐานของ Rails เวลา exception เกิดขึ้นตอน development (เช่น
`ActionController::InvalidAuthenticityToken` ที่เจอตอนทดสอบ CSRF ใน Part นี้เอง) จะพบ meta
tag พิเศษฝังอยู่ใน `<head>`:

```html
<meta name="turbo-visit-control" content="reload">
```

นี่คือกลไกป้องกันที่ Rails ใส่ไว้ใน error page ของตัวเองโดยเฉพาะ — ความหมายคือ **ไม่ว่า
request นี้จะมาจาก Turbo Frame หรือ Turbo Drive ก็ตาม ให้บังคับทำ full page visit เสมอ**
เพื่อให้ผู้พัฒนาเห็นหน้า error เต็มรูปแบบจริงๆ (พร้อม stack trace ทั้งหมด) แทนที่จะเห็นแค่
"Content missing" เล็กๆ ในกรอบที่แทบไม่บอกอะไรเลยว่าเกิด exception ขึ้นจริง — เป็นตัวอย่างที่ดี
ของการออกแบบ error handling ที่คำนึงถึง developer experience ของ Turbo โดยเฉพาะ

### สรุปรายการเช็กเมื่อ Turbo Frame ไม่ทำงานตามที่คาด

1. เปิด **Network tab** → คลิก request ที่สงสัย → เช็คว่ามี Request Header `Turbo-Frame`
   ส่งไปหรือไม่ (ถ้าไม่มี แปลว่าลิงก์/ฟอร์มนั้นอาจไม่ได้อยู่ภายใน `<turbo-frame>` จริง หรือ
   Turbo ยังไม่ได้โหลด)
2. ดู **Response** ของ request นั้น (คลิกเข้าไปดู tab "Response" ใน DevTools หรือ curl ตรงๆ)
   → มี `<turbo-frame id="...">` ที่ id ตรงกับ header `Turbo-Frame` ที่ส่งไปหรือไม่
3. เปิด **Console tab** → มองหาข้อความ `TurboFrameMissingError` หรือ
   `did not contain the expected <turbo-frame ...>`
4. ถ้าเป็นกรณี redirect (เช่นหลัง `update`/`destroy`) ต้องเช็ค**หน้าปลายทางของ redirect**
   ด้วย ไม่ใช่แค่หน้าที่ยิง request ครั้งแรก — เป็นจุดที่พลาดกันบ่อยที่สุด เพราะมักจะแก้แค่
   `edit.html.erb` แล้วลืมว่า `show.html.erb` ก็ต้องมี frame เดียวกันด้วย (Step 507)

---

## แบบฝึกหัด: หน้า Posts ที่แก้ไขได้แบบ Inline ผ่าน Turbo Frame

### โจทย์

สร้างฟีเจอร์ Posts (`Post` model มี `title:string`, `body:text`) ที่ทำงานดังนี้:

1. `/posts` แสดงรายการโพสต์ แต่ละโพสต์ต้องอยู่ใน `<turbo-frame>` ของตัวเอง
2. คลิก "แก้ไข" ที่โพสต์ไหน ฟอร์มแก้ไขต้องโผล่มาแทนที่**เฉพาะโพสต์นั้น** โดยไม่กระทบโพสต์อื่น
   และ URL บน address bar ต้องไม่เปลี่ยน (ยังอยู่ที่ `/posts`)
3. กด submit แล้วข้อมูลอัปเดตสำเร็จ ต้องเห็นผลลัพธ์ใหม่ในกรอบเดิมทันที **ไม่มีการโหลดหน้าใหม่
   ทั้งหน้าเลย**
4. ถ้ากรอกข้อมูลไม่ผ่าน validation (`title` ว่าง) ต้องเห็น error message **อยู่ในกรอบเดิม**
   พร้อมค่าที่กรอกไว้ยังอยู่ครบ
5. ต้องพิสูจน์ (ด้วย `curl`) ว่า contract ของ request/response ถูกต้องตามที่ Turbo Frame
   คาดหวังไว้ทุกขั้นตอน

### เฉลย

**Model:**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
end
```

**Migration** (จาก `rails generate scaffold Post title:string body:text`):

```ruby
# db/migrate/xxxxxxxxxxxxxx_create_posts.rb
class CreatePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :posts do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end
```

**Routes** (ได้มาจาก `resources :posts` ที่ scaffold generator เพิ่มให้อัตโนมัติ ไม่ต้องแก้เอง):

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts
  # ...
end
```

**Controller** — ใช้โค้ดที่ scaffold generate มาให้ **ตรงๆ โดยไม่ต้องแก้อะไรเลยแม้แต่บรรทัด
เดียว** (จุดสำคัญคือ Rails 8.1 generate `status: :see_other` ให้ `update`/`destroy` มาตั้งแต่
ต้นอยู่แล้ว ตามที่เรียนใน Step 505):

```ruby
# app/controllers/posts_controller.rb (ตัดมาเฉพาะส่วนที่เกี่ยวข้อง)
class PostsController < ApplicationController
  before_action :set_post, only: %i[ show edit update destroy ]

  def index
    @posts = Post.all
  end

  def show
  end

  def edit
  end

  def update
    respond_to do |format|
      if @post.update(post_params)
        format.html { redirect_to @post, notice: "Post was successfully updated.", status: :see_other }
        format.json { render :show, status: :ok, location: @post }
      else
        format.html { render :edit, status: :unprocessable_content }
        format.json { render json: @post.errors, status: :unprocessable_content }
      end
    end
  end

  private
    def set_post
      @post = Post.find(params.expect(:id))
    end

    def post_params
      params.expect(post: [ :title, :body ])
    end
end
```

**Views — ส่วนที่ต้องแก้จริงคือแค่ 3 ไฟล์นี้เท่านั้น:**

```erb
<%# app/views/posts/_post.html.erb %>
<%= turbo_frame_tag post do %>
  <div class="post">
    <strong><%= post.title %></strong>
    <p><%= post.body %></p>
    <%= link_to "แก้ไข", edit_post_path(post) %>
  </div>
<% end %>
```

```erb
<%# app/views/posts/index.html.erb %>
<% content_for :title, "รายการโพสต์" %>

<p style="color: green"><%= notice %></p>
<h1>Posts</h1>

<div id="posts">
  <%= render @posts %>
</div>

<%= link_to "เขียนโพสต์ใหม่", new_post_path %>
```

```erb
<%# app/views/posts/edit.html.erb %>
<% content_for :title, "แก้ไขโพสต์" %>

<%= turbo_frame_tag @post do %>
  <%= render "form", post: @post %>
  <%= link_to "ยกเลิก", post_path(@post) %>
<% end %>
```

```erb
<%# app/views/posts/show.html.erb — ต้องเรียก render @post เพื่อให้ frame id ตรงกับ index/edit %>
<p style="color: green"><%= notice %></p>

<%= render @post %>

<p><%= link_to "กลับไปหน้ารายการ", posts_path %></p>
```

`_form.html.erb` ใช้ตัวที่ scaffold generate ให้ตรงๆ ได้เลย ไม่ต้องแก้ (ฟอร์มธรรมดา ไม่มีอะไร
พิเศษเกี่ยวกับ Turbo Frame — เพราะ Turbo Frame ทำงานอัตโนมัติกับฟอร์มที่อยู่ **ภายใน**
`<turbo-frame>` โดยไม่ต้องแก้ตัวฟอร์มเองเลย)

### ยืนยันด้วย curl ว่า contract ครบทุกขั้นตอน

```bash
# 1) index แสดง frame ของแต่ละโพสต์
curl -s http://localhost:3287/posts | grep -oP '<turbo-frame[^>]*>'
```
```
<turbo-frame id="post_1">
<turbo-frame id="post_2">
```

```bash
# 2) เปิดฟอร์มแก้ไขผ่าน frame (header Turbo-Frame: post_1) -> layout เบา, ขนาดเล็กลงชัดเจน
curl -s -o /dev/null -w "status=%{http_code} bytes=%{size_download}\n" \
  http://localhost:3287/posts/1/edit -H "Turbo-Frame: post_1"
```
```
status=200 bytes=1550
```
(เทียบกับโหลดแบบเต็มหน้าปกติที่ได้ 3,095 bytes — ยืนยันว่า layout ถูกสลับเป็นแบบเบาจริง)

```bash
# 3) submit ฟอร์มด้วยข้อมูลถูกต้อง -> 303 See Other กลับไปหน้า show
curl -i -X PATCH http://localhost:3287/posts/1 \
  -H "Turbo-Frame: post_1" -H "X-CSRF-Token: $CSRF" \
  --data-urlencode "post[title]=เริ่มต้นกับ Turbo Frame (แก้ไขแล้ว)" \
  --data-urlencode "post[body]=อัปเดตผ่านฟอร์มในเฟรมโดยไม่รีโหลดทั้งหน้า"
```
```
HTTP/1.1 303 See Other
location: http://localhost:3287/posts/1
content-length: 0
```

```bash
# 4) follow-up GET ที่ Turbo ทำเองอัตโนมัติ -> เจอ frame ที่ตรงกันในหน้า show พร้อมข้อมูลใหม่
curl -s http://localhost:3287/posts/1 -H "Turbo-Frame: post_1"
```
```html
<turbo-frame id="post_1">
  <div class="post">
    <strong>เริ่มต้นกับ Turbo Frame (แก้ไขแล้ว)</strong>
    <p>อัปเดตผ่านฟอร์มในเฟรมโดยไม่รีโหลดทั้งหน้า</p>
    <a href="/posts/1/edit">แก้ไข</a>
  </div>
</turbo-frame>
```

```bash
# 5) กรอกข้อมูลไม่ผ่าน validation -> 422 พร้อม error แสดงในกรอบเดิม
curl -i -X PATCH http://localhost:3287/posts/2 \
  -H "Turbo-Frame: post_2" -H "X-CSRF-Token: $CSRF" \
  --data-urlencode "post[title]=" --data-urlencode "post[body]=x"
```
```
HTTP status: 422
<turbo-frame id="post_2">
  <form action="/posts/2" ...>
    <div style="color: red">
      <h2>1 error prohibited this post from being saved:</h2>
      <ul><li>Title can't be blank</li></ul>
    </div>
    ...
```

ทุกขั้นตอนตรงตาม contract ที่ Turbo Frame คาดหวังไว้ทุกประการ: ขั้นที่ 2 พิสูจน์ว่า Rails
สลับ layout อัตโนมัติเมื่อเจอ header `Turbo-Frame`, ขั้นที่ 3–4 พิสูจน์ pattern การ redirect
303 ที่หน้าปลายทางต้องมี frame id ตรงกัน (Step 505+507), และขั้นที่ 5 พิสูจน์ว่า validation
error ยังทำงานถูกต้องภายใต้ Turbo Frame โดยไม่ต้องเขียนโค้ดพิเศษเพิ่มเลย

### เพื่อความเข้าใจ: ลองทำผิดดูด้วยตัวเอง

ลองลบ `turbo_frame_tag @post do ... end` ออกจาก `show.html.erb` ชั่วคราว (เหลือแค่
`<%= render @post %>` เฉยๆ ตรงๆ โดยไม่ห่อ) แล้วรันคำสั่งขั้นที่ 4 ซ้ำอีกครั้ง จะเห็นว่า
`grep -c 'turbo-frame id="post_1"'` ได้ผลเป็น `0` ทันที — นี่คือสาเหตุที่แท้จริงของปัญหา
"Content missing" ที่อธิบายไว้ใน Step 510 ทดลองดูให้เห็นกับตาตัวเองก่อนแก้กลับ จะช่วยให้จำ
gotcha นี้ได้แม่นกว่าอ่านเฉยๆ

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มปุ่ม "ลบ" ในแต่ละโพสต์ (`button_to ... method: :delete`) พร้อม
   `data: { turbo_frame: "_top", turbo_confirm: "ยืนยันการลบ?" }` ตามที่สอนใน Step 509 —
   ทดสอบด้วย curl ว่า `destroy` ยังคงตอบกลับ 303 เหมือนเดิม แล้วลองอธิบายว่าทำไมปุ่มนี้
   **จำเป็น** ต้องมี `_top` ในขณะที่ปุ่ม "แก้ไข" ไม่จำเป็น (ใบ้: เกิดอะไรขึ้นกับ frame
   `post_1` หลังโพสต์ถูกลบไปแล้ว)
2. เพิ่มฟอร์มค้นหาแบบ Step 509 (ฟอร์มนอกกรอบ ที่ target ไปยัง `<turbo-frame id="search_results">`
   ห่อรอบ `div#posts`) และปรับ `PostsController#index` ให้กรองด้วย `params[:query]` — ทดสอบด้วย
   curl ว่าฟอร์มที่ render ออกมามี attribute `data-turbo-frame="search_results"` ติดอยู่จริง
3. เพิ่ม endpoint `GET /posts/:id/stats` ที่นับจำนวนคำใน `body` แล้วแสดงผ่าน turbo-frame แบบ
   `loading: "lazy"` ในหน้า `show` ตาม Step 508 — ทดสอบด้วย Playwright (หรือเปิด DevTools
   Network tab จริงแล้วสังเกตด้วยตา) ว่า request ไปหา `/stats` ไม่ถูกยิงจนกว่าจะเลื่อนหน้าจอ
   ลงไปถึงตำแหน่งของ frame นั้น
4. (ท้าทายขึ้น) ลองสร้างสถานการณ์ที่ทำให้เกิด `ActiveRecord::RecordNotFound` ขึ้นระหว่าง
   request ที่มี header `Turbo-Frame` ติดไปด้วย (เช่นยิงไปหา `/posts/99999/edit`) แล้วสังเกต
   response ที่ได้ — ใบ้: เทียบกับ `<meta name="turbo-visit-control" content="reload">`
   ที่อธิบายไว้ท้าย Step 510 ว่าทำไมหน้า error ถึงยังแสดงผลได้ครบถ้วนแม้ request ต้นทางจะเป็น
   Turbo Frame request ก็ตาม

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **Hotwire** คือปรัชญา HTML-over-the-wire ของทีม 37signals — ให้ server ยังคงเป็นคนสร้าง HTML
  เหมือนเดิมทั้งหมด ลด custom JavaScript ที่ต้องเขียนเองให้เหลือน้อยที่สุด แทนที่จะแยกเป็น
  SPA + JSON API เต็มรูปแบบ และเข้าใจว่า Rails 8 ติดตั้ง `turbo-rails`/`stimulus-rails` มาให้
  เป็นค่าเริ่มต้นตั้งแต่ `rails new` (ไม่ใช่ `--minimal`)
- **Turbo Drive** ดักจับการคลิกลิงก์/submit ฟอร์มทุกจุดในแอปโดยอัตโนมัติ แล้ว fetch + แทนที่
  `<body>` แทนการโหลดหน้าใหม่เต็มรูปแบบ ทำให้เร็วขึ้นและรักษาตำแหน่ง scroll ได้ — พิสูจน์จริง
  ด้วย headless Chromium ว่า JavaScript context ไม่ถูกรีเซ็ตระหว่างนำทาง
- ใช้ `data-turbo="false"` ปิด Turbo Drive เฉพาะจุดที่จำเป็น (ไฟล์ดาวน์โหลด, ฟอร์มไปโดเมน
  ภายนอก, widget บุคคลที่สามที่ไม่รองรับ) และเข้าใจพฤติกรรม caching/preview ของ Turbo Drive
  ผ่าน meta tag `turbo-cache-control`
- เข้าใจว่าทำไม controller ต้อง `redirect_to` (ไม่ใช่ `render`) หลัง destructive action สำเร็จ
  ตามหลัก PRG (Part 023) และรู้เหตุผลเชิงลึกว่าทำไม Rails 8 scaffold ใช้สถานะ **303 See Other**
  แทน 302 ธรรมดาสำหรับ `update`/`destroy` โดยเฉพาะ
- **Turbo Frames** จำกัดขอบเขตการนำทางให้เหลือแค่ส่วนหนึ่งของหน้า ด้วย `<turbo-frame id="...">`
  และสร้างฟีเจอร์ inline-edit แบบเต็มรูปแบบได้โดยแทบไม่ต้องแก้โค้ด controller เลย (เพียงห่อ
  view ด้วย `turbo_frame_tag` ที่ id ตรงกันทั้งหน้า index/edit/show)
- โหลดเนื้อหาแบบ **lazy** ด้วย `src` + `loading: "lazy"` และสั่งให้ frame **ทะลุออกไปนอกกรอบ**
  ด้วย `data-turbo-frame="_top"` หรือ**ชี้เป้าไปยัง frame อื่น**จากนอกกรอบใดๆ เลยก็ได้
- Debug Turbo ได้ด้วยการเช็ค request header `Turbo-Frame` ใน Network tab และรู้จักสาเหตุที่
  แท้จริงของปัญหา "Content missing" (ลืมห่อ `<turbo-frame>` ที่ id ตรงกันในหน้าปลายทาง) พร้อม
  รู้จัก `turbo-visit-control` ที่ Rails ใช้ป้องกันปัญหานี้ในหน้า error ของตัวเอง

**ต่อไป (Part 052):** เราจะเรียนรู้ **Turbo Streams** — วิธีให้ server ส่งคำสั่งอัปเดต DOM
แบบเจาะจง (append, prepend, replace, remove) กลับไปหา browser ได้โดยตรง ทั้งแบบตอบกลับ
form submission ปกติ และแบบ **real-time ผ่าน WebSocket (Action Cable)** — ทำให้สร้างฟีเจอร์
แบบ "เพื่อนอีกคนพิมพ์ comment แล้วเราเห็นขึ้นทันทีโดยไม่ต้อง refresh" ได้ โดยยังคงไม่ต้องเขียน
JavaScript เองเลยแม้แต่บรรทัดเดียว ต่อยอดจากความเข้าใจเรื่อง Turbo Frame ใน Part นี้โดยตรง
