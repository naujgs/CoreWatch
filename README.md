# CoreWatch

A personal iOS app that monitors your iPhone's thermal state, CPU usage, and memory in real time — with local notifications before things get dangerously hot.

![Platform](https://img.shields.io/badge/platform-iOS%2018%2B-blue)
![Swift](https://img.shields.io/badge/swift-6.3-orange)
![License](https://img.shields.io/badge/license-personal%20use-lightgrey)

---

## What It Does

CoreWatch gives you a live dashboard of three system metrics, always visible in a clean dark UI:

| Tab | What you see |
|-----|-------------|
| **Thermal** | System thermal state (Nominal / Fair / Serious / Critical), color-coded badge, 30-minute session history chart |
| **CPU** | App CPU % and system-wide CPU % |
| **Memory** | App memory footprint (MB) + system breakdown (Free / Active / Inactive / Wired in GB) |

When the device reaches **Serious** or **Critical** thermal state, a local notification fires — even when the app is backgrounded. One alert per escalation cycle; no spam.

---

## Screenshots

| Thermal | CPU | Memory |
|---------|-----|--------|
| ![Thermal tab showing Nominal state](docs/screenshots/thermal.png) | ![CPU tab showing App CPU 2.0% and System CPU 60.4%](docs/screenshots/cpu.png) | ![Memory tab showing App Memory 33 MB and system breakdown](docs/screenshots/memory.png) |

---

## Why Sideload Only

CoreWatch reads system metrics via the **Mach kernel APIs** (`task_threads`, `host_statistics`, `host_statistics64`) — standard BSD-layer calls available in the iOS sandbox. This is not an App Store app because:

- There is no public API for numeric temperature in degrees Celsius. Apple exposes only a 4-level categorical thermal state via `ProcessInfo.thermalState`.
- The app is a personal tool; App Store review overhead adds no value for a single-device install.
- Sideloading via a free Apple Developer account is sufficient.

---

## Requirements

| Requirement | Version |
|-------------|---------|
| iPhone | Any model running iOS 18.0+ |
| Xcode | 26.4.1 (latest stable) |
| Swift | 6.3 (ships with Xcode 26.4.1) |
| Apple Developer account | Free Apple ID (no $99/yr subscription needed) |
| macOS | Sequoia or later (required by Xcode 26.x) |

> **Important:** Do not install on iOS 17 or earlier. The deployment target is iOS 18.0.

---

## Installation

### 1. Clone the repo

```bash
git clone https://github.com/<your-username>/CoreWatch.git
cd CoreWatch
```

### 2. Open in Xcode

```
open CoreWatch/CoreWatch.xcodeproj
```

### 3. Configure signing

1. In the Xcode project navigator, select the **CoreWatch** target.
2. Go to **Signing & Capabilities**.
3. Set **Team** to your personal Apple ID (shown as "Your Name (Personal Team)").
4. Xcode will auto-create a provisioning profile. If there is a bundle ID conflict, change `com.jgs.CoreWatch` to something unique (e.g. `com.yourname.CoreWatch`).

### 4. Connect your iPhone and install

1. Plug in your iPhone via USB.
2. Select your device in the Xcode toolbar.
3. Press **Cmd + R** (Run). Xcode builds and installs the app.
4. On first launch iOS will ask you to **Trust** the developer certificate:
   - Go to **Settings → General → VPN & Device Management**.
   - Tap your Apple ID under Developer App.
   - Tap **Trust**.

### 5. Grant notification permission

On first launch, CoreWatch requests notification permission. Grant it to receive thermal alerts. If you decline and want to enable it later: **Settings → CoreWatch → Notifications → Allow Notifications**.

---

## Re-installing After 7 Days

Free Apple ID certificates expire after **7 days**. The app stops launching when the cert expires. To renew:

1. Connect iPhone via USB.
2. Open the project in Xcode.
3. Press **Cmd + R**. Xcode re-signs and re-installs automatically.
4. No data is lost (session history is in-memory anyway).

To avoid weekly reinstalls, upgrade to a **$99/yr Apple Developer Program** membership — certificates then last 1 year.

---

## How Thermal Alerts Work

CoreWatch uses **local notifications** (no server, no APNs):

- A `Timer` polls `ProcessInfo.thermalState` every **5 seconds** while the app is active.
- A `ProcessInfo.thermalStateDidChangeNotification` observer fires instantly in the background.
- When the state reaches **Serious or Critical**, one notification is scheduled immediately.
- A cooldown gate prevents repeated alerts for the same escalation. The gate resets when the device cools back below Serious.
- The notification is shown as a banner even while the app is in the foreground.

---

## Privacy & Data Handling

CoreWatch collects nothing and sends nothing anywhere.

| Data | Stored? | Sent? |
|------|---------|-------|
| Thermal state readings | In-memory only (cleared on app close) | Never |
| CPU usage | In-memory only | Never |
| Memory usage | In-memory only | Never |
| Notification permission status | System-managed | Never |
| Personal or user data | Not collected | Never |
| Analytics / crash reports | None | Never |

**No network entitlements are requested.** The app has no outbound connections of any kind.

**No private data is written to disk.** Session history is a plain Swift array in the ViewModel — it disappears when the app is closed or the device reboots.

---

## Architecture

```
CoreWatch/
├── CoreWatchApp.swift          # App entry point, notification delegate lifetime
├── ContentView.swift           # Root TabView (Thermal / CPU / Memory)
├── TemperatureViewModel.swift  # Thermal state polling, notifications, background observer
├── MetricsViewModel.swift      # CPU & memory via Mach kernel APIs
├── ThermalView.swift           # Thermal badge, chart, permission banner
├── CPUView.swift               # App CPU + system CPU cards
├── MemoryView.swift            # App memory + system memory breakdown grid
├── SessionChartView.swift      # Swift Charts step-chart (30-min history)
├── ThermalBadgeView.swift      # Color-coded state badge with glow on Critical
├── ThermalState.swift          # ThermalLevel enum wrapping ProcessInfo.ThermalState
├── DesignTokens.swift          # Colors, typography, spacing, animation constants
├── NotificationDelegate.swift  # UNUserNotificationCenterDelegate (foreground banner display)
├── SystemMetrics.swift         # Debug probe engine (Mach API validation)
└── MachProbeDebugView.swift    # Long-press title → debug sheet
```

**Pattern:** MVVM with `@Observable @MainActor` ViewModels.

**Mach APIs used** (all confirmed accessible under free sideload on iOS 18):

| API | Purpose |
|-----|---------|
| `task_threads` + `thread_info(THREAD_BASIC_INFO)` | Per-app CPU usage |
| `task_info(MACH_TASK_BASIC_INFO)` | App memory footprint |
| `host_statistics(HOST_CPU_LOAD_INFO)` | System-wide CPU |
| `host_statistics64(HOST_VM_INFO64)` | System memory breakdown |

---

## Why No Numeric Temperature

The single most-requested feature on iOS temperature monitors is a numeric readout in °C. Here is why CoreWatch does not have one:

- Apple's **public API** exposes only 4 categorical levels via `ProcessInfo.thermalState` (Nominal, Fair, Serious, Critical). No degrees.
- The **private IOKit path** (`IOPMPowerSource` with the `systemgroup.com.apple.powerlog` entitlement) is blocked by AMFI under a standard free sideload.
- **TrollStore** could bypass this restriction but requires iOS ≤ 17.0. CoreWatch targets iOS 18.

The 4-level system is Apple's deliberate abstraction. CoreWatch uses it as designed.

---

## Localization

The app is localized in **English** and **Spanish**. All metric labels, tooltip text, thermal level names, and notification copy are translated.

To add a language: open `Localizable.xcstrings` in Xcode and add the target locale.

---

## Roadmap

Completed milestones: v1.0 (MVP), v1.1 (visual polish), v1.2 (CPU + memory tabs), v1.3 (design system), v1.4 (project rename).

Planned:
- State duration display ("Serious for 4 min")
- Recovery notification ("Back to Nominal")
- CPU and memory history charts
- Battery level and charge state
- Home screen widget

Not planned:
- App Store distribution
- Numeric °C temperature (blocked by iOS sandbox)
- Android or cross-platform support

---

## License

Personal use. Not for redistribution.
