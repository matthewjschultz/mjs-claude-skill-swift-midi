# SwiftMIDICore — events, values, filters

Distilled from the official DocC catalog in `swift-midi-core` (checkout: any
`~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts/swift-midi-core/Sources/SwiftMIDICore/SwiftMIDICore.docc/`),
verified at v1.0.1 (2026). Module: `import SwiftMIDICore` (re-exported by `SwiftMIDIIO`
and the `SwiftMIDI` umbrella — you rarely import it directly).

## Constructing and parsing events (MIDIEvent.md)

Construction = static factory methods (free-function or dot-assignment, identical):

```swift
let event = MIDIEvent.noteOn(60, velocity: .midi1(64), channel: 0)
let event: MIDIEvent = .noteOn(60, velocity: .midi1(64), channel: 0)
```

Parsing = switch on the enum, bind the payload struct, read its properties:

```swift
switch midiEvent {
case let .noteOn(payload):
    payload.note          // MIDINote
    payload.velocity      // hybrid value — see Value types below
    payload.channel       // UInt4
case let .noteOff(payload): ...
default: break
}
```

UMP `group` (0–15) exists on every constructor; default 0. MIDI 1.0 transport strips it.
Keep it 0 unless doing real MIDI 2.0 group multiplexing.
Raw encoding when needed: `midi1RawBytes()`, `midi2RawUMPWords(protocol:)`.

## Value types — the hybrid MIDI 1.0/2.0 system (Value-Types.md)

Every event value (velocity, CC value, pitch bend, …) is a hybrid type constructible
three interchangeable ways and readable three ways:

```swift
.noteOn(60, velocity: .midi1(64), channel: 0)          // 7-bit 0...127
.noteOn(60, velocity: .midi2(32768), channel: 0)       // 16-bit 0...65535 (velocity)
.noteOn(60, velocity: .unitInterval(0.5), channel: 0)  // Double 0.0...1.0

payload.velocity.midi1Value          // UInt7
payload.velocity.midi2Value          // width depends on the event — see families below
payload.velocity.unitIntervalValue   // Double 0...1  ← feed normalized params with this
```

**Doc-flagged gotcha ("⚠️ Avoid this!"):** never `switch` over a value to extract its
`.midi1`/`.midi2`/`.unitInterval` case — always read via the `*Value` properties.
The properties apply the correct **piecewise scaling**: midpoints align exactly
(velocity midi1 64 ↔ midi2 32768 ↔ 0.5, NOT naive 64/127 = 0.5039…). High→low
resolution is lossy.

**MIDI 2.0 widths differ per event type** — verified against source and by test:

- `ChanVoice7Bit16BitValue` — note velocity ONLY (midi2 is UInt16; midpoint 32768)
- `ChanVoice7Bit32BitValue` — CC value, channel pressure, poly note-pressure
  (midi2 is UInt32; midi1 64 ↔ midi2 `0x8000_0000` ↔ 0.5; midi1 127 ↔ `0xFFFF_FFFF`)
- `ChanVoice14Bit32BitValue` — pitch bend (midi1 is UInt14, center 8192 ↔ 0.5;
  midi2 is UInt32, center `0x8000_0000`)
- `ChanVoice32BitValue` — MIDI 2.0-only per-note controllers

Never assume a width — read `unitIntervalValue` for normalized work.

Integer building blocks: `UInt4`, `UInt7`, `UInt14`, `UInt7Pair`, `BytePair`
(+ `Int7`, `UInt9`, `UInt25`, `UMPWord`).

## Channel voice catalog (MIDIEvent-Channel-Voice.md)

MIDI 1.0 + 2.0 (payload struct = `MIDIEvent.<Name>`):

| Case | Constructor (key params) | Notes |
|---|---|---|
| `.noteOn(NoteOn)` | `noteOn(_:velocity:attribute:channel:group:midi1ZeroVelocityAsNoteOff:)` | note as `UInt7` or `MIDINote`; see zero-velocity note below |
| `.noteOff(NoteOff)` | `noteOff(_:velocity:attribute:channel:group:)` | |
| `.notePressure(NotePressure)` | `notePressure(note:amount:channel:group:)` | poly aftertouch |
| `.cc(CC)` | `cc(_:value:channel:group:)` | controller as `UInt7` **or named**: `.cc(.modWheel, value:…)`; value is 32-bit in MIDI 2.0 |
| `.pitchBend(PitchBend)` | `pitchBend(value:channel:group:)` | 14-bit midi1 (center 8192) / unitInterval center 0.5 |
| `.pressure(Pressure)` | `pressure(amount:channel:group:)` | channel aftertouch |
| `.programChange(ProgramChange)` | `programChange(program:bank:channel:group:)` | optional `.bankSelect(msb:lsb:)` |
| `.rpn(RPN)` | `rpn(_:change:channel:group:)` or raw `rpn(parameter:data:…)` | known enum e.g. `.pitchBendSensitivity(semitones:cents:)` |
| `.nrpn(NRPN)` | mirrors RPN | |

MIDI 2.0-only: `.noteCC(note:controller:value:channel:group:)`,
`.notePitchBend(note:value:channel:group:)`, `.noteManagement(note:flags:channel:group:)`.

**Zero-velocity noteOn — two different mechanisms, don't conflate:**
- SEND side: the `midi1ZeroVelocityAsNoteOff:` constructor param (factory default
  `false`) only affects MIDI 1.0 raw ENCODING — when true, a vel-0 noteOn encodes with
  a Note Off status byte (0x80). It never changes the constructed Swift event case.
- RECEIVE side: vel-0 noteOns arrive as `.noteOff` only if the receiver sets the
  option `.translateMIDI1NoteOnZeroVelocityToNoteOff`.

CC payload accessors: `cc.controller.number.uInt8Value`,
`cc.value.midi1Value.uInt8Value` / `cc.value.unitIntervalValue`, `cc.channel.uInt8Value`.

## System real-time (MIDIEvent-System-Real-Time.md)

No data params (only `group`): `.start()`, `.stop()`, `.continue()`, `.timingClock()`,
`.systemReset()`, `.activeSensing()` (deprecated in MIDI 2.0 but still usable — e.g. as a
network-MIDI keep-alive ping).

Clock math: 24 clocks per quarter note → BPM = `60.0 / elapsed(24 clocks)`.

## System common (MIDIEvent-System-Common.md)

`.songPositionPointer(midiBeat:)` (1 MIDI beat = 6 clocks = one 16th; seconds =
`(midiBeat/4.0) * (60.0/bpm)`), `.songSelect(number:)`, `.timecodeQuarterFrame(dataByte:)`
(use the SwiftMIDISync module for real MTC, don't hand-roll), `.tuneRequest()`.

## SysEx (MIDIEvent-System-Exclusive.md)

7-bit (MIDI 1.0+2.0): `try .sysEx7(rawBytes:)`, `try .sysEx7(rawHexString: "F043…F7")`,
`try .sysEx7(manufacturer: .oneByte(0x43), data: [...])` (F0/F7 auto-handled).
Universal: `try .universalSysEx7(universalType:deviceID:subID1:subID2:data:)`.
8-bit variants (`sysEx8`, `universalSysEx8`) are MIDI 2.0 UMP-only — NOT compatible with
7-bit SysEx at all. (`sysEx8(manufacturer:data:)` is non-throwing, unlike the sysEx7
constructors.)

## Filtering (Event-Filters.md — the workhorse for routing)

Collection extensions on `[MIDIEvent]`, chainable:

```swift
let filtered = events
    .filter(sysRealTime: .dropTypes([.activeSensing]))
    .filter(chanVoice: .keepChannel(0x0))
    .filter(chanVoice: .keepCCs([1, 11, 64]))
```

Three mask families per category — `only*` (retain only matches, drops other categories),
`keep*` (retain matches + all non-category events), `drop*` (remove matches, keep rest):

- **chanVoice:** `.only`, `.onlyType(.noteOn)`, `.onlyTypes([…])`, `.onlyChannel(0x0)`,
  `.onlyChannels([…])`, `.onlyCC(1)`, `.onlyCCs([…])`, `.onlyNotesInRange(40...80)`,
  `.onlyNotesInRanges([…])` + keep/drop variants of each.
- **sysCommon / sysEx / sysRealTime:** `.only`, `.onlyType(…)`, `.onlyTypes([…])`,
  `.keepType(…)`, `.keepTypes([…])`, `.drop`, `.dropType(…)`, `.dropTypes([…])`.
- **UMP group:** `.filter(group: 0)`, `.filter(groups: [0,1])`, `.drop(group:)`, `.drop(groups:)`.

Per-event classification (on a single `MIDIEvent`): `isChannelVoice`,
`isChannelVoice(ofType:)/(ofTypes:)`, `isSystemRealTime`, `isSystemCommon`,
`isSystemExclusive`, `isUtility` (+ typed variants). Proxy properties readable across
cases: `event.channel`, `event.group`.

`MIDIEventFilterGroup` = a configurable series of filters applicable to event arrays.

## MIDINote

Note number ↔ name (`"A#-1"`) ↔ frequency (Hz) conversions; `number: UInt7`;
`stringValue()`, `frequencyValue(tuning:)`. `MIDINote` is NOT integer-literal
expressible — construct with `MIDINote(60)` (event constructors accept `UInt7` or
`MIDINote` for note params). Supporting: `MIDINoteRange`, `MIDINoteNumberRange`.
(Method-level detail lives in source doc-comments, not the .docc markdown — read
`checkouts/swift-midi-core/Sources/SwiftMIDICore/Events/MIDINote/` if needed.)

## Not covered by the .docc prose

Packet types (`MIDIPacketData`, `UniversalMIDIPacketData`), parsers
(`MIDI1Parser`/`MIDI2Parser`), and `ReceivesMIDIEvents`/`SendsMIDIEvents` protocols exist
but have no prose docs — read the source `///` comments in the checkout when needed.
