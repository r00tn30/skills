# ADK Agent Prompt Patterns
## Reference: Google Agent Development Kit — Agent Composition, Tool Design, and Reasoning Frameworks

---

## 1. ADK Agent Architecture

### The Three Prompt Surfaces

Every ADK agent exposes three surfaces where prompt quality matters:

| Surface | Audience | Purpose |
|---|---|---|
| `name` | System / logging | Identity in multi-agent routing and traces. Use `snake_case`. |
| `description` | Orchestrator agent | How a parent agent decides to delegate here. Written for the LLM router, not the task model. |
| `instruction` | Gemini model inside this agent | Behavioural constitution — processed first, conditions all tool use and outputs. |

### How ADK Tool Calling Works

ADK wraps the standard Gemini function-calling loop:

```
1. ADK sends: instruction + conversation history + tool definitions (from Python function signatures + docstrings)
2. Gemini responds: prose, tool call request (JSON), or both
3. ADK executes the Python function
4. ADK appends result as a tool response message
5. Gemini generates next response — repeat until prose-only response
```

**What this means for prompts**: Tool docstrings are embedded in the system prompt as function signatures. Gemini reads them as part of the instruction context. Write them as instructions, not comments.

---

## 2. ADK Tool Definition Patterns

### Full Tool Definition Template

```python
from typing import Literal, Optional

def verb_object(
    required_param: str,
    bounded_param: Literal["option_a", "option_b", "option_c"],
    optional_param: Optional[str] = None,
) -> dict:
    """One sentence: what this tool does. What it returns.

    Use when: [specific trigger condition — be unambiguous].
    Do NOT use when: [domain overlap exclusion, if any].

    Args:
        required_param: Description. Format: [format]. Example: "ORD-123456".
        bounded_param: Which [X] to [Y]. One of: option_a (meaning), option_b (meaning).
        optional_param: Description. Omit unless [specific condition].

    Returns:
        dict with:
          - field_a (str): description
          - field_b (int): description
        On failure: {"error": "not_found" | "permission_denied" | "timeout"}
    """
    # Implementation — inject known values here, not as params
    user_id = ctx.get_user_id()  # Never expose user_id as a tool parameter
    ...
```

### Tool Design Decision Tree

```
Does the application already know the value (user_id, session_id, org_id)?
  YES → Inject it inside the function. Never put it in the signature.
  NO  → Is the valid value set bounded?
          YES → Use Literal["value1", "value2", ...] type hint
          NO  → Is there a sensible default?
                  YES → Use Optional[type] = default_value
                  NO  → Required param; describe format precisely in docstring

Is this tool dangerous (writes, deletes, sends, modifies external state)?
  YES → Gate in before_tool_callback — never trust docstring instructions alone
  NO  → Proceed

Does this tool's domain overlap with another tool?
  YES → Add explicit "Do NOT use when [other tool's domain]" to the docstring
  NO  → Proceed

Is a long text blob needed as input?
  YES → Pass via session state or context, not as a function parameter
  NO  → Proceed
```

### Common ADK Tool Patterns

**Search / Retrieval Tool:**
```python
def search_knowledge_base(
    query: str,
    max_results: int = 5,
) -> list[dict]:
    """Search the internal knowledge base for relevant documentation.

    Returns up to max_results passages sorted by relevance score.
    Use when the user asks about product features, policies, or procedures.
    Do NOT use for real-time data or user-specific information — use get_user_data instead.

    Args:
        query: Natural language search query. Be specific for better results.
        max_results: Number of results to return. Default 5, max 10.

    Returns:
        List of dicts, each with: content (str), source (str), relevance_score (float).
        Empty list if no relevant results found.
    """
    ...
```

**Action Tool (dangerous — requires callback gate):**
```python
def send_email(
    to: str,
    subject: str,
    body: str,
) -> dict:
    """Send an email to a recipient.

    Returns a confirmation ID on success.
    Use only after the user has confirmed the recipient and content in this conversation.

    Args:
        to: Recipient email address.
        subject: Email subject line. Max 100 characters.
        body: Email body in plain text. Max 2000 characters.

    Returns:
        dict with confirmation_id (str) on success, or {"error": "send_failed"} on failure.
    """
    # Application layer gates this via before_tool_callback — no "confirm first" instructions needed here
    ...
```

**Zero-Argument Fetch Tool:**
```python
def get_current_user_profile() -> dict:
    """Fetch the current user's profile, preferences, and subscription details.

    Call at the start of any personalisation task.
    No arguments needed — user context is injected by the application layer.

    Returns:
        dict with: name (str), email (str), plan (str), preferences (dict).
    """
    user_id = _get_user_id_from_context()  # Injected — model never sees or hallucinates it
    ...
```

**Finish / Termination Tool (for orchestrator agents):**
```python
def signal_completion(summary: str) -> dict:
    """Signal that the goal is fully complete. Call this when all required steps have succeeded.

    Do NOT call this if any step failed or if you are uncertain about completion.
    This is always the last tool call in a workflow.

    Args:
        summary: One-sentence summary of what was accomplished.

    Returns:
        dict with status: "complete".
    """
    ...
```

---

## 3. Reasoning Framework Patterns for ADK

### Chain-of-Thought in ADK `instruction` Fields

**Use CoT when**: multi-step logic, math, causal reasoning, classification with ambiguous cases.

**Pattern 1: Zero-Shot CoT (add to instruction)**
```
Think step by step before giving your final answer.
```

**Pattern 2: Few-Shot CoT (strongest)**
```python
instruction="""You are a billing dispute classifier.

Examples of how to approach classification:

---
Message: "I was charged twice for the same month"
Reasoning:
- The user is describing a duplicate charge
- This is about a specific transaction, not a general billing question
- The desired outcome is a refund or credit
Classification: duplicate_charge
---

Message: "Why is my bill so high this month?"
Reasoning:
- The user wants an explanation, not an immediate refund
- They may not know about a plan change or usage spike
- The desired outcome is information, then potentially a dispute
Classification: billing_inquiry
---

Message: {{customer_message}}
Reasoning:
"""
```

**Pattern 3: Structured CoT for ADK Agent Decision-Making**
```python
instruction="""You are a triage agent.

Before taking any action, reason through this decision:

Current situation: {{situation}}
Goal: {{goal}}

Analysis:
- What do I currently know?
- What information am I missing?
- Which tool is most likely to provide the missing information?
- What are the risks of this action?

Decision: [chosen action and why]
"""
```

**When NOT to use explicit CoT in ADK:**
- Gemini 2.5 Pro and Flash Thinking models reason internally — external CoT instructions conflict with their internal process
- Simple extraction tasks — CoT adds latency with no quality benefit
- Highly constrained output tasks — CoT preamble competes with format instructions

---

### ReAct in ADK

ADK natively implements the ReAct loop. Gemini automatically uses Think/Act/Observe behaviour when tools are present. The `instruction` shapes the quality of the reasoning, not the loop structure itself.

**Standard ReAct instruction pattern:**
```python
instruction="""You are a research agent. Answer questions by gathering information from available tools.

For every task:
1. Reason about which source is most relevant before calling any tool
2. Call tools one at a time — evaluate each result before deciding the next step
3. After each tool result, assess: do I have enough to answer, or should I search further?
4. Stop calling tools when you have sufficient information
5. Synthesise your answer from tool results only — do not add unretrieved information

If a tool returns an error:
- Reason about why it failed
- Try a different approach (different query, different tool) before giving up
- After 2 failed attempts on the same sub-question, acknowledge the gap in your answer
"""
```

**Plan-and-Solve + ReAct (for complex multi-step tasks):**
```python
instruction="""You are a data analysis agent.

For every task:
STEP 1 — Write a numbered plan before executing anything:
Plan:
1. [what you will retrieve first and why]
2. [what you will compute next]
3. [how you will synthesise the results]

STEP 2 — Execute the plan using your tools, one step at a time.

STEP 3 — Final answer: synthesise from your observations.
"""
```

---

### Reflexion Pattern (Iterative Self-Correction)

```python
instruction="""[Primary task instruction here]

---
After completing your response, self-review against each criterion:

□ Accuracy — all factual claims are from tool results or the provided context
□ Completeness — all parts of the original request are addressed
□ Format — output matches the specified format exactly
□ Conciseness — no redundant sentences

If any criterion fails:
1. Identify the weakest failing element
2. Write a targeted revision of only that element
3. Re-evaluate — stop when all criteria pass or after 3 retries
"""
```

---

## 4. ADK Multi-Agent Workflow Patterns

### Orchestrator Agent Instruction

```python
orchestrator = Agent(
    name="claim_processing_orchestrator",
    model="gemini-2.0-flash",
    description="Orchestrates the full insurance claim processing workflow. Call when a new claim needs to be processed end to end.",
    instruction="""You are a workflow orchestrator for insurance claim processing.

You have access to specialist sub-agents. Delegate — do not do their work yourself.

Sub-agents:
- document_extractor: Extracts structured data from claim documents. Call first with the raw claim document.
- fraud_detector: Analyses extracted data for fraud signals. Call after document_extractor succeeds.
- adjudicator: Applies policy rules and determines payout. Call after fraud_detector returns a clean result.
- payment_processor: Initiates the approved payment. Call last, only after adjudicator approves.

Workflow rules:
- Always call document_extractor first — no other step works without structured data
- Never call adjudicator if fraud_detector returns risk_level: "high" — escalate to human review instead
- Never call payment_processor before adjudicator returns status: "approved"
- If any sub-agent returns an error: stop, report the specific failure and step name to the user, wait for instructions
- Do not skip steps to save time
""",
    sub_agents=[document_extractor, fraud_detector, adjudicator, payment_processor],
)
```

### Task Agent Instruction (sub-agents in a workflow)

```python
extractor_agent = Agent(
    name="document_extractor",
    model="gemini-2.0-flash",
    description="Extracts structured claim data from raw document text. Returns JSON. Call with the raw document content.",
    instruction="""<role>
You are a document extraction specialist. Your only responsibility is to parse raw claim document text into structured JSON. Do not adjudicate, validate, or interpret — only extract.
</role>

<task>
Extract all fields listed in the output schema from the provided document. Be precise — do not infer or guess missing fields.
</task>

<output_schema>
Return ONLY valid JSON matching this exact schema — no preamble, no explanation:
{
  "claimant_name": "string",
  "claim_date": "YYYY-MM-DD",
  "incident_type": "string",
  "claimed_amount": "number",
  "policy_number": "string"
}
</output_schema>

<constraints>
- If a required field is missing from the document: set its value to null
- Do not hallucinate values not present in the document text
- If the input does not appear to be a claim document: return {"error": "not_a_claim_document"}
</constraints>
""",
)
```

### Branch-Solve-Merge Pattern

**Solver agent instruction** (instantiate N times with different roles):
```python
instruction="""You are analysing {{problem}} from the perspective of a {{specific_expert_role}}.

Your specific lens: {{angle_this_solver_emphasises}}

Provide your analysis in this structure:
- Key observations from your perspective (3–5 bullet points)
- Primary recommendation with rationale
- Risks or blind spots in your analysis
- What additional information would change your recommendation
"""
```

**Merge agent instruction:**
```python
instruction="""You have received {{N}} independent expert analyses of the same problem.

Analyses:
<prior_output agent="analyst_1">{{analysis_1}}</prior_output>
<prior_output agent="analyst_2">{{analysis_2}}</prior_output>

Your task: Synthesise into the single best answer.

Process:
1. Points of agreement → high-confidence findings; lead with these
2. Points of disagreement → analyse root cause; adjudicate with reasoning
3. Unique insights from individual analysts → preserve what adds value
4. Gaps identified by multiple analysts → flag explicitly

Final Synthesis:
[Lead with the direct recommendation]
[Support with high-confidence findings]
[Address key disagreements]
[Flag remaining uncertainties]
"""
```

---

## 5. Context Window Management for ADK Agents

### History Management in ADK Sessions

ADK manages conversation history within `Session` objects. Prompt engineering responsibility: control what goes into the instruction vs. what is retrieved on demand.

| Situation | Strategy |
|---|---|
| Short session, bounded length | Full history — ADK handles by default |
| Long session, topic continuity | Summarise older turns; inject summary as context at top of instruction |
| Very long session | Use `AgentTool` to expose a `search_session_history` retrieval tool |
| Clear task boundaries | Use `LoopAgent` or structured handoffs to signal context resets |

### History Compression Prompt

```python
summariser = Agent(
    name="history_summariser",
    model="gemini-2.0-flash",
    instruction="""Produce a concise summary of the conversation history that preserves:
- All decisions made and their rationale
- All information the user provided about their situation
- The current state of any in-progress tasks
- Any open questions or unresolved issues

Do NOT preserve:
- Small talk or pleasantries
- Repeated clarifications of the same point
- Content superseded by later decisions

Target: {{target_tokens}} tokens.
""",
)
```

### Artifact-as-Tool Pattern

When documents are large, expose them as retrieval tools rather than injecting inline:

```python
def search_uploaded_document(query: str) -> str:
    """Search the user's uploaded document for specific information.

    Use when you need details from the document that aren't in the summary.
    Returns the most relevant passage (up to 500 words).

    Args:
        query: What to look for in the document.

    Returns:
        str: The most relevant passage, or "No relevant content found." if not found.
    """
    ...

# Agent instruction then includes only:
# "A summary of the uploaded document: {{summary}}.
#  Use the search_uploaded_document tool to look up specific details."
```

---

## 6. ADK Callback Patterns

### before_tool_callback — Application-Layer Gate

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.tools import BaseTool
from typing import Optional

REQUIRES_APPROVAL = {"send_email", "initiate_refund", "delete_record", "modify_permissions"}

def gate_sensitive_tools(
    tool: BaseTool,
    args: dict,
    ctx: CallbackContext,
) -> Optional[dict]:
    """Gate sensitive tools before execution."""
    if tool.name in REQUIRES_APPROVAL:
        # Short-circuit: return a dict instead of executing the tool
        # The model receives this dict as the tool result
        return {
            "status": "pending_approval",
            "message": f"Action requires user confirmation before executing.",
            "tool": tool.name,
            "args": args,
        }
    return None  # None = proceed with execution
```

### after_tool_callback — Output Enrichment

```python
def enrich_tool_errors(
    tool: BaseTool,
    args: dict,
    result: dict,
    ctx: CallbackContext,
) -> Optional[dict]:
    """Enrich error results with actionable recovery guidance."""
    if isinstance(result, dict) and "error" in result:
        error_guidance = {
            "not_found": f"The requested record was not found. Verify the ID and try again.",
            "timeout": f"The service timed out. Retry once before escalating.",
            "permission_denied": f"Insufficient permissions. This action requires elevated access.",
        }
        result["recovery_hint"] = error_guidance.get(result["error"], "Contact support.")
    return result
```

### on_before_model — Dynamic Instruction Injection

```python
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmRequest

def inject_user_context(
    ctx: CallbackContext,
    request: LlmRequest,
) -> Optional[LlmRequest]:
    """Inject user-specific context into the instruction at runtime."""
    user_profile = ctx.state.get("user_profile", {})
    if user_profile:
        context_injection = f"\n\nCurrent user context: {user_profile}"
        # Append to the last system message
        request.messages[0].content += context_injection
    return request
```

---

## 7. Two-Step Ideation Pattern

Consistently outperforms asking the model to brainstorm and select in a single call.

**Step 1 Agent — Divergent (brainstorm, no filtering):**
```python
brainstorm_agent = Agent(
    name="idea_generator",
    instruction="""Generate {{N}} candidate {{items}} for {{context}}.

Rules for this step:
- Quantity over quality — generate all {{N}} even if some seem unlikely
- One idea per line
- No evaluation, no ranking, no filtering — just generate
- Ensure variety: do not cluster around similar ideas

{{N}} candidates:
""",
)
```

**Step 2 Agent — Convergent (evaluate and select):**
```python
selector_agent = Agent(
    name="best_option_selector",
    instruction="""Select the single best option from the candidates and justify your choice.

Context:
<data>{{context}}</data>

Candidates:
<prior_output agent="idea_generator">{{candidates}}</prior_output>

Evaluation criteria:
- {{criterion_1}}
- {{criterion_2}}
- {{criterion_3}}

Selection:
Best candidate: [name]
Rationale: [2 sentences citing specific details from the context]
Why not the others: [1 sentence on the key weakness of the top alternative]
""",
)
```
