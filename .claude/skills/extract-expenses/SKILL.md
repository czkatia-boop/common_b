---
name: extract-expenses
description: Extracts expense transactions from bank statement screenshots, photos, or PDFs (PKO, Erste, or any other bank) and produces a CSV file formatted for this app's "Upload expenses" import feature (the upload-btn / importUploadedExpenses function in index.html). Use this whenever the user attaches bank transaction history screenshots or images and wants them turned into expenses, asks to "import" or "upload" spending, or mentions pulling data from a bank statement — even if they don't say "CSV" or name this skill directly.
---

# Extract expenses from a bank statement

The user will attach one or more images (screenshots, photos, or PDF pages) of
their bank's transaction history. Your job is to read them, pull out the real
expenses, and hand back a CSV file they can upload directly through the app's
"Upload expenses" button — no code execution needed, this is pure reading and
formatting.

## The output format (must match exactly)

This CSV is consumed by JS code, not a person, so the format below is not
negotiable — get it wrong and rows silently fail to import.

Plain comma-separated CSV, one row per expense, optional header row:

```
date,amount,note,source,category
25.09.2026,135.00,"PANI LUCYNA - przelew na telefon",PKO,
25.09.2026,45.20,"Biedronka",PKO,Groceries
23.09.2026,129.88,"ccc.eu Polkowice",PKO,Clothes
```

Columns, in this exact order:

- **date** — `DD.MM.YYYY` or `YYYY-MM-DD`. Anything else fails to parse and the
  row gets silently dropped by the importer, so stick to one of these two.
- **amount** — a plain positive number. `.` or `,` both work as the decimal
  separator, but never include a currency symbol (`zł`, `PLN`) or a thousands
  separator. Always positive — the bank app shows debits as negative, but
  this importer is expenses-only, so drop the minus sign.
- **note** — the merchant/description text as it appears on the statement.
  Wrap it in double quotes if it contains a comma; double up any embedded
  quotes (`""`).
- **source** — must match, case-insensitively, the *name* of one of the
  user's existing sources in the app (Settings → Sources — things like
  "PKO", "Erste", "Cash", "Partner", or whatever custom name they've set
  up). If you're not sure what they've named it, ask — an unmatched name
  silently falls back to their first configured source instead of erroring,
  which is worse than asking.
- **category** — optional. Match it, case-insensitively, against the name of
  one of the user's existing categories (Settings → Categories). An empty
  value or an unmatched name both fall back to "Other" — same safe behavior
  as source, so it's fine to leave this blank on a row you're not confident
  about instead of guessing wrong silently. See "Guessing the category"
  below for how to fill this in well.

Deliver the finished CSV as an actual file (not pasted as a code block or
inline text) — it's meant to be uploaded through the app's own button.

### Guessing the category

Find out the user's actual category list before guessing — either from
context you already have in the conversation, or by asking them (Settings →
Categories in the app). Don't assume the defaults are still what they have;
categories get renamed, merged, and added over time. The stock set this app
ships with is Groceries, Kids, Transportation, Home stuff, Clothes, Health,
Fun stuff, Eating out, Bills, Travel, Savings, Selfcare/beauty, Presents, and
Other — a reasonable starting guess only if the user hasn't told you
otherwise.

Once you know the real list, match each transaction's merchant/description
against it. Some are obvious from the name alone (a supermarket chain →
Groceries, an airline → Travel, a clothing retailer → Clothes). Others
aren't — a bank transfer to a person, an ATM withdrawal, a generic-sounding
subscription — and guessing wrong there is worse than not guessing, because
a wrong category is a bug the user has to notice and fix, while a blank one
(→ "Other") is an obvious, honest gap they'll expect to fill in themselves.
When you leave a row's category blank for this reason, don't just do it
silently: mention it in your summary, the same way you'd flag an excluded
hold or an illegible row, so the user knows which ones to look at.

## What to actually extract

**Expenses only.** This import path is for spending, not income — money
leaving the account. If a statement shows a credit/deposit, don't put it in
the CSV; just mention it separately to the user in your summary.

**Skip pending holds.** Many banks flag a transaction that hasn't settled yet
— PKO shows a lock icon and the word "Blokada" next to it, for example.
Leave these out of the CSV. They'll typically post as a separate, real
transaction once they settle, so including the hold now risks a duplicate
later. Tell the user which rows you excluded for this reason.

**Skip the bank's own internal transfers by default** — things like PKO's
"Autooszczędzanie-przelew" auto-savings sweep, where money moves to the
user's own sub-account rather than actually being spent. This isn't real
spending, so leave it out — but say so explicitly, and mention that if they'd
rather track these as real expenses (e.g. under a "Savings" category), you're
happy to include them next time. Don't silently make this call forever;
just flag it each time it comes up until the user tells you their preference.

**Ask which bank/source a screenshot belongs to** if it isn't obvious from
the screenshot itself (e.g. a header like "PKO KONTO ZA ZERO" makes it
obvious; a cropped or generic-looking list doesn't). Guessing wrong here
means every expense gets silently attributed to the wrong account.

**Verify before finalizing.** Screenshots can be blurry, and digits like
0/6/8 or 1/7 are easy to misread, especially in a photo rather than a clean
screenshot. If a row is genuinely illegible, don't guess — leave it out and
tell the user which one and why, the same way you'd report a malformed row.

## Wrapping up

Once the CSV is built, give the user a short summary: how many rows made it
in, and a list of what you left out and why (pending holds, internal
transfers, illegible entries, income) — then send the file.
