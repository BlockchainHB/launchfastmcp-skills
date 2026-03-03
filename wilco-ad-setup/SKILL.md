---
name: wilco-ad-setup
description: Use this skill when the user wants to set up Amazon PPC campaigns using Wilco's analytics-driven exact-match strategy. Takes a main keyword and product SKU, performs market research to find competitors, runs batched keyword research via sub-agents, builds a conversion-focused keyword list ranked by purchase rate and relevance, then generates Amazon Bulksheets 2.0 CSV files for 5 Sponsored Products campaigns (Auto, Product Targeting, 3x Manual Exact). Also produces an HTML summary artifact. Lower risk, lower velocity, analytics-first approach. Powered by the LegacyX FBA methodology and LaunchFast MCP tools.
---

# Wilco's Ad Setup Skill

This skill implements Wilco's analytics-driven PPC strategy from the LegacyX FBA course. It creates a 5-campaign Sponsored Products launch package focused on exact match targeting, strict budget isolation, and data-driven keyword selection. Ready for upload via Amazon Bulksheets 2.0.

## Strategy Philosophy

Wilco's method is the **industry-standard approach used by large brands**. It prioritizes control and data over reach:

- **Exact match only** — ads show only for the literal keyword, its plurals, capitalizations, and misspellings. No stray search terms.
- **1 ad group per campaign** — Amazon cannot distribute budget across ad groups reliably. Isolate everything.
- **Max 5 keywords per keyword campaign** — Amazon starves bottom keywords even with unlimited budget. Keep it tight.
- **Max 10 product targets per campaign** — slightly more tolerant than keywords, but still limited.
- **Analytics-driven selection** — pick keywords based on purchase rate, conversion data, and relevance scores, not intuition.
- **Minimal negation needed** — exact match prevents wasted spend on irrelevant terms.
- **Lower velocity, lower risk** — explicitly acknowledged trade-off. This method produces fewer sales initially but wastes less ad spend.

## User Inputs Required

| Input | Required | Example | Notes |
|---|---|---|---|
| **Main keyword** | Yes | `bamboo cutting board` | Primary search term for the niche |
| **Product SKU** | Yes | `BB-CUT-001` | Seller SKU for the product ad |
| **Budget level** | No | `conservative` or `aggressive` | Defaults to conservative |
| **Product name** | No | `Bamboo Cutting Board` | For campaign naming. Derived from keyword if not given |

## Workflow

### Phase 1: Market Research & Competitor Collection

Run `research_products` to identify top competitors:

```
research_products(keyword="<main_keyword>", focus="balanced", product_limit=20)
```

**Collect from results:**
1. Top 10-12 ASINs sorted by `monthly_revenue` descending
2. Market data: `avg_price`, `avg_reviews`, `search_volume`, `opportunity_score`, `market_grade`, `avg_cpc`
3. Note brands, concentration level, and dominant brand

**Important for Wilco's method:** Focus on ASINs that are well-established (higher reviews, stable/growing sales trends). These are the competitors whose keyword portfolios are proven to convert. Unlike Kunze's gap-hunting approach, Wilco targets **proven converting keywords** that top sellers already rank for.

### Phase 2: Batched Keyword Research (SUB-AGENT DELEGATION)

**CRITICAL: Delegate to sub-agents to protect the main context window.**

Split competitor ASINs into batches of 3-4 and run keyword research in parallel.

**Batching rules:**
- Maximum 3-4 ASINs per batch
- Use `limit=50` per batch
- Use `min_search_volume=300` to filter noise
- Run batches in parallel

**Sub-agent prompt template:**

```
You are a keyword research assistant for an analytics-driven PPC strategy. Run keyword research and return conversion-focused results.

Run this tool call:
amazon_keyword_research(asins=["ASIN1", "ASIN2", "ASIN3"], limit=50, min_search_volume=300)

From the results, extract the top 20 keywords sorted by PURCHASE RATE descending (not search volume).

Return results as a STRUCTURED LIST with one keyword per line in this exact format:
KEYWORD | search_volume | purchase_rate | cpc | relevance_score | ASIN1_rank | ASIN2_rank | ASIN3_rank

Example:
bamboo cutting board | 125235 | 5.33 | 1.28 | 892 | 12 | 45 | -
cutting boards for kitchen | 321719 | 3.61 | 2.39 | 756 | 8 | - | 23

Use "-" if an ASIN is not ranking. Use pipe delimiters (|) for easy parsing.

Filter OUT:
- Keywords with search_volume below 300
- Brand names of competitors
- Single generic words (e.g., "bottle", "board", "wood")
- Keywords where fewer than 2 of the input ASINs rank in the top 100

The last filter is key: for Wilco's analytics approach, we want keywords where MULTIPLE competitors already rank — this is data evidence that the keyword converts for this product type.

Also report on separate lines:
TOTAL_FOUND: [number]
PASSED_FILTERS: [number]
TOP_PURCHASE_RATE: [keyword1], [keyword2], [keyword3], [keyword4], [keyword5]
```

**Launch as parallel sub-agents:**
```
Agent(subagent_type="general-purpose", name="kw-batch-1", prompt="<template with ASINs 1-4>")
Agent(subagent_type="general-purpose", name="kw-batch-2", prompt="<template with ASINs 5-8>")
Agent(subagent_type="general-purpose", name="kw-batch-3", prompt="<template with ASINs 9-12>")
```

### Phase 3: Build Analytics-Driven Keyword List

After all sub-agent batches return, merge and process:

**3a. Merge & Deduplicate**
- Combine all keyword arrays from all batches
- Deduplicate by keyword text (case-insensitive)
- For duplicates: merge competitor rank data, keep highest purchase_rate value

**3b. Competitor Coverage Scoring**
- For each keyword, count how many of the 10-12 competitor ASINs rank for it
- **Coverage score** = number of competitors ranking / total competitors analyzed
- Keywords with coverage >= 50% are "proven converters" — strong evidence these keywords drive sales in this niche

**3c. Wilco Score (Analytics-Driven Ranking)**
Calculate a composite score for each keyword using **min-max normalization** (scale each metric to 0-1 within the keyword set):

```
Normalization: normalized_value = (value - min) / (max - min)  [0 if max == min]

wilco_score = (purchase_rate_normalized * 30) + (coverage_score * 25) + (search_volume_normalized * 20) + (relevance_score_normalized * 25)
```

Where:
- `purchase_rate_normalized * 30` — highest weight on conversion potential (Wilco's primary metric). Normalize the raw purchase_rate values across all keywords using min-max.
- `coverage_score * 25` — proven converter signal. Already 0-1 scale (competitors ranking / total competitors).
- `search_volume_normalized * 20` — volume matters but is secondary. Normalize raw search_volume using min-max.
- `relevance_score_normalized * 25` — relevance prevents wasted spend. Normalize raw relevance_score using min-max.

Maximum possible score: 100 (all metrics at their max within the set).

**3d. Categorize & Select**

Sort all keywords by `wilco_score` descending and assign to 3 keyword campaigns:

**Campaign A — Top Converters** (5 keywords):
- Take the 5 highest `wilco_score` keywords
- These have the best combination of purchase rate, competitor coverage, and relevance
- These are your bread-and-butter exact match targets

**Campaign B — Volume Converters** (5 keywords):
- From remaining keywords, take the 5 with highest search_volume that also have purchase_rate >= 3%
- These provide reach while maintaining conversion quality

**Campaign C — Niche Converters** (5 keywords):
- From remaining keywords, take 5 with highest purchase_rate regardless of volume
- These are low-competition, high-conversion gems
- If search_volume < 1000, can group up to 10 keywords per Wilco's rule for low-volume terms

**Product Targeting — Top 10 ASINs:**
- Take the top 10 competitor ASINs by revenue from Phase 1
- Max 10 per Wilco's rule (not 15-35 like Kunze)
- Bid at average CPC

### Phase 4: Generate Amazon Bulksheets 2.0 CSV

**STRONGLY RECOMMENDED: Use Python `csv.writer` to generate the CSV reliably.**

Create the output directory first: `mkdir -p ./campaigns`

Generate a single CSV with all 5 campaigns using the 29-column Amazon Bulksheets 2.0 format:

```
Product,Entity,Operation,Campaign Id,Ad Group Id,Portfolio Id,Ad Id,Keyword Id,Product Targeting Id,Campaign Name,Ad Group Name,Start Date,End Date,Targeting Type,State,Daily Budget,sku,asin,Ad Group Default Bid,Bid,Keyword Text,Match Type,Bidding Strategy,Placement,Percentage,Product Targeting Expression,Audience ID,Shopper Cohort Percentage,Shopper Cohort Type
```

**Campaign naming convention:** `[Product Name] - [Type]`
- Example: `Bamboo Cutting Board - Auto`, `Bamboo Cutting Board - Exact A`, etc.

**Date format:** `YYYYMMDD` (use today's date)

**Budget and bid settings:**

| Setting | Conservative | Aggressive |
|---|---|---|
| Daily Budget per campaign | 10 | 20 |
| Keyword bids | At suggested CPC | 10% above suggested CPC |
| Auto campaign default bid | 0.50 | 0.75 |
| Product targeting bid | At avg CPC | At avg CPC |

**Use Python csv.writer for reliable column counts:**

```python
import csv
from datetime import date

HEADER = ['Product','Entity','Operation','Campaign Id','Ad Group Id','Portfolio Id','Ad Id','Keyword Id','Product Targeting Id','Campaign Name','Ad Group Name','Start Date','End Date','Targeting Type','State','Daily Budget','sku','asin','Ad Group Default Bid','Bid','Keyword Text','Match Type','Bidding Strategy','Placement','Percentage','Product Targeting Expression','Audience ID','Shopper Cohort Percentage','Shopper Cohort Type']
COLS = len(HEADER)  # 29
today = date.today().strftime('%Y%m%d')

def make_row(**kwargs):
    row = [''] * COLS
    field_map = {
        'product': 0, 'entity': 1, 'operation': 2, 'campaign_id': 3,
        'ad_group_id': 4, 'portfolio_id': 5, 'ad_id': 6, 'keyword_id': 7,
        'pt_id': 8, 'campaign_name': 9, 'ad_group_name': 10, 'start_date': 11,
        'end_date': 12, 'targeting_type': 13, 'state': 14, 'daily_budget': 15,
        'sku': 16, 'asin': 17, 'ag_default_bid': 18, 'bid': 19,
        'keyword_text': 20, 'match_type': 21, 'bidding_strategy': 22,
        'placement': 23, 'percentage': 24, 'pt_expression': 25,
        'audience_id': 26, 'shopper_pct': 27, 'shopper_type': 28
    }
    for k, v in kwargs.items():
        if k in field_map:
            row[field_map[k]] = str(v)
    return row

rows = []

# Example: Campaign row
rows.append(make_row(
    product='Sponsored Products', entity='Campaign', operation='Create',
    campaign_id=name, campaign_name=name, start_date=today,
    targeting_type='Auto', state='Enabled', daily_budget=budget,
    bidding_strategy='Dynamic bids - down only'
))

with open(output_path, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(HEADER)
    writer.writerows(rows)
```

#### Campaign 1: Auto Campaign

Structure: Campaign + Ad Group + Product Ad + 4 Auto Targeting rows

The auto campaign uses Amazon's automatic targeting. **Always include the 4 auto targeting expression rows** with individual bids so you can control and optimize each targeting type independently.

- Targeting Type: Auto
- 1 ad group, 1 product ad, **4 product targeting rows**
- Default bid: $0.50 (conservative) or $0.75 (aggressive)
- Dynamic bids - down only
- Auto targeting expressions (each gets its own Product Targeting row):
  - `close-match` — keywords closely related to your product
  - `loose-match` — keywords loosely related to your product
  - `substitutes` — shoppers viewing similar products
  - `complements` — shoppers viewing complementary products

#### Campaign 2: Product Targeting (Max 10 ASINs)

Structure: Campaign + Ad Group + Product Ad + up to 10 Product Targeting rows

- Targeting Type: Manual
- Max 10 competitor ASINs (Wilco's hard limit)
- Product Targeting Expression: `asin="BXXXXXXXXXX"` per ASIN
- Bid at average CPC

**CSV escaping:** The `asin="BXXXXXXXXXX"` value contains double quotes. Use `csv.writer` which handles escaping automatically, or manually escape: `"asin=""BXXXXXXXXXX"""`

#### Campaign 3: Manual Exact — Top Converters (Campaign A)

Structure: Campaign + Ad Group + Product Ad + 5 Keyword rows (all Exact match)

- Targeting Type: Manual
- 5 keywords with highest `wilco_score`
- Match Type: **Exact** (always)
- Individual bids per keyword at their CPC from research

#### Campaign 4: Manual Exact — Volume Converters (Campaign B)

Same structure as Campaign A, with 5 different keywords selected for volume + conversion.

#### Campaign 5: Manual Exact — Niche Converters (Campaign C)

Same structure as Campaign A, with 5 keywords selected for highest purchase rate. Can stretch to 10 keywords if all are low-volume (<1,000 search volume) per Wilco's rule.

#### CSV Generation Rules

1. **Save as UTF-8 CSV:** `./campaigns/[keyword]-wilco-campaigns.csv`
2. **All 5 campaigns in a single file**
3. **No dollar signs** — numeric values only for budget and bid
4. **Campaign ID = Campaign Name** for new campaigns
5. **Ad Group ID = Ad Group Name**
6. **Match Type is ALWAYS `Exact`** for keyword campaigns — this is the defining rule of Wilco's method
7. **Max 5 keywords per keyword campaign** (can be 10 for Campaign C if all low-volume)
8. **Max 10 product targets** in the ASIN campaign

#### CSV Validation (REQUIRED)

After generating the CSV, **always verify column counts** before presenting to the user:

```bash
awk -F',' '{print NF}' ./campaigns/[keyword]-wilco-campaigns.csv | sort | uniq -c
```

**Expected output:** All rows should show `29`. If any row has a different count, fix it before proceeding.

Also verify:
- Total row count matches expected: header + 7 (Auto) + 13 (ASIN) + 8×3 (Exact A/B/C) = 45 rows typical
- All keyword rows have Match Type = `Exact`
- No empty Campaign Name or Ad Group Name fields

### Phase 5: HTML Summary Artifact

Present a comprehensive HTML summary to the user. Structure:

```html
<html>
<head>
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; background: #f8f9fa; }
  .header { background: linear-gradient(135deg, #1a237e 0%, #283593 100%); color: white; padding: 24px; border-radius: 12px; margin-bottom: 24px; }
  .header h1 { margin: 0 0 8px 0; font-size: 24px; }
  .header .subtitle { opacity: 0.85; font-size: 14px; }
  .header .method-badge { display: inline-block; background: rgba(255,255,255,0.2); padding: 4px 12px; border-radius: 16px; font-size: 12px; margin-top: 8px; }
  .card { background: white; border-radius: 10px; padding: 20px; margin-bottom: 16px; box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
  .card h2 { margin-top: 0; font-size: 18px; color: #1a237e; border-bottom: 2px solid #3f51b5; padding-bottom: 8px; }
  .scorecard { display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 12px; }
  .metric { background: #f0f2f5; padding: 14px; border-radius: 8px; text-align: center; }
  .metric .value { font-size: 22px; font-weight: 700; color: #1a237e; }
  .metric .label { font-size: 11px; color: #666; margin-top: 4px; }
  table { width: 100%; border-collapse: collapse; font-size: 13px; }
  th { background: #1a237e; color: white; padding: 10px 12px; text-align: left; }
  td { padding: 8px 12px; border-bottom: 1px solid #e8e8e8; }
  tr:hover td { background: #e8eaf6; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 11px; font-weight: 600; }
  .badge-a { background: #e8f5e9; color: #2e7d32; }
  .badge-b { background: #e3f2fd; color: #1565c0; }
  .badge-c { background: #fff3e0; color: #e65100; }
  .badge-proven { background: #f3e5f5; color: #7b1fa2; }
  .campaign-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid #eee; }
  .campaign-name { font-weight: 600; }
  .campaign-details { font-size: 13px; color: #666; }
  .budget-summary { background: #e8eaf6; border-left: 4px solid #3f51b5; padding: 16px; border-radius: 0 8px 8px 0; margin-top: 16px; }
  .warning { background: #fff3e0; border-left: 4px solid #ff9800; padding: 12px 16px; border-radius: 0 8px 8px 0; font-size: 13px; margin-top: 12px; }
  .file-link { background: #e8f5e9; padding: 12px 16px; border-radius: 8px; font-family: monospace; font-size: 13px; }
</style>
</head>
<body>
  <div class="header">
    <h1>Wilco's Ad Setup: [Product Name]</h1>
    <div class="subtitle">PPC Launch Package &mdash; [Date] &mdash; Keyword: "[main_keyword]"</div>
    <div class="method-badge">Analytics-Driven | Exact Match | Max Control</div>
  </div>

  <!-- Market Scorecard -->
  <div class="card">
    <h2>Market Scorecard</h2>
    <div class="scorecard">
      <div class="metric"><div class="value">[search_volume]</div><div class="label">Search Volume</div></div>
      <div class="metric"><div class="value">$[avg_price]</div><div class="label">Avg Price</div></div>
      <div class="metric"><div class="value">[avg_reviews]</div><div class="label">Avg Reviews</div></div>
      <div class="metric"><div class="value">[opportunity_score]/100</div><div class="label">Opportunity</div></div>
      <div class="metric"><div class="value">[market_grade]</div><div class="label">Market Grade</div></div>
      <div class="metric"><div class="value">$[avg_cpc]</div><div class="label">Avg CPC</div></div>
    </div>
  </div>

  <!-- Keyword List with Analytics Scores -->
  <div class="card">
    <h2>Keyword Selection (Analytics-Ranked)</h2>
    <table>
      <tr><th>Keyword</th><th>Volume</th><th>Purchase %</th><th>CPC</th><th>Coverage</th><th>Wilco Score</th><th>Campaign</th></tr>
      <!-- Top 15 keywords sorted by wilco_score -->
      <tr>
        <td>[keyword]</td><td>[vol]</td><td>[purch%]</td><td>$[cpc]</td>
        <td>[X/Y competitors]</td><td>[score]</td>
        <td><span class="badge badge-[a/b/c]">[A/B/C]</span></td>
      </tr>
    </table>
  </div>

  <!-- Campaign Preview -->
  <div class="card">
    <h2>Campaign Preview (5 Campaigns)</h2>
    <!-- List each campaign with details -->
  </div>

  <!-- Risk/Velocity Warning -->
  <div class="card">
    <h2>Strategy Notes</h2>
    <div class="warning">
      <strong>Expected behavior with Wilco's method:</strong> Sales velocity will be lower than broad/phrase methods during the first 2-4 weeks. This is normal. The trade-off is lower wasted ad spend, no need for negative keyword management, and precise budget control. Monitor conversion rates and CTR per keyword — these are your primary optimization levers.
    </div>
  </div>

  <!-- Budget + Files -->
  <div class="card">
    <h2>Budget Estimate</h2>
    <div class="budget-summary">...</div>
  </div>

  <div class="card">
    <h2>Generated Files</h2>
    <div class="file-link">campaigns/[keyword]-wilco-campaigns.csv</div>
  </div>

  <!-- Analytics Checklist -->
  <div class="card">
    <h2>Analytics Review Checklist (Post-Launch)</h2>
    <ol style="font-size:13px;color:#444;">
      <li><strong>Day 7:</strong> Check CTR per keyword. Pause any keyword with CTR below 0.15% after 1,000+ impressions.</li>
      <li><strong>Day 14:</strong> Check conversion rate per keyword. Keywords converting above average get bid increases; below average get bid decreases.</li>
      <li><strong>Day 21:</strong> Pull search term report from Auto campaign. Extract high-performing search terms and add them as new exact match keywords in Campaigns A/B/C.</li>
      <li><strong>Day 30:</strong> Full ACOS/ROAS review. Pause keywords with ACOS > 2x your target. Increase budget on campaigns with ACOS below target.</li>
      <li><strong>Ongoing:</strong> Use search term mining from Auto to feed new exact keywords into manual campaigns. This is the analytics flywheel of Wilco's method.</li>
    </ol>
  </div>
</body>
</html>
```

## Sub-Agent Delegation Summary

| Phase | Delegate? | Why | Sub-agent Type |
|---|---|---|---|
| Phase 1: Market Research | No | Small result, needed for orchestration | Main agent |
| Phase 2: Keyword Research | **YES** | Large results, 3-4 parallel batches | `general-purpose` per batch |
| Phase 3: Keyword Processing | Optional | Only if context is getting large | `general-purpose` |
| Phase 4: CSV Generation | Optional | Only if context is getting large | `general-purpose` |
| Phase 5: HTML Summary | No | Must be presented in main chat | Main agent |

## Bulksheet Format Quick Reference

### 29 Column Headers (in order)
```
Product | Entity | Operation | Campaign Id | Ad Group Id | Portfolio Id | Ad Id | Keyword Id | Product Targeting Id | Campaign Name | Ad Group Name | Start Date | End Date | Targeting Type | State | Daily Budget | sku | asin | Ad Group Default Bid | Bid | Keyword Text | Match Type | Bidding Strategy | Placement | Percentage | Product Targeting Expression | Audience ID | Shopper Cohort Percentage | Shopper Cohort Type
```

### Entity Types & Required Fields

| Entity | Required Fields (besides Product, Entity, Operation) |
|---|---|
| Campaign | Campaign Id, Campaign Name, Start Date, Targeting Type, State, Daily Budget, Bidding Strategy |
| Ad group | Campaign Id, Ad Group Id, Ad Group Name, State, Ad Group Default Bid |
| Product ad | Campaign Id, Ad Group Id, State, sku (sellers) or asin (vendors) |
| Keyword | Campaign Id, Ad Group Id, State, Bid, Keyword Text, Match Type |
| Product targeting | Campaign Id, Ad Group Id, State, Bid, Product Targeting Expression |

### Key Values for Wilco's Method

| Field | Value | Notes |
|---|---|---|
| Product | `Sponsored Products` | Always |
| Operation | `Create` | For new campaigns |
| Targeting Type | `Auto` or `Manual` | Auto for campaign 1 only |
| State | `Enabled` | Always for new campaigns |
| **Match Type** | **`Exact`** | **ALWAYS Exact for Wilco. Never Broad or Phrase.** |
| Bidding Strategy | `Dynamic bids - down only` | Conservative default |
| Product Targeting (ASIN) | `asin="BXXXXXXXXXX"` | Max 10 per campaign |
| Date format | `YYYYMMDD` | Today's date |

## Wilco vs Kunze Quick Comparison (for user context)

| Aspect | Wilco | Kunze |
|---|---|---|
| Match type | Exact only | Broad + Phrase |
| Keywords per campaign | Max 5 (strict) | Max 5 |
| ASINs per campaign | Max 10 | 15-35 |
| Keyword campaigns | 3 (all exact) | 2 (1 broad + 1 phrase) |
| Keyword selection | Analytics-driven (purchase rate, coverage) | Gap-hunting (low competition) |
| Negation needed | Minimal | Regular |
| Sales velocity | Lower | Higher |
| Risk | Lower | Higher |
| Analytics investment | High | Moderate |

## Tool Reference

| Tool | When to Use |
|---|---|
| `research_products(keyword, focus="balanced", product_limit=20)` | Phase 1: Market scan and competitor ASIN collection |
| `amazon_keyword_research(asins=[...], limit=50, min_search_volume=300)` | Phase 2: Keyword research (run in sub-agents, 3-4 ASINs per batch) |
| `Agent(subagent_type="general-purpose")` | Phase 2+: Delegate keyword research batches and heavy processing |

## Error Handling

- **Keyword research returns too many results**: Reduce `limit` to 30 or increase `min_search_volume` to 500
- **Fewer than 15 quality keywords found**: Reduce Campaign C to 3-4 keywords, or merge B and C into one campaign with up to 10 keywords
- **No competitor ASINs found**: Ask user for specific competitor ASINs
- **Sub-agent fails**: Retry once, skip and note in summary if second attempt fails
- **All keywords have purchase_rate < 1%**: The niche may have low commercial intent. Flag this in the summary and suggest the user verify demand
