---
name: apple
description: Control Apple Notes, Reminders, and Voice Memos via AppleScript and SQLite
user_invocable: true
triggers:
  - apple notes
  - apple reminders
  - voice memos
  - delete note
  - create reminder
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
