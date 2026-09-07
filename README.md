<h1 align="center">Pit Crew</h1>

<p align="center">
  <em>One decides. One builds. One inspects.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Codex-plugin-111111?style=flat-square" alt="Codex plugin">
  <img src="https://img.shields.io/badge/version-0.1.0-111111?style=flat-square" alt="Version 0.1.0">
  <img src="https://img.shields.io/badge/status-v1%20experimental-111111?style=flat-square" alt="V1 experimental">
  <img src="https://img.shields.io/badge/license-MIT-111111?style=flat-square" alt="MIT license">
</p>

<p align="center">
  <strong>Astra calls the strategy. Luna turns the wrench. Sol checks the work.</strong>
</p>

Pit Crew is a bounded multi-model development workflow for Codex.

The main model keeps the work that actually needs high-level judgment: understanding the request, choosing the architecture, setting the implementation contract, and giving final approval. Luna handles implementation. Sol inspects Luna's diff and sends back only local, unambiguous corrections. Anything that changes the design goes straight back to the Crew Chief.

No speculative routing system. No mandatory pre-search pass. No fake token-savings claim. V1 only keeps the split that has a clear job.

---

## Why Pit Crew?

A strong main model can design, implement, review, patch, review again, and eventually finish the task by itself.

It can also spend expensive turns doing work that does not need to be expensive.

### Without Pit Crew

```text
Main model
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
    |
    v
Patch again
```

### With Pit Crew

```text
                  Crew Chief
             architecture / scope
                       |
                       v
                Mechanic (Luna)
                  implementation
                       |
                       v
                Inspector (Sol)
                 /           \
          LOCAL_FIX         ESCALATE
             |                  |
             v                  v
       Mechanic (Luna)      Crew Chief
             |                  |
             +------->----------+
                       |
                     PASS
                       |
                       v
                  Crew Chief
                   final review
```

The point is not to minimize every token. The point is to keep high-level judgment out of low-level correction loops whenever the correction has one obvious answer.

---

## The crew

| Role | Model | Owns | Must not own |
|---|---|---|---|
| **Crew Chief** | Main thread, designed for Astra-class reasoning | User intent, architecture, scope, implementation contract, escalation decisions, final approval | Repetitive low-level correction loops |
| **Mechanic** | Luna when explicit routing is available | Editing, implementation, build/test, bounded fixes | Architecture redesign, scope expansion |
| **Inspector** | Sol when explicit routing is available | Diff review, contract compliance, clear local corrections | New architecture, new abstractions, ambiguous design choices |

The Crew Chief can always reject an Inspector `PASS`.

---

## How it works

### 1. Crew Chief sets the contract

The main thread reads the request or mockup, understands enough of the codebase to make the design decision, and produces a bounded implementation packet.

```text
IMPLEMENTATION PACKET
objective: <one clear objective>
scope:
- <files/systems allowed to change>
contract:
- <required behavior>
- <behavior that must be preserved>
constraints:
- <forbidden redesigns or out-of-scope work>
validation:
- <build/tests/manual checks expected>
```

### 2. Luna builds

The Mechanic implements only that packet, validates what it can, and reports what changed.

```text
MECHANIC REPORT
changed:
- <files or symbols>
validated:
- <build/tests/checks>
uncertainty:
- <none or concrete unresolved point>
```

### 3. Sol inspects

The Inspector reviews the working-tree diff and only the surrounding code necessary to verify it.

It returns one verdict class:

```text
PASS
```

```text
LOCAL_FIX
- location: <file/symbol>
  problem: <concrete problem>
  correction: <bounded required correction>
```

```text
ESCALATE
- location: <file/symbol or contract section>
  reason: <why a design/scope decision is required>
  decision_needed: <smallest question the Crew Chief must answer>
```

### 4. Local fixes stay local

`LOCAL_FIX` goes back to Luna, then Sol checks the resulting diff again.

If fixing the problem would change responsibility, public API, scope, requirements, or architecture, the loop stops and returns to the Crew Chief.

### 5. Crew Chief signs off

Only after Sol returns `PASS`, the main thread performs the final architecture and user-intent review.

---

## The guardrail

Sol is allowed to say:

> The implementation does not match the agreed design. Fix this exact part.

Sol is not allowed to say:

> I prefer a different design. Add another layer and move this responsibility.

### `LOCAL_FIX`

Use it when there is one clear answer inside the existing contract:

- required behavior was plainly omitted
- compile or syntax failure
- clear local logic error
- unnecessary out-of-scope edit that can simply be reverted
- dead code introduced by the patch
- duplicate implementation where an approved existing API already does the job
- local naming or consistency mistake with one established convention

### `ESCALATE`

Return to the Crew Chief when resolution requires:

- moving responsibility between classes or systems
- adding, removing, or materially changing a public API
- introducing a new object, abstraction, layer, or dependency
- expanding implementation scope
- changing the mockup, requirement, or acceptance criteria
- choosing between multiple reasonable designs
- contradicting an earlier Crew Chief decision

---

## Install

### Codex app

Open the plugin marketplace screen and add this repository as a marketplace.

```text
Source: groun519/pitcrew
Git ref: main
Sparse path: <leave empty>
```

Then install **Pit Crew** from the newly added marketplace.

### Codex CLI

```bash
codex plugin marketplace add groun519/pitcrew --ref main
codex plugin add pitcrew@pitcrew
codex plugin list
```

---

## Use

For a normal implementation task:

```text
Use Pit Crew for this task.
```

Or make the workflow explicit:

```text
Use Pit Crew to implement this mockup.
Keep architecture and final approval in the main thread.
Have Luna implement, Sol inspect and request only local corrections,
then do the final review in the main thread.
```

Pit Crew is intentionally aimed at implementation/refactoring work. It should not activate for simple questions, prose-only work, or tasks where you explicitly want a single-agent workflow.

---

## V1 scope

Pit Crew v1 deliberately does **not** include:

- mandatory Sol repository search before implementation
- context-pack or read-range optimization
- MCP servers
- lifecycle hooks
- automatic reasoning-effort routing
- automatic cost-based model routing
- a general-purpose multi-agent framework

Those are candidates only if real usage shows a clear benefit.

### Runtime routing

Pit Crew requests Luna for Mechanic work and Sol for Inspector work when the Codex runtime exposes explicit model routing.

If a client cannot select those models, Pit Crew preserves the role contracts but must not pretend a specific model was used when it was not.

---

## Roadmap

- [x] Codex plugin marketplace packaging
- [x] Crew Chief implementation contract
- [x] Luna Mechanic role
- [x] Sol Inspector role
- [x] `PASS / LOCAL_FIX / ESCALATE` review contract
- [x] Luna <-> Sol local correction loop
- [x] Crew Chief final approval
- [ ] Validate explicit Luna/Sol routing in real Codex work
- [ ] Measure real usage, correction loops, and Astra final-review misses
- [ ] Add optimizations only where measurements justify them

---

## Repository layout

```text
pitcrew/
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── plugins/
│   └── pitcrew/
│       ├── .codex-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── pitcrew/
│               └── SKILL.md
├── LICENSE
└── README.md
```

The repository is both the GitHub marketplace and the source of the Pit Crew plugin.

---

## Why the name?

A racing pit crew works because every specialist has a narrow responsibility and the car does not wait for one person to do everything.

Pit Crew applies the same idea to coding agents:

**decide well, build fast, inspect before release.**

## License

MIT
