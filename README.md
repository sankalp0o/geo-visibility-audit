# GEO Visibility Audit

A free Claude Code skill that audits how visible your brand is in AI-powered search — **ChatGPT**, **Google AI Overviews**, and **Google AI Mode**.

No monthly subscription. No consultant billing hours. 2 minutes to set up, ~15 minutes to run, and you get a client-grade report with competitor visibility, citation sources, and a prioritized action list.

---

## What you get

A structured report covering:

- **Visibility scores** per platform (ChatGPT / Google AI Overview / Google AI Mode / Google Search) across purchase-intent, brand, and niche product queries
- **Competitor visibility table** — who AI engines actually recommend in your category (often not the list you have in your head)
- **Citation sources** — which domains AI engines cite as trusted sources in your space (= outreach targets)
- **Brand mention analysis** — whether AI describes your brand accurately, hedges, hallucinates, or ignores you
- **Prioritized action list** — immediate / short-term / medium-term steps to improve AI visibility

---

## Prerequisites

You need all three of these before running:

### 1. Claude Code

Install from [claude.com/code](https://claude.com/code). You need an active subscription.

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

Confirm it's connected by running `claude mcp list` — you should see `playwright` listed as `connected`.

### 3. A logged-in ChatGPT session

ChatGPT queries use your own logged-in session so responses include live web search (not training-data-only answers). You'll save this session once and reuse it.

**One-time setup:**

1. In any working directory (e.g. `~/geo-audits/`), start Claude Code: `claude`
2. Ask Claude: *"Open chatgpt.com with Playwright so I can log in, then save the session to `.chatgpt-session.json` in this directory."*
3. Log in manually in the Playwright browser window. Once you see your ChatGPT home page, tell Claude to save the session.
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

If you don't have a Claude Code subscription, or you'd rather someone run the audit for you and walk through the report live, DM me on LinkedIn: [linkedin.com/in/sankalp0o](https://www.linkedin.com/in/sankalp0o/).

If the report surfaces serious gaps and you want help closing them (content, entity work, outreach, listings), that's what [Geology](https://geology.dev) does.

---

## Changelog

- **2026-04-15** — Initial public release.

---

## License

MIT. Use it, fork it, run it for your clients, ship it as part of your agency's stack — no attribution required. If it saves you a consulting bill, a GitHub star is appreciated.
