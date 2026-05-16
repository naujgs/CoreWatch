# Domain Pitfalls

**Domain:** iOS SideStore/AltStore distribution — packaging an existing sideloaded app as a SideStore source with auto-refresh
**Researched:** 2026-05-16
**Milestone:** v1.5 — SideStore Distribution
**Confidence:** HIGH (source.json format, Apple ID limits, IPA signing mechanics, VPN dependency); MEDIUM (GitHub rate-limiting specifics, SideStore vs AltStore field-name differences); LOW (iOS 26.x signing changes — still evolving, target device stays on iOS 18)

> This file covers pitfalls specific to the v1.5 milestone: packaging CoreWatch as a SideStore source,
> producing the IPA, hosting it on GitHub Releases, and relying on SideStore's auto-refresh to eliminate
> the weekly Xcode reinstall. The target device runs iOS 18; the dev account is a free Apple ID.
> v1.0–v1.4 pitfalls (Mach APIs, Swift 6 concurrency, icon requirements) are not repeated here.

---

## Critical Pitfalls

### Pitfall 1: Signing the IPA Before SideStore Receives It Breaks Re-Signing

**What goes wrong:**
SideStore re-signs every IPA with the user's own personal development certificate at install and refresh time. If the IPA you distribute is already signed — specifically with an active Development or Ad Hoc certificate — SideStore strips the existing signature and applies its own. This usually works, but shipping a pre-signed IPA creates two specific failure modes:

1. **Entitlement escalation:** If the IPA was exported with entitlements that cannot be granted to a free Apple ID (e.g., `com.apple.developer.sustained-execution`, push entitlements, or any `com.apple.*` private entitlement), SideStore's re-signing will produce an IPA that iOS rejects with "invalid entitlements." The original Xcode-signed build would have worked because Xcode validates and strips entitlements automatically, but the re-signed copy contains the entitlement declaration without the provisioning profile to back it.

2. **`get-task-allow` conflict:** Development builds set `get-task-allow = true` (enables the Xcode debugger). SideStore must also set this to true to perform re-signing via its local device tunnel. If the source IPA already has `get-task-allow = false` (which Ad Hoc and App Store exports do), SideStore may fail to set it, causing the install to hang or crash after launch.

**Why it happens:**
Developers naturally reach for Xcode's "Archive → Export → Ad Hoc" workflow because that's how they do TestFlight builds. Ad Hoc exports are signed with a Distribution certificate and have `get-task-allow = false` — exactly the opposite of what SideStore needs.

**Prevention:**
Distribute an **unsigned IPA** or a **fake-signed IPA** (also called a zero-signed IPA). Xcode does not expose an "unsigned" export directly in the Organizer, but the following terminal workflow produces one:

```bash
xcodebuild archive \
  -scheme CoreWatch \
  -configuration Release \
  -archivePath /tmp/CoreWatch.xcarchive \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO

# Package as IPA manually
cd /tmp/CoreWatch.xcarchive/Products
mv Applications Payload
zip -r CoreWatch.zip Payload
mv CoreWatch.zip CoreWatch.ipa
```

Alternatively: Xcode Archive → Organizer → "Distribute App" → "Custom" → "Release Testing" → "Locally" → disable signing → Export. This is the GUI equivalent.

CoreWatch uses only standard sandbox entitlements (no private APIs, no push, no App Groups, no Keychain groups) — so the entitlement conflict risk is low but confirming an unsigned export is still the safest baseline.

**Detection:**
- SideStore install succeeds but app crashes immediately on first launch after re-signing
- Xcode device console shows "The executable was signed with invalid entitlements"
- Re-signing with a paid certificate on a separate device works; free Apple ID does not

---

### Pitfall 2: source.json Field-Name Mismatch Between AltStore and SideStore Formats

**What goes wrong:**
SideStore and AltStore share a common JSON schema ancestry but have diverged on field names and parsing behavior. The three most dangerous mismatches:

**a) `downloadURL` vs. `versions` array only**
AltStore allows a `versions` array on each app where each version object contains its own `downloadURL`, and AltStore falls back to `apps[0].versions[0].downloadURL` if no top-level `downloadURL` is present. Older SideStore versions (pre-nightly as of February 2025) require a top-level `downloadURL` key on the app object. Without it, SideStore throws:

```
E downloadURL:String or downloadURLs:[[Platform:URL]] key required
```

And worse: after this failure, SideStore may corrupt its CoreData store, blocking **all** source updates until the malformed source is removed and SideStore is restarted.

**b) `permissions` vs. `appPermissions` field name**
AltStore calls the entitlements+privacy array `appPermissions`. SideStore's community source uses `permissions`. Using the wrong key causes the field to be silently ignored — no error, just no permissions declared. If the source declares no permissions and the downloaded IPA contains privacy usage descriptions (e.g., `NSBluetoothAlwaysUsageDescription`), SideStore's permission mismatch check may refuse the install.

**c) `buildVersion` required in `versions` array**
AltStore 2.x requires `buildVersion` (maps to `CFBundleVersion`, the integer build number) in every version object. SideStore may ignore it if absent, but for cross-compatibility, always include both `version` (the marketing version string, e.g., "1.5.0") and `buildVersion` (the build number, e.g., "15").

**Prevention:**
Use this minimal safe template that satisfies both AltStore and SideStore:

```json
{
  "name": "CoreWatch",
  "identifier": "io.github.naujgs.corewatch",
  "apps": [
    {
      "name": "CoreWatch",
      "bundleIdentifier": "com.yourname.corewatch",
      "developerName": "Your Name",
      "version": "1.5.0",
      "buildVersion": "15",
      "versionDate": "2026-05-16T00:00:00Z",
      "downloadURL": "https://github.com/naujgs/CoreWatch/releases/download/v1.5.0/CoreWatch.ipa",
      "localizedDescription": "iPhone health monitor: thermal state, CPU, and memory.",
      "iconURL": "https://raw.githubusercontent.com/naujgs/CoreWatch/main/icon.png",
      "size": 2621440,
      "permissions": [],
      "versions": [
        {
          "version": "1.5.0",
          "buildVersion": "15",
          "date": "2026-05-16T00:00:00Z",
          "downloadURL": "https://github.com/naujgs/CoreWatch/releases/download/v1.5.0/CoreWatch.ipa",
          "localizedDescription": "SideStore distribution release.",
          "size": 2621440
        }
      ]
    }
  ],
  "news": []
}
```

Always place `downloadURL` at **both** the top-level app object and inside each `versions` entry. The redundancy is intentional cross-compatibility insurance.

**Detection:**
- Source loads in SideStore but no apps appear in the list (field-name mismatch parsing failure)
- "CoreData error" appears when refreshing sources (malformed source corrupted the store)
- App appears but shows "Permissions mismatch — cannot install" at install time

---

### Pitfall 3: `bundleIdentifier` in source.json Must Match Info.plist Exactly (Case-Sensitive)

**What goes wrong:**
The `bundleIdentifier` field in source.json is matched against `CFBundleIdentifier` in the IPA's Info.plist by SideStore at install time. Any mismatch — including capitalization differences — causes SideStore to refuse the install with a generic "app is invalid" error. This also interacts with Apple's App ID registration: the bundle ID registered with your Apple account (when SideStore provisions the app) must match what's in the binary.

The CoreWatch project has been through a rename (`Termostato` → `CoreWatch`). Confirm the bundle ID is consistent across:
- Xcode project → Signing & Capabilities → Bundle Identifier
- Info.plist `CFBundleIdentifier` (should be `$(PRODUCT_BUNDLE_IDENTIFIER)`)
- source.json `bundleIdentifier`
- Any previous App ID registrations on your Apple account

**Secondary issue — App ID already registered (Error 3011):**
When SideStore tries to provision an app, it calls Apple's API to register the bundle ID. If that bundle ID was previously registered by an earlier install (Xcode direct or prior SideStore install) and the 7-day window has not expired, Apple returns "bundle identifier already registered." SideStore may show this as Error 3011 or a generic provisioning failure. The fix is to wait for the prior App ID to expire (up to 7 days from the original install date) or install via Xcode once to force re-provision with the same bundle ID.

**Prevention:**
- Build and install the app via Xcode once with the final bundle ID before creating the SideStore source
- Do not change the bundle ID after the source is created and the app is installed on the device
- Run `grep -r "CFBundleIdentifier" CoreWatch/` and verify the single value matches source.json

**Detection:**
- SideStore Error 3011: "The bundle identifier for this app has already been registered"
- Install fails immediately with "app is invalid" even though the IPA installs cleanly via Xcode

---

### Pitfall 4: Free Apple ID App Slot Limit Collides With SideStore Itself Occupying One Slot

**What goes wrong:**
Apple enforces a **3-app simultaneous install limit** and a **10 App ID registrations per 7-day rolling window** for free Apple IDs. SideStore itself (plus the StosVPN/WireGuard app required for its VPN tunnel) already consumes 1–2 of the 3 slots. CoreWatch as a SideStore-managed app takes another slot. This leaves 0–1 slots for anything else.

The practical collision for this project: if the device already has SideStore + WireGuard + one other app installed under the same Apple ID, adding CoreWatch via SideStore will fail with:

```
Error 1009: You cannot register more than 10 App IDs within a 7-day period
```
or a generic "maximum apps installed" failure.

**The 10-App-ID-per-week limit is a harder wall:** Each SideStore install or refresh of an app consumes one App ID registration for 7 days. Over a month of weekly refreshes, 4 App ID slots are consumed just for CoreWatch. If the same Apple ID is used with other sideloading tools simultaneously (Xcode direct install, other AltStore/SideStore apps), the pool drains faster.

The **SparseRestore bypass** for the 3-app limit was patched in iOS 18.1 beta 5 and does not work on iOS 18.x stable. There is no known bypass for the 10 App ID per week limit.

**Prevention:**
- Use a **dedicated Apple ID exclusively for SideStore** — no Xcode installs with the same ID
- Keep the total number of SideStore-managed apps to 2 (SideStore + CoreWatch) to avoid hitting the 3-slot wall
- Monitor App ID usage in SideStore: My Apps tab → "View App IDs"
- If the 7-day limit is hit, wait — App IDs expire on a rolling basis and slots re-open automatically

**Detection:**
- Error 1009 on install or refresh
- SideStore shows "0 App IDs Remaining" (known SideStore bug that misreports free account status)
- Install fails immediately after the 7th or 8th app in a given week

---

### Pitfall 5: SideStore Refresh Requires the VPN to Be Active — Auto-Refresh Silently Fails Without It

**What goes wrong:**
SideStore's core mechanism involves intercepting iOS's local device communication via a loopback VPN (WireGuard or the newer StosVPN/LocalDevVPN). The VPN **must be active** whenever SideStore performs any install or refresh operation. On iOS 18, the VPN profile must be accepted by the user and active in Settings → VPN.

The failure mode is silent: if SideStore background-refreshes and the VPN is off (e.g., user manually disabled it, iOS killed the VPN profile, or the WireGuard config expired), the refresh fails silently. The app's certificate timer keeps counting down. On day 7 the app stops launching with "app is no longer available" — the user thinks SideStore auto-refresh worked but the app expired anyway.

**SideStore 0.6.x ships StosVPN** as a replacement for WireGuard. StosVPN is SideStore-specific and works on both Wi-Fi and cellular, whereas WireGuard-based LocalDevVPN only works on Wi-Fi. If still using WireGuard, cellular connections will silently prevent refresh.

**Prevention:**
- Use StosVPN (the current default in SideStore 0.6.x), not WireGuard/LocalDevVPN
- Verify the SideStore VPN is active after every iOS update — iOS updates sometimes clear VPN profiles
- After initial SideStore install, confirm at least one successful manual refresh before relying on background auto-refresh
- Do not install the app on the device and immediately switch to cellular — the first post-install refresh must happen on Wi-Fi with VPN active

**Detection:**
- App shows "not on WLAN and/or VPN" error in SideStore (Error 1414)
- App launches with a countdown timer saying "X days remaining" and the count approaches zero without SideStore showing a successful refresh
- SideStore shows the last refresh date as older than 7 days despite device staying connected

---

## Moderate Pitfalls

### Pitfall 6: GitHub Releases `downloadURL` — Use Direct Asset Links, Not `api.github.com` Redirects

**What goes wrong:**
GitHub Releases assets are accessible via two URL patterns:

1. **Direct:** `https://github.com/user/repo/releases/download/v1.5.0/CoreWatch.ipa`
2. **API redirect:** `https://api.github.com/repos/user/repo/releases/assets/12345`

SideStore follows HTTP redirects but GitHub's API endpoints apply rate limits (60 requests/hour for unauthenticated requests). GitHub also changed raw download rate-limiting behavior in December 2024, and additional changes occurred in 2025. The `api.github.com` pattern can return HTTP 429 (rate limit) or 302 redirect chains that SideStore's HTTP client handles inconsistently.

The direct releases download URL (`github.com/releases/download/...`) is CDN-backed and not subject to the API rate limit. Always use this form.

**Additional constraint:** GitHub does not allow files >2 GB in Releases, but CoreWatch IPAs will be well under 50 MB, so this is not a practical limit here.

**Prevention:**
- Format `downloadURL` as `https://github.com/USER/REPO/releases/download/TAG/FILENAME.ipa`
- Never use `api.github.com` URLs in source.json
- After uploading a release asset, copy the URL from the GitHub Releases page download link — this is always the direct CDN-backed URL
- Test the download URL directly in Safari on the device before publishing the source

**Detection:**
- SideStore shows a download progress bar that stalls at 0% or errors after starting
- `curl -I "YOUR_DOWNLOAD_URL"` returns a 302 chain ending at `api.github.com`

---

### Pitfall 7: `size` Field Must Match the Actual IPA File Size in Bytes

**What goes wrong:**
The `size` field in source.json must be the precise byte count of the IPA file. SideStore verifies the downloaded file size against this value. A mismatch causes the install to fail with a vague "The download failed" or "Invalid app" error. Common mistakes:

- Copying the size from a previous release without updating it for the new build
- Using kilobytes or megabytes instead of bytes (`2621440` bytes, not `2560` KB)
- Getting the size before the IPA is fully compressed (e.g., measuring the `.xcarchive` size instead of the `.ipa`)

**Prevention:**
Always measure the final IPA file size immediately before publishing:

```bash
stat -f%z CoreWatch.ipa     # macOS: prints byte count
# or
ls -l CoreWatch.ipa         # byte count in the 5th column
```

Update source.json `size` field with this exact value before committing and before uploading to GitHub Releases.

**Detection:**
- Download completes but install fails with no meaningful error message
- `ls -l CoreWatch.ipa` vs. source.json `size` field shows a mismatch

---

### Pitfall 8: `version` and `buildVersion` in source.json Must Match Info.plist Exactly

**What goes wrong:**
SideStore matches `version` against `CFBundleShortVersionString` and `buildVersion` against `CFBundleVersion` from the downloaded IPA's Info.plist. A mismatch causes SideStore to report an "incorrect version" error and abort the install. This commonly happens when:

- The source.json is updated but Xcode project version numbers are not incremented (or vice versa)
- The developer formats the version differently: source.json says `"1.5"` but Info.plist says `"1.5.0"`
- Build number is left at the default `1` in Xcode while source.json says `15`

**Prevention:**
After archiving, extract the Info.plist from the IPA and verify both fields before publishing:

```bash
# Extract Info.plist from IPA
unzip -p CoreWatch.ipa 'Payload/CoreWatch.app/Info.plist' | plutil -convert xml1 - -o - | grep -A1 "CFBundleShortVersionString\|CFBundleVersion"
```

The values printed must exactly match source.json `version` and `buildVersion`.

**Detection:**
- SideStore error: "The app version does not match"
- App shows up as an "update available" for a version that's already installed

---

### Pitfall 9: Anisette Server Rate-Limiting and Account Lockout From Shared v1 Servers

**What goes wrong:**
SideStore authenticates with Apple using an "Anisette" server that generates one-time machine-authentication tokens. Older v1 Anisette servers are shared among many SideStore users. When many accounts authenticate through the same machine, Apple's security detects the shared hardware fingerprint and flags associated Apple IDs as suspicious, triggering:

- "Too many requests" from the Anisette server
- Apple ID soft-lock requiring a phone-number verification to unlock
- Authentication failure with no clear error message

This does not directly affect app distribution (source.json hosting), but it blocks the initial SideStore login and any refresh operation that needs re-authentication.

**Prevention:**
- Use SideStore 0.4.0 or later (adds v3 Anisette server support, which dramatically reduces account lockout risk)
- Switch to one of the SideStore team's official v3 Anisette servers (listed in Settings → SideStore → Anisette URL) — do not stay on the default if it's a v1 server
- Use a **dedicated throwaway Apple ID** for SideStore, not the primary Apple ID associated with purchases, iCloud data, or other apps
- Do not use the same Apple ID simultaneously with other sideloading tools (Sideloadly, AltServer) — concurrent authentication from multiple machines is a strong lockout signal

**Detection:**
- Login hangs indefinitely or returns "Authentication failed" with no error code
- Apple sends an "Unusual sign-in activity" email to the Apple ID
- Switching to a different Anisette URL resolves the issue immediately

---

### Pitfall 10: Certificate Revocation When SideStore Is Used on More Than One Device With the Same Apple ID

**What goes wrong:**
A free Apple ID can have at most **2 simultaneous iOS Development certificates**. When SideStore installs on a new device and needs to generate a certificate, if 2 already exist, it revokes one. Any SideStore-managed apps signed with the revoked certificate — on any device — stop launching until refreshed with the new certificate.

This is a known recurring bug in SideStore (reported in issues #978 and #1156). In practice: if CoreWatch is installed via SideStore on an iPhone and the same Apple ID is used to install SideStore on a second device, CoreWatch on the first device will stop launching within minutes of the second install.

For CoreWatch's use case (single personal device), this is low risk. It becomes a problem if the same Apple ID is used across an iPhone and an iPad, or shared with another person.

**Prevention:**
- Limit SideStore to a single device per Apple ID
- If multiple devices are needed, use separate Apple IDs (each gets its own 2-certificate quota)
- After any certificate-related incident, manually refresh all affected apps in SideStore before the next launch attempt

**Detection:**
- App stops launching with "app is no longer available" shortly after a SideStore install on a second device
- SideStore shows a certificate error with "Unable to use an existing signing certificate"
- Two different Apple IDs in SideStore (on the same or different devices) referencing overlapping certificates

---

### Pitfall 11: `versions` Array Order Determines Which Version SideStore Treats as "Latest"

**What goes wrong:**
AltStore and SideStore both use the **first entry** in the `versions` array as the "latest" version. This is not semantic versioning — it is purely positional. If an older version entry is accidentally placed first (e.g., by inserting a new release at the bottom of the array), SideStore will offer a "downgrade" to the old version or, worse, show no update available when one exists.

**Prevention:**
Always insert new versions at the **top** (index 0) of the `versions` array. The array should read newest-first, oldest-last:

```json
"versions": [
  { "version": "1.6.0", ... },   // newest first
  { "version": "1.5.0", ... },   // previous
  { "version": "1.4.0", ... }    // older
]
```

**Detection:**
- After publishing a new IPA, SideStore shows no update available
- SideStore offers to install an older version labeled as the current version

---

## Minor Pitfalls

### Pitfall 12: source.json MIME Type and Raw Hosting URL

**What goes wrong:**
SideStore fetches source.json via HTTP and expects `Content-Type: application/json`. If the file is hosted on a GitHub repo branch (e.g., `raw.githubusercontent.com`), GitHub serves it with the correct MIME type. However, if hosted on a static site that maps `.json` to `text/plain` or `application/octet-stream`, SideStore may fail to parse it.

GitHub Pages (`github.io`) serves `.json` files with `application/json`. GitHub raw (`raw.githubusercontent.com`) also works. Both are safe choices.

**Prevention:**
Host source.json on GitHub Pages or `raw.githubusercontent.com`. Test with:

```bash
curl -I "YOUR_SOURCE_JSON_URL"
# Expect: Content-Type: application/json
```

---

### Pitfall 13: `iconURL` Must Be a Reachable HTTPS URL Returning a Valid PNG/JPEG

**What goes wrong:**
SideStore fetches the icon at source load time. If the URL returns a 404, requires authentication, or returns an SVG, SideStore may show a broken image or fail to load the source at all. GitHub raw image URLs are reliable as long as the commit or branch they reference exists.

**Prevention:**
Use a `raw.githubusercontent.com` URL pointing to the `main` branch. Avoid commit-hash-pinned URLs for icons (the hash reference may become orphaned after force pushes or branch cleanups).

---

### Pitfall 14: Xcode Development Terms of Service Block SideStore Login (Error 1102 / -1011)

**What goes wrong:**
If the Apple ID used for SideStore has never accepted Apple's Developer Terms of Service, SideStore will fail to generate the provisioning profile needed for re-signing. This appears as Error 1102 ("Apple ID cannot be used for development") or Error -1011 ("Developer Terms Not Accepted").

A dedicated throwaway Apple ID created specifically for SideStore will trigger this on first use — the TOS must be accepted at `developer.apple.com` using a browser before SideStore can use the account for signing.

**Prevention:**
After creating the dedicated Apple ID: log in at `developer.apple.com`, click "Agree" on the free membership TOS, then use the account in SideStore.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Exporting IPA from Xcode | Signed IPA breaks SideStore re-signing (Pitfall 1) | Use unsigned export via `xcodebuild CODE_SIGNING_REQUIRED=NO` |
| Authoring source.json | `downloadURL` missing at top-level breaks SideStore (Pitfall 2) | Use template with redundant `downloadURL` at both levels |
| `bundleIdentifier` in source.json | Case mismatch causes silent install failure (Pitfall 3) | Extract from Info.plist with `plutil`, copy exactly |
| `size` field | Wrong byte count causes download verification failure (Pitfall 7) | Always run `stat -f%z CoreWatch.ipa` after export |
| `version` / `buildVersion` fields | Mismatch with Info.plist aborts install (Pitfall 8) | Extract from IPA with `unzip -p` before committing source.json |
| Publishing GitHub Release | Using api.github.com URL instead of direct CDN URL (Pitfall 6) | Copy URL from GitHub Releases page download button only |
| First device install via SideStore | Apple ID TOS not accepted (Pitfall 14) | Log into developer.apple.com first |
| First SideStore login | Anisette server lockout (Pitfall 9) | Use v3 Anisette server, dedicated Apple ID |
| App slot planning | 3-slot limit blocks install if Apple ID has other apps (Pitfall 4) | Use dedicated Apple ID for SideStore only |
| Testing auto-refresh | VPN not active causes silent refresh failure (Pitfall 5) | Verify StosVPN is on; test first manual refresh before trusting background |
| Updating source.json for v1.6+ | New version entry inserted at wrong array position (Pitfall 11) | Insert newest version at index 0 of `versions` array |

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Signed IPA blocking re-sign | LOW | Re-export with `CODE_SIGNING_REQUIRED=NO`; re-upload to GitHub Releases |
| source.json field-name mismatch | LOW | Fix JSON; remove source from SideStore; re-add — may need SideStore restart if CoreData is corrupt |
| bundleIdentifier mismatch | MEDIUM | Align all three sources (Xcode, Info.plist, source.json); may need to wait 7 days for old App ID to expire |
| App slot limit hit | LOW | Wait for App IDs to expire (rolling 7-day window); or switch to dedicated Apple ID |
| VPN silent refresh failure | LOW | Enable StosVPN; force manual refresh from SideStore My Apps |
| GitHub API rate-limiting | LOW | Replace `api.github.com` URLs with direct CDN release asset URLs |
| Wrong `size` field | LOW | Update source.json with correct byte count; increment version or create new release |
| Version mismatch | LOW | Align `version`/`buildVersion` in source.json with Info.plist; re-publish |
| Anisette lockout | MEDIUM | Switch Anisette server URL in SideStore Settings; may need to recover Apple ID via Apple support if soft-locked |
| Certificate revoked by second device | MEDIUM | On all affected devices: SideStore → My Apps → long-press app → Refresh; do this before app expires |

---

## Sources

- [AltStore: Make a Source](https://faq.altstore.io/developers/make-a-source) — authoritative source.json field specification
- [SideStore: Error Codes](https://docs.sidestore.io/docs/troubleshooting/error-codes) — Error 1009, 3011, 3013, 1414, 1102, -1011 definitions
- [SideStore: FAQ](https://docs.sidestore.io/docs/faq) — 3-app limit, 10-App-ID limit, Apple ID restrictions
- [SideStore: Common Issues](https://docs.sidestore.io/docs/troubleshooting/common-issues) — VPN dependency, minimuxer, AFC errors
- [SideStore Issue #735](https://github.com/SideStore/SideStore/issues/735) — downloadURL vs versions-only parsing incompatibility; CoreData corruption on malformed source
- [SideStore Issue #978](https://github.com/SideStore/SideStore/issues/978) — Certificate revocation when installing on multiple devices
- [SideStore Issue #1156](https://github.com/SideStore/SideStore/issues/1156) — Previous developer accounts soft-locked; "0 App IDs Remaining" false report
- [SideStore Issue #782](https://github.com/SideStore/SideStore/issues/782) — Invalid entitlement error when app has more than 3 App Groups/Keychain groups
- [SideStore: Custom Anisette Server](https://docs.sidestore.io/docs/advanced/anisette) — v3 vs v1 Anisette server differences; account lockout risk
- [MrKai77/Export-unsigned-ipa-files](https://github.com/MrKai77/Export-unsigned-ipa-files) — Tutorial: `CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO` build flags
- [SideStore/apps.json source.json](https://github.com/SideStore/apps.json/blob/main/_includes/source.json) — Official SideStore source reference implementation
- [SideStore sidestore-source-types: Permission interface](https://sidestore.io/sidestore-source-types/interfaces/Permission.html) — Permission type enumeration (15 types)
- [AltStore: App IDs](https://faq.altstore.io/altstore-classic/app-ids) — App ID limit mechanics and expiration
- [How iOS Sideloading Actually Works in 2025 (DEV Community)](https://dev.to/1_king_0b1e1f8bfe6d1/how-ios-sideloading-actually-works-in-2025-dev-certs-altstore-and-the-eu-exception-1m2h) — certificate chain, 7-day expiry, free Apple ID mechanics
- [SideStore Breaks on iOS 26.4 Beta After Apple Change (onejailbreak.com)](https://onejailbreak.com/blog/apple-targets-sidestore-signing-in-ios-26-4-beta/) — VPN loopback targeted in iOS 26.4 beta (not iOS 18; noted for awareness only)
- [GitHub API Rate Limiting Discussion #146957](https://github.com/orgs/community/discussions/146957) — December 2024 raw download rate-limiting change
- [AltStore Error Codes](https://faq.altstore.io/altstore-classic/error-codes) — cross-reference for AltStore vs SideStore error code semantics

---

*Pitfalls research for: CoreWatch v1.5 — SideStore distribution, source.json, IPA signing, GitHub Releases hosting*
*Researched: 2026-05-16*
