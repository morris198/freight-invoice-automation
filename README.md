# freight-invoice-automation
Freight Invoice Automation

An automated accounts-payable pipeline that reads freight invoices, extracts the data with an LLM, validates it against business rules, auto-approves the clean ones, and escalates exceptions to a human — then stores everything in SQL for reporting.

The problem

Accounts-payable teams manually open every invoice, key the numbers into a system, and eyeball whether it looks right. It's slow, it's repetitive, and tired humans miss things — a total that doesn't add up, a duplicate, an amount that should have had a manager's sign-off.

This pipeline handles the routine automatically and surfaces only the cases that genuinely need a person: the routine is handled, the judgment calls are escalated.




Processing invoice_clean.txt
  Parsed invoice INV-20451 from Acme Logistics Supply Co.
  -> AUTO_APPROVED (doc id 1)

Processing invoice_bad_math.txt
  Parsed invoice RPS-7788 from Roadway Parts and Service LLC
  -> NEEDS_REVIEW: Line items + tax (1337.50) do not match stated total (1900.00)

Processing invoice_large.txt
  Parsed invoice NFL-30012 from National Fleet Leasing Inc.
  -> NEEDS_REVIEW: Total 13749.50 exceeds auto-approve ceiling 10000.00
  === REVIEW QUEUE (human action needed) ===
  NFL-30012  National Fleet Leasing Inc.  $13,749.50   over approval ceiling
  RPS-7788   Roadway Parts and Service    $ 1,900.00   totals don't reconcile

=== AUTO-APPROVED PAYABLES ===
  1 invoices, $3,702.20 payable

=== EXCEPTION RATE BY VENDOR ===
  Roadway Parts and Service LLC   1/1 flagged (100.0%)
  National Fleet Leasing Inc.     1/1 flagged (100.0%)
  Acme Logistics Supply Co.       0/1 flagged (0.0%)
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
4. storage.py      ->  SQL              (documents + line_items, with reporting queries
