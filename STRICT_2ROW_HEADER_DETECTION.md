# STRICT 2-Row Header Detection

## Problem

System was incorrectly detecting 2-row headers when it should only be single-row.

**Example of false positive:**
```
Row 5: P.O│ITEM│Description│Quantity│Amount (HEADER)
Row 6: 12345│A001│Wood Panel│100│$1,000 (DATA)
       ↑ System incorrectly thought this was a 2nd header row! ❌
```

---

## Solution: 6 Validation Checks

**Now requires ALL 6 checks to pass before accepting as 2-row header:**

### ✅ CHECK 1: "Quantity" Keyword
```
Must find "Quantity" or "Qty" in header row
```

### ✅ CHECK 2: Next Row Exists
```
Must have a row after the header row
```

### ✅ CHECK 3: More Text Than Numbers
```
Next row must have: text_cells > numeric_cells
```

### ✅ CHECK 4: Bold Formatting
```
Next row must have: bold_cells >= 2
(Sub-headers are formatted bold like main headers)
```

### ✅ CHECK 5: Short Text Only
```
Next row text must be SHORT (≤5 chars)
Examples: "PCS", "SF", "KG", "LBS"
NOT: "Wood Panel", "Description", "Customer Name"
```

### ✅ CHECK 6: Not Long Descriptions
```
short_text_cells > long_text_cells
(Sub-headers are short labels, not descriptions)
```

---

## Examples

### ✅ TRUE 2-Row Header (All Checks Pass)

```
Row 5: │ P.O│ITEM│Description│𝗤𝘂𝗮𝗻𝘁𝗶𝘁𝘆│Amount│
       │ Bold, has "Quantity" ✓
       
Row 6: │    │    │           │𝗣𝗖𝗦│𝗦𝗙 │     │
       │ Bold cells: 2 ✓
       │ Text cells: 2, Numeric: 0 ✓
       │ Short text: "PCS" (3), "SF" (2) ✓
       │ Long text: 0 ✓
       
[DOUBLE_HEADER_DETECTION] ✓ Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] ✓ Confirmed 2-row header:
  Next row: 2 text (2 short, 0 long), 0 numeric, 2 bold cells

Result: 2-row header detected ✓
```

### ❌ FALSE 2-Row Header - Data Row (Fails Check 3, 4, 5)

```
Row 5: │ P.O│ITEM│Description│𝗤𝘂𝗮𝗻𝘁𝗶𝘁𝘆│Amount│
       │ Has "Quantity" ✓
       
Row 6: │12345│A001│Wood Panel │100 │$1,000│
       │ Bold cells: 0 ✗ (Fails CHECK 4)
       │ Text: 2, Numeric: 3 ✗ (Fails CHECK 3)
       │ Long text: "Wood Panel" ✗ (Fails CHECK 5)
       
[DOUBLE_HEADER_DETECTION] ✓ Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] ✗ Next row has only 0 bold cells → Not a header row

Result: Single-row header only ✓
```

### ❌ FALSE 2-Row Header - Description Row (Fails Check 5)

```
Row 5: │ P.O│ITEM│Description│𝗤𝘂𝗮𝗻𝘁𝗶𝘁𝘆│Amount│
       │ Has "Quantity" ✓
       
Row 6: │    │    │𝗗𝗲𝘁𝗮𝗶𝗹𝗲𝗱 𝗗𝗲𝘀𝗰𝗿𝗶𝗽𝘁𝗶𝗼𝗻│𝗨𝗻𝗶𝘁𝘀│    │
       │ Bold cells: 2 ✓
       │ Text: 2, Numeric: 0 ✓
       │ Long text: "Detailed Description" (21 chars) ✗
       │ Short text: "Units" (5 chars)
       │ 1 long > 1 short ✗ (Fails CHECK 6)
       
[DOUBLE_HEADER_DETECTION] ✓ Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] ✗ Next row has 1 long text cells → Looks like data descriptions

Result: Single-row header only ✓
```

### ❌ FALSE 2-Row Header - No Quantity (Fails Check 1)

```
Row 5: │ P.O│ITEM│Description│Price│Amount│
       │ No "Quantity" keyword ✗
       
Row 6: │    │    │           │𝗨𝗦𝗗│𝗖𝗡𝗬│
       │ (Never checked - failed CHECK 1)
       
[DOUBLE_HEADER_DETECTION] ✗ No 'Quantity' keyword found → Single-row header

Result: Single-row header only ✓
```

---

## Debug Output

### Single-Row Header (Correct)
```
[BOLD_DETECTION] Found 1 candidate:
  Row 5: 10 bold cells, 10 text, 0 numeric ← SELECTED (MOST BOLD)
[HEADER_DETECTION] Selected header row: 5
[DOUBLE_HEADER_DETECTION] ✓ Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] ✗ Next row has 2 text, 5 numeric → Looks like data row
[HEADER_DETECTION] Detected single-row header structure
```

### 2-Row Header (Correct)
```
[BOLD_DETECTION] Found 1 candidate:
  Row 5: 10 bold cells, 10 text, 0 numeric ← SELECTED (MOST BOLD)
[HEADER_DETECTION] Selected header row: 5
[DOUBLE_HEADER_DETECTION] ✓ Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] ✓ Confirmed 2-row header:
  Next row: 2 text (2 short, 0 long), 0 numeric, 2 bold cells
[HEADER_DETECTION] Detected 2-row header structure (Quantity found)
```

---

## Validation Rules Summary

| Check | Requirement | Reason |
|-------|------------|--------|
| **1. Quantity Keyword** | "Quantity" or "Qty" in header | 2-row headers only with Quantity |
| **2. Next Row Exists** | Row after header exists | Need 2nd row to check |
| **3. More Text** | text > numeric | Sub-headers are text, not numbers |
| **4. Bold Cells** | ≥ 2 bold cells | Sub-headers formatted bold |
| **5. Short Text** | ≤ 5 characters | Sub-headers short (PCS, SF) |
| **6. Not Descriptions** | short > long text | Sub-headers not long descriptions |

---

## Short Text Examples

**Valid sub-headers (≤5 chars):**
- ✅ "PCS" (3 chars)
- ✅ "SF" (2 chars)
- ✅ "KG" (2 chars)
- ✅ "LBS" (3 chars)
- ✅ "QTY" (3 chars)
- ✅ "UNIT" (4 chars)
- ✅ "USD" (3 chars)

**Invalid (>5 chars = long text):**
- ❌ "Wood Panel" (10 chars)
- ❌ "Description" (11 chars)
- ❌ "Customer" (8 chars)
- ❌ "PIECES" (6 chars)
- ❌ "Amount USD" (10 chars)

---

## Configuration

### Adjust Short Text Length
Default: **5 characters**

To change:
```python
# Line 301 in header_detector.py
if len(cell_value) <= 5:  # Change 5 to your preference
    short_text_cells += 1
```

Recommendations:
- **Strict:** 3 chars (only PCS, SF, KG)
- **Normal:** 5 chars (current default)
- **Loose:** 7 chars (allows PIECES, etc.)

### Adjust Minimum Bold Cells
Default: **2 bold cells**

To change:
```python
# Line 312 in header_detector.py
if bold_cells < 2:  # Change 2 to your preference
```

---

## Benefits

✅ **No False Positives** - Won't confuse data rows as 2nd headers
✅ **Multiple Validations** - 6 checks ensure accuracy
✅ **Clear Feedback** - Debug shows which check failed
✅ **Conservative** - Only accepts obvious 2-row headers

---

## Common Scenarios

### Scenario A: Clear 2-Row Header
```
Row 5: P.O│ITEM│Desc│Quantity│Amount
Row 6:    │    │    │PCS│SF │
All 6 checks pass → 2-row header ✓
```

### Scenario B: Data Row After Header
```
Row 5: P.O│ITEM│Desc│Quantity│Amount
Row 6: 123│A001│Wood│100│$1000
Fails CHECK 3, 4, 5 → Single-row header ✓
```

### Scenario C: No Quantity Keyword
```
Row 5: P.O│ITEM│Description│Price│Amount
Row 6:    │    │           │USD│CNY
Fails CHECK 1 → Single-row header ✓
```

### Scenario D: Long Descriptions Below
```
Row 5: P.O│ITEM│Description│Quantity│Amount
Row 6:    │    │Product Details│Units│
Fails CHECK 6 (long text) → Single-row header ✓
```

---

## Testing

```bash
python config_template_cli/config_data_extractor/analyze_excel.py \
    your_invoice.xlsx \
    --json \
    --quantity-mode
```

**Look for:**
```
[DOUBLE_HEADER_DETECTION] ✓ or ✗ messages
```

**If wrongly detecting 2-row:**
- Check which checks passed/failed
- Adjust thresholds (bold cells, text length)

**If missing 2-row headers:**
- Ensure sub-headers are bold
- Ensure sub-headers are short (≤5 chars)
- Ensure "Quantity" keyword exists

---

## What Changed

### Before (Lenient):
```python
# Only checked:
# 1. Has "Quantity"
# 2. Next row has content
# → Too many false positives! ❌
```

### After (Strict):
```python
# Checks:
# 1. Has "Quantity" ✓
# 2. Next row exists ✓
# 3. More text than numbers ✓
# 4. Has bold cells (≥2) ✓
# 5. Text is short (≤5 chars) ✓
# 6. No long descriptions ✓
# → Very few false positives! ✅
```

---

## Summary

| Issue | Status |
|-------|--------|
| False 2-row header detection | ✅ **FIXED** |
| Data rows seen as 2nd header | ✅ **PREVENTED** |
| Description rows seen as 2nd header | ✅ **PREVENTED** |
| 6 validation checks | ✅ **ADDED** |
| Clear debug output | ✅ **IMPROVED** |

---

**The system is now VERY CONSERVATIVE about 2-row headers!**

Only accepts when ALL 6 checks pass:
1. ✓ Has "Quantity"
2. ✓ Next row exists
3. ✓ More text than numbers
4. ✓ Bold formatted
5. ✓ Short text only
6. ✓ Not long descriptions

**Test it and check the debug output to see all validation results!** 🎯





