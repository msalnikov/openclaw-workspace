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
