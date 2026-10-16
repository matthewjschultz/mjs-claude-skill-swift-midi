# mjs-claude-skill-swift-midi

A Claude Code skill for writing MIDI code in Swift apps with orchetect's swift-midi package family, the successor to MIDIKit.

## Why this exists

MIDIKit was renamed and split into the modular swift-midi packages in April 2026. Claude's training data mostly predates that change, so without help it tends to write imports and API calls that no longer exist. This skill carries the current, verified 1.x API: device I/O, virtual endpoints, CC and note events, MIDI learn, clock sync, network and Bluetooth MIDI, and macOS sandbox entitlements.

Every code snippet in the skill compiles against pinned library versions (swift-midi-io 1.1.0, swift-midi-core 1.0.1), and the factual claims were checked against the library source. Evaluation results are in `evals/`: on four realistic MIDI tasks the skill scored 100% on 18 objective assertions versus 50% without it, and the trigger description scored 100% on a 20 query eval set.

## Contents

- `skills/swift-midi/SKILL.md`: package selection, a receive quickstart, the five rules that prevent the most common mistakes, and an entitlements cheat sheet.
- `skills/swift-midi/references/io.md`: MIDIManager lifecycle, managed connections, virtual ports, thru connections, receivers, notifications, endpoint identity, and transports.
- `skills/swift-midi/references/events.md`: the MIDIEvent catalog, the hybrid MIDI 1.0/2.0 value system, event filters, SysEx, and MIDINote.
- `evals/`: task evals, trigger evals, and benchmark results.
- `swift-midi.skill`: the packaged skill file.

## Install

```
/plugin marketplace add matthewjschultz/mjs-claude-skills
/plugin install swift-midi@mjs-claude-skills
```

Or download `swift-midi.skill` and install it directly.

## Credits

This skill documents [swift-midi](https://github.com/orchetect/swift-midi) by Steffan Andrews (orchetect), MIT license. It is an independent companion and is not affiliated with or endorsed by the library author.

## License

MIT. See [LICENSE](LICENSE).
