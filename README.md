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

#### Current consumers

- [`morgado-ricardo/homeassistant`](https://github.com/morgado-ricardo/homeassistant)
- [`morgado-ricardo/homelab`](https://github.com/morgado-ricardo/homelab)
- [`morgado-ricardo/cv`](https://github.com/morgado-ricardo/cv)

## Repository setup

This repo is private, so **Settings → Actions → General → Access** must be set
to *"Accessible from repositories owned by the morgado-ricardo user"*.
Without it, callers fail with "workflow not found" — which looks exactly like
the file being missing.
