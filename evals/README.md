# swift-midi skill evals

Run 2026-07-03/04 with Claude Fable 5 (claude-fable-5) via Claude Code.

## With-skill vs without-skill benchmark (`benchmark_iteration1.*`, `evals.json`)

Four realistic tasks (CC mapping + MIDI learn, hot-plug reconnect, virtual port
without feedback, sandboxed error -50), each run once with the skill and once
without, graded against 18 objective assertions covering the skill's known
failure modes.

| Configuration | Pass rate |
|---|---|
| with skill | 100% (18/18) |
| without skill | 50% (9/18) |

Baseline failure modes observed (all predicted): `import MIDIKitIO` (pre-rename
module that no longer exists), hand-rolled `value/127.0` scaling instead of
`unitIntervalValue`, persisting endpoints by display name, claiming sandboxed
USB MIDI needs no entitlement, and diagnosing error -50 as generic `paramErr`
instead of the two documented causes.

## Trigger / description eval (`trigger_eval.json`, `trigger_*_results.json`)

20 realistic queries (10 should-trigger, 10 near-miss should-not-trigger),
3 runs each, trigger threshold 0.5.

| Description | Accuracy | Precision | Recall |
|---|---|---|---|
| original | 95% (19/20) | 90.9% | 100% |
| final (adds conceptual-question carve-out) | 100% (20/20) | 100% | 100% |

The one original failure: a purely conceptual "MIDI 1.0 vs 2.0 / UMP" question
triggered 2/3 despite needing no code. The final description appends: "Not
needed for purely conceptual MIDI-protocol questions with no Apple-platform
code involved, nor for MIDI file processing in other languages."

## Methodology notes

- skill-creator's `run_loop.py` could not run as-is in this environment: its
  improve step requires an `ANTHROPIC_API_KEY` (OAuth-only setup here), its temp
  hash-named skill copies compete with the installed skill for triggering, and
  its first-tool-call detection breaks under a superpowers-style setup that
  invokes process skills first. Trigger measurement was instead done by setting
  each candidate description on the installed skill and counting `Skill`
  invocations containing "swift-midi" in headless `claude -p` sessions
  (empty cwd; system note suppressing process skills — applied identically to
  every candidate, so comparisons are fair). Runner:
  `trigger_eval_runner.py` in the project workspace.
- Trigger rates were measured in a superpowers-equipped environment, which is
  more skill-invocation-prone than a stock setup; absolute rates may differ
  slightly elsewhere, relative comparisons should hold.
