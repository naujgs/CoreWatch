# Feature Landscape: AltStore/SideStore Distribution (v1.5)

**Domain:** AltStore/SideStore source.json distribution for a personal sideloaded iOS app
**Researched:** 2026-05-16
**Confidence:** HIGH — AltStore official docs are authoritative; SideStore explicitly defers to AltStore format

---

## Context

CoreWatch v1.5 adds SideStore distribution so the app auto-refreshes every 7 days without Xcode. This document maps the feature landscape for:
1. The `source.json` manifest (what fields to include)
2. The update detection mechanism (what triggers the in-app update prompt)
3. The release workflow (IPA → GitHub Releases → source.json bump)
4. Optional polish features (screenshots, changelogs, news items)

SideStore is fully format-compatible with AltStore sources. All field specs from AltStore docs apply directly.

---

## Table Stakes

Features required for the source to function. Missing any of these = source loads but app cannot install or update.

| Feature | Why Required | Complexity | Notes |
|---------|--------------|------------|-------|
| `source.name` | Displayed in SideStore source list | Trivial | String, e.g. "CoreWatch" |
| `source.apps[]` | The array that holds app entries | Trivial | Must be present even with one app |
| `app.name` | App display name in store UI | Trivial | "CoreWatch" |
| `app.bundleIdentifier` | Links installed app to source for update detection | Trivial | Must exactly match CFBundleIdentifier in Xcode project |
| `app.versions[]` | Version history; drives update detection | Low | Array ordered newest-first; first entry = "latest" |
| `version.version` | Compared against installed CFBundleShortVersionString | Trivial | Must exactly match — this is what triggers update badge |
| `version.downloadURL` | Where SideStore fetches the IPA | Trivial | GitHub Releases raw asset URL is the right approach |
| `version.size` | Required by spec; SideStore validates download integrity | Low | Get exact byte count after IPA export: `wc -c < CoreWatch.ipa` |
| `app.localizedDescription` | Required by AltStore spec for app page rendering | Trivial | Plain text, can be short |
| `app.iconURL` | App icon shown in source UI | Low | Must be a publicly reachable URL; use GitHub raw URL for icon PNG |
| `app.developerName` | Developer credit on app page | Trivial | Your name |

---

## Strongly Recommended (Not Strictly Required, But Expected)

Fields that SideStore will accept without, but which produce a degraded or confusing UX without them.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| `version.date` | Shows release date on app page; used for display ordering in UI | Trivial | ISO 8601: "2026-05-16" |
| `version.localizedDescription` | Per-version changelog text shown in "What's New" | Trivial | Plain text bullet list of changes for this release |
| `version.buildVersion` | Build number; used alongside `version` in update comparison to catch same-version rebuilds | Trivial | Must match CFBundleVersion |
| `version.minOSVersion` | Prevents install on incompatible devices | Trivial | "18.0" for CoreWatch |
| `app.subtitle` | One-line description shown in Browse tab | Trivial | "Live thermal, CPU & memory monitor" |
| `app.tintColor` | Hex color themes the app detail page | Trivial | Match app icon accent color |
| `source.identifier` | Unique source identifier; older AltStore docs list as required | Trivial | Reverse-domain format, e.g. "com.yourname.corewatch-source" |
| `source.sourceURL` | The canonical URL of the source.json itself | Trivial | Enables faster refresh; use GitHub raw URL |
| `app.appPermissions` | Declares entitlements and privacy permissions | Low | AltStore 2.0+ validates this against IPA contents. See details below. |

---

## Differentiators (Polish, Not Expected for Personal App)

Features that set a polished source apart. Not expected for a personal single-app source, but worth noting for completeness.

| Feature | Value | Complexity | Notes |
|---------|-------|------------|-------|
| Screenshots | Visual preview on app detail page | Medium | Requires hosting image assets; 9:19.5 ratio for iPhone |
| `source.iconURL` | Source-level icon in the "Sources" list | Low | Can reuse app icon |
| `source.description` | Full About page for the source | Trivial | Paragraph of text |
| `news[]` items | Announcement/changelog news cards in AltStore's News tab | Low | Good for announcing updates; includes `notify: true` for push to users |
| `source.headerURL` | Banner image on source About page | Low | 3:2 aspect ratio |
| `source.website` | Link to project page | Trivial | GitHub repo URL |
| Multiple versions in `versions[]` | Allows users to install older versions | Trivial | Prepend new entry on each release; keep history |

---

## Anti-Features

Things to explicitly NOT do in the source.json for a personal sideloaded app.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| `marketplaceID` | Required only for AltStore PAL notarized apps (EU distribution) | Omit entirely |
| `patreonURL` / `patreon` object | Patron-gating makes no sense for a personal app | Omit entirely |
| `fediUsername` | Only relevant for AltStore's federated discovery (explore.alt.store); cannot be changed later | Omit unless intentionally publishing to AltStore discovery |
| Hosting IPA anywhere except GitHub Releases | GitHub Releases provides stable, permanent URLs for tagged releases | Use tagged release assets only |
| Matching `version` string to marketing name instead of CFBundleShortVersionString | Update detection breaks — SideStore compares this field against the installed binary's version | Always use the exact plist string: "1.5.0" not "v1.5" |
| Omitting `size` and guessing it | SideStore may reject or warn on mismatched download size | Always compute after export with `wc -c < app.ipa` |

---

## Update Flow: How SideStore Detects New Versions

This is the critical mechanism. Confirmed from AltStore official docs (HIGH confidence).

### Detection Algorithm

1. SideStore periodically re-fetches `source.json` from the `sourceURL`.
2. For each app the user has installed (matched by `bundleIdentifier`), SideStore reads the `versions[]` array from the refreshed source.
3. **The first entry in `versions[]` is treated as the latest release** — position matters, not semantic version comparison.
4. SideStore compares the first entry's `version` (and optionally `buildVersion`) against what is installed on device.
5. If `version` or `buildVersion` differs from the installed binary → update badge appears in "My Apps" tab; user can tap to install.
6. If `minOSVersion`/`maxOSVersion` make the first entry incompatible, SideStore walks down the array to find the first compatible entry, then compares that against installed.

**Critical:** AltStore/SideStore does NOT use the `date` field to determine newer vs older. Date is display-only. Ordering of the array is what counts.

### Update Trigger Workflow (for CoreWatch releases)

```
Xcode archive
  -> export IPA
  -> tag GitHub release (e.g., v1.5.0)
  -> attach IPA as release asset
  -> copy raw asset URL
  -> PREPEND new entry to versions[] in source.json
  -> update version and buildVersion to match Xcode project
  -> push source.json to GitHub (raw URL becomes immediately live)
  -> SideStore fetches on next refresh cycle -> update badge appears
```

The update is immediately live once `source.json` is pushed. SideStore refresh is user-initiated ("Refresh All" button) or happens on a system schedule.

### Known Issue: Caching

SideStore has a documented bug where update detection sometimes fails due to caching of old source data. Workaround: user removes and re-adds the source. This is a SideStore client issue, not a source.json authoring issue.

---

## Minimal Viable source.json (Functional)

```json
{
  "name": "CoreWatch",
  "identifier": "com.yourname.corewatch-source",
  "sourceURL": "https://raw.githubusercontent.com/yourname/CoreWatch/main/source.json",
  "apps": [
    {
      "name": "CoreWatch",
      "bundleIdentifier": "com.yourname.corewatch",
      "developerName": "Your Name",
      "subtitle": "Live thermal, CPU & memory monitor",
      "localizedDescription": "CoreWatch monitors your iPhone's thermal state, CPU usage, and memory in real time. Alerts you before it gets dangerously hot.",
      "iconURL": "https://raw.githubusercontent.com/yourname/CoreWatch/main/icon.png",
      "tintColor": "FF6B35",
      "versions": [
        {
          "version": "1.5.0",
          "buildVersion": "5",
          "date": "2026-05-16",
          "localizedDescription": "- SideStore distribution support\n- Auto-refresh every 7 days",
          "downloadURL": "https://github.com/yourname/CoreWatch/releases/download/v1.5.0/CoreWatch.ipa",
          "size": 1234567,
          "minOSVersion": "18.0"
        }
      ],
      "appPermissions": {
        "entitlements": [],
        "privacy": {
          "NSUserNotificationsUsageDescription": "CoreWatch sends alerts when your device reaches Serious or Critical thermal state."
        }
      }
    }
  ],
  "news": []
}
```

---

## Polished source.json Additions

Beyond the MVP, these fields improve the in-app presentation at negligible cost:

```json
{
  "description": "Personal monitoring tool for iPhone thermal, CPU, and memory health.",
  "iconURL": "same URL as app icon",
  "website": "https://github.com/yourname/CoreWatch",
  "screenshots": [
    {
      "imageURL": "https://raw.githubusercontent.com/yourname/CoreWatch/main/screenshots/thermal.png",
      "width": 393,
      "height": 852
    }
  ],
  "news": [
    {
      "title": "CoreWatch 1.5 — SideStore Distribution",
      "identifier": "corewatch-1-5-release",
      "caption": "Now installs and auto-refreshes via SideStore.",
      "tintColor": "FF6B35",
      "date": "2026-05-16",
      "notify": true,
      "appID": "com.yourname.corewatch"
    }
  ]
}
```

---

## appPermissions Detail for CoreWatch

CoreWatch's actual permission surface under free Apple ID sideload:

**Entitlements:** None beyond standard sandbox. Free Apple ID does not grant private entitlements. Leave `entitlements` as empty array `[]`.

**Privacy permissions used:**
- `NSUserNotificationsUsageDescription` — local notifications for thermal alerts

CoreWatch does NOT use: camera, microphone, location, contacts, health, Bluetooth, local network. `appPermissions.privacy` can be minimal or empty for a monitoring-only app.

Note: AltStore 2.0+ validates `appPermissions` against the IPA's embedded entitlements and Info.plist. If the declared permissions don't match what's in the binary, AltStore may refuse install. Keep this accurate.

---

## Release Workflow Features

The workflow has three implementation components with distinct complexity:

| Component | What to Build | Complexity | Notes |
|-----------|--------------|------------|-------|
| IPA export | Xcode archive + export to `.ipa` | Low | Distribution method: "Ad Hoc" or "Development"; free Apple ID uses Development |
| GitHub Release | Tag, attach IPA, note the raw asset URL | Trivial | URL format: `https://github.com/{user}/{repo}/releases/download/{tag}/{file}.ipa` |
| source.json update | Prepend new version entry, bump version/buildVersion/date/downloadURL/size | Low | Must be done on each release; a shell snippet automates the size calculation |
| Install guide | Document: add source URL in SideStore, install CoreWatch, auto-refresh replaces Xcode weekly reinstall | Trivial | One-time user action |

---

## Feature Dependencies

```
GitHub Releases hosting IPA → downloadURL in source.json
source.json publicly hosted (GitHub raw) → SideStore can fetch it
bundleIdentifier in source.json matches Xcode project → update detection works
version in source.json matches CFBundleShortVersionString → update badge appears
versions[] ordered newest-first → correct "latest" detection
```

---

## MVP Recommendation

For v1.5, build only:

1. `source.json` with all Table Stakes fields plus Strongly Recommended fields (version, buildVersion, date, localizedDescription per version, minOSVersion, subtitle, tintColor, appPermissions)
2. One GitHub Release with IPA attached
3. Install guide (how to add source URL in SideStore, how to install)

Defer screenshots — requires hosting image assets and adds no functional value for a personal single-user app.
Defer news items — adds polish but no functional value for v1.5.

---

## Sources

- [Make a Source | AltStore](https://faq.altstore.io/developers/make-a-source) — official source.json schema (HIGH confidence)
- [Updating Apps | AltStore](https://faq.altstore.io/developers/updating-apps) — update detection algorithm (HIGH confidence)
- [altstoreio/FAQ: make-a-source.md on GitHub](https://github.com/altstoreio/FAQ/blob/main/developers/make-a-source.md) — raw source for the above
- [App Sources | SideStore Docs](https://docs.sidestore.io/docs/advanced/app-sources) — confirms SideStore is fully AltStore-format-compatible
- [SideStore/apps.json source.json example](https://github.com/SideStore/apps.json/blob/main/_includes/source.json) — real-world SideStore source example
- [Apps not detecting new updates (SideStore issue #975)](https://github.com/SideStore/SideStore/issues/975) — caching bug context (MEDIUM confidence — open issue)
- [AltStore-Docs Sources (noah978 GitBook)](https://noah978.gitbook.io/altstore-docs/sources) — additional format reference (MEDIUM confidence — community docs, not official)
- [App Permissions System | DeepWiki](https://deepwiki.com/altstoreio/AltStore/5.2-app-permissions-system) — appPermissions validation behavior (MEDIUM confidence)
