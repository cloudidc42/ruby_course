# Part 090: อ่าน Source Code ของ Rails Framework, Contribute Open Source เบื้องต้น

> **Step ครอบคลุมใน Part นี้:** Step 891–900
> **ระดับ:** Phase 15 — Rails Internals & Ecosystem
> **Ruby version:** 3.3.6 | **Rails version:** 8.1.4

ใน Part 089 ที่ผ่านมา เราได้เรียนรู้การสร้างและ publish gem ของตัวเองไปยัง RubyGems.org รวมถึงการบำรุงรักษา gem และการทำ versioning อย่างถูกต้อง บัดนี้เราจะก้าวไปอีกขั้นหนึ่ง — การอ่าน source code ของ Rails framework เอง และการมีส่วนร่วมกับ open source community โดยตรง

การอ่าน source code ของ framework ที่เราใช้งานทุกวันคือทักษะที่แยกแยะ senior developer ออกจาก junior developer ได้อย่างชัดเจน เมื่อคุณเข้าใจว่า Rails ทำงานอย่างไรภายใน คุณจะสามารถ debug ปัญหาที่ซับซ้อนได้เร็วขึ้น เขียน code ที่ idiomatic มากขึ้น และยังสามารถ contribute กลับคืนสู่ community ที่เราทุกคนได้รับประโยชน์มาตลอด

---

## สารบัญ

- [Step 891: ทำไมต้องอ่าน Rails Source Code](#step-891)
- [Step 892: โครงสร้าง Rails Repository บน GitHub](#step-892)
- [Step 893: อ่าน ActiveRecord — find ทำงานอย่างไร](#step-893)
- [Step 894: อ่าน ActionDispatch — Request Routing](#step-894)
- [Step 895: เทคนิคการอ่าน Source Code](#step-895)
- [Step 896: เข้าใจ ActiveSupport::Concern จาก Source](#step-896)
- [Step 897: Fork Rails บน GitHub](#step-897)
- [Step 898: เขียน Fix/Feature ที่ Conform กับ Rails Style](#step-898)
- [Step 899: เปิด Pull Request ไปยัง Rails](#step-899)
- [Step 900: ก้าวต่อไปหลังเรียนจบ Phase 15](#step-900)
- [แบบฝึกหัด](#exercises)
- [สรุปสิ่งที่ได้เรียนรู้](#summary)

---

## Step 891: ทำไมต้องอ่าน Rails Source Code {#step-891}

### ความสำคัญของการอ่าน Source Code

นักพัฒนาส่วนใหญ่ใช้ Rails เป็นเครื่องมือโดยไม่เคยมองเข้าไปข้างใน เหมือนกับคนขับรถที่รู้แค่วิธีเหยียบคันเร่งและเบรค แต่ไม่รู้ว่าเครื่องยนต์ทำงานอย่างไร การอ่าน source code เปลี่ยนคุณจากผู้ใช้เป็นผู้เชี่ยวชาญ

**เหตุผลที่ต้องอ่าน Rails source code:**

**1. เข้าใจ "magic" ที่เกิดขึ้น**

Rails มีหลาย feature ที่ดูเหมือน magic — `before_action`, `has_many`, `validates` เหล่านี้ทำงานอย่างไร? เมื่อคุณอ่าน source code คุณจะเห็นว่าไม่มี magic จริงๆ มีแค่ Ruby metaprogramming ที่เขียนได้ฉลาดมาก

```ruby
# คุณเคยสงสัยไหมว่า has_many ทำงานอย่างไร?
class Post < ApplicationRecord
  has_many :comments  # บรรทัดนี้ทำอะไรกันแน่?
end

# เมื่ออ่าน source ใน activerecord/lib/active_record/associations.rb
# คุณจะพบว่ามันเรียก macro ที่ define method ให้อัตโนมัติ
module ActiveRecord
  module Associations
    extend ActiveSupport::Concern

    module ClassMethods
      def has_many(name, scope = nil, **options, &extension)
        reflection = Builder::HasMany.build(self, name, scope, options, &extension)
        Reflection.add_reflection self, name, reflection
      end
    end
  end
end
```

**2. Debug ได้ลึกกว่า**

เมื่อเกิด error ที่ไม่เข้าใจ การอ่าน source code ช่วยให้คุณตามรอยได้ว่าเกิดอะไรขึ้น

```ruby
# เช่น error นี้หมายความว่าอะไร?
# ActiveRecord::RecordNotFound: Couldn't find User with 'id'=999

# เมื่ออ่าน source ใน activerecord/lib/active_record/relation/finder_methods.rb
# คุณจะเห็นว่า find ทำการ raise error เมื่อไม่พบ record
def find(*ids)
  # ...
  raise RecordNotFound.new(
    "Couldn't find #{name} with '#{primary_key}'=#{ids.first}",
    name, primary_key, ids.first
  )
end
```

**3. เป็น Senior Developer**

Senior developer ไม่ใช่แค่คนที่เขียน code ได้เร็ว แต่คือคนที่เข้าใจระบบอย่างลึกซึ้ง การอ่าน source code ของ framework ที่ใช้งานคือ marker ที่ชัดเจนของระดับความเชี่ยวชาญ

**4. Contribute กลับคืนสู่ Community**

Rails เป็น open source ที่คนทั่วโลกใช้ฟรี การ contribute กลับคืน — ไม่ว่าจะเป็น bug fix, documentation, หรือ feature — คือการตอบแทน community ที่สร้างเครื่องมือที่เราใช้

**5. เรียนรู้ Pattern ที่ดีที่สุด**

Rails repository ประกอบด้วย code ที่ผ่านการ review โดย developer ระดับโลกมานับสิบปี การอ่าน code เหล่านี้คือการเรียนรู้จาก best practice โดยตรง

### จิตใจที่ถูกต้องในการอ่าน Source Code

อย่ากลัวว่า code จะซับซ้อนเกินไป Rails มี codebase ขนาดใหญ่มาก แต่เราไม่ต้องอ่านทั้งหมด — เราอ่านเฉพาะส่วนที่เกี่ยวข้องกับสิ่งที่อยากรู้

```ruby
# กลยุทธ์การอ่าน: เริ่มจากสิ่งที่ใช้บ่อยที่สุด
# 1. เลือก method ที่ใช้ทุกวัน เช่น User.find(1)
# 2. ตามรอยจาก method นั้นเข้าไปใน source
# 3. อ่านเฉพาะ path ที่ code ของคุณเดินผ่าน
# 4. จด note สิ่งที่ค้นพบ
```

---

## Step 892: โครงสร้าง Rails Repository บน GitHub {#step-892}

### Rails Repository คืออะไร

Rails เป็น "monorepo" — repository เดียวที่ประกอบด้วย gem หลายตัว แต่ละ gem ทำหน้าที่เฉพาะด้าน และทำงานร่วมกันเพื่อสร้าง full-stack web framework

URL: https://github.com/rails/rails

### โครงสร้างหลักของ Repository

```
rails/
├── actioncable/          # WebSocket framework (Action Cable)
├── actionmailbox/        # Inbound email processing
├── actionmailer/         # Email composition and delivery
├── actionpack/           # Request/Response handling, routing, controllers
│   ├── lib/
│   │   ├── action_controller/
│   │   ├── action_dispatch/   # ← Routing อยู่ที่นี่
│   │   └── abstract_controller/
│   └── test/
├── actiontext/           # Rich text content (Trix editor)
├── actionview/           # Template rendering, view helpers
├── activejob/            # Background job framework
├── activemodel/          # Model abstractions (validations, callbacks)
├── activerecord/         # ORM (Object-Relational Mapping)
│   ├── lib/
│   │   └── active_record/
│   │       ├── associations/
│   │       ├── callbacks.rb
│   │       ├── connection_adapters/
│   │       ├── relation/
│   │       │   ├── finder_methods.rb  # ← find อยู่ที่นี่
│   │       │   └── query_methods.rb
│   │       └── validations/
│   └── test/
├── activestorage/        # File upload handling
├── activesupport/        # Core extensions, utilities
│   ├── lib/
│   │   └── active_support/
│   │       ├── concern.rb     # ← ActiveSupport::Concern อยู่ที่นี่
│   │       ├── core_ext/      # Extensions to Ruby stdlib
│   │       └── callbacks.rb
│   └── test/
├── railties/             # Rails initializers, generators, config
│   ├── lib/
│   │   └── rails/
│   │       ├── application.rb
│   │       ├── generators/
│   │       └── railtie.rb
│   └── test/
├── guides/               # Rails Guides documentation
├── Gemfile
├── Gemfile.lock
└── CONTRIBUTING.md       # ← อ่านนี้ก่อน contribute!
```

### แต่ละ Component ทำอะไร

```ruby
# ActiveRecord — database interaction
class User < ApplicationRecord  # inherits from ActiveRecord::Base
  has_many :posts
  validates :email, presence: true
end

# ActionPack — HTTP layer (ประกอบด้วย ActionController + ActionDispatch)
class UsersController < ApplicationController  # inherits from ActionController::Base
  def show
    @user = User.find(params[:id])  # params มาจาก ActionDispatch::Request
  end
end

# ActionView — template rendering
# app/views/users/show.html.erb
# <%= @user.name %>

# ActiveSupport — utilities และ extensions
"hello world".camelize   # => "HelloWorld" (core_ext ของ String)
Time.current             # => timezone-aware Time
1.day.ago                # => Time object (Duration extension)

# Railties — ร้อยทุกอย่างเข้าด้วยกัน
# config/application.rb ใช้ Railtie ในการ boot application
```

### วิธีค้นหา Code ใน Repository

```bash
# Clone Rails repository ไว้บนเครื่องเพื่อ search ได้สะดวก
git clone https://github.com/rails/rails.git
cd rails

# ค้นหาด้วย git grep (เร็วกว่า grep ปกติ)
git grep "def find" -- "activerecord/**/*.rb"

# ค้นหาทั้ง repository
git grep "has_many" -- "*.rb" | head -20

# ดู history ของ method ที่สนใจ
git log --oneline -20 -- activerecord/lib/active_record/relation/finder_methods.rb
```

### การใช้ GitHub Search

```
# Search ใน GitHub UI
# 1. ไปที่ https://github.com/rails/rails
# 2. กด 't' เพื่อ fuzzy file search
# 3. กด '.' เพื่อเปิด github.dev (VS Code ใน browser)
# 4. ใช้ Cmd+Shift+F สำหรับ global search

# ตัวอย่าง search query ใน GitHub
repo:rails/rails def find_by language:Ruby
```

---

## Step 893: อ่าน ActiveRecord — find ทำงานอย่างไร {#step-893}

### เริ่มต้นจาก User.find(1)

ทุกวันเราเรียก `User.find(1)` แต่รู้ไหมว่าเกิดอะไรขึ้น? มาตามรอยกัน

```ruby
# จุดเริ่มต้น: app/controllers/users_controller.rb
def show
  @user = User.find(params[:id])
end
```

### ขั้นตอนที่ 1: Class Method `find`

```ruby
# activerecord/lib/active_record/base.rb
# User inherits from ApplicationRecord inherits from ActiveRecord::Base
# ActiveRecord::Base include modules หลายตัว รวมถึง:

module ActiveRecord
  class Base
    include Core
    include Persistence
    include ReadonlyAttributes
    include ModelSchema
    include Inheritance
    include Scoping
    extend FinderMethods  # ← find อยู่ใน module นี้
    # ... และอีกมากมาย
  end
end
```

### ขั้นตอนที่ 2: FinderMethods module

```ruby
# activerecord/lib/active_record/relation/finder_methods.rb

module ActiveRecord
  module FinderMethods
    # find เป็น class method ที่ถูก extend เข้ามา
    def find(*ids)
      # รองรับ find(1), find(1, 2, 3), find([1, 2, 3])
      return to_a.find { |*block_args| yield(*block_args) } if block_given?

      ids = ids.flatten
      ids = ids.first if ids.size == 1

      if ids.is_a?(Array)
        find_some(ids)    # หาหลาย record
      else
        find_one(ids)     # หา record เดียว
      end
    end

    private

    def find_one(id)
      if ActiveRecord::Base === id
        raise ArgumentError,
          "You are passing an instance of ActiveRecord::Base to `find`. " \
          "Please pass the id of the object by calling `.id`."
      end

      # ใช้ where clause สร้าง SQL
      relation = where(primary_key => id)
      record = relation.take

      raise_record_not_found_exception!(id, 0, 1) unless record

      record
    end

    def find_some(ids)
      return find_some_ordered_with_limit(ids) if order_values.empty? && limit_value.nil?

      # สร้าง WHERE id IN (1, 2, 3)
      result = where(primary_key => ids).to_a
      expected_size = ids.uniq.size

      if result.size != expected_size
        # บาง id ไม่พบ → raise error พร้อมระบุว่า id ไหนหาย
        raise_record_not_found_exception!(ids, result.size, expected_size)
      end

      result
    end

    def raise_record_not_found_exception!(ids, result_size, expected_size, key = primary_key, not_found_ids = nil)
      conditions = " [#{arel.where_sql(self.klass.arel_table)}]" unless where_clause.empty?
      name = @klass.name

      if ids.nil?
        error = +"Couldn't find #{name}"
        error << " with#{conditions}" if conditions
        raise RecordNotFound.new(error, name, key)
      elsif Array === ids
        # ... เพิ่มรายละเอียด error
      else
        error = "Couldn't find #{name} with '#{key}'=#{ids}#{conditions}"
        raise RecordNotFound.new(error, name, key, ids)
      end
    end
  end
end
```

### ขั้นตอนที่ 3: SQL Generation ผ่าน Arel

```ruby
# เมื่อเรียก where(id: 1) มันสร้าง Arel AST (Abstract Syntax Tree)
# ซึ่งถูก compile เป็น SQL ในภายหลัง

# ใน relation/query_methods.rb
def where(opts = :chain, *rest)
  if :chain == opts
    WhereChain.new(spawn)
  elsif opts.blank?
    self
  else
    spawn.where!(opts, *rest)
  end
end

def where!(opts, *rest)
  self.where_clause += build_where_clause(opts, rest)
  self
end
```

### การ trace ด้วย pry

```ruby
# เพิ่มใน Gemfile: gem 'pry-rails'
# แล้วใส่ binding.pry ใน controller

def show
  binding.pry  # ← หยุดตรงนี้
  @user = User.find(params[:id])
end

# ใน pry prompt พิมพ์:
# $ User.method(:find)
# => #<Method: Class(ActiveRecord::FinderMethods)#find(...)>
#    activerecord/lib/active_record/relation/finder_methods.rb:68

# show-source User.method(:find)  หรือ $ User.method(:find)
```

---

## Step 894: อ่าน ActionDispatch — Request Routing {#step-894}

### Routes ทำงานอย่างไร

เมื่อ request เข้ามา Rails ต้องตัดสินว่า controller#action ใดที่จะจัดการ routing engine คือกลไกสำคัญตรงนี้

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :users          # สร้าง routes หลายตัวพร้อมกัน
  get '/about', to: 'pages#about'
end
```

### Routing::Mapper — จุดเริ่มต้น

```ruby
# actionpack/lib/action_dispatch/routing/mapper.rb
# นี่คือ DSL ที่เราใช้ใน routes.rb

module ActionDispatch
  module Routing
    class Mapper
      # routes.draw block ถูก instance_eval ใน Mapper instance นี้
      # ดังนั้น self ใน routes.rb block คือ Mapper

      module Resources
        # resources :users เรียก method นี้
        def resources(*resources, &block)
          options = resources.extract_options!.dup

          resources.each do |resource|
            resource = Resource.new(resource.to_s, api_only?, @scope[:shallow], options)

            with_scope_level(:resources, resource) do
              # สร้าง routes สำหรับ index, show, new, edit, create, update, destroy
              yield if block_given?

              with_scope_level(:collection) do
                get  :index if parent_resource.actions.include?(:index)
                post :create if parent_resource.actions.include?(:create)
              end

              with_scope_level(:member) do
                get :edit if parent_resource.actions.include?(:edit)
                get :show if parent_resource.actions.include?(:show)
                # PUT, PATCH, DELETE ก็สร้างที่นี่
              end
            end
          end
        end
      end
    end
  end
end
```

### Route Recognition

```ruby
# actionpack/lib/action_dispatch/routing/route_set.rb
# เมื่อ request มาถึง RouteSet จะ match URL กับ routes ที่ลงทะเบียนไว้

module ActionDispatch
  module Routing
    class RouteSet
      def recognize_path(path, environment = {})
        # แปลง path เป็น hash ของ params
        # เช่น "/users/1" → { controller: "users", action: "show", id: "1" }

        method = (environment[:method] || "GET").to_s.upcase
        path = Journey::Router::Utils.normalize_path(path) unless path.include?("://")

        begin
          env = Rack::MockRequest.env_for(path, method: method)
        rescue URI::InvalidURIError => e
          raise ActionController::RoutingError, e.message
        end

        req = Rails.application.routes.request_class.new(env)
        @router.recognize(req) do |route, params|
          params.merge!(route.defaults)
          params.delete(:controller) if params[:controller].to_s.include?("/")
          return params
        end

        raise ActionController::RoutingError, "No route matches #{path.inspect}"
      end
    end
  end
end
```

### Journey — Rails Routing Engine

```ruby
# Rails ใช้ library ที่ชื่อ Journey สำหรับ route matching
# Journey ใช้ NFA (Non-deterministic Finite Automaton) สำหรับ pattern matching

# เมื่อ request มาถึง:
# 1. Journey::Router#find_routes ถูกเรียก
# 2. ค้นหา route ที่ match ด้วย NFA simulator
# 3. ส่งต่อไปยัง route's dispatcher

# actionpack/lib/action_dispatch/journey/router.rb
module ActionDispatch
  module Journey
    class Router
      def find_routes(req)
        # routes ถูก sort ตาม specificity
        # exact match → wildcard
        @routes.select do |route|
          route.matches?(req)
        end
      end
    end
  end
end
```

### ดู Routes ทั้งหมดของ App

```bash
# ดู routes ทั้งหมด
rails routes

# กรองเฉพาะ routes ที่เกี่ยวกับ users
rails routes -g users

# ดู routes ด้วยรูปแบบ grep
rails routes | grep "GET"

# ใน Rails 8 สามารถ inspect routes ใน console
rails console
app.routes.routes.map { |r| [r.verb, r.path.spec.to_s] }
```

---

## Step 895: เทคนิคการอ่าน Source Code {#step-895}

### เทคนิคที่ 1: bundle open

`bundle open` เปิด source code ของ gem ที่ติดตั้งอยู่ใน editor ที่ตั้งค่าไว้

```bash
# ตั้งค่า editor (ทำครั้งเดียว)
export EDITOR=code   # VS Code
# หรือ
export EDITOR=vim

# เปิด gem source
bundle open activerecord
bundle open activesupport
bundle open actionpack

# เปิด gem ด้วย version ที่ระบุ
gem open activerecord -v 8.1.4
```

### เทคนิคที่ 2: gem which

```bash
# ค้นหา path ของ gem file
gem which activerecord
# => /usr/local/bundle/gems/activerecord-8.1.4/lib/activerecord.rb

# ดู directory ของ gem
gem contents activerecord | head -20

# เปิด directory ทั้งหมดของ gem
gem open activerecord
```

### เทคนิคที่ 3: Method#source_location

`source_location` คือ method ของ Ruby ที่บอกว่า method หนึ่งถูก define ไว้ที่ไหน

```ruby
# ใน Rails console
rails console

# หา method ที่ต้องการ
User.method(:find).source_location
# => ["/usr/local/bundle/gems/activerecord-8.1.4/lib/active_record/relation/finder_methods.rb", 68]

# สำหรับ instance method
user = User.new
user.method(:save).source_location
# => ["/usr/local/bundle/gems/activerecord-8.1.4/lib/active_record/persistence.rb", 165]

# หา method ใน module
String.instance_method(:camelize).source_location
# => ["/usr/local/bundle/gems/activesupport-8.1.4/lib/active_support/core_ext/string/inflections.rb", 66]

# ดู ancestors chain เพื่อเข้าใจ inheritance
User.ancestors
# => [User, ApplicationRecord, ActiveRecord::Base, ..., ActiveRecord::FinderMethods, ...]
```

### เทคนิคที่ 4: pry $ command

pry เป็น REPL ที่ทรงพลังกว่า irb มาก คำสั่ง `$` แสดง source code ได้ทันที

```ruby
# เพิ่ม gem 'pry-rails' ใน Gemfile (development group)
group :development do
  gem 'pry-rails'
  gem 'pry-doc'    # เพิ่มความสามารถในการดู C extension source
end

# ใน pry session:
$ User.find
# From: /usr/local/bundle/gems/activerecord-8.1.4/lib/active_record/relation/finder_methods.rb:68:
# Owner: ActiveRecord::FinderMethods
# Visibility: public
#
# def find(*ids)
#   return to_a.find { |*block_args| yield(*block_args) } if block_given?
#   ...
# end

# ดู documentation
? User.find
# From: /usr/local/bundle/gems/activerecord-8.1.4/lib/active_record/relation/finder_methods.rb
# ...
# Find by id - This can either be a specific id (1), a list of ids (1, 5, 6),
# or an array of ids ([5, 6, 10]).

# navigate ด้วย edit-method
edit-method User.find
```

### เทคนิคที่ 5: puts caller

```ruby
# หา call stack ว่า method ถูกเรียกจากที่ไหน
def my_model_method
  puts caller.first(10).join("\n")
  # แสดง 10 บรรทัดแรกของ call stack
end

# ใช้ Kernel#caller_locations สำหรับข้อมูลที่ structured มากกว่า
def debug_call_chain
  caller_locations.each_with_index do |loc, i|
    puts "#{i}: #{loc.path}:#{loc.lineno} in #{loc.label}"
  end
end
```

### เทคนิคที่ 6: TracePoint

```ruby
# TracePoint ติดตามการเรียก method ได้อย่างละเอียด
tp = TracePoint.new(:call) do |tp|
  if tp.path.include?("active_record")
    puts "#{tp.path}:#{tp.lineno} #{tp.method_id}"
  end
end

tp.enable do
  User.find(1)  # ← trace ทุก method call ที่เกิดขึ้นใน activerecord
end
```

### เทคนิคที่ 7: อ่าน Test Files

Test files มักอ่านง่ายกว่า source code และอธิบาย behavior ได้ชัดเจน

```ruby
# activerecord/test/cases/finder_test.rb
# ดูว่า find ควร behave อย่างไร

class FinderTest < ActiveRecord::TestCase
  def test_find
    assert_equal @first, Topic.find(1)
  end

  def test_find_with_array_of_one
    assert_equal [@first], Topic.find([1])
  end

  def test_find_raises_with_nil
    assert_raises(ActiveRecord::RecordNotFound) { Topic.find(nil) }
  end
end
```

---

## Step 896: เข้าใจ ActiveSupport::Concern จาก Source {#step-896}

### Concern คืออะไร

`ActiveSupport::Concern` แก้ปัญหา module inclusion ใน Ruby ที่ซับซ้อน เมื่อ module include module อื่น chain ของ ClassMethods จะไม่ propagate อย่างที่ต้องการ

```ruby
# ปัญหาที่ Concern แก้:
module Greetable
  def self.included(base)
    base.extend(ClassMethods)  # ต้อง extend ClassMethods เองทุกครั้ง
    # และถ้า Greetable include module อื่น ปัญหาจะซับซ้อนขึ้น
  end

  module ClassMethods
    def greeting
      "Hello from #{name}"
    end
  end

  def greet
    "Hi! I'm #{self.class.name}"
  end
end

# ด้วย Concern:
module Greetable
  extend ActiveSupport::Concern

  included do
    # code ที่รันใน context ของ base class
    validates :name, presence: true
  end

  module ClassMethods
    def greeting
      "Hello from #{name}"
    end
  end

  def greet
    "Hi! I'm #{self.class.name}"
  end
end
```

### Source Code ของ Concern

```ruby
# activesupport/lib/active_support/concern.rb
# นี่คือ source จริง (simplified)

module ActiveSupport
  module Concern
    # Error สำหรับ circular dependency
    class MultipleIncludedBlocks < StandardError
      def initialize
        super "Cannot define multiple 'included' blocks for a Concern"
      end
    end

    def self.extended(base)
      # เมื่อ module ทำ extend ActiveSupport::Concern
      base.instance_variable_set(:@_dependencies, [])
    end

    def append_features(base)
      # นี่คือ hook ที่ Ruby เรียกเมื่อ module ถูก include
      if base.instance_variable_defined?(:@_dependencies)
        # base เป็น module อีกตัวที่ใช้ Concern
        # เพิ่ม self เข้า dependencies แทนที่จะ include ทันที
        base.instance_variable_get(:@_dependencies) << self
        return false
      else
        # base เป็น class ปกติ
        # resolve dependencies ทั้งหมดก่อน
        return false if base < self

        @_dependencies.each { |dep| base.include(dep) }

        super  # include module จริงๆ

        base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)

        # รัน included block ใน context ของ base
        base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
      end
    end

    def included(base = nil, &block)
      if base.nil?
        # เรียกแบบ block: included do ... end
        raise MultipleIncludedBlocks if instance_variable_defined?(:@_included_block)
        @_included_block = block
      else
        # เรียกแบบ method hook (legacy)
        super
      end
    end

    def prepend_features(base)
      # คล้าย append_features แต่สำหรับ prepend
      if base.instance_variable_defined?(:@_dependencies)
        base.instance_variable_get(:@_dependencies).unshift self
        return false
      else
        return false if base < self
        @_dependencies.each { |dep| base.prepend(dep) }
        super
        base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
      end
    end
  end
end
```

### ทดลองใช้ Concern แบบ Custom

```ruby
# lib/concerns/auditable.rb
module Auditable
  extend ActiveSupport::Concern

  included do
    # code นี้รันใน context ของ class ที่ include Auditable
    before_save :set_updated_by
    before_create :set_created_by

    scope :created_by, ->(user) { where(created_by: user.id) }
  end

  module ClassMethods
    def audit_fields
      [:created_by, :updated_by, :created_at, :updated_at]
    end
  end

  # instance methods
  def audit_trail
    "Created by #{created_by} at #{created_at}"
  end

  private

  def set_updated_by
    self.updated_by = Current.user&.id
  end

  def set_created_by
    self.created_by = Current.user&.id
  end
end

# ใช้งาน:
class Post < ApplicationRecord
  include Auditable
  # ได้ before_save, before_create, scope, class methods, และ instance methods ทั้งหมด
end
```

### Concern Dependency Chain

```ruby
# Concern รองรับ dependency ระหว่าง module
module Commentable
  extend ActiveSupport::Concern

  included do
    has_many :comments
  end
end

module Publishable
  extend ActiveSupport::Concern

  include Commentable  # ← Publishable depend on Commentable

  included do
    scope :published, -> { where(published: true) }
  end
end

class Article < ApplicationRecord
  include Publishable
  # Article จะได้ Commentable ด้วยโดยอัตโนมัติ
  # Concern จัดการ dependency order ให้
end
```

---

## Step 897: Fork Rails บน GitHub {#step-897}

### เตรียมตัวก่อน Contribute

ก่อนเขียน code บรรทัดแรก อ่าน CONTRIBUTING.md ของ Rails ให้จบ มันบอกทุกอย่างที่ต้องรู้

```bash
# CONTRIBUTING.md อยู่ที่ root ของ Rails repository
# https://github.com/rails/rails/blob/main/CONTRIBUTING.md

# สิ่งสำคัญที่ต้องรู้จาก CONTRIBUTING.md:
# 1. Bug reports ต้องมี reproducible test case
# 2. Feature requests ควร discuss ใน GitHub Issues ก่อน
# 3. ต้องเขียน tests เสมอ
# 4. ต้อง update CHANGELOG
# 5. Code ต้องผ่าน existing tests ทั้งหมด
```

### ขั้นตอนที่ 1: Fork Repository

```bash
# 1. ไปที่ https://github.com/rails/rails
# 2. คลิก Fork (มุมขวาบน)
# 3. Fork ไปยัง account ของคุณ

# 4. Clone fork ของคุณลงเครื่อง
git clone git@github.com:YOUR_USERNAME/rails.git
cd rails

# 5. เพิ่ม upstream remote เพื่อ sync กับ Rails official
git remote add upstream https://github.com/rails/rails.git

# ตรวจสอบ remotes
git remote -v
# origin    git@github.com:YOUR_USERNAME/rails.git (fetch)
# origin    git@github.com:YOUR_USERNAME/rails.git (push)
# upstream  https://github.com/rails/rails.git (fetch)
# upstream  https://github.com/rails/rails.git (push)
```

### ขั้นตอนที่ 2: Setup Development Environment

```bash
# ติดตั้ง dependencies
cd rails
bundle install

# ติดตั้ง database สำหรับ test (ต้องการ MySQL และ PostgreSQL ด้วย)
# แต่สำหรับเริ่มต้น SQLite ก็พอ

# รัน test ของ activerecord
cd activerecord
bundle exec rake test

# รัน test ไฟล์เดียว
bundle exec ruby -Itest test/cases/finder_test.rb

# รัน test เฉพาะ method
bundle exec ruby -Itest test/cases/finder_test.rb -n test_find
```

### ขั้นตอนที่ 3: สร้าง Branch สำหรับ Feature/Fix

```bash
# Sync กับ upstream ก่อนสร้าง branch ใหม่เสมอ
git fetch upstream
git checkout main
git merge upstream/main

# สร้าง branch ที่มีชื่อ descriptive
git checkout -b fix-finder-with-nil-id
# หรือ
git checkout -b feature-add-pluck-multiple-columns
# หรือ
git checkout -b docs-clarify-has-many-options
```

### ขั้นตอนที่ 4: เขียน Failing Test ก่อน (TDD)

Rails ใช้ TDD approach — เขียน test ที่ fail ก่อน แล้วจึงเขียน code เพื่อให้ test ผ่าน

```ruby
# สมมติว่าพบ bug: User.find("") ไม่ raise error อย่างที่ควรจะเป็น

# เขียน test ใน activerecord/test/cases/finder_test.rb
def test_find_with_empty_string_raises
  assert_raises(ActiveRecord::RecordNotFound) do
    Topic.find("")
  end
end

# รัน test → ควร FAIL ก่อน
bundle exec ruby -Itest test/cases/finder_test.rb -n test_find_with_empty_string_raises
# Expected ActiveRecord::RecordNotFound but nothing was raised.

# ตอนนี้เราพิสูจน์ได้ว่า bug มีอยู่จริง
# และเรามี test ที่จะบอกเราเมื่อ fix สำเร็จ
```

### ขั้นตอนที่ 5: เขียน Fix

```ruby
# แก้ใน activerecord/lib/active_record/relation/finder_methods.rb
def find_one(id)
  if ActiveRecord::Base === id
    raise ArgumentError, "..."
  end

  # เพิ่ม validation สำหรับ empty string
  if id.blank?  # ← เพิ่มบรรทัดนี้
    raise RecordNotFound.new(
      "Couldn't find #{name} without an ID",
      name, primary_key
    )
  end

  relation = where(primary_key => id)
  record = relation.take
  raise_record_not_found_exception!(id, 0, 1) unless record
  record
end
```

```bash
# รัน test อีกครั้ง → ควร PASS แล้ว
bundle exec ruby -Itest test/cases/finder_test.rb -n test_find_with_empty_string_raises
# 1 tests, 1 assertions, 0 failures, 0 errors

# รัน tests ทั้งหมดของ finder เพื่อให้แน่ใจว่าไม่ได้ break อะไร
bundle exec ruby -Itest test/cases/finder_test.rb
# 150 tests, 250 assertions, 0 failures, 0 errors
```

---

## Step 898: เขียน Fix/Feature ที่ Conform กับ Rails Style {#step-898}

### Rails Coding Conventions

Rails มี coding style ที่ชัดเจน ต้องทำตามเพื่อให้ PR ผ่าน review

```ruby
# 1. ใช้ 2-space indentation (ไม่ใช่ tab)
def my_method
  if condition
    do_something
  end
end

# 2. Method names ใช้ snake_case
def find_by_name  # ✓
def findByName    # ✗

# 3. ไม่มี trailing whitespace

# 4. Blank line ระหว่าง methods
def first_method
  # ...
end

def second_method  # ← blank line ระหว่าง methods
  # ...
end

# 5. Comments อธิบาย "why" ไม่ใช่ "what"
# Find the user by email and verify the password
# We use BCrypt comparison to prevent timing attacks
def authenticate(email, password)
  user = find_by(email: email)
  user&.authenticate(password)
end
```

### RuboCop Configuration

```bash
# Rails ใช้ RuboCop สำหรับ style enforcement
# ดู .rubocop.yml ที่ root ของ Rails repository

# รัน RuboCop
bundle exec rubocop activerecord/lib/active_record/relation/finder_methods.rb

# auto-correct style issues
bundle exec rubocop --autocorrect activerecord/lib/active_record/relation/finder_methods.rb
```

### CHANGELOG Format

ทุก change ที่ส่งไปยัง Rails ต้อง update CHANGELOG.md ของ component ที่แก้

```markdown
# activerecord/CHANGELOG.md

*   Raise `ActiveRecord::RecordNotFound` when `find` is called with an empty string.

    Previously, `User.find("")` would execute a SQL query with `WHERE id = ''`
    which returns no results, but the error message was confusing. Now it raises
    immediately with a clear message.

    *Your Name*
```

```bash
# CHANGELOG format:
# *   [description ในรูป present tense]
#
#     [อธิบาย behavior เดิม ถ้าเกี่ยวข้อง]
#
#     [code example ถ้าจำเป็น]
#
#     *Your Name*

# ดูตัวอย่าง CHANGELOG entries ที่มีอยู่แล้วเพื่อเทียบรูปแบบ
head -100 activerecord/CHANGELOG.md
```

### Writing Tests ที่ดี

```ruby
# tests ของ Rails มี pattern ที่ชัดเจน

class FinderTest < ActiveRecord::TestCase
  # ใช้ fixtures สำหรับ test data
  fixtures :topics, :replies

  # ชื่อ test ต้องบอกว่า test อะไร
  def test_find_with_empty_string_raises_record_not_found
    # arrange — ไม่ต้องทำอะไรในกรณีนี้

    # act and assert — รวมกันใน assert_raises
    assert_raises(ActiveRecord::RecordNotFound) do
      Topic.find("")
    end
  end

  # test error message ด้วยถ้าสำคัญ
  def test_find_error_message_contains_model_name
    error = assert_raises(ActiveRecord::RecordNotFound) do
      Topic.find(999999)
    end
    assert_match "Topic", error.message
    assert_match "999999", error.message
  end
end
```

### Documentation

```ruby
# เพิ่ม RDoc comment ถ้าเป็น public API
# Format ของ Rails documentation:

# Finds the first record matching the specified conditions.
#
# There is no implied ordering so if order matters, you should specify it yourself.
#
#   Person.find_by(name: "Spartacus", rating: 4)
#   # SELECT * FROM people WHERE name = 'Spartacus' AND rating = 4 LIMIT 1
#
# Returns +nil+ if no record is found.
def find_by(arg, *args)
  # ...
end
```

---

## Step 899: เปิด Pull Request ไปยัง Rails {#step-899}

### เตรียม Commit ก่อน Push

```bash
# ตรวจสอบ changes ทั้งหมด
git status
git diff

# เพิ่ม files ที่เปลี่ยน
git add activerecord/lib/active_record/relation/finder_methods.rb
git add activerecord/test/cases/finder_test.rb
git add activerecord/CHANGELOG.md

# สร้าง commit ที่มี message ชัดเจน
git commit -m "Raise RecordNotFound when find is called with empty string

Previously User.find('') would execute SQL with WHERE id = '' which
returns no results silently. Now it raises immediately with a clear error.

Fixes #12345"

# Push ไปยัง fork ของเรา
git push origin fix-finder-with-nil-id
```

### PR Description ที่ดี

```markdown
## Summary

Raise `ActiveRecord::RecordNotFound` when `find` is called with an empty string.

## Problem

Currently `User.find("")` executes a SQL query:
```sql
SELECT * FROM users WHERE id = '' LIMIT 1
```

This query returns no records, but the error message is confusing because
it says "Couldn't find User with 'id'=" which doesn't make it clear that
an empty string was passed.

## Solution

Check for blank ID before executing the query and raise with a clear message:
"Couldn't find User without an ID"

## Code Example

Before:
```ruby
User.find("")
# => Raises RecordNotFound: "Couldn't find User with 'id'="
```

After:
```ruby
User.find("")
# => Raises RecordNotFound: "Couldn't find User without an ID"
```

## Tests

Added test case in `test/cases/finder_test.rb` that verifies the behavior.

## Checklist

- [x] Tests added/updated
- [x] CHANGELOG.md updated
- [x] RuboCop passing
- [x] All existing tests still passing
```

### CI ที่ต้อง Pass

```yaml
# Rails ใช้ GitHub Actions สำหรับ CI
# .github/workflows/ci.yml

# Tests ที่ต้องผ่าน:
# 1. Ruby 3.x (multiple versions)
# 2. Multiple database adapters (SQLite, MySQL, PostgreSQL)
# 3. RuboCop
# 4. Documentation checks

# เพื่อ run CI locally ก่อน push:
# SQLite (ง่ายที่สุด):
bundle exec rake test:sqlite3

# PostgreSQL:
bundle exec rake test:postgresql

# MySQL:
bundle exec rake test:mysql2
```

### Review Process

```
# Rails review process:
# 1. Submit PR
# 2. CI runs automatically (ใช้เวลาประมาณ 30-60 นาที)
# 3. Rails Core หรือ Committer review
# 4. อาจมี request for changes
# 5. แก้ตาม feedback
# 6. Merged! (หรือ closed ถ้าไม่ได้รับการยอมรับ)

# Timeline ที่ realistic:
# - Small bug fix: 1-2 สัปดาห์
# - Medium feature: 2-4 สัปดาห์
# - Large feature: อาจนานหลายเดือน หรือถูกปฏิเสธ

# ไม่ต้องท้อถ้า PR ถูกปฏิเสธ:
# - เรียนรู้จาก feedback
# - ปัญหาบางอย่างไม่ใช่ priority ของ Rails team
# - Contribute ต่อไปในเรื่องอื่น
```

### Good First Issues

```bash
# ถ้ายังไม่แน่ใจจะ contribute อะไร ดู issues ที่ label ว่า:
# - "good first issue"
# - "help wanted"
# - "documentation"

# https://github.com/rails/rails/labels/good%20first%20issue

# issues เหล่านี้มักเป็น:
# - Documentation improvements
# - Test improvements
# - Small bug fixes
# - Adding missing error messages
```

---

## Step 900: ก้าวต่อไปหลังเรียนจบ Phase 15 {#step-900}

### Open Source Etiquette

การเป็น open source contributor ที่ดีต้องการมากกว่าแค่ code

```markdown
# หลักการของ Open Source Etiquette:

## 1. Be Patient
Rails maintainer เป็น volunteer ที่มีงานประจำ
อย่า ping PR ทุกวัน รอ 2-4 สัปดาห์ก่อน follow up

## 2. Be Respectful
แม้ PR จะถูกปฏิเสธ ให้ขอบคุณที่ได้รับ feedback
"Thanks for the review! I'll look into the alternative approach you suggested."

## 3. Be Humble
แม้คุณจะคิดว่าวิธีของคุณดีกว่า maintainer อาจมีเหตุผลที่คุณยังไม่รู้
ถามด้วย "Could you help me understand why this approach was chosen?"

## 4. Start Small
อย่าเริ่มด้วย feature ใหญ่ เริ่มจาก documentation fix หรือ small bug fix

## 5. Read the Docs First
CONTRIBUTING.md, CODE_OF_CONDUCT.md ต้องอ่านก่อนเสมอ
```

### Rails Community

```markdown
# ช่องทางการมีส่วนร่วมกับ Rails community:

## Forums และ Discussion
- https://discuss.rubyonrails.org/ — official Rails forum
- https://github.com/rails/rails/discussions — GitHub Discussions

## Chat
- Ruby on Rails Slack (https://www.rubyonrails.link/)
- #rails บน Libera.chat IRC

## Social Media
- @rails บน Twitter/X
- Rails Foundation blog

## Mailing Lists
- rails-core@googlegroups.com — core team discussions (read-only mostly)
```

### Ruby และ Rails Conferences

```markdown
# Conferences ที่ Ruby/Rails developer ควรรู้จัก:

## International
- RailsConf — annual conference สำหรับ Rails community
- RubyConf — broader Ruby community conference
- Brighton Ruby, Euruko — conferences ในยุโรป

## Regional (Asia)
- RubyKaigi — Japan (มักมี talks เกี่ยวกับ Ruby internals ระดับสูง)
- RedDotRubyConf — Singapore

## ประโยชน์ของการไปประชุม:
- พบ maintainer ตัวเป็นๆ
- เรียนรู้ pattern และ technique ใหม่
- Networking กับ developer ระดับโลก
- Talk proposal = เรียนรู้ลึกเพื่อสอนคนอื่น
```

### Technical Blog

```markdown
# การเขียน blog ช่วยพัฒนาตัวเองอย่างมาก:

## ทำไมต้องเขียน blog?
1. Solidify ความเข้าใจ — เมื่อต้องอธิบายให้คนอื่น คุณจะรู้ว่าตัวเองเข้าใจจริงไหม
2. Portfolio — employer มักดู blog และ GitHub contributions
3. Community contribution — คนอื่นได้ประโยชน์จากสิ่งที่คุณเรียนรู้
4. Personal reference — "ฉันเคยแก้ปัญหานี้มาก่อน มันอยู่ใน blog ของฉัน"

## หัวข้อที่ดีสำหรับ Rails blog:
- "I traced through Rails source code to understand X"
- "How I fixed a bug in ActiveRecord"
- "Understanding Concern from first principles"
- "My first Rails contribution: a retrospective"

## Platform ที่นิยม:
- dev.to — ฟรี, community ดี
- medium.com — reach กว้าง
- hashnode.com — ฟรี, custom domain ได้
- เว็บของตัวเอง (Jekyll, Hugo) — full control
```

### เส้นทางสู่ Rails Committer

```markdown
# ถ้าต้องการเป็น Rails Committer (ระยะยาว):

## ขั้นตอน:
1. Contribute อย่างสม่ำเสมอ — ไม่ต้องเยอะ แต่สม่ำเสมอ
2. Review PRs ของคนอื่น — maintainer เห็นว่าคุณเข้าใจ codebase
3. Help บน forums — ตอบคำถาม, อธิบาย behavior
4. Attend RailsConf — พบ core team ตัวต่อตัว
5. เขียน Guides — Rails Guides contributions ได้รับการยอมรับดี

## Timeline ที่ realistic:
- 6 เดือน: PR แรกถูก merge
- 1-2 ปี: เป็น "regular contributor" ที่ core team รู้จักชื่อ
- 3-5 ปี: อาจได้รับการ invite เป็น committer

## Contributors คนดัง:
- Aaron Patterson (tenderlove) — เริ่มจาก nokogiri ไปสู่ Rails core
- Eileen Uchitelle — เริ่มด้วย bug reports ธรรมดา
- เป็นไปได้สำหรับทุกคนที่ทุ่มเท
```

### Learning Path ต่อจาก Phase 15

```ruby
# สิ่งที่ควรเรียนต่อหลังจบ Phase 15:

# 1. Ruby Internals
# - CRuby source code (C)
# - Ruby VM (YARV)
# - GC mechanics
# Resources: "Ruby Under a Microscope" by Pat Shaughnessy

# 2. Database Internals
# - PostgreSQL internals
# - Query planning
# - Index structures (B-tree, GIN, GiST)
# Resources: "Database Internals" by Alex Petrov

# 3. Performance Engineering
# - rack-mini-profiler
# - Memory profiling (memory_profiler gem)
# - CPU profiling (stackprof)
# - N+1 query detection (bullet gem)

# 4. System Design
# - Distributed systems concepts
# - Event sourcing
# - CQRS pattern
# - Microservices vs Monolith tradeoffs

# 5. DevOps สำหรับ Rails
# - Docker + Kubernetes
# - Kamal (Rails deployment tool)
# - Observability (OpenTelemetry)
```

### สรุป Journey ทั้งหมด

```markdown
# คุณมาถึงจุดนี้ได้แล้ว!

Phase 1-5:   Ruby fundamentals → Object-Oriented Ruby
Phase 6-9:   Rails basics → MVC, ActiveRecord, Testing
Phase 10-12: Advanced Rails → APIs, Performance, Security
Phase 13-14: Production Skills → DevOps, Background Jobs
Phase 15:    Rails Internals → Metaprogramming, Gems, Source Code

# ตอนนี้คุณสามารถ:
# ✓ เขียน Ruby code ได้อย่าง idiomatic
# ✓ สร้าง Rails application ที่ production-ready
# ✓ เข้าใจ Rails internals อย่างลึกซึ้ง
# ✓ สร้างและ publish gem ของตัวเอง
# ✓ Contribute กลับคืนสู่ open source

# เส้นทางข้างหน้า:
# Phase 16 → Capstone Projects
# ถึงเวลาสร้างสิ่งที่ยิ่งใหญ่!
```

---

## แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Trace ActiveRecord Callback

อ่าน source code ของ ActiveRecord callbacks และ trace ว่า `before_save` ทำงานอย่างไร

```ruby
# โจทย์:
# 1. ใช้ Method#source_location เพื่อหาว่า before_save defined ที่ไหน
ActiveRecord::Base.method(:before_save).source_location

# 2. อ่าน file นั้นและ trace ไปที่ ActiveSupport::Callbacks
# 3. เขียน comment อธิบาย chain การทำงานของ:
#    before_save → _run_save_callbacks → run_callbacks(:save)

# 4. ทดสอบความเข้าใจด้วย:
class Post < ApplicationRecord
  before_save :log_save_attempt

  private

  def log_save_attempt
    puts "Saving post: #{title}"
    puts caller.first(3).join("\n")  # ← ดู call stack
  end
end

Post.new(title: "Test").save
```

### แบบฝึกหัดที่ 2: Custom Concern พร้อม Test

```ruby
# สร้าง Concern ชื่อ Sluggable ที่:
# - auto-generate slug จาก title ก่อน save
# - มี ClassMethods ที่มี find_by_slug method
# - มี included block ที่ validates uniqueness ของ slug
# - มี test ที่ครอบคลุม behavior ทั้งหมด

# ตัวอย่างการใช้งาน:
class Article < ApplicationRecord
  include Sluggable
  slugged_from :title
end

article = Article.create!(title: "Hello World")
article.slug  # => "hello-world"
Article.find_by_slug("hello-world")  # => article

# เขียน test:
describe Sluggable do
  it "generates slug from title" do
    article = Article.create!(title: "My First Post")
    expect(article.slug).to eq("my-first-post")
  end

  it "handles special characters in title" do
    article = Article.create!(title: "Rails & Ruby: A Love Story")
    expect(article.slug).to eq("rails-ruby-a-love-story")
  end
end
```

### แบบฝึกหัดที่ 3: ค้นหาและอ่าน Source Code

```ruby
# ทำแบบฝึกหัดนี้ใน Rails console:

# 1. หา source location ของ method เหล่านี้:
"hello".method(:camelize).source_location
1.method(:days).source_location
[].method(:sum).source_location

# 2. อ่าน source ของ ActiveRecord::Base#save
# และ trace ว่า validation รันตอนไหน

# 3. หาว่า params hash ใน controller มาจากไหน
# (hint: ActionDispatch::Request)
# method(:params).source_location ใน controller context

# 4. อ่าน ActiveSupport::HashWithIndifferentAccess
# และอธิบายว่าทำไม params[:key] และ params["key"] ถึงได้ผลเหมือนกัน
```

### แบบฝึกหัดที่ 4: Contributing Simulation

```bash
# ทำ exercise นี้บนเครื่องของคุณ:

# 1. Clone Rails repository
git clone https://github.com/rails/rails.git
cd rails

# 2. รัน tests ของ activesupport ให้ผ่าน
cd activesupport
bundle exec rake test

# 3. ค้นหา "good first issue" บน GitHub
# https://github.com/rails/rails/labels/good%20first%20issue

# 4. อ่าน issue และทำความเข้าใจ
# 5. ลองเขียน failing test สำหรับ issue นั้น
# (ยังไม่ต้อง fix จริง แค่ทำความเข้าใจ)

# 6. เขียน PR description draft ว่าคุณจะ approach อย่างไร
```

### แบบฝึกหัดที่ 5: Source Reading Journal

```markdown
# สร้าง "Source Reading Journal" สำหรับ 1 สัปดาห์:

วันละ 1 method ที่ใช้บ่อยใน Rails:

วันที่ 1: ActiveRecord::Base#find_by
วันที่ 2: ActiveRecord::Relation#where
วันที่ 3: String#pluralize (ActiveSupport)
วันที่ 4: ActionController::Base#render
วันที่ 5: ActionView::Helpers::TagHelper#tag
วันที่ 6: ActiveRecord::Callbacks#run_callbacks
วันที่ 7: ActiveRecord::Associations::ClassMethods#has_many

สำหรับแต่ละ method:
1. ใช้ source_location หาว่าอยู่ที่ไหน
2. อ่าน code 50-100 บรรทัดรอบๆ
3. เขียน 3-5 ประโยคอธิบายสิ่งที่ค้นพบ
4. เขียน test เล็กๆ ที่ demonstrate behavior ที่น่าสนใจ
```

---

## สรุปสิ่งที่ได้เรียนรู้ {#summary}

ใน Part 090 นี้ เราได้ก้าวเข้าสู่ระดับที่ลึกที่สุดของ Phase 15 — การอ่าน source code ของ Rails framework เอง และการ contribute กลับคืนสู่ community ที่สร้างเครื่องมือที่เราใช้ทุกวัน

### สิ่งที่เรียนรู้

**Step 891 — ทำไมต้องอ่าน Rails Source Code**
การอ่าน source code เปลี่ยนคุณจากผู้ใช้เป็นผู้เชี่ยวชาญ ช่วยให้ debug ได้ลึก เข้าใจ magic ที่เกิดขึ้น และเรียนรู้ pattern จาก best practices ของ developer ระดับโลก

**Step 892 — โครงสร้าง Rails Repository**
Rails เป็น monorepo ที่ประกอบด้วย gems หลายตัว แต่ละตัวมีหน้าที่ชัดเจน ได้แก่ ActiveRecord (ORM), ActionPack (HTTP layer), ActionView (templates), ActiveSupport (utilities), และ Railties (framework bootstrap)

**Step 893 — อ่าน ActiveRecord find**
ติดตาม `User.find(1)` ผ่าน `ActiveRecord::Base` → `FinderMethods#find` → `find_one` → SQL query ผ่าน Arel AST → database execution

**Step 894 — อ่าน ActionDispatch Routing**
เข้าใจว่า `resources :users` สร้าง routes อย่างไรผ่าน `Routing::Mapper`, และ request matching ทำงานผ่าน Journey routing engine ที่ใช้ NFA

**Step 895 — เทคนิคการอ่าน Source Code**
เครื่องมือสำคัญ: `bundle open`, `gem which`, `Method#source_location`, pry `$` command, `puts caller`, และ `TracePoint`

**Step 896 — ActiveSupport::Concern**
เข้าใจ `append_features`, `included` block, `ClassMethods` module และวิธีที่ Concern แก้ปัญหา module dependency chain

**Step 897 — Fork Rails บน GitHub**
ขั้นตอนการ fork, setup development environment, สร้าง branch, และเขียน failing test ก่อน (TDD approach)

**Step 898 — Rails Style และ Conventions**
Coding style, RuboCop, CHANGELOG format, และวิธีเขียน test และ documentation ที่ Rails team ยอมรับ

**Step 899 — เปิด Pull Request**
การเขียน PR description ที่ดี, CI ที่ต้องผ่าน, review process, และ timeline ที่ realistic

**Step 900 — ก้าวต่อไป**
Open source etiquette, Rails community channels, conferences, technical blogging, และ learning path สู่ Rails committer

### ทักษะที่ได้พัฒนา

```
✓ อ่านและ navigate codebase ขนาดใหญ่ได้อย่างมีประสิทธิภาพ
✓ ใช้ Ruby tooling (pry, TracePoint, source_location) สำหรับ exploration
✓ เข้าใจ internals ของ ActiveRecord, ActionDispatch, ActiveSupport
✓ เข้าใจ Module patterns ที่ Rails ใช้อย่างลึกซึ้ง
✓ รู้ขั้นตอนการ contribute ไปยัง open source project จริง
✓ มี mindset ของ senior developer ที่อ่านก่อน assume
```

### Phase 15 Complete!

Phase 15 ครอบคลุมทักษะ Rails internals ที่สำคัญ:
- **Part 081-083**: Ruby Metaprogramming
- **Part 084-086**: Rails Engines และ Plugins
- **Part 087-088**: Performance Profiling และ Optimization
- **Part 089**: Creating และ Publishing Gems
- **Part 090**: Reading Rails Source Code และ Open Source Contribution

---

## Preview: Phase 16 — Capstone Projects

Phase 16 คือจุดสูงสุดของ course นี้ — ถึงเวลาสร้างโปรเจกต์จริงที่รวมทุกสิ่งที่เรียนมา

### โปรเจกต์ที่จะสร้างใน Phase 16:

**Project A: Full-Stack SaaS Application**
สร้าง web application ที่ production-ready พร้อม authentication, subscription billing (Stripe), background jobs, real-time features (ActionCable), และ admin dashboard

**Project B: Open Source Gem**
สร้าง gem ที่แก้ปัญหาจริงในชีวิตนักพัฒนา Rails พร้อม documentation สมบูรณ์ test coverage สูง และ publish ไปยัง RubyGems.org

**Project C: Rails API + Modern Frontend**
สร้าง REST/GraphQL API ด้วย Rails ที่ integrate กับ frontend framework (Hotwire/Turbo หรือ React/Vue) พร้อม proper authentication และ rate limiting

**Project D: Contribute to Real Open Source**
Fork Rails หรือ gem ที่ใช้งานจริง แก้ bug ที่พบ หรือเพิ่ม feature เล็กๆ แล้วส่ง Pull Request จริงๆ

ทุกโปรเจกต์จะถูก deploy ไปยัง production environment จริง และ code จะผ่าน code review เพื่อให้มั่นใจว่าถึงระดับ professional quality

---

*Part 090 จบสมบูรณ์ | Phase 15: Rails Internals & Ecosystem*
*Ruby 3.3.6 / Rails 8.1.4*
