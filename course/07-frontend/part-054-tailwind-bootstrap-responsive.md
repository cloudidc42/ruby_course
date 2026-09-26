# Part 054: Tailwind CSS / Bootstrap ใน Rails, Responsive Layout

> **Step ครอบคลุมใน Part นี้:** Step 531–540
> **ระดับ:** กลาง (ต่อจาก Part 053 เรื่อง Stimulus.js — controller, action, target, value)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x, Tailwind CSS 4.x (ผ่าน gem `tailwindcss-rails`
> 4.6.0 ซึ่งใช้ `tailwindcss-ruby` 4.3.3 เป็น standalone binary), Bootstrap 5.3.x (ผ่าน gem
> `cssbundling-rails`) ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วย `rails new` บน Rails 8.1.4 ทั้งฝั่ง
> `--css=tailwind` และ `--css=bootstrap` — รัน `bin/rails tailwindcss:build` จริง ตรวจสอบ CSS
> ที่ compile ออกมาจริง และ `curl` หน้าเว็บที่ render จริงเพื่อยืนยันทุกจุด

ใน Part 029 เราเรียนเรื่อง **Propshaft** ไปแล้ว — กลไก fingerprint และ serve ไฟล์ static ที่เป็น
พื้นฐานของทุกอย่างที่เกี่ยวกับ CSS/JS ใน Rails 8 และใน Part 051–053 เราใช้ **Turbo/Stimulus**
เพิ่ม interactivity ให้หน้าเว็บโดยแทบไม่ต้องเขียน JavaScript เยอะ แต่สิ่งที่ยังขาดไปตลอดคือ
**หน้าตา (visual design)** — ทุกตัวอย่างที่ผ่านมาใช้ HTML เปล่าๆ หรือ CSS ไม่กี่บรรทัดที่เขียนเอง

Part นี้จะพาไปสร้างหน้าเว็บที่ **"ดูเป็นเว็บจริง"** ด้วยสอง framework ที่นิยมที่สุดในโลก Rails
ปัจจุบัน: **Tailwind CSS** (framework แบบ utility-first ที่ Rails 8 รองรับเป็น first-class ผ่าน
`rails new --css=tailwind`) และ **Bootstrap** (framework แบบ component-based ที่เข้าถึงได้สอง
ทาง คือผ่าน `cssbundling-rails` หรือแค่แปะ CDN) พร้อมสอนสร้าง **responsive layout** จริง — nav
bar ที่ยุบเป็นเมนูมือถือ และ card grid ที่ปรับจำนวนคอลัมน์ตามขนาดจอ — ด้วยแนวคิด mobile-first
ที่ทั้งสอง framework ใช้ร่วมกัน

## สารบัญของ Part นี้

- Step 531: Utility-First CSS คืออะไร — ทำไม Tailwind ถึงเปลี่ยนวิธีคิดการเขียน CSS ทั้งวงการ
- Step 532: `rails new myapp --css=tailwind` — ติดตั้งจริง ไม่ต้องมี Node.js
- Step 533: โครงสร้างไฟล์ที่ได้, `bin/dev`, `Procfile.dev`, และ build/watch process ของ Tailwind
- Step 534: สร้าง Responsive Nav Bar ที่ยุบเป็นเมนูมือถือด้วย `sm:`/`md:`/`lg:`
- Step 535: สร้าง Responsive Card ด้วยแนวคิด Mobile-First
- Step 536: แก้ปัญหา "Class Soup" — แตกเป็น View Partial และ Helper (ไม่ใช้ `@apply`)
- Step 537: Dark Mode ด้วย `dark:` Prefix และปุ่ม Toggle ผ่าน Stimulus
- Step 538: เพิ่ม Bootstrap ด้วย `cssbundling-rails` — ทางเลือกที่ต้องมี Node.js จริง
- Step 539: Bootstrap แบบเร็วที่สุด — CDN `<link>`/`<script>` สำหรับ Prototype
- Step 540: กรอบการตัดสินใจ — เลือก Tailwind, Bootstrap, หรือ Custom SCSS Pipeline เมื่อไหร่
- แบบฝึกหัด: Responsive Post Card Grid ด้วย Tailwind แบบ Mobile-First ฉบับสมบูรณ์

---

## Step 531: Utility-First CSS คืออะไร — ทำไม Tailwind ถึงเปลี่ยนวิธีคิดการเขียน CSS ทั้งวงการ

### แนวทางดั้งเดิม: Semantic CSS

การเขียน CSS แบบดั้งเดิม (และแบบที่ทำใน Part 029 ตอนสร้าง `site-header.css`) คือตั้งชื่อ class
ที่สื่อความหมาย (semantic) แล้วเขียน property ของ class นั้นแยกไว้ในไฟล์ `.css` ต่างหาก:

```html
<header class="site-header">
  <img class="site-header__logo" src="logo.png">
</header>
```

```css
.site-header {
  display: flex;
  align-items: center;
  padding: 16px 24px;
  background-color: #1a1a2e;
}
```

วิธีนี้มีปัญหาที่วงการ CSS เจอซ้ำๆ มานานหลายปีในโปรเจกต์ขนาดใหญ่:

1. **ต้องคิดชื่อ class ตลอดเวลา** ("naming things is one of the two hard problems in computer
   science") — `.site-header` ดีไหม หรือควรเป็น `.page-header`, `.top-nav`, `.masthead`?
   ยิ่งโปรเจกต์โตยิ่งมีชื่อชนกันหรือชื่อไม่สื่อความหมายสะสมเยอะขึ้นเรื่อยๆ
2. **ต้อง context-switch ระหว่างไฟล์ตลอด** — แก้ HTML แล้วต้องเปิดไฟล์ CSS อีกไฟล์เพื่อดู/แก้
   สไตล์ที่ผูกกับ class นั้น สมองต้องสลับบริบทไปมาเสมอ
3. **CSS โตขึ้นเรื่อยๆ แบบไม่มีที่สิ้นสุด (CSS bloat)** — เพราะแทบไม่มีใครกล้าลบ CSS เก่าออก
   (ไม่รู้ว่ามี element ไหนใช้ class นั้นอยู่บ้างในระบบใหญ่) ทำให้ไฟล์ CSS ของโปรเจกต์เก่ามักมี
   ขนาดหลายพันบรรทัดที่ไม่มีใครกล้าแตะ

### แนวทางใหม่: Utility-First CSS

**Tailwind CSS** เสนอแนวทางตรงข้ามสุดขั้ว: แทนที่จะตั้งชื่อ class เอง ให้ประกอบ (compose) UI
จาก class สำเร็จรูปเล็กๆ จำนวนมากที่แต่ละตัวทำหน้าที่เดียว (utility class) ตรงๆ ใน HTML เลย:

```erb
<header class="flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2 text-white">
  <img src="logo.png" class="h-10">
</header>
```

ไม่มีไฟล์ CSS แยกให้เปิดดูเลย — `flex` คือ `display: flex`, `items-center` คือ
`align-items: center`, `bg-blue-600` คือ `background-color` จากสี blue ระดับความเข้ม 600 ในสเกล
สีของ Tailwind, `px-4` คือ `padding-left/right: 1rem` (ระดับ 4 ในสเกล spacing), `rounded-lg` คือ
`border-radius` ขนาดใหญ่ ทุกอย่างอ่านออกจากชื่อ class ได้ตรงๆ โดยไม่ต้องเปิดไฟล์อื่นเลย

### Tradeoff ที่ต้องเข้าใจก่อนตัดสินใจใช้

**ข้อดี:**

- **ไม่ต้องคิดชื่อ class เลย** — ปัญหา naming หายไปเกือบทั้งหมด เพราะ class มาจาก "คำศัพท์" ที่
  Tailwind กำหนดไว้แล้วตายตัว (scale ของ spacing, สี, ขนาด font ฯลฯ)
- **ไม่ต้อง context-switch** — เห็น style ของ element ตรงจุดที่ element นั้นอยู่ใน HTML ทันที
  ไม่ต้องเปิดไฟล์ที่สอง
- **CSS ไม่มีวันบวมขึ้นแบบไม่มีที่สิ้นสุด** — เพราะ Tailwind สแกนโค้ด (view/component) แล้ว
  generate เฉพาะ utility class ที่**ถูกใช้จริง**เท่านั้นลงในไฟล์ CSS สุดท้าย (จะพิสูจน์ให้เห็น
  จริงใน Step 533) ลบ element ออกจาก HTML แล้ว CSS ที่เกี่ยวข้องก็หายไปเอง โดยอัตโนมัติ ไม่มี
  "CSS ขยะ" ค้างให้ใครต้องมานั่งเดา
- **Design system มาพร้อมสเกลค่าที่กำหนดไว้แน่นอน (design token)** — spacing เป็นทวีคูณของ
  0.25rem เสมอ (1=0.25rem, 2=0.5rem, 4=1rem, ...) สีมี 11 ระดับความเข้ม (50–950) ต่อโทน ทำให้
  ทีมคุมความสม่ำเสมอ (consistency) ของหน้าตาเว็บได้ง่ายกว่าเขียน CSS เอง เพราะทุกคนเลือกจาก
  สเกลเดียวกัน ไม่มีใครพิมพ์ `padding: 13px` ที่ไม่ตรงกับใครในทีมอีกต่อไป

**ข้อเสีย:**

- **HTML ดูรกกว่าเดิมมาก** — `class="flex items-center gap-2 rounded-lg bg-blue-600 px-4 py-2
  text-white hover:bg-blue-700 focus:ring-2 focus:ring-blue-300"` ยาวกว่า `class="btn-primary"`
  มาก มือใหม่มักตกใจตอนเห็นครั้งแรก
- **เรียนรู้ชื่อ utility เยอะ** — ต้องจำ (หรือเปิด doc บ่อยๆ ช่วงแรก) ว่า `gap-2` ต่างจาก `gap-4`
  ยังไง, `text-sm` vs `text-base`, ฯลฯ
- **ถ้าไม่มีวินัยจะเกิด "class soup" ที่ซ้ำกันเยอะๆ ทั่วโปรเจกต์** — ถ้า combo เดียวกันถูกพิมพ์ซ้ำ
  ใน 20 ที่ แล้ววันหนึ่งต้องเปลี่ยนสีปุ่มทั้งหมด จะกลายเป็นแก้ 20 จุด — Step 536 จะสอนวิธีแก้
  ปัญหานี้แบบที่เหมาะกับ Rails โดยเฉพาะ

> **สรุปแนวคิดสำคัญ:** Tailwind ไม่ได้ "แทนที่ CSS" — มันคือชุด utility class ที่ compile มาจาก
> CSS จริงๆ (ยังคง cascade, specificity, ทุกอย่างของ CSS ปกติ) เพียงแต่เปลี่ยนหน่วยที่เราคิดจาก
> "component-level class" (`.card`, `.btn-primary`) มาเป็น "property-level class" (`.flex`,
> `.px-4`) — เมื่อ combo ของ property-level class ถูกใช้ซ้ำบ่อยๆ คำตอบใน Rails ไม่ใช่การเขียน
> custom CSS class กลับไปใหม่ (นั่นจะเสียข้อดีข้อ 3 ไป) แต่คือการแตกเป็น **View Partial** ซึ่ง
> Rails มีเครื่องมือรองรับอยู่แล้วในตัว (Step 536)

---

## Step 532: `rails new myapp --css=tailwind` — ติดตั้งจริง ไม่ต้องมี Node.js

### รันคำสั่งจริง

```bash
rails new myapp --css=tailwind
```

นี่คือคำสั่งเดียวที่ต้องใช้ — Rails จะรัน generator เสริมหลายตัวต่อกันให้อัตโนมัติ ผลลัพธ์จริง
จากการรันบน Rails 8.1.4 (ตัดส่วน `bundle install` ที่ยาวออก):

```
       rails  importmap:install
  ...
       rails  turbo:install stimulus:install
  ...
       rails  tailwindcss:install
       apply  .../tailwindcss-rails-4.6.0/lib/install/install_tailwindcss.rb
  Add Tailwindcss container element in application layout
      insert    app/views/layouts/application.html.erb
      insert    app/views/layouts/application.html.erb
  Build into app/assets/builds
      create    app/assets/builds
      create    app/assets/builds/.keep
  Add default app/assets/tailwind/application.css
      create    app/assets/tailwind/application.css
  Add default Procfile.dev
      create    Procfile.dev
  Ensure foreman is installed
         run    gem install foreman from "."
  Add bin/dev to start foreman
       force    bin/dev
  Compile initial Tailwind build
         run    rails tailwindcss:build from "."
≈ tailwindcss v4.3.3

Done in 137ms
```

สังเกตบรรทัดสุดท้าย: **`rails tailwindcss:build` รันสำเร็จและ compile CSS ได้จริงทันทีตอน
สร้างโปรเจกต์** — นี่คือจุดสำคัญที่สุดของ Step นี้: **ไม่มีการเรียก `npm install`, ไม่มีการสร้าง
`package.json`, และไม่มี `node_modules/` เกิดขึ้นเลยตลอดกระบวนการ**

### ยืนยันว่าไม่มี Node.js เข้ามาเกี่ยวข้องจริง

```bash
cd myapp
ls package.json 2>&1
# => ls: cannot access 'package.json': No such file or directory

ls node_modules 2>&1
# => ls: cannot access 'node_modules': No such file or directory
```

### แล้ว Tailwind ทำงานได้ยังไงถ้าไม่มี Node.js

คำตอบอยู่ที่ `Gemfile`:

```ruby
# Gemfile
gem "tailwindcss-rails"
```

`tailwindcss-rails` มี gem ที่มัน depend อยู่ชื่อ `tailwindcss-ruby` ซึ่งทำสิ่งที่ฉลาดมาก: มัน
**ห่อ (wrap) ตัว Tailwind CLI ที่ทีม Tailwind เองคอมไพล์เป็น native binary แบบ standalone ไว้**
(Tailwind CLI เขียนด้วย Rust/Go แล้ว distribute เป็นไฟล์ executable เดี่ยวๆ ต่อแพลตฟอร์ม ไม่ใช่
npm package ที่ต้องมี Node.js runtime มารันเลย) ตรวจสอบได้จริงว่า gem ติดตั้งอะไรมาให้:

```bash
gem list tailwindcss

# tailwindcss-rails (4.6.0)
# tailwindcss-ruby (4.3.3 x86_64-linux-gnu)   ← สังเกตชื่อ platform ต่อท้าย
```

ชื่อ gem มี `x86_64-linux-gnu` ต่อท้าย — นี่คือ **platform-specific gem** เวลา `bundle install`
RubyGems จะเลือกดาวน์โหลด gem เวอร์ชันที่ตรงกับ OS/CPU ของเครื่องเราเองอัตโนมัติ (มี variant
สำหรับ macOS arm64, macOS x86_64, Linux x86_64, Linux arm64, Windows ฯลฯ) ข้างในมีไฟล์ binary
จริงฝังอยู่:

```bash
find "$(bundle show tailwindcss-ruby)" -path "*/exe/*"
```

```
.../tailwindcss-ruby-4.3.3-x86_64-linux-gnu/exe/tailwindcss
.../tailwindcss-ruby-4.3.3-x86_64-linux-gnu/exe/x86_64-linux-gnu/tailwindcss
```

`tailwindcss-rails` แค่เรียก executable ตัวนี้ผ่าน Rake task (`rails tailwindcss:build`,
`rails tailwindcss:watch`) — ไม่มีขั้นตอนไหนต้องพึ่ง Node.js/npm/yarn เลยสักจุดเดียว นี่คือ
เหตุผลที่ Rails 8 กล้าใส่ `--css=tailwind` เป็นตัวเลือกมาตรฐานคู่กับ importmap (Part 029): ทั้งคู่
ยึดปรัชญาเดียวกันคือ **"ได้ของทันสมัยโดยไม่ต้องแบก Node.js toolchain เพิ่ม"**

> **เทียบกับ Rails รุ่นก่อน:** ก่อนมี `tailwindcss-rails` แบบปัจจุบัน การใช้ Tailwind ใน Rails
> ต้องพึ่ง `cssbundling-rails` + Node.js เสมอ (วิธีเดียวกับที่ Step 538 จะสอนสำหรับ Bootstrap)
> `tailwindcss-rails` คือ gem ที่ทำให้ Tailwind หลุดจากข้อจำกัดนั้นได้เพราะ Tailwind CLI เอง
> เปลี่ยนมาแจกเป็น standalone binary ตั้งแต่ Tailwind v3.0 เป็นต้นมา

---

## Step 533: โครงสร้างไฟล์ที่ได้, `bin/dev`, `Procfile.dev`, และ Build/Watch Process ของ Tailwind

### ไฟล์ที่ generator สร้างให้ (ของจริง)

```bash
find app/assets/tailwind app/assets/builds -type f
```

```
app/assets/tailwind/application.css
app/assets/builds/.keep
```

```css
/* app/assets/tailwind/application.css — ไฟล์ entry point ที่เราแก้/เพิ่มเติมเอง */
@import "tailwindcss";
```

ไฟล์นี้สั้นมากเพราะ Tailwind v4 เปลี่ยนวิธี config จากไฟล์ JavaScript (`tailwind.config.js` ใน
Tailwind v3) มาเป็น **CSS-first config** — ไม่มี `tailwind.config.js` เกิดขึ้นเลยในโปรเจกต์ที่
generate ใหม่ (ยืนยันได้ด้วย `find . -iname "*tailwind.config*"` แล้วไม่เจอไฟล์ใดๆ) การ
customize theme (เพิ่มสี, breakpoint ของตัวเอง) ทำผ่าน `@theme` block ในไฟล์ CSS นี้โดยตรง ซึ่ง
เป็นเรื่องขั้นสูงเกินขอบเขต Part นี้ — ที่ต้องรู้ตอนนี้คือค่า default (สี, spacing, breakpoint
มาตรฐาน) ใช้งานได้ทันทีโดยไม่ต้อง config อะไรเพิ่มเลย

### `app/assets/builds/` คือไฟล์ CSS ที่ compile แล้วจริง

```bash
bin/rails tailwindcss:build
ls -la app/assets/builds/
```

```
app/assets/builds/.keep
app/assets/builds/tailwind.css
```

`tailwind.css` คือไฟล์ที่ Tailwind CLI generate ออกมาจากการ**สแกนโค้ดทั้งโปรเจกต์** (view, helper,
partial ทุกไฟล์ `.erb`/`.rb`) หา utility class ที่ถูกใช้จริง แล้วเขียนเฉพาะ CSS rule ของ class
เหล่านั้นออกมา — โฟลเดอร์ `app/assets/builds/` **อยู่ใน load path ของ Propshaft** (ตามที่เรียน
ใน Part 029 Step 283) ยืนยันได้:

```ruby
Rails.application.config.assets.paths.each { |p| puts p }
# .../app/assets/builds     ← อยู่ตัวแรกสุด
# .../app/assets/images
# .../app/assets/stylesheets
# ...
```

นี่คือเหตุผลที่ layout ค่า default ที่ generate มาให้ **ไม่ต้องแก้อะไรเพิ่มเลย** ก็โหลด Tailwind
CSS ได้ทันที เพราะ `stylesheet_link_tag :app` (Part 029 Step 285) จะกวาดไฟล์ CSS ทุกไฟล์ใต้
`app/assets/**/*.css` มาให้อัตโนมัติ ซึ่งรวม `app/assets/builds/tailwind.css` ด้วย:

```erb
<%# app/views/layouts/application.html.erb — บรรทัดเดิมที่ generate มา ไม่ต้องแก้ %>
<%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
```

รัน server แล้ว `curl` ดู HTML จริง ยืนยันว่า Propshaft fingerprint ไฟล์นี้เหมือนไฟล์ CSS ปกติ
ทุกประการ:

```html
<link rel="stylesheet" href="/assets/tailwind-7233346f.css" data-turbo-track="reload" />
```

```bash
curl -I http://localhost:3000/assets/tailwind-7233346f.css
```

```
HTTP/1.1 200 OK
content-type: text/css; charset=utf-8
etag: "7233346f"
cache-control: public, max-age=31536000, immutable
content-length: 15500
```

Header เหมือนกับที่เรียนใน Part 029 Step 284 เป๊ะ — Propshaft ไม่สนใจว่าไฟล์ CSS จะมาจาก Sass
ธรรมดา หรือมาจากการ compile ของ Tailwind CLI มันแค่ resolve + fingerprint + serve เหมือนกันหมด
(หน้าที่ compile Tailwind เป็นของ `tailwindcss-rails` ล้วนๆ ไม่เกี่ยวกับ Propshaft เลย —
สอดคล้องกับปรัชญา "แบ่งความรับผิดชอบ" ที่อธิบายไว้ใน Part 029 Step 282)

### พิสูจน์ว่า Tailwind generate เฉพาะ class ที่ใช้จริง

ลองเขียน HTML ที่มี class `sm:grid-cols-2` ในไฟล์ view สักไฟล์ แล้ว build ใหม่:

```bash
grep -o "grid-cols-2" -A0 app/assets/builds/tailwind.css | head -1
```

```css
@media (min-width:40rem){.sm\:grid-cols-2{grid-template-columns:repeat(2,minmax(0,1fr))}}
```

สังเกต 2 อย่าง: (1) Tailwind escape เครื่องหมาย `:` ในชื่อ class ด้วย `\:` เพราะ `:` มีความหมาย
พิเศษใน CSS selector syntax ปกติ (pseudo-class) เวลาต้องใช้เป็นส่วนหนึ่งของชื่อ class จริงๆ ต้อง
escape (2) prefix `sm:` แปลว่า **"ใช้ style นี้ตั้งแต่ breakpoint `sm` (40rem = 640px) ขึ้นไป"**
ซึ่งถูกครอบด้วย `@media (min-width:40rem)` ให้อัตโนมัติ — ถ้าลบ `sm:grid-cols-2` ออกจากทุกไฟล์
view แล้ว build ใหม่ CSS rule นี้จะหายไปจาก `tailwind.css` ทันที (พิสูจน์ข้อดีข้อ 3 จาก Step 531
ได้ตรงๆ)

### `bin/dev` และ `Procfile.dev` — รัน server + watcher พร้อมกัน

```
# Procfile.dev — สร้างมาให้อัตโนมัติ
web: bin/rails server
css: bin/rails tailwindcss:watch
```

`bin/dev` (ก็สร้างมาให้อัตโนมัติเช่นกัน) เป็น shell script ที่เรียก **foreman** gem เพื่อรัน
process ทั้งสองบรรทัดใน `Procfile.dev` พร้อมกันในเทอร์มินัลเดียว:

```bash
#!/usr/bin/env sh

if ! gem list foreman -i --silent; then
  echo "Installing foreman..."
  gem install foreman
fi

export PORT="${PORT:-3000}"
export RUBY_DEBUG_OPEN="true"
export RUBY_DEBUG_LAZY="true"

exec foreman start -f Procfile.dev "$@"
```

ใช้งานจริงตอน develop:

```bash
bin/dev
```

```
07:12:01 web.1  | => Booting Puma
07:12:01 web.1  | => Rails 8.1.4 application starting in development
07:12:01 css.1  | ≈ tailwindcss v4.3.3
07:12:01 css.1  |
07:12:01 css.1  | Done in 89ms
07:12:01 css.1  |
07:12:01 css.1  | Watching for changes...
```

`web` รัน Rails server ตามปกติ ส่วน `css` รัน `tailwindcss:watch` ที่เฝ้าดูไฟล์ view/CSS ทุกไฟล์
ตลอดเวลา — พอแก้ class ใน `.erb` แล้ว save, Tailwind CLI จะ **compile ใหม่ให้ภายในเสี้ยววินาที
อัตโนมัติ** โดยเราไม่ต้องสั่งอะไรเอง แค่ refresh browser (หรือถ้าเปิด Turbo อยู่ก็อาจไม่ต้อง
refresh เองด้วยซ้ำถ้าใช้ผสมกับ `broadcast_refreshes` แบบที่เรียนใน Part 051 เรื่อง Turbo Drive)

> **ข้อควรรู้:** ตั้งแต่ตอนนี้ในหลักสูตร ทุกครั้งที่สั่ง `rails server` เฉยๆ ระหว่าง develop กับ
> Tailwind แล้วแก้ CSS ไม่เห็นผล **ให้เช็คก่อนว่าลืมรัน `bin/dev` แทน `bin/rails server` ตรงๆ**
> หรือไม่ — ถ้ารัน `bin/rails server` เพียวๆ จะไม่มี watcher มา compile CSS ให้อัตโนมัติเลย

---

## Step 534: สร้าง Responsive Nav Bar ที่ยุบเป็นเมนูมือถือด้วย `sm:`/`md:`/`lg:`

### Breakpoint ของ Tailwind (ค่า default)

| Prefix | min-width | อุปกรณ์โดยประมาณ |
|---|---|---|
| (ไม่มี prefix) | 0px | มือถือ (mobile-first: นี่คือค่าตั้งต้นเสมอ) |
| `sm:` | 640px | มือถือแนวนอน / แท็บเล็ตขนาดเล็ก |
| `md:` | 768px | แท็บเล็ต |
| `lg:` | 1024px | จอคอมพิวเตอร์ขนาดเล็ก/แล็ปท็อป |
| `xl:` | 1280px | จอคอมพิวเตอร์ทั่วไป |
| `2xl:` | 1536px | จอกว้างพิเศษ |

หลักการสำคัญที่ Tailwind (และ CSS responsive design สมัยใหม่ทั้งวงการ) ใช้คือ **mobile-first**:
class ที่ไม่มี prefix คือ style ของ**จอเล็กสุดก่อนเสมอ** ส่วน prefix อย่าง `md:` แปลว่า
**"ตั้งแต่ขนาดนี้ขึ้นไป ให้ override ด้วย style นี้"** (ใช้ `min-width` ไม่ใช่ `max-width`) ตรงข้าม
กับแนวคิด desktop-first แบบเก่าที่เขียน style จอใหญ่ก่อนแล้วค่อยไล่ override ลงจอเล็ก

### สร้าง Nav Bar ที่ยุบเป็นเมนูมือถือจริง

โจทย์: nav bar ที่แสดงเมนูเป็นแถวแนวนอนบนจอ `md` ขึ้นไป แต่ยุบเหลือปุ่ม hamburger บนจอมือถือ
ทำงานร่วมกับ Stimulus (เรียนมาแล้วเต็มๆ ใน Part 053):

```bash
bin/rails generate stimulus nav
```

```javascript
// app/javascript/controllers/nav_controller.js
import { Controller } from "@hotwired/stimulus"

// Connects to data-controller="nav"
// สลับเปิด/ปิดเมนูมือถือ (hamburger menu) เมื่อจอแคบกว่า breakpoint md
export default class extends Controller {
  static targets = ["menu"]

  toggle() {
    this.menuTarget.classList.toggle("hidden")
  }
}
```

```erb
<%# app/views/shared/_navbar.html.erb %>
<nav class="bg-white border-b border-slate-200" data-controller="nav">
  <div class="mx-auto max-w-5xl px-4">
    <div class="flex h-16 items-center justify-between">
      <%= link_to "MyApp", root_path, class: "text-lg font-bold text-slate-900" %>

      <%# เมนูปกติ: ซ่อนบนจอมือถือ (< md), แสดงเป็นแถวแนวนอนตั้งแต่ md ขึ้นไป %>
      <div class="hidden md:flex md:items-center md:gap-6">
        <%= link_to "หน้าแรก", root_path,
              class: "text-sm font-medium text-slate-600 hover:text-slate-900" %>
        <%= link_to "บทความ", root_path,
              class: "text-sm font-medium text-slate-600 hover:text-slate-900" %>
      </div>

      <%# ปุ่ม hamburger: แสดงเฉพาะจอมือถือ (< md), ซ่อนตั้งแต่ md ขึ้นไป %>
      <button type="button" data-action="nav#toggle"
              class="md:hidden inline-flex items-center justify-center rounded-lg p-2
                     text-slate-600 hover:bg-slate-100">
        <span class="sr-only">เปิดเมนู</span>
        ☰
      </button>
    </div>

    <%# เมนูมือถือแบบ dropdown: ซ่อนไว้ก่อนด้วย .hidden, Stimulus สลับให้เมื่อกดปุ่ม hamburger %>
    <div data-nav-target="menu" class="hidden md:hidden space-y-1 pb-4">
      <%= link_to "หน้าแรก", root_path,
            class: "block rounded-lg px-3 py-2 text-sm font-medium text-slate-700 hover:bg-slate-100" %>
      <%= link_to "บทความ", root_path,
            class: "block rounded-lg px-3 py-2 text-sm font-medium text-slate-700 hover:bg-slate-100" %>
    </div>
  </div>
</nav>
```

**อธิบายกลไก responsive ที่เกิดขึ้น:**

- `hidden md:flex` — ค่าตั้งต้น (มือถือ) คือ `display: none` (`hidden`) จอกว้างตั้งแต่ `md`
  (768px) ขึ้นไป override เป็น `display: flex` (`md:flex`) — เมนูแนวนอนจะ**โผล่ขึ้นมาเอง**
  โดยไม่ต้องมี JavaScript เข้ามาเกี่ยวข้องเลยตอนข้าม breakpoint
- `md:hidden` บนปุ่ม hamburger — ตรงข้ามกัน คือแสดงเป็นค่าตั้งต้น (มือถือ) แล้วซ่อนตั้งแต่ `md`
  ขึ้นไป — ผลคือปุ่ม hamburger กับเมนูแนวนอน**ไม่มีวันแสดงพร้อมกัน**
- ทั้งสอง class คู่นี้ทำงานล้วนๆ ด้วย CSS media query (ดูได้จาก `tailwind.css` ที่ compile ออกมา
  จะเห็น `.md\:hidden` ถูกครอบด้วย `@media (min-width:48rem)`) — **JavaScript (Stimulus) เข้ามา
  ทำหน้าที่แค่เปิด/ปิดเมนู dropdown บนมือถือเท่านั้น ไม่เกี่ยวกับการสลับ layout ระหว่าง breakpoint
  เลย** นี่คือแนวทางที่ถูกต้อง: **ให้ CSS จัดการเรื่อง layout/breakpoint เสมอที่เป็นไปได้**
  เก็บ JavaScript ไว้ทำแค่ state ที่ CSS ทำเองไม่ได้ (เช่น toggle เปิด/ปิด)

ทดสอบจริงด้วยการ resize browser หรือเปิด developer tools แล้วสลับ device toolbar — จะเห็นเมนู
แนวนอนกับปุ่ม hamburger สลับกันแสดง/ซ่อนที่ 768px พอดี โดยไม่ต้อง reload หน้าเว็บเลย

---

## Step 535: สร้าง Responsive Card ด้วยแนวคิด Mobile-First

### Grid ที่ปรับจำนวนคอลัมน์ตามขนาดจอ

โจทย์คลาสสิกที่สุดของ responsive design: การ์ดที่แสดง **1 คอลัมน์บนมือถือ, 2 คอลัมน์บนแท็บเล็ต,
3 คอลัมน์บนจอคอมพิวเตอร์**

```erb
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
  <%= render partial: "post_card", collection: @posts %>
</div>
```

อ่านจากซ้ายไปขวาตามหลัก mobile-first:

- `grid grid-cols-1` — ค่าตั้งต้น (ไม่มี prefix = ใช้กับทุกขนาดจอ ถ้าไม่ถูก override) คือ CSS
  Grid ที่มี **1 คอลัมน์** — เหมาะกับมือถือที่มีพื้นที่แนวนอนจำกัด
- `sm:grid-cols-2` — ตั้งแต่จอกว้าง 640px ขึ้นไป override เป็น **2 คอลัมน์**
- `lg:grid-cols-3` — ตั้งแต่จอกว้าง 1024px ขึ้นไป override เป็น **3 คอลัมน์**
- `gap-6` — ระยะห่างระหว่าง grid cell ทุกทิศทาง (`1.5rem` = ระดับ 6 ในสเกล spacing) ค่าเดียว
  ใช้ได้กับทุกขนาดจอ เพราะไม่มีเหตุผลต้องเปลี่ยนตาม breakpoint ในกรณีนี้

### การ์ดหนึ่งใบ

```erb
<%# app/views/pages/_post_card.html.erb %>
<article class="flex flex-col rounded-xl border border-slate-200 bg-white p-5 shadow-sm
                transition hover:shadow-md">
  <div class="mb-3 flex items-center justify-between">
    <span class="inline-block rounded-full bg-blue-100 px-2.5 py-0.5 text-xs font-semibold text-blue-800">
      <%= post_card.tag %>
    </span>
    <span class="text-xs text-slate-400"><%= post_card.published_on %></span>
  </div>

  <h2 class="mb-2 text-lg font-bold text-slate-900"><%= post_card.title %></h2>
  <p class="mb-4 flex-1 text-sm text-slate-600"><%= post_card.excerpt %></p>

  <div class="flex items-center gap-2 border-t border-slate-100 pt-3">
    <span class="flex h-8 w-8 items-center justify-center rounded-full bg-slate-200
                 text-xs font-semibold text-slate-700">
      <%= post_card.author.first %>
    </span>
    <span class="text-sm font-medium text-slate-700"><%= post_card.author %></span>
  </div>
</article>
```

**สังเกตแนวคิดของ utility class ที่ใช้บ่อยในการ์ด:**

- `flex flex-col` — จัด element ข้างในเป็นแนวตั้งแทนแนวนอน (default ของ `flex` คือแนวนอน)
- `flex-1` บน `<p>` — ขยายให้กิน space ที่เหลือทั้งหมด ทำให้การ์ดที่มี excerpt ยาว/สั้นต่างกัน
  ยังจัดแถวส่วน author ให้อยู่ตำแหน่งเดียวกันเป๊ะ (ชิดล่างของการ์ดเสมอ) — ปัญหาที่ CSS ปกติต้อง
  เขียน flexbox เองอยู่แล้ว ที่นี่แค่เติม class เดียว
- `transition hover:shadow-md` — เปลี่ยนเงา (shadow) แบบนุ่มนวลตอน hover เมาส์ผ่าน (`transition`
  บอกให้ animate การเปลี่ยนแปลง CSS property ทุกตัวที่เปลี่ยน, `hover:` prefix บอกว่า
  `shadow-md` ให้ใช้เฉพาะตอน `:hover` เท่านั้น)

ทดสอบด้วยการ render จริงแล้วเปิด developer tools ปรับความกว้างหน้าต่าง — จำนวนคอลัมน์จะเปลี่ยน
1 → 2 → 3 ที่จุดตัด 640px และ 1024px พอดี โดยการ์ดแต่ละใบสูงเท่ากันเสมอในแถวเดียวกัน (features
ของ CSS Grid ที่ทำให้ทุก cell ในแถวเดียวกันสูงเท่ากันเป็นค่า default อยู่แล้ว)

---

## Step 536: แก้ปัญหา "Class Soup" — แตกเป็น View Partial และ Helper (ไม่ใช้ `@apply`)

### ปัญหา: combo ของ utility class ซ้ำกันหลายจุด

สมมติ badge สีฟ้าที่ใช้ใน `_post_card.html.erb` (`class="inline-block rounded-full bg-blue-100
px-2.5 py-0.5 text-xs font-semibold text-blue-800"`) ต้องใช้ซ้ำใน 5 หน้าทั่วแอป — ถ้าพิมพ์ class
ยาวๆ นี้ซ้ำทุกที่ วันหนึ่งอยากเปลี่ยนสีจากฟ้าเป็นเขียว ต้องไล่แก้ทีละจุด เสี่ยงพิมพ์ผิดหรือ
แก้ไม่ครบ — นี่คือ "class soup" ที่พูดถึงใน Step 531

### ทางเลือกที่ "ดูเหมือน" จะแก้ได้แต่ไม่แนะนำ: `@apply`

เอกสาร Tailwind แนะนำ directive `@apply` ไว้สำหรับดึง utility class มารวมเป็น custom class ใน
CSS:

```css
/* วิธีนี้ทำได้ แต่ไม่แนะนำในโปรเจกต์ Rails */
.badge-blue {
  @apply inline-block rounded-full bg-blue-100 px-2.5 py-0.5 text-xs font-semibold text-blue-800;
}
```

**ทำไมถึงไม่แนะนำในบริบทของ Rails โดยเฉพาะ** (ตรงข้ามกับโปรเจกต์ frontend ล้วนๆ ที่อาจจะโอเคกว่า):

1. **เสียข้อดีข้อสำคัญที่สุดของ utility-first ไปคือ "ไม่ต้องคิดชื่อ"** — พอสร้าง `.badge-blue`
   ขึ้นมา ก็กลับไปเจอปัญหาเดิมของ semantic CSS ทันที (ต้องคิดชื่อ, ต้องจำว่าไฟล์ไหนมี class
   อะไรบ้าง)
2. **Rails มีเครื่องมือที่ทำงานแบบเดียวกันแต่เหมาะกับ server-rendered app มากกว่าอยู่แล้ว** —
   นั่นคือ **View Partial** และ **Helper** ซึ่งเราใช้มาตั้งแต่ Part 024 — ทั้งสองอย่างนี้ให้ผลลัพธ์
   เดียวกับ `@apply` (ใช้ combo ซ้ำได้จากที่เดียว) แต่**ไม่ต้องมี build step ที่สองซ้อนเข้ามาในระบบ
   CSS** และเป็นแนวทางเดียวกับที่ใช้ reuse HTML ส่วนอื่นๆ ทั้งแอปอยู่แล้ว ทีมไม่ต้องเรียนรู้กลไก
   ใหม่เพิ่ม
3. **`@apply` ทำให้ debug ยากขึ้น** — เปิด developer tools แล้วเห็น `.badge-blue` เฉยๆ ต้องไป
   เปิดไฟล์ CSS ต่อเพื่อดูว่าจริงๆ แล้วมัน apply utility อะไรบ้าง (กลับไปเจอปัญหา context-switch
   ที่ Tailwind ตั้งใจแก้ตั้งแต่แรกอีกครั้ง)

> **นี่คือฉันทามติที่ชุมชน Rails ส่วนใหญ่ยึดถือในปัจจุบัน:** ใช้ `@apply` ให้น้อยที่สุดเท่าที่จะ
> ทำได้ (หรือไม่ใช้เลย) แล้วแก้ปัญหา class ซ้ำด้วยเครื่องมือ view-layer ของ Rails เอง — partial
> สำหรับ component ที่มี markup ซ้ำ (เช่นการ์ด, badge ที่มี text ต่างกัน) และ helper method
> สำหรับ class string ที่ซ้ำแต่ markup รอบๆ อาจไม่เหมือนกันทุกจุด

### ทางเลือกที่ 1: View Partial (เหมาะกับ Markup ที่ซ้ำทั้งชิ้น)

```erb
<%# app/views/shared/_badge.html.erb %>
<span class="inline-block rounded-full bg-<%= color %>-100 px-2.5 py-0.5 text-xs font-semibold text-<%= color %>-800">
  <%= text %>
</span>
```

```erb
<%= render "shared/badge", text: post.tag, color: "blue" %>
<%= render "shared/badge", text: "หมด", color: "red" %>
```

> **ข้อควรระวัง:** อย่าประกอบชื่อ class จาก string ที่มาจาก user input หรือค่าที่ควบคุมไม่ได้
> ตรงๆ แบบ `"bg-#{color}-100"` ในโปรเจกต์จริง เพราะ Tailwind สแกนหา class จาก**ข้อความตรงๆ**ใน
> ไฟล์ ไม่ได้รัน Ruby จริงตอน build — ถ้า `color` มาจากตัวแปรที่ไม่ปรากฏเป็น string เต็มๆ ในโค้ด
> (เช่นมาจาก database) Tailwind จะ**มองไม่เห็น** class นั้นเลยและไม่ generate CSS ให้ (จะพัง
> ตอน production เงียบๆ) วิธีที่ปลอดภัยคือกำหนด mapping แบบเขียนตรงๆ ในโค้ด Ruby:
> `COLORS = { blue: "bg-blue-100 text-blue-800", red: "bg-red-100 text-red-800" }.freeze` แล้ว
> ใช้ `COLORS.fetch(color)` แทน เพื่อให้ class string เต็มๆ ปรากฏในไฟล์ที่ Tailwind สแกนเจอจริง

### ทางเลือกที่ 2: Helper Method (เหมาะกับ Class String ล้วนๆ)

```ruby
# app/helpers/pages_helper.rb
module PagesHelper
  # รวม utility class ที่ซ้ำๆ กันของ "badge สีตามหมวดหมู่" ไว้ที่เดียว แทนที่จะพิมพ์
  # class ยาวๆ ซ้ำในทุก view ที่ต้องโชว์ tag ของโพสต์ — เรียกใช้แค่ tag_badge(post.tag)
  def tag_badge(text)
    tag.span text,
      class: "inline-block rounded-full bg-blue-100 px-2.5 py-0.5 text-xs font-semibold text-blue-800"
  end
end
```

```erb
<%= tag_badge(post.tag) %>
```

`tag.span` คือ Tag Builder ของ Rails (`ActionView::Helpers::TagHelper`) ที่เรียนมาแล้วใน
Part 024 — ใช้แทนการเขียน HTML string เองเพื่อให้ Rails escape ค่าที่ไม่ปลอดภัยให้อัตโนมัติ
(ป้องกัน XSS) วิธีนี้เหมาะกับกรณีที่ combo ของ class เป็นก้อนเดียวชัดเจน ไม่มี markup ซับซ้อน
ล้อมรอบ

### กฎการเลือกใช้ในทางปฏิบัติ

| สถานการณ์ | ใช้อะไร |
|---|---|
| Markup ซ้ำทั้งชิ้น (การ์ด, badge ที่มี structure ซับซ้อน) | View Partial |
| Class string ล้วนๆ ที่ห่อ text/element ธรรมดา | Helper method |
| ต้องการ logic ซับซ้อน + state (เช่น ตรวจ current_page เพื่อ highlight nav link) | Helper method หรือ Presenter (Part 083) |
| Component ที่ต้องมี Ruby object เป็นของตัวเอง มี validation/logic เยอะ | ViewComponent (**Part 055** — Part ถัดไป) |

Partial และ helper แก้ปัญหา class soup ได้ดีในระดับหนึ่ง แต่เมื่อ component เริ่มมี logic ซับซ้อน
ขึ้น (เช่นต้องรับ slot หลายแบบ, ต้อง test แยกจาก view) Rails ยังมีเครื่องมือที่ทรงพลังกว่าคือ
**ViewComponent** ซึ่งเป็นหัวข้อทั้ง Part ถัดไป — Part นี้จึงตั้งใจสอนแค่สองเครื่องมือพื้นฐานที่
เพียงพอสำหรับ 80% ของสถานการณ์จริงก่อน

---

## Step 537: Dark Mode ด้วย `dark:` Prefix และปุ่ม Toggle ผ่าน Stimulus

### กลยุทธ์ Dark Mode สองแบบของ Tailwind

Tailwind รองรับ dark mode ผ่าน prefix `dark:` (เช่น `dark:bg-slate-900` = ใช้ background นี้เมื่อ
อยู่ใน dark mode) โดยมี 2 กลยุทธ์ตัดสินว่า "ตอนนี้คือ dark mode หรือไม่":

1. **`media` (ค่า default ของ Tailwind v4)** — อิงตาม CSS media feature
   `prefers-color-scheme: dark` ของระบบปฏิบัติการ/browser ล้วนๆ ผู้ใช้ต้องไปเปลี่ยนที่ตั้งค่า
   เครื่องเอง เว็บไม่มีปุ่ม toggle ให้กดเอง
2. **`class`** — อิงตามว่า element บรรพบุรุษ (ปกติคือ `<html>`) มี class `dark` ติดอยู่หรือไม่
   วิธีนี้ทำให้เว็บมี**ปุ่ม toggle ของตัวเอง**ได้ ไม่ต้องพึ่งการตั้งค่าของระบบปฏิบัติการ

โจทย์ Part นี้ต้องการปุ่ม toggle บนหน้าเว็บ จึงต้องสลับมาใช้กลยุทธ์ `class` — ทำได้ด้วยการเพิ่ม
`@custom-variant` ในไฟล์ CSS entry point (วิธีของ Tailwind v4 ที่ config ผ่าน CSS ล้วนๆ):

```css
/* app/assets/tailwind/application.css */
@import "tailwindcss";

/* เปลี่ยน dark mode จาก default (ตาม prefers-color-scheme ของ OS) มาเป็นสลับด้วย
   class="dark" ที่ <html> เอง เพื่อให้ปุ่ม toggle บนหน้าเว็บควบคุมได้จริง */
@custom-variant dark (&:where(.dark, .dark *));
```

### ใช้ `dark:` Prefix ในหน้าเว็บ

```erb
<nav class="bg-white dark:bg-slate-900 border-b border-slate-200 dark:border-slate-700">
  <a href="/" class="text-lg font-bold text-slate-900 dark:text-white">MyApp</a>
</nav>
```

อ่านง่ายมาก: `bg-white dark:bg-slate-900` แปลว่า "พื้นหลังสีขาวปกติ แต่ถ้าอยู่ใน dark mode ให้
เปลี่ยนเป็นสี slate เข้มแทน" — Tailwind generate CSS ให้ทั้งสองเวอร์ชันไว้พร้อมกันเสมอ browser
เป็นคนตัดสินใจว่าจะใช้ rule ไหนตาม class `dark` บน `<html>` ตอนนั้น

### ปุ่ม Toggle ด้วย Stimulus + `localStorage`

```javascript
// app/javascript/controllers/theme_controller.js
import { Controller } from "@hotwired/stimulus"

// Connects to data-controller="theme"
// สลับ dark/light mode โดยเติม/ถอด class "dark" ที่ <html> แล้วจำค่าไว้ใน localStorage
export default class extends Controller {
  connect() {
    const saved = this.readStoredTheme()
    if (saved === "dark") document.documentElement.classList.add("dark")
  }

  toggle() {
    document.documentElement.classList.toggle("dark")
    const isDark = document.documentElement.classList.contains("dark")
    this.storeTheme(isDark ? "dark" : "light")
  }

  readStoredTheme() {
    try {
      return localStorage.getItem("theme")
    } catch {
      return null
    }
  }

  storeTheme(value) {
    try {
      localStorage.setItem("theme", value)
    } catch {
      // localStorage ไม่พร้อมใช้งาน (private mode ฯลฯ) — ปล่อยผ่าน ไม่กระทบการทำงานหลัก
    }
  }
}
```

```erb
<button type="button" data-controller="theme" data-action="theme#toggle"
        class="rounded-lg border border-slate-300 px-3 py-1.5 text-sm font-medium
               text-slate-700 hover:bg-slate-100
               dark:border-slate-600 dark:text-slate-200 dark:hover:bg-slate-800">
  🌓 สลับธีม
</button>
```

**อธิบายกลไก:**

- `connect()` รันทันทีตอน controller ต่อกับ DOM (เรียนมาแล้วใน Part 053) — เช็คว่าผู้ใช้เคยเลือก
  ธีมไว้ก่อนหน้านี้หรือไม่จาก `localStorage` (browser storage ที่อยู่ได้ข้ามการปิด/เปิดเว็บ) ถ้า
  เคยเลือก "dark" ไว้ ให้เติม class ทันทีโดยไม่ต้องรอผู้ใช้กดปุ่มซ้ำ
- `toggle()` ทำสองอย่าง: สลับ class `dark` บน `<html>` (ซึ่งทำให้ **ทุก** `dark:` utility ทั่ว
  หน้าเว็บเปลี่ยนพร้อมกันทันที เพราะ CSS selector ที่ตั้งไว้คือ `&:where(.dark, .dark *)` แปลว่า
  "element ไหนก็ตามที่อยู่ใต้ (หรือเป็น) element ที่มี class `dark`") และบันทึกค่าไว้ใน
  `localStorage` เพื่อให้จำธีมได้แม้ปิดแท็บแล้วเปิดใหม่
- ห่อ `localStorage.getItem`/`setItem` ด้วย `try/catch` เพราะบาง browser mode (Safari private
  browsing เก่าบางเวอร์ชัน) อาจ throw exception ตอนเรียก ป้องกันไม่ให้ทั้งหน้าเว็บพังเพราะ
  ฟีเจอร์เสริมนี้เพียงอย่างเดียว

ทดสอบจริง: เปิดหน้าเว็บ กดปุ่ม "สลับธีม" พื้นหลัง/สีตัวอักษรทั้งหน้าเปลี่ยนทันทีโดยไม่มีการ reload
หน้าเว็บเลย ปิดแท็บแล้วเปิดใหม่ ธีมที่เลือกไว้ยังคงอยู่ (อ่านจาก `localStorage` ใน `connect()`)

---

## Step 538: เพิ่ม Bootstrap ด้วย `cssbundling-rails` — ทางเลือกที่ต้องมี Node.js จริง

### ติดตั้งจริงในโปรเจกต์ใหม่

```bash
rails new myapp --css=bootstrap
```

หรือถ้ามีโปรเจกต์อยู่แล้วอยากเพิ่ม Bootstrap เข้าไปทีหลัง:

```bash
bundle add cssbundling-rails
bin/rails css:install:bootstrap
```

ผลลัพธ์จริงตอนรัน (ตัดส่วน warning ของ Sass deprecation ที่ยาวมากออก — เป็น warning จาก Bootstrap
เอง ไม่ใช่จาก Rails):

```
       rails  css:install:bootstrap
  ...
      create  app/assets/stylesheets/application.bootstrap.scss
       force  Procfile.dev
         run  yarn add bootstrap @popperjs/core sass postcss postcss-cli autoprefixer bootstrap-icons
         run  yarn add nodemon --dev
        gsub  Procfile.dev
         run  bundle install --quiet
```

### จุดต่างสำคัญที่สุดจาก Tailwind: มี Node.js เข้ามาเต็มรูปแบบ

```bash
cat package.json
```

```json
{
  "name": "app",
  "private": "true",
  "dependencies": {
    "@popperjs/core": "^2.11.8",
    "autoprefixer": "^10.6.1",
    "bootstrap": "^5.3.8",
    "bootstrap-icons": "^1.13.1",
    "nodemon": "^3.1.14",
    "postcss": "^8.5.28",
    "postcss-cli": "^12.0.0",
    "sass": "^1.105.0"
  },
  "scripts": {
    "build:css:compile": "sass ./app/assets/stylesheets/application.bootstrap.scss:./app/assets/builds/application.css --no-source-map --load-path=node_modules",
    "build:css:prefix": "postcss ./app/assets/builds/application.css --use=autoprefixer --output=./app/assets/builds/application.css",
    "build:css": "yarn run build:css:compile && yarn run build:css:prefix",
    "watch:css": "nodemon --watch ./app/assets/stylesheets/ --ext scss --exec \"yarn run build:css\""
  }
}
```

ต่างจาก `--css=tailwind` แบบสุดขั้ว: มี **`package.json` จริง**, มี **`node_modules/`** เต็ม
โฟลเดอร์ (Bootstrap + Popper.js + Dart Sass compiler + PostCSS + Autoprefixer + dependency ของ
ทุกตัวอีกที รวมแล้วหลายร้อยแพ็กเกจ), และต้องมี **Node.js/npm หรือ yarn ติดตั้งในเครื่องจริง** ถึง
จะรันคำสั่ง build ได้ — นี่คือความต่างที่โจทย์ Part นี้เน้นให้เห็นชัดๆ: **Tailwind (ผ่าน
`tailwindcss-rails`) ไม่ต้องมี Node.js เลย ส่วน Bootstrap ผ่าน `cssbundling-rails` ต้องมี Node.js
เสมอ** เพราะ Bootstrap distribute ตัวเองเป็น Sass source ที่ต้อง compile ด้วย Dart Sass (เขียน
ด้วย Dart แต่ตัว npm package เรียกผ่าน Node.js) ไม่ได้มี standalone CLI binary เหมือน Tailwind

```
# Procfile.dev ของแอป Bootstrap
web: env RUBY_DEBUG_OPEN=true bin/rails server
css: yarn watch:css
```

ใช้ `bin/dev` เหมือนกันทุกประการ (foreman รัน 2 process พร้อมกัน) แค่ process `css` เรียก `yarn`
แทนที่จะเรียก `bin/rails tailwindcss:watch` ตรงๆ

### `application.bootstrap.scss` — จุดที่ import Bootstrap เข้ามาทั้งก้อน

```scss
/* app/assets/stylesheets/application.bootstrap.scss */
@import 'bootstrap/scss/bootstrap';
@import 'bootstrap-icons/font/bootstrap-icons';
```

บรรทัดเดียวนี้ดึง Bootstrap **ทั้งเฟรมเวิร์ก** เข้ามา (grid system, ทุก component, ทุก utility
class) คอมไพล์ผลลัพธ์จริงในเครื่องทดสอบได้ไฟล์ CSS ขนาด **~1.1 MB (unminified)** — เทียบกับ
Tailwind ที่ generate เฉพาะ class ที่ใช้จริงแล้วได้แค่ ~15 KB ในหน้าเว็บตัวอย่างเดียวกัน (Step
533) ส่วนต่างนี้คือ tradeoff โดยตรงของปรัชญาที่ต่างกัน: Bootstrap แจก "ชุดสำเร็จรูปทั้งชุด" ไว้
ก่อนแล้วค่อย opt-out (comment ทิ้งเฉพาะส่วนที่ไม่ใช้ใน Sass import ได้ถ้าต้องการลดขนาด) ส่วน
Tailwind opt-in เฉพาะสิ่งที่ใช้จริงตั้งแต่แรก

### JavaScript ของ Bootstrap ยังผ่าน importmap ได้ (ผสมกันได้)

ข้อสังเกตที่น่าสนใจ: `--css=bootstrap` ไม่ได้เปลี่ยนฝั่ง JavaScript ให้ไปใช้ Node bundler ด้วย —
`config/importmap.rb` ยังใช้ **importmap-rails** ตามปกติ (ค่า default ของ Rails 8) โดยเพิ่มบรรทัด
pin ให้เอง:

```ruby
# config/importmap.rb
pin "bootstrap", to: "bootstrap.bundle.min.js"
```

แต่ที่น่าสนใจกว่าคือไฟล์ `bootstrap.bundle.min.js` **ไม่ได้ถูกดาวน์โหลดไปเก็บใน
`vendor/javascript/`** เหมือนตอนใช้ `bin/importmap pin` กับ npm package ทั่วไป (Part 029
Step 286) แต่ generator เพิ่ม path เข้า Propshaft load path ให้ชี้ตรงไปยัง `node_modules` เลย:

```ruby
# config/initializers/assets.rb — เพิ่มมาให้อัตโนมัติ
Rails.application.config.assets.paths << Rails.root.join("node_modules/bootstrap-icons/font")
Rails.application.config.assets.paths << Rails.root.join("node_modules/bootstrap/dist/js")
Rails.application.config.assets.precompile << "bootstrap.bundle.min.js"
```

ยืนยันด้วย `rails console`:

```ruby
Rails.application.config.assets.paths.each { |p| puts p }
# .../node_modules/bootstrap-icons/font
# .../node_modules/bootstrap/dist/js
# ...
```

พอ render หน้าเว็บจริง จะเห็นไฟล์ JS ถูก fingerprint โดย Propshaft เหมือนไฟล์ asset อื่นๆ ทุก
ประการ แม้ต้นทางจะมาจาก `node_modules`:

```html
<script type="importmap">{
  "imports": {
    "bootstrap": "/assets/bootstrap.bundle.min-2745bfc2.js",
    ...
  }
}</script>
```

นี่คือตัวอย่างที่ดีมากว่า **Propshaft ไม่สนใจเลยว่าไฟล์ asset ต้นทางมาจากไหน** (เขียนเอง, มาจาก
gem, หรือมาจาก `node_modules` ที่ npm จัดการ) — ขอแค่โฟลเดอร์นั้นอยู่ใน load path มันก็ resolve +
fingerprint + serve ให้เหมือนกันหมด สอดคล้องกับสิ่งที่เรียนมาตลอด Part 029

### ใช้งาน Bootstrap Grid/Component จริง

```erb
<nav class="navbar navbar-expand-md navbar-light bg-white border-bottom">
  <div class="container">
    <a class="navbar-brand fw-bold" href="/">MyApp</a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navMenu">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navMenu">
      <ul class="navbar-nav ms-auto">
        <li class="nav-item"><a class="nav-link" href="/">หน้าแรก</a></li>
      </ul>
    </div>
  </div>
</nav>

<div class="container py-5">
  <div class="row g-4">
    <div class="col-12 col-md-6 col-lg-4">
      <div class="card h-100 shadow-sm">
        <div class="card-body">
          <span class="badge bg-primary mb-2">Tailwind</span>
          <h5 class="card-title">เริ่มต้นกับ Tailwind CSS</h5>
          <p class="card-text text-muted">ทำความรู้จัก utility-first CSS</p>
        </div>
      </div>
    </div>
  </div>
</div>
```

**ข้อสังเกตเทียบกับ Tailwind ที่เพิ่งเรียนมา:**

- `navbar-toggler`, `data-bs-toggle="collapse"` — Bootstrap มี **JavaScript component ในตัวเอง**
  (เขียนมาสำเร็จรูปแล้ว ผูกกับ `data-bs-*` attribute) เมนู hamburger ยุบ/ขยายได้**โดยไม่ต้องเขียน
  Stimulus controller เองเลยสักบรรทัด** ต่างจาก Tailwind ที่เป็นแค่ CSS ล้วนๆ ไม่มี JavaScript
  ผูกมาให้ ต้องเขียน Stimulus เองเสมอ (Step 534)
- `row`/`col-12 col-md-6 col-lg-4` — Bootstrap ใช้ระบบ **12-column grid** แบบดั้งเดิม (ตัวเลข
  บวกกันต้องได้ 12 ถ้าอยากเต็มแถว) ต่างจาก Tailwind ที่ใช้ CSS Grid ตรงๆ ผ่าน `grid-cols-N`
  แนวคิดคล้ายกัน (responsive prefix `md:`/`lg:` เทียบกับ `-md-`/`-lg-`) แต่ syntax คนละแบบ
- `card`, `card-body`, `card-title`, `badge` — เป็น **component class สำเร็จรูป** ที่มีสไตล์
  ครบทุกอย่างในตัวเอง (border, shadow, padding, font-size ที่เหมาะสม) ไม่ต้องประกอบ utility
  หลายตัวเองแบบ Tailwind — นี่คือเหตุผลที่คนมักบอกว่า Bootstrap "เริ่มได้เร็วกว่า" (จะอธิบาย
  เพิ่มใน Step 540)

---

## Step 539: Bootstrap แบบเร็วที่สุด — CDN `<link>`/`<script>` สำหรับ Prototype

### เมื่อไหร่ควรข้าม `cssbundling-rails` ไปเลย

ถ้าแค่ต้องการทดลอง prototype เร็วๆ, ทำ demo หน้าเดียว, หรือไม่อยากแตะ Node.js เลยแม้แต่นิดเดียว
Bootstrap มีวิธีใช้งานที่ง่ายกว่ามาก: แปะ `<link>`/`<script>` จาก CDN ตรงๆ ไม่ต้องติดตั้ง gem
เพิ่ม ไม่ต้องรัน generator ใดๆ

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%# ... %>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
        rel="stylesheet"
        crossorigin="anonymous">
</head>
<body>
  <%= yield %>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
          crossorigin="anonymous"></script>
</body>
```

ทำงานได้ทันที ไม่ต้อง `bundle install`, ไม่ต้องแตะ `Gemfile`, ไม่ต้อง precompile อะไรเลย เพราะ
browser โหลดไฟล์ CSS/JS ตรงจาก CDN server ภายนอก **ไม่ผ่าน Propshaft เลย** (เหมือนกับที่เรียนใน
Part 029 Step 288 เรื่อง `image_tag` กับ URL ภายนอก — path ที่ขึ้นต้นด้วย `https://` ไม่ถูก
resolve ผ่าน asset pipeline)

> **หมายเหตุเรื่อง `integrity`:** ในการใช้งานจริงควรเพิ่ม attribute `integrity="sha384-..."`
> (Subresource Integrity — เรียนหลักการไว้แล้วใน Part 029 Step 285) เพื่อยืนยันว่าไฟล์จาก CDN
> ไม่ถูกแก้ไขระหว่างทาง แต่ hash นี้ต้องคัดลอกจากหน้าเว็บทางการของผู้ให้บริการ CDN (เช่น
> jsdelivr.com หรือ getbootstrap.com) ที่ generate hash ให้ตรงกับไฟล์เวอร์ชันนั้นเป๊ะ ห้ามพิมพ์
> hash เดาเอาเองเด็ดขาด เพราะ hash ที่ไม่ตรงจะทำให้ browser **ปฏิเสธโหลดไฟล์ทั้งไฟล์ทันที**

### ข้อดี/ข้อเสียของแนวทาง CDN

**ข้อดี:**

- **เร็วที่สุดเท่าที่จะเป็นไปได้** — เพิ่ม 2 บรรทัดจบ ไม่มีขั้นตอนติดตั้งใดๆ
- **ไม่ต้องมี Node.js แม้แต่นิดเดียว** — ต่างจาก `cssbundling-rails` (Step 538) ตรงนี้เป็น
  ข้อได้เปรียบที่เท่ากับแนวทาง Tailwind มาตรฐาน (Step 532) เพียงแต่ได้ Bootstrap component
  สำเร็จรูปมาแทน utility class
- **CDN มัก cache ไว้แล้วในเครื่องผู้ใช้** — ถ้าผู้ใช้เคยเข้าเว็บอื่นที่ใช้ CDN เดียวกัน (เช่น
  jsDelivr) มาก่อน browser อาจมีไฟล์แคชอยู่แล้ว โหลดเร็วขึ้นอีก (ในทางปฏิบัติปัจจุบันประโยชน์
  ข้อนี้ลดลงมากเพราะ browser สมัยใหม่แยก cache ต่อ origin ของหน้าเว็บแล้ว ไม่ share cache ข้าม
  เว็บไซต์เหมือนสมัยก่อนด้วยเหตุผลด้าน privacy)

**ข้อเสีย (เหตุผลที่ไม่ควรใช้ในโปรเจกต์ production จริง):**

- **ต้องพึ่งอินเทอร์เน็ตภายนอกตลอดเวลาที่เว็บทำงาน** — ถ้า CDN ล่ม หรือถูก block ในบางประเทศ/
  องค์กร (firewall องค์กรบางที่ block `cdn.jsdelivr.net`) หน้าเว็บทั้งเว็บเสียหน้าตาไปทันที
  ต่างจากไฟล์ที่ผ่าน Propshaft ที่ serve จาก server ของเราเอง
- **Customize ไม่ได้เลย** — ได้แค่ CSS/JS ที่ compile ไว้สำเร็จรูปแล้ว เปลี่ยนสีธีมหลัก
  (`$primary` ใน Sass variable ของ Bootstrap) ไม่ได้ ถ้าจะ custom ต้องกลับไปใช้
  `cssbundling-rails` (Step 538) เท่านั้น
- **ไม่มี fingerprinting/cache-busting ที่เราควบคุมได้** — ต้องพึ่ง cache strategy ของ CDN
  ผู้ให้บริการเอง ถ้าอยากอัปเดตเวอร์ชัน Bootstrap ต้องแก้เลขเวอร์ชันในลิงก์เองด้วยมือทุกจุดที่
  อ้างอิงถึง
- **โหลดทั้งไฟล์เต็มเสมอ** — เหมือนปัญหาเดียวกับ Step 538 (CSS ขนาด ~1.1 MB unminified, หรือ
  ~230 KB minified) ไม่มีการตัดส่วนที่ไม่ได้ใช้ออกเลย

> **กฎการเลือกใช้ในทางปฏิบัติ:** CDN เหมาะกับ **prototype ที่ทิ้งได้, demo ให้ลูกค้าดูแบบรีบๆ,
> หรือหน้า internal tool ที่ไม่ต้อง custom หน้าตา** ส่วนโปรเจกต์ที่ตั้งใจดูแลต่อระยะยาวควรใช้
> `cssbundling-rails` (Step 538) เพื่อให้ custom สี/spacing ผ่าน Sass variable ได้ และให้ Propshaft
> ควบคุม cache-busting เอง

---

## Step 540: กรอบการตัดสินใจ — เลือก Tailwind, Bootstrap, หรือ Custom SCSS Pipeline เมื่อไหร่

หลังเห็นทั้งสามแนวทางแบบลงมือทำจริงแล้ว (Tailwind ไม่มี Node, Bootstrap ผ่าน cssbundling ที่มี
Node เต็มรูปแบบ, Bootstrap ผ่าน CDN แบบไม่มี build step) มาสรุปเป็นกรอบการตัดสินใจที่ใช้ได้จริง
ในงานทีม ไม่มีคำตอบเดียวที่ถูกเสมอ — ขึ้นกับบริบทของทีมและโปรเจกต์

### ตารางเปรียบเทียบสรุป

| ประเด็น | Tailwind CSS | Bootstrap (cssbundling) | Bootstrap (CDN) | Custom SCSS Pipeline |
|---|---|---|---|---|
| ต้องมี Node.js | ❌ ไม่ต้อง | ✅ ต้องมี | ❌ ไม่ต้อง | ขึ้นกับเครื่องมือ (มักต้องมี) |
| ความเร็วในการเริ่ม UI ให้ "ดูดี" | ปานกลาง (ต้องประกอบเอง) | เร็วมาก (component สำเร็จรูป) | เร็วที่สุด | ช้าที่สุด (เขียนเองหมด) |
| ควบคุม Design ได้ละเอียดแค่ไหน | สูงมาก (ทุกอย่างเป็น utility) | ปานกลาง (ปรับผ่าน Sass variable) | ต่ำ (แก้ไม่ได้เลย) | สูงสุด (ควบคุม 100%) |
| ขนาด CSS สุดท้าย (production) | เล็กมาก (เฉพาะ class ที่ใช้จริง) | ใหญ่ (framework เต็ม เว้นแต่ตัด Sass import เอง) | ใหญ่ (โหลดทั้งไฟล์เสมอ) | เล็กที่สุด (เขียนเท่าที่ใช้) |
| เหมาะกับทีมขนาดไหน | ทีมที่มี designer/ต้องการ UI เฉพาะตัว | ทีมเล็ก/prototype/back-office | Demo/prototype ทิ้งได้ | ทีมใหญ่ที่มี design system เป็นของตัวเอง |

### กรอบการตัดสินใจแบบเรียงคำถาม

**1. ต้องการควบคุม design เต็มรูปแบบ และทีมพร้อมเรียนรู้ utility class ไหม?**
→ ใช่ → **Tailwind CSS** เหมาะที่สุด โดยเฉพาะทีมที่ทำงานกับ Figma/design system ของตัวเองอยู่
แล้ว เพราะ scale ค่าของ Tailwind (spacing, สี, font-size) แปลงจาก design token ได้ตรงๆ และ
โปรเจกต์ Rails สมัยใหม่ (Rails 8) รองรับแบบไม่ต้องพึ่ง Node.js เลยด้วย

**2. ต้องการ "หน้าตาที่ดูเป็นมืออาชีพทันที" โดยไม่มีเวลา/งบให้ design เอง ไหม?**
→ ใช่ → **Bootstrap** เหมาะกับสถานการณ์นี้มาก — internal tool, admin dashboard, MVP ที่ต้องรีบ
ส่งมอบ, หรือทีมที่ไม่มี designer ประจำ component สำเร็จรูป (`navbar`, `card`, `modal`, `dropdown`)
ทำให้ได้ UI ที่ "ดูโอเค" ทันทีโดยแทบไม่ต้องคิดเรื่อง CSS เลย ถ้าทีมมีเหตุผลให้เลี่ยง Node.js ก็ใช้
วิธี CDN (Step 539) แต่ถ้าต้องการ custom สีธีมหรือ deploy จริงจังก็ใช้ `cssbundling-rails`
(Step 538)

**3. ทีมมีคนอยู่แล้วที่คุ้นเคยหรือ "ลงทุน" กับ framework ใดไปแล้ว?**
→ ทีมที่มี Bootstrap theme/component ที่ custom ไว้เยอะจากโปรเจกต์เก่า มักคุ้มค่ากว่าที่จะใช้ต่อ
แทนการย้ายทั้งหมดมา Tailwind (ต้นทุนการเรียนรู้ + เขียนใหม่ทั้งหมดสูงกว่าประโยชน์ที่ได้ ถ้า
Bootstrap ที่มีอยู่ตอบโจทย์ธุรกิจได้ดีอยู่แล้ว) — "ของที่ทำงานได้ดีอยู่แล้ว ไม่จำเป็นต้องเปลี่ยน
เพราะเทรนด์ใหม่กว่า" เป็นหลักการที่ engineer อาวุโสยึดถือเสมอ

**4. โปรเจกต์เป็นผลิตภัณฑ์ระยะยาวที่ design คือจุดขายหลัก (design-system-heavy product) ไหม?**
→ ใช่ → พิจารณา **Custom SCSS Pipeline เต็มรูปแบบ** (เขียน component CSS เองทั้งหมด ไม่พึ่ง
framework สำเร็จรูปเลย อาจผสม CSS custom properties + BEM naming) เหมาะกับผลิตภัณฑ์ที่ต้องการ
เอกลักษณ์ทางภาพที่ไม่เหมือนใคร (เช่น Basecamp, Linear, Stripe) ทีมต้องมี designer/frontend
engineer ที่แข็งแรงพอจะดูแล design system เอง แนวทางนี้ให้ผลลัพธ์ที่ดีที่สุดในระยะยาวถ้าทำถูกต้อง
แต่ต้นทุนเริ่มต้นสูงที่สุดในบรรดาทั้งหมด และมักเริ่มจาก Tailwind เป็นฐานแล้วค่อยๆ ดึง component
ที่ซับซ้อนออกมาเป็น ViewComponent ของตัวเอง (Part 055) มากกว่าจะเขียน CSS ล้วนๆ ตั้งแต่ศูนย์

> **ข้อสรุปเชิงปฏิบัติสำหรับหลักสูตรนี้:** ตั้งแต่ Part นี้เป็นต้นไป ตัวอย่างส่วนใหญ่ในหลักสูตร
> จะใช้ **Tailwind CSS** เป็นหลัก เพราะสอดคล้องกับ default ของ Rails 8 (`--css=tailwind`) และให้
> ควบคุมได้ละเอียดพอจะสอนแนวคิด responsive design ได้ครบถ้วนที่สุด แต่ทุกความรู้เรื่อง
> responsive breakpoint, mobile-first, และการแตก partial/helper ที่เรียนใน Part นี้ **ใช้ได้กับ
> ทุก CSS framework ไม่ว่าจะเป็น Bootstrap หรือ framework อื่นในอนาคต** เพราะเป็นแนวคิดระดับ CSS/
> Rails view-layer ไม่ใช่เทคนิคเฉพาะของ Tailwind

---

## แบบฝึกหัด: Responsive Post Card Grid ด้วย Tailwind แบบ Mobile-First ฉบับสมบูรณ์

### โจทย์

สร้างหน้าแรก (`root_path`) ของแอปที่รวมทุกอย่างที่เรียนมาใน Part นี้เข้าด้วยกัน:

1. Nav bar ที่ยุบเป็นเมนูมือถือ พร้อมปุ่มสลับ dark mode (Step 534, 537)
2. Card grid ของโพสต์ที่ปรับจำนวนคอลัมน์แบบ mobile-first: 1 คอลัมน์ (มือถือ) → 2 คอลัมน์ (`sm:`)
   → 3 คอลัมน์ (`lg:`) (Step 535)
3. แต่ละการ์ดต้องมี badge หมวดหมู่ที่ใช้ helper method (ไม่ใช่พิมพ์ class ซ้ำ) (Step 536)
4. รองรับ dark mode เต็มรูปแบบทั้งหน้า (Step 537)
5. ยืนยันผลลัพธ์ด้วยการ build จริงและ `curl`/inspect HTML ว่า class ที่ต้องการ render ออกมาครบ

### เฉลย

**1) Model ข้อมูลจำลอง (ยังไม่ผูก database จริง เพื่อโฟกัสที่ view layer):**

```ruby
# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  Post = Struct.new(:title, :excerpt, :author, :published_on, :tag)

  def home
    @posts = [
      Post.new("เริ่มต้นกับ Tailwind CSS",
                "ทำความรู้จัก utility-first CSS และวิธีคิดที่ต่างจาก CSS แบบเดิม",
                "มานี", "26 ก.ย. 2026", "Tailwind"),
      Post.new("Responsive Layout ด้วย Grid",
                "สร้างการ์ดที่ปรับเปลี่ยนจำนวนคอลัมน์ตามขนาดหน้าจอโดยอัตโนมัติ",
                "สมชาย", "24 ก.ย. 2026", "CSS"),
      Post.new("Dark Mode แบบไม่ปวดหัว",
                "ใช้ prefix dark: ของ Tailwind ทำสลับธีมมืด/สว่างได้ในไม่กี่บรรทัด",
                "วิภา", "20 ก.ย. 2026", "UI"),
      Post.new("Bootstrap vs Tailwind",
                "เปรียบเทียบแนวคิด ข้อดีข้อเสีย และกรณีที่ควรเลือกใช้แต่ละตัว",
                "ปรีชา", "18 ก.ย. 2026", "เปรียบเทียบ"),
    ]
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "pages#home"
end
```

**2) Helper สำหรับ badge (Step 536):**

```ruby
# app/helpers/pages_helper.rb
module PagesHelper
  def tag_badge(text)
    tag.span text,
      class: "inline-block rounded-full bg-blue-100 px-2.5 py-0.5 text-xs font-semibold
              text-blue-800 dark:bg-blue-900 dark:text-blue-200"
  end
end
```

**3) Nav bar partial พร้อม dark mode toggle (Step 534, 537):**

```erb
<%# app/views/shared/_navbar.html.erb %>
<nav class="bg-white dark:bg-slate-900 border-b border-slate-200 dark:border-slate-700"
     data-controller="nav">
  <div class="mx-auto max-w-5xl px-4">
    <div class="flex h-16 items-center justify-between">
      <%= link_to "MyApp", root_path,
            class: "text-lg font-bold text-slate-900 dark:text-white" %>

      <div class="hidden md:flex md:items-center md:gap-6">
        <%= link_to "หน้าแรก", root_path,
              class: "text-sm font-medium text-slate-600 hover:text-slate-900 dark:text-slate-300 dark:hover:text-white" %>
        <%= link_to "บทความ", root_path,
              class: "text-sm font-medium text-slate-600 hover:text-slate-900 dark:text-slate-300 dark:hover:text-white" %>

        <button type="button" data-controller="theme" data-action="theme#toggle"
                class="rounded-lg border border-slate-300 px-3 py-1.5 text-sm font-medium
                       text-slate-700 hover:bg-slate-100
                       dark:border-slate-600 dark:text-slate-200 dark:hover:bg-slate-800">
          🌓 สลับธีม
        </button>
      </div>

      <button type="button" data-action="nav#toggle"
              class="md:hidden inline-flex items-center justify-center rounded-lg p-2
                     text-slate-600 hover:bg-slate-100 dark:text-slate-300 dark:hover:bg-slate-800">
        <span class="sr-only">เปิดเมนู</span>
        ☰
      </button>
    </div>

    <div data-nav-target="menu" class="hidden md:hidden space-y-1 pb-4">
      <%= link_to "หน้าแรก", root_path,
            class: "block rounded-lg px-3 py-2 text-sm font-medium text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-800" %>
      <%= link_to "บทความ", root_path,
            class: "block rounded-lg px-3 py-2 text-sm font-medium text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-800" %>
      <button type="button" data-controller="theme" data-action="theme#toggle"
              class="w-full text-left rounded-lg px-3 py-2 text-sm font-medium text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-800">
        🌓 สลับธีม
      </button>
    </div>
  </div>
</nav>
```

**4) Card partial (Step 535, 536):**

```erb
<%# app/views/pages/_post_card.html.erb %>
<article class="flex flex-col rounded-xl border border-slate-200 bg-white p-5 shadow-sm
                transition hover:shadow-md
                dark:border-slate-700 dark:bg-slate-800">
  <div class="mb-3 flex items-center justify-between">
    <%= tag_badge(post.tag) %>
    <span class="text-xs text-slate-400 dark:text-slate-500"><%= post.published_on %></span>
  </div>

  <h2 class="mb-2 text-lg font-bold text-slate-900 dark:text-white"><%= post.title %></h2>
  <p class="mb-4 flex-1 text-sm text-slate-600 dark:text-slate-300"><%= post.excerpt %></p>

  <div class="flex items-center gap-2 border-t border-slate-100 pt-3 dark:border-slate-700">
    <span class="flex h-8 w-8 items-center justify-center rounded-full bg-slate-200
                 text-xs font-semibold text-slate-700 dark:bg-slate-700 dark:text-slate-200">
      <%= post.author.first %>
    </span>
    <span class="text-sm font-medium text-slate-700 dark:text-slate-200"><%= post.author %></span>
  </div>
</article>
```

**5) หน้าแรกที่ประกอบทุกอย่างเข้าด้วยกัน:**

```erb
<%# app/views/pages/home.html.erb %>
<%= render "shared/navbar" %>

<main class="mx-auto max-w-5xl px-4 py-10">
  <h1 class="mb-2 text-3xl font-bold text-slate-900 dark:text-white">บทความล่าสุด</h1>
  <p class="mb-8 text-slate-500 dark:text-slate-400">
    ตัวอย่าง responsive card grid ที่สร้างด้วย Tailwind utility class ล้วนๆ
  </p>

  <%# mobile-first: เริ่มจาก 1 คอลัมน์ (ค่า default ไม่มี prefix) แล้วค่อยเพิ่มคอลัมน์
      เมื่อจอกว้างขึ้นด้วย sm:/lg: %>
  <div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
    <% @posts.each do |post| %>
      <%= render "post_card", post: post %>
    <% end %>
  </div>
</main>
```

```erb
<%# app/views/layouts/application.html.erb — ตัด <main> ที่ generator ใส่มาให้ทิ้ง
    เพราะเราต้องการให้ nav bar เต็มความกว้างจอ ไม่ถูกจำกัดด้วย container ของ layout %>
<body class="bg-slate-50 dark:bg-slate-950">
  <%= yield %>
</body>
```

```css
/* app/assets/tailwind/application.css */
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));
```

### ตรวจสอบผลลัพธ์จริง

```bash
bin/rails tailwindcss:build
bin/rails server -d
curl -s http://localhost:3000/ -o /tmp/home.html
grep -o "grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3" /tmp/home.html
grep -c "inline-block rounded-full bg-blue-100" /tmp/home.html
grep -o "data-controller=\"nav\"" /tmp/home.html
grep -o "data-controller=\"theme\"" /tmp/home.html
```

ผลลัพธ์จริงที่ยืนยันแล้ว:

```
grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3
4
data-controller="nav"
data-controller="theme"
data-controller="theme"
```

`4` คือจำนวนครั้งที่ badge (จาก helper `tag_badge`) render ออกมา ตรงกับจำนวนโพสต์ทั้งหมด — ยืนยัน
ว่า helper method ทำงานถูกต้องและใช้ class เดียวกันทุกจุดโดยไม่ต้องพิมพ์ซ้ำเอง ส่วน
`data-controller="theme"` เจอ 2 ครั้งเพราะมีปุ่มสลับธีมทั้งในเมนูปกติและเมนูมือถือ (ทั้งคู่ผูกกับ
Stimulus controller เดียวกัน)

ตรวจสอบ CSS ที่ compile ออกมาว่ามี class ทุกตัวที่ใช้จริง (ไม่มี class ไหนหายไปเพราะสแกนไม่เจอ):

```bash
for cls in "grid-cols-1" "sm\\:grid-cols-2" "lg\\:grid-cols-3" "dark\\:bg-slate-900" "hover\\:shadow-md"; do
  echo -n "$cls: "
  grep -c "$cls" app/assets/builds/tailwind.css
done
```

```
grid-cols-1: 1
sm\:grid-cols-2: 1
lg\:grid-cols-3: 1
dark\:bg-slate-900: 1
hover\:shadow-md: 1
```

ทุก class เจอครบ — พิสูจน์ว่า mobile-first responsive grid, dark mode, และ hover state ทำงานถูก
ต้องทั้งหมดตั้งแต่ระดับ build จนถึงระดับ HTML output จริง เปิด browser จริงแล้วลองปรับความกว้าง
หน้าต่างดู จะเห็นการ์ดปรับจาก 1 → 2 → 3 คอลัมน์ที่ 640px และ 1024px พอดี กดปุ่ม "สลับธีม" แล้ว
ทั้งหน้าเปลี่ยนเป็นธีมมืดทันทีโดยไม่ reload หน้าเว็บ

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มปุ่ม "กรองตามหมวดหมู่" เหนือ card grid ที่กดแล้ว highlight badge ของหมวดหมู่ที่เลือกด้วย
   `ring-2 ring-blue-500` (Tailwind utility สำหรับใส่ outline แบบ ring) ใช้ Stimulus controller
   ตัวใหม่จัดการ state ว่ากำลังเลือกหมวดหมู่ไหนอยู่ (ไม่ต้อง filter จริงก็ได้ แค่ toggle
   class ให้เห็นภาพ)
2. เขียนหน้าเดียวกันนี้ใหม่ทั้งหมดด้วย **Bootstrap** แทน (ใช้ `navbar`, `card`, `row`/`col-*`
   ตามที่เรียนใน Step 538) แล้วเทียบจำนวนบรรทัด CSS ที่ต้องเขียนเอง กับความยาวของ
   `application.bootstrap.scss` เทียบกับ `app/assets/tailwind/application.css` — สรุปเป็น
   ข้อสังเกตของตัวเองว่า framework ไหนทำให้ "เขียน CSS เอง" น้อยกว่ากัน และ framework ไหนทำให้
   "ควบคุมหน้าตาได้ละเอียดกว่า" มากกว่ากัน
3. ลองรัน `bin/rails tailwindcss:build --minify` (production build) แล้วเทียบขนาดไฟล์กับตอน
   build แบบ development เทียบเปอร์เซ็นต์ที่ลดลง แล้วลองลบ post การ์ดออกให้เหลือ 1 ใบ build ใหม่
   อีกครั้ง สังเกตว่าขนาดไฟล์ CSS เปลี่ยนไปหรือไม่ (ใบ้: ควรไม่เปลี่ยนมาก เพราะ class ที่การ์ด
   ใช้ยังถูกใช้อยู่ในโค้ด แม้จะ render แค่ใบเดียว — Tailwind สแกนจาก**โค้ด** ไม่ได้สแกนจาก
   **จำนวนครั้งที่ render จริง**)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปรัชญา **utility-first CSS** ของ Tailwind: ประกอบ UI จาก class เล็กๆ ที่ทำหน้าที่เดียว
  แทนการตั้งชื่อ semantic class เอง พร้อม tradeoff ที่แท้จริง (HTML รกขึ้นแต่ไม่ต้อง
  context-switch, ไม่ต้องคิดชื่อ, CSS ไม่บวมแบบไม่มีที่สิ้นสุดเพราะ generate เฉพาะ class ที่
  ใช้จริง)
- ติดตั้งและยืนยันจริงว่า `rails new myapp --css=tailwind` **ไม่ต้องมี Node.js เลย** เพราะ
  `tailwindcss-rails` ห่อ standalone binary ของ Tailwind CLI ไว้ผ่าน platform-specific gem
  `tailwindcss-ruby` — ตรวจสอบได้ว่าไม่มี `package.json`/`node_modules` เกิดขึ้นในโปรเจกต์เลย
- รู้จักโครงสร้างไฟล์ที่ได้ (`app/assets/tailwind/application.css`,
  `app/assets/builds/tailwind.css`), `bin/dev`/`Procfile.dev` ที่รัน `bin/rails server` และ
  `bin/rails tailwindcss:watch` พร้อมกันผ่าน foreman, และพิสูจน์แล้วว่า Propshaft fingerprint
  และ serve ไฟล์ CSS ของ Tailwind เหมือนไฟล์ CSS ปกติทุกประการ (ตามหลักการจาก Part 029)
- สร้าง **responsive component จริง**: nav bar ที่ยุบเป็นเมนูมือถือด้วย `hidden md:flex` /
  `md:hidden` (mobile-first, ใช้ `min-width` breakpoint) ผสม Stimulus แค่สำหรับ toggle state
  ที่ CSS ทำเองไม่ได้ และ card grid ที่ปรับคอลัมน์ด้วย `grid-cols-1 sm:grid-cols-2
  lg:grid-cols-3`
- แก้ปัญหา "class soup" ด้วย **View Partial** และ **Helper method** ของ Rails เอง แทนการใช้
  `@apply` — เข้าใจเหตุผลว่าทำไมแนวทางนี้เหมาะกับ Rails มากกว่า (ไม่มี build step ซ้อน, ใช้
  เครื่องมือ reuse เดียวกับที่ใช้ทั่วทั้งแอปอยู่แล้ว) และรู้ข้อควรระวังเรื่อง class string ที่
  ต้องปรากฏเต็มๆ ในโค้ดให้ Tailwind สแกนเจอ
- ทำ **dark mode** ด้วย `@custom-variant dark` (สลับจาก `prefers-color-scheme` เป็น class-based)
  และ prefix `dark:` พร้อมปุ่ม toggle ที่เขียนด้วย Stimulus จำค่าไว้ใน `localStorage`
- ติดตั้ง **Bootstrap ผ่าน `cssbundling-rails`** (`rails css:install:bootstrap`) และเห็นความต่าง
  ที่ชัดเจนจาก Tailwind: ต้องมี Node.js/npm จริง มี `package.json`/`node_modules` เต็มโฟลเดอร์
  ใช้ Dart Sass compile Bootstrap ทั้งเฟรมเวิร์ก (ไฟล์ CSS ใหญ่กว่า Tailwind มาก) และเห็นว่า
  Propshaft ยัง fingerprint/serve ไฟล์จาก `node_modules` ได้เหมือนเดิมถ้าอยู่ใน load path
- รู้จักทางเลือก **Bootstrap ผ่าน CDN** สำหรับ prototype ด่วนที่ไม่ต้องติดตั้งอะไรเลย พร้อม
  ข้อเสียเรื่องพึ่งพาอินเทอร์เน็ตภายนอกและ customize ไม่ได้
- มีกรอบการตัดสินใจที่ใช้ได้จริง: **Tailwind** สำหรับทีมที่ต้องการควบคุม design เต็มรูปแบบ,
  **Bootstrap** สำหรับ prototype/internal tool ที่ต้องการ "ดูดีทันที" หรือทีมที่ลงทุนกับมันแล้ว,
  **Custom SCSS pipeline** สำหรับผลิตภัณฑ์ที่ design คือจุดขายหลักและมีทีมพร้อมดูแลระยะยาว

**ต่อไป (Part 055):** ตอนนี้เรารู้จักการแตก class soup ด้วย View Partial และ Helper แล้ว แต่ทั้ง
สองเครื่องมือนี้เริ่มไม่พอเมื่อ component ต้องมี logic ซับซ้อนขึ้น (รับ slot หลายแบบ, ต้อง
validate input, ต้องเขียน test แยกจาก view) Part 055 จะแนะนำ **ViewComponent** — เฟรมเวิร์ก
component-based UI ที่ทำให้เขียน Ruby class ที่มี template ของตัวเอง ทดสอบได้แยกจาก controller/
request ทั้งระบบ และนำ card/badge/nav bar ที่สร้างไว้ใน Part นี้ไป refactor เป็น component ที่
reuse ได้แข็งแรงกว่าเดิม
