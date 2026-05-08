# Dispatch Test Workflows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create two GitHub Actions workflows that prove the `workflow_run` dispatch pattern works for fork PR secret access with a collaborator trust gate.

**Architecture:** Stage 1 (`dispatch-test.yml`, `pull_request`) uploads a marker artifact. Stage 2 (`dispatch-test-run.yml`, `workflow_run`) downloads the artifact, derives PR identity from the GitHub API, checks collaborator trust, gates untrusted authors via a protected environment, then posts a PR comment proving secret access.

**Tech Stack:** GitHub Actions, `actions/upload-artifact@v4`, `actions/download-artifact@v4`, `actions/github-script@v7`, GitHub REST API

**Spec:** `docs/superpowers/specs/2026-05-08-dispatch-test-workflows-design.md`

---

### Task 1: Create Stage 1 workflow (`dispatch-test.yml`)

**Files:**
- Create: `.github/workflows/dispatch-test.yml`

- [ ] **Step 1: Create the workflow file**

```yaml
name: Dispatch Test

on:
  pull_request:
    branches: [main]

jobs:
  upload-metadata:
    name: Upload Metadata
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Write marker file
        run: echo "triggered" > marker.txt

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: dispatch-metadata
          path: marker.txt
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/dispatch-test.yml')); print('Valid YAML')"`

Expected: `Valid YAML`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/dispatch-test.yml
git commit -m "feat(ci): add dispatch-test.yml — Stage 1 artifact upload"
```

---

### Task 2: Create Stage 2 workflow (`dispatch-test-run.yml`)

**Files:**
- Create: `.github/workflows/dispatch-test-run.yml`

- [ ] **Step 1: Create the workflow file**

```yaml
name: Dispatch Test Run

on:
  workflow_run:
    workflows: ["Dispatch Test"]
    types: [completed]

permissions:
  contents: read
  pull-requests: write
  actions: read

jobs:
  discover:
    name: Discover PR Context
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    outputs:
      pr_number: ${{ steps.metadata.outputs.pr_number }}
      author: ${{ steps.metadata.outputs.author }}
      trusted: ${{ steps.metadata.outputs.trusted }}
    steps:
      - name: Download metadata artifact
        uses: actions/download-artifact@v4
        with:
          name: dispatch-metadata
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Verify artifact contents
        run: |
          echo "Artifact downloaded:"
          cat marker.txt

      - name: Resolve PR identity and check trust
        id: metadata
        uses: actions/github-script@v7
        with:
          script: |
            const headSha = context.payload.workflow_run.head_sha;
            console.log(`Resolving PR for commit ${headSha}`);

            const { data: prs } = await github.rest.repos.listPullRequestsAssociatedWithCommit({
              owner: context.repo.owner,
              repo: context.repo.repo,
              commit_sha: headSha
            });

            const openToMain = prs.filter(p => p.state === 'open' && p.base.ref === 'main');
            if (openToMain.length === 0) {
              core.warning(`No open PR targeting main found for commit ${headSha}`);
              core.setOutput('pr_number', '');
              core.setOutput('author', '');
              core.setOutput('trusted', 'false');
              return;
            }

            const pr = openToMain[0];
            const prNumber = pr.number;
            const author = pr.user.login;
            console.log(`Found PR #${prNumber} by ${author}`);

            core.setOutput('pr_number', prNumber.toString());
            core.setOutput('author', author);

            let trusted = false;
            try {
              const { data } = await github.rest.repos.getCollaboratorPermissionLevel({
                owner: context.repo.owner,
                repo: context.repo.repo,
                username: author
              });
              trusted = ['admin', 'write'].includes(data.permission);
              console.log(`Author ${author}: permission=${data.permission}, trusted=${trusted}`);
            } catch (e) {
              console.log(`Author ${author}: not a collaborator (${e.status}), trusted=false`);
            }

            core.setOutput('trusted', trusted.toString());

  gate:
    name: Approval Gate
    needs: discover
    if: >-
      needs.discover.result == 'success' &&
      needs.discover.outputs.trusted != 'true' &&
      needs.discover.outputs.pr_number != ''
    runs-on: ubuntu-latest
    environment: eval-protected
    steps:
      - run: echo "Approved by reviewer"

  prove-secrets:
    name: Prove Secret Access
    needs: [discover, gate]
    if: >-
      !cancelled() &&
      needs.discover.result == 'success' &&
      needs.discover.outputs.pr_number != '' &&
      (needs.discover.outputs.trusted == 'true' || needs.gate.result == 'success')
    runs-on: ubuntu-latest
    steps:
      - name: Verify secret access
        env:
          TEST_SECRET: ${{ secrets.TEST_SECRET }}
        run: |
          if [ -z "$TEST_SECRET" ]; then
            echo "::error::TEST_SECRET is not available — secret access failed"
            exit 1
          fi
          echo "Secret is present (length: ${#TEST_SECRET})"

      - name: Post PR comment
        env:
          PR_NUMBER: ${{ needs.discover.outputs.pr_number }}
          PR_AUTHOR: ${{ needs.discover.outputs.author }}
          TRUSTED: ${{ needs.discover.outputs.trusted }}
        uses: actions/github-script@v7
        with:
          script: |
            const prNumber = parseInt(process.env.PR_NUMBER);
            const author = process.env.PR_AUTHOR;
            const trusted = process.env.TRUSTED;
            const headSha = context.payload.workflow_run.head_sha;

            const status = trusted === 'true'
              ? 'trusted'
              : 'gated — approved by reviewer';

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prNumber,
              body: [
                '## Dispatch Test Result',
                '',
                '| Field | Value |',
                '|-------|-------|',
                `| Secret access | present |`,
                `| Author | ${author} (${status}) |`,
                `| SHA | \`${headSha.substring(0, 7)}\` |`,
                `| Workflow run | [${context.runId}](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}) |`,
              ].join('\n')
            });
```

- [ ] **Step 2: Validate YAML syntax**

Run: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/dispatch-test-run.yml')); print('Valid YAML')"`

Expected: `Valid YAML`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/dispatch-test-run.yml
git commit -m "feat(ci): add dispatch-test-run.yml — Stage 2 trust check, gate, secret proof"
```

---

### Task 3: Push to GitHub and configure repository

- [ ] **Step 1: Push both workflows to `main`**

```bash
git push -u origin main
```

- [ ] **Step 2: Create repository secret `TEST_SECRET`**

Run:
```bash
gh secret set TEST_SECRET --body "dispatch-test-proof" --repo mrizzi/test-workflow-dispact
```

Expected: `✓ Set Actions secret TEST_SECRET for mrizzi/test-workflow-dispact`

- [ ] **Step 3: Create protected environment `eval-protected`**

This must be done in the GitHub UI:
1. Go to https://github.com/mrizzi/test-workflow-dispact/settings/environments
2. Click "New environment", name: `eval-protected`
3. Check "Required reviewers", add yourself (`mrizzi`)
4. Click "Save protection rules"

- [ ] **Step 4: Commit step acknowledgment**

No code changes — this task is repo configuration only.

---

### Task 4: Test Scenario 1 — Trusted author (repo owner)

- [ ] **Step 1: Create a test branch and PR**

```bash
git checkout -b test/trusted-author
echo "Test file for trusted author scenario" > test-trusted.txt
git add test-trusted.txt
git commit -m "test: trigger dispatch for trusted author scenario"
git push -u origin test/trusted-author
gh pr create --title "Test: Trusted author dispatch" --body "Testing dispatch workflow with repo owner (trusted author)."
```

- [ ] **Step 2: Verify Stage 1 triggers**

```bash
gh run list --workflow dispatch-test.yml --limit 1
```

Expected: A run in `completed` / `success` status for the PR branch.

- [ ] **Step 3: Verify Stage 2 triggers and completes**

```bash
gh run list --workflow dispatch-test-run.yml --limit 1
```

Expected: A run in `completed` / `success` status. The `gate` job should show as `skipped`.

- [ ] **Step 4: Verify PR comment was posted**

```bash
gh pr view --comments --repo mrizzi/test-workflow-dispact
```

Expected: A comment with the "Dispatch Test Result" table showing:
- Secret access: `present`
- Author: `mrizzi (trusted)`
- SHA and workflow run link populated

- [ ] **Step 5: Clean up**

```bash
gh pr close --delete-branch
git checkout main
```

---

### Task 5: Test Scenario 2 — Untrusted author (fork PR)

This requires a second GitHub account or a collaborator with a fork.

- [ ] **Step 1: From the second account, fork the repo**

Go to `https://github.com/mrizzi/test-workflow-dispact` and click "Fork".

- [ ] **Step 2: From the fork, create a branch and open a PR**

```bash
git clone https://github.com/<fork-owner>/test-workflow-dispact.git
cd test-workflow-dispact
git checkout -b test/untrusted-author
echo "Test file from fork" > test-fork.txt
git add test-fork.txt
git commit -m "test: trigger dispatch for untrusted author scenario"
git push -u origin test/untrusted-author
gh pr create --title "Test: Untrusted author dispatch" --body "Testing dispatch workflow from a fork (untrusted author)." --repo mrizzi/test-workflow-dispact
```

- [ ] **Step 3: Verify Stage 1 triggers**

```bash
gh run list --workflow dispatch-test.yml --limit 1 --repo mrizzi/test-workflow-dispact
```

Expected: A run in `completed` / `success` status.

- [ ] **Step 4: Verify Stage 2 triggers and `gate` job is pending approval**

```bash
gh run list --workflow dispatch-test-run.yml --limit 1 --repo mrizzi/test-workflow-dispact
```

Expected: A run in `waiting` status. The `gate` job should show as pending, waiting for environment approval.

- [ ] **Step 5: Approve the gate in the Actions UI**

Go to the workflow run URL and click "Review deployments" > select `eval-protected` > "Approve and deploy".

- [ ] **Step 6: Verify `prove-secrets` completes and PR comment is posted**

```bash
gh run list --workflow dispatch-test-run.yml --limit 1 --repo mrizzi/test-workflow-dispact
gh pr view <PR_NUMBER> --comments --repo mrizzi/test-workflow-dispact
```

Expected: Run in `completed` / `success`. PR comment with:
- Secret access: `present`
- Author: `<fork-owner> (gated — approved by reviewer)`

- [ ] **Step 7: Clean up**

```bash
gh pr close <PR_NUMBER> --delete-branch --repo mrizzi/test-workflow-dispact
```
