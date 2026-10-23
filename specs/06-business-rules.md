# 06 · Business rules, user messages and capability gaps

Every time the bot stops, warns or changes route, it does so because of **one named rule**. Each rule has:
- an ID (`BR-…`);
- a condition;
- the resulting state;
- the exact chat message (es-PE). Messages live in code as a catalog (`pipeline/messages_es.py`), keyed by rule ID, so they are testable and easy to reword.

When the root cause is something **missing from the repository** (a portal, a parsing rule, a template mapping), the rule also raises a **capability gap** (§3). The gap opens a GitHub issue plus a draft PR, keeps CI red, and alerts the admin on WhatsApp.

Placeholders like `{razon_social}` are filled from the invoice. Messages stay short: one idea, one action.

---

## 1. Routing rules (which flow handles the invoice)
Rules are evaluated **in order; the first match wins** (R-3.1). The buyer is `num_doc_adq` from the QR or OCR.

| ID | Condition | Result | Chat message |
|---|---|---|---|
| BR-R01 | Buyer RUC = `owner.ruc` | Flow A (`personal_google`), silently | *(shown inside the summary)* `🏢 Mi empresa → Google` |
| BR-R02 | Buyer RUC = `work.ruc` | Flow B (`work_microsoft`), silently | *(inside the summary)* `💼 Trabajo → Microsoft` |
| BR-R03 | Buyer is missing, is a DNI, or is a RUC that matches no rule | `CONFIRMING`; ask with buttons **Mi empresa / Trabajo / Cancelar** | `Este comprobante está a nombre de {buyer_desc}, que no es tu empresa ni tu trabajo. ¿Dónde lo registro?` |
| BR-R04 | The buyer was read by OCR with confidence below `review_threshold` | `CONFIRMING`, with the detected flow pre-selected | `No pude leer bien el RUC del cliente ({num_doc_adq}?). Lo registro en {flow.label}. ¿Correcto?` [Sí] [Cambiar] [Cancelar] |
| BR-R05 | The routed flow isn't allowed for the sender's profile | `CANCELLED` | `No tienes permiso para registrar comprobantes de {flow.label}. Pídele al administrador que te habilite.` |
| BR-R06 | The same unknown buyer RUC is seen 3 or more times in 30 days | Raises gap `UNROUTED_BUYER`, then continues as BR-R03 | *(no extra chat message; the admin is alerted)* |

`{buyer_desc}`: `el RUC 20… (Razón social)`, `el DNI 4…`, or `ningún cliente (boleta sin datos)`.

## 2. Pipeline rules

### Extraction (E)
| ID | Condition | Result | Chat message |
|---|---|---|---|
| BR-E01 | No QR, and OCR is missing required fields | `EXTRACT_FAILED` (waits for the user) | `No pude leer este comprobante. Envíame una foto más cerca y con buena luz, o escribe los datos así:` + `ruc=… serie=F001 numero=123 fecha=dd/mm/aaaa total=…` |
| BR-E02 | The issuer RUC fails the check digit | Same as E01, with the field highlighted | `El RUC {ruc} no es válido (dígito verificador). Corrígelo con ruc=…` |
| BR-E03 | The date is in the future or older than `max_invoice_age_days` | `CONFIRMING` | `La fecha {fecha} parece incorrecta. ¿La corrijo? fecha=dd/mm/aaaa` |
| BR-E04 | The natural key already exists and isn't failed | Stop | `Este comprobante ya lo registré el {created_at:%d/%m} (#{id}). Si quieres repetirlo: /reprocesar {id}` |
| BR-E05 | Not a supported CPE (`tipo` ∉ config) | `CANCELLED` | `Solo proceso facturas y boletas electrónicas. Esto parece {tipo_desc}.` |
| BR-E06 | OCR fails 2 or more times on photos from the same issuer | Raises gap `PARSER_MISS` | *(no extra message; E01 already asked the user)* |

### SUNAT validation (V)
| ID | Condition | Result | Chat message |
|---|---|---|---|
| BR-V01 | `estadoCp = 0` (doesn't exist yet) | Continue; re-check in 24 h, max 3 times | *(one line in the summary)* `⏳ SUNAT aún no lo registra; vuelvo a revisar mañana.` |
| BR-V02 | `estadoCp = 2` (annulled) | `CONFIRM_INVALID` | `⚠️ SUNAT dice que este comprobante está ANULADO. ¿Lo registro igual?` [Sí] [Cancelar] |
| BR-V03 | Issuer RUC isn't `ACTIVO`/`HABIDO` | Continue; warn | *(one line in the summary)* `⚠️ El emisor figura {estado}/{condicion} en SUNAT.` |
| BR-V04 | The SUNAT API is down or credentials are missing | Continue as "not validated"; retry later | *(one line in the summary)* `ℹ️ No pude validar con SUNAT ahora; lo intento luego.` |

### Document retrieval (F)
| ID | Condition | Result | Chat message |
|---|---|---|---|
| BR-F01 | Flow B: `sunat_sol` isn't applicable (not the owner's RUC) | Skipped by design; **not a gap** | *(only listed in the F07 message)* `• SUNAT: no aplica (solo para facturas de tu empresa)` |
| BR-F02 | No portal entry for the issuer RUC, and no `qr_url` | Raises gap `PORTAL_MISSING`; the chain **continues**; asks the user to teach the link (once per issuer) | `No conozco el portal de descarga de {razon_social} (RUC {ruc_emisor}). Si lo tienes (suele estar impreso: "Consulte su comprobante en…"), envíamelo. Si no, responde "no sé".` |
| BR-F03 | A portal adapter fails `broken_after` (default 3) times in a row for technical reasons (selectors, HTTP 4xx/5xx, timeouts; not "invoice not found") | Raises gap `PORTAL_BROKEN`; the chain continues | *(only in the F07 message)* `• Portal {portal}: con fallas, ya avisé para arreglarlo` |
| BR-F04 | reCAPTCHA or hCaptcha, and no paid solver | Skip to the next fetcher | *(only in F07)* `• Portal {portal}: captcha que no puedo resolver` |
| BR-F05 | The user doesn't answer an image captcha within `ask_user_timeout_s` | Skip to the next fetcher | `Se venció el tiempo del captcha; sigo con otras opciones.` |
| BR-F06 | The fetched XML doesn't match the invoice (RUC, serie, número or total) | Discard it; next fetcher | *(only in F07)* `• {fetcher}: el XML no coincide` |
| BR-F07 | Every fetcher failed | `MANUAL_REQUIRED` (auto-resumable when a linked gap is fixed) | `❗ No pude obtener el XML/PDF de {serie}-{numero}. Intenté:` + one `•` line per fetcher + `Puedes enviarme el PDF/XML aquí y lo completo. {gap_line}` |

`{gap_line}`, when a gap is linked: `Ya avisé para agregar este portal; cuando esté listo lo reintento solo.`

### Delivery (D)
| ID | Condition | Result | Chat message |
|---|---|---|---|
| BR-D01 | A Google or Microsoft token expired or was revoked | `PENDING_AUTH` (auto-resumes after re-auth); **admin-only alert**, not a repo gap | User: `Tu comprobante quedó en cola; hay un problema de conexión con {provider}. Te aviso al terminar.` Admin: `🔑 Reautoriza {provider}: ejecuta scripts/auth_{provider}.py` |
| BR-D02 | The Excel template is missing, or a `cells:` expression references an unknown field | Raises gap `TEMPLATE_INVALID`; `MANUAL_REQUIRED` | `No pude llenar la plantilla de Excel (configuración incompleta). Ya avisé para corregirlo; reintento automáticamente.` |
| BR-D03 | Email send failed after retries | `FAILED` (retry with `/reprocesar`); admin alert | `No pude enviar el correo del lote #{batch}. Reintenta con /enviar o /reprocesar.` |
| BR-D04 | `/enviar` with an empty batch | No-op | `No hay comprobantes pendientes de envío.` |
| BR-D05 | The GitHub token expires in less than 14 days | Admin-only alert, daily | Admin: `🔑 El token de GitHub vence el {date}. Renuévalo.` |

---

## 3. Capability gaps
A **gap** is a missing capability that only a change to the repository can fix. Gaps are deduplicated by **fingerprint**, so 10 invoices from the same unknown store make **1** issue, **1** PR and **1** admin alert.

| Kind | Fingerprint | Raised by | GitHub | Admin WhatsApp | Auto-resume when |
|---|---|---|---|---|---|
| `PORTAL_MISSING` | `portal_missing:{ruc_emisor}` | BR-F02 | Issue + **draft PR** adding a `status: todo` stub to `config/portals.yaml` (with the taught URL, if any) | Yes | A deployed registry has an `active` entry matching the RUC |
| `PORTAL_BROKEN` | `portal_broken:{portal_key}` | BR-F03 | Issue + draft PR setting the entry to `status: broken` | Yes | The entry is `active` again and the version has changed |
| `TEMPLATE_INVALID` | `template_invalid:{flow}:{sink}` | BR-D02 | Issue | Yes | The config reloads and passes validation |
| `PARSER_MISS` | `parser_miss:{ruc_emisor}` | BR-E06 | Issue | Digest only | — (the user already typed the data) |
| `UNROUTED_BUYER` | `unrouted:{num_doc_adq}` | BR-R06 | Issue ("add a routing rule or a new flow?") | Digest only | — |

**What is *not* a gap:** expected business outcomes (flow B has no SOL access, an annulled invoice, a duplicate) and operations problems (expired OAuth tokens, SUNAT down). Those get a chat message and, for operations, an admin alert, but nothing in GitHub.

### Gap lifecycle
```
detected ─► OPEN (row in `gaps`, linked invoices) ─► REPORTED (issue + PR created, admin alerted)
            ─► FIXED (a deploy makes the capability available → linked invoices re-enqueued, users notified)
            ─► CLOSED (the PR merge closes the issue via "Closes #n")
       OPEN/REPORTED ─► IGNORED (admin replies /gap ignorar <id>: no more alerts for this fingerprint)
```

### What goes into GitHub (privacy)
Only **public or technical** data:
- issuer RUC and razón social (public in the padrón);
- CPE type and serie (no número);
- fetcher reasons;
- the portal URL (taught or printed).

**Never:**
- buyer RUC or DNI, totals, phone numbers, photos;
- any text the user typed other than the validated URL.

This keeps the repo safe even if it becomes public.
