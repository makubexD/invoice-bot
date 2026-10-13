# 01 · Research summary (as of 2026-10)

Three parallel research passes were run. Parts of sunat.gob.pe and several `.pe` API sites could not be fetched from the research sandbox, so points marked *(verify)* must be confirmed by hand before coding against them.

## 1. Does a project already do this?
**No.** No open-source project covers *WhatsApp photo → rule-based SUNAT extraction → Drive/Sheets or OneDrive/Excel/email*. The WhatsApp receipt bots on GitHub have 0–2 stars and all depend on LLM OCR (Gemini/Claude), as do the n8n community templates. **Decision: reuse libraries and references, build the glue.**

| Reuse | What for | Notes |
|---|---|---|
| [zxing-cpp](https://github.com/zxing-cpp/zxing-cpp) (Apache-2.0) | Primary QR decoder | Python bindings, robust to rotation and skew |
| OpenCV `wechat_qrcode` (contrib) | 2nd QR attempt for small or blurry QRs | CNN-based but tiny and local |
| [pyzbar](https://github.com/NaturalHistoryMuseum/pyzbar) | 3rd QR attempt | Scored better than zxing on some real-world photo benchmarks |
| [thegreenter/greenter](https://github.com/thegreenter/greenter) (MIT) | Reference for the QR format (`QrRender.php`) and UBL 2.1 XPaths (`xml-parser`) | PHP, so port the logic, don't run it |
| [thegreenter/consulta-cpe](https://github.com/thegreenter/consulta-cpe) | Reference client for SUNAT's **validarcomprobante** REST API | A single HTTP call; port to Python |
| [receipt-parser-legacy](https://github.com/ReceiptManager/receipt-parser-legacy) | Pattern for YAML-driven regex parsing of OCR text | Rules are German; rewrite them. Check the license before copying |
| SUNAT *padrón reducido RUC* | Offline RUC → business name / status | Free daily ZIP, ~14.5M rows |

## 2. Extraction without AI
- **The SUNAT QR is the key insight.** Every printed CPE has carried a QR since 2019 (RS 340-2017). Its content is pipe-separated:
  `RUC | tipo | serie | número | IGV | total | fecha | tipoDocAdq | numDocAdq | [hash] |`
  (RS 357-2015 Anexo C, RS 113-2018 Anexo A, RS 114-2019 Anexo VI). That gives 8 of the fields we need with no OCR at all. The parser must accept:
  - 9–11 fields
  - zero-padded or plain numbers
  - `YYYY-MM-DD` or `DD/MM/YYYY` dates
  - QRs that contain a URL instead of the data
- **The QR does NOT contain the store name.** That comes from the padrón RUC lookup (exact), with OCR of the header as a fallback.
- **OCR fallback:**
  - OpenCV preprocessing: perspective fix, deskew, CLAHE, adaptive threshold.
  - Tesseract 5 `spa`.
  - PaddleOCR is more accurate but needs 1–2 GB of RAM, which **rules it out on a 1 GB host**.
- **Validation:**
  - RUC check digit: weights `5,4,3,2,7,6,5,4,3,2`, `r = 11 − (Σ mod 11)`, where 10 → 0 and 11 → 1. Verified on RUC 20131312955.
  - IGV of 18% cross-check (10.5% for some restaurants under Ley 31556, *verify*).
  - SUNAT validity API.

## 3. SUNAT official services
| Service | Returns | Auth | Use |
|---|---|---|---|
| **Consulta Integrada de Validez CPE** (REST) `POST api.sunat.gob.pe/v1/contribuyente/contribuyentes/{miRUC}/validarcomprobante` | **Status only** (`estadoCp` 0–4, `estadoRuc`, `condDomiRuc`). **No XML/PDF.** | OAuth client_credentials, with credentials created in SOL ("Credenciales de API SUNAT") | Validate **any** invoice, including flow B work invoices, using *your own* credentials *(verify)* |
| **SOL "Nueva consulta de comprobantes de pago"** (web) | Received CPE list; XML download reported for the **buyer RUC** *(verify manually)* | Clave SOL of the buyer RUC | Flow A fallback via Playwright. ~12 months online |
| **SIRE Compras API** | Purchase-record rows (structured), not XML | OAuth password grant (RUC + SOL) | Reconciliation and back-fill for flow A |
| **Padrón reducido** | RUC, razón social, estado, condición, dirección | None | Store name lookup |

**Important correction to the brief:** the SUNAT *API* confirms an invoice exists. It does **not** return the PDF/XML. For flow A the official XML route is the SOL web portal, automated with Playwright, or a paid API that logs in with your SOL user (Factiliza, json.pe, APISUNAT: *prices unverified*). Flow B (work RUC) has no SUNAT XML route for you, so it relies on issuer portals, email, or manual handling.

## 4. Issuer, PSE and OSE portals
- No generic solution exists. Each issuer or PSE (Nubefact, Efact, Bizlinks, Paperless, Digiflow, Ekomercio, TCI, Carvajal…) has its own form. Typical fields are RUC, tipo, serie-número, fecha and total, sometimes the buyer's document number.
- Captchas range from none to an image captcha, reCAPTCHA v2, or reCAPTCHA v3.
- **Approach:**
  - A **portal registry** keyed by issuer RUC, which you fill over time.
  - A **generic YAML-described form adapter** that covers simple portals with no code.
  - Custom adapters for the rest.
- **Captcha handling, in order:**
  1. Image captcha: try Tesseract.
  2. Send a screenshot of the captcha to the user on WhatsApp and type their answer in (free, ToS-friendlier).
  3. Optional paid solver: 2Captcha ≈ $1–3 / 1k reCAPTCHA, disabled by default.
- **Scraper:** Playwright (Python) with `chromium-headless-shell`, one context at a time, plus a stealth patch (patchright or rebrowser) only when needed.
- **Email inbox fallback (free and very effective):**
  - Issuers are legally obliged to make the XML available and usually **email the XML and PDF to the email the buyer gives**.
  - The bot can search Gmail or Outlook for `serie-numero` and pull the attachments.

## 5. WhatsApp channel
- **Meta WhatsApp Cloud API (chosen):**
  - Since **2026-10-01**, service messages are billed per message after **1,000 free per month per number**. **A card on file is required** even at $0.
  - Budget: ≤ 200 invoices × ~3–4 messages ≈ 600–800 per month, which fits but is close. **Keep messages per invoice ≤ 4** (see R-9.3).
  - Media flow: the webhook gives a media id → `GET /{id}` returns a URL valid for **5 minutes** → download it with the Bearer token. Download immediately.
  - Supports reply buttons (≤ 3) and list messages, which we use to choose the flow.
  - Needs a dedicated number. The test number works for up to 5 allowlisted recipients.
- **Telegram** stays as a zero-risk fallback adapter. Unofficial libraries (Baileys/WAHA) are rejected because of ban risk.

## 6. Delivery integrations
- **Google (flow A):**
  - Use **OAuth installed-app with a refresh token**, not a service account. Service accounts have 0 Drive quota on personal Gmail.
  - Publish the consent screen to **"In production"**, otherwise refresh tokens expire after 7 days.
  - Scopes: `drive.file` and `spreadsheets`, plus optionally `gmail.readonly` for the inbox fallback.
- **Microsoft (flow B):**
  - Entra app registration with *"Accounts in any org directory + personal Microsoft accounts"*, delegated **device-code** auth via MSAL, and a persisted token cache.
  - Scopes: `offline_access Files.ReadWrite Mail.Send` (+ `Mail.Read` for the inbox fallback).
  - **Graph's Excel workbook API does NOT work on personal OneDrive.** So: download the .xlsx, edit it locally with **openpyxl** (which keeps the template formatting), and upload it back with an `If-Match` eTag.
  - **Work tenant:** the work admin may block third-party app consent. The plan is to use the **personal** Microsoft account and share the folder with the work account (or the reverse). See open questions.
- **Email:**
  - Gmail API (`gmail.send`) or Graph `sendMail`. Inline attachments up to ~3 MB; use an upload session above that.
  - Outlook.com SMTP basic auth has been dead since Sept 2024.

## 7. Hosting (1 GB RAM)
| Component | Est. RSS |
|---|---|
| Python core (Starlette/uvicorn, 1 worker) | 60–90 MB |
| OpenCV + zxing (per job, transient) | +60–120 MB |
| Tesseract (subprocess, transient) | +80–150 MB |
| Chromium headless-shell, one page (transient) | +200–350 MB |
| cloudflared | 20–30 MB |
| **n8n (optional)** | **250–450 MB resident** |

- **Conclusion:** the full stack *without* n8n peaks at about 600 MB and fits in 1 GB. **n8n alongside it does not fit safely** during a browser job. See design §2 and open question Q1.
- **Hosts:**
  - Any 1 GB VPS works.
  - Oracle Always Free ARM is generous but its terms changed in 2026; keep the setup portable with Docker Compose.
  - Cloudflare Tunnel provides HTTPS for the webhook with no open ports.
