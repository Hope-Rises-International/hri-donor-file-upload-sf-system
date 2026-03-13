# Phase A — Build Plan

## Overview

Build a Python CLI tool that processes Aegis Non Donor CSV files: triages records into Kill List (acquisition suppressions) and Salesforce-bound (existing donor updates), writes Kill List to Google Sheets, and outputs cleaned CSV files.

## Prerequisites (verify before building)

1. **Sample Non Donor CSV files** — at least one recent file from Aegis FTP to validate column headers and triage logic
2. **Kill List sheet access** — service account `hri-automation@gmail-agent-489116.iam.gserviceaccount.com` must have editor access on sheet `11dM2Pf-E195rJUnF79rMHhN5RUb0L-03fS8nZ4WZw7o`
3. **ADC configured** — developer must have run `gcloud auth application-default login` with spreadsheets scope

## File Structure

```
hri-donor-file-upload-sf-system/
├── CLAUDE.md
├── docs/
│   ├── build-plan.md              ← This file
│   └── build-spec.md              ← Original spec from Bill
├── src/
│   └── process_non_donors.py      ← Main processing script
├── requirements.txt               ← google-api-python-client, pandas
└── Claude Non Donor Files/        ← Runtime I/O folder (gitignored)
    ├── Claude Files to Delete/
    ├── Claude Processed Non Donor Files/
    │   └── Uploaded NonDonor Files/
```

## Build Steps

### Step 1: Environment Setup
- Create `requirements.txt` with: `pandas`, `google-api-python-client`, `google-auth`
- Create `.gitignore` — exclude `Claude Non Donor Files/`, `__pycache__/`, `*.pyc`, `.env`
- Verify ADC credentials work by reading a cell from the Kill List sheet

### Step 2: CSV Ingestion
- Scan `Claude Non Donor Files/` for files matching `*Non Donors*.csv`
- Read all columns as strings (`dtype=str`) — critical for zip codes, IDs, flag fields
- Validate expected column headers match spec (CONSID, FINDER, FNAME1, LNAME1, etc.)

### Step 3: Triage Logic
- **Sort** by FINDER column (empty values last)
- **Reclassify** FINDER → CONSID where FINDER starts with `0`, `7`, or `S`
  - Copy FINDER value to CONSID, clear FINDER
- **Re-sort** by FINDER (empty values last)
- **Split:**
  - FINDER non-empty → Kill List rows
  - FINDER empty → Salesforce-bound rows

### Step 4: Kill List Output
- Build Kill List dataset with 8 columns: First Name, Last Name, Street 1, Street 2 (empty), City, State, Zip, Suppress Date (today as `m/d/yy`)
- Save Master Kill List CSV (all original columns minus SUFFX1, plus empty STREET2 after STREET) as `{original_filename}_Master Kill List.csv`

### Step 5: Google Sheets Write
- Authenticate via ADC (`google.oauth2.credentials.Credentials`)
- Find last populated row in Kill List sheet (sheet ID: `11dM2Pf-E195rJUnF79rMHhN5RUb0L-03fS8nZ4WZw7o`, first tab)
- Append Kill List rows (8-column format) starting at next empty row
- Do NOT overwrite existing data

### Step 6: Salesforce-Bound Output
- Save SF-bound rows as cleaned CSV with all original columns
- Use original filename

### Step 7: File Movement
1. Move original input file → `Claude Files to Delete/`
2. Save SF-bound CSV → `Claude Processed Non Donor Files/`
3. Save Master Kill List CSV → `Claude Processed Non Donor Files/`
4. After successful Sheets write, move Master Kill List CSV → `Uploaded NonDonor Files/`

### Step 8: Summary Report
Print to console:
- Number of files processed
- Per file: Kill List row count vs. Salesforce-bound row count
- Google Sheet: rows appended, starting/ending row numbers
- File output locations

## Key Constraints

- **All CSV columns as strings** — no type inference (zip codes, numeric IDs, flags)
- **No browser automation** — entire point is replacing CoWork's Chrome navigation
- **Append only** to Kill List sheet — never overwrite existing rows
- **Date format:** Suppress Date uses `m/d/yy` (e.g., `3/13/26`)
- **SUFFX1 dropped** from Master Kill List CSV output
- **STREET2 column added** (empty) after STREET in Master Kill List CSV

## Testing Plan

1. Verify ADC auth by reading Kill List sheet
2. Process a sample CSV — check Kill List vs SF-bound split counts
3. Verify FINDER reclassification (prefixes 0, 7, S → CONSID)
4. Confirm Kill List rows append correctly to Google Sheet
5. Confirm file movement to correct output folders
6. Edge cases: empty FINDER, all-Kill-List file, all-SF-bound file
