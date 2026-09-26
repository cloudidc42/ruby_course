# Part 009: OOP เบื้องต้น — Class, Object, attr_accessor, initialize, instance/class variable

> **Step ครอบคลุมใน Part นี้:** Step 81–90
> **ระดับ:** เริ่มต้น–กลาง (ต้องผ่าน Part 001–008 มาก่อน โดยเฉพาะเรื่อง Method และ Block)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x (ตัวอย่างทั้งหมดทดสอบบน Ruby >= 3.1 ก็ใช้งานได้)

## สารบัญของ Part นี้

- Step 81: Object-Oriented Programming คืออะไร และทำไม "ทุกอย่างใน Ruby คือ Object"
- Step 82: การประกาศ Class, method `initialize` และ `.new`
- Step 83: Instance Variable (`@var`) — scope และพฤติกรรม `nil` โดยปริยาย
- Step 84: Instance Method เพิ่มเติม, getter/setter ด้วยมือ vs `attr_reader`/`attr_writer`/`attr_accessor`
- Step 85: Class Method (`def self.method_name`) และ `class << self`
- Step 86: Class Variable (`@@var`) กับปัญหาที่มากับมัน และ Class Instance Variable
- Step 87: Override `to_s` และ `inspect`
- Step 88: ความแตกต่างของ `==`, `eql?`, `equal?` และการ override `==`
- Step 89: Constant ภายใน Class
- Step 90: แบบฝึกหัดโปรเจกต์ — สร้าง `BankAccount` class ครบระบบ

---

## Step 81: Object-Oriented Programming คืออะไร และทำไม "ทุกอย่างใน Ruby คือ Object"

จนถึงตอนนี้เราเขียนโปรแกรมแบบ **procedural** มาตลอด คือมี method แยกๆ ทำงานกับข้อมูล
(Array, Hash, String) ที่ส่งผ่านไปมา เมื่อโปรแกรมใหญ่ขึ้น การจัดการข้อมูลกับพฤติกรรม
(behavior) ที่เกี่ยวข้องกันแยกจากกันจะเริ่มยุ่งเหยิง

**Object-Oriented Programming (OOP)** คือแนวคิดการเขียนโปรแกรมที่รวม **ข้อมูล (state)**
กับ **พฤติกรรมที่ทำงานกับข้อมูลนั้น (behavior)** ไว้เป็นก้อนเดียวกัน เรียกว่า **Object**
โดย Object แต่ละตัวถูกสร้างจากพิมพ์เขียวที่เรียกว่า **Class**

แนวคิดนี้ช่วยให้:

1. **จัดกลุ่มข้อมูลกับ logic ที่เกี่ยวข้องไว้ด้วยกัน** — เช่น ข้อมูลบัญชีธนาคาร (ยอดเงิน,
   เลขบัญชี) ควรอยู่คู่กับ method ฝาก/ถอน ไม่ใช่กระจัดกระจายเป็น Hash กับ method แยกกัน
2. **ซ่อนรายละเอียดภายใน (encapsulation)** — ผู้ใช้ Object ไม่จำเป็นต้องรู้ว่าข้างในเก็บ
   ข้อมูลอย่างไร รู้แค่ว่าเรียก method ไหนแล้วได้ผลลัพธ์อะไร
3. **นำโค้ดกลับมาใช้ซ้ำได้ผ่านการสืบทอด (inheritance)** — จะเรียนใน Part 010
4. **ออกแบบระบบใหญ่ให้เป็นระเบียบ** — Rails ทั้ง framework สร้างอยู่บนพื้นฐาน OOP เกือบทั้งหมด
   (Model, Controller, Mailer ล้วนเป็น Class)

### ทุกอย่างใน Ruby คือ Object

จุดเด่นสำคัญของ Ruby เมื่อเทียบกับภาษาอื่น (เช่น Java, C++ ที่มี primitive type แยกจาก
Object) คือ **ทุกค่าใน Ruby เป็น Object หมดโดยไม่มีข้อยกเว้น** แม้แต่ตัวเลข, `nil`,
`true`/`false`, หรือแม้แต่ Class เองก็ยังเป็น Object

```ruby
# frozen_string_literal: true

puts 42.class          # => Integer
puts 42.is_a?(Object)  # => true
puts 3.14.class         # => Float
puts "hello".class      # => String
puts nil.class          # => NilClass
puts true.class         # => TrueClass
puts false.class        # => FalseClass
puts [1, 2].class       # => Array
puts({ a: 1 }.class)    # => Hash
puts :symbol.class      # => Symbol

# แม้แต่ Class เอง ก็เป็น instance ของ Class อีกที
puts String.class       # => Class
puts Integer.class      # => Class
puts Class.class        # => Class (Class เป็น instance ของตัวเอง!)
```

เพราะทุกอย่างเป็น Object เราจึงเรียก method บนตัวเลขได้ตรงๆ โดยไม่ต้องมี syntax พิเศษ
เหมือนภาษาอื่นที่ต้องเขียน `Math.abs(-5)` แบบแยก function กับข้อมูล:

```ruby
puts(-5.abs)        # => 5   (เรียก method .abs บน Integer object ตรงๆ)
puts 5.times.to_a    # => [0, 1, 2, 3, 4]
puts 10.even?        # => true
```

### ลำดับชั้นของ Object ใน Ruby

```ruby
puts 42.class.ancestors
# => [Integer, Numeric, Comparable, Object, Kernel, BasicObject]
```

ทุก Class ใน Ruby (ยกเว้น `BasicObject`) สืบทอดมาจาก `Object` ในที่สุด และ `Object` เอง
รวม module ชื่อ `Kernel` เข้ามา (method อย่าง `puts`, `p`, `require` ที่เราใช้มาตลอด
จริงๆ แล้วเป็น method ใน `Kernel` module ที่ถูกรวมเข้าไปใน `Object` ทำให้เรียกได้จากทุกที่)

**สรุปแนวคิดสำคัญของ Step นี้:** OOP คือการมองโปรแกรมเป็น "สิ่งของ" (Object) ที่มีทั้งข้อมูล
และพฤติกรรมในตัวเอง และเพราะ Ruby ยึดหลัก "ทุกอย่างคือ Object" การเรียนรู้ OOP ให้แน่น
จึงเท่ากับเข้าใจแก่นของภาษา Ruby ทั้งหมด — เป็นพื้นฐานที่จำเป็นอย่างยิ่งก่อนไปเรียน Rails
เพราะ ActiveRecord Model, Controller ทุกตัวคือ Class ที่เราจะสร้างขึ้นเอง

---

## Step 82: การประกาศ Class, method `initialize` และ `.new`

### ประกาศ Class อย่างง่ายที่สุด

```ruby
# frozen_string_literal: true

class Dog
end

my_dog = Dog.new
puts my_dog.class   # => Dog
puts my_dog          # => #<Dog:0x00007f8b1a0a1234>
```

**อธิบาย:**

- คำสั่ง `class Dog ... end` ประกาศ Class ชื่อ `Dog` — ชื่อ Class ต้องขึ้นต้นด้วยตัวพิมพ์ใหญ่
  เสมอ (เป็น convention บังคับของ Ruby ไม่ใช่แค่ style)
- `Dog.new` คือการสร้าง **instance** (หรือ Object) ตัวใหม่จาก Class `Dog` — เรียกขั้นตอนนี้ว่า
  **instantiation**
- `my_dog` คือตัวแปรที่เก็บ **reference** ไปยัง Object ที่เพิ่งสร้าง
- เมื่อ `puts` object ที่ยังไม่ได้ override `to_s` จะได้ string default ที่บอกชื่อ Class กับ
  memory address (object id) เช่น `#<Dog:0x00007f8b1a0a1234>`

### `initialize` — Constructor ของ Ruby

เวลาสร้าง Object ใหม่ เรามักอยากกำหนดค่าเริ่มต้นให้มันทันที (เช่น สุนัขต้องมีชื่อ)
Ruby ทำสิ่งนี้ผ่าน method พิเศษชื่อ `initialize`:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, breed)
    @name = name
    @breed = breed
    puts "สร้างสุนัขชื่อ #{@name} พันธุ์ #{@breed} เรียบร้อย"
  end
end

rex = Dog.new("Rex", "Labrador")
# => สร้างสุนัขชื่อ Rex พันธุ์ Labrador เรียบร้อย
```

**กายวิภาคของกลไกนี้:**

- `initialize` เป็น method ที่ Ruby เรียกให้ **อัตโนมัติ** ทุกครั้งที่มีการเรียก `.new`
- argument ที่ส่งเข้า `.new("Rex", "Labrador")` จะถูกส่งต่อไปยัง `initialize` โดยตรง
  (จำนวนและชนิดของ argument ต้องตรงกับที่ `initialize` รับ เหมือน method ทั่วไปที่เรียนใน
  Part 007 — รองรับ default value, keyword argument, splat ได้เหมือนกันหมด)
- `.new` ไม่สามารถถูกเรียกตรงๆ แทน `initialize` ได้ — `.new` เป็น **class method** ที่ Ruby
  ให้มาโดยอัตโนมัติ ทำหน้าที่จองหน่วยความจำสำหรับ Object ใหม่ แล้วเรียก `initialize`
  (ซึ่งเป็น **instance method**) ให้บน Object นั้นอีกที
- `@name` และ `@breed` คือ **instance variable** ซึ่งจะอธิบายละเอียดใน Step ถัดไป

### `initialize` พร้อม default value และ keyword argument

หลักการเดียวกับการนิยาม method ปกติทุกประการ:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, breed: "ไม่ทราบพันธุ์", age: 0)
    @name = name
    @breed = breed
    @age = age
  end
end

rex = Dog.new("Rex", breed: "Labrador", age: 3)
milo = Dog.new("Milo")  # ใช้ default: breed = "ไม่ทราบพันธุ์", age = 0
```

### `initialize` ไม่จำเป็นต้องมีก็ได้

```ruby
class Empty
end

e = Empty.new  # ทำงานได้ปกติ ถ้าไม่มี initialize, Ruby ใช้ default ที่ไม่รับ argument
```

แต่ถ้าไม่มี `initialize` แล้วพยายามส่ง argument เข้า `.new` จะเกิด error:

```ruby
class Empty
end

Empty.new("something")
# => ArgumentError (wrong number of arguments (given 1, expected 0))
```

**หลักปฏิบัติ:** เกือบทุก Class ที่มีความหมายจริงจังควรมี `initialize` เพื่อกำหนดสถานะ
เริ่มต้นให้ชัดเจนตั้งแต่วินาทีที่ Object ถูกสร้าง (แนวคิดนี้เรียกว่า Object ต้องอยู่ใน
**สถานะที่ valid เสมอ** — never allow an object to exist in an invalid state)

---

## Step 83: Instance Variable (`@var`) — scope และพฤติกรรม `nil` โดยปริยาย

### Instance Variable คืออะไร

**Instance variable** คือตัวแปรที่ขึ้นต้นด้วย `@` (เช่น `@name`, `@balance`) ใช้เก็บ
**สถานะ (state)** ของ Object แต่ละตัว — Object คนละตัวจาก Class เดียวกัน มี instance
variable เป็นของตัวเอง แยกจากกันโดยสิ้นเชิง

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name
  end

  def bark
    puts "#{@name}: โฮ่ง โฮ่ง!"
  end
end

rex = Dog.new("Rex")
milo = Dog.new("Milo")

rex.bark   # => Rex: โฮ่ง โฮ่ง!
milo.bark  # => Milo: โฮ่ง โฮ่ง!
```

`rex` และ `milo` เป็น Object คนละตัว แต่ละตัวมี `@name` เป็นของตัวเอง ไม่ปะปนกัน
นี่คือหัวใจของ encapsulation — ข้อมูลถูกเก็บแยกไว้ในแต่ละ Object

### Scope ของ Instance Variable

instance variable **เข้าถึงได้จากทุก instance method ภายใน Class เดียวกัน** โดยไม่ต้องส่ง
ผ่าน argument (ต่างจาก local variable ที่เห็นเฉพาะใน scope ของ method ที่ประกาศ):

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name  # ตั้งค่าใน initialize
  end

  def introduce
    "สวัสดี ฉันชื่อ #{@name}"  # เรียกใช้ได้ใน method อื่นโดยไม่ต้องส่ง argument
  end

  def rename(new_name)
    @name = new_name  # แก้ไขค่าได้จาก method ไหนก็ได้ในคลาสเดียวกัน
  end
end

rex = Dog.new("Rex")
puts rex.introduce   # => สวัสดี ฉันชื่อ Rex
rex.rename("Rexy")
puts rex.introduce   # => สวัสดี ฉันชื่อ Rexy
```

**สิ่งที่ instance variable ไม่ทำ:** มันไม่ถูกแชร์ข้ามไปยัง instance อื่น และไม่มองเห็น
จากภายนอก Object โดยตรง (ต้องผ่าน method เท่านั้น — จะอธิบายเรื่องนี้ต่อใน Step 84)

### พฤติกรรม `nil` โดยปริยาย (สำคัญมาก และเป็นบ่อเกิด bug บ่อยที่สุดของมือใหม่)

ต่างจาก local variable ที่ถ้าอ้างถึงตัวแปรที่ไม่เคยประกาศจะเกิด `NameError` ทันที
**instance variable ที่ยังไม่เคยถูกกำหนดค่า จะคืนค่า `nil` เฉยๆ โดยไม่ error:**

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name
    # ตั้งใจไม่กำหนด @age ตอนสร้าง
  end

  def show_age
    puts "อายุ: #{@age}"
  end
end

rex = Dog.new("Rex")
rex.show_age   # => อายุ: (บรรทัดว่าง เพราะ @age เป็น nil ไม่ใช่ error!)

puts rex.instance_variable_get(:@age).nil?  # => true
```

เทียบกับ local variable ที่จะ error ทันทีถ้าไม่เคยถูกกำหนดค่า:

```ruby
def show
  puts undeclared_variable  # NameError: undefined local variable or method
end
```

**ผลที่ตามมาในทางปฏิบัติ:** เพราะ instance variable ที่พิมพ์ชื่อผิด (typo) เช่น `@nmae`
แทน `@name` จะไม่ error แต่จะกลายเป็น `nil` เงียบๆ ทำให้บั๊กจากการพิมพ์ผิดตรวจจับยากกว่า
local variable มาก **แนวปฏิบัติที่ดี:** กำหนดค่าเริ่มต้นให้ instance variable ที่สำคัญทุกตัว
ใน `initialize` เสมอ แม้จะเป็นค่า default ก็ตาม เพื่อป้องกันปัญหานี้

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, age: 0)
    @name = name
    @age = age   # กำหนดค่า default ชัดเจน ไม่ปล่อยให้เป็น nil โดยไม่ตั้งใจ
  end
end
```

### Instance Variable นอก Class ก็มีค่า default เป็น `nil` เหมือนกัน

แม้แต่ระดับ top-level (นอก Class ใดๆ) `self` ก็ยังเป็น Object หนึ่ง (คือ `main`) จึงใช้
instance variable ได้เหมือนกัน:

```ruby
@score = 10
puts @score          # => 10
puts @undefined_ivar # => (nil, ไม่ error)
```

---

## Step 84: Instance Method เพิ่มเติม, getter/setter ด้วยมือ vs `attr_reader`/`attr_writer`/`attr_accessor`

### ปัญหา: Instance Variable เข้าถึงจากภายนอกไม่ได้โดยตรง

ลองสังเกตโค้ดนี้:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name
  end
end

rex = Dog.new("Rex")
puts rex.name
# => NoMethodError (undefined method `name' for #<Dog:0x...>)
```

ถึงแม้ `@name` จะถูกตั้งค่าไว้ข้างในแล้ว แต่ **โค้ดภายนอก Class เรียก instance variable
ตรงๆ ไม่ได้เลย** — นี่คือ encapsulation ที่ Ruby บังคับใช้ ต้องเข้าถึงผ่าน **method** เท่านั้น

### แก้ปัญหาด้วยการเขียน getter method เอง (ด้วยมือ)

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name
  end

  # getter method: อ่านค่า @name ออกมา
  def name
    @name
  end
end

rex = Dog.new("Rex")
puts rex.name   # => Rex
```

`name` เป็นเพียง instance method ธรรมดาที่ return ค่า `@name` ออกมา — ไม่มีเวทมนตร์อะไร
เพิ่มเติมเลย

### เพิ่ม setter method เอง (ด้วยมือ)

การเขียน method ให้ **แก้ไขค่า** จากภายนอกได้ ใช้ syntax พิเศษ: ชื่อ method ต้องลงท้ายด้วย
`=` และ Ruby จะอนุญาตให้เรียกแบบ "assignment" ได้ (`obj.attr = value`):

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name)
    @name = name
  end

  def name
    @name
  end

  # setter method: ชื่อ method ต้องลงท้ายด้วย =
  def name=(new_name)
    @name = new_name
  end
end

rex = Dog.new("Rex")
puts rex.name     # => Rex

rex.name = "Rexy"  # จริงๆ แล้วคือการเรียก rex.name=("Rexy") แต่ Ruby ให้เขียนแบบ syntax sugar นี้ได้
puts rex.name     # => Rexy
```

**ข้อสังเกตสำคัญ:** `rex.name = "Rexy"` ที่ดูเหมือน assignment ธรรมดา แท้จริงแล้วคือการเรียก
method `name=` โดยส่ง `"Rexy"` เป็น argument — Ruby แปล syntax `obj.attr = value` ให้เป็น
`obj.attr=(value)` โดยอัตโนมัติ (ต้องไม่มีช่องว่างระหว่างชื่อ method กับ `=` ตอนนิยาม แต่ตอน
เรียกใช้ใส่ช่องว่างรอบ `=` ได้ตามสบาย)

### getter/setter method ที่มี validation หรือ logic เพิ่มเติม

จุดแข็งของการเขียน getter/setter เองคือใส่ logic เพิ่มได้ เช่น validate ค่าก่อนตั้ง:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, age)
    @name = name
    self.age = age  # เรียกผ่าน setter เพื่อให้ validate ตั้งแต่ตอนสร้าง
  end

  def age
    @age
  end

  def age=(new_age)
    raise ArgumentError, "อายุต้องไม่ติดลบ" if new_age.negative?

    @age = new_age
  end
end

rex = Dog.new("Rex", 3)
puts rex.age     # => 3

rex.age = -5
# => ArgumentError (อายุต้องไม่ติดลบ)
```

> **ข้อควรระวัง:** ใน `initialize` เราเรียก `self.age = age` (ระบุ `self.` ชัดเจน) แทนที่จะ
> เขียน `age = age` เฉยๆ เพราะถ้าไม่มี `self.` Ruby จะตีความว่าเป็นการสร้าง **local variable**
> ชื่อ `age` ขึ้นมาใหม่ ไม่ใช่การเรียก setter method — เป็นกับดักคลาสสิกของ Ruby ที่ต้องจำให้ขึ้นใจ

### `attr_reader`, `attr_writer`, `attr_accessor` — ลดโค้ดซ้ำซ้อน

การเขียน getter/setter สำหรับทุก attribute ที่ไม่ต้องมี logic พิเศษ เป็นโค้ดที่ซ้ำซากมาก
Ruby จึงมี method ระดับ Class ให้ช่วย generate getter/setter อัตโนมัติ:

```ruby
# frozen_string_literal: true

class Dog
  attr_reader :name    # generate เฉพาะ getter (อ่านได้อย่างเดียว)
  attr_writer :owner    # generate เฉพาะ setter (เขียนได้อย่างเดียว)
  attr_accessor :breed  # generate ทั้ง getter และ setter

  def initialize(name, owner, breed)
    @name = name
    @owner = owner
    @breed = breed
  end
end

rex = Dog.new("Rex", "สมชาย", "Labrador")

puts rex.name          # => Rex (อ่านผ่าน attr_reader ได้)
# rex.name = "Rexy"    # => NoMethodError เพราะ attr_reader ไม่ generate setter ให้

rex.owner = "สมหญิง"    # เขียนผ่าน attr_writer ได้
# puts rex.owner        # => NoMethodError เพราะ attr_writer ไม่ generate getter ให้

puts rex.breed          # => Labrador (attr_accessor อ่านได้)
rex.breed = "Poodle"     # attr_accessor เขียนได้ด้วย
puts rex.breed          # => Poodle
```

**ตารางสรุปการเลือกใช้:**

| Macro | Generate getter | Generate setter | ใช้เมื่อ |
|---|---|---|---|
| `attr_reader :x` | ✅ | ❌ | ค่าที่อ่านได้อย่างเดียว ห้ามแก้จากภายนอก (เช่น ID, วันที่สร้าง) |
| `attr_writer :x` | ❌ | ✅ | ค่าที่ตั้งได้อย่างเดียว ไม่ต้องอ่านกลับ (พบน้อยในทางปฏิบัติ) |
| `attr_accessor :x` | ✅ | ✅ | ค่าที่อ่านและแก้ไขได้อย่างอิสระ ไม่มี validation พิเศษ |

### สิ่งที่ `attr_accessor :name` generate ให้เท่ากับอะไร (ทำมือเทียบ)

การเรียก `attr_accessor :name` เพียงบรรทัดเดียว เทียบเท่ากับการเขียน method 2 ตัวนี้เอง:

```ruby
# attr_accessor :name  เทียบเท่ากับ...

def name
  @name
end

def name=(value)
  @name = value
end
```

พูดให้ชัดคือ `attr_accessor`/`attr_reader`/`attr_writer` เป็นเพียง **method ระดับ Class**
(technically เป็น method ของ `Module`) ที่ถูกเรียกตอน define class เพื่อ**นิยาม method
ใหม่ๆ ให้เราแบบอัตโนมัติ** ด้วย metaprogramming ภายใน — ไม่ใช่ keyword พิเศษของภาษา

**ทำไมยังต้องรู้วิธีเขียนเองด้วยมือ ทั้งที่มี macro ให้ใช้แล้ว:**

1. เมื่อไหร่ก็ตามที่ต้องการ **validation หรือ logic เพิ่มเติม** ในการตั้งค่า (เช่น
   ตัวอย่าง `age=` ด้านบน) ต้องเขียน setter เองเสมอ — จะใช้ `attr_writer`/`attr_accessor`
   ไม่ได้เพราะมันแค่ assign ค่าตรงๆ โดยไม่ตรวจสอบอะไรเลย
2. บางครั้งต้องการ getter ที่ **คำนวณค่า** แทนที่จะ return `@var` ตรงๆ เช่น
   `def full_name; "#{@first_name} #{@last_name}"; end` — สิ่งนี้ macro ทำให้ไม่ได้
3. เข้าใจว่า Rails เองก็ใช้แนวคิดเดียวกันนี้เต็มไปหมด (`attribute`, `has_many` ล้วนเป็น
   method ที่ generate method อื่นให้ ไม่ใช่ syntax พิเศษ)

```ruby
# frozen_string_literal: true

class Person
  attr_reader :first_name, :last_name  # ทั้งสองอ่านได้อย่างเดียว
  attr_accessor :nickname               # แก้ไขได้อิสระ

  def initialize(first_name, last_name, nickname: nil)
    @first_name = first_name
    @last_name = last_name
    @nickname = nickname
  end

  # getter ที่คำนวณค่า ไม่ได้ผูกกับ instance variable ตัวเดียวตรงๆ — เขียนมือเสมอ
  def full_name
    "#{@first_name} #{@last_name}"
  end
end

p1 = Person.new("สมชาย", "ใจดี", nickname: "ชาย")
puts p1.full_name  # => สมชาย ใจดี
puts p1.nickname   # => ชาย
p1.nickname = "ชายชาย"
puts p1.nickname   # => ชายชาย
```

`attr_reader/writer/accessor` รับได้หลาย symbol พร้อมกันในบรรทัดเดียว เช่น
`attr_accessor :name, :age, :email` — เป็นวิธีที่นิยมที่สุดในการเขียน Class ทั่วไป

---

## Step 85: Class Method (`def self.method_name`) และ `class << self`

### Instance Method vs Class Method

จนถึงตอนนี้ method ทุกตัวที่เขียนเป็น **instance method** — เรียกได้ก็ต่อเมื่อสร้าง
instance ขึ้นมาก่อนด้วย `.new` เช่น `rex.bark`

แต่บางครั้งเราต้องการ method ที่เรียกจาก **ตัว Class เอง** โดยไม่ต้องสร้าง instance
ก่อน เรียกว่า **Class Method** (แนวคิดเดียวกับ `static method` ในภาษาอื่น)

ตัวอย่างที่คุ้นเคยอยู่แล้ว: `Dog.new`, `Array.new`, `Time.now`, `Date.today` ล้วนเป็น
Class Method ทั้งสิ้น (`.new` เองก็เป็น class method ที่ Ruby ให้มาโดยอัตโนมัติ!)

### นิยาม Class Method ด้วย `def self.`

```ruby
# frozen_string_literal: true

class Dog
  SPECIES = "Canis familiaris"

  def self.species_info
    "สุนัขทุกตัวเป็นสปีชีส์: #{SPECIES}"
  end

  def initialize(name)
    @name = name
  end

  def bark
    puts "#{@name}: โฮ่ง!"
  end
end

# เรียก class method ผ่านตัว Class ตรงๆ ไม่ต้องสร้าง instance
puts Dog.species_info
# => สุนัขทุกตัวเป็นสปีชีส์: Canis familiaris

# เรียกผ่าน instance ไม่ได้ (นี่คือ class method ไม่ใช่ instance method)
rex = Dog.new("Rex")
rex.species_info
# => NoMethodError (undefined method `species_info' for an instance of Dog)
```

**อธิบาย:** การเติม `self.` หน้าชื่อ method ตอนนิยาม บอก Ruby ว่า "method นี้เป็นของตัว
Class เอง ไม่ใช่ของ instance" — `self` ในบริบทนี้ (ระดับบนสุดของ class body ตอนกำลัง
นิยาม method) หมายถึงตัว Class นั้นเอง (`Dog`)

### ตัวอย่างที่ใช้บ่อย: Class Method สำหรับสร้าง Object ด้วยวิธีพิเศษ (Factory Method)

```ruby
# frozen_string_literal: true

class Dog
  attr_reader :name, :breed

  def initialize(name, breed)
    @name = name
    @breed = breed
  end

  # class method ที่ทำหน้าที่เป็น "ทางลัด" สร้าง instance แบบพิเศษ
  def self.stray(name)
    new(name, "ไม่ทราบพันธุ์ (สุนัขจรจัด)")
  end
end

rex = Dog.new("Rex", "Labrador")   # สร้างแบบปกติ
milo = Dog.stray("Milo")           # สร้างผ่าน factory method

puts milo.breed  # => ไม่ทราบพันธุ์ (สุนัขจรจัด)
```

สังเกตว่าภายใน class method เราเรียก `new(...)` เฉยๆ โดยไม่ต้องเขียน `Dog.new` เพราะ
`self` ในบริบทนี้คือ `Dog` อยู่แล้ว — pattern นี้เรียกว่า **factory method pattern**
และเป็นแนวทางที่ Rails ใช้เยอะมาก (เช่น `User.find_by(email: "...")` หรือ
`Model.create(...)`)

### `class << self` — วิธีนิยาม Class Method หลายตัวพร้อมกัน

เมื่อมี class method หลายตัว การเขียน `def self.` ซ้ำๆ ทุกบรรทัดอาจดูรก มี syntax
ทางเลือกที่เรียกว่า **singleton class syntax** (`class << self`) ที่รวม class method
ไว้เป็นกลุ่มเดียว:

```ruby
# frozen_string_literal: true

class Dog
  class << self
    def species_info
      "สุนัขทุกตัวเป็นสปีชีส์: Canis familiaris"
    end

    def stray(name)
      new(name, "ไม่ทราบพันธุ์")
    end

    def count
      @count ||= 0
    end
  end

  attr_reader :name, :breed

  def initialize(name, breed)
    @name = name
    @breed = breed
  end
end

puts Dog.species_info
puts Dog.stray("Milo").breed
```

`class << self ... end` เปิด "singleton class" ของ `Dog` ขึ้นมา — ทุก method ที่นิยาม
ข้างในนี้จะกลายเป็น class method ของ `Dog` โดยอัตโนมัติ โดยไม่ต้องเขียน `self.` ซ้ำทุกบรรทัด
วิธีนี้เป็นที่นิยมในโค้ด Ruby ระดับ production โดยเฉพาะเมื่อมี class method จำนวนมาก
แต่สำหรับ class method 1-2 ตัว การใช้ `def self.method_name` ตรงๆ อ่านง่ายกว่าและนิยมกว่า
ในกรณีทั่วไป

> **สรุปเทียบ:** `def self.method_name` เหมาะกับ class method จำนวนน้อย อ่านง่าย
> ส่วน `class << self` เหมาะกับกรณีมี class method จำนวนมากและต้องการแยกกลุ่มให้ชัดเจน
> ทั้งสองแบบทำงานเหมือนกันทุกประการ เป็นเพียงคนละ syntax

---

## Step 86: Class Variable (`@@var`) กับปัญหาที่มากับมัน และ Class Instance Variable

### Class Variable คืออะไร

**Class variable** ขึ้นต้นด้วย `@@` (เช่น `@@count`) เป็นตัวแปรที่ **ใช้ร่วมกันระหว่าง
ทุก instance ของ Class นั้น** (ต่างจาก instance variable `@var` ที่แยกกันในแต่ละ instance)

```ruby
# frozen_string_literal: true

class Dog
  @@total_dogs = 0

  def initialize(name)
    @name = name
    @@total_dogs += 1
  end

  def self.total_dogs
    @@total_dogs
  end
end

Dog.new("Rex")
Dog.new("Milo")
Dog.new("Buddy")

puts Dog.total_dogs  # => 3
```

ทุกครั้งที่สร้าง `Dog` ใหม่ `@@total_dogs` (ซึ่งเป็นตัวแปรร่วมกันของทุก instance) จะ
เพิ่มค่าขึ้น 1 — ดูเผินๆ เหมือนเป็นวิธีที่เหมาะกับการนับจำนวน instance ทั้งหมด

### ปัญหาของ Class Variable: รั่วไหลข้าม Subclass โดยไม่ตั้งใจ

ปัญหาใหญ่ที่สุดของ `@@var` คือ **มันถูกแชร์ข้ามไปยัง subclass ทุกตัวด้วย** (จะเรียนเรื่อง
inheritance เต็มๆ ใน Part 010 แต่ขอยกตัวอย่างสั้นๆ ให้เห็นปัญหานี้ก่อน):

```ruby
# frozen_string_literal: true

class Animal
  @@count = 0

  def initialize
    @@count += 1
  end

  def self.count
    @@count
  end
end

class Dog < Animal
end

class Cat < Animal
end

Dog.new
Dog.new
Cat.new

puts Dog.count   # => 3  (!!) ไม่ใช่ 2 ตามที่คาดหวัง
puts Cat.count   # => 3  (!!) ไม่ใช่ 1 ตามที่คาดหวัง
puts Animal.count # => 3
```

**นี่คือ bug คลาสสิกของ `@@var`:** เพราะ `@@count` เป็นตัวแปรตัวเดียวกันที่ **แชร์กันทุก
Class ในสายการสืบทอด** (`Animal`, `Dog`, `Cat` ทั้งหมดมองเห็น `@@count` ตัวเดียวกัน)
ทำให้การนับจำนวนแยกตาม subclass ทำไม่ได้เลยถ้าใช้ `@@var` — ผลลัพธ์ที่ได้ผิดจากที่
นักพัฒนาส่วนใหญ่คาดหวังอย่างสิ้นเชิง

**ด้วยเหตุนี้ ชุมชน Ruby จึงแนะนำให้ "หลีกเลี่ยงการใช้ Class Variable (`@@var`) เกือบทุก
กรณี"** เพราะพฤติกรรมที่ไม่คาดคิดนี้ทำให้เกิดบั๊กยากต่อการ debug โดยเฉพาะในโค้ดขนาดใหญ่ที่
มีการสืบทอด Class หลายชั้น (ซึ่งเกิดขึ้นเป็นปกติใน Rails ที่ Model เกือบทุกตัวสืบทอดจาก
`ApplicationRecord`)

### ทางออกที่แนะนำ: Class Instance Variable

**Class Instance Variable** คือการใช้ instance variable ธรรมดา (`@var` ตัวเดียว ไม่ใช่
`@@var`) แต่ตั้งค่าไว้ **ในบริบทของตัว Class เอง** (เพราะ Class ก็เป็น Object เหมือนกัน
ตามที่เรียนใน Step 81 — Class จึงมี instance variable ของตัวมันเองได้เช่นกัน!)

```ruby
# frozen_string_literal: true

class Animal
  class << self
    attr_accessor :count
  end

  self.count = 0

  def initialize
    self.class.count += 1
  end
end

class Dog < Animal
  self.count = 0
end

class Cat < Animal
  self.count = 0
end

Dog.new
Dog.new
Cat.new

puts Dog.count    # => 2  (ถูกต้อง — นับเฉพาะของ Dog)
puts Cat.count    # => 1  (ถูกต้อง — นับเฉพาะของ Cat)
```

**อธิบายกลไก:**

- `class << self; attr_accessor :count; end` generate getter/setter `count`/`count=`
  ให้กับ **Class object เอง** (ไม่ใช่ instance ของ Class) — เท่ากับสร้าง `@count` ที่เป็น
  instance variable ของตัว `Animal`, `Dog`, `Cat` แยกกันคนละตัว (เพราะ `Animal`, `Dog`,
  `Cat` ต่างก็เป็น Object คนละตัวกัน ตามหลัก "ทุกอย่างคือ Object")
- `self.class.count += 1` ใน `initialize`: `self.class` คือ Class ของ instance ปัจจุบัน
  (เช่นถ้าเป็น `Dog.new` ก็จะได้ `Dog`) ทำให้ตอนเพิ่มค่า มันไปเพิ่มที่ `@count` ของ Class
  ที่ถูกต้องเสมอ ไม่ปนกัน
- ต้องกำหนด `self.count = 0` แยกในแต่ละ subclass (`Dog`, `Cat`) เพราะ class instance
  variable **ไม่ถูกแชร์ข้าม Class โดยอัตโนมัติ** เหมือนที่ `@@var` เป็นปัญหา — นี่คือ
  จุดต่างสำคัญที่แก้ปัญหาข้างบนได้พอดี

### สรุปเปรียบเทียบ

| | `@@class_var` | Class Instance Variable (`@var` ผ่าน `self`) |
|---|---|---|
| แชร์ระหว่าง instance ของ Class เดียวกัน | ✅ | ✅ (ผ่าน class method) |
| แชร์ข้ามไปยัง subclass โดยไม่ตั้งใจ | ✅ (ปัญหา) | ❌ (แยกกันชัดเจน) |
| คำแนะนำ | หลีกเลี่ยงในโค้ด production | แนะนำให้ใช้แทน |

**บทสรุปของ Step นี้:** ในโค้ดจริงแทบไม่มีเหตุผลต้องใช้ `@@var` เลย ให้ใช้ **class instance
variable ร่วมกับ class method** แทนเสมอ เพื่อหลีกเลี่ยงบั๊กจากการแชร์ค่าข้าม subclass —
นี่เป็นหนึ่งใน best practice ที่ทีมพัฒนา Ruby มืออาชีพยึดถือกันโดยทั่วไป

---

## Step 87: Override `to_s` และ `inspect`

### ปัญหา: default `to_s`/`inspect` อ่านไม่รู้เรื่อง

ตามที่เห็นใน Step 82 เมื่อ `puts` หรือ `p` Object ที่ยังไม่ปรับแต่งอะไร จะได้ string
ที่มีแต่ชื่อ Class กับ memory address ซึ่งไม่มีประโยชน์ต่อการอ่านค่าเลย:

```ruby
class Dog
  def initialize(name, breed)
    @name = name
    @breed = breed
  end
end

rex = Dog.new("Rex", "Labrador")
puts rex   # => #<Dog:0x00007f8b1a0a1234>
p rex      # => #<Dog:0x00007f8b1a0a1234 @name="Rex", @breed="Labrador">
```

(`p` แสดง instance variable ให้ดูด้วย ซึ่งช่วย debug ได้บ้าง แต่ก็ยังไม่ใช่รูปแบบที่
มนุษย์อยากอ่าน)

### `to_s` — ควบคุมว่า `puts`/string interpolation แสดงผลอย่างไร

`to_s` เป็น method ที่มีอยู่แล้วในทุก Object (สืบทอดมาจาก `Object`) มีหน้าที่แปลง Object
เป็น String — เราสามารถ **override** (เขียนทับ) มันเพื่อกำหนดรูปแบบการแสดงผลของเราเองได้:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, breed)
    @name = name
    @breed = breed
  end

  def to_s
    "#{@name} (พันธุ์ #{@breed})"
  end
end

rex = Dog.new("Rex", "Labrador")

puts rex                     # => Rex (พันธุ์ Labrador)   -- puts เรียก to_s ให้อัตโนมัติ
puts "สุนัขของฉันคือ #{rex}"    # => สุนัขของฉันคือ Rex (พันธุ์ Labrador)  -- string interpolation ก็เรียก to_s เช่นกัน
puts "สุนัขของฉันคือ " + rex.to_s  # เรียก to_s ตรงๆ ก็ได้ผลเหมือนกัน
```

**จุดที่ `to_s` ถูกเรียกโดยอัตโนมัติ:** `puts`, string interpolation (`#{}`), และการ
concatenate string ด้วย `+` (ถ้าฝั่งใดฝั่งหนึ่งเป็น String อยู่แล้ว) ล้วนเรียก `to_s`
ให้อัตโนมัติเบื้องหลังทั้งสิ้น

### `inspect` — ควบคุมว่า `p`/`pp`/irb แสดงผลอย่างไร

`inspect` มีจุดประสงค์ต่างจาก `to_s` — มันเน้นการแสดงผลเพื่อ **debug** จึงควรแสดงข้อมูล
โครงสร้างภายในให้ครบถ้วนกว่า `to_s`:

```ruby
# frozen_string_literal: true

class Dog
  def initialize(name, breed)
    @name = name
    @breed = breed
  end

  def to_s
    "#{@name} (พันธุ์ #{@breed})"
  end

  def inspect
    "#<Dog name=#{@name.inspect} breed=#{@breed.inspect}>"
  end
end

rex = Dog.new("Rex", "Labrador")

puts rex   # => Rex (พันธุ์ Labrador)                      -- เรียก to_s
p rex      # => #<Dog name="Rex" breed="Labrador">          -- เรียก inspect
```

**หลักการเลือกใช้:**

- `to_s` — เขียนให้อ่านง่ายสำหรับ **ผู้ใช้ทั่วไป** (human-readable) เหมาะกับ error message,
  การแสดงผลบนหน้าเว็บ, log ที่ต้องอ่านง่าย
- `inspect` — เขียนให้เห็น **โครงสร้างข้อมูลภายใน** ชัดเจนสำหรับตอน debug (คล้าย pattern
  `#<ClassName attr=value attr2=value2>` ที่ใช้กันทั่วไป) มักเรียก `.inspect` ซ้อนกับค่า
  ภายในเพื่อให้เห็น string ที่มี quote ครอบชัดเจน (เช่น `@name.inspect` แทน `@name` ตรงๆ)

> **หมายเหตุ:** ถ้า override เฉพาะ `to_s` แต่ไม่ override `inspect` การเรียก `p` จะยังคง
> ใช้ default (แสดง object id + instance variable ดิบๆ) — ทั้งสอง method เป็นอิสระต่อกัน
> ต้อง override แยกกันถ้าต้องการปรับทั้งคู่

---

## Step 88: ความแตกต่างของ `==`, `eql?`, `equal?` และการ override `==`

Ruby มี method เปรียบเทียบความเท่ากันอยู่ 3 แบบที่ความหมายต่างกันชัดเจน มือใหม่มักสับสน
ทั้งสามตัวนี้จึงต้องแยกให้แน่นตั้งแต่ต้น

### `equal?` — เปรียบเทียบว่าเป็น Object เดียวกันในหน่วยความจำหรือไม่

```ruby
a = "hello"
b = "hello"
c = a

puts a.equal?(b)  # => false (คนละ Object กัน แม้ค่าข้างในจะเหมือนกัน)
puts a.equal?(c)  # => true  (c คือ reference ไปยัง Object เดียวกับ a)
```

`equal?` เทียบ **object identity** (เหมือนเทียบ memory address) — แทบไม่ได้ override
method นี้เอง เพราะความหมายควรตายตัวเสมอว่า "คือของสิ่งเดียวกันในหน่วยความจำจริงๆ หรือไม่"

### `eql?` — เปรียบเทียบทั้งค่าและชนิดข้อมูล (type-strict)

```ruby
puts 1.eql?(1)      # => true  (ค่าเท่ากัน และชนิดเดียวกันคือ Integer)
puts 1.eql?(1.0)    # => false (ค่าเท่ากัน แต่คนละชนิด: Integer vs Float)
puts 1 == 1.0        # => true  (== ไม่สนใจชนิด สนใจแค่ค่าตัวเลขเท่ากันไหม)
```

`eql?` ใช้บ่อยที่สุดเบื้องหลังการทำงานของ `Hash` (ใช้ตัดสินว่า key สองตัวถือว่าเป็น
key เดียวกันหรือไม่) ในโค้ดแอปพลิเคชันทั่วไปแทบไม่ค่อยได้เรียก `eql?` ตรงๆ

### `==` — เปรียบเทียบ "ค่า" ตามความหมายทางธุรกิจ (ใช้บ่อยที่สุด)

```ruby
puts 1 == 1.0          # => true
puts "hello" == "hello" # => true (แม้เป็นคนละ Object กันในหน่วยความจำ)
puts [1, 2] == [1, 2]    # => true (เทียบสมาชิกทีละตัว)
```

`==` คือตัวที่เราใช้เปรียบเทียบค่าในชีวิตประจำวันมากที่สุด และเป็น method ที่ **ควร
override** เมื่อสร้าง Class ของตัวเอง เพื่อกำหนดว่า "อะไรคือความเท่ากันในความหมายทางธุรกิจ
ของเรา"

### ปัญหา: `==` แบบ default เทียบด้วย object identity เหมือน `equal?`

```ruby
class Point
  def initialize(x, y)
    @x = x
    @y = y
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)

puts p1 == p2  # => false (!!) ทั้งที่ x, y เท่ากันทุกประการ
```

เพราะ `==` ที่ยังไม่ override สืบทอดมาจาก `Object#==` ซึ่งทำงานเหมือน `equal?` (เทียบว่า
เป็น Object เดียวกันในหน่วยความจำหรือไม่) — สำหรับ Point สอง Object ที่พิกัดเหมือนกัน
เราน่าจะอยากให้ถือว่า "เท่ากัน" ในทางความหมาย

### Override `==` เพื่อกำหนดความเท่ากันของเราเอง

```ruby
# frozen_string_literal: true

class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def ==(other)
    other.is_a?(Point) && x == other.x && y == other.y
  end

  # แนวปฏิบัติที่ดี: เมื่อ override == ควร alias eql? ให้ทำงานเหมือนกันด้วย
  # (เผื่อ Point ถูกใช้เป็น Hash key หรือสมาชิกใน Set)
  alias eql? ==

  # และควร override hash คู่กันเสมอ เพื่อให้ใช้กับ Hash/Set ได้ถูกต้อง
  def hash
    [x, y].hash
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)
p3 = Point.new(9, 9)

puts p1 == p2  # => true  (ตอนนี้เทียบค่า x, y แทน object identity)
puts p1 == p3  # => false
puts p1.equal?(p2)  # => false (คนละ Object กันในหน่วยความจำ ยังคงเป็น false ตามเดิม เพราะไม่ได้แตะ equal?)
```

**อธิบาย:**

- override `==` ให้เช็คว่า `other` เป็น class ที่ถูกต้อง (`is_a?(Point)`) ก่อนเสมอ เพื่อ
  ป้องกัน error กรณีถูกเทียบกับชนิดข้อมูลอื่นที่ไม่มี method `.x`/`.y`
- `alias eql? ==` ทำให้ `eql?` เรียกใช้ logic เดียวกับ `==` ที่เรา override — จำเป็นถ้า
  ต้องการให้ `Point` ทำงานถูกต้องเวลาใช้เป็น key ของ `Hash`
- override `hash` คู่กันเสมอเมื่อ override `eql?`/`==` เพราะ `Hash` และ `Set` ใช้ค่า
  `hash` ในการจัดกลุ่มข้อมูลภายใน ถ้า Object สองตัวถือว่า `eql?` กัน แต่ `hash` ต่างกัน
  จะทำให้ `Hash`/`Set` ทำงานผิดพลาด (นี่เป็นกฎที่ Ruby documentation ระบุไว้ชัดเจน)

```ruby
points = { Point.new(1, 2) => "จุดแรก" }
puts points[Point.new(1, 2)]  # => "จุดแรก"  (หา key เจอ เพราะ eql? และ hash ถูก override ให้สอดคล้องกัน)
```

**สรุปสั้นๆ:** ในโค้ดแอปพลิเคชันทั่วไป ให้จำแค่ `==` (สำหรับเทียบค่า ใช้บ่อยสุด) และ
`equal?` (สำหรับเทียบ object identity จริงๆ ใช้น้อย) เป็นหลัก ส่วน `eql?`/`hash` ให้
override คู่กับ `==` เฉพาะเมื่อวางแผนจะใช้ Object นั้นเป็น Hash key หรือสมาชิกใน Set

---

## Step 89: Constant ภายใน Class

### การประกาศ Constant

**Constant** ใน Ruby คือตัวแปรที่ตั้งชื่อขึ้นต้นด้วยตัวพิมพ์ใหญ่ (**convention:** นิยม
เขียนด้วยตัวพิมพ์ใหญ่ทั้งหมดคั่นด้วย `_` เช่น `MAX_SPEED`) มีเจตนาว่าค่าจะไม่เปลี่ยนแปลง
ตลอดอายุโปรแกรม

```ruby
# frozen_string_literal: true

class Dog
  SPECIES = "Canis familiaris"
  MAX_AGE_YEARS = 20
  VALID_SIZES = %w[small medium large].freeze

  attr_reader :name, :size

  def initialize(name, size)
    raise ArgumentError, "size ต้องเป็นหนึ่งใน #{VALID_SIZES}" unless VALID_SIZES.include?(size)

    @name = name
    @size = size
  end
end

puts Dog::SPECIES          # => Canis familiaris  (เข้าถึงจากภายนอกด้วย ::)
puts Dog::MAX_AGE_YEARS    # => 20

rex = Dog.new("Rex", "large")
puts rex.size               # => large

Dog.new("Milo", "huge")
# => ArgumentError (size ต้องเป็นหนึ่งใน ["small", "medium", "large"])
```

**อธิบาย:**

- Constant ที่ประกาศในระดับบนสุดของ class body เข้าถึงจาก **ภายใน** Class ได้ตรงๆ
  (เช่น `VALID_SIZES` ใน `initialize`) และเข้าถึงจาก **ภายนอก** ได้ผ่าน **scope
  resolution operator** `::` เช่น `Dog::SPECIES`
- `.freeze` ที่ต่อท้าย Array/Hash/String literal ทำให้ Object นั้น **immutable**
  (แก้ไขเนื้อหาไม่ได้อีก) เป็นแนวปฏิบัติที่ดีสำหรับ constant ที่เป็น Array/Hash เพราะ
  Ruby **ไม่ได้บังคับ** ว่า constant ห้ามถูกแก้ไขเนื้อหาข้างใน (ต่างจากค่าคงที่ในภาษาอื่น)

### Ruby ไม่ได้บังคับว่า Constant ห้ามเปลี่ยนค่าจริงๆ (แค่เตือน)

```ruby
class Config
  TIMEOUT = 30
end

Config::TIMEOUT = 60
# => (irb): warning: already initialized constant Config::TIMEOUT
#    ทำงานสำเร็จ ค่าถูกเปลี่ยนจริง แค่มี warning เตือนขึ้นมา ไม่ error

puts Config::TIMEOUT  # => 60
```

Ruby ให้แค่ **คำเตือน (warning)** ไม่ได้ปิดกั้นการเปลี่ยนค่า constant จริงๆ — ต่างจาก
ภาษาอื่นที่ compiler จะ error ทันที ด้วยเหตุนี้การไม่แก้ไขค่า constant จึงเป็นเรื่องของ
**วินัยของนักพัฒนา** ไม่ใช่กฎที่ภาษาบังคับ (แต่ก็ไม่ควรทำ เพราะทำให้โค้ดเข้าใจยากขึ้นมาก)

และแม้ตัว constant เอง (reference) จะ "ห้ามเปลี่ยน" แต่ถ้า constant นั้นชี้ไปยัง Array/
Hash ที่ mutable เนื้อหาข้างในยังแก้ไขได้ตามปกติถ้าไม่ `.freeze` ไว้:

```ruby
class Dog
  VALID_SIZES = %w[small medium large]  # ไม่ได้ .freeze
end

Dog::VALID_SIZES << "huge"  # แก้ไขเนื้อหาข้างในได้! (ทั้งที่ VALID_SIZES ไม่เปลี่ยน reference)
puts Dog::VALID_SIZES  # => ["small", "medium", "large", "huge"]
```

นี่คือเหตุผลที่ควรใส่ `.freeze` กับ constant ที่เป็น Array/Hash/String เสมอ เพื่อป้องกัน
การแก้ไขเนื้อหาโดยไม่ตั้งใจ

### ใช้เมื่อไหร่

Constant เหมาะกับค่าที่:

1. ไม่เปลี่ยนแปลงตามธรรมชาติของโดเมน เช่น `MAX_LOGIN_ATTEMPTS = 3`,
   `SUPPORTED_CURRENCIES = %w[THB USD EUR].freeze`
2. ต้องอ้างอิงซ้ำหลายที่ในโค้ด — เขียนเป็น constant ครั้งเดียวดีกว่า hardcode ค่าเดิมซ้ำๆ
   (ตรงกับหลัก DRY ที่เรียนใน Part 001)
3. ต้องการให้ผู้อ่านโค้ดรู้ทันทีว่าเป็นค่าคงที่ระดับ configuration ของ Class นั้น

ใน Rails เราจะเห็น pattern นี้บ่อยมาก เช่น `STATUSES = %w[pending active closed].freeze`
ในตัว Model เพื่อกำหนดค่าที่เป็นไปได้ของ field หนึ่งๆ

---

## Step 90: แบบฝึกหัดโปรเจกต์ — สร้าง `BankAccount` class ครบระบบ

### โจทย์

เขียน Class ชื่อ `BankAccount` ที่รวมทุกแนวคิดจาก Step 81–89 เข้าด้วยกัน โดยมีความสามารถ
ดังนี้:

1. สร้างบัญชีด้วยชื่อเจ้าของบัญชี (`owner_name`) และยอดเงินเริ่มต้น (`initial_balance`,
   default = 0) — ถ้ายอดเงินเริ่มต้นติดลบ ต้อง raise error
2. มี method `deposit(amount)` สำหรับฝากเงิน — ถ้า `amount` ไม่เป็นบวก ต้อง raise error
3. มี method `withdraw(amount)` สำหรับถอนเงิน — ถ้า `amount` ไม่เป็นบวก หรือมากกว่ายอดเงิน
   คงเหลือ ต้อง raise error (ห้ามยอดติดลบ)
4. มี `account_number` ที่สร้างอัตโนมัติแบบไม่ซ้ำกันทุกบัญชี (อ่านได้อย่างเดียวจากภายนอก)
5. มี class method `BankAccount.total_accounts` นับจำนวนบัญชีทั้งหมดที่เคยสร้าง (ต้องใช้
   class instance variable ไม่ใช่ `@@var` ตามที่เรียนใน Step 86)
6. Override `to_s` ให้แสดงผลอ่านง่าย และ override `==` ให้เทียบจาก `account_number`
7. เก็บ log การทำธุรกรรมทุกครั้งไว้ใน instance variable แบบ Array (อ่านได้อย่างเดียว)

### เฉลย

```ruby
# frozen_string_literal: true

# bank_account.rb
class BankAccount
  # --- Constants ---
  MIN_INITIAL_BALANCE = 0

  # --- Class Instance Variable pattern (ไม่ใช้ @@var ตามที่เรียนใน Step 86) ---
  class << self
    attr_accessor :accounts_created
  end
  self.accounts_created = 0

  def self.total_accounts
    accounts_created
  end

  # --- Attributes ---
  attr_reader :owner_name, :account_number, :transactions

  def initialize(owner_name, initial_balance: 0)
    raise ArgumentError, "ยอดเงินเริ่มต้นต้องไม่ติดลบ" if initial_balance < MIN_INITIAL_BALANCE

    self.class.accounts_created += 1

    @owner_name = owner_name
    @balance = initial_balance
    @account_number = generate_account_number
    @transactions = []

    log_transaction("เปิดบัญชี", initial_balance) if initial_balance.positive?
  end

  # getter ที่คำนวณ/ป้องกันการแก้ไขจากภายนอกโดยตรง (เขียนมือ ไม่ใช้ attr_accessor เพราะ
  # ไม่อยากให้ใครมาตั้งค่า balance ตรงๆ ได้ ต้องผ่าน deposit/withdraw เท่านั้น)
  def balance
    @balance
  end

  def deposit(amount)
    raise ArgumentError, "จำนวนเงินฝากต้องมากกว่า 0" unless amount.positive?

    @balance += amount
    log_transaction("ฝากเงิน", amount)
    balance
  end

  def withdraw(amount)
    raise ArgumentError, "จำนวนเงินถอนต้องมากกว่า 0" unless amount.positive?
    raise ArgumentError, "ยอดเงินคงเหลือไม่พอ (คงเหลือ #{balance}, ขอถอน #{amount})" if amount > balance

    @balance -= amount
    log_transaction("ถอนเงิน", -amount)
    balance
  end

  def to_s
    format("บัญชี %s (%s) ยอดคงเหลือ %.2f บาท", account_number, owner_name, balance)
  end

  def inspect
    "#<BankAccount account_number=#{account_number.inspect} owner=#{owner_name.inspect} balance=#{balance}>"
  end

  def ==(other)
    other.is_a?(BankAccount) && account_number == other.account_number
  end
  alias eql? ==

  def hash
    account_number.hash
  end

  private

  # method ส่วนตัว (private) ใช้ได้เฉพาะภายใน Class เอง — เรียนละเอียดเรื่อง private/public ใน Part ถัดไป
  # ในที่นี้ใช้เพื่อไม่ให้ generate account_number จากภายนอกได้โดยตรง
  def generate_account_number
    format("ACC-%06d", self.class.accounts_created)
  end

  def log_transaction(type, amount)
    @transactions << { type: type, amount: amount, balance_after: balance, at: Time.now }
  end
end
```

### ทดสอบใช้งาน

```ruby
# frozen_string_literal: true

# main.rb
require_relative "bank_account"

account1 = BankAccount.new("สมชาย ใจดี", initial_balance: 1000)
account2 = BankAccount.new("สมหญิง ยิ้มแย้ม")

puts account1
# => บัญชี ACC-000001 (สมชาย ใจดี) ยอดคงเหลือ 1000.00 บาท

account1.deposit(500)
puts account1.balance  # => 1500

account1.withdraw(200)
puts account1.balance  # => 1300

begin
  account1.withdraw(999_999)
rescue ArgumentError => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
  # => เกิดข้อผิดพลาด: ยอดเงินคงเหลือไม่พอ (คงเหลือ 1300, ขอถอน 999999)
end

begin
  BankAccount.new("คนไม่มีเงิน", initial_balance: -100)
rescue ArgumentError => e
  puts "เกิดข้อผิดพลาด: #{e.message}"
  # => เกิดข้อผิดพลาด: ยอดเงินเริ่มต้นต้องไม่ติดลบ
end

puts BankAccount.total_accounts  # => 2

# ทดสอบ == ที่ override ไว้
account1_copy_ref = account1
puts account1 == account1_copy_ref  # => true (คนละชื่อตัวแปร แต่ reference เดียวกัน)
puts account1 == account2            # => false (account_number ต่างกัน)

# ดู transaction log
account1.transactions.each do |txn|
  puts "#{txn[:type]}: #{txn[:amount]} (คงเหลือหลังทำรายการ: #{txn[:balance_after]})"
end
# => เปิดบัญชี: 1000 (คงเหลือหลังทำรายการ: 1000)
# => ฝากเงิน: 500 (คงเหลือหลังทำรายการ: 1500)
# => ถอนเงิน: -200 (คงเหลือหลังทำรายการ: 1300)

p account1
# => #<BankAccount account_number="ACC-000001" owner="สมชาย ใจดี" balance=1300>
```

**แนวคิดที่ประกอบกันในเฉลยนี้:**

- `initialize` ตรวจสอบความถูกต้องของข้อมูล (validation) ทันทีที่สร้าง Object เพื่อไม่ให้
  Object อยู่ในสถานะที่ผิดพลาดได้เลยตั้งแต่ต้น (ตามที่ย้ำไว้ใน Step 82)
- `balance` ใช้ getter method เขียนมือ (ไม่ใช้ `attr_accessor`) เพราะต้องการบังคับให้แก้ไข
  ยอดเงินผ่าน `deposit`/`withdraw` เท่านั้น ไม่ให้ใครตั้งค่าตรงๆ ได้ — encapsulation
  ที่แท้จริงตามที่อธิบายใน Step 84
- `class << self; attr_accessor :accounts_created; end` คือ class instance variable
  pattern จาก Step 86 ที่ปลอดภัยกว่า `@@var` เพราะถ้าวันหนึ่งมีการสืบทอด `BankAccount`
  (เช่น `SavingsAccount < BankAccount`) การนับจำนวนจะไม่ปนกันข้าม subclass
- `to_s`/`inspect` override ตามหลักการ Step 87 — `to_s` เน้นอ่านง่ายสำหรับผู้ใช้ทั่วไป
  `inspect` เน้นแสดงโครงสร้างสำหรับ debug
- `==`/`eql?`/`hash` override ตามหลักการ Step 88 — สองบัญชีถือว่า "เท่ากัน" ถ้ามี
  `account_number` เดียวกันเท่านั้น ไม่ใช่เทียบทุก field
- `MIN_INITIAL_BALANCE` เป็น constant ตามหลักการ Step 89 ใช้แทน magic number `0` ที่
  hardcode ตรงๆ ทำให้อ่านโค้ดแล้วเข้าใจเจตนาได้ทันที

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม class `SavingsAccount` ที่ยังไม่ต้องสืบทอดจาก `BankAccount` (จะเรียนเรื่อง
   inheritance ใน Part 010) แต่ให้สร้างเป็น Class แยกต่างหากที่มี method
   `add_interest(rate)` คำนวณดอกเบี้ยจากยอดคงเหลือปัจจุบันแล้วฝากเข้าบัญชีอัตโนมัติ
   (เช่น `rate = 0.015` หมายถึงดอกเบี้ย 1.5% ต่อปี) พร้อม class method
   `SavingsAccount.total_accounts` ที่นับแยกจาก `BankAccount.total_accounts` โดยเด็ดขาด
   (ทดสอบว่าไม่ปนกัน)
2. เพิ่ม method `transfer(to_account, amount)` ให้กับ `BankAccount` สำหรับโอนเงินระหว่าง
   สองบัญชี — ต้องตรวจสอบว่ายอดเงินพอก่อนโอน และต้อง log transaction ทั้งฝั่งผู้โอนกับ
   ผู้รับให้ถูกต้อง (ใบ้: เรียก `withdraw`/`deposit` ของอีกฝั่งจากภายใน method นี้)
3. เขียน method `self.richest(accounts)` เป็น class method ที่รับ Array ของ `BankAccount`
   แล้ว return บัญชีที่มียอดเงินคงเหลือสูงที่สุด (ใบ้: ใช้ `max_by` ที่เรียนใน Part 004)
   และเขียนทดสอบด้วยการสร้างบัญชีหลายใบแล้วเรียก `BankAccount.richest([...])`

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจปรัชญา OOP และหลักการว่า "ทุกอย่างใน Ruby คือ Object" ไม่มีข้อยกเว้น
- นิยาม Class, ใช้ `initialize` เป็น constructor ที่ Ruby เรียกให้อัตโนมัติผ่าน `.new`
- เข้าใจ scope ของ instance variable (`@var`) และรู้ว่าตัวที่ยังไม่กำหนดค่าจะเป็น `nil`
  เงียบๆ โดยไม่ error ซึ่งเป็นบ่อเกิดบั๊กจาก typo ที่พบบ่อย
- เขียน getter/setter method ด้วยมือได้ และรู้ว่า `attr_reader`/`attr_writer`/
  `attr_accessor` เป็นเพียงตัวช่วย generate method แบบเดียวกันโดยอัตโนมัติ ไม่ใช่ syntax
  พิเศษของภาษา
- แยกความแตกต่างของ instance method กับ class method (`def self.method_name` และ
  `class << self`) และรู้ว่าเมื่อไหร่ควรใช้แบบไหน
- เข้าใจปัญหาของ Class Variable (`@@var`) ที่แชร์ข้าม subclass โดยไม่ตั้งใจ และรู้วิธี
  แก้ด้วย Class Instance Variable ที่ปลอดภัยกว่า
- Override `to_s` (สำหรับผู้ใช้อ่าน) และ `inspect` (สำหรับ debug) ให้ Object ของตัวเอง
- แยกความแตกต่างของ `==` (เทียบค่า), `eql?` (เทียบค่า+ชนิด, ใช้กับ Hash), และ `equal?`
  (เทียบ object identity) พร้อม override `==`/`eql?`/`hash` ให้ทำงานสอดคล้องกัน
- ประกาศและใช้ constant ภายใน Class อย่างถูกวิธี รวมถึงรู้ข้อจำกัดว่า Ruby ไม่ได้บังคับ
  ไม่ให้แก้ไขค่าคงที่จริงๆ (แค่เตือน)
- ประกอบทุกแนวคิดเข้าด้วยกันในโปรเจกต์ `BankAccount` ที่ใช้งานได้จริง ครบทั้ง validation,
  encapsulation, class-level tracking, และการ override method เปรียบเทียบ/แสดงผล

**ต่อไป (Part 010):** เราจะเรียนเรื่อง **Inheritance** (การสืบทอด Class ด้วย `<`),
**Module** และ **Mixin** (`include`/`extend`) เพื่อแชร์พฤติกรรมระหว่าง Class โดยไม่ต้อง
เขียนโค้ดซ้ำ และปิดท้ายเฟส 1 ทั้งหมดด้วยโปรเจกต์รวบยอดขนาดเล็ก: **Library Management CLI**
ระบบจัดการห้องสมุดที่รวมทุกอย่างตั้งแต่ Part 001–010 เข้าไว้ด้วยกันเป็นโปรแกรมที่ใช้งานได้จริง
