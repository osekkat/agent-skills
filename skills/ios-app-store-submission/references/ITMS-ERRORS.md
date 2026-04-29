# ITMS Error Catalog

ITMS-XXXXX errors come from App Store validation/upload. Each entry has cause, fix, and prevention. Codes drift — verify the current message in your ASC inbox before relying on this catalog.

## Distinguishing errors from warnings

Apple's ASC distinguishes:

- **Errors** — block processing. Build doesn't reach "Ready to Submit". Email subject is "App Store Connect: Your Submission Was Rejected". Must be fixed.
- **Warnings** — non-blocking. Build processes successfully and shows as "Complete" with a yellow ⚠️ triangle next to the green ✓. Email subject is "App Store Connect: Your Submission Was Successful With Warnings". Submittable but App Reviewers may flag.

Both surface as ITMS-XXXXX codes. Check the email subject to know which category.

## Common errors and warnings

### ITMS-90683 — Missing purpose string in Info.plist (warning, often)

**Symptom in email/UI:**
> Missing purpose string in Info.plist. Your app's code references one or more APIs that access sensitive user data, or the app has one or more entitlements that permit such access. The Info.plist file for the "Runner.app" bundle should contain a `NS<Category>UsageDescription` key with a user-facing purpose string explaining clearly and completely why your app needs the data.

**Cause:** A linked SDK references a sensitive-data API (location, camera, mic, photo library, contacts, calendar, motion, Bluetooth, local network, ATT, etc.) but your `Info.plist` is missing the corresponding `NS*UsageDescription` key. Apple's binary scanner walks symbol references — even if your *app* code doesn't directly call the API, an SDK reference is enough.

Common triggers in Flutter / RN / Capacitor apps:

| API category | Required Info.plist key | Common triggering SDKs |
|---|---|---|
| Location (when in use) | `NSLocationWhenInUseUsageDescription` | geolocator, mapbox, google maps |
| Location (always) | `NSLocationAlwaysAndWhenInUseUsageDescription` | geolocator, mapbox |
| Location (always — legacy) | `NSLocationAlwaysUsageDescription` | older SDKs |
| Camera | `NSCameraUsageDescription` | barcode_scan, image_picker, camera |
| Microphone | `NSMicrophoneUsageDescription` | audio recording, video, voice SDKs |
| Photo Library (read) | `NSPhotoLibraryUsageDescription` | image_picker (in legacy mode) |
| Photo Library (add) | `NSPhotoLibraryAddUsageDescription` | image saving |
| Contacts | `NSContactsUsageDescription` | contact picker |
| Calendar | `NSCalendarsUsageDescription` | event scheduling |
| Reminders | `NSRemindersUsageDescription` | reminder integrations |
| Motion / fitness | `NSMotionUsageDescription` | fitness, AR, step counters |
| Bluetooth (peripheral) | `NSBluetoothPeripheralUsageDescription` | iBeacon, BLE peripherals |
| Bluetooth (always) | `NSBluetoothAlwaysUsageDescription` | BLE scanning |
| Local Network | `NSLocalNetworkUsageDescription` | discovery (mDNS / Bonjour), Mapbox in some configs |
| ATT (tracking) | `NSUserTrackingUsageDescription` | ad SDKs (AdMob, Facebook), some analytics |
| Speech recognition | `NSSpeechRecognitionUsageDescription` | speech-to-text |
| Face ID | `NSFaceIDUsageDescription` | local_auth biometric |
| Health | `NSHealthShareUsageDescription`, `NSHealthUpdateUsageDescription` | HealthKit |
| Home | `NSHomeKitUsageDescription` | HomeKit accessories |

**Fix:** Add the cited key to `Info.plist` with a user-facing purpose string (clear, specific to your app, no boilerplate). Even if your app doesn't actually use the always-location API, defensively add the key:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>App uses your location to show nearby places.</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>App uses your location to show nearby places.</string>
```

Bump build number, rebuild, reupload. The warning disappears on the new build. Some old builds will still show the warning forever — that's fine, just attach a newer one to the version.

**Prevention:** When integrating any SDK in a sensitive category, defensively add *all* purpose strings for that category. See [OP-8](OPERATOR-PATTERNS.md#op-8-itms-90683-fires-for-sdk-referenced-apis-even-if-app-code-doesnt-use-them) for context.

---

### ITMS-90809 — UIWebView no longer accepted (error)

**Symptom:** Build rejected at upload time. Apple deprecated UIWebView in iOS 12 and stopped accepting submissions using it as of December 2020.

**Cause:** Your binary contains references to the `UIWebView` class. Could be in your code, but most often in a third-party SDK that hasn't been updated.

**Fix:**
1. Find the offender:
   ```bash
   nm -m path/to/Payload/Runner.app/Frameworks/<each>.framework/<each>
   ```
   Grep for `UIWebView` in the output. The framework that contains it is the offender.
2. Update the SDK to a version that uses `WKWebView`. If no such version exists, replace the SDK.
3. If the offender is your own code, replace `UIWebView` with `WKWebView`.

**Prevention:** Annual SDK audit — every iOS major release deprecates something. Check the framework binaries with `nm` if you're not sure. Add to release-prep checklist.

---

### ITMS-90338 — Non-public API usage (error)

**Symptom:** Rejection email lists specific symbols / selectors flagged as private. Common culprits: `_uuidString`, `_setKnownDevice:`, calls to undocumented frameworks.

**Cause:** Your binary references private (SPI) Apple APIs. Almost always traceable to a third-party SDK that uses unsupported tricks for analytics, A/B testing, or behavior introspection.

**Fix:**
1. Identify the calling code via the symbol Apple cited:
   ```bash
   strings path/to/Payload/Runner.app/Runner | grep <suspicious-selector>
   nm -m path/to/Frameworks/*.framework/* | grep <suspicious-selector>
   ```
2. Find which framework references the symbol. Usually one specific SDK.
3. Update the SDK to a clean version, or replace the SDK, or remove the feature.
4. If your own code, refactor to use only public APIs.

**Prevention:** Be wary of "we use private APIs for analytics" SDKs. Read SDK release notes for ITMS-90338 fixes — vendors usually publish them after Apple flags them.

---

### ITMS-90176 — Invalid signature (error)

**Symptom:** "Invalid Code Signature Identifier. The identifier... is not a valid identifier."

**Cause:** Re-signed binary, mismatched profile, corrupted resources, or build pipeline race condition where the embedded mobileprovision doesn't match the actual code signing.

**Fix:**
1. Clean archive: `flutter clean && rm -rf ios/Pods ios/Podfile.lock && cd ios && pod install`
2. Sync signing: `bundle exec fastlane match appstore --readonly` (or open Xcode and let it refetch)
3. Re-archive: `flutter build ipa --release --export-options-plist=ios/ExportOptions.plist`
4. Inspect the signing chain:
   ```bash
   codesign -dvvv path/to/Payload/Runner.app
   ```
   Authority chain should be `Apple Distribution: <Team>` → `Apple Worldwide Developer Relations Certification Authority` → `Apple Root CA`. Anything else is wrong.

**Prevention:** Don't manually re-sign. If you must, use `codesign --force --sign "<identity>" <bundle>` and keep the entitlements file matching. Better: rebuild from source.

---

### ITMS-90713 — Missing CFBundleIconFiles (error)

**Symptom:** "Missing Info.plist value. A value for the Info.plist key 'CFBundleIcons' is missing in the bundle 'X'."

**Cause:** Asset catalog missing required icon size, OR `Info.plist`'s `CFBundleIcons` key is missing/malformed, OR icons aren't being embedded into `Assets.car` for some reason.

**Fix:**
1. Confirm `CFBundleIcons` is in `Info.plist` (Xcode usually adds this automatically when using Assets.xcassets):
   ```xml
   <key>CFBundleIcons</key>
   <dict>
       <key>CFBundlePrimaryIcon</key>
       <dict>
           <key>CFBundleIconFiles</key>
           <array><string>AppIcon60x60</string></array>
           <key>CFBundleIconName</key>
           <string>AppIcon</string>
       </dict>
   </dict>
   ```
2. Verify icons exist in `Assets.car`:
   ```bash
   /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/bin/assetutil --info path/to/Assets.car | grep -A2 "AppIcon"
   ```
3. If missing the 1024 marketing icon, regenerate via `flutter_launcher_icons` (Flutter) or equivalent.

**Prevention:** Verify the 1024 marketing icon is in `Assets.car` post-build (see [OP-25](OPERATOR-PATTERNS.md#op-25-assetutil---info-reveals-if-1024×1024-marketing-icon-is-in-assetscar)).

---

### ITMS-90334 — Missing iOS 13+ launch storyboard (error)

**Symptom:** "Missing Info.plist value. A value for the Info.plist key 'UILaunchStoryboardName' is missing."

**Cause:** Project still uses old `LaunchImage` asset catalog approach. Apple required storyboard-based launch screens for all submissions starting iOS 13.

**Fix:** Add `LaunchScreen.storyboard` (most Flutter / Xcode projects already have it). Reference it in `Info.plist`:

```xml
<key>UILaunchStoryboardName</key>
<string>LaunchScreen</string>
```

If you need a minimal storyboard see [OP-7](OPERATOR-PATTERNS.md#op-7-xcode-26-ib-compiler-rejects-modern-launchscreenstoryboard-with-internal-error).

**Prevention:** Don't use `LaunchImage.imageset` exclusively. Always have a storyboard.

---

### ITMS-91053 / ITMS-91056 — Privacy manifest missing or invalid (error)

**Symptom:**
- ITMS-91053: "Missing API declaration."
- ITMS-91056: "Invalid privacy manifest."

**Cause:** App uses required-reason APIs (file timestamps, user defaults, disk space, system boot time, active keyboards) without declaring them in `PrivacyInfo.xcprivacy`. Or the manifest is malformed.

**Fix:** Create / fix `PrivacyInfo.xcprivacy` per the schema in `references/PRIVACY-MANIFEST.md`. Common pitfalls:
- File exists in source tree but not added to Xcode target → not in bundle ([OP-3](OPERATOR-PATTERNS.md#op-3-privacyinfoxcprivacy-must-be-explicitly-added-to-xcode-target-membership))
- Used UserDefaults but didn't declare it — most common miss
- SDK in your bundle doesn't ship its own manifest — upgrade SDK

**Prevention:** Validate locally before upload (`xcrun altool --validate-app`). Verify with `unzip -l <ipa> | grep PrivacyInfo` that the manifest is in the bundle.

---

### ITMS-90478 — CFBundleVersion must be incremented (error)

**Symptom:** "Redundant Binary Upload. There already exists a binary upload with build version X for train Y."

**Cause:** Re-uploading the same `CFBundleVersion` (build number) for the same `CFBundleShortVersionString` (version). Apple requires unique build numbers per version.

**Fix:** Bump build number. Flutter: `pubspec.yaml` `version: 1.0.0+2` → `1.0.0+3`. Native: `agvtool next-version -all` or `fastlane increment_build_number`. EAS: `autoIncrement: true` in `eas.json` production profile handles this automatically. Rebuild and re-upload.

**Prevention:** Make build-number bump automatic in your release lane:
```ruby
# Fastfile
lane :release do
  increment_build_number
  build_app(...)
  upload_to_app_store(...)
end
```

For Flutter projects, you can also use the `pubspec_extensions` package or write a small script that increments the `+N` suffix. For EAS, set `autoIncrement: true` in the `eas.json` production profile.

---

### ITMS-90062 + ITMS-90186 — Version train closed (error pair)

**Symptom:** Upload to ASC fails after processing with the following error pair (they almost always appear together):

```
- This bundle is invalid. The value for key CFBundleShortVersionString [2.5.0] in the
  Info.plist file must contain a higher version than that of the previously approved
  version [2.5.0]. (90062)
- Invalid Pre-Release Train. The train version '2.5.0' is closed for new build
  submissions. (90186)
```

**Cause:** The version (`CFBundleShortVersionString`, e.g. `2.5.0`) is **already approved on the App Store**. Once Apple approves a version, that "version train" closes — Apple won't accept new builds (including TestFlight-only uploads) under the same version number. The next upload must bump the version.

This catches teams who think TestFlight is independent of App Store version state. **It isn't.** TestFlight builds are attached to a version train; the train's state in the App Store ladder controls whether new builds can join it.

States that close a train:
- Version reached `READY_FOR_DISTRIBUTION` (live on the App Store)
- Version was approved and you chose Manual Release but haven't released yet (still closed for new TestFlight builds)
- Version was approved then later removed from sale (still closed — you can't reuse the number)

**Fix:** Bump `CFBundleShortVersionString` to a value strictly greater than the latest approved version, then rebuild and re-upload.

| Source of `CFBundleShortVersionString` | Edit |
|---|---|
| Native Xcode | Project → General → Identity → Version, or `agvtool new-marketing-version 2.5.1` |
| Flutter | `pubspec.yaml`: `version: 2.5.0+40` → `version: 2.5.1+1` |
| Expo | `app.config.js`: `version: "2.5.0"` → `version: "2.5.1"` |
| React Native (bare) | `ios/<App>/Info.plist`: `CFBundleShortVersionString` |

After the bump, rebuild (build number resets to 1 or auto-increments depending on tooling) and re-upload.

**Prevention:** Before kicking off a build, check the latest approved version in ASC:

```bash
# Via ASC REST API
curl -H "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appStoreVersions?filter[appStoreState]=READY_FOR_DISTRIBUTION&limit=1" \
  | jq '.data[0].attributes.versionString'
```

Or check ASC web UI → App Store → "Released" version. Compare to the version you're about to ship and bump if equal or lower.

For EAS users: this rejection is at the Apple side, not EAS. The EAS submit log shows "Something went wrong" — the actual ITMS-90062/90186 only appears at the EAS submission web URL (see OP-29). See OP-30 for the full pattern, OP-30.

---

### ITMS-90889 — Invalid document configuration (error, varied)

**Symptom:** "Invalid Document Configuration. The Info.plist contains an invalid value for the key 'X'."

**Cause:** Catch-all for malformed `Info.plist` values. Common specific cases:
- Wrong type for a key (string when array expected, etc.)
- Deprecated key still present
- Conflicting values (e.g., multiple URL scheme handlers)
- Bundle ID format violation (illegal characters)

**Fix:** Read the cited key carefully. `plutil -lint Info.plist` catches simple syntax errors but not type/value issues. Apple Developer Forums has many specific instances; search for the exact key name.

**Prevention:** When editing `Info.plist`, prefer Xcode's UI (which validates types) over hand-editing XML.

---

### Validation-time vs upload-time errors

Some ITMS codes can surface at either `xcrun altool --validate-app` or after `--upload-app`. Validation catches:
- Missing privacy manifest (91053/91056)
- Missing usage descriptions (90683 — sometimes warning, sometimes blocks)
- Invalid signature (90176)
- Missing required fields

Upload-only errors (won't catch in validation):
- Build number conflicts (90478) — needs ASC's view of existing builds
- Some signature-related issues (the IPA is fine but the chain doesn't match Apple's records)

Always run `--validate-app` first. 30-second local validation saves a 10-minute upload + processing round-trip.

## Diagnosing unfamiliar ITMS codes

1. **Apple Developer Forums** — `developer.apple.com/forums` is the most useful single resource. Search the exact code.
2. **Apple's release notes** — when major Xcode/iOS bumps occur, validation rules change. Check release notes for new ITMS codes.
3. **The full ASC error log** — the email summary is truncated; the ASC web UI under "Build Uploads" → click the row → expand warnings shows the full text.
4. **`xcrun altool` log file** — `~/Library/Logs/altool/` contains detailed transfer logs.
5. **fastlane's `report.xml`** — when using `pilot upload`, fastlane writes to `fastlane/report.xml` with full output.
6. **Apple Developer Support contact form** — `developer.apple.com/contact/` for genuinely unclear errors. 24–48h response, typically helpful.

## Catalog grow-pattern

When you hit a new ITMS code, add an entry here with cause / fix / prevention. This file should grow over time, like OPERATOR-PATTERNS.md.
