# Fisio MSK — Clinical Management App

A full-stack web application for musculoskeletal physiotherapists to manage patients, clinical assessments, rehabilitation plans, treatment sessions and functional progress tracking.

Built as a real-world project to solve genuine documentation needs in clinical MSK practice.

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Features](#features)
- [Clinical Domain](#clinical-domain)
- [Security](#security)
- [Testing](#testing)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Key Technical Decisions](#key-technical-decisions)

---

## Overview

Fisio MSK is a clinical records system designed around the actual workflow of a musculoskeletal physiotherapist:

**Patient intake → Initial assessment (anamnesis) → Objective evaluation → Rehabilitation plan → Session-by-session tracking → Progress monitoring → Clinical discharge**

The app enforces clinical data integrity (no physical deletion of records), per-user data isolation, and structured clinical reasoning at each step of the patient journey.

---

## Tech Stack

### Backend
| Technology | Purpose |
|---|---|
| Node.js + Express | REST API server |
| TypeScript | Type safety across the entire backend |
| Prisma ORM | Database access layer with type-safe queries |
| PostgreSQL | Relational database |
| Zod | Request validation and schema definition |
| JWT + cookie-parser | Authentication via httpOnly cookies |
| bcrypt | Password hashing (12 salt rounds) |
| Puppeteer Core | Server-side PDF report generation |
| express-rate-limit | Brute force protection on auth endpoints |

### Frontend
| Technology | Purpose |
|---|---|
| Next.js 15 (App Router) | React framework with server components |
| TypeScript | End-to-end type safety |
| Tailwind CSS | Utility-first styling |
| ShadCN UI | Accessible component primitives |
| React Hook Form + Zod | Client-side form validation |
| Recharts | Progress and metrics visualization |

### Infrastructure
| Technology | Purpose |
|---|---|
| Docker Compose | Local PostgreSQL database |
| npm workspaces | Monorepo management |
| Jest + Supertest | Backend integration testing |

---

## Architecture

The project is organized as a **monorepo** with two applications:

```
fisio-app/
├── apps/
│   ├── backend/          → Express REST API
│   └── frontend/         → Next.js application
├── docker-compose.yml    → Local database
├── CLAUDE.md             → AI agent context file
└── docs/                 → Clinical analysis and roadmap
```

### Backend architecture

The backend follows a strict layered pattern:

```
Request → Route → Controller → Service → Prisma → PostgreSQL
```

Each domain module is self-contained:

```
src/
├── modules/
│   ├── auth/             → Register, login, logout
│   ├── patients/         → Patient CRUD + dashboard summary
│   ├── assessments/      → Initial assessment + objective evaluations
│   ├── sessions/         → Treatment sessions with exercises
│   ├── rehabPlans/       → Rehabilitation plans and clinical goals
│   ├── exercises/        → Exercise library
│   └── progressMetrics/  → Quantitative progress tracking
├── middlewares/
│   ├── require-auth.ts   → JWT verification (cookie + header fallback)
│   ├── error-handler.ts  → Centralized error responses
│   └── not-found.ts      → 404 handler
└── lib/
    ├── jwt.ts            → Single source of JWT secret
    ├── pagination.ts     → Reusable pagination with Zod coercion
    └── async-handler.ts  → Async error wrapper for controllers
```

### Frontend architecture

```
src/
├── app/                  → Next.js App Router pages
├── components/
│   ├── patients/         → All patient-related UI components
│   ├── layout/           → Sidebar, navbar, responsive drawer
│   └── ui/               → Shared UI primitives
├── hooks/
│   └── use-mounted.ts    → SSR hydration helper
└── lib/
    ├── api.ts            → Base fetch client with auth + error handling
    ├── patients-api.ts   → All API calls typed end-to-end
    └── schemas.ts        → Shared Zod schemas for forms
```

---

## Features

### Patient management
- Full CRUD with soft delete (clinical records are never physically removed)
- Search by name or document number with 300ms debounce
- Paginated listing (20 per page)
- Dashboard with recent patients and activity summary

### Clinical assessment
- **Initial assessment (anamnesis):** Chief complaint, pain behavior (NRS 0–10, morning/daytime/night patterns), aggravating/relieving factors, neurological symptoms, red flags, yellow flags (psychosocial factors), irritability level, functional demand, medications and surgical history
- **Objective assessment:** Posture, range of motion, strength, special tests, palpation, movement quality, balance, gait — with multiple assessments over time to track evolution

### Rehabilitation planning
- Phase-based plans (Acute → Subacute → Strengthening → Return to Activity → Maintenance)
- Weekly frequency tracking
- Clinical goals with priority levels (High/Medium/Low) and target dates
- Goal status progression (Active → Achieved → Paused → Cancelled)

### Session tracking
- Pain before/after (NRS 0–10)
- Subjective report, clinical notes, response to treatment
- Structured exercise prescription: sets, reps, duration, load, RPE (Rate of Perceived Exertion)
- Clinical decision at session end: Maintain / Progress / Regress / Refer

### Progress monitoring
- Quantitative metrics: Pain (NRS), ROM (degrees), Strength (kg), Function (%), Balance, Adherence
- Line chart visualization across all sessions
- Metrics linked to specific sessions for context

### PDF report generation
- Server-side PDF via Puppeteer Core
- Full patient summary: demographics, assessment findings, treatment history, progress metrics
- Direct browser download (no popup dependency)

### Responsive navigation
- Desktop: persistent sidebar
- Below 900px: collapsible drawer with overlay
- Hamburger menu with smooth transition

---

## Clinical Domain

The data model reflects real MSK physiotherapy practice:

```
Patient
├── InitialAssessment (1:1)     → Anamnesis with 24h pain behavior,
│                                  irritability, yellow flags
├── ObjectiveAssessment (1:N)   → Multiple evaluations over time
├── RehabPlan (1:N)             → Phase-based rehabilitation planning
├── ClinicalGoal (1:N)          → Prioritized functional goals
├── Session (1:N)               → Per-session records with exercises
│   └── SessionExercise (1:N)   → Exercise prescription with RPE
└── ProgressMetric (1:N)        → Quantitative outcome tracking
```

Key clinical decisions reflected in the schema:
- `ClinicalDecision` enum (MAINTAIN / PROGRESS / REGRESS / REFER) enforces explicit clinical reasoning at each session
- `IrritabilityLevel` (LOW / MEDIUM / HIGH) guides treatment intensity
- `RedFlags` and `YellowFlags` fields surface safety and psychosocial screening
- Soft delete on patients and sessions preserves clinical record integrity

---

## Security

| Layer | Implementation |
|---|---|
| Authentication | JWT stored in `httpOnly` cookie (not localStorage) |
| Cookie flags | `httpOnly: true`, `sameSite: strict`, `secure: true` in production |
| Authorization | Every resource query verifies ownership by `userId` |
| Password storage | bcrypt with 12 salt rounds |
| Rate limiting | 20 requests per 15 minutes on `/auth` endpoints |
| CORS | Explicit origin whitelist via `ALLOWED_ORIGIN` env variable |
| Input validation | Zod schemas on every POST/PATCH endpoint |
| Token fallback | `require-auth` middleware reads cookie first, Authorization header as fallback for testing |

---

## Testing

The backend has **61 integration tests** across 6 test suites:

```
app.test.ts              → Auth flows, route protection, cross-user access
assessments.test.ts      → Initial and objective assessment CRUD
rehab-plans.test.ts      → Rehab plan lifecycle
clinical-goals.test.ts   → Goal management
progress-metrics.test.ts → Metrics recording
exercises.test.ts        → Exercise library
```

Each suite covers:
- Happy path (successful operation)
- 401 — missing or invalid token (via cookie and Authorization header)
- 403/404 — accessing another user's resource

Tests use **Prisma mocks** (no real database required) via `jest.mock` and a shared `prismaMock` factory, allowing the full test suite to run in isolation.

```bash
cd apps/backend && npm test
# 61 passing
```

---

## Getting Started

### Prerequisites
- Node.js 18+
- Docker and Docker Compose
- Chrome or Chromium (for PDF generation)

### Installation

```bash
# Clone the repository
git clone https://github.com/Mrcipo/fisio-app.git
cd fisio-app

# Install dependencies
npm install

# Start the database
docker-compose up -d

# Set up environment variables
cp apps/backend/.env.example apps/backend/.env
# Edit .env with your values

# Run database migrations
cd apps/backend && npx prisma migrate dev

# Seed demo data
npx tsx src/scripts/seed.ts

# Start both servers
cd ../..
npm run dev --workspace=apps/backend   # http://localhost:4000
npm run dev --workspace=apps/frontend  # http://localhost:3000
```

### Demo credentials
```
Email:    demo@fisio.app
Password: demo12345
```

### Environment variables

```env
# apps/backend/.env
DATABASE_URL="postgresql://..."
JWT_SECRET="your-secret-key"
ALLOWED_ORIGIN="http://localhost:3000"
NODE_ENV="development"          # Set to "production" for secure cookies
CHROME_EXECUTABLE_PATH=""       # Optional: path to Chrome for PDF generation
                                # Windows: C:\Program Files\Google\Chrome\Application\chrome.exe
                                # macOS:   /Applications/Google Chrome.app/Contents/MacOS/Google Chrome
                                # Linux:   /usr/bin/google-chrome
```

---

## Project Structure

```
apps/backend/
├── prisma/
│   ├── schema.prisma         → Full data model with clinical enums
│   └── migrations/           → Database migration history
├── src/
│   ├── modules/              → Domain modules (auth, patients, sessions...)
│   ├── middlewares/          → Auth, error handling, not-found
│   ├── lib/                  → Shared utilities (jwt, pagination, errors)
│   ├── test/                 → Integration tests + Prisma mocks
│   └── scripts/seed.ts       → Demo data seeder

apps/frontend/
├── src/
│   ├── app/                  → Next.js App Router (pages and layouts)
│   ├── components/
│   │   ├── patients/         → Patient detail, sessions, assessments, goals
│   │   ├── layout/           → Sidebar, navbar, drawer navigation
│   │   └── ui/               → Button, Card, Input, Select, Tabs...
│   ├── hooks/use-mounted.ts  → Hydration-safe hook
│   └── lib/
│       ├── api.ts            → Fetch client (credentials, error translation, 401 redirect)
│       ├── patients-api.ts   → Typed API layer (all endpoints)
│       └── schemas.ts        → Shared Zod validation schemas
```

---

## Key Technical Decisions

**JWT in httpOnly cookie instead of localStorage**  
Eliminates XSS token theft. The cookie is invisible to JavaScript and sent automatically with every request. The `require-auth` middleware reads `req.cookies.token` with a fallback to the `Authorization` header for test compatibility.

**Soft delete over hard delete**  
Clinical records must never be permanently removed — this is both a legal and ethical requirement in healthcare. Patients and sessions have a `deletedAt` field; all queries filter `WHERE deletedAt IS NULL`. A physical delete would violate clinical record-keeping standards.

**Prisma `$transaction` for paginated lists**  
Every paginated endpoint fetches data and total count in a single database round-trip using `prisma.$transaction([findMany, count])`. This prevents the count from drifting out of sync with the data in concurrent environments.

**Module-scoped user ownership**  
Every resource query includes `userId` from the authenticated session. There is no global admin bypass — a user can only access their own patients, sessions and exercises. This is enforced at the service layer, not the route layer.

**Reusable pagination with Zod coercion**  
`lib/pagination.ts` exports a single Zod schema with `z.coerce.number()` that parses query string params (`?page=1&limit=20`) into typed numbers. Every list endpoint imports this schema, ensuring consistent pagination behavior.

**Debounced search without external libraries**  
Patient search uses native `setTimeout`/`clearTimeout` for 300ms debounce. No extra dependency (lodash, use-debounce) required for a single use case.

---

## Roadmap

The project has an active clinical development roadmap documented in `docs/plan-accion-v3-clinico.md`. Planned features include:

- Structured ROM measurements (numeric, by movement, by region)
- Predefined special tests per body region (Neer, Hawkins, Lachman, SLR...)
- Manual techniques and physical agents in session records
- Validated functional scales (DASH, NDI, Oswestry, KOOS, LEFS)
- Clinical discharge module
- Treatment episode grouping
- Multi-metric progress charts

---

