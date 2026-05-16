# Technology Stack

**Project:** CoreWatch
**Researched:** 2026-05-14 (v1.2 update — CPU usage, memory, battery APIs for iOS 18 free sideload)
**Mode:** Ecosystem / Feasibility

---

## Established Stack (v1.1 — Do Not Change)

| Technology | Version | Purpose | Status |
|------------|---------|---------|--------|
| Xcode | 26.4.1 (stable) | IDE, compiler, device install | Confirmed — do not upgrade to 26.5 beta |
| Swift | 6.3 (ships with Xcode 26.4.1) | Language | Confirmed — strict concurrency on, `@MainActor` on ViewModel |
| iOS SDK target | iOS 18.x (min deployment) | Runtime | Confirmed |
| SwiftUI | bundled | UI framework | Confirmed |
| Swift Charts | bundled (iOS 16+) | Session-length history chart | Confirmed |
| Foundation | bundled | `ProcessInfo.thermalState`, timers | Confirmed |
| UserNotifications | bundled | Local threshold alerts | Confirmed |
| IOKit | bundled (via bridging header) | Private API access point | Header exists; call blocked under free Apple ID |

No external dependencies. No SPM, no CocoaPods, no Carthage.

---

## v1.2 New Data Sources — Verdict by API

This section answers: what is accessible, what is blocked, and why, under **free Apple ID sideload on iOS 18**.

The constraint that governs all answers: AMFI enforces the sandbox profile. Standard free Apple ID sideloading gives apps the standard `container.sb` sandbox with the entitlements Apple's provisioning portal grants to free-tier development certificates. Any API requiring a private Apple entitlement (`com.apple.*` or `systemgroup.*`) is blocked. Public SDK APIs require no special entitlements and work normally.

---

### API 1: `UIDevice.current.batteryLevel` / `batteryState`

**Verdict: ACCESSIBLE**
**Confidence: HIGH**

`UIDevice.batteryLevel` and `UIDevice.batteryState` are public UIKit APIs. They require no entitlements. They are available to all iOS apps including App Store apps, meaning they are unambiguously sandbox-safe. The only activation requirement is setting `UIDevice.current.isBatteryMonitoringEnabled = true` before reading; without it, `batteryLevel` returns `-1.0` and `batteryState` returns `.unknown`.

**Why it works under free sideload:** These properties are in the public iOS SDK. Free sideloading gives the app the same public API surface as an App Store app. No private entitlement gates this API.

**What it provides:**
- `batteryLevel`: Float from `0.0` (empty) to `1.0` (full). Returns `-1.0` if monitoring not enabled.
- `batteryState`: Enum — `.unknown`, `.unplugged`, `.charging`, `.full`

**What it does NOT provide:** Battery health percentage, cycle count, temperature, voltage, current draw. Those fields are IOKit private territory, blocked by AMFI.

**Implementation pattern:**
```swift
// In init or onAppear — enable once
UIDevice.current.isBatteryMonitoringEnabled = true

// Read in polling loop
let level = UIDevice.current.batteryLevel   // e.g. 0.83 → "83%"
let state = UIDevice.current.batteryState   // .charging, .unplugged, etc.
```

**Notification-based updates (optional):**
```swift
NotificationCenter.default.addObserver(forName: UIDevice.batteryLevelDidChangeNotification, ...)
NotificationCenter.default.addObserver(forName: UIDevice.batteryStateDidChangeNotification, ...)
```
Both require `isBatteryMonitoringEnabled = true`.

---

### API 2: `mach_task_basic_info` / `task_vm_info` — App's own memory footprint

**Verdict: ACCESSIBLE (own process only)**
**Confidence: HIGH**

`task_info(mach_task_self_, ...)` is callable for an app's own process without any special entitlement. The key distinction: `mach_task_self_` gives the app a port to its own task — no `task_for_pid()` is needed, which is the sandboxed operation. Apple's own documentation and WWDC sessions confirm that `phys_footprint` from `TASK_VM_INFO` is how Xcode's memory gauge measures app memory.

**Why it works under free sideload:** Accessing your own task port is always permitted. `task_for_pid()` on another process's PID is what the sandbox blocks. Using `mach_task_self_` is equivalent to `getpid()` — the kernel grants this by design.

**Two flavors — use `TASK_VM_INFO` for accuracy:**

`MACH_TASK_BASIC_INFO` gives `resident_size`, which is less accurate because it includes shared memory pages that the system can reclaim. Apple's preferred metric is `phys_footprint` from `TASK_VM_INFO`, which is what Xcode's Memory Report and Instruments use. It matches what counts against the app's memory budget.

**Implementation pattern:**
```swift
import Darwin

func appMemoryFootprint() -> UInt64? {
    var info = task_vm_info_data_t()
    // TASK_VM_INFO_COUNT is too complex for Swift C importer — compute manually
    var count = mach_msg_type_number_t(
        MemoryLayout<task_vm_info_data_t>.size / MemoryLayout<integer_t>.size
    )
    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: Int(count)) {
            task_info(mach_task_self_, task_flavor_t(TASK_VM_INFO), $0, &count)
        }
    }
    guard result == KERN_SUCCESS else { return nil }
    return info.phys_footprint  // bytes; divide by 1_048_576 for MB
}
```

**Also available — total device RAM:**
```swift
ProcessInfo.processInfo.physicalMemory  // UInt64, total RAM in bytes
```
`ProcessInfo.physicalMemory` is a public Foundation API, no entitlement needed.

**What this provides:** The app's own memory footprint in bytes. Not system-wide memory pressure — just this app's physical memory usage.

---

### API 3: `host_cpu_load_info` / `host_statistics` — System-wide CPU

**Verdict: UNCERTAIN — Probably inaccessible from sandbox, use thread-level alternative**
**Confidence: MEDIUM (LOW for sandbox accessibility claim)**

`host_statistics(mach_host_self(), HOST_CPU_LOAD_INFO, ...)` reads system-wide CPU tick counters across all processes. Apple's sandbox philosophy explicitly states that apps should only be able to get information about themselves, not the system as a whole. Apple Developer Forum posts from multiple engineers note that "it's not hard to imagine Mach host APIs running afoul of the sandbox at some point in the future" — suggesting the intent is to restrict these, even if enforcement is inconsistent across iOS versions.

**Risk:** There is no definitive Apple documentation confirming `host_statistics` is whitelisted in `container.sb`. MacOS SystemKit (which uses this API) is macOS-targeted and works in a less restricted environment. Relying on `host_cpu_load_info` for a sideloaded iOS app creates fragility risk — it may silently fail or be tightened in future iOS point releases.

**The safer alternative: thread-level CPU for this process only**

`task_threads(mach_task_self_, ...)` + `thread_info(..., THREAD_BASIC_INFO, ...)` reads CPU usage for all threads in your own process. This is the same mechanism that Xcode's CPU gauge uses to show per-process CPU %. It gives CPU % consumed by the CoreWatch app, not total device CPU — which is arguably more relevant for a self-monitoring tool.

**Implementation pattern (own-process CPU %):**
```swift
import Darwin

func appCPUUsage() -> Double {
    var threadList: thread_act_array_t?
    var threadCount: mach_msg_type_number_t = 0
    guard task_threads(mach_task_self_, &threadList, &threadCount) == KERN_SUCCESS,
          let threads = threadList else { return 0 }
    defer {
        vm_deallocate(
            mach_task_self_,
            vm_address_t(bitPattern: threads),
            vm_size_t(Int(threadCount) * MemoryLayout<thread_t>.stride)
        )
    }

    var totalUsage: Double = 0
    for i in 0..<Int(threadCount) {
        var info = thread_basic_info()
        var infoCount = mach_msg_type_number_t(THREAD_BASIC_INFO_COUNT)
        let result = withUnsafeMutablePointer(to: &info) {
            $0.withMemoryRebound(to: integer_t.self, capacity: Int(infoCount)) {
                thread_info(threads[i], thread_flavor_t(THREAD_BASIC_INFO), $0, &infoCount)
            }
        }
        if result == KERN_SUCCESS {
            let threadInfo = info
            if threadInfo.flags & TH_FLAGS_IDLE == 0 {
                totalUsage += Double(threadInfo.cpu_usage) / Double(TH_USAGE_SCALE)
            }
        }
    }
    return totalUsage * 100  // percentage; can exceed 100% on multi-core devices
}
```

**If system-wide CPU is required:** The app should attempt `host_statistics` and gracefully handle failure (return `nil`). If the call fails on the target iOS 18 device, fall back to own-process CPU or omit the metric. Do not assume it will succeed.

---

### API 4: `sysctl` — Device info and memory

**Verdict: PARTIALLY ACCESSIBLE**
**Confidence: HIGH (for listed keys), MEDIUM (for process-list keys)**

`sysctl` is a broad interface — some keys are accessible from sandbox, others are blocked. Apple's sandbox blocks process enumeration (`CTL_KERN, KERN_PROC`) as of iOS 9. However, hardware info keys work fine.

**Accessible from sandbox (confirmed by multiple sources):**

| Key | Value | Notes |
|-----|-------|-------|
| `hw.physmem` / `hw.memsize` | Total device RAM in bytes | Same as `ProcessInfo.physicalMemory` |
| `hw.ncpu` | CPU core count | Logical cores |
| `hw.machine` | Device model string (e.g. `iPhone14,2`) | |
| `kern.osversion` | iOS build number | |

**Blocked from sandbox:**
- `CTL_KERN, KERN_PROC` — process listing — explicitly blocked since iOS 9
- Process-specific info for other PIDs

**Bottom line for v1.2:** `ProcessInfo.processInfo.physicalMemory` (Foundation) and `ProcessInfo.processInfo.processorCount` (Foundation) already surface the most useful `sysctl` values without the `sysctl` ceremony. Use Foundation APIs instead.

---

### API 5: `IOPSCopyPowerSourcesInfo` — Battery data via IOKit

**Verdict: BLOCKED**
**Confidence: HIGH**

`IOPSCopyPowerSourcesInfo` is part of IOKit's power source subsystem. On iOS, IOKit was added to the public SDK in iOS 16, but its scope is limited: the public iOS IOKit API exists solely to support apps that contain DriverKit extensions (system-level driver code), not for general power source querying. The `IOPowerSources` functions (`IOPSCopyPowerSourcesInfo`, `IOPSCopyPowerSourcesList`) are macOS APIs. Apple removed battery data access via IOKit from iOS in iOS 10 (confirmed via MacRumors developer forum discussion from that period). Under free sideload, there is no entitlement path to access these functions even if headers can be imported.

**What to use instead:** `UIDevice.current.batteryLevel` + `batteryState` (API 1, above).

---

### API 6: `MTLDevice.currentAllocatedSize` — GPU memory usage

**Verdict: ACCESSIBLE but scope is GPU memory only, not GPU utilization %**
**Confidence: MEDIUM**

`MTLDevice.currentAllocatedSize` is a public Metal API property that reports the total bytes of GPU memory currently allocated by all Metal resources created by the app (buffers, textures, heaps). It is available on iOS and does not require entitlements. However:

- It measures **GPU memory allocated by this app's Metal objects**, not GPU compute utilization %.
- For CoreWatch (a SwiftUI dashboard with no Metal rendering), this value will be near-zero and meaningless — the app creates no Metal resources.
- There is no public iOS API to read GPU utilization % at runtime from an arbitrary app. GPU profiling (via Metal Performance Counters and Instruments) is a development-time tool, not a runtime API.

**Conclusion for v1.2:** GPU utilization % is not obtainable via public APIs. `MTLDevice.currentAllocatedSize` is technically accessible but useless for CoreWatch's use case. Do not implement.

---

## v1.2 Recommended New APIs — Summary

| API | Framework | Accessible? | Entitlement Required? | What It Provides |
|-----|-----------|-------------|----------------------|-----------------|
| `UIDevice.batteryLevel` | UIKit | YES | None | Battery % (0.0–1.0) |
| `UIDevice.batteryState` | UIKit | YES | None | Charging/Unplugged/Full |
| `task_info(mach_task_self_, TASK_VM_INFO)` | Darwin/Mach | YES | None | App memory footprint (bytes) |
| `ProcessInfo.physicalMemory` | Foundation | YES | None | Total device RAM (bytes) |
| `task_threads` + `thread_info(THREAD_BASIC_INFO)` | Darwin/Mach | YES (own process) | None | App CPU usage % |
| `host_statistics(HOST_CPU_LOAD_INFO)` | Darwin/Mach | UNCERTAIN | Likely required | System-wide CPU % |
| `IOPSCopyPowerSourcesInfo` | IOKit | NO | Private, blocked | (Battery detail — macOS only) |
| `MTLDevice.currentAllocatedSize` | Metal | YES | None | GPU memory (useless for this app) |
| GPU utilization % | — | NO | N/A | No public API exists |

---

## v1.2 Stack Additions

No new frameworks are required. The new APIs are in frameworks already in use:

| New API | Framework Already in Stack | Notes |
|---------|---------------------------|-------|
| `UIDevice.batteryLevel/batteryState` | UIKit (already imported by SwiftUI) | Needs `isBatteryMonitoringEnabled = true` in init |
| `task_info(TASK_VM_INFO)` | Darwin (available via Foundation import) | Swift C importer limitation for `TASK_VM_INFO_COUNT` — compute count manually |
| `task_threads` + `thread_info(THREAD_BASIC_INFO)` | Darwin | Requires `vm_deallocate` for thread list cleanup |
| `ProcessInfo.physicalMemory` | Foundation | Already imported; zero changes needed |

**No new imports, no new frameworks, no new dependencies.**

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `host_statistics(HOST_CPU_LOAD_INFO)` for system-wide CPU | Likely blocked by iOS sandbox; fragile across iOS versions; philosophy violation | `task_threads` + `thread_info` for own-process CPU |
| `IOPSCopyPowerSourcesInfo` | macOS API; removed from iOS since iOS 10; blocked by AMFI under free sideload | `UIDevice.batteryLevel` + `batteryState` |
| `task_for_pid()` on any PID ≠ own process | Explicitly blocked in iOS sandbox for sandboxed apps | Not applicable — only need own-process data |
| `MTLDevice.currentAllocatedSize` | GPU memory only; near-zero for a non-Metal UI app; not GPU utilization % | N/A — omit GPU entirely |
| Any `IOPMPowerSource` IOKit key | Requires `systemgroup.com.apple.powerlog` private entitlement; AMFI blocks under free sideload | `UIDevice.batteryState` for charging state |
| `BGAppRefreshTask` / `BGProcessingTask` for periodic sensor reads | System-discretionary scheduling; unsuitable for real-time monitoring | Timer-based polling while foregrounded |

---

## Updated Dependency Surface (v1.2)

```
Xcode 26.4.1 (from Mac App Store)
  └── Swift 6.3 (bundled)
  └── iOS 26 SDK (bundled — target iOS 18.x minimum deployment)
  └── SwiftUI (bundled)
  └── Swift Charts (bundled in iOS 16+ SDK)
  └── Foundation (bundled) — ProcessInfo.thermalState, physicalMemory, timers
  └── UIKit (bundled, via SwiftUI) — UIDevice.batteryLevel, batteryState
  └── Darwin / libSystem (bundled) — task_info, task_threads, thread_info (Mach APIs)
  └── UserNotifications (bundled)
  └── IOKit (bundled via bridging header — entitlement-gated, no new uses in v1.2)

Dev tooling: no change from v1.1
```

No new runtime frameworks. No SPM, no CocoaPods, no Carthage.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| `UIDevice.batteryLevel/batteryState` — accessible under free sideload | HIGH | Public UIKit API; no entitlement required; universally documented; App Store safe |
| `task_info(TASK_VM_INFO)` for app memory footprint — accessible | HIGH | Apple Developer Forums confirm; `phys_footprint` is how Xcode's memory gauge works; own-process access requires no entitlement |
| `task_threads` + `thread_info(THREAD_BASIC_INFO)` for app CPU — accessible | MEDIUM-HIGH | Community implementations confirm; Apple forums confirm `task_for_pid` is blocked but own process is allowed; no definitive Apple doc confirming the exact entitlement boundary |
| `host_statistics(HOST_CPU_LOAD_INFO)` blocked in iOS sandbox | MEDIUM | Apple engineers warn sandbox may restrict this; philosophy is apps see only themselves; no definitive test result found, but risk is documented |
| `IOPSCopyPowerSourcesInfo` blocked on iOS | HIGH | macOS-only API; iOS 10 removed battery IOKit detail access; no available entitlement path under free sideload |
| GPU utilization % — no public API | HIGH | No public API exists; Metal Performance Counters are development-time only |

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| App memory | `task_info(TASK_VM_INFO).phys_footprint` | `MACH_TASK_BASIC_INFO.resident_size` | `resident_size` includes shared pages not charged to app; less accurate than `phys_footprint` |
| App CPU | `task_threads` + `thread_info(THREAD_BASIC_INFO)` | `host_statistics(HOST_CPU_LOAD_INFO)` | `host_statistics` requires system-wide host access likely restricted by sandbox; own-thread approach is sandbox-safe and gives app-specific data |
| Battery level | `UIDevice.batteryLevel` | `IOPSCopyPowerSourcesInfo` | IOKit power sources API is macOS-only; blocked by AMFI on iOS under free sideload |
| Total RAM | `ProcessInfo.physicalMemory` | `sysctl hw.memsize` | Foundation property is simpler and already imported; identical data |

---

## Sources

- [UIDevice.batteryLevel — Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uidevice/batterylevel) — public API, no entitlement documented
- [UIDevice.BatteryState — Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uidevice/batterystate) — public API
- [how to overall cpu utilization of iphone device — Apple Developer Forums (thread/11393)](https://developer.apple.com/forums/thread/11393) — confirms sandbox philosophy: apps see themselves only; host APIs may be restricted
- [Obtaining CPU usage by process — Apple Developer Forums (thread/655349)](https://developer.apple.com/forums/thread/655349) — `task_for_pid` blocked; own-process `task_threads` approach documented
- [how to get iOS app specific heap memory usage — Apple Developer Forums (thread/119906)](https://developer.apple.com/forums/thread/119906) — `task_vm_info` with `phys_footprint` confirmed working, no entitlement noted
- [Swift 4 iOS app memory usage gist (pejalo)](https://gist.github.com/pejalo/671dd2f67e3877b18c38c749742350ca) — `MACH_TASK_BASIC_INFO` implementation without special entitlements
- [Battery level with IOPSCopyPowerSourcesInfo — Apple Developer Forums (thread/712711)](https://developer.apple.com/forums/thread/712711) — forum thread confirming IOPSCopyPowerSourcesInfo is not the iOS path
- [Battery data gone in iOS 10 — MacRumors Forums](https://forums.macrumors.com/threads/battery-data-cycles-health-etc-gone-in-ios-10.1977954/) — confirms Apple removed detailed battery data via IOKit in iOS 10
- [MTLDevice.currentAllocatedSize — Apple Developer Documentation](https://developer.apple.com/documentation/metal/mtldevice/currentallocatedsize) — public Metal API, no entitlement; tracks GPU memory not CPU utilization
- [Reading iOS Sandbox Profiles — 8kSec](https://8ksec.io/reading-ios-sandbox-profiles/) — confirms `container.sb` governs all App Store/sideloaded apps; entitlements drive permission differences
- [ProcessInfo — Apple Developer Documentation](https://developer.apple.com/documentation/foundation/processinfo) — `physicalMemory`, `processorCount`, `thermalState` all public, no entitlement
- [iOS Sideloading How It Works 2025 — DEV Community](https://dev.to/1_king_0b1e1f8bfe6d1/how-ios-sideloading-actually-works-in-2025-dev-certs-altstore-and-the-eu-exception-1m2h) — confirms free Apple ID sideload entitlement constraints match App Store sandbox

---

## Appendix: v1.1 Research (App Icon, TrollStore, Polling Interval)

Preserved from v1.1 for traceability. See git history for full v1.1 content.

Key v1.1 decisions still in force:
- IOKit `IOPMPowerSource` Temperature: blocked under free Apple ID, TrollStore path requires iOS ≤17.0 (device on iOS 18 — permanently blocked)
- App icon: single-size 1024×1024 opaque RGB PNG, Xcode 26 handles resizing
- Polling interval: 10s `Timer.publish`
- `ldid` for TrollStore build path — not needed for v1.2

---
*Stack research for: CoreWatch v1.2 — iOS system health APIs under free Apple ID sideload*
*Researched: 2026-05-14*

---

## v1.5 Distribution Stack — SideStore/AltStore Source Distribution

**Researched:** 2026-05-16
**Scope:** Everything needed to distribute CoreWatch via SideStore auto-refresh. No CI, no automation — manual Xcode export only.

---

### What SideStore/AltStore Distribution Actually Does

SideStore (and AltStore Classic) re-sign your IPA with the user's own free Apple ID development certificate. They then refresh that signing every 7 days automatically, eliminating the need to re-install via Xcode each week. The app is distributed as a raw IPA file (your build, signed or unsigned) hosted at a public URL. The sideloading tool downloads it, strips the existing signature, and applies a new signature tied to the user's Apple ID and device UDID.

This means:
- The IPA you publish does not need to be unsigned. SideStore/AltStore will re-sign over whatever is in it.
- The bundle identifier in the IPA is what SideStore uses as-is (it does not rename it unless there is a collision with another app using the same App ID slot).
- Entitlements beyond the standard free Apple ID set are stripped and replaced with what the free certificate allows. Any private entitlements (`com.apple.*`, `systemgroup.*`) will not survive re-signing. CoreWatch uses none of these, so this is a non-issue.

**Confidence: HIGH** — Confirmed via SideStore FAQ ("resigns apps with your personal development certificate"), SideStore GitHub issue tracker, and AltStore Classic documentation.

---

### IPA Export — What to Do in Xcode

The correct export path for a sideload-distribution IPA using a free Apple ID is:

1. Set the build scheme destination to "Any iOS Device (arm64)" — not a simulator, not a connected phone.
2. Product > Archive. Wait for Xcode to complete the archive.
3. Xcode Organizer opens automatically. Select the new archive.
4. Click "Distribute App."
5. In the distribution method picker, choose "Release Testing" (Xcode 16+) or "Development" (older Xcode). Both produce a development-signed IPA. "Release Testing" is the Xcode 16+ name for the same path.
6. On the next screen, keep "Automatically manage signing" selected. Xcode will use the Personal Team (free Apple ID) certificate.
7. Click "Export." Save the `.ipa` file.

**Do NOT use "App Store Connect" or "Ad Hoc."** Ad Hoc requires registered device UDIDs in a paid developer account provisioning profile. App Store Connect uploads to Apple. Neither is appropriate here.

**Result:** A `.ipa` file approximately equal to the size of the app bundle. SideStore will accept this IPA and re-sign it.

**Confidence: MEDIUM-HIGH** — "Release Testing" path confirmed via Apple Community thread (thread/255781574) and multiple IPA export guides. The free Apple ID constraint on Ad Hoc is documented in Apple's developer distribution docs.

---

### IPA Signing Compatibility with SideStore

Free Apple ID development-signed IPAs work with SideStore re-signing. SideStore does not require unsigned IPAs. The re-signing process:

1. Downloads the IPA from the `downloadURL` in `source.json`.
2. Strips the existing code signature.
3. Generates a new provisioning profile from the user's Apple ID, tied to their device UDID and the app's bundle identifier.
4. Re-signs the app binary and all embedded frameworks with that profile and the user's development certificate.
5. Installs and registers the app for automatic 7-day refresh.

**Entitlement constraint to know:** Free Apple ID certificates support a maximum of 10 App IDs concurrently (not 3 — the 3-app visual limit in older AltStore is a UI constraint; SideStore lifts it). Each App ID expires and is reclaimed after 7 days. CoreWatch's bundle ID (e.g. `com.yourname.corewatch`) will occupy one slot.

**Bundle ID:** Keep the existing bundle ID as-is. Do not append a team prefix or change the ID for SideStore distribution. SideStore installs with the bundle ID from the IPA's `Info.plist`, which must match `bundleIdentifier` in `source.json`.

**Confidence: HIGH** — Re-signing behavior confirmed via SideStore FAQ and GitHub issue #735 discussion. 10 App ID limit confirmed via SideStore bypass-limit issue #68.

---

### source.json Format — Authoritative Specification

The `source.json` file (also called an AltSource) is a JSON file hosted at a public URL. Both AltStore Classic and SideStore read this format. SideStore is "fully compatible with AltStore Sources."

**Root object — required fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | YES (effectively) | Source display name shown in the app |
| `apps` | array | YES | Array of App objects |
| `news` | array | YES | Can be empty array `[]` |

The AltStore docs list `apps` and `news` as required, with `name` technically optional but always present in practice.

**Root object — useful optional fields:**

| Field | Type | Notes |
|-------|------|-------|
| `identifier` | string | Source-unique identifier (e.g. `com.yourname.corewatch-source`) |
| `sourceURL` | string | Canonical URL of this source.json file |
| `subtitle` | string | One-line description |
| `description` | string | Full description |
| `iconURL` | string | URL to circular icon |
| `tintColor` | string | Hex color, e.g. `#FF6B35` |

**App object — required fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | YES | App display name |
| `bundleIdentifier` | string | YES | Must match `CFBundleIdentifier` in the IPA exactly, case-sensitive |
| `developerName` | string | YES | Your name |
| `localizedDescription` | string | YES | App description |
| `iconURL` | string | YES | URL to app icon (can be raw GitHub URL to a PNG) |
| `versions` | array | YES | Array of Version objects; first entry is treated as latest |
| `appPermissions` | object | YES (AltStore 2.0+) | See below |

**App object — useful optional fields:**

| Field | Type | Notes |
|-------|------|-------|
| `subtitle` | string | One-liner shown in source listing |
| `tintColor` | string | Hex color |
| `category` | string | One of: `utilities`, `developer`, `entertainment`, `games`, `lifestyle`, `other`, `photo-video`, `social` |
| `screenshotURLs` | array of strings | Screenshot image URLs |

**Version object — required fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `version` | string | YES | Must match `CFBundleShortVersionString` in the IPA exactly, case-sensitive (e.g. `"1.5"`) |
| `buildVersion` | string | YES (AltStore spec) | Must match `CFBundleVersion` (e.g. `"15"` or `"1"`) |
| `date` | string | YES | ISO 8601 format: `"2026-05-16"` or `"2026-05-16T00:00:00"` |
| `downloadURL` | string | YES | Direct URL to the `.ipa` file |
| `size` | number | YES | IPA file size in bytes (integer) |

**Version object — useful optional fields:**

| Field | Type | Notes |
|-------|------|-------|
| `localizedDescription` | string | Release notes / changelog for this version |
| `minOSVersion` | string | e.g. `"18.0"` |
| `maxOSVersion` | string | Use sparingly; omit unless you know a version breaks on newer iOS |

**appPermissions object:**

Required in the AltStore 2.0 spec. Structure:
```json
"appPermissions": {
  "entitlements": [],
  "privacy": {}
}
```
For CoreWatch: `entitlements` is an empty array (no private entitlements), `privacy` lists any `NSUsageDescription` keys the app uses. CoreWatch uses `NSUserNotificationsUsageDescription` (for local notifications — but this is not a `NSUsageDescription` privacy key, it is automatic). If the app has no camera, location, microphone, or contacts access, both can be empty.

---

### Critical SideStore/AltStore Format Compatibility Note

**The `downloadURL` dual-placement issue (RESOLVED in SideStore 0.6.0+):**

SideStore versions before approximately February 2025 required a top-level `downloadURL` key on each app object, in addition to the `downloadURL` key inside each version entry. AltStore Classic only uses the version entry's `downloadURL`. This incompatibility caused the error: `"E downloadURL:String or downloadURLs:[[PlatformURL]] key required."` when adding AltStore-format sources to older SideStore.

The fix landed in SideStore nightly around February 2025 and was included in the 0.6.0 release milestone (PR #807 / #866). SideStore now reads `apps[0].versions[0].downloadURL` as the primary source, with `apps[0].downloadURL` as a backward-compatibility fallback.

**Recommendation:** Include a top-level `downloadURL` on each app object anyway (duplicating `versions[0].downloadURL`) for maximum compatibility with older SideStore installs. This is safe — AltStore ignores the redundant top-level field.

**SideStore also flags `marketplaceID` as a notarized-app indicator.** Do NOT include `marketplaceID` in the source. SideStore will reject the source as a notarized (AltStore PAL) source if this field is present, even if blank.

**Confidence: HIGH** — Confirmed directly from SideStore GitHub issue #735 (bug report + fix timeline), SideStore docs (marketplaceID warning).

---

### Complete Minimal source.json Example

```json
{
  "name": "CoreWatch",
  "identifier": "com.yourname.corewatch-source",
  "sourceURL": "https://raw.githubusercontent.com/YOURNAME/CoreWatch/main/source.json",
  "apps": [
    {
      "name": "CoreWatch",
      "bundleIdentifier": "com.yourname.corewatch",
      "developerName": "Your Name",
      "subtitle": "iPhone thermal, CPU & memory monitor",
      "localizedDescription": "CoreWatch monitors your iPhone's thermal state, CPU usage, and memory in real time with a live dashboard and local alerts.",
      "iconURL": "https://raw.githubusercontent.com/YOURNAME/CoreWatch/main/icon.png",
      "tintColor": "#FF6B35",
      "category": "utilities",
      "downloadURL": "https://github.com/YOURNAME/CoreWatch/releases/download/v1.5/CoreWatch.ipa",
      "appPermissions": {
        "entitlements": [],
        "privacy": {}
      },
      "versions": [
        {
          "version": "1.5",
          "buildVersion": "1",
          "date": "2026-05-16",
          "localizedDescription": "SideStore distribution support — automatic 7-day certificate refresh.",
          "downloadURL": "https://github.com/YOURNAME/CoreWatch/releases/download/v1.5/CoreWatch.ipa",
          "size": 0,
          "minOSVersion": "18.0"
        }
      ]
    }
  ],
  "news": []
}
```

Replace `YOURNAME` with the GitHub username. Replace `size: 0` with the actual byte count of the IPA after export. Replace `buildVersion` with the actual `CFBundleVersion` from the Xcode project.

---

### GitHub Releases Hosting

GitHub Releases is the standard hosting approach for AltStore/SideStore IPA distribution. It is free, has no bandwidth limits for public repos, and produces permanent, versioned URLs.

**Release asset URL format:**
```
https://github.com/{owner}/{repo}/releases/download/{tag}/{filename}
```

Example for CoreWatch v1.5:
```
https://github.com/naujgs/CoreWatch/releases/download/v1.5/CoreWatch.ipa
```

This URL is permanent as long as the release and asset are not deleted. GitHub does not rotate or expire release asset URLs.

**source.json hosting:**

Host `source.json` in the repository itself (committed to `main`), accessed via:
```
https://raw.githubusercontent.com/{owner}/{repo}/main/source.json
```

This URL updates automatically whenever `source.json` is pushed. It is the URL users add to SideStore/AltStore as the source URL.

**GitHub Release structure per version:**

Each CoreWatch version gets one GitHub Release:
- Tag: `v1.5` (semantic version, matches `versions[0].version` in source.json)
- Release title: `CoreWatch v1.5`
- Release body: changelog (same text as `localizedDescription` in source.json)
- Attached assets: `CoreWatch.ipa` (the exported IPA, renamed clearly)

**source.json update on each release:** After creating the GitHub Release and noting the IPA byte size, update `source.json` in the repo: prepend the new version entry to the `versions` array, update the top-level `downloadURL` to point to the new IPA, and push. The `raw.githubusercontent.com` source URL automatically reflects the update.

**Confidence: HIGH** — GitHub Releases asset URL format is stable and documented. Pattern confirmed from multiple real-world AltStore source repositories (Delta, UTM, etc.).

---

### SideStore Deep Link for Source Install

Users add the source to SideStore by tapping this URL (or a button/link that opens it):
```
sidestore://source?url=https://raw.githubusercontent.com/YOURNAME/CoreWatch/main/source.json
```

For AltStore Classic:
```
altstore://source?url=https://raw.githubusercontent.com/YOURNAME/CoreWatch/main/source.json
```

Both schemes can be provided in the install guide. Tapping the appropriate link opens the respective app and pre-fills the source URL field.

---

### Xcode Project Settings — Changes Required for v1.5

No Xcode project settings need to change for SideStore distribution compatibility. The existing configuration is fully compatible:

| Setting | Current Value | SideStore Requirement | Action |
|---------|--------------|----------------------|--------|
| Bundle Identifier | `com.yourname.corewatch` | Must match `bundleIdentifier` in source.json | None — just document the ID in source.json |
| `CFBundleShortVersionString` | Project version (e.g. `1.4`) | Must match `version` in source.json | Bump to `1.5` at release time |
| `CFBundleVersion` | Build number (e.g. `1`) | Must match `buildVersion` in source.json | Record actual value; put in source.json |
| Code signing | Free Apple ID / Personal Team | SideStore re-signs; original signing is irrelevant | None |
| Entitlements | Standard sandbox only | Free Apple ID entitlements only survive re-signing | None — CoreWatch uses no private entitlements |
| Minimum deployment | iOS 18.0 | Must match `minOSVersion` in source.json | None |

**The only action at release time** is to bump the version number (`CFBundleShortVersionString`) in Xcode before archiving, so the IPA's embedded `Info.plist` matches the `version` field in `source.json`. If they do not match, SideStore will not recognize updates.

**Confidence: HIGH** — No special project configuration is needed for SideStore/AltStore compatibility. The re-signing process is done entirely by the tool, not by the IPA's existing signature.

---

### v1.5 Tooling — No New Tools Required

| Task | Tool | Notes |
|------|------|-------|
| IPA export | Xcode 26.4.1 Organizer | Already in use; "Release Testing" export path |
| IPA hosting | GitHub Releases | Free, permanent URLs, already hosting the repo |
| source.json hosting | GitHub (raw.githubusercontent.com) | Committed to repo main branch |
| source.json editing | Any text editor | Hand-edit; no special tooling needed |
| IPA byte size | Finder (Get Info) or `wc -c CoreWatch.ipa` | Needed for `size` field in source.json |
| SideStore install | SideStore app on device | User-facing; not a dev tool |

No new dependencies, no CI/CD, no external services.

---

### v1.5 Confidence Summary

| Area | Confidence | Notes |
|------|------------|-------|
| AltSource JSON format — required fields | HIGH | Verified via official AltStore FAQ documentation |
| SideStore format compatibility | HIGH | Verified via SideStore docs + GitHub issue #735 + official SideStore source.json |
| `downloadURL` dual-placement workaround | HIGH | Confirmed fixed in SideStore 0.6.0; including redundant top-level field is safe |
| `marketplaceID` must be absent | HIGH | Documented explicitly in SideStore docs |
| IPA export via Xcode "Release Testing" | MEDIUM-HIGH | Confirmed pattern; "Release Testing" is the Xcode 16+ name for development distribution |
| Free Apple ID IPA + SideStore re-sign | HIGH | SideStore FAQ confirms re-signing; CoreWatch has no private entitlements that would be stripped |
| GitHub Releases as IPA host | HIGH | Standard pattern used by all major AltStore sources (Delta, UTM, iSH) |
| No Xcode project changes needed | HIGH | Re-signing is tool-side; IPA content compatibility is bundle ID + version string matching only |

---

### v1.5 Sources

- [Make a Source — AltStore FAQ](https://faq.altstore.io/developers/make-a-source) — authoritative AltSource JSON spec
- [App Sources — SideStore Docs](https://docs.sidestore.io/docs/advanced/app-sources) — SideStore compatibility notes, `marketplaceID` warning, deep link format
- [SideStore GitHub — apps.json source.json](https://github.com/SideStore/apps.json/blob/main/_includes/source.json) — official SideStore's own source.json structure
- [SideStore GitHub Issue #735](https://github.com/SideStore/SideStore/issues/735) — `downloadURL` dual-placement bug, fix timeline (0.6.0 / February 2025)
- [SideStore Version interface — sidestore-source-types](https://sidestore.io/sidestore-source-types/interfaces/Version.html) — TypeScript type definitions for version fields
- [SideStore FAQ](https://docs.sidestore.io/docs/faq) — "resigns apps with your personal development certificate" confirmation
- [AltStore App IDs](https://faq.altstore.io/altstore-classic/app-ids) — 10 App ID limit per free Apple ID
- [SideStore Issue #68 — bypass 10 App ID limit](https://github.com/SideStore/SideStore/issues/68) — confirms 10-app limit and universal App ID workarounds
- [iOS Sideloading How It Works 2025 — DEV Community](https://dev.to/1_king_0b1e1f8bfe6d1/how-ios-sideloading-actually-works-in-2025-dev-certs-altstore-and-the-eu-exception-1m2h) — re-signing mechanics
- [How to export an Ad Hoc iOS IPA — Don't Panic](https://www.andrewhoog.com/posts/how-to-export-an-ad-hoc-ios-ipa-using-xcode/) — Xcode archive/export steps
- [Xcode archive export files — Apple Help](https://help.apple.com/xcode/mac/current/en.lproj/deva1f2ab5a2.html) — official Apple Xcode export documentation

---
*Stack research update for: CoreWatch v1.5 — SideStore/AltStore distribution*
*Researched: 2026-05-16*
