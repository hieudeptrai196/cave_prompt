# Example 03 — English input with code (language matching + A2 verbatim code preservation)

## Input

```
I need to refactor this Python function to be more efficient. It's currently O(n²) and
needs to handle up to 1M records. Here's the current code:

```python
def find_duplicates(items):
    duplicates = []
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[i] == items[j] and items[i] not in duplicates:
                duplicates.append(items[i])
    return duplicates
```

It needs to stay pure Python (no numpy), work in Python 3.10+, and the output order
doesn't matter.
```

---

## ❌ Without Cave Prompt — Raw prompt sent directly to LLM

**What the LLM typically does:**

The model reads the prompt and attempts a refactor. Most of the time it gets close — but the subtle constraints are where it fails.

**Common problems with a raw LLM response:**

**1. Violates explicit constraints:**
```python
# LLM "helpfully" imports numpy for performance
import numpy as np
def find_duplicates(items):
    return list(np.unique(items[np.array([...])])) 
```
→ Constraint "pure Python (no numpy)" was in the prompt — but buried in prose, easy to miss.

**2. Silently changes the function signature or name:**
```python
# LLM renames for "clarity"
def get_duplicate_values(lst):
    ...
```
→ If this is called in 200 places in production code, every call breaks. The original name `find_duplicates` should be verbatim-protected.

**3. Changes semantics without saying so:**
```python
# LLM returns a set instead of a list (order change)
def find_duplicates(items):
    seen = set()
    return list({x for x in items if items.count(x) > 1})
```
→ Output type changed. Breaks callers that expect a list. The constraint "output order doesn't matter" ≠ "change the return type".

**4. No fidelity signal — you don't know what was dropped:**
The LLM gives you code. You don't know:
- Was "Python 3.10+" enforced? (walrus operator used? match statement?)
- Was the O(n²) → O(n) reduction actually achieved or just claimed?
- Was `items[i] not in duplicates` semantics preserved?

---

## ✅ With Cave Prompt — Constraints extracted, code verbatim-protected

Cave Prompt reads the prompt before the LLM touches it. It **locks in every constraint** and **copies the code block verbatim** so the main LLM receives a precise brief with no room for creative interpretation.

**What Cave Prompt locks in:**

| Constraint | In raw prompt | Cave Prompt treatment |
|---|---|---|
| `pure Python (no numpy)` | Prose, easy to miss | Extracted to `constraints.technical` |
| `Python 3.10+` | Prose | Extracted to `constraints.technical`, verbatim-protected |
| `1M records` | Prose | Extracted to `constraints.performance`, verbatim-protected |
| `O(n²)` | Prose | Extracted to `execution_requirements`, verbatim-protected |
| `find_duplicates` function name | In code | Verbatim-protected — never renamed |
| Full code block | In prompt | Copied verbatim into execution prompt — never paraphrased |
| `output order doesn't matter` | Prose | Mapped to constraint, NOT interpreted as "change return type" |

**Fidelity score: 0.96** — what was dropped and why is listed explicitly in `dropped_or_uncertain`.

**Hidden requirement surfaced:**
- "preserve semantics: return list of values that appear more than once, each listed once" — this is implied by the original code but never stated. Cave Prompt reads the code and extracts it.

**The execution prompt sent to the main LLM:**

> Refactor the following Python function to reduce complexity from O(n²) to O(n). Constraints: pure Python only (no numpy), Python 3.10+, handle 1M records, output order irrelevant. Preserve semantics: return a list of values appearing more than once, each listed once.
>
> ```python
> def find_duplicates(items):
>     duplicates = []
>     for i in range(len(items)):
>         for j in range(i + 1, len(items)):
>             if items[i] == items[j] and items[i] not in duplicates:
>                 duplicates.append(items[i])
>     return duplicates
> ```
>
> Provide the refactored function with a brief explanation of the algorithmic change.

The main LLM now has zero ambiguity. Every constraint is explicit. The code is exact. There is no prose to misread.

---

## Full envelope (machine-readable)

```json
{
  "blocking_ambiguities": [],
  "semantic_analysis": {
    "intent": "Refactor a duplicate-finding function from O(n²) to a more efficient algorithm, handling 1M records",
    "domain": "Python algorithm optimization",
    "entities": ["find_duplicates", "items", "duplicates"],
    "constraints": {
      "technical": ["pure Python", "Python 3.10+", "no numpy"],
      "performance": ["handle up to 1M records", "reduce from O(n²)"],
      "business": ["output order doesn't matter"]
    },
    "priorities": ["time complexity reduction", "memory efficiency at scale", "standard library only"],
    "response_preferences": {
      "tone": "technical",
      "verbosity": "focused"
    },
    "ambiguities": ["memory budget not specified — assumed unconstrained for a set-based approach"],
    "hidden_requirements": [
      "preserve semantics: return list of values that appear more than once",
      "each duplicate value returned only once (current code ensures this)"
    ]
  },
  "optimized_ir": {
    "task_type": "code refactoring / algorithm optimization",
    "execution_requirements": [
      "replace O(n²) nested loop with O(n) set-based algorithm",
      "handle 1M records without numpy",
      "Python 3.10+ compatible",
      "output: list of duplicate values (order irrelevant)"
    ],
    "context_priority": {
      "high": ["algorithmic complexity reduction", "correctness preservation", "scale to 1M records"],
      "low": ["style preferences", "docstring format"]
    },
    "reasoning_mode": "algorithmic analysis + drop-in replacement code",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.78,
    "redundant_spans": ["I need to", "currently", "Here's the current code"],
    "low_information_spans": ["more efficient", "up to"],
    "execution_critical_spans": ["O(n²)", "1M records", "pure Python", "Python 3.10+", "find_duplicates", "items[i] == items[j]", "items[i] not in duplicates"],
    "summary": "Prompt has good density (0.78). Code block is entirely execution-critical and copied verbatim. Minor noise in framing sentences."
  },
  "verbatim_spans": [
    "O(n²)",
    "1M records",
    "pure Python",
    "Python 3.10+",
    "def find_duplicates(items):\n    duplicates = []\n    for i in range(len(items)):\n        for j in range(i + 1, len(items)):\n            if items[i] == items[j] and items[i] not in duplicates:\n                duplicates.append(items[i])\n    return duplicates"
  ],
  "fidelity_score": 0.96,
  "dropped_or_uncertain": [
    "memory budget — not specified, assumed unconstrained"
  ],
  "execution_prompt": "Refactor the following Python function to reduce complexity from O(n²) to O(n). Constraints: pure Python only (no numpy), Python 3.10+, handle 1M records, output order irrelevant. Preserve semantics: return a list of values appearing more than once, each listed once.\n\nCurrent implementation:\n\n```python\ndef find_duplicates(items):\n    duplicates = []\n    for i in range(len(items)):\n        for j in range(i + 1, len(items)):\n            if items[i] == items[j] and items[i] not in duplicates:\n                duplicates.append(items[i])\n    return duplicates\n```\n\nProvide the refactored function with a brief explanation of the algorithmic change."
}
```

---

*Demonstrates: output_language=match-input (English in → English out), A2 verbatim preservation of code block and technical spans ("O(n²)", "1M records", "Python 3.10+"), hidden semantic extraction from code, surgical compression of framing noise.*
