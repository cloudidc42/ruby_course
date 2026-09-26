# Part 059: GraphQL Mutation และ Authorization ใน GraphQL

> **Step ครอบคลุมใน Part นี้:** Step 581–590
> **ระดับ:** กลาง-สูง (ควรผ่าน Part 058 "GraphQL เบื้องต้นด้วย graphql-ruby: schema, type, query"
> และ Part 043 "Authorization เชิงลึกด้วย Pundit" มาก่อน)
> **เวอร์ชันที่ใช้:** Ruby 3.3.x, Rails 8.1.x, `graphql-ruby` ~> 2.4 (ทดสอบจริงกับ graphql 2.6.11),
> `pundit` ~> 2.5 (ทดสอบจริงกับ pundit 2.5.2), `jwt` ~> 2.9 (ทดสอบจริงกับ jwt 2.10.3)

Part 058 พาไปรู้จัก graphql-ruby เบื้องต้นแล้ว คือการนิยาม `Schema`, `Type` (เช่น
`Types::PostType`), และ root `Query` ที่ทำหน้าที่ **อ่าน** ข้อมูล — เรียกว่า "Query" เพราะ
ตามธรรมเนียมของ GraphQL แล้ว Query ควรเป็น **read-only operation เสมอ** ไม่มีผลข้างเคียง
(side effect) ต่อฐานข้อมูลเลย

Part นี้พาไปสู่อีกครึ่งหนึ่งที่ขาดไม่ได้ของ GraphQL API ที่ใช้งานได้จริง คือการ **เขียนข้อมูล**
(สร้าง/แก้ไข/ลบ) ผ่าน **Mutation** และการปกป้องข้อมูลนั้นด้วย **Authorization** — เราจะไม่พูดซ้ำ
เรื่องพื้นฐานของ schema/type/query จาก Part 058 อีก แต่จะต่อยอดจากโครงสร้างเดิมโดยตรง พร้อม
เชื่อมความรู้เรื่อง Pundit จาก Part 043 เข้ากับ GraphQL resolver

ตัวอย่างทั้งหมดใน Part นี้ทดสอบจริงกับ scratch Rails app (`graphql-ruby` 2.6.11, `pundit` 2.5.2
บน Ruby 3.3.6 / Rails 8.1.4) ทุกคำสั่ง `curl`, ทุก query/mutation ที่แสดงผลลัพธ์ คือค่าที่รันได้
จริงและตรวจสอบแล้ว ไม่ใช่ผลลัพธ์ที่คาดเดา

## สารบัญของ Part นี้

- Step 581: GraphQL Mutation คืออะไร — ทำไม Query กับ Mutation ต้องแยกกันตามธรรมเนียม
- Step 582: สร้าง Mutation แรก — `Mutations::CreatePost`, `argument`, `field`, `resolve`
- Step 583: ธรรมเนียมของ Errors ใน Mutation — ทำไม GraphQL ไม่ใช้ HTTP status code
- Step 584: เชื่อม Mutation เข้ากับ root `Mutation` type และเรียกผ่าน curl/GraphiQL
- Step 585: `UpdatePost` และ `DeletePost` — ทำตามแบบแผนเดียวกัน
- Step 586: Authentication ของ GraphQL request — JWT, `context`, และ `current_user`
- Step 587: Authorization ด้วย Pundit ภายใน Mutation/Resolver — error ที่ไม่ใช่ HTTP 403
- Step 588: Object-level Authorization — ซ่อน field ทั้งหมด vs raise error
- Step 589: Authorization Hook ของ graphql-ruby เอง — `self.authorized?` บน Type
- Step 590: Testing GraphQL Query/Mutation ด้วย RSpec (ต่อยอดจาก Part 019/046)

---

## Step 581: GraphQL Mutation คืออะไร — ทำไม Query กับ Mutation ต้องแยกกันตามธรรมเนียม

ใน Part 058 เรานิยาม root type สองตัวไว้ในไฟล์ schema:

```ruby
# app/graphql/blog_schema.rb (ทบทวนจาก Part 058)
class BlogSchema < GraphQL::Schema
  query(Types::QueryType)
  mutation(Types::MutationType)   # <- ตัวที่ Part นี้จะเจาะลึก
end
```

`query(Types::QueryType)` คือจุดเริ่มต้นของทุกการ **อ่าน** ข้อมูล ส่วน
`mutation(Types::MutationType)` คือจุดเริ่มต้นของทุกการ **เขียน** ข้อมูล — ทั้งสองเป็นเพียง
"ประตูทางเข้า" (root type) ที่ต่างกันแค่ **ชื่อ** ในระดับ GraphQL specification เท่านั้น ไม่มี
อะไรที่ระบบบังคับทางเทคนิคว่า field ใต้ `Query` ห้ามเขียนข้อมูล หรือ field ใต้ `Mutation` ห้าม
อ่านข้อมูลเลย — **นี่คือ "ธรรมเนียม" (convention) ที่ชุมชน GraphQL ทั้งหมดยึดถือร่วมกัน** ไม่ใช่
ข้อบังคับของภาษา

### ทำไมธรรมเนียมนี้ถึงสำคัญมาก

1. **คาดเดาพฤติกรรมได้จากชื่อ field** — client ที่เห็น field อยู่ใต้ `Query` รู้ทันทีว่าเรียกกี่ครั้ง
   ก็ได้อย่างปลอดภัย (idempotent ในแง่ไม่มีผลข้างเคียง) แต่ field ใต้ `Mutation` ต้องระวังว่า
   เรียกซ้ำอาจสร้างข้อมูลซ้ำ (เช่น เรียก `createPost` สองครั้งจะได้โพสต์สองอัน)
2. **GraphQL spec รับประกันลำดับการทำงานต่างกัน** — เมื่อ client ส่งหลาย field มาพร้อมกันใน
   operation เดียว (`mutation { a: createPost(...) { ... } b: createPost(...) { ... } }`)
   สเปกกำหนดว่า field ระดับบนสุดของ **Mutation ต้องรันเรียงลำดับทีละตัว (sequentially)**
   เพื่อป้องกัน race condition ระหว่างการเขียนข้อมูล ในขณะที่ field ของ **Query รันพร้อมกันได้
   (concurrently)** เพราะไม่มีผลข้างเคียงให้ชนกัน — graphql-ruby ปฏิบัติตามกฎนี้ให้อัตโนมัติ
3. **เครื่องมือ (tooling) พึ่งพาธรรมเนียมนี้** — GraphiQL, Apollo Client, Relay ล้วนแสดงผล
   Mutation ต่างจาก Query ใน UI (เช่น เตือนก่อนรันซ้ำ) เพราะสมมติไว้ว่า schema ทุกตัวเคารพ
   กติกานี้

### เปรียบเทียบกับ REST ที่เรียนมาใน Part 056–057

| REST | GraphQL | ความหมาย |
|---|---|---|
| `GET /posts` | `query { posts { ... } }` | อ่านข้อมูล ไม่มีผลข้างเคียง |
| `POST /posts` | `mutation { createPost(...) { ... } }` | สร้างข้อมูลใหม่ |
| `PATCH /posts/:id` | `mutation { updatePost(...) { ... } }` | แก้ไขข้อมูลที่มีอยู่ |
| `DELETE /posts/:id` | `mutation { deletePost(...) { ... } }` | ลบข้อมูล |

สังเกตว่า REST ใช้ **HTTP verb** (`GET`/`POST`/`PATCH`/`DELETE`) เป็นตัวบอกเจตนา ส่วน GraphQL
ใช้ **ชื่อ field** เป็นตัวบอกเจตนาแทน เพราะ HTTP request ของ GraphQL แทบทั้งหมดยิงผ่าน
`POST /graphql` เพียง endpoint เดียว (ตามที่ตั้งค่าไว้ใน `config/routes.rb` จาก Part 058)
ไม่ว่าจะเป็น query หรือ mutation ก็ตาม — ข้อมูลที่ระบุว่ากำลังทำอะไรอยู่ **ทั้งหมดอยู่ใน body
ของ request** ไม่ใช่ที่ HTTP verb หรือ URL

---

## Step 582: สร้าง Mutation แรก — `Mutations::CreatePost`, `argument`, `field`, `resolve`

### โครงสร้างพื้นฐานที่ generator ของ graphql-ruby สร้างให้

ตอนรัน `bin/rails generate graphql:install` (ใน Part 058) generator สร้างไฟล์ base mutation
ให้พร้อมกับ base type ทั้งหมด:

```ruby
# app/graphql/mutations/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject
  end
end
```

**สิ่งสำคัญที่ต้องรู้ทันที:** `BaseMutation` สืบทอดจาก `GraphQL::Schema::RelayClassicMutation`
ไม่ใช่ `GraphQL::Schema::Mutation` เฉยๆ — คำว่า "Relay Classic" หมายถึง graphql-ruby จะ
**ห่อ argument ทั้งหมดของเราไว้ใน object เดียวชื่อ `input`โดยอัตโนมัติ** และ **ห่อ field
ทั้งหมดที่เรา return ไว้ใน payload object โดยอัตโนมัติเช่นกัน** (พร้อมเพิ่ม field
`clientMutationId` ให้ฟรีสำหรับ client ที่ต้องการ track การเรียกแต่ละครั้ง) — นี่คือรูปแบบที่
เป็นมาตรฐานเกือบทุก GraphQL API ในโลกจริงใช้ (Shopify, GitHub GraphQL API ก็ใช้แบบแผนนี้)

### เขียน `Mutations::CreatePost`

สมมติเรามีโมเดล `Post` (`belongs_to :user`, มี `title`, `body`, `published`) เหมือนที่ใช้ใน
Part 043 ทุกประการ สร้างไฟล์ mutation ตัวแรก:

```ruby
# app/graphql/mutations/create_post.rb
module Mutations
  class CreatePost < BaseMutation
    # argument คือ input ที่ client ต้องส่งมา — เทียบเท่า strong parameters ฝั่ง REST
    # controller (`params.require(:post).permit(:title, :body, :published)` ที่เรียนใน
    # Part 031) แต่ graphql-ruby ตรวจสอบ "ชนิดข้อมูล" ให้อัตโนมัติตั้งแต่ก่อนโค้ดเราจะรันด้วยซ้ำ
    argument :title, String, required: true
    argument :body, String, required: false
    argument :published, Boolean, required: false, default_value: false

    # field คือรูปร่างของ "ผลลัพธ์" ที่ mutation นี้จะคืนกลับไปให้ client
    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    # resolve คือ method หลักที่มี logic การทำงานจริง — ชื่อ argument ต้องตรงกับที่ประกาศ
    # ไว้ด้านบนทุกตัวคำต่อคำ (keyword arguments)
    def resolve(title:, body: nil, published: false)
      post = Post.new(title: title, body: body, published: published)

      if post.save
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

**อธิบายทีละส่วน:**

- `argument :title, String, required: true` — ประกาศว่า mutation นี้ต้องการ input ชื่อ
  `title` เป็น GraphQL scalar type `String` และ **ต้องส่งมาเสมอ** (`required: true`) ถ้า
  client ไม่ส่ง `title` มา graphql-ruby จะปฏิเสธ request **ตั้งแต่ก่อนที่ `resolve` จะถูกเรียก
  เลยด้วยซ้ำ** พร้อมข้อความ error ที่ชัดเจนว่าขาด argument ไหน (เดี๋ยว Step 584 จะสาธิตให้ดู)
- `argument :body, String, required: false` และ `argument :published, ...,
  default_value: false` — argument ที่ไม่ required จะมีค่าเป็น `nil` ถ้าไม่ส่งมา (เว้นแต่ตั้ง
  `default_value` ไว้แบบ `published` ที่จะได้ `false` แทน `nil` เสมอ)
- `field :post, Types::PostType, null: true` — ผลลัพธ์ mutation นี้จะมี field ชื่อ `post`
  เป็น `PostType` เดียวกับที่ `QueryType` ใช้ใน Part 058 (นี่คือจุดที่ **object ที่เขียนเสร็จ
  แล้ว ถูกส่งกลับไปให้ client อ่านค่าล่าสุดได้ทันทีในผลลัพธ์เดียวกับที่สั่งสร้าง** ไม่ต้องยิง
  query แยกอีกรอบเหมือน REST ที่บางครั้งต้อง `GET` กลับไปดูผลหลัง `POST`)
- `field :errors, [String], null: false` — จุดสำคัญที่สุดของ Step นี้ จะอธิบายเจาะลึกใน
  Step 583
- `def resolve(title:, body: nil, published: false)` — signature ของ `resolve` รับ
  keyword argument ตรงตามชื่อที่ประกาศด้วย `argument` ทุกตัว โดย graphql-ruby แปลง
  camelCase ที่ client ส่งมา (เช่น `publishedAt`) ให้เป็น snake_case ให้อัตโนมัติแล้วก่อนถึง
  method นี้ (ทบทวนกฎการแปลงชื่อ camelCase ↔ snake_case ได้จาก Part 058)
- ค่าที่ `resolve` return ต้องเป็น **Hash ที่มี key ตรงกับชื่อ field ที่ประกาศไว้ทุกตัว**
  (`{ post: ..., errors: ... }`) — graphql-ruby จะ map ค่าจาก Hash นี้ไปยัง field ที่ตรงกัน
  ให้อัตโนมัติ

> **หมายเหตุ:** เพราะ `resolve` เป็น instance method ธรรมดาของ Ruby class เราเรียกทดลอง
> logic ภายในตรงๆ ได้แม้ยังไม่เชื่อมเข้า schema เลย (`Mutations::CreatePost.new(object: nil,
> field: nil, context: {}).resolve(title: "...")`) มีประโยชน์ตอน debug อย่างรวดเร็ว แต่การ
> ทดสอบแบบเป็นทางการควรยิงผ่าน `Schema.execute` เสมอ (Step 590) เพราะการเรียกตรงๆ แบบนี้
> ข้าม type checking ของ `argument` ไปทั้งหมด

---

## Step 583: ธรรมเนียมของ Errors ใน Mutation — ทำไม GraphQL ไม่ใช้ HTTP Status Code

นี่คือความแตกต่างที่สำคัญที่สุดอย่างหนึ่งระหว่าง REST API (ที่เรียนใน Part 056–057) กับ
GraphQL API และเป็นจุดที่ทำให้มือใหม่งงบ่อยที่สุด

### ทบทวนวิธีคิดแบบ REST ก่อน

ใน REST controller ทั่วไปที่เราเขียนมาตลอด (Part 031 เป็นต้นไป) เมื่อ validation ล้มเหลว เรา
ตอบกลับด้วย **HTTP status code 422 (Unprocessable Entity)**:

```ruby
# REST controller แบบที่คุ้นเคย — ใช้ HTTP status code สื่อความหมาย
def create
  @post = Post.new(post_params)
  if @post.save
    render json: @post, status: :created          # 201
  else
    render json: { errors: @post.errors.full_messages }, status: :unprocessable_entity  # 422
  end
end
```

Client ตรวจสอบ `response.status` ก่อนเป็นอันดับแรกเพื่อรู้ว่าสำเร็จหรือล้มเหลว

### แต่ GraphQL **ไม่ทำแบบนั้น** — และนี่คือเหตุผล

**GraphQL response แทบทุกกรณีตอบกลับด้วย HTTP status `200 OK` เสมอ** ไม่ว่าการ create/update
จะสำเร็จหรือ validation จะล้มเหลวก็ตาม (`200` ถูกใช้ตราบใดที่ **request เข้าใจได้** คือ syntax
ถูกต้องและ execute ได้ — HTTP-level error เช่น `400`/`500` สงวนไว้สำหรับกรณีร้ายแรงกว่านั้น
เช่น query syntax ผิด หรือ server crash เท่านั้น)

เหตุผลเชิงสถาปัตยกรรมคือ **1 HTTP request ของ GraphQL อาจมีหลาย "operation" ปนกันอยู่ในนั้น**
(field หลายตัวใน mutation เดียว หรือแม้แต่ query กับ mutation หลายตัวถ้า schema อนุญาต) ถ้า
field หนึ่งสำเร็จแต่อีก field ล้มเหลว **HTTP status code เดียวไม่พอจะสื่อความหมายที่ถูกต้องของ
ทั้ง response ได้** — จึงต้องมีกลไกที่ **ระดับ field/payload เอง** สื่อว่าอะไรสำเร็จ อะไรล้มเหลว

### แบบแผนที่ชุมชน GraphQL ยึดถือ: `errors` field ใน payload

แทนที่จะพึ่ง HTTP status code เรา **ออกแบบ payload ของ mutation ให้มี field `errors` (หรือ
`userErrors` ในบาง API เช่น Shopify GraphQL API) เป็นส่วนหนึ่งของผลลัพธ์ปกติ** ตามที่เขียนไว้
แล้วใน Step 582:

```ruby
field :post, Types::PostType, null: true
field :errors, [String], null: false
```

เมื่อ validation ล้มเหลว `resolve` **ไม่ raise exception และไม่ทำให้ HTTP request ล้มเหลว**
เพียงแค่ return `post: nil` พร้อม `errors: [...]` เท่านั้น — client ตรวจสอบว่าสำเร็จหรือไม่
จาก **ค่าของ field `errors` ใน response body** ไม่ใช่จาก HTTP status:

```graphql
mutation($title: String!) {
  createPost(input: { title: $title }) {
    post { id title }
    errors
  }
}
```

ถ้าส่ง `title` เป็น string ว่าง (`""`) ผลลัพธ์จริงที่ทดสอบได้คือ:

```json
{
  "data": {
    "createPost": {
      "post": null,
      "errors": ["Title can't be blank"]
    }
  }
}
```

**สังเกตสิ่งสำคัญ:** ทั้ง `"data"` key มีอยู่ปกติ และไม่มี key `"errors"` ที่ระดับบนสุดของ
response เลย (top-level `errors` สงวนไว้สำหรับข้อผิดพลาดระดับ GraphQL execution เอง เช่น
argument ผิดชนิด หรือ authorization error ที่จะเห็นใน Step 587) — นี่คือความแตกต่างสำคัญที่
ต้องแยกให้ออก: `errors` **ระดับบนสุดของ response** (`{"errors": [...], "data": ...}`) กับ
`errors` **ที่เป็นแค่ field หนึ่งใน payload ของเรา** (`data.createPost.errors`) เป็นคนละเรื่อง
กันโดยสิ้นเชิง แม้จะใช้ชื่อ `errors` เหมือนกันก็ตาม

### ทำไมถึงออกแบบแบบนี้ (ข้อดีที่จับต้องได้)

1. **HTTP status สื่อความหมายที่ HTTP ควรสื่อจริงๆ เท่านั้น** — "request ไปถึง server และ
   ประมวลผลได้หรือไม่" ไม่ปนกับ "ข้อมูลถูกต้องตาม business rule หรือไม่" (คนละ layer กัน)
2. **client เขียนโค้ดจัดการ error ได้ตรงไปตรงมา** — เช็ค
   `if (result.data.createPost.errors.length > 0)` แทนที่จะต้อง try/catch HTTP exception
3. **รองรับ partial success ได้เป็นธรรมชาติ** — mutation ที่มีหลาย field ในตัวเดียว แต่ละส่วน
   มี `errors` ของตัวเองแยกกันได้ โดย HTTP request ทั้งก้อนยังคง "สำเร็จ" ในความหมายของ HTTP
4. **Schema เป็นเอกสารในตัวเอง** — client เห็นจากชนิดข้อมูลของ field `errors` ตรงๆ ว่า
   mutation ไหนมีโอกาสล้มเหลวแบบไหนบ้าง ไม่ต้องอ่านเอกสารแยกต่างหาก

> **ข้อควรระวัง:** field `errors: [String]` แบบง่ายๆ ที่ใช้ใน Part นี้เหมาะกับตัวอย่างเพื่อ
> ความเข้าใจ ระบบระดับ production จริงมักออกแบบ `errors` ให้เป็น structured type
> (`type UserError { field: String, message: String, code: String }`) แทนที่จะเป็น
> string ธรรมดา เพื่อให้ client แยกได้ว่า error เกิดจาก field ไหน — แนวทางนี้เรียกว่า
> **"userErrors pattern"** เป็นชื่อที่ Shopify GraphQL API ใช้เรียกแบบแผนนี้ ในแบบฝึกหัดท้าย
> Part จะให้ลองแปลง `errors: [String]` เป็นโครงสร้างแบบนี้ด้วยตัวเอง

---

## Step 584: เชื่อม Mutation เข้ากับ Root `Mutation` Type และเรียกผ่าน curl/GraphiQL

### ลงทะเบียน mutation ใน `Types::MutationType`

Mutation class ที่เขียนไว้ใน Step 582 ยังไม่ทำงานจนกว่าจะ "เสียบ" เข้ากับ root
`Types::MutationType` (คู่กับ `Types::QueryType` ที่ Part 058 ผูกกับ `field` แบบธรรมดา แต่
mutation ผูกด้วย keyword `mutation:` แทน):

```ruby
# app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    field :create_post, mutation: Mutations::CreatePost
  end
end
```

**สังเกตความต่างจากการประกาศ field ปกติ:** ปกติเราต้องเขียน `field :name, Type, null: ...`
พร้อม method แยกต่างหาก แต่พอใช้ `mutation: Mutations::CreatePost` graphql-ruby จะ:

1. อ่าน `argument` ทั้งหมดที่ประกาศไว้ใน `CreatePost` มาสร้าง input type ชื่อ
   `CreatePostInput` ให้อัตโนมัติ
2. อ่าน `field` ทั้งหมดมาสร้าง payload type ชื่อ `CreatePostPayload` ให้อัตโนมัติ
3. แปลงชื่อ class `CreatePost` (PascalCase) เป็นชื่อ field `createPost` (camelCase)
   ให้อัตโนมัติ (กฎการแปลงชื่อเดียวกับที่เรียนใน Part 058)

ตรวจสอบ SDL (Schema Definition Language) ที่ graphql-ruby สร้างให้จริง ด้วยคำสั่ง:

```ruby
# bin/rails console
puts BlogSchema.to_definition
```

ส่วนที่เกี่ยวกับ `createPost` ที่ได้จริงจากการรัน:

```graphql
input CreatePostInput {
  body: String
  clientMutationId: String
  published: Boolean = false
  title: String!
}

type CreatePostPayload {
  clientMutationId: String
  errors: [String!]!
  post: Post
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload
}
```

**อธิบาย:** สังเกตว่า `title: String!` (มี `!` ต่อท้าย หมายถึง non-null ใน GraphQL SDL) ตรงกับ
`required: true` ที่เราประกาศไว้ทุกประการ และ `clientMutationId` ปรากฏขึ้นมาเองทั้งใน input
และ payload — เป็นของแถมจาก `RelayClassicMutation` ตามที่อธิบายไว้ใน Step 582 (client ส่งค่า
อะไรมาก็ได้ผ่าน field นี้ แล้วจะได้ค่าเดิมกลับมาในผลลัพธ์ ใช้ track ว่า response ไหนตรงกับ
mutation call ไหนเวลายิงหลายตัวพร้อมกัน — ในตัวอย่างของ Part นี้จะไม่ใช้ field นี้เพื่อความ
กระชับ)

### เรียก mutation ผ่าน GraphiQL และ `curl`

Part 058 ติดตั้ง `graphiql-rails` ไว้แล้วที่ `/graphiql` — เปิด `bin/rails server` แล้วเข้า
`http://localhost:3000/graphiql` พิมพ์ query panel และ variables panel (JSON) แยกกันได้ใน
หน้าต่างเดียวกัน เหมาะสำหรับทดลองระหว่างพัฒนา แต่ client จริง (mobile app, frontend SPA,
service อื่น) ยิง HTTP request ตรงๆ ผ่าน `POST /graphql` ด้วย body เป็น JSON ที่มี key
`query` และ `variables` แทน — ทั้งสองวิธีเรียก schema ตัวเดียวกัน ผลลัพธ์จึงเหมือนกันทุก
ประการ ต่อไปนี้จะสาธิตด้วย `curl` เพราะทดสอบซ้ำและอ่านผลลัพธ์แบบ script ได้ง่ายกว่า:

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation($title: String!, $body: String) { createPost(input: { title: $title, body: $body }) { post { id title published } errors } }",
    "variables": { "title": "เรียนรู้ GraphQL Mutation", "body": "เนื้อหาโพสต์" }
  }'
```

ผลลัพธ์จริงจากการรัน `curl` นี้กับ scratch app (ตรวจสอบพร้อม HTTP status code):

```json
{"data":{"createPost":{"post":{"id":"1","title":"เรียนรู้ GraphQL Mutation","published":false},"errors":[]}}}
```

HTTP status ของ response นี้คือ `200` — ตรงตามที่อธิบายไว้ใน Step 583 (สำเร็จหรือไม่ ดูจาก
`errors` field ใน payload ไม่ใช่จาก status code)

### ทดสอบกรณีขาด argument ที่ required

ลองส่ง request โดยไม่ส่ง `title` มาเลย (ละ `title` ออกจาก `variables` และเปลี่ยน query ให้ไม่
ส่ง `$title`):

```bash
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { createPost(input: {}) { post { id } errors } }"}'
```

ผลลัพธ์จริง:

```json
{
  "errors": [
    {
      "message": "Argument 'title' on InputObject 'CreatePostInput' is required. Expected type String!"
    }
  ]
}
```

สังเกตว่าครั้งนี้ error ปรากฏที่ **ระดับบนสุดของ response** (ไม่ใช่ใน `data.createPost.errors`)
เพราะเป็น **structural error ระดับ GraphQL execution เอง** (request ไม่ตรงกับ schema เลย
ไม่ใช่ business validation ที่เราเขียนเอง) — graphql-ruby ปฏิเสธ request ตั้งแต่ก่อน `resolve`
จะถูกเรียกด้วยซ้ำ ยืนยันสิ่งที่อธิบายไว้ใน Step 582 ว่า argument ที่ผิดชนิด/ขาดหายไปถูกดักที่
framework layer โดยอัตโนมัติ

---

## Step 585: `UpdatePost` และ `DeletePost` — ทำตามแบบแผนเดียวกัน

เมื่อเข้าใจแบบแผนของ `CreatePost` แล้ว การเขียน mutation อื่นๆ ก็เป็นการทำซ้ำ pattern เดียวกัน

### `Mutations::UpdatePost`

```ruby
# app/graphql/mutations/update_post.rb
module Mutations
  class UpdatePost < BaseMutation
    argument :id, ID, required: true
    argument :title, String, required: false
    argument :body, String, required: false
    argument :published, Boolean, required: false

    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    def resolve(id:, **attrs)
      post = Post.find(id)

      if post.update(attrs)
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

**สิ่งที่ต่างจาก `CreatePost`:**

- `argument :id, ID, required: true` — GraphQL มี scalar type ชื่อ `ID` แยกจาก `String`
  โดยเฉพาะ (แม้ค่าจริงจะเป็น string เสมอเวลาส่งผ่าน JSON) เพื่อสื่อความหมายให้ client เห็น
  ชัดเจนว่า argument ตัวนี้คือ identifier ไม่ใช่ข้อความทั่วไป
- `**attrs` รวบรวม keyword argument ที่เหลือ (`title`, `body`, `published`) เป็น Hash เดียว
  แล้วส่งเข้า `post.update` ตรงๆ — ทำงานเหมือน `post_params` ที่มาจาก strong parameters ใน
  REST controller ทุกประการ ต่างกันแค่ที่มาของการกรอง argument (GraphQL กรองด้วยการประกาศ
  `argument` แทนที่จะกรองด้วย `.permit`)
- ไม่ต้อง handle กรณี `Post.find(id)` หา record ไม่เจอเป็นพิเศษที่นี่ เพราะ
  `ActiveRecord::RecordNotFound` ที่ raise ออกมาจะกลายเป็น GraphQL execution error โดย
  อัตโนมัติ (ปรากฏใน `errors` ระดับบนสุดของ response เหมือนกรณี argument ผิดใน Step 584)

### `Mutations::DeletePost`

```ruby
# app/graphql/mutations/delete_post.rb
module Mutations
  class DeletePost < BaseMutation
    argument :id, ID, required: true

    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    def resolve(id:)
      post = Post.find(id)
      post.destroy

      { post: post, errors: [] }
    end
  end
end
```

**สังเกต:** แม้จะ "ลบ" record ไปแล้ว เรายังคง return `post: post` ได้ตามปกติ เพราะ object
Ruby ใน memory ยังคงมี attribute เดิมอยู่ครบ (`destroy` แค่ลบแถวออกจากฐานข้อมูล ไม่ได้ล้างค่า
attribute ของ object ที่เรียกมัน) — client ที่เรียก `deletePost` จึงยังเห็น `id`/`title` ของ
โพสต์ที่เพิ่งถูกลบไปในผลลัพธ์ได้ ซึ่งมีประโยชน์เวลาต้องการแสดงข้อความยืนยัน (เช่น "ลบโพสต์
'เรียนรู้ GraphQL Mutation' แล้ว")

### ลงทะเบียนทั้งสองตัวเข้า `MutationType`

```ruby
# app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    field :create_post, mutation: Mutations::CreatePost
    field :update_post, mutation: Mutations::UpdatePost
    field :delete_post, mutation: Mutations::DeletePost
  end
end
```

### ทดสอบ `updatePost` และ `deletePost` จริงผ่าน console

```ruby
# bin/rails console
post = Post.first

update_mutation = <<~GQL
  mutation($id: ID!, $title: String) {
    updatePost(input: { id: $id, title: $title }) {
      post { id title }
      errors
    }
  }
GQL

result = BlogSchema.execute(update_mutation, variables: { id: post.id.to_s, title: "แก้ไขแล้ว" })
result.to_h
# => {"data"=>{"updatePost"=>{"post"=>{"id"=>"1", "title"=>"แก้ไขแล้ว"}, "errors"=>[]}}}

delete_mutation = <<~GQL
  mutation($id: ID!) {
    deletePost(input: { id: $id }) {
      post { id title }
      errors
    }
  }
GQL

result = BlogSchema.execute(delete_mutation, variables: { id: post.id.to_s })
result.to_h
# => {"data"=>{"deletePost"=>{"post"=>{"id"=>"1", "title"=>"แก้ไขแล้ว"}, "errors"=>[]}}}

Post.exists?(post.id)  # => false
```

ผลลัพธ์ทั้งหมดข้างต้นคือค่าที่ตรวจสอบจริงจากการรันในสภาพแวดล้อมทดสอบ — สังเกตว่าเรียก
`Schema.execute(query_string, variables: {...})` ตรงๆ ได้โดยไม่ต้องผ่าน HTTP controller เลย
วิธีนี้เป็นหัวใจสำคัญของการทดสอบ GraphQL ด้วย RSpec ที่จะเจาะลึกใน Step 590

> **สังเกตสิ่งที่ยังขาดอยู่:** ตอนนี้ **ใครก็เรียก `updatePost`/`deletePost` กับโพสต์ของใคร
> ก็ได้ทั้งนั้น** ไม่มีการตรวจสอบเลยว่า "ใครเป็นคนเรียก" หรือ "คนเรียกมีสิทธิ์แก้ไข/ลบโพสต์นี้
> หรือไม่" — Step 586–588 จะแก้ปัญหานี้ด้วย authentication และ Pundit authorization

---

## Step 586: Authentication ของ GraphQL Request — JWT, `context`, และ `current_user`

### ปัญหา: GraphQL controller มีแค่ 1 action

Part 045 สอนเรื่อง JWT authentication สำหรับ REST API ที่มีหลาย controller/action แต่ละตัว
เรียก `authenticate_request!` ผ่าน `before_action` ได้อิสระ — แต่ GraphQL ทั้งระบบมักมีแค่
**1 controller, 1 action** (`GraphqlController#execute` ที่ generator สร้างให้ตั้งแต่ Part 058)
เพราะทุก query/mutation ยิงผ่าน endpoint เดียวกันหมด (`POST /graphql`)

คำถามคือ: ถ้ามีแค่ action เดียว แล้ว `current_user` จะไปถึง `resolve` ของ mutation แต่ละตัว
(ที่เป็น class คนละไฟล์กับ controller) ได้อย่างไร — คำตอบคือผ่านสิ่งที่เรียกว่า **`context`**

### `context` คือ Hash ที่เดินทางไปกับทุก field ตลอดการ execute

`BlogSchema.execute` รับ keyword argument ชื่อ `context:` เป็น Hash ธรรมดา — Hash นี้จะถูกส่งต่อเข้าไปให้ **ทุก type, ทุก field,
ทุก mutation** ที่ execute ในรอบนั้นๆ เข้าถึงได้ผ่าน method `context` ที่มีอยู่ในทุก class ที่
สืบทอดจาก `Types::BaseObject` หรือ `Mutations::BaseMutation`

### สร้าง JWT helper (ต่อยอดแนวคิดจาก Part 045)

```ruby
# app/lib/json_web_token.rb
class JsonWebToken
  SECRET_KEY = Rails.application.secret_key_base

  def self.encode(payload)
    JWT.encode(payload, SECRET_KEY, "HS256")
  end

  def self.decode(token)
    body = JWT.decode(token, SECRET_KEY, true, algorithm: "HS256").first
    ActiveSupport::HashWithIndifferentAccess.new(body)
  rescue JWT::DecodeError
    nil
  end
end
```

โค้ดนี้เหมือนกับ helper ที่เรียนใน Part 045 ทุกประการ (encode/decode payload ด้วย HMAC และ
`secret_key_base` ของแอป) — สิ่งที่ต่างคือ**จุดที่เราเรียกใช้มัน**

### แก้ `GraphqlController` ให้ resolve `current_user` แล้วยัดใส่ `context`

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  skip_before_action :verify_authenticity_token, raise: false

  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    context = {
      current_user: current_user_from_token,
    }
    result = BlogSchema.execute(query, variables: variables, context: context, operation_name: operation_name)
    render json: result
  rescue StandardError => e
    raise e unless Rails.env.development?
    handle_error_in_development(e)
  end

  private

  # ดึง JWT จาก header "Authorization: Bearer <token>" แล้วหา User ที่ตรงกับ payload
  # คืนค่า nil ถ้าไม่มี header, token ผิดรูปแบบ, หรือหา user ไม่เจอ (ถือเป็น guest)
  def current_user_from_token
    header = request.headers["Authorization"]
    return nil if header.blank?

    token = header.split(" ").last
    payload = JsonWebToken.decode(token)
    return nil unless payload

    User.find_by(id: payload[:user_id])
  end

  # ... prepare_variables, handle_error_in_development เหมือนเดิมจาก Part 058 ...
end
```

**อธิบาย:**

- `current_user_from_token` **ไม่ raise error เมื่อไม่มี token** (ต่างจาก REST API ทั่วไปที่
  มักตอบ `401 Unauthorized` ทันทีถ้าไม่มี token) เพราะ GraphQL schema ของเราออกแบบให้รองรับ
  **ทั้ง guest และ user ที่ login แล้วในระบบเดียวกัน** (field บางตัว เช่น `posts` ที่เห็นเฉพาะ
  โพสต์ published ควรใช้งานได้แม้เป็น guest) — การตัดสินใจว่า "ต้อง login หรือไม่" ถูกผลักไป
  ให้ **แต่ละ mutation/field ตัดสินใจเอง** ผ่าน Pundit policy (Step 587) แทนที่จะเป็นการเช็ค
  รวมศูนย์ที่ controller เหมือน REST — เป็นข้อแตกต่างเชิงสถาปัตยกรรมที่สำคัญมาก
- `context[:current_user]` เป็น `nil` สำหรับ guest, หรือเป็น `User` instance จริงสำหรับ
  ผู้ใช้ที่ login แล้ว — ไม่ต่างจาก `current_user` ที่คุ้นเคยจาก Devise/session-based auth
  (Part 041–042) เพียงแต่มาจาก JWT header แทน session cookie

### ทดสอบ authentication จริงด้วย `curl`

สร้าง user แล้ว encode JWT:

```ruby
# bin/rails console
alice = User.create!(email: "alice@example.com", password: "password123", role: :member)
token = JsonWebToken.encode(user_id: alice.id)
puts token
# => eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.4RzAA-lIk5ohn8TKfm63BejeiGuGs16SkAjVbxCq59o
```

ยิง request พร้อม header `Authorization: Bearer <token>`:

```bash
TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.4RzAA-lIk5ohn8TKfm63BejeiGuGs16SkAjVbxCq59o"

curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "query": "mutation($title: String!, $body: String) { createPost(input: { title: $title, body: $body }) { post { id title } errors } }",
    "variables": { "title": "เรียนรู้ GraphQL Mutation", "body": "เนื้อหาโพสต์" }
  }'
```

ผลลัพธ์จริงที่ทดสอบได้ (HTTP status `200`):

```json
{"data":{"createPost":{"post":{"id":"1","title":"เรียนรู้ GraphQL Mutation"},"errors":[]}}}
```

ตอนนี้เราส่ง `current_user` ผ่าน `context` เข้าไปในทุก mutation แล้ว แต่ **ยังไม่มี mutation
ตัวไหนใช้ค่านี้เพื่อตัดสินใจอะไรเลย** — Step 587 จะเชื่อม `context[:current_user]` เข้ากับ
Pundit policy ที่เรียนไปแล้วใน Part 043

---

## Step 587: Authorization ด้วย Pundit ภายใน Mutation/Resolver — Error ที่ไม่ใช่ HTTP 403

### ทบทวนสั้นๆ จาก Part 043

`PostPolicy` ที่เขียนไว้ใน Part 043 เป็น Ruby class ธรรมดา ไม่ผูกกับ Rails controller เลย —
`PostPolicy.new(user, post).update?` เรียกได้จากที่ไหนก็ได้ในแอป รวมถึงจาก GraphQL mutation
ด้วย ไม่ต้องแก้ policy แม้แต่บรรทัดเดียว:

```ruby
# app/policies/post_policy.rb (จาก Part 043 ทุกประการ ไม่มีการแก้ไข)
class PostPolicy < ApplicationPolicy
  def show?
    record.published? || owner? || admin?
  end

  def create?
    user.present?
  end

  def update?
    owner? || admin?
  end

  def destroy?
    owner? || admin?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user&.admin?
        scope.all
      elsif user
        scope.where(published: true).or(scope.where(user_id: user.id))
      else
        scope.where(published: true)
      end
    end
  end

  private

  def owner?
    user.present? && record.user_id == user.id
  end

  def admin?
    user&.admin?
  end
end
```

### ปัญหา: `authorize`/`policy_scope` ของ Pundit ผูกกับ Controller

`Pundit::Authorization` module ที่ Part 043 `include` เข้า `ApplicationController` ให้
method `authorize`/`policy_scope`/`policy` — แต่ method เหล่านี้เขียนมาเพื่อเรียกจาก
**controller context** (มันเรียก `current_user` ของ controller เองภายใน) ในขณะที่
`Mutations::CreatePost#resolve` ไม่ใช่ controller เลย

**ทางแก้:** Pundit มี module method แบบ standalone อยู่แล้ว คือ `Pundit.authorize` และ
`Pundit.policy_scope!` ที่ **รับ `user` เป็น argument ตรงๆ** โดยไม่ต้องพึ่ง controller
context ใดๆ เลย — เหมาะกับการใช้ใน background job, Rake task, หรือ GraphQL resolver ทุกกรณี
ที่ "ไม่มี controller" อยู่ตรงหน้า

### เพิ่ม helper `pundit_authorize!` ใน `BaseMutation`

```ruby
# app/graphql/mutations/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject

    private

    # Mutation ทุกตัวเรียก pundit_authorize!(record, :some_action?) แทนการเรียก
    # Pundit.authorize ตรงๆ — รวมจุดที่ดึง current_user จาก context ไว้ที่เดียว
    def pundit_authorize!(record, query)
      Pundit.authorize(context[:current_user], record, query)
    end

    def current_user
      context[:current_user]
    end
  end
end
```

### เรียกใช้ใน `CreatePost`, `UpdatePost`, `DeletePost`

```ruby
# app/graphql/mutations/create_post.rb
module Mutations
  class CreatePost < BaseMutation
    argument :title, String, required: true
    argument :body, String, required: false
    argument :published, Boolean, required: false, default_value: false

    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    def resolve(title:, body: nil, published: false)
      pundit_authorize!(Post, :create?)   # <- เพิ่มบรรทัดนี้บรรทัดเดียว

      post = Post.new(title: title, body: body, published: published, user: current_user)

      if post.save
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

```ruby
# app/graphql/mutations/update_post.rb
def resolve(id:, **attrs)
  post = Post.find(id)
  pundit_authorize!(post, :update?)   # <- ตรวจสอบก่อนแก้ไข

  if post.update(attrs)
    { post: post, errors: [] }
  else
    { post: nil, errors: post.errors.full_messages }
  end
end
```

```ruby
# app/graphql/mutations/delete_post.rb
def resolve(id:)
  post = Post.find(id)
  pundit_authorize!(post, :destroy?)   # <- ตรวจสอบก่อนลบ

  post.destroy
  { post: post, errors: [] }
end
```

**สังเกตว่า `pundit_authorize!(Post, :create?)` ใน `CreatePost` ส่ง `Post` (class เอง) แทน
instance** เพราะตอนที่ตรวจสอบ "สร้างได้ไหม" ยังไม่มี record จริงให้ตรวจสอบเลย — Pundit
รองรับรูปแบบนี้เหมือนกันเพราะ `PostPolicy#create?` ของเราใช้แค่ `user.present?` ไม่ได้แตะ
`record` เลย (แนวทางเดียวกับที่ Part 043 Step 425 ใช้ `authorize @post` ใน controller
`create` action หลังจาก `@post = current_user.posts.build(...)`)

### เมื่อ Policy ปฏิเสธ — `Pundit::NotAuthorizedError` ต้องไม่กลายเป็น HTTP 403

ถ้าไม่จัดการอะไรเลย `Pundit.authorize` ที่คืน `false` จะ `raise Pundit::NotAuthorizedError`
เหมือนกับ `authorize` ใน controller — แต่ถ้าปล่อยให้ exception นี้หลุดออกไปจาก `resolve`
โดยไม่ดักไว้ **graphql-ruby จะปฏิบัติกับมันเหมือน uncaught exception ทั่วไป** คือทำให้
execution ทั้งก้อนล้มเหลวแบบ generic (`"errors": [{"message": "Internal server error"}]`
พร้อม backtrace รั่วไหลออกไปใน development) ซึ่ง **ไม่ใช่พฤติกรรมที่เราต้องการ** — เรา
ต้องการ error message ที่มีความหมาย เป็นส่วนหนึ่งของ `errors` array ตามปกติของ GraphQL

**ทางแก้ที่สะอาดที่สุด:** ใช้กลไก `rescue_from` **ที่ระดับ schema** (คนละตัวกับ `rescue_from`
ของ Rails controller แต่แนวคิดเดียวกัน — graphql-ruby มี DSL นี้ในตัว):

```ruby
# app/graphql/blog_schema.rb
class BlogSchema < GraphQL::Schema
  query(Types::QueryType)
  mutation(Types::MutationType)

  use GraphQL::Dataloader

  # แปลง Pundit::NotAuthorizedError ให้กลายเป็นส่วนหนึ่งของ "errors" array ใน
  # GraphQL response แทนที่จะปล่อยให้มันหลุดออกไปเป็น HTTP 500 — ทำงานได้ทั้งกับ error
  # ที่เกิดใน Query field และใน Mutation#resolve เพราะ rescue_from ของ graphql-ruby
  # ครอบทั้ง schema ไว้ที่จุดเดียว ไม่ต้องเขียนซ้ำในทุก mutation
  rescue_from(Pundit::NotAuthorizedError) do |err, _obj, _args, _ctx, _field|
    raise GraphQL::ExecutionError, "ไม่มีสิทธิ์ทำรายการนี้ (#{err.query})"
  end

  # ... ส่วนที่เหลือเหมือนเดิมจาก Part 058 ...
end
```

**อธิบาย:**

- `rescue_from(Pundit::NotAuthorizedError) do |err, obj, args, ctx, field| ... end` —
  graphql-ruby จะเรียก block นี้ทุกครั้งที่มี `Pundit::NotAuthorizedError` เกิดขึ้นระหว่าง
  execute ไม่ว่าจะเกิดใน field ไหนของ schema ก็ตาม (คล้ายกับ `rescue_from` ของ
  `ActionController::Base` ที่ Part 043 ใช้ แต่ทำงานที่ระดับ **schema** ไม่ใช่ **controller**)
- `err.query` คือ policy method ที่ถูกปฏิเสธ (เช่น `"update?"`) — `Pundit::NotAuthorizedError`
  มี attribute นี้ให้ใช้อยู่แล้วจากตัว gem เอง (ทบทวนได้จากการอ่าน source ของ pundit ตามที่
  Part 043 Step 422 แนะนำ)
- `raise GraphQL::ExecutionError, "..."` คือหัวใจของ Step นี้ — `GraphQL::ExecutionError`
  เป็น error class พิเศษของ graphql-ruby (ไม่ใช่ error ทั่วไปของ Ruby) ที่เมื่อ raise ออกมา
  จาก field ไหนก็ตาม **framework จะดักไว้ให้เองแล้วแปลงเป็น entry ใน `errors` array ระดับ
  บนสุดของ response โดยอัตโนมัติ พร้อม HTTP status ยังคงเป็น `200` เหมือนเดิม** — นี่คือ
  วิธี "มาตรฐาน" ของ graphql-ruby ในการสื่อสาร error ที่เกิดจาก framework/authorization
  layer (ต่างจาก `errors: [String]` field ใน payload ของ Step 583 ที่เป็น **business
  validation error ที่เราออกแบบเอง**)

### ทดสอบจริง: guest สร้างโพสต์ และ bob แก้ไขโพสต์ของ alice

```bash
# guest (ไม่มี header Authorization) พยายาม createPost
curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($title: String!) { createPost(input: { title: $title }) { post { id } errors } }", "variables": { "title": "guest post" }}'
```

ผลลัพธ์จริง (**HTTP status ยังเป็น `200` เหมือนเดิม**):

```json
{
  "errors": [{ "message": "ไม่มีสิทธิ์ทำรายการนี้ (create?)", "path": ["createPost"] }],
  "data": { "createPost": null }
}
```

```bash
# bob (ไม่ใช่เจ้าของ) พยายาม updatePost ของ alice
BOB_TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoyfQ.GhX5ZZ1LyLxp_EyvrR9-JoERVMC16Uz4g4jXgBnLqGI"

curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $BOB_TOKEN" \
  -d '{"query": "mutation($id: ID!, $title: String) { updatePost(input: { id: $id, title: $title }) { post { id title } errors } }", "variables": { "id": "1", "title": "แก้โดยบ็อบ" }}'
```

ผลลัพธ์จริง (HTTP status `200` เช่นกัน):

```json
{
  "errors": [{ "message": "ไม่มีสิทธิ์ทำรายการนี้ (update?)", "path": ["updatePost"] }],
  "data": { "updatePost": null }
}
```

**เปรียบเทียบให้เห็นภาพชัด:**

| | REST + Pundit (Part 043) | GraphQL + Pundit (Part นี้) |
|---|---|---|
| เมื่อไม่มีสิทธิ์ | `redirect_to`/`render` พร้อม HTTP `403` หรือ `303` | HTTP `200` เสมอ, error อยู่ใน `errors` array |
| กลไกที่ใช้ | `rescue_from Pundit::NotAuthorizedError` ใน controller | `rescue_from` ใน schema, แปลงเป็น `GraphQL::ExecutionError` |
| Client ตรวจสอบจาก | `response.status` | `response.body.errors` (โครงสร้าง JSON) |

หลักการของ Pundit (`PostPolicy`, `owner?`, `admin?`) **เหมือนเดิมทุกประการ** — สิ่งที่ต่างคือ
**ชั้นที่ห่อหุ้ม error ก่อนส่งกลับไปหา client** เท่านั้น

---

## Step 588: Object-level Authorization — ซ่อน Field ทั้งหมด vs Raise Error

Step 587 ป้องกันการ "เขียน" ข้อมูลที่ไม่มีสิทธิ์แล้ว แต่ยังมีอีกด้านหนึ่งที่สำคัญไม่แพ้กัน คือ
การ "อ่าน" — field `posts` และ `post` ใน `QueryType` (จาก Part 058) ตอนนี้คืนข้อมูลให้ทุกคน
เหมือนกันหมด ไม่ว่าจะเป็นโพสต์ draft ของคนอื่นหรือไม่ก็ตาม

### ใช้ `PostPolicy::Scope` กรอง list field (เหมือน `policy_scope` ใน REST)

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :posts, [Types::PostType], null: false,
      description: "รายการโพสต์ที่ผู้ใช้ปัจจุบันมีสิทธิ์เห็น"

    def posts
      Pundit.policy_scope!(context[:current_user], Post)
    end

    field :post, Types::PostType, null: true do
      argument :id, ID, required: true
    end

    def post(id:)
      Post.find_by(id: id)
    end
  end
end
```

`Pundit.policy_scope!` คือ standalone version ของ `policy_scope` ที่ใช้ใน REST controller
(Part 043 Step 427) เรียก `PostPolicy::Scope.new(user, Post).resolve` ให้อัตโนมัติเหมือนกัน
ทุกประการ — สังเกตว่า field `posts` **กรองข้อมูลตั้งแต่ต้นทาง** แบบเดียวกับที่ REST `index`
action ทำ ดังนั้น bob จะไม่เห็นโพสต์ draft ของ alice ปนอยู่ใน list เลย

### แต่ field `post(id:)` (ดึงทีละตัวด้วย ID) ยังไม่มีการป้องกันอะไรเลย

ถ้า bob **รู้ ID ตรงๆ** (เช่น เดา ID หรือเห็นจาก URL อื่น) เขายังคง query
`post(id: "1")` แล้วได้ข้อมูล draft ของ alice กลับมาอยู่ดี แม้ field `posts` (list) จะกรอง
ให้แล้วก็ตาม — นี่คือช่องโหว่แบบเดียวกับที่ REST app ต้องระวังเรื่อง "direct object
reference" (เช่น เข้า `/posts/1` ตรงๆ โดยไม่ผ่านหน้า index)

### ทางเลือกที่ 1: เช็คเองใน resolver แล้ว raise error (แบบ REST ที่คุ้นเคย)

```ruby
def post(id:)
  post = Post.find(id)
  pundit_authorize!(post, :show?)   # ต้องมี helper เดียวกับใน Step 587 ใน QueryType ด้วย
  post
end
```

วิธีนี้ตรงไปตรงมา แต่มีข้อเสีย: ถ้า schema มี field ที่คืน `PostType` หลายจุด (เช่น
`post(id:)`, `user.posts`, `comment.post`) **ต้องเขียนเช็คนี้ซ้ำทุกจุด** — เสี่ยงลืมเหมือนกับ
ปัญหาที่ Part 043 Step 428 อธิบายไว้เรื่อง "ลืม authorize ใน controller"

### ทางเลือกที่ 2: ผูก authorization ไว้ที่ตัว `Types::PostType` เอง (แนะนำ)

แทนที่จะเช็คซ้ำทุก field ที่ resolve ออกมาเป็น `PostType` เราผูกกฎ **"ใครเห็น Post ตัวนี้ได้
บ้าง"** ไว้ที่ตัว **type เอง** ครั้งเดียว แล้วให้ทุก field ที่คืนค่าเป็น `PostType` ใช้กฎ
เดียวกันโดยอัตโนมัติ — นี่คือสิ่งที่ Step 589 จะอธิบายกลไกเบื้องหลังแบบเจาะลึก

### ความแตกต่างสำคัญระหว่าง "ซ่อน" กับ "raise error"

ก่อนเข้ากลไกจริงใน Step 589 ต้องเข้าใจ **ผลลัพธ์ที่ต่างกัน** ของสองแนวทางนี้ก่อน:

**แนวทาง A — ซ่อนเงียบๆ (field เป็น nullable, เช่น `post(id: ID!): Post` ไม่มี `!` ต่อท้าย
`Post`):** ถ้าไม่มีสิทธิ์เห็น field คืนค่า `null` เฉยๆ **ไม่มี `errors` ใดๆ เลย** ผลลัพธ์จริง
(bob ยิง `post(id: "1")` ไปหา draft ของ alice): `{ "data": { "post": null } }` — client มอง
ไม่ออกด้วยซ้ำว่า "ไม่มีสิทธิ์" กับ "ID นี้ไม่มีอยู่จริง" ต่างกันอย่างไร ข้อดีคือ **ไม่รั่วไหล
ข้อมูลแม้แต่การมีอยู่ของ record** เหมาะกับข้อมูลที่ละเอียดอ่อนมาก

**แนวทาง B — raise error ชัดเจน (field เป็น non-null, `Post!`):** ถ้า authorization ปฏิเสธ
graphql-ruby จะ raise error ให้อัตโนมัติ เพราะ "สัญญา" ของ schema บอกว่า field นี้ห้าม
null แต่ authorization กลับบังคับให้เป็น null — ผลลัพธ์จริงจากการทดลองประกาศ field ทดสอบ
`secretPost: Post!`:

```json
{
  "errors": [{ "message": "Cannot return null for non-nullable field Query.secretPost", "path": ["secretPost"] }],
  "data": null
}
```

สังเกตว่า **`"data": null` ทั้งก้อน** ไม่ใช่แค่ field เดียว เพราะ GraphQL spec กำหนดว่าถ้า
non-null field คืนค่า null ต้อง "ลอย" ข้อผิดพลาดขึ้นไปยัง parent field ที่ nullable ตัวถัดไป
(ในที่นี้คือ `data` เอง ซึ่งเป็น root จึงกลายเป็น `null` ทั้งหมด) — เข้มงวดและ "ดัง" กว่าแนวทาง A
มาก เหมาะกับกรณีที่ต้องการให้ client รู้ทันทีว่ามีบางอย่างผิดปกติ ไม่ใช่แค่ "ไม่มีข้อมูล"

### หลักการเลือกใช้ในทางปฏิบัติ

| สถานการณ์ | แนะนำแนวทาง |
|---|---|
| field แบบ "ดึงทีละตัว" ที่อาจไม่เจอข้อมูลอยู่แล้วเป็นปกติ (เช่น `post(id:)`) | A: nullable + ซ่อนเงียบๆ (แยกไม่ออกจาก "ไม่มี record") |
| field ที่ควรมีข้อมูลเสมอถ้า query ผ่านมาถึงจุดนี้ได้ (เช่น `me { posts }` หลัง authenticate แล้ว) | B: non-null + ปล่อยให้ raise error ถ้าเกิดขึ้นจริง (ควรไม่เกิดขึ้นเลยถ้า logic ถูกต้อง) |
| mutation ที่ทำสิ่งที่ผิดกฎ business (validation) | ไม่ใช้ทั้ง A/B — ใช้ `errors: [String]` field ใน payload ตาม Step 583 |
| mutation ที่ผู้ใช้ไม่มีสิทธิ์ทำเลย (authorization) | ใช้ `GraphQL::ExecutionError` ตาม Step 587 |

---

## Step 589: Authorization Hook ของ graphql-ruby เอง — `self.authorized?` บน Type

Step 588 ทิ้งท้ายไว้ว่าอยากผูกกฎ "ใครเห็น Post ตัวนี้ได้บ้าง" ไว้ที่ตัว `Types::PostType`
เอง — graphql-ruby **มี hook นี้มาให้ในตัว gem อยู่แล้ว** ไม่ต้องเขียนกลไกเองเลย

### `self.authorized?(object, context)` — Hook ที่ graphql-ruby เรียกให้อัตโนมัติ

ทุก class ที่สืบทอดจาก `GraphQL::Schema::Object` (นั่นคือทุก type รวมถึง `Types::BaseObject`
ที่ generator สร้างให้) มี class method ชื่อ `authorized?` ที่ **ค่า default คืน `true` เสมอ**
(อนุญาตทุกอย่าง) — ถ้าเรา **override** method นี้ graphql-ruby จะเรียกมันให้เองโดยอัตโนมัติ
**ก่อน resolve field ไหนก็ตามที่คืนค่าเป็น type นั้น ทุกครั้ง ทุกจุดในทั้ง schema**

```ruby
# app/graphql/types/post_type.rb
module Types
  class PostType < Types::BaseObject
    field :id, ID, null: false
    field :title, String, null: false
    field :body, String, null: true
    field :published, Boolean, null: false
    field :user_id, ID, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false

    # graphql-ruby เรียก method นี้ให้เองอัตโนมัติ "ก่อน" resolve field ไหนก็ตามที่คืน
    # ค่าเป็น PostType (ทั้ง single object และแต่ละตัวใน list) — ไม่ต้องเรียกเองเลย
    def self.authorized?(object, context)
      super && Pundit.policy!(context[:current_user], object).show?
    end
  end
end
```

**อธิบายทีละส่วน:**

- `def self.authorized?(object, context)` — เป็น **class method** (สังเกต `self.`)
  ไม่ใช่ instance method เหมือน `resolve` ของ mutation — `object` คือ record จริง (ในที่นี้
  คือ instance ของ `Post`) ที่กำลังจะถูกส่งกลับไปให้ client, `context` คือ Hash เดียวกับที่
  ส่งเข้า `Schema.execute(..., context: {...})` ตั้งแต่ Step 586 ทุกประการ
- `super` เรียก parent class chain ก่อน (เผื่อ `Types::BaseObject` มี authorization
  logic ร่วมของตัวเองที่ทุก type ต้องผ่านด้วย) เป็นแนวปฏิบัติที่ปลอดภัยเสมอเมื่อ override
  hook ของ framework
- `Pundit.policy!(context[:current_user], object).show?` — เรียก `PostPolicy` ตัวเดิม
  จาก Part 043 ตรงๆ ไม่มีการเขียน authorization logic ใหม่เลยแม้แต่นิดเดียว
  (`Pundit.policy!` เป็น standalone version ของ `policy` helper ที่ raise error ถ้าหา
  policy class ไม่เจอ ต่างจาก `Pundit.policy` เฉยๆ ที่คืน `nil` เงียบๆ)

### ผลลัพธ์: ทุก field ที่คืน `PostType` ปลอดภัยพร้อมกันทันที ไม่ต้องแก้อะไรเพิ่ม

เพราะ hook นี้ผูกกับ **type** ไม่ใช่ผูกกับ field ใดเป็นพิเศษ ทั้ง `post(id:)`, `posts`,
และ field ในอนาคตที่จะคืน `PostType` (เช่น `user.posts` ถ้าเพิ่ม `UserType` ทีหลัง) **ได้รับ
การป้องกันแบบเดียวกันทันทีโดยไม่ต้องเขียน `pundit_authorize!` ซ้ำในทุก resolver** — แก้ปัญหา
"ลืมป้องกันบาง field" ที่ Step 588 พูดถึงไว้ได้อย่างสมบูรณ์ (คล้ายกับที่ `after_action
:verify_authorized` ใน Part 043 Step 428 แก้ปัญหา "ลืม authorize" ฝั่ง REST controller
เพียงแต่กลไกนี้ **ป้องกันไว้ล่วงหน้าโดยอัตโนมัติ แทนที่จะแค่ตรวจจับความผิดพลาดทีหลัง**)

### ทดสอบจริง: field `post(id:)` และ `posts` ตอนนี้ซ่อนข้อมูลอัตโนมัติแล้ว

```ruby
# bin/rails console
alice = User.find_by(email: "alice@example.com")
bob   = User.find_by(email: "bob@example.com")
draft = Post.create!(title: "ร่างลับของ Alice", body: "...", published: false, user: alice)

query = "query($id: ID!) { post(id: $id) { id title } }"

BlogSchema.execute(query, variables: { id: draft.id.to_s }, context: { current_user: bob }).to_h
# => {"data"=>{"post"=>nil}}          <- bob ไม่ใช่เจ้าของ ไม่เห็นเลย ไม่มี error

BlogSchema.execute(query, variables: { id: draft.id.to_s }, context: { current_user: alice }).to_h
# => {"data"=>{"post"=>{"id"=>"2", "title"=>"ร่างลับของ Alice"}}}   <- alice เห็นของตัวเอง
```

ผลลัพธ์ทั้งสองคือค่าที่ตรวจสอบจริงจากการรันในสภาพแวดล้อมทดสอบ — สังเกตว่า **เราไม่ได้แก้ไข
`def post(id:)` ใน `QueryType` เลยแม้แต่บรรทัดเดียว** ตั้งแต่ Step 588 (ยังคงเป็น
`Post.find_by(id: id)` เฉยๆ) การป้องกันทั้งหมดเกิดขึ้นที่ `PostType.authorized?` เพียงจุด
เดียว

field `posts` (list) ก็ได้รับการป้องกันซ้อนสองชั้นเช่นกัน: `Pundit.policy_scope!` (Step 588)
กรองที่ระดับ SQL query อยู่แล้ว แต่ถ้า `Scope` เขียนพลาด `authorized?` ที่ผูกกับ `PostType`
จะเป็นเกราะป้องกันชั้นที่สอง โดยกรอง item ที่ไม่ผ่านออกจาก list เงียบๆ อัตโนมัติ — นี่คือ
**defense in depth** (การป้องกันหลายชั้น) ที่ไม่พึ่งจุดป้องกันจุดเดียว

> **ข้อควรระวังเรื่อง performance:** `authorized?` ถูกเรียก **ทุกครั้ง ทุก object** ที่ resolve
> ออกมาเป็น type นั้น ถ้า policy ต้อง query ฐานข้อมูลเพิ่มต่อ record การ query list ยาวๆ อาจ
> กลายเป็น N+1 — แนวทางแก้คือ cache ค่าที่ต้องใช้ซ้ำไว้ใน `context` ตั้งแต่ต้น (เช่น
> `context[:current_user]`) ไม่ใช่ปัญหาที่เกิดในตัวอย่างนี้เพราะ `admin?`/`owner?` เช็คจาก
> attribute ที่โหลดมาพร้อม object อยู่แล้ว แต่ต้องระวังเมื่อ policy ซับซ้อนขึ้นในระบบจริง

---

## Step 590: Testing GraphQL Query/Mutation ด้วย RSpec (ต่อยอดจาก Part 019/046)

### ทำไมทดสอบ GraphQL ต่างจากทดสอบ REST controller เพียงเล็กน้อยเท่านั้น

Part 046 สอน request spec สำหรับ REST controller (`post "/posts", params: {...}` แล้วเช็ค
`response.status`/`response.body`) — GraphQL ก็ทดสอบผ่าน request spec ได้เหมือนกัน แต่มี
**ทางลัดที่สะดวกกว่า**: เพราะ `Schema.execute(query_string, variables:, context:)` เป็น
**Ruby method ธรรมดา** (ตามที่ใช้ทดลองใน console มาตลอด Part นี้) เราเรียกมันตรงๆ ใน spec
ได้เลยโดยไม่ต้องผ่าน HTTP layer จริงเลยด้วยซ้ำ (ไม่ต้องมี `post "/graphql", params: {...}`)
ทำให้ spec รันเร็วกว่า และเขียนง่ายกว่า request spec ทั่วไป

### เตรียม Factory (ต่อยอดจาก Part 047)

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { "password123" }
    role { :member }

    trait :admin do
      role { :admin }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    sequence(:title) { |n| "Post #{n}" }
    body { "เนื้อหาโพสต์ตัวอย่าง" }
    published { false }
    association :user
  end
end
```

### Spec สำหรับ `createPost` mutation

```ruby
# spec/graphql/mutations/create_post_spec.rb
require "rails_helper"

RSpec.describe "createPost mutation", type: :request do
  let(:user) { create(:user) }

  let(:mutation) do
    <<~GQL
      mutation($title: String!, $body: String) {
        createPost(input: { title: $title, body: $body }) {
          post { id title published }
          errors
        }
      }
    GQL
  end

  def execute(variables:, current_user: nil)
    BlogSchema.execute(
      mutation,
      variables: variables,
      context: { current_user: current_user }
    ).to_h
  end

  context "when the user is signed in" do
    it "creates a post and returns it with no errors" do
      result = execute(variables: { title: "โพสต์ใหม่", body: "เนื้อหา" }, current_user: user)

      expect(result.dig("data", "createPost", "errors")).to eq([])
      expect(result.dig("data", "createPost", "post", "title")).to eq("โพสต์ใหม่")
      expect(Post.count).to eq(1)
    end

    it "returns validation errors in the payload instead of an HTTP error" do
      result = execute(variables: { title: "" }, current_user: user)

      expect(result.dig("data", "createPost", "post")).to be_nil
      expect(result.dig("data", "createPost", "errors")).to include("Title can't be blank")
      expect(result["errors"]).to be_nil
    end
  end

  context "when there is no signed-in user (guest)" do
    it "does not create a post and reports an authorization error inside the errors array" do
      result = execute(variables: { title: "จะสร้างได้ไหม" }, current_user: nil)

      expect(Post.count).to eq(0)
      expect(result["errors"]).to be_present
      expect(result["errors"].first["message"]).to include("ไม่มีสิทธิ์")
      expect(result.dig("data", "createPost")).to be_nil
    end
  end
end
```

**อธิบาย:**

- `type: :request` ใช้ metadata เดียวกับ request spec ทั่วไปที่เรียนใน Part 046 แม้จะไม่ได้
  ยิง HTTP จริงเลย เพราะเรากำลังทดสอบ "พฤติกรรมของระบบเมื่อรับ input จากภายนอก" ซึ่งเป็น
  concern เดียวกับ request spec (ต่างจาก model spec ที่ทดสอบ logic ภายในของ class เดียว)
- สังเกตว่า `expect(result["errors"]).to be_nil` (บรรทัดสุดท้ายของ context แรก) ยืนยันด้วย
  ว่า **validation error ธรรมดาไม่ควรทำให้เกิด top-level GraphQL error เลย** — ตรงกับหลักการ
  ที่อธิบายไว้ใน Step 583 ทุกประการ ถ้า assertion นี้ล้มเหลว แปลว่ามีบางอย่างผิดปกติ (เช่น
  exception หลุดออกมาโดยไม่ตั้งใจ)
- context ที่สองยืนยันตรงกันข้าม: **authorization error ต้องปรากฏใน top-level `errors`**
  ไม่ใช่ใน `data.createPost.errors` — spec นี้จับความแตกต่างระหว่าง "business validation
  error" (Step 583) กับ "authorization error" (Step 587) ได้อย่างชัดเจนในทดสอบเดียว

### Spec สำหรับ `updatePost` mutation — ครอบคลุมทั้ง owner, admin, และคนอื่น

```ruby
# spec/graphql/mutations/update_post_spec.rb
require "rails_helper"

RSpec.describe "updatePost mutation", type: :request do
  let(:owner)        { create(:user) }
  let(:other_member) { create(:user) }
  let(:admin)        { create(:user, :admin) }
  let(:post_record)  { create(:post, user: owner, title: "เดิม") }

  let(:mutation) do
    <<~GQL
      mutation($id: ID!, $title: String) {
        updatePost(input: { id: $id, title: $title }) {
          post { id title }
          errors
        }
      }
    GQL
  end

  def execute(current_user:)
    BlogSchema.execute(
      mutation,
      variables: { id: post_record.id.to_s, title: "แก้ไขแล้ว" },
      context: { current_user: current_user }
    ).to_h
  end

  it "allows the owner to update their own post" do
    result = execute(current_user: owner)

    expect(result.dig("data", "updatePost", "post", "title")).to eq("แก้ไขแล้ว")
    expect(post_record.reload.title).to eq("แก้ไขแล้ว")
  end

  it "allows an admin to update someone else's post" do
    result = execute(current_user: admin)

    expect(result.dig("data", "updatePost", "post", "title")).to eq("แก้ไขแล้ว")
  end

  it "rejects another member with an authorization error, not an HTTP 403" do
    result = execute(current_user: other_member)

    expect(post_record.reload.title).to eq("เดิม")
    expect(result["errors"].first["message"]).to include("ไม่มีสิทธิ์")
  end
end
```

### Spec สำหรับ `posts` query — ทดสอบ object-level authorization จาก Step 588–589

```ruby
# spec/graphql/queries/posts_spec.rb
require "rails_helper"

RSpec.describe "posts query", type: :request do
  let(:owner)        { create(:user) }
  let(:other_member) { create(:user) }
  let(:admin)        { create(:user, :admin) }

  let!(:owner_draft)     { create(:post, user: owner, published: false) }
  let!(:owner_published) { create(:post, user: owner, published: true) }

  let(:query) { "{ posts { id published } }" }

  def ids_for(user)
    result = BlogSchema.execute(query, context: { current_user: user }).to_h
    result.dig("data", "posts").map { |p| p["id"] }
  end

  it "hides another member's draft from the list" do
    expect(ids_for(other_member)).to eq([owner_published.id.to_s])
  end

  it "shows the owner their own draft" do
    expect(ids_for(owner)).to contain_exactly(owner_draft.id.to_s, owner_published.id.to_s)
  end

  it "shows an admin every post" do
    expect(ids_for(admin)).to contain_exactly(owner_draft.id.to_s, owner_published.id.to_s)
  end

  it "shows a guest only published posts" do
    expect(ids_for(nil)).to eq([owner_published.id.to_s])
  end
end
```

### รันทดสอบทั้งหมดจริง

```bash
bundle exec rspec spec/graphql
```

```
..........

Finished in 0.36412 seconds (files took 1.12 seconds to load)
10 examples, 0 failures
```

ผลลัพธ์นี้คือผลการรันจริงจากสภาพแวดล้อมทดสอบ (Ruby 3.3.6 / Rails 8.1.4 / graphql 2.6.11 /
pundit 2.5.2) — ทั้ง 10 example ผ่านหมด ครอบคลุมทั้ง mutation, query, business validation
error, และ authorization error ที่เรียนมาตลอดทั้ง Part นี้

> **เชื่อมกับ Part 019/046:** โครงสร้าง `describe`/`context`/`it`, การใช้ `let`/`let!`,
> และ FactoryBot ที่ใช้ในทุก spec ข้างต้นเป็นความรู้เดียวกับที่เรียนไปแล้วใน Part 019 (RSpec
> เบื้องต้น) และ Part 046 (RSpec สำหรับ Rails) ทุกประการ — สิ่งเดียวที่ต่างจาก request spec
> ของ REST controller คือ **วิธีเรียก "ระบบที่จะทดสอบ"**: REST เรียกผ่าน `post "/posts",
> params: {...}` ส่วน GraphQL เรียกผ่าน `Schema.execute(query_string, variables:,
> context:)` ตรงๆ แต่ปรัชญาการเขียน spec (arrange-act-assert, การแยก context ตามบทบาท
> ผู้ใช้, การทดสอบทั้ง happy path และ authorization failure) เหมือนกันทุกประการ

---

## แบบฝึกหัด: สร้างระบบ Post ที่มี Mutation + Authorization ครบวงจร

### โจทย์

สร้างระบบ blog แบบง่าย (Post + User ที่ `belongs_to`/`has_many` กันเหมือน Part 043) ผ่าน
GraphQL API ทั้งหมด ให้ครบตามข้อกำหนดต่อไปนี้:

1. Mutation `createPost` — สร้างโพสต์ได้เฉพาะผู้ใช้ที่ login แล้ว โพสต์ที่สร้างเป็นของ
   `current_user` เสมอ (ห้ามให้ client ระบุ `userId` เอง)
2. Mutation `updatePost` — แก้ไขได้เฉพาะเจ้าของโพสต์หรือแอดมิน
3. Mutation `deletePost` — ลบได้เฉพาะเจ้าของโพสต์หรือแอดมิน
4. ทุก mutation ต้อง return `{ post, errors }` ตามธรรมเนียมที่เรียนใน Step 583 — ห้ามใช้
   HTTP status code สื่อความหมายความสำเร็จ/ล้มเหลว
5. Authorization error (ไม่มีสิทธิ์) ต้องปรากฏใน top-level `errors` ของ response (ผ่าน
   `GraphQL::ExecutionError`) ส่วน validation error (เช่น title ว่าง) ต้องปรากฏใน
   `data.<mutation>.errors` เท่านั้น
6. เขียน RSpec spec ยืนยันพฤติกรรมข้อ 5 ให้ครบทุกกรณี
7. ทดสอบจริงด้วย `curl` อย่างน้อย 1 กรณีที่สำเร็จ และ 1 กรณีที่ถูกปฏิเสธเพราะไม่มีสิทธิ์

### เฉลย

โจทย์นี้ไม่ต้องเขียนอะไรใหม่ตั้งแต่ศูนย์เลย — **ใช้ไฟล์ทั้งหมดตามที่เขียนไว้แล้วใน Step
582–589 ตรงๆ ทุกไฟล์ ไม่มีการแก้ไขเพิ่มเติมแม้แต่บรรทัดเดียว**:

- `app/models/user.rb`, `app/models/post.rb` — เหมือน Part 043 ทุกประการ
- `app/policies/post_policy.rb` — เหมือน Part 043 ทุกประการ (`show?`, `create?`,
  `update?`, `destroy?`, `Scope`, `owner?`, `admin?`) — นี่คือประเด็นสำคัญที่สุดของเฉลยนี้:
  **authorization logic ตัวเดียวกันใช้ได้ทั้งกับ REST controller (Part 043) และ GraphQL
  mutation (Part นี้) โดยไม่ต้องเขียนซ้ำแม้แต่บรรทัดเดียว**
- `app/lib/json_web_token.rb`, `app/controllers/graphql_controller.rb` — จาก Step 586
  (encode/decode JWT, ดึง `current_user` จาก header ใส่ `context`)
- `app/graphql/blog_schema.rb` — จาก Step 587 (`rescue_from(Pundit::NotAuthorizedError)`
  แปลงเป็น `GraphQL::ExecutionError`)
- `app/graphql/types/post_type.rb` — จาก Step 589 (`self.authorized?` ผูกกับ
  `PostPolicy#show?`)
- `app/graphql/types/query_type.rb` — จาก Step 588 (`posts` ใช้ `Pundit.policy_scope!`,
  `post(id:)` ดึงทีละตัว)
- `app/graphql/mutations/base_mutation.rb` — จาก Step 587 (helper `pundit_authorize!`,
  `current_user`)
- `app/graphql/mutations/create_post.rb`, `update_post.rb`, `delete_post.rb` — จาก
  Step 582/585/587 ตรงๆ (`pundit_authorize!(Post, :create?)`,
  `pundit_authorize!(post, :update?)`, `pundit_authorize!(post, :destroy?)` ตามลำดับ)
- `app/graphql/types/mutation_type.rb` — ลงทะเบียนทั้งสาม mutation จาก Step 585

**ข้อ 6 ของโจทย์ (RSpec ยืนยัน owner/admin/other_member)** ครอบคลุมไปแล้วโดย
`spec/graphql/mutations/update_post_spec.rb` ที่เขียนไว้ใน Step 590 ทุกประการ — ไม่ต้อง
เขียนเพิ่ม เพราะทดสอบครบทั้ง 3 เคสตามที่โจทย์ต้องการอยู่แล้ว (owner ผ่าน, admin ผ่าน,
other_member ถูกปฏิเสธด้วย error ใน top-level `errors`)

**ทดสอบจริงด้วย `curl` ตามข้อ 7 ของโจทย์ (สองผู้ใช้: alice เป็นเจ้าของ, bob ไม่ใช่เจ้าของ):**

กรณีสำเร็จ — alice แก้ไขโพสต์ของตัวเอง:

```bash
ALICE_TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.4RzAA-lIk5ohn8TKfm63BejeiGuGs16SkAjVbxCq59o"

curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  -d '{
    "query": "mutation($id: ID!, $title: String) { updatePost(input: { id: $id, title: $title }) { post { id title } errors } }",
    "variables": { "id": "1", "title": "แก้โดยอลิซเอง" }
  }'
```

ผลลัพธ์จริง (HTTP `200`):

```json
{"data":{"updatePost":{"post":{"id":"1","title":"แก้โดยอลิซเอง"},"errors":[]}}}
```

กรณีถูกปฏิเสธ — bob พยายามลบโพสต์ของ alice:

```bash
BOB_TOKEN="eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoyfQ.GhX5ZZ1LyLxp_EyvrR9-JoERVMC16Uz4g4jXgBnLqGI"

curl -s -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $BOB_TOKEN" \
  -d '{
    "query": "mutation($id: ID!) { deletePost(input: { id: $id }) { post { id } errors } }",
    "variables": { "id": "1" }
  }'
```

ผลลัพธ์จริง (**HTTP `200` แม้ถูกปฏิเสธ** — ตามธรรมเนียมที่เรียนตลอด Part นี้):

```json
{
  "errors": [
    {
      "message": "ไม่มีสิทธิ์ทำรายการนี้ (destroy?)",
      "locations": [{ "line": 1, "column": 22 }],
      "path": ["deletePost"]
    }
  ],
  "data": { "deletePost": null }
}
```

ยิงซ้ำด้วย `ALICE_TOKEN` แทน (เจ้าของจริง) เพื่อยืนยันว่า mutation เดิมทำงานได้ปกติ:

```json
{"data":{"deletePost":{"post":{"id":"1"},"errors":[]}}}
```

**สิ่งที่ทำให้เฉลยนี้สมบูรณ์:**

- Business rule ทั้งหมด (ใครทำอะไรได้บ้าง) อยู่ใน `PostPolicy` เพียงที่เดียว — เป็น class
  เดียวกับที่ Part 043 เขียนไว้สำหรับ REST controller ไม่มีการเขียน `if user.admin? ||
  user == post.user` ซ้ำที่ไหนในโค้ด GraphQL เลยแม้แต่จุดเดียว
- Authorization error (`Pundit::NotAuthorizedError`) และ validation error
  (`ActiveRecord` validation) แยกช่องทางกันอย่างชัดเจนตลอดทั้งระบบ: อย่างแรกไปที่ top-level
  `errors` ผ่าน `GraphQL::ExecutionError` อย่างหลังไปที่ `data.<mutation>.errors` ที่เรา
  ออกแบบเอง — HTTP status เป็น `200` เสมอไม่ว่ากรณีไหน
- `PostType.authorized?` ป้องกัน "การอ่าน" ที่ไม่มีสิทธิ์แบบอัตโนมัติในทุก field ที่คืน
  `PostType` โดยไม่ต้องเขียนเช็คซ้ำ ส่วน `Pundit.authorize`/`pundit_authorize!` ป้องกัน
  "การเขียน" ในทุก mutation
- ทดสอบครบทั้ง 3 ระดับ: RSpec (เร็ว, ไม่ต้องผ่าน HTTP), `curl` (จำลอง client จริงผ่าน HTTP
  เต็มรูปแบบ), และ console (สำหรับ debug ระหว่างพัฒนา) — ทั้งหมดเรียก `Schema.execute`
  เป็นจุดร่วมเดียวกัน

### แบบฝึกหัดเพิ่มเติม (ทำเอง ไม่มีเฉลย)

1. เพิ่ม mutation `publishPost` ที่เปลี่ยน `published` จาก `false` เป็น `true` เท่านั้น
   (ไม่รับ argument อื่นนอกจาก `id`) โดยต้องเพิ่ม policy method ใหม่ชื่อ `publish?` ใน
   `PostPolicy` (ให้สิทธิ์เหมือน `update?`) ไม่ใช่เรียก `update?` ซ้ำ แล้วเขียน RSpec spec
   ยืนยันว่า guest และสมาชิกที่ไม่ใช่เจ้าของถูกปฏิเสธด้วย `GraphQL::ExecutionError`

2. แปลง field `errors: [String]` ในทุก mutation ให้เป็น structured type ตามแบบ "userErrors
   pattern" ที่กล่าวถึงในกล่องหมายเหตุท้าย Step 583:

   ```graphql
   type UserError {
     field: String
     message: String!
   }
   ```

   โดย `field` ต้องบอกได้ว่า error เกิดจาก attribute ไหน (ใช้
   `post.errors.each.map { |e| { field: e.attribute, message: e.message } }` แทน
   `full_messages`) แล้วแก้ RSpec spec ที่เขียนไว้ให้ตรวจสอบ `field`/`message` แยกกัน

3. เพิ่ม `CommentType` และ `Mutations::CreateComment`/`Mutations::DeleteComment` สำหรับ
   โมเดล `Comment` (`belongs_to :post`, `belongs_to :user`) ตามกฎจาก Part 043 แบบฝึกหัด
   (ลบคอมเมนต์ได้ถ้าเป็นคนเขียนเอง, เป็นเจ้าของโพสต์, หรือเป็นแอดมิน) โดยต้องผูก
   `CommentType.authorized?` เข้ากับ `CommentPolicy#show?` ด้วย เพื่อให้กฎ "ใครเห็นคอมเมนต์
   ได้บ้าง" ถูกบังคับใช้แม้จะ query ผ่าน `post.comments` (field ที่ยังไม่ได้เขียนในแบบฝึกหัด
   นี้ — ต้องเพิ่ม field นี้ใน `PostType` เองด้วย)

---

## สรุปสิ่งที่ได้เรียนรู้ใน Part นี้

- เข้าใจว่า **Mutation** ในธรรมเนียมของ GraphQL คือช่องทางเดียวที่ควรใช้เขียนข้อมูล ในขณะที่
  **Query** ควรเป็น read-only เสมอ — เป็นข้อตกลงร่วมกันของชุมชน (convention) ไม่ใช่ข้อบังคับ
  ทางเทคนิคของภาษา แต่ graphql-ruby ใช้ข้อตกลงนี้กำหนดลำดับการ execute จริง (mutation รัน
  เรียงลำดับ, query รันพร้อมกันได้)
- เขียน mutation class ตามแบบแผน `Mutations::CreatePost < Mutations::BaseMutation` ด้วย
  `argument`, `field`, และ `resolve` พร้อมเข้าใจว่า `BaseMutation` สืบทอดจาก
  `GraphQL::Schema::RelayClassicMutation` ที่ห่อ argument เป็น `input` และห่อผลลัพธ์เป็น
  payload object ให้อัตโนมัติ
- เข้าใจธรรมเนียมสำคัญที่สุดของ GraphQL mutation คือ **ไม่ใช้ HTTP status code สื่อความสำเร็จ/
  ล้มเหลวของ business validation** แต่ใส่ผลลัพธ์ทั้งสองแบบ (สำเร็จ/ล้มเหลว) ไว้ใน payload
  เดียวกันผ่าน field `errors` (หรือ structured `userErrors`) — HTTP status ยังคงเป็น `200`
  เสมอตราบใดที่ request ถูก execute ได้
- เชื่อม mutation เข้ากับ root `MutationType` ด้วย `field :name, mutation: SomeClass` และ
  เรียกใช้จริงผ่านทั้ง GraphiQL และ `curl` พร้อม variables ตาม pattern
  `mutation($x: T!) { field(input: { x: $x }) { ... } }`
- เขียน `UpdatePost`/`DeletePost` ตามแบบแผนเดียวกับ `CreatePost` ทุกประการ ยืนยันว่า pattern
  นี้ scale ไปกับจำนวน mutation ที่เพิ่มขึ้นได้ง่าย
- ตั้งค่า authentication สำหรับ GraphQL request ด้วย JWT (ต่อยอดจาก Part 045) โดยส่ง
  `current_user` ผ่าน `context` ที่ `Schema.execute` รับเข้าไป แทนที่จะพึ่ง `before_action`
  แบบ REST controller เพราะ GraphQL มักมีแค่ 1 controller action สำหรับทุก query/mutation
- ใช้ Pundit policy เดิมจาก Part 043 (`PostPolicy`) ตรงๆ ภายใน mutation ผ่าน
  `Pundit.authorize`/`Pundit.policy_scope!` (standalone version ที่ไม่ต้องพึ่ง controller)
  และแปลง `Pundit::NotAuthorizedError` เป็น `GraphQL::ExecutionError` ผ่าน `rescue_from`
  ระดับ schema เพื่อให้ authorization error กลายเป็นส่วนหนึ่งของ top-level `errors` array
  แทนที่จะเป็น HTTP 403/500
- เข้าใจความแตกต่างระหว่างการ **ซ่อน field เงียบๆ** (nullable field ที่คืน `null` เมื่อไม่มี
  สิทธิ์ ไม่มี error ปรากฏเลย) กับการ **raise error ชัดเจน** (non-null field ที่ graphql-ruby
  บังคับให้ error เมื่อ authorization ปฏิเสธ) พร้อมหลักการเลือกใช้ในสถานการณ์ต่างๆ
- ใช้ authorization hook ของ graphql-ruby เอง `def self.authorized?(object, context)` บน
  type เพื่อผูกกฎ "ใครเห็น object นี้ได้บ้าง" ไว้ที่จุดเดียว แล้วให้ทุก field ที่คืน type นั้น
  ได้รับการป้องกันอัตโนมัติ โดยไม่ต้องเขียนเช็คซ้ำในทุก resolver — ทำงานร่วมกับ
  `Pundit.policy!` ได้ทันทีโดยไม่ต้องเขียน authorization logic ใหม่
- ทดสอบ GraphQL query/mutation ด้วย RSpec โดยเรียก `Schema.execute` ตรงๆ ใน spec (เร็วกว่า
  ยิง HTTP request จริง) ต่อยอดโครงสร้าง `describe`/`context`/`it`/`let`/FactoryBot ที่
  เรียนไปแล้วใน Part 019 และ Part 046 ทุกประการ — สิ่งเดียวที่ต่างคือวิธีเรียกระบบที่ทดสอบ

**ต่อไป (Part 060 — บทสุดท้ายของเฟส 8):** เราจะปิดท้ายเรื่อง API ด้วย **การเขียนเอกสาร API**
ด้วย `rswag`/OpenAPI (สร้าง Swagger UI ที่ทดสอบ endpoint ได้จากหน้าเอกสารโดยตรง ต่อยอดจาก
request spec ที่เขียนมาตลอด Part 046 และ Part นี้) และ **rate limiting** ด้วย `rack-attack`
(ป้องกัน API ถูกยิงถี่เกินไป ทั้งจาก IP เดียวและจาก user เดียวกัน) ซึ่งเป็น Part สรุปรวบยอดของ
เฟส 8 (APIs & GraphQL) ทั้งเฟส ก่อนจะย้ายไปเฟส 9 เรื่อง Background Jobs & Performance
