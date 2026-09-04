# API Report Format

Every run produces a self-contained snapshot of all discovered operations in scope, including clean runs and incomplete reviews. Keep older reports unchanged. Use repository-relative evidence paths and links so reports remain portable.

## Identity and history

- Use the stable operation key from the skill (service + method/operation + effective route, disambiguated by host/version when needed). Display each HTTP method separately. For GraphQL/RPC/message APIs, display each enumerated operation separately.
- Assign findings stable IDs `API-001`, `API-002`, and so on, using the next unused integer from all previous reports. Reuse the ID for the same root cause, including recurrence; unrelated findings get new IDs.
- Include every current unresolved issue and evidence gap, even if already reported. Consolidate a shared root cause but list every affected endpoint key.
- Track findings as `Open`, `Needs verification`, `Resolved`, or `Risk accepted`. Mark resolution only with evidence; lack of reinspection is not resolution. Accepted risks retain the underlying failed/unverified principle state.
- List new, removed, and changed operations since the last comparable scope. Removal needs route/configuration evidence; absence from a search alone is inconclusive. If scopes differ, explain the limits of comparison and do not imply excluded endpoints were removed or findings fixed.

## Required report structure

Populate the following structure. Never leave template placeholders in a saved report. For empty sections write `None` with an explanation where relevant.

```markdown
# API Validation — Report NNN

- Reviewed at: <ISO 8601 timestamp with timezone>
- Project: <name and root>
- Revision: <commit/branch if available, and whether working tree changes were reviewed>
- Scope: <all services or explicitly scoped subset, protocols, environments/configurations>
- Compared with: <previous report link or None; note any scope differences>
- Outcome: <Meets API baseline / Does not meet API baseline / Not ready to approve / No API endpoints found>
- Inventory completeness: <Complete / Incomplete, with evidence or gaps>

## Summary

- Discovered operations: <N>
- Endpoint results: <P Pass, F Fail, U Not verified; P + F + U = N>
- Findings: <confirmed counts by severity, evidence-gap count separately>
- Principal blockers: <finding IDs and concise impacts, or None>

## Discovery and scope evidence

| Service / surface | Discovery sources | Reconciliation / coverage gaps |
| --- | --- | --- |
| <service, host, protocol> | <routes, schemas, gateway config, runtime listing when available> | <agreement, undocumented/documented-only operations, unavailable sources> |

<Assumptions, configuration-dependent registrations, exclusions and unavailable services. Explain how inventory completeness was established.>

## Full endpoint inventory

| Endpoint key | Purpose / exposure | Handler and contract evidence | Overall state | Finding IDs |
| --- | --- | --- | --- | --- |
| <service METHOD /effective/path> | <purpose; public/protected/internal/admin> | <path:line; contract reference or gap> | <Pass / Fail / Not verified> | <IDs or None> |

## Per-endpoint principle results

| Endpoint key | P01 | P02 | P03 | P04 | P05 | P06 | P07 | P08 | P09 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| <same key as inventory> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> | <state; evidence ID> |

<Use Pass, Fail, Not verified, or Not applicable. Include reasons for Not applicable and evidence/gap details below. Split into service tables if needed; retain every row.>

## Evidence and system-level checks

| Evidence ID / principle | Affected endpoint keys | Evidence and verification type | Result / gap / applicability reason |
| --- | --- | --- | --- |
| <E001 / P03> | <explicit keys> | <path:line; static trace or executed test name> | <observation, result, or missing evidence> |

<Record any system-level findings such as incomplete inventory or contract-only operations here as well as in current issues; they must affect the overall decision even when no implemented endpoint can be assigned.>

## Current issues

### API-001 — <specific defect or evidence gap>

- Status: <Open / Needs verification / Risk accepted>
- Severity: <Critical / High / Medium / Low; for unconfirmed gaps use Not assessed unless potential impact can be justified>
- Principle: <P01–P09>
- Affected endpoints: <all known stable keys, or system-level scope>
- Evidence: <repository-relative path:line and observed behavior; redacted local reproduction if available>
- Expected versus observed: <specific requirement and deviation, or evidence needed>
- Impact: <credible client, security, reliability, or data-integrity consequence; qualify unconfirmed risks>
- Remediation: <concrete action>
- Verification: <test or evidence that proves resolution>
- Owner / target: <known values or Unassigned / Unscheduled>
- Risk acceptance: <only if already accepted; owner, rationale, scope, expiry>

## Changes since previous report

- Added / changed / removed operations: <keys and evidence, or None>
- Resolved findings: <IDs and verification evidence, or None>
- Still open / recurring / not reverified: <IDs and current status, or None>

## Verification record

| Check / command | Environment and scope | Result | Limitations |
| --- | --- | --- | --- |
| <actual test command or static review> | <local fixtures, services and operations> | <passed / failed / not run; relevant counts> | <missing credentials, runtime, external dependency, or None> |

## Next actions

<Prioritized confirmed fixes and evidence-gathering tasks, with finding IDs. For a clean run state that no baseline remediation was identified within the reviewed scope. Separate optional recommendations from required fixes.>
```

## Final consistency check

Confirm inventory and matrix have the same endpoint keys, every principle cell has supporting evidence or a gap/reason, totals reconcile, and shared evidence actually applies to every linked route. Endpoint success totals cannot override failed system-level checks. If no operations were found, include zero totals and discovery evidence; do not manufacture endpoint rows. If discovery or testing is partial, preserve that limitation in the summary and outcome.
