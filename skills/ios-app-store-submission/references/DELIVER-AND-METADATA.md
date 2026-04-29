# deliver and App Store Metadata

Two paths to fill App Store metadata: `fastlane deliver` (committed-source-of-truth, multi-locale, repeatable) or direct **App Store Connect REST API** calls (when fastlane is overkill or you're scripting). For first submissions, the ASC web UI is often fastest. Pick once and stick with it.

## Path A: fastlane deliver

`fastlane deliver` pushes metadata, screenshots, and submission to App Store Connect. Treat the metadata directory as source of truth, not the ASC web UI — once you adopt deliver, edits in the web UI will be overwritten on the next push.

### deliver init

```bash
bundle exec fastlane deliver init
```

(uses ASC API key from Appfile/ENV.)

Output directory layout:

```
fastlane/
├── metadata/
│   ├── en-US/
│   │   ├── name.txt
│   │   ├── subtitle.txt
│   │   ├── description.txt
│   │   ├── keywords.txt
│   │   ├── marketing_url.txt
│   │   ├── privacy_url.txt
│   │   ├── support_url.txt
│   │   └── release_notes.txt
│   ├── fr-FR/  (one dir per locale)
│   │   └── ... same files ...
│   ├── copyright.txt
│   ├── primary_category.txt
│   ├── secondary_category.txt
│   └── review_information/
│       ├── demo_user.txt
│       ├── demo_password.txt
│       ├── notes.txt
│       ├── phone_number.txt
│       ├── email_address.txt
│       ├── first_name.txt
│       └── last_name.txt
└── screenshots/
    ├── en-US/
    │   ├── iPhone 6.7 (and 6.5)/
    │   │   ├── 1_Home.png
    │   │   ├── 2_OurPicks.png
    │   │   └── ...
    │   └── iPad Pro (12.9-inch)/
    └── fr-FR/
        └── ...
```

### Required metadata fields

Verify the current required set in ASC at submission time — Apple changes requirements without warning.

| Field | Constraint |
|---|---|
| App name | ≤30 chars |
| Subtitle | ≤30 chars |
| Description | ≤4000 chars |
| Keywords | ≤100 chars total, comma-separated, no leading/trailing whitespace |
| Promotional Text | ≤170 chars (editable without resubmission) |
| Support URL | required, must return HTTP 200 |
| Marketing URL | optional |
| Privacy Policy URL | required for all apps that collect any data; effectively required |
| Primary Category | required |
| Secondary Category | optional |
| Age rating | questionnaire — currently filled in ASC web UI, not deliver (see PRE-FLIGHT.md) |
| Copyright | "YYYY <Entity Name>" format |

### Screenshots

Required device classes change. As of 2026:
- **iPhone 6.7" / 6.9"** (`APP_IPHONE_67`) — required. Accepts 1290×2796 (iPhone 14 Pro Max) and 1320×2868 (iPhone 16 Pro Max). One bucket; submit either size.
- **iPad 13"** (`APP_IPAD_PRO_3GEN_129`) — required only if app supports iPad. 2064×2752 or 2048×2732.

Capture from the iOS Simulator:

```bash
open -a Simulator
xcrun simctl boot "iPhone 16 Pro Max"
flutter run -d "iPhone 16 Pro Max" --release  # or `xcrun simctl launch booted <bundleId>`
# In simulator: Cmd+S captures to ~/Desktop
```

deliver naming convention: `<order>_<name>.png` in the appropriate device folder. Order is the position in the App Store listing.

Generation options:
- **Manual capture** — flexible, requires walking through the app
- **`fastlane snapshot`** — UI test-driven, requires `Snapfile` and a UI test target. Best for >2 locales.
- **`frameit`** — adds device frames to plain captures (optional polish; many apps skip it)
- **Third-party** — `Screenshot.app`, `Picsew` for solo devs who skip snapshot

### App Preview videos (optional)

- Per-device, max 30 seconds each
- Audio considerations: must not contain voiceover that promotes other apps; copyrighted music gets flagged
- deliver supports upload via `app_preview` in metadata directory; less commonly used

### Privacy nutrition labels

Filled via **ASC web UI under App Privacy**. deliver support is evolving but not authoritative as of 2026. The web UI is also the right tool here because the schema has 30+ data-type / purpose / linkage / tracking permutations that are easier to click through than encode.

When SDK dependencies change, update privacy labels — common forgotten step that triggers App Review rejection.

For Flutter apps with Firebase + Mapbox + Geolocator, declare these data types as collected (matching the `PrivacyInfo.xcprivacy` declarations):

| Data Type | Linked to user | Tracking | Purposes |
|---|---|---|---|
| Precise Location | No | No | App Functionality |
| Coarse Location | No | No | App Functionality |
| Crash Data | No | No | App Functionality |
| Performance Data | No | No | App Functionality |
| Product Interaction | No | No | Analytics |
| Other Diagnostic Data | No | No | App Functionality |
| Device ID | No | No | Analytics |

Mark **Tracking: No** for everything unless you actually link to third-party data for ad targeting.

### Submission information

`fastlane/metadata/review_information/`:
- `demo_user.txt` / `demo_password.txt` — demo credentials. Required if app has any login. Reviewers will not create their own accounts.
- `notes.txt` — review notes. Explain non-obvious flows, third-party login providers, server-side feature flags, anything that might confuse a reviewer testing in 5 minutes.
- `phone_number.txt`, `email_address.txt`, `first_name.txt`, `last_name.txt` — Apple's review contact for clarification questions.

For an offline-first travel guide with no login:

```
notes.txt:
The app is a travel guide for visitors to Marrakech, Morocco. No sign-in is required — all features are available immediately on launch. The app works offline using bundled JSON data plus a downloadable Mapbox region for offline maps. Reviewer can test all five tabs (Home, Our Picks, Explore, Map, Tips) without an internet connection or any account setup. The Emergency screen surfaces phrases and embassy contacts that are bundled with the app and always available offline.
```

### Phased release

7-day rollout to App Store users, controlled via `phased_release: true` in deliver config.

When to enable:
- Large user base where catastrophic regression has high blast radius
- Risky update (architecture change, large feature)
- New backend dependency (gives time to scale)

When to disable / pause:
- Crash-spike auto-pause already happened (Apple pauses automatically at 3% crash rate)
- Need to push fix to all users immediately
- Solo dev with small user base — phased rollout barely affects rollout speed perceptibly

### metadata-only mode

Update text/images without re-uploading the binary:

```bash
bundle exec fastlane deliver \
  --skip_binary_upload true \
  --skip_screenshots true \
  --force
```

Most metadata changes don't require a new binary review. Significant copy changes (claims about new features, regulated language) may flag for review.

### Localizations

Add a locale: create `fastlane/metadata/<lang>/`, populate the same files. Apple's locale codes are a subset of standard BCP 47 — verify in ASC.

Localized screenshots: per-language folder under `fastlane/screenshots/<lang>/`. Maintenance cost grows linearly with locales × screenshots per locale × device classes.

## Path B: ASC REST API direct

When fastlane is overkill (single submission, simple metadata, no CI) or you're already in a Python/Ruby/Node script and want to fill metadata without adding fastlane.

### Authentication: ES256 JWT

ASC API uses ECDSA P-256 JWT, NOT the typical HMAC. Key gotcha: the signature must be raw `r || s` (each padded to 32 bytes), not DER. See [OP-23](OPERATOR-PATTERNS.md#op-23-asc-api-jwt-requires-raw-rs-signature-not-der).

Python implementation (no PyJWT dependency, uses `cryptography`):

```python
import base64, json, time, os
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec, utils as asym_utils

KEY_ID = "ABC123XYZ4"
ISSUER_ID = "12345678-1234-1234-1234-123456789abc"
KEY_PATH = os.path.expanduser(f"~/.appstoreconnect/private_keys/AuthKey_{KEY_ID}.p8")

def b64url(d):
    return base64.urlsafe_b64encode(d).rstrip(b"=").decode("ascii")

def make_jwt():
    header = {"alg": "ES256", "kid": KEY_ID, "typ": "JWT"}
    now = int(time.time())
    payload = {"iss": ISSUER_ID, "iat": now, "exp": now + 1200, "aud": "appstoreconnect-v1"}
    h_b = b64url(json.dumps(header, separators=(',', ':')).encode())
    p_b = b64url(json.dumps(payload, separators=(',', ':')).encode())
    msg = (h_b + "." + p_b).encode()
    with open(KEY_PATH, "rb") as f:
        key = serialization.load_pem_private_key(f.read(), password=None)
    der_sig = key.sign(msg, ec.ECDSA(hashes.SHA256()))
    r, s = asym_utils.decode_dss_signature(der_sig)
    raw_sig = r.to_bytes(32, "big") + s.to_bytes(32, "big")
    return h_b + "." + p_b + "." + b64url(raw_sig)
```

Token max lifetime: 20 minutes. Generate fresh per script run.

If using PyJWT (`pip install 'pyjwt[crypto]'`), it handles raw signature internally:

```python
import jwt, time
token = jwt.encode(
    {"iss": ISSUER_ID, "iat": int(time.time()), "exp": int(time.time()) + 1200, "aud": "appstoreconnect-v1"},
    open(KEY_PATH).read(),
    algorithm="ES256",
    headers={"kid": KEY_ID}
)
```

### API key roles

| Role | Use cases | Sufficient for |
|---|---|---|
| **Admin** / **Account Holder** | Cert / profile creation, cloud signing | Everything |
| **App Manager** | Upload, metadata, TestFlight, screenshots, age rating, submit for review | Most automation |
| **Developer** | Read-only mostly | Insufficient for upload |

Cloud signing operations (e.g., `xcodebuild -authenticationKeyPath`) require Admin. App Manager is fine for upload and metadata. See [OP-5](OPERATOR-PATTERNS.md#op-5-cloud-signing-requires-adminaccount-holder-api-key-app-manager-is-insufficient).

### Discovering app + version + localization IDs

```python
# 1. Find app by bundle ID
GET /v1/apps?filter[bundleId]=com.example.app
# → app_id

# 2. Find current version
GET /v1/apps/{app_id}/appStoreVersions?limit=5&filter[platform]=IOS
# → version_id (the one with appStoreState == "PREPARE_FOR_SUBMISSION")

# 3. Find existing localizations
GET /v1/appStoreVersions/{version_id}/appStoreVersionLocalizations?limit=20
# → dict of locale → localization_id

# 4. Find AppInfo (for subtitle, name, privacyPolicyUrl)
GET /v1/apps/{app_id}/appInfos
# → appInfo_id (the editable one with appStoreState == "PREPARE_FOR_SUBMISSION")

GET /v1/appInfos/{appInfo_id}/appInfoLocalizations?limit=20
# → dict of locale → appInfoLocalization_id
```

### What lives where

This is the most common API confusion — fields are split across two resource types:

**`appStoreVersionLocalizations`** (per-version, per-locale):
- `description`
- `keywords`
- `promotionalText`
- `supportUrl`
- `marketingUrl`
- `whatsNew` — **only editable on UPDATE versions, not first submissions** ([OP-16](OPERATOR-PATTERNS.md#op-16-whatsnew-is-not-editable-on-first-submission))

**`appInfoLocalizations`** (per-app, per-locale, persists across versions):
- `name`
- `subtitle`
- `privacyPolicyUrl`
- `privacyPolicyText`
- `privacyChoicesUrl`

**`appStoreVersions`** (per-version):
- `versionString`
- `copyright`
- `releaseType` (`AFTER_APPROVAL` / `MANUAL` / `SCHEDULED`)
- `earliestReleaseDate`

**`apps`** (top-level):
- `contentRightsDeclaration` (`USES_THIRD_PARTY_CONTENT` / `DOES_NOT_USE_THIRD_PARTY_CONTENT`)
- `primaryLocale` — see [OP-14](OPERATOR-PATTERNS.md#op-14-primarylocalefr-fr-forces-french-localization-to-be-filled)

**Build attachment** (relationship, not attribute):
```python
PATCH /v1/appStoreVersions/{version_id}/relationships/build
Body: {"data": {"type": "builds", "id": "{build_id}"}}
```

### Common errors and fixes

| HTTP | Code | Cause | Fix |
|---|---|---|---|
| 400 | `PARAMETER_ERROR.ILLEGAL` on sort | Some endpoints don't allow sort | Drop the sort param ([OP-20](OPERATOR-PATTERNS.md#op-20-asc-api-silently-rejects-sort-param-on-some-endpoints)) |
| 409 | `STATE_ERROR` on whatsNew | First submission can't edit whatsNew | Omit field ([OP-16](OPERATOR-PATTERNS.md#op-16)) |
| 409 | `ENTITY_ERROR.ATTRIBUTE.INVALID.DUPLICATE` on locale | Localization auto-created by ASC | Re-GET, then PATCH instead of POST ([OP-15](OPERATOR-PATTERNS.md#op-15)) |
| 409 | `ENTITY_ERROR.ATTRIBUTE.REQUIRED` on age rating | Apple expanded fields | Add the missing field with default ([OP-17](OPERATOR-PATTERNS.md#op-17)) |
| 409 | `'X' is not a valid value for screenshotDisplayType` | Wrong enum | Use `APP_IPHONE_67` not `APP_IPHONE_69` ([OP-21](OPERATOR-PATTERNS.md#op-21)) |
| 403 | `FORBIDDEN_ERROR` GET ageRatingDeclarations | Resource only allows UPDATE | Discover via PATCH errors ([OP-18](OPERATOR-PATTERNS.md#op-18)) |
| 403 | Cloud signing permission error | API key role too low | Use Admin key, or local cert ([OP-5](OPERATOR-PATTERNS.md#op-5)) |

### Screenshot upload protocol (3 steps)

ASC requires an out-of-band upload pattern for assets. Single POST won't work.

**Step 1 — Reserve:**
```python
POST /v1/appScreenshots
Body: {
    "data": {
        "type": "appScreenshots",
        "attributes": {"fileName": "screenshot1.png", "fileSize": <bytes>},
        "relationships": {
            "appScreenshotSet": {"data": {"type": "appScreenshotSets", "id": <set_id>}}
        }
    }
}
```

Response includes `data.attributes.uploadOperations`: array of `{method, url, length, offset, requestHeaders}`.

**Step 2 — Upload chunks:**
For each operation, raw HTTP `PUT` (or specified method) to the presigned URL with file bytes from `offset` for `length` bytes. Apply `requestHeaders`. **Do NOT include the ASC JWT** — these are S3 / CloudFront presigned URLs that fail with auth headers attached.

```python
for op in upload_operations:
    chunk = file_bytes[op["offset"]:op["offset"] + op["length"]]
    headers = {h["name"]: h["value"] for h in op.get("requestHeaders", [])}
    requests.request(op["method"], op["url"], data=chunk, headers=headers)
```

**Step 3 — Commit:**
```python
PATCH /v1/appScreenshots/{screenshot_id}
Body: {
    "data": {
        "type": "appScreenshots",
        "id": "{screenshot_id}",
        "attributes": {
            "uploaded": True,
            "sourceFileChecksum": "<md5_hex_of_full_file>"
        }
    }
}
```

Without step 3, the screenshot is reserved but not finalized and won't appear. See [OP-22](OPERATOR-PATTERNS.md#op-22-screenshot-upload-is-a-3-step-protocol-not-a-single-post).

### Screenshot display types

| Enum | Use for | Pixel sizes |
|---|---|---|
| `APP_IPHONE_67` | iPhone Pro Max (modern) | 1290×2796, 1320×2868 |
| `APP_IPHONE_65` | iPhone Xs Max / 11 Pro Max (legacy) | 1242×2688, 1284×2778 |
| `APP_IPHONE_61` | iPhone 12 / 13 / 14 / 15 (non-Pro-Max) | 1170×2532, 1179×2556 |
| `APP_IPHONE_58` | iPhone X / Xs (legacy) | 1125×2436 |
| `APP_IPHONE_55` | iPhone 8 Plus etc. (legacy) | 1242×2208 |
| `APP_IPHONE_47` | iPhone SE 2 / 3 (legacy) | 750×1334 |
| `APP_IPAD_PRO_3GEN_129` | 12.9" iPad Pro (modern) | 2064×2752, 2048×2732 |
| `APP_IPAD_PRO_3GEN_11` | 11" iPad Pro | 1668×2388 |

`APP_IPHONE_69` does NOT exist — see [OP-21](OPERATOR-PATTERNS.md#op-21).

You only *need* the largest iPhone bucket (currently `APP_IPHONE_67`) — Apple uses it for all smaller iPhones if no specific set provided. iPad needed only if app supports iPad.

### Age rating field set (snapshot)

See [OP-17](OPERATOR-PATTERNS.md#op-17) for the current snapshot. Apple expands annually; iterate against API errors to discover new fields.

For a 4+ travel guide, all fields default to `"NONE"` (content scales) or `False` (booleans):

```python
PATCH /v1/ageRatingDeclarations/{ard_id}
Body: {
    "data": {
        "type": "ageRatingDeclarations",
        "id": "{ard_id}",
        "attributes": {
            # ... see OP-17 for full field list ...
        }
    }
}
```

### Submitting for review (irreversible)

Once metadata, build, screenshots, and age rating are set, submission to App Review uses the `reviewSubmissions` endpoint:

```python
# Step 1: Create a review submission
POST /v1/reviewSubmissions
Body: {
    "data": {
        "type": "reviewSubmissions",
        "attributes": {"platform": "IOS"},
        "relationships": {
            "app": {"data": {"type": "apps", "id": "{app_id}"}}
        }
    }
}
# → submission_id

# Step 2: Add the App Store version as an item
POST /v1/reviewSubmissionItems
Body: {
    "data": {
        "type": "reviewSubmissionItems",
        "relationships": {
            "reviewSubmission": {"data": {"type": "reviewSubmissions", "id": "{submission_id}"}},
            "appStoreVersion": {"data": {"type": "appStoreVersions", "id": "{version_id}"}}
        }
    }
}

# Step 3: Submit
POST /v1/reviewSubmissions/{submission_id}/actions/submit
```

After submit, version state moves from `PREPARE_FOR_SUBMISSION` to `WAITING_FOR_REVIEW`.

**Do not submit programmatically without explicit user authorization.** This is the irreversible action. The skill's high-blast-radius gate applies here. Always confirm before firing the final submit POST.

### Verifying state via API

```python
GET /v1/apps/{app_id}/appStoreVersions?limit=5&filter[platform]=IOS
# → look at attributes.appStoreState
```

State machine:
- `PREPARE_FOR_SUBMISSION` — draft, not yet submitted
- `WAITING_FOR_REVIEW` — submitted, in queue
- `IN_REVIEW` — Apple is reviewing
- `REJECTED` / `METADATA_REJECTED` — rejected
- `PENDING_DEVELOPER_RELEASE` — approved, waiting for your manual release
- `PROCESSING_FOR_APP_STORE` / `PENDING_APPLE_RELEASE` — releasing
- `READY_FOR_DISTRIBUTION` (formerly `READY_FOR_SALE`) — live on App Store

```python
GET /v1/reviewSubmissions?filter[app]={app_id}&filter[platform]=IOS
# → list of past/current submissions; (no review submissions found) means never submitted
```

## Path C: ASC web UI (recommended for first submissions)

For first submissions, the web UI is faster than building a `Deliverfile` from scratch. The skill's `first-submission` mode router recommends this path. Set up fastlane *after* the first build ships — its real value is repeat releases and CI, not initial setup.

For the web UI:
1. Distribution tab → version 1.0.0
2. Fill description, keywords, subtitle, promotional text, support/marketing/privacy URLs
3. App Privacy → fill nutrition label questionnaire (the click-through is faster than the API equivalent)
4. Age Rating → answer questionnaire (Apple keeps expanding this; web UI always has current schema)
5. Build section → + Add Build → select latest TestFlight build
6. Screenshots → upload per device class
7. Click "Add for Review" when ready

After first submission ships, decide if you want fastlane for repeat releases.
