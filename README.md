# SwiftUI Adaptive Layout

A Codex skill for adapting SwiftUI apps to iPad, resizable iPhone windows, and iPhone Duo, using Apple's Human Interface Guidelines. It helps assess navigation, including when a sidebar is useful, and covers resizing, accessibility, and testing.

## Install in Codex

Ask Codex:

```text
Use $skill-installer to install the skill from https://github.com/yeikobu/swiftui-adaptive-layout/tree/main/skills/swiftui-adaptive-layout
```

Start a new Codex session after installing, then invoke it with `$swiftui-adaptive-layout` or ask Codex to adapt a SwiftUI app for iPad and iPhone Duo.

### Manual installation

Clone the repository and copy the skill directory into your personal skills folder:

```bash
git clone https://github.com/yeikobu/swiftui-adaptive-layout.git
mkdir -p ~/.agents/skills
cp -R swiftui-adaptive-layout/skills/swiftui-adaptive-layout ~/.agents/skills/
```

The destination must not already contain a `swiftui-adaptive-layout` folder; review any existing installation before replacing it. Start a new Codex session after installation. To make the skill available only in one project, copy the folder into that project's `.agents/skills/` instead.

## Structure

```text
skills/
└── swiftui-adaptive-layout/
    └── SKILL.md
```

The skill references current Apple documentation. Check API availability and test against the target app's supported platforms and SDKs.
