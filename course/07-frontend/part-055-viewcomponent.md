# Part 055: ViewComponent — สร้าง UI แบบ Component-Based ใน Rails (ปิด Phase 7: Frontend/Hotwire)

> **Step ครอบคลุมใน Part นี้:** Step 541–550
> **ระดับ:** ปานกลาง–สูง (ต้องผ่าน Part 024 เรื่อง View/Partial/Helper, Part 046 RSpec สำหรับ
> Rails, และ Part 053 Stimulus มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.6 / Rails 8.1.4, gem `view_component` 4.15.0, `rspec-rails` 8.0.4,
> `capybara` 3.40.0 — ทุกตัวอย่างในไฟล์นี้ทดสอบรันจริงในแอป Rails ทดลองที่สร้างด้วย `rails new`
> พร้อม Propshaft + importmap-rails + Turbo + Stimulus + Tailwind (สภาพแวดล้อมเดียวกับที่ตั้งค่า
> ไว้ใน Part 021/029/053/054) แล้วลบทิ้งหลังตรวจสอบเสร็จ

Part นี้คือ **Part สุดท้ายของ Phase 7: Frontend / Hotwire** เราเดินทางมาจาก Turbo Drive/Frames
(Part 051), Turbo Streams (Part 052), Stimulus (Part 053), จนถึง Tailwind/Bootstrap (Part 054)
— ครบเครื่องเรื่องการอัปเดตหน้าเว็บแบบไม่ต้องเขียน JavaScript เยอะ และการจัดสไตล์ให้สวยงามแล้ว
สิ่งที่ยังขาดอยู่คือ **วิธีจัดการกับ UI ที่ซับซ้อนขึ้นเรื่อยๆ** — ปุ่ม, การ์ด, badge, modal,
dropdown ที่ต้องใช้ซ้ำหลายสิบจุดทั่วทั้งแอป Part 024 สอนเรื่อง partial ไปแล้วว่าเป็นเครื่องมือ
พื้นฐานสำหรับ reuse view code แต่เมื่อแอปโตขึ้นถึงจุดหนึ่ง partial จะเริ่มมีปัญหาเชิงโครงสร้างที่
แก้ไม่ได้ด้วยตัวมันเอง — Part นี้จะพาไปรู้จัก **ViewComponent** ไลบรารีที่ GitHub สร้างขึ้นมาเพื่อ
แก้ปัญหานั้นโดยเฉพาะ และปิดท้าย Phase 7 ด้วยภาพรวมของเครื่องมือฝั่ง frontend ทั้งหมดที่ Rails
สมัยใหม่มีให้

## สารบัญของ Part นี้

- Step 541: ปัญหาของ Partial เมื่อโปรเจกต์ใหญ่ขึ้น — Implicit Interface
- Step 542: ติดตั้ง ViewComponent — `gem "view_component"`
- Step 543: Generate component ตัวแรก — กายวิภาคของไฟล์คู่ `.rb` + `.html.erb`
- Step 544: Render component จาก View — `render(Component.new(...))`
- Step 545: Slots — `renders_one`/`renders_many` แทนที่ boolean flag ที่ล้นมือ
- Step 546: Component-local CSS/JS — colocate สไตล์และสคริปต์ไว้กับ component
- Step 547: Preview component แบบแยกโดด — `ViewComponent::Preview` และหน้า `/rails/view_components`
- Step 548: Unit test component ด้วย `render_inline` (RSpec)
- Step 549: Composition — ประกอบ component เข้าด้วยกัน (`CardComponent` ใช้ `ButtonComponent`)
- Step 550: กรอบการตัดสินใจ — Partial vs Helper vs ViewComponent vs Stimulus Controller
- แบบฝึกหัดปิด Part: การ์ดสินค้าที่ประกอบด้วย Header Slot + ButtonComponent พร้อม test ครบชุด
- สรุปสิ่งที่ได้เรียนรู้ + สรุปภาพรวม Phase 7 + เปิดตัว Phase 8

---

## Step 541: ปัญหาของ Partial เมื่อโปรเจกต์ใหญ่ขึ้น — Implicit Interface

ทบทวนจาก Part 024: **partial** คือไฟล์ view ย่อยที่ใช้ `render "ชื่อ", local: value` เรียกใช้ซ้ำได้
จากหลายที่ ตรงตามหลัก DRY — สำหรับโปรเจกต์เล็กถึงกลาง partial เพียงพอและเหมาะสมที่สุดแล้ว แต่เมื่อ
แอปมีขนาดใหญ่ขึ้น (หลายสิบ controller, หลายร้อย view, ทีมพัฒนาหลายคน) partial เริ่มมีจุดอ่อนเชิง
โครงสร้าง 4 ข้อที่แก้ด้วยการเขียน partial ให้ดีขึ้นอย่างเดียวไม่ได้:

### ปัญหาที่ 1: Implicit Interface — เปิดไฟล์แล้วไม่รู้ว่าต้องส่งอะไรมาบ้าง

```erb
<%# app/views/shared/_product_card.html.erb %>
<div class="card">
  <h3><%= product.name %></h3>
  <p><%= number_to_currency(product.price) %></p>
  <% if product.on_sale? %>
    <span class="badge badge-danger">ลดราคา!</span>
  <% end %>
  <% if show_add_to_cart %>
    <%= button_to "เพิ่มลงตะกร้า", cart_items_path(product_id: product.id) %>
  <% end %>
</div>
```

เปิดไฟล์นี้ขึ้นมาอย่างเดียว **ไม่มีทางรู้ได้เลยว่า** partial ตัวนี้ต้องการ local variable อะไรบ้าง
จนกว่าจะอ่านโค้ดทั้งไฟล์จบ (`product` ต้องมี `.name`, `.price`, `.on_sale?`, และยังมี
`show_add_to_cart` ที่แอบใช้อยู่กลางไฟล์โดยไม่มีการประกาศไว้ตรงไหนเลยว่าเป็น parameter ที่จำเป็น)
ถ้ามีคนเรียก `render "shared/product_card", product: product` โดยลืมส่ง `show_add_to_cart` มา จะ
ได้ `NameError: undefined local variable or method 'show_add_to_cart'` ตอน**รันจริง** ไม่ใช่ตอน
เขียนโค้ด — เทียบกับการเรียก method ปกติที่ Ruby บอก argument ที่ขาดหายไปตั้งแต่ตอนเขียนเลย
(ทบทวน Part 007 เรื่อง method signature)

> **ย้อนกลับไปดู Part 024 Step 234:** partial ที่ดีควรรับทุกอย่างผ่าน `locals:` ไม่พึ่ง instance
> variable ตรงๆ — คำแนะนำนั้นยังคงถูกต้องและจำเป็นเสมอ แต่ถึงจะทำตามนั้นครบถ้วน **`locals:` ก็ยัง
> เป็นเพียง Hash ที่ไม่มี schema บังคับ** ไม่มีอะไรห้ามไม่ให้คนเรียก partial ผิดรูปแบบ ไม่มีอะไร
> เตือนล่วงหน้าว่าต้องส่งอะไรมาบ้าง — ปัญหานี้อยู่คนละระดับกับคำแนะนำเรื่อง `locals:` ใน Part 024

### ปัญหาที่ 2: ทดสอบแยกจากกันไม่ได้ง่ายๆ

การจะทดสอบ `_product_card.html.erb` ตัวเดียวโดยไม่พึ่งอะไรอื่นเลย ต้องผ่านหนึ่งในสองทาง: เขียน
system test เปิด browser จริงด้วย Capybara (ทบทวน Part 048 — ช้า ต้อง boot ทั้งแอป) หรือเขียน
request spec ยิง controller ทั้งตัวแล้วไปตรวจ HTML ที่ได้ (ทบทวน Part 046 — ก็ยังต้องผ่าน routing,
controller action, authorization ทั้งชุดกว่าจะถึง partial ตัวที่อยากทดสอบจริงๆ) **ไม่มีทางเขียน
unit test ที่ตรงไปตรง "แค่ partial ตัวนี้" ได้เลย** เพราะ partial ไม่ใช่ object ที่ instantiate
แล้วเรียก method ได้ตรงๆ แบบที่ Part 018/019 สอนเรื่อง unit testing ไว้

### ปัญหาที่ 3: ไม่มีที่ทางให้ colocate CSS/JS เฉพาะของตัวเอง

ถ้า `_product_card.html.erb` ต้องการ CSS เฉพาะตัว (เช่น layout การ์ดแบบพิเศษ) หรือ Stimulus
controller เฉพาะตัว (เช่น toggle รายละเอียดสินค้า) ไฟล์เหล่านั้นต้องแยกไปอยู่คนละที่เสมอ —
`app/assets/stylesheets/product_card.css` กับ `app/javascript/controllers/product_card_controller.js`
— ยิ่งแอปมี partial เป็นร้อยตัว การไล่จับคู่ว่า CSS/JS ไฟล์ไหนเป็นของ partial ไหนก็ยิ่งเป็นภาระ
ทางความจำที่เพิ่มขึ้นเรื่อยๆ ไม่มีกลไกใดบังคับหรือช่วยให้ไฟล์ที่เกี่ยวข้องกันอยู่ใกล้กัน

### ปัญหาที่ 4: Boolean Flag สะสมจนควบคุมไม่ได้

เมื่อ partial ตัวเดียวต้องรองรับหลาย "โหมด" การแสดงผล (มี badge/ไม่มี, มีปุ่ม/ไม่มี, ปุ่มกี่ปุ่ม,
เรียงแนวตั้ง/แนวนอน) วิธีที่ทำกันบ่อยที่สุดคือเพิ่ม boolean parameter ไปเรื่อยๆ:

```erb
<%= render "shared/product_card",
      product: product,
      show_badge: true,
      show_add_to_cart: true,
      show_wishlist_button: false,
      compact_mode: false,
      show_price: true %>
```

ยิ่งเพิ่มมากเท่าไหร่ ยิ่งต้องเดามากขึ้นว่า flag ไหนขึ้นกับ flag ไหน (เช่น `compact_mode: true`
แล้ว `show_add_to_cart: true` พร้อมกันได้ไหม) — ปัญหานี้จะกลับมาอีกครั้งแบบเห็นภาพชัดเจนใน
Step 545 พร้อมทางแก้

### สิ่งที่ ViewComponent เสนอเป็นทางแก้

**ViewComponent** (พัฒนาโดยทีม GitHub เปิดเป็น open source ตั้งแต่ปี 2019 ปัจจุบันเป็นไลบรารี
component-based UI ที่ได้รับความนิยมสูงสุดในระบบนิเวศ Rails) แก้ปัญหาทั้ง 4 ข้อด้วยแนวคิดเดียว
ที่เรียบง่ายมาก: **ทำให้ view fragment เป็น Ruby class ธรรมดาที่มี `initialize` เป็นสัญญา
(explicit interface) แทนที่จะเป็นไฟล์ template ที่รับตัวแปรมาแบบไม่มีการควบคุม**

| ปัญหาของ Partial | ทางแก้ของ ViewComponent |
|---|---|
| Implicit interface — ไม่รู้ว่าต้องส่งอะไรมา | `initialize(title:, variant: :primary)` เป็น method signature จริง เห็นชัดจากหัวไฟล์ |
| ทดสอบแยกยาก ต้องพึ่ง request/system test | เป็น Ruby object → unit test ได้ตรงๆ ด้วย `render_inline` (Step 548) |
| ไม่มีที่ทาง colocate CSS/JS | ไฟล์ `.rb`, `.html.erb`, `.css`, `_controller.js` อยู่โฟลเดอร์เดียวกัน (Step 546) |
| Boolean flag ล้นมือ | Slot API (`renders_one`/`renders_many`) แทนที่ flag ด้วยเนื้อหาที่ยืดหยุ่นกว่า (Step 545) |

ข้อสำคัญที่ต้องเข้าใจตั้งแต่ต้น: **ViewComponent ไม่ได้มาแทนที่ partial ทั้งหมด** — partial ยังคง
เป็นเครื่องมือที่เหมาะสมที่สุดสำหรับ HTML ชิ้นเล็กๆ ที่ไม่มี logic ซับซ้อนและไม่ต้อง reuse ข้าม
โปรเจกต์ (Step 550 จะมีกรอบการตัดสินใจแบบละเอียดว่าเมื่อไหร่ควรใช้ตัวไหน)

---

## Step 542: ติดตั้ง ViewComponent — `gem "view_component"`

เพิ่ม gem ลงใน `Gemfile`:

```ruby
# Gemfile
gem "view_component"
```

```bash
bundle install
```

ทดสอบจริงในสภาพแวดล้อมของ Part นี้ (Rails 8.1.4) ได้เวอร์ชัน:

```
Fetching view_component 4.15.0
Installing view_component 4.15.0
Bundle complete! 24 Gemfile dependencies, 123 gems now installed.
```

**ไม่ต้องสร้างไฟล์ initializer หรือตั้งค่าอะไรเพิ่มเติมเลย** — ทดสอบแล้วยืนยันว่า gem นี้ทำงานได้
ทันทีหลัง `bundle install` เพราะ:

1. `ViewComponent::Engine` เป็น Rails::Engine ที่ hook เข้ากับวงจรชีวิตของแอปให้อัตโนมัติ
   (ทบทวนแนวคิด Rails Engine เบื้องต้นจาก preview ที่ Part 088 จะเจาะลึกอีกครั้ง)
2. component ทุกตัวจะถูกวางไว้ที่ `app/components/` — และตั้งแต่ Rails 6 เป็นต้นมา **ทุกโฟลเดอร์
   ที่อยู่ตรงใต้ `app/` จะถูกเพิ่มเข้า autoload path ให้อัตโนมัติโดย Zeitwerk** (ทบทวน preview
   เรื่อง Zeitwerk จาก Part 020 Step 197) จึงไม่ต้อง `require` class ใดๆ เอง เหมือนที่ `app/models/`
   หรือ `app/helpers/` ทำงานได้เลยโดยไม่ต้อง config เพิ่ม

> **หมายเหตุเรื่อง RSpec:** ถ้าโปรเจกต์ใช้ `rspec-rails` อยู่แล้ว (ทบทวน Part 046) generator ของ
> ViewComponent จะ**ตรวจจับได้เองอัตโนมัติ**และสร้างไฟล์ spec แบบ RSpec ให้ทันทีตอน generate
> component ใหม่ (แสดงให้เห็นจริงใน Step 543) โดยไม่ต้อง config อะไรเพิ่ม เพราะ generator เช็ค
> ว่ามี `rspec-rails` อยู่ใน `Gemfile.lock` หรือไม่ก่อนเลือกใช้ template ที่เหมาะสม

---

## Step 543: Generate component ตัวแรก — กายวิภาคของไฟล์คู่ `.rb` + `.html.erb`

### คำสั่ง generator

```bash
bin/rails generate view_component:component Card title
```

> **ข้อควรระวังเรื่องชื่อคำสั่ง:** บทความ/tutorial เก่าบางแหล่งเขียนคำสั่งย่อไว้ว่า
> `rails generate component Card title` (ไม่มี namespace `view_component:`) — **ทดสอบจริงบน
> view_component 4.15.0 แล้วพบว่าคำสั่งย่อนี้ใช้ไม่ได้** จะได้ error
> `Could not find generator 'component'` ทันที เวอร์ชันปัจจุบันลง namespace ไว้ชัดเจนว่า
> `view_component:component` เสมอ ถ้าไม่แน่ใจว่า generator ชื่อาอะไรกันแน่ในเวอร์ชันที่ใช้อยู่
> ให้รัน `bin/rails generate --help | grep -i component` ดูรายชื่อ generator ทั้งหมดที่มีก่อนเสมอ

ผลลัพธ์การรันจริง:

```
      create  app/components/card_component.rb
      invoke  rspec
      create    spec/components/card_component_spec.rb
      invoke  tailwindcss
      create    app/components/card_component.html.erb
```

สังเกตว่า generator ตรวจพบทั้ง `rspec-rails` (สร้าง spec ให้อัตโนมัติ) และ `tailwindcss-rails`
(เลือก template ERB แบบที่เตรียม data attribute ไว้ให้เผื่อใช้กับ Tailwind IntelliSense) โดยไม่ต้อง
ระบุ flag ใดๆ เอง — เป็นตัวอย่างที่ดีของ **smart generator** ที่ Rails generator หลายตัวทำ (ทบทวน
แนวคิด generator ที่ตรวจ environment จาก Part 028)

### ไฟล์ที่ได้: `app/components/card_component.rb`

```ruby
# frozen_string_literal: true

class CardComponent < ViewComponent::Base
  def initialize(title:)
    @title = title
  end
end
```

นี่คือหัวใจของ ViewComponent: **`initialize(title:)` คือสัญญา (explicit interface) ของ component
ตัวนี้** เปิดไฟล์นี้ขึ้นมาบรรทัดเดียวก็รู้ทันทีว่าต้องส่ง keyword argument `title:` มาเสมอ ถ้าลืม
ส่งมา Ruby จะ raise `ArgumentError: missing keyword: :title` **ทันทีตอนสร้าง object** ไม่ใช่ตอน
render กลางหน้าเว็บเหมือน partial ที่ลืมส่ง local — ตรงตามที่ Step 541 อธิบายไว้ว่า ViewComponent
แก้ปัญหา implicit interface ด้วยการใช้กลไก method signature ของ Ruby เองตรงๆ (ทบทวน keyword
argument จาก Part 007 Step 63)

`CardComponent < ViewComponent::Base` — ทุก component ต้องสืบทอดจาก `ViewComponent::Base` เสมอ
(เทียบได้กับที่ Model ทุกตัวสืบทอดจาก `ApplicationRecord` — ทบทวน Part 025) class นี้ให้ method
พื้นฐานที่จำเป็นทั้งหมด เช่น `render_in`, `content`, และ helper ของ Rails ทุกตัวที่ view ปกติใช้ได้
(`link_to`, `content_tag`, `number_to_currency` ฯลฯ — ทบทวนจาก Part 024 Step 237) ก็เรียกใช้ได้
ภายใน component เช่นกัน

### ไฟล์ที่ได้: `app/components/card_component.html.erb`

Generator สร้างไฟล์ placeholder เปล่าๆ มาให้:

```erb
<div>Add Card template here</div>
```

แก้เป็นเทมเพลตจริง โดยใช้ `@title` ที่ตั้งไว้ใน `initialize`:

```erb
<%# app/components/card_component.html.erb %>
<div class="card">
  <div class="card-header">
    <h3><%= @title %></h3>
  </div>
  <div class="card-body">
    <%= content %>
  </div>
</div>
```

**`<%= content %>`** คือ method พิเศษของ `ViewComponent::Base` ที่คืนค่าเนื้อหาของ block ที่ส่งเข้า
มาตอนเรียก `render` — เทียบได้กับ `yield` ของ Ruby block ทั่วไป (ทบทวน Part 008) เดี๋ยว Step 544
จะสาธิตให้เห็นว่ามันทำงานอย่างไร

### กฎการตั้งชื่อไฟล์ (Sidecar Convention)

สังเกตว่าไฟล์ทั้งสองอยู่ **โฟลเดอร์เดียวกัน** (`app/components/`) และตั้งชื่อ **เหมือนกันทุก
ตัวอักษร** ต่างกันแค่นามสกุล (`card_component.rb` คู่กับ `card_component.html.erb`) — นี่คือ
รูปแบบที่เรียกว่า **sidecar** ViewComponent ใช้ชื่อไฟล์ในการจับคู่ class กับ template ให้อัตโนมัติ
โดยไม่ต้องเขียนโค้ดผูกเอง (ธรรมเนียมเดียวกับที่ Rails จับคู่ `posts_controller.rb` กับโฟลเดอร์
`app/views/posts/` แม้จะเป็นคนละกลไกก็ตาม — ทบทวน Part 023–024)

---

## Step 544: Render component จาก View — `render(Component.new(...))`

การเรียกใช้ component จาก view ปกติทำผ่าน helper `render` ตัวเดียวกับที่ใช้ render partial
(ทบทวน Part 024 Step 235) เพียงแค่ส่ง **instance ของ component class** เข้าไปแทนชื่อ string:

```erb
<%# app/views/products/index.html.erb %>
<h1>สินค้าแนะนำ</h1>

<%= render(CardComponent.new(title: "คีย์บอร์ดเมคานิคอล")) do %>
  ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
<% end %>
```

ทดสอบ render จริงแล้วได้ HTML:

```html
<h1>สินค้าแนะนำ</h1>

<div class="card">
  <div class="card-header">
    <h3>คีย์บอร์ดเมคานิคอล</h3>
  </div>
  <div class="card-body">
    ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
  </div>
</div>
```

เนื้อหาใน `do ... end` block ที่ส่งเข้าไปตอน `render` คือสิ่งที่ `<%= content %>` ใน
`card_component.html.erb` คืนค่าออกมา — ตรงตามที่อธิบายไว้ใน Step 543 พอดี

### สองรูปแบบการเขียนที่เทียบเท่ากัน

```erb
<%# แบบมีวงเล็บรอบ render ทั้งก้อน — ชัดเจนว่าอาร์กิวเมนต์ของ render คืออะไร %>
<%= render(CardComponent.new(title: "สินค้า A")) %>

<%# แบบย่อ ไม่ใส่วงเล็บรอบ render — Ruby parse ได้ผลลัพธ์เดียวกันเป๊ะ %>
<%= render CardComponent.new(title: "สินค้า A") %>
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ (Ruby ตีความ `render CardComponent.new(...)` เป็น
`render(CardComponent.new(...))` อยู่แล้วตามกฎการละวงเล็บของ method call ปกติ — ทบทวน Part 007)
เลือกใช้ตามความชัดเจนที่ทีมต้องการ ส่วนใหญ่นิยมแบบมีวงเล็บเมื่อมี argument หลายตัวหรือ chain
method ต่อท้าย

### เทียบกับ partial ตรงๆ

```erb
<%# วิธีเดิมแบบ partial (Part 024) — ไม่มีการตรวจสอบ argument ใดๆ ตอน compile %>
<%= render "shared/product_card", title: "สินค้า A" %>

<%# วิธีใหม่แบบ ViewComponent — มี explicit interface ตรวจสอบตั้งแต่ initialize %>
<%= render CardComponent.new(title: "สินค้า A") %>
```

หน้าตาการเรียกใช้คล้ายกันมาก (ทั้งคู่ผ่าน helper `render` ตัวเดียวกัน) แต่เบื้องหลังต่างกันโดย
สิ้นเชิงตามที่ Step 541–543 อธิบายไว้ — นี่คือเหตุผลที่ทีมที่มีโปรเจกต์ partial อยู่แล้วสามารถ
ค่อยๆ ย้ายมาใช้ ViewComponent ทีละตัวได้โดยไม่ต้องเขียน view ใหม่ทั้งหมดในคราวเดียว

### Render component จากภายใน component อื่น หรือจาก Controller

Component เรียกใช้ได้จากทุกที่ที่มี view context เข้าถึงได้ ไม่ใช่แค่จาก `.html.erb` เท่านั้น:

```ruby
# ตัวอย่าง: render จาก controller ตรงๆ (ใช้น้อยในโปรเจกต์จริง ส่วนใหญ่ใช้ตอน debug/ทดสอบ)
render(CardComponent.new(title: "ทดสอบ"))
```

จะกลับมาเห็นการ render component ซ้อนใน component อื่นอีกครั้งใน Step 549 เรื่อง Composition

---

## Step 545: Slots — `renders_one`/`renders_many` แทนที่ boolean flag ที่ล้นมือ

Step 541 ทิ้งปัญหาไว้ข้อหนึ่ง: partial ที่ต้องรองรับหลายรูปแบบการแสดงผลมักจบลงด้วย boolean flag
ที่เพิ่มขึ้นเรื่อยๆ ปัญหาเดียวกันนี้เกิดกับ ViewComponent ได้เช่นกันถ้าออกแบบไม่ดี — มาดูตัวอย่าง
ก่อน-หลังที่แสดงให้เห็นว่า **Slot API** ของ ViewComponent แก้ปัญหานี้อย่างไร

### แบบ Before: Boolean Flag ที่ล้นมือ

```ruby
# ตัวอย่างที่ไม่ดี — เพิ่ม flag ไปเรื่อยๆ ทุกครั้งที่ต้องการ "โหมด" การแสดงผลใหม่
class CardComponent < ViewComponent::Base
  def initialize(title:, show_badge: false, badge_text: nil, show_footer: false,
                  footer_button_label: nil, footer_button_variant: :primary)
    @title = title
    @show_badge = show_badge
    @badge_text = badge_text
    @show_footer = show_footer
    @footer_button_label = footer_button_label
    @footer_button_variant = footer_button_variant
  end
end
```

```erb
<%# เรียกใช้แล้วต้องจำ flag 5 ตัวว่าตัวไหนคู่กับตัวไหน %>
<%= render CardComponent.new(
      title: "สินค้า A",
      show_badge: true,
      badge_text: "ลดราคา",
      show_footer: true,
      footer_button_label: "เพิ่มลงตะกร้า",
      footer_button_variant: :primary
    ) %>
```

ปัญหาของแนวทางนี้:

1. **จำกัดเนื้อหาให้เป็น string/symbol เท่านั้น** — ถ้าอยากให้ badge มี icon ประกอบ หรือ footer
   มีปุ่ม 2 ปุ่มพร้อมกัน ต้องเพิ่ม parameter ใหม่ไปเรื่อยๆ ไม่มีที่สิ้นสุด
2. **`initialize` ยาวขึ้นเรื่อยๆ** จนอ่านไม่รู้เรื่องว่า parameter ไหนสำคัญ ไหนเป็น optional
   (เทียบกับปัญหา "long parameter list" ที่ Part 016 พูดถึงตอนเรียน SOLID/design principle)
3. **Component ควบคุม HTML ภายในทุกส่วนเอง** — ถ้าอยากปรับแค่ footer นิดหน่อยในบางหน้า (เช่น
   ใส่ปุ่มที่สาม) ต้องแก้ที่ `CardComponent` เอง กระทบทุกที่ที่เรียกใช้อยู่แล้ว

### แบบ After: Slot API

**Slot** คือกลไกของ ViewComponent ที่ให้ "พื้นที่" ใน component ยอมรับเนื้อหา (ซึ่งอาจเป็น string,
HTML ที่ซับซ้อน หรือแม้แต่ component อื่นซ้อนอยู่ข้างใน — ดู Step 549) แทนที่จะยอมรับแค่ string/
boolean ธรรมดา:

```ruby
# frozen_string_literal: true

class CardComponent < ViewComponent::Base
  renders_one :header
  renders_one :footer
  renders_many :items

  def initialize(title: nil)
    @title = title
  end
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="card">
  <div class="card-header">
    <% if header? %>
      <%= header %>
    <% else %>
      <h3><%= @title %></h3>
    <% end %>
  </div>

  <div class="card-body">
    <%= content %>

    <% if items? %>
      <ul class="card-items">
        <% items.each do |item| %>
          <li><%= item %></li>
        <% end %>
      </ul>
    <% end %>
  </div>

  <% if footer? %>
    <div class="card-footer"><%= footer %></div>
  <% end %>
</div>
```

**`renders_one :header`** ประกาศ slot เดี่ยว (มีได้แค่ 0 หรือ 1 ค่า) ปลดล็อก method ให้ทันที
3 ตัว:

- **`header?`** — predicate เช็คว่ามีใครส่งเนื้อหาเข้ามาใน slot นี้หรือยัง (คืน `true`/`false`)
- **`with_header { ... }`** — เรียกจากฝั่งผู้ใช้ component เพื่อ "ใส่" เนื้อหาเข้าไปใน slot
- **`header`** — เรียกจากภายในเทมเพลตของ component เองเพื่อ "ดึง" เนื้อหาที่ถูกใส่ไว้ออกมาแสดงผล

**`renders_many :items`** เหมือนกันทุกประการแต่รองรับ **หลายค่า** — ได้ `items?`, `with_item`
(เอกพจน์ — เรียกทีละครั้งเพื่อเพิ่มสมาชิกใหม่), และ `items` (คืน Array ให้ `each` วนได้ตามปกติ)

### เรียกใช้จริง

```erb
<%= render(CardComponent.new) do |card| %>
  <% card.with_header do %>
    <span class="badge">ใหม่</span> คีย์บอร์ดเมคานิคอล
  <% end %>

  <% card.with_item { "สวิตช์: Brown (เสียงเบา)" } %>
  <% card.with_item { "การเชื่อมต่อ: USB-C / Bluetooth 5.0" } %>

  ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
<% end %>
```

ทดสอบ render จริงแล้วได้ HTML (ตัด whitespace ส่วนเกินออก):

```html
<div class="card">
  <div class="card-header">
    <span class="badge">ใหม่</span> คีย์บอร์ดเมคานิคอล
  </div>
  <div class="card-body">
    ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
    <ul class="card-items">
      <li>สวิตช์: Brown (เสียงเบา)</li>
      <li>การเชื่อมต่อ: USB-C / Bluetooth 5.0</li>
    </ul>
  </div>
</div>
```

สังเกตว่า **block ที่ส่งให้ `render` รับ argument ตัวหนึ่งเสมอ** (ในตัวอย่างนี้ตั้งชื่อว่า `card`)
ซึ่งก็คือ instance ของ `CardComponent` เอง — เรียก `card.with_header`, `card.with_item` ผ่านตัวนี้
ได้โดยตรง (เทียบกับ pattern `|t, args|` ที่ Rake task รับใน Part 020 Step 196 — เป็นแนวคิดเดียวกัน
คือ block parameter ให้ "handle" ของสิ่งที่กำลังทำงานอยู่กลับมาให้เราสั่งงานต่อ)

ถ้าไม่ส่ง `header` มาเลย component จะ fallback ไปแสดง `@title` แบบเดิมแทน (เพราะเช็ค `header?`
ก่อนเสมอ) — นี่คือข้อดีสำคัญของ slot เมื่อเทียบกับ flag: **หน้าเดิมที่เรียก `CardComponent.new`
แบบง่ายๆ (title อย่างเดียวไม่มี slot) ยังใช้งานได้ตามปกติ ไม่ต้องแก้อะไรเลย** ระบบ backward
compatible กันเองโดยธรรมชาติ

### สรุปเปรียบเทียบ

| | Boolean Flag | Slot API |
|---|---|---|
| เนื้อหาที่ใส่ได้ | string/symbol เท่านั้น | HTML ซับซ้อน หรือ component อื่นซ้อนได้ |
| จำนวน parameter ใน `initialize` | เพิ่มขึ้นเรื่อยๆ ตามจำนวนโหมด | คงที่ ไม่โตตามความซับซ้อนของเนื้อหา |
| ปรับแต่งเฉพาะจุดโดยไม่แก้ component | ทำไม่ได้ | ทำได้ (ผู้เรียกใช้ควบคุมเนื้อหาใน slot เอง) |
| ความชัดเจนตอนอ่านโค้ดที่เรียกใช้ | ต้องจำว่า flag ไหนคู่กับอะไร | อ่านลำดับ `with_header`/`with_item` ตามธรรมชาติ |

---

## Step 546: Component-local CSS/JS — colocate สไตล์และสคริปต์ไว้กับ component

หนึ่งในจุดขายที่สำคัญที่สุดของ ViewComponent คือการให้ไฟล์ CSS และ JavaScript ของ component
**อยู่โฟลเดอร์เดียวกับไฟล์ `.rb`/`.html.erb`** แทนที่จะกระจัดกระจายไปตาม `app/assets/stylesheets/`
กับ `app/javascript/controllers/` เหมือนที่ Step 541 ชี้ปัญหาไว้ มาสร้าง `ButtonComponent` ใหม่
เพื่อสาธิตเรื่องนี้โดยเฉพาะ

### สร้าง ButtonComponent

```bash
bin/rails generate view_component:component Button label variant
```

```
      create  app/components/button_component.rb
      invoke  rspec
      create    spec/components/button_component_spec.rb
      invoke  tailwindcss
      create    app/components/button_component.html.erb
```

แก้เป็นเนื้อหาจริง:

```ruby
# frozen_string_literal: true

# app/components/button_component.rb
class ButtonComponent < ViewComponent::Base
  VARIANTS = {
    primary: "btn btn-primary",
    danger: "btn btn-danger",
    secondary: "btn btn-secondary"
  }.freeze

  def initialize(label:, variant: :primary, href: nil)
    @label = label
    @variant = variant
    @href = href
  end

  private

  # method นี้เป็น private เพราะเป็นรายละเอียดภายในของการ render เทมเพลต ไม่ใช่ส่วนหนึ่งของ
  # "สัญญา" ที่ผู้ใช้ component ต้องรู้ (ทบทวนหลักการซ่อนรายละเอียดภายในจาก Part 009/016)
  def css_class
    VARIANTS.fetch(@variant, VARIANTS[:primary])
  end
end
```

```erb
<%# app/components/button_component.html.erb %>
<% if @href %>
  <%= link_to @label, @href, class: css_class %>
<% else %>
  <button type="button" class="<%= css_class %>"
          data-controller="button-component" data-action="click->button-component#submit">
    <%= @label %>
  </button>
<% end %>
```

Component เดียวเลือก render เป็น `<a>` หรือ `<button>` ได้ตามว่ามี `href:` ส่งมาหรือไม่ — เป็น
การรวม logic ทั้งสองแบบไว้ในที่เดียวโดยผู้เรียกใช้ไม่ต้องสนใจว่าข้างในตัดสินใจอย่างไร (encapsulation
ทบทวน Part 016)

### เพิ่ม CSS colocate — `button_component.css`

```css
/* app/components/button_component.css
   CSS ที่ colocate กับ ButtonComponent โดยเฉพาะ — เห็นไฟล์คู่กันในโฟลเดอร์เดียว ไม่ต้องไปงมหา
   ใน app/assets/stylesheets/ ที่อาจมี partial component อื่นปนกันเป็นร้อยไฟล์ */
.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
  border: none;
}

.btn-primary { background-color: #2563eb; color: white; }
.btn-secondary { background-color: #e5e7eb; color: #111827; }
.btn-danger { background-color: #dc2626; color: white; }
```

### เพิ่ม Stimulus Controller colocate — `button_component_controller.js`

ใช้ generator ของ ViewComponent เองในการสร้าง Stimulus controller คู่กับ component (ทบทวน
Stimulus controller/action/target จาก Part 053):

```bash
bin/rails generate view_component:stimulus Button
```

```
      create  app/components/button_component_controller.js
```

แก้เป็นเนื้อหาจริง:

```javascript
// app/components/button_component_controller.js
import { Controller } from "@hotwired/stimulus";

// Stimulus controller ที่ colocate อยู่กับ ButtonComponent โดยเฉพาะ
// เชื่อมด้วย data-controller="button-component" ในเทมเพลตของ component เอง
export default class extends Controller {
  static values = { loadingText: { type: String, default: "กำลังทำงาน..." } };

  connect() {
    this.originalText = this.element.textContent;
  }

  submit() {
    this.element.disabled = true;
    this.element.textContent = this.loadingTextValue;
  }
}
```

### โครงสร้างไฟล์ที่ได้ทั้งหมด

```
app/components/
├── button_component.rb                  # Ruby class: logic + explicit interface
├── button_component.html.erb            # Template: HTML structure
├── button_component.css                 # Style: เฉพาะของปุ่มตัวนี้เท่านั้น
└── button_component_controller.js       # Behavior: Stimulus controller เฉพาะของปุ่มตัวนี้
```

ไฟล์ทั้ง 4 ตัวคือ "component" เดียวกัน มองจากมุมไฟล์ระบบก็เห็นชัดว่าเกี่ยวข้องกันทันที ไม่ต้องเดา
หรือจำเอง — ตรงตามที่ Step 541 ปัญหาที่ 3 ระบุไว้พอดี

### ต้องต่อสายให้ Propshaft และ importmap รู้จักโฟลเดอร์นี้ (สำคัญมาก)

**ทดสอบจริงแล้วพบว่า** ถ้าหยุดแค่สร้างไฟล์ 4 ไฟล์ข้างบน CSS/JS จะ**ไม่ถูกโหลดเข้าเว็บเพจเลย**
เพราะ Propshaft (asset pipeline ของ Rails 8 — ทบทวน Part 029) และ importmap-rails (ทบทวน
Part 021) ยังไม่รู้จักโฟลเดอร์ `app/components/` ว่าเป็นแหล่ง asset ต้องเพิ่ม config เอง 2 จุด:

**1) เพิ่ม `app/components` เข้า asset path ของ Propshaft (สำหรับ CSS):**

```ruby
# config/initializers/assets.rb
# ให้ Propshaft มองเห็นไฟล์ CSS/JS ที่ colocate (sidecar) อยู่ในโฟลเดอร์เดียวกับแต่ละ component
# เช่น app/components/button_component.css และ app/components/button_component_controller.js
Rails.application.config.assets.paths << Rails.root.join("app/components")
```

**2) Pin ไฟล์ Stimulus controller ที่อยู่ใน `app/components` เข้า importmap (สำหรับ JS):**

```ruby
# config/importmap.rb
pin_all_from "app/javascript/controllers", under: "controllers"

# Pin Stimulus controller ที่ colocate อยู่กับ component แต่ละตัวใน app/components/
# pin_all_from จะสแกนหาเฉพาะไฟล์ .js เท่านั้น (ไม่ยุ่งกับ .rb/.html.erb/.css ที่อยู่โฟลเดอร์เดียวกัน)
# ต้องระบุ `to: ""` เพื่อไม่ให้ path จริงของไฟล์ (ซึ่งไม่มี "controllers/" นำหน้า เพราะรากของ
# asset path คือ app/components ตรงๆ) ถูกเติม "controllers/" ซ้ำเข้าไปโดยไม่ตั้งใจ
pin_all_from "app/components", under: "controllers", to: ""
```

> **ทดสอบจริงแล้วพบ gotcha ที่สำคัญมาก:** ลองใช้ `pin_all_from "app/components", under: "controllers"`
> โดยไม่ใส่ `to: ""` ดูก่อน แล้วตรวจสอบด้วย
> `Rails.application.importmap.preloaded_module_packages(resolver: ApplicationController.helpers)`
> พบว่า **ไม่มี entry ของ `button_component_controller` ปรากฏออกมาเลย** เงียบๆ โดยไม่มี error ใดๆ —
> สาเหตุคือ `pin_all_from` โดยปกติจะประกอบ path ของไฟล์เป็น `"controllers/button_component_controller.js"`
> (เอา `under:` มาต่อหน้า) แต่ Propshaft กลับมองไฟล์จริงเป็น `"button_component_controller.js"` เฉยๆ
> (ไม่มี `"controllers/"` นำหน้า เพราะเราเพิ่ม `app/components` เป็น asset root ตรงๆ ไม่ใช่
> `app/components` ที่ซ้อนอยู่ใต้โฟลเดอร์ชื่อ `controllers`) เมื่อ path ที่ import map คาดหวังกับ
> path จริงที่ Propshaft หาเจอไม่ตรงกัน มันจะถูกข้ามไปเงียบๆ (`filter_map` ทิ้ง entry ที่ resolve
> ไม่ได้) การใส่ `to: ""` คือการบอกว่า **"ใช้ path ของไฟล์ตามที่มันเป็นจริง แค่ให้ตั้งชื่อเรียก
> (specifier) นำหน้าด้วย `controllers/` เท่านั้นพอ"** ซึ่งแก้ปัญหานี้ได้ตรงจุด — เป็นตัวอย่างที่ดี
> ว่าทำไมการทดสอบให้เห็นผลจริงเสมอ (ไม่ใช่แค่เขียนโค้ดตามความรู้สึกว่าน่าจะถูก) สำคัญกับงาน Rails

### ตรวจสอบผลลัพธ์จริง

ทดสอบ boot server แล้วขอไฟล์ทั้งสองตรงๆ ผ่าน browser/`curl`:

```bash
curl -s http://localhost:3000/assets/button_component-d3d1bffd.css
curl -s http://localhost:3000/assets/button_component_controller-a7f94016.js
```

ได้เนื้อหาไฟล์กลับมาครบถ้วนทั้งคู่ (status `200`) เลขฐาน 16 ต่อท้ายชื่อไฟล์คือ **content digest**
ที่ Propshaft ใส่ให้อัตโนมัติเพื่อทำ cache-busting (ทบทวนแนวคิด fingerprinting จาก Part 029) และ
ตรวจสอบ `<script type="importmap">` ที่ฝังอยู่ในหน้า HTML พบ entry ใหม่ปรากฏขึ้นมาจริง:

```json
"controllers/button_component_controller": "/assets/button_component_controller-a7f94016.js"
```

ตรงกับ `data-controller="button-component"` ที่ใส่ไว้ในเทมเพลตพอดี (Stimulus แปลงชื่อไฟล์
`button_component_controller.js` เป็น identifier `button-component` โดยอัตโนมัติ — ตัด `_controller`
ท้ายชื่อออกแล้วเปลี่ยน `_` เป็น `-` ทั้งหมด ทบทวนกฎการตั้งชื่อ Stimulus controller จาก Part 053)
เพิ่ม `<%= stylesheet_link_tag "button_component" %>` ไว้ใน layout หรือหน้าที่เกี่ยวข้อง (ทบทวน
`content_for(:head)` จาก Part 024 Step 239) ก็จะได้ทั้ง style และ behavior ทำงานครบสมบูรณ์

---

## Step 547: Preview component แบบแยกโดด — `ViewComponent::Preview` และหน้า `/rails/view_components`

การจะเห็นหน้าตาของ component หนึ่งตัว ไม่จำเป็นต้องไปประกอบมันเข้ากับ controller/route/หน้าเว็บ
จริงเลยด้วยซ้ำ — ViewComponent มีระบบ **Preview** ในตัว ที่ยืมแนวคิดมาจาก Rails
`ActionMailer::Preview` (ระบบดู preview อีเมลโดยไม่ต้องส่งจริง)

### สร้างไฟล์ Preview

```bash
bin/rails generate view_component:preview Card
```

```
      create  test/components/previews/card_component_preview.rb
```

> **หมายเหตุ:** ไฟล์ preview ถูกสร้างไว้ใต้ `test/components/previews/` เป็นค่า default เสมอ
> (แม้โปรเจกต์จะใช้ RSpec เป็นหลักก็ตาม) เพราะ path นี้ถูกกำหนดไว้ในตัว config เอง ไม่เกี่ยวกับว่า
> ใช้ Minitest หรือ RSpec — เปลี่ยนได้ด้วย `config.view_component.previews.paths` ใน
> `config/application.rb` ถ้าต้องการย้ายไปที่อื่น แต่ส่วนใหญ่ปล่อยตามค่า default ก็เพียงพอ

แก้ไฟล์ให้มีตัวอย่างการใช้งานจริงหลายแบบ:

```ruby
# frozen_string_literal: true

# test/components/previews/card_component_preview.rb
class CardComponentPreview < ViewComponent::Preview
  # http://localhost:3000/rails/view_components/card_component/default
  def default
    render(CardComponent.new) do |card|
      card.with_header { "การ์ดตัวอย่าง" }
      card.with_item { "รายการที่ 1" }
      card.with_item { "รายการที่ 2" }
      card.with_footer { render(ButtonComponent.new(label: "ตกลง")) }
      "เนื้อหาหลักของการ์ด อยู่ตรงนี้"
    end
  end

  # http://localhost:3000/rails/view_components/card_component/without_slots
  def without_slots
    render(CardComponent.new(title: "การ์ดแบบไม่มี slot")) { "แสดงแค่ title ธรรมดา" }
  end
end
```

หนึ่ง **method สาธารณะ** ในคลาส Preview คือหนึ่ง "ตัวอย่าง" ของ component ที่ดูได้แยกกัน — ตั้งชื่อ
method ให้สื่อความหมายว่ากำลังสาธิต state/รูปแบบไหน (`default`, `without_slots`, `with_long_title`,
`disabled_state` ฯลฯ)

### เปิดดูผ่านหน้าเว็บ

รัน dev server ตามปกติ (`bin/dev` หรือ `bin/rails server`) แล้วเปิด:

```
http://localhost:3000/rails/view_components
```

ทดสอบจริงแล้วได้ status `200` พร้อมหน้ารายการ preview ทั้งหมดที่มีในระบบ (มีทั้ง `card_component`
พร้อม 2 ตัวอย่างย่อย `default`/`without_slots`) คลิกเข้าไปที่ตัวอย่างใดตัวอย่างหนึ่งจะได้ URL แบบ:

```
http://localhost:3000/rails/view_components/card_component/default
```

หน้านี้ render component ตัวนั้นแบบ**โดดๆ ไม่มี layout ของแอปครอบ** — เหมาะมากสำหรับนักออกแบบ
(designer) หรือ frontend developer ที่ต้องการดู/ปรับแต่ง component ตัวเดียว โดยไม่ต้องเดินผ่าน
flow ทั้งหมดของแอป (ล็อกอิน, สร้างข้อมูลตัวอย่าง ฯลฯ) ก่อนจะเห็นหน้าตาของมัน — แก้ปัญหาที่ partial
ไม่เคยมีทางแก้ได้เลย (ต้อง render ทั้งหน้าเสมอถึงจะเห็น partial ตัวหนึ่งทำงาน)

### ทำไมฟีเจอร์นี้ถึงเปิดใช้งานอยู่เองโดยไม่ต้อง config

ทดสอบเจาะโค้ดของ gem (`ViewComponent::Config`) พบค่า default ตรงนี้:

```ruby
options.previews.enabled = defined?(Rails.env) && (Rails.env.development? || Rails.env.test?)
options.previews.route = "/rails/view_components"
options.previews.paths = default_preview_paths  # หา test/components/previews/ ให้อัตโนมัติ
```

Preview **เปิดใช้งานเองอัตโนมัติเฉพาะใน development และ test environment เท่านั้น** (ปิดใน
production เสมอโดย default เพื่อไม่ให้ route นี้หลุดออกไปให้คนภายนอกเข้าถึงได้) ถ้าต้องการปิดใน
development ด้วยเหตุผลบางอย่าง ตั้งค่าเองได้ที่ `config/application.rb`:

```ruby
config.view_component.previews.enabled = false
```

---

## Step 548: Unit test component ด้วย `render_inline` (RSpec)

นี่คือจุดที่ ViewComponent ต่างจาก partial อย่างชัดเจนที่สุด — เพราะ component เป็น Ruby object
ธรรมดา จึงเขียน **unit test แยกจากกันได้จริง** โดยไม่ต้องพึ่ง request spec หรือ system test แบบที่
Part 048 สอนไว้ (ทบทวนความแตกต่างระหว่าง unit/request/system test จาก Part 046–048)

### ตั้งค่า RSpec ให้รู้จัก ViewComponent

Generator ของ RSpec/ViewComponent สร้าง spec file เปล่าให้อัตโนมัติแล้ว (เห็นตั้งแต่ Step 543)
แต่ **ทดสอบจริงแล้วพบว่า** ต้องเพิ่ม config อีกเล็กน้อยใน `spec/rails_helper.rb` ก่อน ไม่งั้นจะได้
`NoMethodError: undefined method 'render_inline'`:

```ruby
# spec/rails_helper.rb
require 'rspec/rails'
# Add additional requires below this line. Rails is not loaded until this point!
require 'view_component/test_helpers'
require 'capybara/rspec'

RSpec.configure do |config|
  # ผูก helper สำหรับทดสอบ ViewComponent (render_inline) และ matcher ของ Capybara (have_css ฯลฯ)
  # เข้ากับ example ทุกตัวที่มี type: :component เท่านั้น (ทบทวนแนวคิด metadata-based include
  # จาก Part 019)
  config.include ViewComponent::TestHelpers, type: :component
  config.include Capybara::RSpecMatchers, type: :component

  # ... config อื่นๆ ตามเดิม
end
```

ต้องมี gem `capybara` อยู่ใน `Gemfile` group `:test` ด้วย (ทบทวน Capybara จาก Part 048 — ที่นี่ใช้
แค่ matcher เช่น `have_css` ไม่ได้เปิด browser จริงเหมือนตอนเขียน system test)

### เขียน spec ให้ `ButtonComponent`

```ruby
# frozen_string_literal: true

# spec/components/button_component_spec.rb
require "rails_helper"

RSpec.describe ButtonComponent, type: :component do
  it "แสดงเป็น <button> เมื่อไม่ระบุ href" do
    render_inline(described_class.new(label: "บันทึก"))

    expect(page).to have_css("button.btn.btn-primary", text: "บันทึก")
  end

  it "แสดงเป็น <a> เมื่อระบุ href" do
    render_inline(described_class.new(label: "ไปที่หน้าแรก", href: "/"))

    expect(page).to have_css("a.btn.btn-primary[href='/']", text: "ไปที่หน้าแรก")
  end

  it "เปลี่ยน CSS class ตาม variant ที่ระบุ" do
    render_inline(described_class.new(label: "ลบ", variant: :danger))

    expect(page).to have_css("button.btn.btn-danger", text: "ลบ")
  end

  it "ใช้ variant :primary เป็นค่า default เมื่อระบุ variant ที่ไม่รู้จัก" do
    render_inline(described_class.new(label: "ปุ่ม", variant: :nonexistent))

    expect(page).to have_css("button.btn.btn-primary")
  end
end
```

**`render_inline(described_class.new(...))`** คือหัวใจของการทดสอบ — สร้าง component instance
แล้ว render ออกมาเป็น HTML **โดยไม่ต้องผ่าน controller, routing, หรือ HTTP request ใดๆ เลย** เร็ว
กว่า request spec มาก เพราะข้ามชั้น middleware/routing/controller callback ทั้งหมดไปเลย

**`page`** เป็น method ที่ `ViewComponent::TestHelpers` เตรียมไว้ให้ คืนค่าเป็น
`Capybara::Node::Simple` ที่ห่อ HTML ที่เพิ่ง render ไว้ — ใช้ matcher ของ Capybara (`have_css`,
`have_text`) ตรวจสอบได้ทันที (ทบทวน matcher เหล่านี้จาก Part 048 ที่ใช้กับ system test แต่ตอนนี้
ใช้กับ component เดี่ยวๆ แทน — เป็น API เดียวกัน ต่างแค่ระดับที่ทดสอบ)

### เขียน spec ให้ `CardComponent` — ครอบคลุมทั้ง slot และ fallback

```ruby
# frozen_string_literal: true

# spec/components/card_component_spec.rb
require "rails_helper"

RSpec.describe CardComponent, type: :component do
  it "แสดง title ธรรมดาเมื่อไม่ได้ส่ง header slot มา" do
    render_inline(described_class.new(title: "โปรไฟล์ผู้ใช้")) do
      "เนื้อหาการ์ด"
    end

    expect(page).to have_css("h3", text: "โปรไฟล์ผู้ใช้")
    expect(page).to have_text("เนื้อหาการ์ด")
  end

  it "ใช้ header slot แทน title ธรรมดาได้เมื่อระบุมา" do
    render_inline(described_class.new(title: "จะไม่ถูกใช้")) do |card|
      card.with_header { "<span class='badge'>ใหม่</span> หัวข้อพิเศษ".html_safe }
      "เนื้อหาการ์ด"
    end

    expect(page).to have_css("span.badge", text: "ใหม่")
    expect(page).to have_no_text("จะไม่ถูกใช้")
  end

  it "แสดงรายการจาก renders_many :items เมื่อมีการเพิ่มเข้ามา" do
    render_inline(described_class.new(title: "รายการ")) do |card|
      card.with_item { "ข้อที่ 1" }
      card.with_item { "ข้อที่ 2" }
    end

    expect(page).to have_css("ul.card-items li", count: 2)
  end

  it "ไม่แสดง footer เมื่อไม่ได้ส่ง footer slot มา" do
    render_inline(described_class.new(title: "ไม่มี footer"))

    expect(page).to have_no_css(".card-footer")
  end
end
```

รันจริงแล้วผ่านหมด:

```bash
bundle exec rspec spec/components --format documentation
```

```
ButtonComponent
  แสดงเป็น <button> เมื่อไม่ระบุ href
  แสดงเป็น <a> เมื่อระบุ href
  เปลี่ยน CSS class ตาม variant ที่ระบุ
  ใช้ variant :primary เป็นค่า default เมื่อระบุ variant ที่ไม่รู้จัก

CardComponent
  แสดง title ธรรมดาเมื่อไม่ได้ส่ง header slot มา
  ใช้ header slot แทน title ธรรมดาได้เมื่อระบุมา
  แสดงรายการจาก renders_many :items เมื่อมีการเพิ่มเข้ามา
  ไม่แสดง footer เมื่อไม่ได้ส่ง footer slot มา

Finished in 0.03 seconds (files took 0.79 seconds to load)
8 examples, 0 failures
```

**8 examples ใช้เวลารวมแค่ 0.03 วินาที** — เร็วกว่า system test ของ Part 048 ที่ต้อง boot browser
จริงหลายร้อยเท่า และเร็วกว่า request spec ของ Part 046 ที่ต้องผ่าน routing/middleware เต็มรูปแบบ
อย่างชัดเจน เพราะทดสอบแค่ "Ruby object หนึ่งตัว render ออกมาถูกไหม" ล้วนๆ — นี่คือประโยชน์ที่จับ
ต้องได้จริงของการมี explicit interface ตามที่ Step 541 สัญญาไว้: **component ทดสอบแยกได้จริง
ไม่ต้องพึ่งอะไรอื่นเลย**

> **เชื่อมกับ Part 019/046:** โครงสร้าง `describe`/`it`/`expect().to` ที่ใช้ในนี้คือไวยากรณ์ RSpec
> เดียวกันเป๊ะกับที่เรียนใน Part 019 (RSpec เบื้องต้น) และ Part 046 (RSpec สำหรับ Rails model/
> request spec) ต่างกันแค่ `type: :component` ที่บอก RSpec ให้ผูก `ViewComponent::TestHelpers`
> เข้ามาให้ (เทียบกับ `type: :model`/`type: :request` ที่ผูก helper คนละชุดกัน)

---

## Step 549: Composition — ประกอบ component เข้าด้วยกัน (`CardComponent` ใช้ `ButtonComponent`)

ประโยชน์ที่ทรงพลังที่สุดอย่างหนึ่งของการที่ component เป็น Ruby object ธรรมดา คือการ **render
component หนึ่งไว้ข้างในอีกตัวหนึ่งได้อย่างเป็นธรรมชาติ** เหมือนการเรียก method ซ้อนกัน —
เรียกว่า **Composition** สร้างหน้าสินค้าจริงที่ประกอบ `CardComponent` เข้ากับ `ButtonComponent`
สองตัวไว้ใน footer slot:

```erb
<%# app/views/products/index.html.erb %>
<% content_for(:head) { stylesheet_link_tag "button_component" } %>

<h1 class="text-2xl font-bold mb-4">สินค้าแนะนำ</h1>

<%= render(CardComponent.new) do |card| %>
  <% card.with_header do %>
    <span class="badge">ใหม่</span> คีย์บอร์ดเมคานิคอล
  <% end %>

  <% card.with_item { "สวิตช์: Brown (เสียงเบา)" } %>
  <% card.with_item { "การเชื่อมต่อ: USB-C / Bluetooth 5.0" } %>

  <% card.with_footer do %>
    <%= render(ButtonComponent.new(label: "เพิ่มลงตะกร้า", variant: :primary)) %>
    <%= render(ButtonComponent.new(label: "รายละเอียดเพิ่มเติม", variant: :secondary, href: "#")) %>
  <% end %>

  ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
<% end %>
```

ทดสอบ render จริงทั้งหน้าแล้วได้ HTML สมบูรณ์:

```html
<h1 class="text-2xl font-bold mb-4">สินค้าแนะนำ</h1>

<div class="card">
  <div class="card-header">
    <span class="badge">ใหม่</span> คีย์บอร์ดเมคานิคอล
  </div>

  <div class="card-body">
    ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง

    <ul class="card-items">
      <li>สวิตช์: Brown (เสียงเบา)</li>
      <li>การเชื่อมต่อ: USB-C / Bluetooth 5.0</li>
    </ul>
  </div>

  <div class="card-footer">
    <button type="button" class="btn btn-primary">เพิ่มลงตะกร้า</button>
    <a class="btn btn-secondary" href="#">รายละเอียดเพิ่มเติม</a>
  </div>
</div>
```

`ButtonComponent` ทั้งสองตัวถูก render เข้าไปอยู่ใน `.card-footer` ได้อย่างสมบูรณ์ — **`CardComponent`
ไม่รู้จัก `ButtonComponent` เลยแม้แต่น้อย** (ไม่มีการ `require`/reference ถึงกันในโค้ด `.rb` ของ
ทั้งคู่) ความเชื่อมโยงเกิดขึ้นแค่ตรงจุดที่ **view เรียกใช้ทั้งสองตัวพร้อมกัน** เท่านั้น — นี่คือหลัก
**loose coupling** (การเชื่อมต่อกันแบบหลวมๆ) ที่ Part 016 พูดถึงตอนเรียน SOLID principle:
component ยิ่งไม่รู้จักกันเอง ยิ่งเอาไปประกอบใหม่ในบริบทอื่นได้ง่ายขึ้น

### ข้อควรระวัง: การเรียก `render` ซ้อนภายใน block ของ slot ตอนเขียนเทสต์

**ทดสอบจริงแล้วพบ gotcha ที่สำคัญ:** โค้ดข้างบนใช้ `render(ButtonComponent.new(...))` ได้ตรงๆ
เพราะเขียนอยู่ใน **ไฟล์ view จริง** ซึ่งมี view context (`ActionView::Base`) ให้ `render` เรียกใช้
ได้เสมอ แต่ถ้าลองเขียน test แบบเดียวกันตรงๆ:

```ruby
# ตัวอย่างที่ error — เขียนแบบเดียวกับใน view แต่รันในบริบทของ RSpec example
render_inline(described_class.new) do |card|
  card.with_footer do
    render(ButtonComponent.new(label: "เพิ่มลงตะกร้า"))  # NoMethodError!
  end
end
```

จะได้ `NoMethodError: undefined method 'render'` ทันที เพราะ block ที่ส่งให้ `with_footer` ถูก
**ประเมินค่าในบริบทของ RSpec example เอง** (ไม่ใช่ view context) ตอนเขียน test ที่ต้อง compose
component ซ้อนกันแบบนี้ ให้ใช้ `vc_test_view_context` ที่ `ViewComponent::TestHelpers` เตรียมไว้ให้
แทน:

```ruby
it "ประกอบ ButtonComponent ไว้ใน footer slot ได้ (composition)" do
  render_inline(described_class.new(title: "สินค้า")) do |card|
    card.with_footer do
      vc_test_view_context.render(ButtonComponent.new(label: "เพิ่มลงตะกร้า", variant: :primary))
    end
    "รายละเอียดสินค้า"
  end

  expect(page).to have_css(".card-footer button.btn.btn-primary", text: "เพิ่มลงตะกร้า")
end
```

ทดสอบรันจริงแล้วผ่าน — `vc_test_view_context` คือ `ActionView::Base` instance เดียวกับที่
`render_inline` ใช้ภายใน ให้ `.render` เรียกใช้ได้ตรงๆ เหมือนอยู่ในไฟล์ view จริง **ข้อสังเกต
สำคัญ:** นี่เป็นรายละเอียดเฉพาะตอนเขียน**เทสต์**เท่านั้น โค้ดในไฟล์ `.html.erb` จริงเขียน
`render(...)` ตรงๆ ได้เสมอโดยไม่ต้องกังวลเรื่องนี้เลย

---

## Step 550: กรอบการตัดสินใจ — Partial vs Helper vs ViewComponent vs Stimulus Controller

ตอนนี้หลักสูตรสอนเครื่องมือฝั่ง view ของ Rails ครบทั้ง 4 ตัวแล้ว (partial/helper จาก Part 024,
Stimulus controller จาก Part 053, และ ViewComponent จาก Part นี้) คำถามที่ตามมาคือ **เมื่อไหร่ควร
ใช้ตัวไหน** — สรุปเป็นกรอบการตัดสินใจเดียวที่ใช้ได้จริงในงานประจำวัน:

| สถานการณ์ | เครื่องมือที่เหมาะสม | เหตุผล |
|---|---|---|
| ก้อน HTML ซ้ำๆ ไม่มี logic ซับซ้อน ไม่ต้อง reuse ข้ามโปรเจกต์ | **Partial** (Part 024) | เขียนเร็ว ไม่ต้องสร้าง class ใหม่ เหมาะกับหน้าที่ไม่ซับซ้อน |
| แปลง/จัดรูปแบบข้อมูลเป็น string หรือ HTML สั้นๆ (format ตัวเลข, สร้าง badge อันเดียว) | **View Helper** (Part 024 Step 238) | เป็นแค่ method ธรรมดา ไม่ต้องมี state หรือ template แยก |
| UI element ที่มีหลาย variant, ต้อง unit test แยก, ต้อง colocate CSS/JS ของตัวเอง, ใช้ซ้ำได้ข้ามหลาย controller/แอป | **ViewComponent** (Part นี้) | Explicit interface + unit test + slot + sidecar asset |
| พฤติกรรมฝั่ง client (DOM manipulation, event handling, toggle/show-hide, debounce) ที่ต้องรันใน browser | **Stimulus Controller** (Part 053) | JavaScript ต้องทำงานหลัง page load ในบราวเซอร์ ไม่ใช่ตอน render บน server |

**สิ่งสำคัญที่ต้องเข้าใจ:** 4 เครื่องมือนี้**ไม่ได้แข่งกัน แต่ทำงานร่วมกันเป็นชั้นๆ** ตัวอย่างที่
Step 546–549 สร้างมาแสดงให้เห็นชัดแล้ว:

- **ViewComponent** (`ButtonComponent`) กำหนดโครงสร้าง HTML + CSS ของปุ่ม
- **Stimulus Controller** (`button_component_controller.js`) ที่ colocate อยู่ในไฟล์เดียวกัน
  กำหนดพฤติกรรมตอนคลิก (disable ปุ่มระหว่างส่งฟอร์ม)
- **View Helper** ยังใช้ได้ตามปกติจากภายใน component เอง (`number_to_currency`, `link_to` ฯลฯ
  ทบทวน Part 024 Step 237)
- **Partial** ยังเหมาะกับส่วนที่เหลือของหน้าที่ไม่ต้องการความซับซ้อนระดับ component (เช่น
  breadcrumb, footer ของทั้งเว็บไซต์)

โปรเจกต์ Rails ที่ดีในโลกจริงมักใช้ทั้ง 4 อย่างผสมกันตามความเหมาะสมของแต่ละจุด ไม่ใช่เลือกใช้แค่
อย่างใดอย่างหนึ่งทั้งโปรเจกต์

### สัญญาณเตือนว่าถึงเวลาต้อง "ยกระดับ" partial ขึ้นเป็น ViewComponent

- Partial เริ่มมี parameter (`locals:`) เกิน 4–5 ตัว โดยเฉพาะถ้ามี boolean flag ปนอยู่หลายตัว
- เคยเจอบั๊กจากการลืมส่ง local ที่จำเป็นมาแล้วอย่างน้อยหนึ่งครั้ง
- อยากมี unit test เฉพาะของ UI element ตัวนั้น แยกจาก request/system test ทั้งหน้า
- UI element ตัวนั้นต้องมี CSS/JS เฉพาะตัวที่ซับซ้อนพอจะแยกไฟล์
- มีคนมากกว่า 1 คนในทีมต้องเข้าใจ "วิธีใช้งาน" ของ partial ตัวนั้น (เอกสารในรูป method signature
  ของ `initialize` สื่อสารชัดกว่าการอ่านทั้งไฟล์ template เสมอ)

---

## แบบฝึกหัดปิด Part: การ์ดสินค้าที่ประกอบด้วย Header Slot + ButtonComponent

### โจทย์

รวมทุกอย่างที่เรียนมาใน Step 541–549 เข้าด้วยกันเป็นโปรเจกต์เดียว: สร้างหน้า "สินค้าแนะนำ" ที่ใช้
`CardComponent` (พร้อม header slot สำหรับ badge/title พิเศษ, items slot สำหรับรายการ spec สินค้า,
footer slot สำหรับปุ่ม) ประกอบเข้ากับ `ButtonComponent` สองตัวในการ์ดเดียว โดยต้องมี:

1. Component ทั้งสองตัวติดตั้งครบ (`.rb` + `.html.erb`) ตามที่ออกแบบใน Step 545–546
2. RSpec unit test ครอบคลุมทุก slot และทุก variant ด้วย `render_inline`
3. ไฟล์ Preview ที่ดูผ่าน `/rails/view_components` ได้อย่างน้อย 2 ตัวอย่าง (มี slot ครบ / ไม่มี
   slot เลย)
4. หน้าเว็บจริงหนึ่งหน้าที่ประกอบทุกอย่างเข้าด้วยกันแล้ว render ออกมาถูกต้อง

### เฉลย

**`app/components/card_component.rb`** (เหมือน Step 545 — ทวนอีกครั้งเพื่อความครบถ้วนของเฉลย):

```ruby
# frozen_string_literal: true

class CardComponent < ViewComponent::Base
  renders_one :header
  renders_one :footer
  renders_many :items

  def initialize(title: nil)
    @title = title
  end
end
```

**`app/components/card_component.html.erb`:**

```erb
<div class="card">
  <div class="card-header">
    <% if header? %>
      <%= header %>
    <% else %>
      <h3><%= @title %></h3>
    <% end %>
  </div>

  <div class="card-body">
    <%= content %>

    <% if items? %>
      <ul class="card-items">
        <% items.each do |item| %>
          <li><%= item %></li>
        <% end %>
      </ul>
    <% end %>
  </div>

  <% if footer? %>
    <div class="card-footer"><%= footer %></div>
  <% end %>
</div>
```

**`app/components/button_component.rb`** (เหมือน Step 546):

```ruby
# frozen_string_literal: true

class ButtonComponent < ViewComponent::Base
  VARIANTS = {
    primary: "btn btn-primary",
    danger: "btn btn-danger",
    secondary: "btn btn-secondary"
  }.freeze

  def initialize(label:, variant: :primary, href: nil)
    @label = label
    @variant = variant
    @href = href
  end

  private

  def css_class
    VARIANTS.fetch(@variant, VARIANTS[:primary])
  end
end
```

**`app/components/button_component.html.erb`:**

```erb
<% if @href %>
  <%= link_to @label, @href, class: css_class %>
<% else %>
  <button type="button" class="<%= css_class %>"><%= @label %></button>
<% end %>
```

**`spec/components/card_component_spec.rb`** — unit test ครบทุก slot รวมถึงกรณี composition:

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe CardComponent, type: :component do
  it "แสดง title ธรรมดาเมื่อไม่ได้ส่ง header slot มา" do
    render_inline(described_class.new(title: "โปรไฟล์ผู้ใช้")) { "เนื้อหาการ์ด" }

    expect(page).to have_css("h3", text: "โปรไฟล์ผู้ใช้")
    expect(page).to have_text("เนื้อหาการ์ด")
  end

  it "ใช้ header slot แทน title ธรรมดาได้เมื่อระบุมา" do
    render_inline(described_class.new(title: "จะไม่ถูกใช้")) do |card|
      card.with_header { "<span class='badge'>ใหม่</span> หัวข้อพิเศษ".html_safe }
      "เนื้อหาการ์ด"
    end

    expect(page).to have_css("span.badge", text: "ใหม่")
    expect(page).to have_no_text("จะไม่ถูกใช้")
  end

  it "แสดงรายการจาก renders_many :items เมื่อมีการเพิ่มเข้ามา" do
    render_inline(described_class.new(title: "รายการ")) do |card|
      card.with_item { "ข้อที่ 1" }
      card.with_item { "ข้อที่ 2" }
    end

    expect(page).to have_css("ul.card-items li", count: 2)
  end

  it "ไม่แสดง footer เมื่อไม่ได้ส่ง footer slot มา" do
    render_inline(described_class.new(title: "ไม่มี footer"))

    expect(page).to have_no_css(".card-footer")
  end

  it "ประกอบ ButtonComponent ไว้ใน footer slot ได้ (composition)" do
    render_inline(described_class.new(title: "สินค้า")) do |card|
      card.with_footer do
        vc_test_view_context.render(ButtonComponent.new(label: "เพิ่มลงตะกร้า", variant: :primary))
      end
      "รายละเอียดสินค้า"
    end

    expect(page).to have_css(".card-footer button.btn.btn-primary", text: "เพิ่มลงตะกร้า")
  end
end
```

**`spec/components/button_component_spec.rb`:**

```ruby
# frozen_string_literal: true

require "rails_helper"

RSpec.describe ButtonComponent, type: :component do
  it "แสดงเป็น <button> เมื่อไม่ระบุ href" do
    render_inline(described_class.new(label: "บันทึก"))
    expect(page).to have_css("button.btn.btn-primary", text: "บันทึก")
  end

  it "แสดงเป็น <a> เมื่อระบุ href" do
    render_inline(described_class.new(label: "ไปที่หน้าแรก", href: "/"))
    expect(page).to have_css("a.btn.btn-primary[href='/']", text: "ไปที่หน้าแรก")
  end

  it "เปลี่ยน CSS class ตาม variant ที่ระบุ" do
    render_inline(described_class.new(label: "ลบ", variant: :danger))
    expect(page).to have_css("button.btn.btn-danger", text: "ลบ")
  end

  it "ใช้ variant :primary เป็นค่า default เมื่อระบุ variant ที่ไม่รู้จัก" do
    render_inline(described_class.new(label: "ปุ่ม", variant: :nonexistent))
    expect(page).to have_css("button.btn.btn-primary")
  end
end
```

รันทั้งชุดแล้วผ่านครบ **9 examples, 0 failures** (ยืนยันด้วยการรันจริงแล้วในสภาพแวดล้อมของ Part นี้)

**`test/components/previews/card_component_preview.rb`:**

```ruby
# frozen_string_literal: true

class CardComponentPreview < ViewComponent::Preview
  # http://localhost:3000/rails/view_components/card_component/default
  def default
    render(CardComponent.new) do |card|
      card.with_header { "การ์ดตัวอย่าง" }
      card.with_item { "รายการที่ 1" }
      card.with_item { "รายการที่ 2" }
      card.with_footer { render(ButtonComponent.new(label: "ตกลง")) }
      "เนื้อหาหลักของการ์ด อยู่ตรงนี้"
    end
  end

  # http://localhost:3000/rails/view_components/card_component/without_slots
  def without_slots
    render(CardComponent.new(title: "การ์ดแบบไม่มี slot")) { "แสดงแค่ title ธรรมดา" }
  end
end
```

**`app/views/products/index.html.erb`** — หน้าเว็บจริงที่ประกอบทุกอย่างเข้าด้วยกัน:

```erb
<% content_for(:head) { stylesheet_link_tag "button_component" } %>

<h1 class="text-2xl font-bold mb-4">สินค้าแนะนำ</h1>

<%= render(CardComponent.new) do |card| %>
  <% card.with_header do %>
    <span class="badge">ใหม่</span> คีย์บอร์ดเมคานิคอล
  <% end %>

  <% card.with_item { "สวิตช์: Brown (เสียงเบา)" } %>
  <% card.with_item { "การเชื่อมต่อ: USB-C / Bluetooth 5.0" } %>

  <% card.with_footer do %>
    <%= render(ButtonComponent.new(label: "เพิ่มลงตะกร้า", variant: :primary)) %>
    <%= render(ButtonComponent.new(label: "รายละเอียดเพิ่มเติม", variant: :secondary, href: "#")) %>
  <% end %>

  ราคา 2,490 บาท พร้อมส่งภายใน 24 ชั่วโมง
<% end %>
```

**`app/controllers/products_controller.rb`:**

```ruby
# frozen_string_literal: true

class ProductsController < ApplicationController
  def index
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "products#index"
end
```

ทดสอบเปิด `http://localhost:3000/` จริงแล้วได้หน้าการ์ดสินค้าที่มี badge "ใหม่" ในหัวการ์ด, spec
สินค้า 2 รายการ, ปุ่ม "เพิ่มลงตะกร้า" (สีน้ำเงิน) และ "รายละเอียดเพิ่มเติม" (สีเทา, เป็นลิงก์) อยู่
ใน footer ตามที่ออกแบบไว้ทุกประการ — ครบทั้ง generate, slot, colocated asset, unit test,
preview, และ composition ในโปรเจกต์เดียว

### แบบฝึกหัดเพิ่มเติม (ทำเอง — ไม่มีเฉลย)

1. **เพิ่ม `renders_one :badge` แยกออกจาก `:header`** ให้ `CardComponent` สำหรับติดป้าย "ลดราคา"/
   "หมด" ไว้ที่มุมการ์ดโดยเฉพาะ (ไม่ปนกับหัวข้อ) พร้อมเขียน unit test ว่าถ้าไม่ส่ง badge มาจะไม่มี
   `<div class="card-badge">` โผล่ออกมาเลย
2. **เพิ่ม variant `:outline` ให้ `ButtonComponent`** พร้อม CSS สีขอบ (ไม่มีพื้นหลัง) แล้วต่อยอด
   `button_component_controller.js` ให้แสดง native `confirm()` ก่อนยิง action จริงเมื่อปุ่มมี
   attribute `data-confirm-message` (คล้ายกับที่ `turbo_confirm:` ทำใน Part 024 Step 237 แต่คราวนี้
   เขียนเอง)
3. **แปลง partial `_pagination.html.erb`** จาก Part 024 Step 235 ให้เป็น `PaginationComponent`
   ที่รับ `current_page:`/`total_pages:` เป็น keyword argument แทนการรับผ่าน local variable ที่ไม่
   มีการตรวจสอบ แล้วเขียน unit test เทียบพฤติกรรมเดิมให้ครบทุกกรณี (หน้าแรก, หน้ากลาง, หน้าสุดท้าย)
4. **สร้าง `AccordionComponent`** ที่ใช้ `renders_many :panels` แต่ละ panel มี header/body ของ
   ตัวเอง ประกอบกับ Stimulus controller ที่ colocate อยู่ด้วยสำหรับ toggle เปิด/ปิด (เชื่อม Part 053
   Stimulus target/action เข้ากับ slot ของ Part นี้โดยตรง)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจข้อจำกัดเชิงโครงสร้างของ **partial** เมื่อโปรเจกต์ใหญ่ขึ้น: implicit interface, ทดสอบแยก
  ยาก, ไม่มีที่ทาง colocate asset, และปัญหา boolean flag ที่ล้นมือ
- ติดตั้งและใช้งาน **`view_component`** gem ได้โดยไม่ต้อง config เพิ่มเติม เข้าใจว่า
  `app/components/` ถูก autoload ให้อัตโนมัติเพราะเป็นโฟลเดอร์ใต้ `app/`
- Generate component ด้วย `bin/rails generate view_component:component` และเข้าใจกายวิภาคของไฟล์
  คู่ `.rb`/`.html.erb` (sidecar convention) พร้อม `initialize` ที่ทำหน้าที่เป็น explicit interface
- Render component จาก view ได้ทั้งสองรูปแบบ (`render(Component.new(...))` และแบบย่อ) และเข้าใจ
  `content` สำหรับรับเนื้อหาจาก block
- ใช้ **Slot API** (`renders_one`/`renders_many`, `with_xxx`, `xxx?`) แทนที่ boolean flag ที่
  ล้นมือ พร้อมเข้าใจ before/after ที่ชัดเจน
- Colocate **CSS/JS** ไว้กับ component (sidecar files) และตั้งค่า Propshaft + importmap ให้รู้จัก
  โฟลเดอร์ `app/components/` (พร้อม gotcha เรื่อง `pin_all_from ... to: ""` ที่ทดสอบเจอจริง)
- Preview component แบบแยกโดดด้วย `ViewComponent::Preview` ผ่านหน้า `/rails/view_components`
  โดยไม่ต้องพึ่ง route/controller จริงของแอป
- Unit test component ด้วย `render_inline` + RSpec matcher ของ Capybara (`have_css`) เร็วกว่า
  request/system test อย่างมีนัยสำคัญ เพราะข้ามชั้น routing/middleware ทั้งหมด
- **ประกอบ (compose) component เข้าด้วยกัน** ได้อย่างเป็นธรรมชาติ (`CardComponent` ใช้
  `ButtonComponent` ซ้อนอยู่ใน slot) พร้อมรู้จัก `vc_test_view_context` สำหรับกรณี composition
  ในเทสต์
- มีกรอบการตัดสินใจที่ชัดเจนว่าเมื่อไหร่ควรใช้ Partial, Helper, ViewComponent หรือ Stimulus
  Controller — และเข้าใจว่าทั้ง 4 เครื่องมือทำงานร่วมกันเป็นชั้นๆ ไม่ได้แข่งกัน

## สรุปภาพรวม Phase 7: Frontend / Hotwire

ยินดีด้วย! ตอนนี้ **Phase 7: Frontend / Hotwire (Part 051–055, Step 501–550)** เสร็จสมบูรณ์แล้ว
เราเดินทางผ่านเครื่องมือฝั่ง frontend ที่ทันสมัยที่สุดของ Rails ทีละชั้น:

- **Part 051 (Turbo Drive, Turbo Frames)** สอนให้หน้าเว็บนำทาง (navigate) แบบไม่ reload ทั้งหน้า
  และแบ่งส่วนของหน้าเป็น "กรอบ" อิสระที่อัปเดตแยกจากกันได้ โดยไม่ต้องเขียน JavaScript เอง
- **Part 052 (Turbo Streams)** ต่อยอดให้ server ส่งคำสั่งอัปเดต DOM หลายจุดพร้อมกันแบบ real-time
  ได้ (append, replace, remove) ผ่าน HTML ธรรมดาที่ส่งมาจาก server โดยตรง
- **Part 053 (Stimulus)** เติมเต็มส่วนที่ Turbo ทำให้ไม่ได้ — พฤติกรรมฝั่ง client ที่ต้องมี state
  หรือ interactivity เฉพาะจุด (controller, action, target, value)
- **Part 054 (Tailwind CSS/Bootstrap)** ให้เครื่องมือจัดสไตล์ที่ทำงานร่วมกับ Turbo/Stimulus ได้
  อย่างลงตัว ไม่ต้องเขียน CSS แยกไฟล์เยอะ
- **Part 055 (ViewComponent)** ปิดท้ายด้วยวิธีจัดการความซับซ้อนของ UI ที่ประกอบด้วยหลายชิ้นส่วน
  ให้เป็นระบบ testable และ maintainable

ภาพที่ชัดเจนที่สุดของ Phase นี้คือ **"HTML over the wire"** — ปรัชญาของ Hotwire (คำผสมจาก HTML +
Turbo + Native) ที่เชื่อว่าแอปเว็บส่วนใหญ่ไม่จำเป็นต้องมี JavaScript framework ฝั่ง client หนักๆ
(React/Vue/Angular) เลย เพียงแค่ให้ server ส่ง HTML ที่ถูกต้องออกมาในเวลาที่เหมาะสม (Turbo Frame/
Stream) แล้วเติมพฤติกรรมเล็กๆ น้อยๆ ที่จำเป็นจริงๆ ด้วย JavaScript ปริมาณน้อยที่สุด (Stimulus)
ก็เพียงพอสำหรับสร้างประสบการณ์ผู้ใช้ที่ลื่นไหลทัดเทียมกับ Single Page Application ได้ — และ
**ViewComponent คือชิ้นส่วนที่ทำให้ HTML เหล่านั้นถูกสร้างขึ้นมาอย่างเป็นระบบ** แทนที่จะกระจัด
กระจายอยู่ใน partial นับร้อยไฟล์ที่ไม่มีใครรู้ interface ที่แท้จริงของมัน

ทั้ง 5 Part นี้รวมกันคือ **เครื่องมือ frontend ครบชุดของ Rails สมัยใหม่** — ทีมพัฒนา Rails จำนวน
มากในปัจจุบัน (รวมถึง Basecamp/37signals ผู้สร้าง Hotwire และ GitHub ผู้สร้าง ViewComponent เอง)
สร้างผลิตภัณฑ์ระดับ production ด้วยชุดเครื่องมือนี้ทั้งหมด โดยไม่ต้องพึ่ง JavaScript framework
ฝั่ง client แยกต่างหากเลย

**ต่อไป (Part 056 — เปิด Phase 8: APIs & GraphQL):** เมื่อสร้าง UI ฝั่ง web ครบสมบูรณ์แล้ว
คำถามถัดไปคือทำอย่างไรให้แอปเดียวกันนี้**ให้บริการข้อมูลกับ client อื่นที่ไม่ใช่ browser** ได้ด้วย
เช่น mobile app หรือ third-party integration — Part 056 จะพาไปรู้จัก **Rails API-only mode**
(`rails new --api`), การตอบกลับข้อมูลเป็น **JSON response**, และเปรียบเทียบเครื่องมือ serializer
สองแบบหลักที่ใช้กันในวงการ: **ActiveModel::Serializer** กับ **Jbuilder** — จุดเริ่มต้นของ
**Phase 8: APIs & GraphQL** ที่จะพาไปสร้าง backend สำหรับยุคที่แอปหนึ่งตัวต้องคุยกับหลาย client
พร้อมกัน
