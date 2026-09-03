# Security Report Format

Create a report only when the current review contains at least one `Fail` or `Not verified` control. The report is a self-contained, point-in-time snapshot that a developer can use to review, fix, and verify issues without relying on the conversation.

## Tracking rules

- Read earlier numbered reports before assigning IDs.
- Give each new root cause the next unused stable ID in the form `SEC-001`, `SEC-002`, and so on.
- Reuse the same ID when a previous issue is still present or has recurred from the same root cause. Do not reuse an ID for an unrelated issue.
- Include all current `Open` and `Needs verification` items, not only newly discovered ones.
- When an earlier issue is now demonstrably fixed and the current report is being created for other remaining issues, list it under `Resolved since previous report` with the verification evidence.
- Do not edit an older numbered report merely to make history look current. A risk acceptance is not a fix.

## Required structure

Use this structure, adapting labels only when the application needs clearer terminology:

```markdown
# Software Security Review — Report NNN

- Date: YYYY-MM-DD
- Application: <name and reviewed root>
- Review scope: <components and environments>
- Compared with: <latest prior report or None>
- Outcome: Does not meet minimum baseline | Not ready to approve
- Release decision: Block | Conditional | Not assessed

## Executive summary

<Short statement of the most important risk, issue counts, and limitations.>

## Release blockers

<Critical and High issues, or None.>

## Current issues

### SEC-001 — <specific title>

- Status: Open | Needs verification | Risk accepted
- Severity: Critical | High | Medium | Low
- Baseline area: <area>
- Affected component: <component, endpoint, role, tenant, or data>
- Evidence: `<path:line>` and/or reproducible observation
- Impact: <credible abuse and business effect>
- Remediation: <smallest durable fix>
- Verification: <test or evidence that will prove resolution>
- Owner: <name/team or Unassigned>
- Target: <date/milestone or Unscheduled>
- Acceptance: <owner, rationale, compensating controls, expiry; omit unless accepted>

## Resolved since previous report

<Issue ID, resolution, and verification evidence; or None.>

## Baseline coverage

| Area | State | Evidence or gap |
| --- | --- | --- |
| <area> | Pass / Fail / Not verified / Not applicable | <concise evidence> |

## Remediation order

### Before release

<Ordered actions or None.>

### After release

<Ordered defense-in-depth actions or None.>

## Verification record

- Checks run: <commands/tests/reviews and results>
- Checks not run: <what and why>
- Assumptions: <material assumptions>
- Exclusions: <important exclusions>
- Residual risk: <risk remaining after stated controls>
```

## Content requirements

- Use repository-relative evidence paths inside the report so it remains portable with the application.
- Never include live secret values, full tokens, unnecessary personal data, or exploit instructions against real systems.
- Distinguish confirmed findings from missing evidence. Use `Open` for confirmed issues and `Needs verification` for unresolved evidence gaps.
- Keep remediation testable. “Improve security,” “sanitize inputs,” or “use best practices” is not sufficient.
- If a finding appears in several locations, list the known affected surfaces under one root-cause issue.
- If there is no previous report, omit historical comparison beyond stating `None`.
