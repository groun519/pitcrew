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
  <img src="https://img.shields.io/badge/version-0.1.3-111111?style=flat-square" alt="Version 0.1.3">
  <img src="https://img.shields.io/badge/status-v1%20experimental-111111?style=flat-square" alt="V1 experimental">
  <img src="https://img.shields.io/badge/license-MIT-111111?style=flat-square" alt="MIT license">
</p>

<p align="center">
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

The main thread keeps the work that deserves high-level judgment. Luna implements a bounded packet. Sol supervises Luna's implementation and local correction loop. Only a real design decision returns upstairs before the work is clean; otherwise Astra sees the result again after Sol says `PASS`.

<p align="center">
  <img src="assets/workflow.svg" width="1000" alt="Pit Crew workflow">
</p>

## The rule

> **Do not spend Crew Chief judgment on Mechanic work. Do not wake the Crew Chief just to ask whether work is still running. Do not let the Mechanic or Inspector redesign the car.**

Pit Crew v1 is deliberately small. It does not claim to minimize every token, and it does not add speculative routing machinery just because it might help later.

The split is simple:

| Role | Preferred model | Owns | Must not own |
|---|---|---|---|
| **Crew Chief** | Main thread, designed for Astra-class reasoning | User intent, architecture, scope, implementation contract, escalation decisions, final approval | Routine intermediate implementation review or liveness polling |
| **Mechanic** | Luna when explicit routing is available | Editing, implementation, build/test, bounded fixes | Architecture redesign, scope expansion |
| **Inspector** | Sol when explicit routing is available | Execution supervision, contract compliance, local correction loop, final execution gate | Editing, new architecture, ambiguous design choices, pointless liveness polling |

The Crew Chief can always reject an Inspector `PASS`.

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

Pit Crew v0.1.3 also makes the wait concrete. When Codex exposes a `wait_agent` timeout parameter, a healthy worker wait should explicitly request:

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

The 20-minute value is an **experimental Pit Crew tuning value, not a universal law**. It is deliberately long enough to cover ordinary implementation runs without repeatedly waking a large parent context, but it should be changed if measurements or Codex wait semantics justify a better value.

This rule also applies inside the execution cell. Sol should supervise meaningful Luna states, not burn turns merely asking whether Luna is still running. If Sol itself must wait through a supported `wait_agent` surface, it uses the same explicit long wait.

V0.1.3 expresses this through workflow instructions. It does not modify Codex's internal scheduler, guarantee that every client exposes the same wait surface, or turn `wait_agent` into a different runtime primitive.

## How it works

### 1. Crew Chief sets the contract

The main thread reads the request or mockup, inspects enough existing code to make the architecture decision, and produces a bounded implementation packet.

```text
IMPLEMENTATION PACKET
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

Ask Codex to use Pit Crew for an implementation or refactor task.

```text
Use Pit Crew for this task.
```

Or make the intended route explicit:

```text
Use Pit Crew to implement this mockup.
Keep architecture and final approval in the main thread.
Have Sol supervise Luna's implementation and local correction loop.
Use the Pit Crew long wait when wait_agent supports timeout_ms.
Only return to the main thread for a real escalation or after Sol PASS.
```

## V1 is intentionally missing things

Pit Crew v1 does **not** include:

- mandatory Sol pre-contract repository search
- context-pack or read-compression systems
- MCP servers
- lifecycle hooks
- a custom event-driven scheduler
- automatic reasoning-effort routing
- a general-purpose dynamic multi-agent topology planner
- benchmark claims that have not been measured

These are candidates, not promises. They should be added only when real usage demonstrates a clear benefit.

## What we are measuring next

Before putting numbers in the hero section, Pit Crew should earn them.

The first real-world tests should track:

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

The immediate v0.1.3 experiment is simple: a normal task that finishes inside the long wait should produce completion activity before the timeout instead of a chain of short no-state Crew Chief wake-ups.

The broader desired execution property remains the same: routine Luna failures should increase the Luna/Sol loop count, not the number of Astra implementation reviews, and healthy worker silence should not create repeated reasoning turns.

If the data later shows a reliable cost, token, or latency improvement, the README can say exactly how much. Until then, it will not pretend.

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
