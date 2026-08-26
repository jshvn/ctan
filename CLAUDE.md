# ctan

A daily mirror of `CTAN/systems/texlive/tlnet` (the directory `tlmgr` installs from) on
Cloudflare R2, served at `https://ctan.ijosh.com/systems/texlive/tlnet/`. About 17,400
files and 6.8 GB; the largest file is ~145 MB.

Everything is in four files:

- `Taskfile.yml`: the whole pipeline. `task sync` runs
  `fetch -> verify -> guard -> publish -> smoke -> report -> ping -> page`.
- `.github/workflows/sync.yml`: installs `task` and runs `task sync` daily at 03:30 UTC.
- `.github/workflows/check.yml`: runs `task --dry sync` on pull requests.
- `site/index.html`: the landing page at `https://ctan.ijosh.com/`. It repeats the README's
  prose, so a README edit is usually a page edit too.

`README.md` is for users. Operational detail belongs here and in Taskfile comments.

## Constraints

- Zero running cost: R2 free tier (10 GB-month, 1M Class A ops) and free GitHub Actions.
  Recompute any change that adds storage or uploads against the 6.8 GB baseline.
- No shell scripts. Logic lives in `Taskfile.yml`; workflows install tools and run one task.
- Tools are exactly `rsync`, `aws` (CLI v2), `gpg` (for `gpgv`), `shasum`, `curl`, `task`.
  Network endpoints are exactly the CTAN master, R2, the public domain and healthchecks.io.
- Objects stay under `systems/texlive/tlnet/`; every user's `tlmgr` config carries that path.
- Secrets are exactly four: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
  `HEALTHCHECK_URL`. The workflow maps the first three onto the AWS CLI's env vars.

## Must knows

Each of these is a bug that has happened or a bill that would. Do not undo them.

- **No versioned containers.** `tlmgr` requests `archive/foo.tar.xz`; upstream stores
  `foo.r123.tar.xz` and symlinks to it. R2 has no symlinks, so `fetch` uses `rsync -L` and
  excludes `*.r[0-9]*.tar.xz`. Storing both doubles storage and breaks the free tier.
- **Never `--size-only`.** Revision bumps overwrite files in place, often at the same size.
  `aws s3 sync` uploads when size differs or the local mtime is newer; `rsync -t` keeps
  upstream mtimes, so same-size bumps are caught.
- **Never `aws s3 sync --delete`.** R2 lists `install-tl` after `install-tl.zip.sha512.asc`
  while the CLI walks staging in byte order, so the merge-join emits both an `upload:` and
  a `delete:` for 15 keys and whichever lands last wins. Deletions come from `stale`, a
  byte-sorted `comm` of the bucket listing against staging, fed to `aws s3 rm` (free on
  R2). The 15 keys re-upload every run; that is harmless.
- **Single-part uploads.** `aws.config` sets `multipart_threshold` above the largest file.
  The CLI default is 8 MB, and multipart costs three or more Class A ops per file.
- **`publish` uploads in a fixed order.** `archive/` first, then the full sync (which
  carries the tlpdb), then deletions, so no client reads a tlpdb naming a container the
  bucket lacks.
- **The steps after `smoke` run in a fixed order too.** `report` before `ping` so a
  healthchecks blip cannot lose the summary; `page` last so a broken landing page is one
  email on a fresh mirror; `ping` after `smoke` so a broken domain never looks alive.
- **`index.html` lives at the bucket root**, outside the mirror prefix, so `stale` never
  sees it. It is uploaded with `Cache-Control: no-cache`. A Cloudflare Transform Rule
  rewrites `/` to `/index.html`; without it the root is a 404 and the mirror still works.
- **Do not trust the job log for counts.** GitHub drops lines when `publish` prints
  thousands of `upload:` lines in seconds. `report` counts from `RUN` (`/tmp/tlnet-run`,
  where `fetch` and `publish` `tee` under `pipefail`), not the job log. Judge completeness
  by `smoke`, never by counting.
- **A failed run is the only alert.** `guard` fails before `publish` if staging exceeds
  10 GB; the mirror stays a day stale. healthchecks.io emails when a day passes without `ping`,
  which also catches GitHub disabling the schedule after 60 commit-free days. Do not add
  notification dependencies or upstream-freshness monitoring (tlnet goes quiet for weeks
  before each release).
- `.xz` is not in Cloudflare's default cache list, so there is no stale-edge problem. If a
  cache-everything rule is ever added, `publish` needs a purge step before `smoke`.

## Verification and security

`verify` runs after `fetch` and before anything is uploaded. A tree that fails stays local;
the previous good copy stays live.

1. `texlive.tlpdb.sha512` and the `install-tl*.sha512` files at the tree root (the only
   files the tlpdb's checksums do not reach) are checked with `shasum`, then their `.asc`
   signatures with `gpgv`. The keyring is read as a plain file from the mirror being
   verified, so nothing is imported before it is authenticated.
2. Because the keyring came from the same mirror as the signature, the only real check is
   the pinned fingerprint `TL_KEY`: `VALIDSIG` must end in it. Rotate it only against
   https://www.tug.org/texlive/verify.html. `GOODSIG` is also required because gpgv reports
   an expired or revoked key as `VALIDSIG` with exit 0 (tlmgr accepts that; the mirror is
   stricter). The signing subkey expires 2027-07-13; upstream extends it yearly.
3. No `*.r[0-9]*.tar.xz` survives in `archive/`.
4. Every container the tlpdb names exists and matches its `containerchecksum`,
   `doccontainerchecksum` or `srccontainerchecksum` (source containers are
   `<name>.source.tar.xz`). A tree caught between a tlpdb update and its containers fails.

After `publish`, `smoke` reads `texlive.tlpdb.sha512` back through the public domain and
`cmp`s it with staging. `tlmgr` repeats the signature check on the client.

## Repository guardrails

- Actions are pinned to a full commit SHA with the version in a trailing comment (repo
  setting rejects tags). The allowlist is GitHub-owned plus `go-task/*`.
- Dependabot groups Actions bumps weekly; these are the only routine commits.
- CodeQL default setup scans the workflows. Private vulnerability reporting is on and
  `SECURITY.md` links to it.
- PRs target `main` and are squash merged. `check.yml` must pass.
- Commits: `<type>(<scope>): <summary>`, imperative, under 75 characters.

## Verifying a change

- `task --dry sync` renders the pipeline without touching the network.
- `task guard STAGING=<dir> LIMIT_MB=<n>` exercises the size check against any directory.
- `task smoke URL=file:///<dir> STAGING=<dir>` exercises the read-back check offline.
- `task ping` is a no-op without `HEALTHCHECK_URL`; set it to any `file://` URL.
- `task stale RUN=<dir> STAGING=<dir>` diffs a canned `remote.txt` (one key per line) in
  `<dir>` against `<dir>`'s files and writes `stale.txt`; it must be empty when every key
  exists locally, and it must fail on an empty directory.
- `task report RUN=<dir> STAGING=<dir>` renders the summary to stdout from a canned
  `fetch.txt` (rsync `--stats` block) and `publish.txt` (`aws s3 sync` lines) in `<dir>`.
- `task page` fails at the upload without credentials; run its `sed` by hand to see the
  date filled in.
- `task verify` needs a full `staging/`; with only `tlpkg/` and the root `install-tl*`
  files the first step still exercises sha512, gpgv, GOODSIG and the pin. A partial
  `archive/` fails the container check, as it should.
- `publish` needs R2 credentials in `AWS_*` env vars; there is no mock, and the AWS CLI is
  not installed locally by default.
- Is the mirror fresh?
  `curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512`
  and read `last-modified`.
