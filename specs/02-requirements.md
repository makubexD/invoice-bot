# 02 · Requirements

Acceptance criteria use **EARS** syntax: *WHEN* (trigger), *IF* (unwanted condition), *WHILE* (state), and *THE SYSTEM SHALL* (response).
Priority: **M** = MVP, **S** = should (v1), **C** = could (later).

## Actors
- **Owner:** you. Configures everything and has a RUC (20…) with Clave SOL.
- **Sender:** an allowlisted WhatsApp number (≤ ~10 people, < 200 invoices per month in total).
- **Issuer:** the store or restaurant that issued the CPE.

## Flows (initial)
| Flow id | Buyer RUC on the invoice | Delivery | SUNAT fallback available |
|---|---|---|---|
| `personal_google` (A) | Owner's company RUC | Google Drive + Google Sheets | **Yes** (owner's SOL) |
| `work_microsoft` (B) | Employer's RUC | OneDrive + Excel template + email | **No** (status validation only) |

---

## R-1 · Intake (channel)
- **R-1.1 (M)** WHEN an allowlisted sender sends an image (photo or image document), THE SYSTEM SHALL acknowledge with a single message within 5 s and enqueue a job.
- **R-1.2 (M)** IF the sender isn't allowlisted, THE SYSTEM SHALL ignore the message and log it, without replying.
- **R-1.3 (M)** THE SYSTEM SHALL verify `X-Hub-Signature-256` on every webhook call and reject invalid ones with 401.
- **R-1.4 (M)** THE SYSTEM SHALL download the media within 60 s of the webhook, because Meta's media URL expires after 5 minutes.
- **R-1.5 (S)** WHEN a sender sends a PDF or XML document while an invoice is in `MANUAL_REQUIRED` or `CONFIRMING`, THE SYSTEM SHALL attach it to that invoice. The invoice is the one the message quotes, or the most recent open one.
- **R-1.6 (S)** WHEN a sender sends several photos in a row, THE SYSTEM SHALL process them as independent jobs, serialized (FIFO).
- **R-1.7 (C)** THE SYSTEM SHALL support Telegram as an alternative channel through the same port.
- **R-1.8 (M)** Webhook processing SHALL be idempotent per Meta `message.id`, because Meta retries deliveries.

## R-2 · Extraction (no AI)
- **R-2.1 (M)** THE SYSTEM SHALL try QR decoding first, with a cascade of zxing-cpp → WeChat QR → pyzbar on preprocessed variants (original, grayscale, upscaled, sharpened, rotated 90°/180°/270°), stopping at the first valid SUNAT payload.
- **R-2.2 (M)** WHEN a SUNAT QR is decoded, THE SYSTEM SHALL extract: `ruc_emisor`, `tipo` (01/03/07/08), `serie`, `numero`, `igv`, `total`, `fecha_emision`, `tipo_doc_adq`, `num_doc_adq`.
- **R-2.3 (M)** IF no valid QR is found, THE SYSTEM SHALL run OCR (OpenCV preprocessing + Tesseract `spa`) and a regex/keyword parser that extracts the same fields plus `razon_social` (header text).
- **R-2.4 (M)** THE SYSTEM SHALL validate:
  - the RUC check digit (mod 11);
  - the serie pattern (`[FBE][A-Z0-9]{3}` or 4 digits);
  - the date (not in the future, not older than the configured limit);
  - `total ≈ base + IGV` (with a tolerance of S/ 0.05) when both are present.
- **R-2.5 (M)** THE SYSTEM SHALL resolve `razon_social` and `estado/condición` of the issuer from the local padrón RUC. OCR text is the fallback.
- **R-2.6 (M)** Each field SHALL carry `source ∈ {qr, xml, ocr, padron, user}` and `confidence ∈ [0, 1]`.
- **R-2.7 (S)** WHEN the official XML is later obtained, THE SYSTEM SHALL parse UBL 2.1 and treat the XML values as authoritative. Any discrepancy from the photo data is reported to the user.
- **R-2.8 (S)** IF the QR contains a URL instead of data, THE SYSTEM SHALL store it as `document_url` for the fetcher.
- **R-2.9 (M)** Extraction accuracy target, measured on the golden dataset: ≥ 95% of invoices with a readable QR get every QR field exactly right; with OCR only, ≥ 80% of fields are right.

## R-3 · Confirmation and flow selection
- **R-3.1 (M)** After extraction, THE SYSTEM SHALL send **one** summary message containing:
  - the extracted fields;
  - a flow-choice control (reply buttons, or a list message if there are more than 3 flows);
  - an "Edit" option.
- **R-3.2 (M)** THE SYSTEM SHALL **suggest** a default flow from rules (for example, `num_doc_adq` equals the owner RUC → A, equals the work RUC → B). The user can still override it.
- **R-3.3 (S)** WHEN the user picks "Edit", THE SYSTEM SHALL accept `field=value` lines (for example, `total=45.50`), re-validate, and show the summary again.
- **R-3.4 (M)** IF the natural key `(ruc_emisor, tipo, serie, numero)` already exists in a non-failed state, THE SYSTEM SHALL tell the user it is a duplicate and stop, unless the user sends `/reprocesar <id>`.
- **R-3.5 (S)** IF the user doesn't answer within the configured timeout (default 24 h), THE SYSTEM SHALL park the invoice as `PENDING_USER`. It is resumable with `/pendientes`.

## R-4 · Validation with SUNAT
- **R-4.1 (S)** WHEN SUNAT API credentials are configured, THE SYSTEM SHALL call `validarcomprobante` for every confirmed invoice, regardless of flow, and store `estadoCp`, `estadoRuc` and `condDomiRuc`.
- **R-4.2 (S)** IF `estadoCp ≠ 1` (accepted), THE SYSTEM SHALL warn the user and ask whether to continue.
- **R-4.3 (M)** THE SYSTEM SHALL cache the OAuth token until 60 s before it expires.

## R-5 · Document retrieval (PDF/XML)
- **R-5.1 (M)** THE SYSTEM SHALL try fetchers in the **order configured per flow**, stopping at the first one that returns a valid XML. A PDF is optional but desired.
- **R-5.2 (M)** Fetchers available:
  1. **`qr_url`:** follow the URL printed in the QR or on the receipt.
  2. **`portal_registry`:** look up the issuer RUC → portal definition (generic YAML form adapter or custom adapter).
  3. **`email_inbox`:** search the configured mailbox (Gmail or Outlook) for `serie-numero` / issuer RUC and download the XML/PDF attachments.
  4. **`sunat_sol`:** *(flow A only)* log into SOL with the owner's secondary user, query received CPE, and download the XML.
  5. **`paid_api`:** *(optional, off by default)* Factiliza, json.pe or similar.
  6. **`manual`:** always the last step.
- **R-5.3 (M)** WHEN a portal shows an image captcha, THE SYSTEM SHALL first try local OCR (≤ 2 attempts). It SHALL then send the captcha image to the user and wait up to 5 min for a text reply.
- **R-5.4 (S)** WHEN a portal shows reCAPTCHA/hCaptcha, THE SYSTEM SHALL skip to the next fetcher, unless a paid solver is enabled.
- **R-5.5 (M)** IF every fetcher fails, THE SYSTEM SHALL set `MANUAL_REQUIRED`, tell the user which fetchers failed and why (one line each), and accept the files later (R-1.5).
- **R-5.6 (M)** A fetched XML SHALL be checked to match the invoice: its RUC, serie, número and total equal the confirmed data.
- **R-5.7 (S)** THE SYSTEM SHALL record per-portal success and failure stats, so you can see which registry entries are broken.
- **R-5.8 (S)** Portal registry entries SHALL be editable without redeploying (a YAML file or a `/portal` command), keyed by issuer RUC, with an optional match by name.
- **R-5.9 (C)** IF the flow allows it, THE SYSTEM MAY continue delivery without the PDF/XML, marking the row "documents pending", and attach them later when they arrive.

## R-6 · Flow A: Google Drive + Sheets
- **R-6.1 (M)** THE SYSTEM SHALL upload the photo, PDF and XML to the Drive folder `{root}/{YYYY}/{MM}/` with names `{fecha}_{ruc}_{serie}-{numero}.{ext}`. The path template is configurable.
- **R-6.2 (M)** THE SYSTEM SHALL append one row to the configured Sheet and tab. The column mapping is configurable and includes Drive links.
- **R-6.3 (M)** IF the row for the natural key already exists, THE SYSTEM SHALL update it, not duplicate it.
- **R-6.4 (S)** THE SYSTEM SHALL use OAuth with a refresh token, with the consent screen in production mode.

## R-7 · Flow B: OneDrive + Excel template + email
- **R-7.1 (M)** THE SYSTEM SHALL store the photo, PDF and XML in the configured OneDrive folder (personal Microsoft account through Graph). The path template is configurable.
- **R-7.2 (M)** THE SYSTEM SHALL fill the Excel template by writing **mapped cells** (`cell → expression`, for example `C5: "{issuer.name}"`). It works on a copy of the template downloaded from OneDrive, edited with openpyxl, and uploaded with `If-Match`.
- **R-7.3 (M)** The email trigger SHALL be configurable per flow:
  1. **`per_invoice`:** one email per invoice, with its PDF, XML and filled Excel.
  2. **`on_command`:** invoices accumulate in an open "batch"; `/enviar` (or a button) sends **one** email with all PDFs, XMLs and Excel files.
  3. **`scheduled`:** a cron expression (for example, every Friday 18:00 Lima) sends the open batch automatically.
  4. Modes can be combined (for example, scheduled plus manual early send).
- **R-7.4 (M)** Email recipients (to, cc), subject and body SHALL be templates in config (Jinja-style placeholders: totals, count, date range).
- **R-7.5 (S)** IF the attachments exceed the inline limit (3 MB on Graph), THE SYSTEM SHALL use an upload session, or zip the files.
- **R-7.6 (S)** Before sending, THE SYSTEM SHALL send the user a preview (recipients, count, total) with Send/Cancel buttons, configurable `confirm_before_send: true`.
- **R-7.7 (M)** THE SYSTEM SHALL send email via Graph `sendMail` (Outlook) **or** the Gmail API, chosen by config.
- **R-7.8 (C)** Optional mirror to a folder shared with the work account (see Q2).

## R-8 · Completion message
- **R-8.1 (M)** WHEN a flow completes, THE SYSTEM SHALL send one WhatsApp message containing:
  - the status (✅ / ⚠️ docs pending / ❌);
  - the key fields;
  - links (Sheet row and Drive folder, or OneDrive folder) and the batch status for flow B.
- **R-8.2 (M)** IF any step fails irrecoverably, THE SYSTEM SHALL say which step failed and what the user can do.

## R-9 · Non-functional
- **R-9.1 (M)** Memory: steady state ≤ 350 MB RSS, peak ≤ 800 MB on a 1 GB host, verified by a load test of 10 queued invoices.
- **R-9.2 (M)** Latency: acknowledgement in < 5 s. Summary (QR path) in < 20 s, (OCR path) in < 60 s, on 1 vCPU.
- **R-9.3 (M)** WhatsApp messages per invoice: ≤ 4 in the happy path, to stay within 1,000 free service messages per month.
- **R-9.4 (M)** Durability: a restart in any state resumes or reports the job. There are no lost invoices.
- **R-9.5 (M)** Observability: structured JSON logs, a `/health` endpoint, and per-job step timings. Secrets are never logged.
- **R-9.6 (M)** Configuration: one YAML file plus `.env` for secrets. Changing flows, portals or templates needs a restart at most (hot reload: C).
- **R-9.7 (S)** Data retention: local temporary files are deleted after delivery. The SQLite record is kept. Originals live in Drive or OneDrive.
- **R-9.8 (M)** Language: user-facing messages in Spanish (es-PE), code and logs in English.

## R-10 · Commands (chat)
| Command | Purpose | Priority |
|---|---|---|
| `/ayuda` | List the commands | M |
| `/estado <id>` | Show an invoice's status | S |
| `/pendientes` | List invoices waiting for user input or in MANUAL_REQUIRED | S |
| `/lote` | Show the open flow B batch (count, total) | M |
| `/enviar` | Send the open batch now | M |
| `/reprocesar <id>` | Re-run fetch and delivery | S |
| `/cancelar <id>` | Cancel an invoice | S |
| `/portal <ruc> <url>` | Register a portal URL for an issuer (generic adapter stub) | C |
