# Example 02 — Blocking Ambiguity (stop-and-ask)

## Input

```
Make me an app.
```

---

## ❌ Without Cave Prompt — Raw prompt sent directly to LLM

**What the LLM typically does:**

The model receives an almost-empty request and has two options: ask for clarification, or make assumptions and proceed. Most LLMs default to **making assumptions** to appear helpful.

**Typical LLM response:**

> "Sure! I'll help you build a simple web app. Here's a Todo List application using React:
>
> ```javascript
> import React, { useState } from 'react';
> function App() { ... }
> ```
> ..."

**What went wrong:**

- You asked for "an app" — you got a React todo list. Was that what you wanted? **Nobody knows.**
- The LLM committed to: React (not Vue/native), web (not mobile/desktop), todo list (not ecommerce/dashboard/tool), JavaScript (not Python/Swift/Go)
- You discover the mismatch **after** reading 200 lines of generated code
- You re-prompt, re-run, waste tokens and time
- In a pipeline or agent system: **the wrong app gets built, downstream steps fail silently**

**The core problem:**

Semantic density = **0.05**. There is almost no information in this prompt. A raw LLM call cannot safely compile "Make me an app" into anything meaningful — it fills every gap with assumptions you never agreed to.

---

## ✅ With Cave Prompt — Blocking ambiguity caught before execution

Cave Prompt detects that the prompt cannot be safely compiled. Instead of guessing, it **stops and asks** — before a single token of output is wasted on the wrong thing.

**CLI output:**

```
$ cave compile "Make me an app."

Blocking ambiguity — clarify before compiling:
  - What platform? (web, mobile iOS/Android, desktop, CLI?)
  - What is the app's primary function? (e.g. task manager, ecommerce, social, internal tool...)
  - Preferred tech stack, or open to suggestions?

[exit code 2]
```

**Why this matters:**

| | Raw prompt | Cave Prompt |
|---|---|---|
| Detects ambiguity | ✗ (proceeds with assumptions) | ✅ (stops, asks) |
| Wastes tokens on wrong output | ✅ yes | ✗ nothing generated |
| Makes assumptions explicit | ✗ | ✅ |
| Pipeline-friendly exit signal | ✗ | ✅ exit code 2 |
| Forces clarification upfront | ✗ | ✅ |

**Exit code 2** is intentional — it signals "ambiguity, not a system error". Pipelines can branch on it:
```python
if exit_code == 2:
    ask_user_for_clarification()
elif exit_code == 1:
    raise SystemAlert()
```

After the user answers the three questions, Cave Prompt compiles a clean, precise execution prompt — and the main LLM receives a brief instead of a guess.

---

## Full envelope (machine-readable)

```json
{
  "blocking_ambiguities": [
    "What platform? (web, mobile iOS/Android, desktop, CLI?)",
    "What is the app's primary function? (e.g. task manager, ecommerce, social, internal tool...)",
    "Preferred tech stack, or open to suggestions?"
  ],
  "semantic_analysis": {
    "intent": "",
    "domain": "",
    "entities": [],
    "constraints": {},
    "priorities": [],
    "response_preferences": {},
    "ambiguities": [
      "Platform not specified",
      "Functionality not specified",
      "Tech stack not specified"
    ],
    "hidden_requirements": []
  },
  "optimized_ir": {
    "task_type": "",
    "execution_requirements": [],
    "context_priority": {},
    "reasoning_mode": "",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.05,
    "redundant_spans": [],
    "low_information_spans": ["Make me an app"],
    "execution_critical_spans": [],
    "summary": "Extremely low semantic density (0.05). Entire prompt is an unresolvable ambiguity — cannot compile."
  },
  "verbatim_spans": [],
  "fidelity_score": 0.0,
  "dropped_or_uncertain": [],
  "execution_prompt": ""
}
```

---

*Demonstrates: blocking ambiguity policy — stop-and-ask, no execution prompt generated, exit code 2. A raw LLM call would silently assume and produce wrong output.*
