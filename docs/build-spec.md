# Aegis Non Donor Pipeline — Build Specification

**Owner:** Bill Simmons, CEO — Hope Rises International
**Target:** Claude Code build
**Date:** March 13, 2026
**Status:** Phase A — Proof of Concept

---

## Purpose

Replace the current CoWork browser-automation workflow for processing Aegis (Moore DM Group) Non Donor CSV files. The CoWork version navigates Chrome to paste data into Google Sheets — fragile and slow. This pipeline runs as Python, writes to Google Sheets via API, and produces clean output files with zero browser dependency.

This spec covers **Phase A only** — manual CSV input, automated triage, Sheets API write to Kill List, and file output. Phases B and C are documented below for architectural context but are separate builds.

---

## Pipeline Overview (All Phases)

```
Phase A (this build):
  Local CSV files → Python triage → Kill List Google Sheet (API write)
                                   → Cleaned Non Donor CSV (file output)

Phase B (future):
  Cloud Scheduler → Cloud Run → SFTP pull from Aegis FTP
                               → Phase A logic runs automatically
                               → Email notification to Bekah

Phase C (future):
  Bekah reviews staging Google Sheet → Approves
                                     → Cloud Run pushes to Salesforce via REST API
                                     → Confirmation email
```

---

## Prerequisites for Phase A

Before Claude Code begins the build:

1. **Sample Non Donor CSV files** — one or two recent files from Aegis so CC can validate triage logic against real data and confirm column headers match this spec.
2. **Google Sheets API access** — the service account (`hri-automation@gmail-agent-489116.iam.gserviceaccount.com`) must have editor access on the Kill List sheet. CC should verify by reading a cell from the sheet as its first step. If permission error, Kelly needs to share the sheet with the service account email.
3. **Service account impersonation configured** — the developer running CC must have completed `gcloud auth application-default login --impersonate-service-account hri-automation@gmail-agent-489116.iam.gserviceaccount.com` per the Service Account Migration spec.

---

## Phase A: Detailed Specification

### Input

Two CSV files downloaded from the Aegis FTP site. Both are the same column format:

| Column | Description |
|--------|-------------|
| CONSID | Constituent ID (donor identifier in Salesforce) |
| FINDER | Finder number (matchback/acquisition identifier) |
| FNAME1 | First name (primary) |
| LNAME1 | Last name (primary) |
| FNAME2 | First name (secondary/spouse) |
| LNAME2 | Last name (secondary/spouse) |
| TITLE1 | Salutation (primary) |
| TITLE2 | Salutation (secondary) |
| SUFFX1 | Suffix (primary) |
| SUFFX2 | Suffix (secondary) |
| STREET | Street address |
| CITY | City |
| STATE | State |
| ZIPCOD | Zip code |
| COUNTRY | Country |
| PHONE | Phone number |
| EMAIL | Email address |
| SRCCDE | Appeal/source code |
| DNRAMT | Donation amount |
| DNRDDT | Deposit date |
| CONDEC | Deceased flags |
| CONMAI | Mail preference flags |
| CONOPT | Opt-out flags |
| TRFLAG | Transaction flags |
| TRACK | Acknowledgement preference |
| TRCHK# | Check reference number |
| TRPTYP | Payment method |
| TRMBID | Cager record note |

**File naming convention:** Files containing "Non Donors" in the filename are Non Donor files. The other file is the Household (donation) file. Phase A processes **only** Non Donor files.

### Processing Logic

**This is the core triage.** All CSV columns should be read as strings (no type inference).

#### Step 1: Sort by FINDER

Sort the entire dataframe by FINDER column, empty values last.

#### Step 2: Reclassify known donor FINDERs

Some FINDER values are actually donor constituent IDs that belong in the CONSID column. Identify them by prefix:

- FINDER starts with `0` → move to CONSID
- FINDER starts with `7` → move to CONSID
- FINDER starts with `S` → move to CONSID

For matching rows: copy FINDER value to CONSID, then clear FINDER.

#### Step 3: Re-sort by FINDER

Sort again by FINDER. Empty FINDER values go to the bottom.

#### Step 4: Split into two sets

- **Kill List rows:** FINDER is non-empty after Steps 1-3. These are acquisition contacts requesting mail suppression.
- **Salesforce-bound rows:** FINDER is empty. These are existing donors with data updates (deceased, address change, opt-out, comments, white mail).

#### Step 5: Format Kill List output

From Kill List rows, produce a dataset with these columns in this order:

| Output Column | Source Column |
|---------------|--------------|
| First Name | FNAME1 |
| Last Name | LNAME1 |
| Street 1 | STREET |
| Street 2 | *(empty — new column)* |
| City | CITY |
| State | STATE |
| Zip | ZIPCOD |
| Suppress Date | Today's date as `m/d/yy` |

Also produce a full Kill List CSV preserving all original columns **except** SUFFX1 (drop it), and **adding** a STREET2 column (empty) after STREET. Save as `{original_filename}_Master Kill List.csv`.

#### Step 6: Format Salesforce-bound output

Save the Salesforce-bound rows as a cleaned CSV with all original columns intact. Use the original filename.

### Google Sheets Integration

**Kill List Google Sheet ID:** `11dM2Pf-E195rJUnF79rMHhN5RUb0L-03fS8nZ4WZw7o`

**Sheet name:** First sheet (gid=0)

**Columns (A through H):**
| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| First Name | Last Name | Street 1 | Street 2 | City | State | Zip | Suppress Date |

**Write method:** Append rows after the last populated row. Use the Google Sheets API (`gspread` with ADC credentials or `googleapiclient`). Do NOT use browser navigation.

**Important:** The sheet has existing data. Find the true last row with data, then append starting at the next row. Do not overwrite existing rows.

### File I/O

#### Input folder
```
Claude Non Donor Files/
```

#### Output structure
```
Claude Non Donor Files/
├── Claude Files to Delete/              ← Original input files archived here
├── Claude Processed Non Donor Files/    ← Cleaned SF-bound CSVs land here
│   └── Uploaded NonDonor Files/         ← Master Kill List CSVs moved here after Sheet write
```

#### File flow
1. Read `*Non Donors*.csv` from input folder
2. Process per logic above
3. Move original input file → `Claude Files to Delete/`
4. Save cleaned SF-bound CSV → `Claude Processed Non Donor Files/`
5. Save Master Kill List CSV → `Claude Processed Non Donor Files/`
6. Write Kill List rows to Google Sheet via API
7. Move Master Kill List CSV → `Uploaded NonDonor Files/`

### Output Report

After processing, print a summary:
- Number of files processed
- Per file: rows to Kill List vs. rows kept for Salesforce
- Google Sheet: total rows appended, starting and ending row numbers
- File locations for all outputs

---

## Salesforce Field Mapping (Reference for Phase C)

Both Household and Non Donor files target the same Salesforce object: `npsp__DataImport__c` (NPSP Data Import). Operation: **Insert**.

| CSV Column | Salesforce API Field |
|------------|---------------------|
| Batch Name | `Batch_Name__c` |
| CITY | `Constituent_City__c` |
| CONDEC | `DM_Deceased_Flags__c` |
| CONMAI | `DM_Mail_Preference_Flags__c` |
| CONOPT | `DM_Opt_Out_Flags__c` |
| CONSID | `Inbound_Constituent_Id__c` |
| COUNTRY | `Constituent_Country__c` |
| DNRAMT | `npsp__Donation_Amount__c` |
| DNRDDT | `DM_Deposit_Date__c` |
| Donation Donor | `npsp__Donation_Donor__c` |
| Donation Gift Source | `Donation_Gift_Source__c` |
| EMAIL | `npsp__Contact1_Personal_Email__c` |
| FINDER | `Finder_Number__c` |
| FNAME1 | `npsp__Contact1_Firstname__c` |
| FNAME2 | `npsp__Contact2_Firstname__c` |
| LNAME1 | `npsp__Contact1_Lastname__c` |
| LNAME2 | `npsp__Contact2_Lastname__c` |
| PHONE | `npsp__Contact1_Home_Phone__c` |
| SRCCDE | `Appeal_Code__c` |
| STATE | `Constituent_State__c` |
| STREET | `Constituent_Street__c` |
| SUFFX1 | `Contact1_Suffix__c` |
| SUFFX2 | `Contact2_Suffix__c` |
| TITLE1 | `npsp__Contact1_Salutation__c` |
| TITLE2 | `npsp__Contact2_Salutation__c` |
| TRACK | `DM_Acknowledgement_Preference__c` |
| TRCHK# | `npsp__Payment_Check_Reference_Number__c` |
| TRFLAG | `DM_Transaction_Flags__c` |
| TRMBID | `Cager_Record_Note__c` |
| TRPTYP | `npsp__Payment_Method__c` |
| ZIPCOD | `Constituent_Postal_Code__c` |

**Source:** `Aegis_Import_Mapping.sdl` (Data Loader mapping file, dated November 16, 2023)

---

## Phase B: SFTP Automation (Future Build)

**Objective:** Eliminate manual FTP download. Cloud Run job pulls files automatically.

### Architecture
- **Runtime:** Cloud Run service on `gmail-agent-489116` GCP project — single service handles both Phase B and Phase C via separate endpoints
- **Endpoints:** `/pull` (triggered by Cloud Scheduler daily) and `/push` (triggered by Bekah's approval action)
- **Schedule:** Cloud Scheduler triggers `/pull` daily (time TBD based on Aegis posting schedule)
- **SFTP credentials:** Stored in Secret Manager
- **Service account:** Runs as `hri-automation@gmail-agent-489116.iam.gserviceaccount.com`
- **Repository:** One repo for this entire service — all phases live together
- **Flow:** Connect to Aegis SFTP → download both files → run Phase A triage logic on Non Donor file → write Kill List to Google Sheet → write SF-bound records to a staging Google Sheet → send email notification to Bekah via Gmail API

### Household File Handling (Phase B)
The Household (donation) file downloads alongside the Non Donor file. In Phase B, it passes through untouched — either saved to a designated Google Drive folder for Bekah's existing Data Loader workflow, or written to its own staging sheet tab for review. The Household file does not require triage; it is already formatted for direct insert. When the Non Donor flow is proven stable, add Household to the `/push` endpoint with its own approval step.

### Staging Sheet
A new Google Sheet (to be created) where Bekah reviews Non Donor records before they go to Salesforce. Columns match the `npsp__DataImport__c` field mapping. Each row shows the action type (deceased, address change, opt-out, etc.) derived from the flag fields.

### Notification Email
Sent to Bekah after processing. Contains:
- File date and record counts
- Link to Kill List sheet (for awareness, already written)
- Link to staging sheet (requires her review and approval)

---

## Phase C: Salesforce Push (Future Build)

**Objective:** Replace Data Loader entirely. Approved records go to Salesforce via REST API.

### Architecture
- **Trigger:** Bekah approves in staging sheet (custom menu button or checkbox column, Apps Script `onEdit` trigger fires webhook)
- **Endpoint:** Cloud Run `/push` endpoint on same service as Phase B (single repo, single deployment)
- **Operation:** Insert records to `npsp__DataImport__c` using Salesforce REST API SObject Collections endpoint (up to 200 records per request — sufficient for daily volumes)
- **Do NOT use Bulk API 2.0** — it is async/job-based and designed for thousands+ records; overkill for this volume
- **Confirmation:** Email to Bekah with insert results (success count, any failures with row detail)
- **Cleanup:** Mark staging sheet rows as uploaded with timestamp

### BDI Processing Remains Manual
Inserting records to `npsp__DataImport__c` does NOT automatically create or update Contacts/Accounts. NPSP's Batch Data Import (BDI) must run to process the Data Import records into matched Contacts, Accounts, and Opportunities. **Bekah triggers BDI manually in the NPSP Data Import UI after the API insert completes.** This preserves her ability to review matching results and catch errors before records are committed. Automating BDI is technically possible (`BDI_DataImport_API` Apex class via REST) but is explicitly deferred — do not add it to this build.

### Batch Naming Convention
The `Batch_Name__c` field on each `npsp__DataImport__c` record uses this format:

- Non Donor files: `ALMMMDDYYYY Non Donors` (e.g., `ALM03122026 Non Donors`)
- Household files: `ALMMMDDYYYY Households` (e.g., `ALM03122026 Households`)

The date is the processing date. "ALM" is the legacy prefix (American Leprosy Missions).

### Household File (Phase C)
When the Non Donor Salesforce push is stable, extend the `/push` endpoint to handle Household file records. Same field mapping, same insert operation to `npsp__DataImport__c`, separate staging sheet tab, separate approval step. Batch name format: `ALMMMDDYYYY Households` (e.g., `ALM03122026 Households`).

---

## Technical Notes

- **All CSV columns read as strings.** No type inference — zip codes with leading zeros, numeric-looking IDs, and flag fields must all be preserved as-is.
- **Google Sheets API auth:** Use Application Default Credentials (ADC) via service account impersonation. The developer authenticates with their `@hoperises.org` account and impersonates `hri-automation@gmail-agent-489116.iam.gserviceaccount.com`. The service account must have editor access on the Kill List sheet. See `Service_Account_Migration_CC_Spec.md` for setup details.
- **Repository:** Create new repo under `bsimmons3rd` org following HRI repo conventions (CLAUDE.md, standard structure). The CLAUDE.md must include the service account authentication block per the migration spec.
- **No browser automation.** The entire point of this build is to eliminate the CoWork Chrome navigation pattern.
- **Date format for Kill List:** The Suppress Date column uses `m/d/yy` format (e.g., `3/12/26`). Google Sheets may auto-convert this to its internal date format — that is expected and correct.

---

## Context: Record Types in Non Donor File

For clarity on what the triage is separating (from process owner transcript):

1. **Existing donors with updates** — Have a CONSID (or a FINDER that gets reclassified to CONSID in Step 2). These carry flag updates: deceased (`CONDEC`), mail preference (`CONMAI`), opt-out (`CONOPT`), address changes, comments. They route to Salesforce.

2. **Acquisition suppressions** — Have a FINDER starting with `3` or other acquisition prefixes. These are people from acquisition mailings requesting no further contact. They route to the Kill List for suppression against future acquisition drops.

3. **White mail** — Envelopes returned without a scannable reply device. Identified by appeal codes starting with `A` or `M` in SRCCDE. If the FINDER can be resolved to a donor ID (starts with `0`, `7`, or `S`), it gets reclassified in Step 2. Remaining white mail with unresolvable FINDERs goes to the Kill List.

The triage logic in Steps 1-4 handles all three cases through the FINDER prefix routing. No separate classification step is needed — the prefix-based split produces the correct result for each record type.
