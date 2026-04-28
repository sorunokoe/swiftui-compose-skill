# State Sharing — Compose Multiplatform ↔ SwiftUI

> **Official reference**: https://kotlinlang.org/docs/multiplatform/compose-swiftui-integration.html  
> **Apple makeCoordinator()**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8  
> **Apple updateUIViewController**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/updateuiviewcontroller(_:context:)  
> **Apple dismantleUIViewController**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/dismantleuiviewcontroller(_:coordinator:)  
> **Apple AsyncStream**: https://developer.apple.com/documentation/swift/asyncstream  

---

## The Core Problem

`UIViewControllerRepresentable` calls `makeUIViewController` **once**. When SwiftUI state
changes, only `updateUIViewController` is called. This means:

- You cannot pass new state by constructing a new controller — Compose lifecycle would restart.
- You must push state changes **into** the existing controller via update methods.

The solution is the **Coordinator pattern**.

## State Ownership Decision Tree

```
Does Compose need to know about SwiftUI state changes?
├── No → Pattern 1: Unidirectional (Compose owns state)
└── Yes → Is it only SwiftUI → Compose?
           ├── Yes → Pattern 2: Unidirectional Push (coordinator + updateFilters)
           └── No → Pattern 3: Bidirectional (callbacks + coordinator)
```

---

### Apple's Lifecycle Guarantee (from the docs)

- **`makeCoordinator()`** — *"SwiftUI calls this method before calling `makeUIViewController(context:)`. The system provides your coordinator either directly or as part of a context structure when calling the other methods of your representable instance."*
- **`updateUIViewController(_:context:)`** — *"When the state of your app changes, SwiftUI updates the portions of your interface affected by those changes."*
- **`dismantleUIViewController(_:coordinator:)`** — *"Use this method to perform additional clean-up work related to your custom view controller. For example, you might use this method to remove observers."*

---

## Pattern 1: Unidirectional (Compose owns state)

When Compose fully controls its own state and SwiftUI only positions it:

```swift
struct ComposeFeatureView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        FeatureKt.FeatureViewController()  // Compose ViewModel inside owns all state
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

No coordinator needed. Compose reacts to its own state changes (ViewModel / StateFlow).

---

## Pattern 2: SwiftUI → Compose (Unidirectional Push)

When SwiftUI drives some state that Compose must reflect:

### Kotlin Side

Expose a mutable state holder and an update method:

```kotlin
// shared/src/iosMain/kotlin/FilterableViewController.kt
import androidx.compose.runtime.*
import androidx.compose.ui.window.ComposeUIViewController

class FilterableViewControllerHolder(private val initialFilters: List<String>) {
    private val filters = mutableStateOf(initialFilters)

    val controller: UIViewController = ComposeUIViewController {
        FilterableContent(filters = filters.value)
    }

    fun updateFilters(newFilters: List<String>) {
        filters.value = newFilters  // Compose recomposits automatically
    }
}
```

### Swift Side — Coordinator holds the holder

```swift
final class FilterCoordinator {
    let holder: FilterableViewControllerHolder

    init(initialFilters: [String]) {
        holder = FilterableViewControllerHolder(initialFilters: initialFilters)
    }

    func update(filters: [String]) {
        holder.updateFilters(newFilters: filters)
    }
}

struct FilterableComposeView: UIViewControllerRepresentable {
    var filters: [String]  // SwiftUI @State — may change

    func makeCoordinator() -> FilterCoordinator {
        FilterCoordinator(initialFilters: filters)
        // Coordinator and Kotlin UIViewController created once here
    }

    func makeUIViewController(context: Context) -> UIViewController {
        context.coordinator.holder.controller
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        context.coordinator.update(filters: filters)
        // Called on every SwiftUI state change — pushes into Compose
    }
}
```

---

## Pattern 3: Bidirectional (both directions)

When Compose also needs to push events back to SwiftUI (e.g. user tapped a marker,
completed a form):

### Kotlin Side

Add callback parameters to the controller factory:

```kotlin
class MapViewControllerHolder(
    initialFilters: List<MapFilter>,
    private val onMarkerClick: (String) -> Unit  // Compose → Swift callback
) {
    private val filtersState = mutableStateOf(initialFilters)

    val controller: UIViewController = ComposeUIViewController {
        MapContent(
            filters = filtersState.value,
            onMarkerClick = onMarkerClick  // passed from Swift at creation
        )
    }

    fun updateFilters(newFilters: List<MapFilter>) {
        filtersState.value = newFilters
    }
}
```

### Swift Side

```swift
struct BidirectionalMapView: UIViewControllerRepresentable {
    var filters: [MapFilter]
    var onMarkerClick: (String) -> Void  // Compose → SwiftUI

    func makeCoordinator() -> MapViewControllerHolder {
        MapViewControllerHolder(
            initialFilters: filters,
            onMarkerClick: onMarkerClick  // provided once at creation
        )
    }

    func makeUIViewController(context: Context) -> UIViewController {
        context.coordinator.controller
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        context.coordinator.updateFilters(newFilters: filters)
        // Note: onMarkerClick is captured at creation and does not update here.
        // If the callback needs to change, use a stable reference (e.g. class wrapper).
    }
}
```

---

## Reusable Generic Helper

For projects with many bidirectional Compose views, a generic helper reduces boilerplate:

```swift
// Protocol: Kotlin holder must conform to this:
protocol ControllerPrepresentable {
    var controller: UIViewController { get }
}

// Generic representable:
struct ComposeViewPrepresentable<T: ControllerPrepresentable>: UIViewControllerRepresentable {
    let build: () -> T       // called once — creates Kotlin holder + UIViewController
    var update: (T) -> Void  // called on every SwiftUI state change

    func makeCoordinator() -> T { build() }

    func makeUIViewController(context: Context) -> UIViewController {
        context.coordinator.controller
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        update(context.coordinator)
    }
}

// Declare the conformance on each Kotlin holder in the bridge layer:
extension FilterableViewControllerHolder: ControllerPrepresentable {}

// Usage:
ComposeViewPrepresentable {
    FilterableViewControllerHolder(initialFilters: filters)
} update: { context in
    context.updateFilters(newFilters: filters)
}
```

---

## Compose State → Swift: `StateFlow` Pattern

For Compose state that needs to propagate to Swift (not just via callbacks):

```kotlin
// Kotlin shared code:
class SessionStateHolder {
    private val _sessionState = MutableStateFlow<SessionState>(SessionState.Loading)
    val sessionState: StateFlow<SessionState> = _sessionState.asStateFlow()

    val controller: UIViewController = ComposeUIViewController {
        SessionScreen(stateFlow = _sessionState)
    }
}
```

```swift
// Swift — observe as AsyncStream:
let holder = SessionStateHolder()

// Observe Kotlin StateFlow via SKIE:
Task {
    for await state in holder.sessionState {
        // React to Compose-driven state changes in Swift
        switch onEnum(of: state) {
        case .loading: showLoadingIndicator()
        case let .loaded(data): showContent(data)
        case .error: showError()
        }
    }
}
```

See `swift-kmp/references/flow-bridging.md` for the full `AsyncStream` wrapping pattern.

---

## Callback Stability

When passing Swift closures into Kotlin that reference `self`, use `[weak self]` to avoid
retain cycles. Store callbacks in a class wrapper if they need to be mutable:

```swift
// ❌ Potential retain cycle:
MapViewControllerHolder(
    initialFilters: filters,
    onMarkerClick: { [self] markerId in self.handleMarkerClick(markerId) }
)

// ✅ Weak self:
MapViewControllerHolder(
    initialFilters: filters,
    onMarkerClick: { [weak self] markerId in self?.handleMarkerClick(markerId) }
)
```

---

## Coordinator Cleanup with `dismantleUIViewController`

When the representable leaves the SwiftUI hierarchy, implement `dismantleUIViewController`
to clean up any resources held by the coordinator (subscriptions, observers, tasks):

```swift
struct FilterableComposeView: UIViewControllerRepresentable {
    // … makeCoordinator, makeUIViewController, updateUIViewController …

    static func dismantleUIViewController(
        _ uiViewController: UIViewController,
        coordinator: FilterCoordinator
    ) {
        // Cancel any Task or subscription the coordinator owns:
        coordinator.cancelAll()
    }
}

final class FilterCoordinator {
    let holder: FilterableViewControllerHolder
    private var observationTask: Task<Void, Never>?

    func cancelAll() {
        observationTask?.cancel()
    }
}
```

---

## `AsyncStream` for Compose → SwiftUI Events

Apple's `AsyncStream` docs describe the `onTermination` pattern used to bridge callbacks:

> *"An asynchronous sequence generated from a closure that calls a continuation to produce
> new elements. `AsyncStream` conforms to `AsyncSequence`, providing a convenient way to
> create an asynchronous sequence without manually implementing an asynchronous iterator.
> In particular, an asynchronous stream is well-suited to adapt callback- or
> delegation-based APIs to participate with `async`-`await`."*
> — [AsyncStream](https://developer.apple.com/documentation/swift/asyncstream) Apple Developer Documentation

The Apple-canonical pattern for `onTermination`:
```swift
AsyncStream { continuation in
    let monitor = EventMonitor()
    monitor.eventHandler = { event in
        continuation.yield(event)
    }
    continuation.onTermination = { @Sendable _ in
        monitor.stop()  // ← called when stream consumer cancels or finishes
    }
    monitor.start()
}
```

When adapting Kotlin flows (via SKIE), the same `onTermination` pattern ensures the driving
`Task` is cancelled when the consumer stops iterating — see `swift-kmp/references/flow-bridging.md`.

