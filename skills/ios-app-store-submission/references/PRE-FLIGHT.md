# Pre-flight Checklist

Verify everything in this list before starting Phase 1. Failure here cascades into cryptic build/signing/upload errors later, often at the worst time.

> **EAS users:** This file documents the fastlane-style pre-flight (local Distribution cert, ASC API `.p8` on disk, etc.). For Expo / EAS pipelines, several items below are not required — EAS manages signing on its servers. See `EAS-SUBMISSION.md` § "Phase 0 — Pre-flight (EAS variant)" for the EAS-specific checklist. The Apple-side items (developer enrollment, ASC record, bundle ID, version-train check, encryption questionnaire) apply to both paths.

## Apple Developer Program

- Enrollment status: developer.apple.com/account → Membership tab. Active = green badge.
- Renewal date: shown on Membership page. Don't let it lapse — apps go offline immediately and you can't push updates until you re-pay.
- **Team ID**: 10-char alphanumeric (e.g. `ABCD123456`), shown on Membership page. Save in `Appfile`.
- **Individual vs Organization** for solo dev:
  - Individual ($99/yr): name appears in App Store as the developer name. Faster setup.
  - Organization ($99/yr): "Acme Inc, LLC" appears. Requires D-U-N-S number (free to apply, ~14 days). Recommended for professional brands.

## App Store Connect setup (`first-submission` mode only)

### Agreements / Tax / Banking

- ASC → Agreements, Tax, and Banking
- **Paid Apps Agreement**: even if your app is free, this must be signed for any in-app purchase or future paid app. Sign it now, save the round-trip later.
- **Tax Forms**: W-9 (US) or W-8BEN (non-US). Required before any payouts.
- **Banking**: bank account for payouts. Required if app has IAP or paid pricing.

### App Information panel

- ASC → My Apps → New App → fill bundle ID, primary language, SKU.
- **Bundle ID**: select the App ID you registered in developer.apple.com. Reverse-DNS format (`com.example.app`). Must match Xcode project.
- **Primary language**: load-bearing for localization. If you set fr-FR, French becomes the fallback for any unlocalized region (see [OP-14](OPERATOR-PATTERNS.md#op-14-primarylocalefr-fr-forces-french-localization-to-be-filled)). Pick the language your most-targeted region uses (usually English).
- **SKU**: internal identifier; user-visible nowhere. Use the bundle ID or a sortable internal code.

### App record creation: UI vs `fastlane produce`

- **Most solo devs:** create in ASC web UI ("My Apps" → "+" → "New App") once and never again. Fastest for single app.
- **`fastlane produce`** for full automation: creates the App ID + ASC record from CLI, useful if shipping many apps or doing infra-as-code. Uses an undocumented private endpoint (session-cookie auth, not the API key).
- **The public ASC REST API does NOT support creating apps** — `POST /v1/apps` does not exist (see [OP-42](OPERATOR-PATTERNS.md#op-42-asc-rest-api-does-not-expose-create-app)). Don't waste time hunting for a body schema; the operation isn't exposed. Web UI or `fastlane produce` are the only paths.
- After web-UI creation, the new app takes 30–90s to appear in `GET /v1/apps?filter[bundleId]=...` (see [OP-43](OPERATOR-PATTERNS.md#op-43-asc-api-has-3090s-propagation-lag-for-newly-created-app-records)). Build a poll-until-visible loop, not a one-shot query.
- The bundle ID dropdown in the ASC New App form is auto-populated from the Developer Portal. If you've already run `xcodebuild archive -allowProvisioningUpdates` once, the bundle ID is registered automatically — see [OP-41](OPERATOR-PATTERNS.md#op-41-allowprovisioningupdates-auto-registers-brand-new-bundle-ids-in-the-developer-portal). The "register the App ID in developer.apple.com first" step from older docs is no longer required.
- Trade-off: `produce` requires ASC API key + iTunes Connect API key separately. Web UI is faster for one app.

**Mid-flow rename trap:** If the user changes the brand name *during* the New App form (e.g. picks a different bundle ID from the dropdown, or types a different display name than what's in the IPA's `CFBundleDisplayName`), the IPA needs to be rebuilt to match. Cost is ~2 minutes. Lock the brand name *before* starting Phase 2 if possible. See [OP-44](OPERATOR-PATTERNS.md#op-44-asc-app-name-and-cfbundledisplayname-are-independent--match-them-anyway) for the matrix of what to update at each stage.

## Export compliance / encryption questionnaire

Every submission asks "does your app use encryption?" — even if you only use HTTPS.

**Trivial case** (HTTPS only, no custom crypto): set `ITSAppUsesNonExemptEncryption` to `false` in `Info.plist`. ASC stops asking on every TestFlight upload. See [OP-24](OPERATOR-PATTERNS.md#op-24-itsappusesnonexemptencryptionfalse-in-infoplist-pre-answers-asc-encryption-questionnaire).

```xml
<!-- ios/Runner/Info.plist -->
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

**Non-trivial case** (custom crypto, end-to-end encrypted messaging, etc.): you still answer the questionnaire in ASC and may need to file annual self-classification reports with the U.S. Bureau of Industry and Security (BIS). Apple's documentation on this is at [help.apple.com/app-store-connect/#/dev88f5c7bf9](https://help.apple.com/app-store-connect/#/dev88f5c7bf9).

Forgetting this doesn't block TestFlight uploads but blocks moving builds from TestFlight to App Store review.

## Privacy nutrition labels

- Filled in ASC web UI under App Privacy. Not authoritatively automatable via fastlane today (verify current state).
- Once filled, persists across versions until you change SDKs / data collection.
- **Update when adding/removing third-party SDKs** — every SDK adds disclosures. Common forgotten step.
- See `references/PRIVACY-MANIFEST.md` for the relationship between `PrivacyInfo.xcprivacy` (in the binary) and the ASC nutrition label (in the listing).

## Age rating questionnaire

- Web UI only, in ASC. First submission only — answers persist across updates.
- Apple expands the questionnaire periodically; you may be asked to re-answer when adding new content categories.
- For most utility / travel / productivity apps, all questions answer "None" → 4+ rating.
- See [OP-17](OPERATOR-PATTERNS.md#op-17) for the current API field set if you want to script it (web UI is usually faster).

## Distribution certificate

The single most common signing failure cause. Verify *before* you start Phase 2.

```bash
security find-identity -v -p codesigning
```

You need to see at least one identity matching:
```
<HASH> "Apple Distribution: <Team Name> (<TEAM_ID>)"
```

If absent — even if you have `Apple Development` certs — **Phase 2 will likely fail** unless you only do one archive and rely on Xcode cloud signing (which can succeed once and fail thereafter, see [OP-4](OPERATOR-PATTERNS.md#op-4-first-archive-succeeds-via-xcode-cloud-signing-subsequent-archives-fail)).

To create the cert (one-time):
1. `open -a Xcode`
2. **Xcode → Settings (⌘,) → Accounts**
3. Sign in with the team's Apple ID
4. Select the account → **Manage Certificates…** (lower right)
5. **+** in lower left → **Apple Distribution**
6. Cert created in login keychain. Verify with `security find-identity -v -p codesigning`.

Backup the cert + private key:
- Keychain Access → find "Apple Distribution: ..." → right-click → Export → save `.p12` with passphrase
- Store `.p12` + passphrase in 1Password / encrypted backup
- "Lost private key" recovery: cert exists in Apple Developer portal but you can't sign without the private key. Must revoke + regenerate. Any signed builds with that cert continue to work; you just can't sign new ones.

For multi-machine setup (work laptop + CI), use `fastlane match` (see MATCH-AND-SIGNING.md) to sync certs via private git repo.

## App Store provisioning profile

Less critical to verify manually — Xcode auto-manages App Store profiles when `signingStyle: automatic`. But know the failure mode (see [OP-6](OPERATOR-PATTERNS.md#op-6)): when you create a new Distribution cert, the old auto-managed profile doesn't include the new cert. Use `-allowProvisioningUpdates` on the next export to let Xcode regenerate.

To inspect existing profiles:
```bash
ls ~/Library/MobileDevice/Provisioning\ Profiles/
```
Each `.mobileprovision` is a binary plist; decode with:
```bash
security cms -D -i <file>.mobileprovision
```

If using `match`: profiles are stored encrypted in your match git repo.

## Bundle ID hygiene

The single most common source of cryptic errors. Bundle ID must match exactly across:

- Xcode project → General → Bundle Identifier
- `Info.plist` → `CFBundleIdentifier` (often `$(PRODUCT_BUNDLE_IDENTIFIER)` — that's fine, it pulls from the project setting)
- Provisioning profile → registered App ID
- App Store Connect record → primary bundle ID
- fastlane `Appfile` → `app_identifier`
- For RN/Flutter/Capacitor: framework-level config (e.g. Capacitor's `capacitor.config.json`, RN's `app.json`)

Verification command:
```bash
grep -E "PRODUCT_BUNDLE_IDENTIFIER" ios/Runner.xcodeproj/project.pbxproj | head -5
```

Should all show the same value. If not, edit the project until they do (Xcode UI handles project + Info.plist; manually update Appfile and any framework config).

## ASC API key

Generate before Phase 4 (Upload):
1. ASC → Users and Access → Integrations → Keys tab (URL: <https://appstoreconnect.apple.com/access/integrations/api>) → "Generate API Key"
2. Role: **App Manager** for upload + metadata. **Admin** if you also need cloud signing (see [OP-5](OPERATOR-PATTERNS.md#op-5-cloud-signing-requires-adminaccount-holder-api-key-app-manager-is-insufficient)).
3. Download `.p8` (one-time download — Apple shows it only at creation time).
4. Note **Key ID** (10 alphanumeric chars, e.g. `ABC123XYZ4`) and **Issuer ID** (UUID at top of Keys page, e.g. `12345678-1234-1234-1234-123456789abc`).
5. Move to convention path so `xcrun altool` finds it without `--apiKeyPath`:
   ```bash
   mkdir -p ~/.appstoreconnect/private_keys
   mv ~/Downloads/AuthKey_*.p8 ~/.appstoreconnect/private_keys/
   chmod 600 ~/.appstoreconnect/private_keys/AuthKey_*.p8
   ```

### Sanity-check the credentials before Phase 2

Don't discover an issuer-id typo or expired key during a real upload. Mint a JWT and call `/v1/apps`:

```bash
KEY_ID=ABC123XYZ4
ISSUER=12345678-1234-1234-1234-123456789abc
BUNDLE=com.example.app

uv tool run --with pyjwt --with cryptography python - <<PY
import jwt, json, os, time, urllib.request
key = open(os.path.expanduser("~/.appstoreconnect/private_keys/AuthKey_${KEY_ID}.p8")).read()
tok = jwt.encode(
    {"iss": "${ISSUER}", "exp": int(time.time())+60*15, "aud": "appstoreconnect-v1"},
    key, algorithm="ES256", headers={"kid": "${KEY_ID}", "typ": "JWT"})
req = urllib.request.Request(
    "https://api.appstoreconnect.apple.com/v1/apps?filter[bundleId]=${BUNDLE}",
    headers={"Authorization": f"Bearer {tok}"})
data = json.load(urllib.request.urlopen(req))
print(json.dumps([(a['id'], a['attributes']['name']) for a in data['data']], indent=2))
PY
```

Expected: prints `[["6764402132", "Marrakech Compass"]]` (numeric app ID + display name).

Failure modes:
- `urllib.error.HTTPError: HTTP Error 401: Unauthorized` → wrong issuer ID, wrong key ID, expired key, or `.p8` doesn't match the key ID. Re-verify all three from the Keys tab.
- `[]` empty array → app record doesn't exist for that bundle ID. Create it in ASC web UI before Phase 4.
- `ImportError: jwt`, `ModuleNotFoundError`, or `pip` complaints → install `uv` first (`brew install uv` or `curl -LsSf https://astral.sh/uv/install.sh | sh`). See [OP-34](OPERATOR-PATTERNS.md#op-34-macos-system-python-is-externally-managed-use-uv-for-asc-api-scripting).

Save the numeric app ID — you'll need it for build polling and metadata operations.

See PILOT-AND-TESTFLIGHT.md "Verifying state via API" for the full ASC-REST patterns.

## Xcode + SDK compatibility

- Verify your Xcode version is current enough for ASC's "must build with Xcode N" deadline. Apple announces these annually (usually around iOS major release in September).
- Check current minimum at submission time: ASC → Build will reject if too old.
- Beta Xcode caveats: ASC sometimes accepts beta-Xcode builds for TestFlight but rejects them for App Store. Stick to GA Xcode versions for App Store submissions.
- Deployment target (iOS X.Y in your Xcode project): can be older than the SDK; this is the minimum iOS version users need to install. Most apps target iOS 13.0+ or iOS 15.0+.

## fastlane install (Bundler-managed)

Why Bundler over global gem:
- Version pinning per project (avoid breakage from gem updates)
- CI reproducibility (Gemfile.lock locks fastlane version)
- No `sudo gem install` (system Ruby is read-only on macOS, breaks on `sudo`)

```bash
# .ruby-version
3.2.2  # or whatever stable Ruby

# Gemfile
source "https://rubygems.org"
gem "fastlane", "~> 2.220"
gem "cocoapods", "~> 1.15"  # if using CocoaPods

# install
bundle install --path vendor/bundle
```

Then always: `bundle exec fastlane <lane>` (not naked `fastlane`).

`.gitignore` additions:
```
vendor/bundle/
fastlane/report.xml
fastlane/test_output/
*.p8
keys/
```

## Pre-flight script (optional)

Wire all these checks into a `fastlane preflight` lane that refuses to start later lanes:

```ruby
# fastlane/Fastfile
lane :preflight do
  # Distribution cert
  cert_check = sh("security find-identity -v -p codesigning | grep 'Apple Distribution'", error_callback: ->(_) { false })
  UI.user_error!("No Apple Distribution cert in keychain. Run Xcode → Settings → Accounts → Manage Certificates → + → Apple Distribution.") unless cert_check

  # Bundle ID consistency
  expected = "com.example.app"
  pbx = File.read("../ios/Runner.xcodeproj/project.pbxproj")
  found = pbx.scan(/PRODUCT_BUNDLE_IDENTIFIER = ([\w\.]+)/).flatten.uniq
  UI.user_error!("Bundle ID mismatch in pbxproj: #{found}") unless found == [expected]

  # Privacy manifest
  UI.user_error!("PrivacyInfo.xcprivacy missing") unless File.exist?("../ios/Runner/PrivacyInfo.xcprivacy")

  # Encryption export pre-answer
  info_plist = File.read("../ios/Runner/Info.plist")
  UI.user_error!("ITSAppUsesNonExemptEncryption not set in Info.plist") unless info_plist.include?("ITSAppUsesNonExemptEncryption")

  # ASC API key
  UI.user_error!("ASC API key not at conventional path") unless Dir.glob(File.expand_path("~/.appstoreconnect/private_keys/AuthKey_*.p8")).any?

  UI.success("Pre-flight clean ✓")
end

lane :release do
  preflight  # always run first
  build_app(...)
  upload_to_app_store(...)
end
```

Cheap insurance against discovering a missing cert at archive time.
