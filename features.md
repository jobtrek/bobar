# Bobar - Feature Specification (v1)

> Related documents: [architecture.md](./architecture.md) | [plan.md](./plan.md) | [spec-questions-v1.md](./spec-questions-v1.md)

---

## Module Overview

| Module | Description | Users |
|--------|-------------|-------|
| **M1 - Core Backend** | Auth, app management, database, GitHub provider abstraction | Admin, Dev |
| **M2 - Feedback Toolbar** | Embeddable script for collecting feedback | Tester |
| **M3 - Roadmap & Changelog** | Public pages for releases and milestone progress | Public / Tester |
| **M4 - Feature Voting** | Public page for voting on proposed features | Public / Tester |

---

## M1 - Core Backend

### M1.1 - Initial Setup & Authentication

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M1.1.1 | Project scaffolding | Bun monorepo with workspaces, Elysia API, Vite+React dashboard, shared package | `bun dev` starts both API and dashboard, hot reload works |
| M1.1.2 | Database setup | PostgreSQL with Drizzle ORM, migration system, seed script | `bun db:migrate` runs cleanly, schema matches [architecture.md](./architecture.md) |
| M1.1.3 | Better Auth integration | Session-based auth with configurable SSO provider | Admin can log in via configured SSO, session persists |
| M1.1.4 | Role system | Global `admin` role, per-app `dev` role | Admin can promote users, devs can only access their apps |
| M1.1.5 | Docker Compose (dev) | Dev setup with hot reload, PostgreSQL, Caddy | `docker compose -f docker-compose.dev.yml up` starts everything |

### M1.2 - App Management

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M1.2.1 | Create app | Form: name, slug, repo URL, provider (GitHub) | App created in DB, slug is unique |
| M1.2.2 | Configure GitHub token | Input for fine-grained PAT, validation on save | Token is encrypted and stored, API verifies required permissions on the repo |
| M1.2.3 | Token permission check | On save, call GitHub API to verify token has required scopes | Clear error message if permissions are insufficient |
| M1.2.4 | App API key | Auto-generated unique key for toolbar authentication | Key displayed once, copyable, used in script tag |
| M1.2.5 | App deletion | Soft delete with confirmation | App hidden from list, webhook unregistered, data retained 30 days |

### M1.3 - URL Pattern Configuration

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M1.3.1 | Add URL pattern | Form: pattern string, environment type (pr/staging/prod) | Pattern saved, validated as parseable |
| M1.3.2 | PR number extraction | Pattern uses `{pr_number}` placeholder (e.g. `pr-{pr_number}.preview.example.com`) | Toolbar correctly extracts PR number from matching URLs |
| M1.3.3 | Origin allowlist | Derive CORS allowed origins from configured patterns | API responds with correct CORS headers for toolbar requests |
| M1.3.4 | Pattern tester | Input field in admin to test a URL against configured patterns | Shows: "matches PR #42" or "matches staging" or "no match" |

### M1.4 - GitHub Sync

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M1.4.1 | Webhook registration | On app creation, register webhook on GitHub repo | Webhook registered with correct events and secret |
| M1.4.2 | Webhook receiver | Endpoint to receive and verify GitHub webhook payloads | Events update `cached_data` table, invalid signatures rejected |
| M1.4.3 | Cron full sync | Hourly job to refresh releases, milestones, labeled issues | `cached_data` is fresh even if webhooks fail |
| M1.4.4 | Manual sync trigger | Button in admin to force-refresh cached data for an app | Data refreshed immediately, button shows loading state |

### M1.5 - Feedback Form Builder

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M1.5.1 | Field list UI | Ordered list of form fields with type, label, options | Fields displayed in configured order |
| M1.5.2 | Add field | "+" button, field types: text, checkbox, select, radio, file upload | New field appended, immediately editable |
| M1.5.3 | Edit field | Inline editing of label, type, options (for select/radio), required flag | Changes saved on blur/confirm |
| M1.5.4 | Reorder fields | Drag handle or up/down arrows to change field position | Order persisted, toolbar renders in same order |
| M1.5.5 | Delete field | Remove with confirmation | Field removed, toolbar no longer shows it |
| M1.5.6 | i18n labels | Per-field label translations as key/value pairs (locale -> label) | Toolbar displays labels in configured/detected locale |
| M1.5.7 | Label configuration | Configure labels for feedback issues/comments (e.g. "user feedback", "staging") | Labels applied to created issues |

---

## M2 - Feedback Toolbar

### M2.1 - Script Loading & Initialization

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M2.1.1 | Script serving | API serves toolbar JS at `/api/toolbar/:apiKey/script.js` | `<script defer src="...">` loads and executes without errors |
| M2.1.2 | Config fetch | On load, toolbar fetches app config (fields, patterns, i18n, auth mode) | Config loaded, toolbar has all info to render |
| M2.1.3 | URL validation | Compare `window.location` against configured patterns | Toolbar renders only on matching URLs, silent exit otherwise |
| M2.1.4 | Deployment detection | Determine environment type and extract PR number if applicable | Toolbar knows if it's PR #42, staging, or prod |
| M2.1.5 | Shadow DOM mount | Create closed Shadow DOM, inject styles and floating button | Button visible, no style conflicts with host app |

### M2.2 - Tester Authentication

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M2.2.1 | Anonymous mode | No auth required, fingerprint cookie generated | Tester can submit feedback immediately |
| M2.2.2 | Magic link request | Email input, domain validation, send magic link | Email sent only for allowed domains, clear error otherwise |
| M2.2.3 | Magic link verification | Click link -> toolbar stores session token | Tester authenticated, token persists across pages |
| M2.2.4 | Session persistence | Token stored in localStorage scoped to Bobar domain | Tester stays logged in across page navigations |

### M2.3 - Feedback Capture

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M2.3.1 | Feedback panel | Clicking floating button opens slide-out panel inside Shadow DOM | Panel opens/closes smoothly, doesn't affect host layout |
| M2.3.2 | Dynamic form rendering | Render configured fields (text, checkbox, select, radio, file) with i18n labels | All field types work, labels match detected locale |
| M2.3.3 | Element selection mode | "Select element" button -> cursor changes, hovering highlights elements, click captures | Captured data: CSS selector, XPath, tag/class/id, text content, bounding box |
| M2.3.4 | Screenshot capture | On submit, html2canvas captures current page (lazy-loaded) | Screenshot taken as PNG, respects 1MB max payload |
| M2.3.5 | Context collection | Auto-collect: browser UA, viewport size, current URL, timestamp | Metadata attached to feedback payload |
| M2.3.6 | Console/network errors | Capture recent console.error and failed network requests | Last N errors included in context (configurable N, default 10) |
| M2.3.7 | File upload | User can attach additional files (images) via file input | Files included in submission, type/size validated client-side |
| M2.3.8 | Submit feedback | POST to API with form data + screenshot + context + element data | Success/error feedback shown in toolbar UI |

### M2.4 - GitHub Output

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M2.4.1 | PR comment creation | For PR deployments: create comment on the PR with feedback content | Comment visible on GitHub PR with structured content |
| M2.4.2 | Issue creation | For non-PR deployments: create issue with configured labels | Issue created with feedback label + environment label |
| M2.4.3 | Image upload to GitHub | Screenshot and attached files uploaded as GitHub assets | Images embedded in comment/issue body, no external hosting |
| M2.4.4 | Comment/issue template | Structured markdown: user message, form fields, metadata table, screenshot, element info | Readable, well-formatted output on GitHub |

---

## M3 - Roadmap & Changelog

### M3.1 - Releases Timeline

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M3.1.1 | Releases page | Standalone page at `/public/:appSlug/releases` | Page loads and displays release data |
| M3.1.2 | Vertical timeline layout | Left side: release title, date, rendered changelog (markdown). Right side: resolved issues | Timeline scrollable, visually clear |
| M3.1.3 | Issue association | Show issues closed between two consecutive releases, filtered by configured labels | Only relevant issues shown per release |
| M3.1.4 | Markdown rendering | Parse and render GitHub release body as rich HTML | Formatting, links, code blocks render correctly |
| M3.1.5 | Search | Global text search across release titles, bodies, and issue titles | Results highlight matches, filter timeline in real-time |
| M3.1.6 | Access control | Page public or private based on app config | Private pages require tester auth, public pages open |
| M3.1.7 | Content filtering | Hide issues with configured "hidden" labels from public view | Filtered issues never appear on public page |

### M3.2 - Milestone / Roadmap

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M3.2.1 | Roadmap page | Standalone page at `/public/:appSlug/roadmap` | Page loads and displays milestone data |
| M3.2.2 | Current milestone detection | Auto-select milestone with nearest future due date | Correct milestone displayed |
| M3.2.3 | Progress bar | Visual bar showing % of closed vs total issues | Accurate percentage, updates on sync |
| M3.2.4 | Issue list | List all issues in milestone with status (open/closed) and title | Issues grouped or sorted by status |
| M3.2.5 | Due date display | Show milestone due date with relative time ("in 12 days") | Date formatted clearly |
| M3.2.6 | Access control | Same public/private config as releases | Consistent behavior |

---

## M4 - Feature Voting

### M4.1 - Voting Page

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M4.1.1 | Voting page | Standalone page at `/public/:appSlug/vote` | Page loads and displays votable issues |
| M4.1.2 | Issue list | Display all open issues with the configured vote label | Only labeled issues shown, title + body preview |
| M4.1.3 | Closed issues display | Show completed issues as "shipped", not-planned as "declined" | Visual distinction between statuses |
| M4.1.4 | Three-level voting | Per issue: "Necessary", "Interesting", "Not needed" buttons/radios | User can select one option per issue |
| M4.1.5 | Vote persistence | Store vote in DB, linked to user (auth) or fingerprint (anonymous) | Vote persists across sessions, one vote per user per issue |
| M4.1.6 | Vote change | User can change their vote | Previous vote replaced, counts updated |
| M4.1.7 | Vote display | Show aggregated counts per level for each issue | Counts accurate, update immediately on vote |
| M4.1.8 | Sorting | Sort by: most voted (total), newest, oldest | Sort persists in URL search params |
| M4.1.9 | Anonymous votes | Votes anonymous to other users, no "who voted" visible | No user info exposed in public API |

### M4.2 - Vote Sync to GitHub

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| M4.2.1 | Vote summary format | Generate markdown table: `| Necessary | Interesting | Not needed |` with counts | Clear, readable summary |
| M4.2.2 | Issue body update | Append/replace vote summary section in issue body | Summary updated without destroying existing body content |
| M4.2.3 | Debounced sync | Cron job syncs votes to GitHub, max once per hour per app | No excessive API calls, data eventually consistent |

---

## Cross-Cutting Features

### CC.1 - Docker & Deployment

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| CC.1.1 | Production Docker Compose | Full setup: API, dashboard, PostgreSQL, Caddy | `docker compose up` starts production system |
| CC.1.2 | Environment configuration | `.env.example` with all required variables documented | Admin can configure by copying and editing `.env` |
| CC.1.3 | Caddy configuration | Example Caddyfile with auto-HTTPS, proxy rules | HTTPS works out of the box with a domain |
| CC.1.4 | Health check endpoint | `GET /api/health` returns DB and GitHub connectivity status | Docker can use for container health checks |

### CC.2 - i18n

| ID | Feature | Description | Done when |
|----|---------|-------------|-----------|
| CC.2.1 | Toolbar locale detection | Detect browser locale, fallback to app default | Toolbar labels match user's language if configured |
| CC.2.2 | Form field i18n | Per-field label translations | Labels rendered in detected locale |

---

## Open Feature Questions

> **[Q-FEAT-1]** For M2.3.6 (console/network error capture): Should we capture errors that occurred *before* the toolbar loaded (by monkey-patching `console.error` early), or only errors from the moment the toolbar initializes? Early capture requires the script tag to be placed high in the `<head>`.

> **[Q-FEAT-2]** For M2.4.3 (image upload to GitHub): GitHub's documented API for uploading images to comments is limited. The common approach is to use the `repos/{owner}/{repo}/git/blobs` endpoint to create a blob, then reference it. Alternatively, images can be uploaded via the GitHub upload API (used by the web UI, but undocumented). Should we use the blob approach (reliable, documented) or investigate the upload API?

> **[Q-FEAT-3]** For M1.5.6 (i18n): How should the admin configure available locales? A global list of supported locales for the entire Bobar instance? Or per-app locale configuration?

> **[Q-FEAT-4]** For M4.1.7 (vote display): Should the vote counts be shown as raw numbers, percentages, or a visual bar chart? All three?

> **[Q-FEAT-5]** For M3.1.3 (issue association): "Issues closed between two releases" requires knowing the release dates. Should we use the release `published_at` date, the tag creation date, or the commit date of the tagged commit? `published_at` is most intuitive but can be edited after the fact.

> **[Q-FEAT-6]** For M2.3.1 (feedback panel): What should the panel look like? A slide-out drawer from the right? A modal overlay? A small popover near the floating button? This affects how much screen real estate is available for the form.

> **[Q-FEAT-7]** For M1.2.2 (GitHub token): Should we guide the user through creating the token (link to GitHub's token creation page with pre-filled permissions), or just provide documentation?

> **[Q-FEAT-8]** For M2.1.5 (Shadow DOM): Should the floating button position be configurable (bottom-left, bottom-right, etc.) per app in the admin?

---

*Review the open questions. Answers will refine this document and feed into [plan.md](./plan.md).*
