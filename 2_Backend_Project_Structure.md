# Backend Project Structure & Best Practices Standard

**Stack:** Node.js + Express + TypeScript
**ใช้สำหรับ:** มาตรฐานโครงสร้าง backend ของทีม (คู่กับ frontend React TS)

> Express เป็น unopinionated — ไม่มีโครงสร้างทางการ เอกสารนี้ยึดแนวทาง **layered architecture** + [nodebestpractices](https://github.com/goldbergyoni/nodebestpractices) ซึ่งเป็น de-facto standard ของวงการ

---

## หลักการ (Principles)

1. **แยกชั้นหน้าที่ชัดเจน** — route ไม่มี logic, controller จัดการ HTTP, service ถือ business logic, model คุยกับ DB
2. **แยก `app.ts` ออกจาก `server.ts`** — app คือการประกอบ express, server คือการ listen → ทำให้ test ง่าย
3. **config มาจาก env เสมอ** — ไม่ hardcode ค่าหรือ secret (ตาม 12-factor)
4. **error จัดการที่เดียว** — centralized error handler ปลายทาง
5. **validate input ที่ขอบ** — ตรวจ request ก่อนเข้า logic เสมอ

---

## โครงสร้างเต็ม (Layered)

```
backend/
├── src/
│   ├── server.ts                # bootstrap: โหลด env, เชื่อม DB, app.listen()
│   ├── app.ts                   # สร้าง express app, ผูก middleware + route (ไม่ listen)
│   │
│   ├── config/                  # อ่าน + validate env, ค่าคงที่ของระบบ
│   │   ├── env.ts               # validate env ตอนเริ่ม (ขาดตัวไหน fail ทันที)
│   │   └── index.ts
│   │
│   ├── routes/                  # map path → controller (บางที่สุด ไม่มี logic)
│   │   ├── index.ts             # รวมทุก route
│   │   ├── auth.routes.ts
│   │   └── ai.routes.ts
│   │
│   ├── controllers/             # รับ req/res, เรียก service, ส่ง response
│   │   ├── auth.controller.ts
│   │   └── ai.controller.ts
│   │
│   ├── services/                # ⭐ business logic จริง (ไม่รู้จัก req/res)
│   │   ├── auth.service.ts      # จัดการ OIDC flow, สร้าง session
│   │   └── ai.service.ts        # proxy ไป Python AI service
│   │
│   ├── repositories/            # data layer — คุยกับ DB เท่านั้น (Prisma/etc.)
│   │   └── user.repository.ts
│   │
│   ├── middlewares/             # auth, error handler, validation, rate limit
│   │   ├── auth.middleware.ts   # ตรวจ session ก่อนเข้า protected route
│   │   ├── error.middleware.ts  # centralized error handler
│   │   └── validate.middleware.ts
│   │
│   ├── validators/              # schema ตรวจ input (zod)
│   │   └── ai.validator.ts
│   │
│   ├── types/                   # TypeScript type ที่ใช้ร่วม
│   │   └── index.ts
│   │
│   ├── utils/                   # helper pure (logger wrapper, format)
│   │   ├── logger.ts            # pino/winston instance
│   │   └── ApiError.ts          # custom error class
│   │
│   └── lib/                     # client ภายนอกที่ตั้งค่าครั้งเดียว
│       └── aiClient.ts          # axios instance ชี้ Python AI service
│
├── tests/                       # โครงสร้างเลียนแบบ src/
├── .env
├── .env.example                 # commit อันนี้ (ไม่ commit .env จริง)
├── .eslintrc.cjs
├── .prettierrc
├── tsconfig.json
├── package.json
└── README.md
```

> **เมื่อโปรเจกต์โตขึ้น** สามารถเปลี่ยนจาก layered (แยกตามชั้น) ไปเป็น feature/component-based (แยกตาม domain เช่น `src/components/auth/` ที่รวม route+controller+service ของ auth ไว้ด้วยกัน) ซึ่ง nodebestpractices แนะนำสำหรับแอปใหญ่ — แต่เริ่มด้วย layered ก่อนได้

---

## หน้าที่ของแต่ละชั้น (สำคัญสุด)

| ชั้น | หน้าที่ | ห้ามทำ |
|------|---------|--------|
| `routes` | map path → controller เท่านั้น | ห้ามมี business logic |
| `controllers` | อ่าน req, เรียก service, ส่ง res | ห้ามคุย DB ตรงๆ |
| `services` | business logic ทั้งหมด | ห้ามแตะ req/res |
| `repositories` | query DB | ห้ามมี business logic |
| `middlewares` | auth, validation, error | — |

**ทำไมต้องแยก:** เปลี่ยน framework, เขียน test, หรือ reuse logic ได้ง่าย เพราะ business logic ไม่ผูกกับ HTTP

---

## Best Practices

### 1. แยก app กับ server
```ts
// app.ts — ประกอบ express (test ตรงนี้ได้โดยไม่ต้อง listen)
import express from 'express'
const app = express()
app.use(/* middlewares */)
app.use('/api', routes)
app.use(errorHandler)   // error handler อยู่ท้ายสุดเสมอ
export default app

// server.ts — bootstrap จริง
import app from './app'
import { env } from './config/env'
app.listen(env.PORT, () => logger.info(`running on ${env.PORT}`))
```

### 2. Centralized error handling
- โยน `ApiError` จาก service → จับที่ error middleware ตัวเดียว
- ใช้ wrapper หรือ `express-async-errors` กัน async error หลุด
- **ห้ามส่ง stack trace ออก response ใน production**

### 3. Config + secret
- อ่าน env ผ่าน `config/env.ts` และ **validate ตอนเริ่ม** (เช่นด้วย zod) — ขาดตัวไหน crash ทันที ดีกว่าพังตอน runtime
- ห้าม commit `.env` — commit แค่ `.env.example`

### 4. Security (สำคัญมากสำหรับงานธนาคาร)
- `helmet` — ตั้ง security header
- `cors` — จำกัด origin ให้เฉพาะ frontend ของเรา
- `express-rate-limit` — กัน brute force / abuse
- validate + sanitize ทุก input ด้วย `zod`
- session cookie เป็น `httpOnly` + `Secure` + `SameSite`

### 5. Logging
- ใช้ structured logger (`pino` หรือ `winston`) **ไม่ใช่ `console.log`**
- ใส่ request id เพื่อ trace ข้าม service ได้
- **ห้าม log ข้อมูลอ่อนไหว** (token, รหัส, ข้อมูลลูกค้า)

### 6. Production
- รันด้วย **PM2** (cluster mode ใช้หลาย core ได้)
- ทำ **graceful shutdown** — ปิด DB connection ก่อน process ตาย
- มี health check endpoint (`/health`) ให้ load balancer เช็ก

---

## หมายเหตุสำหรับบริบทนี้ (ธ.ก.ส.)

### Express = ประตูเดียว (BFF / Gateway)
- **OIDC flow** อยู่ใน `auth.service.ts` — Express ถือ client_secret, แลก code เป็น token, สร้าง session
- frontend ได้แค่ httpOnly session cookie — ไม่เห็น token ดิบ
- `auth.middleware.ts` ตรวจ session ก่อนเข้า protected route

### Proxy ไป Python AI service
- `ai.service.ts` เรียก Python ผ่าน `lib/aiClient.ts`
- แนบ **internal JWT สั้นๆ ที่ Express เซ็น** ไป — Python แค่ตรวจ JWT นี้ ไม่ต้องรู้จัก OIDC
- Python AI service เป็น **internal-only** ไม่เปิด public

```ts
// services/ai.service.ts (ย่อ)
export async function askAI(question: string, userId: string) {
  const internalToken = signInternalJwt({ userId })   // Express เซ็นเอง
  const { data } = await aiClient.post('/chat', { question }, {
    headers: { Authorization: `Bearer ${internalToken}` },
  })
  return data
}
```

---

## เครื่องมือมาตรฐานทีม

- **TypeScript** — type ทั้งโปรเจกต์
- **zod** — validate env + request
- **pino** — structured logging
- **helmet + cors + express-rate-limit** — security
- **ESLint + Prettier + Husky + lint-staged** — บังคับ style + รันก่อน commit
- **Vitest / Jest + supertest** — unit + integration test
- **PM2** — process manager ตอน production

---

## อ้างอิง

- nodebestpractices (de-facto standard): https://github.com/goldbergyoni/nodebestpractices
- Express security best practices: https://expressjs.com/en/advanced/best-practice-security.html
- Express performance best practices: https://expressjs.com/en/advanced/best-practice-performance.html