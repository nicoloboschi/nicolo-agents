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

## Architecture

### The skill (`agent-knowledge`)

Installed into each agent's workspace at `skills/agent-knowledge/SKILL.md`. The agent reads it at session startup (patched into AGENTS.md step 5). It provides:

- `hindsight-agent pages list <agent-id>` — read all knowledge pages (mandatory at startup)
- `hindsight-agent pages create <agent-id> "<name>" "<source_query>"` — create a new page
- `hindsight-agent pages update/delete` — manage existing pages
- `hindsight-agent recall <agent-id> "<query>"` — search all memories for ad-hoc research
- `hindsight-agent documents <agent-id>` — list retained reference docs

The agent ID is baked into the skill at setup time. The CLI resolves agent ID → bank + API URL via `~/.hindsight-agent/config.json`. The agent never sees Hindsight internals.

### Mental Models (knowledge pages)

Each page has:
- **`source_query`** — a question the system re-asks after every consolidation to rebuild the page. This is the key abstraction: the agent writes it once, the system runs it forever.
- **`trigger.mode: "delta"`** — only processes new observations since last refresh
- **`trigger.refresh_after_consolidation: true`** — auto-updates after each consolidation cycle
- **`trigger.exclude_mental_models: true`** — pages synthesize from observations only, not from each other
- **`trigger.fact_types: ["observation"]`** — scoped to observations (not raw world/experience facts)

The source_query steers what the page becomes. The skill includes patterns:

```
# Best practices (merges reference docs with real results)
"What are the best practices for [topic], combining industry standards
with what has actually worked for us? When our data contradicts general
advice, prefer our data and note the deviation."

# Performance data
"What [topic] strategies have performed well or poorly based on analytics
and user feedback? Include specific numbers when available."

# User preferences
"What are the user's preferences for [topic], including explicit rules
they've stated and patterns observed from their feedback?"
```

### Data flow

```
[Reference docs]  ──→  retain (setup --content)  ──→  [Bank]
[Conversations]   ──→  retain (openclaw plugin)  ��─→  [Bank]
                                                        ↓
                                                  consolidation
                                                        ↓
                                                  observations
                                                        ↓
                                              page refresh (delta)
                                                        ↓
                                              source_query → reflect
                                                        ↓
                                              updated page content
                                                        ↓
                                    agent reads at next session startup
```

## Adding a new agent

```
my-agent/
  template.json   # Bank config + knowledge pages (see marketing-seo-blog-posts for format)
  content/         # Reference docs to ingest (.md, .txt, .html, .json)
```

The template defines: reflect/retain missions, disposition, pre-configured pages with source queries, and directives. Pages use `"mode": "delta"` so each refresh only processes new observations.

## Getting started

### Requirements

- [Hindsight](https://github.com/vectorize-io/hindsight) API running locally (default: `http://localhost:8888`)
- [OpenClaw](https://docs.openclaw.ai) installed with gateway running
- Python 3.11+ with [uv](https://docs.astral.sh/uv/)
- Node.js 22+

### 1. Clone and checkout the hindsight branch

```bash
git clone https://github.com/vectorize-io/hindsight.git ~/dev/hindsight
cd ~/dev/hindsight
git checkout feat/agent-procedural-memory
```

### 2. Start Hindsight (API + Control Plane)

```bash
cd ~/dev/hindsight
cp .env.example .env  # edit with your LLM API key
./scripts/dev/start.sh
```

### 4. Install the hindsight-agent CLI

```bash
cd ~/dev/hindsight/hindsight-agent
uv tool install -e .
```

### 5. Set up the agent

```bash
hindsight-agent setup marketing-seo \
  --bank-id demo-marketing-seo \
  --harness openclaw \
  --template ~/dev/nicolo-agents/marketing-seo-blog-posts/template.json \
  --content ~/dev/nicolo-agents/marketing-seo-blog-posts/content
```

### 6. Restart the OpenClaw gateway

```bash
openclaw gateway restart
```

### 7. Chat with the agent

```bash
openclaw tui --session agent:marketing-seo:main:random1
```

The agent reads its knowledge pages at startup and applies them. Feed it preferences, analytics data, and watch the pages evolve after each consolidation cycle.

### Verify pages

```bash
hindsight-agent pages list marketing-seo
```
