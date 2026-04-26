---
description: Content-ROI feedback for a published URL — Peec citations × Amplitude conversions
argument-hint: <url-or-slug> [--days=7]
---

# /peec-loop

Audit how a specific published URL is performing in AI search (Peec) and on the product funnel (Amplitude), compared to the hypothesis in its brief. Writes `briefs/<slug>.followup.md`.

## Project context (cached)

- Peec project_id: `<your-peec-project-id>`
- Peec own brand_id: `<your-own-brand-id>`
- Amplitude Production projectId: `<your-amplitude-project-id>` (Europe/Berlin)

## Input resolution

`$ARGUMENTS` may be:
- A full URL (e.g. `https://example.com/blog/my-post`)
- A slug (`whatsapp-business-api-kosten`) — look up `briefs/<slug>.md` to resolve the URL
- Empty — prompt the user to pass a URL or slug

Default analysis window is the last 7 days unless `--days=N` is passed.

## Step 1 — Peec data for this URL

Run in parallel:

1. `get_url_report(project_id, start_date, end_date, filters=[{field:"url", operator:"in", values:[URL]}], dimensions=["prompt_id","model_id"], limit=100)` — every (prompt × model) combination that pulled this URL, with citation_rate per row.
2. `get_url_report(...same filters, dimensions=["chat_id"], limit=5)` — sample chat_ids that cited this URL.
3. For 1–2 sample chat_ids from step 2, `get_chat(project_id, chat_id)` — pull the full chat, including the `brands_mentioned` list (with positions) and the `sources` list (with per-source `citationCount`).
4. `get_actions(scope="editorial", url_classification=<URL's classification>)` — editorial recommendations for the URL's content type. The URL's classification comes from step 1's `classification` column (COMPARISON, LISTICLE, ARTICLE, etc.).
5. Optionally `get_url_content(url=<top competitor URL from step 3 sources>)` — scrape the competing page's markdown for structural comparison.

### URL's classification → editorial scope mapping

- COMPARISON → `scope="editorial", url_classification="COMPARISON"`
- LISTICLE → `scope="editorial", url_classification="LISTICLE"`
- HOW_TO_GUIDE → `scope="editorial", url_classification="HOW_TO_GUIDE"`
- ARTICLE → `scope="editorial", url_classification="ARTICLE"`
- Other or null → skip step 4.

### Known issue

Peec `get_actions(scope="owned", url_classification=...)` can return HTTP 422. If that happens, retry 3x, then skip — document in the Methodology section of the followup.

## Step 2 — Amplitude per-URL business check (full PLG funnel)

Amplitude's `page_viewed` event fires on your marketing-site pages for cookie-consented users, with event property `url` (required, full URL) — giving us real per-URL attribution. For the downstream PLG funnel stages we switch to cohort attribution (`gp:initial_referring_domain`) because Amplitude can't strictly join "users who viewed this URL" to "users who later upgraded" without user-session stitching we don't have.

Before running the queries below, **check for a recent `/peec-report` run** (current or previous week). If found, inherit its `internal_share` pre-flight result and note it in the followup. If no recent report, run the pre-flight from `/peec-report` Step 3a inline. The pre-flight result determines whether Edit Suggestions lead with a tracking-fix action or content actions.

Use `query_dataset` with `eventsSegmentation`. Run in parallel:

1. **Pageviews on this URL** (funnel step 1, per-URL exact match):
   ```
   event_type: "page_viewed",
   filters: [{ subprop_type: "event", subprop_key: "url",
     subprop_op: "is", subprop_value: [<full URL>] }],
   group_by: [{ type: "user", value: "gp:initial_referring_domain" }]
   ```
   Per-domain origin mix for visits to this specific URL. Compute cite/view ratio = `peec_citations / total_pageviews`.

2. **Same query for the previous period** (same-length window immediately before) for deltas.

3. **AI-referred pageviews on this URL**: repeat query 1 with an extra filter on `gp:initial_referring_domain ∈ {chatgpt.com, chat.openai.com, openai.com, www.perplexity.ai, perplexity.ai, claude.ai, gemini.google.com, copilot.microsoft.com, grok.com}`. Any result rows are direct AI-referred visits to this URL. Zero rows on a high-citation URL is the UTM-tagging signal (see Edit Suggestions).

4. **CTA clicks on this URL** (funnel step 2, per-URL exact match):
   ```
   event_type: "button_clicked",
   filters: [{ subprop_type: "event", subprop_key: "url",
     subprop_op: "is", subprop_value: [<full URL>] }],
   group_by: [{ type: "user", value: "gp:initial_referring_domain" }]
   ```
   Captures whether the URL's readers actually click through to signup. Low or zero counts on a high-pageview URL indicate the content isn't converting readers to signup intent — which is more actionable than just "this URL is AI-cited but low click-through."

5. **Per-URL signup attribution via UTM** (funnel step 3, exact per-URL): if signup events have `gp:initial_utm_content` matching this URL's slug, query for those — true per-URL signup attribution. Most blog URLs aren't UTM-tagged, so this will usually return zero; that zero is itself a recommendation.

6. **Cohort funnel downstream (signup → activation → paid)** — for the top origins appearing in Query 1, pull their whole-cohort events to show the full PLG funnel for this URL's audience mix:
   - `<your-signup-event>` grouped by `gp:initial_referring_domain` (funnel step 3)
   - `<your-activation-event>` grouped by `gp:initial_referring_domain` (funnel step 4 — PLG aha-moment)
   - `<your-paid-event>` grouped by `gp:initial_referring_domain` (funnel step 5 — revenue)

   Same shape as peec-report queries 5/6/7. Copy the batch from `/peec-report` verbatim. Note clearly in the followup that these are **cohort event totals in the window, not strict per-URL attribution** — the point is directional: does this URL's origin mix (from Query 1) overlap with origins that produce paid conversions?

**Consent caveat:** Absolute pageview numbers are the cookie-consented fraction of total traffic. The cite/view ratio is the robust signal, not the absolute count.

**Cohort caveat:** Queries 6–7 group by initial_referring_domain, which means a user who signed up 6 months ago via chatgpt.com is still in the "chatgpt.com cohort" today. Their plan_upgrade in this week attributes back to that origin. This is the right framing for "does AI visibility drive ongoing revenue?" but wrong for "did this week's blog pageview turn into this week's revenue." Both framings belong in the followup.

## Step 3 — Load the brief (if exists)

If `briefs/<slug>.md` exists, parse the YAML frontmatter and the Success Criteria and Editorial Hooks sections. These become the "Hypothesis" column in the Hypothesis-vs-Reality table.

If no brief exists, the followup labels all hypothesis cells as `—` and recommends creating a brief retrospectively.

## Step 4 — Write the followup

Path: `briefs/<slug>.followup.md`. Overwrite if exists (unlike the weekly report which asks — followups are expected to be re-run).

Sections, in order:

1. **Frontmatter** — slug, url, parent_brief, generated date, data_window.
2. **Headline** — 2–3 sentence take on whether the content is performing its brief, including one sentence on the URL's cohort funnel (e.g. "Viewers came from chatgpt.com, google.com, and bing.com; those same origins produced N plan_upgrade events this week").
3. **Hypothesis vs Reality table** — one row per Success Criteria metric.
4. **"What's pulling the content"** — table of prompts ranked by citations, with avg citation_rate.
5. **Sample citation** — one quoted excerpt from a `get_chat` response showing the own-brand mention + the full source list with citation counts. Call out any competitor that gets double-cited in the same chat.
6. **Competitor delta** — identify the top 1–2 competing URLs from the sample chat's sources, compute their citation weight, and suggest what to inspect (optionally using `get_url_content` on one competitor URL for concrete structural diff).
7. **Amplitude business check** — two parts:
   - *Per-URL* (Queries 1, 3, 4, 5): pageviews, CTA clicks, UTM-attributed signups on this URL. Clear language about what attribution is available.
   - *Cohort PLG funnel* (Query 6): origin-by-origin funnel table — rows = top origins appearing in Query 1, columns = pageview, CTA click, verified signup, channel created, plan upgrade. Include AI engines as their own rows even at small volumes. Inherit the pre-flight ⚠ callout from `/peec-report` if PARTIAL or FAIL. Paragraph of Reading: name any origin that produced a paid upgrade this week.
8. **Edit suggestions** — ranked list. If pre-flight is FAIL/PARTIAL, the top suggestion is a tracking-fix rather than a content edit. Otherwise combine structural hypotheses from step 6 with Peec's `get_actions` editorial output.
9. **Next review** — today + 14 days, with the exact re-run command.
10. **Footer** — generation credit with `#BuiltWithPeec`.

## Presentation rules

- Resolve all IDs to human-readable names (prompt text from `list_prompts`, brand names from `list_brands`, etc.). Cache list results from the latest `/peec-report` run if available.
- Never dump raw JSON.
- Metrics: visibility, share_of_voice, retrieved_percentage are 0–1 ratios (×100 for display). sentiment is 0–100 (as-is). position is a rank (as-is). citation_rate and retrieval_rate can exceed 1.0 (as-is, never ×100).
- Explicitly call out the Amplitude marketing-site data gap in every followup until it's resolved — it's the single biggest factor in interpreting the numbers.
- Quote actual excerpts from `get_chat` responses when surfacing citation context. Seeing the AI's words is worth more than any metric.

## Starting action

Silently execute Steps 1–4 and present the followup. Do not narrate intermediate tool calls to the user. Final output is the followup file path plus a one-paragraph summary in the chat.
