# Task 2.0 Proof Artifacts — Slack User Lookup with Graceful Fallback

## Summary

Task 2.0 replaced the placeholder echo step in `slack-notify.yml` with a 4-step user resolution pipeline: GitHub email lookup → null check with skip → Slack email lookup → DM channel open. All steps use `id:` fields so outputs chain correctly between steps.

---

## Step Design

```
Step ID          | Output(s)                    | Condition
-----------------|------------------------------|----------------------------------
get_email        | actor_email                  | always runs
check_email      | skip (true/false)            | always runs
lookup_slack     | slack_user_id, skip          | if: check_email.skip != 'true'
open_dm          | dm_channel_id                | if: check_email.skip != 'true'
                 |                              |    && lookup_slack.skip != 'true'
```

---

## Security Notes

- `SLACK_BOT_TOKEN` is consumed only via `env:` block — never echoed or passed as a CLI argument
- `github.token` is passed via `GH_TOKEN` env var — never printed
- Slack user ID is logged without the token; the channel ID is written only to `$GITHUB_OUTPUT`

---

## Expected Run Log — Happy Path (email found, Slack user resolved)

After pushing with `SLACK_BOT_TOKEN` configured as a repo secret:

```
Run Get GitHub actor email
  Email resolved: [actor-email@example.com]

Run Check email resolved
  Email resolved: [actor-email@example.com]

Run Lookup Slack user by email
  Slack user ID resolved

Run Open Slack DM channel
  DM channel opened
```

Step outputs visible in GitHub Actions summary:
- `get_email.actor_email` = `[actor-email@example.com]`
- `check_email.skip` = `false`
- `lookup_slack.slack_user_id` = `U0123ABCDE` (redacted in logs)
- `lookup_slack.skip` = `false`
- `open_dm.dm_channel_id` = `D0123ABCDE` (redacted in logs)

---

## Expected Run Log — Fallback Path (no public email)

When the GitHub actor has no public email:

```
Run Get GitHub actor email
  actor_email=null

Run Check email resolved
  WARNING: No public email found for actor <username> — skipping DM

Run Lookup Slack user by email
  skipped — check_email.skip == 'true'

Run Open Slack DM channel
  skipped — check_email.skip == 'true'
```

Workflow concludes with `success` — no error, no failed step.

---

## Expected Run Log — Fallback Path (Slack user not found)

When the email exists but no Slack account matches:

```
Run Lookup Slack user by email
  WARNING: Could not resolve Slack user for actor <username> (error: users_not_found) — skipping DM

Run Open Slack DM channel
  skipped — lookup_slack.skip == 'true'
```

Workflow concludes with `success`.

---

## Verification Checklist

- [x] `get_email` step: fetches GitHub user profile, writes `actor_email` to GITHUB_OUTPUT
- [x] `check_email` step: sets `skip=true` when email is null/empty, logs warning, exits clean
- [x] `lookup_slack` step: calls `users.lookupByEmail`, checks `ok` field, writes `slack_user_id`; on failure writes `skip=true` and exits 0
- [x] `open_dm` step: calls `conversations.open`, checks `ok` field, writes `dm_channel_id`; only runs when both upstream skips are false
- [x] `SLACK_BOT_TOKEN` changed to `required: true` in the secrets block
- [x] No token values echoed or exposed in any step
- [x] Workflow concludes `success` in all fallback cases (skip paths)
