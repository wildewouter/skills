---
name: control
description: Interface-first, human-gated development loop. Use when user invokes the /control command, or says things like "let's build this with control", "interface first then review". Structures feature work as Interface -> review gate -> Minimal Stub -> review gate -> Deep Impl (per method) -> review gate (repeat per method) -> Test Path Selection -> Tests. Persists progress in CONTROL.md at repo root so a fresh session can resume.
---

# Control

Workflow skill. Purpose: stop the agent from guessing wrong interfaces and
running away with implementation before a human has signed off on the shape
of the code.

## Persistence

ACTIVE for the whole feature once invoked via `/control`, across as many
messages/sessions as it takes. Governed entirely by `CONTROL.md` at repo
root — state lives in the file, not in chat memory, so a brand new session
can pick up exactly where the last one left off. Never delete `CONTROL.md`
when a plan finishes; it's the record of what was decided at each gate.

## CONTROL.md format

Create at repo root if missing when `/control` starts. One file per active
plan (if a repo has multiple in-flight plans, ask user how to disambiguate,
don't guess).

```markdown
# Control: <feature name>

## Interface
- [ ] drafted
- [ ] reviewed

## Stub
- [ ] compiles / minimal stub in place
- [ ] reviewed

## Deep implementation
- [ ] MethodA
- [ ] MethodB
- [ ] MethodC

## Tests
- [ ] paths enumerated
- [ ] paths selected
- [ ] tests written

## Revisions
<!-- append-only log. new insight sends work back to an earlier phase —
     log it here instead of unchecking prior boxes. -->
- <!-- e.g. "back to Interface: MethodB needs a callback param, discovered while implementing MethodA" -->
```

Check off items and add notes inline as you go. Update `CONTROL.md` before
every stop, not just at phase end — a resumed session must see the true
state without asking chat history. On resume: read this file first, it
tells you exactly which phase and which method you're on.

## Loop

### 1. Interface draft

From the plan, write only types/signatures/data models — no bodies. Check
off "drafted" in `CONTROL.md`. Then **stop**. Say plainly: "Interfaces
drafted, need your review before I go further. Reply when ready to
continue." Do not touch implementation past this point until told to
proceed. Human may edit the interface directly — if so, re-read their
version before resuming, don't fight it. On approval, check off "reviewed"
before moving to phase 2.

### 2. Initial stub
Fill bodies with the smallest thing that satisfies the compiler/type
checker: `throw new Error("not implemented")` (or language equivalent) —
never silent defaults, a stub must fail loudly if actually called. One pass
across the whole interface. Check off "compiles / minimal stub in place" in
`CONTROL.md`. Then **stop**, same review-gate wording as above. On
approval, check off "reviewed" before moving to phase 3.

If the interface involves a real architectural decision (how a service is
structured, which pattern/abstraction/dependency to use, data flow shape,
etc.), do not silently pick one. Present the viable options (with brief
tradeoffs) via `ask_user` — or propose your preferred option and an
alternative — and let the human decide before drafting. Only skip this when
there is truly one obvious way to do it.

### 3. Deep implementation, per method
Present user with `ask_user` tool to pick one method/function from the 
"Deep implementation" checklist. Implement it fully. Update `CONTROL.md` 
(check it off) immediately, before stopping — never batch the checkbox
update for later. Then **stop**, same gate. Repeat for the next unchecked
method only after explicit go-ahead. Never batch multiple methods in one
pass — that's the whole point of this skill.

### 4. Tests
Once all methods are checked off: enumerate the distinct code paths/branches
in the implementation. Present them with the `ask_user` tool as a
multi-select list — let the human pick which paths are worth testing, don't
assume "all of them". Check off "paths enumerated" then "paths selected" in
`CONTROL.md` as each happens. Write tests only for selected paths. Keep
tests small (one behavior each) and comprehensive within that scope (edge
cases for the paths chosen) — this skill is standalone for testing style, it
does not defer to other testing skills even if loaded alongside. Check off
"tests written" once done.

## Rules

- No hardcoded build tool/language — generic across repos. Detect from repo
  (package.json, go.mod, Cargo.toml, etc.) or ask if ambiguous.
- Never skip a gate "to save time" — the whole value of this skill is that
  a human looks at every interface and every method before the next step.
- If the human requests changes at a gate, make them, then re-stop at the
  same gate — don't auto-advance to the next phase.
- If new insight (yours or the human's) sends work back to an earlier phase
  (e.g. deep-implementing MethodA reveals the Interface needs a new param),
  do not uncheck the earlier boxes — checked means "was reviewed at time X",
  it's history, not current-truth. When the insight is yours, never
  implement it straight away: stop and present it via `ask_user` (what
  changed, why, which earlier phase it affects) and get a go-ahead first.
  Once agreed, append a line to the `Revisions` log naming the phase
  revisited and the insight, make the change, then re-run that phase's own
  gate (interface change → re-review at gate 1, etc.) before returning to
  where you left off.
- Avoid leaky abstractions at all times: interfaces and stubs must not expose
  implementation details (internal types, storage/framework specifics, error
  types from a dependency, etc.) through their public shape. Flag it and
  propose a fix rather than let a leak through a review gate unmentioned.
- Any architectural decision (service structure, choice of pattern/library/
  abstraction, data flow) must be surfaced to the human before committing to
  it — present options or a recommendation plus an alternative, never decide
  silently.
