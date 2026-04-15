# GEO Visibility Audit

A Claude Code skill that checks how your brand shows up in ChatGPT, Google AI Overviews, and Google AI Mode.

Set up in 2 minutes, run in ~15 minutes. Output is a markdown report you can read, share, or hand to a client.

---

## What you get

A single markdown report with:

- Visibility scores per platform (ChatGPT / Google AI Overview / Google AI Mode / Google Search) across purchase-intent, brand, and niche product queries
- A competitor table — who AI engines actually name in your category
- A citation-source table — which domains AI engines cite in your space
- Brand mention analysis — whether AI describes you accurately, hedges, hallucinates, or doesn't know you
- A prioritized action list (immediate / short-term / medium-term)

Raw data (Playwright snapshots, ChatGPT conversation JSON, brand context, locked prompt set) is kept alongside the report so you can re-run or diff over time.

---

## Prerequisites

You need all three of these before running:

### 1. Claude Code

Install from [claude.com/code](https://claude.com/code).

**Subscription tier**: The audit runs for ~15 minutes and burns through a lot of tokens on Playwright snapshots and ChatGPT response parsing. A **Max subscription** is strongly recommended — Pro can run out of quota mid-audit.

Verify it's working:
```bash
claude --version
```

### 2. Playwright MCP

The skill queries Google and ChatGPT via Playwright MCP (a browser automation server that plugs into Claude Code).

Install:
```bash
claude mcp add playwright npx @playwright/mcp@latest
```

Confirm it's connected by running `claude mcp list` — you should see `playwright` listed as `connected`. **You may need to restart your Claude Code session for the new MCP to take effect.**

### 3. A logged-in ChatGPT session

ChatGPT queries use a logged-in session so responses include live web search (not training-data-only answers). You'll save this session once and reuse it.

**Use a separate ChatGPT account** — not your main one. The audit fires 10 queries in quick succession, and ChatGPT sometimes throttles or soft-blocks accounts it sees heavy automation on (especially free accounts). Create a free account on a different email address and log into that for the audit.

**One-time setup:**

1. Start Claude Code: `claude`
2. Ask Claude: *"Open chatgpt.com with Playwright so I can log in, then save the session to `.chatgpt-session.json` in this directory."*
3. Log in manually in the Playwright browser window. Once you see the ChatGPT home page, tell Claude to save the session.
4. Claude will save a `.chatgpt-session.json` file in the current directory. Keep this file — the audit reuses it.

The session lasts weeks. If it expires, repeat the step — the skill will tell you if re-login is needed.

---

## Install the skill

1. Clone or download this repo:
   ```bash
   git clone https://github.com/sankalp0o/geo-visibility-audit.git
   ```

2. Copy `SKILL.md` into your user skills directory:
   ```bash
   mkdir -p ~/.claude/skills/geo-visibility-audit
   cp geo-visibility-audit/SKILL.md ~/.claude/skills/geo-visibility-audit/SKILL.md
   ```

3. Verify Claude Code can see it — start `claude` and run `/skills`. You should see `geo-visibility-audit` in the list.

That's it. The skill is now available in every Claude Code session on this machine.

---

## Run an audit

1. **Create a working directory** for this audit (separate from the cloned repo — reports and raw data land here):
   ```bash
   mkdir -p ~/geo-audits/<your-domain>
   cd ~/geo-audits/<your-domain>
   ```

2. **Copy your ChatGPT session file** into this directory (or any ancestor):
   ```bash
   cp ~/path/to/.chatgpt-session.json .
   ```

3. **Start Claude Code**:
   ```bash
   claude
   ```

4. **Invoke the skill** by asking for the audit in plain language:
   ```
   Run a GEO audit for yourdomain.com
   ```

   Claude will route to the `geo-visibility-audit` skill automatically.

5. **Walk away for ~15 minutes**. The skill runs through six phases:

   | Phase | What happens | Approx time |
   |-------|--------------|-------------|
   | 1. Brand context | Fetches your homepage + runs one branded Google search to understand what you sell, who you compete with, and whether there are any name conflicts | 1 min |
   | 2. Entity readiness | Checks Wikipedia, Crunchbase, G2, Trustpilot, marketplaces | 2 min |
   | 3. Question generation | Writes 10 targeted questions grounded in your brand context — 4 purchase-intent, 3 branded, 3 niche-product | 1 min |
   | 4. Platform queries | Runs all 10 questions against Google (Search + AI Overview), Google AI Mode (branded + geo queries only), and ChatGPT | 8–10 min |
   | 5. Analysis | Extracts brand mentions, competitors, citations; classifies finding types | 2 min |
   | 6. Report | Writes the client-ready report + internal methodology notes | 1 min |

6. **Find your report** at:
   ```
   seo-research/<your-domain>/geo-audit-report.md
   ```

   Raw data (snapshots, JSON payloads, brand context, locked prompt set) lives under `seo-research/<your-domain>/geo-audit/`.

---

## Reading the report

The report has seven sections. Read them in this order:

1. **Executive Summary** — current visibility level, biggest threat, one immediate action
2. **Brand Context** — what the audit understood about you (sanity check — if this is wrong, re-run after fixing your homepage / About page)
3. **Purchase-Intent Queries** — what AI recommends when buyers are ready to purchase in your category. If your brand is absent and competitors appear, that's the displacement gap you're solving for.
4. **Brand Queries** — what AI says when asked about your brand directly. Hedged language ("appears to be", "you should verify") means you have no third-party coverage to cite. Hallucinated facts usually mean a same-name conflict with another company.
5. **Niche Product Queries** — what AI surfaces when your products are described by attributes, not brand name. Gap here = content/listing gap.
6. **Competitor Visibility** — who dominates your category's AI surface area. Your highest-visibility competitor is your content benchmark.
7. **Citation Sources** — which domains AI trusts in your space. Every row is a concrete outreach target.

The **Recommended Actions** section at the bottom consolidates the above into a prioritized list — that's your backlog.

---

## Troubleshooting

**"Playwright is not connected"**
Run `claude mcp list` — if Playwright is missing or disconnected, re-add it with `claude mcp add playwright npx @playwright/mcp@latest` and restart Claude Code.

**"ChatGPT session expired"**
The skill will tell you during Phase 4 if the session is no longer valid. Re-do the one-time login step (see Prerequisites → ChatGPT session) to refresh `.chatgpt-session.json`.

**"No AI Overview showed up for some queries"**
Not every query triggers an AI Overview — that's Google's behavior, not a bug. The skill notes it and moves on. If more than half your queries show no AIO, your category may be one Google doesn't trigger AIOs for (typically highly commercial or regulated).

**"The report mentions a company that isn't me"**
That's a disambiguation finding — there's another brand with a similar name and AI systems are conflating you. The report traces it to the source URL. Fix candidates: schema markup on your homepage, a clearer About page, a Wikipedia entry, or a press release that anchors your entity identity.

**"AI Mode results look non-US / wrong region"**
Playwright's browser sometimes picks up your local geolocation and overrides URL params. The skill adds `gl=us&hl=en&cr=countryUS` to force US results — if it's still wrong, note the region in your re-run or ask Claude to use a VPN-backed Playwright context.

**"The audit asked me a question mid-run"**
The skill will pause and ask rather than guess on things like ambiguous brand positioning or region. Answer in the chat and it'll continue.

---

## Want help running it?

If you don't have Claude Code, or you'd rather someone run the audit for you and walk through the report live, reach out:

- Email: [sankalp@getgeology.com](mailto:sankalp@getgeology.com)
- Web: [getgeology.com](https://getgeology.com)

If the report surfaces serious gaps and you want help closing them (content, entity work, outreach, listings), that's what Geology does.

---

## Changelog

- **2026-04-15** — Initial public release.

---

## License

MIT. Use it, fork it, run it for your clients, ship it as part of your agency's stack — no attribution required. If it saves you a consulting bill, a GitHub star is appreciated.
