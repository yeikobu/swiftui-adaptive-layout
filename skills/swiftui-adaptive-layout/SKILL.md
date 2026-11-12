---
name: swiftui-adaptive-layout
description: Adapt, implement, or review SwiftUI layouts so they work across iPad, resizable iPhone windows, and the foldable iPhone Duo, following Apple's Human Interface Guidelines. Use whenever a SwiftUI app needs to support iPad or iPhone Duo, handle multitasking, Split View, Stage Manager, or resizable windows, or when deciding between a sidebar, NavigationSplitView, and tabs. Also use for layouts that break when a window resizes or a device folds or rotates, for code that depends on UIScreen.main, device idiom, orientation, or hardcoded sizes, and for work involving size classes, safe areas, the fold, reserved regions, ArrangementView, or vertical toolbars.
---

# SwiftUI adaptive layout

Build one coherent, native experience across compact and spacious windows. The core idea: **lay out for the space a view actually has, and keep it correct as that space changes.** A window's size is no longer implied by the device, and it changes while the app is running.

Follow the project's own instructions, design system, supported OS versions, and the scope that was asked for. Apple's documentation is the authority; the reference files summarize it and link to it.

## Pick the mode

- **Review** (assess, audit, "is this ready for iPad?", "was this change necessary?"): follow [references/review.md](references/review.md). Report findings in its format and don't edit files.
- **Implement or adapt**: follow the workflow below.

## Workflow

### 1. Audit before designing

Find the deployment target, the SDK, the navigation structure, and the flows in scope. Then run the searches in [references/audit.md](references/audit.md) to locate screen-based sizing, device and orientation checks, hardcoded dimensions, geometry read once, custom bars, and unsafe insets. Include UIKit bridges. Read each hit in context: a search hit is a lead, not a defect.

### 2. Lay out from available space

- Prefer containers that adapt by themselves: adaptive grids, `ViewThatFits`, `NavigationSplitView`, system toolbars and presentations.
- When a layout must branch, branch on fit, size class, or measured space, never on device idiom or physical orientation. Regular width doesn't mean iPad and compact doesn't mean iPhone.
- Measure only when needed, with `onGeometryChange` or `GeometryReader`, reduce the measurement to the value the layout needs, and react to changes instead of capturing it once.
- Don't use `UIScreen.main`. Use `@Environment(\.displayScale)` for scale.
- Give large windows purpose: cap line length, show more context, keep controls reachable. A stretched phone layout is not an iPad layout.

[references/audit.md](references/audit.md) has before-and-after code for each of these.

### 3. Decide the navigation

Ask whether a sidebar or list-detail layout helps people here: do they switch often between areas, or refer to a list while inspecting items? A sidebar isn't mandatory. A linear flow, a few destinations, or space-hungry content is often better served by a `NavigationStack` or tabs. Don't restructure navigation that wasn't part of the request; recommend it separately.

If the app is already built from system containers and the audit is clean, it already adapts. Further changes are improvements to weigh, not fixes to apply, and each one should earn its place: an adaptation that changes how the app looks on a conventional iPhone needs a stronger reason than one that doesn't.

Whatever the structure, selection, navigation paths, and in-progress work must survive columns collapsing and expanding. When a home screen becomes the sidebar, make sure hiding it can't take the app's main action with it. Read [references/navigation.md](references/navigation.md) for the decision table, a `NavigationSplitView` pattern, and the state-preservation rules.

### 4. Handle iPad specifics

Support resizable windows, multitasking widths, and every orientation. Check sheets, popovers, and the keyboard at the narrowest width as well as the widest. Where the task involves interaction, account for keyboard and pointer input.

### 5. Handle iPhone Duo specifics

**Read [references/iphone-duo.md](references/iphone-duo.md) before writing or reviewing any iPhone Duo code.** These APIs are newer than most training data, and guessing at them produces code that doesn't compile. The file covers what each SDK version gives you, vertical bars, `ArrangementView`, reserved regions for the fold and cameras, availability checks, and the poses to test.

In short: system containers already adapt, so most views need nothing Duo-specific. Custom bars, two-pane custom layouts, and views that position content themselves do.

### 6. Check accessibility and environment

Dynamic Type through the accessibility sizes at the narrowest width, VoiceOver labels and reading order after the layout rearranges, touch targets, right-to-left layout, and light and dark appearance. Confirm focus and keyboard occlusion hold up during a resize.

### 7. Validate and report honestly

Exercise the affected flows in previews, a simulator, or hardware: small and large iPad, narrow and wide windows, a conventional iPhone, and the iPhone Duo poses. Resize in the middle of a task, with a sheet up, and while typing. Run the project's tests and add regression tests where they're meaningful.

State what was verified and how. If an SDK, simulator, or device wasn't available, list what remains untested. Don't claim device validation or Human Interface Guidelines compliance from reading code or from a successful build.

## Constraints

- Check each API's availability against the deployment target and provide a fallback. Building with a newer SDK doesn't require raising the deployment target.
- Don't change deployment targets, supported orientations, or app capabilities without a reason that comes from the task, and say so when you do.
- Preserve existing behavior and user state. Adapting a layout shouldn't change what the app does.

## Reference files

| File | Read it when |
| :- | :- |
| [references/audit.md](references/audit.md) | Starting any task: search patterns, how to judge each hit, before-and-after fixes |
| [references/navigation.md](references/navigation.md) | Choosing or changing navigation, or state is lost on resize |
| [references/iphone-duo.md](references/iphone-duo.md) | Anything involving iPhone Duo, the fold, cameras, or vertical bars |
| [references/review.md](references/review.md) | The request is a review: process, severity scale, report format |

## Apple references

- [Layout](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Designing for iPadOS](https://developer.apple.com/design/human-interface-guidelines/designing-for-ipados)
- [Designing for iOS](https://developer.apple.com/design/human-interface-guidelines/designing-for-ios)
- [Sidebars](https://developer.apple.com/design/human-interface-guidelines/sidebars)
- [Split views](https://developer.apple.com/design/human-interface-guidelines/split-views)
- [Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Preparing your app for iPhone Duo](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo)
- [iPhone Duo preparation steps](https://developer.apple.com/iphone-duo/prepare/)
