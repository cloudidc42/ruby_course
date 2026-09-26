# Part 029: Asset Pipeline — Propshaft, Static Asset, image_tag, link_to

> **Step ครอบคลุมใน Part นี้:** Step 281–290
> **ระดับ:** กลาง (ต่อจาก Part 028 เรื่อง Scaffold และ Generator)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x — **Propshaft** เป็น asset pipeline เริ่มต้น,
> **importmap-rails** สำหรับ JavaScript (ไม่ต้องมี Node.js/Webpack/esbuild ก็ใช้ได้)
> ทุกตัวอย่างใน Part นี้ทดสอบจริงด้วย `rails new` (เวอร์ชันเต็ม ไม่ใช้ `--minimal`) บน Rails 8.1.4

ใน Part 024 เราใช้ `image_tag` และ `link_to` มาแล้วในฐานะ view helper ธรรมดา — เรียกแล้วได้
`<img>`/`<a>` tag ออกมา ไม่ได้ลงลึกว่าเบื้องหลังมันหา path ของไฟล์รูปหรือ path ของ route
ได้อย่างไร ใน Part นี้เราจะเจาะลึกฝั่ง **static asset** (CSS, JavaScript, รูปภาพ, ฟอนต์)
โดยเฉพาะ: asset pipeline คืออะไร, ทำไม Rails 8 ถึงเปลี่ยนมาใช้ **Propshaft** แทน Sprockets
ตัวเก่า, กลไกการ fingerprint ไฟล์เพื่อทำ cache-busting, และวิธีที่ Rails ส่ง JavaScript ไปหา
browser แบบไม่ต้องมี build step ผ่าน **importmap-rails**

## สารบัญของ Part นี้

- Step 281: Asset Pipeline คืออะไร และทำไมเว็บแอปทุกตัวต้องมีระบบนี้
- Step 282: Sprockets vs Propshaft — ประวัติโดยย่อ และทำไม Rails 8 เลือกทางที่ "ทำน้อยลง"
- Step 283: โครงสร้าง `app/assets/` และ Load Path ของ Propshaft
- Step 284: กลไก Resolve และ Fingerprint เบื้องหลัง Cache-busting
- Step 285: `stylesheet_link_tag`, `image_tag`, `asset_path` — Helper ที่ผูกกับ Propshaft
- Step 286: `importmap-rails` — จัดการ JavaScript แบบไม่มี Node.js Build Step
- Step 287: เพิ่ม Stylesheet ของตัวเอง และอ้างอิงใช้งานจริง
- Step 288: เพิ่มรูปภาพของตัวเอง และใช้งานผ่าน `image_tag`
- Step 289: `bin/rails assets:precompile` — เตรียม Asset สำหรับ Production
- Step 290: เมื่อไหร่ควรมองหา esbuild/Vite/Node Bundling แทน importmap
- แบบฝึกหัด: หน้าแรกที่มี Custom Stylesheet, โลโก้ และ JS Behavior ผ่าน Importmap

---

## Step 281: Asset Pipeline คืออะไร และทำไมเว็บแอปทุกตัวต้องมีระบบนี้

เว็บแอปพลิเคชันไม่ได้มีแค่ HTML ที่ Rails render จาก ERB เท่านั้น มันยังต้องส่ง **CSS**
(สไตล์การแสดงผล), **JavaScript** (พฤติกรรมโต้ตอบฝั่ง browser), **รูปภาพ** และ **ฟอนต์**
ไปให้ browser โหลดด้วยเสมอ ไฟล์เหล่านี้เรียกรวมๆ ว่า **static asset** (ต่างจาก HTML ที่
render แบบ dynamic ทุกครั้งตาม request)

ถ้าไม่มีระบบจัดการอะไรเลย แนวทางที่ตรงไปตรงมาที่สุดคือเขียน path ตรงๆ ในไฟล์ view:

```erb
<link rel="stylesheet" href="/css/style.css">
<img src="/img/logo.png">
```

วิธีนี้ใช้งานได้ แต่มีปัญหาสำคัญ 2 อย่างที่โผล่มาทันทีเมื่อแอปโตขึ้นระดับ production จริง:

### ปัญหาที่ 1: Cache-busting

Browser (และ CDN ระหว่างทาง) จะ**แคชไฟล์ static ไว้อย่างดุดัน**เพื่อประสิทธิภาพ — ยิ่งแคชนาน
ยิ่งลดจำนวน request ที่ต้องยิงกลับมาที่ server ซ้ำ ปัญหาคือ: ถ้าไฟล์ `style.css` เดิมถูกแคชไว้
30 วัน แล้ววันพรุ่งนี้เราแก้สีปุ่มใน CSS ไฟล์เดียวกัน ผู้ใช้ที่แคชไว้แล้วจะ**ไม่เห็นการ
เปลี่ยนแปลงเลย** จนกว่าแคชจะหมดอายุหรือกด hard refresh — ในทางปฏิบัติ deploy โค้ดใหม่แล้ว
แต่ผู้ใช้ยังเห็นของเก่าอยู่คือบั๊กที่น่ารำคาญมาก

วิธีแก้แบบดั้งเดิม (ยุคก่อนมี asset pipeline) คือแปะ query string เปลี่ยนเลขเวอร์ชันเอามือ
เช่น `style.css?v=2` — ใช้งานได้แต่ต้องจำเปลี่ยนเลขเองทุกครั้ง ลืมบ่อยมาก

### ปัญหาที่ 2: การจัดระเบียบไฟล์จำนวนมาก

โปรเจกต์จริงมักมี CSS/JS กระจายเป็นหลายสิบไฟล์ (แยกตามหน้า, แยกตาม component) จะ organize
ยังไงให้แต่ละหน้าโหลดเฉพาะไฟล์ที่ต้องใช้ ไม่ใช่โหลดทุกไฟล์ทุกหน้า และจะอ้างอิง path ของไฟล์
โดยไม่ hardcode ตำแหน่งจริงบนดิสก์อย่างไร

**Asset Pipeline** คือระบบที่ Rails สร้างมาเพื่อแก้ปัญหาทั้งสองข้อนี้ (และในยุคก่อนหน้ายังทำ
งานอื่นเพิ่มด้วย เช่น bundling/minification ซึ่งเราจะเล่าใน Step 282) หัวใจสำคัญที่สุดของมัน
คือกลไกที่เรียกว่า **fingerprinting**: เปลี่ยนชื่อไฟล์ให้มี hash ของเนื้อหาไฟล์นั้นต่อท้าย
เช่น `style.css` กลายเป็น `style-8b441ae0.css`

ผลลัพธ์คือ:

1. ตราบใดที่เนื้อหาไฟล์ไม่เปลี่ยน ชื่อไฟล์ก็ไม่เปลี่ยน → สั่งให้ browser/CDN **แคชได้นานสุด
   เท่าที่จะนานได้** (1 ปี) โดยไม่มีความเสี่ยงเรื่องข้อมูลเก่าค้าง
2. พอเนื้อหาไฟล์เปลี่ยนแม้แต่ตัวอักษรเดียว hash จะเปลี่ยน → ได้ชื่อไฟล์ใหม่ทันที → เท่ากับ
   URL ใหม่ → browser ที่ไม่เคยเห็น URL นี้มาก่อนจะดึงไฟล์ใหม่มาเสมอ **โดยอัตโนมัติ ไม่ต้องมี
   ใครมานั่งเปลี่ยนเลขเวอร์ชันเอง**

เรื่อง organize ไฟล์และการอ้างอิง path ไม่ให้ hardcode ก็แก้ด้วยการมี **helper method**
(`image_tag`, `stylesheet_link_tag`, `asset_path`) ที่คำนวณ path จริง (พร้อม fingerprint) ให้
เราตอน render แทนที่จะเขียน path ตรงๆ เอง — เราใช้ helper พวกนี้มาตั้งแต่ Part 024 แล้ว
แต่ยังไม่เห็นว่าเบื้องหลังมันทำอะไรบ้าง — นี่คือสิ่งที่ Part นี้จะเจาะลึก

---

## Step 282: Sprockets vs Propshaft — ประวัติโดยย่อ และทำไม Rails 8 เลือกทางที่ "ทำน้อยลง"

### ยุค Sprockets (Rails 3.1 – Rails 7, ยังใช้ได้ถ้าติดตั้งเพิ่มเอง)

**Sprockets** คือ asset pipeline ตัวแรกที่ Rails ผนวกเข้ามาเป็นค่า default ตั้งแต่ Rails 3.1
(ปี 2011) นอกจากจะทำ fingerprinting แล้ว Sprockets ยังทำงานอีกหลายอย่างในตัวเดียว:

- **Bundling/Concatenation** — รวมหลายไฟล์ CSS/JS เข้าเป็นไฟล์เดียวผ่าน directive แบบ
  `//= require_tree .` หรือ `//= require jquery` ในไฟล์ manifest (`application.js`,
  `application.css`)
- **Minification** — บีบอัดโค้ด CSS/JS ให้เหลือขนาดเล็กที่สุดตอน precompile สำหรับ production
- **Transpilation/Preprocessing** — คอมไพล์ Sass/SCSS เป็น CSS, คอมไพล์ CoffeeScript เป็น
  JavaScript ผ่าน pipeline เดียวกัน
- ต้องมีไฟล์ `app/assets/config/manifest.js` ประกาศชัดเจนว่าจะ compile/link อะไรบ้าง

เหตุผลที่ Sprockets ออกแบบมาแบบนี้ **สมเหตุสมผลมากในปี 2011**:

- ยุคนั้นเป็น **HTTP/1.1** ซึ่ง browser เปิด connection แบบขนานได้จำกัด (ปกติ 6 connection
  ต่อ domain) การยิง request แยกไฟล์เป็นสิบๆ ไฟล์ทำให้หน้าเว็บโหลดช้าลงจริง — การรวมทุกไฟล์
  เป็นไฟล์เดียวจึงช่วยเรื่อง performance ได้จริง
- Browser ยุคนั้น**ไม่รองรับ ES Modules** (`import`/`export`) เลย ต้องพึ่งการรวมไฟล์และ wrap
  ด้วย pattern อย่าง IIFE เพื่อจำลอง module
- CSS ยุคนั้นไม่มี custom properties (`--var`), ไม่มี nesting ในตัว — การใช้ Sass เพื่อได้
  ความสามารถพวกนี้แทบจำเป็น

### ยุค Propshaft (Rails 8 default, เปิดตัวจริงใน Rails 7.1 เป็นทางเลือก)

**Propshaft** คือ asset pipeline ที่ทีม Rails เขียนขึ้นใหม่โดย**ตั้งใจทำน้อยกว่า Sprockets
มาก** หน้าที่ของมันแคบลงเหลือแค่ 2 อย่าง:

1. หาไฟล์ asset จาก **load path** (โฟลเดอร์ที่กำหนดไว้) แล้วคำนวณ fingerprint จากเนื้อหาไฟล์
2. Serve ไฟล์นั้นด้วย path ที่มี fingerprint ต่อท้าย พร้อม cache header ที่เหมาะสม

**Propshaft ไม่ทำ bundling, ไม่ทำ minification, และไม่ทำ preprocessing ใดๆ ในตัวมันเอง**
ไฟล์ CSS แต่ละไฟล์ยังเป็นไฟล์แยกกันเสมอ ไม่ถูกรวมเป็นไฟล์เดียว (จะพิสูจน์ให้เห็นจริงด้วยการ
render จริงใน Step 285)

ทำไมทีม Rails ถึงกล้าตัดความสามารถที่เคยมีออกไปในเวอร์ชัน 8 ซึ่งเป็นเวอร์ชันหลัก ไม่ใช่แค่
ทดลอง — มีเหตุผลเชิงเทคนิคที่เปลี่ยนไปจริงตลอด 10+ ปีที่ผ่านมา:

1. **HTTP/2 (และ HTTP/3) กลายเป็นมาตรฐาน** — connection เดียวรองรับหลาย stream พร้อมกันได้
   (multiplexing) การยิง request แยกไฟล์สิบๆ ไฟล์ผ่าน HTTP/2 **แทบไม่มีค่าใช้จ่ายส่วนเกิน**
   เหมือนสมัย HTTP/1.1 อีกต่อไป เหตุผลหลักที่เคยต้อง bundle ไฟล์รวมกันจึงอ่อนลงมาก
2. **Native ES Modules รองรับในทุก browser สมัยใหม่แล้ว** (ตั้งแต่ราวปี 2018 เป็นต้นมา) —
   เขียน `import`/`export` ใน `<script type="module">` แล้วให้ browser จัดการ resolve เองได้
   เลย ไม่ต้องมี bundler แปลงให้ก่อน (รายละเอียดใน Step 286 เรื่อง importmap)
3. **CSS สมัยใหม่มีความสามารถที่เคยต้องพึ่ง Sass** อยู่ในตัวแล้ว (CSS custom properties,
   nesting ในบาง browser) ทีมที่ยังต้องการ Sass/Tailwind จริงๆ ก็ติดตั้งเครื่องมือเฉพาะทาง
   เพิ่มได้ (`dartsass-rails`, `tailwindcss-rails`, `cssbundling-rails`) โดยไม่ต้องแบก
   ความสามารถนั้นไว้ใน core ของทุกแอป
4. **ลดความซับซ้อนของ default stack** — Sprockets มี "เวทมนตร์" เยอะ (`//= require_tree`,
   asset manifest compilation, sprockets directive comment syntax) ที่มือใหม่งงบ่อยเวลาไฟล์
   ไม่โหลดตามคาด และการ boot ต้อง scan/hash ทั้ง asset tree ทำให้ boot ช้าในโปรเจกต์ใหญ่
   Propshaft มีโค้ดหลักเพียงไม่กี่ร้อยบรรทัด (ดูได้จาก `gem contents propshaft` มีไฟล์แค่
   ~15 ไฟล์) ทำงานเข้าใจง่าย debug ง่าย

ตารางเปรียบเทียบสรุป:

| ความสามารถ | Sprockets | Propshaft |
|---|---|---|
| Fingerprinting (cache-busting) | ✅ | ✅ |
| Bundling หลายไฟล์เป็นไฟล์เดียว | ✅ (`require_tree`) | ❌ (ไฟล์แยกกันเสมอ) |
| Minification | ✅ | ❌ |
| Sass/SCSS/CoffeeScript ในตัว | ✅ | ❌ (ต้องเพิ่ม gem เฉพาะทาง) |
| ต้องมี `app/assets/config/manifest.js` | ✅ | ❌ (ไม่มี concept นี้เลย) |
| ต้องมี Node.js ถึงจะใช้งานพื้นฐานได้ | ไม่จำเป็น | ไม่จำเป็น |
| ขนาดโค้ดของ gem (ความซับซ้อน) | ใหญ่ | เล็กมาก (~500 บรรทัด) |

> **สรุปแนวคิดสำคัญ:** Propshaft ไม่ได้ "แย่กว่า" Sprockets — มันแค่**ตั้งใจรับผิดชอบให้น้อย
> ลง** โดยอาศัยว่า browser และโปรโตคอลเครือข่ายสมัยใหม่ทำงานหลายอย่างที่ Sprockets เคยต้องทำ
> แทนให้แล้ว ส่วนงานที่ยังจำเป็นจริงๆ (เช่น compile Tailwind, bundle React) ก็ผลักให้เป็น
> ความรับผิดชอบของเครื่องมือเฉพาะทางแยกต่างหาก แทนที่จะฝังไว้ใน core ของทุกแอป Rails —
> ปรัชญานี้สอดคล้องกับที่ Rails 8 เลือกใช้ importmap เป็นค่า default สำหรับ JavaScript ด้วย
> (Step 286) โปรเจกต์เก่าที่ยังพึ่ง Sprockets เต็มรูปแบบยังอัปเกรดมาใช้ Propshaft ไม่ได้ทันที
> ถ้ายังต้องพึ่งความสามารถ bundling/minification ในตัว แต่โปรเจกต์ใหม่ที่เริ่มจาก Rails 8
> ได้ประโยชน์จากความเรียบง่ายนี้เต็มๆ

---

## Step 283: โครงสร้าง `app/assets/` และ Load Path ของ Propshaft

สร้างแอปใหม่ด้วย `rails new myapp` (ไม่ใส่ `--minimal`) แล้วดูโครงสร้างโฟลเดอร์ asset ที่ได้
จริง:

```bash
find app/assets -type f
```

```
app/assets/images/.keep
app/assets/stylesheets/application.css
```

สังเกตว่า **ไม่มีไฟล์ `app/assets/config/manifest.js`** เลย — ในยุค Sprockets ไฟล์นี้จำเป็น
มาก (ต้องเขียน `//= link_tree ../images` ประกาศชัดเจนว่าจะรวมโฟลเดอร์ไหนเข้า pipeline บ้าง)
Propshaft **ไม่มี concept ของ manifest แบบนั้นเลย** — ไฟล์ไหนก็ตามที่อยู่ในโฟลเดอร์ที่ถูก
ลงทะเบียนไว้ใน **load path** จะถูกมองเห็นและ resolve ได้ทันที ไม่ต้องประกาศอะไรเพิ่ม

### Load Path คืออะไร

Load path คือรายการโฟลเดอร์ทั้งหมดที่ Propshaft จะค้นหาไฟล์ asset ดูได้จริงด้วยคำสั่งนี้ใน
`rails console`:

```ruby
Rails.application.config.assets.paths.each { |p| puts p }
```

ผลลัพธ์จริงจากแอปที่ generate ด้วย Rails 8.1.4 (มี Propshaft + importmap-rails + turbo-rails
+ stimulus-rails):

```
/path/to/app/app/assets/images
/path/to/app/app/assets/stylesheets
/path/to/app/app/javascript
/path/to/app/vendor/javascript
.../gems/stimulus-rails-1.3.4/app/assets/javascripts
.../gems/turbo-rails-2.0.23/app/assets/javascripts
.../gems/actiontext-8.1.4/app/assets/javascripts
.../gems/action_text-trix-2.1.19/app/assets/javascripts
.../gems/action_text-trix-2.1.19/app/assets/stylesheets
.../gems/actioncable-8.1.4/app/assets/javascripts
.../gems/activestorage-8.1.4/app/assets/javascripts
.../gems/actionview-8.1.4/app/assets/javascripts
```

ข้อสังเกตสำคัญ:

1. `app/assets/images` และ `app/assets/stylesheets` คือโฟลเดอร์หลักที่เราจะใส่ไฟล์ของ
   ตัวเอง
2. **`app/javascript` และ `vendor/javascript` ก็อยู่ใน load path ของ Propshaft ด้วย** —
   นี่คือเหตุผลที่ JavaScript ที่เขียนเองใน `app/javascript/` ถูก Propshaft resolve และ
   fingerprint ได้เหมือนไฟล์ CSS/รูปภาพทุกประการ (ใช้ pipeline เดียวกัน ไม่ใช่คนละระบบ)
3. **Gem อื่นเพิ่มโฟลเดอร์ของตัวเองเข้า load path ได้เอง** ผ่าน Railtie — นี่คือกลไกที่ทำให้
   `turbo.min.js`, `stimulus.min.js`, Trix editor's CSS/JS ฯลฯ ถูก serve ผ่าน Propshaft ได้
   ทันทีโดยเราไม่ต้อง `npm install` หรือ copy ไฟล์อะไรเองเลย — แค่มี gem ใน `Gemfile` ก็พอ

### เพิ่มโฟลเดอร์ของตัวเองเข้า load path (ถ้าต้องการ)

ถ้าต้องการเก็บ asset ไว้ในโฟลเดอร์อื่นนอกเหนือจากที่ Rails ให้มา (เช่น `app/assets/fonts`)
เพิ่มเข้า load path ได้ใน `config/initializers/assets.rb`:

```ruby
# config/initializers/assets.rb
Rails.application.config.assets.paths << Rails.root.join("app/assets/fonts")
```

### `app/assets/*` ต่างจาก `public/*` อย่างไร

จุดที่มือใหม่มักสับสน: ไฟล์ static ทุกไฟล์ไม่จำเป็นต้องผ่าน asset pipeline เสมอไป ลองดู
`public/` ของแอปที่ generate ใหม่:

```bash
ls public/
```

```
400.html  404.html  406-unsupported-browser.html  422.html  500.html
assets  icon.png  icon.svg  robots.txt
```

`icon.png`, `icon.svg`, `robots.txt`, และหน้า error (`404.html` ฯลฯ) **ถูกวางไว้ตรงๆ ใน
`public/` ไม่ผ่าน Propshaft เลย** — Rails (ผ่าน middleware `ActionDispatch::Static`) serve
ไฟล์พวกนี้ตรงๆ ตามชื่อจริง **ไม่มีการ fingerprint ไม่มีการเปลี่ยนชื่อไฟล์** เหตุผลที่เหมาะสม:

- `robots.txt`, `favicon.ico` ต้องมี**ชื่อไฟล์คงที่ตายตัว**ตาม convention ของเว็บ (browser/
  search engine bot จะไปเรียก `/robots.txt` ตรงๆ เสมอ เปลี่ยนชื่อไม่ได้)
- หน้า error (`500.html`) ต้อง serve ได้แม้ตอนที่ตัวแอป Rails เองพังจน render ผ่าน pipeline
  ไม่ได้แล้ว จึงต้องเป็นไฟล์ static ล้วนๆ ที่ไม่พึ่งพา asset pipeline เลย

**กฎการเลือกใช้ในทางปฏิบัติ:** ไฟล์ที่**เนื้อหาเปลี่ยนบ่อย**และอยากได้ cache-busting +
helper method (CSS, JS, รูปภาพในเนื้อหาเว็บ) → ใส่ใน `app/assets/` ไฟล์ที่**ต้องมีชื่อ/ตำแหน่ง
คงที่ตายตัว** (favicon, robots.txt, custom error page) → ใส่ใน `public/` ตรงๆ

---

## Step 284: กลไก Resolve และ Fingerprint เบื้องหลัง Cache-busting

มาดูกันจริงๆ ว่า Propshaft คำนวณ fingerprint ยังไง เริ่มจากสร้างไฟล์ CSS ทดลอง:

```bash
echo ".site-header { color: red; }" > app/assets/stylesheets/custom.css
```

เรียก resolver ผ่าน `rails console` เพื่อดู path ที่ resolve ได้จริง:

```ruby
Rails.application.assets.resolver.resolve("custom.css")
# => "/assets/custom-b52c70d8.css"

Rails.application.assets.load_path.find("custom.css").digested_path
# => #<Pathname:custom-b52c70d8.css>
```

ทดสอบ cache-busting จริง: แก้เนื้อหาไฟล์แล้วเช็ค digest ใหม่

```bash
echo ".site-header { color: blue; }" > app/assets/stylesheets/custom.css
```

```ruby
Rails.application.assets.load_path.find("custom.css").digested_path
# => #<Pathname:custom-9b5f244c.css>   ← digest เปลี่ยนจาก b52c70d8 ไปเป็น 9b5f244c
```

แค่เปลี่ยนคำว่า `red` เป็น `blue` คำเดียว digest ก็เปลี่ยนไปทันที — นี่คือกลไกที่ทำให้
cache-busting ทำงานได้แบบอัตโนมัติ 100% โดยไม่ต้องมีใครมาคอยเปลี่ยนเลขเวอร์ชันเอง

### Digest คำนวณจากอะไร

ดูจาก source code จริงของ `Propshaft::Asset#digest` (gem `propshaft`):

```ruby
def digest
  @digest ||= Digest::SHA1.hexdigest("#{content_with_compile_references}#{load_path.version}").first(8)
end
```

สรุปสั้นๆ: digest คือ **SHA1 hash ของเนื้อหาไฟล์ (บวกเนื้อหาไฟล์อื่นที่มันอ้างอิงถึง เช่น
`url()` ใน CSS ที่ชี้ไปยังไฟล์รูป บวก version ของ load path)** แล้วตัดมาแค่ **8 ตัวอักษรแรก**
— นี่คือเหตุผลที่เราเห็น hash 8 หลักต่อท้ายชื่อไฟล์เสมอ เช่น `application-8b441ae0.css`

### Response header จริงที่ browser ได้รับ

รัน dev server แล้ว `curl -I` ดู header ของไฟล์ที่ fingerprint แล้ว:

```bash
curl -I http://localhost:3000/assets/custom-b52c70d8.css
```

```
HTTP/1.1 200 OK
content-type: text/css; charset=utf-8
etag: "b52c70d8"
cache-control: public, max-age=31536000, immutable
content-length: 29
```

`cache-control: public, max-age=31536000, immutable` แปลว่า **"แคชไฟล์นี้ไว้ได้นานสุด 1 ปี
เต็ม (31,536,000 วินาที) และไม่ต้องเช็คซ้ำกับ server เลยระหว่างนั้น"** — นี่คือค่า cache ที่
aggressive ที่สุดที่ทำได้ ปลอดภัยเพราะถ้าเนื้อหาเปลี่ยน URL ก็เปลี่ยนตามไปด้วยเสมอ (ไม่มีทาง
ที่ URL เดิมจะชี้ไปยังเนื้อหาใหม่ได้)

### จะเกิดอะไรถ้าเดา digest ผิดหรือไฟล์ไม่มีอยู่จริง

```bash
curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/assets/custom-WRONGHASH.css
# => 404

curl -o /dev/null -w "%{http_code}\n" http://localhost:3000/assets/custom.css
# => 404  (ไม่มี digest ต่อท้ายก็หาไม่เจอเช่นกัน — Propshaft serve เฉพาะ digested path เท่านั้น)
```

และถ้าเรียกผ่าน helper อย่าง `image_tag`/`asset_path` กับไฟล์ที่ไม่มีอยู่จริงในทุก load path
เลย จะได้ exception ทันทีตอน render (ทดสอบจริง):

```
Propshaft::MissingAssetError (The asset 'does-not-exist.png' was not found in the load path.)
```

Error นี้มีประโยชน์มาก — มันทำให้เรารู้ทันทีตอน dev/test ว่า "ลืมใส่ไฟล์รูป" หรือ "พิมพ์ชื่อ
ไฟล์ผิด" แทนที่จะปล่อยให้ `<img>` broken เงียบๆ แล้วมารู้ทีหลังตอนขึ้น production

---

## Step 285: `stylesheet_link_tag`, `image_tag`, `asset_path` — Helper ที่ผูกกับ Propshaft

### `image_tag` — ทบทวนจาก Part 024 แล้วเจาะลึกเบื้องหลัง

```erb
<%= image_tag "logo.png", alt: "โลโก้บริษัท" %>
```

```html
<img alt="โลโก้บริษัท" src="/assets/logo-f0eaede3.png" />
```

`image_tag` เรียก `asset_path`/`compute_asset_path` ภายใน ซึ่งไปเรียก
`Rails.application.assets.resolver.resolve` ตัวเดียวกับที่เราทดลองใน Step 284 ตรงๆ — ถ้า
resolve ไม่เจอไฟล์ จะ raise `Propshaft::MissingAssetError` ทันที (ไม่ใช่ปล่อยผ่านเป็น src
เปล่าๆ)

### `stylesheet_link_tag` — จุดที่พิสูจน์ว่า Propshaft "ไม่ bundle"

นี่คือจุดสำคัญที่สุดที่แสดงความต่างระหว่าง Sprockets กับ Propshaft ชัดเจนที่สุด ลองมีไฟล์ CSS
2 ไฟล์ในโปรเจกต์:

```
app/assets/stylesheets/application.css
app/assets/stylesheets/custom.css
```

แล้วเรียกด้วย symbol พิเศษ `:app` (ค่า default ที่ `rails new` generate ไว้ให้ใน layout):

```erb
<%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
```

ผลลัพธ์ HTML ที่ render ออกมาจริง:

```html
<link rel="stylesheet" href="/assets/application-8b441ae0.css" data-turbo-track="reload" />
<link rel="stylesheet" href="/assets/custom-b52c70d8.css" data-turbo-track="reload" />
```

สังเกตว่าได้ **`<link>` สองแท็กแยกกัน คนละไฟล์ คนละ HTTP request** — ถ้าเป็น Sprockets ยุคเก่า
CSS ทั้งสองไฟล์นี้จะถูก**รวมเป็นไฟล์เดียว** (`application.css` ไฟล์เดียวที่มีเนื้อหาทั้งสอง
ไฟล์อยู่ข้างใน) ตาม directive `//= require_tree .` ที่เคยเขียนไว้ใน manifest — Propshaft
**ไม่ทำแบบนั้นเลย** แต่ละไฟล์ยังเป็นไฟล์แยกกันเสมอ ที่ทำได้แบบนี้เพราะ HTTP/2 ทำให้การยิงหลาย
request ไม่มีต้นทุนสูงเหมือนสมัยก่อน (ตามที่อธิบายเหตุผลไว้ใน Step 282)

`stylesheet_link_tag` รับ symbol พิเศษ 2 ตัว:

- `:app` — รวมเฉพาะไฟล์ CSS ที่อยู่ใต้ `app/assets/**/*.css` เท่านั้น (ไม่รวมของ gem อื่น)
- `:all` — รวมไฟล์ CSS **ทุกไฟล์ที่อยู่ใน load path ทั้งหมด** (รวมของ gem ด้วย เช่น
  `trix.css` จาก Action Text)

```erb
<%= stylesheet_link_tag :app %>   <%# เฉพาะ CSS ของแอปเรา %>
<%= stylesheet_link_tag :all %>   <%# ทุก CSS รวมของ gem %>
```

### Subresource Integrity (SRI) — option `integrity: true`

Propshaft เพิ่มความสามารถที่ Sprockets ไม่มีมาก่อน: สร้าง hash SRI ให้อัตโนมัติ (ใช้ยืนยันว่า
ไฟล์ที่โหลดมาไม่ถูกแก้ไขระหว่างทาง เช่นตอนโหลดผ่าน CDN ภายนอก):

```erb
<%= stylesheet_link_tag "application", integrity: true %>
<%# => <link rel="stylesheet" href="/assets/application-8b441ae0.css"
              integrity="sha256-xyz789..."> %>
```

SRI จะถูกคำนวณให้ก็ต่อเมื่อ request เป็น HTTPS หรือมาจาก localhost เท่านั้น (secure context) —
เหตุผลด้านความปลอดภัยเรื่องนี้จะเจาะลึกเพิ่มเติมใน Part 079

### `asset_path` — ดึง path ดิบๆ ไปใช้เอง

บางครั้งต้องการแค่ path ของ asset ไปใช้ในที่ที่ไม่ใช่ tag เช่นใน CSS แบบ inline หรือ JSON:

```erb
<%= asset_path("logo.png") %>
<%# => /assets/logo-f0eaede3.png %>

<div style="background-image: url(<%= asset_path('logo.png') %>)"></div>
```

---

## Step 286: `importmap-rails` — จัดการ JavaScript แบบไม่มี Node.js Build Step

### ปัญหาที่ JavaScript เคยมี และทำไมตอนนี้แก้ต่างไปได้

JavaScript สมัยก่อนไม่มีระบบ module มาตรฐานในตัว browser — ทุก `<script>` ที่โหลดเข้ามาจะ
แชร์ global scope เดียวกันหมด การจัดการ dependency ระหว่างไฟล์จึงยุ่งยากมาก (ต้องเรียง
`<script>` ให้ถูกลำดับเอง) นี่คือเหตุผลหลักที่วงการ JavaScript สร้าง bundler อย่าง Webpack
ขึ้นมา — ใช้ syntax `import`/`export` แบบสะดวก แล้วให้ bundler แปลงเป็นไฟล์เดียวที่ browser
เข้าใจได้ตอน build

**Import Maps** คือ feature ของ browser เอง (เป็นมาตรฐานเว็บ ไม่ใช่ของ Rails) ที่รองรับใน
browser สมัยใหม่ทุกตัวแล้ว ให้เราเขียน `<script type="importmap">` เป็น JSON บอก browser ว่า
ชื่อ package แต่ละตัว (bare specifier) ควรไปหาไฟล์จริงที่ URL ไหน จากนั้นใน
`<script type="module">` ก็เขียน `import "some-package"` ได้ตรงๆ โดย**ไม่ต้องมี bundler
แปลงให้ก่อนเลย** — browser resolve เองได้ทันที

**importmap-rails** คือ gem ที่ทำให้ Rails สร้าง `<script type="importmap">` นี้ให้อัตโนมัติ
จากไฟล์ config เดียว และประสานงานกับ Propshaft เพื่อ fingerprint ไฟล์ JS แต่ละไฟล์เหมือน CSS/
รูปภาพทุกประการ

### `config/importmap.rb`

ไฟล์นี้คือจุดศูนย์กลางที่ประกาศว่า "ชื่อ" (bare specifier) ไหนแมปไปที่ไฟล์จริงไฟล์ไหน สำหรับ
แอปที่ generate ด้วย Rails 8.1 (มี Turbo + Stimulus ติดมาด้วย):

```ruby
# config/importmap.rb
# Pin npm packages by running ./bin/importmap

pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

- `pin "application"` — ประกาศว่า `"application"` แมปไปที่ `app/javascript/application.js`
  (ชื่อ default ตรงกับชื่อไฟล์ ไม่ต้องระบุ `to:`)
- `pin "@hotwired/turbo-rails", to: "turbo.min.js"` — ประกาศว่าเมื่อโค้ดไหน
  `import "@hotwired/turbo-rails"` ให้ browser ไปโหลดไฟล์ `turbo.min.js` จริง (ไฟล์นี้ถูก
  vendor มาพร้อม gem `turbo-rails` แล้ว serve ผ่าน Propshaft load path — ดู Step 283)
- `pin_all_from "app/javascript/controllers", under: "controllers"` — pin ไฟล์ทุกไฟล์ใน
  โฟลเดอร์นั้นให้อัตโนมัติทีเดียว (สะดวกสำหรับ Stimulus controllers ที่เพิ่มเรื่อยๆ)

### ผลลัพธ์จริงที่ render ออกมาใน `<head>`

Layout ค่า default มี helper `javascript_importmap_tags` ซึ่ง render ออกมาเป็นชุด HTML แบบนี้
(ทดสอบจริงบน Rails 8.1.4):

```html
<script type="importmap" data-turbo-track="reload">{
  "imports": {
    "application": "/assets/application-bfcdf840.js",
    "@hotwired/turbo-rails": "/assets/turbo.min-9fd88cd5.js",
    "@hotwired/stimulus": "/assets/stimulus.min-4b1e420e.js",
    "@hotwired/stimulus-loading": "/assets/stimulus-loading-1fc53fe7.js",
    "controllers/application": "/assets/controllers/application-3affb389.js",
    "controllers/hello_controller": "/assets/controllers/hello_controller-708796bd.js",
    "controllers": "/assets/controllers/index-ee64e1f1.js"
  }
}</script>
<link rel="modulepreload" href="/assets/application-bfcdf840.js">
<link rel="modulepreload" href="/assets/turbo.min-9fd88cd5.js">
<link rel="modulepreload" href="/assets/stimulus.min-4b1e420e.js">
<link rel="modulepreload" href="/assets/stimulus-loading-1fc53fe7.js">
<link rel="modulepreload" href="/assets/controllers/application-3affb389.js">
<link rel="modulepreload" href="/assets/controllers/hello_controller-708796bd.js">
<link rel="modulepreload" href="/assets/controllers/index-ee64e1f1.js">
<script type="module">import "application"</script>
```

สังเกต 3 ส่วน:

1. **`<script type="importmap">`** — JSON map ชื่อ package → URL จริง (ผ่าน fingerprint ของ
   Propshaft แล้ว) นี่คือสิ่งที่ทำให้ `import "application"` ในโค้ดที่อื่นทำงานได้
2. **`<link rel="modulepreload">`** — สั่งให้ browser เริ่มโหลดไฟล์ JS ล่วงหน้าแบบขนาน ก่อน
   ที่ `import` จะถูกเรียกจริง (เพิ่มความเร็ว ลด waterfall ของการโหลดไฟล์ทีละต่อ)
3. **`<script type="module">import "application"</script>`** — จุดเริ่มต้นจริงที่สั่งให้
   browser รัน `app/javascript/application.js` ซึ่งข้างในมี `import "@hotwired/turbo-rails"`
   และ `import "controllers"` ต่อกันไปเป็นทอดๆ

**ทั้งหมดนี้เกิดขึ้นโดยไม่ต้องมี Node.js, ไม่ต้อง `npm install`, ไม่ต้องมีขั้นตอน build ใดๆ
เลย** — Rails เพียงแค่ประกาศ mapping แล้วปล่อยให้ฟีเจอร์ import map ของ browser ทำงานเอง
ทั้งหมด

### `bin/importmap pin` — เพิ่ม package จาก npm โดยไม่ต้องมี Node.js

เมื่อต้องการใช้ library จาก npm ตัวใดตัวหนึ่ง (เช่น `dayjs` สำหรับจัดการวันที่) ไม่ต้องไปหา
ไฟล์ CDN เอง ใช้คำสั่งนี้:

```bash
bin/importmap pin dayjs
```

เบื้องหลัง `importmap-rails` จะยิง request ไปที่บริการ **jspm.io** (`api.jspm.io/generate`)
เพื่อขอไฟล์ JavaScript ตัวจริงของ package นั้น (แปลงจาก npm module format ให้เป็น ES module
ที่ browser ใช้ได้ตรงๆ) แล้ว**ดาวน์โหลดไฟล์นั้นมาเก็บไว้ในเครื่องที่ `vendor/javascript/`**
พร้อมเติมบรรทัดต่อท้าย `config/importmap.rb` ให้อัตโนมัติ:

```ruby
pin "dayjs" # 1.11.13
```

> **ข้อควรรู้:** ขั้นตอนนี้ต้องมี**การเชื่อมต่ออินเทอร์เน็ต** เพื่อติดต่อ jspm.io ครั้งเดียว
> ตอนรันคำสั่ง `pin` (ในสภาพแวดล้อมที่ปิดกั้นการเชื่อมต่อออกภายนอก เช่น CI แบบ sandbox
> คำสั่งนี้จะ error) แต่**หลังจากดาวน์โหลดเสร็จแล้ว ไฟล์จะถูกเก็บไว้ใน `vendor/javascript/`
> เป็นไฟล์ปกติในโปรเจกต์** (commit เข้า git ได้เลย) การรันแอปหลังจากนั้นไม่ต้องพึ่งอินเทอร์เน็ต
> หรือ Node.js อีกเลย ต่างจาก workflow ของ `npm install` ที่ต้องดาวน์โหลด `node_modules`
> ใหม่ทุกครั้งที่ deploy หรือ clone โปรเจกต์ใหม่

คำสั่งอื่นที่เกี่ยวข้อง:

```bash
bin/importmap json      # แสดง import map ปัจจุบันเป็น JSON (เหมือนที่ฝังใน <head>)
bin/importmap unpin dayjs   # เอา pin ออก และลบไฟล์ที่ vendor ไว้
bin/importmap pristine  # ดาวน์โหลดทุก package ที่ pin ไว้ใหม่ทั้งหมด (เผื่อไฟล์เสียหาย/หาย)
```

### Pin ไฟล์ JavaScript ของตัวเองแบบไม่ต้องพึ่ง npm เลย

ไม่จำเป็นต้องมาจาก npm เสมอไป — ไฟล์ JS ที่เราเขียนเองใน `app/javascript/` ก็ pin ได้ตรงๆ
เพราะโฟลเดอร์นี้อยู่ใน load path ของ Propshaft อยู่แล้ว (ตามที่แสดงใน Step 283):

```ruby
# config/importmap.rb — เพิ่มเข้าไปเอง
pin "greeter"   # จะไปหา app/javascript/greeter.js ให้อัตโนมัติ
```

จะสาธิตแบบเต็มในแบบฝึกหัดท้าย Part นี้

---

## Step 287: เพิ่ม Stylesheet ของตัวเอง และอ้างอิงใช้งานจริง

มาลองเพิ่ม CSS ไฟล์ใหม่ในโปรเจกต์จริงทีละขั้นตอน สมมติกำลังสร้างหน้า Landing Page แล้วต้องการ
สไตล์เฉพาะของ header:

### ขั้นที่ 1: สร้างไฟล์ CSS ใหม่

```css
/* app/assets/stylesheets/site_header.css */
.site-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  background-color: #1a1a2e;
  color: #ffffff;
}

.site-header__logo {
  height: 40px;
}

.site-header__nav a {
  color: #eaeaea;
  margin-left: 20px;
  text-decoration: none;
  font-weight: 600;
}

.site-header__nav a:hover {
  color: #4ecdc4;
}
```

### ขั้นที่ 2: ตรวจสอบว่า Propshaft มองเห็นไฟล์นี้แล้ว

ไม่ต้องแก้ config ใดๆ เพิ่ม เพราะ `app/assets/stylesheets/` อยู่ใน load path เป็นค่า default
อยู่แล้ว (Step 283) ยืนยันได้ทันทีด้วย `rails console`:

```ruby
Rails.application.assets.resolver.resolve("site_header.css")
# => "/assets/site_header-<hash>.css"
```

### ขั้นที่ 3: อ้างอิงใช้งานในหน้าเว็บ

ถ้าใช้ทุกหน้า ให้รวมเข้ากับ `:app` ที่มีอยู่แล้วในเลย์เอาต์ (ไม่ต้องทำอะไรเพิ่ม เพราะ
`stylesheet_link_tag :app` โหลดทุกไฟล์ CSS ใต้ `app/assets/**/*.css` ให้อัตโนมัติอยู่แล้ว —
ยืนยันจาก Step 285 ที่ทดสอบว่ามันโหลดทั้ง `application.css` และ `custom.css` พร้อมกัน)

ถ้าต้องการโหลดเฉพาะบางหน้าเท่านั้น (ไม่ใช่ทุกหน้า) ให้เรียกแยกด้วยชื่อไฟล์ตรงๆ ผ่าน
`content_for(:head)` ตามที่สอนไว้ใน Part 024 Step 239:

```erb
<%# app/views/landing/index.html.erb %>
<% content_for(:head) do %>
  <%= stylesheet_link_tag "site_header" %>
<% end %>

<header class="site-header">
  <img src="<%= asset_path('logo.png') %>" alt="โลโก้" class="site-header__logo">
  <nav class="site-header__nav">
    <a href="#">หน้าแรก</a>
    <a href="#">เกี่ยวกับเรา</a>
  </nav>
</header>
```

### จัดกลุ่มไฟล์ CSS หลายไฟล์เป็นโฟลเดอร์ย่อยได้

Propshaft ไม่สนใจโครงสร้างโฟลเดอร์ย่อยเลย — ไฟล์ที่อยู่ลึกแค่ไหนก็ resolve ได้ด้วยชื่อไฟล์
(logical path จริงๆ คือ path สัมพัทธ์จาก asset root):

```
app/assets/stylesheets/
├── application.css
├── components/
│   ├── button.css
│   └── card.css
└── pages/
    └── landing.css
```

```erb
<%= stylesheet_link_tag "components/button" %>
<%= stylesheet_link_tag "pages/landing" %>
```

---

## Step 288: เพิ่มรูปภาพของตัวเอง และใช้งานผ่าน `image_tag`

### ขั้นที่ 1: วางไฟล์รูปในโฟลเดอร์ที่ถูกต้อง

```bash
cp ~/Downloads/company-logo.png app/assets/images/logo.png
```

Propshaft รองรับไฟล์รูปแบบมาตรฐานทุกชนิด (`.png`, `.jpg`, `.jpeg`, `.svg`, `.gif`, `.webp`
ฯลฯ) โดยไม่ต้องตั้งค่าอะไรเพิ่มเติม เพราะมัน serve ไฟล์ "ตามที่มันเป็น" ไม่มีการแปลง format
หรือบีบอัดใดๆ ในตัว (ต่างจาก Active Storage ที่มี variant/resize — เรื่องนั้นเป็นคนละระบบ จะ
สอนใน Part 066–067)

### ขั้นที่ 2: เรียกใช้ผ่าน `image_tag`

```erb
<%= image_tag "logo.png", alt: "โลโก้บริษัท" %>
```

```html
<img alt="โลโก้บริษัท" src="/assets/logo-f0eaede3.png" />
```

### Options ที่ใช้บ่อย

```erb
<%= image_tag "logo.png",
      alt: "โลโก้บริษัท",
      width: 120,
      height: 40,
      class: "site-header__logo",
      loading: "lazy" %>
```

```html
<img width="120" height="40" class="site-header__logo" loading="lazy"
     alt="โลโก้บริษัท" src="/assets/logo-f0eaede3.png" />
```

- **`alt:`** — ควรใส่เสมอ (accessibility และ SEO) — ถ้าไม่ใส่ Rails จะไม่ generate ให้เองและ
  `<img>` จะไม่มี `alt` attribute เลย (ผิด HTML best practice)
- **`width:`/`height:`** — ควรระบุให้ตรงกับขนาดจริงเสมอที่เป็นไปได้ ช่วยลด **Cumulative
  Layout Shift (CLS)** เพราะ browser จองพื้นที่ไว้ล่วงหน้าได้ทันทีโดยไม่ต้องรอโหลดรูปเสร็จก่อน
  ถึงจะรู้ขนาด
- **`loading: "lazy"`** — บอก browser ให้เลื่อนการโหลดรูปที่ยังไม่เข้า viewport ออกไปก่อน
  (ฟีเจอร์มาตรฐานของ browser ไม่เกี่ยวกับ Propshaft โดยตรง)

### รูปที่อยู่ในโฟลเดอร์ย่อย

```
app/assets/images/
├── logo.png
└── icons/
    └── cart.svg
```

```erb
<%= image_tag "icons/cart.svg", alt: "ตะกร้าสินค้า" %>
```

### รูปจาก URL ภายนอก (ไม่ผ่าน Propshaft)

ถ้า path ที่ส่งให้ `image_tag` ขึ้นต้นด้วย `http://`, `https://`, หรือ `//` Rails จะรู้ว่านี่
คือ URL ภายนอก **ไม่พยายาม resolve ผ่าน Propshaft เลย** ใช้ URL นั้นตรงๆ:

```erb
<%= image_tag "https://cdn.example.com/banner.jpg", alt: "แบนเนอร์โปรโมชัน" %>
<%# => <img alt="แบนเนอร์โปรโมชัน" src="https://cdn.example.com/banner.jpg" /> %>
```

---

## Step 289: `bin/rails assets:precompile` — เตรียม Asset สำหรับ Production

### ทำไม development ไม่ต้อง precompile แต่ production ต้อง

ใน**development** เมื่อ browser ขอไฟล์ asset Propshaft จะ resolve และคำนวณ digest **แบบสด
(on-the-fly)** ทุกครั้งที่มี request เข้ามา สะดวกมากตอนพัฒนา (แก้ CSS แล้ว refresh เห็นผลทันที
ไม่ต้องรันคำสั่งอะไรเพิ่ม) แต่วิธีนี้มีต้นทุนด้าน performance เล็กน้อยต่อ request (ต้อง scan
ไฟล์ คำนวณ hash) ซึ่งยอมรับได้ตอน dev คนเดียวใช้งาน แต่**ไม่เหมาะกับ production ที่มีผู้ใช้
จำนวนมากพร้อมกัน**

ใน**production** เราจึง **precompile** asset ทั้งหมดล่วงหน้าตอน deploy ครั้งเดียว ให้ได้ไฟล์
จริงที่มี digest ต่อท้ายชื่อวางอยู่ใน `public/assets/` แล้ว จากนั้น request ทุกครั้งก็แค่ serve
ไฟล์ static ตรงๆ ไม่ต้องคำนวณอะไรซ้ำอีกเลย (เร็วกว่ามาก และเซิร์ฟเวอร์เว็บอย่าง Nginx/CDN
serve ไฟล์ static ได้ตรงๆ โดยไม่ต้องผ่าน Ruby process ด้วยซ้ำ)

### รันคำสั่งจริง

```bash
RAILS_ENV=production SECRET_KEY_BASE=dummy bin/rails assets:precompile
```

(ในการ deploy จริงไม่ต้องใส่ `SECRET_KEY_BASE=dummy` เอง เพราะจะดึงจาก
`config/credentials.yml.enc` หรือ environment variable ที่ตั้งไว้แล้วบน server — ที่ใส่ตรงนี้
เพราะทดสอบ local โดยยังไม่ได้ตั้งค่า credentials ของ production)

ผลลัพธ์ log ที่เห็นจริง (ตัดบางส่วน):

```
Writing logo-f0eaede3.png
Writing custom-b52c70d8.css
Writing application-8b441ae0.css
Writing controllers/hello_controller-708796bd.js
Writing application-bfcdf840.js
Writing stimulus.min-4b1e420e.js
Writing turbo.min-9fd88cd5.js
...
```

สังเกตว่า Rails precompile **ไฟล์ asset ของ gem อื่นด้วย** (Turbo, Stimulus, Action Text/Trix,
Action Cable, Active Storage) เพราะไฟล์เหล่านั้นอยู่ใน load path เดียวกันทั้งหมด (ตามที่แสดงใน
Step 283) — production ต้องมีไฟล์ครบทุกไฟล์ที่แอปอาจจะเรียกใช้ ไม่ใช่แค่ไฟล์ที่เราเขียนเอง

### สิ่งที่เปลี่ยนไปหลัง precompile

```bash
find public/assets -maxdepth 1 | sort | head
```

```
public/assets/.manifest.json
public/assets/application-8b441ae0.css
public/assets/application-bfcdf840.js
public/assets/controllers
public/assets/custom-b52c70d8.css
public/assets/logo-f0eaede3.png
...
```

ไฟล์ **`.manifest.json`** (ซ่อนด้วย `.` นำหน้า) คือแผนที่จาก "ชื่อ logical" ไปยัง "ชื่อจริงที่
มี digest" ที่ Propshaft ใช้อ้างอิงตอนต้อง resolve โดยไม่ต้องอ่านไฟล์จริงซ้ำทุกครั้ง:

```json
{
  "logo.png": {"digested_path": "logo-f0eaede3.png", "integrity": null},
  "custom.css": {"digested_path": "custom-b52c70d8.css", "integrity": null},
  "application.css": {"digested_path": "application-8b441ae0.css", "integrity": null}
}
```

> **เทียบกับ Sprockets:** Sprockets เก็บ manifest ไว้เป็นไฟล์ชื่อ
> `manifest-<random-hash>.json` (ชื่อไฟล์ manifest เองก็มี hash แปลกไปทุกครั้งที่ precompile)
> ส่วน Propshaft ใช้ชื่อคงที่ `.manifest.json` เสมอ — รายละเอียดเล็กน้อยแต่สะท้อนปรัชญา "ทำให้
> เรียบง่ายและคาดเดาได้" ของ Propshaft

### Cache header ใน production

```ruby
# config/environments/production.rb
config.public_file_server.headers = { "cache-control" => "public, max-age=#{1.year.to_i}" }
```

นี่คือบรรทัดที่ Rails 8.1 generate ให้อัตโนมัติใน `production.rb` — บอก web server ให้ใส่
cache header อายุ 1 ปีกับไฟล์ static ทุกไฟล์ใน `public/` (รวมถึง `public/assets/` ที่เพิ่ง
precompile มา) ทำงานร่วมกับ fingerprinting ได้พอดี ตามหลักการที่อธิบายไว้ใน Step 284

### เมื่อไหร่ต้องรันคำสั่งนี้เอง

ในทางปฏิบัติ เราแทบไม่ต้องรัน `assets:precompile` มือเองเลย เพราะ:

- เครื่องมือ deploy มาตรฐานของ Rails 8 อย่าง **Kamal** (Part 076) จะรันขั้นตอนนี้ให้อัตโนมัติ
  ตอน build Docker image
- Platform อย่าง Render/Heroku (Part 077) ก็รันให้อัตโนมัติเป็นส่วนหนึ่งของ build step

รู้ไว้เพื่อ debug เวลาเจอปัญหา "asset หายตอน production" เท่านั้น — เช่นถ้าลืม precompile
ก่อน deploy แล้ว `config.assets.compile = false` (ค่า default ใน production) จะทำให้ Rails
**ไม่ยอม compile แบบสดให้เหมือน dev** และ raise error ทันทีเมื่อหา digested file ไม่เจอ

---

## Step 290: เมื่อไหร่ควรมองหา esbuild/Vite/Node Bundling แทน importmap

importmap-rails เหมาะมากสำหรับเว็บแอปสไตล์ Rails ทั่วไปที่ใช้ **Turbo + Stimulus** เป็นหลัก —
JavaScript ส่วนใหญ่เป็นชิ้นเล็กๆ ที่เพิ่ม interactivity ให้หน้าเว็บ ไม่ใช่แอปเดี่ยวขนาดใหญ่ฝั่ง
client แต่ก็มีสถานการณ์ที่ทีมจริงยังเลือกใช้ Node.js bundler แทน:

1. **แอปที่มีส่วน frontend เป็น Single Page Application (SPA) เต็มรูปแบบ** เช่นฝัง React/Vue
   dashboard ที่ซับซ้อนไว้ในหน้าเดียว การจัดการ state, routing ฝั่ง client, JSX/TSX ต้องพึ่ง
   ecosystem ของ Node.js (Babel/TypeScript compiler) ที่ import maps เพียวๆ ทำไม่ได้
2. **ต้องพึ่ง npm package ที่ซับซ้อนมาก มี dependency tree ลึก** — บาง package ออกแบบมาให้
   bundler resolve dependency ให้เอง (เช่น package ที่ import กันเองเป็นสิบๆ ชั้น) การ pin
   ทีละตัวด้วยมือผ่าน importmap เริ่มไม่คุ้มค่าเมื่อจำนวน dependency เยอะมากๆ
3. **ทีมมี pipeline frontend อยู่แล้ว** ที่ใช้ TypeScript, JSX, หรือ CSS preprocessor ขั้นสูง
   (PostCSS plugin เฉพาะทาง) ซึ่งต้องมีขั้นตอน compile ก่อนเสมออยู่แล้วไม่ว่าจะใช้ Rails หรือ
   ไม่

สำหรับกรณีเหล่านี้ `rails new` มี flag ให้เลือก JavaScript approach อื่นได้ตั้งแต่สร้างโปรเจกต์
(ตรวจสอบจริงจาก `rails new --help` บน Rails 8.1.4):

```
-j, --js, [--javascript=JAVASCRIPT]  # Choose JavaScript approach
                                      # Default: importmap
                                      # Possible values: importmap, bun, webpack, esbuild, rollup
```

```bash
rails new myapp --javascript=esbuild   # ใช้ esbuild เป็น bundler (เร็ว เขียนด้วย Go)
rails new myapp --javascript=bun       # ใช้ bun เป็น runtime + bundler (เร็วมาก ตัวเดียวจบ)
rails new myapp --javascript=webpack   # Webpack แบบดั้งเดิม (นิยมน้อยลงในโปรเจกต์ใหม่)
```

เมื่อเลือกตัวเลือกเหล่านี้ Rails จะติดตั้ง gem อย่าง `jsbundling-rails` แทน `importmap-rails`
ซึ่งจะสร้าง `package.json`, ต้องมี Node.js ติดตั้งในเครื่อง/production server, และมีขั้นตอน
build (`yarn build`/`npm run build`) ก่อน asset จะพร้อมใช้งาน — ไฟล์ที่ build เสร็จแล้วจะถูก
ส่งต่อให้ **Propshaft** serve เหมือนเดิม (Propshaft ยังทำหน้าที่ fingerprint + serve เป็น asset
pipeline อยู่ดี เพียงแต่ไฟล์ JS ตั้งต้นมาจาก build step ของ bundler แทนที่จะเป็นไฟล์ดิบๆ)

ในทำนองเดียวกัน ฝั่ง CSS ถ้าต้องการ Tailwind CSS หรือ Sass เต็มรูปแบบ มี flag
`--css=tailwind` หรือใช้ gem `cssbundling-rails`/`tailwindcss-rails` เพิ่มเข้ามาต่างหาก
(รายละเอียดเรื่อง Tailwind ใน Rails จะสอนเต็มรูปแบบใน **Part 054**)

> **คำแนะนำเชิงปฏิบัติ:** เริ่มต้นด้วย importmap เสมอสำหรับโปรเจกต์ใหม่ (เป็นค่า default ของ
> Rails 8 ด้วยเหตุผล) แล้วค่อยย้ายไป esbuild/Vite เฉพาะตอนที่เจอความจำเป็นจริงๆ (npm package
> ที่ import maps จัดการไม่ไหว, ต้องมี TypeScript/JSX) การเริ่มจากง่ายไปยากง่ายกว่าการเริ่ม
> จากซับซ้อนเกินจำเป็นแล้วต้องมาตัดออกทีหลังเสมอ

---

## แบบฝึกหัด: หน้าแรกที่มี Custom Stylesheet, โลโก้ และ JS Behavior ผ่าน Importmap

### โจทย์

สร้างหน้าแรก (`root_path`) ของแอปที่แสดงผลครบวงจรตามที่เรียนมาใน Part นี้:

1. เพิ่ม stylesheet ของตัวเอง (`site.css`) ที่มีสไตล์จริงอย่างน้อย 3 selector แล้วให้โหลด
   ผ่าน `:app` อัตโนมัติ
2. เพิ่มรูปโลโก้ (`logo.png`) แล้วแสดงผลผ่าน `image_tag` พร้อม `alt`, `width`, `height`
3. เขียนไฟล์ JavaScript เล็กๆ ของตัวเอง (ไม่ใช่ Stimulus controller) แล้ว pin ผ่าน
   `config/importmap.rb` จากนั้นเรียกใช้งานผ่าน `<script type="module">` ธรรมดาที่ import
   ชื่อที่ pin ไว้ตรงๆ
4. ยืนยันผลลัพธ์ด้วย `curl` ว่า HTML ที่ได้มี fingerprinted path ถูกต้องครบทุกจุด

### เฉลย

**1) Stylesheet ของตัวเอง:**

```css
/* app/assets/stylesheets/site.css */
body {
  font-family: -apple-system, "Segoe UI", sans-serif;
  margin: 0;
  background-color: #f4f4f9;
  color: #222;
}

.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 60px 20px;
  text-align: center;
}

.hero__logo {
  border-radius: 8px;
  margin-bottom: 16px;
}

.hero__greeting {
  font-size: 1.5rem;
  font-weight: 700;
  color: #1a1a2e;
}
```

ไม่ต้องแก้ layout เพิ่มเติม เพราะ `app/views/layouts/application.html.erb` ที่ Rails generate
ให้มีอยู่แล้วเรียก `stylesheet_link_tag :app` ซึ่งจะดึงไฟล์นี้เข้ามารวมกับ `application.css`
โดยอัตโนมัติ (คนละ `<link>` กันตามที่พิสูจน์แล้วใน Step 285)

**2) รูปโลโก้:** วางไฟล์ `app/assets/images/logo.png` (รูปอะไรก็ได้ ขนาดแนะนำ ~200×200px)

**3) JavaScript เล็กๆ ที่ pin เอง (ไม่ใช้ Stimulus):**

```javascript
// app/javascript/greeter.js
export function greet(name) {
  const hour = new Date().getHours();
  const period = hour < 12 ? "อรุณสวัสดิ์" : hour < 18 ? "สวัสดีตอนบ่าย" : "สวัสดีตอนเย็น";
  return `${period}, ${name}!`;
}
```

```ruby
# config/importmap.rb — เพิ่มบรรทัดนี้ต่อท้ายไฟล์เดิม
pin "greeter"
```

`app/javascript/` อยู่ใน load path ของ Propshaft อยู่แล้ว (Step 283) จึงไม่ต้องตั้งค่าอะไร
เพิ่มนอกจากเพิ่มบรรทัด `pin` เข้าไป

**4) หน้า view ที่ประกอบทุกอย่างเข้าด้วยกัน:**

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "pages#home"
end
```

```ruby
# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  def home
  end
end
```

```erb
<%# app/views/pages/home.html.erb %>
<% content_for(:title, "หน้าแรก — Asset Pipeline Demo") %>

<section class="hero">
  <%= image_tag "logo.png", alt: "โลโก้บริษัท", width: 120, height: 120, class: "hero__logo" %>
  <p id="greeting" class="hero__greeting">กำลังโหลด...</p>
</section>

<script type="module">
  import { greet } from "greeter"

  document.getElementById("greeting").textContent = greet("ผู้เยี่ยมชม")
</script>
```

สังเกตว่า `<script type="module">` ตัวนี้**ไม่ผ่าน Stimulus เลย** — เป็น native ES module
ธรรมดาที่ `import` bare specifier ชื่อ `"greeter"` ตรงๆ ซึ่งใช้งานได้เพราะ browser หาคำตอบจาก
`<script type="importmap">` ที่ Rails generate ไว้ให้ใน `<head>` โดยอัตโนมัติ

### ตรวจสอบผลลัพธ์จริงด้วย `curl`

```bash
bin/rails server -p 3000 &
curl -s http://localhost:3000/ | grep -E "site-?css|logo-|greeter-|importmap"
```

ผลลัพธ์ที่ควรได้ (hash จริงจะต่างกันไปตามเนื้อหาไฟล์ในเครื่องแต่ละคน):

```html
<link rel="stylesheet" href="/assets/application-8b441ae0.css" data-turbo-track="reload" />
<link rel="stylesheet" href="/assets/site-<hash>.css" data-turbo-track="reload" />
<script type="importmap" data-turbo-track="reload">{
  "imports": {
    "application": "/assets/application-bfcdf840.js",
    ...
    "greeter": "/assets/greeter-<hash>.js"
  }
}</script>
<img alt="โลโก้บริษัท" width="120" height="120" class="hero__logo"
     src="/assets/logo-<hash>.png" />
```

ยืนยันจุดสำคัญ 3 อย่างที่เรียนมาทั้ง Part:

1. `site.css` ถูกโหลดเป็น `<link>` แยกต่างหากจาก `application.css` (ไม่ bundle รวมกัน)
2. `logo.png` ได้ path แบบ fingerprint (`logo-<hash>.png`) จาก `image_tag` โดยอัตโนมัติ
3. `"greeter"` ปรากฏใน import map JSON ชี้ไปยังไฟล์ที่ fingerprint แล้วเช่นกัน และหน้าเว็บที่
   เปิดจริงใน browser จะเห็นข้อความทักทายที่เปลี่ยนตามเวลาปัจจุบัน (พิสูจน์ว่า
   `import { greet } from "greeter"` ทำงานได้จริงโดยไม่มี build step ใดๆ)

ลองแก้เนื้อหา `site.css` หรือ `greeter.js` แล้วรัน `curl` ซ้ำ — จะเห็น hash ต่อท้ายไฟล์ที่แก้
เปลี่ยนไปทันที ส่วนไฟล์อื่นที่ไม่ได้แตะเลข hash จะเหมือนเดิม (ยืนยันหลักการ cache-busting ที่
ทำงานแยกกันเป็นไฟล์ต่อไฟล์ ตามที่อธิบายไว้ใน Step 284)

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม `stylesheet_link_tag "site", integrity: true` แล้วเปิด developer tools ของ browser
   ดู response header/attribute `integrity` ที่เกิดขึ้นจริง ลองเปลี่ยนเนื้อหาไฟล์ CSS แล้ว
   สังเกตว่า hash ของ `integrity` เปลี่ยนตามไปด้วยหรือไม่ (ใบ้: มันคำนวณจาก `compiled_content`
   เดียวกับ digest)
2. ลองสร้างไฟล์ `app/assets/images/banner.svg` (SVG ธรรมดาก็ได้ วาดสี่เหลี่ยมกับข้อความ)
   แล้วใช้ `image_tag "banner.svg"` เทียบกับการฝัง SVG ตรงๆ ด้วย
   `<%= inline_svg... %>` (ถ้ามีเวลาลองค้น gem `inline_svg` เพิ่มเติม) — พิจารณาว่ากรณีไหนควร
   ใช้ path แบบ fingerprint กรณีไหนควรฝัง SVG ตรงๆ ในหน้า HTML (ใบ้: SVG ที่ต้องการเปลี่ยนสี
   ด้วย CSS `fill`/`stroke` มักต้องฝังตรงๆ ในหน้า)
3. ลองรัน `bin/rails assets:precompile` ในเครื่อง แล้วลบไฟล์ `public/assets/` ทั้งหมดออก
   จากนั้นลองรัน server ใน `RAILS_ENV=production` โดยไม่ precompile ใหม่ — สังเกต error ที่
   เกิดขึ้น แล้วอธิบายว่าทำไม production ถึงไม่ยอม fallback ไปคำนวณ asset แบบสดเหมือน
   development (ใบ้: ดูค่า `config.assets.compile` ใน `config/environments/production.rb`)
4. (ท้าทายขึ้น) ลองใช้ `bin/importmap pin` ดึง npm package เล็กๆ ตัวหนึ่งมาจริง (ต้องมี
   อินเทอร์เน็ต) แล้วดูว่าไฟล์ที่ถูกดาวน์โหลดมาอยู่ใน `vendor/javascript/` หน้าตาเป็นอย่างไร
   เทียบกับไฟล์ต้นฉบับบน npm — สังเกตว่ามันถูกแปลงเป็น ES module format แล้วหรือยัง

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Asset Pipeline** มีไว้แก้ปัญหา cache-busting (ผ่าน fingerprinting) และการจัด
  ระเบียบไฟล์ static (CSS/JS/รูปภาพ/ฟอนต์) โดยไม่ต้อง hardcode path เอง
- เข้าใจประวัติและเหตุผลเชิงเทคนิคที่ Rails 8 เปลี่ยนจาก **Sprockets** (ทำ bundling,
  minification, Sass compilation ในตัว) มาเป็น **Propshaft** (ทำแค่ resolve + fingerprint +
  serve) เพราะ HTTP/2 และ native ES Modules ทำให้เหตุผลเดิมของการ bundle อ่อนลงมาก
- รู้จักโครงสร้าง `app/assets/` และแนวคิด **load path** ของ Propshaft ที่ไม่ต้องมี
  `manifest.js` อีกต่อไป พร้อมเข้าใจว่า `app/javascript/`, `vendor/javascript/`, และโฟลเดอร์
  asset ของ gem อื่นก็อยู่ใน load path เดียวกันหมด
- เข้าใจกลไก **fingerprint** เบื้องหลัง: digest คำนวณจาก SHA1 ของเนื้อหาไฟล์ (8 ตัวอักษรแรก)
  ทำให้เปลี่ยนเนื้อหาแล้ว URL เปลี่ยนอัตโนมัติ รองรับ `cache-control: immutable` ได้อย่าง
  ปลอดภัย และรู้จัก error `Propshaft::MissingAssetError` เมื่อหาไฟล์ไม่เจอ
- ใช้ `stylesheet_link_tag` (`:app`/`:all`/`integrity:`), `image_tag`, `asset_path` ได้อย่าง
  เข้าใจเบื้องหลัง และพิสูจน์ได้ด้วยตัวเองว่า Propshaft **ไม่ bundle** ไฟล์ CSS/JS หลายไฟล์
  เป็นไฟล์เดียวเหมือน Sprockets
- เข้าใจว่า **importmap-rails** ใช้ฟีเจอร์ import maps ของ browser เองในการจัดการ JavaScript
  แบบไม่ต้องมี Node.js build step, รู้จัก `config/importmap.rb`, `pin`, `pin_all_from`, และ
  คำสั่ง `bin/importmap pin` สำหรับดึง package จาก npm ผ่าน jspm.io มาเก็บไว้ใน
  `vendor/javascript/`
- เพิ่ม custom stylesheet และรูปภาพของตัวเองเข้าโปรเจกต์จริงได้ พร้อม pin ไฟล์ JavaScript
  ของตัวเองผ่าน importmap และเรียกใช้จาก `<script type="module">` ธรรมดาโดยไม่ต้องพึ่ง
  Stimulus
- รู้ว่า `bin/rails assets:precompile` ทำอะไรต่างจาก development (เขียนไฟล์ fingerprint จริง
  ลง `public/assets/` พร้อม `.manifest.json`) และทำไม production ถึงไม่ compile แบบสด
- รู้ขอบเขตของ importmap: เหมาะกับแอปสไตล์ Turbo+Stimulus ทั่วไป แต่ถ้าต้องมี SPA ซับซ้อน
  หรือพึ่ง npm ecosystem หนักๆ ก็ยังเลือก `--javascript=esbuild`/`--javascript=bun` ตอน
  `rails new` ได้ โดย Propshaft ยังคงทำหน้าที่ serve ไฟล์ที่ build เสร็จแล้วเหมือนเดิม

**ต่อไป (Part 030):** เราจะเอาทุกอย่างที่เรียนมาตลอดเฟส 3 — Routing, Controller, View,
Model/ActiveRecord เบื้องต้น, Validation, Association, Scaffold, และ Asset Pipeline — มา
ประกอบร่างเป็น**โปรเจกต์รวบยอดของเฟส 3: ระบบ Blog แบบง่าย** ที่มี `Post` และ `Comment` พร้อม
CRUD ครบวงจรจริงทั้งระบบ (ไม่ใช่ข้อมูลจำลองแบบ `Struct` เหมือนแบบฝึกหัดก่อนหน้านี้อีกต่อไป)
ตั้งแต่ migration, routes ที่ nested กันระหว่าง Post/Comment, ไปจนถึงหน้าเว็บที่ใช้ partial,
helper, และ stylesheet ของตัวเองที่เรียนมาใน Part นี้ประกอบกันเป็นเว็บบล็อกที่ใช้งานได้จริง