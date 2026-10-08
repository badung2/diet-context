---
name: diet-context
description: Reduce context growth during long or tool-heavy workflows by offloading large/noisy tool results to retrievable artifacts and replacing accumulated history with structured handoffs when useful. Use when conversations, agent workflows, coding sessions, research tasks, test/build logs, command output, or other intermediate results are becoming costly or unwieldy to carry forward. Apply adaptively based on expected reuse, information density, task length, and loss risk rather than fixed token, byte, or line thresholds. Compose with orchestration skills without deciding delegation, agent roles, parallelism, or execution order.
---

# Diet Context

Keep active context small without discarding information that may still be needed. Use two complementary techniques: tool-result offloading and structured handoff/compaction.

## Operating Boundary

Treat this skill as context-management policy only.

- Do not decide whether to create or delegate to another agent.
- Do not change agent roles, model selection, permissions, parallelism, workflow stages, or execution order.
- Do not invoke a delegation tool solely because this skill is active.
- When another workflow or orchestration skill delegates work, optimize only the context passed into, between, or back from those steps.
- Preserve higher-priority workflow instructions when they conflict with context reduction.

This separation allows orchestration skills to answer **who does what and when**, while this skill answers **what context should travel with the work**.

## Decide Adaptively

Do not use fixed size thresholds. Judge whether reduction is worthwhile from the current task.

Prefer reduction when one or more of these are true:

- Tool output is long, repetitive, noisy, or mostly diagnostic.
- The same historical context would otherwise be carried through several future turns or agents.
- Only a small portion of a large result is likely to matter later.
- A workflow has accumulated many completed steps whose details can be summarized safely.
- Full raw output can be stored and selectively retrieved if needed.

Prefer keeping context intact when one or more of these are true:

- The task is short and likely to finish in the next turn or two.
- Exact wording, ordering, formatting, numerical detail, or legal/technical precision is important.
- The information cannot be stored or retrieved reliably later.
- A summary would create meaningful ambiguity or force the next step to reread most of the original anyway.
- The cost of producing and validating a handoff would exceed the likely savings.

When uncertain, preserve more context rather than compressing aggressively.

## Technique 1: Tool-Result Offloading

Offload a large intermediate result when its full contents are not needed in active context.

1. Preserve the complete raw result in a retrievable artifact when a filesystem or artifact mechanism is available.
2. Keep a compact in-context record containing:
   - what produced the artifact;
   - a concise result summary;
   - important errors, warnings, decisions, or values;
   - the artifact path or stable reference;
   - useful retrieval hints such as relevant filenames, symbols, line ranges, search terms, or sections when known.
3. Include a short excerpt only when it materially helps the next step.
4. Retrieve selectively later. Prefer search, grep, targeted ranges, tail/head, or section reads over loading the whole artifact.
5. Read the full artifact only when the task genuinely requires the complete content.

Do not pretend data was preserved if artifact storage failed. In that case, retain the necessary raw content in context or use a less aggressive reduction.

### Offload Record

Use a compact record shaped like this when useful:

```text
Artifact: <path-or-reference>
Produced by: <tool/action>
Summary: <what happened and why it matters>
Key details:
- <important detail>
- <important detail>
Retrieval hints: <search terms / ranges / sections>
```

Do not spend tokens filling fields that add no value.

## Technique 2: Structured Handoff / Compaction

Compact accumulated working history when future work needs the state of the task more than the full transcript.

Create a handoff that preserves operational state, not conversational narration. Include only sections that are relevant:

```text
Goal
- <current objective and success condition>

Constraints
- <important requirements, prohibitions, environment limits>

Progress
Done
- <completed work that affects future steps>

In Progress
- <current unfinished work>

Blocked
- <blockers, if any>

Key Decisions
- <decision + rationale when the rationale matters>

Next Steps
- <ordered actionable continuation>

Critical Context
- <facts that must not be lost>

Artifacts / Relevant Files
- <path or reference> — <why it matters / what to retrieve>
```

Preserve exact identifiers, commands, paths, API names, error text, numeric values, user constraints, and unresolved questions when they remain important. Do not replace precise facts with vague prose.

After compaction, use the handoff as the working baseline instead of repeatedly carrying the full historical transcript when the runtime allows this.

## Handoff Between Agents or Stages

When another system or skill has already decided to hand work to a new agent or stage:

1. Pass the smallest sufficient handoff for that recipient's task.
2. Include exact task scope and success criteria.
3. Include only the artifacts and prior decisions relevant to that scope.
4. Point to raw artifacts instead of embedding them when selective retrieval is practical.
5. Do not hide uncertainty. Mark assumptions, unresolved issues, and unverified conclusions explicitly.
6. Do not force the recipient to reread a large artifact merely to discover the basic task state; summarize the important parts first.

## Preserve Recoverability

Optimize for **recoverable compression**, not irreversible deletion.

- Keep raw high-volume evidence externally when practical.
- Keep enough metadata to find the evidence again.
- Separate facts from interpretations in summaries when that distinction matters.
- Retain provenance: record which tool, file, command, or stage produced important information.
- If later evidence conflicts with the handoff, trust the authoritative source and update the handoff.

## Avoid False Economy

Do not offload or compact merely because the capability exists. Account for the extra work required to summarize, store, retrieve, and re-read information.

The optimization is worthwhile when:

```text
cost of repeated full-context carry
    >
cost of compaction/offload + selective retrieval + loss risk
```

Favor the simpler path for small tasks. Favor offloading and handoff as context grows, repetition increases, or multiple downstream steps would otherwise inherit the same bulky history.

## Completion Check

Before handing off or compacting, verify that the reduced context still answers:

- What is the current goal?
- What has already been done?
- What constraints must remain true?
- What important decisions were made?
- What is unresolved?
- What should happen next?
- Where can omitted raw evidence be retrieved?

If any answer necessary for continuation is missing, preserve or restore that information before proceeding.