# Changelog

All notable changes to this skill are documented here.

---

## [1.0.0] — 2025-04-28

### Initial release

First public release of the `swiftui-compose` AI coding skill, covering
bidirectional interop between Compose Multiplatform and SwiftUI at the
Compose Multiplatform 1.8.0 stable baseline.

#### Skill files

- **`SKILL.md`** — quick-reference cheatsheet, anti-pattern guide, review checklist,
  and Reference Router for AI agents
- **`references/compose-in-swiftui.md`** — `ComposeUIViewController`, stateless and
  bidirectional `UIViewControllerRepresentable` wiring, coordinator pattern, layout,
  `UITabBarController` integration, `dismantleUIViewController`, Info.plist requirement
- **`references/swiftui-in-compose.md`** — `UIKitView`, `UIKitViewController`,
  `UIHostingController`, touch interactivity via `UIKitInteropProperties` /
  `UIKitInteropInteractionMode`, lifecycle callbacks
- **`references/state-sharing.md`** — all 3 state patterns (unidirectional, push,
  bidirectional), reusable generic `ComposeViewPrepresentable`, `StateFlow` → `AsyncStream`
  bridge via SKIE, `dismantleUIViewController` cleanup, callback stability / `[weak self]`,
  state ownership decision tree

#### API baseline (Compose Multiplatform 1.8.0)

- `UIKitView` and `UIKitViewController` use `properties: UIKitInteropProperties` for
  touch routing (replaces the old `interactive: Boolean` parameter from pre-1.8 builds)
- `UIKitInteropInteractionMode.NonCooperative` and `Cooperative` documented
  (`@ExperimentalComposeUiApi`)

#### Fact-checked against

- Apple Developer Documentation (raw Markdown API endpoints)
- Compose Multiplatform 1.8.0 sources JAR (`ui-1.8.0-sources.jar`)
- JetBrains KMP documentation at `kotlinlang.org/docs/multiplatform/`
