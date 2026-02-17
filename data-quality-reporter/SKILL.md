---
name: data-quality-reporter
description: Generate comprehensive, professional data quality reports combining inspection findings, cleaning operations, and recommendations. Creates both technical summaries and executive-friendly overviews. Use when asked to create a report of data quality analysis, create a summary of data quality analysis, or documentation of data quality analysis.
---

# Data Quality Reporter

Creates polished, multi-format data quality reports for technical and non-technical audiences.

## Overview

This skill transforms data inspection results and cleaning logs into professional reports suitable for stakeholders, documentation, and auditing. Generates visualizations, executive summaries, and detailed technical appendices.

## Inputs

- **Inspection report** (required): JSON output from data-inspector
- **Cleaning summary** (optional): JSON output from data-cleaner
- **Changelog** (optional): CSV of cleaning operations from data-cleaner
- **Report preferences**: Format (PDF/HTML/Markdown), audience level (executive/technical/mixed)
- **Dataset context** (optional): Dataset name, purpose, source, owner for report header

## Outputs

- **Executive summary**: 1-page overview with key metrics and top recommendations
- **Technical report**: Detailed findings with data, charts, and methodology
- **Visualizations**: Charts showing data quality metrics (PNG/SVG)
- **Action plan**: Prioritized list of next steps with effort estimates

## Report Structure

### Executive Summary (1 page)

1. **Header**: Dataset name, date, quality score
2. **At-a-Glance Metrics**:
   - Overall quality score (0-100)
   - Completeness percentage
   - Total issues found (by severity)
   - Data loss from cleaning (if applicable)
3. **Top 3 Critical Findings**: Brief description of most important issues
4. **Impact Statement**: What these issues mean for analysis/decision-making
5. **Recommended Actions**: Top 3 priority fixes with effort estimates

### Technical Report (Multi-page)

1. **Dataset Profile**:
   - Dimensions (rows × columns)
   - Data types distribution
   - Size and memory usage
   - Column inventory table

2. **Quality Assessment**:
   - Quality score breakdown (scoring methodology)
   - Completeness analysis by column
   - Issue summary table (count by type and severity)

3. **Detailed Findings**:
   - **Missing Values Section**: Table with column, count, %, pattern
   - **Duplicates Section**: Count, examples, distribution
   - **Type Issues Section**: Column, expected vs actual, examples
   - **Outliers Section**: Detection method, count, examples
   - **Structural Issues Section**: Empty rows/columns, formatting problems

4. **Data Cleaning Operations** (if cleaning was performed):
   - Summary of operations performed
   - Before/after comparison metrics
   - Sample of changes from changelog
   - Validation results

5. **Visualizations**:
   - Quality score gauge chart
   - Completeness heatmap (by column)
   - Issue distribution pie chart
   - Before/after comparison (if cleaned)

6. **Recommendations**:
   - Prioritized action items table
   - Effort estimates (low/medium/high)
   - Expected impact
   - Implementation notes

7. **Appendices**:
   - Full changelog (if cleaning performed)
   - Methodology documentation
   - Column-by-column detailed statistics

## Workflow

### Step 1: Parse Inputs and Determine Report Type

1. Load inspection report (required)
2. Load cleaning summary and changelog if provided
3. Determine report scope:
   - Inspection-only report (no cleaning performed)
   - Full report (inspection + cleaning)
4. Extract dataset context or use defaults
5. Set report format based on user preference (default: HTML with embedded charts)

### Step 2: Calculate Derived Metrics

1. **Quality score**: Use from inspection or recalculate if cleaning changed it
2. **Issue density**: Issues per 100 rows
3. **Completeness score**: (1 - missing_pct) * 100
4. **Cleaning effectiveness** (if applicable): Issues resolved / issues found
5. **Data retention**: Final rows / original rows

### Step 3: Generate Visualizations

Use Python with matplotlib/seaborn for charts:

1. **Quality Score Gauge**: Semicircle gauge showing 0-100 score
   - 0-40: Red (Poor)
   - 41-70: Yellow (Fair)
   - 71-85: Light Green (Good)
   - 86-100: Dark Green (Excellent)

2. **Completeness Heatmap**: Grid showing missing % by column
   - Each cell colored by severity
   - Column names on Y-axis

3. **Issue Distribution**: Pie chart of issues by type
   - Missing values
   - Duplicates
   - Type issues
   - Outliers
   - Structural issues

4. **Before/After Comparison** (if cleaning performed): Side-by-side bar chart
   - Missing values: before vs after
   - Duplicates: before vs after
   - Type issues: before vs after

Save charts as PNG files with 300 DPI for quality.

### Step 4: Build Executive Summary

1. Create header with dataset info and quality score
2. Format at-a-glance metrics in clean table
3. Extract top 3 issues by severity score:
   - Severity score = severity_weight × count × impact
   - HIGH issues weighted 3x, MEDIUM 2x, LOW 1x
4. Write impact statement based on issue types:
   - Missing data → "May lead to biased analysis"
   - Type issues → "Blocks quantitative analysis"
   - Duplicates → "Inflates counts and distorts metrics"
5. Select top 3 recommendations prioritized by:
   - Impact (how much it improves quality)
   - Effort (how hard to implement)
   - Dependencies (what else needs to be done first)

### Step 5: Build Technical Report

1. **Dataset Profile Section**:
   - Create table with column name, data type, non-null count, unique values
   - Calculate memory usage per column
   - List top 5 most complete and least complete columns

2. **Quality Assessment Section**:
   - Explain quality score methodology
   - Show breakdown: Starting 100, subtract per issue type
   - Create completeness table sorted by missing %

3. **Detailed Findings Sections**:
   - For each issue type, create formatted table with examples
   - Include sample values for type issues and outliers
   - Show distribution of duplicates (how many groups, sizes)

4. **Cleaning Operations Section** (if applicable):
   - Summary statistics table
   - Top 10 rows from changelog
   - Before/after comparison for each metric
   - Validation status (passed/warnings/errors)

5. **Recommendations Section**:
   - Table with columns: Priority, Issue, Action, Effort, Impact
   - Sort by priority (calculated score)
   - Include implementation notes for complex actions

### Step 6: Format and Export

1. **HTML Format** (default):
   - Professional CSS styling
   - Embedded base64-encoded charts
   - Collapsible sections for detailed tables
   - Print-friendly stylesheet

2. **Markdown Format**:
   - GitHub-flavored markdown
   - Image links to chart files
   - Tables in markdown format
   - Code blocks for technical details

3. **PDF Format**:
   - Convert HTML to PDF using weasyprint
   - Include page numbers and headers
   - Embedded charts at high resolution
   - Professional typography

### Step 7: Generate Action Plan Document

Create separate actionable checklist:

```markdown
# Data Quality Action Plan
**Dataset**: [name]
**Generated**: [date]
**Priority**: Critical → High → Medium → Low

## Critical Actions (Do First)
- [ ] Fix type inconsistencies in 'age' column (Effort: Low, Impact: High)
      Method: Convert 'N/A' to null, ensure numeric type
      Expected time: 15 minutes
      
## High Priority Actions
- [ ] Remove 23 duplicate customer records (Effort: Medium, Impact: High)
      Method: Review duplicates manually, keep most recent
      Expected time: 1 hour

## Medium Priority Actions
- [ ] Handle 4.5% missing emails (Effort: Low, Impact: Medium)
      Method: Impute with mode or mark as "unknown"
      Expected time: 10 minutes

## Low Priority Actions
- [ ] Review outliers in price column (Effort: High, Impact: Low)
      Method: Validate if legitimate, cap or remove
      Expected time: 2 hours
```

## Output Formats

### HTML Report Template

```html
<!DOCTYPE html>
<html>
<head>
    <title>Data Quality Report - [Dataset Name]</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 1200px; margin: 0 auto; padding: 20px; }
        .metric-box { display: inline-block; padding: 15px; margin: 10px; border: 1px solid #ddd; border-radius: 5px; }
        .score-high { color: green; font-weight: bold; }
        .score-medium { color: orange; font-weight: bold; }
        .score-low { color: red; font-weight: bold; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #4CAF50; color: white; }
        .severity-high { background-color: #ffcccc; }
        .severity-medium { background-color: #fff4cc; }
        .severity-low { background-color: #ccffcc; }
    </style>
</head>
<body>
    <h1>Data Quality Report</h1>
    <div class="header">
        <p><strong>Dataset:</strong> [name]</p>
        <p><strong>Generated:</strong> [date]</p>
        <p><strong>Quality Score:</strong> <span class="score-[level]">[score]/100</span></p>
    </div>
    
    <h2>Executive Summary</h2>
    [content]
    
    <h2>Detailed Findings</h2>
    [content]
    
    <h2>Recommendations</h2>
    [content]
</body>
</html>
```

## Constraints and Best Practices

**Constraints:**
- Maximum report size: 50 pages for PDF, unlimited for HTML
- Charts limited to 10 per report to avoid clutter
- Changelog samples limited to top 100 rows in appendix
- Executive summary must fit on single page/screen

**Best Practices:**
1. **Use clear language**: Avoid jargon in executive summary
2. **Provide context**: Explain what metrics mean for the business
3. **Be specific**: "45 missing emails" not "some missing data"
4. **Visual hierarchy**: Use headers, bullets, and tables effectively
5. **Actionable recommendations**: Tell users exactly what to do next

**Known Limitations:**
1. **Subjective severity**: Severity ratings may not match user's context
   - Mitigation: Allow user to override severity in preferences
2. **Generic recommendations**: May not account for domain-specific rules
   - Mitigation: Include "Notes" field for user to add context
3. **Chart rendering**: Complex datasets may produce cluttered visualizations
   - Mitigation: Sample/aggregate data for charts if >50 columns

## Example Usage

**User prompt:**
"Generate an HTML report for the customer dataset inspection and cleaning results. Make it suitable for sharing with my manager."

**Expected behavior:**
1. Load inspection JSON and cleaning summary
2. Set report format to HTML, audience to "executive-friendly"
3. Calculate all derived metrics
4. Generate 4 visualization charts
5. Build executive summary (1 page equivalent)
6. Build technical sections with detailed tables
7. Create action plan checklist
8. Export as HTML with embedded charts
9. Display summary:
   ```
   Report Generated Successfully!
   
   Files created:
   - data_quality_report.html (main report)
   - action_plan.md (checklist)
   - charts/ (4 PNG files)
   
   Report Highlights:
   - Quality Score: 67/100 (Fair)
   - 12 issues identified, 8 resolved through cleaning
   - 3 critical actions recommended
   - Data retention: 97.5%
   
   Share data_quality_report.html with your manager.
   ```

## Quality Criteria

Success means:
- Report is visually professional and easy to read
- Executive summary fits on one page and is non-technical
- Technical sections provide enough detail for reproducibility
- Recommendations are specific, prioritized, and actionable
- Charts accurately represent data without distortion
- Report can be understood by someone unfamiliar with the dataset
- All metrics are correctly calculated and clearly explained
