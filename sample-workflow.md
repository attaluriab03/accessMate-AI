# AccessMate: Accessibility Fixes (Run #42)
> Source PR: #187  ·  Repo: acme/web  ·  Date: 2025-09-26

This PR applies a batch of **high-confidence accessibility fixes** detected by AccessMate.
Each fix is listed below with WCAG citations, file/line, and review notes.

---

## ✅ Summary
- Issues fixed: **5** (2 Critical, 3 Moderate)
- Routes scanned: `/`, `/pricing`, `/checkout`
- Evidence: before/after screenshots + Lighthouse + axe report (links below)

---

## 📦 Changes in this PR
- Add missing `alt` text for images
- Associate labels with form inputs
- Improve color contrast via design tokens
- Add `aria-label` for icon-only buttons

---

## 🔍 Review Checklist (uncheck to request removal before merge)
- [x] **IMG missing alt** — `src/components/Hero.tsx:34`
  - Fix: `alt="Acme Analytics dashboard preview"`
  - WCAG 1.1.1 Non-text Content
- [x] **Button needs accessible name** — `src/components/IconButton.tsx:18`
  - Fix: `aria-label="Open navigation menu"`
  - WCAG 4.1.2 Name, Role, Value
- [x] **Form control lacks label** — `src/pages/checkout.tsx:77`
  - Fix: `<label htmlFor="email">Email</label>` + `id="email"`
  - WCAG 1.3.1 Info and Relationships
- [x] **Insufficient color contrast** — `src/styles/tokens.css:52`
  - Fix: `--color-primary-600` from `#6BA6FF` → `#377DFF` (AA @ 14px)
  - WCAG 1.4.3 Contrast (Minimum)
- [x] **Heading level jump (h2→h4)** — `src/pages/pricing.tsx:41`
  - Fix: `h3` to maintain outline
  - WCAG 1.3.1 Info and Relationships

> To exclude a fix, comment: `@accessmate remove <item-number>` (e.g., `@accessmate remove 3`)

---

## 🧪 Evidence (Artifacts)
- 📸 **Screenshots (before/after)**: [View album](https://artifacts.example/run-42/screenshots)
- 🧠 **axe report**: [JSON](https://artifacts.example/run-42/axe.json)
- 🔦 **Lighthouse (Accessibility)**: [HTML](https://artifacts.example/run-42/lh.html)

**Lighthouse delta (Accessibility):**
| Route      | Before | After | Δ  |
|------------|--------|-------|----|
| `/`        | 83     | 94    | +11|
| `/pricing` | 88     | 96    | +8 |
| `/checkout`| 76     | 90    | +14|

---

## 🗂 Affected Files
- `src/components/Hero.tsx`
- `src/components/IconButton.tsx`
- `src/pages/checkout.tsx`
- `src/pages/pricing.tsx`
- `src/styles/tokens.css`

---

## 🧭 How to Review
1. Skim the **Review Checklist** above.
2. Open “Files changed” to spot the minimal diffs (AST-safe edits).
3. If any fix isn’t desired, comment `@accessmate remove <item-number>` and I’ll update the branch.
4. Approve & merge when satisfied.

---

## 🔁 Commands
- `@accessmate re-run` — Re-run audits for this branch
- `@accessmate remove <item-number>` — Drop a specific fix
- `@accessmate split` — Split this PR into one PR per severity (optional)

---

## 📝 Notes
- Contrast fix uses tokens to avoid visual regressions.
- Form labels keep existing styles; no layout changes.
- No runtime behavior changes expected. CI build passed.
