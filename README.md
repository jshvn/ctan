# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)
[![license](https://img.shields.io/github/license/jshvn/ctan)](https://github.com/jshvn/ctan/blob/main/LICENSE)
[![mirror](https://healthchecks.io/b/2/f4ad55cc-4b2d-4cce-b133-aba5381d9e71.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

An hourly mirror of all of [CTAN](https://ctan.org) on Cloudflare R2, served at
`https://ctan.ijosh.com/` with every CTAN path at the root. About 511,000 files and 140 GB.

## How to use

Your request served to `https://mirrors.ctan.org/` might be redirected to this mirror automatically (once I apply for official mirror status). Until then, the full mirror of CTAN is served from `https://ctan.ijosh.com/`.

TeX Live and TinyTeX both use `tlmgr`:

```sh
tlmgr option repository https://ctan.ijosh.com/systems/texlive/tlnet/
tlmgr update --self --all
```

For a fresh install, give the installer the same URL:

```sh
install-tl -repository https://ctan.ijosh.com/systems/texlive/tlnet/
```

Browse it: any directory URL, `https://ctan.ijosh.com/systems/knuth/`, lists what the
mirror holds there.

To go back to CTAN's mirror rotation: `tlmgr option repository ctan`.

## How it works

On an hourly schedule this GitHub action is triggered and runs the following pipeline.
Every step is a task in [`Taskfile.yml`](https://github.com/jshvn/ctan/blob/main/Taskfile.yml):

1. **`clock` `list` `state` `rebuild`** — stamp the hour, list CTAN's master (dante), and
   fetch the listing the previous run left in the bucket, rebuilding it if it went missing.
2. **`diff` `plan`** — take what upstream has and the state lacks, and split it into batches
   of at most 4 GB. The mirror is never a local copy: the runner has 14 GB, the tree has 140.
3. **`tlpdb` `batches`** — per batch, rsync the files, check the signed TeX Live control
   files against a pinned key fingerprint and every package container against the tlpdb's
   checksums, upload, and write the new state. A run that dies repeats one batch, not all.
4. **`delete` `reconcile`** — drop the keys that left upstream; once a day, sweep the bucket
   against the state for anything neither owns.
5. **`index`** — redraw the directory pages the run changed, since R2 has no listings.
6. **`smoke` `report` `ping`** — read a sample of the run's keys back over the public domain,
   summarise what landed, and ping healthchecks.io. Silence is the alert.

**Is it fresh?**

```sh
curl -s https://ctan.ijosh.com/timestamp
```

## Why use this?

I built this for myself. `mirrors.ctan.org` hands out a different volunteer mirror on every
request, and sooner or later one is overloaded, stale, or unreachable — which breaks the CI
that builds my documents. This is one hostname on
[Cloudflare's network](https://cloudflare.com/network) of 300+ cities, a short and
consistent hop wherever you are. It costs under $2 a month to run: R2 bills for storage and
nothing for bandwidth, so traffic doesn't move the bill.

## Want your own?

1. Fork [this repo](https://github.com/jshvn/ctan) and create an R2 bucket named `ctan` with
   a custom domain pointing at it.
2. Set `HOST` in `Taskfile.yml` to that domain — the only line in the repo that names the
   hostname.
3. Add the four repository secrets below. They are the whole requirement.
4. Actions -> sync -> Run workflow, with `seed` checked and `max_batches` at 40. The first
   run uploads everything (about 140 GB, a few hours); every run after it pushes the hourly
   delta. Storage past R2's free 10 GB costs about $1.95 a month.
5. Uncomment the `schedule:` block in `sync.yml` with a minute per hour to run the sync
   automation.

| Secret | What it is |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | R2 API token with Object Read & Write on the bucket |
| `AWS_SECRET_ACCESS_KEY` | That token's secret |
| `AWS_ENDPOINT_URL` | `https://<account-id>.r2.cloudflarestorage.com` |
| `AWS_REGION` | `auto` |

## Optional configuration

| Secret | What it turns on |
| --- | --- |
| `HEALTHCHECK_URL` | A healthchecks.io ping URL. The last step of every run pings it |

**Alerting.** `HEALTHCHECK_URL` is the only alert the mirror has: a failed or missing run
stops the ping, and healthchecks.io mails you when the grace passes. Point a check at cron
`42 * * * *` UTC with a 3 hour grace, which absorbs a queued run and a full one. It is also
the only thing watching whatever starts your runs. Unset, a run that stops arriving tells
nobody. Pause the check before a seed — a multi-hour run outlasts any sensible grace.

**Zone rules.** Four rules are worth setting on the zone by hand: HTML rewriters off (the
one that matters — Cloudflare otherwise alters every HTML file it serves), cache bypass, `/`
rewritten to `/index.html`, and every other directory URL rewritten to the mirror's own page
for that directory. The pipeline does not touch
Cloudflare and needs no zone token; section 6 of
[`docs/reference.md`](docs/reference.md) has each rule, its expression and its settings.
Skip them and every run still syncs, but `/` and every directory URL are 404s and HTML is
served rewritten.

To run the pipeline locally, `task run -- task --dry sync` renders it inside the toolbox
image (Apple `container` or Docker); with `AWS_*` variables exported, `task run -- task sync`
runs it for real. Every run happens in that image, the hourly one included, so a local run
and a run in Actions differ only in wall clock.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
