# LaunchFast MCP Skills — AI-Powered Amazon Seller Tools for Claude Code

> The fastest Amazon FBA research agent on the market. Product research, PPC keyword research, Alibaba supplier outreach, and IP checks — all running inside Claude Code via the LaunchFast MCP.

[![Watch the demo](https://img.youtube.com/vi/A58hnQ4AYfQ/maxresdefault.jpg)](https://www.youtube.com/watch?v=A58hnQ4AYfQ)

---

## 🔑 Get Access

These skills require the **LaunchFast MCP** — a free add-on for LaunchFast subscribers.

**[Sign up at launchfastlegacyx.com →](https://launchfastlegacyx.com)**

Already a member? [Get your MCP credentials →](https://launchfastlegacyx.com/dashboard/settings)

---

## What is LaunchFast MCP?

LaunchFast is an Amazon product research tool trusted by FBA sellers to find winning products, analyze keywords, source suppliers, and run IP checks — all from one dashboard.

The **LaunchFast MCP** (Model Context Protocol) connects LaunchFast's data directly into Claude Code, turning your AI assistant into a full Amazon seller agent. No more copy-pasting between tools. Just type a command and get a complete FBA research report.

Think of it as **Helium 10 or Jungle Scout — but running natively inside Claude**, with automation that can find products, contact suppliers, and build PPC campaigns in a single session.

---

## Skills

### `/launchfast-product-research` — Amazon Product Research Tool

Scan up to 10 product keywords simultaneously. Each keyword is graded using LaunchFast's A10–F1 scoring system and ranked by opportunity score with a clear **Go / Investigate / Pass** verdict.

**What it does:**
- Runs parallel product research across multiple niches
- Grades market quality, revenue potential, and competition
- Identifies the best-graded products per keyword
- Recommends next steps (suppliers, IP check, PPC)

```bash
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-product-research
```

**Usage:** `/launchfast-product-research silicone spatula bamboo cutting board`

**Requires:** `mcp__launchfast__research_products`

---

### `/launchfast-ppc-research` — Amazon Keyword Research Tool

Pull keyword intelligence from up to 15 competitor ASINs. Segments keywords by match type and opportunity tier, then generates a **ready-to-upload Amazon Bulk Operations CSV** — plug it straight into Seller Central.

**What it does:**
- Extracts and deduplicates keywords across ASINs
- Classifies into Tier 1 (Priority), Tier 2 (Growth), Tier 3 (Discovery)
- Assigns match types and bid estimates
- Outputs a campaign-ready TSV for Amazon Sponsored Products

```bash
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-ppc-research
```

**Usage:** `/launchfast-ppc-research B08N5WRWNW B07XYZABC1`

**Requires:** `mcp__launchfast__amazon_keyword_research`

---

### `/alibaba-supplier-outreach` — Alibaba Supplier Finder & Outreach Agent

Find Gold Suppliers on Alibaba, generate psychologically-optimized outreach messages, and send them automatically via Chrome automation. Tracks replies and manages negotiations through staged memory files.

**What it does:**
- Finds and scores Alibaba suppliers via LaunchFast data
- Generates credibility-anchored outreach messages
- Automates sending via Alibaba's contact form (no copy-paste)
- Tracks conversations and negotiation stages locally
- Supports 3 modes: Outreach, Check Replies, Negotiate

```bash
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill alibaba-supplier-outreach
```

**Usage:** `/alibaba-supplier-outreach silicone spatula`

**Requires:** `mcp__launchfast__supplier_research` + Chrome automation (`mcp__claude-in-chrome__*`) + logged-in Alibaba account

---

### `/launchfast-full-research-loop` — Complete FBA Opportunity Report

The full Amazon FBA product research pipeline in one command. Runs all four phases and compiles results into a **clean, downloadable HTML report** — ready to share with partners, VAs, or your team.

**What it does:**
- **Phase 1** — Product research across your keywords
- **Phase 2** — IP check (trademark + patent via USPTO)
- **Phase 3** — Alibaba supplier sourcing
- **Phase 4** — PPC keyword research from competitor ASINs
- **Phase 5** — Generates a branded HTML report with all findings

```bash
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-full-research-loop
```

**Usage:** `/launchfast-full-research-loop silicone spatula`

**Requires:** All four LaunchFast MCP tools (see setup below)

---

## Quick Start

### Step 1 — Sign up for LaunchFast

Create your account at **[launchfastlegacyx.com](https://launchfastlegacyx.com)** to get MCP access.

### Step 2 — Add the LaunchFast MCP to Claude Code

Run this in your terminal:

```bash
claude mcp add launchfast -- npx -y @launchfast/mcp
```

Or add manually to `~/.claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "launchfast": {
      "command": "npx",
      "args": ["-y", "@launchfast/mcp"]
    }
  }
}
```

### Step 3 — Install the skills

```bash
# Install all skills at once
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills

# Or install individually
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-product-research
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-ppc-research
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill alibaba-supplier-outreach
npx skills add https://github.com/BlockchainHB/launchfastmcp-skills --skill launchfast-full-research-loop
```

### Step 4 — Run your first research

```
/launchfast-full-research-loop silicone spatula
```

---

## Why LaunchFast vs Helium 10 / Jungle Scout

| | LaunchFast MCP | Helium 10 | Jungle Scout |
|---|---|---|---|
| Runs inside Claude Code | ✅ | ❌ | ❌ |
| Multi-keyword parallel research | ✅ | Manual | Manual |
| Auto-generates PPC bulk upload | ✅ | ❌ | ❌ |
| Alibaba supplier outreach automation | ✅ | ❌ | ❌ |
| IP + trademark check built-in | ✅ | Paid add-on | ❌ |
| Full HTML research report | ✅ | ❌ | ❌ |
| Agentic (works while you sleep) | ✅ | ❌ | ❌ |

---

## Requirements

- [Claude Code](https://claude.ai/code) installed
- [LaunchFast account](https://launchfastlegacyx.com) (required for MCP access)
- Node.js 18+
- For supplier outreach: Chrome with [Claude-in-Chrome extension](https://github.com/anthropics/claude-in-chrome), logged-in Alibaba.com account

---

## License

MIT
