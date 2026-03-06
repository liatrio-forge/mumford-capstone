# mumford-capstone — Shared Slack Notification Workflow

A centralized GitHub Actions reusable workflow that sends Slack direct message notifications to the user who triggered a workflow whenever it completes. Any repository in the `liatrio-forge` organization can adopt this in about 5 minutes by copying one file and adding one secret.

---

## How It Works

```
Your repo                           mumford-capstone
─────────────────────────────────   ───────────────────────────────
workflow_run event fires
  └─ notify-on-completion.yml  ──►  .github/workflows/slack-notify.yml
       (caller — you copy this)           │
                                          ├─ Looks up GitHub actor email
                                          ├─ Resolves Slack user by email
                                          ├─ Opens a DM channel
                                          ├─ On success: sends summary DM
                                          └─ On failure: fetches job logs,
                                             extracts error snippet, sends
                                             detailed failure DM
```

On **success**, the DM includes: workflow name, branch, commit, and a View Run link.

On **failure**, the DM includes: failed job name, failed step name, the last 20 lines of the error output, branch, commit, and a View Run link.

---

## Prerequisites

### 1. Create a Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App** → **From scratch**
2. Give it a name (e.g. `CI Notifier`) and select your workspace
3. Go to **OAuth & Permissions** → **Scopes** → **Bot Token Scopes** and add:
   - `users:read`
   - `users:read.email`
   - `im:write`
   - `chat:write`
4. Click **Install to Workspace** and authorize
5. Copy the **Bot User OAuth Token** (starts with `xoxb-`)

### 2. Enable Cross-Repo Reusable Workflows (org admins only)

For repos in `liatrio-forge` to call this reusable workflow, the org must allow it:

1. Go to `github.com/organizations/liatrio-forge/settings/actions`
2. Under **Policies**, ensure **Allow all actions and reusable workflows** is selected
   (or add `liatrio-forge/mumford-capstone` to the allowlist if using a restrictive policy)

---

## Onboarding a New Repo (3 Steps)

### Step 1 — Copy the caller workflow

Copy [`docs/caller-template/workflow-notify-caller.yml`](docs/caller-template/workflow-notify-caller.yml) into your repo at:

```
.github/workflows/notify-on-completion.yml
```

Edit the `workflows:` list to include the **display names** of the workflows you want notifications for:

```yaml
on:
  workflow_run:
    workflows:
      - "CI"        # replace with your workflow's name: field
      - "Deploy"
    types:
      - completed
```

> **Note:** GitHub Actions does not support wildcards in `workflow_run.workflows`. You must list each workflow name explicitly.

### Step 2 — Add the Slack bot token as a secret

In your repo, go to **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
|---|---|
| `SLACK_BOT_TOKEN` | Your Slack bot token (`xoxb-...`) |

### Step 3 — Push

Commit and push the caller workflow file. The next time any listed workflow completes, you will receive a Slack DM.

---

## Security Notes

- `SLACK_BOT_TOKEN` is never logged or echoed — it is consumed only via GitHub Actions `env:` blocks
- **Log snippet warning:** The error output included in failure DMs is extracted directly from your job logs. If your workflows print environment variable values or other sensitive data to stdout, those values may appear in the DM. Avoid printing secrets in workflow steps
- `GITHUB_TOKEN` is scoped to `actions: read` and `contents: read` within the notification workflow
- Proof artifact screenshots in `docs/proof/` do not contain token values

---

## Repository Structure

```
.github/
  workflows/
    slack-notify.yml            # Reusable workflow (the shared implementation)
    test-success.yml            # Test workflow — always succeeds (workflow_dispatch + push to main)
    test-failure.yml            # Test workflow — always fails (workflow_dispatch)
    test-caller.yml             # Internal caller for test workflows
docs/
  caller-template/
    workflow-notify-caller.yml  # Template for consuming repos to copy
  proof/                        # Screenshots from live test runs
  specs/                        # SDD specification and task tracking
```

---

## Triggering Test Runs

To test the notification workflows in this repo:

1. Add `SLACK_BOT_TOKEN` as a secret in this repo's settings
2. Go to **Actions → Test Success → Run workflow** to test a success DM
3. Go to **Actions → Test Failure → Run workflow** to test a failure DM
