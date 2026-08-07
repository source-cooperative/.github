# .github

Org-wide defaults and shared GitHub Actions workflows for Source Cooperative.

## Reusable workflow: Claude PR review

`.github/workflows/claude-pr-review.yml` runs the Claude review on a pull
request and posts it as a single sticky comment that updates in place on every
push. Repos call it instead of keeping their own copy, so a change to the
prompt or the action version lands everywhere at once.

Add this as `.github/workflows/claude-review.yml` in the calling repo:

```yaml
name: Claude Auto Review
on:
  pull_request:
    types: [opened, synchronize, reopened]
    # The action can't validate or run against modified workflow files and
    # fails with "401 Unauthorized - Workflow validation failed".
    paths-ignore:
      - '.github/workflows/**'

jobs:
  review:
    uses: source-cooperative/.github/.github/workflows/claude-pr-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    secrets: inherit
```

That is the whole caller. Three things have to stay in it and cannot move into
the shared workflow:

- **`on:`** — GitHub requires the trigger to live in the calling repo.
- **`permissions:`** — a called workflow can only maintain or reduce the
  caller's token permissions, never elevate them. Granting them in the shared
  workflow would be a no-op wherever the repo's default token is read-only.
- **`secrets: inherit`** — passes `CLAUDE_CODE_OAUTH_TOKEN` through. The
  shared workflow declares it `required`, so a caller that omits this fails
  immediately with a clear message rather than a vague auth error.

`CLAUDE_CODE_OAUTH_TOKEN` must be available to the calling repo, either as an
org secret or a repo secret.

### Pinning

The snippet tracks `@main` so fixes propagate without touching every caller.
Pin a tag or commit SHA instead if a repo needs the review to be reproducible.
