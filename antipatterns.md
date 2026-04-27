# Agent Script — Antipatterns

Common mistakes when writing Agent Script, why they fail, and what to do
instead. Use as a review aid and as a checklist when an agent misbehaves
unexpectedly.

## Contents

- [Embedding business rules in prompt text](#embedding-business-rules-in-prompt-text)
- [Overloaded prompts (the "doomprompt")](#overloaded-prompts-the-doomprompt)
- [Independent tools where you needed a chain](#independent-tools-where-you-needed-a-chain)
- [Conflicting system instructions](#conflicting-system-instructions)
- [Stuffing all logic into one mega-subagent](#stuffing-all-logic-into-one-mega-subagent)
- [Trusting the LLM with state](#trusting-the-llm-with-state)
- [Forgotten transition flag-flip](#forgotten-transition-flag-flip)
- [Code after a transition](#code-after-a-transition)
- [Overlapping subagent descriptions](#overlapping-subagent-descriptions)
- [Default-mutable variables](#default-mutable-variables)
- [Mixing tabs and spaces](#mixing-tabs-and-spaces)
- [Forgetting bang-brace in prompt text](#forgetting-bang-brace-in-prompt-text)
- [`else if` (not supported)](#else-if-not-supported)
- [Putting expensive lookups in `before_reasoning` unconditionally](#putting-expensive-lookups-in-before_reasoning-unconditionally)
- [Reusing `escalate` as a subagent name](#reusing-escalate-as-a-subagent-name)

## Embedding business rules in prompt text

```
| If the order total is over $100, offer free shipping. Otherwise charge
| $7.99 for shipping. Premium members always get free shipping.
```

**Why it fails:** the LLM may misread the threshold, miscount, or apply the
rule inconsistently across turns. You also have no audit trail — the rule
is buried in conversational text.

**Do instead:** put the rule in logic, the phrasing in the prompt.

```
-> if @variables.order_total > 100 or @variables.is_premium == True
->   set @variables.shipping_cost to 0
-> else
->   set @variables.shipping_cost to 7.99
| Shipping for this order is {!@variables.shipping_cost} dollars.
```

## Overloaded prompts (the "doomprompt")

The Salesforce team coined "doomprompting" for prompts that try to encode every
edge case in natural language. Symptoms: prompts longer than 30 lines, lots
of "if X then say Y, but only when Z, unless W."

**Why it fails:** LLMs degrade with prompt length and instruction density. The
more rules you cram in, the more inconsistent the agent becomes.

**Do instead:** decompose. Move rules to logic. Move scenarios to separate
topics with focused prompts. Move data fetches to `before_reasoning` so the
prompt sees resolved values, not instructions to look things up.

The shorter your reasoning instructions, the more reliable the agent.

## Independent tools where you needed a chain

```
reasoning:
  actions:
    create_lead:
      tool: @actions.create_lead
      description: Creates a new lead.
    send_welcome_email:
      tool: @actions.send_welcome_email
      description: Sends the welcome email.
```

**Why it fails:** the LLM may call `create_lead` and never call
`send_welcome_email`, or call them in the wrong order, depending on how the
prompt is phrased.

**Do instead:** if both must always run in order, chain them deterministically
in reasoning instructions:

```
-> run @actions.create_lead with email = @variables.email
-> set @variables.lead_id = @outputs.id
-> run @actions.send_welcome_email with lead_id = @variables.lead_id
```

## Conflicting system instructions

Global `system.instructions` says "always speak casually." A topic prompt says
"speak formally for this disclosure." A reasoning instruction says "use the
customer's first name."

**Why it fails:** the LLM averages the conflicting guidance, producing
inconsistent tone within a single response.

**Do instead:** when a topic genuinely needs a different voice, use
`system.instructions` *override* at the topic level (see Patterns). Don't
layer contradictions in the prompt instructions.

## Stuffing all logic into one mega-subagent

A single subagent with 80 lines of `if`/`else`, six action definitions, and
prompt text trying to handle four different customer intents.

**Why it fails:** the LLM gets a sprawling resolved prompt with everything
relevant *and* irrelevant. Routing decisions inside the prompt become
unreliable. Reviewing or modifying the subagent becomes painful.

**Do instead:** decompose into 3–7 focused subagents. Use `start_agent` to
route. Each subagent should fit on one screen and answer one question:
"what is this subagent for?"

The Salesforce guidance is explicit: "small and separate scripts for tasks
like identity checks make your system easier to maintain and scale."

## Trusting the LLM with state

```
| Remember that the customer's name is Alex and their order ID is 1234.
| Use these values in subsequent steps.
```

**Why it fails:** the LLM's working memory is the conversation transcript. It
*usually* remembers, but it's not reliable for values that drive logic. State
that's wrong silently is worse than state that's missing loudly.

**Do instead:** put state in variables. Variables are deterministic; the LLM
sees them through interpolation but does not have to "remember."

```
variables:
  customer_name: mutable string
  order_id: mutable string

reasoning:
  instructions:
    | Address the customer as {!@variables.customer_name} when speaking to them.
```

## Forgotten transition flag-flip

A `start_agent` redirects unverified customers to the identity subagent. The
identity subagent verifies the customer but never sets `is_verified` to
`True`.

**Symptom:** infinite redirect loop. The customer verifies, gets routed back
to start_agent, gets sent back to identity, and so on.

**Do instead:** review every gating variable for *both* its check site (the
`if`) and its set site (the `set`). Both must exist; both must be in the
right places.

## Code after a transition

```
reasoning:
  instructions:
    -> if @variables.is_eligible == False
    ->   transition to @subagent.ineligible
    -> set @variables.eligibility_checked to True   # never runs!
    | Walk the customer through enrollment.
```

**Why it fails:** `transition to` is immediate. Lines after it in the same
subagent do not execute. The set will never happen for the ineligible branch.

**Do instead:** treat `transition to` like `return` in a function. Hoist any
state mutation that must happen before the transition above the transition,
and recognize that nothing after it will run on that branch.

## Overlapping subagent descriptions

```
subagent orders:
  description: Help the customer with their orders.
subagent order_returns:
  description: Help the customer return an order.
subagent refunds:
  description: Help the customer get a refund.
```

**Why it fails:** "return an order" and "get a refund" overlap heavily.
The LLM sees three plausible candidates and picks inconsistently. The
customer's experience varies turn to turn.

**Do instead:** make descriptions distinct. Use customer vocabulary
consistently. State what the subagent *is for* and, where it helps, what
it is *not* for.

```
subagent order_management:
  description: |
    Track an order, change the delivery address, or cancel an unshipped order.
    Use this topic only for changes BEFORE the order has shipped.
subagent returns_and_refunds:
  description: |
    Process a return for a delivered order, including refund status. Use this
    topic AFTER the order has shipped or been delivered.
```

## Default-mutable variables

Marking every variable `mutable` "just in case."

**Why it fails:** mutability is a permission. Mutable variables can change
under your feet — from logic, from `@utils.set` slot filling, from the LLM
in another topic. Bugs compound.

**Do instead:** start immutable. Add `mutable` only when you have a concrete
need to mutate. The compiler does not enforce immutability rules across
topics for you, so this is purely a discipline — but it prevents whole
classes of state bugs.

## Mixing tabs and spaces

Agent Script is whitespace-sensitive. The compiler rejects mixed indentation
within a block, sometimes with cryptic errors.

**Do instead:** use spaces (Salesforce recommends 3 per indentation level).
Avoid tabs entirely — the Salesforce-published guidance is spaces-only, and
mixing has caused parser errors in the field. Configure your editor to enforce
spaces for `.agent` files. The Agentforce DX VS Code extension handles this
automatically.

## Forgetting bang-brace in prompt text

```
| Hi @variables.user_name, your order @variables.order_id is on the way.
```

**Why it fails:** outside logic, the bare `@…` reference is **literal text**.
The LLM sees the placeholder string verbatim and may either include it
literally in the response or hallucinate values for it.

**Do instead:** in prompt text (after `|`), wrap every reference in
`{!…}`:

```
| Hi {!@variables.user_name}, your order {!@variables.order_id} is on the way.
```

In logic (after `->`), use the bare form: `if @variables.is_member == True`.

## `else if` (not supported)

```
-> if @variables.tier == "gold"
->   set @variables.discount to 20
-> else if @variables.tier == "silver"   # PARSE ERROR
->   set @variables.discount to 10
```

**Why it fails:** Agent Script supports only `if` and `else`. There is no
`else if`.

**Do instead:** nest:

```
-> if @variables.tier == "gold"
->   set @variables.discount to 20
-> else
->   if @variables.tier == "silver"
->     set @variables.discount to 10
->   else
->     set @variables.discount to 0
```

For more than three branches, hoist the logic into a Flow or Apex action and
return the result as a variable. Deep nesting in script becomes unreadable
fast.

## Putting expensive lookups in `before_reasoning` unconditionally

```
subagent answer_question:
  before_reasoning:
    -> run @actions.fetch_full_customer_history   # always runs, even for "what's your return policy?"
    -> set @variables.history = @outputs.history
```

**Why it fails:** the action runs on every entry to the topic, even when the
data is not needed. Costs add up; latency degrades the customer experience.

**Do instead:** fetch conditionally inside the topic, gated on whether the
data is actually used:

```
reasoning:
  instructions:
    -> if @variables.user_input contains_keyword "my orders"   # conceptual; use a tool
    ->   run @actions.fetch_full_customer_history
    ->   set @variables.history = @outputs.history
```

Or move the lookup into a tool the LLM can choose to call when relevant.

## Reusing `escalate` as a subagent name

`escalate` is a reserved keyword for the `@utils.escalate` utility. Naming a
subagent or action `escalate` causes a parse error.

**Do instead:** use specific names: `escalate_to_human`, `human_handoff`,
`agent_transfer`, etc.
