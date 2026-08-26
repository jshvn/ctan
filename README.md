# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

A nearly no-code mirror of the TeX Live network repository (`CTAN/systems/texlive/tlnet`),
served from Cloudflare R2 and refreshed daily. Every platform, docs and sources included.
It runs for $0, inside the R2 free tier.

This is not a full CTAN mirror. It carries only `systems/texlive/tlnet`, the directory that
`tlmgr` installs and updates from, so it is useful to TeX Live and TinyTeX users and to
nobody else. Packages on CTAN outside tlnet are not here.

## Why

I kept running into build failures on the way to CTAN: mirrors with broken TLS, mirrors that
were unreachable, mirrors a day behind. This is one copy of the one directory I need, pulled
from the CTAN master every day and checked before it is published.

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

This is a TeX Live 2026 mirror. It tracks tlnet as-is, so it moves to 2027 when upstream
does, and `tlmgr` will then refuse to update a 2026 install from it, as it would from any
mirror.

## How it works

A GitHub Actions cron runs `task sync` once a day. Every step is in
[`Taskfile.yml`](https://github.com/jshvn/ctan/blob/main/Taskfile.yml):

1. **fetch**: rsync tlnet from the CTAN master, dereferencing symlinks so each container is
   stored once under the name `tlmgr` requests.
2. **verify**: check `texlive.tlpdb` against its SHA-512 and GPG signature (TeX Live key
   fingerprint pinned), then every container against the checksums the tlpdb carries. A tree
   caught mid-update is never published.
3. **guard**: refuse to publish past 10 GB, the edge of the free tier.
4. **page**: render this README into the landing page at
   [ctan.ijosh.com](https://ctan.ijosh.com/).
5. **publish**: `aws s3 sync` the containers first, then the rest, so no client ever reads a
   `texlive.tlpdb` that names a file the bucket lacks.
6. **smoke**: read `texlive.tlpdb.sha512` back through the public URL and compare.
7. **ping**: tell healthchecks.io the run completed.
8. **guard** again: fail the job past 9 GB. The mirror is already updated; the failure email
   is the alert.

A failed run emails me. A run that never happens trips healthchecks.io. Any Cloudflare spend
at all trips a budget alert.

Is it fresh? Check `last-modified` on the index:

```sh
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
```

## Want your own?

1. Fork [this repo](https://github.com/jshvn/ctan).
2. Create an R2 bucket, an API token with Object Read & Write scoped to it, and a custom
   domain pointing at the bucket. Set `BUCKET` and `URL` in `Taskfile.yml`. For a landing
   page at `/`, add a Cloudflare Transform Rule rewriting the path `/` to `/index.html`, and
   put your own links in the footer of `site/template.html`.
3. Add the repository secrets `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
   and optionally `HEALTHCHECK_URL` (a healthchecks.io ping URL).
4. Actions -> sync -> Run workflow. The first run uploads the whole mirror in about seven
   minutes; every run after that pushes the daily delta.

tlnet is about 6.8 GB, inside R2's 10 GB free tier, so a mirror like this costs nothing to
run. Pull requests are welcome, especially anything that keeps the pipeline smaller; the
`check` workflow renders the Taskfile on every PR.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
