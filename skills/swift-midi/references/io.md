# SwiftMIDIIO — manager, connections, receivers, transports

Distilled from the official DocC catalog + source in `swift-midi-io` (checkout: any
`~/Library/Developer/Xcode/DerivedData/*/SourcePackages/checkouts/swift-midi-io/`),
verified at v1.1.0 (2026). `import SwiftMIDIIO`. Apple platforms only; no tvOS/watchOS
(no CoreMIDI there). MIT.

## MIDIManager — one per app, for the app's lifetime

```swift
let midiManager = MIDIManager(
    clientName: "MyApp",       // internal CoreMIDI client ID, not user-visible
    model: "MyApp",            // user-visible, baked into virtual ports
    manufacturer: "MyCompany"  // optional 4th param: notificationHandler
)
try midiManager.start()        // REQUIRED before anything else; later calls no-op
```

- One instance app-wide, stored at app scope. Don't recreate mid-lifecycle.
- **Core MIDI error -50 ⇒ you called something before `start()`, or a sandboxed/iOS app
  is missing a required entitlement.** This is the documented meaning — check these two
  causes before anything else.
- No manual cleanup: manager and everything it owns dispose on deinit; removing an
  individual port/connection disposes it.
- Managed dictionaries, keyed by the `tag` you supplied: `managedInputs`,
  `managedOutputs`, `managedInputConnections`, `managedOutputConnections`,
  `managedThruConnections`.

### SwiftUI integration — pick ONE

1. **`ObservableMIDIManager`** — `@Observable` subclass; `devices`/`endpoints` become
   observable. Requires macOS 14 / iOS 17. Inject via `.environment(midiManager)`,
   consume via `@Environment(ObservableMIDIManager.self)`. Same init as `MIDIManager`
   (it declares no initializers of its own).
2. **`ObservableObjectMIDIManager`** — Combine `ObservableObject` subclass for
   macOS 10.15 / iOS 13 back-compat.
3. **AsyncStreams** on any plain manager: `devicesStream()` / `endpointsStream()` —
   emit current value immediately, then on change; designed for `.task { for await … }`.
4. Manual: just use `notificationHandler`. No KVO anywhere.

## Connections — managed means AUTO-RECONNECTING

`addInputConnection`/`addOutputConnection` create **managed connections**: after
creation they automatically re-subscribe when target endpoints disappear and reappear,
and criteria may name endpoints that don't exist yet (they connect when they show up).
**You do NOT need to tear down and rebuild connections on device hot-plug** — that
manual dance (seen in older app code) is only needed to *change* what you target.

```swift
// RECEIVE: connect your input to THEIR OUTPUTS (direction is from the other end's view)
try midiManager.addInputConnection(
    to: .allOutputs,                    // or .outputs(matching: [.uniqueID(id)])
    tag: "MainInput",
    filter: .owned(),                   // exclude your OWN virtual outputs — avoids feedback loop
    receiver: .events { events, timeStamp, source in … }   // ⚠️ background thread
)

// SEND: connect your output to THEIR INPUTS
try midiManager.addOutputConnection(
    to: .inputs(matching: [.uniqueID(id)]),   // or .allInputs
    tag: "MainOut"
)
try midiManager.managedOutputConnections["MainOut"]?.send(event: .noteOn(60, velocity: .midi1(127), channel: 0))
// send(events: [MIDIEvent]) also exists
```

- Connection modes: `.allOutputs` / `.outputs(matching: Set<MIDIEndpointIdentity>)` /
  `.none`; mirrored as `.allInputs` / `.inputs(matching:)` / `.none`. Two overload
  families: identity criteria take the `matching:` label (Set or Array); passing
  endpoint OBJECTS uses no label — `.outputs([someOutputEndpoint])`.
- Retarget live via the connection object (non-throwing):
  `conn?.add(outputs: [.uniqueID(id)])`, `conn?.remove(outputs: […])`.
- Remove: `midiManager.remove(.inputConnection, .withTag("MainInput"))`,
  `remove(.output, .all)`, or `removeAll()`.

### Endpoint filters on connections

`MIDIEndpointFilter(owned:criteria:)`; presets `.default()` (nothing filtered) and
`.owned()` (drop manager-owned virtual endpoints — use this with `.allOutputs` whenever
you also create virtual ports). Standalone use:
`endpoints.filter(filter, in: midiManager)` / `endpoints.filter(dropping:in:)`.

## Virtual ports (what other apps see)

```swift
try midiManager.addInput(                       // a virtual destination others SEND TO
    name: "MyApp MIDI In",                      // user-visible
    tag: "VirtIn",
    uniqueID: .userDefaultsManaged(key: "VirtIn"),
    receiver: .events { events, _, _ in … }     // source param is always nil here
)
try midiManager.addOutput(                      // a virtual source others LISTEN TO
    name: "MyApp MIDI Out", tag: "VirtOut",
    uniqueID: .userDefaultsManaged(key: "VirtOut")
)
midiManager.managedOutputs["VirtOut"]?.send(event: …)
```

- iOS/Catalyst: virtual ports require the Background Modes **Audio** entitlement.
- The receiver's `source` endpoint parameter is populated only for
  `MIDIInputConnection`s; it is always nil on virtual `MIDIInput`s.

### Unique ID persistence (`MIDIIdentifierPersistence`)

- `.userDefaultsManaged(key:suite:)` — **recommended**: auto-generates, persists,
  regenerates + re-stores on collision.
- `.managedStorage(readHandler:storeHandler:)` — same semantics, custom backing store.
- `.unmanaged(id)` — you store it; MUST re-read `uniqueID` after creation (collisions
  can change it). `.adHoc` — random every launch, testing only.
- Doc rule: NEVER hand-generate an ID; Core MIDI generates it, you persist what it made.

## Receivers (`MIDIReceiver`)

- `.events(options:_:)` — **recommended**; typed `[MIDIEvent]` + timestamp + source.
- `.eventsLogging(options:_:)` — auto console logging, DEBUG builds only.
- `.rawData` / `.rawDataLogging` — raw packets, debugging only.
- `.strong(obj, options:)` / `.weak(obj, options:)` — object conforming to
  `ReceivesMIDIEvents` held strongly/weakly.
- `.group([MIDIReceiver])` — chain several receivers.
- All receive callbacks fire on a Core MIDI **background thread** — hop to main for UI.
- There is NO AsyncStream for events — streams exist only for devices/endpoints lists.

`MIDIReceiverOptions` (OptionSet, combinable):
- `.translateMIDI1NoteOnZeroVelocityToNoteOff` — vel-0 noteOn arrives as `.noteOff`.
- `.filterActiveSensingAndClock` — drop the spam at the receiver level.
- `.bundleRPNAndNRPNDataEntryLSB` — merge split RPN/NRPN MSB/LSB pairs into one event
  (tiny latency on those messages only).

## System notifications (`MIDIIONotification`)

Set `midiManager.notificationHandler` (`@Sendable (MIDIIONotification) -> Void`; may
fire off-main; can also be passed to the manager's init). Cases: `.setupChanged`
(redundant if you handle specific ones), `.added(object:parent:)`,
`.removed(object:parent:)`, `.propertyChanged(property:forObject:)`,
`.thruConnectionChanged`, `.serialPortOwnerChanged`, `.ioError(device:error:)`,
`.internalStart`, `.other(…)`.
Use these to refresh device-picker UI — NOT to rebuild managed connections (those
self-heal). Safe to ignore entirely if you don't need a live device list.

## Endpoint identity for persistence (`MIDIEndpointIdentity`)

- `.uniqueID(id)` — recommended.
- `.uniqueIDWithFallback(id:fallbackDisplayName:)` — for vendors whose endpoints don't
  persist their own ID reliably.
- `.endpoint(someEndpoint)` — static helper that builds the right identity for storage.
- `.name` / `.displayName` — first-match, collision-prone, debugging only.
- Helpers: `identity.locate(in: endpoints.outputs)`, `identity.matches(endpoint:)`.

Endpoint collections: `midiManager.endpoints.inputs` / `.outputs` (flat — use these);
`midiManager.devices` for the Device→Entity hierarchy (rarely needed).

## Thru connections (OS-level pass-through)

`addThruConnection(outputs:inputs:tag:lifecycle:params:)` — routes up to 8→8 endpoints
inside the OS with zero app code after setup. `.nonPersistent` (manager-owned, default)
or `.persistent(ownerID:)` (survives reboot; look up via
`unmanagedPersistentThruConnections(ownerID:)`, purge via
`removeAllUnmanagedPersistentThruConnections(ownerID:)`).
**Doc-flagged bug: persistent thru connections are broken on macOS 11/12 and iOS 14/15
(no workaround).**

## Transports

- **USB**: class-compliant just works, no code. iOS↔Mac direct USB via Audio MIDI Setup
  → Audio Devices → Enable ("IDAM MIDI Host" endpoints appear on iOS).
- **Network (RTP-MIDI)**: call once at startup
  `setMIDINetworkSession(policy: .anyone /* or .hostsInContactList */)` — a free
  function; the policy type is `MIDIIONetworkConnectionPolicy?` (`.noOne` /
  `.hostsInContactList` / `.anyone`); passing `nil` disables network MIDI. User pairs
  via Audio MIDI Setup. **Sandboxed macOS needs BOTH network entitlements
  (`com.apple.security.network.client` + `com.apple.security.network.server`).** iOS
  needs none. Doc-flagged: "Live Routings" broken on some macOS versions.
- **Bluetooth**: iOS physical device only (never the simulator); use CoreAudioKit's
  `CABTMIDICentralViewController` / `CABTMIDILocalPeripheralViewController` for pairing
  UI; needs `NSBluetoothAlwaysUsageDescription` in Info.plist. Paired devices then
  appear as ordinary endpoints.
- **iOS background MIDI**: Background Modes Audio entitlement + actually playing
  (silent) audio keeps the app alive; the docs include a full `BackgroundAudioManager`
  recipe (Send-and-Receive-on-iOS-in-Background.md).

## Canonical minimal listener (from the official example)

```swift
import SwiftMIDIIO

public class MIDIHelper {
    private let midiManager = MIDIManager(clientName: "MyApp", model: "MyApp", manufacturer: "MyCompany")
    public init() {
        do {
            try midiManager.start()
            try midiManager.addInput(
                name: "MyApp MIDI In", tag: "Virtual_MIDI_In",
                uniqueID: .userDefaultsManaged(key: "Virtual_MIDI_In"),
                receiver: .events { [weak self] events, timeStamp, source in
                    DispatchQueue.main.async { events.forEach { self?.received(midiEvent: $0) } }
                }
            )
        } catch { print("MIDI Setup Error:", error) }
    }
    private func received(midiEvent: MIDIEvent) {
        switch midiEvent {
        case let .noteOn(p): print("Note On:", p.note, p.velocity, p.channel)
        case let .cc(p):     print("CC:", p.controller, p.value, p.channel)
        default: break
        }
    }
}
```
