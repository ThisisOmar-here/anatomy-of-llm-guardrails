# Defender's Taxonomy of Prompt-Framing Pressure

This document gives defenders a concise vocabulary for classifying prompt-pressure patterns. It is not a bypass guide. Each entry describes the pattern at a conceptual level, explains why it can influence a language model, and lists practical defenses.

Use this taxonomy to design evaluations, review logs, and discuss failures without relying on fragile prompt strings.

---

## 1. Authority or Tool Persona

**Pattern**

The user assigns the model a role that makes restricted behavior sound routine, such as a debugger, simulator, evaluator, internal tool, or diagnostic assistant.

**Why it can work**

Language models are strongly shaped by the frame of the conversation. A technical persona can make an unsafe request appear like a normal step inside a tool workflow.

**Defenses**

- Treat persona changes as untrusted user text.
- Keep safety policy independent from roleplay framing.
- Re-check output content even when the model claims to be simulating or debugging.

---

## 2. State Fabrication

**Pattern**

The user supplies fake internal state and asks the model to continue as if it were true. This can include fabricated classifier results, fake debug logs, invented permissions, or claims that a safety check already passed.

**Why it can work**

Models often try to maintain coherence with the conversation context. If fake state is written in a technical format, the model may treat it as part of the scenario rather than as an untrusted claim.

**Defenses**

- Never let user-provided text define runtime state.
- Keep trusted state in application code, not in the prompt.
- Make classifiers and tool permissions independent from conversation content.
- Test prompts where the user claims that safety checks, permissions, or tools have already allowed the request.

---

## 3. Hypothetical or Fictional Distance

**Pattern**

The user wraps the request in fiction, simulation, academic framing, or "for a story" language.

**Why it can work**

The wrapper attempts to separate the content from its real-world use. The model may focus on the fictional frame instead of the operational effect of the answer.

**Defenses**

- Evaluate what the output enables, not only how the request is framed.
- Block operational harmful detail even when it appears in fiction or simulation.
- Allow safe high-level discussion when appropriate, but avoid procedural guidance.

---

## 4. Decomposition

**Pattern**

The user splits a restricted goal into small requests that look harmless in isolation, then combines the answers outside the model.

**Why it can work**

Single-turn filters may not see the broader intent when each message only asks for a small piece.

**Defenses**

- Score conversation trajectory, not only the latest turn.
- Track unresolved user goals across turns.
- Evaluate whether a sequence of benign-looking requests forms an unsafe workflow.

---

## 5. Incremental Escalation

**Pattern**

The user starts with an allowed topic, then gradually asks for more specific, more operational, or more policy-sensitive details.

**Why it can work**

Once the model has accepted a frame, continuing in the same direction can feel coherent. The risk increases when the system inherits earlier compliance instead of re-evaluating each turn.

**Defenses**

- Re-check policy boundaries on every turn.
- Treat earlier safe answers as context, not permission.
- Log transitions from high-level educational discussion to operational detail.

---

## 6. Output-Filter Evasion Framing

**Pattern**

The user asks the model to produce a draft, hidden answer, intermediate artifact, encoded text, or "internal" version before applying safety checks.

**Why it can work**

The request tries to move unsafe content into a supposedly temporary or non-user-facing form. In a plain chat setting, however, any generated text is still output.

**Defenses**

- Treat drafts, simulations, hidden answers, and intermediate artifacts as outputs.
- Run output checks on every generated artifact.
- Do not expose chain-of-thought or internal draft content as a way to satisfy unsafe requests.

---

## 7. Policy Confusion

**Pattern**

The user asks the model to explain, debug, quote, transform, translate, or evaluate unsafe content in a way that pressures the model to reproduce it.

**Why it can work**

The task appears to be about analysis rather than assistance, but the model may still emit the unsafe details.

**Defenses**

- Separate allowed analysis from disallowed reproduction.
- Summarize unsafe content at a safe abstraction level.
- Redact operational details when transforming or reviewing risky text.

---

## How To Use This Taxonomy

1. Build evaluation cases for each category.
2. Keep public examples non-operational.
3. Test across multiple turns, not only single prompts.
4. Log refusals and near misses with category labels.
5. Update defenses based on categories, not exact prompt wording.
6. Use responsible disclosure for specific vendor or model failures.

The aim is to improve safety engineering. Do not publish prompts that directly reproduce harmful behavior.
