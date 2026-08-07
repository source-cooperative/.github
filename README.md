# .github

Org-wide defaults and [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for Source Cooperative.

## Claude PR Review

[`claude-pr-review.yml`](.github/workflows/claude-pr-review.yml) has Claude
review every pull request for correctness and security issues, plus an
over-engineering pass via the [ponytail](https://github.com/DietrichGebert/ponytail)
review skill. Findings land in one sticky PR comment that updates in place on
every push.

Add it to a repo by picking the "Claude PR Review" starter workflow under
**Actions → New workflow**, or by copying this in:

```yaml
# .github/workflows/claude-review.yml
name: Claude Auto Review

on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths-ignore:
      - '.github/workflows/**'

jobs:
  review:
    uses: source-cooperative/.github/.github/workflows/claude-pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

That is the whole caller. Three things have to stay in it and cannot move into
the shared workflow:

- **`on:`** — GitHub requires the trigger to live in the calling repo. The
  `paths-ignore` guard matters: the action can't validate or run against
  modified workflow files and fails with "401 Unauthorized - Workflow
  validation failed".
- **`permissions:`** — a called workflow can only maintain or reduce the
  caller's token permissions, never elevate them. Granting them only in the
  shared workflow would be a no-op wherever the repo's default token is
  read-only.
- **`secrets:`** — mapped explicitly rather than `secrets: inherit`, so the
  review job receives only this token instead of every secret the repo holds.

### Inputs

All optional, passed under `with:`:

- `ponytail` (default `true`) — set `false` to drop the over-engineering pass.
- `plugins` / `plugin_marketplaces` — install your own Claude Code plugins
  (newline-separated `name@marketplace` and marketplace `.git` URLs).
  Caller-supplied marketplaces install unpinned from their default branch; only
  the built-in ponytail install is pinned to an exact commit.
- `extra_instructions` — text appended to the review prompt, e.g. to direct
  Claude to use a custom plugin's skill.
- `show_cost` (default `true`) — appends the estimated review cost
  (API-equivalent), duration, and turn count to the review comment.
- `model` — model ID for the review (e.g. `claude-opus-4-8`). Empty uses Claude
  Code's default. With subscription auth, bigger models consume the plan's
  usage quota faster.

### Creating the `CLAUDE_CODE_OAUTH_TOKEN` secret

1. On a machine with [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
   installed and logged in with a Claude subscription, run `claude setup-token`.
2. Approve the OAuth flow and copy the long-lived token (starts with
   `sk-ant-oat01-`).
3. Save it as an Actions secret named `CLAUDE_CODE_OAUTH_TOKEN` — at the org
   level (Settings → Secrets and variables → Actions → New organization secret)
   so every repo can map it, or per-repo. The token bills the subscription of
   whoever ran `setup-token`, so prefer a service/bot account for the org
   secret.

### Pinning

The snippet tracks `@main` so fixes propagate without touching every caller.
Pin a tag or commit SHA instead if a repo needs the review to be reproducible.

### Upstream

Adapted from [`developmentseed/.github`](https://github.com/developmentseed/.github),
which runs the same workflow. Keep the diff against upstream small so
improvements there stay cheap to pull in.
