# Security

## What this mirror guarantees

Every daily run verifies the tree before publishing it:

- `texlive.tlpdb` is checked against its SHA-512 and its GPG signature, with the TeX Live
  primary key fingerprint pinned in `Taskfile.yml`.
- Every package container is checked against the `containerchecksum` recorded in the
  signed tlpdb.
- A tree that fails either check is never published; the previous good copy stays live.

`tlmgr` repeats the signature check on the client, so a tampered mirror is rejected there
too. After each publish, the index is read back through the domain and compared with what
was uploaded.

Uploads are not atomic. Containers land first and the tlpdb that names them a minute or so
later (around 03:35 UTC daily), and containers are overwritten in place, so a `tlmgr` run
that overlaps the publish can see checksum errors. Rerun it.

## Reporting

If you find a way to serve altered or unsigned content through `ctan.ijosh.com`, or a
weakness in the pipeline itself, report it privately through
[GitHub's vulnerability reporting](https://github.com/jshvn/ctan/security/advisories/new).
Please do not open a public issue for it.

Problems with the packages themselves (a malicious or broken upstream package) belong to
[TeX Live](https://tug.org/texlive/) and [CTAN](https://ctan.org/); this mirror copies
what they publish, byte for byte.

## Supported versions

Only the current `main` branch and the live mirror. There are no releases.
