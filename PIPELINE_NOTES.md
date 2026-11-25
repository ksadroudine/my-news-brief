# My News Brief — Pipeline Notes

This file documents special handling rules, known quirks, and source-specific extraction patterns for the daily pipeline. Read this before running the pipeline.

---

## Entry Dating Rule (permanent, added 2026-11-15 by Karim — overrides all other dating rules)

**Every entry is dated to the day of the run that processes it** (the run's America/Toronto date, `TZ=America/Toronto date +%F`), not to the email's own send date. Any email that arrives after a run finishes appears in the *next* run's brief. Karim reads only the current day's brief and does not go back to earlier days, so nothing new may be filed under a past date.

- This overrides the earlier rule that dated The Information Briefing to the prior day: Briefing entries now also get the run's date.
- Do not use UTC dates. Gmail `date` headers are UTC (e.g. `2026-11-15T02:15Z` is Saturday 21:15 in Toronto); always convert with the America/Toronto zone.
- The cross-day duplicate check still applies: if a story is already in the brief from an earlier run, skip it rather than re-adding it under today's date.
- Example of the failure this fixes: a WSJ alert that arrived Sat 2026-11-14 at 21:15 Toronto, after the Saturday run, was first filed under 2026-11-14 and never appeared in Sunday's brief. It was re-dated to 2026-11-15 (id 2026-11-15-004).

---

## Source Priority Order (permanent, set 2026-11-25 by Karim — overrides all earlier orderings)

Use this order wherever a run needs source priority: the order for processing emails (Step 2), deduplication (Step 5) and the News Highlight tie-break (Step 6):

1. Wall Street Journal
2. The Logic
3. The Globe and Mail
4. Bloomberg
5. The Economist
6. The Information
7. Financial Times
8+ Any other source, in `sources.json` rank order

`sources.json` matches this order (ranks 1–7).

---

## Bloomberg Newsletter (added 2026-11-25 by Karim — in effect from the 2026-11-25 run onward)

Karim subscribed to a Bloomberg newsletter sent from **`noreply@news.bloomberg.com`**. It is in scope for every run.

- **Gmail search:** Karim added this sender to the scheduled prompt's Query 2 on 2026-11-25. If a run's Query 2 does not include `noreply@news.bloomberg.com`, run this query separately (`pageSize` 50) and process the results with the rest: `after:YESTERDAY from:noreply@news.bloomberg.com`.
- **Fields:** set `source` to "Bloomberg". Set `newsletter` to the newsletter's own name as shown in the subject or header (e.g. "Bloomberg Evening Briefing"), or "Bloomberg" if no name is shown.
- **Priority:** Bloomberg is rank 4. See "Source Priority Order" above. When it covers the same story as WSJ, The Logic or The Globe and Mail, keep that source's entry and list the Bloomberg newsletter in `also_covered_by`. When it shares a story with The Economist, The Information, FT or any other source, keep the Bloomberg entry.
- **Extraction:** follow the General Rules below and extract 1–3 substantive stories per issue. Skip sponsor content, event promotions and subscription offers. Use bloomberg.com article URLs from the body as `link`. If they are wrapped in tracking redirects, decode them only when the real URL is readable from the redirect itself; never construct one.
- **Format (verified 2026-11-25, first issue "Canada Daily: Pensions confront AI risk"):** the newsletter is **Bloomberg Canada Daily** (subject prefix "Canada Daily:", arrives ~17:00 Toronto). `get_thread` returns a readable `plaintextBody`. Structure: a lead essay, then "Top stories", "Markets and commodities" and "Before you go...". Links are direct `bloomberg.com/news/articles/...` URLs in angle brackets right after the linked words; strip the `?cmpid=...&utm_...` query string. Set `newsletter` to "Bloomberg Canada Daily".

---

## Email Extraction: General Rules

The Gmail `get_thread` tool returns `plaintextBody` and a short `snippet`. For most newsletters, the plaintext body contains the full article content and is sufficient for extraction.

### Substantive Stories Only

Extract only stories that meet at least one of the following criteria:
- Contains a quantified claim (revenue figures, user counts, deal sizes, growth rates, headcount numbers, etc.)
- Represents a significant strategic, policy, or market development (IPO filing, executive change, regulatory ruling, major product launch, M&A deal, etc.)
- Offers analysis or perspective not available from a primary news headline alone (column pieces, editorial takes, expert commentary with substance)

Skip the following regardless of newsletter:
- Brief "keep reading" bullets or one-line items without enough detail for a standalone summary
- Pure brand campaign announcements or product marketing
- Conference invitations, event promotions, or sponsored content
- Earnings data bullets that merely restate a number already covered by a higher-priority source
- Entertainment, lifestyle, or cultural items with no business or technology relevance
- Sponsor callouts, upgrade pitches, and subscription promotions

When a newsletter contains a mix of substantive and non-substantive items — which most do — extract 1–3 stories and leave the rest. Erring toward fewer, higher-quality entries is always preferable to padding the database with marginal items.

---

**Never skip a newsletter solely because it has no outbound hyperlink to a published article.** If `plaintextBody` contains full narrative prose (column text, analysis, multi-paragraph summaries), extract content directly from the email body. The absence of a clickable link is not a reason to skip — it is simply a reason to set `"link": null` on the resulting entries.

The two cases to distinguish:
1. **Full prose in body** (e.g. The Information Weekend, Dealmaker columns) → extract directly; `link` = null unless a real URL appears elsewhere in the body.
2. **Body is a headline list** (e.g. SmartBrief) → WebSearch each headline.

When a newsletter email has no usable `plaintextBody`, strip its `htmlBody` to text and follow the source-specific rules below.

---

## Source-Specific Handling

### The Globe and Mail Business Brief (`noreply@globeandmailnewsletters.com`)

**Format (verified 2026-11-24):** daily weekday newsletter by Chris Wilson-Smith. `get_thread` returns a readable `plaintextBody`. Sections: an "In focus" lead story, "More top stories", "Charted", "Up next" and a "Morning update" markets summary.

**Extraction:** take the "In focus" story and 1–2 substantive items from "More top stories", preferring Canadian business and policy stories. Skip "Up next" calendar items, the "Morning update" market recap and podcast or app promotions.

**Links:** the body only contains `click.globeandmail.com` tracking redirects, not article URLs, so set `link` to null.

---

### The Information (`hello@theinformation.com` / `info@theinformation.com`)

**Format:** Plaintext body available and extractable.

**Newsletter types:**
- **The Information AM** (`hello@theinformation.com`, arrives ~11am ET): numbered list of articles with full summaries. Primary daily extraction source.
- **The Information Briefing** (`info@theinformation.com`, arrives ~midnight ET): nightly column by Martin Peers, written in prose. Its dateline is the prior day, but per the Entry Dating Rule (2026-11-15) its entries are dated to the run date like every other newsletter. If a story it covers is already in the brief for yesterday or today, skip it rather than adding a duplicate; otherwise create an entry dated to the run date.
- **The Information Weekend** (`hello@theinformation.com`, subject: "Weekend: [headline]", arrives Saturday): weekly column by Abram Brown. Full plaintext body — no link needed. Extract the main column story (typically 4–8 paragraphs of analysis on one theme) plus any secondary items that are distinct business/tech facts (e.g. funding milestones, product announcements, notable deals mentioned in passing). Skip entertainment recommendations (books, TV, film) and cultural observations with no business relevance. Link = null unless a real URL appears in the body.
- **The Information Dealmaker** (`hello@theinformation.com`, subject: "Dealmaker: [headline]"): weekly VC/funding column by Katie Roof. Full plaintext body. Extract the main column thesis as a single entry. Link = null.
- **The Information article alerts** (`hello@theinformation.com`, subject is the article headline): single-article promotional emails. The `plaintextBody` contains 1–3 paragraphs of article text plus a real article URL (e.g. `https://www.theinformation.com/articles/[slug]`). Extract the article and use the URL as the link — these are valid provenance-compliant URLs because they appear in `get_thread` output... **but** per the Web Fetch Provenance Rule, URLs from `get_thread` cannot be fetched directly. However, the article summary in the body is usually sufficient to write the entry without fetching. Use the URL as the `link` field directly (it was present in the email and can be cited as the source).
- **Applied AI** (`hello@theinformation.com`, subject: "Applied AI: [headline]"): in scope. Covers enterprise AI adoption, AI-driven business model shifts, and the impact of AI on software procurement and vendor relationships. Format is a short analytical piece on one main story. Extract 1 entry per issue; focus on the core business or strategic insight. Set `source` to "The Information" and `newsletter` to "Applied AI". Link = null unless a real article URL appears in the body.
- **Digest / recap emails** (`hello@theinformation.com`): skip. These are promotional round-ups of already-covered stories (e.g. "Top Posts Today," "This Week's Most Popular Stories," "Sunday recap," "Before X was everywhere, it was in The Information"). Identify by subject patterns like "recap," "most popular," "before X was everywhere," or "don't miss."

---

### WSJ Newsletters (`access@interactive.wsj.com`)

**Format:** Multiple distinct newsletters from the same sender. Identify by subject line:
- **WSJ Morning Download** — main daily CIO/tech newsletter. Full plaintext body.
- **WSJ Pro Cybersecurity** — daily cybersecurity newsletter. Full plaintext body.
- **WSJ CEO Brief** — daily leadership newsletter. Extract 1–2 stories max; focus on macro or operational insights (e.g. shipping disruptions, CEO strategy).
- **WSJ Future of Everything** — in scope. Covers science, technology, health, and society through a forward-looking lens. Full plaintext body. Extract 1–2 stories per issue; prioritize stories with quantified claims or meaningful implications for business or technology strategy. Set `source` to "Wall Street Journal" and `newsletter` to "WSJ Future of Everything". Category is typically "Technology & AI" or "Business & Strategy" depending on the story.
- **WSJ Heard on the Street** — in scope when available. Financial analysis and market commentary column covering public companies, earnings, valuations, and capital markets. Full plaintext body. Extract 1–2 stories per issue; focus on stories with material financial insight or strategic implications (e.g. earnings surprises, valuation shifts, M&A analysis). Skip brief market-data summaries with no analytical content. Set `source` to "Wall Street Journal" and `newsletter` to "WSJ Heard on the Street". Category is typically "Business & Strategy".
- **WSJ conference / event-marketing newsletters** — skip. Identify by subject patterns referencing specific WSJ Live events, conference invitations, or attendee promotions (e.g. "WSJ Tech Live," "WSJ CEO Council," "Join us at..."). These contain no original reporting.
- **WSJ Technology** (weekly, weekend) — in scope. **Sender note (2026-11-16):** issues can arrive from `wallstreetjournal@mail.sailthru.com` (display name "WSJ Technology") instead of `access@interactive.wsj.com`, and the subject is the lead headline (no newsletter keyword), so neither Gmail query catches it. The Gmail Query 2 sender list must include `wallstreetjournal@mail.sailthru.com`. HTML-only body: strip `htmlBody` to text and decode `trk.wsj.com` links. Extract the lead story plus 1–2 uncovered Big Stories; skip items already in the brief from earlier WSJ alerts.
- **WSJ CMO Today** — in scope. Daily marketing and advertising newsletter covering brand strategy, marketing technology, AI in advertising, consumer behavior, and media industry trends. Full plaintext body. Extract 1–2 stories per issue; prioritize stories with quantified claims, strategic business implications, or AI/technology angles. Skip pure brand campaign announcements and editorial calendar reminders. Set `source` to "Wall Street Journal" and `newsletter` to "WSJ CMO Today". Category is typically "Business & Strategy" or "Technology & AI" depending on the story angle.

---

### The Logic (`anna@thelogic.co` / `news@thelogic.co`)

**Format:** Two types:
- **Individual article alerts** (sender: `news@thelogic.co`): single article per email. Plaintext body available; extract if the article is a unique Canadian business story not covered elsewhere.
- **Weekly digest** (sender: `anna@thelogic.co`, subject: "Don't miss these stories from the past week"): skip. These are recaps of already-published content.

---

### The Rundown Tech (`crew@technews.therundown.ai`)

**Format:** daily newsletter with a full plaintext body covering consumer and enterprise technology, hardware, product launches and market trends. (The Rundown AI, `news@daily.therundown.ai`, is a separate newsletter and is skipped.)

**Extraction guidance:**
- Extract 2–3 stories per issue. Focus on stories with quantified claims or announcements not already covered by a higher-priority source.
- Skip: sponsor callouts, "tool of the day" sections, and brief one-line items without enough detail for a standalone summary.
- Links are often present in the body as direct source URLs. Use these as the `link` field if they satisfy the Link Integrity Rule (real article URLs with hash or path components, not constructed slugs).

---

### Enterprise AI Executive (`generativeaienterprise@mail.beehiiv.com`)

**Format:** Beehiiv-hosted newsletter. Formerly named "Generative AI Enterprise" — same sender address, rebranded. Full plaintext body available. Focus is enterprise AI adoption, agentic workflows, and AI-in-consulting. Published by Lewis Walker. Website: `enterpriseaiexecutive.ai`.

**Issue structure:**
- Named sections with labels like MARKET & BEST PRACTICE INSIGHT, MARKET INSIGHT, BEST PRACTICE INSIGHT & CASE STUDIES, AI-NATIVE PROFESSIONAL — each covering one substantive story with a Brief + Breakdown + Why it's important format.
- A "Transformation and technology in the news" digest section: multiple one-paragraph items about AI moves from specific companies (Anthropic, Microsoft, OpenAI, Google, etc.).
- Career opportunities, events, and upgrade/sponsorship pitches — skip all of these.
- A "In partnership with The Boardroom" section — skip (promotional).

**Extraction guidance:**
- Extract 2–3 stories per issue. Prioritize the named sections (MARKET INSIGHT, BEST PRACTICE INSIGHT, etc.) over the news digest bullets.
- The source field should reflect the originating organization (e.g. "Deloitte", "Microsoft", "OpenAI") rather than "Enterprise AI Executive" when the newsletter is reporting on a primary source document or report. Set `newsletter` to "Enterprise AI Executive" and `source` to the primary source organization.
- When the newsletter is the primary analysis (no external report), set both `source` and `newsletter` to "Enterprise AI Executive".
- Prioritize stories not already covered by a higher-priority source — this newsletter often surfaces consulting firm reports (Deloitte, McKinsey, Bain, Accenture, Google Cloud) before they reach general news.
- Skip: how-to guides (e.g. "How to prepare for board meetings with Claude"), career listings, events, upgrade pitches, and one-line digest bullets without enough detail for a standalone summary.

---

### SmartBrief on Connectivity (`connectivity@b2b.smartbrief.com`)

**Format:** Plain text email containing a headline list only — no summaries. Full summaries are on a web page behind a tracking URL (`r.smartbrief.com/resp/[token]`). That URL cannot be fetched directly (provenance restriction) and the tracking redirect makes it unsearchable.

**How to extract content:**
1. Read the headline list from `plaintextBody`.
2. For each relevant headline, run `WebSearch` to find the underlying story and source article.
3. Build the summary from the search results and underlying article if fetchable.
4. Skip headlines that are already covered by a higher-priority source.
5. Skip section headers that are not news stories: "ICYMI," "SmartBreak: Question of the Day," "SmartPod," "Friday Fun," "Leadership" (unless genuinely notable).

**Focus areas:** Tower/telco infrastructure, spectrum policy, FCC/NTIA regulation, AI in telecom, broadband policy. Skip consumer gadget or soft-feature items.

**Frequency:** Daily (weekdays).

---

## Cross-Day Coverage

The same story may appear in newsletters over several days as coverage evolves. If an entry with a matching or closely matching title already exists for yesterday or today, skip it. A genuinely new development or angle on an older story gets its own entry dated to the run date.

---

## Link Resolution Method (Step 7b: WSJ, Bloomberg, The Economist, FT)

Use `WebSearch` with the `site:` operator and the title words **unquoted**, e.g. `site:wsj.com Nvidia board reinstates Jensen Huang` (or `site:bloomberg.com`, `site:economist.com`, `site:ft.com`). Quoting the full title, or writing the domain as plain text instead of `site:`, often returns nothing.

Take the URL from the first result on the right domain whose title closely matches. For WSJ, check the slug against any decoded `trk.wsj.com` link from the email. If nothing matches, leave `link` null; never force a match.

---

## index.html Theming (permanent, added 2026-10-05)

index.html follows the device's light/dark setting via `@media (prefers-color-scheme: dark)` and a `<meta name="color-scheme" content="light dark">` tag. All colors are CSS variables on `:root`, redefined in the dark media block; category pills carry a `--pill-fg` inline variable that the dark block uses to derive dark-friendly pill colors.

Rules for every run:
- Step 9 must only replace the `// @@BRIEF_DATA_START@@` ... `// @@BRIEF_DATA_END@@` block. Never regenerate index.html from a template or an older copy, which would strip the dark-mode styles.
- If index.html styling is ever edited, keep new colors as CSS variables with a dark-mode value, and keep the `--pill-fg` variable in `pillHtml()`.
- Before pushing, if the remote index.html has a dark-mode block and the local copy does not, stop and report rather than pushing.

---

## Web Fetch Provenance Rule

`web_fetch` can only retrieve URLs that appeared in:
1. A user message
2. A prior `web_fetch` result
3. A `WebSearch` result

URLs extracted from email bodies (via `get_thread`) do **not** satisfy the provenance requirement. To fetch a URL found in an email:
- Run `WebSearch` using the URL or its slug/title as the query
- Once the URL appears in search results, it can be fetched with `web_fetch`

---

## Link Integrity Rule

**Never construct or guess article URLs.** A `link` field in an article entry must come from one of these sources only:
1. A URL explicitly present in the email `plaintextBody` (e.g. a "Read more" or article link)
2. A URL returned by `WebSearch` for that specific article
3. A URL returned by `web_fetch` for that specific article

If no valid URL is available from those sources, set `"link": null`. Do not construct plausible-looking slugs like `wsj.com/articles/fed-holds-rates-steady` — these are fabricated and will produce broken links.

**Indicators of a fabricated link:**
- Generic slug with no alphanumeric article ID (real WSJ/FT/The Information URLs always contain a hash or UUID, e.g. `wsj.com/articles/apple-chip-deal-69eb9370`)
- URL not seen in any `get_thread`, `WebSearch`, or `web_fetch` result during the current run
- URL constructed by combining a domain with a paraphrased headline

When in doubt, set `link` to null. The index.html rendering treats null links as plain text — no broken anchor tags will appear.

---

*Last updated: 2026-11-25 (source priority reordered; Bloomberg added; desktop-era instructions, Techpresso and The Neuron removed; Globe and Mail format updated to the daily plaintext Business Brief).*
