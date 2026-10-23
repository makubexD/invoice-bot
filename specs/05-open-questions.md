# 05 · Open questions and decisions

## Decided (Q&A on 2026-10-13 and 2026-10-23)
| # | Decision |
|---|---|
| D1 | Country: **Peru (SUNAT)** |
| D2 | Channel: **official WhatsApp Cloud API**, with Telegram as an optional fallback |
| D3 | Users: several allowlisted people, **< 200 invoices per month** |
| D4 | ~~Ask in chat~~ → **revised 2026-10-23:** the flow is picked **automatically from the buyer RUC** (owner → A, work → B). A menu appears only when nothing matches (BR-R03) |
| D5 | Flow A: buyer is the owner's company RUC (20…); **SUNAT is the fallback** (owner has SOL) |
| D6 | Flow B: buyer is the employer's RUC; **no SUNAT access** → portals, then email, then manual |
| D7 | Hosting: **1 GB RAM VPS**, highly optimized (see constitution P2) |
| D8 | Build style: **code first + n8n**. Revised by the 1 GB constraint: n8n is an optional sink (Q1) |
| D9 | Flow B email: **all three trigger modes** (per invoice, on command, scheduled), configurable and combinable |
| D10 | Flow B Excel: **existing template, update specific cells** |
| D11 | When every fallback fails → **MANUAL_REQUIRED** and the user is told what is missing |
| D12 | Unknown, DNI or empty buyer → **ask then, and only then**, with buttons Mi empresa / Trabajo / Cancelar |
| D13 | Missing capability → **GitHub issue + draft PR + red CI** (spec 06 §3) |
| D14 | Unknown store portal → the bot **asks the user for the link** (once per issuer) and puts it into the draft PR |
| D15 | Admin alerts also go to **WhatsApp** (in addition to GitHub's own notifications) |

## Open (needed before or during Phase 0)
| # | Question | Why it matters | Default if unanswered |
|---|---|---|---|
| Q1 | OK with n8n as an *optional* add-on (off on the 1 GB host), with the core in Python? Or do you prefer a 2 GB host (~US$4–6/mo) so n8n runs alongside? | RAM budget | Core in Python, n8n off |
| Q2 | Microsoft accounts: is your personal account outlook.com/hotmail (MSA)? Does your work tenant allow consenting to a third-party app? Should the bot write to your **personal** OneDrive and share that folder with your work account, or should the work account share a folder with your personal one? | Determines auth and folder location | Personal MSA OneDrive, folder shared with work |
| Q3 | Email for flow B: which provider sends it (Outlook personal through Graph, or Gmail), and who receives it (to and cc)? | Mailer config | Graph from the personal account |
| Q4 | Can you share the **Excel template** (cells to fill, sheet name, .xlsx or .xlsm, formulas, and whether a batch means one file per invoice or one file with many)? | Cell mapping; batch format | One file per invoice |
| Q5 | Which email do you give restaurants and stores for the electronic invoice? Can the bot read that inbox (Gmail or Outlook)? | The `email_inbox` fetcher is the strongest free fallback, especially for flow B | Gmail, read-only scope |
| Q6 | Google Sheet columns and Drive folder structure for flow A: keep the proposed defaults, or do you have an existing sheet? | Sheets mapping | Proposed defaults |
| Q7 | Required docs per flow: should flow B still deliver if only the PDF *or* only the XML is found? | Delivery rule | Both are required for B, XML for A |
| Q8 | Do you also need **boletas** (B-series, buyer DNI) or only **facturas** (F-series)? | Boletas often lack a buyer RUC; SOL won't list them | Facturas first, boletas best-effort |
| Q9 | Any currency other than PEN (USD invoices)? | Parsing and Sheets formatting | PEN, USD read from XML when present |
| Q10 | Comfortable creating a **secondary SOL user** with query-only permissions for the bot? | Security of the `sunat_sol` fetcher | Yes |
| Q11 | Do you already have a VPS provider or a domain for Cloudflare Tunnel? | Deploy | Any 1 GB VPS, plus a free `trycloudflare` URL for dev |
| Q12 | Is the GitHub repo **private** or **public**? | Private is recommended. The gap issues are already privacy-safe (spec 06), but the real `config.yaml` (your RUCs, phone numbers) must never be committed to a public repo | Private; the real config lives only on the VPS |
| Q13 | GitHub credential for the bot: a **fine-grained PAT** (5 min to set up, expires after ≤ 1 year, and the bot warns 14 days before) or a **GitHub App** (no expiry, ~20 min to set up)? | Auth for R-11.5 | Fine-grained PAT |
| Q14 | Who gets the admin WhatsApp alerts: only you, or more numbers? | `gaps.admins` | Only you |
| Q15 | Who fixes a gap PR? Only you, or should we enable **Claude Code on GitHub** (mention `@claude` on the issue) to draft the portal selectors for you to review? | It turns a gap into "review and merge" | You; `@claude` is optional |
| Q16 | OK for the VPS to **pull `main` every 10 min** and hot-reload the config (so a merged portal fix is live within ~10 min, with no manual deploy)? | R-11.9 auto-resume | Yes |
| Q17 | `confirm_mode: auto` (no tap needed when everything reads fine; you can still tap Editar/Cancelar) vs `always` (tap to confirm every invoice)? | UX and message count | `auto` |
