# ci-shared

Reusable GitHub Actions workflows shared across `morgado-ricardo`'s repositories.
Change something here and every consuming repo picks it up on its next run — no
commit needed in the consumers.

## Workflows

### `.github/workflows/claude-review.yml` — Claude code review

AI code review with
[`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action),
authenticated with a Claude Pro/Max subscription. It replaces the earlier
Gemini review (`jgunnink/gemini-review-bot`). That bot sent the diff to Gemini
Flash in one call and never saw the rest of the repository, so its reviews
were shallow. Claude Code works as an agent. It checks out the PR head with
full history, reads the code around each change, and posts findings as inline
comments plus one summary comment.

What the shared workflow adds on top of the action:

- **One model for every repo.** Change the `model` default here and every
  consumer uses it on its next run.
- **A review prompt tuned for signal.** It asks for bugs, security problems
  and broken edge cases, with no praise and no nits unless they hide a real
  problem. Each repo's own `CLAUDE.md` / `REVIEW.md` adds its own guidance.
- **A short tool allowlist.** The agent can read the checkout, run
  `gh pr view|diff|comment`, and post inline comments. It can do nothing else.
- **`continue-on-error`**, with a note in the run summary when the review
  fails. The review is advisory, so a usage limit or an expired token must
  never block a merge.
- **A fork-PR guard.** Pull requests from forks don't get repository secrets.
  The workflow skips quietly instead of failing on something the contributor
  cannot fix.
- **An author check on `/claude-review`.** Only `OWNER`, `MEMBER` and
  `COLLABORATOR` comments trigger a review. A comment-triggered run has the
  secret and checks out the PR head, and anyone can comment on a public repo.
- **A concurrency group**, so a new push cancels the review in flight instead
  of spending usage on a superseded diff.

| Input | Default | Purpose |
| --- | --- | --- |
| `model` | `claude-opus-5-5` | Review model. **Change it here to change it everywhere.** Switch to a Sonnet model if reviews use too much of the subscription. |
| `max_turns` | `30` | Upper bound on agent turns per review. |

Required secret: `claude_code_oauth_token`. Run `claude setup-token` locally
once, then store the result as `CLAUDE_CODE_OAUTH_TOKEN` in each consumer.

#### Consuming it

Add this as `.github/workflows/claude-review.yml` in the consumer. It is the
same in every repo. Only the triggers live in the consumer, because GitHub
does not let a called workflow declare its own.

```yaml
name: Claude code review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
  issue_comment:
    types: [created]

jobs:
  review:
    # Gate at the caller: without this GitHub spawns a run for every comment
    # on every issue before discovering the called job is skipped. The shared
    # workflow repeats the condition as a safety net.
    if: >-
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' &&
       github.event.issue.pull_request &&
       startsWith(github.event.comment.body, '/claude-review'))
    uses: morgado-ricardo/ci-shared/.github/workflows/claude-review.yml@main
    # Granted here, not in the shared workflow: a called workflow can only
    # narrow the caller's permissions, never widen them. issues: write is for
    # the summary comment (PR comments go through the issues API).
    permissions:
      contents: read
      pull-requests: write
      issues: write
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

Two things the consumer keeps for itself:

- `CLAUDE.md` or `REVIEW.md` at the repository root, with repo-specific
  review guidance. This replaces the old `.github/gemini-review.yml`.
  Delete that file when migrating.
- The `CLAUDE_CODE_OAUTH_TOKEN` secret. Secrets never cross repositories.

Comment `/claude-review` on a pull request to re-run a review.
`issue_comment` is not a pull-request event, so GitHub runs the caller
workflow **from the default branch**. The shared workflow still checks out the
PR head for the review itself, but a comment will not exercise changes to the
caller file that are still unmerged.

#### Why `@main` and not a pinned SHA

Deliberate. The mutable reference is the entire point: a change here has to
reach every consumer without a commit in each of them, which is the problem
this repo exists to solve. Pinning consumers to a SHA or a release tag would
reintroduce exactly the per-repo update work that made `gemini-flash-latest`
go unfixed in three places at once back in the Gemini days.

The usual argument for SHA-pinning is supply-chain risk from a third-party
action you do not control. That does not apply here: this repo is public to
read but writable only by the account that owns its consumers, and the only
third-party action it calls (`anthropics/claude-code-action@v1`) is referenced
from here, so its version is itself centrally controlled. The blast radius of
a bad commit is an advisory review job that is already `continue-on-error`.

#### Current consumers

| Consumer | Visibility |
| --- | --- |
| [`morgado-ricardo/homeassistant`](https://github.com/morgado-ricardo/homeassistant) | private |
| [`morgado-ricardo/homelab`](https://github.com/morgado-ricardo/homelab) | private |
| [`morgado-ricardo/cv`](https://github.com/morgado-ricardo/cv) | private |
| [`morgado-ricardo/ev-plug-charging`](https://github.com/morgado-ricardo/ev-plug-charging) | **public** |

## Repository setup

**This repo must stay public.** Not for the usual reasons — nothing here needs
publishing — but because a **public** repository cannot call a reusable
workflow from a **private** one. GitHub's "share a private repo's workflows
with the rest of the account" setting only reaches private consumers.

That is not a hypothetical: it was found the hard way. While this repo was
private, `ev-plug-charging` (public) was wired up exactly like the other three
and every run failed **in 0 seconds with 0 jobs** — GitHub could not resolve
the `uses:` reference, so it failed the run before creating a job. There is no
error message pointing at visibility; the failure looks identical to a typo in
the path or a missing file.

Making it private again silently breaks every public consumer.

Note the old **Settings → Actions → General → Access** requirement is now moot:
a public repo's reusable workflows are callable by anyone, so the setting has
nothing left to grant. It only mattered while this repo was private.
