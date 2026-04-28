```
  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
  ┃                                                     ┃
  ┃   ◆  swiftui · compose             AI Coding Skill  ┃
  ┃   ─────────────────────────────────────────────     ┃
  ┃                                                     ┃
  ┃   Compose  ◄────────────────────────────► SwiftUI   ┃
  ┃                                                     ┃
  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

<div align="center">

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Compose Multiplatform](https://img.shields.io/badge/CMP-1.8+-4CAF50)](https://www.jetbrains.com/compose-multiplatform/)
[![Swift](https://img.shields.io/badge/Swift-6.x-FA7343?logo=swift&logoColor=white)](https://swift.org)
[![iOS stable](https://img.shields.io/badge/iOS-stable%20since%201.8-blue)](https://www.jetbrains.com/compose-multiplatform/)
[![Works with Claude](https://img.shields.io/badge/Works%20with-Claude-9370DB)](https://claude.ai)
[![Works with GitHub Copilot](https://img.shields.io/badge/Works%20with-GitHub%20Copilot-2ea44f?logo=github)](https://github.com/features/copilot)

**An AI coding skill for bidirectional interop between  
Compose Multiplatform and SwiftUI.**

[What it covers](#what-this-skill-covers) · [Files](#files) · [Core principle](#core-principle) · [Quick start](#quick-start)

</div>

---

## What This Skill Covers

- ✅ **Compose-in-SwiftUI** — `ComposeUIViewController` + `UIViewControllerRepresentable` wiring
- ✅ **Stateless embedding** — full-screen Compose apps with SwiftUI shell (navigation, tab bar)
- ✅ **Bidirectional state** — `makeCoordinator()` + `update` closure pattern, Kotlin `mutableStateOf` bridge
- ✅ **SwiftUI-in-Compose** — `UIKitView`, `UIKitViewController`, `UIHostingController` for the reverse direction
- ✅ **UIKit embedding** — maps, text fields, camera, pickers via `UIKitView`/`UIKitViewController`
- ✅ **Teardown** — `dismantleUIViewController` for cleanup, `AsyncStream.onTermination` for flow safety
- ✅ **Touch interactivity** — `interactive` / `interactionMode` for gesture routing
- ✅ **Apple doc citations** — `makeCoordinator()`, `updateUIViewController`, `dismantleUIViewController`, `UIHostingController` official quotes
- ✅ **`Info.plist` requirement** — the `CADisableMinimumFrameDurationOnPhone` entry that prevents crashes

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

| File | Load when |
|------|-----------|
| [`SKILL.md`](SKILL.md) | **Always first** — anti-patterns, quick reference, review checklist |
| [`references/compose-in-swiftui.md`](references/compose-in-swiftui.md) | Embedding Compose in SwiftUI; coordinator wiring; `ComposeUIViewController` Kotlin setup |
| [`references/swiftui-in-compose.md`](references/swiftui-in-compose.md) | Embedding SwiftUI/UIKit inside Compose; `UIKitViewController`; `UIKitView` |
| [`references/state-sharing.md`](references/state-sharing.md) | Bidirectional state; all 3 patterns; `StateFlow` → `AsyncStream`; `dismantleUIViewController` |

---

## Quick Start

### GitHub Copilot

```bash
# Copy to your repo:
cp -r swiftui-compose /path/to/your-project/.github/skills/
```

Then in Copilot Chat:
```
@swiftui-compose Help me embed a Compose map screen in my SwiftUI app with filter state
```

### Claude / Any AI agent

```
Load swiftui-compose/SKILL.md, then swiftui-compose/references/state-sharing.md.

I need to embed a Kotlin Compose map screen in SwiftUI. The SwiftUI side has
@State var filters: [MapFilter] that need to flow into Compose when they change.
Compose calls back via onMarkerClick. Implement the full coordinator pattern.
```

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
| Compose Multiplatform ↔ SwiftUI | https://kotlinlang.org/docs/multiplatform/swiftui-compose-integration.html |
| Compose Multiplatform ↔ UIKit | https://kotlinlang.org/docs/multiplatform/compose-uikit-integration.html |
| Apple `UIViewControllerRepresentable` | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable |
| Apple `makeCoordinator()` | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8 |
| Apple `dismantleUIViewController` | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/dismantleuiviewcontroller(_:coordinator:) |
| Apple `UIHostingController` | https://developer.apple.com/documentation/swiftui/uihostingcontroller |
| Compose Multiplatform interop examples | https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop |

---

## Requirements

| | Version |
|---|---|
| Compose Multiplatform | 1.6+ (1.8+ for **stable** iOS support) |
| Kotlin | 1.9+ |
| Swift | 5.9+ |
| iOS | 15.0+ |

> Compose Multiplatform iOS is **stable** as of version 1.8.0 (May 2025).

---

---

## Related Skills

> 🔗 **Bridging Kotlin data and logic into your Swift feature modules?**  
> Check out [**swift-kmp**](https://github.com/sorunokoe/swift-kmp-skill) — the companion skill covering the bridge layer architecture, interactors, `SkieSwiftFlow` → `AsyncStream` wrapping, type mapping, and `KotlinThrowable` containment.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to improve patterns or add new reference files.

## License

[MIT](LICENSE)
