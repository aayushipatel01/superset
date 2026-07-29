# automation-metrics

Orphan branch that holds the metrics store for the Devin scan-and-remediate
automation. It shares no history with `master` and contains no application
code — only `metrics.json` and this README — so appending a run never touches
the source tree and never shows up in a code diff.

## Files

- `metrics.json` — the store. Two top-level arrays, `runs` and `sessions`,
  seeded empty.

## Who writes it

The **runtime scan session** appends to `metrics.json`. That session is created
by `.github/workflows/devin-scan-orchestrator.yml` on `master` (a manual
`workflow_dispatch`), which makes a single Devin API call and does nothing
else — no scanning and no fan-out happens in CI. The orchestrator session:

1. appends exactly one object to `runs` per orchestrator run, and
2. appends one object to `sessions` per remediation child session it launches,
   updating that object in place as the child progresses (PR opened, CI result).

Appends are commits to this branch. Because the workflow serializes
orchestrator runs on a `devin-scan-orchestrator` concurrency group, concurrent
writers are not expected; a writer should still re-read the file immediately
before committing rather than caching it across a long run.

## Who reads it

The **dashboard** reads `metrics.json` from this branch (raw content at
`refs/heads/automation-metrics`). It is read-only for the dashboard — the
dashboard never writes back.

## Schema

### `runs[]`

| field                 | type            | notes                                                             |
| --------------------- | --------------- | ----------------------------------------------------------------- |
| `run_id`              | string          | The GitHub Actions run id of the dispatching workflow run.        |
| `trigger_source`      | string          | `"manual"` for `workflow_dispatch`; `"scheduled"` for cron.       |
| `started_at`          | string          | ISO 8601 UTC, when the orchestrator session began work.           |
| `finished_at`         | string \| null  | ISO 8601 UTC; `null` while the run is still in flight.            |
| `status`              | string          | e.g. `running`, `completed`, `no_findings`, `failed`.             |
| `findings_discovered` | integer         | Total findings pooled across all scanners, before filtering.      |
| `issues_filed`        | integer         | Issues actually opened (duplicates skipped are not counted).      |
| `sessions_launched`   | integer         | Remediation child sessions created for this run.                  |
| `prs_opened`          | integer         | PRs opened by those child sessions.                               |

### `sessions[]`

| field           | type            | notes                                                              |
| --------------- | --------------- | ------------------------------------------------------------------ |
| `session_id`    | string          | Devin session id of the remediation child session.                 |
| `run_id`        | string          | Foreign key onto `runs[].run_id`.                                  |
| `issue_number`  | integer \| null | The fork issue the child session is fixing.                        |
| `status`        | string          | e.g. `running`, `completed`, `failed`, `skipped_duplicate`.        |
| `pr_url`        | string \| null  | PR the child session opened; `null` until it opens one.            |
| `scan_before`   | object \| null  | Scanner state for the finding before the fix.                      |
| `scan_after`    | object \| null  | Scanner state after the fix, for verifying the finding is gone.    |
| `ci_status`     | string \| null  | CI conclusion on the child session's PR, e.g. `success`, `failure`. |

A run that finds nothing writes a `runs` record with `status: "no_findings"`
and every counter at zero, and adds no `sessions` records.
