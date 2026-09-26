# Part 058: GraphQL เบื้องต้นด้วย graphql-ruby — Schema, Type, Query

> **Step ครอบคลุมใน Part นี้:** Step 571–580
> **ระดับ:** กลาง-สูง (ควรผ่าน Part 056 เรื่อง Rails API-only mode/serializer และ Part 057 เรื่อง
> API versioning/pagination มาก่อน — Part นี้สมมติว่าคุ้นเคยกับการสร้าง JSON API ด้วย Rails มาแล้ว
> และจะเปรียบเทียบกับแนวทางนั้นตลอดทั้ง Part รวมถึงต้องเข้าใจเรื่อง N+1 query จาก Part 034 มาก่อน
> ด้วย เพราะ Part นี้จะพา N+1 ไปอีกระดับหนึ่งที่อันตรายกว่าเดิม)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x / Rails 8.1.x / gem `graphql` (graphql-ruby) 2.6.x — ทุกคำสั่ง
> terminal, ทุก query, ทุก SQL log และทุก response JSON ในเอกสารนี้ capture มาจากการรันจริงบน
> Ruby 3.3.6 + Rails 8.1.4 + graphql-ruby 2.6.11 + ฐานข้อมูล SQLite3 ไม่มีตัวอย่างไหนเป็นการเดา
> พฤติกรรมของ library

จนถึงตอนนี้ API ที่เราสร้างมาทั้งหมด (Part 056–057) เป็นแบบ **REST**: แต่ละ resource มี URL
ของตัวเอง (`/posts`, `/posts/1`, `/posts/1/comments`) และ server เป็นคนกำหนดตายตัวว่า response
ของแต่ละ endpoint จะมีหน้าตาอย่างไร Part นี้จะแนะนำสถาปัตยกรรม API อีกแบบที่ได้รับความนิยมมาก
ในระบบที่มี client หลากหลาย (web, mobile, third-party) — **GraphQL** — ซึ่งพลิกกลับแนวคิดนี้
โดยสิ้นเชิง: มี endpoint เดียว และ **client เป็นคนกำหนดเองว่าต้องการ field อะไรบ้าง**

เราจะสร้างโปรเจกต์ทดลองใหม่ชื่อ `graphql_demo` ที่มีโดเมนคุ้นเคย (`Post` มี `has_many :comments`
เหมือนที่ใช้มาตั้งแต่ Part 030) แต่คราวนี้เปิดผ่าน **GraphQL endpoint เดียว** แทนที่จะเป็น REST
หลาย endpoint เพื่อให้เห็นความต่างชัดเจนที่สุด และทุกตัวอย่างจะยิง query จริงผ่าน `curl` ไปที่
`/graphql` พร้อม capture SQL log จริงเพื่อพิสูจน์ปัญหา N+1 ที่ Part นี้จะเน้นเป็นพิเศษ

## สารบัญของ Part นี้

- Step 571: GraphQL คืออะไร และปรัชญาที่ต่างจาก REST โดยสิ้นเชิง
- Step 572: ติดตั้ง `graphql-ruby` (`rails generate graphql:install`) และสำรวจไฟล์ที่ generate มา
  ทั้งหมด
- Step 573: Object Type คือกลไกอะไร — นิยาม `Types::PostType`
- Step 574: Query root type — เพิ่ม field เข้า entry point ของ schema
- Step 575: รัน query จริงผ่าน GraphiQL และ `curl POST /graphql`
- Step 576: Nested field — ดึง `Post` พร้อม `Comments` ในคำขอเดียว (หัวใจของ GraphQL)
- Step 577: ปัญหา N+1 ใน GraphQL — อันตรายกว่าที่คิด
- Step 578: แก้ N+1 ด้วย `GraphQL::Dataloader`
- Step 579: Scalar Types vs Custom Types
- Step 580: Connection/Pagination แบบ Relay-style cursor pagination

---

## Step 571: GraphQL คืออะไร และปรัชญาที่ต่างจาก REST โดยสิ้นเชิง

### GraphQL คืออะไร

**GraphQL** คือ query language สำหรับ API พร้อม runtime สำหรับ execute query เหล่านั้น สร้างขึ้นที่
Facebook ปี 2012 เพื่อแก้ปัญหาที่แอปมือถือของ Facebook เจอจริง (ต้องยิงหลาย endpoint REST เพื่อ
render หน้าจอเดียว บาง field ที่ได้มาก็ไม่ได้ใช้) แล้วเปิดเป็น open specification ในปี 2015
ปัจจุบันมี implementation ในแทบทุกภาษา — สำหรับ Ruby/Rails คือ gem **`graphql-ruby`** ซึ่งเป็น
implementation หลักที่ GitHub, Shopify (ผู้ดูแลหลักของ gem นี้) ใช้งานจริงใน production

หลักการสำคัญ 3 ข้อที่ทำให้ GraphQL ต่างจาก REST โดยสิ้นเชิง:

**1. Single endpoint** — REST มี URL ตาม resource (`GET /posts`, `GET /posts/1`,
`GET /posts/1/comments`) แต่ GraphQL มี **endpoint เดียว** (ปกติคือ `POST /graphql`) รับคำขอทุก
รูปแบบผ่าน body เดียวกัน ไม่ว่าจะขอ post, comment, หรือทั้งสองอย่างพร้อมกัน

**2. Client กำหนดเอง ไม่ใช่ server** — นี่คือหัวใจที่สุด ใน REST server เป็นคนตัดสินใจว่า
`GET /posts/1` จะคืน field อะไรบ้าง (ทั้งหมดของ serializer) client ไม่มีสิทธิ์เลือก แต่ใน GraphQL
client ส่ง **query** ที่ระบุ field ที่ต้องการทุกครั้ง เช่น ถ้าหน้าจอต้องการแค่ชื่อโพสต์กับจำนวน
ความเห็น ก็ขอแค่นั้นได้เลย ไม่ต้องรับ `body` ยาวๆ ที่ไม่ได้ใช้มาด้วย

**3. Strongly-typed schema เป็นสัญญา (contract)** — ทุก GraphQL API ต้องประกาศ **Schema** ที่
ระบุชัดเจนว่ามี Type อะไรบ้าง แต่ละ Type มี field อะไร, field นั้น type อะไร, nullable หรือไม่
Schema นี้ตรวจสอบได้เอง (introspection) ทำให้เครื่องมืออย่าง GraphiQL หรือ Apollo Studio สร้าง
เอกสาร API และ autocomplete ให้อัตโนมัติแบบ real-time โดยไม่ต้องเขียนเอกสารแยกต่างหาก (ต่างจาก
REST ที่ต้องพึ่ง OpenAPI/Swagger spec แยกไฟล์ ซึ่งจะเจาะลึกใน Part 060)

### ปัญหาที่ GraphQL แก้ได้ตรงจุด: Over-fetching และ Under-fetching

**Over-fetching** คือการได้ข้อมูลเกินความจำเป็น เช่น หน้าจอ mobile ต้องการแค่ชื่อโพสต์ แต่
`GET /posts` ของ REST คืนทั้ง `body`, `created_at`, `author`, ฯลฯ มาด้วยเสมอ เพราะ serializer
ตัวเดียวต้องใช้ได้กับทุกหน้าจอ

**Under-fetching** คือการได้ข้อมูลไม่พอในคำขอเดียว เช่น ต้องการโพสต์พร้อมความเห็นทั้งหมด แต่ REST
ต้องยิง 2 คำขอแยกกัน (`GET /posts/1` แล้วค่อย `GET /posts/1/comments`) หรือไม่ก็ต้องสร้าง endpoint
พิเศษ (`GET /posts/1?include=comments`) ที่ทีม backend ต้องออกแบบไว้ล่วงหน้าทีละกรณี

GraphQL แก้ทั้งสองปัญหาพร้อมกันด้วยกลไกเดียว: client ขอ field ที่ต้องการแบบ nested ได้ในคำขอ
เดียว ไม่ว่าจะลึกแค่ไหน โดยไม่ต้องรอ backend สร้าง endpoint ใหม่ให้ — นี่คือสิ่งที่ Step 576 จะ
สาธิตให้เห็นจริงด้วยการดึง `Post` พร้อม `Comments` ในคำขอเดียว

### ตารางเปรียบเทียบแบบตรงไปตรงมา: REST vs GraphQL

GraphQL **ไม่ได้ดีกว่า REST เสมอไป** มันแลก simplicity ของ REST กับความยืดหยุ่นที่ต้องจ่ายด้วย
ความซับซ้อนจริงฝั่ง server ตารางนี้สรุปข้อดี-ข้อเสียของทั้งสองแบบอย่างตรงไปตรงมา ไม่เชียร์ฝั่งใด
ฝั่งหนึ่ง:

| มิติ | REST | GraphQL |
|------|------|---------|
| Endpoint | หลาย URL ตาม resource (`/posts`, `/posts/1/comments`) | endpoint เดียว (`/graphql`) รับ `POST` เกือบทุกครั้ง |
| ใครกำหนดรูปร่าง response | Server (serializer ตายตัวต่อ endpoint) | Client (เขียน query เองทุกครั้ง) |
| Over-fetching/Under-fetching | เกิดได้บ่อย ต้องแก้ด้วย query param เฉพาะกิจ หรือสร้าง endpoint ใหม่ | แก้ปัญหานี้โดยตรงตั้งแต่การออกแบบ |
| HTTP caching (CDN, browser, `Cache-Control` ตาม URL) | ทำได้ตรงไปตรงมา เพราะ `GET` + URL เป็น cache key อยู่แล้ว | ยากกว่ามาก เพราะทุก query ยิงไป URL เดียวกันด้วย `POST` — ต้อง cache ที่ระดับ query/response เอง (เช่น persisted queries, client-side cache อย่าง Apollo/Relay) |
| N+1 query | เกิดได้เหมือนกัน แต่ควบคุมได้จากโค้ด server เอง (รู้ล่วงหน้าว่า endpoint ไหน include อะไร) | **เกิดง่ายกว่ามาก** เพราะ client เป็นคนกำหนดความลึกของ nested field เอง — เพิ่ม field เดียวใน query จาก client ก็สร้างจุด N+1 ใหม่ในโค้ด server ได้ทันทีโดยไม่มีใครแก้โค้ด server เลย (Step 577) |
| Rate limiting / ควบคุม cost | ตรงไปตรงมา จำกัดต่อ route/endpoint ได้เลย | ซับซ้อนกว่า เพราะ query เดียวอาจ "แพง" มากหรือน้อยขึ้นกับความลึกที่ client เลือก ต้องคำนวณ query complexity/depth เอง (ปูพื้นใน Step 578–580 เจาะลึกเรื่อง cost ใน Part 060) |
| เส้นโค้งการเรียนรู้ทีม | ต่ำ HTTP verb + resource ตรงไปตรงมา | สูงกว่า ต้องเข้าใจ SDL, resolver, N+1 เฉพาะทาง, Dataloader |
| เหมาะกับ | CRUD ตรงไปตรงมา, public API ที่อยากได้ HTTP cache แรงๆ, ทีมเล็กที่อยากเรียบง่าย | Client หลายแบบที่ต้องการ data shape ต่างกัน (web/mobile/partner), หน้าจอที่ต้องรวมข้อมูลจากหลาย resource พร้อมกัน, ทีมที่ frontend/backend แยกกันชัดเจนและอยากได้ contract ที่ type-safe |

**สรุปแบบตรงไปตรงมา:** ถ้า API ของคุณเป็น CRUD ธรรมดา มี client แบบเดียว และอยากได้ประโยชน์จาก
HTTP caching เต็มที่ — REST ยังคงเป็นตัวเลือกที่ดีกว่าและง่ายกว่าในการดูแลระยะยาว แต่ถ้า client
หลายแบบต้องการข้อมูลรูปร่างต่างกันจาก resource ชุดเดียวกัน หรือหน้าจอต้องรวมข้อมูลจากหลาย
resource ที่สัมพันธ์กันเป็นชั้นๆ — GraphQL จะประหยัดจำนวน round-trip และลดภาระการดูแล endpoint
เฉพาะกิจได้มาก แลกกับที่ต้องระวังเรื่อง N+1 และ caching เป็นพิเศษ ระบบจริงจำนวนมาก (รวมถึง
GitHub) ใช้ **ทั้งสองแบบคู่กัน**: REST/JSON:API สำหรับ integration ภายนอกที่ต้องการ caching
มาตรฐาน และ GraphQL สำหรับ client ภายในที่ต้องการความยืดหยุ่นสูง

> **ข้อควรรู้:** GraphQL ไม่ผูกกับ HTTP method ใดเป็นพิเศษ (spec ไม่ได้บังคับว่าต้องเป็น `POST`)
> แต่ implementation แทบทั้งหมดรวมถึง `graphql-ruby` เลือกใช้ `POST` เป็นค่าเริ่มต้น เพราะ query
> string อาจยาวเกิน limit ของ URL ใน `GET` และ `POST` body ไม่ถูก cache โดย proxy/CDN ทั่วไป
> (ซึ่งเป็นสาเหตุหลักที่ caching ของ GraphQL ยากกว่า REST ตามตารางด้านบน)

---

## เตรียม Rails App สำหรับทดลอง GraphQL ใน Part นี้

สร้างโปรเจกต์ใหม่และโมเดล `Post`/`Comment` แบบเดียวกับที่คุ้นเคย:

```bash
rails new graphql_demo --minimal
cd graphql_demo

bin/rails generate model Post title:string body:text
bin/rails generate model Comment body:text commenter:string post:references

bin/rails db:migrate
```

ผูก association:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy
end
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
end
```

Seed ข้อมูลทดลองไว้เล็กน้อย (ตั้งใจให้จำนวน comment ต่อโพสต์ไม่เท่ากัน รวมถึงมีโพสต์ที่ไม่มี
comment เลย — เพื่อให้เห็นปัญหา N+1 ชัดเจนใน Step 577 และเห็น edge case ของการ batch load ใน
Step 578):

```ruby
# db/seeds.rb
p1 = Post.create!(title: "GraphQL คืออะไร", body: "เนื้อหาโพสต์แรก")
p1.comments.create!(commenter: "มานี", body: "บทความดีมาก")
p1.comments.create!(commenter: "ปิติ", body: "อยากรู้เรื่อง N+1 ต่อ")

p2 = Post.create!(title: "REST vs GraphQL", body: "เนื้อหาโพสต์สอง")
p2.comments.create!(commenter: "ชูใจ", body: "เห็นด้วยว่า REST ก็ยังมีที่ใช้")

Post.create!(title: "โพสต์ที่ยังไม่มีคอมเมนต์", body: "ทดสอบ edge case")
```

```bash
bin/rails db:seed
```

---

## Step 572: ติดตั้ง `graphql-ruby` และสำรวจไฟล์ที่ generate มาทั้งหมด

### ติดตั้ง gem

```bash
bundle add graphql
```

Gem `graphql` คือ core implementation ทั้งหมดของ GraphQL runtime ฝั่ง Ruby (parser, type system,
execution engine) — ไม่ผูกกับ Rails โดยตรง แต่มี generator ที่ผูกเข้ากับโครงสร้าง Rails ให้
สะดวก

### รัน install generator

```bash
bin/rails generate graphql:install
```

ผลลัพธ์จริงจากการรัน (บน graphql-ruby 2.6.11):

```
      create  app/graphql/types
      create  app/graphql/types/.keep
      create  app/graphql/graphql_demo_schema.rb
      create  app/graphql/types/base_object.rb
      create  app/graphql/types/base_argument.rb
      create  app/graphql/types/base_field.rb
      create  app/graphql/types/base_enum.rb
      create  app/graphql/types/base_input_object.rb
      create  app/graphql/types/base_interface.rb
      create  app/graphql/types/base_scalar.rb
      create  app/graphql/types/base_union.rb
      create  app/graphql/resolvers/base_resolver.rb
      create  app/graphql/types/query_type.rb
add_root_type  query
      create  app/graphql/mutations
      create  app/graphql/mutations/.keep
      create  app/graphql/mutations/base_mutation.rb
      create  app/graphql/types/mutation_type.rb
add_root_type  mutation
      create  app/controllers/graphql_controller.rb
       route  post "/graphql", to: "graphql#execute"
     gemfile  graphiql-rails
       route  graphiql-rails
      create  app/graphql/types/node_type.rb
      insert  app/graphql/types/query_type.rb
      create  app/graphql/types/base_connection.rb
      create  app/graphql/types/base_edge.rb
      insert  app/graphql/types/base_object.rb
      insert  app/graphql/types/base_object.rb
      insert  app/graphql/types/base_union.rb
      insert  app/graphql/types/base_union.rb
      insert  app/graphql/types/base_interface.rb
      insert  app/graphql/types/base_interface.rb
      insert  app/graphql/graphql_demo_schema.rb
      insert  config/application.rb
Gemfile has been modified, make sure you `bundle install`
```

```bash
bundle install
```

Generator ตัวนี้สร้างไฟล์เยอะมากในครั้งเดียว มาไล่ทีละกลุ่มว่าแต่ละไฟล์ทำหน้าที่อะไร

### กลุ่ม 1: `app/graphql/types/base_*.rb` — คลาสฐานที่ generate ให้ปรับแต่งพฤติกรรมรวม

Generator สร้างคลาสฐาน 8 ตัวไว้ล่วงหน้า (`BaseObject`, `BaseArgument`, `BaseField`, `BaseEnum`,
`BaseInputObject`, `BaseInterface`, `BaseScalar`, `BaseUnion`) แนวคิดเหมือน `ApplicationRecord`
หรือ `ApplicationController` — ทุก Type ที่เราเขียนเองจะสืบทอดจากคลาสฐานเหล่านี้ ไม่ใช่จากคลาส
ของ gem โดยตรง เพื่อให้มีจุดเดียวสำหรับปรับพฤติกรรมรวม (เช่น authorization ทุก field พร้อมกัน —
จะเจาะลึกใน Part 059)

```ruby
# app/graphql/types/base_object.rb
module Types
  class BaseObject < GraphQL::Schema::Object
    edge_type_class(Types::BaseEdge)
    connection_type_class(Types::BaseConnection)
    field_class Types::BaseField
  end
end
```

```ruby
# app/graphql/types/base_field.rb
module Types
  class BaseField < GraphQL::Schema::Field
    argument_class Types::BaseArgument
  end
end
```

`BaseObject` ผูก `field_class` ให้ทุก field ที่ประกาศผ่านคลาสนี้ใช้ `BaseField` แทน
`GraphQL::Schema::Field` ตรงๆ — และ `edge_type_class`/`connection_type_class` คือสิ่งที่ทำให้
Relay-style pagination (Step 580) ใช้งานได้ทันทีโดยไม่ต้องตั้งค่าเพิ่ม

### กลุ่ม 2: `app/graphql/types/query_type.rb` และ `mutation_type.rb` — Root Type

```ruby
# app/graphql/types/query_type.rb (ก่อนแก้ไข)
module Types
  class QueryType < Types::BaseObject
    field :node, Types::NodeType, null: true, description: "Fetches an object given its ID." do
      argument :id, ID, required: true, description: "ID of the object."
    end

    def node(id:)
      context.schema.object_from_id(id, context)
    end

    field :nodes, [Types::NodeType, null: true], null: true,
      description: "Fetches a list of objects given a list of IDs." do
      argument :ids, [ID], required: true, description: "IDs of the objects."
    end

    def nodes(ids:)
      ids.map { |id| context.schema.object_from_id(id, context) }
    end

    # Add root-level fields here.
    # They will be entry points for queries on your schema.

    # TODO: remove me
    field :test_field, String, null: false,
      description: "An example field added by the generator"
    def test_field
      "Hello World!"
    end
  end
end
```

`QueryType` คือจุดเริ่มต้นของทุก query — field ทุกอันที่ประกาศตรงนี้คือสิ่งที่ client เรียกได้
โดยตรงจาก root (`{ post(id: 1) { ... } }`) ส่วน `node`/`nodes` ที่ generator ใส่มาให้เป็นกลไก
Relay Global Object Identification (ค้นหา object จาก UUID แบบไม่สนใจ type — ไม่ได้ใช้ใน Part นี้
แต่เก็บไว้เพราะ Relay-style connection ใน Step 580 อ้างอิงแนวคิดเดียวกัน) `test_field` เป็นตัวอย่าง
ที่ generator ใส่มาให้ทดสอบว่าติดตั้งสำเร็จ — เราจะลบทิ้งใน Step 574

`MutationType` มีโครงสร้างเดียวกันแต่ว่างเปล่า เพราะ mutation ยังไม่ได้สอนจน Part 059:

```ruby
# app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    # TODO: remove me
    field :test_field, String, null: false,
      description: "An example field added by the generator"
    def test_field
      "Hello World"
    end
  end
end
```

### กลุ่ม 3: `app/graphql/graphql_demo_schema.rb` — Schema class ตัวหลัก

ไฟล์นี้สำคัญที่สุด เพราะเป็นจุดประกอบร่างทุกอย่างเข้าด้วยกัน ชื่อไฟล์อิงจากชื่อแอป (ที่นี่คือ
`graphql_demo` → `GraphqlDemoSchema`):

```ruby
# app/graphql/graphql_demo_schema.rb
class GraphqlDemoSchema < GraphQL::Schema
  mutation(Types::MutationType)
  query(Types::QueryType)

  # For batch-loading (see https://graphql-ruby.org/dataloader/overview.html)
  use GraphQL::Dataloader

  # GraphQL-Ruby calls this when something goes wrong while running a query:
  def self.type_error(err, context)
    super
  end

  # Union and Interface Resolution
  def self.resolve_type(abstract_type, obj, ctx)
    raise(GraphQL::RequiredImplementationMissingError)
  end

  # Limit the depth and size of incoming queries:
  max_depth(15)
  max_query_string_tokens(5000)

  # Stop validating when it encounters this many errors:
  validate_max_errors(100)

  # Relay-style Object Identification:
  def self.id_from_object(object, type_definition, query_ctx)
    object.to_gid_param
  end

  def self.object_from_id(global_id, query_ctx)
    GlobalID.find(global_id)
  end
end
```

รายละเอียดที่ควรสังเกตทุกบรรทัด:

- **`mutation(...)` / `query(...)`** — ผูก root type ทั้งสองเข้ากับ schema นี้ (ทุก query/mutation
  ที่ client ส่งมาต้องเริ่มจาก field ที่ประกาศไว้ใน type เหล่านี้)
- **`use GraphQL::Dataloader`** — บรรทัดนี้สำคัญมากและมักถูกมองข้าม เพราะเป็นสิ่งที่ทำให้ Step 578
  ใช้งานได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม graphql-ruby เวอร์ชันปัจจุบันเปิดใช้
  **`GraphQL::Dataloader`** เป็นค่าเริ่มต้นตั้งแต่ generator สร้าง schema ให้เลย — นี่คือหลักฐาน
  ว่า Dataloader คือแนวทางที่ทีม graphql-ruby แนะนำอย่างเป็นทางการสำหรับแก้ปัญหา N+1 ในปัจจุบัน
  (ไม่ใช่ gem แยกต่างหากอย่าง `graphql-batch` ที่เคยเป็นมาตรฐานก่อนหน้านี้ — รายละเอียดเต็มอยู่ใน
  Step 578)
- **`max_depth(15)`** — จำกัดความลึกของ nested query ไม่ให้เกิน 15 ชั้น ป้องกัน client ยิง query
  ที่ nest ลึกจนระเบิด server (เชื่อมโยงกับตารางเปรียบเทียบใน Step 571 เรื่อง "ควบคุม cost ยากกว่า
  REST" — นี่คือ safeguard ตัวแรกที่ generator ใส่มาให้)
- **`max_query_string_tokens(5000)`** และ **`validate_max_errors(100)`** — จำกัดขนาด query string
  และจำนวน validation error ที่จะรายงาน ป้องกัน query ที่ประดิษฐ์มาทำให้ server ทำงานหนักตอน parse
  (denial of service ระดับ parser)
- **`id_from_object`/`object_from_id`** — ใช้ Rails' `GlobalID` แปลง object เป็น UUID string และ
  กลับกัน รองรับกลไก `node(id: ...)` ที่เห็นใน `QueryType`

### กลุ่ม 4: `app/controllers/graphql_controller.rb` — จุดเข้า HTTP เดียว

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  # If accessing from outside this domain, nullify the session
  # This allows for outside API access while preventing CSRF attacks,
  # but you'll have to authenticate your user separately
  # protect_from_forgery with: :null_session

  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    context = {
      # Query context goes here, for example:
      # current_user: current_user,
    }
    result = GraphqlDemoSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
    render json: result
  rescue StandardError => e
    raise e unless Rails.env.development?
    handle_error_in_development(e)
  end

  private

  def prepare_variables(variables_param)
    case variables_param
    when String
      variables_param.present? ? (JSON.parse(variables_param) || {}) : {}
    when Hash
      variables_param
    when ActionController::Parameters
      variables_param.to_unsafe_hash
    when nil
      {}
    else
      raise ArgumentError, "Unexpected parameter: #{variables_param}"
    end
  end

  def handle_error_in_development(e)
    logger.error e.message
    logger.error e.backtrace.join("\n")
    render json: { errors: [{ message: e.message, backtrace: e.backtrace }], data: {} }, status: 500
  end
end
```

สังเกตว่าไม่ว่า query จะซับซ้อนแค่ไหน controller มี **action เดียว** (`execute`) เสมอ ทุกอย่างที่
"routing" ตาม resource ใน REST ถูกย้ายไปอยู่ใน **query string ของ GraphQL เอง** แทน — นี่คือผลจาก
หลักการ "single endpoint" ใน Step 571 บรรทัด `# protect_from_forgery with: :null_session` ที่
comment ไว้คือสิ่งที่ Step 575 จะต้องเปิดใช้งาน เพราะ Rails ป้องกัน CSRF กับทุก `POST` request ที่
ไม่มี token มาด้วยเป็นค่าเริ่มต้น (แอปแบบ full-stack ไม่ใช่ `--api` mode) และ `curl`/mobile client
ไม่มีทางส่ง CSRF token มาด้วยได้ตามธรรมชาติ

### กลุ่ม 5: routes, Gemfile, และ config อื่นๆ

```ruby
# config/routes.rb (ส่วนที่ generator เพิ่ม)
if Rails.env.development?
  mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
end
post "/graphql", to: "graphql#execute"
```

```bash
bin/rails routes | grep -i graph
```

```
    graphiql_rails      /graphiql          GraphiQL::Rails::Engine {:graphql_path=>"/graphql"}
           graphql POST /graphql(.:format) graphql#execute
Routes for GraphiQL::Rails::Engine:
       GET  /           graphiql/rails/editors#show
```

**`/graphiql`** คือ web-based IDE สำหรับทดลอง query แบบ interactive (มี autocomplete จาก schema,
ดู documentation ของทุก type ได้ทันที) mount เฉพาะ `development` เท่านั้นเพื่อไม่ให้หลุดไป
production โดยไม่ตั้งใจ ส่วน route จริงที่รับ query ทุกตัวคือ **`POST /graphql`** เพียงเส้นเดียว

Generator ยังเพิ่ม gem 2 ตัวใน `Gemfile`:

```ruby
gem "graphql"
gem "graphiql-rails", group: :development
```

และเพิ่ม config สำหรับ query log tags ใน `config/application.rb` ที่มีประโยชน์มากตอน debug N+1
(Step 577 จะใช้ประโยชน์จากส่วนนี้โดยตรง):

```ruby
# config/application.rb (ส่วนที่ generator เพิ่ม)
config.active_record.query_log_tags_enabled = true
config.active_record.query_log_tags = [
  :application, :controller, :action, :job,
  current_graphql_operation: -> { GraphQL::Current.operation_name },
  current_graphql_field: -> { GraphQL::Current.field&.path },
  current_dataloader_source: -> { GraphQL::Current.dataloader_source_class },
]
```

Config นี้ทำให้ทุกบรรทัด SQL log มี comment ต่อท้ายบอกว่า **query นี้เกิดจาก GraphQL field ไหน**
(`current_graphql_field='Post.comments'`) และถ้าเกิดจาก Dataloader source ตัวไหนก็บอกด้วย
(`current_dataloader_source='Sources::CommentsForPost'`) — เป็นเครื่องมือ debug N+1 ที่ทรงพลังมาก
ที่ generator ติดตั้งให้ฟรีตั้งแต่ต้น ไม่ต้องพึ่ง gem อย่าง Bullet (ซึ่งจะเจาะลึกสำหรับ REST ใน
Part 064) เลยด้วยซ้ำ

---

## Step 573: Object Type คือกลไกอะไร — นิยาม `Types::PostType`

### Object Type คืออะไร

**Object Type** คือการประกาศว่า "ข้อมูลชนิดนี้มี field อะไรบ้าง type อะไร nullable หรือไม่" —
เทียบเท่ากับ Serializer ใน REST (Part 056) แต่ต่างกันตรงที่ Object Type เป็นส่วนหนึ่งของ
**Schema ที่ตรวจสอบได้เอง** (introspectable) ไม่ใช่แค่ hash ที่แปลงเป็น JSON เฉยๆ

สร้าง `Types::PostType`:

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :body, String, null: false
  end
end
```

และ `Types::CommentType`:

```ruby
# app/graphql/types/comment_type.rb
module Types
  class CommentType < Types::BaseObject
    field :id, ID, null: false
    field :body, String, null: false
    field :commenter, String, null: false
  end
end
```

### กายวิภาคของ `field`

```ruby
field :title, String, null: false
```

- **`:title`** — ชื่อ field ฝั่ง Ruby เขียนแบบ `snake_case` ตามธรรมเนียม Ruby เสมอ
- **`String`** — GraphQL scalar type ของ field นี้ (รายละเอียดเต็มเรื่อง scalar อยู่ใน Step 579)
- **`null: false`** — field นี้ **ห้ามเป็น `nil`** ถ้า resolver คืนค่า `nil` มาจริง GraphQL จะโยน
  `GraphQL::InvalidNullError` แทนที่จะคืน `null` เงียบๆ — นี่คือส่วนหนึ่งของ "schema เป็นสัญญา"
  ตาม Step 571: client อ่าน schema แล้วมั่นใจได้ 100% ว่า `title` จะไม่มีวันเป็น `null` โดยไม่ต้อง
  เขียนโค้ดเช็ค `nil` ฝั่ง client เลย

> **จุดที่มือใหม่งงบ่อย:** ชื่อ field ฝั่ง Ruby เขียนเป็น `snake_case` (`comments_count`) แต่
> graphql-ruby จะ **แปลงเป็น `camelCase` ให้อัตโนมัติ** ตอนส่งออกไปเป็นส่วนหนึ่งของ schema
> (`commentsCount`) เพราะธรรมเนียมของ GraphQL community (ทั้ง JavaScript/TypeScript ฝั่ง client)
> ใช้ `camelCase` เป็นมาตรฐาน — เราจะเห็นสิ่งนี้ชัดเจนตอนยิง query จริงใน Step 575

### Object Type ไม่ได้ผูกกับ ActiveRecord model โดยตรง — แต่ resolve ผ่าน method เริ่มต้นได้ทันที

พฤติกรรม default ของ `field` (ถ้าไม่เขียน resolver method เอง) คือ**เรียก method ชื่อเดียวกันบน
object ที่ผูกกับ type นั้น** ดังนั้น `field :title, String, null: false` ใน `PostType` ที่ผูกกับ
`Post` instance จะเรียก `post.title` ให้อัตโนมัติ — เพราะ `Post` เป็น ActiveRecord model ที่มี
attribute `title` อยู่แล้ว จึงไม่ต้องเขียนอะไรเพิ่มเลยสำหรับ field พื้นฐานที่ตรงกับ column ตรงๆ
(นี่คือเหตุผลที่ `PostType` ด้านบนไม่มี method ใดๆ เลยแต่ก็ใช้งานได้ทันที — จะพิสูจน์จริงใน
Step 575)

field `id` ที่มี type `ID` แม้ column จริงในฐานข้อมูลเป็น `integer` แต่ GraphQL scalar `ID` จะ
serialize ออกมาเป็น **string เสมอ** (`"1"` ไม่ใช่ `1`) — เป็นธรรมเนียมของ GraphQL spec ที่ถือว่า
`ID` เป็น type สำหรับ "ระบุตัวตน" ไม่ใช่ type สำหรับคำนวณทางคณิตศาสตร์ จึงไม่ควรผูกกับชนิดตัวเลข
ของภาษาใดภาษาหนึ่งตรงๆ (รายละเอียดเรื่อง scalar เต็มรูปแบบอยู่ใน Step 579)

---

## Step 574: Query root type — เพิ่ม field เข้า entry point ของ schema

มี Object Type แล้วยังไม่พอ — ต้องมี **ทางเข้า** ให้ client เรียกถึง `PostType` ได้ ทางเข้านั้นคือ
field ที่ประกาศไว้ใน `QueryType` (root query type ที่ผูกไว้ใน schema ตั้งแต่ Step 572)

ลบ `test_field` ตัวอย่างออก แล้วเพิ่ม field จริง:

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :node, Types::NodeType, null: true, description: "Fetches an object given its ID." do
      argument :id, ID, required: true, description: "ID of the object."
    end

    def node(id:)
      context.schema.object_from_id(id, context)
    end

    field :nodes, [Types::NodeType, null: true], null: true,
      description: "Fetches a list of objects given a list of IDs." do
      argument :ids, [ID], required: true, description: "IDs of the objects."
    end

    def nodes(ids:)
      ids.map { |id| context.schema.object_from_id(id, context) }
    end

    # Add root-level fields here.
    # They will be entry points for queries on your schema.

    field :post, Types::PostType, null: true do
      argument :id, ID, required: true
    end

    def post(id:)
      Post.find_by(id: id)
    end

    field :posts, [Types::PostType], null: false

    def posts
      Post.all
    end
  end
end
```

### `field :post` พร้อม argument

```ruby
field :post, Types::PostType, null: true do
  argument :id, ID, required: true
end

def post(id:)
  Post.find_by(id: id)
end
```

- **`argument :id, ID, required: true`** — ประกาศว่า field นี้ **ต้อง** รับ argument ชื่อ `id`
  เสมอ ถ้า client เรียก `{ post { title } }` โดยไม่ใส่ `id` มา GraphQL จะปฏิเสธ query ตั้งแต่ชั้น
  validation **ก่อน** ที่ resolver method จะถูกเรียกด้วยซ้ำ (พิสูจน์จริงใน Step 575)
- **`def post(id:)`** — resolver method รับ argument เป็น keyword argument ชื่อเดียวกับที่ประกาศ
  ไว้ (`id:`) เขียนโค้ดตรงไปตรงมาเหมือน method ธรรมดา ไม่มี magic ซ่อนอยู่
- **`null: true` บน field** — ตั้งใจให้ nullable เพราะ `Post.find_by(id: id)` คืน `nil` ได้ถ้าไม่
  เจอโพสต์ที่มี id นั้น (ต่างจาก `find!` ที่จะ raise exception) — การเลือก `find_by` + `null: true`
  คือการออกแบบที่ถูกต้อง: "ไม่เจอ" ควรเป็น `null` ธรรมดาในผลลัพธ์ ไม่ใช่ error ที่ทำให้ทั้ง query
  ล้มเหลว

### `field :posts` — list type และความหมายของ nullability ที่ซ้อนกัน

```ruby
field :posts, [Types::PostType], null: false

def posts
  Post.all
end
```

`[Types::PostType]` คือ list type — แต่ nullability ของ "list เอง" กับ "item ข้างในแต่ละตัว" เป็น
**คนละเรื่องกัน** และเป็นจุดที่มือใหม่สับสนบ่อยที่สุด ตรวจสอบ type signature จริงด้วย
`to_type_signature` (`!` หลัง type หมายถึง non-null ตาม SDL syntax มาตรฐานของ GraphQL):

```ruby
# ทดลองใน rails runner
Types::QueryType.fields["posts"].type.to_type_signature
# => "[Post!]!"
```

ตารางสรุปพฤติกรรมจริงที่ทดสอบแล้ว (ตัวแปรคือตำแหน่งที่ใส่ `null:` — ที่ field เอง กับที่ในวงเล็บ
ของ list):

| ประกาศแบบ Ruby | Type signature ที่ได้ | ความหมาย |
|---|---|---|
| `field :x, [Types::PostType], null: false` | `[Post!]!` | field ไม่มีวันเป็น `null` **และ** ไม่มี item ไหนในลิสต์เป็น `null` ได้เลย |
| `field :x, [Types::PostType, null: true], null: false` | `[Post]!` | field ไม่มีวันเป็น `null` แต่ **item ข้างในเป็น `null` ได้** (เช่น บาง id หาไม่เจอ) |
| `field :x, [Types::PostType], null: true` | `[Post!]` | field เป็น `null` ได้ทั้งก้อน (เช่นยังไม่ query) แต่ถ้ามี list มาจริงจะไม่มี item ไหนเป็น `null` |

`field :posts` ในตัวอย่างของเราใช้แบบแรก (`[Post!]!`) ซึ่งถูกต้องตามความหมาย: `Post.all` ไม่มีวัน
คืน `nil`, คืน `ActiveRecord::Relation` ว่างเปล่าอย่างมากที่สุด (list ว่าง `[]` ยังนับเป็น "ไม่
null" ตาม GraphQL — ต่างจาก field เป็น `null` โดยสิ้นเชิง) และแต่ละ `Post` ใน relation ก็เป็น
record จริงเสมอ ไม่มีทางเป็น `nil` ปะปนอยู่ในลิสต์

---

## Step 575: รัน query จริงผ่าน GraphiQL และ `curl POST /graphql`

### ผ่าน GraphiQL (สำหรับพัฒนา/ทดลองด้วยมือ)

```bash
bin/rails server
```

เปิดเบราว์เซอร์ไปที่ `http://localhost:3000/graphiql` จะเจอ editor ฝั่งซ้ายให้พิมพ์ query และปุ่ม
Play สำหรับรัน พร้อม autocomplete จาก schema ทันที (พิมพ์ `{ p` แล้วกด `Ctrl+Space` จะเห็นตัวเลือก
`post`/`posts` ขึ้นมาจาก schema จริง ไม่ใช่การเดา) — เหมาะมากสำหรับพัฒนา แต่ใน Part นี้เราจะเน้น
`curl` เพื่อให้เห็นว่า protocol เบื้องหลังเป็น HTTP ธรรมดาที่ไม่มี magic อะไรซ่อนอยู่

### ผ่าน `curl` — ต้องเปิด CSRF exception ก่อน

ลองยิง query ตรงๆ ก่อน:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { id title } }"}'
```

ถ้ายังไม่แก้ controller จะได้ error ทันที:

```
ActionController::InvalidAuthenticityToken (Can't verify CSRF token authenticity.)
```

นี่คือสิ่งที่ Step 572 ทิ้งปมไว้ — เพราะแอปนี้เป็น Rails app แบบเต็ม (ไม่ใช่ `--api` mode) ทุก
controller ที่สืบทอดจาก `ApplicationController` มี CSRF protection เปิดอยู่โดยปริยาย ซึ่งออกแบบมา
ป้องกัน form submission ที่ไม่มี token จากหน้าเว็บของตัวเอง — แต่ `curl`, mobile app, หรือ
JavaScript client จากโดเมนอื่นไม่มีทางมี session/token นั้นได้ตามธรรมชาติ ต้องเปิด comment ที่
generator เตรียมไว้ให้:

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  # If accessing from outside this domain, nullify the session
  # This allows for outside API access while preventing CSRF attacks,
  # but you'll have to authenticate your user separately
  protect_from_forgery with: :null_session
  # ...
end
```

`with: :null_session` หมายถึง "ถ้าไม่มี CSRF token ที่ถูกต้อง ให้ทำเหมือนไม่มี session แทนที่จะ
โยน exception" — request ยังทำงานต่อได้ปกติ แต่ `session`/`current_user` ที่ผูกกับ cookie จะว่าง
เปล่า (comment บอกไว้ตรงๆ ว่า "ต้องแยก authenticate เอง" — ซึ่งจะเจาะลึกเรื่อง authorization ของ
GraphQL ใน Part 059)

รันใหม่หลังแก้:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { id title } }"}'
```

ผลลัพธ์จริง:

```json
{
  "data": {
    "posts": [
      { "id": "1", "title": "GraphQL คืออะไร" },
      { "id": "2", "title": "REST vs GraphQL" },
      { "id": "3", "title": "โพสต์ที่ยังไม่มีคอมเมนต์" }
    ]
  }
}
```

สังเกต 2 อย่าง: **`id` เป็น string** (`"1"` ไม่ใช่ `1`) ตามที่อธิบายไว้ใน Step 573 และ response
มีแค่ 2 field ที่ query ขอมา (`id`, `title`) ไม่มี `body` ติดมาด้วยเลยทั้งที่ `PostType` มี field
`body` — เพราะ **client เป็นคนเลือก** field ที่จะได้ ตรงตามหลักการ Step 571 เป๊ะๆ

### query แบบมี argument และ variable

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"query($id: ID!) { post(id: $id) { title body } }","variables":{"id":"2"}}'
```

```json
{
  "data": {
    "post": {
      "title": "REST vs GraphQL",
      "body": "เนื้อหาโพสต์สอง"
    }
  }
}
```

`variables` คือวิธีมาตรฐานในการส่งค่าตัวแปรเข้า query แทนการต่อ string เอง (เหมือนหลักการ
parameterized query ที่ใช้กับ SQL ใน Part 034 — ป้องกันปัญหา injection ในตัว)

### query โพสต์ที่ไม่มีอยู่จริง — ได้ `null` แบบ graceful

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ post(id: 999) { title } }"}'
```

```json
{ "data": { "post": null } }
```

ตรงตามที่ออกแบบไว้ใน Step 574 — `null: true` ทำให้ "ไม่เจอ" เป็นแค่ `null` ธรรมดา ไม่ใช่ HTTP
error

### ลืมใส่ argument ที่ required — error ตั้งแต่ก่อน resolver ทำงาน

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ post { title } }"}'
```

```json
{
  "errors": [
    {
      "message": "Field 'post' is missing required arguments: id",
      "locations": [{ "line": 1, "column": 3 }],
      "path": ["query", "post"],
      "extensions": {
        "code": "missingRequiredArguments",
        "className": "Field",
        "name": "post",
        "arguments": "id"
      }
    }
  ]
}
```

ไม่มี `data` key เลยในกรณีนี้ (ต่างจากกรณี `post(id: 999)` ที่ได้ `"data": {"post": null}`) —
เพราะ error นี้เกิดจาก **validation layer ก่อน execution เริ่มด้วยซ้ำ** ทั้ง query ถือว่าไม่ผ่าน
ไวยากรณ์/schema เลย ผลลัพธ์เดียวกันจะเกิดถ้า query ขอ field ที่ไม่มีจริงในเมื่อ schema เป็น
"สัญญา" ตายตัว:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { id nonExistentField } }"}'
```

```json
{
  "errors": [
    {
      "message": "Field 'nonExistentField' doesn't exist on type 'Post'",
      "locations": [{ "line": 1, "column": 14 }],
      "path": ["query", "posts", "nonExistentField"],
      "extensions": {
        "code": "undefinedField",
        "typeName": "Post",
        "fieldName": "nonExistentField"
      }
    }
  ]
}
```

นี่คือประโยชน์ของ strongly-typed schema ที่พูดถึงใน Step 571 แบบจับต้องได้: client พิมพ์ field
ผิดจะรู้ทันทีตั้งแต่ก่อน deploy (เพราะ tool ส่วนใหญ่ตรวจ query กับ schema ตอน build time) ไม่ต้อง
รอเจอ `NoMethodError` ตอน production เหมือน REST ที่ serializer อาจ silently ไม่ raise อะไรถ้า
เขียนโค้ดพลาด

---

## Step 576: Nested field — ดึง `Post` พร้อม `Comments` ในคำขอเดียว (หัวใจของ GraphQL)

นี่คือ feature เด่นที่สุดของ GraphQL ที่ REST ทำได้ยากกว่ามาก — เพิ่ม field ความสัมพันธ์เข้าไปใน
`PostType`:

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :body, String, null: false
    field :comments, [Types::CommentType], null: false
  end
end
```

และเพิ่ม field `post` กลับใน `CommentType` เพื่อให้ query ย้อนกลับได้ทั้งสองทิศทาง (เหมือน
`belongs_to`/`has_many` สองทาง):

```ruby
# app/graphql/types/comment_type.rb
module Types
  class CommentType < Types::BaseObject
    field :id, ID, null: false
    field :body, String, null: false
    field :commenter, String, null: false
    field :post, Types::PostType, null: false
  end
end
```

**ไม่ต้องเขียน resolver method เพิ่มเลย** เพราะ default behavior ของ `field` (Step 573) เรียก
`post.comments` และ `comment.post` ให้อัตโนมัติ ซึ่งตรงกับชื่อ association ที่ประกาศไว้ใน
ActiveRecord model พอดี

### ยิง query ที่ขอ `Post` พร้อม `Comments` ในคำขอเดียว

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"query($id: ID!) { post(id: $id) { id title body comments { id commenter body } } }","variables":{"id":"1"}}'
```

ผลลัพธ์จริง:

```json
{
  "data": {
    "post": {
      "id": "1",
      "title": "GraphQL คืออะไร",
      "body": "เนื้อหาโพสต์แรก",
      "comments": [
        { "id": "1", "commenter": "มานี", "body": "บทความดีมาก" },
        { "id": "2", "commenter": "ปิติ", "body": "อยากรู้เรื่อง N+1 ต่อ" }
      ]
    }
  }
}
```

**หนึ่ง HTTP request หนึ่ง response ได้ข้อมูลจากสองตาราง (`posts` + `comments`) ครบตามโครงสร้างที่
ขอ** — เทียบกับ REST ที่ต้องยิงอย่างน้อย 2 endpoint (`GET /posts/1` แล้ว `GET /posts/1/comments`)
หรือต้องมีคนออกแบบ endpoint พิเศษ `GET /posts/1?include=comments` ไว้ล่วงหน้า (แนวคิด `include`
แบบ JSON:API ที่เรียนใน Part 057) นี่คือการสาธิตให้เห็นจริงถึงสิ่งที่ Step 571 อธิบายไว้เป็นทฤษฎี

### สลับทิศทาง — จาก Comment ย้อนกลับไปหา Post

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { comments { commenter post { title } } } }"}'
```

```json
{
  "data": {
    "posts": [
      {
        "comments": [
          { "commenter": "มานี", "post": { "title": "GraphQL คืออะไร" } },
          { "commenter": "ปิติ", "post": { "title": "GraphQL คืออะไร" } }
        ]
      },
      {
        "comments": [
          { "commenter": "ชูใจ", "post": { "title": "REST vs GraphQL" } }
        ]
      },
      { "comments": [] }
    ]
  }
}
```

สังเกตว่า **client เป็นคนตัดสินใจความลึกของ nesting เอง** — จาก `posts` ไป `comments` ไป `post`
กลับมาอีกที (loop กลับไปกลับมาได้เรื่อยๆ ถ้า schema เปิดทางไว้) นี่คือความยืดหยุ่นที่ทรงพลังที่สุด
ของ GraphQL แต่ก็เป็นสาเหตุโดยตรงของปัญหาที่ Step 577 กำลังจะเปิดโปง

---

## Step 577: ปัญหา N+1 ใน GraphQL — อันตรายกว่าที่คิด

### ทำไม N+1 ใน GraphQL ถึงร้ายกว่า REST

ใน REST (Part 034) N+1 เกิดจาก**โค้ด server เอง** เขียน loop แล้วเรียก association โดยไม่ได้
`includes` ไว้ — ทีม backend ควบคุมจุดเกิดปัญหาได้ทั้งหมด เพราะรู้ล่วงหน้าว่า endpoint ไหน serialize
field อะไรบ้าง (serializer เขียนไว้ตายตัว)

แต่ใน GraphQL **client เป็นคนเลือก field เอง** โดยที่ทีม backend ไม่รู้ล่วงหน้าว่า query ไหนจะถูก
ส่งมาบ้าง วันนี้ query ขอแค่ `{ posts { title } }` ไม่มีปัญหาอะไร แต่พรุ่งนี้ client เพิ่ม field
`comments { commenter }` เข้าไปเฉยๆ (ไม่ต้องแก้โค้ด server แม้แต่บรรทัดเดียว — schema เปิดทางให้
อยู่แล้วตั้งแต่ Step 576) ก็เกิด N+1 ทันทีโดยไม่มีใคร deploy โค้ดใหม่เลย นี่คือความหมายของ "อันตราย
กว่า" ที่พูดถึงในตาราง Step 571

### พิสูจน์ด้วย SQL log จริง

เปิด logger แล้วยิง query ที่ขอ `posts` พร้อม `comments` (ยังไม่แก้ N+1):

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { title comments { commenter } } }"}'
```

SQL log ที่เกิดขึ้นจริง (มี `current_graphql_field` ต่อท้ายจาก config ที่ Step 572 ติดตั้งไว้ให้):

```
Post Load (2.1ms)  SELECT "posts".* FROM "posts" /*current_graphql_field='Query.posts'*/
Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = 1 /*current_graphql_field='Post.comments'*/
Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = 2 /*current_graphql_field='Post.comments'*/
Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = 3 /*current_graphql_field='Post.comments'*/
```

**1 query สำหรับ posts + 1 query แยกต่างหากต่อโพสต์แต่ละอันสำหรับ comments = 4 query** สำหรับ
โพสต์แค่ 3 อัน — สังเกตว่า comment ที่ต่อท้ายแต่ละบรรทัดบอกชัดเจนว่า query นั้นมาจาก field ไหน
(`Post.comments`) ซึ่งเป็นเครื่องมือ debug ที่มีค่ามากเวลาสืบว่า N+1 มาจาก field ตัวไหนใน schema
ใหญ่ๆ ที่มี type เป็นร้อย

**เหตุผลที่ N+1 เกิดขึ้น:** field `comments` ของ `PostType` ใช้ default resolver ที่เรียก
`post.comments` ตรงๆ ตอน execution engine loop ผ่าน `Post` แต่ละอันใน list เพื่อ resolve field
`comments` มันเรียก `post.comments` ทีละ post แยกกัน **ไม่รู้ล่วงหน้าว่า post อื่นในลิสต์เดียวกัน
ก็ต้องการ comments เหมือนกัน** — นี่คือกลไกเดียวกับที่ทำให้ `Post.all.each { |p| p.comments }` เกิด
N+1 ใน REST (Part 034) เพียงแต่ตรงนี้ **client เป็นคนกระตุ้นให้เกิดจากการเลือก field เท่านั้น**
ไม่มีใครใน backend เขียนโค้ด loop ผิดพลาดเลย

### ทำไม `includes` แบบ REST ใช้ไม่ได้ตรงๆ ที่นี่

ใน REST เราแก้ด้วย `Post.includes(:comments)` ที่จุดเริ่มต้น controller เพราะรู้ล่วงหน้าแน่นอนว่า
serializer จะ render `comments` เสมอ แต่ใน GraphQL `Query.posts` resolver (`Post.all`) **ไม่รู้
ล่วงหน้าว่า client จะขอ field `comments` มาด้วยหรือเปล่า** — ถ้าใส่ `includes(:comments)` แบบเหมา
รวมไว้ทุกครั้งจะเสีย performance โดยใช่เหตุตอน client ขอแค่ `{ posts { title } }` (เจอปัญหา
over-fetching ที่ระดับ database แทนที่จะเป็นระดับ HTTP response) วิธีที่ถูกต้องคือให้ field
`comments` เอง**ตัดสินใจแบบ batch เฉพาะตอนที่ถูกเรียกจริง** — นี่คือสิ่งที่ `GraphQL::Dataloader`
ทำให้ใน Step 578

---

## Step 578: แก้ N+1 ด้วย `GraphQL::Dataloader`

### `GraphQL::Dataloader` คืออะไร และทำไมถึงเป็นตัวเลือกที่แนะนำในปัจจุบัน

`GraphQL::Dataloader` เป็น mechanism ที่ built-in มากับ graphql-ruby เอง (ตั้งแต่เวอร์ชัน 1.12 เป็น
ต้นไป) และเป็น**ค่าเริ่มต้นที่ generator เปิดใช้ให้แล้ว** ตั้งแต่ Step 572
(`use GraphQL::Dataloader` ใน schema class) — แนวคิดคือ: แทนที่ field resolver จะยิง query ทันที
ที่ถูกเรียก มันจะ "รอ" ให้ execution engine เก็บ key ที่ต้องการมาให้ครบก่อน (เช่น เก็บ post id
ทั้งหมดที่ต้องการ comments) แล้วค่อยยิง **query เดียวที่ query ทีเดียวสำหรับทุก key พร้อมกัน**
(`WHERE post_id IN (...)`)

ก่อนหน้านี้ (graphql-ruby เวอร์ชันเก่ากว่า) วิธีมาตรฐานคือใช้ gem แยกต่างหากชื่อ **`graphql-batch`**
(ของ Shopify เช่นกัน) ซึ่งใช้แนวคิดคล้ายกันแต่ต้องติดตั้งเพิ่มและมี API ที่ต่างออกไป (`Loader`
class + `Promise`) ปัจจุบัน `GraphQL::Dataloader` เข้ามาแทนที่ในฐานะ built-in solution ของ
graphql-ruby เอง โปรเจกต์ใหม่จึงควรเริ่มด้วย `GraphQL::Dataloader` ตั้งแต่ต้น (`graphql-batch` ยัง
ใช้งานได้และมีในหลายโปรเจกต์เก่า แต่ไม่ใช่แนวทางที่ทีม graphql-ruby ผลักดันต่อแล้ว)

### เขียน `Source` class

Dataloader ทำงานผ่านการประกาศ **`GraphQL::Dataloader::Source`** — คลาสที่นิยาม "จะดึงข้อมูลชุด
หนึ่งมาแบบ batch ได้อย่างไร":

```ruby
# app/graphql/sources/comments_for_post.rb
module Sources
  class CommentsForPost < GraphQL::Dataloader::Source
    def fetch(posts)
      comments_by_post_id = Comment.where(post_id: posts.map(&:id)).group_by(&:post_id)
      posts.map { |post| comments_by_post_id[post.id] || [] }
    end
  end
end
```

- **`fetch(posts)`** — รับ **array ของทุก `Post`** ที่ต้องการ comments ในรอบ execution นี้พร้อมกัน
  ทีเดียว (ไม่ใช่ทีละอันเหมือน default resolver)
- ยิง query เดียว (`WHERE post_id IN (...)`) แล้ว `group_by` แยกผลลัพธ์กลับเป็นก้อนตาม `post_id`
- **`posts.map { |post| comments_by_post_id[post.id] || [] }`** — คืนค่าต้องเรียงลำดับตรงกับ
  `posts` ที่รับเข้ามาเป๊ะๆ (ตำแหน่งที่ 1 ของผลลัพธ์คือ comments ของ `posts[0]`) และ**ต้องใส่
  `|| []` เสมอ** สำหรับโพสต์ที่ไม่มี comment เลย (เช่นโพสต์ที่ 3 ในข้อมูลทดลองของเรา) ไม่งั้นจะได้
  `nil` ปนอยู่ในลิสต์ซึ่งขัดกับ type `[Comment!]!` ที่ประกาศไว้ทันที

แก้ `PostType` ให้ field `comments` ใช้ Source นี้แทน default resolver:

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :body, String, null: false
    field :comments, [Types::CommentType], null: false

    def comments
      dataloader.with(Sources::CommentsForPost).load(object)
    end
  end
end
```

- **`dataloader`** — helper method ที่ทุก resolver เข้าถึงได้ทันที (มาจาก `use GraphQL::Dataloader`
  ใน schema) ให้ context ปัจจุบันของ dataloader
- **`.with(Sources::CommentsForPost)`** — เลือกว่าจะใช้ Source class ไหน (ถ้ามีหลาย Source ที่รับ
  argument ต่างกัน สามารถส่ง argument เพิ่มเข้า `.with(...)` ได้ด้วย)
- **`.load(object)`** — ขอโหลดข้อมูลสำหรับ `object` (คือ `Post` instance ปัจจุบันที่ field นี้
  กำลัง resolve อยู่) — ตรงนี้เองที่ dataloader "รวบ" คำขอจากทุก `Post` ในรอบเดียวกันก่อนยิง query
  จริง

### พิสูจน์ผลลัพธ์ด้วย SQL log — จาก N+1 เหลือ 2 query คงที่

ยิง query เดิมซ้ำอีกครั้งหลังแก้:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { title comments { commenter } } }"}'
```

SQL log ที่เกิดขึ้นจริงหลังแก้:

```
Post Load (2.1ms)  SELECT "posts".* FROM "posts" /*current_graphql_field='Query.posts'*/
Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3) /*current_dataloader_source='Sources::CommentsForPost'*/
```

จาก **4 query** (1 posts + 3 comments แยกทีละโพสต์) เหลือ **2 query คงที่** (1 posts + 1 comments
แบบ `IN (...)` เดียว) — และตัวเลข 2 นี้จะ **คงที่ไม่ว่าจะมีโพสต์กี่ร้อยกี่พันอันก็ตาม** เหมือน
หลักการเดียวกับ `includes` ใน REST (Part 034 Step 335) เพียงแต่คราวนี้ทำงานที่ **ระดับ field
resolver** แทนที่จะเป็นระดับ controller เพราะ backend ไม่รู้ล่วงหน้าว่า client จะขอ field ไหนบ้าง
อย่างที่อธิบายไว้ท้าย Step 577

ผลลัพธ์ JSON ยังถูกต้องเหมือนเดิมทุกประการ — Dataloader เปลี่ยนแค่ **วิธียิง SQL** ไม่เปลี่ยน
รูปร่าง response ที่ client เห็นเลย:

```json
{
  "data": {
    "posts": [
      { "title": "GraphQL คืออะไร", "comments": [{ "commenter": "มานี" }, { "commenter": "ปิติ" }] },
      { "title": "REST vs GraphQL", "comments": [{ "commenter": "ชูใจ" }] },
      { "title": "โพสต์ที่ยังไม่มีคอมเมนต์", "comments": [] }
    ]
  }
}
```

> **ข้อควรระวัง:** N+1 ใน GraphQL ซ่อนอยู่ได้ทุก field ที่ resolve ผ่าน association ไม่ใช่แค่
> field ที่เป็น list เท่านั้น ตัวอย่างเช่นถ้าเพิ่ม `field :comments_count, Integer, null: false`
> ที่เขียน `object.comments.size` ตรงๆ (ไม่ผ่าน Dataloader) มันจะสร้าง `SELECT COUNT(*)` แยกต่างหาก
> ต่อโพสต์อีกชุดหนึ่ง กลายเป็น N+1 ซ่อนอยู่ในอีก field ที่ดูเหมือนไม่เกี่ยวกับ Step 577 เลย —
> รายละเอียดและวิธีแก้เป็นแบบฝึกหัดเพิ่มเติมท้าย Part นี้

---

## Step 579: Scalar Types vs Custom Types

### Scalar Type คืออะไร

**Scalar** คือ type ที่เป็น "ค่าเดี่ยว" ไม่มี field ย่อยข้างในให้ query ต่อ (ต่างจาก Object Type
อย่าง `PostType` ที่มี field ย่อยหลายตัว) GraphQL spec กำหนด scalar พื้นฐานไว้ 5 ตัว:

| Scalar | ตรงกับ Ruby | ตัวอย่าง |
|--------|-------------|----------|
| `Int` | `Integer` (32-bit signed) | `42` |
| `Float` | `Float` | `3.14` |
| `String` | `String` | `"hello"` |
| `Boolean` | `true`/`false` | `true` |
| `ID` | `String` (serialize เสมอแม้ต้นทางเป็นตัวเลข) | `"1"` |

graphql-ruby เพิ่ม scalar ที่มีประโยชน์ให้เกินมาตรฐานหลายตัว เช่น `GraphQL::Types::ISO8601DateTime`
สำหรับวันเวลา ลองใช้กับ `created_at`:

```ruby
# app/graphql/types/post_type.rb
field :created_at, GraphQL::Types::ISO8601DateTime, null: false
```

ยิง query ทดสอบ:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ post(id: 1) { title createdAt } }"}'
```

```json
{ "data": { "post": { "title": "GraphQL คืออะไร", "createdAt": "2026-09-26T07:37:49Z" } } }
```

`GraphQL::Types::ISO8601DateTime` แปลง Ruby `Time`/`ActiveSupport::TimeWithZone` เป็น ISO 8601
string โดยอัตโนมัติ (`coerce_result`) และแปลงกลับเป็น `Time` object ให้อัตโนมัติถ้า client ส่งค่า
นี้เข้ามาเป็น argument (`coerce_input`) — ไม่ต้อง parse/format เองเลยทั้งสองทิศทาง

### Custom Scalar — เมื่อ scalar สำเร็จรูปไม่พอ

ถ้าต้องการ validate หรือ format รูปแบบเฉพาะ (เช่น URL, email) เขียน custom scalar ได้โดยสืบทอด
`Types::BaseScalar` แล้ว implement `coerce_input`/`coerce_result`:

```ruby
# app/graphql/types/url_type.rb
module Types
  class UrlType < Types::BaseScalar
    description "A valid HTTP/HTTPS URL string"

    def self.coerce_input(value, context)
      uri = URI.parse(value)
      raise GraphQL::CoercionError, "#{value.inspect} is not a valid URL" unless uri.is_a?(URI::HTTP)

      value
    rescue URI::InvalidURIError
      raise GraphQL::CoercionError, "#{value.inspect} is not a valid URL"
    end

    def self.coerce_result(value, context)
      value.to_s
    end
  end
end
```

- **`coerce_input`** — เรียกตอน client ส่งค่านี้เข้ามาเป็น **argument** (แปลงจาก wire format →
  Ruby object พร้อม validate) ถ้า raise `GraphQL::CoercionError` client จะได้ error message ที่
  ชัดเจนกลับไปตั้งแต่ชั้น validation
- **`coerce_result`** — เรียกตอนจะส่งค่านี้กลับไปเป็น**ผลลัพธ์** (แปลงจาก Ruby object → wire
  format)

Custom scalar เหมาะกับ "ค่าเดี่ยวที่มีกฎการ validate ของตัวเอง" ส่วน **Custom Object Type**
(อย่าง `PostType`, `CommentType` ที่เขียนมาตลอด Part นี้) เหมาะกับ "ก้อนข้อมูลที่มีหลาย field
สัมพันธ์กัน" — สองอย่างนี้ตอบโจทย์คนละแบบ อย่าใช้ scalar แทน object type เพียงเพราะอยากทำให้เรียบ
ง่าย เพราะจะเสียความสามารถ query แบบ nested field ไปทันที

### Enum Type — อีกรูปแบบของ scalar ที่จำกัดค่าที่เป็นไปได้

ถ้ามี field ที่ค่าเป็นไปได้จำกัด เช่นสถานะโพสต์ ใช้ **Enum** แทน `String` ธรรมดา เพื่อให้ schema
บังคับค่าที่ถูกต้องแทนที่จะพึ่ง validation ฝั่ง Ruby เพียงอย่างเดียว:

```ruby
# app/graphql/types/post_status_type.rb
module Types
  class PostStatusType < Types::BaseEnum
    value "DRAFT", value: "draft"
    value "PUBLISHED", value: "published"
    value "ARCHIVED", value: "archived"
  end
end
```

ถ้า client ส่งค่าที่ไม่อยู่ใน enum เข้ามาเป็น argument จะได้ validation error ทันทีเหมือน argument
ที่ผิด type อื่นๆ ที่เห็นใน Step 575 — schema เป็นคนบังคับ ไม่ต้องเขียน `if status.in?(%w[...])`
ซ้ำในทุก resolver

---

## Step 580: Connection/Pagination แบบ Relay-style cursor pagination

### ทำไมต้องมี pagination convention เฉพาะของ GraphQL

Part 057 สอน pagination แบบ REST (page-based หรือ cursor-based ผ่าน query parameter) — GraphQL
community มีธรรมเนียมของตัวเองที่เรียกว่า **Relay Connection Specification** ซึ่ง client ฝั่ง
JavaScript ยอดนิยม (Relay, Apollo Client) รองรับ pattern นี้โดยตรง (auto-merge หน้าถัดไปเข้ากับ
cache ให้อัตโนมัติ) — graphql-ruby รองรับ pattern นี้ **ในตัวทันทีโดยไม่ต้องเขียน pagination logic
เองเลย** เพราะ `BaseObject` ผูก `connection_type_class`/`edge_type_class` ไว้ให้แล้วตั้งแต่
Step 572

### เปิดใช้งาน connection field

```ruby
# app/graphql/types/query_type.rb
field :posts_connection, Types::PostType.connection_type, null: false

def posts_connection
  Post.all.order(:id)
end
```

**`Types::PostType.connection_type`** คือ method ที่ graphql-ruby generate **`PostConnection`
type ให้อัตโนมัติ** จาก `PostType` ที่มีอยู่แล้ว โดยไม่ต้องเขียน type ใหม่เอง (ยืนยันจริงด้วย
`Types::PostType.connection_type.graphql_name # => "PostConnection"`) resolver แค่คืน
`ActiveRecord::Relation` ธรรมดา (ต้อง `order` ให้ผลลัพธ์มีลำดับแน่นอนเสมอ ไม่งั้น cursor จะเลื่อน
ไม่ตรงกันระหว่างหน้า) ที่เหลือ graphql-ruby จัดการให้หมด รวมถึงแปลง `ActiveRecord::Relation` เป็น
`LIMIT`/`OFFSET` ที่เหมาะสมให้อัตโนมัติผ่าน `GraphQL::Pagination::ActiveRecordRelationConnection`

### รูปร่างของ Connection: `edges`, `node`, `pageInfo`, `cursor`

ยิงหน้าแรก ขอทีละ 1 รายการด้วย argument `first` ที่ generator เพิ่มให้อัตโนมัติ:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ postsConnection(first: 1) { edges { cursor node { id title } } pageInfo { hasNextPage endCursor } } }"}'
```

ผลลัพธ์จริง:

```json
{
  "data": {
    "postsConnection": {
      "edges": [
        { "cursor": "MQ", "node": { "id": "1", "title": "GraphQL คืออะไร" } }
      ],
      "pageInfo": { "hasNextPage": true, "endCursor": "MQ" }
    }
  }
}
```

- **`edges`** — array ของ "ขอบ" แต่ละอันมี `node` (ข้อมูลจริง) และ `cursor` (ตำแหน่งอ้างอิงสำหรับ
  ขอหน้าถัดไป) — `cursor` เป็น string ที่ถูก encode ไว้ (ที่นี่คือ base64 ของเลข `1` → `"MQ"`) ไม่
  ควรตีความหรือแกะเองฝั่ง client เพียงแค่ "ส่งกลับไปตามที่ได้มา" เท่านั้น
- **`pageInfo.hasNextPage`** — บอกว่ายังมีหน้าถัดไปหรือไม่ โดยไม่ต้องนับจำนวน record ทั้งหมดเอง
  ฝั่ง client
- **`pageInfo.endCursor`** — cursor ของรายการสุดท้ายในหน้านี้ ใช้ส่งต่อเป็น `after` เพื่อขอหน้า
  ถัดไป

ขอหน้าถัดไปด้วย `after`:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ postsConnection(first: 1, after: \"MQ\") { edges { cursor node { id title } } pageInfo { hasNextPage hasPreviousPage endCursor } } }"}'
```

```json
{
  "data": {
    "postsConnection": {
      "edges": [
        { "cursor": "Mg", "node": { "id": "2", "title": "REST vs GraphQL" } }
      ],
      "pageInfo": { "hasNextPage": true, "hasPreviousPage": true, "endCursor": "Mg" }
    }
  }
}
```

สังเกตว่า `after: "MQ"` (cursor ของรายการที่ 1) คืนรายการที่ 2 มาให้พอดี — **cursor-based
pagination** แบบนี้ไม่มีปัญหาเดียวกับ `OFFSET` ที่เจอใน Part 034 (ข้อมูลเลื่อนตำแหน่งระหว่างหน้าถ้า
มีการเพิ่ม/ลบ record ระหว่างที่ client กำลังเปิดดูหลายหน้าอยู่) เพราะ cursor อ้างอิงจาก "ตำแหน่ง
ของ record จริง" ไม่ใช่ "จำนวนที่ข้ามมา"

> **เชื่อมโยงกับ N+1:** ถ้า `PostConnection` มี field ย่อยที่ resolve ผ่าน association (เช่น
> `edges { node { comments { commenter } } }`) หลักการ Dataloader จาก Step 578 ยังใช้ได้เหมือนเดิม
> ทุกประการ — connection แค่เปลี่ยนวิธี "list ของ Post ถูกส่งมาให้" (แบ่งหน้า) ไม่ได้เปลี่ยนวิธี
> resolve field ย่อยของแต่ละ node เลย

---

## แบบฝึกหัด: สร้าง Post/Comment GraphQL Schema พร้อม Dataloader ป้องกัน N+1 และตรวจสอบด้วย `curl` จริง

### โจทย์

จากโปรเจกต์ `graphql_demo` ที่สร้างมาตลอด Part นี้ ให้ประกอบทุกอย่างเข้าด้วยกันเป็น schema ที่
สมบูรณ์ตามข้อกำหนดต่อไปนี้:

1. `PostType` มี field: `id`, `title`, `body`, `createdAt` (ใช้ `ISO8601DateTime`), `comments`
   (list ของ `CommentType`), `commentsCount` (จำนวน comment)
2. `CommentType` มี field: `id`, `body`, `commenter`, `post` (ย้อนกลับไปหา `PostType`)
3. `Query` มี field: `post(id:)` (คืน `null` ถ้าไม่เจอ), `posts` (list ทั้งหมด),
   `postsConnection` (Relay-style pagination)
4. **`comments` และ `commentsCount` ต้องไม่เกิด N+1 เลย** ไม่ว่าจะ query กี่โพสต์พร้อมกันก็ตาม —
   ต้องพิสูจน์ด้วย SQL log จริงว่ายิงกี่ query
5. ทดสอบทุกอย่างด้วย `curl` จริง ไม่ใช่แค่เขียนโค้ดแล้วเชื่อว่าใช้ได้

### เฉลย

**1) Source สำหรับ comments (เหมือน Step 578 เป๊ะ):**

```ruby
# app/graphql/sources/comments_for_post.rb
module Sources
  class CommentsForPost < GraphQL::Dataloader::Source
    def fetch(posts)
      comments_by_post_id = Comment.where(post_id: posts.map(&:id)).group_by(&:post_id)
      posts.map { |post| comments_by_post_id[post.id] || [] }
    end
  end
end
```

**2) `commentsCount` ต้องใช้ Source เดียวกัน ไม่เขียน `object.comments.size` ตรงๆ**

จุดนี้คือส่วนที่พลาดได้ง่ายที่สุด — ถ้าเขียน `def comments_count; object.comments.size; end` ตรงๆ
จะเกิด N+1 ซ่อนอยู่อีกชุดหนึ่งที่ไม่เกี่ยวกับ field `comments` เลย (พิสูจน์ได้จริงด้วย SQL log
ด้านล่าง) วิธีแก้ที่ถูกต้องคือให้ `commentsCount` **ใช้ผลลัพธ์ที่โหลดมาจาก Source เดียวกัน** แล้ว
นับความยาว array แทนที่จะยิง `COUNT(*)` แยก:

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :body, String, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :comments, [Types::CommentType], null: false
    field :comments_count, Integer, null: false

    def comments
      dataloader.with(Sources::CommentsForPost).load(object)
    end

    def comments_count
      dataloader.with(Sources::CommentsForPost).load(object).size
    end
  end
end
```

เพราะ `GraphQL::Dataloader::Source` cache ผลลัพธ์ของ `.load(object)` เดียวกันไว้ในรอบ execution
เดียวกันโดยอัตโนมัติ (เรียกซ้ำกี่ field ก็ตาม ถ้า argument ที่ส่งเข้า `.load` เหมือนกัน จะยิง
`fetch` แค่ครั้งเดียว) การเรียก `.load(object)` ซ้ำใน `comments_count` จึง**ไม่เพิ่ม query ใหม่
เลย** แค่ใช้ผลลัพธ์ที่ถูก batch มาแล้วจาก field `comments`

**3) `CommentType` พร้อมทางย้อนกลับไปหา Post:**

```ruby
# app/graphql/types/comment_type.rb
module Types
  class CommentType < Types::BaseObject
    field :id, ID, null: false
    field :body, String, null: false
    field :commenter, String, null: false
    field :post, Types::PostType, null: false
  end
end
```

**4) `QueryType` ครบทั้ง 3 field ตามโจทย์:**

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :post, Types::PostType, null: true do
      argument :id, ID, required: true
    end

    def post(id:)
      Post.find_by(id: id)
    end

    field :posts, [Types::PostType], null: false

    def posts
      Post.all
    end

    field :posts_connection, Types::PostType.connection_type, null: false

    def posts_connection
      Post.all.order(:id)
    end
  end
end
```

**5) ทดสอบด้วย `curl` จริง — ยืนยันว่าไม่มี N+1**

เปิด SQL logger แล้วยิง query ที่ขอทั้ง `comments` และ `commentsCount` พร้อมกันในคำขอเดียว (สถานการณ์
ที่มีความเสี่ยง N+1 สูงสุด เพราะมี 2 field ที่ต้องใช้ข้อมูล comments):

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ posts { id title commentsCount comments { commenter } } }"}'
```

Response ที่ได้ (ครบทั้ง 3 โพสต์ รวมโพสต์ที่ไม่มี comment เลย):

```json
{
  "data": {
    "posts": [
      {
        "id": "1", "title": "GraphQL คืออะไร", "commentsCount": 2,
        "comments": [{ "commenter": "มานี" }, { "commenter": "ปิติ" }]
      },
      {
        "id": "2", "title": "REST vs GraphQL", "commentsCount": 1,
        "comments": [{ "commenter": "ชูใจ" }]
      },
      { "id": "3", "title": "โพสต์ที่ยังไม่มีคอมเมนต์", "commentsCount": 0, "comments": [] }
    ]
  }
}
```

SQL log ที่เกิดขึ้นจริง:

```
Post Load (1.4ms)  SELECT "posts".* FROM "posts" /*current_graphql_field='Query.posts'*/
Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3) /*current_dataloader_source='Sources::CommentsForPost'*/
```

**รวมแค่ 2 query คงที่** สำหรับ 3 โพสต์ × 2 field ที่ต้องใช้ข้อมูล comments (`comments` และ
`commentsCount`) — ถ้าไม่ได้แชร์ Source เดียวกัน ตัวเลขนี้จะแตกออกเป็นอย่างน้อย 1 (posts) + 1
(comments แบบ IN) + 3 (COUNT แยกทีละโพสต์สำหรับ commentsCount) = 5 query ทันที ตามที่อธิบายไว้ใน
"ข้อควรระวัง" ท้าย Step 578

ทดสอบ nested query แบบเต็มรูปแบบอีกครั้งเพื่อยืนยันว่าโครงสร้างข้อมูลถูกต้องครบถ้วน:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"query($id: ID!) { post(id: $id) { title createdAt comments { commenter post { title } } } }","variables":{"id":"1"}}'
```

```json
{
  "data": {
    "post": {
      "title": "GraphQL คืออะไร",
      "createdAt": "2026-09-26T07:37:49Z",
      "comments": [
        { "commenter": "มานี", "post": { "title": "GraphQL คืออะไร" } },
        { "commenter": "ปิติ", "post": { "title": "GraphQL คืออะไร" } }
      ]
    }
  }
}
```

และทดสอบ `postsConnection` เพื่อยืนยันว่า pagination ทำงานถูกต้อง:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ postsConnection(first: 2) { edges { node { title } } pageInfo { hasNextPage } } }"}'
```

```json
{
  "data": {
    "postsConnection": {
      "edges": [{ "node": { "title": "GraphQL คืออะไร" } }, { "node": { "title": "REST vs GraphQL" } }],
      "pageInfo": { "hasNextPage": true }
    }
  }
}
```

`hasNextPage: true` ถูกต้อง เพราะมีโพสต์ทั้งหมด 3 อันแต่ขอมาแค่ 2 — schema, N+1 prevention, และ
pagination ทำงานถูกต้องครบทุกจุดตามโจทย์ ยืนยันด้วยการรันจริงทั้งหมด ไม่ใช่การอ่านโค้ดแล้วเชื่อ
เฉยๆ

### แบบฝึกหัดเพิ่มเติม (ทำเอง)

1. เพิ่ม field `field :author_name, String, null: false` ใน `CommentType` (สมมติว่า `Comment` มี
   column `commenter` อยู่แล้วแทนชื่อผู้เขียน) แล้วลองสร้างสถานการณ์ N+1 แบบใหม่โดยตั้งใจ: เพิ่ม
   model `Author` ที่ `Post belongs_to :author` แล้วเพิ่ม field `author` ใน `PostType` ที่ resolve
   ผ่าน `object.author` ตรงๆ (ไม่ใช้ Dataloader) ลอง query `{ posts { author { name } } }` แล้วดู
   SQL log ว่าเกิด N+1 กี่ query จากนั้นแก้ด้วย `GraphQL::Dataloader::Source` ตัวใหม่ชื่อ
   `Sources::AuthorById` ให้เหลือ 2 query คงที่เหมือนที่ทำกับ `comments`
2. ลองเปลี่ยน `posts_connection` ให้รับ argument `search: String` เพิ่มเข้าไป แล้วกรอง
   `Post.where("title LIKE ?", "%#{search}%")` ก่อนส่งเข้า connection (ระวังเรื่อง SQL injection —
   ทบทวน parameterized query จาก Part 034) ทดสอบว่า `pageInfo.hasNextPage` ยังคำนวณถูกต้องหลังกรอง
   ข้อมูลแล้วหรือไม่
3. เขียน custom scalar `Types::NonEmptyStringType` ที่ raise `GraphQL::CoercionError` ถ้า client
   ส่ง string ว่างเปล่าหรือมีแต่ whitespace เข้ามาเป็น argument แล้วลองใช้แทน `String` ธรรมดาใน
   argument `commenter` ของ mutation ในจินตนาการ (ยังไม่ต้องสร้าง mutation จริง เพราะ Part 059
   จะสอนเรื่องนี้เต็มรูปแบบ) — แค่ทดสอบว่า scalar เพียวๆ raise error ถูกต้องผ่าน
   `GraphqlDemoSchema.execute` ใน `rails runner` ก็พอ

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- **GraphQL** พลิกปรัชญาการออกแบบ API จาก "server กำหนดรูปร่าง response" (REST) เป็น
  **"client กำหนดเองทุกครั้งผ่าน query"** โดยใช้ **endpoint เดียว** (`POST /graphql`) และ
  **schema ที่ตรวจสอบได้เอง** เป็นสัญญาระหว่าง client กับ server — แก้ปัญหา over-fetching/
  under-fetching ได้ตรงจุด แต่แลกมาด้วยความซับซ้อนจริงเรื่อง **N+1 ที่เกิดง่ายกว่า** และ
  **HTTP caching ที่ทำยากกว่า** REST (ตารางเปรียบเทียบ Step 571) — ไม่มีฝั่งไหน "ดีกว่าเสมอ"
  ขึ้นกับรูปแบบ client และความต้องการจริงของระบบ
- `rails generate graphql:install` สร้างโครงทั้งหมดให้: **Base classes** สำหรับปรับพฤติกรรมรวม,
  **`QueryType`/`MutationType`** เป็น root entry point, **Schema class** ที่ผูกทุกอย่างเข้าด้วยกัน
  พร้อม safeguard (`max_depth`, `max_query_string_tokens`) และเปิด **`GraphQL::Dataloader`** เป็น
  ค่าเริ่มต้นตั้งแต่ต้น, **`GraphqlController`** ที่มี action เดียว (`execute`) รับทุก query, และ
  **`GraphiQL`** สำหรับทดลอง query แบบ interactive ใน development
- **Object Type** (`Types::PostType < Types::BaseObject`) ประกาศ field ด้วย
  `field :name, Type, null: bool` — resolve ผ่าน method ชื่อเดียวกันบน object โดยอัตโนมัติถ้าไม่
  เขียน resolver เอง, ชื่อ field แปลงจาก `snake_case` เป็น `camelCase` อัตโนมัติตอนส่งออกไปเป็น
  ส่วนหนึ่งของ schema, และ nullability ของ list กับ item ข้างในเป็นคนละเรื่องกัน
  (`[Type]` vs `[Type, null: true]`)
- Field ที่ root **`QueryType`** คือทางเข้าเดียวที่ client เรียกได้ — เพิ่ม `argument` เพื่อรับค่า
  เข้า resolver เป็น keyword argument, `null: true` ที่ field เหมาะกับกรณี "หาไม่เจอ" (คืน `null`
  แบบ graceful แทนที่จะ error)
- ยิง query จริงผ่าน `curl POST /graphql` ต้องเปิด `protect_from_forgery with: :null_session`
  เพราะ CSRF protection ปกป้อง `POST` ทุกตัวเป็นค่าเริ่มต้นในแอปแบบเต็มรูปแบบ — argument ที่ขาด
  หรือ field ที่ไม่มีจริงถูกจับตั้งแต่ **validation layer** (ไม่มี `data` key ในผลลัพธ์เลย)
  ต่างจากข้อมูล "หาไม่เจอ" จริง (`"data": {"post": null}`)
- **Nested field** คือ feature เด่นที่สุด — ขอ `Post` พร้อม `Comments` (และย้อนกลับได้ทั้งสองทาง)
  ในคำขอ HTTP เดียว โดยไม่ต้องออกแบบ endpoint พิเศษล่วงหน้าเหมือน REST
- **N+1 ใน GraphQL อันตรายกว่า REST** เพราะ client เป็นคนกำหนดความลึกของ nested field เอง เพิ่ม
  field เดียวก็สร้างจุด N+1 ใหม่ได้โดยไม่ต้องแก้โค้ด server — พิสูจน์ได้จริงด้วย
  `query_log_tags` ที่ generator ติดตั้งให้ (`current_graphql_field` ต่อท้ายทุกบรรทัด SQL log)
- แก้ N+1 ด้วย **`GraphQL::Dataloader::Source`** (built-in ของ graphql-ruby, แทนที่แนวทาง
  `graphql-batch` แบบเก่า): เขียน `fetch(objects)` ที่ยิง query เดียวแบบ `IN (...)` แล้ว
  `group_by`/`map` กลับเป็นลำดับเดิม — เรียก `.load(object)` ซ้ำกี่ field ก็ cache ผลลัพธ์เดียวกัน
  ให้อัตโนมัติ (พิสูจน์ด้วยแบบฝึกหัดที่แก้ `commentsCount` ให้ใช้ Source เดียวกับ `comments`)
- **Scalar** คือ type ค่าเดี่ยวไม่มี field ย่อย (`Int`/`Float`/`String`/`Boolean`/`ID` ตาม spec
  บวก `ISO8601DateTime` ที่ graphql-ruby เพิ่มให้) เขียน **Custom Scalar** เองได้ผ่าน
  `coerce_input`/`coerce_result` เมื่อต้องการ validate รูปแบบเฉพาะ ส่วน **Enum** ใช้จำกัดค่าที่
  เป็นไปได้ของ field ให้ schema บังคับแทนการเขียน validation ซ้ำเอง
- **Relay-style Connection** (`Type.connection_type`) ให้ pagination แบบ `edges`/`node`/
  `pageInfo`/`cursor` ที่ client GraphQL มาตรฐาน (Relay, Apollo) รองรับในตัวทันที โดยไม่ต้องเขียน
  logic เอง — cursor อ้างอิงตำแหน่งจริงของ record จึงไม่มีปัญหาข้อมูลเลื่อนตำแหน่งเหมือน
  `OFFSET` แบบ REST

**ต่อไป (Part 059):** ตอนนี้เรา query ข้อมูลผ่าน GraphQL ได้อย่างปลอดภัยจาก N+1 แล้ว Part ถัดไป
จะสอนฝั่งที่ยังขาดไป — **GraphQL Mutation** สำหรับสร้าง/แก้ไข/ลบข้อมูล (`Types::BaseMutation`,
`resolve` method, การจัดการ validation error ของ ActiveRecord ให้ออกมาเป็น GraphQL error ที่มี
โครงสร้างชัดเจน) และ **Authorization ใน GraphQL** — จะผูก Pundit (จาก Part 043) เข้ากับ field และ
mutation แต่ละตัวอย่างไร เมื่อ "endpoint" ไม่ได้แยกเป็นหลาย URL ให้ใส่ `before_action` แบบ REST
อีกต่อไป รวมถึงข้อควรระวังเรื่อง `context[:current_user]` ที่ต้องส่งผ่าน `GraphqlController` เข้า
ไปให้ resolver ทุกตัวเข้าถึงได้
