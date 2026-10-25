# Zola → Cloudflare Pages

GitHub Action that checks, builds and (optionally) deploys a [Zola](https://www.getzola.org) site to [Cloudflare Pages](https://pages.cloudflare.com).

It runs, in order: `pre-build-command` (optional) → install Zola → `zola check` → `zola build` → `wrangler pages deploy public`.

## Usage

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  site:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: OWNER/zola-cloudflare-pages-action@v1
        with:
          api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

> Replace `OWNER/zola-cloudflare-pages-action` with the repository where this action is published.

That is all. By default the action:

- builds and checks the site on every run, including pull requests;
- deploys only on pushes, schedules and manual runs on the default branch (never on pull requests);
- uses the repository name as the Cloudflare Pages project and the default branch as the Pages branch.

## Requirements

1. A **Direct Upload** project in Cloudflare Pages, named like your repository (or set `project-name`).
2. A Cloudflare API token with **Cloudflare Pages – Edit** permission, stored as the repository secret `CLOUDFLARE_API_TOKEN`.
3. An `ubuntu-latest` runner (x86_64 Linux); Zola is downloaded from the official releases.

## Inputs

| Input | Default | Description |
|---|---|---|
| `api-token` | – | Cloudflare API token. Required to deploy. |
| `account-id` | – | Cloudflare Account ID. Only needed if the token has access to more than one account. |
| `project-name` | repository name | Cloudflare Pages project name. |
| `branch` | default branch | Pages branch. The project's production branch publishes to production; any other branch creates a preview. |
| `deploy` | `auto` | `auto` deploys on the default branch (never on pull requests). `true` / `false` force it. |
| `zola-version` | `0.21.0` | Zola version (without the `v`). |
| `working-directory` | `.` | Directory containing `config.toml`. |
| `check-links` | `true` | Run `zola check --skip-external-links`. |
| `pre-build-command` | – | Shell command run before the build (lint, tests, data validation). If it fails, the action fails. |

## Outputs

| Output | Description |
|---|---|
| `deployment-url` | Deployment URL (empty if `deploy=false`). |

## Examples

**Validate data before publishing** (used by the site this action came from):

```yaml
with:
  pre-build-command: |
    cargo clippy --locked --manifest-path tools/profile-lint/Cargo.toml -- -D warnings
    cargo run --locked --release --manifest-path tools/profile-lint/Cargo.toml
```

If the command needs a toolchain (Rust, Node…), install it in a previous step.

**Weekly rebuild** (useful when content depends on the build date):

```yaml
on:
  schedule:
    - cron: "0 9 * * 1"
```

**Protected environment and deployment URL**:

```yaml
jobs:
  site:
    environment: cloudflare-pages
    steps:
      - uses: actions/checkout@v4
      - id: site
        uses: OWNER/zola-cloudflare-pages-action@v1
        with: { ... }
      - run: echo "${{ steps.site.outputs.deployment-url }}"
```

## Notes

- Build and deploy happen in the same job, so what gets published is exactly what was validated.
- Recommended: set `concurrency` in your workflow to avoid two deploys running at once.
- Input values are passed through environment variables, never interpolated into the script.
- Secrets are not available on pull requests from forks; that is fine, since pull requests never deploy with `deploy: auto`.

## Releasing

```bash
git tag v1.0.0 && git tag -f v1 && git push origin v1.0.0 v1 --force
```

Users reference `@v1` (moving tag) or `@v1.0.0` (pinned).

## License

MIT
