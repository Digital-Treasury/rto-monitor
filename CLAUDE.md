# RTO Regulatory News Monitor

You are running a scheduled content research routine for Digital Treasury,
a Melbourne-based web agency. The audience is Australian RTO owners and
training providers — clients of Digital Treasury's sister brand RTOPilot.

Your job each run is to identify NEW regulatory and industry news relevant
to RTO owners and training providers, dedupe against past runs, and produce
a concise report.

## Audience profile

RTO owners, compliance managers, training managers, and CEOs of small-to-mid
size training providers. They care about:

- ASQA audits, audit outcomes, enforcement actions, deregistrations
- Standards for RTOs (compliance changes, new standards)
- Training package and qualification updates
- USI (Unique Student Identifier) changes
- AVETMISS reporting requirements
- VET sector funding announcements
- CRICOS and international student requirements (for RTOs with overseas students)
- Director responsibilities (ASIC-related, for RTO owners as company directors)

## Process every run

### 1. Read state

- Read `sources.json` for the list of URLs to check
- Read `history/seen.json` for items already captured in past runs
- Note today's date (provided in the prompt)
- Confirm `FIRECRAWL_API_KEY` is available as an environment variable

### 2. Fetch each source

For each source in `sources.json`, check the `requires_firecrawl` flag:

**If `requires_firecrawl` is `true`**, use Firecrawl via bash + curl:

```bash
curl -s -X POST 'https://api.firecrawl.dev/v2/scrape' \
  -H "Authorization: Bearer $FIRECRAWL_API_KEY" \
  -H 'Content-Type: application/json' \
  --data-raw "{\"url\":\"SOURCE_URL_HERE\"}"
```

The response is JSON. The scraped content is in `data.markdown`. Parse
that markdown for article titles, dates, and URLs. Firecrawl returns
clean markdown with item titles as `**bold**` text, dates in plain text,
and links in standard markdown format.

**If `requires_firecrawl` is `false` or unset**, use plain curl:

```bash
curl -sL \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -H "Accept: application/rss+xml, application/xml, text/xml, text/html" \
  --max-time 30 \
  --retry 1 \
  "SOURCE_URL_HERE"
```

**For both methods:**

- Check the `type` field:
  - If `type` is `rss`: parse `<item>` blocks for title, link, pubDate, description
  - If `type` is `html`: extract recent articles from the markdown/HTML structure
- If response is empty, returns an error JSON (`success: false`), or
  contains bot-challenge text, log under "Sources needing manual review"
  and move on
- Never retry a failed source more than once

**Important:** Do NOT use the WebFetch tool — it performs a robots.txt
check that times out on many sites. curl (with or without Firecrawl) is
the correct tool for this environment.

### 3. Filter to new and recent items

- Only consider items published or updated in the last 7 days
- Skip anything whose URL OR title-hash matches an entry in
  `history/seen.json`
- Older items that appear "new" to the scraper (e.g., reordered lists)
  are NOT new

### 4. Judge relevance

Score each candidate item:

- **HIGH**: Direct compliance impact — ASQA audits/enforcement, Standards
  for RTOs changes, training package updates, USI/AVETMISS changes, ASQA
  fees, CRICOS changes, anything that affects how an RTO operates
  day-to-day
- **MEDIUM**: VET policy direction, funding announcements, peak body
  submissions, NCVER research with practical implications, ASIC changes
  affecting RTO directors
- **LOW**: General education news, ministerial speeches without policy
  content, event announcements, regulator hiring, conference recaps,
  award winners

**Drop LOW-relevance items entirely.**

### 5. Update history

Append every NEW item (HIGH and MEDIUM) to `history/seen.json` with:

```json
{
  "url": "https://...",
  "title_hash": "first 8 chars of sha256 of title",
  "captured": "YYYY-MM-DD",
  "source": "Source Name"
}
```

### 6. Write the daily report

Write to `reports/daily-YYYY-MM-DD.md` using this structure:

```markdown
# RTO Monitor — Daily Report
**Date:** YYYY-MM-DD
**Sources checked:** X
**New items found:** Y
**Flagged for content:** Z

## Summary
[One-line summary of the run]

## New items

### [Source Name]

**[Item title]**
*Published: YYYY-MM-DD | Relevance: HIGH/MEDIUM*

[2-3 sentence summary of what changed and the practical impact for RTOs.
Plain language. No marketing voice.]

Content-worthy: YES / NO
[If YES, one-line angle for LinkedIn]

URL: [link]

---

[repeat for each item, grouped by source]

## Suggested LinkedIn content topics

[Bullet list of 1-3 strongest content angles from this run. Each one line.
Write the actual hook. If nothing content-worthy, write "None this run."]

## Sources needing manual review

[List any source that blocked access, timed out, or returned unexpected
structure. Format: "Source Name — reason"]

[If no issues: "All sources fetched cleanly."]
```

### 7. Friday weekly compilation

If today is a Friday, also write `reports/weekly-YYYY-W##.md`:

- Read all daily reports from the past 7 days
- De-duplicate items across days
- Group by theme (not by day)
- Lead with the 3-5 most significant items
- Include a "This week's content angles" section
- Keep under 2 pages — read in a Monday meeting

### 8. Post summary to Notion

Use the Notion connector to post a summary to the "RTO Monitor" database.

- If database doesn't exist, create it with: Date (date), Source (text),
  Title (text), Relevance (select: HIGH/MEDIUM), Content-worthy (checkbox),
  URL (URL), Summary (text)
- Add one row per new item from this run
- Check by URL before inserting to avoid duplicates
- **If the Notion connector is read-only or fails, note this at the
  bottom of the daily report and continue. Do not fail the run.**

### 9. Commit and push directly to main

```bash
git add -A
git commit -m "Monitor run: YYYY-MM-DD (X new items)"
git push origin HEAD:main
```

Do NOT push to a claude/-prefixed branch. The main branch must receive
updates directly so deduplication works across runs.

## Constraints

- **Plain language.** No marketing voice. No "exciting developments"
  or "groundbreaking changes". Just what changed and what it means.
- **Don't pad.** Short clean report > long padded one.
- **Never fabricate.** If a date or detail isn't clearly visible, write
  "date unclear" rather than guess.
- **Token budget.** Daily report ≤2000 tokens. Weekly ≤3000.
- **Firecrawl credit awareness.** Each Firecrawl call uses 1 credit.
  Don't re-fetch a source within the same run.

## What "content-worthy" means

Flag YES when the item could become a useful LinkedIn post for an RTO
audience. Good signals:

- A specific change RTOs need to act on (deadline, new requirement)
- An audit finding or enforcement action others can learn from
- A common misconception the news clarifies
- A practical compliance tip embedded in the announcement

Skip YES when:

- Purely informational with no action implied
- Would require speculation beyond what's stated
- A story everyone in the sector already knows
