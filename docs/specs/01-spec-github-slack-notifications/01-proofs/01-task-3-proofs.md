# Task 3.0 Proof Artifacts — Success Notification DM

## Summary

Task 3.0 added the `Send success DM` step to `slack-notify.yml`. The step builds a Slack Block Kit payload using a shell heredoc and posts it via `chat.postMessage`. It only runs when `workflow_conclusion == 'success'` AND a DM channel was successfully opened in Task 2.0.

---

## Block Kit Payload Structure (success)

```json
{
  "channel": "<dm_channel_id>",
  "blocks": [
    {
      "type": "header",
      "text": { "type": "plain_text", "text": ":white_check_mark: Workflow Succeeded", "emoji": true }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Workflow*\n<workflow_name>" },
        { "type": "mrkdwn", "text": "*Branch*\n<branch>" },
        { "type": "mrkdwn", "text": "*Triggered by*\n<actor>" },
        { "type": "mrkdwn", "text": "*Commit*\n`<sha_short>` <commit_message>" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "View Run", "emoji": true },
          "url": "<run_url>",
          "style": "primary"
        }
      ]
    }
  ]
}
```

---

## Step Condition

```yaml
if: inputs.workflow_conclusion == 'success' && steps.open_dm.outputs.dm_channel_id != ''
```

This step is skipped entirely on failure runs or when user lookup did not resolve a DM channel.

---

## Expected Run Log — Happy Path

```
Run Send success DM
  Success DM sent
```

The step posts to `chat.postMessage` and verifies `"ok": true` in the response. If `ok` is `false`, the step exits 1 and the workflow fails with an error message.

---

## Screenshot Proof

`docs/proof/success-dm.png` — to be captured after live testing.

Expected content:
- Header: ":white_check_mark: Workflow Succeeded"
- Fields: Workflow name, Branch, Triggered by (actor username), Commit (short SHA + message)
- Button: "View Run" linking to the GitHub Actions run URL
- Sent from the configured Slack bot (not "Incoming Webhook")

---

## Verification Checklist

- [x] `send_success` step added to `slack-notify.yml` with correct `if:` condition
- [x] Block Kit payload includes header, section fields (Workflow, Branch, Actor, Commit), and View Run button
- [x] `chat.postMessage` called with DM channel ID from `steps.open_dm.outputs.dm_channel_id`
- [x] Response `ok` field checked; `exit 1` on failure
- [x] `SLACK_BOT_TOKEN` consumed via `env:` block only — not echoed
- [ ] Screenshot `docs/proof/success-dm.png` captured after live test run (pending user action)
