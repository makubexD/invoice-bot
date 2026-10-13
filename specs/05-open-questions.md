# 05 · Open questions and decisions

## Decided (from the Q&A on 2026-10-13)
| # | Decision |
|---|---|
| D1 | Country: **Peru (SUNAT)** |
| D2 | Channel: **official WhatsApp Cloud API**, with Telegram as an optional fallback |
| D3 | Users: several allowlisted people, **< 200 invoices per month** |
| D4 | Flow selection: **ask in chat**. The bot suggests a default from the buyer RUC |
| D5 | Flow A: buyer is the owner's company RUC (20…); **SUNAT is the fallback** (owner has SOL) |
| D6 | Flow B: buyer is the employer's RUC; **no SUNAT access** → portals, then email, then manual |
| D7 | Hosting: **1 GB RAM VPS**, highly optimized (see constitution P2) |
| D8 | Build style: **code first + n8n**. Revised by the 1 GB constraint: n8n is an optional sink (Q1) |
| D9 | Flow B email: **all three trigger modes** (per invoice, on command, scheduled), configurable and combinable |
| D10 | Flow B Excel: **existing template, update specific cells** |
| D11 | When every fallback fails → **MANUAL_REQUIRED** and the user is told what is missing |

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
