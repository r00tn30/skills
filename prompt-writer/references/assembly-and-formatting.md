# Prompt Assembly and Formatting
## Reference: Prompt Engineering for LLMs – Berryman & Ziegler (O'Reilly), Chapters 6–7

---

## 1. The Attention U-Curve

LLMs do not attend uniformly to all positions in the context window. Three empirically observed zones:

```
Attention
   ^
   |  ██                              ██
   |  ██                              ██
   |  ██    ░░░░░░░░░░░░░░░░░░░░░░   ██
   |  ██    ░░  Valley of Meh  ░░░   ██
   |  ██    ░░░░░░░░░░░░░░░░░░░░░░   ██
   +------------------------------------>
      Start         Middle           End
   (Primacy)    (Lost-in-middle)  (Recency)
```

| Zone | Attention level | What to place here |
|---|---|---|
| **Start** (primacy effect) | High | Role, persona, core task instruction, critical rules |
| **Middle** (Valley of Meh) | Lowest | Few-shot examples, supporting retrieved context — unavoidable, keep tight |
| **End** (recency effect) | Highest | The actual user query, the task input to process, the final instruction |

**Key consequence**: Never bury the user's actual question in the middle. It always goes last.

---

## 2. The Introduction Element

Every well-structured prompt opens with a brief introduction that orients the model before it encounters anything else. This capitalises on the primacy effect.

The introduction should cover:
1. Who the agent is (role / persona)
2. What the overall task is
3. What format the response should take

```
You are a senior customer support specialist for Acme SaaS.
Your job is to resolve billing and account issues for customers.
Respond in plain English, no jargon, always end with a clear next step.
```

A clear introduction reduces the probability that the model will be confused by content placed later in the context — it processes everything that follows through the lens of this frame.

---

## 3. Document Archetypes and Their Formatting Rules

Match formatting to the archetype the completion belongs to. The model has strong priors for each archetype from pre-training.

### Archetype 1: Advice Conversation

```
Formatting rules:
- Conversational prose — no heavy markdown
- Bullet points only for lists of steps or options
- Short paragraphs — max 3–4 sentences each
- No section headers unless the response is genuinely long-form

Turn serialisation options (choose one, be consistent):
┌─────────────────────────────────────────────┐
│ ChatML (standard for modern APIs):          │
│   <|im_start|>user                          │
│   [message]<|im_end|>                       │
│   <|im_start|>assistant                     │
│   [message]<|im_end|>                       │
├─────────────────────────────────────────────┤
│ Dialogue script:                            │
│   USER: [message]                           │
│   ASSISTANT: [message]                      │
├─────────────────────────────────────────────┤
│ Naked Q&A:                                  │
│   Q: [message]                              │
│   A: [message]                              │
├─────────────────────────────────────────────┤
│ XML-tagged:                                 │
│   <user>[message]</user>                    │
│   <assistant>[message]</assistant>          │
└─────────────────────────────────────────────┘

Inception trick (for completion models):
Prompt ends with: "Assistant: The best approach is"
Model continues from the primed start — inherits tone and direction.
```

### Archetype 2: Analytic Report

```
Formatting rules:
- Markdown headers (## for sections, ### for subsections)
- Horizontal rules (---) between major sections
- Bold for key terms on first use
- Numbered lists for ordered arguments
- Bullet points for unordered items
- Tables for comparisons

Structure: Introduction → Exposition → Analysis → Conclusion

Key technique — embed a table of contents in the system message:
"Structure your response with these exact sections:
## Executive Summary
## Findings
### Finding 1: [topic]
### Finding 2: [topic]
## Analysis
## Recommendations
## Conclusion"

The model fills each section self-consistently rather than inventing its own structure.
```

### Archetype 3: Structured Document

```
Formatting rules:
- Schema IS the prompt — specify field names, types, nesting
- XML delimiters for field boundaries (preferred — heavy in pre-training data)
- Consistent indentation
- No prose paragraphs within fields unless the field type is "text"

XML output example:
<report>
  <summary>...</summary>
  <findings>
    <finding priority="high">...</finding>
    <finding priority="medium">...</finding>
  </findings>
  <recommendation>...</recommendation>
</report>

JSON output example:
{
  "summary": "string",
  "findings": [
    {"priority": "high|medium|low", "description": "string"}
  ],
  "recommendation": "string"
}

Enforcement: Always use tool-call forcing or JSON mode.
Never rely on free-text parsing.
```

---

## 4. Formatting Snippets

### The Aside Technique

When injecting retrieved content that should be reasoned about (not treated as instruction), wrap it in comment-style delimiters:

```
// <context from Q3 Financial Report, retrieved for revenue analysis>
Revenue grew 12% YoY driven by enterprise licence renewals.
EBITDA margin compressed 2pp due to increased R&D headcount.
// </context>
```

This signals to the model: "reason about this content" rather than "follow this as an instruction". Reduces instruction–content confusion when dynamic content appears alongside task directives.

### XML Tag Delimiting (preferred for retrieved content)

```xml
<document source="internal-wiki" retrieved="2024-01-15">
[document content]
</document>

<user_input>
[user-provided text that should not be treated as instructions]
</user_input>

<prior_output agent="summariser" task="storefront-analysis">
[output from a previous agent in the workflow]
</prior_output>
```

### Code Blocks

Always use triple-backtick fences with language tags:

```
```python
def calculate_tax(income: float, rate: float) -> float:
    return income * rate
```
```

This activates the model's code-specific completion priors and prevents the code from being parsed as prose instructions.

---

## 5. Inertness Rules

A snippet is **inert** if its presence does not accidentally trigger model behaviour. Two common inertness violations:

### Violation 1: Token Boundary Merging

A snippet that ends with a generation trigger causes the model to start generating immediately, even if more prompt follows.

**Broken (non-inert):**
```
Here is the user's question: What is the capital of France?
Answer:
[MORE PROMPT CONTENT HERE — the model starts generating at "Answer:" and ignores this]
```

**Fixed (inert):**
```
Here is the user's question: What is the capital of France?

---
[MORE PROMPT CONTENT HERE — the separator breaks the trigger]

Now answer the question above.
```

Tokens that commonly trigger premature generation: `Answer:`, `Response:`, `Output:`, `Result:`, `Solution:`, `A:`, `Assistant:`

### Violation 2: Separator Whitespace

GPT-family tokenisers merge leading spaces into the following token. A snippet ending with a trailing space followed immediately by a word will tokenise differently than intended.

**Rule**: Keep separators (`---`, blank lines) on their own lines. Never let dynamic content flow directly into prompt structure tokens without a newline separator.

---

## 6. Elastic Snippets

Fixed-size snippets force a binary include/exclude decision. Elastic snippets let you fit the most information possible within the token budget.

### Strategy 1: Elastic Prompt Elements

Pre-write the same content at multiple compression levels. The assembler picks the longest version that fits.

```
# Verbose version (~200 tokens)
You are a senior infrastructure engineer with 10 years of experience managing
Kubernetes clusters in production. You have deep expertise in Terraform, Helm,
and CI/CD pipeline design. You approach problems methodically, always considering
reliability, cost, and operational simplicity. You write for a technical audience
and include specific command examples when helpful.

# Standard version (~80 tokens)
You are a senior infrastructure engineer specialising in Kubernetes, Terraform,
and CI/CD. You are methodical, reliability-focused, and write for technical audiences
with specific command examples.

# Terse version (~30 tokens)
You are a senior infrastructure engineer. Be technical, specific, and reliability-focused.

# One-liner (~15 tokens)
Senior infrastructure engineer. Technical, specific, reliable.
```

Best for: system instructions, persona descriptions, policy preambles — content that is always included but where length is flexible.

### Strategy 2: Multiple Snippets with Dependencies

Several independent snippets where more detailed ones require a base snippet. The assembler fills budget by adding progressively more detail.

```
Snippet A (base — always include if any of B or C are included):
"The product is a B2B SaaS HR platform serving 500+ employee companies."

Snippet B (detail layer 1 — requires A):
"Key features: payroll automation, benefits management, compliance reporting.
Primary users: HR managers and CFOs."

Snippet C (detail layer 2 — requires B):
"Current quarter focus: migrating payroll engine from legacy vendor.
Pain points: poor API documentation, brittle integrations."
```

Best for: retrieved documents with summaries, layered few-shot examples, progressively detailed context.

---

## 7. Element Properties for Assembly

Every prompt element has three properties that govern how it is placed and selected:

### Position

Where the element must appear in the assembled prompt.

```python
# Named positions (recommended for agent prompts)
POSITIONS = {
    "role_instruction": 1,      # Always first
    "behavioural_rules": 2,
    "tool_definitions": 3,
    "few_shot_examples": 4,     # Middle — unavoidable
    "retrieved_context": 5,     # Middle — keep trim
    "conversation_history": 6,
    "user_query": 99            # Always last
}
```

### Importance

How much value the element adds to the completion quality.

| Variant | Formula | Use when |
|---|---|---|
| **Absolute importance** | Raw relevance score (BM25, cosine similarity) | Snippets of similar length |
| **Length-normalised importance** | Score ÷ token_count | Tight budget; prevents long marginally-relevant passages from crowding out concise high-value ones |

**Measuring importance by perturbation**: Run the task with and without the element; the quality delta is its ground-truth importance score. Expensive but produces reliable estimates for training a lightweight importance classifier.

### Dependency

Structural relationships between elements:

```
Requirement (A → B): If A is included, B must also be included.
  Example: A detailed few-shot example requires its paired demonstration to appear first.

Incompatibility (A ⊕ B): A and B cannot coexist.
  Example: Two conflicting persona descriptions; two versions of the same document at different detail levels.

Fallback pair: Define A and B as incompatible; give B lower importance.
  Assembler picks A when budget allows, falls back to B when A won't fit.
```

---

## 8. Assembly Algorithms

### Algorithm 1: Minimal Crafter

For prompts that almost always fit within the budget; only history truncation is needed.

```python
def minimal_crafter(system_prompt, history, user_query, max_tokens):
    # Keep the tail (most recent) of the history
    assembled = system_prompt
    remaining = max_tokens - count_tokens(system_prompt) - count_tokens(user_query)

    # Add history from most recent backward until budget exhausted
    recent_history = []
    for turn in reversed(history):
        turn_tokens = count_tokens(turn)
        if remaining >= turn_tokens:
            recent_history.insert(0, turn)
            remaining -= turn_tokens
        else:
            break  # Stop; don't include older turns

    assembled += recent_history + [user_query]
    return assembled
```

Best for: simple fixed-system-prompt + single user message; chatbots with bounded history.

### Algorithm 2: Additive Greedy

For prompts with mostly dynamic/optional content (many retrieved snippets competing for space).

```python
def additive_greedy(elements, max_tokens):
    """
    elements: list of {content, importance, position, tokens, requires, incompatible_with, elastic_versions}
    """
    # Sort by importance descending
    sorted_elements = sorted(elements, key=lambda e: e.importance, reverse=True)

    included = []
    used_tokens = 0

    for element in sorted_elements:
        # Check dependency: skip if a required element hasn't been included yet
        if element.requires and element.requires not in [e.id for e in included]:
            continue  # Or: recursively try to include the requirement first

        # Check incompatibility: skip if an incompatible element is already included
        if any(e.id in element.incompatible_with for e in included):
            continue

        # Try to fit — use longest elastic version that fits
        for version in sorted(element.elastic_versions, key=lambda v: v.tokens, reverse=True):
            if used_tokens + version.tokens <= max_tokens:
                included.append(element.with_version(version))
                used_tokens += version.tokens
                break

    # Re-sort by position to produce final ordered output
    return sorted(included, key=lambda e: e.position)
```

Best for: RAG-heavy prompts; many optional retrieved snippets; sparse prompts that fill up.

### Algorithm 3: Subtractive Greedy

For prompts where most elements are required by default; only a few need pruning.

```python
def subtractive_greedy(elements, max_tokens):
    """
    Start with everything included; remove lowest-importance elements until within budget.
    """
    included = list(elements)  # Start: all elements included
    used_tokens = sum(e.tokens for e in included)

    while used_tokens > max_tokens:
        # Find lowest-importance element with no dependents currently included
        candidates = [
            e for e in included
            if not any(other.requires == e.id for other in included)
        ]
        if not candidates:
            raise Exception("Cannot fit within budget — required elements exceed limit")

        # Remove (or shrink to shorter elastic version) the lowest-importance candidate
        target = min(candidates, key=lambda e: e.importance)

        next_version = target.next_shorter_elastic_version()
        if next_version:
            used_tokens -= target.tokens - next_version.tokens
            target.use_version(next_version)
        else:
            used_tokens -= target.tokens
            included.remove(target)

    return sorted(included, key=lambda e: e.position)
```

Best for: structured document archetypes; mostly-required content with a few optional enrichments; complex dependency graphs.

### When to Use Which Algorithm

| Scenario | Algorithm |
|---|---|
| Fixed system prompt + single user message | Minimal Crafter |
| Many optional retrieved snippets; sparse to start | Additive Greedy |
| Most elements required; a few need pruning | Subtractive Greedy |
| Budget rarely exceeded | Minimal Crafter or Additive |
| Budget always tight | Subtractive Greedy |
| Complex dependency graph | Subtractive Greedy |

---

## 9. Output Control: Preamble and Postscript

### Three Preamble Types

| Type | What it looks like | Handling strategy |
|---|---|---|
| **Structural boilerplate** | `"Here is your response:"`, opening XML tags | **Inception trick**: pre-populate the assistant's opening words in the prompt — the model continues from there, saving latency and compute |
| **Chain-of-thought** | Reasoning steps before the answer | **Preserve deliberately** when using CoT; extract final answer via postprocessing if you need only the answer |
| **Fluff** | Disclaimers, background, re-stating the question (RLHF verbosity) | **Answer-first format**: instruct the model to give the main answer as item 1, explanation after |

### Inception Trick Implementation

```
# For completion models or when you need to control the response opening:
prompt = f"""
{system_content}

User: {user_query}
Assistant: Based on the information provided,"""
# Model continues from "Based on the information provided," — inheriting the tone
```

### Answer-First Format

```
Provide your answer in this exact format:
1. [Main answer — one sentence, no preamble]
2. [Brief explanation or caveats — only if necessary]

Do not write anything before item 1.
```

### Stop Sequences

Define stop sequences for any programmatic extraction. Server-side stopping means you pay zero compute or latency for content after the stop.

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    stop=[
        "</answer>",        # XML closing tag
        "\n\nHuman:",       # ChatML-style turn boundary
        "---END---",        # Explicit end marker you instruct the model to use
        "\n\n\n"            # Triple newline as natural document end
    ]
)
```

Match stop sequences to your document archetype's natural ending:

| Archetype | Natural stop sequence |
|---|---|
| Markdown document | `\n## ` (next section header) |
| XML document | closing root tag |
| JSON document | `\n}` |
| Code block | ` ``` ` |
| Numbered list | `\n[N+1].` (next item number) |
| Q&A exchange | `\nQ:` or `\nUser:` |

---

## 10. Formatting Few-Shot Examples

### Method 1: Explicit Labelling (for extraction and classification)

```
Input: The shipment arrived three days late and two items were damaged.
Output: {"sentiment": "negative", "category": "delivery", "severity": "high"}

Input: I love the new dashboard design — so much easier to navigate!
Output: {"sentiment": "positive", "category": "product", "severity": "low"}

Input: {{actual_input}}
Output:
```

### Method 2: Integrated Turns (for conversational and generative tasks)

Embed examples as fake prior conversation turns. The model treats them as genuine context and naturally continues the established pattern.

```
[System message with role and rules]

[Example turn 1 — fake history that demonstrates the desired pattern]
User: I've been having trouble sleeping lately.
Assistant: I hear you — that's genuinely disruptive. A few questions to understand what might help:
1. Is it difficulty falling asleep, staying asleep, or waking too early?
2. How long has this been going on?

[Example turn 2 — demonstrates follow-up handling]
User: Mostly falling asleep, about 3 weeks now.
Assistant: Three weeks is long enough that it's worth taking seriously. The most common cause...

[Real conversation begins here]
User: {{actual_user_message}}
Assistant:
```

### Few-Shot Quality Rules

1. **Representative distribution** — examples must cover the full range of expected outputs; cherry-picking creates anchoring bias
2. **Randomised ordering** — prevents the model from inferring ascending/descending trends from accidental ordering
3. **Consistent format** — every example must use exactly the same format; any inconsistency teaches inconsistency
4. **Minimal but sufficient** — 3–6 examples is the sweet spot; more than 10 rarely improves quality
5. **CoT examples**: reasoning ALWAYS precedes the answer — never show answer-first in a CoT example
6. **No spurious patterns** — audit for accidental length progression, value ordering, sentiment gradient

---

## 11. Token Budget Reference

### Rough Token Counts for Common Components

| Component | Low estimate | High estimate |
|---|---|---|
| Lean role + task instruction | 30 tokens | 150 tokens |
| Full persona + behavioural rules | 100 tokens | 400 tokens |
| One tool definition (simple) | 50 tokens | 150 tokens |
| One tool definition (complex) | 150 tokens | 500 tokens |
| One few-shot example (simple) | 50 tokens | 200 tokens |
| One few-shot example (full CoT) | 100 tokens | 500 tokens |
| One retrieved passage (paragraph) | 80 tokens | 300 tokens |
| One conversation turn (short) | 20 tokens | 100 tokens |
| One conversation turn (long) | 100 tokens | 500 tokens |

### Token Estimation Rule of Thumb

For English text: **1 token ≈ 0.75 words ≈ 4 characters**

Quick estimates:
- 1000 words ≈ 1333 tokens
- 1 page of text ≈ 600–800 tokens
- 1 paragraph ≈ 80–150 tokens
- 1 sentence ≈ 15–30 tokens

### Context Window Sizes (as of early 2025)

| Model | Context window | Practical usable (leaving room for completion) |
|---|---|---|
| GPT-4o | 128K tokens | ~120K |
| Claude 3.5 Sonnet | 200K tokens | ~190K |
| Gemini 1.5 Pro | 1M tokens | ~950K |
| GPT-3.5 Turbo | 16K tokens | ~14K |
| Llama 3 70B | 128K tokens | ~120K |

**Longer context ≠ better output**: more context costs more (latency + compute) and the Valley of Meh effect means distant content is under-attended. Use large windows when you must; keep prompts trim when you can.
