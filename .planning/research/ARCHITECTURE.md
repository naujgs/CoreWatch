# Architecture Research

**Domain:** iOS system metrics dashboard (sideloaded, single-screen SwiftUI MVVM)
**Researched:** 2026-05-14 (v1.2 — supersedes v1.1 document dated 2026-05-13)
**Confidence:** HIGH

---

## Standard Architecture

### System Overview

```
CoreWatchApp (@main)
  └── ContentView
        └── TemperatureViewModel (@State, @Observable @MainActor)   ← one ViewModel, expanded
              ├── Thermal domain
              │     ├── thermalState: ProcessInfo.ThermalState
              │     └── history: [ThermalReading]                    ← ring buffer (360 entries)
              ├── CPU domain  [v1.2 new]
              │     ├── cpuUsage: Double                             ← 0.0–1.0
              │     └── cpuHistory: [SystemReading]
              ├── Memory domain  [v1.2 new]
              │     ├── memoryUsedBytes: UInt64
              │     └── memoryHistory: [SystemReading]
              ├── Battery domain  [v1.2 new]
              │     ├── batteryLevel: Float                          ← 0.0–1.0
              │     └── batteryState: UIDevice.BatteryState
              └── Notification / background state (unchanged)
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| `TemperatureViewModel` | All data acquisition, state, history, alerting | `@Observable @MainActor final class` |
| `ThermalReading` (existing) | One thermal snapshot for the chart | `struct` with `id`, `timestamp`, `state` |
| `SystemReading` (new) | One numeric snapshot for CPU or memory charts | `struct` with `id`, `timestamp`, `value: Double` |
| `ContentView` | Dumb display — reads ViewModel, owns no data logic | SwiftUI `View` |
| Mach C API helpers | Pure nonisolated functions: read CPU ticks and memory | Free functions in a new `SystemMetrics.swift` file |

---

## Recommended Project Structure

```
CoreWatch/
├── TemperatureViewModel.swift   # expanded — CPU/memory/battery properties added
├── ContentView.swift            # expanded — new metric panels
├── ThermalReading.swift         # (extract if currently inline) unchanged
├── SystemReading.swift          # NEW — shared value type for CPU and memory history
├── SystemMetrics.swift          # NEW — nonisolated free functions wrapping Mach C APIs
└── CoreWatch-Bridging-Header.h # unchanged — Mach headers already available via SDK
```

### Structure Rationale

- **`SystemMetrics.swift`:** Isolates all Mach C API calls into `nonisolated` free functions. They have no actor context, return plain Swift value types, and are trivially testable. Keeps ViewModel clean of low-level pointer arithmetic.
- **`SystemReading.swift`:** A single `struct SystemReading: Identifiable` shared by CPU history and memory history arrays. Avoids duplicating an identical type twice.
- Everything else stays in the existing two files — no services layer, no coordinator, no DI container. The app is a single-screen tool; structural overhead would add zero value.

---

## Architectural Patterns

### Pattern 1: One Expanded ViewModel (Not Multiple ViewModels)

**What:** Keep a single `TemperatureViewModel` and add CPU, memory, and battery properties to it rather than splitting into `CPUViewModel`, `MemoryViewModel`, etc.

**When to use:** When all data sources share the same polling cadence, the same notification cooldown logic, and are displayed on the same screen with no navigation between them.

**Trade-offs:**
- Pro: Single source of truth. `ContentView` reads one `@State` object. `scenePhase` lifecycle hooks call one `startPolling()` / `stopPolling()`. Cooldown logic and background task management stay in one place.
- Pro: All `@Observable` mutation stays on `@MainActor`. No cross-actor coordination needed.
- Con: File grows to ~600–700 lines. Acceptable for a personal tool. Mitigate by extracting Mach helpers to `SystemMetrics.swift`.
- Reject split ViewModels because: SwiftUI `@State` ownership requires the ViewModel to live in `ContentView`. Multiple `@State` ViewModels on a single `ContentView` is awkward — you cannot inject one into another without a `@Bindable` or `@Environment` hop that adds complexity for no gain.

**Example (adding CPU to the existing ViewModel):**
```swift
@Observable
@MainActor
final class TemperatureViewModel {
    // --- existing ---
    private(set) var thermalState: ProcessInfo.ThermalState = .nominal
    private(set) var history: [ThermalReading] = []

    // --- v1.2 additions ---
    private(set) var cpuUsage: Double = 0          // 0.0 – 1.0
    private(set) var cpuHistory: [SystemReading] = []
    private(set) var memoryUsedBytes: UInt64 = 0
    private(set) var memoryHistory: [SystemReading] = []
    private(set) var batteryLevel: Float = 0
    private(set) var batteryState: UIDevice.BatteryState = .unknown
}
```

### Pattern 2: Mach C API Calls via nonisolated Free Functions

**What:** Wrap `host_cpu_load_info` and `task_vm_info` in `nonisolated` free functions that return plain Swift value types. Call those functions synchronously from the `@MainActor` ViewModel's poll method.

**When to use:** Whenever a C API is synchronous, sub-millisecond, and returns a value type (not a class or reference type that would need `Sendable` conformance).

**Trade-offs:**
- Pro: No concurrency boundary crossing. Swift 6 strict concurrency is fully satisfied because the C call site is on `@MainActor` and the result is a plain value type.
- Pro: No `Task.detached` or `await` required. The call is inlined in the existing synchronous `updateThermalState()`.
- Pro: `nonisolated` free functions can be called from any isolation context — no actor annotation needed.
- Con: The Mach call blocks the main thread for the duration of the kernel call (~0.05–0.5 ms). This is the same as `ProcessInfo.processInfo.thermalState`, which already blocks the main thread. Acceptable at 10s polling cadence.
- Reject `Task.detached` pattern because: it would require crossing an actor boundary to store the result (back on `@MainActor`), adding `await`, and making the update non-atomic. The synchronous call is strictly simpler and correct.

**Swift 6 concurrency rules for C APIs (HIGH confidence):**
- C functions imported from Mach headers (`host_statistics`, `task_info`, etc.) are global C functions. Swift treats them as `nonisolated`.
- They can be called from any isolation context (including `@MainActor`) without a concurrency boundary crossing.
- Their arguments and return values are C types (`integer_t`, `mach_msg_type_number_t`, etc.) — not Swift types, not `Sendable`. The Swift concurrency checker does not reason about them.
- The only Swift types that must be `Sendable` are values that cross actor boundaries. If the Mach call is made and its result consumed within the same `@MainActor` method, nothing crosses a boundary.

**Example — CPU usage nonisolated helper in `SystemMetrics.swift`:**
```swift
import Darwin

// Stored once; mach_host_self() allocates a kernel port — do not call per-poll.
// nonisolated(unsafe) because it is a C port value, not a Swift Sendable type.
private nonisolated(unsafe) let _machHost: mach_port_t = mach_host_self()

// Previous tick snapshot for delta calculation. Must persist across calls.
// nonisolated(unsafe) is correct here: this variable is accessed only from
// updateSystemMetrics(), which is always called on @MainActor. No concurrent
// access is possible. The unsafe annotation documents the invariant.
private nonisolated(unsafe) var _previousCPULoad = host_cpu_load_info()

/// Returns current CPU usage as a fraction 0.0–1.0, or nil on kernel error.
/// Nonisolated: pure C calls, returns a value type. Safe to call from @MainActor.
nonisolated func readCPUUsage() -> Double? {
    var info = host_cpu_load_info()
    var count = HOST_CPU_LOAD_INFO_COUNT
    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: Int(count)) {
            host_statistics(_machHost, HOST_CPU_LOAD_INFO, $0, &count)
        }
    }
    guard result == KERN_SUCCESS else { return nil }

    let user = Double(info.cpu_ticks.0 - _previousCPULoad.cpu_ticks.0)
    let sys  = Double(info.cpu_ticks.1 - _previousCPULoad.cpu_ticks.1)
    let idle = Double(info.cpu_ticks.2 - _previousCPULoad.cpu_ticks.2)
    let nice = Double(info.cpu_ticks.3 - _previousCPULoad.cpu_ticks.3)
    let total = user + sys + idle + nice
    _previousCPULoad = info
    guard total > 0 else { return nil }
    return (user + sys + nice) / total
}
```

**Important:** The first call to `readCPUUsage()` will return an inaccurate result (delta against zeroed initial state). Discard the first sample or pre-warm by calling once in `init()` and discarding the result. The second and subsequent calls at 10s intervals are accurate.

**Example — App memory footprint helper:**
```swift
/// Returns app physical memory footprint in bytes, or nil on kernel error.
/// Uses task_vm_info / phys_footprint — matches Xcode Debug Navigator's value.
/// Do NOT use mach_task_basic_info.resident_size — it diverges from Instruments.
nonisolated func readMemoryFootprint() -> UInt64? {
    var info = task_vm_info_data_t()
    var count = mach_msg_type_number_t(MemoryLayout<task_vm_info_data_t>.size / MemoryLayout<integer_t>.size)
    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: Int(count)) {
            task_info(mach_task_self_, TASK_VM_INFO, $0, &count)
        }
    }
    guard result == KERN_SUCCESS else { return nil }
    return info.phys_footprint
}
```

**Note on `TASK_VM_INFO_COUNT` and `TASK_VM_INFO_REV1_COUNT`:** These constants are not auto-imported by the Swift C importer in some SDK versions. Compute `count` manually from `MemoryLayout` as shown above — this is the standard workaround documented in Apple Developer Forums.

### Pattern 3: Battery via UIDevice — Event-Driven, Not Polled

**What:** Enable battery monitoring once in `startPolling()`, then read `UIDevice.current.batteryLevel` and `.batteryState` synchronously on each poll tick. Optionally supplement with `UIDeviceBatteryLevelDidChange` notification for between-tick updates.

**When to use:** Battery state changes slowly (level changes ~0.03%/min at idle). The 10s polling cadence is more than sufficient. No dedicated battery timer needed.

**Trade-offs:**
- Pro: `UIDevice` properties are public API, sandbox-safe, no entitlements needed.
- Pro: Stays on `@MainActor` — `UIDevice.current` must be accessed on the main thread (UIKit main-thread rule, same as all UIKit APIs). Since the ViewModel is `@MainActor`, this is already satisfied.
- Pro: `isBatteryMonitoringEnabled` is a one-time toggle, not a resource that needs lifecycle management.
- Con: Level is a `Float` with ~1% granularity on hardware.

**Example:**
```swift
// In startPolling():
UIDevice.current.isBatteryMonitoringEnabled = true

// In updateSystemMetrics() — called from the existing 10s timer:
batteryLevel = UIDevice.current.batteryLevel   // -1.0 if monitoring not enabled
batteryState = UIDevice.current.batteryState   // .unknown if monitoring not enabled
```

**No `Task.detached` needed.** `UIDevice` reads are synchronous and sub-microsecond. The `@MainActor` isolation of the ViewModel satisfies UIKit's main-thread requirement automatically.

### Pattern 4: Shared 10s Timer, One Combined Update Method

**What:** All three new metrics (CPU, memory, battery) are sampled in the same timer callback as the existing thermal state read. No separate timers.

**When to use:** Always, for this app. CPU and memory are O(microseconds) to read. Battery is O(nanoseconds). Separate timers add complexity with zero benefit.

**What changes in `updateThermalState()`:** Rename it to `updateAllMetrics()` (or keep the name and expand it). Add CPU, memory, and battery reads after the thermal state read.

**Cadence rationale:**
- CPU: 10s is appropriate. CPU usage is a rolling average; sub-10s cadence would create noise rather than signal.
- Memory: 10s is appropriate. App memory footprint changes slowly under normal use.
- Battery: 10s is appropriate. Battery level changes on the order of minutes.
- Thermal: Already 10s. No change.

**Example:**
```swift
private func updateAllMetrics() {
    // Thermal (existing)
    thermalState = ProcessInfo.processInfo.thermalState
    let thermalReading = ThermalReading(timestamp: Date(), state: thermalState)
    appendToHistory(&history, thermalReading, max: Self.maxHistory)
    checkAndFireNotification()

    // CPU (new)
    if let usage = readCPUUsage() {
        cpuUsage = usage
        let cpuReading = SystemReading(timestamp: Date(), value: usage)
        appendToHistory(&cpuHistory, cpuReading, max: Self.maxHistory)
    }

    // Memory (new)
    if let bytes = readMemoryFootprint() {
        memoryUsedBytes = bytes
        let memReading = SystemReading(timestamp: Date(), value: Double(bytes))
        appendToHistory(&memoryHistory, memReading, max: Self.maxHistory)
    }

    // Battery (new)
    batteryLevel = UIDevice.current.batteryLevel
    batteryState = UIDevice.current.batteryState
}
```

### Pattern 5: Mach Port Lifecycle — Acquire Once, Reuse Forever

**What:** Call `mach_host_self()` once at module initialization and store the result in a file-private constant. Never call it per poll tick.

**Why:** `mach_host_self()` allocates a kernel port right on every call. The returned port represents the same kernel host object each time, but each call allocates a new name in the process's Mach port namespace, consuming a finite kernel resource. Calling it at 10s intervals for hours would leak port names. Store once, reuse forever.

**Evidence:** SystemKit (widely referenced iOS/macOS system monitoring library) uses exactly this pattern — `static let machHost = mach_host_self()` initialized once at the struct level.

**Contrast with `mach_task_self_`:** This is a macro/global that is always valid — no allocation, no call needed. Use it directly in `task_info()`.

**`IOKit` services in `readIOKitTemperature()` (existing v1.1 code):** The current pattern of acquiring the service per call and releasing it with `defer { IOObjectRelease(service) }` is correct for IOKit — do not cache the io_object_t across calls. IOKit objects have reference-counted lifetimes and the service reference may become stale.

### Pattern 6: UI Organization — Vertical ScrollView with Metric Cards

**What:** Replace the existing `VStack` root in `ContentView` with a `ScrollView` containing a vertical stack of metric cards. Each card is a standalone SwiftUI subview.

**When to use:** When content exceeds the visible screen height on smaller devices (iPhone SE), or when new panels will be added iteratively.

**Why not TabView:** The data is all related system health info — a user glancing at this app wants to see everything at once, not switch tabs. TabView implies independent, non-simultaneous concerns. Metrics on a dashboard are complementary, not alternative.

**Why not expanding cards (DisclosureGroup):** Adds unnecessary interaction for a glance-first tool. The user opens the app to see the state — hiding data behind a tap creates friction.

**Recommended card layout:**
```
ScrollView (vertical)
  VStack(spacing: 16)
    ┌─ App header "CoreWatch" ─────────────────────────────┐
    │                                                        │
    ├─ ThermalCard ─────────────────────────────────────────┤
    │  Colored badge (Nominal/Fair/Serious/Critical)         │
    │  Thermal history chart (existing)                      │
    │                                                        │
    ├─ CPUCard ─────────────────────────────────────────────┤
    │  "CPU" label + current % (large number)                │
    │  Mini line chart (cpuHistory)                          │
    │                                                        │
    ├─ MemoryCard ──────────────────────────────────────────┤
    │  "Memory" label + current MB (large number)            │
    │  Mini line chart (memoryHistory)                       │
    │                                                        │
    └─ BatteryCard ─────────────────────────────────────────┘
       Battery level % + charging state badge
       No chart needed — level changes too slowly to be useful
```

**Subview decomposition:** Extract each card into a private SwiftUI `View` struct within `ContentView.swift`, or as separate files if the file grows past ~300 lines. Pass ViewModel data as value-type parameters (not the ViewModel itself) to keep cards self-contained and previewable.

**Example card struct:**
```swift
private struct CPUCard: View {
    let usage: Double          // 0.0–1.0
    let history: [SystemReading]

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("CPU")
                .font(.headline)
            Text(String(format: "%.0f%%", usage * 100))
                .font(.largeTitle.monospacedDigit())
            // Mini chart...
        }
        .padding(16)
        .background(RoundedRectangle(cornerRadius: 16).fill(Color(.secondarySystemBackground)))
    }
}
```

---

## Data Flow

### Poll Cycle (Foreground, every 10s)

```
Timer.publish(every: 10) → onReceive
    → updateAllMetrics()
          ├── ProcessInfo.processInfo.thermalState    (public, sync, main thread)
          ├── readCPUUsage()                          (nonisolated C call, sync, <1ms)
          ├── readMemoryFootprint()                   (nonisolated C call, sync, <1ms)
          └── UIDevice.current.batteryLevel/State     (UIKit, sync, main thread)
    → @Observable mutation (all properties)
    → SwiftUI auto-redraw (ContentView and card subviews)
```

### Background Path (unchanged from v1.1)

```
thermalStateDidChangeNotification
    → handleBackgroundThermalChange()
          → ProcessInfo.processInfo.thermalState
          → checkAndFireNotification()
    NOTE: CPU/memory/battery NOT read in background path.
          beginBackgroundTask window is ~30s; only thermal alerting needed.
```

### Key Data Flows

1. **CPU history:** `readCPUUsage()` returns `Double?`. On non-nil: append `SystemReading(timestamp: Date(), value: usage)` to `cpuHistory[]` ring buffer. Same ring-buffer mechanics as `history[]`.
2. **Memory history:** Same pattern as CPU, value is `Double(memoryUsedBytes)` in bytes. Format as MB in the View (`value / 1_048_576`).
3. **Battery:** No history array. Display current level and state only. Level changes are too slow (~1 reading/min perceptible change) to make a chart meaningful.

---

## Scaling Considerations

This is a personal single-device tool — scaling in the user-count sense is not applicable. Relevant scaling is complexity growth as features are added.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 3–4 metrics (current target) | Single ViewModel, expanded `updateAllMetrics()`, no service layer needed |
| 6–8 metrics | Consider extracting Mach helpers to a dedicated `SystemMetrics` actor or struct with static methods; ViewModel stays as coordinator |
| >8 metrics or multi-screen | Services layer warranted; each domain (thermal, cpu, memory, network) becomes a focused model object injected into a coordinator ViewModel |

### Scaling Priorities

1. **First constraint:** `updateAllMetrics()` method length. At 4 metrics with history appends it will be ~40 lines. Beyond ~80 lines, extract per-domain helpers (`updateCPUMetrics()`, etc.) called from a single orchestrator method.
2. **Second constraint:** `ContentView.swift` file length. At 4 metric cards it will reach ~300 lines. Extract cards to separate files before the file becomes hard to navigate.

---

## Anti-Patterns

### Anti-Pattern 1: Multiple ViewModels for a Single-Screen Dashboard

**What people do:** Create `CPUViewModel`, `MemoryViewModel`, `BatteryViewModel` alongside `TemperatureViewModel`.

**Why it's wrong:** All four data sources share the same timer, the same `scenePhase` lifecycle hooks, and the same screen. Splitting forces `ContentView` to hold multiple `@State` ViewModel instances with no coordination between them — polling cadence, cooldown logic, and background task management must be duplicated or synchronized manually. This is pure overhead.

**Do this instead:** One `TemperatureViewModel` renamed (optionally) to `SystemViewModel` or `DashboardViewModel`, with all metric properties consolidated.

### Anti-Pattern 2: Task.detached for Mach C API Calls

**What people do:** Wrap `host_cpu_load_info` in `Task.detached { ... }` to "avoid blocking the main thread."

**Why it's wrong:** The Mach calls complete in under 1ms — the same order of magnitude as `ProcessInfo.processInfo.thermalState` (which is already called on main). Offloading to a detached task requires an actor-boundary crossing to write the result back to `@MainActor` properties, adding `await`, making the update asynchronous, and potentially causing a frame where the UI shows stale data. The complexity buys nothing.

**Do this instead:** Call synchronously from the `@MainActor` method. If a Mach call ever takes >1ms (it won't for these APIs), profile first, then optimize.

### Anti-Pattern 3: Calling mach_host_self() Per Poll Tick

**What people do:** Call `mach_host_self()` inside `readCPUUsage()` on every timer fire.

**Why it's wrong:** Each call allocates a new Mach port name in the process namespace, a finite kernel resource. Over hours of polling at 10s intervals this is ~360 leaked port names per hour. Mach port exhaustion causes crashes.

**Do this instead:** Store the result once in a file-private `nonisolated(unsafe) let` constant at module scope. Mark `nonisolated(unsafe)` because it is a C integer type that the Swift concurrency system cannot verify as `Sendable` — but it is safe in practice because it is a read-only constant after initialization.

### Anti-Pattern 4: Using resident_size for Memory Display

**What people do:** Use `mach_task_basic_info.resident_size` because it is simpler to obtain.

**Why it's wrong:** `resident_size` does not match what Xcode's Debug Navigator shows and diverges significantly from Instruments. Users who cross-reference the app's reading against Xcode will see different numbers and distrust the app.

**Do this instead:** Use `task_vm_info_data_t.phys_footprint` via `task_info(mach_task_self_, TASK_VM_INFO, ...)`. It matches Xcode's memory gauge exactly.

### Anti-Pattern 5: TabView for Multi-Metric Dashboard

**What people do:** Put each metric in a TabView tab so the screen does not feel crowded.

**Why it's wrong:** A health dashboard's value is seeing all signals simultaneously. A tab forces the user to swipe to find the metric they care about — exactly when they are anxious about device health. ScrollView preserves the simultaneous-glance model.

**Do this instead:** Vertical `ScrollView` with compact metric cards. Each card shows the current value prominently and a mini chart below it. The thermal badge stays at the top as the primary signal.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Mach kernel (`host_statistics`) | Direct C call via Darwin import | No bridging header needed — Darwin module auto-imported in Swift |
| Mach kernel (`task_info`) | Direct C call via Darwin import | Same — `mach_task_self_` macro available |
| `UIDevice` (battery) | Synchronous property read on `@MainActor` | Must enable `isBatteryMonitoringEnabled` before first read |
| `ProcessInfo` (thermal) | Unchanged from v1.1 | |
| `UNUserNotificationCenter` (alerts) | Unchanged from v1.1 | |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| `SystemMetrics.swift` → `TemperatureViewModel` | Direct function call (nonisolated → @MainActor, no crossing) | Return plain value types (`Double?`, `UInt64?`) |
| `TemperatureViewModel` → `ContentView` | `@Observable` auto-tracking | View reads properties; no explicit binding needed |
| `ContentView` → Card subviews | Value-type parameters (not the ViewModel) | Keeps subviews previewable and self-contained |

---

## Build Order for v1.2

```
1. SystemReading.swift (new value type)
   → No dependencies. Unblocks CPU and memory history arrays.
   → ~10 lines. Verifiable in isolation.

2. SystemMetrics.swift (nonisolated Mach helpers)
   → Depends on: Darwin import (built-in, no changes needed).
   → Implement readCPUUsage() and readMemoryFootprint().
   → Test: call both from a temp debug print in startPolling() before wiring to history.
   → First CPU reading will be inaccurate (delta vs zeroed baseline); discard by calling once in init().

3. TemperatureViewModel — new properties and updateAllMetrics()
   → Depends on: SystemReading.swift, SystemMetrics.swift.
   → Add cpuUsage, cpuHistory, memoryUsedBytes, memoryHistory, batteryLevel, batteryState.
   → Enable UIDevice.current.isBatteryMonitoringEnabled = true in startPolling().
   → Rename/expand updateThermalState() → updateAllMetrics().
   → Simulator will show CPU and memory (they work in simulator). Battery requires device.

4. ContentView — ScrollView refactor + new metric cards
   → Depends on: TemperatureViewModel step above.
   → Replace VStack root with ScrollView { VStack }.
   → Add CPUCard, MemoryCard, BatteryCard as private subviews.
   → Thermal card is the existing badge + chart, extracted into a ThermalCard subview.
   → Test on simulator first; no device required until battery display.

5. Device verification
   → CPU and memory: verify values are plausible (cpu 0–100%, memory matches Xcode gauge).
   → Battery: verify level and state update when plugged/unplugged.
   → All notification and background behavior: regression test unchanged paths.
```

**Rationale for this order:** Steps 1–2 are pure additions with no risk to existing behavior. Step 3 expands the ViewModel — the rename of `updateThermalState()` is the highest-risk edit (one call site in `startPolling()`). Step 4 refactors the View — the ScrollView change is structural but ContentView has no unit tests, so visual inspection is the gate. Step 5 is device-gated but non-blocking for steps 1–4.

---

## Sources

- `TemperatureViewModel.swift` (existing, read 2026-05-14) — `@Observable @MainActor` pattern, Timer.publish on .main, existing ring buffer mechanics
- `ContentView.swift` (existing, read 2026-05-14) — VStack root structure, scenePhase hook, chart subview
- [SystemKit/System.swift — beltex/SystemKit (GitHub)](https://github.com/beltex/SystemKit/blob/master/SystemKit/System.swift) — `static let machHost = mach_host_self()` one-time init pattern; delta tick CPU calculation; MEDIUM confidence (community library, widely referenced)
- [SystemEye/CPU.swift — zixun/SystemEye (GitHub)](https://github.com/zixun/SystemEye/blob/master/SystemEye/Classes/CPU.swift) — `host_statistics` with `withUnsafeMutablePointer`/`withMemoryRebound` pattern; MEDIUM confidence
- [phys_footprint — Apple Developer Documentation](https://developer.apple.com/documentation/kernel/task_vm_info_data_t/1553210-phys_footprint) — physical footprint definition; HIGH confidence
- [Apple Developer Forums: how XCode calculates Memory](https://developer.apple.com/forums/thread/105088) — `phys_footprint` vs `resident_size` divergence; MEDIUM confidence
- [Apple Developer Forums: how to get iOS app specific heap memory usage](https://forums.developer.apple.com/thread/119906) — `TASK_VM_INFO` count calculation via MemoryLayout; MEDIUM confidence
- [Mach Port Leakage — Apple Developer Forums](https://developer.apple.com/forums/thread/110688) — port allocation per `mach_host_self()` call; MEDIUM confidence
- [Adopting strict concurrency in Swift 6 — Apple Developer Documentation](https://developer.apple.com/documentation/swift/adoptingswift6) — nonisolated, C global import rules; HIGH confidence
- [batteryState — Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uidevice/batterystate-swift.property) — requires `isBatteryMonitoringEnabled = true`; HIGH confidence
- [Exploring concurrency changes in Swift 6.2 — Donny Wals](https://www.donnywals.com/exploring-concurrency-changes-in-swift-6-2/) — approachable concurrency, MainActor defaults; HIGH confidence

---
*Architecture research for: CoreWatch v1.2 — system metrics integration*
*Researched: 2026-05-14*

---
---

# Distribution Architecture: SideStore/AltStore Source (v1.5)

**Researched:** 2026-05-16
**Confidence:** HIGH (source.json schema, hosting mechanics), MEDIUM (SideStore refresh polling interval — not officially documented)

---

## What This Section Covers

How to package CoreWatch as a SideStore source so a user (the developer, in this case) can install and auto-refresh the app without touching Xcode. The three concrete decisions this section answers:

1. Where does `source.json` live in the repo?
2. GitHub raw URL or GitHub Pages?
3. What is the end-to-end data flow from Xcode archive to SideStore update?

---

## System Overview

```
GitHub repo (CoreWatch)
  ├── source.json              ← app manifest (root of repo, tracked in git)
  └── Releases (GitHub)
        └── CoreWatch-v1.5.ipa ← IPA attached to each GitHub Release tag

SideStore (on device)
  └── polls source.json URL
        → reads versions[0].downloadURL → points to GitHub Release IPA
        → compares version string against installed CFBundleShortVersionString
        → shows "Update" button when version differs
        → user taps Update → SideStore downloads IPA, re-signs, installs
```

---

## Decision 1: Where Does source.json Live?

**Decision: repo root (`/source.json`), tracked in git.**

Rationale:

- The overwhelming community convention for AltStore/SideStore sources is root placement. Browsing the `altstore-source` GitHub topic confirms this is the de facto standard across dozens of public sources.
- Root placement means the raw URL is the shortest possible: `https://raw.githubusercontent.com/naujgs/CoreWatch/main/source.json`. Easy to share, easy to type into SideStore.
- Subdirectories like `docs/` or `.github/` add no benefit and require a longer URL. `docs/` is for GitHub Pages deployment (not needed here). `.github/` is for workflow and issue template files — putting app distribution metadata there is semantically wrong.
- **Do not** use a `docs/` subdirectory unless you specifically want GitHub Pages to serve the file (see Decision 2 for why you do not need that here).

**Repo structure change for v1.5:**

```
CoreWatch/                         (Xcode project folder — unchanged)
source.json                        (NEW — root of repo, same level as CLAUDE.md)
.github/
  workflows/
    release.yml                    (NEW — optional automation, see build order)
```

---

## Decision 2: GitHub Raw URL vs. GitHub Pages

**Decision: raw.githubusercontent.com URL. GitHub Pages is not needed.**

### Raw URL

Format: `https://raw.githubusercontent.com/naujgs/CoreWatch/main/source.json`

- Serves the exact file content as committed to `main`.
- No setup required — works the moment the file is committed.
- No rate limiting concern for a personal single-user source. Rate limits only matter for high-traffic public sources serving thousands of users per hour.
- Content-Type is `text/plain` — SideStore parses the JSON body regardless of Content-Type header, so this does not cause problems.
- Update latency: SideStore fetches fresh each time it checks (no CDN caching layer between the client and GitHub's raw file server for raw.githubusercontent.com URLs). File reflects the commit within seconds of push.

### GitHub Pages

- Requires enabling Pages in repo settings and either using a `docs/` folder or a `gh-pages` branch.
- Serves with `Content-Type: application/json` if the file extension is `.json` — but this is not a meaningful advantage for SideStore.
- Adds a CDN caching layer (GitHub Pages uses Fastly). Cache invalidation after a push can take 1–10 minutes. For a personal source this is a minor nuisance (stale source briefly visible) with no upside.
- Meaningful only for sources with custom domains that need SSL termination or caching for scale. Neither applies here.

**Conclusion:** Raw URL is simpler, immediate, and zero-config. Use it.

---

## Decision 3: The End-to-End Release Workflow

### Manual Workflow (recommended for v1.5 — no CI/CD needed)

This is a personal sideloaded app. Releases are infrequent. A simple manual process is correct; automation would be over-engineering.

```
Step 1 — Xcode: Archive the app
  Product → Archive
  (Organizer window opens)

Step 2 — Xcode: Export IPA
  Organizer → Distribute App
  → "Development" distribution (not App Store, not TestFlight)
  → Select your device / team
  → Export → saves CoreWatch.ipa to a local folder

Step 3 — GitHub: Create a Release
  gh release create v1.5 CoreWatch.ipa \
    --title "CoreWatch v1.5" \
    --notes "SideStore distribution support"
  (or use GitHub web UI: Releases → Draft new release → attach IPA)

  Result: IPA is publicly downloadable at a stable URL:
  https://github.com/naujgs/CoreWatch/releases/download/v1.5/CoreWatch.ipa

Step 4 — Repo: Update source.json
  Edit source.json:
    - Bump "version" to match CFBundleShortVersionString (e.g., "1.5")
    - Set "downloadURL" to the GitHub Release IPA URL from Step 3
    - Set "date" to today (YYYY-MM-DD)
    - Set "size" to IPA file size in bytes (get via: stat -f%z CoreWatch.ipa)
  Commit and push to main.

Step 5 — SideStore: Picks up update
  Next time SideStore refreshes its sources, it fetches source.json,
  sees a new version string, and shows an Update button.
  User taps Update → SideStore downloads IPA → re-signs → installs.
```

### Why "Development" Export, Not "Ad Hoc"

Free Apple ID accounts cannot create Ad Hoc provisioning profiles — that requires a paid Developer account ($99/yr). Development export is what free Apple ID sideloading uses. SideStore re-signs the IPA on-device using its own signing mechanism, so the original signing method in the export is irrelevant — what matters is that the IPA is not encrypted (App Store IPAs are encrypted and cannot be re-signed).

### IPA Size for source.json

The `size` field is required and must be the IPA byte count. Get it:

```bash
stat -f%z CoreWatch.ipa   # macOS
```

SideStore displays this to the user before download. Wrong values cause user confusion but not a functional failure.

---

## source.json Schema

Minimal valid source.json for CoreWatch. Both AltStore and SideStore consume this format identically (HIGH confidence — confirmed against official AltStore docs and SideStore type definitions).

```json
{
  "name": "CoreWatch",
  "identifier": "com.jgs.CoreWatch.source",
  "apps": [
    {
      "name": "CoreWatch",
      "bundleIdentifier": "com.jgs.CoreWatch",
      "developerName": "jgs",
      "localizedDescription": "iPhone health dashboard — thermal state, CPU, and memory at a glance with overheat alerts.",
      "iconURL": "https://raw.githubusercontent.com/naujgs/CoreWatch/main/assets/icon.png",
      "tintColor": "#FF6B35",
      "versions": [
        {
          "version": "1.5",
          "date": "2026-05-16",
          "downloadURL": "https://github.com/naujgs/CoreWatch/releases/download/v1.5/CoreWatch.ipa",
          "size": 0,
          "localizedDescription": "SideStore distribution. Auto-refresh replaces weekly Xcode reinstall."
        }
      ]
    }
  ],
  "news": []
}
```

**Critical field notes:**

| Field | Requirement | Notes |
|-------|-------------|-------|
| `bundleIdentifier` (app) | Must exactly match `CFBundleIdentifier` in Info.plist | Case-sensitive. Mismatch causes SideStore to treat installed app and source entry as different apps. |
| `identifier` (source root) | Unique reverse-domain string for the source itself | Different from the app's bundle ID. Used by SideStore to deduplicate sources. |
| `version` | Must match `CFBundleShortVersionString` | This is the string SideStore compares against the installed version to detect updates. |
| `downloadURL` | Must be a direct download link, not a redirect | GitHub Release asset URLs are direct. Do not use the HTML release page URL. |
| `size` | Integer bytes | Required. Use `stat -f%z`. |
| `date` | `YYYY-MM-DD` | Versions array is ordered newest-first. First entry is what SideStore treats as latest. |
| `iconURL` | Optional but strongly recommended | Can point to any publicly accessible PNG. A raw GitHub URL to a committed asset works. |
| `marketplaceID` | Do NOT include | Including this field causes SideStore to interpret the app as a notarized PAL app and reject the IPA. |

**SideStore-specific compatibility note (MEDIUM confidence — from SideStore GitHub issue #735):**

SideStore expects `downloadURL` at the version object level (inside the `versions` array). This is identical to how AltStore's current schema works. The older AltStore v1 schema had `downloadURL` at the app top-level — do not use that format. The schema above is correct for both clients.

---

## How SideStore Detects Updates

**Mechanism:** Version string comparison. SideStore fetches `source.json`, reads `versions[0].version`, compares it against the `CFBundleShortVersionString` of the installed app. If they differ, it shows an Update badge. (HIGH confidence — confirmed by AltStore official docs and the `sidestore-source-types` Version interface definition.)

**What triggers a check:** SideStore checks sources when the user opens the app (on foreground) and periodically in the background via iOS background app refresh. The background interval is at Apple's discretion (BGAppRefreshTask) — typically every few hours. There is no documented fixed polling interval. For a personal source, this is irrelevant: the developer is both publisher and sole user, so a manual "Refresh All" in SideStore is sufficient to trigger an immediate check.

**What SideStore does NOT do:** Watch for GitHub webhook events, compare file ETags, or respond to push notifications from the source host. It is pure pull — it fetches the JSON and checks the version string.

**Caching note (MEDIUM confidence — from SideStore GitHub issue #975 and community reports):** SideStore caches source data locally. If an update is not detected after pushing a new `source.json`, removing and re-adding the source clears the cache and forces a fresh fetch. This is the documented workaround for stale-cache update detection failures.

**Update detection gotcha:** SideStore compares version strings as strings, not as semantic version numbers. `"1.10"` and `"1.9"` compare as strings, not numbers. Use simple monotonically increasing version numbers (e.g., `"1.5"`, `"1.6"`) rather than complex semver strings to avoid unexpected ordering.

---

## Data Flow Diagram

```
Developer machine                    GitHub                      Device (SideStore)
─────────────────                    ──────                      ─────────────────

1. Xcode Archive
   → Product → Archive
   → Export IPA (Development)
   → CoreWatch.ipa (local)

2. gh release create v1.5
   CoreWatch.ipa                →   Release asset created
                                    URL: /releases/download/
                                         v1.5/CoreWatch.ipa

3. Edit source.json
   version: "1.5"
   downloadURL: (release URL)
   size: (bytes)
   git commit + push            →   source.json updated
                                    raw URL reflects new content
                                    within seconds of push

                                                            4. SideStore refresh
                                                               GET raw.githubusercontent.com/
                                                               naujgs/CoreWatch/main/source.json
                                                               ← JSON response

                                                            5. Version comparison
                                                               source version "1.5"
                                                               vs installed "1.4"
                                                               → mismatch → Update badge shown

                                                            6. User taps Update
                                                               GET /releases/download/
                                                                   v1.5/CoreWatch.ipa
                                                               ← IPA download

                                                            7. SideStore re-signs IPA
                                                               using its own certificate
                                                               → installs over existing app
                                                               → 7-day signing clock reset
```

---

## Build Order for v1.5

The ordering constraint is strict: the GitHub Release IPA URL must exist before `source.json` can reference it. `source.json` must be committed before SideStore can detect the update.

```
1. [GATE] IPA must be built and exported from Xcode before anything else.
   → Archive via Product → Archive
   → Export as Development IPA
   → Note the IPA file size (stat -f%z)
   → No source.json changes yet — the download URL doesn't exist yet.

2. [GATE] GitHub Release must be created and IPA attached before source.json update.
   → Create the release tag (e.g., v1.5)
   → Attach the IPA as a release asset
   → Confirm the asset download URL is live and accessible
   → This URL is what source.json will reference.

3. source.json — create or update
   → If first release: create source.json in repo root with the schema above
   → If subsequent release: update versions[0] with new version, date, downloadURL, size
   → Keep older versions in the versions array (prepend new entry at index 0)
   → Commit and push to main

4. Verify
   → Open raw URL in browser: confirm JSON is valid and reflects new version
   → In SideStore: add source (first time) or tap Refresh All
   → Confirm Update badge appears
   → Tap Update, confirm install succeeds
   → Confirm app version in Settings → General → iPhone Storage matches new version
```

**Why this order is strict:**

- If you commit `source.json` before the GitHub Release exists, SideStore will fetch a `downloadURL` that returns 404. SideStore will cache this failure. Users will see a broken update.
- If you create the Release before exporting the IPA, you have nothing to attach. The Release can be created as a draft, then published after the IPA is attached.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Including marketplaceID in source.json

**What goes wrong:** SideStore interprets any source entry with `marketplaceID` (or `buildVersion` alongside it) as a notarized Apple PAL app. It will reject the IPA with a signing error.

**Prevention:** Never include `marketplaceID` or `buildVersion` in the source.json for a sideloaded Development IPA. AltStudio auto-generates these fields — remove them before use.

### Anti-Pattern 2: Using the GitHub Release HTML Page URL as downloadURL

**What goes wrong:** The HTML release page (`github.com/user/repo/releases/tag/v1.5`) is not an IPA download — it's a web page. SideStore will download HTML and fail to parse it as an IPA.

**Prevention:** Use the release asset download URL format: `https://github.com/naujgs/CoreWatch/releases/download/v1.5/CoreWatch.ipa`. Get this URL from the release asset's "Copy link" in the GitHub web UI or from `gh release view --json assets`.

### Anti-Pattern 3: Committing source.json Before the GitHub Release Exists

**What goes wrong:** SideStore fetches the source, gets the version string, tries to download the IPA, gets a 404. It may cache this failure. The update will appear broken until the cache clears (remove/re-add source).

**Prevention:** Follow the build order strictly: IPA built → Release created and asset attached → source.json updated and pushed.

### Anti-Pattern 4: Putting source.json in .github/ or docs/

**What goes wrong:** `.github/` is semantically for GitHub workflow files, issue templates, and pull request templates — not app distribution metadata. `docs/` triggers GitHub Pages expectations. Neither adds value for a raw URL serving scenario.

**Prevention:** Keep source.json at repo root. The raw URL is shorter and the placement is semantically correct.

### Anti-Pattern 5: Omitting the size Field

**What goes wrong:** `size` is required by the schema. SideStore may display "0 bytes" or reject the entry depending on version. Either causes a confusing UX.

**Prevention:** Always run `stat -f%z CoreWatch.ipa` after exporting and set the integer byte count in source.json.

---

## Component Boundaries: Repo Structure After v1.5

```
CoreWatch/                         ← Xcode project (unchanged)
  CoreWatch.xcodeproj/
  ...Swift source files...

source.json                        ← NEW: app manifest, root of repo
                                      Updated manually each release

.planning/                         ← GSD planning artifacts (unchanged)
README.md                          ← Existing (can add source URL here)
```

No new Swift files, no new Xcode targets, no new build phases. The v1.5 work is entirely outside the Xcode project — it is repo and GitHub infrastructure only.

---

## Sources (v1.5 section)

- [Make a Source — AltStore official docs](https://faq.altstore.io/developers/make-a-source) — authoritative schema reference; HIGH confidence
- [App Sources — SideStore Docs](https://docs.sidestore.io/docs/advanced/app-sources) — confirms AltStore format compatibility; HIGH confidence
- [sidestore-source-types Version interface](https://sidestore.io/sidestore-source-types/interfaces/Version.html) — required/optional fields, version comparison semantics; HIGH confidence
- [SideStore GitHub issue #735](https://github.com/SideStore/SideStore/issues/735) — downloadURL field placement, marketplaceID rejection; MEDIUM confidence (community issue report, corroborated by schema docs)
- [SideStore GitHub issue #975](https://github.com/SideStore/SideStore/issues/975) — source cache behavior, stale-cache workaround; MEDIUM confidence (community issue report)
- [SideSource GitHubInput interface](https://sidestore.io/SideSource/interfaces/GitHubInput.html) — GitHubInput schema (not used for manual workflow, documented for awareness); MEDIUM confidence
- [GitHub raw.githubusercontent.com vs GitHub Pages hosting analysis](https://sidestore.io/SideSource/) — raw URL recommended for non-cached personal sources; MEDIUM confidence
- GitHub `altstore-source` topic (community convention survey) — root placement is de facto standard; MEDIUM confidence

---
*Distribution architecture for: CoreWatch v1.5 — SideStore source*
*Researched: 2026-05-16*
