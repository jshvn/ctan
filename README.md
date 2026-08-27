# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)
[![license](https://img.shields.io/github/license/jshvn/ctan)](https://github.com/jshvn/ctan/blob/main/LICENSE)
[![mirror](https://healthchecks.io/b/2/f4ad55cc-4b2d-4cce-b133-aba5381d9e71.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

An hourly mirror of all of [CTAN](https://ctan.org) on Cloudflare R2, served at
`https://ctan.ijosh.com/` with every CTAN path at the root. About 496,000 files and 133 GB.

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

To go back to CTAN's mirror rotation: `tlmgr option repository ctan`.

## How it works

Every hour GitHub Actions lists CTAN's master (dante), diffs the listing against the listing
the previous run left in the bucket, fetches only the changed files in batches, verifies the
signed TeX Live files (`texlive.tlpdb`, the installers and the `tlmgr` updaters, with the TeX
Live key fingerprint pinned) and every package container against the tlpdb's checksums, and
uploads. Every step is in the
[`Taskfile.yml`](https://github.com/jshvn/ctan/blob/main/Taskfile.yml).

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

1. Fork [this repo](https://github.com/jshvn/ctan).
2. Create an R2 bucket named `ctan`, an API token with Object Read & Write scoped to it,
   and a custom domain pointing at the bucket. Set `HOST` to that domain in `Taskfile.yml`
   and in `cloudflare/*.json`.
3. Add the repository secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`
   (`https://<account-id>.r2.cloudflarestorage.com`) and `AWS_REGION` (`auto`); optionally
   `HEALTHCHECK_URL` (a healthchecks.io ping URL) and `CF_API_TOKEN` plus
   `CF_ZONE_ID` (a token with the zone's Cache Rules, Transform Rules and Single Redirect
   edit permissions) so the `/` rewrite and the directory redirects are applied from
   `cloudflare/`; without them, add a Transform Rule rewriting `/` to `/index.html` by hand.
4. Actions -> sync -> Run workflow with `seed` checked and `max_batches` at 40. The first
   run uploads everything (about 133 GB, a few hours); every run after that pushes the
   hourly delta. Storage past R2's free 10 GB costs about $1.86 a month.

To run the pipeline locally, `task run -- task --dry sync` renders it inside the toolbox
image (Apple `container` or Docker); with `AWS_*` variables exported, `task run -- task sync`
runs it for real.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
