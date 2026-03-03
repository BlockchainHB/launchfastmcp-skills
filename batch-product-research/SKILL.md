---
name: batch-product-research
description: Use this skill when the user wants to research multiple Amazon FBA keywords at once (batch research). Accepts a list of up to 20 keywords, runs parallel product research using sub-agents, and generates a professional HTML report + CSV. Automates the single-keyword product-research workflow across many keywords simultaneously.
---

# Batch Amazon FBA Product Research Skill

This skill processes a batch of Amazon FBA keywords (up to 20) through parallel sub-agent research rounds, then generates a professional HTML report and CSV export. It builds on the single-keyword product-research methodology (LegacyX FBA criteria) but optimizes for throughput and context efficiency.

## When to Use

- User provides multiple keywords to research (e.g., "research these 10 keywords for me")
- User wants a comparison report across several niches
- User says "batch research", "bulk research", or provides a list of keywords

## Input

A list of 1-20 keywords. Accept them from:
- A comma-separated list
- A numbered list
- A file path (read and parse)
- Direct conversation input

## Pre-Processing

Before dispatching keywords:

1. **De-duplicate** — Remove exact duplicates (case-insensitive). Inform the user which were removed.
2. **Count check** — If more than 20 keywords, split into batches of 20 and run each batch sequentially. Inform the user.
3. **Trim whitespace** — Clean up each keyword string.

## Sub-Agent Architecture

**Why sub-agents:** Each `research_products` response returns ~20 products with ~30 fields each. Running 20 keywords in the main agent context would consume the entire context window, leaving no room for report generation. Sub-agents isolate the large JSON payloads and return pre-built HTML cards + CSV rows, so the main agent only needs to assemble them.

### Dispatch Strategy

Split keywords evenly across up to 4 sub-agents, each handling ~5 keywords:

| Keywords | Sub-agents | Distribution |
|----------|-----------|--------------|
| 1-5      | 1         | All to one agent |
| 6-10     | 2         | ~5 each |
| 11-15    | 3         | ~5 each |
| 16-20    | 4         | ~5 each |

### How to Spawn Sub-Agents

Use the `Agent` tool with `subagent_type: "general-purpose"` for each batch. Spawn all sub-agents in a **single message** so they run in parallel.

**Sub-agent prompt (copy exactly, fill in keywords and path):**

```
You are a batch product research sub-agent. Your job:

1. Read the file at {absolute_path_to_this_skill}/.claude/skills/batch-product-research/SKILL.md
2. Find the section "## Sub-Agent Research Protocol" and follow it exactly
3. Your assigned keywords are: {keyword_1}, {keyword_2}, {keyword_3}, {keyword_4}, {keyword_5}
4. Return your results in the exact 3-section format specified in "## Sub-Agent Output Format"

IMPORTANT: Read the SKILL.md FIRST before doing anything else. It contains all instructions.
```

That's it. ~6 lines per sub-agent instead of ~200. The sub-agent reads the full protocol from this file.

---

## Sub-Agent Research Protocol

**This section is read by sub-agents.** If you are a sub-agent, follow these instructions exactly.

### Step 0: Load MCP Tools

Use `ToolSearch` to load the LaunchFast tools BEFORE making any research calls:
- Call `ToolSearch` with query `"+launchfast research"` to load `research_products`
- Call `ToolSearch` with query `"+launchfast keyword"` to load `amazon_keyword_research`

### Step 1: Round 1 — Initial Market Scan

For ALL of your assigned keywords **in parallel**, call:

```
research_products(keyword="<keyword>", focus="balanced", product_limit=20)
```

For each keyword, compute from the response:

1. **Search volume** — `market.search_volume`. If 0 or missing → NOT RECOMMENDED, note "no search volume".
2. **Average price** — `market.avg_price`. PASS if >= $25.
3. **Average reviews** — `market.avg_reviews`. PASS if <= 500. Exception: if niche revenue > $1M, reviews up to 750 can pass if low-review sellers (<200 reviews) show $30K+ revenue.
4. **Total niche revenue** — Sum all products' `monthly_revenue`. PASS if > $200,000.
5. **Average revenue per seller** — Total revenue / product count. PASS if >= $5,000.
6. **Top-seller dominance** — Sort by `monthly_revenue` desc. Top 3 sellers' share of total. PASS if < 50%.
7. **Estimated margin** — Use `market.avg_price`, estimate 15% Amazon fees, $4-6 manufacturing, $2.50 shipping per unit. PASS if >= 30%.

**Screening verdict:**
- **VIABLE** = 6-7 of 7 criteria pass
- **MARGINAL** = 4-5 criteria pass
- **NOT RECOMMENDED** = 3 or fewer pass

Also extract: `market.opportunity_score`, `market.market_grade`, `market.brand_concentration`, `market.dominant_brand`.

Record the top 3 products by `monthly_revenue` (ASIN, title first 50 chars, revenue, review count).

### Step 2: Round 2 — Financial Deep Dive (VIABLE and MARGINAL only)

For keywords that scored VIABLE or MARGINAL, call in parallel:

```
research_products(keyword="<keyword>", focus="financial", product_limit=20)
```

Compute:
- **% growing** — Products with `sales_trend_label: "growing"` / total products that have a trend label × 100
- **% stable** — Products with `sales_trend_label: "stable"` / total products that have a trend label × 100
- **% declining** — Products with `sales_trend_label: "declining"` / total products that have a trend label × 100
- **% unknown** — Products with null/missing `sales_trend_label` / total × 100
- **Average MoM growth** — Mean of all products' `month_over_month_growth_pct` (exclude nulls)
- **7d momentum** — General direction from `trend_7d_vs_prev7d_pct` values (positive/negative/mixed)

**Important:** Normalize growing + stable + declining to sum to 100% among products that HAVE trend data. Track unknown % separately.

### Step 3: Round 3 — Keyword Intelligence (VIABLE and MARGINAL only)

For keywords that scored VIABLE or MARGINAL, pick top 2-3 ASINs by revenue from Round 1 and call in parallel:

```
amazon_keyword_research(asins=["<ASIN1>", "<ASIN2>", "<ASIN3>"], limit=20)
```

Extract:
- **Keyword diversity** — Count of unique keywords returned
- **Average CPC** — Mean of `cpc` values
- **Average sponsored density** — Mean of `sponsored_density` values
- **Top 3 purchase-rate keywords** — Sort by `purchase_rate` desc, take top 3 with volume and rate

### Step 4: Error Handling Within Sub-Agent

- If an MCP tool call fails for a keyword → set verdict to ERROR, include error in notes, continue with other keywords
- If keyword returns zero products → set verdict to ERROR, note "no products found"
- If zero search volume → NOT RECOMMENDED with note "no search volume — keyword may be too specific"

---

## Sub-Agent Output Format

After completing all rounds, return your results in **exactly 3 delimited sections**. This format is critical — the main agent parses these sections programmatically.

### Section 1: DATA (compact metrics for exec summary + comparison table)

```
===DATA_START===
KEYWORD: {keyword}
VERDICT: {VIABLE|MARGINAL|NOT RECOMMENDED|ERROR}
OPPORTUNITY_SCORE: {X}
MARKET_GRADE: {X}
SEARCH_VOLUME: {X}
NICHE_REVENUE: {X}
AVG_PRICE: {X}
AVG_REVIEWS: {X}
AVG_REV_PER_SELLER: {X}
TOP_SELLER_DOMINANCE: {X}
EST_MARGIN: {X}
BRAND_CONCENTRATION: {low|medium|high}
DOMINANT_BRAND: {name}
CRITERIA_PASSED: {X}
CRITERIA_DETAIL: revenue:{P|F} price:{P|F} reviews:{P|F} rev_per_seller:{P|F} dominance:{P|F} search_vol:{P|F} margin:{P|F}
PCT_GROWING: {X}
PCT_STABLE: {X}
PCT_DECLINING: {X}
PCT_UNKNOWN_TREND: {X}
AVG_MOM_GROWTH: {X}
KEYWORD_DIVERSITY: {X}
AVG_CPC: {X}
AVG_SPONSORED_DENSITY: {X}
TOP_PURCHASE_KW_1: {keyword}|{volume}|{rate}
TOP_PURCHASE_KW_2: {keyword}|{volume}|{rate}
TOP_PURCHASE_KW_3: {keyword}|{volume}|{rate}
TOP_PRODUCT_1: {asin}|{title_50chars}|{revenue}|{reviews}
TOP_PRODUCT_2: {asin}|{title_50chars}|{revenue}|{reviews}
TOP_PRODUCT_3: {asin}|{title_50chars}|{revenue}|{reviews}
NOTES: {any exceptions, errors, or flags}
---
(repeat for each keyword)
===DATA_END===
```

**Rules for DATA section:**
- All numbers are RAW — no `$`, no `%`, no commas. Just digits and decimals (e.g., `1979928` not `$1,979,928`).
- Use `N/A` for fields not populated (keyword was NOT RECOMMENDED or ERROR, didn't reach Rounds 2/3).
- Use `0` for truly zero values (not `N/A`).
- Separate each keyword block with `---`.
- CRITERIA_DETAIL uses `P` for pass, `F` for fail, space-separated pairs.

### Section 2: HTML_CARDS (pre-built HTML for each keyword)

```
===HTML_CARDS_START===
(all HTML card divs here, one per keyword)
===HTML_CARDS_END===
```

Generate one HTML card per keyword using the CSS classes defined below. The main agent will paste these directly into the report.

**For VIABLE and MARGINAL keywords, generate a full detail card:**

```html
<div class="keyword-card">
  <div class="keyword-card-header">
    <h3>{keyword}</h3>
    <span class="badge {verdict_class}">{verdict}</span>
  </div>
  <div class="keyword-card-body">
    <h3>Market Scorecard</h3>
    <div class="scorecard">
      <div class="scorecard-row"><span class="label">Niche Revenue</span><span class="value {pass_or_fail}">${niche_revenue_formatted} (&gt;$200K)</span></div>
      <div class="scorecard-row"><span class="label">Avg Price</span><span class="value {pass_or_fail}">${avg_price} (&gt;=$25)</span></div>
      <div class="scorecard-row"><span class="label">Avg Reviews</span><span class="value {pass_or_fail}">{avg_reviews}{exception_note} (&lt;=500)</span></div>
      <div class="scorecard-row"><span class="label">Rev/Seller</span><span class="value {pass_or_fail}">${avg_rev_per_seller_formatted} (&gt;=$5K)</span></div>
      <div class="scorecard-row"><span class="label">Top Seller Dominance</span><span class="value {pass_or_fail}">{dominance}% (&lt;50%)</span></div>
      <div class="scorecard-row"><span class="label">Search Volume</span><span class="value {pass_or_fail}">{search_volume_formatted}</span></div>
      <div class="scorecard-row"><span class="label">Est. Margin</span><span class="value {pass_or_fail}">{margin}% (&gt;=30%)</span></div>
    </div>
    <p><strong>Brand Concentration:</strong> {brand_concentration} &middot; <strong>Dominant Brand:</strong> {dominant_brand}</p>
    <p><strong>Market Grade:</strong> {grade} &middot; <strong>Opportunity Score:</strong> {score}/100</p>
    <h3>Sales Trend Distribution</h3>
    <div class="trend-bar-container">
      <div class="trend-bar">
        <div class="growing" style="width: {pct_growing}%"></div>
        <div class="stable" style="width: {pct_stable}%"></div>
        <div class="declining" style="width: {pct_declining}%"></div>
      </div>
      <div class="trend-bar-labels">
        <span>Growing: {pct_growing}%</span>
        <span>Stable: {pct_stable}%</span>
        <span>Declining: {pct_declining}%</span>
      </div>
    </div>
    <p>Avg MoM Growth: {mom}% &middot; 7d Momentum: {momentum}</p>
    <h3>Top Products</h3>
    <table class="mini-table">
      <thead><tr><th>ASIN</th><th>Title</th><th>Revenue</th><th>Reviews</th></tr></thead>
      <tbody>
        <tr><td>{asin1}</td><td>{title1}</td><td>${rev1}</td><td>{reviews1}</td></tr>
        <tr><td>{asin2}</td><td>{title2}</td><td>${rev2}</td><td>{reviews2}</td></tr>
        <tr><td>{asin3}</td><td>{title3}</td><td>${rev3}</td><td>{reviews3}</td></tr>
      </tbody>
    </table>
    <h3>Keyword Intelligence</h3>
    <p><strong>Keyword Diversity:</strong> {diversity} terms &middot; <strong>Avg CPC:</strong> ${cpc} &middot; <strong>Avg Sponsored Density:</strong> {density}</p>
    <table class="mini-table">
      <thead><tr><th>Keyword</th><th>Volume</th><th>Purchase Rate</th></tr></thead>
      <tbody>
        <tr><td>{kw1}</td><td>{vol1}</td><td>{rate1}%</td></tr>
        <tr><td>{kw2}</td><td>{vol2}</td><td>{rate2}%</td></tr>
        <tr><td>{kw3}</td><td>{vol3}</td><td>{rate3}%</td></tr>
      </tbody>
    </table>
    <p><em>{notes}</em></p>
  </div>
</div>
```

**For NOT RECOMMENDED and ERROR keywords, generate a collapsed card:**

```html
<div class="keyword-card collapsed">
  <div class="keyword-card-header">
    <h3>{keyword}</h3>
    <div>
      <span class="badge not-recommended">{verdict}</span>
      <span class="collapsed-note">{criteria_passed}/7 criteria &middot; {short_reason}</span>
    </div>
  </div>
  <div class="keyword-card-body">
    <div class="scorecard">
      <div class="scorecard-row"><span class="label">Niche Revenue</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Avg Price</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Avg Reviews</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Rev/Seller</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Top Seller Dominance</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Search Volume</span><span class="value {p_or_f}">{value_or_dash}</span></div>
      <div class="scorecard-row"><span class="label">Est. Margin</span><span class="value {p_or_f}">{value_or_dash}</span></div>
    </div>
  </div>
</div>
```

**CSS class reference for HTML cards:**
- Verdict badge: `badge viable`, `badge marginal`, `badge not-recommended`, `badge error`
- Scorecard values: `value pass` (green) or `value fail` (red)
- Use `&middot;` for centered dots, `&gt;` for >, `&lt;` for <
- Format numbers with commas (e.g., `$1,979,928`)
- Use `&mdash;` for missing data
- **Trend bars**: growing + stable + declining widths should sum to 100% among products with trend data. If some products had unknown trends, note this in the text below the bar (e.g., "45% of products lacked trend data").
- **Sort cards**: VIABLE first, then MARGINAL (both by opportunity score desc within tier), then NOT RECOMMENDED, then ERROR.

### Section 3: CSV_ROWS (pre-built CSV rows)

```
===CSV_ROWS_START===
{keyword},{verdict},{opportunity_score},{market_grade},{search_volume},{niche_revenue},{avg_price},{avg_reviews},{avg_rev_per_seller},{top_seller_dominance_pct},{est_margin_pct},{brand_concentration},{dominant_brand},{criteria_passed},7,{pct_growing},{pct_stable},{pct_declining},{avg_mom_growth_pct},{keyword_diversity},{avg_cpc},{avg_sponsored_density},{asin1},{asin1_rev},{asin1_reviews},{asin2},{asin2_rev},{asin2_reviews},{asin3},{asin3_rev},{asin3_reviews}
(one row per keyword)
===CSV_ROWS_END===
```

**Rules for CSV rows:**
- No dollar signs, no percent signs, no commas in numbers — raw values only.
- Empty string for N/A fields (just consecutive commas: `,,`).
- Quote any field containing commas with double quotes.
- Ensure every row has exactly 31 comma-separated values (matching the 31-column header).

---

## Main Agent Assembly Workflow

After all sub-agents return, the main agent does:

### 1. Parse DATA sections

Extract all `===DATA_START===...===DATA_END===` blocks from sub-agent responses. Parse each keyword's metrics.

### 2. Compute aggregates

- Total keywords researched
- Count by verdict: viable, marginal, not recommended, error
- Top 3 opportunities: among VIABLE keywords, sort by opportunity score desc. If fewer than 3 viable, include top MARGINAL keywords to fill.

### 3. Sort keywords

VIABLE first (by opportunity score desc), then MARGINAL (by opportunity score desc), then NOT RECOMMENDED, then ERROR.

### 4. Build HTML report

Assemble the full HTML file by combining:

**a) HTML shell (header + CSS + exec summary + comparison table + footer)** — The main agent generates this from the template below, filling in aggregate stats and the comparison table from parsed DATA.

**b) HTML cards** — Extract all `===HTML_CARDS_START===...===HTML_CARDS_END===` blocks from sub-agent responses. Re-order the cards to match the sort order (VIABLE → MARGINAL → NOT RECOMMENDED → ERROR). Paste them between the comparison table and methodology footer.

### 5. Build CSV

- Write the header row (31 columns, see schema below)
- Extract all `===CSV_ROWS_START===...===CSV_ROWS_END===` blocks
- Sort rows to match keyword sort order
- Concatenate

### 6. Write files

```
{cwd}/reports/batch-research-{YYYY-MM-DD-HHmm}.html
{cwd}/reports/batch-research-{YYYY-MM-DD-HHmm}.csv
```

Create `reports/` directory if needed. Use current date/time for timestamp.

### 7. Report to user

Print file paths and a brief summary: how many viable/marginal/not recommended, top 3 opportunities.

---

## CSV Schema

Header row (31 columns):

```
keyword,verdict,opportunity_score,market_grade,search_volume,niche_revenue,avg_price,avg_reviews,avg_rev_per_seller,top_seller_dominance_pct,est_margin_pct,brand_concentration,dominant_brand,criteria_passed,criteria_total,pct_products_growing,pct_products_stable,pct_products_declining,avg_mom_growth_pct,keyword_diversity_count,avg_cpc,avg_sponsored_density,top_asin_1,top_asin_1_revenue,top_asin_1_reviews,top_asin_2,top_asin_2_revenue,top_asin_2_reviews,top_asin_3,top_asin_3_revenue,top_asin_3_reviews
```

---

## HTML Shell Template

The main agent generates the outer HTML structure. Sub-agent HTML cards get inserted where marked `{KEYWORD_DETAIL_CARDS_HERE}`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Batch Product Research Report — {date}</title>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html { font-size: 15px; -webkit-font-smoothing: antialiased; }
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  color: #1a1a1a; background: #ffffff; line-height: 1.6; padding: 2rem; max-width: 1200px; margin: 0 auto;
}
h1 { font-size: 1.75rem; font-weight: 700; margin-bottom: 0.25rem; color: #111; }
h2 { font-size: 1.3rem; font-weight: 600; margin: 2.5rem 0 1rem; color: #111; border-bottom: 2px solid #e5e5e5; padding-bottom: 0.5rem; }
h3 { font-size: 1.1rem; font-weight: 600; margin: 1.5rem 0 0.75rem; color: #333; }
p { margin-bottom: 0.75rem; }
.subtitle { color: #666; font-size: 0.9rem; margin-bottom: 2rem; }
.methodology-tag {
  display: inline-block; background: #f0f0f0; color: #555; font-size: 0.75rem;
  padding: 0.2rem 0.6rem; border-radius: 3px; font-weight: 500; margin-left: 0.75rem; vertical-align: middle;
}
.stat-cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 1rem; margin: 1.5rem 0; }
.stat-card { background: #fafafa; border: 1px solid #e5e5e5; border-radius: 8px; padding: 1.25rem; text-align: center; }
.stat-card .number { font-size: 2rem; font-weight: 700; line-height: 1.2; }
.stat-card .label { font-size: 0.8rem; color: #666; text-transform: uppercase; letter-spacing: 0.05em; margin-top: 0.25rem; }
.stat-card.viable .number { color: #16a34a; }
.stat-card.marginal .number { color: #d97706; }
.stat-card.fail .number { color: #dc2626; }
.stat-card.total .number { color: #111; }
.top-opportunities { background: #f8faf8; border: 1px solid #d1e7d1; border-radius: 8px; padding: 1.25rem 1.5rem; margin: 1.5rem 0; }
.top-opportunities h3 { margin-top: 0; color: #16a34a; font-size: 1rem; }
.top-opportunities ol { margin: 0.75rem 0 0 1.25rem; }
.top-opportunities li { margin-bottom: 0.5rem; }
.top-opportunities .score { font-weight: 600; color: #16a34a; }
.no-viable { background: #fef8f8; border: 1px solid #e7d1d1; border-radius: 8px; padding: 1.25rem 1.5rem; margin: 1.5rem 0; color: #666; }
.table-wrapper { overflow-x: auto; margin: 1.5rem 0; border-radius: 8px; border: 1px solid #e5e5e5; }
table { width: 100%; border-collapse: collapse; font-size: 0.85rem; }
thead { background: #fafafa; }
th {
  text-align: left; padding: 0.75rem 1rem; font-weight: 600; color: #555;
  border-bottom: 2px solid #e5e5e5; white-space: nowrap; font-size: 0.78rem;
  text-transform: uppercase; letter-spacing: 0.03em;
}
td { padding: 0.65rem 1rem; border-bottom: 1px solid #f0f0f0; }
tbody tr:nth-child(even) { background: #fafafa; }
tbody tr:hover { background: #f5f5f5; }
td.pass { color: #16a34a; font-weight: 600; }
td.fail { color: #dc2626; font-weight: 600; }
td.marginal-cell { color: #d97706; font-weight: 600; }
.badge {
  display: inline-block; padding: 0.2rem 0.65rem; border-radius: 4px;
  font-size: 0.75rem; font-weight: 600; text-transform: uppercase; letter-spacing: 0.03em;
}
.badge.viable { background: #dcfce7; color: #16a34a; }
.badge.marginal { background: #fef3c7; color: #92400e; }
.badge.not-recommended { background: #fee2e2; color: #dc2626; }
.badge.error { background: #f3f4f6; color: #6b7280; }
.keyword-card { border: 1px solid #e5e5e5; border-radius: 8px; margin: 1.5rem 0; overflow: hidden; }
.keyword-card-header {
  padding: 1rem 1.5rem; background: #fafafa; border-bottom: 1px solid #e5e5e5;
  display: flex; align-items: center; justify-content: space-between;
}
.keyword-card-header h3 { margin: 0; font-size: 1.05rem; }
.keyword-card-body { padding: 1.5rem; }
.keyword-card.collapsed .keyword-card-body { display: none; }
.keyword-card.collapsed .keyword-card-header { border-bottom: none; background: #fefefe; }
.collapsed-note { font-size: 0.85rem; color: #999; font-style: italic; }
.scorecard { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 0.5rem 1.5rem; margin: 1rem 0; }
.scorecard-row { display: flex; justify-content: space-between; padding: 0.4rem 0; border-bottom: 1px solid #f5f5f5; }
.scorecard-row .label { color: #666; font-size: 0.85rem; }
.scorecard-row .value { font-weight: 600; font-size: 0.85rem; }
.scorecard-row .value.pass { color: #16a34a; }
.scorecard-row .value.fail { color: #dc2626; }
.trend-bar-container { margin: 1rem 0; }
.trend-bar { display: flex; height: 24px; border-radius: 4px; overflow: hidden; background: #f0f0f0; }
.trend-bar .growing { background: #86efac; }
.trend-bar .stable { background: #fde68a; }
.trend-bar .declining { background: #fca5a5; }
.trend-bar-labels { display: flex; justify-content: space-between; font-size: 0.75rem; color: #666; margin-top: 0.35rem; }
.mini-table { width: 100%; font-size: 0.82rem; margin: 0.75rem 0; }
.mini-table th {
  text-transform: none; font-size: 0.78rem; letter-spacing: 0;
  border-bottom: 1px solid #e5e5e5; padding: 0.4rem 0.75rem;
}
.mini-table td { padding: 0.4rem 0.75rem; border-bottom: 1px solid #f5f5f5; }
.methodology { margin-top: 3rem; padding-top: 2rem; border-top: 2px solid #e5e5e5; font-size: 0.82rem; color: #888; }
.methodology h2 { font-size: 1rem; color: #888; border-bottom: none; margin-bottom: 0.75rem; }
.methodology table { font-size: 0.8rem; }
.methodology th, .methodology td { padding: 0.4rem 0.75rem; }
.disclaimer {
  margin-top: 1.5rem; padding: 1rem; background: #fafafa; border-radius: 6px;
  font-size: 0.78rem; color: #999; line-height: 1.5;
}
@media print {
  body { padding: 0; font-size: 12px; }
  .keyword-card { break-inside: avoid; }
  .keyword-card.collapsed .keyword-card-body { display: block; }
  .stat-cards { grid-template-columns: repeat(4, 1fr); }
  tbody tr:hover { background: inherit; }
  .table-wrapper { overflow: visible; }
}
</style>
</head>
<body>

<h1>Batch Product Research Report <span class="methodology-tag">LegacyX FBA Method</span></h1>
<p class="subtitle">{date} &middot; {keyword_count} keywords analyzed</p>

<h2>Executive Summary</h2>
<div class="stat-cards">
  <div class="stat-card total"><div class="number">{total}</div><div class="label">Total Researched</div></div>
  <div class="stat-card viable"><div class="number">{viable}</div><div class="label">Viable</div></div>
  <div class="stat-card marginal"><div class="number">{marginal}</div><div class="label">Marginal</div></div>
  <div class="stat-card fail"><div class="number">{fail}</div><div class="label">Not Recommended</div></div>
</div>

<!-- If viable > 0, show top opportunities box. If viable == 0, show the no-viable box instead. -->
<div class="top-opportunities">
  <h3>Top Opportunities</h3>
  <ol>
    <li><strong>{keyword}</strong> &mdash; Score: <span class="score">{score}/100</span> &middot; ${revenue} niche revenue &middot; {passed}/7 criteria</li>
    <!-- up to 3 items -->
  </ol>
</div>
<!-- OR if no viable keywords: -->
<!-- <div class="no-viable">No viable keywords identified in this batch. Consider broader keyword variations or different product categories.</div> -->

<h2>Keyword Comparison</h2>
<div class="table-wrapper">
<table>
<thead>
<tr><th>Keyword</th><th>Verdict</th><th>Score</th><th>Revenue</th><th>Avg Price</th><th>Avg Reviews</th><th>Rev/Seller</th><th>Dominance</th><th>Margin</th><th>Criteria</th><th>Trend</th></tr>
</thead>
<tbody>
<!-- One row per keyword from parsed DATA. Use pass/fail classes on cells:
  Revenue: pass if > 200000, else fail
  Avg Price: pass if >= 25, else fail
  Avg Reviews: pass if <= 500 (or exception), else fail
  Rev/Seller: pass if >= 5000, else fail
  Dominance: pass if < 50, else fail
  Margin: pass if >= 30, else fail
  Trend: show "{pct_growing}% ↑" for viable/marginal, "—" for not recommended
-->
<tr>
  <td><strong>{keyword}</strong></td>
  <td><span class="badge {class}">{verdict}</span></td>
  <td>{score}</td>
  <td class="{p_or_f}">${revenue_formatted}</td>
  <td class="{p_or_f}">${price}</td>
  <td class="{p_or_f}">{reviews}</td>
  <td class="{p_or_f}">${rev_per_seller_formatted}</td>
  <td class="{p_or_f}">{dominance}%</td>
  <td class="{p_or_f}">{margin}%</td>
  <td>{passed}/7</td>
  <td>{trend_or_dash}</td>
</tr>
</tbody>
</table>
</div>
<p style="font-size:0.78rem;color:#999;margin-top:0.5rem;">* Reviews passed via large-market exception</p>

<h2>Keyword Details</h2>

{KEYWORD_DETAIL_CARDS_HERE}

<div class="methodology">
  <h2>Methodology</h2>
  <p>Each keyword was evaluated against 7 criteria from the LegacyX FBA product research framework:</p>
  <table>
    <thead><tr><th>Criteria</th><th>Threshold</th><th>What It Measures</th></tr></thead>
    <tbody>
      <tr><td>Niche Revenue</td><td>&gt; $200,000/mo</td><td>Total market size from top 20 products</td></tr>
      <tr><td>Average Price</td><td>&gt;= $25</td><td>Margin viability &mdash; below $25 is too tight</td></tr>
      <tr><td>Average Reviews</td><td>&lt;= 500</td><td>Barrier to entry &mdash; high reviews = hard to compete</td></tr>
      <tr><td>Revenue per Seller</td><td>&gt;= $5,000/mo</td><td>Per-seller opportunity &mdash; enough revenue to go around</td></tr>
      <tr><td>Top Seller Dominance</td><td>&lt; 50%</td><td>Market concentration &mdash; dominated markets are risky</td></tr>
      <tr><td>Search Volume</td><td>Must exist</td><td>Demand validation &mdash; no searches = no customers</td></tr>
      <tr><td>Estimated Margin</td><td>&gt;= 30%</td><td>Pre-advertising profitability estimate</td></tr>
    </tbody>
  </table>
  <p style="margin-top:1rem;"><strong>Verdicts:</strong> VIABLE = 6-7 criteria passed &middot; MARGINAL = 4-5 passed &middot; NOT RECOMMENDED = 0-3 passed</p>
  <p>Financial trends (Round 2) and keyword intelligence (Round 3) were collected for VIABLE and MARGINAL keywords only.</p>
  <div class="disclaimer">
    <strong>Disclaimer:</strong> This report is generated from LaunchFast MCP market data and automated analysis.
    All figures are estimates based on available data at the time of research. Actual market conditions,
    margins, and performance may vary. This report does not constitute financial or business advice.
    Always validate with supplier quotes and additional due diligence before making sourcing decisions.
  </div>
</div>

</body>
</html>
```

---

## Error Handling

| Scenario | How to Handle |
|----------|---------------|
| Sub-agent fails entirely | Mark its keywords as VERDICT: ERROR in report with note "sub-agent failure". Generate empty HTML cards and CSV rows for those keywords. |
| Single keyword MCP call fails | Sub-agent marks that keyword as VERDICT: ERROR with error details in NOTES. Continue with remaining keywords. |
| Zero search volume | NOT RECOMMENDED with note "no search volume — keyword may be too specific". |
| All keywords fail screening | Generate report normally with all NOT RECOMMENDED. Show "no viable" message instead of Top Opportunities. |
| Duplicate keywords in input | De-duplicate before dispatching. Inform user which duplicates were removed. |
| More than 20 keywords | Split into batches of 20. Process each batch as a separate report. |
| Keyword returns zero products | ERROR with note "no products found for this keyword". |

---

## Verification Checklist

After generating the report, verify:

- [ ] HTML file opens in browser with no broken styles
- [ ] All keyword data appears in the comparison table
- [ ] VIABLE keywords have green badges, MARGINAL amber, NOT RECOMMENDED red
- [ ] Collapsed cards exist for NOT RECOMMENDED / ERROR keywords
- [ ] CSV is well-formed: all rows have exactly 31 columns
- [ ] Trend bars render with correct proportions (growing + stable + declining = 100%)
- [ ] Print view (`Cmd+P` in browser) looks clean
- [ ] Top Opportunities section shows correct top-3 (or "no viable" message)
