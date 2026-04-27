# Salesforce Agent Script — A Claude Skill

A focused, source-grounded knowledge base for working with Salesforce
Agentforce **Agent Script**, packaged as a [Claude Skill](https://docs.claude.com/en/api/agent-sdk/skills).

When loaded into Claude, this skill gives the assistant accurate, current
knowledge of Agent Script syntax, idiomatic patterns, common antipatterns,
and the surrounding tooling (Agentforce Builder, Agentforce DX, Agentforce
Vibes) — instead of relying on whatever happened to be in its training data
about a young, fast-moving DSL.

---

## Why this exists

Agent Script is Salesforce's declarative language for building hybrid-reasoning
AI agents on the Agentforce 360 Platform. It went into Public Beta in
November 2025 and has been moving fast since: what was called a *topic* until
April 2026 is now a *subagent*, the recipe library grows weekly, and the
tooling (Agentforce DX CLI, Agentforce Vibes) ships on a continuous cadence.

Out of the box, large language models — Claude included — have unreliable
knowledge of Agent Script. Common failure modes I've seen in the field:

- Mixing `@topic.X` (legacy) and `@subagent.X` (current) without flagging the
  rename.
- Inventing a `then:` action-chaining property without checking the spec.
- Treating the slot-fill `...` token as a variable default when it's only
  valid in action inputs.
- Recommending the JSON-map `with { key: value }` syntax instead of the
  canonical `with key = value` form.

Each of those costs hours of debugging time inside the Builder. This skill
closes the gap by distilling the official Salesforce developer guide, the
Salesforce Developers Blog *Agent Script Decoded* series, and the
open-source `salesforce/agentscript` repository into a structured reference
that Claude consults whenever a conversation turns to Agentforce or Agent
Script.

---

## What's inside

```
skill/
├── SKILL.md                            # entry point — loads when skill triggers
└── references/
    ├── syntax-reference.md             # complete language reference
    ├── patterns.md                     # idiomatic patterns
    ├── antipatterns.md                 # common mistakes and how to avoid them
    └── builder-and-dx.md               # Agentforce Builder, DX CLI, VS Code, Vibes
```

The skill follows the Anthropic *progressive disclosure* convention: the
SKILL.md is small (~300 lines) and always loads when the skill triggers;
reference files (~200–550 lines each) are pulled into context only on demand.

### Scope

**In scope.** Agent Script language (blocks, expressions, hooks, variables,
transitions, action chaining), Hybrid Reasoning concepts at the script level,
Agentforce Builder authoring (Canvas / Script view, AI Assist), Agentforce DX
(CLI, VS Code extension), Agentforce Vibes integration, the `topic` →
`subagent` migration.

**Deliberately out of scope** (different skills, different concerns): Atlas
Reasoning Engine internals, Data Cloud grounding, multi-org deployment
strategy, Apex / Flow / Prompt-Template authoring, model selection.

---

## Installation

### Claude.ai (web, desktop, mobile)

1. Download the latest `salesforce-agent-script.skill` from the
   [Releases page](../../releases).
2. In Claude.ai, open **Settings → Capabilities → Skills**.
3. Click **Upload skill** and select the `.skill` file.

The skill activates automatically when your conversation mentions Agentforce,
Agent Script, subagents, transitions, `@utils.*`, `@variables.*`, or related
concepts. You don't need to invoke it explicitly.

### Claude Code, Cowork, and the API

Skills can be loaded by reference in agent SDK contexts. See the [Anthropic
skills documentation](https://docs.claude.com/en/api/agent-sdk/skills) for
the current loading pattern in your environment.

---

## How it works

Claude Skills use a three-tier loading model:

| Tier | Always in context? | Contains |
|---|---|---|
| 1. Metadata (name + description) | Yes | When the skill should trigger |
| 2. SKILL.md body | When triggered | Mental model, terminology, decision rubrics, quick reference, pointers |
| 3. Reference files | On demand | Full syntax, patterns, antipatterns, tooling |

The metadata description is the most important piece — it's what Claude
matches against the user's intent to decide whether to consult the skill.
This skill's description is engineered to trigger on Agentforce, Agent
Script, and related vocabulary even when the user doesn't say "Agent Script"
explicitly (e.g., "help me debug my subagent transition" or "how do I do
slot filling").

---

## Source material

The skill is a distillation of publicly available Salesforce documentation
and source code. Primary sources:

- [Agent Script developer guide](https://developer.salesforce.com/docs/ai/agentforce/guide/agent-script.html)
- [Agent Script Reference index](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-reference.html)
- [Agent Script Patterns](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-patterns.html)
- [Agent Script Recipes (sample apps)](https://developer.salesforce.com/sample-apps/agent-script-recipes)
- [`salesforce/agentscript` — open-source language tooling](https://github.com/salesforce/agentscript)
- Salesforce Developers Blog — *Agent Script Decoded* series

When the skill states a fact that I couldn't confirm against a canonical
source, the content is marked `[Unverified]`. This convention signals to
Claude that the claim should be re-checked against live documentation before
being relied upon — and signals to human reviewers where the gaps are.

---

## Currency

Agent Script is moving fast. The content in this repository reflects the
state of Salesforce documentation on the date of each release (see
[`CHANGELOG.md`](CHANGELOG.md)). Between releases, expect drift on:

- New utilities in `@utils.*`
- New action target schemes (`flow://`, `apex://`, `promptTemplate://` are
  stable; others may appear)
- `sf agent ...` CLI flags (the Salesforce CLI ships weekly)
- Beta-to-GA promotion of individual features

The skill itself instructs Claude to verify against the live developer
guide whenever syntax looks unfamiliar or when the Builder rejects something
the skill claims should work. Pull requests with updated syntax, new
recipes, and freshly observed antipatterns are welcome.

---

## Roadmap

Planned additions, in rough priority order:

- [ ] `references/recipes.md` — annotated walk-throughs of the most useful
      entries from the `agent-script-recipes` repository.
- [ ] `references/migration.md` — `topic` → `subagent` migration playbook
      for existing 2025-era Agentforce implementations.
- [ ] `references/testing.md` — deeper coverage of `sf agent test run`,
      Testing Center assertions, and CI/CD integration.
- [ ] `references/multi-org.md` — patterns for sharing agents across
      Salesforce orgs (split out from the multi-org DevOps body of work).

---

## Contributing

Contributions are welcome — especially:

- **Syntax corrections.** If a fact in the skill doesn't match current
  Salesforce documentation, open an issue or pull request with the source
  link.
- **New patterns** that recur in real Agentforce work and aren't yet
  covered.
- **Antipatterns** observed in production. Anonymize as needed.
- **Translations** of the SKILL.md (English-only for triggering reliability,
  but reference files in other languages are an option).

For larger changes, please open an issue first to discuss scope before
investing the time.

### Local development

```bash
# Clone
git clone https://github.com/<your-username>/salesforce-agent-script-skill.git
cd salesforce-agent-script-skill

# Edit the markdown files
$EDITOR skill/SKILL.md skill/references/*.md

# Validate the skill structure
python3 -m scripts.quick_validate skill/

# Package into a .skill file
python3 -m scripts.package_skill skill/
```

`quick_validate` and `package_skill` are part of Anthropic's
[skill-creator](https://github.com/anthropics) tooling. Use whichever
local copy you have available.

---

## License

Released under the **MIT License**. See [`LICENSE`](LICENSE).

You are free to use, modify, redistribute, and adapt this skill for any
purpose — commercial or otherwise — provided the copyright notice is
preserved.

---

## Disclaimer

This is an **unofficial, community-maintained** project. It is **not
affiliated with, endorsed by, or sponsored by Salesforce, Inc. or
Anthropic, PBC.**

- *Salesforce*, *Agentforce*, *Atlas Reasoning Engine*, and related marks
  are trademarks of Salesforce, Inc.
- *Claude* and the Anthropic skill format are products of Anthropic, PBC.

The skill content is a distillation of publicly available Salesforce
documentation, provided for educational and productivity purposes. Always
verify critical syntax against the live Salesforce developer guide before
shipping production agents.

---

## Author

**Christopher Ramm**
DCX CTO Germany, Capgemini · Salesforce Certified Technical Architect ·
Salesforce MVP (Class of 2025)

[cramm.dev](https://cramm.dev) · [GitHub @rammc](https://github.com/rammc) ·
[hello@cramm.dev](mailto:hello@cramm.dev)
