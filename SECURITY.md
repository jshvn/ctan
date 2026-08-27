# Security

## What this mirror guarantees

Every hourly run verifies the signed part of the tree before uploading it:

- `texlive.tlpdb`, the three installers and the two `update-tlmgr-latest` updaters under
  `systems/texlive/tlnet/` are checked against their SHA-512 and GPG signatures, with the
  TeX Live primary key fingerprint pinned in `Taskfile.yml`. A signature from an expired or
  revoked key is rejected.
- `texlive.tlpdb.xz`, the copy `tlmgr` downloads, must decompress to the verified
  `texlive.tlpdb` byte for byte.
- Every package container is checked against the checksum recorded in the signed tlpdb, and
  the tlpdb goes live only after every container it names is in the bucket.
- A batch that fails any check is not uploaded; the bucket stays at its last checkpoint.

Those are the only files on CTAN with a signature this mirror can pin. Everything else on
CTAN, including tlcontrib and the TeX Live ISO images, is copied as served, byte for byte,
and carries whatever checksums upstream publishes beside it.

`tlmgr` repeats the signature check on the client, so a tampered mirror is rejected there
too. After each publish, `texlive.tlpdb.sha512` is read back through the domain and compared
with what was uploaded.

Uploads are batched. Containers land before the tlpdb that names them and `timestamp` lands
last, so a `tlmgr` run that overlaps a publish sees the previous tlpdb, never a tlpdb naming
a file that is not there.

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
