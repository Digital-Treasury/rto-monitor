# RTO Monitor

Scheduled regulatory news monitor for Australian RTO owners and training
providers. Powered by Claude Code Routines.

## What this does

Every 2 days, a Claude Code routine runs in Anthropic's cloud, fetches a
curated list of regulatory and industry sources, identifies new items
relevant to RTOs, dedupes against past runs, and writes a report to
`reports/`. New items are also pushed to a Notion database for content
planning.

On Fridays, the routine also produces a weekly compilation report for
the Monday team meeting.

## Repo structure

```
.
├── CLAUDE.md              # Instructions the routine reads each run
├── sources.json           # Source list with tier and notes
├── history/
│   └── seen.json          # Deduplication state — appended each run
├── reports/
│   ├── daily-*.md         # One per run
│   └── weekly-*.md        # One per Friday run
└── README.md              # This file
```

## Editing the routine

To change what gets monitored or how items are scored:

- Add/remove sources: edit `sources.json`
- Change relevance criteria, output format, or audience targeting: edit
  `CLAUDE.md`
- Reset deduplication (force everything to look "new" again): replace
  `history/seen.json` with `{"items": []}`

The next routine run picks up changes automatically — no redeploy needed.

## Routine configuration

- **Name:** RTO Regulatory Monitor
- **Schedule:** Every 2 days, 7:00 AM Australia/Melbourne
- **Model:** Sonnet
- **Connectors:** Notion (for posting summaries)
- **Trigger:** Scheduled

Managed at [claude.ai/code/routines](https://claude.ai/code/routines).

## Manual runs

Use "Run now" on the routine page in claude.ai/code/routines to fire an
ad-hoc run — useful when testing changes to `CLAUDE.md` or `sources.json`.

## Maintained by

Digital Treasury — Melbourne.
