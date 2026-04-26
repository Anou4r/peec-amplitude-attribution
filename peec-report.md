---
description: Generate this week's Peec × Amplitude AI-visibility attribution report
argument-hint: [week-offset — 0 = current week, -1 = last week, -2 = two weeks ago, default 0]
---

# /peec-report

Generate a weekly AI-visibility attribution report. Joins Peec's brand/URL/domain metrics with Amplitude conversion events on the URL paths Peec shows as cited by AI engines, then grades the full PLG funnel per origin. Writes to `reports/YYYY-Www.md`.

## Project context (cached)

Your workspace IDs go here so they aren't re-discovered on every run. Only call the `list_*` tools if a cached ID is missing. **Replace the placeholders below with your own values before first run.**

- Peec project_id: `<your-peec-project-id>`
- Peec own brand_id: `<your-own-brand-id>`
- Amplitude Production projectId: `<your-amplitude-project-id>` (timezone Europe/Berlin)

## Step 1 — Compute date window

- Today is derived from the system clock.
- Current ISO week runs Monday – Sunday. Compute the target week from `$ARGUMENTS` (default 0 = current week, -1 = previous week, etc.).
- Always also compute the immediately prior week for W-o-W deltas.
- Format both as `start_date` / `end_date` (YYYY-MM-DD).
- Label the report file `reports/<YYYY>-W<ww>.md` using the ISO week number of `end_date`.

## Step 2 — Pull Peec metrics (parallel)

Run these in parallel — they're independent. If any call fails mid-flight (transport drops, 422s), retry once with back-off before falling back.

1. `list_brands(project_id)` — only if own brand name mapping for competitors isn't already cached. Build a `brand_id → name` lookup.
2. `list_tags(project_id)`, `list_topics(project_id)`, `list_models(project_id)` — only if needed to label dimensions in the output; otherwise skip.
3. `get_brand_report(project_id, start_date, end_date)` for the **target week** — no dimensions, all brands aggregated.
4. `get_brand_report(project_id, prev_start, prev_end)` for the **previous week** — same shape, used for W-o-W deltas.
5. `get_url_report(project_id, start_date, end_date, filters=[mentioned_brand_id in [own_brand_id]], limit=25)` — top URLs that cite the own brand.
6. `get_domain_report(project_id, start_date, end_date, limit=15)` — gives the source-authority picture across own/competitor/editorial/UGC domains.
7. `get_actions(project_id, scope="overview", start_date, end_date)` — opportunity map.

### Step 2b — Drill into top actions

From the overview result, pick the top 3 slices by `opportunity_score` and drill down:

- `action_group_type="OWNED"` with a `url_classification` → call `get_actions(scope="owned", url_classification=...)`.
- `action_group_type="EDITORIAL"` with a `url_classification` → `scope="editorial", url_classification=...`.
- `action_group_type="UGC"` with a `domain` → `scope="ugc", domain=...`.
- `action_group_type="REFERENCE"` with a `domain` → `scope="reference", domain=...`.

**Known issue (2026-04-21):** `scope="owned"` sometimes returns HTTP 422 on the Peec backend. Retry twice with 1s/3s back-off; if all three attempts fail, skip that slice and note it in the Methodology section of the report ("Owned-scope recommendations unavailable for this run: Peec backend 422").

## Step 3 — Pull Amplitude data (parallel with Step 2 where possible)

### Step 3a — Cross-domain stitch pre-flight (run first, blocking)

Before the main batch, run one query to verify Amplitude's cross-domain cookie is actually stitching users across your marketing-site and app subdomains (e.g. `marketing.example.com` → `auth.example.com` → `app.example.com`). If it isn't, every downstream "AI-attributed" signup is misleading and the report must lead with a tracking-fix action, not a UTM-tagging action.

```
type: "eventsSegmentation", app: "<your-amplitude-project-id>",
name: "Pre-flight: signup by initial_referring_domain",
params: {
  range: "Last 30 Days",
  events: [{ event_type: "<your-signup-event>", filters: [],
    group_by: [{ type: "user", value: "gp:initial_referring_domain" }] }],
  metric: "totals", countGroup: "User", interval: 1,
  segments: [{ conditions: [] }]
}
```

**Pass/fail rule** — compute `internal_share = (count where initial_referring_domain ∈ {<your-product-domains>, EMPTY, (none)}) / total`. Substitute your own product domains (the ones that would be "internal" — e.g. `app.example.com`, `www.example.com`, `auth.example.com`). The null buckets (`EMPTY`, `(none)`) are lumped in because a broken stitch typically produces null referrers.

- `internal_share ≥ 0.95` → **FAIL.** Recommended Action #1 becomes "Fix cross-domain attribution"; all Funnel-by-Origin tables get a ⚠ callout.
- `0.80 ≤ internal_share < 0.95` → **PARTIAL.** Marketing-site acquisition tracking gap is real. Becomes Recommended Action #2 (above UTM tagging, below any Peec `get_actions` slice with a higher `opportunity_score`).
- `internal_share < 0.80` → **PASS.** Proceed normally; cite measured share in Methodology.

Store `internal_share` and the top external referrers list (with counts); both get cited in Methodology and the PLG Funnel by Origin section.

---

Expected event set (adjust to match your workspace's schema — these are the events the author confirmed present when writing this command):

**Events — the full PLG funnel (pageview → CTA click → signup → activation → paid):**
- `page_viewed` **[step 1]** — marketing-site pageview. Expected properties: `url` (required, full URL), `path`, `referrer`. Most workspaces fire this on cookie-consent.
- `button_clicked` **[step 2 — CTA-click bridge]** — custom event with `url` + `target_url` + `title` properties. Links a specific blog URL to signup intent via `url contains "<your-blog-path>"` filter. Preferred over `[Amplitude] Element Clicked` (autotrack, noisy).
- `signup` **[step 3]** — your verified-signup event. Replace `<your-signup-event>` in the queries below with the actual event name in your workspace (e.g. `account_verified`, `user_signed_up`, `SignupCompleted`). Use the bot-filtered variant if you have one.
- `activation` **[step 4 — PLG aha-moment]** — the event representing "user got value from the product". Product-specific: first project created, first channel connected, first workflow run, first AI agent deployed, etc. Replace `<your-activation-event>` in the queries. This is the real PLG success signal — email-verified isn't business value, activation is.
- `paid_conversion` **[step 5 — revenue]** — your paid-conversion event (e.g. `plan_upgraded`, `subscription_started`, `checkout_completed`). Replace `<your-paid-event>` in the queries. Note: many paid-conversion events fire on *any* subscription change (upgrades, downgrades, plan switches), not only first-payment — event counts are cohort activity, not strict first-conversions.

**Autotrack note:** `[Amplitude] Page Viewed` is Amplitude's autotracked pageview event. Don't use it alongside a custom `page_viewed` — pick one, consistently.

**User properties carrying attribution (set once at first touch, persist across events):**
- `gp:initial_referring_domain` — domain of first-touch referrer (best for grouping signup and pageview data by origin)
- `gp:initial_referrer` — full first-touch URL
- `gp:initial_utm_source`, `gp:initial_utm_campaign`, `gp:initial_utm_content`, etc. — UTM at first touch (critical for AI attribution since most AI engines strip the referrer header)

**Important context:** `page_viewed` only fires for cookie-consented users, so absolute pageview counts are the consented fraction of total traffic (typically 30–60% in DE depending on consent-banner config). What matters is the **ratio of Peec citations to Amplitude pageviews** per URL — that ratio surfaces whether AI is recommending a URL more than users are clicking through. 2–5× is healthy for authority content; >10× indicates the URL is being cited in in-situ AI answers that don't create click-through.

### Queries to run

Use `mcp__Amplitude__query_dataset`. Canonical chart type: `eventsSegmentation`. Group_by objects use `{type, value}`, NOT `{group_type, group_key}`. Interval is a numeric enum (1 = daily, 7 = weekly, etc.).

Run these in parallel for both the target week and the previous week:

1. **Blog pageviews per URL** (marketing-side traffic):
   ```
   type: "eventsSegmentation", app: "<your-amplitude-project-id>",
   name: "Blog pageviews by URL <week>",
   params: {
     start: "<ISO start>", end: "<ISO end>",
     events: [{
       event_type: "page_viewed",
       filters: [{ subprop_type: "event", subprop_key: "url",
         subprop_op: "contains", subprop_value: ["<your-blog-path>"] }],
       group_by: [{ type: "event", value: "url" }]
     }],
     metric: "totals", countGroup: "User", interval: 1,
     segments: [{ conditions: [] }]
   }
   ```
   Use groupByLimit 50, timeSeriesLimit 0.

2. **Blog pageviews by initial_referring_domain** — origin mix of blog traffic:
   Same event+filter as above; change `group_by` to `[{ type: "user", value: "gp:initial_referring_domain" }]`.

3. **AI-referred blog pageviews by URL × AI domain** — the directly-attributable AI rows:
   Add a second filter to Query 1's filters array:
   ```
   { subprop_type: "user", subprop_key: "gp:initial_referring_domain",
     subprop_op: "is", subprop_value: [
       "chatgpt.com", "chat.openai.com", "openai.com",
       "www.perplexity.ai", "perplexity.ai", "claude.ai",
       "gemini.google.com", "copilot.microsoft.com", "grok.com"
     ] }
   ```
   Group by `[{ type: "event", value: "url" }, { type: "user", value: "gp:initial_referring_domain" }]`.

4. **CTA clicks on blog pages by initial_referring_domain** — the bridge from blog URL to signup intent:
   ```
   events: [{ event_type: "button_clicked",
     filters: [{ subprop_type: "event", subprop_key: "url",
       subprop_op: "contains", subprop_value: ["<your-blog-path>"] }],
     group_by: [{ type: "user", value: "gp:initial_referring_domain" }] }]
   ```
   Captures funnel step 2. If the event fires at low volume, note it in the Methodology section but don't fail.

5. **Verified signups by initial_referring_domain** (funnel step 3, primary signup metric):
   ```
   events: [{ event_type: "<your-signup-event>", filters: [],
     group_by: [{ type: "user", value: "gp:initial_referring_domain" }] }]
   ```

6. **Activations by initial_referring_domain** (funnel step 4 — PLG aha-moment):
   Same shape as #5 with `event_type: "<your-activation-event>"`. The real PLG success signal — email-verified isn't business value, activation is.

7. **Paid conversions by initial_referring_domain** (funnel step 5 — revenue):
   Same shape as #5 with `event_type: "<your-paid-event>"`. Remember: event count ≠ distinct paying users (paid events often fire on upgrades/downgrades too). Treat as cohort revenue activity.

8. **Same queries for the previous week** for W-o-W deltas.

If a chart definition fails with a schema error, call `mcp__Amplitude__verify_chart_definition` — it auto-corrects common mistakes.

### How to interpret

- **Cite-to-view ratio** remains the headline visibility metric. For each Peec-top-cited URL, compute `peec_citations / amplitude_pageviews`. 2–5× is healthy; >10× indicates in-situ citation with no click-through.
- **PLG Funnel by Origin** is the headline *business* metric — the new money quote. For each origin (`gp:initial_referring_domain`), show the full funnel: pageview → CTA click → signup → activation → paid_conversion. AI engines (chatgpt.com, www.perplexity.ai, copilot.microsoft.com, claude.ai, gemini.google.com) should be listed explicitly even at small volumes — small AI cohorts that make it all the way to paid conversion are the PLG-validation insight for Peec. **Do not compute "conversion rate" as paid_conversion / signup**: paid events often fire on upgrades/downgrades from users who signed up in earlier periods, so the ratio is meaningless. Report absolute counts per stage per origin.
- **AI-direct pageviews** = sum of pageviews where `initial_referring_domain` is in the AI-domain list. Strict lower bound (most AI engines send `no-referrer`).
- **Recommended Action #1 is conditional on the pre-flight result:**
  - Pre-flight **FAIL** → "Fix cross-domain attribution" (stitch broken, nothing downstream is trustworthy).
  - Pre-flight **PARTIAL** → "Close the marketing-site → app acquisition tracking gap" (most verified signups have no measurable origin; this dwarfs UTM tagging in impact).
  - Pre-flight **PASS** → the existing UTM-tagging priority list is the headline action.
- **Consent-gating caveat:** Amplitude pageviews are consented-users-only, so absolute numbers are the consented fraction (typically 30–60% in DE). Ratios matter, not absolutes.
- **Cohort vs. funnel caveat (two effects):** Event counts grouped by `gp:initial_referring_domain` are cohort activity in the window, not a strict new-user funnel. (1) **Time:** a user who signed up a year ago still generates paid_conversion events this week, attributed to their first-touch origin. (2) **AI → Google attribution bleed:** users doing AI research in ChatGPT / Perplexity often google the brand name afterwards to verify before clicking — that click attributes to `www.google.com`, not the AI engine. The `chatgpt.com` / `perplexity.ai` rows are therefore a **strict lower bound** on AI's real contribution. Right framing: "does AI visibility drive ongoing revenue?"; wrong framing: "how many of this week's AI-referred signups became paying customers?" Both get called out in Methodology.

## Step 4 — Join & rank

For each URL in Peec's top-20 cited list, attach the Amplitude pageview and signup counts. Derive `est. AI-attributed signups` per URL using the heuristic:

```
est. AI-attributed signups = signups_on_url × citation_rate_on_url
```

The `citation_rate_on_url` is already in the `get_url_report` response. This is a conservative lower bound (most rigorous: only count signups where the URL is *both* highly cited and has unusually high AI-referrer share vs. its baseline).

**Winners:** top 5 prompts by SoV delta W-o-W (positive).
**Losers:** top 3 prompts by SoV delta W-o-W (negative).

### Root-cause for losers

For each loser, call `list_chats(project_id, start_date, end_date, prompt_id=<prompt>, limit=3)`, then for one representative `chat_id` call `get_chat(project_id, chat_id)`. Extract: (a) which competitor brands appeared higher, (b) the URL(s) that competitor was cited from. Include a 1–2 sentence root-cause snippet per loser in the report.

## Step 5 — Write the report

File path: `reports/<YYYY>-W<ww>.md`.

If the file already exists, **ask the user whether to overwrite** — do not overwrite automatically.

Use this section order, with real markdown tables (no raw JSON):

1. **Title + subtitle** with the date range and the generation timestamp.
2. **Executive Summary** — 3–4 sentences. Sentence 1: visibility headline (SoV, W-o-W delta). Sentence 2: the PLG-funnel insight (e.g. *"Chatgpt.com-referred users reached paid_conversion this week — AI visibility is validated as a revenue channel"*). Sentence 3: the pre-flight result framed as an action (*"~X% of verified signups have no measurable origin — acquisition tracking gap is the #1 fix"*). Optional sentence 4: one sharp competitor delta.
3. **Brand Standings** — table of all 16 brands with Visibility %, SoV %, Mention count, Sentiment, Avg Position. Own brand bolded. Include a 1-sentence "Reading" interpretation.
4. **Source Domain Footprint** — table of top 12-15 domains with classification, retrieved %, citation rate. Bold the own domain. Include a 1-sentence Reading.
5. **Top-Cited Own URLs** — table of top 15-20 URLs that cite the own brand, with classification, citation_count, retrievals, citation_rate. Call out any URL with citation_rate > 1.3 as "authority" content.
6. **Attribution Table** — merge the top 8-10 own URLs with their Amplitude pageviews and signup counts, plus the computed `est. AI-attributed signups`.
7. **PLG Funnel by Origin** — the money section for PLG validation. Table where rows are the top 8-10 origins (`gp:initial_referring_domain`) and columns are the five funnel stages: Blog pageview, CTA click, Signup, Activation, Paid conversion. Always include a dedicated "AI engines (sum)" row AND the individual AI engines as their own rows even at small volumes — the point is to show that AI-referred users reach paid, not to sort by volume. If the pre-flight returned PARTIAL or FAIL, prepend the table with a ⚠ callout ("~X% of verified signups have no measurable origin — the cohort rows below describe the measurable fraction only"). Include a one-paragraph Reading that names the best AI-referred paid-conversion example from the week (e.g. *"chatgpt.com cohort: 4 blog pageviews → 1 signup → 1 paid conversion — AI-referred users reach revenue"*).
8. **Recommended Actions** — the top action is conditional on the pre-flight:
   - Pre-flight **FAIL** → Action #1 is "Fix cross-domain attribution". Include the measured `internal_share`, the specific internal domains over-represented, and a pointer to the Amplitude SDK cookieDomain setting. Then continue with Peec `get_actions` drill-downs.
   - Pre-flight **PARTIAL** → Action #1 is "Close the marketing-site → app acquisition tracking gap". Reference the measured `internal_share` and the specific null buckets (`EMPTY`, `(none)`). UTM tagging becomes Action #3 or lower.
   - Pre-flight **PASS** → keep the existing flow: top action is the UTM-tagging priority list.
   Remaining actions: narrative form, grouped by `action_group_type` (OWNED / EDITORIAL / UGC / REFERENCE). Use the textual recommendations from `get_actions` drill-downs verbatim but formatted as bullet lists with links preserved.
9. **Winners & Losers** — two sub-sections. Each item: prompt text (from `list_prompts`), SoV delta, and (for losers) the one-sentence root-cause snippet from `get_chat`.
10. **Attribution Methodology** — a short section explaining:
    - Cross-domain stitch pre-flight result: measured `internal_share`, pass/partial/fail, interpretation
    - Primary attribution = URL-path matching between Peec-cited URLs and Amplitude `page_viewed` / signup events
    - Secondary attribution = AI-referrer regex as a lower bound on direct AI-referred traffic
    - Cohort-vs-funnel caveat: events-by-initial_referring_domain are cohort-window totals, not strict new-user funnels
    - Consent-gating caveat on absolute pageview/signup numbers
    - Any per-run issues (e.g. "Peec `get_actions scope=owned` returned 422 — Owned actions omitted" or "activation event returned no rows this week")
    - Explicit note that this measures *own-content performance in AI answers*, not SEO organic traffic
11. **Footer** — short generation credit with `#BuiltWithPeec`.

## Presentation rules

- Never show raw IDs (`kw_...`, `pr_...`, `to_...`) in the output. Always resolve to human-readable names via the cached `list_*` results.
- Never display raw JSON blocks.
- Metrics: visibility, share_of_voice, retrieved_percentage are 0–1 ratios — display as percentages (×100). sentiment is 0–100 (display as-is). position is a rank (display as-is; lower is better). citation_rate and retrieval_rate are averages that can exceed 1.0 — display as-is, never ×100.
- State the queried `date_range` in the subtitle and again in the Methodology section.
- Use the ISO week number of the week's end date to name the file.

## Idempotency

- If `reports/<YYYY>-W<ww>.md` exists, ask the user before overwriting.
- If the Amplitude event discovery cache is stale (> 30 days old) or missing, re-run `get_events` once.

## Starting action

Silently execute Steps 1–5 and present the report. Do not narrate intermediate tool calls, ID resolution, or data wrangling to the user — per Peec's own MCP guidance, present the analysis directly. The final output is the written report file path plus a one-paragraph summary in the chat.
