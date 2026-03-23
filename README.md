# OpenClaw Skills — Full-Stack Developer Workflow

A modular set of OpenClaw skills that cover the complete development lifecycle for web applications. Includes 7 skills for the Vercel/Supabase stack, 1 consolidated super-skill for the Google Cloud Platform stack, a UI theming skill for shadcn/ui, a web scraping/content comprehension skill, pre-production QA gate skills for both stacks, and an integration architecture skill for multi-app monorepos.

## Skills Overview

| Skill | Purpose | Frequency |
|-------|---------|-----------|
| `stack-scaffold` | Create new projects from scratch | Per project |
| `supabase-ops` | DB migrations, RLS, types, edge functions | Every schema change |
| `feature-forge` | Build complete features from descriptions | Daily |
| `test-sentinel` | Write/run tests, lint, auto-fix | Before every merge |
| `qa-gate-vercel` | Pre-production validation gate — test plans, API/UI/toast/auth/LLM quality validation, go/no-go reports | Before every production deploy |
| `deploy-pilot` | Build validation, deploy, monitoring | Every deploy |
| `cloudflare-guard` | DNS, caching, security, Workers | On demand |
| `firebase-auth-setup` | Auth providers, hooks, user sync | Per project + on demand |

### GCP Stack (alternative)

| Skill | Purpose | Frequency |
|-------|---------|-----------|
| `gcp-fullstack` | Full lifecycle — scaffold, compute (Cloud Run/Functions/App Engine), database (Firestore/Cloud SQL), auth (Firebase/Identity Platform), deploy, Cloudflare DNS/CDN/security | All stages |
| `qa-gate-gcp` | Pre-production validation gate for GCP — test plans, API/UI/toast/auth/LLM quality, infra health (Cloud Run, Cloud SQL, Firestore rules, Secret Manager), go/no-go reports | Before every production deploy |

### UI / Design System

| Skill | Purpose | Frequency |
|-------|---------|-----------|
| `shadcn-theme-default` | Enforces default shadcn/ui Neutral theme (black/white/gray) with OKLCH variables, Tailwind v4, and dark mode | Every component/page |

### Data / Content Extraction

| Skill | Purpose | Frequency |
|-------|---------|-----------|
| `web-scraper` | Multi-strategy web scraping with cascade fallback (static/Playwright/Scrapy), news detection, boilerplate removal (trafilatura), metadata extraction, and LLM entity extraction via OpenRouter | On demand |

### Integration / Multi-App

| Skill | Purpose | Frequency |
|-------|---------|-----------|
| `interop-forge` | Integration architect for multi-app monorepos — shared contracts (@repo/contracts), API-first with OpenAPI, cross-app JWT auth, auto-generated typed SDKs, full MCP server per app | Per project + on demand |

## Workflow Diagram

```
New Project:
  stack-scaffold → firebase-auth-setup → cloudflare-guard → deploy-pilot (initial deploy)

Feature Development Cycle:
  feature-forge → supabase-ops (if DB changes) → test-sentinel → qa-gate-vercel (pre-prod) → deploy-pilot

Maintenance:
  supabase-ops (migrations) → test-sentinel → deploy-pilot
  cloudflare-guard (security updates, cache purge)

GCP Stack (alternative — single skill handles all):
  gcp-fullstack (scaffold) → gcp-fullstack (database setup) → gcp-fullstack (auth) → qa-gate-gcp (pre-prod) → gcp-fullstack (deploy + Cloudflare)

Web Scraping Pipeline:
  web-scraper (detect article) → web-scraper (extract content) → web-scraper (clean + metadata) → web-scraper (LLM entities)

Multi-App Integration (interop-forge handles all):
  interop-forge (monorepo setup) → interop-forge (shared contracts) → interop-forge (OpenAPI specs)
  → interop-forge (SDK generation) → interop-forge (cross-app auth) → interop-forge (MCP servers)
```

## When to Use GCP vs Vercel Stack

| Factor | Vercel/Supabase Stack | GCP Stack |
|--------|----------------------|-----------|
| **Best for** | Startups, MVPs, content sites, SaaS | Enterprise apps, complex backends, ML workloads |
| **Deploy speed** | Instant (Git push → live) | Minutes (Docker build → Cloud Run) |
| **Database** | Supabase (Postgres + realtime) | Cloud SQL (Postgres/MySQL) or Firestore (NoSQL) |
| **Auth** | Firebase Auth (both stacks) | Firebase Auth or Identity Platform |
| **Scaling** | Automatic (Vercel edge) | Manual or auto (Cloud Run min/max instances) |
| **Cost at scale** | Can get expensive with high traffic | More predictable, better for high-volume |
| **Cold starts** | None (edge functions) | Possible (Cloud Run, Cloud Functions) |
| **Vendor lock-in** | Medium (Vercel + Supabase) | High (GCP services) |
| **Setup complexity** | Low (7 focused skills) | Medium (1 comprehensive skill) |
| **Offline/local dev** | Supabase local + Next.js dev server | Docker Compose + emulators |

**Choose Vercel/Supabase if:** you want fast iteration, real-time features, and minimal DevOps overhead.

**Choose GCP if:** you need fine-grained infrastructure control, complex backend processing, or integration with other Google Cloud services (BigQuery, Pub/Sub, Vertex AI).

**You can mix both:** Use Vercel for the frontend and GCP for backend services. The `interop-forge` skill helps set up multi-app architectures that span both stacks.

## Example Outputs

The `examples/` directory contains sample output files generated by the skills:

| File | Generated by | Description |
|------|-------------|-------------|
| `examples/test-plan.json` | qa-gate-vercel | Sample test plan with API, UI, and auth test suites |
| `examples/go-no-go.json` | qa-gate-vercel | Sample go/no-go report with categorized checks and verdict |
| `examples/preflight-report.json` | preflight-check | Sample environment validation report |
| `examples/monorepo-structure.txt` | interop-forge | Example monorepo directory structure |

These examples help you understand what each skill produces before running it.

## Installation

### Option 1: Workspace Skills (recommended for team use)
Copy the skill directories into your OpenClaw workspace:

```bash
cp -r openclaw-skills/* <your-workspace>/skills/
```

### Option 2: Local Skills
Copy to your local OpenClaw skills directory:

```bash
cp -r openclaw-skills/* ~/.openclaw/skills/
```

### Option 3: ClawHub
After publishing (see Publishing section), install via:

```bash
clawhub install stack-scaffold
clawhub install supabase-ops
clawhub install feature-forge
clawhub install test-sentinel
clawhub install deploy-pilot
clawhub install cloudflare-guard
clawhub install firebase-auth-setup

# QA Gate (Vercel/Supabase stack)
clawhub install qa-gate-vercel

# GCP alternative stack
clawhub install gcp-fullstack

# QA Gate (GCP stack)
clawhub install qa-gate-gcp

# UI / Design System
clawhub install shadcn-theme-default

# Data / Content Extraction
clawhub install web-scraper

# Integration / Multi-App
clawhub install interop-forge
```

---

## Credentials Guide — How to Get Each One

The agent needs the following credentials to operate. Here is where to get each one and how to configure it.

### Supabase

| Variable | Where to get it |
|----------|----------------|
| `SUPABASE_URL` | Supabase Dashboard > your project > Settings > API > Project URL |
| `SUPABASE_ANON_KEY` | Same page > Project API keys > `anon` `public` |
| `SUPABASE_SERVICE_ROLE_KEY` | Same page > Project API keys > `service_role` (keep secret, server-only) |
| `SUPABASE_PROJECT_ID` | Supabase Dashboard > your project > Settings > General > Reference ID |

Steps:
1. Go to https://supabase.com/dashboard and sign in.
2. Create a new project (or select an existing one).
3. Navigate to **Settings > API**.
4. Copy the **Project URL** → this is `SUPABASE_URL`.
5. Copy the **anon public key** → this is `SUPABASE_ANON_KEY`.
6. Copy the **service_role secret key** → this is `SUPABASE_SERVICE_ROLE_KEY`. NEVER expose this in client-side code.
7. Navigate to **Settings > General** and copy the **Reference ID** → this is `SUPABASE_PROJECT_ID`.

For the `NEXT_PUBLIC_` variants, use the same values:
```bash
NEXT_PUBLIC_SUPABASE_URL=$SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY=$SUPABASE_ANON_KEY
```

### Firebase

| Variable | Where to get it |
|----------|----------------|
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase Console > Project Settings > General > Your apps > Web app > Config |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Same config object |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Same config object |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | Same config object |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Same config object |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Same config object |
| `FIREBASE_PROJECT_ID` | Same as `NEXT_PUBLIC_FIREBASE_PROJECT_ID` |
| `FIREBASE_CLIENT_EMAIL` | Firebase Console > Project Settings > Service accounts > Generate new private key (JSON file) > `client_email` field |
| `FIREBASE_PRIVATE_KEY` | Same JSON file > `private_key` field |

Steps:
1. Go to https://console.firebase.google.com and sign in.
2. Create a new project (or select an existing one).
3. Go to **Project Settings** (gear icon) > **General**.
4. Scroll down to **Your apps**. If no web app exists, click **Add app** > **Web** (</>).
5. Copy the `firebaseConfig` object. Each key maps to the env vars listed above.
6. For the server-side credentials, go to **Project Settings > Service accounts**.
7. Click **Generate new private key**. A JSON file will be downloaded.
8. Open the JSON file. Copy `client_email` → `FIREBASE_CLIENT_EMAIL`. Copy `private_key` → `FIREBASE_PRIVATE_KEY`.
9. IMPORTANT: The `private_key` contains `\n` characters. Store it wrapped in double quotes in your `.env` file:
   ```
   FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEv...rest of key...\n-----END PRIVATE KEY-----\n"
   ```
10. Enable the auth providers you need: **Authentication > Sign-in method** > enable Google, Apple, Email/Password, etc.

### Vercel

| Variable | Where to get it |
|----------|----------------|
| `VERCEL_TOKEN` | Vercel Dashboard > Settings > Tokens |

Steps:
1. Go to https://vercel.com/account/tokens.
2. Click **Create** token.
3. Give it a name (e.g., `openclaw-deploy`) and set the scope to your team (or personal).
4. Set an expiration (recommended: 90 days, then rotate).
5. Copy the token immediately — it will not be shown again.
6. To link your project to Vercel (first time only):
   ```bash
   cd <your-project>
   npx vercel link
   ```

### Cloudflare

| Variable | Where to get it |
|----------|----------------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare Dashboard > My Profile > API Tokens |
| `CLOUDFLARE_ZONE_ID` | Cloudflare Dashboard > your domain > Overview (right sidebar) |

Steps:
1. Go to https://dash.cloudflare.com/profile/api-tokens.
2. Click **Create Token**.
3. Use the **Edit zone DNS** template as a starting point.
4. Add these permissions:
   - Zone > DNS > Edit
   - Zone > Zone Settings > Edit
   - Zone > Cache Purge > Purge
   - Zone > Page Rules > Edit
   - Zone > Firewall Services > Edit (for rate limiting and WAF)
5. Under **Zone Resources**, select **Include > Specific zone** > your domain.
6. Click **Continue to summary** > **Create Token**.
7. Copy the token immediately.
8. To get the Zone ID: go to your domain in the Cloudflare Dashboard > **Overview**. The Zone ID is in the right sidebar under **API**.

### Google Cloud Platform (for gcp-fullstack skill)

| Variable | Where to get it |
|----------|----------------|
| `GCP_PROJECT_ID` | Google Cloud Console > select project > Project ID in the dashboard header |
| `GCP_REGION` | Your preferred region (e.g., `us-central1`, `southamerica-east1`) |
| `GOOGLE_APPLICATION_CREDENTIALS` | Path to a service account JSON key file |

Steps:
1. Go to https://console.cloud.google.com and sign in.
2. Create a new project (or select an existing one). Note the **Project ID** (not the project name).
3. Enable the required APIs:
   ```bash
   gcloud services enable \
     run.googleapis.com \
     cloudbuild.googleapis.com \
     sqladmin.googleapis.com \
     firestore.googleapis.com \
     secretmanager.googleapis.com \
     cloudfunctions.googleapis.com \
     identitytoolkit.googleapis.com
   ```
4. Create a service account for CI/CD:
   ```bash
   gcloud iam service-accounts create openclaw-deployer \
     --display-name="OpenClaw Deployer"
   ```
5. Grant necessary roles:
   ```bash
   PROJECT_ID=$(gcloud config get-value project)
   SA_EMAIL="openclaw-deployer@$PROJECT_ID.iam.gserviceaccount.com"
   for ROLE in roles/run.admin roles/cloudbuild.builds.editor roles/cloudsql.admin roles/datastore.owner roles/secretmanager.admin roles/iam.serviceAccountUser roles/storage.admin; do
     gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SA_EMAIL" --role="$ROLE"
   done
   ```
6. Generate a key file:
   ```bash
   gcloud iam service-accounts keys create ~/openclaw-deployer-key.json \
     --iam-account=$SA_EMAIL
   ```
7. Set the env var:
   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS=~/openclaw-deployer-key.json
   ```
8. Install the `gcloud` CLI: https://cloud.google.com/sdk/docs/install
9. Authenticate locally:
   ```bash
   gcloud auth login
   gcloud config set project $GCP_PROJECT_ID
   gcloud config set run/region $GCP_REGION
   ```

### OpenRouter (for web-scraper LLM entity extraction)

| Variable | Where to get it |
|----------|----------------|
| `OPENROUTER_API_KEY` | OpenRouter Dashboard > Keys |

Steps:
1. Go to https://openrouter.ai and sign in (or create an account).
2. Navigate to **Keys** (https://openrouter.ai/keys).
3. Click **Create Key**.
4. Give it a name (e.g., `openclaw-scraper`) and set usage limits if desired.
5. Copy the key immediately — it starts with `sk-or-`.
6. The web-scraper skill uses this key in generated Python scripts for LLM entity extraction (Stage 5). Models used: `google/gemini-flash-1.5` or similar low-cost models for structured extraction.

### GitHub CLI (gh)

Not an environment variable, but required by `deploy-pilot`. Install and authenticate:

```bash
# Install
brew install gh          # macOS
sudo apt install gh      # Ubuntu/Debian

# Authenticate
gh auth login
# Choose: GitHub.com > HTTPS > Login with browser
```

### Summary of All Environment Variables

```bash
# === Supabase ===
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGci...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGci...
SUPABASE_PROJECT_ID=xxxx
NEXT_PUBLIC_SUPABASE_URL=$SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY=$SUPABASE_ANON_KEY

# === Firebase Client (safe for client-side) ===
NEXT_PUBLIC_FIREBASE_API_KEY=AIza...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your-project-id
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=123456789
NEXT_PUBLIC_FIREBASE_APP_ID=1:123456789:web:abc123

# === Firebase Admin (server-only, NEVER expose) ===
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxx@your-project.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# === Vercel (Vercel/Supabase stack) ===
VERCEL_TOKEN=your-vercel-token

# === GCP (GCP stack) ===
GCP_PROJECT_ID=your-gcp-project-id
GCP_REGION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account-key.json

# === Cloudflare ===
CLOUDFLARE_API_TOKEN=your-cf-api-token
CLOUDFLARE_ZONE_ID=your-zone-id

# === OpenRouter (web-scraper LLM entity extraction) ===
OPENROUTER_API_KEY=sk-or-...
```

### Where to Store These

For **OpenClaw** to load the skills, these variables must be available in the shell where OpenClaw runs. Options:

1. **Shell profile** (`~/.zshrc` or `~/.bashrc`): Add `export VAR=value` lines. Best for development machine.
2. **`.env` file in the OpenClaw workspace**: If OpenClaw supports dotenv loading.
3. **Per-project `.env.local`**: For Next.js runtime (not for OpenClaw skill gating, but for the app itself).

IMPORTANT: Never commit `.env.local` or any file containing real credentials. Add them to `.gitignore`.

### Credential Rotation Schedule

| Credential | Rotation frequency | How to rotate |
|------------|--------------------|---------------|
| `VERCEL_TOKEN` | Every 90 days | Create new token in Vercel Dashboard, update env var, delete old token |
| `CLOUDFLARE_API_TOKEN` | Every 90 days | Same process in Cloudflare Dashboard |
| `SUPABASE_SERVICE_ROLE_KEY` | When compromised | Regenerate in Supabase Dashboard > Settings > API |
| `FIREBASE_PRIVATE_KEY` | Every 6 months | Generate new service account key, update env var, delete old key from Firebase Console |
| `GOOGLE_APPLICATION_CREDENTIALS` | Every 6 months | Generate new service account key via `gcloud iam service-accounts keys create`, update file path, delete old key |
| `OPENROUTER_API_KEY` | When compromised or on billing cycle | Create new key in OpenRouter Dashboard, update env var, delete old key |

---

## Security Guidelines

1. **API keys are NEVER hardcoded** in any skill. They are always loaded from environment variables at runtime.
2. **Production operations** include safety mechanisms: dry-runs for DB migrations, health checks after deploys, team notifications before prod deploys.
3. **Supabase RLS is mandatory.** The `supabase-ops` skill enforces RLS on every table.
4. **Service role keys** are only used server-side (Firebase Admin, Supabase admin operations). Never expose in client code.
5. **Review before enabling.** Even though these are your own skills, always review after updates — especially if team members contribute.
6. **Post-ClawHavoc precautions.** In February 2026, 341 malicious skills were found on ClawHub. Always inspect third-party skills before installing (`clawhub inspect <slug>`).

## Autonomy Model

These skills are configured for maximum autonomy with a mandatory planning phase:

- **Every skill** includes a Planning Protocol that forces the agent to survey the environment, build an execution plan, and identify risks BEFORE taking any action. This prevents the agent from rushing into execution and making costly mistakes.
- **Dev environment:** Full autonomy after planning. Skills execute without confirmation.
- **Production environment:** Skills execute autonomously but include safety nets:
  - `deploy-pilot` sends deployment summaries before pushing to prod.
  - `supabase-ops` runs dry-runs before applying prod migrations.
  - `cloudflare-guard` logs all API calls for audit.

---

## Publishing to ClawHub

### Prerequisites

1. A GitHub account that is at least 1 week old.
2. The ClawHub CLI authenticated:
   ```bash
   openclaw auth login
   ```

### Validating Skills Before Publishing

Always validate and test locally first:

```bash
# Validate manifest and structure
openclaw skill validate ./stack-scaffold

# Test the skill locally
openclaw skill test ./stack-scaffold --verbose
```

Repeat for each skill.

### Publishing

For each skill, run:

```bash
clawhub publish ./stack-scaffold \
  --slug stack-scaffold \
  --name "Stack Scaffold" \
  --version 1.0.0 \
  --changelog "Initial release: Next.js + Supabase + Firebase Auth + Vercel + Cloudflare scaffolding"
```

The full list of publish commands:

```bash
# 1. Stack Scaffold
clawhub publish ./stack-scaffold \
  --slug stack-scaffold \
  --name "Stack Scaffold" \
  --version 1.0.0 \
  --changelog "Initial release"

# 2. Supabase Ops
clawhub publish ./supabase-ops \
  --slug supabase-ops \
  --name "Supabase Ops" \
  --version 1.0.0 \
  --changelog "Initial release"

# 3. Feature Forge
clawhub publish ./feature-forge \
  --slug feature-forge \
  --name "Feature Forge" \
  --version 1.0.0 \
  --changelog "Initial release"

# 4. Test Sentinel
clawhub publish ./test-sentinel \
  --slug test-sentinel \
  --name "Test Sentinel" \
  --version 1.0.0 \
  --changelog "Initial release"

# 5. Deploy Pilot
clawhub publish ./deploy-pilot \
  --slug deploy-pilot \
  --name "Deploy Pilot" \
  --version 1.0.0 \
  --changelog "Initial release"

# 6. Cloudflare Guard
clawhub publish ./cloudflare-guard \
  --slug cloudflare-guard \
  --name "Cloudflare Guard" \
  --version 1.0.0 \
  --changelog "Initial release"

# 7. Firebase Auth Setup
clawhub publish ./firebase-auth-setup \
  --slug firebase-auth-setup \
  --name "Firebase Auth Setup" \
  --version 1.0.0 \
  --changelog "Initial release"

# 8. GCP Fullstack (alternative stack)
clawhub publish ./gcp-fullstack \
  --slug gcp-fullstack \
  --name "GCP Fullstack" \
  --version 1.0.0 \
  --changelog "Initial release: Cloud Run, Cloud SQL, Firestore, Firebase Auth, Identity Platform, Cloudflare, GitHub"

# 9. shadcn Theme Default
clawhub publish ./shadcn-theme-default \
  --slug shadcn-theme-default \
  --name "shadcn Theme Default" \
  --version 1.0.0 \
  --changelog "Initial release: Neutral theme with OKLCH variables, Tailwind v4 integration, dark mode, component patterns"

# 10. Web Scraper
clawhub publish ./web-scraper \
  --slug web-scraper \
  --name "Web Scraper" \
  --version 1.0.0 \
  --changelog "Initial release: multi-strategy cascade (static/Playwright/Scrapy), news detection, trafilatura cleaning, metadata extraction, LLM entity extraction via OpenRouter"

# 11. QA Gate Vercel
clawhub publish ./qa-gate-vercel \
  --slug qa-gate-vercel \
  --name "QA Gate Vercel" \
  --version 1.0.0 \
  --changelog "Initial release: pre-production validation for Vercel/Supabase/Firebase — test plans, API/UI/toast/auth/LLM quality, go/no-go reports"

# 12. QA Gate GCP
clawhub publish ./qa-gate-gcp \
  --slug qa-gate-gcp \
  --name "QA Gate GCP" \
  --version 1.0.0 \
  --changelog "Initial release: pre-production validation for GCP — test plans, API/UI/toast/auth/LLM quality, Cloud Run/SQL/Firestore/Secret Manager health, go/no-go reports"

# 13. Interop Forge
clawhub publish ./interop-forge \
  --slug interop-forge \
  --name "Interop Forge" \
  --version 1.0.0 \
  --changelog "Initial release: monorepo integration — shared contracts, OpenAPI specs, typed SDKs, cross-app JWT auth, full MCP server scaffolding per app"
```

### What Happens After Publishing

1. **VirusTotal scan** — Every skill is scanned automatically. If flagged, it enters manual review.
2. **Public listing** — Once approved, the skill appears on clawhub.ai with install count, stars, and comments.
3. **Daily re-scans** — ClawHub re-scans published skills daily for emerging threats.

### Updating Published Skills

To push a new version:

```bash
clawhub publish ./stack-scaffold \
  --slug stack-scaffold \
  --name "Stack Scaffold" \
  --version 1.1.0 \
  --changelog "Added support for Drizzle ORM as alternative to Supabase direct queries"
```

Use semantic versioning: patch (1.0.1) for fixes, minor (1.1.0) for new features, major (2.0.0) for breaking changes.

### Publishing Private vs Public Versions

For the **public ClawHub versions**, remove any team-specific conventions before publishing:
- Internal naming patterns
- Company-specific env var names
- References to internal repos or domains
- Team Slack/Discord notification URLs

Keep the **workspace versions** (Option 1) as your private, team-specific copies.

---

## Team Setup

For a team of 2-5 developers:

1. Store skills in the team's shared workspace (e.g., a `skills/` directory in your monorepo or a dedicated skills repo).
2. Each team member loads the workspace skills via OpenClaw.
3. Use `skills.load.extraDirs` in OpenClaw config to point to the shared location.
4. Version skills with git — treat them like code.

## Dependencies

- Node.js 18+
- Git
- GitHub CLI (`gh`) — `brew install gh` or `sudo apt install gh`
- Supabase CLI — available via `npx supabase` (no global install needed)
- Vercel CLI — available via `npx vercel` (no global install needed)
- curl (for Cloudflare API calls) — pre-installed on macOS and Linux
- Google Cloud SDK (`gcloud`) — https://cloud.google.com/sdk/docs/install (required for `gcp-fullstack`)
- Docker — required for Cloud Run container builds (required for `gcp-fullstack`)
- pnpm — `npm install -g pnpm` (required for `interop-forge` monorepo management)
- Python 3.10+ with pip (required for `web-scraper`)
- Playwright browsers — installed via `npx playwright install` (required for `web-scraper` JS-rendered pages)

## Integration Testing Between Skills

To validate that skills work correctly together, run these integration test sequences:

### Vercel Stack — Full Lifecycle Test

```bash
# 1. Preflight check
# Run: "preflight check for Vercel stack"
# Expected: All checks pass, all skills marked as ready

# 2. Scaffold → Auth → Theme → Deploy
# Run: "scaffold a new project called integration-test"
# Verify: Project created with correct structure
# Run: "set up Firebase auth with Google provider"
# Verify: Auth hook, provider component, sync route created
# Run: "apply shadcn default theme"
# Verify: globals.css has OKLCH variables, dark mode works
# Run: "deploy to Vercel (preview)"
# Verify: Preview URL accessible, health endpoint returns 200

# 3. Feature → Test → QA → Deploy
# Run: "create a user profile page that shows the logged-in user's info"
# Verify: Page component, API route, and types created
# Run: "write tests for the profile feature"
# Verify: Unit and E2E tests pass
# Run: "run QA gate for production"
# Verify: Go/no-go report generated with GO verdict
# Run: "deploy to production"
# Verify: Production URL accessible, all health checks pass
```

### GCP Stack — Full Lifecycle Test

```bash
# 1. Preflight check
# Run: "preflight check for GCP stack"
# Expected: All checks pass including gcloud and Docker

# 2. Scaffold → Database → Auth → Deploy
# Run: "scaffold a GCP project with Cloud Run and Firestore"
# Verify: Project structure, Dockerfile, cloudbuild.yaml created
# Run: "set up Firestore with user collection and security rules"
# Verify: Firestore rules file, admin SDK initialization
# Run: "deploy to Cloud Run"
# Verify: Service accessible, health endpoint returns 200

# 3. QA Gate
# Run: "run GCP QA gate"
# Verify: Cloud Run health, Firestore rules validation, go/no-go report
```

### Cross-Stack Test (interop-forge)

```bash
# 1. Set up monorepo
# Run: "create a monorepo with a Next.js frontend and an Express API"
# Verify: Turborepo config, shared contracts package, workspace structure

# 2. Generate contracts and SDK
# Run: "create shared types for User and Product entities"
# Verify: Zod schemas in @repo/contracts, TypeScript types exported
# Run: "generate OpenAPI spec for the API app"
# Verify: OpenAPI YAML file generated with correct endpoints
# Run: "generate typed SDK from the OpenAPI spec"
# Verify: SDK package created with typed client functions
```

These tests should be run manually after significant changes to any skill to ensure compatibility.
