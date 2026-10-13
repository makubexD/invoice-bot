# 03 · Design

## 1. Architecture overview

```
 WhatsApp Cloud API ──HTTPS──► Cloudflare Tunnel ──► app (single Python process, asyncio)
                                                      │
   ┌──────────────────────────────────────────────────┼─────────────────────────────────────┐
   │ channels/        ← port: Channel  (whatsapp, telegram)                                  │
   │ api/webhook      → verify signature → dedupe msg.id → persist → enqueue → 200 OK        │
   │ queue/           SQLite-backed job queue (lease + retry + backoff)                      │
   │ worker/          1 async worker loop; HEAVY semaphore = 1 (OCR / browser)               │
   │ pipeline/        state machine per invoice (see §4)                                     │
   │   extract/       qr.py → ocr.py → parse.py → validate.py   (+ padron lookup)            │
   │   sunat/         validez API client (token cache)                                       │
   │   fetchers/      chain: qr_url → portal_registry → email_inbox → sunat_sol → paid → manual│
   │   sinks/         google_drive, google_sheets, onedrive, excel_template, mailer, n8n_hook│
   │ flows/           declarative YAML: which fetchers, which sinks, batching, templates     │
   │ scheduler/       tiny cron (asyncio) for scheduled batches + housekeeping               │
   │ store/           SQLite (WAL): messages, invoices, jobs, batches, files, kv cache       │
   └─────────────────────────────────────────────────────────────────────────────────────────┘
         ▲ on demand only: chromium-headless-shell (Playwright), tesseract (subprocess)
```

**Pattern map:**
- **Hexagonal (ports and adapters):** channels, fetchers and sinks.
- **Chain of Responsibility:** fetchers.
- **State machine:** invoice lifecycle.
- **Strategy:** email trigger modes.
- **Registry:** portals and flows.
- **Outbox:** outgoing WhatsApp messages are persisted, then sent, so they're retry-safe.

## 2. Tech stack (chosen for a 1 GB RAM host)
| Concern | Choice | Why |
|---|---|---|
| Language | Python 3.12, `uv` | Best OCR, QR, openpyxl and Playwright ecosystem |
| HTTP server | Starlette + uvicorn (1 worker, `--loop uvloop`) | ~40 MB, async |
| Models / config | pydantic v2 + `pydantic-settings`, YAML | Validation of config and extracted data |
| HTTP client | `httpx` (one shared AsyncClient, HTTP/2 off, pooled) | Thin REST to Meta, Google, Graph and SUNAT; avoids the heavy `google-api-python-client` |
| DB / queue / cache | SQLite WAL via `aiosqlite` | Zero extra processes |
| QR | `zxing-cpp`, `opencv-contrib-python-headless` (wechat_qrcode), `pyzbar` | Cascade |
| OCR | Tesseract 5 + `spa` (apt), invoked as a subprocess | Memory is freed when it exits |
| Images | OpenCV headless; downscale long side to ≤ 2000 px; `cv2.IMREAD_REDUCED_*` when huge | Bounded memory |
| Browser | Playwright + `chromium-headless-shell`, launched per fetch batch, closed after 60 s idle | Never resident |
| Excel | `openpyxl` (keep formatting; `keep_vba` for .xlsm) | Graph Excel API is unsupported on personal OneDrive |
| XML | `lxml` with UBL 2.1 XPaths (ported from Greenter) | Fast, small |
| Auth | Google OAuth refresh token (manual HTTP), `msal` token cache for Graph | |
| Padrón RUC | SQLite table `padron(ruc INTEGER PRIMARY KEY, name, estado, condicion) WITHOUT ROWID`, refreshed weekly by streaming the ZIP (no full load in RAM) | ~O(log n) lookup, ~0 RAM |
| Deploy | Docker Compose: `app` (mem_limit 850m) + `cloudflared`; 2 GB swapfile on host | |
| n8n | **Optional** `sinks/n8n_hook` (POST JSON to an n8n webhook) | See below |

### Note on n8n (open question Q1)
You chose "code + n8n". On 1 GB, n8n (250–450 MB resident) **plus** a Chromium fetch (≈300 MB) plus OCR can exceed the budget.

**Design decision:** the Python core implements every adapter natively, through thin REST calls. n8n is an **optional sink**: any flow can add `n8n_hook` to hand the confirmed invoice JSON and file URLs to an n8n workflow.

Enable it when:
- a. the host has ≥ 2 GB, or
- b. n8n runs on another machine.

You keep visual customization when you want it, and the core never depends on it.

## 3. Data model (SQLite)
```
messages(id TEXT PK /*meta msg id*/, sender, kind, received_at, payload_json)            -- idempotency
invoices(id INTEGER PK, sender, flow_id, state, natural_key TEXT UNIQUE NULL,
         ruc_emisor, tipo, serie, numero, fecha, moneda, base, igv, total,
         ruc_adq, razon_social, fields_json /*value+source+confidence*/,
         sunat_status_json, batch_id NULL, error, created_at, updated_at)
files(id PK, invoice_id, kind /*photo|pdf|xml|xlsx|captcha*/, local_path NULL,
      remote_ref NULL, sha256, size, created_at)
jobs(id PK, invoice_id, step, run_after, attempts, lease_until, last_error)              -- queue
batches(id PK, flow_id, state /*open|sending|sent*/, opened_at, sent_at, email_ref)
fetch_attempts(id PK, invoice_id, fetcher, portal_key, ok, reason, ms, at)               -- R-5.7 stats
outbox(id PK, to, payload_json, state, attempts, created_at)                              -- outbox pattern
kv(key PK, value, expires_at)                                                            -- token cache etc.
padron(ruc INTEGER PK, razon_social, estado, condicion) WITHOUT ROWID
```
Natural key: `"{ruc_emisor}-{tipo}-{serie}-{int(numero)}"`. It is set once confirmed.

## 4. Invoice state machine
```
RECEIVED ─► EXTRACTING ─► CONFIRMING ──(user picks flow / edits)──► VALIDATING ─► FETCHING_DOCS
                │              │  ▲ edit                                  │            │  ├─► WAITING_CAPTCHA ─┐
                │              ▼  │                                       │ estadoCp≠1 │  ◄────────────────────┘
                │         PENDING_USER (timeout)                          ▼            ▼
                ▼                                                  CONFIRM_INVALID   DELIVERING ─► DONE
           EXTRACT_FAILED → asks user for data / better photo                         │   (flow B on_command/scheduled:
                                                                                      │    DONE = added to batch)
                                         FETCHING_DOCS (all fetchers failed) ─► MANUAL_REQUIRED ─(user sends files)─► DELIVERING
   any ─► CANCELLED (/cancelar) ;  transient error ─► retry w/ exp. backoff (max 5) ─► FAILED
```
Each transition is a single SQLite transaction (state update + next job insert + outbox message).

## 5. Extraction pipeline detail
1. **Load:**
   - Decode with `cv2.imdecode`, downscale to ≤ 2000 px on the long side.
   - Keep a grayscale copy and drop the color image as soon as possible.
2. **QR cascade** (stop at the first payload that parses as SUNAT):
   1. zxing-cpp `read_barcodes(gray, formats=QRCode|PDF417)`. Older CPEs used PDF417.
   2. If nothing: crop candidate regions (bottom 40%, and contour-detected squares), apply 2× upscale + unsharp mask, retry zxing.
   3. If nothing: WeChat detector (its models are ~1 MB, loaded lazily).
   4. If nothing: pyzbar on the binarized image, then on rotations.
3. **QR parse:**
   - Split on `|`, strip, and map by position.
   - Validate the RUC check digit, `tipo ∈ {01,03,07,08,12…}`, the serie regex, decimals, and the date (two formats).
   - A payload starting with `http` becomes `document_url`.
4. **OCR fallback** (subprocess `tesseract - - -l spa --psm 4 --oem 1`, 30 s timeout):
   - Preprocess: 4-point perspective warp → deskew → CLAHE → adaptive Gaussian threshold → morphological open.
   - Rule file `config/parsing_rules.yaml`, keyword-anchored regexes:
     - `R\.?U\.?C\.?\s*:?\s*(\d{11})`
     - `(FACTURA|BOLETA)\s+ELECTR[OÓ]NICA`
     - serie-número `([FBE][A-Z0-9]{3}|\d{4})\s*[-–]\s*(\d{1,8})`
     - date formats
     - `(IMPORTE\s+)?TOTAL(\s+A\s+PAGAR)?\s*:?\s*S/?\.?\s*([\d.,]+)`
     - `IGV`, `OP\.?\s*GRAVADA`
   - The first 11-digit RUC with a valid check digit near the header is the issuer; the second one is the buyer.
5. **Enrich:**
   - Padrón lookup gives `razon_social`, `estado` and `condicion`.
   - Optional: a `config/issuers.yaml` alias table gives a friendly name (for example, "Tanta Larcomar").
6. **Score:**
   - Per-field confidence: QR = 1.0, padrón = 1.0, OCR scaled by the Tesseract word confidence and by cross-checks.
   - The overall status is `ok` when every required field has confidence ≥ 0.8, otherwise `needs_review`. Uncertain fields are highlighted in the summary.

## 6. Fetcher chain
Interface: `async def fetch(inv: Invoice, ctx: FetchContext) -> FetchResult(ok, files[], reason)`. The `FetchContext` gives access to:
- the shared browser (lazy),
- `ask_user(image, prompt, timeout)` for captchas,
- the HTTP client,
- secrets.

**Portal registry** (`config/portals.yaml`):
```yaml
- key: nubefact
  match: { ruc: ["20xxxxxxxxx", "..."], url_contains: ["nubefact"] }
  adapter: generic_form                 # or python: fetchers.portals.nubefact:Adapter
  url: "https://.../buscar"
  fields:                               # CSS selector -> expression
    "#ruc": "{ruc_emisor}"
    "#tipo": { select: "{tipo}" }
    "#serie": "{serie}"
    "#numero": "{numero}"
    "#fecha": "{fecha:%d/%m/%Y}"
    "#total": "{total:.2f}"
  captcha: { image: "#captchaImg", input: "#captcha", solver: [tesseract, user] }
  submit: "#btnBuscar"
  downloads: { xml: "a:has-text('XML')", pdf: "a:has-text('PDF')" }
```
- **Seeding the registry:** you supply a list of frequent issuers (RUC → link), which matches what you described ("in parallel we are going to define…"). New issuers default to the next fetcher.
- **`email_inbox`:**
  - Gmail query `has:attachment (filename:xml OR filename:pdf) "{serie}-{numero}" newer_than:30d`.
  - Graph `$search` equivalent on Outlook.
  - Validate the XML against the invoice (R-5.6).
  - Retry schedule: +5 min, +1 h, +6 h, +24 h, because issuers send asynchronously. Meanwhile the invoice stays in `FETCHING_DOCS` with "waiting for email". Final failure → `MANUAL_REQUIRED`.
- **`sunat_sol` (flow A):**
  - Playwright login with the secondary SOL user, navigate to "Consulta de comprobantes de pago", query by issuer, serie and número, download the XML.
  - The selectors live in a file so they're easy to fix when SUNAT changes the UI.
  - **To validate manually in Phase 0:** that this menu actually offers the XML for received invoices.
- **`manual`:** marks the state and composes the message listing the failed fetchers and reasons.

## 7. Sinks and flows (declarative)
`config/example.yaml` shows the full shape. Key ideas:
- A flow lists `fetchers` (ordered), `required_docs`, and `sinks` (ordered). Some sinks are `batch_aware`.
- A sink receives a `DeliveryContext` (invoice, files, previous sink outputs such as Drive links) and returns refs that later sinks or the final message can use.
- Template expressions use a tiny safe formatter (`str.format_map` with a restricted namespace plus formatting filters), not full Jinja, to save memory. Jinja is used only for email bodies, and imported lazily.

## 8. Resource budget and efficiency tactics
- **One uvicorn worker.** CPU work (OpenCV) runs in `asyncio.to_thread`, guarded by `HEAVY = asyncio.Semaphore(1)`. Tesseract and Chromium run as subprocesses, so their memory is returned to the OS when they exit.
- **Browser lifecycle:** `BrowserPool(max=1, idle_close=60s)` with one context per fetch.
  - Launch flags: `--disable-dev-shm-usage --no-zygote --renderer-process-limit=1 --disable-gpu --blink-settings=imagesEnabled=false` (images are re-enabled only for captcha pages).
  - Block fonts, media and analytics with route interception.
- **Lazy imports:** heavy modules (cv2, playwright, openpyxl, lxml, jinja2) are imported inside the functions that need them. Idle RSS stays at about 60 MB.
- **Bounded queues:** at most 20 queued jobs per sender. Back-pressure message: "estoy procesando N facturas, te aviso".
- **SQLite:** `journal_mode=WAL`, `synchronous=NORMAL`, `cache_size=-8000` (8 MB), `mmap_size=64MB`, indices on `invoices(state)`, `jobs(run_after)`, `invoices(natural_key)`.
- **Files:** stream downloads to `/data/tmp/{invoice_id}/`, using SHA-256 for dedup of identical photos. Purge after `DONE`, plus a nightly housekeeping cron.
- **Padrón import:** stream-unzip, parse line by line, `executemany` in chunks of 50k inside one transaction. Peak RAM stays under 50 MB.
- **Measure, don't guess:** a `scripts/memprofile.py` runs the golden set and records peak RSS per step (`psutil`). CI fails if the peak is over 800 MB (R-9.1).

## 9. Security
- Webhook: HMAC SHA-256 check with the app secret. Verify-token handshake on GET.
- Allowlist: `senders:` in config, mapped to a profile (default flow suggestions, permissions such as `can_send_batch`).
- Secrets in `.env` (root-only permissions). Optional `sops` + `age` for encrypted-at-rest files in the repo. The token caches (`msal`, Google refresh token) live in `/data/secrets/` with mode 600.
- SOL: secondary user with only the query options. Credentials are never sent to paid APIs unless that adapter is explicitly enabled.
- The scraper honors per-domain rate limits (≥ 10 s between requests to the same portal), uses a realistic UA, and makes no parallel requests.

## 10. Repository layout (planned)
```
invoice-bot/
  specs/                 # this spec set
  config/                example.yaml, portals.yaml, parsing_rules.yaml, issuers.yaml
  src/invoicebot/
    app.py               # Starlette app, routes, lifespan
    settings.py
    channels/{base,whatsapp,telegram}.py
    store/{db,migrations,repo}.py
    queue/{jobs,worker,scheduler}.py
    pipeline/{states,machine,messages_es}.py
    extract/{image,qr,ocr,parse,validate,padron,ubl}.py
    sunat/{validez,sol}.py
    fetchers/{base,qr_url,portal_registry,email_inbox,sunat_sol,paid_api,manual}.py
    fetchers/portals/{generic_form,...}.py
    sinks/{base,google_drive,google_sheets,onedrive,excel_template,mailer_gmail,mailer_graph,n8n_hook}.py
    auth/{google_oauth,msal_cache}.py
    browser/pool.py
  scripts/{auth_google.py,auth_microsoft.py,import_padron.py,memprofile.py}
  tests/{unit,integration,golden/{images,expected}}/
  deploy/{Dockerfile,docker-compose.yml,cloudflared.yml}
```

## 11. Testing strategy
- **Unit tests:**
  - QR payload parser (variants), RUC check digit, regex rules (table-driven), UBL parser, state machine transitions, cell-mapping formatter, batch strategies.
- **Golden set:** ≥ 30 real photos (anonymized buyer data) with `expected.json`, run in CI. It reports per-field accuracy against R-2.9.
- **Contract tests:**
  - Recorded fixtures (VCR-style) for Meta, Google, Graph and SUNAT.
  - Fake channel for end-to-end conversation tests.
- **Portal adapters:** a smoke test command `invoicebot portal-test <key> <sample-invoice>` that you run by hand, because live portals stay out of CI.
- **Resource test:** `memprofile.py` (R-9.1) and a latency report (R-9.2).
