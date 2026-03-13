# Bobar - Architecture Document (v1)

> Related documents: [features.md](./features.md) | [plan.md](./plan.md) | [spec-questions-v1.md](./spec-questions-v1.md)

---

## 1. High-Level Overview

Bobar is a self-hosted, multi-tenant application composed of three main runtime units:

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Compose                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   API Server  │  │   Dashboard  │  │  PostgreSQL   │  │
│  │   (Elysia)   │  │  (Vite SPA)  │  │              │  │
│  │   port 3000   │  │  port 5173   │  │  port 5432   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                  │                  │          │
│         └──────────────────┴──────────────────┘          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Reverse Proxy (Caddy)               │    │
│  │   bobar.example.com -> dashboard                 │    │
│  │   bobar.example.com/api/* -> api                 │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

         ▲                              ▲
         │ REST API                     │ <script> tag
         │                              │
┌────────┴────────┐           ┌─────────┴─────────┐
│   GitHub API     │           │  Toolbar Script    │
│  (issues, PRs,   │           │  (injected in      │
│   releases...)   │           │   target app)      │
└─────────────────┘           └───────────────────┘
```

### Runtime Units

| Unit | Role | Tech |
|------|------|------|
| **API Server** | REST API, auth, GitHub sync, toolbar endpoint, cron jobs, webhook receiver | Elysia + Bun |
| **Dashboard** | Admin UI, public pages (roadmap, changelog, voting) | Vite + React + TanStack Router |
| **Database** | Persistent storage | PostgreSQL |
| **Toolbar Script** | Embeddable feedback overlay served as static JS | Vanilla JS, Shadow DOM |

---

## 2. Tech Stack

### 2.1 Backend - API Server

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Runtime | **Bun** | Fast startup, native TypeScript, built-in SQLite (useful for tests) |
| Framework | **Elysia** | Type-safe, Bun-native, plugin ecosystem (CORS, cron, static) |
| ORM | **Drizzle ORM** | Type-safe schema, lightweight, great migration tooling, PostgreSQL + SQLite support |
| Database | **PostgreSQL** | Production-grade, JSON support for flexible config, Better Auth compatible |
| Auth | **Better Auth** | Framework-agnostic, SSO plugins, magic link, organization support |
| Validation | **Valibot** | Lighter than Zod (~1KB vs ~12KB), tree-shakable, same DX |
| Cron | **@elysiajs/cron** | Native Elysia plugin for scheduled GitHub sync jobs |
| CORS | **@elysiajs/cors** | Per-app origin allowlisting from configured hostnames |

### 2.2 Frontend - Dashboard

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Bundler | **Vite** | Fast HMR, Bun-compatible, ecosystem standard |
| Framework | **React** | As specified |
| Routing | **TanStack Router** | File-based routing, type-safe params/search, built-in data loading |
| Data fetching | **TanStack Query** | Cache, deduplication, background refetch for GitHub-synced data |
| UI | **shadcn/ui** | Copy-paste components, Tailwind-based, minimal overhead |
| Forms | **TanStack Form** or **React Hook Form** | For the admin form builder and config pages |

### 2.3 Toolbar Script

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Vanilla TypeScript** | No framework dependency, minimal bundle size |
| Bundler | **esbuild** (via Bun.build) | Single-file IIFE output, tree-shaking, < 30KB target |
| DOM isolation | **Shadow DOM** | Prevents CSS/JS conflicts with host application |
| Screenshots | **html2canvas** | Client-side, no server access needed. ~40KB but loaded lazily on feedback trigger |
| Styling | **Inline styles in Shadow DOM** | No external CSS to conflict |

> **Note on html2canvas limitations**: It cannot render cross-origin images unless CORS headers are present, and some CSS properties (filters, blend modes) are not fully supported. This is acceptable for feedback-quality screenshots. An alternative is `html-to-image` (uses SVG foreignObject, smaller but less compatible). We should evaluate both.

### 2.4 Infrastructure

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Container | **Docker** | Self-hosted requirement |
| Orchestration | **Docker Compose** | Single-command deployment |
| Reverse proxy | **Caddy** | Auto HTTPS, simple config, good for self-hosted |
| File storage | **Local volume** (screenshots) | Simple, no external dependency. Screenshots are uploaded to GitHub after processing |

---

## 3. Repository Structure (Monorepo)

```
bobar/
├── docker-compose.yml
├── docker-compose.dev.yml
├── Caddyfile
├── package.json                  # workspace root
│
├── packages/
│   ├── api/                      # Elysia backend
│   │   ├── src/
│   │   │   ├── index.ts          # Elysia app entry
│   │   │   ├── db/
│   │   │   │   ├── schema.ts     # Drizzle schema
│   │   │   │   └── migrate.ts
│   │   │   ├── auth/             # Better Auth config
│   │   │   ├── routes/
│   │   │   │   ├── apps.ts       # CRUD for configured apps/repos
│   │   │   │   ├── toolbar.ts    # Feedback submission endpoint
│   │   │   │   ├── github.ts     # GitHub sync, webhooks
│   │   │   │   ├── public.ts     # Public pages API (releases, milestones, votes)
│   │   │   │   └── admin.ts      # Admin config endpoints
│   │   │   ├── services/
│   │   │   │   ├── github/       # GitHub API abstraction (provider-agnostic interface)
│   │   │   │   ├── screenshot.ts # Screenshot processing
│   │   │   │   └── sync.ts       # Cron-based sync logic
│   │   │   └── lib/
│   │   │       ├── providers.ts  # Git provider interface (GitHub first, extensible)
│   │   │       └── patterns.ts   # URL pattern matching logic
│   │   ├── drizzle.config.ts
│   │   └── package.json
│   │
│   ├── dashboard/                # React SPA
│   │   ├── src/
│   │   │   ├── routes/           # TanStack file-based routes
│   │   │   │   ├── __root.tsx
│   │   │   │   ├── index.tsx                    # Landing / login
│   │   │   │   ├── admin/
│   │   │   │   │   ├── apps.tsx                 # List configured apps
│   │   │   │   │   ├── apps.$appId.tsx          # App config (form builder, URLs, labels)
│   │   │   │   │   └── settings.tsx             # Global settings
│   │   │   │   └── public/
│   │   │   │       ├── $appSlug/
│   │   │   │       │   ├── releases.tsx         # Public releases timeline
│   │   │   │       │   ├── roadmap.tsx          # Current milestone
│   │   │   │       │   └── vote.tsx             # Feature voting page
│   │   │   ├── components/
│   │   │   └── lib/
│   │   │       └── api.ts        # API client (typed, from Elysia Eden or manual)
│   │   └── package.json
│   │
│   ├── toolbar/                  # Embeddable script
│   │   ├── src/
│   │   │   ├── index.ts          # Entry: init, auth check, URL validation
│   │   │   ├── ui/               # Shadow DOM UI components
│   │   │   ├── capture.ts        # Screenshot + element selection logic
│   │   │   ├── context.ts        # Browser info, console errors, network errors
│   │   │   └── api.ts            # Communication with Bobar API
│   │   ├── build.ts              # Bun.build config -> single IIFE file
│   │   └── package.json
│   │
│   └── shared/                   # Shared types and utilities
│       ├── src/
│       │   ├── types.ts          # App config, feedback payload, form field definitions
│       │   ├── patterns.ts       # URL pattern matching (shared between toolbar and API)
│       │   └── validation.ts     # Valibot schemas shared between API and dashboard
│       └── package.json
│
└── deploy/
    ├── Dockerfile.api
    ├── Dockerfile.dashboard
    └── Caddyfile.example
```

### Monorepo Tooling

Use **Bun workspaces** for dependency management. No need for Turborepo or Nx at this scale — Bun workspaces + simple scripts are sufficient.

```json
// root package.json
{
  "workspaces": ["packages/*"]
}
```

---

## 4. Database Schema (Drizzle ORM)

### Core Entities

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
│    users      │       │      apps         │       │  app_members     │
│──────────────│  1──N  │──────────────────│  N──N  │──────────────────│
│ id           │◄──────►│ id               │◄──────►│ app_id           │
│ email        │       │ name              │       │ user_id          │
│ role (global)│       │ slug              │       │ role             │
│ ...          │       │ repo_url          │       └──────────────────┘
└──────────────┘       │ provider (github) │
                       │ access_token (enc)│       ┌──────────────────┐
                       │ webhook_secret    │       │  url_patterns    │
                       │ feedback_label    │  1──N  │──────────────────│
                       │ vote_label        │◄──────►│ app_id           │
                       │ public_releases   │       │ pattern          │
                       │ public_roadmap    │       │ env_type         │
                       │ public_votes      │       │ (pr|staging|prod)│
                       │ allowed_origins   │       └──────────────────┘
                       │ locale_default    │
                       └──────────┬───────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │ 1──N        │ 1──N        │ 1──N
          ┌─────────▼──┐  ┌──────▼───────┐  ┌──▼───────────┐
          │ form_fields │  │ cached_data  │  │    votes      │
          │────────────│  │──────────────│  │──────────────│
          │ app_id     │  │ app_id       │  │ app_id       │
          │ type       │  │ data_type    │  │ issue_number │
          │ label      │  │ (release,    │  │ user_id /    │
          │ options    │  │  milestone,  │  │ fingerprint  │
          │ required   │  │  issue)      │  │ value        │
          │ position   │  │ data (jsonb) │  │ (necessary,  │
          │ i18n_labels│  │ synced_at    │  │  interesting,│
          └────────────┘  └──────────────┘  │  not_needed) │
                                            │ synced_at    │
                                            └──────────────┘
```

### Key Design Decisions

- **`access_token`**: Encrypted at rest (AES-256-GCM with app-level secret). Never exposed via API.
- **`cached_data`**: JSONB column storing GitHub responses. Avoids modeling every GitHub entity as a table. Indexed by `(app_id, data_type)`.
- **`form_fields.i18n_labels`**: JSONB storing `{ "en": "Severity", "fr": "Sévérité" }` for i18n support.
- **`votes.value`**: Enum with three levels: `necessary`, `interesting`, `not_needed`.
- **`url_patterns`**: Separate table to allow multiple patterns per app (one for PR, one for staging, etc).

---

## 5. Authentication Architecture

### Admin / Dev Authentication (Better Auth)

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Dashboard   │────►│  Better Auth  │────►│  SSO / OIDC │
│  (React)     │     │  (Elysia)    │     │  Provider   │
└─────────────┘     └──────────────┘     └─────────────┘
```

- Better Auth handles session management, SSO via configurable provider.
- Admin configures the SSO provider (Google, GitHub, OIDC, etc.) during initial setup.
- Roles: `admin` (global), `dev` (per-app), enforced via Better Auth middleware.
- Better Auth's **organization plugin** can model per-app team membership.

### Tester Authentication (Toolbar)

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Toolbar     │────►│  Bobar API    │────►│  Magic Link │
│  (script)    │     │  /auth/magic  │     │  (email)    │
└─────────────┘     └──────────────┘     └─────────────┘
```

- Configurable per app: **magic link** (with email domain restriction) or **anonymous**.
- Magic link: user enters email -> receives link -> toolbar stores session token.
- Anonymous: toolbar generates a fingerprint cookie for vote deduplication.
- The toolbar uses a lightweight auth flow (no full Better Auth client needed).

---

## 6. GitHub Integration Architecture

### Provider Abstraction

To allow future support for GitLab, Gitea, etc., the GitHub logic is behind an interface:

```typescript
// packages/shared/src/types.ts
interface GitProvider {
  createIssue(opts: CreateIssueOpts): Promise<Issue>
  createPRComment(opts: CreatePRCommentOpts): Promise<Comment>
  uploadImage(opts: UploadImageOpts): Promise<string> // returns URL
  listReleases(): Promise<Release[]>
  listMilestones(): Promise<Milestone[]>
  listIssues(labels: string[]): Promise<Issue[]>
  updateIssueBody(issueNumber: number, body: string): Promise<void>
  registerWebhook(url: string, events: string[]): Promise<void>
}
```

### Token Strategy

**Recommended: Fine-grained Personal Access Tokens (PATs)**

- One token per configured app/repo.
- Dev creates a fine-grained PAT scoped to the specific repo with permissions: `issues: read/write`, `pull_requests: read/write`, `contents: read` (for releases).
- Token stored encrypted in the database.

**Future consideration: GitHub App**
- Would allow automatic installation per org, no manual token management.
- More complex initial setup but better UX for multi-repo.
- The `GitProvider` interface makes this swap transparent.

### Data Sync Strategy

```
GitHub ──webhook──► Bobar API ──► Update cached_data table
                                        │
Bobar API ──cron (hourly)──► GitHub API ──► Refresh cached_data
                                        │
Dashboard/Public pages ◄── Read from cached_data
```

- **Webhooks** (primary): On app creation, Bobar registers a webhook on the repo for events: `issues`, `pull_request`, `release`, `milestone`. Webhook URL: `bobar.example.com/api/webhooks/github/{app_id}`. Verified with webhook secret.
- **Cron** (fallback): `@elysiajs/cron` job runs hourly to refresh all cached data. Catches anything webhooks might miss.
- **Vote sync**: Separate cron job, debounced at 1 hour. Updates issue body with vote summary table.

---

## 7. Toolbar Architecture

### Loading Flow

```
1. Host app loads <script defer src="https://bobar.example.com/api/toolbar/{api_key}/script.js">
2. Script executes:
   a. Reads api_key from script src URL
   b. Calls GET /api/toolbar/{api_key}/config
      -> Returns: app config, form fields, URL patterns, allowed origins, i18n strings
   c. Validates current page URL against allowed origins
      -> If no match: script exits silently (no UI rendered)
   d. Determines deployment type from URL patterns (PR + number, staging, prod...)
   e. Creates Shadow DOM container, injects floating button
3. User clicks button:
   a. Opens feedback panel (inside Shadow DOM)
   b. If auth required and no session: shows magic link form
   c. Shows configured form fields
   d. "Select element" mode: click-to-highlight, captures element metadata
   e. On submit: takes screenshot (html2canvas, lazy-loaded), collects context
   f. POST /api/toolbar/{api_key}/feedback with all data
4. API server:
   a. Validates payload
   b. Uploads screenshot to GitHub (as issue/comment attachment)
   c. Creates PR comment or issue based on deployment type
```

### Shadow DOM Isolation

```typescript
// toolbar/src/index.ts
const host = document.createElement('div');
host.id = 'bobar-toolbar';
const shadow = host.attachShadow({ mode: 'closed' });

// All toolbar UI lives inside shadow DOM
// Styles are injected as <style> inside shadow root
// No global CSS leaks in or out
document.body.appendChild(host);
```

### Bundle Strategy

- **Core script** (~15-20KB gzipped): UI, form, API communication, element selector.
- **html2canvas** (~40KB gzipped): Lazy-loaded only when user triggers feedback submission. Loaded as a dynamic import or fetched as a separate chunk.
- Total initial load: < 25KB. Total with screenshot: < 65KB.

---

## 8. API Design (Key Endpoints)

### Toolbar API (public, API-key authenticated)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/toolbar/:apiKey/config` | Returns app config, form fields, URL patterns, i18n |
| POST | `/api/toolbar/:apiKey/feedback` | Submit feedback (multipart: JSON + screenshot file) |
| POST | `/api/toolbar/:apiKey/auth/magic-link` | Request magic link email |
| GET | `/api/toolbar/:apiKey/auth/verify` | Verify magic link token |

### Admin API (session authenticated)

| Method | Path | Description |
|--------|------|-------------|
| GET/POST | `/api/apps` | List / create configured apps |
| GET/PUT/DELETE | `/api/apps/:id` | Read / update / delete app config |
| GET/POST/PUT/DELETE | `/api/apps/:id/fields` | CRUD form fields |
| GET/POST/PUT/DELETE | `/api/apps/:id/patterns` | CRUD URL patterns |
| POST | `/api/apps/:id/test-token` | Validate GitHub token permissions |

### Public API (optionally authenticated per app config)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/public/:appSlug/releases` | Paginated releases with resolved issues |
| GET | `/api/public/:appSlug/roadmap` | Current milestone with issue list |
| GET | `/api/public/:appSlug/votes` | Votable issues with counts |
| POST | `/api/public/:appSlug/votes/:issueNumber` | Cast/update a vote |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/webhooks/github/:appId` | Receive GitHub webhook events |

---

## 9. Security Model

| Layer | Mechanism |
|-------|-----------|
| **Toolbar -> API** | API key in URL + origin allowlist (CORS). Rate limiting per fingerprint. |
| **Dashboard -> API** | Better Auth session cookie. CSRF protection built-in. |
| **Public pages -> API** | Optional: access controlled by app config (`public_releases`, etc.) |
| **GitHub tokens** | AES-256-GCM encrypted at rest. Decrypted only in-memory for API calls. Encryption key from environment variable. |
| **Webhooks** | HMAC-SHA256 signature verification using per-app webhook secret. |
| **CORS** | Per-app origin allowlist configured in dashboard. `@elysiajs/cors` with dynamic origin function. |

---

## 10. Open Architecture Questions

> **[Q-ARCH-1]** Should the dashboard be served by the Elysia API (as static files via `@elysiajs/static`) or as a separate container behind the reverse proxy?
> Separate container is cleaner for dev experience (Vite HMR), but single container simplifies deployment. A hybrid approach: in dev, two servers; in prod, API serves the built dashboard as static files.

> **[Q-ARCH-2]** For the toolbar's lazy-loaded html2canvas: should it be bundled as a separate chunk served from the Bobar backend, or loaded from a CDN like unpkg/esm.sh? Self-hosting is more reliable for self-hosted deployments. CDN is lighter on the Bobar server.

> **[Q-ARCH-3]** Should we use Elysia Eden (end-to-end type-safe client) for dashboard-to-API communication, or a manual typed client with TanStack Query? Eden is elegant but couples dashboard tightly to Elysia types.

> **[Q-ARCH-4]** For the GitHub image upload: GitHub supports attaching images to issues/comments via the API, but the "official" way is through their upload endpoint (which is undocumented). An alternative is to create the image as a blob in the repo (in a dedicated branch like `bobar-assets`), then reference it. Which approach do you prefer?

> **[Q-ARCH-5]** Should the toolbar auth (magic link) reuse Better Auth's magic link plugin, or be a simpler custom implementation? Better Auth's plugin is full-featured but adds weight. A minimal implementation (generate token, send email, verify) might be enough for the toolbar's needs.

> **[Q-ARCH-6]** Email delivery for magic links: should we require the admin to configure an SMTP server, or support a provider like Resend/Postmark? For self-hosted, SMTP makes most sense. Better Auth supports custom email transports.

> **[Q-ARCH-7]** The `cached_data` table uses JSONB for flexibility. An alternative is to model each GitHub entity (release, milestone, issue) as its own table with typed columns. JSONB is faster to implement but harder to query. Typed tables are more work upfront but cleaner long-term. Preference?

---

*Review the open questions above. Answers will feed into the next iteration of this document and the [features.md](./features.md) / [plan.md](./plan.md) documents.*
