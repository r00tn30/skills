# Prompt Writer — Quick Reference Cheat Sheet
## Rapid lookup for all key concepts
### Source: Prompt Engineering for LLMs – Berryman & Ziegler (O'Reilly)

---

## THE TWO-SENTENCE RULE (internalize this first)

> **1.** LLMs are text completion engines that mimic their training data.
> **2.** Write prompts that look like the beginning of a document the model was trained to complete correctly.

Everything else is application of these two sentences.

---

## PROMPT ANATOMY — UNIVERSAL ORDER

```
┌─────────────────────────────────────────────────────────────┐
│  1. ROLE + TASK INSTRUCTION          ← primacy effect       │
│  2. BEHAVIOURAL RULES & CONSTRAINTS                         │
│  3. TOOL DEFINITIONS (if any)                               │
│  4. FEW-SHOT EXAMPLES (if any)       ← Valley of Meh        │
│  5. RETRIEVED CONTEXT (RAG)          ← Valley of Meh        │
│  6. CONVERSATION HISTORY                                     │
│  7. USER QUERY / TASK INPUT          ← recency effect ★    │
└─────────────────────────────────────────────────────────────┘
★ Always last. Highest attention position. Never bury this in the middle.
```

---

## ATTENTION ZONES

| Zone | Effect | Use for |
|---|---|---|
| **Start** | Primacy — high attention | Role, rules, critical constraints |
| **Middle** | Valley of Meh — LOWEST attention | Examples, supporting context (unavoidable — keep tight) |
| **End** | Recency — HIGHEST attention | The actual question / task input |

---

## TEMPERATURE QUICK GUIDE

| Task type | Temperature |
|---|---|
| Structured extraction, classification, tool selection | `0` |
| Factual Q&A, precise answers | `0.1–0.3` |
| General conversation, explanations | `0.7` |
| Creative writing, brainstorming | `0.9–1.2` |
| Parallel completions `n > 1` | `temperature = sqrt(n) / 10` |

---

## DOCUMENT ARCHETYPES

| Archetype | Best for | Key formatting rule |
|---|---|---|
| **Advice Conversation** | Chat, support, tutoring, agent dialogue | Conversational prose; ChatML; inception trick |
| **Analytic Report** | Research summaries, post-mortems, briefs | Markdown headers; embed ToC in prompt to enforce structure |
| **Structured Document** | Extraction, pipelines, tool outputs | Schema IS the prompt; XML/JSON; tool-call forcing for output |

---

## CONTENT TYPES

| Type | Provides | Key rule |
|---|---|---|
| Static explicit | Rules and instructions | Positive phrasing; specific not vague; order critical rules early |
| Static implicit (few-shot) | Examples of desired output | Representative distribution; randomise order; CoT: reasoning before answer |
| Dynamic retrieved (RAG) | External / private knowledge | One idea per chunk; score before including; aside technique for injection |
| Dynamic compressed | Long context condensed | Task-specific summarisation; hierarchical for very long docs |

---

## FEW-SHOT RULES (5 non-negotiables)

1. **Representative** — cover the full output distribution, not cherry-picked
2. **Randomised order** — prevents the model inferring ascending/descending trends
3. **3–6 examples** — sweet spot between anchoring risk and token cost
4. **CoT: reasoning before answer** — always; never show answer-first in a CoT example
5. **Audit for spurious patterns** — length progression, value gradient, sentiment ordering

---

## TOOL DESIGN (5 non-negotiables)

1. **≤10 tools** active at once — more = confusion and hallucination
2. **Self-documenting names** — `fetchUserOrderHistory` not `getData`
3. **≤3 arguments** — use `enum` for bounded values; `default` for optional args
4. **Remove known values** — if the app knows `user_id`, never put it in the tool definition
5. **Dangerous tools: gate in application layer** — never rely on `"check with user first"` in a prompt

---

## REASONING FRAMEWORK SELECTOR

```
Multi-step logical / mathematical task?
  YES → Chain-of-Thought (CoT)
        Zero-shot: append "Think step by step before answering"
        Few-shot:  examples with reasoning BEFORE answer

Task requires tool calls interleaved with reasoning?
  YES → ReAct  (Think → Act → Observe loop)
        Production note: fine-tune on ~3k examples for real gains
        Fine-tuned 8B ReAct > vanilla 62B standard prompting

Complex task with benefit from upfront planning?
  YES → Plan-and-Solve + ReAct
        Write numbered plan first, then execute with ReAct loop

Output quality needs iterative verification + correction?
  YES → Reflexion
        Review against rubric; retry weakest element; repeat until PASS

Multiple perspectives improve the answer?
  YES → Branch-Solve-Merge
        N parallel solvers (different roles/temps) + 1 merge agent

Using a reasoning model (o1, o3, Claude extended thinking)?
  YES → Skip explicit CoT — model reasons internally
        State goals + constraints clearly; let the model reason on its own
```

---

## OUTPUT CONTROL QUICK FIXES

| Problem | Solution |
|---|---|
| Model generates preamble fluff | Answer-first: `"1. [main answer] 2. [explanation]"` |
| Need to control the response opening | Inception trick: pre-populate the assistant's first words |
| Need clean programmatic extraction | Stop sequences: `stop=["</answer>", "---END---"]` |
| Structured output parsing is fragile | Tool-call forcing (`tool_choice="required"`) or JSON mode |
| Model generates beyond what's needed | Stop sequences — server-side; zero cost after the stop |

---

## PREAMBLE TYPES AT A GLANCE

| Type | Looks like | Handle with |
|---|---|---|
| Structural boilerplate | `"Here is your answer:"`, opening XML tags | Inception trick — pre-populate it in the prompt |
| Chain-of-thought | Reasoning steps before the answer | Preserve; postprocess to extract final answer if needed |
| Fluff | Disclaimers, background, restating the question | Answer-first format + explicit instructions |

---

## RAG QUICK RULES

| Rule | Detail |
|---|---|
| Same model for index + query | Different embedding models = incompatible vector spaces |
| One main idea per chunk | Multi-topic chunks defocus retrieval |
| Aside technique for injection | `// <context from X>` … `// </context>` |
| Task-specific summarisation | Tell the summariser what details matter to the downstream task |
| Hybrid retrieval beats either alone | Lexical (BM25) + neural (embeddings) re-ranking |

---

## ASSEMBLY ALGORITHM SELECTOR

| Situation | Algorithm |
|---|---|
| Fixed system prompt + history truncation only | Minimal Crafter |
| Many optional retrieved snippets | Additive Greedy |
| Most elements required, few need pruning | Subtractive Greedy |
| Complex dependency graph between elements | Subtractive Greedy |

---

## TOKEN QUICK MATH

```
1 token  ≈  0.75 words  ≈  4 characters  (English)
1,000 words   ≈  1,333 tokens
1 paragraph   ≈  80–150 tokens
1 sentence    ≈  15–30 tokens
1 simple tool definition  ≈  50–150 tokens
1 few-shot example        ≈  50–200 tokens
```

---

## EVALUATION QUICK GUIDE

| Stage | Method | When to use |
|---|---|---|
| Early dev | Example suite (5–20 inputs + git diff) | Any new prompt change |
| Pre-ship | Evaluation harness (100s of examples + auto scoring) | Before production rollout |
| Tool-calling tasks | Gold standard: correct tool + valid args | Highest-leverage partial match |
| Executable output (code, SQL) | Functional testing: run test suite | When external oracle exists |
| Free-form output | LLM-as-judge with SOMA framework | When above methods don't apply |
| Post-ship | A/B testing with implicit signals | Production monitoring |

**SOMA = Specific questions + Ordinal scale (5-pt asymmetric) + Multi-aspect coverage**
State rubric BEFORE showing the completion to be judged — model cannot backtrack.

---

## WORKFLOW TASK DESIGN RULES

| Rule | Detail |
|---|---|
| Single responsibility | One task, one output type, no branching inside the prompt |
| Explicit I/O schema | Define input format AND output format completely |
| Self-contained | Works without conversation history |
| Failure mode specified | What to return on error / edge case |
| Two-step ideation | Step 1: brainstorm (diverge). Step 2: select + justify (converge). Outperforms one-shot. |
| Right-sized model | Cheap/fast for extraction; large for reasoning; specialist for creative |

---

## ANTI-PATTERNS → FIXES

| Anti-pattern | Fix |
|---|---|
| `"You are a helpful assistant"` | Specify domain, expertise level, concrete objective |
| Instructions placed after the content they govern | Move instructions BEFORE the governed content |
| `"Be concise"` | `"Respond in exactly 2 sentences"` |
| `"Don't hallucinate"` | `"If uncertain, say 'I don't know' rather than guessing"` |
| Free-text parsing of structured output | Tool-call forcing or JSON mode |
| `"Check with user before dangerous action"` in tool def | Application-layer interception gate |
| All few-shot examples with same output value | Distribute across full output range |
| No stop sequences for programmatic extraction | Always define stop sequences |
| Dynamic content injected without delimiters | XML tags or triple-backtick fences |
| User query buried in the middle of the prompt | Always last — recency = highest attention |
| Explicit CoT with reasoning model (o1, o3) | Remove CoT scaffolding; state goals + constraints only |

---

## MULTI-AGENT ARCHITECTURE HIERARCHY

```
Prefer simpler → only add complexity when evaluation data demands it

1. No LLM — use rule-based code or classical ML if it solves the problem
       ↓
2. LLM tasks + deterministic graph (pipeline or DAG)   ← most maintainable
       ↓
3. LLM-driven routing (add only when task order is genuinely data-dependent)
       ↓
4. Agent of agents / stateful agents / multi-role delegation
   (most flexible, least stable, hardest to debug — frontier territory)
```

---

## LOGPROBS QUICK REFERENCE

```
logprob = ln(probability)       probability = exp(logprob)

logprob  0.0  →  probability 1.00  =  certain
logprob -0.1  →  probability 0.90  =  very likely
logprob -0.7  →  probability 0.50  =  coin flip
logprob -2.3  →  probability 0.10  =  unlikely
logprob -5.0  →  probability 0.007 =  very unlikely
logprob -13   →  typo or deeply unexpected token

Single-digit negatives: normal
Double-digit negatives: genuine anomaly — typo, unexpected content, or confabulation signal

Quality scoring:
  Simple average:           (sum of logprobs) / n
  Early-token probability:  avg(exp(logprob)) for first k tokens  ← best predictor of overall quality
```

---

## CHAPTER-TO-TOPIC INDEX

| Chapter | Key topic |
|---|---|
| Ch 1 | LLM history; 5 levels of prompt engineering sophistication |
| Ch 2 | Tokens; autoregressive generation; temperature; hallucination |
| Ch 3 | RLHF; ChatML; system/user/assistant roles; alignment tax |
| Ch 4 | LLM application loop; feedforward pass; RAG intro; evaluation intro |
| Ch 5 | Static vs dynamic content; few-shot prompting; RAG retrieval methods; summarisation |
| Ch 6 | Prompt assembly; Valley of Meh; document archetypes; elastic snippets; knapsack algorithm |
| Ch 7 | Preamble types; stop sequences; logprobs; model selection; fine-tuning (LoRA, soft prompting) |
| Ch 8 | Tool calling; CoT; ReAct; Reflexion; Branch-Solve-Merge; agent context management |
| Ch 9 | LLM workflows; pipeline/DAG/cyclic topologies; AutoGen; CrewAI; stateful agents |
| Ch 10 | Evaluation harnesses; SOMA framework; A/B testing; 5 online metric types |
| Ch 11 | Multimodality; reasoning models; benchmark saturation; the two core book lessons |
