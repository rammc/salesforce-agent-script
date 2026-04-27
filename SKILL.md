---
name: salesforce-agent-script
description: "Implement, review, or improve Salesforce Agentforce agents written in Agent Script — the declarative DSL for hybrid reasoning in the Agentforce Builder. Use when working with Agent Script syntax (blocks, expressions, variables, hooks, transitions, action chaining); when designing topics or subagents and the start_agent block; when exposing actions as tools to the LLM via reasoning actions; when balancing deterministic logic instructions against LLM prompt instructions; when migrating between Topics and Subagents terminology (April 2026 rename); when building agents in Agentforce Builder Canvas or Script view; when using Agentforce DX CLI or the Agentforce DX VS Code extension; or when writing agents with Agentforce Vibes. Trigger this skill whenever the user mentions Agentforce, Agent Script, agent topics, subagents, Atlas Reasoning Engine in a scripting context, agent transitions, reasoning actions, @utils, @variables, or @actions — even if they don't explicitly name the skill."
---

# Salesforce Agent Script

Build, review, and refactor Agentforce agents written in Agent Script. Agent
Script is Salesforce's declarative DSL for hybrid reasoning agents on the
Agentforce 360 Platform. It compiles to portable JSON metadata consumed by the
Atlas Reasoning Engine.

This skill covers the language itself and the surrounding pro-code tooling
(Agentforce Builder Script view, Agentforce DX CLI, VS Code extension,
Agentforce Vibes). It does **not** cover Atlas Reasoning Engine internals,
Data Cloud grounding configuration, multi-org deployment strategy, or Apex /
Flow / Prompt Template authoring — those are adjacent disciplines.

## Contents

- [When to use this skill](#when-to-use-this-skill)
- [Mental Model](#mental-model)
- [Terminology: Topics vs. Subagents](#terminology-topics-vs-subagents)
- [Anatomy of an Agent](#anatomy-of-an-agent)
- [The Hybrid Reasoning Decision](#the-hybrid-reasoning-decision)
- [Quick Reference](#quick-reference)
- [Authoring Workflow](#authoring-workflow)
- [Review Checklist](#review-checklist)
- [Reference Files](#reference-files)
- [Verification Pointers](#verification-pointers)

## When to use this skill

Trigger this skill when the task involves:

- Writing, reading, or refactoring Agent Script source (`.agent` files or
  scripts shown in the Builder).
- Designing the structure of a new agent: `start_agent` routing, topic
  decomposition, action surfaces, variable design.
- Debugging unexpected agent behavior that comes from prompt vs. logic
  mis-balance, transitions, or variable scope.
- Choosing between deterministic logic (`->`) and prompt instructions (`|`).
- Translating natural-language requirements ("if total > 100, free shipping")
  into Agent Script.
- Migrating from a chat/Canvas-built agent to Script view, or from natural
  language–only agents to hybrid reasoning.
- Integrating with the surrounding tooling (Agentforce DX, Vibes, VS Code).

Skip this skill for: Apex action implementation, Flow design, Data Cloud
configuration, model selection / Atlas tuning, multi-org metadata deployment.

## Mental Model

Agent Script combines two execution modes in one file:

- **Logic instructions** prefixed with `->` run deterministically every time
  the subagent is parsed. Use them for business rules, action calls, variable
  mutation, transitions, conditional branches.
- **Prompt instructions** prefixed with `|` are concatenated into a string
  that is sent to the LLM as a prompt. The LLM then decides what to say or
  which exposed tool to call.

The execution flow per customer turn:

1. Enter at `start_agent` (or current subagent after a transition).
2. **Parse phase.** Resolve all reasoning instructions top-to-bottom: run
   logic, mutate variables, execute deterministic actions, concatenate prompt
   strings (conditionally where `if` gates apply).
3. **Reasoning phase.** Send the resolved prompt + the list of `reasoning.actions`
   tools to the LLM. The LLM may answer directly, or may call one or more tools.
4. **Reasoning loop.** After each tool execution, the system **loops back to
   step 2** to re-resolve and re-send. The cycle repeats until the LLM
   responds without calling a tool.
5. **`after_reasoning` runs** (if defined) once the loop exits.
6. The agent waits for the next customer utterance, then re-enters at
   `start_agent`.

The central insight: **reasoning is a loop, not a single pass**. A subagent
may resolve its prompt many times in a single customer turn as the LLM calls
tools. Variables mutated between iterations are visible to the next iteration.
Design accordingly — and when debugging, inspect the trace, not just the
final prompt.

## Terminology: Topics vs. Subagents

Beginning April 2026, agent **topics** are renamed **subagents**. This is more
than a UI change — `topic` is **deprecated** in the official Reference index,
and `subagent` is the keyword to prefer for new code.

In practice, both terms appear in the field:

- **`subagent <name>:`** — preferred block keyword for new code.
- **`topic <name>:`** — legacy keyword, still parses, still appears in older
  recipes, blog posts, and existing customer agents.
- **`@subagent.<name>`** — preferred reference syntax.
- **`@topic.<name>`** — legacy reference syntax, still works.

Functionally identical. When writing new agents, prefer `subagent` /
`@subagent.<name>` throughout. When reviewing or editing an existing agent,
match the form already in use; do not silently flip terminology mid-file
unless the user asks for migration.


## Anatomy of an Agent

Every agent is a single Agent Script file containing these block types
(roughly in this order):

| Block | Required | Purpose |
|---|---|---|
| `config` | yes | Agent identity: `developer_name`, `agent_label`, `description`, `agent_type`, `default_agent_user`. |
| `system` | yes | Global instructions and message templates. Must define `welcome` and `error`. |
| `variables` | optional | Global state shared across subagents: regular, linked, and (predefined) system variables. |
| `language` | optional | Supported languages. |
| `connection` | optional | External integrations, e.g. Enhanced Chat for `@utils.escalate`. |
| `start_agent <name>` | yes | The entry router. Runs at the start of every customer turn. Handles classification and routing. |
| `subagent <name>` | 1+ | A specialized capability. Contains `description`, `reasoning.instructions`, `reasoning.actions`, `actions`. (Legacy keyword: `topic <name>`.) |

For full block reference, see `references/syntax-reference.md`.

### File layout

In a Salesforce DX project, an Agent Script lives at:

```
force-app/main/aiAuthoringBundles/<AgentName>/<AgentName>.agent
```

The directory is called the **authoring bundle** — the source-of-truth during
development, alongside any supporting metadata. Agentforce DX commands
(`sf agent ...`) operate on authoring bundles.

## The Hybrid Reasoning Decision

The single most important design decision in Agent Script is **what runs
deterministically vs. what the LLM decides**. Use this rubric:

| Situation | Use |
|---|---|
| Business rule with a clear true/false outcome (eligibility, threshold, status check) | Logic (`->` with `if`/`else`) |
| Sequence of actions that must run in a fixed order | Logic (`run @actions.x` chained) |
| Setting state from an action's output | Logic (`set @variables.x = @outputs.y`) |
| Generating a friendly, contextual customer message | Prompt (`|`) |
| Choosing which of N optional capabilities to invoke based on intent | Tool surface (`reasoning.actions`) — LLM decides |
| Filling an action input from free-form user input | `...` token in the action input (LLM-extracted) |
| Capturing free-form input directly into a variable | `@utils.setVariables` tool with variable's `description` |
| Hard handoff to another subagent the moment a condition is met | `transition to @subagent.x` from logic |
| Soft handoff offered to the LLM as one option among several | `@utils.transition to @subagent.x` exposed in `reasoning.actions` |

**Bias toward determinism for business rules; bias toward LLM for phrasing,
classification, and slot filling.** Overloaded prompts with embedded business
rules are the most common antipattern (see `references/antipatterns.md`).

## Quick Reference

Resource references — always use the `@` prefix:

```
@actions.<name>              # action defined in the current subagent
@variables.<name>            # global variable
@outputs.<name>              # action output (within run/set context)
@subagent.<name>             # another subagent (preferred)
@topic.<name>                # legacy alias for @subagent
@utils.<function>            # built-in utility (transition, setVariables, escalate)
@system_variables.user_input # latest customer utterance (read-only)
```

Variable interpolation in prompt text — use the **bang-brace** form:

```
| Hi {!@variables.user_name}, your order {!@variables.order_id} is on the way.
```

Operators (full list in `syntax-reference.md`):

```
==  !=  <  <=  >  >=        # comparison
is None     is not None     # null check
and  or  not                # logical
+  -                        # arithmetic (numbers only)
```

`if` / `else` only — there is **no `else if`**. Nest if you need it.

Indentation is whitespace-sensitive. **Use spaces only — Salesforce
recommends 3 spaces per level.** Comments start with `#`.

For deeper coverage see:
- `references/syntax-reference.md` for the complete language reference.
- `references/patterns.md` for idiomatic patterns.
- `references/antipatterns.md` for common pitfalls.

## Authoring Workflow

When writing or refactoring an agent, follow this sequence:

### Step 1 — Map the conversation surface

Before touching script, list:

1. The customer intents the agent must handle.
2. For each intent: what actions/data lookups are needed, and which business
   rules gate which behavior.
3. Which intents are independent (separate subagents) vs. variations of one
   capability (one subagent).

This becomes your subagent decomposition. Aim for 3–7 subagents for typical
agents — fewer means an overloaded subagent; more usually means
over-decomposition.

### Step 2 — Design the start_agent

`start_agent` runs at the **start of every customer turn**, not just the first.
Use it for:

- Initializing variables that must always be set (e.g., session timestamps,
  channel context).
- Classifying intent and exposing transitions to subagents as tools.
- Filtering — refusing to handle out-of-scope requests with a guarded prompt.

Keep `start_agent` short. Long classification prompts hurt routing accuracy.

### Step 3 — Write each subagent

For each subagent:

1. **Description first.** The description tells the LLM when to pick this
   subagent. Be specific and use the same vocabulary the customer would use.
2. **Variables you depend on.** If the subagent assumes the customer is verified,
   transition unverified users elsewhere or guard with `if`.
3. **Reasoning instructions.** Start with the minimum natural language needed
   for the LLM to do its job. Layer in logic (`->`) only where determinism
   matters.
4. **Actions and tools.** Define actions in `actions:`. Expose them to the LLM
   only via `reasoning.actions:` — and only when LLM choice is genuinely
   useful. Otherwise, call them deterministically with `run @actions.<name>`.

### Step 4 — Wire transitions

Map every "and then…" path between subagents:

- **Hard handoffs** (always go to subagent X next): `transition to
  @subagent.x` from the logic block.
- **Optional handoffs** (LLM decides): expose a `reasoning.actions` tool
  wrapping `@utils.transition to @subagent.x`.
- **Delegated calls with return** (rare): use a direct `@topic.<name>`
  reference in `reasoning.actions`. Flow returns to the caller after the
  delegated subagent completes — this differs from `transition to`, which is
  one-way.

### Step 5 — Test the prompt that actually reaches the LLM

In Agentforce Builder, preview the conversation and inspect the resolved
prompt for each subagent. Most "agent does the wrong thing" bugs are visible in
the resolved prompt: variables not interpolating, instructions in the wrong
order, prompt instructions that contradict logic-set variables.

## Review Checklist

When reviewing an Agent Script file, check in this order:

1. **`config.developer_name`** — unique in the org, follows naming rules
   (letters, alphanumerics + underscore, no trailing underscore, no `__`).
2. **`system.welcome` and `system.error`** are present and on-brand.
3. **Subagent descriptions** are specific, distinct, and use customer
   language. Overlapping descriptions cause routing flakiness.
4. **`start_agent` is lean** — no business logic, just routing and required
   variable initialization.
5. **Variable mutability** — `mutable` only where truly needed. Immutable
   variables prevent whole classes of state bugs.
6. **Logic vs. prompt balance** — business rules in `->`, customer-facing
   phrasing in `|`. Flag any prompt text containing literal numeric thresholds
   or hard-coded business rules (a smell for misplaced logic).
7. **Action exposure** — every action in `reasoning.actions` should benefit
   from LLM choice. If it always runs, move it to logic with `run @actions`.
8. **Transitions** — one-way intent is clear. No accidental loops between
   subagents. No code "after" a transition that expects to run.
9. **Conflicting instructions** — global `system.instructions` and
   subagent-level overrides are not contradictory.
10. **Indentation** — consistent within the file, not mixed spaces/tabs.

For each finding, distinguish *correctness* (will misbehave), *robustness*
(might misbehave under edge cases), and *style* (works fine, but violates
Salesforce's documented patterns).

## Reference Files

Load on demand:

- **`references/syntax-reference.md`** — Complete language reference: every
  block type, every property, all operators, variable types, action targets,
  utility functions. Read when writing new script or verifying syntax.
- **`references/patterns.md`** — Idiomatic patterns: identity verification,
  slot filling, action chaining, conditional prompting, available-when
  filtering, system overrides, fetch-data-before-reasoning, required workflows.
  Read when designing a new subagent or stuck on "how would I express X".
- **`references/antipatterns.md`** — Common mistakes and why they fail. Read
  during reviews and when debugging unexpected agent behavior.
- **`references/builder-and-dx.md`** — Surrounding tooling: Agentforce
  Builder Canvas vs. Script view, Agentforce DX CLI commands, VS Code
  extension, Agentforce Vibes, authoring bundles, source-control workflow.
  Read when the question is about *how* to author or deploy, not *what* to
  write.

## Verification Pointers

Agent Script is young (Public Beta November 2025, GA still rolling out across
features). When in doubt, verify against:

- Canonical reference: <https://developer.salesforce.com/docs/ai/agentforce/guide/agent-script.html>
- Agent Script Recipes (sample apps): <https://developer.salesforce.com/sample-apps/agent-script-recipes>
- Open-source language tooling: <https://github.com/salesforce/agentscript>
- Salesforce CLI release notes (weekly): <https://github.com/forcedotcom/cli/blob/main/releasenotes/README.md>

If a feature is unfamiliar or the syntax shown here looks outdated, prefer
fetching the live documentation over relying on this skill's frozen content.
Mark uncertain claims as `[Unverified]` in the response.
