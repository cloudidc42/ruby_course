# Part 053: Stimulus.js — Controller, Action, Target, Value

> **Step ครอบคลุมใน Part นี้:** Step 521–530
> **ระดับ:** กลาง (ต่อจาก Part 052 เรื่อง Turbo Streams — ควรเข้าใจ Turbo Drive/Frames/Streams
> มาก่อน เพราะ Stimulus ถูกออกแบบมาให้ใช้งานคู่กับ Turbo)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, gem **stimulus-rails** (เวอร์ชันที่ทดสอบจริงคือ
> 1.3.4), ส่ง JavaScript ผ่าน **importmap-rails** เหมือนเดิม (ไม่มี Node.js/esbuild build step)
> ทุกตัวอย่างใน Part นี้สร้างเป็นแอปทดลองจริงด้วย `rails new` (ไม่ใช้ `--minimal`) บน Rails
> 8.1.4 แล้วรัน server จริง คลิกจริงผ่าน headless Chromium (Playwright) เพื่อยืนยันพฤติกรรม
> ก่อนเขียนเนื้อหาทุกตัวอย่าง

ใน Part 051–052 เราเห็นแล้วว่า Turbo Drive/Frames/Streams ทำให้หน้าเว็บอัปเดตแบบไม่ reload
เต็มหน้าได้โดยแทบไม่ต้องเขียน JavaScript เองเลย — แต่ Turbo แก้ปัญหาแค่ "การอัปเดต HTML จาก
server" เท่านั้น ยังมีพฤติกรรมอีกกลุ่มใหญ่ที่ต้องพึ่ง JavaScript ฝั่ง browser ล้วนๆ โดยไม่ต้อง
ยิง request กลับไปหา server เลย เช่น เปิด/ปิดเมนู dropdown, นับจำนวนตัวอักษรที่พิมพ์ในกล่อง
ข้อความ, ยืนยันก่อนลบข้อมูล, auto-save draft ระหว่างพิมพ์ — งานเหล่านี้คือหน้าที่ของ
**Stimulus.js** ซึ่งเป็นเสาที่สามของ **Hotwire** (Turbo + Stimulus + [Strada]) ต่อจาก Turbo
ที่เรียนไปแล้ว

## สารบัญของ Part นี้

- Step 521: Stimulus คืออะไร — ปรัชญา "a modest JavaScript framework"
- Step 522: ติดตั้งและโครงสร้างไฟล์ — `rails generate stimulus`, `stimulus:manifest:update`
- Step 523: Controller Lifecycle — `data-controller`, `connect()`, `disconnect()`
- Step 524: Action — `data-action` เชื่อม DOM event เข้ากับ method ของ controller
- Step 525: Target — อ้างอิง element ลูกโดยไม่ต้อง `querySelector` เอง
- Step 526: Value — state ที่มี type ชัดเจน ซิงค์กับ DOM attribute แบบ reactive
- Step 527: Class — สลับ CSS class โดยไม่ hardcode ชื่อ class ไว้ใน JS
- Step 528: สร้าง Controller ใช้งานจริงทีละขั้น — เมนู Dropdown แบบครบวงจร
- Step 529: หลาย Controller บน Element เดียวกัน/ที่ซ้อนกัน
- Step 530: Outlet — การสื่อสารข้าม Controller (หัวข้อขั้นสูง)
- แบบฝึกหัด: Character Counter สำหรับช่อง Body ของโพสต์

---

## Step 521: Stimulus คืออะไร — ปรัชญา "a modest JavaScript framework"

### ปัญหาที่ Stimulus พยายามแก้

เฟรมเวิร์ก JavaScript สาย SPA อย่าง React, Vue, Angular ถูกออกแบบมาบนสมมติฐานที่ว่า
**JavaScript คือเจ้าของ UI ทั้งหมด** — โครงสร้าง HTML ถูกสร้างขึ้นจาก JavaScript ทั้งหน้า
(หรือเกือบทั้งหน้า) ผ่านกลไก virtual DOM แล้ว render/reconcile กับ DOM จริงเอง ข้อมูล state
ทั้งหมดอยู่ใน JavaScript, server ทำหน้าที่แค่ส่ง JSON ผ่าน API

แนวทางนี้ทรงพลังมากสำหรับแอปที่มี UI ซับซ้อนระดับ Gmail หรือ Figma แต่สำหรับเว็บแอปพลิเคชัน
ทั่วไปแบบที่ Rails ถนัด (แสดงข้อมูล, ฟอร์ม, CRUD, dashboard) การแบก virtual DOM framework
ทั้งชุดมาแค่เพื่อ "เปิด/ปิดเมนู" หรือ "นับตัวอักษร" ถือเป็นการลงทุนที่แพงเกินความจำเป็นมาก —
ต้องมี build step (Webpack/Vite), ต้องเขียน component ที่ render HTML ซ้ำสิ่งที่ server เพิ่ง
render มาแล้วอีกรอบ (เสีย double rendering: server render ERB แล้ว JS ก็ยัง render UI เดิมซ้ำ)

**Stimulus** สร้างโดยทีมเดียวกับ Rails/Turbo (Basecamp) เลือกแนวทางตรงกันข้าม โดยยึดปรัชญาที่
DHH เรียกว่า **"a modest JavaScript framework"** (เฟรมเวิร์ก JavaScript แบบถ่อมตัว):

> Stimulus is designed to **enhance** static or server-rendered HTML — the "boring" HTML you
> already have — by connecting JavaScript objects to elements automatically, using simple
> annotations in the markup itself.

แปลเป็นหลักปฏิบัติ 3 ข้อ:

1. **HTML คือแหล่งความจริง (source of truth)** ไม่ใช่ JavaScript — Rails/ERB ยังคง render
   HTML เหมือนเดิมทุกประการตามที่เรียนมาตั้งแต่ Part 024 Stimulus ไม่ได้มาแทนที่ ERB แต่มา
   **เสริม** (enhance) HTML ที่มีอยู่แล้วให้มีพฤติกรรมโต้ตอบเพิ่มเข้าไป
2. **ไม่มี virtual DOM ไม่มีการ render UI ซ้ำใน JavaScript** — controller ของ Stimulus แค่
   "เกาะ" (attach) เข้ากับ element ที่มีอยู่แล้วในหน้า แล้วสั่งงาน DOM จริงตรงๆ (เพิ่ม/ลบ class,
   เปลี่ยน text, ซ่อน/แสดง element) เหมือนเขียน jQuery ยุคก่อน แต่มีโครงสร้างและ convention
   ที่ชัดเจนกว่ามาก
3. **State ที่จำเป็นต้องมีอยู่น้อยที่สุด และเก็บไว้ใน DOM attribute** ไม่ใช่ใน JavaScript
   object แยกต่างหาก — ทำให้ debug ง่าย (เปิด DevTools ดู HTML ก็เห็น state ปัจจุบันได้เลย)
   และทำให้ Turbo กับ Stimulus ทำงานร่วมกันได้อย่างเป็นธรรมชาติ (รายละเอียดจะเห็นชัดใน
   Step 523 เรื่อง lifecycle)

### ทำไม Stimulus ถึงจับคู่กับ Turbo ได้อย่างเป็นธรรมชาติ

Turbo (Part 051–052) รับผิดชอบ "จะเอา HTML ชิ้นไหนมาแทนที่ตรงไหนของหน้า เมื่อไหร่" — เช่น
สลับหน้าทั้งหน้าแบบไม่ reload (Turbo Drive), แทนที่แค่ region เดียว (Turbo Frame), หรือ
prepend/replace หลายจุดพร้อมกัน (Turbo Stream) ส่วน Stimulus รับผิดชอบ **"HTML ชิ้นที่แสดงอยู่
ตอนนี้ต้องมีพฤติกรรมโต้ตอบอะไรบ้างระหว่างที่ยังอยู่บนหน้าจอ"** — สองอย่างนี้ไม่ทับซ้อนกัน
เลยแม้แต่นิดเดียว ทำงานเป็นทีมเดียวกัน: ทุกครั้งที่ Turbo เอา HTML ชิ้นใหม่มาแทรกลงหน้า (ไม่ว่า
จะจาก Drive, Frame หรือ Stream) Stimulus จะสแกนหา `data-controller` attribute ใน HTML ชิ้นนั้น
โดยอัตโนมัติ แล้ว "เกาะ" controller ที่ตรงกันให้ทันที **โดยไม่ต้องเขียนโค้ดเชื่อมต่ออะไรเองเลย**
— นี่คือเหตุผลที่ Rails 8 เลือกให้ทั้งคู่มาเป็นค่า default พร้อมกันตั้งแต่ตอน `rails new`

### เทียบสั้นๆ กับแนวทาง SPA

| ประเด็น | React/Vue (SPA) | Stimulus |
|---|---|---|
| HTML มาจากไหน | JavaScript render (virtual DOM) | Server render (ERB) — Stimulus แค่เสริม |
| ต้องมี build step (Webpack/Vite) | ต้องมี | ไม่จำเป็น (ใช้ importmap ได้ตรงๆ) |
| เหมาะกับ | UI ซับซ้อนมาก, state เยอะ, offline-first | เว็บแอปทั่วไปที่ server-render เป็นหลัก |
| State อยู่ที่ไหน | JavaScript memory/store | DOM attribute (`data-*`) เป็นหลัก |
| เขียนโค้ดต่อ 1 พฤติกรรมย่อย | มักต้องมี component ทั้งไฟล์ | ไฟล์ controller เล็กๆ ไฟล์เดียว |

> **ข้อคิดสำคัญ:** Stimulus ไม่ใช่คู่แข่งของ React/Vue และไม่ได้ "ด้อยกว่า" — มันแค่แก้ปัญหา
> คนละขนาดกัน ทีม Rails ส่วนใหญ่ที่ใช้ Hotwire (Basecamp, Hey.com, Shopify บางส่วน) เลือกใช้
> Stimulus เพราะแอปของพวกเขาเป็นเว็บแอปแบบ server-rendered เป็นหลัก ไม่ใช่ SPA เต็มรูปแบบ
> ถ้าโปรเจกต์ต้องการ UI ระดับ editor/canvas ที่ซับซ้อนมากจริงๆ (เช่น Figma-like) การผสม React
> เข้ามาเฉพาะจุดนั้นก็ยังทำได้ ไม่ใช่ทางเลือกที่ตัดกันขาด

---

## Step 522: ติดตั้งและโครงสร้างไฟล์ — `rails generate stimulus`, `stimulus:manifest:update`

### stimulus-rails มาพร้อม `rails new` อยู่แล้ว

ถ้าสร้างแอปด้วย `rails new myapp` แบบเต็มรูปแบบ (ไม่ใส่ `--minimal`) gem `stimulus-rails` และ
`turbo-rails` จะถูกเพิ่มใน `Gemfile` และติดตั้งให้อัตโนมัติ ทดสอบจริงจาก log ตอนสร้างแอปใหม่บน
Rails 8.1.4:

```
       rails  turbo:install stimulus:install
       apply  .../turbo-rails-2.0.23/lib/install/turbo_with_importmap.rb
  Import Turbo
      append    app/javascript/application.js
  Pin Turbo
      append    config/importmap.rb
       apply  .../stimulus-rails-1.3.4/lib/install/stimulus_with_importmap.rb
  Create controllers directory
      create    app/javascript/controllers
      create    app/javascript/controllers/index.js
      create    app/javascript/controllers/application.js
      create    app/javascript/controllers/hello_controller.js
  Import Stimulus controllers
      append    app/javascript/application.js
  Pin Stimulus
  Appending: pin "@hotwired/stimulus", to: "stimulus.min.js"
      append    config/importmap.rb
  Appending: pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
      append    config/importmap.rb
  Pin all controllers
  Appending: pin_all_from "app/javascript/controllers", under: "controllers"
      append    config/importmap.rb
```

ตรวจดูไฟล์ที่ generate มาให้จริง 4 ไฟล์:

```ruby
# config/importmap.rb (ส่วนที่เกี่ยวกับ Stimulus)
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

```js
// app/javascript/controllers/application.js
import { Application } from "@hotwired/stimulus"

const application = Application.start()

// Configure Stimulus development experience
application.debug = false
window.Stimulus   = application

export { application }
```

```js
// app/javascript/controllers/index.js
// Import and register all your controllers from the importmap via controllers/**/*_controller
import { application } from "controllers/application"
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading"
eagerLoadControllersFrom("controllers", application)
```

```js
// app/javascript/controllers/hello_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.textContent = "Hello World!"
  }
}
```

```js
// app/javascript/application.js
// Configure your import map in config/importmap.rb. Read more: https://github.com/rails/importmap-rails
import "@hotwired/turbo-rails"
import "controllers"
```

**อ่านความหมายทีละไฟล์:**

- `pin_all_from "app/javascript/controllers", under: "controllers"` — สั่งให้ importmap-rails
  pin ไฟล์ **ทุกไฟล์** ในโฟลเดอร์ `app/javascript/controllers/` ให้อัตโนมัติ ภายใต้ namespace
  `"controllers/..."` เพิ่ม controller ไฟล์ใหม่กี่ไฟล์ก็ตามในโฟลเดอร์นี้ ไม่ต้องมาแก้
  `config/importmap.rb` เองเลยแม้แต่บรรทัดเดียว (ต่างจาก `pin "application"` ที่ pin ทีละไฟล์)
- `controllers/application.js` คือจุดที่สร้าง Stimulus `Application` instance เดียวของทั้งแอป
  (`Application.start()`) — แอปทั้งแอปมี instance นี้แค่ตัวเดียว ทุก controller ถูกลงทะเบียน
  เข้า instance นี้ร่วมกัน `window.Stimulus = application` ทำให้เปิด browser console แล้วพิมพ์
  `Stimulus.controllers` ดู controller ที่กำลัง active อยู่บนหน้าปัจจุบันได้ (มีประโยชน์มากตอน
  debug)
- `controllers/index.js` ใช้ `eagerLoadControllersFrom("controllers", application)` จาก
  `@hotwired/stimulus-loading` — ฟังก์ชันนี้จะสแกนทุกไฟล์ที่ pin ไว้ภายใต้ namespace
  `"controllers"` (มาจาก `pin_all_from` ด้านบน) แล้ว **import และ register ให้อัตโนมัติทุก
  ไฟล์** โดยแปลงชื่อไฟล์เป็นชื่อ controller ให้เอง (อธิบายกฎการแปลงชื่อในหัวข้อถัดไป) — นี่คือ
  เหตุผลที่เราแทบไม่ต้องแตะไฟล์นี้เลยตลอดทั้ง Part นี้ ต่อให้เพิ่ม controller ใหม่กี่ตัวก็ตาม
- `hello_controller.js` คือไฟล์ตัวอย่างที่ generate มาให้ดูโครงสร้างพื้นฐาน (จะลบทิ้งทีหลังได้
  เมื่อไม่ได้ใช้)

### สร้าง Controller ใหม่ด้วย `rails generate stimulus`

วิธีมาตรฐานที่สุดในการเพิ่ม controller ใหม่คือใช้ generator (ทดสอบจริง):

```bash
bin/rails generate stimulus toggle
```

```
      create  app/javascript/controllers/toggle_controller.js
```

เนื้อไฟล์ที่ได้ (สังเกต comment ที่ generator ใส่ไว้ให้บอกว่าจะผูกกับ
`data-controller="toggle"`):

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

// Connects to data-controller="toggle"
export default class extends Controller {
  connect() {
  }
}
```

จุดที่สำคัญมาก (ทดสอบจริงแล้ว): **ไม่ต้องแก้ `config/importmap.rb` เพิ่มเลยแม้แต่บรรทัดเดียว**
เพราะไฟล์นี้อยู่ใต้ `app/javascript/controllers/` อยู่แล้วซึ่งถูก `pin_all_from` ครอบไว้ตั้งแต่
ตอน `rails new` — แค่รัน generator แล้วเซฟไฟล์ Propshaft/importmap-rails จะมองเห็นและ pin ให้
โดยอัตโนมัติทันที (ยืนยันจากการ curl หน้าเว็บจริงแล้วเห็น
`"controllers/toggle_controller": "/assets/controllers/toggle_controller-<hash>.js"` โผล่ใน
`<script type="importmap">` โดยไม่ได้แก้ config เอง)

### กฎการแปลงชื่อไฟล์ → ชื่อ controller

Stimulus แปลงชื่อไฟล์เป็นชื่อที่ใช้ใน `data-controller` โดยอัตโนมัติตามกฎนี้:

1. ตัด suffix `_controller` ออก
2. แปลง underscore (`_`) เป็นขีด (`-`)
3. ถ้าอยู่ในโฟลเดอร์ย่อย แปลง `/` เป็น `--` (สอง dash) เพื่อบอก namespace

| ชื่อไฟล์ | ชื่อ `data-controller` |
|---|---|
| `toggle_controller.js` | `toggle` |
| `character_counter_controller.js` | `character-counter` |
| `users/list_controller.js` | `users--list` |

กฎนี้อ้างอิงจาก source code จริงของ gem (`Stimulus::Manifest`):

```ruby
tag_name = module_path.remove(/_controller/).gsub(/_/, "-").gsub(/\//, "--")
```

### ทางเลือกเก่า: `bin/rails stimulus:manifest:update`

ก่อนที่ importmap-rails จะมี `eagerLoadControllersFrom` อัตโนมัติ (และในโปรเจกต์ที่ยังใช้
bundler อย่าง esbuild/webpack แทน importmap) มี rake task ชื่อ `stimulus:manifest:update`
ที่จะ**เขียนทับ** `controllers/index.js` ใหม่ทั้งไฟล์ ให้ import และ register ทุก controller
แบบเจาะจงทีละบรรทัด ทดสอบรันจริงในแอป importmap:

```bash
bin/rails stimulus:manifest:update
```

```js
// controllers/index.js ที่ได้ (เขียนทับของเดิม)
// This file is auto-generated by ./bin/rails stimulus:manifest:update
// Run that command whenever you add a new controller or create them with
// ./bin/rails generate stimulus controllerName

import { application } from "./application"

import CharacterCounterController from "./character_counter_controller"
application.register("character-counter", CharacterCounterController)

import ConfirmController from "./confirm_controller"
application.register("confirm", ConfirmController)
...
```

> **ข้อควรระวังที่ยืนยันได้จริง (สำคัญมาก):** ในสแตกของ Part นี้ (importmap-rails + Propshaft)
> **ห้ามรัน `stimulus:manifest:update` แล้วปล่อยไฟล์นี้ไว้แบบนี้** — ทดสอบจริงพบว่าแอปพัง
> ทันที (`404 Not Found` ทุก controller, เมนู dropdown กดแล้วไม่ทำงาน) สาเหตุคือไฟล์ที่
> generate มาใช้ relative import แบบไม่มีนามสกุลไฟล์ (`import { application } from
> "./application"`) — วิธีนี้ถูกออกแบบมาสำหรับ workflow ที่มี **bundler** (esbuild/webpack)
> คอย resolve extensionless import ให้ แต่ browser ธรรมดาจะ resolve relative import ตาม URL
> ของไฟล์ปัจจุบันตรงๆ กลายเป็นไปขอไฟล์ `/assets/controllers/application` (ไม่มี `.js` และไม่มี
> fingerprint hash) ซึ่ง Propshaft หาไม่เจอเพราะไฟล์จริงถูก fingerprint เป็น
> `application-<hash>.js` เสมอ (ตามที่เรียนใน Part 029) **สรุปคือ: ในสแตก importmap-rails ค่า
> default ที่ generate มาให้ตอน `rails new` (ไฟล์ `index.js` ที่ใช้ `eagerLoadControllersFrom`)
> คือทางที่ถูกต้องอยู่แล้ว ไม่ต้องรัน `stimulus:manifest:update` เลย** — task นี้มีประโยชน์เฉพาะ
> โปรเจกต์ที่ตั้งค่า Stimulus ผ่าน esbuild/webpack ที่มี build step จริงๆ เท่านั้น

---

## Step 523: Controller Lifecycle — `data-controller`, `connect()`, `disconnect()`

### `data-controller` ผูก element เข้ากับ JS class

หัวใจของ Stimulus คือ HTML attribute `data-controller` — ใส่ชื่อ controller (ตามกฎการแปลงชื่อ
ใน Step 522) ลงใน element ไหนก็ตาม แล้ว Stimulus จะสร้าง instance ของ controller class นั้น
ผูกกับ element นั้นให้อัตโนมัติทันทีที่ element ปรากฏบนหน้า:

```erb
<div data-controller="toggle">
  <!-- element นี้ถูก "เกาะ" ด้วย instance ของ ToggleController -->
</div>
```

ไม่ต้องเขียน `document.querySelector` หรือ `addEventListener` ผูกเองเลยแม้แต่บรรทัดเดียว —
Stimulus ใช้ `MutationObserver` เฝ้าดู DOM ทั้งหน้าอยู่เบื้องหลังตลอดเวลา ทุกครั้งที่มี element
ที่มี `data-controller` โผล่ขึ้นมาใหม่ (ไม่ว่าจะมาจาก page load ปกติ, Turbo Drive/Frame/Stream,
หรือ JavaScript อื่นแทรก HTML เข้ามาตรงๆ) controller จะถูกสร้างและเชื่อมต่อให้ทันทีโดยอัตโนมัติ

### `connect()` และ `disconnect()`

Controller class สืบทอดจาก `Controller` ของ `@hotwired/stimulus` มี lifecycle callback สำคัญ
2 ตัวที่ใช้บ่อยที่สุด:

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    console.log("toggle controller connected", this.element)
  }

  disconnect() {
    console.log("toggle controller disconnected")
  }
}
```

- **`connect()`** ถูกเรียกทันทีที่ element ที่มี `data-controller="toggle"` ถูกเพิ่มเข้า DOM —
  ใช้ทำ initialization เช่น ตั้งค่าเริ่มต้น, ผูก event listener ที่ทำเองนอกเหนือจาก
  `data-action` (จะเรียนใน Step 524), หรือ fetch ข้อมูลเริ่มต้น
- **`disconnect()`** ถูกเรียกทันทีที่ element นั้นถูก**เอาออกจาก DOM** — ใช้ cleanup เช่น ยกเลิก
  `setInterval`, ตัด event listener ที่ผูกกับ `window`/`document` ตรงๆ (ถ้าไม่เคลียร์จะเกิด
  memory leak เพราะ listener ยังค้างอ้างอิง element ที่ถูกลบไปแล้ว)
- `this.element` คือ DOM element ที่มี `data-controller` attribute นั้นอยู่ (คือ element ที่
  Stimulus ใช้เป็นจุดเกาะ ไม่ใช่ document ทั้งหน้า)

**ทดสอบจริง**: เขียนโค้ดข้างบน วาง `<div data-controller="toggle">` ไว้ในหน้า แล้วเปิด
DevTools console — เห็น log `"toggle controller connected"` ทันทีตอนโหลดหน้า จากนั้นทดสอบ
navigate ไปหน้าอื่นด้วยลิงก์ธรรมดา (ที่ Turbo Drive ควบคุมอยู่) แล้วกลับมาหน้าเดิม — เห็น
`connect()` ถูกเรียกใหม่ทุกครั้งที่ element กลับมาอยู่ใน DOM จริง ยืนยันว่า Stimulus ผูกกับ
"การมีอยู่ของ element ใน DOM" ไม่ใช่ผูกกับ "การโหลดหน้าเว็บ" (page load event) แบบที่ jQuery
ยุคเก่ามักทำผ่าน `$(document).ready()`

> **ทำไมเรื่องนี้สำคัญกับ Turbo:** เมื่อ Turbo Stream สั่ง `remove` element ที่มี
> `data-controller` ออกจากหน้า `disconnect()` จะถูกเรียกให้อัตโนมัติ และเมื่อ Turbo Stream
> `append`/`replace` เอา HTML ใหม่ที่มี `data-controller` เข้ามา `connect()` ก็จะถูกเรียกให้
> อัตโนมัติเช่นกัน — ทั้งหมดนี้เกิดขึ้นเองโดยเราไม่ต้องเขียนโค้ดเชื่อมระหว่าง Turbo กับ Stimulus
> แม้แต่บรรทัดเดียว นี่คือสิ่งที่ทำให้ทั้งสองระบบ "ทำงานเป็นทีมเดียวกัน" ตามที่อธิบายไว้ใน
> Step 521

### callback อื่นที่ควรรู้จักไว้ (ใช้น้อยกว่า)

```js
export default class extends Controller {
  initialize() {
    // เรียกครั้งเดียวตอน controller ถูกสร้างขึ้นครั้งแรก (ก่อน connect() เสมอ)
    // ต่างจาก connect() ตรงที่ initialize() เรียกแค่ครั้งเดียวตลอดอายุของ instance
    // ส่วน connect()/disconnect() เรียกซ้ำได้ทุกครั้งที่ element เข้า/ออก DOM
  }
}
```

ในทางปฏิบัติ 90%+ ของ controller ที่เขียนจริงใช้แค่ `connect()` และ `disconnect()` เท่านั้น
`initialize()` ใช้เมื่อต้องการทำอะไรบางอย่างแค่ครั้งเดียวจริงๆ ไม่ต้องทำซ้ำเวลา
reconnect (เช่น ตั้งค่า config object ที่ไม่เปลี่ยนแปลงตามการเชื่อมต่อ)

---

## Step 524: Action — `data-action` เชื่อม DOM Event เข้ากับ Method ของ Controller

### รูปแบบพื้นฐาน: `event->controller#method`

```erb
<div data-controller="toggle">
  <button data-action="click->toggle#toggle">เมนู</button>
</div>
```

อ่านจากขวาไปซ้าย: เมื่อเกิด **`click`** event บน element นี้ ให้เรียก method ชื่อ **`toggle`**
ของ controller ชื่อ **`toggle`** ที่เกาะอยู่กับ element ใดก็ตามที่ใกล้ที่สุด (closest ancestor
รวมตัวเองด้วย) ที่มี `data-controller="toggle"`

```js
// app/javascript/controllers/toggle_controller.js
export default class extends Controller {
  toggle() {
    console.log("ถูกคลิก!")
  }
}
```

### Event shorthand — บาง element ไม่ต้องระบุชื่อ event

Stimulus มี **default event** ให้กับบาง HTML element ที่รู้กันอยู่แล้วว่าปกติจะฟัง event
อะไร ทำให้เขียนสั้นลงได้:

| Element | Default event |
|---|---|
| `<button>` (submit ปกติ), `<a>` | `click` |
| `<form>` | `submit` |
| `<input>`, `<textarea>`, `<select>` | `input` |

```erb
<!-- เขียนเต็ม -->
<button data-action="click->toggle#toggle">เมนู</button>

<!-- เขียนย่อ (ผลลัพธ์เหมือนกันทุกประการ เพราะ button คลิก default เป็น click อยู่แล้ว) -->
<button data-action="toggle#toggle">เมนู</button>
```

ในบทเรียนนี้จะเขียนแบบเต็ม (`click->toggle#toggle`) เสมอเพื่อความชัดเจนเวลาอ่านโค้ด แม้จะย่อ
ได้ก็ตาม — เป็นแนวปฏิบัติที่แนะนำในทีมจริงหลายทีมเพราะอ่านแล้วรู้ทันทีว่าฟัง event อะไรโดยไม่
ต้องจำ default table ด้านบน

### หลาย action บน element เดียวกัน — คั่นด้วยช่องว่าง

```erb
<button data-action="mouseenter->highlight#add mouseleave->highlight#remove">
  ชี้เมาส์เพื่อไฮไลต์
</button>
```

ทดสอบจริงแล้ว: เมื่อเมาส์เข้า (`mouseenter`) จะเรียก `highlight#add`, เมื่อเมาส์ออก
(`mouseleave`) จะเรียก `highlight#remove` — เขียน 2 พฤติกรรมบน element เดียวกันได้โดยไม่ต้อง
สร้าง controller แยก ไม่ต้องมี method ที่เช็ค event type เอง

### ตัวอย่างการรับ `event` object ใน method

Method ที่ผูกกับ `data-action` จะได้รับ `Event` object มาตรฐานของ browser เป็น argument
เสมอ ใช้ `event.preventDefault()`, `event.target`, `event.params` (จะพูดถึงเพิ่มใน Step 526)
ได้ตามปกติ:

```js
// app/javascript/controllers/confirm_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { message: { type: String, default: "แน่ใจหรือไม่?" } }

  check(event) {
    if (!window.confirm(this.messageValue)) {
      event.preventDefault()   // ยกเลิก submit ถ้าผู้ใช้กด "ยกเลิก" ใน confirm dialog
    }
  }
}
```

```erb
<form data-controller="confirm"
      data-confirm-message-value="ต้องการลบจริงหรือไม่?"
      data-action="submit->confirm#check"
      action="/posts/1" method="post">
  <button type="submit">ลบโพสต์</button>
</form>
```

**ทดสอบจริงผ่าน headless Chromium**: คลิกปุ่ม "ลบโพสต์" → เกิด native `confirm()` dialog
ขึ้นมาจริงพร้อมข้อความ `"ต้องการลบจริงหรือไม่?"` (ตรงตามค่าใน `data-confirm-message-value`)
→ กด "ยกเลิก" (dismiss) → ฟอร์ม**ไม่ submit** (URL ของหน้ายังคงเดิม ไม่มีการ navigate ออกไป
เลย) ยืนยันว่า `event.preventDefault()` ทำงานถูกต้องตามที่คาด

### กฎการหา controller เป้าหมาย: ต้องเป็น ancestor (หรือตัวเอง) เท่านั้น

`data-action` จะมองหา controller ที่ระบุชื่อไว้จาก element ปัจจุบันไล่ขึ้นไปหา **ancestor
ที่ใกล้ที่สุด** (closest ancestor รวมตัวเองด้วย) ที่มี `data-controller` ตรงชื่อ — ไม่จำเป็นต้อง
อยู่ element เดียวกันกับ `data-controller` เสมอไป:

```erb
<div data-controller="toggle">
  <!-- data-action อยู่คนละ element กับ data-controller ก็ได้ ตราบใดที่เป็น descendant -->
  <button data-action="click->toggle#toggle">เมนู</button>
  <ul data-toggle-target="menu" hidden>...</ul>
</div>
```

นี่คือรูปแบบที่ใช้บ่อยที่สุดในทางปฏิบัติ: ใส่ `data-controller` ไว้ที่ element แม่ (wrapper)
แล้วใส่ `data-action`/`data-*-target` กระจายอยู่ตาม element ลูกต่างๆ ข้างใน — รายละเอียดเรื่อง
target จะเรียนต่อใน Step ถัดไป

---

## Step 525: Target — อ้างอิง Element ลูกโดยไม่ต้อง `querySelector` เอง

### ปัญหาที่ target แก้

ถ้าไม่มี target เวลา controller ต้องการอ้างอิง element ลูกที่เจาะจง (เช่น เมนู `<ul>` ที่ต้อง
ซ่อน/แสดง) จะต้องเขียน `this.element.querySelector(".dropdown-menu")` เอง — วิธีนี้ผูกโค้ด
JavaScript เข้ากับโครงสร้าง/class ของ HTML แน่นเกินไป (ถ้าแก้ HTML แล้วลืมแก้ selector ใน JS
จะพังแบบเงียบๆ) และอ่านไม่รู้เรื่องว่า element ไหนสำคัญกับ controller บ้างถ้าไม่เปิดไฟล์ JS
มาดู

### ประกาศ target ด้วย `static targets`

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]

  connect() {
    console.log(this.menuTarget)    // <ul data-toggle-target="menu" ...>
    console.log(this.buttonTarget)  // <button data-toggle-target="button" ...>
  }
}
```

ฝั่ง HTML ผูกด้วย `data-<controller-name>-target="<target-name>"`:

```erb
<div data-controller="toggle">
  <button data-toggle-target="button" data-action="click->toggle#toggle">เมนู</button>

  <ul data-toggle-target="menu" hidden>
    <li>โปรไฟล์</li>
    <li>ตั้งค่า</li>
  </ul>
</div>
```

ประกาศ `static targets = ["menu", "button"]` แล้ว Stimulus จะสร้าง **getter** ให้อัตโนมัติ
2 ตัวต่อชื่อ target หนึ่งชื่อ:

- **`this.<name>Target`** — คืน element ตัวแรกที่ match (ถ้าไม่มีเลยจะ throw error ทันที
  ป้องกันบัคจาก `undefined` ที่ debug ยาก)
- **`this.has<Name>Target`** — คืน `true`/`false` ว่ามี target นั้นอยู่ในหน้าปัจจุบันหรือไม่
  (ใช้ guard ก่อนเรียก `xxxTarget` เวลา target นั้นไม่ได้บังคับต้องมีเสมอ)
- **`this.<name>Targets`** — คืน **Array** ของ element ที่ match **ทุกตัว** (ใช้เมื่อมี target
  ชื่อเดียวกันซ้ำหลาย element เช่น รายการ checkbox หลายอัน)

```js
export default class extends Controller {
  static targets = ["menu", "button"]

  connect() {
    if (this.hasMenuTarget) {
      console.log("มี target menu อยู่จริง")
    }
  }
}
```

### สร้างเมนู Dropdown ที่ทำงานได้จริงด้วย target + action

รวม target เข้ากับ action ที่เรียนมาใน Step 524:

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]

  toggle() {
    this.menuTarget.hidden = !this.menuTarget.hidden
  }

  hide() {
    this.menuTarget.hidden = true
  }
}
```

```erb
<div data-controller="toggle">
  <button data-toggle-target="button" data-action="click->toggle#toggle" aria-expanded="false">
    เมนู
  </button>

  <ul data-toggle-target="menu" hidden>
    <li><a href="#">โปรไฟล์</a></li>
    <li><a href="#">ตั้งค่า</a></li>
    <li><a data-action="click->toggle#hide" href="#">ออกจากระบบ</a></li>
  </ul>
</div>
```

**ทดสอบจริงผ่าน headless Chromium** ตามลำดับ:

1. โหลดหน้า → `<ul>` มี `hidden` อยู่ (เมนูซ่อนอยู่) ✅ ตรวจสอบผ่าน
   `page.isHidden('ul')` ได้ `true`
2. คลิกปุ่ม "เมนู" → `<ul>` แสดงขึ้นมาทันที (`hidden` ถูกลบ) ✅
3. คลิกลิงก์ "ออกจากระบบ" (ที่ผูก `data-action="click->toggle#hide"` แยกไว้) → เมนูซ่อนกลับ
   ไปอีกครั้ง ✅

สังเกตว่าไม่มี `document.querySelector` แม้แต่บรรทัดเดียวในโค้ด JavaScript ทั้งหมด — ทุกการ
อ้างอิงมาจาก `this.menuTarget`/`this.buttonTarget` ที่ผูกกับ HTML ผ่าน `data-*-target`
ตรงๆ ทำให้เวลาอ่านไฟล์ HTML ก็รู้ทันทีว่า element ไหน "สำคัญ" กับ JavaScript บ้าง (ต่างจาก
class ธรรมดาที่อาจจะใช้แค่ทำสไตล์ หรือใช้ทั้งสไตล์และ JS ปนกันจนแยกไม่ออก)

---

## Step 526: Value — State ที่มี Type ชัดเจน ซิงค์กับ DOM Attribute แบบ Reactive

### ปัญหาที่ value แก้

Controller มักต้องเก็บ "สถานะ" บางอย่างไว้ เช่น เมนู dropdown เปิดอยู่หรือปิดอยู่, ตัวเลข limit
ของ character counter, endpoint URL ที่ต้องยิง fetch ไป — ถ้าเก็บไว้เป็นแค่ instance property
ธรรมดาของ JavaScript (`this.open = false`) จะมีปัญหา 2 อย่าง: (1) ค่าจะหายไปทันทีที่ Turbo
render element นั้นใหม่ (เพราะสร้าง controller instance ใหม่) และ (2) ไม่มีทางกำหนดค่าเริ่มต้น
ที่ต่างกันได้จากฝั่ง HTML/server เลย (เช่น อยากให้เมนูบางอันเปิดค้างไว้ตั้งแต่แรกตามเงื่อนไข
ของข้อมูลใน database)

**Value** คือกลไกของ Stimulus ที่แก้ปัญหานี้โดยตรง โดยเก็บ state ไว้เป็น **DOM attribute**
(`data-<controller>-<name>-value="..."`) ไม่ใช่ instance property — ทำให้ state คงอยู่ใน HTML
เอง (server กำหนดค่าเริ่มต้นได้ตรงๆ ผ่าน ERB) และ debug ง่ายมาก (เปิด DevTools ดู HTML ก็เห็น
state ปัจจุบันได้ทันที)

### ประกาศด้วย `static values` พร้อมระบุ type

```js
// app/javascript/controllers/toggle_controller.js
export default class extends Controller {
  static values = { open: { type: Boolean, default: false } }
}
```

ฝั่ง HTML:

```erb
<div data-controller="toggle" data-toggle-open-value="true">
  <!-- ถ้าไม่ใส่ attribute นี้เลย จะใช้ default: false ตามที่ประกาศไว้ -->
</div>
```

Type ที่ Stimulus รองรับ: `Boolean`, `Number`, `String`, `Array`, `Object` — Stimulus จะแปลง
string ใน HTML attribute ให้เป็น type ที่ถูกต้องให้อัตโนมัติ (`"true"` → `true` จริงๆ ไม่ใช่
string, `"20"` → `20` เป็น Number จริงๆ ไม่ใช่ string) ทำให้เวลาเทียบค่าใน JavaScript ด้วย `===`
ทำงานถูกต้องโดยไม่ต้อง `parseInt`/`=== "true"` เอง

### getter/setter ที่ Stimulus สร้างให้อัตโนมัติ

ประกาศ `static values = { open: Boolean }` แล้วได้:

- **`this.openValue`** — getter/**setter** อ่าน/เขียนค่าได้ตรงๆ การ**เขียน**ค่าจะไปอัปเดต
  `data-toggle-open-value` ใน DOM ให้อัตโนมัติทันที (ซิงค์สองทาง)
- **`this.hasOpenValue`** — `true`/`false` ว่ามี attribute นี้อยู่จริงหรือไม่ (มีประโยชน์เมื่อ
  ไม่ได้ตั้ง `default` ไว้)
- **`openValueChanged(value, previousValue)`** — callback พิเศษที่ Stimulus **เรียกให้
  อัตโนมัติทุกครั้งที่ค่าเปลี่ยน** (ทั้งตอน `connect()` ครั้งแรก และทุกครั้งที่มีการเขียนค่า
  ใหม่ผ่าน `this.openValue = ...` หรือแม้แต่ถ้ามีโค้ดอื่นไปแก้ attribute ใน DOM ตรงๆ)

### สร้างเมนู Dropdown เวอร์ชันเต็ม ด้วย value แทน target ที่ toggle ตรงๆ

รวม value เข้ากับโค้ดเดิมจาก Step 525 ให้ state ชัดเจนขึ้น (แยก "การเก็บสถานะ" ออกจาก "การ
อัปเดตหน้าจอ" ตาม pattern ที่ทำให้ debug ง่ายและ react ต่อ state เปลี่ยนแปลงได้จากหลายทาง
ไม่ใช่แค่จากการคลิก):

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  hide() {
    this.openValue = false
  }

  openValueChanged(open) {
    this.menuTarget.hidden = !open
    this.buttonTarget.setAttribute("aria-expanded", open)
  }
}
```

**ทดสอบจริง**: หลังคลิกปุ่ม "เมนู" ตรวจสอบ HTML ใน DOM ด้วย DevTools เห็น
`data-toggle-open-value="true"` ปรากฏขึ้นเองบน `<div data-controller="toggle">` ทันที (ไม่ได้
เขียน `setAttribute` ตรงนี้เองเลยในโค้ด — Stimulus จัดการให้อัตโนมัติจากการเขียน
`this.openValue = true`) และ `aria-expanded="true"` ก็ถูกอัปเดตผ่าน `openValueChanged()` ที่
ถูกเรียกให้อัตโนมัติทันทีที่ค่า value เปลี่ยน

> **ทำไมแยก `toggle()`/`hide()` ออกจาก `openValueChanged()`:** สังเกตว่า `toggle()` และ
> `hide()` ไม่ได้ไปยุ่งกับ `this.menuTarget.hidden` ตรงๆ เลย มันแค่ "ประกาศเจตนา" ผ่านการ
> เปลี่ยนค่า value ส่วน `openValueChanged()` ทำหน้าที่ "แปลง state เป็นสิ่งที่ตาเห็น" เพียง
> จุดเดียว ข้อดีคือถ้ามีทางอื่นมาเปลี่ยน `openValue` ในอนาคต (เช่น ปุ่มลัดคีย์บอร์ด, event จาก
> controller อื่นผ่าน outlet ใน Step 530) หน้าจอก็จะอัปเดตถูกต้องเสมอโดยไม่ต้องเขียนโค้ด
> อัปเดต UI ซ้ำที่จุดใหม่นั้นอีกเลย — เป็นแนวคิดเดียวกับ "state → UI เป็น pure function ของ
> state" ที่ React/Vue ใช้ เพียงแต่ Stimulus ทำแบบเบาๆ ไม่มี virtual DOM

---

## Step 527: Class — สลับ CSS Class โดยไม่ Hardcode ชื่อ Class ไว้ใน JS

### ปัญหาที่ class (Stimulus) แก้

วิธีที่ตรงไปตรงมาที่สุดในการ toggle CSS class คือเขียนชื่อ class ตรงๆ ในโค้ด JavaScript:

```js
// วิธีที่ไม่แนะนำ — hardcode ชื่อ class ไว้ใน JS
this.buttonTarget.classList.add("active")
```

ปัญหาคือถ้าโปรเจกต์เปลี่ยนไปใช้ CSS framework คนละตัว (เช่นจาก custom CSS ไปเป็น Tailwind ใน
Part 054) หรือแค่อยากเปลี่ยนชื่อ class ให้สอดคล้องกับ design system ใหม่ ต้องไปไล่แก้ไฟล์
JavaScript ทุกไฟล์ที่ hardcode ชื่อ class ไว้ — ผูก JavaScript (logic) เข้ากับ CSS (การแสดงผล)
แน่นเกินไป

### ประกาศด้วย `static classes` แล้วกำหนดชื่อจริงจาก HTML

```js
// app/javascript/controllers/toggle_controller.js
export default class extends Controller {
  static classes = ["active"]
}
```

```erb
<div data-controller="toggle" data-toggle-active-class="active">
  <button data-toggle-target="button">เมนู</button>
</div>
```

ประกาศ `static classes = ["active"]` แล้วได้ **`this.activeClass`** เป็น string ที่มาจากค่า
`data-toggle-active-class` ใน HTML — โค้ด JavaScript ใช้ `this.activeClass` แทนการ hardcode
string `"active"` ตรงๆ:

```js
openValueChanged(open) {
  this.menuTarget.hidden = !open
  this.buttonTarget.classList.toggle(this.activeClass, open)
  this.buttonTarget.setAttribute("aria-expanded", open)
}
```

ประโยชน์ที่ได้จริง: ถ้าอยากเปลี่ยนชื่อ class เป็น `is-open` หรือ (เมื่อย้ายไปใช้ Tailwind ใน
Part 054) เปลี่ยนเป็นหลาย class พร้อมกันอย่าง `"ring-2 ring-blue-500"` ก็แก้แค่ HTML attribute
`data-toggle-active-class="..."` จุดเดียว **ไม่ต้องแตะไฟล์ JavaScript เลยแม้แต่บรรทัดเดียว** —
controller ตัวเดียวกันนี้ยังเอาไปใช้ซ้ำกับหน้าอื่นที่อยากได้ class คนละชื่อได้ทันทีด้วย เพราะ
ชื่อ class ไม่ได้ถูกผูกตายตัวไว้ในโค้ด

**ทดสอบจริง**: หลังคลิกปุ่ม "เมนู" ตรวจสอบผ่าน
`document.querySelector('button').classList.contains('active')` ได้ `true` ทันที ยืนยันว่า
`this.activeClass` อ่านค่า `"active"` จาก `data-toggle-active-class` มาใช้ถูกต้อง

### `classes` รองรับหลาย class ต่อ 1 ชื่อได้เช่นกัน

```erb
<div data-controller="toggle" data-toggle-active-class="ring-2 ring-blue-500 shadow-lg">
```

`this.activeClass` จะได้ string `"ring-2 ring-blue-500 shadow-lg"` มาทั้งก้อน และเมื่อส่งเข้า
`classList.toggle(this.activeClass, ...)` — ถ้ามีมากกว่า 1 class ต้องใช้
`classList.add(...this.activeClass.split(" "))` แทน เพราะ `classList.toggle()` รับได้แค่
class เดียวต่อครั้ง (เป็นข้อจำกัดของ Web API `classList` เอง ไม่ใช่ของ Stimulus)

---

## Step 528: สร้าง Controller ใช้งานจริงทีละขั้น — เมนู Dropdown แบบครบวงจร

มาประกอบทุกอย่างที่เรียนมา (controller, action, target, value, class) เข้าด้วยกันเป็น
controller เดียวที่ใช้งานจริงได้ ทีละขั้นตอน โดยใช้ generator ตามที่เรียนใน Step 522 เสมอ
(ไม่เขียนไฟล์มือ):

### ขั้นที่ 1: generate controller

```bash
bin/rails generate stimulus toggle
#      create  app/javascript/controllers/toggle_controller.js
```

### ขั้นที่ 2: วาง HTML ที่จะ "เสริม" พฤติกรรมให้

เริ่มจาก HTML ล้วนๆ ที่ทำงานได้แม้ไม่มี JavaScript เลย (progressive enhancement — เมนูก็ยัง
เป็นลิงก์ธรรมดาที่กดแล้วไปหน้าอื่นได้อยู่ ถึงแม้ JavaScript จะโหลดไม่สำเร็จก็ตาม):

```erb
<%# app/views/layouts/_user_menu.html.erb %>
<div class="user-menu">
  <button aria-expanded="false">สวัสดี, <%= current_user.name %></button>

  <ul>
    <li><%= link_to "โปรไฟล์", profile_path %></li>
    <li><%= link_to "ตั้งค่า", settings_path %></li>
    <li><%= link_to "ออกจากระบบ", logout_path, data: { turbo_method: :delete } %></li>
  </ul>
</div>
```

### ขั้นที่ 3: เพิ่ม `data-controller` และ `data-*-target`

```erb
<div class="user-menu" data-controller="toggle" data-toggle-active-class="active">
  <button data-toggle-target="button"
          data-action="click->toggle#toggle"
          aria-expanded="false">
    สวัสดี, <%= current_user.name %>
  </button>

  <ul data-toggle-target="menu" hidden>
    <li><%= link_to "โปรไฟล์", profile_path %></li>
    <li><%= link_to "ตั้งค่า", settings_path %></li>
    <li><%= link_to "ออกจากระบบ", logout_path,
          data: { turbo_method: :delete, action: "click->toggle#hide" } %></li>
  </ul>
</div>
```

สังเกตว่า `link_to` ของ Rails รองรับการใส่ `data: { action: "..." }` ปนกับ
`data: { turbo_method: ... }` ที่เคยเรียนใน Part 051 ได้เลยในแฮชเดียวกัน — Turbo กับ Stimulus
อ่าน `data-*` attribute คนละชุดกัน ไม่ชนกัน

### ขั้นที่ 4: เขียน controller

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]
  static values = { open: { type: Boolean, default: false } }
  static classes = ["active"]

  toggle() {
    this.openValue = !this.openValue
  }

  hide() {
    this.openValue = false
  }

  openValueChanged(open) {
    this.menuTarget.hidden = !open
    this.buttonTarget.classList.toggle(this.activeClass, open)
    this.buttonTarget.setAttribute("aria-expanded", open)
  }
}
```

### ขั้นที่ 5: ปิดเมนูอัตโนมัติเมื่อคลิกนอกเมนู (เพิ่ม UX ที่คาดหวังจริง)

ผู้ใช้คาดหวังว่าคลิกที่ไหนก็ได้นอกเมนูแล้วเมนูต้องปิด ไม่ใช่แค่คลิกปุ่ม "ออกจากระบบ" เท่านั้น
ถึงต้อง fallback ไปใช้ `connect()`/`disconnect()` ผูก listener กับ `document` เอง (เพราะ
`data-action` ผูก event ได้เฉพาะกับ element ที่อยู่ใน controller เท่านั้น ไม่ใช่ทั้งหน้า):

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "button"]
  static values = { open: { type: Boolean, default: false } }
  static classes = ["active"]

  connect() {
    // ผูก handler เก็บไว้เป็น bound function เพื่อให้ removeEventListener ถอดตัวเดียวกันได้จริง
    this.closeOnClickOutside = this.closeOnClickOutside.bind(this)
    document.addEventListener("click", this.closeOnClickOutside)
  }

  disconnect() {
    // สำคัญมาก: ต้องถอด listener ตอน disconnect ไม่งั้นเกิด memory leak
    // และ handler จะยังทำงานอยู่แม้ element นี้จะถูกลบออกจาก DOM ไปแล้ว
    document.removeEventListener("click", this.closeOnClickOutside)
  }

  toggle() {
    this.openValue = !this.openValue
  }

  hide() {
    this.openValue = false
  }

  closeOnClickOutside(event) {
    if (this.openValue && !this.element.contains(event.target)) {
      this.hide()
    }
  }

  openValueChanged(open) {
    this.menuTarget.hidden = !open
    this.buttonTarget.classList.toggle(this.activeClass, open)
    this.buttonTarget.setAttribute("aria-expanded", open)
  }
}
```

จุดสำคัญที่ต้องระวัง (เป็นข้อผิดพลาดที่พบบ่อยที่สุดตอนเขียน Stimulus controller จริง): **ต้อง
`.bind(this)` เก็บ reference ของ handler ไว้ในตัวแปร แล้วใช้ reference ตัวเดียวกันทั้งตอน
`addEventListener` และ `removeEventListener`** ถ้าเขียน
`document.removeEventListener("click", this.closeOnClickOutside.bind(this))` ตรงๆ ใน
`disconnect()` จะ**ไม่ทำงาน** เพราะ `.bind()` สร้าง function ใหม่ทุกครั้งที่เรียก ทำให้
reference ไม่ตรงกับตัวที่ผูกไว้ตอน `connect()` — listener จะไม่ถูกถอดออกจริง และเกิด memory
leak สะสมทุกครั้งที่ element ถูกสร้างใหม่ (เช่นทุกครั้งที่ Turbo navigate ไปมา)

---

## Step 529: หลาย Controller บน Element เดียวกัน/ที่ซ้อนกัน

### ประกาศหลาย controller คั่นด้วยช่องว่างใน `data-controller`

```erb
<div data-controller="toggle highlight"
     data-toggle-active-class="active"
     data-highlight-on-class="on-highlight">
  <button data-toggle-target="button"
          data-action="click->toggle#toggle mouseenter->highlight#add mouseleave->highlight#remove">
    เปิดเมนู (ลองชี้เมาส์ด้วย)
  </button>

  <ul data-toggle-target="menu" hidden>
    <li>รายการ 1</li>
    <li>รายการ 2</li>
  </ul>
</div>
```

```js
// app/javascript/controllers/highlight_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = ["on"]

  add() {
    this.element.classList.add(this.onClass)
  }

  remove() {
    this.element.classList.remove(this.onClass)
  }
}
```

`data-controller="toggle highlight"` สั่งให้ Stimulus สร้าง controller instance **2 ตัวแยก
กัน** ผูกกับ element เดียวกัน — `ToggleController` กับ `HighlightController` ทำงานเป็นอิสระต่อ
กันโดยสิ้นเชิง ไม่รู้จักกันเลย (ไม่มี state หรือ method ใดใช้ร่วมกัน) การผูกหลาย action ที่ชี้
ไปคนละ controller บน element เดียวกัน (`data-action="click->toggle#toggle
mouseenter->highlight#add mouseleave->highlight#remove"`) ก็ทำได้ปกติ เพราะ `data-action`
อ่านทีละ token คั่นด้วยช่องว่างเสมอ ไม่สนใจว่า controller แต่ละ token จะเป็นตัวเดียวกันหรือไม่

### ข้อควรระวังที่ยืนยันได้จริง: `this.element` ผูกกับ controller ของตัวเอง ไม่ใช่ตำแหน่งของ `data-action`

จุดที่มือใหม่สับสนบ่อยที่สุด (และพิสูจน์ได้จริงจากการทดสอบ): `this.element` ใน controller
หมายถึง **element ที่มี `data-controller` ของ controller นั้นอยู่** เสมอ **ไม่ใช่** element ที่
เขียน `data-action` ไว้ ถึงแม้ทั้งสอง controller ในตัวอย่างข้างบนจะ "ถูกกระตุ้น" จาก event บน
`<button>` เดียวกัน แต่ `this.element` ของทั้งคู่ชี้ไปที่ `<div>` (element ที่มี
`data-controller="toggle highlight"`) **ไม่ใช่** `<button>`

ทดสอบจริงผ่าน headless Chromium ยืนยันเรื่องนี้ชัดเจน: หลัง hover เมาส์เข้า `<button>` แล้ว
เช็ค `document.querySelector('button').classList.contains('on-highlight')` ได้ `false` แต่
เช็คที่ `<div>` (parent) กลับพบ class `on-highlight` ติดอยู่จริง — ยืนยันว่า
`this.element.classList.add(this.onClass)` ใน `HighlightController#add()` เติม class ให้
`<div>` (เจ้าของ `data-controller`) ไม่ใช่ `<button>` (เจ้าของ `data-action`) ตามที่โค้ดตั้งใจ
ไว้จริง

> **บทเรียนที่ได้:** ถ้าต้องการให้ class ไปติดที่ element ที่ถูกคลิกจริงๆ (เช่น `<button>`)
> ไม่ใช่ที่ `this.element` ต้องใช้ `event.currentTarget` ที่รับมาจาก parameter ของ method
> แทน (`add(event) { event.currentTarget.classList.add(this.onClass) }`) หรือใช้ target
> (`data-highlight-target="button"` แล้วอ้างผ่าน `this.buttonTarget`) แทนการพึ่ง
> `this.element` เมื่อ controller กับ element ที่ต้องการแก้ไขไม่ใช่ตัวเดียวกัน

### วาง controller ซ้อนกัน (nested) — ancestor ที่ใกล้ที่สุดชนะเสมอ

```erb
<div data-controller="toggle" data-toggle-active-class="outer-active">
  <button data-action="click->toggle#toggle">ปุ่มนอก</button>

  <div data-controller="toggle" data-toggle-active-class="inner-active">
    <button data-action="click->toggle#toggle">ปุ่มใน</button>
  </div>
</div>
```

เมื่อคลิก "ปุ่มใน" — `data-action="click->toggle#toggle"` จะหา controller ชื่อ `toggle` จาก
ancestor **ที่ใกล้ที่สุด** ก่อน ซึ่งคือ `<div>` ชั้นในสุด ไม่ใช่ชั้นนอก — `ToggleController`
ทั้งสอง instance (ชั้นนอกกับชั้นใน) เป็นคนละ instance กันโดยสมบูรณ์ ไม่แชร์ state ใดๆ ต่อกัน
แม้จะเป็น class เดียวกันก็ตาม กฎนี้เหมือนกับการ resolve CSS scope หรือ variable scoping ใน
โปรแกรมมิ่งทั่วไป — "ใกล้สุดชนะ" เสมอ

---

## Step 530: Outlet — การสื่อสารข้าม Controller (หัวข้อขั้นสูง)

Target (Step 525) ใช้อ้างอิง element **ภายใน controller เดียวกัน** เท่านั้น แต่บางสถานการณ์
controller หนึ่งจำเป็นต้อง**เรียก method ของ controller อีกตัวที่อยู่คนละ element กันโดย
สิ้นเชิง** (ไม่ใช่ ancestor/descendant กัน) เช่น ฟอร์มเขียนโพสต์ต้องการอัปเดตตัวเลขจำนวนคำใน
กล่องสรุปที่อยู่แยกกันคนละส่วนของหน้า — **Outlet** คือกลไกที่ Stimulus สร้างมาสำหรับกรณีนี้
โดยเฉพาะ

### ประกาศด้วย `static outlets`

```js
// app/javascript/controllers/post_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["body"]
  static outlets = ["word-count"]   // ชื่อ controller อีกตัวที่จะเชื่อมถึง (dash-case)

  updateWordCount() {
    const words = this.bodyTarget.value.trim().split(/\s+/).filter(Boolean)
    this.wordCountOutlets.forEach((outlet) => outlet.set(words.length))
  }
}
```

```js
// app/javascript/controllers/word_count_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["output"]

  set(count) {
    this.outputTarget.textContent = `${count} คำ`
  }
}
```

ฝั่ง HTML ต้องระบุ **CSS selector** ที่ชี้ไปหา element ของ controller เป้าหมาย ผ่าน
`data-<controller>-<outlet-name>-outlet`:

```erb
<div data-controller="post-form" data-post-form-word-count-outlet="#word-count-box">
  <textarea data-post-form-target="body"
            data-action="input->post-form#updateWordCount"></textarea>
</div>

<!-- อยู่คนละที่กันโดยสิ้นเชิงในหน้า ไม่ใช่ descendant ของ div ด้านบนเลย -->
<div id="word-count-box" data-controller="word-count">
  <output data-word-count-target="output">0 คำ</output>
</div>
```

ประกาศ `static outlets = ["word-count"]` แล้วได้:

- **`this.wordCountOutlets`** — Array ของ **controller instance** (ไม่ใช่ DOM element) ทุกตัว
  ที่ match selector ใน `data-post-form-word-count-outlet` และมี `data-controller="word-count"`
  จริง เรียก method ของมันได้ตรงๆ เหมือนเรียก method ปกติ (`outlet.set(words.length)`)
- **`this.wordCountOutlet`** — ตัวแรกตัวเดียว (เผื่อกรณีที่รู้อยู่แล้วว่ามีแค่ตัวเดียว)
- **`this.hasWordCountOutlet`** — เช็คว่ามี outlet เชื่อมต่ออยู่จริงหรือไม่

**ทดสอบจริงผ่าน headless Chromium**: พิมพ์ข้อความ `"สวัสดี stimulus outlets ทำงานได้จริง"`
ลงในกล่อง `<textarea>` ของ `post-form` controller → กล่อง `<output>` ที่อยู่คนละ `<div>`
โดยสิ้นเชิง อัปเดตข้อความเป็น `"4 คำ"` ทันที ยืนยันว่าการสื่อสารข้าม controller ที่ไม่ได้เป็น
ancestor/descendant กันเลยทำงานได้จริงผ่าน outlet

> **เมื่อไหร่ควรใช้ outlet เทียบกับทางเลือกอื่น:** Outlet เหมาะกับกรณีที่ทั้งสอง controller
> "รู้จักกันโดยตรง" อยู่แล้วในเชิงหน้าที่ (post-form ควรรู้ว่ามี word-count อยู่) แต่ถ้าความ
> สัมพันธ์หลวมกว่านั้นมาก (controller A ไม่ควรรู้จัก B เลยในเชิง design) มักจะใช้วิธียิง custom
> DOM event ผ่าน `this.dispatch("wordCountChanged", { detail: { count } })` แล้วให้ B ฟัง
> event นั้นแทน (loose coupling กว่า outlet) — Outlet ใช้บ่อยพอสมควรในโปรเจกต์ขนาดกลางที่
> ความสัมพันธ์ระหว่าง controller ชัดเจนอยู่แล้ว แต่ไม่ใช่เครื่องมือที่ควรใช้เป็นค่าเริ่มต้นทุก
> ครั้งที่ต้องเชื่อม 2 controller เข้าด้วยกัน

---

## แบบฝึกหัด: Character Counter สำหรับช่อง Body ของโพสต์

### โจทย์

สร้าง Stimulus controller ชื่อ `character-counter` ผูกกับช่อง `body` (textarea) ของฟอร์ม
สร้าง/แก้ไขโพสต์ (`Post` model จาก Part 030) โดยต้องทำงานดังนี้:

1. แสดงจำนวนตัวอักษร**ที่เหลือ**ได้ (limit ลบด้วยจำนวนตัวอักษรที่พิมพ์ไปแล้ว) อัปเดตแบบ
   real-time ทุกครั้งที่พิมพ์ ไม่ต้องกด submit ก่อน
2. ถ้าพิมพ์เกิน limit (ตัวเลขเหลือติดลบ) ตัวเลขต้องเปลี่ยนเป็น**สีแดง**ทันที เพื่อเตือนผู้ใช้
   ก่อนที่จะกด submit แล้วโดน validation error จาก server
3. ค่า limit ต้องกำหนดได้จากฝั่ง HTML (ไม่ hardcode ไว้ใน JavaScript) เผื่อในอนาคตอยากใช้
   limit คนละค่ากับฟิลด์อื่น (เช่น ชื่อเรื่องอาจจำกัดสั้นกว่าเนื้อหา)

### เฉลย

#### ขั้นที่ 1: generate controller

```bash
bin/rails generate stimulus character_counter
#      create  app/javascript/controllers/character_counter_controller.js
```

(ชื่อไฟล์ `character_counter_controller.js` ตามกฎการแปลงชื่อใน Step 522 จะกลายเป็น
`data-controller="character-counter"` โดยอัตโนมัติ)

#### ขั้นที่ 2: เขียนโค้ด controller

```js
// app/javascript/controllers/character_counter_controller.js
import { Controller } from "@hotwired/stimulus"

// Connects to data-controller="character-counter"
export default class extends Controller {
  static targets = ["input", "count"]
  static values = { limit: Number }
  static classes = ["over"]

  connect() {
    this.updateCount()
  }

  updateCount() {
    const length = this.inputTarget.value.length
    const remaining = this.limitValue - length

    this.countTarget.textContent = remaining
    this.countTarget.classList.toggle(this.overClass, remaining < 0)
  }
}
```

อธิบายทีละส่วน:

- `static targets = ["input", "count"]` — `input` คือ `<textarea>` ที่ผู้ใช้พิมพ์,
  `count` คือ element ที่แสดงตัวเลขจำนวนตัวอักษรที่เหลือ
- `static values = { limit: Number }` — จำนวนตัวอักษรสูงสุด กำหนดจากฝั่ง HTML ผ่าน
  `data-character-counter-limit-value` (ไม่มี `default` เพราะบังคับให้ผู้ใช้ controller
  ต้องระบุมาเสมอ ไม่มีค่าที่ "สมเหตุสมผล" โดยทั่วไป)
- `static classes = ["over"]` — ชื่อ class ที่จะติดตอนเกินจำนวน กำหนดจาก HTML ตามหลักที่
  เรียนใน Step 527 (ไม่ hardcode ชื่อ class ไว้ใน JS)
- `connect()` เรียก `updateCount()` ทันทีตอนโหลดหน้า เพื่อให้ตัวเลขเริ่มต้นถูกต้องแม้ผู้ใช้จะ
  ยังไม่พิมพ์อะไรเลย (สำคัญมากตอน**แก้ไข**โพสต์เก่าที่มีเนื้อหาอยู่แล้วในช่อง — ถ้าไม่เรียก
  `connect()` ตัวเลขจะว่างเปล่าจนกว่าผู้ใช้จะพิมพ์ตัวอักษรแรก)
- `updateCount()` คำนวณ `remaining` แล้วอัปเดตทั้งข้อความและ class ในจุดเดียว เพื่อไม่ให้โค้ด
  สองส่วนไม่ sync กัน (ถ้าแยก method คำนวณกับ method แสดงผลออกจากกันมีความเสี่ยงที่จะเรียกไม่
  ครบคู่)

#### ขั้นที่ 3: ผูก HTML เข้ากับฟอร์มโพสต์จริง

```erb
<%# app/views/posts/_form.html.erb (ส่วนของช่อง body) %>
<%= form_with model: @post do |f| %>
  <div class="field" data-controller="character-counter"
       data-character-counter-limit-value="280"
       data-character-counter-over-class="char-counter--over">
    <%= f.label :body, "เนื้อหา" %>
    <%= f.text_area :body,
          data: {
            character_counter_target: "input",
            action: "input->character-counter#updateCount"
          } %>
    <p class="char-counter">
      เหลือ <span data-character-counter-target="count" class="char-counter__number"></span>
      ตัวอักษร
    </p>
  </div>

  <%= f.submit %>
<% end %>
```

#### ขั้นที่ 4: เพิ่ม CSS สำหรับสถานะ "เกิน limit"

```css
/* app/assets/stylesheets/character_counter.css */
.char-counter__number {
  font-weight: 600;
}

.char-counter--over .char-counter__number {
  color: #dc3545;
}
```

สังเกตว่าใช้ `.char-counter--over .char-counter__number` (descendant selector) เพราะ class
`char-counter--over` ถูกเติมที่ตัว `<span>` เอง (`this.countTarget.classList.toggle(...)`) จึง
เขียนแบบเจาะจงลงตัวเองตรงๆ ก็ได้เช่นกัน (`.char-counter--over { color: #dc3545; }`) — ทั้งสอง
แบบให้ผลเหมือนกันในกรณีนี้ เพราะ `this.countTarget` ชี้ไปที่ `<span>` โดยตรง ไม่ใช่ element แม่

#### ยืนยันผลลัพธ์ (อ้างอิงจากการทดสอบโค้ดชุดเดียวกันจริงในแอปทดลอง)

โค้ด controller ชุดนี้คือโค้ดชุดเดียวกับที่ทดสอบจริงแล้วใน Step ก่อนหน้าทุกประการ (สร้างด้วย
`bin/rails generate stimulus`, pin ผ่าน `pin_all_from` อัตโนมัติ ไม่ต้องแก้ `importmap.rb`)
ทดสอบผ่าน headless Chromium ด้วยค่า `limit-value="20"`:

| การกระทำ | ผลลัพธ์ที่ยืนยันจริง |
|---|---|
| โหลดหน้า (ยังไม่พิมพ์อะไร) | ตัวเลขแสดง `20` (เท่ากับ limit เต็ม) |
| พิมพ์ `"Hello Stimulus"` (14 ตัวอักษร) | ตัวเลขเปลี่ยนเป็น `6` แบบ real-time ทันทีที่พิมพ์แต่ละตัว |
| พิมพ์ข้อความยาว 53 ตัวอักษร (เกิน 20) | ตัวเลขแสดง `-33` และ class `over-limit` ถูกเติมเข้า `<span>` ทันที |

ครบตามโจทย์ทั้ง 3 ข้อ: อัปเดต real-time ไม่ต้อง submit, เปลี่ยนสีเมื่อเกิน limit, และค่า limit
กำหนดจาก HTML attribute ไม่ hardcode ไว้ใน JS

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **Confirm-delete controller** — สร้าง controller ชื่อ `confirm` ผูกกับปุ่มลบ (ใช้
   `button_to` หรือ `link_to ... data: { turbo_method: :delete }`) ให้เด้ง
   `window.confirm()` ก่อนเสมอ ถ้าผู้ใช้กด "ยกเลิก" ต้องไม่ยิง request ลบออกไปเลย (ใบ้: ใช้
   `data-action="click->confirm#check"` กับ `event.preventDefault()`, ให้ข้อความยืนยัน
   กำหนดได้จาก `data-confirm-message-value` เพื่อใช้ซ้ำกับหลายปุ่มที่มีข้อความต่างกันได้)
2. **Auto-save draft indicator** — ผูกกับช่อง body เดียวกับแบบฝึกหัดหลัก แต่เพิ่ม debounce
   (รอผู้ใช้หยุดพิมพ์ประมาณ 1 วินาทีก่อนค่อยทำงาน ด้วย `setTimeout`/`clearTimeout` ใน
   `connect()`/`disconnect()`) แล้วยิง `fetch` แบบ `PATCH` ไปบันทึก draft ที่ server พร้อม
   แสดงสถานะ "กำลังบันทึก..." → "บันทึกแล้วเมื่อ HH:MM" ด้วย value ที่เก็บสถานะปัจจุบันไว้
   (ใบ้: ต้องแนบ `X-CSRF-Token` header จาก `<meta name="csrf-token">` เพราะ `fetch` ไม่ผ่าน
   `form` เหมือน Turbo)
3. **Tab switcher ด้วย value + class** — สร้าง controller ชื่อ `tabs` ที่มีปุ่มแท็บหลายปุ่ม
   และเนื้อหาแต่ละแท็บ โดยเก็บ "แท็บที่กำลังเลือกอยู่" เป็น `static values = { active: Number }`
   ใช้ `activeValueChanged()` เพื่อสลับ `hidden` ของเนื้อหาแต่ละแท็บ และสลับ class ที่บ่งบอกว่า
   ปุ่มไหน "กำลังถูกเลือกอยู่" ผ่าน `static classes = ["selected"]` (ใบ้: ใช้
   `this.tabTargets` แบบ array คู่กับ `this.contentTargets` แล้ววนด้วย `forEach` พร้อม index
   เทียบกับ `activeValue`)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปรัชญา **"a modest JavaScript framework"** ของ Stimulus — เสริม (enhance) HTML ที่
  server render มาแล้ว แทนที่จะแทนที่ด้วย virtual DOM framework แบบ SPA และทำไมมันจับคู่กับ
  Turbo ได้อย่างเป็นธรรมชาติ
- สร้าง controller ใหม่ด้วย `bin/rails generate stimulus <name>` และเข้าใจว่าทำไมสแตก
  importmap-rails ไม่ควรใช้ `bin/rails stimulus:manifest:update` (ยืนยันจากการทดสอบจริงว่า
  ทำให้แอปพัง เพราะ relative import ไม่มีนามสกุลไฟล์ชนกับกลไก fingerprint ของ Propshaft)
- เข้าใจ **lifecycle** ของ controller (`connect()`/`disconnect()`) และผูก element เข้ากับ
  controller ผ่าน `data-controller`
- ผูก DOM event เข้ากับ method ของ controller ผ่าน **action**
  (`data-action="click->toggle#flip"`) รวมถึง event shorthand และการผูกหลาย action พร้อมกัน
- อ้างอิง element ลูกแบบไม่ต้อง `querySelector` เองผ่าน **target**
  (`data-toggle-target="content"`, `this.contentTarget`)
- เก็บ state แบบมี type ชัดเจนที่ซิงค์กับ DOM attribute แบบ reactive ผ่าน **value**
  (`static values`, `xxxValueChanged()`)
- สลับ CSS class โดยไม่ hardcode ชื่อ class ไว้ใน JavaScript ผ่าน **classes**
  (`static classes`, `this.xxxClass`)
- สร้างเมนู dropdown ที่ใช้งานได้จริงทีละขั้น พร้อมจัดการ event listener ที่ผูกกับ `document`
  เองอย่างถูกต้อง (bind + cleanup ใน `disconnect()`)
- ผูกหลาย controller บน element เดียวกัน/ที่ซ้อนกันได้ และเข้าใจว่า `this.element` ผูกกับ
  controller ของตัวเองเสมอ ไม่ใช่ element ที่เขียน `data-action` ไว้
- รู้จัก **outlet** สำหรับสื่อสารข้าม controller ที่ไม่ได้เป็น ancestor/descendant กัน และรู้ว่า
  ควรใช้เมื่อไหร่เทียบกับการยิง custom event

ทุกตัวอย่างในบทเรียนนี้ทดสอบจริงด้วยแอป Rails 8.1.4 ที่ generate ขึ้นมาใหม่ (มี stimulus-rails
1.3.4 ติดมาด้วย) และคลิก/พิมพ์ทดสอบจริงผ่าน headless Chromium (Playwright) ก่อนเขียนเนื้อหา
ทุกตัวอย่าง — ไม่ใช่โค้ดที่เขียนจากความจำเฉยๆ

**ต่อไป (Part 054):** ตอนนี้เรามี Turbo (อัปเดต HTML แบบไม่ reload) และ Stimulus (พฤติกรรม
โต้ตอบฝั่ง browser) ครบแล้ว ยังขาดแค่เรื่อง**หน้าตา** — Part 054 จะพาไปติดตั้งและใช้งาน
**Tailwind CSS** และ **Bootstrap** ใน Rails พร้อมออกแบบ responsive layout ที่ใช้งานได้จริงบน
ทุกขนาดหน้าจอ ซึ่งเราจะเอา controller อย่าง `toggle` ที่สร้างไว้ใน Part นี้ไปจับคู่กับ utility
class ของ Tailwind ทำเมนู responsive แบบมืออาชีพต่อยอดกันทันที
