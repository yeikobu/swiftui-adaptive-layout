# iPhone Duo

iPhone Duo is a foldable iPhone with a compact outer display and a large inner display. This file summarizes Apple's guidance as of November 2026. These APIs are recent and may have changed: when a signature here disagrees with the installed SDK, the SDK wins, and the links in [Sources](#sources) are the authority.

## Contents

- [What changes for an app](#what-changes-for-an-app)
- [What you get per SDK](#what-you-get-per-sdk)
- [Availability](#availability)
- [Decide how much work a view needs](#decide-how-much-work-a-view-needs)
- [Vertical bars](#vertical-bars)
- [ArrangementView](#arrangementview)
- [Reserved regions](#reserved-regions)
- [Camera and media](#camera-and-media)
- [Testing](#testing)
- [Sources](#sources)

## What changes for an app

- **The window resizes mid-session.** Opening, closing, partially folding, and rotating the device moves the app between displays and changes its size while it's running. Anything measured once is wrong afterwards.
- **Any size-class combination is possible on an iPhone.** The inner display can be regular width. On it, the app should feel like an extension of its iPad layout, not a stretched phone layout.
- **Orientation doesn't describe shape.** The device can be in one orientation while the app occupies a space of a different shape.
- **Bars can become vertical.** Navigation bars, toolbars, and tab bars are grouped along a side edge in some poses.
- **Parts of the view are reserved.** The fold divides the view when the device is partially folded, and cameras cover part of it. The outer front camera always occludes; the inner front camera occludes only while in use.

An app that already resizes well on iPad or in iPhone Mirroring is most of the way there. Fix general resizing first ([audit.md](audit.md)), then handle bars, then the fold.

## What you get per SDK

The SDK the app is built with decides how the system presents it. Building with a newer SDK doesn't require raising the deployment target.

| Built with | Inner display, open | Outer display, closed |
| :- | :- | :- |
| iOS 26 SDK or earlier | Centered window with empty space around it | Content sits beside the status bar and camera, not under them |
| iOS 27 SDK | Resizes to fill most of the display, still avoiding the status bar on the side edge | Resizes dynamically |
| iOS 27.1 SDK or later | Uses the full display; toolbars and tab bars appear vertically below the status bar | Full display, with vertical bars |

Before shipping a build made with the iOS 27.1 SDK, confirm the content adapts when bars become vertical. That is the step where an untested app visibly breaks.

## Availability

| API | Introduced |
| :- | :- |
| `ReservedRegion`, `GeometryProxy.reservedRegions(kind:options:layoutDirectionBehavior:)` | iOS 27.1 |
| `ArrangementView`, `arrangementViewStyle(_:)` | iOS 27.1 |
| `toolbarVerticalBehavior(_:)`, `\.toolbarVerticalEdge`, `axisBehavior(_:)` | iOS 27.1 |
| `visibilityPriority(_:)`, `ToolbarOverflowMenu`, `.topBarPinnedTrailing`, `presentationPlacement(_:)` | iOS 27.0 |
| `backgroundExtensionEffect()` | iOS 26.0 |

Most apps deploy to earlier versions, so every use needs an availability check and a fallback that still lays out correctly:

```swift
if #available(iOS 27.1, *) {
    ArrangementView { controls } secondary: { player }
        .arrangementViewStyle(.overlay)
} else {
    ZStack { player; controls }
}
```

For modifiers, wrap the check in a small `ViewModifier` or `ToolbarContent` extension so call sites stay readable. Don't raise the deployment target to avoid the check.

## Decide how much work a view needs

Work down this list and stop at the first match. Most views stop at the first line.

1. **The view is built from system containers** (`NavigationStack`, `NavigationSplitView`, `TabView`, sheets, alerts, context menus, toolbars). They already adapt to the fold, the cameras, and vertical bars. Do nothing Duo-specific; verify in the simulator.
2. **The view has custom bars or a crowded toolbar.** Move the items into system toolbars and organize them. See [Vertical bars](#vertical-bars).
3. **The view is two related panes**, side by side, stacked, or one layered over the other (player and controls, map and details, document and inspector). Use [`ArrangementView`](#arrangementview).
4. **The view positions content itself**: floating controls, overlays, explicit frames, a custom `Layout`, a canvas. Read [reserved regions](#reserved-regions) in the view that does the positioning.
5. **The view is a scrolling container whose rows are hard to use across the fold.** Lists, tables, and scroll views don't split their content around a division on their own. Act only when a row that straddles the fold is a real problem, then read division regions and arrange the content.

## Vertical bars

On the outer display when the device is closed, and for some leading or trailing views on the inner display when it's open, the system presents navigation bar, toolbar, and tab bar items in one vertical bar on a side edge.

**The system only does this for bars it owns.** Attach `.toolbar` to content inside a `NavigationStack` or `NavigationSplitView` and use `TabView` for tabs. Hand-built bars (an `HStack` pinned with `safeAreaInset`, a wrapped `UIToolbar`) stay horizontal and waste the space the layout was designed around.

Presentation varies by container:

| Context | Bars appear |
| :- | :- |
| Inspector | Horizontally |
| Split view showing several columns | Horizontally for sidebar and content, vertically for detail |
| Sheet on the outer display | Vertically by default |
| Sheet on the inner display | Horizontally when centered or leading, vertically when trailing |

### Give every item an icon and a title

An item can end up in a vertical bar, a horizontal bar, or the overflow menu, and each uses a different part of its label:

- Vertical bar: icon only. **An item with a title and no icon is never shown vertically, and neither is an item built from a custom view.** Such an item isn't dropped; see [what happens to it](#the-cost-of-adding-an-icon).
- Horizontal bar: icon or title, preferring the icon.
- Overflow menu: icon and title.

So use `Label` (or `Button(_:systemImage:action:)`) for every action that should move into the vertical bar:

```swift
.toolbar {
    ToolbarItem(placement: .cancellationAction) {
        Button("Close", systemImage: "xmark") { dismiss() }
    }
    ToolbarItem(placement: .topBarPinnedTrailing) {        // iOS 27.0
        Button("Done", systemImage: "checkmark") { save() }
    }
    ToolbarItem(placement: .primaryAction) {
        Button("Share", systemImage: "square.and.arrow.up") { share() }
    }
    .visibilityPriority(.high)                             // iOS 27.0: last to overflow
    ToolbarItem {
        Toggle("Grid", systemImage: "square.grid.2x2", isOn: $showsGrid)
    }
    .axisBehavior(.horizontalOnly)                         // iOS 27.1: never in a vertical bar
    ToolbarOverflowMenu {                                  // iOS 27.0: always in overflow
        Button("Duplicate", systemImage: "plus.square.on.square") { duplicate() }
        Button("Export", systemImage: "arrow.up.doc") { export() }
    }
}
```

#### The cost of adding an icon

Adding an icon to a text button changes how it looks everywhere, not only on iPhone Duo. Because a horizontal bar prefers the icon, a button that read "Cancel" becomes a bare ✕ on every iPhone and iPad once it has one. `.labelStyle(.titleAndIcon)` doesn't bring the text back: a toolbar still shows the icon alone.

And the icon often isn't needed to keep the button reachable. Observed in the iPhone Duo simulator (iOS 27.1 SDK, detail column of a split view on the inner display): **a title-only item stays visible in a horizontal bar at the top, with its text, while the items that have icons move into the vertical bar.** It doesn't disappear and it doesn't go to overflow. Other contexts, such as sheets on the outer display, weren't checked; verify the one you care about.

So a missing icon costs some consistency (one item sits apart from the vertical bar), while an added icon costs the label on every device. Which matters more depends on the app:

- **Keep text-only buttons** where the text does real work: apps for older or less technical people, unfamiliar actions, or a project whose guidelines require visible labels. Cancel, Close, and Done in a sheet are the usual cases.
- **Add icons** to actions with a well-known symbol (share, add, delete, more) and wherever a tidy vertical bar matters more than a visible word.
- When unsure, raise the trade-off with the person instead of converting every button silently.
- Check the result in the simulator in the pose that matters. This takes a minute, and the behavior is easy to get wrong from the rule alone.

Ordering and placement:

- Keep the top of the bar for navigation (Back, Close), then prominent actions (Done). A navigation container adds Back itself.
- Use semantic placements so the system can group items. `.cancellationAction` for a custom Back or Close, `.topBarPinnedTrailing` for a prominent confirming action. Pinned items leave the bar only when search is active and space runs out.
- `visibilityPriority(_:)` takes `.low`, `.automatic`, `.high`, or a custom priority made with `init(lowerThan:)` / `init(higherThan:)`. Lower priorities move to overflow first.
- `axisBehavior(_:)` takes `.automatic`, `.horizontalOnly`, or `.verticalPreferred`.

### Opt out, or react to the bar

```swift
// Use horizontal bars for this whole container.
TabView { ... }
    .toolbarVerticalBehavior(.disabled)
```

Disable vertical bars only where horizontal bars clearly serve the UI better: a full-screen video player with toolbar controls, or a fixed non-scrolling layout such as a calculator where width is scarce. Treat it as a stable property of the screen. Don't toggle it from view state, and to hide bars use `toolbarVisibility(_:for:)` instead. A `NavigationStack` resolves the behavior from its top view, a `TabView` from the selected tab, and a `NavigationSplitView` from its trailing-most column.

To place custom UI relative to the bar, read where the system puts it:

```swift
@Environment(\.toolbarVerticalEdge) private var barEdge    // HorizontalEdge?

FloatingPalette()
    .frame(maxWidth: .infinity, alignment: barEdge == .trailing ? .leading : .trailing)
```

The value is the system's preferred edge for the context whether or not a vertical bar is currently showing, and `nil` where a vertical bar is never used.

Related tools:

- `presentationPlacement(.leading)` or `.trailing` on a sheet's content chooses which side of the inner display it sits on, which also decides whether its bars are vertical. Only sheets respect it.
- `backgroundExtensionEffect()` extends a hero or background image under a vertical bar so the bar doesn't sit on a blank strip.

## ArrangementView

A container for one primary and one secondary view that rearranges them by size class, orientation, and the fold. It replaces hand-written switching between `HStack`, `VStack`, and `ZStack`, and it keeps the two views on separate sides of the fold when the device is partially folded.

### Split: two views that always have their own space

```swift
ArrangementView {
    NowPlayingView()      // primary
} secondary: {
    LyricsView()
}
.arrangementViewStyle(.split)
```

| Situation | Primary | Secondary |
| :- | :- | :- |
| Taller than wide | Top | Bottom |
| Wider than tall | Leading | Trailing |
| Closed, landscape | Shown | Hidden |

Use it where you would otherwise switch between an `HStack` and a `VStack`.

### Overlay: foreground over background, separated when folded

```swift
ArrangementView {
    PlayerControls()      // primary: the foreground
} secondary: {
    VideoPlayer(player: player)
}
.arrangementViewStyle(.overlay)
```

| Situation | Primary | Secondary |
| :- | :- | :- |
| Closed or fully open | Layered on top | Underneath |
| Partially folded, portrait | Bottom half | Top half |
| Partially folded, landscape | Trailing half | Leading half |

Use it where you would otherwise use a `ZStack` and the two views have a clear foreground and background relationship. Note that the primary view is the *foreground* (the controls), not the main content.

### Limit the axes

```swift
.arrangementViewStyle(.split.axes(.horizontal))   // side by side only; primary alone when vertical
.arrangementViewStyle(.overlay.axes(.vertical))
```

### Placement rules

- Don't put an `ArrangementView` inside a `ScrollView`, `List`, `NavigationSplitView`, grid, or any container that lays out its children, because part of it can become unreachable.
- Don't put a `NavigationSplitView` or `TabView` inside an `ArrangementView`.
- It's a top-level arrangement for a screen's two main areas, not a general-purpose stack.

## Reserved regions

A reserved region is an area of a view that hardware or the system claims. There are two kinds:

- **`.occlusion`**: something covers the content (a camera, the Dynamic Island, window controls). Keep content out of it.
- **`.division`**: the content is split into parts (the fold). Keep each view entirely on one side.

Read them from a `GeometryProxy`:

```swift
func reservedRegions(
    kind: ReservedRegion.Kind,
    options: ReservedRegion.QueryOptions = [],
    layoutDirectionBehavior: LayoutDirectionBehavior = .mirrors
) -> [ReservedRegion]
```

Facts that prevent the common mistakes:

- **Only overlapping regions are returned**, including partial overlaps. A region's `frame` can start at negative coordinates or run past the view's size.
- **`frame` is in the reading view's coordinate space**, in points, with `(0, 0)` at the top-leading corner. Read regions in the view that positions the content, or the coordinates won't match.
- **`frame` already includes the region's `margins`.** Don't add them again.
- **Only active regions are returned by default.** The fold is active when partially folded and inactive when fully open, so a fully open device returns no division.
- **Frames are mirrored for right-to-left layouts by default**, which matches a layout that flips automatically. Pass `layoutDirectionBehavior: .fixed` only for content positioned in a way that doesn't flip.
- **Regions are `Identifiable`.** Use `id` to track one across queries or to drive a `ForEach`.

### Keep content out of an occlusion

```swift
struct TopControls: View {
    private let barHeight: CGFloat = 60

    var body: some View {
        GeometryReader { proxy in
            let barArea = CGRect(x: 0, y: 0, width: proxy.size.width, height: barHeight)
            let inset = proxy.reservedRegions(kind: .occlusion)
                .filter { $0.frame.intersects(barArea) }   // regions can be anywhere in the view
                .map(\.frame.maxY)
                .max() ?? 0

            ControlBar()
                .frame(height: barHeight)
                .padding(.top, inset)
                .frame(maxHeight: .infinity, alignment: .top)
        }
    }
}
```

Filter to the regions that intersect the area you're placing content in. Padding for every occlusion in the view pushes content away from a camera that's nowhere near it.

### Split content at the fold

The fold can run either way. Folded like a book, it's a vertical strip and content sits side by side. Folded like a laptop, it's a horizontal strip and content stacks. Compare the division's width and height to tell which:

```swift
GeometryReader { proxy in
    if let fold = proxy.reservedRegions(kind: .division).first {
        let f = fold.frame
        if f.height > f.width {                    // vertical fold: side by side
            HStack(spacing: 0) {
                Sidebar().frame(width: f.minX)
                Color.clear.frame(width: f.width)
                Detail().frame(width: proxy.size.width - f.maxX)
            }
        } else {                                   // horizontal fold: stacked
            VStack(spacing: 0) {
                Sidebar().frame(height: f.minY)
                Color.clear.frame(height: f.height)
                Detail().frame(height: proxy.size.height - f.maxY)
            }
        }
    } else {
        Detail()                                   // no active fold
    }
}
```

If the content is just two related views, use `ArrangementView` instead of this.

Guidance for folded layouts:

- Move only what's needed to keep things visible and tappable. Controls that jump or vanish when the device folds are hard to track.
- In a grid, use an even number of columns so the content divides cleanly at the fold.

### Reserve space before a region becomes active

```swift
let regions = proxy.reservedRegions(kind: .occlusion, options: .includeInactive)
```

Each region reports `isActive`, and an inactive region still has a frame. Leaving room for it stops the layout from jumping when the device folds or the inner camera turns on.

### In a custom `Layout`

Pass the regions in from a `GeometryReader` that wraps the layout. Region frames start at the reading view's `(0, 0)`, but the `bounds` in `placeSubviews(in:proposal:subviews:cache:)` may not. Compute positions from `(0, 0)` and add `bounds.origin` only when placing each subview.

## Camera and media

- An app that captures with AVFoundation can use the outer front camera, the inner front camera, and the rear camera. Folding or rotating can move the app to the other display and leave the selected camera facing the opposite way. Choose cameras by the direction they face, not by a fixed device, and re-evaluate on configuration changes.
- With the device fully open and the rear camera in use, an app can also show content on the outer display. This needs explicit registration as a camera capture accessory; see Apple's documentation before attempting it.
- Hero images and video that use `.scaledToFill()` crop differently on each display. Check the subject survives at both shapes, or choose between fill and fit from the available aspect ratio.

## Testing

Tools, in order of fidelity:

1. **iPhone Duo simulator** (Xcode 27.1 or later): the only way to see poses, the fold, and vertical bars.
2. **iPhone Mirroring** on macOS 27: resize the window to extremes in both directions.
3. **The resizable iOS simulator** in Device Hub.

Poses to exercise for each affected flow:

- Closed, on the outer display, portrait and landscape.
- Fully open, portrait and landscape.
- Partially folded like a book and like a laptop.
- Opening and closing in the middle of a task: while typing, with a sheet or popover up, mid-scroll, during playback.

What to look for:

- Bars: did they go vertical where expected, are the frequent actions still visible, and does the overflow menu contain what was pushed out?
- Sheets and popovers that end up awkwardly placed after a fold.
- Controls or text sitting in the fold or under a camera.
- State lost across a transition: scroll position, selection, text being edited, navigation path.

If the simulator or SDK isn't available, say so and report which of these were not checked. Building successfully isn't evidence that a pose works.

Apple also ships an app resizability skill that scans a project for many of these patterns. If the toolchain includes it, `xcrun agent skills export` exports it. It can complement [audit.md](audit.md); confirm what it flags by reading the code.

## Sources

- [Preparing your app for iPhone Duo](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo)
- [iPhone Duo preparation steps](https://developer.apple.com/iphone-duo/prepare/)
- [ReservedRegion](https://developer.apple.com/documentation/swiftui/reservedregion)
- [ArrangementView](https://developer.apple.com/documentation/swiftui/arrangementview)
- [toolbarVerticalBehavior(_:)](https://developer.apple.com/documentation/swiftui/view/toolbarverticalbehavior(_:))
- [Prepare your app for iPhone Duo: Tech Talk](https://developer.apple.com/videos/play/tech-talks/111461/)
