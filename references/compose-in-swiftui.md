# Compose Multiplatform in SwiftUI

> **Official reference**: https://kotlinlang.org/docs/multiplatform/compose-swiftui-integration.html  
> **Official interop examples**: https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop/ios-compose-in-swiftui  
> **Apple UIViewControllerRepresentable**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable  
> **Apple sample — Using SwiftUI with UIKit**: https://developer.apple.com/documentation/UIKit/using-swiftui-with-uikit (WWDC22)  

---

## Overview

Compose Multiplatform on iOS renders into a `UIViewController`. You embed it in SwiftUI using
`UIViewControllerRepresentable` — exactly how you'd embed any UIKit view controller.

The key insight: **create the Kotlin `UIViewController` once, then update state through methods
rather than recreating it.**

---

## Step 1: Kotlin Side — `ComposeUIViewController`

Expose a Kotlin function in your shared `iosMain` source set that returns a `UIViewController`:

```kotlin
// shared/src/iosMain/kotlin/MainViewController.kt
import androidx.compose.ui.window.ComposeUIViewController

fun MainViewController(): UIViewController =
    ComposeUIViewController {
        // Your root Compose composable:
        App()
    }
```

`ComposeUIViewController` is a Compose Multiplatform library function that wraps Compose
content in a `UIViewController`.

> For more on accessing Kotlin functions from Swift:
> https://kotlinlang.org/docs/native-objc-interop.html#top-level-functions-and-properties
> The generated Swift name for `MainViewController()` is `MainKt.MainViewController()`.

---

## Step 2: Swift Side — Minimal `UIViewControllerRepresentable`

For a **stateless** embedding (Compose owns all state):

```swift
struct ComposeContentView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        // Call the Kotlin function (generated class name: <ModuleName>Kt):
        Main_iosKt.MainViewController()
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        // Intentionally empty — Compose manages its own state
    }
}

// Usage in SwiftUI:
struct ContentView: View {
    var body: some View {
        ComposeContentView()
            .ignoresSafeArea()
    }
}
```

**When to use this pattern:**
- Full-screen Compose app with SwiftUI shell (navigation, tab bar)
- Compose feature screen that manages all its own state
- No SwiftUI state needs to flow into Compose

---

## Step 3: Passing Initial Parameters to Compose

Pass values at construction time (called once):

```kotlin
// Kotlin:
fun ProfileViewController(userId: String): UIViewController =
    ComposeUIViewController {
        ProfileScreen(userId = userId)
    }
```

```swift
// Swift:
struct ProfileView: UIViewControllerRepresentable {
    let userId: String

    func makeUIViewController(context: Context) -> UIViewController {
        ProfileKt.ProfileViewController(userId: userId)
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

> ⚠️ Changing `userId` in SwiftUI will NOT update Compose — `makeUIViewController` is only
> called once. For dynamic state, use the bidirectional coordinator pattern (see
> `references/state-sharing.md`).

---

## Step 4: Bidirectional State — Coordinator Pattern

When SwiftUI state needs to flow into Compose **after initial creation**, use `makeCoordinator()`.
This is Apple's recommended mechanism:

> *"Implement this method if changes to your view controller might affect other parts of your app.
> In your implementation, create a custom Swift instance that can communicate with other parts of
> your interface. For example, you might provide an instance that binds its variables to SwiftUI
> properties, causing the two to remain synchronized. If your view controller doesn't interact with
> other parts of your app, providing a coordinator is unnecessary.*
>
> *SwiftUI calls this method before calling `makeUIViewController(context:)`. The system provides
> your coordinator either directly or as part of a context structure when calling the other methods
> of your representable instance."*
> — [makeCoordinator()](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8) Apple Developer Documentation

> *"When the state of your app changes, SwiftUI updates the portions of your interface affected by
> those changes. SwiftUI calls this method for any changes affecting the corresponding UIKit view
> controller. Use this method to update the configuration of your view controller to match the
> new state information provided in the `context` parameter."*
> — [updateUIViewController(_:context:)](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/updateuiviewcontroller(_:context:)) Apple Developer Documentation

```swift
// Coordinator holds the Kotlin state wrapper (created once per representable lifetime):
final class MapCoordinator {
    // Created once — holds the Kotlin UIViewController and its state bridge:
    let holder: MapViewControllerHolder
    let controller: UIViewController

    init(component: AppComponent, initialFilters: [MapFilter]) {
        // Kotlin creates both the holder and the UIViewController here:
        holder = component.sharedViewsProvider.mapViewControllerHolder(filters: initialFilters)
        controller = holder.controller
    }

    func updateFilters(_ filters: [MapFilter]) {
        holder.updateFilters(newFilters: filters)
    }
}

struct MapComposeView: UIViewControllerRepresentable {
    let component: AppComponent
    var filters: [MapFilter]  // SwiftUI @State — may change

    func makeCoordinator() -> MapCoordinator {
        // Called ONCE — Kotlin controller created here
        MapCoordinator(component: component, initialFilters: filters)
    }

    func makeUIViewController(context: Context) -> UIViewController {
        // Returns the already-created controller from coordinator
        context.coordinator.controller
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        // Called on every SwiftUI state change — push into Compose:
        context.coordinator.updateFilters(filters)
    }
}
```

See `references/state-sharing.md` for the reusable generic `ComposeViewPrepresentable` pattern.

---

## Common Sizes and Layout

```swift
// Full screen:
ComposeContentView()
    .ignoresSafeArea()

// Fixed frame:
ComposeContentView()
    .frame(width: 300, height: 200)

// Fill available space:
ComposeContentView()
    .frame(maxWidth: .infinity, maxHeight: .infinity)
```

---

## UITabBarController / UINavigationController Integration

You can mix Compose ViewControllers with native UIKit controllers:

```swift
// AppDelegate or SceneDelegate:
let composeVC = Main_iosKt.MainViewController()
let nativeVC = NativeSettingsViewController()

let tabBar = UITabBarController()
tabBar.viewControllers = [
    UINavigationController(rootViewController: composeVC),
    UINavigationController(rootViewController: nativeVC)
]
```

---

## Teardown: `dismantleUIViewController`

Use this static method for cleanup when the representable is removed from the hierarchy.

> *"Use this method to perform additional clean-up work related to your custom view controller.
> For example, you might use this method to remove observers or update other parts of your
> SwiftUI interface."*
> — [dismantleUIViewController(_:coordinator:)](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/dismantleuiviewcontroller(_:coordinator:)) Apple Developer Documentation

```swift
struct ComposeViewPrepresentable<T: ControllerPrepresentable>: UIViewControllerRepresentable {
    // … makeCoordinator, makeUIViewController, updateUIViewController …

    static func dismantleUIViewController(
        _ uiViewController: UIViewController,
        coordinator: T
    ) {
        // Cancel any long-lived subscriptions or observers stored on the coordinator:
        coordinator.cancelSubscriptions()
    }
}
```

---

## Layout Warning from Apple

> *"SwiftUI fully controls the layout of the UIKit view controller's view using the view's
> `center`, `bounds`, `frame`, and `transform` properties. Don't directly set these
> layout-related properties on the view managed by a `UIViewControllerRepresentable`
> instance from your own code because that conflicts with SwiftUI and results in
> undefined behavior."*
> — [UIViewControllerRepresentable](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable) Apple Developer Documentation

---

## Required `Info.plist` Entry

Without this entry, Compose Multiplatform crashes on iOS at runtime:

```xml
<key>CADisableMinimumFrameDurationOnPhone</key>
<true/>
```

This disables the 60fps cap on high-refresh-rate devices, required for Compose's render loop.

