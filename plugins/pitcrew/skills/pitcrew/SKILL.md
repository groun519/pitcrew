---
name: pitcrew
description: Run the Pit Crew coding workflow. Use for implementation or refactoring tasks where the user wants a high-quality main-thread decision, bounded Luna implementation, Sol inspection and local correction, then final main-thread approval. Trigger when the user says Pit Crew, crew chief, Luna implement then Sol review, multi-model coding workflow, or asks to reduce expensive main-model implementation/review loops. Do not use for simple questions, prose-only tasks, or when the user explicitly asks for a single-agent workflow.
license: MIT
---

# Pit Crew

Run one bounded implementation pipeline:

```text
Crew Chief -> Mechanic -> Inspector <-> Mechanic -> Crew Chief
```

The parent/main thread is the Crew Chief. Pit Crew v1 is designed for an Astra-class main thread, but never claim the active model is Astra unless the runtime actually identifies it that way.

## V1 purpose

Keep architecture and final judgment in the main thread while moving bulk implementation and low-level correction loops to subagents.

Do not add speculative cost optimizations. In v1:

- do not require Sol before implementation
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
- the implementation packet sent to the Mechanic
- escalation decisions
- final diff review and approval

The Crew Chief must not outsource a design decision merely to save tokens.

### Mechanic

Prefer `gpt-5.6-luna` when the runtime supports explicit subagent model selection.

The Mechanic:

- edits code within the packet
- builds or runs targeted tests when practical
- follows existing project conventions
- does not redesign the architecture
- does not expand scope without escalation
- does not commit or push unless the user explicitly asked for it

The Mechanic returns a concise report containing:

```text
MECHANIC REPORT
changed:
- <files or symbols>
validated:
- <build/tests/checks>
uncertainty:
- <none or concrete unresolved point>
```

### Inspector

Prefer `gpt-5.6-sol` when the runtime supports explicit subagent model selection.

The Inspector is review-only. It may inspect the current working tree, diff, and narrowly necessary surrounding code. It must not edit files.

Review in this order:

1. implementation contract compliance
2. correctness and regressions
3. unrequested or out-of-scope changes
4. use of existing APIs versus needless duplicate logic
5. unnecessary abstractions or dead code introduced by the patch
6. naming and local consistency where there is one obvious answer
7. build/test evidence and obvious validation gaps

The Inspector must return exactly one verdict class:

```text
PASS
```

or

```text
LOCAL_FIX
- location: <file/symbol>
  problem: <concrete problem>
  correction: <bounded required correction>
```

or

```text
ESCALATE
- location: <file/symbol or contract section>
  reason: <why a design/scope decision is required>
  decision_needed: <the smallest question the Crew Chief must answer>
```

Do not mix `LOCAL_FIX` and `ESCALATE` in one verdict. If any item requires escalation, return `ESCALATE` and include the other concrete findings as supporting notes only when they matter to that decision.

## What counts as LOCAL_FIX

The Inspector may request a correction without a new Crew Chief decision only when there is one clear implementation answer inside the existing contract, for example:

- a required behavior was plainly omitted
- compile or syntax failure
- clear local logic error
- unnecessary out-of-scope edit that can simply be reverted
- dead code introduced by the patch
- duplicate implementation where the already-approved existing API covers the same job
- a local naming or consistency mistake with one established project convention

The Inspector must make the correction request as small as possible.

## What must ESCALATE

Escalate instead of locally fixing when resolution would require any of the following:

- moving responsibility between classes or systems
- adding, removing, or materially changing a public API
- introducing a new object, abstraction, layer, or dependency
- expanding the implementation scope
- changing the mockup, requirement, or acceptance criteria
- choosing between multiple reasonable designs
- contradicting a Crew Chief decision
- resolving a mismatch between the contract and the real codebase where intent is unclear

## Workflow

### 1. Crew Chief: establish the contract

Read the user request, supplied mockup, and enough existing code to make the architecture decision.

Produce an internal bounded implementation packet. It should contain only what the Mechanic needs:

```text
IMPLEMENTATION PACKET
objective: <one clear objective>
scope:
- <files/systems allowed to change>
contract:
- <required behavior>
- <required preserved behavior>
constraints:
- <forbidden redesigns or out-of-scope work>
validation:
- <build/tests/manual checks expected>
```

Do not send the full parent conversation to the Mechanic unless the runtime makes that unavoidable. Prefer a fresh or minimally forked subagent context plus the packet.

### 2. Mechanic: implement

Delegate the implementation packet to one Mechanic subagent.

If explicit model routing is available, request `gpt-5.6-luna`. Do not set a reasoning effort in v1 unless the user explicitly requested one; effort tuning is intentionally deferred until usage data exists.

Wait for the Mechanic to finish. Keep its code changes in the working tree.

### 3. Inspector: review the resulting change

Delegate review to one Inspector subagent.

Give it:

- the implementation contract
- the Mechanic report
- access to the current diff/working tree

Do not ask Sol to re-design the implementation. If explicit model routing is available, request `gpt-5.6-sol`. Do not set a reasoning effort in v1 unless the user explicitly requested one.

Prefer a fresh or minimally forked Inspector context. The Inspector should inspect `git status`, the final diff, and only the surrounding code needed to validate findings.

### 4. Local correction loop

If the Inspector returns `LOCAL_FIX`, route only that fix packet back to the Mechanic.

The Mechanic applies only those corrections and returns an updated report. Then ask the Inspector to review the resulting current diff again.

Do not loop mechanically forever. If the same issue recurs, the correction stops being local, or the Mechanic/Inspector disagree about architecture, escalate to the Crew Chief.

### 5. Escalation

If the Inspector returns `ESCALATE`, stop the local loop.

The Crew Chief reads the minimum original code needed, makes the missing design decision, updates the implementation contract, and sends a new bounded packet to the Mechanic. The Inspector then reviews the new result.

### 6. Crew Chief final review

Only after the Inspector returns `PASS`, the Crew Chief performs the final review.

The final review is not a duplicate style pass. Check the things only the Crew Chief owns:

- does the final code actually satisfy the user's intent?
- does responsibility live in the right place?
- did the implementation preserve the intended architecture?
- did the local fix loop accidentally narrow or distort the original requirement?
- is any change unnecessary at the architecture level?

The Crew Chief may still reject a Sol `PASS`.

### 7. Report to the user

Report:

- what was implemented
- what was validated
- any unresolved limitation or runtime routing limitation

Do not expose internal chain-of-thought. Do not claim a specific subagent model was used unless the runtime actually routed that model.

## Runtime fallback

Codex subagent and model-selection surfaces can differ by client/runtime.

If explicit Luna/Sol selection is unavailable:

1. preserve the Mechanic and Inspector role contracts
2. use available subagents only if delegation itself is supported
3. state the routing limitation in the final report
4. never pretend a generic/inherited subagent was Luna or Sol

If subagents are unavailable entirely, do not simulate multiple agents in prose. Continue as a normal single-agent implementation and say Pit Crew delegation was unavailable in that runtime.
