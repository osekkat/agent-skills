# Operator Patterns

Session-mined gotchas. Each entry follows:

```
## OP-N: Short title

**Symptom:** what you observed (error message, build state, ASC message)
**Root cause:** what was actually wrong
**Fix:** the minimum change that resolved it
**Prevention:** what to add to PRE-FLIGHT.md or a lane to catch this earlier
**Source:** date, project
```

Add new entries at the bottom; never renumber. When a pattern is permanently solved by tooling, keep the entry but add a "**Status:** resolved by X" line.

---

## OP-1: Flutter scaffold default icon ships through validation

**Symptom:** Build validates clean (`xcrun altool --validate-app` returns no errors), uploads to ASC clean, processes to "Ready to Submit". Apple's automated checks pass. But the App Icon visible to App Review is the default Flutter logo — pale blue chevron on white. App Review rejects under Guideline 4.0 "Design — placeholder content."

**Root cause:** `flutter create` ships `ios/Runner/Assets.xcassets/AppIcon.appiconset/` populated with default Flutter chevron PNGs at all required sizes. The build pipeline doesn't flag this as a problem — Xcode treats them as "icons present" because the asset catalog is well-formed. Flutter's own `flutter build ipa` does emit `! App icon is set to the default placeholder icon. Replace with unique icons.` but the warning is buried in stdout and the build still succeeds with exit code 0.

**Fix:** Verify the 1024×1024 source visually before any upload. Use Read tool / Preview / `qlmanage -p`. If it's the Flutter chevron, regenerate via `flutter_launcher_icons`:

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_launcher_icons: ^0.14.1

flutter_launcher_icons:
  ios: true
  android: false
  image_path: "assets/icons/app_icon.png"   # 1024×1024, no alpha
  remove_alpha_ios: true
```

Then `dart run flutter_launcher_icons` and rebuild.

**Prevention:** Before Phase 2 (Archive), MD5-hash the 1024 icon and compare against the known Flutter default hash. If it matches, fail the lane. Add to `references/FRAMEWORK-BUILDS.md` Flutter section.

**Source:** 2026-04-28, Flutter first submission.

---

## OP-2: Default LaunchImage warning is a red herring if storyboard doesn't reference it

**Symptom:** `flutter build ipa` emits `! Launch image is set to the default placeholder icon. Replace with unique launch image.` but you've replaced the launch screen with a `LaunchScreen.storyboard` that uses a solid background color (no `LaunchImage` reference). The warning persists.

**Root cause:** Flutter checks `ios/Runner/Assets.xcassets/LaunchImage.imageset/` for the default 68-byte placeholder PNGs and warns regardless of whether `LaunchScreen.storyboard` actually uses them. The placeholder PNGs sit in the asset bundle but are never displayed.

**Fix:** Ignore the warning if your storyboard is self-contained (background color + label, no `<imageView image="LaunchImage">`). Optionally, replace the 68-byte PNGs with a 1×1 transparent PNG to silence the warning.

**Prevention:** When evaluating Flutter's launch-image warning, first check `LaunchScreen.storyboard` for any `image="LaunchImage"` reference. If absent, the warning is cosmetic.

**Source:** 2026-04-28, Flutter project.

---

## OP-3: PrivacyInfo.xcprivacy must be explicitly added to Xcode target membership

**Symptom:** You drop `PrivacyInfo.xcprivacy` into `ios/Runner/`, build, upload. ASC rejects with ITMS-91053 / ITMS-91056 "Missing privacy manifest" — even though the file exists in the project directory.

**Root cause:** Files in `ios/Runner/` are not auto-added to the Runner target. Xcode requires four entries in `project.pbxproj`:
1. `PBXBuildFile` section: `<UUID> /* PrivacyInfo.xcprivacy in Resources */ = {isa = PBXBuildFile; fileRef = <UUID2> ...};`
2. `PBXFileReference` section: `<UUID2> /* PrivacyInfo.xcprivacy */ = {isa = PBXFileReference; lastKnownFileType = text.xml; path = PrivacyInfo.xcprivacy; sourceTree = "<group>";};`
3. The Runner `PBXGroup` `children` list: `<UUID2> /* PrivacyInfo.xcprivacy */`
4. The `PBXResourcesBuildPhase` for Runner `files` list: `<UUID> /* PrivacyInfo.xcprivacy in Resources */`

Without all four, the file isn't included in the bundled `.app` and ASC sees no manifest.

**Fix:** Edit `project.pbxproj` to add all four references. Use unique 24-char hex UUIDs that don't collide with existing IDs (e.g., prefix with `ABCD` if no existing ID starts with that). Verify post-edit by opening the workspace in Xcode and inspecting Runner → Build Phases → Copy Bundle Resources.

**Prevention:** After creating `PrivacyInfo.xcprivacy`, build the IPA and verify with `unzip -l <ipa> | grep PrivacyInfo` that it's actually inside the app bundle. Add to `references/PRIVACY-MANIFEST.md`.

**Source:** 2026-04-28, Flutter project.

---

## OP-4: First archive succeeds via Xcode cloud signing; subsequent archives fail

**Symptom:** First `flutter build ipa --release --export-options-plist=…` succeeds and produces a Distribution-signed IPA. Inspecting it: `Authority=Apple Distribution: <Team>` ✓. Next attempt at the same command (or `xcodebuild -exportArchive`) fails with:

```
error: exportArchive No signing certificate "iOS Distribution" found
error: exportArchive No Accounts
```

`security find-identity -v -p codesigning` shows only Apple Development certs — no Distribution cert ever appeared in the keychain. Yet the first IPA was definitely Distribution-signed. Mystery.

**Root cause:** Xcode 14+ supports "cloud signing" — Apple's servers hold the private key and sign on demand when Xcode is logged in to the Apple ID. The first archive used a transient cloud-signing session. Between archives, the session can lapse (24-hour token, sign-out, network blip), and the CLI `xcodebuild` finds no local cert and no logged-in Apple ID accessible from CLI context.

**Fix:** Generate a real local Distribution cert, one-time:
1. `open -a Xcode`
2. **Xcode → Settings (⌘,) → Accounts**
3. Sign in with the team's Apple ID if not already
4. Select the account → **Manage Certificates…** (lower right)
5. **+** in lower left → **Apple Distribution**
6. Cert is created locally and stored in the login keychain

Verify: `security find-identity -v -p codesigning` should now show `Apple Distribution: <Team> (<TeamID>)` as a third identity.

**Prevention:** Phase 0 pre-flight should explicitly check for `Apple Distribution` in `security find-identity -v -p codesigning` output and refuse to start Phase 2 if missing. Don't trust that cloud signing will work twice in a row.

**Source:** 2026-04-28, Flutter project.

---

## OP-5: Cloud signing requires Admin/Account Holder API key; "App Manager" is insufficient

**Symptom:** Pass an ASC API key to `xcodebuild -exportArchive` via `-authenticationKeyPath/-authenticationKeyID/-authenticationKeyIssuerID`. Export fails with:

```
error: exportArchive Cloud signing permission error
error: exportArchive No signing certificate "iOS Distribution" found
```

(Repeats many times due to xcodebuild's retry loop.) Yet `xcrun altool --validate-app` and `--upload-app` work fine with the same key — the upload-side API permissions are clearly OK.

**Root cause:** Cloud signing operations (creating/managing certs and provisioning profiles via API) require **Admin** or **Account Holder** roles. **App Manager** is sufficient for upload, build management, TestFlight, and most metadata APIs — but not for signing operations. Apple does not document this clearly.

**Fix:** Either:
- Generate a second API key with **Admin** role specifically for signing operations (kept separate from the upload key for blast-radius hygiene), OR
- Use a local Distribution cert (OP-4) and skip cloud signing entirely.

**Prevention:** When generating ASC API keys for fastlane/match cloud signing, default to **Admin**. For pure upload pipelines, **App Manager** is fine. Document the role requirement next to the API key generation step.

**Source:** 2026-04-28, Flutter project.

---

## OP-6: New local Distribution cert orphans the existing auto-managed provisioning profile

**Symptom:** After OP-4 fix (creating local Apple Distribution cert), retry `xcodebuild -exportArchive`. New error:

```
error: exportArchive Provisioning profile "iOS Team Store Provisioning Profile: com.example.app"
       doesn't include signing certificate "Apple Distribution: <Team> (TEAMID)".
```

**Root cause:** With automatic signing, Xcode previously created an auto-managed App Store provisioning profile bound to the *cloud-signed* Distribution cert. Your new local cert has a different serial number → the existing profile doesn't include it. Apple's auto-management can regenerate the profile to include the new cert, but only if `-allowProvisioningUpdates` is set AND Xcode has a logged-in Apple ID accessible from CLI context.

**Fix:** Run the export with `-allowProvisioningUpdates` and *without* the API key (let Xcode use the UI-logged-in account):

```
xcodebuild -exportArchive \
  -archivePath build/ios/archive/Runner.xcarchive \
  -exportOptionsPlist ios/ExportOptions.plist \
  -exportPath build/ios/ipa-new \
  -allowProvisioningUpdates
```

Xcode will regenerate the provisioning profile to bind the new cert. Subsequent archives reuse the new profile without intervention.

**Prevention:** When introducing a new Distribution cert into a project that previously used cloud signing, expect the provisioning profile to need regeneration. Always pair `-allowProvisioningUpdates` with a logged-in Xcode UI account on the first export after a cert change.

**Source:** 2026-04-28, Flutter project.

---

## OP-7: Xcode 26 IB compiler rejects modern LaunchScreen.storyboard with "Internal error"

**Symptom:** Custom `LaunchScreen.storyboard` written with modern attributes (`useTraitCollections="YES"`, `useSafeAreas="YES"`, `<device id="retina6_12">`, recent `toolsVersion`/`systemVersion`, label with autolayout constraints) causes `flutter build ipa` to fail at archive step with:

```
Error (Xcode): Internal error. Please file a bug at feedbackassistant.apple.com
and attach "/var/folders/.../IB-agent-diagnostics_<timestamp>".
/path/to/LaunchScreen.storyboard
Encountered error while archiving for device.
```

The IB diagnostics bundle contains internal stack traces from Xcode's Interface Builder compiler.

**Root cause:** Xcode 26's IB compiler is stricter than older versions about storyboard attribute combinations. The legacy Flutter scaffold storyboard uses `toolsVersion="12121" systemVersion="16G29"` (an ancient format) that compiles reliably. Modern attributes can trip internal asserts in specific combinations — not always reproducible, often device-tag-related.

**Fix:** Use the minimal legacy format. Replace your storyboard with:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<document type="com.apple.InterfaceBuilder3.CocoaTouch.Storyboard.XIB" version="3.0"
          toolsVersion="12121" systemVersion="16G29" targetRuntime="iOS.CocoaTouch"
          propertyAccessControl="none" useAutolayout="YES" launchScreen="YES"
          colorMatched="YES" initialViewController="01J-lp-oVM">
    <dependencies>
        <deployment identifier="iOS"/>
        <plugIn identifier="com.apple.InterfaceBuilder.IBCocoaTouchPlugin" version="12089"/>
    </dependencies>
    <scenes>
        <scene sceneID="EHf-IW-A2E">
            <objects>
                <viewController id="01J-lp-oVM" sceneMemberID="viewController">
                    <layoutGuides>
                        <viewControllerLayoutGuide type="top" id="Ydg-fD-yQy"/>
                        <viewControllerLayoutGuide type="bottom" id="xbc-2k-c8Z"/>
                    </layoutGuides>
                    <view key="view" contentMode="scaleToFill" id="Ze5-6b-2t3">
                        <autoresizingMask key="autoresizingMask" widthSizable="YES" heightSizable="YES"/>
                        <color key="backgroundColor" red="0.78" green="0.32" blue="0.23" alpha="1"
                               colorSpace="custom" customColorSpace="sRGB"/>
                    </view>
                </viewController>
                <placeholder placeholderIdentifier="IBFirstResponder" id="iYj-Kq-Ea1"
                             userLabel="First Responder" sceneMemberID="firstResponder"/>
            </objects>
            <point key="canvasLocation" x="53" y="375"/>
        </scene>
    </scenes>
</document>
```

Solid background color is App-Review-safe and compiles reliably across Xcode versions. If you need a label, design it in Xcode UI (which emits a known-good combination of attributes) rather than hand-writing.

**Prevention:** When hand-writing launch screens for cross-Xcode compatibility, stay in the legacy format (`toolsVersion="12121"`, no `<device>` tag, no `useTraitCollections`). For richer designs, open Xcode and edit the storyboard in IB so the format matches the current Xcode's expectations.

**Source:** 2026-04-28, Flutter project.

---

## OP-8: ITMS-90683 fires for SDK-referenced APIs even if app code doesn't use them

**Symptom:** Build uploads cleanly, ASC processing completes, build shows "Complete" with green checkmark + yellow ⚠️ triangle. Clicking the warning reveals:

> 90683: Missing purpose string in Info.plist. Your app's code references one or more APIs that access sensitive user data, or the app has one or more entitlements that permit such access. The Info.plist file for the "Runner.app" bundle should contain a NSLocationAlwaysAndWhenInUseUsageDescription key with a user-facing purpose string explaining clearly and completely why your app needs the data. If you're using external libraries or SDKs, they may reference APIs that require a purpose string. While your app might not use these APIs, a purpose string is still required.

Your app only uses *when in use* location, never *always*. You haven't called any always-location API.

**Root cause:** A linked SDK references the `kCLAuthorizationStatusAuthorizedAlways` symbol or an equivalent always-API. Apple's binary scanner sees the symbol and flags it. Common offenders:
- `geolocator` (Flutter)
- `mapbox_maps_flutter` (Flutter)
- `react-native-geolocation-service` (RN)
- Various ad / analytics SDKs

The warning is non-blocking — TestFlight builds still process — but App Reviewers may cite it.

**Fix:** Add the always-and-when-in-use purpose string to `Info.plist` with the same text as your when-in-use string:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>App uses your location to show nearby places.</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>App uses your location to show nearby places.</string>
```

Rebuild + reupload (bump build number). The warning disappears.

**Prevention:** When integrating any SDK that touches a sensitive API category, defensively add *all* purpose strings for that category, not just the ones your code calls. Common categories: location (always + whenInUse + temporaryUsage), camera, microphone, photo library (read + add), contacts, calendar, reminders, motion, Bluetooth, local network, tracking (ATT). Add to `references/ITMS-ERRORS.md`.

**Source:** 2026-04-28, Flutter project.

---

## OP-9: flutter_launcher_icons "No platform provided" warning is benign

**Symptom:** Running `dart run flutter_launcher_icons` outputs:

```
════════════════════════════════════════════
   FLUTTER LAUNCHER ICONS (v0.14.4)
════════════════════════════════════════════

• Overwriting default iOS launcher icon with new icon
No platform provided

✓ Successfully generated launcher icons
```

Looks like a warning saying nothing happened. But the icons were actually replaced.

**Root cause:** The "No platform provided" message is misleading. When you set only `ios: true` in pubspec config, the package processes iOS but emits this confusing line for the (absent) Android target. The "Overwriting default iOS launcher icon" message above is the actual confirmation iOS icons were generated.

**Fix:** Ignore the message. Verify by checking the MD5 of `ios/Runner/Assets.xcassets/AppIcon.appiconset/Icon-App-1024x1024@1x.png` before and after — it changes when generation succeeds.

**Prevention:** Add a post-generation hash check to the lane:
```bash
md5 ios/Runner/Assets.xcassets/AppIcon.appiconset/Icon-App-1024x1024@1x.png
```
and visually inspect the 1024 icon (e.g. via `qlmanage -p`).

**Source:** 2026-04-28, Flutter project.

---

## OP-10: Pre-rounded icon designs get double-rounded by iOS mask

**Symptom:** Your designer ships a 1024×1024 icon with rounded corners baked into the artwork (squircle shape on a flat background). When the icon renders on a real iPhone home screen, the corners look tighter / more aggressive than the design intent.

**Root cause:** iOS automatically applies its own squircle mask when rendering app icons. If your source already has rounded corners, the mask cuts inside them again, producing a smaller-radius squircle than the designer drew.

**Fix:** For best results, the 1024 source should be **square edge-to-edge**, with the artwork extending to all four corners (or padded with a flat color that matches the visible content). iOS does the rounding.

If you're stuck with a pre-rounded source and can't redesign:
- Confirm the corners are filled with a solid color (no alpha, no white background that would cut weird)
- Accept the slight double-rounding — usually still ships fine

**Prevention:** Specify "edge-to-edge artwork, no rounded corners, no alpha" when briefing the designer. Reference Apple's [Human Interface Guidelines: App Icons](https://developer.apple.com/design/human-interface-guidelines/app-icons).

**Source:** 2026-04-28, Flutter project.

---

## OP-11: ASC build placeholder icon during processing is normal

**Symptom:** Upload completes, build appears in TestFlight → Builds with status "Processing". The icon next to the build row is a generic gray placeholder (squircle outline with grid pattern, X corners) — not your app icon. The header "App Name" icon at top of the page also shows the placeholder. You panic that something went wrong with the icon upload.

**Root cause:** ASC extracts the 1024×1024 marketing icon from the build's `Assets.car` during processing. The placeholder is shown until extraction completes. Once status flips to "Ready to Submit" or "Complete", the real icon appears.

**Fix:** Wait. Verify before uploading that `Assets.car` contains the 1024 marketing icon:

```bash
unzip -p path/to.ipa Payload/Runner.app/Assets.car > /tmp/Assets.car
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/bin/assetutil --info /tmp/Assets.car | grep -A2 '"1024'
```

You should see `"1024x1024 index:11 idiom:marketing"`.

**Prevention:** Don't escalate when the placeholder appears during processing. Check `assetutil` first.

**Source:** 2026-04-28, Flutter project.

---

## OP-12: "Complete" with ⚠️ in Build Uploads is non-blocking but worth fixing

**Symptom:** ASC TestFlight → Build Uploads row shows the build status as "Complete" with green ✓ next to a yellow ⚠️ triangle. Clicking opens a dialog with ITMS warnings (e.g., ITMS-90683). The build is still processed and submittable.

**Root cause:** Apple distinguishes between *errors* (block processing) and *warnings* (don't block). A green check + warning triangle means upload succeeded but reviewers may flag the warnings.

**Fix:** Read each cited ITMS code. For each:
- If clearly fixable (e.g., add Info.plist purpose string for ITMS-90683), bump build number and reupload before submitting to App Review.
- If genuinely non-applicable (rare), submit and document in review notes.

**Prevention:** Always click the warning triangle and address each warning before submitting to App Review. The 10-minute round-trip of fixing locally is much shorter than a review rejection round-trip (1–2 days).

**Source:** 2026-04-28, Flutter project.

---

## OP-13: "Ready to Submit" ≠ "Submitted to App Review"

**Symptom:** User says "submission is complete" because they see TestFlight build status "Ready to Submit" with green checkmark. They expect the app to be in Apple's review queue.

**Root cause:** ASC's status language is misleading. "Ready to Submit" means the *build* is processed and can be attached to a *version* for submission. The actual submission is a separate manual click ("Add for Review" → confirm). Until that click, the build is just sitting in TestFlight.

**Fix:** Verify state via API or UI:
- API: `GET /v1/reviewSubmissions?filter[app]={appId}&filter[platform]=IOS` — `(no review submissions found)` means not yet submitted.
- UI: ASC → Distribution tab → version 1.0.0 — top-right button label is "Add for Review" (not yet submitted) or "Cancel Review" (in review).

**Prevention:** When walking users through the pipeline, use precise language: "build is ready" vs "submission queued for review" vs "in review" vs "approved". The skill router already distinguishes Phase 5 (TestFlight verification) from Phase 7 (Submit for Review) — keep them mentally distinct.

**Source:** 2026-04-28, Flutter project.

---

## OP-14: primaryLocale=fr-FR forces French localization to be filled

**Symptom:** App was created in ASC with primary locale French. You drafted English copy. When you POST en-US localization via API and leave fr-FR alone, the version stays in PREPARE_FOR_SUBMISSION but ASC web UI flags fr-FR fields as required.

**Root cause:** Apple requires the *primary* locale to have all required fields filled, regardless of what other locales you've added. The primary locale is what users see when no specific localization matches their region. If primary is fr-FR and fr-FR has no description/keywords, submission is blocked.

**Fix:** Two options:
- **Quick:** Populate fr-FR with the English copy as a fallback (Apple won't reject for "wrong language" — they review for accuracy/policy compliance). Replace with French translations later.
- **Proper:** Change `primaryLocale` on the App via `PATCH /v1/apps/{id}` with `attributes.primaryLocale = "en-US"`. Note: this can only be changed when no version is in review.

**Prevention:** When creating the App record in ASC, set primary locale to your most-supported language (usually English). If the app was created with a different primary, fix it before drafting metadata.

**Source:** 2026-04-28, Flutter project (created with fr-FR primary).

---

## OP-15: ASC sometimes auto-creates en-US AppInfoLocalization

**Symptom:** First GET on `/v1/appInfos/{id}/appInfoLocalizations` shows only fr-FR. POST to create en-US returns:

```
HTTP 409 ENTITY_ERROR.ATTRIBUTE.INVALID.DUPLICATE
"An 'appInfoLocalizations' with a 'locale' of 'en-US' already exists."
```

Subsequent GET shows both fr-FR and en-US. So Apple created en-US between your two requests.

**Root cause:** ASC has internal triggers that lazily create localizations (e.g., when the App Store version is touched). Race condition between your discovery and creation.

**Fix:** Always re-GET before POST. If en-US now exists, PATCH it instead. Wrap in a get-or-create pattern:

```python
existing = {l["attributes"]["locale"]: l["id"] for l in current_locs["data"]}
if locale in existing:
    PATCH(f"/v1/appInfoLocalizations/{existing[locale]}", attrs)
else:
    POST("/v1/appInfoLocalizations", {..., "locale": locale, ...})
```

**Prevention:** Treat ASC localization creation as racy. Always re-discover state immediately before mutating.

**Source:** 2026-04-28, Flutter project.

---

## OP-16: whatsNew is not editable on first submission

**Symptom:** PATCH `/v1/appStoreVersionLocalizations/{id}` with `attributes.whatsNew = "..."` returns:

```
HTTP 409 STATE_ERROR
"Attribute 'whatsNew' cannot be edited at this time"
```

**Root cause:** `whatsNew` is the "What's New in This Version" field shown on App Store updates. For first submissions, there's nothing "new" — the field doesn't apply. Apple makes it read-only until the next version.

**Fix:** Omit `whatsNew` from your PATCH attributes for first submissions. Include it only when submitting subsequent versions (UPDATE mode).

**Prevention:** In your localization-fill code, conditionally include `whatsNew` based on whether this is `first-submission` vs `update` mode.

**Source:** 2026-04-28, Flutter project.

---

## OP-17: Age rating questionnaire fields expand annually

**Symptom:** PATCH `/v1/ageRatingDeclarations/{id}` with what you think is a complete attribute set fails with HTTP 409 listing many `ENTITY_ERROR.ATTRIBUTE.REQUIRED` errors. After fixing those, more required fields appear. Iterating reveals fields like `ageAssurance`, `lootBox`, `messagingAndChat`, `parentalControls`, `advertising`, `gunsOrOtherWeapons`, `healthOrWellnessTopics` — none of which were in the schema a year ago.

**Root cause:** Apple expands the age rating questionnaire periodically (multiple times per year now). New fields are required on existing endpoints without API version bumps. Older code that worked breaks silently.

**Fix:** As of 2026-04, the full known field set is:

```python
attrs = {
    # Override
    "ageRatingOverride": "NONE",
    # Content scale: NONE / INFREQUENT_OR_MILD / FREQUENT_OR_INTENSE
    "alcoholTobaccoOrDrugUseOrReferences": "NONE",
    "contests": "NONE",
    "gamblingSimulated": "NONE",
    "gunsOrOtherWeapons": "NONE",
    "horrorOrFearThemes": "NONE",
    "matureOrSuggestiveThemes": "NONE",
    "medicalOrTreatmentInformation": "NONE",
    "profanityOrCrudeHumor": "NONE",
    "sexualContentGraphicAndNudity": "NONE",
    "sexualContentOrNudity": "NONE",
    "violenceCartoonOrFantasy": "NONE",
    "violenceRealistic": "NONE",
    "violenceRealisticProlongedGraphicOrSadistic": "NONE",
    # Booleans
    "advertising": False,
    "ageAssurance": False,
    "gambling": False,
    "healthOrWellnessTopics": False,
    "lootBox": False,
    "messagingAndChat": False,
    "parentalControls": False,
    "unrestrictedWebAccess": False,
    "userGeneratedContent": False,
}
```

**Removed fields** (will return `ENTITY_ERROR.ATTRIBUTE.UNKNOWN` if you include them):
- `seventeenPlus`
- `loginRequired`
- `kidsAgeBand` — still settable but optional, send `None` to omit

**Prevention:** Don't hardcode the field list. Iterate against the API: PATCH with what you think is correct, parse `ENTITY_ERROR.ATTRIBUTE.REQUIRED` errors, add the missing fields with sensible defaults, retry. Better: skip API for age rating and use the ASC web UI questionnaire (5 click-through) which always has the current schema.

**Source:** 2026-04-28, Flutter project.

---

## OP-18: ageRatingDeclarations forbids GET; required fields discoverable only via PATCH errors

**Symptom:** `GET /v1/ageRatingDeclarations/{id}` returns:

```
HTTP 403 FORBIDDEN_ERROR
"The resource 'ageRatingDeclarations' does not allow 'GET_INSTANCE'.
 Allowed operation is: UPDATE"
```

You cannot inspect what fields exist or what their current values are. You can only blind-PATCH and read errors.

**Root cause:** Apple's API has many resources with restricted operation sets. `ageRatingDeclarations` only supports UPDATE. Designed for the ASC UI, not third-party introspection.

**Fix:** Discovery via error iteration is the only path:
1. PATCH with empty attributes
2. Read `ENTITY_ERROR.ATTRIBUTE.REQUIRED` errors
3. Add each missing field with a default (`"NONE"` for content scales, `False` for booleans)
4. Repeat until HTTP 200

Alternatively, fetch the parent `appInfos/{id}/ageRatingDeclaration` (GET on the relationship) — sometimes returns current values via that path.

**Prevention:** Maintain a local snapshot of the current age rating field set in this skill (see OP-17 for the snapshot). Update when Apple expands.

**Source:** 2026-04-28, Flutter project.

---

## OP-19: ASC normalizes versionString "1.0.0" to "1.0"

**Symptom:** PATCH `/v1/appStoreVersions/{id}` with `attributes.versionString = "1.0.0"`. Subsequent GET shows `versionString: "1.0"`. The trailing zero was stripped.

**Root cause:** ASC normalizes version strings by removing trailing zeros. Cosmetic only — the build number (CFBundleVersion) inside the IPA stays at "2" or whatever, the App Store user sees "1.0" not "1.0.0".

**Fix:** Either accept the normalization (most apps display as "1.0", "2.1" not "1.0.0", "2.1.0") OR force "1.0.0" by re-PATCHing — sometimes sticks if you set it explicitly without other field changes.

**Prevention:** Use 2-component versions (`1.0`, `1.1`, `2.0`) for the App Store-facing version string. Use 3-component versions internally if needed (Flutter's `version: 1.0.0+1` maps cleanly: 1.0.0 → CFBundleShortVersionString, 1 → CFBundleVersion).

**Source:** 2026-04-28, Flutter project.

---

## OP-20: ASC API silently rejects sort param on some endpoints

**Symptom:** `GET /v1/apps/{id}/appStoreVersions?limit=5&sort=-createdDate` returns HTTP 400 with:

```
PARAMETER_ERROR.ILLEGAL
"The parameter 'sort' can not be used with this request"
```

The same `sort=` works on `/v1/builds`. Inconsistent.

**Root cause:** Each ASC endpoint has its own allowed parameter set. Sort is supported on some (builds, betaTesters) but not others (appStoreVersions, reviewSubmissions). Apple's API docs don't always reflect this.

**Fix:** Drop the sort param on appStoreVersions and reviewSubmissions. Default ordering is usually newest-first.

**Prevention:** When building API helpers, parameterize sort and gate it per-endpoint. Or just always omit and re-sort client-side from the response data.

**Source:** 2026-04-28, Flutter project.

---

## OP-21: APP_IPHONE_69 doesn't exist; iPhone 16 Pro Max screenshots use APP_IPHONE_67

**Symptom:** POST `/v1/appScreenshotSets` with `screenshotDisplayType: "APP_IPHONE_69"` returns HTTP 409:

```
'APP_IPHONE_69' is not a valid value for the attribute 'screenshotDisplayType'
```

**Root cause:** Apple's `screenshotDisplayType` enum tops out at `APP_IPHONE_67` for the largest iPhone bucket, even though physical iPhone 16 Pro Max is 6.9" and the simulator output is 1320×2868. The "6.7-inch Display" bucket accepts both legacy 6.7" devices (iPhone 14 Pro Max, 1290×2796) and the newer 6.9" (1320×2868).

**Fix:** Use `APP_IPHONE_67` for any modern Pro Max screenshot. The pixel dimensions just need to be one of the accepted sizes for that bucket: 1290×2796, 1320×2868, etc.

**Prevention:** Cache the known good `screenshotDisplayType` enum locally. Don't infer from physical device size.

Known iOS screenshot display types (verify periodically — Apple drops deprecated ones):
- `APP_IPHONE_67` — 6.7" / 6.9" (modern Pro Max bucket)
- `APP_IPHONE_65` — 6.5" (older, X/11 Pro Max)
- `APP_IPHONE_61` — 6.1"
- `APP_IPHONE_58` — 5.8"
- `APP_IPHONE_55` — 5.5" (legacy)
- `APP_IPHONE_47` — 4.7" (legacy)
- `APP_IPAD_PRO_3GEN_129` — 12.9" iPad Pro
- `APP_IPAD_PRO_3GEN_11` — 11" iPad Pro

**Source:** 2026-04-28, Flutter project.

---

## OP-22: Screenshot upload is a 3-step protocol, not a single POST

**Symptom:** You expect `POST /v1/appScreenshots` with multipart body containing the image. Apple returns a JSON response with no image stored. Subsequent GET shows the screenshot as "not uploaded".

**Root cause:** Apple uses an out-of-band upload pattern with three steps:

1. **Reserve:** `POST /v1/appScreenshots` with `attributes: {fileName, fileSize}` and the `appScreenshotSet` relationship. Response includes `data.attributes.uploadOperations`: an array of objects with `{method, url, length, offset, requestHeaders}`.
2. **Upload chunks:** For each operation, raw HTTP `PUT` (or specified method) to the `url` with the file bytes from `offset` for `length` bytes, and the headers from `requestHeaders`. The URL is presigned (S3 or CloudFront) — do NOT add the JWT Authorization header.
3. **Commit:** `PATCH /v1/appScreenshots/{id}` with `attributes: {uploaded: true, sourceFileChecksum: <md5>}`. Without this, the screenshot is "reserved but not finalized" and won't show up.

Files <50MB usually return a single upload operation; larger files may chunk across multiple operations.

**Fix:** Implement all three steps. Critical details:
- The presigned upload URLs do NOT take the ASC JWT — strip the `Authorization` header when PUTing.
- The `requestHeaders` array contains things like `Content-Type: image/png` — apply them.
- `sourceFileChecksum` is the MD5 hex of the *full* file bytes.

**Prevention:** Wrap the 3-step in a helper and document the gotcha clearly. Add to `references/DELIVER-AND-METADATA.md` as the canonical screenshot upload pattern.

**Source:** 2026-04-28, Flutter project.

---

## OP-23: ASC API JWT requires raw r||s signature, not DER

**Symptom:** You sign the JWT with `cryptography.hazmat.primitives.asymmetric.ec.sign(msg, ec.ECDSA(hashes.SHA256()))`. ASC returns 401 Unauthorized, even though the key/issuer/expiration are correct.

**Root cause:** `cryptography` returns ECDSA signatures in DER format (variable length, ASN.1-encoded). JWT spec (RFC 7515) requires the raw concatenation `r || s`, each padded to 32 bytes (for P-256). DER signatures fail signature verification on Apple's side.

**Fix:** Convert DER to raw:

```python
from cryptography.hazmat.primitives.asymmetric import utils as asym_utils
der_sig = key.sign(msg, ec.ECDSA(hashes.SHA256()))
r, s = asym_utils.decode_dss_signature(der_sig)
raw_sig = r.to_bytes(32, "big") + s.to_bytes(32, "big")
jwt_token = h_b + "." + p_b + "." + b64url(raw_sig)
```

If using PyJWT (`pyjwt[crypto]`), it handles this internally — pass the .p8 contents and let it sign.

**Prevention:** Prefer PyJWT for JWT signing; only fall back to manual signing when PyJWT isn't available. Document the raw-vs-DER trap loudly.

**Source:** 2026-04-28, Flutter project.

---

## OP-24: ITSAppUsesNonExemptEncryption=false in Info.plist pre-answers ASC encryption questionnaire

**Symptom:** Every TestFlight build prompts you to answer the export compliance questionnaire ("Does your app use encryption?") in ASC before it can be distributed externally. Annoying for solo dev iterating on builds.

**Root cause:** Apple requires every binary to declare its encryption use. The ASC web UI questionnaire is the default mechanism. But you can pre-answer in `Info.plist` for the trivial case (HTTPS only, no custom crypto).

**Fix:** Add to `ios/Runner/Info.plist`:

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

Now ASC stops prompting. Builds go straight from Processing → Ready to Submit.

**Caveats:**
- Only valid if your app uses *exempt* encryption: standard HTTPS, Apple's own CryptoKit/CommonCrypto for trivial uses, signing/verification of your own data.
- *Non-exempt* encryption (custom protocols, end-to-end encrypted messaging, novel crypto): you must answer the questionnaire and may need to file annual self-classification reports with the U.S. Bureau of Industry and Security (BIS).

**Prevention:** Add this to every project's `Info.plist` early. Add to PRE-FLIGHT.md checklist.

**Source:** 2026-04-28, Flutter project.

---

## OP-25: assetutil --info reveals if 1024×1024 marketing icon is in Assets.car

**Symptom:** You're not sure whether the 1024 marketing icon made it into the IPA. Looking at the `.app` bundle directly only shows `AppIcon60x60@2x.png` and `AppIcon76x76@2x~ipad.png` — the small runtime icons. The 1024 isn't a loose file.

**Root cause:** The 1024×1024 marketing icon (used by App Store / TestFlight thumbnails, never displayed on-device) is compiled into `Assets.car`, the binary asset catalog format. It's not a separate PNG.

**Fix:** Use Apple's `assetutil` to inspect:

```bash
unzip -p path/to.ipa Payload/Runner.app/Assets.car > /tmp/Assets.car
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/bin/assetutil --info /tmp/Assets.car | grep -A2 '"1024'
```

Expected output:
```
    "Sizes" : [
      "1024x1024 index:11 idiom:marketing"
    ],
```

Note: `assetutil` lives under `Platforms/iPhoneOS.platform/usr/bin/`, not the standard Xcode CLI bin path — invoke with full path.

**Prevention:** Add `assetutil --info` check to a post-build verification step. Failing to ship the 1024 marketing icon results in App Store listing showing a placeholder forever.

**Source:** 2026-04-28, Flutter project.

---

## OP-26: Distribution Cert "App Manager" API key role unable to revoke profiles either

**Symptom:** When trying to clean up old auto-managed profiles via API to force regeneration with new cert (alternative path for OP-6), the API call returns 403 even with `-allowProvisioningUpdates`.

**Root cause:** Profile management (create / delete) is also restricted to Admin / Account Holder API key roles. Same restriction as cloud signing.

**Fix:** Don't try to manage profiles via App Manager API key. Either:
- Use Xcode UI with logged-in account (UI flows have different permissions than API)
- Generate a separate Admin API key for profile/cert management
- Manually delete old profiles in developer.apple.com → Certificates, Identifiers & Profiles

**Prevention:** Document API key role requirements explicitly: App Manager for upload/metadata, Admin for cert/profile/cloud signing.

**Source:** 2026-04-28, Flutter project (inferred from OP-5 pattern).

---

## OP-27: First archive that succeeds with cloud signing produces "iOS Team Store Provisioning Profile" name

**Symptom:** When inspecting the embedded provisioning profile of the IPA produced by Xcode 14+ cloud signing, the profile `Name` is `"iOS Team Store Provisioning Profile: com.example.app"`. Looks suspicious — "Team Store" sounds like it might be a development profile, not App Store.

**Root cause:** It's actually an App Store profile. Apple's automatic-managed App Store profiles are named with the "Team Store" prefix when generated by Xcode cloud signing. Confirmed by inspecting:
- `get-task-allow: false` (App Store characteristic)
- Authority chain via `codesign -dvvv` shows `Apple Distribution: <Team>` (production cert)

**Fix:** None — the profile is correct. Don't waste time chasing the suspicious name.

**Prevention:** When auditing IPA signing, trust `codesign -dvvv` over the profile display name. Specifically: `Authority=Apple Distribution: <Team>` confirms App Store distribution; `get-task-allow: false` confirms production.

```bash
unzip -q path/to.ipa -d /tmp/check
codesign -dvvv /tmp/check/Payload/Runner.app | grep -E "Authority|Identifier="
unzip -p path/to.ipa Payload/Runner.app/embedded.mobileprovision | security cms -D | grep -A1 "get-task-allow\|Name"
```

**Source:** 2026-04-28, Flutter project.

---

## OP-28: `eas submit --non-interactive` requires `ascAppId` in eas.json

**Symptom:** Running `eas submit --platform=ios --latest --non-interactive` fails immediately with:

```
Set ascAppId in the submit profile (eas.json) or re-run this command in interactive mode.
Submission failed
    Error: submit command failed.
```

The CLI doesn't even attempt to upload — it bails before contacting Apple. Interactive mode (no `--non-interactive`) prompts you to pick the ASC app from a list.

**Root cause:** Non-interactive mode refuses to discover the App Store Connect app ID via API lookup; it requires the numeric ID to be hard-pinned in `eas.json`. EAS won't guess it from the bundle ID even though it could (the lookup involves hitting `GET /v1/apps?filter[bundleId]=...` which works fine in interactive mode).

**Fix:** Find the ASC App ID in your App Store Connect URL — it's the number in `https://appstoreconnect.apple.com/apps/<NUMBER>/...`. Add it to `eas.json`:

```json
"submit": {
  "production": {
    "ios": {
      "ascAppId": "1234567890"
    }
  }
}
```

Re-run `eas submit --platform=ios --latest --non-interactive`. Now succeeds.

**Prevention:** When running `eas submit:configure` or after first successful interactive submit, write the discovered `ascAppId` back to `eas.json` immediately so subsequent CI runs work non-interactively. Treat the missing-`ascAppId` failure as a Phase 0 pre-flight bug, not a runtime issue.

**Source:** 2026-04-29, Expo / EAS update submission.

---

## OP-29: EAS submit Apple-side errors are opaque locally; only visible at submission web URL

**Symptom:** `eas submit` succeeds at queueing but fails at the upload step with a generic message:

```
✔ Scheduled iOS submission

Submission details: https://expo.dev/accounts/<account>/projects/<slug>/submissions/<submission-id>

Waiting for submission to complete. You can press Ctrl+C to exit.
- Submitting
✖ Something went wrong when submitting your app to Apple App Store Connect.
```

Re-running with `--verbose --verbose-fastlane` produces *exactly the same* generic message. No ITMS code, no fastlane stack trace, no details. The local CLI is a dead end.

**Root cause:** `eas submit` queues the job; the actual upload-to-ASC happens on EAS workers using their internal fastlane runner. Logs from that worker are written to the EAS dashboard, not streamed back to the local CLI. Even `--verbose-fastlane` only enables verbose mode on the *worker side* — the output goes to the worker log, not your terminal.

**Fix:** Open the submission URL printed by the CLI:

```
https://expo.dev/accounts/<account>/projects/<slug>/submissions/<submission-id>
```

Expand the "Upload to App Store Connect" job. Scroll to the bottom — `Errors:` section contains the actual ITMS codes and fastlane error text. Copy-paste from there to diagnose.

**Prevention:** When `eas submit` reports a generic failure, don't waste time re-running with verbose flags. Go straight to the submission URL. Add to your runbook: "EAS submit failed → open submission URL → read Errors block → consult ITMS-ERRORS.md."

If you need to programmatically fetch the submission log (e.g. CI), there's no first-class CLI command in eas-cli ≤18.8.x — the dashboard is the only sanctioned reading interface. Workarounds: scrape the dashboard HTML, or run `eas submit` interactively in CI and tee output (limited utility — same generic message).

**Source:** 2026-04-29, Expo / EAS update submission. Tested with eas-cli 16.28.0.

---

## OP-30: Apple closes a "version train" after approval — TestFlight is not exempt

**Symptom:** After version `2.5.0` is approved + released to the App Store, you build a new TestFlight-only build of the same app at version `2.5.0` (intent: keep iterating internal QA without bumping the public version). EAS submit / `pilot upload` / Transporter all fail with the pair:

```
- This bundle is invalid. The value for key CFBundleShortVersionString [2.5.0] in the
  Info.plist file must contain a higher version than that of the previously approved
  version [2.5.0]. (90062)
- Invalid Pre-Release Train. The train version '2.5.0' is closed for new build
  submissions. (90186)
```

ITMS-90062 and ITMS-90186 fire **as a pair** — when you see one, you'll see the other.

**Root cause:** Apple models versions as "trains" — a train is the set of builds attached to a single `CFBundleShortVersionString`. When a version's train reaches "approved" state (released, or pending Manual Release after approval), the train **closes** for new uploads. Apple does not distinguish TestFlight uploads from App Store uploads at this layer: both attach to the train, both are rejected once the train is closed.

This catches teams who model TestFlight as an independent beta channel. It isn't. TestFlight builds attach to whichever version they were built with; that version's lifecycle in the App Store ladder controls whether new TestFlight builds can join.

States that close a train:
- Version state `READY_FOR_DISTRIBUTION` (released)
- Version was approved with Manual Release option, awaiting your release tap (still closed for TF uploads)
- Version was `REMOVED_FROM_SALE` (you pulled it post-launch)

States that keep a train open:
- `PREPARE_FOR_SUBMISSION` (never submitted yet)
- `WAITING_FOR_REVIEW` / `IN_REVIEW`
- `DEVELOPER_REJECTED` / `INVALID_BINARY` / `REJECTED`

**Fix:** Bump `CFBundleShortVersionString` to a higher value, then rebuild. Patch bump (`2.5.0` → `2.5.1`) is the natural choice for "I just want a fresh TestFlight build of nearly identical code."

| Stack | Edit |
|---|---|
| Native Xcode | Project → General → Identity → Version, or `agvtool new-marketing-version 2.5.1` |
| Flutter | `pubspec.yaml`: `version: 2.5.0+40` → `version: 2.5.1+1` |
| Expo | `app.config.js`: `version: "2.5.0"` → `version: "2.5.1"` |
| Bare RN | `ios/<App>/Info.plist`: `CFBundleShortVersionString` |

For Expo apps with `runtimeVersion.policy: "appVersion"`, the bump also rolls the OTA runtime channel — see OP-33.

**Prevention:** Phase 0 pre-flight should query ASC for the latest approved version and refuse to start a build at the same or lower version:

```bash
APPROVED=$(curl -sH "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appStoreVersions?filter[appStoreState]=READY_FOR_DISTRIBUTION&limit=1" \
  | jq -r '.data[0].attributes.versionString')
LOCAL=$(jq -r '.expo.version' app.json 2>/dev/null \
  || node -e 'console.log(require("./app.config.js").default.expo.version)')

# fail if LOCAL <= APPROVED
```

Or simpler: every time you ship a release, immediately bump the version locally to "next" so the working tree is always staged for the next train.

**For EAS users specifically:** the rejection happens at the Apple side, after EAS has already uploaded. The local `eas submit` output shows only "Something went wrong" (OP-29). The actual ITMS codes appear at the submission URL.

**Source:** 2026-04-29, Expo / EAS project. Version 2.5.0 was already live on the App Store; build 41 of the same version was rejected at upload despite intent for TestFlight only. Resolved by bumping to 2.5.1.

---

## OP-31: zsh `status` is a read-only built-in — name your polling variable anything else

**Symptom:** A polling loop that looks correct in bash fails immediately under zsh (default macOS shell since Catalina):

```bash
until status=$(eas build:view "$id" --json | grep '"status"' | head -1 | sed 's/.*: "//;s/",.*//') \
      && [[ "$status" == "FINISHED" ]]; do
  sleep 30
done
```

```
(eval):1: read-only variable: status
```

The script exits with code 1 on the first iteration, never running the body.

**Root cause:** zsh has `status` as a read-only built-in variable (an alias for `?`, the exit status of the last command). Assigning to it is a hard error, not a warning. This is zsh-only — bash has no such restriction.

**Fix:** Rename the variable to anything else. Idiomatic single-letter `s`, descriptive `state`, or namespaced `build_status` all work.

```bash
until s=$(eas build:view "$id" --json | grep '"status"' | head -1 | sed 's/.*: "//;s/",.*//') \
      && [[ "$s" == "FINISHED" ]]; do
  sleep 30
done
```

**Prevention:** Treat `status` as a reserved word in shell scripts that may run under zsh. Other zsh read-onlies to avoid: `argv`, `pipestatus`, `errno`, `funcstack`, `funcsourcetrace`, `LINENO`. Default to descriptive variable names (`build_state`, `submission_status`) — they read better and dodge the entire class of conflict.

If you must use bash semantics, prefix with `#!/bin/bash` and ensure the script is invoked through bash (not sourced under zsh).

**Source:** 2026-04-29, EAS build polling. First iteration of polling loop failed; renamed `status` → `s`, worked.

---

## OP-32: EAS build artifacts expire after 30 days

**Symptom:** You build via `eas build`, all goes well, IPA is FINISHED. You then defer submission — maybe waiting on metadata, maybe testing another path. Six weeks later you run `eas submit --latest` and the upload fails because the artifact URL returns 404 / "expired."

Inspecting `eas build:view <id> --json` shows the FINISHED build but with an `expirationDate` in the past:

```json
"expirationDate": "2026-02-11T19:33:02.829Z",
"completedAt": "2026-01-12T19:39:13.016Z"
```

That's exactly 30 days from `createdAt`.

**Root cause:** EAS retains build artifacts for 30 days from creation, then garbage-collects them. The build record itself persists (you can see it in `eas build:list`), but the IPA / xcarchive download URLs return 404. This is a hosted-service cost-control measure.

**Fix:** Rebuild. There is no "extend retention" knob; you have to run `eas build` again. The new build will have a fresh fingerprint and a new 30-day window.

**Prevention:**
- **For active development:** ship promptly after building. If you're holding a build for review, plan to submit within a week, not a month.
- **For any "save this binary" need:** download the IPA locally as soon as the build finishes:
  ```bash
  curl -sL -o builds/v$VERSION-build$BUILDNUM.ipa "$applicationArchiveUrl"
  ```
  Local IPAs don't expire. You can also re-upload them via `eas submit --path <local.ipa>` if the build's still valid against ASC's view of the world (but the version-train rules from OP-30 still apply).
- **For long-term archival** (regulatory, audit): download both IPA and dSYM bundle; commit the dSYM hash to your release notes for symbol-server lookups later.

**Source:** 2026-04-29, Expo / EAS project. Build 39 (v2.5.0) had `completedAt: 2026-01-12`, `expirationDate: 2026-02-11`, and was no longer downloadable on 2026-04-29 — necessitating a rebuild.

---

## OP-33: Expo `runtimeVersion.policy: "appVersion"` couples OTA updates to native version bumps

**Symptom:** You bump `version: "2.5.0"` → `"2.5.1"` in `app.config.js`, ship build 41 to TestFlight. Existing TestFlight users on build 40 (still v2.5.0) stop receiving OTA updates from your `eas update` channel. New OTA updates published after the bump only target 2.5.1 clients.

**Root cause:** With `runtimeVersion: { policy: "appVersion" }`, Expo derives the runtime version from the `version` field. Each unique `version` opens a separate OTA channel. OTA updates published with `eas update --channel production` target whichever runtime version was current at publication time — they don't backfill.

This is by design: `appVersion` policy assumes that any version bump may include native code changes that incompatible with old JS bundles. So Expo refuses to roll forward a 2.5.1 OTA bundle to a 2.5.0 client (and vice versa). Safe; just surprising the first time.

**Fix:** Three options depending on your need:

1. **Accept the channel split.** During the transition, publish OTA updates for both runtime versions. Most teams don't bother — TestFlight users update to the new build quickly, and the old runtime channel goes dark.

2. **Switch to `runtimeVersion: "1.0.0"` (literal string).** Pin a single runtime across many `version` bumps. OTA updates roll forward freely. Trade-off: easy to ship an OTA that breaks an older build's native dependencies. Use only if you tightly control native dep updates.

3. **Switch to `policy: "fingerprint"`** (Expo SDK 51+). Runtime version is computed from the actual native dependency fingerprint, not the version string. Bumping `version` without changing native deps keeps the same runtime. This is the modern recommended setting for most apps.

To migrate from `appVersion` to `fingerprint`:

```js
// app.config.js
runtimeVersion: { policy: "fingerprint" }  // was: { policy: "appVersion" }
```

Then `eas build` and `eas update --channel production` once to seed the new runtime. Existing clients on the old runtime won't get further OTA updates (they're on a frozen runtime), but new builds onwards share a runtime as long as native deps don't change.

**Prevention:** Decide on `runtimeVersion.policy` early in the project. For solo dev / tight feedback loop on TestFlight, `fingerprint` is almost always right. `appVersion` is a footgun that pretends to be conservative.

When auditing an Expo project for OTA strategy: `grep runtimeVersion app.config.js`. If you see `policy: "appVersion"`, check `git log` for how often `version` is bumped — frequent bumps mean lots of OTA channel drift.

**Source:** 2026-04-29, Expo / EAS project. Bumping 2.5.0 → 2.5.1 for the version-train fix (OP-30) was noted to also break the OTA channel for any 2.5.0 clients still in TestFlight.

---

## OP-34: macOS system Python is externally-managed; use `uv` for ASC API scripting

**Symptom:** You write a small Python script to mint a JWT and query the ASC API. Running it dies immediately:

```
ModuleNotFoundError: No module named 'jwt'
```

`pip3 install pyjwt` then refuses:

```
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try brew install
    pyjwt, if it is packaged.
    ...
hint: See PEP 668 for the detailed specification.
```

`--break-system-packages` works but pollutes the system Python.

**Root cause:** macOS Python (system Python or Homebrew Python 3.12+) ships an `EXTERNALLY-MANAGED` marker per PEP 668. pip refuses to write to it because Homebrew owns the package manifest. This applies to *every* macOS Python the user is likely to have on PATH unless they've set up venvs or `pipx`.

**Fix:** Use `uv` for one-shot ephemeral envs. It's the cleanest path for ASC API scripting because it (a) installs deps in milliseconds, (b) doesn't touch system Python, (c) caches across runs:

```bash
uv tool run --with pyjwt --with cryptography python /path/to/asc_script.py
```

First run installs `pyjwt` + `cryptography` in ~8ms (cached afterwards). Subsequent runs are effectively instant.

If `uv` isn't installed: `brew install uv` or `curl -LsSf https://astral.sh/uv/install.sh | sh`.

Alternatives if uv is unavailable:
- `pipx install pyjwt; pipx inject pyjwt cryptography` — heavier setup but persistent
- Manual venv: `python3 -m venv .venv && .venv/bin/pip install pyjwt cryptography && .venv/bin/python script.py`
- `pip3 install --user --break-system-packages pyjwt cryptography` — pollutes user site-packages, not recommended

**Prevention:** Default to `uv tool run --with <deps> python` for all ASC API scripts in the skill. Don't write `pip install` instructions and assume macOS users can run them.

**Source:** 2026-04-29, Flutter TestFlight upload session — needed PyJWT to query `/v1/builds` after `xcrun altool --upload-app`. System Python blocked pip; `uv tool run` ran the script in ~12s end-to-end including dep install.

---

## OP-35: ASC API "build not yet visible" window; build can skip PROCESSING and go straight to VALID

**Symptom:** `xcrun altool --upload-app` succeeds with a Delivery UUID. Immediately querying `GET /v1/builds?filter[app]={app_id}&sort=-uploadedDate` does not return the new build — only the previously uploaded ones. After a few minutes the build appears, often already in state `VALID`. You never observe state `PROCESSING` via the API.

**Root cause:** Two separate phenomena:
1. **Visibility lag (~1–3 min after upload):** The upload acknowledgment commits the binary to ASC's pipeline, but the build resource isn't published to the public REST API until ASC ingests it. During this window the build ID exists in their backend (and would conflict with a duplicate upload) but is not returned by `GET /v1/builds`.
2. **State sampling miss:** For small/medium IPAs, ASC processes faster than your poll interval. With a ~62MB Flutter IPA and a 45s poll cadence, the build went from "not visible" directly to `VALID` in two consecutive samples — `PROCESSING` was never observed.

**Fix:** Polling code must handle three states, not two:

```python
while time.time() < deadline:
    try:
        builds = call(f"/v1/builds?filter[app]={app_id}&limit=10&sort=-uploadedDate")
    except Exception as e:
        # Network blips happen. Don't kill the loop on one bad request.
        print(f"api error: {e}", flush=True); time.sleep(45); continue

    found = next((b for b in builds["data"] if b["attributes"]["version"] == TARGET_BUILD), None)
    if found is None:
        print("build not yet visible in ASC", flush=True)  # treat as PROCESSING
    else:
        state = found["attributes"]["processingState"]
        if state == "VALID":
            return  # done
        if state in ("FAILED", "INVALID"):
            raise SystemExit(2)
    time.sleep(45)
```

Use `limit=10` (not `limit=1`) so a parallel upload of a different version doesn't push your target out of the slice.

**Prevention:**
- Set the polling deadline to 30+ min, not 5. Even fast processing has a long tail.
- Don't fail fast on "build not in the response" — that's the normal first sample.
- Log state transitions only when they change, not every poll. Reduces noise.
- 45s poll interval is a good balance: ASC rate limit is generous, but every-5s polling is wasteful for a process that takes minutes.

**Source:** 2026-04-29, Flutter project (build 3, 62MB IPA). Upload completed 11:20:13 local. First poll at 11:21:51 (1m38s after upload): not yet visible. Second poll at 11:25:40 (5m27s after upload): VALID, expired=False. Total upload→VALID was ~5.5 min, with `PROCESSING` never observed.

---

## OP-36: Bundle ID hides behind `$(PRODUCT_BUNDLE_IDENTIFIER)`; verify inside the IPA

**Symptom:** You read `ios/Runner/Info.plist` looking for the bundle ID. It says:

```xml
<key>CFBundleIdentifier</key>
<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
```

You read `ios/Runner.xcodeproj/project.pbxproj` and see *two* matches:

```
PRODUCT_BUNDLE_IDENTIFIER = com.example.app;
PRODUCT_BUNDLE_IDENTIFIER = com.example.app.RunnerTests;
```

Which value actually shipped in the IPA? You can't tell from the source alone.

**Root cause:** Xcode build settings interpolate at build time. The active scheme's *target* (Runner vs RunnerTests vs an extension) determines which `PRODUCT_BUNDLE_IDENTIFIER` baked into `Info.plist`. Multiple build configs (Debug/Release) can hold different values. A misconfigured CI matrix or a stale `xcconfig` can produce an IPA whose bundle ID doesn't match what you'd assume from the project file.

**Fix:** Always verify the actual bundle ID *inside* the IPA before upload. The IPA is a zip; the runtime `Info.plist` lives at `Payload/<App>.app/Info.plist`:

```bash
unzip -p build/ios/ipa/app.ipa "Payload/Runner.app/Info.plist" | \
  plutil -p - | \
  grep -E "CFBundleIdentifier|CFBundleVersion|CFBundleShortVersion|MinimumOSVersion"
```

Output:
```
  "CFBundleIdentifier" => "com.example.app"
  "CFBundleShortVersionString" => "1.0.0"
  "CFBundleVersion" => "3"
  "MinimumOSVersion" => "13.0"
```

Confirm: bundle ID matches the ASC app record, version is what you think you're shipping, build number is greater than the previous upload, and `MinimumOSVersion` matches your `IPHONEOS_DEPLOYMENT_TARGET`.

**Prevention:** Run this command as part of any pre-upload sanity check. It catches:
- Wrong scheme baked into the IPA (RunnerTests instead of Runner)
- Stale `MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` from a missed build-number bump
- Forgotten `IPHONEOS_DEPLOYMENT_TARGET` change that would silently regress min-iOS support

In a `fastlane preflight` lane:

```ruby
ipa = "../build/ios/ipa/Runner.ipa"
bundle_id = sh("unzip -p #{ipa} 'Payload/Runner.app/Info.plist' | plutil -extract CFBundleIdentifier raw -").strip
UI.user_error!("Wrong bundle id #{bundle_id}") unless bundle_id == "com.example.app"
```

**Source:** 2026-04-29, Flutter project. Verified `com.example.app` was the actual baked-in bundle ID (not the `.RunnerTests` sibling) before uploading.

---

## OP-37: Flutter app → TestFlight in zero fastlane lines

**Symptom:** A Flutter dev opens the skill and sees fastlane pervasively in every example. They assume fastlane is required for TestFlight submission and start a 30-min Bundler/Gemfile/Fastfile yak shave. None of it is needed for the first submission.

**Root cause:** The skill's primary examples are fastlane-led because fastlane's value compounds across releases (lane composition, automatic build-number bumps, screenshot generation, deliver/match integration). For a *single* TestFlight upload, the bare-bones path is shorter, has fewer moving parts, and uses tooling that ships with Xcode.

**Fix:** Document the zero-fastlane Flutter flow as the headline first-submission path. The complete pipeline is four commands plus one ASC web UI pass:

```bash
# 1. Build (Flutter does archive + export in one step)
flutter build ipa --release --export-options-plist=ios/ExportOptions.plist

# 2. Verify bundle ID baked into the IPA matches your ASC record (OP-36)
unzip -p build/ios/ipa/<App>.ipa "Payload/Runner.app/Info.plist" | \
  plutil -p - | grep -E "CFBundleIdentifier|CFBundleVersion|CFBundleShortVersion"

# 3. Validate (catches ITMS errors locally in 30s)
xcrun altool --validate-app -f build/ios/ipa/<App>.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_UUID>

# 4. Upload
xcrun altool --upload-app -f build/ios/ipa/<App>.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_UUID>
```

Then: ASC web UI → My Apps → `<App>` → TestFlight tab → wait 5–30 min for processing → assign build to Internal Testing group → install via TestFlight on your iPhone.

Pre-reqs that survive across submissions (do once):
- `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` — ASC API key (App Manager role is sufficient — OP-5 caveat applies only to cloud signing)
- Apple Distribution cert in keychain (`security find-identity -v -p codesigning | grep "Apple Distribution"`)
- App Store provisioning profile (Xcode auto-manages for free with cloud signing — OP-4 caveat applies)
- `ios/ExportOptions.plist` with `method=app-store-connect` and your team ID (Flutter generates this on first `flutter build ipa`)
- ASC app record created in the web UI with matching bundle ID
- `ITSAppUsesNonExemptEncryption=false` in `Info.plist` (saves the per-build encryption questionnaire — OP-24)

**Prevention:** When the user says "I just need to ship this Flutter app to TestFlight," route to the four-command flow first. fastlane comes later when the user wants:
- Repeated releases with build-number bumps (`increment_build_number`)
- Metadata managed in version control (`fastlane deliver`)
- Screenshot generation (`fastlane snapshot`)
- CI integration without Apple ID 2FA

**Source:** 2026-04-29, Flutter first TestFlight submission. Total wall-clock from "IPA is here" to "build is VALID in TestFlight": ~7 minutes (90s validate + 4s upload + 5.5min processing). fastlane was never installed.

---

## OP-38: ExportOptions.plist `method=app-store-connect` is the current name; `app-store` is legacy but still accepted

**Symptom:** Skill examples and older fastlane docs show `method: app-store` in `ExportOptions.plist`. Flutter's `flutter build ipa --export-method app-store-connect` (and fastlane's newer templates) write `method: app-store-connect`. You're not sure which one to use, and worry that the wrong one will cause an export failure.

**Root cause:** Apple renamed the export method in Xcode 15.3 from `app-store` to `app-store-connect` to align with the ASC product name. Both values currently work — Xcode silently accepts the legacy name. Future Xcode versions may deprecate `app-store`.

**Fix:** Use `app-store-connect` in any new ExportOptions.plist. Example (verified shipping — Flutter 3.27, Xcode 16.x):

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

If you have a long-lived `ExportOptions.plist` with `method: app-store`, leave it — both work, and changing it adds noise to the diff. Flag the rename only when the file is being touched for another reason.

**Prevention:** When generating ExportOptions.plist by hand or via a template, default to `app-store-connect`. When reviewing fastlane configs (`gym_options.export_method`) or Flutter `--export-method` flags, accept both spellings as valid.

**Source:** 2026-04-29, Flutter project. Flutter wrote `method: app-store-connect` to `build/ios/ipa/ExportOptions.plist`; the IPA exported, validated, and uploaded successfully.

---

## OP-39: XcodeGen-managed projects: put signing config in `project.yml`, never edit the `.xcodeproj`

**Symptom:** You set `DEVELOPMENT_TEAM` and `CODE_SIGN_STYLE` in the Xcode UI (or a `pbxproj` patch) on a native Swift project. Archive succeeds. Next time you (or CI) run `xcodegen generate` — for any reason, including a teammate adding a new file — those settings vanish and the next archive fails with `error: No Accounts` or `Signing for "<target>" requires a development team`.

**Root cause:** XcodeGen-managed projects regenerate `.xcodeproj` from `project.yml` every time `xcodegen generate` runs. Anything edited directly in the pbxproj — including signing config Xcode adds when you click checkboxes in the UI — is overwritten. The pbxproj is a build artifact, not source of truth. This trips first-time native+XcodeGen submitters because the standard "open in Xcode → Signing & Capabilities → set team" flow that works for non-XcodeGen projects silently doesn't persist.

**Fix:** Set signing in `project.yml` under `targets.<target>.settings.base` so it's regenerated into every build:

```yaml
targets:
  MyApp:
    type: application
    platform: iOS
    sources:
      - path: MyApp
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.myco.myapp
        DEVELOPMENT_TEAM: ABCD123456              # 10-char team ID, NOT team name
        CODE_SIGN_STYLE: Automatic                 # or Manual + PROVISIONING_PROFILE_SPECIFIER
        ASSETCATALOG_COMPILER_APPICON_NAME: AppIcon
```

For manual signing, add `PROVISIONING_PROFILE_SPECIFIER: "<profile-name>"` and `CODE_SIGN_IDENTITY: "Apple Distribution"` (or specific cert name).

**Find your team ID:** `security find-identity -v -p codesigning | grep "Apple Distribution"` — the parenthesized 10-char string is your team ID.

**Prevention:** When you see a `project.yml` at the repo root, treat the `.xcodeproj` as a generated artifact — like `Cargo.lock` or `node_modules`. All persistent project config goes through `project.yml`. Add `.xcodeproj/` to `.gitignore` if it isn't already (most XcodeGen projects do). Before archiving, always re-run `xcodegen generate` to be sure your local `.xcodeproj` matches `project.yml`.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Initial `project.yml` had no team config; `xcodebuild archive` failed until `DEVELOPMENT_TEAM` and `CODE_SIGN_STYLE` were added under `targets.MyApp.settings.base`.

---

## OP-40: `xcodebuild archive` log shows Development signing — Distribution happens at `-exportArchive`

**Symptom:** During `xcodebuild archive` for an App Store build, the CodeSign step prints something alarming:
```
Signing Identity:     "Apple Development: Jane Developer (XYZW987EFG)"
Provisioning Profile: "iOS Team Provisioning Profile: *"
```
You expected `Apple Distribution: <Team>` and a Distribution profile. You assume the archive is wrong and won't ship.

**Root cause:** Apple's archive pipeline is two-stage:

1. **Archive stage** (`xcodebuild archive`) embeds the binary inside a `.xcarchive` and signs it with *whatever profile matches* the bundle ID at archive time. If a Development profile is what's installed, that's what gets used. The archive isn't shippable yet, but it's a valid archive.
2. **Export stage** (`xcodebuild -exportArchive` with `ExportOptions.plist` specifying `method: app-store-connect`) **re-signs** the binary with the Distribution cert + App Store provisioning profile, then packages it as the `.ipa` you upload.

So Development signing in the archive log is fine — the export is what determines App Store eligibility. The archive log only matters if it shows an error or skips signing entirely.

**Fix:** Don't fix; verify. After export:

```bash
codesign -dvvv "build/ipa/MyApp.app" 2>&1 | grep -E "Authority|TeamIdentifier"
# Expected:
# Authority=Apple Distribution: <Team Name> (<Team ID>)
# TeamIdentifier=<Team ID>
```

Then check the embedded provisioning profile:
```bash
unzip -p build/ipa/MyApp.ipa "Payload/MyApp.app/embedded.mobileprovision" \
  | security cms -D | plutil -p - | grep -E "Name|application-identifier"
# Expected: "iOS Team Store Provisioning Profile: <bundle.id>"
```

If the export succeeded with `method: app-store-connect` (or legacy `app-store`) and `signingStyle: automatic`, you're good — `xcodebuild` won't silently produce a Development-signed IPA when asked for App Store.

**Prevention:** Read the archive log for *errors only*. The `Signing Identity:` line in archive output is informational; trust the export output and the embedded profile in the resulting IPA. Cross-reference with [OP-27](#op-27-first-archive-that-succeeds-with-cloud-signing-produces-ios-team-store-provisioning-profile-name).

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Archive log showed `Apple Development: Jane Developer`; export with `app-store-connect` method correctly re-signed with `Apple Distribution: Acme Labs, LLC` and the IPA validated and uploaded clean.

---

## OP-41: `-allowProvisioningUpdates` auto-registers brand-new bundle IDs in the Developer Portal

**Symptom:** You set `PRODUCT_BUNDLE_IDENTIFIER: com.myco.brandnew` in `project.yml`. The bundle ID has never been registered in the Apple Developer portal (no App ID record exists). You expect `xcodebuild archive` to fail with "no provisioning profile found." Instead, archive succeeds, export succeeds, and a Distribution profile materializes for a bundle ID you never explicitly created.

**Root cause:** `xcodebuild -allowProvisioningUpdates` does more than the docs explicitly state. With automatic signing + a `DEVELOPMENT_TEAM` set + an Xcode-logged-in Apple ID with sufficient role (Account Holder, Admin, or App Manager), it will:

1. Register the bundle ID in the Developer Portal (`POST /v1/bundleIds`) if not already registered.
2. Create or refresh an App Store provisioning profile for it.
3. Embed the freshly-minted profile into the archive on export.

This means the manual "Certificates, Identifiers & Profiles → Identifiers → +" step that older docs describe is unnecessary for first-time bundle IDs in 2025+.

**Fix:** Use it. Pass `-allowProvisioningUpdates` to *both* the archive and export commands:

```bash
xcodebuild archive \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/MyApp.xcarchive \
  -allowProvisioningUpdates \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=<TeamID>

xcodebuild -exportArchive \
  -archivePath build/MyApp.xcarchive \
  -exportOptionsPlist build/ExportOptions.plist \
  -exportPath build/ipa \
  -allowProvisioningUpdates
```

Verify via API after first archive:
```bash
# Bundle ID should now appear:
curl -H "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/bundleIds?filter[identifier]=com.myco.brandnew"
```

**Caveat:** This only registers the bundle ID in the *Developer Portal*. It does NOT create an App Store Connect app record (see [OP-42](#op-42-asc-rest-api-does-not-expose-create-app)). For first submission you still need the ASC web UI step.

**Caveat 2:** The Apple ID used for cloud signing must have permission to register App IDs. App Manager is enough. See [OP-5](#op-5-app-manager-api-key-cannot-create-distribution-cert).

**Prevention:** When the skill or another doc says "register the bundle ID in developer.apple.com first," that step is now optional in modern Xcode (16.x / 26.x). Don't waste time on the portal UI for fresh bundle IDs — let xcodebuild handle it. Still worth verifying via API afterward to confirm the registration stuck.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Bundle ID `com.example.app1` (later renamed to `com.example.app2`) was never explicitly registered; first archive auto-registered it. Confirmed via `GET /v1/bundleIds?filter[identifier]=...`.

---

## OP-42: ASC REST API does not expose Create App

**Symptom:** You assume that if `GET /v1/apps` exists, `POST /v1/apps` exists too, and try to script first-time app creation as part of a one-button release pipeline. Apple's OpenAPI spec lists `apps` as a resource, hinting at full CRUD. You waste 30+ minutes hunting for the right body schema, hitting 404s, 405s, and "Not Found" responses for permutations of the request.

**Root cause:** The public ASC REST API genuinely does not support creating App Store Connect app records. `POST /v1/apps` is not implemented. `fastlane produce` works, but it uses an undocumented private endpoint (`appstoreconnect.apple.com/iris/v1/apps` or similar) that requires session-cookie auth, not the API key. Apple has not exposed the create-app operation to the public REST API as of 2026-04. The OpenAPI spec lists `apps` but the only POST/PATCH-able fields are post-creation (e.g. age rating, app info). Initial creation is web-UI-only via the public API.

**Fix:** For first submissions, accept that ASC web UI app creation is unavoidable unless you use `fastlane produce`. The web UI step is ~60 seconds:

1. <https://appstoreconnect.apple.com/apps> → "+" → New App
2. Pick platform (iOS), bundle ID from dropdown (auto-populated after [OP-41](#op-41) registers it), name, primary language, SKU
3. Click Create

If you're scripting and don't want fastlane, the workflow is:
1. Script does `xcodebuild archive` + `-allowProvisioningUpdates` (registers bundle ID)
2. Script pauses with a clear message: "Open ASC and click Create New App with bundle ID X, name Y, SKU Z. Press Enter when done."
3. Script polls `GET /v1/apps?filter[bundleId]=...` until it returns the new record (see [OP-43](#op-43))
4. Script continues with metadata + upload

For repeat releases (after the first one), the app record already exists — the rest of the pipeline is fully scriptable.

**`fastlane produce` alternative:**
```ruby
# Fastfile
lane :create_app do
  produce(
    app_identifier: "com.myco.myapp",
    app_name: "My App",
    language: "English",
    sku: "myapp-001",
    team_id: ENV["APPLE_TEAM_ID"]
  )
end
```
This works but requires fastlane, which is heavier than the web-UI step for solo first submissions.

**Prevention:** Don't search for `POST /v1/apps` — it's not there. When designing release tooling, plan for the web UI hand-off on first submission and full automation on every subsequent release.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. After confirming `GET /v1/apps?filter[bundleId]=com.example.app1` returned empty, considered scripting `POST /v1/apps`. Apple's API reference confirmed no such endpoint exists; user created the record via web UI in <2 min.

---

## OP-43: ASC API has 30–90s propagation lag for newly created app records

**Symptom:** User clicks "Create" in ASC web UI for a new app. You poll `GET /v1/apps?filter[bundleId]=com.myco.myapp` immediately. Empty result. You assume the create failed silently. User confirms via web UI that the app exists.

**Root cause:** ASC's web UI writes to a different store (or replication tier) than the public REST API reads from. The propagation delay is typically 30–90s but can stretch to ~3 minutes during high-load periods. There is no API endpoint to confirm propagation has completed; the only way is to poll the same filter you'd use later.

**Fix:** Implement a poll loop with backoff, not a one-shot query:

```python
import time, json, urllib.request, jwt, os, sys

def get(token, path):
    req = urllib.request.Request(
        f"https://api.appstoreconnect.apple.com{path}",
        headers={"Authorization": f"Bearer {token}"},
    )
    with urllib.request.urlopen(req) as r:
        return json.loads(r.read())

def wait_for_app(token, bundle_id, timeout_s=300, poll_interval_s=15):
    deadline = time.time() + timeout_s
    while time.time() < deadline:
        data = get(token, f"/v1/apps?filter[bundleId]={bundle_id}&limit=1")
        if data.get("data"):
            return data["data"][0]
        time.sleep(poll_interval_s)
    raise TimeoutError(f"App with bundle ID {bundle_id} not visible in ASC API after {timeout_s}s")
```

Or with a shell `until` loop:
```bash
until uv tool run --with pyjwt --with cryptography python check_app.py 2>&1 | grep -q "APP RECORD EXISTS"; do
  sleep 15
done
```

**Prevention:** Build the poll-until-visible pattern into any first-submission tooling. Don't gate the next step on a single API response. Skill kernel rule #11 logic applies in reverse here: if the user just created the app, expect a delay before the API agrees.

The same lag applies to:
- Builds appearing in `/v1/builds` after `altool --upload-app` ([OP-35](#op-35) — 1–3 min)
- New API keys becoming usable after creation
- New screenshots becoming visible after upload

Lag is normal; treat it as a state machine, not a binary "exists / doesn't exist."

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Initial poll (<10s after web UI Create click) returned empty; ~45s later the record was visible.

---

## OP-44: ASC App "Name" and `CFBundleDisplayName` are independent — match them anyway

**Symptom:** App Store listing shows "Final Name" but the home-screen icon under the app says "Mid-flow Name" because `CFBundleDisplayName` was set during archive before the user picked the final ASC name. Beta testers complain. You wonder if it'll get rejected.

**Root cause:** Two separate fields:
- **ASC App Name** (`/v1/apps[].attributes.name`): what shows on the App Store listing, in search results, and on the App Store Connect dashboard. Editable in ASC at any time (subject to App Review approval on next submission).
- **`CFBundleDisplayName`** in `Info.plist`: what shows on the device home screen under the app icon. Locked into the IPA at build time; only changes on the user's device after a TestFlight/App Store update.

Apple does not require these to match. There's no validation step that flags divergence. App Review *can* (rarely) flag drastically inconsistent names under Guideline 4.0, but matching ASC name to the in-app branding is more typical than matching `CFBundleDisplayName`.

**Fix:** Match them at first submission to avoid user confusion:
1. Decide the final brand name *before* archiving.
2. Set `CFBundleDisplayName` in `Info.plist` (or `app.config.js` for Expo) to that name.
3. Use the same name when creating the ASC record.

Mid-flow rename workflow (encountered repeatedly with first-time submitters):

| Stage | What to update |
|---|---|
| Before archive | `CFBundleDisplayName`, ASC app name (if record exists) |
| After archive, before upload | Re-archive with new `CFBundleDisplayName`; ASC name is web-UI editable |
| After upload, before submit | Only ASC name (PATCH `/v1/apps/{id}`); `CFBundleDisplayName` is locked into the build |
| After approval | Only ASC name (subject to next-build review); `CFBundleDisplayName` requires new build + new version train |

**Prevention:** During Phase 0 / first-submission planning, lock the brand name *before* the user starts clicking around in ASC. If it changes mid-flow, expect a 2-minute rebuild cycle (re-archive → re-export → re-validate → re-upload). The cost is cheap; just don't promise the user a one-shot upload until the name is stable.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. App started as "Working Title", renamed to "Mid-flow Name" during ASC create-app, renamed to "Final Name" before clicking Create. `CFBundleDisplayName` was updated and re-archived to keep the home-screen name aligned with the App Store listing.

---

## OP-45: ASC `/v1/builds` filter — `version` is build number, `preReleaseVersion.version` is marketing version

**Symptom:** You're polling `/v1/builds?filter[version]=1.0` to find the build you just uploaded as `CFBundleShortVersionString=1.0`, `CFBundleVersion=1`. Empty result. Apple's docs say the filter exists but it doesn't seem to work.

**Root cause:** Apple's API uses "version" to mean two different things and doesn't disambiguate in the schema field name:
- On `/v1/builds`, `attributes.version` = `CFBundleVersion` = the build number (e.g. `"1"`, `"42"`, `"2026.04.29"`).
- The marketing version (`CFBundleShortVersionString`, e.g. `"1.0"`, `"1.2.3"`) lives on the related `preReleaseVersion` resource as `attributes.version`.

So `filter[version]=1.0` searches for builds whose *build number* is `1.0`, which is unusual and rarely matches. The filter you actually want for "marketing version 1.0, build 1" is:

```
/v1/builds
  ?filter[app]=<APP_ID>
  &filter[preReleaseVersion.version]=1.0
  &filter[version]=1
  &fields[builds]=version,processingState,uploadedDate,expired,minOsVersion
```

Naming alignment table:
| What you call it | Info.plist key | ASC API field | Builds endpoint filter |
|---|---|---|---|
| Marketing version | `CFBundleShortVersionString` | `preReleaseVersion.version` (relation) | `filter[preReleaseVersion.version]=1.0` |
| Build number | `CFBundleVersion` | `builds.attributes.version` | `filter[version]=1` |

**Fix:** Use the dual filter shown above. Working Python poll snippet:

```python
path = (f"/v1/builds?filter[app]={APP_ID}"
        f"&filter[preReleaseVersion.version]={MARKETING_VERSION}"
        f"&filter[version]={BUILD_NUMBER}"
        f"&fields[builds]=version,processingState,uploadedDate,expired,minOsVersion"
        f"&limit=5")
data = get(path)
builds = data.get("data", [])
```

**Prevention:** Whenever an ASC API call has a "version" field, consult the schema to confirm whether it means build number or marketing version. The convention is consistent (`builds.version` = build number, `preReleaseVersion.version` = marketing version, `appStoreVersions.versionString` = marketing version) but the bare word "version" is ambiguous. Cache discovered schemas with a `last verified` date — see kernel axiom #10.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Initial poll used `filter[version]=1.0` (marketing version), returned empty. Fixed by combining `filter[preReleaseVersion.version]=1.0` + `filter[version]=1` (build number) — found the build immediately.

---

## OP-46: macOS `sips` resizes icons and detects alpha in one line — no Pillow / ImageMagick needed

**Symptom:** You need to verify a 1024×1024 marketing icon has no alpha channel (Apple rejects icons with alpha) and resize a 1254×1254 source PNG to 1024. You reach for Python + Pillow or `brew install imagemagick`, then realize neither is installed and the user is on a fresh Mac.

**Root cause:** macOS ships `sips` (Scriptable Image Processing System) preinstalled at `/usr/bin/sips`. It handles the common image-prep operations for App Store icons in one or two lines, no third-party tools required.

**Fix:** Three idiomatic operations:

```bash
# 1. Check dimensions, color depth, and alpha presence
sips -g pixelWidth -g pixelHeight -g hasAlpha /path/to/icon.png
# Output:
#   pixelWidth: 1254
#   pixelHeight: 1254
#   hasAlpha: no

# 2. Resize to 1024x1024 (preserves aspect ratio if input is square)
sips -z 1024 1024 /path/to/source.png --out /path/to/AppIcon.png

# 3. Strip alpha (if input has it — Apple rejects PNGs with alpha)
sips -s format png -s formatOptions normal --deleteColorManagementProperties \
  --setProperty hasAlpha false /path/to/icon.png --out /path/to/AppIcon.png
```

Combine into a one-shot pre-flight check:

```bash
icon=/path/to/AppIcon.png
read -r w h alpha < <(sips -g pixelWidth -g pixelHeight -g hasAlpha "$icon" \
  | awk '/pixelWidth/{w=$2}/pixelHeight/{h=$2}/hasAlpha/{a=$2}END{print w, h, a}')
if [[ "$w$h" != "10241024" || "$alpha" != "no" ]]; then
  echo "FAIL: icon is ${w}x${h}, hasAlpha=${alpha}; expected 1024x1024 hasAlpha=no"
  exit 1
fi
echo "OK: $icon is 1024x1024 RGB without alpha"
```

**Prevention:** Add the dimension+alpha check to the Phase 1 placeholder-asset audit. Catches three common rejection causes (wrong size, alpha channel, framework default icon — see [OP-1](#op-1) and [OP-25](#op-25)) in one pass. Don't pull in heavier tooling until you actually need it.

**Source:** 2026-04-29, native Swift / XcodeGen first submission. Source icon was 1254×1254 RGB with no alpha; `sips -z 1024 1024` produced a clean 1024×1024 marketing icon that compiled into `Assets.car` and shipped through validation without ITMS errors.
