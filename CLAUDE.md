# ctan -- Project Instructions for AI Agents

## What This Is

A daily-synced mirror of `CTAN/systems/texlive/tlnet` on Cloudflare R2, served at
`https://ctan.ijosh.com/systems/texlive/tlnet/`. All logic is in `Taskfile.yml`;
`.github/workflows/sync.yml` installs tools and runs `task sync` on a cron; `check.yml` runs
`task --dry sync` on pull requests. tlnet is ~17,400 files / 6.8 GB; the largest file is
~145 MB.

## Hard Constraints

- Zero running cost: R2 free tier (10 GB-month, 1M Class A, 10M Class B) and free GitHub
  Actions. If a change adds storage, recompute against the 6.8 GB baseline first.
- No shell scripts. All logic lives in `Taskfile.yml`; workflows install tools and run one
  task, nothing more.
- No dependencies beyond `rsync`, `aws` (AWS CLI v2), `gpg`, `shasum`, `curl`, `task`.
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
- `publish` order is load-bearing: `archive/` first, then the full `sync`, so no client reads
  a `texlive.tlpdb` that names a container the bucket lacks.
- `sync` order is load-bearing around `guard`: the 10 GB check before `publish` (never bill),
  the 9 GB check after (alert without an outage). A failed run is the alert; do not add a
  notification dependency.
- `ping` sits after `smoke` and before the soft `guard`. Later: every size warning becomes two
  emails. Earlier: a broken domain looks alive.
- Objects live under `systems/texlive/tlnet/` in the bucket. Do not move them; every user's
  tlmgr config carries the path.
- `verify` pins the TeX Live primary key fingerprint; the keyring comes from the mirror being
  verified, so the pin is the only real check. Rotate only against
  https://www.tug.org/texlive/verify.html. It then checks every container against the
  tlpdb's `containerchecksum` fields; source containers are `<name>.source.tar.xz`.
- `.xz` is not in Cloudflare's default cache list, so there is no stale-edge problem. If a
  cache-everything rule is ever added, add a purge step to `publish` before `smoke`.
- Do not add upstream-freshness monitoring: tlnet goes quiet for weeks before each release.
- Public repos have scheduled workflows disabled after 60 days without a commit; the
  healthchecks ping catches it within a day.
- Secrets are exactly four: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`
  (the workflow maps them to `AWS_ENDPOINT_URL`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`;
  `AWS_REGION` is the constant `auto`) and `HEALTHCHECK_URL`.

## Deployment Status

Not yet deployed. Verified on a runner: fetch from dante (6.8 GB in ~50 s), `verify`
including the container check (~17 s), `guard`. Verified offline: `smoke` and `ping` in both
directions. Not verified: `publish` against R2, `tlmgr` through the domain, the failure
email, the healthchecks ping, the budget alert.

Remaining setup, in order:
1. Cloudflare: R2 -> bucket `tlnet`; API token Object Read & Write scoped to it (account id
   is in the S3 endpoint); custom domain `ctan.ijosh.com`; lifecycle rule to abort incomplete
   multipart uploads after 1 day; Billing -> Billable Usage -> budget alert at $1.
2. healthchecks.io check: period 1 day, grace 3 hours (job starts 03:30 UTC, ~10 min).
3. Repository secrets, then Actions -> sync -> Run workflow (~6.8 GB, ~17k Class A ops).
4. `tlmgr option repository https://ctan.ijosh.com/systems/texlive/tlnet/` and
   `tlmgr update --self --all` from a real machine.

## Verifying a Change

- `task --dry sync` renders the pipeline without touching the network.
- `task guard STAGING=<dir> LIMIT_MB=<n>` exercises the size check against any directory.
- `task smoke URL=file:///<dir> STAGING=<dir>` exercises the read-back check offline.
- `task ping` is a no-op without `HEALTHCHECK_URL`; set it to any `file://` URL to exercise it.
- `task size` is the live upstream dry run (must stay under 10 GB). If it hangs, the master
  is stalling on recursive listing; the rsync `--timeout` turns that into an error in CI.
- `task verify` needs a full `staging/` (the container check reads every archive); with only
  `tlpkg/` present the first three commands still exercise sha512, gpg and the pin.
- `publish` needs R2 credentials in `AWS_*` env vars (see the Taskfile header); there is no
  mock, and the AWS CLI is not installed locally by default.
