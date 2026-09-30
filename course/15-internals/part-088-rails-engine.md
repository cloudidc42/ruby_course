# Part 088: Rails Engine — การสร้าง Mountable Engine

> **Step ครอบคลุมใน Part นี้:** Step 871–880
> **ระดับ:** Advanced
> **Ruby version:** 3.3.6
> **Rails version:** 8.1.4

ใน Part 087 เราได้เรียนรู้เกี่ยวกับ Rack และ Middleware ซึ่งเป็นพื้นฐานของ request/response cycle ใน Rails แล้ว ใน Part นี้เราจะยกระดับขึ้นไปอีกขั้น โดยเรียนรู้เรื่อง **Rails Engine** — ซึ่งเป็นกลไกที่ทรงพลังที่สุดอย่างหนึ่งใน Rails ที่ช่วยให้เราสามารถสร้าง mini-Rails application แบบ self-contained แล้ว embed เข้าไปใน application หลักได้ Rails Engine เป็นพื้นฐานของ gems ชื่อดังอย่าง Devise, ActiveAdmin และ Spree ซึ่งทำให้เราสามารถ plug-and-play features ขนาดใหญ่เข้ากับ application ของเราได้อย่างสะดวก

---

## สารบัญ

- [Step 871: Rails Engine คืออะไร](#step-871-rails-engine-คืออะไร)
- [Step 872: สร้าง Engine ใหม่](#step-872-สร้าง-engine-ใหม่)
- [Step 873: Engine Routing](#step-873-engine-routing)
- [Step 874: Models, Controllers, Views ใน Engine](#step-874-models-controllers-views-ใน-engine)
- [Step 875: แชร์ Resources กับ Host App](#step-875-แชร์-resources-กับ-host-app)
- [Step 876: Engine Migrations](#step-876-engine-migrations)
- [Step 877: Asset Pipeline ใน Engine](#step-877-asset-pipeline-ใน-engine)
- [Step 878: Test Engine อย่างเดียว](#step-878-test-engine-อย่างเดียว)
- [Step 879: Publish Engine เป็น Gem](#step-879-publish-engine-เป็น-gem)
- [Step 880: Full Engine vs Mountable Engine vs Isolated Namespace](#step-880-full-engine-vs-mountable-engine-vs-isolated-namespace)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## การเตรียมตัวก่อนเริ่ม

ก่อนเริ่ม Part นี้ ให้แน่ใจว่าคุณมีความเข้าใจพื้นฐานในเรื่องต่อไปนี้:

- Rack Middleware จาก Part 087
- Ruby modules และ namespacing
- Rails routing พื้นฐาน
- Gem development พื้นฐาน

ตรวจสอบ environment:

```bash
ruby --version   # ruby 3.3.6
rails --version  # Rails 8.1.4
```

---

## Step 871: Rails Engine คืออะไร

### ความหมายและแนวคิด

**Rails Engine** คือ mini-Rails application ที่สามารถ embed เข้าไปใน Rails application อื่นได้ พูดง่ายๆ คือ Engine เป็น Rails app ที่ถูกห่อหุ้มให้เป็น library ที่แชร์ได้

ลองคิดภาพแบบนี้: สมมติว่าคุณสร้างระบบ blog ที่ดีมาก และต้องการใช้มันใน application หลายตัว แทนที่จะ copy-paste code คุณสามารถสร้างมันเป็น Engine แล้ว mount เข้าไปใน app แต่ละตัวได้เลย

```
Host Application (my_app)
├── app/
├── config/
├── db/
└── Gemfile
    └── gem 'blog_engine'  <-- Engine ถูก require ตรงนี้

blog_engine (Rails Engine)
├── app/
│   ├── controllers/
│   ├── models/
│   └── views/
├── config/
│   └── routes.rb
└── db/
    └── migrate/
```

### Gems ดังที่ใช้ Rails Engine

Rails Engine ถูกใช้โดย gems ชื่อดังหลายตัว:

**1. Devise** — ระบบ Authentication
```ruby
# Gemfile ของ host app
gem 'devise'

# config/routes.rb
Rails.application.routes.draw do
  devise_for :users  # Devise mount routes ให้เอง
end
```

Devise ใช้ Engine เพื่อ inject controllers, views, และ routes ของระบบ login/logout/register เข้ามาใน app ของเรา

**2. ActiveAdmin** — ระบบ Admin Panel
```ruby
# Gemfile
gem 'activeadmin'

# config/routes.rb
Rails.application.routes.draw do
  ActiveAdmin.routes(self)  # mount admin panel ที่ /admin
end
```

**3. Spree** — ระบบ E-commerce
```ruby
# Gemfile
gem 'spree'
gem 'spree_auth_devise'
gem 'spree_gateway'

# config/routes.rb
Rails.application.routes.draw do
  mount Spree::Core::Engine, at: '/'
end
```

### ทำไมต้องใช้ Rails Engine?

**ข้อดีของ Engine:**

1. **Reusability** — เขียนครั้งเดียว ใช้ได้หลาย app
2. **Isolation** — namespace แยกออกจาก app หลัก ไม่ conflict
3. **Self-contained** — มี routes, models, controllers, views ของตัวเอง
4. **Testable** — test ได้อย่างอิสระผ่าน dummy app
5. **Distributable** — publish เป็น gem แล้วแชร์กับคนอื่นได้

**ตัวอย่าง use case:**
- ระบบ blog ที่ใช้หลาย microservices
- ระบบ authentication ที่มาตรฐาน
- Admin panel สำหรับหลาย projects
- Payment system ที่ reuse ได้
- Feature flags system

### Engine vs Gem ธรรมดา

| คุณสมบัติ | Gem ธรรมดา | Rails Engine |
|-----------|------------|--------------|
| Routes | ไม่มี | มี |
| Controllers | อาจมีหรือไม่มี | มี |
| Views | ไม่มี | มี |
| Migrations | ไม่มี | มี |
| Assets | ไม่มี | มี |
| Mount ใน routes | ไม่ได้ | ได้ |

---

## Step 872: สร้าง Engine ใหม่

### การสร้าง Engine ด้วย Rails Generator

Rails มี generator พิเศษสำหรับสร้าง Engine:

```bash
# สร้าง mountable engine ชื่อ blog_engine
rails plugin new blog_engine --mountable

# หรือถ้าต้องการ full engine
rails plugin new blog_engine --full

# ถ้าต้องการ engine พร้อม database support
rails plugin new blog_engine --mountable --database=sqlite3
```

Flag สำคัญ:
- `--mountable` สร้าง isolated mountable engine (แนะนำ)
- `--full` สร้าง full engine ที่ไม่ isolated namespace
- `--database` กำหนด database adapter

### โครงสร้างโฟลเดอร์ของ Engine

หลังจากรัน `rails plugin new blog_engine --mountable` จะได้โครงสร้างดังนี้:

```
blog_engine/
├── app/
│   ├── assets/
│   │   ├── config/
│   │   │   └── blog_engine_manifest.js
│   │   ├── images/
│   │   │   └── blog_engine/
│   │   ├── javascripts/
│   │   │   └── blog_engine/
│   │   └── stylesheets/
│   │       └── blog_engine/
│   ├── controllers/
│   │   └── blog_engine/
│   │       └── application_controller.rb
│   ├── helpers/
│   │   └── blog_engine/
│   │       └── application_helper.rb
│   ├── jobs/
│   │   └── blog_engine/
│   │       └── application_job.rb
│   ├── mailers/
│   │   └── blog_engine/
│   │       └── application_mailer.rb
│   ├── models/
│   │   └── blog_engine/
│   │       └── application_record.rb
│   └── views/
│       └── layouts/
│           └── blog_engine/
│               └── application.html.erb
├── bin/
│   └── rails
├── config/
│   └── routes.rb
├── db/
│   └── migrate/
├── lib/
│   ├── blog_engine/
│   │   ├── engine.rb         <-- หัวใจของ Engine
│   │   └── version.rb
│   ├── blog_engine.rb
│   └── tasks/
│       └── blog_engine_tasks.rake
├── test/
│   ├── blog_engine_test.rb
│   ├── dummy/                <-- Mini Rails app สำหรับ testing
│   │   ├── app/
│   │   ├── config/
│   │   └── ...
│   └── test_helper.rb
├── blog_engine.gemspec       <-- Gem specification
├── Gemfile
├── MIT-LICENSE
├── Rakefile
└── README.md
```

### ไฟล์สำคัญ: engine.rb

ไฟล์ที่สำคัญที่สุดคือ `lib/blog_engine/engine.rb`:

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
  end
end
```

- `Rails::Engine` — class หลักที่ทำให้ gem นี้กลายเป็น Rails Engine
- `isolate_namespace BlogEngine` — บอกให้ Engine ใช้ namespace แยกจาก host app

### ไฟล์ gemspec

```ruby
# blog_engine.gemspec
require_relative "lib/blog_engine/version"

Gem::Specification.new do |spec|
  spec.name        = "blog_engine"
  spec.version     = BlogEngine::VERSION
  spec.authors     = ["Your Name"]
  spec.email       = ["your@email.com"]
  spec.homepage    = "https://github.com/yourname/blog_engine"
  spec.summary     = "A mountable blog engine for Rails applications"
  spec.description = "Blog engine that can be mounted into any Rails application"
  spec.license     = "MIT"

  spec.metadata["homepage_uri"] = spec.homepage
  spec.metadata["source_code_uri"] = spec.homepage

  spec.files = Dir.chdir(File.expand_path(__dir__)) do
    Dir["{app,config,db,lib}/**/*", "MIT-LICENSE", "Rakefile", "README.md"]
  end

  spec.add_dependency "rails", ">= 8.1.4"
end
```

### version.rb

```ruby
# lib/blog_engine/version.rb
module BlogEngine
  VERSION = "0.1.0"
end
```

### lib/blog_engine.rb (entry point)

```ruby
# lib/blog_engine.rb
require "blog_engine/version"
require "blog_engine/engine"

module BlogEngine
  # ใส่ module-level code ที่นี่
end
```

### Application Controller ใน Engine

```ruby
# app/controllers/blog_engine/application_controller.rb
module BlogEngine
  class ApplicationController < ActionController::Base
    protect_from_forgery with: :exception
  end
end
```

---

## Step 873: Engine Routing

### Mount Engine ใน Host Application

เมื่อสร้าง Engine แล้ว ต้องทำการ "mount" เข้ากับ host application:

```ruby
# config/routes.rb ของ host application
Rails.application.routes.draw do
  # mount Engine ที่ path /blog
  mount BlogEngine::Engine, at: '/blog'
  
  # หรือตั้งชื่อ route helper
  mount BlogEngine::Engine, at: '/blog', as: 'blog'
  
  # routes ของ host app เอง
  root 'home#index'
  resources :users
end
```

ตอนนี้ทุก request ที่ขึ้นต้นด้วย `/blog` จะถูก route ไปยัง Engine

### Routes ภายใน Engine

```ruby
# config/routes.rb ของ Engine
BlogEngine::Engine.routes.draw do
  resources :posts do
    resources :comments
  end
  
  root 'posts#index'
  
  # named routes ใน engine
  get 'archive/:year/:month', to: 'posts#archive', as: :archive
end
```

### Namespace Isolation

เนื่องจากเรา mount engine ที่ `/blog` routes ภายใน engine จะเป็น:
- `/blog` → `BlogEngine::PostsController#index`
- `/blog/posts` → `BlogEngine::PostsController#index`
- `/blog/posts/1` → `BlogEngine::PostsController#show`
- `/blog/posts/1/comments` → `BlogEngine::CommentsController#index`

### การใช้ Route Helpers

ใน Engine ใช้ route helpers ผ่าน `blog_engine` prefix:

```ruby
# ใน Engine controllers/views
blog_engine.posts_path          # => /blog/posts
blog_engine.post_path(@post)    # => /blog/posts/1
blog_engine.root_path           # => /blog

# หรือ
engine_routes = BlogEngine::Engine.routes.url_helpers
engine_routes.posts_path
```

ใน Host Application เข้าถึง Engine routes ผ่าน:

```ruby
# ใน host app controllers/views
blog_engine.posts_path  # /blog/posts (ถ้า mount as: 'blog_engine')
blog.posts_path         # /blog/posts (ถ้า mount as: 'blog')
```

เข้าถึง host app routes จาก Engine:

```ruby
# ใน engine controllers/views
main_app.root_path      # / (root ของ host app)
main_app.users_path     # /users
```

### ตัวอย่าง Controller ใน Engine

```ruby
# app/controllers/blog_engine/posts_controller.rb
module BlogEngine
  class PostsController < ApplicationController
    before_action :set_post, only: [:show, :edit, :update, :destroy]

    def index
      @posts = Post.published.order(created_at: :desc)
    end

    def show
    end

    def new
      @post = Post.new
    end

    def create
      @post = Post.new(post_params)
      
      if @post.save
        redirect_to post_path(@post), notice: 'Post was successfully created.'
      else
        render :new, status: :unprocessable_entity
      end
    end

    def edit
    end

    def update
      if @post.update(post_params)
        redirect_to post_path(@post), notice: 'Post was successfully updated.'
      else
        render :edit, status: :unprocessable_entity
      end
    end

    def destroy
      @post.destroy
      redirect_to posts_path, notice: 'Post was successfully destroyed.'
    end

    private

    def set_post
      @post = Post.find(params[:id])
    end

    def post_params
      params.require(:post).permit(:title, :content, :published_at)
    end
  end
end
```

---

## Step 874: Models, Controllers, Views ใน Engine

### Models ใน Engine — Namespace และ Table Prefix

ใน mountable engine ทุก model จะอยู่ใน namespace ของ engine และมี table prefix:

```ruby
# app/models/blog_engine/application_record.rb
module BlogEngine
  class ApplicationRecord < ActiveRecord::Base
    self.abstract_class = true
  end
end
```

```ruby
# app/models/blog_engine/post.rb
module BlogEngine
  class Post < ApplicationRecord
    # table name จะเป็น "blog_engine_posts" โดยอัตโนมัติ
    # เพราะ Rails เพิ่ม engine namespace เป็น prefix
    
    validates :title, presence: true, length: { minimum: 3, maximum: 200 }
    validates :content, presence: true
    
    belongs_to :author, class_name: 'BlogEngine::Author', optional: true
    has_many :comments, class_name: 'BlogEngine::Comment', dependent: :destroy
    has_many :taggings, class_name: 'BlogEngine::Tagging', dependent: :destroy
    has_many :tags, through: :taggings, class_name: 'BlogEngine::Tag'
    
    scope :published, -> { where.not(published_at: nil).where('published_at <= ?', Time.current) }
    scope :draft, -> { where(published_at: nil) }
    scope :recent, -> { order(created_at: :desc) }
    
    def published?
      published_at.present? && published_at <= Time.current
    end
    
    def publish!
      update!(published_at: Time.current)
    end
    
    def excerpt(length = 200)
      content.truncate(length)
    end
  end
end
```

### Table Naming Convention

เมื่อใช้ `isolate_namespace BlogEngine` Rails จะใช้ convention ดังนี้:

| Model | Table Name |
|-------|------------|
| `BlogEngine::Post` | `blog_engine_posts` |
| `BlogEngine::Comment` | `blog_engine_comments` |
| `BlogEngine::Tag` | `blog_engine_tags` |
| `BlogEngine::Author` | `blog_engine_authors` |

ถ้าต้องการกำหนด table name เองทำได้แบบนี้:

```ruby
module BlogEngine
  class Post < ApplicationRecord
    self.table_name = 'blog_posts'  # override default
  end
end
```

### Controllers ใน Engine

```ruby
# app/controllers/blog_engine/comments_controller.rb
module BlogEngine
  class CommentsController < ApplicationController
    before_action :set_post
    before_action :set_comment, only: [:show, :edit, :update, :destroy]

    def index
      @comments = @post.comments.order(created_at: :asc)
    end

    def create
      @comment = @post.comments.build(comment_params)
      
      if @comment.save
        redirect_to blog_engine.post_path(@post),
                    notice: 'Comment was successfully created.'
      else
        redirect_to blog_engine.post_path(@post),
                    alert: @comment.errors.full_messages.join(', ')
      end
    end

    def destroy
      @comment.destroy
      redirect_to blog_engine.post_path(@post),
                  notice: 'Comment was successfully destroyed.'
    end

    private

    def set_post
      @post = Post.find(params[:post_id])
    end

    def set_comment
      @comment = @post.comments.find(params[:id])
    end

    def comment_params
      params.require(:comment).permit(:body, :author_name, :author_email)
    end
  end
end
```

### Views ใน Engine

Views ก็ต้องอยู่ใน subfolder ที่มีชื่อ namespace:

```
app/views/blog_engine/
├── layouts/
│   └── blog_engine/
│       └── application.html.erb
├── posts/
│   ├── index.html.erb
│   ├── show.html.erb
│   ├── new.html.erb
│   ├── edit.html.erb
│   └── _form.html.erb
└── comments/
    ├── index.html.erb
    └── _comment.html.erb
```

```erb
<%# app/views/blog_engine/posts/index.html.erb %>
<div class="blog-engine">
  <h1>Blog Posts</h1>
  
  <%= link_to 'New Post', new_post_path, class: 'btn btn-primary' %>
  
  <% @posts.each do |post| %>
    <article class="post">
      <h2><%= link_to post.title, post_path(post) %></h2>
      <p class="meta">
        Published: <%= post.published_at&.strftime('%B %d, %Y') || 'Draft' %>
        | Comments: <%= post.comments.count %>
      </p>
      <p><%= post.excerpt %></p>
      <%= link_to 'Read More', post_path(post), class: 'btn btn-link' %>
    </article>
  <% end %>
</div>
```

```erb
<%# app/views/blog_engine/posts/show.html.erb %>
<div class="blog-engine">
  <article>
    <h1><%= @post.title %></h1>
    <p class="meta">
      <%= @post.published_at&.strftime('%B %d, %Y') %>
    </p>
    <div class="content">
      <%= simple_format @post.content %>
    </div>
  </article>
  
  <section class="comments">
    <h2>Comments (<%= @post.comments.count %>)</h2>
    
    <% @post.comments.each do |comment| %>
      <%= render 'comments/comment', comment: comment %>
    <% end %>
    
    <h3>Leave a Comment</h3>
    <%= render 'comments/form', post: @post, comment: BlogEngine::Comment.new %>
  </section>
  
  <%= link_to 'Back to Posts', posts_path %>
  
  <%# เข้าถึง route ของ host app %>
  <%= link_to 'Home', main_app.root_path %>
</div>
```

### Layout ของ Engine

Engine มี layout ของตัวเองแยกต่างหาก:

```erb
<%# app/views/layouts/blog_engine/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title>Blog Engine</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    
    <%# include engine assets %>
    <%= stylesheet_link_tag "blog_engine/application", media: "all" %>
    <%= javascript_include_tag "blog_engine/application", "data-turbo-track": "reload", defer: true %>
  </head>

  <body>
    <nav>
      <%= link_to 'Blog Home', blog_engine.root_path %>
      <%= link_to 'Main Site', main_app.root_path %>
    </nav>
    
    <% if notice.present? %>
      <div class="alert alert-success"><%= notice %></div>
    <% end %>
    
    <% if alert.present? %>
      <div class="alert alert-danger"><%= alert %></div>
    <% end %>
    
    <%= yield %>
  </body>
</html>
```

---

## Step 875: แชร์ Resources กับ Host App

### การ Hook เข้า Host Application

บางครั้ง Engine ต้องการใช้ resources จาก host app เช่น User model หรือ ApplicationController ของ host:

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    # กำหนด initializer ที่รันเมื่อ Rails boot
    initializer "blog_engine.action_controller" do
      ActiveSupport.on_load(:action_controller) do
        # inject helper methods เข้าทุก controller
        include BlogEngine::ControllerHelper
      end
    end
    
    # กำหนด generators defaults
    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
      g.assets false
      g.helper false
    end
    
    # เพิ่ม load paths
    config.autoload_paths += %W(
      #{config.root}/lib
    )
  end
end
```

### การ Hook เข้า ApplicationController

วิธีที่ดีที่สุดในการทำให้ Engine ใช้ authentication จาก host app:

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    # สร้าง mattr_accessor เพื่อรับ config จาก host app
    mattr_accessor :parent_controller
    self.parent_controller = 'ApplicationController'  # default
    
    mattr_accessor :current_user_method
    self.current_user_method = :current_user  # default
  end
end
```

```ruby
# app/controllers/blog_engine/application_controller.rb
module BlogEngine
  class ApplicationController < BlogEngine::Engine.parent_controller.constantize
    # ตอนนี้ Engine ใช้ ApplicationController ของ host app เป็น parent
    # ทำให้ inherit methods ทั้งหมด รวมถึง authenticate_user!
    
    protect_from_forgery with: :exception
    
    # helper method ที่ใช้ current_user จาก host app
    def current_blog_user
      send(BlogEngine::Engine.current_user_method)
    end
    
    helper_method :current_blog_user
  end
end
```

การ configure ใน host app:

```ruby
# config/initializers/blog_engine.rb
BlogEngine::Engine.tap do |engine|
  engine.parent_controller = 'ApplicationController'
  engine.current_user_method = :current_user
end
```

### การแชร์ Model จาก Host App

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    mattr_accessor :user_class
    self.user_class = 'User'
  end
end
```

```ruby
# app/models/blog_engine/post.rb
module BlogEngine
  class Post < ApplicationRecord
    # dynamic association กับ User model ของ host app
    def author_class
      BlogEngine::Engine.user_class.constantize
    end
    
    # ไม่ใช้ belongs_to เพราะ model อยู่คนละ namespace
    def author
      author_class.find_by(id: author_id) if author_id
    end
  end
end
```

### Railtie และ Initializers

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    # initializer สำหรับ load default config
    initializer 'blog_engine.defaults', before: :load_config_initializers do
      config.blog_engine = ActiveSupport::OrderedOptions.new
      config.blog_engine.posts_per_page = 10
      config.blog_engine.enable_comments = true
      config.blog_engine.moderate_comments = false
    end
    
    # initializer สำหรับ assets
    initializer 'blog_engine.assets.precompile' do |app|
      app.config.assets.precompile += %w[
        blog_engine/application.css
        blog_engine/application.js
      ]
    end
    
    # initializer สำหรับ helpers
    initializer 'blog_engine.helpers' do
      ActiveSupport.on_load(:action_view) do
        include BlogEngine::ApplicationHelper
      end
    end
  end
end
```

การใช้ config ใน host app:

```ruby
# config/application.rb หรือ config/initializers/blog_engine.rb
Rails.application.config.blog_engine.posts_per_page = 15
Rails.application.config.blog_engine.enable_comments = false
```

---

## Step 876: Engine Migrations

### สร้าง Migrations ใน Engine

```bash
# อยู่ใน engine directory
cd blog_engine

# สร้าง migration
rails generate migration CreateBlogEnginePosts title:string content:text published_at:datetime
rails generate migration CreateBlogEngineComments post:references body:text author_name:string author_email:string
rails generate migration CreateBlogEngineTags name:string slug:string
```

Migration file ที่ได้:

```ruby
# db/migrate/20240101000001_create_blog_engine_posts.rb
class CreateBlogEnginePosts < ActiveRecord::Migration[8.1]
  def change
    create_table :blog_engine_posts do |t|
      t.string :title, null: false
      t.text :content, null: false
      t.datetime :published_at
      t.integer :author_id
      t.string :author_type
      t.integer :views_count, default: 0
      
      t.timestamps
    end
    
    add_index :blog_engine_posts, :published_at
    add_index :blog_engine_posts, :author_id
    add_index :blog_engine_posts, [:author_type, :author_id]
  end
end
```

```ruby
# db/migrate/20240101000002_create_blog_engine_comments.rb
class CreateBlogEngineComments < ActiveRecord::Migration[8.1]
  def change
    create_table :blog_engine_comments do |t|
      t.references :post, null: false, foreign_key: { to_table: :blog_engine_posts }
      t.text :body, null: false
      t.string :author_name
      t.string :author_email
      t.boolean :approved, default: false
      
      t.timestamps
    end
    
    add_index :blog_engine_comments, :approved
  end
end
```

### Copy Migrations ไปยัง Host App

Host application ต้อง copy migrations จาก engine ก่อน:

```bash
# จาก host application directory
# คำสั่งมาตรฐาน Rails
rake blog_engine:install:migrations

# หรือกับ Rails 8.1
bin/rails blog_engine:install:migrations
```

คำสั่งนี้จะ copy migrations ทั้งหมดจาก engine ไปยัง `db/migrate/` ของ host app

ตัวอย่าง output:

```
Copied migration 20240101000001_create_blog_engine_posts.blog_engine.rb from blog_engine
Copied migration 20240101000002_create_blog_engine_comments.blog_engine.rb from blog_engine
Copied migration 20240101000003_create_blog_engine_tags.blog_engine.rb from blog_engine
```

ไฟล์ที่ copy จะมี suffix `.blog_engine.rb` เพื่อบอกว่ามาจาก engine

```bash
# รัน migrations
bin/rails db:migrate
```

### Task สำหรับ Install

สร้าง rake task ใน engine เพื่อ automate การ setup:

```ruby
# lib/tasks/blog_engine_tasks.rake
namespace :blog_engine do
  desc "Install BlogEngine migrations and run them"
  task install: :environment do
    Rake::Task["blog_engine:install:migrations"].invoke
    Rake::Task["db:migrate"].invoke
  end
  
  desc "Generate default configuration file"
  task config: :environment do
    config_path = Rails.root.join('config', 'initializers', 'blog_engine.rb')
    
    unless File.exist?(config_path)
      File.write(config_path, <<~RUBY)
        # BlogEngine Configuration
        Rails.application.config.blog_engine.posts_per_page = 10
        Rails.application.config.blog_engine.enable_comments = true
        Rails.application.config.blog_engine.moderate_comments = false
      RUBY
      puts "Created #{config_path}"
    else
      puts "Config file already exists at #{config_path}"
    end
  end
end
```

### Generator สำหรับ Install

สร้าง install generator เพื่อทำทุกอย่างในครั้งเดียว:

```ruby
# lib/generators/blog_engine/install/install_generator.rb
module BlogEngine
  module Generators
    class InstallGenerator < Rails::Generators::Base
      source_root File.expand_path('templates', __dir__)
      
      desc "Install BlogEngine into your Rails application"
      
      def mount_engine
        route "mount BlogEngine::Engine, at: '/blog'"
      end
      
      def copy_initializer
        template 'initializer.rb', 'config/initializers/blog_engine.rb'
      end
      
      def install_migrations
        rake "blog_engine:install:migrations"
      end
      
      def show_readme
        readme 'README'
      end
    end
  end
end
```

```ruby
# lib/generators/blog_engine/install/templates/initializer.rb
# BlogEngine Configuration
# See https://github.com/yourname/blog_engine for full documentation

Rails.application.config.tap do |config|
  # จำนวน posts ต่อหน้า
  config.blog_engine.posts_per_page = 10
  
  # เปิด/ปิด comments
  config.blog_engine.enable_comments = true
  
  # ต้องอนุมัติ comment ก่อน publish
  config.blog_engine.moderate_comments = false
end
```

ใช้งานใน host app:

```bash
rails generate blog_engine:install
```

---

## Step 877: Asset Pipeline ใน Engine

### โครงสร้าง Assets ใน Engine

```
app/assets/
├── config/
│   └── blog_engine_manifest.js    <-- Asset manifest
├── images/
│   └── blog_engine/
│       └── default_avatar.png
├── javascripts/
│   └── blog_engine/
│       ├── application.js
│       └── posts.js
└── stylesheets/
    └── blog_engine/
        ├── application.css
        ├── posts.css
        └── comments.css
```

### Manifest File

```js
// app/assets/config/blog_engine_manifest.js
//= link_tree ../images
//= link blog_engine/application.css
//= link blog_engine/application.js
```

### CSS ใน Engine

```css
/* app/assets/stylesheets/blog_engine/application.css */
/*
 *= require_tree .
 *= require_self
 */

/* Base styles สำหรับ blog engine */
.blog-engine {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: Georgia, serif;
}
```

```css
/* app/assets/stylesheets/blog_engine/posts.css */
.post {
  margin-bottom: 30px;
  padding-bottom: 30px;
  border-bottom: 1px solid #eee;
}

.post h2 {
  font-size: 1.5rem;
  margin-bottom: 10px;
}

.post .meta {
  color: #666;
  font-size: 0.9rem;
  margin-bottom: 15px;
}

.post .content {
  line-height: 1.7;
}
```

### JavaScript ใน Engine

```js
// app/assets/javascripts/blog_engine/application.js
// This is a manifest file that'll be compiled into application.js, which will include all the files
// listed below.
//
//= require_tree .

// BlogEngine namespace
const BlogEngine = {};
```

```js
// app/assets/javascripts/blog_engine/posts.js
BlogEngine.Posts = {
  init: function() {
    this.bindEvents();
  },
  
  bindEvents: function() {
    document.querySelectorAll('.post-delete-btn').forEach(btn => {
      btn.addEventListener('click', this.confirmDelete.bind(this));
    });
  },
  
  confirmDelete: function(event) {
    if (!confirm('Are you sure you want to delete this post?')) {
      event.preventDefault();
    }
  }
};

document.addEventListener('DOMContentLoaded', () => {
  BlogEngine.Posts.init();
});
```

### ประกาศ Assets ใน Engine Initializer

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    initializer 'blog_engine.assets.precompile' do |app|
      app.config.assets.precompile += %w[
        blog_engine/application.css
        blog_engine/application.js
        blog_engine/default_avatar.png
      ]
    end
    
    # เพิ่ม engine assets path
    initializer 'blog_engine.assets.paths' do |app|
      app.config.assets.paths += config.assets.paths
    end
  end
end
```

### Importmap และ Modern JavaScript

สำหรับ Rails 8.1 ที่ใช้ importmap:

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
    
    # เพิ่ม JavaScript files เข้า importmap
    initializer 'blog_engine.importmap', before: 'importmap' do |app|
      if app.config.respond_to?(:importmap)
        app.config.importmap.paths << root.join('config/importmap.rb')
        app.config.importmap.cache_sweepers << root.join('app/assets/javascripts/blog_engine')
      end
    end
  end
end
```

```ruby
# config/importmap.rb (ของ engine)
pin "blog_engine", to: "blog_engine/application.js"
pin "blog_engine/posts", to: "blog_engine/posts.js"
```

---

## Step 878: Test Engine อย่างเดียว

### Dummy App ใน Engine

เมื่อสร้าง engine ด้วย `rails plugin new` Rails จะสร้าง `test/dummy` app ให้เราอัตโนมัติ นี่คือ Rails application ขนาดเล็กที่ใช้เฉพาะสำหรับ test:

```
test/dummy/
├── app/
│   ├── controllers/
│   │   └── application_controller.rb
│   └── models/
│       └── user.rb            <-- User model สำหรับ test
├── config/
│   ├── application.rb
│   ├── database.yml
│   ├── environment.rb
│   └── routes.rb
└── db/
    └── schema.rb
```

```ruby
# test/dummy/config/routes.rb
Rails.application.routes.draw do
  mount BlogEngine::Engine => '/blog'
end
```

```ruby
# test/dummy/app/models/user.rb
class User < ApplicationRecord
  # Mock User model สำหรับ test authentication
end
```

### Setup RSpec ใน Engine

```bash
# เพิ่ม gems ใน gemspec
spec.add_development_dependency 'rspec-rails', '~> 6.0'
spec.add_development_dependency 'factory_bot_rails'
spec.add_development_dependency 'capybara'
spec.add_development_dependency 'selenium-webdriver'
```

```ruby
# spec/rails_helper.rb ของ engine
ENV['RAILS_ENV'] ||= 'test'

# load dummy app แทน host app
require File.expand_path('../test/dummy/config/environment', __dir__)

require 'rspec/rails'
require 'factory_bot_rails'
require 'capybara/rspec'

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.use_transactional_fixtures = true
  
  config.before(:suite) do
    FactoryBot.find_definitions
  end
end
```

### เขียน Model Tests

```ruby
# spec/models/blog_engine/post_spec.rb
require 'rails_helper'

module BlogEngine
  RSpec.describe Post, type: :model do
    subject(:post) { build(:blog_engine_post) }
    
    describe 'validations' do
      it { is_expected.to validate_presence_of(:title) }
      it { is_expected.to validate_presence_of(:content) }
      it { is_expected.to validate_length_of(:title).is_at_least(3).is_at_most(200) }
    end
    
    describe 'associations' do
      it { is_expected.to have_many(:comments).dependent(:destroy) }
      it { is_expected.to have_many(:taggings).dependent(:destroy) }
      it { is_expected.to have_many(:tags).through(:taggings) }
    end
    
    describe 'scopes' do
      describe '.published' do
        it 'returns only published posts' do
          published_post = create(:blog_engine_post, :published)
          draft_post = create(:blog_engine_post)
          
          expect(Post.published).to include(published_post)
          expect(Post.published).not_to include(draft_post)
        end
      end
      
      describe '.recent' do
        it 'orders by created_at desc' do
          old_post = create(:blog_engine_post, created_at: 1.day.ago)
          new_post = create(:blog_engine_post, created_at: 1.hour.ago)
          
          expect(Post.recent.first).to eq(new_post)
          expect(Post.recent.last).to eq(old_post)
        end
      end
    end
    
    describe '#published?' do
      context 'when published_at is set and in the past' do
        it 'returns true' do
          post = build(:blog_engine_post, published_at: 1.day.ago)
          expect(post.published?).to be true
        end
      end
      
      context 'when published_at is nil' do
        it 'returns false' do
          post = build(:blog_engine_post, published_at: nil)
          expect(post.published?).to be false
        end
      end
      
      context 'when published_at is in the future' do
        it 'returns false' do
          post = build(:blog_engine_post, published_at: 1.day.from_now)
          expect(post.published?).to be false
        end
      end
    end
    
    describe '#publish!' do
      it 'sets published_at to current time' do
        post = create(:blog_engine_post)
        
        expect { post.publish! }.to change { post.published_at }.from(nil)
        expect(post.published_at).to be_within(1.second).of(Time.current)
      end
    end
  end
end
```

### เขียน Controller Tests

```ruby
# spec/controllers/blog_engine/posts_controller_spec.rb
require 'rails_helper'

module BlogEngine
  RSpec.describe PostsController, type: :controller do
    routes { BlogEngine::Engine.routes }
    
    describe 'GET #index' do
      it 'returns http success' do
        get :index
        expect(response).to have_http_status(:success)
      end
      
      it 'assigns @posts with published posts only' do
        published_post = create(:blog_engine_post, :published)
        draft_post = create(:blog_engine_post)
        
        get :index
        
        expect(assigns(:posts)).to include(published_post)
        expect(assigns(:posts)).not_to include(draft_post)
      end
    end
    
    describe 'GET #show' do
      let(:post) { create(:blog_engine_post, :published) }
      
      it 'returns http success' do
        get :show, params: { id: post.id }
        expect(response).to have_http_status(:success)
      end
      
      it 'assigns the requested post' do
        get :show, params: { id: post.id }
        expect(assigns(:post)).to eq(post)
      end
    end
    
    describe 'POST #create' do
      context 'with valid params' do
        let(:valid_params) do
          { post: { title: 'Test Post', content: 'Test Content' } }
        end
        
        it 'creates a new post' do
          expect { post :create, params: valid_params }.to change(Post, :count).by(1)
        end
        
        it 'redirects to the created post' do
          post :create, params: valid_params
          expect(response).to redirect_to(post_path(Post.last))
        end
      end
      
      context 'with invalid params' do
        let(:invalid_params) do
          { post: { title: '', content: '' } }
        end
        
        it 'does not create a new post' do
          expect { post :create, params: invalid_params }.not_to change(Post, :count)
        end
        
        it 'renders the new template' do
          post :create, params: invalid_params
          expect(response).to render_template(:new)
        end
      end
    end
  end
end
```

### Factories สำหรับ Engine

```ruby
# spec/factories/blog_engine/posts.rb
FactoryBot.define do
  factory :blog_engine_post, class: 'BlogEngine::Post' do
    title { "Test Post #{SecureRandom.hex(4)}" }
    content { 'This is test content for the blog post.' * 3 }
    published_at { nil }
    
    trait :published do
      published_at { 1.hour.ago }
    end
    
    trait :draft do
      published_at { nil }
    end
    
    trait :scheduled do
      published_at { 1.day.from_now }
    end
    
    trait :with_comments do
      after(:create) do |post|
        create_list(:blog_engine_comment, 3, post: post)
      end
    end
  end
end
```

```ruby
# spec/factories/blog_engine/comments.rb
FactoryBot.define do
  factory :blog_engine_comment, class: 'BlogEngine::Comment' do
    association :post, factory: :blog_engine_post, :published
    body { 'This is a test comment.' }
    author_name { 'Test Author' }
    author_email { 'test@example.com' }
    approved { true }
  end
end
```

### Integration Tests ด้วย Capybara

```ruby
# spec/features/blog_engine/posts_spec.rb
require 'rails_helper'

RSpec.feature 'Blog Posts', type: :feature do
  scenario 'visitor views published posts' do
    post1 = create(:blog_engine_post, :published, title: 'First Post')
    post2 = create(:blog_engine_post, :published, title: 'Second Post')
    draft = create(:blog_engine_post, title: 'Draft Post')
    
    visit blog_engine.posts_path
    
    expect(page).to have_content('First Post')
    expect(page).to have_content('Second Post')
    expect(page).not_to have_content('Draft Post')
  end
  
  scenario 'visitor reads a post' do
    post = create(:blog_engine_post, :published,
                  title: 'Hello World',
                  content: 'This is the post content')
    
    visit blog_engine.post_path(post)
    
    expect(page).to have_content('Hello World')
    expect(page).to have_content('This is the post content')
  end
end
```

---

## Step 879: Publish Engine เป็น Gem

### เตรียม Gemspec

```ruby
# blog_engine.gemspec
require_relative "lib/blog_engine/version"

Gem::Specification.new do |spec|
  spec.name        = "blog_engine"
  spec.version     = BlogEngine::VERSION
  spec.authors     = ["Your Name"]
  spec.email       = ["your@email.com"]
  
  spec.summary     = "A mountable blog engine for Rails 8.1+"
  spec.description = <<~DESC
    BlogEngine is a full-featured mountable Rails Engine that provides
    blog functionality including posts, comments, tags, and RSS feeds.
    Mount it into any Rails 8.1+ application.
  DESC
  
  spec.homepage    = "https://github.com/yourname/blog_engine"
  spec.license     = "MIT"
  
  spec.required_ruby_version = ">= 3.3.0"
  
  spec.metadata = {
    "homepage_uri"    => spec.homepage,
    "source_code_uri" => "#{spec.homepage}/tree/v#{spec.version}",
    "changelog_uri"   => "#{spec.homepage}/blob/main/CHANGELOG.md",
    "bug_tracker_uri" => "#{spec.homepage}/issues",
    "documentation_uri" => "https://rubydoc.info/gems/blog_engine/#{spec.version}"
  }
  
  # ไฟล์ที่จะ include ใน gem
  spec.files = Dir.chdir(File.expand_path(__dir__)) do
    Dir[
      "{app,config,db,lib}/**/*",
      "MIT-LICENSE",
      "Rakefile",
      "README.md",
      "CHANGELOG.md"
    ].reject { |f| File.directory?(f) }
  end
  
  # ไม่ include test files ใน gem
  spec.test_files = []
  
  # Runtime dependencies
  spec.add_dependency "rails", ">= 8.1.4"
  
  # Development dependencies
  spec.add_development_dependency "sqlite3", "~> 2.0"
  spec.add_development_dependency "rspec-rails", "~> 6.0"
  spec.add_development_dependency "factory_bot_rails", "~> 6.0"
  spec.add_development_dependency "capybara", "~> 3.0"
  spec.add_development_dependency "simplecov", "~> 0.22"
end
```

### Versioning

ทำตาม Semantic Versioning (semver.org):

```ruby
# lib/blog_engine/version.rb
module BlogEngine
  # MAJOR.MINOR.PATCH
  # MAJOR: breaking changes
  # MINOR: new features, backward compatible
  # PATCH: bug fixes
  VERSION = "1.2.3"
end
```

### Build และ Push Gem

```bash
# build gem
gem build blog_engine.gemspec
# สร้างไฟล์ blog_engine-1.2.3.gem

# test gem locally ก่อน
gem install ./blog_engine-1.2.3.gem

# push ไป RubyGems.org
gem push blog_engine-1.2.3.gem

# ถ้าต้องการ login ก่อน
gem signin
```

### ใช้ Engine จาก RubyGems ใน Host App

```ruby
# Gemfile ของ host application
source 'https://rubygems.org'

gem 'rails', '~> 8.1.4'

# ใช้ engine จาก RubyGems
gem 'blog_engine', '~> 1.2'

# หรือใช้ engine จาก GitHub (development)
gem 'blog_engine', github: 'yourname/blog_engine', branch: 'main'

# หรือใช้จาก local path (development)
gem 'blog_engine', path: '../blog_engine'
```

```bash
bundle install
rails generate blog_engine:install
rails db:migrate
```

### CHANGELOG

```markdown
# Changelog

All notable changes to BlogEngine will be documented in this file.

## [1.2.3] - 2024-01-15

### Fixed
- Fixed comment pagination bug
- Fixed CSS styling on mobile

## [1.2.0] - 2024-01-10

### Added
- Added RSS feed support
- Added post scheduling

### Changed
- Improved performance with eager loading

## [1.1.0] - 2024-01-05

### Added
- Added tag support
- Added comment moderation

## [1.0.0] - 2024-01-01

### Added
- Initial release
- Basic blog functionality (posts, comments)
```

### CI/CD สำหรับ Gem

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        ruby-version: ['3.3.6']
        rails-version: ['8.1.4']
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: ${{ matrix.ruby-version }}
        bundler-cache: true
    
    - name: Run tests
      env:
        RAILS_VERSION: ${{ matrix.rails-version }}
      run: bundle exec rspec
    
    - name: Run RuboCop
      run: bundle exec rubocop
  
  publish:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 3.3.6
        bundler-cache: true
    
    - name: Build gem
      run: gem build blog_engine.gemspec
    
    - name: Push to RubyGems
      run: gem push *.gem
      env:
        GEM_HOST_API_KEY: ${{ secrets.RUBYGEMS_API_KEY }}
```

---

## Step 880: Full Engine vs Mountable Engine vs Isolated Namespace

### ความแตกต่างระหว่าง 3 ประเภท

**1. Full Engine (ไม่ใช้ `isolate_namespace`)**

```bash
rails plugin new blog_engine --full
```

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    # ไม่มี isolate_namespace
  end
end
```

คุณสมบัติ:
- Routes **ไม่** isolated — routes ของ engine รวมกับ host app
- Namespace **ไม่** isolated — models, controllers อยู่ใน root namespace
- Models ชื่อ `Post`, `Comment` (ไม่มี `BlogEngine::` prefix)
- Table names: `posts`, `comments` (ไม่มี prefix)
- **ระวัง:** อาจ conflict กับ host app!

**2. Mountable Engine (ใช้ `isolate_namespace`)**

```bash
rails plugin new blog_engine --mountable
```

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
  end
end
```

คุณสมบัติ:
- Routes **isolated** — ต้อง mount ที่ path ที่กำหนด
- Namespace **isolated** — models, controllers อยู่ใน `BlogEngine::` namespace
- Models ชื่อ `BlogEngine::Post`, `BlogEngine::Comment`
- Table names: `blog_engine_posts`, `blog_engine_comments` (มี prefix)
- **แนะนำ:** สำหรับ gems ที่จะ share กับผู้อื่น

**3. Plain Rails Plugin**

```bash
rails plugin new blog_plugin
```

ไม่ใช่ Engine เลย — แค่ Ruby code ที่ extend Rails

### เปรียบเทียบโดยละเอียด

```
Feature                  | Full Engine    | Mountable Engine | Plain Plugin
-------------------------|----------------|------------------|-------------
Routes isolated          | ไม่            | ใช่              | N/A
Namespace isolated       | ไม่            | ใช่              | ไม่
Table prefix             | ไม่มี         | มี               | N/A
Mount at path            | ต้อง           | ต้อง             | N/A
Risk of conflict         | สูง            | ต่ำ              | ต่ำ
Complexity               | ต่ำ            | กลาง             | ต่ำ
Best for                 | Internal apps  | Public gems      | Libraries
```

### ตัวอย่างการเลือกใช้

**เลือก Full Engine เมื่อ:**
- Engine ใช้เฉพาะใน organization ของตัวเอง
- ต้องการ share models โดยตรงกับ host app
- ไม่กังวลเรื่อง namespace conflict

```ruby
# Full engine: routes ไม่ isolated
# config/routes.rb ของ host app
Rails.application.routes.draw do
  # Engine routes มารวมอัตโนมัติ ไม่ต้อง mount
  resources :posts   # อาจ conflict ถ้า host app มี posts ด้วย!
end
```

**เลือก Mountable Engine เมื่อ:**
- จะ publish เป็น public gem
- ต้องการ namespace isolation แน่ๆ
- Engine จะ work กับ host app หลายตัว

```ruby
# Mountable engine: routes isolated
# config/routes.rb ของ host app
Rails.application.routes.draw do
  mount BlogEngine::Engine, at: '/blog'    # ชัดเจน ไม่ conflict
  resources :posts                          # host app posts ของตัวเอง
end
```

**เลือก Plain Plugin เมื่อ:**
- แค่ต้องการ extend Rails functionality
- ไม่มี routes, controllers, views
- เช่น: custom validators, helpers, concerns

```ruby
# Plain plugin: แค่ extend Rails
module MyValidations
  extend ActiveSupport::Concern
  
  included do
    validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  end
end
```

### เปลี่ยนจาก Full เป็น Mountable

ถ้าเริ่มด้วย full engine แล้วอยากเปลี่ยนเป็น mountable:

```ruby
# ก่อน (Full Engine)
module BlogEngine
  class Engine < ::Rails::Engine
  end
end

# หลัง (Mountable Engine)
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine
  end
end
```

ต้อง update ทุกอย่างให้ใช้ namespace:

```ruby
# ก่อน
class Post < ApplicationRecord; end
class PostsController < ApplicationController; end

# หลัง
module BlogEngine
  class Post < ApplicationRecord; end
  class PostsController < ApplicationController; end
end
```

และ update database migrations:

```ruby
# ก่อน
create_table :posts do |t|
  ...
end

# หลัง
create_table :blog_engine_posts do |t|
  ...
end
```

### Summary: เลือกอะไรเมื่อไหร่

```
ต้องการสร้างอะไร?
│
├─ Reusable feature สำหรับแชร์กับคนอื่น
│  └─ → Mountable Engine (isolate_namespace)
│
├─ Feature สำหรับใช้ใน organization เดียวกัน
│  ├─ กังวลเรื่อง conflict
│  │  └─ → Mountable Engine
│  └─ ไม่กังวลเรื่อง conflict, ต้องการ simplicity
│     └─ → Full Engine
│
└─ แค่ extend Rails (helpers, validators, concerns)
   └─ → Plain Plugin หรือ Ruby module ธรรมดา
```

### ตัวอย่าง Gems ที่ใช้แต่ละแบบ

| Gem | ประเภท | เหตุผล |
|-----|--------|--------|
| Devise | Mountable Engine | ต้องการ namespace isolation, ใช้ใน production apps หลายตัว |
| ActiveAdmin | Mountable Engine | mount ที่ /admin, isolated namespace |
| Spree | Mountable Engine | E-commerce engine ที่ต้องแยก namespace ชัดเจน |
| Ransack | Plain Plugin | แค่ extend ActiveRecord query |
| Kaminari | Plain Plugin | แค่ add pagination methods |
| Pundit | Plain Plugin | แค่ add authorization methods |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง FAQ Engine

สร้าง mountable engine ชื่อ `faq_engine` ที่มี:

1. Model `Question` ที่มี fields: `title`, `body`, `category`, `published`
2. Controller สำหรับ CRUD operations
3. Mount ที่ `/faq` ใน host app
4. Scope `published` และ `by_category`
5. Migration ที่สร้าง table `faq_engine_questions`

```bash
# สร้าง engine
rails plugin new faq_engine --mountable

# สร้าง migration
rails generate migration CreateFaqEngineQuestions \
  title:string \
  body:text \
  category:string \
  published:boolean \
  position:integer
```

### แบบฝึกหัดที่ 2: Engine Configuration

เพิ่ม configuration options ใน FAQ engine:

```ruby
# ต้องการให้ host app configure ได้แบบนี้:
FaqEngine::Engine.tap do |engine|
  engine.questions_per_page = 20
  engine.enable_search = true
  engine.categories = ['General', 'Technical', 'Billing']
end
```

เขียน code ใน `engine.rb` เพื่อรองรับ configuration นี้

### แบบฝึกหัดที่ 3: Engine Tests

เขียน RSpec tests สำหรับ FAQ engine:

1. Model spec สำหรับ `Question`:
   - ทดสอบ validations
   - ทดสอบ scopes
   
2. Controller spec สำหรับ `QuestionsController`:
   - ทดสอบ `GET #index`
   - ทดสอบ `POST #create` กับ valid/invalid params

3. Feature spec:
   - ทดสอบ visitor สามารถดู FAQ ได้
   - ทดสอบ filter by category

### แบบฝึกหัดที่ 4: Publish Engine

เตรียม gemspec สำหรับ `faq_engine`:

1. เพิ่ม metadata ครบถ้วน (homepage, source_code_uri, etc.)
2. กำหนด required_ruby_version และ Rails version
3. แยก runtime dependencies และ development dependencies ให้ถูกต้อง
4. สร้าง VERSION constant ด้วย Semantic Versioning
5. Build gem และทดสอบ local installation

### แบบฝึกหัดที่ 5: เปรียบเทียบ Engine Types

สร้าง Rails app ใหม่และทดลอง mount engine ทั้ง 3 แบบ:

1. Full Engine — สังเกตความแตกต่างของ routes
2. Mountable Engine — สังเกต namespace isolation
3. Plain Plugin — สังเกตว่าไม่มี routes และ controllers

จด notes ความแตกต่างที่เห็นจริงๆ

---

## สรุปสิ่งที่ได้เรียนรู้

ใน Part นี้เราได้เรียนรู้เรื่อง Rails Engine อย่างครบถ้วน:

### Step 871 — Rails Engine คืออะไร
เข้าใจว่า Engine คือ mini-Rails application ที่ embed ได้ เห็นตัวอย่างจาก Devise, ActiveAdmin, Spree และเข้าใจข้อดีของ Engine เทียบกับ gem ธรรมดา

### Step 872 — สร้าง Engine ใหม่
เรียนรู้การใช้ `rails plugin new blog_engine --mountable` และเข้าใจโครงสร้างโฟลเดอร์ทั้งหมด รวมถึงไฟล์สำคัญอย่าง `engine.rb` และ `gemspec`

### Step 873 — Engine Routing
เข้าใจการ mount Engine ใน host app ด้วย `mount BlogEngine::Engine, at: '/blog'` และการใช้ route helpers ทั้งใน engine (`blog_engine.posts_path`) และการเข้าถึง host app routes (`main_app.root_path`)

### Step 874 — Models, Controllers, Views ใน Engine
เรียนรู้ namespace convention ที่ทุกอย่างอยู่ใน `BlogEngine::` module และ table prefix `blog_engine_` ที่เกิดขึ้นอัตโนมัติ รวมถึงโครงสร้าง views และ layout ของ engine

### Step 875 — แชร์ Resources กับ Host App
เข้าใจการ hook เข้า host app ผ่าน initializers และการใช้ `mattr_accessor` สำหรับ configuration เพื่อให้ engine flexible

### Step 876 — Engine Migrations
เรียนรู้การสร้าง migrations ใน engine และการ copy ไปยัง host app ด้วย `rake blog_engine:install:migrations` รวมถึงการสร้าง install generator

### Step 877 — Asset Pipeline ใน Engine
เข้าใจโครงสร้าง assets ของ engine ทั้ง manifest, CSS, JavaScript และการประกาศ precompile assets ใน initializer

### Step 878 — Test Engine อย่างเดียว
เรียนรู้การใช้ `test/dummy` app สำหรับ testing, การ setup RSpec ใน engine, การเขียน model/controller/feature specs และการใช้ factories

### Step 879 — Publish Engine เป็น Gem
เข้าใจการเตรียม gemspec ที่สมบูรณ์, Semantic Versioning, การ build และ push gem ไป RubyGems.org และ CI/CD workflow

### Step 880 — Full Engine vs Mountable Engine vs Isolated Namespace
เข้าใจความแตกต่างระหว่างทั้ง 3 ประเภทอย่างชัดเจน และรู้ว่าควรเลือกใช้แบบไหนในสถานการณ์ใด

### Key Takeaways

1. **Rails Engine = mini-Rails app** ที่ embed เข้า app หลักได้
2. **Mountable Engine + isolate_namespace** คือตัวเลือกที่แนะนำสำหรับ public gems
3. **Table prefix** เกิดขึ้นอัตโนมัติจาก isolated namespace
4. **Route helpers** ใน engine ใช้ prefix ของ engine, เข้า host app ผ่าน `main_app`
5. **Dummy app** ใน `test/` ช่วยให้ test engine ได้โดยอิสระ
6. **Migrations** ต้อง copy ไปยัง host app ก่อน run

### Rails Engine ในโลกจริง

```
Gem ที่ใช้ Engine ในโลกจริง:
├── Devise      — Authentication (Mountable)
├── ActiveAdmin — Admin panel (Mountable)
├── Spree       — E-commerce (Mountable)
├── Refinery    — CMS (Mountable)
├── Thredded    — Forum (Mountable)
└── Forem       — Community platform (Mountable)
```

---

## ตัวอย่างโปรเจกต์สมบูรณ์

สรุป Engine ที่เราสร้างใน Part นี้:

```
blog_engine/
├── app/
│   ├── controllers/blog_engine/
│   │   ├── application_controller.rb
│   │   ├── posts_controller.rb
│   │   └── comments_controller.rb
│   ├── models/blog_engine/
│   │   ├── application_record.rb
│   │   ├── post.rb
│   │   ├── comment.rb
│   │   └── tag.rb
│   ├── views/blog_engine/
│   │   ├── layouts/
│   │   ├── posts/
│   │   └── comments/
│   └── assets/
│       ├── config/blog_engine_manifest.js
│       ├── stylesheets/blog_engine/
│       └── javascripts/blog_engine/
├── config/routes.rb
├── db/migrate/
│   ├── 001_create_blog_engine_posts.rb
│   ├── 002_create_blog_engine_comments.rb
│   └── 003_create_blog_engine_tags.rb
├── lib/
│   ├── blog_engine/
│   │   ├── engine.rb
│   │   └── version.rb
│   └── blog_engine.rb
├── spec/
│   ├── models/blog_engine/
│   ├── controllers/blog_engine/
│   ├── features/blog_engine/
│   └── factories/
└── blog_engine.gemspec
```

---

## Preview: Part 089

ใน **Part 089** เราจะเรียนรู้เรื่อง **Active Support — Utilities และ Extensions** ซึ่งเป็น library ที่ Rails ใช้ภายในและ developer ทุกคนควรรู้จัก:

- `ActiveSupport::Concern` — การจัด module ให้ดีขึ้น
- `ActiveSupport::Callbacks` — callback system
- `ActiveSupport::Notifications` — instrumentation framework
- `ActiveSupport::Cache` — caching abstraction
- `ActiveSupport::MessageEncryptor` — encryption utilities
- Extensions บน String, Array, Hash, Integer ที่ Rails เพิ่มให้
- Lazy loading และ autoloading ด้วย `Zeitwerk`
- Time zones และ date/time utilities

Active Support เป็น "Swiss Army knife" ของ Rails — เข้าใจมันจะทำให้ code ของเราสะอาดและมีประสิทธิภาพขึ้นมาก!

---

*Part 088 จบแล้ว — ตอนนี้คุณรู้จัก Rails Engine อย่างครบถ้วนแล้ว ไม่ว่าจะเป็นการสร้าง, mount, test, หรือ publish เป็น gem สำหรับแชร์กับโลก!*
