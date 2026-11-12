# SwiftUI Adaptive Layout

An agent skill for adapting SwiftUI apps to iPad, resizable iPhone windows, and iPhone Duo, following Apple's Human Interface Guidelines. It works with **Claude** (Claude Code and the Claude apps) and **Codex**.

With the skill installed, your agent will:

- Audit the project for screen-based sizing, device and orientation checks, and hardcoded dimensions.
- Design for the space each container actually has, not for a device name or orientation.
- Assess navigation, including when a sidebar or list-detail layout helps and when it doesn't.
- Handle resizable windows, multitasking, and iPhone Duo's fold, cameras, and vertical bars, using `ArrangementView` and reserved regions with availability fallbacks.
- Preserve selection, navigation, and in-progress work as layouts collapse and expand.
- Check Dynamic Type, VoiceOver, right-to-left layout, and keyboard behavior.
- Validate the affected flows across iPad sizes, resizable windows, and iPhone Duo poses.
- In a review, report findings by severity with the file, the configuration that breaks, and the fix.

## Install

| You use | Quickest way |
| :- | :- |
| Claude Code | [Install as a plugin](#claude-code) |
| Claude desktop app or claude.ai | [Download the ZIP and upload it](#claude-desktop-app-and-claudeai) |
| Codex | [Ask Codex to install it](#codex) |
| Several agents at once | [Use the `skills` CLI](#any-agent-with-the-skills-cli) |

### Claude Code

Run these two commands inside a Claude Code session:

```text
/plugin marketplace add yeikobu/swiftui-adaptive-layout
```

```text
/plugin install swiftui-adaptive-layout@yeikobu
```

Or run the same thing from your shell:

```bash
claude plugin marketplace add yeikobu/swiftui-adaptive-layout
```

```bash
claude plugin install swiftui-adaptive-layout@yeikobu
```

To get later versions, run `claude plugin marketplace update yeikobu` and then `claude plugin update swiftui-adaptive-layout@yeikobu`.

<details>
<summary>Install manually, without the plugin system</summary>

Copy the skill folder into your personal skills directory. It is then available in every project:

```bash
git clone https://github.com/yeikobu/swiftui-adaptive-layout.git
mkdir -p ~/.claude/skills
cp -R swiftui-adaptive-layout/skills/swiftui-adaptive-layout ~/.claude/skills/
```

To make it available in one project only, and to share it with your team through the repository, copy the folder into that project's `.claude/skills/` instead.

If `~/.claude/skills/` didn't exist before, run `/reload-skills` or start a new session.

</details>

### Claude desktop app and claude.ai

1. Download [`swiftui-adaptive-layout.zip`](https://github.com/yeikobu/swiftui-adaptive-layout/releases/latest/download/swiftui-adaptive-layout.zip) from the latest release.
2. In Claude, open **Customize > Skills**, click **+**, choose **Create skill**, and then **Upload a skill**.
3. Select the ZIP, without unpacking it, and make sure the skill is toggled on.

Skills need code execution to be enabled. On individual plans, turn it on in **Settings > Capabilities**. On Team and Enterprise plans, an owner enables it for the organization.

Don't use GitHub's green **Code > Download ZIP** button for this. That archive contains the whole repository, and the upload is rejected because `SKILL.md` has to be in the ZIP's top-level folder.

<details>
<summary>Build the ZIP yourself</summary>

Compress the inner skill folder, not the repository and not `skills/`:

```bash
git clone https://github.com/yeikobu/swiftui-adaptive-layout.git
cd swiftui-adaptive-layout/skills
zip -r swiftui-adaptive-layout.zip swiftui-adaptive-layout
```

The ZIP must contain `swiftui-adaptive-layout/SKILL.md` and `swiftui-adaptive-layout/references/`.

</details>

### Codex

Ask Codex:

```text
Use $skill-installer to install the skill from https://github.com/yeikobu/swiftui-adaptive-layout/tree/main/skills/swiftui-adaptive-layout
```

Start a new Codex session after installing.

<details>
<summary>Install manually</summary>

```bash
git clone https://github.com/yeikobu/swiftui-adaptive-layout.git
mkdir -p ~/.agents/skills
cp -R swiftui-adaptive-layout/skills/swiftui-adaptive-layout ~/.agents/skills/
```

The destination must not already contain a `swiftui-adaptive-layout` folder; review any existing installation before replacing it. To make the skill available in one project only, copy the folder into that project's `.agents/skills/` instead. Start a new Codex session after installing.

</details>

### Any agent, with the `skills` CLI

The [`skills`](https://github.com/vercel-labs/skills) CLI installs the skill for the agent you name. It needs a recent Node.js.

```bash
npx skills add yeikobu/swiftui-adaptive-layout -a claude-code -g
```

Replace `claude-code` with `codex` or another supported agent. Drop `-g` to install into the current project instead of your home directory.

## Use

Describe the work and the agent picks up the skill on its own:

```text
Adapt the library screen for iPad and iPhone Duo. Decide whether a sidebar makes sense.
```

```text
Review this SwiftUI view for fixed sizes and device checks that break in resizable windows.
```

You can also call it by name:

| Where | Command |
| :- | :- |
| Claude Code, installed as a plugin | `/swiftui-adaptive-layout:swiftui-adaptive-layout` |
| Claude Code, installed manually or with the `skills` CLI | `/swiftui-adaptive-layout` |
| Codex | `$swiftui-adaptive-layout` |

## Uninstall

- **Claude Code plugin**: `claude plugin uninstall swiftui-adaptive-layout@yeikobu`
- **Manual install**: delete the `swiftui-adaptive-layout` folder from `~/.claude/skills/` or `~/.agents/skills/`
- **Claude apps**: remove the skill in **Customize > Skills**

## Structure

```text
.claude-plugin/
└── marketplace.json        # lets Claude Code install the repo as a plugin
.github/workflows/
└── release.yml             # attaches the skill ZIP to each published release
skills/
└── swiftui-adaptive-layout/
    ├── SKILL.md            # workflow and constraints, always loaded with the skill
    └── references/         # loaded only when the task needs them
        ├── audit.md        # search patterns and before-and-after fixes
        ├── navigation.md   # sidebar, split view, and tab decisions; preserving state
        ├── iphone-duo.md   # vertical bars, ArrangementView, reserved regions, testing
        └── review.md       # review process, severity scale, report format
```

The skill references current Apple documentation. Check API availability and test against the target app's supported platforms and SDKs.
