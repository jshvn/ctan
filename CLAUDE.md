# ctan -- Project Instructions for AI Agents

## What This Is

A daily-synced mirror of `CTAN/systems/texlive/tlnet` on Cloudflare R2, served at
`https://ctan.ijosh.com/systems/texlive/tlnet/`, with `README.md` rendered as the landing page
at `https://ctan.ijosh.com/`. All logic is in `Taskfile.yml`; `.github/workflows/sync.yml`
installs tools and runs `task sync` on a cron; `check.yml` runs `task --dry sync` on pull
requests. tlnet is ~17,400 files / 6.8 GB; the largest file is ~145 MB.

`README.md` is written for users. Operational detail (guards, sizes, cost, alerting) belongs
in this file and in Taskfile comments, not there.

## Hard Constraints

- Zero running cost: R2 free tier (10 GB-month, 1M Class A, 10M Class B) and free GitHub
  Actions. If a change adds storage, recompute against the 6.8 GB baseline first.
- No shell scripts. All logic lives in `Taskfile.yml`; workflows install tools and run one
  task, nothing more.
- No dependencies beyond `rsync`, `aws` (AWS CLI v2), `gpg`, `shasum`, `curl`, `task`.
  The only network endpoints are the CTAN master, R2, the public domain, healthchecks.io
  and `api.github.com/markdown` (used by `page`).
- No AI attribution anywhere. No emojis.

## Gotchas

- `tlmgr` requests the unversioned `archive/foo.tar.xz`; upstream stores `foo.r123.tar.xz`
  and symlinks to it. R2 has no symlinks, so `fetch` uses `rsync -L` and excludes
  `*.r[0-9]*.tar.xz`; `verify` fails if a versioned file survives. Storing both doubles
  storage and breaks the free tier.
- Files are overwritten in place on revision bumps, often at the same size. `aws s3 sync`
  uploads when size differs or the local mtime is newer than the object's LastModified;
  `rsync -t` keeps upstream mtimes and a bump is always built after our previous upload, so
  the daily cadence catches them. Never add `--size-only`.
- Keep uploads single-part: `aws.config` sets `multipart_threshold` above the largest file
  (145 MB). The CLI default is 8 MB, and multipart costs three or more Class A ops per file.
- `publish` order is load-bearing: `archive/` first, then the full sync (tlpdb), then the
  deletions from `stale`, so no client reads a `texlive.tlpdb` that names a container the
  bucket lacks. `index.html` goes last, to the bucket root, outside the mirror prefix so the
  `stale` listing never sees it, with `Cache-Control: no-cache` so the page is never served
  stale.
- Never add `--delete` to `aws s3 sync`. R2 lists a key after the keys it is a prefix of
  (`install-tl` after `install-tl.zip.sha512.asc`; its end-of-key sorts like `/`, after `-`
  and `.`), while the CLI walks staging in byte order. The merge-join then emits both an
  `upload:` and a `delete:` for every such key (15 in tlnet: the installers and their
  `.sha512`, `texlive.tlpdb` and its `.sha512`, two tlperl files), and whichever lands last
  wins; on 2026-08-26 that took `texlive.tlpdb.sha512` off the mirror. Without `--delete`
  the 15 keys are re-uploaded every run, which is harmless. Deletions are `stale`'s
  byte-sorted `comm` of the bucket listing against staging, run through `aws s3 rm`
  (DeleteObject is free on R2), so they cannot race an upload of the same key.
- `sync` order is load-bearing around `guard`: the 10 GB check before `page`/`publish`
  (never bill), the 9 GB check after (alert without an outage). A failed run is the alert;
  do not add a notification dependency.
- `fetch` and `publish` `tee` their output into `RUN` (`/tmp/tlnet-run`) for `report`, under
  `set: [pipefail]` so `tee` never masks a failed rsync or upload. `report` counts
  `upload:`/`delete:` lines from that file (`aws s3 rm` prints the same `delete:` form), not
  the job log, so the log-capture gap below does not affect it.
- `ping` sits after `smoke` and before the soft `guard`. Later: every size warning becomes two
  emails. Earlier: a broken domain looks alive.
- Objects live under `systems/texlive/tlnet/` in the bucket. Do not move them; every user's
  tlmgr config carries the path.
- `verify` pins the TeX Live primary key fingerprint; the keyring comes from the mirror being
  verified, so the pin is the only real check. Rotate only against
  https://www.tug.org/texlive/verify.html. It then checks every container against the
  tlpdb's `containerchecksum` fields; source containers are `<name>.source.tar.xz`.
- `page` POSTs `README.md` to GitHub's Markdown API and splices the result into
  `site/template.html`; `site/index.html` is build output and gitignored. In Actions the
  automatic `GITHUB_TOKEN` lifts the per-IP rate limit; locally the anonymous limit
  (60/hour) is plenty. The `grep markdown-heading` line is the render check. A Cloudflare
  Transform Rule rewrites `/` to `/index.html`; without it the root is a 404 while the
  mirror path still works.
- `.xz` is not in Cloudflare's default cache list, so there is no stale-edge problem. If a
  cache-everything rule is ever added, add a purge step to `publish` before `smoke`.
- Do not add upstream-freshness monitoring: tlnet goes quiet for weeks before each release.
- Public repos have scheduled workflows disabled after 60 days without a commit; the
  healthchecks ping catches it within a day. Dependabot's weekly Actions PRs are the only
  routine commits.
- GitHub's log capture drops lines when `publish` prints thousands of `upload:` lines in a
  few seconds; the first full run's log shows ~7.9k of 17.4k uploads although the bucket was
  complete. Judge completeness by `smoke` and spot checks, never by counting log lines.
- Secrets are exactly four: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`
  (the workflow maps them to `AWS_ENDPOINT_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`;
  `AWS_REGION` is the constant `auto`) and `HEALTHCHECK_URL`. `GITHUB_TOKEN` is the automatic
  read-only token, passed through as an env var, not a secret to set.

## Deployment Status

Live since 2026-08-26 (run 32933376123, 6m55s): fetch from dante 6.8 GB in ~5 min (22 to
143 MB/s seen), `verify` with the container check ~16 s, `publish` ~80 s for the full first
upload, `smoke` and `ping` passed, healthchecks reports up, and `tlmgr` installs from the
domain. Scheduled runs push the daily delta.

Run 32941682244 (2026-08-26, 1m47s) verified `page`, the `index.html` upload, the Transform
Rule for `/`, `stale` against the real bucket (zero `delete:` lines, exactly the 15 prefix
keys re-uploaded) and that `report` writes to the job page. Still open: the edge serves
`index.html` with `max-age=86400` although `publish` sets `no-cache`, so a Cloudflare cache
setting is overriding the object header (`curl -sI https://ctan.ijosh.com/index.html`); and
a `report` with populated Mirror and Fetched rows has not been seen yet. The failure email
fired once, for the 2026-08-26 `smoke` 404; the budget alert has not.

Dashboard setup that exists and would need redoing on a new account: R2 bucket `tlnet`;
API token Object Read & Write scoped to it; custom domain `ctan.ijosh.com`; lifecycle rule
aborting incomplete multipart uploads after 1 day; budget alert at $1; healthchecks.io check
(period 1 day, grace 3 h, job starts 03:30 UTC) with its status badge in the README.

Repository: topics set, private vulnerability reporting on (`SECURITY.md` links to it),
`CONTRIBUTING.md` carries the ground rules for outside PRs, Dependabot groups Actions bumps
weekly.

## Verifying a Change

- `task --dry sync` renders the pipeline without touching the network.
- `task guard STAGING=<dir> LIMIT_MB=<n>` exercises the size check against any directory.
- `task smoke URL=file:///<dir> STAGING=<dir>` exercises the read-back check offline.
- `task ping` is a no-op without `HEALTHCHECK_URL`; set it to any `file://` URL to exercise it.
- `task stale RUN=<dir> STAGING=<dir>` diffs a canned `remote.txt` (one key per line, any
  order) in `<dir>` against `<dir>`'s files and writes `stale.txt`; it must be empty when
  every listed key exists locally, and it must fail on an empty directory.
- `task report RUN=<dir> STAGING=<dir>` renders the summary to stdout from a canned
  `fetch.txt` (rsync `--stats` block) and `publish.txt` (`aws s3 sync` lines) in `<dir>`.
- `task page` renders the README locally (needs `api.github.com`); open `site/index.html`.
- `task size` is the live upstream dry run (must stay under 10 GB). If it hangs, the master
  is stalling on recursive listing; the rsync `--timeout` turns that into an error in CI.
- `task verify` needs a full `staging/` (the container check reads every archive); with only
  `tlpkg/` present the first three commands still exercise sha512, gpg and the pin. A partial
  `archive/` fails the container check by design.
- `publish` needs R2 credentials in `AWS_*` env vars (see the Taskfile header); there is no
  mock, and the AWS CLI is not installed locally by default.
- Is the mirror fresh? `curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512`
  and read `last-modified`.
