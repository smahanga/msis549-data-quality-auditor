---
name: data-inspector
description: Analyze tabular datasets (CSV, Excel, TSV) to identify data quality issues including missing values, duplicates, outliers, type inconsistencies, and structural problems. Use when asked to audit the data, check data quality, or inspect data quality.
---

# Data Inspector

Performs comprehensive data quality analysis on tabular datasets and generates a structured assessment report.

## Overview

This skill examines datasets to identify quality issues that could affect analysis or reporting. It detects missing values, duplicates, outliers, data type problems, and structural anomalies, then prioritizes findings by severity.

## Inputs

- **Dataset file**: CSV, Excel (.xlsx, .xls), or TSV file
- **Column context** (optional): User-provided information about what columns represent (e.g., "user_id should be unique", "age should be 0-120")
- **Thresholds** (optional): Custom thresholds for outlier detection or missing value tolerance (defaults provided)

## Outputs

- **Quality assessment JSON**: Structured report with issues categorized by type and severity
- **Summary statistics**: Row count, column count, data types, completeness percentage
- **Issue details**: Specific rows/values flagged with explanations
- **Recommendations**: Prioritized list of suggested fixes

## Workflow

### Step 1: Load and Profile Dataset

1. Load the dataset using appropriate library (pandas for CSV/Excel)
2. Capture basic statistics:
   - Total rows and columns
   - Column names and inferred data types
   - Memory usage
3. If loading fails, report the error with diagnostic info (file format, encoding issues, corruption)

### Step 2: Detect Missing Values

1. Count missing values per column (NaN, None, empty strings, whitespace-only)
2. Calculate missingness percentage per column
3. Flag columns with >20% missing as HIGH severity, 5-20% as MEDIUM, <5% as LOW
4. Identify patterns (e.g., all values missing after certain row, clustered missingness)

### Step 3: Check for Duplicates

1. Identify exact duplicate rows (all columns identical)
2. Count duplicate occurrences and list row indices
3. If user specified key columns, check for duplicates on those columns only
4. Flag duplicate groups with more than 2 occurrences as HIGH severity

### Step 4: Detect Type Inconsistencies

1. For each column, analyze value types:
   - Numeric columns with text values
   - Date columns with invalid formats
   - Mixed types within same column
2. Sample inconsistent values (up to 5 examples per column)
3. Flag columns with >10% type inconsistencies as HIGH severity

### Step 5: Identify Outliers

1. For numeric columns, use IQR method (values beyond Q1 - 1.5*IQR or Q3 + 1.5*IQR)
2. For categorical columns, flag values appearing <0.1% of time if column has <100 unique values
3. List outlier values with their row indices (up to 10 per column)
4. If user provided expected ranges, flag values outside those ranges as HIGH severity

### Step 6: Structural Checks

1. Check for empty rows (all values null)
2. Check for empty columns (all values null)
3. Identify columns with single unique value (potentially useless)
4. Flag inconsistent row lengths in CSV files
5. Check for special characters or encoding issues in column names

### Step 7: Generate Prioritized Report

1. Compile all findings into structured JSON format
2. Calculate overall quality score (0-100):
   - Start at 100
   - Subtract points for each issue based on severity (HIGH: -15, MEDIUM: -5, LOW: -2)
   - Floor at 0
3. Prioritize recommendations by impact:
   - Critical issues that block analysis (type inconsistencies, structural problems)
   - Important issues that reduce quality (duplicates, missing values)
   - Minor issues that improve usability (outliers, naming conventions)

## Output Format

```json
{
  "summary": {
    "total_rows": 1000,
    "total_columns": 15,
    "quality_score": 67,
    "completeness_pct": 92.3,
    "issues_found": 12
  },
  "missing_values": [
    {
      "column": "email",
      "count": 45,
      "percentage": 4.5,
      "severity": "LOW",
      "pattern": "Random distribution"
    }
  ],
  "duplicates": {
    "exact_duplicates": 23,
    "duplicate_groups": [
      {"rows": [45, 67, 89], "count": 3}
    ],
    "severity": "MEDIUM"
  },
  "type_issues": [
    {
      "column": "age",
      "expected_type": "numeric",
      "inconsistent_count": 12,
      "examples": ["N/A", "unknown", ""],
      "severity": "HIGH"
    }
  ],
  "outliers": [
    {
      "column": "price",
      "method": "IQR",
      "count": 8,
      "values": [999999, 0.01],
      "severity": "MEDIUM"
    }
  ],
  "structural_issues": [
    {
      "type": "empty_column",
      "column": "unused_field",
      "severity": "LOW"
    }
  ],
  "recommendations": [
    {
      "priority": 1,
      "issue": "Type inconsistencies in 'age' column",
      "action": "Convert or remove non-numeric values",
      "impact": "Blocks numeric analysis"
    }
  ]
}
```

## Constraints and Failure Modes

**Constraints:**
- Maximum dataset size: 100MB (recommend sampling for larger files)
- Assumes tabular data with consistent column structure
- Requires at least 2 rows (header + data) to perform analysis
- Outlier detection only works on numeric columns with sufficient variation

**Known Failure Modes:**
1. **Encoding issues**: Non-UTF-8 files may fail to load properly
   - Mitigation: Try common encodings (UTF-8, latin1, cp1252) in order
2. **Malformed CSV**: Inconsistent delimiters or quote characters
   - Mitigation: Use csv.Sniffer to detect format, report if detection fails
3. **Memory constraints**: Very wide datasets (1000+ columns) may cause slowdowns
   - Mitigation: Sample columns if >500 detected
4. **False positives on outliers**: Legitimate extreme values flagged as outliers
   - Mitigation: Always provide context; user decides whether to act

## Example Usage

**User prompt:**
"Analyze this customer dataset and tell me about data quality issues"

**Expected behavior:**
1. Load the provided CSV/Excel file
2. Run all 7 analysis steps
3. Generate quality report JSON
4. Display summary with top 3 most critical issues
5. Provide recommendations ranked by priority

**Sample output snippet:**
```
Quality Score: 67/100

Critical Issues Found:
1. [HIGH] Type inconsistencies in 'age' column: 12 non-numeric values
2. [HIGH] 23 duplicate customer records detected
3. [MEDIUM] 'email' column has 4.5% missing values

Recommendations:
1. Clean 'age' column by standardizing values or removing invalid entries
2. Review duplicates and merge or remove as appropriate
3. Consider requiring email at data entry to reduce missing values
```

## Edge Cases to Handle

1. **All-null columns**: Flag but don't fail; recommend removal
2. **Single-row datasets**: Skip duplicate and outlier detection
3. **Text-only datasets**: Skip outlier detection, focus on missing values and duplicates
4. **Dates in multiple formats**: Detect format inconsistency, list examples
5. **Boolean columns**: Don't flag as outliers; check for unexpected values beyond true/false

## Quality Criteria

Success means:
- All issues detected are actionable and accurate
- No false negatives on critical issues (missing duplicates, type errors)
- Severity ratings align with actual impact on analysis
- Report is clear enough for non-technical users to understand
- Recommendations are specific and implementable
