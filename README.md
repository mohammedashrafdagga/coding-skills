# Coding Skills

A private collection of reusable Agent Skills for software work. Each skill lives in its own top-level directory, so more skills can be added without changing the repository's installation model.

## Supported agents

This repository officially supports only:

- Codex
- Claude Code
- Cursor

The `skills` CLI supports many agents, but this collection is maintained and tested only for these three. Always use the explicit agent flags below; do not use `--all` or `--agent '*'`.

## Available skills

| Skill | Purpose |
| --- | --- |
| `software-security-baseline` | Reviews minimum practical security controls for small-business applications and tracks actionable issues in numbered Markdown reports. |

## Install from the private repository

The machine must have access to the private GitHub repository through Git credentials, GitHub CLI authentication, or SSH.

First, list the available skills:

```bash
npx skills add mohammedashrafdagga/coding-skills --list
```

Install one skill into the current application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill software-security-baseline --agent codex --agent claude-code --agent cursor
```

Install every current and future skill from the collection:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill '*' --agent codex --agent claude-code --agent cursor
```

Project scope is the default. Add `--global` to any install command to make the selected skills available across projects:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill software-security-baseline --agent codex --agent claude-code --agent cursor --global
```

If GitHub shorthand authentication does not work, use the SSH repository URL:

```bash
npx skills add git@github.com:mohammedashrafdagga/coding-skills.git --skill software-security-baseline --agent codex --agent claude-code --agent cursor
```

To update installed skills:

```bash
npx skills update
```

## Use the security skill

Ask the agent to use `software-security-baseline` and review the current application. For example:

```text
Use software-security-baseline to review this application before release.
```

The skill will:

1. determine the application and review scope;
2. ensure the application has a `security/` directory;
3. inspect earlier numbered security reports;
4. assess core and applicable conditional controls;
5. create the next `security/report_NNN.md` only when a control fails or still needs verification;
6. avoid creating a report when the review is clean.

Security reports can contain sensitive architectural and vulnerability information. Keep the application repository and its reports access-controlled, and never place live secrets in a report.

## Add future skills

Add each new skill as a sibling directory at the repository root. The directory name must match the `name` in its `SKILL.md` frontmatter.

```text
coding-skills/
├── README.md
├── software-security-baseline/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── minimum-baseline.md
│       └── report-format.md
└── future-skill/
    └── SKILL.md
```

After adding a skill, validate it and confirm repository discovery with:

```bash
npx skills add . --list
```

The collection follows the Agent Skills `SKILL.md` format. Files under a skill's `agents/` directory may provide agent-specific metadata and can be ignored by agents that do not use them.
