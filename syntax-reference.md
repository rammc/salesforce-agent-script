# Agent Script — Syntax Reference

Comprehensive reference for the Agent Script language. Read this when writing
new script, verifying property names, or looking up the exact form of a
construct.

## Contents

- [Language Characteristics](#language-characteristics)
- [Resource References](#resource-references)
- [Reasoning Instructions](#reasoning-instructions)
- [Operators](#operators)
- [Conditional Expressions](#conditional-expressions)
- [Variables](#variables)
- [Actions](#actions)
- [Tools (Reasoning Actions)](#tools-reasoning-actions)
- [Utils](#utils)
- [Block Reference](#block-reference)

## Language Characteristics

- **Compiled.** Saving an agent version compiles the script to lower-level
  metadata for the Atlas Reasoning Engine.
- **Hybrid.** Combines deterministic logic (`->`) with LLM prompt instructions
  (`|`) in one workflow.
- **Declarative + procedural.** Top-level blocks are declarative; reasoning
  instructions are procedural.
- **Whitespace-sensitive.** Indentation defines structure (like YAML or
  Python). **Spaces only — Salesforce recommends 3 spaces per level.** Tabs
  are explicitly discouraged. Never mix the two.
- **Property syntax.** Everything is `key: value`. Multiline values use `|`.
- **Comments.** Start with `#`; ignored to end of line.
- **Blocks** are top-level properties (`config`, `system`, `topic`, etc.).

### Trailing-line gotcha

If you encounter an unexpected error in Agentforce Builder on the **last line**
of your script, add a blank line or a comment to the end. This is a known
parser quirk.

## Resource References

All resource references use the `@` prefix.

| Reference | Refers to |
|---|---|
| `@actions.<name>` | An action defined in the current topic |
| `@variables.<name>` | A global variable (defined in the `variables` block) |
| `@outputs.<name>` | An action output, used inside `set` after a `run` |
| `@subagent.<name>` | Another subagent (direct call returns to caller) |
| `@topic.<name>` | Legacy alias for `@subagent.<name>` (deprecated) |
| `@subagent.<name>` | Newer alias for `@topic` |
| `@utils.<function>` | A built-in utility (e.g., `@utils.transition`, `@utils.escalate`) |
| `@system_variables.user_input` | The customer's most recent utterance (read-only) |

### Bang-brace interpolation

Inside prompt instruction text, references must be wrapped in `{!…}`:

```
| Hi {!@variables.user_name}, your order {!@variables.order_id} ships {!@variables.ship_date}.
```

Outside prompt text — in logic, conditions, action bindings — use the bare
form:

```
-> if @variables.is_member == True
->   set @variables.discount to 10
```

## Reasoning Instructions

A topic's `reasoning.instructions` block contains a sequence that resolves
top-to-bottom into a single prompt sent to the LLM.

Two instruction types:

- **Logic instructions** (prefix `->`): deterministic. Run during parsing.
  Used for `if`/`else`, `run` actions, `set` variables, `transition to`.
- **Prompt instructions** (prefix `|`): natural-language strings concatenated
  to the prompt and sent to the LLM after parsing.

```
reasoning:
  instructions:
    -> run @actions.get_delivery_date
    -> set @variables.delivery_date = @outputs.date
    | Tell the customer their package arrives on {!@variables.delivery_date}.
    -> run @actions.check_if_late
    -> set @variables.is_late = @outputs.is_late
    -> if @variables.is_late == True
    |   Apologize for the delay.
```

**Order matters.** Instructions resolve sequentially. Variable mutations
earlier in the topic are visible to later instructions; mutations later in
the topic do *not* travel back.

**Prompt assembly.** The LLM only sees the resolved prompt — the concatenation
of all `|` strings (and conditionally included strings whose `if` evaluated
true). It does not see the `->` lines.

**Shorter prompts win.** The Salesforce guidance is explicit: shorter
reasoning instructions yield more accurate, reliable agent behavior. Resist
the temptation to "explain everything" in prompt text.

### Multiline strings

Use `|` for multiline values in `system` messages, descriptions, and prompt
instructions:

```
description: |
  Helps customers track an order, request a refund, or change the
  delivery address. Available only after the customer has been verified.
```

### before_reasoning and after_reasoning

A topic can include `before_reasoning` and `after_reasoning` blocks that run
logic before/after the reasoning step:

```
subagent order_status:
  before_reasoning:
    -> set @variables.num_turns to @variables.num_turns + 1
  reasoning:
    instructions:
      -> # ... main logic and prompt
  after_reasoning:
    -> if @variables.num_turns > 5
    ->   transition to @subagent.escalation
```

`before_reasoning` is the place for required pre-conditions and counters.
`after_reasoning` is the place for post-conditions, including conditional
transitions out of the topic.

## Operators

| Category | Operator | Notes |
|---|---|---|
| Comparison | `==` `!=` | Equality / inequality |
| Comparison | `<` `<=` `>` `>=` | Numeric / date ordering |
| Identity | `is None`, `is not None` | Null check |
| Logical | `and`, `or`, `not` | Boolean composition |
| Arithmetic | `+`, `-` | Numbers only; no `*`, `/`, `%` in script |

**No string operators.** Concatenation, contains, regex — all out of scope.
Use an action when you need string manipulation.

**No `else if`.** Only `if` and `else`. Nest a second `if` inside the `else`
branch when you need a chain.

## Conditional Expressions

```
-> if @variables.balance > 100
->   set @variables.tier to "gold"
->   run @actions.notify_premium
-> else
->   set @variables.tier to "standard"
```

Conditions can also gate prompt text:

```
-> if @variables.is_member == True
|   You're getting member pricing today.
-> else
|   Sign up to unlock member pricing.
```

`if` blocks may contain logic, prompts, action calls, transitions — anything
valid in reasoning instructions.

## Variables

Declared in the global `variables` block; visible to every topic.

### Variable kinds

- **Regular** — has a default value, optionally `mutable`. Can be read and (if
  mutable) written by both logic and the LLM (when `description` is set).
- **Linked** — value is bound to an action output. No default, never mutable
  by agent, restricted to primitive types.
- **System** — predefined, read-only. Currently only
  `@system_variables.user_input` (the latest customer utterance).

### Naming rules (same for actions and agents)

- Begins with a letter, not underscore.
- Alphanumeric + underscore only.
- Cannot end with underscore.
- Cannot contain consecutive underscores (`__`).
- Maximum 80 characters.

### Regular variable types

| Type | Notes | Example |
|---|---|---|
| `string` | Alphanumeric text. | `name: mutable string = "Alex"` |
| `number` | IEEE 754 double; integers and decimals. | `price: mutable number = 99.99` |
| `boolean` | `True` / `False`. **Capitalized.** | `is_active: mutable boolean = True` |
| `object` | JSON object literal. | `order: mutable object = {"sku": "abc", "qty": 2}` |
| `date` | Date literal `YYYY-MM-DD`. | `start_date: mutable date = 2026-01-15` |
| `id` | Salesforce record ID. | `account_id: id = "0015000000XyZ12"` |
| `list[<type>]` | Homogeneous list of any primitive type. | `flags: mutable list[boolean] = [True, False]` |

### Linked variable types

`string`, `number`, `boolean`, `date`, `id` only. No `object`, no `list`.

### Variable properties

- `mutable` — flag that allows mutation. Omit for immutable.
- `description` — when set, the LLM may set the value via slot filling
  (using `@utils.set` with a description).
- `label` — UI display name. Auto-generated from the variable name if omitted.

### Slot filling (LLM-extracts values)

Two distinct mechanisms — pick based on what you need.

**Mechanism 1 — `...` token in action inputs** (canonical, for action calls).
The `...` token tells the LLM to fill the value from conversation context when
it calls the tool:

```
reasoning:
  actions:
    capture_user_info:
      tool: @actions.capture_user_info
      description: Record the customer's first and last name.
      inputs:
        first_name: ...
        last_name: ...
```

The `...` token is **only valid for action inputs in `reasoning.actions`** —
do NOT use it as a default value for a variable. The LLM extracts the value
from the customer's utterance and passes it as the input parameter.

**Mechanism 2 — `@utils.setVariables`** (when the goal is to set a variable
directly without calling a domain action). The variable's `description` tells
the LLM what to extract:

```
variables:
  shipping_address:
    type: string
    mutable: True
    description: |
      The customer's full shipping address, including street, city, state,
      and zip code.

reasoning:
  actions:
    capture_address:
      tool: @utils.setVariables
      description: Use to record the shipping address when the customer provides it.
```

Pick `...` in action inputs when the value flows into a downstream system
call. Pick `@utils.setVariables` when you want to capture state into the
variables block without an action invocation.

## Actions

An action wraps an executable: Apex, Flow, or Prompt Template. Defined in a
topic's `actions` block; consumed either by logic (deterministic call) or by
the LLM through `reasoning.actions` (tool exposure).

### Action properties

| Property | Required | Notes |
|---|---|---|
| `<action_name>` | yes | The action's identifier; same naming rules as variables. |
| `description` | optional | LLM uses this to decide when to call (when exposed as tool). |
| `inputs` | optional | Parameter map (see types below). |
| `outputs` | optional | Output parameter map. Available via `outputs.<name>`. |
| `target` | yes | `apex://<DEVELOPER_NAME>`, `flow://<DEVELOPER_NAME>`, or `prompt://<DEVELOPER_NAME>`. |
| `label` | optional | UI label. |
| `include_in_progress_indicator` | optional | Show progress UI while running. |
| `require_user_confirmation` | optional | Prompt customer before running. |

### Parameter types

`string`, `number`, `integer`, `long`, `boolean`, `object`, `date`,
`datetime`, `time`, `currency`, `id`, `list[<type>]`.

(Note: action parameter types are broader than variable types — `integer`,
`long`, `datetime`, `time`, `currency` exist for parameters but not as
top-level variable types.)

### Output properties

| Property | Notes |
|---|---|
| `description` | Auto-generated from name if omitted. |
| `developer_name` | Required. Override the underlying parameter name. |
| `label` | UI label. Auto-generated if omitted. |
| `complex_data_type_name` | Required for complex object outputs (e.g., `lightning__recordInfoType`). |
| `filter_from_agent` | If `True`, output is excluded from the agent's context. Default `False`. |

### Calling actions from logic

```
reasoning:
  instructions:
    -> run @actions.get_business_hours with timezone = @variables.user_tz
    -> set @variables.is_open = @outputs.is_open
    -> set @variables.next_open = @outputs.next_open
```

Use `run` to invoke, `with` to bind inputs, `set` to capture outputs.

### Exposing actions as tools

```
subagent order_lookup:
  reasoning:
    actions:
      lookup_order:
        tool: @actions.get_order_by_number
        description: Use when the customer provides an order number and wants details.
    instructions:
      | Help the customer find their order.
      | If they provide an order number, use {!@actions.get_order_by_number} to look it up.
  actions:
    get_order_by_number:
      description: Retrieves order details by order number.
      inputs:
        order_number:
          type: string
          required: True
      outputs:
        order_status:
          type: string
        delivery_date:
          type: date
      target: flow://Get_Order_By_Number
```

Same action can appear in both `reasoning.actions` (LLM-callable) and be
called directly via `run` in logic — they're different surfaces over the same
definition.

### Chaining tools

[Unverified] Some Salesforce documentation references a `then:` property on a
reasoning action to specify a follow-up that runs deterministically after the
tool returns:

```
reasoning:
  actions:
    get_order:
      tool: @actions.get_order_by_number
      then: @actions.schedule_order
```

The exact property name and shape (`then:` vs. `next:` vs. another spelling)
varies across blog posts and recipes. Verify against the
[Action Chaining pattern doc](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-patterns-action-chaining.html)
before using.

The reliable, always-correct alternative is deterministic chaining in
`reasoning.instructions` (covered in [Calling actions from logic](#calling-actions-from-logic))
— call action A with `run`, then call action B with `run`. No LLM choice
involved; both run in order, every time.

## Tools (Reasoning Actions)

Two `actions` blocks exist on every topic — keep them straight:

| Block | Audience | Purpose |
|---|---|---|
| `topic.actions` | Your logic | Action definitions you can `run` directly. |
| `topic.reasoning.actions` | The LLM | "Tools" — the LLM may call them based on context. |

A reasoning action ("tool") can wrap:

- A regular action (`tool: @actions.<name>`)
- A utility (`tool: @utils.transition to @subagent.<name>`, `@utils.setVariables`,
  `@utils.escalate`)
- A direct subagent call (`tool: @subagent.<name>` — flow returns after completion)

### Tool properties

```
reasoning:
  actions:
    transfer_to_billing:
      tool: @utils.transition to @subagent.billing
      description: Use when the customer asks about invoices, payments, or refunds.
      available_when: @variables.is_verified == True
```

`available_when` deterministically gates whether the LLM can see this tool
on a given turn. Useful for pre-condition gating without deeply nested
reasoning logic.

### Direct subagent call vs. transition

| Style | Syntax | Returns? |
|---|---|---|
| Transition | `@utils.transition to @subagent.x` | No — one-way, prompt is discarded |
| Direct call | `tool: @subagent.x` | Yes — flow returns to caller |

Use direct call when you have a reusable sub-capability (identity check,
data lookup) and the flow should resume in the calling subagent afterward.

## Utils

Built-in utility functions usable as tools or directly in logic.

### `@utils.transition to @subagent.<name>`

One-way handoff to another topic. Discards any pending prompt. Executes
**immediately** when encountered in logic — code after it does not run. The
target topic starts fresh from its top.

After the target topic completes, control does **not** return; instead, the
agent waits for the next customer utterance and re-enters at `start_agent`.

Available from: reasoning instructions, reasoning actions, `before_reasoning`,
`after_reasoning`, and from within `actions:` blocks.

### `@utils.setVariables`

Wrap as a tool to instruct the LLM to set variable values from the
conversation. The variable's `description` is the slot-filling instruction;
the LLM extracts the value from the user's utterance.

```
variables:
  shipping_address:
    type: string
    mutable: True
    description: |
      The customer's full shipping address — street, city, state, and zip.

reasoning:
  actions:
    capture_address:
      tool: @utils.setVariables
      description: Record the shipping address when the customer provides it.
```

The legacy form `@utils.set @variables.<n>` appears in some older recipes;
prefer `@utils.setVariables` for new code per the canonical Reference table.


### `@utils.escalate`

Hands off to a human service rep through Omni-Channel. Requires a
`connection messaging` block with `outbound_route_type` and
`outbound_route_name` configured.

`escalate` is a reserved keyword; do not use it as a topic or action name.

## Block Reference

### `config`

Agent identity and runtime parameters.

| Property | Notes |
|---|---|
| `developer_name` | Required. API name; unique in the org. Naming rules apply. Max 80 chars. |
| `default_agent_user` | Required for `AgentforceServiceAgent`. API name or ID of the running user. |
| `agent_label` | UI label. Auto-generated from `developer_name` if omitted. |
| `description` | Free-form description. |
| `company` | Optional company info. |
| `role` | Optional role description. |
| `agent_version` | Auto-set on save. |
| `agent_type` | `AgentforceServiceAgent` (default) or `AgentforceEmployeeAgent`. |
| `enable_enhanced_event_logs` | `True`/`False`. Default `False`. Conversation logging. |
| `user_locale` | Optional locale. |

### `system`

Global instructions and required messages.

```
system:
  instructions: |
    You are a friendly retail assistant for ACME Corp. Always speak in plain
    language and avoid jargon.
  welcome: Hi {!@variables.user_name}, how can I help today?
  error: Something went wrong on my side. Could you try that again?
```

`welcome` and `error` are required.

### `variables`

See [Variables](#variables).

### `language`

```
language:
  default: en_US
  supported: [en_US, de_DE, fr_FR]
```

### `connection`

External integrations. Most common: Enhanced Chat for `@utils.escalate`.

```
connection messaging:
  outbound_route_type: <type>
  outbound_route_name: <name>
```

### `start_agent <name>`

The router topic. Same structure as a regular topic, but uses `start_agent`
prefix. Runs at the start of every customer turn.

```
start_agent topic_selector:
  description: Routes the conversation based on intent.
  reasoning:
    instructions:
      | Determine what the customer wants and route accordingly.
    actions:
      go_to_billing:
        tool: @utils.transition to @subagent.billing
        description: For invoice, payment, refund questions.
      go_to_orders:
        tool: @utils.transition to @subagent.order_lookup
        description: For tracking, modifying, or cancelling orders.
```

### `topic <name>`

A specialized capability.

| Property | Notes |
|---|---|
| `description` | Specific, distinct, customer-vocabulary. Drives routing. |
| `system.instructions` | Optional override of global system instructions for this topic. |
| `before_reasoning` | Optional. Logic that runs before the reasoning step. |
| `reasoning.instructions` | Required. Logic + prompt sequence sent to the LLM. |
| `reasoning.actions` | Optional. Tools exposed to the LLM. |
| `after_reasoning` | Optional. Logic that runs after the reasoning step. |
| `actions` | Optional. Action definitions for this topic. |

Each topic owns its own copies of any imported actions — actions are not
shared across topics.
