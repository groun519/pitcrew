<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.svg">
    <img src="assets/logo.svg" width="220" alt="Pit Crew logo">
  </picture>
</p>

<h1 align="center">Pit Crew</h1>

<p align="center">
  <em>One decides. One builds. One inspects.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Codex-plugin-111111?style=flat-square" alt="Codex plugin">
  <img src="https://img.shields.io/badge/version-0.1.4-111111?style=flat-square" alt="Version 0.1.4">
  <img src="https://img.shields.io/badge/status-v1%20experimental-111111?style=flat-square" alt="V1 experimental">
  <img src="https://img.shields.io/badge/license-MIT-111111?style=flat-square" alt="MIT license">
</p>

<p align="center">
  <a href="#profiles">Profiles</a> &middot;
  <a href="#how-it-works">How it works</a> &middot;
  <a href="#install">Install</a> &middot;
  <a href="#use">Use</a> &middot;
  <a href="#v1-is-intentionally-missing-things">V1 boundaries</a>
</p>

<p align="center">
  <img src="assets/crew-banner.svg" width="1100" alt="Pit Crew roles in a racing garage">
</p>

<p align="center">
  <strong>The Crew Chief makes the call. The Mechanic gets under the hood. The Inspector keeps the work in the bay until it is ready to go upstairs.</strong>
</p>

---

## Why Pit Crew?

Give one strong model a coding task and it can do everything: understand the request, choose the architecture, write the code, inspect its own diff, patch small mistakes, inspect again, and keep going until the job is clean.

**Pit Crew gives that model a crew instead.**

The main thread keeps the work that deserves high-level judgment. Luna implements a bounded packet. Sol supervises Luna's implementation and local correction loop. Only a real design decision returns upstairs before the work is clean; otherwise the Crew Chief sees the result again after the Inspector says `PASS`.

<p align="center">
  <img src="assets/workflow.svg" width="1000" alt="Pit Crew workflow">
</p>

## The rule

> **Do not spend Crew Chief judgment on Mechanic work. Do not wake the Crew Chief just to ask whether work is still running. Do not let the Mechanic or Inspector redesign the car.**

Pit Crew v1 is deliberately small. It does not claim to minimize every token, and it does not add speculative routing machinery just because it might help later.

The role split stays the same across profiles:

| Role | Owns | Must not own |
|---|---|---|
| **Crew Chief** | User intent, architecture, scope, implementation contract, escalation decisions, final approval | Routine intermediate implementation review or liveness polling |
| **Mechanic** | Editing, implementation, build/test, bounded fixes | Architecture redesign, scope expansion |
| **Inspector** | Execution supervision, contract compliance, local correction loop, final execution gate | Editing, new architecture, ambiguous design choices, pointless liveness polling |

The Crew Chief can always reject an Inspector `PASS`.

## Profiles

Pit Crew v0.1.4 has two fixed model profiles.

| Profile | Crew Chief | Inspector | Mechanic |
|---|---|---|---|
| **Quality** | **Astra / high** | **Sol / xhigh** | **Luna / xhigh** |
| **Balanced** | **Sol / xhigh** | **Sol / high** | **Luna / high** |

### Quality

Quality is the default Pit Crew route and prioritizes reducing avoidable quality loss over minimizing model effort.

```text
Crew Chief : Astra high
Inspector  : Sol xhigh
Mechanic   : Luna xhigh
```

Use it for architecture-heavy work, important refactors, new systems, responsibility changes, or any task where the strongest main-thread judgment is worth the extra usage.

### Balanced

Balanced keeps the same harness and the same review boundaries, but removes Astra from ordinary implementation work.

```text
Crew Chief : Sol xhigh
Inspector  : Sol high
Mechanic   : Luna high
```

It is intended for work where the architecture is already reasonably understood: bounded feature implementation, normal refactoring, bug fixing, extending an established system, or implementing an already-approved mockup.

Balanced is not an Economy mode. Sol still owns every judgment/review role, and Luna still gets a high reasoning budget for implementation.

### Why Luna stays strong

Pit Crew does not lower Luna to medium or low in either profile. Luna is the cheapest execution role in the crew, and a stronger first implementation may avoid a more expensive cycle of:

```text
Luna implementation
      -> Sol review
LOCAL_FIX
      -> Luna correction
      -> Sol re-review
```

The profile is therefore designed to spend reasoning earlier in the bounded Mechanic step when that can reduce later Inspector work.

### The main thread matters

The Crew Chief is the main Codex thread that is already open. Pit Crew does not silently replace that parent model or its reasoning effort.

Before using a profile, select the matching main-thread setting:

```text
Quality  -> start Codex with Astra high
Balanced -> start Codex with Sol xhigh
```

If the runtime can verify the current main model/effort and it does not match the requested profile, Pit Crew should report the mismatch instead of pretending the profile is active. If the runtime cannot expose that information, the Crew Chief route remains unverified and should be reported as such.

A plain `Use Pit Crew ...` means **Quality**. Use `Pit Crew Balanced` explicitly when you want the lower-cost profile. Pit Crew does not automatically choose between Quality and Balanced from its own estimate of task difficulty.

## Before / after

### One model doing the whole stop

```text
Main model
    |
    v
Design
    |
    v
Implement
    |
    v
Self-review
    |
    v
Patch
    |
    v
Self-review again
```

### Pit Crew

```text
Crew Chief
    |
    | contract
    v
+------------------------+
|     Execution Cell     |
|                        |
|  Mechanic <-> Inspector|
|   Luna          Sol    |
|     fixes <-> review   |
|                        |
+-----------+------------+
            |
       PASS | ESCALATE
            v
       Crew Chief
      final review
```

The intended saving is not "make every model read less." V1 targets a clearer problem: keep the expensive main thread out of repetitive, low-level implementation supervision and correction loops.

The normal Crew Chief touch points are intentionally narrow:

```text
1. Set contract
2. Sleep while execution is healthy
3. Receive Inspector PASS or real ESCALATE
4. Final architecture / intent review
```

A Mechanic mistake, failed patch, build error, local implementation defect, wait timeout, or quiet worker is not by itself a reason for the Crew Chief to inspect the diff.

## Stay asleep while the work is healthy

Pit Crew treats idle orchestration as part of the harness, not as useful reasoning work.

A worker that has not produced a new state is not automatically stuck. A wait timeout is not automatically a failure. If the runtime can wake on completion or meaningful state change, Pit Crew prefers that behavior.

When Codex exposes a `wait_agent` timeout parameter, a healthy worker wait should explicitly request:

```text
wait_agent timeout_ms = 1200000
```

That is a **20-minute maximum wait**. Current Codex wait implementations can return earlier when completion or relevant activity arrives, so a worker that finishes after three minutes does not need to sit idle for twenty.

```text
BAD
Crew Chief -> short wait -> no change
Crew Chief -> short wait -> no change
Crew Chief -> short wait -> no change

BETTER
Crew Chief -> contract
            [execution continues]
            [one long wait]
            PASS / ESCALATE
Crew Chief -> decision
```

The 20-minute value is an **experimental Pit Crew tuning value, not a universal law**. It should change if measurements or Codex wait semantics justify a better value.

This rule also applies inside the execution cell. Sol should supervise meaningful Luna states, not burn turns merely asking whether Luna is still running. If Sol itself must wait through a supported `wait_agent` surface, it uses the same explicit long wait.

Pit Crew expresses this through workflow instructions. It does not modify Codex's internal scheduler, guarantee that every client exposes the same wait surface, or turn `wait_agent` into a different runtime primitive.

## How it works

### 1. Choose the profile and set the Crew Chief

Select the main-thread model/effort before starting the task:

```text
Quality  : Astra high
Balanced : Sol xhigh
```

Pit Crew then reads the request or mockup, inspects enough existing code to make the architecture decision, and produces a bounded implementation packet.

```text
IMPLEMENTATION PACKET
profile: <Quality or Balanced>
objective: <one clear objective>
scope:
- <files/systems/state allowed to change>
contract:
- <required behavior>
- <behavior that must be preserved>
constraints:
- <forbidden redesigns or out-of-scope work>
validation:
- <build/tests/manual checks expected>
```

### 2. Start the execution cell

The packet goes to Luna for implementation and Sol becomes the execution supervisor for that same contract.

When Codex supports explicit subagent model and effort routing, Pit Crew requests the profile exactly:

```text
QUALITY
Inspector : Sol xhigh
Mechanic  : Luna xhigh

BALANCED
Inspector : Sol high
Mechanic  : Luna high
```

The exact runtime messaging topology may vary. The invariant is more important: routine implementation success, failure, and correction stay inside the Luna/Sol cell until `PASS` or a real `ESCALATE`.

If the main thread must relay subagent messages because the runtime does not support direct subagent communication, it should relay actual new state without doing its own intermediate implementation review. It should not create extra reasoning turns merely to check liveness.

When the runtime exposes `wait_agent(timeout_ms=...)`, Pit Crew asks for `timeout_ms=1200000` while waiting on a healthy subagent instead of relying on a short default wait.

### 3. Mechanic attempts the work

Luna implements only the packet, validates what it can, and reports the concrete result.

```text
MECHANIC REPORT
changed:
- <files, symbols, repository state, or attempted operation>
validated:
- <build/tests/checks>
uncertainty:
- <none or concrete unresolved point>
```

The report does not need to pretend the first attempt is perfect. Failed validation, a blocked repository operation, or an incomplete local result goes to Sol first, not back upstairs.

### 4. Inspector supervises and gates

Sol reviews the contract and current execution state. It may inspect a finished diff, an incomplete attempt, staged state, build output, patch failure, or only the surrounding code needed to validate a concrete finding.

Sol does not edit. But it may reason independently about the correct bounded result, diagnose why Luna's attempt failed, and tell Luna to use a different concrete method when there is one clear answer inside the approved contract.

It returns exactly one verdict class:

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

### 5. Local problems stay in the bay

`LOCAL_FIX` goes back to Luna. Luna changes only that bounded issue, then Sol checks the current state again.

```text
Luna attempt
    |
    v
   Sol
  /   \
PASS  LOCAL_FIX
        |
        v
      Luna
        |
        +------> Sol
```

A local fix is appropriate when there is one clear answer inside the existing contract, such as:

- an obvious requirement was omitted
- compile or syntax failure
- a patch, staging, line-ending, or repository-state error with one clear correction
- a clear local logic error
- a needless out-of-scope edit that can simply be reverted
- dead code introduced by the patch
- duplicate logic where the approved existing API already covers the job
- a local naming or consistency error with one established project convention
- a clearly bounded deterministic method can replace an unproductive implementation attempt

### 6. Design problems go upstairs

Only a real `ESCALATE` should interrupt the execution cell.

Sol must escalate when resolution would require any of the following:

- moving responsibility between classes or systems
- adding, removing, or materially changing a public API beyond the approved contract
- introducing a new object, abstraction, layer, or dependency
- expanding the implementation scope
- changing the mockup, requirement, or acceptance criteria
- choosing between multiple reasonable designs
- contradicting a Crew Chief decision
- resolving a contract/codebase mismatch where intent is unclear

A Mechanic failure, quiet worker, or wait timeout by itself is not an escalation.

### 7. Crew Chief closes the stop

Only after the Inspector returns `PASS`, the main thread performs the final architecture and intent review.

That review asks different questions from Sol:

- does the result actually satisfy the user's intent?
- does responsibility live in the right place?
- did the implementation preserve the intended architecture?
- did the local-fix loop distort or narrow the original requirement?
- is any architecture-level change unnecessary?

If the Crew Chief finds only a local implementation defect, it sends the result back into the Sol-led execution cell instead of correcting Luna directly. If the problem changes architecture or contract, the Crew Chief updates the contract first.

## Install

### Codex app (recommended)

1. Open **Plugins**.
2. Add a GitHub marketplace.
3. Use `groun519/pitcrew` as the source.
4. Use `main` as the Git ref.
5. Leave the sparse path empty.
6. Open the imported **Pit Crew** marketplace and install `pitcrew`.

### Codex CLI

```bash
codex plugin marketplace add groun519/pitcrew --ref main
codex plugin add pitcrew@pitcrew
codex plugin list
```

## Use

### Quality

Select **Astra high** as the main thread, then ask:

```text
Use Pit Crew Quality to implement this mockup.
```

A plain Pit Crew request also defaults to Quality:

```text
Use Pit Crew for this task.
```

### Balanced

Select **Sol xhigh** as the main thread, then ask:

```text
Use Pit Crew Balanced to implement this mockup.
```

Both profiles preserve the same contract, Inspector-led execution cell, `PASS / LOCAL_FIX / ESCALATE` behavior, and long-wait policy.

## V1 is intentionally missing things

Pit Crew v1 does **not** include:

- an Economy profile
- automatic task-complexity based profile selection
- mandatory Sol pre-contract repository search
- context-pack or read-compression systems
- MCP servers
- lifecycle hooks
- a custom event-driven scheduler
- a general-purpose dynamic multi-agent topology planner
- benchmark claims that have not been measured

These are candidates, not promises. They should be added only when real usage demonstrates a clear benefit.

## What we are measuring next

Before putting numbers in the hero section, Pit Crew should earn them.

The first real-world tests should track:

- selected profile
- Crew Chief substantive touch points per task
- unwanted Crew Chief intermediate implementation reviews
- no-state Crew Chief wake-ups or polling turns
- `wait_agent` timeouts during healthy execution
- Sol <-> Luna correction loops
- no-state Inspector wake-ups or polling turns
- issues still found by the Crew Chief after Sol `PASS`
- unnecessary or incorrect Inspector corrections
- per-model usage
- total turns to a clean result
- Quality versus Balanced usage and final-result differences on comparable work

The desired execution property remains the same: routine Luna failures should increase the Luna/Sol loop count, not the number of Crew Chief implementation reviews, and healthy worker silence should not create repeated reasoning turns.

If the data later shows a reliable cost, token, latency, or quality difference between the profiles, the README can say exactly how much. Until then, it will not pretend.

## Repository layout

```text
pitcrew/
|-- .agents/
|   `-- plugins/
|       `-- marketplace.json
|-- assets/
|   |-- logo.svg
|   |-- logo-dark.svg
|   |-- crew-banner.svg
|   |-- workflow.svg
|   `-- social-preview.svg
|-- plugins/
|   `-- pitcrew/
|       |-- .codex-plugin/
|       |   `-- plugin.json
|       `-- skills/
|           `-- pitcrew/
|               `-- SKILL.md
|-- LICENSE
`-- README.md
```

## Why the name?

A pit crew works because specialists do not all grab the same wrench.

One person makes the call. One does the work. One checks whether the car is ready to leave.

That is the whole idea.

## License

MIT
