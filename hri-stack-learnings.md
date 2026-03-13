# HRI Stack Learnings

> **This is the single source of truth for stack-level gotchas across all HRI repos.**
> Every Claude Code session should read this file before infrastructure work.
> Update this file directly (via GitHub API) when you discover a stack-level learning.
> Do NOT create local copies of this file in project repos.

Shared knowledge base of stack-level gotchas and fixes discovered across all HRI projects. Every Claude Code user in the org should have access to this file.

**Scope:** Only infrastructure, tooling, and platform issues. Not project-specific logic or business rules.

---

## macOS / Terminal

### npm cache owned by root
Some directories under `~/.npm/_cacache` end up owned by `root` from a past `sudo npm install`. Symptom: `EACCES: permission denied` on `npm install`.
- **Workaround:** `npm install --cache /tmp/npm-cache`
- **Permanent fix:** `sudo chown -R $(whoami) ~/.npm`

### Python 3.9 vs 3.13 PATH priority
macOS system Python 3.9 takes priority even with 3.13 installed via Homebrew. Google API libraries throw deprecation warnings on 3.9 but still work.
- **Fix:** Add to `~/.zshrc`: `export PATH="/opt/homebrew/bin:$PATH"`

### gcloud "Python 3.9 is no longer supported"
gcloud SDK uses the system Python by default.
- **Fix:** Add to `~/.zshrc`: `export CLOUDSDK_PYTHON=/opt/homebrew/bin/python3`

---

## Google Cloud / Authentication

### ADC scopes must match intent
Application Default Credentials are scoped at auth time. `drive.readonly` silently blocks uploads — no error until the API call. Current working scopes: `cloud-platform` + `drive` (full).
- **Re-auth command:** `gcloud auth application-default login --scopes="https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/drive"`
- **Credentials location:** `~/.config/gcloud/application_default_credentials.json`

### Shared drive API calls need two flags
Without these flags, shared drive files are completely invisible to the API:
```python
service.files().list(
    supportsAllDrives=True,
    includeItemsFromAllDrives=True,
    ...
)
```
This applies to every operation: list, get, create, update, delete.

### Use HRI custom OAuth client for ADC — not gcloud default
Google has blocked (or is blocking) the `drive` scope with gcloud's default client ID. New team members will get Drive access denied if they use the default `gcloud auth application-default login`.
- **Fix:** Use the HRI custom OAuth client file (`hri-oauth-client.json`, stored in `claude-repo` shared Drive folder):
```bash
gcloud auth application-default login \
  --client-id-file="$HOME/Downloads/hri-oauth-client.json" \
  --scopes="https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/drive"
```
- **Note:** OAuth client creation cannot be done via API — requires GCP Console UI.
- **Client lives in:** GCP project `gmail-agent-489116`, OAuth consent screen is Internal (hoperises.org only).

---

## Google Apps Script

### All .gs files share one global namespace
Every `.gs` file in a GAS project compiles into the same global scope. Top-level `const`, `let`, and `function` declarations must be unique across ALL files — a duplicate name causes `SyntaxError: Identifier already declared` at page load, which silently breaks the entire web app.
- **Fix:** Keep shared constants in one file (e.g., `Config.gs`) and reference them everywhere else. Never redeclare them in engine files.
- **Symptom:** Web app loads but a tab or render function silently fails with `renderXxxView is not defined` in the console — caused by a SyntaxError earlier in the same script block preventing parsing.

### /dev vs /exec URLs — testing after clasp push
`clasp push` updates the project source immediately, but only the `/dev` URL serves the latest pushed code. The `/exec` URL serves the last *deployed* version and requires `clasp deploy` to update.
- **Get the /dev URL from:** Apps Script editor → Deploy → Test deployments. Do not manually swap `/exec` → `/dev` in an old URL — the subdomain can differ.
- **Browser caching:** The GAS iframe aggressively caches. Use DevTools with "Disable cache" checked, or open in Incognito, when testing after a push.

### Session.getActiveUser() fails in time-based triggers
Time-based triggers have no "active user." The call throws: `Exception: You do not have permission to call Session.getActiveUser`.
- **Fix:** Use `Session.getEffectiveUser()` instead (returns the trigger owner's email).
- **Also required:** Add `https://www.googleapis.com/auth/userinfo.email` to the `oauthScopes` array in `appsscript.json`.

### clasp deploy version flag is `-V`, not `--version`
When updating an existing deployment to a specific version, use the short flag `-V`:
```bash
clasp deploy -i DEPLOYMENT_ID -V 6
```
The long form `--version 6` silently fails (deployment doesn't update). Always use `-V`.

### clasp push does NOT update the live web app
`clasp push` updates the project source code but the deployed web app URL still serves the old version. You must also create or update a deployment.
- **Fix:** Run `clasp push && clasp deploy` (or update an existing deployment ID).

### POST responses redirect (302)
Apps Script web apps return a 302 redirect on POST. A simple `curl -X POST` gets an empty HTML page, not your JSON response.
- **Fix:** Two-step curl pattern:
```bash
REDIRECT_URL=$(curl -s -o /dev/null -w "%{redirect_url}" -X POST "$URL" \
  -H "Content-Type: application/json" -d "$PAYLOAD") && curl -s -L "$REDIRECT_URL"
```

### API key must go in JSON body, not HTTP headers
Apps Script web apps don't reliably receive custom HTTP headers from all callers. The Sheets Bridge expects `apiKey` as a field in the JSON body.

### Sheets Bridge column names are case-sensitive
`"repository"` ≠ `"Repository"`. The API returns `"action": "error"` with a message, but it's easy to miss. Always copy column names exactly from the CLAUDE.md reference.

---

## GitHub CLI

### gh repo list: use "visibility" not "private"
`gh repo list --json name,private` crashes — the field doesn't exist. Use `--json name,visibility` instead.

### Default branch isn't always main
Some repos have feature branches as their default (e.g., `hri-receipt-generator` defaults to `claude/add-receipt-generator-KPX7y`). Never assume `main`.
- **Check:** `git remote show origin | grep "HEAD branch"`

### GitHub org creation cannot be done via API
Must use the web UI at https://github.com/settings/organizations. Repo transfers CAN be done via API: `gh api /repos/{owner}/{repo}/transfer --method POST --field new_owner="Hope-Rises-International"`.

### GitHub nonprofit Teams plan is free
Apply at https://github.com/solutions/industry/nonprofits. Takes up to a week to process. Start on free plan, apply for upgrade.

---

## Frontend / Portable Builds

### file:// protocol blocks ES module loading
Browsers refuse to load `<script type="module">` from `file://` URLs due to CORS policy. The page renders blank with no visible error (check browser console).
- **Fix option 1:** Serve locally: `python3 -m http.server 8080`
- **Fix option 2:** Build as single file using `vite-plugin-singlefile` — inlines all JS and CSS into one HTML file.

### Use HashRouter for portable React apps
`BrowserRouter` requires a server to handle routes. `HashRouter` works everywhere — local files, GitHub Pages, Google Drive.
```tsx
import { HashRouter } from 'react-router-dom';
```

### Single-file HTML for Drive and non-technical users
When building artifacts that need to live in Google Drive or be shared outside the dev team, bundle everything into one HTML file. Install `vite-plugin-singlefile`, set `base: './'` in vite config, build. The output is a single `index.html` with all assets inlined.

---

## Google Sheets API

### New tabs have a 1000-row grid limit — expand before writing large datasets

When you create a new spreadsheet tab (via `addSheet`), Sheets provisions it with a default grid of 1000 rows × 26 columns. Writing data beyond row 1000 returns HTTP 400: `Range exceeds grid limits`.

**Fix:** Before batch-writing rows, call `appendDimension` to expand the grid:

```python
sheets.spreadsheets().batchUpdate(
    spreadsheetId=sheet_id,
    body={"requests": [{
        "appendDimension": {
            "sheetId": tab_sheet_id,
            "dimension": "ROWS",
            "length": needed_rows - current_rows + 100,
        }
    }]},
).execute()
```

Or set `rowCount` when creating the tab:
```python
{"addSheet": {"properties": {"title": "my_tab", "gridProperties": {"rowCount": 50000}}}}
```

**Discovered:** 2026-03-13, hri-gl-data-repo — trial_balance tab needed ~7,000 rows.

---

## Google Drive API

### Shared drive roles affect API operations
Contributor role can create files but may not delete them. Content Manager can create, delete, and organize. If a Drive API `files().delete()` returns 404 on a shared drive, check the user's role — it's likely Contributor, not Content Manager.

### Upload to shared Drive folder
```python
service.files().create(
    body={'name': 'filename', 'parents': ['FOLDER_ID']},
    media_body=MediaFileUpload(path, mimetype=mime),
    supportsAllDrives=True,
    fields='id, name'
).execute()
```
The `supportsAllDrives=True` flag is required even on create, not just list/get.

### claude-repo shared Drive folder
Folder ID: `1iq-yqA0us2y3epYvbMstF_apdVV-O57e`. Used for sharing artifacts between Claude Code users and with non-technical staff. Upload here when user says "put that in the shared drive."
