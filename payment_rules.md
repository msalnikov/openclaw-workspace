# Utility Bills Parsing Rules (Discord channel: #платежки)

This file stores the agreed parsing/output rules for recurring bill processing.

## Supported bill types and output fields

### 1) Київводоканал
- Organization label in output: **Водоканал**
- Output fields:
  - Organization
  - Bill date (billing month/date from document)
  - **Загальна сума до сплати**
- Validation rule:
  - Recalculate by summing the **Нарахов.** column.
  - If mismatch or unreadable required fields: return an error, do not guess.

### 2) Київтеплоенерго
- Organization label in output: **Теплоенерго**
- Output fields:
  - Organization
  - Bill date
  - **До сплати** (sum of all rows in the **До сплати** column)
- Fallback rule:
  - If **До сплати** values are unreadable, use **Нараховано за ...** values (they should match).
- While returning this bill separately, show addends used to build total.

### 3) Київські енергетичні послуги
- Organization label in output: **Електроенергія**
- Output fields:
  - Organization
  - Bill date
  - **До сплати** (from **До сплати за фактичне споживання**)
- Validation rule:
  - Cross-check with line **Нараховано у <MONTH>**.

### 4) ОСББ "Ломоносова 83а"
- Organization label in output: **ОСББ**
- Output fields:
  - Organization
  - Bill date
  - **До сплати** (value from row **РАЗОМ**, column **До сплати**)
- Validation rule:
  - Cross-check with row **РАЗОМ** in column **Нараховано у <MONTH> <YEAR>**.

## Unknown bill formats
- If a bill does not match trained/supported templates, explicitly report that it is an unknown/untrained type.
- Do not fabricate values.

## Combined summary format (when 4 bills are sent together)
- First line: billing month/year only (no prefix like "Месяц квитанций").
  - If months differ, show month per organization.
- Then show per-organization amounts:
  - Водоканал: <number>
  - Електроенергія: <number>
  - Теплоенерго: <number>
  - ОСББ: <number>
- Do **not** append "грн" to organization lines.
- Final line:
  - **Всього: <number> грн**

## Style constraints from user
- For successful checks, do not mention validation steps.
- Mention validation/consistency issues only when there is an error or mismatch.
