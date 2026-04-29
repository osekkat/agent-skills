# agent-skills

*By [Oussama Sekkat](https://x.com/osekkat)*

A growing collection of [Claude Code](https://www.anthropic.com/claude-code) skills, distilled from real engineering work.

These are *Claude Code* skills, not [claude.ai skills](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills) — different format, different install path. Each skill is a directory with a `SKILL.md` (YAML frontmatter + markdown body) plus a `references/` folder of supporting docs the agent loads on demand.

## Why this exists

App Store submission is a domain where Claude has read all of Apple's docs but doesn't have the tacit knowledge that comes from shipping. This skill encodes the scars from real first submissions — captured as session-mined "operator patterns" in the format below — so the next agent (yours or someone else's) doesn't have to earn them again.

## Example operator patterns

Three of the 47 [operator patterns](./skills/ios-app-store-submission/references/OPERATOR-PATTERNS.md) currently in the iOS skill, in the same format used throughout the catalog:

### OP-1: Flutter scaffold default icon ships through validation

**Symptom:** Build validates clean (`xcrun altool --validate-app` returns no errors), uploads to ASC clean, processes to "Ready to Submit". Apple's automated checks pass. But the App Icon visible to App Review is the default Flutter logo — pale blue chevron on white. App Review rejects under Guideline 4.0 "Design — placeholder content."

**Root cause:** `flutter create` ships default Flutter chevron PNGs at all required icon sizes. Xcode treats them as "icons present" because the asset catalog is well-formed. The validator never compares the bytes against known framework defaults.

**Fix:** Verify the 1024×1024 source visually before every upload. If it's the Flutter chevron, regenerate via `flutter_launcher_icons`.

**Prevention:** Pre-Phase-2 lane: MD5-hash the 1024 icon and fail if it matches the known Flutter default.

**Source:** First submission of a Flutter project, 2026-04.

### OP-30: Apple closes a "version train" after approval — TestFlight is not exempt

**Symptom:** Version `2.5.0` is approved + released. You build a new TestFlight-only build of the *same* app at `2.5.0` (intent: keep iterating internal QA without bumping the public version). EAS submit / `pilot upload` / Transporter all fail with `ITMS-90062 + ITMS-90186` as a pair.

**Root cause:** Apple models versions as "trains" — the set of builds attached to one `CFBundleShortVersionString`. Once a train reaches "approved" state, it **closes** for new uploads. Apple does not distinguish TestFlight uploads from App Store uploads at this layer; both attach to the train, both are rejected once it's closed. This catches teams who model TestFlight as an independent beta channel. It isn't.

**Fix:** Bump `CFBundleShortVersionString` (e.g. `2.5.0` → `2.5.1`) and rebuild.

**Prevention:** Phase 0 pre-flight queries ASC for the latest approved version and refuses to start a build at the same or lower number.

**Source:** Mid-cycle version bump on a shipped Expo / EAS project, 2026-04.

### OP-22: Screenshot upload is a 3-step protocol, not a single POST

**Symptom:** You expect `POST /v1/appScreenshots` with a multipart body containing the image. Apple returns a JSON response with no image stored. Subsequent `GET` shows the screenshot as "not uploaded".

**Root cause:** ASC uses an out-of-band upload pattern with three steps: (1) **Reserve** — `POST /v1/appScreenshots` returns an `uploadOperations` array of presigned URLs; (2) **Upload chunks** — raw HTTP `PUT` to the presigned URLs, *without* the JWT Authorization header; (3) **Commit** — `PATCH /v1/appScreenshots/{id}` with `{uploaded: true, sourceFileChecksum: <md5>}`. Without all three, the screenshot is "reserved but not finalized" and won't show up.

**Fix:** Implement all three steps. Strip `Authorization` when PUTing to the presigned URL. `sourceFileChecksum` is the MD5 hex of the *full* file bytes.

**Prevention:** Wrap the 3-step in a helper. Document as the canonical screenshot upload pattern alongside the `deliver` and web-UI alternatives.

**Source:** ASC REST API integration on the same Flutter project, 2026-04.

For the rest, see the [full catalog](./skills/ios-app-store-submission/references/OPERATOR-PATTERNS.md).

## Skills in this repo

| Skill | Description |
|---|---|
| [`ios-app-store-submission`](./skills/ios-app-store-submission/) | Build and submit iOS apps to App Store Connect via fastlane, EAS, or bare-bones `xcrun altool`. Covers native (Swift/SwiftUI/UIKit), React Native, Expo, Flutter, and Capacitor. ~5,300 lines, 47 session-mined operator patterns from real first submissions. |

## Install

For each skill you want, symlink it into your local Claude Code skills directory:

```bash
git clone https://github.com/osekkat/agent-skills.git ~/agent-skills
cd ~/agent-skills

mkdir -p ~/.claude/skills
ln -s "$(pwd)/skills/ios-app-store-submission" ~/.claude/skills/ios-app-store-submission
```

Then start a new Claude Code session — skills load at session start and become available via `/ios-app-store-submission` or implicitly through prompts that match the skill's `description`.

To uninstall a skill: `rm ~/.claude/skills/<skill-name>` (removes the symlink only; the source in this repo is untouched).

## Skill format

Each skill follows the [Claude Code skill spec](https://code.claude.com/docs/en/skills.md):

- `SKILL.md` — YAML frontmatter (`name` + `description`) plus markdown body. The `description` is what triggers Claude Code to load the skill, so it's written to match likely user prompts.
- `references/` — supporting docs the skill points to as needed; loaded on demand, not up front.

## Operator-patterns (OP) format

Several skills include a `references/OPERATOR-PATTERNS.md` catalog of session-mined gotchas in this format:

```
## OP-N: Short title

**Symptom:** what you observed (error message, build state, weird UI)
**Root cause:** what was actually wrong
**Fix:** the minimum change that resolved it
**Prevention:** what to add to the workflow to catch this earlier
**Source:** date, project (anonymized)
```

These are real scars from real work, anonymized. Numbers never get re-used; new entries go at the bottom. When a pattern is permanently solved by tooling, the entry stays but gets a `**Status:** resolved by X` line so the history is preserved.

## Contributing

PRs welcome — especially new OPs from real submissions. Add at the bottom of the relevant `OPERATOR-PATTERNS.md`; don't renumber existing OPs. Anonymize project names, team IDs, and any other identifying info before submitting.

For new skills, follow the existing structure. Each skill should be self-contained under `skills/<skill-name>/`.

## License

[MIT](./LICENSE) — do whatever you want, just keep the copyright notice.
