# Part 092: Capstone 1 — E-Commerce Platform: Cart, Checkout, Payment, Order

> **Step ครอบคลุมใน Part นี้:** Step 911–920

**ระดับ:** ขั้นสูง (ต้องผ่าน Part 091 ก่อน)
**Ruby on Rails เวอร์ชัน:** 7.1+
**Ruby เวอร์ชัน:** 3.2+
**เครื่องมือหลัก:** Stripe, Turbo Frames, Sidekiq, Action Mailer, AASM, Kamal

---

## บทนำ

ใน Part นี้เราจะสร้างส่วนที่สำคัญที่สุดของ E-Commerce platform นั่นคือ **กระบวนการซื้อสินค้าทั้งหมด** ตั้งแต่การเพิ่มสินค้าลงตะกร้าไปจนถึงการชำระเงินและการจัดส่ง

เนื้อหาใน Part นี้จะครอบคลุม:
- **Shopping Cart** แบบ hybrid (session + database) เพื่อรองรับทั้ง guest และ logged-in users
- **Turbo Frames** สำหรับ cart UI ที่ responsive ไม่ต้อง reload หน้า
- **Checkout Flow** หลายขั้นตอนด้วย Form Object pattern
- **Stripe Payment** พร้อม webhook สำหรับยืนยันการชำระเงิน
- **Order State Machine** ด้วย AASM gem
- **Email Notifications** async ด้วย Sidekiq
- **Deploy** ไปยัง production ด้วย Kamal

เมื่อจบ Part นี้คุณจะมี E-Commerce platform ที่ทำงานได้จริงพร้อม deploy!

---

## สารบัญ

- [Step 911: Shopping Cart — Session vs Database Cart](#step-911)
- [Step 912: Cart UI ด้วย Turbo Frames](#step-912)
- [Step 913: Checkout Flow — Multi-step Form Object](#step-913)
- [Step 914: Stripe Checkout Integration](#step-914)
- [Step 915: Order Lifecycle ด้วย State Machine](#step-915)
- [Step 916: Email Notifications ด้วย Action Mailer](#step-916)
- [Step 917: Seller Dashboard](#step-917)
- [Step 918: Product Reviews ด้วย Polymorphic](#step-918)
- [Step 919: Background Jobs ด้วย Sidekiq](#step-919)
- [Step 920: Deploy E-Commerce ด้วย Kamal](#step-920)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุป)
- [ก้าวต่อไป](#ถัดไป)

---

## Step 911: Shopping Cart — Session-based vs Database Cart {#step-911}

### ความแตกต่างระหว่าง Session Cart และ Database Cart

```
SESSION CART:
- เก็บข้อมูลใน browser session (cookie)
- ไม่ต้อง login
- หมดอายุเมื่อ browser ปิด (หรือตาม session timeout)
- ไม่ sync ข้ามอุปกรณ์

DATABASE CART:
- เก็บใน CartItem table ในฐานข้อมูล
- ต้อง login
- คงอยู่ข้ามอุปกรณ์
- ดู analytics ได้

HYBRID APPROACH (ที่เราจะใช้):
- Guest → session cart
- หลัง login → merge session cart เข้า database cart
- ดีที่สุดทั้งสองแบบ
```

### CartItem Model

```bash
rails generate model CartItem \
  user:references \
  product:references \
  quantity:integer
```

```ruby
# db/migrate/XXXXXX_create_cart_items.rb
class CreateCartItems < ActiveRecord::Migration[7.1]
  def change
    create_table :cart_items do |t|
      t.references :user, null: true, foreign_key: true  # null สำหรับ guest
      t.references :product, null: false, foreign_key: true
      t.integer :quantity, null: false, default: 1
      t.string :session_id  # สำหรับ guest cart

      t.timestamps
    end

    add_index :cart_items, [:user_id, :product_id], unique: true, where: "user_id IS NOT NULL"
    add_index :cart_items, [:session_id, :product_id], unique: true, where: "session_id IS NOT NULL"
  end
end
```

```ruby
# app/models/cart_item.rb
class CartItem < ApplicationRecord
  belongs_to :user, optional: true
  belongs_to :product

  validates :quantity, numericality: { greater_than: 0, less_than_or_equal_to: 99 }
  validates :product_id, uniqueness: { scope: :user_id, message: "มีในตะกร้าแล้ว" },
            if: :user_id?

  # Scopes
  scope :for_user, ->(user) { where(user: user) }
  scope :for_session, ->(session_id) { where(session_id: session_id, user_id: nil) }

  # Instance methods
  def subtotal
    product.price * quantity
  end

  def increment!(amount = 1)
    update!(quantity: [quantity + amount, 99].min)
  end

  def decrement!(amount = 1)
    new_qty = quantity - amount
    new_qty <= 0 ? destroy! : update!(quantity: new_qty)
  end
end
```

### Cart Service Object

```ruby
# app/services/cart_service.rb
class CartService
  attr_reader :user, :session

  def initialize(user: nil, session: nil)
    @user = user
    @session = session
  end

  # ดึง cart items ตาม context
  def items
    if user
      CartItem.for_user(user).includes(:product)
    else
      CartItem.for_session(session_id).includes(:product)
    end
  end

  # เพิ่มสินค้าลงตะกร้า
  def add_item(product_id:, quantity: 1)
    product = Product.published.find(product_id)
    raise "สินค้าหมด" unless product.in_stock?
    raise "จำนวนเกินสต็อก" if quantity > product.stock_quantity

    cart_item = find_or_initialize_item(product_id)

    if cart_item.persisted?
      cart_item.increment!(quantity)
    else
      cart_item.quantity = quantity
      cart_item.save!
    end

    cart_item
  end

  # อัปเดตจำนวน
  def update_item(cart_item_id:, quantity:)
    item = find_item(cart_item_id)
    if quantity <= 0
      item.destroy!
    else
      item.update!(quantity: quantity)
    end
  end

  # ลบสินค้า
  def remove_item(cart_item_id)
    find_item(cart_item_id).destroy!
  end

  # ยอดรวมทั้งหมด
  def total
    items.sum(&:subtotal)
  end

  # จำนวนสินค้าในตะกร้า
  def count
    items.sum(:quantity)
  end

  # Merge session cart เข้า user cart หลัง login
  def merge_session_cart_to_user!(target_user)
    return unless session_id.present?

    CartItem.for_session(session_id).each do |session_item|
      existing = CartItem.find_by(user: target_user, product_id: session_item.product_id)
      if existing
        existing.update!(quantity: existing.quantity + session_item.quantity)
        session_item.destroy!
      else
        session_item.update!(user: target_user, session_id: nil)
      end
    end
  end

  private

  def session_id
    session[:cart_session_id] ||= SecureRandom.hex(16)
  end

  def find_or_initialize_item(product_id)
    if user
      CartItem.find_or_initialize_by(user: user, product_id: product_id)
    else
      CartItem.find_or_initialize_by(session_id: session_id, product_id: product_id)
    end
  end

  def find_item(id)
    if user
      CartItem.for_user(user).find(id)
    else
      CartItem.for_session(session_id).find(id)
    end
  end
end
```

### CartItems Controller

```ruby
# app/controllers/cart_items_controller.rb
class CartItemsController < ApplicationController
  before_action :set_cart_service

  def create
    @cart_service.add_item(
      product_id: params[:product_id],
      quantity: params[:quantity].to_i
    )

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to cart_path }
    end
  rescue => e
    redirect_to product_path(params[:product_id]), alert: e.message
  end

  def update
    @cart_service.update_item(
      cart_item_id: params[:id],
      quantity: params[:quantity].to_i
    )

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to cart_path }
    end
  end

  def destroy
    @cart_service.remove_item(params[:id])

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to cart_path }
    end
  end

  private

  def set_cart_service
    @cart_service = CartService.new(user: current_user, session: session)
  end
end
```

---

## Step 912: Cart UI ด้วย Turbo Frames — Realtime Update {#step-912}

Turbo Frames ทำให้เราอัปเดตเฉพาะส่วนของหน้าได้ โดยไม่ต้อง reload ทั้งหน้า

### Cart View

```html
<!-- app/views/carts/show.html.erb -->
<div class="max-w-4xl mx-auto">
  <h1 class="text-2xl font-bold mb-6">ตะกร้าสินค้า</h1>

  <%= turbo_frame_tag "cart" do %>
    <% if @cart_items.any? %>
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">

        <!-- Cart Items List -->
        <div class="lg:col-span-2 space-y-4">
          <% @cart_items.each do |item| %>
            <%= render "cart_items/item", item: item %>
          <% end %>
        </div>

        <!-- Order Summary -->
        <div class="bg-white rounded-lg shadow p-5 h-fit">
          <h2 class="font-bold text-lg mb-4">สรุปคำสั่งซื้อ</h2>

          <div class="space-y-2 text-sm">
            <div class="flex justify-between">
              <span class="text-gray-600">ราคาสินค้า</span>
              <span>฿<%= number_with_delimiter(@subtotal.to_i) %></span>
            </div>
            <div class="flex justify-between">
              <span class="text-gray-600">ค่าจัดส่ง</span>
              <span class="text-green-600">ฟรี</span>
            </div>
            <div class="border-t pt-2 flex justify-between font-bold text-lg">
              <span>รวมทั้งหมด</span>
              <span class="text-red-600">฿<%= number_with_delimiter(@subtotal.to_i) %></span>
            </div>
          </div>

          <%= link_to "ดำเนินการชำระเงิน",
                new_checkout_path,
                class: "mt-4 block w-full text-center bg-orange-500 text-white py-3 rounded-lg font-bold hover:bg-orange-600" %>

          <%= link_to "ช้อปต่อ", products_path,
                class: "mt-2 block w-full text-center border border-gray-300 py-2 rounded-lg text-sm hover:bg-gray-50" %>
        </div>
      </div>

    <% else %>
      <div class="text-center py-16">
        <p class="text-4xl mb-4">🛒</p>
        <p class="text-xl font-medium text-gray-600 mb-2">ตะกร้าว่างเปล่า</p>
        <p class="text-gray-400 mb-6">เพิ่มสินค้าที่คุณสนใจลงในตะกร้า</p>
        <%= link_to "เริ่มช้อปเลย", products_path,
              class: "bg-blue-600 text-white px-6 py-3 rounded-lg hover:bg-blue-700" %>
      </div>
    <% end %>
  <% end %>
</div>
```

```html
<!-- app/views/cart_items/_item.html.erb -->
<%= turbo_frame_tag dom_id(item) do %>
  <div class="bg-white rounded-lg shadow p-4 flex gap-4">
    <!-- รูปสินค้า -->
    <div class="w-24 h-24 rounded overflow-hidden bg-gray-100 shrink-0">
      <% if item.product.images.attached? %>
        <%= image_tag item.product.primary_image.variant(resize_to_fill: [96, 96]),
              class: "w-full h-full object-cover" %>
      <% end %>
    </div>

    <!-- รายละเอียด -->
    <div class="flex-1 min-w-0">
      <h3 class="font-medium line-clamp-2">
        <%= link_to item.product.name, product_path(item.product),
              class: "hover:text-blue-600" %>
      </h3>
      <p class="text-sm text-gray-500">ร้าน <%= item.product.seller.display_name %></p>
      <p class="text-red-600 font-bold mt-1">฿<%= number_with_delimiter(item.product.price) %></p>
    </div>

    <!-- ควบคุมจำนวน -->
    <div class="flex flex-col items-end gap-2">
      <%= form_with url: cart_item_path(item),
                    method: :patch,
                    data: { turbo_frame: dom_id(item) } do |f| %>
        <div class="flex items-center border rounded overflow-hidden">
          <button type="button"
                  onclick="let q=this.nextElementSibling; q.value=Math.max(1,parseInt(q.value)-1); this.closest('form').requestSubmit()"
                  class="px-2 py-1 bg-gray-100 hover:bg-gray-200 text-sm">−</button>
          <%= f.number_field :quantity, value: item.quantity, min: 1, max: 99,
                class: "w-12 text-center py-1 border-x text-sm",
                onchange: "this.closest('form').requestSubmit()" %>
          <button type="button"
                  onclick="let q=this.previousElementSibling; q.value=Math.min(99,parseInt(q.value)+1); this.closest('form').requestSubmit()"
                  class="px-2 py-1 bg-gray-100 hover:bg-gray-200 text-sm">+</button>
        </div>
      <% end %>

      <p class="font-semibold text-sm">
        ฿<%= number_with_delimiter(item.subtotal.to_i) %>
      </p>

      <%= button_to "ลบ", cart_item_path(item),
            method: :delete,
            data: { turbo_frame: "cart", confirm: "ลบสินค้านี้ออกจากตะกร้า?" },
            class: "text-xs text-red-500 hover:text-red-700" %>
    </div>
  </div>
<% end %>
```

### Turbo Stream Response สำหรับ cart update

```html
<!-- app/views/cart_items/create.turbo_stream.erb -->
<%= turbo_stream.replace "cart" do %>
  <%= render "carts/cart_content", cart_items: @cart_service.items, subtotal: @cart_service.total %>
<% end %>

<%= turbo_stream.replace "cart_count" do %>
  <span id="cart_count" class="bg-red-500 text-white text-xs rounded-full px-1.5 py-0.5">
    <%= @cart_service.count %>
  </span>
<% end %>
```

```html
<!-- app/views/cart_items/destroy.turbo_stream.erb -->
<%= turbo_stream.remove dom_id(@cart_item) %>
<%= turbo_stream.replace "cart_summary" do %>
  <%= render "carts/summary", subtotal: @cart_service.total %>
<% end %>
```

---

## Step 913: Checkout Flow — Multi-step Form ด้วย Form Object {#step-913}

Checkout flow มักมีหลายขั้นตอน เราจะใช้ **Form Object pattern** เพื่อจัดการ validation ในแต่ละ step

### ขั้นตอน Checkout

```
Step 1: ที่อยู่จัดส่ง (Shipping Address)
Step 2: ยืนยันคำสั่งซื้อและเลือกวิธีชำระเงิน
Step 3: (Stripe redirect) → ชำระเงิน
Step 4: หน้า Thank You (หลัง Stripe callback)
```

### Checkout Form Objects

```ruby
# app/forms/checkout/address_form.rb
module Checkout
  class AddressForm
    include ActiveModel::Model
    include ActiveModel::Attributes

    attribute :full_name, :string
    attribute :phone, :string
    attribute :address_line1, :string
    attribute :address_line2, :string
    attribute :city, :string
    attribute :province, :string
    attribute :postal_code, :string

    validates :full_name, presence: true, length: { maximum: 100 }
    validates :phone, presence: true,
              format: { with: /\A0[0-9]{8,9}\z/, message: "ต้องเป็นเบอร์โทรศัพท์ไทย" }
    validates :address_line1, presence: true, length: { maximum: 200 }
    validates :city, presence: true
    validates :province, presence: true
    validates :postal_code, presence: true,
              format: { with: /\A[0-9]{5}\z/, message: "ต้องเป็นรหัสไปรษณีย์ 5 หลัก" }

    PROVINCES = [
      "กรุงเทพมหานคร", "กระบี่", "กาญจนบุรี", "กาฬสินธุ์", "กำแพงเพชร",
      "ขอนแก่น", "จันทบุรี", "ฉะเชิงเทรา", "ชลบุรี", "ชัยนาท",
      "ชัยภูมิ", "ชุมพร", "เชียงราย", "เชียงใหม่", "ตรัง",
      "ตราด", "ตาก", "นครนายก", "นครปฐม", "นครพนม",
      "นครราชสีมา", "นครศรีธรรมราช", "นครสวรรค์", "นนทบุรี", "นราธิวาส",
      "น่าน", "บึงกาฬ", "บึงกาฬ", "ปทุมธานี", "ประจวบคีรีขันธ์",
      "ปราจีนบุรี", "ปัตตานี", "พระนครศรีอยุธยา", "พะเยา", "พังงา",
      "พัทลุง", "พิจิตร", "พิษณุโลก", "เพชรบุรี", "เพชรบูรณ์",
      "แพร่", "ภูเก็ต", "มหาสารคาม", "มุกดาหาร", "แม่ฮ่องสอน",
      "ยโสธร", "ยะลา", "ร้อยเอ็ด", "ระนอง", "ระยอง",
      "ราชบุรี", "ลพบุรี", "ลำปาง", "ลำพูน", "เลย",
      "ศรีสะเกษ", "สกลนคร", "สงขลา", "สตูล", "สมุทรปราการ",
      "สมุทรสงคราม", "สมุทรสาคร", "สระแก้ว", "สระบุรี", "สิงห์บุรี",
      "สุโขทัย", "สุพรรณบุรี", "สุราษฎร์ธานี", "สุรินทร์", "หนองคาย",
      "หนองบัวลำภู", "อ่างทอง", "อำนาจเจริญ", "อุดรธานี", "อุตรดิตถ์",
      "อุทัยธานี", "อุบลราชธานี"
    ].freeze

    def to_json_snapshot
      {
        full_name: full_name,
        phone: phone,
        address_line1: address_line1,
        address_line2: address_line2,
        city: city,
        province: province,
        postal_code: postal_code
      }
    end
  end
end
```

### Checkouts Controller

```ruby
# app/controllers/checkouts_controller.rb
class CheckoutsController < ApplicationController
  before_action :authenticate_user!
  before_action :ensure_cart_not_empty

  def new
    @step = params[:step]&.to_i || 1
    @address_form = Checkout::AddressForm.new(session[:checkout_address] || {})
    @cart_items = current_cart_service.items
    @total = current_cart_service.total
  end

  def create
    @step = params[:step].to_i

    case @step
    when 1
      handle_address_step
    when 2
      handle_payment_step
    else
      redirect_to new_checkout_path
    end
  end

  private

  def handle_address_step
    @address_form = Checkout::AddressForm.new(address_params)

    if @address_form.valid?
      session[:checkout_address] = @address_form.to_json_snapshot
      redirect_to new_checkout_path(step: 2)
    else
      @cart_items = current_cart_service.items
      @total = current_cart_service.total
      @step = 1
      render :new, status: :unprocessable_entity
    end
  end

  def handle_payment_step
    address_data = session[:checkout_address]
    redirect_to new_checkout_path(step: 1), alert: "กรุณากรอกที่อยู่ก่อน" unless address_data

    # สร้าง Order
    order = OrderCreationService.new(
      user: current_user,
      cart_service: current_cart_service,
      address: address_data
    ).call

    session.delete(:checkout_address)

    # Redirect ไป Stripe
    stripe_session = StripeService.create_checkout_session(order: order, return_url: request.base_url)
    redirect_to stripe_session.url, allow_other_host: true
  end

  def address_params
    params.require(:checkout_address_form).permit(
      :full_name, :phone, :address_line1, :address_line2,
      :city, :province, :postal_code
    )
  end

  def ensure_cart_not_empty
    redirect_to cart_path, alert: "ตะกร้าว่างเปล่า" if current_cart_service.count == 0
  end

  def current_cart_service
    @current_cart_service ||= CartService.new(user: current_user, session: session)
  end
end
```

### Checkout View Step 1 — ที่อยู่

```html
<!-- app/views/checkouts/new.html.erb -->
<!-- Progress Bar -->
<div class="max-w-2xl mx-auto mb-8">
  <div class="flex items-center">
    <% [["1", "ที่อยู่"], ["2", "ยืนยันคำสั่งซื้อ"], ["3", "ชำระเงิน"]].each_with_index do |(num, label), i| %>
      <div class="flex items-center <%= i > 0 ? 'flex-1' : '' %>">
        <% if i > 0 %>
          <div class="flex-1 h-1 <%= @step > i ? 'bg-blue-500' : 'bg-gray-200' %>"></div>
        <% end %>
        <div class="w-8 h-8 rounded-full flex items-center justify-center text-sm font-bold
                    <%= @step >= num.to_i ? 'bg-blue-600 text-white' : 'bg-gray-200 text-gray-500' %>">
          <%= num %>
        </div>
        <span class="ml-2 text-sm <%= @step >= num.to_i ? 'text-blue-600 font-medium' : 'text-gray-400' %>">
          <%= label %>
        </span>
      </div>
    <% end %>
  </div>
</div>

<% if @step == 1 %>
  <div class="max-w-2xl mx-auto bg-white rounded-lg shadow p-6">
    <h2 class="text-xl font-bold mb-6">ที่อยู่จัดส่ง</h2>

    <%= form_with model: @address_form,
                  url: checkouts_path,
                  class: "space-y-4" do |f| %>
      <%= hidden_field_tag :step, 1 %>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <%= f.label :full_name, "ชื่อ-นามสกุล", class: "block text-sm font-medium mb-1" %>
          <%= f.text_field :full_name, class: "w-full border rounded-lg px-3 py-2 focus:ring-2 focus:ring-blue-500" %>
          <% if @address_form.errors[:full_name].any? %>
            <p class="text-red-500 text-xs mt-1"><%= @address_form.errors[:full_name].first %></p>
          <% end %>
        </div>

        <div>
          <%= f.label :phone, "เบอร์โทรศัพท์", class: "block text-sm font-medium mb-1" %>
          <%= f.text_field :phone, placeholder: "0812345678", class: "w-full border rounded-lg px-3 py-2" %>
          <% if @address_form.errors[:phone].any? %>
            <p class="text-red-500 text-xs mt-1"><%= @address_form.errors[:phone].first %></p>
          <% end %>
        </div>
      </div>

      <div>
        <%= f.label :address_line1, "ที่อยู่", class: "block text-sm font-medium mb-1" %>
        <%= f.text_field :address_line1, placeholder: "บ้านเลขที่ ถนน ซอย", class: "w-full border rounded-lg px-3 py-2" %>
      </div>

      <div>
        <%= f.label :address_line2, "ตำบล/แขวง และ อำเภอ/เขต (ถ้ามี)", class: "block text-sm font-medium mb-1" %>
        <%= f.text_field :address_line2, class: "w-full border rounded-lg px-3 py-2" %>
      </div>

      <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
        <div>
          <%= f.label :city, "เมือง/อำเภอ", class: "block text-sm font-medium mb-1" %>
          <%= f.text_field :city, class: "w-full border rounded-lg px-3 py-2" %>
        </div>

        <div>
          <%= f.label :province, "จังหวัด", class: "block text-sm font-medium mb-1" %>
          <%= f.select :province,
                options_for_select(Checkout::AddressForm::PROVINCES, @address_form.province),
                { prompt: "เลือกจังหวัด" },
                class: "w-full border rounded-lg px-3 py-2" %>
        </div>

        <div>
          <%= f.label :postal_code, "รหัสไปรษณีย์", class: "block text-sm font-medium mb-1" %>
          <%= f.text_field :postal_code, placeholder: "10110", maxlength: 5,
                class: "w-full border rounded-lg px-3 py-2" %>
        </div>
      </div>

      <%= f.submit "ดำเนินการต่อ →",
            class: "w-full bg-blue-600 text-white py-3 rounded-lg font-bold hover:bg-blue-700 cursor-pointer" %>
    <% end %>
  </div>
<% elsif @step == 2 %>
  <!-- Step 2: ยืนยันคำสั่งซื้อ -->
  <!-- ... order summary + confirm button ... -->
<% end %>
```

---

## Step 914: Stripe Checkout Integration {#step-914}

### ติดตั้ง Stripe Gem

```bash
bundle add stripe
```

### ตั้งค่า Stripe

```ruby
# config/initializers/stripe.rb
Stripe.api_key = ENV["STRIPE_SECRET_KEY"]
```

```
# .env (ใช้ dotenv หรือ Rails credentials)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

### StripeService

```ruby
# app/services/stripe_service.rb
class StripeService
  def self.create_checkout_session(order:, return_url:)
    line_items = order.order_items.includes(:product).map do |item|
      {
        price_data: {
          currency: "thb",
          product_data: {
            name: item.product.name,
            description: item.product.description&.truncate(200)
          },
          unit_amount: (item.unit_price * 100).to_i  # Stripe ใช้ satang
        },
        quantity: item.quantity
      }
    end

    Stripe::Checkout::Session.create({
      payment_method_types: ["card", "promptpay"],
      line_items: line_items,
      mode: "payment",
      success_url: "#{return_url}/orders/#{order.id}?payment=success",
      cancel_url: "#{return_url}/cart",
      metadata: {
        order_id: order.id,
        user_id: order.user_id
      },
      customer_email: order.user.email,
      payment_intent_data: {
        metadata: { order_id: order.id }
      }
    })
  end
end
```

### Webhook Controller

```ruby
# app/controllers/webhooks_controller.rb
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token
  skip_before_action :authenticate_user!, raise: false

  def stripe
    payload = request.body.read
    sig_header = request.env["HTTP_STRIPE_SIGNATURE"]
    endpoint_secret = ENV["STRIPE_WEBHOOK_SECRET"]

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, endpoint_secret)
    rescue JSON::ParserError, Stripe::SignatureVerificationError => e
      render json: { error: e.message }, status: :bad_request
      return
    end

    case event.type
    when "checkout.session.completed"
      handle_checkout_completed(event.data.object)
    when "payment_intent.payment_failed"
      handle_payment_failed(event.data.object)
    end

    render json: { received: true }
  end

  private

  def handle_checkout_completed(session)
    order_id = session.metadata["order_id"]
    order = Order.find_by(id: order_id)
    return unless order

    order.pay!(stripe_session_id: session.id)
    OrderConfirmationMailerJob.perform_later(order.id)
  end

  def handle_payment_failed(payment_intent)
    order_id = payment_intent.metadata["order_id"]
    order = Order.find_by(id: order_id)
    order&.fail_payment!
  end
end
```

### Order Creation Service

```ruby
# app/services/order_creation_service.rb
class OrderCreationService
  def initialize(user:, cart_service:, address:)
    @user = user
    @cart_service = cart_service
    @address = address
  end

  def call
    ActiveRecord::Base.transaction do
      order = create_order
      create_order_items(order)
      clear_cart
      order
    end
  end

  private

  def create_order
    Order.create!(
      user: @user,
      status: :pending,
      total_price: @cart_service.total,
      shipping_address: @address.to_json,
      order_number: generate_order_number
    )
  end

  def create_order_items(order)
    @cart_service.items.each do |cart_item|
      product = cart_item.product
      # ลด stock ทันที (lock เพื่อป้องกัน race condition)
      product.with_lock do
        raise "สินค้า '#{product.name}' มีสต็อกไม่พอ" if product.stock_quantity < cart_item.quantity
        product.decrement!(:stock_quantity, cart_item.quantity)
      end

      order.order_items.create!(
        product: product,
        quantity: cart_item.quantity,
        unit_price: product.price,
        product_snapshot: {
          name: product.name,
          price: product.price,
          description: product.description
        }.to_json
      )
    end
  end

  def clear_cart
    @cart_service.items.destroy_all
  end

  def generate_order_number
    "TH-#{Time.zone.now.strftime('%Y%m%d')}-#{SecureRandom.hex(4).upcase}"
  end
end
```

---

## Step 915: Order Lifecycle ด้วย State Machine {#step-915}

### Order Model พร้อม AASM

```bash
rails generate model Order \
  user:references \
  status:integer \
  total_price:decimal \
  shipping_address:text \
  order_number:string
```

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  include AASM

  belongs_to :user
  has_many :order_items, dependent: :destroy
  has_one :payment, dependent: :destroy
  has_many :products, through: :order_items

  validates :order_number, presence: true, uniqueness: true
  validates :total_price, numericality: { greater_than: 0 }

  # State Machine ด้วย AASM
  aasm column: :status, enum: true do
    state :pending, initial: true     # รอชำระเงิน
    state :paid                        # ชำระเงินแล้ว
    state :processing                  # กำลังเตรียมสินค้า
    state :shipped                     # จัดส่งแล้ว
    state :delivered                   # ส่งถึงแล้ว
    state :cancelled                   # ยกเลิก
    state :refunded                    # คืนเงินแล้ว

    # Transitions
    event :pay do
      transitions from: :pending, to: :paid
      after do |stripe_session_id: nil|
        create_payment!(
          stripe_session_id: stripe_session_id,
          amount: total_price,
          paid_at: Time.current
        )
      end
    end

    event :start_processing do
      transitions from: :paid, to: :processing
    end

    event :ship do
      transitions from: :processing, to: :shipped
      after { |tracking_number: nil| update!(tracking_number: tracking_number) }
    end

    event :deliver do
      transitions from: :shipped, to: :delivered
      after { update!(delivered_at: Time.current) }
    end

    event :cancel do
      transitions from: [:pending, :paid], to: :cancelled
      after { restore_stock! }
    end

    event :refund do
      transitions from: [:paid, :processing, :shipped], to: :refunded
      after { process_refund! }
    end

    event :fail_payment do
      transitions from: :pending, to: :cancelled
    end
  end

  # Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :for_seller, ->(seller) {
    joins(order_items: :product).where(products: { seller_id: seller.id }).distinct
  }

  def shipping_address_hash
    JSON.parse(shipping_address) rescue {}
  end

  def formatted_address
    addr = shipping_address_hash
    [addr["address_line1"], addr["address_line2"], addr["city"], addr["province"], addr["postal_code"]]
      .compact_blank.join(", ")
  end

  def status_label_th
    {
      "pending"     => "รอชำระเงิน",
      "paid"        => "ชำระเงินแล้ว",
      "processing"  => "กำลังเตรียมสินค้า",
      "shipped"     => "จัดส่งแล้ว",
      "delivered"   => "ส่งถึงแล้ว",
      "cancelled"   => "ยกเลิก",
      "refunded"    => "คืนเงินแล้ว"
    }[status.to_s] || status
  end

  def status_color
    {
      "pending"     => "yellow",
      "paid"        => "blue",
      "processing"  => "indigo",
      "shipped"     => "purple",
      "delivered"   => "green",
      "cancelled"   => "red",
      "refunded"    => "gray"
    }[status.to_s] || "gray"
  end

  private

  def restore_stock!
    order_items.each do |item|
      item.product.increment!(:stock_quantity, item.quantity)
    end
  end

  def process_refund!
    # Stripe refund logic
    if payment&.stripe_session_id
      Stripe::Refund.create(payment_intent: payment.stripe_payment_intent_id)
    end
  end
end
```

### Orders Controller

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  before_action :authenticate_user!

  def index
    @pagy, @orders = pagy(
      current_user.orders.includes(order_items: :product).recent,
      limit: 10
    )
  end

  def show
    @order = current_user.orders.find(params[:id])
    flash.now[:success] = "ชำระเงินสำเร็จ! ขอบคุณที่ใช้บริการ" if params[:payment] == "success"
  end
end
```

---

## Step 916: Email Notifications ด้วย Action Mailer {#step-916}

### สร้าง Order Mailer

```bash
rails generate mailer OrderMailer order_confirmation shipping_update
```

```ruby
# app/mailers/order_mailer.rb
class OrderMailer < ApplicationMailer
  default from: "noreply@ecommerce.th"

  def order_confirmation(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "ยืนยันคำสั่งซื้อ ##{@order.order_number} - ขอบคุณที่ช้อปกับเรา!"
    )
  end

  def shipping_update(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "สินค้าของคุณถูกจัดส่งแล้ว! Order ##{@order.order_number}"
    )
  end

  def order_cancelled(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "คำสั่งซื้อ ##{@order.order_number} ถูกยกเลิก"
    )
  end
end
```

### Email Templates

```html
<!-- app/views/order_mailer/order_confirmation.html.erb -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: 'Sarabun', sans-serif; color: #333; }
    .container { max-width: 600px; margin: 0 auto; }
    .header { background: #1e40af; color: white; padding: 20px; text-align: center; }
    .content { padding: 20px; }
    .order-table { width: 100%; border-collapse: collapse; }
    .order-table th, .order-table td { padding: 10px; border-bottom: 1px solid #eee; text-align: left; }
    .total { font-size: 18px; font-weight: bold; color: #dc2626; }
    .footer { background: #f3f4f6; padding: 15px; text-align: center; font-size: 12px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>ขอบคุณสำหรับคำสั่งซื้อ!</h1>
      <p>Order #<%= @order.order_number %></p>
    </div>

    <div class="content">
      <p>สวัสดีคุณ <%= @user.full_name %>,</p>
      <p>เราได้รับคำสั่งซื้อของคุณเรียบร้อยแล้ว</p>

      <h3>รายการสินค้า</h3>
      <table class="order-table">
        <thead>
          <tr>
            <th>สินค้า</th>
            <th>จำนวน</th>
            <th>ราคา</th>
          </tr>
        </thead>
        <tbody>
          <% @order.order_items.each do |item| %>
            <tr>
              <td><%= item.product.name %></td>
              <td><%= item.quantity %> ชิ้น</td>
              <td>฿<%= number_with_delimiter(item.unit_price) %></td>
            </tr>
          <% end %>
        </tbody>
        <tfoot>
          <tr>
            <td colspan="2"><strong>รวมทั้งหมด</strong></td>
            <td class="total">฿<%= number_with_delimiter(@order.total_price) %></td>
          </tr>
        </tfoot>
      </table>

      <h3>ที่อยู่จัดส่ง</h3>
      <p><%= @order.formatted_address %></p>

      <p>
        <%= link_to "ดูรายละเอียดคำสั่งซื้อ",
              order_url(@order),
              style: "background: #1e40af; color: white; padding: 12px 24px; text-decoration: none; border-radius: 6px;" %>
      </p>
    </div>

    <div class="footer">
      <p>© 2024 ร้านค้าออนไลน์ | <a href="<%= root_url %>">เยี่ยมชมร้านค้า</a></p>
    </div>
  </div>
</body>
</html>
```

---

## Step 917: Seller Dashboard {#step-917}

### Seller Dashboard Controller

```ruby
# app/controllers/seller/dashboard_controller.rb
class Seller::DashboardController < Seller::BaseController
  def index
    @stats = calculate_stats
    @recent_orders = recent_orders_query.limit(5)
    @top_products = top_selling_products
    @monthly_revenue = monthly_revenue_data
  end

  private

  def calculate_stats
    my_products = current_user.products

    {
      total_products: my_products.count,
      published_products: my_products.published.count,
      total_orders: orders_for_seller.count,
      pending_orders: orders_for_seller.pending.count,
      total_revenue: orders_for_seller.paid.sum(:total_price),
      this_month_revenue: orders_for_seller.paid
                            .where(created_at: Time.zone.now.beginning_of_month..)
                            .sum(:total_price)
    }
  end

  def orders_for_seller
    @orders_for_seller ||= Order.for_seller(current_user)
  end

  def recent_orders_query
    orders_for_seller.includes(:user, order_items: :product).recent
  end

  def top_selling_products
    current_user.products
               .joins(:order_items)
               .select("products.*, SUM(order_items.quantity) as total_sold")
               .group("products.id")
               .order("total_sold DESC")
               .limit(5)
  end

  def monthly_revenue_data
    orders_for_seller
      .paid
      .where(created_at: 6.months.ago..)
      .group("DATE_TRUNC('month', created_at)")
      .sum(:total_price)
      .transform_keys { |k| k.strftime("%b %Y") }
  end
end
```

### Seller Products Controller

```ruby
# app/controllers/seller/products_controller.rb
class Seller::ProductsController < Seller::BaseController
  before_action :set_product, only: [:show, :edit, :update, :destroy, :publish, :archive]

  def index
    @pagy, @products = pagy(
      current_user.products.includes(:category).order(created_at: :desc),
      limit: 15
    )
  end

  def new
    @product = current_user.products.build
  end

  def create
    @product = current_user.products.build(product_params)

    if @product.save
      redirect_to seller_product_path(@product), notice: "เพิ่มสินค้าสำเร็จ"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @product.update(product_params)
      redirect_to seller_product_path(@product), notice: "อัปเดตสินค้าสำเร็จ"
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def publish
    @product.update!(status: :published)
    redirect_to seller_products_path, notice: "เผยแพร่สินค้าแล้ว"
  end

  def destroy
    @product.destroy!
    redirect_to seller_products_path, notice: "ลบสินค้าแล้ว"
  end

  private

  def set_product
    @product = current_user.products.find(params[:id])
  end

  def product_params
    params.require(:product).permit(
      :name, :description, :price, :stock_quantity, :category_id, :status,
      images: []
    )
  end
end
```

---

## Step 918: Product Reviews — Polymorphic และ Counter Cache {#step-918}

### Review Model

```bash
rails generate model Review \
  user:references \
  product:references \
  rating:integer \
  body:text \
  order_item:references
```

```ruby
# app/models/review.rb
class Review < ApplicationRecord
  belongs_to :user
  belongs_to :product, counter_cache: :reviews_count
  belongs_to :order_item, optional: true

  validates :rating, presence: true,
            numericality: { only_integer: true, in: 1..5 }
  validates :body, length: { maximum: 1000 }
  validates :user_id, uniqueness: {
    scope: :product_id,
    message: "คุณเคยรีวิวสินค้านี้แล้ว"
  }
  validate :user_must_have_purchased

  # Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :with_rating, ->(r) { where(rating: r) }
  scope :verified, -> { where.not(order_item_id: nil) }

  after_create  :update_product_average_rating
  after_update  :update_product_average_rating
  after_destroy :update_product_average_rating

  def rating_stars
    "★" * rating + "☆" * (5 - rating)
  end

  private

  def user_must_have_purchased
    return if order_item_id.present?

    purchased = OrderItem.joins(:order)
                         .where(orders: { user_id: user_id, status: :delivered })
                         .where(product_id: product_id)
                         .exists?

    errors.add(:base, "คุณต้องซื้อและรับสินค้านี้ก่อนจึงจะรีวิวได้") unless purchased
  end

  def update_product_average_rating
    product.update_average_rating!
  end
end
```

### Reviews Controller

```ruby
# app/controllers/reviews_controller.rb
class ReviewsController < ApplicationController
  before_action :authenticate_user!

  def create
    @product = Product.find(params[:product_id])
    @review = @product.reviews.build(review_params)
    @review.user = current_user

    if @review.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to product_path(@product), notice: "รีวิวของคุณถูกบันทึกแล้ว" }
      end
    else
      redirect_to product_path(@product), alert: @review.errors.full_messages.to_sentence
    end
  end

  def destroy
    @review = current_user.reviews.find(params[:id])
    @product = @review.product
    @review.destroy!
    redirect_to product_path(@product), notice: "ลบรีวิวแล้ว"
  end

  private

  def review_params
    params.require(:review).permit(:rating, :body, :order_item_id)
  end
end
```

### Counter Cache Migration

```ruby
# db/migrate/XXXXXX_add_reviews_count_to_products.rb
class AddReviewsCountToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :reviews_count, :integer, default: 0, null: false
  end
end
```

---

## Step 919: Background Jobs ด้วย Sidekiq {#step-919}

### ติดตั้ง Sidekiq

```ruby
# Gemfile
gem "sidekiq"
gem "sidekiq-web"  # Web UI สำหรับ monitor jobs
```

```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq
```

```yaml
# config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]
  - [default, 2]
  - [mailers, 1]
  - [low, 1]
```

### Job Classes

```ruby
# app/jobs/order_confirmation_mailer_job.rb
class OrderConfirmationMailerJob < ApplicationJob
  queue_as :mailers

  retry_on StandardError, wait: :polynomially_longer, attempts: 3

  def perform(order_id)
    order = Order.find(order_id)
    OrderMailer.order_confirmation(order).deliver_now
  rescue ActiveRecord::RecordNotFound
    # Order ถูกลบแล้ว ไม่ต้อง retry
    Rails.logger.warn "Order ##{order_id} not found, skipping email"
  end
end
```

```ruby
# app/jobs/shipping_notification_job.rb
class ShippingNotificationJob < ApplicationJob
  queue_as :mailers

  def perform(order_id, tracking_number)
    order = Order.find(order_id)
    order.update!(tracking_number: tracking_number)
    OrderMailer.shipping_update(order).deliver_now
  end
end
```

```ruby
# app/jobs/inventory_update_job.rb
class InventoryUpdateJob < ApplicationJob
  queue_as :default

  # อัปเดต stock หลัง order ถูก cancel หรือ refund
  def perform(order_id)
    order = Order.includes(order_items: :product).find(order_id)

    order.order_items.each do |item|
      item.product.with_lock do
        item.product.increment!(:stock_quantity, item.quantity)
      end
    end

    Rails.logger.info "Inventory restored for Order ##{order.order_number}"
  end
end
```

```ruby
# app/jobs/product_stats_update_job.rb
class ProductStatsUpdateJob < ApplicationJob
  queue_as :low

  # รัน scheduled job ทุกคืนเพื่อ update stats
  def perform
    Product.published.find_each do |product|
      product.update_average_rating!
    end
    Rails.logger.info "Product stats updated at #{Time.current}"
  end
end
```

### Sidekiq Web UI

```ruby
# config/routes.rb
require "sidekiq/web"

Rails.application.routes.draw do
  # Mount Sidekiq web UI สำหรับ admin เท่านั้น
  authenticate :user, ->(u) { u.admin? } do
    mount Sidekiq::Web => "/admin/sidekiq"
  end

  # ... other routes
end
```

### ใช้งาน Jobs

```ruby
# ใน webhook controller
def handle_checkout_completed(session)
  order = Order.find_by(id: session.metadata["order_id"])
  order&.pay!(stripe_session_id: session.id)

  # ส่ง email async
  OrderConfirmationMailerJob.perform_later(order.id)
end

# ใน Admin orders controller
def ship_order
  @order.ship!(tracking_number: params[:tracking_number])
  ShippingNotificationJob.perform_later(@order.id, params[:tracking_number])
  redirect_to admin_order_path(@order), notice: "อัปเดตสถานะการจัดส่งแล้ว"
end
```

---

## Step 920: Deploy E-Commerce ด้วย Kamal {#step-920}

### ทำความเข้าใจ Kamal

Kamal (ก่อนหน้านี้ชื่อ MRSK) คือ deployment tool สำหรับ Rails apps ที่ใช้ Docker และ SSH ในการ deploy ไปยัง server ใดก็ได้

```bash
# ติดตั้ง Kamal
gem install kamal

# หรือใส่ใน Gemfile
gem "kamal", require: false
```

### Kamal Configuration

```yaml
# config/deploy.yml
service: ecommerce
image: your-dockerhub-username/ecommerce

servers:
  web:
    - 203.0.113.10  # IP ของ server หลัก
  worker:
    hosts:
      - 203.0.113.10
    cmd: bundle exec sidekiq

proxy:
  ssl: true
  host: ecommerce.example.com
  app_port: 3000

registry:
  username: your-dockerhub-username
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: true
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - STRIPE_SECRET_KEY
    - STRIPE_PUBLISHABLE_KEY
    - STRIPE_WEBHOOK_SECRET
    - REDIS_URL
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY
    - AWS_BUCKET

accessories:
  db:
    image: postgres:16
    host: 203.0.113.10
    env:
      clear:
        POSTGRES_DB: ecommerce_production
      secret:
        - POSTGRES_PASSWORD
    files:
      - config/init.sql:/docker-entrypoint-initdb.d/setup.sql
    directories:
      - data:/var/lib/postgresql/data

  redis:
    image: redis:7
    host: 203.0.113.10
    port: 6379
    directories:
      - data:/data

volumes:
  - ecommerce_storage:/rails/storage
```

### Environment Variables

```bash
# .env.production (อย่า commit ไฟล์นี้!)
RAILS_MASTER_KEY=<your master key>
DATABASE_URL=postgresql://postgres:password@localhost/ecommerce_production
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
REDIS_URL=redis://localhost:6379/0
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_BUCKET=ecommerce-production-uploads
KAMAL_REGISTRY_PASSWORD=<dockerhub token>
```

### Dockerfile ที่ Rails สร้างให้

```dockerfile
# Dockerfile (Rails scaffold)
FROM ruby:3.2-slim

# ติดตั้ง dependencies
RUN apt-get update && apt-get install -y \
  build-essential \
  git \
  libpq-dev \
  nodejs \
  npm \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /rails

# ติดตั้ง gems
COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

# Copy source
COPY . .

# Precompile assets
RUN SECRET_KEY_BASE_DUMMY=1 bundle exec rails assets:precompile

# Expose port
EXPOSE 3000

# Startup command
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Deploy Commands

```bash
# ครั้งแรก (setup server)
kamal setup

# Deploy ปกติ
kamal deploy

# Rollback ถ้ามีปัญหา
kamal rollback

# ดู logs
kamal logs
kamal logs --follow

# เข้า console บน production
kamal console

# Run database migrations
kamal exec --roles=web --cmd="bundle exec rails db:migrate"

# Restart app
kamal app restart

# ดูสถานะ
kamal app details
```

### Database Backup Strategy

```bash
#!/bin/bash
# scripts/backup_db.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="ecommerce_backup_${DATE}.sql.gz"
S3_BUCKET="ecommerce-backups"

# Dump database
docker exec $(docker ps -q -f name=ecommerce-db) \
  pg_dump -U postgres ecommerce_production | gzip > /tmp/${BACKUP_FILE}

# Upload to S3
aws s3 cp /tmp/${BACKUP_FILE} s3://${S3_BUCKET}/daily/${BACKUP_FILE}

# ลบไฟล์ backup เก่ากว่า 30 วันใน S3
aws s3 ls s3://${S3_BUCKET}/daily/ | \
  awk '{print $4}' | \
  while read key; do
    if [[ $(aws s3api head-object --bucket ${S3_BUCKET} --key daily/${key} --query 'LastModified' --output text) < $(date -d '30 days ago' --iso-8601) ]]; then
      aws s3 rm s3://${S3_BUCKET}/daily/${key}
    fi
  done

echo "Backup complete: ${BACKUP_FILE}"
```

```ruby
# config/schedule.rb (ใช้ whenever gem หรือ cron job)
# รัน backup ทุกวันตอน 02:00 น.
every 1.day, at: "2:00 am" do
  command "/path/to/scripts/backup_db.sh"
end
```

### Health Check และ Monitoring

```ruby
# config/routes.rb
get "/health", to: "health#show"
```

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!

  def show
    checks = {
      database: database_ok?,
      redis: redis_ok?,
      storage: storage_ok?
    }

    status = checks.values.all? ? :ok : :service_unavailable

    render json: {
      status: status == :ok ? "healthy" : "degraded",
      checks: checks,
      timestamp: Time.current.iso8601
    }, status: status
  end

  private

  def database_ok?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue
    false
  end

  def redis_ok?
    Redis.new.ping == "PONG"
  rescue
    false
  end

  def storage_ok?
    ActiveStorage::Blob.service.exist?("health_check")
    true
  rescue
    true  # ไม่ fail ถ้า object ไม่มี
  end
end
```

### Production Checklist

```
=== Pre-Deploy Checklist ===

[ ] ตรวจสอบ Rails credentials (rails credentials:edit)
[ ] ตั้งค่า environment variables ทั้งหมดใน Kamal
[ ] Stripe webhook URL ชี้ไปที่ production domain
[ ] Active Storage ตั้งค่าเป็น S3 (ไม่ใช่ local disk)
[ ] Sidekiq ทำงานได้ (ตรวจสอบ /admin/sidekiq)
[ ] Database backup ทำงานตามกำหนด
[ ] SSL certificate ใช้งานได้
[ ] Error monitoring (Sentry หรือ Honeybadger) ตั้งค่าแล้ว
[ ] Log aggregation ตั้งค่าแล้ว
[ ] Health check endpoint ตอบสนองได้
[ ] Load test ผ่าน

=== Stripe Production Setup ===

[ ] เปลี่ยนจาก test keys เป็น live keys
[ ] ตั้งค่า webhook endpoint ใน Stripe Dashboard
[ ] ทดสอบ payment flow บน production ด้วย real card
[ ] ตรวจสอบว่า webhook ถูกส่งและ handle ถูกต้อง
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### แบบฝึกหัดที่ 1: Coupon/Discount System
สร้าง `Coupon` model ที่มี `code`, `discount_type` (percentage/fixed), `amount`, `min_order_value`, `expires_at`, `usage_limit` และ integrate เข้ากับ checkout flow โดยผู้ใช้กรอก coupon code ใน cart และ total price ลดลงอัตโนมัติ

### แบบฝึกหัดที่ 2: Wishlist Feature ด้วย Turbo Streams
สร้าง `Wishlist` model (user has_many products through wishlists) และ toggle button บน product card ที่ใช้ Turbo Streams เพื่ออัปเดต heart icon โดยไม่ reload หน้า

### แบบฝึกหัดที่ 3: Order Timeline
เพิ่ม `OrderEvent` model ที่บันทึกทุกการเปลี่ยนแปลงสถานะของ order (เช่น "paid at 10:30", "shipped at 14:00") และแสดง timeline ใน order detail page

### แบบฝึกหัดที่ 4: Seller Analytics Chart
ใน seller dashboard สร้าง chart แสดงยอดขายรายวัน 30 วันล่าสุด โดยใช้ JavaScript chart library (Chartkick หรือ Chart.js) พร้อมข้อมูลจาก Rails API endpoint

### แบบฝึกหัดที่ 5: Product Search ด้วย Pg_Search
เพิ่ม `pg_search` gem และ implement full-text search ที่ค้นหาในทั้ง `name`, `description`, และ `seller.shop_name` พร้อม highlight คำค้นหาใน results

---

## สรุปสิ่งที่ได้เรียนรู้ {#สรุป}

ใน Part 092 นี้ เราได้สร้าง E-Commerce platform ครบวงจร ครอบคลุม:

| สิ่งที่เรียน | เนื้อหา |
|---|---|
| **Shopping Cart** | Hybrid cart (session + database), CartService |
| **Turbo Frames** | Real-time cart update ไม่ต้อง reload หน้า |
| **Form Object** | Multi-step checkout ด้วย `ActiveModel::Model` |
| **Stripe Integration** | Checkout Session + Webhook สำหรับ confirm payment |
| **AASM State Machine** | Order lifecycle (pending → paid → shipped → delivered) |
| **Action Mailer** | Email templates ภาษาไทย + HTML layout |
| **Seller Dashboard** | Analytics, revenue report, product management |
| **Counter Cache** | `reviews_count` อัปเดต auto ด้วย callback |
| **Sidekiq** | Async email + inventory jobs |
| **Kamal Deploy** | Config, commands, environment variables |

### Key Patterns ที่ควรจำ

1. **Service Object** — `CartService`, `OrderCreationService`, `StripeService` ช่วย extract business logic ออกจาก controller

2. **Form Object** — `Checkout::AddressForm` ช่วยจัดการ validation ที่ซับซ้อนแยกจาก model

3. **AASM State Machine** — กำหนด states, events, และ transitions อย่างชัดเจน พร้อม callbacks

4. **Turbo Frames** — `turbo_frame_tag` + `data: { turbo_frame: "frame_id" }` ทำให้ UI responsive

5. **Sidekiq + ActiveJob** — `queue_as :mailers` + `perform_later` ส่ง jobs ไป background

6. **Stripe Webhooks** — verify signature → handle event → update order status → queue email

### Architecture Overview

```
Browser → Turbo/Hotwire → Rails Controller
                               ↓
                         Service Objects
                           ↓         ↓
                       Model       Stripe API
                         ↓
                      Database
                         ↓
                    ActiveJob → Sidekiq → Email/Background Tasks
```

---

## ก้าวต่อไป {#ถัดไป}

หลังจากสร้าง E-Commerce platform เสร็จสมบูรณ์แล้ว ขั้นตอนต่อไปที่ควรทำ:

### Capstone Project 2 — Social Platform
**Part 093–096** จะครอบคลุม:
- Real-time features ด้วย Action Cable (chat, notifications)
- GraphQL API
- React/Inertia.js frontend
- Advanced caching strategies

### การปรับปรุง E-Commerce ต่อ
- เพิ่ม Elasticsearch สำหรับ full-text search
- Implement recommendation engine
- Multi-vendor support
- Mobile app ด้วย React Native + Rails API

### DevOps & Scalability
- CI/CD pipeline ด้วย GitHub Actions
- Load balancing ด้วย Nginx
- Database read replicas
- Redis caching สำหรับ product listings
- CDN สำหรับ static assets

```ruby
# ตัวอย่างโค้ดที่จะเรียนใน Capstone 2
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat_#{params[:room_id]}"
  end

  def speak(data)
    message = Message.create!(
      user: current_user,
      room_id: params[:room_id],
      content: data["message"]
    )

    ActionCable.server.broadcast(
      "chat_#{params[:room_id]}",
      {
        message: message.content,
        user: message.user.full_name,
        timestamp: message.created_at.strftime("%H:%M")
      }
    )
  end
end
```

---

**ยินดีด้วย! คุณสร้าง E-Commerce Platform ที่ทำงานได้จริงเสร็จแล้ว! 🎉**

*Part 092 สิ้นสุดแล้ว — Capstone 1 เสร็จสมบูรณ์! ไปต่อที่ Part 093 สำหรับ Capstone 2!*
