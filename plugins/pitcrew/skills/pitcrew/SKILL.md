---
name: pitcrew
description: Run the Pit Crew coding workflow. Use for implementation or refactoring tasks where the user wants high-quality main-thread decisions, bounded Luna implementation under Sol supervision, and final main-thread approval. Trigger when the user says Pit Crew, crew chief, Luna implement with Sol supervision, multi-model coding workflow, or asks to keep expensive main-model reasoning out of implementation and correction loops. Do not use for simple questions, prose-only tasks, or when the user explicitly asks for a single-agent workflow.
license: MIT
---

# Pit Crew

Run one bounded execution cell:

```text
Crew Chief -> [ Inspector <-> Mechanic ] -> Crew Chief
```

The parent/main thread is the Crew Chief. Pit Crew v1 is designed for an Astra-class main thread, but never claim the active model is Astra unless the runtime actually identifies it that way.

## V1 purpose

Keep architecture and final judgment in the main thread while moving implementation, implementation supervision, and low-level correction loops into the execution cell.

The target shape is:

```text
Crew Chief
    -> contract
Inspector supervises Mechanic
    -> PASS or ESCALATE
Crew Chief
    -> final judgment
```

The Crew Chief should normally see the implementation only twice: once when establishing the contract and once after Inspector `PASS` for final review.

Do not add speculative cost optimizations. In v1:

- do not require Sol repository search before the contract exists
- do not build or require context packs
- do not add MCP servers
- do not add lifecycle hooks
- do not invent automatic reasoning-effort routing
- do not turn the workflow into a general multi-agent framework

## Roles

### Crew Chief

The parent thread owns:

- user intent and acceptance criteria
- architecture and responsibility boundaries
- scope of the implementation
- the implementation contract sent into the execution cell
- decisions that genuinely require escalation
- final architecture and intent review
- final approval

During the execution loop, the Crew Chief must not perform Inspector work merely because the Mechanic is incomplete, blocked, or wrong.

Unless the Inspector returns `ESCALATE`, the Crew Chief should not:

- inspect the working diff for local implementation defects
- diagnose build, patch, staging, syntax, or local logic failures
- issue implementation corrections directly to the Mechanic
- repeatedly re-read the same implementation state between Mechanic attempts

If the runtime requires the main thread to relay messages between subagents, relay the packets mechanically. Do not turn that routing step into a substantive implementation review.

The Crew Chief must not outsource a real design decision merely to save tokens.

### Mechanic

Prefer `gpt-5.6-luna` when the runtime supports explicit subagent model selection.

The Mechanic:

- edits code or repository state within the contract
- chooses practical implementation tools and commands for the bounded task
- builds or runs targeted tests when practical
- follows existing project conventions
- does not redesign the architecture
- does not expand scope without escalation
- does not commit or push unless the user explicitly asked for it

A failed or incomplete Mechanic attempt is not automatically a Crew Chief event. Return the concrete state to the Inspector first.

The Mechanic returns a concise report containing:

```text
MECHANIC REPORT
changed:
- <files, symbols, repository state, or attempted operation>
validated:
- <build/tests/checks>
uncertainty:
- <none or concrete unresolved point>
```

### Inspector

Prefer `gpt-5.6-sol` when the runtime supports explicit subagent model selection.

The Inspector is the execution supervisor and quality gate for the Mechanic. It does not edit files itself, but it is not limited to passively reviewing a finished diff.

The Inspector may inspect:

- incomplete or failed Mechanic attempts
- the current working tree, staged state, and diff
- build or test failures
- narrowly necessary surrounding code
- the implementation contract and prior local-fix history

Within the approved contract, the Inspector may reason independently about the correct bounded result, diagnose why an attempt failed, and tell the Mechanic to use a different concrete implementation method when there is one clear answer. For mechanical or repository-state work, this can include recommending deterministic commands, patches, or validation steps rather than repeating an unproductive reasoning loop.

The Inspector must not:

- edit files or repository state directly
- redesign architecture
- expand scope
- choose between multiple reasonable product or architecture decisions

Review and supervise in this order:

1. implementation contract compliance
2. correctness and regressions
3. unrequested or out-of-scope changes
4. build, test, patch, staging, or other concrete execution failures
5. use of existing APIs versus needless duplicate logic
6. unnecessary abstractions or dead code introduced by the patch
7. naming and local consistency where there is one obvious answer
8. validation evidence and obvious gaps

The Inspector must return exactly one verdict class:

```text
PASS
```

or

```text
LOCAL_FIX
- location: <file/symbol/state/operation>
  problem: <concrete problem>
  correction: <bounded required correction>
```

or

```text
ESCALATE
- location: <file/symbol/state/contract section>
  reason: <why a design/scope decision is required>
  decision_needed: <the smallest question the Crew Chief must answer>
```

Do not mix `LOCAL_FIX` and `ESCALATE` in one verdict. If any item requires escalation, return `ESCALATE` and include other concrete findings only when they materially support that decision.

## What counts as LOCAL_FIX

The Inspector may keep the problem inside the execution cell whenever there is one clear answer inside the existing contract, for example:

- a required behavior was plainly omitted
- compile or syntax failure
- patch, staging, line-ending, or repository-state error with one clear correction
- clear local logic error
- unnecessary out-of-scope edit that can simply be reverted
- dead code introduced by the patch
- duplicate implementation where the already-approved existing API covers the same job
- a local naming or consistency mistake with one established project convention
- the Mechanic used an unproductive method and a clearly bounded deterministic method can produce the same approved result

The Inspector must make the correction request as small and executable as possible.

## What must ESCALATE

Escalate instead of locally fixing when resolution would require any of the following:

- moving responsibility between classes or systems
- adding, removing, or materially changing a public API beyond the approved contract
- introducing a new object, abstraction, layer, or dependency
- expanding implementation scope
- changing the mockup, requirement, or acceptance criteria
- choosing among multiple reasonable designs
- contradicting a Crew Chief decision
- resolving a mismatch between the contract and the real codebase where intent is unclear

A Mechanic failure by itself is not an escalation. Escalate only when the missing decision actually belongs to the Crew Chief.

## Workflow

### 1. Crew Chief: establish the contract

Read the user request, supplied mockup, and enough existing code to make the architecture decision.

Produce an internal bounded implementation packet. It should contain only what the execution cell needs:

```text
IMPLEMENTATION PACKET
objective: <one clear objective>
scope:
- <files/systems/state allowed to change>
contract:
- <required behavior>
- <required preserved behavior>
constraints:
- <forbidden redesigns or out-of-scope work>
validation:
- <build/tests/manual checks expected>
```

Do not send the full parent conversation unless the runtime makes that unavoidable. Prefer fresh or minimally forked subagent contexts plus the packet.

### 2. Start the execution cell

Delegate the implementation packet to the Mechanic and establish the Inspector as the supervisor for that packet.

If explicit model routing is available:

- request `gpt-5.6-luna` for the Mechanic
- request `gpt-5.6-sol` for the Inspector

Do not set reasoning effort in v1 unless the user explicitly requested one; effort tuning is intentionally deferred until usage data exists.

The important rule is not which subagent physically starts first. The important rule is that routine implementation success, failure, and correction stay inside the Mechanic/Inspector cell until `PASS` or a real `ESCALATE`.

### 3. Mechanic: attempt the work

The Mechanic implements or performs the bounded task and returns a `MECHANIC REPORT`.

Do not require the Mechanic to present a supposedly perfect result before the Inspector can participate. An incomplete attempt, failed validation, or blocked repository operation should go to the Inspector rather than triggering Crew Chief review.

### 4. Inspector: supervise and gate

Give the Inspector:

- the implementation contract
- the latest Mechanic report
- access to the current diff, working tree, staged state, or validation output as relevant
- prior `LOCAL_FIX` history when useful

The Inspector returns `PASS`, `LOCAL_FIX`, or `ESCALATE`.

If explicit model routing is available, request `gpt-5.6-sol`. Prefer a fresh or minimally forked Inspector context with only the evidence needed to supervise the current execution state.

### 5. Local correction loop

If the Inspector returns `LOCAL_FIX`, route only that fix packet back to the Mechanic.

The Mechanic applies only those corrections and returns an updated report. Then the Inspector checks the new current state again.

```text
Mechanic attempt
      -> Inspector
LOCAL_FIX -> Mechanic
      -> Inspector
LOCAL_FIX -> Mechanic
      -> Inspector
PASS
```

The Crew Chief stays out of this loop.

Do not loop mechanically forever. If the same issue recurs, the Inspector should diagnose whether a different bounded implementation method can solve it. If the remaining problem actually requires a design, contract, or scope decision, return `ESCALATE`.

### 6. Escalation

Only `ESCALATE` returns substantive control to the Crew Chief during execution.

The Crew Chief reads only the minimum original code and evidence needed to make the missing decision, updates the contract, and returns the task to the execution cell.

Do not use escalation merely because the Mechanic made a mistake, a command failed, or a local correction needs another attempt.

### 7. Crew Chief final review

Only after the Inspector returns `PASS`, the Crew Chief performs the final review.

This is the first normal point after contract creation where the Crew Chief should inspect the completed implementation.

The final review is not a duplicate Sol pass. Check the things only the Crew Chief owns:

- does the final code actually satisfy the user's intent?
- does responsibility live in the right place?
- did the implementation preserve the intended architecture?
- did the execution loop accidentally narrow or distort the original requirement?
- is any change unnecessary at the architecture level?

The Crew Chief may reject an Inspector `PASS`.

If the final review finds only a local implementation defect with one clear correction, send the result back to the Inspector-led execution cell rather than correcting the Mechanic directly. If the problem is architectural or changes the contract, the Crew Chief updates the contract first.

### 8. Report to the user

Report:

- what was implemented
- what was validated
- whether the Inspector returned `PASS`
- any unresolved limitation or runtime routing limitation

Do not expose internal chain-of-thought. Do not claim a specific subagent model was used unless the runtime actually routed that model.

## Runtime fallback

Codex subagent and model-selection surfaces can differ by client/runtime.

If explicit Luna/Sol selection is unavailable:

1. preserve the Mechanic and Inspector role contracts
2. use available subagents only if delegation itself is supported
3. preserve the rule that the Crew Chief does not perform routine intermediate implementation review
4. state the routing limitation in the final report
5. never pretend a generic or inherited subagent was Luna or Sol

If the runtime cannot let subagents communicate directly but can invoke them separately, the main thread may relay the Mechanic report and Inspector verdict without independently reviewing the implementation state.

If subagents are unavailable entirely, do not simulate multiple agents in prose. Continue as a normal single-agent implementation and say Pit Crew delegation was unavailable in that runtime.
