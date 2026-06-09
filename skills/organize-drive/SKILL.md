---
name: organize-drive
description: >-
  Organize Google Drive files by moving them into folders based on configurable
  rules (MIME type, name pattern, age). Three modes: PLAN (dry-run showing all
  proposed moves before touching anything), APPLY (execute the moves, log to
  audit Sheet), UNDO (reverse from audit log). Light on tokens — uses Drive API
  search/move calls directly, never reads file contents. Builds on the Google
  Workspace MCP (search_drive_files, list_drive_items, create_drive_folder,
  update_drive_file). Trigger phrases: "organize my Google Drive", "sort my
  Drive files", "move Drive files into folders", "clean up my Drive", "archive
  old files in Drive", "auto-sort Drive by type", "organize Drive folder".
---

# Organize Google Drive

Rule-based file mover for Google Drive. Dry-run first, execute second, undo
any time. Never reads file content — all operations use metadata + Drive API
move calls, so token cost is proportional to the number of files found, not
their size.

**Three modes:**
- **PLAN** — query Drive, compute proposed moves, present the list, stop.
- **APPLY** — execute the plan, create missing folders, log every move to the
  audit Sheet.
- **UNDO** — read the audit Sheet, reverse the last run.

---

## Step 1 — Establish the scope

Ask the user (or infer from context):

1. **Source folder** — the Drive folder to organize (name or ID). Defaults to
   root ("My Drive") if not specified. Be explicit — never silently recurse the
   entire Drive.
2. **Recurse?** — organize only the top level, or also sub-folders?
   Default: top level only.
3. **Rules** — see rule types below. If the user just says "organize by type",
   use the default MIME-type ruleset.
4. **Dry-run or apply?** — default to PLAN; require explicit confirmation before
   APPLY.

---

## Rule types

Rules are evaluated in order; first match wins.

### MIME-type rules (most common)
Move files whose MIME type matches into a named subfolder.

| Target folder | MIME types to catch |
|---|---|
| `Docs/` | `application/vnd.google-apps.document`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| `Sheets/` | `application/vnd.google-apps.spreadsheet`, `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` |
| `Slides/` | `application/vnd.google-apps.presentation`, `application/vnd.ms-powerpoint`, `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| `PDFs/` | `application/pdf` |
| `Images/` | `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/svg+xml`, `image/heic` |
| `Videos/` | `video/mp4`, `video/quicktime`, `video/avi`, `video/webm` |
| `Forms/` | `application/vnd.google-apps.form` |
| `Scripts/` | `application/vnd.google-apps.script` |
| `Shortcuts/` | `application/vnd.google-apps.shortcut` |
| `Archives/` | `application/zip`, `application/x-tar`, `application/gzip` |

### Name-pattern rules
Move files whose name matches a regex or glob.

Examples:
- `"Invoice*"` → `Finance/Invoices/`
- `"*receipt*"` (case-insensitive) → `Finance/Receipts/`
- `"Screenshot*"` → `Screenshots/`
- `"*backup*"` → `Archive/Backups/`

### Age rules
Move files not modified in N days.

Examples:
- Older than 365 days → `Archive/Old/`
- Older than 90 days AND is a PDF → `Archive/Old-PDFs/`

### Combined rules
Rules can combine type + pattern + age. First match wins — put specific rules
before broad ones.

---

## Step 2 — PLAN mode (always run this first)

Search the source folder with `search_drive_files`. For each rule, build a Drive
query string and call the tool. Aggregate results, deduplicate, compute the
destination for each file.

### Query patterns

**By MIME type in a folder:**
```
mimeType='application/pdf' and '<folder_id>' in parents and trashed=false
```

**By name pattern:**
```
name contains 'Invoice' and '<folder_id>' in parents and trashed=false
```

**By age (not modified in 365 days):**
```
modifiedTime < '2024-01-01T00:00:00Z' and '<folder_id>' in parents and trashed=false
```

**Exclude subfolders from results:**
```
mimeType != 'application/vnd.google-apps.folder' and '<folder_id>' in parents and trashed=false
```

### Present the plan as a table

```
PLAN — 47 files to move  (source: My Drive root, top-level only)

  Docs/          12 files   Q1 Report.docx, Project Brief.doc, ...
  Sheets/         8 files   Budget 2024.xlsx, Tracker.gsheet, ...
  PDFs/          15 files   Invoice-001.pdf, Receipt-Mar.pdf, ...
  Images/         7 files   logo.png, banner.jpg, ...
  Archive/Old/    5 files   (not modified since 2022)

  SKIP (already in subfolder): 0
  SKIP (folder): 3 folders left in place

Proceed with APPLY? (yes / no / adjust rules)
```

Folders already in the correct place → skip silently.
Files that match no rule → leave in place, list them separately as "unmatched".

---

## Step 3 — APPLY mode

Execute only after the user confirms the plan.

### For each destination folder in the plan:
1. Check if the folder already exists: `list_drive_items` filtered by name +
   `mimeType='application/vnd.google-apps.folder'` inside the source folder.
2. If missing, create it: `create_drive_folder` with the parent set to the
   source folder.
3. Move each file: `update_drive_file` with `addParents=<dest_folder_id>` and
   `removeParents=<source_folder_id>`.

### After moving, log to the audit Sheet:

Append one row per file to a Sheet named **"Drive Organizer Log"** (create it
in the user's Drive root if it doesn't exist). Columns:

| Timestamp | Run ID | File Name | File ID | From Folder | From Folder ID | To Folder | To Folder ID | Rule Matched |
|---|---|---|---|---|---|---|---|---|

Use `modify_sheet_values` or `create_sheet` + `modify_sheet_values` to write
the log. The run ID is a short timestamp string (e.g. `2026-06-09T14:32`) so
all rows from one run share the same ID for easy undo.

### Report after APPLY:
```
DONE — 47 files moved

  Docs/          12 ✓
  Sheets/         8 ✓
  PDFs/          15 ✓
  Images/         7 ✓
  Archive/Old/    5 ✓

Audit log: "Drive Organizer Log" sheet (47 rows appended, Run ID: 2026-06-09T14:32)
To undo: /organize-drive undo run=2026-06-09T14:32
```

---

## Step 4 — UNDO mode

Read the audit Sheet, filter by run ID (or default to the most recent run),
and reverse each move: call `update_drive_file` with `addParents=<from_folder_id>`
and `removeParents=<to_folder_id>`.

After reversing, append a note to the audit Sheet rows (add "UNDONE" to a status
column) so the log stays accurate.

```
UNDO — run 2026-06-09T14:32 (47 files)

Reversing moves...  47/47 ✓

All files returned to original locations.
Audit log updated (status: UNDONE).
```

---

## Token efficiency rules

1. **Never use `get_drive_file_content`** unless the user explicitly asks to
   read a file. Content reads are expensive and unnecessary for organization.
2. **Batch search queries** — one `search_drive_files` call per rule (not one
   per file). The Drive API returns up to 1000 results per page; use
   `pageToken` to paginate.
3. **Reuse folder IDs** — look up each destination folder once, cache the ID
   for the whole run. Don't re-query the same folder name on every move.
4. **Skip already-organized files** — if a file's current parent ID already
   matches the destination folder ID, skip it without an API call.
5. **Report counts, not file lists, for large runs** — if more than 20 files
   in a category, show "23 files (first 3: foo.pdf, bar.pdf, baz.pdf...)".
   Full names only on request.

---

## Google Apps Script (server-side, scheduled)

For automated / scheduled runs without Claude in the loop, deploy this Apps
Script. Rules live in a companion Sheet — no code changes needed to adjust them.

### Setup
1. Open Google Sheets → Extensions → Apps Script → paste the script below.
2. Create a Sheet tab named **"Rules"** with these columns:
   `Rule Name | MIME Type | Name Contains | Older Than Days | Target Folder | Enabled`
3. Create a Sheet tab named **"Log"** with columns:
   `Timestamp | Run ID | File Name | File ID | From Folder | To Folder | Rule`
4. Set `SOURCE_FOLDER_NAME` in the script to your target folder (or `""` for root).
5. Run `setupTrigger()` once to schedule daily execution, or run `organizeFiles()`
   manually.

```javascript
// ============================================================
// Drive Organizer — Google Apps Script
// Rules in "Rules" sheet. Log in "Log" sheet.
// ============================================================

const SOURCE_FOLDER_NAME = "";  // "" = My Drive root; or folder name like "Inbox"
const DRY_RUN = false;          // true = log only, no moves

function organizeFiles() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const rulesSheet = ss.getSheetByName("Rules");
  const logSheet   = ss.getSheetByName("Log") || ss.insertSheet("Log");
  
  // Ensure log header
  if (logSheet.getLastRow() === 0) {
    logSheet.appendRow(["Timestamp","Run ID","File Name","File ID","From Folder","To Folder","Rule","Dry Run"]);
  }
  
  const runId = Utilities.formatDate(new Date(), "America/New_York", "yyyy-MM-dd'T'HH:mm");
  const rules = loadRules(rulesSheet);
  const sourceFolder = getSourceFolder(SOURCE_FOLDER_NAME);
  const files = sourceFolder.getFiles();
  const folderCache = {};
  let moved = 0, skipped = 0;

  while (files.hasNext()) {
    const file = files.next();
    const rule = matchRule(file, rules);
    if (!rule) { skipped++; continue; }
    
    const destFolder = getOrCreateFolder(sourceFolder, rule.targetFolder, folderCache);
    
    logSheet.appendRow([
      new Date(), runId,
      file.getName(), file.getId(),
      sourceFolder.getName(), rule.targetFolder,
      rule.name, DRY_RUN ? "YES" : "NO"
    ]);
    
    if (!DRY_RUN) {
      destFolder.addFile(file);
      sourceFolder.removeFile(file);
      moved++;
    }
  }
  
  Logger.log("Run %s: moved=%d skipped=%d dry_run=%s", runId, moved, skipped, DRY_RUN);
}

function loadRules(sheet) {
  const data = sheet.getDataRange().getValues();
  return data.slice(1)  // skip header
    .filter(r => r[5] === true || r[5] === "TRUE" || r[5] === "Yes")
    .map(r => ({
      name:           r[0],
      mimeType:       r[1] || null,
      nameContains:   r[2] || null,
      olderThanDays:  r[3] ? Number(r[3]) : null,
      targetFolder:   r[4],
    }));
}

function matchRule(file, rules) {
  const mime    = file.getMimeType();
  const name    = file.getName().toLowerCase();
  const modTime = file.getLastUpdated();
  const ageMs   = Date.now() - modTime.getTime();
  const ageDays = ageMs / 86400000;

  for (const rule of rules) {
    const mimeOk = !rule.mimeType    || mime === rule.mimeType;
    const nameOk = !rule.nameContains || name.includes(rule.nameContains.toLowerCase());
    const ageOk  = !rule.olderThanDays || ageDays > rule.olderThanDays;
    if (mimeOk && nameOk && ageOk) return rule;
  }
  return null;
}

function getSourceFolder(name) {
  if (!name) return DriveApp.getRootFolder();
  const iter = DriveApp.getFoldersByName(name);
  if (!iter.hasNext()) throw new Error("Source folder not found: " + name);
  return iter.next();
}

function getOrCreateFolder(parent, name, cache) {
  if (cache[name]) return cache[name];
  const iter = parent.getFoldersByName(name);
  const folder = iter.hasNext() ? iter.next() : parent.createFolder(name);
  cache[name] = folder;
  return folder;
}

function setupTrigger() {
  // Daily at 2am Eastern — delete existing triggers first to avoid dupes
  ScriptApp.getProjectTriggers()
    .filter(t => t.getHandlerFunction() === "organizeFiles")
    .forEach(t => ScriptApp.deleteTrigger(t));
  ScriptApp.newTrigger("organizeFiles")
    .timeBased().atHour(2).everyDays(1).create();
  Logger.log("Daily trigger set for organizeFiles at 2am.");
}
```

### Example Rules sheet rows

| Rule Name | MIME Type | Name Contains | Older Than Days | Target Folder | Enabled |
|---|---|---|---|---|---|
| Google Docs | application/vnd.google-apps.document | | | Docs | TRUE |
| Spreadsheets | application/vnd.google-apps.spreadsheet | | | Sheets | TRUE |
| PDFs | application/pdf | | | PDFs | TRUE |
| Invoices | | invoice | | Finance/Invoices | TRUE |
| Receipts | | receipt | | Finance/Receipts | TRUE |
| Screenshots | | screenshot | | Screenshots | TRUE |
| Old files | | | 365 | Archive/Old | TRUE |

---

## Gotchas

- **Shared files in "My Drive"** — `'<folder_id>' in parents` only finds files
  where you are the owner or where the file is directly in that folder. Shared
  files added to "My Drive" are in root but owned by others; they will appear
  in queries. Add `'me' in owners` to rules if you only want to move your own
  files.
- **Shortcuts** — shortcuts have MIME type `application/vnd.google-apps.shortcut`.
  Moving a shortcut moves the shortcut, not the target. Usually fine; flag if
  the user seems to expect otherwise.
- **Files in multiple parents** — Drive supports a file in multiple folders.
  `removeParents` only removes the specified parent; if the file has others, it
  stays in them. This is correct behavior (non-destructive), but worth noting.
- **Apps Script scope** — the Apps Script uses `DriveApp` which only sees files
  the script owner can access. For Shared Drives, use `Drive.Files.list()` with
  `includeItemsFromAllDrives=true` and `supportsAllDrives=true` instead of
  `DriveApp`.
- **Rate limits** — Drive API: 1000 req/100s per user. For large folders (1000+
  files), add `Utilities.sleep(100)` between batches in the Apps Script.
- **Undo window** — the audit log is the only undo path. Keep the "Log" sheet.
  Don't delete it between runs.
