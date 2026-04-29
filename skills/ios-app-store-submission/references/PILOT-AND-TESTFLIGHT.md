# pilot and TestFlight

Upload to TestFlight, manage tester groups, navigate Beta App Review, verify state via API.

## Authentication: ASC API key (recommended)

Generate at <https://appstoreconnect.apple.com/access/integrations/api> (Keys tab) → "Generate API Key". Download the `.p8` private key — Apple shows it **only once at creation time**. Lose it and you have to revoke + regenerate.

This bypasses Apple ID 2FA prompts and is what makes fastlane CI-friendly. It's the single most important setup step for any non-interactive automation.

### Step-by-step

1. ASC → Users and Access → Integrations → Keys tab → "Generate API Key"
2. Name it descriptively (e.g., "Upload Key", "CI Key", "Cloud Signing Key")
3. Pick the role:
   - **App Manager** — covers TestFlight upload + most metadata work + screenshots + age rating + submit-for-review
   - **Admin** — additionally covers cloud signing, cert / profile management
   - **Developer** — read-only mostly, insufficient for upload
   - **Account Holder** — single-account-only, all permissions

   For a pure upload pipeline, App Manager. For fastlane match / cloud signing operations, Admin (see [OP-5](OPERATOR-PATTERNS.md#op-5)).
4. Generate and download the `.p8` file immediately (one-time download)
5. Note the **Key ID** (10 chars) and **Issuer ID** (UUID at top of Keys page)

### Three values needed

- `key_id` — alphanumeric, e.g. `ABC123XYZ4`
- `issuer_id` — UUID format, e.g. `12345678-1234-1234-1234-123456789abc`
- `key` — the `.p8` file path or contents

### Storage conventions

Store the `.p8` at one of the conventional locations so `xcrun altool` finds it without `--apiKeyPath`:

```bash
mkdir -p ~/.appstoreconnect/private_keys
mv ~/Downloads/AuthKey_*.p8 ~/.appstoreconnect/private_keys/
```

`xcrun altool` searches:
- `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`
- `~/.private_keys/AuthKey_<KEY_ID>.p8`
- `./private_keys/AuthKey_<KEY_ID>.p8`

For backup: 1Password / Bitwarden / encrypted blob. Never commit to git.

For CI: base64-encode and store as a secret:
```bash
base64 -i AuthKey_KEY_ID.p8 | pbcopy  # copy to clipboard
# Paste into GitHub Actions secret as ASC_API_KEY_P8_B64
```

In CI, decode at runtime:
```yaml
- run: |
    mkdir -p ~/.appstoreconnect/private_keys
    echo "${{ secrets.ASC_API_KEY_P8_B64 }}" | base64 -d > ~/.appstoreconnect/private_keys/AuthKey_${KEY_ID}.p8
```

### fastlane integration

Three options (pick one):

```ruby
# Option 1: Appfile (committed, references ENV)
# fastlane/Appfile
app_store_connect_api_key(
  key_id: ENV["ASC_KEY_ID"],
  issuer_id: ENV["ASC_ISSUER_ID"],
  key_filepath: ENV["ASC_KEY_PATH"]
)

# Option 2: Lane-level
# fastlane/Fastfile
lane :release do
  api_key = app_store_connect_api_key(
    key_id: "ABC123XYZ4",
    issuer_id: "...",
    key_filepath: "keys/asc.p8"
  )
  pilot(api_key: api_key, ipa: "build/MyApp.ipa")
end

# Option 3: Per-action --api_key_path flag
# Command line:
bundle exec fastlane pilot upload \
  --api_key_path keys/asc.json \
  --ipa build/MyApp.ipa
```

`keys/asc.json` is fastlane's bundled-key format — generate via `fastlane app_store_connect_api_key_to_json`.

### Key revocation

If compromised: ASC → Keys → Revoke. Generate new key, update CI secret + local config. Old key stops working immediately.

## Bare-bones alternative: `xcrun altool` direct upload

If you skip fastlane entirely (recommended for first submissions and Flutter projects — see [OP-37](OPERATOR-PATTERNS.md#op-37-flutter-app--testflight-in-zero-fastlane-lines)), the four-step flow is:

```bash
# 1. (Optional but recommended) Verify bundle ID + version baked into the IPA
unzip -p build/ios/ipa/MyApp.ipa "Payload/Runner.app/Info.plist" | \
  plutil -p - | \
  grep -E "CFBundleIdentifier|CFBundleVersion|CFBundleShortVersion|MinimumOSVersion"
# Confirms: bundle id matches ASC record, version > previous, build number > previous,
# min iOS version is what you intend. See OP-36.

# 2. Validate (catches ITMS errors locally in 30s — saves 10+ min processing per fix)
xcrun altool --validate-app \
  -f build/ios/ipa/MyApp.ipa \
  -t ios \
  --apiKey ABC123XYZ4 \
  --apiIssuer 12345678-1234-1234-1234-123456789abc

# 3. Upload
xcrun altool --upload-app \
  -f build/ios/ipa/MyApp.ipa \
  -t ios \
  --apiKey ABC123XYZ4 \
  --apiIssuer 12345678-1234-1234-1234-123456789abc
```

`--apiKey` is the alphanumeric Key ID. Don't pass `--apiKeyPath` — altool finds the `.p8` by convention at `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` (and a few other paths above). Keeping the key at the conventional path means commands stay short and portable.

This uploads the binary only. Listing metadata and submit-for-review still require either ASC web UI or your own ASC API calls (see DELIVER-AND-METADATA.md Path B).

### Validation output

Success:
```
2026-04-29 11:18:50.081  INFO: ...
==========================================
VERIFY SUCCEEDED with no errors
==========================================
No errors validating archive at '/abs/path/build/ios/ipa/MyApp.ipa'.
```

Failure (representative — actual errors vary by ITMS code):
```
2026-04-29 11:18:50.081  ERROR ITMS-90683: Missing purpose string in Info.plist
```

ITMS errors are listed in `references/ITMS-ERRORS.md`. Most common at first submission: 90683 (missing usage description), 91053/91056 (privacy manifest), 90208 (invalid bundle structure).

### Upload output

Success:
```
==========================================
UPLOAD SUCCEEDED with no errors
Delivery UUID: aabbccdd-eeff-1122-3344-556677889900
Transferred 64547298 bytes in 3.843 seconds (16.8MB/s, 134.369Mbps)
==========================================
```

Save the Delivery UUID — useful for cross-referencing with ASC inbox emails. The build is now in ASC's processing queue but **does not appear in the API for ~1–3 minutes** (see [OP-35](OPERATOR-PATTERNS.md#op-35-asc-api-build-not-yet-visible-window-build-can-skip-processing-and-go-straight-to-valid)).

### When `altool --upload-app` is the right call vs `pilot upload`

- **altool:** simplest, no fastlane setup, one-shot use, ships with Xcode. Picks up `.p8` from convention path.
- **pilot:** lane integration, post-upload notifications, automatic build-number bump, processing-wait gating, ties into a larger lane.

For first submissions and TestFlight-only flows, `altool` is shorter and has fewer moving parts. Switch to `pilot` once you're doing repeat releases with metadata in version control.

`xcrun altool --validate-app` is still supported but Apple recommends Transporter for validation in some workflows. altool is fine for solo dev — the validation passed/failed signal is what matters.

## pilot upload (fastlane)

```bash
bundle exec fastlane pilot upload \
  --ipa build/MyApp.ipa \
  --api_key_path keys/asc.json \
  --skip_waiting_for_build_processing false \
  --skip_submission true
```

Flags:
- `--skip_waiting_for_build_processing` — `false` waits for ASC to finish processing (5–30 min, sometimes longer); `true` returns immediately after upload.
- `--skip_submission` — `true` uploads to TestFlight only, doesn't push to external Beta App Review. **Recommended:** let humans gate that.
- `--changelog` — visible to TF testers; not the same as App Store "What's New" in version metadata.

In `Fastfile`:
```ruby
lane :beta do
  build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
  pilot(
    skip_waiting_for_build_processing: false,
    skip_submission: true,
    changelog: "Bug fixes and performance improvements"
  )
end
```

## Processing time

- Typical: 5–30 min from upload completion to "Ready to Submit"
- Outliers: up to 1+ hour for first-time submissions or large binaries
- Stuck >2 hours: likely processing failure. Check ASC inbox for an email with ITMS error.
- During processing, the build icon shows a placeholder (see [OP-11](OPERATOR-PATTERNS.md#op-11-asc-build-placeholder-icon-during-processing-is-normal))

What to do while waiting:
- Verify in ASC web UI (don't re-upload — same build number conflicts)
- Set up next steps: screenshots, metadata, App Privacy questionnaire
- Tail your email — Apple sends "Build is ready" or "Build is not ready" notifications

Processing failures:
- Invalid binary
- Missing entitlements that local validation didn't catch
- Signature chain issue
- Privacy manifest issue (sometimes catches what local validation misses)

## Verifying state via API

### What "version" means in ASC API contexts

ASC's API uses "version" to mean two different things across resources, and the schema field name doesn't disambiguate. From [OP-45](OPERATOR-PATTERNS.md#op-45-asc-v1builds-filter--version-is-build-number-prereleaseversionversion-is-marketing-version):

| What you call it | Info.plist key | ASC API field | Builds endpoint filter |
|---|---|---|---|
| Marketing version | `CFBundleShortVersionString` | `preReleaseVersion.version` (relation) | `filter[preReleaseVersion.version]=1.0` |
| Build number | `CFBundleVersion` | `builds.attributes.version` | `filter[version]=1` |

Use the dual filter when looking up a specific build:

```
GET /v1/builds
  ?filter[app]=<APP_ID>
  &filter[preReleaseVersion.version]=1.0
  &filter[version]=1
```

Naked `filter[version]=1.0` searches for builds whose *build number* equals `1.0`, which is unusual and rarely matches.

### Endpoints

Two endpoints to know the build state:

```
# Build processing state
GET /v1/builds?filter[app]={app_id}&sort=-uploadedDate&limit=10
# → data[].attributes.processingState: "PROCESSING" / "VALID" / "FAILED" / "INVALID"
# → data[].attributes.expired: bool (90 days from upload)
# → data[].attributes.version: build number (e.g., "3")
# → data[].attributes.uploadedDate: ISO 8601, e.g. "2026-04-29T03:21:28-07:00"

# App Store version state
GET /v1/apps/{app_id}/appStoreVersions?limit=5&filter[platform]=IOS
# → data[].attributes.appStoreState: "PREPARE_FOR_SUBMISSION" / "WAITING_FOR_REVIEW" /
#                                    "IN_REVIEW" / "READY_FOR_DISTRIBUTION" / etc.
```

Note: drop `sort` param on `appStoreVersions` (see [OP-20](OPERATOR-PATTERNS.md#op-20-asc-api-silently-rejects-sort-param-on-some-endpoints)).

If you don't already know the app's numeric ID, look it up by bundle ID:

```
GET /v1/apps?filter[bundleId]=com.example.app
# → data[0].id is the numeric app ID, data[0].attributes.name is the marketing name
```

### Minting the JWT (ES256)

ASC API requests need a JWT signed with the `.p8` private key:

```python
import jwt, time

KEY_ID    = "ABC123XYZ4"
ISSUER    = "12345678-1234-1234-1234-123456789abc"
KEY_PATH  = "~/.appstoreconnect/private_keys/AuthKey_ABC123XYZ4.p8"

with open(os.path.expanduser(KEY_PATH)) as f:
    private_key = f.read()

token = jwt.encode(
    {"iss": ISSUER, "exp": int(time.time()) + 60 * 15, "aud": "appstoreconnect-v1"},
    private_key,
    algorithm="ES256",
    headers={"kid": KEY_ID, "typ": "JWT"},
)
```

Token rules:
- Algorithm: ES256 (P-256 ECDSA), not RS256.
- Header: `kid` = Key ID, `typ` = JWT.
- Claims: `iss` = Issuer ID, `exp` ≤ 20 minutes ahead (Apple rejects longer; 15 min is a safe ceiling), `aud` = `appstoreconnect-v1`.
- Re-mint per request (or per-loop iteration in long-running scripts) — don't reuse a token close to expiry.
- See [OP-23](OPERATOR-PATTERNS.md#op-23-asc-api-jwt-requires-raw-rs-signature-not-der) if you mint signatures by hand without PyJWT.

### Running the script — `uv` ephemeral env (recommended)

macOS system Python is externally-managed (PEP 668), so `pip install pyjwt` fails. Use `uv` to run the script in a one-shot env that installs deps in milliseconds:

```bash
uv tool run --with pyjwt --with cryptography python /path/to/asc_poll.py
```

First run installs PyJWT + cryptography in ~8ms (cached after). See [OP-34](OPERATOR-PATTERNS.md#op-34-macos-system-python-is-externally-managed-use-uv-for-asc-api-scripting) for fallbacks if `uv` isn't available.

### Polling loop — handles three states

The polling loop must handle "build not yet visible" as a normal first sample, not as failure. Builds can also skip `PROCESSING` and go straight to `VALID` for small/medium IPAs ([OP-35](OPERATOR-PATTERNS.md#op-35-asc-api-build-not-yet-visible-window-build-can-skip-processing-and-go-straight-to-valid)).

```python
#!/usr/bin/env python3
# Poll ASC for a specific build to reach VALID. Use under uv:
#   uv tool run --with pyjwt --with cryptography python asc_poll.py
import json, os, sys, time, urllib.request
import jwt

KEY_ID       = os.environ.get("ASC_KEY_ID",    "ABC123XYZ4")
ISSUER       = os.environ.get("ASC_ISSUER_ID", "12345678-1234-1234-1234-123456789abc")
APP_ID       = os.environ.get("ASC_APP_ID",    "6764402132")
TARGET_BUILD = os.environ.get("TARGET_BUILD",  "3")  # CFBundleVersion you just uploaded
KEY_PATH     = os.path.expanduser(f"~/.appstoreconnect/private_keys/AuthKey_{KEY_ID}.p8")
DEADLINE_SEC = 30 * 60   # generous; outliers exist
POLL_SEC     = 45        # ASC rate-limit-friendly

with open(KEY_PATH) as f:
    private_key = f.read()

def token():
    now = int(time.time())
    return jwt.encode(
        {"iss": ISSUER, "exp": now + 60 * 15, "aud": "appstoreconnect-v1"},
        private_key, algorithm="ES256",
        headers={"kid": KEY_ID, "typ": "JWT"},
    )

def call(path):
    req = urllib.request.Request(
        "https://api.appstoreconnect.apple.com" + path,
        headers={"Authorization": f"Bearer {token()}"},
    )
    with urllib.request.urlopen(req) as r:
        return json.load(r)

deadline = time.time() + DEADLINE_SEC
last_state = None
while time.time() < deadline:
    try:
        builds = call(f"/v1/builds?filter[app]={APP_ID}&limit=10&sort=-uploadedDate")
    except Exception as e:
        # Network blips happen. Don't kill the loop on one bad request.
        print(f"[{time.strftime('%H:%M:%S')}] api error: {e}", flush=True)
        time.sleep(POLL_SEC); continue

    found = next((b for b in builds["data"]
                  if b["attributes"].get("version") == TARGET_BUILD), None)

    if found is None:
        if last_state != "missing":
            print(f"[{time.strftime('%H:%M:%S')}] build {TARGET_BUILD}: not yet visible in ASC", flush=True)
            last_state = "missing"
    else:
        a = found["attributes"]
        state = a.get("processingState")
        if state != last_state:
            print(f"[{time.strftime('%H:%M:%S')}] build {TARGET_BUILD}: {state} "
                  f"uploaded={a.get('uploadedDate')} expired={a.get('expired')}", flush=True)
            last_state = state
        if state == "VALID":
            print(f"[{time.strftime('%H:%M:%S')}] build {TARGET_BUILD} ready for TestFlight", flush=True)
            sys.exit(0)
        if state in ("FAILED", "INVALID"):
            print(f"[{time.strftime('%H:%M:%S')}] build {TARGET_BUILD} failed processing — check ASC inbox", flush=True)
            sys.exit(2)

    time.sleep(POLL_SEC)

print(f"[{time.strftime('%H:%M:%S')}] timeout after {DEADLINE_SEC // 60} min", flush=True)
sys.exit(3)
```

Why `limit=10` not `limit=1`: a parallel upload of a different version (or a slow ingest) can push your target out of a 1-element slice. The 10 most-recent uploads is a safer window.

Why log only state *changes*: per-poll logging produces noise. Only emit when something actually changed.

Why 45s poll interval: ASC rate limits are generous, but every-5s polling is wasteful for a process that takes minutes. 30–60s strikes the right balance.

Sample successful run (Flutter 62MB IPA, build 3, 2026-04-29):
```
[11:21:51] build 3: not yet visible in ASC
[11:25:40] build 3: VALID  uploaded=2026-04-29T03:21:28-07:00  expired=False
[11:25:40] build 3 ready for TestFlight
```

Total wall-clock from `altool --upload-app` exit to VALID: ~5.5 min. `PROCESSING` was never observed in the API.

## Internal vs external testing

| | Internal | External |
|---|---|---|
| Tester limit | ≤100 users | ≤10K users per app |
| Must be on dev team | Yes | No |
| Beta App Review | Not required | Required for first build of each version |
| Time to access | Instant after processing | 24–48h after Beta Review |
| Public link | No | Optional |
| Notification | Email + push | Email + push |

When to skip straight to external: rare for solo dev. Mostly when running a structured private beta with non-team users and you've completed Beta App Review already on a previous version.

## Beta App Review

Lighter than full App Review but uses similar guideline subset:
- 2.1 (App completeness) — most common Beta rejection
- 4.0 (Design) — sometimes flagged for placeholder content
- 5.1.1 (Privacy)
- Sign-in flows that don't work
- Crashes on launch

Common Beta rejections:
- App incomplete (missing core feature, "coming soon" tabs)
- Broken functionality
- Missing demo account when login required
- Crashes during Apple's smoke test
- Privacy manifest issues

First build of every new version triggers Beta review for external groups. Subsequent builds of the same version skip it (only the version metadata change counts as a new "release" for Beta review purposes).

## Tester groups

### Internal

ASC → TestFlight → Internal Testing → "+" → name the group → add testers (must be in your dev team).

`pilot add` / `pilot remove` for scripted invitations:
```bash
bundle exec fastlane pilot add user@example.com --first_name "User" --last_name "Name"
```

### External

ASC → TestFlight → External Testing → "+" → name → "Add" testers by email or "Public Link".

Public links: a URL anyone can use to install the build. Useful for open beta. Limit applies (10K).

Redemption codes: legacy mechanism; rarely used.

### Test plans

Modern TestFlight: define a "test plan" with expected behaviors, areas of focus, known issues. Testers see this when they install. Add via ASC → Test Information.

## TestFlight verification checklist (Phase 5 gate)

What counts as "verified" before App Review submission:

- [ ] Install on at least one real device (not just simulator)
- [ ] App launches without immediate crash
- [ ] Walk primary user flow end-to-end (the most common path a real user would take)
- [ ] Test offline behavior if the app claims to work offline
- [ ] Test with location services denied if the app uses location
- [ ] Test sign-in with the demo account credentials you provided to App Review
- [ ] Check ASC → TestFlight → Crashes panel for new entries (anything pre-existing was probably yours)
- [ ] Check Feedback panel for tester reports
- [ ] Verify the app icon visible on the device home screen matches your design (not the placeholder)

## TestFlight-only mode

When the right mode (`testflight-only` per the skill router):
- Early beta — gathering feedback before committing to public release
- Paid beta groups — TestFlight is fine for paid early access
- Internal QA pipeline — ship to internal testers continuously, only push App Store when stable

Iterate without Store release pressure: bump build number per upload, push to internal group, ship feedback loop.

When to graduate to App Store: when crash-free rate is acceptable + tester feedback positive + product is feature-complete enough that App Review won't reject for incompleteness.

## Version trains (TestFlight ≠ independent of App Store)

A common misconception: "TestFlight is just a beta channel, I can iterate freely on the same version number." **Wrong.** TestFlight builds attach to a *version train*, and the train's state on the App Store ladder controls whether you can keep adding builds.

A "version train" is the set of builds attached to a single `CFBundleShortVersionString`. Examples: `2.5.0` is one train, `2.5.1` is another. Within a train, you can upload many builds (`CFBundleVersion: 1, 2, 3, ...`) — TestFlight tracks them all.

### When a train is open (you can keep uploading TestFlight builds)

- Version exists in ASC in `PREPARE_FOR_SUBMISSION` state (never submitted)
- Version was submitted and is `IN_REVIEW`
- Version was rejected and is `DEVELOPER_REJECTED` / `INVALID_BINARY` / awaiting fix
- Version was approved but you set Manual Release and chose NOT to release yet — *partially open*: you can keep uploading TestFlight builds until you actually release

### When a train is closed (Apple refuses new uploads, even TestFlight)

- Version is `READY_FOR_DISTRIBUTION` (released to the App Store)
- Version was approved + auto-released
- Version is `REMOVED_FROM_SALE` (you pulled it post-launch — still closed)

The first time this bites: you ship `1.0.0`, get approved, release. You then want to push a TestFlight-only build of `1.0.0` for QA before bumping to `1.1.0`. Apple says no — `1.0.0` is closed. You must use a higher version (`1.0.1`, `1.1.0`, etc.) for any new TestFlight build.

### Symptoms when you hit it

ITMS-90062 + ITMS-90186 in the upload log (both fire together — see `ITMS-ERRORS.md`):

```
- This bundle is invalid. The value for key CFBundleShortVersionString [1.0.0] in the
  Info.plist file must contain a higher version than that of the previously approved
  version [1.0.0]. (90062)
- Invalid Pre-Release Train. The train version '1.0.0' is closed for new build
  submissions. (90186)
```

For EAS users: this surfaces only in the EAS submission web log, not the local CLI output (see OP-29, OP-30).

### Fix

Bump `CFBundleShortVersionString`. Patch bump (`1.0.0` → `1.0.1`) is the typical choice for "I just want to push a new TestFlight build of essentially the same code." Minor or major bumps are warranted only when the changes justify them.

After the bump:
- Build number can either reset to `1` (cleaner ASC display) or continue from where it left off (`autoIncrement: true` in EAS will continue)
- The new version starts a fresh train; you can iterate TestFlight builds within it freely until *that* version gets approved + released

### Pre-flight check to avoid this

Before any submit, verify your build's version is strictly greater than the most recent `READY_FOR_DISTRIBUTION` (or any approved-state) version in ASC:

```bash
# Latest approved version
curl -H "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/apps/$APP_ID/appStoreVersions?filter[appStoreState]=READY_FOR_DISTRIBUTION&limit=1" \
  | jq '.data[0].attributes.versionString'
```

Or in the ASC web UI: My Apps → <App> → App Store tab → "Released" version. Compare to the version you're about to upload.
