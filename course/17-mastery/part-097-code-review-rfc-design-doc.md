# Part 097: Code Review ระดับมืออาชีพ, การเขียน RFC/Design Doc

> **Step ครอบคลุมใน Part นี้:** Step 961–970

**ระดับ:** Senior/Mastery | **Rails version:** 7.x / 8.x | **Ruby version:** 3.2+

ในฐานะ Senior Rails Engineer สิ่งที่แยกระดับ Senior ออกจาก Mid-level ไม่ใช่แค่ความสามารถเขียนโค้ด แต่คือความสามารถในการ **สื่อสารทางเทคนิค** ผ่าน Code Review, RFC, Design Doc และเอกสารสถาปัตยกรรม Part นี้จะพาคุณเรียนรู้กระบวนการที่ทีมวิศวกรระดับโลกใช้เพื่อให้แน่ใจว่าทุกคนในทีมเดินไปในทิศทางเดียวกัน ลดความขัดแย้ง และสร้างโค้ดที่มีคุณภาพสูงอย่างต่อเนื่อง

---

## สารบัญ

- [Step 961: บทบาทของ Code Review](#step-961)
- [Step 962: PR ที่ดี](#step-962)
- [Step 963: Review comment ที่ constructive](#step-963)
- [Step 964: เทคนิค review ที่มีประสิทธิภาพ](#step-964)
- [Step 965: RFC คืออะไร](#step-965)
- [Step 966: โครงสร้าง RFC ที่ดี](#step-966)
- [Step 967: Design Doc สำหรับ feature ขนาดกลาง-ใหญ่](#step-967)
- [Step 968: ADR (Architecture Decision Record)](#step-968)
- [Step 969: การ facilitate technical discussion](#step-969)
- [Step 970: เอกสาร CONTRIBUTING.md และ ARCHITECTURE.md](#step-970)
- [แบบฝึกหัด](#exercises)
- [สรุปสิ่งที่ได้เรียนรู้](#summary)

---

## Step 961: บทบาทของ Code Review — ไม่ใช่แค่หาบัก แต่คือ knowledge sharing และ quality gate {#step-961}

หลายคนมองว่า Code Review คือการ "ตรวจงาน" เพื่อหาข้อผิดพลาด แต่ในความเป็นจริง Code Review ที่ดีทำหน้าที่หลายอย่างพร้อมกัน

### ประโยชน์ที่แท้จริงของ Code Review

**1. Knowledge Sharing — กระจายความรู้ในทีม**

เมื่อคุณ review โค้ดของเพื่อนร่วมทีม คุณกำลังเรียนรู้ว่าเขาแก้ปัญหาอย่างไร ในทางกลับกัน เมื่อโค้ดของคุณถูก review คุณได้รับมุมมองใหม่ที่คุณอาจไม่เคยคิดถึง ผลลัพธ์คือทีมมีความรู้กระจายไปทั่ว ไม่กระจุกอยู่ที่คนใดคนหนึ่ง

```
ทีมที่ไม่ทำ Code Review:
- Person A รู้เรื่อง Payment เท่านั้น
- Person B รู้เรื่อง Notifications เท่านั้น
- Person C รู้เรื่อง Reports เท่านั้น
→ ถ้า Person A ลาออก ไม่มีใครรู้เรื่อง Payment!

ทีมที่ทำ Code Review อย่างสม่ำเสมอ:
- ทุกคนรู้ระบบในภาพรวม
- ทุกคนสามารถแก้ไขบัก critical ได้
- ความเสี่ยงจากการ "bus factor" ลดลง
```

**2. Quality Gate — กรองคุณภาพก่อนเข้า main branch**

Code Review ทำหน้าที่เป็น checkpoint สุดท้ายก่อนที่โค้ดจะเข้า production ช่วยจับ:
- Logic errors ที่ automated tests พลาด
- Security vulnerabilities ที่ไม่มี scanner ตรวจ
- Performance issues จาก N+1 query หรือการใช้ memory ไม่ถูกต้อง
- Inconsistency กับ coding standards ของทีม

**3. Mentoring — พัฒนา Junior Engineer**

สำหรับวิศวกรระดับ Senior การ review โค้ดของ Junior เป็นโอกาสสอนงานที่มีประสิทธิภาพสูงมาก เพราะเป็นการสอนบนโค้ดจริงที่เขาเขียน ไม่ใช่แค่ทฤษฎี

**4. Collective Code Ownership**

เมื่อโค้ดผ่านการ review แล้ว "ความเป็นเจ้าของ" โค้ดนั้นกระจายออกไปทั้งทีม ไม่มีใครบอกว่า "นี่ไม่ใช่โค้ดฉัน" เมื่อเกิดปัญหา

### สิ่งที่ Code Review ไม่ควรเป็น

```
❌ การตำหนิ หรือแสดงให้เห็นว่าตัวเองเก่งกว่า
❌ Rubber stamp — approve ทุกอย่างโดยไม่อ่าน
❌ Gatekeeping — ขัดขวาง PR ด้วยเหตุผลส่วนตัว
❌ Style war — เถียงเรื่อง formatting ที่ควรใช้ linter แทน
```

### Metrics ของ Code Review ที่ดีต่อสุขภาพทีม

- **Time to first review:** ควรน้อยกว่า 24 ชั่วโมง
- **PR size:** ควรน้อยกว่า 400 lines changed (ยิ่งเล็กยิ่งดี)
- **Cycle time:** เวลาตั้งแต่ PR open จนถึง merge
- **Review comments per PR:** ควรสม่ำเสมอ ถ้าไม่มี comment เลยอาจ rubber stamp

---

## Step 962: PR ที่ดี — scope เล็ก, description ชัด, screenshot สำหรับ UI change, self-review ก่อน {#step-962}

ก่อนจะเรียกร้องให้คนอื่น review ได้ดี คุณต้องสร้าง PR ที่ "reviewable" ก่อน

### หลักการ: PR ที่ดีต้องทำให้ reviewer อ่านง่ายที่สุด

**1. Scope เล็ก และ Focused**

PR หนึ่งอันควรทำสิ่งหนึ่งสิ่ง ไม่ใช่หลายสิ่ง

```
❌ PR ที่ไม่ดี: "Add payment feature + refactor user model + fix 3 bugs"
✅ PR ที่ดี: "Add Stripe webhook handler for payment_intent.succeeded"
```

ทำไม? เพราะ reviewer ต้องคิด context ใหม่ทุกครั้งที่เปลี่ยนหัวข้อ PR ที่ใหญ่เกินไปทำให้ reviewer เหนื่อย และมีแนวโน้มที่จะ approve โดยไม่ตรวจจริงๆ

**2. Description ที่สมบูรณ์**

ใช้ template นี้เป็นแนวทาง:

```markdown
## สรุปการเปลี่ยนแปลง
เพิ่ม Stripe webhook handler สำหรับ event `payment_intent.succeeded`
เมื่อ payment สำเร็จ ระบบจะ:
1. อัปเดตสถานะ Order เป็น `paid`
2. ส่ง confirmation email ให้ลูกค้า
3. Trigger fulfillment process

## ทำไมต้องเปลี่ยน
ปัจจุบัน Order ถูกอัปเดตผ่าน redirect URL ซึ่งไม่ reliable
เพราะผู้ใช้อาจปิด browser ก่อน redirect กลับมา

## วิธี Test
1. ใช้ Stripe CLI: `stripe listen --forward-to localhost:3000/webhooks/stripe`
2. ทดสอบ: `stripe trigger payment_intent.succeeded`
3. ตรวจสอบ log และ Order status ใน database

## Checklist
- [x] Unit tests เพิ่มแล้ว
- [x] Integration test กับ Stripe CLI ผ่าน
- [x] Idempotency key ป้องกัน duplicate processing
- [x] Error handling สำหรับกรณี webhook signature invalid

## Related
Fixes #234
Depends on #228 (Stripe gem upgrade)
```

**3. Screenshot สำหรับ UI Changes**

ถ้า PR มีการเปลี่ยน UI ต้องมี screenshot หรือ GIF เสมอ

```markdown
## Before
![before](https://...screenshot-before.png)

## After  
![after](https://...screenshot-after.png)
```

วิธีง่ายๆ สำหรับ macOS: `Cmd + Shift + 4` แล้วแนบรูปใน PR description โดยตรง

**4. Self-review ก่อน request review**

ก่อน assign reviewer ให้ทบทวน diff ของตัวเองก่อน

```bash
# ดู diff ก่อน push
git diff main...HEAD

# หรือบน GitHub: กด "Files changed" ใน PR ของตัวเอง
```

ถามตัวเองว่า:
- มี debug code ที่ลืมเอาออกไหม? (`puts`, `binding.pry`, `console.log`)
- มี TODO comment ที่ควรแก้ในงานนี้ไหม?
- Test ครอบคลุม happy path และ edge case หรือยัง?
- โค้ดอ่านเข้าใจง่ายไหม? ถ้าตัวเองอ่านยัง ต้องเพิ่ม comment

**5. Draft PR สำหรับงาน WIP**

ถ้ายังทำไม่เสร็จแต่อยากให้ทีมเห็นทิศทาง ให้เปิด PR เป็น Draft:

```
GitHub: เลือก "Create draft pull request" แทน "Create pull request"
```

Draft PR จะไม่ถูก request review จนกว่าคุณจะ mark as "Ready for review"

---

## Step 963: Review comment ที่ constructive — nit vs blocker, ถามก่อน assume, acknowledge good code {#step-963}

วิธีที่คุณ comment บน PR ส่งผลต่อวัฒนธรรมทีมโดยตรง

### ระดับ severity ของ comment

ใช้ prefix เพื่อบอก severity อย่างชัดเจน:

```
🔴 [BLOCKER] ต้องแก้ก่อน merge — มี security vulnerability หรือ logic error ชัดเจน
🟡 [SUGGESTION] แนะนำให้แก้ — มีวิธีที่ดีกว่า แต่ไม่ critical
🔵 [NIT] เรื่องเล็กน้อย — style, naming ที่ไม่ตรงกับ convention
💬 [QUESTION] ถามเพื่อทำความเข้าใจ — ไม่ใช่ข้อผิดพลาด
✨ [PRAISE] ชมโค้ดที่ดี — สำคัญมาก!
```

ตัวอย่างการใช้:

```ruby
# โค้ดที่ถูก review
def calculate_discount(user, order)
  if user.premium?
    order.total * 0.2
  else
    order.total * 0.1
  end
end
```

```
💬 [QUESTION] ค่า 0.2 และ 0.1 นี้มาจากไหนครับ? 
มี business rule ที่ define ไว้ที่ไหนไหม? 
อยากเข้าใจว่า premium user ได้ discount 20% เสมอ 
หรือมี edge case อื่นอีก?

🔵 [NIT] อาจแยก magic number ออกเป็น constant เพื่อความชัดเจน:
PREMIUM_DISCOUNT_RATE = 0.2
STANDARD_DISCOUNT_RATE = 0.1
```

### ถามก่อน assume

แทนที่จะบอกว่า "โค้ดนี้ผิด" ให้ถามก่อนว่าทำไมถึงทำแบบนั้น

```
❌ "วิธีนี้ผิด ควรใช้ includes แทน"

✅ "เห็น N+1 query ที่อาจเกิดขึ้นตรงนี้ครับ 
   ลองใช้ includes(:user) ดูไหม? 
   หรือมีเหตุผลที่ใช้ lazy loading ตรงนี้อยู่แล้ว?"
```

### Acknowledge good code

อย่าแสดงความคิดเห็นเฉพาะตอนมีปัญหา การชมโค้ดที่ดีสร้าง psychological safety ในทีม

```
✨ [PRAISE] ชอบ approach นี้มากครับ! 
การใช้ Form Object แทนที่จะ validate ใน Controller 
ทำให้โค้ดสะอาดและ testable มากขึ้น
```

### การให้ comment บน thread ที่ resolved

เมื่อ author แก้ comment แล้ว ให้ close thread พร้อม acknowledge:

```
✅ แก้แล้วครับ ขอบคุณที่ชี้แนะ!
(reviewer กด Resolve)
```

### Synchronous vs Asynchronous review

- **Comment เล็กๆ:** ใช้ async GitHub comment
- **เถียงกันในวงกว้าง (3+ round trips):** นัด call สั้นๆ 15 นาทีดีกว่า

```
ถ้า thread มี comment มากกว่า 5 ไปๆ มาๆ 
ให้พิจารณา: "เราควร call กันสั้นๆ ดีกว่า"
```

---

## Step 964: เทคนิค review ที่มีประสิทธิภาพ — อ่าน tests ก่อน, run locally, check edge cases {#step-964}

การ review โค้ดให้ดีต้องมีระบบ ไม่ใช่แค่เลื่อนดู diff แล้ว approve

### ลำดับการ review ที่มีประสิทธิภาพ

**ขั้นที่ 1: อ่าน PR description ก่อนเสมอ**

เข้าใจ context ก่อนดูโค้ด ถ้า description ไม่ชัด ให้ถามก่อนเริ่ม review

**ขั้นที่ 2: อ่าน tests ก่อน**

```ruby
# ดู spec ก่อน implementation
# tests บอก "intention" ของโค้ดได้ดีกว่า implementation
RSpec.describe StripeWebhookService do
  describe "#process" do
    context "when payment_intent.succeeded" do
      it "marks order as paid" do
        # อ่านแค่นี้ก็รู้แล้วว่า service ควรทำอะไร
      end
    end
    
    context "when webhook signature is invalid" do
      it "raises InvalidSignatureError" do
        # edge case ที่ต้อง handle
      end
    end
  end
end
```

ถ้า tests ไม่ครอบคลุม edge cases ที่คุณคิดถึง ให้ comment ขอเพิ่ม

**ขั้นที่ 3: ดู high-level structure**

ก่อนลงรายละเอียด ดู files changed ในภาพรวม:
- มี file ที่ไม่ควรอยู่ใน PR นี้ไหม?
- โครงสร้างโดยรวมสมเหตุสมผลไหม?

**ขั้นที่ 4: อ่าน implementation อย่างละเอียด**

ตรวจสอบ:

```ruby
# ❓ Security: มี input validation ไหม?
def create
  @post = Post.new(post_params)  # ✅ strong params
  # vs
  @post = Post.new(params[:post])  # ❌ mass assignment vulnerability
end

# ❓ Performance: มี N+1 query ไหม?
# ❌
users.each { |u| puts u.orders.count }

# ✅
users.includes(:orders).each { |u| puts u.orders.size }

# ❓ Error handling: ครอบคลุมกรณี failure ไหม?
def charge_card(amount)
  Stripe::Charge.create(amount: amount)
  # ❌ ถ้า Stripe raise error จะเกิดอะไร?
end

def charge_card(amount)
  Stripe::Charge.create(amount: amount)
rescue Stripe::CardError => e
  # ✅ handle gracefully
  Rails.logger.error "Card charge failed: #{e.message}"
  false
end
```

**ขั้นที่ 5: Run locally สำหรับ PR ที่ซับซ้อน**

```bash
# Checkout PR branch
git fetch origin pull/123/head:pr-123
git checkout pr-123

# Run tests
bundle exec rspec spec/services/stripe_webhook_service_spec.rb

# ทดสอบด้วยตัวเองถ้า UI change
rails server
```

### Checklist mental model สำหรับ reviewer

```
□ โค้ดทำในสิ่งที่ PR description บอกไหม?
□ Tests ครอบคลุม happy path และ edge cases ไหม?
□ มี security vulnerability ไหม? (SQL injection, XSS, CSRF)
□ มี N+1 query หรือ performance issue ไหม?
□ Error handling เหมาะสมไหม?
□ Naming ชัดเจนและสื่อความหมายไหม?
□ มี magic number/string ที่ควรเป็น constant ไหม?
□ มี dead code หรือ debug code หลงเหลือไหม?
□ Database migration reversible ไหม?
□ โค้ดสอดคล้องกับ existing patterns ในโปรเจกต์ไหม?
```

---

## Step 965: RFC (Request for Comments) คืออะไร — เมื่อไหรควรเขียน RFC ก่อนลงมือ code {#step-965}

RFC คือเอกสารที่เสนอการเปลี่ยนแปลงสำคัญต่อระบบ และเชิญชวนให้ทีมแสดงความคิดเห็น **ก่อนที่จะเริ่มเขียนโค้ด**

### ทำไม RFC จึงสำคัญ?

ลองนึกภาพ: นักพัฒนาทำงาน 2 สัปดาห์บนฟีเจอร์ใหม่ พอ demo ให้ทีมดู ปรากฏว่า:
- Approach ขัดกับ architecture decision ที่ทีมตัดสินใจไว้เดือนก่อน
- มีคนอีกคนกำลังทำงานที่ overlap กัน
- Security team มีข้อกังวลที่ทำให้ต้องออกแบบใหม่ทั้งหมด

เวลา 2 สัปดาห์สูญเปล่า! RFC ป้องกันเรื่องแบบนี้

### เมื่อไหรต้องเขียน RFC

```
✅ ควรเขียน RFC เมื่อ:
- เปลี่ยน database schema ใหญ่ๆ (เพิ่ม table, เปลี่ยน relation)
- เพิ่ม dependency สำคัญ (gem ใหม่ที่ affect ทั้งระบบ)
- เปลี่ยน authentication/authorization approach
- ออกแบบ API ใหม่ที่ public-facing
- Introduce ระบบใหม่ (search engine, message queue, cache layer)
- เปลี่ยน deployment process
- ตัดสินใจเรื่อง architecture ที่มีผลระยะยาว

❌ ไม่ต้องเขียน RFC สำหรับ:
- Bug fix ทั่วไป
- Refactor เล็กๆ ที่ไม่กระทบ behavior
- เพิ่ม feature เล็กๆ ใน existing flow
- Update gem versions (minor)
```

### กระบวนการ RFC

```
1. เขียน RFC draft → share กับทีม
2. Feedback period (3-7 วัน ขึ้นกับความซับซ้อน)
3. Incorporate feedback → update draft
4. Final decision: approve / reject / postpone
5. ถ้า approve → เริ่ม implementation
6. Link RFC ใน PR ที่ implement
```

### ตัวอย่าง RFC ที่ดีจาก open source

Rails เองก็ใช้ RFC process ผ่าน GitHub Issues ที่ label "proposal" ก่อนที่จะ merge feature ใหม่ เช่น Hotwire, Solid Queue, Kredis

---

## Step 966: โครงสร้าง RFC ที่ดี — Context, Problem, Proposed Solution, Alternatives, Trade-offs {#step-966}

RFC ที่ดีต้องครอบคลุมองค์ประกอบสำคัญเหล่านี้:

### Template RFC สำหรับ Rails Project

```markdown
# RFC-042: เปลี่ยนระบบ Authentication จาก Devise เป็น custom JWT

**สถานะ:** Draft | Proposed | Accepted | Rejected | Superseded
**ผู้เขียน:** @your-name
**วันที่:** 2024-01-15
**Reviewers:** @tech-lead, @security-team

---

## 1. Context และ Background

ปัจจุบัน application ใช้ Devise สำหรับ authentication 
โดยใช้ session-based auth ที่เก็บ state บน server

ทีมกำลังพัฒนา mobile app และ third-party integrations 
ที่ต้องการ stateless authentication

## 2. Problem Statement

Session-based authentication มีข้อจำกัดสำหรับ:
- Mobile clients ที่ไม่ใช้ cookies
- Microservice ที่ต้องการ auth โดยไม่ต้อง hit database
- Third-party API consumers

## 3. Proposed Solution

เพิ่ม JWT token endpoint ควบคู่กับ Devise (ไม่ replace):
- `POST /api/auth/token` → return JWT
- JWT มีอายุ 1 ชั่วโมง + refresh token 30 วัน
- ใช้ `jwt` gem v2.x
- เก็บ refresh tokens ใน database (revocable)

### Implementation Plan
\`\`\`ruby
# app/controllers/api/auth/tokens_controller.rb
class Api::Auth::TokensController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    if user&.valid_password?(params[:password])
      token = JwtService.encode(user_id: user.id)
      render json: { token: token, expires_in: 3600 }
    else
      render json: { error: "Invalid credentials" }, status: :unauthorized
    end
  end
end
\`\`\`

## 4. Alternatives Considered

### Alternative A: ใช้ Devise JWT gem
- Pros: integrate กับ Devise ได้ทันที
- Cons: gem ไม่ค่อย maintained, opinionated มากเกินไป

### Alternative B: ใช้ Doorkeeper (OAuth2)
- Pros: มาตรฐาน OAuth2, third-party support
- Cons: complexity สูง, overkill สำหรับ use case ปัจจุบัน

### Alternative C: ใช้ external auth service (Auth0)
- Pros: ไม่ต้อง maintain เอง
- Cons: vendor lock-in, cost เพิ่ม, latency

## 5. Trade-offs และ Risks

| ด้าน | Benefit | Risk |
|------|---------|------|
| Security | Stateless, ไม่ต้อง DB per request | JWT cannot be invalidated immediately |
| Performance | ลด DB queries | Token size ใหญ่กว่า session ID |
| Complexity | Simple implementation | ต้อง handle token refresh logic |

**Security Risk ที่ต้องระวัง:**
- JWT ที่ถูก steal ใช้ได้จน expire (1 ชั่วโมง)
- Mitigation: refresh token rotation + blacklist critical tokens

## 6. Migration Plan

Phase 1 (Week 1-2): เพิ่ม JWT endpoint โดยไม่กระทบ web
Phase 2 (Week 3-4): Mobile app ใช้ JWT
Phase 3 (Month 2): Monitor และ optimize

## 7. Success Metrics

- Mobile auth latency < 100ms
- Zero session-related errors บน mobile
- Token refresh success rate > 99.9%

## 8. Open Questions

- [ ] ควร revoke JWT ทันทีเมื่อ user เปลี่ยน password ไหม?
- [ ] จะ handle rate limiting บน token endpoint อย่างไร?

---
**Comments ยินดีรับทาง GitHub Discussions ภายใน 15 Jan 2024**
```

---

## Step 967: Design Doc สำหรับ feature ขนาดกลาง-ใหญ่ — ER diagram, API contract, sequence diagram {#step-967}

Design Doc ต่างจาก RFC ตรงที่เน้นรายละเอียดทางเทคนิคมากกว่า และมักเขียนหลังจากที่ RFC ผ่านการ approve แล้ว

### ส่วนประกอบของ Design Doc ที่ดี

**1. ER Diagram สำหรับ Database Design**

```
ใช้ Mermaid.js ในเอกสาร Markdown:

\`\`\`mermaid
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "included in"
    ORDER {
        bigint id PK
        bigint user_id FK
        string status
        decimal total_amount
        timestamp created_at
    }
    ORDER_ITEM {
        bigint id PK
        bigint order_id FK
        bigint product_id FK
        integer quantity
        decimal unit_price
    }
\`\`\`
```

**2. API Contract**

```markdown
### POST /api/v1/orders

**Request:**
\`\`\`json
{
  "order": {
    "items": [
      { "product_id": 123, "quantity": 2 },
      { "product_id": 456, "quantity": 1 }
    ],
    "shipping_address_id": 789,
    "coupon_code": "SAVE10"
  }
}
\`\`\`

**Response (201 Created):**
\`\`\`json
{
  "data": {
    "id": "ord_abc123",
    "status": "pending",
    "items": [...],
    "subtotal": "1500.00",
    "discount": "150.00",
    "total": "1350.00",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
\`\`\`

**Error (422 Unprocessable Entity):**
\`\`\`json
{
  "errors": [
    { "field": "items[0].quantity", "message": "Exceeds available stock" }
  ]
}
\`\`\`
```

**3. Sequence Diagram สำหรับ flow ที่ซับซ้อน**

```
\`\`\`mermaid
sequenceDiagram
    participant Client
    participant Rails
    participant Stripe
    participant Worker

    Client->>Rails: POST /orders { items, payment_method_id }
    Rails->>Rails: Validate stock availability
    Rails->>Stripe: Create PaymentIntent
    Stripe-->>Rails: { client_secret, status: "requires_confirmation" }
    Rails-->>Client: { order_id, client_secret }
    
    Client->>Stripe: Confirm payment (3DS if needed)
    Stripe->>Rails: webhook: payment_intent.succeeded
    Rails->>Worker: Enqueue FulfillmentJob
    Worker->>Rails: Mark order as "processing"
    Rails->>Client: (email notification)
\`\`\`
```

### Rails-specific Design Considerations

```ruby
# ในส่วน "Implementation Notes" ของ Design Doc ให้ระบุ:

# 1. Service Objects ที่จะสร้าง
# - OrderCreationService: orchestrate order creation
# - StockReservationService: lock stock atomically
# - StripePaymentService: handle Stripe interactions

# 2. Database indexes ที่จำเป็น
# add_index :orders, :user_id
# add_index :orders, [:status, :created_at]
# add_index :order_items, [:order_id, :product_id], unique: true

# 3. Background jobs
# - FulfillmentJob: สั่งของจาก warehouse
# - OrderReminderJob: remind ถ้า order pending นานเกินไป
```

---

## Step 968: ADR (Architecture Decision Record) — บันทึกการตัดสินใจสำคัญไว้ใน repository {#step-968}

ADR แก้ปัญหาคลาสสิกในทีม software: "ทำไมเราถึงทำแบบนี้?" และ "ใครตัดสินใจ?"

### ADR คืออะไร?

ADR เป็นเอกสารสั้นๆ ที่บันทึก:
- **บริบท:** ทำไมต้องตัดสินใจ
- **การตัดสินใจ:** เลือกอะไร
- **เหตุผล:** ทำไมถึงเลือกแบบนั้น
- **ผลที่ตามมา:** ผลดีและผลเสีย

### โครงสร้าง repository

```
docs/
  adr/
    0001-use-postgresql-as-primary-database.md
    0002-use-sidekiq-for-background-jobs.md
    0003-adopt-api-first-design-with-json-api.md
    0004-use-react-for-frontend-not-erb.md
    README.md  (index ของ ADR ทั้งหมด)
```

### Template ADR

```markdown
# ADR-0003: ใช้ JSON:API format สำหรับ REST API

**Date:** 2024-01-10
**Status:** Accepted
**Deciders:** @tech-lead, @backend-team
**Supersedes:** -
**Superseded by:** -

## Context

กำลังออกแบบ public API สำหรับ mobile app และ third-party
ทีมต้องตัดสินใจว่าจะใช้ response format แบบไหน

## Decision

เลือกใช้ JSON:API specification (jsonapi.org)

## Rationale

1. **Standardized:** client ที่รองรับ JSON:API ทำงานได้กับ API เราโดยไม่ต้องเขียน custom code
2. **Relationships:** JSON:API handle `included` resources ได้ดี ลด N+1 บน client side
3. **Tooling:** `jsonapi-serializer` gem มี community support ดี

## Consequences

### ด้านดี
- Client developers ที่คุ้นกับ JSON:API เข้าใจ API ได้ทันที
- Pagination, sorting, filtering มี convention ชัดเจน

### ด้านเสีย
- Response verbose กว่า custom JSON
- Team ต้องเรียน JSON:API specification (learning curve ~1 วัน)
- `jsonapi-serializer` เพิ่ม dependency

### Neutral
- Existing internal API ยังใช้ custom JSON ได้ ไม่ต้อง migrate

## Links
- https://jsonapi.org/
- https://github.com/jsonapi-serializer/jsonapi-serializer
- RFC-031 (API Design)
```

### เมื่อไหรต้องเขียน ADR

```
✅ เขียน ADR สำหรับ:
- เลือก database (PostgreSQL vs MySQL vs MongoDB)
- เลือก background job (Sidekiq vs GoodJob vs Solid Queue)
- เลือก frontend approach (Hotwire vs React vs Vue)
- เลือก deployment platform (Heroku vs AWS vs Fly.io)
- ตัดสินใจ deprecate หรือ remove feature สำคัญ
- เปลี่ยน authentication strategy

❌ ไม่ต้องเขียน ADR สำหรับ:
- เลือก Ruby version minor release
- เลือก gem สำหรับ utility เล็กๆ
- Implementation details ที่ไม่ affect architecture
```

### Tools สำหรับ ADR

```bash
# adr-tools: command line tool สำหรับ manage ADR
gem install adr-tools  # หรือ brew install adr-tools

# สร้าง ADR ใหม่
adr new "Use Solid Queue for background jobs"
# → สร้าง docs/adr/0005-use-solid-queue-for-background-jobs.md

# List ADR ทั้งหมด
adr list
```

---

## Step 969: การ facilitate technical discussion — ตั้งคำถามที่ดี, consensus vs consent, เมื่อไหรต้อง escalate {#step-969}

Senior Engineer ต้องเป็นมากกว่า technical expert — ต้องเป็น facilitator ของการตัดสินใจทางเทคนิค

### ตั้งคำถามที่ดีแทนการ dictate

แทนที่จะบอกคำตอบทันที ให้ตั้งคำถามที่นำทางทีมไปสู่ความเข้าใจร่วมกัน:

```
❌ "เราควรใช้ Redis ตรงนี้"

✅ "เราต้องการ cache นี้มีลักษณะแบบไหน?
   - อยากให้ expire อัตโนมัติไหม?
   - ต้องการ cache warmup ไหม?
   - ถ้า Redis ล่ม ระบบควร fallback อย่างไร?
   
   ถ้าต้องการทุกอย่างนั้น Redis น่าจะเหมาะ
   แต่ถ้าต้องการแค่ in-memory cache เดี๋ยวเดียว 
   อาจใช้ Rails.cache ที่ memory store แทนก็ได้"
```

### Consensus vs Consent

ความแตกต่างสำคัญที่ทีมเทคนิคต้องเข้าใจ:

```
Consensus: ทุกคนเห็นด้วย 100%
→ ช้ามาก ในทางปฏิบัติแทบเป็นไปไม่ได้
→ ทำให้ทีมติดอยู่กับการตัดสินใจที่ไม่สำคัญนาน

Consent: ไม่มีใคร "object อย่างจริงจัง"
→ เร็วกว่ามาก
→ ทุกคนอาจไม่ใช่ fan ของการตัดสินใจ แต่ไม่มีใข้คัดค้านอย่างแข็งขัน
→ เหมาะกับการตัดสินใจส่วนใหญ่ในทีม engineering
```

วิธีใช้ consent ใน technical discussion:

```
"ผมอยากเสนอให้ใช้ Solid Queue แทน Sidekiq 
เพราะไม่ต้อง maintain Redis แยก
มีใครมีข้อกังวลที่ต้อง address ก่อนที่เราจะตัดสินใจไหม?"

(รอ 1 สัปดาห์)

ถ้าไม่มีข้อกังวลที่ถูก raise → ถือว่า consent แล้ว
```

### เมื่อไหรต้อง escalate

```
Escalate เมื่อ:
1. ทีมเถียงกันเกิน 2-3 วัน โดยไม่มีทีท่าจะ resolve
2. การตัดสินใจมีผลกระทบข้ามทีม (cross-team dependency)
3. การตัดสินใจต้องใช้ budget หรือ resource นอกเหนือ scope
4. มี security หรือ compliance implication ที่ต้องการ expert opinion

ผู้ที่ escalate ไป:
- Technical Lead / Engineering Manager
- Security Team (สำหรับ security decisions)
- Architecture Committee (ถ้ามี)
- VP Engineering (สำหรับ strategic decisions)
```

### การ timeboxing การตัดสินใจ

```
ก่อน discussion: "เราจะใช้เวลาพูดคุยเรื่องนี้ 30 นาที 
ถ้ายังไม่ได้ข้อสรุป จะให้ Tech Lead ตัดสินใจ"

→ ลด analysis paralysis
→ ทุกคนรู้ว่ามี deadline ชัดเจน
→ ใครที่อยากมีผลต้องแสดงความคิดเห็นภายในเวลา
```

---

## Step 970: เอกสาร CONTRIBUTING.md และ ARCHITECTURE.md ที่ดี — ทำให้คนใหม่เข้ามา contribute ง่าย {#step-970}

เอกสารที่ดีคือการ scale ความรู้ของคุณให้ทำงานแม้ตอนคุณไม่อยู่

### CONTRIBUTING.md

ไฟล์นี้ตอบคำถามของคนที่อยากมีส่วนร่วมในโปรเจกต์ว่า "ฉันจะเริ่มต้นอย่างไร?"

```markdown
# Contributing to ProjectName

## Setup Development Environment

\`\`\`bash
# 1. Clone repository
git clone https://github.com/org/project.git
cd project

# 2. Install dependencies
bundle install
yarn install

# 3. Setup database
cp .env.example .env
# แก้ไข .env ตามความต้องการ
bundle exec rails db:create db:migrate db:seed

# 4. Run tests
bundle exec rspec
bundle exec rubocop

# 5. Start development server
bin/dev
\`\`\`

## Branch Naming Convention

\`\`\`
feature/TICKET-123-short-description
bugfix/TICKET-456-fix-login-error
hotfix/TICKET-789-critical-payment-bug
chore/update-dependencies
docs/add-api-documentation
\`\`\`

## Commit Message Format

เราใช้ Conventional Commits:
\`\`\`
feat(payment): add Stripe webhook handler
fix(auth): resolve JWT expiration edge case
docs(api): update authentication endpoint docs
chore(deps): update Rails to 7.2.1
\`\`\`

## Pull Request Process

1. สร้าง feature branch จาก `main`
2. เขียน tests ก่อน (TDD ถ้าทำได้)
3. ตรวจสอบ `bundle exec rspec` และ `bundle exec rubocop` ผ่าน
4. เปิด PR พร้อม description ที่ครบถ้วน
5. ขอ review จากทีมสมาชิกอย่างน้อย 1 คน
6. Squash merge เมื่อ approve แล้ว

## Code Style

- ใช้ RuboCop: `bundle exec rubocop -a` แก้ auto-fixable issues
- Line length สูงสุด 120 characters
- ใช้ double quotes สำหรับ strings
- Avoid `unless` กับ complex conditions

## Testing Requirements

- Model specs: ทุก validation และ method
- Service specs: ทุก service object
- Request specs: ทุก API endpoint
- System specs: สำหรับ critical user flows

## ต้องการความช่วยเหลือ?

- สร้าง Discussion ใน GitHub Discussions
- ถาม ใน Slack channel #engineering
- Tag @tech-lead บน PR ถ้าไม่มี reviewer ใน 24 ชั่วโมง
```

### ARCHITECTURE.md

ไฟล์นี้อธิบาย "big picture" ของระบบสำหรับ developer ใหม่

```markdown
# Architecture Overview

## System Overview

ProjectName เป็น e-commerce platform สร้างด้วย Ruby on Rails 7
ให้บริการ 50,000+ users และ process ~10,000 orders/day

## Tech Stack

| Component | Technology | เหตุผล |
|-----------|-----------|--------|
| Backend | Ruby on Rails 7.1 | ADR-0001 |
| Database | PostgreSQL 16 | ADR-0002 |
| Cache | Redis 7 | ADR-0003 |
| Background Jobs | Sidekiq 7 | ADR-0004 |
| Frontend | Hotwire (Turbo + Stimulus) | ADR-0005 |
| Search | Elasticsearch 8 | ADR-0006 |
| File Storage | AWS S3 | ADR-0007 |
| CDN | CloudFront | ADR-0007 |
| Deploy | Kamal 2 | ADR-0010 |

## Application Structure

\`\`\`
app/
├── controllers/
│   ├── api/v1/           # API endpoints
│   └── admin/            # Admin panel
├── models/               # ActiveRecord models
├── services/             # Business logic (Service Objects)
│   ├── orders/           # Order-related services
│   ├── payments/         # Payment processing
│   └── notifications/    # Email/SMS
├── jobs/                 # Background jobs
├── serializers/          # JSON serializers (jsonapi-serializer)
└── policies/             # Authorization (Pundit)
\`\`\`

## Data Flow

ดู docs/diagrams/data-flow.png สำหรับ diagram

หลักการ:
- Web request → Controller → Service → Model → Database
- Async operations → Sidekiq Job → Service → Model
- Events → EventBus → Subscriber → Service

## Key Concepts

### Service Objects
ทุก business logic อยู่ใน Service Objects ไม่ใช่ใน Controllers หรือ Models
- Input: plain Ruby objects
- Output: Result object (success/failure)
- Testable แบบ standalone

### Authorization
ใช้ Pundit สำหรับ authorization ทุกอย่าง
Policy อยู่ใน `app/policies/`
ทุก controller action ต้อง call `authorize @resource`

## ADR Index

ดูการตัดสินใจ architectural ทั้งหมดใน docs/adr/
```

---

## แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: เขียน PR Description ที่ดี

สมมุติว่าคุณต้องการเพิ่มฟีเจอร์ "บันทึกที่อยู่จัดส่งหลายอัน" ให้ผู้ใช้ เขียน PR description ที่สมบูรณ์รวมถึง:
- สรุปการเปลี่ยนแปลง
- เหตุผล
- วิธี test
- Checklist

### แบบฝึกหัดที่ 2: Review โค้ดต่อไปนี้

```ruby
class UsersController < ApplicationController
  def index
    @users = User.all
    @users.each do |user|
      user.update(last_seen: Time.now)
    end
    render json: @users
  end
  
  def create
    @user = User.new(params[:user])
    @user.save
    render json: @user
  end
end
```

เขียน review comments ที่ constructive โดยระบุ severity ของแต่ละ comment

### แบบฝึกหัดที่ 3: เขียน RFC

เขียน RFC สำหรับการเพิ่ม feature "Two-Factor Authentication" ให้กับ Rails application ของคุณ ครอบคลุม: Context, Problem, Proposed Solution, Alternatives, Trade-offs

### แบบฝึกหัดที่ 4: สร้าง ADR

เขียน ADR สำหรับการตัดสินใจเลือก background job framework สำหรับโปรเจกต์ใหม่ เปรียบเทียบระหว่าง Sidekiq, GoodJob, และ Solid Queue

### แบบฝึกหัดที่ 5: ปรับปรุง CONTRIBUTING.md

เปิด open source project ที่คุณใช้บ่อย และดู CONTRIBUTING.md ของเขา วิเคราะห์ว่ามีอะไรดีและอะไรที่สามารถปรับปรุงได้

---

## สรุปสิ่งที่ได้เรียนรู้ {#summary}

ใน Part 097 นี้เราได้เรียนรู้:

| หัวข้อ | สิ่งสำคัญที่ได้เรียน |
|--------|---------------------|
| Code Review | ไม่ใช่แค่หาบัก แต่เป็น knowledge sharing, mentoring, และ quality gate |
| PR ที่ดี | Scope เล็ก, description ชัด, self-review ก่อน, screenshot สำหรับ UI |
| Review Comments | ระบุ severity (BLOCKER/SUGGESTION/NIT), ถามก่อน assume, ชมโค้ดที่ดี |
| เทคนิค Review | อ่าน tests ก่อน, run locally, ตรวจ security/performance/error handling |
| RFC | เขียนก่อน code เมื่อมีการเปลี่ยนแปลงสำคัญ เพื่อรวบรวม feedback |
| Design Doc | ER diagram, API contract, sequence diagram สำหรับ feature ใหญ่ |
| ADR | บันทึกการตัดสินใจ architectural ไว้ใน repository พร้อมเหตุผล |
| Technical Discussion | Consensus vs Consent, timeboxing, รู้จัก escalate เมื่อจำเป็น |
| CONTRIBUTING.md | Setup guide, conventions, PR process สำหรับคนใหม่ |
| ARCHITECTURE.md | Big picture ของระบบ tech stack, data flow, key concepts |

**Key Takeaway:** Senior Engineer ที่ดีไม่ได้แค่เขียนโค้ดเก่ง แต่ต้องสื่อสารทางเทคนิคได้อย่างมีประสิทธิภาพ ทำให้ทีมทำงานร่วมกันได้ดีขึ้น และสร้างระบบที่ maintainable ในระยะยาว เอกสารที่ดีคือ "force multiplier" ที่ทำให้ความรู้ของคุณทำงานแม้ตอนคุณไม่อยู่

**ต่อไป:** Part 098 จะพาไปเรียนรู้ System Design สำหรับ Rails application ขนาดใหญ่ รวมถึง Scaling Checklist ที่ครอบคลุมทุกด้านตั้งแต่ database ไปจนถึง observability
