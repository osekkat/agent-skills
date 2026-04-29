# agent-skills

A growing collection of [Claude Code](https://www.anthropic.com/claude-code) skills, distilled from real engineering work.

These are *Claude Code* skills, not [claude.ai skills](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills) — different format, different install path. Each skill is a directory with a `SKILL.md` (YAML frontmatter + markdown body) plus a `references/` folder of supporting docs the agent loads on demand.

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
