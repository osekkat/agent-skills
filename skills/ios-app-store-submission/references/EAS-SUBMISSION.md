# EAS Build + Submit

Expo Application Services (EAS) is a hosted build + submit pipeline. For Expo (and bare-React-Native projects opted in to EAS), it collapses Phases 1–4 of the skill (framework build → archive → validate → upload) into two CLI calls. Fastlane is not required.

This reference is the authoritative path for **Expo apps** and an alternative for **bare React Native apps** that already have EAS configured. For native (Swift/UIKit) apps, fastlane remains the right tool — EAS doesn't add value when you already own the Xcode workspace.

## When to choose EAS over fastlane

| Situation | EAS | Fastlane |
|---|---|---|
| Expo managed workflow (no `ios/` dir) | ✅ required | ✗ not applicable |
| Expo bare workflow (has `ios/` dir) | ✅ recommended | ✅ works |
| Bare React Native, EAS configured | ✅ already wired | ✅ alternative |
| Bare React Native, no EAS | ✗ extra setup | ✅ recommended |
| Native Swift/UIKit | ✗ wrong tool | ✅ recommended |
| Need self-hosted CI runners | ✗ EAS is hosted | ✅ |
| Need air-gapped builds | ✗ EAS is cloud | ✅ |

EAS = pay-per-build hosted convenience. Fastlane = own-your-pipeline portability. They don't compose; pick one per project.

## EAS pipeline mapping

EAS collapses the skill's phases:

| Skill phase | EAS equivalent |
|---|---|
| Phase 0 (pre-flight) | Same — Apple Developer enrollment, ASC record, bundle ID hygiene still required |
| Phase 1 (framework build) | Done implicitly inside `eas build` (cloud worker runs `npx expo prebuild` + Xcode archive) |
| Phase 2 (archive `gym`) | `eas build --platform=ios --profile=production` |
| Phase 3 (validate) | Implicit; ASC validation runs at submit time |
| Phase 4 (upload `pilot`) | `eas submit --platform=ios --latest` |
| Phase 5 (TestFlight verify) | Same — install on real device |
| Phase 6 (metadata) | EAS does NOT cover. Use `fastlane deliver`, ASC REST API, or web UI |
| Phase 7 (submit for review) | EAS does NOT cover. Use ASC web UI (recommended for solo dev) or ASC REST API |

**Critical:** EAS handles **build + TestFlight upload**. It does **not** handle App Store metadata, screenshots, age rating, or submit-for-review. For those, you fall back to one of the Phase 6/7 paths in `DELIVER-AND-METADATA.md`.

## One-time setup

```bash
npm install -g eas-cli       # or use npx
eas login                    # prompts for Expo account
eas build:configure          # creates eas.json if missing
```

EAS uses your **Expo account**, not your Apple Developer account directly. On the first iOS build, EAS prompts to upload your Apple ID credentials (it stores Distribution cert + provisioning profile + ASC API key on EAS servers, encrypted, and re-uses them for subsequent builds).

### Apple credentials EAS manages

After the first build:

```
✔ Using remote iOS credentials (Expo server)

Distribution Certificate
Serial Number             44DB17044344FA49E4200D0FEEF45C15
Expiration Date           Fri, 11 Dec 2026 12:06:53 GMT
Apple Team                ABCD123456 (Jane Developer (Individual))

Provisioning Profile
Developer Portal ID       J3JL6898JD
Status                    active
Expiration                Fri, 11 Dec 2026 12:06:53 GMT
```

The Distribution cert and App Store provisioning profile live on EAS's servers, not your local keychain. This is the **opposite** of OP-4's "must have local cert" rule for fastlane workflows: EAS-managed apps don't need a local Distribution cert because EAS does the signing on its workers.

To inspect or rotate: `eas credentials --platform ios` (interactive). To force EAS to re-fetch from Apple: delete the Distribution cert in `developer.apple.com` → Certificates, then run `eas build` again — EAS will regenerate.

### `eas.json` reference

Minimal `eas.json` for a typical Expo iOS pipeline:

```json
{
  "cli": {
    "version": ">= 16.28.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "1234567890"
      }
    }
  }
}
```

Field-by-field:

- `cli.version` — minimum eas-cli version. Bump when relying on a new flag.
- `cli.appVersionSource` — `"remote"` means **build numbers** (`CFBundleVersion`) are stored on EAS servers and auto-incremented; `"local"` reads from `app.config.js` `ios.buildNumber`. **Important:** this only governs the build number, not the version (`CFBundleShortVersionString`) — see OP-29.
- `build.production.autoIncrement` — `true` bumps the build number on each build. With `appVersionSource: "remote"`, EAS authoritatively tracks the next number.
- `submit.production.ios.ascAppId` — App Store Connect numeric app ID, found in the ASC URL `https://appstoreconnect.apple.com/apps/<ascAppId>/...`. **Required for non-interactive `eas submit`** — see OP-28.

## Phase 0 — Pre-flight (EAS variant)

The skill's Phase 0 still applies, with these EAS-specific additions:

- `eas whoami` succeeds (Expo account is authenticated)
- `eas.json` exists at the project root
- `app.config.js` (or `app.json`) has correct `ios.bundleIdentifier`, `version`, and Expo `extra.eas.projectId`
- For `update` mode: ASC App ID known and added to `eas.json` `submit.production.ios.ascAppId` (OP-28)
- For `first-submission` mode: ASC record exists with bundle ID matching `app.config.js`
- `ITSAppUsesNonExemptEncryption: false` set under `ios.infoPlist` in `app.config.js` (Expo translates this into the built `Info.plist` automatically — verified working in SDK 54)

Verify the version field in `app.config.js` is **higher than the latest approved App Store version**. EAS will not catch this — it only surfaces at the Apple-side upload step (see ITMS-90062 in `ITMS-ERRORS.md` and OP-30).

```bash
# From inside the project
eas whoami
grep -E '"version"|bundleIdentifier' app.config.js app.json 2>/dev/null
eas build:list --platform=ios --limit=5 --json | grep -E '"appVersion"|"appBuildVersion"|"completedAt"' | head -20
```

## Phase 2 — `eas build`

```bash
eas build --platform=ios --profile=production --non-interactive --no-wait
```

Flags worth knowing:

- `--platform=ios` — explicit; `--platform=all` builds both
- `--profile=production` — references the named profile in `eas.json`
- `--non-interactive` — fails fast if any prompt would appear (CI-friendly)
- `--no-wait` — queue the build and return immediately. Without it, the CLI streams logs and blocks for the entire build duration (5–25 min). Use `--no-wait` then poll with `eas build:view` so you can do other work.
- `--message "..."` — annotation visible in the EAS dashboard

The CLI prints a build ID + dashboard URL on submission. Save the build ID for later commands:

```
✔ Uploaded to EAS
✔ Computed project fingerprint

See logs: https://expo.dev/accounts/<account>/projects/<slug>/builds/<build-id>
```

### Polling for build completion

Use `eas build:view <build-id> --json` to query state. **Do not** name your shell variable `status` in zsh — it's a read-only built-in (see OP-31).

```bash
build_id="<id>"
until s=$(eas build:view "$build_id" --json 2>/dev/null \
        | grep '"status"' | head -1 | sed 's/.*: "//;s/",.*//') \
      && [[ "$s" == "FINISHED" || "$s" == "ERRORED" || "$s" == "CANCELED" ]]; do
  echo "[$(date +%H:%M:%S)] status=$s"
  sleep 30
done
echo "TERMINAL: $s"
```

States to expect:
- `NEW` → `IN_QUEUE` → `IN_PROGRESS` → `FINISHED`
- Or terminal failures: `ERRORED`, `CANCELED`

Typical durations (m4-class Apple Silicon EAS workers, mid-2026):
- Queue wait: 30s–2 min
- Build duration: 5–7 min for an Expo SDK 54 app with ~50 modules
- Total: ~6–10 min start to FINISHED

### Build artifacts

The build's artifacts URL is in the `eas build:view` JSON output:

```
"buildUrl": "https://expo.dev/artifacts/eas/<hash>.ipa",
"applicationArchiveUrl": "https://expo.dev/artifacts/eas/<hash>.ipa"
```

**Artifacts expire after 30 days.** Visible as `expirationDate` in the JSON. Practical implication: if you build but delay submission >30 days, you must rebuild. See OP-32.

### Inspecting an EAS-built IPA locally

```bash
# Download
curl -sL -o app.ipa "<applicationArchiveUrl>"

# Unpack
mkdir extracted && cd extracted && unzip -q ../app.ipa

# Verify Info.plist
plutil -p Payload/*.app/Info.plist | grep -E 'CFBundleVersion|CFBundleShortVersionString|CFBundleIdentifier|MinimumOSVersion|ITSApp|Usage'

# Verify provisioning profile
security cms -D -i Payload/*.app/embedded.mobileprovision \
  | plutil -extract Name xml1 -o - -
# Expected for Expo-managed: "*[expo] com.example.app AppStore <date>"

# Verify code signing
codesign -dvvv Payload/*.app 2>&1 | head -10
# Authority should be: Apple Distribution: <Team Name> (<TEAM_ID>)
```

The Expo-generated provisioning profile name has an asterisk prefix (`*[expo] ...`) indicating wildcard-app-id support — this is normal, not a placeholder.

## Phase 4 — `eas submit`

```bash
eas submit --platform=ios --latest --non-interactive
```

`--latest` picks the most recent FINISHED build. Alternatives:
- `--id <build-id>` — submit a specific build
- `--path <local-ipa>` — upload an IPA you built outside EAS
- `--url <ipa-url>` — upload an IPA hosted somewhere accessible

EAS uploads to App Store Connect using the EAS-managed ASC API key. **No local `.p8` file is required** — the key lives on EAS servers and is shown in CLI output as e.g. `Key Name: [Expo] EAS Submit mE64klxnNv`.

### `eas submit` non-interactive prerequisites

Non-interactive submit fails immediately if `submit.production.ios.ascAppId` is not in `eas.json`:

```
Set ascAppId in the submit profile (eas.json) or re-run this command in interactive mode.
Submission failed
```

See OP-28. Find your ASC App ID in the URL `https://appstoreconnect.apple.com/apps/<NUMBER>/...` and add it under `submit.production.ios.ascAppId`.

### Reading EAS submit errors

The local CLI is opaque about Apple-side errors:

```
✔ Scheduled iOS submission
- Submitting
✖ Something went wrong when submitting your app to Apple App Store Connect.
```

Even with `--verbose --verbose-fastlane`, the **actual Apple error is not printed to stdout** — the upload happens on EAS workers, and only summarized status comes back. The full ITMS error and fastlane log are at:

```
https://expo.dev/accounts/<account>/projects/<slug>/submissions/<submission-id>
```

The submission ID is in the CLI output (`Submission details: ...`). Open it in your browser, expand the "Upload to App Store Connect" job, and read the `Errors:` section at the bottom. See OP-29.

### Common `eas submit` failures

| Symptom | Cause | Fix |
|---|---|---|
| `Set ascAppId in the submit profile` | Missing `submit.production.ios.ascAppId` | Add to `eas.json`, OP-28 |
| `Something went wrong when submitting` | Some Apple-side error; details at submission URL | Open submission URL in browser, OP-29 |
| ITMS-90062 + ITMS-90186 in submission log | Version already approved on App Store | Bump `version` in `app.config.js`, rebuild, OP-30 |
| ITMS-90478 in submission log | Build number duplicate | Bump build number (`autoIncrement` should handle; if disabled, increment manually) |

### What `eas submit` does NOT do

- **Submit for App Store review.** EAS only uploads to TestFlight. Review submission is a separate step in the ASC web UI or via the ASC REST API. This is the right default — most teams don't want CI auto-submitting for review.
- **Configure metadata** (name, description, keywords, screenshots). Use `fastlane deliver` (Path A in `DELIVER-AND-METADATA.md`) or the ASC web UI.
- **Add testers to TestFlight groups.** Use `--groups <group-name>` flag on `eas submit` for pre-existing internal groups, or manage manually in ASC.

## Mode router for EAS

| Mode | EAS variant |
|---|---|
| `first-submission` | `eas build` → `eas submit` (lands in TF) → fill metadata in ASC web UI → submit for review in ASC web UI |
| `update` | `eas build` → `eas submit` (lands in TF). Then ASC web UI "+ Version" if shipping to App Store, or stop here for TF only |
| `testflight-only` | `eas build` → `eas submit`. Stop. **Note:** version still must be > latest approved App Store version (OP-30) |
| `metadata-only` | EAS not involved. Use `fastlane deliver` or ASC web UI |
| `resubmission-after-rejection` | `eas build` (after fixing root cause) → `eas submit`. Address rejection in ASC web UI when resubmitting for review |
| `xcode-version-rebuild` | `eas build` (EAS workers update Xcode automatically; verify the build profile pins `image: "latest"` or the Xcode version you want) |

## Version and build number management

The two numbers Apple cares about:

- **`CFBundleShortVersionString`** — user-visible version, e.g. `2.5.1`. Comes from `app.config.js` `version` field. **Always read from local config**, regardless of `appVersionSource` setting (see OP-29).
- **`CFBundleVersion`** — internal build number, e.g. `41`. With `appVersionSource: "remote"` and `autoIncrement: true`, EAS manages this.

To bump version: edit `app.config.js`:
```js
expo: {
  version: "2.5.0" → "2.5.1"
}
```

Then `eas build --platform=ios --profile=production`. Build number auto-increments to next.

To force a build-number reset (rare): `eas build:version:set --platform=ios --build-version <n>` (interactive).

### Implication for OTA updates

Expo's `runtimeVersion.policy: "appVersion"` couples OTA update compatibility to `version`. Bumping `version` from `2.5.0` to `2.5.1` means:
- OTA updates published before this build won't apply to 2.5.1 clients
- OTA updates published after this build target only 2.5.1 clients

For a pure native version bump (no JS changes), this means OTA updates can't roll out to the new build until you publish at least one update for the new runtime version. Usually fine, but worth knowing if you depend on OTA for hotfixes.

See OP-33.

## Migrating an EAS project to fastlane

If you outgrow EAS (cost, runner control, custom signing requirements):

1. `npx expo prebuild --platform ios --clean` — generates a managed `ios/` directory
2. Add `Gemfile`, `fastlane init` in `ios/` — see `FASTLANE-SETUP.md`
3. Export Apple credentials from EAS:
   - Distribution cert: `eas credentials --platform ios` → "Use existing Distribution Certificate" → "Download .p12 file"
   - Provisioning profile: download from `developer.apple.com` → Profiles
4. Set up `fastlane match` (see `MATCH-AND-SIGNING.md`)
5. Decide on the ASC API key — generate a new one in ASC (the EAS-generated one is locked to EAS servers)
6. Build/submit lane in `Fastfile` per `SKILL.md` Quick Start
7. Once verified, you can `eas build:cancel` any pending builds and stop using EAS for iOS

The reverse migration (fastlane → EAS) is rare and usually means starting from a clean Expo project.

## Cost considerations

- EAS Free tier: 30 builds/month for personal projects (subject to current pricing — verify at https://expo.dev/pricing)
- EAS Production: priced per build minute
- Builds count even if they fail (cloud minutes consumed)
- iOS builds typically 5–8 min each on m4-class workers
- High-iteration projects can hit the free tier ceiling fast — use `--profile preview` or `--profile development` for non-store builds, or build locally with `npx expo run:ios` for dev iteration

## Common EAS-specific gotchas

See operator patterns:
- OP-28 — `eas submit` non-interactive needs `ascAppId`
- OP-29 — EAS submit errors are opaque locally; web URL has the truth
- OP-30 — Version-train closure rejects re-uploads for already-approved versions
- OP-31 — zsh `status` reserved variable in polling loops
- OP-32 — EAS build artifacts expire 30 days after build
- OP-33 — `runtimeVersion.policy: "appVersion"` couples OTA to native version
