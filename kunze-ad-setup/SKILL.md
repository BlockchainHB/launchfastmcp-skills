---
name: kunze-ad-setup
description: Use this skill when the user wants to set up Amazon PPC campaigns using Kunze's ecosystem-driven strategy. Takes a main keyword and product SKU, performs market research to find competitors, runs batched keyword research via sub-agents, builds a filtered keyword master list with gap analysis, then generates Amazon Bulksheets 2.0 CSV files for 5 Sponsored Products campaigns (Auto, Broad, Phrase, Category, ASIN Target). Also produces an HTML summary artifact with market scorecard, keyword list, campaign previews, and budget estimates. Powered by the LegacyX FBA methodology and LaunchFast MCP tools.
---

# Kunze's Ad Setup Skill

This skill implements Kunze's ecosystem-driven PPC launch strategy from the LegacyX FBA course. It creates a complete 5-campaign Sponsored Products launch package using Amazon Bulksheets 2.0 format, ready for upload to Amazon Ads Console.

## User Inputs Required

When the user activates this skill, collect these inputs:

| Input | Required | Example | Notes |
|---|---|---|---|
| **Main keyword** | Yes | `bamboo cutting board` | The primary search term for the product niche |
| **Product SKU** | Yes | `BB-CUT-001` | The user's seller SKU for the product ad |
| **Budget level** | No | `conservative` or `aggressive` | Defaults to conservative if not specified |
| **Product name** | No | `Bamboo Cutting Board` | Used in campaign naming. If not given, derive from keyword |

## Strategy Overview (Kunze's Method)

Kunze's approach is **ecosystem-driven** using **phrase and broad match** to cast a wider net and capture how people actually shop on Amazon. Key principles:

- **Broad/phrase match focused** — captures keyword variations Amazon associates with your product
- **1 ad group per campaign** — Amazon is bad at distributing budget across multiple ad groups
- **Max 5 keywords per keyword campaign** — prevents budget dilution across too many keywords
- **Dynamic bids down only** — conservative bidding that reduces bids when conversion is unlikely
- **Keyword stacking** — mix 1 high-volume keyword with a few mid/low-volume keywords per campaign
- **Gap analysis** — find keywords competitors don't rank well for (higher opportunity, lower CPC)

## Workflow

### Phase 1: Market Research & Competitor Collection

Run `research_products` to identify the top competitors and collect ASINs:

```
research_products(keyword="<main_keyword>", focus="balanced", product_limit=20)
```

**From the results, collect:**
1. Top 10-15 ASINs sorted by `monthly_revenue` descending
2. Market-level data: `avg_price`, `avg_reviews`, `search_volume`, `opportunity_score`, `market_grade`
3. Category information for the Category targeting campaign
4. Note the dominant brand and brand concentration

**Store these for later phases:**
- `competitor_asins`: Array of 10-15 top ASINs
- `market_data`: The market-level summary object
- `product_list`: Full product array with revenue, reviews, price per product

### Phase 2: Batched Keyword Research (SUB-AGENT DELEGATION)

**CRITICAL: This phase MUST be delegated to sub-agents to protect the main context window.**

Keyword research returns very large result sets. Running it in the main context will consume too much of the context window. Instead, split the competitor ASINs into batches of 3-4 and delegate each batch to a sub-agent.

**Batching rules:**
- Maximum 3-4 ASINs per batch
- Use `limit=50` per batch to keep results manageable
- Use `min_search_volume=300` to filter out ultra-low-volume keywords
- Run batches in parallel for speed
- Each sub-agent should return a filtered, deduplicated keyword list

**Sub-agent prompt template for each batch:**

```
You are a keyword research assistant. Run the following keyword research and return a clean filtered list.

Run this tool call:
amazon_keyword_research(asins=["ASIN1", "ASIN2", "ASIN3"], limit=50, min_search_volume=300)

From the results, extract the top 20 keywords sorted by search_volume descending.

Return results as a STRUCTURED LIST with one keyword per line in this exact format:
KEYWORD | search_volume | cpc | purchase_rate | relevance_score | ASIN1_rank | ASIN2_rank | ASIN3_rank

Example:
bamboo cutting board | 125235 | 1.28 | 5.33 | 892 | 12 | 45 | -
cutting boards for kitchen | 321719 | 2.39 | 3.61 | 756 | 8 | - | 23

Use "-" if an ASIN is not ranking. Use pipe delimiters (|) for easy parsing.

Filter OUT keywords that:
- Have search_volume below 300
- Are brand names of competitors (e.g., Epicurean, Boos, specific brand names)
- Are completely irrelevant to the product category
- Are single generic words (e.g., "board", "wood", "bamboo")

Also report on separate lines:
TOTAL_FOUND: [number]
PASSED_FILTERS: [number]
TOP_PURCHASE_RATE: [keyword1], [keyword2], [keyword3]
```

**Launch sub-agents like this (example with 12 ASINs = 3 batches):**

```
# Launch all 3 in parallel using the Agent tool:
Agent(subagent_type="general-purpose", name="kw-batch-1", prompt="<template with ASINs 1-4>")
Agent(subagent_type="general-purpose", name="kw-batch-2", prompt="<template with ASINs 5-8>")
Agent(subagent_type="general-purpose", name="kw-batch-3", prompt="<template with ASINs 9-12>")
```

### Phase 3: Build Keyword Master List

After all sub-agent batches return, merge and process their results:

**3a. Merge & Deduplicate**
- Combine all keyword arrays from all batches
- Deduplicate by keyword text (case-insensitive)
- For duplicates: keep the entry with highest search volume, merge competitor rank data from all appearances

**3b. Overlap Scoring**
- Keywords appearing in multiple batches are stronger signals
- Add an `overlap_count` field (how many batches contained this keyword)
- Keywords with overlap_count >= 2 get priority

**3c. Gap Analysis**
- For each keyword, check competitor ranks
- **Gap keyword**: majority of competitors have rank > 50 or don't rank at all
- **Opportunity score** = `search_volume * (1 / avg_competitor_rank)` — higher = better opportunity
- Flag keywords where most competitors rank poorly as "gap opportunities"

**3d. Final Filtering & Categorization**
Categorize keywords into tiers:
- **High volume**: search_volume >= 5,000
- **Mid volume**: search_volume 1,000 - 4,999
- **Low volume**: search_volume 300 - 999

**3e. Select Campaign Keywords**
Apply Kunze's keyword stacking rules:

For **Broad campaign** (5 keywords max):
- 1-2 high-volume keywords
- 2-3 mid-volume keywords
- Prioritize gap opportunity keywords
- Use the `cpc`/`suggested_bid` from research as starting bid

For **Phrase campaign** (5 keywords max):
- Same 5 keywords as Broad campaign OR different 5 from master list
- Using phrase match with these keywords provides tighter targeting
- Use the `cpc`/`suggested_bid` from research as starting bid

For **ASIN targeting campaign**:
- **Default: Top 15 ASINs** by monthly revenue from Phase 1
- Expand to 20+ only if the user provides additional specific competitor ASINs
- Bid at or slightly below the average CPC from keyword research

For **Category targeting campaign**:
- Use the product category identified in Phase 1
- Target the broadest relevant category

### Phase 4: Generate Amazon Bulksheets 2.0 CSV Files

**IMPORTANT: This phase can also be delegated to a sub-agent if context is getting large.**

Generate a single CSV file containing all 5 campaigns. The CSV must use the exact Amazon Bulksheets 2.0 format with these 29 columns in order:

```
Product,Entity,Operation,Campaign Id,Ad Group Id,Portfolio Id,Ad Id,Keyword Id,Product Targeting Id,Campaign Name,Ad Group Name,Start Date,End Date,Targeting Type,State,Daily Budget,sku,asin,Ad Group Default Bid,Bid,Keyword Text,Match Type,Bidding Strategy,Placement,Percentage,Product Targeting Expression,Audience ID,Shopper Cohort Percentage,Shopper Cohort Type
```

**Campaign naming convention:** `[Product Name] - [Type]`
- Example: `Bamboo Cutting Board - Auto`, `Bamboo Cutting Board - Broad`, etc.

**Date format:** `YYYYMMDD` (use today's date for Start Date)

**Budget and bid settings by level:**

| Setting | Conservative | Aggressive |
|---|---|---|
| Daily Budget per campaign | 10 | 20 |
| Keyword bids | At suggested CPC | 10% above suggested CPC |
| Auto campaign default bid | 0.50 | 0.75 |
| ASIN targeting bid | At avg CPC | At avg CPC |
| Category targeting bid | At avg CPC | At avg CPC |

**CRITICAL: Before generating the CSV, create the output directory:**
```
mkdir -p ./campaigns
```

#### Campaign 1: Auto Campaign

Structure: Campaign + Ad Group + Product Ad + **4 Auto Targeting rows** (7 rows total)

**Always include the 4 auto targeting expression rows** with individual bids so you can control and optimize each targeting type independently:
- `close-match` — keywords closely related to your product
- `loose-match` — keywords loosely related to your product
- `substitutes` — shoppers viewing similar products
- `complements` — shoppers viewing complementary products

Each targeting type gets its own Product Targeting row with the expression in the Product Targeting Expression column and an individual bid.

#### Campaign 2: Manual Broad Match

Structure: Campaign + Ad Group + Product Ad + 5 Keywords (8 rows total)

- Targeting Type: Manual
- 5 keywords with Broad match type
- Individual bids per keyword at their CPC from research

#### Campaign 3: Manual Phrase Match

Structure: Campaign + Ad Group + Product Ad + 5 Keywords (8 rows total)

Same structure as Broad but with `Phrase` match type. Can use the same 5 keywords or a different set of 5 from the master list.

#### Campaign 4: Category Targeting

Structure: Campaign + Ad Group + Product Ad + 1 Product Targeting (4 rows total)

- Targeting Type: Manual
- Product Targeting Expression: `category="[CATEGORY_ID]"`
- Bid at average CPC

**Note on Category ID:** The category ID must come from the user or from Amazon's category taxonomy. If the user doesn't provide it, include a placeholder `[CATEGORY_ID]` and instruct the user to find it by:
1. Go to Amazon Ads Console > Campaign Manager > Create Campaign > Manual Targeting > Product Targeting
2. Browse or search categories to find the relevant one
3. The category ID is the numeric ID shown (e.g., `category="5524098011"`)
4. Alternatively, download an existing campaign's bulksheet that has category targeting — the ID will be in the Product Targeting Expression column

#### Campaign 5: ASIN Targeting

Structure: Campaign + Ad Group + Product Ad + 15 Product Targeting rows (18 rows total)

- Targeting Type: Manual
- **Default: Top 15 ASINs** by monthly revenue from Phase 1. Expand to 20+ only if user provides additional specific competitor ASINs.
- Product Targeting Expression: `asin="BXXXXXXXXXX"` per ASIN
- Bid at average CPC

**CSV escaping for ASIN expressions:** The value `asin="B071NV54JD"` contains double quotes. Use Python's `csv.writer` which handles escaping automatically.

#### CSV Generation Rules

1. **Save as UTF-8 CSV** in the project directory: `./campaigns/[keyword]-kunze-campaigns.csv`
2. **All 5 campaigns in a single file** — Amazon supports multiple campaigns per upload
3. **No dollar signs or currency symbols** — numeric values only for budget and bid fields
4. **Empty fields** — leave truly empty (no space, no quote), just consecutive commas
5. **Campaign ID = Campaign Name** — for new campaigns, use the same text-based name in both fields
6. **Ad Group ID = Ad Group Name** — same rule applies
7. **Dates** — format `YYYYMMDD`, use today's date

#### CSV Validation (REQUIRED)

After generating the CSV, **always verify column counts** before presenting to the user:

```bash
awk -F',' '{print NF}' ./campaigns/[keyword]-kunze-campaigns.csv | sort | uniq -c
```

**Expected output:** All rows should show `29`. If any row has a different count, fix it before proceeding.

Also verify:
- Total row count matches expected: header + 7 (Auto) + 8 (Broad) + 8 (Phrase) + 4 (Category) + 18 (ASIN) = 46 rows typical
- Broad keywords have Match Type = `Broad`, Phrase keywords have Match Type = `Phrase`
- No empty Campaign Name or Ad Group Name fields

**REQUIRED: Use Python csv.writer to generate the CSV.** Manual comma counting is error-prone and has caused column misalignment bugs. Use the `make_row()` helper pattern for reliable output:

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

# Campaign row example:
rows.append(make_row(
    product='Sponsored Products', entity='Campaign', operation='Create',
    campaign_id=name, campaign_name=name, start_date=today,
    targeting_type='Auto', state='Enabled', daily_budget=budget,
    bidding_strategy='Dynamic bids - down only'
))

# Ad group row example:
rows.append(make_row(
    product='Sponsored Products', entity='Ad group', operation='Create',
    campaign_id=name, ad_group_id=ag_name, ad_group_name=ag_name,
    state='Enabled', ag_default_bid=default_bid
))

# Keyword row example (for Broad/Phrase campaigns):
rows.append(make_row(
    product='Sponsored Products', entity='Keyword', operation='Create',
    campaign_id=name, ad_group_id=ag_name, state='Enabled',
    bid=keyword_bid, keyword_text=keyword, match_type='Broad'
))

# ASIN targeting row example:
rows.append(make_row(
    product='Sponsored Products', entity='Product targeting', operation='Create',
    campaign_id=name, ad_group_id=ag_name, state='Enabled',
    bid=asin_bid, pt_expression=f'asin="{asin}"'
))

with open(output_path, 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(HEADER)
    writer.writerows(rows)
```

**NEVER use inline CSV string concatenation with commas.** Always use the `make_row()` helper above.

### Phase 5: Pro-Tip Low-Bid Auto Campaign (Optional)

Per Kunze's pro tip: create an additional auto campaign with very low bids ($0.20-$0.25) to catch high-ROAS traffic during low-competition hours. Only include this if the user selected `aggressive` budget level or explicitly requests it.

Structure: Campaign + Ad Group + Product Ad (3 rows)
- Campaign Name: `[ProductName] - Auto Low Bid`
- Targeting Type: Auto
- Daily Budget: $5
- Default bid: $0.22
- Use `make_row()` helper — same pattern as Campaign 1 but with lower bids

### Phase 6: HTML Summary Artifact

After generating the CSV, present a comprehensive HTML summary to the user. This should be displayed inline in the chat as an HTML artifact.

**HTML template structure:**

```html
<html>
<head>
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; background: #f8f9fa; }
  .header { background: linear-gradient(135deg, #232f3e 0%, #37475a 100%); color: white; padding: 24px; border-radius: 12px; margin-bottom: 24px; }
  .header h1 { margin: 0 0 8px 0; font-size: 24px; }
  .header .subtitle { opacity: 0.85; font-size: 14px; }
  .card { background: white; border-radius: 10px; padding: 20px; margin-bottom: 16px; box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
  .card h2 { margin-top: 0; font-size: 18px; color: #232f3e; border-bottom: 2px solid #ff9900; padding-bottom: 8px; }
  .scorecard { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 12px; }
  .metric { background: #f0f2f5; padding: 14px; border-radius: 8px; text-align: center; }
  .metric .value { font-size: 24px; font-weight: 700; color: #232f3e; }
  .metric .label { font-size: 12px; color: #666; margin-top: 4px; }
  table { width: 100%; border-collapse: collapse; font-size: 13px; }
  th { background: #232f3e; color: white; padding: 10px 12px; text-align: left; }
  td { padding: 8px 12px; border-bottom: 1px solid #e8e8e8; }
  tr:hover td { background: #fff8f0; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 11px; font-weight: 600; }
  .badge-high { background: #e8f5e9; color: #2e7d32; }
  .badge-mid { background: #fff3e0; color: #e65100; }
  .badge-low { background: #fce4ec; color: #c62828; }
  .badge-gap { background: #e3f2fd; color: #1565c0; }
  .campaign-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid #eee; }
  .campaign-name { font-weight: 600; }
  .campaign-details { font-size: 13px; color: #666; }
  .budget-summary { background: #fff8e1; border-left: 4px solid #ff9900; padding: 16px; border-radius: 0 8px 8px 0; margin-top: 16px; }
  .file-link { background: #e8f5e9; padding: 12px 16px; border-radius: 8px; font-family: monospace; font-size: 13px; }
</style>
</head>
<body>
  <div class="header">
    <h1>Kunze's Ad Setup: [Product Name]</h1>
    <div class="subtitle">PPC Launch Package — [Date] — Keyword: "[main_keyword]"</div>
  </div>

  <!-- Market Scorecard -->
  <div class="card">
    <h2>Market Scorecard</h2>
    <div class="scorecard">
      <div class="metric"><div class="value">[search_volume]</div><div class="label">Search Volume</div></div>
      <div class="metric"><div class="value">$[avg_price]</div><div class="label">Avg Price</div></div>
      <div class="metric"><div class="value">[avg_reviews]</div><div class="label">Avg Reviews</div></div>
      <div class="metric"><div class="value">[opportunity_score]/100</div><div class="label">Opportunity Score</div></div>
      <div class="metric"><div class="value">[market_grade]</div><div class="label">Market Grade</div></div>
      <div class="metric"><div class="value">$[avg_cpc]</div><div class="label">Avg CPC</div></div>
    </div>
  </div>

  <!-- Keyword Master List (top 20) -->
  <div class="card">
    <h2>Keyword Master List (Top 20)</h2>
    <table>
      <tr><th>Keyword</th><th>Volume</th><th>CPC</th><th>Tier</th><th>Gap?</th><th>Selected For</th></tr>
      <!-- Repeat for each keyword -->
      <tr>
        <td>[keyword]</td>
        <td>[volume]</td>
        <td>$[cpc]</td>
        <td><span class="badge badge-[tier]">[TIER]</span></td>
        <td>[Yes/No]</td>
        <td>[Broad, Phrase, or —]</td>
      </tr>
    </table>
  </div>

  <!-- Campaign Preview -->
  <div class="card">
    <h2>Campaign Preview (5 Campaigns)</h2>
    <!-- Repeat for each campaign -->
    <div class="campaign-row">
      <div>
        <div class="campaign-name">[Campaign Name]</div>
        <div class="campaign-details">[Type] | [X] entities | Bid: $[bid] | Budget: $[budget]/day</div>
      </div>
    </div>
  </div>

  <!-- Budget Summary -->
  <div class="card">
    <h2>Budget Estimate</h2>
    <div class="budget-summary">
      <strong>[Conservative/Aggressive] Plan</strong><br>
      Daily: $[total_daily] across 5 campaigns<br>
      Monthly estimate: ~$[monthly_estimate]<br>
      Starting bids: [at/above] suggested CPC ($[avg_cpc])
    </div>
  </div>

  <!-- File Output -->
  <div class="card">
    <h2>Generated Files</h2>
    <div class="file-link">campaigns/[keyword]-kunze-campaigns.csv</div>
    <p style="font-size:13px;color:#666;margin-top:8px;">Upload this CSV to Amazon Ads Console > Bulk Operations > Upload</p>
  </div>
</body>
</html>
```

## Sub-Agent Delegation Summary

To manage context window limits, delegate these phases to sub-agents:

| Phase | Delegate? | Why | Sub-agent Type |
|---|---|---|---|
| Phase 1: Market Research | No | Small result, needed for orchestration | Main agent |
| Phase 2: Keyword Research | **YES** | Large results, 3-4 parallel batches | `general-purpose` per batch |
| Phase 3: Keyword Processing | Optional | Only if main context is getting large | `general-purpose` |
| Phase 4: CSV Generation | Optional | Only if main context is getting large | `general-purpose` |
| Phase 5: Low-Bid Auto | No | Trivial, just extra rows | Main agent |
| Phase 6: HTML Summary | No | Must be presented in main chat | Main agent |

**Minimum sub-agents needed:** 3-4 (one per keyword research batch)
**Maximum sub-agents:** 6-7 (if also delegating processing and CSV generation)

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
| Negative keyword | Campaign Id, Ad Group Id, State, Keyword Text, Match Type (negativeExact/negativePhrase) |
| Campaign negative keyword | Campaign Id, State, Keyword Text, Match Type (negativeExact/negativePhrase) |
| Bidding adjustment | Campaign Id, Placement, Percentage |

### Key Values

| Field | Allowed Values |
|---|---|
| Product | `Sponsored Products` |
| Operation | `Create`, `Update`, `Archive` |
| Targeting Type | `Auto`, `Manual` |
| State | `Enabled`, `Paused` |
| Match Type (keywords) | `Exact`, `Broad`, `Phrase` |
| Match Type (negative) | `negativeExact`, `negativePhrase` |
| Bidding Strategy | `Dynamic bids - down only`, `Dynamic bids - up and down`, `Fixed bid` |
| Placement | `placement top`, `placement product page`, `placement rest of search(BETA)` |
| Product Targeting (ASIN) | `asin="BXXXXXXXXXX"` |
| Product Targeting (category) | `category="XXXX"` |
| Product Targeting (auto) | `close-match`, `loose-match`, `substitutes`, `complements` |
| Date format | `YYYYMMDD` |
| Budget/Bid | Numeric only, no symbols, use decimal point |

## Tool Reference

| Tool | When to Use |
|---|---|
| `research_products(keyword, focus="balanced", product_limit=20)` | Phase 1: Market scan and competitor ASIN collection |
| `amazon_keyword_research(asins=[...], limit=50, min_search_volume=300)` | Phase 2: Keyword research (run in sub-agents, 3-4 ASINs per batch) |
| `Agent(subagent_type="general-purpose")` | Phase 2-4: Delegate keyword research batches and heavy processing |

## Error Handling

- **Keyword research returns too many results**: Reduce `limit` to 30 or increase `min_search_volume` to 500
- **No competitor ASINs found**: Ask user for specific competitor ASINs they want to target
- **Category ID unknown**: Leave as placeholder `[CATEGORY_ID]` in CSV with instructions for user to fill in from Amazon Ads console
- **Sub-agent fails**: Retry the failed batch once. If it fails again, skip that batch and note it in the summary
- **Less than 5 good keywords found**: Use what's available. A campaign can have fewer than 5 keywords — it just can't have more
