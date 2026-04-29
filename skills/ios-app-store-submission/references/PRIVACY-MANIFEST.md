# Privacy Manifest (PrivacyInfo.xcprivacy)

Required since May 1, 2024 for apps using "required reason APIs" or shipping certain third-party SDKs. Validation rejects builds that need a manifest but don't have one (ITMS-91053 / ITMS-91056).

## When required

Almost always for any non-trivial app. The trigger conditions are:

1. **App uses any "required reason API"** — even if accessed transitively through Flutter / RN / Capacitor plugins. The categories below are *very* broad:
   - File timestamp APIs (most apps that read or write files trigger this)
   - System boot time (anything using `mach_absolute_time` for elapsed-time calculation)
   - Disk space queries (caching libraries, file managers, image pipelines)
   - Active keyboards list (keyboard apps, accessibility tools)
   - User defaults (`NSUserDefaults` / `UserDefaults` — Flutter framework itself uses this)

2. **App uses tracking domains** — domains that link user data to third-party data for advertising or analytics across apps. Different from "we collect analytics" — tracking specifically means cross-app/cross-vendor data linkage. Only declare if you actually do this; most travel/utility apps don't.

3. **App ships any third-party SDK on Apple's "commonly used SDKs" list** — Firebase, Google Mobile Ads, Branch, AppsFlyer, Adjust, RevenueCat, Sentry, Mapbox, etc. Each SDK ships its own manifest; you also need an app-level one declaring what *your code* does.

For a Flutter / RN / Capacitor app of any complexity, treat the manifest as mandatory.

## File location

- **Path:** `ios/Runner/PrivacyInfo.xcprivacy` (Flutter) — or equivalent for your framework
- **Target membership:** Must be added to the main app target's Copy Bundle Resources phase
- **Verification:** `unzip -l <ipa> | grep PrivacyInfo` should show `Payload/<App>.app/PrivacyInfo.xcprivacy`. If not present, the file isn't in the bundle even if it exists in the source tree (see OP-3).

For Flutter specifically, files placed in `ios/Runner/` are *not* auto-added to the Runner target. You must edit `ios/Runner.xcodeproj/project.pbxproj` to add four entries:

1. **PBXBuildFile section:**
   ```
   <NEW_UUID_1> /* PrivacyInfo.xcprivacy in Resources */ = {isa = PBXBuildFile; fileRef = <NEW_UUID_2> /* PrivacyInfo.xcprivacy */; };
   ```

2. **PBXFileReference section:**
   ```
   <NEW_UUID_2> /* PrivacyInfo.xcprivacy */ = {isa = PBXFileReference; lastKnownFileType = text.xml; path = PrivacyInfo.xcprivacy; sourceTree = "<group>"; };
   ```

3. **PBXGroup → Runner group's `children` list:** add `<NEW_UUID_2> /* PrivacyInfo.xcprivacy */,`

4. **PBXResourcesBuildPhase → Runner's `files` list:** add `<NEW_UUID_1> /* PrivacyInfo.xcprivacy in Resources */,`

UUIDs are 24-char hex; pick prefixes that don't collide with existing IDs in the file (e.g., `ABCD1234567890ABCDEF1234`).

For RN/Capacitor: the Xcode-generated project usually has a similar setup. Plugin-shipped manifests get bundled automatically when the plugin's podspec or framework declares them as resources.

## Schema

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <!-- entries here -->
    </array>
</dict>
</plist>
```

### NSPrivacyTracking (boolean, required)

`true` if your app or its SDKs perform tracking (cross-app/cross-vendor user data linking) per Apple's [App Tracking Transparency definition](https://developer.apple.com/app-store/user-privacy-and-data-use/). Most travel/utility/productivity apps are `false`. Even apps with Firebase Analytics + Crashlytics + Performance are `false` *unless* they enable IDFA collection or Audience Network features.

### NSPrivacyTrackingDomains (array of strings)

Domains that perform tracking. Empty array if `NSPrivacyTracking=false`. If you enable tracking, list every domain (including all third-party tracking SDK endpoints).

### NSPrivacyCollectedDataTypes (array of dicts)

Each entry declares one type of data your app code collects. Distinct from the App Privacy "nutrition label" in ASC web UI — both must be filled and consistent. Schema per entry:

```xml
<dict>
    <key>NSPrivacyCollectedDataType</key>
    <string>NSPrivacyCollectedDataTypePreciseLocation</string>
    <key>NSPrivacyCollectedDataTypeLinked</key>
    <false/>
    <key>NSPrivacyCollectedDataTypeTracking</key>
    <false/>
    <key>NSPrivacyCollectedDataTypePurposes</key>
    <array>
        <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
    </array>
</dict>
```

Common data type identifiers:
- `NSPrivacyCollectedDataTypePreciseLocation`
- `NSPrivacyCollectedDataTypeCoarseLocation`
- `NSPrivacyCollectedDataTypeName`
- `NSPrivacyCollectedDataTypeEmailAddress`
- `NSPrivacyCollectedDataTypePhoneNumber`
- `NSPrivacyCollectedDataTypeUserID`
- `NSPrivacyCollectedDataTypeDeviceID`
- `NSPrivacyCollectedDataTypeProductInteraction`
- `NSPrivacyCollectedDataTypeCrashData`
- `NSPrivacyCollectedDataTypePerformanceData`
- `NSPrivacyCollectedDataTypeOtherDiagnosticData`

Common purposes:
- `NSPrivacyCollectedDataTypePurposeAppFunctionality`
- `NSPrivacyCollectedDataTypePurposeAnalytics`
- `NSPrivacyCollectedDataTypePurposeProductPersonalization`
- `NSPrivacyCollectedDataTypePurposeAdvertising`
- `NSPrivacyCollectedDataTypePurposeAppDeveloperCommunications`

Most apps can declare `NSPrivacyCollectedDataTypes` as an empty array if all data collection happens via third-party SDKs (Firebase, Mapbox, etc.) that ship their own manifests. Apple combines all manifests at review time.

### NSPrivacyAccessedAPITypes (array of dicts, the load-bearing one)

Each entry declares one required-reason API category your code uses + the specific reasons.

```xml
<dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
        <string>C617.1</string>
    </array>
</dict>
```

#### Required-reason API categories and common reason codes

Apple maintains the canonical list at [developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api). Codes change — verify before relying on this snapshot.

**`NSPrivacyAccessedAPICategoryFileTimestamp`** (`creationDate`, `modificationDate`, `getattrlist`, etc.)
- `DDA9.1` — Display file timestamps to the user
- `C617.1` — Inside app, no third-party SDK, for app's own files
- `3B52.1` — Inside app, third-party SDK functionality
- `0A2A.1` — Display app's own file timestamps to the user

**`NSPrivacyAccessedAPICategorySystemBootTime`** (`mach_absolute_time`, `kernelboottime`)
- `35F9.1` — Measure time elapsed for events
- `8FFB.1` — Calculate timeouts in user-facing flows

**`NSPrivacyAccessedAPICategoryDiskSpace`**
- `E174.1` — Display free space to user / prevent write failures
- `85F4.1` — Inside app, third-party SDK use (reporting)
- `7D9E.1` — Inside app, no third-party SDK

**`NSPrivacyAccessedAPICategoryActiveKeyboards`**
- `3EC4.1` — Custom keyboard apps providing the keyboard

**`NSPrivacyAccessedAPICategoryUserDefaults`** (`NSUserDefaults`/`UserDefaults`)
- `CA92.1` — Read/write app's own preferences
- `1C8F.1` — Inside app, third-party SDK functionality
- `C56D.1` — Inside app, no third-party SDK

#### Sensible Flutter app starter

For a Flutter app using `path_provider`, `flutter_cache_manager`, `cached_network_image`, `drift`, `connectivity_plus`, Firebase, Mapbox:

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array><string>C617.1</string></array>
    </dict>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array><string>CA92.1</string></array>
    </dict>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array><string>E174.1</string></array>
    </dict>
    <dict>
        <key>NSPrivacyAccessedAPIType</key>
        <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
        <key>NSPrivacyAccessedAPITypeReasons</key>
        <array><string>35F9.1</string></array>
    </dict>
</array>
```

These four cover ~95% of Flutter apps. SDKs (Firebase, Mapbox) ship their own manifests covering their additional usage.

## Third-party SDK signatures

Since fall 2024, Apple requires "commonly used SDKs" to ship signed manifests. The list is at [developer.apple.com/support/third-party-SDK-requirements](https://developer.apple.com/support/third-party-SDK-requirements).

How to verify a vendored SDK has its manifest + signature:

```bash
# Check the .xcframework / .framework has a PrivacyInfo.xcprivacy
find ios/Pods/<SDKName> -name "PrivacyInfo.xcprivacy"

# Check the framework signature
codesign -dv ios/Pods/<SDKName>/path/to/Framework.framework 2>&1 | grep -i "authority\|identifier"
```

If the SDK is missing a manifest or signature → upgrade to the latest version. If still missing → file an issue with the vendor and consider replacement. ASC will reject builds with any SDK missing the required manifest+signature.

Common SDK versions known to have stumbled (verify when planning upgrades):
- Older Firebase < 10.x — signed manifests added in 10.x
- Older Mapbox SDK — signature added in v11.x
- (Maintain this list as you encounter issues)

## Validation

Local validation before upload:

```bash
xcrun altool --validate-app -f build/MyApp.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

If a manifest is missing or invalid, ITMS errors surface here before the upload round-trip. Catches:
- Missing manifest entirely (ITMS-91053)
- Manifest references unknown API categories (ITMS-91056)
- Required reason API used without declaration (specific reason code listed)
- Third-party SDK missing signature

ASC web UI also generates a "Privacy Report" in the Build details page after processing — sanity-check this before submitting to App Review.

## Common gotchas

- **OP-3:** File exists in source tree but not added to Xcode target → not bundled. Always verify with `unzip -l <ipa> | grep PrivacyInfo`.
- **Forgot to declare UserDefaults:** Flutter framework uses `NSUserDefaults` internally for things like accessibility settings. Most common miss.
- **Outdated SDK without a manifest:** check the SDK's release notes; manifest support was added in specific versions.
- **Manifest only declares APIs the app code uses:** SDKs declare their own. Don't duplicate.
- **`NSPrivacyTracking=true` by accident:** tracking has a specific Apple-defined meaning (cross-app data linking). Setting `true` without actually tracking will cause incorrect ATT prompts to be required.
- **Multiple manifests merged at build time:** Xcode merges your app-level manifest with each SDK's manifest. Verify the merged result by examining `assetutil` output or by reading Apple's "Privacy Report" PDF in the ASC build details.
