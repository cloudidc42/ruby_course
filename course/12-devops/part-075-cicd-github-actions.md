# Part 075: CI/CD — GitHub Actions สำหรับ Rails (Test, Lint, Security Scan)

> **Step ครอบคลุมใน Part นี้:** Step 741–750
> **ระดับ:** ปานกลาง (ควรผ่าน Part 046–050 เรื่อง RSpec/FactoryBot/Capybara/SimpleCov, Part 073
> เรื่อง Docker, และ Part 074 เรื่อง environment variables/credentials มาก่อน — Part นี้จะอ้างอิง
> ทักษะเหล่านั้นแต่ไม่สอนซ้ำตั้งแต่ต้น)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ทดสอบจริงบน Ruby 3.3.6, Rails 8.1.4, RuboCop 1.91.0
> พร้อม `rubocop-rails-omakase`, Brakeman 8.0.6, gem `bundler-audit`, gem `rspec-rails`,
> `factory_bot_rails`, `simplecov`, GitHub Actions `actions/checkout@v6`, `ruby/setup-ruby@v1`,
> `actions/cache@v4`, `actions/upload-artifact@v4`)

> **หมายเหตุเรื่องการทดสอบ — อ่านก่อนเริ่ม (สำคัญมากสำหรับ Part นี้):**
>
> GitHub Actions รันจริงบนเซิร์ฟเวอร์ของ GitHub เท่านั้น แซนด์บ็อกซ์ที่ใช้เขียน Part นี้ไม่สามารถ
> push โค้ดไปสั่งรัน workflow บนโครงสร้างพื้นฐานจริงของ GitHub ได้ ทีมผู้เขียนหลักสูตรจึงยึดหลัก
> เดียวกับ Part ก่อนหน้า ๆ คือ **ทดสอบทุกสิ่งที่ทดสอบได้จริงในเครื่อง แล้วแยกให้ชัดเจนว่าส่วนไหน
> มาจากเอกสารทางการ**:
>
> - **สร้างแอป Rails 8.1.4 จริงด้วย `rails new`** (แยกจาก repository ของหลักสูตรโดยสิ้นเชิง ลบทิ้ง
>   หลังทดสอบเสร็จ ไม่มีไฟล์หลงเหลือ) แล้ว **อ่านไฟล์ `.github/workflows/ci.yml` และ
>   `.github/dependabot.yml` ที่ Rails generate ให้จริง** — เนื้อหาที่แสดงใน Step 742 คือเนื้อหา
>   จริงตัวต่อตัวที่ generator สร้างให้ ไม่ใช่ตัวอย่างที่เขียนขึ้นมาประกอบบทเรียน
> - **รันทุกคำสั่งที่อยู่ในแต่ละ step ของ workflow จริงในเครื่อง** ด้วยเวอร์ชัน gem เดียวกับที่
>   `bundle install` ติดตั้งให้: `bin/rubocop`, `bin/brakeman`, `bin/bundler-audit`,
>   `bin/rails db:test:prepare test` — ทุกคำสั่งรันผ่านจริงกับ PostgreSQL จริงที่รันอยู่ในแซนด์บ็อกซ์
>   (ตั้ง `DATABASE_URL=postgres://postgres:postgres@localhost:5432` ให้ตรงกับรูปแบบเดียวกับที่
>   `services:` block ของ workflow จริงกำหนดไว้ทุกตัวอักษร)
> - **จงใจแทรกช่องโหว่ SQL Injection เข้าไปในโค้ดจริง** เพื่อพิสูจน์ว่า Brakeman จับได้จริง (พบ
>   คำเตือนพร้อมเลขบรรทัดถูกต้อง, exit code เปลี่ยนจาก `3` เป็น `0` หลังแก้เป็น parameterized
>   query ด้วย `--exit-on-warn`) และจงใจเขียนโค้ดผิด style เพื่อพิสูจน์ว่า RuboCop คืน exit code
>   ไม่เท่ากับ 0 จริงเมื่อมี offense (พร้อม annotation format `::error file=...,line=...::...`
>   ของฟอร์แมต `-f github` ที่ได้จริงจากการรัน)
> - **ติดตั้ง `rspec-rails`/`factory_bot_rails` ลงแอปทดสอบจริง** เขียน model spec + factory จริง
>   รันผ่าน `bundle exec rspec` กับฐานข้อมูล PostgreSQL จริง ยืนยันว่าการสลับจาก Minitest (ค่า
>   default ของ Rails) มาเป็น RSpec (ที่หลักสูตรนี้สอนใน Phase 6) ทำได้จริงโดยแก้แค่คำสั่งใน
>   workflow บรรทัดเดียว
> - **ติดตั้ง SimpleCov จริง วัด coverage จริง** ได้ผลลัพธ์ `Line coverage: 82.22%` และไฟล์
>   `coverage/.last_run.json` ที่ใช้เขียนสรุปผลจริง สคริปต์ที่อ่านค่า JSON แล้วเขียนลง
>   `$GITHUB_STEP_SUMMARY` ในแบบฝึกหัดท้าย Part ก็รันผ่านจริงในเครื่องก่อนนำไปใส่ใน YAML
> - **วัดเวลาจริง**: `bundle install` แบบไม่มี cache ใช้เวลา **42.5 วินาที** บนแอปตั้งต้นเปล่า ๆ
>   ส่วนแบบมี cache (จำลองสถานการณ์ที่ `actions/cache` restore สำเร็จ) ใช้เวลาเพียง **0.58 วินาที**
>   ตัวเลขนี้เป็นของจริงจากเครื่องที่ใช้เขียนบทเรียน ไม่ใช่ตัวเลขสมมติ
> - **สาธิต matrix build จริงข้ามเวอร์ชัน Ruby**: รัน test เดียวกันบน Ruby 3.1.6, 3.2.6, 3.3.6
>   จริง (ทั้งสามเวอร์ชันติดตั้งอยู่ในแซนด์บ็อกซ์ผ่าน `rbenv`) ผ่านทั้งหมด
> - **ยืนยัน YAML syntax ของทุก workflow file ที่กำหนดขึ้นเอง** ด้วยการโหลดผ่าน Ruby's
>   `YAML.safe_load` จริง (รวมถึงพบข้อเท็จจริงที่น่าสนใจว่าคีย์ `on:` ที่ไม่ใส่เครื่องหมายคำพูด
>   จะถูกตีความเป็น boolean `true` โดย YAML parser ทั่วไปตามสเปก YAML 1.1 — GitHub Actions
>   จัดการเรื่องนี้ให้เองโดยไม่กระทบผู้ใช้งาน แต่เป็นความรู้ที่มีประโยชน์เมื่อเขียนเครื่องมือ
>   ประมวลผลไฟล์ YAML ของ Actions เอง)
> - **สิ่งที่มาจากเอกสารทางการเท่านั้น ไม่ได้รันจริง** (ระบุไว้ชัดเจนตรงจุด): หน้าตาจริงของ GitHub
>   Actions UI ขณะรัน workflow, หน้า Settings ของ GitHub สำหรับตั้งค่า Branch Protection Rules,
>   และพฤติกรรมของ GitHub-hosted runner จริง (เวลาบูตเครื่อง, Chrome ที่ติดตั้งมาให้บน
>   `ubuntu-latest`) — ส่วนเหล่านี้อธิบายตามเอกสารทางการของ GitHub แต่ไม่สามารถรันจริงจาก
>   แซนด์บ็อกซ์นี้ได้

จบจาก Part 073 ที่ทำให้แอป Rails รันในรูปแบบ container ที่คาดเดาได้ (Docker) และ Part 074 ที่ทำให้
การตั้งค่าต่างสภาพแวดล้อม (development/test/staging/production) ปลอดภัยด้วย environment variables
และ Rails encrypted credentials มาถึงจุดที่สำคัญที่สุดจุดหนึ่งของการทำงานเป็นทีมแบบมืออาชีพ นั่นคือ
**การทำให้ "การทดสอบ" ไม่ใช่สิ่งที่ต้องพึ่งความจำหรือวินัยของแต่ละคนอีกต่อไป** ไม่ว่าใครในทีมจะลืมรัน
test ก่อน push หรือลืมรัน linter ก่อนหรือไม่ **ระบบจะรันให้อัตโนมัติทุกครั้ง** และไม่ยอมให้โค้ดที่ทำ
test พัง, สไตล์ไม่ผ่าน, หรือมีช่องโหว่ความปลอดภัยที่ตรวจจับได้ถูก merge เข้า main branch โดยไม่มีใคร
รู้ตัว — นี่คือหัวใจของ **Continuous Integration (CI)** และ **Continuous Deployment (CD)**

ข่าวดีสำหรับคนที่ใช้ Rails 8: คุณมี CI pipeline ที่ใช้งานได้จริงอยู่แล้ว **ตั้งแต่วินาทีที่รัน
`rails new`** โดยไม่ต้องตั้งค่าอะไรเพิ่มเลยสักบรรทัด Part นี้จะพาไปทำความเข้าใจว่า workflow ที่ Rails
สร้างให้ทำงานอย่างไรทุกบรรทัด ก่อนจะขยายความรู้ไปสู่แนวคิดที่ลึกขึ้นของ GitHub Actions เอง

## สารบัญของ Part นี้

- Step 741: CI/CD คืออะไร และทำไมสำคัญ — แนวคิด "ทำให้คำถามว่าพังไหมตอบได้อัตโนมัติ"
- Step 742: `.github/workflows/ci.yml` ที่ Rails 8 สร้างให้อัตโนมัติ — เดินอ่านทุก job จริง
- Step 743: กายวิภาคของ GitHub Actions: workflow, job, step, runner, trigger
- Step 744: Service Containers — ตั้งค่า PostgreSQL/Redis ให้ job ทดสอบมีฐานข้อมูลจริงใช้งาน
- Step 745: Caching bundler/gem dependencies เพื่อให้ CI เร็วขึ้นอย่างมีนัยสำคัญ
- Step 746: รันชุดทดสอบจริงด้วย RSpec ใน CI (เชื่อมกับ Phase 6)
- Step 747: รัน RuboCop ใน CI และทำให้ lint failure = CI failure
- Step 748: รัน Brakeman security scan อัตโนมัติ — Brakeman ตรวจอะไรบ้างกันแน่
- Step 749: Branch Protection Rules — บังคับให้ CI ผ่านก่อน merge PR (ตั้งค่าที่ GitHub ไม่ใช่ในไฟล์)
- Step 750: CD เบื้องต้น (deploy อัตโนมัติเมื่อ merge เข้า main) และ Matrix Build (ทดสอบหลาย
  เวอร์ชัน Ruby)

---

## Step 741: CI/CD คืออะไร และทำไมสำคัญ

### ปัญหาที่ CI/CD แก้

ลองนึกภาพทีมพัฒนาที่ไม่มี CI: นักพัฒนา A แก้โค้ดเสร็จ รัน test บนเครื่องตัวเองผ่านหมด (หรือบางที
ก็ลืมรันเพราะรีบ) แล้ว push ขึ้น GitHub เปิด Pull Request นักพัฒนา B มา review โค้ดโดยอ่านผ่านตา
เท่านั้น ไม่ได้ pull ลงมารันเองเพราะเสียเวลา แล้ว approve merge เข้า main จากนั้นอีกสองวันถัดมา
ทีมถึงรู้ว่า production พังเพราะโค้ดที่ merge ไปทำให้ test เดิมที่เคยผ่านกลับพังในสภาพแวดล้อมอื่น
(เช่น เครื่อง B ใช้ gem เวอร์ชันแตกต่างจาก `Gemfile.lock` ที่ล็อกไว้, หรือไม่มีการรัน migration
ที่จำเป็นในเครื่อง B เลยไม่เจอ error)

นี่คือปัญหาคลาสสิกที่เรียกว่า **"works on my machine"** — โค้ดทำงานได้ในเครื่องคนเขียน แต่ไม่มีใคร
ยืนยันว่ามันทำงานได้ใน**สภาพแวดล้อมที่สะอาดและสม่ำเสมอ** (clean, consistent environment) ก่อนที่
จะไปถึงมือคนอื่นหรือ production

**Continuous Integration (CI)** คือแนวปฏิบัติที่ทำให้ทุกครั้งที่มีการ push โค้ดหรือเปิด Pull Request
ระบบจะ**อัตโนมัติ**:

1. ดึงโค้ดเวอร์ชันล่าสุดมาไว้ในเครื่องเสมือนที่สะอาด (ไม่มีร่องรอยการตั้งค่าที่ค้างจากรันก่อนหน้า)
2. ติดตั้ง dependencies ทั้งหมดตาม `Gemfile.lock` ที่ล็อกเวอร์ชันไว้แน่นอน
3. รันชุดทดสอบทั้งหมด (unit test, integration test, system test)
4. ตรวจสอบ code style ด้วย linter
5. สแกนหาช่องโหว่ความปลอดภัยที่รู้จัก
6. รายงานผล **ผ่าน/ไม่ผ่าน** กลับมาที่ Pull Request ให้ทุกคนเห็นทันที

พูดง่าย ๆ CI คือการทำให้คำถามที่สำคัญที่สุดของการเขียนโค้ดร่วมกันเป็นทีม — **"การเปลี่ยนแปลงนี้ทำให้
อะไรพังไหม?"** — ตอบได้แบบอัตโนมัติ รวดเร็ว และเชื่อถือได้ ไม่ต้องพึ่งความจำหรือความขยันของใครคนใด
คนหนึ่งอีกต่อไป

**Continuous Deployment/Delivery (CD)** ต่อยอดจาก CI อีกขั้น: เมื่อ CI ผ่านทั้งหมดแล้ว (และมักจะ
ต้องผ่านการ review/approve ของมนุษย์ด้วย) ระบบจะ**อัตโนมัติ** deploy โค้ดเวอร์ชันนั้นออกไปที่
production (Continuous Deployment) หรืออย่างน้อยก็เตรียมพร้อมให้กดปุ่มเดียวเพื่อ deploy ได้ทันที
(Continuous Delivery) Part 076–077 จะลงลึกเรื่องกลไกการ deploy จริงด้วย Kamal — Part นี้จะพูดถึง
แนวคิด CD เพียงผิวเผินใน Step 750 เพื่อให้เห็นภาพว่า pipeline เชื่อมต่อกันอย่างไร

### ทำไม CI/CD ถึงสำคัญมากในการทำงานระดับมืออาชีพ

1. **จับบัคก่อน merge ไม่ใช่หลัง deploy** — ยิ่งเจอบัคเร็วเท่าไหร่ ต้นทุนในการแก้ยิ่งต่ำเท่านั้น
   บัคที่เจอตอนเขียนโค้ด (ผ่าน editor/linter) แก้ง่ายที่สุด บัคที่เจอใน CI ก่อน merge แก้ง่ายรองลงมา
   บัคที่เจอใน production หลัง deploy แล้วแพงที่สุด (อาจกระทบผู้ใช้จริง เสียชื่อเสียง ต้อง rollback)
2. **ทำให้ Pull Request review มีข้อมูลมากขึ้น** — reviewer ไม่ต้องเสียเวลาถามว่า "รัน test แล้วยัง"
   เพราะเห็น status check (✅/❌) ที่ GitHub แสดงในหน้า PR ทันที ทำให้ focus การ review ไปที่คุณภาพ
   ของ design/logic แทนที่จะกังวลเรื่องพื้นฐานว่าโค้ดรันได้หรือเปล่า
3. **บังคับมาตรฐานคุณภาพแบบเท่าเทียมกันทั้งทีม** — ไม่ว่าใครจะเขียนโค้ด ทุกคนต้องผ่านมาตรฐานเดียวกัน
   (test ต้องผ่าน, style ต้องตรง RuboCop, ห้ามมีช่องโหว่ที่ Brakeman ตรวจพบ) ไม่มีข้อยกเว้นเพราะ
   "คนนี้อาวุโสกว่า" หรือ "รีบมาก ขอผ่านก่อน"
4. **ทำให้กล้า merge บ่อย ๆ เป็นชิ้นเล็ก ๆ (trunk-based development)** — เมื่อมั่นใจว่า CI จะจับบัค
   ให้ ทีมกล้าที่จะ merge การเปลี่ยนแปลงเล็ก ๆ บ่อยครั้ง แทนที่จะสะสมการเปลี่ยนแปลงไว้เป็น branch
   ใหญ่ที่ merge ยากและเสี่ยง conflict สูง
5. **เอกสารที่เชื่อถือได้ของ "สถานะปัจจุบัน"** — ป้าย badge สีเขียวที่ README (`build: passing`)
   บอกทุกคนที่เข้ามาดู repository ว่าโค้ดใน main branch อยู่ในสถานะที่ใช้งานได้จริง ไม่ใช่แค่คำสัญญา

### ทำไมต้อง GitHub Actions

ตลาดเครื่องมือ CI/CD มีผู้เล่นหลายราย เช่น GitLab CI, CircleCI, Jenkins (self-hosted แบบดั้งเดิม),
Buildkite, Travis CI (ที่เคยนิยมมากในอดีตแต่ลดความนิยมลงหลังเปลี่ยนนโยบายค่าใช้จ่าย) แต่สำหรับ
โปรเจกต์ Rails ที่โฮสต์บน GitHub (ซึ่งเป็นกรณีส่วนใหญ่ของโปรเจกต์ Ruby ทั่วโลก) **GitHub Actions**
เป็นตัวเลือกเริ่มต้นที่สมเหตุสมผลที่สุดด้วยเหตุผล:

1. **ผนวกอยู่ใน GitHub อยู่แล้ว** — ไม่ต้องเชื่อมต่อบริการภายนอกเพิ่ม ไม่ต้องจัดการ webhook เอง
   สร้างไฟล์ `.github/workflows/*.yml` แล้วมันทำงานทันที
2. **ฟรีสำหรับ public repository อย่างไม่จำกัด และมี free tier ที่ใจกว้างสำหรับ private repository**
   (2,000 นาที/เดือนสำหรับบัญชีฟรี ที่ระดับ organization/plan ที่สูงขึ้นก็เพิ่มขึ้นตามลำดับ)
3. **Marketplace ของ reusable Actions ขนาดใหญ่มาก** — แทบทุกงานที่ต้องการมีคนทำ Action สำเร็จรูป
   ไว้ให้แล้ว (`ruby/setup-ruby`, `actions/cache`, `actions/upload-artifact` ที่จะเห็นใน Part นี้)
4. **และที่สำคัญที่สุดสำหรับ Part นี้: Rails 8 สร้าง workflow file ให้อัตโนมัติทันทีที่รัน
   `rails new`** ไม่ต้องเขียนอะไรจากศูนย์เลย — นี่คือสิ่งที่ Step 742 จะพาไปดูโดยละเอียด

---

## Step 742: `.github/workflows/ci.yml` ที่ Rails 8 สร้างให้อัตโนมัติ

### ทดลองสร้างแอปใหม่แล้วดูว่าได้อะไรมาบ้าง

```bash
rails new my_app --database=postgresql
```

หลังคำสั่งนี้รันเสร็จ ลองเปิดโฟลเดอร์ `.github/` ดู จะพบว่ามีไฟล์เกิดขึ้นมาให้แล้วโดยไม่ต้องทำอะไร
เพิ่มเติมเลยสักบรรทัด:

```
.github/
├── dependabot.yml
└── workflows/
    └── ci.yml
```

นี่คือฟีเจอร์ที่เพิ่มเข้ามาตั้งแต่ Rails 7.1 และแข็งแรงขึ้นเรื่อย ๆ ใน Rails 8: **generator สร้าง
CI pipeline ที่ใช้งานได้จริงให้ตั้งแต่วันแรก** ต่อไปนี้คือเนื้อหาจริงของ `ci.yml` ที่ได้จากการรัน
`rails new` ด้วย Rails 8.1.4 จริงในแซนด์บ็อกซ์ (คัดลอกมาทั้งไฟล์ ไม่มีการตัดต่อ):

```yaml
# .github/workflows/ci.yml (ของจริงจาก rails new ด้วย Rails 8.1.4)
name: CI

on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  scan_ruby:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Scan for common Rails security vulnerabilities using static analysis
        run: bin/brakeman --no-pager

      - name: Scan for known security vulnerabilities in gems used
        run: bin/bundler-audit

  scan_js:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Scan for security vulnerabilities in JavaScript dependencies
        run: bin/importmap audit

  lint:
    runs-on: ubuntu-latest
    env:
      RUBOCOP_CACHE_ROOT: tmp/rubocop
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Prepare RuboCop cache
        uses: actions/cache@v4
        env:
          DEPENDENCIES_HASH: ${{ hashFiles('.ruby-version', '**/.rubocop.yml', '**/.rubocop_todo.yml', 'Gemfile.lock') }}
        with:
          path: ${{ env.RUBOCOP_CACHE_ROOT }}
          key: rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}
          restore-keys: |
            rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-

      - name: Lint code for consistent style
        run: bin/rubocop -f github

  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=3

      # redis:
      #   image: valkey/valkey:8
      #   ports:
      #     - 6379:6379
      #   options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - name: Install packages
        run: sudo apt-get update && sudo apt-get install --no-install-recommends -y libpq-dev libvips

      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Run tests
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432
          # RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
          # REDIS_URL: redis://localhost:6379/0
        run: bin/rails db:test:prepare test

  system-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=3

      # redis:
      #   image: valkey/valkey:8
      #   ports:
      #     - 6379:6379
      #   options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - name: Install packages
        run: sudo apt-get update && sudo apt-get install --no-install-recommends -y libpq-dev libvips

      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Run System Tests
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432
          # RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
          # REDIS_URL: redis://localhost:6379/0
        run: bin/rails db:test:prepare test:system

      - name: Keep screenshots from failed system tests
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: screenshots
          path: ${{ github.workspace }}/tmp/screenshots
          if-no-files-found: ignore
```

### เดินอ่านทีละ job

Workflow นี้มี **5 jobs** ที่ทำงานคนละหน้าที่กันชัดเจน:

1. **`scan_ruby`** — สแกนความปลอดภัยฝั่ง Ruby สองชั้น: `bin/brakeman` (static analysis หาช่องโหว่
   ในโค้ดแอปเอง เช่น SQL Injection, Mass Assignment) และ `bin/bundler-audit` (ตรวจว่า gem ที่ใช้
   มีเวอร์ชันที่มี CVE ที่รู้จักหรือไม่) — ลงลึกใน Step 748
2. **`scan_js`** — รัน `bin/importmap audit` ตรวจสอบว่า JavaScript library ที่ประกาศไว้ใน
   `config/importmap.rb` มีช่องโหว่ที่รู้จักหรือไม่ (Rails 8 ใช้ Importmap เป็นค่าเริ่มต้นแทน
   `node_modules`/webpack เลยไม่มี `npm audit` แบบที่ระบบ frontend ทั่วไปใช้)
3. **`lint`** — รัน `bin/rubocop -f github` ตรวจสอบ code style พร้อม cache ผลการตรวจสอบไว้เพื่อให้
   รันเที่ยวถัดไปเร็วขึ้น — ลงลึกใน Step 745 และ 747
4. **`test`** — รันชุดทดสอบหลัก (`bin/rails db:test:prepare test` เป็น Minitest ที่มาเป็นค่า
   เริ่มต้นของ Rails) พร้อม PostgreSQL service container จริง — ลงลึกใน Step 744 และ 746
5. **`system-test`** — เหมือนกับ `test` แต่รันเฉพาะ System Test (ที่ใช้ Capybara ขับเบราว์เซอร์จริง
   ทดสอบ user flow แบบ end-to-end) และมี step พิเศษที่เก็บ screenshot ไว้เป็น artifact เมื่อ test
   ล้มเหลว (`if: failure()`) เพื่อช่วยดีบักโดยไม่ต้องเดา

สังเกตว่า **ทุก job แยกเป็นอิสระจากกันโดยสมบูรณ์** (ไม่มี `needs:` เชื่อมกันเลยใน workflow ต้นฉบับ)
หมายความว่าทั้ง 5 jobs จะรัน**พร้อมกัน** (parallel) บน runner คนละเครื่องทันทีที่ trigger เข้ามา ทำให้
ได้ผลลัพธ์เร็วกว่าการรันทีละ job ตามลำดับมาก — ถ้า `scan_ruby` ใช้เวลา 1 นาที และ `test` ใช้เวลา 3
นาที ทั้ง workflow จะเสร็จใน ~3 นาที ไม่ใช่ 4 นาที

### ไฟล์ที่มาคู่กัน: `.github/dependabot.yml`

Rails ยังสร้างไฟล์ตั้งค่า **Dependabot** มาให้ด้วย ซึ่งเป็นฟีเจอร์ของ GitHub เองที่คอยตรวจสอบ
dependency ที่ล้าสมัยหรือมีช่องโหว่ แล้วเปิด Pull Request อัปเดตให้อัตโนมัติ (คนละกลไกกับ CI ที่พูด
ถึงในไฟล์นี้ แต่ทำงานเสริมกัน — Dependabot คอยเสนอการอัปเดต ส่วน CI คอยยืนยันว่าอัปเดตนั้นไม่ทำให้
อะไรพัง):

```yaml
# .github/dependabot.yml (ของจริงจาก rails new)
version: 2
updates:
- package-ecosystem: bundler
  directory: "/"
  schedule:
    interval: weekly
  open-pull-requests-limit: 10
- package-ecosystem: github-actions
  directory: "/"
  schedule:
    interval: weekly
  open-pull-requests-limit: 10
```

ไฟล์นี้บอกให้ Dependabot ตรวจสอบทั้ง gem (`bundler`) และ Actions ที่ใช้ในไฟล์ workflow เอง
(`github-actions` — เพราะ `uses: actions/checkout@v6` ก็มีเวอร์ชันที่ควรอัปเดตเช่นกัน) ทุกสัปดาห์
และเปิด PR ได้สูงสุด 10 PR ค้างพร้อมกัน

### เกร็ดความรู้: Rails 8.1 ยังมี local CI runner ในตัวด้วย

เมื่อสำรวจไฟล์ `bin/` ของแอปที่สร้างใหม่ จะพบไฟล์ `bin/ci` ที่ไม่เคยมีมาก่อนใน Rails รุ่นเก่า:

```ruby
#!/usr/bin/env ruby
# bin/ci (ของจริงจาก rails new ด้วย Rails 8.1.4)
require_relative "../config/boot"
require "active_support/continuous_integration"

CI = ActiveSupport::ContinuousIntegration
require_relative "../config/ci.rb"
```

พร้อมไฟล์ `config/ci.rb` ที่นิยาม step ต่าง ๆ ไว้แบบเดียวกับใน `ci.yml`:

```ruby
# config/ci.rb (ของจริงจาก rails new)
# Run using bin/ci

CI.run do
  step "Setup", "bin/setup --skip-server"

  step "Style: Ruby", "bin/rubocop"

  step "Security: Gem audit", "bin/bundler-audit"
  step "Security: Importmap vulnerability audit", "bin/importmap audit"
  step "Security: Brakeman code analysis", "bin/brakeman --quiet --no-pager --exit-on-warn --exit-on-error"
  step "Tests: Rails", "bin/rails test"
  step "Tests: Seeds", "env RAILS_ENV=test bin/rails db:seed:replant"

  # Optional: Run system tests
  # step "Tests: System", "bin/rails test:system"

  # Optional: set a green GitHub commit status to unblock PR merge.
  # Requires the `gh` CLI and `gh extension install basecamp/gh-signoff`.
  # if success?
  #   step "Signoff: All systems go. Ready for merge and deploy.", "gh signoff"
  # else
  #   failure "Signoff: CI failed. Do not merge or deploy.", "Fix the issues and try again."
  # end
end
```

`ActiveSupport::ContinuousIntegration` เป็นฟีเจอร์ใหม่ที่ให้รัน "ชุดเดียวกันกับที่ CI จะรัน" ในเครื่อง
ตัวเองได้ด้วยคำสั่งเดียวคือ `bin/ci` — แสดงผลแต่ละ step แบบมีสีและสรุปผลรวมท้ายสุด **นี่ไม่ได้แทนที่
GitHub Actions** (เพราะไม่ได้ยืนยันว่าทำงานได้ในสภาพแวดล้อมที่สะอาดจริงเหมือน CI จริง — ยังรันบน
เครื่องตัวเองที่อาจมีร่องรอยการตั้งค่าเก่าค้างอยู่) แต่มีประโยชน์มากสำหรับ**เช็คตัวเองเร็ว ๆ ก่อน push**
โดยไม่ต้องรอ CI จริงที่ใช้เวลานานกว่า และไม่กิน CI minutes ฟรีที่ GitHub ให้มา — เป็นการเติมเต็ม
pipeline ไม่ใช่แข่งกัน Part นี้จะโฟกัสที่ GitHub Actions เป็นหลักตามหัวข้อ แต่ควรรู้ว่า `bin/ci` มีอยู่
และใช้เป็นเครื่องมือเสริมได้

---

## Step 743: กายวิภาคของ GitHub Actions — workflow, job, step, runner, trigger

ก่อนจะไปปรับแต่ง workflow ที่ Rails สร้างให้ ต้องเข้าใจคำศัพท์และโครงสร้างพื้นฐานของ GitHub Actions
ให้แน่นก่อน มิเช่นนั้นการอ่าน YAML จะเหมือนท่องจำแทนที่จะเข้าใจ

### ลำดับชั้นของแนวคิด

```
Workflow (ไฟล์ .yml หนึ่งไฟล์)
 └── Job (ทำงานบน runner หนึ่งเครื่อง)
      └── Step (คำสั่งหรือ action หนึ่งอย่าง ทำงานตามลำดับจากบนลงล่าง)
```

- **Workflow** คือไฟล์ YAML หนึ่งไฟล์ในโฟลเดอร์ `.github/workflows/` หนึ่ง repository มีได้หลาย
  workflow พร้อมกัน (เช่นแยกไฟล์ `ci.yml`, `deploy.yml`, `stale-issues.yml` ก็ได้) แต่ละไฟล์ทำงาน
  เป็นอิสระจากกัน
- **Job** คือกลุ่มของ step ที่รันบน **runner** เครื่องเดียวกัน โดย default แล้ว jobs ในไฟล์เดียวกัน
  รันพร้อมกัน (parallel) เว้นแต่จะระบุ `needs:` เพื่อบังคับลำดับ (เช่น job `deploy` ต้องรอ job
  `test` ผ่านก่อน — จะเห็นใน Step 750)
- **Step** คือคำสั่งเดียวภายใน job หนึ่ง รันตามลำดับจากบนลงล่างในเครื่องเดียวกัน (แชร์ filesystem,
  ตัวแปรที่ export ไว้ระหว่าง step ได้) มีสองแบบคือ `run:` (รันคำสั่ง shell ตรง ๆ) กับ `uses:`
  (เรียกใช้ Action สำเร็จรูปจาก Marketplace)
- **Runner** คือเครื่องเสมือนที่ job รันอยู่จริง ๆ ปกติใช้ **GitHub-hosted runner** อย่าง
  `ubuntu-latest` (เครื่องที่ GitHub เตรียมไว้ให้ สร้างใหม่สะอาดทุกครั้งที่รัน แล้วทำลายทิ้งเมื่อ job
  จบ) มี `macos-latest`/`windows-latest` ให้เลือกด้วยถ้าต้องการทดสอบข้าม OS ส่วน **self-hosted
  runner** คือเครื่องของเราเองที่ติดตั้ง GitHub Actions agent ไว้ (นิยมใช้เมื่อต้องการ GPU เฉพาะทาง
  หรือควบคุม environment เอง) — บทเรียนนี้ใช้ GitHub-hosted runner ตามค่า default ของ Rails
- **Trigger** (คีย์ `on:`) กำหนดว่า workflow จะเริ่มทำงานเมื่อเกิดเหตุการณ์อะไร

### Trigger: `on:` ในไฟล์จริงของ Rails

```yaml
on:
  pull_request:
  push:
    branches: [ main ]
```

อ่านได้ว่า: รัน workflow นี้ทุกครั้งที่ (1) มี event เกี่ยวกับ pull request เกิดขึ้น (เปิดใหม่, push
commit เพิ่มเข้า branch ที่มี PR เปิดอยู่, reopen ฯลฯ — ค่า default ของ `pull_request` ครอบคลุม
event types เหล่านี้โดยไม่ต้องระบุเพิ่ม) และ (2) มีการ push เข้า branch `main` โดยตรง (ซึ่งมักเกิด
ขึ้นตอน merge PR เข้า main สำเร็จ)

**ทำไมต้องมีทั้งสอง trigger ทั้งที่ push เข้า main ก็มักจะมาจาก PR ที่ผ่าน CI แล้ว?** เพราะการ merge
บางแบบ (เช่น merge commit ที่รวมหลาย branch, หรือ merge ผ่าน GitHub UI ด้วยปุ่ม "Update branch")
อาจสร้าง commit ใหม่ที่ไม่เคยผ่านการรัน CI แบบตรง ๆ มาก่อน การรัน CI ซ้ำอีกครั้งตอน push เข้า main
จึงเป็นเซฟตี้เน็ตสุดท้ายก่อนที่โค้ดจะกลายเป็น "ของจริง" บน main branch นอกจากนี้ trigger อื่น ๆ ที่พบ
บ่อยแต่ไม่มีในไฟล์เริ่มต้นของ Rails ได้แก่ `schedule:` (รันตามเวลา cron เช่น สแกนความปลอดภัยทุกคืน)
และ `workflow_dispatch:` (ให้กดปุ่มรันเองผ่าน GitHub UI ได้ เช่น ปุ่ม "Run workflow" สำหรับ deploy
manual)

### `uses:` vs `run:`

```yaml
steps:
  - name: Checkout code
    uses: actions/checkout@v6        # เรียกใช้ Action สำเร็จรูป

  - name: Set up Ruby
    uses: ruby/setup-ruby@v1
    with:
      bundler-cache: true            # พารามิเตอร์ที่ส่งให้ Action

  - name: Lint code for consistent style
    run: bin/rubocop -f github       # รันคำสั่ง shell ตรง ๆ
```

- `uses: actions/checkout@v6` คือ Action มาตรฐานที่ทำหน้าที่ **clone repository เข้ามาใน runner**
  (โดย default รันแรกสุดของแทบทุก job เสมอ เพราะ runner เริ่มต้นเป็นเครื่องเปล่า ไม่มีโค้ดของเรา
  อยู่เลย) เลขเวอร์ชันต่อท้าย `@v6` คือ tag ที่ผูกไว้ — ควรระบุเวอร์ชันเสมอ (ไม่ใช้ `@main` หรือไม่
  ระบุเวอร์ชันเลย) เพื่อความปลอดภัย (ป้องกันไม่ให้ Action เปลี่ยนพฤติกรรมโดยไม่รู้ตัวถ้าเจ้าของ
  Action แก้โค้ดใน branch หลัก) และความเสถียร (build วันนี้กับพรุ่งนี้ใช้ logic เดียวกัน)
- `uses: ruby/setup-ruby@v1` คือ Action ที่ติดตั้ง Ruby เวอร์ชันที่ต้องการ (อ่านจากไฟล์
  `.ruby-version` ของ repository โดยอัตโนมัติถ้าไม่ระบุ `ruby-version:` ตรง ๆ) แล้วรัน
  `bundle install` ให้ด้วยถ้าตั้ง `bundler-cache: true` — ลงลึกเรื่อง caching ใน Step 745
- `run:` คือคำสั่ง shell (bash บน Linux runner) รันตรง ๆ เหมือนพิมพ์ในเทอร์มินัล รองรับหลายบรรทัด
  ด้วย YAML block scalar (`|`) เช่น

```yaml
run: |
  bin/rails db:test:prepare
  bundle exec rspec
```

### env, secrets, และตัวแปรพิเศษของ GitHub

```yaml
env:
  RAILS_ENV: test
  DATABASE_URL: postgres://postgres:postgres@localhost:5432
  # RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
```

`env:` ตั้งค่าตัวแปรสภาพแวดล้อมให้ step หรือ job นั้น ๆ เห็น ส่วน `${{ secrets.X }}` คือวิธีดึงค่าลับ
ที่ตั้งไว้ใน **Settings → Secrets and variables → Actions** ของ repository (เช่น
`RAILS_MASTER_KEY` ที่ต้องใช้ถอดรหัส `config/credentials.yml.enc` — ตรงกับที่สอนใน Part 074) ค่า
เหล่านี้จะถูกปิดบัง (masked) อัตโนมัติในหน้า log ไม่ให้ใครเห็นค่าจริงแม้จะพิมพ์ออกทาง `echo` โดย
ไม่ตั้งใจ นอกจากนี้ยังมีตัวแปรระบบที่ GitHub เตรียมให้ฟรีโดยไม่ต้องตั้งค่าเอง เช่น `github.ref`,
`github.event_name`, `runner.os`, `github.run_id` ที่จะเห็นใช้งานจริงใน Step 745 และ 750

### ข้อควรระวังเรื่อง YAML syntax

GitHub Actions ใช้ YAML ล้วน ๆ ซึ่งมีกับดักที่พบบ่อย:

- **Indentation ต้องเป๊ะ** — ใช้ space เท่านั้น ห้ามผสม tab เด็ดขาด (YAML ไม่รองรับ tab) ระดับการ
  ย่อหน้าที่ผิดแม้แค่ช่องเดียวทำให้โครงสร้างข้อมูลเปลี่ยนไปเงียบ ๆ โดยไม่มี error ชัดเจน
- **คีย์ `on:` มีประวัติเป็นกับดักที่มีชื่อเสียง** — ทดสอบจริงด้วย Ruby's `YAML.safe_load` พบว่า
  ถ้าเขียน `on:` โดยไม่ใส่เครื่องหมายคำพูด YAML parser ทั่วไปที่ยึดสเปก YAML 1.1 (รวมถึงของ Ruby)
  จะตีความคำว่า `on` เป็นค่า **boolean `true`** ไม่ใช่ string `"on"` (เพราะ YAML 1.1 กำหนดให้คำว่า
  `on`/`off`/`yes`/`no` เป็น alias ของ boolean) ทำให้เมื่อโหลดไฟล์ด้วยไลบรารี YAML ทั่วไปแล้วเรียก
  `hash["on"]` อาจได้ `nil` แทนที่จะได้ค่า trigger ที่ตั้งไว้ (ต้องเรียก `hash[true]` แทน) — GitHub
  Actions เองจัดการเรื่องนี้ให้เรียบร้อยแล้วเมื่อรันจริงบนแพลตฟอร์ม ไม่กระทบผู้ใช้งานทั่วไปที่แค่
  เขียนไฟล์ workflow แต่เป็นความรู้ที่สำคัญถ้าจะเขียนสคริปต์หรือเครื่องมือที่ต้อง parse ไฟล์
  workflow ของ Actions เอง
- **ทดสอบ syntax ก่อน push เสมอ** — ใช้ `ruby -ryaml -e 'YAML.safe_load(File.read("path/to/file.yml"))'`
  หรือ extension ของ VS Code สำหรับ GitHub Actions (ตรวจ schema เฉพาะของ Actions ได้ละเอียดกว่า
  การเช็ค YAML เฉย ๆ) เพื่อจับ syntax error ก่อนที่จะต้อง push แล้วรอดู CI ล้มเหลวเพราะไฟล์ตัวเอง
  parse ไม่ผ่านตั้งแต่ต้น

---

## Step 744: Service Containers — ตั้งค่า PostgreSQL/Redis ให้ job ทดสอบมีฐานข้อมูลจริงใช้งาน

### ปัญหา: test ต้องการฐานข้อมูลจริง แต่ runner เป็นเครื่องเปล่า

Job `test` ต้องรัน `bin/rails db:test:prepare test` ซึ่งต้องการ PostgreSQL ที่ใช้งานได้จริง แต่
GitHub-hosted runner เริ่มต้นเป็นเครื่องเปล่าที่ไม่มี PostgreSQL ติดตั้งไว้ ทางแก้คือ **service
containers** — ฟีเจอร์ของ GitHub Actions ที่รัน Docker container คู่ขนานไปกับ container หลักของ job
โดยทั้งสองอยู่ใน network เดียวกัน ทำให้ job เข้าถึง service ผ่าน `localhost` ได้ทันที

```yaml
services:
  postgres:
    image: postgres
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - 5432:5432
    options: --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=3
```

อ่านทีละส่วน:

- `image: postgres` — ดึง Docker image ชื่อ `postgres` (official image จาก Docker Hub, tag
  `latest` โดยปริยาย) มารันเป็น container ขึ้นมาก่อนที่ step ใด ๆ ของ job นี้จะเริ่ม
- `env:` — ตั้งค่าตัวแปรสภาพแวดล้อมให้ container ของ PostgreSQL เอง (ไม่ใช่ของ job) กำหนด
  username/password ที่จะใช้เชื่อมต่อ — สังเกตว่า **ไม่ได้ตั้ง `POSTGRES_DB`** หมายความว่า
  PostgreSQL จะสร้างฐานข้อมูลเริ่มต้นชื่อ `postgres` ให้ ซึ่งไม่เป็นไรเพราะ
  `bin/rails db:test:prepare` จะเป็นคนสร้างฐานข้อมูลจริงที่แอปต้องการ (ตามชื่อใน
  `config/database.yml`) อยู่แล้วผ่าน connection นี้
- `ports: - 5432:5432` — map port 5432 ของ container ออกมาที่ port 5432 ของ runner ทำให้ขั้นตอน
  ถัดไปที่รันบน runner (ไม่ใช่ใน container ของ postgres) เชื่อมต่อผ่าน `localhost:5432` ได้ตรง ๆ
- `options: --health-cmd=...` — ตั้ง healthcheck ให้ GitHub Actions รอจนกว่า PostgreSQL จะพร้อม
  รับ connection จริง (`pg_isready` คือคำสั่งของ PostgreSQL เองที่เช็คว่า server พร้อมหรือยัง) ก่อน
  จะเริ่มรัน step ถัดไป ป้องกันปัญหา race condition ที่ test อาจล้มเหลวเพราะพยายามเชื่อมต่อไปยัง
  ฐานข้อมูลที่ยังไม่พร้อมรับการเชื่อมต่อ (ยัง booting อยู่)

การตั้งค่าตรงนี้**ตรงกับที่ทดสอบจริงในแซนด์บ็อกซ์เป๊ะ**: ใช้ `DATABASE_URL=postgres://postgres:postgres@localhost:5432`
กับ PostgreSQL ที่รันอยู่ในเครื่องจริง (ไม่ใช่ container แต่พฤติกรรมการเชื่อมต่อเหมือนกันทุก
ประการ) แล้วรัน `bin/rails db:test:prepare test` ผ่านได้จริงทั้ง 7 test ที่ scaffold สร้างให้

### Redis service — สำหรับแอปที่ต้องการ background job / ActionCable ระหว่างทดสอบ

สังเกตว่าในไฟล์จริงที่ Rails generate มี block ของ Redis อยู่ **แต่ comment ปิดไว้**:

```yaml
      # redis:
      #   image: valkey/valkey:8
      #   ports:
      #     - 6379:6379
      #   options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5
```

Rails 8 ใช้ `solid_cache`/`solid_queue` เป็นค่าเริ่มต้น (เก็บ cache และ background job ไว้ใน
PostgreSQL ตัวเดียวกัน ไม่ต้องพึ่ง Redis แยกต่างหาก) จึง comment ส่วนนี้ไว้เป็นค่าเริ่มต้น แต่ถ้า
โปรเจกต์ใช้ **Sidekiq** (ตามที่สอนใน Part 062) หรือ ActionCable ที่ต้องการ Redis adapter จริง
จำเป็นต้องเปิด comment ส่วนนี้กลับมา:

```yaml
services:
  postgres:
    image: postgres
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - 5432:5432
    options: --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=3

  redis:
    image: valkey/valkey:8
    ports:
      - 6379:6379
    options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5

steps:
  # ...
  - name: Run tests
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432
      REDIS_URL: redis://localhost:6379/0
    run: bin/rails db:test:prepare test
```

(สังเกตว่า Rails ใช้ image ชื่อ `valkey/valkey` แทน `redis` — Valkey คือ fork แบบ open-source ของ
Redis ที่เกิดขึ้นหลัง Redis เปลี่ยนสัญญาอนุญาตใช้งานในปี 2024 ใช้แทนกันได้ทุกประการเพราะ protocol
เดียวกัน 100%) ทดสอบจริงในแซนด์บ็อกซ์ยืนยันว่า `redis-server` เปิดใช้งานแล้วตอบ `PONG` ต่อคำสั่ง
`redis-cli ping` และ `set`/`get` คีย์ได้ปกติ — พฤติกรรมเดียวกับที่ healthcheck `redis-cli ping`
ใน service container ข้างต้นตรวจสอบ

### หลักการสำคัญ: service container คือ container แยก ไม่ใช่ dependency ที่ต้องติดตั้งเอง

ข้อดีของวิธีนี้เทียบกับการ apt-get ติดตั้ง PostgreSQL/Redis ลงตรง ๆ บน runner คือ **ได้เวอร์ชันที่
ตรงกับที่ใช้จริงบน production เป๊ะ** (ระบุ tag ของ image ได้ เช่น `postgres:16` ให้ตรงกับเวอร์ชัน
ที่ deploy จริง) รวมถึง **เริ่มต้นสะอาดทุกครั้ง** ไม่มีข้อมูลทดสอบเก่าตกค้างข้ามการรัน — แก้ปัญหา
"test ผ่านบนเครื่องฉัน เพราะมีข้อมูลเก่าที่ทำให้ query ทำงานได้ แต่พังบนเครื่องอื่น" ได้อย่างสิ้นเชิง

---

## Step 745: Caching bundler/gem dependencies เพื่อให้ CI เร็วขึ้นอย่างมีนัยสำคัญ

### ปัญหา: runner เป็นเครื่องเปล่าทุกครั้ง แปลว่าต้อง bundle install ใหม่ทุกครั้ง

เนื่องจาก GitHub-hosted runner ถูกทำลายทิ้งทันทีที่ job จบ (ไม่มี state เหลือข้ามการรัน) ถ้าไม่ทำ
อะไรเพิ่มเติม ทุกครั้งที่ workflow รัน `bundle install` จะต้องดาวน์โหลดและติดตั้ง gem ทั้งหมดใหม่
ตั้งแต่ศูนย์ ซึ่งรวมถึง gem ที่มี native extension (เช่น `pg`, `nokogiri`) ที่ต้อง compile ด้วย ซึ่ง
ใช้เวลานาน

**ตัวเลขจริงที่วัดได้ในแซนด์บ็อกซ์**: แอป Rails ตั้งต้นเปล่า ๆ (23 gems โดยตรงใน Gemfile, 123 gems
รวม dependency) รัน `bundle install` แบบไม่มี cache เลยใช้เวลา **42.5 วินาที** ในขณะที่รันซ้ำครั้งที่
สองแบบที่ gem ถูกติดตั้งไว้แล้ว (จำลองสถานการณ์ cache hit) ใช้เวลาเพียง **0.58 วินาที** — เร็วขึ้น
กว่า **70 เท่า** สำหรับแอปจริงที่มี gem จำนวนมากกว่านี้ (โดยเฉพาะที่มี `nokogiri`, `ffi`,
`grpc` ที่ compile นาน) ส่วนต่างนี้จะยิ่งมหาศาลขึ้นไปอีก เมื่อคูณด้วยจำนวน job ที่รันพร้อมกัน (5 jobs
ในไฟล์นี้ ต่างก็ต้อง `bundle install` แยกกันคนละ runner) ยิ่งเห็นความสำคัญของการ cache ชัดเจน

### `bundler-cache: true` — วิธีที่ง่ายที่สุดและเป็นค่าเริ่มต้นของ Rails

```yaml
- name: Set up Ruby
  uses: ruby/setup-ruby@v1
  with:
    bundler-cache: true
```

Action `ruby/setup-ruby` มีความสามารถแฝงอยู่ในตัว: เมื่อตั้ง `bundler-cache: true` มันจะ:

1. คำนวณ cache key จาก hash ของไฟล์ `Gemfile.lock` รวมกับข้อมูลเวอร์ชัน Ruby และ OS ของ runner
2. เรียก `actions/cache` (Action คนละตัวแต่ทำงานร่วมกันภายใน) เพื่อกู้คืน (restore) โฟลเดอร์ gem ที่
   เคย cache ไว้จากการรันครั้งก่อนที่มี `Gemfile.lock` เหมือนกันทุกตัวอักษร
3. รัน `bundle install` ตามปกติ — ถ้า cache ตรงกัน ขั้นตอนนี้จะเจอ gem ที่ติดตั้งไว้แล้วเกือบทั้งหมด
   ทำให้เสร็จเร็วมาก (ตรงกับ 0.58 วินาทีที่วัดได้)
4. หลัง job จบ (ถ้า cache key ไม่เคยมีมาก่อน) จะบันทึก (save) cache ใหม่ไว้ให้การรันครั้งถัดไปใช้

จุดสำคัญคือ **cache key ผูกกับ hash ของ `Gemfile.lock`** ดังนั้นทันทีที่มีการเพิ่ม/ลบ/เปลี่ยนเวอร์ชัน
gem (ซึ่งทำให้ `Gemfile.lock` เปลี่ยน) cache เก่าจะไม่ตรงกันอีกต่อไป (cache miss) และระบบจะ
`bundle install` ใหม่ทั้งหมดโดยอัตโนมัติ — ไม่มีความเสี่ยงที่จะได้ gem เวอร์ชันเก่าค้างจาก cache
ทั้งหมดนี้เกิดขึ้นเบื้องหลังโดยที่ Rails **ไม่ต้องเขียน `actions/cache` เองเลยสักบรรทัด** — เพียงตั้ง
`bundler-cache: true` ตัวเดียวเท่านั้น

### `actions/cache` แบบ manual — ใช้กับสิ่งที่ไม่ใช่ gem

Job `lint` ในไฟล์จริงของ Rails ใช้ `actions/cache` แบบเขียนเองเพิ่มเติม เพื่อ cache **สิ่งที่ไม่ใช่
gem** นั่นคือผลลัพธ์การตรวจสอบของ RuboCop เอง (`tmp/rubocop`):

```yaml
env:
  RUBOCOP_CACHE_ROOT: tmp/rubocop
steps:
  # ... checkout, setup ruby (พร้อม bundler-cache: true) ...

  - name: Prepare RuboCop cache
    uses: actions/cache@v4
    env:
      DEPENDENCIES_HASH: ${{ hashFiles('.ruby-version', '**/.rubocop.yml', '**/.rubocop_todo.yml', 'Gemfile.lock') }}
    with:
      path: ${{ env.RUBOCOP_CACHE_ROOT }}
      key: rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}
      restore-keys: |
        rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-
```

ตรงนี้มี**สอง layer ของ cache ที่แยกกันชัดเจนแต่ทำงานเสริมกัน**:

- **Layer 1 (gem cache)** — จัดการโดย `bundler-cache: true` ในทุก job อัตโนมัติ คือสิ่งที่ทำให้
  `bin/rubocop` ตัวมันเองพร้อมใช้งานเร็ว (ไม่ต้อง `gem install rubocop` ใหม่ทุกครั้ง)
- **Layer 2 (RuboCop's own analysis cache)** — จัดการโดย `actions/cache` แบบ manual นี้ คือ cache
  ผลการ**วิเคราะห์**ไฟล์แต่ละไฟล์ของ RuboCop เอง (ไม่ใช่ตัว gem) RuboCop มีกลไก incremental cache
  ในตัวอยู่แล้วที่จำผลตรวจสอบไฟล์ที่ไม่เปลี่ยนแปลง ไม่ต้องวิเคราะห์ซ้ำ — การ cache โฟลเดอร์
  `tmp/rubocop` ข้าม CI run ทำให้ RuboCop ใช้ประโยชน์จาก incremental cache นี้ได้แม้จะรันบน runner
  คนละเครื่องในแต่ละครั้ง (ปกติ incremental cache จะมีประโยชน์เฉพาะตอนรันซ้ำบนเครื่องเดียวกัน)

สังเกต cache key ที่ซับซ้อนกว่าปกติ: `${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}`
เป็น ternary expression ของ GitHub Actions (`condition && true_value || false_value`) ที่แปลว่า
"ถ้ากำลังรันบน default branch (main) ให้ผูก key กับ `run_id` เฉพาะของรันนี้ (การันตีว่า cache ใหม่
เสมอบน main); ถ้าไม่ใช่ (กำลังรันบน PR branch) ให้ใช้คำว่า `default` ตรง ๆ (ใช้ cacheร่วมกันได้ข้าม
PR หลายอัน)" ผสานกับ `restore-keys` ที่ตัด `run_id`/`default` ออก ทำให้ระบบยัง fallback ไปใช้ cache
เก่าที่ใกล้เคียงที่สุดได้แม้ key แบบเต็มจะไม่ตรงพอดี — เป็นรูปแบบมาตรฐานของการทำ cache แบบ
"精确 key ก่อน แล้ว fallback แบบกว้างขึ้นถ้าไม่เจอ"

### แนวทางเดียวกันนำไปใช้กับสิ่งอื่นได้ — เช่น `ruby-advisory-db`

`bin/bundler-audit` (ที่จะพูดถึงใน Step 748) ต้องดาวน์โหลดฐานข้อมูลช่องโหว่จาก
`ruby-advisory-db` (repository บน GitHub ที่รวบรวม CVE ของ gem) ทุกครั้งที่รัน — ทดสอบจริงพบว่า
โฟลเดอร์นี้มีขนาด **12MB** และการ `git clone --depth 1` ใหม่ทุกครั้งแม้จะเร็วในเครือข่ายที่ดี แต่ก็
เป็นการพึ่งพา network เพิ่มเติมที่ไม่จำเป็นทุกครั้งที่ CI รัน (เสี่ยง flaky ถ้า GitHub ช้าหรือ
rate-limit) เราจะเพิ่ม caching ให้ขั้นตอนนี้ในแบบฝึกหัดท้าย Part นี้เช่นกัน

---

## Step 746: รันชุดทดสอบจริงด้วย RSpec ใน CI (เชื่อมกับ Phase 6)

### Rails ใช้ Minitest เป็นค่าเริ่มต้น แต่หลักสูตรนี้สอน RSpec

Workflow ที่ Rails สร้างให้ใช้คำสั่ง `bin/rails db:test:prepare test` ซึ่งรัน **Minitest** (test
framework ที่มากับ Rails core) แต่ Phase 6 ของหลักสูตรนี้ (Part 046–050) สอน **RSpec** ร่วมกับ
FactoryBot, Capybara, และ SimpleCov ซึ่งเป็นชุดเครื่องมือที่ทีม Rails มืออาชีพส่วนใหญ่นิยมใช้มากกว่า
Minitest ในทางปฏิบัติ ข่าวดีคือการสลับ CI ให้รัน RSpec แทน **แก้แค่คำสั่งใน step เดียว** โดยไม่ต้อง
แตะโครงสร้าง workflow ส่วนอื่นเลย

### ติดตั้ง RSpec เข้าไปในแอป (ทบทวนจาก Part 046)

```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end

group :test do
  gem "simplecov", require: false
end
```

```bash
bundle install
bin/rails generate rspec:install
```

### เขียน model spec จริง (verified)

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
  validates :body, presence: true, length: { minimum: 10 }
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { "CI/CD ด้วย GitHub Actions" }
    body { "เนื้อหาตัวอย่างสำหรับทดสอบ model spec ที่ยาวพอผ่าน validation" }
  end
end
```

```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  it "มี factory ที่ valid" do
    expect(build(:post)).to be_valid
  end

  it "ต้องมี title" do
    post = build(:post, title: nil)
    expect(post).not_to be_valid
    expect(post.errors[:title]).to include("can't be blank")
  end

  it "body ต้องยาวอย่างน้อย 10 ตัวอักษร" do
    post = build(:post, body: "สั้นไป")
    expect(post).not_to be_valid
  end
end
```

อย่าลืมเปิดใช้งาน syntax สั้นของ FactoryBot ใน `spec/rails_helper.rb` (ทบทวนจาก Part 047):

```ruby
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  # ... ค่าอื่น ๆ ที่ generator สร้างให้ ...
end
```

รันจริงในแซนด์บ็อกซ์ด้วย PostgreSQL จริงตัวเดียวกับที่ CI service container จะใช้:

```bash
export RAILS_ENV=test
export DATABASE_URL="postgres://postgres:postgres@localhost:5432"
bin/rails db:test:prepare
bundle exec rspec
```

```
...

Finished in 0.0377 seconds (files took 1.12 seconds to load)
3 examples, 0 failures
```

ผ่านทั้ง 3 examples จริง ยืนยันว่า pattern การเขียน model spec ที่สอนใน Part 046 ทำงานได้ปกติทุก
ประการเมื่อรันผ่าน `DATABASE_URL` แบบเดียวกับที่ service container ของ GitHub Actions จะสร้างให้

### แก้ไข workflow: เปลี่ยนคำสั่งใน job `test`

```yaml
# .github/workflows/ci.yml — เปลี่ยนจาก Minitest เป็น RSpec
- name: Run tests
  env:
    RAILS_ENV: test
    DATABASE_URL: postgres://postgres:postgres@localhost:5432
  run: |
    bin/rails db:test:prepare
    bundle exec rspec
```

แค่นี้เอง — ไม่ต้องแตะ `services:`, ไม่ต้องแตะ `Install packages`, ไม่ต้องแตะ `Set up Ruby` เลย
เพราะ `bundler-cache: true` จะติดตั้ง `rspec-rails`/`factory_bot_rails` ให้อัตโนมัติทันทีที่มันอยู่ใน
`Gemfile`/`Gemfile.lock` โดยไม่ต้องแก้ไข workflow เพิ่มเติมแต่อย่างใด — นี่คือประโยชน์ของการที่
`bundle install` (ผ่าน `bundler-cache: true`) เป็นขั้นตอนที่อ่าน `Gemfile.lock` ตรง ๆ แทนที่จะ
hardcode รายชื่อ gem ไว้ในไฟล์ workflow

### System test ด้วย Capybara ก็ทำงานหลักการเดียวกัน

Job `system-test` ที่ Rails สร้างให้เรียก `bin/rails db:test:prepare test:system` (Minitest system
test) การแปลงมาใช้ RSpec + Capybara system spec (ทบทวนจาก Part 048) ก็ทำแบบเดียวกัน:

```yaml
- name: Run System Tests
  env:
    RAILS_ENV: test
    DATABASE_URL: postgres://postgres:postgres@localhost:5432
  run: |
    bin/rails db:test:prepare
    bundle exec rspec spec/system
```

runner `ubuntu-latest` ของ GitHub มาพร้อม Chrome ติดตั้งไว้ให้แล้วเป็นค่าเริ่มต้น (ใช้กับ
`Capybara.javascript_driver = :selenium_chrome_headless` ที่ Rails ตั้งไว้ให้เมื่อรัน
`bin/rails generate system_test`) จึงไม่ต้องติดตั้งเบราว์เซอร์เพิ่มเติมเองในกรณีทั่วไป — ส่วน step
`Keep screenshots from failed system tests` ที่ใช้ `actions/upload-artifact@v4` พร้อมเงื่อนไข
`if: failure()` ยังใช้ได้เหมือนเดิมโดยไม่ต้องแก้ไข (Capybara + RSpec เขียนไฟล์ screenshot ไปที่
`tmp/screenshots` แบบเดียวกับ Minitest system test)

---

## Step 747: รัน RuboCop ใน CI และทำให้ lint failure = CI failure

### RuboCop คืน exit code ที่ไม่ใช่ 0 เมื่อมี offense — นี่คือกลไกที่ทำให้ job fail

```yaml
- name: Lint code for consistent style
  run: bin/rubocop -f github
```

หัวใจสำคัญที่ทำให้ RuboCop "gate" CI ได้คือ **exit code ของโปรเซส** — ทุก step ใน GitHub Actions
job จะทำให้ job ทั้งหมด fail ทันทีถ้า exit code ของคำสั่งนั้นไม่ใช่ `0` (เป็นพฤติกรรมมาตรฐานของ
shell script ทุกตัว ไม่ใช่ฟีเจอร์พิเศษเฉพาะของ Actions) และ RuboCopถูกออกแบบมาให้คืนค่า `0` เมื่อ
ไม่พบ offense เลย และคืนค่าอื่น (ปกติ `1`) เมื่อพบ offense อย่างน้อยหนึ่งจุด

ทดสอบจริงโดยจงใจเขียนโค้ดผิด style:

```ruby
# app/models/bad_style.rb (ตัวอย่างที่จงใจเขียนผิดเพื่อทดสอบ)
class BadStyle
  def initialize( )
    @x=1
    if @x == 1 then
      puts "one"
    end
  end
end
```

```bash
bin/rubocop -f github app/models/bad_style.rb
echo "exit code: $?"
```

ผลลัพธ์จริง:

```
::error file=app/models/bad_style.rb,line=2,col=17::Style/DefWithParentheses: Omit the parentheses in defs when the method doesn't accept any arguments.
::error file=app/models/bad_style.rb,line=2,col=18::Layout/SpaceInsideParens: Space inside parentheses detected.
exit code: 1
```

หลังรัน `bin/rubocop -A app/models/bad_style.rb` เพื่อแก้ไขอัตโนมัติแล้วรันตรวจซ้ำ:

```bash
bin/rubocop -f github app/models/bad_style.rb
echo "exit code: $?"
```

```
exit code: 0
```

ยืนยันชัดเจนว่า exit code เปลี่ยนจาก `1` เป็น `0` พอดีตามที่แก้ offense ครบ — กลไกนี้เองที่ทำให้ CI
job แสดงเป็น ❌ สีแดงเมื่อมี style ผิด และ ✅ สีเขียวเมื่อผ่าน โดยไม่ต้องเขียน logic ตรวจสอบเพิ่มเติม
ใน workflow เลย

### ทำไมต้องใช้ `-f github` โดยเฉพาะ

Flag `-f github` (ย่อจาก `--format github`) สั่งให้ RuboCop พิมพ์ผลลัพธ์ในรูปแบบพิเศษที่ GitHub
Actions เข้าใจ เรียกว่า **workflow command** — บรรทัดที่ขึ้นต้นด้วย `::error file=...,line=...,col=...::message`
จะถูก GitHub แปลงเป็น **annotation** ที่แสดงตรงบรรทัดโค้ดที่มีปัญหาในหน้า "Files changed" ของ Pull
Request โดยอัตโนมัติ (เหมือนมี reviewer คอมเมนต์บรรทัดนั้นให้ทันที) ต่างจาก formatter ปกติที่แค่
พิมพ์ log แบนราบให้ไปไล่หาเอาเอง — ผลลัพธ์ตัวอย่างข้างบนที่ทดสอบจริงคือฟอร์แมตเดียวกันเป๊ะกับที่
Rails ตั้งไว้ในไฟล์ workflow

### ทำ RuboCop ให้ผ่อนปรนได้เมื่อจำเป็น (`.rubocop_todo.yml`)

สำหรับโปรเจกต์ที่มีโค้ดเก่าอยู่ก่อนแล้วและยังไม่พร้อมแก้ style ทั้งหมดในคราวเดียว สามารถสร้าง
"บันทึกหนี้" ไว้ก่อนด้วย:

```bash
bin/rubocop --auto-gen-config
```

คำสั่งนี้สร้างไฟล์ `.rubocop_todo.yml` ที่ปิด (`Enabled: false`) หรือผ่อนปรน threshold ของ cop
ที่ยังมี offense ค้างอยู่จำนวนมาก ทำให้ CI ผ่านได้ในระหว่างทยอยแก้ แต่ **ไม่ควรปล่อยไว้ถาวร** —
ควรตั้งเป้าลดจำนวนบรรทัดใน `.rubocop_todo.yml` ลงเรื่อย ๆ จนลบทิ้งได้ในที่สุด ไฟล์นี้ยังถูกรวมอยู่ใน
`DEPENDENCIES_HASH` ของ cache key ใน Step 745 ด้วย (`hashFiles(..., '**/.rubocop_todo.yml', ...)`)
เพื่อให้ RuboCop cache หมดอายุทันทีที่กฎเปลี่ยนแปลง

---

## Step 748: รัน Brakeman security scan อัตโนมัติ — Brakeman ตรวจอะไรบ้างกันแน่

### Brakeman คือ Static Application Security Testing (SAST) เฉพาะทางสำหรับ Rails

```yaml
- name: Scan for common Rails security vulnerabilities using static analysis
  run: bin/brakeman --no-pager
```

**Brakeman** เป็นเครื่องมือ**อ่านโค้ด Ruby/Rails โดยไม่ต้องรันแอปจริง** (static analysis) แล้ว
วิเคราะห์ **data flow** ว่าข้อมูลจากผู้ใช้ (เช่น `params`) ถูกส่งต่อไปยังจุดที่อันตราย (เช่น
SQL query, `render`, `system()`) โดยไม่ผ่านการ sanitize ที่เหมาะสมหรือไม่ รันจริงในแซนด์บ็อกซ์กับแอป
scaffold เปล่า ๆ (2 controllers, 2 models, 8 templates) ใช้เวลา **2.7 วินาที** และตรวจสอบครบ **79
checks** พร้อมกัน (แสดงรายชื่อ check ทั้งหมดในผลลัพธ์จริง เช่น `SQL`, `CrossSiteScripting`,
`MassAssignment`, `Execute`, `FileAccess`, `UnsafeReflection`, `Deserialize`)

### พิสูจน์ว่า Brakeman จับช่องโหว่ได้จริง

ทดสอบโดยจงใจเขียน SQL Injection ลงในโค้ดจริง:

```ruby
# app/controllers/posts_controller.rb (ช่องโหว่ที่จงใจใส่เพื่อทดสอบ)
class PostsController < ApplicationController
  def search
    @posts = Post.where("title LIKE '%#{params[:q]}%'")
    render json: @posts
  end
end
```

รัน `bin/brakeman --no-pager -q` ได้ผลลัพธ์จริง:

```
== Overview ==

Controllers: 2
Models: 2
Templates: 8
Errors: 0
Security Warnings: 1

== Warning Types ==

SQL Injection: 1

== Warnings ==

Confidence: High
Category: SQL Injection
Check: SQL
Message: Possible SQL injection
Code: Post.where("title LIKE '%#{params[:q]}%'")
File: app/controllers/posts_controller.rb
Line: 4
```

Brakeman จับได้ทันทีว่า `params[:q]` (ข้อมูลจากผู้ใช้ที่ควบคุมไม่ได้) ถูกแทรกเข้าไปใน SQL string
โดยตรงผ่าน string interpolation แทนที่จะใช้ placeholder — พร้อมระบุเลขบรรทัดและระดับความมั่นใจ
(`Confidence: High`) ให้แก้ไขตรงจุดได้ทันที หลังแก้เป็น parameterized query ตามหลักการที่สอนใน
Part 079 (OWASP Top 10):

```ruby
class PostsController < ApplicationController
  def search
    @posts = Post.where("title LIKE ?", "%#{params[:q]}%")
    render json: @posts
  end
end
```

รัน Brakeman ซ้ำแล้วพบ `Security Warnings: 0` ทันที ยืนยันว่ากลไก data-flow analysis ทำงานถูกต้อง
จริง ไม่ใช่แค่ pattern matching ผิวเผิน

### หมวดหมู่ที่ Brakeman ตรวจบ่อยที่สุดในแอป Rails ทั่วไป

- **SQL Injection** — ตามตัวอย่างข้างบน
- **Cross-Site Scripting (XSS)** — เช่นการใช้ `raw()`, `html_safe`, หรือ `<%= ... %>` ที่ปิดการ
  escape อัตโนมัติของ ERB กับข้อมูลที่มาจากผู้ใช้
- **Mass Assignment** — การอนุญาต attribute อันตรายผ่าน strong parameters แบบหลวมเกินไป (เช่น
  `params.require(:user).permit!` ที่อนุญาตทุก attribute รวมถึง `admin: true`)
- **Command Injection** — การส่งข้อมูลจากผู้ใช้เข้า `system()`, `` ` `` (backtick), `exec()` โดยตรง
- **Unsafe Deserialization** — การใช้ `Marshal.load`, `YAML.load` (ไม่ใช่ `YAML.safe_load`) กับข้อมูล
  ที่ไม่น่าเชื่อถือ
- **Insecure redirect / Open Redirect** — `redirect_to params[:url]` โดยไม่ตรวจสอบ whitelist ก่อน
- **Session/CSRF misconfiguration** — เช่นการปิด `protect_from_forgery` โดยไม่มีเหตุผลที่ดีพอ

ทั้งหมดนี้ Part 079–081 (Security phase) จะลงรายละเอียดเชิงลึกอีกครั้งว่าแก้แต่ละหมวดอย่างไร —
Part นี้เน้นที่การทำให้ **การตรวจจับเกิดขึ้นอัตโนมัติทุก PR** ไม่ใช่รอให้ใครนึกขึ้นได้เองว่าต้องรัน
Brakeman

### `--exit-on-warn` — ทำให้ Brakeman fail CI จริง ๆ เมื่อเจอปัญหา

ข้อสังเกตสำคัญ: คำสั่งเริ่มต้นของ Rails คือ `bin/brakeman --no-pager` **โดยไม่มี** flag
`--exit-on-warn` — แต่ Brakeman คืน exit code ที่ไม่ใช่ 0 เมื่อพบ warning อยู่แล้วเป็นพฤติกรรม
ค่าเริ่มต้น (ทดสอบจริง: `--exit-on-warn` เปลี่ยน exit code จาก `3` เมื่อมีช่องโหว่ เป็น `0` เมื่อไม่มี
— ยืนยันว่า flag นี้ทำให้ Brakeman ใช้ exit code สื่อสารผลลัพธ์แทนการพิมพ์รายงานเฉย ๆ) น่าสังเกตว่า
ไฟล์ `config/ci.rb` ที่ใช้กับ `bin/ci` (Step 742) เรียก Brakeman ด้วย
`--quiet --no-pager --exit-on-warn --exit-on-error` ครบทุก flag ในขณะที่ `ci.yml` ของ GitHub
Actions เรียกแบบสั้นกว่า — ทั้งสองทำงานได้ถูกต้องเหมือนกันในทางปฏิบัติเพราะ exit code ที่ไม่ใช่ 0
ก็ทำให้ job fail อยู่ดี แต่การใส่ flag ให้ครบชัดเจนกว่าช่วยให้อ่านโค้ดเข้าใจเจตนาง่ายขึ้นและป้องกัน
พฤติกรรมเปลี่ยนแปลงในอนาคตหาก Brakeman ปรับ default

### `bundler-audit` — สแกนหาช่องโหว่ที่ตัว gem เอง ไม่ใช่โค้ดของเรา

```yaml
- name: Scan for known security vulnerabilities in gems used
  run: bin/bundler-audit
```

ถ้า Brakeman ตรวจ**โค้ดที่เราเขียนเอง** `bundler-audit` ตรวจ**เวอร์ชันของ gem ที่ระบุใน
`Gemfile.lock`** เทียบกับฐานข้อมูล `ruby-advisory-db` ที่รวบรวม CVE ที่ประกาศต่อสาธารณะของ gem ทุกตัว
ในระบบนิเวศ Ruby รันจริงในแซนด์บ็อกซ์ได้ผลลัพธ์:

```
Download ruby-advisory-db ...
ruby-advisory-db:
  advisories:	1251 advisories
  last updated:	2026-09-27 11:40:47 -0400
No vulnerabilities found
```

ยืนยันว่า gem ทั้ง 123 ตัวที่ `bundle install` ไว้ในแอปทดสอบไม่มี CVE ที่รู้จักในฐานข้อมูล ณ ขณะนั้น
— ทั้งสองเครื่องมือ (Brakeman + bundler-audit) ทำงาน**เสริมกัน** ไม่ทับซ้อนกัน: Brakeman จับปัญหาที่
**เราเขียนขึ้นมาเอง**, bundler-audit จับปัญหาที่**มากับ dependency ที่เราไม่ได้เขียน** — จำเป็นต้องมี
ทั้งคู่พร้อมกันเพื่อความครอบคลุม

---

## Step 749: Branch Protection Rules — บังคับให้ CI ผ่านก่อน merge PR

### สิ่งสำคัญที่ต้องเข้าใจ: การตั้งค่านี้ไม่ได้อยู่ในไฟล์ YAML เลย

จนถึงตอนนี้ ทุกอย่างที่เรียนมาอยู่ในไฟล์ `.github/workflows/ci.yml` ทั้งหมด — แต่ workflow ที่รันแล้ว
รายงานผล ✅/❌ กลับมา **ไม่ได้บังคับให้ใครทำอะไรโดยอัตโนมัติ** ถ้าไม่มีการตั้งค่าเพิ่มเติม คนที่มีสิทธิ์
merge ยังสามารถกด "Merge" ได้ตามปกติแม้ CI จะแสดง ❌ สีแดงอยู่ก็ตาม — การบังคับจริง ๆ ต้องตั้งค่าที่
**GitHub repository settings** ซึ่งเป็นฟีเจอร์ของแพลตฟอร์ม ไม่ใช่ของไฟล์ workflow

> **หมายเหตุ:** ขั้นตอนต่อไปนี้อ้างอิงจากเอกสารทางการของ GitHub เกี่ยวกับหน้า UI ของ Settings
> (ซึ่งค่อนข้างเสถียรมาต่อเนื่องหลายปี) แซนด์บ็อกซ์ที่ใช้เขียน Part นี้ไม่มีการสร้าง repository
> ใหม่บน GitHub จริงเพื่อคลิกทดสอบหน้า UI นี้โดยตรง จึงระบุไว้ตรงนี้อย่างชัดเจนตามหลักการเดียวกับ
> ส่วนอื่นของหลักสูตรที่แยกสิ่งที่ทดสอบจริงออกจากสิ่งที่มาจากเอกสาร

### ขั้นตอนการตั้งค่า Branch Protection Rule

1. เข้าไปที่ repository บน GitHub → **Settings** → **Branches** (เมนูซ้าย ใต้หัวข้อ "Code and
   automation")
2. กด **Add branch protection rule** (หรือ "Add rule" ในเวอร์ชัน UI เก่ากว่า)
3. ตั้ง **Branch name pattern** เป็น `main` (หรือ pattern อื่นถ้าต้องการครอบคลุมหลาย branch เช่น
   `release/*`)
4. เปิดใช้งาน **Require a pull request before merging** — ห้าม push ตรงเข้า `main` โดยไม่ผ่าน PR
   เลย (ปิดช่องทางลัดที่ทำให้ CI ไม่ทันได้ตรวจสอบ)
5. เปิดใช้งาน **Require status checks to pass before merging** — นี่คือหัวใจของ Step นี้ เมื่อเปิด
   ตัวเลือกนี้ ช่องค้นหา "status checks" จะปรากฏขึ้น ให้พิมพ์ชื่อ job ที่ต้องการบังคับ (ชื่อที่ปรากฏ
   ในลิสต์นี้มาจาก **job ที่เคยรันมาแล้วอย่างน้อยหนึ่งครั้งบน repository** เท่านั้น — ถ้ายังไม่เคยรัน
   `ci.yml` เลยสักครั้ง ต้อง push โค้ดที่มีไฟล์ workflow ขึ้นไปก่อน ให้ Actions รันอย่างน้อยหนึ่งรอบ
   ก่อนที่ชื่อ job จะมาปรากฏในช่องค้นหานี้) เลือกให้ครบทุก job ที่ต้องการบังคับ เช่น `scan_ruby`,
   `scan_js`, `lint`, `test`, `system-test`
6. แนะนำให้เปิด **Require branches to be up to date before merging** ควบคู่ไปด้วย — บังคับให้
   branch ของ PR ต้อง merge/rebase เอา commit ล่าสุดของ `main` เข้ามาก่อน แล้วรัน CI ใหม่อีกครั้ง
   ก่อน merge ได้ เพื่อป้องกันสถานการณ์ที่ PR ผ่าน CI ตอนเปิดแรก ๆ แต่พอมี commit อื่นถูก merge
   เข้า `main` ไปก่อนแล้วเกิด conflict เชิง logic (ไม่ใช่ merge conflict แบบ syntax) ที่ CI เดิมไม่
   เคยตรวจพบ
7. ตัวเลือกเสริมที่นิยมเปิดร่วมด้วย: **Require approvals** (บังคับให้มี reviewer อนุมัติอย่างน้อย
   1 คนก่อน merge ได้), **Require conversation resolution before merging** (ห้าม merge ถ้ายังมี
   comment ใน PR ที่ยังไม่ resolve), และ **Do not allow bypassing the above settings** (ทำให้กฎนี้
   บังคับใช้แม้กับ repository admin เองด้วย ป้องกันการ "ลัดคิว" merge เองตอนรีบ)
8. กด **Create** เพื่อบันทึกกฎ

หลังตั้งค่าเสร็จ ปุ่ม "Merge pull request" บนหน้า PR จะถูก**ปิดใช้งาน (สีเทา กดไม่ได้)** โดยอัตโนมัติ
จนกว่า status check ทุกตัวที่เลือกไว้จะรายงานผลเป็น ✅ ครบทั้งหมด พร้อมข้อความอธิบายตรง ๆ บนหน้า PR
ว่า "Required statuses must pass before merging" — ทำให้กฎนี้เป็นสิ่งที่**บังคับใช้ได้จริงทางเทคนิค**
ไม่ใช่แค่ข้อตกลงทางสังคมของทีมที่พึ่งวินัยส่วนบุคคลอีกต่อไป

### เกร็ดเพิ่มเติม: Merge Queue

สำหรับ repository ที่มีทีมใหญ่และมี PR merge เข้า `main` พร้อมกันบ่อยมาก GitHub มีฟีเจอร์ **Merge
Queue** (ตั้งค่าในหน้าเดียวกับ Branch Protection) ที่จะ**จัดคิว** PR ที่พร้อม merge แล้วรัน CI ซ้ำ
อีกครั้งกับสถานะ "ถ้า merge เข้าไปจริงตอนนี้จะเป็นอย่างไร" (speculative merge กับ main ล่าสุด)
ก่อนจะ merge จริงทีละคิว ป้องกันสถานการณ์ที่ PR สองอันต่างก็ผ่าน CI แยกกัน แต่พอ merge รวมกันแล้ว
กลับพังเพราะตัดกันเชิง logic — เป็นฟีเจอร์ขั้นสูงที่ทีมขนาดใหญ่ควรรู้จักไว้ แต่ทีมขนาดเล็กถึงกลาง
Branch Protection Rule ธรรมดาตามที่อธิบายข้างต้นก็เพียงพอแล้ว

---

## Step 750: CD เบื้องต้น (deploy อัตโนมัติเมื่อ merge เข้า main) และ Matrix Build

### แนวคิด Continuous Deployment ต่อยอดจาก CI

เมื่อทุก job (`scan_ruby`, `scan_js`, `lint`, `test`, `system-test`) ผ่านหมดแล้วบน `main` branch
ขั้นตอนถัดไปตามธรรมชาติคือ **deploy อัตโนมัติ** แทนที่จะให้คนต้อง deploy มือทุกครั้ง เพิ่ม job ใหม่
ที่ **รอ** ให้ job อื่นผ่านก่อนด้วย `needs:` และ**รันเฉพาะ**เมื่อเป็นการ push เข้า `main` (ไม่ใช่แค่
เปิด PR) ด้วยเงื่อนไข `if:`:

```yaml
deploy:
  needs: [ scan_ruby, scan_js, lint, test, system-test ]
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  runs-on: ubuntu-latest
  environment: production
  steps:
    - name: Checkout code
      uses: actions/checkout@v6

    - name: Deploy (รายละเอียดเต็มใน Part 076)
      run: echo "จะรัน kamal deploy ที่นี่ หลังผ่านทุก job ด้านบน"
```

อ่านทีละส่วน:

- `needs: [ scan_ruby, scan_js, lint, test, system-test ]` — บังคับให้ job `deploy` **รอ** ทั้ง 5
  job ก่อนหน้าให้จบและผ่านทั้งหมดก่อน ถ้ามี job ใดล้มเหลว job `deploy` จะไม่ถูกรันเลย (default
  behavior ของ `needs:` — ไม่ต้องเขียนเงื่อนไขเพิ่มเติมสำหรับกรณีนี้)
- `if: github.ref == 'refs/heads/main' && github.event_name == 'push'` — ป้องกันไม่ให้ job นี้รัน
  ตอนแค่เปิด/อัปเดต Pull Request (`pull_request` event) ซึ่งอยากได้แค่ผลตรวจสอบ ไม่อยาก deploy
  โค้ดที่ยังไม่ merge จริง — เงื่อนไขนี้ทำให้ deploy เกิดขึ้นเฉพาะตอน push (มักจะเป็นตอน merge PR
  สำเร็จ) เข้า `main` เท่านั้น
- `environment: production` — ผูก job นี้กับ **GitHub Environment** ชื่อ `production` (ตั้งค่าที่
  Settings → Environments) ซึ่งสามารถกำหนด **required reviewers** เพิ่มอีกชั้นได้ (ต้องมีคนกด
  approve ก่อน job จะรันต่อ แม้ `needs:` จะผ่านหมดแล้วก็ตาม) และเก็บ secret เฉพาะของ environment
  นั้น (เช่น production deploy key ที่แยกจาก secret ของ staging) — เป็นเซฟตี้เน็ตอีกชั้นสำหรับ
  การ deploy ที่กระทบผู้ใช้จริง

Step นี้เป็นเพียงภาพรวมแนวคิด — **กลไกการ deploy จริง** (SSH เข้าเซิร์ฟเวอร์, build/push Docker
image, restart service แบบไม่มี downtime) จะสอนอย่างละเอียดใน **Part 076 ด้วย Kamal** ซึ่งเป็น
เครื่องมือ deploy ที่ Rails 8 แนะนำเป็นค่าเริ่มต้น (37signals/Basecamp เป็นผู้พัฒนา) สิ่งที่ควรจำจาก
Step นี้คือ**รูปแบบการเชื่อม CI เข้ากับ CD**: ใช้ `needs:` gate ด้วย job ทดสอบทั้งหมด และใช้ `if:`
จำกัดให้ deploy เกิดเฉพาะตอน merge เข้า main จริงเท่านั้น

### Matrix Build — ทดสอบข้ามหลายเวอร์ชัน Ruby พร้อมกัน

สำหรับ**แอป** Rails ทั่วไปที่ deploy ด้วย Ruby เวอร์ชันเดียวตายตัว (ตามที่ระบุไว้ใน `.ruby-version`
— ทบทวนจาก Part 074) การทดสอบข้ามหลายเวอร์ชัน Ruby ไม่ค่อยจำเป็นเพราะ production รันเวอร์ชันเดียว
เท่านั้น แต่สำหรับ **gem/library ที่คนอื่นเอาไปใช้** (ทบทวนจาก Part 089 เรื่องการสร้าง gem) จำเป็น
ต้องรู้ว่าโค้ดทำงานถูกต้องบน Ruby หลายเวอร์ชันที่ผู้ใช้ปลายทางอาจใช้อยู่ — **matrix build** คือ
ฟีเจอร์ของ GitHub Actions ที่รัน job เดียวกันซ้ำหลายครั้งด้วยค่าพารามิเตอร์ต่างกัน โดยอัตโนมัติสร้าง
runner แยกสำหรับแต่ละชุดค่า:

```yaml
test:
  runs-on: ubuntu-latest
  strategy:
    fail-fast: false
    matrix:
      ruby-version: [ "3.2", "3.3", "3.4" ]
  steps:
    - name: Checkout code
      uses: actions/checkout@v6

    - name: Set up Ruby ${{ matrix.ruby-version }}
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: ${{ matrix.ruby-version }}
        bundler-cache: true

    - name: Run tests
      run: bundle exec rspec
```

`strategy.matrix.ruby-version` ที่ระบุ 3 ค่า ทำให้ GitHub Actions สร้าง **3 runner แยกกัน** รันคู่ขนาน
โดยแต่ละ runner ได้รับค่า `matrix.ruby-version` ของตัวเองไปแทนที่ `${{ matrix.ruby-version }}` ทั้ง
ในชื่อ step และใน `with: ruby-version:` ผลคือได้ผลตรวจสอบแยกกันชัดเจนว่าโค้ดผ่านหรือไม่ผ่านบน Ruby
เวอร์ชันไหนบ้าง (เช่น "✅ test (3.2)", "✅ test (3.3)", "❌ test (3.4)" ถ้ามีเวอร์ชันใดพัง)

`fail-fast: false` เป็นตัวเลือกสำคัญ — ค่า default ของ matrix คือ `fail-fast: true` ซึ่งจะ**ยกเลิก**
runner อื่นที่ยังรันอยู่ทันทีที่มี combination ใดหนึ่งล้มเหลว (ประหยัด CI minutes แต่ทำให้ไม่เห็นภาพ
รวมว่าพังกี่เวอร์ชัน) ส่วน `fail-fast: false` ปล่อยให้ทุก combination รันจนจบเพื่อเห็นภาพรวมทั้งหมด
ว่าปัญหาเกิดเฉพาะเวอร์ชันไหนบ้าง — มีประโยชน์มากตอนดีบักปัญหาที่เกี่ยวกับความเข้ากันได้ของเวอร์ชัน

ทดสอบแนวคิดนี้จริงในแซนด์บ็อกซ์ (ซึ่งมี Ruby 3.1.6, 3.2.6, 3.3.6 ติดตั้งอยู่ผ่าน `rbenv`) ด้วยการรัน
ชุดทดสอบเดียวกันข้ามทั้งสามเวอร์ชันโดยตรง (จำลองพฤติกรรมของ matrix build โดยไม่ผ่าน Actions จริง):

```bash
for v in 3.1.6 3.2.6 3.3.6; do
  echo "=== Ruby $v ==="
  /opt/rbenv/versions/$v/bin/ruby test/greeter_test.rb
done
```

```
=== Ruby 3.1.6 ===
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
=== Ruby 3.2.6 ===
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
=== Ruby 3.3.6 ===
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

ผ่านทั้งสามเวอร์ชันจริง — นี่คือสิ่งที่ matrix build บน GitHub Actions ทำให้อัตโนมัติโดยไม่ต้องสลับ
เวอร์ชัน Ruby รันคำสั่งซ้ำ ๆ เองแบบนี้ **matrix ยังขยายได้มากกว่าหนึ่งมิติ** เช่น ทดสอบทั้งข้าม Ruby
version และ Rails version พร้อมกัน (`matrix.ruby-version` × `matrix.rails-version`) ซึ่ง GitHub
Actions จะสร้าง combination ทุกคู่ให้อัตโนมัติ — ใช้ `exclude:`/`include:` เพื่อตัด combination ที่
ไม่รองรับออก (เช่น Rails รุ่นเก่าที่ไม่รองรับ Ruby รุ่นใหม่ล่าสุด) เป็นเทคนิคขั้นสูงที่จะฝึกในแบบฝึกหัด
เพิ่มเติมท้าย Part นี้

---

## แบบฝึกหัด: ปรับแต่ง Workflow เพิ่ม Postgres Service, Gem Caching, และ Coverage Report

### โจทย์

Workflow เริ่มต้นของ Rails 8 (Step 742) มี Postgres service และ gem caching ให้อยู่แล้วเป็นค่า
เริ่มต้น (ผ่าน `services: postgres:` และ `bundler-cache: true`) แต่ยังขาดสิ่งที่จำเป็นสำหรับทีมที่
ใช้ RSpec + SimpleCov ตามที่สอนใน Phase 6 ให้ปรับแต่ง job `scan_ruby` และ `test` ของไฟล์
`.github/workflows/ci.yml` ดังนี้:

1. เปลี่ยน job `test` ให้รันด้วย RSpec แทน Minitest (ตาม Step 746) พร้อมระบุ `POSTGRES_DB` ให้
   ตรงกับชื่อฐานข้อมูล test ของแอปอย่างชัดเจน (ไม่พึ่งฐานข้อมูล `postgres` เริ่มต้นเฉย ๆ)
2. เพิ่มการ cache โฟลเดอร์ `ruby-advisory-db` ที่ `bundler-audit` ใช้ ในไฟล์ job `scan_ruby` (ตาม
   แนวคิด Step 745 แต่ประยุกต์กับสิ่งที่ยังไม่ถูก cache ในไฟล์ต้นฉบับ)
3. เพิ่ม step วัด **test coverage** ด้วย SimpleCov ใน job `test` แล้วเขียนสรุปเปอร์เซ็นต์ลงใน
   GitHub Step Summary (`$GITHUB_STEP_SUMMARY`) พร้อม upload โฟลเดอร์ `coverage/` เป็น artifact
   เพื่อดาวน์โหลดดูรายงานแบบเต็มได้

### เฉลย

**1) เพิ่ม gem ที่จำเป็นใน `Gemfile`:**

```ruby
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end

group :test do
  gem "simplecov", require: false
end
```

**2) เปิดใช้งาน SimpleCov ใน `spec/rails_helper.rb` (ต้องอยู่บรรทัดบนสุดก่อน require อื่นทั้งหมด):**

```ruby
# spec/rails_helper.rb
require "simplecov"
SimpleCov.start "rails"

require "spec_helper"
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
# ... ส่วนที่เหลือที่ generator สร้างให้ ...
```

รันจริงในแซนด์บ็อกซ์ยืนยันผลลัพธ์:

```
Coverage report generated for RSpec to coverage/index.html
Line coverage: 37 / 45 (82.22%)
```

พร้อมไฟล์ `coverage/.last_run.json`:

```json
{
  "result": {
    "line": 82.22
  }
}
```

**3) ไฟล์ `.github/workflows/ci.yml` ฉบับสมบูรณ์หลังปรับแต่ง (ตรวจสอบ YAML syntax ผ่านจริงด้วย
`YAML.safe_load` ก่อนนำไปใช้):**

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [ main ]

jobs:
  scan_ruby:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Cache ruby-advisory-db (ฐานข้อมูลช่องโหว่ของ bundler-audit)
        uses: actions/cache@v4
        with:
          path: ~/.local/share/ruby-advisory-db
          key: ruby-advisory-db-${{ github.run_id }}
          restore-keys: |
            ruby-advisory-db-

      - name: Scan for common Rails security vulnerabilities using static analysis
        run: bin/brakeman --no-pager --exit-on-warn

      - name: Scan for known security vulnerabilities in gems used
        run: bin/bundler-audit --update

  scan_js:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Scan for security vulnerabilities in JavaScript dependencies
        run: bin/importmap audit

  lint:
    runs-on: ubuntu-latest
    env:
      RUBOCOP_CACHE_ROOT: tmp/rubocop
    steps:
      - name: Checkout code
        uses: actions/checkout@v6
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - name: Prepare RuboCop cache
        uses: actions/cache@v4
        env:
          DEPENDENCIES_HASH: ${{ hashFiles('.ruby-version', '**/.rubocop.yml', '**/.rubocop_todo.yml', 'Gemfile.lock') }}
        with:
          path: ${{ env.RUBOCOP_CACHE_ROOT }}
          key: rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-${{ github.ref_name == github.event.repository.default_branch && github.run_id || 'default' }}
          restore-keys: |
            rubocop-${{ runner.os }}-${{ env.DEPENDENCIES_HASH }}-
      - name: Lint code for consistent style
        run: bin/rubocop -f github

  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: ci_demo_app_test
        ports:
          - 5432:5432
        options: --health-cmd="pg_isready" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - name: Install packages
        run: sudo apt-get update && sudo apt-get install --no-install-recommends -y libpq-dev libvips

      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Run tests with coverage
        env:
          RAILS_ENV: test
          DATABASE_URL: postgres://postgres:postgres@localhost:5432
        run: |
          bin/rails db:test:prepare
          bundle exec rspec

      - name: Write coverage summary
        if: always()
        run: |
          if [ -f coverage/.last_run.json ]; then
            LINE_COVERAGE=$(ruby -rjson -e 'puts JSON.parse(File.read("coverage/.last_run.json"))["result"]["line"]')
            echo "### 📊 Test coverage" >> "$GITHUB_STEP_SUMMARY"
            echo "Line coverage: **${LINE_COVERAGE}%**" >> "$GITHUB_STEP_SUMMARY"
          fi

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          if-no-files-found: ignore
```

**คำอธิบายส่วนที่เพิ่มเข้ามา:**

- `POSTGRES_DB: ci_demo_app_test` ใน service ทำให้ PostgreSQL สร้างฐานข้อมูลชื่อนี้ให้ตั้งแต่
  container เริ่มทำงาน ตรงกับชื่อฐานข้อมูล test ที่ `config/database.yml` คาดหวัง (รูปแบบ
  `<app_name>_test`) ทำให้ `bin/rails db:test:prepare` ทำงานเร็วขึ้นเล็กน้อยเพราะไม่ต้องสร้าง
  ฐานข้อมูลใหม่เอง เพียงแค่ migrate schema เข้าไป
- Step "Cache ruby-advisory-db" cache โฟลเดอร์ `~/.local/share/ruby-advisory-db` (ที่ยืนยันแล้วว่า
  มีขนาดจริง 12MB) ด้วย key ที่ผูกกับ `github.run_id` เสมอ (การันตี cache ใหม่ทุกรัน เพื่อให้ฐาน
  ข้อมูลช่องโหว่อัปเดตล่าสุดเสมอ) แต่ใช้ `restore-keys:` แบบ prefix ให้ fallback ไปใช้ cache ของ
  รันก่อนหน้าได้ถ้า cache ของรันปัจจุบันยังไม่เคยถูกสร้าง — แลกความสดใหม่ของข้อมูลกับความเร็ว
  (`bundler-audit --update` ยังคงอัปเดตส่วนที่ขาดหายไปจาก cache เก่าอยู่ดี ไม่ใช่ข้อมูลเก่าค้างตลอด
  ไป)
- Step "Write coverage summary" ใช้ `if: always()` เพื่อให้ทำงานแม้ step ทดสอบก่อนหน้าจะ fail
  (อยากเห็น coverage แม้ test บางตัวพัง) อ่านค่าจาก `coverage/.last_run.json` ที่ SimpleCov สร้าง
  ให้ (ทดสอบจริงแล้วว่ามีโครงสร้าง `{"result": {"line": <number>}}`) แล้วเขียนลง
  `$GITHUB_STEP_SUMMARY` ซึ่งเป็นตัวแปรพิเศษที่ GitHub เตรียมไว้ — ข้อความ Markdown ที่ echo เข้าไป
  จะไปแสดงผลในหน้าสรุปผลของ workflow run โดยตรง (แท็บ "Summary" ด้านบนหน้า) เห็นได้ทันทีโดยไม่ต้อง
  ดาวน์โหลด artifact
- Step "Upload coverage report" ใช้ `actions/upload-artifact@v4` (Action เดียวกับที่ Rails ใช้เก็บ
  screenshot ของ system test ที่ล้มเหลว) อัปโหลดทั้งโฟลเดอร์ `coverage/` (รวม `index.html` ที่ดู
  รายละเอียดรายไฟล์ได้) ให้ดาวน์โหลดจากหน้า workflow run ได้ — `if-no-files-found: ignore` ป้องกัน
  ไม่ให้ job fail เฉย ๆ ถ้าด้วยเหตุผลใดก็ตามไม่มีโฟลเดอร์ `coverage/` เกิดขึ้น

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม step แจ้งเตือนผ่าน Slack/Discord webhook เมื่อ workflow ล้มเหลวบน `main` branch เท่านั้น
   (ใช้เงื่อนไข `if: failure() && github.ref == 'refs/heads/main'` ในตอนท้ายของ job ที่เกี่ยวข้อง
   หรือสร้าง job แยกที่ `needs:` ทุก job อื่นแล้วเช็คผลรวมด้วย `if: failure()`) โดยเก็บ webhook URL
   ไว้เป็น GitHub secret ไม่ hardcode ลงในไฟล์
2. ขยาย matrix build ของ job `test` ให้ทดสอบทั้งข้าม Ruby version (`["3.2", "3.3"]`) และ Rails
   version (`["7.2", "8.1"]`) พร้อมกันแบบสองมิติ โดยใช้ `matrix.include`/`matrix.exclude` เพื่อตัด
   combination ที่ไม่รองรับออก (เช่น Rails 8.1 ที่ต้องการ Ruby ขั้นต่ำสูงกว่า 3.2) — ทดสอบว่า
   `Gemfile` ต้องปรับให้รับตัวแปร `RAILS_VERSION` จาก environment เพื่อเลือกเวอร์ชัน Rails ที่จะ
   ติดตั้งแบบไดนามิกได้อย่างไร (ใบ้: `gem "rails", ENV.fetch("RAILS_VERSION", "~> 8.1")`)
3. เพิ่ม job ใหม่ที่รันเฉพาะเมื่อไฟล์ที่เกี่ยวกับ JavaScript/import map เปลี่ยนแปลงเท่านั้น (ใช้
   Action `dorny/paths-filter` หรือ `on.pull_request.paths:` เพื่อกรอง path) เพื่อประหยัดเวลา CI
   โดยไม่ต้องรัน `scan_js` ทุกครั้งที่มีแค่การแก้ไข Ruby code ที่ไม่เกี่ยวข้องกับ JavaScript เลย

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **CI/CD** แก้ปัญหาอะไร — เปลี่ยนคำถาม "การเปลี่ยนแปลงนี้ทำให้อะไรพังไหม" จากสิ่งที่ต้อง
  พึ่งความจำ/วินัยของแต่ละคน ให้กลายเป็นกระบวนการอัตโนมัติที่รันทุกครั้งโดยไม่มีข้อยกเว้น
- สำรวจไฟล์ `.github/workflows/ci.yml` และ `.github/dependabot.yml` ที่ **Rails 8 สร้างให้
  อัตโนมัติทันทีที่รัน `rails new`** โดยไม่ต้องตั้งค่าอะไรเพิ่มเติม พร้อมเดินอ่านครบทั้ง 5 jobs
  (`scan_ruby`, `scan_js`, `lint`, `test`, `system-test`) และรู้จัก local CI runner ในตัวอย่าง
  `bin/ci`/`config/ci.rb` ที่มาใหม่ใน Rails 8.1
- เข้าใจกายวิภาคของ GitHub Actions ครบทุกระดับ: **workflow → job → step**, **runner**
  (GitHub-hosted vs self-hosted), **trigger** (`on: push`/`pull_request`/`schedule`/
  `workflow_dispatch`), ความต่างของ `uses:` กับ `run:`, และกับดัก YAML ที่พบบ่อย (รวมถึงกรณีคีย์
  `on:` ที่ parser ทั่วไปตีความเป็น boolean ตามสเปก YAML 1.1)
- ตั้งค่า **Service Containers** (`services:` block) ให้ job ทดสอบมี PostgreSQL/Redis จริงใช้งาน
  ผ่าน `localhost` โดยไม่ต้องติดตั้งเองบน runner พร้อม healthcheck ป้องกัน race condition
- เข้าใจกลไก **caching** สองชั้น: `bundler-cache: true` ของ `ruby/setup-ruby` (จัดการ gem cache
  อัตโนมัติทั้งหมด) กับ `actions/cache` แบบ manual (ใช้กับสิ่งที่ไม่ใช่ gem เช่น RuboCop's own
  analysis cache หรือ `ruby-advisory-db`) พร้อมตัวเลขจริงที่วัดได้ (42.5 วินาทีแบบไม่มี cache เทียบ
  กับ 0.58 วินาทีแบบมี cache)
- สลับชุดทดสอบใน CI จาก Minitest (ค่าเริ่มต้นของ Rails) มาเป็น **RSpec + FactoryBot** ตามที่สอนใน
  Phase 6 ได้จริง โดยแก้แค่คำสั่งในหนึ่ง step โดยไม่ต้องแตะโครงสร้าง workflow ส่วนอื่น
- ทำให้ **RuboCop** และ **Brakeman** เป็น gate ที่บังคับใช้จริงได้ผ่านกลไก exit code — พิสูจน์ด้วย
  การจงใจเขียนโค้ดผิด style และแทรกช่องโหว่ SQL Injection แล้วดู exit code เปลี่ยนจริงหลังแก้ไข
  พร้อมเข้าใจว่า Brakeman ตรวจอะไรบ้าง (SQL Injection, XSS, Mass Assignment, Command Injection,
  Unsafe Deserialization ฯลฯ) และ `bundler-audit` ตรวจ CVE ของ gem dependency แยกต่างหาก
- เข้าใจว่า **Branch Protection Rules** เป็นการตั้งค่าที่ฝั่ง GitHub repository settings ไม่ใช่ใน
  ไฟล์ workflow — และรู้ขั้นตอนตั้งค่า "Require status checks to pass before merging" ให้บังคับ
  merge ได้จริงทางเทคนิค ไม่ใช่แค่ข้อตกลงทางสังคม
- เห็นภาพรวมของการเชื่อม **CD** เข้ากับ CI ด้วย `needs:` + `if:` (deploy เฉพาะเมื่อ push เข้า main
  และผ่านทุก job ทดสอบแล้วเท่านั้น) และรู้จัก **Matrix Build** สำหรับทดสอบข้ามหลายเวอร์ชัน Ruby
  พร้อมกัน ซึ่งสำคัญมากสำหรับผู้ดูแล gem/library แต่เป็นทางเลือกสำหรับแอปทั่วไปที่ pin เวอร์ชันเดียว
- ลงมือปรับแต่ง workflow จริง เพิ่ม explicit `POSTGRES_DB`, cache สำหรับ `ruby-advisory-db`, และ
  step รายงาน coverage ผ่าน `$GITHUB_STEP_SUMMARY` พร้อม upload เป็น artifact — ทุกคำสั่งภายใน
  ผ่านการรันจริงยืนยันแล้วในเครื่อง

**ต่อไป (Part 076):** ตอนนี้เรามี pipeline ที่ทดสอบ, ตรวจ style, และสแกนความปลอดภัยให้อัตโนมัติทุก
PR พร้อม gate การ merge ด้วย Branch Protection Rule แล้ว — สิ่งที่ยังเป็นแค่ placeholder
(`echo "จะรัน kamal deploy ที่นี่"`) ใน job `deploy` ของ Step 750 จะกลายเป็นของจริงใน Part 076
ที่จะสอน **Kamal** เครื่องมือ deploy ที่ Rails 8 แนะนำเป็นค่าเริ่มต้น (พัฒนาโดยทีม 37signals/
Basecamp ผู้สร้าง Rails เอง) ครอบคลุมตั้งแต่การตั้งค่า `config/deploy.yml`, การ deploy ผ่าน SSH
ไปยังเซิร์ฟเวอร์ที่รัน Docker (เชื่อมกับความรู้เรื่อง Docker จาก Part 073), ไปจนถึงการทำ zero-downtime
deployment จริงที่ผูกเข้ากับ job `deploy` ที่วางโครงไว้แล้วใน Part นี้พอดี
