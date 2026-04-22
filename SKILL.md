---
name: apple-macos
description: Control Apple Notes, Reminders, Voice Memos, and Shortcuts on macOS via AppleScript and SQLite. Triggers on "apple notes", "apple reminders", "apple shortcuts", "voice memos", "macOS automation".
user-invocable: true
triggers:
  - apple notes
  - apple reminders
  - voice memos
  - apple shortcuts
  - delete note
  - create reminder
  - shortcut
  - check-in shortcut
---

# Apple Apps — Notes, Reminders, Voice Memos

Control Apple Notes, Reminders, and Voice Memos from Claude via AppleScript and SQLite.

---

## Apple Notes

Notes supports full AppleScript control.

### List all notes in a folder
```bash
osascript -e 'tell application "Notes" to get name of every note in folder "Notes"'
```

### Read a note's content
```bash
osascript -e 'tell application "Notes" to get plaintext of note "Note Title"'
```

### Delete a note
```bash
osascript -e 'tell application "Notes" to delete note "Note Title"'
```

### Create a note
```bash
osascript -e 'tell application "Notes" to make new note at folder "Notes" with properties {name:"Title", body:"<h1>Title</h1><p>Body text</p>"}'
```

### List folders
```bash
osascript -e 'tell application "Notes" to get name of every folder'
```

### Count notes in a folder
```bash
osascript -e 'tell application "Notes" to get count of notes in folder "Notes"'
```

### Notes gotchas
- Note body uses **HTML**, not plain text. Use `<h1>`, `<p>`, `<br>`, `<ul><li>` tags.
- `plaintext` returns stripped text; `body` returns HTML source.
- Folder names are case-sensitive.
- Deleting moves to "Recently Deleted" — not permanent.

---

## Apple Reminders

Reminders supports full AppleScript control.

### List all reminders in a list
```bash
osascript -e 'tell application "Reminders" to get name of every reminder in list "Reminders"'
```

### Create a reminder
```bash
osascript -e 'tell application "Reminders" to make new reminder in list "Reminders" with properties {name:"Task title", body:"Details", due date:date "2026-03-20"}'
```

### Complete a reminder
```bash
osascript -e 'tell application "Reminders" to set completed of reminder "Task title" in list "Reminders" to true'
```

### Delete a reminder
```bash
osascript -e 'tell application "Reminders" to delete reminder "Task title" in list "Reminders"'
```

### List all reminder lists
```bash
osascript -e 'tell application "Reminders" to get name of every list'
```

### Reminders gotchas
- Date format in AppleScript: `date "YYYY-MM-DD"` or `date "March 20, 2026"`.
- Recurrence (repeat) cannot be set via AppleScript — must be done in the Reminders UI or Google Tasks API.
- `completed` is a boolean property, not a status field.

---

## Voice Memos

Voice Memos does **NOT** have an AppleScript dictionary. It cannot be controlled via `osascript`. Use the SQLite database instead.

### Database location
```
~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db
```

### Recording files location
```
~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/*.m4a
```

### Table: ZCLOUDRECORDING

| Column | Type | Description |
|--------|------|-------------|
| `ZENCRYPTEDTITLE` | VARCHAR | **User-visible title** (the name shown in Voice Memos app). Not actually encrypted — just a misleading Core Data column name. |
| `ZCUSTOMLABEL` | VARCHAR | Stores the ISO date string (e.g., `2026-03-31T11:29:15Z`). **Not the user-assigned name** despite the column name. |
| `ZDATE` | TIMESTAMP | Core Data epoch (seconds since 2001-01-01 UTC). Convert: `ZDATE + 978307200` = Unix epoch |
| `ZDURATION` | FLOAT | Duration in seconds |
| `ZPATH` | VARCHAR | Filename (e.g., `20260312 221848.m4a`) |
| `ZFOLDER` | INTEGER | FK to ZFOLDER table (folder/grouping) |

### Query all recordings
```bash
sqlite3 -header -separator '|' \
  ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db \
  "SELECT ZENCRYPTEDTITLE as title, datetime(ZDATE + 978307200, 'unixepoch') as date_utc, round(ZDURATION,1) as duration_sec, ZPATH FROM ZCLOUDRECORDING ORDER BY ZDATE DESC;"
```

### Query recordings by date range
```bash
sqlite3 -header -separator '|' \
  ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db \
  "SELECT ZENCRYPTEDTITLE as title, datetime(ZDATE + 978307200, 'unixepoch') as date_utc, round(ZDURATION,1) as duration_sec, ZPATH FROM ZCLOUDRECORDING WHERE ZDATE + 978307200 > strftime('%s', '2026-03-01') ORDER BY ZDATE;"
```

### Export to JSON (full inventory)
```bash
sqlite3 -json \
  ~/Library/Group\ Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db \
  "SELECT ZENCRYPTEDTITLE as title, datetime(ZDATE + 978307200, 'unixepoch') as date_utc, ZDATE + 978307200 as unix_timestamp, round(ZDURATION,1) as duration_seconds, ZPATH as path FROM ZCLOUDRECORDING ORDER BY ZDATE;"
```

### Voice Memos gotchas
- **No AppleScript support** — `Application("Voice Memos")` fails. Use SQLite only.
- **ZDATE uses Core Data epoch** (2001-01-01), not Unix epoch (1970-01-01). Always add `978307200` to convert.
- **ZDATE is UTC** — the filename contains local time, but the database stores UTC.
- **File path in ZPATH is relative** — prepend the Recordings directory path to get the full path.
- **Title is in ZENCRYPTEDTITLE, not ZCUSTOMLABEL** — despite the names, `ZENCRYPTEDTITLE` holds the user-visible title (not encrypted), while `ZCUSTOMLABEL` holds an ISO date string. Core Data naming is misleading.
- **iCloud stubs** — some .m4a files may be iCloud placeholders (tiny files ~3-4KB). Check file size before processing.
- **Timezone warning:** When creating Airtable records from Voice Memos timestamps, use the `unix_timestamp` (ZDATE + 978307200) directly — do NOT parse the filename or `recording_date` string, as those are in local device timezone and converting them requires knowing which timezone the device was in at recording time.

---

## Apple Shortcuts

Shortcuts can be controlled and inspected via two methods: AppleScript (via `Shortcuts Events`) and SQLite (direct database read). The action content of each shortcut lives in the `ZSHORTCUTACTIONS` table as a binary plist blob.

### Database location
```
~/Library/Shortcuts/Shortcuts.sqlite
```

### Key tables

| Table | Contents |
|-------|----------|
| `ZSHORTCUT` | One row per shortcut: `Z_PK`, `ZNAME`, `ZFOLDER` (FK), subtitle (action count), color |
| `ZSHORTCUTACTIONS` | One row per shortcut: `ZSHORTCUT` (FK), `ZDATA` (bplist blob of all actions) |
| `ZCOLLECTION` | Folders: `Z_PK`, `ZDISPLAYNAME` |
| `ZTRIGGER` | Automation triggers (iOS only — empty on macOS) |

### List all shortcuts with their folder
```bash
sqlite3 ~/Library/Shortcuts/Shortcuts.sqlite \
  "SELECT s.ZNAME, c.ZDISPLAYNAME as folder
   FROM ZSHORTCUT s
   LEFT JOIN ZCOLLECTION c ON s.ZFOLDER = c.Z_PK
   ORDER BY c.ZDISPLAYNAME, s.ZNAME;"
```

### List shortcuts in a specific folder
```bash
sqlite3 ~/Library/Shortcuts/Shortcuts.sqlite \
  "SELECT s.Z_PK, s.ZNAME
   FROM ZSHORTCUT s
   JOIN ZCOLLECTION c ON s.ZFOLDER = c.Z_PK
   WHERE c.ZDISPLAYNAME = 'YOUR_FOLDER_NAME'
   ORDER BY s.ZNAME;"
```

### Get metadata for specific shortcuts (action count, UUID)
```bash
osascript << 'EOF'
tell application "Shortcuts Events"
  repeat with s in shortcuts
    if name of s is in {"Shortcut One", "Shortcut Two"} then
      log "Name: " & name of s & " | Subtitle: " & subtitle of s
    end if
  end repeat
end tell
EOF
```

### Extract and decode shortcut actions (full action list)
```bash
# 1. Write ZDATA blob to a file
sqlite3 ~/Library/Shortcuts/Shortcuts.sqlite \
  "SELECT writefile('/tmp/sc.bin', ZDATA) FROM ZSHORTCUTACTIONS WHERE ZSHORTCUT=<PK>;"

# 2. Convert binary plist to XML
plutil -convert xml1 -o /tmp/sc.xml /tmp/sc.bin

# 3. Parse with Python
python3 << 'PYEOF'
import plistlib
with open('/tmp/sc.xml', 'rb') as f:
    actions = plistlib.load(f)
for i, action in enumerate(actions):
    ident = action.get('WFWorkflowActionIdentifier', '').replace('is.workflow.actions.', '')
    params = action.get('WFWorkflowActionParameters', {})
    name = params.get('WFWorkflowName') or params.get('WFSelectedApp', {}).get('Name', '')
    print(f"{i+1}. {ident}" + (f" → {name}" if name else ''))
PYEOF
```

### Run a shortcut by name
```bash
shortcuts run "Shortcut Name"
# or via AppleScript:
osascript -e 'tell application "Shortcuts Events" to run shortcut "Shortcut Name"'
```

### Open the Shortcuts app
```bash
osascript -e 'tell application "Shortcuts" to activate'
```

### Common action identifiers

| Identifier (after stripping `is.workflow.actions.`) | Meaning |
|------------------------------------------------------|---------|
| `runworkflow` | Run another shortcut. Key param: `WFWorkflowName` |
| `openapp` | Open an app. Key param: `WFAppIdentifier`, `WFSelectedApp.Name` |
| `waittoreturn` | Pause until user returns to Shortcuts |
| `choosefrommenu` | Present a menu to the user |
| `repeat.each` | Loop over a list |
| `conditional` | If/else branch |
| `filter.notes` | Filter Apple Notes |
| `shownote` | Display a note |
| `filter.photos` | Filter Photos library |
| `previewdocument` | Quick Look preview |
| `delay` | Wait N seconds |
| `gettext` | Get/produce text |
| `text.split` | Split text |
| `file.getfoldercontents` | List folder contents |
| `file.move` | Move a file |
| `file.delete` | Delete a file |
| `makezip` | Create a zip archive |
| `documentpicker.save` | Save to Files |
| `deletephotos` | Delete photos from library |
| `addnewreminder` | Create a reminder |
| `reminders.TTROpenSmartListAppIntent` | Open Reminders smart list |
| `mobilenotes.SharingExtension` | Save to Notes via share sheet |
| `mobilesafari.TabEntity` | Get Safari tabs |
| `mobilesafari.OpenTab` | Open a Safari tab |
| `mobilesafari.CloseTab` | Close a Safari tab |
| `mobilesafari.OpenView` | Open a Safari view |
| `readinglist` | Add to Reading List |
| `com.alexhay.ToolboxProForShortcuts.GlobalVariablesIntent` | Read/write Toolbox Pro global variable |
| `VoiceMemos.RCRecordingEntity` | Get voice memo recordings |

### Toolbox Pro global variables

Toolbox Pro (`com.alexhay.ToolboxProForShortcuts`) provides persistent cross-shortcut storage via its `GlobalVariablesIntent` action. Variables persist across shortcut runs (unlike local Shortcuts variables which reset each run).

To read: action identifier `GlobalVariablesIntent`, parameter `WFInput` = variable name.
To write: same action with write mode.

### Key gotchas

- **ZTRIGGER is empty on macOS** — Shortcuts Automations (time-based or app-based triggers) are created on iPhone and sync via iCloud, but the trigger metadata stays on the iOS device. The Mac SQLite has no trigger records.
- **`shortcuts list` CLI** works on macOS but only returns names. For action content, use SQLite + plutil decode.
- **Action blob format** — `ZDATA` in `ZSHORTCUTACTIONS` is a signed binary plist (Apple proprietary). `plutil -convert xml1` reliably converts it. Do not try to read it as raw text.
- **`Shortcuts Events` vs `Shortcuts`** — use `Shortcuts Events` for AppleScript control (list shortcuts, run shortcuts). `Shortcuts` (the UI app) can be activated but has limited AppleScript dictionary.
- **Folder-based dynamic step lists** — a shortcut can read all shortcuts in a given folder at runtime via `getmyworkflows` action, then loop over them. This means folder order = execution order. Adding a step = add a shortcut to the folder, no code change needed.

### Full extraction script (all shortcuts in a folder)
```bash
FOLDER_NAME="YOUR_FOLDER_NAME"
sqlite3 ~/Library/Shortcuts/Shortcuts.sqlite << SQL
.mode tabs
SELECT s.Z_PK, s.ZNAME
FROM ZSHORTCUT s
JOIN ZCOLLECTION c ON s.ZFOLDER = c.Z_PK
WHERE c.ZDISPLAYNAME = '$FOLDER_NAME';
SQL
```

Then for each PK, extract and decode as shown above.
