# Issue Triage: Finding Closeable Issues

Instructions for scanning a GitHub repository's open issues to find ones that can be closed because they've been implemented or become irrelevant.

## Core Principle

Every closure proposal must be backed by verified evidence. False positives waste maintainer time and erode trust. It is better to miss a closeable issue than to incorrectly propose closing one that is still valid.

## Step-by-step Process

### 1. Build an exclusion list using closingIssuesReferences

PRs that use GitHub closing keywords (`closes`, `fixes`, `resolves`) automatically close the linked issue when merged. If such an issue is still open, it means a maintainer **intentionally reopened it** because the PR did not fully resolve it. Use this API to identify these issues so you can **skip them**:

```bash
gh api graphql -f query='{
  repository(owner:"OWNER", name:"REPO") {
    pullRequests(states:MERGED, last:100, orderBy:{field:CREATED_AT, direction:DESC}) {
      nodes {
        number title mergedAt
        closingIssuesReferences(first:10) {
          nodes { number title state }
        }
      }
      pageInfo { startCursor hasPreviousPage }
    }
  }
}' --jq '.data.repository.pullRequests.nodes[] | select(.closingIssuesReferences.nodes | length > 0) | {pr: .number, prTitle: .title, mergedAt: .mergedAt, issues: [.closingIssuesReferences.nodes[] | select(.state == "OPEN") | {number: .number, title: .title}]} | select(.issues | length > 0)'
```

Use cursor-based pagination to cover the full PR history. **Do not propose closing these issues** — they were already auto-closed by GitHub and deliberately reopened by maintainers who determined more work was needed.

### 2. Check issue timelines for cross-referenced merged PRs

For each open issue, check if merged PRs reference it:

```bash
gh api repos/OWNER/REPO/issues/{number}/timeline \
  --jq '[.[] | select(.event == "cross-referenced") | select(.source.issue.pull_request != null) | select(.source.issue.state == "closed") | {pr: .source.issue.number, title: .source.issue.title}]'
```

This finds PRs that mention the issue without closing keywords — these are the real candidates since they wouldn't have triggered GitHub's auto-close. However, a mention is not a fix; these require deeper investigation via the validation checks below.

### 3. Validate every candidate

For every issue-PR pair that looks like a match, you **must** perform all of the following checks before proposing closure.

#### Check A: Verify the PR was actually merged

```bash
gh pr view {PR_NUMBER} --repo OWNER/REPO --json state,mergedAt
```

The `state` must be `MERGED`. A PR with state `CLOSED` was closed without merging — its changes are **not** in the codebase. Do not propose closing an issue based on an unmerged PR.

**Common mistake:** The timeline API returns cross-references from closed (unmerged) PRs with `source.issue.state == "closed"`. A closed PR is not the same as a merged PR. Always verify explicitly.

#### Check B: Verify the PR was merged AFTER the issue was created

```bash
gh pr view {PR_NUMBER} --repo OWNER/REPO --json mergedAt
gh issue view {ISSUE_NUMBER} --repo OWNER/REPO --json createdAt
```

If the PR was merged before the issue was created, the issue was opened **despite** that PR existing. This means the PR does not resolve the issue — the author already knew about the PR and still felt the issue was necessary. This applies even when the time difference is small (minutes or seconds).

#### Check C: Verify the issue was not previously auto-closed and reopened

If a merged PR used a closing keyword for an issue, GitHub would have auto-closed it. If the issue is currently open, it was reopened by a maintainer who determined the PR did not fully resolve it. Check the issue timeline for reopen events:

```bash
gh api repos/OWNER/REPO/issues/{number}/timeline \
  --jq '[.[] | select(.event == "reopened") | {actor: .actor.login, date: .created_at}]'
```

Issues that were auto-closed and reopened should generally remain open.

#### Check D: Verify the PR actually addresses the issue's concern

Read both the issue body and the PR body/description:

```bash
gh issue view {ISSUE_NUMBER} --repo OWNER/REPO --json body
gh pr view {PR_NUMBER} --repo OWNER/REPO --json body
```

Watch out for:
- **Partial fixes**: PRs that say "partially resolves", "part of", "fixes some of", or "partially targeting" do not fully close an issue.
- **Related but different**: A PR that touches the same area of code but solves a different problem. Title similarity is not sufficient evidence.
- **Multi-part issues**: Issues with checkboxes, numbered stages, or multiple requests where the PR only addresses some of them.

#### Check E: Verify the fix exists in the current codebase

This is the most important check. PR metadata and descriptions can be misleading — the actual code is the ground truth. Clone the repo (or read files via the GitHub API) and verify that the fix described by the PR is present in the current default branch:

```bash
# Check the files the PR changed still contain the fix
gh pr view {PR_NUMBER} --repo OWNER/REPO --json files --jq '.files[].path'
# Then read the relevant files and confirm the fix is present
gh api repos/OWNER/REPO/contents/{FILE_PATH} --jq '.content' | base64 -d
```

Reasons the fix may not be in the codebase despite a merged PR:
- **The PR was later reverted** — check `git log` for revert commits referencing the PR.
- **The code was refactored or rewritten** — the fix may have been lost in a subsequent rewrite of the same area.
- **The fix was on a non-default branch** — the PR may have merged into a feature branch, not `main`.

For bug fixes, confirm the specific code change (e.g. the added validation, the changed conditional, the new error handling) is still present. For feature requests, confirm the feature exists and works as described. Do not rely solely on the PR having been merged.

### 4. Title matching is not evidence

Never propose closing an issue solely because a PR title seems related. PR and issue titles can look similar while addressing different aspects of the same area. Always verify through the checks above.

### 5. Batch efficiently but validate individually

When scanning hundreds of issues, it's appropriate to use batch API calls and parallel agents for the initial scan. But the validation steps (Checks A-E) must be done for each individual candidate — do not skip validation to save time.

## Writing Closure Comments

When proposing closure:
- Reference the specific merged PR(s) with `#NUMBER`
- Briefly explain why the PR resolves the issue
- For medium-confidence cases, explicitly ask the maintainer to verify
- Use the `proposed_for_closure` label (if the repo uses one) rather than closing directly

## Checklist Before Proposing Closure

For each issue you want to propose closing, confirm:

- [ ] The referenced PR has `state: MERGED` (not just `CLOSED`)
- [ ] The PR `mergedAt` date is after the issue `createdAt` date
- [ ] The issue was not previously auto-closed and then reopened by a maintainer
- [ ] The PR body/description addresses the specific concern in the issue body
- [ ] The PR is not a partial fix (look for "partially", "part of", "some of")
- [ ] You have read both the issue body and PR body, not just titles
- [ ] You have checked the current codebase and confirmed the fix is actually present (not reverted, rewritten, or lost)
