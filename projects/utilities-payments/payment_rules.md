# Utilities Payments — Full Processing Specification

This document is the single source of truth for migrating the utility-bills workflow to another machine without quality loss.

## 1) Project scope

- Project name: **Utilities Payments**
- Channel scope: Discord channel **#платежки**
- Goal: parse utility bills from images and return stable, compact payment summaries.
- Input mode:
  - Single bill image (per-provider extraction)
  - Batch of 4 bill images (monthly combined summary)

## 2) Storage and files

- Project root: `projects/utilities-payments/`
- Rules file: `projects/utilities-payments/payment_rules.md`
- Project README: `projects/utilities-payments/README.md`
- Bill image archive for this workflow: `projects/utilities-payments/media/`
- Git ignore for media: `projects/utilities-payments/.gitignore` contains `media/`

### Migration requirement
During migration, preserve:
- `projects/utilities-payments/payment_rules.md`
- `projects/utilities-payments/README.md`
- `projects/utilities-payments/.gitignore`
- (optional, if historical dataset is needed) `projects/utilities-payments/media/`

## 3) Supported bill templates

Exactly 4 supported organizations/templates.

### A) Київводоканал
- Output organization label: **Водоканал**
- Required extracted fields:
  - Organization
  - Bill date (month/date from the bill)
  - `Загальна сума до сплати`
- Validation:
  - Recalculate from column `Нарахов.` sum.
  - If required values are unreadable or mismatch occurs: return explicit error, no guessing.

### B) Київтеплоенерго
- Output organization label: **Теплоенерго**
- Required extracted fields:
  - Organization
  - Bill date
  - `До сплати` = sum of all rows in column `До сплати`
- Fallback:
  - If `До сплати` row values are hard to read, use `Нараховано за ...` row values.
- Mandatory display rule:
  - Always show addends used for the final `До сплати` total.

### C) Київські енергетичні послуги
- Output organization label: **Електроенергія**
- Required extracted fields:
  - Organization
  - Bill date
  - `До сплати` = value from `До сплати за фактичне споживання`
- Validation:
  - Cross-check against `Нараховано у <MONTH>`.

### D) ОСББ "Ломоносова 83а"
- Output organization label: **ОСББ**
- Required extracted fields:
  - Organization
  - Bill date
  - `До сплати` = row `РАЗОМ`, column `До сплати`
- Validation:
  - Cross-check against row `РАЗОМ` in `Нараховано у <MONTH> <YEAR>`.

## 4) Unknown templates / low confidence

If a received bill does not match one of the 4 trained templates, or required values cannot be reliably read:
- explicitly report: unknown/untrained bill type or extraction error
- do not invent, estimate, or autofill values

## 5) Output contract

## 5.1 Single-bill output
Use provider-specific fields:

- Водоканал:
  - `Организация: Водоканал`
  - `Дата квитанции: ...`
  - `Загальна сума до сплати: ...`

- Електроенергія:
  - `Организация: Електроенергія`
  - `Дата квитанции: ...`
  - `До сплати: ...`

- Теплоенерго:
  - `Организация: Теплоенерго`
  - `Дата квитанции: ...`
  - `До сплати: ... (a + b + c + ...)`

- ОСББ:
  - `Организация: ОСББ`
  - `Дата квитанции: ...`
  - `До сплати: ...`

## 5.2 Combined output (all 4 bills)
Expected structure:

1. First line: month/year only (no prefix text)
   - If all bills share same month → one common line (e.g., `Січень 2026`)
   - If months differ → show month per organization
2. Organization lines:
   - `Водоканал: <number>`
   - `Електроенергія: <number>`
   - `Теплоенерго: <number> (<addends>)`
   - `ОСББ: <number>`
3. Final line:
   - `Всього: <number> грн`

Formatting constraints:
- Do not append `грн` on organization lines
- Keep `грн` only in `Всього`
- For `Теплоенерго`, addends are mandatory in both single and combined outputs

## 6) Validation visibility rules

- Validation is always performed internally.
- On successful validation, do not print validation details.
- If mismatch/error occurs, explicitly report the issue.

## 7) Operational behavior in this channel

- Treat this channel as recurring monthly processing context.
- Store incoming bill images in `projects/utilities-payments/media/` for this project workflow.
- Maintain backward compatibility with this output contract unless the user updates rules.

## 8) Quality checklist (for new environment acceptance)

A migrated setup is considered correct only if it can:
1. Correctly classify all 4 bill templates.
2. Extract required values for each provider.
3. Apply provider-specific validation/fallback rules.
4. Produce exact combined format (`month line`, 4 org lines, `Всього ... грн`).
5. Include addends for `Теплоенерго`.
6. Refuse unknown templates without hallucinating values.
7. Preserve numeric formatting with comma decimals as in source style.

## 9) Diagnostics and Quality Verification (mandatory after migration)

Purpose: quickly confirm that processing quality in a new environment is identical to the previous one.

### 9.1 Smoke test sequence
Run tests in this order:
1. One bill per provider (4 single-bill tests)
2. One combined batch test (all 4 bills in one message)
3. One negative test (unknown template)
4. One low-quality image test (tilt/blur/noise)

### 9.2 Required assertions per provider
For each single-bill test, verify:
- Correct provider classification label:
  - Водоканал / Електроенергія / Теплоенерго / ОСББ
- Correct date extraction
- Correct payment field extraction per provider rules
- Validation logic applied silently unless mismatch detected
- No invented values when source is unreadable

### 9.3 Required assertions for combined mode
For a 4-bill batch, verify output strictly matches contract:
- Line 1: month/year only (or per-org months if different)
- Organization lines present in stable order:
  1) Водоканал
  2) Електроенергія
  3) Теплоенерго
  4) ОСББ
- Organization lines contain numbers only (without `грн`)
- `Теплоенерго` includes addends in parentheses
- Final line format: `Всього: <number> грн`
- Arithmetic consistency:
  - `Всього` equals sum of all four organization values

### 9.4 Numeric validation policy
- Decimal separator style should remain comma-based for user-facing values.
- Any internal recalculation must match extracted totals exactly.
- If mismatch occurs, return explicit error instead of corrected guess.

### 9.5 Error handling expectations
System must explicitly report errors for:
- Unknown/untrained template
- Missing required field
- Unreadable critical numeric cell
- Validation mismatch between control and target fields

No hallucinated fallback values are allowed.

### 9.6 Regression baseline (recommended)
Keep a small fixed test pack in project media archive:
- 4 known-good bills (one per provider)
- 1 known mixed-month batch
- 1 intentionally unknown template
- 1 low-quality but readable bill

After migration, run baseline and compare outputs with expected golden results.
Any difference in fields, formatting, or totals is a migration regression.
