---
name: product-research
description: Use this skill when the user wants to research Amazon FBA products, evaluate a niche or keyword for viability, find product opportunities, or validate whether a product idea meets launch criteria. Covers market analysis, competitor assessment, profitability estimation, keyword validation, and the "rabbit hole" discovery strategy. Powered by the LegacyX FBA methodology and LaunchFast MCP tools.
---

# Amazon FBA Product Research Skill

This skill implements the LegacyX FBA product research methodology using LaunchFast MCP tools. It guides structured evaluation of Amazon niches and products to identify profitable launch opportunities.

## Core Research Criteria (LegacyX FBA)

Every product opportunity MUST be evaluated against these criteria. These are non-negotiable baselines:

| Criteria | Threshold | How to Check with LaunchFast |
|---|---|---|
| **Total niche revenue** | > $200,000/month | Sum `monthly_revenue` across top 20 products from `research_products` (balanced, product_limit=20) |
| **Average price** | $25+ minimum ($40+ preferred) | `market.avg_price` from `research_products` |
| **Average reviews** | 500 or fewer | `market.avg_reviews` from `research_products` |
| **Avg revenue per seller** | $5,000+/month | Compute: total niche revenue / number of sellers |
| **Market not dominated** | Top 2-3 sellers < 50% of niche revenue | Check `market.brand_concentration` + sort products by `monthly_revenue` descending and calculate top-seller share |
| **Profit margin** | 30% minimum before ad costs | Manual calculation (see Profitability section below) |
| **Search volume exists** | NOT "N/A" | `market.search_volume` must be a real number. If 0 or missing, find the broader main keyword and re-research |

### Exceptions to the 500-Review Rule

In large markets (>$1M niche revenue), average reviews above 500 can be acceptable IF:
- Multiple low-review sellers (<200 reviews) are doing $30K-$40K+/month
- Use `research_products` with `focus=financial, product_limit=20` to verify low-review sellers with strong revenue

## Research Workflow

### Step 1: Initial Market Scan

When the user provides a keyword, run `research_products` with `focus=balanced` and `product_limit=20`:

```
research_products(keyword="<keyword>", focus="balanced", product_limit=20)
```

**Evaluate against criteria checklist:**

1. **Search volume** — Is `market.search_volume` a real number (not 0/null)?
   - If missing: the keyword is too specific. Ask the user for (or suggest) a broader keyword and re-run.
2. **Average price** — Is `market.avg_price` >= $25? Ideally >= $40.
   - Below $25 = margins too tight. Flag as a concern.
3. **Average reviews** — Is `market.avg_reviews` <= 500?
   - If above 500, check for the large-market exception (Step 1b).
4. **Opportunity score** — Report `market.opportunity_score` and `market.market_grade`.
5. **Brand concentration** — Report `market.brand_concentration` and `market.dominant_brand`.

**Calculate from product data:**

6. **Total niche revenue** — Sum all `monthly_revenue` values. Must be > $200K.
7. **Average revenue per seller** — Total revenue / number of products. Must be > $5K.
8. **Top-seller dominance** — Sort by `monthly_revenue` desc. Do the top 2-3 ASINs account for > 50% of total? If yes, flag.

### Step 1b: Large Market Exception Check (if avg reviews > 500)

If total niche revenue > $1M AND avg reviews > 500, look for low-review winners:
- Filter the product list for items with < 200 reviews
- Check if any have `monthly_revenue` > $30,000
- If multiple exist, the market may still be viable despite high average reviews

### Step 2: Financial Deep Dive

Run `research_products` with `focus=financial` to check trends:

```
research_products(keyword="<keyword>", focus="financial", product_limit=20)
```

**Evaluate:**
- **Sales trends** — How many products show `sales_trend_label: "declining"` vs `"stable"` or `"growing"`?
  - Majority declining = market may be shrinking. Flag as risk.
- **Month-over-month growth** — Check `month_over_month_growth_pct` across products.
  - Widespread negative MoM = seasonal or declining market.
- **7-day vs prior 7-day** — `trend_7d_vs_prev7d_pct` shows short-term momentum.

### Step 3: Competitive & Listing Quality Analysis

Run `research_products` with `focus=titles` to identify listing gaps:

```
research_products(keyword="<keyword>", focus="titles", product_limit=10)
```

**Look for:**
- Products with low `listing_quality_score` (below 60) but high revenue = opportunity to win with better listings
- `opportunity_flag: "listing_quality_gap"` = direct signal of beatable competitors
- Chinese sellers with poor English/images (differentiation opportunity per LegacyX method)

### Step 4: Keyword Validation

Pick 2-3 top ASINs (ideally a mix: one top performer, one low-review seller doing well) and run:

```
amazon_keyword_research(asins=["<ASIN1>", "<ASIN2>", "<ASIN3>"], limit=20)
```

**Evaluate:**
- **Main keyword search volume** — Confirms the market size is real.
- **Keyword diversity** — Are there multiple high-volume keywords? More keywords = more organic traffic paths.
- **CPC / sponsored_density** — High CPC + high density = expensive PPC. Factor into profitability.
- **Purchase rate** — Keywords with `purchase_rate` > 5% are high-intent buyers.
- **Ranking gaps** — ASINs with low rank on high-volume keywords = room to compete.

### Step 5: Profitability Estimation

This is a manual calculation using the data gathered. Present it clearly:

```
Selling Price:          $XX.XX  (use market avg or slightly below)
Amazon Fees (~15%):    -$X.XX
Est. Manufacturing:    -$X.XX  (user must provide or estimate)
Est. Shipping/unit:    -$X.XX  (use $2.50/kg as conservative default)
─────────────────────────────
Est. Profit/unit:       $X.XX
Profit Margin:          XX%    (MUST be >= 30%)
```

- Always calculate at or below the market average price (worst-case scenario)
- Use conservative shipping estimates — better to overestimate costs
- If margin < 30%, flag the product as likely not viable unless manufacturing costs can be reduced
- Remind the user: this is BEFORE advertising costs. Actual net margin will be lower during launch.

### Step 6 (Optional): Rabbit Hole Discovery

This is the LegacyX "rabbit hole" strategy — one product leads to variations and adjacent opportunities:

1. From Step 1 results, identify interesting product variations (different styles, materials, pack sizes)
2. Find the main keyword for each variation
3. Run `research_products` on those new keywords
4. Repeat — each market scan may reveal more adjacent niches

**When to use this:**
- User says "find me products" without a specific keyword
- Initial keyword didn't pass criteria but showed adjacent opportunities
- User wants to explore a category broadly

## Preferred Categories

When the user wants category-level exploration, start with these (per LegacyX method):
- Baby Products
- Home & Kitchen
- Office Products
- Patio, Lawn & Garden
- Pet Supplies
- Sports & Outdoors
- Toys & Games

**Avoid:** Products with a primary electronic function (higher return rates).

## How to Present Results

Always present a structured verdict:

### Market Scorecard Format

```
## Market Scorecard: [keyword]

| Criteria | Threshold | Actual | Status |
|---|---|---|---|
| Niche Revenue | > $200K | $XXX,XXX | PASS/FAIL |
| Avg Price | >= $25 | $XX.XX | PASS/FAIL |
| Avg Reviews | <= 500 | XXX | PASS/FAIL |
| Avg Rev/Seller | >= $5K | $X,XXX | PASS/FAIL |
| Top Seller Dominance | < 50% | XX% | PASS/FAIL |
| Search Volume | Exists | XX,XXX | PASS/FAIL |
| Est. Margin | >= 30% | XX% | PASS/ESTIMATE |

**Market Grade:** X | **Opportunity Score:** XX/100
**Brand Concentration:** low/medium/high
**Dominant Brand:** [name]

### Verdict: [VIABLE / MARGINAL / NOT RECOMMENDED]
[1-2 sentence summary of why]
```

### Trend Summary

After the scorecard, include:
- How many products are growing vs declining vs stable
- Any short-term momentum shifts (7d trends)
- Seasonal risk flags if widespread MoM decline

### Actionable Next Steps

Always end with clear next steps:
- If VIABLE: "Run supplier research to estimate manufacturing costs" or "Track top ASINs with rank tracker"
- If MARGINAL: "Try [broader/narrower keyword] to see if adjacent niche is stronger"
- If NOT RECOMMENDED: "Here are 2-3 adjacent keywords worth exploring instead: ..."

## Tool Reference

| Tool | When to Use |
|---|---|
| `research_products(keyword, focus="balanced", product_limit=20)` | Initial market scan — always start here |
| `research_products(keyword, focus="financial", product_limit=20)` | Trend/revenue deep dive |
| `research_products(keyword, focus="titles", product_limit=10)` | Listing quality gap analysis |
| `research_products(keyword, focus="images", product_limit=10)` | Visual differentiation audit |
| `amazon_keyword_research(asins=[...], limit=20)` | Keyword validation + ranking gaps |
| `ip_check_manage(action="ip_conflict_check", keyword="...")` | Check for trademark/patent conflicts before committing |
| `supplier_research(keyword="...")` | Estimate manufacturing costs for profitability calc |
| `rank_tracker_manage(action="track", asin="...")` | Track promising ASINs over time (max 3 slots) |
