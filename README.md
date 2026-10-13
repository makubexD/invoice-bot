# invoice-bot

WhatsApp bot that takes a photo of a Peruvian electronic invoice (SUNAT CPE). It:

1. extracts the key data **without an LLM** (QR first, OCR fallback),
2. asks you which workflow to run,
3. fetches the official PDF/XML,
4. delivers the result, either to **Google Drive + Sheets** or to **OneDrive + Excel template + email**,
5. replies on WhatsApp with the outcome.

The project uses **spec-driven development**. The specs below are the source of truth, and the code follows them.

| # | Document | Purpose |
|---|----------|---------|
| 0 | [specs/00-constitution.md](specs/00-constitution.md) | Non-negotiable principles (cost, 1 GB RAM, no AI, security) |
| 1 | [specs/01-research.md](specs/01-research.md) | What exists, what to reuse, with sources |
| 2 | [specs/02-requirements.md](specs/02-requirements.md) | User stories plus testable acceptance criteria (EARS) |
| 3 | [specs/03-design.md](specs/03-design.md) | Architecture, components, data model, state machine, resource budget |
| 4 | [specs/04-tasks.md](specs/04-tasks.md) | Phased implementation plan, traced to requirements |
| 5 | [specs/05-open-questions.md](specs/05-open-questions.md) | Decisions still pending from the owner |
| – | [config/example.yaml](config/example.yaml) | Shape of the user-editable configuration |

Status: **planning**. No code yet.
