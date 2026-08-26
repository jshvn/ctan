# Security

## What this mirror guarantees

Every daily run verifies the tree before publishing it:

- `texlive.tlpdb` is checked against its SHA-512 and its GPG signature, with the TeX Live
  primary key fingerprint pinned in `Taskfile.yml`.
- Every package container is checked against the `containerchecksum` recorded in the
  signed tlpdb.
- A tree that fails either check is never published; the previous good copy stays live.

`tlmgr` repeats the signature check on the client, so a tampered mirror is rejected there
too. Read-back from the public URL is compared to what was uploaded.

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
