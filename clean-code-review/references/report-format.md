# Clean-Code Review Report Format

Create a new self-contained report on every review. A developer should be able to prioritize, implement, and verify the work without relying on the conversation.

## Tracking rules

- Read earlier numbered reports before assigning finding IDs.
- Assign new root causes the next unused stable ID: `CCR-001`, `CCR-002`, and so on.
- Reuse an ID only when the same root cause remains or recurs. Never reuse an ID for an unrelated problem.
- Include all current `Open` and `Needs verification` items, not only findings discovered in this run.
- List an earlier issue as resolved only when current evidence proves the root cause is gone. A rename, partial cleanup, or accepted debt is not resolution.
- Never edit an older report to make history appear current.

## Required structure

```markdown
# Clean-Code Review — Report NNN

- Date: YYYY-MM-DD
- Project: <name and reviewed root>
- Revision: <commit or working-tree state>
- Review scope: <applications, services, packages, and layers>
- Architecture expectation: <documented architecture and evidence source>
- Compared with: <latest prior report or None>
- Outcome: Meets clean-code baseline | Does not meet clean-code baseline | Not ready to approve

## Executive summary

<Most important conclusion, issue counts by severity/status, and material limitations.>

## Current findings

### CCR-001 — <specific root-cause title>

- Status: Open | Needs verification | Accepted debt
- Severity: Critical | High | Medium | Low
- Principle: <baseline area and project rule>
- Affected surfaces: <features, modules, layers, or packages>
- Evidence: `<path:line>` plus concise observation
- Impact: <credible correctness, change-cost, testability, or maintenance consequence>
- Root cause: <why the issue exists across the affected surfaces>
- Remediation: <smallest durable change and important sequencing>
- Verification: <test, static rule, review, or measurable observation>
- Owner: <team/person or Unassigned>
- Target: <milestone/date or Unscheduled>
- Acceptance: <owner, rationale, guardrails, and revisit date; omit unless accepted>

## Resolved since previous report

<Finding ID, resolution, and current verification evidence; or None.>

## Baseline coverage

| Area | State | Reviewed surfaces | Evidence or gap |
| --- | --- | --- | --- |
| Architecture conformance and boundaries | Pass / Fail / Not verified / Not applicable | <scope> | <evidence> |
| Separation of concerns and cohesion | ... | ... | ... |
| DRY and duplication | ... | ... | ... |
| SOLID and dependency design | ... | ... | ... |
| Simplicity: KISS and YAGNI | ... | ... | ... |
| Readability and local design | ... | ... | ... |
| Domain rules, data, and type integrity | ... | ... | ... |
| Errors, resources, and asynchronous behavior | ... | ... | ... |
| Testability and test design | ... | ... | ... |
| Change safety and repository hygiene | ... | ... | ... |
| Frontend quality | ... | ... | ... |
| Backend quality | ... | ... | ... |
| DDD and feature separation | ... | ... | ... |

## Review ledger

| Surface | Kind | Review depth | Result | Notes |
| --- | --- | --- | --- | --- |
| <application/package/feature/layer> | Frontend / Backend / Shared / Tooling | Complete / Targeted / Not reviewed | Pass / Fail / Not verified | <evidence or gap> |

## Remediation plan

### Fix first

<Ordered Critical and High items and prerequisites, or None.>

### Fix next

<Ordered material maintainability work, or None.>

### Consider later

<Low-priority findings and clearly labeled optional improvements, or None.>

## Verification record

- Checks run: <commands, static reviews, and results>
- Checks not run: <check, reason, and how to run it>
- Assumptions: <material assumptions>
- Exclusions: <generated/vendor code and explicit scope exclusions>
- Coverage gaps: <unreviewed or insufficiently verified surfaces>
- Residual risk: <what the review cannot establish>
```

## Content requirements

- Use repository-relative evidence paths so reports remain portable.
- Cite exact lines for findings whenever the format supports stable line references. Include all known locations for a duplicated rule or boundary violation.
- Separate confirmed failures from evidence gaps: use `Open` for a demonstrated violation and `Needs verification` when evidence is insufficient.
- Explain architecture findings in terms of the project's actual dependency or ownership rule.
- Make remediation specific and testable. “Follow SOLID,” “remove duplication,” or “clean this file” is insufficient.
- Prefer one root-cause finding over many symptom-level findings, while preserving the affected-surface list.
- Put non-blocking suggestions with no baseline failure under `Consider later`; do not inflate issue counts with taste-based recommendations.
- A clean report still includes the full coverage and review-ledger tables, verification record, and `None` for findings and remediation sections.
