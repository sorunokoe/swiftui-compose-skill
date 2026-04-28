# Contributing to swiftui-compose

Thank you for contributing! This skill is a community resource — your real-world patterns,
corrections, and additions make it better for every developer doing Compose ↔ SwiftUI interop.

---

## Ways to Contribute

- **Fix a bug** — a pattern that's incorrect or produces broken code
- **Improve a pattern** — add a missing edge case, better example, or official Apple/Kotlin doc quote
- **Add a reference file** — a new deep-dive covering a pattern not yet documented
- **Update for new CMP versions** — Compose Multiplatform iOS evolves quickly; keep patterns current

---

## Principles

1. **Grounded in official docs** — cite [kotlinlang.org](https://kotlinlang.org/docs/multiplatform/swiftui-compose-integration.html) or Apple developer docs. If undocumented, label it clearly.
2. **Battle-tested** — include only patterns that work in production apps.
3. **Explicit about tradeoffs** — show both ✅ correct pattern and ❌ anti-pattern with a *why*.
4. **AI-agent-friendly** — skills are read under token budget. Every sentence earns its place.
5. **Stable vs experimental** — note when a CMP API is `@ExperimentalForeignApi` or unstable.

---

## How to Improve a Pattern

1. Fork and clone the repo
2. Edit the relevant `.md` file in `references/`
3. Verify code examples compile (paste into Xcode with a CMP project)
4. Open a PR titled: `[topic] Brief description` (e.g., `[state-sharing] Fix AsyncStream teardown example`)
5. Include the official doc link you're citing in the PR description

---

## How to Add a New Reference File

Add a reference file when a topic needs 100–300 lines of focused deep-dive content.

**Template:**
```markdown
# <Topic Name> — swiftui-compose

> **Official reference**: <URL>

---

## Pattern: <Name>

```swift
// ✅ Correct:
<example>

// ❌ Anti-pattern:
<wrong example>
```

<explanation>
```

After adding the file, update the **Reference Router** table in `SKILL.md`.

---

## Code Example Guidelines

Use generic placeholder names — never project-specific class names:

| Concept | Use this |
|---------|----------|
| Kotlin module name | `<ModuleName>Kt` |
| Compose entry point | `MainViewController()`, `FeatureViewController()` |
| Feature-specific data | `MapFilter`, `DataModel` |
| Bundle ID | `com.example.app` |

---

## Companion Skill

This skill pairs with [**swift-kmp**](https://github.com/sorunokoe/swift-kmp-skill).
If your contribution touches how data flows from KMP into the Compose view (interactors,
type mapping, flow bridging), it likely belongs in that skill instead.
