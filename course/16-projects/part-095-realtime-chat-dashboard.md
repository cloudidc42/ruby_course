# Part 095: Capstone 3 — Real-Time Chat/Dashboard App ด้วย Hotwire + ActionCable

> **Step ครอบคลุมใน Part นี้:** Step 941–950

**ระดับ:** ขั้นสูง (Advanced) | **Rails Version:** 7.x / 8.x | **Ruby Version:** 3.2+

ยินดีต้อนรับสู่ Capstone Project ชิ้นที่ 3 ซึ่งเป็น Capstone ที่ซับซ้อนและน่าตื่นเต้นที่สุดในหลักสูตรนี้ ในโปรเจกต์นี้เราจะสร้าง **Real-Time Chat และ Dashboard Application** แบบเต็มรูปแบบ โดยใช้ Hotwire (Turbo + Stimulus) ร่วมกับ ActionCable เพื่อทำให้แอปพลิเคชันตอบสนองแบบ Real-time ได้อย่างสมบูรณ์ ผู้เรียนจะได้เรียนรู้ถึงการออกแบบสถาปัตยกรรมของแอปที่ต้องการการเชื่อมต่อแบบถาวร (persistent connection) การจัดการ state ของผู้ใช้ที่ออนไลน์อยู่ การส่ง notification แบบ real-time รวมถึงการแสดงผล dashboard ที่อัปเดตข้อมูลโดยอัตโนมัติจาก background job ทุกแนวคิดที่ได้เรียนมาตั้งแต่ต้นหลักสูตรจะถูกนำมาประยุกต์ใช้ในโปรเจกต์นี้

---

## สารบัญ

- [Step 941: Product Spec สำหรับ Real-time App](#step-941)
- [Step 942: ActionCable Setup](#step-942)
- [Step 943: ChatRoom + Message Models](#step-943)
- [Step 944: Turbo Streams + ActionCable](#step-944)
- [Step 945: Online Presence](#step-945)
- [Step 946: Notification System](#step-946)
- [Step 947: Live Dashboard](#step-947)
- [Step 948: Typing Indicator](#step-948)
- [Step 949: File Sharing ใน Chat](#step-949)
- [Step 950: Performance ของ ActionCable at Scale](#step-950)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## Step 941: Product Spec สำหรับ Real-time App {#step-941}

ก่อนที่เราจะเริ่มเขียนโค้ด สิ่งสำคัญอันดับแรกคือการกำหนด **Product Spec** ที่ชัดเจน เพื่อให้เข้าใจว่าเราจะสร้างอะไร ผู้ใช้คือใคร และฟีเจอร์อะไรบ้างที่จำเป็น

### ภาพรวมของแอปพลิเคชัน

แอปของเราจะมีชื่อว่า **"RailsChat"** ซึ่งเป็นแพลตฟอร์ม chat แบบ real-time ที่มี dashboard สำหรับผู้ดูแลระบบ โดยมี features หลักดังนี้:

**1. Chat System**
- ห้องแชทส่วนตัว (Direct Message) ระหว่างผู้ใช้สองคน
- ห้องแชทกลุ่ม (Group Room) ที่สามารถมีผู้ใช้ได้หลายคน
- การส่งข้อความแบบ real-time โดยไม่ต้อง refresh หน้า
- การแสดงสถานะว่าใครออนไลน์อยู่บ้าง (Online Indicator)
- การแสดง typing indicator เมื่อคนอื่นกำลังพิมพ์
- การแนบไฟล์และรูปภาพในการสนทนา

**2. Dashboard (สำหรับ Admin)**
- แสดงจำนวนผู้ใช้ที่ออนไลน์อยู่ในขณะนั้น
- กราฟแสดงจำนวนข้อความที่ส่งในแต่ละชั่วโมง
- แสดง activity feed แบบ real-time
- Metric cards ที่อัปเดตอัตโนมัติ

**3. Notification System**
- แจ้งเตือนเมื่อมีข้อความใหม่
- Badge counter บน icon แจ้งเตือน
- Mark as read functionality

### User Stories

```
เป็นผู้ใช้ทั่วไป:
- ฉันสามารถเข้าสู่ระบบและเห็นรายการห้องแชทของฉัน
- ฉันสามารถส่งข้อความในห้องแชท และเห็นข้อความตอบกลับแบบทันที
- ฉันสามารถเห็นว่าเพื่อนออนไลน์อยู่หรือไม่
- ฉันสามารถเห็น notification เมื่อมีข้อความใหม่
- ฉันสามารถแนบรูปภาพในการสนทนา

เป็น Admin:
- ฉันสามารถดู dashboard ที่แสดง metric แบบ real-time
- ฉันสามารถเห็นจำนวนผู้ใช้ที่ออนไลน์ตลอดเวลา
- ฉันสามารถเห็น activity ล่าสุดของระบบ
```

### Data Model (ภาพรวม)

```
User
  ├── has_many :chat_room_memberships
  ├── has_many :chat_rooms, through: :chat_room_memberships
  ├── has_many :messages
  └── has_many :notifications

ChatRoom
  ├── belongs_to :creator (User)
  ├── has_many :chat_room_memberships
  ├── has_many :members (User), through: :chat_room_memberships
  └── has_many :messages
  
Message
  ├── belongs_to :chat_room
  ├── belongs_to :sender (User)
  └── has_many_attached :attachments (Active Storage)

Notification
  ├── belongs_to :recipient (User)
  ├── belongs_to :notifiable (polymorphic)
  └── read_at: datetime (nullable)
```

### เทคโนโลยีที่ใช้

| Layer | Technology | เหตุผล |
|-------|-----------|--------|
| Backend | Ruby on Rails 7+ | Framework หลัก |
| Real-time | ActionCable | WebSocket สำหรับ Rails |
| Frontend | Hotwire (Turbo + Stimulus) | Real-time UI updates |
| Background Jobs | Sidekiq + Redis | Async processing |
| File Storage | Active Storage + S3 | จัดการไฟล์แนบ |
| Database | PostgreSQL | Production-ready |
| Cache/PubSub | Redis | ActionCable adapter |

### สร้าง Rails App ใหม่

```bash
# สร้างโปรเจกต์ใหม่
rails new rails_chat \
  --database=postgresql \
  --skip-jbuilder \
  --asset-pipeline=sprockets

cd rails_chat

# ตรวจสอบว่า Turbo และ Stimulus ถูก install แล้ว
cat package.json

# เพิ่ม gems ที่จำเป็น
bundle add redis
bundle add sidekiq
bundle add devise
bundle add image_processing  # สำหรับ Active Storage variants
```

```ruby
# Gemfile (ส่วนที่เพิ่ม)
gem "redis", "~> 5.0"
gem "sidekiq", "~> 7.0"
gem "devise"
gem "image_processing", "~> 1.2"

group :development, :test do
  gem "factory_bot_rails"
  gem "faker"
end
```

---

## Step 942: ActionCable Setup — Adapter, Connection Authentication {#step-942}

ActionCable เป็นกรอบงาน (framework) ที่ Rails ใช้สำหรับการสื่อสารแบบ WebSocket ซึ่งทำให้เราสามารถส่งข้อมูลจาก server ไปยัง browser ได้แบบ real-time โดยไม่ต้องให้ browser คอย polling ซ้ำๆ

### ทำความเข้าใจ ActionCable Architecture

```
Browser (Consumer)
    ↕ WebSocket
Rails Server (Cable Server)
    ↕ Redis PubSub
Background Jobs / Other Processes
```

ActionCable มีส่วนประกอบหลักสามส่วน:
1. **Connection** — การเชื่อมต่อ WebSocket ระดับบนสุด (หนึ่งต่อ browser tab)
2. **Channel** — ช่องทางการส่งข้อมูลเฉพาะเรื่อง (เช่น ChatChannel, NotificationChannel)
3. **Subscription** — การลงทะเบียนรับข้อมูลจาก Channel เฉพาะ

### Configure ActionCable Adapter

```yaml
# config/cable.yml
development:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: rails_chat_production
```

### ตั้งค่า Connection Authentication

Connection คือจุดที่เราตรวจสอบว่าผู้ใช้มีสิทธิ์เชื่อมต่อหรือไม่

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
      logger.add_tags "ActionCable", "User #{current_user.id}"
    end

    def disconnect
      # เรียกเมื่อ WebSocket ตัดการเชื่อมต่อ
      # ใช้สำหรับ cleanup เช่น อัปเดต online status
      current_user.update(last_seen_at: Time.current) if current_user
    end

    private

    def find_verified_user
      # ตรวจสอบ session ของ Devise
      if verified_user = env["warden"].user
        verified_user
      else
        # ถ้าไม่ได้ login ให้ reject การเชื่อมต่อ
        reject_unauthorized_connection
      end
    end
  end
end
```

### Channel Base Class

```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
    # สามารถเพิ่ม shared logic ที่นี่ได้
    # เช่น before_subscribe callbacks, shared methods
    
    protected
    
    # Helper method สำหรับ authorize subscription
    def authorized_to_subscribe?(room)
      current_user.chat_rooms.include?(room)
    end
  end
end
```

### ตั้งค่า Routes สำหรับ ActionCable

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # ActionCable endpoint (Rails จัดการให้อัตโนมัติ)
  mount ActionCable.server => "/cable"
  
  # Routes อื่นๆ
  devise_for :users
  
  root "chat_rooms#index"
  
  resources :chat_rooms do
    resources :messages, only: [:create]
  end
  
  resources :notifications, only: [:index] do
    member do
      patch :mark_as_read
    end
    collection do
      patch :mark_all_as_read
    end
  end
  
  namespace :admin do
    get "dashboard", to: "dashboard#show"
  end
end
```

### JavaScript Consumer Setup

```javascript
// app/javascript/channels/consumer.js
// Turbo Rails จัดการ consumer ให้อัตโนมัติแล้ว
// แต่ถ้าต้องการ custom config ทำได้ที่นี่

import { createConsumer } from "@rails/actioncable"

// กำหนด URL ของ cable server
// สามารถส่ง token ใน query string ถ้าต้องการ
export default createConsumer("/cable")
```

### ทดสอบการเชื่อมต่อ ActionCable

```ruby
# ใน Rails console
ActionCable.server.connections.count
# แสดงจำนวน connections ที่เชื่อมต่ออยู่

# ส่งข้อความไปยัง channel ทดสอบ
ActionCable.server.broadcast("test_channel", { message: "Hello!" })
```

```javascript
// ใน Browser Console
// ตรวจสอบว่า ActionCable เชื่อมต่อแล้ว
App.cable.connection.isOpen()
// => true
```

---

## Step 943: ChatRoom + Message Models {#step-943}

ในขั้นตอนนี้เราจะสร้าง data models หลักของแอป ซึ่งรวมถึงการออกแบบ polymorphic rooms และการจัดการ file attachments ผ่าน Active Storage

### Migration Files

```bash
# สร้าง migrations
rails generate model ChatRoom \
  room_type:string \
  name:string \
  creator:references \
  slug:string

rails generate model ChatRoomMembership \
  chat_room:references \
  user:references \
  role:string \
  last_read_at:datetime

rails generate model Message \
  content:text \
  chat_room:references \
  sender:references \
  edited_at:datetime \
  deleted_at:datetime

# รัน migrations
rails db:migrate
```

```ruby
# db/migrate/xxx_create_chat_rooms.rb
class CreateChatRooms < ActiveRecord::Migration[7.0]
  def change
    create_table :chat_rooms do |t|
      t.string :room_type, null: false, default: "group"
      t.string :name
      t.references :creator, null: false, foreign_key: { to_table: :users }
      t.string :slug, null: false
      t.text :description
      t.boolean :is_private, default: false, null: false

      t.timestamps
    end
    
    add_index :chat_rooms, :slug, unique: true
    add_index :chat_rooms, :room_type
  end
end
```

```ruby
# db/migrate/xxx_create_messages.rb
class CreateMessages < ActiveRecord::Migration[7.0]
  def change
    create_table :messages do |t|
      t.text :content
      t.references :chat_room, null: false, foreign_key: true
      t.references :sender, null: false, foreign_key: { to_table: :users }
      t.datetime :edited_at
      t.datetime :deleted_at  # soft delete

      t.timestamps
    end
    
    add_index :messages, [:chat_room_id, :created_at]
  end
end
```

### ChatRoom Model

```ruby
# app/models/chat_room.rb
class ChatRoom < ApplicationRecord
  ROOM_TYPES = %w[direct group].freeze
  
  belongs_to :creator, class_name: "User"
  has_many :chat_room_memberships, dependent: :destroy
  has_many :members, through: :chat_room_memberships, source: :user
  has_many :messages, dependent: :destroy
  
  validates :room_type, inclusion: { in: ROOM_TYPES }
  validates :name, presence: true, if: :group?
  validates :slug, presence: true, uniqueness: true
  
  before_validation :generate_slug, on: :create
  
  scope :direct, -> { where(room_type: "direct") }
  scope :group, -> { where(room_type: "group") }
  scope :for_user, ->(user) { joins(:chat_room_memberships).where(chat_room_memberships: { user: user }) }
  
  def direct?
    room_type == "direct"
  end
  
  def group?
    room_type == "group"
  end
  
  def display_name_for(user)
    if direct?
      # สำหรับ direct room ให้แสดงชื่ออีกฝ่าย
      other_member = members.where.not(id: user.id).first
      other_member&.display_name || "Unknown"
    else
      name
    end
  end
  
  def unread_count_for(user)
    membership = chat_room_memberships.find_by(user: user)
    return 0 unless membership
    
    messages.where("created_at > ?", membership.last_read_at || Time.at(0)).count
  end
  
  # หา direct room ระหว่างผู้ใช้สองคน
  def self.find_or_create_direct_room(user1, user2)
    # หา room ที่มีสมาชิกทั้งสองคนพอดี
    room = joins(:chat_room_memberships)
             .where(room_type: "direct", chat_room_memberships: { user_id: [user1.id, user2.id] })
             .group(:id)
             .having("COUNT(chat_room_memberships.id) = 2")
             .first
    
    return room if room
    
    # ถ้าไม่มีให้สร้างใหม่
    transaction do
      room = create!(
        room_type: "direct",
        creator: user1,
        name: "Direct: #{user1.id}-#{user2.id}"
      )
      room.members << [user1, user2]
    end
    room
  end
  
  private
  
  def generate_slug
    self.slug ||= SecureRandom.hex(8)
  end
end
```

### Message Model พร้อม Active Storage

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :chat_room
  belongs_to :sender, class_name: "User"
  
  # Active Storage สำหรับไฟล์แนบ
  has_many_attached :attachments
  
  validates :content, presence: true, unless: :has_attachments?
  validates :content, length: { maximum: 5000 }
  
  # Soft delete
  scope :visible, -> { where(deleted_at: nil) }
  scope :recent, -> { order(created_at: :asc) }
  
  # Callback เพื่อ broadcast หลัง save
  after_create_commit :broadcast_to_room
  after_create_commit :create_notifications
  
  def deleted?
    deleted_at.present?
  end
  
  def soft_delete!
    update!(deleted_at: Time.current, content: "[ข้อความนี้ถูกลบ]")
  end
  
  def display_content
    deleted? ? "[ข้อความนี้ถูกลบ]" : content
  end
  
  private
  
  def has_attachments?
    attachments.any?
  end
  
  def broadcast_to_room
    # broadcast ไปยัง channel ของห้องนั้น
    # จะอธิบายรายละเอียดใน Step 944
    ChatRoomChannel.broadcast_new_message(self)
  end
  
  def create_notifications
    # สร้าง notification สำหรับ member อื่นๆ ในห้อง
    NotificationCreatorJob.perform_later(self)
  end
end
```

### ChatRoomMembership Model

```ruby
# app/models/chat_room_membership.rb
class ChatRoomMembership < ApplicationRecord
  ROLES = %w[member admin].freeze
  
  belongs_to :chat_room
  belongs_to :user
  
  validates :role, inclusion: { in: ROLES }
  validates :user_id, uniqueness: { scope: :chat_room_id }
  
  scope :admins, -> { where(role: "admin") }
  scope :members_role, -> { where(role: "member") }
  
  def mark_as_read!
    update!(last_read_at: Time.current)
  end
end
```

---

## Step 944: Turbo Streams + ActionCable — Broadcast Message แบบ Real-time {#step-944}

นี่คือหัวใจสำคัญของ real-time chat เราจะเชื่อม ActionCable เข้ากับ Turbo Streams เพื่อให้ข้อความใหม่ปรากฏในหน้า chat โดยอัตโนมัติ

### สร้าง ChatRoomChannel

```ruby
# app/channels/chat_room_channel.rb
class ChatRoomChannel < ApplicationCable::Channel
  
  def subscribed
    # รับ chat_room_id จาก params ที่ JavaScript ส่งมา
    @chat_room = ChatRoom.find(params[:chat_room_id])
    
    # ตรวจสอบว่า user มีสิทธิ์เข้าห้องนี้หรือไม่
    if authorized_to_subscribe?(@chat_room)
      # stream_for ใช้กับ Turbo::StreamsChannel
      stream_for @chat_room
      
      # อัปเดต last_read_at
      @chat_room.chat_room_memberships
                .find_by(user: current_user)
                &.mark_as_read!
    else
      reject  # ปฏิเสธการ subscribe
    end
  end
  
  def unsubscribed
    # ทำ cleanup เมื่อ user ออกจากห้อง
    stop_all_streams
  end
  
  def receive(data)
    # รับข้อความจาก client (ถ้าต้องการ)
    # แต่ส่วนใหญ่เราจะใช้ HTTP POST แทน
  end
  
  # Class method สำหรับ broadcast ข้อความใหม่
  def self.broadcast_new_message(message)
    # Broadcast Turbo Stream ไปยัง chat room channel
    broadcast_append_to(
      message.chat_room,
      target: "messages",
      partial: "messages/message",
      locals: { message: message }
    )
  end
end
```

### Controller สำหรับส่งข้อความ

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_chat_room
  
  def create
    @message = @chat_room.messages.build(message_params)
    @message.sender = current_user
    
    if @message.save
      # ไม่ต้อง render อะไรเพราะ after_create_commit จะ broadcast ให้
      head :ok
    else
      render json: { errors: @message.errors.full_messages }, status: :unprocessable_entity
    end
  end
  
  private
  
  def set_chat_room
    @chat_room = current_user.chat_rooms.find(params[:chat_room_id])
  rescue ActiveRecord::RecordNotFound
    render json: { error: "ไม่พบห้องแชท หรือคุณไม่มีสิทธิ์เข้าถึง" }, status: :forbidden
  end
  
  def message_params
    params.require(:message).permit(:content, attachments: [])
  end
end
```

### View สำหรับห้องแชท

```erb
<%# app/views/chat_rooms/show.html.erb %>
<div class="chat-room" data-controller="chat-room">
  <div class="chat-header">
    <h2><%= @chat_room.display_name_for(current_user) %></h2>
    <div class="online-indicators" id="online-indicators-<%= @chat_room.id %>">
      <%# Online indicators จะถูก inject ผ่าน Turbo Stream %>
    </div>
  </div>
  
  <%# Turbo Stream subscription %>
  <%= turbo_stream_from @chat_room %>
  
  <div class="messages-container">
    <%# Container ที่ Turbo Stream จะ append messages เข้ามา %>
    <div id="messages">
      <%= render @messages %>
    </div>
  </div>
  
  <%# Typing indicator area %>
  <div id="typing-indicator-<%= @chat_room.id %>" class="typing-indicator">
  </div>
  
  <%# Message form %>
  <%= render "messages/form", chat_room: @chat_room %>
</div>
```

```erb
<%# app/views/messages/_message.html.erb %>
<%# ID ต้องไม่ซ้ำสำหรับ Turbo Stream %>
<div id="<%= dom_id(message) %>" 
     class="message <%= message.sender == current_user ? 'own-message' : 'other-message' %>">
  
  <div class="message-avatar">
    <%= image_tag message.sender.avatar_url, class: "avatar-small" %>
  </div>
  
  <div class="message-content">
    <div class="message-header">
      <span class="sender-name"><%= message.sender.display_name %></span>
      <span class="message-time"><%= message.created_at.strftime("%H:%M") %></span>
    </div>
    
    <div class="message-body">
      <% if message.deleted? %>
        <em class="deleted-message">ข้อความนี้ถูกลบแล้ว</em>
      <% else %>
        <p><%= message.content %></p>
        
        <% if message.attachments.any? %>
          <div class="attachments">
            <% message.attachments.each do |attachment| %>
              <% if attachment.image? %>
                <%= image_tag attachment, class: "chat-image" %>
              <% else %>
                <%= link_to attachment.filename, rails_blob_path(attachment), class: "file-attachment" %>
              <% end %>
            <% end %>
          </div>
        <% end %>
      <% end %>
    </div>
  </div>
</div>
```

### Form สำหรับส่งข้อความ

```erb
<%# app/views/messages/_form.html.erb %>
<div class="message-form-container" 
     data-controller="message-form"
     data-message-form-chat-room-id-value="<%= chat_room.id %>">
  
  <%= form_with(
    url: chat_room_messages_path(chat_room),
    data: {
      message_form_target: "form",
      action: "turbo:submit-end->message-form#afterSubmit"
    }
  ) do |form| %>
    
    <div class="input-area">
      <%= form.text_area :content,
            placeholder: "พิมพ์ข้อความ...",
            rows: 1,
            class: "message-input",
            data: {
              message_form_target: "input",
              action: "input->message-form#resize keydown->message-form#submitOnEnter"
            } %>
      
      <div class="action-buttons">
        <%# File attachment button %>
        <label class="attachment-btn" data-message-form-target="attachLabel">
          📎
          <%= form.file_field :attachments,
                multiple: true,
                direct_upload: true,
                class: "hidden",
                data: { message_form_target: "fileInput", action: "change->message-form#handleFileSelect" } %>
        </label>
        
        <%= form.submit "ส่ง", class: "send-button" %>
      </div>
    </div>
  <% end %>
</div>
```

---

## Step 945: Online Presence — Track Connected Users {#step-945}

การแสดงสถานะออนไลน์เป็นฟีเจอร์สำคัญของ chat app เราจะใช้ ActionCable connection lifecycle เพื่อติดตามว่าผู้ใช้คนไหนออนไลน์อยู่

### PresenceChannel

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  PRESENCE_KEY_PREFIX = "online_users".freeze
  ONLINE_TIMEOUT = 30.seconds
  
  def subscribed
    stream_from "presence:#{params[:chat_room_id]}"
    
    # บันทึกว่า user เชื่อมต่อแล้ว
    mark_online!
    
    # Broadcast ให้คนอื่นรู้ว่ามีคนเข้ามา
    broadcast_online_status(true)
  end
  
  def unsubscribed
    # Broadcast ให้คนอื่นรู้ว่า user ออกไปแล้ว
    broadcast_online_status(false)
    
    # ลบออกจาก online set
    mark_offline!
  end
  
  # Client สามารถส่ง heartbeat มาเพื่อยืนยันว่ายังออนไลน์อยู่
  def heartbeat
    mark_online!
  end
  
  def self.online_users_for_room(chat_room_id)
    user_ids = Redis.current.smembers("#{PRESENCE_KEY_PREFIX}:room:#{chat_room_id}")
    User.where(id: user_ids)
  end
  
  private
  
  def mark_online!
    key = "#{PRESENCE_KEY_PREFIX}:room:#{params[:chat_room_id]}"
    Redis.current.sadd(key, current_user.id)
    Redis.current.expire(key, ONLINE_TIMEOUT * 10)
    
    # บันทึก last_seen_at ลงฐานข้อมูลด้วย
    current_user.update_column(:last_seen_at, Time.current)
  end
  
  def mark_offline!
    key = "#{PRESENCE_KEY_PREFIX}:room:#{params[:chat_room_id]}"
    Redis.current.srem(key, current_user.id)
  end
  
  def broadcast_online_status(is_online)
    ActionCable.server.broadcast(
      "presence:#{params[:chat_room_id]}",
      {
        user_id: current_user.id,
        user_name: current_user.display_name,
        avatar_url: current_user.avatar_url,
        online: is_online,
        timestamp: Time.current.iso8601
      }
    )
  end
end
```

### Stimulus Controller สำหรับ Online Indicators

```javascript
// app/javascript/controllers/presence_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static values = {
    chatRoomId: Number,
    currentUserId: Number
  }
  
  static targets = ["onlineList", "indicator"]
  
  connect() {
    this.onlineUsers = new Map()
    this.subscription = this.createSubscription()
  }
  
  disconnect() {
    if (this.subscription) {
      this.subscription.unsubscribe()
    }
  }
  
  createSubscription() {
    return consumer.subscriptions.create(
      {
        channel: "PresenceChannel",
        chat_room_id: this.chatRoomIdValue
      },
      {
        received: (data) => {
          this.handlePresenceUpdate(data)
        },
        
        connected: () => {
          // เริ่ม heartbeat ทุก 20 วินาที
          this.heartbeatInterval = setInterval(() => {
            this.subscription.perform("heartbeat")
          }, 20000)
        },
        
        disconnected: () => {
          if (this.heartbeatInterval) {
            clearInterval(this.heartbeatInterval)
          }
        }
      }
    )
  }
  
  handlePresenceUpdate(data) {
    if (data.online) {
      this.onlineUsers.set(data.user_id, {
        name: data.user_name,
        avatarUrl: data.avatar_url
      })
    } else {
      this.onlineUsers.delete(data.user_id)
    }
    
    this.renderOnlineUsers()
  }
  
  renderOnlineUsers() {
    const container = this.onlineListTarget
    container.innerHTML = ""
    
    this.onlineUsers.forEach((user, userId) => {
      const indicator = document.createElement("div")
      indicator.className = "online-indicator"
      indicator.dataset.userId = userId
      indicator.title = `${user.name} (ออนไลน์)`
      indicator.innerHTML = `
        <img src="${user.avatarUrl}" alt="${user.name}" class="avatar-tiny">
        <span class="online-dot"></span>
      `
      container.appendChild(indicator)
    })
    
    // อัปเดต counter
    const count = this.onlineUsers.size
    const countEl = document.getElementById("online-count")
    if (countEl) {
      countEl.textContent = `${count} คนออนไลน์`
    }
  }
}
```

---

## Step 946: Notification System — Broadcast to User Channel {#step-946}

ระบบ notification จะแจ้งเตือนผู้ใช้เมื่อมีเหตุการณ์สำคัญเกิดขึ้น เช่น มีข้อความใหม่ในห้องที่ตนเองไม่ได้เปิดอยู่

### Notification Model

```ruby
# app/models/notification.rb
class Notification < ApplicationRecord
  belongs_to :recipient, class_name: "User"
  belongs_to :notifiable, polymorphic: true
  belongs_to :actor, class_name: "User"
  
  validates :notification_type, presence: true
  
  scope :unread, -> { where(read_at: nil) }
  scope :recent, -> { order(created_at: :desc).limit(20) }
  
  after_create_commit :broadcast_to_recipient
  
  def read?
    read_at.present?
  end
  
  def mark_as_read!
    update!(read_at: Time.current)
  end
  
  def message
    case notification_type
    when "new_message"
      "#{actor.display_name} ส่งข้อความใหม่"
    when "mention"
      "#{actor.display_name} กล่าวถึงคุณ"
    else
      "คุณมีการแจ้งเตือนใหม่"
    end
  end
  
  private
  
  def broadcast_to_recipient
    NotificationChannel.broadcast_to(
      recipient,
      {
        id: id,
        message: message,
        notification_type: notification_type,
        unread_count: recipient.notifications.unread.count,
        url: notifiable_url,
        created_at: created_at.iso8601
      }
    )
  end
  
  def notifiable_url
    case notifiable_type
    when "Message"
      "/chat_rooms/#{notifiable.chat_room_id}"
    else
      "/notifications"
    end
  end
end
```

### NotificationChannel

```ruby
# app/channels/notification_channel.rb
class NotificationChannel < ApplicationCable::Channel
  def subscribed
    # stream_for ใช้ model instance เป็น stream identifier
    stream_for current_user
  end
  
  def unsubscribed
    stop_all_streams
  end
  
  def mark_as_read(data)
    notification = current_user.notifications.find(data["notification_id"])
    notification.mark_as_read!
    
    # Broadcast unread count ที่อัปเดตแล้ว
    broadcast_unread_count
  end
  
  def mark_all_as_read
    current_user.notifications.unread.update_all(read_at: Time.current)
    broadcast_unread_count
  end
  
  private
  
  def broadcast_unread_count
    NotificationChannel.broadcast_to(
      current_user,
      { type: "unread_count", count: current_user.notifications.unread.count }
    )
  end
end
```

### Background Job สำหรับสร้าง Notification

```ruby
# app/jobs/notification_creator_job.rb
class NotificationCreatorJob < ApplicationJob
  queue_as :notifications
  
  def perform(message)
    # ส่ง notification ให้ members ทุกคน ยกเว้น sender
    members_to_notify = message.chat_room.members
                                         .where.not(id: message.sender_id)
    
    members_to_notify.find_each do |member|
      # ไม่ส่ง notification ถ้า user กำลังดูห้องนั้นอยู่
      next if user_viewing_room?(member, message.chat_room)
      
      Notification.create!(
        recipient: member,
        actor: message.sender,
        notifiable: message,
        notification_type: "new_message"
      )
    end
  end
  
  private
  
  def user_viewing_room?(user, chat_room)
    # ตรวจสอบผ่าน Redis ว่า user subscribe อยู่กับ room นั้นหรือไม่
    online_in_room = Redis.current.sismember(
      "online_users:room:#{chat_room.id}",
      user.id
    )
    online_in_room
  end
end
```

### Stimulus Controller สำหรับ Notification Badge

```javascript
// app/javascript/controllers/notification_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static targets = ["badge", "list", "count"]
  
  connect() {
    this.subscription = consumer.subscriptions.create(
      "NotificationChannel",
      {
        received: (data) => {
          if (data.type === "unread_count") {
            this.updateBadge(data.count)
          } else {
            this.addNotification(data)
            this.updateBadge(data.unread_count)
          }
        }
      }
    )
  }
  
  disconnect() {
    this.subscription?.unsubscribe()
  }
  
  updateBadge(count) {
    if (this.hasBadgeTarget) {
      if (count > 0) {
        this.badgeTarget.textContent = count > 99 ? "99+" : count
        this.badgeTarget.classList.remove("hidden")
      } else {
        this.badgeTarget.classList.add("hidden")
      }
    }
  }
  
  addNotification(notification) {
    const item = document.createElement("li")
    item.className = "notification-item unread"
    item.dataset.notificationId = notification.id
    item.innerHTML = `
      <a href="${notification.url}" class="notification-link">
        <span class="notification-message">${notification.message}</span>
        <span class="notification-time">${this.formatTime(notification.created_at)}</span>
      </a>
    `
    
    if (this.hasListTarget) {
      this.listTarget.prepend(item)
    }
    
    // แสดง toast notification
    this.showToast(notification.message)
  }
  
  showToast(message) {
    const toast = document.createElement("div")
    toast.className = "toast-notification"
    toast.textContent = message
    document.body.appendChild(toast)
    
    setTimeout(() => toast.classList.add("show"), 100)
    setTimeout(() => {
      toast.classList.remove("show")
      setTimeout(() => toast.remove(), 300)
    }, 3000)
  }
  
  formatTime(isoString) {
    const date = new Date(isoString)
    return date.toLocaleTimeString("th-TH", { hour: "2-digit", minute: "2-digit" })
  }
}
```

---

## Step 947: Live Dashboard — Chart ที่ Update Real-time {#step-947}

สำหรับ admin dashboard เราจะสร้างกราฟและ metric ที่อัปเดตแบบ real-time ผ่านการ broadcast จาก background job

### Dashboard Channel

```ruby
# app/channels/dashboard_channel.rb
class DashboardChannel < ApplicationCable::Channel
  def subscribed
    # ตรวจสอบว่าเป็น admin เท่านั้น
    if current_user.admin?
      stream_from "admin:dashboard"
    else
      reject
    end
  end
  
  def unsubscribed
    stop_all_streams
  end
end
```

### Dashboard Metrics Job

```ruby
# app/jobs/dashboard_metrics_job.rb
class DashboardMetricsJob < ApplicationJob
  queue_as :default
  
  def perform
    metrics = collect_metrics
    
    # Broadcast ไปยัง dashboard channel
    ActionCable.server.broadcast("admin:dashboard", {
      type: "metrics_update",
      data: metrics,
      timestamp: Time.current.iso8601
    })
  end
  
  private
  
  def collect_metrics
    {
      online_users: online_users_count,
      messages_last_hour: messages_last_hour,
      messages_by_hour: messages_by_hour_chart_data,
      active_rooms: active_rooms_count,
      new_users_today: new_users_today
    }
  end
  
  def online_users_count
    # นับจาก Redis
    keys = Redis.current.keys("online_users:room:*")
    user_ids = keys.flat_map { |key| Redis.current.smembers(key) }
    user_ids.uniq.count
  end
  
  def messages_last_hour
    Message.where("created_at > ?", 1.hour.ago).count
  end
  
  def messages_by_hour_chart_data
    # ข้อมูล 24 ชั่วโมงล่าสุด
    24.times.map do |hours_ago|
      start_time = hours_ago.hours.ago.beginning_of_hour
      end_time = start_time + 1.hour
      
      {
        hour: start_time.strftime("%H:00"),
        count: Message.where(created_at: start_time..end_time).count
      }
    end.reverse
  end
  
  def active_rooms_count
    Message.where("created_at > ?", 1.hour.ago)
           .select(:chat_room_id)
           .distinct
           .count
  end
  
  def new_users_today
    User.where("created_at > ?", Time.current.beginning_of_day).count
  end
end
```

### รัน Job แบบ Recurring ด้วย Sidekiq Cron

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0") }
end

# ตั้งค่า recurring job (ต้องใช้ sidekiq-cron gem)
# หรือใช้ clockwork / whenever
```

```yaml
# config/sidekiq.yml
:concurrency: 5
:queues:
  - [critical, 3]
  - [default, 2]
  - [notifications, 2]

# sidekiq-cron สำหรับ recurring jobs
:cron:
  dashboard_metrics:
    cron: "*/30 * * * * *"  # ทุก 30 วินาที
    class: DashboardMetricsJob
    queue: default
```

### Dashboard View

```erb
<%# app/views/admin/dashboard/show.html.erb %>
<div class="dashboard" 
     data-controller="dashboard"
     data-dashboard-channel-value="admin:dashboard">
  
  <div class="metric-cards">
    <div class="metric-card" id="online-users-card">
      <h3>ผู้ใช้ออนไลน์</h3>
      <span class="metric-value" data-dashboard-target="onlineUsers">
        <%= @metrics[:online_users] %>
      </span>
    </div>
    
    <div class="metric-card" id="messages-hour-card">
      <h3>ข้อความใน 1 ชั่วโมง</h3>
      <span class="metric-value" data-dashboard-target="messagesHour">
        <%= @metrics[:messages_last_hour] %>
      </span>
    </div>
    
    <div class="metric-card" id="active-rooms-card">
      <h3>ห้องที่มีกิจกรรม</h3>
      <span class="metric-value" data-dashboard-target="activeRooms">
        <%= @metrics[:active_rooms] %>
      </span>
    </div>
  </div>
  
  <div class="chart-container">
    <canvas id="messages-chart" 
            data-dashboard-target="chart"
            data-chart-data="<%= @metrics[:messages_by_hour].to_json %>">
    </canvas>
  </div>
</div>
```

---

## Step 948: Typing Indicator — Stimulus + ActionCable + Debounce {#step-948}

Typing indicator ช่วยให้ผู้ใช้รู้ว่าคนอื่นกำลังพิมพ์ข้อความอยู่ เราจะสร้างโดยใช้ ActionCable ส่ง event เมื่อผู้ใช้พิมพ์ และใช้ debounce เพื่อหยุดส่งเมื่อหยุดพิมพ์

### TypingChannel

```ruby
# app/channels/typing_channel.rb
class TypingChannel < ApplicationCable::Channel
  def subscribed
    @chat_room = ChatRoom.find(params[:chat_room_id])
    
    if authorized_to_subscribe?(@chat_room)
      stream_from "typing:#{@chat_room.id}"
    else
      reject
    end
  end
  
  def unsubscribed
    # ส่ง event ว่าหยุดพิมพ์แล้ว
    broadcast_typing_status(false)
    stop_all_streams
  end
  
  def typing(data)
    broadcast_typing_status(data["is_typing"])
  end
  
  private
  
  def broadcast_typing_status(is_typing)
    ActionCable.server.broadcast(
      "typing:#{params[:chat_room_id]}",
      {
        user_id: current_user.id,
        user_name: current_user.display_name,
        is_typing: is_typing
      }
    )
  end
end
```

### Stimulus Controller สำหรับ Typing Indicator

```javascript
// app/javascript/controllers/typing_indicator_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static values = {
    chatRoomId: Number,
    currentUserId: Number,
    debounceMs: { type: Number, default: 1000 }
  }
  
  static targets = ["indicator", "input"]
  
  connect() {
    this.typingUsers = new Map()
    this.isTyping = false
    this.stopTypingTimer = null
    
    this.subscription = this.createSubscription()
  }
  
  disconnect() {
    this.subscription?.unsubscribe()
    if (this.stopTypingTimer) {
      clearTimeout(this.stopTypingTimer)
    }
  }
  
  createSubscription() {
    return consumer.subscriptions.create(
      {
        channel: "TypingChannel",
        chat_room_id: this.chatRoomIdValue
      },
      {
        received: (data) => {
          // ไม่ต้องแสดง indicator สำหรับตัวเอง
          if (data.user_id === this.currentUserIdValue) return
          
          if (data.is_typing) {
            this.typingUsers.set(data.user_id, data.user_name)
          } else {
            this.typingUsers.delete(data.user_id)
          }
          
          this.updateIndicator()
        }
      }
    )
  }
  
  // เรียกเมื่อผู้ใช้พิมพ์ใน input
  handleInput() {
    if (!this.isTyping) {
      this.isTyping = true
      this.subscription.perform("typing", { is_typing: true })
    }
    
    // รีเซ็ต timer ทุกครั้งที่พิมพ์
    if (this.stopTypingTimer) {
      clearTimeout(this.stopTypingTimer)
    }
    
    // ส่ง "หยุดพิมพ์" หลังจากไม่พิมพ์ 1.5 วินาที
    this.stopTypingTimer = setTimeout(() => {
      this.isTyping = false
      this.subscription.perform("typing", { is_typing: false })
    }, this.debounceMsValue)
  }
  
  updateIndicator() {
    if (!this.hasIndicatorTarget) return
    
    const names = Array.from(this.typingUsers.values())
    
    if (names.length === 0) {
      this.indicatorTarget.textContent = ""
      this.indicatorTarget.classList.add("hidden")
    } else if (names.length === 1) {
      this.indicatorTarget.innerHTML = `
        <span class="typing-dots">
          <span></span><span></span><span></span>
        </span>
        <span>${names[0]} กำลังพิมพ์...</span>
      `
      this.indicatorTarget.classList.remove("hidden")
    } else if (names.length === 2) {
      this.indicatorTarget.textContent = `${names[0]} และ ${names[1]} กำลังพิมพ์...`
      this.indicatorTarget.classList.remove("hidden")
    } else {
      this.indicatorTarget.textContent = `${names.length} คนกำลังพิมพ์...`
      this.indicatorTarget.classList.remove("hidden")
    }
  }
}
```

---

## Step 949: File Sharing ใน Chat — Direct Upload ด้วย Active Storage {#step-949}

การแนบไฟล์ในแชทเป็นฟีเจอร์ที่จำเป็น เราจะใช้ Active Storage Direct Upload ซึ่งช่วยให้อัปโหลดไฟล์ตรงไปยัง Cloud Storage (S3) โดยไม่ต้องผ่าน Rails server

### ตั้งค่า Active Storage

```ruby
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= ENV["AWS_ACCESS_KEY_ID"] %>
  secret_access_key: <%= ENV["AWS_SECRET_ACCESS_KEY"] %>
  region: <%= ENV["AWS_REGION"] %>
  bucket: <%= ENV["AWS_S3_BUCKET"] %>
  
# config/environments/production.rb
config.active_storage.service = :amazon
config.active_storage.default_url_options = { host: ENV["APP_HOST"] }
```

### Stimulus Controller สำหรับ Direct Upload

```javascript
// app/javascript/controllers/file_upload_controller.js
import { Controller } from "@hotwired/stimulus"
import { DirectUpload } from "@rails/activestorage"

export default class extends Controller {
  static values = {
    url: String,  // direct_uploads_url
    maxSize: { type: Number, default: 10 }  // MB
  }
  
  static targets = ["preview", "progress", "input", "list"]
  
  connect() {
    this.files = []
  }
  
  handleFileSelect(event) {
    const files = Array.from(event.target.files)
    
    files.forEach(file => {
      // ตรวจสอบขนาดไฟล์
      if (file.size > this.maxSizeValue * 1024 * 1024) {
        alert(`ไฟล์ ${file.name} ใหญ่เกิน ${this.maxSizeValue}MB`)
        return
      }
      
      this.uploadFile(file)
    })
  }
  
  uploadFile(file) {
    const upload = new DirectUpload(file, this.urlValue, this)
    
    // สร้าง preview item
    const previewItem = this.createPreviewItem(file)
    this.listTarget.appendChild(previewItem)
    
    upload.create((error, blob) => {
      if (error) {
        console.error("Upload failed:", error)
        previewItem.classList.add("error")
        previewItem.querySelector(".status").textContent = "อัปโหลดล้มเหลว"
      } else {
        // เพิ่ม hidden input เพื่อส่ง blob ID ไปกับ form
        const hiddenField = document.createElement("input")
        hiddenField.type = "hidden"
        hiddenField.name = "message[attachments][]"
        hiddenField.value = blob.signed_id
        this.element.querySelector("form").appendChild(hiddenField)
        
        previewItem.classList.add("completed")
        previewItem.querySelector(".status").textContent = "อัปโหลดสำเร็จ"
        
        this.files.push({ blob, name: file.name })
      }
    })
  }
  
  // Callback จาก DirectUpload สำหรับ progress
  directUploadWillStoreFileWithXHR(request) {
    request.upload.addEventListener("progress", (event) => {
      const progress = event.loaded / event.total * 100
      this.updateProgress(event.target._fileId, progress)
    })
  }
  
  createPreviewItem(file) {
    const item = document.createElement("div")
    item.className = "upload-preview-item"
    
    if (file.type.startsWith("image/")) {
      const reader = new FileReader()
      reader.onload = (e) => {
        item.querySelector("img").src = e.target.result
      }
      reader.readAsDataURL(file)
      
      item.innerHTML = `
        <img src="" alt="${file.name}" class="preview-thumbnail">
        <div class="upload-info">
          <span class="file-name">${file.name}</span>
          <div class="progress-bar">
            <div class="progress-fill" style="width: 0%"></div>
          </div>
          <span class="status">กำลังอัปโหลด...</span>
        </div>
        <button type="button" class="remove-btn" data-action="click->file-upload#removeFile">✕</button>
      `
    } else {
      item.innerHTML = `
        <div class="file-icon">📎</div>
        <div class="upload-info">
          <span class="file-name">${file.name}</span>
          <span class="file-size">${this.formatSize(file.size)}</span>
          <div class="progress-bar">
            <div class="progress-fill" style="width: 0%"></div>
          </div>
          <span class="status">กำลังอัปโหลด...</span>
        </div>
        <button type="button" class="remove-btn" data-action="click->file-upload#removeFile">✕</button>
      `
    }
    
    return item
  }
  
  formatSize(bytes) {
    if (bytes < 1024) return `${bytes} B`
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
    return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
  }
}
```

---

## Step 950: Performance ของ ActionCable at Scale {#step-950}

เมื่อมีผู้ใช้งานมากขึ้น ActionCable ต้องรับมือกับ connections จำนวนมาก ในขั้นตอนนี้เราจะศึกษาวิธีเพิ่มประสิทธิภาพของระบบ

### ทำความเข้าใจ Connection Limits

ActionCable ใช้ WebSocket ซึ่งแต่ละ connection จะถือครอง thread หรือ fiber ไว้ Rails สามารถรองรับ connections ได้มากน้อยแค่ไหนขึ้นอยู่กับ:

```ruby
# config/environments/production.rb
# ActionCable สามารถทำงานกับ Puma ได้ดี
# เนื่องจาก Puma ใช้ thread pool

# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# แต่ละ worker รองรับ connections ได้มากถึงหลักพัน
# เพราะ ActionCable ใช้ async I/O
```

### Redis Pub/Sub ภายใน ActionCable

```
ผู้ส่งข้อความ (Client A)
    ↓ HTTP POST
Rails Server
    ↓ ActionCable.server.broadcast
Redis (PUBLISH "action_cable:chat_room_1" data)
    ↓ SUBSCRIBE
Rails Server (Worker 1, Worker 2, Worker 3...)
    ↓ WebSocket push
ผู้รับข้อความ (Client B, C, D...)
```

Redis ทำหน้าที่เป็น message broker ระหว่าง Rails processes ทำให้แอปสามารถ scale แบบ horizontal ได้ (รัน Rails หลาย process)

### ปรับแต่ง ActionCable Configuration

```ruby
# config/application.rb
module RailsChat
  class Application < Rails::Application
    # กำหนด allowed request origins สำหรับ security
    config.action_cable.allowed_request_origins = [
      "https://your-domain.com",
      /http:\/\/localhost:.*/
    ]
    
    # กำหนด worker pool size
    config.action_cable.worker_pool_size = 4
    
    # Disable ActionCable logging ใน production เพื่อลด overhead
    # config.action_cable.log_tags = []
  end
end
```

### Cable Ready Gem สำหรับ Advanced Operations

```ruby
# Gemfile
gem "cable_ready"
gem "morphdom-rails"  # สำหรับ efficient DOM updates

# app/channels/chat_room_channel.rb
class ChatRoomChannel < ApplicationCable::Channel
  include CableReady::Broadcaster
  
  def subscribed
    stream_for current_user
  end
  
  def self.update_unread_count(user, count)
    cable_ready[user].text_content(
      selector: "#unread-badge",
      text: count.to_s
    )
    cable_ready.broadcast
  end
end
```

### Monitoring ActionCable Connections

```ruby
# lib/action_cable_stats.rb
module ActionCableStats
  def self.connection_count
    ActionCable.server.connections.count
  end
  
  def self.subscriptions_per_channel
    ActionCable.server.connections.flat_map(&:subscriptions).
      group_by { |sub| sub.class.name }.
      transform_values(&:count)
  end
end

# ใน admin endpoint
# GET /admin/stats
class Admin::StatsController < ApplicationController
  def index
    @stats = {
      connections: ActionCableStats.connection_count,
      subscriptions: ActionCableStats.subscriptions_per_channel,
      redis_info: Redis.current.info("memory").slice("used_memory_human")
    }
    render json: @stats
  end
end
```

### Best Practices สรุป

```ruby
# 1. ใช้ Turbo::StreamsChannel แทนการ broadcast เอง
# ดีกว่า: ใช้ broadcast_append_to, broadcast_replace_to
# ไม่ดี: เขียน ActionCable.server.broadcast ตรงๆ ทุกที่

# 2. หลีกเลี่ยง Heavy computation ใน channel callbacks
# ดีกว่า:
def subscribed
  stream_for @room
end
# แล้วค่อย compute ใน background job

# 3. ใช้ identify_by อย่างถูกต้อง
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    # identified_by ทำให้ current_user accessible ทุก channel
  end
end

# 4. Batch broadcasts เมื่อต้องส่งหลาย operations
# ใช้ CableReady หรือ queue broadcasts แล้วส่งพร้อมกัน
```

---

## แบบฝึกหัด {#แบบฝึกหัด}

### ระดับพื้นฐาน

1. **สร้าง Direct Message Room**: เขียน controller action และ view สำหรับสร้างห้องแชทส่วนตัวระหว่างผู้ใช้สองคน โดยตรวจสอบว่าไม่มีห้องซ้ำกัน
2. **Message Pagination**: เพิ่ม infinite scroll ให้กับ message list โดยใช้ Turbo Frames เพื่อโหลดข้อความเก่าเมื่อเลื่อนขึ้นไปถึงด้านบน
3. **Read Receipt**: เพิ่ม feature แสดงว่าข้อความถูกอ่านแล้วหรือยัง (เครื่องหมายถูกสองขีด)

### ระดับกลาง

4. **Message Reactions**: เพิ่มระบบ emoji reactions บนข้อความ (กดไลค์/heart/thumbs up) โดย broadcast การเปลี่ยนแปลงแบบ real-time
5. **Room Search**: สร้าง search functionality สำหรับค้นหาข้อความในห้องแชท พร้อม highlight คำที่ค้นหา
6. **Online Status Persistence**: เก็บ last_seen_at ลงฐานข้อมูลและแสดงว่า "ออนไลน์เมื่อ X นาทีที่แล้ว"

### ระดับสูง

7. **Video Preview**: เมื่อส่งลิงก์ YouTube หรือ Vimeo ในแชท ให้แสดง embed preview อัตโนมัติ
8. **Message Threading**: เพิ่ม thread replies ให้กับข้อความ (คล้าย Slack) โดยแต่ละ thread มี channel เป็นของตัวเอง
9. **E2E Encryption Basics**: ศึกษาการ implement end-to-end encryption อย่างง่ายโดยใช้ Web Crypto API ฝั่ง JavaScript

---

## สรุปสิ่งที่ได้เรียนรู้ {#สรุปสิ่งที่ได้เรียนรู้}

ใน Part 095 นี้ เราได้สร้าง Real-Time Chat Application ที่ครบวงจร โดยได้เรียนรู้สิ่งสำคัญต่อไปนี้:

**ActionCable Architecture**
- การทำงานของ Connection, Channel และ Subscription
- การ authenticate WebSocket connection ผ่าน Devise session
- การใช้ Redis เป็น pub/sub adapter สำหรับ horizontal scaling

**Turbo Streams + Real-time**
- การ broadcast Turbo Stream fragments จาก model callbacks
- การใช้ `stream_for` กับ model objects
- การ append/replace DOM elements แบบ real-time โดยไม่ refresh

**Online Presence System**
- การใช้ Redis Sets เพื่อติดตาม online users
- การ broadcast presence events ผ่าน ActionCable
- การจัดการ heartbeat เพื่อ keep-alive connections

**Background Jobs Integration**
- การสร้าง notifications ผ่าน background job
- การ broadcast metrics จาก Sidekiq job สู่ dashboard
- การ schedule recurring jobs ด้วย sidekiq-cron

**Active Storage Direct Upload**
- การ upload ไฟล์ตรงสู่ S3 ผ่าน browser
- การแสดง progress indicator ด้วย DirectUpload API
- การ render attachments ทั้ง images และ files

**Performance Considerations**
- การเข้าใจ connection limits ของ ActionCable
- การใช้ CableReady สำหรับ advanced DOM operations
- Best practices สำหรับ production-ready WebSocket apps

---

## Preview ของ Part ถัดไป

ใน **Part 096: Capstone 3 ต่อ — Deployment เต็มรูปแบบ + Load Testing เบื้องต้น** เราจะนำแอปนี้ไป deploy จริง ซึ่งจะครอบคลุม:
- Production checklist ก่อน deploy (security headers, secrets rotation)
- การ deploy แอป ActionCable ด้วย Kamal ซึ่งต้องการ sticky sessions
- การตั้งค่า Redis สำหรับ production ทั้ง ActionCable และ Sidekiq
- การ setup SSL/TLS ด้วย Let's Encrypt ผ่าน Kamal
- Load testing ด้วย k6 และการ interpret ผลลัพธ์
- การหา bottleneck และ optimize ให้รองรับ concurrent users ได้มากขึ้น
