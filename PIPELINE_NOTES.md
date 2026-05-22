# My News Brief — Pipeline Notes

This file documents special handling rules, known quirks, and source-specific extraction patterns for the daily pipeline. Read this before running the pipeline.

---

## Email Extraction: General Rules

The Gmail `get_thread` tool returns `plaintextBody` and a short `snippet`. For most newsletters, the plaintext body contains the full article content and is sufficient for extraction.

**Never skip a newsletter solely because it has no outbound hyperlink to a published article.** If `plaintextBody` contains full narrative prose (column text, analysis, multi-paragraph summaries), extract content directly from the email body. The absence of a clickable link is not a reason to skip — it is simply a reason to set `"link": null` on the resulting entries.

The three cases to distinguish:
1. **Full prose in body** (e.g. The Information Weekend, Dealmaker columns) → extract directly; `link` = null unless a real URL appears elsewhere in the body.
2. **Body is a URL only** (e.g. Techpresso) → use WebSearch to surface the URL, then web_fetch.
3. **Body is a headline list** (e.g. SmartBrief) → WebSearch each headline.

When a newsletter email has **no `plaintextBody`** or the plaintext body contains **only a URL and no article content**, follow the source-specific rules below.

---

## Source-Specific Handling

### Techpresso (`techpresso@dupple.com`)

**Format:** Beehiiv-hosted newsletter. The plaintext version of the email is intentionally stripped to just a single archive URL in the form:
`https://archive.techpresso.co/p/[slug]`

There is no article content in the email body.

**How to extract content:**
1. Pull the archive URL from `plaintextBody`.
2. The archive URL cannot be fetched directly via `web_fetch` due to provenance restrictions — it was not provided by the user or a prior `web_fetch`/`web_search` result.
3. Use `WebSearch` with the slug or subject line (e.g. `site:archive.techpresso.co [topic] 2026`) to surface the URL in search results.
4. Once the URL appears in `WebSearch` results, it enters the provenance set and `web_fetch` can retrieve it.
5. If the archive page is not indexed by search engines, use the email subject line and snippet to identify the main story and secondary bullet topics, then search for each topic independently (e.g. `"Google Fitbit Air launch May 2026"`).

**Format of content:** Each issue has one main story (usually the email subject) and several secondary items, each with a title, a link, and 2–4 bullet points summarizing the topic.

**Known issue (as of May 2026):** The Techpresso archive domain (`archive.techpresso.co`) is not reliably indexed by search engines. The fallback is to identify topics from the subject line and snippet, search for those topics, and reconstruct coverage from primary sources.

---

### The Globe and Mail Business Brief (`noreply@globeandmailnewsletters.com`)

**Format:** HTML-only email. `get_thread` returns **no `plaintextBody`** regardless of the `messageFormat` parameter — the Gmail tool does not expose the HTML body. The `snippet` field is boilerplate header text only ("Trouble viewing this email...") and contains no article content.

**Important:** The email body *does* contain full readable article text in its HTML version. The limitation is purely on the tool side — `get_thread` does not return HTML body content.

**Root cause:** The Gmail MCP connector (`get_thread`) only returns `plaintextBody`. It does not expose the HTML body, and this cannot be worked around by changing the `messageFormat` parameter — explicitly passing `FULL_CONTENT` was tested and still returns no body for this email.

**How to extract content (in order of preference):**
1. **With Computer Use enabled (recommended for automation):** Navigate to Gmail in a browser, open the email, and read the HTML content directly. This is the only fully automated solution. Enable at Settings → Desktop app → Computer use.
2. **For automated runs without Computer Use:** Search for the published article: `site:theglobeandmail.com "business brief" "five files" [month] [year]`. URL pattern: `https://www.theglobeandmail.com/business/article-business-brief-five-files-to-follow-this-week-[N]/`. The site is **paywalled** — `web_fetch` returns empty — but `WebSearch` snippets typically recover 2–3 of the 5 files. Cross-reference with The Logic newsletters from the same week for overlapping Canadian business stories.
3. **Do not ask the user to paste the content manually.** This is not acceptable for a scheduled pipeline.

**Newsletter format:** Weekly (Monday), 5 numbered sections. Each covers a Canadian or macro business story with 3–5 short paragraphs. Author: Stefanie Marotta.

---

### The Information (`hello@theinformation.com` / `info@theinformation.com`)

**Format:** Plaintext body available and extractable.

**Newsletter types:**
- **The Information AM** (`hello@theinformation.com`, arrives ~11am ET): numbered list of articles with full summaries. Primary daily extraction source.
- **The Information Briefing** (`info@theinformation.com`, arrives ~midnight ET): nightly column by Martin Peers, written in prose. Dateline is the *prior* day (e.g. an email arriving midnight May 8 covers May 7 news). Treat as supplementary/also_covered_by for the prior day's articles. Do not create a new entry dated to the email's arrival date.
- **The Information Weekend** (`hello@theinformation.com`, subject: "Weekend: [headline]", arrives Saturday): weekly column by Abram Brown. Full plaintext body — no link needed. Extract the main column story (typically 4–8 paragraphs of analysis on one theme) plus any secondary items that are distinct business/tech facts (e.g. funding milestones, product announcements, notable deals mentioned in passing). Skip entertainment recommendations (books, TV, film) and cultural observations with no business relevance. Date entries to the email's arrival date (Saturday). Link = null unless a real URL appears in the body.
- **The Information Dealmaker** (`hello@theinformation.com`, subject: "Dealmaker: [headline]"): weekly VC/funding column by Katie Roof. Full plaintext body. Extract the main column thesis as a single entry. Date to email arrival date. Link = null.
- **The Information article alerts** (`hello@theinformation.com`, subject is the article headline): single-article promotional emails. The `plaintextBody` contains 1–3 paragraphs of article text plus a real article URL (e.g. `https://www.theinformation.com/articles/[slug]`). Extract the article and use the URL as the link — these are valid provenance-compliant URLs because they appear in `get_thread` output... **but** per the Web Fetch Provenance Rule, URLs from `get_thread` cannot be fetched directly. However, the article summary in the body is usually sufficient to write the entry without fetching. Use the URL as the `link` field directly (it was present in the email and can be cited as the source).
- **Applied AI** (`hello@theinformation.com`, subject: "Applied AI: [headline]"): in scope. Covers enterprise AI adoption, AI-driven business model shifts, and the impact of AI on software procurement and vendor relationships. Format is a short analytical piece on one main story. Extract 1 entry per issue; focus on the core business or strategic insight. Set `source` to "The Information" and `newsletter` to "Applied AI". Link = null unless a real article URL appears in the body.
- **Digest / recap emails** (`hello@theinformation.com`): skip. These are promotional round-ups of already-covered stories (e.g. "Top Posts Today," "This Week's Most Popular Stories," "Sunday recap," "Before X was everywhere, it was in The Information"). Identify by subject patterns like "recap," "most popular," "before X was everywhere," or "don't miss."

---

### The Neuron (`theneuron@newsletter.theneurondaily.com`)

**Format:** Beehiiv-hosted. Unlike Techpresso, The Neuron's plaintext body contains the full article content — sections include a featured story, "Around the Horn" (bullet list of secondary stories), "Treats to Try" (product launches), and "Intelligent Insights" (opinion/analysis links).

**Extraction guidance:**
- Extract 4–6 articles per issue: typically the main story, 2–3 "Around the Horn" items with enough detail for a standalone summary, and 1–2 "Treats to Try" items if they represent meaningful product launches.
- Skip "AI Skill of the Day," podcast promotions, and "A Cat's Commentary."

---

### WSJ Newsletters (`access@interactive.wsj.com`)

**Format:** Multiple distinct newsletters from the same sender. Identify by subject line:
- **WSJ Morning Download** — main daily CIO/tech newsletter. Full plaintext body.
- **WSJ Pro Cybersecurity** — daily cybersecurity newsletter. Full plaintext body.
- **WSJ CEO Brief** — daily leadership newsletter. Extract 1–2 stories max; focus on macro or operational insights (e.g. shipping disruptions, CEO strategy).
- **WSJ Future of Everything / conference newsletters** — skip. These are event-marketing emails.
- **WSJ CMO Today** — in scope. Daily marketing and advertising newsletter covering brand strategy, marketing technology, AI in advertising, consumer behavior, and media industry trends. Full plaintext body. Extract 1–2 stories per issue; prioritize stories with quantified claims, strategic business implications, or AI/technology angles. Skip pure brand campaign announcements and editorial calendar reminders. Set `source` to "Wall Street Journal" and `newsletter` to "WSJ CMO Today". Category is typically "Business & Strategy" or "Technology & AI" depending on the story angle.

---

### The Logic (`anna@thelogic.co` / `news@thelogic.co`)

**Format:** Two types:
- **Individual article alerts** (sender: `news@thelogic.co`): single article per email. Plaintext body available; extract if the article is a unique Canadian business story not covered elsewhere.
- **Weekly digest** (sender: `anna@thelogic.co`, subject: "Don't miss these stories from the past week"): skip. These are recaps of already-published content.

---

### The Rundown Tech (`crew@technews.therundown.ai`) and The Rundown AI (`news@daily.therundown.ai`)

**Format:** Both are daily newsletters with full plaintext bodies. The two are distinct newsletters from the same publisher (therundown.ai) with different sender addresses and editorial focus:
- **The Rundown Tech** — consumer and enterprise technology, hardware, product launches, market trends.
- **The Rundown AI** — frontier AI research, model releases, AI policy, applied AI in science and industry.

**Extraction guidance:**
- Extract 2–3 stories per issue. Focus on stories with quantified claims or announcements not already covered by WSJ or The Information.
- Skip: sponsor callouts, "tool of the day" sections, and brief one-line items without enough detail for a standalone summary.
- Links are often present in the body as direct source URLs. Use these as the `link` field if they appeared in `get_thread` output and satisfy the Link Integrity Rule (real article URLs with hash or path components, not constructed slugs).

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
- Prioritize stories not already covered by WSJ or The Information — this newsletter often surfaces consulting firm reports (Deloitte, McKinsey, Bain, Accenture, Google Cloud) before they reach general news.
- Skip: how-to guides (e.g. "How to prepare for board meetings with Claude"), career listings, events, upgrade pitches, and one-line digest bullets without enough detail for a standalone summary.

---

### SmartBrief on Connectivity (`connectivity@b2b.smartbrief.com`)

**Format:** Plain text email containing a headline list only — no summaries. Full summaries are on a web page behind a tracking URL (`r.smartbrief.com/resp/[token]`). That URL cannot be fetched directly (provenance restriction) and the tracking redirect makes it unsearchable.

**How to extract content:**
1. Read the headline list from `plaintextBody`.
2. For each relevant headline, run `WebSearch` to find the underlying story and source article.
3. Build the summary from the search results and underlying article if fetchable.
4. Skip headlines that are already covered by higher-priority sources (WSJ, The Information, etc.).
5. Skip section headers that are not news stories: "ICYMI," "SmartBreak: Question of the Day," "SmartPod," "Friday Fun," "Leadership" (unless genuinely notable).

**Focus areas:** Tower/telco infrastructure, spectrum policy, FCC/NTIA regulation, AI in telecom, broadband policy. Skip consumer gadget or soft-feature items.

**Frequency:** Daily (weekdays).

---



- **Cross-day duplicates:** The same story (e.g. the OpenAI-Broadcom chip deal) may appear in newsletters over multiple days as coverage evolves. Each day's newsletter coverage is a distinct entry dated to that day. Do not suppress a May 8 entry just because the same story has a May 11 entry.
- **The Information Briefing is May N–1 content:** Always date Briefing articles to the prior calendar day, not the email arrival date.
- **Techpresso and The Neuron often cover the same tech stories** as WSJ/The Information but from a different angle. Use the higher-priority source as primary and add the other as `also_covered_by`.

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

*Last updated: 2026-05-22 (WSJ CMO Today and Applied AI added to scope; digest/recap skip rule clarified)*
