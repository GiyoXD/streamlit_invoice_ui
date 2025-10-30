# Header Detection Improvement Ideas

## Current Problems

### 1. **Simple First-Match Strategy**
The current algorithm finds the **first row** that contains **any** header keyword and stops:
```python
for idx, row in enumerate(worksheet.iter_rows()):
    for cell in row:
        if self._matches_keyword(cell_value, keyword):
            header_row_found = cell.row  # Takes first match!
    if header_row_found:
        break  # Stops immediately
```

**Problem:** If there's a title, company name, or date that contains words like "Description" or "Amount", it will incorrectly identify that as the header row.

### 2. **No Scoring/Ranking System**
There's no way to evaluate which row is the **best candidate** - just the first match wins.

### 3. **No Multi-Keyword Verification**
A single keyword match is enough to declare a row as the header, even if it's clearly not a table header.

### 4. **Limited Context Awareness**
Doesn't check:
- What's below the header (should be data rows)
- What's above the header (should be metadata/titles)
- Row density (headers usually have many filled cells)

---

## Recommended Improvements

### **Strategy 1: Scoring-Based Detection (Recommended) ⭐**

Instead of taking the first match, score ALL candidate rows and pick the best one.

```python
def find_headers_with_scoring(self, worksheet: Worksheet) -> List[HeaderMatch]:
    """
    Find header row using a comprehensive scoring system.
    """
    max_rows_to_check = 30
    candidate_rows = []
    
    # Score each row
    for row_num in range(1, min(max_rows_to_check + 1, worksheet.max_row + 1)):
        score = self._score_header_row(worksheet, row_num)
        if score > 0:
            candidate_rows.append((row_num, score))
    
    if not candidate_rows:
        return []
    
    # Sort by score (descending) and pick the best
    candidate_rows.sort(key=lambda x: x[1], reverse=True)
    best_row = candidate_rows[0][0]
    
    print(f"[HEADER_DETECTION] Candidates: {candidate_rows}")
    print(f"[HEADER_DETECTION] Selected row {best_row} with score {candidate_rows[0][1]}")
    
    # Extract headers from best row
    is_double_header = self._is_double_header(worksheet, best_row)
    if is_double_header:
        return self._extract_double_header(worksheet, best_row)
    else:
        return self._extract_all_headers_from_row(worksheet, best_row)

def _score_header_row(self, worksheet: Worksheet, row_num: int) -> float:
    """
    Score a row based on how likely it is to be a header row.
    Higher score = more likely to be header.
    """
    score = 0.0
    cells_with_data = 0
    cells_with_keywords = 0
    total_keyword_quality = 0
    
    for col in range(1, min(20, worksheet.max_column + 1)):
        cell = worksheet.cell(row=row_num, column=col)
        if cell.value is not None:
            cell_value = str(cell.value).strip()
            if cell_value:
                cells_with_data += 1
                
                # Check if this cell matches any keyword
                best_match_quality = 0
                for keyword in self.header_keywords:
                    quality = self._keyword_match_quality(cell_value, keyword)
                    if quality > best_match_quality:
                        best_match_quality = quality
                
                if best_match_quality > 0:
                    cells_with_keywords += 1
                    total_keyword_quality += best_match_quality
    
    if cells_with_data == 0:
        return 0
    
    # Scoring factors:
    
    # 1. Keyword match ratio (0-40 points)
    keyword_ratio = cells_with_keywords / cells_with_data
    score += keyword_ratio * 40
    
    # 2. Number of keyword matches (0-30 points)
    score += min(cells_with_keywords * 5, 30)
    
    # 3. Average keyword quality (0-20 points)
    if cells_with_keywords > 0:
        avg_quality = total_keyword_quality / cells_with_keywords
        score += avg_quality * 20
    
    # 4. Row density bonus (0-10 points)
    # Headers typically have many consecutive filled cells
    if cells_with_data >= 5:
        score += min(cells_with_data, 10)
    
    return score

def _keyword_match_quality(self, cell_value: str, keyword: str) -> float:
    """
    Return match quality from 0.0 (no match) to 1.0 (perfect match).
    """
    cell_lower = cell_value.lower().strip()
    keyword_lower = keyword.lower()
    
    # Exact match - highest quality
    if cell_lower == keyword_lower:
        return 1.0
    
    # Cell contains only the keyword with some punctuation/whitespace
    import re
    cell_clean = re.sub(r'[^\w\s]', '', cell_lower)
    cell_clean = ' '.join(cell_clean.split())
    keyword_clean = re.sub(r'[^\w\s]', '', keyword_lower)
    
    if cell_clean == keyword_clean:
        return 0.95
    
    # Keyword is substantial part of cell
    if keyword_lower in cell_lower and len(cell_lower) <= 30:
        ratio = len(keyword_lower) / len(cell_lower)
        if ratio >= 0.7:
            return 0.8
        elif ratio >= 0.5:
            return 0.6
    
    return 0.0
```

**Benefits:**
- ✅ Finds the BEST header row, not just the first
- ✅ Handles cases where titles/metadata contain keywords
- ✅ More robust and predictable
- ✅ Easy to tune scoring weights

---

### **Strategy 2: Context-Aware Detection ⭐**

Check what's around the candidate row to verify it's actually a header.

```python
def _score_header_row(self, worksheet: Worksheet, row_num: int) -> float:
    """Enhanced scoring with context awareness."""
    base_score = # ... calculate base score as above ...
    
    # Context bonuses/penalties:
    
    # 1. Check row below - should have data, not more headers
    if row_num < worksheet.max_row:
        below_looks_like_data = self._looks_like_data_row(worksheet, row_num + 1)
        if below_looks_like_data:
            base_score += 15  # Bonus for data below
        else:
            base_score -= 10  # Penalty if next row also looks like headers
    
    # 2. Check row above - should NOT look like a header
    if row_num > 1:
        above_is_header = self._looks_like_header_row(worksheet, row_num - 1)
        if above_is_header:
            base_score -= 20  # Strong penalty - likely we're looking at wrong row
    
    # 3. Consistency check - headers should be relatively aligned
    alignment_score = self._check_cell_alignment(worksheet, row_num)
    base_score += alignment_score * 5
    
    return base_score

def _looks_like_data_row(self, worksheet: Worksheet, row_num: int) -> bool:
    """Check if a row looks like data (numbers, short values)."""
    numeric_cells = 0
    text_cells = 0
    
    for col in range(1, min(15, worksheet.max_column + 1)):
        cell = worksheet.cell(row=row_num, column=col)
        if cell.value is not None:
            try:
                float(str(cell.value))
                numeric_cells += 1
            except:
                text_cells += 1
    
    # Data rows typically have more numbers than header rows
    return numeric_cells > text_cells

def _looks_like_header_row(self, worksheet: Worksheet, row_num: int) -> bool:
    """Check if a row looks like a header."""
    score = self._score_header_row(worksheet, row_num)
    return score > 30  # Threshold for "looks like header"
```

**Benefits:**
- ✅ Avoids false positives from title rows
- ✅ Validates that the structure makes sense
- ✅ More reliable in complex layouts

---

### **Strategy 3: Multi-Row Analysis with Confidence Threshold**

Analyze multiple candidate rows and require a minimum confidence level.

```python
def find_headers_with_confidence(self, worksheet: Worksheet, 
                                  min_confidence: float = 40.0) -> List[HeaderMatch]:
    """
    Find header row with confidence threshold.
    """
    max_rows = 30
    candidates = []
    
    for row_num in range(1, min(max_rows + 1, worksheet.max_row + 1)):
        score = self._score_header_row(worksheet, row_num)
        if score >= min_confidence:
            candidates.append({
                'row': row_num,
                'score': score,
                'keywords_found': self._count_keywords_in_row(worksheet, row_num)
            })
    
    if not candidates:
        print(f"[WARNING] No header row found with confidence >= {min_confidence}")
        return []
    
    # Pick candidate with highest score
    best_candidate = max(candidates, key=lambda x: x['score'])
    
    print(f"[HEADER_DETECTION] Found {len(candidates)} candidates:")
    for c in candidates:
        marker = " ← SELECTED" if c == best_candidate else ""
        print(f"  Row {c['row']}: score={c['score']:.1f}, "
              f"keywords={c['keywords_found']}{marker}")
    
    return self._extract_headers(worksheet, best_candidate['row'])
```

**Benefits:**
- ✅ Transparent decision-making
- ✅ Easy debugging (shows all candidates)
- ✅ Configurable confidence threshold

---

### **Strategy 4: Pattern-Based Detection**

Use common Excel patterns to detect headers.

```python
def _score_header_row(self, worksheet: Worksheet, row_num: int) -> float:
    """Score with pattern detection."""
    score = # ... base score ...
    
    # Pattern 1: Headers often have bold formatting
    bold_cells = 0
    for col in range(1, min(15, worksheet.max_column + 1)):
        cell = worksheet.cell(row=row_num, column=col)
        if cell.font and cell.font.bold:
            bold_cells += 1
    
    if bold_cells >= 3:
        score += 10  # Bonus for bold formatting
    
    # Pattern 2: Headers often have background color
    colored_cells = 0
    for col in range(1, min(15, worksheet.max_column + 1)):
        cell = worksheet.cell(row=row_num, column=col)
        if cell.fill and cell.fill.start_color and \
           cell.fill.start_color.index != '00000000':  # Not transparent
            colored_cells += 1
    
    if colored_cells >= 3:
        score += 10  # Bonus for background color
    
    # Pattern 3: Headers often have borders (especially bottom border)
    bordered_cells = 0
    for col in range(1, min(15, worksheet.max_column + 1)):
        cell = worksheet.cell(row=row_num, column=col)
        if cell.border and cell.border.bottom and \
           cell.border.bottom.style:
            bordered_cells += 1
    
    if bordered_cells >= 3:
        score += 5  # Bonus for borders
    
    return score
```

**Benefits:**
- ✅ Uses visual cues that humans use
- ✅ Works even with unusual header text
- ✅ More robust for varied formats

---

### **Strategy 5: Machine Learning / Statistical Approach** (Advanced)

For the most robust solution, use statistical features:

```python
def _extract_row_features(self, worksheet: Worksheet, row_num: int) -> dict:
    """Extract features for ML-based detection."""
    features = {
        'keyword_matches': 0,
        'avg_cell_length': 0,
        'bold_cells': 0,
        'colored_cells': 0,
        'numeric_cells': 0,
        'text_cells': 0,
        'filled_cells': 0,
        'cells_with_spaces': 0,
        'cells_with_caps': 0,
        'position_ratio': row_num / worksheet.max_row,  # Relative position
    }
    
    # ... calculate features ...
    
    return features

def _predict_header_likelihood(self, features: dict) -> float:
    """
    Use a simple rule-based model or trained classifier.
    Returns probability that this row is a header (0-1).
    """
    # Simple heuristic model
    score = 0
    
    # Keywords are strong indicator
    score += features['keyword_matches'] * 0.3
    
    # Headers have more text cells than numeric
    if features['text_cells'] > features['numeric_cells']:
        score += 0.2
    
    # Headers are usually in first 30% of sheet
    if features['position_ratio'] < 0.3:
        score += 0.15
    
    # Headers often have caps
    score += min(features['cells_with_caps'] / 10, 0.15)
    
    # Formatting matters
    score += min(features['bold_cells'] / 10, 0.1)
    score += min(features['colored_cells'] / 10, 0.1)
    
    return min(score, 1.0)
```

---

## **Recommended Implementation Plan**

### Phase 1: Quick Win (1-2 hours)
Implement **Strategy 1 (Scoring-Based Detection)** - this will immediately fix most issues.

### Phase 2: Enhanced (2-3 hours)
Add **Strategy 2 (Context-Aware Detection)** - this handles edge cases.

### Phase 3: Polish (1-2 hours)
Add **Strategy 4 (Pattern-Based Detection)** - this handles unusual formats.

### Phase 4: Advanced (Optional, 4-6 hours)
Implement **Strategy 5 (ML/Statistical)** if needed for maximum robustness.

---

## Testing Strategy

Create test cases with:
1. ✅ Title rows before headers
2. ✅ Company names containing keywords
3. ✅ Multi-row headers
4. ✅ Headers with minimal keywords
5. ✅ Headers in different positions (row 5, 10, 15, etc.)
6. ✅ Multiple tables in one sheet

---

## Example Output (Debug Mode)

```
[HEADER_DETECTION] Analyzing rows 1-30...
[HEADER_DETECTION] Candidates found:
  Row 3: score=25.5, keywords=2 (likely title)
  Row 5: score=78.3, keywords=8 ← SELECTED
  Row 7: score=15.2, keywords=1 (likely subtitle)
[HEADER_DETECTION] Selected row 5 with confidence 78.3
[HEADER_DETECTION] Extracted 12 headers from row 5
```

This makes debugging much easier!

---

## Configuration Options

Add to `mapping_config.json`:

```json
{
  "header_detection": {
    "min_confidence_score": 40.0,
    "max_rows_to_scan": 30,
    "min_keywords_required": 3,
    "use_formatting_hints": true,
    "use_context_analysis": true,
    "debug_mode": false
  }
}
```

This allows users to tune detection for their specific needs.

