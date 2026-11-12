# Navigation across compact and spacious windows

How to decide the navigation structure for a larger window, and how to keep state intact as it collapses and expands.

## Contents

- [Choose the structure](#choose-the-structure)
- [List-detail with NavigationSplitView](#list-detail-with-navigationsplitview)
- [Tabs that become a sidebar](#tabs-that-become-a-sidebar)
- [Keep state through collapse and expand](#keep-state-through-collapse-and-expand)
- [When the sidebar is the home screen](#when-the-sidebar-is-the-home-screen)
- [Sidebar content](#sidebar-content)
- [Toolbars and presentations](#toolbars-and-presentations)
- [Other input on iPad](#other-input-on-ipad)

## Choose the structure

A sidebar is a tool, not a requirement of large screens. Decide from the app's information hierarchy:

| The app has | Prefer | Why |
| :- | :- | :- |
| A list people keep referring to while they inspect and switch between items (mail, notes, files) | `NavigationSplitView`, two columns | The list stays visible, so switching doesn't cost a round trip |
| Collections, then items, then detail (library › album › track) | `NavigationSplitView`, three columns | Each level stays in view |
| Three to five peer areas with shallow content | `TabView` | A sidebar would spend width on five rows |
| Many top-level areas, or user-created collections alongside fixed areas | `TabView` with `.tabViewStyle(.sidebarAdaptable)` | Tabs when compact, a sidebar when there's room |
| A linear flow (onboarding, checkout, a form) | `NavigationStack` | There's nothing to switch between |
| Space-hungry content (canvas, map, video, reader) | `NavigationStack`, with an inspector or sheet for secondary content | A permanent column takes space the content needs |

Then check the choice against the task:

- **How often do people switch?** Rare switching doesn't justify a permanent column.
- **How much width does the detail need?** If the detail is cramped beside a sidebar at common window sizes, the sidebar is costing more than it gives.
- **Was a navigation change requested?** If the task is "make this screen work on iPad", adapt that screen. Propose a restructure as a separate recommendation instead of doing it unasked.

## List-detail with NavigationSplitView

```swift
struct LibraryView: View {
    @State private var selection: Album.ID?
    @State private var columnVisibility: NavigationSplitViewVisibility = .automatic
    let albums: [Album]

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            List(albums, selection: $selection) { album in
                NavigationLink(value: album.id) {
                    Label(album.title, systemImage: "square.stack")
                }
            }
            .navigationTitle("Library")
        } detail: {
            if let album = albums.first(where: { $0.id == selection }) {
                AlbumDetail(album: album)
            } else {
                ContentUnavailableView("Select an album", systemImage: "square.stack")
            }
        }
    }
}
```

What this gets right, and what to preserve when adapting an existing screen:

- **Selection is state owned above the split view**, so it survives the columns collapsing into a stack and expanding again.
- **The detail is derived from the selection**, not pushed imperatively. If the selected album is deleted, the lookup fails and the empty state appears, instead of a stale detail.
- **The empty state is explicit.** In a wide window with nothing selected, the detail column is visible and must say something useful. When a sensible default exists (the first item, the last opened), select it instead.
- **`columnVisibility` is `.automatic`** unless there's a reason. Forcing `.all` on a narrow window squeezes the detail.

Tune column widths only when the defaults fail, with `navigationSplitViewColumnWidth(min:ideal:max:)`. Choose how columns behave when space is tight with `navigationSplitViewStyle(.balanced)` (the detail shrinks) or `.prominentDetail` (the sidebar overlays).

When collapsed, the split view shows one column. Control which one with `NavigationSplitView(preferredCompactColumn:)` so a compact window opens on the list or the detail as the flow requires.

## Tabs that become a sidebar

```swift
TabView(selection: $selectedTab) {
    Tab("Home", systemImage: "house", value: .home) { HomeView() }
    Tab("Search", systemImage: "magnifyingglass", value: .search, role: .search) { SearchView() }
    TabSection("Collections") {
        ForEach(collections) { collection in
            Tab(collection.name, systemImage: "folder", value: AppTab.collection(collection.id)) {
                CollectionView(collection: collection)
            }
        }
    }
}
.tabViewStyle(.sidebarAdaptable)
```

Each tab should own its own `NavigationStack` so every area keeps an independent path. `TabSection` content appears as a group in the sidebar and stays out of the compact tab bar, which is how a long list of collections fits both forms. This style requires iOS 18; check the deployment target.

## Keep state through collapse and expand

Resizing is not navigation. Nothing the person did should be lost because a window got narrower.

- **Selection and paths**: hold them in `@State` or a model above the adaptive container. For restoration across launches use `@SceneStorage`.
- **Don't swap the root view by size class.** `if sizeClass == .regular { SplitRoot() } else { StackRoot() }` destroys and rebuilds everything below it, discarding scroll positions, text fields, and navigation paths on every crossing. Use one container that adapts (`NavigationSplitView` collapses by itself), or switch only the layout with `AnyLayout`.
- **`ViewThatFits` rebuilds its chosen child** when the choice changes. Keep stateful content out of it, or lift the state above it.
- **In-progress work**: text being typed, a draft, a half-filled form. Keep it in a model, not in a view that might be recreated.
- **Presentations**: a sheet or popover anchored to a control that disappears when the layout collapses loses its anchor. Present from a stable ancestor.
- **Hidden columns must stay reachable.** When the sidebar is hidden, the system toggle provides the way back. If the navigation bar is hidden or replaced, that route is gone; restore one.
- **Removed items**: when the selected item is deleted or the active collection changes, clear or move the selection deliberately.

## When the sidebar is the home screen

Converting a single-column app often turns its home screen into the sidebar: the list still holds settings, summary rows, and the app's main action, and the detail shows whatever was selected. This is a legitimate layout, but the sidebar is then not optional chrome, and the system's defaults assume it is.

- **Hiding the sidebar hides the app.** The toggle the system adds collapses the column, and with it the primary action and everything else on the home screen. What remains is a detail stretched across the whole window, and the only way back is an unlabeled icon. If the sidebar holds anything people need that the detail doesn't offer, remove the toggle so the column stays:

  ```swift
  List(selection: $selection) { ... }
      .toolbar(removing: .sidebarToggle)
  ```

  Alternatively, repeat the primary action in the detail's toolbar so it survives the sidebar being hidden.
- **Don't open on an empty detail.** With a home-screen sidebar, the detail is most of the window. Select a sensible default in regular width (the most recent item) so the space is used from the first moment. Apply the default only where both columns are visible; in compact width a selection pushes the detail and people would land past the home screen.
- **Distinguish "nothing selected" from "nothing to select".** An empty state that says "select an item" is wrong when the list is empty. Show guidance toward creating the first item instead.
- **Check what the column width does to the home content.** A sidebar is roughly 320 points wide. Rows written for a full-width phone screen wrap, and a bottom-pinned button covers more of the list. Scroll the sidebar at the smallest regular width, such as iPhone Duo's inner display, and confirm the selectable rows are reachable without hunting.

## Sidebar content

- Group related items under short section headings.
- Use familiar labels with symbols (`Label`), since a collapsed or narrow sidebar may show less text.
- Keep the hierarchy shallow. Two levels in a sidebar is usually the limit before it becomes a tree to manage.
- Show the current selection clearly; `List(selection:)` does this.
- Put actions that affect the list (add, edit) in the sidebar's toolbar, and actions that affect the detail in the detail's toolbar, so they move with their column.

## Toolbars and presentations

- Put actions in system toolbars with semantic placements (`.primaryAction`, `.confirmationAction`, `.cancellationAction`). The system then moves them to the right place for the window's width and, on iPhone Duo, into a vertical bar. Hand-built bars can't adapt. See [iphone-duo.md](iphone-duo.md#vertical-bars).
- Give a toolbar action both a title and an icon when it should adapt to every presentation, because each one shows a different part. This has a cost: a horizontal bar then shows the icon and drops the title, and a text-only button stays visible on iPhone Duo anyway. See [the trade-off](iphone-duo.md#the-cost-of-adding-an-icon) before converting text buttons.
- Sheets on a large window can be dismissed by tapping outside them, which is far easier to do by accident than a swipe on a phone. Add `interactiveDismissDisabled` while a form holds unsaved input.
- Check what falls into overflow at the narrowest width. The most frequent action should not be the first to go.
- Sheets on a large window: set `presentationDetents` or `presentationSizing(.form)` / `.page` so a short form isn't a full-height sheet on a 13-inch iPad.
- Popovers become sheets in compact width. Make sure the content has a way to dismiss in both forms.
- An inspector (`inspector(isPresented:)`) suits secondary detail next to space-hungry content; it's a trailing column when wide and a sheet when compact.

## Other input on iPad

When the task touches interaction, not just layout:

- **Keyboard**: add `keyboardShortcut` for primary actions, and make sure lists and forms are focusable and navigable with arrow keys and Tab.
- **Pointer**: system controls get hover effects automatically; add `hoverEffect` to custom tappable views, and offer context menus where a long press exists.
- **Multitasking**: the same scene can be very narrow next to another app. The compact layout has to work on an iPad, not only on an iPhone.
- **Multiple windows**: if the app supports several scenes, each needs its own selection and navigation state. `@SceneStorage` and per-scene models keep them independent.
