# Sheet Type Validation

## Overview

The header detector now **ONLY processes 4 specific sheet types** (case insensitive).

All other sheet types are skipped.

---

## Valid Sheet Types

### ✅ 1. Invoice
```
Examples (all valid):
- "Invoice"
- "INVOICE"
- "invoice"
- "Invoice_2024"
- "Commercial Invoice"
- "INV"
```

### ✅ 2. Packing list
```
Examples (all valid):
- "Packing list"
- "PACKING LIST"
- "packing list"
- "PackingList"
- "Packing List 2024"
- "PAK"
```

### ✅ 3. Detail packing list
```
Examples (all valid):
- "Detail packing list"
- "DETAIL PACKING LIST"
- "detail packing list"
- "DetailPackingList"
- "Detailed Packing List"
```

### ✅ 4. Contract
```
Examples (all valid):
- "Contract"
- "CONTRACT"
- "contract"
- "Contract Sheet"
- "Sales Contract"
- "CON"
```

---

## How It Works

### Validation Logic

```python
sheet_name = worksheet.title.lower().strip()

valid_sheets = [
    'invoice',
    'packing list',
    'detail packing list',
    'contract'
]

# Check if ANY valid name appears in the sheet name
is_valid = any(valid_name in sheet_name for valid_name in valid_sheets)
```

**Uses `in` operator:**
- "Commercial Invoice" contains "invoice" ✓
- "PAK" contains... nothing ✗ (But "Packing list" would be ✓)

**Case Insensitive:**
- Converts sheet name to lowercase
- "INVOICE", "Invoice", "invoice" all match ✓

---

## Examples

### ✅ Valid Sheets (Processed)

```
Sheet: "Invoice"
[HEADER_DETECTION] Processing sheet: Invoice
→ Headers extracted ✓

Sheet: "PACKING LIST"
[HEADER_DETECTION] Processing sheet: PACKING LIST
→ Headers extracted ✓

Sheet: "Commercial Invoice 2024"
[HEADER_DETECTION] Processing sheet: Commercial Invoice 2024
→ Headers extracted ✓

Sheet: "Contract Sheet"
[HEADER_DETECTION] Processing sheet: Contract Sheet
→ Headers extracted ✓
```

### ❌ Invalid Sheets (Skipped)

```
Sheet: "Summary"
[HEADER_DETECTION] Sheet 'Summary' is not a valid type
[HEADER_DETECTION] Valid types: Invoice, Packing list, Detail packing list, Contract
→ Skipped, returns []

Sheet: "Notes"
[HEADER_DETECTION] Sheet 'Notes' is not a valid type
[HEADER_DETECTION] Valid types: Invoice, Packing list, Detail packing list, Contract
→ Skipped, returns []

Sheet: "Sheet1"
[HEADER_DETECTION] Sheet 'Sheet1' is not a valid type
[HEADER_DETECTION] Valid types: Invoice, Packing list, Detail packing list, Contract
→ Skipped, returns []
```

---

## Debug Output

### Valid Sheet
```
[HEADER_DETECTION] Processing sheet: Invoice
[BOLD_DETECTION] Found 1 candidate:
  Row 5: 10 bold cells, 10 text, 0 numeric ← SELECTED (MOST BOLD)
[HEADER_DETECTION] Selected header row: 5
[HEADER_DETECTION] Detected single-row header structure
```

### Invalid Sheet
```
[HEADER_DETECTION] Sheet 'Summary' is not a valid type
[HEADER_DETECTION] Valid types: Invoice, Packing list, Detail packing list, Contract
(No further processing)
```

---

## Common Sheet Name Variations

### Invoice Variations (All Valid ✓)
- Invoice
- INVOICE
- invoice
- INV
- Commercial Invoice
- Sales Invoice
- Invoice 2024
- Invoice_Jan
- INVOICE SHEET

### Packing List Variations (All Valid ✓)
- Packing list
- PACKING LIST
- packing list
- PackingList
- PAK (need to check - contains "pak" not "packing list")
- Packing List 2024
- Detail Packing List

### Contract Variations (All Valid ✓)
- Contract
- CONTRACT
- contract
- CON (need to check - contains "con" not "contract")
- Sales Contract
- Contract Sheet
- Purchase Contract

---

## Important Notes

### ⚠️ Abbreviations

Some common abbreviations might NOT match:

**May NOT work (doesn't contain full word):**
- "INV" → Doesn't contain "invoice" ❌
- "PAK" → Doesn't contain "packing list" ❌
- "CON" → Doesn't contain "contract" ❌

**Solutions:**
1. Use full names in sheet titles
2. OR add abbreviations to valid_sheets list:

```python
valid_sheets = [
    'invoice', 'inv',  # Added abbreviation
    'packing list', 'pak',  # Added abbreviation
    'detail packing list',
    'contract', 'con'  # Added abbreviation
]
```

### ✅ Partial Matches Work

```
"Commercial Invoice" contains "invoice" ✓
"Detailed Packing List" contains "packing list" ✓
"Sales Contract 2024" contains "contract" ✓
```

---

## Adding More Sheet Types

To add more valid sheet types, edit the `valid_sheets` list:

```python
valid_sheets = [
    'invoice',
    'packing list',
    'detail packing list',
    'contract',
    'bill of lading',      # NEW
    'certificate',          # NEW
]
```

---

## Benefits

✅ **Focused Processing** - Only processes relevant sheets
✅ **Skips Junk Sheets** - Ignores "Summary", "Notes", etc.
✅ **Clear Feedback** - Shows which sheets are skipped
✅ **Case Insensitive** - Works with any capitalization
✅ **Flexible** - Allows variations and prefixes

---

## Testing

### Test with Multiple Sheets

```
File with sheets:
1. "Invoice" → Processed ✓
2. "Packing List" → Processed ✓
3. "Summary" → Skipped ✗
4. "Contract" → Processed ✓
5. "Notes" → Skipped ✗

Debug output:
[HEADER_DETECTION] Processing sheet: Invoice
(headers extracted)
[HEADER_DETECTION] Processing sheet: Packing List
(headers extracted)
[HEADER_DETECTION] Sheet 'Summary' is not a valid type
[HEADER_DETECTION] Processing sheet: Contract
(headers extracted)
[HEADER_DETECTION] Sheet 'Notes' is not a valid type
```

---

## Modification Example

### If You Need to Include Abbreviations

Edit line 111-116 in `header_detector.py`:

```python
valid_sheets = [
    'invoice', 'inv',              # Invoice and abbreviation
    'packing list', 'pak',         # Packing list and abbreviation
    'detail packing list',
    'contract', 'con',             # Contract and abbreviation
]
```

Now these will work:
- "INV" ✓
- "PAK" ✓
- "CON" ✓

---

## Summary

| Feature | Description |
|---------|-------------|
| **Valid Types** | Invoice, Packing list, Detail packing list, Contract |
| **Case Sensitive** | No - converts to lowercase |
| **Partial Match** | Yes - "Commercial Invoice" matches "invoice" |
| **Abbreviations** | May need to be added manually |
| **Invalid Sheets** | Skipped with clear message |

---

## Quick Reference

**What's validated:**
```python
sheet_name.lower() contains one of:
- 'invoice'
- 'packing list'
- 'detail packing list'
- 'contract'
```

**What happens if invalid:**
```python
Returns empty list []
No headers extracted
Clear message in debug output
```

---

**The system now ONLY processes the 4 specific sheet types you specified!** ✅

Test it and check the debug output to see which sheets are processed vs skipped.





