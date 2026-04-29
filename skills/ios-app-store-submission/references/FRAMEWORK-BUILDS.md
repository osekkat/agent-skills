# Framework-Specific Build Steps

Phase 1 collapses each framework to a buildable Xcode workspace. After this phase, the rest of the pipeline is framework-agnostic.

## Native (Swift/SwiftUI/UIKit)

- Workspace exists if Pods or SPM-with-Pods present (`MyApp.xcworkspace`); otherwise `MyApp.xcodeproj` works directly
- `bundle exec pod install` if `Podfile` is present
- SPM-only projects: nothing to do; Xcode resolves on open
- Run `pod repo update` first time on a new machine, then `pod install`

Common issues:
- **CocoaPods version mismatch** — `gem install cocoapods` to update, or use Bundler (`bundle install`)
- **Apple Silicon arch issues** — historical, no longer needed on modern CocoaPods
- **`Podfile.lock` conflicts** — let `pod install` regenerate

### XcodeGen-managed projects (`project.yml` at repo root)

If the repo has a `project.yml` and no committed `.xcodeproj` (or the `.xcodeproj` is gitignored), it's an XcodeGen project. The `.xcodeproj` is a generated artifact — like `Cargo.lock` or `node_modules`. Treat `project.yml` as the source of truth.

**Detection:**
```bash
test -f project.yml && grep -q "^name:" project.yml && echo "XcodeGen project"
```

**Pre-build sequence:**
```bash
brew install xcodegen   # one-time
xcodegen generate       # writes <Name>.xcodeproj from project.yml
```

**Required `project.yml` keys for App Store builds** — see [OP-39](OPERATOR-PATTERNS.md#op-39-xcodegen-managed-projects-put-signing-config-in-projectyml-never-edit-the-xcodeproj). At minimum, your application target needs:

```yaml
targets:
  MyApp:
    type: application
    platform: iOS
    sources:
      - path: MyApp
    settings:
      base:
        INFOPLIST_FILE: MyApp/App/Info.plist
        PRODUCT_BUNDLE_IDENTIFIER: com.myco.myapp
        DEVELOPMENT_TEAM: <10-char Team ID>      # required for archive
        CODE_SIGN_STYLE: Automatic                # or Manual + PROVISIONING_PROFILE_SPECIFIER
        ASSETCATALOG_COMPILER_APPICON_NAME: AppIcon
```

**Workflow:** Edit `project.yml` → `xcodegen generate` → `xcodebuild archive`. Never edit the generated `.xcodeproj` — it gets blown away on the next regeneration. The "open in Xcode → Signing & Capabilities → set team" UI flow that works for non-XcodeGen projects silently doesn't persist here.

**Common issues:**
- **`error: No Accounts` on archive** — `DEVELOPMENT_TEAM` not in `project.yml`. Add it under `targets.<target>.settings.base` and re-run `xcodegen generate`.
- **`xcodegen` not found** — `brew install xcodegen`.
- **Pre-existing `.xcodeproj` checked into git but `project.yml` also exists** — the repo is in transition; ask the user which one wins. Most modern XcodeGen repos `.gitignore` the `.xcodeproj`.
- **Schemes missing after regenerate** — declare them under top-level `schemes:` in `project.yml`, not inside the target.

**Find your Team ID:** `security find-identity -v -p codesigning | grep "Apple Distribution"` — the parenthesized 10-char string (e.g. `ABCD123456`) is your Team ID. *Not* the team name.

### Tuist projects

Tuist (`Project.swift` at repo root) plays the same role as XcodeGen with a Swift DSL. Same gotcha as OP-39: signing config goes in `Project.swift` settings, not in the generated `.xcodeproj`. Use `tuist generate` instead of `xcodegen generate`. Otherwise Phase 2+ is identical.

## React Native (bare, no Expo)

```bash
# Node
nvm use  # if .nvmrc present
npm install  # or yarn install / pnpm install

# iOS native
cd ios && bundle exec pod install
```

Configuration considerations:
- **Hermes vs JSC** — set in `Podfile` via `:hermes_enabled => true`. Hermes is the modern default; faster startup, smaller binary.
- **Flipper** — disable for release builds (`:flipper_configuration => FlipperConfiguration.disabled`). Can cause submission issues if left on.
- **Metro bundling** — happens during Xcode build phase, not separately. The `.xcodeproj` runs a script that calls Metro.
- **Offline JS bundle (CI)** — for fully reproducible builds, pre-bundle:
  ```bash
  npx react-native bundle --platform ios --dev false \
    --entry-file index.js \
    --bundle-output ios/main.jsbundle \
    --assets-dest ios
  ```

Common issues:
- **`pod install` failures** — `pod repo update` first; check Ruby version (`bundle install` uses Bundler-pinned)
- **`node-gyp` failures** — Xcode CLI tools missing (`xcode-select --install`)
- **iOS native modules not linking** — `cd ios && pod install` again, then `npx react-native start --reset-cache`

## Expo (React Native + EAS)

Expo apps are React Native projects that use the Expo SDK and (typically) the EAS build pipeline. Two project shapes:

- **Managed workflow** — no `ios/` directory committed. Native code is generated on each build via `npx expo prebuild`. Most Expo apps. Cannot use fastlane locally without prebuilding first.
- **Bare workflow** — `ios/` directory committed (after running `npx expo prebuild` once). Can use fastlane OR EAS.

Detect the project shape:

```bash
ls ios/ 2>/dev/null && echo "bare" || echo "managed"
```

### Detection: is this an Expo project?

Markers:
- `app.config.js` or `app.config.ts` or `app.json` with an `expo: { ... }` key
- `package.json` has `"expo"` in dependencies
- `eas.json` exists at project root

If any of these are present, treat it as Expo and prefer EAS over raw fastlane — see `EAS-SUBMISSION.md` for the full pipeline.

### Path A: EAS Build + Submit (recommended for Expo)

```bash
eas build --platform=ios --profile=production --non-interactive --no-wait
# wait for build to finish (5–10 min)
eas submit --platform=ios --latest --non-interactive
```

EAS performs the prebuild, archive, signing, and TestFlight upload on its hosted workers. No local Xcode setup required.

Detailed configuration, polling patterns, error reading, and migration paths in `EAS-SUBMISSION.md`.

### Path B: Local fastlane (bare workflow only)

If you've ejected to bare workflow and want to use fastlane:

```bash
npm install                            # resolve JS deps
npx expo prebuild --platform ios       # regenerate native iOS code (only needed if managed)
cd ios && bundle exec pod install      # native pods
fastlane gym \
  --workspace <App>.xcworkspace \
  --scheme <App> \
  --export_method app-store
```

The workspace name comes from `app.config.js` `name` field, sanitized — e.g. `"AFCON 2025"` → `AFCON2025.xcworkspace`. Check `ls ios/*.xcworkspace`.

### Expo Info.plist generation

Don't edit `ios/<App>/Info.plist` directly — Expo regenerates it on prebuild and your edits will be lost. Configure via `app.config.js`:

```js
ios: {
  bundleIdentifier: "com.example.app",
  infoPlist: {
    ITSAppUsesNonExemptEncryption: false,
    UIBackgroundModes: ["audio", "remote-notification"],
    NSLocationWhenInUseUsageDescription: "Show nearby fan zones"
  }
}
```

These are merged into the generated `Info.plist` on each build. After the build, verify with:

```bash
unzip -p build/<App>.ipa Payload/<App>.app/Info.plist | plutil -p -
```

### Privacy manifest in Expo

Expo SDK 54+ ships `PrivacyInfo.xcprivacy` automatically with per-module privacy bundles (e.g. `ExpoApplication_privacy.bundle`, `ExpoConstants_privacy.bundle`). Verify after build:

```bash
unzip -l build/<App>.ipa | grep -i privacy
```

Should show the top-level `PrivacyInfo.xcprivacy` plus one per Expo module that needs one. If you've added third-party native deps, they bring their own manifests; if missing, ITMS-91056 fires at upload.

### Version vs build number in Expo

- `app.config.js` `version` → `CFBundleShortVersionString`. **Always local; not managed remotely.**
- `app.config.js` `ios.buildNumber` → `CFBundleVersion`. With `eas.json` `cli.appVersionSource: "remote"` + `build.production.autoIncrement: true`, EAS manages this. Otherwise local.

Bumping the version is a manual `app.config.js` edit. EAS does not auto-bump versions — only build numbers. See OP-29.

### `runtimeVersion` and OTA coupling

```js
runtimeVersion: { policy: "appVersion" }
```

This couples OTA update compatibility to the `version` field. Each `version` bump opens a new OTA channel. Implication: existing OTA updates won't apply to the new build until you publish at least one update for the new runtimeVersion. See OP-33.

### Common Expo build issues

| Symptom | Cause | Fix |
|---|---|---|
| `EAS_BUILD_ASCAPPID is not set` | Missing in eas.json | Add `submit.production.ios.ascAppId` (OP-28) |
| Build fails with "Pod install" error | Stale `ios/` from older prebuild | `rm -rf ios && npx expo prebuild --platform ios --clean` |
| Build succeeds but uploads fail with ITMS-90062/90186 | Reusing version that's already approved | Bump `version` in `app.config.js` (OP-30) |
| OTA updates don't reach existing users after bump | `runtimeVersion.policy: "appVersion"` | Publish updates targeting both old and new runtimeVersions during transition (OP-33) |
| `eas submit` can't find IPA | Build artifact expired | Rebuild — artifacts expire after 30 days (OP-32) |

## Flutter

**Recommended for first submission and TestFlight-only flows: zero-fastlane four-command pipeline** (see [OP-37](OPERATOR-PATTERNS.md#op-37-flutter-app--testflight-in-zero-fastlane-lines)).

```bash
flutter doctor          # verify environment
flutter pub get         # resolve packages
flutter build ipa --release --export-options-plist=ios/ExportOptions.plist
```

`flutter build ipa` archives, signs, and exports in one step — bypasses fastlane gym entirely.

After the IPA is on disk:

```bash
# Sanity check — bundle ID, version, build number actually baked into the IPA (OP-36)
unzip -p build/ios/ipa/<App>.ipa "Payload/Runner.app/Info.plist" | \
  plutil -p - | grep -E "CFBundleIdentifier|CFBundleVersion|CFBundleShortVersion|MinimumOSVersion"

# Validate
xcrun altool --validate-app -f build/ios/ipa/<App>.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_UUID>

# Upload
xcrun altool --upload-app -f build/ios/ipa/<App>.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_UUID>
```

The `.p8` lives at `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` so altool finds it without `--apiKeyPath`. Polling for processing → VALID happens via the ASC REST API (see PILOT-AND-TESTFLIGHT.md "Verifying state via API").

**Fastlane alternative** (use when you want lane composition, automatic build-number bumps, screenshot/metadata sync, or CI integration):

```bash
flutter build ios --release --no-codesign  # produces unsigned .app
cd ios && fastlane gym                      # archives + exports + signs
cd ios && fastlane pilot upload --ipa ../build/ios/ipa/<App>.ipa
```

Pick the fastlane path on the second or third release once the bare-bones flow has worked end-to-end at least once.

### Critical: verify your assets aren't placeholders

`flutter create` ships default assets that ship through validation but get rejected at App Review:

- **Default app icon** (`ios/Runner/Assets.xcassets/AppIcon.appiconset/`) — Flutter chevron logo, ~10KB at 1024×1024. ASC accepts it, App Review rejects under Guideline 4.0. See [OP-1](OPERATOR-PATTERNS.md#op-1-flutter-scaffold-default-icon-ships-through-validation).
- **Default launch image** (`ios/Runner/Assets.xcassets/LaunchImage.imageset/`) — 68-byte placeholder PNGs at 1x/2x/3x. Benign if your `LaunchScreen.storyboard` doesn't reference them; problematic if it does. See [OP-2](OPERATOR-PATTERNS.md#op-2-default-launchimage-warning-is-a-red-herring-if-storyboard-doesnt-reference-it).

Pre-Phase-2 verification:

```bash
# Visual check of the 1024 marketing icon
qlmanage -p ios/Runner/Assets.xcassets/AppIcon.appiconset/Icon-App-1024x1024@1x.png

# Check launch screen storyboard for image references
grep -E 'image="LaunchImage"|imageView' ios/Runner/Base.lproj/LaunchScreen.storyboard
```

If the icon is the Flutter default, regenerate:

```yaml
# pubspec.yaml — add to dev_dependencies
dev_dependencies:
  flutter_launcher_icons: ^0.14.1

# Add config block at root level
flutter_launcher_icons:
  ios: true
  android: false
  image_path: "assets/icons/app_icon.png"   # 1024×1024, no alpha, no rounded corners
  remove_alpha_ios: true
```

```bash
flutter pub get
dart run flutter_launcher_icons
```

The "No platform provided" warning in the output is benign — see [OP-9](OPERATOR-PATTERNS.md#op-9-flutter_launcher_icons-no-platform-provided-warning-is-benign).

### Source icon requirements (Apple)

- **Dimensions:** 1024×1024 px exactly. If yours is bigger, `sips -z 1024 1024 source.png --out resized.png` resizes correctly.
- **Format:** PNG, no alpha channel. Verify: `sips -g hasAlpha source.png` should show `hasAlpha: no`.
- **Color space:** sRGB. `sips -g space source.png` should show `space: RGB`.
- **No rounded corners.** iOS applies its own squircle mask — pre-rounded sources get double-rounded ([OP-10](OPERATOR-PATTERNS.md#op-10-pre-rounded-icon-designs-get-double-rounded-by-ios-mask)).

### Plugin pod install

`flutter pub get` triggers iOS plugin pod install when iOS target is present. Verify `ios/Podfile.lock` is checked into version control to ensure CI builds match local.

### Workspace vs project

Always use `Runner.xcworkspace` (not `Runner.xcodeproj`). The workspace includes the Pods project; using the bare project skips plugin native code.

```ruby
# fastlane/Gymfile
workspace "Runner.xcworkspace"
scheme "Runner"
```

### `flutter build ipa` vs `flutter build ios`

- `flutter build ios --release --no-codesign` — builds the `.app`, no archive, no IPA. Useful when fastlane gym does the archive.
- `flutter build ipa --release` — full archive + export. Produces `build/ios/ipa/<app>.ipa`. With `--export-options-plist=ios/ExportOptions.plist` for signed App Store IPA.
- `flutter build ipa --release --export-method app-store-connect` — alternative to passing the plist; uses default options.

For first submissions, `flutter build ipa --release --export-options-plist=ios/ExportOptions.plist` is the simplest path. For CI / repeat releases, prefer fastlane's `gym` for lane composition.

### Build output paths

```
build/ios/archive/Runner.xcarchive       # archive (kept for re-export)
build/ios/ipa/<App>.ipa                  # signed IPA, ready for upload
build/ios/ipa/ExportOptions.plist        # the plist Flutter passed to xcodebuild -exportArchive
build/ios/ipa/Packaging.log              # detailed signing + export log; first place to look on failure
build/ios/ipa/DistributionSummary.plist  # what was exported (signing identity, profile, entitlements) — informational
```

If the IPA doesn't appear, check `Packaging.log` first — usually a signing error in the export step (see GYM-AND-ARCHIVE.md cloud-vs-local section). `DistributionSummary.plist` is also useful for confirming which signing identity and provisioning profile were used.

The auto-generated `ExportOptions.plist` next to the IPA is what Flutter actually used — it's a useful reference if you want to commit a stable copy back into `ios/ExportOptions.plist`. Sample shipping content (Flutter 3.27, Xcode 16.x):

```xml
<plist version="1.0">
<dict>
  <key>method</key>            <string>app-store-connect</string>
  <key>teamID</key>             <string>ABCD123456</string>
  <key>signingStyle</key>       <string>automatic</string>
  <key>destination</key>        <string>export</string>
  <key>uploadSymbols</key>      <true/>
  <key>stripSwiftSymbols</key>  <true/>
  <key>compileBitcode</key>     <false/>
</dict>
</plist>
```

`method=app-store-connect` is the current name (renamed from `app-store` in Xcode 15.3 — see [OP-38](OPERATOR-PATTERNS.md#op-38-exportoptionsplist-methodapp-store-connect-is-the-current-name-app-store-is-legacy-but-still-accepted)). Both still work today.

### Bundle ID + version verification before upload

Trust the IPA, not the source. Always confirm what's actually baked in:

```bash
unzip -p build/ios/ipa/<App>.ipa "Payload/Runner.app/Info.plist" | \
  plutil -p - | \
  grep -E "CFBundleIdentifier|CFBundleVersion|CFBundleShortVersion|MinimumOSVersion"
```

Output you'd expect:
```
"CFBundleIdentifier"        => "com.example.app"
"CFBundleShortVersionString" => "1.0.0"
"CFBundleVersion"           => "3"
"MinimumOSVersion"          => "13.0"
```

This catches: scheme misconfiguration baking the test target's bundle ID into the IPA, stale version/build numbers, accidental `IPHONEOS_DEPLOYMENT_TARGET` regressions. See [OP-36](OPERATOR-PATTERNS.md#op-36-bundle-id-hides-behind-product_bundle_identifier-verify-inside-the-ipa).

### Bumping the build number

Flutter encodes both version and build in `pubspec.yaml`:

```yaml
version: 1.0.0+3   # CFBundleShortVersionString=1.0.0, CFBundleVersion=3
```

For a re-upload to TestFlight without changing the user-visible version, bump only the suffix: `1.0.0+3` → `1.0.0+4`. ASC rejects duplicate `(version, build)` pairs (ITMS-90062).

Or pass on the command line for one-off CI builds:
```bash
flutter build ipa --release --build-number=4 --build-name=1.0.0
```

## Capacitor

```bash
npm run build              # build the web app
npx cap sync ios           # copy web assets + plugin native code
cd ios/App && bundle exec pod install  # if Podfile present
```

`npx cap sync` is required after every web-side change. Forgetting it ships stale assets — debug by inspecting `ios/App/App/public/` (should match your `dist/` or `build/` output).

Workspace: `ios/App/App.xcworkspace` (note the doubled `App/App`).

Plugin install: `npm install @capacitor/<plugin>` then `npx cap sync ios`. Some plugins need extra Info.plist keys (camera, location, etc.) — check the plugin docs.

## Common dependency-install failures

**`pod install` errors:**

| Symptom | Fix |
|---|---|
| "CocoaPods could not find compatible versions" | `pod repo update` then `pod install` |
| "Unable to find a specification for X" | `pod repo update`, check Podfile spec sources |
| Ruby version mismatch | Use Bundler (`bundle install` then `bundle exec pod install`) |
| Native extension compile errors | `xcode-select --install`, verify Xcode Command Line Tools |
| `Pods.xcconfig` corrupted | `rm -rf Pods Podfile.lock && pod install` |

**`bundle install` errors:**

| Symptom | Fix |
|---|---|
| Native extension compile fails (e.g. `nokogiri`) | `gem update --system`, install Xcode CLI tools |
| Ruby version mismatch | Use `rbenv` or `asdf`; pin via `.ruby-version` |
| "permission denied" on system Ruby | Use Bundler-managed gems, never `sudo gem install` |

**`npm install` errors specific to iOS:**

| Symptom | Fix |
|---|---|
| `node-gyp` fails | `xcode-select --install` |
| Native module post-install fails | Check Xcode CLI tools, Ruby version |

**`flutter pub get` errors:**

| Symptom | Fix |
|---|---|
| Plugin resolution conflicts | Check `pubspec.lock`; usually a `flutter_X` pinned to incompatible Dart SDK |
| "Because X depends on Y..." | Read the conflict graph; usually a transitive dep needs upgrade |
| Plugin version requires newer Flutter | `flutter upgrade` or pin plugin to older version |

**Capacitor sync errors:**

| Symptom | Fix |
|---|---|
| "Cannot find directory `ios`" | `npx cap add ios` first |
| Web assets stale | Re-run `npm run build && npx cap sync ios` |
| Plugin native code missing | `npm install @capacitor/<plugin>` then `npx cap sync ios` |
