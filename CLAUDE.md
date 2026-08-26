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
- No dependencies beyond `rsync`, `rclone`, `gpg`, `shasum`, `curl`, `task`.
- No AI attribution anywhere. No emojis.

## Gotchas

- `tlmgr` requests the unversioned `archive/foo.tar.xz`; upstream stores `foo.r123.tar.xz`
  and symlinks to it. R2 has no symlinks, so `fetch` uses `rsync -L` and excludes
  `*.r[0-9]*.tar.xz`; `verify` fails if a versioned file survives. Storing both doubles
  storage and breaks the free tier.
- Files are overwritten in place on revision bumps, so `rclone` must run with `--checksum`;
  size-only comparison misses same-size updates.
- Keep uploads single-part (`--s3-upload-cutoff` above the largest file). Multipart costs
  extra Class A ops and the ETag stops being an MD5, which breaks `--checksum`.
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
  https://www.tug.org/texlive/verify.html.
- `.xz` is not in Cloudflare's default cache list, so there is no stale-edge problem. If a
  cache-everything rule is ever added, add a purge step to `publish` before `smoke`.
- Do not add upstream-freshness monitoring: tlnet goes quiet for weeks before each release.
- Public repos have scheduled workflows disabled after 60 days without a commit; the
  healthchecks ping catches it within a day.
- Secrets are exactly four: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`
  (rclone reads them as `RCLONE_CONFIG_R2_*`, no rclone.conf) and `HEALTHCHECK_URL`.

## Deployment Status

Not yet deployed. Verified: `task --dry sync`, `task size`, `task verify` against a real
`tlpkg/`, and `guard`/`smoke`/`ping` offline in both directions. Not verified: a real
`rclone` push, the workflow on a runner, `tlmgr` through the domain, the failure email, the
healthchecks ping, the budget alert.

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
- `task size` is the live upstream dry run (must stay under 10 GB).
- `task verify` runs locally after `rsync`-ing only `tlpkg/` into `staging/tlpkg/`.
- `publish` needs R2 credentials in `RCLONE_CONFIG_R2_*` env vars; there is no mock.
