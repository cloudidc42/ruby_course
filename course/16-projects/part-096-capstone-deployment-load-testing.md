# Part 096: Capstone 3 ต่อ — Deployment เต็มรูปแบบ + Load Testing เบื้องต้น

> **Step ครอบคลุมใน Part นี้:** Step 951–960

**ระดับ:** ขั้นสูง (Advanced) | **Rails Version:** 7.x / 8.x | **Ruby Version:** 3.2+

ใน Part นี้เราจะนำ Real-Time Chat Application ที่สร้างมาตั้งแต่ Part 095 ไปสู่ production จริง ซึ่งถือเป็นขั้นตอนที่สำคัญที่สุดในวงจรการพัฒนา เพราะแอปที่ดีต้องทำงานได้จริง ไม่ใช่แค่บน localhost เราจะเรียนรู้ถึง Production Checklist, การ deploy ด้วย Kamal สำหรับแอปที่ใช้ ActionCable, การตั้งค่า Redis สำหรับ production, การ setup SSL/TLS และที่สำคัญที่สุดคือการทดสอบว่าแอปของเราสามารถรองรับผู้ใช้งานจริงได้มากแค่ไหน ผ่านเครื่องมือ Load Testing เช่น k6 พร้อมเรียนรู้วิธีอ่านผลและหา bottleneck เพื่อ optimize แอปต่อไป

---

## สารบัญ

- [Step 951: Production Checklist ก่อน Deploy](#step-951)
- [Step 952: Kamal Deployment สำหรับ Real-time App](#step-952)
- [Step 953: Redis สำหรับ Production](#step-953)
- [Step 954: Database Backup และ Restore](#step-954)
- [Step 955: SSL/TLS ใน Production](#step-955)
- [Step 956: Monitoring Real-time App](#step-956)
- [Step 957: Load Testing เบื้องต้น](#step-957)
- [Step 958: อ่านผล Load Test](#step-958)
- [Step 959: Bottleneck ที่พบบ่อย](#step-959)
- [Step 960: สรุป Phase 16](#step-960)
- [แบบฝึกหัด](#แบบฝึกหัด)
- [สรุปสิ่งที่ได้เรียนรู้](#สรุปสิ่งที่ได้เรียนรู้)

---

## Step 951: Production Checklist ก่อน Deploy {#step-951}

ก่อนที่จะ deploy แอปขึ้น production ทุกครั้ง ควรมี checklist ที่ครอบคลุมเพื่อป้องกันปัญหาที่พบบ่อย โดยเฉพาะด้าน security ที่อาจเปิดช่องโหว่ให้ผู้ไม่ประสงค์ดีโจมตีได้

### Security Headers Checklist

```ruby
# config/initializers/security_headers.rb
Rails.application.config.action_dispatch.default_headers = {
  # ป้องกัน clickjacking
  "X-Frame-Options" => "SAMEORIGIN",
  
  # ป้องกัน MIME type sniffing
  "X-Content-Type-Options" => "nosniff",
  
  # ป้องกัน XSS (legacy browsers)
  "X-XSS-Protection" => "1; mode=block",
  
  # บังคับใช้ HTTPS
  "Strict-Transport-Security" => "max-age=31536000; includeSubDomains",
  
  # Referrer Policy
  "Referrer-Policy" => "strict-origin-when-cross-origin",
  
  # Permissions Policy
  "Permissions-Policy" => "camera=(), microphone=(), geolocation=()"
}
```

```ruby
# config/environments/production.rb
Rails.application.configure do
  # บังคับใช้ SSL
  config.force_ssl = true
  
  # ตั้งค่า HSTS
  config.ssl_options = {
    hsts: {
      subdomains: true,
      preload: true,
      expires: 1.year
    }
  }
  
  # CSP (Content Security Policy)
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.script_src  :self, :unsafe_inline  # ต้องการสำหรับ Turbo/Stimulus
    policy.style_src   :self, :unsafe_inline
    policy.img_src     :self, :data, "https://your-s3-bucket.s3.amazonaws.com"
    policy.connect_src :self, "wss://your-domain.com"  # สำหรับ WebSocket
    policy.font_src    :self
  end
end
```

### Secret Rotation Checklist

```bash
# ตรวจสอบว่าไม่มี secrets hardcode ในโค้ด
git log --all --full-history -- "*.yml" | head -20
git grep -n "password\|secret\|api_key" -- "*.rb" "*.yml"

# ตรวจสอบ credentials ที่ใช้งาน
rails credentials:show --environment production

# Rotate master key (ทำเมื่อมีการ leak)
# 1. สร้าง credentials ใหม่
rails credentials:edit --environment production

# 2. อัปเดต environment variable บน server
export RAILS_MASTER_KEY=new_key_here

# 3. ตรวจสอบว่า credentials เดิมยกเลิกใช้งานแล้ว
```

```ruby
# config/credentials.yml.enc (ตัวอย่าง structure)
# ห้าม hardcode ค่าเหล่านี้ในโค้ด ต้องใช้ credentials หรือ ENV เสมอ
secret_key_base: "... (auto-generated)"
database:
  password: "strong_password_here"
aws:
  access_key_id: "AKIA..."
  secret_access_key: "..."
  region: "ap-southeast-1"
  bucket: "rails-chat-prod"
redis:
  url: "redis://:password@your-redis-host:6379/0"
devise:
  secret: "..."
```

### Asset Precompile Verification

```bash
# ทดสอบ asset precompile ใน local environment ก่อน deploy
RAILS_ENV=production rails assets:precompile

# ตรวจสอบว่าไฟล์ที่ต้องการถูก compile แล้ว
ls public/assets/ | head -20

# ตรวจสอบ JavaScript bundle
ls public/assets/*.js

# ล้าง cache ก่อน rebuild
rails assets:clobber
RAILS_ENV=production rails assets:precompile
```

```ruby
# config/initializers/assets.rb
Rails.application.config.assets.precompile += %w[
  application.css
  application.js
  # เพิ่ม fonts, icons ที่ต้องการ
]
```

### Database Readiness Checklist

```ruby
# ตรวจสอบ indexes ที่จำเป็น
# db/migrate/xxx_add_performance_indexes.rb
class AddPerformanceIndexes < ActiveRecord::Migration[7.0]
  def change
    # Index สำหรับ message queries ที่พบบ่อย
    add_index :messages, [:chat_room_id, :created_at],
              name: "index_messages_on_room_and_time"
    
    # Index สำหรับ notification queries
    add_index :notifications, [:recipient_id, :read_at],
              name: "index_notifications_on_recipient_unread"
    
    # Index สำหรับ online presence queries  
    add_index :users, :last_seen_at,
              name: "index_users_on_last_seen_at"
    
    # Partial index สำหรับ unread notifications
    add_index :notifications, :recipient_id,
              where: "read_at IS NULL",
              name: "index_notifications_unread"
  end
end
```

### Environment Variables Checklist

```bash
# .env.production.example (commit ได้ แต่ .env ห้าม commit)
RAILS_ENV=production
RAILS_MASTER_KEY=           # ต้องตั้งค่าเอง
DATABASE_URL=               # postgresql://user:pass@host/db
REDIS_URL=                  # redis://:pass@host:6379/0
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-southeast-1
AWS_S3_BUCKET=
SENTRY_DSN=
APP_HOST=your-domain.com
WEB_CONCURRENCY=2
RAILS_MAX_THREADS=5
SIDEKIQ_CONCURRENCY=10
```

---

## Step 952: Kamal Deployment สำหรับ Real-time App {#step-952}

Kamal (เดิมคือ MRSK) เป็นเครื่องมือ deployment สำหรับ Rails ที่ใช้ Docker และสามารถ deploy ได้บน VPS หรือ cloud ใดก็ได้ การ deploy แอปที่ใช้ ActionCable มีข้อควรระวังพิเศษเรื่อง sticky sessions

### ติดตั้งและ Configure Kamal

```bash
# ติดตั้ง Kamal
gem install kamal

# Initialize Kamal configuration
kamal init

# จะสร้างไฟล์ config/deploy.yml
```

### config/deploy.yml สำหรับ Real-time App

```yaml
# config/deploy.yml
service: rails-chat
image: your-dockerhub-username/rails-chat

servers:
  web:
    hosts:
      - your-server-ip
    labels:
      traefik.http.routers.rails-chat.rule: Host(`your-domain.com`)
      traefik.http.services.rails-chat.loadbalancer.sticky.cookie: "true"
      traefik.http.services.rails-chat.loadbalancer.sticky.cookie.name: "lb_session"
    options:
      publish:
        - "3000:3000"
  
  # Worker สำหรับ Sidekiq
  worker:
    hosts:
      - your-server-ip
    cmd: bundle exec sidekiq -C config/sidekiq.yml

registry:
  username:
    - KAMAL_REGISTRY_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: "true"
    WEB_CONCURRENCY: 2
    RAILS_MAX_THREADS: 5
    REDIS_URL: redis://redis:6379/0
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY

# Accessories (services ที่ run พร้อมกัน)
accessories:
  redis:
    image: redis:7.2-alpine
    host: your-server-ip
    port: "127.0.0.1:6379:6379"
    cmd: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1
    volumes:
      - /var/lib/redis:/data
  
  db:
    image: postgres:16-alpine
    host: your-server-ip
    port: "127.0.0.1:5432:5432"
    env:
      clear:
        POSTGRES_DB: rails_chat_production
      secret:
        - POSTGRES_PASSWORD
    volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data

# Traefik (reverse proxy + SSL)
traefik:
  options:
    publish:
      - "443:443"
    volume:
      - "/letsencrypt/acme.json:/letsencrypt/acme.json"
  args:
    entrypoints.web.address: ":80"
    entrypoints.websecure.address: ":443"
    certificatesresolvers.letsencrypt.acme.tlschallenge: "true"
    certificatesresolvers.letsencrypt.acme.email: "your@email.com"
    certificatesresolvers.letsencrypt.acme.storage: "/letsencrypt/acme.json"

healthcheck:
  path: /up
  port: 3000
  interval: 10s
  timeout: 5s
  threshold: 3
```

### ทำไม Sticky Sessions จึงสำคัญสำหรับ ActionCable

```
โดยไม่มี Sticky Sessions:
  Request 1 → Server A (WebSocket connected)
  Request 2 → Server B (WebSocket ไม่พบ!)
  
โดยมี Sticky Sessions:
  Request 1 → Server A (WebSocket connected)
  Request 2 → Server A (WebSocket ยังคงอยู่!)
```

Traefik รองรับ sticky sessions ผ่าน cookie ซึ่งทำให้ browser ถูก route ไปยัง server เดิมเสมอ

### Thruster สำหรับ HTTP/2 + WebSocket

```ruby
# Gemfile
gem "thruster"  # HTTP/2 proxy สำหรับ Puma

# config/puma.rb
# Thruster จัดการ HTTP/2 termination ให้
# Rails app ยังทำงานเป็น HTTP/1.1 ปกติ
```

```bash
# Dockerfile (ที่ Kamal ใช้)
FROM ruby:3.2-slim

# ติดตั้ง dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    nodejs \
    yarn

WORKDIR /rails

COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

COPY . .

# Precompile assets
RUN RAILS_ENV=production bundle exec rails assets:precompile

# Entrypoint
COPY bin/docker-entrypoint /usr/bin/
RUN chmod +x /usr/bin/docker-entrypoint
ENTRYPOINT ["docker-entrypoint"]

EXPOSE 3000
CMD ["./bin/thrust", "./bin/rails", "server"]
```

### Kamal Deployment Commands

```bash
# Deploy ครั้งแรก
kamal setup

# Deploy updates
kamal deploy

# ตรวจสอบ logs
kamal app logs
kamal app logs --follow

# Rollback ถ้ามีปัญหา
kamal app rollback

# SSH เข้า server
kamal app exec --interactive --reuse "bash"

# ตรวจสอบ app status
kamal app details

# Run rake tasks บน production
kamal app exec "bundle exec rails db:migrate"
```

---

## Step 953: Redis สำหรับ Production {#step-953}

Redis มีบทบาทสำคัญมากในแอปนี้ ใช้สำหรับทั้ง ActionCable pub/sub และ Sidekiq job queue การตั้งค่า Redis ให้ถูกต้องส่งผลโดยตรงต่อความเสถียรของแอป

### ทำไม ActionCable และ Sidekiq ควรใช้ Redis instance เดียวกัน (หรือต่างกัน?)

**Option 1: Redis instance เดียว (พร้อมแยก database)**
```
Redis Instance
  ├── Database 0: ActionCable
  ├── Database 1: Sidekiq  
  └── Database 2: Rails Cache
```

**Option 2: Redis instances แยกกัน (สำหรับ high traffic)**
```
Redis Instance 1: ActionCable (real-time pub/sub)
Redis Instance 2: Sidekiq (job queue)
Redis Instance 3: Rails Cache
```

สำหรับแอปขนาดกลาง Option 1 เพียงพอ แต่ถ้ามี traffic สูงมาก ควรแยกเพื่อป้องกัน Redis เดียวกัน saturate

### Configure Redis ด้วย Database Separation

```yaml
# config/cable.yml
development:
  adapter: redis
  url: redis://localhost:6379/1
  channel_prefix: rails_chat_dev

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") %>
  channel_prefix: rails_chat_prod
```

```ruby
# config/initializers/sidekiq.rb
redis_config = {
  url: ENV.fetch("REDIS_SIDEKIQ_URL", ENV.fetch("REDIS_URL")),
  db: 2,  # database 2 สำหรับ Sidekiq
  password: ENV["REDIS_PASSWORD"]
}

Sidekiq.configure_server do |config|
  config.redis = redis_config
end

Sidekiq.configure_client do |config|
  config.redis = redis_config
end
```

```ruby
# config/initializers/redis.rb
# Global Redis client สำหรับ custom usage (เช่น presence tracking)
$redis = Redis.new(
  url: ENV.fetch("REDIS_URL"),
  db: 3,  # database 3 สำหรับ application cache
  timeout: 5,
  reconnect_attempts: 3
)

# หรือใช้ Redis::Namespace เพื่อ prefix keys
require "redis-namespace"
$redis = Redis::Namespace.new("rails_chat", redis: Redis.new(url: ENV.fetch("REDIS_URL")))
```

### Redis Cloud Setup

```bash
# Option 1: Redis Cloud (managed service)
# 1. สมัครที่ redis.io/try-free
# 2. สร้าง database ใหม่
# 3. Copy connection string

# Connection string format:
# redis://default:password@host:port

# Option 2: Self-hosted Redis บน VPS
# ใน config/deploy.yml (Kamal accessories):
accessories:
  redis:
    image: redis:7.2-alpine
    cmd: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --save 60 10000
    volumes:
      - /var/data/redis:/data
```

### Redis Persistence Configuration

```bash
# สำหรับ ActionCable — ไม่จำเป็นต้อง persist (real-time data)
# สำหรับ Sidekiq — ต้อง persist เพื่อไม่ให้ job หาย

# redis.conf สำหรับ Sidekiq
save 900 1      # save ถ้ามี 1 key เปลี่ยนใน 900 วินาที
save 300 10     # save ถ้ามี 10 keys เปลี่ยนใน 300 วินาที
save 60 10000   # save ถ้ามี 10000 keys เปลี่ยนใน 60 วินาที

appendonly yes  # AOF (Append Only File) สำหรับ durability สูง
appendfsync everysec
```

### ตรวจสอบ Redis Health

```ruby
# lib/tasks/redis.rake
namespace :redis do
  desc "Check Redis connection and stats"
  task health: :environment do
    begin
      info = Redis.current.info
      puts "Redis Version: #{info['redis_version']}"
      puts "Connected Clients: #{info['connected_clients']}"
      puts "Used Memory: #{info['used_memory_human']}"
      puts "Total Commands Processed: #{info['total_commands_processed']}"
      puts "Uptime: #{info['uptime_in_days']} days"
      puts "Status: OK ✓"
    rescue Redis::CannotConnectError => e
      puts "Status: FAILED ✗"
      puts "Error: #{e.message}"
      exit 1
    end
  end
  
  desc "Show ActionCable channel subscriptions"
  task cable_stats: :environment do
    pubsub = Redis.current.pubsub("channels", "action_cable:*")
    puts "Active ActionCable channels: #{pubsub.count}"
    pubsub.first(10).each do |channel|
      puts "  #{channel}"
    end
  end
end
```

---

## Step 954: Database Backup และ Restore {#step-954}

ข้อมูลในฐานข้อมูลคือทรัพย์สินที่มีค่าที่สุดของแอป การมีกลยุทธ์ backup ที่ดีช่วยป้องกันความเสียหายจากเหตุการณ์ไม่คาดคิด

### pg_dump สำหรับ PostgreSQL Backup

```bash
# Backup database ด้วย pg_dump
pg_dump \
  --host=$DB_HOST \
  --port=$DB_PORT \
  --username=$DB_USER \
  --dbname=$DB_NAME \
  --format=custom \         # compressed binary format
  --file=backup_$(date +%Y%m%d_%H%M%S).dump \
  --verbose

# หรือ SQL format (อ่านได้ง่ายกว่า)
pg_dump \
  --host=$DB_HOST \
  --username=$DB_USER \
  $DB_NAME > backup_$(date +%Y%m%d).sql

# Backup เฉพาะ schema (ไม่มีข้อมูล)
pg_dump --schema-only $DATABASE_URL > schema_backup.sql

# Backup เฉพาะข้อมูล (ไม่มี schema)
pg_dump --data-only $DATABASE_URL > data_backup.sql
```

### Automated Backup ด้วย Cron

```bash
# Script สำหรับ backup อัตโนมัติ
# bin/backup_database.sh
#!/bin/bash

set -e

BACKUP_DIR="/var/backups/rails_chat"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.dump"
DAYS_TO_KEEP=30

mkdir -p $BACKUP_DIR

echo "Starting backup at $(date)"

# Backup
pg_dump \
  --format=custom \
  --file=$BACKUP_FILE \
  $DATABASE_URL

echo "Backup completed: $BACKUP_FILE"
echo "Size: $(du -h $BACKUP_FILE | cut -f1)"

# Upload ไปยัง S3 (optional)
if [ ! -z "$S3_BACKUP_BUCKET" ]; then
  aws s3 cp $BACKUP_FILE s3://$S3_BACKUP_BUCKET/db-backups/
  echo "Uploaded to S3"
fi

# ลบ backup เก่าที่เกิน 30 วัน
find $BACKUP_DIR -name "*.dump" -mtime +$DAYS_TO_KEEP -delete
echo "Cleaned up old backups"

echo "Backup process completed successfully"
```

```bash
# crontab -e
# Backup ทุกวันเวลา 02:00
0 2 * * * /app/bin/backup_database.sh >> /var/log/db_backup.log 2>&1

# Backup ทุก 6 ชั่วโมงสำหรับแอปที่มีข้อมูลเปลี่ยนบ่อย
0 */6 * * * /app/bin/backup_database.sh >> /var/log/db_backup.log 2>&1
```

### Restore Drill — ทดสอบการ Restore จริง

การทดสอบ restore เป็นสิ่งสำคัญมาก เพราะ backup ที่ restore ไม่ได้คือ backup ที่ไม่มีค่า ควรทำ restore drill เป็นประจำ (อย่างน้อยเดือนละครั้ง)

```bash
# ขั้นตอน Restore Drill
# 1. สร้าง database ทดสอบ
createdb rails_chat_restore_test

# 2. Restore จาก backup file
pg_restore \
  --host=$DB_HOST \
  --username=$DB_USER \
  --dbname=rails_chat_restore_test \
  --verbose \
  --clean \              # drop objects ก่อน restore
  backup_20241201.dump

# 3. ตรวจสอบข้อมูล
psql rails_chat_restore_test -c "SELECT COUNT(*) FROM users;"
psql rails_chat_restore_test -c "SELECT COUNT(*) FROM messages;"
psql rails_chat_restore_test -c "SELECT MAX(created_at) FROM messages;"

# 4. รัน migration (ถ้า backup เก่ากว่า schema)
DATABASE_URL=postgresql://localhost/rails_chat_restore_test \
  rails db:migrate

# 5. ลบ database ทดสอบ
dropdb rails_chat_restore_test

echo "Restore drill completed successfully!"
```

### Rails Task สำหรับ Backup/Restore

```ruby
# lib/tasks/db_backup.rake
namespace :db do
  desc "Create a database backup"
  task backup: :environment do
    timestamp = Time.current.strftime("%Y%m%d_%H%M%S")
    filename = "backup_#{Rails.env}_#{timestamp}.dump"
    path = Rails.root.join("tmp", "backups", filename)
    
    FileUtils.mkdir_p(File.dirname(path))
    
    system(
      "pg_dump",
      "--format=custom",
      "--file=#{path}",
      ENV["DATABASE_URL"] || ActiveRecord::Base.connection_db_config.url,
      exception: true
    )
    
    puts "Backup created: #{path}"
    puts "Size: #{File.size(path) / 1024 / 1024}MB"
  end
  
  desc "Verify latest backup"
  task verify_backup: :environment do
    backup_dir = Rails.root.join("tmp", "backups")
    latest = Dir.glob("#{backup_dir}/*.dump").max_by { |f| File.mtime(f) }
    
    if latest.nil?
      puts "ERROR: No backup files found!"
      exit 1
    end
    
    age_hours = (Time.current - File.mtime(latest)) / 3600
    puts "Latest backup: #{File.basename(latest)}"
    puts "Age: #{age_hours.round(1)} hours"
    puts "Size: #{File.size(latest) / 1024 / 1024}MB"
    
    if age_hours > 25
      puts "WARNING: Backup is more than 25 hours old!"
    else
      puts "Status: OK ✓"
    end
  end
end
```

---

## Step 955: SSL/TLS ใน Production {#step-955}

SSL/TLS เป็นสิ่งจำเป็นสำหรับ production ไม่ใช่แค่เพื่อ security แต่ยังต้องการสำหรับ WebSocket (ActionCable ใช้ wss:// แทน ws://) และ HTTP/2

### Let's Encrypt ด้วย Kamal / Traefik

Kamal ใช้ Traefik เป็น reverse proxy ซึ่งรองรับ Let's Encrypt อัตโนมัติ

```yaml
# config/deploy.yml (Traefik configuration)
traefik:
  options:
    publish:
      - "80:80"
      - "443:443"
    volume:
      - "/letsencrypt:/letsencrypt"
  
  args:
    # HTTP entrypoint
    entrypoints.web.address: ":80"
    # Redirect HTTP → HTTPS
    entrypoints.web.http.redirections.entrypoint.to: "websecure"
    entrypoints.web.http.redirections.entrypoint.scheme: "https"
    
    # HTTPS entrypoint
    entrypoints.websecure.address: ":443"
    
    # Let's Encrypt
    certificatesresolvers.letsencrypt.acme.tlschallenge: "true"
    certificatesresolvers.letsencrypt.acme.email: "admin@your-domain.com"
    certificatesresolvers.letsencrypt.acme.storage: "/letsencrypt/acme.json"
    
    # Logging
    log.level: "INFO"
    accesslog: "true"
```

```yaml
# server labels ใน deploy.yml
servers:
  web:
    hosts:
      - your-server-ip
    labels:
      # Router สำหรับ HTTPS
      traefik.http.routers.rails-chat-secure.rule: "Host(`your-domain.com`)"
      traefik.http.routers.rails-chat-secure.entrypoints: "websecure"
      traefik.http.routers.rails-chat-secure.tls: "true"
      traefik.http.routers.rails-chat-secure.tls.certresolver: "letsencrypt"
      
      # WebSocket support
      traefik.http.middlewares.websocket.headers.customrequestheaders.Connection: "Upgrade"
      traefik.http.middlewares.websocket.headers.customrequestheaders.Upgrade: "websocket"
      
      # Sticky sessions สำหรับ ActionCable
      traefik.http.services.rails-chat.loadbalancer.sticky.cookie: "true"
      traefik.http.services.rails-chat.loadbalancer.sticky.cookie.secure: "true"
      traefik.http.services.rails-chat.loadbalancer.sticky.cookie.httpOnly: "true"
```

### HSTS Preload

HSTS (HTTP Strict Transport Security) บอก browser ให้ใช้ HTTPS เสมอ HSTS Preload ก้าวไปอีกขั้นโดยบรรจุ domain เข้าใน list ที่ browser เชื่อถือ

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = {
  hsts: {
    subdomains: true,
    preload: true,      # สำคัญ! ต้องมีเพื่อ preload
    expires: 1.year     # ต้องอย่างน้อย 1 ปี สำหรับ preload
  }
}
```

```bash
# ขั้นตอนเพิ่ม domain ใน HSTS Preload List
# 1. ตรวจสอบว่า HSTS header ถูกต้อง
curl -I https://your-domain.com | grep "Strict-Transport"
# Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# 2. ทดสอบที่ hstspreload.org
# เข้า https://hstspreload.org และกรอก domain

# 3. Submit domain (ถ้าผ่านการตรวจสอบทั้งหมด)
# หมายเหตุ: การ remove ออกจาก list ทำได้ยาก ให้แน่ใจก่อน submit
```

### SSL Certificate Monitoring

```ruby
# lib/tasks/ssl_check.rake
namespace :ssl do
  desc "Check SSL certificate expiry"
  task check: :environment do
    require "net/http"
    require "openssl"
    
    host = ENV.fetch("APP_HOST", "your-domain.com")
    
    begin
      tcp_client = TCPSocket.new(host, 443)
      ssl_client = OpenSSL::SSL::SSLSocket.new(tcp_client)
      ssl_client.connect
      
      cert = ssl_client.peer_cert
      expiry = cert.not_after
      days_left = (expiry - Time.current) / 86400
      
      puts "Certificate for: #{host}"
      puts "Expires: #{expiry}"
      puts "Days remaining: #{days_left.floor}"
      
      if days_left < 30
        puts "WARNING: Certificate expires in less than 30 days!"
        # ส่ง alert (email, Slack, etc.)
      else
        puts "Status: OK ✓"
      end
    rescue => e
      puts "ERROR: #{e.message}"
    ensure
      ssl_client&.close
      tcp_client&.close
    end
  end
end
```

---

## Step 956: Monitoring Real-time App {#step-956}

การ monitor แอป production ช่วยให้เราตรวจพบและแก้ไขปัญหาก่อนที่ผู้ใช้จะได้รับผลกระทบ

### Sentry สำหรับ Error Tracking

```ruby
# Gemfile
gem "sentry-ruby"
gem "sentry-rails"
gem "sentry-sidekiq"

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV.fetch("SENTRY_DSN")
  config.environment = Rails.env
  
  # ส่ง performance data
  config.traces_sample_rate = Rails.env.production? ? 0.1 : 1.0
  
  # ส่ง user context
  config.set_user do
    request.env["warden"]&.user&.then do |user|
      {
        id: user.id,
        email: user.email,
        username: user.display_name
      }
    end
  end
  
  # กรอง sensitive data
  config.before_send = lambda do |event, hint|
    # ลบ password จาก params
    if event.request&.data
      event.request.data.delete("password")
      event.request.data.delete("password_confirmation")
    end
    event
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_sentry_user
  
  private
  
  def set_sentry_user
    return unless current_user
    
    Sentry.set_user(
      id: current_user.id,
      email: current_user.email,
      username: current_user.display_name
    )
  end
end
```

### Skylight / Scout APM สำหรับ Performance

```ruby
# Gemfile
gem "skylight"
# หรือ
gem "scout_apm"

# config/initializers/skylight.rb
# (Skylight ใช้ config/skylight.yml แทน)

# config/skylight.yml
authentication: <%= ENV["SKYLIGHT_AUTHENTICATION"] %>
# Skylight จะ track ทุก request โดยอัตโนมัติ
# รวมถึง N+1 queries และ slow actions
```

```ruby
# Custom instrumentation สำหรับ ActionCable
class ChatRoomChannel < ApplicationCable::Channel
  def subscribed
    # Instrument ActionCable subscriptions
    ActiveSupport::Notifications.instrument(
      "subscribed.action_cable",
      channel: self.class.name,
      user_id: current_user.id
    ) do
      stream_for @chat_room
    end
  end
end
```

### Health Check Endpoint

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  skip_before_action :authenticate_user!
  
  def show
    checks = {
      database: check_database,
      redis: check_redis,
      sidekiq: check_sidekiq,
      storage: check_storage
    }
    
    status = checks.values.all? { |c| c[:status] == "ok" } ? :ok : :service_unavailable
    
    render json: {
      status: status == :ok ? "ok" : "degraded",
      checks: checks,
      version: ENV["APP_VERSION"] || "unknown",
      timestamp: Time.current.iso8601
    }, status: status
  end
  
  private
  
  def check_database
    ActiveRecord::Base.connection.execute("SELECT 1")
    { status: "ok", response_time_ms: measure { ActiveRecord::Base.connection.execute("SELECT 1") } }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_redis
    start = Time.current
    Redis.current.ping
    { status: "ok", response_time_ms: ((Time.current - start) * 1000).round }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_sidekiq
    stats = Sidekiq::Stats.new
    { 
      status: "ok", 
      processed: stats.processed,
      failed: stats.failed,
      queues: stats.queues
    }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def check_storage
    ActiveStorage::Blob.service.exist?("health_check_file") rescue nil
    { status: "ok" }
  rescue => e
    { status: "error", message: e.message }
  end
  
  def measure
    start = Time.current
    yield
    ((Time.current - start) * 1000).round
  end
end
```

---

## Step 957: Load Testing เบื้องต้นด้วย k6 {#step-957}

Load testing คือการจำลองผู้ใช้จำนวนมากใช้งานแอปพร้อมกัน เพื่อดูว่าแอปสามารถรับมือได้แค่ไหนก่อนที่ performance จะลดลง

### ติดตั้ง k6

```bash
# macOS
brew install k6

# Ubuntu/Debian
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
  --keyserver hkp://keyserver.ubuntu.com:80 \
  --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69

echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/k6.list

sudo apt-get update
sudo apt-get install k6
```

### สร้าง k6 Test Script

```javascript
// load_tests/chat_load_test.js
import http from "k6/http"
import ws from "k6/ws"
import { check, sleep } from "k6"
import { Counter, Trend, Rate } from "k6/metrics"

// Custom metrics
const messagesSent = new Counter("messages_sent")
const messageSendTime = new Trend("message_send_time")
const wsConnectionTime = new Trend("ws_connection_time")
const wsErrors = new Rate("ws_errors")

// Test configuration
export const options = {
  stages: [
    { duration: "1m", target: 10 },   // ramp-up: เพิ่ม 10 users ใน 1 นาที
    { duration: "3m", target: 50 },   // ramp-up: เพิ่มเป็น 50 users
    { duration: "5m", target: 50 },   // steady state: คงที่ที่ 50 users
    { duration: "2m", target: 100 },  // peak load: เพิ่มเป็น 100 users
    { duration: "1m", target: 0 },    // ramp-down: ลดลงเหลือ 0
  ],
  
  thresholds: {
    // 95% ของ requests ต้องเสร็จใน 500ms
    http_req_duration: ["p(95)<500"],
    // Error rate ต้องต่ำกว่า 1%
    http_req_failed: ["rate<0.01"],
    // 90% ของ WebSocket connections ต้องเชื่อมต่อได้ใน 1 วินาที
    ws_connection_time: ["p(90)<1000"],
    // WebSocket error rate ต้องต่ำกว่า 2%
    ws_errors: ["rate<0.02"],
  },
}

// Test data
const BASE_URL = __ENV.BASE_URL || "https://your-domain.com"
const TEST_ROOM_ID = __ENV.ROOM_ID || "1"

export default function () {
  // 1. Login
  const loginRes = http.post(`${BASE_URL}/users/sign_in`, {
    "user[email]": `test_user_${__VU}@example.com`,
    "user[password]": "password123",
  })
  
  check(loginRes, {
    "login successful": (r) => r.status === 302 || r.status === 200,
  })
  
  const cookies = loginRes.cookies
  
  // 2. เปิดหน้า chat room
  const roomRes = http.get(`${BASE_URL}/chat_rooms/${TEST_ROOM_ID}`, {
    cookies: cookies,
  })
  
  check(roomRes, {
    "room page loaded": (r) => r.status === 200,
    "turbo stream present": (r) => r.body.includes("turbo-cable-stream-source"),
  })
  
  // 3. เชื่อมต่อ WebSocket
  const wsStart = Date.now()
  
  const wsRes = ws.connect(
    `wss://${BASE_URL.replace("https://", "")}/cable`,
    { headers: { Cookie: formatCookies(cookies) } },
    function (socket) {
      wsConnectionTime.add(Date.now() - wsStart)
      
      socket.on("open", function () {
        // Subscribe to chat room
        socket.send(JSON.stringify({
          command: "subscribe",
          identifier: JSON.stringify({
            channel: "ChatRoomChannel",
            chat_room_id: TEST_ROOM_ID,
          }),
        }))
      })
      
      socket.on("message", function (data) {
        const msg = JSON.parse(data)
        if (msg.type === "confirm_subscription") {
          // ส่งข้อความทดสอบ
          const msgStart = Date.now()
          
          const sendRes = http.post(
            `${BASE_URL}/chat_rooms/${TEST_ROOM_ID}/messages`,
            { "message[content]": `Load test message from VU ${__VU}` },
            { cookies: cookies }
          )
          
          messageSendTime.add(Date.now() - msgStart)
          messagesSent.add(1)
          
          check(sendRes, {
            "message sent successfully": (r) => r.status === 200 || r.status === 201,
          })
        }
        
        if (msg.type === "disconnect") {
          wsErrors.add(1)
        }
      })
      
      socket.on("error", function (e) {
        wsErrors.add(1)
        console.error("WebSocket error:", e)
      })
      
      // คง connection ไว้ 30 วินาที
      socket.setTimeout(function () {
        socket.close()
      }, 30000)
    }
  )
  
  check(wsRes, { "ws connected": (r) => r && r.status === 101 })
  
  sleep(Math.random() * 5 + 1)  // random pause 1-6 วินาที
}

function formatCookies(cookies) {
  return Object.entries(cookies)
    .map(([name, values]) => `${name}=${values[0].value}`)
    .join("; ")
}
```

### Apache Bench (ab) สำหรับ Simple HTTP Load Test

```bash
# ติดตั้ง Apache Bench
sudo apt-get install apache2-utils   # Ubuntu
brew install ab                       # macOS (through httpd)

# Test แบบง่าย: 1000 requests, 100 concurrent
ab -n 1000 -c 100 \
   -H "Cookie: _session_id=your_session_cookie" \
   https://your-domain.com/chat_rooms/1

# Test POST request (ส่งข้อความ)
ab -n 500 -c 50 \
   -p /tmp/post_data.txt \
   -T "application/x-www-form-urlencoded" \
   -H "Cookie: _session_id=your_session_cookie" \
   -H "X-CSRF-Token: your_csrf_token" \
   https://your-domain.com/chat_rooms/1/messages
```

### รัน k6 Test

```bash
# Test พื้นฐาน (local)
k6 run load_tests/chat_load_test.js

# Test กับ target URL จริง
k6 run --env BASE_URL=https://your-domain.com \
       --env ROOM_ID=1 \
       load_tests/chat_load_test.js

# Test พร้อม output ไปยัง InfluxDB (สำหรับ visualization)
k6 run --out influxdb=http://localhost:8086/k6 \
       load_tests/chat_load_test.js

# Test พร้อม Cloud dashboard
k6 cloud load_tests/chat_load_test.js
```

---

## Step 958: อ่านผล Load Test {#step-958}

การ interpret ผล load test ให้ถูกต้องเป็นทักษะสำคัญ ตัวเลขสถิติต่างๆ บอกเราถึงสุขภาพของระบบในแง่มุมที่ต่างกัน

### ทำความเข้าใจ Percentiles

```
Percentile (Pxx) คืออะไร?
  P50 (Median): 50% ของ requests เสร็จเร็วกว่าหรือเท่ากับค่านี้
  P90: 90% ของ requests เสร็จเร็วกว่าหรือเท่ากับค่านี้  
  P95: 95% ของ requests เสร็จเร็วกว่าหรือเท่ากับค่านี้
  P99: 99% ของ requests เสร็จเร็วกว่าหรือเท่ากับค่านี้
  P99.9: 99.9% ของ requests เสร็จเร็วกว่าหรือเท่ากับค่านี้

ตัวอย่าง:
  P50 = 100ms หมายความว่า user ครึ่งหนึ่งรอไม่เกิน 100ms
  P95 = 500ms หมายความว่า 95% ของ users รอไม่เกิน 500ms
  P99 = 2000ms หมายความว่ามีบาง users รอนานถึง 2 วินาที

ทำไม Average ไม่เพียงพอ?
  Average = 150ms อาจหมายความว่า:
    - ส่วนใหญ่ได้ 100ms แต่บางคนได้ 5000ms
    - Average ถูก skew โดยค่า outliers

  ให้ focus ที่ P95 หรือ P99 สำหรับ SLO (Service Level Objective)
```

### ตัวอย่างผลลัพธ์จาก k6 และวิธีอ่าน

```
          /\      |‾‾| /‾‾/   /‾‾/   
     /\  /  \     |  |/  /   /  /    
    /  \/    \    |     (   /   ‾‾\  
   /          \   |  |\  \ |  (‾)  | 
  / __________ \  |__| \__\ \_____/ 

  execution: local
     script: load_tests/chat_load_test.js
     output: -

  scenarios: (100.00%) 1 scenario, 100 max VUs
           * default: Up to 100 looping VUs for 12m0s over 5 stages

✓ login successful
✓ room page loaded  
✓ turbo stream present
✓ message sent successfully
✗ ws connected
  ↳ 3% — ✓ 97 / ✗ 3    ← 3% ของ WebSocket connections ล้มเหลว

checks.........................: 98.25% ✓ 15720 ✗ 280

data_received..................: 45 MB  62 kB/s
data_sent......................: 8.2 MB 11 kB/s

# HTTP Metrics
http_req_blocked...............: avg=1.2ms    min=1µs    med=4µs    max=312ms  p(90)=8µs    p(95)=12µs
http_req_connecting............: avg=0.5ms    min=0µs    med=0µs    max=205ms  p(90)=0µs    p(95)=0µs  
http_req_duration..............: avg=187ms    min=52ms   med=145ms  max=3420ms p(90)=312ms  p(95)=485ms 
    ↳ expected_response: avg=183ms    min=52ms   med=143ms  max=3420ms p(90)=308ms  p(95)=471ms
http_req_failed................: 0.18%  ✓ 0 ✗ 14
http_req_receiving.............: avg=1.4ms    min=12µs   med=0.4ms  max=312ms  p(90)=2.9ms  p(95)=5.2ms
http_req_sending...............: avg=0.1ms    min=6µs    med=87µs   max=14ms   p(90)=203µs  p(95)=310µs
http_req_tls_handshaking.......: avg=0.8ms    min=0µs    med=0µs    max=185ms  p(90)=0µs    p(95)=0µs  
http_req_waiting...............: avg=186ms    min=52ms   med=144ms  max=3370ms p(90)=310ms  p(95)=479ms
http_reqs......................: 7840   10.9/s     ← Throughput = 10.9 requests/second

# Custom Metrics
message_send_time...............: avg=210ms    min=80ms   med=185ms  max=1850ms p(90)=380ms  p(95)=520ms
messages_sent...................: 3920   5.4/s
ws_connection_time..............: avg=245ms    min=85ms   med=210ms  max=2100ms p(90)=420ms  p(95)=580ms
ws_errors......................: 2.50%  ✓ 97 ✗ 3

# Iteration/VU metrics
iteration_duration.............: avg=31.5s    min=10.2s  med=28.4s  max=95.3s  p(90)=52.1s  p(95)=63.8s
iterations.....................: 160    0.22/s
vus............................: 5      min=1      max=100
vus_max........................: 100    min=100    max=100
```

### การวิเคราะห์ผล

```
สิ่งที่ต้องสังเกตจากผลข้างต้น:

1. HTTP Response Time (http_req_duration)
   - P50 = 145ms ✓ (ดี)
   - P95 = 485ms ✓ (ผ่าน threshold 500ms พอดี)
   - P99 ≈ 1000ms ⚠️ (อาจต้องปรับปรุง)
   - Max = 3420ms ⚠️ (มี outliers ที่ช้ามาก)

2. Error Rate
   - HTTP: 0.18% ✓ (ต่ำกว่า threshold 1%)
   - WebSocket: 2.5% ✗ (เกิน threshold 2%)
   → ต้องตรวจสอบ WebSocket connection issues

3. Throughput
   - 10.9 requests/second ที่ 100 concurrent users
   - ถ้าต้องการ scale เป็น 1000 concurrent users ต้องเตรียม infrastructure
   
4. WebSocket Connection Time
   - P95 = 580ms ⚠️ (สูงกว่าที่ควร)
   → อาจมีปัญหาที่ Redis pub/sub หรือ connection handshake

การตัดสิน: แอปยังต้องปรับปรุงด้าน WebSocket reliability
ก่อนที่จะรองรับ concurrent users มากกว่านี้
```

### Throughput และ Concurrent Users คำนวณอย่างไร

```
Throughput = จำนวน requests ที่สำเร็จ ต่อ หน่วยเวลา

ถ้าได้ 10.9 requests/second ที่ 100 users:
- คนละ ~0.11 requests/second
- หรือ ~9 วินาทีต่อ request

สำหรับ chat app ที่ users ส่งข้อความทุก ~10 วินาที:
- ที่ 100 concurrent users ต้องการ 10 req/s → ผ่าน
- ที่ 1000 concurrent users ต้องการ 100 req/s → อาจต้องการ scaling
- ที่ 10000 concurrent users ต้องการ 1000 req/s → ต้องการ horizontal scaling
```

---

## Step 959: Bottleneck ที่พบบ่อย {#step-959}

หลังจาก load test เราจะพบ bottleneck ต่างๆ ต่อไปนี้คือสาเหตุที่พบบ่อยและวิธีแก้ไข

### N+1 Query Problem

```ruby
# ปัญหา: N+1 query ทำให้ database รับภาระมากเกินไป
class ChatRoomsController < ApplicationController
  def index
    @chat_rooms = current_user.chat_rooms
    # สำหรับแต่ละ room จะมี query เพิ่มอีก 2 queries:
    # 1. room.members (สำหรับแสดง online status)
    # 2. room.messages.last (สำหรับแสดง preview)
  end
end

# แก้ไข: ใช้ includes/eager_load
def index
  @chat_rooms = current_user.chat_rooms
                             .includes(:members, :messages)
                             .order(updated_at: :desc)
end

# ตรวจหา N+1 ด้วย Bullet gem
# Gemfile
gem "bullet", group: :development

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
  Bullet.add_footer = true
end
```

### Missing Database Indexes

```ruby
# ตรวจสอบ slow queries ด้วย pg_stat_statements
# ใน psql:
SELECT 
  calls,
  total_exec_time / calls as avg_time_ms,
  query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY avg_time_ms DESC
LIMIT 20;

# ตรวจสอบว่า query ใช้ index หรือไม่
EXPLAIN ANALYZE 
  SELECT * FROM messages 
  WHERE chat_room_id = 1 
  ORDER BY created_at DESC 
  LIMIT 50;

# ถ้าผลลัพธ์แสดง "Seq Scan" แทน "Index Scan" แสดงว่าขาด index
# แก้ไข:
add_index :messages, [:chat_room_id, :created_at]
```

### ActionCable Connection Pool Issues

```ruby
# ปัญหา: Thread exhaustion ใน Puma
# แต่ละ ActionCable connection ใช้ thread จาก pool

# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# ด้วย 2 workers และ 5 threads = 10 threads ทั้งหมด
# ถ้ามี 100 WebSocket connections เกิน capacity!

# แก้ไข option 1: เพิ่ม threads (ระวัง memory)
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 20 }

# แก้ไข option 2: ใช้ async adapter สำหรับ development
# config/cable.yml
development:
  adapter: async  # ใช้ async I/O แทน threads

# แก้ไข option 3: แยก ActionCable server
# รัน ActionCable บน process แยก (advanced)
```

### Redis Connection Pool Exhaustion

```ruby
# ปัญหา: too many Redis connections
# Sidekiq และ ActionCable ต้องการ Redis connections จำนวนมาก

# ตรวจสอบจำนวน connections
Redis.current.info("clients")["connected_clients"]

# แก้ไข: ใช้ connection pool
# Gemfile
gem "connection_pool"

# config/initializers/redis.rb
REDIS_POOL = ConnectionPool.new(size: 10, timeout: 5) do
  Redis.new(url: ENV.fetch("REDIS_URL"))
end

# ใช้งาน:
REDIS_POOL.with do |redis|
  redis.set("key", "value")
end

# Sidekiq มี connection pool built-in
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch("REDIS_URL"), pool_size: 10 }
end
```

### Memory Leaks

```ruby
# ตรวจสอบ memory usage ด้วย derailed_benchmarks
# Gemfile (development)
gem "derailed_benchmarks"

# รัน memory benchmark
bundle exec derailed exec perf:mem

# ปัญหาที่พบบ่อย: Long-lived objects ที่ไม่ถูก GC
# เช่น: Global cache ที่โตเรื่อยๆ

# ตรวจสอบ memory growth
bundle exec derailed exec perf:mem_over_time

# ใน production ตรวจสอบ memory usage ของ Puma workers
# หน่วยความจำที่โตเรื่อยๆ = memory leak
```

### ภาพรวมของ Performance Optimization Workflow

```
1. วัด (Measure):
   Load test → เก็บ baseline metrics
   
2. หา Bottleneck (Profile):
   - Skylight/Scout APM → หา slow actions
   - Bullet gem → หา N+1 queries
   - Redis monitor → หา slow Redis commands
   - pg_stat_statements → หา slow queries
   
3. แก้ไข (Fix):
   - เพิ่ม index
   - แก้ N+1 ด้วย eager loading
   - เพิ่ม caching
   - ปรับ connection pool size
   
4. วัดใหม่ (Re-measure):
   Load test อีกครั้ง → เปรียบเทียบกับ baseline
   
5. ทำซ้ำจนถึง performance target
```

---

## Step 960: สรุป Phase 16 — 3 Capstone Projects เสร็จสมบูรณ์ {#step-960}

ยินดีด้วย! คุณได้ผ่าน Phase 16 แล้วซึ่งถือว่าเป็นเฟสที่ท้าทายที่สุดในหลักสูตร Phase นี้ประกอบด้วย 3 Capstone Projects ที่สร้างขึ้นจากความรู้ทั้งหมดที่สะสมมา

### ทบทวน 3 Capstone Projects

**Capstone 1 (Part 081-085): E-Commerce Platform**
- สร้างระบบ e-commerce เต็มรูปแบบ
- Product catalog, Shopping cart, Checkout flow
- Order management, Stripe payment integration
- Admin dashboard, Inventory tracking
- ได้เรียนรู้: Complex business logic, Payment processing, File uploads

**Capstone 2 (Part 086-094): Multi-tenant SaaS App**
- สร้างระบบ SaaS ที่รองรับหลาย tenants
- Tenant isolation ด้วย row-level security
- Subscription management ด้วย Stripe Billing
- Role-based access control (RBAC)
- ได้เรียนรู้: Multi-tenancy architecture, Subscription billing, Authorization

**Capstone 3 (Part 095-096): Real-Time Chat/Dashboard**
- สร้างระบบ chat แบบ real-time ด้วย ActionCable + Hotwire
- Online presence, Typing indicators, Notifications
- Live dashboard สำหรับ admin
- Full deployment ด้วย Kamal
- Load testing และ performance optimization
- ได้เรียนรู้: WebSockets, Real-time architecture, Production deployment, Performance

### สิ่งที่ทำให้เป็น Rails Developer ระดับ Professional

หลังจากจบ Phase 16 คุณมีทักษะที่ครบครันสำหรับงาน Rails developer ระดับ professional:

```
Technical Skills:
✓ Rails MVC architecture อย่างลึกซึ้ง
✓ Database design + query optimization
✓ Authentication + Authorization (Devise, Pundit)
✓ Real-time features (ActionCable + Hotwire)
✓ Background jobs (Sidekiq)
✓ API design (REST + optional GraphQL)
✓ Testing (RSpec, Capybara, FactoryBot)
✓ File storage (Active Storage + S3)
✓ Payment processing (Stripe)
✓ Production deployment (Kamal + Docker)
✓ Performance monitoring + optimization
✓ Load testing

Project Experience:
✓ E-Commerce Platform
✓ Multi-tenant SaaS Application
✓ Real-Time Messaging Application
```

### สิ่งที่ควรทำต่อไปหลังจาก Phase 16

```ruby
# Path ที่แนะนำสำหรับการพัฒนาต่อ:

# 1. ทำโปรเจกต์ส่วนตัวที่แก้ปัญหาจริง
#    - Deploy จริง ดูแลแบบ production
#    - รับ feedback จากผู้ใช้จริง

# 2. Open Source Contribution
#    - Fork Rails หรือ gem ที่ใช้บ่อย
#    - แก้ bug เล็กๆ หรือ improve documentation
#    - เรียนรู้จากการอ่านโค้ดคนอื่น

# 3. ศึกษา Advanced Topics
#    - GraphQL API (gem: graphql-ruby)
#    - Event Sourcing / CQRS
#    - Microservices architecture
#    - gRPC สำหรับ service communication

# 4. DevOps/Infrastructure
#    - Kubernetes (k8s) สำหรับ container orchestration
#    - Terraform สำหรับ infrastructure as code
#    - CI/CD pipelines ที่ซับซ้อนมากขึ้น

# 5. สอบ certifications หรือ portfolios
#    - สร้าง portfolio ที่แสดง 3 Capstone projects
#    - เขียน blog posts อธิบายสิ่งที่เรียนรู้
```

### Phase 17 Preview

Phase 17 จะนำความรู้ด้าน Rails ไปสู่ขั้นต่อไปด้วยการศึกษา **Advanced Rails Patterns และ API Development**:
- GraphQL API ด้วย graphql-ruby
- REST API best practices (versioning, rate limiting, documentation)
- Rails Engine สำหรับ modular architecture
- Service Objects, Query Objects, Form Objects
- Event-driven architecture ด้วย Active Support Notifications
- Testing strategies สำหรับ complex applications (Contract testing, Integration testing)
- Rails upgrade strategies

---

## แบบฝึกหัด {#แบบฝึกหัด}

### ระดับพื้นฐาน

1. **Deployment Checklist**: สร้าง checklist ของตัวเองที่ครอบคลุมทุกขั้นตอนที่ต้องทำก่อน deploy production โดยใช้ความรู้จาก Part นี้และประสบการณ์ของตัวเอง

2. **Redis Monitoring Script**: เขียน Ruby script (หรือ Rake task) ที่ monitor Redis health และส่ง alert ผ่าน email หรือ Slack webhook เมื่อ memory usage เกิน 80%

3. **Simple Load Test**: เขียน k6 test สำหรับ endpoint `/chat_rooms` ที่ทดสอบว่าหน้า list ของห้องแชทรองรับ 50 concurrent users ได้หรือไม่ โดยกำหนด threshold: P95 < 300ms

### ระดับกลาง

4. **Database Backup Automation**: ตั้งค่า automated backup ที่:
   - รัน backup ทุกวันตอนตี 2
   - Upload ไปยัง S3 bucket
   - ส่ง notification ผ่าน email เมื่อ backup สำเร็จหรือล้มเหลว
   - ทดสอบ restore drill และ document ขั้นตอน

5. **Performance Profiling**: ติดตั้ง rack-mini-profiler และ bullet gem แล้วหา N+1 queries ทั้งหมดในแอป จากนั้นแก้ไขด้วย eager loading และวัดผลว่า query count ลดลงเท่าไหร่

6. **SSL Setup**: Setup staging environment ด้วย Kamal พร้อม Let's Encrypt certificate และทดสอบว่า WebSocket connection ทำงานได้ถูกต้องผ่าน wss://

### ระดับสูง

7. **Full Load Test Suite**: สร้าง k6 test suite ที่ครอบคลุม user journeys หลัก (login, chat, send file, view notifications) และ integrate เข้ากับ CI pipeline ให้รัน load test อัตโนมัติก่อน deploy ทุกครั้ง

8. **Monitoring Dashboard**: สร้าง admin dashboard ที่แสดง real-time metrics จาก Sentry errors, application performance (response times), Redis stats, Sidekiq queue sizes และ ActionCable connection count โดยอัปเดตทุก 30 วินาที

9. **Chaos Engineering เบื้องต้น**: ทดสอบความทนทานของแอปโดย simulate scenarios ต่างๆ เช่น Redis ไม่ตอบสนอง, Database connection timeout, Sidekiq worker crash แล้ว document ผลลัพธ์และวิธีที่แอปรับมือ

---

## สรุปสิ่งที่ได้เรียนรู้ {#สรุปสิ่งที่ได้เรียนรู้}

ใน Part 096 นี้ เราได้ครอบคลุมการนำแอป real-time ออก production อย่างสมบูรณ์ โดยสิ่งสำคัญที่ได้เรียนรู้มีดังนี้:

**Production Readiness**
- Security headers ที่จำเป็น (HSTS, CSP, X-Frame-Options)
- Secret rotation workflow และ credentials management
- Asset precompilation และ database indexes ที่จำเป็น

**Kamal Deployment**
- การ configure Kamal สำหรับแอปที่ใช้ ActionCable
- ทำไม sticky sessions จึงสำคัญสำหรับ WebSocket
- การใช้ Traefik เป็น reverse proxy พร้อม automatic SSL

**Redis Production Configuration**
- การแยก Redis databases สำหรับ ActionCable, Sidekiq และ Cache
- การตั้งค่า persistence สำหรับ Sidekiq jobs
- Connection pooling เพื่อป้องกัน exhaustion

**Database Operations**
- pg_dump สำหรับ backup และ restore
- Automated backup ด้วย cron
- Restore drill เพื่อตรวจสอบความถูกต้องของ backup

**SSL/TLS**
- Let's Encrypt ผ่าน Traefik ใน Kamal
- HSTS preload configuration
- WebSocket ต้องใช้ wss:// ใน production

**Load Testing**
- การใช้ k6 สร้าง realistic load test scenarios
- การอ่านผล: percentiles (P50/P95/P99), throughput, error rate
- การวิเคราะห์หา bottleneck จากผล load test

**Performance Optimization**
- N+1 queries: ตรวจหาด้วย Bullet, แก้ด้วย eager loading
- Missing indexes: ตรวจหาด้วย EXPLAIN ANALYZE
- ActionCable connection pool management
- Redis connection pooling ด้วย ConnectionPool gem

---

## Preview ของ Part ถัดไป

ใน **Part 097: GraphQL API ด้วย graphql-ruby** ใน Phase 17 เราจะเริ่มต้นการสร้าง API ที่มีพลังมากขึ้นด้วย GraphQL ซึ่งครอบคลุม:
- ทำไม GraphQL ถึงดีกว่า REST ในบางกรณี
- การ setup graphql-ruby gem
- Schema definition, Types, Queries, Mutations
- Authentication ใน GraphQL API
- N+1 สำหรับ GraphQL ด้วย graphql-batch
- Testing GraphQL endpoints
- GraphQL subscriptions สำหรับ real-time data
