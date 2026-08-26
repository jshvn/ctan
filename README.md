# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)
[![license](https://img.shields.io/github/license/jshvn/ctan)](https://github.com/jshvn/ctan/blob/main/LICENSE)
[![mirror](https://healthchecks.io/badge/8955b5d3-ba3b-4e8a-ac39-8501494333f5/otTXcui6-2.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

A daily mirror of `CTAN/systems/texlive/tlnet` on Cloudflare R2. This is the directory
`tlmgr` installs and updates from, and it is the only part of CTAN here, complete with every
platform, docs and sources.

## How to use

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

The path mirrors CTAN's own layout, so the host works anywhere a CTAN mirror URL does. It
tracks the current TeX Live release and moves to the next one when upstream does.

## How it works

Once a day GitHub Actions runs a job to sync the tlnet directory to R2. It verifies
`texlive.tlpdb` against its SHA-512 and GPG signature (via pinned TeX Live key) and every
package container against its checksum. Every step is in the
[`Taskfile.yml`](https://github.com/jshvn/ctan/blob/main/Taskfile.yml).

**Is it fresh?**

Check `last-modified` on the index:

```sh
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
```

## Why use this?

`tlmgr`'s default repository is CTAN's mirror rotation: every request lands on a different
volunteer mirror, and any one of them can be unreachable, on a slow connection, behind on
TLS, or a few days stale.

This setup ensures a consistent, reliable source for TeX Live updates built on
[Cloudflare's network](https://cloudflare.com/network).

## Want your own?

1. Fork [this repo](https://github.com/jshvn/ctan).
2. Create an R2 bucket named `tlnet`, an API token with Object Read & Write scoped to it,
   and a custom domain pointing at the bucket. Set `HOST` to that domain in `Taskfile.yml`.
   For a landing page at `/`, add a Cloudflare Transform Rule rewriting the path `/` to
   `/index.html`, and put your own links and text in `site/index.html`.
3. Add the repository secrets `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
   and optionally `HEALTHCHECK_URL` (a healthchecks.io ping URL).
4. Actions -> sync -> Run workflow. The first run uploads the whole mirror in about seven
   minutes; every run after that pushes the daily delta.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
