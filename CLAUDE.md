# CLAUDE.md — Claims Guard (repo: 079-nestle-vm)

> Working name: **Claims Guard**. Built by Anergy Academy for the Nestlé Vietnam AI Reinvention challenge
> (GenAI Fund / BeyondPilot), challenge area: *"Check every ad, label and post against the law and approved claims before it goes out."*
> Engine lineage: reuses the audit-engine pattern from ShoppingAI (shoppingai.anergyacademy.com):
> **split content → check each item against rules → report + suggested rewrite**.

---

## 0. How to work in this repo (read first)

- Work **one milestone at a time** (section 9). After finishing a milestone: run tests, commit, write a 5-line summary in `docs/PROGRESS.md`, then **stop and wait** for the human.
- Credit budget is small (~USD 90). Do not refactor, do not add features outside the current milestone, do not install heavy frameworks.
- Never invent laws, article numbers, or approved claims. If data is missing, create a clearly marked placeholder (`TODO-VERIFY`) and tell the human.
- All demo brand data is **fictional** unless the human adds real material. Do not scrape Nestlé sites. Do not use Nestlé logos.

---

## 1. Problem

Food/beverage marketing teams publish ads, packaging labels and social posts. Every "claim" (e.g. "rich in calcium", "boosts immunity", "100% natural", "doctor recommended") must:

1. comply with Vietnamese law (advertising, food labelling, children's nutrition products), and
2. match the company's **approved claims library** (exact wording, numbers, conditions).

Today this is manual review by legal/regulatory staff: slow and expensive. Claims Guard does the first-pass check and hands a cited, explainable report to a human approver.

**AI never gives the final decision. A human approves.**

---

## 2. MVP scope (what the demo must do)

Input (any one):
- an image (ad, label, social post screenshot) — PNG/JPG
- pasted text (caption, script)
- language: Vietnamese or English

Output:
- list of every claim found, each with a verdict:
  - `APPROVED` — matches approved claims library
  - `VARIANT` — close to an approved claim but wording/number differs
  - `NOT_IN_LIBRARY` — not an approved claim
  - `REG_RISK` — conflicts with a regulatory rule (must cite rule)
  - `NEEDS_HUMAN` — cannot decide with evidence available
- for each flag: the rule/claim it was checked against, with a **verbatim excerpt** from the stored source text
- a suggested rewrite built **only** from approved-claim wording
- overall status: `PASS` / `REVIEW` / `BLOCK`
- human buttons: Approve / Reject / Comment → written to audit log
- report export: JSON + printable HTML
- UI language toggle: EN / VI (labels only; Vietnamese strings may be machine-drafted, mark them `TODO-VERIFY-VI`)

Out of scope for MVP: login/SSO, multi-tenant, video, workflow integrations, model fine-tuning.

---

## 3. Pipeline

```
input (image|text)
  └─ 1. EXTRACT     vision/OCR → text blocks (keep position if available)
  └─ 2. SEGMENT     split into individual claim candidates
  └─ 3. CLASSIFY    claim type: nutrient | health_function | comparative |
                    natural_origin | endorsement | child_targeted | other
  └─ 4. MATCH       vs approved claims library (normalise numbers/units,
                    semantic + fuzzy match) → APPROVED | VARIANT | NOT_IN_LIBRARY
  └─ 5. RULE CHECK  retrieve relevant rules from regulatory corpus,
                    LLM judges, MUST return rule_id + verbatim excerpt
  └─ 6. VALIDATE    citation guard (section 4) — drop/downgrade bad citations
  └─ 7. REWRITE     suggest compliant wording using approved claims only
  └─ 8. REPORT      JSON + HTML, overall status, audit log entry
```

Each step is a separate module with a pure function and its own unit test.

---

## 4. Anti-hallucination rules (non-negotiable)

1. Every `REG_RISK` must carry `rule_id` and `excerpt`. The validator checks that `excerpt` is an **exact substring** of the stored rule text for that `rule_id`. If not → downgrade to `NEEDS_HUMAN`.
2. Every `APPROVED`/`VARIANT` must carry a `claim_id` that exists in the library.
3. Rewrites may only reuse wording from library entries; validator checks each rewrite references ≥1 `claim_id`.
4. LLM calls use structured JSON output with a schema; invalid JSON → retry once → `NEEDS_HUMAN`.
5. Temperature 0 for judging steps.
6. If retrieval returns no relevant rule, say so. Never "fill in" a law.

---

## 5. Data (all files live in the repo or on our own server)

```
data/
  rules/vn/               regulatory corpus, one YAML per source document
    _sources.yaml         list of sources + URL + status (TODO-VERIFY until checked)
    advertising_law.yaml
    food_labelling.yaml
    child_nutrition_ads.yaml
  claims/
    demo_approved_claims.yaml   FICTIONAL brand library for demo
  fixtures/
    cases/                test inputs (text + images) with expected verdicts
```

Rule file format:
```yaml
source_id: VN-ADV-LAW
title: "Law on Advertising (TODO-VERIFY exact title/number)"
url: "TODO-VERIFY"
status: TODO-VERIFY        # becomes VERIFIED only after human/VN partner check
rules:
  - rule_id: VN-ADV-LAW-001
    article: "TODO-VERIFY"
    text_vi: "..."         # verbatim source text
    text_en: "..."         # translation, marked as translation
    tags: [health_function, child_targeted]
```

Candidate sources to seed (human must verify each one before status = VERIFIED):
- Vietnam Law on Advertising (2012)
- Food safety decree covering product declaration and advertising of food / health supplements (Decree 15/2018/ND-CP)
- Goods labelling decree (Decree 43/2017/ND-CP, amended by Decree 111/2021/ND-CP)
- Rules on nutrition products for young children / breast-milk substitutes (Decree 100/2014/ND-CP)

Approved claims format:
```yaml
brand: "DemoMalt (fictional)"
claims:
  - claim_id: DM-001
    text_en: "Source of vitamin B to support normal energy release"
    text_vi: "TODO-VERIFY-VI"
    conditions: "Only with ≥15% NRV per serving"
    allowed_channels: [pack, tv, social]
    not_for: [infant_products]
```

Fixtures: build **at least 12 fictional test cases** (6 VI, 6 EN) covering every verdict type, incl. obvious traps: "makes kids smarter", "cures", "doctor recommended", wrong % number, child-targeted health claim.

---

## 6. Tech stack

- Python 3.11, **FastAPI** backend
- Single-page frontend: `web/index.html` (plain HTML + vanilla JS + one CSS file). Visual style close to ShoppingAI (cards, step badges, report panel).
- LLM: provider-agnostic wrapper `app/llm.py`. Default: Anthropic Claude (vision for extraction). Keys from `.env`. Keep a `MOCK_LLM=1` mode that returns fixture answers so tests and demos run with zero credit.
- Retrieval: start simple — tag filter + BM25 (`rank_bm25`) over rule texts. No vector DB in MVP.
- Storage: SQLite (`storage/claims_guard.db`) for audit log + report index; files in `storage/uploads/`, `storage/reports/`.
- Tests: `pytest`.
- Deploy: `Dockerfile` + `docker-compose.yml` (one service + a volume for `storage/`).

Do not add: Supabase, n8n, Redis, Celery, React, auth libraries — not in MVP.

---

## 7. Repo layout

```
079-nestle-vm/
  CLAUDE.md
  README.md
  .env.example
  .gitignore              (storage/, .env, __pycache__, *.db)
  Dockerfile
  docker-compose.yml
  requirements.txt
  app/
    main.py               FastAPI routes
    llm.py                provider wrapper + MOCK mode
    schemas.py            pydantic models (Claim, Verdict, Report)
    pipeline/
      extract.py  segment.py  classify.py  match.py
      rules.py    validate.py rewrite.py   report.py
    store.py              SQLite audit log
  data/                   (section 5)
  web/
    index.html  app.js  style.css  i18n.json
  tests/
  docs/
    PROGRESS.md
    DEMO_SCRIPT.md
  storage/                (gitignored, created at runtime)
```

---

## 8. API

- `POST /api/check` — multipart (image) or JSON `{text, lang}` → Report JSON
- `GET  /api/report/{id}` — Report JSON
- `GET  /api/report/{id}/html` — printable report
- `POST /api/report/{id}/decision` — `{decision: approve|reject, comment, reviewer}`
- `GET  /api/health`

---

## 9. Milestones (stop after each)

**M0 — Skeleton**: repo layout, FastAPI health route, `.env.example`, Docker runs locally, pytest runs. *Done when* `docker compose up` serves `/api/health`.

**M1 — Data**: rule YAML schema + loader, demo claims library, `_sources.yaml`, 12 fixtures with expected verdicts. *Done when* loader tests pass and every rule file has `status`.

**M2 — Text pipeline (MOCK_LLM)**: segment → classify → match → rules → validate → report, text input only, mock LLM. *Done when* all 12 fixtures produce expected verdicts in mock mode.

**M3 — Real LLM + image**: real provider in `llm.py`, vision extraction for images, citation validator active. *Done when* ≥10/12 fixtures pass with the real model and zero unvalidated citations reach the report.

**M4 — UI + audit**: `web/` page (upload, results table with colour verdicts, excerpt popovers, rewrite suggestion, approve/reject), EN/VI toggle, HTML report export. *Done when* a full demo run works in browser.

**M5 — Deploy + demo pack**: deploy to our Singapore server via docker compose behind HTTPS; write `docs/DEMO_SCRIPT.md` (3-minute walkthrough using 3–5 fixtures). *Done when* the public demo URL works and the script runs end-to-end.

---

## 10. Deployment & data residency

- Runs on **our own server** (Tencent Cloud, Singapore) with docker compose. Data stays in the `storage/` volume on that server.
- The only data leaving the server is the LLM API call (content being checked). Document this in README. Keep the `llm.py` wrapper swappable so a client could later point it to a model hosted in their own environment.
- Subdomain suggestion: `claims.anergyacademy.com` (human to set DNS).
- Never commit `.env`, uploads, reports, or the DB.

---

## 11. Positioning line (for README and demo)

"Claims Guard is a first-pass compliance reviewer for food & beverage marketing. It extends Anergy Academy's live content-audit engine (ShoppingAI) from marketplace listings to regulatory claims checking. Every flag is cited; a human always approves."
