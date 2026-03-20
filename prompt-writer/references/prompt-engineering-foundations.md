# Prompt Engineering Foundations
## Reference: Prompt Engineering for LLMs (Berryman & Ziegler, O'Reilly)

---

## 1. How LLMs Actually Work — The Mechanical Reality

Understanding these mechanics is what separates expert prompt engineers from guessers.

### LLMs Are Document Completion Engines

Every interaction — chat, tool call, code generation, extraction — is the model completing a document. The model's only job: find the statistically most likely continuation of the text you gave it.

**Consequence for prompting**: The framing of the prompt determines which region of the training distribution the model draws from. A prompt that looks like a tech support forum → support-like completions. A prompt that looks like academic writing → academic completions. A prompt that looks like a coding tutorial → code-like completions.

### Tokens, Not Words

LLMs see text as **tokens** — sub-word chunks from a deterministic tokeniser. Key implications:

- Typos shatter tokens → force the model to reason about unfamiliar sub-token sequences → degraded output
- `For` (sentence start) and `for` (mid-sentence) are different token IDs — capitalisation has non-trivial effects
- Character-level tasks (reversing letters, counting characters, anagrams) are disproportionately hard
- Token budget estimation: ~0.75 words per token = ~1333 tokens per 1000 words for English

### Autoregressive Generation

The model generates one token at a time. Each new token is appended to the prompt and the model runs again. This means:
- **No revision** — once a token is emitted, it's locked in as context for all future tokens
- **Error compounding** — a wrong early token steers all subsequent tokens in the wrong direction
- **Repetition loops** — patterns reinforce themselves; once a pattern appears, the model finds it more likely to continue it
- **Front-load constraints** — position critical rules early; late rules may arrive after the model has already committed to a direction

### Temperature and Sampling

The model computes a probability distribution over all vocabulary tokens at each step. Temperature controls how this distribution is used:

| Temperature | Effect | Use case |
|---|---|---|
| 0 | Greedy — always picks highest probability token. Deterministic. | Code, extraction, classification, structured output |
| 0.1–0.4 | Sharpened — conservative, low randomness | Precise factual answers, tool selection |
| 0.7–1.0 | Matches training distribution | General conversation, explanations |
| > 1.0 | Flattened — more random, more surprising | Creative tasks; errors compound at high values |

**Temperature 0 for agents doing structured extraction or tool selection** — determinism and reliability matter more than diversity.

### Hallucination Is Structural, Not a Bug

LLMs have no grounding mechanism — they complete documents statistically. When asked about something they don't know, they generate a *plausible-looking* completion based on patterns. A hallucinated URL looks like a real URL because the model has seen millions of real URLs.

**Mitigation strategies (in order of effectiveness)**:
1. RAG — inject factual content from external sources directly into the prompt
2. RLHF alignment — trained models learn to say "I don't know" (reinforce this via explicit instruction)
3. Tool use — give the model tools to look up information rather than relying on parametric memory
4. Logprob monitoring — low confidence on early tokens signals likely confabulation

---

## 2. RLHF and Chat Models

### What RLHF Does

RLHF (Reinforcement Learning from Human Feedback) trains base LLMs into **HHH**-aligned assistants:
- **Helpful** — actually addresses the user's request
- **Honest** — expresses uncertainty rather than confabulating
- **Harmless** — refuses or redirects harmful requests

Three stages: Supervised Fine-Tuning (SFT) → Reward Model → RL fine-tuning.

The alignment tax: RLHF optimises for HHH, not raw intelligence. RLHF models can be less capable at complex reasoning than base models. Mitigate with explicit CoT instructions.

### ChatML and System Messages

Chat models use ChatML — a structured transcript format:
```
<|im_start|>system
[System message content — the prompt engineer's primary control surface]
<|im_end|>
<|im_start|>user
[User message]
<|im_end|>
<|im_start|>assistant
```

The API hides ChatML behind structured JSON (`{"role": "system", "content": "..."}`) and handles serialisation. This creates the prompt injection barrier — users cannot inject `<|im_start|>` tokens through the API.

**System message = primary control surface for agent behaviour.** RLHF models are specifically trained to weight system message directives above user input. All agent rules, persona, tool policy, and format specifications belong in the system message.

---

## 3. The LLM Application Loop

Every LLM application is a transformation loop:
```
User problem domain
    ↓ (feedforward pass)
LLM text domain → completion
    ↓ (transform back)
Action / answer in user domain
```

### The Feedforward Pass (4 steps)

1. **Context retrieval** — gather raw text from conversation history, user profile, external APIs, document stores, search results
2. **Snippetizing** — chunk into independently meaningful passages; convert non-text to text (voice → transcript, JSON → prose)
3. **Scoring and prioritising** — assign relevance scores (continuous) and priority tiers (integer); Tier 1 must fill before Tier 2
4. **Prompt assembly** — fit everything within token budget, in the right order, in a training-data-like format

### Four Feedforward Criteria

A prompt must satisfy all four simultaneously:
1. Closely resembles training data (Little Red Riding Hood principle)
2. Contains all information relevant to the problem
3. Leads the model to generate a useful completion
4. Has a natural stopping point — generation must terminate cleanly

---

## 4. Document Archetypes

Before writing a prompt, decide which archetype the output belongs to. The archetype determines formatting, structure, and serialisation approach.

### Archetype 1: Advice Conversation

Multi-turn Q&A exchange. The model plays a knowledgeable interlocutor.
- **Best for**: customer support, tutoring, exploratory research assistants, agent dialogue
- **Serialisation**: ChatML (standard for APIs); or dialogue script (`USER: / ASSISTANT:`), naked Q&A (`Q: / A:`), XML-tagged
- **Inception trick**: Pre-populate the last assistant turn to prime tone and direction
- **Format**: Minimal — conversational prose; bullet points only for lists

### Archetype 2: Analytic Report

Structured document with predictable section hierarchy: Introduction → Exposition → Analysis → Conclusion.
- **Best for**: research summaries, incident post-mortems, investment briefs, audit reports
- **Key technique**: Embed a Markdown table of contents in the system prompt specifying exact section headings — the model fills each section self-consistently
- **Format**: Markdown headers (`##`, `###`), bold for key terms, numbered lists for ordered arguments

### Archetype 3: Structured Document

Schema-bound output: JSON, XML, YAML, code, or any formally delimited format.
- **Best for**: data extraction pipelines, API response generation, tool-calling outputs, workflow task outputs
- **Key technique**: The schema IS the prompt — specify field names, types, and nesting; model fills the slots
- **Format**: XML delimiters outperform ad-hoc separators because XML is heavily represented in pre-training data
- **Enforcement**: Use tool-call forcing or JSON mode — never rely on free-text parsing

---

## 5. Assembly Algorithms

### The Knapsack Framing

Prompt assembly is formally a 0-1 knapsack problem:
- Items = prompt elements, each with importance (value) and token count (weight)
- Capacity = token budget
- Hard constraints = dependency and incompatibility relationships between elements
- Objective = maximise total importance within budget

Three algorithms for practical use:

### Algorithm 1: Minimal Crafter
For prompts that almost always fit with only history truncation needed.
- Keep the tail (most recent) of each content section
- Use when: simple fixed-system-prompt + single user message

### Algorithm 2: Additive Greedy
For prompts with mostly dynamic/optional content (many retrieved snippets competing for space).
1. Sort all candidate elements by importance (descending)
2. Start with empty prompt; used_tokens = 0
3. For each element (highest importance first): check dependencies; check incompatibilities; pick longest elastic version that fits; if fits → add, update token count
4. Re-sort assembled elements by position property to produce final ordered string

### Algorithm 3: Subtractive Greedy
For prompts where most elements are required, only a few need pruning.
1. Start with all elements included; compute total token count
2. While total > budget: find lowest-importance element with no dependents; remove or shrink to shorter elastic version; update token count
3. Sort remaining by position

### Element Properties

Every prompt element has three properties that govern assembly:

| Property | Description | Use |
|---|---|---|
| **Position** | Ordered slot or named position (e.g., `system`, `few_shot`, `user_query`) | Determines final ordering after assembly |
| **Importance** | Absolute value score or length-normalised (value ÷ token count) | Drives selection priority in greedy algorithms |
| **Dependency** | Requirements (A needs B) or incompatibilities (A excludes B) | Hard constraints before importance ranking |

**Length-normalised importance** = importance ÷ token_count. Measures value per token. Prevents long but marginally relevant passages from crowding out concise high-value ones. Use when token budget is tight.

### Elastic Snippets

Same content at multiple compression levels — pick the longest version that fits:
- **Full**: complete document excerpt
- **Summary**: 3–5 key points
- **Terse**: 1–2 sentence essence
- **One-liner**: single sentence minimum

Two elasticity strategies:
1. **Elastic prompt elements** — one logical element with multiple pre-written compression levels; best for system instructions and persona descriptions
2. **Multiple snippets with dependencies** — several snippets where detailed ones require the base snippet; best for retrieved documents with summaries

---

## 6. RAG Fundamentals

### Why RAG Exists

LLMs have a training cutoff and no access to private data. Without RAG:
- Questions about recent events or private knowledge → hallucination or refusal
- RAG injects relevant knowledge at inference time directly into the prompt

### Retrieval Approaches

| Approach | How it works | Strengths | Weaknesses |
|---|---|---|---|
| **Lexical (BM25)** | Jaccard similarity on stemmed word sets | Fast, debuggable, mature tooling | Fails on synonyms, paraphrases, typos |
| **Neural (embeddings)** | Cosine similarity on dense embedding vectors | Handles synonyms, paraphrases, cross-language | Opaque, requires embedding pipeline + vector DB |
| **Hybrid** | Lexical candidate retrieval + neural re-ranking | Best of both | More complex pipeline |

**Hybrid search beats either alone** for production RAG. Start with lexical; add neural when exact-word matching demonstrably fails.

### Snippetizing Rules for RAG

Each chunk must satisfy all three:
1. Fits within the embedding model's token limit (e.g., ≤8,191 for OpenAI)
2. Contains **one main idea** — multi-topic chunks produce defocused retrieval
3. Is an appropriate size for inclusion in the final LLM prompt

Chunking strategies:
- **Sliding window** — fixed word count (e.g., 256 words) with overlap (e.g., 50 words) to prevent ideas being cut at boundaries
- **Natural boundaries** — split at paragraph or section breaks; ensures each snippet contains exactly one topic
- **Augmented snippets** — append metadata that belongs conceptually to the snippet (e.g., function signatures added to code snippet bodies)

### Critical RAG Rule

Always use the same embedding model for indexing and querying. Vectors from different models are in incompatible spaces — mixing them produces meaningless results.

---

## 7. Evaluation Foundations

### Offline Evaluation Progression

1. **Example suite** (start here) — 5–20 diverse inputs; script saves prompt + completion to files; review output deltas with your editor diff tools
2. **Evaluation harness** — hundreds of examples + automated scoring
3. **Online A/B testing** — real users after harness validates quality

### Three Scoring Methods

| Method | When to use | Key notes |
|---|---|---|
| **Gold standard matching** | Binary decisions; tool-calling correctness | Check correct tool called with valid arguments — highest-leverage partial match |
| **Functional testing** | Executable outputs (code, SQL, config) | Use existing test suite as external oracle; no gold standard needed |
| **LLM-as-judge (SOMA)** | Free-form completions where above don't apply | Yields relative quality only; must be calibrated against human labels |

### SOMA Framework for LLM-as-Judge

**S — Specific questions**: Ask narrow, answerable sub-questions, not "Is this good?"

**O — Ordinal scaled answers**: 5-point scale with explicitly labelled options. Use asymmetric scale (shift neutral toward positive) to correct for LLM positivity bias.

**MA — Multi-aspect coverage**: Multiple independent quality dimensions, each with its own question + scale. Split "Goldilocks questions" (was it just right?) into two: "was it enough?" and "was it too much?"

**Key structural rule**: State evaluation framework and aspects to grade **before** showing the example to be judged — the model cannot backtrack, so it must encounter the rubric before reading the completion.

### Online Metric Types (most to least reliable)

1. **Direct feedback** — thumbs up/down, star rating (high quality, low volume)
2. **Functional correctness** — output executed and tested by external oracle
3. **User acceptance** — user acts on suggestion without modifying (easy to measure, confounded by inertia)
4. **Achieved impact** — downstream outcome after suggestion is followed (truest measure, requires long observation window)
5. **Incidental metrics** — session depth, return rate, edit distance (noisiest, but free from existing analytics)
