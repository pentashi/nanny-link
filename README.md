# NannyLink

Global caregiver and nanny matching platform. Verified profiles, video introductions, and pay-to-unlock contact system.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Database](#database)
- [API Reference](#api-reference)
- [Authentication](#authentication)
- [Video Upload Flow](#video-upload-flow)
- [Payment & Contact Unlock Flow](#payment--contact-unlock-flow)
- [Deployment](#deployment)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

NannyLink connects families with verified caregivers worldwide. Every caregiver profile is manually reviewed before going live. Clients browse verified profiles, watch video introductions, and pay a one-time fee to unlock a caregiver's contact information.

**Core flows:**

- Caregivers sign up, upload a video introduction, and await admin verification
- Clients browse verified profiles and pay to unlock WhatsApp/phone contact
- Admins review pending profiles via a secure dashboard and toggle verification status
- Contact information is permanently unlocked after a single payment — no repeat charges

---

## Architecture

```
┌─────────────────────┐        ┌──────────────────────────┐
│   Next.js Frontend  │◄──────►│   NestJS Backend API     │
│   (Loveable / GCP)  │        │   (Google Cloud Run)     │
└─────────────────────┘        └──────────┬───────────────┘
                                           │
                  ┌────────────────────────┼────────────────────────┐
                  │                        │                        │
       ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌─────────▼──────────┐
       │  Supabase           │  │  Google Cloud       │  │  Stripe / Telr     │
       │  (PostgreSQL)       │  │  Storage (GCS)      │  │  (Payments)        │
       └─────────────────────┘  └─────────────────────┘  └────────────────────┘
```

**Tech stack:**

| Layer | Technology |
|---|---|
| Frontend | Next.js (TypeScript) |
| Backend | NestJS (TypeScript) |
| ORM | Prisma |
| Database | Supabase (PostgreSQL) |
| File Storage | Google Cloud Storage |
| Payments | Stripe (global), Telr / PayTabs (MENA fallback) |
| Deployment | Google Cloud Run |
| Auth | JWT — three isolated secrets (maid / client / admin) |

---

## Repository Structure

```
nannylink/
├── frontend/                   # Next.js application
│   ├── app/                    # App router pages and layouts
│   ├── components/             # Shared UI components
│   ├── lib/                    # API client, utilities
│   └── public/                 # Static assets
│
├── backend/                    # NestJS API
│   ├── prisma/
│   │   ├── schema.prisma       # Database models
│   │   └── seed.ts             # Admin user seeder
│   ├── src/
│   │   ├── auth/               # JWT strategies, guards, login/register
│   │   ├── maids/              # Maid profile endpoints
│   │   ├── clients/            # Client profile endpoints
│   │   ├── admin/              # Admin dashboard endpoints
│   │   ├── payments/           # Stripe unlock flow + webhook
│   │   ├── storage/            # GCS signed URL generation
│   │   ├── prisma/             # Prisma module and service
│   │   ├── app.module.ts
│   │   └── main.ts             # CORS, cookie config, bootstrap
│   ├── Dockerfile
│   └── .env.example
│
├── .agents.md                  # AI agent context and rules
├── LICENSE
└── README.md
```

---

## Prerequisites

- Node.js >= 20
- npm >= 10
- Docker (for local containerized runs)
- Google Cloud SDK (`gcloud` CLI)
- A Supabase project with a PostgreSQL connection string
- A Stripe account (test mode for development)

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/pentashi/nannylink.git
cd nannylink
```

### 2. Install dependencies

```bash
# Backend
cd backend
npm install

# Frontend
cd ../frontend
npm install
```

### 3. Configure environment variables

```bash
cd backend
cp .env.example .env
# Fill in all values — see Environment Variables section below
```

### 4. Set up the database

```bash
cd backend
npx prisma migrate dev --name init
npx prisma generate
npx prisma db seed
```

### 5. Authenticate with Google Cloud (local development)

```bash
gcloud auth application-default login
```

No service account key file is used. The project enforces
`constraints/iam.disableServiceAccountKeyCreation`. Local development
uses Application Default Credentials (ADC). Cloud Run uses Workload Identity.

### 6. Start development servers

```bash
# Backend (from /backend)
npm run start:dev

# Frontend (from /frontend)
npm run dev
```

Backend runs on `http://localhost:8080`
Frontend runs on `http://localhost:3000`

---

## Environment Variables

All environment variables are required unless marked optional.

```env
# Database
DATABASE_URL=postgresql://...              # Supabase connection string

# JWT — three separate secrets, never share across roles
JWT_SECRET_MAID=
JWT_SECRET_CLIENT=
JWT_SECRET_ADMIN=

# Google Cloud Storage
GCS_BUCKET_NAME=nannylink-videos
GCP_PROJECT_ID=

# Payments
PAYMENT_WEBHOOK_SECRET=                   # Stripe webhook signing secret
STRIPE_SECRET_KEY=

# CORS
CORS_ORIGIN=https://your-frontend-url     # No trailing slash

# Admin seed (used by prisma/seed.ts only)
ADMIN_EMAIL=
ADMIN_PASSWORD=

# Server
PORT=8080
```

> Never commit `.env` to version control. The `.gitignore` excludes it by default.

---

## Database

### Models

| Model | Description |
|---|---|
| `Maid` | Caregiver profile — includes `isVerified`, `videoUrl`, `whatsappNumber` |
| `Client` | Parent/employer account |
| `UnlockRequest` | Records a paid contact unlock — unique per `clientId + maidId` pair |
| `Admin` | Platform administrator — seeded only, no self-registration |

### Migrations

```bash
# Create a new migration after schema changes
npx prisma migrate dev --name describe_your_change

# Apply migrations in production
npx prisma migrate deploy

# Open Prisma Studio (local DB browser)
npx prisma studio
```

### Seeding

The seed script creates the first admin account from environment variables.
Running it twice is safe — it uses upsert.

```bash
npx prisma db seed
```

---

## API Reference

All protected routes require a `Bearer` token in the `Authorization` header.
The token type must match the route's guard — maid tokens cannot access client
routes and vice versa.

### Auth

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| POST | `/auth/maid/register` | None | Register a new maid account |
| POST | `/auth/maid/login` | None | Maid login — returns JWT |
| POST | `/auth/client/register` | None | Register a new client account |
| POST | `/auth/client/login` | None | Client login — returns JWT |
| POST | `/auth/admin/login` | None | Admin login — returns JWT |

### Maids

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| GET | `/maids` | None | Browse all verified maid profiles |
| GET | `/maids/:id` | None | Single maid profile |
| GET | `/maids/:id/contact` | JwtClient | Returns WhatsApp number if unlocked |
| POST | `/maids/video-upload-url` | JwtMaid | Generate GCS signed URL for video upload |
| PATCH | `/maids/profile` | JwtMaid | Maid updates own profile |

### Admin

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| GET | `/admin/pending` | JwtAdmin | All maids pending review |
| PATCH | `/admin/maids/:id/verify` | JwtAdmin | Approve a maid profile |
| PATCH | `/admin/maids/:id/suspend` | JwtAdmin | Suspend a live profile |
| GET | `/admin/maids` | JwtAdmin | Searchable list of all maids |

### Payments

| Method | Endpoint | Guard | Description |
|---|---|---|---|
| POST | `/payments/unlock` | JwtClient | Initiate contact unlock payment |
| POST | `/payments/webhook` | None (signature verified) | Stripe webhook receiver |

---

## Authentication

NannyLink uses three completely isolated JWT contexts. Each has its own
secret and its own Passport strategy. A token issued for one role will be
rejected by any other role's guard.

```
JWT_SECRET_MAID    →  JwtMaidStrategy    →  JwtMaidGuard
JWT_SECRET_CLIENT  →  JwtClientStrategy  →  JwtClientGuard
JWT_SECRET_ADMIN   →  JwtAdminStrategy   →  JwtAdminGuard
```

**Data visibility rules:**

`passwordHash` is stripped at the service layer and never returned by any
endpoint under any circumstance.

`whatsappNumber` is stripped from all responses except `GET /maids/:id/contact`,
and only when a valid `UnlockRequest` row exists for the requesting client.

---

## Video Upload Flow

Videos are uploaded directly from the browser to GCS. They never pass
through the backend server.

```
1. Maid calls    POST /maids/video-upload-url
                 ↓
2. Backend generates a GCS signed URL (v4, PUT, 15 min expiry)
   using Workload Identity — no key file involved
                 ↓
3. Frontend uploads video directly to GCS using the signed URL
                 ↓
4. Frontend calls PATCH /maids/profile with the resulting GCS public URL
                 ↓
5. Backend stores the URL string in Maid.videoUrl
```

This pattern keeps video bandwidth off the backend, reduces Cloud Run costs,
and removes file size constraints from the API layer.

---

## Payment & Contact Unlock Flow

```
1. Client calls  POST /payments/unlock  { maidId }
                 ↓
2. Backend checks UnlockRequest table for clientId + maidId pair
   → Row exists:  return whatsappNumber immediately (no charge)
   → No row:      create Stripe payment session, return checkout URL
                 ↓
3. Client completes payment on Stripe
                 ↓
4. Stripe fires  POST /payments/webhook
                 ↓
5. Backend verifies webhook signature (rejects if invalid)
   On payment.succeeded: INSERT INTO UnlockRequest
                 ↓
6. Client returns to maid profile — contact is now visible
                 ↓
7. Maid receives email notification
```

The `UnlockRequest` table has a `@@unique([clientId, maidId])` constraint.
A client is never charged twice for the same caregiver.

---

## Deployment

NannyLink backend is deployed to Google Cloud Run. The project enforces
Workload Identity — no service account key files are created or stored.

### Build and push

```bash
cd backend

# Build the image
docker build -t gcr.io/YOUR_PROJECT_ID/nannylink-backend .

# Push to Google Container Registry
docker push gcr.io/YOUR_PROJECT_ID/nannylink-backend
```

### Deploy to Cloud Run

```bash
gcloud run deploy nannylink-backend \
  --image gcr.io/YOUR_PROJECT_ID/nannylink-backend \
  --platform managed \
  --region europe-west1 \
  --allow-unauthenticated \
  --set-env-vars="NODE_ENV=production" \
  --set-secrets="DATABASE_URL=DATABASE_URL:latest,JWT_SECRET_MAID=JWT_SECRET_MAID:latest"
```

All secrets are managed via Google Secret Manager — never passed as plain
environment variables in production.

### Run database migrations on deploy

```bash
npx prisma migrate deploy
```

Run this before or as part of your Cloud Run deployment pipeline, not inside
the container startup.

---

## Contributing

This is a proprietary commercial project. External contributions are not
accepted at this time.

For bug reports or security disclosures, contact the repository owner directly.

**Security vulnerabilities** — do not open a public issue. Email the maintainer
privately with full reproduction steps.

---

## License

Proprietary. Copyright (c) 2026 Mbongwe Brandon Egbe. All rights reserved.

See [LICENSE](./LICENSE) for full terms.
