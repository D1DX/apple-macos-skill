# Apple macOS Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![Apple](https://img.shields.io/badge/Apple-Skill-000000?style=flat&logo=apple&logoColor=white)](https://www.apple.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

Apple skill for AI agents. Covers controlling Notes, Reminders, and Voice Memos via AppleScript and SQLite — no third-party dependencies required. Includes the undocumented Voice Memos SQLite schema (Apple provides no AppleScript or public API for Voice Memos).

## What's Included

| Topic | What it covers |
|-------|---------------|
| Apple Notes | List folders, read content, create, delete — full AppleScript reference |
| Apple Reminders | List lists, list reminders, create, complete, delete — full AppleScript reference |
| Voice Memos | SQLite database location, `ZCLOUDRECORDING` schema, query patterns |
| Voice Memos Queries | List all recordings, filter by date range, export to JSON |
| Core Data Epoch | ZDATE uses 2001-01-01 epoch — always add 978307200 to convert to Unix |
| Column Name Gotchas | ZENCRYPTEDTITLE holds the title (not encrypted); ZCUSTOMLABEL holds an ISO date string |
| iCloud Stubs | Detecting placeholder .m4a files (~3–4 KB) that haven't synced locally |
| Notes Gotchas | HTML body format, plaintext vs body, case-sensitive folder names, "Recently Deleted" |
| Reminders Gotchas | AppleScript date format, recurrence limitations, completed as boolean |

## Install

### Claude Code

```bash
git clone https://github.com/D1DX/apple-skill.git
cp -r apple-skill ~/.claude/skills/apple
```

Or as a git submodule:

```bash
git submodule add https://github.com/D1DX/apple-skill.git path/to/skills/apple
```

### Other AI Agents

Copy `SKILL.md` into your agent's prompt or knowledge directory. The skill is structured markdown — works with any LLM agent that reads reference files.

## Structure

```
apple-skill/
└── SKILL.md    — Main skill (Notes AppleScript, Reminders AppleScript, Voice Memos SQLite schema)
```

## Note on Voice Memos

Apple's Voice Memos app has no AppleScript dictionary and no public API. The SQLite schema documented in this skill was reverse-engineered from the live database at `~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/CloudRecordings.db`. Column names are misleading Core Data artifacts — the skill documents what each column actually contains.

## Sources

- **Apple Notes & Reminders:** Verified against macOS AppleScript dictionaries and live usage (April 2026).
- **Voice Memos SQLite schema:** Reverse-engineered from live database inspection on macOS Sequoia (March 2026). Apple does not document this schema — treat as subject to change with OS updates.

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
