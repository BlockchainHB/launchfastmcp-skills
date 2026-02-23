# LaunchFast MCP Skills

Reusable Claude Code skills for Amazon FBA sellers, powered by the [LaunchFast MCP](https://launchfast.com).

Install any skill with:

```bash
npx skills add https://github.com/launchfast/launchfast-mcp-skills --skill [skill-name]
```

---

## Skills

### `launchfast-product-research`
Multi-keyword Amazon product opportunity scanner. Researches up to 10 keywords in parallel, grades each using LaunchFast's A10–F1 scoring system, and delivers ranked Go/Investigate/Pass verdicts.

```bash
npx skills add https://github.com/launchfast/launchfast-mcp-skills --skill launchfast-product-research
```

**Invoke:** `/launchfast-product-research silicone spatula bamboo cutting board`

**Requires:** `mcp__launchfast__research_products`

---

### `launchfast-ppc-research`
PPC keyword research for Amazon Sponsored Products. Analyzes up to 15 competitor ASINs, segments keywords by match type and opportunity tier, and generates a ready-to-upload Amazon Bulk Operations TSV file.

```bash
npx skills add https://github.com/launchfast/launchfast-mcp-skills --skill launchfast-ppc-research
```

**Invoke:** `/launchfast-ppc-research B08N5WRWNW B07XYZABC1`

**Requires:** `mcp__launchfast__amazon_keyword_research`

---

### `alibaba-supplier-outreach`
Find Alibaba suppliers and automate outreach via Chrome. Generates psychologically-optimized contact messages, sends them via Alibaba's contact form, checks replies, and manages negotiations — all tracked in local memory files.

```bash
npx skills add https://github.com/launchfast/launchfast-mcp-skills --skill alibaba-supplier-outreach
```

**Invoke:** `/alibaba-supplier-outreach silicone spatula`

**Requires:** `mcp__launchfast__supplier_research` + Chrome automation (`mcp__claude-in-chrome__*`) + logged-in Alibaba account

---

### `launchfast-full-research-loop`
Complete FBA opportunity pipeline in one command. Runs product research → IP check → supplier sourcing → PPC keywords → compiles everything into a clean downloadable HTML report.

```bash
npx skills add https://github.com/launchfast/launchfast-mcp-skills --skill launchfast-full-research-loop
```

**Invoke:** `/launchfast-full-research-loop silicone spatula`

**Requires:** All four LaunchFast MCP tools:
- `mcp__launchfast__research_products`
- `mcp__launchfast__ip_check_manage`
- `mcp__launchfast__supplier_research`
- `mcp__launchfast__amazon_keyword_research`

---

## Installing the LaunchFast MCP

Add to your Claude Code MCP config:

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

---

## License

MIT
