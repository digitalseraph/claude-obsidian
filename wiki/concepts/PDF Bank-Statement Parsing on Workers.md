---
type: concept
title: "PDF Bank-Statement Parsing on Cloudflare Workers"
created: 2026-05-17
updated: 2026-05-17
tags:
  - pdf-parsing
  - cloudflare-workers
  - cloudflare-d1
  - unpdf
  - pippa
status: developing
related:
  - "[[Source-First Synthesis]]"
---

# PDF Bank-Statement Parsing on Cloudflare Workers

Reusable, non-obvious engineering knowledge from building multi-bank PDF
statement import for the Pippa personal-finance app (Next.js on Cloudflare
Workers via OpenNext, Drizzle + D1, `unpdf` for text extraction). These points
are not derivable from reading the code; each was found by debugging real
failures against real statements.

## unpdf returns one newline-free blob

`unpdf`'s `extractText(bytes, { mergePages: true })` returns the entire
multi-page PDF as a single string with no newlines: words are space-joined
across the whole document. Any parser that splits on `\n` and runs a per-line
state machine silently returns zero rows. A hand-authored fixture with
newlines makes the unit test pass while production parses nothing. The fix is
a global regex tokenizer over the continuous text, with section context
assigned by header byte-offset rather than by line position.

## PNC bank statement layout

Column order is Date, then Amount, then Description (not the intuitive
Date / Description / Amount). Sign comes from the section header the record
falls under: deposit/credit sections are positive, withdrawal/debit sections
are negative. The billing period appears as `For the Period MM/DD/YYYY to
MM/DD/YYYY`. Records carry `MM/DD` only; the year is resolved from the
extracted period (handles cross-year statements).

## Capital One credit-card layout is fundamentally different

Capital One Venture statements share no structure with a bank statement:

- Billing cycle: `Mon DD, YYYY - Mon DD, YYYY` (month name, comma, year).
  Requiring the comma and year in the period regex disambiguates it from
  record dates, which have neither, so the two never cross-match.
- Records: `Mon DD  Mon DD  DESCRIPTION  [- ]$AMOUNT` (Trans Date, Post Date
  as month abbreviation plus day, no year on the row). Credits carry a
  leading `- ` before the dollar sign.
- Sign convention is inverted versus a bank: payments, credits and
  adjustments are positive (they reduce what is owed); purchases, fees and
  interest are negative. A single generic heuristic parser cannot get signs
  right across bank and card statements, so per-issuer parsers are required.

This is why the design is a `BankStatementParser` interface
(`extractPeriod(text)` plus `parse({ text, period })`) with one
implementation per institution, selected by a registry.

## Cloudflare D1 bound-parameter cap

D1 caps bound parameters per statement at roughly 100. A multi-row INSERT
binds `columnsPerRow * rowsInChunk` parameters, so chunking must be derived
from the column count (`floor(maxParams / columnsPerRow)`), never a fixed row
count. A fixed chunk such as 50 rows silently overflows once the row is wide
enough (about 12 columns gives roughly 600 parameters) and D1 throws. If the
route has no try/catch, the Worker returns an empty body and the browser
client surfaces it as the cryptic `Failed to execute 'json' on 'Response':
Unexpected end of JSON input`. Always wrap route pipelines so a throw returns
structured JSON, and log the stack for `wrangler tail`.

## Production account rows are Plaid-sourced, not seeded

Production `accounts` rows come from Plaid with opaque generated IDs
(`acct_plaid_*`), not the seeded `acct_*` IDs used in local development. A
migration that backfills a new column by seeded IDs no-ops in production for
those institutions. Always verify the backfill against live D1
(`wrangler d1 execute --remote`) and, where the UI hardcodes account choices,
point them at the real production IDs. Plaid IDs can rotate if the Item is
re-linked; a dynamic importable-accounts list is the durable fix.

## Evidence-driven, PII-safe debugging method

Financial PDFs cannot be pasted into logs. The method that works:

1. Pull the real failing file (from R2, or a local copy on the same machine).
2. Run the exact production extractor (`unpdf` with the same options) so the
   reproduction matches reality, not an idealized fixture.
3. Print only PII-safe shape: replace `[A-Za-z]` with `a` and `[0-9]` with
   `9`, keep punctuation, plus structural counts and which section headers
   matched. Never print real descriptions, amounts, or account numbers.
4. From the shape, derive the real regexes; rebuild the fixture to mirror the
   real layout with synthetic values; finalize against all available real
   statements.
5. Delete the local copy and the temporary diagnostic afterward.

This is the same loop that took the PNC parser from 0 to 109 rows and the
Capital One parser from non-functional to correct across four real monthly
statements (including a cross-year December-to-January cycle).

## Defense-in-depth for free-text descriptions

A lazy description capture between two delimiter dates can run away into
end-of-statement legalese when the next boundary is far off. Bound the
description by the nearest of: the next record's date+amount lookahead, the
next section-header offset, a furniture/legalese phrase stop-list, and a hard
character cap (about 160). The cap guarantees no single record can absorb a
page regardless of unrecognized furniture. The review-before-commit step
absorbs the rare residual phantom row.
