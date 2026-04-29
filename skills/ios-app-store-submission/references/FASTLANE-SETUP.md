# fastlane Setup

Bundler-managed install and recommended lane structure for solo iOS dev.

**When fastlane is the right call vs alternatives:**
- ✅ You ship more than 2 releases per year — lane composition pays off
- ✅ You have CI you want to drive releases from
- ✅ Multiple targets (app + extensions + watch app)
- ⚠️ First submission only — the ASC web UI is faster than building a `Deliverfile` from scratch. Set up fastlane *after* the first build ships.
- ⚠️ Tiny utility you'll release once and forget — `xcrun altool --upload-app` + ASC web UI is fine.

## Install via Bundler

Why Bundler over global gem:
- Version pinning per project (avoid breakage from `gem update`)
- CI reproducibility (`Gemfile.lock` locks fastlane version exactly)
- No `sudo gem install` (system Ruby is read-only on macOS)

`Gemfile`:
```ruby
source "https://rubygems.org"
gem "fastlane", "~> 2.220"
gem "cocoapods", "~> 1.15"  # if using CocoaPods
```

`.ruby-version`:
```
3.2.2
```

(Pick a stable Ruby version. Don't use system Ruby on macOS — it's read-only and breaks on `bundle install`.)

```bash
bundle install --path vendor/bundle
```

`.gitignore` additions:
```
vendor/bundle/
fastlane/report.xml
fastlane/test_output/
*.p8
keys/
```

`bundle exec fastlane <lane>` discipline (vs naked `fastlane`) — ensures the project's pinned version is used, not whatever's globally installed.

## Initial init

```bash
bundle exec fastlane init
```

Mode choice:
- "Manual setup" — recommended for solo dev. Skips the auto-config of `deliver` and lets you compose lanes from scratch.
- "Automate beta distribution" / "Automate App Store distribution" — sets up `pilot` / `deliver` interactively. Useful if you want guided setup for first-time fastlane.

`Appfile` contents:
```ruby
app_identifier "com.example.app"
apple_id ENV["FASTLANE_USER"]              # don't commit personal Apple ID
team_id "ABCD123456"                       # 10-char team ID
itc_team_id ENV["FASTLANE_ITC_TEAM_ID"]    # numeric, optional, auto-resolved if not set
```

Never commit `Appfile` if it has personal Apple ID. Use ENV vars + a `.env.default` (committed) and `.env.local` (gitignored), via fastlane's built-in dotenv support:

```ruby
# Appfile
app_identifier "com.example.app"
apple_id ENV["FASTLANE_USER"]
team_id ENV["FASTLANE_TEAM_ID"]
```

```
# .env.default (committed)
FASTLANE_TEAM_ID=ABCD123456

# .env.local (gitignored)
FASTLANE_USER=you@example.com
ASC_KEY_ID=ABC123XYZ4
ASC_ISSUER_ID=...
ASC_KEY_PATH=~/.appstoreconnect/private_keys/AuthKey_ABC123XYZ4.p8
```

Run with `--env local` to load `.env.local`:
```bash
bundle exec fastlane release --env local
```

## Recommended Fastfile lanes

```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  before_all do
    setup_circle_ci if is_ci
    ensure_git_status_clean unless is_ci
  end

  # Pre-flight check (optional but recommended)
  lane :preflight do
    cert_check = sh("security find-identity -v -p codesigning | grep 'Apple Distribution'", error_callback: ->(_) { false })
    UI.user_error!("No Apple Distribution cert. Run Xcode → Settings → Accounts → Manage Certificates → +.") unless cert_check
    UI.user_error!("PrivacyInfo.xcprivacy missing") unless File.exist?("../ios/Runner/PrivacyInfo.xcprivacy")
    UI.success("Pre-flight clean ✓")
  end

  # Sync signing
  lane :sync_signing do |options|
    match(
      type: "appstore",
      readonly: options[:readonly] || is_ci,
      api_key_path: ENV["ASC_KEY_PATH"]
    )
  end

  # Build IPA
  lane :build do
    sync_signing
    increment_build_number(xcodeproj: "ios/Runner.xcodeproj")
    build_app(
      workspace: "ios/Runner.xcworkspace",
      scheme: "Runner",
      configuration: "Release",
      export_method: "app-store-connect",
      output_directory: "build",
      include_symbols: true,
      clean: true
    )
  end

  # TestFlight upload
  lane :beta do
    preflight
    build
    pilot(
      api_key_path: ENV["ASC_KEY_PATH"],
      skip_waiting_for_build_processing: false,
      skip_submission: true,
      changelog: prompt(text: "TestFlight changelog: ")
    )
    slack(message: "Beta build #{lane_context[SharedValues::BUILD_NUMBER]} uploaded") if ENV["SLACK_URL"]
  end

  # Full release pipeline
  lane :release do
    preflight
    build
    pilot(
      api_key_path: ENV["ASC_KEY_PATH"],
      skip_waiting_for_build_processing: false,
      skip_submission: true
    )
    deliver(
      api_key_path: ENV["ASC_KEY_PATH"],
      submit_for_review: false,  # default false; set true when ready
      automatic_release: false,
      force: true,
      skip_screenshots: true,    # screenshots updated separately
      skip_metadata: false
    )
  end

  # Screenshots only
  lane :screenshots do
    snapshot
    deliver(
      skip_binary_upload: true,
      skip_metadata: true,
      skip_screenshots: false,
      api_key_path: ENV["ASC_KEY_PATH"],
      force: true
    )
  end

  # Metadata only (description, keywords changed but not the binary)
  lane :refresh_metadata do
    deliver(
      skip_binary_upload: true,
      skip_screenshots: true,
      skip_metadata: false,
      api_key_path: ENV["ASC_KEY_PATH"],
      force: true
    )
  end

  # Bump build number (commit + tag)
  lane :bump_build do
    increment_build_number(xcodeproj: "ios/Runner.xcodeproj")
    build_num = lane_context[SharedValues::BUILD_NUMBER]
    sh("git", "commit", "-am", "Bump build to #{build_num}")
    add_git_tag(tag: "build-#{build_num}")
  end

  error do |lane, exception|
    slack(message: "Lane #{lane} failed: #{exception.message}", success: false) if ENV["SLACK_URL"]
  end
end
```

Lane composition principles:
- Small, named lanes that compose. `release` calls `preflight` + `build` + `pilot` + `deliver`.
- `before_all` for env validation and CI setup.
- `error` block for cleanup / Slack/Discord notification on failure.
- `lane_context` for cross-action data (build number, IPA path, version).
- `dotenv` integration for environment-specific config (`fastlane <lane> --env staging`).

## Plugins

```bash
bundle exec fastlane add_plugin <name>
```

`Pluginfile` is auto-managed. Useful plugins for solo iOS dev (no version endorsement; check current at install):

- `fastlane-plugin-versioning` — semantic version bumps tied to git
- `fastlane-plugin-firebase_app_distribution` — push beta builds to Firebase distribution alongside TestFlight
- `fastlane-plugin-sentry` — upload dSYMs to Sentry post-build
- `fastlane-plugin-badge` — add version badges to icons for beta builds
- `fastlane-plugin-changelog` — manage changelog file for App Store "What's New"

## CI-vs-local config

```ruby
# Detecting CI
is_ci  # built-in; true on GitHub Actions, CircleCI, etc.

# Read-only match in CI
match(type: "appstore", readonly: is_ci)

# KEYCHAIN handling — CI needs keychain unlocked
setup_circle_ci if is_ci  # also handles keychain on CircleCI
# For GitHub Actions, use:
create_keychain(...) if is_ci
```

ASC API key path resolution priority:
1. Lane-level `app_store_connect_api_key` action result
2. Environment var `APP_STORE_CONNECT_API_KEY_PATH`
3. `Appfile`'s `app_store_connect_api_key`
4. Conventional path `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`

For CI, prefer setting `APP_STORE_CONNECT_API_KEY_PATH` to a workspace-relative path you write at runtime:

```yaml
- name: Setup ASC API key
  run: |
    mkdir -p .keys
    echo "${{ secrets.ASC_API_KEY_P8_B64 }}" | base64 -d > .keys/asc.p8
- name: Build and upload
  env:
    APP_STORE_CONNECT_API_KEY_KEY_ID: ${{ secrets.ASC_KEY_ID }}
    APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
    APP_STORE_CONNECT_API_KEY_KEY: <(cat .keys/asc.p8)
  run: bundle exec fastlane release
```

## When fastlane fights you

`fastlane gym` failing with cryptic error → drop to raw `xcodebuild` (see GYM-AND-ARCHIVE.md). The actionable error is in xcodebuild's output, not gym's summary.

`fastlane match` saying "could not find profile" → run with `--verbose` for the underlying API error. Often it's an Apple Developer portal issue (cert revoked, API key expired) that match's wrapper hides.

`fastlane pilot` upload stuck or timing out → fall back to `xcrun altool --upload-app` directly (see PILOT-AND-TESTFLIGHT.md). Same outcome, simpler error path.

`fastlane deliver` rejecting metadata for unclear reasons → log into ASC web UI, find the version, fix manually. Then `bundle exec fastlane deliver download_metadata` to pull the corrected metadata back into your `fastlane/metadata/` directory.
