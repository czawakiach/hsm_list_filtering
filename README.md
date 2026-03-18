# HSM Tape Data Reconciliation Project

## Project Overview

This project was developed to support the full reconciliation of seismic and subsurface exploration data stored on physical tapes managed by **Katalyst Data Management (KDM)**. The original tape consolidation was performed by KDM in 2014, combining data from multiple tapes into a smaller set of physical media.

In the current phase, these tapes were returned to Shell in **two batches**:

- **Batch 1:** 28 tapes
- **Batch 2:** 300+ tapes (remaining)

The core objective was to **verify that all data on the physical tapes is accounted for in Shell's HSM (Hierarchical Storage Management) system** before the physical media could be safely decommissioned and storage resources freed. This required comparing file listings extracted from each tape against the HSM database (~1.7 million rows) to identify matched, unmatched, and orphaned entries.

A secondary, exploratory effort involved **mapping tape contents to KDM activity codes** to understand the business context of the data (which seismic surveys, countries, and activities were represented).

---

## Technology Stack

- **Language:** Python 3.10+
- **GUI Framework:** Tkinter (all tools have a desktop GUI)
- **Excel Output:** openpyxl
- **Matching Strategy:** Hash-based indexing for fast lookups against 1.7M-row HSM database; substring and exact matching depending on tool

---

## Tool Inventory and Workflow

The tools below are listed roughly in the order they were developed and used during the project. Each tool is a standalone Python script with a Tkinter GUI.

### Phase 1 — Initial Comparison (Batch 1)

| # | Script | Purpose |
|---|--------|---------|
| 1 | `compare_files.py` | **First prototype.** Compares a small tape file (~40 entries) against the large HSM list (~1.7M rows) using case-insensitive substring matching. Outputs matched and unique entries to text files. |
| 2 | `multiple_process.py` | **Batch processor (naive).** Processes all tape files in a folder against HSM. Scans every HSM line for every tape entry (O(n×m) — slow for large files). Outputs Excel with per-tape detail sheets. |
| 3 | `Tape_batch_processor.py` | **Batch processor (optimized — first match).** Same concept as above but stops at the first HSM match per tape entry, providing a significant speedup. Extracts file paths from HSM lines (strips metadata before the date). Saves incrementally after each tape. |
| 4 | `tape_batch_processor_hash.py` | **Batch processor (hash-indexed).** Builds an n-gram hash index over the HSM data for O(1) candidate lookups. 10–20× faster than linear scan. Configurable minimum character threshold to skip short/junk entries. Outputs 3-sheet Excel: Summary, All Matches, All Unique. |

### Phase 2 — UD Files Comparison (Batch 2)

| # | Script | Purpose |
|---|--------|---------|
| 5 | `ud_files_hsm_comparison.py` | **Filename-only exact matching.** Extracts everything after the last `/` from both ud_files.lst and HSM lines. Uses hash-based dict lookup for fast exact matching. |
| 6 | `ud_files_hsm_comparison_v3.py` | **Extended extraction (5 chars + filename).** Extracts last 5 characters before the final `/` plus the filename to reduce false positive matches. Filters out `drwx` directory entries, separates `bg-ingest` matches into their own sheet. |
| 7 | `ud_files_hsm_comparison_v4.py` | **Simple filename-only variant (v4).** Same as v3 but reverts to filename-only extraction for scenarios where the extra context caused mismatches. Includes drwx exclusion and bg-ingest separation. |
| 8 | `advanced_comparison.py` | **Configurable single/dual-scenario comparison.** Lets the user choose extraction method (filename only vs. N chars + filename) and run one or two scenarios side by side. Dual mode produces a difference report showing entries that matched in one scenario but not the other. |

### Phase 3 — Data Standardization and Cross-Checking

| # | Script | Purpose |
|---|--------|---------|
| 9 | `standardize.py` | **Data standardization.** Loads tape file entries (your data) and a coworker's HSM comparison results (on_hsm / not_on_hsm files) into a single Excel workbook with standardized sheets for downstream comparison. |
| 10 | `compare_with_coworker.py` | **Cross-comparison between two analysts.** Compares your standardized data against a coworker's data using hash-indexed exact matching. Identifies entries present in both, only in yours, or only in the coworker's dataset. |
| 11 | `duplicates_removal.py` | **Duplicate remover.** Scans the ALL MATCHES and ALL UNIQUE sheets in a comparison results Excel file, removes duplicate Tape Entry values (keeps first occurrence), and produces a cleaned file plus a duplicates report. |

### Phase 4 — Supporting / Exploratory Analysis

| # | Script | Purpose |
|---|--------|---------|
| 12 | `code.py` | **Russian files finder (v1).** Searches HSM database for lines containing `rus` (case-insensitive) to identify Russian-region data. Simple first pass. |
| 13 | `code_false_positives.py` | **Russian files finder (v2 — with false positive filtering).** Adds a list of known false positives (cyprus, brunei, darussalam, trust, virus, etc.) to filter out non-Russian matches. Outputs valid and excluded files separately. |
| 14 | `interactive_review.py` | **Interactive pattern review tool.** Extracts unique contextual patterns around `rus` occurrences for manual review. User marks each pattern as valid/invalid/skip via CLI. Filters the full file based on reviewed patterns. |
| 15 | `eu_ru.py` | **EU/RU path finder.** Specifically searches for the `eu/ru` path pattern in HSM data to identify European/Russian regional entries. |
| 16 | `country_code.py` | **Country code extractor.** Parses log files to extract 3-character country codes from a specific path position (after the 7th `/`). Removes duplicates, counts frequencies, moves single-occurrence codes to "Others." Exports formatted Excel report. |

### Phase 5 — KDM Activity Mapping

| # | Script | Purpose |
|---|--------|---------|
| 17 | `Activity_extraction_excel_report.py` | **Path segment extractor (basic).** Extracts the segment between the 4th and 5th `/` in each line. GUI with matched/unmatched tabs. Exports 3-sheet Excel (Summary, Matched Segments, Unmatched Lines). |
| 18 | `Activity_extraction_kdm_full.py` | **Path segment extractor + KDM matching.** Extends the basic extractor with a KDM activity file. Three-tier matching: exact → normalized (dash/underscore equivalence) → strip 2D/3D suffix. Adds KDM activity code, full name, country, and match method to the Excel report. |
| 19 | `ud_list_activity_extraction.py` | **UD list activity extractor.** Variant for UD file format — extracts the token after a HH:MM timestamp and before the next `/`. Handles country-name tokens (skips them to get the actual activity code). Same KDM matching and Excel output. |

### Utilities

| # | Script | Purpose |
|---|--------|---------|
| 20 | `text_to_excel.py` | **Text-to-Excel converter (basic).** Converts a text file (one string per line) to Excel. Uses openpyxl write-only mode for memory efficiency with 300k+ rows. |
| 21 | `text_to_excel_cleaned.py` | **Text-to-Excel converter (cleaned).** Same as above but strips metadata before the HH:MM timestamp and skips `drwx` directory entries. |
| 22 | `file_comparator_nas_treelisting_batch_2_v1.py` | **Three-way file comparator.** Compares NAS-to-HSM file list against both a treelisting (Batch 1) and a files_log (Batch 2). Categorizes entries as matched in both, matched in one only, or unmatched in either. Outputs a 4-sheet Excel report. |

---

## Typical Workflow

```
1. Receive tape file listings (one file per tape, or consolidated)
         │
2. Convert raw text to Excel if needed (text_to_excel)
         │
3. Run batch comparison against HSM database
   ├── Batch 1 (28 tapes): tape_batch_processor_hash.py
   └── Batch 2 (300+ tapes): ud_files_hsm_comparison (v3 or v4)
         │
4. Remove duplicates from results (duplicates_removal.py)
         │
5. Cross-check with coworker's results
   ├── Standardize both datasets (standardize.py)
   └── Compare (compare_with_coworker.py)
         │
6. Investigate specific regions/patterns as needed
   ├── Russian data: code.py → code_false_positives.py → interactive_review.py
   ├── EU/RU paths: eu_ru.py
   └── Country codes: country_code.py
         │
7. Map tape contents to KDM activities (exploratory)
   ├── Activity_extraction_kdm_full.py (for standard path format)
   └── ud_list_activity_extraction.py (for UD timestamp format)
         │
8. Final reconciliation — confirm all tape data is in HSM
```

---

## Input / Output Summary

**Inputs:**
- Tape file listings (`.txt`, one file path per line, per tape)
- HSM database export (~1.7M rows, file metadata + paths)
- KDM activity reference file (optional, for activity mapping)
- Coworker's comparison results (for cross-checking)

**Outputs:**
- Excel reports (`.xlsx`) with color-coded sheets: Summary, Matches, Unique/Unmatched, and tool-specific extras (bg-ingest, drwx, duplicates, country codes, KDM activity mappings)

---

## Dependencies

```
Python >= 3.10
openpyxl    (Excel read/write)
tkinter     (GUI — included with standard Python on most systems)
```

Install openpyxl if not already available:

```bash
pip install openpyxl
```

---

## Known Limitations

- **Matching accuracy depends on extraction method.** Filename-only matching can produce false positives when different files share the same name. The N-chars-before-slash method reduces this but can miss matches if path structures differ between tape and HSM listings.
- **HSM line parsing assumes a specific format.** The date-detection heuristic (looking for a 4-digit year) may fail on unusual HSM export formats.
- **Memory usage.** The hash-indexed tools load the full HSM dataset into memory. For the ~1.7M row HSM file this is manageable, but significantly larger files may require chunked processing.
- **GUI is Tkinter-based.** Functional but not polished — designed for analyst use, not end-user distribution.
- **Some scripts contain duplicated code** across versions (v3, v4, advanced). This reflects iterative development as matching requirements evolved during the project.

---

## Project Status

**Complete.** All tape data has been reconciled against the HSM database. The physical tapes can be decommissioned based on the reconciliation results documented in the output Excel reports.

---

*Last updated: March 2026*
