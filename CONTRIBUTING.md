# Contributing

Pull requests are welcome, especially ones that make the pipeline smaller.

## Ground rules

- All logic lives in `Taskfile.yml`. Workflows install tools and run one task, nothing
  more. No shell scripts.
- No dependencies beyond `rsync`, `aws` (AWS CLI v2), `gpg`, `shasum`, `curl` and `task`.
- The mirror must stay inside the R2 free tier (10 GB, 1M Class A ops a month). If a
  change adds storage or upload operations, say by how much in the PR.
- Objects stay under `systems/texlive/tlnet/`; every user's `tlmgr` config carries that
  path.
- Actions are pinned to a full commit SHA with the version in a trailing comment;
  Dependabot keeps them current. GitHub rejects a workflow that references a tag.

## Checking a change

```sh
task --dry sync                                   # render the pipeline, no network
task guard STAGING=<dir> LIMIT_MB=<n>             # size check against any directory
task smoke URL=file:///<dir> STAGING=<dir>        # read-back check, offline
```

The `check` workflow runs `task --dry sync` on every pull request.

`publish` needs real R2 credentials and has no mock; test it on your own fork (see
"Want your own?" in the README).

## Commits

`<type>(<scope>): <summary>` in the imperative, under 75 characters. Types: feat, fix,
refactor, docs, test, chore, ci. One PR per change; PRs are squash merged.
