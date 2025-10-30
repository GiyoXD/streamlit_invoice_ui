# Fixed: Data Row Pollution Issue

## Problem

The system was extracting **data rows** as headers, polluting the header extraction with actual data values.

---

## Root Cause

**Before:**
- Only checked for bold cells
- Didn't validate if the row contained HEADERS (text) or DATA (numbers)
- Could select a data row if it had many bold cells

**Example of the problem:**
```
Row 5: P.O│ITEM│Description│Quantity│Price (HEADER - bold, text) ✓
Row 6: 12345│A001│Wood Panel│100│$1,000 (DATA - bold, numbers) 
       ↑ If Row 6 was bold, it could be selected! ❌
```

---

## Solution

Added **TWO validation checks**:

### ✅ 1. Header Row Validation
**Rule:** Header row must have **more TEXT than NUMBERS**

```python
# Now checks content type
text_count = 0      # "P.O", "ITEM", "Description" (text)
numeric_count = 0   # "12345", "100", "1000" (numbers)

# Only accept if: text_count > numeric_count
if bold_count >= 3 and text_count > numeric_count:
    # This is a HEADER row ✓
```

### ✅ 2. Second Row Validation (for 2-row headers)
**Rule:** Second row must also have **more TEXT than NUMBERS**

```python
# When checking if it's a 2-row header:
# Verify next row contains SUB-HEADERS (text), not DATA (numbers)
if text_cells > numeric_cells:
    # Next row is SUB-HEADERS (PCS, SF) ✓
    return True  # 2-row header
else:
    # Next row is DATA (100, 50) ❌
    return False  # Single-row header only
```

---

## How It Works Now

### Example 1: Single-Row Header (Correct)

```
Excel File:
┌─────────────────────────────────────────────────┐
│ Row 5: │ 𝗣.𝗢│𝗜𝗧𝗘𝗠│𝗗𝗲𝘀𝗰│𝗤𝘁𝘆│𝗣𝗿𝗶𝗰𝗲│𝗔𝗺𝘁   │
│        │ 8 bold, 8 text, 0 numeric          │
│        │ ✓ More text than numbers           │
│        │ ← SELECTED as header               │
├─────────────────────────────────────────────────┤
│ Row 6: │ 12345│A001│Wood Panel│100│$10│$1K │
│        │ 6 cells, 2 text, 4 numeric         │
│        │ ✗ More numbers than text           │
│        │ ← REJECTED (data row)              │
└─────────────────────────────────────────────────┘

Result: Only Row 5 extracted ✓
```

### Example 2: 2-Row Header (Correct)

```
Excel File:
┌─────────────────────────────────────────────────┐
│ Row 5: │ 𝗣.𝗢│𝗜𝗧𝗘𝗠│𝗗𝗲𝘀𝗰│𝗤𝘂𝗮𝗻𝘁𝗶𝘁𝘆│𝗔𝗺𝘁      │
│        │ 8 bold, 8 text, 0 numeric          │
│        │ ✓ Has "Quantity" keyword           │
├─────────────────────────────────────────────────┤
│ Row 6: │    │    │    │𝗣𝗖𝗦│𝗦𝗙│           │
│        │ 2 text, 0 numeric                  │
│        │ ✓ More text than numbers           │
│        │ ← CONFIRMED as sub-header row      │
├─────────────────────────────────────────────────┤
│ Row 7: │ 12345│A001│Wood│100│50│$1000      │
│        │ ← NOT extracted (data row)         │
└─────────────────────────────────────────────────┘

Result: Rows 5 and 6 extracted (2-row header) ✓
```

### Example 3: False 2-Row Header (Rejected)

```
Excel File:
┌─────────────────────────────────────────────────┐
│ Row 5: │ 𝗣.𝗢│𝗜𝗧𝗘𝗠│𝗗𝗲𝘀𝗰│𝗤𝘂𝗮𝗻𝘁𝗶𝘁𝘆│𝗔𝗺𝘁      │
│        │ ✓ Has "Quantity" keyword           │
├─────────────────────────────────────────────────┤
│ Row 6: │ 12345│A001│Wood Panel│100│$1K    │
│        │ 2 text, 3 numeric                  │
│        │ ✗ More numbers than text           │
│        │ ← REJECTED (data row, not header)  │
└─────────────────────────────────────────────────┘

[DOUBLE_HEADER_DETECTION] Rejected 2-row header: 
    Next row looks like data (2 text, 3 numeric)

Result: Only Row 5 extracted (single-row header) ✓
```

---

## Debug Output Explanation

### New Output Format

```bash
[BOLD_DETECTION] Found 2 candidates with bold cells:
  Row 5: 8 bold cells, 8 text, 0 numeric ← SELECTED (MOST BOLD)
  Row 6: 6 bold cells, 2 text, 4 numeric (rejected - more numbers)
[HEADER_DETECTION] Selected header row: 5
[DOUBLE_HEADER_DETECTION] Found 'Quantity' at row 5, col 4
[DOUBLE_HEADER_DETECTION] Rejected 2-row header: 
    Next row looks like data (2 text, 4 numeric)
[HEADER_DETECTION] Detected single-row header structure
```

**What to look for:**
- `X text, Y numeric` - Shows content analysis
- `text > numeric` → Header row ✓
- `numeric > text` → Data row ✗

---

## Content Type Detection

### How It Classifies Cells

```python
# TEXT (header labels):
"P.O"           → text ✓
"ITEM"          → text ✓
"Description"   → text ✓
"Quantity"      → text ✓
"PCS"           → text ✓
"SF"            → text ✓
"Wood Panel"    → text ✓

# NUMERIC (data values):
"12345"         → numeric (pure number)
"100"           → numeric (pure number)
"$1,000"        → numeric (removes $ and ,)
"1.5"           → numeric (decimal)
"A001"          → text ✓ (has letters)
```

**Smart Detection:**
- Removes `$`, `,`, `.` before checking
- If remaining is all digits → numeric
- If has any letters → text

---

## Validation Rules Summary

| Check | Rule | Purpose |
|-------|------|---------|
| **Bold Count** | ≥ 3 bold cells | Minimum to be considered |
| **Text vs Numeric** | text > numeric | Headers are text labels |
| **2nd Row Check** | text > numeric | Sub-headers are text too |

---

## Benefits

✅ **No Data Pollution** - Data rows won't be extracted as headers
✅ **Accurate 2-Row Detection** - Only extracts real sub-headers
✅ **Smart Content Analysis** - Distinguishes text from numbers
✅ **Clear Debug Output** - Shows exactly why decisions were made

---

## Edge Cases Handled

### Case 1: Mixed Content Row
```
Row: "No."│"1"│"Description"│"Amount"
     text│num│text        │text
     → 3 text, 1 numeric → HEADER ✓
```

### Case 2: Mostly Numbers
```
Row: "1"│"12345"│"100"│"50"│"$1000"
     num│num   │num │num│num
     → 0 text, 5 numeric → DATA ✗
```

### Case 3: Product Codes (Text)
```
Row: "P.O"│"ITEM"│"A001"│"Qty"
     text│text │text │text
     → 4 text, 0 numeric → HEADER ✓
```

---

## Testing

Test with your problematic file:
```bash
python config_template_cli/config_data_extractor/analyze_excel.py \
    your_file.xlsx \
    --json \
    --quantity-mode
```

Look for:
```
[BOLD_DETECTION] Row X: Y bold cells, Z text, W numeric
```

**Verify:**
- Header rows show: `more text than numeric`
- Data rows show: `more numeric than text`
- Only header rows are selected

---

## What Changed

### Before:
```python
# Only checked bold count
if bold_count >= 3:
    candidates.append(row)  # Could include data rows! ❌
```

### After:
```python
# Checks bold count AND content type
if bold_count >= 3 and text_count > numeric_count:
    candidates.append(row)  # Only headers! ✓
```

---

## Summary

| Issue | Status |
|-------|--------|
| Data rows extracted as headers | ✅ **FIXED** |
| 2-row detection extracts data row | ✅ **FIXED** |
| Content validation missing | ✅ **ADDED** |
| Debug output unclear | ✅ **IMPROVED** |

---

**The system now ONLY extracts HEADER rows (text labels), never DATA rows (numeric values)!** ✅

Test it and check the debug output to see the text/numeric counts for each row!

