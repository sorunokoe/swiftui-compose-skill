# SwiftUI and UIKit Inside Compose Multiplatform

> **Official reference**: https://kotlinlang.org/docs/multiplatform/compose-swiftui-integration.html  
> **UIKit interop reference**: https://kotlinlang.org/docs/multiplatform/compose-uikit-integration.html  
> **Official examples**: https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop  
> **Apple UIHostingController**: https://developer.apple.com/documentation/swiftui/uihostingcontroller  
> **Apple sample — Using SwiftUI with UIKit**: https://developer.apple.com/documentation/UIKit/using-swiftui-with-uikit (WWDC22)  

---

## Overview

Compose Multiplatform provides two APIs for embedding native iOS views inside Compose:
- **`UIKitView`** — wraps a `UIView` subclass
- **`UIKitViewController`** — wraps a `UIViewController`

Both use factory/update/properties closures to manage the native view lifecycle.

> ⚠️ You cannot write SwiftUI views directly in Kotlin. You must wrap them in a
> `UIViewController` in Swift, then pass that controller to `UIKitViewController`.

### `UIHostingController`

`UIHostingController` is Apple's bridge going **in the opposite direction** — it wraps SwiftUI
views in a `UIViewController` so they can be used in UIKit or passed as `UIViewController`
factories into Kotlin/Compose.

> *"Create a `UIHostingController` object when you want to integrate SwiftUI views into a UIKit
> view hierarchy. At creation time, specify the SwiftUI view you want to use as the root view
> for this view controller; you can change that view later using the `rootView` property."*
> — [UIHostingController](https://developer.apple.com/documentation/swiftui/uihostingcontroller) Apple Developer Documentation

```swift
// @MainActor @preconcurrency class UIHostingController<Content> where Content: View
let hostingController = UIHostingController(rootView: MySwiftUIView())
// Update the root view later:
hostingController.rootView = MySwiftUIView(newState: updatedState)
// Intrinsic sizing (from Apple WWDC22 sample):
hostingController.sizingOptions = .intrinsicContentSize
```

---

## Embedding SwiftUI in Compose

### Step 1: Create a `UIViewController` in Swift that hosts SwiftUI

```swift
// Swift — bridge file in the iOS app target:
import SwiftUI
import UIKit

final class SwiftUIMapViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let hostingController = UIHostingController(rootView: NativeMapView())
        addChild(hostingController)
        view.addSubview(hostingController.view)
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
        hostingController.didMove(toParent: self)
    }
}
```

### Step 2: Pass a factory closure to Kotlin

```kotlin
// In your shared entry point or iosMain helper:
@OptIn(ExperimentalForeignApi::class)
fun rootViewController(
    createNativeMap: () -> UIViewController
): UIViewController = ComposeUIViewController {
    Column(modifier = Modifier.fillMaxSize()) {
        Text("Content above the map")
        UIKitViewController(
            factory = createNativeMap,
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f)
        )
        Text("Content below the map")
    }
}
```

```swift
// Swift entry point — pass the factory:
let vc = RootKt.rootViewController(
    createNativeMap: { SwiftUIMapViewController() }
)
```

---

## Embedding UIKit Views in Compose

For UIKit views (not view controllers), use `UIKitView`:

```kotlin
@OptIn(ExperimentalForeignApi::class)
@Composable
fun NativeTextField(text: String, onTextChange: (String) -> Unit) {
    UIKitView(
        factory = {
            UITextField().apply {
                placeholder = "Enter text"
            }
        },
        update = { textField ->
            textField.text = text
        },
        modifier = Modifier.fillMaxWidth().height(44.dp)
    )
}
```

### `UIKitView` Parameters (v1.10.3)

| Parameter | Purpose |
|-----------|---------|
| `factory` | Called once (or on reset) to create the UIView |
| `update` | Called initially and on every state change — push state into the view here |
| `modifier` | Compose modifier for sizing/placement |
| `onRelease` | Called when the view exits composition permanently — free resources |
| `onReset` | If set, called on node reuse instead of recreating the view (`null` = always recreate) |
| `properties` | `UIKitInteropProperties` — controls touch routing and native accessibility |

---

## Embedding UIKit View Controllers in Compose

For view controllers (camera, maps, pickers, etc.), use `UIKitViewController`:

```kotlin
@OptIn(ExperimentalForeignApi::class)
@Composable
fun NativeMapView(
    annotations: List<MapAnnotation>,
    onAnnotationTap: (MapAnnotation) -> Unit
) {
    UIKitViewController(
        factory = {
            NativeMapViewController(onAnnotationTap = onAnnotationTap)
        },
        update = { controller ->
            controller.setAnnotations(annotations)
        },
        modifier = Modifier.fillMaxSize()
    )
}
```

---

## Touch Interactivity

By default, Compose uses a **cooperative** touch model — it intercepts gestures for 150 ms before
passing them to the native view. Use `UIKitInteropProperties` to change this:

```kotlin
// Default: cooperative touch (Compose can intercept gestures):
UIKitView(
    factory = { mapView },
    update = { … }
)

// Non-cooperative: all touches go directly to the native UIView (maps, text fields):
@OptIn(ExperimentalComposeUiApi::class)
UIKitView(
    factory = { mapView },
    update = { … },
    properties = UIKitInteropProperties(
        interactionMode = UIKitInteropInteractionMode.NonCooperative
    )
)

// No interaction: native view is non-interactive (decorative overlays):
UIKitView(
    factory = { overlayView },
    update = { … },
    properties = UIKitInteropProperties(isInteractive = false)
)
```

For view controllers, same `properties` parameter applies:
```kotlin
@OptIn(ExperimentalComposeUiApi::class)
UIKitViewController(
    factory = { cameraVC },
    update = { … },
    modifier = Modifier.fillMaxSize(),
    properties = UIKitInteropProperties(
        interactionMode = UIKitInteropInteractionMode.NonCooperative
    )
)
```

---

## Lifecycle Notes

| Callback | Called | Purpose |
|----------|--------|---------|
| `factory` | Once (or on `onReset` reuse) | Create the native view/controller |
| `update` | Initially + on every Compose state change | Sync Compose state → native |
| `onReset` | On node reuse (if non-null) | Reset view to blank state for reuse |
| `onRelease` | When the composable leaves composition permanently | Release resources |

The `factory` closure captures its environment at creation. For callbacks (Compose → Swift),
capture them in the factory closure and store on the controller — they don't change.

---

## Common Use Cases

| Native Component | API | Notes |
|------------------|-----|-------|
| `MKMapView` | `UIKitView` | Use `UIKitInteropProperties(interactionMode = NonCooperative)` for native gestures |
| `UITextField` / `UITextView` | `UIKitView` | Use `update` to sync text |
| `AVPlayerViewController` | `UIKitViewController` | Provide player via factory |
| `WKWebView` | `UIKitView` | Use `UIKitInteropProperties(interactionMode = NonCooperative)` |
| Custom SwiftUI screen | `UIKitViewController` | Wrap in `UIHostingController` first |
| `PHPickerViewController` | `UIKitViewController` | Manage delegate in Swift |
