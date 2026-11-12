---
name: swiftui-adaptive-layout
description: Adapt, implement, or review SwiftUI app layouts for iPad, resizable iPhone windows, and iPhone Duo, following Apple's Human Interface Guidelines. Use for responsive navigation, resizing, folding poses, safe areas, and accessibility across these devices.
---

# SwiftUI adaptive layout

Build a coherent, native experience across compact and spacious windows, including iPad and iPhone Duo. Follow the project's instructions, existing design system, supported OS versions, and requested scope. Apple guidance is the authority; consult the linked current documentation for the APIs and HIG sections relevant to the work. Do not claim full HIG compliance from a checklist alone.

## Workflow

1. Inspect the app's deployment target, SDK, existing navigation, reusable components, and affected flows. Identify fixed screen dimensions, cached geometry, device or orientation-based layout decisions, custom bars, unsafe insets, and assumptions about keyboards, sheets, and media. Search UIKit bridges too if present. Preserve behavior and user state through resizing.
2. Design for the space available to each container, not a device name. Prefer adaptive system containers, layout priorities, `ViewThatFits`, size classes, and local geometry where appropriate. Use `GeometryReader` or `onGeometryChange` only where measurements are needed; react to changes, not just initial dimensions. Avoid treating regular width as synonymous with iPad or compact width as synonymous with iPhone. Do not use physical orientation to infer available layout space. Avoid `UIScreen.main` for layout; if screen information is actually necessary, obtain it from the relevant window scene dynamically. Use SwiftUI's display scale environment when needed.
3. Evaluate whether a sidebar would help people navigate between main areas or collections, or keep a frequently used list visible while inspecting and switching between details. Consider the app's information hierarchy, frequency of switching, available width, and the space required by the detail. A sidebar is not mandatory: a simple flow, a few main destinations, or space-sensitive content may be better served by a `NavigationStack` or tab bar. Respect the requested scope and existing navigation; do not redesign an unrelated flow just because a larger screen is available.
   - When a sidebar or list-detail layout fits the task, prefer adaptive system navigation such as `NavigationSplitView`, or a sidebar-adaptable `TabView` when appropriate. Group related items with short, clear section headings and familiar labels or symbols; keep hierarchy shallow and provide a clear indication of the selected item. Consider an empty detail state where there is no meaningful default selection.
   - Preserve selection, navigation, and in-progress work as columns collapse and expand. Make hidden columns discoverable and accessible, ensure a logical compact navigation path, and verify behavior when a selected item is removed or the active collection changes. Avoid assuming the same layout fits every iPad or iPhone Duo window.
   - Use system toolbars, semantic placements, and native presentations; give adaptable toolbar actions both a title and an icon. Check action visibility and overflow when bars become vertical.
4. For iPad, accommodate resizable windows, multitasking and orientations. Use space to elevate content while keeping lines, forms, and controls comfortable to scan and reach. Consider keyboard, pointer, and other input modes when relevant. Keep dialogs, sheets, popovers, and keyboard interactions usable at narrow widths as well as large sizes.
5. For iPhone Duo, test outer and inner displays, opening and closing during a task, rotations, partial folds, and split-screen configurations where available. Respect asymmetric safe areas and margins independently on each edge; keep foreground controls in safe areas and extend only suitable backgrounds. Prefer system containers that adapt around system UI. Inspect critical content and controls near the fold and cameras. For custom layouts, consult Apple's current `ReservedRegion` and `ArrangementView` guidance; use them only when appropriate and available, with deployment-target-compatible fallbacks. Review media cropping and camera behavior if the app uses them.
6. Support Dynamic Type, including accessibility sizes, VoiceOver labels and actions, adequate touch targets, right-to-left layout, and light/dark appearances as appropriate. Preserve content hierarchy, discoverability, and input focus; test keyboard occlusion and state during transitions.
7. Validate the affected flows in Xcode previews and/or simulator or hardware as appropriate: small and large iPad, narrow and wide resizable windows, a conventional iPhone, and iPhone Duo poses. Exercise navigation, forms, presentations, and resizing mid-task. Run relevant project tests and add meaningful regression tests where feasible. If the necessary SDK, simulator, or hardware is unavailable, report what was actually verified and what remains untested; do not claim device validation.

## SDK and availability

Check the installed SDK and each API's availability before use. Apple describes different Duo behavior for builds using iOS 26 SDK or earlier, iOS 27 SDK, and iOS 27.1 SDK or later; the latter supports the full display and vertical system bars. Building with a newer SDK does not by itself require raising the app's minimum deployment target. Do not change deployment targets or app capabilities without a task-specific reason. If available, Apple's App Resizability skill can supplement inspection (`xcrun agent skills export`); follow its relevant guidance but verify its findings independently.

## Apple references

- [iPhone Duo preparation](https://developer.apple.com/iphone-duo/prepare/)
- [Preparing your app for iPhone Duo](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo)
- [Prepare your app for iPhone Duo: Tech Talk](https://developer.apple.com/videos/play/tech-talks/111461/)
- [Designing for iPadOS](https://developer.apple.com/design/human-interface-guidelines/designing-for-ipados)
- [Layout](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Sidebars](https://developer.apple.com/design/human-interface-guidelines/sidebars)
- [Split views](https://developer.apple.com/design/human-interface-guidelines/split-views)
- [Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Designing for iOS](https://developer.apple.com/design/human-interface-guidelines/designing-for-ios)
