---
name: prompt-writer
description: Expert prompt engineer for LLM agents and sub-agents. Invoke whenever you need to write, critique, or improve system prompts, task instructions, tool definitions, reasoning frameworks (CoT/ReAct), sub-agent delegation prompts, or any structured prompt component destined for an LLM agent. This skill distills the complete "Prompt Engineering for LLMs" (O'Reilly, Berryman & Ziegler) into actionable rules applied proactively at prompt-write time — not after the fact. Use this before generating any prompt, not just when reviewing existing ones.
---

# Prompt Writer — Expert Agent Prompt Engineering

**CRITICAL**: Apply every rule below *as you write prompts*. Never hand-wave the structure, content, or assembly. If a prompt component would violate a rule, apply the correct technique immediately.

Full reference material in:
- `references/prompt-engineering-foundations.md` — LLM mechanics, document archetypes, assembly algorithms
- `references/agent-prompt-patterns.md` — tool design, CoT, ReAct, multi-agent workflows
- `references/assembly-and-formatting.md` — formatting rules, elastic snippets, token budget management
- `assets/quick-reference.md` — rapid-lookup cheat sheet for all key concepts

---

## The Four Non-Negotiable LLM Facts

Internalise these before writing a single token. Every rule below flows from them.

| Fact | Prompt Engineering Implication |
|---|---|
| **1. LLMs are document completion engines** | Every prompt is the start of a document the model was trained to complete. Write it to look like one. |
| **2. LLMs mimic their training data** | Prompts resembling high-quality training documents produce high-quality completions. Sloppy → sloppy. |
| **3. LLMs generate one token at a time, no backtracking** | Front-load constraints. Errors compound forward. Position critical content early. |
| **4. LLMs read left to right, once** | Context before instructions. The model interprets instructions through what it has already read. |

**The Little Red Riding Hood Principle**: Don't stray from the path the model was trained on. Use document formats the model has seen millions of times: Markdown headers, Q&A patterns, TypeScript signatures, XML tags, ChatML. Unusual formats → unpredictable outputs.

---

## When Writing Agent System Prompts

### Required Structure (prevents: vague behaviour, scope creep)

A system prompt is the model's **behavioural constitution** — processed first, conditions everything. Get it right.

**Elements, in order:**
1. **Role declaration** — who the agent is and its specific objective (1–2 concrete sentences)
2. **Scope boundaries** — what it handles AND what it explicitly does NOT handle
3. **Behavioural rules** — tone, format, uncertainty handling
4. **Tool usage policy** — when to use tools vs respond directly
5. **Output format specification** — exactly how responses must be structured

```
You are [specific role], specialised in [domain]. Your job is to [concrete objective].

You DO: [in-scope behaviours]
You DO NOT: [explicit exclusions — critical for preventing hallucination and scope creep]

When responding:
- [Format rule 1 — be specific, not vague]
- [Format rule 2]
- If unsure: say so explicitly rather than guessing

Tool usage:
- Use [tool A] when [condition]
- Never use tools unless [precondition]
```

### Role and Persona Rules

- **Be domain-specific** — activates a specific region of the model's training distribution
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

## When Writing Sub-Agent Task Prompts

Sub-agent prompts are **task specifications**, not conversations. Self-contained, unambiguous, structured output required.

### The Five-Element Task Prompt

```
<role>
You are a [task-specific role]. Your only job is to [verb + object].
</role>

<task>
[Imperative instruction. One paragraph max. Concrete and unambiguous.]
</task>

<input>
{{serialised_input}}
</input>

<output_schema>
Return ONLY valid JSON matching this schema — no preamble, no explanation:
{
  "field_name": "type — description",
  "field_name": "type — description"
}
</output_schema>

<constraints>
- If [edge case]: [specific handling]
- Do not [specific exclusion]
</constraints>
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

**Inertness rule**: Snippets must not accidentally trigger generation. Never let injected content end with `Answer:`, `Output:`, `Result:` — always append a separator (`---` or newline) after dynamic content.

### Enforcing Structured Output (order of preference)

1. **Tool-call forcing** (`tool_choice="required"`) — define output schema as a tool; parse tool call arguments, not completion text
2. **JSON mode** — grammar-constrained decoding guarantees valid JSON
3. **Format instruction + postprocessing** — `"Return only valid JSON, nothing else"` + robust parser with retry

Never rely on free-text parsing. It breaks on any format variation.

---

## When Writing Tool Definitions

Tool definitions are themselves a prompt — the model reads them to understand what it can do and when. Design for the model, not the API.

### Tool Design Rules

**Scope and count:**
- Maximum 10 tools active at once — more creates confusion and hallucination
- Tools must **partition the domain** — full coverage, no overlap
- Strip web APIs to their minimal useful interface; complex argument lists cause misuse

**Naming (camelCase for OpenAI — matches TypeScript training data priors):**
- Self-documenting: `fetchUserOrderHistory` not `getData`
- Bad: `process`, `handle`, `run` — always be specific about what, on what, when

**Descriptions:**
- One clear sentence: what it does + when to use it
- Describe the output format so the model can anticipate what it will receive
- No legalese, no excessive caveats — they consume attention capacity

**Arguments:**
- Keep to 3 or fewer
- Use `enum` for bounded values; `default` for optional args (prevents hallucination)
- Named arguments force the model to "say" the parameter name before its value → fewer assignment errors

**Argument hallucination mitigation:**
- If the application already knows the value (`user_id`, `org`), **remove it from the tool definition entirely**
- Provide `default` values for optional args so the model's fallback is catchable

**Dangerous tools — NEVER rely on prompt instructions:**
- Never write `"check with the user before executing"` — models are stochastic; this fails ~5% of calls
- Implement application-layer interception: let the model request any call; gate dangerous ones in code; require explicit user sign-off in the UI

```typescript
// Good tool definition
{
  name: "searchOrderHistory",
  description: "Search a customer's order history by date range. Returns up to 20 orders sorted by date descending. Use when the user asks about past purchases or order status. Do NOT use for real-time inventory.",
  parameters: {
    type: "object",
    properties: {
      dateFrom: {
        type: "string",
        description: "Start date YYYY-MM-DD. Defaults to 30 days ago."
      },
      category: {
        type: "string",
        enum: ["electronics", "clothing", "food", "all"],
        default: "all"
      }
    },
    required: []
  }
}
```

---

## When Writing Reasoning Prompts (CoT / ReAct)

### Chain-of-Thought (CoT)

**Use when**: multi-step logic, math, causal reasoning, decisions where the first intuition is likely wrong.

**Zero-shot CoT** (simplest):
```
Append: "Think step by step before giving your final answer."
```

**Few-shot CoT** (strongest — reasoning MUST precede the answer):
```
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

**CoT for reasoning models** (o1, o3, DeepSeek-R1, Claude extended thinking):
- Do NOT add explicit CoT scaffolding — these models reason internally; external CoT conflicts with the internal process
- Focus on: clear goal, explicit constraints, leave reasoning to the model

### ReAct (Reasoning + Acting)

**Use when**: multi-step tasks requiring tool calls interleaved with reasoning, any task where the plan depends on discovered information.

**ReAct system prompt template:**
```
You are [agent role].

You have access to these tools:
[tool definitions]

For every task, follow this exact format:
Think: [reason about what to do next and why]
Act: [call exactly one tool]
Observe: [tool result — provided by the system]
... repeat until sufficient information gathered ...
Finish: [final answer]

Rules:
- Always Think before Acting
- One tool call per Act step
- If a tool errors, Think about why and try a different approach
- Use Finish as soon as you have enough — don't over-call tools
```

**ReAct production note**: Few-shot examples alone underperform standard prompting without fine-tuning. For production: fine-tune on ~3,000 think-act-observe examples. A fine-tuned 8B ReAct model outperforms a vanilla 62B standard model.

### Plan-and-Solve (before entering ReAct):
```
Before you begin, write a numbered plan:
Plan:
1. [step]
2. [step]
Then execute the plan using Think/Act/Observe.
```

### Reflexion (self-correction loop):
```
After completing your response, review it against:
[quality rubric]
Identify the weakest element and retry only that part.
Repeat until all criteria pass or [N] retries exhausted.
```

### Branch-Solve-Merge (multi-perspective):
- Spawn N parallel solver prompts with different temperatures or perspective constraints
- Merge agent: `"You have [N] independent analyses of the same problem. Synthesise them into the single best answer, preserving unique insights from each."`

---

## When Writing Workflow Task Prompts (LLM Pipelines)

Workflow tasks are deterministic processing units — not conversations. Each must:

- Have **single responsibility** — one task, one output type, no branching inside the prompt
- Define **explicit I/O schema** — input format and output format completely specified
- Be **self-contained** — works without any conversation history
- Specify **failure mode** — what to output on error or edge case

### Two-Step Ideation Pattern (proven to outperform one-shot):

```
Step 1: "Brainstorm [N] candidate [items] for [context]. Return a numbered list, no evaluation."

Step 2: "Given these candidates and this context, select the single best option.
         Justify in 2 sentences citing specific details from the context."
```

### Workflow Prompt Template:
```
<role>You are a [task-specific role]. Your only job is to [specific task].</role>

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
```

---

## Prompt Assembly Rules

### The Attention U-Curve (where to place what)

| Position | Attention level | What to put here |
|---|---|---|
| **Start** (primacy effect) | High | Role, task instruction, critical rules |
| **Middle** (Valley of Meh) | Low | Few-shot examples, supporting context — keep tight |
| **End** (recency effect) | Highest | The actual user query / task input |

**Assembly order for agent prompts:**
```
[1] Role + task instruction          ← primacy effect — use it
[2] Behavioural rules & constraints
[3] Tool definitions (if any)
[4] Few-shot examples (if any)       ← middle — unavoidable, keep them concise
[5] Retrieved context (RAG)          ← middle — top-k by relevance only
[6] Actual user query / task input   ← recency effect — maximum attention here
```

### Token Budget (treat like a spreadsheet)

| Component | Priority | Typical tokens |
|---|---|---|
| Role + instruction | Tier 1 — always include | 50–200 |
| Behavioural rules | Tier 1 — always include | 50–300 |
| Tool definitions | Tier 1 (if tools used) | 100–500 |
| Few-shot examples | Tier 2 — include if budget allows | 200–1000 |
| Retrieved context | Tier 2 — top-k by relevance | 500–3000 |
| Conversation history | Tier 3 — trim from oldest first | Variable |
| User query | Fixed — always last | Variable |

---

## Output Control Rules

### Preamble Management

| Preamble type | What it looks like | How to handle |
|---|---|---|
| **Structural boilerplate** | `"Here is your answer:"`, opening XML tags | Inception trick: pre-populate the assistant's opening words in the prompt |
| **Chain-of-thought** | Reasoning steps before the answer | Preserve deliberately; extract final answer via postprocessing |
| **Fluff** | Disclaimers, background, re-stating the question | `"Answer first, then any explanation"` format |

**Inception trick** — pre-populate the assistant turn:
```
...your full prompt...
Assistant: The answer is
```
Model continues from that start — inherits tone and direction.

**Answer-first format** — eliminates most fluff:
```
"Provide your answer in this format:
1. [Main answer — one sentence]
2. [Explanation or caveats, if needed]"
```

### Stop Sequences

Always define stop sequences for programmatic extraction — server-side stopping costs nothing after the stop:
```python
response = client.chat.completions.create(
    stop=["</answer>", "---END---", "\n\nHuman:"]
)
```

---

## Few-Shot Prompting Rules

- **Zero-shot first** — if the model succeeds, examples aren't worth the token cost
- **3–6 examples** — fewer risks anchoring; more than 10 rarely helps
- **Represent the full output distribution** — if you expect ratings 1–5, cover all five in examples
- **Randomise example ordering** — prevents the model from inferring ascending/descending trends
- **CoT examples**: reasoning MUST precede the answer in every example

---

## RAG Context Rules

1. Chunk for single meaning — one idea per snippet; multi-topic chunks defocus retrieval
2. Score before including — every snippet must earn its budget; irrelevant context degrades output
3. Use aside technique for retrieved content:
   ```
   // <context from [source], retrieved for [reason]>
   [content]
   // </context>
   ```
4. Task-specific summarisation — prompt the summariser with the downstream task in mind:
   `"Summarise focusing especially on [details relevant to task], even if they appear minor"`

---

## Quality Checklist — Before Finalising Any Prompt

**Content:**
- [ ] Role is specific — activates the right training distribution region
- [ ] Scope boundaries explicit — what agent does AND does not do
- [ ] Output format completely specified — no ambiguity
- [ ] Uncertainty handling explicitly instructed
- [ ] Critical constraints in first half of the prompt

**Structure:**
- [ ] Context precedes instructions
- [ ] Important content at start or end (not buried in middle)
- [ ] Injected content delimited and inert (no accidental generation triggers)
- [ ] Token budget tracked — fits within context window

**Agent-specific:**
- [ ] Tool definitions self-documenting (name = what, description = when)
- [ ] Argument hallucination mitigated (known values removed, defaults provided)
- [ ] Dangerous tool calls gated at application layer, not prompt layer
- [ ] Reasoning pattern matches task (CoT for logic, ReAct for tool use)

**Output:**
- [ ] Stop sequences defined for programmatic extraction
- [ ] Preamble type anticipated and handled
- [ ] Output schema enforced via tool-call forcing or JSON mode

---

## Anti-Patterns — Fix These Immediately

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| `"You are a helpful assistant"` | Broadest distribution, lowest quality | Specify domain, expertise, specific objective |
| Instructions after context | Model interprets instructions through wrong lens | Move instructions before the governed content |
| `"Be concise"` | Undefined | `"Respond in exactly 2 sentences"` |
| `"Don't hallucinate"` | Prohibition without positive instruction | `"If uncertain, say 'I don't know' rather than guessing"` |
| Free-text parsing of structured output | Breaks on any format variation | Tool-call forcing or JSON mode |
| `"Check with user before action"` in tool definition | Fails ~5% of the time stochastically | Application-layer interception gate |
| All few-shot examples with same output value | Anchoring bias | Distribute across full output range |
| No stop sequences | Pays compute for postscript fluff | Define stop sequences for all programmatic extraction |
| Dynamic content without delimiters | Model confuses content with instructions | Always delimit with XML tags or backtick fences |
| User query buried in the middle | Valley of Meh — lowest attention | Always put the actual task at the very end |
