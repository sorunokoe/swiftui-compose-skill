---
name: swiftui-compose
description: 'Compose Multiplatform ↔ SwiftUI integration patterns. USE FOR: embedding Compose Multiplatform views in a SwiftUI app; embedding SwiftUI/UIKit components inside Compose Multiplatform; sharing state between Compose and SwiftUI; managing the Compose UIViewController lifecycle; bidirectional data flow with the coordinator pattern.'
argument-hint: 'Describe the Compose/SwiftUI integration scenario'
applyTo: '**/*.{swift,kt}'
---

# Compose Multiplatform ↔ SwiftUI

Compose Multiplatform produces a `UIViewController` (via `ComposeUIViewController`). SwiftUI
consumes it through `UIViewControllerRepresentable`. This skill covers both directions.

> Compose Multiplatform iOS support is **stable** as of version 1.8.0 (May 2025).

---

## Official Documentation

| Topic | Link |
|-------|------|
| Compose Multiplatform ↔ SwiftUI | https://kotlinlang.org/docs/multiplatform/compose-swiftui-integration.html |
| Compose Multiplatform ↔ UIKit | https://kotlinlang.org/docs/multiplatform/compose-uikit-integration.html |
| UIViewControllerRepresentable | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable |
| makeCoordinator() | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8 |
| Compose Multiplatform examples (interop) | https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop |

---

## Quick Pattern Reference

| Need | Pattern |
|------|---------|
| Embed Compose in SwiftUI (stateless) | `ComposeUIViewController` + minimal `UIViewControllerRepresentable` |
| Embed Compose in SwiftUI (bidirectional state) | `makeCoordinator()` — coordinator holds the UIViewController |
| Embed SwiftUI inside Compose | `UIKitViewController` factory with a wrapping `UIViewController` |
| Embed UIKit inside Compose | `UIKitView` / `UIKitViewController` |
| Pass SwiftUI state → Compose | `updateUIViewController` calls `context.coordinator.update*(…)` |
| Pass Compose events → SwiftUI | Callback closures provided at `ComposeUIViewController` creation |

---

## Anti-Patterns

- ❌ **`.id(someState)` on a Compose view** — tears down and recreates the `UIViewController` on every change; causes visual glitches and wasted memory
- ❌ **Calling `buildController:` closure inside `updateUIViewController`** — creates a new Compose lifecycle on every SwiftUI update
- ❌ **Storing a `@State` copy of the Kotlin holder** — use `makeCoordinator()` for a stable single reference
- ❌ **Passing KMP types into feature modules** — wrap in Swift types before the feature boundary (see `swift-kmp` skill)

---

## Reference Router

| Reference | Load when |
|-----------|-----------|
| `references/compose-in-swiftui.md` | Embedding Compose in SwiftUI; `UIViewControllerRepresentable` wiring; `ComposeUIViewController` Kotlin setup |
| `references/swiftui-in-compose.md` | Embedding SwiftUI/UIKit inside Compose; `UIKitViewController`; `UIKitView` |
| `references/state-sharing.md` | Bidirectional state: `makeCoordinator()` pattern; Kotlin `mutableStateOf` bridge; `StateFlow` → `AsyncStream` |

---

## Review Checklist

- [ ] Stateless Compose view uses simple `UIViewControllerRepresentable` with empty `updateUIViewController`
- [ ] Stateful/bidirectional view uses `makeCoordinator()` — Kotlin controller created once per lifetime
- [ ] `.id()` trick is not used to force Compose state updates
- [ ] SwiftUI state flows into Compose via `context.coordinator.update*(…)` in `updateUIViewController`
- [ ] Compose-to-SwiftUI callbacks are provided at build time in the `makeCoordinator()` / `build` closure
- [ ] `Info.plist` contains `CADisableMinimumFrameDurationOnPhone = YES` (required for Compose on iOS)
- [ ] UIKit views embedded in Compose use `UIKitView` (view) or `UIKitViewController` (controller)
- [ ] `interactionMode` is set correctly for Compose views that need native touch passthrough

---

## `Info.plist` Requirement

Compose Multiplatform on iOS requires one `Info.plist` entry or the app crashes at runtime:

```xml
<key>CADisableMinimumFrameDurationOnPhone</key>
<true/>
```

---

## See Also

- For KMP-specific bridge architecture (interactors, type mapping, flow bridging) → `swift-kmp` skill
- For Kotlin `ComposeUIViewController` setup in KMP shared code → `references/compose-in-swiftui.md`
