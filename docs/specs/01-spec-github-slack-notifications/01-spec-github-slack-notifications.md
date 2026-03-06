# 01-spec-github-slack-notifications

## Introduction/Overview

This feature creates a centralized GitHub Actions reusable workflow, hosted in `mumford-capstone`, that any other repository in the organization can call to receive Slack direct message notifications when their workflows complete. On success, the message confirms completion. On failure, the message includes the specific job and step that failed, relevant error log lines extracted from the GitHub API, a direct link to the failed run, and the branch and commit that triggered it.

The goal is to let developers stay informed about their CI/CD runs without having to monitor GitHub, with a single shared implementation that any repo can adopt by adding one small caller workflow.

## Goals

- Send a Slack DM to the triggering user on every workflow completion (success or failure)
- On failure, surface the failed job name, failed step name, and a truncated error log snippet
- Resolve the Slack user automatically by matching the GitHub actor's email to their Slack account email
- Host the reusable notification workflow centrally in `mumford-capstone` so any org repo can adopt it without duplicating logic
- Allow any consuming repo to integrate by adding a single small caller workflow (~15 lines) with no other code changes
- Store all credentials as encrypted GitHub Actions secrets with no hardcoded values

## User Stories

**As a developer**, I want a Slack DM when my workflow succeeds so that I know it is safe to proceed without checking GitHub.

**As a developer**, I want a Slack DM with failure details when my workflow fails so that I can understand and fix the problem without opening GitHub.

**As a team lead**, I want notifications sent only to the person who triggered the run so that teammates are not interrupted by someone else's workflow results.

**As a platform engineer**, I want to maintain the notification logic in one place (`mumford-capstone`) so that all repos in the org benefit from fixes and improvements automatically.

## Demoable Units of Work

### Unit 1: Centralized Reusable Workflow

**Purpose:** Establishes the shared workflow in `mumford-capstone` and proves it can be called from another repo.

**Functional Requirements:**
- The system shall create a reusable workflow at `.github/workflows/slack-notify.yml` in `mumford-capstone` declared with `on: workflow_call` so other repos can invoke it
- The reusable workflow shall accept the following inputs: `workflow_name`, `workflow_conclusion`, `run_id`, `run_url`, `branch`, `commit_sha`, `commit_message`, `actor` (the GitHub username of the triggering user)
- The system shall create a caller workflow template at `docs/caller-template/workflow-notify-caller.yml` that consuming repos can copy into their own `.github/workflows/` directory
- The caller template shall use `on: workflow_run` to trigger on all workflows in the consuming repo and pass the required inputs to `mumford-capstone`'s reusable workflow via `uses: <org>/mumford-capstone/.github/workflows/slack-notify.yml@main`
- The `SLACK_BOT_TOKEN` secret shall be passed from the consuming repo to the reusable workflow via the `secrets: inherit` directive

**Proof Artifacts:**
- GitHub Actions run log (consuming repo): shows the caller workflow triggering and successfully calling the reusable workflow in `mumford-capstone`
- GitHub Actions run log (`mumford-capstone`): shows the reusable workflow receiving inputs and completing

### Unit 2: Success Notification

**Purpose:** Proves the reusable notification workflow delivers a clear success DM.

**Functional Requirements:**
- The system shall look up the triggering GitHub actor's public email via the GitHub API (`GET /users/{username}`)
- The system shall look up the Slack user ID by calling `users.lookupByEmail` with the GitHub actor's email
- The system shall open a Slack DM channel and send a message containing: workflow name, result (success), branch name, and a link to the run
- The system shall skip sending a DM and log a warning if the Slack user cannot be resolved from the email

**Proof Artifacts:**
- Screenshot: Slack DM showing a success message with workflow name, branch, and run URL demonstrates the notification is delivered
- GitHub Actions run log: shows the `slack-notify` workflow completing successfully after a test workflow run in a consuming repo

### Unit 3: Failure Notification

**Purpose:** Proves that failure details are extracted from GitHub API logs and included in the DM.

**Functional Requirements:**
- The system shall detect when the triggering workflow's conclusion is `failure`
- The system shall call `GET /repos/{owner}/{repo}/actions/runs/{run_id}/jobs` to identify the failed job and failed step
- The system shall call `GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs` to retrieve the job log
- The system shall extract the last 20 lines of the failed step's log output and truncate to 3000 characters if needed
- The system shall send a Slack DM containing: workflow name, failed job name, failed step name, the error snippet, branch name, commit SHA, commit message, and a link to the run

**Proof Artifacts:**
- Screenshot: Slack DM showing a failure message with the failed step name and error snippet after a deliberately failing test workflow
- GitHub Actions run log: shows the log extraction and DM send steps completing successfully

### Unit 4: Slack User Lookup and Graceful Fallback

**Purpose:** Proves the user resolution works end-to-end and fails safely when a user cannot be found.

**Functional Requirements:**
- The system shall fetch the triggering actor's public profile email via `GET /users/{username}` using `GITHUB_TOKEN`
- The system shall call the Slack API `users.lookupByEmail` using `SLACK_BOT_TOKEN` to resolve the Slack user ID
- The system shall open a DM channel via `conversations.open` before posting the message
- The user shall receive a Slack DM from the bot (not a generic webhook post)
- The system shall log a clear warning and exit successfully (without failing the notification workflow) if the GitHub email is not public or no matching Slack user is found

**Proof Artifacts:**
- GitHub Actions run log: shows the user lookup step returning a Slack user ID
- Screenshot: Slack DM is sent from a named bot (not "Incoming Webhook") demonstrating bot token auth
- GitHub Actions run log (fallback case): shows the warning message logged and workflow completing with `success` conclusion when user lookup fails

## Non-Goals (Out of Scope)

1. **Channel notifications**: All notifications are DMs only; no shared channel posting
2. **Per-workflow opt-in/out**: All workflows trigger notifications; no filtering by workflow name or label
3. **Cancellation notifications**: Only `success` and `failure` conclusions are handled; `cancelled` and `skipped` are ignored
4. **Custom message templates per workflow**: All notifications use the same message format
5. **Slack app setup automation**: This spec covers the workflow code only; the Slack app must be created manually and the bot token stored as a secret
6. **Automatic repo onboarding**: Consuming repos must manually copy and commit the caller workflow template; there is no auto-registration mechanism

## Design Considerations

No specific UI design requirements. The Slack DM messages should use Slack Block Kit for readable formatting:
- Use a header block with a status emoji (green check for success, red X for failure)
- Use section blocks for workflow name, branch, and commit info
- Use a code block for the error snippet (failure only)
- Include a button block linking to the GitHub Actions run

## Repository Standards

No existing standards are established — this is a greenfield project. Implementation should follow:
- GitHub Actions best practices: use `permissions` blocks to scope token access
- Shell scripts within workflow steps for log extraction (no external action dependencies unless well-maintained official actions)
- Secrets referenced via `${{ secrets.SECRET_NAME }}` syntax only

## Technical Considerations

- **Cross-repo reusable workflow**: `mumford-capstone` hosts the workflow declared with `on: workflow_call`. Consuming repos call it with `uses: <org>/mumford-capstone/.github/workflows/slack-notify.yml@main`. The `mumford-capstone` repo must be set to allow reusable workflow access from other repos in the org (GitHub org settings → Actions → "Allow all actions and reusable workflows" or allowlist).
- **`workflow_run` cannot cross repos**: Because `workflow_run` only fires within the same repo, each consuming repo needs its own small caller workflow that listens for `workflow_run` events locally, then delegates to the centralized reusable workflow via `workflow_call`. The caller template lives in `docs/caller-template/` in `mumford-capstone` for easy copying.
- **Caller template inputs**: The caller workflow extracts run context from `github.event.workflow_run` and passes it as `with:` inputs to the reusable workflow. The `SLACK_BOT_TOKEN` secret is forwarded via `secrets: inherit`.
- **GitHub token permissions**: The reusable workflow needs `actions: read` to fetch job details and logs. Declare this explicitly via the `permissions` block. The token used is the one from the consuming repo (passed automatically by `workflow_call`).
- **Slack Bot Token scopes required**: `users:read`, `users:read.email`, `im:write`, `chat:write`
- **Log size**: GitHub Actions job logs can be very large. Fetch the full log but extract only lines from the failed step using step name markers, then truncate to 3000 characters.
- **GitHub public email caveat**: GitHub users may not have a public email. The workflow must handle a `null` email response gracefully.
- **Slack API calls**: All Slack API calls should be made with `curl` within shell steps, checking the `ok` field in the JSON response and exiting with an error if `false`.

## Security Considerations

- `SLACK_BOT_TOKEN` must be stored as an encrypted GitHub Actions secret in **each consuming repo** (since `secrets: inherit` passes secrets from the calling repo, not `mumford-capstone`) and never hardcoded or logged
- `GITHUB_TOKEN` is automatically provided by GitHub Actions and should be scoped to `actions: read` only via the `permissions` block
- Error log snippets extracted from job logs may contain sensitive values (e.g., environment variable values printed during a failed command). Document this risk in the README and advise teams to avoid printing secrets in workflow steps.
- Proof artifact screenshots must be taken with real but non-sensitive workflows — they must not capture `SLACK_BOT_TOKEN` or any credentials in logs

## Success Metrics

1. **Delivery reliability**: DM is sent within 60 seconds of workflow completion for 100% of runs where the triggering user has a public GitHub email matching a Slack account
2. **Failure detail accuracy**: The failed step name and error snippet in the DM match the actual failure visible in the GitHub Actions UI
3. **Zero false failures**: The notification workflow itself never fails due to a missing user lookup — it always exits cleanly with a warning instead

## Open Questions

1. Should the bot fall back to posting to a default channel (e.g., `#ci-notifications`) when the Slack user cannot be resolved, rather than silently dropping the notification?
2. What should the Slack bot's display name and icon be?
