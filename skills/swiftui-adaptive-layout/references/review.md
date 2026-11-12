# Reviewing layouts

Use this when the request is to review, audit, or assess, and no code changes were asked for. A review reports findings; it doesn't edit files unless the person then asks for fixes.

## Process

1. Establish the scope: which screens or flows, which devices matter to the person, and the deployment target and SDK.
2. Run the searches in [audit.md](audit.md) across the scope, and read each hit in context.
3. Read the navigation structure and the affected views for the problems a search can't find: lost state, unreachable columns, presentations that lose their anchor.
4. If the project builds and a simulator is available, resize the affected flows and, for iPhone Duo, go through the poses in [iphone-duo.md](iphone-duo.md#testing). A finding seen in a running app is stronger than one inferred from code, so say which it is.
5. Report in the format below.

## Reviewing a change that was already made

When the question is "was this adaptation necessary?" or "did I overdo it?", the subject is a diff, not the whole app. Judge each part of the change separately, because one commit usually mixes essential fixes with optional polish.

1. **Establish the baseline.** Run the [audit.md](audit.md) searches against the code *before* the change (`git show <commit>^:<path>` or the diff's removed lines). If the baseline had no screen-based sizing, device checks, or custom bars and was built from system containers, the app already adapted and nothing in the change was required for correctness.
2. **Sort each part of the diff** into one of these:

   | Verdict | Meaning |
   | :- | :- |
   | **Required** | Without it, content or an action is unreachable or broken in a supported configuration |
   | **Worthwhile** | The app worked without it, but the change makes real use of the space or removes a clear awkwardness |
   | **No visible effect** | The condition it handles doesn't occur in this app today, such as a width cap on a view that only appears in narrow containers |
   | **Costs something** | It fixes one configuration by changing behavior in others; name what was traded |

3. **Check for regressions where most people are.** An adaptation for iPad or iPhone Duo is judged first by whether a conventional iPhone still looks and behaves as before. Run the compact layout before anything else.
4. **Observe, don't infer, where the verdict depends on system behavior.** Whether an item disappears from a vertical bar or a sidebar shows by default is quick to see in a simulator and easy to get wrong from memory.

Report the per-part verdicts as a table ahead of the findings, then list as findings only the problems the change introduced or exposed.

## Severity

Rate by what the person using the app experiences, and weigh it against who that person is. A missing text label is minor in a developer tool and major in an app for people who rely on reading the button.

| Severity | Meaning | Examples |
| :- | :- | :- |
| **Blocker** | Content or an action is unreachable, or work is lost | A control under the camera or in the fold, a column with no way back, a draft discarded on resize, a clipped form with no scrolling |
| **Major** | The layout is clearly wrong but everything still works | A phone layout stretched across an iPad, text lines spanning the full width, a text label replaced by a bare icon in an app whose audience relies on reading it, a primary action that vanishes when the sidebar is hidden |
| **Minor** | Polish | Uneven margins at one size, a sheet taller than its content, a missing hover effect |

Don't inflate. A fixed 44-point icon frame is not a finding. A deprecated API that still adapts correctly is a note, not a major issue.

## Report format

Lead with the verdict, then the findings ordered by severity. Use this structure:

```markdown
## Summary

Two or three sentences: how well the scope adapts today, the most serious problem,
and the size of the work to fix it.

## Findings

### 1. Short statement of the defect — Blocker
- **Where**: `Sources/Library/LibraryView.swift:42`
- **Breaks in**: iPad in Split View at one-third width; iPhone Duo outer display
- **What happens**: what the person sees or loses, in one or two sentences
- **Cause**: the assumption in the code, with the relevant line quoted
- **Fix**: the specific change, with a short snippet when it clarifies
- **Evidence**: observed in the iPad simulator | inferred from code

### 2. ...

## Not verified

What could not be checked and why: no iPhone Duo simulator, the project doesn't build,
flows behind a login, hardware-only behavior.

## Recommendations

Optional. Larger changes worth considering that go beyond the scope, such as
restructuring navigation. Keep these separate from defects.
```

Rules for the report:

- **One finding per root cause.** If `UIScreen.main.bounds` appears in twelve files for the same purpose, report it once and list the locations.
- **Name the configuration that breaks.** "Doesn't adapt" is not actionable; "the detail column is 180 points wide in a 600-point window" is.
- **Separate evidence from inference.** State whether each finding was observed or deduced from reading the code.
- **Don't claim compliance.** Say what was checked. A review can show problems; it can't certify that the app follows the Human Interface Guidelines or works on hardware nobody ran it on.
- **Report clean results plainly.** If the scope adapts well, say so and list what was checked, instead of padding the report with minor notes.
