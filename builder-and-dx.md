# Agent Script — Builder & DX

The tooling around Agent Script: where to write it, how to move it between
your DX project and your org, and how to integrate with source control.

## Contents

- [Tooling Overview](#tooling-overview)
- [Agentforce Builder](#agentforce-builder)
- [Agentforce DX](#agentforce-dx)
- [Agentforce Vibes](#agentforce-vibes)
- [Authoring Bundle vs. Agent Metadata](#authoring-bundle-vs-agent-metadata)
- [Recommended Workflow](#recommended-workflow)
- [Source Control](#source-control)
- [Testing](#testing)

## Tooling Overview

| Tool | Surface | Use for |
|---|---|---|
| **Agentforce Builder** | Web UI in Agentforce Studio | Low-code authoring, live preview, conversational AI Assist. |
| **Canvas view** | Builder, low-code | Visual block layout. Quick edits via `/` (expressions) and `@` (resources). |
| **Script view** | Builder, pro-code | Direct Agent Script editing in the org with syntax highlighting and validation. |
| **Agentforce DX CLI** | Salesforce CLI plugin | Generate, preview, deploy, and retrieve agent metadata. |
| **VS Code Extension** | Local IDE | Full Agent Script language support (highlighting, autocompletion, validation). |
| **Agentforce Vibes** | AI pair-programming IDE | Generate / modify Agent Script with an AI assistant that understands your DX project. |
| **DevOps Center** | Salesforce DevOps tooling | Promote agent metadata across environments. |
| **Testing Center** | Org UI | Run test conversations and assertions against your agent. |

## Agentforce Builder

Three authoring modes inside the Builder, all backed by the same metadata:

### AI-assisted ("chat with Agentforce")

Describe what the agent should do in natural language. Agentforce generates
topics, actions, instructions, and expressions. Best for first drafts; expect
to refine in Canvas or Script view afterward.

### Canvas view

Visual layout where blocks summarize into expandable cards. Type `/` to insert
common expression patterns (e.g., if/else). Type `@` to add references to
topics, actions, and variables.

Use Canvas for: quick edits, navigating large agents, showing the structure
to non-technical stakeholders.

### Script view

Direct text editing of the underlying Agent Script. Includes syntax
highlighting, autocompletion, and validation.

Use Script view for: power-user editing, complex logic, copy-paste of patterns
from references, side-by-side diffs.

The two views are bidirectional — edits in Script view show up in Canvas after
save, and vice versa.

## Agentforce DX

Salesforce CLI plugin and VS Code Extension for pro-code agent development.
Treats agents as metadata, like Apex or Flows.

### Salesforce CLI commands

The Agentforce DX commands extend `sf`. Common patterns (verify exact
flag names against your CLI version — these change weekly):

```bash
# Generate a new agent's authoring bundle in your DX project
sf agent generate authoring-bundle --name MyAgent

# Preview an agent (simulated mode — no org call)
sf agent preview --name MyAgent --simulated

# Preview against an org (live mode)
sf agent preview --name MyAgent

# Publish authoring bundle to org
sf agent publish --name MyAgent --target-org my-scratch

# Retrieve agent metadata from org back to DX project
sf project retrieve start --metadata Agent:MyAgent

# Deploy agent metadata from DX project to org
sf project deploy start --metadata Agent:MyAgent
```

The Salesforce CLI ships weekly. Authoritative source for current commands:
the [Salesforce CLI release notes](https://github.com/forcedotcom/cli/blob/main/releasenotes/README.md).

### VS Code Extension

The Agentforce DX VS Code Extension provides:

- Syntax highlighting for `.agent` files.
- Autocompletion for blocks, properties, references, and utilities.
- Inline validation against the language spec.
- Standard editor features: outline view, go-to-definition for references.

Install via the VS Code Marketplace (search "Agentforce DX").

## Agentforce Vibes

An AI pair-programming environment for Salesforce projects, with deep
awareness of the project context — including agent metadata.

Useful for:

- Generating Agent Script topics from a description.
- Refactoring existing scripts (e.g., extracting a sub-flow into its own
  topic, adding `available_when` filtering across many tools).
- Implementing the Apex or Flow target of a new action while keeping the
  Agent Script in sync.
- Writing tests for an agent in the same context.

Vibes Skills (notable for this skill's domain): the *Agentforce Development
Skill* in the Agentforce Vibes Library is the canonical Vibes-side complement
to Agent Script editing.

## Authoring Bundle vs. Agent Metadata

Two related artifacts:

- **Authoring bundle** — the source-of-truth for the agent during development.
  Contains the Agent Script blueprint plus supporting metadata. Lives in your
  DX project. Generated with `sf agent generate authoring-bundle`.
- **Agent metadata** — the deployed form in the org. Multiple metadata types
  (Bot, BotVersion, GenAiPlanner, GenAiFunction, etc.) make up a single
  agent.

The CLI commands convert between the two. For day-to-day editing, work on the
authoring bundle. For source control, commit the authoring bundle plus the
metadata directories produced by retrieve operations.

For the full metadata-type list, see Salesforce's *Agent Metadata: A Shallow
Dive*: <https://developer.salesforce.com/docs/ai/agentforce/guide/agent-dx-metadata.html>

## Recommended Workflow

A concrete pro-code/low-code workflow that keeps your DX project as source
of truth:

1. **Bootstrap from local.** Generate the authoring bundle in your DX project
   (`sf agent generate authoring-bundle`). Hand-write or AI-assist the Agent
   Script in VS Code.
2. **Publish to a scratch org or sandbox** (`sf agent publish`). Open in the
   Agentforce Builder UI for live preview against real org data.
3. **Iterate in either tool.** Use Canvas / Script view for in-org editing;
   use VS Code for refactoring, mass edits, or when you need diff view.
4. **Retrieve before committing.** After in-org changes, `sf project retrieve`
   so the DX project reflects the org state. Commit to VCS.
5. **Add Apex / Flow / Prompt Template targets locally.** Implement actions in
   VS Code (with Vibes if helpful). Reference them from the Agent Script.
   Deploy back with `sf project deploy`.
6. **Test from the CLI** (`sf agent test run`) for CI/CD; use the in-org
   Testing Center for richer assertions and historical results.
7. **Promote with DevOps Center** to higher-tier environments. Release notes
   and approvals attach to the metadata bundle.

The principle: VCS is the source of truth. Any change made in-org must come
back to VCS before the next promotion.

## Source Control

Agent metadata is verbose. A typical agent produces a dozen or more metadata
files across multiple types. Strategies:

- **Commit the authoring bundle plus retrieved metadata.** The bundle is
  human-readable; the metadata is what actually deploys. Both belong in VCS.
- **Diff at the script level.** Most meaningful changes show up in the
  Agent Script file. Enabling Git's textual diff for `.agent` makes review
  practical.
- **Avoid checking in `developer_name` collisions.** If two developers create
  agents with the same `developer_name`, the first deploy wins. Coordinate
  naming in the team or use prefixes.
- **Treat agent versions as you'd treat database migrations.** Each saved
  version produces new metadata. Avoid version churn in shared branches —
  version on release branches.

## Testing

Two layers:

- **Unit-style testing of actions.** Apex unit tests for Apex actions; Flow
  Test Coverage for autolaunched flows; Prompt Template testing for prompt
  actions. Cover the action contract.
- **Conversation testing of the agent.** Use Testing Center (in-org) or
  `sf agent test run` (CLI). Define test conversations with expected
  utterances and assertions about variable state, action calls, or final
  responses.

Conversation tests are non-deterministic — the LLM's exact wording varies
turn to turn. Assert on structural properties (was the action called, did the
variable end in this state, did the agent transition here) rather than exact
strings.

For details on the testing API and CLI: <https://developer.salesforce.com/docs/ai/agentforce/guide/agent-dx-test.html> and <https://developer.salesforce.com/docs/ai/agentforce/guide/testing-api.html>.
