# Handling App Review Rejections

Rejection-response sub-loop. Do not resubmit without working through all four steps. The fastest path to a second rejection is responding to a rejection without addressing the *specific* clause cited.

## Step 1 — Read the cited guideline verbatim

> TODO:
> - Locate rejection text in ASC → App Review → Resolution Center
> - Apple usually attaches screenshots/videos showing the failure — view them all
> - Open the cited guideline at developer.apple.com/app-store/review/guidelines/ and read it
> - Note: the guideline is the law; the rejection text is one reviewer's interpretation. Both matter.

## Step 2 — Reproduce locally

> TODO:
> - Build the *same version* the reviewer saw (`git checkout <tag>`) — never debug against trunk
> - Try the cited flow with a fresh install on the same device class Apple used (look at attached screenshots)
> - Video the bug for your own records — useful for the response if you disagree
> - If you can't reproduce: ask Apple for clarification (Step 4 path A) — don't guess

## Step 3 — Fix root cause + scan for similar

> TODO:
> - Don't just fix the cited instance — scan the rest of the app for the same pattern
> - Example: if reviewer hit "missing privacy disclosure for camera", check every camera/mic/location/photo usage and verify each has its disclosure
> - This is the `multi-pass-bug-hunting` discipline applied to App Review
> - Document the scan in your commit message; helps if rejected again on a similar clause

## Step 4 — Respond or resubmit

> TODO:
> - **Reply (no resubmit) when:** clarification needed, you believe the rejection is incorrect, you need an extension on a deadline. Use Resolution Center reply.
> - **Resubmit (with reply) when:** you fixed the issue. Reply should: (a) confirm the specific clause, (b) describe the specific fix, (c) point at the version/build the fix lands in.
> - Tone: factual, brief, no defensiveness, no theory of why the reviewer was wrong unless you're appealing.
> - Don't fight in Resolution Center — escalate to Apple appeal process (`developer.apple.com/contact/app-store/`) if you genuinely disagree.

## Common guideline routing

### 2.1 — App completeness

> TODO: missing functionality, broken sign-up, demo account broken, placeholder content. Common fix patterns.

### 2.5.1 — Software requirements

> TODO: private API usage, undocumented features, deprecated frameworks. Often surfaces as ITMS-90338 too.

### 4.0 — Design

The most subjective guideline category. Reviewers cite 4.0 for:

**Placeholder content** (most common silent trap)
- Default app icon from framework scaffold (Flutter chevron, RN logo, Capacitor placeholder) — see [OP-1](OPERATOR-PATTERNS.md#op-1)
- Lorem ipsum / "Coming soon" / "Test" content visible in production
- Default launch screen images
- App name still set to "Untitled" or scaffold default

**Pattern:** validation passes, processing succeeds, build reaches TestFlight, but App Review rejects within hours. Cost: 1–2 day rejection round-trip per occurrence.

**Prevention:**
1. Pre-Phase-2 checklist: visually inspect 1024×1024 marketing icon, walk every screen looking for placeholder text, verify launch screen is branded.
2. After every dependency upgrade or `flutter create` migration, re-verify icons (some operations regenerate scaffold defaults).
3. Add to `fastlane preflight` lane: hash-check the app icon against known framework defaults; fail if match.

**Confusing flows / hidden features**
- Required actions invisible (no clear CTA on home screen)
- Sign-in required for features that don't need it
- Settings buried in unexpected locations
- Modal dialogs without dismissal

**Fix patterns:** add onboarding flow for first-launch, ensure primary CTA is visible above the fold, reduce sign-in walls.

Hardest of the rejection categories to address mechanically — requires reviewing the user flow and applying basic UX heuristics.

### 4.3 — Spam / template apps / duplicates

> TODO: especially harsh for white-label / reskin patterns. Common in cross-platform tooling output. Distinguishing characteristics needed.

### 5.1.1 — Data collection and storage (privacy)

> TODO: missing privacy policy, undisclosed data collection, tracking without ATT prompt, sign-in requirement for non-account features.

### 5.1.2 — Data use and sharing

> TODO: third-party data sharing without disclosure, IDFA mishandling.

### 1.1 — Objectionable content

> TODO: usually content moderation gaps, user-generated content without reporting/blocking.

### 3.1.1 — In-app purchase

> TODO: must use IAP for digital goods/services; external payment for physical goods/services is OK; the gray area in between.

### 4.7 — HTML5 / web app concerns

> TODO: relevant for Capacitor / RN apps that are essentially webviews; mitigations.

## Response templates

> TODO: examples for each of:
> - Clarification request (you don't understand the rejection)
> - Fix-in-place resubmit note (you fixed it, here's what you fixed)
> - Disagreement / appeal escalation
> - Demo account / additional info request response

## When Apple is wrong

> TODO: rare but happens. The appeal process, expected timeline, typical outcomes. Be picky about which battles to fight — appeals are reputational with the team that reviews future updates.
