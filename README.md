# Software Security Baseline

A private, reusable Agent Skill for checking the minimum practical security controls in small-business software. It reviews application evidence, identifies release blockers, and creates sequential Markdown reports under the reviewed application's `security/` directory when issues need attention.

## Supported agents

This repository officially supports only:

- Codex
- Claude Code
- Cursor

The `skills` CLI supports many agents, but this skill is maintained and tested only for these three. Always use the explicit agent flags below; do not use `--all` or `--agent '*'`.

## Install from the private repository

The machine must have access to the private GitHub repository through Git credentials, GitHub CLI authentication, or SSH.

From the root of an application project, install the skill for all three supported agents:

```bash
npx skills add mohammedashrafdagga/software-security-baseline --skill software-security-baseline --agent codex --agent claude-code --agent cursor
```

This uses project scope by default. To make the skill available across all projects, add `--global`:

```bash
npx skills add mohammedashrafdagga/software-security-baseline --skill software-security-baseline --agent codex --agent claude-code --agent cursor --global
```

If GitHub shorthand authentication does not work, use the SSH repository URL:

```bash
npx skills add git@github.com:mohammedashrafdagga/software-security-baseline.git --skill software-security-baseline --agent codex --agent claude-code --agent cursor
```

To inspect the repository before installing:

```bash
npx skills add mohammedashrafdagga/software-security-baseline --list
```

To update an installed copy:

```bash
npx skills update software-security-baseline
```

## Use

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

## Repository layout

```text
software-security-baseline/
├── README.md
└── software-security-baseline/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── references/
        ├── minimum-baseline.md
        └── report-format.md
```

The skill follows the Agent Skills `SKILL.md` format. `agents/openai.yaml` adds Codex-specific interface metadata and is optional for Claude Code and Cursor.
