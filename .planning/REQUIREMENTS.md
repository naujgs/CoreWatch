# Requirements: CoreWatch v1.5 — SideStore Distribution

## Milestone Goal

Package CoreWatch as a SideStore/AltStore source so the app auto-refreshes every 7 days without touching Xcode. Eliminate the weekly USB reinstall cycle.

---

## v1.5 Requirements

### Distribution Infrastructure

- [ ] **DIST-01**: User can add the source URL to SideStore and see CoreWatch listed as an available app
  - source.json at repo root with all required AltSource fields: `name`, `bundleIdentifier`, `developerName`, `localizedDescription`, `iconURL`, `versions[]`, `downloadURL` (app-level + versions-level for SideStore compat), `size`, `permissions`, `appPermissions`
- [ ] **DIST-02**: source.json references a real hosted app icon (not the xcassets entry)
  - 1024×1024 PNG exported from asset catalog to `assets/icon.png`, committed to repo, referenced via raw GitHub URL
- [ ] **DIST-03**: CoreWatch app version is bumped to 1.5.0 before IPA export
  - `CFBundleShortVersionString` set to `1.5.0`, `CFBundleVersion` set to `1` (or incremented) in Xcode project

### Release Workflow

- [ ] **REL-01**: User can export a valid IPA from Xcode that SideStore can install and re-sign
  - Archived via Xcode Organizer, exported as Development distribution; IPA byte size measured with `stat -f%z CoreWatch.ipa`
- [ ] **REL-02**: CoreWatch IPA is hosted at a stable GitHub Releases download URL
  - GitHub Release tagged `v1.5.0` created, `CoreWatch.ipa` attached as release asset
- [ ] **REL-03**: source.json `downloadURL`, `size`, `version`, and `buildVersion` fields are wired to the v1.5.0 IPA
  - Exact values taken from the exported IPA and GitHub Release URL; source.json committed after Release asset is live

### SideStore Setup

- [ ] **SETUP-01**: User has a step-by-step guide to install SideStore on their device and add the CoreWatch source
  - Guide covers: SideStore app install, StosVPN requirement, anisette server setup, adding source URL, installing CoreWatch
- [ ] **SETUP-02**: CoreWatch installs via SideStore source and certificate auto-refreshes without Xcode
  - End-to-end verified on physical iOS 18 device: install from source, manual refresh confirms cert renewed, app remains usable past 7-day mark

---

## Future Requirements (Deferred)

- [ ] GitHub Actions CI — automated IPA build and release on tag push (deferred: manual workflow sufficient for personal use)
- [ ] SideStore screenshots — 9:19.5 aspect ratio marketing screenshots in source.json (deferred: no UX value for personal source)
- [ ] SideStore news items — changelog entries in source.json `news[]` (deferred: release notes in GitHub Release are sufficient)
- [ ] Public source distribution — making source URL available to others (deferred: private personal use only for v1.5)

---

## Out of Scope

- GitHub Actions / CI pipeline — manual Xcode export is the right tool for an infrequent personal release cadence
- Ad Hoc or App Store distribution — free Apple ID supports Development signing only
- AltStore PAL / notarized distribution — requires `marketplaceID` field; incompatible with SideStore and unnecessary for personal use
- TestFlight — requires paid Apple Developer account

---

## Traceability

| REQ-ID | Phase | Status |
|--------|-------|--------|
| DIST-01 | — | Pending |
| DIST-02 | — | Pending |
| DIST-03 | — | Pending |
| REL-01 | — | Pending |
| REL-02 | — | Pending |
| REL-03 | — | Pending |
| SETUP-01 | — | Pending |
| SETUP-02 | — | Pending |
