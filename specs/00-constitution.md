# 00 · Constitution

These principles govern every spec, design decision, and pull request. A change that breaks one of them needs an explicit amendment here first.

## P1 · Zero or near-zero cost
- Use free tiers and open-source software only. A paid service may appear only as an **optional, disabled-by-default** adapter.
- Target running cost: US$0/month, plus a VPS if one isn't already available.

## P2 · Built for a 1 GB RAM host
- **Steady-state RSS ≤ 350 MB. Peak ≤ 800 MB.** Use 1–2 GB of swap as a safety net, not as the plan.
- Run one process with an async core. **At most one heavy job at a time** (OCR or browser), enforced by a semaphore.
- Start headless Chromium **on demand** and close it after an idle timeout. Never keep it resident.
- No Redis, Postgres, Celery or Kafka. **SQLite (WAL)** is the database, the job queue, and the cache.
- Stream media to disk, downscale images before processing, and delete temporary files right away.
- Prefer thin REST calls (`httpx`) over heavy SDKs when the SDK pulls in large dependency trees.

## P3 · Deterministic extraction (no AI/LLM)
- Order of precedence: **QR decode → official XML → OCR + regex**.
- The only ML allowed is local OCR (Tesseract LSTM). No cloud AI. No LLMs.
- Every extracted field carries its `source` (qr|xml|ocr|user) and a `confidence`.

## P4 · Human in the loop over silent failure
- The user always sees the parsed data and **confirms or edits it** before anything is delivered.
- When automation can't get a document (captcha, portal down, no access), the bot tries the next fallback. When all fallbacks are exhausted, it sets **`MANUAL_REQUIRED`** and tells the user exactly what is missing.

## P5 · Pluggable everything
- Channels (WhatsApp, Telegram), extractors, document fetchers, and delivery sinks (Google, Microsoft, email, n8n) are **ports with adapters**, chosen by configuration.
- Flows are **declarative config** (YAML), not code. Adding a third flow should need no core change.

## P6 · Security and privacy
- Accept messages only from allowlisted phone numbers.
- Verify the Meta webhook signature (`X-Hub-Signature-256`) on every request.
- Keep secrets (OAuth refresh tokens, SUNAT client secret, SOL credentials) out of git. Encrypt them at rest, never log them.
- Use a **secondary SOL user** with query-only permissions.
- Scrape only the owner's own invoices, rate-limited, and respect the target sites' terms.

## P7 · Idempotent and resumable
- Each invoice has a natural key `(ruc_emisor, tipo, serie, numero)`. Duplicates are detected, never processed twice.
- Every step is a persisted state transition, so a crash or restart resumes where it stopped.

## P8 · Spec-first and test-first
- Every requirement has an ID (`R-x.y`), and every task and test references one.
- A golden dataset of real invoice photos with expected JSON is the acceptance test for extraction.

## P9 · Fail loud, never silent
- Every stop, warning or reroute has a **business rule ID** (`BR-*`) and a clear Spanish message telling the user what happened and what comes next (spec 06).
- When the cause is a missing capability in the repo, the system raises a **gap**:
  - a GitHub issue plus a draft PR;
  - CI goes red until the gap is fixed;
  - the admin gets a WhatsApp alert.
- When the fix is deployed, the affected invoices resume automatically.
- The UX stays smooth: no menus when the code can decide (the flow comes from the buyer RUC). Questions are asked only when data is missing or ambiguous.
