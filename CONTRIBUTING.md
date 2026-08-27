# Contributing

Pull requests are welcome, especially ones that make the pipeline smaller.

## Ground rules

- All logic lives in `Taskfile.yml`. Workflows install tools and run one task, nothing
  more. No shell scripts.
- No dependencies beyond `rsync`, `aws` (AWS CLI v2), `gpg`, `shasum`, `xz`, `curl` and
  `task`. Network endpoints are dante, R2, the public domain and healthchecks.io.
- Storage is the bill. The tree is 133 GB and the pipeline refuses to run past 200 GB
  upstream; if a change adds storage or Class A operations, say by how much in the PR.
- Objects sit at the bucket root under CTAN's own paths, so every CTAN path is a URL path.
  `.state/` is the one reserved prefix.
- The zone is configured by hand and the pipeline never calls the Cloudflare API. Section 6
  of `docs/reference.md` has the rules it wants.
- Actions are pinned to a full commit SHA with the version in a trailing comment;
  Dependabot keeps them current. GitHub rejects a workflow that references a tag.

## Checking a change

```sh
task --dry --force sync                             # render every command, no network
task lint                                           # the cron minute agrees with CRON_MINUTE
task smoke RUN=<dir> URL=file:///<dir>              # read-back check, offline
```

The `check` workflow runs the first two on every pull request. `CLAUDE.md` lists the rest of
the offline checks; they read a `fixtures/` tree (git-excluded) you supply yourself: a real
dante listing and a signed `tlpkg/`.

`publish`, `checkpoint`, `delete` and `rebuild` need real R2 credentials and have no mock;
test them on your own fork against a scratch bucket (see "Want your own?" in the README).

## Commits

`<type>(<scope>): <summary>` in the imperative, under 75 characters. Types: feat, fix,
refactor, docs, test, chore, ci. One PR per change; PRs are squash merged.
