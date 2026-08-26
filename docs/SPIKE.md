# Spike: a self-owned CTAN mirror on Cloudflare R2 at $0/month

Measured 2026-08-25 against rsync://rsync.dante.ctan.org/CTAN/ (the primary node).

## Sizes (real files, symlinks not double-counted)

| Tree                                  | Files   | Size     | Fits R2 free 10 GB? |
|---------------------------------------|---------|----------|---------------------|
| Full CTAN                             | 351,866 | 68.98 GB | No (~$0.89/month)   |
| systems/texlive/tlnet (all of it)     |  16,971 |  6.78 GB | Yes, 3.2 GB spare   |
| tlnet minus docs                      |         |  3.07 GB | Yes                 |
| tlnet, Yihui's subset (no doc/src, 5 platforms) |  |  2.40 GB | Yes           |

Full CTAN top-level: systems 40.6 GB (win32 17.6, mac 7.7, texlive/Images 6.8 ISO,
texlive/tlnet 6.8), macros 7.6, fonts 6.8, obsolete 3.4, graphics 2.9, support 2.2, info 1.9,
install 1.7. Two files exceed 5 GB (MacTeX pkg, TL ISO).

## What is actually "free"

| Piece            | Limit that matters                                  | tlnet mirror usage |
|------------------|-----------------------------------------------------|--------------------|
| R2 storage       | 10 GB-month (averaged peak per day over 30 days)    | 6.8 GB             |
| R2 Class A ops   | 1M/month (PutObject, ListObjects, multipart parts)  | ~17k first run, then daily delta (hundreds); 18 ListObjects pages |
| R2 Class B ops   | 10M/month (GetObject, HeadObject)                   | your tlmgr traffic; a full install is ~5k GETs |
| R2 egress        | free                                                |                    |
| R2 deletes       | free                                                |                    |
| GitHub Actions   | free on public repos; private gets 2,000 min/month  | ~5-10 min/day      |
| Cloudflare zone  | free plan; custom domain on R2 bucket is free       |                    |
| Cache Rules      | 10 rules on free plan; purge-everything is free     | optional           |

Not free, or not zero-friction:
- R2 activation requires a payment method on file (community reports a temporary $5
  preauth hold, no charge). You cannot enable R2 without one.
- A domain, if you want `tlnet.example.com` instead of the rate-limited `*.r2.dev`
  subdomain (dev-only, not for real traffic). You need a zone in the same Cloudflare account.
- Full CTAN: 59 GB over the free tier = ~$0.89/month. There is no free object store that
  holds 69 GB; Pages/Workers static assets cap at 20,000 files and 25 MiB each (out on both
  counts: 352k files, 417 files over 25 MB).

## Why a plain object store works for tlmgr (and where it bites)

- tlmgr over HTTP requests `archive/<pkg>.tar.xz` (unversioned) -- TLPDB.pm, `media eq 'NET'`
  branch. Upstream stores `<pkg>.r<rev>.tar.xz` as the real file and the unversioned name as a
  symlink (14,867 symlinks). R2 has no symlinks, so the mirror must publish dereferenced
  unversioned files and drop the versioned originals. `rsync -rLt --exclude='*.r[0-9]*.tar.xz'`
  does exactly that in one command: 17,393 files, 6.78 GB (verified by dry-run).
- Consequence: a revision bump overwrites `foo.tar.xz` in place, so change detection must be
  by content, not size (`rclone --checksum`: local MD5 vs R2 ETag, which is MD5 for
  single-part uploads; the largest tlnet file is 145 MB, so `--s3-upload-cutoff 200M` keeps
  every upload single-part = 1 Class A op).
- Publish containers before `tlpkg/texlive.tlpdb`, so a client never reads a tlpdb that names
  a file the bucket lacks. `rclone copy archive/` then `rclone sync .` gives that order plus
  deletions.
- Trust boundary: the tree carries `texlive.tlpdb.sha512` and a detached signature. Verify
  both before publishing and pin the TeX Live primary key fingerprint
  (C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC, tug.org/texlive/verify.html) -- the keyring
  ships inside the mirrored tree, so without the pin a hostile mirror could self-sign.
  Verified live in this spike against mirrors.mit.edu.
- Cloudflare's default cache list has no `xz`, so every `.tar.xz` GET goes to R2 (Class B,
  cheap) and there is no stale-cache problem after an in-place overwrite. If you later add a
  "cache everything" rule, add a purge-everything call to the publish task (free plan has it).

## Design: stateless daily job, no shell scripts, one Taskfile

```
GitHub Actions cron (daily)
  -> go-task/setup-task, apt rclone
  -> task sync
       fetch   : rsync -rLt --delete from a CTAN rsync mirror into ./staging (6.8 GB, ~5 min)
       verify  : shasum -c tlpdb.sha512; gpg --verify with pinned fingerprint
       publish : rclone copy archive/ -> r2:tlnet/archive ; rclone sync staging -> r2:tlnet
Cloudflare R2 bucket "tlnet"  <-  custom domain tlnet.example.com  <-  tlmgr option repository https://tlnet.example.com/
```

Deliberate simplifications (ponytail):
- Re-download the full 6.8 GB every run instead of caching. Zero state, zero cache-eviction
  code, fits the runner's ~19 GB free disk. Upgrade path if it ever matters: `rclone mount`
  the bucket and rsync into it, or build a phantom tree from `rclone lsjson` (sparse files with
  upstream sizes/mtimes) so rsync transfers only the delta.
- Pull from `mirrors.mit.edu` (rsync, near the Azure US runners) rather than dante; we are not
  an official mirror, so the "must sync from the primary node hourly" rule does not apply.
- No directory index pages. tlmgr does not need them.

Repository hygiene:
- Public repo: GitHub disables cron workflows after 60 days without a commit. Either commit
  occasionally, add a keepalive action, or make the repo private (2,000 free minutes/month
  covers ~10 min/day and private repos are exempt from the 60-day rule).
- Secrets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` from an R2 API token
  scoped to Object Read & Write on the one bucket. rclone reads them as
  `RCLONE_CONFIG_R2_*` env vars -- no rclone.conf anywhere.

## Cloudflare setup (one-time, dashboard)

1. R2 -> enable (payment method prompt) -> create bucket `tlnet` (location hint: ENAM).
2. R2 -> Manage API tokens -> Object Read & Write, bucket `tlnet` -> copy key id/secret.
3. Bucket -> Settings -> Custom Domains -> `tlnet.<your zone>`. Cloudflare adds the DNS record.
4. Optional: Cache Rule "hostname eq tlnet.<zone>" -> Eligible for cache, Edge TTL 1 day,
   plus the purge step in `publish`.
5. `tlmgr option repository https://tlnet.<zone>/`.

## Options if "entire CTAN" is a hard requirement

- Pay ~$0.89/month (R2 storage over 10 GB at $0.015/GB). Everything else in this design
  scales: 352k PutObjects fit the 1M Class A allowance for the initial load; ~25k CTAN symlinks
  need `-L` (adds a few GB) or `--no-links`.
- $0 hybrid: a Cloudflare Worker (free: 100k requests/day) that serves `systems/texlive/tlnet/`
  from the R2 binding and reverse-proxies every other path to `mirror.ctan.org` with edge
  caching. You own the part that breaks every March (tlnet) and only front the rest.
- Not viable at $0: Pages/Workers assets (20k files, 25 MiB), GitHub Releases as bulk storage
  (2 GB per asset and it is not what releases are for).

## Verification done in this spike

- `rsync -rLtn --stats --exclude='*.r[0-9]*.tar.xz'` against dante: 17,393 files, 6.78 GB.
- Full-CTAN and tlnet listings parsed for the size tables above.
- `TLPDB.pm` from the current install-tl (TL 2026) read to confirm unversioned names over NET.
- `task --dry sync` renders the pipeline; `task size` runs the live dry-run; `task verify`
  passed against a real `tlpkg/` pulled from mirrors.mit.edu (sha512 OK, VALIDSIG with the
  pinned primary key).
- Not tested: an actual `rclone` push to R2 (no credentials in this session).
