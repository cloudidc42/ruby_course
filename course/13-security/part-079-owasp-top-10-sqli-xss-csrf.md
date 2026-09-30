# Part 079: OWASP Top 10 ใน Context ของ Rails — SQL Injection, XSS, CSRF

> **Step ครอบคลุมใน Part นี้:** Step 781–790
> **ระดับ:** กลาง–สูง (ควรผ่าน Phase 1–12 มาแล้ว โดยเฉพาะ Part 024, 034, 038, 045, 075)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x
> **นี่คือจุดเริ่มต้นของ Phase 13: Security (Part 079–081)**

## สารบัญของ Part นี้

- Step 781: OWASP Top 10 คืออะไร และทำไมเป็น Checklist มาตรฐานของอุตสาหกรรม
- Step 782: SQL Injection คืออะไร — กลไกเบื้องหลัง และ Pattern อันตรายที่พบใน Rails จริง
- Step 783: ป้องกัน SQL Injection ใน Rails — Parameterized Query (พิสูจน์การป้องกันด้วยของจริง)
- Step 784: Cross-Site Scripting (XSS) คืออะไร — Stored, Reflected, DOM-based
- Step 785: ป้องกัน XSS ใน Rails — Auto-Escaping, `sanitize` (พิสูจน์การป้องกันด้วยของจริง)
- Step 786: Content Security Policy (CSP) — Defense-in-Depth ชั้นที่สองเมื่อ Escaping พลาด
- Step 787: Cross-Site Request Forgery (CSRF) คืออะไร — กลไกการโจมตีเชิงแนวคิด
- Step 788: Rails CSRF Protection — `protect_from_forgery` และ Authenticity Token (พิสูจน์ของจริง)
- Step 789: CSRF ใน API-only Apps — ทำไมจัดการต่างจากแอปแบบ Session ปกติ (ทวน Part 045)
- Step 790: OWASP Top 10 หมวดอื่นในหลักสูตร + แบบฝึกหัดปิดท้าย Part

---

## Step 781: OWASP Top 10 คืออะไร และทำไมเป็น Checklist มาตรฐานของอุตสาหกรรม

### OWASP คืออะไร

**OWASP (Open Worldwide Application Security Project)** เป็นองค์กรไม่แสวงหาผลกำไรระดับโลก
ที่ทำงานด้านความปลอดภัยของซอฟต์แวร์มาตั้งแต่ปี 2001 ผลิตเอกสาร เครื่องมือ และมาตรฐานด้าน
security ที่ทั้งอุตสาหกรรมใช้ร่วมกันแบบเปิด (open source, ไม่มีค่าใช้จ่าย) เอกสารที่มีชื่อเสียง
ที่สุดของ OWASP คือ **OWASP Top 10**

### OWASP Top 10 คืออะไร

**OWASP Top 10** คือรายการ **10 ความเสี่ยงด้านความปลอดภัยของเว็บแอปพลิเคชันที่ร้ายแรงและพบบ่อย
ที่สุด** จัดอันดับโดยอาศัยข้อมูลจริงจากหลายร้อยองค์กรทั่วโลก (ผลการสแกน, ผล penetration test,
รายงานช่องโหว่จริง) ผสมกับความเห็นของผู้เชี่ยวชาญด้านความปลอดภัยหลายร้อยคน อัปเดตทุก 3–4 ปี
(2003, 2004, 2007, 2010, 2013, 2017, และฉบับล่าสุดที่ยังใช้อยู่คือ **2021**)

**ทำไมถึงสำคัญและถือเป็นมาตรฐานอุตสาหกรรม:**

1. **ใช้เป็น baseline ของ compliance** — มาตรฐานอย่าง PCI-DSS (สำหรับระบบที่รับชำระเงิน) อ้างอิง
   OWASP Top 10 โดยตรงว่าเป็นเกณฑ์ขั้นต่ำที่ระบบต้องป้องกันได้
2. **ใช้เป็นขอบเขตของ penetration test/security audit** — เวลาจ้างบริษัทมาทดสอบความปลอดภัยระบบ
   (pentest) ขอบเขตพื้นฐานที่สุดมักอ้างอิง OWASP Top 10 เป็นจุดตั้งต้นเสมอ
3. **เครื่องมือ static analysis สร้างมาเพื่อจับหมวดหมู่นี้โดยตรง** — Brakeman ที่เราใช้ใน Part 075
   ตรวจ check ที่ชื่อ `SQL`, `CrossSiteScripting`, `MassAssignment` ฯลฯ ตรงกับหมวดของ OWASP Top 10
   แทบทุกตัว
4. **ใช้เป็นภาษากลางในการสื่อสารเรื่อง security ระหว่างทีม** — เวลาบอกว่า "เจอ A03 Injection ใน
   endpoint นี้" ทุกคนในวงการเข้าใจตรงกันทันทีว่าเสี่ยงระดับไหน โดยไม่ต้องอธิบายยาว

### OWASP Top 10 ฉบับ 2021 (ฉบับปัจจุบัน)

| อันดับ | หมวด | ตัวอย่าง |
|--------|------|----------|
| A01 | Broken Access Control | ผู้ใช้ทำสิ่งที่ตัวเองไม่มีสิทธิ์ทำได้ เช่น แก้ไขข้อมูลของคนอื่นผ่านการเปลี่ยน id ใน URL |
| A02 | Cryptographic Failures | เก็บรหัสผ่าน/ข้อมูลอ่อนไหวแบบไม่เข้ารหัส, ใช้ algorithm เข้ารหัสที่อ่อนแอ |
| A03 | **Injection** | **SQL Injection, Cross-Site Scripting (XSS), Command Injection** — Part นี้เน้น 2 ตัวแรก |
| A04 | Insecure Design | ออกแบบ flow/feature ผิดพลาดตั้งแต่ระดับ architecture ไม่ใช่แค่บั๊กโค้ด |
| A05 | Security Misconfiguration | ตั้งค่า default ที่ไม่ปลอดภัย, เปิด debug mode ใน production, HTTP header ที่ควรมีแต่ไม่มี |
| A06 | Vulnerable and Outdated Components | ใช้ gem/library เวอร์ชันเก่าที่มีช่องโหว่ที่รู้จักแล้ว (known CVE) |
| A07 | Identification and Authentication Failures | ระบบ login/session อ่อนแอ, ไม่ล็อกบัญชีหลัง brute force |
| A08 | Software and Data Integrity Failures | เชื่อ code/data จากแหล่งที่ตรวจสอบไม่ได้ (เช่น CI/CD supply chain, insecure deserialization) |
| A09 | Security Logging and Monitoring Failures | ระบบถูกโจมตีสำเร็จแต่ไม่มี log/alert ให้รู้ตัวเลย |
| A10 | Server-Side Request Forgery (SSRF) | หลอกให้ server ยิง request ไปยังปลายทางที่ผู้โจมตีกำหนด (เช่น internal network) |

> **หมายเหตุสำคัญเรื่อง CSRF:** สังเกตว่า **CSRF ไม่ได้อยู่ในลิสต์ OWASP Top 10 ฉบับปัจจุบันแล้ว**
> (ถูกถอดออกตั้งแต่ฉบับ 2017) เหตุผลไม่ใช่เพราะ CSRF ไม่อันตราย แต่เพราะ **framework สมัยใหม่
> ส่วนใหญ่ (รวมถึง Rails) มีกลไกป้องกันมาให้ในตัวโดย default อยู่แล้ว** ทำให้อัตราการพบช่องโหว่นี้
> ในระบบจริงลดลงมากจนหลุดจาก Top 10 — แต่นั่นไม่ได้แปลว่าเราไม่ต้องเข้าใจมัน **ตรงกันข้าม
> วิศวกรที่ดีต้องเข้าใจว่า "ทำไมมันถึงถูกป้องกันได้ดีแล้ว"** เพราะทันทีที่สร้างระบบ auth เอง, ทำ API
> ที่ผสมกับ cookie, หรือปิด default protection โดยไม่รู้ตัว ความเสี่ยงนี้ก็กลับมาได้ทันที — Part นี้
> เลยยังสอน CSRF อย่างเจาะลึกเป็นหนึ่งในสามเสาหลัก

### Roadmap: Part ไหนในหลักสูตรคุมหมวดไหนของ OWASP Top 10

หลักสูตรนี้ไม่ได้ยัดทุกหมวดไว้ Part เดียว แต่กระจายไปตามจุดที่เนื้อหาเกี่ยวข้องโดยตรง (เพื่อให้เรียน
พร้อม context จริง ไม่ใช่ท่องทฤษฎีลอยๆ):

- **A01 Broken Access Control** → สอนไปแล้วใน **Part 043 (Pundit)** และ **Part 044 (CanCanCan/RBAC)**
- **A02 Cryptographic Failures** → **Part 081** (secrets management, ActiveRecord Encryption)
- **A03 Injection (SQLi/XSS)** → **Part นี้ (079)**
- **A05 Security Misconfiguration** → **Part 080** (ต่อจาก Part นี้ทันที)
- **A06 Vulnerable and Outdated Components** → สอนไปแล้วใน **Part 075** (Brakeman + `bundler-audit`
  ใน CI) จะสรุปทวนสั้นๆ ใน Step 790
- **A07 Identification and Authentication Failures** → **Part 041, 042, 045** (auth ทั้งหมด)
- **A09 Security Logging and Monitoring Failures** → **Part 078** (structured logging, Sentry)
- **A04 Insecure Design, A08 Integrity Failures, A10 SSRF** → แทรกอยู่ในหลักการออกแบบตลอดทั้ง
  หลักสูตร โดยเฉพาะ Phase 14 (Architecture) และจะถูกพูดถึงในเช็กลิสต์ก่อนขึ้น production ที่
  **Part 081**

Part นี้จะโฟกัสที่ **A03 Injection ในรูปแบบที่พบบ่อยที่สุดสองแบบ (SQL Injection และ XSS)** และ
**CSRF** (แม้จะหลุดจาก Top 10 อย่างเป็นทางการแล้ว แต่ยังเป็นความรู้พื้นฐานที่ขาดไม่ได้) — ทั้งสาม
เรื่องนี้มีจุดร่วมกันคือ: **เกิดจากการเอา "ข้อมูลที่ผู้ใช้ควบคุมได้" ไปปนกับ "โครงสร้างคำสั่ง/โค้ด/ความ
น่าเชื่อถือของ request" โดยไม่ผ่านการตรวจสอบ/แยกให้ชัดเจนก่อน** — เป็น pattern ทางความคิดเดียวกัน
ที่ปรากฏในรูปแบบต่างกัน 3 แบบ

---

## Step 782: SQL Injection คืออะไร — กลไกเบื้องหลัง และ Pattern อันตรายที่พบใน Rails จริง

### กลไกเบื้องหลัง SQL Injection

ฐานข้อมูล SQL ทำงานโดยรับ **string คำสั่ง** เข้ามาแล้ว parse เป็นโครงสร้างคำสั่ง (query plan) ก่อน
รัน ปัญหาเกิดขึ้นเมื่อโปรแกรมสร้าง string คำสั่งนั้นด้วยการ **เอาข้อมูลที่ผู้ใช้ควบคุมได้ (เช่นจาก
`params`) ไปต่อ (concatenate/interpolate) เข้ากับ string คำสั่ง SQL โดยตรง** — เมื่อทำแบบนี้
**ฐานข้อมูลไม่มีทางแยกแยะได้เลยว่าส่วนไหนของ string คือ "ข้อมูล (data)" และส่วนไหนคือ "คำสั่ง
(code/syntax)"** ทั้งหมดถูกมองเป็น SQL syntax เดียวกันหมด ถ้าข้อมูลที่ผู้ใช้ส่งมาบังเอิญ (หรือตั้งใจ)
มีอักขระที่มีความหมายพิเศษใน SQL (เช่น `'`, `;`, `--`, `)`) อักขระเหล่านั้นก็จะถูกตีความเป็นส่วนหนึ่ง
ของโครงสร้างคำสั่งทันที ไม่ใช่แค่ข้อมูลธรรมดาอีกต่อไป — นี่คือรากของช่องโหว่ทั้งหมด ไม่ใช่แค่กับ SQL
เท่านั้น แต่เป็นรากเดียวกับ Command Injection, LDAP Injection ฯลฯ ในภาษาอื่นด้วย

### Pattern อันตรายที่พบบ่อยในโค้ด Rails จริง

```ruby
# อันตรายที่ 1: string interpolation ตรงๆ ใน where
class UsersController < ApplicationController
  def search
    @users = User.where("email = '#{params[:email]}'")
  end
end
```

```ruby
# อันตรายที่ 2: ต่อ id เข้า SQL ตรงๆ (แม้จะ "ดูเหมือน" เป็นตัวเลขก็ตาม)
class OrdersController < ApplicationController
  def show
    @order = Order.where("id = #{params[:id]}").first
  end
end
```

```ruby
# อันตรายที่ 3: raw SQL ผ่าน connection.execute โดยตรง
class ReportsController < ApplicationController
  def monthly
    sql = "SELECT SUM(total) FROM orders WHERE customer_name = '#{params[:name]}'"
    result = ActiveRecord::Base.connection.execute(sql)
  end
end
```

```ruby
# อันตรายที่ 4: sort/order จาก params (ทวนจาก Part 038 Step 375)
class PostsController < ApplicationController
  def index
    @posts = Post.order("#{params[:sort]} #{params[:direction]}")
  end
end
```

จุดร่วมของทั้ง 4 ตัวอย่าง: **ค่าที่มาจาก `params` (ซึ่งผู้ใช้ควบคุมได้ 100% ผ่าน URL, form, หรือ
API request ธรรมดาที่สุด — ไม่ต้องใช้เครื่องมือพิเศษใดๆ เลย) ถูกนำไปประกอบเป็นส่วนหนึ่งของ string
คำสั่ง SQL โดยตรง** ไม่ว่าจะผ่าน string interpolation (`#{}`) หรือการต่อ string ด้วยวิธีอื่นก็ตาม

### สิ่งที่ผู้โจมตีทำได้ในทางทฤษฎี (อธิบายเชิงแนวคิด ไม่สาธิตการโจมตีจริง)

เมื่อค่าที่ผู้ใช้ควบคุมได้หลุดเข้าไปเป็นส่วนหนึ่งของ syntax คำสั่ง SQL โดยไม่มีการป้องกัน ผู้โจมตี
สามารถสร้าง input ที่ทำให้:

1. **เปลี่ยนโครงสร้าง logic ของ `WHERE` clause** — เดิมทีเงื่อนไขอาจถูกออกแบบให้ match แค่แถวเดียว
   (เช่น ตรวจสอบ email + password ในระบบ login ที่เขียนด้วย raw SQL) แต่ input ที่ออกแบบมาเฉพาะ
   สามารถทำให้เงื่อนไขทั้งก้อนกลายเป็นจริงเสมอ (always-true) ทำให้ query คืนค่าแถวที่ไม่ควรจะ match
   เลย เช่น หลุดผ่านการตรวจสอบ authentication โดยไม่ต้องรู้รหัสผ่านจริง
2. **อ่านข้อมูลจากตารางอื่นที่ไม่เกี่ยวข้องกับ query เดิม** — ด้วยเทคนิคที่เรียกว่า UNION-based
   injection ผู้โจมตีสามารถต่อท้าย query เดิมด้วยคำสั่งที่ดึงคอลัมน์จากตารางอื่น (เช่น ตาราง users ที่
   เก็บ password hash) มาแสดงปนกับผลลัพธ์ที่หน้าเว็บตั้งใจจะแสดงอยู่แล้ว
3. **สืบข้อมูลทีละบิตผ่าน error message หรือ response ที่แตกต่างกัน** (error-based / boolean-based
   blind injection) — แม้หน้าเว็บจะไม่แสดงผลลัพธ์ query ตรงๆ ผู้โจมตีก็ยังสามารถถามคำถามใช่/ไม่ใช่
   กับฐานข้อมูลทีละคำถามจนประกอบข้อมูลทั้งหมดขึ้นมาได้ (ใช้เวลานานแต่เป็นไปได้จริงในทางปฏิบัติ)
4. **ทำให้ query ทำงานหนักผิดปกติจนเป็น denial-of-service** — เช่น แทรก subquery ที่คำนวณหนักมาก
   เข้าไปใน `ORDER BY` (แบบที่ Part 038 Step 375 สาธิตให้เห็น error `UnknownAttributeReference`
   ตอนพยายามฉีด subquery ผ่าน `Arel.sql` โดยไม่กรอง)
5. **ในกรณีเลวร้ายที่สุด (ขึ้นกับ adapter/permission ของ DB user ที่แอปใช้เชื่อมต่อ)** แก้ไขหรือลบ
   ข้อมูลได้ผ่าน statement เพิ่มเติม แม้ adapter มาตรฐานอย่าง `pg`/`mysql2`/`sqlite3` ที่ ActiveRecord
   ใช้จะจำกัดให้รันได้ทีละ statement ต่อการเรียกหนึ่งครั้งเป็นค่า default ก็ตาม ความเสี่ยงจึงขึ้นกับ
   ว่าช่องโหว่นั้นอยู่ในตำแหน่งที่รับ multi-statement ได้หรือไม่

**ข้อสังเกตสำคัญ:** ช่องโหว่นี้ไม่ต้องการเครื่องมือแฮ็กพิเศษใดๆ เลย — แค่ HTTP request ธรรมดาที่สุด
(เปลี่ยนค่าใน URL หรือ form ปกติ) ก็เพียงพอแล้ว นี่คือเหตุผลที่ Part 038 ถึงเตือนไว้อย่างจริงจังแม้กับ
ฟีเจอร์เล็กๆ อย่าง "การเรียงลำดับตารางจากการคลิกหัวคอลัมน์" และเป็นเหตุผลที่ Brakeman (Part 075)
มี check ชื่อ `SQL` แยกออกมาเฉพาะเพื่อตรวจจับ pattern นี้โดยเฉพาะ — Brakeman ตรวจจับ `data flow`
จาก `params` ไปยังจุดอันตรายแบบนี้ได้โดยอัตโนมัติในทุกครั้งที่รัน CI (ดู Part 075 Step 748 ที่จงใจใส่
โค้ดแบบ Pattern อันตรายที่ 3 ข้างต้นเพื่อพิสูจน์ว่า Brakeman จับได้จริง)

---

## Step 783: ป้องกัน SQL Injection ใน Rails — Parameterized Query (พิสูจน์การป้องกันด้วยของจริง)

### หลักการแก้: แยก "ข้อมูล" ออกจาก "โครงสร้างคำสั่ง" เสมอ

วิธีแก้ที่ถูกต้องและเป็นมาตรฐานอุตสาหกรรมคือ **parameterized query (หรือ prepared statement)** —
แทนที่จะประกอบค่าเข้าไปใน string คำสั่งเอง เราส่งค่านั้นแยกต่างหากผ่าน **placeholder** แล้วให้
database driver เป็นผู้รับผิดชอบแทนที่ค่าลงไปอย่างปลอดภัย (การแทนที่นี้เกิดขึ้น **หลัง** จากที่คำสั่ง
SQL ถูก parse เป็นโครงสร้างเรียบร้อยแล้ว ค่าที่ใส่เข้าไปจึงไม่มีทางเปลี่ยนโครงสร้างคำสั่งได้อีกเลย ไม่ว่า
จะมีอักขระพิเศษอะไรอยู่ในนั้นก็ตาม)

```ruby
# ปลอดภัย: ใช้ placeholder ?
User.where("email = ?", params[:email])

# ปลอดภัย: ใช้ named placeholder
User.where("email = :email", email: params[:email])

# ปลอดภัยที่สุดและอ่านง่ายที่สุด: hash condition (ActiveRecord แปลงเป็น placeholder ให้อัตโนมัติ)
User.where(email: params[:email])
```

ทั้งสามแบบทำงานเหมือนกันภายใน — ActiveRecord จะไม่มีวันเอาค่า `params[:email]` ไปต่อ string ตรงๆ
เด็ดขาด แบบที่ 3 (hash condition) เป็นแบบที่แนะนำที่สุดในกรณีเทียบค่าปกติ เพราะไม่มีโอกาสเขียนผิด
พลาดเลย (ไม่มี string SQL ให้เขียนเองด้วยซ้ำ)

### พิสูจน์ว่าการป้องกันนี้ทำงานจริง — ทดสอบด้วย Rails 8.1 app จริง

เราจะพิสูจน์ด้วยข้อมูลที่ **ไม่ใช่การโจมตี** เลยแม้แต่น้อย แค่ **ชื่อคนธรรมดาที่มีเครื่องหมาย
apostrophe** (เช่น `O'Brien`) ซึ่งเป็นข้อมูลจริงที่พบได้ทั่วไปในระบบ — ข้อมูลปกติชนิดนี้เพียงพอแล้วที่
จะพิสูจน์ให้เห็นว่าทำไมการต่อ string เข้า SQL ถึงเป็นปัญหาเชิงโครงสร้าง ไม่ใช่แค่เรื่องของ "คนร้าย":

```ruby
name = "O'Brien"
Person.create!(name: name, bio: "test")
```

**เวอร์ชันอันตราย (string interpolation) กับชื่อธรรมดานี้:**

```ruby
sql = "name = '#{name}'"
Person.where(sql).to_sql
# => SELECT "people".* FROM "people" WHERE (name = 'O'Brien')
#                                                       ^ apostrophe ของชื่อไปปิด string literal
#                                                         ก่อนกำหนด ทำให้ syntax พังทันที

Person.where(sql).count
# => ActiveRecord::StatementInvalid: SQLite3::SQLException: near "Brien": syntax error:
#    SELECT COUNT(*) FROM "people" WHERE (name = 'O'Brien') /*application='OwaspDemo'*/
```

**แค่ผู้ใช้ธรรมดาชื่อ O'Brien กรอกฟอร์มตามปกติ ระบบก็พังทันทีด้วย SQL syntax error** — นี่คือ
หลักฐานที่ชัดเจนที่สุดว่าการต่อ string ไม่ใช่แค่ "เสี่ยงถ้าโดนโจมตี" แต่ **ผิดหลักการตั้งแต่ต้น**
เพราะฐานข้อมูลตีความ apostrophe ของชื่อเป็นส่วนหนึ่งของ syntax คำสั่งไปแล้ว (จุดนี้เองที่เปิดช่องให้
ผู้โจมตีที่ตั้งใจร้ายใส่อักขระแบบนี้ **โดยจงใจ** เพื่อเปลี่ยนโครงสร้างคำสั่งตามที่ต้องการต่อไปได้)

**เวอร์ชันปลอดภัย (parameterized) กับชื่อเดียวกัน:**

```ruby
result = Person.where("name = ?", name)
result.to_sql
# => SELECT "people".* FROM "people" WHERE (name = 'O''Brien')
#                                                    ^^ ActiveRecord/driver escape apostrophe
#                                                       ให้อัตโนมัติ (double ตัว ' ตามมาตรฐาน SQL)

result.count   # => 1
result.first.name  # => "O'Brien"  -- ได้ค่าที่ถูกต้องเป๊ะ ไม่มี syntax error ใดๆ
```

ผลลัพธ์จริงจากการรันทดสอบ (Rails 8.1.4 / Ruby 3.3.6 / SQLite adapter — พฤติกรรมนี้เหมือนกันบน
PostgreSQL/MySQL เพราะการ escape ค่าใน placeholder เป็นหน้าที่ของ ActiveRecord connection
adapter ไม่ใช่โค้ดเฉพาะ database ใดๆ):

```
--- naive string interpolation with a perfectly ordinary name containing an apostrophe ---
SQL generated: SELECT "people".* FROM "people" WHERE (name = 'O'Brien')
ERROR: ActiveRecord::StatementInvalid: SQLite3::SQLException: near "Brien": syntax error:
SELECT COUNT(*) FROM "people" WHERE (name = 'O'Brien') /*application='OwaspDemo'*/
                                               ^

--- parameterized query (safe) with the same name ---
SQL generated: SELECT "people".* FROM "people" WHERE (name = 'O''Brien')
Result count: 1
Found: O'Brien
```

เห็นได้ชัดว่า **placeholder ไม่ได้แค่ "ป้องกันการโจมตี" แต่ทำให้โปรแกรมถูกต้องตั้งแต่ต้น** — ค่าที่
ส่งเข้าไปถูกปฏิบัติเป็น **ข้อมูล (literal value)** เสมอ ไม่ว่าจะมีอักขระอะไรอยู่ข้างในก็ตาม ต่างจาก
การต่อ string ที่ปฏิบัติกับมันเป็น **โค้ด/syntax** โดยไม่ได้ตั้งใจ

### ทวนกฎจาก Part 034 และ Part 038

- **Part 034 Step 332** เตือนไว้แล้วว่า **"ห้ามใช้ string interpolation ตรงๆ กับค่าที่มาจาก user
  input เด็ดขาด"** สำหรับ array condition (`where("name = ?", ...)`) — Step นี้คือเหตุผลเบื้องลึก
  เต็มรูปแบบว่าทำไมกฎนั้นถึงเข้มงวดขนาดนี้
- **Part 038 Step 376** สอน pattern **allowlist** สำหรับกรณีที่ค่าจาก `params` ต้องกำหนด
  **โครงสร้างของ query** (เช่น ชื่อคอลัมน์ที่จะ sort, ทิศทางการเรียง) ไม่ใช่แค่ **ค่าของเงื่อนไข**
  ในกรณีนั้น placeholder ใช้ไม่ได้เลยเพราะชื่อคอลัมน์ไม่ใช่ "ค่า" — ต้องใช้ allowlist ที่ hardcode
  ไว้ในโค้ดแทน

### กฎสรุปที่ใช้ได้กับทุกสถานการณ์

| ค่าจาก `params` จะถูกใช้เป็น... | วิธีป้องกันที่ถูกต้อง |
|--------------------------------|------------------------|
| **ค่าเงื่อนไข** (เช่น ค่าที่เทียบใน `WHERE`, ค่าที่จะ `INSERT`/`UPDATE`) | Placeholder (`?`, `:name`) หรือ hash condition — ActiveRecord escape ให้อัตโนมัติ |
| **โครงสร้าง query** (ชื่อคอลัมน์, ทิศทาง sort, ชื่อ table, ชื่อ scope ที่จะเรียกแบบ dynamic) | **Allowlist** ที่เป็น constant ในโค้ด (Part 038 Step 376) — ไม่มีทางหลุด เพราะเป็น element ของ array คงที่ |

> **หมายเหตุเรื่อง `LIKE`:** เมื่อสร้างเงื่อนไขค้นหาแบบ `LIKE`, wildcard character (`%`, `_`) ที่มากับ
> ค่าของผู้ใช้ (เช่น ผู้ใช้พิมพ์ `%` เข้ามาเอง) จะถูกตีความเป็น wildcard จริงแม้จะผ่าน placeholder แล้ว
> ก็ตาม (เพราะ placeholder ป้องกันแค่ SQL injection ไม่ได้ป้องกันความหมายพิเศษของ wildcard ใน
> `LIKE`) กรณีนี้ Rails มี helper `ActiveRecord::Base.sanitize_sql_like` ให้ escape wildcard เหล่านี้
> แยกต่างหากเมื่อจำเป็น — เป็นคนละปัญหากับ SQL Injection แต่เกี่ยวข้องกันในเรื่องความปลอดภัยของ
> query ที่รับ input จากผู้ใช้

---

## Step 784: Cross-Site Scripting (XSS) คืออะไร — Stored, Reflected, DOM-based

### XSS ต่างจาก SQL Injection ตรงไหน

SQL Injection โจมตี **ฐานข้อมูลฝั่ง server** — เหยื่อคือระบบเอง ส่วน **Cross-Site Scripting (XSS)**
โจมตี **เบราว์เซอร์ของผู้ใช้คนอื่นที่มาดูหน้าเว็บ** — เหยื่อคือผู้ใช้อีกคนหนึ่ง ไม่ใช่ server กลไกคือ:
ถ้าแอปพลิเคชันนำข้อมูลที่ผู้ใช้ควบคุมได้ไปแสดงผลเป็น **HTML/JavaScript ที่รันได้จริง** โดยไม่ผ่านการ
escape ที่เหมาะสม ผู้โจมตีสามารถฝัง JavaScript ของตัวเองลงไป แล้วโค้ดนั้นจะถูก **รันจริงในเบราว์เซอร์
ของผู้ใช้คนอื่นที่มาเปิดหน้านั้น** — เสมือนผู้โจมตี "ยืมมือ" เบราว์เซอร์ของเหยื่อรันโค้ดให้

### 3 ประเภทของ XSS

**1. Stored (Persistent) XSS — อันตรายที่สุด**

เนื้อหาที่เป็นอันตรายถูก**บันทึกลงฐานข้อมูล** (เช่น ในช่อง comment, bio, ชื่อสินค้าที่ผู้ขายกรอกเอง)
แล้วถูกแสดงผลแบบไม่ escape ให้กับ **ทุกคน** ที่เข้ามาดูหน้านั้นในภายหลัง — อันตรายที่สุดเพราะ
ผู้โจมตีโพสต์แค่ครั้งเดียว แต่ผู้ใช้ทุกคนที่มาดูหน้านั้นตกเป็นเหยื่อโดยอัตโนมัติ ไม่ต้องมีการกระทำ
พิเศษใดๆ จากฝั่งเหยื่อเลยนอกจากแค่เปิดหน้าเว็บตามปกติ

**2. Reflected XSS**

เนื้อหาที่เป็นอันตรายมาจาก **request ปัจจุบันเอง** (เช่น query parameter ที่ถูกนำไปแสดงในข้อความ
"ผลการค้นหาสำหรับ: ...") และแสดงผลกลับทันทีในหน้าเดียวกัน ไม่ได้ถูกบันทึกไว้ถาวร — ต้องอาศัยการ
**หลอกให้เหยื่อคลิกลิงก์ที่มี payload ฝังอยู่แล้ว** (เช่น ส่งลิงก์ทาง email หรือฝังไว้ในหน้าเว็บอื่น)
เหยื่อต้องคลิกลิงก์นั้นเองถึงจะโดน ต่างจาก Stored ที่ทุกคนโดนอัตโนมัติ

**3. DOM-based XSS**

ช่องโหว่อยู่**ทั้งหมดฝั่ง client-side JavaScript** — โค้ด JS อ่านค่าจากแหล่งที่ผู้ใช้ควบคุมได้ (เช่น
`location.hash`, `document.referrer`, หรือค่าที่ได้จาก JS ฝั่งอื่น) แล้วเขียนค่านั้นลงไปใน DOM โดยตรง
(เช่นผ่าน `innerHTML`) **โดยไม่เคยผ่าน server เลยแม้แต่น้อย** — เป็นประเภทที่สำคัญมากสำหรับแอป
Rails ยุคใหม่ที่มี Stimulus/JavaScript ฝั่ง client ค่อนข้างเยอะ (Phase 7) เพราะ **การ escape ฝั่ง
server (ERB) ช่วยอะไรไม่ได้เลยกับช่องโหว่ประเภทนี้** — ต้องระวังตอนเขียน JavaScript เองโดยตรง
โดยเฉพาะทุกจุดที่ใช้ `innerHTML =`, `document.write()`, หรือ jQuery `.html()` กับค่าที่มาจากแหล่งที่
ผู้ใช้ควบคุมได้

### สิ่งที่ XSS ทำได้เมื่อโจมตีสำเร็จ (อธิบายเชิงแนวคิด)

เมื่อ JavaScript ของผู้โจมตีรันสำเร็จใน**เบราว์เซอร์ของเหยื่อ ภายใต้ session ที่เหยื่อ login อยู่แล้ว**
โค้ดนั้นจะได้รับสิทธิ์เท่ากับที่เหยื่อมีทุกอย่างในหน้านั้น ซึ่งเปิดช่องให้:

- **อ่าน cookie/token ของ session เหยื่อ** แล้วส่งออกไปยัง server ของผู้โจมตี (เพื่อนำไปใช้ปลอมตัว
  เป็นเหยื่อในภายหลัง — เรียกว่า session hijacking)
- **ยิง request ที่มีสิทธิ์เท่าเหยื่อไปยัง server ของแอปเอง** โดยเหยื่อไม่รู้ตัว เช่น เปลี่ยนอีเมล/
  รหัสผ่านของเหยื่อ, โพสต์เนื้อหาในนามเหยื่อ, หรือโอนเงิน/สั่งซื้อในนามเหยื่อ (ทำได้เพราะ JavaScript
  ที่รันอยู่ในหน้านั้นมีสิทธิ์เท่ากับที่เหยื่อ login ไว้ทุกประการ)
- **เปลี่ยนหน้าตา/เนื้อหาของหน้าเว็บที่เหยื่อเห็น (defacement)** เช่น แสดงฟอร์ม login ปลอมทับของจริง
  เพื่อหลอกเอา credential
- **redirect เหยื่อไปหน้า phishing** ที่ดูเหมือนเว็บจริงทุกประการ

จุดสำคัญคือ **XSS ไม่ได้แค่ "ทำให้เว็บดูแปลก" แต่มันคือการรันโค้ดตามใจผู้โจมตีในบริบทที่เหยื่อ
เชื่อถืออยู่แล้ว** — อันตรายมากกว่าที่คนทั่วไปคิด

### ทวนคำเตือนจาก Part 024

Part 024 Step 240 เตือนไว้แล้วว่า `html_safe`/`raw` เป็นการ "สัญญา" กับ Rails ว่า string นั้น
ปลอดภัยแน่นอน ทั้งที่ Rails ไม่ได้ตรวจสอบอะไรให้เลย และถ้าเรียกกับ string ที่มาจากผู้ใช้โดยตรงโดยไม่
ผ่าน sanitize ก่อน จะเปิดช่องให้เกิด Stored XSS ทันที — Step 785 ถัดไปจะพิสูจน์เรื่องนี้ด้วยการทดสอบ
จริงในแอป Rails 8.1

---

## Step 785: ป้องกัน XSS ใน Rails — Auto-Escaping, `sanitize` (พิสูจน์การป้องกันด้วยของจริง)

### แนวป้องกันหลัก: Auto-Escaping ที่เป็น Default

Rails ป้องกัน XSS ขั้นพื้นฐานที่สุดให้ **ฟรีโดยอัตโนมัติ** ตั้งแต่ Rails 3 เป็นต้นมา: `<%= %>` จะ
escape อักขระพิเศษของ HTML (`<`, `>`, `&`, `"`, `'`) ให้เป็น HTML entity เสมอ ไม่ว่าค่าที่ส่งเข้ามา
จะเป็นอะไรก็ตาม — Part 024 สอนหลักการนี้ไปแล้ว ในที่นี้เราจะ **พิสูจน์ด้วย automated test จริงใน
Rails 8.1 app** ว่ากลไกนี้ทำงานตามที่อธิบายจริงๆ

### ทดสอบจริง: view และ controller ตัวอย่าง

```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def show
    @untrusted_body = params[:body] || "<strong>ทดสอบ</strong> escaping <em>อัตโนมัติ</em>"
  end
end
```

```erb
<%# app/views/comments/show.html.erb %>
<p id="auto-escaped"><%= @untrusted_body %></p>
<p id="raw-unsafe"><%= raw @untrusted_body %></p>
<p id="sanitized"><%= sanitize(@untrusted_body, tags: %w[strong em]) %></p>
```

Integration test ที่ส่งค่าที่มีทั้ง tag ปกติ (`<strong>`) และ attribute ที่เป็น event handler
(`onerror`) ซึ่งเป็นรูปแบบ vector ของ XSS จริงในโลกจริง (ไม่ใช่แค่ `<script>` เท่านั้นที่รันโค้ดได้ —
attribute อย่าง `onerror`/`onload`/`onclick` ก็รันโค้ดได้เช่นกันถ้าอยู่ใน HTML ที่ browser ตีความว่า
เป็น markup จริง ไม่ใช่ text ธรรมดา):

```ruby
# test/controllers/comments_controller_test.rb
require "test_helper"

class CommentsControllerTest < ActionDispatch::IntegrationTest
  test "escapes HTML by default with <%= %>" do
    get comments_show_url(body: '<strong>hi</strong><img src="x" onerror="x">')
    assert_response :success
    body = @response.body

    escaped_block = body[/<p id="auto-escaped">(.*?)<\/p>/m, 1]
    assert_includes escaped_block, "&lt;strong&gt;"
    assert_includes escaped_block, "&lt;img"
    assert_not_includes escaped_block, "<strong>"

    raw_block = body[/<p id="raw-unsafe">(.*?)<\/p>/m, 1]
    assert_includes raw_block, "<strong>hi</strong>"
    assert_includes raw_block, "<img"

    sanitized_block = body[/<p id="sanitized">(.*?)<\/p>/m, 1]
    assert_includes sanitized_block, "<strong>hi</strong>"
    assert_not_includes sanitized_block, "<img"
  end
end
```

ผลลัพธ์จริงจากการรัน (`bin/rails test`):

```
auto-escaped output: &lt;strong&gt;hi&lt;/strong&gt;&lt;img src=&quot;x&quot; onerror=&quot;x&quot;&gt;
raw/html_safe output: <strong>hi</strong><img src="x" onerror="x">
sanitize(tags: %w[strong em]) output: <strong>hi</strong>

1 runs, 15 assertions, 0 failures, 0 errors, 0 skips
```

สังเกตความแตกต่างของ HTML ที่ render ออกมาจริงทั้งสามบรรทัด:

1. **`<%= @untrusted_body %>` (auto-escaped):** ทุก tag ถูกแปลงเป็น HTML entity หมด
   (`&lt;strong&gt;`, `&lt;img`) — browser จะแสดงเป็น **ตัวหนังสือธรรมดา** ทั้งก้อน ไม่มีทางที่
   `onerror` จะถูกตีความเป็น attribute จริง หรือ `<strong>` จะกลายเป็น bold ได้เลย นี่คือค่า default
   ที่ปลอดภัยที่สุด
2. **`raw @untrusted_body` (ไม่ escape):** HTML ผ่านไปตรงๆ ทั้งก้อน ทั้ง `<strong>` และ
   `<img onerror>` จะถูก browser ตีความเป็น markup จริง — ถ้าค่านี้มาจากผู้ใช้จริงโดยไม่ผ่านการ
   ตรวจสอบก่อน นี่คือจุดที่ XSS เกิดขึ้นจริง (การทดสอบนี้เป็นแค่การตรวจสอบ string ที่ฝั่ง server
   เพื่อพิสูจน์ความแตกต่างของพฤติกรรม escape ไม่ใช่การรันหน้าเว็บจริงในเบราว์เซอร์)
3. **`sanitize(..., tags: %w[strong em])`:** เก็บเฉพาะ tag ที่อยู่ใน allowlist (`<strong>`) ไว้ และ
   **ตัด `<img onerror>` ออกไปทั้งหมด** เพราะไม่ได้อยู่ใน allowlist — ได้ผลลัพธ์ที่ยังมี formatting
   พื้นฐานแต่ปลอดภัยจากอักขระ/tag ที่ไม่ได้อนุญาต

### `sanitize` — ทางเลือกเมื่อต้องแสดง HTML ที่มาจากผู้ใช้บางส่วน

บางฟีเจอร์จำเป็นต้องให้ผู้ใช้ใส่ HTML ได้จริง (เช่น rich text editor สำหรับเขียนบทความ) กรณีนี้ใช้
`sanitize` ซึ่งใช้ Rails::Html::Sanitizer (สร้างบน Loofah/Nokogiri) กรอง tag/attribute อันตรายออก
โดยเก็บเฉพาะที่อยู่ใน allowlist:

```erb
<%# ใช้ allowlist default ของ Rails (p, strong, em, a, ul, li, ...) %>
<%= sanitize(@post.body) %>

<%# กำหนด allowlist เอง จำกัดให้แคบกว่า default %>
<%= sanitize(@comment.body, tags: %w[strong em a], attributes: %w[href]) %>
```

`sanitize` จะตัด tag อย่าง `<script>`, `<iframe>` และ attribute อย่าง `onclick=`, `onerror=`,
`javascript:` ใน `href` ออกให้อัตโนมัติ แม้จะพยายามซ่อนอยู่ในรูปแบบที่ผิดปกติ (เช่น case แปลกๆ,
whitespace แทรก) เพราะมันทำงานบน parsed HTML tree ไม่ใช่ regex ผิวเผิน

### ข้อควรระวังเรื่อง JSON ที่ฝังใน inline `<script>`

อีกจุดที่มักถูกมองข้าม: การ interpolate ข้อมูลลงใน inline `<script>` tag โดยตรงมีความเสี่ยงคล้ายกัน
แม้จะดูเหมือนเป็น "แค่ JSON" ก็ตาม:

```erb
<%# เสี่ยง: ถ้า @user.name มีอักขระ </script> อยู่ในนั้น จะปิด script tag ก่อนกำหนด %>
<script>
  const user = <%= raw @user.to_json %>;
</script>
```

วิธีที่ปลอดภัยกว่าคือใช้ helper `j` (alias ของ `escape_javascript`) หรือ **หลีกเลี่ยงการฝังข้อมูลใน
inline script โดยสิ้นเชิง** แล้วส่งผ่าน `data-*` attribute ที่ auto-escape อยู่แล้วแทน:

```erb
<%# ปลอดภัยกว่า: ส่งผ่าน data attribute ที่ escape ให้อัตโนมัติ %>
<div data-controller="user" data-user-value="<%= @user.to_json %>"></div>
```

### หลักปฏิบัติสรุป

> **Escaping คือพฤติกรรม default และถูกต้องเสมอ** ทุกครั้งที่เขียน `raw(...)`, `.html_safe`, หรือ
> `<%== %>` คือการ **ปิดการป้องกันด้วยมือตัวเอง** อย่างจงใจ — ต้องมั่นใจ 100% ว่าเนื้อหานั้นมาจาก
> แหล่งที่เชื่อถือได้ (HTML ที่แอปสร้างเอง) หรือผ่าน `sanitize` มาแล้วเท่านั้น ห้ามใช้กับ input ดิบจาก
> ผู้ใช้เด็ดขาดไม่ว่ากรณีใด

---

## Step 786: Content Security Policy (CSP) — Defense-in-Depth ชั้นที่สองเมื่อ Escaping พลาด

### CSP คืออะไร

**Content Security Policy (CSP)** เป็น HTTP response header (`Content-Security-Policy`) ที่บอก
**เบราว์เซอร์เอง** ว่าอนุญาตให้โหลด/รัน resource ประเภทไหน (script, style, image, font, ฯลฯ) จาก
แหล่งใดได้บ้าง — เป็นแนวป้องกันชั้นที่สอง (**defense-in-depth**) ที่ทำงาน **แม้ escaping จะพลาดไป
แล้วก็ตาม**: สมมติมีบั๊กที่ escaping หลุดจริงๆ จนผู้โจมตีสามารถแทรก `<script src="https://evil.com/x.js">`
เข้าไปในหน้าเว็บได้สำเร็จ ถ้า CSP ถูกตั้งไว้อย่างเข้มงวดว่า script ต้องโหลดจาก origin ของตัวเอง
เท่านั้น (`script-src 'self'`) เบราว์เซอร์จะ **ปฏิเสธไม่โหลด script จาก `evil.com` เลย** ทำให้การ
โจมตีล้มเหลวแม้ escaping จะพลาดไปแล้ว — นี่คือเหตุผลที่เรียกว่า defense-in-depth: ไม่ได้แทนที่การ
escape/sanitize ที่ถูกต้อง แต่เป็นเกราะสำรองอีกชั้นในกรณีที่ชั้นแรกพลาด

### ตั้งค่า CSP ใน Rails

Rails มี DSL สำหรับตั้งค่า CSP มาให้ในตัวที่ `config/initializers/content_security_policy.rb`:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.font_src    :self, :data
    policy.img_src     :self, :data, "https://res.cloudinary.com"
    policy.object_src  :none
    policy.script_src  :self
    policy.style_src   :self, :unsafe_inline # หลีกเลี่ยง unsafe_inline ถ้าทำได้ (ดูเรื่อง nonce ด้านล่าง)
    policy.connect_src :self, "https://api.example.com"

    # ระบุ URI สำหรับส่ง report เมื่อ policy ถูก violate (เอาไว้ debug/monitor)
    policy.report_uri "/csp-violation-report"
  end

  # เปิดโหมด "รายงานอย่างเดียว" ก่อน ใช้ตอนเริ่ม rollout CSP กับระบบเดิมที่มีอยู่แล้ว
  # (ยังไม่บล็อกจริง แค่ log ว่าอะไรจะถูกบล็อกถ้าเปิดใช้งานจริง — ช่วยหา false positive ก่อน)
  config.content_security_policy_report_only = true # เปลี่ยนเป็น false เมื่อมั่นใจแล้ว
end
```

### Nonce — อนุญาต inline script แบบเจาะจง โดยไม่ต้องเปิด `unsafe-inline` ทั้งหมด

การตั้ง `script-src 'self'` แบบเข้มงวดจะบล็อก inline `<script>` ทุกตัวไปด้วย (รวมถึงของแอปเอง) ถ้า
จำเป็นต้องมี inline script จริงๆ Rails รองรับการใส่ **nonce** (ค่าสุ่มที่สร้างใหม่ทุก request) เพื่อ
อนุญาตเฉพาะ script ที่มี nonce ตรงกันเท่านั้น:

```ruby
# config/initializers/content_security_policy.rb
config.content_security_policy do |policy|
  policy.script_src :self
end
config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
config.content_security_policy_nonce_directives = %w[script-src]
```

```erb
<%= javascript_tag nonce: true do %>
  console.log("inline script ที่ได้รับอนุญาตเพราะมี nonce ตรงกับ header");
<% end %>
```

Script ที่ผู้โจมตีแทรกเข้ามาจะไม่มีทางรู้ค่า nonce ที่ถูกสร้างใหม่ทุก request ล่วงหน้าได้เลย จึงไม่
สามารถผ่านเงื่อนไขนี้ได้ ไม่ว่าจะพยายามฝัง inline script ด้วยวิธีใดก็ตาม

> **ย้ำหลักการ:** CSP เป็น **defense-in-depth ไม่ใช่ตัวป้องกันหลัก** — หน้าที่หลักในการป้องกัน XSS
> ยังคงเป็น auto-escaping และ `sanitize` ที่สอนใน Step 785 เสมอ CSP มีไว้เผื่อกรณีที่ชั้นแรกพลาด
> ไม่ใช่ข้ออ้างให้ละเลยการ escape ข้อมูลอย่างถูกต้องตั้งแต่ต้น

---

## Step 787: Cross-Site Request Forgery (CSRF) คืออะไร — กลไกการโจมตีเชิงแนวคิด

### กลไกของการโจมตี

**Cross-Site Request Forgery (CSRF)** อาศัยพฤติกรรมพื้นฐานของเบราว์เซอร์ที่ว่า: **เบราว์เซอร์จะแนบ
cookie ของโดเมนหนึ่งไปกับทุก request ที่ส่งไปยังโดเมนนั้นโดยอัตโนมัติ ไม่ว่า request นั้นจะถูกสร้าง
ขึ้นจากหน้าเว็บของโดเมนไหนก็ตาม** — กลไกนี้เองที่ CSRF ใช้ประโยชน์:

1. เหยื่อ login เข้าเว็บเป้าหมาย (เช่น `bank.example`) อยู่แล้ว ทำให้เบราว์เซอร์มี session cookie ที่
   valid เก็บไว้
2. ผู้โจมตีหลอกให้เหยื่อเปิดหน้าเว็บอื่น (ของผู้โจมตีเอง หรือหน้าที่ถูกฝัง content ที่เป็นอันตราย) ซึ่ง
   มี form ที่ auto-submit หรือ element ที่ส่ง request ไปยัง `bank.example` โดยอัตโนมัติ (เช่น form ที่
   มี JavaScript สั่ง `submit()` ทันทีที่หน้าโหลดเสร็จ ชี้ไปยัง endpoint ที่ทำ action สำคัญ เช่น การโอน
   เงินหรือเปลี่ยนอีเมล)
3. เบราว์เซอร์ของเหยื่อส่ง request นั้นไปยัง `bank.example` **พร้อมแนบ session cookie ของเหยื่อไป
   ด้วยโดยอัตโนมัติ** เพราะ cookie นั้นเป็นของโดเมน `bank.example` (ไม่ใช่ของหน้าเว็บผู้โจมตี)
4. เซิร์ฟเวอร์ของ `bank.example` เห็น request ที่มี session cookie ที่ valid มาถึง จึง **ไม่มีทางแยก
   ได้เลยว่า request นี้มาจาก UI ของตัวเองจริงๆ หรือถูกสร้างขึ้นมาจากหน้าเว็บอื่นที่หลอกให้เหยื่อคลิก**
   — ถ้าไม่มีการตรวจสอบอย่างอื่นเพิ่มเติม server ก็จะทำ action นั้นให้เสมือนเป็นคำขอที่ถูกต้อง

### เงื่อนไขที่ทำให้ CSRF เกิดขึ้นได้

1. เว็บเป้าหมายใช้ **cookie-based session** สำหรับ authentication (เบราว์เซอร์แนบให้อัตโนมัติ)
2. มี **state-changing action** (เปลี่ยนแปลงข้อมูล เช่น โอนเงิน, เปลี่ยนรหัสผ่าน, ลบข้อมูล) ที่เข้าถึง
   ได้ผ่าน URL/params ที่คาดเดาได้ โดยไม่มี "ความลับเฉพาะ request นั้น" ที่ต้องรู้ล่วงหน้า
3. เบราว์เซอร์แนบ cookie ข้าม origin ให้อัตโนมัติตาม default (เว้นแต่จะถูกจำกัดด้วย `SameSite`
   attribute ซึ่งจะพูดถึงใน Step 789)

### ทำไม "แค่ login อยู่" ไม่พอ

ข้อสรุปสำคัญของ Step นี้คือ: **การที่ request มี session cookie ที่ valid มาด้วย ไม่ได้แปลว่า request
นั้นมีเจตนาจากผู้ใช้จริง** — server ต้องมีวิธีพิสูจน์เพิ่มเติมว่า **request นี้ถูกสร้างขึ้นจากหน้าเว็บ
ของตัวเองจริงๆ** ไม่ใช่ถูกแอบสร้างจากที่อื่นแล้วอาศัย cookie ที่เบราว์เซอร์แนบให้ฟรี — นี่คือสิ่งที่
`authenticity_token` ของ Rails ถูกออกแบบมาแก้โดยเฉพาะ ซึ่งจะสอนเต็มรูปแบบใน Step 788

---

## Step 788: Rails CSRF Protection — `protect_from_forgery` และ Authenticity Token (พิสูจน์ของจริง)

### กลไกป้องกัน: Synchronizer Token Pattern

Rails ป้องกัน CSRF ด้วยแนวคิดที่เรียกว่า **synchronizer token pattern**:

1. ทุกครั้งที่ Rails render หน้าที่มีฟอร์ม (ผ่าน `form_with`/`form_tag`) หรือ layout ที่มี
   `csrf_meta_tags` มันจะฝัง **`authenticity_token`** ซึ่งเป็นค่าที่ผูกกับ session ปัจจุบันลงไปด้วย
   (เป็น hidden field ในฟอร์ม หรือ `<meta name="csrf-token">` ใน layout สำหรับใช้กับ AJAX/`fetch`)
2. ทุกครั้งที่มี request แบบ **state-changing** (`POST`/`PUT`/`PATCH`/`DELETE` — ยกเว้น `GET`/`HEAD`
   ที่ถือว่า "ปลอดภัย" เพราะไม่ควรเปลี่ยนแปลงข้อมูล) `before_action` ที่ชื่อ
   `verify_authenticity_token` (ถูกเปิดใช้งานอัตโนมัติผ่าน `protect_from_forgery with: :exception`
   ซึ่ง Rails wire ให้ทุก `ActionController::Base` app โดย default ผ่านค่า config
   `action_controller.default_protect_from_forgery = true` — **ไม่ต้องเขียนเองใน
   `ApplicationController` เลยสำหรับแอปที่สร้างด้วย `rails new` แบบปกติ**) จะตรวจสอบว่า token ที่
   ส่งมากับ request นั้น**ตรงกับที่ session คาดหวังหรือไม่**
3. ถ้าไม่ตรง (ไม่มี token เลย, token ผิด, หรือ token มาจาก session อื่น) Rails จะเรียก
   `handle_unverified_request` ตามกลยุทธ์ที่กำหนด (default คือ `with: :exception` — raise
   `ActionController::InvalidAuthenticityToken` ซึ่งถูก rescue กลายเป็น **HTTP 422 Unprocessable
   Content** โดยอัตโนมัติในโหมดปกติ ไม่ปล่อยให้ request ทำงานต่อ)

**เหตุผลที่ผู้โจมตีปลอม token นี้ไม่ได้:** หน้าเว็บของผู้โจมตีไม่มีทางอ่านค่า `authenticity_token`
จากหน้าจริงของเป้าหมายได้เลย เพราะ **Same-Origin Policy ของเบราว์เซอร์ห้าม JavaScript จาก origin
หนึ่งอ่าน response (รวมถึง HTML/token ที่อยู่ในนั้น) ของอีก origin หนึ่ง** — ผู้โจมตีสร้างได้แค่
request ที่ **ไม่มี token ที่ถูกต้อง** เท่านั้น ซึ่ง Rails จะปฏิเสธทันที

### `form_with` ฝัง token ให้อัตโนมัติ

```erb
<%= form_with model: @post do |f| %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
```

HTML ที่ได้จะมี hidden field แบบนี้ฝังอยู่โดยอัตโนมัติ (ไม่ต้องเขียนเอง):

```html
<input type="hidden" name="authenticity_token" value="<ค่าที่ผูกกับ session, สร้างใหม่ทุก request>">
```

สำหรับ JavaScript/`fetch` ที่ไม่ได้ใช้ฟอร์ม ต้องดึง token จาก `<meta>` tag ใน layout
(`csrf_meta_tags`) แล้วแนบเป็น header `X-CSRF-Token` เอง — Rails UJS/Turbo จัดการเรื่องนี้ให้
อัตโนมัติอยู่แล้วเมื่อใช้ `data-turbo` หรือ `rails-ujs`

### พิสูจน์ว่ากลไกนี้ทำงานจริง — ทดสอบด้วย Rails 8.1 app จริง

ตั้งค่า controller/route ง่ายๆ:

```ruby
# app/controllers/people_controller.rb
class PeopleController < ApplicationController
  def new
  end

  def create
    Person.create!(name: params[:name])
    render plain: "created"
  end
end
```

```erb
<%# app/views/people/new.html.erb %>
<%= form_with url: people_create_path, method: :post do |f| %>
  <%= f.text_field :name %>
  <%= f.submit %>
<% end %>
```

> **หมายเหตุ:** Rails ปิด CSRF protection ไว้ใน `test` environment โดย default
> (`config.action_controller.allow_forgery_protection = false` ใน
> `config/environments/test.rb`) เพื่อความสะดวกในการเขียนเทสต์ทั่วไปที่ไม่อยากยุ่งกับ token —
> สำหรับการพิสูจน์นี้โดยเฉพาะ เราเปิดกลับมาเป็น `true` ชั่วคราวในแอปทดลอง เพื่อยืนยันว่ากลไกจริง
> ทำงานตามที่อธิบาย ไม่ใช่แค่ทฤษฎี

```ruby
# test/controllers/people_controller_test.rb
require "test_helper"

class PeopleControllerTest < ActionDispatch::IntegrationTest
  test "rejects POST with no authenticity token" do
    assert_no_difference "Person.count" do
      post people_create_url, params: { name: "Attacker-Controlled" }
    end
    assert_response :unprocessable_content
  end

  test "rejects POST with a wrong (forged) authenticity token" do
    assert_no_difference "Person.count" do
      post people_create_url,
           params: { name: "Attacker-Controlled", authenticity_token: "totally-forged-value" }
    end
    assert_response :unprocessable_content
  end

  test "accepts POST with the real authenticity token embedded by form_with" do
    get people_new_url
    assert_response :success
    valid_token = response.body[/name="authenticity_token" value="([^"]+)"/, 1]
    assert valid_token.present?

    assert_difference "Person.count", 1 do
      post people_create_url, params: { name: "Real User", authenticity_token: valid_token }
    end
    assert_response :success
  end
end
```

ผลลัพธ์จริงจากการรัน (`bin/rails test`):

```
POST ด้วย token ปลอม -> HTTP 422 (ถูก reject), Person.count = 2
POST ไม่มี authenticity_token -> HTTP 422 (ถูก reject), Person.count = 2

3 runs, 11 assertions, 0 failures, 0 errors, 0 skips
```

ผลลัพธ์ยืนยันชัดเจนทั้งสามกรณี:

1. **POST ไม่มี `authenticity_token` เลย** → ถูกปฏิเสธด้วย HTTP 422 ทันที ไม่มีการสร้าง record ใดๆ
2. **POST มี `authenticity_token` ที่เป็นค่าปลอม/สุ่มขึ้นมาเอง** → ถูกปฏิเสธด้วย HTTP 422 เช่นกัน
   (พิสูจน์ว่าไม่ใช่แค่ "เช็กว่ามี field นี้หรือเปล่า" แต่เช็ก **ความถูกต้องของค่า** จริงๆ)
3. **POST ที่มี token จริงซึ่งดึงมาจากฟอร์มที่ render จริงในเซสชันเดียวกัน** (จำลองพฤติกรรมผู้ใช้จริง
   ที่เปิดฟอร์มแล้วกดส่ง) → **สำเร็จ** สร้าง record ได้ตามปกติ

นี่คือหลักฐานที่ชัดเจนว่า mechanism ที่อธิบายไปใน Step ก่อนหน้าทำงานได้จริงตามทฤษฎีทุกประการ:
ไม่มี token ที่ถูกต้อง = ไม่มีทางทำ state-changing request สำเร็จ ไม่ว่าจะมี session cookie ที่
valid แนบมาด้วยหรือไม่ก็ตาม

### กลยุทธ์อื่นของ `protect_from_forgery`

```ruby
protect_from_forgery with: :exception      # default — raise error (422)
protect_from_forgery with: :null_session   # ล้าง session ทิ้งแทนการ raise (ใช้ได้กับบาง API)
protect_from_forgery with: :reset_session  # reset session ทั้งหมด (เข้มงวดที่สุด)
```

`:exception` เป็นตัวเลือกที่แนะนำที่สุดสำหรับแอปทั่วไป เพราะทำให้ปัญหา (เช่น token หมดอายุจริงๆ
จาก session timeout) แสดงผลเป็น error ที่ชัดเจนแทนที่จะทำงานแบบเงียบๆ ในสถานะที่ไม่คาดคิด

---

## Step 789: CSRF ใน API-only Apps — ทำไมจัดการต่างจากแอปแบบ Session ปกติ (ทวน Part 045)

### ทำไม token-based auth ไม่ต้องพึ่ง CSRF token แบบเดียวกัน

Part 045 สอนไปแล้วว่าแอป API มักใช้ **token-based authentication** (เช่น JWT) ที่ส่งผ่าน
`Authorization` header แทนการใช้ cookie-based session — ความแตกต่างนี้ส่งผลโดยตรงต่อความเสี่ยง
CSRF:

- **Cookie ถูกเบราว์เซอร์แนบไปกับ request โดยอัตโนมัติเสมอ** (ตามโดเมนที่เจ้าของ cookie กำหนด) —
  นี่คือรากของปัญหา CSRF ทั้งหมด เพราะเหยื่อไม่ต้อง "ทำอะไร" cookie ก็ถูกแนบให้เอง
- **`Authorization` header ไม่ถูกเบราว์เซอร์แนบให้อัตโนมัติข้าม origin** — หน้าเว็บของผู้โจมตี
  **ไม่มีทางรู้ค่า token ที่ถูกต้องของเหยื่อได้เลย** (token มักถูกเก็บใน JavaScript memory หรือ
  `localStorage` ของ origin เป้าหมายเท่านั้น ซึ่ง Same-Origin Policy ป้องกันไม่ให้ origin อื่นอ่านได้)
  แม้แต่การพยายามตั้งค่า custom header ข้าม origin ด้วย JavaScript ก็ยังถูกบล็อกด้วยกลไก CORS
  preflight request ตาม default

เพราะเหตุนี้ **ช่องทางการโจมตีแบบ CSRF คลาสสิก (อาศัยการที่เบราว์เซอร์แนบ credential ให้อัตโนมัติ)
จึงไม่สามารถทำงานได้กับ auth แบบ header-based** — `ActionController::API` (base class ที่ใช้เมื่อ
สร้างแอปด้วย `rails new --api`) **ไม่ได้ include module `RequestForgeryProtection` เลยด้วยซ้ำ**
เพราะไม่มีประโยชน์อะไรกับสถาปัตยกรรมแบบนี้ มีแต่จะเพิ่ม friction โดยไม่จำเป็น

### แต่ความเสี่ยงกลับมาทันทีถ้ายังใช้ cookie อยู่บางส่วน

ถ้าแอป API-only หรือแอปแบบ hybrid (เช่น SPA ที่เลือกใช้ cookie แทน token เพื่อความสะดวกในการ
จัดการ auth state) ยังคงพึ่งพา cookie ในส่วนใดส่วนหนึ่ง ความเสี่ยง CSRF จะกลับมาทันที และต้อง
จัดการอย่างชัดเจน — แนวทางหลักคือ **`SameSite` cookie attribute** (ทวนจาก Part 045):

```ruby
cookies[:session_token] = {
  value: token,
  httponly: true,
  secure: true,
  same_site: :strict # หรือ :lax — ไม่ส่ง cookie นี้ไปกับ cross-site request เลย ป้องกัน CSRF ไปในตัว
}
```

- `same_site: :strict` — เบราว์เซอร์จะไม่แนบ cookie นี้เลยถ้า request มาจาก origin อื่น (ป้องกัน
  CSRF ได้เกือบสมบูรณ์ แต่ก็แปลว่าลิงก์จากเว็บอื่นมาเปิดหน้าแอปครั้งแรกจะไม่มี session ติดมาด้วย)
- `same_site: :lax` — อนุญาตให้แนบ cookie กับ top-level navigation แบบ `GET` จากที่อื่นได้ (เช่น
  คลิกลิงก์ธรรมดา) แต่ยังบล็อก cross-site `POST` อยู่ — เป็นค่าที่สมดุลและเป็น default ของเบราว์เซอร์
  สมัยใหม่อยู่แล้ว
- `same_site: :none` — ปิดการป้องกันนี้ทั้งหมด (จำเป็นเฉพาะกรณีต้อง embed cross-site จริงๆ เช่น
  widget ที่ฝังในเว็บอื่น) **ต้องมาพร้อม CSRF protection แบบอื่นเสมอถ้าเลือกใช้ค่านี้**

### ตารางสรุปการตัดสินใจ

| กลไก Authentication | ต้องมี CSRF protection แบบ `authenticity_token` ไหม | สิ่งที่ต้องดูแลแทน |
|---------------------|------------------------------------------------------|---------------------|
| Session cookie (Rails full-stack app ปกติ) | **ต้องมี** (Rails เปิดให้อัตโนมัติแล้ว) | ตรวจสอบอย่าปิด `protect_from_forgery` โดยไม่ตั้งใจ |
| Token/JWT ใน `Authorization` header | ไม่จำเป็น | CORS config ให้ถูกต้อง, เก็บ token อย่างปลอดภัยฝั่ง client |
| Hybrid: cookie สำหรับ API/SPA | ต้องพิจารณาเป็นกรณีไป | `SameSite` attribute เป็นแนวป้องกันหลัก |

---

## Step 790: OWASP Top 10 หมวดอื่นในหลักสูตร + แบบฝึกหัดปิดท้าย Part

### ทวน Roadmap OWASP Top 10 ที่เหลือในหลักสูตร

ก่อนปิด Part นี้ ขอสรุปอีกครั้งว่าหมวดอื่นๆ ของ OWASP Top 10 ที่ยังไม่ได้เจาะลึกใน Part นี้ ถูก
ครอบคลุมอยู่ที่ไหนในหลักสูตร:

- **A01 Broken Access Control** — สอนไปแล้วอย่างละเอียดใน **Part 043 (Pundit policy/scope)** และ
  **Part 044 (CanCanCan, Role-Based Access Control)** — ถ้ายังไม่แม่นเรื่องนี้ ควรย้อนกลับไปทบทวน
  ก่อน เพราะเป็นหมวดอันดับ 1 ของ OWASP Top 10 ในแง่ความถี่ที่พบจริง
- **A02 Cryptographic Failures** — จะสอนใน **Part 081** (secrets management, ActiveRecord
  Encryption สำหรับเข้ารหัสข้อมูลอ่อนไหวในฐานข้อมูล)
- **A03 Injection** — Part นี้ (SQLi, XSS) — ครบแล้ว
- **A05 Security Misconfiguration** — **Part 080** ต่อจากนี้ทันที (mass assignment, secure HTTP
  headers, `rails-html-sanitizer` config, การตั้งค่า default ที่ปลอดภัย)
- **A06 Vulnerable and Outdated Components** — สอนไปแล้วใน **Part 075**: `Brakeman` สแกนโค้ดที่
  เราเขียนเอง (static analysis, ครอบคลุม SQL Injection, XSS, Mass Assignment และอื่นๆ อีกกว่า 70
  check) ส่วน `bundler-audit` สแกนเวอร์ชันของ gem ที่ระบุใน `Gemfile.lock` เทียบกับฐานข้อมูล
  ช่องโหว่ที่รู้จักแล้ว (Ruby Advisory Database) — ทั้งสองตัวรันอัตโนมัติใน CI pipeline ทุกครั้งที่มี
  การ push โค้ด
- **A07 Identification and Authentication Failures** — **Part 041, 042, 045** (authentication
  ทั้งหมดของหลักสูตร)
- **A09 Security Logging and Monitoring Failures** — **Part 078** (structured logging, error
  tracking ด้วย Sentry, uptime monitoring)
- **A04 Insecure Design, A08 Software and Data Integrity Failures, A10 SSRF** — ไม่มี Part เฉพาะ
  เพราะเป็นหลักการออกแบบที่แทรกอยู่ตลอดทั้งหลักสูตร โดยเฉพาะ Phase 14 (Architecture & Scaling) และ
  จะถูกรวบเป็นเช็กลิสต์ก่อนขึ้น production ใน **Part 081**

### แบบฝึกหัด: Code Review หาช่องโหว่และแก้ไข

**โจทย์:** ด้านล่างนี้คือโค้ดของฟีเจอร์ "ข้อความในบอร์ดสนทนา" ในแอปทดลองเล็กๆ ให้ทำ **code review**
โดยใช้กรอบความคิดที่เรียนไปใน Part นี้ (แยกแยะว่าค่าจาก `params`/user input ตัวไหนถูกใช้เป็น
"โครงสร้างคำสั่ง" หรือ "HTML ที่รันได้จริง" โดยไม่ผ่านการป้องกัน) ระบุว่าแต่ละจุดเสี่ยงต่อช่องโหว่
ประเภทใด (SQLi/XSS/CSRF) แล้วแก้ไขให้ปลอดภัย พร้อมอธิบายว่าจะ **ตรวจสอบ (verify) การแก้ไขแบบ
defensive อย่างไร** (ไม่ต้องรันการโจมตีจริงเพื่อพิสูจน์ — แค่ยืนยันว่ากลไกป้องกันทำงานถูกต้องก็พอ)

```ruby
# app/controllers/board_messages_controller.rb (โค้ดที่มีช่องโหว่ให้ตรวจสอบ)
class BoardMessagesController < ApplicationController
  skip_before_action :verify_authenticity_token, only: [:create]

  def index
    @messages = BoardMessage.where("author_name = '#{params[:author]}'") if params[:author].present?
    @messages ||= BoardMessage.order("#{params[:sort] || 'created_at'} desc")
  end

  def create
    BoardMessage.create!(author_name: params[:author_name], body: params[:body])
    redirect_to board_messages_path
  end
end
```

```erb
<%# app/views/board_messages/index.html.erb (โค้ดที่มีช่องโหว่ให้ตรวจสอบ) %>
<% @messages.each do |message| %>
  <div class="message">
    <strong><%= message.author_name %></strong>
    <p><%= message.body.html_safe %></p>
  </div>
<% end %>
```

### เฉลย

**จุดที่ 1 — SQL Injection ผ่าน `params[:author]` (Step 782–783):**

```ruby
# อันตราย
@messages = BoardMessage.where("author_name = '#{params[:author]}'") if params[:author].present?
```

`params[:author]` เป็นค่าเงื่อนไข (ไม่ใช่โครงสร้าง query) ถูกต่อ string เข้า `WHERE` ตรงๆ — เสี่ยง
SQL Injection เต็มรูปแบบ (ผู้ใช้ชื่อที่มี apostrophe ธรรมดาก็ทำให้ query พังได้แล้ว ทำนองเดียวกับที่
พิสูจน์ไว้ใน Step 783) **แก้ไข:**

```ruby
@messages = BoardMessage.where(author_name: params[:author]) if params[:author].present?
```

**วิธี verify แบบ defensive:** เขียนเทสต์สร้าง `BoardMessage` ที่มี `author_name` เป็นชื่อที่มี
apostrophe (เช่น `"O'Brien"`) แล้วยืนยันว่า `BoardMessagesController#index` กับ
`params[:author] = "O'Brien"` คืนผลลัพธ์ถูกต้องโดยไม่ raise error ใดๆ (เหมือนที่ทำใน Step 783)

**จุดที่ 2 — SQL Injection ผ่าน `params[:sort]` (Step 782–783, ทวน Part 038 Step 375–376):**

```ruby
# อันตราย
@messages ||= BoardMessage.order("#{params[:sort] || 'created_at'} desc")
```

`params[:sort]` กำหนด **โครงสร้าง query** (ชื่อคอลัมน์) ต้องใช้ **allowlist** ไม่ใช่ placeholder
**แก้ไข (ตาม pattern ของ Part 038 Step 376):**

```ruby
SORTABLE_COLUMNS = %w[created_at author_name].freeze

def index
  # ...
  @messages ||= BoardMessage.order(sort_column => :desc)
end

private

def sort_column
  SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
end
```

**วิธี verify แบบ defensive:** เขียนเทสต์ส่ง `params[:sort]` เป็นค่าที่ไม่อยู่ใน
`SORTABLE_COLUMNS` (เช่น ชื่อคอลัมน์ปลอมหรือค่าที่มีอักขระพิเศษ) แล้วยืนยันว่า controller **ไม่
raise error และ fallback ไปที่ `created_at` เสมอ** โดยไม่มีทางที่ค่านั้นจะถูกส่งเข้า `order` ตรงๆ ได้

**จุดที่ 3 — Stored XSS ผ่าน `message.body.html_safe` (Step 784–785, ทวน Part 024 Step 240):**

```erb
<%# อันตราย %>
<p><%= message.body.html_safe %></p>
```

`body` เป็นข้อความที่ผู้ใช้พิมพ์เองตอน `create` (ผ่าน `params[:body]` โดยตรง ไม่มีการกรองใดๆ เลย)
การเรียก `.html_safe` กับค่านี้เท่ากับ "รับรอง" กับ Rails ว่าเนื้อหานี้ปลอดภัย ทั้งที่มันมาจากผู้ใช้
โดยตรง 100% — เปิดช่องให้เกิด Stored XSS ทันที เพราะข้อความนี้ถูกบันทึกในฐานข้อมูลและแสดงให้ทุกคน
ที่มาดูบอร์ดเห็น **แก้ไข (เลือกได้ 2 ทาง ขึ้นกับว่าต้องการอนุญาต HTML บางส่วนหรือไม่):**

```erb
<%# ทางเลือกที่ 1: ไม่อนุญาต HTML เลย (escape ธรรมดา — ปลอดภัยที่สุด เหมาะกับบอร์ดข้อความทั่วไป) %>
<p><%= message.body %></p>

<%# ทางเลือกที่ 2: อนุญาต HTML บางส่วนแบบจำกัด (ถ้าต้องการให้ format ข้อความได้) %>
<p><%= sanitize(message.body, tags: %w[strong em a], attributes: %w[href]) %></p>
```

**วิธี verify แบบ defensive:** เขียน integration test ที่สร้าง `BoardMessage` โดยมี `body` เป็น
string ที่มี tag/attribute แปลกๆ (เช่น `<img src="x" onerror="x">`) แล้วยืนยันว่า HTML ที่ถูก
render ออกมาจริงไม่มี `<img` ปรากฏอยู่เลย (แบบเดียวกับเทสต์ที่พิสูจน์ไว้ใน Step 785)

**จุดที่ 4 — ปิด CSRF protection โดยไม่จำเป็น (Step 787–788):**

```ruby
# อันตราย
skip_before_action :verify_authenticity_token, only: [:create]
```

บรรทัดนี้ **ปิดการป้องกัน CSRF ของ Rails ด้วยมือ** สำหรับ action `create` ทั้งที่แอปนี้เป็นแอปแบบ
session ปกติ (ใช้ cookie, มีฟอร์มที่ render จาก view) ไม่มีเหตุผลอะไรที่ต้องปิดเลย — ทำให้ endpoint
นี้เปิดรับ CSRF attack เต็มรูปแบบ (หน้าเว็บอื่นสามารถ auto-submit ฟอร์มมาสร้างข้อความในนามเหยื่อได้
โดยไม่ต้องมี token ที่ถูกต้องเลย) **แก้ไข:** ลบบรรทัด `skip_before_action` ออกทั้งหมด แล้วให้ view
ที่มีฟอร์มสร้างข้อความใช้ `form_with` ตามปกติ (ซึ่งจะฝัง `authenticity_token` ให้อัตโนมัติอยู่แล้ว
ไม่ต้องทำอะไรเพิ่ม)

**วิธี verify แบบ defensive:** เขียนเทสต์แบบเดียวกับ Step 788 — ยืนยันว่า `POST` ไปยัง
`board_messages#create` **โดยไม่มี `authenticity_token`** ถูกปฏิเสธด้วย HTTP 422 และไม่มีการสร้าง
`BoardMessage` ใหม่เกิดขึ้น

**Controller/View ฉบับแก้ไขสมบูรณ์:**

```ruby
# app/controllers/board_messages_controller.rb (แก้ไขแล้ว)
class BoardMessagesController < ApplicationController
  SORTABLE_COLUMNS = %w[created_at author_name].freeze

  def index
    @messages = BoardMessage.where(author_name: params[:author]) if params[:author].present?
    @messages ||= BoardMessage.order(sort_column => :desc)
  end

  def create
    BoardMessage.create!(author_name: params[:author_name], body: params[:body])
    redirect_to board_messages_path
  end

  private

  def sort_column
    SORTABLE_COLUMNS.include?(params[:sort]) ? params[:sort] : "created_at"
  end
end
```

```erb
<%# app/views/board_messages/index.html.erb (แก้ไขแล้ว) %>
<% @messages.each do |message| %>
  <div class="message">
    <strong><%= message.author_name %></strong>
    <p><%= sanitize(message.body, tags: %w[strong em a], attributes: %w[href]) %></p>
  </div>
<% end %>
```

รวมทั้งหมดแล้วโค้ดชุดนี้มีช่องโหว่ครบทั้ง 3 คลาสหลักที่ Part นี้สอน (SQL Injection 2 จุด, XSS 1 จุด,
CSRF 1 จุด) ในไฟล์เดียวกัน — สะท้อนสถานการณ์จริงว่าช่องโหว่เหล่านี้มักไม่ได้มาทีละตัว แต่ซ่อนปนกัน
อยู่ในโค้ด "ธรรมดา" ที่ดูทำงานได้ปกติทุกประการถ้าไม่ได้ตั้งใจมองหา

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่มโมเดล `Product` ที่มี scope แบบ dynamic ที่เรียกชื่อ scope จาก `params[:filter]` โดยตรง
   (เช่น `Product.public_send(params[:filter])` ถ้า `params[:filter]` เป็น `"in_stock"` จะเรียก
   scope `in_stock` ที่นิยามไว้) วิเคราะห์ว่านี่เป็นช่องโหว่ประเภทเดียวกับ Step 782–783 หรือไม่
   (ใบ้: ไม่ใช่ SQL Injection ตรงๆ แต่เป็นปัญหาจากหลักการเดียวกันคือ "ปล่อยให้ `params` กำหนด
   โครงสร้างโค้ดที่จะรันโดยไม่ผ่าน allowlist") แล้วเขียน allowlist มาแก้ไข
2. เพิ่ม Content Security Policy ให้กับ layout ของแอปทดลองใน Step 790 (ใช้ config จาก Step 786)
   ตั้งให้ `script-src` อนุญาตเฉพาะ `:self` แล้วเขียน request spec ยืนยันว่า response header
   `Content-Security-Policy` มีค่า `script-src 'self'` ปรากฏอยู่จริง
3. เพิ่มฟีเจอร์ "แก้ไขข้อความของตัวเอง" ให้กับ `BoardMessage` (ผู้ใช้ระบุชื่อตัวเองตอนสร้างข้อความ
   ไม่มีระบบ login จริง) แล้ววิเคราะห์ว่าฟีเจอร์นี้เปิดช่องให้เกิดช่องโหว่ประเภท **Broken Access
   Control (A01)** อย่างไร (ใบ้: อะไรคือสิ่งที่ป้องกันไม่ให้คนอื่นแก้ไขข้อความที่ไม่ใช่ของตัวเองได้
   ในโค้ดปัจจุบัน) — ไม่ต้องแก้จริง แค่บอกว่าจะใช้เครื่องมือจาก Part ไหนของหลักสูตร (043/044) มาแก้

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า OWASP Top 10 คืออะไร ทำไมถึงเป็น checklist มาตรฐานที่ใช้ทั่วทั้งอุตสาหกรรม (compliance,
  pentest scope, เครื่องมืออย่าง Brakeman) และรู้ roadmap ว่าแต่ละหมวดถูกสอนที่ Part ไหนในหลักสูตร
- เข้าใจกลไกเบื้องหลัง **SQL Injection** อย่างลึกซึ้ง (ข้อมูลปนกับโครงสร้างคำสั่ง) ระบุ pattern
  อันตรายในโค้ด Rails ได้ และพิสูจน์ด้วยของจริงว่า parameterized query ป้องกันได้อย่างไร (แม้แต่กับ
  ข้อมูลปกติที่มี apostrophe)
- เข้าใจ **XSS** ทั้ง 3 ประเภท (Stored, Reflected, DOM-based) ผลกระทบเชิงแนวคิดของแต่ละแบบ และ
  พิสูจน์ด้วยของจริงว่า Rails auto-escape `<%= %>` อย่างไร พร้อมรู้วิธีใช้ `sanitize` อย่างถูกต้อง
- รู้จัก **Content Security Policy** ในฐานะ defense-in-depth ชั้นที่สอง และวิธีตั้งค่าใน Rails
  (รวมถึง nonce สำหรับ inline script)
- เข้าใจกลไกการโจมตีแบบ **CSRF** เชิงแนวคิด และพิสูจน์ด้วยของจริงว่า `protect_from_forgery` กับ
  authenticity token ของ Rails ป้องกันได้อย่างไร (request ที่ไม่มี/มี token ผิด ถูกปฏิเสธด้วย HTTP
  422 เสมอ)
- เข้าใจว่าทำไม API-only app ที่ใช้ token-based auth ถึงไม่ต้องพึ่ง CSRF token แบบเดิม และรู้ว่าเมื่อ
  ไหร่ที่ความเสี่ยงนี้กลับมา (เมื่อยังมี cookie เกี่ยวข้องอยู่)
- ฝึก code review หาช่องโหว่ทั้ง 3 คลาสในโค้ดชิ้นเดียวกัน และแก้ไขพร้อมวิธี verify แบบ defensive

**ต่อไป (Part 080):** เราจะเจาะลึก **A05 Security Misconfiguration** ต่อเนื่องจาก Part นี้ —
**Mass Assignment** (ช่องโหว่ที่เกิดจาก Strong Parameters ตั้งค่าไม่รัดกุม), **Secure HTTP
Headers** (`X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security` และอื่นๆ ที่
Rails ตั้งให้เป็น default บางส่วนแต่ควรเข้าใจว่าทำไม) และการอ่านผลสแกนของ **Brakeman** แบบเจาะลึก
กว่าที่เคยเห็นใน Part 075 (ครอบคลุม check เพิ่มเติมนอกเหนือจาก SQL Injection ที่เคยสาธิตไปแล้ว)
