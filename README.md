# ctan

[![sync](https://github.com/jshvn/ctan/actions/workflows/sync.yml/badge.svg)](https://github.com/jshvn/ctan/actions/workflows/sync.yml)

A nearly no-code mirror of the TeX Live network repository (`CTAN/systems/texlive/tlnet`),
served from Cloudflare R2 and refreshed daily. Every platform, docs and sources included.

This is not a full CTAN mirror. It carries only `systems/texlive/tlnet`, the directory that
`tlmgr` installs and updates from, so it is useful to TeX Live and TinyTeX users and to
nobody else. Packages on CTAN outside tlnet are not here.

### Why

I kept running into build failures on the way to CTAN: mirrors with broken TLS, mirrors that 
were unreachable, or some other issues. This is a simple copy of CTAN to avoid those problems.

## How to use

```sh
tlmgr option repository https://ctan.ijosh.com/systems/texlive/tlnet/
```

The path mirrors CTAN's own layout, so the host works anywhere a CTAN mirror URL does.

This is a TeX Live 2026 mirror. It tracks tlnet as-is, so it moves to 2027 when upstream
does, and `tlmgr` will then refuse to update a 2026 install from it, as it would from any
mirror.

## How it works

A GitHub Actions cron runs `task sync` once a day: rsync tlnet from a CTAN mirror, verify
the GPG signature on `texlive.tlpdb`, and `aws s3 sync` the tree to R2. Every step is in
[`Taskfile.yml`](Taskfile.yml).

## Want your own?

1. Clone this repo.
2. Create an R2 bucket named `tlnet`, an API token with Object Read & Write on it, and a
   custom domain pointing at the bucket. Change `URL` in `Taskfile.yml` to that domain.
3. Add the repository secrets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
   and optionally `HEALTHCHECK_URL`.
4. Actions -> sync -> Run workflow. The first run uploads the whole mirror; every run after
   that pushes the daily delta.

Pull requests are welcome, especially anything that keeps the pipeline smaller.
