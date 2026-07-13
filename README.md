# research-council

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill for **deep, verified, cited research** that fills a table you define — column by column — using *engine-diverse* parallel research agents reconciled by an adversarial judge.

> Registers inside Claude Code as the skill **`jem-research`** (that's the `name:` in `SKILL.md`) — so install it into a `jem-research/` directory, per the command below.

You give it a topic and the columns you want (`Company | Pricing | Funding | Key differentiator`); it hands you back a markdown table where **every cell has a source** and conflicts are surfaced, not silently resolved.

## Why it's different

Most "research agent" setups run several agents over the *same* search index with different prompts. That's not independence — three agents over one index inherit the same blind spots. This skill assigns each finder a **different retrieval engine** *and* a **different search angle**, so their coverage genuinely diverges. A judge agent then reconciles their findings cell-by-cell and builds a re-check list of every conflict, suspicious gap, and single-source claim, which the main session verifies against primary evidence before anything ships.

The result: fewer confident-but-wrong cells, and an honest "Unverified" that's *earned* (a finder pass **and** a targeted re-check both failed) rather than defaulted.

## How it works

| Phase | What happens |
|---|---|
| **0 · Clarify** | Asks 2–3 targeted questions if the topic, columns, or scope are underspecified. |
| **1 · Brief** | Defines what qualifies as a row, maps each column to the finder best placed to answer it. |
| **2 · Finders** | Spawns 3 parallel finder agents, each on a **distinct engine + angle**: (A) official/primary via Tavily, (B) directories & aggregators via Firecrawl, (C) community & adversarial via web search. Blind to each other. |
| **3 · Discovery-closure loop** | Sweeper agents hunt for rows the finders missed, round after round, until a round comes back **dry** (hard cap: 3). Completeness is *measured*, not assumed. |
| **4 · Judge** | One judge agent unions the rows, reconciles cells, and builds a re-check list: every conflict, every suspicious `not-found`, every single-source row. It analyzes only — it doesn't search, so it stays adversarial. |
| **5 · Re-checks** | The main session opens the disputed URLs and resolves conflicts against primary evidence. |
| **6 · Write** | Emits a dated, cited markdown report. |
| **7 · Summarize** | Reports the file path, the table, key takeaways, ⚠️ flags, the conflict count, and how discovery closed. |

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/jthoma74/research-council ~/.claude/skills/jem-research
```

(Or drop it into a project's `.claude/skills/` directory to scope it to one repo.)

Claude Code auto-discovers it. Trigger it by asking for research in the shape it expects — *"research the top voice-AI vendors into Company | Pricing | Funding | Differentiator"* — or invoke it by name.

## Requirements

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — the skill orchestrates subagents via the Agent/Task tool, so it needs the agent-spawning harness (it has a documented single-agent degraded mode if that's unavailable).
- **[Tavily MCP](https://docs.tavily.com/)** — Finder A's primary engine. Configure it as an MCP server in Claude Code.
- **Firecrawl API key** — Finder B's engine. Set `FIRECRAWL_API_KEY` in your environment. Get a key at [firecrawl.dev](https://www.firecrawl.dev/).
- **Web search** — Finder C uses the harness's built-in `WebSearch`.
- *(Optional)* **[SearXNG](https://github.com/searxng/searxng)** — if you self-host a SearXNG instance, Finder C can use it as an extra retrieval index. Purely a nice-to-have; the skill falls back to `WebSearch` without it.

If neither Tavily nor Firecrawl is available, the skill tells you rather than inventing a fallback.

## Output

By default the report lands at `./research/<YYYY-MM-DD>-<slug>.md` in your working directory. If you keep a knowledge vault (e.g. Obsidian), point it at a `<vault>/research/` folder instead — the skill respects a source-vs-compiled split and only ever writes to the output side.

Each row's `Source` column carries its primary citation; a numbered Sources list at the bottom covers everything used; ⚠️ callouts flag conflicts, single-source rows, and cells left unverified after re-check.

## Honest limits

- **Only as good as the open web + your engines.** Paywalled, offline, or un-indexed facts stay `Unverified` — by design, not defaulted.
- **Table-scale, not survey-scale.** Discovery closes after one dry sweeper round (cap 3); for exhaustive census work you'd want more rounds.
- **Rate limits are real.** Tavily ~20 req/min; Firecrawl credits are finite — the skill scrapes specific pages rather than crawling whole sites, but a very wide table still costs API calls.
- **Judgment stays with you.** The skill surfaces conflicts and confidence; it doesn't pretend the last uncertainty is gone.

## License

MIT — see [LICENSE](LICENSE).
