# Shared workflows

Reusable GitHub Actions workflows for martinfrancois's open source repositories. They run the issue claim flow: contributors claim an issue with `/take`, quiet claims are released, and pull requests that collide with someone else's claim are flagged. Each repository keeps three short caller workflows; the logic lives here, so a fix lands everywhere with one release.

The flow exists so contributors don't spend time on pull requests that collide with someone else's work. Each repository's CONTRIBUTING.md explains it to contributors.

## Workflows

### `claim-issue.yml`

Called from an `issue_comment` workflow. A comment with a line that reads `/take` (outside quotes and code) assigns the commenter if the issue has one of the claim labels, nobody holds it, no open pull request in the repository closes or mentions it, and the commenter holds fewer than `max-open-claims` open issues there. Otherwise it replies with the reason. If two claims race, the earlier assignment wins and the later claimer gets a reply. A request in plain words ("can I take this", "can I work on this", "assign me", and the other phrases listed in the workflow) gets one answer per person and issue saying whether `/take` would work; it never assigns anyone. Maintainers (`OWNER`, `MEMBER`, `COLLABORATOR`) have no cap. Bot comments are ignored.

| Input             | Type   | Default                        | Meaning                                               |
| ----------------- | ------ | ------------------------------ | ----------------------------------------------------- |
| `max-open-claims` | number | `3`                            | Open issues one contributor can hold at a time        |
| `claim-labels`    | string | `help wanted,good first issue` | Comma-separated labels that open an issue for `/take` |

Job permissions to grant: `issues: write`, `pull-requests: read`.

### `claim-reminders.yml`

Called from a scheduled workflow. For each open issue with a contributor assignee, it reminds them in the issue `nudge-after-days` after the assignment, their last comment there or the close of their last unmerged pull request for it. If `unassign-after-days` pass after the reminder without a reply or a pull request, it unassigns them with a note. An open, draft or merged pull request from the assignee that references the issue stops the reminders for good. Assignees with write access, the repository owner and bots are never touched.

| Input                 | Type   | Default | Meaning                                                |
| --------------------- | ------ | ------- | ------------------------------------------------------ |
| `nudge-after-days`    | number | `14`    | Days without a pull request before the reminder        |
| `unassign-after-days` | number | `7`     | Days after the reminder before the assignee is removed |

Job permissions to grant: `issues: write`, `pull-requests: read`.

### `pr-claim-check.yml`

Called from a `pull_request_target` workflow. A pull request that closes an issue its author doesn't hold, or that another open pull request already closes, gets one comment (updated on later runs) and the conflict label. When a later run finds nothing, the label goes and the comment says so. A pull request that closes no issue passes. It reads metadata through the API only and never checks out or runs pull request code.

| Input            | Type   | Default          | Meaning                                                                         |
| ---------------- | ------ | ---------------- | ------------------------------------------------------------------------------- |
| `conflict-label` | string | `claim conflict` | Label for a pull request with a claim conflict; it must exist in the repository |

Job permissions to grant: `issues: read`, `pull-requests: write`.

## Calling the workflows

A caller job must grant exactly the permissions listed for the workflow: GitHub refuses to start a run whose caller grants less than the called job requests. Pin the reference to a release commit with the version as a comment, so Renovate can update it:

```yaml
name: Claim issue
on:
  issue_comment:
    types: [created]
permissions: {}
jobs:
  claim:
    if: ${{ !github.event.issue.pull_request && github.event.comment.user.type != 'Bot' }}
    permissions:
      issues: write
      pull-requests: read
    uses: martinfrancois/.github/.github/workflows/claim-issue.yml@<release commit SHA> # v1.0.0
    with:
      max-open-claims: 3
```

```yaml
name: Claim reminders
on:
  schedule:
    - cron: "23 7 * * *"
  workflow_dispatch:
permissions: {}
concurrency:
  group: claim-reminders
  cancel-in-progress: false
jobs:
  remind:
    permissions:
      issues: write
      pull-requests: read
    uses: martinfrancois/.github/.github/workflows/claim-reminders.yml@<release commit SHA> # v1.0.0
    with:
      nudge-after-days: 14
      unassign-after-days: 7
```

```yaml
name: PR claim check
on:
  pull_request_target:
    types: [opened, edited, reopened, synchronize]
permissions: {}
concurrency:
  group: pr-claim-check-${{ github.event.pull_request.number }}
  cancel-in-progress: true
jobs:
  check:
    if: ${{ github.event.pull_request.user.type != 'Bot' && !contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.pull_request.author_association) }}
    permissions:
      issues: read
      pull-requests: write
    uses: martinfrancois/.github/.github/workflows/pr-claim-check.yml@<release commit SHA> # v1.0.0
```

The repository also needs the `claim conflict` label (or the label passed as `conflict-label`) and the claim labels.

## Security

- No `${{ }}` expression carries comment, title or body text into a script. The scripts read the event payload as data, and there are no `run:` steps.
- Bot replies repeat only logins and numbers, never comment text. Other people's logins appear in backticks, so they aren't pinged.
- The markers that drive the reminder, hint and pull request comments only count when a bot wrote them, so nobody can fake a reminder to get someone unassigned early.
- `pull_request_target` runs the caller from the default branch, so a pull request can't change what runs, and nothing here checks out pull request code.
- Comments made with `GITHUB_TOKEN` don't trigger workflows, and bot comments are skipped, so the workflows can't loop.

## Releases

Every change goes through a pull request. A release is a `vX.Y.Z` tag with a GitHub release; callers move to it through Renovate's pull requests. A change to inputs, replies or behavior that callers rely on is a new major version.

## License

[MIT](LICENSE)
