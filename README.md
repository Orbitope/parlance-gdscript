# Parlance for Godot

A GDScript runtime for [Parlance](https://github.com/Orbitope/parlance), a
git-native narrative design tool for story-driven games.

**Parlance is the authoring tool; this is one engine's runtime for what it
produces.** You write your story in Parlance's visual editor — dialogue and
quest canvases, a searchable reference index, live playtest — and it saves as
human-readable JSON directly in your repo. No database, no import/export step,
git as the single source of truth.

This addon reads that JSON and runs it in Godot 4: dialogues, conditions,
effects, skill checks (with conditional modifiers), character dialogue offers,
quests, endings. It is verified against Parlance's published conformance
vectors rather than against its author's confidence.

```
  PASS  mulberry32                     6 vectors
  PASS  evaluate                      53 vectors
  PASS  applyEffect                   27 vectors
  PASS  resolveCheck                  24 vectors
  PASS  stepDialogue                  11 vectors
  PASS  chooseChoice                  10 vectors
  PASS  advanceNode                   12 vectors
  PASS  resolveCharacterDialogue      16 vectors
  PASS  nextContinuations              4 vectors
  PASS  resolveQuests                  6 vectors
  PASS  progression                   13 vectors

182 passed, 0 failed, 0 skipped (not yet ported)
```

## Compatibility

| parlance-gdscript | Parlance spec | Families |
|---|---|---|
| `main` (unreleased) | v0.14.0 — pinned to [`f4a25b0`](conformance/PIN) | 11 of 11 |

**Versions here are independent of Parlance's**, deliberately, and this table is
how the two are tied together. Parlance uses the patch slot itself (`v0.4.3`
exists), so a mirrored version would leave this port no room to release its own
fixes without colliding with a spec release.

`v1.0.0` means something checkable: complete against the spec, not merely
current with it. As of the v0.14.0 pin every family passes, so `main` meets
that bar; it is not tagged yet.

[`conformance/PIN`](conformance/PIN) is the authoritative record of which
upstream ref the vectors came from — this table is the human-readable summary of
it, and if they ever disagree, PIN is right.

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

Every conformance family: `evaluate`, `applyEffect`, `resolveCheck`,
`stepDialogue`, `chooseChoice`, `advanceNode`, `resolveCharacterDialogue`,
`nextContinuations`, `resolveQuests`, the progression functions, and the
mulberry32 PRNG. That covers dialogue end to end (gated choices, active skill
checks and their conditional modifiers, effects, character dialogue offers),
what to offer when a scene ends, quest effects, and levelling. `check_bonus`,
`passive_check_passes` and `condition_specificity` are exported too:
`passive_check_passes` is the passive-check reveal threshold
(`skill + Σbonus >= difficulty`) an engine should use to show or hide a passive
choice, so a modifier means the same thing in both modes.

The rest of the public surface, in the same file:

- **`next_continuations(state, project, visited, current_dialogue_id)`** — what
  to offer when a scene ends. A pending cutscene first; then any character
  routed with `set_active_dialogue` whose winning offer reads its flag (queued,
  visited set ignored); otherwise each character's best eligible offer.
  `clear_active_dialogue` and `clear_pending_cutscene` consume them.
- **`resolve_quests(state, project)`** — fires quest stage `onComplete` and
  outcome `effects` whose condition holds, once each, to a fixpoint. Run it
  after every state change. It never advances a stage for you.
- **Progression** — `level_for_xp`, `points_earned`, `available_points`,
  `recompute_skills`, `invest_skill_point`, `effective_skill`, `skill_cap`.
  `xp` is total-earned; levels and points are derived. Investing is a guarded
  player action, not an effect. Pass the skills registry to honour per-skill
  `max`.

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
get 6 failures across `evaluate`, `resolveCheck` and `stepDialogue`. If it stays
green, the suite is lying and that is worth an issue.

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
