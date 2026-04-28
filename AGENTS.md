# AGENTS.md — swiftui-compose

This file is for AI coding agents reading or contributing to this repository.

---

## What This Repository Is

A structured AI coding skill for developers embedding
[Compose Multiplatform](https://www.jetbrains.com/compose-multiplatform/) views in SwiftUI
apps, and embedding SwiftUI/UIKit components inside Compose Multiplatform.

**The core rule**: Create the Compose `UIViewController` **once**, then push state updates
through methods — never recreate it (no `.id(someState)` trick).

---

## Repository Structure

```
SKILL.md                         ← ALWAYS load first — anti-patterns, cheatsheet, review checklist
references/
├── compose-in-swiftui.md        ← ComposeUIViewController, UIViewControllerRepresentable, coordinator
├── swiftui-in-compose.md        ← UIKitView, UIKitViewController, UIHostingController
└── state-sharing.md             ← 3 state patterns, AsyncStream bridge, dismantleUIViewController
```

---

## How to Apply This Skill

1. Load `SKILL.md` — anti-patterns and review checklist
2. Check the **Reference Router** table in `SKILL.md` to identify which reference is needed
3. Load that one reference file only

### Reference selection guide

| Task | Load |
|------|------|
| Embedding Compose in SwiftUI | `references/compose-in-swiftui.md` |
| Embedding SwiftUI/UIKit in Compose | `references/swiftui-in-compose.md` |
| Bidirectional state / coordinator / teardown | `references/state-sharing.md` |

---

## Key Invariants

When modifying any file in this repository:

1. **No project-specific names** — use `<ModuleName>Kt`, generic feature names
2. **Both ✅ and ❌ examples required** — never show only the correct pattern
3. **Official doc links must be live** — verify Apple docs URLs
4. **JetBrains doc links must be live** — verify `kotlinlang.org/docs/multiplatform/` URLs (the SwiftUI integration URL has changed before; always confirm with `curl -sI`)
5. **The Info.plist requirement must stay** — `CADisableMinimumFrameDurationOnPhone = YES` is always required; never remove this note
6. **Apple doc quotes must be verbatim** — don't paraphrase Apple's official documentation

---

## Companion Skill

This skill is paired with [**swift-kmp**](https://github.com/sorunokoe/swift-kmp-skill),
which covers the bridge layer between KMP and Swift feature modules.

When a task involves *how Kotlin data flows into* the Compose view (interactors, type mapping,
flow bridging), refer the user to that skill. Use this skill for the SwiftUI/UIKit lifecycle
and state-sharing side of the equation.

---

## Out of Scope

- How KMP data flows from Kotlin → Swift bridge (see `swift-kmp`)
- General SwiftUI architecture (MVVM, TCA) beyond Compose embedding
- Android-specific Compose patterns
- KMP Gradle build configuration
