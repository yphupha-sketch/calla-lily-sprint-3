# การทำงานของระบบ — Calla Lily E-commerce

> เอกสารนี้อธิบายว่า **เว็บไซต์และ API ทำงานอย่างไรตอนรันจริง**
> ตั้งแต่ผู้ใช้กดอะไร → ข้อมูลวิ่งไปไหน → ระบบตอบกลับมาอย่างไร
> ใช้สำหรับคนที่ต้องอธิบายระบบ ทดสอบ หรือแก้บั๊กโดยไม่ต้องไล่อ่านโค้ดทั้งหมด
>
> ถ้าต้องการดูว่า "ทีมสร้างมันอย่างไร" ให้ดู `docs/build-order.md` แทน

---

## 1. ภาพรวบรวม

เว็บ Calla Lily เป็นร้านขายสินค้าอาบน้ำและบำรุงผิว (สบู่ มิสต์ ครีม บอดี้วอช น้ำหอม)
แสดงราคาเป็นบาท มี 2 ส่วนที่ทำงานแยกกันแต่เชื่อมผ่าน REST API

```
┌─────────────────────────────────────────────────────────────┐
│  Browser (ผู้ใช้)                                            │
│                                                              │
│   React 19 SPA  ── React Router ── React Context            │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │ AuthContext  │  │ CartContext  │  │ProductContext│      │
│   │ บัญชี + JWT  │  │ ตะกร้า       │  │ cache สินค้า  │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│                          │                                   │
│                    src/api.js  ← จุดเดียวที่ยิง HTTP          │
└──────────────────────────┬───────────────────────────────────┘
                           │  fetch() + Authorization: Bearer <JWT>
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  Express 5 API (Render)                                      │
│   CORS guard → cors → express.json → router                 │
│   routes/  →  middlewares/ (JWT, role)  →  controllers/     │
└──────────────────────────┬───────────────────────────────────┘
                           │  Mongoose
                           ▼
                    ┌─────────────┐
                    │  MongoDB    │  products / users / orders
                    └─────────────┘
                           ▲
                           │ เรียกจาก server เท่านั้น
        ┌──────────────────┴───────────────────┐
   ┌────┴─────┐                          ┌─────┴──────┐
   │  Stripe  │                          │  Gemini    │
   │ Checkout │                          │  (AI Chat) │
   └──────────┘                          └────────────┘
```

**หลักการสำคัญ:** secret ทั้งหมด (Stripe key, Gemini key, JWT secret, MongoDB URI)
อยู่ฝั่ง backend เท่านั้น เบราว์เซอร์ไม่เคยเห็น — เบราว์เซอร์คุยกับ Stripe และ Gemini ผ่าน backend เท่านั้น

---

## 2. ทุก request วิ่งผ่านอะไรบ้าง

### 2.1 ฝั่งเบราว์เซอร์ (`frontend/src/api.js`)

ทุกการเรียก API ต้องผ่าน `request()` ตัวเดียว ไม่มี component ไหนเรียก `fetch` เอง

```
component เรียก api.getProducts()
        │
        ▼
normalizeBase(VITE_API_URL)      ← ตัด "/" ท้าย + เติม "/api" อัตโนมัติ
        │                          กันตั้งค่า env ผิด
        ▼
อ่าน localStorage["calla-token"]  ← JWT
        │
        ▼
fetch(BASE + path, {
  headers: { Content-Type, Authorization: "Bearer <JWT>" },
  body: JSON.stringify(...)
})
        │
        ├─ res.ok  → return data
        └─ !res.ok → ถ้าเป็น 401:
                       ├── ลบ calla-token ออกจาก localStorage
                       └── ยิง event "calla:unauthorized" ที่ window
                                    │
                                    ▼
                          AuthContext ฟัง event นี้
                          → setToken(null) + setCurrentUser(null)
                          = ออกจากระบบอัตโนมัติทั้งระบบ
```

มี 2 แบบการเรียกที่ต่างกันตรงวิธีจัดการ error:

| แบบ | พฤติกรรมเมื่อพัง | ใช้ที่ไหน |
|-----|----------------|----------|
| `request()` | **throw** `Error` พร้อม `.status` → ผู้เรียกต้อง `try/catch` | สินค้า ออเดอร์ การเงิน แชท |
| `safeRequest()` | **คืนค่า** `{ error: "..." }` ไม่ throw | login / register / แก้โปรไฟล์ / เปลี่ยนรหัสผ่าน |

### 2.2 ฝั่ง server (`backend/src/server.js`)

Express ประมวลผลทุก request ตามลำดับนี้

```
1. CORS guard (โค้ดเอง)   ถ้ามี Origin header และไม่อยู่ใน CORS_ORIGIN
                          → ตอบ 403 { message: 'Origin not allowed by CORS' } ทันที
2. cors()                  ตั้ง Access-Control header
3. express.json()          แปลง body เป็น JSON object
4. router                  เลือก route ที่ตรง method + path
5. requireAuth/requireAdmin ตรวจ JWT + role (ถ้า route นั้นต้องล็อกอิน)
6. controller              ทำ logic ทั้งหมดใน try/catch
7. to<Entity>JSON()        แปลงเอกสาร Mongo เป็น JSON ที่ client ใช้ได้
                            (_id → id, ตัด password ออก)
```

ถ้า controller throw → จบที่ `res.status(500).json({ message: error.message })`
ระบบ **ไม่มี** global error handler และ **ไม่มี** 404 handler สำหรับ route ที่ไม่มีอยู่

### 2.3 เงื่อนไขการเชื่อมต่อ

| สถานการณ์ | ผลลัพธ์ |
|-----------|--------|
| `connectDB()` ล้มเหลว | พิมพ์ error แล้ว `process.exit(1)` |
| `MONGODB_URI` ไม่ได้ตั้ง | เชื่อมต่อไม่ได้ → process ตายทันที |
| `CORS_ORIGIN` ไม่ได้ตั้ง | พิมพ์คำเตือน `CORS_ORIGIN not set — allowing all origins (dev only)` แล้วเปิดให้ทุก origin |
| `JWT_SECRET` ไม่ได้ตั้ง | ใช้ค่า fallback `'calla-lily-dev-secret'` แบบเงียบ ๆ (ระบบยังทำงาน) |

---

## 3. ระบบบัญชีผู้ใช้

### 3.1 สมัครสมาชิก

```
หน้า /register
  กรอก 8 ช่อง + ยืนยันรหัสผ่าน
  → validatePassword() ฝั่ง client: ต้องยาว 8-14 ตัวอักษร
  → ถ้าผ่าน เรียก api.register()
        │
        ▼
POST /api/users/register   { name, email, phone, address, city, zip, password }
        │
        ├─ ไม่มี name หรือ email            → 400 'Name and email are required'
        ├─ password ไม่ใช่ 8-14 ตัวอักษร   → 400 'Password must be 8-14 characters'
        ├─ email ซ้ำ                         → 400 'An account with this email already exists'
        │
        ├─ email ถูก normalize: trim() + toLowerCase()  ← ทำทุกครั้ง ทั้งตอนค้นและตอนบันทึก
        ├─ password ถูก hash: bcrypt.hash(password, 10)
        └─ role = 'user' เสมอ (สมัครเองเป็นแอดมินไม่ได้)
        │
        ▼
201 { token, user }
        │
        ├─ token → localStorage["calla-token"]
        └─ user  → localStorage["calla-current-user"]
        ▼
navigate("/account")
```

### 3.2 เข้าสู่ระบบ

```
POST /api/users/login  { email, password }
        │
        ├─ User.findOne({ email: normalize(email) })
        ├─ ไม่เจอผู้ใช้  → 401 'Invalid email or password'
        ├─ bcrypt.compare(password, user.password)
        ├─ รหัสผ่านผิด → 401 'Invalid email or password'   ← ข้อความเดียวกันทั้ง 2 กรณี
        │                                                      (กันเดาว่าอีเมลนี้มีบัญชีไหม)
        ▼
200 { token, user }
```

### 3.3 JWT ทำงานอย่างไร

ตอน login/register สำเร็จ backend จะ **เซ็น token ให้ทันที** (ไม่ต้องมี endpoint `/me`)

```
payload = { id: <userId>, role: 'user' | 'admin' }
secret  = process.env.JWT_SECRET
อายุ    = 7 วัน
```

ทุกครั้งที่ยิง API ที่ต้องล็อกอิน `requireAuth` จะทำงาน 4 ขั้น

```
Authorization: Bearer <token>
        │
        ├─ ไม่มี token        → 401 'Not authorized, no token'
        ├─ jwt.verify() ผิด    → 401 'Not authorized, token failed'
        ├─ หา user ไม่เจอ      → 401 'Not authorized, user not found'   (บัญชีถูกลบไปแล้ว)
        └─ สำเร็จ → req.user = { id, role, email }  → next()
```

> ระบบ **query MongoDB ทุก request** เพื่อเช็คว่าผู้ใช้ยังมีอยู่จริง
> แปลว่าถ้าลบผู้ใช้ออกจาก DB token ที่ยังไม่หมดอายุจะใช้ไม่ได้ทันที

`requireAdmin` ไม่ได้เขียนแยก — มัน**เรียก `requireAuth` แล้วเช็ค role ต่อ**

```js
requireAdmin = async (req, res, next) => {
  await requireAuth(req, res, () => {
    if (req.user?.role !== 'admin') return 403 'Not authorized as admin';
    next();
  });
};
```

ดังนั้นใน route file เขียนแค่ `requireAdmin` พอ ไม่ต้องเขียน `requireAuth, requireAdmin` ซ้ำ

### 3.4 แก้ไขบัญชี / เปลี่ยนรหัสผ่าน

| การกระทำ | ปุ่มใน UI | เงื่อนไข |
|---------|----------|--------|
| แก้ชื่อ/อีเมล/เบอร์/ที่อยู่ | `PUT /api/users/:id/profile` | เจ้าของบัญชี หรือ admin เท่านั้น<br>เช็ค email ซ้ำแบบ `_id: { $ne: user._id }`<br>ช่องที่ไม่ได้ส่งมา (null) จะ**ไม่ถูกแก้** (ต่างจากช่องที่ส่งค่าว่าง) |
| เปลี่ยนรหัสผ่าน | `PUT /api/users/:id/password` | ต้องยืนยันรหัสผ่านเดิมถูกต้องก่อน<br>รหัสใหม่ต้องยาว 8-14 ตัวอักษร<br>hash ใหม่ด้วย bcrypt cost 10 |

> ข้อสังเกต: **รหัสผ่านสูงสุด 14 ตัวอักษร** เป็นข้อจำกัดที่ตั้งใจกำหนดเอง
> ทั้งฝั่ง client (`validatePassword`) และ backend บังคับใช้ข้อความเดียวกัน

### 3.5 เงื่อนไขการเข้าถึงหน้า (ฝั่ง client)

| หน้า | พฤติกรรมเมื่อยังไม่ล็อกอิน / ไม่ใช่ admin |
|------|------------------------------------------|
| `/checkout` | **ไม่ redirect** — แสดงปุ่ม Log In / Register ในหน้า |
| `/account` | **ไม่ redirect** — แสดง CTA ให้เข้าสู่ระบบ |
| `/admin/*` | **redirect ทันที** → `<Navigate to="/login" replace />` |

> ⚠️ การตรวจ role ฝั่ง client อ่านค่าจาก `localStorage` ซึ่งผู้ใช้แก้เองได้
> จึงเป็นแค่ UX ไม่ใช่ความปลอดภัย — **การป้องกันจริงอยู่ที่ `requireAdmin` ฝั่ง backend เท่านั้น**

---

## 4. ระบบสินค้า

### 4.1 สินค้าถูกโหลดครั้งเดียวแล้วแชร์ทั้งแอป

```
แอปเริ่มทำงาน
    │
    ▼
ProductProvider (useEffect ครั้งเดียว)
    │  GET /api/products        ← public ไม่ต้องล็อกอิน
    ▼
เก็บลง state "products"  ──────────┐
    │                              │
    ├─ ถ้าเรียกสำเร็จ: ได้ array สินค้า
    └─ ถ้าเรียกพัง:    setProducts([])  (เงียบ ๆ ไม่แสดง error)
                                   │
    ┌──────────────────────────────┴───────────────┐
    ▼              ▼            ▼          ▼      ▼
  Home        Products   ProductDetail  Dashboard  AdminProducts
```

**ผลของการออกแบบแบบนี้:**
- โหลดสินค้า 1 ครั้งต่อ session ทั้งที่มีหลายหน้า
- `ProductDetail` หา product จาก cache (`products.find(p => p.id === id)`)
  **ไม่ได้ยิง `GET /api/products/:id`** แม้ API นั้นจะมีอยู่
- กรอง/ค้นหา/แบ่งหน้า/เรียงลำดับ ทำทั้งหมดบน client → ไม่กิน bandwidth
- **ไม่มีการ sync กลับ** — ถ้าแอดมินแก้สินค้าจาก browser อีกเครื่อง
  หน้านี้จะไม่เห็นจนกว่าจะ reload

### 4.2 การจัดการสินค้า (แอดมิน)

```
หน้า /admin/products
    │
    ├─ เพิ่ม   → api.createProduct(data)   POST   /api/products
    ├─ แก้ไข  → api.updateProduct(id,data) PUT   /api/products/:id
    └─ ลบ     → window.confirm() → api.deleteProduct(id) DELETE /api/products/:id
    │
    ▼ (ทุกครั้ง: ยิง API ให้สำเร็จก่อน แล้วค่อยแก้ state ท้องถิ่น)
```

รายละเอียดที่ควรรู้:

| เรื่อง | พฤติกรรมจริง |
|-------|-----------|
| `requireAdmin` | ทุก endpoint เขียน/แก้/ลบสินค้าต้องเป็น admin |
| ตัวเลข | `Number(x) || 0` → ถ้าส่งค่าที่แปลงเป็นตัวเลขไม่ได้จะกลายเป็น 0 |
| `stock` ไม่ได้ส่งมา | ตอนสร้างใหม่จะเป็น `0` |
| อัปเดตบางฟิลด์ | ใช้ `if (x != null)` → **ฟิลด์ที่ส่งมาเป็น `null` จะถูกข้าม ไม่ถูกล้าง** |
| ลบสินค้า | ลบ document ถาวร ไม่มี soft delete / กู้คืนไม่ได้ |
| สินค้าถูกลบแล้ว | ออเดอร์เก่ายังแสดงรายการนั้นได้ เพราะเก็บ `name`/`price`/`image` ซ้ำไว้ในออเดอร์ |

### 4.3 การเลือกดูสินค้าในหน้าร้าน

| หน้า | ทำอะไร |
|------|--------|
| `/` (Home) | แสดง hero + สินค้าแนะนำ 3 ชิ้นแรก |
| `/product` | ค้นหาข้อความ, กรองหมวด, เรียงราคา, แบ่งหน้า 9 ชิ้น/หน้า<br>ค่าทั้งหมดอยู่ใน URL (`?cat=`, `?page=`) → กด F5 / share ลิงก์แล้วได้ผลเดิม |
| `/product/:id` | ดูสินค้าเดียว + สินค้าที่เกี่ยวข้อง (หมวดเดียวกัน สูงสุด 3)<br>stepper จำนวนถูกจำกัดระหว่าง `1` ถึง `stock` |

---

## 5. ระบบตะกร้าสินค้า

ตะกร้าเก็บ **เฉพาะในเบราว์เซอร์** ไม่มีตาราง cart ในฐานข้อมูล

```
localStorage["calla-cart"]  =  [
  { id, name, price, image, category, stock, description, quantity },
  ...
]
```

| การกระทำ | ผลลัพธ์ |
|---------|--------|
| `addToCart(product, qty=1)` | ถ้ามี id นี้อยู่แล้ว → **บวกจำนวนเข้าไป** ไม่ใช่เพิ่มแถวใหม่ |
| `increaseQty(id)` | `quantity + 1` |
| `decreaseQty(id)` | `quantity - 1` แต่**ไม่ต่ำกว่า 1** (ลบทิ้งทางลัดไม่ได้ ต้องกด Remove) |
| `removeFromCart(id)` | เอาออกทั้งแถว |
| `clearCart()` | ล้างทั้งหมด (ถูกเรียกหลังชำระเงินสำเร็จ) |
| `itemCount` | ผลรวม `quantity` ทั้งหมด → ตัวเลขบน badge ใน Navbar |
| `totalPrice` | ผลรวม `price × quantity` |

**การบันทึกอัตโนมัติ:** ทุกครั้งที่ `cart` เปลี่ยน `useEffect` จะเขียนทับ `localStorage`
เปิดเว็บใหม่จึงยังเห็นตะกร้าเดิม (ถ้าข้อมูลใน localStorage เสีย จะเริ่มที่ตะกร้าว่าง)

> ⚠️ ข้อจำกัดที่ต้องทราบ: ตะกร้า **ผูกกับ browser ไม่ผูกกับผู้ใช้**
> ล็อกอินจากเครื่องอื่อ → ตะกร้าว่าง, เปิดแบบ incognito → ตะกร้าว่าง,
> ล้างข้อมูลเว็บ → ตะกร้าหายถาวร

**ค่าคงที่:** ค่าจัดส่ง = `0` (ส่งฟรี) กำหนดไว้ใน `Cart.jsx` และส่ง `shipping: 0` ไปทุกครั้ง

---

## 6. ระบบสั่งซื้อและชำระเงิน (Stripe) — หัวใจของระบบ

นี่คือ flow ที่ยาวที่สุดของเว็บ มี 4 ฝั่งที่ต้องทำงานสอดคล้องกัน:
ผู้ใช้ → เว็บเรา → Stripe → เว็บเรา (อีกครั้ง)

### 6.1 ภาพรวมทั้ง flow

```
[1] /checkout            ผู้ใช้กรอกที่อยู่ + กด "Pay with Card"
   │                       (ยังไม่มีออเดอร์, ยังไม่ตัดสต็อก)
   ▼
[2] POST /api/payments/checkout        →  backend สร้าง Stripe Session
   │                                          คืนค่า { url }
   ▼
[3] window.location = url              →  ออกจากเว็บเราไปหน้า Stripe
   │                       (ข้อมูลบัตรถูกกรอกบนเว็บ Stripe เท่านั้น)
   ▼
[4] ผู้ใช้จ่ายเงินบน Stripe           →  สำเร็จ
   ▼
[5] Stripe redirect กลับมา            →  /payment-success?session_id=cs_xxx
   │
   ▼
[6] POST /api/orders  { sessionId, ... }  →  backend ยืนยันกับ Stripe
   │                                          แล้วสร้างออเดอร์ + ตัดสต็อก
   ▼
[7] clearCart() → /order-success        →  แสดงใบเสร็จ
```

### 6.2 ขั้นที่ 1 — หน้า Checkout

```
ผู้ใช้เปิด /checkout
    │
    ├─ ไม่ได้ล็อกอิน → แสดงปุ่ม Log In / Register (ไม่ redirect ออกจากหน้า)
    ├─ ตะกร้าว่าง     → แสดง "Nothing to check out" + ปุ่มไปหน้าสินค้า
    ▼
ฟอร์มถูกเติมข้อมูลล่วงหน้าจากโปรไฟล์ใน currentUser
(ชื่อ, อีเมล, เบอร์, ที่อยู่, เมือง, รหัสไปรษณีย์ — ผู้ใช้แก้ทับได้)
    │
    ▼
กดปุ่ม "Pay with Card"  →  handleSubmit()
```

> UI มีวิธีชำระเงิน**ทางเดียว** คือบัตรผ่าน Stripe
> (โค้ด backend ยังรองรับ `payment` แบบ COD ไว้ แต่ไม่มีปุ่มเรียกใช้แล้ว)

### 6.3 ขั้นที่ 2 — Backend สร้าง Stripe Checkout Session

```
POST /api/payments/checkout          (ต้องล็อกอิน)
{ items, customer, subtotal, shipping: 0, discount: 0, total }
    │
    ├─ items ว่าง                        → 400 'Your cart is empty'
    ├─ ไม่มี customer.fullName/email     → 400 'Shipping details are required'
    │
    ▼
สำหรับแต่ละชิ้นในตะกร้า — อ่านข้อมูลจริงจาก DB ใหม่ทุกครั้ง:
    │
    ├─ แปลง item.id เป็น ObjectId  (ล้มเหลว = ไม่ใช่ id ที่ถูกต้อง)
    ├─ Product.findById() → ไม่เจอ        → 400 'Product "..." was not found'
    ├─ product.stock < item.quantity      → 400 '"..." has only N in stock...'
    │
    ▼
สร้าง line_items จากข้อมูลใน DB เท่านั้น:
    currency:     'thb'                    (Stripe ใช้หน่วยสตางค์)
    unit_amount:   Math.round(product.price * 100)     ← บาท → สตางค์
    product_data:  { name, images }         ← ชื่อ/รูปจาก DB ไม่ใช่จาก client
    │
    ▼
คำนวณยอดใหม่ทั้งหมดฝั่ง server:
    subtotal    = ผลรวมจาก line_items
    totalAmount = subtotal + shipping×100 − discount×100
    ถ้า totalAmount ≤ 0 → 400 'Order total must be greater than zero'
    │
    ▼
stripe.checkout.sessions.create({
  mode: 'payment',
  line_items,
  customer_email,
  metadata: { shipping: JSON.stringify(customer) },  ← เก็บที่อยู่ไว้ใน Stripe
  success_url: `${CLIENT_URL}/payment-success?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url:  `${CLIENT_URL}/checkout`,
})
    │
    ▼
200 { url }        ← frontend ทำ window.location.href = url
```

**ทำไมต้องอ่านราคาใหม่จาก DB:** ถ้าเชื่อ `item.price` ที่ client ส่งมา
ผู้ใช้แก้ request ใน devtools เป็น 1 บาทได้ → ระบบจะขายสินค้าราคาจริงในราคา 1 บาท
โค้ดชุดนี้คือ **แนวป้องกันหลักของระบบทั้งหมด**

**ทำไมเก็บที่อยู่ใน `metadata`:** เพราะหลังจาก redirect ออกจากเว็บเราไปที่ Stripe
ตัวแปร React (`form`) จะหายไป แต่ `metadata` ของ session ยังอ่านกลับมาได้ตอนยืนยันออเดอร์

### 6.4 ขั้นที่ 3–4 — ผู้ใช้จ่ายเงิน (บนเว็บ Stripe)

```
หน้า Checkout ถูกปล่อยทิ้ง → browser ไปที่ Stripe
  → ผู้ใช้กรอกข้อมูลบัตรบนเว็บ Stripe
  → จ่ายเงิน
       ├─ สำเร็จ            → redirect ไป success_url
       └─ กดยกเลิก/ปิดหน้า  → redirect ไป cancel_url = /checkout
```

> ข้อมูลบัตร **ไม่เคยผ่านเซิร์ฟเวอร์เรา** เลย — เป็นไปตามข้อกำหนด PCI DSS
> ทางเลือกนี้ทำให้โค้ดฝั่งเราเบาขึ้นมาก ไม่ต้องจัดการข้อมูลบัตรเอง

### 6.5 ขั้นที่ 5–6 — ยืนยันและสร้างออเดอร์ (สำคัญที่สุด)

หน้า `/payment-success` จะยิง `POST /api/orders` **อัตโนมัติทันทีที่โหลด**
ป้องกันการยิงซ้ำด้วย state `done` (โค้ดมี eslint-disable เพราะตั้งใจให้ยิงครั้งเดียว)

```
POST /api/orders                      (ต้องล็อกอิน)
{ items, customer, subtotal, shipping, discount, total, sessionId }
    │
    ▼ ⑥a. กันออเดอร์ซ้ำ (idempotency)
    Order.findOne({ stripeSessionId: sessionId })
    ├─ เจอแล้ว → คืนออเดอร์เดิมทันที (200)
    │           ★ ทำให้กด refresh หน้า payment-success ซ้ำไม่ทำให้ออเดอร์ซ้ำ
    │
    ▼ ⑥b. ยืนยันกับ Stripe ว่าจ่ายจริง
    stripe.checkout.sessions.retrieve(sessionId)
    ├─ เรียกไม่ได้            → 400 'Payment session could not be verified'
    ├─ payment_status ≠ 'paid' → 400 'Payment has not been completed'
    │
    ▼ ⑥c. เทียบยอดเงิน
    Math.round(total × 100)  vs  stripeSession.amount_total
    ├─ ไม่ตรง → 400 'Order total does not match the paid amount'
    │          ★ กันกรณียอดในระบบกับยอดที่จ่ายจริงไม่ตรงกัน
    │
    ▼ ⑥d. กู้ที่อยู่จาก Stripe metadata
    customer = JSON.parse(stripeSession.metadata.shipping)   ← ข้อมูลที่กรอกตอน checkout
    (ถ้าอ่านไม่ได้จะใช้ customer จาก request body แทน)
    ├─ ไม่มี fullName/email → 400 'Shipping details are required'
    │
    ▼ ⑥e. ตรวจสต็อกอีกครั้ง (DB ใหม่)
    ทุกชิ้น: ต้องมีอยู่จริง และ stock ≥ quantity
    ├─ ขาด            → 400 'Product "..." was not found'
    └─ สต็อกไม่พอ     → 400 '"..." has only N in stock...'
    │
    ▼ ⑥f. สร้างออเดอร์
    orderId        = `CL-` + เลข 6 หลักท้ายของ Date.now()      (เช่น CL-481293)
    userId         = req.user.id
    payment        = 'card'                                    (เพราะมี sessionId)
    stripeSessionId = sessionId                                (ใช้กันซ้ำ + ตามหาในอนาคต)
    customer, subtotal, shipping, discount, total
    status         = 'processing'                              (ค่า default)
    │
    ▼ ⑥g. ตัดสต็อกทีละชิ้น
    Product.findByIdAndUpdate(id, { $inc: { stock: -quantity } })
    │
    ▼
201 { order }   →   frontend clearCart() → /order-success
```

### 6.6 ขั้นที่ 7 — ใบเสร็จ

```
<OrderSuccess />  อ่านข้อมูลจาก location.state.order
  ↑ ถูกส่งมาตอน navigate(..., { state: { order } }) ไม่ได้ยิง API ซ้ำ
  │
  └─ ถ้าผู้ใช้กด refresh → location.state หาย → แสดง "No order found"
```

> ⚠️ ใบเสร็จจึงเป็นหน้าที่ "ดูครั้งเดียว" ไม่มีปุ่ม reload ออเดอร์
> ถ้าต้องการดูซ้ำให้ไปที่ `/account` (ประวัติคำสั่งซื้อ) หรือ `/tracking?id=CL-xxxxxx`

### 6.7 จุดที่ต้องระวัง (พฤติกรรมจริง ไม่ใช่ข้อตั้งใจ)

| สถานการณ์ | สิ่งที่เกิดขึ้นจริง |
|-----------|----------------|
| จ่ายเงินสำเร็จ แต่ปิดแท็บ/เน็ตล่มก่อน redirect กลับมา | **เงินเข้าแล้ว แต่ไม่มีออเดอร์ในระบบ สต็อกไม่ถูกตัด**<br>เพราะระบบไม่มี Stripe webhook — ออเดอร์ถูกสร้างโดยหน้าเว็บเท่านั้น |
| สร้างออเดอร์สำเร็จ แต่ `$inc` ตัดสต็อกพัง | ออเดอร์มีแล้ว สต็อกไม่ลด (ไม่มี transaction ครอบ 2 ขั้นนี้) |
| ผู้ใช้กดสั่งซื้อพร้อมกัน 2 คนจนสต็อกเหลือ 1 | `$inc` ไม่มีเงื่อนไขกันติดลบ → สต็อกอาจติดลบได้ (oversell) |
| กด refresh ที่ `/payment-success` | ปลอดภัย — ระบบเจอ `stripeSessionId` เดิมแล้วคืนออเดอร์เดิม ไม่สร้างซ้ำ |
| ยอดในตะกร้าเปลี่ยนระหว่างสร้าง session กับยืนยัน | ตอนยืนยันจะเทียบยอดกับ Stripe แล้ว → ถ้าไม่ตรงจะถูกปฏิเสธ (400) |

> ถ้าต้องการแก้ข้อ 1 ให้ถูกต้องจริง ๆ ต้องเพิ่ม Stripe webhook
> (`checkout.session.completed`) ให้ backend สร้างออเดอร์เองโดยไม่ต้องรอหน้าเว็บ
> รายละเอียดข้อควรระวังอยู่ใน `docs/stripe-payments.md`

---

## 7. สถานะออเดอร์ และการติดตามพัสดุ

### 7.1 สถานะที่ระบบรองรับ

```
pending ──► confirmed ──► processing ──► shipped ──► delivered
                          ▲
                    (ค่าเริ่มต้น)
                          └──► cancelled
```

| จุดที่เก็บ | รูปแบบ |
|---------|-------|
| ในฐานข้อมูล (Mongoose enum) | ตัวพิมพ์เล็ก `processing`, `shipped`, ... ค่า default = `processing` |
| ตอนส่งให้ client (`readStatus()`) | ขึ้นต้นตัวใหญ่ `Processing`, `Shipped` |
| ในหน้าแอดมิน | `<select>` มีให้เลือกแค่ **3 ค่า** — Processing / Shipped / Delivered |
| ในหน้าติดตามของลูกค้า | รูปจำลอง 3 ขั้น — Processing → Shipped → Delivered |

> `pending`, `confirmed`, `cancelled` อยู่ใน schema แต่**ไม่มีใน UI**
> และถ้าส่งค่าอื่นเข้ามา (ผ่าน API โดยตรง) Mongoose จะ throw เพราะไม่ตรง enum → กลายเป็น 500

### 7.2 แอดมินเปลี่ยนสถานะ

```
หน้า /admin/orders → เปลี่ยนค่าใน <select>
    │
    ▼
PATCH /api/orders/:id  { status }         (ต้อง admin)
    │
    ├─ หา order จาก Mongo _id ก่อน → ถ้าไม่ได้ (CL-xxxxxx ไม่ใช่ ObjectId)
    │  ให้ลอง findOne({ orderId }) ก็ไป → เจอ
    ├─ order.status = status.toLowerCase()   ← แปลงกลับเป็นตัวพิมพ์เล็กก่อนบันทึก
    └─ save()
    │
    ▼
คืนออเดอร์ที่อัปเดตแล้ว → frontend อัปเดต state ทันที
              └─ ถ้าล้มเหลว → ดึงข้อมูลใหม่ทั้งหมดจาก API (fallback)
```

**หมายเหตุ:** ช่อง `<select>` ส่งค่า `order.id` ซึ่งคือเลข `CL-xxxxxx` ไม่ใช่ Mongo `_id`
ระบบจึงต้องลองทั้งสองแบบ — เป็นการออกแบบให้รองรับทั้งสองรูปแบบ ID โดยเจ้าของโค้ดให้อนุญาต

### 7.3 ติดตามพัสดุ (ไม่ต้องล็อกอิน)

```
/tracking?id=CL-481293
    │
    ├─ ค่าใน URL เป็นแหล่งข้อมูล → เปิดลิงก์ตรง ๆ / share ได้ / กด F5 ได้
    ▼
GET /api/orders/track?q=CL-481293          (public — ไม่ต้องล็อกอิน)
    │
    ├─ q ว่าง        → 400 'Query is required'
    ├─ Order.findOne({ orderId: q } หรือ { 'customer.email': q.toLowerCase() })
    │   ├─ ไม่เจอ → 404 → หน้าเว็บแสดง "Order not found"
    │   └─ เจอ   → คืนออเดอร์ 1 รายการ
    ▼
แสดง 3 ขั้น โดยขั้นปัจจุบันมาจาก STEP_INDEX[status]
  Processing (0) → Shipped (1) → Delivered (2)
  ถ้า status ไม่อยู่ในแผนผัง (เช่น cancelled) → ถือว่าอยู่ขั้น 0
```

> ⚠️ **ค้นด้วย email จะได้ผลลัพธ์เดียวเสมอ** เพราะ backend ใช้ `findOne`
> ถ้าลูกค้าคนเดียวมีหลายออเดอร์ จะเห็นแค่ออเดอร์เดียว
> และไม่ได้ sort ด้วย → ไม่รับประกันว่าเป็นออเดอร์ล่าสุด
> (หน้า `/account` จะแสดงครบทุกออเดอร์ เพราะใช้ endpoint คนละตัว)

### 7.4 ประวัติคำสั่งซื้อในหน้า Account

```
หน้า /account → api.getMyOrders()
    │
    ▼
GET /api/orders/mine                    (ต้องล็อกอิน)
    │
    Order.find({
      $or: [
        { userId: req.user.id },              ← สร้างจากบัญชีนี้
        { 'customer.email': req.user.email }  ← สั่งด้วยอีเมลนี้ (แม้ทั้งออเดอร์)
      ]
    }).sort({ createdAt: -1 })
    │
    ▼
แสดงรายการ (เรียงใหม่สุดก่อนอีกครั้งด้วย sortOrdersNewest ฝั่ง client)
  แต่ละออเดอร์: ป้ายสถานะ + ปุ่ม "Track →" → /tracking?id=CL-xxxxxx
```

> เงื่อนไข `$or` สองทางนี้ทำให้ **ออเดอร์ที่สั่งก่อนสมัครสมาชิก
> (ด้วย email เดียวกัน) ยังเจอในประวัติ**

---

## 8. ระบบหลังบ้าน (Admin)

### 8.1 การป้องกันหน้า

```
ผู้ใช้เปิด /admin/orders
    │
    ▼
AdminLayout.jsx
    if (!currentUser || currentUser.role !== 'admin')
        → <Navigate to="/login" replace />          ← ออกจากระบบ = เข้า /login ใหม่
    │
    ▼ ผ่าน
Layout 2 ชั้น: sidebar (Dashboard / Products / Orders / Customers) + <Outlet/>
```

| ชั้นความปลอดภัย | กลไก |
|--------------|------|
| UX (ฝั่ง client) | ตรวจ `currentUser.role` จาก localStorage ก่อนแสดงเนื้อหา |
| ความปลอดภัยจริง | `requireAdmin` ที่ backend ตรวจทุก endpoint |

> แม้ผู้ใช้จะแก้ role ใน localStorage ให้เป็น admin ได้ แต่เมื่อเรียก API
> ฝั่ง server ยังตรวจจาก **JWT ที่เซ็นไว้** ซึ่งแก้ไม่ได้ → ได้ 403 ทันที

### 8.2 หน้าต่าง ๆ

| หน้า | ข้อมูลที่ต้องโหลด | การกระทำ |
|------|----------------|---------|
| `/admin` (Dashboard) | `getOrders()` + `getUsers()` + `products` จาก context | อ่านอย่างเดียว<br>4 การ์ดสรุป (ยอดขาย, จำนวนออเดอร์, สินค้า, ลูกค้า) + ตาราง 5 ออเดอร์ล่าสุด |
| `/admin/products` | `products` จาก context | CRUD เต็มรูปแบบ (ฟอร์มเดียวใช้ทั้งเพิ่มและแก้ไข) |
| `/admin/orders` | `getOrders()` | เปลี่ยนสถานะ, ลิงก์ไปหน้า Tracking |
| `/admin/customers` | `getUsers()` | อ่านอย่างเดียว: ชื่อ อีเมล เบอร์ role วันที่สมัคร |

**ตัวเลขบน Dashboard คำนวณจากอะไร**

| การ์ด | สูตร |
|-------|------|
| ยอดขายรวม | ผลรวม `order.total` ของ**ทุกออเดอร์** (รวมที่ยกเลิกแล้ว) |
| จำนวนออเดอร์ | `orders.length` |
| สินค้า | `products.length` |
| ลูกค้า | `users.length` |

---

## 9. ระบบ Calla AI (แชทผ่าน Gemini)

ปุ่มแชทลอยอยู่มุมขวาล่างของทุกหน้า (อยู่ใน `Layout` จึงโผล่ทั้งเว็บ)

### 9.1 flow การทำงาน

```
ผู้ใช้พิมพ์คำถาม
    │
    ▼
AIChat.jsx ส่งประวัติทั้งหมดไปเลย
  POST /api/chat   { messages: [{ role, text }, ...] }      (public — ไม่ต้องล็อกอิน)
    │
    ▼ ⑨a. ตรวจ config
    ไม่มี GEMINI_API_KEY → 503 'AI chat is not configured...'
    │
    ▼ ⑨b. ทำความสะอาดประวัติ
    ├─ เอาแค่ 16 ข้อความล่าสุด
    ├─ เก็บเฉพาะ role ที่เป็น 'user' หรือ 'model' ที่ text ไม่ว่าง
    ├─ trim และตัดที่ 2,000 ตัวอักษร
    └─ ★ ตัดทิ้งทุกข้อความก่อนถึงข้อความ 'user' แรก
         (เพราะหน้าเว็บมีข้อความต้อนรับจาก AI ก่อน ซึ่ง Gemini ไม่ยอมรับ)
    ├─ ไม่เหลืออะไรเลย หรือข้อความล่าสุดไม่ใช่จาก user
    │  → 400 'Please send a message for the assistant.'
    │
    ▼ ⑨c. อ่านสินค้าจริงจาก DB
    Product.find({}).select('name price category description stock')
    แปลงเป็นบรรทัด: "- ชื่อ | หมวด | ราคา บาท | พร้อมจำหน่าย/สินค้าหมด | คำอธิบาย"
    │
    ▼ ⑨d. ประกอบ system instruction
    • บทบาท: ผู้ช่วย Calla AI ของร้าน ตอบภาษาไทย สุภาพ กระชับ
    • ใช้รายการสินค้าที่แนบมาเป็นแหล่งข้อมูลเดียวสำหรับ ชื่อ/ราคา/หมวด/สต็อก
    • ★ ห้ามแต่งข้อมูลเรื่องสินค้า ราคา สต็อก โปรโมชั่น หรือสถานะออเดอร์
    • ถ้าไม่มีข้อมูลที่ถาม → บอกตรง ๆ และแนะนำให้ติดต่อร้าน
    │
    ▼ ⑨e. เรียก Gemini (timeout 25 วินาที)
    POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent
      header: x-goog-api-key: <GEMINI_API_KEY>     ← อ่านจาก env ตอน runtime
      body:   { systemInstruction, contents, generationConfig: { temperature: 0.5, maxOutputTokens: 700 } }
    │
    ▼ ⑨f. แปลง error ให้เป็นข้อความที่ผู้ใช้อ่านเข้าใจ
    ├─ Gemini ตอบ 429 (ติดลิมิต) → 429 'AI is busy right now...'
    ├─ Gemini ผิดพลาดอื่น      → 502 'The AI service could not answer...'
    ├─ คำตอบว่างเปล่า          → 502 '...returned an empty response...'
    ├─ เกิน 25 วินาที           → 504 'The AI service is temporarily unavailable...'
    └─ สำเร็จ                   → 200 { reply }   (รวมทุก part ของคำตอบ)
```

### 9.2 กลไกการกัน "AI แต่งข้อมูล"

ระบบแชทใช้ **grounding** 2 ชั้น ไม่ใช่แค่สั่งโมเดลใน prompt อย่างเดียว

```
ชั้นที่ 1  ข้อมูลใน prompt มาจาก DB จริง ณ เวลาที่ถาม (ไม่ใช่ข้อมูลตายตัว)
ชั้นที่ 2  สั่งห้ามแต่ง + ถ้าไม่มีข้อมูลให้บอกว่าไม่มี
ชั้นที่ 3  การอ่าน DB เป็น read-only (lean + select) ไม่แตะ stock
```

### 9.3 ข้อควรระวัง

| เรื่อง | รายละเอียด |
|-------|-----------|
| API key | อยู่ฝั่ง server อย่างเดียว เบราว์เซอร์ไม่เคยเห็น — ปลอดภัยเรื่องการรั่วไหล |
| ไม่มี rate limit | endpoint เปิดสาธารณะ **ไม่ต้องล็อกอิน** แต่เรียกใช้โควตา Gemini ทุกครั้ง<br>มีต้นทุนจริงและถูกเรียกจาก script ได้ |
| ประวัติสูงสุด 16 ข้อความ | บทสนทนายาว ๆ จะลืมบริบท แต่ลดการกินโควตา |
| ตอบได้ครั้งละ 700 token | คำตอบยาว ๆ จะถูกตัด |
| ตอบเป็นภาษาไทย | กำหนดไว้ใน prompt ไม่มีการตรวจภาษาฝั่ง server |

---

## 10. ตาราง API ทั้งหมด

### 10.1 ฝั่งสินค้า

| Method | Path | ใครเรียกได้ | ทำอะไร |
|--------|------|------------|--------|
| GET | `/api/products` | ทุกคน | ดึงสินค้าทั้งหมด |
| GET | `/api/products/:id` | ทุกคน | ดึงสินค้า 1 ชิ้น *(มีใน API แต่ยังไม่มีหน้าไหนเรียกใช้)* |
| POST | `/api/products` | admin | เพิ่มสินค้า |
| PUT | `/api/products/:id` | admin | แก้ไขสินค้า |
| DELETE | `/api/products/:id` | admin | ลบสินค้า |

### 10.2 ฝั่งผู้ใช้

| Method | Path | ใครเรียกได้ | ทำอะไร |
|--------|------|------------|--------|
| POST | `/api/users/register` | ทุกคน | สมัครสมาชิก → `{ token, user }` |
| POST | `/api/users/login` | ทุกคน | เข้าสู่ระบบ → `{ token, user }` |
| GET | `/api/users` | admin | ดูผู้ใช้ทั้งหมด |
| PUT | `/api/users/:id/profile` | เจ้าของ หรือ admin | แก้โปรไฟล์ |
| PUT | `/api/users/:id/password` | เจ้าของ หรือ admin | เปลี่ยนรหัสผ่าน |

### 10.3 ฝั่งออเดอร์ / การเงิน / AI

| Method | Path | ใครเรียกได้ | ทำอะไร |
|--------|------|------------|--------|
| POST | `/api/payments/checkout` | ผู้ล็อกอิน | สร้าง Stripe Checkout Session → `{ url }` |
| POST | `/api/orders` | ผู้ล็อกอิน | สร้างออเดอร์ + ตัดสต็อก (ยืนยันกับ Stripe ก่อน) |
| GET | `/api/orders` | admin | ดูออเดอร์ทั้งหมด (เรียงใหม่สุดก่อน) |
| GET | `/api/orders/mine` | ผู้ล็อกอิน | ดูออเดอร์ของตัวเอง |
| GET | `/api/orders/track?q=` | **ทุกคน** | ค้นหาด้วยเลขออเดอร์หรือ email → 1 รายการ |
| GET | `/api/orders/:id` | เจ้าของ หรือ admin | ดูออเดอร์เดียว *(ยังไม่มีหน้าไหนเรียกใช้)* |
| PATCH | `/api/orders/:id` | admin | เปลี่ยนสถานะออเดอร์ |
| POST | `/api/chat` | **ทุกคน** | ถาม Calla AI → `{ reply }` |
| GET | `/` | ทุกคน | health check → `{ message: 'Calla Lilly API is running' }` |

> ⚠️ จุดที่เปิดสาธารณะโดยไม่ต้องล็อกอิน: `GET /api/products*`,
> `POST /api/users/{login,register}`, `GET /api/orders/track`, `POST /api/chat`

### 10.4 รูปแบบ JSON ที่ client ได้รับ

ทุก controller แปลงเอกสาร Mongo ก่อนส่งออก ดังนั้น client **ไม่เคยเห็น `_id`**

```jsonc
// สินค้า
{ "id": "6f1a...", "name": "...", "price": 180, "category": "Relaxation",
  "image": "...", "description": "...", "stock": 24, "createdAt": "..." }

// ผู้ใช้ (ไม่มี password)
{ "id": "6f2b...", "name": "...", "email": "...", "phone": "...",
  "address": "...", "city": "...", "zip": "...", "role": "user",
  "memberSince": "..." }

// ออเดอร์ — สังเกตว่า id คือเลข CL-xxxxxx ไม่ใช่ Mongo id
{ "id": "CL-481293", "date": "...",
  "items": [{ "id": "...", "productId": "...", "name": "...", "price": 180,
              "quantity": 2, "image": "..." }],
  "customer": { "fullName": "...", "email": "...", "phone": "...",
                "address": "...", "city": "...", "zip": "..." },
  "payment": "card", "status": "Processing",
  "subtotal": 360, "shipping": 0, "discount": 0, "total": 360 }
```

> สินค้าในออเดอร์ถูก "ตรึงค่าไว้" ตอนสั่งซื้อ (name/price/image)
> ดังนั้นถ้าวันหลังแอดมินแก้ราคาสินค้า หน้าเว็บจะยังแสดงราคาเดิมในออเดอร์เก่า

---

## 11. โครงสร้างข้อมูลในฐานข้อมูล

```
users                     products                 orders
┌──────────────────┐      ┌──────────────────┐     ┌──────────────────────┐
│ _id  (ObjectId)  │      │ _id  (ObjectId)  │     │ _id      (ObjectId)  │
│ name             │      │ name             │     │ orderId  CL-xxxxxx   │ ← unique
│ email   ← unique │      │ price   (min 0)  │     │ userId ────┐         │
│ password  (hash) │      │ category        │     │ items[] ───┼──┐      │
│ role user|admin  │      │ image           │     │ customer   │  │      │
│ phone            │      │ description     │     │ payment    │  │      │
│ address/city/zip │      │ stock   (min 0)  │     │ stripeSessionId     │
│ memberSince      │      └──────────────────┘     │ subtotal/shipping   │
└──────────────────┘             ▲                 │ discount/total      │
        ▲                        │                 │ status  (enum ×6)   │
        └────────────────────────┴─────────────────┴──────────────────────┘
         1 ผู้ใช้ → หลายออเดอร์            1 สินค้า → หลาย item ในออเดอร์
```

**จุดที่ต้องรู้เรื่อง schema**

| เรื่อง | คำอธิบาย |
|-------|----------|
| `Order.items[].id` กับ `productId` | มีทั้งสองฟิลด์ — `productId` คือ ObjectId จริงที่ผูกกับ `products`<br>`id` เป็น string ที่ client ส่งมา (legacy จากตอนยังใช้ localStorage ที่ id เป็นตัวเลข) |
| `Order.payment` | String อิสระ ไม่มี enum — ค่าที่เขียนจริงคือ `"card"` แต่ default ของ schema คือ `"COD"` |
| `Order.shippingAddress` | **ฟิลด์ที่ไม่เคยถูกเขียน** — เป็นของค้างจากระยะก่อน ไม่มี controller ไหนส่งค่านี้ |
| `Order.stripeSessionId` | ใช้กันออเดอร์ซ้ำ แต่ **ไม่ได้ทำ index** และไม่มี unique |
| `status` default | `"processing"` — ออเดอร์ใหม่จึงเริ่มที่ขั้น "กำลังจัดการ" ทันที |
| ไม่มี `paymentStatus` | ไม่มีการแยก "จ่ายแล้ว/ยังไม่จ่าย" — เชื่อ `stripeSessionId` ว่ามี = จ่ายแล้ว |
| `createdAt`/`updatedAt` | ทั้ง 3 collection ใช้ `timestamps: true` |

---

## 12. localStorage ทั้งหมดที่เว็บใช้

| Key | เก็บอะไร | เขียนเมื่อ | ลบเมื่อ |
|-----|----------|-----------|--------|
| `calla-cart` | ตะกร้าสินค้า (array) | ทุกครั้งที่ตะกร้าเปลี่ยน | `clearCart()` หลังจ่ายเงินสำเร็จ |
| `calla-token` | JWT (ดิบ ไม่มี `JSON.stringify`) | login / register สำเร็จ | logout หรือ API ตอบ 401 |
| `calla-current-user` | ข้อมูลผู้ใช้ (JSON) | login / register / แก้โปรไฟล์ | logout หรือ API ตอบ 401 |

> ทั้ง 3 key นี้อยู่ **เฉพาะในเบราว์เซอร์** ไม่มี cookie, ไม่มี session store ฝั่ง server
> ข้อดีคือ backend ไม่ต้องเก็บ session — ข้อเสียคือ token อยู่ในเครื่องผู้ใช้
> และถ้าล้างข้อมูลเว็บก็ต้องล็อกอินใหม่

---

## 13. สรุปเส้นทางของผู้ใช้ (happy path)

```
เปิดเว็บ
  │
  ├─ ProductProvider โหลดสินค้าจาก API (ครั้งเดียว)
  └─ เห็นหน้า Home → เลือกสินค้า → /product/:id
       │
       ├─ ไม่ล็อกอิน ──► สมัคร/เข้าสู่ระบบ ──┐
       └─ ล็อกอินแล้ว ───────────────────────┤
                                             ▼
                                    เพิ่มลงตะกร้า (calla-cart)
                                             │
                                             ▼
                                          /cart ──► แก้จำนวน / ลบ
                                             │
                                             ▼
                                    /checkout ──► กรอกที่อยู่
                                             │      (ต้องล็อกอิน)
                                             ▼
                              POST /payments/checkout ──► ไปที่ Stripe
                                             │
                                    ┌────────┴────────┐
                              จ่ายสำเร็จ            ยกเลิก
                                    │                  │
                                    ▼                  ▼
                          /payment-success          /checkout
                                    │
                       POST /orders (ยืนยันกับ Stripe)
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
              สร้างออเดอร์ + ตัดสต็อก          ข้อมูลไม่ผ่าน → 400
                    │
                    ▼
              ล้างตะกร้า → /order-success (ใบเสร็จ)
                                    │
    ┌───────────────────────────────┼──────────────────────────┐
    ▼                               ▼                          ▼
/account (ประวัติ)          /admin เปลี่ยนสถานะ        /tracking?id=CL-xxxxxx
                                    │                          │
                                    ▼                          ▼
                            status เปลี่ยนใน DB          ลูกค้าเห็น 3 ขั้น
                                                            ขยับตามทันที
```

---

## 14. สิ่งที่ระบบ "ไม่ทำ" (ข้อจำกัดที่ต้องรู้)

เพื่อไม่ให้เข้าใจระบบผิดว่ามีความสามารถที่ไม่มีจริง

| ไม่มี | ผลกระทบ |
|------|--------|
| **ตะกร้าบน server** | ตะกร้าผูกกับ browser ไม่ผูกกับบัญชี เปลี่ยนอุปกรณ์แล้วตะกร้าหาย |
| **Stripe webhook** | จ่ายเงินสำเร็จแต่ redirect ไม่กลับมา → เงินเข้าแล้วแต่ไม่มีออเดอร์ |
| **การคืนเงิน (refund)** | ไม่มี endpoint คืนเงิน ต้องทำเองผ่าน Stripe Dashboard |
| **ระบบโปรโมชั่น / คูปอง** | มีฟิลด์ `discount` ใน schema แต่ไม่มี UI ให้ใช้ ค่าทุกออเดอร์เป็น 0 |
| **คำนวณค่าจัดส่งจริง** | ค่าจัดส่งเป็น 0 เสมอ (ส่งฟรี) |
| **รายการโปรด / รีวิว** | ไม่มี |
| **ระบบค้นหาฝั่ง server** | ค้นหา/กรอง/เรียง/แบ่งหน้าทำบน client จากข้อมูลทั้งหมดที่โหลดมา |
| **อัปโหลดรูปภาพสินค้า** | admin ต้องกรอก URL รูปเอง ไม่มีปุ่มอัปโหลด |
| **แจ้งเตือนอีเมล** | ไม่มี email ยืนยันบัญชีหรือแจ้งสถานะพัสดุ |
| **ประวัติการเปลี่ยนแปลงสถานะ** | เก็บแค่สถานะปัจจุบัน ไม่มี timeline |
| **การล้างออเดอร์เก่า** | ออเดอร์กินพื้นที่ฐานข้อมูลเรื่อย ๆ |
| **rate limiting** | ทั้ง `/api/chat` และ `/api/users/login` เรียกได้ไม่จำกัด |

---

## 15. วิธีรันและทดสอบด้วยมือ

### 15.1 เริ่มระบบ

```bash
# Terminal 1 — backend
cd backend
cp .env.example .env       # ตั้งค่า MONGODB_URI, JWT_SECRET, STRIPE_SECRET_KEY,
                          # CLIENT_URL, GEMINI_API_KEY, GEMINI_MODEL
npm install
npm run seed               # เติมสินค้า 12 ชิ้น + ผู้ใช้ 2 คน
npm run dev                # http://localhost:5000

# Terminal 2 — frontend
cd frontend
cp .env.example .env       # เว้นว่างไว้ก็ได้ ใช้ /api ซึ่ง proxy ไปที่ :5000
npm install
npm run dev                # http://localhost:5173
```

### 15.2 บัญชีทดสอบ

| บทบาท | อีเมล | รหัสผ่าน |
|-------|-------|---------|
| ลูกค้า | `suda@example.com` | `hello123` |
| แอดมิน | `admin@callalily.com` | `admin123` |

### 15.3 ทดสอบ API ด้วย curl

```bash
# ข้อมูลสาธารณะ
curl http://localhost:5000/api/products

# ล็อกอินแล้วเก็บ token ไว้ใช้ต่อ
TOKEN=$(curl -s -X POST http://localhost:5000/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@callalily.com","password":"admin123"}' | jq -r .token)

# ต้องมี token
curl -H "Authorization: Bearer $TOKEN" http://localhost:5000/api/users

# สิทธิ์ admin
curl -X PATCH http://localhost:5000/api/orders/CL-481293 \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"status":"Shipped"}'

# ติดตามออเดอร์ (ไม่ต้องล็อกอิน)
curl "http://localhost:5000/api/orders/track?q=CL-481293"
```

> ถ้าลืมใส่ token จะได้ `401 Not authorized, no token`
> ถ้าใส่ token ของลูกค้าเรียก endpoint ของ admin จะได้ `403 Not authorized as admin`

### 15.4 ทดสอบการชำระเงิน

ใช้ **test key** ของ Stripe เท่านั้น (`sk_test_...`) และการ์ดทดสอบ

| หมายเลขการ์ด | ผลลัพธ์ |
|-------------|--------|
| `4242 4242 4242 4242` | จ่ายสำเร็จ |
| `4000 0000 0000 0002` | ถูกปฏิเสธ |

กดผ่านหน้า Checkout แล้วดูว่า:
1. ถูกพาไปหน้า Stripe จริง
2. หลังจ่าย กลับมาที่ `/payment-success` แล้วเด้งไป `/order-success`
3. ตะกร้าว่างขึ้น
4. สินค้าในหน้า Admin มี stock ลดลง
5. เปิด `/tracking?id=CL-xxxxxx` เห็นสถานะ `Processing`
6. เปลี่ยนสถานะที่ `/admin/orders` แล้วกลับไปดูหน้า Tracking เปลี่ยนตาม

---

## 16. ดูเอกสารอื่นต่อได้ที่

| ไฟล์ | เนื้อหา |
|------|--------|
| `docs/build-order.md` | ลำดับการสร้างโปรเจกต์และหลักการทำงานของทีม |
| `docs/DESIGN.md` | สัญญา design system (สี/ตัวอักษรที่อนุญาตให้ใช้) |
| `docs/stripe-payments.md` | แผนรองรับ Stripe และแนวทางสร้าง webhook |
| `docs/system-design.md` | ภาพรวมสถาปัตยกรรมระบบ |
| `docs/code-explanation-th.md` | อธิบายโค้ดเป็นภาษาไทย |
| `docs/presentation-20min.md` | สคริปต์นำเสนอโปรเจกต์ 20 นาที |
| `docs/README.md` | ภาพรวมโปรเจกต์ + สิ่งที่ทำได้ดี + สิ่งที่ควรปรับปรุง |
