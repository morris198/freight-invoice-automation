# Freight Invoice Automation

An automated accounts-payable pipeline that reads freight invoices, extracts the
data with an LLM, validates it against business rules, **auto-approves the clean
ones, and escalates exceptions to a human** — then stores everything in SQL for
reporting.

## The problem

Accounts-payable teams manually open every invoice, key the numbers into a
system, and eyeball whether it looks right. It's slow, it's repetitive, and tired
humans miss things — a total that doesn't add up, a duplicate, an amount that
should have had a manager's sign-off.

This pipeline handles the routine automatically and surfaces only the cases that
genuinely need a person: the *routine is handled, the judgment calls are
escalated.*

## What it does (sample run)

Three invoices in, three different decisions out — each one exercises a different
branch of the decision logic:

```
Processing invoice_clean.txt
  Parsed invoice INV-20451 from Acme Logistics Supply Co.
  -> AUTO_APPROVED (doc id 1)

Processing invoice_bad_math.txt
  Parsed invoice RPS-7788 from Roadway Parts and Service LLC
  -> NEEDS_REVIEW: Line items + tax (1337.50) do not match stated total (1900.00)

Processing invoice_large.txt
  Parsed invoice NFL-30012 from National Fleet Leasing Inc.
  -> NEEDS_REVIEW: Total 13749.50 exceeds auto-approve ceiling 10000.00
```

And the operational report any manager would want:

```
=== REVIEW QUEUE (human action needed) ===
  NFL-30012  National Fleet Leasing Inc.  $13,749.50   over approval ceiling
  RPS-7788   Roadway Parts and Service    $ 1,900.00   totals don't reconcile

=== AUTO-APPROVED PAYABLES ===
  1 invoices, $3,702.20 payable

=== EXCEPTION RATE BY VENDOR ===
  Roadway Parts and Service LLC   1/1 flagged (100.0%)
  National Fleet Leasing Inc.     1/1 flagged (100.0%)
  Acme Logistics Supply Co.       0/1 flagged (0.0%)
```

The clean invoice flows straight through. The one whose line items don't reconcile
to the total is caught. The high-dollar one is valid but escalated anyway because
it's over the approval ceiling — a *policy* decision, not an error.

## Architecture

```
document (PDF / scan / text)
        |
        v
1. extraction.py   ->  raw text        (pdfplumber for text PDFs, OCR for scans)
        |
        v
2. llm_extract.py  ->  structured JSON  (LLM + strict schema; mock mode for offline runs)
        |
        v
3. validation.py   ->  decision         (deterministic business rules)
        |                                 AUTO_APPROVED | NEEDS_REVIEW + reasons
        v
4. storage.py      ->  SQL              (documents + line_items, with reporting queries)
```

**Design principle: the LLM extracts, but auditable code decides.** A language
model never has the final say on whether to pay an invoice — plain, testable
Python rules make that call, and anything suspicious is routed to a human with a
readable reason.

| Stage | File | Responsibility |
|-------|------|----------------|
| Ingest | `extraction.py` | PDF text, OCR for scans, plain text |
| Structure | `llm_extract.py` | LLM returns JSON matching a fixed schema; mock mode runs offline |
| Decide | `validation.py` | Math reconciliation, required fields, duplicates, approval ceiling |
| Store | `storage.py` | Normalised schema (FK), inserts, aggregate reporting queries |
| Orchestrate | `pipeline.py` | Wires the stages together, logging, CLI |

## Quick start

Runs out of the box with **no API key and no dependencies** — mock mode stands in
for the model so you can see the whole pipeline work:

```bash
python pipeline.py sample_data/invoice_clean.txt sample_data/invoice_bad_math.txt sample_data/invoice_large.txt
python pipeline.py --report
```

Use the real Anthropic model:

```bash
pip install anthropic
export ANTHROPIC_API_KEY=sk-...
python pipeline.py --real sample_data/invoice_clean.txt
```

Process real PDFs / scans (adds OCR):

```bash
pip install pdfplumber pytesseract pillow   # plus the tesseract binary
python pipeline.py path/to/real_invoice.pdf
```

## Key engineering decisions

- **Structured output** — the model is forced to return JSON matching a fixed
  schema, so downstream code acts on data, not prose.
- **Swappable extractor** — the API call sits behind one interface with a mock
  implementation, so the pipeline is testable offline, with no cost and no
  network flakiness.
- **Deterministic validation** — the math, duplicate, and approval-ceiling checks
  are plain Python you can read, test, and audit. That's what makes it safe to
  point at money.
- **One document = one transaction** — a document's header and line items commit
  together, so a failure never leaves orphaned rows.
- **Resilient batch** — each document runs in its own `try/except`; one bad file
  logs an error and the batch keeps going.

## Roadmap

- [ ] `JOIN` query: flagged invoices with their line-item detail
- [ ] `UPDATE` / review workflow: mark a flagged invoice reviewed + approved
- [ ] FastAPI layer: submit a document, read the review queue, approve an invoice
- [ ] Second document type (rate confirmation) with its own schema
- [ ] Retries with exponential backoff for API rate limits

## Tech

Python · Anthropic API · SQLite (portable to Postgres) · pdfplumber / Tesseract OCR

---

*Built as a demonstration of an end-to-end, LLM-powered document-automation
pattern with human-in-the-loop controls.*
