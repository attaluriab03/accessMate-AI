# AccessMate (A11y Copilot) — **GitHub App** System Architecture (v0.2)

## 1 Overview

**AccessMate** is a GitHub App + web platform that analyzes repositories and pull requests for accessibility issues (WCAG/ADA), explains impact in plain English (with citations), and can auto‑generate remediation commits/PRs (with unit/E2E tests and before/after Lighthouse evidence). It combines deterministic static checks (DOM/CSS/JS rules), dynamic audits (Playwright + axe + Lighthouse), computer vision (contrast, OCR), and LLM/RAG pipelines.

### Goals

* Near‑real‑time a11y feedback inside GitHub on every push/PR.
* High precision on common, fixable issues; precise WCAG citations.
* One‑click remediation via auto‑generated commits/PRs + tests + CI audit.
* Org‑scale: multiple repos/projects, RBAC, and policy controls.

### Non‑Goals

* Replacing expert audits for complex flows, legal interpretations, or AT usability.
* Fixing deep application logic or architectural defects.

---

## 2 Core User Roles & Flows

**Roles:** Developer, QA/Designer, Admin (Org/Project).

**Primary flows:**

1. **PR Checks** → GitHub App runs on PR events → Adds "AccessMate Checks" with findings, WCAG cites, and suggested fixes.
2. **Auto‑Fix** → User clicks "Create Fix PR" (from a PR comment, Checks UI, or dashboard) → Backend generates patches & tests → Opens a new PR or pushes to a bot branch → Kicks CI/Lighthouse → Reports back in PR thread & dashboard.
3. **Batch/Repo Scan** → Admin schedules scans (e.g., nightly on default branch) → Crawl configured routes/sitemaps → Trend reports & CSV/JSON exports in web app.
4. **Captioning** → On media changes or by command → Whisper job → Commit .vtt and markup updates via PR.

---

## 3 High‑Level Architecture

**GitHub Surfaces (Client Tier)**

* **GitHub App**: Handles webhooks (installation, push, PR opened/synchronize/reopened, check\_rerun, issue\_comment commands). Publishes GitHub Checks, PR comments, review suggestions.
* **Web App (Next.js)**: Org dashboard, findings, bot PRs, policies, schedules, routes/sitemaps, secrets, and audit artifacts.

**Service Tier**

* **Webhook/API Gateway** (Node/Express or FastAPI): Validates signatures, rate limits, org/project RBAC, queues work.
* **Analysis Orchestrator**: Coordinates static & dynamic analyzers, CV, and LLM/RAG; aggregates findings; synthesizes patches.
* **Static Analyzer**: Repo‑aware rules (component/library patterns, lint rules, code‑to‑a11y heuristics). AST tools (ts‑morph/recast/eslint custom rules), CSS analyzers, template engines (React/Next, Vue, plain HTML).
* **Dynamic Audit Runner**: Ephemeral Playwright workers running axe-core + Lighthouse with user‑configured routes, auth fixtures, and device profiles.
* **CV Service**: Contrast ratio/luminance, color extraction, OCR fallback, color‑blindness simulations.
* **Captioning Service**: Whisper pipeline for media; stores VTT/SRT; emits code diffs.
* **RAG Service**: WCAG corpus embeddings and retrieval with exact paragraph cites and remediation patterns.
* **PR/SCM Service**: GitHub integration for branches/PRs/commits, review comments, status updates, and artifact uploads.

**Data Tier**

* **PostgreSQL**: Users, orgs, installations, projects(repos), policies, scans, findings, decisions, PRs, schedules.
* **Object Store (S3/GCS)**: Audit artifacts (DOM snapshots, screenshots, videos), Lighthouse reports, Whisper assets.
* **Vector Store (pgvector/Pinecone)**: WCAG embeddings + internal remediation snippets.
* **Redis**: Job queues (BullMQ/RQ), caches, rate‑limits, idempotency keys.

---

## 4 Detailed Components

### 4.1 GitHub App & Events

* **Permissions (least‑privilege):**

  * Checks: read/write; Pull requests: read/write; Contents: read; Commit status: read/write; Issues/Comments: read/write (optional); Projects: read (optional); Workflows: read (for artifacts if needed).
* **Triggers:**

  * `pull_request` (opened, synchronize, reopened), `push` to default branch (scheduled scans), `check_suite` (rerequested), `issue_comment` ("/accessmate …" commands), `installation` events.
* **Outputs:**

  * GitHub Checks with annotations, summaries, and WCAG citations.
  * PR review comments on specific hunks/selectors.
  * Labels (e.g., `a11y:needs-fix`, `a11y:auto-fixed`).
  * Optional "Suggested changes" on markdown/docs.

### 4.2 Web App (Next.js + Tailwind)

* Repo list (per installation), policy settings, route sets/sitemaps, secrets (auth cookies/headers for Playwright), device profiles, schedules.
* Triage: *Fix now*, *Accept as Risk*, *Ignore (with rationale)*.
* Reports: trend charts, coverage, top rule failures, PR merge‑rate, Lighthouse deltas, token spend.

### 4.3 Webhook/API Gateway

* Validates GitHub webhook signatures; resolves installation → org/project; enqueues jobs.
* Endpoints (internal/public hybrid):

  * `POST /webhooks/github` (single entrypoint)
  * `POST /v1/runs` (manual/kickoff), `GET /v1/runs/{id}`
  * `POST /v1/fixes` (generate patch for finding(s))
  * `POST /v1/prs` (open PR)
  * `GET /v1/projects/{id}/reports`
* Cross‑cutting: JWT session, org/project RBAC, audit logging, usage metering.

### 4.4 Analysis Orchestrator (Workers)

**Pipeline:**

1. **Checkout** shallow clone (sparse when possible) using installation token.
2. **Discover** app framework (Next, Vue, Svelte, Rails+ERB, Django templates) and build output paths.
3. **Static analysis**: ESLint custom rules, template parsers, CSS cascade map, landmark/role inference.
4. **Route harvesting**: From framework config, sitemap, or user‑provided list.
5. **Dynamic audit**: Launch Playwright; run axe-core + Lighthouse; capture DOM/screenshot per route/device/theme.
6. **CV hooks**: Contrast/luminance, OCR fallback for images/canvas; color‑blind previews.
7. **LLM+RAG**: Explain issues, synthesize minimal patches, cite WCAG sections/techniques.
8. **Test generation**: Playwright/RTL tests for regressions; axe assertions where applicable.
9. **Outputs**: Findings (severity, impact, repro, selector/code span, patch diff, test stubs), artifacts, and a `Run` summary.

### 4.5 Patch Generation & Safety

* **Patch strategies:**

  * AST‑based edits for JSX/TSX/TS/JS (ts‑morph/recast), templating for HTML/ERB/Jinja.
  * Style fixes via CSS vars/tokens; contrast fixes prefer design‑token adjustments.
  * Content fixes (alt text/aria‑label) constrained by context, never hallucinate selectors.
* **Guardrails:**

  * Dry‑run build (`pnpm build`/`npm run build` or framework‑specific) to ensure diffs compile.
  * Unit/E2E tests updated/added; pre‑commit hooks (lint/format).
  * Patch scope limited to files implicated by findings; require reviewer approval for semantics.

### 4.6 Dynamic Audit Runner

* **Playwright** with project profiles (mobile/desktop, light/dark); auth setup via stored secrets (cookies/headers/login scripts).
* **axe-core** for rules/annotations; **Lighthouse** for before/after deltas; artifacts uploaded to S3.
* **Flake controls**: retries, timeouts, network shaping for determinism.

### 4.7 Captioning Service (Whisper)

* Media diff detection (changed/added assets) or on command → transcode → Whisper inference → VTT/SRT → commit changes + update markup.
* Optional language ID, translation, profanity filtering.

### 4.8 RAG Service (WCAG Corpus)

* Ingest WCAG 2.2+, ARIA APG, techniques; chunk by headings; store doc+paragraph refs.
* Retrieval conditioned on rule type + code context; always return cites in Checks output and PR body.

### 4.9 PR/SCM Service

* Uses GitHub App installation access tokens (rotated, short‑lived).
* Branch naming: `accessmate/{project}/{rule}/{findingId}`.
* Commit includes: code diff, tests, CHANGELOG/README note when needed.
* PR body: before/after screenshots, Lighthouse deltas, WCAG cites, checklist per finding; links to artifacts.

---

## 5 Data Model (Selected)

```
User(id, email, name, orgId, role)
Org(id, name, plan, billingId)
Installation(id, orgId, githubInstallationId, permissions)
Project(id, orgId, repoFullName, defaultBranch, framework, siteBaseUrl)
Run(id, projectId, trigger, headSha, baseSha, prNumber?, createdAt, status)
Finding(id, runId, rule, severity, selector, filePath, codeSpan, snippet, wcagRef, suggestionId)
Suggestion(id, findingId, patch, testStubPaths[], rationale)
PR(id, projectId, provider, repo, prNumber, branch, status, lighthouseBeforeRef, lighthouseAfterRef)
Policy(id, projectId, minContrast, allowedRoles, failOnSeverity, routesConfig)
MediaAsset(id, projectId, type, pathOrUrl, captionsUrl)
APIKey(id, projectId, scopes, createdAt)
```

---

## 6 Sequence: PR Event → Fix PR

1. **GitHub**: `pull_request` → Webhook received.
2. **Gateway**: Create `Run`; shallow clone; enqueue `run:{id}`.
3. **Workers**: Static rules + dynamic audits + CV; for each finding call LLM+RAG → save `Finding` + `Suggestion`.
4. **Checks**: Post summary + annotations to PR; link artifacts.
5. **User clicks “Create Fix PR”** (Checks action/PR comment/Web app): `POST /v1/prs` with findingIds.
6. **PR Service**: Create branch → apply patches → build/tests → push → open PR.
7. **Audit Runner**: Execute before/after audits; attach reports to PR.
8. **Dashboard**: Shows PR status; merging closes associated findings.

---

## 7 API/Integration Sketches

**Webhook** `POST /webhooks/github`

* Verifies signature, maps installation → org.

**Checks Output (example)**

* Title: `AccessMate: 6 issues (2 critical)`
* Annotations: file/line or URL/selector scoped
* Actions: `Create Fix PR`, `Open Dashboard`, `Re-run`.

**Command** (optional): `/accessmate fix contrast --route /checkout`

**Public API** (for dashboard/manual control)

* `POST /v1/runs { projectId, trigger, routes? } → { runId }`
* `GET /v1/runs/{id}` → findings, artifacts
* `POST /v1/fixes { findingIds[] }`
* `POST /v1/prs { projectId, findingIds[] } → { prUrl, branch }`

---

## 8 Prompting & Guardrails

* **System prompt**: “You are an accessibility engineer. Produce minimal, standards‑compliant patches; never change semantics unless required; always cite WCAG.”
* **Content filters**: No selector hallucinations; only edit files from the checkout; AST diffs must parse; run `--dry-run` builds; abort on test regressions.
* **Cost controls**: Rule checks first; batch LLM calls; cache common remediations; reuse WCAG retrievals.

---

## 9 Security & Privacy

* GitHub App: installation tokens (short‑lived), repo‑scoped; no PATs.
* Shallow/sparse clones; ephemeral workspaces; artifacts redacted; opt‑out paths (e.g., `/account/*`).
* Secrets: stored in cloud secrets manager; used only in sandboxed Playwright runners.
* Encryption in transit (TLS) + at rest (KMS); immutable audit logs for PR actions.

---

## 10 Observability

* **Metrics**: runs/min, findings/run, fix‑rate, PR merge‑rate, Lighthouse delta, token spend, rerun rate.
* **Tracing**: Webhook → gateway → workers (OpenTelemetry); queue latency histograms.
* **Logging**: Structured logs with PII redaction; GitHub delivery IDs for correlation.

---

## 11 Performance & Scaling

* Stateless gateway; autoscaled workers by queue depth; ephemeral Playwright pods with headless Chrome.
* Artifact storage in S3 via presigned URLs; CDN for report viewing.
* Caching of WCAG chunks/prompts; warm CV models.

---

## 12 Testing Strategy

* **Unit**: rule engine, contrast calc, AST patchers.
* **Integration**: repo samples (Next, Vue, Rails+ERB) to validate multi‑stack fixes.
* **E2E**: From webhook → Checks → Fix PR on demo repos.
* **Contract**: Webhook/event schemas (zod/pydantic); GitHub App mocks.
* **A11y self‑test**: Dashboard meets WCAG AA.

---

## 13 Deployment & Environments

* **Envs**: dev/staging/prod with isolated queues, runners, and DBs.
* **CI/CD**: GitHub Actions; IaC (Terraform); smoke tests post‑deploy; signed containers.
* **Runners**: Kubernetes Jobs for audits; concurrency limits per installation.

---

## 14 Roadmap (MVP → Plus)

**MVP**

* GitHub App with PR Checks; static + basic dynamic audits on configured routes; LLM explanations; manual "Create Fix PR"; Lighthouse before/after; basic dashboard.

**Plus**

* Scheduled repo scans; captioning service; color‑blind previews; Jira auto‑filing when auto‑fix not feasible; org policies (fail PR on ≥High severity); monorepo awareness; SARIF export.

---

## 15 Edge Cases & Limitations

* Framework diversity (SSR/CSR/islands): require adapters for common stacks.
* Auth‑gated pages: need user‑provided secrets/fixtures or session captures.
* Shadow DOM/virtualized lists: runtime hooks; component‑library heuristics.
* Canvas/WebGL text: OCR or developer hints.
* Dynamic themes: per‑theme contrast checks.
* CI time limits: shard routes; cache builds.

---

## 16 Real‑World Fit & Rollout

* **Adoption path:** Start read‑only (Checks & advice). Enable guarded auto‑fix in bot branches. Expand to direct PRs after trust is built.
* **Success criteria:** Reduced a11y violations per K LoC; faster PR a11y sign‑offs; measurable Lighthouse accessibility score improvements; merge‑rate of bot PRs.
