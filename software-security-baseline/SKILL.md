---
name: software-security-baseline
description: Review or harden a software application against the minimum practical security baseline for a small business. Use for pre-release security checks, security-focused code reviews, minimum-control audits, or remediation of essential application safeguards; do not use as a substitute for a penetration test, compliance assessment, or legal advice.
metadata:
  author: "mohammedashrafdagga"
  version: "1.0.0"
  supported-agents: "codex,claude-code,cursor"
---

# Software Security Baseline

Assess whether an application has a defensible minimum level of security for its actual exposure and data. Prioritize exploitable weaknesses and practical remediation over checklist volume.

## Establish scope

Before judging controls, determine from the repository and available context:

- application type, deployment model, environments, and internet exposure;
- users, roles, trust boundaries, administrative paths, and tenant boundaries;
- sensitive data, credentials, payments, regulated data, and retention needs;
- APIs, third-party integrations, webhooks, uploads, background jobs, and outbound requests;
- build, deployment, hosting, database, backups, logging, and update ownership.

Ask only for material facts that cannot be discovered safely. If facts remain unavailable, state assumptions and mark affected controls `Not verified`.

## Review workflow

1. Identify the application root. In a single-application repository, use the repository root. In a monorepo, use the scoped application's root unless the user requested a repository-wide review. Never place application reports inside this skill's directory.
2. Ensure `<application-root>/security/` exists, preserving any existing content. Do not add a placeholder solely to make an empty directory visible to version control.
3. Inspect existing `security/report_*.md` files to understand prior findings and retain stable issue IDs where the same root cause remains.
4. Inspect architecture, authentication and authorization flows, data paths, configuration, dependencies, deployment files, and tests. Follow repository instructions and use ecosystem-native security checks when available.
5. Read [references/minimum-baseline.md](references/minimum-baseline.md). Assess every core area and each applicable conditional area.
6. Prefer direct evidence: code paths, configuration values with secrets redacted, tests, lockfiles, CI rules, infrastructure definitions, and operational documentation. Do not award a pass from filenames, dependencies, comments, or stated intent alone.
7. Trace security boundaries end to end. For example, confirm that authorization is enforced by the trusted backend on every relevant operation, not only hidden in the UI.
8. When the user requests a review, do not change application code; creating the requested `security/` directory and issue report is allowed. When the user requests hardening or fixes, make the smallest cohesive changes, preserve existing behavior where safe, and add or run proportionate tests.
9. Re-run relevant checks after fixes and report remaining risk and unverified operational controls.

Use these control states:

- `Pass`: implementation and relevant verification evidence are present.
- `Fail`: absent, ineffective, or bypassable control.
- `Not verified`: evidence is missing or cannot be tested in the current environment.
- `Not applicable`: the exposure or feature does not exist; include the reason.

## Severity and release decision

Rate findings by realistic impact and exploitability, considering internet exposure, privileges, data sensitivity, tenant reach, and compensating controls:

- `Critical`: likely direct compromise, mass sensitive-data exposure, authentication bypass, arbitrary code execution, or equivalent business-threatening impact.
- `High`: plausible account takeover, privilege or tenant-boundary bypass, significant data exposure, or material supply-chain/deployment compromise.
- `Medium`: meaningful weakness that needs pre-release treatment when exposed, but requires stronger preconditions or has limited impact.
- `Low`: defense-in-depth, limited exposure, or hygiene issue with credible security value.

The minimum release gate fails when any unresolved Critical or High finding exists. It also remains `Not ready to approve` when a release-critical control cannot be verified. A risk acceptance must name the owner, rationale, compensating controls, and expiration; do not silently convert acceptance into a pass.

## Safety boundaries

- Keep assessment activity read-only unless the user requested changes.
- Do not attack production, third-party systems, or real user accounts without explicit authorization and a defined test scope. Prefer local tests and non-destructive validation.
- Never print, copy, or store live secrets or unnecessary personal data. Report secret locations and identifiers in redacted form.
- Do not claim the application is “secure.” State the reviewed scope, evidence, limitations, and date.
- Flag possible legal, privacy, payment, health, or industry obligations for specialist review; do not declare compliance from this baseline.

## Deliverable

Lead with one outcome: `Meets minimum baseline`, `Does not meet minimum baseline`, or `Not ready to approve`.

If at least one control is `Fail` or `Not verified`, read [references/report-format.md](references/report-format.md) and create a new Markdown report in the application's `security/` directory. A `Not verified` control is tracked because it still requires review, evidence, or testing; do not describe it as a confirmed vulnerability.

Name the report `report_NNN.md`:

- Find existing files whose names match `report_` followed by an integer and `.md`.
- Use one greater than the largest existing report number, starting at `report_001.md` and padding to at least three digits.
- Never overwrite, delete, or renumber an earlier report. If the chosen path now exists, recalculate before writing.

Do not create a report when every applicable control is `Pass` or `Not applicable` and there are no actionable findings. Keep the `security/` directory even when no report is created. In the response, explicitly say that no report was created because no issues were found.

If the application root is not writable, do not place the report elsewhere. Return the intended path and report content in the response, and state that persistence is blocked by write access.

The response and any created report must provide:

1. scope, assumptions, and important exclusions;
2. release blockers first;
3. findings with severity, affected component and evidence location, credible impact, specific remediation, and verification method;
4. a compact coverage table for every baseline area using the four control states;
5. prioritized next actions split into `before release` and `after release`;
6. checks run, checks not run, and residual risk.

Avoid vague advice. Cite exact file paths and lines when reviewing code. Consolidate repeated instances that share one root cause, but identify all known affected surfaces. Link the newly created report in the response.

## Standards basis

Use the baseline reference as the operational checklist. Treat OWASP ASVS as the verification-oriented source, OWASP Top 10 as risk awareness rather than complete coverage, NIST SSDF as development-process guidance, and CISA Secure by Design as the basis for safe defaults and vendor ownership of customer security outcomes. Re-check current versions when the user requests a formally current or externally cited assessment.
