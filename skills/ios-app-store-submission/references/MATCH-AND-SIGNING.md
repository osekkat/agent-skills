# match and Code Signing

Centralized cert + profile storage via `fastlane match`. Solo-dev variant: private git repo as the storage backend.

## When to use match vs Xcode automatic signing

| Scenario | Use |
|---|---|
| Single dev, single machine, first submission | Xcode automatic signing (simplest) |
| Multi-machine (work + home + CI) | `match` |
| Multiple devs on same project | `match` |
| Complex entitlements (push, app groups, iCloud, Sign in with Apple) | `match` (manual signing more reliable) |
| Just want to ship and figure it out later | Xcode automatic, then migrate to match if you grow |

Critical caveat for Xcode automatic signing: it has two flavors — *cloud signing* (Apple holds keys, fragile across CLI sessions) and *local signing* (cert in keychain, stable). See [OP-4](OPERATOR-PATTERNS.md#op-4-first-archive-succeeds-via-xcode-cloud-signing-subsequent-archives-fail). Always create a local Distribution cert before relying on automatic signing for repeat builds.

## match init

Storage modes:
- `git` (default, private repo) — recommended for solo dev
- `gcs` (Google Cloud Storage)
- `s3` (AWS)
- `gitlab` (GitLab project secure files)

For solo dev, use a private GitHub repo:

```bash
# Create private repo named e.g. "ios-certificates"
# Then in your project:
bundle exec fastlane match init
```

Answer:
- Storage mode: `git`
- Git URL: `git@github.com:<you>/ios-certificates.git`
- Encryption passphrase: invent one, store in password manager — **never commit**

`Matchfile` contents:

```ruby
git_url("git@github.com:youruser/ios-certificates.git")
storage_mode("git")

# What types of certs/profiles to manage
type("appstore")   # most lanes will pull this

# App identifier (single app or array)
app_identifier(["com.example.app"])
# or for multi-target:
# app_identifier(["com.example.app", "com.example.app.WidgetExtension"])

username("you@example.com")  # Apple ID
team_id("ABCD123456")
```

## Initial cert generation

```bash
# Distribution cert + App Store profile
bundle exec fastlane match appstore

# Development cert + Development profile (for dev builds on personal device)
bundle exec fastlane match development

# Ad-hoc profile (for direct device installs outside TestFlight — rare)
bundle exec fastlane match adhoc
```

First run:
- Prompts for ASC password / API key
- Prompts for match passphrase (the one from init)
- Creates new cert + profile in Apple Developer portal
- Encrypts and pushes to your match git repo
- Installs locally in keychain + provisioning profiles dir

Subsequent runs: pulls existing certs from repo, decrypts, installs. Idempotent.

## Day-to-day sync

```bash
# Sync without creating new (use this 99% of the time)
bundle exec fastlane match appstore --readonly
```

`--readonly` ensures you never accidentally create a new cert (which would invalidate the old one for everyone else / break CI).

When to drop `--readonly`:
- Cert is expired or expiring soon
- Adding a new app ID
- Setting up a new device in development

## CI integration

Required secrets:
- `MATCH_PASSWORD` — the encryption passphrase from init
- `MATCH_GIT_BASIC_AUTHORIZATION` — base64-encoded `user:personal_access_token` for HTTPS git auth, OR set up SSH deploy key

```yaml
# .github/workflows/release.yml
- name: Sync signing
  env:
    MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
    MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
  run: bundle exec fastlane match appstore --readonly
```

Always `--readonly` in CI. Why CI must not generate new certs:
- Race conditions (two CI runs creating certs at the same time)
- Accidental cert revocation (newer match versions revoke conflicting certs)
- Recovering from CI-created certs requires manual cleanup in Apple Developer portal

## Migrating from Xcode automatic to match

After running `match appstore` for the first time:

1. In Xcode project settings: Signing & Capabilities → uncheck "Automatically manage signing"
2. Set Team to your team
3. Set Provisioning Profile to "match AppStore com.example.app"
4. Set Signing Certificate to "Apple Distribution"
5. Repeat for any extension targets

`ExportOptions.plist` for manual signing:
```xml
<key>signingStyle</key>
<string>manual</string>
<key>signingCertificate</key>
<string>Apple Distribution</string>
<key>provisioningProfiles</key>
<dict>
    <key>com.example.app</key>
    <string>match AppStore com.example.app</string>
</dict>
```

## Manual signing fallback (no match)

When `match` is broken or you need to bootstrap without it:

1. **Generate cert manually:**
   - developer.apple.com → Certificates → "+"
   - Type: Apple Distribution (or iOS Distribution for legacy)
   - Generate CSR from Keychain Access → Certificate Assistant → Request Certificate from Certificate Authority
   - Download cert, double-click to install in keychain
   - Export private key: Keychain Access → find cert → right-click → Export → save `.p12`

2. **Generate App Store profile manually:**
   - developer.apple.com → Profiles → "+"
   - Type: App Store
   - Pick App ID, pick the cert from step 1
   - Download `.mobileprovision`, double-click to install

3. **Point gym at manual config:**
   - Edit `ExportOptions.plist` per the manual signing example above

Useful when:
- match repo is corrupted or inaccessible
- You're on a machine where you can't run match (locked-down corp dev, debugging CI)
- You've inherited an existing project with non-match signing setup

## Common signing errors

### `No profiles for 'com.example.app' were found`

**Cause:** No App Store profile registered in Apple Developer portal for this bundle ID, or your local cache is stale.

**Fix:** Either run `bundle exec fastlane match appstore` (drop `--readonly` if first time), or in Xcode let automatic signing regenerate.

### `No signing certificate "iOS Distribution" found`

**Cause:** Distribution cert not in keychain. Could be:
- Never generated (run match)
- Cloud signing only — see [OP-4](OPERATOR-PATTERNS.md#op-4)
- Wrong keychain searched (rare)

**Fix:** `security find-identity -v -p codesigning` to verify. Generate via Xcode UI (PRE-FLIGHT.md "Distribution certificate" section) or `match`.

### `The bundle ID does not match the bundle ID 'com.example.app' in the provisioning profile`

**Cause:** Bundle ID mismatch between Xcode project and provisioning profile.

**Fix:** Verify everywhere (PRE-FLIGHT.md "Bundle ID hygiene"). Update Xcode to match the profile's bundle ID, or regenerate the profile for the correct bundle ID.

### `Your account does not have permission`

**Cause:** ASC API key role too low for the operation, or your Apple Developer account doesn't have the right role on the team.

**Fix:** Check Apple Developer portal → Membership → role. Need at least Admin to manage certs. App Manager API key is fine for most ASC operations but not for cloud signing — see [OP-5](OPERATOR-PATTERNS.md#op-5).

### `The data couldn't be read because it isn't in the correct format`

**Cause:** Corrupt match repo. Often happens when match passphrase is wrong, or the repo was edited externally.

**Fix:** Verify `MATCH_PASSWORD` is correct. If still failing, `match nuke` (destructive — see below) and regenerate.

### Two certs of the same type in keychain

**Symptom:** `security find-identity` shows two `Apple Distribution: <Team>` entries.

**Cause:** match created a new cert without revoking the old one, OR you generated manually and ran match.

**Fix:** Decide which to keep. Revoke the other in Apple Developer portal, then `security delete-certificate -c "Apple Distribution: ..." -t` to remove from keychain.

### Cloud signing intermittent failure (OP-4 pattern)

**Symptom:** First archive succeeds with `Authority=Apple Distribution: <Team>` in `codesign -dvvv`, but subsequent archives fail with `No Accounts` / `No signing certificate "iOS Distribution" found`. `security find-identity` never showed the cert locally.

**Cause:** Xcode used cloud signing for the first archive — Apple holds the private key in cloud, signs on demand when Xcode is logged in. Session lapses (24h tokens, sign-out, network blip) cause subsequent archives to fail because CLI `xcodebuild` can't reach cloud auth context.

**Fix:** Generate a real local Distribution cert (PRE-FLIGHT.md "Distribution certificate" section). Stops depending on cloud signing for repeat builds.

**Prevention:** Always create local Distribution cert before Phase 2. Don't rely on cloud signing for repeat builds.

## match nuke (destructive)

`match nuke` revokes all certs of a given type from Apple Developer portal AND removes them from the match repo. Use when:
- Cert compromised (laptop stolen)
- Wrong cert type generated by mistake
- Recovering from match corruption

```bash
bundle exec fastlane match nuke distribution
bundle exec fastlane match nuke development
```

**Implications:**
- All team members must re-sync (`match appstore`) to get new certs
- CI must run `match` (drop `--readonly` once) to regenerate
- Any local `.mobileprovision` files become invalid
- Existing TestFlight / App Store builds keep working — only future signing breaks

Don't run nuke unless you understand the impact. For solo dev with one machine, the cost is just running `match appstore` again. For teams, expect a half-day of churn.
