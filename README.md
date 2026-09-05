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
| `api-validation-principle` | Audits all API endpoints against design, validation, security, and reliability principles and creates a complete report under `docs/api_report/` on every run. |
| `clean-code-review` | Reviews frontend and backend code quality, design principles, and architecture conformance, then creates a prioritized numbered report. |

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

## Use the API validation skill

Install it into the application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill api-validation-principle --agent codex --agent claude-code --agent cursor
```

Ask the agent:

```text
Use api-validation-principle to review every API endpoint across this system and create an API report.
```

The skill inventories all services and operations in scope, checks each operation against the API baseline, and records evidence, issues, and missing verification. It creates the next `docs/api_report/report_NNN.md` in the reviewed project on every run, including clean runs, starting at `report_001.md` and preserving earlier reports. It creates the directory when needed. It also reports when no APIs are found or a review cannot be completed.

A system passes only when inventory coverage is complete and every applicable check passes. Unverified endpoints remain visible in the report. To remediate findings, explicitly ask the agent to fix them and revalidate; an audit by itself changes only the reports.

## Use the clean-code skill

Install it into the application project:

```bash
npx skills add mohammedashrafdagga/coding-skills --skill clean-code-review --agent codex --agent claude-code --agent cursor
```

Ask the agent to use `clean-code-review` for a repository-wide quality and architecture assessment. For example:

```text
Use clean-code-review to assess the frontend and backend, including DDD and feature boundaries.
```

Every run creates the next `docs/clean-code-report/report_NNN.md` with coverage, evidence-backed findings, stable issue IDs, and a dependency-aware remediation plan. Reviewing does not modify application code unless the user explicitly asks for fixes.

## Add future skills

Add each new skill as a sibling directory at the repository root. The directory name must match the `name` in its `SKILL.md` frontmatter.

```text
coding-skills/
├── README.md
├── api-validation-principle/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── api-baseline.md
│       └── report-format.md
├── clean-code-review/
│   ├── SKILL.md
│   ├── agents/
│   │   └── openai.yaml
│   └── references/
│       ├── quality-baseline.md
│       └── report-format.md
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
