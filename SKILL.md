---
name: review-eng1-ticket
description: Review and test ENG1 Jira tickets — resolve the Code Review subtask, merge the ticket branch into stage when present, run test instructions (including browser), and update Jira/GitHub on pass, fail, or merge conflict. Use when the user explicitly invokes this skill with a Jira key to review, test, or code-review an ENG1 ticket.
disable-model-invocation: true
---

# Review ENG1 Ticket

Personal review-and-test workflow for ENG1 Jira tickets. Counterpart to `work-eng1-ticket`. Follow steps in order. Do not skip gates.

The user provides a Jira key. Assume we are operating in a test environment with test data that is safe to modify as needed.

## Prerequisites

- Atlassian MCP available (`plugin-atlassian-atlassian`).
- Prefer GitHub MCP when available and authenticated (server may be `github` or `user-github`). GitHub CLI is acceptable as a fallback if the GitHub MCP is not available.
- Browser tools (`cursor-ide-browser`) for UI test steps.

## Setup (once per invocation)

1. Resolve `cloudId` via `getAccessibleAtlassianResources`.
2. Resolve current user via `atlassianUserInfo` (account id for assignee).
3. Load the provided issue with `getJiraIssue` — include description, comments, issuetype, parent, assignee, status.
4. List subtasks via `searchJiraIssuesUsingJql` (`parent = <parent key>`).
5. Discover Subtask issue type via `getJiraProjectIssueTypesMetadata` + `getJiraIssueTypeMetaWithFields` (do not hardcode field ids). Use that type when creating **Resolve Merge Conflict** or **Rework**.

## Resolve the review target

- **Subtask** (issuetype is a subtask): that issue is the review target. Test instructions are on that subtask.
- **Parent:** find the **Code Review** child. Match by summary **Code Review**. Hard stop if missing.
- Parent key = the target’s parent when the user passed a subtask; otherwise the provided key.

Transition the review target to **In Progress** via `getTransitionsForJiraIssue` → `transitionJiraIssue`. If assignee is not the current user, assign with `editJiraIssue`.

## Workflow checklist

Copy and track progress:

```
ENG1 Review Progress:
- [ ] 1. Resolve Code Review (or provided subtask) and read instructions
- [ ] 2. Mark review target In Progress
- [ ] 3. Stay on assigned repo (move_agent_to_root if needed)
- [ ] 4. Find ticket branch + open PR
- [ ] 5. If stage exists: fetch, checkout, merge ticket branch (do not push)
- [ ] 6. Run test instructions (automate; prompt for manual/login steps)
- [ ] 7a. Fail → PR comments + new Rework; Code Review → To Do
- [ ] 7b. Pass → comment success; Code Review → Done; push stage if merged; ask to merge PR
- [ ] 8. If other repos: comment this repo’s portion + handoff prompt
```

### 1–2. Ticket prep

Read the full description and comments on the review target (and parent if useful) before testing.

### 3. Repo

Stay in the repo this agent is assigned to. If the workspace is not that repo, locate the local clone and call `move_agent_to_root` before any git work.

If the ticket targets multiple repos, focus on the assigned repo only (see step 8).

### 4. Branch and PR

- Ticket branch: prefer the parent key in **lowercase** (e.g. `ENG1-123` → `eng1-123`). Also search remotes for branches containing the key.
- Open PR: GitHub MCP `search_pull_requests` scoped to this repo (title/body/head contain the ticket key). Fallback: `gh pr list`.
- Derive `owner` and `repo` from `git remote get-url origin`.

### 5. Stage merge (only if `stage` exists)

Detect with `git ls-remote --heads origin stage` (or a local `stage` ref). If there is no stage branch, skip to step 6.

If present:

1. Fetch and checkout `stage`.
2. Fast-forward `stage` from remote so it is up to date.
3. Merge the ticket branch into `stage`.
4. **Do not push yet.**

**Merge conflict — hard stop:**

1. Abort the merge so the working tree is clean.
2. Move **Code Review** back to **To Do**.
3. Always `createJiraIssue` a **new** subtask on the parent with summary **Resolve Merge Conflict**. Description must explain this round’s conflict to the developer (conflicting files, both sides).
4. Stop. Do not continue testing.

It is acceptable to create a new **Resolve Merge Conflict** even if one already exists, as long as existing tasks are for a previous issue. There can be multiple rounds of merge conflict resolution for each parent ticket. Do not edit or reuse older **Resolve Merge Conflict** subtasks.

### 6. Run tests

Run through the test instructions on the review target.

- Automate as much as possible, including browser actions.
- Prompt the user to provide logins or complete any other steps that must be done manually. Wait before continuing.
- Assume we are operating in a test environment with test data that is safe to modify as needed.

### 7a. Tests fail

1. Add comments to the pull request. Target specific lines of code if possible (`pull_request_review_write` method `create` with no event → `add_comment_to_pending_review` with `subjectType: LINE` → `submit_pending` with `REQUEST_CHANGES`). Otherwise a generic PR review comment is OK (`COMMENT` / `REQUEST_CHANGES` with a body).
2. Always create a **new Rework** subtask on the parent (`createJiraIssue`). Description must refer back to the comments on the pull request (include the PR URL).
3. Move the **Code Review** subtask back to **To Do**.

It is acceptable to create a new **Rework** even if one already exists, as long as existing tasks are for a previous issue. There can be multiple rounds of rework for each parent ticket. Do not edit or reuse older **Rework** subtasks.

If there is no open PR, comment the failures on the **Code Review** subtask instead, then still create **Rework** and move **Code Review** to **To Do**.

### 7b. Tests pass

1. Add a comment to the **Code Review** subtask describing the success (`addCommentToJiraIssue`).
2. Mark the subtask as **Done**.
3. Push the `stage` branch if the code was merged into it.
4. Ask the user if they want to merge the pull request or not. Merge only if they explicitly say yes (`merge_pull_request` / `gh pr merge`).

### 8. Multiple repos

If the ticket targets multiple repos, focus on the repo the agent was assigned to, comment on that portion of the tests, and then provide the user with a prompt to provide an agent running on the other repo to complete testing.

The handoff prompt must include: the Jira key, this repo’s name, what was already verified (or failed), and the remaining test instructions for the other repo.

## New subtasks (Rework / Resolve Merge Conflict)

| Summary | When | Notes |
|---------|------|--------|
| **Resolve Merge Conflict** | Stage merge conflict | New subtask every round. Explain the conflict to the developer. |
| **Rework** | Tests fail | New subtask every round. Refer to the PR comments. |

Always create a **new** issue for the current review round. Earlier **Rework** / **Resolve Merge Conflict** subtasks belong to previous issues or rounds.

## Tool mapping

| Area | Tools |
|------|--------|
| Jira | `atlassianUserInfo`, `getAccessibleAtlassianResources`, `getJiraIssue`, `searchJiraIssuesUsingJql`, `editJiraIssue`, `getTransitionsForJiraIssue`, `transitionJiraIssue`, `createJiraIssue`, `addCommentToJiraIssue`, `getJiraProjectIssueTypesMetadata`, `getJiraIssueTypeMetaWithFields` |
| Workspace | `move_agent_to_root` when the assigned repo is not the current workspace |
| Git | fetch, checkout `stage`, merge ticket branch, abort on conflict, push `stage` only after tests pass |
| GitHub | Prefer MCP `search_pull_requests`, `pull_request_read`, `pull_request_review_write`, `add_comment_to_pending_review`, `merge_pull_request`; fallback `gh` |
| Tests | Shell/HTTP where sufficient; `cursor-ide-browser` for UI steps; prompt the user for logins and other manual steps |

## Rules

- Do not skip the In Progress transition on the review target.
- Do not push `stage` until tests pass. Never push a conflicted merge.
- Merge conflict: hard stop — abort, **Code Review** → **To Do**, new **Resolve Merge Conflict**, stop.
- Tests fail: PR comments, new **Rework**, **Code Review** → **To Do**.
- Tests pass: comment success on **Code Review**, mark **Done**, push `stage` if merged, then ask whether to merge the PR.
- Always create a new **Resolve Merge Conflict** or **Rework** for the current round, even if the parent already has one. Do not reuse older tasks.
- Stay on the assigned repo. For multi-repo tickets, hand off remaining tests with a copy-paste prompt.
- Prefer GitHub MCP; GitHub CLI is acceptable as a fallback if the GitHub MCP is not available.
