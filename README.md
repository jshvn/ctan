# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)
[![license](https://img.shields.io/github/license/jshvn/ctan)](https://github.com/jshvn/ctan/blob/main/LICENSE)
[![mirror](https://healthchecks.io/badge/8955b5d3-ba3b-4e8a-ac39-8501494333f5/otTXcui6-2.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

A daily mirror of `CTAN/systems/texlive/tlnet`, the directory `tlmgr` installs and updates
from, served from Cloudflare R2. It is not a full CTAN mirror: only tlnet is here, with every
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

The path mirrors CTAN's own layout, so the host works anywhere a CTAN mirror URL does.

It tracks the current TeX Live release and moves to the next one when upstream does.

## Why

CTAN's mirror rotation kept breaking my builds: broken TLS, unreachable hosts, stale copies.
This is one mirror I control, pulled from the CTAN master daily and verified before it goes
live.

## How it works

A GitHub Actions cron runs `task sync` once a day. Every step is in
[`Taskfile.yml`](https://github.com/jshvn/ctan/blob/main/Taskfile.yml):

1. **fetch**: rsync tlnet from the CTAN master, dereferencing symlinks so each container is
   stored once under the name `tlmgr` requests.
2. **verify**: check `texlive.tlpdb` against its SHA-512 and GPG signature (TeX Live key
   fingerprint pinned), then every container against the checksums the tlpdb carries. A tree
   caught mid-update is never published.
3. **publish**: `aws s3 sync` the containers first, then the rest, so no client ever reads a
   `texlive.tlpdb` that names a file the bucket lacks.
4. **smoke**: read `texlive.tlpdb.sha512` back through the public URL and compare.
5. **report**: write the numbers above to the run's summary on GitHub.
6. **ping**: tell healthchecks.io the run completed.
7. **page**: render this README into the landing page at
   [ctan.ijosh.com](https://ctan.ijosh.com/).

A failed run emails me. A run that never happens trips healthchecks.io.

Is it fresh? Check `last-modified` on the index:

```sh
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
```

## Want your own?

1. Fork [this repo](https://github.com/jshvn/ctan).
2. Create an R2 bucket named `tlnet`, an API token with Object Read & Write scoped to it,
   and a custom domain pointing at the bucket. Set `HOST` to that domain in `Taskfile.yml`.
   For a landing page at `/`, add a Cloudflare Transform Rule rewriting the path `/` to
   `/index.html`, and put your own links in the header of `site/template.html`.
3. Add the repository secrets `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
   and optionally `HEALTHCHECK_URL` (a healthchecks.io ping URL).
4. Actions -> sync -> Run workflow. The first run uploads the whole mirror in about seven
   minutes; every run after that pushes the daily delta.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
