# Diet Context

A small, reusable agent Skill for keeping long-running LLM workflows lean without throwing away information that may still matter.

Diet Context focuses on two complementary techniques:

1. **Tool-result offloading** — move bulky intermediate outputs out of active context and keep only a compact summary plus a retrievable reference.
2. **Structured handoff / compaction** — replace an accumulated conversation or workflow history with a concise operational handoff that preserves the state needed to continue.

The goal is not to make every prompt as short as possible. The goal is to reduce **repeated context carry** while preserving **recoverability**.

> In short: keep the working set small, keep the evidence retrievable, and avoid paying repeatedly for context that the next step probably does not need in full.

[한국어 설명](README.ko.md)

---

## Why this exists

Long agent workflows tend to accumulate context in a predictable way:

```text
system / workflow instructions
+ conversation history
+ previous tool calls
+ command output
+ logs
+ test results
+ research notes
+ file excerpts
+ previous agent reports
+ current task
```

Even when most of that material is no longer actively useful, it is often carried into later turns or passed to downstream agents. That creates several problems:

- input context grows continuously;
- the same old material may be paid for repeatedly;
- important facts get buried in noisy history;
- large logs and command output dominate the working context;
- downstream agents may receive far more information than their task requires;
- compaction performed too late can become lossy or expensive.

Diet Context introduces a simple policy layer that asks:

> What information must remain active, what can be summarized, and what can be stored externally for selective retrieval later?

It does **not** attempt to replace the model runtime, orchestrator, memory system, or tool framework.

---

## Core idea

### Without Diet Context

```text
Turn 1:  12k active context
Turn 2:  26k active context
Turn 3:  41k active context
Turn 4:  63k active context
Turn 5:  82k active context
```

A large amount of old context may be repeatedly carried forward even though only a fraction still matters.

### With Diet Context

```text
raw history / large tool output
            |
            v
   summarize or offload
            |
            v
 compact working context
            |
            +--> retrievable artifact for omitted detail
```

The optimization is worthwhile when:

```text
cost of repeatedly carrying full context
    >
cost of compaction/offload + selective retrieval + information-loss risk
```

That last term matters. Diet Context is deliberately conservative: if shortening the context is likely to lose critical precision, it should preserve more rather than compress aggressively.

---

# Technique 1: Tool-result offloading

Tool-result offloading handles **large intermediate outputs** such as:

- build logs;
- test output;
- shell command output;
- compiler diagnostics;
- package installation logs;
- database query results;
- long API responses;
- crawled pages;
- search dumps;
- machine-generated reports;
- verbose subagent output.

Instead of keeping the entire raw output in active context, preserve it in a retrievable artifact and keep a compact record in context.

### Example

Raw result:

```text
53 KB test output
812 lines
many repeated stack traces
three actual failures
```

Active context after offloading:

```text
Artifact: .diet-context/test-run-2026-10-08.txt
Produced by: pytest / test runner
Summary: 182 tests ran; 179 passed; 3 failed.
Key details:
- auth.test: token refresh assertion failed
- router.test: expected provider fallback was not selected
- cache.test: stale cache entry remained after reset
Retrieval hints:
- search for "FAILED"
- inspect auth.test stack around the first failure
- inspect router fallback assertion
```

The raw artifact still exists. The model simply stops carrying all of it by default.

## Selective retrieval

Offloading only helps if the next step does **not** immediately load the whole artifact again.

Prefer targeted retrieval:

```text
search / grep
specific line ranges
specific sections
head / tail
symbol lookup
error-only extraction
```

Instead of:

```text
read the entire 50k artifact back into context
```

If the entire artifact is genuinely required, read it. Diet Context is an optimization policy, not a prohibition.

---

# Technique 2: Structured handoff / compaction

Structured handoff handles **accumulated workflow history**.

A long session may contain planning, implementation attempts, corrections, reviews, tests, decisions, and repeated explanations. The next agent or stage often needs the current state, not the full narration of how that state was reached.

Diet Context converts that history into a compact operational handoff.

Recommended structure:

```text
Goal
- current objective and success condition

Constraints
- requirements, prohibitions, environment limits

Progress
Done
- completed work that affects future steps

In Progress
- unfinished work

Blocked
- blockers, if any

Key Decisions
- decisions that future work must respect
- rationale when the rationale still matters

Next Steps
- ordered actions for continuation

Critical Context
- facts that must not be lost

Artifacts / Relevant Files
- reference — why it matters / how to retrieve it
```

### Example

Before compaction:

```text
60k tokens of discussion, diffs, test output, revisions,
review feedback, repeated explanations, and command logs
```

After compaction:

```text
Goal
- Implement provider effort discovery for the router.

Constraints
- Probe only models whose effort mode is parameter/default.
- Skip embedded effort models.
- Preserve current model_scores schema compatibility.

Progress
Done
- Schema migration implemented.
- Provider resolver implemented.
- 38 regression tests pass.

In Progress
- Fallback discovery timeout handling.

Key Decisions
- Do not scan all providers; only registered combo models.
- Unsupported effort values are filtered before scoring.

Next Steps
1. Add provider timeout handling.
2. Run regression suite.
3. Review fallback behavior.

Artifacts / Relevant Files
- artifacts/provider-probe.log — raw provider responses; search by model id.
```

The next agent starts from the handoff instead of inheriting the entire historical transcript.

---

# Why the two techniques belong together

They solve different sources of context growth.

```text
Tool-result offloading
= reduce bulky intermediate evidence

Structured handoff / compaction
= reduce accumulated workflow history
```

Together:

```text
Long workflow
   |
   +--> large tool output -----> artifact storage
   |                               |
   |                               +--> selective retrieval later
   |
   +--> old workflow history ---> structured handoff
                                   |
                                   +--> compact new working baseline
```

---

# Adaptive behavior, not fixed thresholds

Diet Context intentionally avoids rigid rules such as:

```text
"offload anything above 8 KB"
"compact after exactly 20,000 tokens"
"handoff after every third agent"
```

A fixed threshold is easy to implement but often wrong in practice.

A 30 KB repetitive log may be safe to offload immediately, while a 12 KB legal diff or precise numerical report may need to stay intact.

The model should judge whether reduction is useful based on:

- expected reuse;
- information density;
- task length;
- number of downstream steps;
- ability to retrieve omitted material later;
- precision requirements;
- risk of losing important nuance;
- cost of summarizing and re-reading.

When uncertain, preserve more context.

---

# Relationship with orchestration skills

Diet Context is intentionally **not an orchestrator**.

It must not decide:

- whether another agent should be created;
- which agent should receive work;
- how many agents should run;
- which model an agent should use;
- whether work should be parallel or sequential;
- agent permissions;
- workflow stage order.

Those decisions belong to an orchestration layer such as a `dynamic-workflow` skill.

The separation is:

```text
dynamic-workflow
    -> who does what, when, and in what order

diet-context
    -> what context should travel with that work
```

This makes the two skills complementary rather than competing.

### Example

```text
Parent agent
   |
   | dynamic-workflow decides to delegate
   v
Coder agent
   |
   | Diet Context trims bulky logs and prepares a handoff
   v
Reviewer agent
```

Diet Context does not cause the delegation. It only optimizes the payload that crosses the boundary.

---

# Recoverable compression

The guiding principle is **recoverable compression**, not irreversible deletion.

Good reduction keeps:

- exact file paths;
- exact identifiers;
- exact commands when still relevant;
- unresolved questions;
- important numerical values;
- important error text;
- provenance;
- enough metadata to locate omitted evidence again.

Bad reduction turns this:

```text
Provider X returned HTTP 429 after 30.0 seconds while probing effort=high.
```

into this:

```text
There was some provider problem.
```

That is smaller, but no longer operationally useful.

---

# When Diet Context should *not* activate aggressively

Do not compact merely because the skill exists.

Keep the original context when:

- the task is short;
- the workflow is about to finish;
- exact wording matters;
- formatting or ordering matters;
- detailed numerical evidence must remain visible;
- a summary would force the next step to reread nearly everything;
- external artifact storage is unavailable or unreliable;
- retrieval would be more expensive than simply keeping the information active;
- evidence is too sensitive to place in a shared or persistent artifact location.

Sometimes the most efficient context optimization is doing nothing.

---

# Failure safety

Diet Context must not claim information was preserved unless it actually was.

If an artifact write fails:

```text
DO NOT discard the raw result.
```

If a handoff cannot preserve critical state:

```text
DO NOT compact aggressively.
```

If retrieval metadata is uncertain:

```text
preserve enough original detail to recover safely.
```

The skill should trade efficiency for correctness whenever the two conflict.

---

# Security and privacy

Offloading creates persistent or semi-persistent artifacts, so the storage destination matters.

Before offloading sensitive material, consider:

- whether the artifact location is shared;
- whether secrets, credentials, personal data, or proprietary data are present;
- whether the runtime automatically uploads files elsewhere;
- whether downstream agents have permission to access the artifact;
- whether the artifact should be deleted after use.

Context reduction should never become an excuse to weaken data handling practices.

---

# Installation

This repository is structured so the **repository root is the Skill**.

```text
diet-context/
├── README.md
├── README.ko.md
├── SKILL.md
└── agents/
    └── openai.yaml
```

A compatible Skill runtime can install the repository by placing or linking the repository root into its global Skill directory.

Because runtimes differ, Diet Context does not hard-code a universal installation path.

A generic installer should:

1. clone or download this repository;
2. detect the runtime's global Skill directory;
3. place or link the repository root there;
4. verify that `SKILL.md` is discoverable;
5. start a fresh session if the runtime only loads Skill metadata at session startup.

### Hermes-friendly install request

After publishing this repository, a user can give Hermes the repository URL and say something like:

```text
Read the README and SKILL.md in this repository.
Understand the role and operating boundaries of Diet Context,
then install it as a global Skill for this environment.
Do not modify dynamic-workflow; Diet Context should complement it.
Verify that the Skill is discoverable after installation.
```

That lets Hermes inspect the public documentation before deciding how to install it in its own environment.

---

# Skill behavior summary

Diet Context should:

- monitor context growth qualitatively;
- recognize noisy or low-density intermediate results;
- preserve large raw outputs externally when practical;
- keep summaries and stable references in active context;
- compact long workflow history into a structured handoff when useful;
- preserve exact operational details that still matter;
- retrieve omitted evidence selectively;
- stay out of orchestration decisions;
- avoid compression when expected savings do not justify the risk or overhead.

Diet Context should **not**:

- spawn agents by itself;
- change workflow topology;
- select models;
- silently delete unrecoverable evidence;
- summarize precise data into vague prose;
- force compaction on small tasks;
- reload entire offloaded artifacts by default;
- override higher-priority instructions.

---

# Design philosophy

Diet Context is based on a simple observation:

> Agent systems often waste context not because every piece of information is useless, but because too much information remains active at the same time.

The answer is therefore not merely "delete more." It is to separate:

```text
active working context
from
recoverable background evidence
```

and to maintain a compact operational state that can continue the work without carrying the complete history of the work.

---

# Inspiration

The design was inspired in part by context-efficiency ideas observed in the open-source [Mixdog](https://github.com/tribgames/mixdog) project, especially patterns around large tool-output handling, context compaction, and selective retrieval.

Diet Context is an independent, lightweight policy Skill rather than a port of the Mixdog runtime. It contains no Mixdog runtime code and does not require Mixdog.

---

# Status

Early / experimental.

The Skill is intentionally small and policy-oriented. Real-world feedback should drive future refinements, especially around:

- compaction timing;
- retrieval quality;
- agent-to-agent handoff patterns;
- artifact lifecycle management;
- context-loss detection;
- integration with different agent runtimes.