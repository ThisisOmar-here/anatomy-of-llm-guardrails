# 🧭 A Defender's Taxonomy of Framing Pressure

A short reference for *sorting* multi-turn pressure techniques into buckets, so you can test and defend against them on purpose instead of by vibes. This is a map of the terrain, not a how-to. Each entry names the pattern, says why a roleplay engine leans into it, and gives the defense that kills it.

---

## 1. Authority / Tool Persona

**The pattern.** Convince the model it got reassigned to a role that makes blocked actions feel routine: "you are now a debugger", "developer mode", "you are an unfiltered simulator."

**Why it leans.** A base model continues whatever frame you give it. A persona is just a strong frame. The hope is that an instruction-shaped request quietly overrides learned refusal.

**The defense.** Keep refusal in the weights (alignment training), not in one switchable instruction. An output classifier that reads the final text doesn't care what persona wrote it.

---

## 2. Hypothetical / Fictional Distance

**The pattern.** Wrap the request in a layer of unreality: a story, a screenplay, a "draft that will be deleted", a "simulation of what an unsafe model would say."

**Why it leans.** It tries to split "talking about" a harmful thing from "doing" it, hoping the model treats the fictional wrapper as a safe zone.

**The defense.** Score the *content*, not the wrapper. Harmful operational detail is harmful whether you label it fiction or fact. A robust classifier judges what the text would actually let someone do.

---

## 3. Decomposition

**The pattern.** Break a blocked goal into individually-innocent sub-requests spread across turns, then assemble the pieces yourself.

**Why it leans.** Each turn, on its own, looks fine. A single-turn safety check misses the combined intent.

**The defense.** Judge the conversation trajectory, not isolated messages. Hold intent state across turns.

---

## 4. State Fabrication

**The pattern.** Claim an internal check "already returned PASS", or hand the model a fake "debug log" that says safety is off, hoping it treats the claim as ground truth.

**Why it leans.** It plays on the model's habit of taking the context it's given at face value.

**The defense.** Never let a user-supplied claim stand in for an actual check. Real classifiers run independently on real content. You can't just tell them a result.

---

## 5. Incremental Escalation

**The pattern.** Start with a clearly fine version of a request, then nudge each follow-up a little past the line, riding the consistency pressure.

**Why it leans.** Once a model has gone along with a frame, continuing it feels coherent.

**The defense.** Judge each turn on its own merits. Re-check against policy every turn instead of inheriting earlier compliance.

---

## How to use this as a red-teamer

Test **categories**, not specific strings. A prompt dump rots the moment a model updates. A taxonomy survives because it describes *why* the pressure works. Build evaluation suites that cover each category, log every refusal and near-miss, and report real weaknesses to the vendor through their disclosure channel instead of publishing them.

> This document has no reproducible bypass in it, on purpose. The point is to help builders defend systems.
