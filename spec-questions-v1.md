# Bobar - Specification Clarification Questions (v1)

This document lists questions and considerations to refine the initial specification.
Each section maps to a major feature area described in `ideas.md`.

---

## 1. General / Architecture

### 1.1 Deployment & Hosting
- Is Bobar intended to be self-hosted by the developer (e.g. on their own VPS, Docker), or should it be a SaaS offering?

We want to provide a full self hostable system. The repo should provie a full production docker compose setup with instructions for deployment (like proxy config).

- Should the backend be a single deployable unit, or split into services (API, worker for GitHub sync, frontend dashboard)?

The backend architecture can be splitted if technically beneficial. no real constraints here, as we use docker.

- What is the target tech stack? Any preferences or constraints (language, framework, database)?

The backend and administration web should be a full typescript setup (Elysia, Better-auth, Bun, Tanstack, React, shadcn, valibot or zod or simillar....)
The autonomous toolbar, no idea for now, we should find the best solution for performance, simplicity and avoiding conflicts with the undelying app.

- Should the embeddable toolbar script be served from the Bobar backend, or distributed as an npm package / CDN script?

Served from the backend. with a unique key system, to allow the script to authenticate to the right app configured on the backent.

### 1.2 Multi-tenancy & Scope
- Is Bobar designed for a single GitHub repository, or should one instance support multiple repos / organizations?

Many repo. User can connect to bobar backend with SSO and add a new repo for feedback. They will need to add a github token for the backend to interact with github. The architecture should allow us to easily add suport for other providers one day.

- Can multiple projects (repos) share the same Bobar backend with separate configurations?

Yes, for eyample, one company should be able to have one Bobar deployment for all their developed applications.

### 1.3 Authentication & Users
- Who are the distinct user roles?
  - **Admin/Dev**: configures Bobar, manages global forms and default config. have all acess.
  - **Dev**: add new repos for use with bobar, add correspondig github api key, manages repos, labels, forms.
  - **Tester/End-user**: uses the toolbar to leave feedback, votes on features.

- Does the tester need to authenticate? If yes, how? (GitHub OAuth, email/password, magic link, anonymous with optional identity?)

Configurable. We should be able to choose between authentication with magic links (specifying the email domains accepted for this app, to restrict authentication). Or no authentication.

- For the voting feature, do we need to prevent duplicate votes? If so, how do we identify unique users (authentication required, or cookie/fingerprint-based)?

Authentication based if used, or cookie based.

- Should the admin authentication use GitHub OAuth (since the app is GitHub-centric), or a separate auth system?

This should be configured by the admin who makes the Bobar deployment... any SSO available from better-auth.

### 1.4 GitHub Integration
- Should Bobar use a GitHub App (installed on the repo/org) or a personal access token? A GitHub App would be cleaner for permissions and multi-repo support.

The idea is to have one specific token per app. To ensure fine grained access. not sure of the better approach. please propose solutions.

- What GitHub API permissions are needed? At minimum: read/write issues, read/write PR comments, read releases, read milestones. Anything else?

Exactly

- Should we handle GitHub API rate limits gracefully (caching, webhooks instead of polling)?

Yes we should cache data and use webhook for updates (if possible).

- Should data (releases, milestones, issues) be cached/synced locally, or always fetched live from GitHub?

cached

---

## 2. Feedback Toolbar

### 2.1 Script Integration
- How does the developer include the toolbar? A simple `<script>` tag with a config object? Something like:
  ```html
  <script src="https://bobar.example.com/toolbar.js" data-project="my-repo"></script>
  ```
Yes a simple defered script.

- Should the toolbar be framework-agnostic (vanilla JS), or also provide React/Vue/Svelte wrappers?

Should be compatible with any application.

- Should the toolbar be removable automatically in production builds (e.g. via environment variable detection), or is it the developer's responsibility to conditionally include it?

It is developper responsability.

### 2.2 Deployment Detection
- How does the toolbar determine the deployment type (PR deploy vs. staging vs. production)? Possible approaches:
  - URL pattern matching configured by the dev (e.g. `pr-{number}.preview.example.com` -> PR deploy)
  - Environment variable injected at build time
  - HTTP header or meta tag
  - Manual configuration in the script tag

URL patern configured by the dev. We should be able to indicate the root of the url, to ensure the app is the right one, and add pattern for the pr part and other staging or dev deployments.
The dev also should configure the app url to manage CORS.

- For PR deployments, how do we extract the PR number from the URL? This varies greatly between hosting providers (Vercel, Netlify, Cloudflare Pages, custom). Should we support a set of known providers + a custom pattern?

For now, only url with pr id on it will be suported.

### 2.3 Screenshot & Element Selection
- For screenshots: should we use a client-side approach (html2canvas, dom-to-image) or a server-side one (headless browser on backend)? Client-side is simpler but less accurate; server-side requires the backend to access the deployment.

If possible a client side solution.

- When the user clicks an element, what exactly do we capture?
  - CSS selector path? Yes
  - XPath? Yes
  - Element tag, classes, id, text content? Yes
  - Bounding box coordinates relative to the viewport? Yes
  - Timestamp
- Should the user be able to annotate the screenshot (draw arrows, highlight areas, add text) before submitting? No
- Should we capture additional context automatically? Yes, browser info, viewport size, console errors, network errors, current URL/route
- What is the maximum screenshot/payload size we should handle? Screenshots can be large.
1mo

### 2.4 Feedback Form
- The spec mentions configurable checkboxes/selects. Can you give examples of the kind of fields a dev might want?
Examples :
  - Severity (critical / major / minor / cosmetic)
  - Category (UI bug / functional bug / content error / suggestion)

- Should the form support conditional fields (show field B only if field A has value X)?
Not for now.

- Should fields have types beyond text, checkbox, and select? (e.g. radio, number, file upload for additional screenshots?)
Yes : text, checkbox, select, radio, file upload

- Is there a maximum number of configurable fields? No
- Should the form support i18n (multiple languages)? Yes

### 2.5 GitHub Output (PR comment / Issue)
- For PR comments: what format should the comment have? Should it use a specific template? Should it include an embedded screenshot (uploaded as image to GitHub), or a link to the screenshot hosted on the Bobar backend?

Not clear for now, the comment should list the user feedback, then display the metadatas collected and images. All datas must be uploaded to GitHuub.

- For issue creation (non-PR deployments): should the issue be assigned to anyone by default? Should it use a specific issue template?

Issue with same form as comment. No assignement, but a label that can be defined in the Bobar backend for this app. And a label for the type of deployment. Example an issue from staging deployment can have, the label "staging" and the label "user feedback".

- Should the toolbar allow the user to see previous feedback for the same page/deployment, or is it purely for submitting new feedback?
Purely submitting feedbacks.

- If the same user submits multiple feedbacks on the same PR, should they be grouped in a single comment or posted as separate comments?

No, always a new comment

- Should there be a way to track feedback status (open/resolved) from within the toolbar?
No

---

## 3. Roadmap & Changelog

### 3.1 Releases Timeline
- Should the releases page be a standalone public page (e.g. `bobar.example.com/releases/my-repo`), or embedded within the developer's app?
Can be set as public or private, managed by the developer app

- How should releases be displayed? A vertical timeline? Cards? Grouped by date/version?
A vertical timeline. With release title, date and changelog on the left, issues resolved on the right.

- Should we parse release notes as markdown and render them?
Yes

- Should we show release assets (downloads)?
No

- How do we associate "resolved issues" with a release? Via:
issues between the release and the previous one. only issues with labels configured in the backend.

- Should there be filtering/search capabilities on the timeline?
Yes, a global seach is a good idea, to quickly find news.

### 3.2 Milestone Progress
- Should milestone display be a standalone page, or a widget/component?
Standalone page. Only one page per app, displaying the next milestone.

- What information should be shown for each milestone?
  - Title, due date
  - Progress bar (open vs closed issues)
  - List of issues with their status
- Should only the current/active milestone be shown, or all open milestones?
Only the current
- How do we determine which milestone is "current"?
Nearest due date

### 3.3 Public vs. Private
- Are the roadmap/changelog pages meant to be fully public, or can access be restricted?
Configurable.
- Should there be any content filtering (e.g. hide issues with certain labels from the public view)?
Yes, configurable
- How should we handle private repositories? The public page would need authenticated access to the GitHub API.
The app will have a token to access github. the private/public view of pages it is configurable on the app

---

## 4. Feature Voting

### 4.1 Issue Selection
- The spec mentions a "vote" label. Should this be a single configurable label, or could there be multiple labels for different categories of votable features?
Only one for now
- Should the voting page show any grouping/categorization of issues?
No
- Should closed/completed votable issues still be visible (as "shipped")?
Yes, when a voted one is closed as completed.
Also display if closed as not planned.

### 4.2 Voting Mechanism
- Upvote only, or upvote + downvote?
With tree level of priorities, like necessary, interesting, not needed
- Should users be able to leave comments alongside their vote?
No
- How are votes stored? The spec mentions updating the issue with vote counts. Specifically:
In the Bobar database, then synced back in the issue body.
- Should vote counts be real-time or periodically synced?
Periodycal sync, like with a debounce of 1 hour.

### 4.3 Display
- Should the voting page support sorting (most voted, newest, oldest)?
Yes
- Should there be a search/filter on the voting page?
Not for now
- Can users see who voted (transparency), or are votes anonymous?
No

---

## 5. Backend & Administration

### 5.1 Configuration UI
- What should the admin dashboard look like? Minimal settings page, or a full management interface?
minimal, only needed. shadcn ui style.
- Should the admin be able to:
  - Preview the toolbar and feedback form?
  - View submitted feedback history?
  - Moderate votes?
  - See analytics (number of feedbacks, votes over time)?
Nothing for now, the app only serve to sync these infos to github.

### 5.2 Feedback Form Builder
- How complex should the form builder be? A simple list of fields with type/label/options, or a full drag-and-drop form designer?
Simple list of fiels, with a + button to add fiels.
- Should there be form templates (e.g. "Bug Report", "Feature Request") that the dev can choose from?
No
- Can different deployment environments have different forms? (e.g. a simpler form for production, a detailed one for staging)
Not for now

### 5.3 URL Pattern Configuration
- How should the dev configure the deployment URL patterns? Examples:
  - `pr-*.preview.example.com` -> PR deployment, extract PR number
  - `staging.example.com` -> staging environment
  - `example.com` -> production
Exactly. the toolbar shoult be addable only ont the configured environnement with corresponding pattern.
- Should Bobar provide presets for common hosting providers (Vercel, Netlify, etc.)?
No

---

## 6. Non-functional Requirements

### 6.1 Performance
- The toolbar script must be lightweight. What is the acceptable bundle size? (e.g. < 50KB gzipped?)
Minimal
- Should the script load asynchronously to avoid blocking the host app?
Yes
- Should screenshots be compressed before upload?
Maybe, but not a priority.

### 6.2 Security
- How do we prevent abuse of the toolbar (spam feedback, XSS in feedback content)?
Rate limiting per fingerprint. Most of the time app will only be accesible ont local networks or with vpn.
- Should the toolbar script require an API key, or rely on domain allowlisting?
Allowlist, only load if the host correspont to the one configured in the app, and api key
- How do we handle CORS for the toolbar -> backend communication?
When creating the app on the backend, we specify the hostname.
- Should feedback content be sanitized before posting to GitHub?
Not for now

### 6.3 Privacy
- Does the toolbar capture any PII? If screenshots contain sensitive data, should there be a way to redact?
No PII,
- Should there be a GDPR consideration for the voting feature (storing user preferences/votes)?
No

### 6.4 Compatibility
- What browsers must the toolbar support? (Modern only, or IE11?)
Modern only
- Should the toolbar work on mobile / touch devices?
No
- Should the toolbar work inside iframes?
No

---

## 7. Scope & Prioritization

- Which feature should be built first? I suggest this priority order:
  1. Backend + GitHub App setup + authentication
  2. Feedback toolbar (core value proposition, most complex)
  3. Roadmap/Changelog (read-only, lower complexity)
  4. Feature voting (can be added incrementally)
Okay
- Is there a target timeline or milestone for an MVP? No
- Are there any existing tools or competitors you've looked at for reference? (Vercel Toolbar, Marker.io, Canny, ProductBoard?) No
- Should there be an API for external integrations beyond GitHub (Slack notifications, webhook on new feedback, etc.)?
Not for now
