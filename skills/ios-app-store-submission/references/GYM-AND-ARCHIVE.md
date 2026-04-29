# gym and Archive

`fastlane gym` (alias `build_app`) wraps `xcodebuild archive` + `xcodebuild -exportArchive`. When it fails, drop down to raw `xcodebuild` to see the real error — `gym`'s summary often hides the actual line.

For Flutter, `flutter build ipa` is an alternative that wraps the same xcodebuild commands but adds Flutter-specific steps (Dart compilation, plugin pod install).

## Choosing the build path

| Scenario | Use |
|---|---|
| Native Swift/SwiftUI/UIKit + fastlane setup | `fastlane gym` |
| React Native + fastlane setup | `fastlane gym` (after `pod install`) |
| Flutter, first-time submission, no fastlane | `flutter build ipa --release --export-options-plist=ios/ExportOptions.plist` |
| Flutter, repeat releases / CI | `flutter build ipa --no-codesign` then `fastlane gym` (manual archive flow) OR raw `flutter build ipa` (simpler but less control) |
| Capacitor + fastlane | `npx cap sync ios && fastlane gym` |
| Diagnosing a `gym` failure | Drop to raw `xcodebuild archive` |

## Gymfile

`Gymfile` holds defaults; lane args override.

```ruby
# fastlane/Gymfile
workspace "Runner.xcworkspace"
scheme "Runner"
configuration "Release"
output_directory "build/ios/ipa"
output_name "MyApp"
include_symbols true
include_bitcode false  # Apple removed bitcode — leave false
clean true             # slower but reproducible for releases
export_method "app-store-connect"  # newer name for "app-store"
```

Common keys:

- `workspace` / `project` — `.xcworkspace` if Pods/SPM, else `.xcodeproj`
- `scheme` — Xcode scheme name (`Runner` for Flutter, `<App>` for RN)
- `configuration` — `Release` for ASC, `Debug` for local
- `output_directory` — where the IPA lands
- `include_symbols` — `true` for App Store (dSYMs uploaded), `false` for ad-hoc
- `include_bitcode` — Apple removed bitcode requirement; always `false` now
- `clean` — `true` for releases (~20% slower, deterministic)
- `export_method` — see ExportOptions section
- `manageAppVersionAndBuildNumber` — Xcode 14+ auto-bumps in some flows; usually disable to keep your own bump logic deterministic

## ExportOptions.plist

Used by `xcodebuild -exportArchive` and indirectly by `gym`. The single source of truth for *how* the archive becomes an IPA.

### Minimal app-store template (automatic signing, what worked)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>teamID</key>
    <string>ABCD123456</string>
    <key>signingStyle</key>
    <string>automatic</string>
    <key>destination</key>
    <string>export</string>
    <key>uploadSymbols</key>
    <true/>
    <key>stripSwiftSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

Key notes:

- `method` — Xcode 15+ renamed `app-store` → `app-store-connect`. Both work in Xcode 26 but prefer `app-store-connect`.
- `teamID` — your 10-char Apple Developer Team ID (find in developer.apple.com → Membership)
- `signingStyle: automatic` — Xcode auto-fetches certs and profiles. Requires logged-in Apple ID account in Xcode UI for the export step (see OP-4, OP-5, OP-6).
- `signingStyle: manual` — see manual variant below; needed when you have entitlements that the auto-managed profile won't include.
- `uploadSymbols: true` — bundles dSYMs for crash symbolication. Always enable for App Store.
- `stripSwiftSymbols: true` — reduces binary size; recommended.
- `compileBitcode: false` — bitcode is dead.
- Don't set `manageAppVersionAndBuildNumber: true` unless you want Xcode to auto-bump build numbers (conflicts with explicit bump scripts).

### Manual signing variant

When automatic fails or you have specific entitlement needs:

```xml
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>teamID</key>
    <string>ABCD123456</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>signingCertificate</key>
    <string>Apple Distribution</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>com.example.app</key>
        <string>match AppStore com.example.app</string>
    </dict>
    <key>destination</key>
    <string>export</string>
    <key>uploadSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
```

The `provisioningProfiles` dict maps bundle ID → profile name (the human-readable name from the profile, not the UUID). Names from `match` are predictable: `match AppStore <bundleId>`.

For multi-target apps (Share Extension, Watch app, Widget), add an entry per bundle ID.

## Cloud signing vs local signing (read this!)

Modern Xcode supports two signing flavors when `signingStyle: automatic`:

1. **Cloud signing:** Apple holds the private key in their cloud and signs on demand when Xcode is logged in to your Apple ID. Convenient — no manual cert setup. **Failure mode:** session lapses (24h tokens, sign-out, network blips), and CLI `xcodebuild` can't reach the cloud auth context. First archive may succeed; subsequent archives fail with `No signing certificate "iOS Distribution" found` and `No Accounts`. See OP-4.

2. **Local signing:** real Apple Distribution cert in your login keychain. Stable, deterministic. Set up once via Xcode → Settings → Accounts → Manage Certificates → + → Apple Distribution.

**Recommendation:** Always create a local Distribution cert before Phase 2. Don't rely on cloud signing for repeat builds. Cloud is fine for the very first archive; flip to local after that.

When using ASC API key for cloud signing operations (`-authenticationKeyPath` etc. on `xcodebuild`), the API key role must be **Admin** or **Account Holder**. **App Manager** insufficient — see OP-5. For pure upload (post-archive), App Manager is fine.

## Two-stage signing: archive ≠ export

Apple's archive pipeline signs the binary twice. Understanding which stage does what saves panic when you read the archive log.

| Stage | Command | What signs the binary | What you'll see in the log |
|---|---|---|---|
| **Archive** | `xcodebuild archive` | First profile that matches the bundle ID — often a Development profile if that's what's installed | `Signing Identity: "Apple Development: <name>"`, `Provisioning Profile: "iOS Team Provisioning Profile: *"` |
| **Export** | `xcodebuild -exportArchive` (with `method: app-store-connect` in `ExportOptions.plist`) | Re-signs with `Apple Distribution` cert + App Store provisioning profile | The export log is quieter; verify with `codesign -dvvv` on the exported `.app` |

**Implication:** When the archive log shows Development signing, it doesn't mean the build is unshippable. The export step is what determines App Store eligibility. Don't panic when reading archive output — read for *errors*, not for the cert name. See [OP-40](OPERATOR-PATTERNS.md#op-40-xcodebuild-archive-log-shows-development-signing--distribution-happens-at--exportarchive) for the verification flow.

Verification after export:
```bash
codesign -dvvv "build/ipa/MyApp.app" 2>&1 | grep -E "Authority|TeamIdentifier"
# Want: Authority=Apple Distribution: <Team Name>
```

## Raw xcodebuild fallthrough

When `gym` fails, run the underlying commands directly to see the real error:

### Archive

```bash
xcodebuild archive \
  -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/ios/archive/Runner.xcarchive \
  -allowProvisioningUpdates \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=<10-char Team ID> \
  | xcpretty
```

Key flags:
- `-destination 'generic/platform=iOS'` — required; not a specific simulator
- `-archivePath` — where to write the archive
- `-allowProvisioningUpdates` — also auto-registers fresh bundle IDs in the Developer Portal on first archive ([OP-41](OPERATOR-PATTERNS.md#op-41-allowprovisioningupdates-auto-registers-brand-new-bundle-ids-in-the-developer-portal))
- `CODE_SIGN_STYLE` / `DEVELOPMENT_TEAM` as `xcconfig` overrides — necessary when project settings don't include them, e.g. fresh XcodeGen projects before signing config is added to `project.yml` ([OP-39](OPERATOR-PATTERNS.md#op-39-xcodegen-managed-projects-put-signing-config-in-projectyml-never-edit-the-xcodeproj))
- Pipe through `xcpretty` for readable output (or omit if you need full xcodebuild logs)

### Export

```bash
xcodebuild -exportArchive \
  -archivePath build/ios/archive/Runner.xcarchive \
  -exportOptionsPlist ios/ExportOptions.plist \
  -exportPath build/ios/ipa-new \
  -allowProvisioningUpdates
```

Key flags:
- `-allowProvisioningUpdates` — let Xcode regenerate profiles when needed (essential after cert changes; see OP-6)
- `-authenticationKeyPath`, `-authenticationKeyID`, `-authenticationKeyIssuerID` — for non-interactive signing via ASC API key (requires Admin role; see OP-5)

**Both `archive` and `-exportArchive` should usually receive `-allowProvisioningUpdates`** — archive uses it to register the bundle ID in the Developer Portal on first run; export uses it to refresh the Distribution profile after cert changes.

### Reading the full log

When export fails:

```bash
ls -lt /var/folders/h2/cg2_qmws*/T/Runner_*.xcdistributionlogs/  # find latest
cat /var/folders/h2/.../Runner_<timestamp>.xcdistributionlogs/IDEDistribution.standard.log
```

When archive fails:

```bash
# fastlane writes to:
cat fastlane/report.xml

# xcodebuild buffer:
~/Library/Logs/gym/<scheme>-<configuration>.log
```

When IB compiler fails (Internal error on storyboard):

```bash
ls -lt /var/folders/h2/cg2_qmws*/T/IB-agent-diagnostics_*  # find latest
# Inside that bundle, look for stack traces — usually impenetrable, but
# the bundle path itself confirms it's an IB compiler bug. See OP-7.
```

## When to suspect what

- **"Code signing is required..."** — no signing identity matches the configured profile. Check `security find-identity -v -p codesigning`. Likely missing Distribution cert (OP-4).
- **"No Accounts"** — Xcode CLI can't see a logged-in Apple ID. Open Xcode UI, sign in, retry.
- **"Cloud signing permission error"** — API key role too low. Either upgrade key or fall back to local cert (OP-5).
- **"Provisioning profile X doesn't include signing certificate Y"** — cert / profile mismatch. Use `-allowProvisioningUpdates` to let Xcode regenerate (OP-6). If it still fails, the auto-managed profile may need to be deleted in developer.apple.com first.
- **"No applicable devices found"** — wrong destination (used a specific simulator instead of `generic/platform=iOS`).
- **"exportArchive: error: exportOptionsPlist error"** — malformed plist. Run `plutil -lint ios/ExportOptions.plist` to find the issue.
- **"Internal error. Please file a bug at feedbackassistant.apple.com"** with `IB-agent-diagnostics` — Xcode 26 IB compiler hit a stricter check. Simplify the storyboard (OP-7).
- **`gym` summary says "Build failed" with no details** — pipe through `xcpretty -r html` and open `report.html`, or run `xcodebuild` directly to see the real error.

## dSYMs

Located at `<archive>/dSYMs/Runner.app.dSYM` after archive. Used for symbolicating crash reports.

Upload to crash reporters:

```bash
# Crashlytics (Firebase)
"$PODS_ROOT/FirebaseCrashlytics/upload-symbols" -gsp Runner/GoogleService-Info.plist \
  -p ios path/to/Runner.app.dSYM

# Sentry
fastlane run upload_symbols_to_sentry dsym_path:"path/to/Runner.app.dSYM" \
  org_slug:"YOUR_ORG" project_slug:"YOUR_PROJECT" auth_token:$SENTRY_TOKEN

# Bugsnag
fastlane run upload_symbols_to_bugsnag dsym_path:"path/to/Runner.app.dSYM"
```

For TestFlight builds where Apple bitcode-recompiled (legacy), download the Apple-symbolicated dSYMs:

```bash
fastlane run download_dsyms version:"1.0.0" build_number:"2"
```

## Multi-target / extensions

Apps with extensions (Share, Widget, Watch app, App Clip, Messages) need:

1. Each extension to have its own bundle ID, registered in developer.apple.com
2. Each extension to have its own provisioning profile
3. `provisioningProfiles` dict in `ExportOptions.plist` to list every bundle ID:

```xml
<key>provisioningProfiles</key>
<dict>
    <key>com.example.app</key>
    <string>match AppStore com.example.app</string>
    <key>com.example.app.ShareExtension</key>
    <string>match AppStore com.example.app.ShareExtension</string>
    <key>com.example.app.watchkitapp</key>
    <string>match AppStore com.example.app.watchkitapp</string>
</dict>
```

Each extension also needs its own signing target in Xcode and matching entitlements.

## Common gym failures (with fixes)

- **"Your account does not have permission"** — signing role / team membership issue. Check Apple Developer portal → Membership → role.
- **"Provisioning profile X doesn't include the currently selected device"** — wrong profile type for the export method. Verify profile is App Store type.
- **"Code signing is required for product type 'Application' in SDK 'iOS X.Y'"** — no signing identity. See cloud-vs-local section.
- **"No applicable devices found"** — wrong `destination` flag. Use `generic/platform=iOS`.
- **`exportArchive: error: exportOptionsPlist error`** — malformed plist. `plutil -lint` to debug.
- **"Internal error" with IB diagnostics path** — Xcode IB compiler bug. Simplify storyboard (OP-7).
- **"Cannot find module 'Foundation'"** in archive — Swift compiler issue, usually transient. Clean derived data: `rm -rf ~/Library/Developer/Xcode/DerivedData`.
- **`flutter build ipa` fails with empty stdout but nonzero exit** — pipe is buffering. Run without piping to see error: `flutter build ipa --release --export-options-plist=ios/ExportOptions.plist 2>&1 | tee /tmp/flutter_build.log`.
