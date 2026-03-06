# Task 4.0 Proof Artifacts — Failure Notification DM with Log Extraction

## Summary

Task 4.0 added the full failure pipeline to `slack-notify.yml`: fetching job/step metadata and raw logs from the GitHub API, extracting and truncating the error snippet, and sending a rich Block Kit failure DM. A `test-failure.yml` workflow and `test-caller.yml` update enable end-to-end testing.

---

## New Files

- `.github/workflows/test-failure.yml` — triggers via `workflow_dispatch`; single job that runs `exit 1`
- `.github/workflows/test-caller.yml` — updated `workflow_run.workflows` list to include `"Test Failure"`

---

## New Steps Added to slack-notify.yml

```
Step ID          | Condition                           | Output(s)
-----------------|-------------------------------------|------------------------------
get_job          | conclusion == 'failure'             | failed_job_id, failed_job_name
get_step         | conclusion == 'failure'             | failed_step_name
fetch_logs       | conclusion == 'failure'             | writes /tmp/job.log
extract_snippet  | conclusion == 'failure'             | error_snippet (multiline)
send_failure     | conclusion == 'failure'             | —
                 | && dm_channel_id != ''              |
```

---

## Log Extraction Design

1. Strip ANSI codes: `sed 's/\x1b\[[0-9;]*m//g' /tmp/job.log > /tmp/job_clean.log`
2. Find the failed step's log section by matching `##[group]Run <step_name>`
3. Take the last 20 lines; truncate to 3000 characters with `head -c 3000`
4. Fallback: if the step header is not found, use the last 20 lines of the whole log
5. Write using `<<SNIPPET_EOF` heredoc syntax to safely handle multiline content in `$GITHUB_OUTPUT`

---

## Block Kit Payload Structure (failure)

```json
{
  "channel": "<dm_channel_id>",
  "blocks": [
    { "type": "header", "text": { "text": ":x: Workflow Failed" } },
    {
      "type": "section",
      "fields": [
        "*Workflow*\n<workflow_name>",
        "*Branch*\n<branch>",
        "*Failed Job*\n<job_name>",
        "*Failed Step*\n<step_name>",
        "*Commit*\n`<sha_short> <commit_message>`"
      ]
    },
    {
      "type": "section",
      "text": { "type": "mrkdwn", "text": "*Error Output*\n```<error_snippet>```" }
    },
    {
      "type": "actions",
      "elements": [{ "type": "button", "text": "View Run", "style": "danger", "url": "<run_url>" }]
    }
  ]
}
```

Payload is built with `jq -n --arg ...` to safely handle special characters in all values.

---

## Expected Run Log — Failure Path

```
Run Get failed job details
  Failed job: fail (ID: 12345678)

Run Get failed step name
  Failed step: Always fail

Run Fetch failed job logs
  Log fetched (47 lines)

Run Extract error snippet
  Extracted 3 lines for error snippet

Run Send failure DM
  Failure DM sent
```

---

## Screenshot Proof

`docs/proof/failure-dm.png` — to be captured after live testing.

Expected content:
- Header: ":x: Workflow Failed"
- Fields: Workflow name, Branch, Failed Job, Failed Step, Commit (short SHA + message)
- Code block: last 20 lines of the `Always fail` step log (showing the `exit 1` error)
- Button: "View Run" (danger/red style) linking to the GitHub Actions run
- Sent from the configured Slack bot

---

## Verification Checklist

- [x] `test-failure.yml` created with `workflow_dispatch` trigger and `exit 1` step
- [x] `test-caller.yml` updated to include `"Test Failure"` in `workflow_run.workflows` list
- [x] `get_job` step: fetches jobs API, finds first failed job, writes `failed_job_id` + `failed_job_name`
- [x] `get_step` step: reads `/tmp/failed_job.json`, writes `failed_step_name`
- [x] `fetch_logs` step: fetches job logs to `/tmp/job.log`
- [x] `extract_snippet` step: strips ANSI, extracts step section, truncates to 3000 chars, writes multiline output safely
- [x] `send_failure` step: uses `jq -n` for JSON safety, posts Block Kit payload, verifies `ok: true`
- [x] All failure steps guarded with `if: inputs.workflow_conclusion == 'failure'`
- [ ] Screenshot `docs/proof/failure-dm.png` captured after live test run (pending user action)
