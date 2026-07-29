# automation-metrics

Orphan branch that holds the metrics store for the Devin scan-and-remediate
automation. It shares no history with `master` and contains no application
code — only `metrics/remediation-metrics.json` and this README — so appending a
run never touches the source tree and never shows up in a code diff.

## Files

- `metrics/remediation-metrics.json` — the store. `schema_version` plus two
  top-level arrays, `runs` and `sessions`, seeded empty.

## Who writes it

The **runtime scan session** appends to `metrics/remediation-metrics.json`
(step 6 of its prompt). That session is created by
`.github/workflows/devin-scan-orchestrator.yml` on `master` — a manual
`workflow_dispatch` that makes a single Devin API call and does nothing else;
no scanning and no fan-out happens in CI. The orchestrator session:

1. appends exactly one object to `runs` per orchestrator run, and
2. appends one object to `sessions` per remediation child session it launched,
   filled in from that child's structured output (`pr_url`, `scan_before`,
   `scan_after`, `status`) plus the CI status of its PR.

Appends are commits to this branch, landed via a PR titled
`chore(metrics): record run <run_id>`. **Append only** — existing records are
never rewritten or deleted. If two runs race, the writer re-reads, re-appends,
and retries rather than force-pushing. A no-op run (the floor rule found
nothing at all) still gets a `runs` record with `status: "no_findings"` and
every counter at zero, and no `sessions` records.

## Who reads it

The **dashboard** reads `metrics/remediation-metrics.json` from this branch
(raw content at `refs/heads/automation-metrics`). It is read-only for the
dashboard — the dashboard never writes back.

## Schema

### `runs[]`

| field                 | type            | notes                                                                              |
| --------------------- | --------------- | ---------------------------------------------------------------------------------- |
| `run_id`              | string          | `run-<UTC ISO8601 compact>-<6 hex>`. Unique per run; foreign key for `sessions`.   |
| `trigger_source`      | string          | `"scheduled"` \| `"manual"`.                                                        |
| `started_at`          | string          | ISO 8601 UTC.                                                                       |
| `finished_at`         | string \| null  | ISO 8601 UTC; `null` while the run is in flight.                                    |
| `status`              | string          | `"running"` \| `"success"` \| `"partial"` \| `"failed"` \| `"no_findings"`.         |
| `findings_discovered` | integer         | Size of the severity-ranked top-5 selection after filtering; `0` for a no-op run.  |
| `issues_filed`        | integer         | New issues only; duplicates skipped by `stable_finding_key` are excluded.          |
| `sessions_launched`   | integer         | Equals `issues_filed` unless the concurrency cap left some queued and unlaunched.  |
| `prs_opened`          | integer         | Children reporting a non-null `pr_url`.                                             |

### `sessions[]`

| field          | type            | notes                                                                                          |
| -------------- | --------------- | ----------------------------------------------------------------------------------------------- |
| `session_id`   | string          | Devin session id of the remediation child.                                                      |
| `run_id`       | string          | Parent run.                                                                                      |
| `issue_number` | integer         | The `devin-fix` issue the child was assigned.                                                    |
| `status`       | string          | `"success"` \| `"failed"` \| `"skipped_duplicate"` \| `"not_reproducible"` \| `"running"`.        |
| `pr_url`       | string \| null  | `null` when no PR was opened.                                                                    |
| `scan_before`  | string          | Trimmed verbatim scan output for the finding before the fix.                                     |
| `scan_after`   | string          | Same after the fix; must not contain the finding's `acceptance_check` line.                      |
| `ci_status`    | string          | `"passing"` \| `"failing"` \| `"pending"` \| `"unknown"` at the time the record was written.      |
