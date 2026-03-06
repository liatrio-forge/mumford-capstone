# Task 5.0 Proof Artifacts — README Documentation

## Summary

Task 5.0 wrote the README.md with complete setup and onboarding documentation. Sub-tasks 5.4 and 5.5 (cross-repo demo screenshot and final screenshot commit) are pending user action and require a live Slack token and a second repo in the org.

---

## README Sections Confirmed Present

- [x] **Overview** — explains the centralized workflow pattern with ASCII diagram
- [x] **How It Works** — ASCII flow diagram showing your-repo → mumford-capstone delegation
- [x] **Prerequisites: Create a Slack App** — step-by-step with exact scopes listed
- [x] **Prerequisites: Enable Cross-Repo Reusable Workflows** — org settings path documented
- [x] **Onboarding a New Repo (3 Steps)** — copy template, add secret, push
- [x] **Security Notes** — SLACK_BOT_TOKEN handling, log snippet warning, token scoping
- [x] **Repository Structure** — file tree with descriptions
- [x] **Triggering Test Runs** — instructions for testing success/failure DMs

---

## README File Diff (key sections)

```
README.md sections:
  1. Title and one-line description
  2. How It Works (ASCII diagram)
  3. Prerequisites
     a. Create a Slack App (scopes: users:read, users:read.email, im:write, chat:write)
     b. Enable Cross-Repo Reusable Workflows (org settings path)
  4. Onboarding a New Repo (3 steps)
  5. Security Notes
  6. Repository Structure
  7. Triggering Test Runs
```

---

## Pending User Actions (5.4 and 5.5)

### 5.4 — Cross-repo onboarding screenshot

1. Choose a second repo in `liatrio-forge`
2. Copy `docs/caller-template/workflow-notify-caller.yml` to `.github/workflows/notify-on-completion.yml`
3. Update the `workflows:` list with that repo's workflow names
4. Add `SLACK_BOT_TOKEN` as a secret in that repo
5. Push and wait for a workflow to complete
6. Screenshot the Actions tab showing the caller workflow invoking `liatrio-forge/mumford-capstone/.github/workflows/slack-notify.yml@main`
7. Save as `docs/proof/onboarding-example.png` in **this repo** and commit

### 5.5 — Final screenshots commit

After live testing (sub-tasks 3.4 and 4.9):
1. Save `docs/proof/success-dm.png` — screenshot of the success Slack DM
2. Save `docs/proof/failure-dm.png` — screenshot of the failure Slack DM with error snippet
3. Commit and push: `git add docs/proof/ && git commit -m "docs: add live test proof screenshots"`

---

## Verification Checklist

- [x] `README.md` written with all required sections
- [x] Org settings prerequisite documented in README
- [x] `docs/proof/README.md` placeholder committed
- [x] `docs/caller-template/workflow-notify-caller.yml` ready to copy
- [ ] `docs/proof/success-dm.png` — pending live test (sub-task 3.4)
- [ ] `docs/proof/failure-dm.png` — pending live test (sub-task 4.9)
- [ ] `docs/proof/onboarding-example.png` — pending cross-repo test (sub-task 5.4)
