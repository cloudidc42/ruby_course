# Outline เต็มรูปแบบ: หลักสูตร Ruby on Rails (Step 1–1000 / Part 1–100+)

เอกสารนี้คือแผนที่ทั้งหมดของหลักสูตร ใช้เป็นจุดอ้างอิงเดียวว่า Step ไหนอยู่ Part ไหน
และ Part ไหนเขียนเสร็จแล้ว

## สถานะความคืบหน้า

> อัปเดตล่าสุด: ดูวันที่ commit ล่าสุดในประวัติ git ของไฟล์นี้

- [x] เฟส 1 (Part 001–010, Step 1–100) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/01-fundamentals/`
- [x] เฟส 2 (Part 011–020, Step 101–200) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/02-ruby-deep-dive/`
- [x] เฟส 3 (Part 021–030, Step 201–300) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/03-rails-fundamentals/`
- [x] เฟส 4 (Part 031–040, Step 301–400) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/04-crud-forms/`
- [x] เฟส 5 (Part 041–045, Step 401–450) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/05-auth/`
- [x] เฟส 6 (Part 046–050, Step 451–500) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/06-testing/` (**ครึ่งทางของหลักสูตรแล้ว!**)
- [x] เฟส 7 (Part 051–055, Step 501–550) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/07-frontend/`
- [x] เฟส 8 (Part 056–060, Step 551–600) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/08-apis/`
- [x] เฟส 9 (Part 061–065, Step 601–650) — เสร็จสมบูรณ์ ดูไฟล์ใน `course/09-jobs-performance/`
- [ ] เฟส 10–17 (Part 066–100+) — กำลังเขียนต่อเนื่อง ดูสถานะจริงจากไฟล์ที่มีอยู่ในแต่ละโฟลเดอร์ `course/<phase>/`

กฎการตั้งชื่อไฟล์: `course/<เฟส>/part-<เลข 3 หลัก>-<slug ภาษาอังกฤษ>.md`

---

## เฟส 1: Ruby Fundamentals (Part 1–10 / Step 1–100)

เป้าหมาย: เขียนโปรแกรม Ruby พื้นฐานได้คล่อง เข้าใจ syntax, data types, control flow, methods, OOP เบื้องต้น

| Part | Step | หัวข้อ |
|------|------|--------|
| 001 | 1–10 | ติดตั้งสภาพแวดล้อม (rbenv/asdf, Ruby version), REPL (irb/pry), Hello World, การรันไฟล์ .rb, cli พื้นฐาน |
| 002 | 11–20 | ตัวแปร, ชนิดข้อมูลพื้นฐาน (Integer, Float, String, Symbol, nil, true/false), การแปลงชนิดข้อมูล |
| 003 | 21–30 | String methods, String interpolation, Heredoc, Regular Expression เบื้องต้น |
| 004 | 31–40 | Array: การสร้าง, iteration, method สำคัญ (map, select, reduce, each_with_index) |
| 005 | 41–50 | Hash: การสร้าง, iteration, nested hash, symbol vs string keys |
| 006 | 51–60 | Control flow: if/unless/case, ternary, loop, while, until, for, break/next/redo |
| 007 | 61–70 | Methods: การนิยาม method, arguments (positional, keyword, default, splat, double splat), return values |
| 008 | 71–80 | Blocks, yield, Proc, Lambda เบื้องต้น |
| 009 | 81–90 | OOP เบื้องต้น: Class, Object, attr_accessor, initialize, instance/class variable |
| 010 | 91–100 | Inheritance, Module, Mixin (include/extend), แบบฝึกหัดโปรเจกต์เล็ก: Library Management CLI |

## เฟส 2: Ruby Deep Dive (Part 11–20 / Step 101–200)

เป้าหมาย: เข้าใจ Ruby ระดับลึก พร้อมเขียนโค้ดแบบมืออาชีพ testable และ maintainable

| Part | Step | หัวข้อ |
|------|------|--------|
| 011 | 101–110 | Exception handling: begin/rescue/ensure, custom exception class, retry |
| 012 | 111–120 | File I/O, การอ่าน/เขียนไฟล์, CSV, JSON, YAML |
| 013 | 121–130 | Enumerable module ขั้นสูง: each_slice, group_by, partition, flat_map, lazy enumerator |
| 014 | 131–140 | Comparable module, Struct, OpenStruct, Data class (Ruby 3.2+) |
| 015 | 141–150 | Metaprogramming เบื้องต้น: method_missing, define_method, send, respond_to? |
| 016 | 151–160 | Duck typing, SOLID principles ใน Ruby, design pattern เบื้องต้น (Strategy, Observer) |
| 017 | 161–170 | Gem คืออะไร, Bundler, Gemfile, การสร้าง gem ของตัวเอง (เบื้องต้น) |
| 018 | 171–180 | Testing เบื้องต้นด้วย Minitest: unit test, assertion, test doubles เบื้องต้น |
| 019 | 181–190 | RSpec เบื้องต้น: describe/context/it, matcher, let, before/after |
| 020 | 191–200 | Rake, Rakefile, การเขียน CLI tool ด้วย Ruby ล้วน (โปรเจกต์: Todo CLI พร้อม test) |

## เฟส 3: Rails Fundamentals — MVC (Part 21–30 / Step 201–300)

เป้าหมาย: เข้าใจสถาปัตยกรรม MVC และสร้างเว็บแอปพื้นฐานด้วย Rails ได้

| Part | Step | หัวข้อ |
|------|------|--------|
| 021 | 201–210 | ติดตั้ง Rails, `rails new`, โครงสร้างโฟลเดอร์, การรัน server, config เบื้องต้น |
| 022 | 211–220 | Routing: `config/routes.rb`, resources, RESTful routes, route helpers |
| 023 | 221–230 | Controller: action, params, session, flash, before_action |
| 024 | 231–240 | View: ERB, layout, partial, helper, view ที่ reuse ได้ |
| 025 | 241–250 | Model & ActiveRecord เบื้องต้น: migration, schema, CRUD ผ่าน console |
| 026 | 251–260 | Validation เบื้องต้น, callback เบื้องต้น (before_save, after_create) |
| 027 | 261–270 | Association เบื้องต้น: belongs_to, has_many, has_one |
| 028 | 271–280 | Scaffold, generator ของ Rails, การอ่านโค้ดที่ generate มา |
| 029 | 281–290 | Asset pipeline / Propshaft, static asset, image_tag, link_to |
| 030 | 291–300 | โปรเจกต์รวบยอดเฟส 3: Blog แบบง่าย (Post + Comment) CRUD ครบ |

## เฟส 4: CRUD, Forms, ActiveRecord ขั้นสูง (Part 31–40 / Step 301–400)

| Part | Step | หัวข้อ |
|------|------|--------|
| 031 | 301–310 | form_with เชิงลึก, strong parameters, nested attributes |
| 032 | 311–320 | Validation ขั้นสูง: custom validator, conditional validation, uniqueness with scope |
| 033 | 321–330 | Association ขั้นสูง: has_many :through, has_and_belongs_to_many, polymorphic association |
| 034 | 331–340 | Query interface: where, order, joins, includes (N+1), scope |
| 035 | 341–350 | ActiveRecord callback ขั้นสูง, lifecycle ทั้งหมด, ข้อควรระวัง |
| 036 | 351–360 | Concern (ActiveSupport::Concern), การจัดโครงสร้างโมเดลขนาดใหญ่ |
| 037 | 361–370 | Multiple models form (nested_attributes, accepts_nested_attributes_for), fields_for |
| 038 | 371–380 | Pagination (Kaminari/Pagy), sorting, filtering แบบมืออาชีพ |
| 039 | 381–390 | Internationalization (I18n), locale, การจัดการ error message ภาษาไทย |
| 040 | 391–400 | โปรเจกต์รวบยอดเฟส 4: ระบบจัดการร้านค้าเล็ก (Product, Category, Order) |

## เฟส 5: Authentication & Authorization (Part 41–45 / Step 401–450)

| Part | Step | หัวข้อ |
|------|------|--------|
| 041 | 401–410 | Authentication เบื้องต้น: has_secure_password, session-based login/logout |
| 042 | 411–420 | Devise: ติดตั้ง, custom, confirmable, lockable |
| 043 | 421–430 | Authorization: Pundit เชิงลึก (policy, scope) |
| 044 | 431–440 | Authorization ทางเลือก: CanCanCan, role-based access control (RBAC) |
| 045 | 441–450 | Token-based auth: JWT, API authentication, OAuth (Omniauth) เบื้องต้น |

## เฟส 6: Testing (TDD/BDD) (Part 46–50 / Step 451–500)

| Part | Step | หัวข้อ |
|------|------|--------|
| 046 | 451–460 | RSpec สำหรับ Rails: model spec, request spec |
| 047 | 461–470 | FactoryBot, Faker, การจัดการ test data |
| 048 | 471–480 | Feature test / System test ด้วย Capybara |
| 049 | 481–490 | Mocking/stubbing, VCR สำหรับ external API, test coverage (SimpleCov) |
| 050 | 491–500 | TDD workflow เต็มรูปแบบ: เขียนฟีเจอร์ใหม่ด้วย Red-Green-Refactor |

## เฟส 7: Frontend / Hotwire / Stimulus (Part 51–55 / Step 501–550)

| Part | Step | หัวข้อ |
|------|------|--------|
| 051 | 501–510 | Turbo Drive, Turbo Frames |
| 052 | 511–520 | Turbo Streams (real-time update แบบไม่ใช้ JS เยอะ) |
| 053 | 521–530 | Stimulus.js: controller, action, target, value |
| 054 | 531–540 | Tailwind CSS / Bootstrap ใน Rails, responsive layout |
| 055 | 541–550 | ViewComponent, component-based UI ใน Rails |

## เฟส 8: APIs & GraphQL (Part 56–60 / Step 551–600)

| Part | Step | หัวข้อ |
|------|------|--------|
| 056 | 551–560 | Rails API-only mode, JSON response, serializer (ActiveModel::Serializer / Jbuilder) |
| 057 | 561–570 | API versioning, JSON:API spec, pagination สำหรับ API |
| 058 | 571–580 | GraphQL เบื้องต้นด้วย graphql-ruby: schema, type, query |
| 059 | 581–590 | GraphQL mutation, authorization ใน GraphQL |
| 060 | 591–600 | API documentation (rswag/OpenAPI), rate limiting (rack-attack) |

## เฟส 9: Background Jobs & Performance (Part 61–65 / Step 601–650)

| Part | Step | หัวข้อ |
|------|------|--------|
| 061 | 601–610 | ActiveJob เบื้องต้น, adapter ต่างๆ |
| 062 | 611–620 | Sidekiq เชิงลึก: queue, retry, scheduled job |
| 063 | 621–630 | Caching: fragment cache, Russian doll caching, low-level caching (Rails.cache) |
| 064 | 631–640 | Database performance: index, explain analyze, N+1 detection (Bullet) |
| 065 | 641–650 | Performance profiling: rack-mini-profiler, memory profiling |

## เฟส 10: File Upload / Search (Part 66–70 / Step 651–700)

| Part | Step | หัวข้อ |
|------|------|--------|
| 066 | 651–660 | Active Storage: upload, variant, direct upload |
| 067 | 661–670 | Active Storage กับ cloud storage (S3-compatible), image processing |
| 068 | 671–680 | Full-text search ด้วย pg_search |
| 069 | 681–690 | Elasticsearch/OpenSearch integration เบื้องต้น |
| 070 | 691–700 | โปรเจกต์: ระบบอัปโหลดรูปสินค้า + ค้นหาสินค้า |

## เฟส 11: Payment & Third-party Integration (Part 71–72 / Step 701–720)

| Part | Step | หัวข้อ |
|------|------|--------|
| 071 | 701–710 | Stripe integration: checkout, webhook, subscription |
| 072 | 711–720 | Omniauth (Google/Facebook login), ส่งอีเมลด้วย Action Mailer, SMS/notification integration |

## เฟส 12: DevOps & Deployment (Part 73–78 / Step 721–780)

| Part | Step | หัวข้อ |
|------|------|--------|
| 073 | 721–730 | Docker สำหรับ Rails: Dockerfile, docker-compose |
| 074 | 731–740 | Environment variable, credentials, multi-environment config |
| 075 | 741–750 | CI/CD: GitHub Actions สำหรับ Rails (test, lint, security scan) |
| 076 | 751–760 | Deployment ด้วย Kamal (Rails 8 style deploy) |
| 077 | 761–770 | Deployment ทางเลือก: Render/Fly.io/Heroku, database migration ใน production |
| 078 | 771–780 | Monitoring & logging: structured logging, error tracking (Sentry), uptime |

## เฟส 13: Security (Part 79–81 / Step 781–810)

| Part | Step | หัวข้อ |
|------|------|--------|
| 079 | 781–790 | OWASP Top 10 ใน context ของ Rails: SQLi, XSS, CSRF |
| 080 | 791–800 | Mass assignment, secure headers, brakeman scan |
| 081 | 801–810 | Secrets management, encryption (ActiveRecord Encryption), security checklist ก่อน production |

## เฟส 14: Architecture & Scaling (Part 82–86 / Step 811–860)

| Part | Step | หัวข้อ |
|------|------|--------|
| 082 | 811–820 | Service Object pattern, Form Object pattern |
| 083 | 821–830 | Query Object, Decorator pattern (Draper), Presenter |
| 084 | 831–840 | Clean Architecture / Hexagonal ใน Rails, dependency injection เบื้องต้น |
| 085 | 841–850 | Multi-tenancy: row-based vs schema-based, Apartment gem |
| 086 | 851–860 | Microservices กับ Rails: service communication, message queue (RabbitMQ/Kafka เบื้องต้น) |

## เฟส 15: Rails Internals & Gem Building (Part 87–90 / Step 861–900)

| Part | Step | หัวข้อ |
|------|------|--------|
| 087 | 861–870 | Rack เบื้องลึก, middleware ของ Rails |
| 088 | 871–880 | Rails Engine: การสร้าง mountable engine |
| 089 | 881–890 | การสร้างและ publish gem ของตัวเองไปยัง RubyGems |
| 090 | 891–900 | อ่าน source code ของ Rails framework, contribute open source เบื้องต้น |

## เฟส 16: Real-world Capstone Projects (Part 91–96 / Step 901–960)

| Part | Step | หัวข้อ |
|------|------|--------|
| 091 | 901–910 | Capstone 1: E-commerce platform — planning, ER diagram, setup |
| 092 | 911–920 | Capstone 1: E-commerce platform — cart, checkout, payment, order fulfillment |
| 093 | 921–930 | Capstone 2: SaaS multi-tenant app — planning, billing (subscription) |
| 094 | 931–940 | Capstone 2: SaaS multi-tenant app — team/organization, permission, invite flow |
| 095 | 941–950 | Capstone 3: Real-time chat/dashboard app ด้วย Hotwire + ActionCable |
| 096 | 951–960 | Capstone 3 ต่อ: deployment เต็มรูปแบบ + load testing เบื้องต้น |

## เฟส 17: Career & Staff Engineer Mastery (Part 97–100+ / Step 961–1000)

| Part | Step | หัวข้อ |
|------|------|--------|
| 097 | 961–970 | Code review ระดับมืออาชีพ, การเขียน RFC/design doc |
| 098 | 971–980 | System design สำหรับ Rails application ขนาดใหญ่ (scaling checklist) |
| 099 | 981–990 | การเตรียมสัมภาษณ์งาน Ruby on Rails ระดับ Senior/Staff, live coding practice |
| 100 | 991–1000 | สรุปหลักสูตร, roadmap การเรียนรู้ต่อ, การสร้าง portfolio และ contribute ให้ community |

---

## หมายเหตุการเขียนเนื้อหา

- แต่ละ Part ต้องมีตัวอย่างโค้ดที่รันได้จริง (ระบุเวอร์ชัน Ruby/Rails/gem ที่ใช้)
- แต่ละ Part ปิดท้ายด้วย "แบบฝึกหัด" และ "สรุปสิ่งที่ได้เรียนรู้"
- ไฟล์ Part ควรมีความยาว 500–3000+ บรรทัดตามที่ระบุในโจทย์ (เนื้อหาเชิงลึก ไม่ใช่การเติมคำฟุ่มเฟือย)
- Part ใหม่ที่เขียนเสร็จ ต้องอัปเดตเครื่องหมาย `[x]` ในหัวข้อ "สถานะความคืบหน้า" ด้านบนของไฟล์นี้
