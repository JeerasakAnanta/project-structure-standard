# AI Service Project Structure & Best Practices Standard

**Stack:** Python + FastAPI + AI
**ใช้สำหรับ:** มาตรฐานโครงสร้าง AI service ของทีม เรา

> FastAPI ไม่มีโครงสร้างทางการบังคับ เอกสารนี้ยึดแนวทาง layered + dependency injection ตาม [fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices) ซึ่งเป็นแนวที่ community ใช้กันแพร่หลาย

---

## หลักการ (Principles)

1. **แยกชั้นชัดเจน** — router (HTTP) → service (logic) → ai (AI pipeline) 
2. **config มาจาก env ผ่าน pydantic Settings** — validate ตอนเริ่ม (12-factor)
3. **โหลด resource หนักครั้งเดียวตอน startup** — model, vector store client ไม่โหลดใหม่ทุก request
4. **ใช้ dependency injection ของ FastAPI** — แชร์ client, ตรวจ auth, ผ่าน `Depends()`
5. **internal-only** — service นี้ไม่เปิด public, ไม่รู้จัก OIDC, แค่ตรวจ internal JWT จาก backend

---

## โครงสร้างเต็ม

```
ai-service/
├── app/
│   ├── main.py                  # สร้าง FastAPI app, mount router, lifespan
│   │
│   ├── core/                    # config, security, logging ส่วนกลาง
│   │   ├── config.py            # pydantic Settings อ่าน + validate env
│   │   ├── security.py          # ตรวจ internal JWT ที่ backend เซ็นมา
│   │   └── logging.py           # ตั้งค่า structured logging
│   │
│   ├── api/                     # ชั้น HTTP (endpoint)
│   │   ├── deps.py              # dependencies ใช้ร่วม (auth, get client)
│   │   └── v1/                  # version ของ API
│   │       ├── router.py        # รวม endpoint ทั้งหมด
│   │       └── endpoints/
│   │           ├── chat.py      # POST /chat — ถาม AI
│   │           ├── ingest.py    # เพิ่มเอกสารเข้า vector store
│   │           └── health.py    # GET /health
│   │
│   ├── schemas/                 # pydantic model สำหรับ request/response
│   │   ├── chat.py              # ChatRequest, ChatResponse
│   │   └── ingest.py
│   │
│   ├── services/                # ⭐ business logic (ไม่รู้จัก HTTP)
│   │   ├── rag_service.py       # orchestrate: retrieve → prompt → LLM
│   │   └── ingest_service.py    # แตกเอกสาร → embed → เก็บลง Qdrant
│   │
│   ├── rag/                     # ส่วน AI โดยเฉพาะ
│   │   ├── embeddings.py        # embedding model
│   │   ├── vectorstore.py       # Qdrant client + retriever
│   │   ├── chains.py            # LangChain chain
│   │   └── prompts.py           # prompt template (แยกออกมาแก้ง่าย)
│   │
│   └── utils/                   # helper ทั่วไป
│       └── text.py              # chunk เอกสาร, ทำความสะอาด text
│
├── tests/                       # โครงสร้างเลียนแบบ app/
├── .env
├── .env.example                 # commit อันนี้ (ไม่ commit .env จริง)
├── pyproject.toml               # หรือ requirements.txt
├── Dockerfile                   # non-root user, multi-stage
├── .dockerignore
└── README.md
```

> **เมื่อโตขึ้น** เปลี่ยนเป็น domain/module-based ได้ (เช่น `app/chat/` ที่รวม router+service+schema ของ chat ไว้ด้วยกัน) แต่เริ่มด้วย layered ก่อนได้

---

## หน้าที่ของแต่ละชั้น

| ชั้น | หน้าที่ | ห้ามทำ |
|------|---------|--------|
| `api/endpoints` | รับ request, เรียก service, ส่ง response | ห้ามมี AI logic |
| `services` | orchestrate การทำงาน (retrieve → LLM) | ห้ามแตะ request/response |
| `rag` | คุยกับ embedding, vector store, LangChain | ห้ามมี HTTP logic |
| `schemas` | นิยาม + validate รูปร่างข้อมูล | — |
| `core` | config, security, logging | — |

---

## Best Practices

### 1. Config ผ่าน pydantic Settings (validate ตอนเริ่ม)
```python
# core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    QDRANT_URL: str
    QDRANT_API_KEY: str
    LLM_API_KEY: str
    INTERNAL_JWT_SECRET: str        # ใช้ตรวจ token จาก backend
    LLM_MODEL: str = "gpt-4o-mini"

    class Config:
        env_file = ".env"

settings = Settings()   # ขาด env ตัวไหน → crash ทันทีตอนเริ่ม
```

### 2. โหลด resource หนักครั้งเดียวด้วย lifespan
```python
# main.py — โหลด vector store client ตอน startup ไม่ใช่ทุก request
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.vectorstore = init_qdrant()   # ครั้งเดียว
    yield
    # cleanup ตอนปิด

app = FastAPI(lifespan=lifespan)
app.include_router(api_router, prefix="/api/v1")
```

### 3. ตรวจ internal JWT (ไม่ทำ OIDC เอง)
```python
# api/deps.py
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer
import jwt

bearer = HTTPBearer()

def verify_internal_token(cred = Depends(bearer)):
    try:
        payload = jwt.decode(cred.credentials,
                             settings.INTERNAL_JWT_SECRET,
                             algorithms=["HS256"])
        return payload   # มี userId ที่ backend ใส่มา
    except jwt.PyJWTError:
        raise HTTPException(401, "invalid internal token")
```
แล้วป้องกัน endpoint ด้วย `Depends(verify_internal_token)` — service เชื่อ token นี้ ไม่ต้องคุยกับ IdP

### 4. Streaming response สำหรับ LLM
LLM ตอบช้า อย่าให้ผู้ใช้รอค้าง — ใช้ `StreamingResponse` หรือ SSE ส่ง token ทีละชิ้น

### 5. Validation
- pydantic schema ตรวจ request อัตโนมัติ (FastAPI ทำให้)
- ตรวจขนาด input (จำกัดความยาว question, ขนาดไฟล์ ingest) กัน abuse

### 6. Logging
- structured logging (เช่น `structlog`) ไม่ใช่ `print`
- **ห้าม log เนื้อหา prompt/คำตอบที่มีข้อมูลอ่อนไหว** หรือ API key
- log latency ของ retrieve กับ LLM แยกกัน เพื่อหาคอขวด

### 7. Docker (ตามที่ทีมใช้)
- multi-stage build, รันด้วย **non-root user**
- ใส่ `.dockerignore` (ตัด `.env`, `__pycache__`, `tests`)
- ไม่ฝัง secret ใน image — รับผ่าน env ตอน runtime
- health check endpoint ให้ orchestrator เช็ก

---

## หมายเหตุสำหรับบริบทนี้ (ธ.ก.ส.)

- service นี้ **อยู่หลัง backend** ไม่เปิด public — frontend ไม่ยิงตรงมา
- รับ request จาก Express/FastAPI backend เท่านั้น พร้อม internal JWT
- งาน RAG: เอกสาร ธ.ก.ส. → chunk → embed → เก็บใน Qdrant → ตอนถามดึง context กลับมาใส่ prompt
- แยก prompt ไว้ใน `rag/prompts.py` — แก้ทีหลังง่าย ไม่ต้องไปยุ่ง logic
- ระวังข้อมูลลูกค้า: filter/mask ข้อมูลอ่อนไหวก่อนส่งเข้า LLM ภายนอก (ถ้าใช้ LLM cloud)

---

## เครื่องมือมาตรฐานทีม

- **FastAPI** — web framework
- **pydantic / pydantic-settings** — validate ทั้ง schema และ config
- **LangChain** — orchestrate RAG
- **Qdrant** — vector store
- **PyJWT** — ตรวจ internal token
- **structlog** — structured logging
- **ruff + black** — lint + format
- **pytest + httpx** — test
- **uv / poetry** — จัดการ dependency

---

## อ้างอิง

- FastAPI docs: https://fastapi.tiangolo.com
- fastapi-best-practices: https://github.com/zhanymkanov/fastapi-best-practices
- The Twelve-Factor App: https://12factor.net