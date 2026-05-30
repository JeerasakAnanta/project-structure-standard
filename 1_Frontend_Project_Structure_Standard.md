# Frontend Project Structure Standard


**Stack:** Vite + React + TypeScript
**ใช้สำหรับ:** ใช้งาน สำหรับการพัฒนา โครงสร้างโปรเจกต์ หน้าบ้าน ของทีมเรา

---

## หลักการ (Principles)
  
1. **Shared อยู่ชั้นบน, feature-specific อยู่ใน feature** — ถ้าโค้ดใช้หลาย feature ให้ไว้ชั้นบน (`components/`, `hooks/`, `lib/`) ถ้าใช้เฉพาะ feature เดียวให้ co-locate ไว้ใน `features/<name>/`
2. **เลือกแล้วต้องสม่ำเสมอ** — ทุกคนในทีมวางของที่เดิมเสมอ
3. **Flat ไว้ก่อน** — อย่าซ้อน folder ลึกเกินจำเป็น
4. **ทุกอย่างเป็น TypeScript** — `.tsx` สำหรับ component, `.ts` สำหรับ logic/util
5. **Feature ห้ามนำเข้ากันเองข้ามโมดูล** — ถ้าต้อง share ให้ยกขึ้นไปชั้นบน

---

## โครงสร้างเต็ม

```
my-app/
├── public/                      # static asset ที่ไม่ผ่าน build (favicon, robots.txt)
├── src/
│   ├── main.tsx                 # entry point — mount React เข้า DOM
│   ├── App.tsx                  # root component + provider หลัก
│   ├── vite-env.d.ts            # type ของ Vite (อย่าลบ)
│   │
│   ├── assets/                  # รูป/ฟอนต์ที่ import เข้าโค้ด (ผ่าน build)
│   │
│   ├── components/              # UI ที่ใช้ซ้ำได้ทั้งแอป (ไม่ผูกกับ feature)
│   │   ├── ui/                  # primitive: Button, Input, Modal, Card
│   │   └── layout/              # Header, Sidebar, Footer, PageContainer
│   │
│   ├── features/                # ⭐ หัวใจของแอป — แยกตาม domain
│   │   ├── auth/                # การ login ผ่าน OIDC
│   │   │   ├── components/      # LoginButton, ProtectedRoute
│   │   │   ├── hooks/           # useAuth, useUser
│   │   │   ├── api/             # เรียก /login, /callback, /logout ผ่าน backend
│   │   │   ├── types.ts         # User, Session, AuthState
│   │   │   └── index.ts         # export เฉพาะของที่ให้คนอื่นใช้
│   │   ├── dashboard/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── types.ts
│   │   └── ai-chat/             # ฟีเจอร์คุยกับ AI (ยิงผ่าน backend)
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api/
│   │       └── types.ts
│   │
│   ├── pages/                   # หน้าตาม route (ประกอบจาก feature + component)
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx
│   │   └── NotFoundPage.tsx
│   │
│   ├── routes/                  # นิยาม route ทั้งหมด (react-router)
│   │   └── index.tsx
│   │
│   ├── hooks/                   # custom hook ที่ใช้ร่วมหลาย feature
│   │   └── useDebounce.ts
│   │
│   ├── lib/                     # โค้ดกลาง ไม่ผูก UI
│   │   ├── apiClient.ts         # axios/fetch instance + interceptor (แนบ session)
│   │   └── queryClient.ts       # ตั้งค่า TanStack Query (ถ้าใช้)
│   │
│   ├── utils/                   # ฟังก์ชัน pure ช่วยทั่วไป (format วันที่, เงิน)
│   │
│   ├── types/                   # type ที่ใช้ร่วมทั้งแอป
│   │   └── index.ts
│   │
│   ├── constants/               # ค่าคงที่ (role, route path, สถานะ)
│   │
│   ├── config/                  # อ่าน env, รวม config
│   │   └── env.ts
│   │
│   └── styles/                  # global style, theme, ตัวแปร CSS
│       └── globals.css
│
├── .env                         # ตัวแปร — ต้องขึ้นต้น VITE_ ถึงจะเข้าถึงได้ใน client
├── .env.example                 # template ของ env (commit อันนี้ ไม่ commit .env จริง)
├── .eslintrc.cjs                # กฎ lint
├── .prettierrc                  # กฎ format
├── tsconfig.json                # config TypeScript
├── vite.config.ts               # config Vite (alias, plugin, proxy)
├── package.json
└── README.md
```

---

## หน้าที่ของแต่ละ folder

| Folder | ใส่อะไร | ตัวอย่าง |
|--------|---------|----------|
| `components/ui` | UI ชิ้นเล็กที่ใช้ซ้ำได้ ไม่มี business logic | Button, Input, Badge |
| `components/layout` | โครงหน้าหลัก | Sidebar, Header |
| `features/<name>` | ทุกอย่างของ feature นั้น (UI + hook + api + type) | auth, dashboard, ai-chat |
| `pages` | หน้า 1 หน้าต่อ 1 route — ประกอบจากของอื่น | DashboardPage |
| `routes` | map path → page + ป้องกัน route ที่ต้อง login | `/dashboard` → DashboardPage |
| `hooks` | custom hook ที่ข้าม feature | useDebounce, useMediaQuery |
| `lib` | client/instance ที่ตั้งค่าครั้งเดียวใช้ทั้งแอป | apiClient, queryClient |
| `utils` | ฟังก์ชัน pure ไม่มี side effect | formatCurrency, formatDate |
| `types` | type กลาง | ApiResponse, Pagination |
| `config` | อ่านและ validate env | env.ts |

---

## กฎการตั้งชื่อ (Naming Conventions)

- **Component / Page** → `PascalCase` + `.tsx` — เช่น `UserCard.tsx`, `LoginPage.tsx`
- **Hook** → `camelCase` ขึ้นต้น `use` — เช่น `useAuth.ts`
- **Util / config / api** → `camelCase` + `.ts` — เช่น `formatDate.ts`, `apiClient.ts`
- **Type / Interface** → `PascalCase` — เช่น `type User = {...}`
- **Constant** → `UPPER_SNAKE_CASE` — เช่น `const MAX_RETRY = 3`
- **Folder** → `kebab-case` หรือ `camelCase` (เลือกแบบเดียวทั้งทีม) — เช่น `ai-chat/`

---

## Import Alias (สำคัญ — ตั้งครั้งเดียวใช้ทั้งทีม)

แทนที่จะ import ด้วย path ยาวๆ `../../../components/ui/Button`
ให้ตั้ง alias `@/` ชี้ไปที่ `src/`

**`vite.config.ts`**
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
})
```

**`tsconfig.json`**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

จากนั้น import สวยๆ ได้เลย:
```ts
import { Button } from '@/components/ui/Button'
import { useAuth } from '@/features/auth'
```

---

## กฎตัดสินใจ: ของชิ้นนี้วางตรงไหน?

1. **ใช้แค่ feature เดียวไหม?** → ใช่ → ไว้ใน `features/<name>/`
2. **ใช้หลาย feature ไหม?** → ใช่ → ยกขึ้นชั้นบน (`components/`, `hooks/`, `utils/`)
3. **เป็น UI หรือ logic?** → UI → `components/` | logic → `lib/` หรือ `utils/`
4. **เป็นหน้า (route) ไหม?** → ใช่ → `pages/` + ลงทะเบียนใน `routes/`

> เริ่มไว้ใน feature ก่อนเสมอ พอมีคนที่ 2 ต้องใช้ค่อยยกขึ้นชั้นบน — ป้องกันการ over-engineer ตั้งแต่แรก

---

## หมายเหตุสำหรับบริบทนี้ (ธ.ก.ส. backoffice)

- **Auth (OIDC):** หน้าบ้าน **ไม่จัดการ OIDC เอง** — แค่เรียก endpoint `/login`, `/callback`, `/logout` ของ backend ผ่าน `features/auth/api/` ส่วน session อยู่ใน httpOnly cookie ที่ backend ออกให้
- **AI chat:** ยิงผ่าน backend เสมอ (`features/ai-chat/api/`) ไม่ต่อ Python AI service ตรงๆ
- **apiClient:** ตั้ง interceptor ใน `lib/apiClient.ts` ให้แนบ cookie อัตโนมัติ + จัดการ 401 (เด้งไป login)
- **env:** ค่าที่ client ต้องเห็นต้องขึ้นต้น `VITE_` เท่านั้น (เช่น `VITE_API_BASE_URL`) — **ห้ามใส่ secret ใดๆ** เพราะ client อ่านได้หมด

---

## เครื่องมือเสริมที่แนะนำให้เป็นมาตรฐานทีม

- **ESLint + Prettier** — บังคับ style เดียวกันทั้งทีม
- **React Router** — สำหรับ routing
- **TanStack Query** — จัดการ data fetching/cache (ลดโค้ด useEffect ซ้ำซ้อน)
- **Zod** — validate ข้อมูลจาก API ให้ตรง type จริง
- **Husky + lint-staged** — รัน lint/format อัตโนมัติก่อน commit

---
# สร้างโปรเจกต์ใหม่ 

*เริ่มโปรเจกต์:* `npm create vite@latest my-app -- --template react-ts`