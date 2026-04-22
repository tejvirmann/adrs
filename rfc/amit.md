# amit

**Deploy a shared AI assistant your whole organization actually uses.**

A similar internal tool at Target is gaining popularity because teams can publish context once and make it available everywhere. The same pattern can be applied in many places: engineering, hospitals, construction, finance, and anywhere else people work across shared systems and documents.

---

## 1. What It Is

amit is an AI assistant system built on top of [OpenCode](https://opencode.ai) — an open-source AI assistant with native support for [skills](https://opencode.ai/docs/skills/), [MCP servers](https://opencode.ai/docs/mcp-servers/), [rules](https://opencode.ai/docs/rules/), and [commands](https://opencode.ai/docs/commands/). amit adds the organizational registry layer that decides which of those are enabled for a given user, team, or session.

It has two parts:

```
┌─────────────────────────────────────────────────────────────┐
│  Part 1 — The Harness  (amit)                               │
│                                                             │
│  • Pulls context packages from your org's content store     │
│  • Injects skills, rules, commands, and tool configs        │
│  • Manages credentials — stored in OS keychain, never disk  │
│  • Runs OpenCode inside a sandbox (restricted file/network) │
│  • Syncs updates automatically — employees see nothing      │
└───────────────────────────────┬─────────────────────────────┘
                                │ configures & launches
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  Part 2 — OpenCode  (open source)                           │
│                                                             │
│  • The conversation engine and TUI / chat interface         │
│  • Model routing (Copilot, Azure OpenAI, Anthropic, etc.)   │
│  • Agent execution and tool calling via MCP                 │
│  • Session management and multi-agent delegation            │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
              Employee opens an app and asks a question.
              Everything above is invisible to them.
```

The harness is what makes amit an **organizational system** rather than a personal tool. It controls what context OpenCode sees, which tools it can call, and what credentials it has access to — and it keeps all of that synchronized across every person in the organization, automatically.

When a nurse opens amit, it already knows the hospital's protocols. When a project manager opens amit, it already knows the job site specs. That knowledge was put there by the people in your organization who own it — and it propagates to everyone on their next session.

### The critical layer: the registry

The most important part of amit is the registry, which amit configures and resolves before launching OpenCode. OpenCode already knows how to run agents, load instructions, and call MCP tools. amit is the layer that decides which contexts and MCP servers are turned on for a given organization, role, or task.

If amit were exposed as a CLI, the control surface could look like this:

```text
$ amit registry

Enabled contexts
[x] company-handbook
[x] nursing-protocols
[x] release-playbooks
[ ] construction-safety

Enabled MCP servers
[x] github
[x] jira
[x] internal-search
[ ] dbaas
```

That makes the system legible. You can see what context is active, see what tools the agent can use, and turn them on or off depending on the work in front of you.

The registry itself can be a simple manifest managed by amit:

```jsonc
// registry.jsonc
{
  "contexts": {
    "company-handbook": {
      "enabled": true,
      "instructions": ["contexts/company-handbook/AGENTS.md"]
    },
    "nursing-protocols": {
      "enabled": true,
      "instructions": ["contexts/nursing-protocols/AGENTS.md"],
      "skills": ["contexts/nursing-protocols/skills/shift-handoff"]
    },
    "release-playbooks": {
      "enabled": true,
      "skills": ["contexts/release-playbooks/skills/release-notes"],
      "commands": ["contexts/release-playbooks/commands/create-rc.md"]
    },
    "construction-safety": {
      "enabled": false,
      "instructions": ["contexts/construction-safety/AGENTS.md"]
    }
  },
  "mcp": {
    "github": {
      "enabled": true,
      "type": "remote",
      "url": "https://mcp.company.internal/github"
    },
    "jira": {
      "enabled": true,
      "type": "remote",
      "url": "https://mcp.company.internal/jira"
    },
    "internal-search": {
      "enabled": true,
      "type": "remote",
      "url": "https://mcp.company.internal/search"
    },
    "dbaas": {
      "enabled": false,
      "type": "local",
      "command": ["npx", "-y", "@company/dbaas-mcp"]
    }
  }
}
```

Then when someone types `amit`, the harness resolves that registry, renders the enabled context into OpenCode-compatible files, generates an `opencode.jsonc`, and launches the agent:

```text
$ amit
Launching OpenCode with:
- contexts: company-handbook, nursing-protocols, release-playbooks
- MCP: github, jira, internal-search
- disabled: construction-safety, dbaas
```

```jsonc
// opencode.jsonc (generated by amit at launch)
{
  "$schema": "https://opencode.ai/config.json",
  "default_agent": "build",
  "instructions": [
    ".amit/rendered/company-handbook/AGENTS.md",
    ".amit/rendered/nursing-protocols/AGENTS.md"
  ],
  "mcp": {
    "github": {
      "type": "remote",
      "url": "https://mcp.company.internal/github",
      "enabled": true
    },
    "jira": {
      "type": "remote",
      "url": "https://mcp.company.internal/jira",
      "enabled": true
    },
    "internal-search": {
      "type": "remote",
      "url": "https://mcp.company.internal/search",
      "enabled": true
    },
    "dbaas": {
      "type": "local",
      "command": ["npx", "-y", "@company/dbaas-mcp"],
      "enabled": false
    }
  },
  "tools": {
    "github": true,
    "jira": true,
    "internal-search": true,
    "dbaas": false
  }
}
```

`registry.jsonc` is amit's source of truth. `opencode.jsonc` is the generated runtime config. amit decides what to load, syncs the enabled skills and commands into OpenCode-compatible locations, and then OpenCode runs with that resolved result.

---

## 2. Where This Came From

A similar internal tool at Target — a large retail tech company with hundreds of engineering teams — helped prove the pattern. Engineers needed an AI assistant that understood Target's internal systems: GitHub, Jira, internal search, team-specific codebases and documentation.

The problem was that every team had different tools, different context, different ways of working. A single generic assistant wasn't useful. But asking every engineer to configure their own assistant wasn't realistic either.

So they built a harness. Teams started adding their own MCP servers — the DBaaS team added a database management MCP, teams brought in the GitHub MCP, Jira MCP, internal search MCPs. Each addition was registered once and became available to everyone who needed it. An agent that had access to GitHub *and* Jira *and* the internal codebase search could give answers that none of those tools could give individually — it could connect context across systems.

It got popular fast. Not because some central team built everything — but because it was easy for any team to plug in their piece, and everyone else immediately benefited.

The important point is not Target specifically. It's that the same architecture works anywhere people need shared context, shared tools, and a simple way to turn those capabilities on for the right users.

---

## 3. The Problem It Solves

AI assistants today give every employee a blank slate and expect them to figure it out.

To make an AI assistant genuinely useful, someone has to configure it with the right knowledge, connect it to the right systems, enforce the right guardrails, and keep all of that current as the organization changes. Most employees don't know how. Most IT teams don't have bandwidth to do it per-person. So it doesn't get done.

The result is that almost everyone in the organization uses AI the same way: as a slightly better search engine, with no connection to their actual work.

amit is the layer that solves this. It turns the work of configuring an AI assistant into something an organization does once, centrally — and distributes to everyone, automatically.

---

## 4. Real Examples

These aren't hypothetical. The same pattern applies well beyond a retail engineering org.

#### Hospital — Nursing Staff

A charge nurse is between rounds. She opens amit on the ward terminal.

She asks:
- *"What's the sepsis protocol for a patient presenting with these vitals?"*
- *"Summarize the last 24 hours of notes for bed 14."*
- *"Draft discharge instructions for a post-op hip replacement patient."*
- *"What's the current formulary substitution for this medication?"*

Her amit is connected to the hospital's clinical knowledge base, procedure documentation, and the formulary — within the hospital's HIPAA-compliant boundary. Clinical informatics published the protocols. She opens the app and asks.

---

#### Construction Company

A field superintendent arrives on-site. She opens amit on her tablet.

She asks:
- *"What are the load-bearing specs for the south facade per the current drawings?"*
- *"Generate today's daily safety report for crew B."*
- *"What permits are still outstanding for phase 3?"*
- *"There's a weather hold — what's the protocol for resuming crane operations after wind?"*

Her amit is connected to Procore (project data), the job's document library, the company safety playbook, and local permit databases. The safety team published the protocols. The ops team published the templates. She didn't configure any of it.

---

#### Investment Firm

A junior analyst is preparing for an IC meeting. She opens amit on her laptop.

She asks:
- *"Summarize last quarter's performance across our healthcare portfolio."*
- *"Draft a one-page IC memo for this deal based on the data room."*
- *"What's our standard covenant package for Series B?"*
- *"Pull the latest news on competitor activity in this space."*

Her amit is connected to the firm's portfolio database, the data room, market data feeds, and internal deal templates. The research team published the market context. Compliance published the guardrails. She opens the app and asks.

---

## 5. Where to Start — and Where It Goes

### Start here: connect your documents

The fastest way to get amit working is to give it access to your organization's existing knowledge. Two easy entry points:

- **A search MCP over an internal RAG database** — if your org has a knowledge base, wiki, or document store, a search connector lets the assistant query it as a tool. The assistant can find, summarize, and reason over your documents in conversation.
- **Web fetch or GitHub MCP** — for teams whose knowledge lives in repos, READMEs, or internal wikis, a GitHub MCP or a simple web fetch connector gets you there without any database infrastructure.

No agents to write. No rules to configure. Just connect the documents you already have.

What you get immediately: an assistant that can search, summarize, and reason across everything your organization has ever written. New employees stop spending their first weeks hunting through folders and asking senior staff basic questions. They ask amit. The institutional knowledge that usually takes months to absorb is accessible on day one.

**Onboarding alone pays for it.** In most knowledge-work organizations, bringing a new hire to full productivity takes 3–6 months. A big chunk of that is just finding information that already exists somewhere. An assistant trained on your own documentation compresses that materially — the new nurse finds the protocol, the new analyst finds the precedent, the new superintendent finds the spec. Immediately.

One MCP server. Your existing documents. Deployed once. Available to everyone.

---

### The measure of success

**How well this works is almost entirely determined by how easy you make it for your own people to contribute.**

If adding a skill requires filing a ticket and waiting two weeks, people won't do it. If it's as easy as writing a document and pushing it to a folder, people will do it constantly — and the assistant compounding improves with every contribution.

The assistant gets better over time as it gets access to more context: more tools connected, more skills published, more of your organization's knowledge made legible to it. That's not a feature you configure at setup. It's a property that emerges from making contribution frictionless.

The goal isn't a perfect assistant at launch. It's a system that keeps getting better because your people keep making it better.

---

### Next level: skills and custom MCP servers

Once the foundation is in place, teams start building on it — and this is where it compounds.

**Skills** are the mechanism for capturing how your organization does things. A skill is a structured guide — written in plain text — that activates when the conversation is relevant. A nurse writes a clinical summary template that works well. She publishes it as a skill. Every nurse in the organization gets it on their next session, automatically, with no action required on their part.

This is the key property: **when one person improves the system, everyone benefits.** In every other AI tool, knowledge stays per-person. When someone figures something out, they have to spread it manually. When they leave, it leaves with them. In amit, that knowledge lives in the system.

**Custom MCP servers** take this further. When a team builds a connector to a system specific to their domain — a project management API, an internal database, a specialized data feed — that connector becomes available to everyone in the organization who needs it. The investment in building it is made once.

The result is an assistant that gets measurably better over time as the people in your organization contribute to it. Not through a vendor update. Through your own accumulated expertise.
