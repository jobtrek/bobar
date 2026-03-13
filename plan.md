# Bobar - Development Plan (v1)

> Related documents: [architecture.md](./architecture.md) | [features.md](./features.md) | [spec-questions-v1.md](./spec-questions-v1.md)

---

## Plan Philosophy

The plan is organized in **iterative slices**, each delivering a vertically integrated, demonstrable increment. Each slice includes backend, frontend, and infrastructure work as needed. No slice depends on unanswered questions from other documents — those are flagged separately.

---

## Slice Overview

```
Slice 0 ─ Foundation          ██░░░░░░░░░░░░░░░░░░  ~1 slice
Slice 1 ─ Feedback Core       ████████░░░░░░░░░░░░  ~3 slices
Slice 2 ─ Releases & Roadmap  ████████████░░░░░░░░  ~2 slices
Slice 3 ─ Feature Voting      ██████████████░░░░░░  ~1.5 slices
Slice 4 ─ Polish & Deploy     ████████████████████  ~1.5 slices
```

---

## Slice 0 — Foundation

**Goal**: Bootable monorepo, auth works, one app can be created.

| Step | Features ([features.md](./features.md)) | Deliverable |
|------|---------|-------------|
| 0.1 | M1.1.1 | Bun monorepo: `packages/api`, `packages/dashboard`, `packages/shared`, `packages/toolbar` (empty). Bun workspaces, dev scripts. |
| 0.2 | M1.1.2 | PostgreSQL + Drizzle ORM setup. Schema for `users`, `apps`, `app_members`, `url_patterns`, `form_fields`. First migration. |
| 0.3 | M1.1.3 | Better Auth with one SSO provider (start with email/password for dev, swap to configurable SSO later). Login/logout on dashboard. |
| 0.4 | M1.1.4 | Role system: `admin` global role. Middleware guards on admin routes. |
| 0.5 | M1.1.5 | Docker Compose dev: PostgreSQL container, API with watch mode, dashboard with Vite HMR. |
| 0.6 | M1.2.1 | Dashboard: "Create App" form (name, slug, repo URL). API CRUD for apps. |
| 0.7 | M1.2.4 | Auto-generate API key on app creation. Display in dashboard. |

**Checkpoint**: Admin can log in, create an app, see it listed, copy its API key.

---

## Slice 1 — Feedback Toolbar (Core)

**Goal**: Tester can submit feedback from the toolbar, it appears on GitHub.

### Slice 1.1 — Toolbar Shell

| Step | Features | Deliverable |
|------|----------|-------------|
| 1.1.1 | M2.1.1 | Toolbar build pipeline: `packages/toolbar` with Bun.build -> single IIFE. API serves it at `/api/toolbar/:apiKey/script.js`. |
| 1.1.2 | M2.1.2 | Config endpoint: `GET /api/toolbar/:apiKey/config` returns app config JSON. |
| 1.1.3 | M2.1.5 | Shadow DOM mount: floating button renders on page, no style leak. |
| 1.1.4 | M2.1.3, M2.1.4 | URL validation + deployment detection. Script exits on non-matching URLs. |
| 1.1.5 | M1.3.1, M1.3.2, M1.3.3 | Admin UI: URL pattern configuration with environment type and PR number placeholder. CORS derived from patterns. |

**Checkpoint**: Adding the `<script>` tag to any page shows a floating button on matching URLs, nothing on non-matching.

### Slice 1.2 — Feedback Form & Capture

| Step | Features | Deliverable |
|------|----------|-------------|
| 1.2.1 | M2.3.1 | Feedback panel UI inside Shadow DOM (opens on button click). |
| 1.2.2 | M1.5.1-M1.5.5 | Admin form builder: add/edit/reorder/delete fields. |
| 1.2.3 | M2.3.2 | Toolbar renders configured form fields dynamically. |
| 1.2.4 | M2.3.3 | Element selection mode: highlight on hover, capture metadata on click. |
| 1.2.5 | M2.3.4 | Screenshot capture with html2canvas (lazy-loaded). |
| 1.2.6 | M2.3.5, M2.3.6 | Auto-collect context: browser info, viewport, URL, console errors. |
| 1.2.7 | M2.3.7 | File upload field support in toolbar. |

**Checkpoint**: Tester can open panel, fill form, select element, and see screenshot preview. Data is collected locally.

### Slice 1.3 — GitHub Integration

| Step | Features | Deliverable |
|------|----------|-------------|
| 1.3.1 | M1.2.2, M1.2.3 | Admin: GitHub token input with permission validation. |
| 1.3.2 | — | Implement `GitProvider` interface for GitHub (see [architecture.md §6](./architecture.md#6-github-integration-architecture)). |
| 1.3.3 | M2.4.3 | Image upload to GitHub (blob API or upload API, see [Q-FEAT-2](./features.md#open-feature-questions)). |
| 1.3.4 | M2.4.4 | Comment/issue markdown template: feedback, form values, metadata, images, element info. |
| 1.3.5 | M2.4.1 | PR comment creation: toolbar submits -> API posts comment on PR. |
| 1.3.6 | M2.4.2 | Issue creation: for non-PR deployments with configured labels. |
| 1.3.7 | M1.5.7 | Admin: label configuration for feedback and environment types. |
| 1.3.8 | M2.3.8 | Full submit flow: toolbar -> API -> GitHub. Success/error feedback in toolbar UI. |

**Checkpoint**: Full feedback loop works: tester submits from toolbar -> comment/issue appears on GitHub with screenshot and metadata.

---

## Slice 2 — Releases & Roadmap

**Goal**: Public pages showing release timeline and current milestone.

### Slice 2.1 — GitHub Data Sync

| Step | Features | Deliverable |
|------|----------|-------------|
| 2.1.1 | M1.4.1 | Webhook registration on app creation. |
| 2.1.2 | M1.4.2 | Webhook receiver endpoint with HMAC verification. Updates `cached_data`. |
| 2.1.3 | M1.4.3 | Cron job (hourly) to sync releases, milestones, labeled issues. |
| 2.1.4 | M1.4.4 | Admin: manual sync button. |
| 2.1.5 | — | Public API endpoints: `GET /api/public/:appSlug/releases`, `/roadmap`. |

**Checkpoint**: GitHub data is synced locally, accessible via public API.

### Slice 2.2 — Public Pages

| Step | Features | Deliverable |
|------|----------|-------------|
| 2.2.1 | M3.1.1, M3.1.2 | Releases page: vertical timeline with release info + resolved issues. |
| 2.2.2 | M3.1.3 | Issue association: issues closed between consecutive releases, label-filtered. |
| 2.2.3 | M3.1.4 | Markdown rendering of release notes. |
| 2.2.4 | M3.1.5 | Search across releases and issues. |
| 2.2.5 | M3.1.6, M3.1.7 | Access control (public/private) + content filtering by label. |
| 2.2.6 | M3.2.1-M3.2.5 | Roadmap page: current milestone, progress bar, issue list. |
| 2.2.7 | M3.2.6 | Roadmap access control. |

**Checkpoint**: Public users can view a polished releases timeline and current milestone progress.

---

## Slice 3 — Feature Voting

**Goal**: Public voting page with sync back to GitHub.

| Step | Features | Deliverable |
|------|----------|-------------|
| 3.1 | M4.1.1, M4.1.2 | Voting page: list open issues with vote label. |
| 3.2 | M4.1.3 | Show closed issues with "shipped" / "declined" status. |
| 3.3 | M4.1.4, M4.1.5, M4.1.6 | Three-level voting UI + persistence in DB (user or fingerprint). |
| 3.4 | M4.1.7 | Display aggregated vote counts. |
| 3.5 | M4.1.8 | Sorting: most voted, newest, oldest. |
| 3.6 | M4.1.9 | Anonymous votes (no user info in public API). |
| 3.7 | M4.2.1, M4.2.2 | Vote summary markdown generation + issue body update. |
| 3.8 | M4.2.3 | Debounced cron sync (1 hour). |

**Checkpoint**: Users can vote on features, votes are synced to GitHub issue bodies.

---

## Slice 4 — Polish & Production Deploy

**Goal**: Production-ready self-hosted deployment.

### Slice 4.1 — Tester Auth & i18n

| Step | Features | Deliverable |
|------|----------|-------------|
| 4.1.1 | M2.2.1 | Anonymous mode with fingerprint cookie. |
| 4.1.2 | M2.2.2, M2.2.3, M2.2.4 | Magic link auth flow in toolbar. |
| 4.1.3 | CC.2.1, CC.2.2, M1.5.6 | i18n: locale detection, form field translations, admin UI for locale labels. |

### Slice 4.2 — Deployment & Hardening

| Step | Features | Deliverable |
|------|----------|-------------|
| 4.2.1 | CC.1.1 | Production Docker Compose: multi-stage Dockerfiles, optimized images. |
| 4.2.2 | CC.1.2 | `.env.example` with full documentation. |
| 4.2.3 | CC.1.3 | Caddyfile example with auto-HTTPS. |
| 4.2.4 | CC.1.4 | Health check endpoint. |
| 4.2.5 | — | Rate limiting on toolbar endpoints (per fingerprint). |
| 4.2.6 | — | Token encryption at rest (AES-256-GCM). |
| 4.2.7 | M1.3.4 | URL pattern tester in admin. |
| 4.2.8 | M1.2.5 | App soft deletion. |
| 4.2.9 | — | Error handling, logging, graceful shutdown. |

**Checkpoint**: `docker compose up` starts a fully functional, secure, production Bobar instance.

---

## Dependency Graph

```
Slice 0 ─── Foundation
   │
   ├──► Slice 1.1 ─── Toolbar Shell
   │       │
   │       ├──► Slice 1.2 ─── Feedback Form & Capture
   │       │       │
   │       │       └──► Slice 1.3 ─── GitHub Integration
   │       │               │
   │       │               ├──► Slice 2.1 ─── GitHub Data Sync
   │       │               │       │
   │       │               │       └──► Slice 2.2 ─── Public Pages
   │       │               │
   │       │               └──► Slice 3 ─── Feature Voting
   │       │
   │       └──► Slice 4.1 ─── Tester Auth & i18n
   │
   └──► Slice 4.2 ─── Deployment & Hardening (can run in parallel from Slice 1.3+)
```

Key insight: **Slice 4.2 (deployment)** can be worked on in parallel once the core backend is stable (after Slice 1.3). This means Docker/infra work doesn't block feature development.

---

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| html2canvas rendering quality | Screenshots may not match actual page | Evaluate `html-to-image` as alternative. Accept "good enough" quality for feedback purposes. |
| GitHub image upload API | Undocumented upload API may break | Use documented blob API as primary, store in `bobar-assets` branch. See [Q-FEAT-2](./features.md#open-feature-questions). |
| GitHub API rate limits | Sync may be throttled for repos with many issues | Webhook-first strategy, cron as fallback, aggressive caching. Conditional requests (If-None-Match). |
| Shadow DOM browser support | Some edge cases in older browsers | Target modern browsers only (Chrome 90+, Firefox 90+, Safari 15+). Documented in requirements. |
| Toolbar bundle size | html2canvas is ~40KB gzipped | Lazy-load on feedback trigger only. Core toolbar stays < 25KB. |
| Better Auth SSO setup | Admin must configure SSO provider correctly | Provide clear docs + a "test connection" button. Email/password fallback for initial setup. |

---

## Open Planning Questions

> **[Q-PLAN-1]** Should Slice 0 include a basic email/password auth as a quick start (simpler to develop/test), with SSO added in Slice 4.1? Or should SSO be the only auth method from the start? Email/password for dev bootstrapping is pragmatic but adds code we may remove later.

> **[Q-PLAN-2]** Should we invest in E2E tests early (Slice 0-1) or defer testing to Slice 4? Early testing slows velocity but catches integration issues. A compromise: unit tests for critical paths (URL pattern matching, GitHub provider), E2E deferred.

> **[Q-PLAN-3]** For the toolbar, should we build a small dev playground (a test page where we can load the toolbar during development) as part of Slice 1.1? This would speed up toolbar development significantly.

> **[Q-PLAN-4]** Should the admin dashboard support dark mode from the start (shadcn/ui supports it natively) or defer it?

> **[Q-PLAN-5]** Is there a preference for the public pages design? Should they match the admin dashboard style (shadcn), or have their own design identity? The public pages are user-facing and could benefit from a more polished/branded look, but that's more design work.

---

*Answer the open questions from this document, [architecture.md](./architecture.md), and [features.md](./features.md). We'll consolidate answers and start implementation.*
