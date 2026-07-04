---
name: swift-midi
description: 'Work with MIDI in Swift apps using orchetect''s swift-midi package family (the 2026 successor to MIDIKit) — device I/O, virtual endpoints, CC/note/transport events, MIDI learn, clock sync, network/Bluetooth MIDI, sandbox entitlements. Use this whenever a task touches MIDI in Swift/SwiftUI on Apple platforms: adding MIDI device support, mapping CC to parameters, note-triggered actions, MIDI learn UI, BPM/clock sync, virtual MIDI ports, or debugging MIDI connections — even if the user just says "hook up my MIDI controller", "make knobs respond to hardware", or mentions "MIDIKit" (the old name). Training-data knowledge of MIDIKit is stale; this skill carries the verified 1.x API. Not needed for purely conceptual MIDI-protocol questions with no Apple-platform code involved, nor for MIDI file processing in other languages.'
---

# swift-midi (formerly MIDIKit)

**The one fact that prevents most errors:** orchetect's MIDIKit mono-repo was renamed
and split into the modular **swift-midi** family in April 2026 (v1.0). Module names,
package URLs, and some APIs differ from what training data knows as "MIDIKit". Never
write `import MIDIKit`. When an API detail isn't in this skill, read the real source —
any DerivedData checkout under
`~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts/swift-midi-*` —
rather than recalling from memory.

## Package selection

| Need | Depend on | `import` |
|---|---|---|
| MIDI I/O + events (the usual case) | `swift-midi-io` | `SwiftMIDIIO` (re-exports core) |
| Everything incl. MTC sync, .mid files, HUI, SwiftUI controls | `swift-midi` umbrella | `SwiftMIDI` |
| Events/values only, no I/O | `swift-midi-core` | `SwiftMIDICore` |

```swift
.package(url: "https://github.com/orchetect/swift-midi-io", from: "1.0.0")
```

MIT license (add a notice to the app's acknowledgements screen for a shipping app).
Apple platforms only for I/O; macOS 10.13+ / iOS 12+; no tvOS/watchOS. Swift 6
strict-concurrency compliant. The manager class is literally named `MIDIManager` —
qualify as `SwiftMIDIIO.MIDIManager` if your own wrapper shares the name (a common
collision when apps wrap the library in their own `MIDIManager` class).

## Quickstart (receive CC + notes, macOS 14+ SwiftUI)

```swift
import SwiftMIDIIO

let midiManager = ObservableMIDIManager(      // @Observable; plain MIDIManager also fine
    clientName: "MyApp", model: "MyApp", manufacturer: "MyCompany"
)
try midiManager.start()                        // BEFORE anything else; error -50 = forgot this (or missing entitlement)

try midiManager.addInputConnection(
    to: .allOutputs,                           // RECEIVE = connect to THEIR outputs
    tag: "MainInput",
    filter: .owned(),                          // don't hear your own virtual ports
    receiver: .events(options: [.translateMIDI1NoteOnZeroVelocityToNoteOff]) { events, timeStamp, source in
        // ⚠️ background thread — funnel to @MainActor before touching UI/engine
        for event in events {
            switch event {
            case let .cc(p):     _ = (p.controller.number.uInt8Value, p.value.unitIntervalValue, p.channel.uInt8Value)
            case let .noteOn(p): _ = (p.note.number.uInt8Value, p.velocity.unitIntervalValue)
            case let .noteOff(p): _ = p.note.number.uInt8Value
            default: break
            }
        }
    }
)
```

## The five load-bearing rules

1. **Direction:** endpoints are named from the other end's perspective. RECEIVE =
   `addInputConnection(to: .allOutputs / .outputs(matching:))`. SEND =
   `addOutputConnection(to: .allInputs / .inputs(matching:))`. "Connected but silent"
   → check this first.
2. **Threading:** every receive callback fires on a Core MIDI background thread. Keep
   the handler `nonisolated`, funnel results to the main actor; never mutate observable
   state or touch an audio engine from the handler.
3. **Managed connections self-heal:** they auto-reconnect when endpoints vanish and
   reappear, and may target not-yet-present endpoints. Don't rebuild connections on
   hot-plug notifications — use notifications only for picker UI. Rebuild only to
   change targets.
4. **Values:** read event values via `.midi1Value` / `.midi2Value` /
   `.unitIntervalValue` — never switch on the raw value enum (doc-flagged; the
   properties apply correct piecewise 1.0↔2.0 scaling; use `unitIntervalValue` to feed
   normalized parameters). MIDI 2.0 widths differ per event type (velocity 16-bit, CC
   32-bit) — one more reason to read `unitIntervalValue` and never assume widths.
5. **Identity:** persist endpoints by `uniqueID` (or `.uniqueIDWithFallback`), never by
   name. For your own virtual ports use `uniqueID: .userDefaultsManaged(key:)`; never
   hand-generate IDs.

## Entitlements cheat sheet

- Sandboxed macOS + USB MIDI devices: `com.apple.security.device.usb`
  (if you dual-distribute, add it to BOTH the App Store and Developer-ID entitlement files).
- Sandboxed macOS + network MIDI: `network.client` AND `network.server`.
- iOS virtual ports or background MIDI: Background Modes → Audio (+ actually play
  silent audio to stay alive in background).
- Bluetooth MIDI pairing UI (iOS): `NSBluetoothAlwaysUsageDescription`; real device
  only, never the simulator.

## Where to go deeper (read the matching reference before writing code)

- **`references/io.md`** — MIDIManager lifecycle, connections/virtual ports/thru,
  all receiver types + options, notifications, endpoint identity/filters,
  SwiftUI/Observable/AsyncStream integration, USB/network/Bluetooth/background
  transports, doc-flagged bugs (persistent thru broken on macOS 11/12).
- **`references/events.md`** — full MIDIEvent catalog with constructors and payload
  accessors, the hybrid MIDI 1.0/2.0 value system, collection filter API
  (`events.filter(chanVoice: .onlyCCs([1,11,64]))`), SysEx, MIDINote.
