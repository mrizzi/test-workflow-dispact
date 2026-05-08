# Dispatch Test Workflows

**Date**: 2026-05-08
**Scope**: Two sample GitHub workflows to validate the `workflow_run` dispatch pattern for fork PR secret access
**Validates**: [Eval PR Fork Dispatch](https://github.com/mrizzi/sdlc-plugins/blob/main/docs/specs/2026-05-07-eval-pr-fork-dispatch-design.md) design
**Reference implementation**: [PR #123](https://github.com/mrizzi/sdlc-plugins/pull/123)

## Purpose

Prove that the two-stage `workflow_run` dispatch pattern works end-to-end in an isolated test repository (`mrizzi/test-workflow-dispact`) before relying on it in production. The test workflows mirror the spec's structure with a simplified payload: post a PR comment proving secret access, or fail if the secret is missing.

## Architecture

```
PR opened to main
    │
    ▼
┌──────────────────────────────────┐
│  dispatch-test.yml (pull_request)│  No secrets needed
│  1. Checkout                     │
│  2. Upload marker artifact       │
└──────────────┬───────────────────┘
               │ workflow_run (completed)
               ▼
┌──────────────────────────────────────────┐
│  dispatch-test-run.yml (workflow_run)    │  Base repo context — secrets available
│                                          │
│  Job 1: discover                         │
│  - Download artifact (proves handoff)    │
│  - Derive PR# and author from API       │
│  - Check collaborator trust              │
│                                          │
│  Job 2: gate                             │
│  - Runs ONLY if trusted != true          │
│  - environment: eval-protected           │
│  - Blocks until reviewer approves        │
│                                          │
│  Job 3: prove-secrets                    │
│  - Fail if TEST_SECRET is missing        │
│  - Post PR comment confirming success    │
└──────────────────────────────────────────┘
```

## Security Model

Follows the hardened pattern from [PR #123](https://github.com/mrizzi/sdlc-plugins/pull/123):

- The artifact carries **no PR identity data** — only a trivial marker file proving the upload/download handoff works.
- `pr_number` and `author` are derived in Stage 2 from tamper-proof sources:
  - `head_sha`: `context.payload.workflow_run.head_sha` (set by GitHub)
  - PR lookup: `listPullRequestsAssociatedWithCommit` API (authoritative)
  - Author: `pr.user.login` from the API response (authoritative)
- A fork attacker modifying Stage 1 cannot spoof author identity to bypass the trust gate.

## Workflow 1: `dispatch-test.yml`

**Trigger:** `pull_request` on `main`, no path filter.

**Permissions:** `contents: read`

**Steps:**

1. **Checkout** — `actions/checkout@v4`
2. **Write marker** — `echo "triggered" > marker.txt`
3. **Upload artifact** — `actions/upload-artifact@v4` with name `dispatch-metadata`, containing `marker.txt`

**Workflow name:** `"Dispatch Test"` — Stage 2 references this name in its `workflows:` filter.

The artifact carries zero security-sensitive data. It exists solely to prove the cross-workflow artifact handoff mechanism works.

## Workflow 2: `dispatch-test-run.yml`

**Trigger:** `workflow_run` on `"Dispatch Test"`, type `completed`.

**Permissions:** `contents: read`, `pull-requests: write`, `actions: read`

### Job 1: `discover`

Skips if `github.event.workflow_run.conclusion != 'success'`.

**Steps:**

1. **Download artifact** from the triggering run via `actions/download-artifact@v4` with `run-id: ${{ github.event.workflow_run.id }}`.

2. **Resolve PR identity and check trust** via `actions/github-script@v7`:
   - Get `head_sha` from `context.payload.workflow_run.head_sha`
   - Call `listPullRequestsAssociatedWithCommit` to find the open PR targeting `main`
   - Extract `pr_number` and `author` from the PR object
   - Call `getCollaboratorPermissionLevel` for the author
   - Set `trusted = true` if permission is `admin` or `write`

3. **Set outputs:** `pr_number`, `author`, `trusted`

### Job 2: `gate`

```yaml
gate:
  needs: discover
  if: >-
    needs.discover.result == 'success' &&
    needs.discover.outputs.trusted != 'true' &&
    needs.discover.outputs.pr_number != ''
  runs-on: ubuntu-latest
  environment: eval-protected
  steps:
    - run: echo "Approved by reviewer"
```

Skipped for trusted authors. For untrusted authors, blocks until a reviewer approves in the Actions UI.

### Job 3: `prove-secrets`

```yaml
prove-secrets:
  needs: [discover, gate]
  if: >-
    !cancelled() &&
    needs.discover.result == 'success' &&
    needs.discover.outputs.pr_number != '' &&
    (needs.discover.outputs.trusted == 'true' || needs.gate.result == 'success')
  runs-on: ubuntu-latest
```

**Steps:**

1. **Check secret exists** — Fail if `TEST_SECRET` is empty:
   ```yaml
   - name: Verify secret access
     env:
       TEST_SECRET: ${{ secrets.TEST_SECRET }}
     run: |
       if [ -z "$TEST_SECRET" ]; then
         echo "::error::TEST_SECRET is not available — secret access failed"
         exit 1
       fi
       echo "Secret is present (length: ${#TEST_SECRET})"
   ```

2. **Post PR comment** — Use `actions/github-script@v7` to post a comment on the PR:
   ```javascript
   const prNumber = parseInt(process.env.PR_NUMBER);
   const author = process.env.PR_AUTHOR;
   const trusted = process.env.TRUSTED;
   const headSha = context.payload.workflow_run.head_sha;

   const status = trusted === 'true'
     ? `trusted`
     : `gated — approved by reviewer`;

   await github.rest.issues.createComment({
     owner: context.repo.owner,
     repo: context.repo.repo,
     issue_number: prNumber,
     body: [
       `## Dispatch Test Result`,
       ``,
       `| Field | Value |`,
       `|-------|-------|`,
       `| Secret access | present |`,
       `| Author | ${author} (${status}) |`,
       `| SHA | \`${headSha.substring(0, 7)}\` |`,
       `| Workflow run | [${context.runId}](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}) |`,
     ].join('\n')
   });
   ```

   The `PR_NUMBER`, `PR_AUTHOR`, and `TRUSTED` values are passed via `env:` to avoid `${{ }}` interpolation of untrusted data in shell/script contexts.

## Test Scenarios

### Scenario 1: Trusted author (repo owner)

1. Create a branch, add any file, open a PR to `main`
2. Stage 1 fires, uploads marker artifact
3. Stage 2 fires, `discover` resolves owner via API → `trusted=true`
4. `gate` job is **skipped**
5. `prove-secrets` reads `TEST_SECRET`, posts PR comment

**Expected comment:** "Author: mrizzi (trusted). Secret access: present."

### Scenario 2: Untrusted author (fork PR)

1. Fork the repo from a different GitHub account
2. Open a PR from the fork to `main`
3. Stage 1 fires in fork context (no secrets)
4. Stage 2 fires in base repo context, resolves fork author → `trusted=false`
5. `gate` blocks, waiting for reviewer approval in `eval-protected` environment
6. Reviewer approves in the Actions UI
7. `prove-secrets` reads `TEST_SECRET`, posts PR comment

**Expected comment:** "Author: fork-user (gated — approved by reviewer). Secret access: present."

## Setup Requirements

### Before testing

| Requirement | How |
|-------------|-----|
| Repository secret `TEST_SECRET` | Settings > Secrets > Actions > New: name=`TEST_SECRET`, value=any non-empty string |
| Protected environment `eval-protected` | Settings > Environments > New: name=`eval-protected`, add at least one required reviewer |
| Initial commit on `main` | The repo needs at least one commit with the workflow files before PRs can be opened |
| Second GitHub account (Scenario 2) | Fork the repo from a different account and open a PR |

### No other infrastructure

- No external services, APIs, or credentials beyond `TEST_SECRET`
- No GitHub App or PAT — uses the default `GITHUB_TOKEN`

## File Changes

| File | Change |
|------|--------|
| `.github/workflows/dispatch-test.yml` | New — Stage 1: checkout + artifact upload |
| `.github/workflows/dispatch-test-run.yml` | New — Stage 2: trust check, gate, secret proof, PR comment |
