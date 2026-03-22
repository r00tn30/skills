# ADK Prompt Writer — Quick Reference Cheat Sheet
## Rapid lookup for ADK-specific prompt engineering

---

## THE TWO-SENTENCE RULE (internalize this first)

> **1.** LLMs are text completion engines that mimic their training data.
> **2.** Write ADK `instruction` fields that look like the beginning of a document Gemini was trained to complete correctly.

Everything else is application of these two sentences.

---

## ADK AGENT ANATOMY

```
┌─────────────────────────────────────────────────────────────────┐
│  Agent(                                                          │
│    name="snake_case_name",        ← identity for routing/logs   │
│    model="gemini-2.0-flash",                                     │
│    description="...",             ← written FOR the orchestrator │
│    instruction="...",             ← written FOR the Gemini model │
│    tools=[fn_1, fn_2],            ← docstrings ARE the prompts  │
│    sub_agents=[agent_1],          ← delegated sub-tasks         │
│    before_tool_callback=gate_fn,  ← application-layer safety    │
│  )                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## `instruction` FIELD STRUCTURE — REQUIRED ORDER

```
┌─────────────────────────────────────────────────────────────┐
│  1. ROLE + TASK INSTRUCTION          ← primacy effect       │
│  2. BEHAVIOURAL RULES & CONSTRAINTS                         │
│  3. TOOL USAGE POLICY (when / when not)                     │
│  4. FEW-SHOT EXAMPLES (if any)       ← Valley of Meh        │
│  5. RETRIEVED CONTEXT (RAG)          ← Valley of Meh        │
│  6. USER QUERY / TASK INPUT          ← recency effect ★    │
└─────────────────────────────────────────────────────────────┘
★ Always last. Highest attention. Never bury the task input in the middle.
```

---

## `description` FIELD RULES

| What it is | What it is NOT |
|---|---|
| Written for the orchestrator agent | Written for the model inside this agent |
| "Call when [condition]" format | Task instructions |
| One sentence: capability + trigger | Vague capability labels |

Strong: `"Looks up real-time order status from the fulfilment API. Call when user asks about an existing order."`
Weak: `"Handles order related things"`

---

## ADK TOOL DESIGN (5 non-negotiables)

1. **≤10 tools** active at once — more = confusion and hallucination
2. **`verb_object` snake_case** — `search_order_history` not `get_data`
3. **≤3 parameters** — use `Literal[...]` for bounded values; `Optional[x] = None` for optional args
4. **Remove known values** — if the app knows `user_id`, inject it inside the function, never expose as param
5. **Dangerous tools: gate via `before_tool_callback`** — never rely on `"confirm first"` in docstring

---

## TOOL DOCSTRING FORMAT

```python
def verb_object(param: str) -> dict:
    """Line 1: what it does + what it returns.

    Use when: [specific trigger].
    Do NOT use when: [overlap exclusion].

    Args:
        param: Description. Format: [format]. Example: "value".

    Returns:
        dict with field (type): description.
        On failure: {"error": "reason"}.
    """
```

---

## ADK AGENT TYPE SELECTOR

```
Is step order fixed and each output feeds the next?
  YES → SequentialAgent

Can sub-tasks run concurrently without dependencies?
  YES → ParallelAgent

Does the task need iteration / self-correction / retry?
  YES → LoopAgent (always set max_iterations)

Do you need LLM-driven routing between sub-agents?
  YES → Standard Agent with sub_agents + routing instruction
```

---

## REASONING FRAMEWORK SELECTOR (ADK)

```
Multi-step logical / mathematical task?
  YES → Chain-of-Thought (CoT)
        Add to instruction: "Think step by step before answering"
        Or use few-shot CoT with reasoning BEFORE answer

Task requires tool calls interleaved with reasoning?
  YES → ADK handles ReAct natively — shape reasoning quality in instruction:
        "Reason before each tool call. Evaluate each result before the next."
        Production: fine-tune on ~3k think-act-observe traces

Complex task with benefit from upfront planning?
  YES → Add to instruction: "Write a numbered plan before executing any tools."

Output quality needs iterative verification?
  YES → Reflexion — add self-review rubric at end of instruction
        Or use LoopAgent with test_runner sub-agent

Multiple perspectives improve the answer?
  YES → ParallelAgent (N solver agents) + merge agent

Using Gemini 2.5 Pro or Flash Thinking?
  YES → Skip explicit CoT — state goals + constraints only
```

---

## ADK CALLBACK QUICK GUIDE

| Callback | When to use |
|---|---|
| `before_tool_callback` | Gate dangerous tools; require user approval before execution |
| `after_tool_callback` | Validate/enrich tool outputs; add recovery hints to errors |
| `on_before_model` | Inject user-specific context into instruction at runtime |
| `on_after_model` | Post-process model output before it's returned |

**Pattern for dangerous tools:**
```python
def gate(tool, args, ctx):
    if tool.name in DANGEROUS:
        return {"status": "pending_approval", ...}  # Short-circuit
    return None  # Proceed
```

---

## STRUCTURED OUTPUT IN ADK

| Method | When to use |
|---|---|
| `response_schema` + `response_mime_type="application/json"` | Extraction agents, data pipelines |
| Tool-call forcing (output schema as a tool) | Complex structured outputs with validation |
| Instruction + postprocessing | Simple cases only — fragile on format variation |

**Never rely on free-text parsing of structured output.**

---

## INPUT DELIMITING RULES

| Content type | Delimiter |
|---|---|
| Retrieved documents | `<document source="...">...</document>` |
| User-provided text | `<user_input>...</user_input>` |
| Prior agent output | `<prior_output agent="name">...</prior_output>` |
| Code | Triple backticks with language tag |
| Structured data | `<data>...</data>` |

**Inertness rule**: Never let injected content end with `Answer:`, `Output:`, `Result:` — append `---` after dynamic content.

---

## MULTI-AGENT ORCHESTRATOR INSTRUCTION TEMPLATE

```
You are a workflow orchestrator for [goal].

Sub-agents:
- [agent_name]: [capability]. Call when [condition].
- [agent_name]: [capability]. Call after [prerequisite].

Rules:
- Always call [first_agent] first
- Never call [agent_B] before [agent_A] succeeds
- If any agent returns an error: stop, report the failure, wait for instructions
- Call signal_completion only when ALL steps have succeeded
```

---

## ATTENTION ZONES

| Zone | Effect | Use for |
|---|---|---|
| **Start** | Primacy — high attention | Role, rules, critical constraints |
| **Middle** | Valley of Meh — LOWEST attention | Examples, supporting context (keep tight) |
| **End** | Recency — HIGHEST attention | The actual question / task input |

---

## GEMINI MODEL SELECTION

| Model | Best for |
|---|---|
| `gemini-2.0-flash` | Most agent tasks — fast, capable, cost-effective |
| `gemini-1.5-pro` | Long context (1M tokens), complex reasoning |
| `gemini-2.5-pro` | Hard reasoning tasks — do NOT add explicit CoT |
| `gemini-2.0-flash-thinking` | Multi-step reasoning — do NOT add explicit CoT |

**Temperature for ADK agents:**
- Extraction, classification, tool selection → `0.0`
- General agent tasks → `0.7` (default)
- Creative brainstorming → `0.9`

---

## ANTI-PATTERNS → FIXES

| Anti-pattern | Fix |
|---|---|
| `instruction="You are a helpful assistant"` | Specify domain, expertise, concrete objective |
| `description` written for the model | Write for the orchestrator: "Call when [condition]" |
| `"Confirm with user before calling"` in docstring | `before_tool_callback` gate in application layer |
| `user_id` as tool parameter | Inject inside the function body |
| No `Literal` for bounded params | Use `Literal["a", "b", "c"]` type hint |
| Sub-agent output not delimited | Wrap in `<prior_output agent="name">` tags |
| User query buried in middle of instruction | Always last — recency = highest attention |
| Explicit CoT with Gemini thinking models | State goals + constraints only |
| `LoopAgent` without `max_iterations` | Always set a hard iteration ceiling |
| Free-text parsing of structured output | `response_schema` or tool-call forcing |

---

## TOKEN QUICK MATH

```
1 token  ≈  0.75 words  ≈  4 characters  (English)
1,000 words   ≈  1,333 tokens
1 paragraph   ≈  80–150 tokens
1 sentence    ≈  15–30 tokens
1 tool def (simple docstring)   ≈  50–150 tokens
1 few-shot example              ≈  50–200 tokens
```

---

## MULTI-AGENT ARCHITECTURE HIERARCHY

```
Prefer simpler → only add complexity when evaluation data demands it

1. Single Agent + tools              ← most maintainable; start here
       ↓
2. SequentialAgent / ParallelAgent / LoopAgent (fixed topology)
       ↓
3. Orchestrator Agent with sub_agents (LLM-driven routing)
       ↓
4. Nested orchestrators / stateful multi-agent pipelines
   (most flexible, least stable, hardest to debug)
```
