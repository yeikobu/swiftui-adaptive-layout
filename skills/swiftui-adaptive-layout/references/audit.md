# Auditing a project for layout assumptions

Run this before designing anything. It turns "look for device assumptions" into a repeatable search, and every hit is a candidate finding, not an automatic defect: read the surrounding code before judging it.

## Contents

- [Search patterns](#search-patterns)
- [What each hit usually means](#what-each-hit-usually-means)
- [Fixes, before and after](#fixes-before-and-after)
- [What a search can't find](#what-a-search-cant-find)

## Search patterns

Run from the project root. Use `grep -rnE` with `--include='*.swift'` if `rg` isn't installed.

```bash
# Sizing from the screen instead of the container
rg -n --type swift 'UIScreen\.main|UIScreen\.screens|\.nativeBounds|UIWindow\(frame:'

# Layout chosen by device or orientation
rg -n --type swift 'userInterfaceIdiom|UIDevice\.current\.(orientation|model|name)|statusBarOrientation|interfaceOrientation|isLandscape|isPortrait'

# Hardcoded device dimensions (844, 932, 1024...) and fixed frames
rg -n --type swift '(width|height)\s*(==|>=|<=|>|<)\s*[0-9]{3,4}\b'
rg -n --type swift '\.frame\((width|height|minWidth|maxWidth|minHeight|maxHeight):\s*[0-9]{3,}'

# Size class used as a device check
rg -n --type swift 'SizeClass\s*==\s*\.(regular|compact)'

# Geometry read once and stored
rg -n --type swift 'GeometryReader|onGeometryChange|\.onAppear'

# Navigation and bars that may not adapt
rg -n --type swift 'NavigationView|NavigationSplitView|NavigationStack|TabView|UIToolbar|UINavigationBar|UITabBar|\.safeAreaInset|ignoresSafeArea'

# Full-bleed media that can lose content when the aspect ratio changes
rg -n --type swift 'scaleAspectFill|contentMode:\s*\.fill|scaledToFill'

# UIKit bridges, where SwiftUI's adaptivity stops
rg -n --type swift 'UIViewRepresentable|UIViewControllerRepresentable|UIHostingController'
```

Also check `Info.plist` and the target's build settings for `UIRequiresFullScreen`, a restricted `UISupportedInterfaceOrientations` list, and the deployment target. A locked orientation or a full-screen requirement constrains what the layout work can achieve, so surface it instead of working around it silently.

## What each hit usually means

| Pattern | Why it breaks | Judge it by |
| :- | :- | :- |
| `UIScreen.main.bounds` | The app's window isn't the screen on iPad, in iPhone Mirroring, or on iPhone Duo, and it changes mid-session | Any use for layout is a defect. Use for a one-off like a screenshot scale is fine but should still come from the window scene |
| `userInterfaceIdiom == .pad` | An iPhone can now have regular width, and an iPad window can be compact | A defect when it picks a layout. Fine when it gates a truly device-specific capability |
| `UIDevice.current.orientation` | Physical orientation doesn't describe the shape of the space the app has | A defect when it drives layout |
| `horizontalSizeClass == .regular` | Fine as a layout signal, wrong as a synonym for "iPad" | Read what the branch does. If the regular branch assumes pointer, keyboard, or iPad-only features, it's a defect |
| `width == 844`, lists of known sizes | No list of sizes includes a resizable window | Always a defect |
| `.frame(width: 390)` | Fixed sizes clip or leave dead space as the container changes | A defect for containers and content. Fine for icons, avatars, and other intrinsically fixed elements |
| Geometry stored in `onAppear` or in `@State` once | The size goes stale the first time the window resizes or the device folds | A defect unless it's refreshed on change |
| `NavigationView` | Deprecated, and its column behavior isn't controllable | Migrate when the task touches that flow; otherwise note it |
| Custom bars built from `UIToolbar`, stacks pinned to the top or bottom | The system can't move them to a vertical bar or collapse them into overflow | A defect on iPhone Duo. See [iphone-duo.md](iphone-duo.md#vertical-bars) |
| `.ignoresSafeArea()` on content | Controls can land under the camera, the fold, or a vertical bar | Fine on backgrounds, a defect on foreground content |
| `.scaledToFill()` on hero media | The crop changes with aspect ratio and can cut the subject | Check at the widest and tallest supported shapes |

## Fixes, before and after

### Size from the container, and keep it current

```swift
// Before: stale as soon as the window resizes.
let columns = Int(UIScreen.main.bounds.width / 180)

// After: measured from the view, updated on every change.
@State private var columns = 2

var body: some View {
    grid
        .onGeometryChange(for: Int.self) { proxy in
            max(1, Int(proxy.size.width / 180))
        } action: { columns = $0 }
}
```

Reduce the geometry to the value the layout needs (a column count, a Boolean) inside the transform closure. Storing the raw size invalidates the view on every pixel of a resize.

Often no measurement is needed at all:

```swift
// Adapts to any width without reading it.
LazyVGrid(columns: [GridItem(.adaptive(minimum: 160), spacing: 16)]) { ... }
```

### Choose a layout by fit, not by device

```swift
// Before
if UIDevice.current.userInterfaceIdiom == .pad {
    HStack { artwork; details }
} else {
    VStack { artwork; details }
}

// After: the first arrangement that fits wins.
ViewThatFits(in: .horizontal) {
    HStack(alignment: .top, spacing: 24) { artwork; details }
    VStack(spacing: 16) { artwork; details }
}
```

`ViewThatFits` picks by the children's ideal sizes, so it suits content with a natural minimum width. When the two branches have different structure or state, switch an `AnyLayout` instead so the subviews keep their identity and don't reset:

```swift
@Environment(\.horizontalSizeClass) private var widthClass

var body: some View {
    let layout = widthClass == .regular
        ? AnyLayout(HStackLayout(alignment: .top, spacing: 24))
        : AnyLayout(VStackLayout(spacing: 16))
    layout { artwork; details }
}
```

### Shape, not orientation

```swift
// Before
let isLandscape = UIDevice.current.orientation.isLandscape

// After: compare the space the view actually has.
.onGeometryChange(for: Bool.self) { $0.size.width > $0.size.height } action: { isWide = $0 }
```

### Keep readable widths on large windows

```swift
// Before: lines of text 1,200 points wide on a large iPad.
ScrollView { article.padding() }

// After
ScrollView {
    article
        .frame(maxWidth: 700)
        .frame(maxWidth: .infinity)
        .padding()
}
```

### Scale and screen information

```swift
// Before
let scale = UIScreen.main.scale

// After
@Environment(\.displayScale) private var displayScale
```

In UIKit code, get the screen from `view.window?.windowScene?.screen` at the moment of use and don't store it; create windows with `UIWindow(windowScene:)`.

### Backgrounds extend, controls don't

```swift
// Before: the button can sit under a camera or a vertical bar.
ZStack { hero; playButton }.ignoresSafeArea()

// After
ZStack { hero.ignoresSafeArea(); playButton }
```

## What a search can't find

Open these flows and resize them, because no pattern reveals them:

- State that resets when a split view collapses or a `ViewThatFits` branch changes (scroll position, text being typed, selection).
- Sheets, popovers, and alerts presented from a view that moves or disappears during a resize.
- Keyboard occlusion in a short, wide window.
- Text at accessibility sizes in the narrowest supported width.
- Right-to-left layouts with hand-positioned content.
- Custom gestures near screen edges that conflict with system gestures in a resizable window.
