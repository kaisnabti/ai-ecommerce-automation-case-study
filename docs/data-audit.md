# Data Quality Audit — Product Database

> A systematic audit of a 1,200+ row product database, conducted as part of a Discovery & Architecture phase to assess data readiness before building further automation.

---

## Context

The client flagged concerns about their primary product spreadsheet: duplicates, placeholder links, broken CTAs, and test data mixed with production records. Before building the next automation phase (automated affiliate link generation and publication), a data quality audit was required to determine whether the existing data layer could support reliable automation — or whether cleanup was a prerequisite.

The audit was scoped as a deliverable within the Discovery & Architecture engagement, not as an open-ended investigation.

## Methodology

The audit followed a structured approach across five dimensions:

1. **Structural analysis** — Loaded the full workbook (multiple tabs) to understand the data model, tab purposes, column schemas, and relationships between sheets
2. **Link classification** — Categorised every URL in the affiliate link columns by type: valid affiliate redirect, product page, signup page, placeholder, test link, error, or empty
3. **Duplicate detection** — Identified product names appearing more than once, empty product names, and header text duplicated as data rows
4. **Error classification** — Separated API error messages logged as product rows from actual product data
5. **Cross-reference validation** — Compared the product sheet against the live pipeline sheet to identify orphaned records, stale references, and test/production contamination

Each finding was categorised as **systemic** (pattern affecting many rows, indicating a workflow or integration problem) or **isolated** (one-off data entry issue).

---

## Findings

### 1. Link Quality Crisis

The most critical finding: the vast majority of rows lack functional affiliate links.

| Category | Approximate % |
|---|---|
| **No affiliate link at all** | ~65% |
| Placeholder links with unfilled template parameters | ~10% |
| Platform page URLs (not affiliate redirect links) | ~3% |
| Platform signup page URLs (registration, not affiliate) | <1% |
| Shortened/test links | <1% |
| Placeholder URLs (example.com, N/A) | <1% |
| **Actual functioning affiliate redirect links** | **~7%** |

**Root cause analysis:** Multiple issues contributed. The placeholder links suggest a bulk import that populated URLs without replacing the affiliate ID token. The platform page URLs indicate confusion between different URL types. The placeholder and shortened entries are test data that was never cleaned up.

**Impact:** Any automation built on this data that reads affiliate links would fail or produce broken outputs for over 90% of rows.

### 2. Duplicate Contamination

| Issue | Observation |
|---|---|
| Duplicate product names | Dozens of distinct names appearing more than once |
| Worst single duplicate | One product name appearing 30+ times |
| Empty product names | ~90 rows with no product name at all |
| Header row text duplicated as data | Multiple rows containing column headers as data values |

**Root cause analysis:** The high-count duplicate suggests an automation that ran repeatedly without deduplication logic — likely a Zap that re-processed the same product on each trigger without checking whether it already existed in the sheet. The empty names indicate rows created by an automation that wrote partial data. The header-as-data rows suggest a bulk import that included the CSV/sheet header as a data row.

**Impact:** Any product lookup or matching logic will return incorrect or ambiguous results. Duplicate rows inflate reporting metrics and create confusion about which record is authoritative.

### 3. Error Data Mixed with Product Records

Multiple categories of API error messages were found logged directly as product rows — authentication failures, HTTP errors, and platform-specific error responses mixed into the product database.

**Root cause analysis:** An automation step that calls an external API was writing error responses into the product sheet instead of a separate error log. When the API returned an error, the Zap's "Create Row" step still fired, writing the error message as if it were a product name.

**Impact:** These rows pollute the product database and will be processed as products by any downstream automation. They also mask the frequency and nature of API failures — the errors are scattered across the product sheet rather than consolidated in a log where they could be analysed and resolved.

### 4. Test Data in Production

| Issue | Detail |
|---|---|
| Test product entries | Multiple rows with test URLs |
| Test gate entry | A pipeline validation test left in the production sheet |
| Test row as only valid record | Only 1 of ~30 pipeline rows had an affiliate link — and it was the test row |

**Root cause analysis:** Test data was created during development and never removed. The test gate entry indicates a pipeline validation test that was left in production. The fact that the only pipeline row with a valid affiliate link was the test row suggests the production pipeline had never successfully processed a real product end-to-end with a valid affiliate link.

**Impact:** Test data skews reporting, triggers production automations unnecessarily, and creates false positives when validating that the pipeline "works."

### 5. Data Integrity Issues

| Issue | Detail |
|---|---|
| **AI hallucination in column header** | One column's header contained a multi-paragraph unrelated text block instead of a column name — the AI that generated the sheet structure hallucinated content |
| **Date format inconsistency** | Multiple different date formats across the sheet |
| **Orphaned workflow rows** | Rows stuck at a status from an abandoned workflow that no longer exists in the pipeline |
| **Deleted product references** | Products deleted from the fulfilment platform still referenced in the pipeline sheet |
| **Empty URLs** | Rows with completely empty URL fields (not placeholder — genuinely blank) |

### 6. Workbook Structure Issues

The workbook contained multiple tabs with overlapping purposes and no documentation:

| Tab Type | Observation |
|---|---|
| Main product sheet | Primary audit target (1,200+ rows) |
| Sales tracker | Unclear relationship to main sheet |
| Promotion log | Overlaps with main sheet data |
| Deals tab | Purpose undocumented |
| Secondary network tabs | Different affiliate network, separate schema |
| Pipeline tab | Near-empty |
| Reporting tabs | Empty |

**Root cause analysis:** Tabs were created organically as the business explored different affiliate networks and product strategies. No tab was formally retired when its purpose was superseded, leading to data spread across multiple locations with no clear ownership.

---

## Categorisation Summary

Of the rows in the primary sheet:

| Category | Approximate % |
|---|---|
| Potentially valid product rows (has name + some link) | ~20% |
| Rows with valid affiliate redirect links | ~7% |
| Rows with no usable affiliate link | ~65% |
| Duplicate/empty product names | ~19% |
| Error log entries masquerading as products | ~5% |
| Test data | ~1% |

> Note: Categories overlap — a row can be both a duplicate AND have no affiliate link.

---

## Recommendations

### Immediate (before any further automation)

1. **Establish a single source of truth:** One row per product, one valid affiliate link per product. Deduplicate by defining which record is authoritative for each product.
2. **Separate error logging from product data:** Create a dedicated error log sheet. Modify the automation to write API errors there, not into the product database.
3. **Remove all test data from production sheets:** Delete test entries, test URLs, and placeholder values. Create a separate test sheet if needed.
4. **Fix the column header issue:** Replace the AI-hallucinated content with the intended header name.

### Structural (before scaling)

5. **Standardise date formats:** Pick one format and enforce it via data validation rules or automation-side formatting.
6. **Add data validation rules:** Prevent placeholder strings and test URLs from being written to production columns.
7. **Document tab purposes and ownership:** Each tab should have a header row or documentation note explaining what it contains, what writes to it, and whether it's active or deprecated.
8. **Archive or delete empty/deprecated tabs:** Empty reporting tabs and deprecated workflow statuses should be cleaned up or archived.

### Principle

**Clean data before automating.** Automation amplifies whatever it's built on. A pipeline that processes 1,200+ rows — over 90% of which have broken or missing affiliate links — will reliably produce over 1,100 broken outputs. The automation will work perfectly; the results will be useless.

---

> **Disclaimer:** This case study describes generalised patterns from automation consulting work. Specific implementation details — including row counts, percentages, and platform names — have been adapted for portfolio purposes and do not represent any single client engagement.
