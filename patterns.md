# Agent Script — Patterns

Idiomatic patterns for common Agent Script tasks. Each pattern shows the
canonical shape, when to apply it, and what to watch for. Patterns are
distilled from the Salesforce developer guide and the agent-script-recipes
sample app.

## Contents

- [Topic Selector / Routing (start_agent block)](#topic-selector--routing-start_agent-block)
- [Identity Verification](#identity-verification)
- [Slot Filling](#slot-filling)
- [Action Chaining and Sequencing](#action-chaining-and-sequencing)
- [Fetch Data Before Reasoning](#fetch-data-before-reasoning)
- [Conditional Prompting](#conditional-prompting)
- [Transitions](#transitions)
- [Filtering with available_when](#filtering-with-available_when)
- [System Instruction Overrides](#system-instruction-overrides)
- [Required Subagent Workflow](#required-subagent-workflow)
- [Resource References in Prompts](#resource-references-in-prompts)
- [Variables for State Across Turns](#variables-for-state-across-turns)

## Topic Selector / Routing (start_agent block)

The `start_agent` block runs at the start of every customer turn. It is the
agent's router. Keep it focused on classification + routing — no business
logic, minimal prompting.

```
start_agent topic_selector:
  description: Routes the customer to the appropriate subagent.
  reasoning:
    instructions:
      | Determine what the customer is asking about and route them to the
      | correct topic. If the request is unclear, ask one short clarifying
      | question. If the request is out of scope (medical, legal, financial
      | advice), politely decline.
    actions:
      go_to_orders:
        tool: @utils.transition to @subagent.order_management
        description: Order tracking, status, modifications, cancellations.
      go_to_billing:
        tool: @utils.transition to @subagent.billing
        description: Invoices, payments, refunds, payment methods.
      go_to_returns:
        tool: @utils.transition to @subagent.returns
        description: Return requests, refund status, exchange policies.
```

**Why it works:** The LLM sees three clearly distinct tools with
non-overlapping descriptions. The prompt explicitly delegates classification
to the LLM rather than forcing keyword matching.

**Watch for:** descriptions that overlap ("orders" and "order returns") cause
inconsistent routing. Use distinct vocabulary.

## Identity Verification

A reusable pattern: capture email, send a code, verify it. Combines
deterministic logic for the security-critical steps with LLM-driven
conversation for everything else.

```
subagent identity:
  description: Verifies the customer's identity via email verification code.
  reasoning:
    instructions:
      -> if @variables.customer_email is None
      |   Ask the customer for their email address. Wait for their reply.
      -> if @variables.customer_email is not None
      ->   if @variables.code_sent == False
      ->     run @actions.send_verification_code with email = @variables.customer_email
      ->     set @variables.code_sent to True
      |   I just sent a verification code to {!@variables.customer_email}. Please share it with me.
    actions:
      capture_email:
        tool: @utils.setVariables
        description: |
          Record the customer's email address when they provide it. Set
          @variables.customer_email.
      verify_code:
        tool: @actions.verify_code
        description: Use when the customer provides the verification code they received.

  actions:
    send_verification_code:
      target: apex://SendVerificationCode
      inputs:
        email:
          type: string
          required: True
    verify_code:
      target: apex://VerifyCode
      description: Verifies a code the customer provides against the one we sent.
      inputs:
        code:
          type: string
          required: True
      outputs:
        is_valid:
          type: boolean
```

**Why it works:** Sending the code is gated by deterministic logic so it
happens exactly once. Capturing the email and verifying the code are LLM-
driven because both depend on parsing free-form customer input.

**Watch for:** repeating `if @variables.customer_email is not None` blocks
gets verbose fast. Once your subagent has more than two nested `if`s,
consider splitting into smaller subagents or moving steps to
`before_reasoning`.

## Slot Filling

Let the LLM extract structured data from free-form customer input. Two
mechanisms, depending on whether the extracted value should drive a downstream
action or just be captured as state.

### Slot filling into action inputs (canonical)

The `...` token in `reasoning.actions[].inputs` tells the LLM to extract that
input from the conversation when it decides to call the tool:

```
variables:
  first_name:
    type: string
    mutable: True
  last_name:
    type: string
    mutable: True

reasoning:
  actions:
    capture_user_info:
      tool: @actions.capture_user_info
      description: Capture the customer's first and last name.
      inputs:
        first_name: ...
        last_name: ...
  instructions:
    | Greet the customer warmly and ask for their first and last name. Use
    | {!@actions.capture_user_info} to record what they provide.

actions:
  capture_user_info:
    description: Stores the customer's name for use across the conversation.
    inputs:
      first_name:
        type: string
        required: True
      last_name:
        type: string
        required: True
    target: apex://CaptureUserInfo
```

### Slot filling directly into variables

When you just need to capture state (no domain action runs), wrap
`@utils.setVariables` in a tool. The variable's `description` is the
extraction instruction:

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
      description: Record the shipping address when the customer provides it.
```

**Why it works:** the variable's `description` (or the action input's name)
is the slot-filling instruction. The LLM reads it, extracts the value from
the conversation, and binds it.

**Watch for:** vague descriptions yield vague extractions. Be explicit about
format: "including street, city, state, and zip code" gives the LLM a
checklist. The `...` token is **only valid in action inputs** — never use it
as a variable default.

## Action Chaining and Sequencing

When N actions must run in a fixed order, do **not** expose them as
independent tools. The LLM might call them out of order, or skip one.

### Chain in logic

```
reasoning:
  instructions:
    -> run @actions.lookup_account with customer_id = @variables.customer_id
    -> set @variables.account_status = @outputs.status
    -> set @variables.account_balance = @outputs.balance
    -> if @variables.account_status == "active"
    ->   run @actions.calculate_eligible_offers with balance = @variables.account_balance
    ->   set @variables.offers = @outputs.offer_list
    | Walk the customer through their available offers in
    | {!@variables.offers}.
```

### Chain through a tool follow-up

[Unverified] If the chain starts from an LLM-callable tool but the follow-up
must run deterministically, Salesforce documentation describes a follow-up
property on the reasoning action — likely `then:`. Verify the exact spelling
in the [Action Chaining pattern doc](https://developer.salesforce.com/docs/ai/agentforce/guide/ascript-patterns-action-chaining.html)
before using it in production.

If in doubt, prefer the deterministic chain in logic (above) — it is
unambiguous and always works.

**Watch for:** "the LLM forgot to call action B after action A" is the canonical
sign you needed a chain, not two independent tools.

## Fetch Data Before Reasoning

When the LLM's response depends on data that must be loaded first, fetch in
`before_reasoning` so the prompt has the data already resolved.

```
subagent order_status:
  before_reasoning:
    -> run @actions.get_order_details with order_id = @variables.order_id
    -> set @variables.order_status = @outputs.status
    -> set @variables.eta = @outputs.estimated_delivery
  reasoning:
    instructions:
      | Tell the customer their order is currently {!@variables.order_status}
      | and is expected to arrive on {!@variables.eta}.
```

**Why it works:** the action runs once, deterministically, before the prompt
is assembled. The LLM does not get the chance to "decide" whether to look up
the order — it has to.

**Watch for:** putting expensive lookups in `before_reasoning` even when the
data is not always needed. If only some branches need the data, fetch inside
the relevant `if`.

## Conditional Prompting

Different prompt content depending on state.

```
reasoning:
  instructions:
    -> if @variables.is_premium_member == True
    |   Greet the customer warmly and mention their VIP status. Offer them
    |   priority handling.
    -> else
    |   Greet the customer politely. If they ask about benefits, mention that
    |   premium membership unlocks priority handling.
```

**Why it works:** business rules (membership state) live in logic; tone and
phrasing live in prompt instructions.

**Watch for:** the antipattern is the inverse — embedding the rule in the
prompt: `| If they're premium, say X, otherwise Y`. The LLM may misclassify;
logic gives you certainty.

## Transitions

### Hard transition (always go there next)

From inside reasoning instructions:

```
reasoning:
  instructions:
    -> run @actions.check_eligibility
    -> set @variables.is_eligible = @outputs.is_eligible
    -> if @variables.is_eligible == False
    ->   transition to @subagent.ineligible_handler
    | The customer is eligible. Walk them through enrollment.
```

The transition fires immediately; everything after it in the topic is
discarded.

### Soft transition (LLM may choose)

Expose the transition as a tool in `reasoning.actions`:

```
reasoning:
  actions:
    transfer_to_billing:
      tool: @utils.transition to @subagent.billing
      description: Use when the customer's question is about invoices or payments.
```

The LLM decides whether and when to call it.

### Conditional transition out of a subagent

Use `after_reasoning` to transition based on a final condition:

```
subagent order_lookup:
  reasoning:
    instructions:
      | Help the customer find their order. Use the lookup tool if they
      | provide an order number.
    actions:
      lookup:
        tool: @actions.get_order_by_number
  after_reasoning:
    -> if @variables.lookup_failed == True
    ->   transition to @subagent.escalation
```

**Watch for:** transitions are one-way. Anything you needed to communicate to
the customer must already be in the resolved prompt of the *current* topic
before the transition fires. After the transition, only the destination
topic's prompt reaches the LLM.

## Filtering with `available_when`

Hide a tool or topic from the LLM when a precondition is not met.

```
reasoning:
  actions:
    cancel_order:
      tool: @actions.cancel_order
      description: Cancels the order if eligible.
      available_when: |
        @variables.is_verified == True and
        @variables.order_status == "pending"
```

**Why it works:** the LLM never sees the tool when the condition is false, so
there is no risk of it being called inappropriately. The agent does not need
to maintain "if not allowed, refuse" instructions in the prompt.

**Watch for:** complex `available_when` expressions become hard to debug. If
you have more than two clauses, consider hoisting the check into a variable
set in `before_reasoning`.

## System Instruction Overrides

System-level instructions (in the global `system` block) apply to every
subagent. A subagent-level `system.instructions` override replaces them for
that subagent.
This avoids contradictory instructions reaching the LLM.

```
system:
  instructions: |
    You are a friendly retail assistant for ACME Corp. Always speak casually
    and use first names.

subagent legal_disclosure:
  system:
    instructions: |
      You must speak formally and precisely. Never use first names. Read the
      customer the disclosure exactly as written; do not paraphrase.
  description: Reads the legally required disclosure to the customer.
```

**Why it works:** the override scopes the new tone to the one subagent
where it matters, without polluting the rest of the agent.

**Watch for:** if you find yourself overriding `system.instructions` in
most subagents, the global `system.instructions` is probably too prescriptive
— pull it back to the truly universal rules.

## Required Subagent Workflow

Force the customer through a required step before allowing other
capabilities.

```
start_agent topic_selector:
  reasoning:
    instructions:
      -> if @variables.is_verified == False
      ->   transition to @subagent.identity
      | Route the customer to the appropriate topic.
    actions:
      go_to_orders:
        tool: @utils.transition to @subagent.order_management
        description: For order tracking and modifications.
        available_when: @variables.is_verified == True
      go_to_billing:
        tool: @utils.transition to @subagent.billing
        description: For billing questions.
        available_when: @variables.is_verified == True
```

After identity verification completes (and presumably sets
`@variables.is_verified to True`), subsequent turns hit the start_agent again
and skip the forced redirect.

**Watch for:** the verification topic must explicitly set the gating variable.
A common bug is verifying the user but never flipping the flag, causing an
infinite redirect loop.

## Resource References in Prompts

Explicitly reference variables, actions, and subagents in prompt instructions
to give the LLM more context.

```
reasoning:
  instructions:
    | The customer has placed {!@variables.order_count} orders this year and
    | their loyalty tier is {!@variables.loyalty_tier}. If they ask about
    | rewards, use {!@actions.get_loyalty_balance} to look up their points.
```

The bang-brace form interpolates the value at prompt-resolution time. The LLM
sees the actual numbers and the tool reference.

**Why it works:** "@-mentioning" tools in the prompt increases the chance the
LLM calls them when intended.

**Watch for:** over-referencing makes prompts noisy and harder for the LLM to
parse. Reference the resources that genuinely guide the LLM's decision; trust
the LLM to find the rest from descriptions.

## Variables for State Across Turns

Variables are the agent's reliable memory between turns. The LLM remembers
the conversation, but for anything that drives logic, use a variable.

```
variables:
  num_failed_attempts: mutable number = 0
  is_verified: mutable boolean = False
  current_order_id: mutable string

subagent identity:
  reasoning:
    instructions:
      -> run @actions.verify_code with code = @variables.code_attempt
      -> set @variables.is_valid = @outputs.is_valid
      -> if @variables.is_valid == True
      ->   set @variables.is_verified to True
      |   Verification successful. How can I help?
      -> else
      ->   set @variables.num_failed_attempts to @variables.num_failed_attempts + 1
      ->   if @variables.num_failed_attempts >= 3
      ->     transition to @subagent.escalation
      |   That code didn't match. Want me to send another?
```

**Why it works:** the failed-attempts counter is reliable across turns. The
LLM does not need to "remember" how many tries the customer has had — the
variable does.

**Watch for:** mutable variables let the LLM (with `description`) and your
logic both write. If both can write, define clear ownership in your design,
or you'll chase race-style bugs.
