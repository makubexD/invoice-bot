# 04 · Implementation plan

Each phase ends with a **demo plus passing tests**. Tasks reference requirement IDs, and nothing is built without a requirement.

## Phase 0: Setup and spikes (de-risk first) · ~1 week
| # | Task | Reqs | Done when |
|---|---|---|---|
| 0.1 | Collect **≥ 30 real invoice photos** (factura and boleta, restaurant and retail, good and bad light) plus expected JSON → `tests/golden/` | R-2.9 | Dataset committed (buyer data anonymized) |
| 0.2 | Spike: QR cascade on the golden set; measure the hit rate per decoder | R-2.1 | Report in `specs/spikes/qr.md` |
| 0.3 | Spike: Tesseract on the no-QR cases; measure field accuracy | R-2.3 | Report |
| 0.4 | **Manual check of SOL "Consulta de comprobantes"**: can the buyer download XML of received invoices? | R-5.2.4 | Yes/no plus screenshots of the menu path |
| 0.5 | Manual check: SUNAT validez API with your credentials on a *work* invoice (a different buyer RUC) | R-4.1 | Works or not |
| 0.6 | List your top 10–20 frequent issuers (RUC, link, required fields, captcha type) → seed `portals.yaml` | R-5.8 | File committed |
| 0.7 | Meta app plus WhatsApp Cloud API test number, add a payment method, webhook through Cloudflare Tunnel | R-1.x | Echo bot receives an image |
| 0.8 | Microsoft: register an Entra app (personal accounts) and try device-code login with **both** personal and work accounts | R-7.1 | Learn whether the work tenant allows consent (Q2) |
| 0.9 | Memory spike: Chromium headless-shell plus a page on a 1 GB VM | R-9.1 | Peak RSS measured |

## Phase 1: Core skeleton and extraction (MVP-1) · ~1–2 weeks
1.1 Project scaffold: uv, ruff, mypy, pytest, pre-commit, Dockerfile, compose, `/health` (R-9.5)
1.2 Settings plus YAML config validation (R-9.6)
1.3 SQLite store, migrations, repos, job queue with lease and backoff, outbox (R-9.4, R-1.8)
1.4 WhatsApp channel: webhook verify and signature check, media download, send text, buttons and lists (R-1.1–1.4)
1.5 Extraction: image loader → QR cascade → QR parser → validators (R-2.1, 2.2, 2.4, 2.6)
1.6 OCR fallback plus `parsing_rules.yaml` (R-2.3)
1.7 Padrón import script plus lookup (R-2.5)
1.8 State machine `RECEIVED → … → CONFIRMING`, summary message, flow buttons, edit, duplicates (R-3.1–3.4)
**Demo:** send a photo and get back the parsed summary with A/B buttons. Golden-set accuracy report.

## Phase 2: Flow A, end to end (MVP-2) · ~1–2 weeks
2.1 Google OAuth bootstrap script plus refresh (R-6.4)
2.2 Drive sink (folders, upload, links) (R-6.1)
2.3 Sheets sink (column mapping, upsert) (R-6.2, 6.3)
2.4 SUNAT validez client plus token cache (R-4.x)
2.5 Fetcher chain framework plus `manual` plus `qr_url` (R-5.1, 5.2, 5.5, 5.6)
2.6 `email_inbox` (Gmail) fetcher with a retry schedule
2.7 Completion message (R-8.x), commands `/ayuda /estado /pendientes /cancelar /reprocesar` (R-10)
**Demo:** photo → confirm A → row in Sheets plus files in Drive (docs from email, or MANUAL_REQUIRED).

## Phase 3: Browser fetchers · ~1–2 weeks
3.1 BrowserPool (lazy, idle close, resource blocking) (R-9.1)
3.2 `generic_form` portal adapter driven by `portals.yaml` (R-5.2.2, 5.8)
3.3 Captcha: local OCR, then ask the user over WhatsApp (R-5.3, 5.4)
3.4 `sunat_sol` fetcher for flow A (depends on 0.4) (R-5.2.4)
3.5 Per-portal stats plus `portal-test` CLI (R-5.7)
3.6 Custom adapters for the 2–3 most frequent portals that the generic adapter can't handle
**Demo:** an invoice from a seeded portal gets its XML/PDF automatically. Memory stays under budget.

## Phase 4: Flow B · ~1–2 weeks
4.1 MSAL device-code bootstrap plus token cache (R-7.1)
4.2 OneDrive sink (upload, folders)
4.3 Excel template sink: download the template, fill cells with openpyxl, upload with If-Match (R-7.2)
4.4 Batches: model, `/lote`, `/enviar`, per-invoice, on-command and scheduled strategies, preview and confirmation (R-7.3, 7.6)
4.5 Mailer (Graph and Gmail), templates, zip or upload session for large attachments (R-7.4, 7.5, 7.7)
4.6 Outlook `email_inbox` variant
**Demo:** 3 work invoices → `/enviar` → one email with 3 PDFs, 3 XMLs and filled Excel files.

## Phase 5: Hardening and operations · ~1 week
5.1 `memprofile.py` and latency report wired into CI (R-9.1, 9.2)
5.2 Backups: nightly copy of SQLite and config to Drive or OneDrive
5.3 Housekeeping cron (temp purge, stale jobs, padrón refresh)
5.4 Runbook: re-authorizing tokens, adding a portal, rotating secrets
5.5 Optional: Telegram channel (R-1.7), n8n hook sink, paid API and paid captcha adapters

## Definition of done (every task)
- Requirement IDs referenced in the PR, tests added, `ruff`, `mypy` and `pytest` green.
- No secrets in code or logs. Memory impact noted if the task touches extraction or the browser.
