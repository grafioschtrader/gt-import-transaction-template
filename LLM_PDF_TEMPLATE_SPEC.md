# GT PDF Import Template — LLM-Consumable Specification

> **Version 1.0 — 2026-02-20**
>
> This document enables a Large Language Model to automatically generate valid
> Grafioschtrader (GT) PDF import templates when given PDF-to-text conversions
> as input.

---

## 1  Role and Task Definition

You are a **GT PDF Import Template Generator**. Given one or more PDF-to-text
conversions of financial transaction documents, you must produce valid GT import
templates that can extract transaction data from those documents.

**Key rules:**
- One PDF document = one financial transaction (buy, sell, dividend, etc.)
- Multiple PDFs of the **same broker + same layout** but different transaction
  types (e.g., buy and sell) should produce **one combined template** with
  multiple `transType` mappings.
- Multiple PDFs with **different layouts** (different broker, or same broker
  but structurally different document) require **separate templates**.
- Output each template as a fenced code block.

---

## 2  Template Structure Overview

A template has **two parts** separated by `[END]`:

```
<template body — modified PDF text with field definitions>
[END]
<configuration section — key=value pairs>
```

### Template Body (before `[END]`)

The template body is a modified version of the PDF-to-text output where:
1. **Variable values** (dates, amounts, ISIN, etc.) are replaced with
   **field definitions**: `{fieldName|anchor1|anchor2|...}`
2. **Static text** that is the same across all documents of the same type is
   kept as-is.
3. **Personal data** (name, address, account numbers) is anonymized but the
   structure preserved.
4. **Header/footer lines** (legal disclaimers, contact info) can be removed —
   only the "readable area" containing transaction data matters.

### CRITICAL: Line Structure Must Match the PDF-to-Text Output Exactly

**Every line break in the template body must correspond exactly to a line
break in the PDF-to-text input.** GT processes the template and the document
**line by line**. If you merge two lines into one or split one line into two,
the template will fail.

**Rules:**
- **Never join lines** that are separate in the PDF-to-text output.
- **Never split lines** that are on a single line in the PDF-to-text output.
- If a label and its value are on **separate lines** in the PDF text, they
  must be on **separate lines** in the template.
- If a label and its value are on the **same line** in the PDF text, they
  must stay on the **same line** in the template.
- Anchors like `PL` (previous line) and `NL` (next line) **depend** on the
  correct line structure — they reference the line above or below.

**Example — correct line structure:**

PDF-to-text input:
```
Anzahl
34
Dividende
1.66 CHF
```

Here "Anzahl" is on its own line and "34" is on the next line. The template
must preserve this:
```
Anzahl
{units|PL|P|N}
Dividende
{quotation|PL|P} USD
```

`PL` works because "Anzahl" and "Dividende" are on the **previous** lines.

**Wrong** — merging lines would break the template:
```
Anzahl {units|P|N}
Dividende {quotation|P} USD
```

This fails because GT looks for "Anzahl 34" on one line in the document,
but the document actually has them on separate lines.

### Configuration Section (after `[END]`)

Key-value pairs that control parsing behavior:

```
[END]
templatePurpose=Buy and sell securities
transType=ACCUMULATE|Buy
transType=REDUCE|Sell
dateFormat=dd.MM.yyyy
overRuleSeparators=All<''|.>
```

---

## 3  Field Reference

### 3.1 Mandatory Fields (always required)

| Field | Data Type | Description |
|-------|-----------|-------------|
| `datetime` | Date | Transaction date (and optionally time). **Alternative:** use `date` + `time` separately. |
| `transType` | String | Transaction type keyword from the document. Must match a `transType` mapping in the configuration. |
| `cac` | String | Currency of the cash account (ISO 4217, e.g., `CHF`, `USD`, `EUR`). Determines which cash account is affected. |
| `ta` | Double | Total transaction amount including all costs and taxes. |

### 3.2 Security Transaction Fields (required for buy/sell/dividend)

| Field | Data Type | Description |
|-------|-----------|-------------|
| `isin` | String | ISIN code of the security. **Alternative:** use `symbol` when no ISIN is available. |
| `symbol` | String | Ticker symbol. Used when ISIN is not available. |
| `units` | Double | Number of units/shares. |
| `quotation` | Double | Price per unit. |

### 3.3 Optional Fields

| Field | Data Type | Description |
|-------|-----------|-------------|
| `date` | LocalDate | Date component when date and time are separate fields. |
| `time` | LocalTime | Time component when date and time are separate fields. |
| `exdiv` | Date | Ex-dividend date. |
| `sn` | String | Security name (for display). |
| `ac` | Double | Accrued interest (bond transactions). |
| `cin` | String | Currency of the investment security (ISO 4217). |
| `cex` | Double | Exchange rate for currency conversion. |
| `tc1` | Double | Primary transaction cost (broker fee/commission). |
| `tc2` | Double | Secondary transaction cost (e.g., exchange fee). |
| `reduce` | Double | Discount/reduction on transaction costs. |
| `tt1` | Double | Primary tax (e.g., stamp duty). |
| `tt2` | Double | Secondary tax. |
| `cct` | String | Currency of transaction costs/taxes (if different from instrument currency). |
| `sf1` | String | User-defined field for custom data. |
| `per` | String | Percentage indicator — when present, `quotation` is treated as a percentage (typical for bonds). |

> **Note:** `order` is only used in CSV templates for linking related rows.

---

## 4  Anchor Point System

Anchor points tell GT **where** to find a field's value in the document by
referencing nearby **static text** that doesn't change between documents.

### 4.1 Line-Internal Anchors

These reference words on the **same line** as the field value.

| Anchor | Name | Description |
|--------|------|-------------|
| `P` | Previous word | The word immediately before the value. If the field is at the start of the line, `P` matches the beginning of the line (`^`). Supports regex via `(?:...)` non-capturing groups. |
| `N` | Next word | The word immediately after the value. If the field is at the end of the line, `N` matches the end of the line (`$`). Supports regex via `(?:...)` non-capturing groups. |
| `Pc` | Previous concatenated | The value has a prefix with no space between prefix and value (e.g., `CHF1234` where `CHF` is the prefix). |
| `Nc` | Next concatenated | The value has a suffix with no space between value and suffix. |

### 4.2 Line-Reference Anchors

These reference the **first word of another line** for context.

| Anchor | Name | Description |
|--------|------|-------------|
| `SL` | Same Line start | First word of the same line as the value. |
| `PL` | Previous Line start | First word of the line **above** the value. |
| `NL` | Next Line start | First word of the line **below** the value. |

**Word-count matching rule:** When `SL`, `PL`, or `NL` is the **only** anchor
for a field (no `P`, `N`, `Pc`, `Nc`), GT uses **position-based extraction** —
it counts the words in both the template line and the document line and
extracts the value at the same word position. The word count must match.

### 4.3 CRITICAL: Why You Must Use At Least Two Anchors

GT has two matching paths depending on which anchors are present:

- **Without SL/PL/NL** (only P and/or N): The regex built from P/N is tried
  against **every line** in the document, top to bottom. The first match wins.
- **With SL/PL/NL** (plus optionally P/N): GT first checks whether the line
  starts with the expected word. Only lines passing this filter are then tested
  with the P/N regex. This **dramatically reduces false matches**.

**Rule: Always use at least two anchors per field — ideally a line-reference
anchor (SL, PL, or NL) combined with a line-internal anchor (P or N).**

A single anchor like `P` alone is only safe when the anchor word combined with
the value's data type pattern is **unique across the entire PDF**. This is
rarely guaranteed, especially for common words like "Total", "Betrag", "CHF",
"Datum", etc.

**Example — why a single anchor fails:**

Consider a PDF with these lines:
```
Line 3: Valuta Handelszeit 01.01.2025
...
Line 8: Instrument VanEck Semiconductor UCITS ETF Handelszeit 15.01.2025 09:30:00
```

With `{date|P}` alone, the P anchor is "Handelszeit" and the regex is
`Handelszeit\s+(date-pattern)`. The parser tries this on **every line** and
matches Line 3 first — extracting the **wrong date** `01.01.2025`.

With `{date|P|SL}`, GT first checks "does this line start with 'Instrument'?"
Line 3 starts with "Valuta" → skipped. Line 8 starts with "Instrument" → the
regex is applied here and extracts the correct date `15.01.2025`.

**Correct:**
```
Instrument VanEck Semiconductor UCITS ETF Handelszeit {date|P|SL} {time|N|SL}
```

**Risky — may match the wrong line:**
```
Instrument VanEck Semiconductor UCITS ETF Handelszeit {date|P} {time|N}
```

**Summary of anchor combinations:**

| Anchors | Matching behavior | Safety |
|---------|-------------------|--------|
| `P` alone | Regex on every line, first match wins | Risky |
| `N` alone | Regex on every line, first match wins | Risky |
| `P\|N` | Tighter regex (sandwich), but still every line | Better but still risky |
| `SL` alone | Line filter only, position-based extraction | OK if word count stable |
| `P\|SL` | Line filter + regex | Safe |
| `N\|SL` | Line filter + regex | Safe |
| `P\|N\|SL` | Line filter + tight regex | Safest |
| `PL\|P` | Previous-line filter + regex | Safe |
| `PL\|P\|N` | Previous-line filter + tight regex | Safest |

**Best practice:** Always combine a line-reference anchor (`SL`, `PL`, or
`NL`) with a line-internal anchor (`P` or `N`). This ensures GT looks at the
right line first, then extracts the right value from it.

### 4.4 Field Configuration Options

| Option | Description |
|--------|-------------|
| `R` | **Repeatable** — the line can repeat (e.g., partial fills in a trade). Must be on the **first** field definition of the repeatable line. |
| `O` | **Optional** — the field may not be present in all documents. Scanned in a second pass after all required fields are matched. |

### 4.5 Regex Support with `(?:...)`

The non-capturing group `(?:...)` is a **powerful construct** that can be used
**anywhere on a template line** — not just adjacent to field definitions.
GT treats any word matching `(?:...)` as a regular expression instead of
literal text. This enables flexible matching of document variants.

**IMPORTANT:** Always use `(?:...)` (non-capturing). Never use plain `(...)`
— that would break GT's internal capture group numbering.

#### Use Case 1: Variant words adjacent to fields (P/N position)

When the word before or after a field varies between documents:

```
(?:Börsengeschäft:|Börsentransaktion:) {transType|P|N}
```

Here `(?:Börsengeschäft:|Börsentransaktion:)` is the `P` anchor for
`transType`. It matches either variant.

#### Use Case 2: Variant words anywhere on a line

`(?:...)` can appear at **any position** on a template line, not only next to
a field. Use it for any static word that varies between documents:

```
BETRAG, DEN WIR DEM KONTO (?:GUTSCHREIBEN|BELASTEN) {cac|P|SL} {ta|SL|N}
```

Here `(?:GUTSCHREIBEN|BELASTEN)` is in the middle of the line. For buy
transactions the document says "GUTSCHREIBEN", for sell "BELASTEN".

#### Use Case 3: Pattern matching with full regex

`(?:...)` supports the full Java regex syntax, not just simple alternatives:

```
{units|P|NL} Vanguard FTSE Japan ETF {quotation|NL|N} (?:[A-Z]{3}$)
```

Here `(?:[A-Z]{3}$)` matches any three uppercase letters at the end of the
line — typically a currency code like "USD" or "CHF". The `$` anchors it to
the line end.

#### Use Case 4: Combining with line-start alternatives `[...|...]`

For variant **first words** of a line, use the `[word1|word2]` bracket syntax
instead of `(?:...)`:

```
[Quellensteuer|Verrechnungssteuer] 35% (CH) CHF {tt1|SL|N|O}
```

This matches lines starting with either "Quellensteuer" or
"Verrechnungssteuer". The `[...|...]` syntax is specifically for line-start
alternatives used with `SL`, `PL`, and `NL` anchors.

**Summary:** Use `(?:...)` for variant words **within** a line. Use
`[...|...]` for variant **first words** of a line (SL/PL/NL anchors).

---

## 5  Configuration Reference

All configuration goes **after** the `[END]` marker, one `key=value` per line.

### 5.1 `dateFormat` (mandatory)

Java `SimpleDateFormat` pattern for parsing dates.

```
dateFormat=dd.MM.yyyy
dateFormat=yyyy-MM-dd
dateFormat=dd MMM yyyy
```

### 5.2 `timeFormat` (optional)

Java `SimpleDateFormat` pattern for time, used with the `time` field.

```
timeFormat=HH:mm:ss
```

### 5.3 `transType` (mandatory, one or more lines)

Maps document keywords to GT transaction types. Format:
`transType=GT_TYPE|word1,word2,...`

Valid GT types for PDF templates:
- `ACCUMULATE` — Buy
- `REDUCE` — Sell / Redemption
- `DIVIDEND` — Dividend / Interest payment

```
transType=ACCUMULATE|Kauf,Purchase,Buy
transType=REDUCE|Verkauf,Sale,Sell,Redemption
transType=DIVIDEND|Dividende,Dividend
```

Multiple words can map to the same type (comma-separated). Each word is matched
exactly against the extracted `transType` field value.

### 5.4 `overRuleSeparators` (recommended)

Overrides locale-dependent thousand/decimal separators. Format:
`overRuleSeparators=LOCALE<thousandSep|decimalSep>`

Use `All` to apply regardless of user locale:

```
overRuleSeparators=All<''|.>
```

This means: apostrophe (`'`) as thousand separator, period (`.`) as decimal
separator. The thousand separator can be empty (no thousand separator):

```
overRuleSeparators=All<|,>
```

Locale-specific overrides are also possible:

```
overRuleSeparators=de-CH<'|.>de-DE<.|,>
```

### 5.5 `otherFlagOptions` (optional)

Pipe-separated feature flags. Available flags (without the `CAN_` prefix):

| Flag | Description |
|------|-------------|
| `BOND_QUOTATION_CORRECTION` | Adjusts bond interest rates for payment frequency. |
| `NO_TAX_ON_DIVIDEND_INTEREST` | Marks dividends/interest as tax-exempt. |
| `BASE_CURRENCY_MAYBE_INVERSE` | Handles reverse currency pair scenarios. |
| `CASH_SECURITY_CURRENCY_MISMATCH_BUT_EXCHANGE_RATE` | Auto-calculates missing exchange rates when dividend currency differs from security currency. |
| `BOND_ADJUST_UNITS_AND_QUOTATION_WHEN_UNITS_EQUAL_ONE` | Adjusts bond units when document shows 1 instead of face value. |

```
otherFlagOptions=BOND_QUOTATION_CORRECTION|BASE_CURRENCY_MAYBE_INVERSE
```

### 5.6 `ignoreTaxOnDivInt` (optional)

Transaction type keyword for which tax should be ignored (e.g., capital gains
treated as tax-free):

```
ignoreTaxOnDivInt=Kapitalgewinn
```

### 5.7 `templatePurpose` (recommended)

Free-text description of what this template handles:

```
templatePurpose=Buy and sell securities (with repeatable units)
```

---

## 6  Step-by-Step Workflow

When given PDF-to-text input, follow these steps:

### Step 1: Analyze Input
- Identify the broker/trading platform from header/footer text.
- Identify the language (German, English, French, etc.).
- Identify the transaction type (buy, sell, dividend, redemption, etc.).

### Step 2: Group PDFs by Structure
- Same broker + same layout + different transaction types → **one template**
- Same broker + different layout → separate templates
- Different brokers → always separate templates

### Step 3: Identify the Readable Area
- Exclude personal data lines (name, address) at the top.
- Exclude legal disclaimers, contact info, footer at the bottom.
- Keep all lines containing transaction-relevant data.

### Step 4: Locate Extractable Values
For each document, identify:
- Transaction date → `datetime`
- Transaction type keyword → `transType`
- Security identifier (ISIN preferred) → `isin` or `symbol`
- Number of units → `units`
- Price per unit → `quotation`
- Currency of cash account → `cac`
- Total amount → `ta`
- Any costs → `tc1`, `tc2`
- Any taxes → `tt1`, `tt2`
- Exchange rate → `cex`
- Accrued interest → `ac`
- Other fields as applicable.

### Step 5: Determine Number Format
Look at how numbers are formatted in the document:
- `8'236.75` → apostrophe thousands, period decimal → `overRuleSeparators=All<''|.>`
- `8.236,75` → period thousands, comma decimal → `overRuleSeparators=All<.|,>`
- `8,236.75` → comma thousands, period decimal → `overRuleSeparators=All<,|.>`
- `8 236,75` → space thousands, comma decimal → `overRuleSeparators=All< |,>`

### Step 6: Determine Date Format
Match the date pattern to Java SimpleDateFormat:
- `13.05.2019` → `dd.MM.yyyy`
- `2019-05-13` → `yyyy-MM-dd`
- `13 May 2019` → `dd MMM yyyy`
- `05/13/2019` → `MM/dd/yyyy`

### Step 7: Build the Template Body

**First, copy the PDF-to-text output and preserve every line break exactly.**
Then, for each extractable value:
1. Replace the value with `{fieldName|anchors}` **on the same line where it
   appears in the PDF text**. Never move values to a different line.
2. Choose anchors strategically — **always use at least two anchors**:
   - **Always add a line-reference anchor** (`SL`, `PL`, or `NL`) to constrain
     which line GT considers. Without it, the regex is tried against every
     line in the document and may match the wrong one.
   - **Combine with `P` and/or `N`** for the value extraction within that line.
   - Common safe patterns: `P|SL`, `P|N`, `P|PL`, `SL|N`, `PL|P|N`.
   - Use **`PL`** when the value's own line has no reliable static start, but
     the previous line does.
   - Use **`NL`** similarly for the next line.
   - Add **`O`** for optional fields (costs, taxes that may not appear).
   - Add **`R`** on the first field of repeatable table rows.
3. Keep all static text that is the same across documents. Use `(?:...)`
   for any static word that varies between document variants.
4. Anonymize personal data (names, account numbers) but keep the line
   structure.
5. **Verify line-by-line**: compare your template against the PDF-to-text
   input. Each template line must correspond to exactly one input line.

### Step 8: Generalize for Multiple Transaction Types
When combining buy and sell (or other variants) into one template:
- Use `(?:word1|word2)` for words that vary at `P`/`N` positions.
- Use `[word1|word2]` for variant line starts at `SL`/`PL`/`NL` positions.
- Add multiple `transType` lines in the configuration.
- Lines that only appear in one variant can often be handled with `O`
  (optional).

### Step 9: Build the Configuration Section
After `[END]`, add:
1. `templatePurpose=...` (describe what the template handles)
2. `transType=...` lines (one per transaction type mapping)
3. `dateFormat=...`
4. `overRuleSeparators=...` (if non-default number format)
5. Any other flags as needed.

### Step 10: Self-Validate
Run through the validation checklist in Section 8.

---

## 7  Worked Examples

### Example 1: Equity Buy/Sell — Swissquote (German)

#### Input: Buy document (PDF-to-text)

```
Gland, 13.05.2019
Börsentransaktion: Kauf Unsere Referenz: 12345678
Gemäss Ihrem Kaufauftrag vom 13.05.2019 haben wir folgende Transaktionen vorgenommen:
Titel Ort der Ausführung
FISCHER N ISIN: CH0001752309 SIX Swiss Exchange
NKN: 175230
Anzahl Preis Betrag
3 904.5 CHF 2'713.5
Total CHF 2'713.5
Kommission Swissquote Bank AG CHF 30.85
Abgabe (Eidg. Stempelsteuer) CHF 2.05
Börsengebühren CHF 1.00
Zu Ihren Lasten CHF 2'747.40
Betrag belastet auf Kontonummer  99999900, Valutadatum 15.05.2019
```

#### Input: Sell document (PDF-to-text)

```
Gland, 05.02.2018
Börsentransaktion: Verkauf Unsere Referenz: 12345678
Gemäss Ihrem Verkaufsauftrag vom 05.02.2018 haben wir folgende Transaktionen vorgenommen:
Titel Ort der Ausführung
IDORSIA N ISIN: CH0363463438 SIX Swiss Exchange
NKN: 36346343
Anzahl Preis Betrag
322 25.58 CHF 8'236.75
Total CHF 8'236.75
Kommission Swissquote Bank AG CHF 30.85
Abgabe (Eidg. Stempelsteuer) CHF 6.20
Börsengebühren CHF 1.00
Zu Ihren Gunsten CHF 8'198.70
Betrag gutgeschrieben auf Ihrer Kontonummer  99999900, Valutadatum 07.02.2018
```

#### Analysis

- **Broker:** Swissquote (identified by "Swissquote Bank AG", "Gland" address)
- **Language:** German
- **Layout:** Both documents share the same structure
- **Differences:** "Kauf" vs "Verkauf", "Kaufauftrag" vs "Verkaufsauftrag",
  "Zu Ihren Lasten" vs "Zu Ihren Gunsten" — but the template lines containing
  field definitions are structurally identical.
- **One template** can handle both because the transaction type word appears
  on the same line with the same anchors.
- **Number format:** `2'713.5` → apostrophe thousand separator, period decimal
- **Date format:** `13.05.2019` → `dd.MM.yyyy`
- **Repeatable row:** The "Anzahl Preis Betrag" row may repeat (partial fills).
- **Note:** "Börsentransaktion:" appears in both docs. Older documents use
  "Börsengeschäft:" — use `(?:...)` regex to handle both.

#### Output: Combined Template

```
Gland, {datetime|P|N}
(?:Börsengeschäft:|Börsentransaktion:) {transType|P|N} Unsere Referenz: 12345678
Gemäss Ihrem Kaufauftrag vom 28.01.2013 haben wir folgende Transaktionen vorgenommen:
Titel Ort der Ausführung
ISHARES $ CORP BND ISIN: {isin|P} SIX Swiss Exchange
NKN: 1613957
Anzahl Preis Betrag
{units|PL|R} {quotation} {cac} 8'000.00
Total USD 8'250.00
Kommission Swissquote Bank AG USD {tc1|SL|N|O}
[Abgabe (Eidg. Stempelsteuer)|Eidgenössische Stempelsteuer] USD {tt1|SL|N}
Börsengebühren USD {tc2|SL|N}
Zu Ihren Lasten USD {ta|SL|N}
[END]
templatePurpose=Kauf und Verkauf Wertpapier (Wiederholung units)
transType=ACCUMULATE|Kauf
transType=REDUCE|Verkauf
dateFormat=dd.MM.yyyy
overRuleSeparators=All<''|.>
```

#### Explanation of Key Decisions

- **Line 1:** `{datetime|P|N}` — date is after "Gland," (P) and at end of
  line (N = `$`).
- **Line 2:** `(?:Börsengeschäft:|Börsentransaktion:)` — regex non-capture
  group handles two document variants. `{transType|P|N}` extracts "Kauf" or
  "Verkauf".
- **Line 3:** Static text kept as-is. The specific words ("Kaufauftrag",
  the date) don't matter because this line has no field definitions. GT only
  cares about lines WITH field definitions.
- **Line 5:** `{isin|P}` — ISIN follows "ISIN:" which is the P anchor.
  Only one anchor is needed because "ISIN:" is globally unique.
- **Line 8:** `{units|PL|R} {quotation} {cac}` — `PL` references "Anzahl"
  from the previous line. `R` on the first field marks this as a repeatable
  row. `{quotation}` and `{cac}` are on the same repeatable line and inherit
  the table row pattern.
- **Line 10:** `{tc1|SL|N|O}` — `SL` = "Kommission" (first word), `N` = end
  of line, `O` = optional (some documents may not have this fee).
- **Line 11:** `[Abgabe...|Eidgenössische...]` — bracket syntax for variant
  line starts.

---

### Example 2: Dividend — Postfinance (German)

#### Input: Dividend document (PDF-to-text)

```
TRANSAKTIONSBELEG
Kunde: 999999 - TRADING
Herrn Vorname Nachname
Strasse Hausnummer
CH-2540 Grenchen
Bern, 06.09.2017
Dividende
Unsere Referenz: 12345678
Im Hinblick auf folgenden Titel:
Titel
Bezeichnung Anzahl
ISIN: CH0032912732
UBS ETF CH - SLI CHF A 34
NKN: 3291273
haben wir Ihrem Konto den folgenden Betrag gutgeschrieben:
Kontonummer 99999900
Ausführungsdatum 06.09.2017
Valutadatum 08.09.2017
Anzahl 34
Dividende 1.66 CHF
Betrag CHF 56.44
Verrechnungssteuer 35% (CH) CHF 19.75
Total CHF 36.69
```

#### Output: Dividend Template

```
Bern, 25.04.2018
{transType|PL|P|N}
Unsere Referenz: 145360471
Im Hinblick auf folgenden Titel:
Titel
Bezeichnung Anzahl
ISIN: {isin|P|N}
iShares Global HY Corp Bnd CHF 350
NKN: 22134231
haben wir Ihrem Konto den folgenden Betrag gutgeschrieben:
Kontonummer 12345678
Ausführungsdatum {datetime|P|N}
Valutadatum 25.04.2018
Anzahl {units|P|N}
Dividende {quotation|P|SL}  CHF
Betrag CHF 1'000.31
[Quellensteuer|Verrechnungssteuer] 35% (CH) CHF {tt1|SL|N|O}
Total {cac|P|SL} {ta|SL|N}
[END]
templatePurpose=Dividende für Aktien/ETF
transType=DIVIDEND|Dividende
dateFormat=dd.MM.yyyy
overRuleSeparators=All<''|.>
```

#### Explanation of Key Decisions

- **Header lines** (TRANSAKTIONSBELEG, Kunde, address) are excluded — they're
  outside the readable area.
- **Line 2:** `{transType|PL|P|N}` — "Dividende" is on its own line. `PL`
  references "Bern," from the previous line, `P` matches beginning of line,
  `N` matches end of line.
- **Line 7:** `{isin|P|N}` — ISIN after "ISIN:" with next word anchor for
  additional context.
- **Line 17:** `[Quellensteuer|Verrechnungssteuer]` — different tax names
  for the same withholding tax. `{tt1|SL|N|O}` — optional because some
  dividends may not have withholding tax.
- **Line 18:** `{cac|P|SL}` — currency after "Total" (P), verified by line
  start (SL = "Total"). `{ta|SL|N}` — total amount, also anchored by line
  start and end of line.

---

### Example 3: Bond Buy/Sell — Postfinance (German)

#### Output Template (demonstrating `per` field and `ac` field)

```
Bern, {datetime|P|N}
Börsentransaktion: {transType|P|N}
Unsere Referenz: 142494149
Gemäss Ihrem Kaufauftrag vom 08.03.2018 haben wir folgende Transaktionen vorgenommen:
Titel Ort der Ausführung
2.20 BALIFE 17-48 ISIN: {isin|P} SIX Swiss Exchange
NKN: 37961100
Anzahl Preis Betrag
{units|PL|R} {quotation} {per|O} {cin} 4'741.56
CHF
Marchzinsen {ac|SL|N|O}
Total CHF 5'068.60
Kommission CHF {tc1|SL|N|O}
Abgabe (Eidg. Stempelsteuer) CHF {tt1|SL|N}
Börsengebühren CHF {tc2|SL|N|O}
Benutzter Trading Credit {reduce|SL|N|O}
Zu Ihren Lasten {cac|SL} {ta|SL|N}
[END]
templatePurpose=Kauf und Verkauf Wertpapier
transType=ACCUMULATE|Kauf
transType=REDUCE|Verkauf
dateFormat=dd.MM.yyyy
overRuleSeparators=All<''|.>
```

#### Key Features Demonstrated

- **`{per|O}`** — The percentage indicator field. When present (bonds quoted
  as percentage of face value), GT adjusts the quotation calculation. It's
  optional because equity trades don't have it.
- **`{ac|SL|N|O}`** — Accrued interest ("Marchzinsen"), optional because
  it only appears on bond transactions.
- **`{cin}`** — Currency of the instrument, extracted from the repeatable row.
- **`{reduce|SL|N|O}`** — Trading credit discount, optional.

---

## 8  Validation Checklist

Before outputting a template, verify:

- [ ] **`[END]` marker** is present, separating body from configuration.
- [ ] **`dateFormat`** is present and matches the date pattern in the document.
- [ ] **At least one `transType`** mapping exists.
- [ ] **Mandatory fields** are present: `datetime` (or `date`+`time`),
      `transType`, `cac`, `ta`.
- [ ] For security transactions: at least `isin` or `symbol` is present,
      plus `units` and `quotation`.
- [ ] **Each field has at least two anchors** — ideally a line-reference
      anchor (`SL`, `PL`, or `NL`) combined with a line-internal anchor
      (`P` or `N`). A single anchor risks matching the wrong line.
- [ ] **`R` is on the first field** of any repeatable line.
- [ ] **Regex groups use `(?:...)`** non-capturing syntax (not `(...)`).
- [ ] **`[word1|word2]` syntax** is used for variant line starts, not
      `(?:...)`.
- [ ] **Personal data is anonymized** — names, addresses, real account
      numbers are replaced with placeholders, but line structure preserved.
- [ ] **Number format** is correctly identified and `overRuleSeparators`
      is set if non-default.
- [ ] **Optional fields** (`tc1`, `tc2`, `tt1`, `tt2`, `ac`, `reduce`,
      `per`) have the `O` flag when they may not appear in all documents.
- [ ] **No lines from outside the readable area** (headers, footers, legal
      text) are included in the template body, unless they contain field
      definitions.

---

## 9  Multi-PDF Handling

When given multiple PDF-to-text conversions:

### Same broker + same layout + different transaction types
→ **One template** with multiple `transType` lines.

Compare the documents line by line. Where they differ:
- In field value positions: already handled by field definitions.
- In static words: use `(?:word1|word2)` for P/N anchors, or
  `[word1|word2]` for line starts.
- Lines present in only one variant: use `O` if they contain fields,
  or remove them if they're purely static.

### Same broker + different layout
→ **Separate templates**. Different layouts mean different line structures
that can't be generalized into one template.

### Different brokers
→ **Always separate templates**. Each broker has its own document design.

### Output format
Label each template clearly:

```
### Template 1: [Broker] — [Purpose]

\`\`\`
<template content>
\`\`\`

### Template 2: [Broker] — [Purpose]

\`\`\`
<template content>
\`\`\`
```

---

## Appendix A: Complete Anchor Syntax Quick Reference

```
{fieldName|P}          — previous word is anchor
{fieldName|N}          — next word is anchor
{fieldName|P|N}        — both previous and next word
{fieldName|Pc}         — value has prefix (no space)
{fieldName|Nc}         — value has suffix (no space)
{fieldName|SL}         — first word of same line
{fieldName|PL}         — first word of previous line
{fieldName|NL}         — first word of next line
{fieldName|SL|N}       — same line start + next word
{fieldName|PL|P|N}     — prev line start + prev word + next word
{fieldName|SL|N|O}     — same line start + next word + optional
{fieldName|PL|R}       — prev line start + repeatable (first field)
```

## Appendix B: Two-Phase Parsing

GT processes templates in two phases:

1. **Phase A (Required):** All non-optional fields are matched sequentially,
   line by line. The parser advances through the document looking for matches
   in order. Once all required fields are matched, the form is considered
   "matched".

2. **Phase B (Optional):** After Phase A succeeds, optional fields (marked
   with `O`) are scanned across a range of lines between surrounding required
   fields. This means optional fields don't need to be at a fixed position —
   they can appear anywhere within their expected range.

This is why optional fields should always have the `O` flag — it tells GT
to defer their matching to Phase B, preventing them from blocking the
sequential matching of required fields in Phase A.
