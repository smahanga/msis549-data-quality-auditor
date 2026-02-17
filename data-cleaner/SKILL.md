---
name: data-cleaner
description: Apply data quality fixes to datasets based on inspection results or user-specified cleaning rules. Handles missing values, duplicates, type conversions, and outliers while documenting all changes. Use when asked to clean, fix, or repair data.
---

# Data Cleaner

Applies systematic data quality fixes with full transparency and reversibility.

## Overview

This skill takes a dataset and an inspection report (from data-inspector) or user-specified rules, then applies appropriate cleaning operations. All changes are documented in a changelog for auditability.

## Inputs

- **Original dataset**: CSV, Excel, or TSV file to be cleaned
- **Inspection report** (optional): JSON output from data-inspector skill
- **Cleaning preferences**: User choices for how to handle each issue type
- **Custom rules** (optional): Specific transformations (e.g., "convert 'N/A' to null in age column")

## Outputs

- **Cleaned dataset**: Same format as input, with fixes applied
- **Changelog**: Detailed record of every change made (row, column, old value, new value, reason)
- **Cleaning summary**: Statistics on operations performed
- **Validation report**: Confirmation that fixes didn't introduce new issues

## Cleaning Strategies

### Default Strategies (Applied When User Doesn't Specify)

| Issue Type | Default Strategy | Alternative Options |
|------------|------------------|---------------------|
| Missing values (numeric) | Mean imputation for <10% missing; flag for review if >10% | Median, mode, drop rows, forward fill |
| Missing values (categorical) | Mode imputation for <10% missing; "UNKNOWN" category if >10% | Drop rows, forward fill, custom value |
| Exact duplicates | Keep first occurrence, remove rest | Keep last, keep all, manual review |
| Type inconsistencies | Convert to most common type; null if conversion fails | Drop invalid rows, custom conversion rules |
| Date format inconsistencies | Standardize to YYYY-MM-DD (ISO 8601); infer from context if malformed | Keep original format, convert to MM/DD/YYYY, flag for manual review |
| Outliers | Remove, flag only, winsorize, custom bounds |
| Empty columns | Remove column entirely | Keep with warning |
| Empty rows | Remove row entirely | Keep with warning |

## Workflow

### Step 1: Parse Inputs and Set Strategy

1. Load the original dataset
2. If inspection report provided, extract issues list
3. For each issue type, determine strategy:
   - Use user-specified preference if provided
   - Otherwise use default strategy
4. Display strategy summary and ask for user confirmation before proceeding
5. Create changelog structure

### Step 2: Handle Missing Values

For each column with missing values:

1. Identify strategy (mean/median/mode/drop/custom)
2. Apply strategy:
   - **Mean imputation**: Calculate mean of non-null numeric values, fill nulls
   - **Median imputation**: Calculate median of non-null numeric values, fill nulls
   - **Mode imputation**: Use most frequent value, fill nulls
   - **Forward fill**: Use previous valid value in sequence
   - **Custom value**: Fill with user-specified value
   - **Drop rows**: Remove rows where this column is null
3. Log each change: `{"row": 45, "column": "age", "old_value": null, "new_value": 32, "reason": "Mean imputation"}`
4. Update summary: `{"missing_values_filled": 45, "rows_dropped": 0}`

### Step 3: Remove Duplicates

1. Identify duplicate rows based on:
   - All columns (exact duplicates)
   - Key columns if user specified
2. Apply de-duplication strategy:
   - Keep first occurrence (default)
   - Keep last occurrence
   - Keep row with most complete data
3. Log removed rows: `{"rows_removed": [67, 89], "reason": "Duplicate of row 45"}`
4. Update summary: `{"duplicates_removed": 23}`

### Step 4: Fix Type Inconsistencies

For each column with type issues:

1. Determine target type (numeric, date, categorical, text)
2. Apply conversion:
   - **To numeric**: Use pd.to_numeric with errors='coerce' (invalid → NaN)
   - **To datetime**: Use pd.to_datetime with format detection
   - **To categorical**: Convert all to string type
3. Handle conversion failures:
   - Log failed conversions: `{"row": 12, "column": "age", "old_value": "N/A", "new_value": null, "reason": "Could not convert to numeric"}`
   - If >10% fail conversion, warn user and recommend manual review
4. Update summary: `{"type_conversions": 12, "conversion_failures": 3}`

### Step 5: Handle Outliers

For each numeric column with outliers:

1. Calculate boundaries:
   - IQR method: Q1 - 1.5*IQR, Q3 + 1.5*IQR
   - Percentile method: 1st and 99th percentiles
   - Custom bounds if user provided
2. Apply strategy:
   - **Cap**: Replace values beyond bounds with boundary values
   - **Remove**: Drop rows with outlier values
   - **Flag only**: Keep values but add outlier indicator column
   - **Winsorize**: Replace with nearest non-outlier value
3. Log changes: `{"row": 234, "column": "price", "old_value": 999999, "new_value": 1250, "reason": "Capped at 99th percentile"}`
4. Update summary: `{"outliers_capped": 8, "outliers_removed": 0}`

### Step 6: Remove Structural Issues

1. Drop empty columns (all values null)
2. Drop empty rows (all values null)
3. Remove columns with single unique value (if user confirms)
4. Log: `{"columns_removed": ["unused_field"], "rows_removed": [1001, 1002]}`
5. Update summary: `{"empty_columns_removed": 1, "empty_rows_removed": 2}`

### Step 7: Validate and Finalize

1. Run basic validation checks:
   - Row count change matches expected (duplicates removed, etc.)
   - No new null values introduced (except from failed conversions)
   - Data types are consistent within columns
   - No unexpected data loss
2. Generate validation report
3. Save cleaned dataset to file
4. Save detailed changelog as CSV
5. Display cleaning summary

## Output Formats

### Cleaned Dataset
Same format as input (CSV → CSV, Excel → Excel)

### Changelog CSV
```csv
row_index,column,old_value,new_value,operation,reason
45,age,,32,imputation,Mean imputation for missing values
67,customer_id,12345,REMOVED,deduplication,Duplicate of row 45
12,price,999999,1250,capping,Outlier capped at 99th percentile
```

### Cleaning Summary JSON
```json
{
  "operations_summary": {
    "missing_values_filled": 45,
    "duplicates_removed": 23,
    "type_conversions": 12,
    "outliers_handled": 8,
    "empty_columns_removed": 1,
    "empty_rows_removed": 2
  },
  "final_stats": {
    "original_rows": 1000,
    "final_rows": 975,
    "original_columns": 15,
    "final_columns": 14,
    "data_loss_pct": 2.5
  },
  "validation": {
    "passed": true,
    "warnings": ["3 values could not be converted to numeric in 'age' column"],
    "errors": []
  }
}
```

## Constraints and Safety Rules

**Constraints:**
- Never modify the original file directly; always create a new cleaned version
- Maximum 25% data loss allowed (rows or values) - if exceeded, warn and ask for confirmation
- Preserve data types where possible unless explicitly converting
- Maintain column order unless removing empty columns

**Safety Rules:**
1. **Always ask before destructive operations**: Removing >10 rows, dropping columns, or losing >5% of data
2. **Preserve IDs**: Never impute or modify columns with "id" in the name
3. **Protect dates**: Don't apply numeric imputation to date columns
4. **Log everything**: Every single change must be in the changelog

**Known Failure Modes:**
1. **Imputation introduces bias**: Mean/median may not reflect true distribution
   - Mitigation: Document imputation method, recommend user validation
2. **Type conversion loses precision**: Numeric rounding or date truncation
   - Mitigation: Log precision loss, offer user choice
3. **Duplicate removal loses information**: May remove legitimate entries
   - Mitigation: Require user confirmation, log which rows removed
4. **Outlier capping distorts distribution**: May hide important edge cases
   - Mitigation: Offer "flag only" option, document bounds used

## Example Usage

**User prompt:**
"Clean this dataset using the inspection report, but keep all outliers and just flag them"

**Expected behavior:**
1. Load original dataset and inspection report
2. Set default strategies except outliers → "flag only"
3. Display strategy summary:
   ```
   Cleaning Strategy:
   - Missing values: Mean/mode imputation
   - Duplicates: Keep first, remove rest
   - Type issues: Convert to target type, null if fails
   - Outliers: Flag only (no removal/capping)
   - Structural: Remove empty rows/columns
   
   Proceed? [y/n]
   ```
4. If confirmed, apply all operations
5. Generate changelog, save cleaned dataset
6. Display summary:
   ```
   Cleaning Complete!
   
   Operations Applied:
   - Filled 45 missing values
   - Removed 23 duplicate rows
   - Converted 12 values to numeric
   - Flagged 8 outliers (new column: price_outlier_flag)
   - Removed 1 empty column
   
   Final Dataset: 975 rows × 15 columns
   Data Loss: 2.5% (within acceptable range)
   
   Files created:
   - cleaned_data.csv
   - changelog.csv
   - cleaning_summary.json
   ```

## Edge Cases to Handle

1. **All rows flagged as duplicates**: Keep at least one row, warn user
2. **100% missing values in column**: Drop column instead of imputing
3. **No valid values for imputation**: Use global fallback (0 for numeric, "UNKNOWN" for categorical)
4. **Conflicting user rules**: Prioritize explicit rules over defaults, warn about conflicts
5. **Data becomes smaller than minimum viable**: Require confirmation if <10 rows remain

## Quality Criteria

Success means:
- All intended operations completed without errors
- Changelog accurately reflects every change made
- No unintended data loss or corruption
- Validation checks pass
- User understands what was changed and why
- Cleaned dataset is immediately usable for analysis
