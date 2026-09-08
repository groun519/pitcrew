---
name: pitcrew
description: Run the Pit Crew coding workflow. Use for implementation or refactoring tasks where the user wants high-quality main-thread decisions, bounded Luna implementation under Sol supervision, and final main-thread approval. Supports Quality and Balanced model profiles. Trigger when the user says Pit Crew, Pit Crew Quality, Pit Crew Balanced, crew chief, Luna implement with Sol supervision, multi-model coding workflow, or asks to keep expensive main-model reasoning out of implementation and correction loops. Do not use for simple questions, prose-only tasks, or when the user explicitly asks for a single-agent workflow.
license: MIT
---

# Pit Crew

Run one bounded execution cell:

```text
Crew Chief -> [ Inspector <-> Mechanic ] -> Crew Chief
```

The parent/main thread is the Crew Chief. The selected Pit Crew profile defines the intended Crew Chief model/effort and the requested Inspector/Mechanic model/effort. Never claim that a requested model or effort is active unless the runtime actually identifies or confirms it.

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

While the execution cell is healthy, the Crew Chief should also stay dormant. Do not wake a large parent context merely to ask whether a worker is still running.

## Profiles

Pit Crew v0.1.4 has two fixed profiles. Do not add an Economy profile and do not silently choose a profile from perceived task difficulty.

| Profile | Crew Chief | Inspector | Mechanic |
|---|---|---|---|
| `Quality` | Astra / `high` | `gpt-5.6-sol` / `xhigh` | `gpt-5.6-luna` / `xhigh` |
| `Balanced` | `gpt-5.6-sol` / `xhigh` | `gpt-5.6-sol` / `high` | `gpt-5.6-luna` / `high` |

Profile intent:

- `Quality` minimizes avoidable quality loss. Use the strongest Crew Chief route and give both execution roles generous reasoning budgets.
- `Balanced` removes Astra from ordinary work while keeping Sol in every judgment/review role. Luna remains the bounded implementation role with a high reasoning budget.
- Luna is deliberately not reduced to medium or low in either profile. A stronger first implementation can reduce Sol correction and re-review loops.
- Models below Sol are not used for Crew Chief or Inspector judgment in these v1 profiles.

Profile selection:

- `Use Pit Crew Quality ...` selects `Quality`.
- `Use Pit Crew Balanced ...` selects `Balanced`.
- A plain `Use Pit Crew ...` selects `Quality` to preserve the original high-quality behavior.
- Do not automatically downgrade from `Quality` to `Balanced` or upgrade from `Balanced` to `Quality` based on your own complexity estimate.

The Crew Chief is the already-open main thread. Pit Crew does not silently replace that parent model or its effort setting.

Before delegation:

- For `Quality`, the user should start the task with Astra at `high` effort.
- For `Balanced`, the user should start the task with Sol at `xhigh` effort.
- If the runtime can identify the current main model/effort and it conflicts with the selected profile, do not claim the profile is active. Tell the user which main setting the profile expects before starting execution.
- If the runtime cannot identify the current main model/effort, proceed with the requested profile but treat the Crew Chief route as unverified and report that limitation at the end.

When explicit subagent model and effort routing is supported, request the Inspector and Mechanic model/effort exactly from the selected profile. If model selection is supported but effort selection is not, preserve the requested models, use the runtime's available effort behavior, and report the limitation rather than pretending the requested effort was applied.

Do not add speculative cost optimizations. In v1:

- do not require Sol repository search before the contract exists
- do not build or require context packs
- do not add MCP servers
- do not add lifecycle hooks
- do not automatically select profiles from task complexity
- do not turn the workflow into a general multi-agent framework
- do not add extra polling machinery beyond the runtime's wait primitive

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
- repeatedly poll worker liveness at short intervals when no meaningful state has changed

If the runtime requires the main thread to relay messages between subagents, relay packets only when there is a meaningful state transition to relay. Do not turn a timer wake-up, empty wait result, or liveness check into a substantive parent-model turn.

The Crew Chief must not outsource a real design decision merely to save tokens.

### Mechanic

Use the Mechanic model/effort from the selected profile when the runtime supports explicit routing.

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

Use the Inspector model/effort from the selected profile when the runtime supports explicit routing.

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
- repeatedly wake just to ask whether the Mechanic is still running when no new execution state exists

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

A Mechanic failure by itself is not an escalation. A wait timeout or lack of new worker output is not an escalation either. Escalate only when the missing decision actually belongs to the Crew Chief.

## Waiting and wake-up policy

Idle orchestration should not consume reasoning turns when the runtime can avoid it.

Pit Crew uses one explicit experimental wait value for healthy Codex subagent waits:

```text
wait_agent timeout_ms = 1200000
```

That is 20 minutes. When `wait_agent` exposes `timeout_ms`, pass `1200000` explicitly instead of omitting the argument and falling back to a short runtime default. On current Codex wait implementations, completion or relevant activity can return before the timeout, so this value is a maximum wait rather than a forced 20-minute delay.

Use these rules while waiting for a Mechanic or Inspector:

- Prefer event-driven, completion-driven, or meaningful-state-change wake-up when the runtime exposes it.
- When `wait_agent` supports `timeout_ms`, use `timeout_ms=1200000` for a healthy worker wait.
- Worker silence means only that no new state is available. It does not mean failure.
- A wait timeout means only that the wait returned without completion. It is not evidence that the worker is stuck, wrong, or should be replaced.
- Do not interrupt, duplicate, restart, or replace a healthy worker merely because a wait expired.
- Do not repeat short waits when one supported long wait can cover the same healthy execution period.
- Treat 20 minutes as an experimental Pit Crew tuning value, not a universal law. Revisit it when measurements or runtime semantics justify a change.
- Wake the Crew Chief substantively only for `PASS`, `ESCALATE`, a meaningful execution-state change that actually needs Crew Chief action, a user interruption, or a runtime failure that requires a decision.
- The same principle applies inside the execution cell: the Inspector should supervise meaningful Mechanic states, not spend repeated reasoning turns checking liveness.

If the runtime does not expose `timeout_ms`, rejects the requested value, or has materially different wait semantics, use the longest practical supported wait and preserve the no-busy-polling rule.

If the runtime itself wakes the main thread periodically and that behavior cannot be disabled, keep those wake-ups mechanical: wait or relay only. Do not inspect repository state or reconsider the implementation unless new evidence actually requires it.

## Workflow

### 1. Crew Chief: establish the contract

Resolve the profile first, verify the Crew Chief route when the runtime exposes it, then read the user request, supplied mockup, and enough existing code to make the architecture decision.

Produce an internal bounded implementation packet. It should contain only what the execution cell needs:

```text
IMPLEMENTATION PACKET
profile: <Quality or Balanced>
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

When explicit model and effort routing is available, use the selected profile exactly:

```text
QUALITY
Crew Chief : Astra / high
Inspector  : gpt-5.6-sol / xhigh
Mechanic   : gpt-5.6-luna / xhigh

BALANCED
Crew Chief : gpt-5.6-sol / xhigh
Inspector  : gpt-5.6-sol / high
Mechanic   : gpt-5.6-luna / high
```

The important rule is not which subagent physically starts first. The important rule is that routine implementation success, failure, and correction stay inside the Mechanic/Inspector cell until `PASS` or a real `ESCALATE`.

After delegation, do not keep the Crew Chief active merely to monitor liveness. When waiting through a Codex `wait_agent` surface that accepts `timeout_ms`, explicitly request `1200000` rather than relying on the short default.

### 3. Mechanic: attempt the work

The Mechanic implements or performs the bounded task and returns a `MECHANIC REPORT`.

Do not require the Mechanic to present a supposedly perfect result before the Inspector can participate. An incomplete attempt, failed validation, or blocked repository operation should go to the Inspector rather than triggering Crew Chief review.

### 4. Inspector: supervise and gate

Give the Inspector:

- the selected profile
- the implementation contract
- the latest Mechanic report
- access to the current diff, working tree, staged state, or validation output as relevant
- prior `LOCAL_FIX` history when useful

The Inspector returns `PASS`, `LOCAL_FIX`, or `ESCALATE`.

Prefer a fresh or minimally forked Inspector context with only the evidence needed to supervise the current execution state.

Do not ask the Inspector to wake repeatedly between meaningful Mechanic states. If the Inspector must wait on a healthy Mechanic through `wait_agent` and `timeout_ms` is supported, use `1200000` there too.

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

The Crew Chief stays out of this loop and should remain dormant while the loop is healthy.

Do not loop mechanically forever. If the same issue recurs, the Inspector should diagnose whether a different bounded implementation method can solve it. If the remaining problem actually requires a design, contract, or scope decision, return `ESCALATE`.

### 6. Escalation

Only `ESCALATE` returns substantive control to the Crew Chief during execution.

The Crew Chief reads only the minimum original code and evidence needed to make the missing decision, updates the contract, and returns the task to the execution cell.

Do not use escalation merely because the Mechanic made a mistake, a command failed, a wait timed out, a worker has been quiet, or a local correction needs another attempt.

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

- selected profile
- what was implemented
- what was validated
- whether the Inspector returned `PASS`
- whether the requested Crew Chief, Inspector, and Mechanic model/effort routes were confirmed or unverified
- any unresolved runtime routing limitation

Do not expose internal chain-of-thought. Do not claim a specific model or reasoning effort was used unless the runtime actually routed or identified it.

## Runtime fallback

Codex subagent, effort, wait, and model-selection surfaces can differ by client/runtime.

If explicit Luna/Sol selection or effort selection is unavailable:

1. preserve the selected profile's role contracts
2. use available subagents only if delegation itself is supported
3. request the selected profile's model and effort wherever the runtime exposes those controls
4. preserve the rule that the Crew Chief does not perform routine intermediate implementation review
5. preserve the no-busy-polling rule as far as the runtime allows
6. when `wait_agent` supports `timeout_ms`, still request `1200000` for healthy waits
7. state exactly which model/effort routes could not be verified or applied
8. never pretend a generic or inherited subagent matched the requested profile

If the runtime cannot let subagents communicate directly but can invoke them separately, the main thread may relay the Mechanic report and Inspector verdict without independently reviewing the implementation state. Relay on actual new state rather than repeatedly polling for it when possible.

If subagents are unavailable entirely, do not simulate multiple agents in prose. Continue as a normal single-agent implementation and say Pit Crew delegation was unavailable in that runtime.
