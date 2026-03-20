# Agent Prompt Patterns
## Reference: Conversational Agency, Workflows, and Multi-Agent Systems
### Source: Prompt Engineering for LLMs – Berryman & Ziegler (O'Reilly), Chapters 8–9

---

## 1. Conversational Agency Architecture

### The Three Ingredients of Every Agent

1. **Tools** — reach into the real world to gather information and take actions
2. **Reasoning** — think before acting (CoT, ReAct, plan-and-solve)
3. **Context** — know what matters (prior conversation, artifacts, retrieved knowledge)

### Tool Calling — What's Actually Happening

Tool calling is fine-tuned document completion with API-level syntactic sugar — the same trick as ChatML for chat. Tool definitions are embedded in the system message as TypeScript function signatures. The model has been fine-tuned to generate structured JSON tool-call tokens rather than prose when a tool is needed. The API intercepts these and routes them back to the application code.

**The tool call loop:**
```
1. Application sends: all messages + tool definitions
2. Model responds: prose, tool call requests, or both
3. Application executes each requested tool
4. Application appends results as tool messages
5. Go to 1 — repeat until model responds with prose only
```

### Context Sources for Task-Based Agents

| Source | Description | Management rule |
|---|---|---|
| **Preamble** | Agent rules, tool definitions, few-shot examples (system message) | Always included; Tier 1 |
| **Prior conversation** | All recent user ↔ assistant turns | Drop when topic shifts; expire old sessions |
| **Artifacts** | Data attached to messages (docs, code, API results) | Include via elastic snippets; expose full version as retrieval tool |
| **Current exchange** | Current user request + newly attached artifacts | Always included; goes last |
| **Agent response** | Model's reply | Becomes prior conversation on next turn |

**Artifact selection strategies:**
- **Include all** — model has maximum information; risk: confusion and token bloat
- **Ask the model to select** — side-trip model call; adds latency and cost
- **RAG / retrieval tool** — expose artifact access as a search tool; scalable; adds tool-call round trip
- **Best hybrid**: provide bulleted summary inline + expose full artifact as a retrieval tool on demand

---

## 2. Tool Definition Patterns

### Full Tool Definition Template

```typescript
{
  name: "[verbObject]",             // camelCase; self-documenting; what + on what
  description: "[One sentence: what it does + when to use it. Output format described. When NOT to use if ambiguous.]",
  parameters: {
    type: "object",
    properties: {
      paramName: {
        type: "string | number | integer | boolean | array",
        enum: ["value1", "value2"],   // use when values are bounded
        default: "defaultValue",      // always provide for optional args
        description: "[What this arg does. Valid values. Format if non-obvious.]"
      }
    },
    required: ["onlyTrulyRequiredParam"]
  }
}
```

### Tool Design Decision Tree

```
Does the application already know the value (user_id, org, session)?
  YES → Remove the argument from the tool definition entirely
  NO  → Is the valid value set bounded?
          YES → Use enum type
          NO  → Is there a sensible default?
                  YES → Provide a default value
                  NO  → Mark required; describe format precisely in description

Is this tool dangerous (writes, deletes, sends, modifies external state)?
  YES → Gate in application layer; never trust prompt instructions alone
  NO  → Proceed

Does this tool overlap with another tool's purpose?
  YES → Redesign to partition cleanly or merge into one
  NO  → Proceed

Will long-form text be passed as an argument?
  YES (OpenAI) → Avoid — JSON escaping corrupts multiline strings; pass via tool output instead
  YES (Anthropic) → OK — Anthropic uses XML for function arguments, no escaping issues
```

### Common Tool Patterns

**Search / Retrieval Tool:**
```typescript
{
  name: "searchKnowledgeBase",
  description: "Search the internal knowledge base for relevant documentation. Returns up to 5 passages sorted by relevance. Use when the user asks about product features, policies, or procedures. Do NOT use for real-time or user-specific information.",
  parameters: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "Natural language search query. Be specific for better results."
      },
      maxResults: {
        type: "integer",
        default: 5,
        description: "Number of results to return. Max 10."
      }
    },
    required: ["query"]
  }
}
```

**Action Tool (with application-layer gate):**
```typescript
// Note: NO "check with user first" instruction — that goes in the application layer, not here
{
  name: "sendEmail",
  description: "Send an email to a recipient. Returns a confirmation ID on success. Use only after confirming recipient and content with the user in the conversation.",
  parameters: {
    type: "object",
    properties: {
      to: { type: "string", description: "Recipient email address." },
      subject: { type: "string", description: "Email subject line. Max 100 characters." },
      body: { type: "string", description: "Email body in plain text. Max 2000 characters." }
    },
    required: ["to", "subject", "body"]
  }
}
// Application layer intercepts this call → shows to user → requires explicit sign-off → then executes
```

**Zero-Argument Data Fetch Tool:**
```typescript
{
  name: "getCurrentUserProfile",
  description: "Fetch the current user's profile, preferences, and recent activity. Call at the start of any personalisation task. Returns JSON.",
  parameters: {
    type: "object",
    properties: {},
    required: []
    // user_id injected by application layer — model never sees or hallucinates it
  }
}
```

**Finish / Termination Tool (for orchestrator agents):**
```typescript
{
  name: "finish",
  description: "Signal that the goal is fully complete and no more tool calls are needed. Call this when you have successfully completed all required steps and delivered the final output to the user.",
  parameters: {
    type: "object",
    properties: {
      summary: {
        type: "string",
        description: "One-sentence summary of what was accomplished."
      }
    },
    required: ["summary"]
  }
}
```

---

## 3. Reasoning Framework Patterns

### Chain-of-Thought (CoT) — Complete Patterns

**Use CoT when**: multi-step logic, math, causal reasoning, classification with ambiguous cases, decisions where the first intuition is likely wrong.

**Pattern 1: Zero-Shot CoT (simplest)**
```
[Prompt content]

Think step by step before giving your final answer.
```

**Pattern 2: Few-Shot CoT (strongest — reasoning MUST precede the answer)**
```
[System: role + task instruction]

Here are examples of how to approach this task:

---
Problem: A train leaves Chicago at 9 AM at 60 mph. Another leaves Detroit (280 miles away)
         at 10 AM at 80 mph. When do they meet?
Reasoning:
- Train 1 has a 1-hour head start: by 10 AM it has covered 60 miles; 220 miles remain.
- Combined closing speed: 60 + 80 = 140 mph
- Time to close 220 miles: 220 ÷ 140 ≈ 1.57 hours after 10 AM
- Meeting time: ≈ 11:34 AM
Answer: The trains meet at approximately 11:34 AM, about 157 miles from Detroit.
---

Problem: {{actual_problem}}
Reasoning:
```

**Pattern 3: Structured CoT for Agent Decision-Making**
```
Before taking any action, reason through this decision:

Current situation: {{situation}}
Goal: {{goal}}
Available actions: {{available_tools}}

Analysis:
- What do I currently know?
- What information am I missing?
- Which action is most likely to provide the missing information?
- What are the risks of this action?

Decision: [chosen action and why]
```

**Pattern 4: CoT for Classification (prevents anchoring to first impression)**
```
Classify the following into exactly one of: [billing | technical | feature-request | complaint | other]

Think through this before deciding:
1. What is the core topic of the message?
2. What outcome does the sender want?
3. Which category captures both the topic and the desired outcome?
4. Does this classification feel right? If not, reconsider.

Classification:
```

**When NOT to use explicit CoT:**
- Reasoning models (o1, o3, DeepSeek-R1, Claude extended thinking) — they reason internally; explicit CoT instructions can conflict with their internal process. Instead: state the goal and constraints clearly, let the model reason on its own.
- Simple extraction tasks — CoT adds latency and tokens with no quality benefit
- Highly constrained output tasks — CoT preamble competes with format instructions

---

### ReAct (Reasoning + Acting) — Complete Patterns

**Use ReAct when**: multi-step tasks requiring tool calls interleaved with reasoning, tasks where the plan depends on discovered information, research-style tasks.

**Pattern 1: Standard ReAct System Prompt**
```
You are {{agent_role}}.

You have access to tools to help you complete tasks. Use them when you need
information you don't have or when you need to take real-world actions.

For every task, use this exact format — do not deviate:

Thought: [Reason about what to do next. What do I know? What am I missing? What's the best next step?]
Action: [tool_name]
Action Input: {"arg": "value"}
Observation: [tool result — provided by the system, NOT generated by you]
... repeat Thought / Action / Action Input / Observation as needed ...
Thought: [I now have enough information to give a complete answer.]
Final Answer: [Your complete, well-formed response to the original request]

Rules:
- Always start with a Thought
- Exactly one Action per step — never call multiple tools simultaneously
- Never write an Observation yourself — wait for the system to provide it
- Use Final Answer only when you have sufficient information to fully respond
- If a tool returns an error: Thought about what went wrong and try a different approach
- Do not repeat the same Action with the same inputs if it already failed
```

**Pattern 2: Plan-and-Solve + ReAct (for complex multi-step tasks)**
```
You are {{agent_role}}.

For every task:
STEP 1 — Write a numbered plan before executing anything:
Plan:
1. [what you'll do first and why]
2. [what you'll do second]
3. [how you'll synthesise the results]

STEP 2 — Execute the plan:
Thought: [starting reasoning]
Action: [tool_name]
Action Input: {"arg": "value"}
Observation: [result]
...

Final Answer: [synthesis based on all observations]
```

**Pattern 3: ReAct for Research / Multi-Hop QA**
```
You are a research agent. Answer questions by gathering information from multiple
sources and synthesising a well-supported response.

For each research task:

Thought: [What aspect should I investigate first? What sources are most relevant?]
Action: search
Action Input: {"query": "[targeted query — not the raw user question, a focused sub-query]"}
Observation: [results]
Thought: [What did I learn? What gaps remain? What should I search next?]
Action: search
Action Input: {"query": "[refined query targeting the gap]"}
Observation: [results]
Thought: [I have enough to synthesise an answer.]
Final Answer:
[Direct answer first]
[Supporting evidence from observations — cite specific sources]
[Acknowledge gaps or uncertainty if present]
[Do not introduce any information not found in observations]
```

**ReAct production performance note**:
- Without fine-tuning: ReAct with few-shot examples UNDERPERFORMS standard prompting
- After fine-tuning on ~3,000 think-act-observe examples: fine-tuned 8B ReAct > vanilla 62B standard; fine-tuned 62B ReAct > vanilla 540B standard
- For production agents: collect real think-act-observe traces and fine-tune; the quality jump is worth it

**Why reasoning matters (from ALFWorld benchmark)**:
- ReAct (Thought + Action): 71% task success
- Action-only (no Thought): 45% task success
- Reasoning contributes: goal decomposition, commonsense injection, exception handling, progress tracking

---

### Reflexion Pattern (Iterative Self-Correction)

```
[After completing the primary task, append this section:]

---
Self-Review:
Evaluate your response against each criterion below. Score each as PASS or FAIL.

□ Accuracy — all factual claims are supported by the provided context or tool observations
□ Completeness — all parts of the original request are addressed
□ Format — output matches the specified format exactly
□ Conciseness — no redundant sentences; every sentence adds value

If any criterion is FAIL:
1. Identify the weakest failing element
2. Write a targeted Revision of only that element
3. Re-evaluate — stop when all criteria PASS or after {{max_retries}} retries

Weakest failing element: [identify]
Revision: [improved version of that element only]
```

**Reflexion for code tasks (use failed test output as the feedback signal):**
```
[Initial code generation prompt]

---
After generating your code, it will be tested. If tests fail, you will receive the
failure messages below. Analyse each failure, identify the root cause, and produce
a corrected version — do not regenerate the entire file, only fix the failing logic.

Test failures:
{{test_output}}

Root cause analysis:
- [failure 1]: [why it failed]
- [failure 2]: [why it failed]

Corrected implementation:
```

---

### Branch-Solve-Merge Pattern

**Solver prompt** (instantiate N times with different roles/temperatures):
```
You are analysing {{problem}} from the perspective of a {{specific_expert_role}}.

Your specific lens: {{angle_this_solver_emphasises}}

Provide your analysis in this structure:
- Key observations from your perspective (3–5 bullet points)
- Primary recommendation with rationale
- Risks or blind spots in your analysis
- What additional information would change your recommendation
```

**Merge prompt:**
```
You have received {{N}} independent expert analyses of the same problem.
Each analyst approached it from a different professional perspective.

Analyses:
---
Analyst 1 ({{role_1}}):
{{analysis_1}}
---
Analyst 2 ({{role_2}}):
{{analysis_2}}
---
[... repeat ...]

Your task: Synthesise these into the single best answer.

Process:
1. Points of agreement → high-confidence findings; lead with these
2. Points of disagreement → analyse root cause; adjudicate with reasoning
3. Unique insights from individual analysts → preserve what adds value
4. Gaps identified by multiple analysts → flag explicitly

Final Synthesis:
[Lead with the direct recommendation]
[Support with high-confidence findings]
[Address key disagreements and their resolution]
[Flag remaining uncertainties]
```

---

## 4. Multi-Agent Workflow Prompt Patterns

### Orchestrator Agent Prompt

```
You are a workflow orchestrator responsible for completing: {{high_level_goal}}

You have access to these specialist agents as tools:
- {{task_agent_1}}: [what it does, when to use it]
- {{task_agent_2}}: [what it does, when to use it]
- finish: Signal that the entire goal has been achieved

Workflow rules:
- Always begin with {{mandatory_first_step}}
- Never call {{dangerous_step}} before {{prerequisite_step}}
- If any agent returns an error, explain it to the user and ask how to proceed
- Call finish only when ALL required steps have been successfully completed
- Do not skip steps to save time

Current goal: {{goal}}
Begin.
```

### Task Agent Prompt (for sub-agents within a workflow)

```
<role>
You are a {{specific_role}}. Your only responsibility in this workflow is to
{{single_specific_task}}. Do not attempt to do anything outside this scope.
</role>

<task>
{{imperative_task_description}}
</task>

<input>
{{serialised_input}}
</input>

<output_schema>
Return ONLY valid JSON matching this exact schema — no preamble, no explanation, no markdown:
{{json_schema}}
</output_schema>

<constraints>
- If {{edge_case_condition}}: return {"error": "{{description}}", "retry": {{true_or_false}}}
- Do not {{specific_exclusion}}
- Maximum {{N}} items in any list field
</constraints>
```

### AutoGen AssistantAgent Prompt

```
You are an expert {{domain_specialist}}.

Your task: {{specific_task_description}}

You have access to these tools: [tool definitions injected by framework]

Work step by step:
1. Analyse the problem
2. Propose an approach
3. Execute using available tools
4. Verify the result
5. When the task is fully complete, end your message with exactly: TERMINATE
```

### AutoGen UserProxyAgent Prompt

```
You are an execution proxy. Your only job is to execute code and tool calls
requested by the assistant and return results verbatim.

Rules:
- Execute every code block the assistant provides
- Return full stdout, stderr, and return codes — do not summarise
- When the assistant message contains "TERMINATE", stop and report completion
- Never modify the assistant's code before executing
```

### CrewAI Agent Role Template

```python
Agent(
    role="Senior Data Analyst",          # Specific job title — activates training priors
    goal="Extract and validate all financial metrics from the Q3 earnings report, "
         "flagging any figures that require human review",  # Concrete, measurable objective
    backstory="You have 10 years of experience analysing SEC filings at a top-tier "
              "investment bank. You are methodical, cite specific line items by page "
              "and section, and never present uncertain figures without flagging them.",
    # Backstory activates domain-specific training distribution — be specific and vivid
)
```

---

## 5. Two-Step Ideation Pattern

Consistently outperforms asking the model to brainstorm and select in a single call.

**Step 1 — Divergent (brainstorm, no filtering):**
```
Generate {{N}} candidate {{items}} for {{context}}.

Rules for this step:
- Quantity over quality — generate all {{N}} even if some seem unlikely
- One idea per line
- No evaluation, no ranking, no filtering — just generate
- Ensure variety: do not cluster around similar ideas

Context:
{{context}}

{{N}} candidates:
```

**Step 2 — Convergent (evaluate and select):**
```
You have {{N}} candidate {{items}} and a specific context.
Select the single best option and justify your choice.

Context:
{{context}}

Candidates:
{{candidates_from_step_1}}

Evaluation criteria:
- {{criterion_1}}
- {{criterion_2}}
- {{criterion_3}}

Selection:
Best candidate: [name]
Rationale: [2 sentences citing specific details from the context that make this the strongest choice]
Why not the others: [1 sentence identifying the key weakness of the top alternative]
```

---

## 6. Context Window Management for Agents

### History Management Decision Matrix

| Situation | Strategy | Implementation |
|---|---|---|
| Short session, bounded length | Full history injection | Pass all prior messages on every call |
| Long session, topic continuity | Summarisation | Periodic summarise + inject summary as system context |
| Very long session, topic jumps | Vector retrieval | Store turns in vector DB; retrieve relevant turns per request |
| Clear task boundaries | Topic-based expiry | Drop history when task completes; start fresh |

### Summarisation Prompt for History Compression

```
The following is a conversation history that is becoming too long to include in full.

Produce a concise summary that preserves:
- All decisions made and their rationale
- All information the user provided about their situation
- The current state of any in-progress tasks
- Any open questions or unresolved issues

Do NOT preserve:
- Small talk or pleasantries
- Repeated clarifications of the same point
- Content that was superseded by later decisions

Conversation history:
{{history}}

Summary (target: {{target_tokens}} tokens):
```

### Artifact-as-Tool Pattern

When artifacts are large, expose them as retrieval tools rather than injecting them inline:

```typescript
// Instead of injecting 10,000 tokens of a document inline, define this tool:
{
  name: "searchDocument",
  description: "Search the attached document for specific information. Use when you need details from the document that aren't in the summary. Returns the most relevant passage.",
  parameters: {
    properties: {
      query: { type: "string", description: "What to look for in the document." }
    },
    required: ["query"]
  }
}
```

System message then includes only: `"A summary of the attached document: {{summary}}. Use the searchDocument tool to look up specific details."`
