# Utilities Payments

Project for recurring utility bill parsing in Discord channel `#платежки`.

## Purpose
Process 4 monthly bill types and return a compact payment summary:
- Водоканал
- Електроенергія
- Теплоенерго
- ОСББ

## Storage
- Bill images for this channel are stored in:
  - `projects/utilities-payments/media/`
- Parsing/output rules are stored in:
  - `projects/utilities-payments/payment_rules.md`

## Output format (combined 4 bills)
- First line: month/year only (or per-org month if different)
- Per-organization amounts (without `грн`)
- `Всього: <number> грн`
- For `Теплоенерго`, include addends used to build the total.

## Notes
- Unknown bill templates must be reported as unknown (no guessed values).
- Validation checks are performed silently unless mismatch/error is found.
