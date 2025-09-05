# A11y Copilot — System Architecture (v0.1)

## 1. Overview
**A11y Copilot** is a browser extension + web platform that scans webpages for accessibility issues (WCAG/ADA), explains impact in plain English, and optionally auto-generates remediation diffs and GitHub PRs (with tests and a before/after Lighthouse report). The system combines deterministic checks (DOM/CSS rules), computer vision (contrast, OCR), and LLM/RAG pipelines.

### Goals
- Near‑real‑time a11y feedback during development and QA.
- High precision on common, fixable issues; explicit citations to WCAG.
- One‑click remediation via auto‑generated PRs + tests + CI audit.
- Designed for multi-repo, multi-project orgs with role-based access.

### Non‑Goals
- Replacing expert audits for complex flows, legal interpretations, or assistive tech usability studies.
- Fixing deep application logic or architecture issues.

---

## 2. Core User Roles & Flows
**Roles:** Developer, QA/Designer, Admin (Org/Project).

**Primary flows:**
1. **Scan in Browser** → Extension analyzes page → Shows issues with WCAG cites → User chooses *Fix*, *Defer*, or *Ignore*.
2. **Auto‑Fix** → Backend generates patch & tests → Opens PR → Kicks CI (Lighthouse) → Posts result back to PR & dashboard.
3. **Batch Scan** (web app) → Crawl/sitemap import → Scheduled scans → Trend reports & CSV/JSON exports.
4. **Captioning** → Whisper job for videos → Attach .vtt and update markup.

---

## 3. High‑Level Architecture
**Client Tier**
- **Browser Extension (MV3)**: DOM snapshot, CSS capture, network metadata; local rules; invokes backend for advanced analysis. Auth via OAuth device/code flow + short‑lived tokens.
- **Web App (Next.js)**: Dashboard, findings, PRs, projects, schedules, settings; same auth provider(s).

**Service Tier**
- **API Gateway** (Node/Express or FastAPI): REST/GraphQL; rate limiting; authZ.
- **Analysis Orchestrator**: Queues work; composes rule-based checks + CV + LLM; aggregates findings; generates suggested fixes.
- **CV Service**: Contrast ratio, text color extraction, OCR as fallback, color‑blindness simulation previews.
- **Captioning Service**: Whisper pipeline for media; stores .vtt; updates diffs.
- **RAG Service**: WCAG corpus ingestion; embeddings (pgvector/Pinecone); cite spans & remediation guidance.
- **PR/SCM Service**: GitHub/GitLab integration; branch/PR creation; commit patch & tests; PR comments with CI evidence.
- **CI Audit Runner**: Headless audit (Lighthouse/axe-core) before & after; uploads artifacts.

**Data Tier**
- **PostgreSQL (+ Prisma/SQLModel)**: Users, orgs, projects, scans, issues, decisions, PRs, auth, billing.
- **Object Store (S3/GCS)**: DOM snapshots, screenshots, Lighthouse reports, Whisper assets.
- **Vector Store (pgvector/Pinecone)**: WCAG embeddings + internal knowledge.
- **Redis**: Job queues (BullMQ/RQ), short‑lived caches, rate-limits.

---

## 4. Detailed Components
### 4.1 Browser Extension (MV3)
- **Capture**: DOM HTML, computed styles for visible nodes, CSS variables, viewport screenshots.
- **Local Rules**: Fast checks (missing alt, empty labels, heading jumps) to reduce backend calls.
- **Submission**: POST /scan with payload (minified DOM, CSS map, screenshot hash) → gets `scanId`.
- **UI**: Inline issue pins; panel with explanations, WCAG citations, suggested fixes & diffs.

### 4.2 Web App (Next.js + Tailwind)
- Projects, repo links, teams; Findings list; PR status; Scheduled batch scans.
- Triage: *Fix*, *Accept as Risk*, *Ignore with rationale*.
- Reports: Trend charts, coverage, top rule failures.

### 4.3 API Gateway
- Endpoints (examples):
  - `POST /v1/scans` (create scan)
  - `GET /v1/scans/{id}` (status & findings)
  - `POST /v1/fixes` (generate patch for finding[s])
  - `POST /v1/prs` (open PR for fix-set)
  - `POST /v1/captions` (enqueue media captioning)
  - `GET /v1/projects/{id}/reports` (aggregated metrics)
- Cross‑cutting: JWT validation, org/project RBAC, audit logging, usage metering.

### 4.4 Analysis Orchestrator (Workers)
- **Pipeline:**
  1) Parse DOM → build semantic tree (landmarks, roles).
  2) Rule engine (axe-core equivalents + custom rules).
  3) CV hooks (contrast/luminance on screenshots and computed styles).
  4) LLM prompt for explanation + remediation (diffs, aria labels, alt text).
  5) RAG cite: return WCAG guideline with paragraph refs.
- **Outputs:** Findings with severity, impact, repro steps, suggested patch, test stub.

### 4.5 CV Service
- Contrast ratio calc (foreground vs background); color extraction when styles cascade.
- OCR fallback for text-in-images warnings; recommends semantic HTML.
- Color‑blindness simulation thumbnails for preview.

### 4.6 Captioning Service (Whisper)
- Ingest media URL or file → transcode → Whisper inference → VTT/SRT → object store.
- Optional language ID + translation, profanity filters.

### 4.7 RAG Service (WCAG Corpus)
- Ingestion pipeline for WCAG 2.2+ guidelines, techniques, ARIA authoring practices.
- Chunking with hierarchical headings; embeddings; store citations (docId, hRef, line spans).
- Query by rule type; return top‑k with exact cites.

### 4.8 PR/SCM Service
- GitHub App install per org; stores installation id only (no PATs).
- Branch naming: `a11y-fix/{project}/{rule}/{issueId}`.
- Commit includes: code diff, test file(s), README change log when necessary.
- PR body: before/after screenshots, Lighthouse deltas, WCAG citations; checkbox list per finding.

### 4.9 CI Audit Runner
- Uses Playwright to load page state; runs Lighthouse/axe-core; exports JSON + HTML reports; stores artifacts; posts summary to PR.

---

## 5. Data Model (Selected)
```
User(id, email, name, orgId, role)
Org(id, name, plan, billingId)
Project(id, orgId, name, repoUrl, defaultBranch, siteUrl)
Scan(id, projectId, source, url, createdAt, status)
Finding(id, scanId, rule, severity, selector, snippet, wcagRef, suggestionId)
Suggestion(id, findingId, patch, testStubPath, rationale)
PR(id, projectId, provider, repo, prNumber, branch, status, lighthouseBefore, lighthouseAfter)
MediaAsset(id, projectId, type, url, captionsUrl)
APIKey(id, projectId, scopes, createdAt)
```

---

## 6. Sequence: Scan → PR
1. **Extension**: POST /scans (DOM snapshot + screenshotRef).
2. **API**: Create `Scan`; enqueue `scan:{id}`.
3. **Worker**: Run rule engine + CV; for each finding call LLM+RAG for explanation & patch; save `Finding`+`Suggestion`.
4. **Extension/Web**: Poll `GET /scans/{id}` → render issues.
5. **User clicks “Fix”**: POST /prs with findingIds.
6. **PR Service**: Create branch → apply patches → add tests → push → open PR.
7. **CI Runner**: Execute before/after audits; attach reports to PR.
8. **Web**: PR status shown in dashboard; merge closes associated findings.

---

## 7. API Sketches
**POST /v1/scans**
- Body: { url, dom, cssMap, screenshotRef, projectId }
- Returns: { scanId }

**GET /v1/scans/{id}**
- Returns: findings [ { id, rule, wcagRef, selector, explanation, patch } ]

**POST /v1/prs**
- Body: { projectId, findingIds[] }
- Returns: { prUrl, branch }

---

## 8. Prompting & Guardrails
- **System prompt**: “You are an accessibility engineer. Produce minimal, standards-compliant patches; never change semantics unless required; always cite WCAG.”
- **Content filters**: Avoid hallucinating selectors; patch only within provided file scopes; require diffs to compile via dry-run.
- **Cost controls**:
  - Run cheap rule checks first; LLM only for complex fixes/explanations.
  - Batch multiple findings into a single LLM call with structured schema.

---

## 9. Security & Privacy
- OAuth/OIDC (GitHub, Google). Extension uses short‑lived access tokens (PKCE/device code).
- GitHub App integration (least privilege; repo‑scoped). No long‑lived PATs.
- Store minimal DOM (hash or redacted PII selectors for regulated pages). Opt‑out for sensitive routes.
- Encryption at rest (KMS) and in transit (TLS); audit logs for all PR actions.

---

## 10. Observability
- **Metrics**: scans/min, findings/scan, fix‑rate, PR merge‑rate, Lighthouse delta, LLM token spend.
- **Tracing**: extension → API → workers (OpenTelemetry); job latency histograms.
- **Logging**: structured logs; redaction of secrets/PII.

---

## 11. Performance & Scaling
- Stateless API nodes behind CDN/WAF; autoscaled workers by queue depth.
- Artifact storage offloaded to S3; presigned URLs.
- Caching of WCAG chunks and frequent prompts; warm contrast models.

---

## 12. Testing Strategy
- **Unit**: rule engine, contrast calc, patch generators.
- **E2E**: Playwright flows from scan → PR on demo repo.
- **Contract**: API schemas with zod/pydantic; GitHub App mocks.
- **A11y self‑test**: Ensure the dashboard itself meets WCAG AA.

---

## 13. Deployment & Environments
- **Envs**: dev/staging/prod with separate queues & databases.
- **CI/CD**: GitHub Actions; infra as code (Terraform); smoke tests post‑deploy.
- **Secrets**: Vault or cloud secrets manager.

---

## 14. Roadmap (MVP → Plus)
**MVP**
- Extension scans with local rules; server analysis for contrast + LLM explanations; manual PR creation.
- GitHub App for PRs; Lighthouse before/after; basic dashboard.

**Plus**
- Batch sitemap scanning; scheduling; captioning service; multi‑project reporting; color‑blind previews; Jira issues auto‑file when fix not possible.

---

## 15. Edge Cases & Limitations
- Shadow DOM & virtualized lists (need runtime hooks).
- Auth‑gated pages (Playwright session export to extension for scans).
- Canvas/WebGL text (OCR or developer hints required).
- Dynamic theming (light/dark) requires per‑theme contrast checks.

