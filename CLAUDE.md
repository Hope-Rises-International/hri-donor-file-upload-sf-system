# Claude Code — Project Instructions

## About this project

**hri-donor-file-upload-sf-system** — Aegis Non Donor CSV pipeline for Hope Rises International.

Replaces the CoWork browser-automation workflow that processes Aegis (Moore DM Group) Non Donor CSV files. The CoWork version navigated Chrome to paste data into Google Sheets — fragile and slow. This pipeline runs as Python, writes to Google Sheets via API, and produces clean output files with zero browser dependency.

- **Phase A (this build):** Local CSV → Python triage → Kill List Google Sheet (API write) + cleaned Non Donor CSV output
- **Phase B (future):** Cloud Run + Cloud Scheduler → SFTP pull → auto-triage → email notification
- **Phase C (future):** Salesforce REST API push → replace Data Loader entirely

**Kill List Sheet ID:** `11dM2Pf-E195rJUnF79rMHhN5RUb0L-03fS8nZ4WZw7o`
**GCP Project:** `gmail-agent-489116`
**Service Account:** `hri-automation@gmail-agent-489116.iam.gserviceaccount.com`

### Authentication

Uses Application Default Credentials (ADC) via service account impersonation:
```bash
gcloud auth application-default login \
  --client-id-file="$HOME/Downloads/hri-oauth-client.json" \
  --scopes="https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/drive,https://www.googleapis.com/auth/spreadsheets"
```
ADC credentials at: `~/.config/gcloud/application_default_credentials.json`

## Stack Learnings (canonical source)

Stack-level learnings live in ONE place:
- Repo: `Hope-Rises-International/hri-template-repository`
- File: `hri-stack-learnings.md`
- Read before any infrastructure, auth, deployment, or tooling work.
- Update directly via GitHub API when you discover a stack-level gotcha. See session-end protocol below.

Do NOT create a local `learnings.md` or `hri-stack-learnings.md` in this repo. If one exists, merge any unique content upstream and delete the local copy.

## Project knowledge

**[2026-03-13 | Bill | Initial repo setup and build plan]**
- **Decided:** Phase A build plan finalized from Aegis_NonDonor_Pipeline_Build_Spec. Python script with Google Sheets API integration. No browser automation.
- **Changed:** Created repo from template, added build spec, wrote Phase A implementation plan.
- **Watch out:** Before building, need sample Non Donor CSV files from Aegis to validate column headers and triage logic. Service account needs editor access on Kill List sheet (ID: `11dM2Pf-E195rJUnF79rMHhN5RUb0L-03fS8nZ4WZw7o`).
- **Open:** Phase A build not started — plan is ready, need sample CSV files and SA sheet access verified before coding begins. See `docs/build-plan.md` for the full implementation plan.

---

## Session Start

Before beginning work, check the most recent Project Knowledge entry above. If it contains an "Open" field, surface the open items to the user and confirm whether to continue that work or start something new.

---

## Session-End Protocol

**The full protocol lives in one place:** `session-end-protocol.md` in `hri-template-repository`.

At session close, fetch and follow it:

```bash
gh api /repos/Hope-Rises-International/hri-template-repository/contents/session-end-protocol.md \
  --jq '.content' | base64 -d > /tmp/session-end-protocol.md
```

Then read `/tmp/session-end-protocol.md` and execute all steps.

This ensures every repo uses the latest protocol without needing per-repo updates.
