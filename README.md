# Create pull request

A composite GitHub Action that commits the changes made to tracked files in the
working tree to a new branch, pushes it and opens a pull request. It uses only
`git` and the `gh` CLI that are preinstalled on GitHub-hosted runners.

## Behaviour

- **Idempotent.** If a pull request for the branch already exists in any state
  (open, merged or declined), the action does nothing. Because of this, include
  something unique in the branch name, such as the new version number.
- **Tracked files only.** The action runs `git add --update`, so only
  modifications to files that already exist in the repository are committed.
  New, untracked files are never added.
- **No changes, no PR.** If the working tree has no changes to tracked files,
  the action logs a warning and exits successfully.
- **Leftover branches are replaced.** If the branch exists but has no pull
  request (for example from a half-finished earlier run), it is rebuilt from the
  current checkout and force-pushed with `--force-with-lease`, so a push by
  someone else in the meantime is never overwritten.
- **Clean failure.** If the branch was pushed but the pull request could not be
  created, the branch is deleted again.
- **Token handling.** The token is supplied to git through a credential helper
  that reads it from the environment. It never appears in a command line, a URL
  or the log.
- **Labels are best effort.** A label that cannot be added (for example because
  it does not exist) only produces a warning and does not undo the pull request.

## Requirements

The calling job must:

1. Check out the repository, preferably with `persist-credentials: false`.
2. Modify the files it wants in the pull request before calling this action.
3. Grant these permissions:

   ```yaml
   permissions:
     contents: write
     pull-requests: write
   ```

4. If the token is the default `GITHUB_TOKEN`, the repository setting
   **Settings → Actions → General → Allow GitHub Actions to create and approve
   pull requests** must be enabled.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github-token` | yes | | Token that can push a branch and create a pull request in this repository. |
| `branch` | yes | | Name of the branch to create. Must be unique per change (see above). Validated against `^[A-Za-z0-9][A-Za-z0-9._/+-]*$` and `git check-ref-format`. |
| `title` | yes | | Title of the pull request. |
| `body` | no | `""` | Body of the pull request. |
| `commit-message` | no | the title | Commit message. |
| `base-branch` | no | `github.ref_name` | Branch the pull request targets. |
| `labels` | no | `""` | Comma-separated labels to add. |
| `commit-user-name` | no | `github-actions[bot]` | Login of the account the commit is attributed to (`<app-slug>[bot]` for a GitHub App). |
| `commit-user-id` | no | `41898282` | Numeric user ID of that account. The default is the ID of `github-actions[bot]` on github.com. Look it up with `gh api users/<login> --jq .id` (for an app: `gh api "users/<app-slug>[bot]" --jq .id`). It is used to build the commit email `<id>+<login>@users.noreply.github.com`, which links the commit to the account. |

## Outputs

| Output | Description |
|---|---|
| `created` | `true` if this run created a pull request, otherwise `false`. |
| `pull-request-url` | URL of the created pull request (empty if none was created). |

## Usage

```yaml
name: Bump something

on:
  schedule:
    - cron: "0 6 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  bump:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Change a file
        run: echo "2.0.0" > VERSION

      - name: Create pull request
        id: pr
        uses: OWNER/REPO/create-pull-request@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          branch: chore/bump/2.0.0
          title: "chore: bump to 2.0.0"
          body: "Bumps the version to 2.0.0."
          labels: dependencies

      - name: Show result
        run: echo "created=${{ steps.pr.outputs.created }} url=${{ steps.pr.outputs.pull-request-url }}"
```

Replace `OWNER/REPO` with the repository that hosts this action and pin a tag or
commit SHA. A relative path such as `./create-pull-request` only works when the
workflow runs in the same repository as the action.

## Notes

- Pull requests created with the default `GITHUB_TOKEN` do not trigger other
  workflows (for example your CI checks). To have checks run on the pull
  request, pass a personal access token or a GitHub App token as `github-token`.
- If you use a GitHub App token, set `commit-user-name` to `<app-slug>[bot]` and
  `commit-user-id` to that bot's user ID so commits are attributed to the app.
