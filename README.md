<div align="center">

<pre align="center">
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃                                                     ┃
┃   ◆  swiftui · compose             AI Coding Skill  ┃
┃   ─────────────────────────────────────────────     ┃
┃                                                     ┃
┃   Compose  ◄────────────────────────────► SwiftUI   ┃
┃                                                     ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
</pre>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Compose Multiplatform](https://img.shields.io/badge/CMP-1.8+-4CAF50)](https://www.jetbrains.com/compose-multiplatform/)
[![Swift](https://img.shields.io/badge/Swift-5.9+-FA7343?logo=swift&logoColor=white)](https://swift.org)
[![iOS stable](https://img.shields.io/badge/iOS-stable%20since%201.8-blue)](https://www.jetbrains.com/compose-multiplatform/)
[![Maintained by skills-evolution](https://img.shields.io/badge/maintained%20by-skills--evolution-0ea5e9)](https://github.com/sorunokoe/skills-evolution)
<br>
[![Works with Claude](https://img.shields.io/badge/Works%20with-Claude-9370DB)](https://claude.ai)
[![Works with Codex](https://img.shields.io/badge/Works%20with-Codex-412991?logo=openai&logoColor=white)](https://openai.com/codex)
[![Works with GitHub Copilot](https://img.shields.io/badge/Works%20with-GitHub%20Copilot-2ea44f?logo=github)](https://github.com/features/copilot)
[![Works with Cursor](https://img.shields.io/badge/Works%20with-Cursor-000000)](https://cursor.com)
[![Works with Gemini](https://img.shields.io/badge/Works%20with-Gemini-4285F4?logo=google&logoColor=white)](https://gemini.google.com)
<!-- skill-evolution:badge:begin -->
[![Skill evolved 2×](https://img.shields.io/badge/evolved-2%C3%97_evolving-green?style=flat-square&logo=dna&logoColor=white)](https://github.com/sorunokoe/swiftui-compose-skill/pulls?q=is%3Apr+is%3Amerged)
<!-- skill-evolution:badge:end -->

**An AI coding skill for bidirectional interop between  
Compose Multiplatform and SwiftUI.**

[What it covers](#what-this-skill-covers) · [Files](#files) · [Core principle](#core-principle) · [Quick start](#quick-start) · [Key rules](#key-rules-summary) · [Docs](#official-documentation) · [Automated maintenance](#automated-maintenance)

</div>

---

## What This Skill Covers

| | |
|---|---|
| **Compose in SwiftUI** | `ComposeUIViewController` + `UIViewControllerRepresentable` wiring |
| **State bridging** | `makeCoordinator()`, push and bidirectional patterns, Kotlin `mutableStateOf` |
| **SwiftUI in Compose** | `UIKitView`, `UIKitViewController`, `UIHostingController` |
| **UIKit embedding** | Maps, text fields, camera, pickers via `UIKitView` |
| **Teardown** | `dismantleUIViewController`, `AsyncStream.onTermination` |
| **Touch routing** | `UIKitInteropProperties` cooperative vs non-cooperative gestures |

> 🔗 Bridging KMP data and logic into Swift? See the companion [**swift-kmp**](https://github.com/sorunokoe/swift-kmp-skill) skill.

---

## Core Principle

> **Create the Compose `UIViewController` once, then update state through methods —  
> never recreate it.**

```
❌ Anti-pattern: .id(someState) on a Compose view
   → tears down and recreates UIViewController on every state change
   → visual glitches, wasted memory, Compose lifecycle restart

✅ Correct: makeCoordinator() holds the UIViewController reference (created once)
           updateUIViewController() calls context.coordinator.update*(newState)
```

Apple's documentation confirms the intended lifecycle:
> *"SwiftUI calls `makeCoordinator()` before calling `makeUIViewController(context:)`.
> The system provides your coordinator either directly or as part of a context structure
> when calling the other methods of your representable instance."*

---

## Files

| File | Load when | Tokens |
|------|-----------|--------|
| [`SKILL.md`](SKILL.md) | **Always first** — anti-patterns, quick reference, review checklist | ~1.5k |
| [`references/compose-in-swiftui.md`](references/compose-in-swiftui.md) | Embedding Compose in SwiftUI; coordinator wiring; `ComposeUIViewController` Kotlin setup | ~2k |
| [`references/swiftui-in-compose.md`](references/swiftui-in-compose.md) | Embedding SwiftUI/UIKit inside Compose; `UIKitViewController`; `UIKitView` | ~2k |
| [`references/state-sharing.md`](references/state-sharing.md) | Bidirectional state; all 3 patterns; `StateFlow` → `AsyncStream`; `dismantleUIViewController` | ~2.5k |

> **Token budget:** `SKILL.md` (~1.5k tokens) covers anti-patterns and routing. Load one reference only when you need the detailed API for that topic.

---

## Quick Start

### 1. Install

```bash
# Clone into the canonical skill location:
git clone https://github.com/sorunokoe/swiftui-compose-skill.git \
  /path/to/your-project/.github/skills/swiftui-compose

# Optional: detach from upstream git history
rm -rf /path/to/your-project/.github/skills/swiftui-compose/.git
```

> **Why `.github/skills/swiftui-compose/`?** This is the path GitHub Copilot, Cursor, and
> [skills-evolution](https://github.com/sorunokoe/skills-evolution) discover skills from.

### 2. Use with GitHub Copilot

```
@swiftui-compose Help me embed a Compose map screen in my SwiftUI app with filter state
```

### 3. Use with Claude / any AI agent

```
Load swiftui-compose/SKILL.md, then swiftui-compose/references/state-sharing.md.

I need to embed a Kotlin Compose map screen in SwiftUI. The SwiftUI side has
@State var filters: [MapFilter] that need to flow into Compose when they change.
Compose calls back via onMarkerClick. Implement the full coordinator pattern.
```

---

---

## Key Rules (summary)

- ❌ **`.id(someState)` on a Compose view** — recreates the entire lifecycle
- ❌ **Calling the `build:` closure inside `updateUIViewController`** — creates a new Compose lifecycle on every update
- ❌ **Storing a `@State` copy of the Kotlin holder** — use `makeCoordinator()` for a stable reference
- ✅ **`CADisableMinimumFrameDurationOnPhone = YES` in `Info.plist`** — required for Compose on iOS
- ✅ **`context.coordinator.update*(…)` in `updateUIViewController`** — correct way to push SwiftUI state to Compose
- ✅ **`dismantleUIViewController` implemented** when coordinator owns subscriptions or tasks

Full rules and review checklist in [`SKILL.md`](SKILL.md).

---

## The Three State Patterns

| When | Pattern |
|------|---------|
| Compose owns all state; SwiftUI just positions | **Unidirectional** — simple `UIViewControllerRepresentable`, empty `updateUIViewController` |
| SwiftUI state needs to flow into Compose | **Push** — `makeCoordinator()` + `context.coordinator.update*()` in `updateUIViewController` |
| State flows both ways + Compose fires Swift callbacks | **Bidirectional** — coordinator holds holder, callbacks provided at build time |

Full patterns with code in [`references/state-sharing.md`](references/state-sharing.md).

---

## Official Documentation

| Topic | Link |
|-------|------|
| Compose Multiplatform ↔ SwiftUI | [kotlinlang.org →](https://kotlinlang.org/docs/multiplatform/compose-swiftui-integration.html) |
| Compose Multiplatform ↔ UIKit | [kotlinlang.org →](https://kotlinlang.org/docs/multiplatform/compose-uikit-integration.html) |
| Apple `UIViewControllerRepresentable` | [developer.apple.com →](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable) |
| Apple `makeCoordinator()` | [developer.apple.com →](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8) |
| Apple `dismantleUIViewController` | [developer.apple.com →](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/dismantleuiviewcontroller(_:coordinator:)) |
| Apple `UIHostingController` | [developer.apple.com →](https://developer.apple.com/documentation/swiftui/uihostingcontroller) |
| Compose Multiplatform interop examples | [github.com/JetBrains →](https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop) |

---

## Requirements

| | Version |
|---|---|
| Compose Multiplatform | 1.8+ (iOS **stable** since 1.8.0, May 2025) |
| Kotlin | 2.1+ |
| Swift | 5.9+ |
| iOS | 15.0+ |

> Compose Multiplatform iOS is **stable** as of version 1.8.0 (May 2025). CMP 1.8+ requires Kotlin 2.1+.

---

## Related Skills

> 🔗 **Bridging Kotlin data and logic into your Swift feature modules?**  
> Check out [**swift-kmp**](https://github.com/sorunokoe/swift-kmp-skill) — the companion skill covering the bridge layer architecture, interactors, `SkieSwiftFlow` → `AsyncStream` wrapping, type mapping, and `KotlinThrowable` containment.

---

## Automated Maintenance

This skill is governed by [**skills-evolution**](https://github.com/sorunokoe/skills-evolution) — AI skill governance that keeps guidance files accurate and up to date automatically.

### gh-aw (recommended)

```bash
# PR review — AI feedback on every PR touching SKILL.md or references/**
gh aw add sorunokoe/skills-evolution/workflows/oss-skill-pr-check.md@latest

# Monthly update — version checks, AI content patches, opens PR
gh aw add sorunokoe/skills-evolution/workflows/oss-skill-update.md@latest

gh aw compile
```

### GitHub Actions

<details>
<summary>Monthly skill health workflow</summary>

[`.github/workflows/skill-health.yml`](.github/workflows/skill-health.yml) — runs monthly and on demand:

- **Structural audit** — broken local links, missing frontmatter fields
- **AI content update** — checks `SKILL.md` + `references/*.md` against the latest `compose-multiplatform` release; proposes conservative patches via GitHub Models

```yaml
# .github/workflows/skill-health.yml
name: Skill Health
on:
  schedule:
    - cron: "0 3 1 * *"
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
  models: read
jobs:
  health:
    uses: sorunokoe/skills-evolution/.github/workflows/oss-skill-health.yml@latest
    with:
      enable_ai_skill_update: true
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```
</details>

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to improve patterns or add new reference files.

## License

[MIT](LICENSE)
