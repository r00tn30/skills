---
name: adk-prompt-writer
description: Expert prompt engineer for Google ADK (Agent Development Kit) agents and sub-agents. Invoke whenever you need to write, critique, or improve ADK agent instructions, tool docstrings, sub-agent task prompts, reasoning frameworks (CoT/ReAct), orchestration logic, or any structured prompt component destined for an ADK agent. Applies the complete prompt engineering canon to the specific constructs of the ADK framework — instruction fields, Python tool docstrings, SequentialAgent/ParallelAgent/LoopAgent compositions, and Gemini model configurations. Use this before generating any ADK agent definition, not just when reviewing existing ones.
---

# ADK Prompt Writer — Expert Agent Prompt Engineering for Google ADK

**CRITICAL**: Apply every rule below *as you write ADK agent definitions*. The `instruction` field is the model's behavioural constitution. Tool docstrings are themselves prompts. Get both right before writing any code.

Full reference material in:
- `references/prompt-engineering-foundations.md` — LLM mechanics, document archetypes, assembly algorithms
- `references/adk-agent-patterns.md` — ADK-specific patterns: instruction fields, tool docstrings, agent composition, callbacks
- `references/assembly-and-formatting.md` — formatting rules, elastic snippets, token budget management
- `assets/quick-reference.md` — rapid-lookup cheat sheet including ADK-specific constructs

---

## The Four Non-Negotiable LLM Facts

Internalise these before writing a single token. Every rule below flows from them.

| Fact | Prompt Engineering Implication |
|---|---|
| **1. LLMs are document completion engines** | Every `instruction` is the start of a document the model was trained to complete. Write it to look like one. |
| **2. LLMs mimic their training data** | Instructions resembling high-quality training documents produce high-quality completions. Sloppy → sloppy. |
| **3. LLMs generate one token at a time, no backtracking** | Front-load constraints in the `instruction` field. Errors compound forward. |
| **4. LLMs read left to right, once** | Context before instructions. The model interprets tool docstrings and sub-agent descriptions through what it has already read. |

**The Little Red Riding Hood Principle**: Don't stray from the path the model was trained on. Use document formats the model has seen millions of times: Markdown headers, Q&A patterns, Python docstrings, XML tags. Unusual formats → unpredictable outputs.

---

## ADK Agent Anatomy

### The Three ADK Prompt Surfaces

Every ADK agent has three places where you write prompts. All three matter.

```python
Agent(
    name="agent_name",               # Surface 1: identity signal — used in multi-agent routing
    model="gemini-2.0-flash",
    description="...",               # Surface 2: orchestrator-facing description — how parent agents pick this sub-agent
    instruction="...",               # Surface 3: model-facing system prompt — the behavioural constitution
    tools=[tool_fn_1, tool_fn_2],
    sub_agents=[sub_agent_1],
)
```

**Surface 1 — `name`**: Machine-readable identity. Used in multi-agent routing and logging. Use `snake_case`, be specific: `order_status_agent` not `agent1`.

**Surface 2 — `description`**: Written for the *orchestrator agent*, not the model inside this agent. It is how a parent agent decides to delegate to this child. Treat it as a one-sentence capability summary:
- Strong: `"Looks up real-time order status and shipping updates from the fulfilment API. Call when the user asks about an existing order."`
- Weak: `"Handles order stuff"`

**Surface 3 — `instruction`**: Written for the Gemini model inside this agent. This is the full system prompt. All the prompt engineering rules in this skill apply here.

---

## When Writing ADK `instruction` Fields

### Required Structure (prevents: vague behaviour, scope creep)

The `instruction` is the model's **behavioural constitution** — processed first, conditions everything.

**Elements, in order:**
1. **Role declaration** — who the agent is and its specific objective (1–2 concrete sentences)
2. **Scope boundaries** — what it handles AND what it explicitly does NOT handle
3. **Behavioural rules** — tone, format, uncertainty handling
4. **Tool usage policy** — when to use tools vs respond directly
5. **Output format specification** — exactly how responses must be structured

```python
Agent(
    name="billing_support_agent",
    model="gemini-2.0-flash",
    description="Resolves billing disputes and processes refund requests. Call when the user has a payment complaint or refund request.",
    instruction="""You are a billing support specialist for Acme SaaS. Your job is to resolve payment disputes and process refund requests.

You DO: look up invoices, explain charges, initiate refunds within policy limits, escalate edge cases.
You DO NOT: modify subscription plans, change user permissions, or discuss non-billing topics.

When responding:
- Acknowledge the issue before explaining or acting
- Cite specific invoice numbers and dates when referencing charges
- If the refund amount exceeds $500, escalate — never approve unilaterally
- If you cannot resolve the issue: say so explicitly and provide the escalation path

Tool usage:
- Use get_invoice_history when the user references a specific charge
- Use initiate_refund only after confirming the amount and reason with the user in this conversation
- Never call initiate_refund without first calling get_invoice_history
""",
    tools=[get_invoice_history, initiate_refund],
)
```

### Role and Persona Rules

- **Be domain-specific** — activates a specific region of Gemini's training distribution
  - Strong: `"You are a senior infrastructure engineer who has debugged Kubernetes production incidents at 3 AM"`
  - Weak: `"You are a helpful technical assistant"`
- **Avoid contradictory signals** — `"be extremely concise"` + `"explain everything thoroughly"` = incoherent output
- **Positive phrasing over prohibitions** — "respond in two sentences" beats "don't be verbose"
- **Specific over vague** — "respond in exactly 2 bullet points" beats "be concise"
- **State uncertainty handling explicitly** — RLHF models are trained to admit uncertainty; leverage this:
  - `"If you don't know with confidence, say so rather than guessing"`

### Alignment Tax Awareness

RLHF trades some raw capability for safety. For complex reasoning agents:
- Add `"Think step by step before responding"` to recover reasoning depth lost to alignment
- For maximum capability on sub-tasks: use explicit chain-of-thought scaffolding
- Match verbosity instruction to task: analytical tasks need space; extraction needs tight constraints

---

## When Writing ADK Tool Definitions

ADK tools are Python functions. The docstring IS the tool description — the model reads it to understand what it can do and when. The function signature IS the parameter schema. Design both for the model, not the API.

### ADK Tool Design Rules

```python
def get_order_status(order_id: str) -> dict:
    """Look up the current status and shipping details for a specific order.

    Returns a dict with keys: status, carrier, tracking_number, estimated_delivery.
    Use when the user asks about an existing order's whereabouts or status.
    Do NOT use for order history or browsing past orders — use search_order_history instead.

    Args:
        order_id: The order identifier (format: ORD-XXXXXX). Found on the order confirmation email.

    Returns:
        dict with status, carrier, tracking_number, estimated_delivery, or
        {"error": "not_found"} if the order_id does not exist.
    """
    ...
```

**Docstring rules:**
- First line: what it does + what it returns (one sentence)
- Second line: when to use it
- Third line: when NOT to use it (if ambiguous domain overlap with another tool)
- Args section: format of each arg, where to get it, valid values
- Returns section: exact shape of the return value including error cases

**Naming:**
- `verb_object` snake_case — matches Python training data priors
- Self-documenting: `search_order_history` not `get_data`
- Bad: `process`, `handle`, `run` — always be specific about what, on what, when

**Arguments:**
- Keep to 3 or fewer per tool
- Use `Literal["value1", "value2"]` type hints for bounded values — prevents hallucination
- Use `Optional[str] = None` for optional args — the default is catchable
- If the application already knows the value (`user_id`, `session_id`), inject it inside the function — **never expose it as a parameter**

**Dangerous tools — NEVER rely on docstring instructions:**
- Never write `"confirm with user before calling"` in the docstring — models are stochastic; this fails ~5% of calls
- Use ADK's `before_tool_callback` to intercept dangerous calls at the application layer:

```python
def gate_dangerous_tools(tool: BaseTool, args: dict, ctx: CallbackContext):
    if tool.name in {"delete_account", "initiate_refund", "send_email"}:
        # Pause execution, surface to user for explicit sign-off
        return {"pending_approval": True, "tool": tool.name, "args": args}
    return None  # None = proceed

agent = Agent(
    ...,
    before_tool_callback=gate_dangerous_tools,
)
```

**Tool count:**
- Maximum 10 tools active at once — more creates confusion and hallucination
- Tools must **partition the domain** — full coverage, no overlap

### ADK Tool Definition Template

```python
def verb_object_noun(
    required_param: str,
    bounded_param: Literal["option_a", "option_b", "option_c"],
    optional_param: Optional[str] = None,
) -> dict:
    """One sentence: what this tool does and what it returns.

    Use when: [specific trigger condition].
    Do NOT use when: [overlap exclusion if needed].

    Args:
        required_param: Description. Format: [format]. Example: [example].
        bounded_param: Which [X] to [Y]. Defaults to None which means [default behaviour].
        optional_param: Description. Omit unless [condition].

    Returns:
        dict with [field]: [type] — [description], or {"error": "[reason]"} on failure.
    """
    ...
```

---

## When Writing Sub-Agent Task Prompts

Sub-agent `instruction` fields are **task specifications**, not conversations. Self-contained, unambiguous, structured output required.

### The Five-Element Task Instruction

```python
sub_agent = Agent(
    name="invoice_extractor",
    model="gemini-2.0-flash",
    description="Extracts structured invoice data from raw text. Use when you have invoice text that needs parsing into structured fields.",
    instruction="""<role>
You are an invoice data extractor. Your only job is to parse raw invoice text into structured JSON.
</role>

<task>
Extract all invoice fields from the provided text. Be precise — do not infer or guess missing fields.
</task>

<output_schema>
Return ONLY valid JSON matching this schema — no preamble, no explanation:
{
  "invoice_number": "string — the invoice ID",
  "vendor_name": "string — the issuing company",
  "total_amount": "number — total in the invoice currency",
  "currency": "string — ISO 4217 code",
  "due_date": "string — YYYY-MM-DD format"
}
</output_schema>

<constraints>
- If a required field is missing from the text: set its value to null
- Do not hallucinate values not present in the text
- If the text does not appear to be an invoice: return {"error": "not_an_invoice"}
</constraints>
""",
)
```

### Input Delimiting Rules (prevents: instruction–content confusion)

Always wrap injected content — never let dynamic content bleed into instructions:

| Content type | Recommended delimiter |
|---|---|
| Retrieved documents | `<document source="...">...</document>` |
| User-provided text | `<user_input>...</user_input>` |
| Prior agent output | `<prior_output agent="...">...</prior_output>` |
| Code | Triple backticks with language tag |
| Structured data | `<data>...</data>` |

**Inertness rule**: Injected content must not accidentally trigger generation. Never let dynamic content end with `Answer:`, `Output:`, `Result:` — always append `---` or a newline after dynamic content.

### Enforcing Structured Output

1. **Tool-call forcing** — define output schema as a tool; parse tool call arguments, not completion text
2. **JSON mode** — via `google.generativeai` `response_mime_type="application/json"`
3. **Format instruction + postprocessing** — `"Return only valid JSON, nothing else"` + robust parser with retry

Never rely on free-text parsing. It breaks on any format variation.

---

## When Writing Reasoning Prompts (CoT / ReAct)

### Chain-of-Thought (CoT)

**Use when**: multi-step logic, math, causal reasoning, decisions where the first intuition is likely wrong.

**Zero-shot CoT** (simplest — add to the `instruction`):
```
Think step by step before giving your final answer.
```

**Few-shot CoT** (strongest — reasoning MUST precede the answer):
```
Here are examples of how to approach this task:

---
Problem: [example 1]
Reasoning:
- Step 1: [first reasoning step]
- Step 2: [second reasoning step]
Answer: [final answer]

Problem: [example 2]
Reasoning:
- ...
Answer: ...

Problem: {{actual_problem}}
Reasoning:
```

**CoT for Gemini reasoning models** (Gemini 2.0 Flash Thinking, Gemini 2.5 Pro):
- Do NOT add explicit CoT scaffolding — these models reason internally
- Focus on: clear goal, explicit constraints, leave reasoning to the model

### ReAct (Reasoning + Acting)

**Use when**: multi-step tasks requiring tool calls interleaved with reasoning.

ADK natively supports the ReAct loop — the framework handles Think/Act/Observe automatically when tools are defined. Your job is to configure the `instruction` to guide the reasoning quality:

```python
Agent(
    name="research_agent",
    model="gemini-2.0-flash",
    instruction="""You are a research agent. Answer questions by gathering information from multiple sources.

For every task:
1. Before calling any tool, reason about which source is most relevant
2. After receiving a tool result, evaluate what gaps remain
3. Call tools one at a time — do not batch tool calls
4. Stop calling tools once you have sufficient information
5. Synthesise your answer from tool observations only — do not add information you did not retrieve

If a tool returns an error: reason about why, try a different approach before giving up.
""",
    tools=[search_web, fetch_document, query_database],
)
```

**ReAct production note**: For production agents, collect real think-act-observe traces from ADK's built-in tracing and fine-tune. A fine-tuned 8B ReAct model outperforms a vanilla 62B standard model.

### Plan-and-Solve (before entering ReAct):
```
Before you begin, write a numbered plan:
Plan:
1. [step]
2. [step]
Then execute the plan using your available tools.
```

---

## ADK Multi-Agent Composition Patterns

### Orchestrator Agent Instruction

```python
orchestrator = Agent(
    name="onboarding_orchestrator",
    model="gemini-2.0-flash",
    description="Orchestrates the full customer onboarding workflow end to end.",
    instruction="""You are a workflow orchestrator responsible for completing customer onboarding.

You have access to specialist sub-agents. Delegate tasks to them — do not attempt to do their work yourself.

Sub-agents available:
- account_creator: Creates the user account. Call first, before anything else.
- email_verifier: Sends and verifies the confirmation email. Call after account_creator succeeds.
- provisioner: Provisions the user's workspace. Call after email_verifier succeeds.
- welcome_notifier: Sends the welcome message. Call last, only after all other steps succeed.

Workflow rules:
- Always call account_creator first — no other step works without an account
- Never call provisioner before email_verifier confirms success
- If any sub-agent returns an error: stop the workflow, report the specific failure to the user, and ask how to proceed
- Do not skip steps to save time — the order is required for correctness
""",
    sub_agents=[account_creator, email_verifier, provisioner, welcome_notifier],
)
```

### SequentialAgent — When Order Is Fixed

Use `SequentialAgent` when the pipeline has a strict linear order and each step's output feeds the next:

```python
from google.adk.agents import SequentialAgent

pipeline = SequentialAgent(
    name="document_processing_pipeline",
    description="Extracts, classifies, and routes incoming documents.",
    sub_agents=[extractor_agent, classifier_agent, router_agent],
    # Instruction applies to the orchestration layer, not the sub-agents
    instruction="Process each document through extraction, classification, and routing in order. Stop and report if any stage fails.",
)
```

Each sub-agent's `instruction` should:
- Accept the previous agent's output as its input (inject via `<prior_output>` tags)
- Produce a clean, typed output for the next agent

### ParallelAgent — When Steps Are Independent

Use `ParallelAgent` when sub-tasks can run concurrently:

```python
from google.adk.agents import ParallelAgent

researcher = ParallelAgent(
    name="multi_source_researcher",
    description="Queries multiple data sources concurrently and returns combined results.",
    sub_agents=[web_search_agent, database_agent, document_agent],
    instruction="Run all sub-agents concurrently. Combine their results into a unified response. If one source fails, report it but do not block the others.",
)
```

### LoopAgent — When Iteration Is Needed

Use `LoopAgent` for self-correction, polling, or retry patterns:

```python
from google.adk.agents import LoopAgent

validator = LoopAgent(
    name="code_validator",
    description="Generates and iteratively fixes code until all tests pass.",
    sub_agents=[code_generator_agent, test_runner_agent],
    max_iterations=5,
    instruction="Each iteration: generate or fix code, then run tests. Continue until all tests pass or max iterations reached. On final failure, return the best attempt with the remaining failures listed.",
)
```

---

## When Writing Workflow Task Prompts (LLM Pipelines)

Workflow task `instruction` fields are deterministic processing units — not conversations. Each must:

- Have **single responsibility** — one task, one output type, no branching inside the instruction
- Define **explicit I/O schema** — input format and output format completely specified
- Be **self-contained** — works without any conversation history
- Specify **failure mode** — what to output on error or edge case

### ADK Workflow Task Template

```python
processing_agent = Agent(
    name="task_name",
    model="gemini-2.0-flash",
    description="One sentence: what this agent does. Used by orchestrators to decide when to call it.",
    instruction="""<role>You are a [task-specific role]. Your only job is to [specific task].</role>

<input_schema>
You will receive:
- field_name: type — description
</input_schema>

<task>[Imperative instruction. Concrete. One paragraph.]</task>

<input>{{serialised_input}}</input>

<output_schema>
Return ONLY valid JSON — no preamble:
{"field": "value"}
</output_schema>

<edge_cases>
- If [condition]: return {"error": "description", "retry": true}
</edge_cases>
""",
)
```

### Two-Step Ideation Pattern (proven to outperform one-shot):

```
Step 1 agent instruction: "Brainstorm [N] candidate [items] for [context]. Return a numbered list, no evaluation."

Step 2 agent instruction: "Given these candidates and this context, select the single best option.
Justify in 2 sentences citing specific details from the context."
```

---

## Prompt Assembly Rules

### The Attention U-Curve (where to place what in `instruction`)

| Position | Attention level | What to put here |
|---|---|---|
| **Start** (primacy effect) | High | Role, task instruction, critical rules |
| **Middle** (Valley of Meh) | Low | Few-shot examples, supporting context — keep tight |
| **End** (recency effect) | Highest | The actual user query / task input |

**Assembly order for ADK agent `instruction` fields:**
```
[1] Role + task instruction          ← primacy effect — use it
[2] Behavioural rules & constraints
[3] Tool usage policy (when, when not)
[4] Few-shot examples (if any)       ← middle — unavoidable, keep them concise
[5] Retrieved context (RAG)          ← middle — top-k by relevance only
[6] Actual user query / task input   ← recency effect — maximum attention here
```

### Token Budget for ADK `instruction` Fields

| Component | Priority | Typical tokens |
|---|---|---|
| Role + instruction | Tier 1 — always include | 50–200 |
| Behavioural rules | Tier 1 — always include | 50–300 |
| Tool usage policy | Tier 1 (if tools used) | 50–200 |
| Few-shot examples | Tier 2 — include if budget allows | 200–1000 |
| Retrieved context | Tier 2 — top-k by relevance | 500–3000 |
| User query | Fixed — always last | Variable |

---

## Output Control Rules

### Preamble Management in ADK

| Preamble type | What it looks like | How to handle |
|---|---|---|
| **Structural boilerplate** | `"Here is your answer:"`, opening XML tags | Inception trick: pre-populate the model's opening words in the instruction |
| **Chain-of-thought** | Reasoning steps before the answer | Preserve deliberately; extract final answer via postprocessing |
| **Fluff** | Disclaimers, background, re-stating the question | `"Answer first, then any explanation"` format |

**Answer-first format** — eliminates most fluff:
```
Provide your answer in this format:
1. [Main answer — one sentence]
2. [Explanation or caveats, if needed]
```

### ADK Structured Output Enforcement

```python
# Option 1: response_schema (recommended for extraction agents)
from google.generativeai.types import GenerationConfig

agent = Agent(
    name="extractor",
    model="gemini-2.0-flash",
    generation_config=GenerationConfig(
        response_mime_type="application/json",
        response_schema=MyOutputSchema,  # Pydantic model or dict schema
    ),
    instruction="Extract the requested fields. Return only the JSON object.",
)

# Option 2: Tool-call forcing (best for complex outputs)
# Define output schema as a tool; the model's tool call IS the structured output
def submit_extraction_result(invoice_number: str, total: float, currency: str) -> dict:
    """Submit the extracted invoice fields. Call this exactly once with all extracted values."""
    return {"status": "submitted"}
```

---

## Few-Shot Prompting Rules

- **Zero-shot first** — if Gemini succeeds, examples aren't worth the token cost
- **3–6 examples** — fewer risks anchoring; more than 10 rarely helps
- **Represent the full output distribution** — if you expect ratings 1–5, cover all five in examples
- **Randomise example ordering** — prevents the model from inferring ascending/descending trends
- **CoT examples**: reasoning MUST precede the answer in every example

---

## ADK Callback Patterns for Application-Layer Control

### before_tool_callback — Gate dangerous tools

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.tools import BaseTool

DANGEROUS_TOOLS = {"delete_account", "send_email", "initiate_payment", "modify_permissions"}

def require_user_approval(tool: BaseTool, args: dict, ctx: CallbackContext) -> Optional[dict]:
    """Intercept dangerous tools before execution."""
    if tool.name in DANGEROUS_TOOLS:
        # Return a dict to short-circuit tool execution and surface to UI
        return {
            "approval_required": True,
            "tool": tool.name,
            "args": args,
            "message": f"Confirm: execute {tool.name} with {args}?"
        }
    return None  # None = allow the tool to execute

agent = Agent(..., before_tool_callback=require_user_approval)
```

### after_tool_callback — Validate and transform tool outputs

```python
def validate_tool_output(tool: BaseTool, args: dict, result: dict, ctx: CallbackContext) -> Optional[dict]:
    """Intercept and validate tool outputs before the model sees them."""
    if "error" in result:
        # Enrich error with retry guidance so the model can recover
        result["retry_suggestion"] = f"Try {tool.name} with different parameters."
    return result  # Return modified result; return None to pass through unchanged
```

---

## Quality Checklist — Before Finalising Any ADK Agent Definition

**`instruction` field:**
- [ ] Role is specific — activates the right Gemini training distribution region
- [ ] Scope boundaries explicit — what agent does AND does not do
- [ ] Output format completely specified — no ambiguity
- [ ] Uncertainty handling explicitly instructed
- [ ] Critical constraints in first half of the instruction

**`description` field:**
- [ ] Written for the *orchestrator*, not the model
- [ ] One sentence: capability + trigger condition
- [ ] No overlap with sibling agents' descriptions

**Tool docstrings:**
- [ ] First line: what it does + what it returns
- [ ] When to use + when NOT to use (if ambiguous)
- [ ] Arg descriptions include format and valid values
- [ ] Known application values injected inside function — not exposed as parameters
- [ ] Dangerous tools gated via `before_tool_callback`, not docstring instructions

**Multi-agent structure:**
- [ ] Agent type matches topology (`SequentialAgent` for ordered, `ParallelAgent` for concurrent, `LoopAgent` for iteration)
- [ ] Orchestrator instruction lists each sub-agent's name, capability, and call condition
- [ ] Sub-agent `name` fields are `snake_case` and self-documenting
- [ ] Output schema of each agent matches input expectation of the next

**Output:**
- [ ] Structured output enforced via `response_schema` or tool-call forcing
- [ ] No reliance on free-text parsing of structured data

---

## Anti-Patterns — Fix These Immediately

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| `instruction="You are a helpful assistant"` | Broadest distribution, lowest quality | Specify domain, expertise, specific objective |
| `description` written for the model, not the orchestrator | Orchestrator can't determine when to delegate | Write description as "call when [condition]" |
| Tool docstring says "confirm with user before calling" | Fails ~5% of calls stochastically | Use `before_tool_callback` to gate at application layer |
| `user_id` or `session_id` as tool parameter | Model can hallucinate it | Inject known values inside the function body |
| No `Literal` type hints for bounded tool args | Model hallucinates invalid values | Use `Literal["a", "b", "c"]` to constrain |
| Sub-agent outputs not delimited when passed to next agent | Instruction–content confusion | Wrap in `<prior_output agent="name">` tags |
| All few-shot examples with same output value | Anchoring bias | Distribute across full output range |
| User query buried in the middle of `instruction` | Valley of Meh — lowest attention | Always put the actual task at the very end |
| Explicit CoT scaffolding with Gemini 2.5 Pro / Flash Thinking | Conflicts with internal reasoning | State goals + constraints only for thinking models |
| `LoopAgent` without `max_iterations` | Infinite loop on persistent failure | Always set a hard iteration ceiling |
