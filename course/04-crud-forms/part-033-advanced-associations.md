# Part 033: Association ขั้นสูง — `has_many :through`, `has_and_belongs_to_many`, Polymorphic Association

> **Step ครอบคลุมใน Part นี้:** Step 321–330
> **ระดับ:** กลาง-สูง (ต้องผ่าน Part 027 เรื่อง `belongs_to`/`has_many`/`has_one`, `dependent:`,
> N+1 เบื้องต้น และ `inverse_of` มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x (ตัวอย่างทั้งหมดทดสอบจริงบน Ruby 3.3.6, Rails
> 8.1.4, ฐานข้อมูล SQLite3)

Part 027 พาไปรู้จัก association สามตัวที่ใช้บ่อยที่สุด: `belongs_to`, `has_many`, `has_one` —
ครอบคลุมความสัมพันธ์แบบ **หนึ่งต่อกลุ่ม** (one-to-many) และ **หนึ่งต่อหนึ่ง** (one-to-one) ทั้งหมด
พร้อม `dependent:`, ปัญหา N+1 เบื้องต้น และ `inverse_of` ไปแล้ว (Part นี้จะไม่พูดซ้ำเรื่องพวกนั้น
อีก ถ้าจำไม่ได้ให้กลับไปทบทวน Part 027 ก่อน)

แต่โลกจริงยังมีความสัมพันธ์อีกแบบที่ `belongs_to`/`has_many` เดี่ยวๆ **ทำไม่ได้เลย**: ความสัมพันธ์
แบบ **กลุ่มต่อกลุ่ม** (many-to-many) เช่น "นักเรียนหนึ่งคนลงเรียนได้หลายวิชา และวิชาหนึ่งวิชามีนักเรียน
ได้หลายคน" — Part นี้จะพาไปแก้ปัญหานี้ด้วยสองวิธี: `has_and_belongs_to_many` (habtm) และ
`has_many :through` (ทางเลือกที่แนะนำในยุคปัจจุบัน) พร้อมกับความสัมพันธ์พิเศษอีกสองแบบที่พบบ่อยใน
แอปจริง: **self-referential association** (Model อ้างอิงกลับไปหาตัวเอง เช่น ระบบ follow, ผังองค์กร)
และ **polymorphic association** (Model หนึ่งอ้างอิงไปยังได้หลาย Model โดยไม่ต้องมี foreign key
แยกทีละ column) ปิดท้ายด้วยตารางสรุปเปรียบเทียบว่าเมื่อไหร่ควรใช้เทคนิคไหน

## สารบัญของ Part นี้

- Step 321: ปัญหาความสัมพันธ์แบบกลุ่มต่อกลุ่ม (many-to-many) — ทำไม `belongs_to`/`has_many`
  เดี่ยวๆ ทำไม่ได้
- Step 322: `has_and_belongs_to_many` (habtm) — ตารางกลาง (join table) แบบไม่มี `id` ไม่มี
  column พิเศษ
- Step 323: `has_many :through` — ทางเลือกที่แนะนำ เพราะเพิ่ม column/validation/callback บน
  join model ได้
- Step 324: แปลง `has_and_belongs_to_many` เป็น `has_many :through` จริง (migration
  before/after พร้อม backfill ข้อมูล)
- Step 325: Self-referential association แบบ `has_many :through` — ระบบ follow (`User`
  ติดตามกันเองผ่าน `Follow`)
- Step 326: Self-referential association แบบ `belongs_to` — ผังองค์กร (`Employee belongs_to
  :manager`)
- Step 327: Polymorphic association เบื้องต้น — `Comment belongs_to :commentable,
  polymorphic: true`
- Step 328: ข้อเสียของ Polymorphic Association — ไม่มี foreign key constraint จริงในฐานข้อมูล
- Step 329: `has_one :through` — เดินความสัมพันธ์แบบหนึ่งต่อหนึ่งผ่านตัวกลาง
- Step 330: ตารางสรุปเปรียบเทียบ — habtm vs `has_many :through` vs polymorphic vs STI (เมื่อไหร่
  ควรใช้อะไร) + แบบฝึกหัด

---

## Step 321: ปัญหาความสัมพันธ์แบบกลุ่มต่อกลุ่ม (many-to-many) — ทำไม `belongs_to`/`has_many` เดี่ยวๆ ทำไม่ได้

ลองนึกภาพระบบลงทะเบียนเรียน: **นักเรียนหนึ่งคนลงเรียนได้หลายวิชา** และ **วิชาหนึ่งวิชามีนักเรียน
ลงทะเบียนได้หลายคน** — นี่คือความสัมพันธ์แบบ **many-to-many** ลองใช้ความรู้จาก Part 027
มาแก้ปัญหานี้ดูก่อนว่าทำไมถึงไปต่อไม่ได้

### ลองแบบที่ 1: เพิ่ม `course_id` ในตาราง `students`

```
students
+----+-------+-----------+
| id | name  | course_id |
+----+-------+-----------+
| 1  | Alice | 1         |
+----+-------+-----------+
```

วิธีนี้เทียบเท่ากับ `Student belongs_to :course` — ปัญหาคือ **นักเรียนหนึ่งคนมี `course_id` ได้แค่
ค่าเดียว** ถ้า Alice อยากลงเรียนทั้ง "Math 101" และ "Physics 101" พร้อมกัน ทำไม่ได้เลยเพราะ column
เดียวเก็บได้แค่ค่าเดียว

### ลองแบบที่ 2: เพิ่ม `student_id` ในตาราง `courses`

```
courses
+----+-----------+------------+
| id | title     | student_id |
+----+-----------+------------+
| 1  | Math 101  | 1          |
+----+-----------+------------+
```

วิธีนี้กลับกัน กลายเป็น `Course belongs_to :student` — ปัญหาเดียวกันแต่สลับฝั่ง: วิชาหนึ่งวิชามี
`student_id` ได้แค่ค่าเดียว ถ้า Math 101 มีนักเรียนลงทะเบียน 30 คน เก็บไม่ได้เลย

### ทางออก: ตารางที่สาม (Join Table) มาคั่นกลาง

ทั้งสองวิธีข้างบนพังเพราะพยายามยัด "ความสัมพันธ์แบบหลายต่อหลาย" ลงใน foreign key column เดียว
ซึ่งเก็บได้แค่ค่าเดียวเท่านั้น (คุณสมบัติพื้นฐานของ column ในฐานข้อมูลเชิงสัมพันธ์) — วิธีแก้มาตรฐานคือ
สร้าง **ตารางที่สามมาคั่นกลาง** (เรียกว่า **join table** หรือ **junction table**) ที่แต่ละแถวเก็บแค่
"คู่" ของ `student_id` กับ `course_id` หนึ่งคู่:

```
students          courses_students          courses
+----+-------+    +------------+-----------+    +----+-----------+
| id | name  |    | student_id | course_id |    | id | title     |
+----+-------+    +------------+-----------+    +----+-----------+
| 1  | Alice |    | 1          | 1         | -->| 1  | Math 101  |
| 2  | Bob   |    | 1          | 2         | -->| 2  | Physics 101|
+----+-------+    | 2          | 1         |    +----+-----------+
                  +------------+-----------+
```

Alice (id 1) มีสองแถวในตารางกลาง ชี้ไปทั้ง Math 101 และ Physics 101 พร้อมกันได้ — Bob (id 2) มีแค่
แถวเดียวชี้ไป Math 101 — **แต่ละแถวในตารางกลางไม่จำกัดจำนวน** ทำให้นักเรียนคนหนึ่งลงได้หลายวิชา และ
วิชาหนึ่งมีนักเรียนได้หลายคนพร้อมกันโดยไม่ชนกันเลย

ActiveRecord มีสองวิธีสร้างตารางกลางแบบนี้: **`has_and_belongs_to_many`** (Step 322) และ
**`has_many :through`** (Step 323) — ทั้งสองแก้ปัญหาเดียวกัน แต่ต่างกันตรงที่ตารางกลางมีหน้าตา
เป็น "แค่ตารางเชื่อม" ล้วนๆ หรือเป็น "Model เต็มรูปแบบที่เก็บข้อมูลเพิ่มได้ด้วย"

---

## Step 322: `has_and_belongs_to_many` (habtm) — ตารางกลาง (join table) แบบไม่มี `id` ไม่มี column พิเศษ

`has_and_belongs_to_many` (มักเรียกย่อว่า **habtm**) คือวิธีที่ตรงไปตรงมาที่สุดในการสร้าง
many-to-many — ประกาศที่ทั้งสอง Model โดยตรง ไม่ต้องมี Model ตัวที่สามมาคั่นกลาง

### สร้าง Model และตารางกลาง

```bash
bin/rails generate model Student name:string
bin/rails generate model Course title:string
```

จากนั้นสร้างตารางกลางด้วย generator migration เปล่าๆ:

```bash
bin/rails generate migration CreateCoursesStudentsJoinTable
```

> **ข้อควรระวัง:** ต่างจาก `references` ที่เจอใน Part 027 — migration ที่ได้จากคำสั่งนี้จะ
> **ว่างเปล่า** ไม่มีอะไรให้เลย ต้องเติมโค้ดสร้างตารางกลางเองด้วย method พิเศษชื่อ
> `create_join_table`:

```ruby
class CreateCoursesStudentsJoinTable < ActiveRecord::Migration[8.1]
  def change
  end
end
```

แก้ให้เป็น:

```ruby
class CreateCoursesStudentsJoinTable < ActiveRecord::Migration[8.1]
  def change
    create_join_table :courses, :students do |t|
      t.index [:course_id, :student_id]
      t.index [:student_id, :course_id]
    end
  end
end
```

```bash
bin/rails db:migrate
```

```
== CreateCoursesStudentsJoinTable: migrating =================================
-- create_join_table(:courses, :students)
   -> 0.0068s
== CreateCoursesStudentsJoinTable: migrated (0.0068s) =========================
```

### วิเคราะห์ `create_join_table`

`create_join_table :courses, :students` สร้างตารางชื่อ **`courses_students`** โดยอัตโนมัติ —
สังเกตว่าชื่อตารางเรียงตาม **ลำดับตัวอักษร** (`courses` มาก่อน `students` เพราะ `c` มาก่อน `s`)
ไม่ใช่ลำดับที่ใส่ argument เสมอไป (ถ้าเขียน `create_join_table :students, :courses` ก็ยังได้ตาราง
ชื่อ `courses_students` เหมือนเดิม) — นี่คือ convention ที่ต้องจำ ถ้าต้องการชื่อตารางอื่น ระบุเองได้
ด้วย `table_name:`:

```ruby
create_join_table :courses, :students, table_name: :enrollments_raw
```

ดู schema ที่ได้:

```ruby
create_table "courses_students", id: false, force: :cascade do |t|
  t.bigint "course_id", null: false
  t.bigint "student_id", null: false
  t.index ["course_id", "student_id"], name: "index_courses_students_on_course_id_and_student_id"
  t.index ["student_id", "course_id"], name: "index_courses_students_on_student_id_and_course_id"
end
```

จุดที่ต้องสังเกตให้ดี: **`id: false`** — ตารางกลางของ habtm **ไม่มี primary key `id`**
และ **มีแค่สอง column** (`course_id`, `student_id`) เท่านั้น ไม่มีที่ว่างให้เพิ่ม column อื่น เช่น
วันที่ลงทะเบียน หรือเกรดที่ได้ — พิสูจน์ได้ตรงๆ จาก console:

```irb
irb(main):001> ActiveRecord::Base.connection.columns(:courses_students).map(&:name)
=> ["course_id", "student_id"]
irb(main):002> ActiveRecord::Base.connection.primary_key("courses_students")
=> nil
```

### ประกาศ `has_and_belongs_to_many` ใน Model ทั้งสองฝั่ง

```ruby
# app/models/student.rb
class Student < ApplicationRecord
  has_and_belongs_to_many :courses
end
```

```ruby
# app/models/course.rb
class Course < ApplicationRecord
  has_and_belongs_to_many :students
end
```

**สังเกต:** ต้องประกาศ **ทั้งสองฝั่ง** เสมอ (ไม่มีแนวคิด `belongs_to` แยกฝั่งแบบ Part 027) และชื่อ
ที่ใช้เป็นพหูพจน์ทั้งคู่ เพราะทั้งสองฝั่งมีได้หลาย record

### ใช้งานจริง

```irb
irb(main):001> math = Course.create!(title: "Math 101")
irb(main):002> physics = Course.create!(title: "Physics 101")
irb(main):003> alice = Student.create!(name: "Alice")
irb(main):004> bob = Student.create!(name: "Bob")

irb(main):005> alice.courses << math
irb(main):006> alice.courses << physics
irb(main):007> bob.courses << math

irb(main):008> alice.courses.pluck(:title)
=> ["Math 101", "Physics 101"]
irb(main):009> math.students.pluck(:name)
=> ["Alice", "Bob"]
irb(main):010> bob.courses.pluck(:title)
=> ["Math 101"]
```

`alice.courses << math` ยิง `INSERT INTO courses_students (course_id, student_id) VALUES
(...)` เบื้องหลัง — เขียนได้เหมือน `has_many` ทุกประการ (`build`, `create`, `<<`, `ids=`
ที่เรียนมาใน Part 027 ใช้ได้หมด) ความแตกต่างอยู่ที่ **โครงสร้างตารางเบื้องหลัง** เท่านั้น

### ทำไม habtm ถึงไม่ค่อยแนะนำในยุคปัจจุบัน

ดูเผินๆ habtm สั้นและง่ายกว่า `has_many :through` มาก (ไม่ต้องสร้าง Model ที่สาม) แต่มีข้อจำกัด
ที่ทำให้ Rails community และ Rails Guide เองแนะนำให้ **เลี่ยง habtm ในโปรเจกต์ใหม่ส่วนใหญ่**:

1. **เพิ่ม column ข้อมูลอื่นบนความสัมพันธ์ไม่ได้** — ถ้าวันหนึ่งอยากรู้ว่า "นักเรียนคนนี้ลงทะเบียน
   วิชานี้เมื่อไหร่" หรือ "ได้เกรดอะไร" ตารางกลางแบบ habtm ไม่มีที่เก็บเลย (ไม่มี column, ไม่มี
   `id` ให้ query แยกเป็นแถวด้วยซ้ำ)
2. **ใส่ validation ตรงๆ บนแถวความสัมพันธ์ไม่ได้** — เพราะไม่มี Model ตัวเต็มที่ ActiveRecord
   มองว่าเป็น "record" จริงๆ (habtm รองรับแค่ callback ระดับ association เช่น
   `before_add`/`after_add` บน `has_and_belongs_to_many` เท่านั้น ไม่ใช่ validation/callback
   มาตรฐานแบบ `before_save` ที่ผูกกับแถวความสัมพันธ์แต่ละแถวได้)
3. **บังคับ uniqueness (ห้ามลงทะเบียนซ้ำวิชาเดิม) ได้แค่ระดับ unique index ในฐานข้อมูล**
   ไม่มี Model ให้เขียน `validates :xxx, uniqueness: true` เหมือนปกติ ต้องอาศัยจับ
   `ActiveRecord::RecordNotUnique` เอง ซึ่งจัดการใน controller ได้ไม่สวยงามเท่า
4. **แทบทุกโปรเจกต์จริงสุดท้ายต้องการข้อมูลเพิ่มเติมบนความสัมพันธ์เสมอ** — ต่อให้วันนี้ยังไม่ต้องการ
   แต่พอถึงวันที่ต้องการ (แทบจะแน่นอน) จะต้อง**ย้ายจาก habtm ไปเป็น `has_many :through`**
   อยู่ดี (Step 324 จะสอนวิธี migrate ข้อมูลจริง) ทำให้ทีมงานจำนวนมากเลือกเริ่มด้วย
   `has_many :through` ตั้งแต่แรกไปเลย ต่อให้ตอนนี้ดูเหมือนยังไม่จำเป็นก็ตาม

> **แนวปฏิบัติที่แนะนำ:** ใช้ `has_and_belongs_to_many` ได้เฉพาะกรณีที่มั่นใจจริงๆ ว่าความสัมพันธ์นี้
> จะเป็น "แค่ลิงก์เชื่อมสองสิ่งเข้าด้วยกัน" ตลอดไป ไม่มีทางต้องการข้อมูลอื่นเพิ่มบนความสัมพันธ์นั้นแน่ๆ
> เช่น `Article has_and_belongs_to_many :tags` แบบง่ายที่สุด — แต่ถ้าลังเลแม้แต่นิดเดียว **ให้เริ่ม
> ด้วย `has_many :through` ไปเลยตั้งแต่แรก** เพราะต้นทุนตอนเริ่มต้นสูงกว่าแค่เล็กน้อย (ต้องสร้าง
> Model เพิ่มอีกตัว) แต่ประหยัดเวลามหาศาลในระยะยาวเมื่อ requirement เปลี่ยน

---

## Step 323: `has_many :through` — ทางเลือกที่แนะนำ เพราะเพิ่ม column/validation/callback บน join model ได้

`has_many :through` แก้ปัญหาแบบเดียวกับ habtm (many-to-many) แต่ใช้ **Model จริงเต็มรูปแบบ** เป็น
ตัวกลางแทนตารางกลางเปล่าๆ — ลองสร้างระบบนัดหมายแพทย์: **หมอหนึ่งคนมีคนไข้ได้หลายคน และคนไข้หนึ่งคน
พบหมอได้หลายคน** เชื่อมกันผ่าน **การนัดหมาย (Appointment)** ซึ่งมีข้อมูลของตัวเอง (วันเวลานัด,
บันทึกการตรวจ)

### สร้าง Model ทั้งสาม

```bash
bin/rails generate model Doctor name:string specialty:string
bin/rails generate model Patient name:string
bin/rails generate model Appointment doctor:references patient:references appointment_date:datetime notes:text
bin/rails db:migrate
```

migration ของ `Appointment` ที่ได้ (เหมือน `belongs_to` สองอันธรรมดาตาม Part 027):

```ruby
class CreateAppointments < ActiveRecord::Migration[8.1]
  def change
    create_table :appointments do |t|
      t.references :doctor, null: false, foreign_key: true
      t.references :patient, null: false, foreign_key: true
      t.datetime :appointment_date
      t.text :notes

      t.timestamps
    end
  end
end
```

**สังเกตความต่างจาก habtm ตั้งแต่ตอนนี้:** `appointments` เป็นตารางปกติทุกประการ มี `id` เป็น
primary key และมี column เพิ่มเติม (`appointment_date`, `notes`) ที่ habtm ทำไม่ได้เลย

### ประกาศ association ทั้งสามฝั่ง

`Appointment` เป็นตัวกลาง เขียน `belongs_to` ปกติสองอัน (เหมือน Part 027 ทุกประการ) แถมยังใส่
validation/callback ได้เต็มที่เพราะเป็น Model จริง:

```ruby
# app/models/appointment.rb
class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient

  validates :appointment_date, presence: true
  validate :doctor_is_available, on: :create

  private

  def doctor_is_available
    return if doctor.blank? || appointment_date.blank?

    conflict = Appointment.where(doctor: doctor, appointment_date: appointment_date)
                          .where.not(id: id)
                          .exists?
    errors.add(:appointment_date, "หมอคนนี้มีนัดอื่นอยู่แล้วในเวลานี้") if conflict
  end
end
```

`Doctor` และ `Patient` ใช้ `has_many :through` แทน `has_and_belongs_to_many`:

```ruby
# app/models/doctor.rb
class Doctor < ApplicationRecord
  has_many :appointments, dependent: :destroy
  has_many :patients, through: :appointments
end
```

```ruby
# app/models/patient.rb
class Patient < ApplicationRecord
  has_many :appointments, dependent: :destroy
  has_many :doctors, through: :appointments
end
```

**อ่าน `has_many :patients, through: :appointments` เป็นภาษาธรรมดา:** "หมอคนหนึ่งมี `patients`
ได้หลายคน โดยเดินผ่าน (`through`) `appointments` ของหมอคนนั้น" — ActiveRecord จะไป join ตาราง
`appointments` แล้วดึง `patient` ที่ผูกอยู่กับแต่ละแถวโดยอัตโนมัติ ต้องมี `has_many :appointments`
เขียนไว้ก่อนเสมอ (`through:` ต้องชี้ไปยังชื่อ association ที่มีอยู่จริงในบรรทัดก่อนหน้า ไม่ใช่ชื่อ
Model ตรงๆ)

### ใช้งานจริง

```irb
irb(main):001> dr_somchai = Doctor.create!(name: "หมอสมชาย", specialty: "อายุรกรรม")
irb(main):002> dr_suda = Doctor.create!(name: "หมอสุดา", specialty: "กุมารเวช")
irb(main):003> patient_a = Patient.create!(name: "คนไข้ ก")
irb(main):004> patient_b = Patient.create!(name: "คนไข้ ข")

irb(main):005> slot = Time.zone.parse("2026-10-01 09:00")
irb(main):006> Appointment.create!(doctor: dr_somchai, patient: patient_a, appointment_date: slot, notes: "ตรวจทั่วไป")
irb(main):007> Appointment.create!(doctor: dr_somchai, patient: patient_b, appointment_date: slot + 1.hour, notes: "ติดตามอาการ")
irb(main):008> Appointment.create!(doctor: dr_suda, patient: patient_a, appointment_date: slot + 2.hours)

irb(main):009> dr_somchai.patients.pluck(:name)
=> ["คนไข้ ก", "คนไข้ ข"]
irb(main):010> patient_a.doctors.pluck(:name)
=> ["หมอสมชาย", "หมอสุดา"]
```

`dr_somchai.patients` **ไม่ใช่แค่ id สองคอลัมน์แบบ habtm** — เดินผ่าน `appointments` แล้วดึง
`Patient` เต็มรูปแบบมาให้ ใช้งานเหมือน `has_many` ปกติทุกประการ

### จุดเด่นที่ habtm ทำไม่ได้: ข้อมูลเพิ่มเติมบนความสัมพันธ์

```irb
irb(main):011> appt1 = dr_somchai.appointments.first
irb(main):012> appt1.notes
=> "ตรวจทั่วไป"
irb(main):013> appt1.appointment_date
=> Thu, 01 Oct 2026 09:00:00 UTC +00:00
```

`Appointment` เก็บ `notes` และ `appointment_date` ไว้ได้ตามปกติ — สิ่งที่ habtm ทำไม่ได้เลย
เพราะไม่มีที่เก็บ

### จุดเด่นที่สอง: validation/callback เต็มรูปแบบบน join model

Validation ที่เขียนไว้ใน `Appointment#doctor_is_available` ทำงานเหมือน Model ปกติทุกประการ
(ป้องกันหมอคนเดียวกันมีนัดซ้อนเวลาเดียวกัน):

```irb
irb(main):014> dup_attempt = Appointment.new(doctor: dr_somchai, patient: patient_b, appointment_date: slot)
irb(main):015> dup_attempt.valid?
=> false
irb(main):016> dup_attempt.errors.full_messages
=> ["Appointment date หมอคนนี้มีนัดอื่นอยู่แล้วในเวลานี้"]
```

Habtm ไม่มีทางทำแบบนี้ได้เลย เพราะไม่มี Model ให้เขียน `validate` ผูกกับแถวความสัมพันธ์แต่ละแถว —
นี่คือเหตุผลหลักที่ `has_many :through` ถูกแนะนำมากกว่า habtm ในเกือบทุกสถานการณ์: **ได้ความสามารถ
ของ many-to-many เท่ากัน แต่ยืดหยุ่นกว่ามาก เพราะตัวกลางเป็น Model เต็มรูปแบบ**

### `includes` กับ `has_many :through`

Eager loading (ที่เรียนเบื้องต้นใน Part 027 Step 269) ใช้กับ `:through` ได้ปกติ แต่มีรายละเอียด
เล็กน้อยที่ควรรู้ — ต้อง query **3 ครั้ง** แทนที่จะเป็น 2 ครั้งเหมือน `has_many` ปกติ เพราะต้อง
โหลดตัวกลาง (`appointments`) มาก่อนถึงจะรู้ว่าต้องไปดึง `patients` แถวไหนบ้าง:

```ruby
Doctor.includes(:patients).each do |d|
  puts "#{d.name}: #{d.patients.map(&:name).join(', ')}"
end
```

```
Doctor Load (0.1ms)  SELECT "doctors".* FROM "doctors"
Appointment Load (0.1ms)  SELECT "appointments".* FROM "appointments" WHERE "appointments"."doctor_id" IN (1, 2)
Patient Load (0.1ms)  SELECT "patients".* FROM "patients" WHERE "patients"."id" IN (1, 2)
```

ยังคงเป็น **จำนวนคงที่ (constant)** ไม่ว่าจะมีหมอกี่คนก็ตาม (ไม่ใช่ N+1) แค่เป็น 3 แทน 2 —
รายละเอียดเชิงลึกเรื่อง eager loading ทั้งหมดรอใน Part 034

---

## Step 324: แปลง `has_and_belongs_to_many` เป็น `has_many :through` จริง (migration before/after พร้อม backfill ข้อมูล)

สถานการณ์ที่เจอบ่อยมากในงานจริง: เริ่มต้นด้วย habtm อย่าง `Student`/`Course` ใน Step 322 ไปก่อน
เพราะดูง่ายกว่า แล้ววันหนึ่งทีมงานต้องการเก็บ **วันที่ลงทะเบียน** และ **เกรดที่ได้** — ต้องแปลงเป็น
`has_many :through` โดยที่ **ข้อมูลเดิมที่มีอยู่แล้วห้ามหาย**

### ก่อนแปลง: มีข้อมูล habtm อยู่แล้วในตาราง `courses_students`

```irb
irb(main):001> ActiveRecord::Base.connection.select_all("SELECT * FROM courses_students").to_a
=> [{"course_id"=>1, "student_id"=>1}, {"course_id"=>2, "student_id"=>1}, {"course_id"=>1, "student_id"=>2}]
```

สามแถวนี้คือ: Alice (id 1) ลงเรียน Math 101 (id 1) และ Physics 101 (id 2), Bob (id 2) ลงเรียนแค่
Math 101

### สร้าง migration แปลงข้อมูล

```bash
bin/rails generate migration ConvertCoursesStudentsHabtmToEnrollments
```

```ruby
class ConvertCoursesStudentsHabtmToEnrollments < ActiveRecord::Migration[8.1]
  def up
    create_table :enrollments do |t|
      t.references :student, null: false, foreign_key: true
      t.references :course, null: false, foreign_key: true
      t.date :enrolled_on
      t.string :grade

      t.timestamps
    end
    add_index :enrollments, [:student_id, :course_id], unique: true

    # backfill ข้อมูลเก่าจากตาราง join แบบ habtm มาไว้ในตารางใหม่
    execute <<~SQL
      INSERT INTO enrollments (student_id, course_id, enrolled_on, created_at, updated_at)
      SELECT student_id, course_id, DATE('now'), datetime('now'), datetime('now')
      FROM courses_students
    SQL

    drop_table :courses_students
  end

  def down
    create_join_table :courses, :students do |t|
      t.index [:course_id, :student_id]
      t.index [:student_id, :course_id]
    end

    execute <<~SQL
      INSERT INTO courses_students (course_id, student_id)
      SELECT course_id, student_id FROM enrollments
    SQL

    drop_table :enrollments
  end
end
```

**สิ่งที่เกิดขึ้นในนี้มี 4 ขั้นตอน:** (1) สร้างตาราง `enrollments` ใหม่ พร้อม column เพิ่มเติมที่
habtm ไม่มี, (2) ยิง SQL `INSERT ... SELECT` ดึงข้อมูลทุกแถวจาก `courses_students` เดิมมาใส่ใน
`enrollments` (ใช้ `execute` ยิง raw SQL ตรงๆ เพราะเป็นการย้ายข้อมูลระดับตาราง ไม่ผ่าน
ActiveRecord Model), (3) ลบตารางเก่า `courses_students` ทิ้ง, และ (4) เขียน `down` ให้ทำย้อนกลับ
ได้ครบถ้วน (สร้างตาราง habtm กลับมา, ย้ายข้อมูลกลับ, ลบ `enrollments`) — เพราะ migration ที่มีการ
`drop_table`/`execute` แบบนี้ ActiveRecord **อนุมานทิศทางย้อนกลับให้เองไม่ได้** จึงต้องแยกเขียน
`up`/`down` เอง (ทบทวนหลักการนี้ได้จาก Part 025 Step 243)

```bash
bin/rails db:migrate
```

```
== ConvertCoursesStudentsHabtmToEnrollments: migrating =======================
-- create_table(:enrollments)
   -> 0.0068s
-- add_index(:enrollments, [:student_id, :course_id], {:unique=>true})
   -> 0.0060s
-- execute("INSERT INTO enrollments ...")
   -> 0.0048s
-- drop_table(:courses_students)
   -> 0.0022s
== ConvertCoursesStudentsHabtmToEnrollments: migrated (0.0199s) ===============
```

### สร้าง Model `Enrollment` และแก้ Model เดิม

```bash
bin/rails generate model Enrollment student:references course:references enrolled_on:date grade:string
```

> **หมายเหตุ:** เพราะสร้างตาราง `enrollments` ไปแล้วในขั้นตอนก่อนหน้า ให้ลบไฟล์ migration ที่
> generator สร้างมาให้รอบนี้ทิ้ง (`db/migrate/..._create_enrollments.rb`) เก็บไว้แค่ไฟล์ Model
> (`app/models/enrollment.rb`) เท่านั้น ไม่งั้นจะพยายามสร้างตารางซ้ำตอน `db:migrate`

```ruby
# app/models/enrollment.rb
class Enrollment < ApplicationRecord
  belongs_to :student
  belongs_to :course

  validates :student_id, uniqueness: { scope: :course_id }
end
```

```ruby
# app/models/student.rb — เปลี่ยนจาก has_and_belongs_to_many เป็น has_many :through
class Student < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :courses, through: :enrollments
end
```

```ruby
# app/models/course.rb
class Course < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :students, through: :enrollments
end
```

### ตรวจสอบว่าข้อมูลเดิมยังอยู่ครบ และใช้ความสามารถใหม่ได้

```irb
irb(main):001> alice = Student.find_by(name: "Alice")
irb(main):002> alice.courses.pluck(:title)
=> ["Math 101", "Physics 101"]   # <- ข้อมูลเดิมยังอยู่ครบ ไม่หายไปไหน

irb(main):003> alice.enrollments.first.enrolled_on
=> Sat, 26 Sep 2026

irb(main):004> enr = alice.enrollments.find_by(course: Course.find_by(title: "Math 101"))
irb(main):005> enr.update!(grade: "A")
irb(main):006> enr.reload.grade
=> "A"   # <- ความสามารถใหม่ที่ habtm ทำไม่ได้: เก็บเกรดได้แล้ว

irb(main):007> dup = Enrollment.new(student: alice, course: Course.find_by(title: "Math 101"))
irb(main):008> dup.valid?
=> false
irb(main):009> dup.errors.full_messages
=> ["Student has already been taken"]   # <- uniqueness validation ระดับ Model ทำงานได้แล้วเช่นกัน
```

ข้อมูลความสัมพันธ์เดิมทั้งหมดถูกเก็บรักษาไว้ครบ (Alice ยังลงทะเบียนสองวิชาเหมือนเดิม) และตอนนี้ยัง
เพิ่มความสามารถใหม่ (`grade`, `enrolled_on`, `validates uniqueness`) ที่ habtm ไม่เคยทำได้ —
นี่คือรูปแบบการ migrate ที่ปลอดภัยที่จะใช้จริงเมื่อต้อง "อัปเกรด" habtm เป็น `has_many :through`
ในโปรเจกต์ที่มีข้อมูล production อยู่แล้ว

---

## Step 325: Self-referential association แบบ `has_many :through` — ระบบ follow (`User` ติดตามกันเองผ่าน `Follow`)

**Self-referential association** คือความสัมพันธ์ที่ Model **อ้างอิงกลับไปหา Model ชนิดเดียวกัน**
กับตัวเอง เช่นระบบโซเชียลที่ผู้ใช้ติดตาม (follow) ผู้ใช้คนอื่นได้ — ทั้ง "คนติดตาม" และ "คนถูกติดตาม"
ล้วนเป็น `User` เหมือนกันทั้งคู่ ทำให้ตั้งชื่อ column/association แบบปกติ (`user_id`) ไม่พอ ต้องแยก
บทบาทให้ชัดเจน

### สร้าง Model `Follow` เป็นตัวกลาง

```bash
bin/rails generate model User name:string
bin/rails generate model Follow follower_id:integer followed_id:integer
```

แก้ migration ของ `Follow` ให้ใช้ `references` แบบระบุ `to_table:` (เพราะทั้งคู่ต้องชี้ไปยังตาราง
`users` เดียวกัน แต่คนละ column):

```ruby
class CreateFollows < ActiveRecord::Migration[8.1]
  def change
    create_table :follows do |t|
      t.references :follower, null: false, foreign_key: { to_table: :users }
      t.references :followed, null: false, foreign_key: { to_table: :users }

      t.timestamps
    end

    add_index :follows, [:follower_id, :followed_id], unique: true
  end
end
```

**จุดสำคัญ:** `foreign_key: { to_table: :users }` บอก Rails ว่า column `follower_id` และ
`followed_id` ไม่ได้ชี้ไปตาราง `followers`/`followeds` ตามชื่อ (เพราะไม่มีตารางนั้นอยู่จริง) แต่ชี้
ไปยังตาราง **`users`** ทั้งคู่ — ถ้าไม่ระบุ `to_table:` Rails จะพยายามเดาชื่อตารางจากชื่อ column
ผิดไปเลย และ `add_index` แบบ `unique: true` ป้องกันไม่ให้ user คนเดียวกัน follow อีกคนซ้ำสองครั้ง

```bash
bin/rails db:migrate
```

### เขียน association ทั้งสองทิศทางใน `Follow` และ `User`

```ruby
# app/models/follow.rb
class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"

  validates :follower_id, uniqueness: { scope: :followed_id }
end
```

`class_name: "User"` บอกว่า `:follower` และ `:followed` ทั้งคู่เป็น **`User`** จริงๆ (ไม่มี Model
ชื่อ `Follower`/`Followed` อยู่จริง) เทคนิคนี้เรียนผ่านมาบ้างแล้วใน Part 027 Step 270
(`class_name`/`foreign_key` แบบ custom) ตอนนี้เจอในบริบทที่จำเป็นต้องใช้จริงๆ

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :active_follows, class_name: "Follow", foreign_key: "follower_id",
                             inverse_of: :follower, dependent: :destroy
  has_many :passive_follows, class_name: "Follow", foreign_key: "followed_id",
                              inverse_of: :followed, dependent: :destroy

  has_many :followed_users, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower

  def follow(other_user)
    followed_users << other_user unless self == other_user || following?(other_user)
  end

  def unfollow(other_user)
    active_follows.find_by(followed: other_user)&.destroy
  end

  def following?(other_user)
    followed_users.include?(other_user)
  end
end
```

**อ่านทีละบรรทัด:**

- `has_many :active_follows` — แถวใน `follows` ที่ user คนนี้เป็นฝั่ง "ผู้ติดตาม" (`follower_id`
  ชี้มาที่ตัวเอง) ตั้งชื่อว่า "active" เพราะเป็นฝั่งที่ **กระทำ** การ follow
- `has_many :passive_follows` — แถวที่ user คนนี้เป็นฝั่ง "ถูกติดตาม" (`followed_id` ชี้มาที่ตัวเอง)
  ตั้งชื่อว่า "passive" เพราะเป็นฝั่งที่ **ถูกกระทำ**
- `has_many :followed_users, through: :active_follows, source: :followed` — เดินผ่าน
  `active_follows` แล้วดึง `:followed` ของแต่ละแถว (**ต้องระบุ `source:` เสมอ** เพราะชื่อ
  association (`followed_users`) ไม่ตรงกับชื่อ column/association ที่แท้จริงใน `Follow`
  (`:followed`) — ต่างจาก `has_many :through` ปกติใน Step 323 ที่ชื่อ Model กับชื่อ association
  ตรงกันพอดี Rails เดา `source` เองได้ แต่กรณีนี้เดาไม่ได้ ต้องบอกตรงๆ)
- `has_many :followers, through: :passive_follows, source: :follower` — หลักการเดียวกัน
  เดินผ่าน `passive_follows` แล้วดึง `:follower` ของแต่ละแถว

`inverse_of:` ที่ใส่ไว้ก็เพื่อป้องกันปัญหา object คนละตัวแบบที่เรียนใน Part 027 Step 270 —
กรณี self-referential/`:through` แบบนี้เป็นหนึ่งในสถานการณ์ที่ Rails auto-detect ไม่ได้เอง
ต้องระบุด้วยตนเองเสมอ

### ใช้งานจริง

```irb
irb(main):001> alice = User.create!(name: "Alice")
irb(main):002> bob = User.create!(name: "Bob")
irb(main):003> carol = User.create!(name: "Carol")

irb(main):004> alice.follow(bob)
irb(main):005> alice.follow(carol)
irb(main):006> bob.follow(carol)

irb(main):007> alice.followed_users.pluck(:name)
=> ["Bob", "Carol"]
irb(main):008> carol.followers.pluck(:name)
=> ["Alice", "Bob"]

irb(main):009> alice.following?(bob)
=> true
irb(main):010> bob.following?(alice)
=> false   # <- follow ทางเดียว ไม่ใช่ mutual โดยอัตโนมัติ

irb(main):011> alice.unfollow(bob)
irb(main):012> alice.followed_users.pluck(:name)
=> ["Carol"]
```

`carol.followers` แสดงให้เห็นชัดว่า **Carol ถูกติดตามโดยทั้ง Alice และ Bob** — และ `bob.following?
(alice)` คืน `false` เพราะการ follow เป็น**ทิศทางเดียว** (Bob follow Carol ไม่ได้แปลว่า Carol
follow Bob กลับ) ตรงตามธรรมชาติของ social follow system ทั่วไป (ต่างจาก "เพื่อน" แบบ Facebook
ที่ต้องยืนยันสองทาง ซึ่งจะออกแบบต่างออกไปเล็กน้อย)

---

## Step 326: Self-referential association แบบ `belongs_to` — ผังองค์กร (`Employee belongs_to :manager`)

Self-referential ไม่จำเป็นต้องมี Model ที่สามมาคั่นกลางเสมอไป — ถ้าความสัมพันธ์เป็นแบบ
**หนึ่งต่อกลุ่มธรรมดา** (ไม่ใช่ many-to-many) ก็ใช้ `belongs_to`/`has_many` ตรงๆ ได้เลย เพียงแค่
ทั้งสองฝั่งเป็น Model เดียวกัน — ตัวอย่างคลาสสิกคือ **ผังองค์กร**: พนักงานหนึ่งคนมีหัวหน้า (manager)
ได้แค่คนเดียว แต่หัวหน้าหนึ่งคนมีลูกน้อง (subordinates) ได้หลายคน

### สร้าง Model และ Migration

```bash
bin/rails generate model Employee name:string manager_id:integer
```

แก้ migration ให้เพิ่ม index และ foreign key **ชี้กลับไปที่ตารางเดียวกัน**:

```ruby
class CreateEmployees < ActiveRecord::Migration[8.1]
  def change
    create_table :employees do |t|
      t.string :name
      t.integer :manager_id

      t.timestamps
    end

    add_index :employees, :manager_id
    add_foreign_key :employees, :employees, column: :manager_id
  end
end
```

`add_foreign_key :employees, :employees, column: :manager_id` คือจุดที่ต่างจาก foreign key
ปกติ — ทั้ง argument แรกและ argument ที่สองเป็นตารางเดียวกัน (`:employees` ชี้กลับไปหาตัวเอง)
ต้องระบุ `column:` ให้ชัดเจนเสมอเพราะชื่อ column (`manager_id`) ไม่ตรงกับชื่อ Model (`employee`)
ตาม convention ปกติ

```bash
bin/rails db:migrate
```

### เขียน association ทั้งสองทิศทางในตัวเดียว

```ruby
# app/models/employee.rb
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true, inverse_of: :subordinates
  has_many :subordinates, class_name: "Employee", foreign_key: "manager_id", inverse_of: :manager
end
```

**สังเกต:** `belongs_to :manager, optional: true` — ต้องใส่ `optional: true` เสมอสำหรับ
self-referential แบบนี้ เพราะ **ตำแหน่งบนสุด (เช่น CEO) ไม่มี manager** (ถ้าไม่ใส่ `optional:
true` จะสร้าง CEO ไม่ได้เลยตามกฎ "required by default" จาก Part 027 Step 262) และ `has_many
:subordinates, foreign_key: "manager_id"` ต้องระบุ `foreign_key:` เองเพราะชื่อ association
(`subordinates`) ไม่ตรงกับชื่อ column (`manager_id`) ตาม convention ปกติเหมือนกัน

### ใช้งานจริง

```irb
irb(main):001> ceo = Employee.create!(name: "CEO สมหมาย")
irb(main):002> vp = Employee.create!(name: "VP วิชัย", manager: ceo)
irb(main):003> eng1 = Employee.create!(name: "Engineer มานะ", manager: vp)
irb(main):004> eng2 = Employee.create!(name: "Engineer มานี", manager: vp)

irb(main):005> vp.manager.name
=> "CEO สมหมาย"
irb(main):006> ceo.subordinates.pluck(:name)
=> ["VP วิชัย"]
irb(main):007> vp.subordinates.pluck(:name)
=> ["Engineer มานะ", "Engineer มานี"]
irb(main):008> ceo.manager
=> nil
```

`ceo.manager` คืน `nil` อย่างปลอดภัย (เพราะ `optional: true`) — ผังองค์กรทำงานได้ครบทั้งขึ้นและลง
ด้วย `belongs_to`/`has_many` ธรรมดา ไม่ต้องมี Model ตัวที่สามเหมือนระบบ follow ใน Step 325 เพราะ
ความสัมพันธ์เป็นแบบ **หนึ่งต่อกลุ่ม** (พนักงานหนึ่งคนมี manager ได้แค่คนเดียว) ไม่ใช่
**กลุ่มต่อกลุ่ม** (ระบบ follow ที่ user หนึ่งคน follow ได้หลายคน และถูก follow ได้จากหลายคนพร้อมกัน
แบบไม่มีขีดจำกัดทั้งสองทิศทาง)

### เดินลึกหลายชั้น (recursive) ด้วย Ruby ธรรมดา

ถ้าอยากได้ลูกน้อง**ทุกระดับ** (ไม่ใช่แค่ระดับถัดไป) เขียน method แบบ recursive ได้ตรงไปตรงมา:

```ruby
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true, inverse_of: :subordinates
  has_many :subordinates, class_name: "Employee", foreign_key: "manager_id", inverse_of: :manager

  def all_subordinates
    subordinates.flat_map { |sub| [sub] + sub.all_subordinates }
  end
end
```

```irb
irb(main):001> ceo.all_subordinates.map(&:name)
=> ["VP วิชัย", "Engineer มานะ", "Engineer มานี"]
```

> **ข้อควรระวังเรื่องประสิทธิภาพ:** `all_subordinates` แบบนี้ยิง query แยกทีละชั้น (N+1 ตามความลึก
> ของผังองค์กร) ใช้ได้กับผังที่ไม่ลึกมาก (ไม่กี่สิบ-ร้อยคน) แต่ถ้าองค์กรใหญ่มากและมีผังลึกหลายสิบชั้น
> ควรพิจารณาใช้ **Recursive CTE** (Common Table Expression) ที่ฐานข้อมูล query ให้ครบทุกชั้นใน
> คำสั่งเดียว หรือใช้ gem อย่าง `closure_tree`/`ancestry` — เป็นหัวข้อขั้นสูงที่อยู่นอกขอบเขตของ
> หลักสูตรพื้นฐานนี้ ให้จำไว้แค่ว่ามีทางเลือกที่เร็วกว่านี้อยู่เมื่อข้อมูลใหญ่ขึ้นจริง

---

## Step 327: Polymorphic association เบื้องต้น — `Comment belongs_to :commentable, polymorphic: true`

จนถึงตอนนี้ทุก `belongs_to` ที่เรียนมาชี้ไปยัง **Model เดียวที่แน่นอน** (`Book belongs_to
:author` ชี้ไป `Author` เท่านั้น) แต่บางสถานการณ์ต้องการให้ Model หนึ่ง**อ้างอิงไปยังได้หลาย Model**
เช่น ระบบคอมเมนต์ที่อยากให้คอมเมนต์แปะได้ทั้งใต้ **โพสต์ (`Post`)** และใต้ **รูปภาพ (`Photo`)**
โดยใช้ตาราง `comments` ตารางเดียว ไม่ต้องแยกเป็น `post_comments`/`photo_comments`

### วิธีที่ไม่ใช้ polymorphic (และทำไมไม่ดี)

ถ้าใช้ `belongs_to` ธรรมดา ต้องมี column แยกกันสอง column:

```ruby
create_table :comments do |t|
  t.references :post, foreign_key: true      # nullable ถ้าคอมเมนต์นี้อยู่ใต้ photo แทน
  t.references :photo, foreign_key: true     # nullable ถ้าคอมเมนต์นี้อยู่ใต้ post แทน
  t.text :body
end
```

ปัญหาคือ **ทุกแถวจะมี column ที่ไม่ได้ใช้เหลือเป็น `NULL` เสมอ** (คอมเมนต์ใต้ post จะมี
`photo_id = NULL` ค้างอยู่ตลอด) และยิ่งมี "สิ่งที่คอมเมนต์แปะได้" เพิ่มขึ้นอีก (เช่น video, album)
ก็ต้องเพิ่ม column ใหม่ไปเรื่อยๆ ไม่มีที่สิ้นสุด — Rails มีวิธีที่สะอาดกว่านี้เรียกว่า
**polymorphic association**

### สร้าง Model `Comment` แบบ polymorphic

```bash
bin/rails generate model Post title:string body:text
bin/rails generate model Photo caption:string
bin/rails generate model Comment commentable:references{polymorphic} body:text
```

สังเกต syntax พิเศษ `commentable:references{polymorphic}` — บอก generator ว่า column นี้เป็น
polymorphic reference migration ที่ได้:

```ruby
class CreateComments < ActiveRecord::Migration[8.1]
  def change
    create_table :comments do |t|
      t.references :commentable, polymorphic: true, null: false
      t.text :body

      t.timestamps
    end
  end
end
```

```bash
bin/rails db:migrate
```

schema ที่ได้:

```ruby
create_table "comments", force: :cascade do |t|
  t.string "commentable_type", null: false
  t.integer "commentable_id", null: false
  t.text "body"
  t.datetime "created_at", null: false
  t.datetime "updated_at", null: false
  t.index ["commentable_type", "commentable_id"], name: "index_comments_on_commentable"
end
```

**จุดสำคัญที่สุดของ polymorphic:** `t.references :commentable, polymorphic: true` สร้าง
**สอง column** แทนที่จะเป็นหนึ่ง:

| Column | หน้าที่ |
|---------|---------|
| `commentable_id` | เก็บ `id` ของ record ที่ถูกคอมเมนต์ (เหมือน foreign key ปกติ) |
| `commentable_type` | เก็บ **ชื่อ class** ของ record นั้นเป็น string เช่น `"Post"` หรือ `"Photo"` |

ทั้งสอง column ทำงานร่วมกันเป็นคู่เสมอ — `commentable_id: 5, commentable_type: "Post"` แปลว่า
"คอมเมนต์นี้ผูกกับ `Post.find(5)`" ส่วน `commentable_id: 5, commentable_type: "Photo"` แปลว่า
"ผูกกับ `Photo.find(5)`" (คนละ record กันแม้ `id` จะเท่ากันก็ตาม เพราะเช็คคู่กับ `_type` เสมอ)
มี composite index บนทั้งสอง column พร้อมกัน (`index_comments_on_commentable`) เพื่อให้ query
ไวตอนค้นหาคอมเมนต์ของ record หนึ่งๆ

### ประกาศ association

```ruby
# app/models/comment.rb (generator สร้างให้อัตโนมัติ)
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end
```

```ruby
# app/models/photo.rb
class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end
```

**`has_many :comments, as: :commentable`** คือกุญแจสำคัญฝั่ง "แม่" — `as:` บอกว่า `Post`/`Photo`
เล่นบทบาทเป็น `:commentable` ให้กับ `Comment` (คนละคำกับ `through:` ที่เจอใน Step 323 อย่าสับสน
กัน — `through:` ใช้ตอนเดินผ่านตัวกลาง ส่วน `as:` ใช้ตอนบอกบทบาทของฝั่ง polymorphic)

### ใช้งานจริง

```irb
irb(main):001> post = Post.create!(title: "เปิดตัวฟีเจอร์ใหม่", body: "รายละเอียด...")
irb(main):002> photo = Photo.create!(caption: "รูปทริปเชียงใหม่")

irb(main):003> c1 = post.comments.create!(body: "เยี่ยมมาก!")
irb(main):004> c2 = post.comments.create!(body: "รอฟีเจอร์นี้มานานแล้ว")
irb(main):005> c3 = photo.comments.create!(body: "สวยมากครับ")

irb(main):006> post.comments.pluck(:body)
=> ["เยี่ยมมาก!", "รอฟีเจอร์นี้มานานแล้ว"]
irb(main):007> photo.comments.pluck(:body)
=> ["สวยมากครับ"]

irb(main):008> c1.commentable
=> #<Post id: 1, title: "เปิดตัวฟีเจอร์ใหม่", ...>
irb(main):009> c1.commentable_type
=> "Post"
irb(main):010> c3.commentable_type
=> "Photo"
```

`post.comments.create!(...)` ตั้งค่า `commentable_id`/`commentable_type` ให้อัตโนมัติทั้งคู่
(ไม่ต้องเขียนเอง) เหมือนกับ `has_many`/`belongs_to` ปกติทุกประการ — และ `c1.commentable` เดินกลับ
ไปหา `Post` ตัวจริงได้เลยโดยไม่ต้องรู้ล่วงหน้าว่ามันคือ `Post` หรือ `Photo`

### Query ผ่าน column polymorphic โดยตรง

```irb
irb(main):011> Comment.where(commentable_type: "Post").count
=> 2
irb(main):012> Comment.where(commentable: post).count
=> 2
```

`Comment.where(commentable: post)` ActiveRecord ฉลาดพอที่จะแปลงเป็นเงื่อนไขทั้งสอง column ให้
อัตโนมัติ (`WHERE commentable_type = 'Post' AND commentable_id = 1`) โดยไม่ต้องเขียนแยกเอง

---

## Step 328: ข้อเสียของ Polymorphic Association — ไม่มี foreign key constraint จริงในฐานข้อมูล

Polymorphic ดูทรงพลังมากใน Step 327 แต่มีข้อเสียสำคัญที่ต้องเข้าใจให้ชัดก่อนใช้จริง — **ฐานข้อมูล
ไม่สามารถสร้าง foreign key constraint จริงให้ column polymorphic ได้เลย**

### ทำไมถึงสร้าง foreign key ไม่ได้

จำได้จาก Part 027 Step 264 ไหมว่า `foreign_key: true` ทำให้ฐานข้อมูลปฏิเสธค่า `id` ที่ไม่มีอยู่จริง
ในตารางที่ชี้ไป — แต่ foreign key constraint ของฐานข้อมูลเชิงสัมพันธ์ (ทั้ง PostgreSQL, MySQL,
SQLite) **ถูกออกแบบมาให้ชี้ไปยัง "ตารางเดียว" ที่แน่นอนตายตัวเท่านั้น** ไม่มีแนวคิด "ชี้ไปตารางนี้
หรือตารางโน้น สลับกันได้ตามค่าใน column อื่น" อยู่ในมาตรฐาน SQL เลย — `commentable_id` ต้อง
สามารถชี้ได้ทั้งไปยัง `posts.id` และ `photos.id` แล้วแต่ค่า `commentable_type` ซึ่งเป็นสิ่งที่
ฐานข้อมูลระดับ constraint ทำไม่ได้ตั้งแต่ต้น — ลองดู migration ของ `comments` อีกครั้ง
สังเกตว่าไม่มีบรรทัด `add_foreign_key` เหมือนตัวอย่างอื่นๆ ก่อนหน้าเลย

### พิสูจน์ด้วยการใส่ข้อมูลกำพร้า (orphan) เข้าไปตรงๆ

```irb
irb(main):001> bad_comment = Comment.new(commentable_type: "Post", commentable_id: 999_999, body: "orphan")
irb(main):002> bad_comment.valid?
=> false   # <- validation ระดับ Model (Rails 5+ required by default) จับได้
irb(main):003> bad_comment.save!(validate: false)
=> true    # <- แต่ถ้าข้าม validation (เช่น import script, seed เก่า, bug) ฐานข้อมูลไม่ปฏิเสธเลย!
irb(main):004> bad_comment.persisted?
=> true
irb(main):005> bad_comment.commentable
=> nil     # <- เข้าถึงแล้วได้ nil เงียบๆ ไม่มี error ใดๆ
irb(main):006> Comment.count
=> 4       # <- comment กำพร้าตัวนี้ยังนับรวมอยู่ในฐานข้อมูลจริง
```

`commentable_id: 999_999` ไม่มี `Post` id นี้อยู่จริงเลย แต่ **ฐานข้อมูลไม่มีทางรู้และไม่มีทาง
ปฏิเสธ** เพราะไม่มี foreign key constraint ผูกอยู่ — ต่างจากตัวอย่าง `belongs_to` ธรรมดาใน Part
027 Step 268 ที่ฐานข้อมูลจะ raise `ActiveRecord::InvalidForeignKey` ทันทีถ้าพยายามทำแบบนี้
กับ foreign key ปกติที่มี `foreign_key: true`

> **นี่คือ tradeoff หลักของ polymorphic association ที่ต้องยอมรับก่อนเลือกใช้:** ความยืดหยุ่นที่
> ได้มา (Model เดียวอ้างอิงได้หลาย Model) แลกมาด้วย **ไม่มีตาข่ายรองรับชั้นสุดท้ายจากฐานข้อมูล**
> เหมือนที่เรียนหลักการไว้ใน Part 027 Step 263 — ความถูกต้องของข้อมูล (data integrity) ทั้งหมด
> ต้องพึ่งพา **โค้ดระดับแอปพลิเคชันเพียงอย่างเดียว** (validation ระดับ Model, การไม่ข้าม
> validation ด้วย `save(validate: false)`/`update_column`/`insert_all` อย่างไม่ระวัง) ถ้าโค้ด
> มีบั๊กหรือมีใครยิง SQL ตรงๆ ข้าม ActiveRecord ไปเลย ข้อมูลกำพร้าแบบนี้จะหลุดเข้าไปในฐานข้อมูล
> ได้เงียบๆ โดยไม่มีสัญญาณเตือนใดๆ

### แนวทางลดความเสี่ยง

ไม่มีทางแก้ที่สมบูรณ์แบบ 100% แต่ลดความเสี่ยงได้ด้วยแนวทางเหล่านี้:

1. **ห้ามข้าม validation โดยไม่จำเป็น** — เลี่ยง `save(validate: false)`, `update_column`,
   `insert_all` กับตารางที่มี polymorphic association เว้นแต่มั่นใจจริงๆ ว่าข้อมูลถูกต้อง
2. **จำกัดชนิด `commentable_type` ที่ยอมรับได้ด้วย validation** — ป้องกัน string มั่วๆ หลุดเข้ามา:

   ```ruby
   class Comment < ApplicationRecord
     belongs_to :commentable, polymorphic: true

     validates :commentable_type, inclusion: { in: %w[Post Photo] }
   end
   ```

3. **เขียน rake task/script ตรวจสอบข้อมูลกำพร้าเป็นระยะ** (data integrity check) แทนที่จะพึ่ง
   ฐานข้อมูลบังคับให้อัตโนมัติ — เช่น query หา `commentable_id` ที่ไม่มีอยู่จริงในตารางเป้าหมาย
   แล้วแจ้งเตือนหรือลบทิ้งเป็นรอบๆ
4. **พิจารณาทางเลือกอื่นถ้าจำนวนชนิดที่อ้างอิงได้มีจำกัดและรู้ล่วงหน้าตายตัว** — เช่นถ้ารู้แน่ว่า
   คอมเมนต์แปะได้แค่ `Post` กับ `Photo` เท่านั้นตลอดไป (ไม่มีวันเพิ่มชนิดที่สาม) การใช้ two
   nullable foreign keys ธรรมดา (`post_id`, `photo_id` ที่มี foreign key constraint จริง)
   อาจปลอดภัยกว่าในระยะยาว แลกกับความยืดหยุ่นที่น้อยลงเมื่อต้องเพิ่มชนิดใหม่ในอนาคต — Rails 6.1+
   ยังมี **Delegated Types** (`ActiveRecord::DelegatedType`) เป็นทางเลือกลูกผสมระหว่างสองแบบนี้
   ที่ยังคง foreign key constraint จริงได้ แต่อยู่นอกขอบเขตของหลักสูตรพื้นฐานนี้

---

## Step 329: `has_one :through` — เดินความสัมพันธ์แบบหนึ่งต่อหนึ่งผ่านตัวกลาง

`has_many :through` (Step 323) มีคู่แฝดชื่อ **`has_one :through`** — ใช้เมื่อความสัมพันธ์ทุกขั้น
เป็นแบบ **หนึ่งต่อหนึ่ง** ตลอดสาย ตัวอย่างคลาสสิกจาก Rails Guide เอง: **Supplier (ผู้ผลิต) มี
Account (บัญชี) ได้แค่บัญชีเดียว และ Account หนึ่งบัญชีมี AccountHistory (ประวัติเครดิต) ได้แค่
ชุดเดียว** — อยากให้ `supplier.account_history` เดินลัดข้าม `Account` ไปถึง `AccountHistory`
เลยในคำเดียว

### สร้าง Model ทั้งสาม

```bash
bin/rails generate model Supplier name:string
bin/rails generate model Account supplier:references account_number:string
bin/rails generate model AccountHistory account:references credit_rating:integer
bin/rails db:migrate
```

### เขียน association

```ruby
# app/models/account.rb
class Account < ApplicationRecord
  belongs_to :supplier
  has_one :account_history, dependent: :destroy
end
```

```ruby
# app/models/supplier.rb
class Supplier < ApplicationRecord
  has_one :account, dependent: :destroy
  has_one :account_history, through: :account
end
```

**`has_one :account_history, through: :account`** อ่านว่า "Supplier มี `account_history` ได้
หนึ่งชุด โดยเดินผ่าน `account` ของตัวเอง" — ต้องมี `has_one :account` เขียนไว้ก่อนบรรทัดนี้เสมอ
(เหมือนกฎเดียวกับ `has_many :through` ใน Step 323 ที่ `through:` ต้องชี้ไปยัง association ที่มี
อยู่แล้ว)

### ใช้งานจริง

```irb
irb(main):001> supplier = Supplier.create!(name: "บริษัท ผู้ผลิต เอบีซี จำกัด")
irb(main):002> account = supplier.create_account!(account_number: "ACC-001")
irb(main):003> history = account.create_account_history!(credit_rating: 85)

irb(main):004> supplier.account.account_number
=> "ACC-001"
irb(main):005> supplier.account_history.credit_rating
=> 85
irb(main):006> supplier.reload.account_history.credit_rating
=> 85
```

`supplier.account_history` ยิง query แบบ join ข้ามสองตาราง (`accounts` แล้วต่อไป
`account_histories`) ให้อัตโนมัติในคำสั่งเดียว โดยไม่ต้องเขียน `supplier.account
.account_history` เอง — สะดวกมากเมื่อมีสายความสัมพันธ์แบบหนึ่งต่อหนึ่งซ้อนกันหลายชั้น และทุกชั้น
ล้วนเป็น `has_one` ทั้งหมด (ถ้าขั้นใดขั้นหนึ่งเป็น `has_many` ต้องใช้ `has_many :through` แทน
ไม่ใช่ `has_one :through` — กฎง่ายๆ คือดูที่ **ความสัมพันธ์ขั้นสุดท้าย** ที่ต้องการเดินไปถึง)

---

## Step 330: ตารางสรุปเปรียบเทียบ — habtm vs `has_many :through` vs Polymorphic vs STI

ตอนนี้เรียนมาครบทุกเทคนิคของ association ขั้นสูงแล้ว มาสรุปเป็นตารางเดียวเพื่อใช้ตัดสินใจในงานจริง
พร้อมแนะนำเทคนิคที่ห้า (**Single Table Inheritance — STI**) แบบสั้นๆ เพื่อให้เห็นภาพครบ (STI
ไม่ได้แก้ปัญหาความสัมพันธ์ระหว่างตารางโดยตรงเหมือนสี่แบบแรก แต่มักถูกเทียบเคียงกับ polymorphic
เพราะแก้ปัญหาคล้ายกันในบางสถานการณ์)

### `has_and_belongs_to_many` vs `has_many :through`

| ประเด็น | `has_and_belongs_to_many` | `has_many :through` |
|---------|---------------------------|----------------------|
| ตารางกลาง | ไม่มี `id`, ไม่มี column เพิ่มได้ | ตารางปกติ มี `id`, เพิ่ม column ได้ตามต้องการ |
| Validation บนความสัมพันธ์ | ทำไม่ได้ (มีแค่ callback ระดับ association) | ทำได้เต็มรูปแบบเหมือน Model ปกติ |
| Callback บนความสัมพันธ์ | จำกัดมาก (`before_add`/`after_add` เท่านั้น) | ใช้ callback มาตรฐานทั้งหมดได้ (`before_save` ฯลฯ) |
| จำนวนบรรทัดโค้ดตอนเริ่ม | สั้นกว่าเล็กน้อย (ไม่ต้องสร้าง Model ที่สาม) | ต้องสร้าง Model เพิ่มอีกหนึ่งตัว |
| แนะนำเมื่อไหร่ | ความสัมพันธ์เป็น "แค่ลิงก์" ล้วนๆ ไม่มีวันต้องการข้อมูลเพิ่ม | เกือบทุกกรณีในทางปฏิบัติ (ค่า default ที่ปลอดภัยกว่า) |

### Polymorphic Association vs STI (Single Table Inheritance)

**Polymorphic** (Step 327–328) แก้ปัญหา **"record ฝั่งลูกอ้างอิงไปยัง record ฝั่งแม่ที่เป็นได้
หลายชนิด"** เช่น `Comment` แปะได้ทั้งใต้ `Post` และ `Photo` — Model ฝั่งแม่ (`Post`, `Photo`)
ยังคงเป็น**คนละ Model กันอย่างสมบูรณ์** มีตารางแยกกันเอง (`posts`, `photos`)

**STI** แก้ปัญหาคนละแบบ: **"มีหลาย Model ที่มีโครงสร้างคล้ายกันมาก และอยากให้ทุก subclass ใช้
ตารางฐานข้อมูลตารางเดียวกัน"** เช่น `Vehicle` เป็น base class แล้วมี `Car < Vehicle`, `Motorcycle
< Vehicle`, `Truck < Vehicle` ทั้งหมดเก็บอยู่ในตาราง `vehicles` ตารางเดียว โดยมี column พิเศษชื่อ
`type` (string) บอกว่าแถวนั้นเป็น subclass ไหน (`"Car"`, `"Motorcycle"`, `"Truck"`) —
ActiveRecord จะ instantiate เป็น class ที่ถูกต้องให้อัตโนมัติตามค่าใน column `type` เมื่อ query
กลับมา

| ประเด็น | Polymorphic Association | STI |
|---------|--------------------------|-----|
| แก้ปัญหาอะไร | ลูกอ้างอิงแม่ได้หลายชนิด (แม่เป็นคนละ Model คนละตารางจริง) | มี Model หลายตัวที่คล้ายกัน อยากรวมตารางเป็นตารางเดียว |
| จำนวนตาราง | หลายตาราง (แต่ละ Model ฝั่งแม่มีตารางของตัวเอง) | ตารางเดียว (`type` column แยก subclass) |
| ความสัมพันธ์ | "has-a" / "belongs-to" (Comment มี commentable) | "is-a" (Car เป็น Vehicle ชนิดหนึ่ง) |
| Column ที่ต้องมี | `xxx_type` + `xxx_id` (สอง column) | `type` (หนึ่ง column) |

> **หมายเหตุ:** STI มีรายละเอียดปลีกย่อยเยอะ (การจัดการ column ที่ใช้ไม่ตรงกันระหว่าง subclass,
> validation เฉพาะ subclass, ข้อจำกัดเรื่องตารางกว้างเกินไปเมื่อ subclass มีเยอะและ attribute
> ต่างกันมาก) หลักสูตรนี้แนะนำให้รู้จักแค่แนวคิดพื้นฐานไว้ก่อนตามที่เห็นด้านบน ยังไม่ลงลึกเต็มรูปแบบ
> ในเนื้อหาที่วางแผนไว้ ณ ตอนนี้ — ถ้าเจอสถานการณ์ที่ดูเหมือนต้องใช้ STI ในโปรเจกต์จริง ให้ศึกษา
> เพิ่มเติมจาก Rails Guide โดยตรงเป็นการเฉพาะ

### ตารางตัดสินใจแบบเร็ว: เจอโจทย์แบบนี้ ควรใช้อะไร

| ลักษณะโจทย์ | เทคนิคที่ควรใช้ |
|--------------|-------------------|
| "A หนึ่งตัวมี B ได้หลายตัว และ B หนึ่งตัวก็มี A ได้หลายตัว ไม่มีข้อมูลอื่นบนความสัมพันธ์เลย และมั่นใจว่าจะไม่มีวันต้องการ" | `has_and_belongs_to_many` |
| "A หนึ่งตัวมี B ได้หลายตัว และ B หนึ่งตัวก็มี A ได้หลายตัว" (กรณีทั่วไป ไม่แน่ใจ หรือรู้ว่าต้องการข้อมูล/validation เพิ่มบนความสัมพันธ์) | `has_many :through` |
| "Model เดียวกันอ้างอิงตัวเองเป็นกลุ่ม (follow, friendship, block)" | `has_many :through` แบบ self-referential (Step 325) |
| "Model เดียวกันอ้างอิงตัวเองแบบหนึ่งต่อกลุ่มธรรมดา (ผังองค์กร, comment ตอบกลับ comment)" | `belongs_to`/`has_many` self-referential ตรงๆ (Step 326) |
| "record หนึ่งอยากอ้างอิงไปยัง record ที่เป็นได้หลาย Model ที่ไม่เกี่ยวข้องกันโดยตรง (comment, like, tag, attachment)" | Polymorphic Association (Step 327) — แต่ต้องยอมรับ tradeoff เรื่อง data integrity (Step 328) |
| "มีความสัมพันธ์แบบหนึ่งต่อหนึ่งซ้อนกันหลายชั้น อยากเดินลัดข้ามชั้นกลาง" | `has_one :through` (Step 329) |
| "มีหลาย Model ที่โครงสร้างคล้ายกันมาก อยากรวมเป็นตารางเดียวและใช้ inheritance ของ Ruby" | STI (นอกขอบเขตหลักสูตรนี้ ศึกษาเพิ่มเติมเอง) |

---

## แบบฝึกหัด: ระบบ Tag แบบ Polymorphic (`Post`/`Photo` มี Tag ได้หลายอันผ่าน `Tagging`)

### โจทย์

สร้างระบบติดแท็ก (tagging system) ที่ให้ทั้ง `Post` และ `Photo` มี `Tag` ได้หลายอัน และ `Tag`
หนึ่งอันก็ผูกกับได้ทั้ง `Post` และ `Photo` หลายรายการพร้อมกัน โดยใช้ **polymorphic +
`has_many :through` ผสมกัน** ผ่าน Model กลางชื่อ `Tagging` — โจทย์นี้รวมสองเทคนิคที่เรียนมาทั้ง
Part เข้าด้วยกัน (many-to-many ผ่าน `has_many :through` + polymorphic) เป็นรูปแบบที่ใช้จริงบ่อย
ที่สุดรูปแบบหนึ่งในแอป Rails

**ข้อกำหนด:**

1. สร้าง Model `Tag` (มีแค่ `name:string` ต้อง unique)
2. สร้าง Model `Tagging` เป็นตัวกลางแบบ polymorphic (`tag:references`,
   `taggable:references{polymorphic}`) พร้อมป้องกันการติดแท็กซ้ำ (tag เดียวกันติดซ้ำใน record
   เดียวกันไม่ได้) ทั้งระดับ Model และระดับฐานข้อมูล
3. เพิ่ม `has_many :tags, through: :taggings` ให้ทั้ง `Post` และ `Photo`
4. ทำให้ `tag.posts` และ `tag.photos` แยกชนิดกลับมาได้ถูกต้อง (ไม่ปนกัน)
5. ทดสอบ: สร้างโพสต์/รูปภาพ ติดแท็กหลายอัน, ลบแท็กออกจากบาง record, ลบ record แล้วดูว่า
   `Tagging` ที่เกี่ยวข้องถูกลบตามแต่ `Tag` เองไม่ถูกลบ

### เฉลย

**1) สร้าง Model `Tag`**

```bash
bin/rails generate model Tag name:string
bin/rails db:migrate
```

```ruby
# app/models/tag.rb
class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :posts, through: :taggings, source: :taggable, source_type: "Post"
  has_many :photos, through: :taggings, source: :taggable, source_type: "Photo"

  validates :name, presence: true, uniqueness: true
end
```

**จุดใหม่ที่ยังไม่เคยเจอ:** `source_type:` — ตอนเดินจากฝั่ง `Tag` กลับไปหา `posts`/`photos` ผ่าน
`taggings` (ซึ่งเป็น polymorphic) ActiveRecord ไม่รู้เองว่าจะกรองเอาแต่แถวที่ `taggable_type =
"Post"` หรือ `"Photo"` จึงต้องระบุ **`source_type:`** เพิ่มเข้ามาคู่กับ `source:` เพื่อบอกว่า
"เดินผ่าน `taggings.taggable` แต่กรองเฉพาะที่ชนิดเป็น Post/Photo เท่านั้น" — เป็น pattern มาตรฐาน
เวลาทำ `has_many :through` ที่ตัวกลางเป็น polymorphic

**2) สร้าง Model `Tagging` แบบ polymorphic**

```bash
bin/rails generate model Tagging tag:references taggable:references{polymorphic}
```

migration ที่ได้ (เพิ่ม unique index เองเพื่อกันติดแท็กซ้ำที่ระดับฐานข้อมูล):

```ruby
class CreateTaggings < ActiveRecord::Migration[8.1]
  def change
    create_table :taggings do |t|
      t.references :tag, null: false, foreign_key: true
      t.references :taggable, polymorphic: true, null: false

      t.timestamps
    end

    add_index :taggings, [:tag_id, :taggable_type, :taggable_id], unique: true,
                          name: "index_taggings_uniqueness"
  end
end
```

```bash
bin/rails db:migrate
```

```ruby
# app/models/tagging.rb
class Tagging < ApplicationRecord
  belongs_to :tag
  belongs_to :taggable, polymorphic: true

  validates :tag_id, uniqueness: { scope: [:taggable_type, :taggable_id] }
end
```

ใส่ทั้ง **unique index ระดับฐานข้อมูล** (`add_index ... unique: true`) และ **validation ระดับ
Model** (`validates :tag_id, uniqueness: { scope: ... }`) คู่กันเสมอ — ตรงตามหลักการสองชั้นที่
เรียนมาจาก Part 027 Step 263: validation กันไว้ชั้นแรกให้ error message อ่านง่าย ส่วน unique
index กันไว้ชั้นสุดท้ายกรณี race condition หรือมีทางเข้าอื่นที่ข้าม validation ไป

**3) เพิ่ม `has_many :tags` ใน `Post` และ `Photo`**

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
  has_many :taggings, as: :taggable, dependent: :destroy
  has_many :tags, through: :taggings
end
```

```ruby
# app/models/photo.rb
class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
  has_many :taggings, as: :taggable, dependent: :destroy
  has_many :tags, through: :taggings
end
```

สังเกตว่าฝั่ง `Post`/`Photo` **ไม่ต้องใส่ `source_type:`** เหมือนฝั่ง `Tag` เพราะเดินทิศทางนี้
ActiveRecord รู้ชัดเจนอยู่แล้วว่ากำลังกรองหา `Tagging` ที่ `taggable_id`/`taggable_type` ตรงกับ
`Post`/`Photo` ตัวที่กำลังเรียก (เห็นได้จาก `as: :taggable` ที่ประกาศไว้แล้ว) ความกำกวมเกิดขึ้น
แค่ฝั่ง `Tag` ที่เดินไปหา "อีกฝั่งที่เป็นได้หลายชนิด" เท่านั้น

**4) ทดสอบครบวงจร**

```irb
irb(main):001> post1 = Post.create!(title: "รีวิว Ruby on Rails 8", body: "...")
irb(main):002> post2 = Post.create!(title: "10 เทคนิค ActiveRecord", body: "...")
irb(main):003> photo1 = Photo.create!(caption: "งาน RubyConf Thailand")

irb(main):004> rails_tag = Tag.create!(name: "rails")
irb(main):005> ruby_tag = Tag.create!(name: "ruby")
irb(main):006> event_tag = Tag.create!(name: "event")

irb(main):007> post1.tags << rails_tag
irb(main):008> post1.tags << ruby_tag
irb(main):009> post2.tags << rails_tag
irb(main):010> photo1.tags << event_tag
irb(main):011> photo1.tags << ruby_tag

irb(main):012> post1.tags.pluck(:name)
=> ["rails", "ruby"]
irb(main):013> post2.tags.pluck(:name)
=> ["rails"]
irb(main):014> photo1.tags.pluck(:name)
=> ["event", "ruby"]
```

ทดสอบเดินย้อนกลับจากฝั่ง `Tag` (จุดที่ต้องใช้ `source_type:`):

```irb
irb(main):015> rails_tag.posts.pluck(:title)
=> ["รีวิว Ruby on Rails 8", "10 เทคนิค ActiveRecord"]
irb(main):016> ruby_tag.posts.pluck(:title)
=> ["รีวิว Ruby on Rails 8"]
irb(main):017> ruby_tag.photos.pluck(:caption)
=> ["งาน RubyConf Thailand"]
irb(main):018> event_tag.photos.pluck(:caption)
=> ["งาน RubyConf Thailand"]
```

`ruby_tag.posts` และ `ruby_tag.photos` แยกชนิดกลับมาถูกต้องแม้จะผูกอยู่กับทั้งสองชนิดพร้อมกัน —
พิสูจน์ว่า `source_type:` ทำงานตามที่ตั้งใจ

ทดสอบ validation กันติดแท็กซ้ำ:

```irb
irb(main):019> dup = Tagging.new(tag: rails_tag, taggable: post1)
irb(main):020> dup.valid?
=> false
irb(main):021> dup.errors.full_messages
=> ["Tag has already been taken"]
```

ทดสอบลบแท็กออกจาก record เดียว (ไม่กระทบ record อื่นที่ผูกแท็กเดียวกันอยู่):

```irb
irb(main):022> post1.tags.delete(ruby_tag)
irb(main):023> post1.tags.pluck(:name)
=> ["rails"]
irb(main):024> Tagging.count
=> 4
```

ทดสอบลบ `Post` แล้วดู `Tagging` ที่เกี่ยวข้องถูกลบตามด้วย `dependent: :destroy` แต่ `Tag`
เองไม่ถูกลบ (เพราะแท็กอาจยังผูกกับ record อื่นอยู่):

```irb
irb(main):025> post2.destroy
irb(main):026> Tagging.where(taggable_type: "Post").count
=> 1
irb(main):027> Tag.exists?(rails_tag.id)
=> true   # <- rails_tag ยังอยู่ ไม่ถูกลบตาม แม้ post2 ที่เคยใช้แท็กนี้จะถูกลบไปแล้ว
```

`post2.destroy` ลบ `post2` และ `Tagging` ที่ `taggable` ชี้มาที่ `post2` ไปด้วย (ตาม
`dependent: :destroy` ที่ตั้งไว้ใน `Post#taggings`) แต่ `rails_tag` เองยังอยู่ครบ เพราะยังมี
`post1` ผูกอยู่ — ตรงตามพฤติกรรมที่ระบบแท็กควรเป็น: **ลบความสัมพันธ์ (tagging) ตาม record ที่ถูกลบ
แต่ตัวแท็กเองยังคงอยู่ให้ record อื่นใช้ต่อได้เสมอ**

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่มระบบ "กดถูกใจ" (`Like`) แบบ polymorphic ที่ `Post` และ `Comment` กดถูกใจได้ทั้งคู่
   (`Like belongs_to :likeable, polymorphic: true`) พร้อมป้องกันไม่ให้ผู้ใช้คนเดียวกัน (สมมติมี
   `User` จาก Step 325 อยู่แล้ว) กดถูกใจ record เดียวกันซ้ำสองครั้ง (ใบ้: ต้องมี `belongs_to
   :user` เพิ่มใน `Like` และทำ unique index/validation แบบเดียวกับที่ทำใน `Tagging`)
2. ลองแปลง `has_and_belongs_to_many` ธรรมดา (ไม่ใช่ polymorphic) ระหว่าง `Product` กับ
   `Category` ให้กลายเป็น `has_many :through` ผ่าน Model `Categorization` ที่มี column
   `featured:boolean` (บอกว่าสินค้านี้เป็นสินค้าแนะนำในหมวดนั้นหรือไม่) — ฝึกเขียน migration
   แปลงข้อมูลแบบ `up`/`down` เหมือน Step 324 ให้ครบ พร้อม backfill ข้อมูลเดิม
3. ขยาย `Employee` org chart จาก Step 326 ให้มี method `ancestors` (คืนลิสต์หัวหน้าทุกระดับ
   ขึ้นไปจนถึงบนสุด เรียงจากใกล้ตัวไปไกลตัว) และเขียนเทียบกับ `all_subordinates` ที่มีอยู่แล้ว
   ทดสอบกับผังองค์กรที่มีอย่างน้อย 4 ระดับ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- ความสัมพันธ์แบบ **many-to-many** (กลุ่มต่อกลุ่ม) ทำด้วย `belongs_to`/`has_many` เดี่ยวๆ ไม่ได้
  เพราะ foreign key column เก็บได้แค่ค่าเดียว ต้องมี **ตารางกลาง (join table)** มาคั่นเสมอ
- **`has_and_belongs_to_many`** สร้างตารางกลางแบบ `id: false` ไม่มี column พิเศษ ใช้งานสั้นแต่
  จำกัดมาก (เพิ่มข้อมูล/validation/callback บนความสัมพันธ์ไม่ได้) — ในทางปฏิบัติแนะนำให้เริ่มด้วย
  **`has_many :through`** แทนแทบทุกกรณี เพราะตัวกลางเป็น Model เต็มรูปแบบที่ใส่ column, validation,
  callback ได้อิสระ
- แปลง habtm เป็น `has_many :through` ทำได้จริงด้วย migration ที่สร้างตารางใหม่, `INSERT ...
  SELECT` backfill ข้อมูลเดิมมา, แล้วค่อย `drop_table` ตารางเก่าทิ้ง (ต้องเขียน `up`/`down`
  แยกเองเสมอเมื่อมีการย้าย/ลบข้อมูลแบบนี้)
- **Self-referential association** มีสองรูปแบบ: แบบ **`has_many :through`** เมื่อความสัมพันธ์เป็น
  many-to-many (เช่นระบบ follow ที่ต้องมี Model กลางอย่าง `Follow` พร้อม `class_name`,
  `foreign_key`, `source:`, `inverse_of:` ครบ) และแบบ **`belongs_to`/`has_many` ตรงๆ** เมื่อเป็น
  หนึ่งต่อกลุ่มธรรมดา (เช่นผังองค์กร `Employee belongs_to :manager`)
- **Polymorphic association** (`belongs_to ..., polymorphic: true` คู่กับ `has_many ..., as:
  ...`) ทำให้ Model หนึ่งอ้างอิงไปยังได้หลาย Model ผ่านสอง column (`xxx_type`, `xxx_id`) แทนที่
  จะต้องมี foreign key แยกทีละ column — แต่แลกมาด้วยการ**ไม่มี foreign key constraint จริงในระดับ
  ฐานข้อมูล** ทำให้ data integrity ต้องพึ่งพาโค้ดระดับแอปพลิเคชันเป็นหลัก ต้องระวังเป็นพิเศษ
- **`has_one :through`** ใช้เมื่อความสัมพันธ์ทุกขั้นเป็นหนึ่งต่อหนึ่งตลอดสาย ทำให้เดินลัดข้าม
  ตัวกลางได้ในคำสั่งเดียว
- ตารางสรุปการตัดสินใจ: **habtm** เมื่อความสัมพันธ์เป็นแค่ลิงก์ล้วนๆ ไม่มีวันต้องการข้อมูลเพิ่ม,
  **`has_many :through`** เป็นค่า default ที่ปลอดภัยกว่าสำหรับ many-to-many ทั่วไป,
  **polymorphic** เมื่อ record หนึ่งต้องอ้างอิงไปยังหลาย Model ที่ไม่เกี่ยวข้องกันโดยตรง (พร้อม
  ยอมรับ tradeoff เรื่อง data integrity), และ **STI** (แนะนำแบบสั้นๆ) เมื่อมีหลาย Model โครงสร้าง
  คล้ายกันมากจนอยากรวมเป็นตารางเดียว
- แบบฝึกหัดรวบยอด: ระบบ tag แบบ polymorphic ผสม `has_many :through` (`Tagging` เป็นทั้งตัวกลาง
  many-to-many และ polymorphic พร้อมกัน) ซึ่งเป็นรูปแบบที่ใช้จริงบ่อยที่สุดรูปแบบหนึ่งในแอป Rails
  พร้อมเทคนิคใหม่ **`source_type:`** สำหรับกรองชนิดตอนเดินย้อนกลับจากฝั่ง polymorphic

**ต่อไป (Part 034):** ตอนนี้เรามี association ครบทุกแบบที่ใช้บ่อยในงานจริงแล้ว แต่ตลอดสาม Part ที่
ผ่านมาเราใช้ `where`, `pluck`, `includes` แบบผิวเผินเท่านั้น Part 034 จะพาไปเจาะลึก **Query
Interface** ของ ActiveRecord ทั้งหมด: `where` แบบละเอียด (เงื่อนไขซับซ้อน, `or`, `not`),
`order`, `joins` เทียบกับ `includes`, ความต่างของ `includes`/`preload`/`eager_load` แบบเจาะลึก
(ที่ Part 027 และ Part นี้บอกไว้ว่าจะมาเรียนเต็มๆ ที่นี่), และ **`scope`** สำหรับเก็บเงื่อนไข
query ที่ใช้ซ้ำบ่อยๆ ไว้เป็น method อ่านง่ายบน Model โดยตรง
