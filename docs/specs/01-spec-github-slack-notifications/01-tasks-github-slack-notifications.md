# 01-tasks-github-slack-notifications

## Relevant Files

- `.github/workflows/slack-notify.yml` - The centralized reusable workflow declared with `on: workflow_call`; this is the main deliverable that all other repos call
- `.github/workflows/test-success.yml` - A minimal test workflow with one job that always passes; used as the "consuming workflow" to trigger notification testing within this repo
- `.github/workflows/test-failure.yml` - A minimal test workflow with one job that always fails (`exit 1`); used to trigger failure notification testing
- `.github/workflows/test-caller.yml` - A `workflow_run` caller workflow that triggers when `test-success` or `test-failure` complete and delegates to the local `slack-notify.yml`
- `docs/caller-template/workflow-notify-caller.yml` - The ready-to-copy caller workflow template for consuming repos
- `docs/proof/success-dm.png` - Screenshot proof artifact for the success DM
- `docs/proof/failure-dm.png` - Screenshot proof artifact for the failure DM
- `docs/proof/onboarding-example.png` - Screenshot proof artifact showing a second repo successfully calling this reusable workflow
- `README.md` - Setup guide and onboarding documentation

### Notes

- All GitHub API calls should use `curl` with `-H "Authorization: Bearer $TOKEN"` and `-H "Accept: application/vnd.github+json"`.
- All Slack API calls should use `curl` with `-H "Authorization: Bearer $SLACK_BOT_TOKEN"` and `-H "Content-Type: application/json"`. Always check `"ok": true` in the JSON response.
- Use `jq` for JSON parsing within shell steps. It is pre-installed on GitHub-hosted `ubuntu-latest` runners.
- Job log content (`GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs`) is returned as plain text with ANSI escape codes. Strip ANSI codes with `sed 's/\x1b\[[0-9;]*m//g'` before using in Slack messages.
- Never `echo` or `cat` the value of `SLACK_BOT_TOKEN` in any step. Use step outputs (`$GITHUB_OUTPUT`) to pass values between steps rather than environment variables where possible.
- Proof artifact screenshots should use a real workflow run but must not show token values. The `docs/proof/` directory should be committed with placeholder images initially and replaced with real screenshots after a successful test run.

## Tasks

### [x] 1.0 Create Reusable Workflow Scaffold and Caller Template

**Purpose:** Establish the `workflow_call` interface in `mumford-capstone` and provide a ready-to-copy caller template for consuming repos. This is the infrastructure that all subsequent notification logic builds on.

#### 1.0 Proof Artifact(s)

- YAML lint: `.github/workflows/slack-notify.yml` passes `actionlint` or GitHub's built-in workflow validation with no errors, demonstrating the workflow is syntactically valid
- GitHub Actions run log (`mumford-capstone`): a test `workflow_call` invocation shows the reusable workflow receiving all declared inputs and completing with `success` conclusion, demonstrating the interface works
- File diff: `docs/caller-template/workflow-notify-caller.yml` exists and contains a valid `on: workflow_run` + `uses:` reference, demonstrating the template is ready for consuming repos to copy

#### 1.0 Tasks

- [x] 1.1 Create the `.github/workflows/` directory and create `.github/workflows/slack-notify.yml` with the `on: workflow_call` trigger block (no jobs yet)
- [x] 1.2 Add the `inputs:` block to `slack-notify.yml` declaring all 8 required string inputs: `workflow_name`, `workflow_conclusion`, `run_id`, `run_url`, `branch`, `commit_sha`, `commit_message`, `actor`; mark all as `required: true` with `type: string`
- [x] 1.3 Add the `secrets:` block declaring `SLACK_BOT_TOKEN` as a required secret
- [x] 1.4 Add a single job `notify` with `runs-on: ubuntu-latest` and a `permissions:` block scoping `actions: read` and `contents: read`
- [x] 1.5 Add a placeholder step inside the `notify` job that echoes all received inputs (e.g. `echo "Received: ${{ inputs.workflow_name }} / ${{ inputs.workflow_conclusion }}"`) — this verifies the interface before real logic is added
- [x] 1.6 Create `docs/caller-template/workflow-notify-caller.yml` with: `on: workflow_run` (triggering on all workflows, types `[completed]`), a single job that calls `uses: liatrio-forge/mumford-capstone/.github/workflows/slack-notify.yml@main`, maps all `with:` inputs from `github.event.workflow_run` context fields, and passes `secrets: inherit`
- [x] 1.7 Create `.github/workflows/test-success.yml` with a single job containing one step: `run: echo "test workflow completed successfully"`
- [x] 1.8 Create `.github/workflows/test-caller.yml` that triggers via `workflow_run` on `test-success` (types `[completed]`) and calls `./.github/workflows/slack-notify.yml` using a local path reference with all inputs mapped from `github.event.workflow_run`
- [x] 1.9 Commit and push all files; verify in the GitHub Actions UI that `test-success` and `test-caller` both appear and that `test-caller` completes with the echo output visible in the logs

---

### [x] 2.0 Implement Slack User Lookup with Graceful Fallback

**Purpose:** Resolve the GitHub actor's Slack user ID by fetching their public email from the GitHub API and looking it up in Slack. Handle the case where the email is not found without failing the workflow.

#### 2.0 Proof Artifact(s)

- GitHub Actions run log: shows the `Resolve Slack user` step outputting a valid Slack user ID (redacted, e.g. `U0123ABCDE`) after a real workflow run, demonstrating end-to-end email → Slack ID resolution
- GitHub Actions run log (fallback case): shows the step logging `WARNING: Could not resolve Slack user for actor <username> — skipping DM` and the workflow concluding with `success`, demonstrating graceful fallback when no email match is found

#### 2.0 Tasks

- [x] 2.1 Replace the placeholder step in `slack-notify.yml` with a step named `Get GitHub actor email` that calls `GET /users/${{ inputs.actor }}` using `curl` with `GITHUB_TOKEN`, parses the `email` field with `jq -r '.email'`, and writes it to `$GITHUB_OUTPUT` as `actor_email`
- [x] 2.2 Add a step named `Check email resolved` that reads `steps.<prev>.outputs.actor_email`; if the value is `null` or empty, it writes `skip=true` to `$GITHUB_OUTPUT` and logs `WARNING: No public email found for actor ${{ inputs.actor }} — skipping DM`
- [x] 2.3 Add a step named `Lookup Slack user by email` with `if: steps.check_email.outputs.skip != 'true'` that calls the Slack `users.lookupByEmail` API endpoint with the actor email using `curl`, checks the `ok` field in the response, and writes the user ID (`response.user.id`) to `$GITHUB_OUTPUT` as `slack_user_id`
- [x] 2.4 Add fallback handling inside the `Lookup Slack user by email` step: if `ok` is `false`, log `WARNING: Could not resolve Slack user for actor ${{ inputs.actor }} — skipping DM`, write `skip=true` to `$GITHUB_OUTPUT` for a `lookup_result` output, and exit `0` (not `1`) so the workflow does not fail
- [x] 2.5 Add a step named `Open Slack DM channel` with `if: steps.lookup_slack.outputs.skip != 'true'` that calls `conversations.open` with the resolved `slack_user_id` via `curl` and writes the returned channel ID to `$GITHUB_OUTPUT` as `dm_channel_id`
- [x] 2.6 Commit, push, and trigger a test run; verify the run log shows the user ID resolved step output and the DM channel opened

---

### [x] 3.0 Implement Success Notification DM

**Purpose:** Send a formatted Slack Block Kit DM to the resolved user when a workflow completes successfully, including workflow name, branch, and a link to the run.

#### 3.0 Proof Artifact(s)

- Screenshot: `docs/proof/success-dm.png` — Slack DM showing a green check header, workflow name, branch, and "View Run" button link, demonstrating the success message is delivered and formatted correctly
- GitHub Actions run log: shows the `Send success DM` step receiving `"ok": true` from the Slack API, demonstrating the message was accepted

#### 3.0 Tasks

- [x] 3.1 Add a step named `Send success DM` with `if: inputs.workflow_conclusion == 'success' && steps.open_dm.outputs.dm_channel_id != ''` that builds a Slack Block Kit JSON payload inline using a shell heredoc
- [x] 3.2 The Block Kit payload for success should contain: a `header` block with text `:white_check_mark: Workflow Succeeded`, a `section` block with fields for `Workflow`, `Branch`, and `Triggered by`, and an `actions` block with a single button labeled `View Run` linking to `${{ inputs.run_url }}`
- [x] 3.3 Post the payload using `curl` to `chat.postMessage` with the `channel` set to `steps.open_dm.outputs.dm_channel_id`; parse the response with `jq` and fail the step (`exit 1`) if `ok` is `false`
- [ ] 3.4 Commit and push; trigger `test-success.yml` manually via the GitHub Actions UI; take a screenshot of the resulting Slack DM and save it as `docs/proof/success-dm.png`

---

### [~] 4.0 Implement Failure Notification DM with Log Extraction

**Purpose:** When a workflow fails, fetch the failed job and step details from the GitHub API, extract the relevant error lines from the job log, and send a detailed Slack Block Kit DM to the triggering user.

#### 4.0 Proof Artifact(s)

- Screenshot: `docs/proof/failure-dm.png` — Slack DM showing a red X header, workflow name, failed job name, failed step name, a code-formatted error snippet, branch, commit SHA, and "View Run" button, demonstrating full failure detail delivery
- GitHub Actions run log: shows the `Fetch failed job logs` step retrieving log content and the `Extract error snippet` step outputting the truncated error lines, demonstrating the log extraction pipeline works

#### 4.0 Tasks

- [x] 4.1 Create `.github/workflows/test-failure.yml` with a single job containing one step: `run: echo "About to fail" && exit 1` so there is a workflow to trigger failure notifications with
- [x] 4.2 Update `.github/workflows/test-caller.yml` to also trigger on `test-failure` completing (add it to the `workflow_run` workflows list)
- [x] 4.3 Add a step named `Get failed job details` with `if: inputs.workflow_conclusion == 'failure'` that calls `GET /repos/${{ github.repository }}/actions/runs/${{ inputs.run_id }}/jobs` using `curl`, finds the first job where `conclusion == "failure"` using `jq`, and writes the job's `id` and `name` to `$GITHUB_OUTPUT` as `failed_job_id` and `failed_job_name`
- [x] 4.4 Add a step named `Get failed step name` that reads the jobs response from the previous step, finds the first step within the failed job where `conclusion == "failure"`, and writes its `name` to `$GITHUB_OUTPUT` as `failed_step_name`
- [x] 4.5 Add a step named `Fetch failed job logs` that calls `GET /repos/${{ github.repository }}/actions/jobs/${{ steps.get_job.outputs.failed_job_id }}/logs` using `curl` and saves the response body to a temporary file `/tmp/job.log`
- [x] 4.6 Add a step named `Extract error snippet` that reads `/tmp/job.log`, strips ANSI escape codes with `sed`, finds the section of the log belonging to the failed step (lines between the step header and the next step header), takes the last 20 lines, truncates to 3000 characters, and writes the result to `$GITHUB_OUTPUT` as `error_snippet`
- [x] 4.7 Add a step named `Send failure DM` with `if: inputs.workflow_conclusion == 'failure' && steps.open_dm.outputs.dm_channel_id != ''` that builds a Slack Block Kit JSON payload containing: a `header` block with `:x: Workflow Failed`, `section` fields for `Workflow`, `Failed Job`, `Failed Step`, `Branch`, `Commit` (SHA shortened to 7 chars + message), and a `section` with a code-formatted `error_snippet` in a text block, plus an `actions` button linking to the run URL
- [x] 4.8 Post the failure payload with `chat.postMessage` (same pattern as task 3.3); verify `ok: true` in the response
- [ ] 4.9 Commit and push; trigger `test-failure.yml` manually; take a screenshot of the resulting Slack DM and save it as `docs/proof/failure-dm.png`

---

### [ ] 5.0 Add Test Workflow and README Documentation

**Purpose:** Provide a working test workflow in `mumford-capstone` to demonstrate the full notification pipeline, and write a README explaining how any repo in the org can adopt the integration.

#### 5.0 Proof Artifact(s)

- GitHub Actions run log: `.github/workflows/test-caller.yml` run showing both a success and a deliberate failure completing and the `slack-notify` reusable workflow executing, demonstrating end-to-end integration within this repo
- File diff: `README.md` contains setup instructions including required Slack bot scopes, secret names, and the 3-step onboarding guide for consuming repos
- Screenshot: `docs/proof/onboarding-example.png` — a second repo's Actions tab showing its caller workflow successfully invoking `liatrio-forge/mumford-capstone/.github/workflows/slack-notify.yml@main`, demonstrating cross-repo reusability

#### 5.0 Tasks

- [ ] 5.1 Write `README.md` with the following sections: **Overview** (what this repo does), **Slack App Setup** (step-by-step: create Slack app, add bot scopes `users:read`, `users:read.email`, `im:write`, `chat:write`, install to workspace, copy bot token), **GitHub Secret Setup** (add `SLACK_BOT_TOKEN` as a repository secret in each consuming repo), **Onboarding a New Repo** (3 steps: copy `docs/caller-template/workflow-notify-caller.yml` into `.github/workflows/`, add `SLACK_BOT_TOKEN` secret, push — no other changes needed), **Security Note** (warn about log snippets potentially containing sensitive values)
- [ ] 5.2 Add a `docs/proof/` directory with placeholder `README.md` explaining that screenshots are added after live testing; commit the placeholder so the directory is tracked by git
- [ ] 5.3 Confirm the GitHub org Actions settings allow reusable workflows: navigate to `github.com/organizations/liatrio-forge/settings/actions` and verify "Allow all actions and reusable workflows" is enabled (or add `mumford-capstone` to the allowlist); document this prerequisite in the README
- [ ] 5.4 In a second repo in the org, copy `docs/caller-template/workflow-notify-caller.yml` into `.github/workflows/`, add `SLACK_BOT_TOKEN` as a secret, and push a commit to trigger a test run; capture a screenshot of the Actions tab showing the caller workflow invoking the reusable workflow and save as `docs/proof/onboarding-example.png`
- [ ] 5.5 Commit all final files (`README.md`, `docs/proof/` screenshots) and push; verify the GitHub repo at `github.com/liatrio-forge/mumford-capstone` shows all workflows healthy and the README renders correctly
