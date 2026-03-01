---
name: ai-agent-stack
description: Add production safety to any AI agent codebase — budget enforcement, prompt injection detection, and decision tracing. Use when the user wants to protect their LLM app from cost overruns, prompt injection attacks, or needs to audit and trace AI agent decisions. Triggers on phrases like "add cost protection", "protect my LLM calls", "instrument my agent", "add tracing", "prevent prompt injection", "add safety to my bot", "make my agent production-ready".
license: Apache-2.0
---

# AI Agent Infrastructure Stack

Add three production safety layers to any AI agent codebase — in under 5 minutes, zero dependencies.

## The Stack

| Library | Problem it solves | Install |
|---|---|---|
| `ai-cost-guard` | Hard budget cap before the LLM call — raises `BudgetExceededError` instead of burning money | `pip install ai-cost-guard` |
| `ai-injection-guard` | Scans inputs for prompt injection before they reach the model — 22 patterns, 5 categories | `pip install ai-injection-guard` |
| `ai-decision-tracer` | Records every decision an AI agent makes — what it saw, decided, and why. JSON + Markdown output | `pip install ai-decision-tracer` |

All three: MIT licensed, zero dependencies, pure Python stdlib, no network calls.

---

## Process

### Step 1 — Understand what the user has

Before writing any code, search the codebase to understand:
- Which LLM provider they use (Anthropic, OpenAI, Google, LangChain, etc.)
- How they call the LLM (direct SDK, wrapper, decorator, etc.)
- Whether they already have any guards

Search for: `anthropic`, `openai`, `client.chat`, `client.messages`, `llm.invoke`, `chain.run`, `model.generate`

Ask the user which of the three layers they want if it's not obvious from context. Default: add all three.

### Step 2 — Install the libraries

Add to their `requirements.txt`, `pyproject.toml`, or show the pip install commands:

```
ai-cost-guard
ai-injection-guard
ai-decision-tracer
```

### Step 3 — Add each layer

#### ai-cost-guard (budget enforcement)

```python
from ai_cost_guard import CostGuard, BudgetExceededError

guard = CostGuard(
    weekly_budget_usd=5.00,      # hard cap — adjust to their use case
    storage_path=".ai_cost_guard"  # persists across restarts
)

# Wrap their LLM function:
@guard.protect(model="anthropic/claude-haiku-4-5-20251001")
def call_llm(prompt):
    return client.messages.create(...)  # their existing code
```

**Model string format**: `provider/model-id`
- Anthropic: `anthropic/claude-haiku-4-5-20251001`, `anthropic/claude-sonnet-4-6`
- OpenAI: `openai/gpt-4o`, `openai/gpt-4o-mini`
- Google: `google/gemini-1.5-pro`
- Ollama/local: `ollama/llama3.2` (cost = $0, still tracks usage)

**Handle the exception**:
```python
try:
    result = call_llm(prompt)
except BudgetExceededError as e:
    print(f"Budget exceeded: {e}")
    # graceful fallback
```

#### ai-injection-guard (prompt injection detection)

```python
from ai_injection_guard import PromptScanner, InjectionDetectedError

scanner = PromptScanner(threshold="MEDIUM")  # LOW | MEDIUM | HIGH

# Option A — decorator (cleanest):
@scanner.protect(arg_name="user_input")
def analyze(user_input):
    return llm.analyze(user_input)

# Option B — inline scan (for complex flows):
result = scanner.scan(user_input)
if result.is_threat:
    raise InjectionDetectedError(f"Injection detected: {result.threats}")
```

**Threshold guidance**:
- `HIGH` — only block obvious attacks (low false positives, good for internal tools)
- `MEDIUM` — balanced (recommended for most use cases)
- `LOW` — aggressive blocking (good for untrusted external inputs)

**What it catches**: role overrides ("ignore previous instructions"), jailbreaks, exfiltration attempts, fake authority signals, unicode encoding tricks

#### ai-decision-tracer (decision tracing)

```python
from ai_trace import Tracer

tracer = Tracer(
    "agent_name",                  # name shown in trace files
    trace_dir="traces",            # where to write files
    meta={"model": "claude-haiku"} # optional metadata
)

# Wrap decision points:
with tracer.step("classify", input_preview=text[:50]) as step:
    result = llm.classify(text)
    step.log(label=result.label, confidence=result.score, tokens=result.usage)

# Save at end of session:
tracer.save()            # → traces/agent_name_TIMESTAMP.json
tracer.save_markdown()   # → traces/agent_name_TIMESTAMP.md
```

**What gets recorded**: step name, context inputs, log entries, outcome (ok/error), duration_ms, full traceback on exceptions

**View traces from terminal**:
```bash
ai-trace list                    # list all sessions
ai-trace view agent_20240301.jsonl  # view a session
ai-trace tail                    # live-follow latest trace
ai-trace stats                   # aggregate stats
```

### Step 4 — Using all three together

Show a complete integrated example if the user wants all three:

```python
from ai_cost_guard import CostGuard, BudgetExceededError
from ai_injection_guard import PromptScanner, InjectionDetectedError
from ai_trace import Tracer

guard   = CostGuard(weekly_budget_usd=5.00)
scanner = PromptScanner(threshold="MEDIUM")
tracer  = Tracer("my_agent", meta={"model": "claude-haiku-4-5-20251001"})

@guard.protect(model="anthropic/claude-haiku-4-5-20251001")
@scanner.protect(arg_name="prompt")
def call_llm(prompt):
    with tracer.step("llm_call", prompt_len=len(prompt)) as step:
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        step.log(
            output_tokens=response.usage.output_tokens,
            stop_reason=response.stop_reason
        )
    return response

# Use it:
try:
    result = call_llm(user_input)
except BudgetExceededError:
    print("Weekly budget exhausted — pausing until next week")
except InjectionDetectedError:
    print("Suspicious input blocked before reaching the model")
```

### Step 5 — Verify

After adding the code, verify it works by running a quick test:

```python
# Test cost guard is tracking
print(guard.remaining_budget())

# Test injection guard with a safe input
result = scanner.scan("What is the weather today?")
assert not result.is_threat

# Test tracer created a file
import os
assert os.path.exists("traces/")
```

---

## Common patterns

### LangChain integration
```python
from langchain_core.runnables import RunnableLambda
from ai_injection_guard import PromptScanner

scanner = PromptScanner(threshold="MEDIUM")

def safe_invoke(input_dict):
    scanner.scan(input_dict["input"])  # raises on detection
    return chain.invoke(input_dict)

safe_chain = RunnableLambda(safe_invoke)
```

### Async agents
```python
import asyncio
from ai_trace import Tracer

tracer = Tracer("async_agent")

async def process(item):
    with tracer.step("process", item_id=item.id) as step:
        result = await llm.agenerate(item.prompt)
        step.log(tokens=result.llm_output["token_usage"]["total_tokens"])
    return result
```

### Environment-based budget tiers
```python
import os
from ai_cost_guard import CostGuard

budget = float(os.environ.get("WEEKLY_BUDGET_USD", "5.00"))
guard = CostGuard(weekly_budget_usd=budget)
```

---

## GitHub repos
- ai-cost-guard: github.com/LuciferForge/ai-cost-guard
- ai-injection-guard: github.com/LuciferForge/prompt-shield
- ai-decision-tracer: github.com/LuciferForge/ai-trace
