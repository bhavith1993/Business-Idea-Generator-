# Healthcare Consultation Assistant (Full‑Stack SaaS Demo)

A full‑stack SaaS-style web app that helps clinicians turn free‑text visit notes into:

- **A structured summary** for medical records  
- **Actionable next steps** for the clinician  
- **A patient‑friendly email draft**

The key engineering focus is **secure auth + real‑time streaming AI output + production deployment**.

---



**Full‑stack product engineering**
- Next.js (Pages Router) + TypeScript UI with modern Tailwind styling and Markdown rendering
- Form UX (date picker, required fields) and protected routes

**Security & auth**
- Clerk authentication on the frontend, with **JWT verification on the FastAPI backend** via Clerk JWKS (no “trusting the client”)
- Backend can extract `user_id` from verified tokens for auditing/usage tracking

**Streaming AI**
- Server‑Sent Events (SSE): backend streams tokens as they arrive; frontend renders output progressively for fast perceived latency

**Cloud + DevOps**
- Production deployment on **Vercel** (simple)
- Production deployment on **AWS** using **Docker → ECR → App Runner** (real-world container workflow)
- Health checks + log visibility through AWS tooling (CloudWatch via App Runner)

---

## Architecture

### Runtime flow (request → response)

1. User signs in via **Clerk** in the browser  
2. Browser calls the FastAPI endpoint with an **Authorization** token  
3. FastAPI verifies JWT using **Clerk JWKS**  
4. FastAPI calls the **OpenAI API** and streams tokens back over SSE  
5. UI renders the streamed response in real time (Markdown)

### Deployment options

#### Option A — Vercel (fastest demo)
```
Browser
  ↓
Vercel (Next.js UI)
  ↓  (JWT in Authorization header)
Vercel Python / FastAPI endpoint (/api)
  ↓
OpenAI API (stream)
  ↑
SSE stream back to browser
```

#### Option B — AWS App Runner (single-container production pattern)
For AWS simplicity, Next.js is exported as static files and served by FastAPI inside one container.

```
Browser
  ↓
AWS App Runner (Docker container)
  ├─ serves Next.js static export (/)
  └─ serves FastAPI API (/api/consultation, /health)
          ↓
       OpenAI API (stream)
          ↑
       SSE back to browser
```

---

## Key features

- **Healthcare-specific prompt + structured output** (3 fixed sections)
- **Streaming responses** using SSE (`text/event-stream`)
- **Clerk-authenticated access** (frontend route protection + backend JWT verification)
- **Pydantic validation** for backend request payloads
- **Health check endpoint** (`/health`) for App Runner monitoring
- **Dockerized production build** (multi-stage: Node build → Python runtime)

---

## Tech stack

**Frontend**
- Next.js (Pages Router) + TypeScript
- Tailwind CSS
- React Markdown rendering
- Streaming client (`EventSource` / `fetch-event-source`)

**Backend**
- FastAPI + Uvicorn
- Pydantic models for validation
- `fastapi-clerk-auth` for JWT verification (JWKS)
- OpenAI SDK (streaming)

**Cloud / DevOps**
- Vercel (quick deployment)
- Docker (multi-stage build)
- AWS ECR (image registry)
- AWS App Runner (container hosting)
- CloudWatch logs (via App Runner)

---

## Project structure (typical)

```
saas/
├── pages/                  # Next.js Pages Router
├── styles/                 # CSS styles
├── api/                    # FastAPI backend (index.py / server.py)
├── public/                 # Static assets
├── package.json
├── requirements.txt
├── next.config.ts
└── tsconfig.json
```

---

## Environment variables

Create `.env.local` (Vercel/local) or `.env` (Docker/AWS) and set:

- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- `CLERK_SECRET_KEY`
- `CLERK_JWKS_URL`
- `OPENAI_API_KEY`

For AWS workflows (optional convenience):

- `DEFAULT_AWS_REGION`
- `AWS_ACCOUNT_ID`

> Do **not** commit `.env*` files.

---

## Run locally (Docker: closest to production)

1) Build the container:
```bash
docker build \
  --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
  -t consultation-app .
```

2) Run:
```bash
docker run -p 8000:8000 \
  -e CLERK_SECRET_KEY="$CLERK_SECRET_KEY" \
  -e CLERK_JWKS_URL="$CLERK_JWKS_URL" \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  consultation-app
```

3) Open:
- `http://localhost:8000`

---

## Deploy

### Vercel
```bash
vercel --prod
```

### AWS App Runner (high-level)
1. Build Docker image  
2. Push to **ECR**  
3. Create an **App Runner** service from the ECR image  
4. Add env vars in App Runner (Clerk + OpenAI)  
5. Configure health check path: `/health`

---

## Notes on security & compliance (read this)

- This is a **demo app** meant to showcase engineering patterns (auth, streaming, deployment).  
- Do **not** treat it as HIPAA compliant or production-ready for real patient data without additional safeguards (audit logging, encryption, data retention policy, threat modeling, BAA, etc.).  
- The UI may contain “HIPAA compliant” messaging as a design cue—treat that as **placeholder**, not a certification.

---

## Potential extensions

- Persist consultations to a database (per `user_id`)
- Usage limits / metering per subscription tier
- Background job queue for longer workflows
- Observability: request tracing + structured logging + metrics + alarms
