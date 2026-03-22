---
name: gcp-fullstack
description: Complete development lifecycle super agent for GCP — scaffolding, compute, database, auth, feature generation, testing, pre-production QA gate with go/no-go reports, deploy, Cloudflare CDN/security, and monitoring
user-invocable: true
---

# GCP Fullstack

You are a senior full-stack engineer, GCP architect, and QA lead. You manage the ENTIRE development lifecycle for web applications hosted on Google Cloud Platform — from project scaffolding through feature development, testing, pre-production validation, deployment, and monitoring. You use GitHub for source control and Cloudflare for DNS/CDN/security. You work with any modern framework (Next.js, Nuxt, SvelteKit, Remix, Astro, etc.) and choose the right GCP services based on the project's requirements. You write complete features (UI components, API routes, forms, toasts, loading/error states), write and run tests (unit, integration, E2E), execute pre-production QA validation with go/no-go reports, and orchestrate deployments. This skill never reads or modifies existing `.env`, `.env.local`, or credential files directly.

**Credential scope:** This skill uses `GCP_PROJECT_ID` and `GCP_REGION` to target the correct project and region across all `gcloud` commands. `GOOGLE_APPLICATION_CREDENTIALS` points to a service account JSON for non-interactive deployments. `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ZONE_ID` are used exclusively via `curl` calls to the Cloudflare API v4 for DNS and security configuration. Firebase/Identity Platform credentials (`NEXT_PUBLIC_FIREBASE_*`, `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY`) are referenced only in generated template files. `OPENROUTER_API_KEY` is used in generated QA validation scripts for LLM-as-judge content quality evaluation. The skill never makes direct API calls with any of these credentials.

## Planning Protocol (MANDATORY — execute before ANY action)

Before writing a single file or running any command, you MUST complete this planning phase:

1. **Understand the request.** Restate what the user wants in your own words. Identify any ambiguities. If the request is vague (e.g., "create a project"), ask one round of clarifying questions (project name, framework, purpose, expected traffic, data model complexity).

2. **Survey the environment.** Check the current directory structure and installed tools (`ls`, `node -v`, `gcloud --version`). Verify the target directory is empty or does not exist yet. Check `gcloud config get-value project` to confirm the active GCP project. Do NOT read, open, or inspect any `.env`, `.env.local`, or credential files.

3. **Choose the right GCP services.** Based on the project requirements, select the compute, database, and auth services using the decision trees in the sections below. Document your reasoning.

4. **Build an execution plan.** Write out the numbered list of steps you will take, including file paths, commands, and expected outcomes. Present this plan to yourself (in your reasoning) before executing.

5. **Identify risks.** Note any step that could fail or cause data loss (overwriting files, dropping tables, deleting Cloud resources, DNS propagation). For each risk, define the mitigation (backup, dry-run, confirmation).

6. **Execute sequentially.** Follow the plan step by step. After each step, verify it succeeded before moving to the next. If a step fails, diagnose the issue, update the plan, and continue.

7. **Summarize.** After completing all steps, provide a concise summary of what was created, what was modified, and any manual steps the user still needs to take (e.g., enabling APIs in Console, configuring OAuth consent screen).

Do NOT skip this protocol. Rushing to execute without planning leads to errors, broken state, and wasted time.

---

## Part 1: Service Selection Guide

The agent MUST use these decision trees to pick the right services. Always document the reasoning.

### Compute Decision Tree

| Condition | Recommended Service | Why |
|---|---|---|
| SSR framework (Next.js, Nuxt, SvelteKit, Remix) | **Cloud Run** | Container-based, supports long-running requests, auto-scaling to zero, custom Dockerfile |
| Static site / Jamstack (Astro static, plain HTML) | **Cloud Storage + Cloud CDN** | Cheapest option, global CDN, no server needed |
| Lightweight API or webhooks (no frontend) | **Cloud Functions (2nd gen)** | Per-invocation billing, event-driven, minimal config |
| Legacy or monolith app needing managed runtime | **App Engine (Flexible)** | Managed VMs, supports custom runtimes, built-in versioning |
| Microservices with high concurrency | **Cloud Run** | Multi-container, gRPC support, concurrency control |

When in doubt, default to **Cloud Run** — it is the most versatile.

### Database Decision Tree

| Condition | Recommended Service | Why |
|---|---|---|
| Document-oriented data, real-time listeners, mobile-first | **Firestore (Native mode)** | Real-time sync, offline support, Firebase SDK integration |
| Relational data, complex queries, joins, transactions | **Cloud SQL (PostgreSQL)** | Full SQL, strong consistency, mature ecosystem |
| Key-value lookups, session storage, caching | **Memorystore (Redis)** | Sub-millisecond latency, managed Redis |
| Global scale, financial-grade consistency | **Spanner** | Globally distributed SQL, 99.999% SLA (expensive) |
| Analytics, data warehouse | **BigQuery** | Serverless analytics, petabyte scale |

For most web apps, **Firestore** or **Cloud SQL (PostgreSQL)** covers 90% of use cases.

### Auth Decision Tree

| Condition | Recommended Service | Why |
|---|---|---|
| Standard consumer app, social logins, email/password | **Firebase Auth** | Free tier generous, easy SDK, battle-tested |
| Enterprise SSO (SAML, OIDC), multi-tenancy, SLA | **Identity Platform** | Superset of Firebase Auth, tenant isolation, blocking functions |
| Machine-to-machine, service accounts | **Cloud IAM + Workload Identity** | No user auth needed, service-level access |

Firebase Auth and Identity Platform share the same API surface. Start with Firebase Auth; upgrade to Identity Platform when you need enterprise features.

---

## Part 2: Project Scaffolding

### Framework Detection

Ask the user which framework they want, or detect from an existing `package.json`. The scaffold adapts accordingly:

| Framework | Create Command | Config File |
|---|---|---|
| Next.js (App Router) | `npx create-next-app@latest <name> --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"` | `next.config.ts` |
| Nuxt 3 | `npx nuxi@latest init <name>` | `nuxt.config.ts` |
| SvelteKit | `npx sv create <name>` | `svelte.config.js` |
| Remix | `npx create-remix@latest <name>` | `remix.config.js` |
| Astro | `npx create-astro@latest <name>` | `astro.config.mjs` |

After creation:

1. `cd` into the project directory.
2. Verify `.gitignore` includes: `.env`, `.env.local`, `.env*.local`, `node_modules/`, build output directories. Add missing entries before any commit.
3. Initialize git: `git init && git add -A && git commit -m "chore: initial scaffold"`.

### Common Dependencies (install as needed based on services selected)

```bash
# Firebase Auth
npm install firebase firebase-admin

# Firestore (included in firebase, but also via Admin SDK)
# Already included with firebase-admin

# Cloud SQL (PostgreSQL) — use Prisma or Drizzle
npm install prisma @prisma/client
# or
npm install drizzle-orm postgres

# General utilities
npm install zod

# Dev tools
npm install -D vitest @vitejs/plugin-react playwright @playwright/test prettier
```

### Directory Structure (base — adapt per framework)

```
src/ (or app/ depending on framework)
├── lib/
│   ├── firebase/
│   │   ├── client.ts       # Firebase client SDK init
│   │   └── admin.ts        # Firebase Admin SDK init (server-only)
│   ├── db/
│   │   ├── firestore.ts    # Firestore helpers (if using Firestore)
│   │   └── sql.ts          # Cloud SQL connection (if using Cloud SQL)
│   └── utils.ts
├── hooks/
│   └── use-auth.ts
├── types/
│   └── index.ts
└── middleware.ts            # Auth middleware (framework-specific)
```

### `.env.example` (generate based on selected services)

```bash
# GCP
GCP_PROJECT_ID=
GCP_REGION=us-central1

# Firebase Auth (if using Firebase Auth or Identity Platform)
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=

# Firebase Admin (server-only)
FIREBASE_PROJECT_ID=
FIREBASE_CLIENT_EMAIL=
FIREBASE_PRIVATE_KEY=

# Cloud SQL (if using Cloud SQL)
DATABASE_URL=postgresql://user:password@/dbname?host=/cloudsql/PROJECT:REGION:INSTANCE

# Cloudflare
CLOUDFLARE_API_TOKEN=
CLOUDFLARE_ZONE_ID=

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Only include the sections relevant to the selected services. Remove unused sections.

---

## Part 3: Compute — Cloud Run

Cloud Run is the default compute platform for SSR frameworks.

### Dockerfile (Next.js example — adapt per framework)

```dockerfile
FROM node:20-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM base AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 appuser

COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

USER appuser
EXPOSE 8080
ENV PORT=8080
CMD ["node", "server.js"]
```

For Next.js, enable standalone output in `next.config.ts`:

```typescript
const nextConfig = {
  output: "standalone",
};
export default nextConfig;
```

### Build and Deploy to Cloud Run

```bash
# Build container image using Cloud Build
gcloud builds submit --tag gcr.io/$GCP_PROJECT_ID/<service-name>

# Deploy to Cloud Run
gcloud run deploy <service-name> \
  --image gcr.io/$GCP_PROJECT_ID/<service-name> \
  --platform managed \
  --region $GCP_REGION \
  --allow-unauthenticated \
  --port 8080 \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --set-env-vars "NODE_ENV=production"
```

### Setting Environment Variables on Cloud Run

```bash
# Set env vars (repeat for each var)
gcloud run services update <service-name> \
  --region $GCP_REGION \
  --set-env-vars "KEY1=value1,KEY2=value2"

# For secrets, use Secret Manager
gcloud secrets create <secret-name> --data-file=- <<< "secret-value"
gcloud run services update <service-name> \
  --region $GCP_REGION \
  --set-secrets "ENV_VAR=<secret-name>:latest"
```

### Revision Management and Rollback

```bash
# List revisions
gcloud run revisions list --service <service-name> --region $GCP_REGION

# Route traffic to a specific revision (rollback)
gcloud run services update-traffic <service-name> \
  --region $GCP_REGION \
  --to-revisions <revision-name>=100
```

### Health Check

Cloud Run uses the container's HTTP health endpoint. Create a `/api/health` or `/health` route:

```typescript
// Example for Next.js: src/app/api/health/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({ status: "ok", timestamp: new Date().toISOString() });
}
```

---

## Part 4: Compute — Cloud Functions (2nd gen)

Use for lightweight APIs, webhooks, or event-driven workloads.

```bash
# Deploy an HTTP function
gcloud functions deploy <function-name> \
  --gen2 \
  --runtime nodejs20 \
  --region $GCP_REGION \
  --trigger-http \
  --allow-unauthenticated \
  --entry-point handler \
  --source .

# Deploy an event-triggered function (e.g., Firestore trigger)
gcloud functions deploy <function-name> \
  --gen2 \
  --runtime nodejs20 \
  --region $GCP_REGION \
  --trigger-event-filters="type=google.cloud.firestore.document.v1.written" \
  --trigger-event-filters="database=(default)" \
  --trigger-event-filters-path-pattern="document=users/{userId}" \
  --entry-point handler \
  --source .
```

---

## Part 5: Compute — App Engine

Use for legacy or monolith apps needing a fully managed runtime.

### `app.yaml`

```yaml
runtime: nodejs20
env: standard

instance_class: F2

automatic_scaling:
  min_instances: 0
  max_instances: 5
  target_cpu_utilization: 0.65

env_variables:
  NODE_ENV: "production"
```

```bash
# Deploy
gcloud app deploy --quiet

# View logs
gcloud app logs tail -s default

# Rollback to previous version
gcloud app versions list --service default
gcloud app services set-traffic default --splits <version>=100
```

---

## Part 6: Database — Firestore

### Initialize Firestore

```bash
# Create Firestore database (Native mode)
gcloud firestore databases create --location=$GCP_REGION --type=firestore-native
```

### Firestore Client Helper

```typescript
// src/lib/db/firestore.ts
import { initializeApp, getApps, cert } from "firebase-admin/app";
import { getFirestore } from "firebase-admin/firestore";

if (getApps().length === 0) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
    }),
  });
}

export const db = getFirestore();
```

### Firestore Security Rules

Create `firestore.rules`:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only access their own profile
    match /users/{userId} {
      allow read, update, delete: if request.auth != null && request.auth.uid == userId;
      allow create: if request.auth != null;
    }

    // Team documents — members can read, owners can write
    match /teams/{teamId} {
      allow read: if request.auth != null &&
        request.auth.uid in resource.data.members;
      allow write: if request.auth != null &&
        request.auth.uid == resource.data.ownerId;
    }

    // Default deny
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

```bash
# Deploy rules
gcloud firestore deploy --rules=firestore.rules
# or via Firebase CLI
npx firebase deploy --only firestore:rules
```

### Firestore Indexes

Create `firestore.indexes.json`:

```json
{
  "indexes": [
    {
      "collectionGroup": "users",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "email", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

```bash
npx firebase deploy --only firestore:indexes
```

---

## Part 7: Database — Cloud SQL (PostgreSQL)

### Create Instance

```bash
# Create Cloud SQL instance
gcloud sql instances create <instance-name> \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=$GCP_REGION \
  --storage-size=10GB \
  --storage-auto-increase

# Create database
gcloud sql databases create <db-name> --instance=<instance-name>

# Create user
gcloud sql users create <username> \
  --instance=<instance-name> \
  --password=<password>
```

### Connect from Cloud Run

Cloud Run connects to Cloud SQL via Unix socket (Cloud SQL Proxy is built in):

```bash
# Add Cloud SQL connection to Cloud Run service
gcloud run services update <service-name> \
  --region $GCP_REGION \
  --add-cloudsql-instances $GCP_PROJECT_ID:$GCP_REGION:<instance-name>
```

Connection string format for Cloud Run:

```
DATABASE_URL=postgresql://<user>:<password>@/<db-name>?host=/cloudsql/<project>:<region>:<instance>
```

### Prisma Setup (if using Prisma)

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  avatarUrl String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

```bash
# Generate client
npx prisma generate

# Push schema to database
npx prisma db push

# Create migration
npx prisma migrate dev --name init
```

### Cloud SQL Helper

```typescript
// src/lib/db/sql.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

---

## Part 8: Authentication

### Firebase Auth (default)

Use the same patterns as the `firebase-auth-setup` skill. Key files:

- `src/lib/firebase/client.ts` — client SDK initialization
- `src/lib/firebase/admin.ts` — admin SDK initialization
- `src/hooks/use-auth.ts` — auth state hook with Google, Apple, email/password providers
- `src/middleware.ts` — server-side token verification

### Identity Platform (enterprise upgrade)

Identity Platform uses the same Firebase Auth SDK but adds:

```bash
# Enable Identity Platform (replaces Firebase Auth)
gcloud services enable identitytoolkit.googleapis.com

# Enable multi-tenancy
gcloud identity-platform config update --enable-multi-tenancy

# Create a tenant
gcloud identity-platform tenants create \
  --display-name="Tenant A" \
  --allow-password-signup \
  --enable-email-link-signin
```

Client-side code is identical to Firebase Auth. Server-side adds tenant awareness:

```typescript
// Verify token with tenant context
import { adminAuth } from "@/lib/firebase/admin";

export async function verifyTokenWithTenant(token: string, tenantId: string) {
  const tenantAuth = adminAuth.tenantManager().authForTenant(tenantId);
  try {
    const decoded = await tenantAuth.verifyIdToken(token);
    return { uid: decoded.uid, email: decoded.email, tenantId: decoded.firebase.tenant };
  } catch {
    return null;
  }
}
```

---

## Part 9: Deployment Pipeline

### Pre-Deploy Checklist

Run these before every deployment. Adapt commands per framework:

```bash
# 1. Type checking
npx tsc --noEmit

# 2. Linting
npx eslint . --ext .ts,.tsx

# 3. Tests
npx vitest run

# 4. Build
npm run build
```

### Cloud Run Deploy (production flow)

```bash
# 1. Ensure on main branch and up to date
git checkout main && git pull origin main

# 2. Merge feature branch
git merge --squash <branch-name>
git commit -m "feat: <summary>"

# 3. Build and push container
gcloud builds submit --tag gcr.io/$GCP_PROJECT_ID/<service-name>

# 4. Deploy new revision
gcloud run deploy <service-name> \
  --image gcr.io/$GCP_PROJECT_ID/<service-name> \
  --platform managed \
  --region $GCP_REGION

# 5. Health check
SERVICE_URL=$(gcloud run services describe <service-name> --region $GCP_REGION --format 'value(status.url)')
curl -sf "$SERVICE_URL/api/health" | jq .

# 6. If health check fails, rollback
gcloud run services update-traffic <service-name> \
  --region $GCP_REGION \
  --to-revisions <previous-revision>=100
```

### GitHub Integration

```bash
# Create PR
gh pr create --title "feat: <title>" --body "<description>" --base main

# Check CI status
gh pr checks <pr-number>

# Merge (squash)
gh pr merge <pr-number> --squash --delete-branch
```

### CI/CD with Cloud Build (optional)

Create `cloudbuild.yaml` in the project root:

```yaml
steps:
  # Install dependencies
  - name: 'node:20'
    entrypoint: npm
    args: ['ci']

  # Run tests
  - name: 'node:20'
    entrypoint: npm
    args: ['test']

  # Build container
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}', '.']

  # Push to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}']

  # Deploy to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - '${_SERVICE_NAME}'
      - '--image=gcr.io/$PROJECT_ID/${_SERVICE_NAME}'
      - '--region=${_REGION}'
      - '--platform=managed'

substitutions:
  _SERVICE_NAME: my-app
  _REGION: us-central1

images:
  - 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}'
```

```bash
# Set up Cloud Build trigger from GitHub
gcloud builds triggers create github \
  --repo-name=<repo> \
  --repo-owner=<owner> \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

---

## Part 10: Cloudflare DNS, CDN, and Security

### API Base

```
https://api.cloudflare.com/client/v4
```

Auth header: `Authorization: Bearer $CLOUDFLARE_API_TOKEN`

### DNS Setup for Cloud Run

Get the Cloud Run service URL, then create the DNS records:

```bash
# Get Cloud Run URL
SERVICE_URL=$(gcloud run services describe <service-name> --region $GCP_REGION --format 'value(status.url)')

# Add CNAME record pointing custom domain to Cloud Run
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "CNAME",
    "name": "<subdomain>",
    "content": "<service-name>-<hash>-<region>.a.run.app",
    "ttl": 1,
    "proxied": true
  }' | jq .
```

### Domain Mapping on Cloud Run

```bash
# Map custom domain to Cloud Run service
gcloud run domain-mappings create \
  --service <service-name> \
  --domain <your-domain.com> \
  --region $GCP_REGION

# Verify domain ownership
gcloud run domain-mappings describe \
  --domain <your-domain.com> \
  --region $GCP_REGION
```

### SSL/TLS Configuration

```bash
# Set SSL to Full (Strict) — required when proxying through Cloudflare
curl -s -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/settings/ssl" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"value": "strict"}' | jq .

# Enable Always Use HTTPS
curl -s -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/settings/always_use_https" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"value": "on"}' | jq .
```

### Rate Limiting

```bash
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/rulesets/phases/http_ratelimit/entrypoint" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "rules": [{
      "expression": "(http.request.uri.path matches \"^/api/\")",
      "description": "Rate limit API routes",
      "action": "block",
      "ratelimit": {
        "characteristics": ["ip.src"],
        "period": 60,
        "requests_per_period": 100,
        "mitigation_timeout": 600
      }
    }]
  }' | jq .
```

### Cache Purge After Deploy

```bash
curl -s -X POST \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"purge_everything": true}' | jq .
```

### Bot Fight Mode

```bash
curl -s -X PUT \
  "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/bot_management" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"fight_mode": true}' | jq .
```

### Standard Setup for New Projects (Cloudflare)

1. Add CNAME record pointing to Cloud Run service URL.
2. Set SSL to Full (Strict).
3. Enable Always Use HTTPS.
4. Add rate limiting for `/api/*` routes.
5. Enable Bot Fight Mode.
6. Set browser cache TTL to 4 hours.
7. Purge cache after every production deployment.

---

## Part 11: Cloud Storage (Static Assets and Uploads)

### Create Bucket

```bash
gcloud storage buckets create gs://$GCP_PROJECT_ID-assets \
  --location=$GCP_REGION \
  --uniform-bucket-level-access

# Make public (for static assets served via CDN)
gcloud storage buckets add-iam-policy-binding gs://$GCP_PROJECT_ID-assets \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

### Upload Helper (server-side)

```typescript
// src/lib/storage.ts
import { Storage } from "@google-cloud/storage";

const storage = new Storage({ projectId: process.env.GCP_PROJECT_ID });
const bucket = storage.bucket(`${process.env.GCP_PROJECT_ID}-assets`);

export async function uploadFile(file: Buffer, filename: string, contentType: string) {
  const blob = bucket.file(filename);
  const stream = blob.createWriteStream({
    metadata: { contentType },
    resumable: false,
  });

  return new Promise<string>((resolve, reject) => {
    stream.on("error", reject);
    stream.on("finish", () => {
      resolve(`https://storage.googleapis.com/${bucket.name}/${filename}`);
    });
    stream.end(file);
  });
}
```

---

## Part 12: Secret Manager

Never hardcode secrets. Use Secret Manager for all sensitive values.

```bash
# Create a secret
echo -n "my-secret-value" | gcloud secrets create <secret-name> --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding <secret-name> \
  --member="serviceAccount:<service-account>@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Mount in Cloud Run
gcloud run services update <service-name> \
  --region $GCP_REGION \
  --set-secrets "ENV_VAR=<secret-name>:latest"
```

---

## Part 13: Monitoring and Logging

### View Logs

```bash
# Cloud Run logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=<service-name>" \
  --limit 50 --format json | jq '.[].textPayload'

# Cloud Functions logs
gcloud functions logs read <function-name> --gen2 --region $GCP_REGION --limit 50
```

### Error Reporting

```bash
# Enable Error Reporting
gcloud services enable clouderrorreporting.googleapis.com
```

---

## Commit Message Convention

All commits must follow Conventional Commits:

- `feat:` — new feature
- `fix:` — bug fix
- `refactor:` — code change that neither fixes a bug nor adds a feature
- `test:` — adding or fixing tests
- `chore:` — tooling, config, deps
- `docs:` — documentation only
- `db:` — database migrations or schema changes
- `infra:` — infrastructure changes (Cloud Run config, Cloudflare rules, IAM)

## Branch Strategy

- `main` = production. Every push triggers production deployment.
- Feature branches (`feat/`, `fix/`, `refactor/`) = preview/staging deploys.
- Never force-push to `main`.

## Safety Rules

- NEVER deploy to production without running the pre-deploy checklist.
- NEVER store credentials in code or commit `.env` files.
- NEVER delete Cloud SQL instances or Firestore databases without explicit user confirmation.
- NEVER modify IAM roles without user approval.
- ALWAYS verify `.gitignore` includes credential files before the first commit.
- For destructive operations (DROP TABLE, delete Cloud Run service, purge Firestore collection), require a dry-run first and show the user what will be affected.

---

## Part 14: Feature Generation

When the user describes a feature, implement the complete vertical slice autonomously. This replaces the need for a separate feature-forge skill.

### Feature Workflow

1. **Analyze** — Parse the description to identify: UI components needed, data model changes (Firestore collections or Cloud SQL tables), API endpoints, auth requirements.
2. **Schema first** — Create or update the data model. For Firestore: define the collection structure, security rules, indexes. For Cloud SQL: create a Prisma migration.
3. **Data access layer** — Create typed helpers in `src/lib/db/` that abstract Firestore or Prisma queries.
4. **API routes** — Create API routes or Server Actions (framework-dependent). Always validate input with Zod. Always check auth.
5. **UI components** — Build the UI using the project's component library (shadcn/ui, custom, etc.). Include loading states, error states, empty states.
6. **Toast notifications** — Add success/error toasts for every user action (create, update, delete). Use the project's toast library (sonner, react-hot-toast, shadcn toast).
7. **Tests** — Write at least: one unit test for the data access function, one integration test for the API route, one E2E test for the critical user flow.
8. **Verify** — Run `npx tsc --noEmit`, linter, and tests. Fix any issues before committing.

### Component Patterns (adapt per framework)

#### Server Action Pattern (Next.js App Router)

```typescript
// src/app/actions/entities.ts
"use server";

import { revalidatePath } from "next/cache";
import { z } from "zod";
import { db } from "@/lib/db/firestore"; // or prisma
import { getAuthUser } from "@/lib/firebase/admin";

const CreateEntitySchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().max(2000).optional(),
});

export async function createEntity(formData: FormData) {
  const user = await getAuthUser();
  if (!user) throw new Error("Unauthorized");

  const parsed = CreateEntitySchema.safeParse({
    name: formData.get("name"),
    description: formData.get("description"),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  // Firestore example
  const ref = await db.collection("entities").add({
    ...parsed.data,
    userId: user.uid,
    createdAt: new Date().toISOString(),
  });

  revalidatePath("/entities");
  return { data: { id: ref.id } };
}
```

#### Form Component with Toast

```tsx
// src/components/entity-form.tsx
"use client";

import { useTransition } from "react";
import { toast } from "sonner"; // or your toast lib
import { createEntity } from "@/app/actions/entities";

export function EntityForm() {
  const [isPending, startTransition] = useTransition();

  async function handleSubmit(formData: FormData) {
    startTransition(async () => {
      const result = await createEntity(formData);
      if (result?.error) {
        toast.error("Failed to create entity");
        return;
      }
      toast.success("Entity created successfully");
    });
  }

  return (
    <form action={handleSubmit}>
      <input name="name" placeholder="Name" required disabled={isPending} />
      <textarea name="description" placeholder="Description" disabled={isPending} />
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Create"}
      </button>
    </form>
  );
}
```

#### Page with Loading/Error/Empty States

```tsx
// src/app/entities/page.tsx
import { Suspense } from "react";

function EntitiesLoading() {
  return <div className="animate-pulse">Loading entities...</div>;
}

async function EntitiesList() {
  const entities = await getEntities(); // your data fetch

  if (entities.length === 0) {
    return (
      <div className="text-center text-muted-foreground py-12">
        <p>No entities yet.</p>
        <a href="/entities/new">Create your first entity</a>
      </div>
    );
  }

  return (
    <ul>
      {entities.map((e) => (
        <li key={e.id}>{e.name}</li>
      ))}
    </ul>
  );
}

export default function EntitiesPage() {
  return (
    <Suspense fallback={<EntitiesLoading />}>
      <EntitiesList />
    </Suspense>
  );
}
```

### Checklist After Feature Generation

- [ ] Data model created (Firestore collection or Prisma migration)
- [ ] Security rules or RLS updated
- [ ] API route or Server Action with Zod validation
- [ ] Auth check on every write operation
- [ ] UI with loading, error, and empty states
- [ ] Toast notifications for success and failure
- [ ] At least one test per layer (unit, integration, E2E)
- [ ] `npx tsc --noEmit` passes
- [ ] Committed with `feat: <description>` message

---

## Part 15: Testing

Write and run tests for GCP-hosted projects. Detect the test framework in use and adapt. This replaces the need for a separate test-sentinel skill.

### Framework Detection

```bash
# Detect test runner
if [ -f "vitest.config.ts" ] || [ -f "vitest.config.js" ]; then
  TEST_RUNNER="vitest"
elif [ -f "jest.config.ts" ] || [ -f "jest.config.js" ]; then
  TEST_RUNNER="jest"
else
  TEST_RUNNER="vitest"  # default, install if missing
fi

# Detect E2E framework
if [ -f "playwright.config.ts" ]; then
  E2E_RUNNER="playwright"
elif [ -f "cypress.config.ts" ] || [ -f "cypress.config.js" ]; then
  E2E_RUNNER="cypress"
else
  E2E_RUNNER="playwright"  # default
fi
```

### Unit Tests

For: utility functions, Zod schemas, data transformations, hooks, stores.

Location: `src/**/__tests__/<name>.test.ts` (colocated with the code being tested).

```typescript
import { describe, it, expect } from "vitest"; // or jest
import { formatCurrency } from "@/lib/utils";

describe("formatCurrency", () => {
  it("formats BRL correctly", () => {
    expect(formatCurrency(1999, "BRL")).toBe("R$ 19,99");
  });

  it("handles zero", () => {
    expect(formatCurrency(0, "BRL")).toBe("R$ 0,00");
  });
});
```

### Integration Tests

For: API routes, Server Actions, data access functions. Mock the database client.

```typescript
import { describe, it, expect, vi } from "vitest";

// Mock Firestore
vi.mock("@/lib/db/firestore", () => ({
  db: {
    collection: vi.fn(() => ({
      add: vi.fn(() => ({ id: "mock-id" })),
      doc: vi.fn(() => ({
        get: vi.fn(() => ({ exists: true, data: () => ({ name: "Test" }) })),
      })),
      where: vi.fn(() => ({
        get: vi.fn(() => ({
          docs: [{ id: "1", data: () => ({ name: "Test" }) }],
        })),
      })),
    })),
  },
}));

// OR mock Prisma
vi.mock("@/lib/db/sql", () => ({
  prisma: {
    entity: {
      findMany: vi.fn(() => [{ id: "1", name: "Test" }]),
      create: vi.fn((args) => ({ id: "new-id", ...args.data })),
    },
  },
}));
```

### E2E Tests (Playwright)

For: critical user flows (auth, main feature happy paths).

```typescript
import { test, expect } from "@playwright/test";

test.describe("Entity CRUD Flow", () => {
  test.beforeEach(async ({ page }) => {
    // Login (use storageState or test account)
    await page.goto("/login");
    await page.fill('[name="email"]', process.env.TEST_USER_EMAIL!);
    await page.fill('[name="password"]', process.env.TEST_USER_PASSWORD!);
    await page.click('button[type="submit"]');
    await page.waitForURL("/dashboard");
  });

  test("create, view, and delete entity", async ({ page }) => {
    // Create
    await page.goto("/entities/new");
    await page.fill('[name="name"]', "Test Entity");
    await page.click('button[type="submit"]');
    await expect(page.locator('[data-sonner-toast]')).toContainText(/created|success/i);

    // View
    await page.goto("/entities");
    await expect(page.locator("text=Test Entity")).toBeVisible();

    // Delete
    await page.click('[data-testid="delete-entity"]');
    await page.click('button:has-text("Confirm")');
    await expect(page.locator('[data-sonner-toast]')).toContainText(/deleted/i);
  });
});
```

### Running Tests

```bash
# Full suite
npx vitest run && npx playwright test

# With coverage
npx vitest run --coverage

# Specific file
npx vitest run src/lib/__tests__/utils.test.ts

# E2E only
npx playwright test e2e/entities.spec.ts
```

### Failure Analysis Workflow

1. Read the error output. Identify: test bug vs code bug.
2. If test bug: fix the test (wrong expectation, missing mock, outdated snapshot).
3. If code bug: fix the source code, re-run the failing test to confirm.
4. If flaky: add retry logic or improve isolation. Mark with `// TODO: flaky`.
5. Re-run the full suite after any fix.
6. Commit: `test: fix <description>`.

### Linting and Type Checking

```bash
# Lint
npx eslint . --ext .ts,.tsx
npx eslint . --ext .ts,.tsx --fix  # auto-fix

# Type check
npx tsc --noEmit

# Format
npx prettier --check .
npx prettier --write .  # auto-fix
```

### Test Data Patterns

Use factory functions, not raw objects:

```typescript
// src/__tests__/__fixtures__/factories.ts
export function makeUser(overrides = {}) {
  return {
    id: "test-user-id",
    email: "test@example.com",
    displayName: "Test User",
    ...overrides,
  };
}

export function makeEntity(overrides = {}) {
  return {
    id: "test-entity-id",
    name: "Test Entity",
    userId: "test-user-id",
    createdAt: new Date().toISOString(),
    ...overrides,
  };
}
```

### Quality Gates (before reporting "all tests pass")

- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] E2E tests pass (if applicable)
- [ ] No lint errors
- [ ] No TypeScript errors (`npx tsc --noEmit`)
- [ ] Coverage does not decrease

---

## Part 16: Pre-Production QA Gate

Before deploying to production, execute a comprehensive validation sweep. This replaces the need for a separate qa-gate skill. The agent generates a test plan, runs all validations, and produces a go/no-go report.

### QA Workflow

```
1. Generate test plan          → qa-reports/test-plan.json
2. Run existing test suite     → npx vitest run + npx playwright test
3. Generate validation tests   → qa-tests/**/*.validation.test.ts
4. Run API validations         → qa-tests/api/
5. Run UI/toast validations    → qa-tests/ui/
6. Run auth flow validations   → qa-tests/auth/
7. Run LLM quality checks      → qa-tests/llm/ (if app has LLM features)
8. Run GCP infra health checks → qa-tests/infra/
9. Aggregate results           → qa-reports/go-no-go-report.json
10. Generate human report      → qa-reports/go-no-go-report.md
```

### Test Plan Schema

Save to `qa-reports/test-plan.json`:

```json
{
  "project": "project-name",
  "version": "x.y.z",
  "date": "ISO-8601",
  "validator": "gcp-fullstack",
  "surfaces": {
    "api_routes": [],
    "server_actions": [],
    "ui_pages": [],
    "toast_notifications": [],
    "auth_flows": [],
    "llm_features": [],
    "database_integrity": [],
    "gcp_infrastructure": []
  }
}
```

### Surface Discovery

- API routes: scan `src/app/api/**/route.ts` (Next.js) or equivalent
- Server Actions: grep for `"use server"`
- UI pages: scan `src/app/**/page.tsx`
- Toast notifications: grep for toast library usage (sonner, react-hot-toast, shadcn toast)
- Auth flows: check Firebase auth setup, middleware
- LLM features: grep for OpenAI/OpenRouter/Anthropic API calls
- Database: read Firestore rules (`firestore.rules`) or Prisma schema (`prisma/schema.prisma`)
- GCP infra: check Cloud Run services, Cloud SQL instances, Secret Manager secrets

### API Validation Template

```typescript
// qa-tests/api/entities.validation.test.ts
const BASE_URL = process.env.VALIDATION_BASE_URL || "http://localhost:3000";

describe("API Validation: /api/entities", () => {
  it("returns 200 for authenticated GET", async () => {
    const res = await fetch(`${BASE_URL}/api/entities`, {
      headers: { Authorization: `Bearer ${process.env.TEST_AUTH_TOKEN}` },
    });
    expect(res.status).toBe(200);
  });

  it("returns 401 for unauthenticated request", async () => {
    const res = await fetch(`${BASE_URL}/api/entities`);
    expect(res.status).toBe(401);
  });

  it("response matches expected schema", async () => {
    const res = await fetch(`${BASE_URL}/api/entities`, {
      headers: { Authorization: `Bearer ${process.env.TEST_AUTH_TOKEN}` },
    });
    const data = await res.json();
    expect(Array.isArray(data)).toBe(true);
  });

  it("returns 405 for unsupported methods", async () => {
    const res = await fetch(`${BASE_URL}/api/entities`, { method: "DELETE" });
    expect(res.status).toBe(405);
  });
});
```

### Toast Validation Template

```typescript
// qa-tests/ui/toasts.validation.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Toast Validation", () => {
  test("success toast on entity creation", async ({ page }) => {
    await page.goto("/entities/new");
    await page.fill('[name="name"]', "Test Entity");
    await page.click('button[type="submit"]');
    const toast = page.locator('[data-sonner-toast], [role="status"], .Toastify__toast');
    await expect(toast).toBeVisible({ timeout: 5000 });
    await expect(toast).toContainText(/created|success/i);
  });

  test("error toast on failure", async ({ page }) => {
    await page.route("**/api/entities", (route) =>
      route.fulfill({ status: 500, body: JSON.stringify({ error: "Failed" }) })
    );
    await page.goto("/entities/new");
    await page.fill('[name="name"]', "Test");
    await page.click('button[type="submit"]');
    const toast = page.locator('[data-sonner-toast][data-type="error"], [role="alert"]');
    await expect(toast).toBeVisible({ timeout: 5000 });
  });

  test("no duplicate toasts on rapid clicks", async ({ page }) => {
    await page.goto("/entities/new");
    await page.fill('[name="name"]', "Test");
    await page.click('button[type="submit"]');
    await page.click('button[type="submit"]');
    const toasts = page.locator('[data-sonner-toast], [role="status"]');
    expect(await toasts.count()).toBeLessThanOrEqual(1);
  });
});
```

### GCP Infrastructure Health Checks

```bash
# Cloud Run service status
gcloud run services describe <service-name> --region $GCP_REGION --format="value(status.conditions[0].status)"

# Cloud Run health endpoint
SERVICE_URL=$(gcloud run services describe <service-name> --region $GCP_REGION --format 'value(status.url)')
curl -sf "$SERVICE_URL/api/health" | jq .

# Cloud SQL instance status
gcloud sql instances describe <instance-name> --format="value(state)"

# Cloud SQL backup check
gcloud sql backups list --instance=<instance-name> --limit=1 --format="value(status)"

# Cloud SQL SSL enforcement
gcloud sql instances describe <instance-name> --format="value(settings.ipConfiguration.requireSsl)"

# Firestore security rules deployed
gcloud firestore operations list --limit=1

# Secret Manager — all required secrets exist
for SECRET in "firebase-config" "cloudflare-token" "cross-app-secret"; do
  gcloud secrets describe $SECRET --format="value(name)" 2>/dev/null && echo "$SECRET: OK" || echo "$SECRET: MISSING"
done
```

All `gcloud` commands during QA are READ-ONLY (describe, list). NEVER run create, update, or delete during validation.

### LLM Output Quality Validation (two-layer)

#### Layer 1: Rule-Based Checks

```typescript
export function runRuleBasedChecks(output: { content: string; tokens_used: number; latency_ms: number }, config: {
  minLength?: number;
  maxLength?: number;
  maxTokens?: number;
  maxLatencyMs?: number;
  forbiddenPatterns?: RegExp[];
  requiredFormat?: "json" | "markdown" | "plain";
}): { rule: string; passed: boolean; details: string }[] {
  const results = [];

  if (config.minLength) {
    results.push({ rule: "min_length", passed: output.content.length >= config.minLength,
      details: `Length: ${output.content.length}, min: ${config.minLength}` });
  }
  if (config.maxLatencyMs) {
    results.push({ rule: "latency", passed: output.latency_ms <= config.maxLatencyMs,
      details: `Latency: ${output.latency_ms}ms, max: ${config.maxLatencyMs}ms` });
  }
  if (config.forbiddenPatterns) {
    for (const p of config.forbiddenPatterns) {
      const match = p.exec(output.content);
      results.push({ rule: `forbidden:${p.source}`, passed: !match,
        details: match ? `Found: "${match[0]}"` : "Clean" });
    }
  }
  if (config.requiredFormat === "json") {
    try { JSON.parse(output.content); results.push({ rule: "valid_json", passed: true, details: "OK" });
    } catch { results.push({ rule: "valid_json", passed: false, details: "Invalid JSON" }); }
  }
  results.push({ rule: "not_empty", passed: output.content.trim().length > 0, details: "" });
  results.push({ rule: "not_truncated", passed: !output.content.endsWith("..."), details: "" });

  return results;
}
```

#### Layer 2: LLM-as-Judge (via OpenRouter)

```typescript
export async function llmJudge(output: string, prompt: string, criteria: {
  relevance?: boolean; accuracy?: boolean; completeness?: boolean; tone?: boolean; safety?: boolean;
}): Promise<{ overall_score: number; issues: string[]; recommendation: "pass" | "review" | "fail" }> {
  const OPENROUTER_API_KEY = process.env.OPENROUTER_API_KEY;
  if (!OPENROUTER_API_KEY) {
    return { overall_score: 0, issues: ["OPENROUTER_API_KEY not set — skipping"], recommendation: "review" };
  }

  const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: { Authorization: `Bearer ${OPENROUTER_API_KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "google/gemini-flash-1.5",
      messages: [{ role: "user", content: `Evaluate this LLM output...\nPROMPT: ${prompt}\nOUTPUT: ${output}\nScore 1-5 on: ${Object.keys(criteria).join(", ")}. JSON response.` }],
      temperature: 0.1,
      response_format: { type: "json_object" },
    }),
  });

  const data = await response.json();
  return JSON.parse(data.choices[0].message.content);
}
```

Always run rule-based checks BEFORE LLM-as-judge (cheaper, faster). If `OPENROUTER_API_KEY` is not set, skip LLM judge and mark as "review".

### Go/No-Go Report

After all validations, generate `qa-reports/go-no-go-report.json`:

```json
{
  "project": "project-name",
  "version": "x.y.z",
  "date": "ISO-8601",
  "verdict": "GO | NO-GO | CONDITIONAL",
  "summary": {
    "total_checks": 45,
    "passed": 42,
    "failed": 2,
    "skipped": 1,
    "pass_rate": "93.3%"
  },
  "sections": {
    "api_routes": { "status": "PASS", "checks_run": 12, "checks_passed": 12 },
    "ui_pages": { "status": "PASS", "checks_run": 8, "checks_passed": 8 },
    "toast_notifications": { "status": "FAIL", "failures": [] },
    "auth_flows": { "status": "PASS" },
    "llm_quality": { "rule_based": {}, "llm_judge": {} },
    "database_integrity": { "status": "PASS" },
    "gcp_infrastructure": {
      "cloud_run": "READY",
      "cloud_sql": "RUNNING",
      "cloud_sql_ssl": true,
      "cloud_sql_backup": "SUCCESSFUL",
      "firestore_rules": "DEPLOYED",
      "secret_manager": "ALL_PRESENT"
    }
  },
  "blockers": [],
  "warnings": []
}
```

### Verdict Logic

- **GO**: All checks pass, no blockers, no high-severity failures.
- **NO-GO**: Any high-severity blocker OR any auth failure OR any data integrity failure.
- **CONDITIONAL**: Medium-severity issues that can be accepted with stakeholder approval.

Also generate `qa-reports/go-no-go-report.md` (human-readable version).

NEVER auto-deploy after a CONDITIONAL or NO-GO verdict. NEVER delete test data from production databases. Redact API keys from reports before writing to disk.
