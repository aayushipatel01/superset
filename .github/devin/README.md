# .github/devin

Prompt text for the Devin automations that run against this fork.

## `scan-orchestrator-prompt.md`

The full master scan-orchestrator prompt, manual variant (`trigger_source =
"manual"` hardcoded in the text). It is passed **verbatim** as the `prompt`
field of the single Devin API call made by
`../workflows/devin-scan-orchestrator.yml`:

```
POST https://api.devin.ai/v3/organizations/{org_id}/sessions
```

It is deliberately *not* referenced as a `playbook_id`. Embedding the text
means the exact instructions a given run executed are recoverable from this
repo at that commit, and editing the prompt goes through code review like any
other change.

The file is the prompt body, nothing else — no front matter, no HTML comments.
Anything added here is read by the orchestrator as an instruction. The workflow
appends one machine-generated "Run context" section (run id, run url,
`trigger_source`, target repo, `max_parallel`, and a note that
`DEVIN_API_KEY` / `DEVIN_ORG_ID` are available as session secrets), so per-run
values must not be hardcoded here.

A near-identical copy with `trigger_source` hardcoded to `"scheduled"` backs
the schedule-triggered Automation; the two are maintained in parallel.
