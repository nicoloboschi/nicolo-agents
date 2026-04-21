# nicolo-agents

Self-learning OpenClaw agents backed by [Hindsight](https://github.com/vectorize-io/hindsight) memory.

Each agent has a **template** (bank config + pre-configured knowledge pages) and optional **content** (reference docs ingested at setup). The agent learns from conversations — preferences, corrections, and performance data feed back into its knowledge pages automatically.

## How it works

1. **Setup** — `hindsight-agent setup` creates the Hindsight bank, knowledge pages, ingests reference docs, and configures the OpenClaw agent
2. **Conversations** — a lightweight plugin retains every conversation into Hindsight (async, tool calls filtered out)
3. **Consolidation** — Hindsight extracts observations from conversations and refreshes knowledge pages via their `source_query`
4. **Next session** — the agent reads its updated pages at startup and applies what it learned

The agent decides *what* to track (creates pages). The system handles *capture* (plugin) and *synthesis* (consolidation).

## Setup

```bash
# Install the CLI (once)
cd ~/dev/hindsight-wt3/hindsight-agent && uv tool install -e .

# Set up an agent
hindsight-agent setup <agent-id> \
  --bank-id <bank-id> \
  --harness openclaw \
  --template <path>/template.json \
  --content <path>/content

# Restart gateway
openclaw gateway restart
```

## Agents

### marketing-seo-blog-posts

SEO marketing agent that combines industry best practices with learned editorial preferences and real performance data.

```bash
hindsight-agent setup marketing-seo \
  --bank-id demo-marketing-seo \
  --harness openclaw \
  --template ~/dev/nicolo-agents/marketing-seo-blog-posts/template.json \
  --content ~/dev/nicolo-agents/marketing-seo-blog-posts/content
```

**Pre-configured pages:**
- **SEO Best Practices** — synthesized from reference doc, adapts when your data contradicts industry advice
- **Content Performance** — accumulates analytics and identifies what formats/topics work
- **Editorial Preferences** — captures tone, length, formatting rules from your feedback

**Content:** `seo-specialist.md` — comprehensive SEO playbook (ingested at setup)

See [DEMO.md](../hindsight-wt3/hindsight-agent/DEMO.md) for the full demo walkthrough.

## Adding a new agent

```
my-agent/
  template.json   # Bank config + knowledge pages (see marketing-seo-blog-posts for format)
  content/         # Reference docs to ingest (.md, .txt, .html, .json)
```

The template defines: reflect/retain missions, disposition, pre-configured pages with source queries, and directives. Pages use `"mode": "delta"` so each refresh only processes new observations.
