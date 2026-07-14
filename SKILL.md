---
name: jem-research
description: Deep, verified, cited research that fills a user-defined table via engine-diverse parallel finder agents (Tavily + Firecrawl + web search) reconciled by a judge agent, then writes a dated, cited markdown report to disk. Use when the user wants a research table on a topic, asks to "research X into columns", "build a comparison table", "scan the market for", or gives a topic + the column headers they want filled. Asks clarifying questions when the topic, columns, or scope are underspecified.
---

# Deep Research — engine-diverse finders + judge

Turn a topic + a set of table columns into a **verified, cited** markdown table, saved to disk as a dated file.

**Architecture:** 3 parallel **finder agents** — each on a **different retrieval engine** and search angle, blind to each other → **discovery-closure loop** (sweeper rounds until dry) → one **judge agent** reconciles cell-by-cell → main session runs targeted re-checks on **conflicts, gaps, and single-source rows** → write. Prompt diversity alone is NOT independence: three agents over one search index inherit the same blind spots (2026-07-02 Dubai-dentists run — no amount of Tavily-only agents could surface a live Google Maps count, a top-10 clinic was missed by single-pass discovery, and a directory-verifiable rating shipped as "Unverified").

## Inputs

1. **Topic** — what to research (e.g. "voice-AI competitors to ElevenLabs").
2. **Output format** — the table column headers (e.g. `Company | Pricing | Funding | Key differentiator`).
3. **Additional details** — constraints: row count, region, time window, domains to include/exclude, must-have columns.

## Step 0 — Clarify if underspecified

Before searching, ask 2–3 targeted questions (one `AskUserQuestion` call) when ANY of these is unclear:
- **Topic too broad** — no angle, segment, or boundary ("research AI" → ask for the sub-area).
- **Columns ambiguous** — a header could mean several things ("Score" → of what, by whom?).
- **Scope undefined** — how many rows, what region, how recent.

If all three are clear, skip straight to Step 1. Do not ask just to ask.

## Step 1 — Plan the brief

Write a one-paragraph **research brief**: row-set definition (what qualifies as a row), the columns and what fact fills each, region/time constraints, and any user-supplied facts (these are ground truth — finders verify details around them; if evidence contradicts one, surface it to the user, never silently override either way).

**Map every column to a specialty finder** — the angle best placed to answer it (ratings/review counts → B; prices/official offerings → A; sentiment/red flags/dirt → C). This mapping drives the query budget in Step 2.

## Step 2 — Deploy engine-diverse finder agents

Spawn **3 finder agents in a single message** (so they run concurrently), each `general-purpose`, each given the same brief but a **distinct angle AND a distinct primary retrieval engine**:

| Finder | Angle | Primary retrieval |
|---|---|---|
| **A — official/primary** | Vendor & clinic sites, price pages, product docs. Best for prices, offerings, named people. | **Tavily MCP** — load via ToolSearch (`query: "tavily"`); `tavily_search` (`search_depth: "advanced"`, `max_results: 5–8`), `tavily_extract` (`extract_depth: "advanced"`) for exact figures. |
| **B — directories & aggregators** | Rating/review aggregators, industry directories, registers, comparison portals (Doctify, DrFive, Zavis-style register-backed directories, Trustindex, G2/Capterra for software). Best for ratings, review counts, rankings — beats self-reported numbers. | **Firecrawl** — API key in the `FIRECRAWL_API_KEY` environment variable (`curl -s -X POST https://api.firecrawl.dev/v2/search` / `/v2/scrape` with `Authorization: Bearer $FIRECRAWL_API_KEY`). Plus **WebFetch** aimed directly at known directory URL patterns. |
| **C — community & adversarial** | Reddit, forums, complaint boards, news. Explicitly hunts rows the other angles miss and negatives/red flags. | **WebSearch** (harness) + Tavily for Reddit specifically (Firecrawl is blocked there). |

> **Optional third index — SearXNG (self-hosted):** if you run a local [SearXNG](https://github.com/searxng/searxng) instance (a self-hostable metasearch engine that merges Brave/DuckDuckGo/Startpage/Google behind one JSON API), Finder C can use it as its primary index — query `http://<your-searxng-host>/search?q=<query>&format=json` via WebFetch or `curl` (default host `http://localhost:8080`). Health-check it first (`curl -s -m 5 http://<your-searxng-host>/ -o /dev/null -w '%{http_code}'`); if it's down, or you don't run one, fall back to WebSearch (the harness engine) and note the missing engine in the report. This is a nice-to-have extra retrieval index, not a requirement.
>
> **If the user would benefit from this third index but doesn't have SearXNG running, offer them the setup options rather than silently skipping it** — don't assume. Present the two paths and let them choose (or decline): **(1) Docker** — simplest and cross-platform (`searxng/searxng` image, one `docker run`, then enable the JSON format in `settings.yml`); **(2) bare-metal Python** — clone SearXNG into a venv, run `python -m searx.webapp`, keep it alive with a process manager (systemd on Linux, launchd on macOS). Both are written up in the README's *"Optional: self-host SearXNG"* section — point them there. Standing it up is their call; the skill works fine without it.

**Query budget per finder:** fill **every** column opportunistically from whatever pages you're already reading (cross-coverage is what creates the disagreement signal for the judge) — but run **dedicated queries only for your specialty columns** per the Step 1 mapping. A community agent burning searches on price pages is waste; a directory agent chasing Reddit threads is too.

Each finder's prompt must include: the brief + column mapping; its engine instructions from the table above; to run its own **discovery pass** (build its own candidate-row list — do NOT hand finders a fixed row list, the union of independent discovery is the point); and to return **structured output only** — one block per row: `row | cell values | source URL per cell | confidence (high/med/low) per cell`. A finder that can't fill a cell returns `not-found (searched: <queries tried>)`, never a guess.

**Name-collision guard** (all finders): for common entity names, always qualify with city/area and use exact-match phrasing — unrelated same-name entities elsewhere WILL pollute results and must not produce a false `not-found`.

## Step 3 — Discovery-closure loop (until dry)

When the finders return, union and dedupe their candidate rows. Discovery is not done until it's **measured** done:

1. Spawn one **sweeper agent** with the brief + the full union list, instructed: *"find qualifying rows NOT on this list"* — fresh tactics only (top-N listicles, category/area pages, map-pack queries, "competitor of X" searches, non-English sources if regional), any engine.
2. Sweeper returns ≥1 new qualifying row → add to union, run another sweeper round.
3. **Stop when a round returns zero new rows (dry). Hard cap: 3 sweeper rounds.** One dry round is sufficient closure evidence at table scale.

The report must state how discovery closed: `closed dry after N rounds` or `hit round cap — completeness not guaranteed`.

## Step 4 — Judge agent

Spawn **one judge agent** (`general-purpose`) with: the brief, all finder + sweeper outputs verbatim, and these instructions:

1. **Union the rows.** Any row found by ANY finder is a candidate. Dedupe (same entity, different naming). A row found by only one finder is tagged `single-source-row`.
2. **Reconcile cell-by-cell.** Agreement across ≥2 finders → accept. Divergence → **do not resolve by hierarchy; send to the re-check list** with both values and both URLs. Hierarchy (user-supplied facts > register/aggregator-backed > official-site self-reported > community; recency as tiebreak) applies only AFTER re-check evidence still disagrees.
3. **Build the re-check list** — three mandatory categories:
   - **(a) every conflicted cell**, with competing values + their URLs so the re-check can open primary evidence;
   - **(b) suspicious `not-found`s** — where the finder's query log suggests a missed angle (name collision, wrong site type), with the specific query/domain most likely to resolve it;
   - **(c) key columns of every `single-source-row`** — a targeted directory/aggregator lookup each, so no self-reported number ships merely because only one finder saw the row.
4. Return: final row list, per-cell values + winning source, **conflict log**, the **re-check list**, and rows/cells to mark ⚠️.

The judge does not search — it only analyzes. This keeps it adversarial rather than anchored on its own findings.

## Step 5 — Targeted re-checks (main session)

Run the judge's re-check list yourself. **Resolve conflicts by opening the disputed URLs** — `tavily_extract` (`extract_depth: "advanced"`), or Firecrawl `/v2/scrape` for JS-heavy/protected pages — and reading the primary evidence; fall back to the source hierarchy only if the pages themselves still disagree (then ⚠️ with both values). For single-source rows, hit a directory/aggregator directly. **"Unverified" may only land in a cell after this pass also fails** — an unresolved cell must show it was hunted twice (finder + re-check), never defaulted.

**Degraded mode:** if the Agent tool is unavailable or the task is trivial (≤3 rows, ≤3 columns), run single-agent with second-source verification per cell — and say in the report that the multi-agent layer was skipped.

## Step 6 — Write the report to the vault

Write to your research output directory as `<YYYY-MM-DD>-<slug>.md` — default to `./research/<YYYY-MM-DD>-<slug>.md` in the current working directory, or point it at a knowledge vault (e.g. an Obsidian `<vault>/research/` folder) if you keep one. Today's date is in the session context; never call `date` for randomness. Use this layout:

```md
# <Topic>
*Researched <YYYY-MM-DD> · Deep-research skill (engine-diverse finders + judge)*

**Scope:** <row set, region, time window, constraints>
**Discovery closure:** <closed dry after N rounds | hit round cap>

| <col1> | <col2> | ... | Source |
|--------|--------|-----|--------|
| ...    | ...    | ... | [n](url) |

## Notes & confidence
- ⚠️ <conflicts (both values + which won and why), single-source rows, cells unresolved after re-check>

## Sources
1. [title](url) — what it backed
```

Each row's `Source` column links the primary citation; the numbered Sources list covers everything used.

## Step 7 — Summarize inline

After writing, report back: the vault file path, the table (or a trimmed version if large), a 2–3 line takeaway, any ⚠️ flags, **the judge's conflict count** (e.g. "finders disagreed on 3 cells; re-checks resolved 2, flagged 1"), and how discovery closed. Do not bury the conflicts.

## Rules

- **Cite or omit** — no cell without a source; unverifiable facts are marked `Unverified`, never invented.
- **Unverified is earned, not defaulted** — it requires a failed finder pass AND a failed targeted re-check.
- **Conflicts are resolved by evidence, not hierarchy** — open the disputed page; hierarchy is the tiebreak of last resort.
- **Discovery is parallel, independent, and closed** — never seed all finders with one agent's row list; never stop discovery without a dry sweeper round (or an explicit cap note).
- **Engines are assigned for independence** — don't collapse all finders onto Tavily because it's convenient; the engine split is the point.
- User-supplied facts are ground truth; if a finder contradicts one, surface it to the user rather than silently overriding either way.
- Write only to the research output directory; never modify the source material you're researching from. If you keep a knowledge vault with a source-vs-compiled split (e.g. an Obsidian `raw/` vs `wiki/`), respect it — write only to the compiled/output side.
- Tavily rate limit ~20 req/min — batch, don't spam retries. Firecrawl credits are finite — scrape specific pages, don't crawl whole sites.
- If neither Tavily MCP nor the Firecrawl key is available, say so — this skill does not invent a fallback.
