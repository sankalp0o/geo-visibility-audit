---
name: geo-visibility-audit
description: |
  Run a GEO (Generative Engine Optimization) audit for a domain. Detects regime,
  builds brand context, generates 10 targeted questions (bottom-funnel + branded +
  ultra-niche), queries Google and ChatGPT, analyzes brand mentions and citation
  sources, and produces a client-ready report. Use when asked to "run a GEO audit
  for X", "check AI visibility for X", or "how does X appear on AI search".
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - mcp__playwright__browser_navigate
  - mcp__playwright__browser_snapshot
  - mcp__playwright__browser_click
  - mcp__playwright__browser_type
  - mcp__playwright__browser_evaluate
  - mcp__playwright__browser_wait_for
  - mcp__playwright__browser_run_code
  - mcp__playwright__browser_network_requests
  - mcp__playwright__browser_tabs
---

# GEO Audit

Run a GEO (Generative Engine Optimization) audit for: **$ARGUMENTS**

## What this audit answers

**"When users ask AI-powered search engines questions in this brand's space, does the brand appear — and if not, who does?"**

For each of 10 generated questions, you will query three AI surfaces — **all via Playwright MCP**:
- **Google Search + AI Overview** — standard Google search (`google.com`, no `udm` param). The snapshot captures organic results, AI Overviews, shopping panels, and People Also Ask — everything on the page.
- **Google AI Mode** (`udm=50`) — run on **branded queries + geo-qualified ultra-niche queries only** (typically 3–4 of the 10). AI Mode is opt-in and surfaces a richer response including the local Google Business Profile layer, which AI Overviews do not.
- **ChatGPT** — live ChatGPT with web search enabled. Uses a saved authenticated session (`.chatgpt-session.json` at the working directory root). Falls back to anonymous mode if session is expired, but anonymous mode may not trigger web search on all queries.

**Do NOT use ScrapingDog, Oxylabs, or any scraping API.** All queries go through Playwright MCP. If Playwright has issues (browser closed, rate limited), ask the user before falling back to any alternative.

---

## Setup

**ChatGPT session**: A saved Playwright session lives at `.chatgpt-session.json` in the current working directory (gitignored). This contains auth cookies for chatgpt.com so ChatGPT queries use live web search. If the session is expired, the user will need to re-login via Playwright and re-save it.

Create output directories:
```bash
mkdir -p seo-research/<domain>/geo-audit/raw
```

---

## Step 1: Detect Regime + Build Brand Context

Do both in one pass — fetch the homepage, then run one branded Google search.

### 1a. Fetch the homepage

```bash
curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" "https://<domain>"
```

If that fails (Cloudflare, JS-rendered), try the Playwright MCP or `http://`. Extract title, meta description, H1/H2s, key body copy (first 2000 chars), and any About/product description text.

### 1b. Run one branded Google search via Playwright

Navigate to `https://www.google.com/search?q=<brand+name>&gl=us&hl=en&cr=countryUS` via Playwright MCP. Wait 4 seconds, then take a snapshot.

Scan organic results for: `wikipedia.org`, `crunchbase.com`, `linkedin.com`, `g2.com`, `trustpilot.com`.

**Also scan for brand name disambiguation issues**: if results for a similar-sounding brand name appear (e.g., a different company with the same word in its name), note it. This is a hallucination risk — AI systems may conflate the two brands. Record any such conflicts in `brand_context.md`.

### 1c. Save brand_context.md

Save to `seo-research/<domain>/geo-audit/brand_context.md`:

```markdown
## Brand Identity
- Brand name + aliases
- One-line positioning (homepage hero + meta desc — note if they conflict)
- Products/services: specific names, not generic
- Pricing tier (infer from copy: budget / mid / premium)
- Location, founding year (if findable)

## Ground-Truth Facts
Verifiable claims to use for hallucination detection later:
- Specific product names and prices
- Certifications or awards stated on-site
- Key differentiators ("fallen areca leaves", "zero chemical processing", etc.)
- Founders or team members if public

## Regime
- Classification: Established / New
- Reasoning: [evidence — entity footprint, site size, press mentions]
- Entity footprint: Wikipedia (Y/N) · Crunchbase (Y/N) · G2 (Y/N) · Trustpilot (Y/N) · LinkedIn (Y/N)

## Disambiguation Risk
- Are there other companies/products with similar or identical names?
- If yes: describe the conflict and which AI surfaces are most at risk of conflation
- Source: [which organic result revealed this]

## Category & Competitors
- Category: [the space this brand competes in]
- Subcategory: [the more specific niche — this drives question generation]
- Competitors visible on homepage or in branded search results
- Brand's stated differentiation
```

**Regime definitions:**
- **Established**: recognisable entity footprint, meaningful press/review coverage, brand is known in its category
- **New / low-authority**: no Wikipedia/Crunchbase entry, minimal third-party coverage — *most domains fall here*

**Why this matters for the audit:**
- **Established**: zero AI mentions are a real gap — the audit finds where leverage is
- **New**: zero AI visibility is the *expected baseline*, not a finding. The competitive landscape map and action list are the deliverables.

---

## Step 2: Check Entity Readiness

Quick checks — 2 minutes, directly feed recommendations.

```bash
# Wikipedia API (direct — no scraping needed)
curl -s "https://en.wikipedia.org/w/api.php?action=query&titles=<brand+name>&format=json"
```

For Crunchbase, G2, Faire, and Trustpilot checks, use Playwright to run site-scoped Google searches:
```
Navigate: https://www.google.com/search?q=site:crunchbase.com+"<brand+name>"&gl=us&hl=en
Navigate: https://www.google.com/search?q=site:g2.com+"<brand+name>"&gl=us&hl=en
Navigate: https://www.google.com/search?q=site:faire.com+"<brand+name>"&gl=us&hl=en
Navigate: https://www.google.com/search?q=site:trustpilot.com+"<brand+name>"&gl=us&hl=en
```

Take a quick snapshot of each to check for results. These can be done rapidly (navigate + 2s wait + check title/snippet).

Record findings in `brand_context.md` under Entity Footprint. Note: missing marketplace listings (Amazon, Etsy, Faire) are often the most actionable finding for product brands.

---

## Step 3: Generate 10 Questions

Generate exactly 10 questions across three types, grounded in `brand_context.md`. Every question must be traceable to a specific field in brand context — do not invent angles.

### Type 1 — 4 × Bottom-of-funnel / transactional

**Must be purchase or comparison intent**: "buy X", "best X brand for Y", "X wholesale", "order X bulk"

Critical rules:
- **Operate at the subcategory level**, not the broad category. Broad category queries are dominated by large incumbents. If the brand sells palm leaf plates, ask about *palm leaf plates* — not "eco-friendly tableware". Subcategory specificity surfaces the real competitive set.
- **No informational queries** ("how does X work", "what is X made of") — these rarely trigger brand mentions or competitor recommendations.
- For **established** brands: include at least one comparison query ("X vs [competitor]")

### Type 2 — 3 × Branded / brand-adjacent

These reveal the "AI knowledge gap" — whether AI knows the brand exists and what it knows.

1. **Review query**: "[Brand name] [product] review" — does AI have any review content to draw from?
2. **Trust query**: "is [domain] legit" or "is [brand] good quality" — does AI have enough coverage to vouch for the brand, or does it hedge?
3. **Indexing check**: "site:[domain]" — send to **ChatGPT only** (Google shows indexed pages natively in organic; ChatGPT will interpret this as a question about the domain, revealing what the model knows about the brand's content)

**What to look for in branded responses:**
- **Full recognition**: AI describes the brand accurately and positively — brand has editorial coverage
- **Hedged recognition**: AI says "appears to be" / "you should check reviews yourself" — brand exists in AI's world but has no third-party coverage to cite. Most common state for new brands.
- **Hallucinated identity**: AI describes the brand confidently but with wrong facts (wrong business type, wrong products, wrong name) — usually caused by a same-name disambiguation conflict. Flag immediately and trace to the source.
- **No recognition**: AI has no information about the brand at all

### Type 3 — 3 × Ultra-niche without naming the brand

Describe the brand's *specific product attributes* — origin, material, process, use case, supply chain — without using the brand name. These should be narrow enough that only a handful of suppliers worldwide could be the answer.

Purpose: if the brand is one of those suppliers but AI doesn't surface it, that's a concrete content/listing gap. If AI surfaces competitors instead, those competitors are the placement targets.

**Mark at least one ultra-niche question as geo-qualified** (include a location) if the brand is geographically concentrated — this question will be run on Google AI Mode as well, since AI Mode surfaces the local GBP layer that AI Overviews do not.

Example: "handcrafted areca palm leaf dinner plates shipped to United States" rather than "eco tableware online"

---

Save to `seo-research/<domain>/geo-audit/questions.json`:
```json
{
  "domain": "<domain>",
  "regime": "new | established",
  "questions": [
    {
      "id": 1,
      "type": "bottom-funnel | branded | ultra-niche",
      "question": "...",
      "rationale": "one sentence: which brand_context.md field this targets",
      "platforms": ["google_aio", "chatgpt"]
    }
  ]
}
```

The `platforms` field controls which surfaces run for each question:
- All questions: `["google_aio", "chatgpt"]`
- Branded queries (Q5, Q6): add `"google_ai_mode"`
- Geo-qualified ultra-niche query: add `"google_ai_mode"`
- Site: indexing check (Q7): `["chatgpt"]` only

The `rationale` field makes this the **locked prompt set** — re-runs use the same questions to track delta over time.

---

## Step 4: Query All Platforms

Run questions sequentially. For each question, fire Google search first (captures organic + AI Overview), then ChatGPT. Run Google AI Mode only on questions marked for it. All queries use Playwright MCP.

### 4a. Google Organic + AI Overview (Playwright — all questions except site: check)

Navigate to standard Google search with `gl=us&hl=en` (no `udm` parameter). Wait 4–5 seconds for the AI Overview to fully render, then save a snapshot.

```
Navigate: https://www.google.com/search?q=<URL_ENCODED_QUERY>&gl=us&hl=en&cr=countryUS
Wait: 4–5 seconds
Snapshot: seo-research/<domain>/geo-audit/raw/q<NN>_google_aio.md
```

Playwright captures the fully rendered DOM including AI Overview text, citation footnotes, shopping panels, and People Also Ask — everything visible on the page.

**Extracting content from the snapshot** (run after each save):
```python
import re

with open("q<NN>_google_aio.md") as f:
    content = f.read()

# All text nodes (includes AI Overview + organic snippets)
texts = re.findall(r'text: (.+)', content)

# Formal AIO citations = "Opens in new tab" links with /url
aio_citations = re.findall(
    r'link "([^"]+)\. Opens in new tab\." \[ref=\S+\] \[cursor=pointer\]:\s*\n\s*- /url: (https?://[^\s]+)',
    content
)

# Organic result links
organic = re.findall(
    r'link "([^"]+)" \[ref=\S+\] \[cursor=pointer\]:\s*\n\s*- /url: (https?://[^\s]+)',
    content
)
organic = [(t, u) for t, u in organic if 'google' not in u and len(t) > 8
           and not any(x in t for x in ['Skip', 'Sign in', 'Images', 'Videos', 'Opens in new tab'])]
```

**Brand/competitor detection**: text-search brand name and domain in the joined text nodes. Also extract bold entity names (they appear as `strong` or nested `generic` nodes in the snapshot tree).

### 4b. Google AI Mode (Playwright — branded queries + geo-qualified ultra-niche only)

```
Navigate: https://www.google.com/search?q=<URL_ENCODED_QUERY>&udm=50&gl=us&hl=en&cr=countryUS
Wait: 5 seconds
Snapshot: seo-research/<domain>/geo-audit/raw/q<NN>_google_aimode.md
```

**What AI Mode adds over AI Overviews:**
- **Local GBP layer**: for geo-qualified queries, AI Mode embeds Google Business Profile listings (star ratings, location, hours) that AI Overviews do not show. If the brand lacks a GBP, it is invisible in this layer.
- **Richer competitor set**: AI Mode sometimes names different specialist brands than AI Overviews, particularly for niche sub-categories.
- **Longer brand profile on branded queries**: more detail to check for hallucinations.

Use the same extraction approach as 4a. Citation footnotes use the same "Opens in new tab" pattern.

**Hallucination tracing on branded queries**: If AI Mode describes the brand with wrong facts, cross-reference with the Disambiguation Risk section of `brand_context.md`. Check which organic results appeared for the branded search (Step 1b) — a co-ranking article about a same-named company is usually the hallucination source.

### 4c. ChatGPT (Playwright — all questions)

ChatGPT is queried via Playwright MCP with a saved authenticated session, which enables live web search. This produces richer, structured data including entity annotations, search result groups, and citation mappings.

**Session setup**: The saved session at `.chatgpt-session.json` (current working directory) must be loaded into the Playwright browser context before starting. If the session is expired (login page appears instead of the chat interface), ask the user to re-login via Playwright and re-save with:
```js
await page.context().storageState({ path: '.chatgpt-session.json' });
```

**Per-question workflow** (run sequentially, not in parallel):

1. **Navigate** to `https://chatgpt.com` (fresh chat each question)

2. **Type the question** via `browser_type` into the chat input, with `submit: true`

3. **Wait 25 seconds** for the response + web search to complete

4. **Install route interceptor and reload** via `browser_run_code`. On reload, ChatGPT fetches the full conversation as a single JSON payload from `backend-api/conversation/<id>` (not SSE-streamed). Playwright's `page.route()` API intercepts this at the network level and survives page reload:

```js
async (page) => {
  await page.route('**/backend-api/conversation/**', async (route) => {
    const response = await route.fetch();
    const body = await response.text();
    await page.evaluate(({ url, status, bodyLength, body }) => {
      window._convData = window._convData || [];
      window._convData.push({ url, status, bodyLength, body });
    }, { url: route.request().url(), status: response.status(), bodyLength: body.length, body });
    await route.fulfill({ response, body });
  });

  await page.reload({ waitUntil: 'networkidle' });
  await page.waitForTimeout(5000);

  // Verify capture
  const result = await page.evaluate(() => ({
    captured: (window._convData || []).map(d => ({
      url: d.url, bodyLength: d.bodyLength, status: d.status
    }))
  }));
  return JSON.stringify(result, null, 2);
}
```

   The main payload is the request matching `/conversation/<uuid>` (not `/stream_status`, `/init`, or `/textdocs`). It's typically 20–40KB of structured JSON.

5. **Extract structured data** via `browser_run_code`:

```js
async (page) => {
  return await page.evaluate(() => {
    const entry = window._convData.find(d =>
      d.url.match(/conversation\/[a-f0-9-]{36}$/)
    );
    if (!entry) return JSON.stringify({ error: 'no conversation data' });
    const parsed = JSON.parse(entry.body);

    const result = { conversationId: parsed.conversation_id, title: parsed.title };

    for (const [id, node] of Object.entries(parsed.mapping)) {
      if (!node.message) continue;
      const msg = node.message;
      const role = msg.author?.role;

      // Tool message: search queries ChatGPT generated
      if (role === 'tool' && msg.metadata?.search_model_queries) {
        result.searchQueries = msg.metadata.search_model_queries.queries;
      }

      // Final assistant message: response text + all structured metadata
      if (role === 'assistant' && msg.content?.content_type === 'text'
          && msg.content?.parts?.length > 0) {
        result.responseText = msg.content.parts[0];
        result.searchResultGroups = (msg.metadata?.search_result_groups || []).map(g => ({
          entries: (g.entries || []).map(e => ({ title: e.title, url: e.url }))
        }));
        result.contentReferences = (msg.metadata?.content_references || []).map(ref => {
          if (ref.type === 'entity') {
            return { type: 'entity', name: ref.name, category: ref.category,
                     disambiguation: ref.extra_params?.disambiguation };
          }
          if (ref.type === 'grouped_webpages') {
            return { type: 'citation',
                     urls: ref.safe_urls || [],
                     items: (ref.items || []).map(i => ({
                       title: i.title, url: i.url, attribution: i.attribution
                     })) };
          }
          return { type: ref.type };
        });
      }
    }
    result.safeUrls = parsed.safe_urls || [];
    return JSON.stringify(result);
  });
}
```

   **What each field provides for the audit:**

   | Field | Contents | Use |
   |-------|----------|-----|
   | `searchQueries` | The search queries ChatGPT generated internally | Shows how ChatGPT interpreted the question |
   | `searchResultGroups` | Web pages ChatGPT retrieved (title + URL) | The "search result" tier — pages ChatGPT read |
   | `contentReferences` (type=entity) | Named entities with `category` and `disambiguation` | Every org/brand ChatGPT recognized in its response |
   | `contentReferences` (type=citation) | `safe_urls` + `items` with title/url/attribution | Formal inline citations — the highest-evidence tier |
   | `safe_urls` (top-level) | Deduplicated list of all cited URLs | Quick tally of every domain cited |
   | `responseText` | Full markdown response text | Brand mention detection, competitor extraction |

6. **Save output** to `seo-research/<domain>/geo-audit/raw/q<NN>_chatgpt.json`:

```json
{
  "question": "<the question>",
  "conversationId": "<from response>",
  "searchQueries": ["query ChatGPT generated"],
  "responseText": "<full markdown response>",
  "searchResultGroups": [
    { "entries": [{ "title": "Page Title", "url": "https://..." }] }
  ],
  "contentReferences": [
    { "type": "entity", "name": "Brand X", "category": "organization", "disambiguation": "..." },
    { "type": "citation", "urls": ["https://..."], "items": [{ "title": "...", "url": "...", "attribution": "..." }] }
  ],
  "safeUrls": ["https://example.com?utm_source=chatgpt.com", ...]
}
```

   If `safeUrls` is empty and `searchResultGroups` is empty, web search was not triggered — add `"note": "No web search triggered — ChatGPT answered from training data"`.

7. **Delete the conversation** after saving. Navigate to `https://chatgpt.com` for the next question.

**Key notes about the ChatGPT data**:
- Live web search produces `?utm_source=chatgpt.com` citation URLs — completely different competitor sets than training-data-only responses
- Structured `contentReferences` give entity annotations (org names with disambiguation) and citation-to-search-result mappings — no regex parsing needed
- `searchQueries` reveals how ChatGPT reformulated the question internally
- Some queries (especially broad questions) still won't trigger web search — ChatGPT answers from training data

**Key asymmetry to watch for**: Google AI (both surfaces) can retrieve live signals (BBB, Trustpilot, GBP) and may recognise a brand that ChatGPT doesn't. When this split appears, report it explicitly — it's a highly actionable finding.

---

## Step 5: Analyse Responses

### Brand mention detection

For each question × platform:

**Google AI Overview / AI Mode**: join all `text:` nodes from the snapshot and text-search for brand name + domain. If found, note:
- Surface: AI Overview / AI Mode / organic only
- Sentiment: positive / hedged / hallucinated
- Quote: 1–2 sentences containing the mention
- For AI Mode branded queries: check if the description matches `brand_context.md` ground-truth facts

**ChatGPT**: text-search brand name and domain in `responseText`. Also check `contentReferences` for entity annotations naming the brand and `safeUrls` for the brand's domain. If found, note position (early / middle / late), sentiment, and quote.

### Four finding types — treat differently

| Type | Definition | Priority |
|------|-----------|----------|
| **Absent, competitors present** | Brand missing, 1+ competitors named/cited | Highest — specific displacement finding |
| **Hedged recognition** | AI knows brand exists but qualifies with "appears to be", "you should verify" | High — signals no third-party coverage |
| **Hallucinated identity** | AI describes brand with wrong facts — wrong business type, wrong name alias, wrong products | High — actively misleads prospects; trace to disambiguation conflict |
| **Universal absence** | Brand and competitors both missing, generic answer | Medium — educational content gap |
| **Present, not leading** | Brand mentioned but not first recommendation | Lower — positioning gap |

### Competitor extraction

**Google AI Overview / AI Mode**: extract bold (`strong`) entity names from the snapshot tree — these are the brands Google's AI recommends by name. Also note the response structure type:
- **Tiered recommendation list** — most actionable displacement finding
- **Local GBP panel** (AI Mode only) — brand needs a GBP listing to appear here
- **Coverage explainer** — educational gap, no brands named

**ChatGPT**: extract competitor names from `contentReferences` where `type === "entity"` and `category === "organization"` — these are the brands ChatGPT explicitly recognized and annotated. Cross-reference with `responseText` for any additional brands mentioned without entity annotation.

### Citation tally — build master table across all surfaces

For each domain cited as a formal footnote (AIO "Opens in new tab" links, AI Mode cited sources, ChatGPT `safeUrls`), tally frequency across all questions and platforms. Build one master table:

| Domain | AIO cites | AI Mode cites | ChatGPT cites | ChatGPT search results | Total | Type | Implication |
|--------|-----------|---------------|---------------|----------------------|-------|------|-------------|

For ChatGPT, distinguish between two tiers:
- **`safeUrls` / `contentReferences` citations** — pages ChatGPT actually linked in its response (highest evidence)
- **`searchResultGroups` entries** — pages ChatGPT retrieved but may not have cited (shows what it considered)

Classify each domain:
- **Marketplace**: Amazon, Etsy, Faire → brand needs a listing
- **UGC/review**: Reddit, Quora, Trustpilot, G2 → content seeding / review generation
- **Association**: trade body, industry org → editorial placement opportunity
- **Specialist competitor**: direct competitors → content/positioning gap
- **Aggregator**: Insureon, NerdWallet, etc. → getting listed seeds multiple AI surfaces simultaneously
- **Editorial**: blogs, review sites, publications → outreach/feature placement

If the same domain is cited across 3+ queries on any single surface, it is the category's authority anchor — highest-leverage outreach target.

### Hallucination check

For any response that mentions the brand, cross-reference against ground-truth facts in `brand_context.md`. Flag:
- Wrong business type or category
- Wrong name aliases (e.g., "Brand Health" instead of "Brand Insurance")
- Wrong products, pricing, or location
- Invented claims not on the site

**Trace hallucinations to their source**: check which third-party pages rank for the branded query in organic results. A co-ranking article about a same-named company is typically the source. Report the specific URL causing the conflict — this guides the fix (schema markup, About page, disambiguation content).

Do not update `brand_context.md` to match hallucinated claims — the AI is wrong.

---

## Step 6: Write the Reports

Write TWO files:

### File 1: Client-facing report → `seo-research/<domain>/geo-audit-report.md`

**CRITICAL: This report is sent directly to end clients.** It must NEVER contain:
- Internal terminology: "regime", "new-regime", "established", question IDs (Q1, Q4, etc.)
- Tool or methodology names: ScrapingDog, Playwright, Oxylabs, curl, MCP, etc.
- References to raw data files, CSVs, JSON files, or internal paths
- Platform methodology notes ("via Playwright", "via curl", "via ScrapingDog")
- Question-by-question breakdowns — group by category instead

```markdown
# AI Visibility Audit: <domain>
**Date:** <date>

---

## Executive Summary

[3–5 sentences: current visibility level, most important finding, biggest competitor threat,
one immediate action. Frame in terms of business impact, not methodology.]

**AI Visibility Overview:**
| Platform           | Purchase-intent queries | Brand queries | Niche product queries |
|--------------------|------------------------|---------------|-----------------------|
| Google Search      | X/4                    | X/3           | X/3                   |
| Google AI Overview | X/4                    | X/3           | X/3                   |
| Google AI Mode     | X/4                    | X/2           | X/1                   |
| ChatGPT            | X/4                    | X/3           | X/3                   |

---

## Brand Context
[One paragraph — brand, positioning, what they sell — so the report is self-contained]

**Online presence:**

| Platform | Listed? | Notes |
|----------|---------|-------|
| Wikipedia | Y/N | |
| Crunchbase | Y/N | |
| G2 / Trustpilot | Y/N | rating if present |
| Amazon | Y/N | |
| Etsy / Faire | Y/N | |

[If disambiguation risk exists, describe it in plain language — "There is a similarly-named company X that may cause AI systems to confuse the two brands."]

---

## Purchase-Intent Queries

*We tested [4] queries that someone ready to buy would search — e.g. "[example query]", "[example query]".*

[2–4 paragraph summary of what we observed across all purchase-intent queries and across all platforms:
- Was the brand mentioned on any platform? If so, which and in what context?
- Which competitors appeared most consistently?
- What type of response did AI engines give (product listings, recommendation lists, educational content)?
- Key gap: what's missing that would get the brand included?]

---

## Brand Queries

*We tested [3] queries where someone searches for the brand directly — e.g. "[brand] review", "is [domain] legit".*

[2–4 paragraph summary:
- Does AI know the brand? How accurately does it describe it?
- What sources does AI cite when talking about the brand? (Trustpilot, own site, etc.)
- Any inaccuracies or hedged language?
- What would strengthen the brand's AI profile?]

---

## Niche Product Queries

*We tested [3] queries describing the brand's specific products without naming it — e.g. "[example query]".*

[2–4 paragraph summary:
- Was the brand surfaced when described by its attributes?
- Which competitors appeared instead?
- What content or signals would help AI connect these product descriptions to the brand?]

---

## Competitor Visibility

*How often each brand appeared across all queries and platforms. Higher = more AI authority in this space.*

| Brand | Google Search | Google AI Overview | Google AI Mode | ChatGPT | Total mentions |
|-------|-------------|-------------------|----------------|---------|---------------|
| **<audited brand>** | X | X | X | X | X |
| Competitor A | X | X | X | X | X |
| Competitor B | X | X | X | X | X |
| ... | | | | | |

[1–2 sentences on the dominant competitor and what makes them visible]

---

## Citation Sources

*Domains that AI engines cited as sources when answering queries in this space. These are the "trusted voices" that AI relies on — getting featured on these sites directly increases AI visibility.*

| Domain | Times cited | Type | Action |
|--------|-----------|------|--------|
| example.com | X | Marketplace / Review site / Blog | Get listed / Get featured / etc. |

[For top 5–10 cited domains: one sentence on the specific action it implies]

---

## Key Findings

[5–7 numbered findings, each 2–3 sentences. Focus on business impact and actionability.
Do NOT reference question numbers or internal categorization. Use natural language:
"When potential customers search for...", "AI engines consistently recommend...", etc.]

---

## Recommended Actions

### Immediate (this month)
[2–3 specific, concrete actions with expected impact]

### Short-term (next 1–3 months)
[3–4 actions]

### Medium-term (3–6 months)
[2–3 actions]
```

### File 2: Internal notes → `seo-research/<domain>/geo-audit/internal-notes.md`

This file captures methodology details, raw data references, and per-question breakdowns that are useful for follow-up audits but not for the client:

```markdown
# GEO Audit Internal Notes: <domain>
**Date:** <date>

## Methodology
- Platforms queried: [list with tool details]
- Session files used: [paths]
- Any issues encountered: [browser crashes, rate limits, expired sessions]

## Question-by-Question Detail
[Full per-question breakdown with platform details, finding types, exact quotes, etc.]

## Raw Data Files
[List of all raw files produced and what they contain]

## Re-run Notes
[Anything needed to reproduce or update this audit]
```

---

## Error Handling

- **AI Overview not visible after 4s wait**: wait an additional 3s and re-snapshot. If still absent, note it and proceed — some queries genuinely don't trigger AIOs.
- **AI Mode shows localised (non-US) results**: add `&gl=us&hl=en&cr=countryUS` to the URL. If still localised, the Playwright browser's IP geolocation is overriding URL params — note this in the report.
- **ChatGPT session expired**: if `chatgpt.com` shows a login page instead of the chat interface, ask the user to re-login via Playwright and re-save the session with `await page.context().storageState({ path: '.chatgpt-session.json' })`.
- **ChatGPT route interceptor captures no conversation data**: if `window._convData` is empty after reload, the conversation URL may not match the `**/backend-api/conversation/**` pattern. Check `browser_network_requests` to find the actual endpoint and adjust the route pattern.
- **ChatGPT rate limiting**: if ChatGPT shows a rate limit message, wait 60 seconds before the next question.
- **Homepage fetch fails**: try Playwright MCP; if still blocked, ask the user for a description before continuing.
- **Branded query returns mostly results for a different company**: record this as a disambiguation risk in `brand_context.md` and check AI Mode's branded response carefully for hallucination.

---

## Output Checklist

- [ ] `seo-research/<domain>/geo-audit/brand_context.md`
- [ ] `seo-research/<domain>/geo-audit/questions.json` (locked prompt set with rationale + platforms per question)
- [ ] `seo-research/<domain>/geo-audit/raw/q01_google_aio.md` … `q10_google_aio.md` (Playwright snapshots, skip q07)
- [ ] `seo-research/<domain>/geo-audit/raw/q05_google_aimode.md`, `q06_google_aimode.md` + geo-query (Playwright AI Mode snapshots)
- [ ] `seo-research/<domain>/geo-audit/raw/q01_chatgpt.json` … `q10_chatgpt.json` (structured ChatGPT data: response text, entities, citations, search results)
- [ ] `seo-research/<domain>/geo-audit/chatgpt_all_urls.json` (structured: all URLs with per-platform breakdown)
- [ ] `seo-research/<domain>/geo-audit/all_cited_urls.md` (client-facing: URL+count table and domain+count table)
- [ ] `seo-research/<domain>/geo-audit-report.md`
