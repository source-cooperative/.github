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

## Cloudflare Preview

[`cloudflare-preview.yml`](.github/workflows/cloudflare-preview.yml) builds a
static site, publishes it as a per-PR Worker on a `*.workers.dev` URL, comments
the link on the pull request, and **deletes that Worker when the PR closes**.

Add it by picking the "Cloudflare Preview" starter workflow under **Actions →
New workflow**, or by copying this in:

```yaml
# .github/workflows/cloudflare-preview.yml
name: Cloudflare Preview

on:
  pull_request:
    # `closed` is required — it is what triggers teardown. Without it, preview
    # Workers are never deleted.
    types: [opened, synchronize, reopened, closed]

jobs:
  preview:
    uses: source-cooperative/.github/.github/workflows/cloudflare-preview.yml@main
    with:
      output_dir: dist          # `build` for the SvelteKit viewers
    permissions:
      contents: read
      pull-requests: write
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Deploy and teardown live in one workflow deliberately. The per-PR Worker name
is a contract between them; split across two workflows the convention gets
defined twice, and when it drifts the failure is silent — teardown deletes a
name that does not exist, exits clean, and orphaned Workers accumulate.

### Inputs

All optional, passed under `with:`:

- `output_dir` (default `dist`) — directory the build writes static assets to.
  The SvelteKit viewers use `build`.
- `build_command` — defaults to `<package manager> run build`. The package
  manager is detected from the lockfile, so callers with either `pnpm-lock.yaml`
  or `package-lock.json` need not declare anything.
- `node_version` (default `22`).
- `compatibility_date` (default `2026-08-01`) — Workers runtime compatibility
  date for the preview Worker.

### Base paths

Previews are served from the root of a `workers.dev` hostname, not a repo
subpath. A site built for GitHub Pages with a hardcoded `base` of `/<repo>/`
will deploy but load nothing — every asset 404s. Build root-relative for
previews; if the repo also deploys to Pages, gate the subpath on an env var set
only by the Pages workflow (see `zarr-viewer`'s `BASE_PATH`).

### Cloudflare setup

**None per repo.** `wrangler deploy` creates the Worker on first run; there is
nothing to pre-create in the dashboard. That is the advantage over Pages git
integration, where connecting each repo is an interactive authorization that
cannot be scripted — and which gives no teardown.

Account-level, once:

1. A registered `workers.dev` subdomain.
2. `CLOUDFLARE_API_TOKEN` with **Workers Scripts: Edit** — covers both the
   deploy and the delete on teardown.
3. `CLOUDFLARE_ACCOUNT_ID`.

Note that per-PR Workers count against the account's Worker limit; teardown is
what keeps that bounded.

### Blast radius

Think before making these org secrets visible to every repository.
Cloudflare API token permissions are granted at the **account** level —
`Workers Scripts: Edit` applies to every Worker in the account, and there is no
per-script or name-prefix scoping. The same token that creates
`zarr-viewer-pr-77` can delete a production Worker.

Reduce it, in rough order of value:

1. **Use a separate Cloudflare account for previews.** The only real fix, since
   the token cannot be scoped below account level. Production Workers then live
   somewhere CI cannot reach at all.
2. **Scope the org secret to selected repositories** rather than all — only the
   repos that actually deploy previews.
3. **Never put the credentials in a workflow- or job-level `env:` block.** That
   exposes them to `install` and `build`, where any postinstall script or
   transitive dependency can read them. This workflow sets them on the two
   wrangler steps only.

GitHub does not pass secrets to `pull_request` runs from forks, so an outside
contributor's PR cannot read them. The exposure is to anyone with write access
to a repo that can see the secret, and to that repo's dependency tree.
