# Parlance for Godot

A GDScript runtime for the [Parlance](https://github.com/Orbitope/parlance)
narrative format.

Parlance stores a game's story as plain JSON — dialogues, conditions, effects,
skill checks, character dialogue ladders, quests, endings. This addon executes
that JSON in Godot 4, and is verified against the published conformance vectors
rather than against its author's confidence.

```
  PASS  mulberry32                     6 vectors
  PASS  evaluate                      53 vectors
  PASS  applyEffect                   27 vectors
  PASS  resolveCheck                  18 vectors
  PASS  stepDialogue                   8 vectors
  PASS  chooseChoice                   9 vectors
  PASS  advanceNode                    9 vectors
  PASS  resolveCharacterDialogue       6 vectors
  SKIP  resolveQuests                  6 vectors — not ported yet
  SKIP  progression                   13 vectors — not ported yet

136 passed, 0 failed, 19 skipped (not yet ported)
```

## Install

Copy `addons/parlance/` into your project's `addons/` directory. That's it —
the runtime is static classes, so there is no plugin to enable and nothing to
add to your autoloads.

## Use

Load your JSON however you like, then call the runtime. State is immutable:
every entry point returns a new state and never mutates its input.

```gdscript
const State := preload("res://addons/parlance/state.gd")
const Runtime := preload("res://addons/parlance/runtime.gd")
const Rng := preload("res://addons/parlance/rng.gd")

var project := {"dialogues": {...}, "characters": {...}, "skills": {...}}
var state = State.from_dict({})            # or a saved SerializedGameState

# Present a node: filters choices by showIf, interpolates {placeholders}.
var step: Dictionary = Runtime.step_dialogue(dialogue, "node_start", state, project)
print(step["node"]["text"])
for choice in step["visibleChoices"]:
    print(choice["text"])

# onEnter effects are RETURNED, not applied. You decide when they fire —
# on first arrival, not on replay.
state = Runtime.apply_effects(step["onEnterEffects"], state, project)

# Take a choice. Pass a seeded RNG so checks are reproducible.
var outcome: Dictionary = Runtime.choose_choice(
    dialogue, "node_start", "ch_ask", state, project, Rng.for_step(seed, step_index)
)
state = outcome["newState"]
if outcome.has("checkResult"):
    print(outcome["checkResult"])          # {passed, roll, total, skillValue, dice}
var next = outcome["nextNodeId"]           # null on a terminal choice
```

Three things that are easy to get wrong, and are contract rather than style:

- **`onEnter` effects are not applied for you.** `step_dialogue` returns them;
  firing them is the caller's job, on first arrival only. Applying them on every
  render double-counts on a rewind.
- **A node with no choices is not necessarily over.** If it has `next`, call
  `advance_node` — that's a listen-only beat. Treating it as the end silently
  truncates ambient chains.
- **Skills start empty by contract.** `createDefaultState` leaves them for the
  caller to fill from your character's stats or `progression.json`'s
  `startingSkills`. Skip it and every check rolls at zero.

## What's ported

`evaluate`, `applyEffect`, `resolveCheck`, `stepDialogue`, `chooseChoice`,
`advanceNode`, `resolveCharacterDialogue`, and the mulberry32 PRNG — enough to
run dialogue end to end, including gated choices, active skill checks, effects,
and character ladders.

**Not ported:** `resolveQuests` and `progression`. They report as SKIP rather
than passing by omission. Note that quest *conditions* still work: `questOutcome`
re-evaluates an outcome's own `reachedWhen` against current state rather than
reading a fired-record, so endings gated on outcomes resolve without them.

**Implemented but not vector-covered:** `resolveSpeaker`, `effectiveSpeakerId`,
and `resolvePortrait`. The suite has no family for them, so they come from the
written contract alone with nothing mechanically checking them. They are marked
in-file as the least-trusted code here. If you rely on them, test them.

## Verify it yourself

```bash
godot --headless --script tests/conformance_runner.gd
```

Exit code is 0 only if every implemented family passes. Skipped families are
reported and never mask a failure — a suite that goes green by not running is
the failure mode this runner exists to avoid.

To confirm the suite can actually fail, break something on purpose: change a
`>=` to `>` in `_compare` in `addons/parlance/runtime.gd` and re-run. You should
get 5 failures across `evaluate` and `stepDialogue`. If it stays green, the
suite is lying and that is worth an issue.

CI runs the same command on every push.

## The vectors

`conformance/` and `schema/` are a pinned copy of the Parlance spec — see
[`conformance/PIN`](conformance/PIN) for the exact ref. **Never hand-edit them.**
They are generated from and asserted against the reference implementation, so an
edit makes this port chase a ghost. Re-copy from a newer ref instead.

They are also MIT and meant to be copied. If you are writing a port for another
engine, take them; you never need the reference implementation, only the
vectors.

## A worked example

[Mistfall Inn](https://github.com/Orbitope/mistfall-inn) is a small, complete
murder mystery built on this addon — three rooms, thirteen dialogues, three
endings. Useful if you want to see the runtime driving real authored content
rather than test fixtures.

## Licence

MIT, including the vendored spec files. See [LICENSE](LICENSE).
