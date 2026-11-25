Your job is to produce a complete daily news brief for Karim Sadroudine (karim.sadroudine@gmail.com). Run every step in sequence. Do not skip steps or stop early.

This task runs as a Claude Code scheduled task in Anthropic's cloud. The GitHub repository `ksadroudine/my-news-brief` is attached to the task and is cloned into your working directory at the start of every run. That repository is the master copy of all brief data: the weekly JSON files, `index.html`, `sources.json` and `PIPELINE_NOTES.md` all live at its root. Vercel deploys the site automatically whenever a commit reaches the `main` branch.

---

### STEP 0 — Prepare the repository

1. Confirm you are at the repository root (it contains `index.html` and files like `2026-11-08.json`). If not, `cd` into the cloned `my-news-brief` directory.
2. Make sure you are working on `main` and up to date: `git checkout main && git pull --ff-only origin main`.
3. Record, for later safety checks, the full set of entry ids across all JSON files at this point (e.g. save them to a file in `/tmp`).

Do not create a `claude/` branch or a pull request. All changes are committed directly to `main` (Step 11).

---

### STEP 1 — Determine today's date and window

Get the current date in the America/Toronto time zone in YYYY-MM-DD format (`TZ=America/Toronto date +%F`). The search window covers the past 24 hours (since yesterday at 18:00 Toronto time).

Always use America/Toronto dates, never UTC. Gmail date headers are in UTC and must be converted.

---

### STEP 2 — Retrieve newsletter emails from Gmail

Use the Gmail connector.

Search for newsletter emails received in the past 24 hours. Run **both** of the following `search_threads` queries (with `pageSize` 50) and combine the results (deduplicate by thread ID). Replace YESTERDAY with yesterday's date as YYYY/MM/DD.

**Query 1 — inbox, subject keywords:**
`after:YESTERDAY subject:(briefing OR espresso OR morning OR digest OR newsletter)`

**Query 2 — inbox, known senders:**
`after:YESTERDAY from:(connectivity@b2b.smartbrief.com OR mwl@email.mobileworldlive.com OR crew@technews.therundown.ai OR noreply@globeandmailnewsletters.com OR generativeaienterprise@mail.beehiiv.com OR publishing@email.mckinsey.com OR hello@theinformation.com OR info@theinformation.com OR access@interactive.wsj.com OR wallstreetjournal@mail.sailthru.com OR news@thelogic.co OR noreply@news.bloomberg.com)`

After combining both result sets, retrieve each thread's full content with `get_thread`.

**Sources to skip entirely — do not retrieve or process:**
- The Neuron (sender: theneuron@newsletter.theneurondaily.com)
- The Rundown AI (sender: news@daily.therundown.ai)

If threads from these senders appear in search results, discard them and do not call `get_thread` on them. Also skip purely promotional emails (subscription offers, Black Friday sales, event/webinar invitations, "we reported it first" recap emails) without retrieving them when the subject and snippet make that clear.

**Cross-day duplicate risk (IMPORTANT):** Gmail's `after:` filter is day-granular, so this window overlaps with the previous run's. Before creating an entry, check whether an entry with a matching or closely matching title already exists with yesterday's or today's date. If one does, the story is already covered: do not add it again. An email that arrived after the previous run and is not yet in the brief goes into today's brief. Genuinely new coverage of an older topic is fine to add.

Priority sources (process first, but do not ignore others):
1. Wall Street Journal
2. The Logic
3. The Globe and Mail
4. Bloomberg
5. The Economist
6. The Information
7. Financial Times

Also process every other newsletter-style email as a lower-priority source.

**Large emails:** if `get_thread` reports that its result was too large and saved to a file, read that file with python or jq (extract `plaintextBody`, or strip the `htmlBody` to text). WSJ tracking links (`trk.wsj.com/click/.../<base64>/...`) contain the real article URL base64-encoded in the path segment — decode it to recover the wsj.com link. Only if the file cannot be read, fall back to the subject line and snippet plus `WebSearch` against the source's own domain. Note any fallbacks in the completion report.

---

### STEP 2b — Read pipeline notes

Read `PIPELINE_NOTES.md` at the repository root and apply all of its rules throughout Steps 3–8, including the Entry Dating Rule.

---

### STEP 3 — Extract articles

For each newsletter email, extract the distinct, substantive stories it covers. For each, capture:
- Title (as written in the newsletter)
- A 2–3 sentence neutral, factual summary in your own words (never copy newsletter text verbatim)
- A URL link if present in the email (otherwise null)
- The source publication name
- The newsletter name

---

### STEP 4 — Assign categories

Assign each article exactly one category from this fixed list:
Markets & Finance; Technology & AI; Business & Strategy; Politics & Policy; Energy & Commodities; Canadian Business; Global Economy; Real Estate; Professional Services; Science & Health; Culture & Ideas.

Do not invent new categories.

---

### STEP 5 — Deduplicate across sources

Group articles covering the same underlying story (same event, announcement or report). Keep one entry per story, from the highest-priority source:
1. Wall Street Journal
2. The Logic
3. The Globe and Mail
4. Bloomberg
5. The Economist
6. The Information
7. Financial Times
8+ Any other source

On the kept entry, list every other newsletter that covered the story in `also_covered_by` (newsletter names, not publication names). Use semantic similarity, not identical URLs or titles.

---

### STEP 6 — Compute the News Highlight

`coverage_count = 1 + length(also_covered_by)`. The article with the highest count is the News Highlight; break ties by source priority (lower rank wins). Exactly one entry per day gets `"priority_pick": true`; all others get `false`.

---

### STEP 7 — Build the entry objects

Each entry must match this schema exactly:

```json
{
  "id": "YYYY-MM-DD-NNN",
  "date": "YYYY-MM-DD",
  "source": "Publication Name",
  "newsletter": "Newsletter Name",
  "title": "Article Title",
  "summary": "2–3 sentence neutral summary.",
  "link": "https://...",
  "category": "Category Name",
  "also_covered_by": ["Newsletter Name A"],
  "priority_pick": false
}
```

- `id`: the entry's date plus a zero-padded 3-digit sequence number that does not collide with existing ids for that date.
- `link`: null if no valid URL is available. Strip tracking parameters such as `?mod=...` from WSJ links.

---

### STEP 7b — Resolve missing links for WSJ, Bloomberg, The Economist and FT

For entries whose `source` is Wall Street Journal, Bloomberg, The Economist or Financial Times and whose `link` is null, run `WebSearch` with `site:wsj.com <title words>` (or `site:bloomberg.com` / `site:economist.com` / `site:ft.com`), unquoted. If a result's title closely matches, use that URL. Only use URLs that appear in search results — never construct or guess a URL. If nothing matches, leave `link` null. Do not search for other sources in this step.

---

### STEP 8 — Write to the JSON file

Weekly files at the repository root are named by the window's start date:
- `window_day = ((day_of_month - 1) // 7) * 7 + 1`
- filename: `YYYY-MM-{window_day:02d}.json` (e.g. Nov 1–7 → `2026-11-01.json`, Nov 8–14 → `2026-11-08.json`, Nov 29–30 → `2026-11-29.json`).
- Legacy file `2026-05.json` covers all of May 2026 and must never be modified.

Every entry's date is today's run date in America/Toronto (TZ=America/Toronto date +%F), and it goes into that date's file. Emails that arrived after the previous run belong to today's brief, never to an earlier day. This includes The Information Briefing. Create the file as `[]` if it doesn't exist yet.

Append the new entries to the end of the array. Never delete, reorder or rewrite existing entries. Skip any entry whose id already exists. Write each file once, as valid JSON (`json.dumps(..., indent=2, ensure_ascii=False)`), and re-load it to confirm it parses.

If the Gmail search returned zero newsletters, write nothing, note it in the report, and skip to Step 12.

---

### STEP 9 — Rebuild index.html

`index.html` contains a data block delimited by these exact markers:

```
// @@BRIEF_DATA_START@@
const BRIEF_DATA = [ ... ];
// @@BRIEF_DATA_END@@
```

Replace everything from the start marker through the end marker with a fresh block built from ALL entries in ALL JSON files at the repository root matching `20[0-9][0-9]-[0-9][0-9].json` or `20[0-9][0-9]-[0-9][0-9]-[0-9][0-9].json` (do not include `sources.json`). Sort by id ascending, then stably by date descending. Run this from the repository root:

```bash
python3 - <<'EOF'
import json, glob
files = sorted(set(glob.glob('20[0-9][0-9]-[0-9][0-9].json') + glob.glob('20[0-9][0-9]-[0-9][0-9]-[0-9][0-9].json')))
entries = []
for f in files:
    with open(f, encoding='utf-8') as fh:
        entries.extend(json.load(fh))
entries.sort(key=lambda e: e['id'])
entries.sort(key=lambda e: e['date'], reverse=True)
block = '// @@BRIEF_DATA_START@@\nconst BRIEF_DATA = ' + json.dumps(entries, indent=2, ensure_ascii=False) + ';\n// @@BRIEF_DATA_END@@'
c = open('index.html', encoding='utf-8').read()
a = c.index('// @@BRIEF_DATA_START@@')
b = c.index('// @@BRIEF_DATA_END@@') + len('// @@BRIEF_DATA_END@@')
open('index.html', 'w', encoding='utf-8').write(c[:a] + block + c[b:])
print('BRIEF_DATA entries:', len(entries))
EOF
```

Do not alter any other part of `index.html` (it contains light/dark-mode styling that must be preserved).

---

### STEP 10 — Handle new sources

Read `sources.json`. If any newsletter processed today, or any primary source named in a `source` field, is not already listed in `priority_sources` or `other_sources`, append it to `other_sources` with the next rank (max existing rank + 1), the fields `name`, `newsletter_name`, `domain` (if known), `"auto_discovered": true` and a short `note` ("Auto-added YYYY-MM-DD; …"), and set `last_updated` to today. Mention each new source in the report and ask Karim whether he wants to adjust its rank.

---

### STEP 11 — Safety checks, commit and push to main

1. **No data loss:** confirm every id recorded in Step 0 is still present across the JSON files. If any is missing, stop: do not commit, and report the missing ids.
2. **Styling intact:** confirm `index.html` still contains `prefers-color-scheme: dark`. If not, restore it with `git checkout -- index.html`, report the problem, and do not commit it.
3. Commit and push directly to `main`:

```bash
git config user.email "ksadroudine@users.noreply.github.com"
git config user.name "Karim Sadroudine"
git add 20*.json index.html sources.json
git commit -m "my-news-brief YYYY-MM-DD" || echo "Nothing to commit"
git push origin HEAD:main
```

4. If the push is rejected because `main` moved during the run, run `git pull --rebase origin main`, repeat check 1, and push again once. If the rebase conflicts on a JSON file or `index.html`, abort the rebase (`git rebase --abort`) and report rather than forcing.
5. Never use `--force`. Never push to any other branch or open a pull request.

If the push fails for any other reason, report the exact error. If a notification tool is available in this session, also send Karim a short notification that the site was not updated, quoting the error.

---

### STEP 12 — Completion report

Finish with a brief report:
- Date processed
- Newsletters found / processed / skipped (with reasons)
- Articles extracted before dedup, and entries added after dedup (per date)
- News Highlight title and coverage count
- JSON file(s) written, with entry counts before and after
- Total BRIEF_DATA entries in index.html
- Commit hash and whether the push to main succeeded (or the exact error)
- New sources added, with ranks
- Any errors, warnings or fallbacks

---

### KEY CONSTRAINTS

- Never delete existing JSON entries. Append only. Never modify `2026-05.json`.
- Never alter `index.html` outside the BRIEF_DATA markers.
- Summaries are 2–3 sentences, neutral, factual, in your own words.
- Exactly one `priority_pick: true` entry per day.
- Categories come only from the fixed list.
- Links come only from the email body or WebSearch results; never construct URLs.
- Commit directly to `main`; never force-push; never create branches or pull requests.
- Never put credentials or tokens in commands, files or commit messages.
