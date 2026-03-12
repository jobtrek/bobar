# Bobar - Specification Clarification Questions (v1)

This document lists questions and considerations to refine the initial specification.
Each section maps to a major feature area described in `ideas.md`.

---

## 1. General / Architecture

### 1.1 Deployment & Hosting
- Is Bobar intended to be self-hosted by the developer (e.g. on their own VPS, Docker), or should it be a SaaS offering?
- Should the backend be a single deployable unit, or split into services (API, worker for GitHub sync, frontend dashboard)?
- What is the target tech stack? Any preferences or constraints (language, framework, database)?
- Should the embeddable toolbar script be served from the Bobar backend, or distributed as an npm package / CDN script?

### 1.2 Multi-tenancy & Scope
- Is Bobar designed for a single GitHub repository, or should one instance support multiple repos / organizations?
- Can multiple projects (repos) share the same Bobar backend with separate configurations?

### 1.3 Authentication & Users
- Who are the distinct user roles? I identify at least:
  - **Admin/Dev**: configures Bobar, manages repos, labels, forms.
  - **Tester/End-user**: uses the toolbar to leave feedback, votes on features.
- Does the tester need to authenticate? If yes, how? (GitHub OAuth, email/password, magic link, anonymous with optional identity?)
- For the voting feature, do we need to prevent duplicate votes? If so, how do we identify unique users (authentication required, or cookie/fingerprint-based)?
- Should the admin authentication use GitHub OAuth (since the app is GitHub-centric), or a separate auth system?

### 1.4 GitHub Integration
- Should Bobar use a GitHub App (installed on the repo/org) or a personal access token? A GitHub App would be cleaner for permissions and multi-repo support.
- What GitHub API permissions are needed? At minimum: read/write issues, read/write PR comments, read releases, read milestones. Anything else?
- Should we handle GitHub API rate limits gracefully (caching, webhooks instead of polling)?
- Should data (releases, milestones, issues) be cached/synced locally, or always fetched live from GitHub?

---

## 2. Feedback Toolbar

### 2.1 Script Integration
- How does the developer include the toolbar? A simple `<script>` tag with a config object? Something like:
  ```html
  <script src="https://bobar.example.com/toolbar.js" data-project="my-repo"></script>
  ```
- Should the toolbar be framework-agnostic (vanilla JS), or also provide React/Vue/Svelte wrappers?
- Should the toolbar be removable automatically in production builds (e.g. via environment variable detection), or is it the developer's responsibility to conditionally include it?

### 2.2 Deployment Detection
- How does the toolbar determine the deployment type (PR deploy vs. staging vs. production)? Possible approaches:
  - URL pattern matching configured by the dev (e.g. `pr-{number}.preview.example.com` -> PR deploy)
  - Environment variable injected at build time
  - HTTP header or meta tag
  - Manual configuration in the script tag
- For PR deployments, how do we extract the PR number from the URL? This varies greatly between hosting providers (Vercel, Netlify, Cloudflare Pages, custom). Should we support a set of known providers + a custom pattern?

### 2.3 Screenshot & Element Selection
- For screenshots: should we use a client-side approach (html2canvas, dom-to-image) or a server-side one (headless browser on backend)? Client-side is simpler but less accurate; server-side requires the backend to access the deployment.
- When the user clicks an element, what exactly do we capture?
  - CSS selector path?
  - XPath?
  - Element tag, classes, id, text content?
  - Bounding box coordinates relative to the viewport?
  - All of the above?
- Should the user be able to annotate the screenshot (draw arrows, highlight areas, add text) before submitting?
- Should we capture additional context automatically? (browser info, viewport size, console errors, network errors, current URL/route, localStorage state?)
- What is the maximum screenshot/payload size we should handle? Screenshots can be large.

### 2.4 Feedback Form
- The spec mentions configurable checkboxes/selects. Can you give examples of the kind of fields a dev might want? E.g.:
  - Severity (critical / major / minor / cosmetic)
  - Category (UI bug / functional bug / content error / suggestion)
  - Browser / device (auto-detected or user-selected)
  - Custom fields?
- Should the form support conditional fields (show field B only if field A has value X)?
- Should fields have types beyond text, checkbox, and select? (e.g. radio, number, file upload for additional screenshots?)
- Is there a maximum number of configurable fields?
- Should the form support i18n (multiple languages)?

### 2.5 GitHub Output (PR comment / Issue)
- For PR comments: what format should the comment have? Should it use a specific template? Should it include an embedded screenshot (uploaded as image to GitHub), or a link to the screenshot hosted on the Bobar backend?
- For issue creation (non-PR deployments): should the issue be assigned to anyone by default? Should it use a specific issue template?
- Should the toolbar allow the user to see previous feedback for the same page/deployment, or is it purely for submitting new feedback?
- If the same user submits multiple feedbacks on the same PR, should they be grouped in a single comment or posted as separate comments?
- Should there be a way to track feedback status (open/resolved) from within the toolbar?

---

## 3. Roadmap & Changelog

### 3.1 Releases Timeline
- Should the releases page be a standalone public page (e.g. `bobar.example.com/releases/my-repo`), or embedded within the developer's app?
- How should releases be displayed? A vertical timeline? Cards? Grouped by date/version?
- Should we parse release notes as markdown and render them?
- Should we show release assets (downloads)?
- How do we associate "resolved issues" with a release? Via:
  - Issues mentioned/closed in the release body?
  - Issues closed between two release tags (git history)?
  - GitHub's "automatically generated release notes" feature?
- Should there be filtering/search capabilities on the timeline?

### 3.2 Milestone Progress
- Should milestone display be a standalone page, or a widget/component?
- What information should be shown for each milestone?
  - Title, description, due date
  - Progress bar (open vs closed issues)
  - List of issues with their status?
- Should only the current/active milestone be shown, or all open milestones?
- How do we determine which milestone is "current"? (Nearest due date? A specific label/naming convention?)

### 3.3 Public vs. Private
- Are the roadmap/changelog pages meant to be fully public, or can access be restricted?
- Should there be any content filtering (e.g. hide issues with certain labels from the public view)?
- How should we handle private repositories? The public page would need authenticated access to the GitHub API.

---

## 4. Feature Voting

### 4.1 Issue Selection
- The spec mentions a "vote" label. Should this be a single configurable label, or could there be multiple labels for different categories of votable features?
- Should the voting page show any grouping/categorization of issues?
- Should closed/completed votable issues still be visible (as "shipped")?

### 4.2 Voting Mechanism
- Upvote only, or upvote + downvote?
- Should users be able to leave comments alongside their vote?
- How are votes stored? The spec mentions updating the issue with vote counts. Specifically:
  - As a comment on the issue?
  - In the issue body (edited programmatically)?
  - Using GitHub reactions (thumbs up/down) on the issue?
  - In the Bobar database only, with a summary synced to GitHub periodically?
- Using GitHub reactions would be elegant but limits us to GitHub-authenticated users. Is that acceptable?
- Should vote counts be real-time or periodically synced?

### 4.3 Display
- Should the voting page support sorting (most voted, newest, oldest)?
- Should there be a search/filter on the voting page?
- Can users see who voted (transparency), or are votes anonymous?
- Should there be a status indicator (planned / in progress / shipped)?

---

## 5. Backend & Administration

### 5.1 Configuration UI
- What should the admin dashboard look like? Minimal settings page, or a full management interface?
- Should the admin be able to:
  - Preview the toolbar and feedback form?
  - View submitted feedback history?
  - Moderate votes?
  - See analytics (number of feedbacks, votes over time)?

### 5.2 Feedback Form Builder
- How complex should the form builder be? A simple list of fields with type/label/options, or a full drag-and-drop form designer?
- Should there be form templates (e.g. "Bug Report", "Feature Request") that the dev can choose from?
- Can different deployment environments have different forms? (e.g. a simpler form for production, a detailed one for staging)

### 5.3 URL Pattern Configuration
- How should the dev configure the deployment URL patterns? Examples:
  - `pr-*.preview.example.com` -> PR deployment, extract PR number
  - `staging.example.com` -> staging environment
  - `example.com` -> production
- Should Bobar provide presets for common hosting providers (Vercel, Netlify, etc.)?

---

## 6. Non-functional Requirements

### 6.1 Performance
- The toolbar script must be lightweight. What is the acceptable bundle size? (e.g. < 50KB gzipped?)
- Should the script load asynchronously to avoid blocking the host app?
- Should screenshots be compressed before upload?

### 6.2 Security
- How do we prevent abuse of the toolbar (spam feedback, XSS in feedback content)?
- Should the toolbar script require an API key, or rely on domain allowlisting?
- How do we handle CORS for the toolbar -> backend communication?
- Should feedback content be sanitized before posting to GitHub?

### 6.3 Privacy
- Does the toolbar capture any PII? If screenshots contain sensitive data, should there be a way to redact?
- Should there be a GDPR consideration for the voting feature (storing user preferences/votes)?

### 6.4 Compatibility
- What browsers must the toolbar support? (Modern only, or IE11?)
- Should the toolbar work on mobile / touch devices?
- Should the toolbar work inside iframes?

---

## 7. Scope & Prioritization

- Which feature should be built first? I suggest this priority order:
  1. Backend + GitHub App setup + authentication
  2. Feedback toolbar (core value proposition, most complex)
  3. Roadmap/Changelog (read-only, lower complexity)
  4. Feature voting (can be added incrementally)
- Is there a target timeline or milestone for an MVP?
- Are there any existing tools or competitors you've looked at for reference? (Vercel Toolbar, Marker.io, Canny, ProductBoard?)
- Should there be an API for external integrations beyond GitHub (Slack notifications, webhook on new feedback, etc.)?

---

*Please review these questions and provide answers. We'll iterate on the specification based on your responses.*
