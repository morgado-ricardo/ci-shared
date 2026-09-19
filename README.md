# ci-shared

Reusable GitHub Actions workflows shared across `morgado-ricardo`'s repositories.
Change something here and every consuming repo picks it up on its next run — no
commit needed in the consumers.

## Workflows

### `.github/workflows/gemini-review.yml` — Gemini code review

AI code review via Google's Gemini API, a free self-hosted replacement for the
sunset Gemini Code Assist bot. Wraps
[`jgunnink/gemini-review-bot`](https://github.com/jgunnink/gemini-review-bot)
and adds what that action doesn't do itself:

- **A pinned GA model** instead of the floating `gemini-flash-latest` alias.
  That alias hot-swaps onto each new Flash release, preview ones included, and
  a just-launched model is exactly the one returning `503` "high demand" while
  Google grows capacity. This was the cause of constant CI failures.
- **A fallback model** on a second attempt. A capacity spike hits one model's
  serving pool at a time, so the other usually answers.
- **A longer retry window.** The action retries a 503 three times over ~40s
  internally, which is nothing next to a demand spike; a wait between the two
  attempts spreads them over minutes.
- **`continue-on-error` on both attempts**, with a run-summary note when
  neither lands. The review is advisory and must never block a merge on a
  provider outage.
- **A fork-PR guard.** Pull requests from forks don't get repository secrets;
  skip quietly rather than fail on something the contributor cannot fix.
- **A concurrency group**, so a new push cancels the review in flight instead
  of spending quota on a superseded diff.

| Input | Default | Purpose |
| --- | --- | --- |
| `model` | `gemini-3.5-flash` | Primary model. **Change it here to change it everywhere.** |
| `fallback_model` | `gemini-3.7-flash` | Tried when the primary is unavailable. |
| `retry_delay_seconds` | `120` | Wait between the two attempts. |

Required secret: `gemini_api_key` (a Google AI Studio key belonging to the
calling repository).

#### Consuming it

Add this as `.github/workflows/gemini-review.yml` in the consumer. It is
identical in every repo — only the triggers live here, because GitHub does not
let a called workflow declare its own.

```yaml
name: Gemini code review

on:
  pull_request:
    types: [opened, synchronize]
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
       startsWith(github.event.comment.body, '/gemini-review'))
    uses: morgado-ricardo/ci-shared/.github/workflows/gemini-review.yml@main
    # Granted here, not in the shared workflow: a called workflow can only
    # narrow the caller's permissions, never widen them.
    permissions:
      contents: read
      pull-requests: write
    secrets:
      gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
```

Two things the consumer keeps for itself:

- `.github/gemini-review.yml` — repo-specific review instructions and ignore
  globs. Do **not** set `model` there; the shared workflow passes it as an
  action input, which overrides the config file.
- The `GEMINI_API_KEY` secret. Secrets never cross repositories.

Re-run a review on demand by commenting `/gemini-review` on a pull request.
Note that `issue_comment` is not a pull-request event, so GitHub runs the
caller workflow **from the default branch**, not from the PR head — a comment
will not exercise caller changes that are still unmerged.

#### Why `@main` and not a pinned SHA

Deliberate. The mutable reference is the entire point: a change here has to
reach every consumer without a commit in each of them, which is the problem
this repo exists to solve. Pinning consumers to a SHA or a release tag would
reintroduce exactly the per-repo update work that made `gemini-flash-latest`
go unfixed in three places at once.

The usual argument for SHA-pinning is supply-chain risk from a third-party
action you do not control. That does not apply here: this repo is public to
read but writable only by the account that owns its consumers, and the only
third-party action it calls (`jgunnink/gemini-review-bot@v1`) is referenced
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
