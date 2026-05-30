# Example 01 — NestJS Chatbot (verbatim preservation + hidden requirement extraction)

## Input

```
I want to build a customer support chatbot with NestJS, needs to handle 100k concurrent users,
use Redis for session caching, with token streaming. Infrastructure must be cheap,
and I need concise technical responses.
```

---

## ❌ Without Cave Prompt — Raw prompt sent directly to LLM

**What the LLM typically does:**

The model reads the prompt and tries to answer immediately. Without explicit structure, it has to guess:

- Does "cheap" mean serverless? shared hosting? spot instances? → **guesses**
- Does "token streaming" mean SSE or WebSocket? → **picks one arbitrarily**
- "100k concurrent users" — does the user want architecture advice, or actual code? → **often gives both, wastes tokens**
- "I want to" is a filler — the model processes it as signal → **attention budget wasted**

**What you typically get:**

> "To build a NestJS customer support chatbot handling 100k concurrent users, here's what you'll need:
> First, install NestJS using `npm install -g @nestjs/cli` and scaffold a new project..."

Problems:
- Starts with a basic NestJS install guide → irrelevant, wastes output tokens
- No mention of backpressure handling → hidden requirement missed entirely
- "token streaming" interpreted as WebSocket with no mention of SSE trade-offs
- Rephrase the same prompt slightly → get a completely different structure and priorities

**Consistency test — same intent, different wording:**

| Prompt variant | LLM output focus |
|---|---|
| "I want to build a NestJS chatbot..." | Starts with setup guide |
| "Build a NestJS chatbot, scale to 100k users..." | Jumps straight to architecture |
| "NestJS chatbot 100k concurrent, Redis, cheap" | Returns a concise bullet list |

→ **3 different structures. 3 different priorities. No guarantee which is correct.**

---

## ✅ With Cave Prompt — Compiled before execution

Cave Prompt reads the prompt and **externalizes its understanding** as a structured IR before anything executes.

**What it catches that the raw prompt hides:**

| Signal | Raw prompt | Cave Prompt |
|---|---|---|
| "I want to" | processed as content | flagged as `redundant_spans`, stripped |
| "token streaming" | ambiguous → guess | recorded in `ambiguities`, kept verbatim |
| stateless session requirement | never mentioned → missed | surfaced as `hidden_requirements` |
| backpressure handling | never mentioned → missed | surfaced as `hidden_requirements` |
| "cheap" | vague → guess | mapped to `cost` constraint, `priorities: cost efficiency` |

**Fidelity score: 0.95** — 95% of original meaning preserved and verifiable.

**Verbatim-protected spans:** `100k`, `Redis`, `NestJS`, `token streaming` — never paraphrased.

**The execution prompt sent to the main LLM:**

> Design and implement a customer support chatbot on NestJS handling 100k concurrent users. Hard requirements: Redis session caching, token streaming (SSE or WebSocket) with backpressure handling, cost-optimized infrastructure. Output priorities: scalability architecture, Redis session strategy, streaming implementation. Skip basic NestJS setup. Format: technical, concise.

No filler. No guessing. No boilerplate intro. The main LLM receives a precise brief.

---

## Full envelope (machine-readable)

```json
{
  "blocking_ambiguities": [],
  "semantic_analysis": {
    "intent": "Build a customer support chatbot on NestJS with high throughput, token streaming, and Redis session caching",
    "domain": "Backend engineering / conversational AI",
    "entities": ["NestJS", "Redis", "chatbot", "customer support"],
    "constraints": {
      "technical": ["NestJS framework", "Redis session cache", "token streaming"],
      "performance": ["100k concurrent users"],
      "cost": ["cheap infrastructure"]
    },
    "priorities": ["concurrency scalability", "cost efficiency", "streaming UX"],
    "response_preferences": {
      "tone": "technical",
      "verbosity": "concise"
    },
    "ambiguities": ["LLM provider not specified", "database/storage not specified"],
    "hidden_requirements": [
      "stateless or externalized session management",
      "backpressure handling for token streams",
      "horizontal scaling capability"
    ]
  },
  "optimized_ir": {
    "task_type": "system design + implementation guide",
    "execution_requirements": [
      "NestJS architecture for 100k concurrent users",
      "Redis session caching strategy",
      "SSE or WebSocket token streaming with backpressure",
      "cost-optimized infrastructure recommendations"
    ],
    "context_priority": {
      "high": ["scalability architecture", "Redis integration", "streaming implementation"],
      "low": ["basic NestJS setup", "boilerplate code", "introductory explanations"]
    },
    "reasoning_mode": "technical depth with concise output",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.82,
    "redundant_spans": ["I want to"],
    "low_information_spans": ["I need concise technical responses"],
    "execution_critical_spans": ["100k", "Redis", "NestJS", "token streaming"],
    "summary": "High semantic density (0.82). Noise: filler opener. All technical spans are execution-critical and preserved verbatim."
  },
  "verbatim_spans": ["100k", "Redis", "NestJS", "token streaming"],
  "fidelity_score": 0.95,
  "dropped_or_uncertain": [
    "concise — interpreted as concise technical depth, not brevity at cost of completeness"
  ],
  "execution_prompt": "Design and implement a customer support chatbot on NestJS handling 100k concurrent users. Hard requirements: Redis session caching, token streaming (SSE or WebSocket) with backpressure handling, cost-optimized infrastructure. Output priorities: scalability architecture, Redis session strategy, streaming implementation. Skip basic NestJS setup. Format: technical, concise."
}
```

---

*Demonstrates: verbatim preservation of "100k", "Redis", "NestJS", "token streaming", hidden requirement extraction (backpressure, stateless session), filler stripping.*
