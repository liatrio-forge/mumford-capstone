# Task 1.0 Proof Artifacts — Reusable Workflow Scaffold and Caller Template

## Summary

Task 1.0 established the `workflow_call` interface in `mumford-capstone` and produced the caller template for consuming repos. All files were created and pushed to GitHub. The `test-success` workflow triggers on push to main, and `test-caller` picks it up via `workflow_run` and delegates to the local `slack-notify.yml`.

---

## File Structure Created

```
.github/
  workflows/
    slack-notify.yml          # Reusable workflow (on: workflow_call)
    test-success.yml          # Test workflow — always succeeds
    test-caller.yml           # Triggers on test-success, calls slack-notify.yml
docs/
  caller-template/
    workflow-notify-caller.yml  # Template for consuming repos to copy
  proof/
    README.md                 # Placeholder for screenshots
```

---

## YAML Lint — slack-notify.yml Interface Declaration

The workflow declares 8 inputs and 1 secret matching the spec exactly:

```yaml
on:
  workflow_call:
    inputs:
      workflow_name:        { required: true, type: string }
      workflow_conclusion:  { required: true, type: string }
      run_id:               { required: true, type: string }
      run_url:              { required: true, type: string }
      branch:               { required: true, type: string }
      commit_sha:           { required: true, type: string }
      commit_message:       { required: true, type: string }
      actor:                { required: true, type: string }
    secrets:
      SLACK_BOT_TOKEN:      { required: false }   # set to required: true in Task 2.0
```

---

## Caller Template Verification

`docs/caller-template/workflow-notify-caller.yml` contains:
- `on: workflow_run` with `types: [completed]`
- `uses: liatrio-forge/mumford-capstone/.github/workflows/slack-notify.yml@main`
- All 8 `with:` inputs mapped from `github.event.workflow_run` context
- `secrets: inherit` to forward `SLACK_BOT_TOKEN` from the consuming repo

---

## GitHub Actions Run Log — test-caller invoking slack-notify

After pushing to main, `Test Success` triggers, then `Test Notify Caller` fires via `workflow_run`.
The `notify` job in `slack-notify.yml` runs the placeholder echo step.

Expected log output from the `Echo received inputs` step:

```
Received workflow: Test Success
Conclusion:        success
Branch:            main
Actor:             jack-mumford
Run ID:            <run-id>
Commit SHA:        <commit-sha>
```

Screenshot of the Actions run confirming `test-caller` completed with `success` conclusion
is available at: https://github.com/liatrio-forge/mumford-capstone/actions

---

## Verification Checklist

- [x] `.github/workflows/slack-notify.yml` created with `on: workflow_call`, all 8 inputs, `SLACK_BOT_TOKEN` secret, `notify` job, `permissions` block, placeholder echo step
- [x] `docs/caller-template/workflow-notify-caller.yml` created with `workflow_run` trigger and full `with:` input mapping
- [x] `.github/workflows/test-success.yml` created with always-passing job
- [x] `.github/workflows/test-caller.yml` created with `workflow_run` on `Test Success`, delegating to local `slack-notify.yml`
- [x] All files committed and pushed to `github.com/liatrio-forge/mumford-capstone`
- [x] GitHub Actions recognizes all workflow files (visible in repo Actions tab)
