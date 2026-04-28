# SwiftUI and UIKit Inside Compose Multiplatform

> **Official reference**: https://kotlinlang.org/docs/multiplatform/swiftui-compose-integration.html  
> **UIKit interop reference**: https://kotlinlang.org/docs/multiplatform/compose-uikit-integration.html  
> **Official examples**: https://github.com/JetBrains/compose-multiplatform/tree/master/examples/interop  
> **Apple UIHostingController**: https://developer.apple.com/documentation/swiftui/uihostingcontroller  
> **Apple sample — Using SwiftUI with UIKit**: https://developer.apple.com/documentation/UIKit/using-swiftui-with-uikit (WWDC22)  

---

## Overview

Compose Multiplatform provides two APIs for embedding native iOS views inside Compose:
- **`UIKitView`** — wraps a `UIView` subclass
- **`UIKitViewController`** — wraps a `UIViewController`

Both use factory/update/interactive closures to manage the native view lifecycle.

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

### `UIKitView` Parameters

| Parameter | Purpose |
|-----------|---------|
| `factory` | Called once to create the UIView — do not store external state here |
| `update` | Called when Compose recompositions with new state — push state into the view here |
| `modifier` | Compose modifier for sizing/placement |
| `interactive` | Controls touch event routing (see below) |

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

By default, Compose handles all touch events. Use `interactionMode` to control routing:

```kotlin
// All touches go to the native UIView (e.g. interactive maps, text fields):
UIKitView(
    factory = { mapView },
    update = { … },
    interactive = true  // UIView receives touch events natively
)

// Compose handles touches (e.g. decorative overlay, custom gesture recognizer needed):
UIKitView(
    factory = { overlayView },
    update = { … },
    interactive = false
)
```

For view controllers:
```kotlin
UIKitViewController(
    factory = { cameraVC },
    update = { … },
    modifier = Modifier.fillMaxSize(),
    // interactionMode = UIKitInteropInteractionMode.NonCooperative  // Compose handles gestures
)
```

---

## Lifecycle Notes

| Callback | Called | Purpose |
|----------|--------|---------|
| `factory` | Once | Create the native view/controller |
| `update` | On every Compose recomposition with new state | Sync Compose state → native |
| Dispose | When the composable leaves composition | Native view is removed |

The `factory` closure captures its environment at creation. For callbacks (Compose → Swift),
capture them in the factory closure and store on the controller — they don't change.

---

## Common Use Cases

| Native Component | API | Notes |
|------------------|-----|-------|
| `MKMapView` | `UIKitView` | Set `interactive = true` for gestures |
| `UITextField` / `UITextView` | `UIKitView` | Use `update` to sync text |
| `AVPlayerViewController` | `UIKitViewController` | Provide player via factory |
| `WKWebView` | `UIKitView` | Set `interactive = true` |
| Custom SwiftUI screen | `UIKitViewController` | Wrap in `UIHostingController` first |
| `PHPickerViewController` | `UIKitViewController` | Manage delegate in Swift |
