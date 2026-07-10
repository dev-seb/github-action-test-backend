# Linear Release Sync

Composite action that, when a GitHub Release is published, moves every Linear
issue included in that release to the configured **Done** state and adds a
`🚀 Released in <tag>` comment.

It is a **pure-bash** composite action (see [`action.yml`](./action.yml)) with
no dependencies to install: it uses `gh` for the GitHub API, `curl` for
Linear's GraphQL endpoint, and `jq` — all preinstalled on GitHub runners.

## Inputs

| Input                  | Required | Default | Description                                                                                    |
| ---------------------- | -------- | ------- | ---------------------------------------------------------------------------------------------- |
| `github-token`         | yes      | —       | Token with `contents:read` and `pull-requests:read`. The default `GITHUB_TOKEN` is enough.     |
| `release-tag`          | yes      | —       | The published release tag, e.g. `v1.8.2`.                                                      |
| `linear-api-key`       | yes      | —       | Linear API key (`LINEAR_API_KEY` secret).                                                      |
| `linear-done-state-id` | yes      | —       | Target Linear workflow-state id (`LINEAR_ENGINEERING_DONE_STATE_ID` variable). Never hard-coded. |

## Required secrets

- **`LINEAR_API_KEY`** — a Linear API key allowed to update issues and create
  comments (_Linear → Settings → API → Personal API keys_).
- **`LINEAR_ENGINEERING_DONE_STATE_ID`** — the id of the workflow state issues are moved to.
  Fetch it once:

  ```graphql
  query {
    workflowStates(filter: { name: { eq: "Done" } }) {
      nodes {
        id
        name
        team {
          key
        }
      }
    }
  }
  ```

  State ids are team-specific — use the id for the team this repo tracks.

## How PRs are discovered

The action checks out the repository with full history and tags
(`fetch-depth: 0`), then:

1. finds the **previous tag** (`git tag --sort=-creatordate`, first one that
   isn't the release tag; the whole history is used for the first release);
2. lists commits in `previous..current` (`git log`);
3. for each commit, calls GitHub's
   `listPullRequestsAssociatedWithCommit` to find the merged PR(s).

## How Linear issues are resolved

1. Issue keys (`ABC-123`) are matched from each PR's **title**, **body** and
   **branch name** and de-duplicated. Keys that don't correspond to a real
   Linear issue (e.g. `UTF-8`) simply resolve to nothing and are skipped.
2. Each key is resolved with Linear's `issue(id: "ABC-123")` query — the `id`
   argument accepts the human identifier.
3. The issue is moved to `LINEAR_ENGINEERING_DONE_STATE_ID` (`issueUpdate`) and gets a
   `commentCreate` comment.

## Idempotency & error handling

- Safe to re-run: moving an issue already in Done is a no-op, and the comment
  is only added when one for that tag isn't already present.
- A single issue that fails is logged as a warning and **skipped**; the rest
  keep processing.
- The job **fails** only on invalid configuration/authentication (missing
  secrets, or a `LINEAR_API_KEY` that can't authenticate — checked up front
  with a `viewer` query).

## Note on scale

PRs are looked up one API call per commit, which is the simplest approach and
fine for typical releases. For releases with a very large number of commits,
batching PR lookups via the GitHub GraphQL API would reduce the request count.

## Standalone usage

Wired into [`release.yml`](../../workflows/release.yml), but usable on its own
from a `release: [published]` trigger:

```yaml
on:
  release:
    types: [published]

jobs:
  sync-linear:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
    steps:
      - uses: syrocolab/workflows/.github/actions/linear-release-sync@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          release-tag: ${{ github.event.release.tag_name }}
          linear-api-key: ${{ secrets.LINEAR_API_KEY }}
          linear-done-state-id: ${{ vars.LINEAR_ENGINEERING_DONE_STATE_ID }}
```
