# Pit Crew

A multi-model development crew for Codex. One inspects, one decides, one builds.

Pit Crew keeps the expensive judgment work in the main thread and pushes implementation and low-level review into bounded subagent loops.

## Crew

- Crew Chief: the main thread. Designed for Astra-class reasoning. Owns requirements, architecture, implementation scope, and final approval.
- Mechanic: Luna. Implements the bounded work packet, builds/tests, and reports what changed.
- Inspector: Sol. Reviews the Mechanic's changes, requests only local corrections, and escalates design questions back to the Crew Chief.

## V1 workflow

```text
User / Mockup
      |
      v
Crew Chief
- understand requirements
- decide architecture
- define bounded implementation packet
      |
      v
Mechanic (Luna)
- implement
- build/test
      |
      v
Inspector (Sol)
- review current changes
- PASS / LOCAL_FIX / ESCALATE
      |
      +--> LOCAL_FIX --> Mechanic --> Inspector
      |
      +--> ESCALATE --> Crew Chief
      |
      v
Crew Chief
- final architecture and quality review
- approve or revise
```

Pit Crew v1 intentionally does not use Sol as a mandatory pre-implementation codebase search layer. It also does not include context-pack optimization, MCP, lifecycle hooks, or automatic reasoning-effort tuning. Those should be added only after real usage shows a clear benefit.

## Install

Register this repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add groun519/pitcrew --ref main
```

Then install Pit Crew:

```bash
codex plugin add pitcrew@pitcrew
```

Verify the installation:

```bash
codex plugin list
```

You can also open the Codex plugin browser and install `pitcrew` from the `Pit Crew` marketplace after registering it.

## Use

Ask Codex to use Pit Crew for an implementation task, or invoke the `pitcrew` skill when it is available in your client.

Example:

```text
Use Pit Crew to implement this mockup. Keep the architecture decision in the main thread, have Luna implement it, have Sol clean up the implementation, then do the final review in the main thread.
```

## Runtime note

Current Codex releases support subagents and model-specific subagent configuration, but the exact model-selection surface can vary by client/runtime. Pit Crew requests Luna for Mechanic work and Sol for Inspector work when explicit model routing is available. If the runtime cannot select those models, the workflow must preserve the role boundaries and must not pretend a model was used when it was not.

## V1 boundaries

Inspector may directly request a local correction only when the fix does not require a new design decision. Examples include:

- missed requirement with an obvious correction
- compile or syntax issue
- clear logic error
- unnecessary or out-of-scope change
- dead code introduced by the patch
- obvious duplication when an existing API already covers the same behavior
- local naming or consistency issue that has one clear answer

Inspector must escalate to the Crew Chief when the fix would require:

- changing class responsibility
- adding or removing a public API
- introducing a new object or abstraction
- expanding the agreed scope
- changing the mockup or acceptance criteria
- choosing between multiple valid designs

## License

MIT
