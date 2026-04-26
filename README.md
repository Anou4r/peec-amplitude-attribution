# Peec × Amplitude — AI-Visibility to Revenue Attribution

*A Claude Code workflow that answers one question your Peec dashboard alone can't: **does AI visibility actually drive signups and revenue?***

Submission for the [Peec MCP Challenge](https://peec.ai/mcp-challenge). `#BuiltWithPeec`

---

## The problem this solves

If you're running Peec, you can see your brand's citation count, share-of-voice, position, and which URLs AI engines pull when someone asks about your category. Great — you know you're *visible*.

But Peec can't tell you what happened *after* the citation. Did the reader click through? Did they sign up? Did they upgrade to a paid plan? That answer lives in your product analytics (Amplitude, in this case), which doesn't know anything about AI engines. The two tools sit in different tabs, and most marketing leads close the loop with a Monday-morning spreadsheet, if at all.

This workflow closes the loop. One command pulls Peec's visibility metrics, joins them with Amplitude's full PLG funnel — pageview → CTA click → signup → activation → paid plan — and groups every stage by `initial_referring_domain` so you can see, per origin (including ChatGPT, Perplexity, Claude, Copilot, Gemini), how many people actually made it all the way through. It produces a committed Markdown report you can drop into Slack, review weekly, and point at when someone asks "is AI visibility actually moving revenue, or just vanity?"

The business impact is simple: if even one person from an AI-engine cohort reaches `plan_upgraded`, AI is a revenue channel — not a vanity metric — and you can start investing in AI-visibility content with the same confidence you'd invest in SEO. If no AI cohort reaches revenue, you know it's vanity and you can redirect. Either way, you stop guessing.

---

## What's in the repo

Two Claude Code **slash commands**, defined as prompt files in `.claude/commands/`:

| Command | Question it answers | Writes |
| :--- | :--- | :--- |
| **`/peec-report`** | *Where are we visible in AI answers this week, and did that visibility move our PLG funnel — all the way to paid?* | `reports/YYYY-Www.md` |
| **`/peec-loop <url>`** | *Is this specific blog post converting the readers AI is sending to it?* | `briefs/<slug>.followup.md` |

Both commands are pure prompt files — no code, no scripts, no build step. Claude Code is the runtime. The Peec MCP and Amplitude MCP provide the data.

---

## Setting it up in Claude Code

### 1. Prerequisites

- **Claude Code** — install from [claude.com/claude-code](https://claude.com/claude-code) if you don't have it.
- A **[Peec](https://peec.ai)** workspace with at least one project, your own brand marked `is_own=true`, and a few tracked prompts that have been running long enough to have data.
- An **[Amplitude](https://amplitude.com)** project with these events (or near-equivalents) instrumented:
  - `page_viewed` on your marketing site
  - `button_clicked` on your CTAs
  - `signup`
  - `activation/aha-moment`
  - `paid_conversion`
  - The user property `gp:initial_referring_domain` — Amplitude sets this automatically under most configs.

### 2. Clone the repo

### 3. Authorize the MCPs

When you launch Claude Code in this directory, it reads `.mcp.json` and offers to connect to both MCP servers:

```bash
claude
```

Then, inside Claude Code:

```
/mcp
```

This triggers OAuth in your browser for both Peec (`api.peec.ai/mcp`) and Amplitude (`mcp.eu.amplitude.com/mcp`). One-time setup. No API keys, no `.env` file.

### 4. Discover the slash commands

Claude Code auto-discovers slash commands from `.claude/commands/*.md`. Once you're in this directory with Claude running, `/peec-report` and `/peec-loop` will show up in the command autocomplete. No install step — the presence of the files *is* the install.

### 5. Substitute your IDs

Open `.claude/commands/peec-report.md` and `.claude/commands/peec-loop.md`. Near the top of each you'll see a "Project context (cached)" block with placeholders:

```
- Peec project_id: <your-peec-project-id>
- Peec own brand_id: <your-own-brand-id>
- Amplitude Production projectId: <your-amplitude-project-id>
```

Find your IDs by asking Claude directly:

```
List my Peec projects.
List the brands in project <name> and show which one is marked is_own.
Show my Amplitude projects via get_context.
```

Paste the three IDs into both command files or let claude handle it (as well as everything else).

### 6. Adjust the Amplitude filters to your workspace

In the Amplitude query blocks inside `peec-report.md`, you'll see a placeholder `<your-blog-path>` (e.g. change it to `example.com/blog` if your marketing blog lives there). This is the URL fragment that identifies marketing-site pageviews and CTA clicks vs. in-app events.

Also sanity-check the `event_type` strings (`page_viewed`, `button_clicked`, `account_verified`, `message_channel_created`, `plan_upgraded`) against your Amplitude schema. If your events are named differently, update the strings in the command file — the queries won't error, they'll just return empty rows.

### 7. Run it

```
/peec-report            # current week
/peec-report -1         # previous week
/peec-loop <url-or-slug>
```

First run for a given week writes to `reports/YYYY-Www.md`. Re-running the same week asks before overwriting.

---

## What you'll see in the output

### `/peec-report` — the weekly PLG attribution

The key section is **PLG Funnel by Origin**: a table where each row is an origin (`www.google.com`, `chatgpt.com`, `www.perplexity.ai`, `copilot.microsoft.com`, direct, social, etc.) and each column is a funnel stage — `page_viewed` → `button_clicked` → `signup_form_submitted` → `account_verified` → `message_channel_created` → `plan_upgraded`. AI engines are always listed as their own rows, even at small volumes. That's the whole point: small AI cohorts that still make it to revenue are the PLG-validation signal.

Around that table, the report carries brand and domain leaderboards from Peec, a top-cited-own-URLs table with **cite/view ratios** (AI citations / measurable pageviews — a 2–5× ratio is healthy; above 10× means AI is answering in-situ without click-through), Peec-sourced Recommended Actions, and a Methodology section that explains the caveats.

### `/peec-loop <url>` — per-URL deep dive

For a single published URL: Peec retrieval/citation counts, which prompts pulled the URL, a sample AI response with the full source list, competitor double-citation callouts, the per-URL Amplitude funnel (pageviews + CTA clicks on *this exact URL*), and ranked edit suggestions.

---

## Design notes

### Cross-domain stitch pre-flight (the self-check)

Before the main Amplitude batch, `/peec-report` runs a sanity check: what fraction of verified signups have a null or internal `initial_referring_domain`? If it's ≥ 95 %, the cross-domain cookie isn't stitching and the downstream funnel is measuring shadows — Recommended Action #1 becomes "Fix your tracking" and every funnel row gets a ⚠ callout. If it's 80–95 %, the report surfaces the tracking gap as a priority action. Below 80 %, it proceeds normally. When the tracking underneath it is broken, the workflow says so instead of quietly producing numbers that look fine but mean nothing.

### Cohort-vs-funnel framing

Events grouped by `initial_referring_domain` are *cohort* activity in the window — a user who signed up months ago via Google still contributes to Google's `plan_upgraded` this week. That's the right framing for "does AI visibility drive ongoing revenue for a cohort we acquired via that origin?" It's the *wrong* framing for "did this week's blog pageview turn into this week's revenue?" Both are legitimate questions; this workflow answers the first, and the Methodology section says so every run.

### Consent gating

`page_viewed` typically fires only for cookie-consented users, so absolute pageview numbers are the consented fraction of total traffic (often 30–60 % in DE/EU). The **cite/view ratio** is the robust signal, not the absolutes.

---

## Tools used

- **Peec MCP** (`https://api.peec.ai/mcp`) — 13 tools: `list_projects`, `list_brands`, `list_tags`, `list_topics`, `list_prompts`, `list_models`, `list_chats`, `get_chat`, `get_url_content`, `get_brand_report`, `get_url_report`, `get_domain_report`, `get_actions` *(scopes: `overview`, `editorial`, `ugc`, `reference`)*.
- **Amplitude MCP** (`https://mcp.eu.amplitude.com/mcp`) — `get_context`, `get_project_context`, `get_properties`, `query_dataset` *(chart type `eventsSegmentation`)*.

---

## Known limitations

- `get_actions(scope="owned")` intermittently returns HTTP 422 from the Peec backend. The command retries 3× with back-off and, if all retries fail, documents the skip in the report's Methodology section rather than silently drop the slice.
- Queries assume the event names / user-property keys documented above. If your workspace uses different names, the commands won't error — they'll return empty rows. Update the `event_type` strings in the command files to match.

---

*Built by Anouar Springer with Claude Code, Peec MCP, and Amplitude MCP.*
