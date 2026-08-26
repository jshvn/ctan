# ctan

A self-owned, CDN-fronted mirror of the TeX Live network repository
(`CTAN/systems/texlive/tlnet`), served from Cloudflare R2 and refreshed daily by GitHub
Actions. Whole tlnet, every platform, docs and sources included: about 17,400 files, 6.8 GB,
inside R2's 10 GB free tier. Total running cost: $0.

Why: CTAN has no CDN, the volunteer mirrors drift out of sync, and every March a new TeX Live
release leaves `tlmgr` pointed at a mirror on the wrong side of the cutover. One origin, one
state, updated once a day.

## Use it

```sh
tlmgr option repository https://tlnet.<your-zone>/
```

## How it works

```
GitHub Actions cron (daily)
  task sync
    fetch    rsync tlnet from a CTAN mirror, dereferencing symlinks
    verify   sha512 of texlive.tlpdb + GPG signature, TeX Live key fingerprint pinned
    publish  rclone copy archive/ then rclone sync .  ->  R2 bucket "tlnet"
Cloudflare R2 bucket  <-  custom domain  <-  tlmgr
```

Everything runs from `Taskfile.yml`; the workflow only installs `task` and `rclone` and runs
`task sync`. `task size` dry-runs against upstream and reports the current file count and
bytes without downloading anything.

## One-time setup

1. Cloudflare -> R2 -> enable (a payment method is required on file; the free tier is not
   charged) -> create bucket `tlnet`.
2. R2 -> Manage API tokens -> Object Read & Write scoped to `tlnet` -> note the key id and
   secret. The account id is in the bucket's S3 endpoint.
3. Bucket -> Settings -> Custom Domains -> `tlnet.<zone>` (the zone must be in the same
   Cloudflare account).
4. Repository secrets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`.
5. Actions -> sync -> Run workflow. First run uploads ~6.8 GB; later runs push the daily delta.

## Free-tier budget

| Resource | Allowance | This mirror |
|---|---|---|
| R2 storage | 10 GB-month | 6.8 GB |
| R2 Class A ops (Put, List) | 1M/month | ~17k on first run, then hundreds/day |
| R2 Class B ops (Get, Head) | 10M/month | your tlmgr traffic |
| R2 egress | free | |
| GitHub Actions | free on public repos | ~10 min/day |

Design notes, measurements and the rejected alternatives (full CTAN is 69 GB and cannot be
free) are in [docs/SPIKE.md](docs/SPIKE.md). Current state and next steps are in
[HANDOFF.md](HANDOFF.md).
