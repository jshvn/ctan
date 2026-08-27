# Verification and security

What the full mirror verifies before it publishes, what it cannot verify, where the trust
boundaries lie, and how the pipeline and the repository are guarded. Every number was
computed on 2026-08-26 from the dereferenced listing (`rsync -rL --list-only` of the whole
of CTAN, `SCRATCH/ctan-list-deref.txt`, 538,289 lines), the real tlpdb and keyring in
`staging/tlpkg/`, and pages fetched the same day. Commands are given so the numbers can be
regenerated.

Summary:

- CTAN signs almost nothing. Of 496,149 objects, 157 are `.asc` files and 458 are `.sig`
  files, and most of those are not signatures of anything a client trusts. Five indexes
  are signed with a key the mirror can pin: the tlnet tlpdb, the seven tlnet root
  installer and updater checksums, the tlcontrib tlpdb, and the two TeX Live ISO checksums.
  Everything else is copied as-is, and one upstream checksum file (`MacTeX.pkg.sha512`) is
  stale today, which is why unsigned checksums are never enforced.
- With only the delta on disk, the five tlnet checks still run in full. A container is
  checked against the tlpdb of the same run, so `tlpkg/` is fetched and verified first and
  uploaded last.
  "Every named container exists" is checked against what the bucket will hold after the
  run (state file plus delta minus deletions), not against dante's listing.
- The TeX Live signing subkey expires 2027-07-13, 320 days from today, confirmed against
  tug.org. `report` prints the days remaining. On the day it lapses the run fails and the
  mirror goes stale; that is the intended behaviour.
- The symlink-inflation guard is two numbers from the listing, checked before any byte is
  fetched: 496,149 objects and 132.99 GB against `CEILING_OBJECTS=650000` and
  `CEILING_GB=200`. A release day adds 20.94 GB and stays under it.
- Nothing here needs a tool outside `rsync`, `aws`, `gpg`/`gpgv`, `shasum`, `xz`, `curl`,
  `task`. Two secrets are added for Cloudflare (caching.md).

## 1. What is signed on CTAN

### Survey

Checksum and signature files by top-level directory, regular files only, counted from the
dereferenced listing, 496,149 objects once tlnet's versioned containers and the six root `update-tlmgr-r*` files are dropped (alias copies such as `bibliography/` count twice, as R2 stores
them). The path is fields 5 to the end (25 paths contain spaces); 523 zero-size files print a
blank size column and shift the fields by one, and none of them is a checksum file.

```sh
awk '$1 ~ /^-/ { p=$5; for(i=6;i<=NF;i++) p=p" "$i; n=split(p,a,"/"); f=a[n]; top=(n>1)?a[1]:"(root)";
  lf=tolower(f); k="";
  if (f ~ /\.asc$/) k="asc"; else if (f ~ /\.sig$/) k="sig"; else if (f ~ /\.sha512$/) k="sha512";
  else if (f ~ /\.sha256$/) k="sha256"; else if (f ~ /\.md5$/) k="md5";
  else if (lf ~ /^(checksums|sha256sums|sha512sums|md5sums|sha1sums)(\.[a-z0-9]+)*$/) k="SUMS";
  else if (f ~ /\.gpg$/) k="gpg"; else if (f ~ /\.(pem|crt)$/) k="pem";
  if (k!="") c[top" "k]++ } END { for (x in c) print c[x], x }' ctan-list-deref.txt | sort -k2
```

| Kind | Count | Where |
|---|---:|---|
| `.asc` | 157 | `systems` 138 (MiKTeX rpm `repomd.xml.asc` x2 aliases, MiKTeX source tarballs x2, TeX Live ISO x4, tlnet x7 pairs, tlcontrib x1), `macros` 8, `web` 7, `fonts` 2, `biblio` 1 (x2). At least 11 are plain text, not signatures: `systems/msdos/gtexedit/readme.asc`, `web/clip/*/ex01_?.asc`, `fonts/greek/kd/emtex/greek.asc`, `fonts/kixfont/kix.mf.asc` |
| `.sig` | 458 | `support` 443 (`aspell/dict` 177, `aspell/w32` 33, `preview-latex` ~50), `fonts` 11 (gnu-freefont, freetype), `obsolete` 4 (xpdf). No public key for any of them is in the tree |
| `.sha512` | 25 | `systems/mac/mactex` 14 (unsigned), `systems/texlive/Images` 2 (signed), tlnet 7 (signed), tlcontrib 1 (signed) |
| `.sha256` | 3 | `support` |
| `.md5` | 24 | MacTeX 14, ISO 2, tlnet and tlcontrib 2, misc 6 |
| `CHECKSUMS`-style | 10 | `dviware/catdvi`, `dviware/mdvi`, `obsolete/.../teTeX`, `obsolete/.../ghostscript` |
| `.gpg` keyrings and Release signatures | 24 | tlnet `tlpkg/gpg/{pubring,secring,trustdb}.gpg`, MiKTeX deb `dists/*/Release.gpg` x11 x2 aliases |
| `.pem`/`.crt` | 2 | tlnet's bundled CA files (curl and Mozilla::CA) |

Nothing at the archive root (`timestamp`, `FILES.byname`, `FILES.last07days`,
`CTAN.sites`, `tds.zip`, `index.html`) is signed or checksummed.

### Every index a client trusts

| Index | Files | Signed by | Verifiable with `shasum`+`gpgv`? | Checked today | Decision |
|---|---|---|---|---|---|
| tlnet tlpdb | `systems/texlive/tlnet/tlpkg/texlive.tlpdb{,.xz,.sha512,.sha512.asc}` | TeX Live Distribution, subkey `D8F2…8C70` of `C78B…B6BC` | yes, keyring in tree, fingerprint pinned | `GOODSIG` + `VALIDSIG …C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC`, sha512 OK, `.xz` matches | **verify** (unchanged) |
| tlnet installers and updaters | 7 `.sha512`/`.sha512.asc` pairs at the tlnet root (`install-tl-unx.tar.gz`, `install-tl-windows.exe`, `install-tl.zip`, `update-tlmgr-latest.{sh,exe}`, `update-tlmgr-r79982.{sh,exe}`) | same key | yes | `install-tl-unx.tar.gz.sha512.asc` fetched via mirror.ctan.org: `GOODSIG`, `VALIDSIG …B6BC` | **verify** (unchanged; the `update-tlmgr-r*` pair stays excluded as today) |
| tlcontrib tlpdb | `systems/texlive/tlcontrib/tlpkg/texlive.tlpdb{,.xz,.sha512,.sha512.asc}`, 530 files, 1.7 MB tlpdb | Norbert Preining, subkey `EBCC…0066` of `F7D8A92826E316A19FA0ACF06CACA448860CDC13`, present in tlnet's `pubring.gpg` | yes, with a second pin | fetched via mirror.ctan.org: `GOODSIG D80E09B087140066`, `VALIDSIG … F7D8A92826E316A19FA0ACF06CACA448860CDC13` | **verify**, same task, second pin `TLCONTRIB_KEY` |
| TeX Live ISO | `systems/texlive/Images/texlive2026.iso.sha512{,.asc}`, `texlive2026-20260301.iso.sha512{,.asc}`; three ISO copies of 6,784,798,720 bytes | TeX Live key | yes; hashing 6.78 GB takes ~12 s at the 570 MB/s measured here | `texlive2026.iso.sha512.asc`: `GOODSIG`, `VALIDSIG …B6BC` | **verify signature always; hash the ISO when it is in the delta** (release day only) |
| ISO `.md5.asc` | `texlive2026*.iso.md5.asc` | TeX Live key | yes | not needed | copy as-is; the sha512 is the stronger of the two |
| MacTeX | `systems/mac/mactex/*.pkg.{sha512,md5}`, 14 pairs | nobody | hash only, unsigned | `MacTeX.pkg.sha512` (2026-03-02) and `mactex-20260324.pkg.sha512` (2026-03-24) hold **different** hashes while the two `.pkg` are the same bytes (same size and mtime in the listing; `MacTeX.pkg` is upstream's symlink) | **copy as-is**; enforcing it would fail every run until MacTeX fixes the file. `.pkg` files carry Apple's own signature inside (unverified here) |
| MiKTeX package index | `systems/win32/miktex/tm/packages/{pr.ini,files.csv.lzma}` and `next/` | nobody | no | 0 `.sig` files under `miktex/` | copy as-is |
| MiKTeX rpm/deb repos | `setup/rpm/*/repodata/repomd.xml.asc` x18, `setup/deb/dists/*/{InRelease,Release.gpg}` x11 | MiKTeX Packager `D6BC243565B2087BC3F897C9277A7293F59E4889`, key shipped in the same tree as `setup/miktex.org.key` (expires about 2027-05-04) | technically yes; `repomd.xml.asc` for fedora/44 verifies `GOODSIG` against the in-tree key | key and signature come from the same unsigned directory; pinning a third party's packaging key is not this mirror's job | copy as-is |
| `.sig` files (aspell, freefont, freetype, xpdf) | 458 | keys not in the tree | no | | copy as-is |
| Archive root (`timestamp`, `FILES.*`, `CTAN.sites`) | 6 | nobody | no | | copy as-is; `timestamp` is what mirmon reads |

Why the line is drawn there: a check is worth adding only when (a) the key can be pinned
to a fingerprint published somewhere other than the mirror being checked, (b) a failure
means "do not publish" rather than "upstream's checksum file is stale", and (c) a
`tlmgr`-class client depends on it. tlnet, tlcontrib and the ISO meet all three; tlcontrib
and the ISO are the same twenty lines of Taskfile with a different path and pin. MacTeX
fails (b) today. MiKTeX fails (a) and (c) as far as this mirror can tell.

Commands run today (small files fetched through `https://mirror.ctan.org/`, keyring from
`staging/tlpkg/gpg/pubring.gpg`):

```
$ gpgv --status-fd 1 --keyring $K systems/texlive/tlcontrib/tlpkg/texlive.tlpdb.sha512.asc systems/texlive/tlcontrib/tlpkg/texlive.tlpdb.sha512
[GNUPG:] GOODSIG D80E09B087140066 Norbert Preining <norbert@preining.info>
[GNUPG:] VALIDSIG EBCC2CD2FAC0DAFA105F9DC8D80E09B087140066 2026-04-06 1775477881 0 4 0 1 10 01 F7D8A92826E316A19FA0ACF06CACA448860CDC13
$ gpgv --status-fd 1 --keyring $K systems/texlive/Images/texlive2026.iso.sha512.asc systems/texlive/Images/texlive2026.iso.sha512
[GNUPG:] GOODSIG 4CE1877E19438C70 TeX Live Distribution <tex-live@tug.org>
[GNUPG:] VALIDSIG D8F2F86057A857E42A88106A4CE1877E19438C70 2026-03-01 1772386654 0 4 0 1 10 01 C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC
$ cat systems/mac/mactex/MacTeX.pkg.sha512 systems/mac/mactex/mactex-20260324.pkg.sha512
204c019e1d3f…6fd0  MacTeX.pkg
68a62ac1d9e9…5623  mactex-20260324.pkg
```

## 2. The delta-scoped tlnet verification

Vocabulary from sync-with-dante.md: `RUN/upstream.txt` is dante's listing normalised to
`path<TAB>size<TAB>mtime`, `LC_ALL=C` sorted, path first so a whole-line `comm` and a
path-only `comm` sort the same way; `RUN/applied.txt` is the state file (the same format, what the
bucket holds and has verified); `RUN/changed.txt` is `comm -13 applied upstream`, the
delta; `RUN/deleted.txt` is the path-only `comm -23`. `staging/` holds one batch, under
CTAN's own paths, so the tlnet tree is at `staging/systems/texlive/tlnet/`.

### Which tlpdb a container is checked against

"The container checksum of every container in the delta" is checked against a tlpdb. Which
one? If `archive/foo.tar.xz` is in batch 1 and `tlpkg/` is in the last batch, the run has
no tlpdb on disk when it hashes `foo`, and the tlpdb in the bucket is the old one, whose
checksum for `foo` is the old revision. The check would fail on every real update.

So the tlpdb is fetched first and uploaded last. A `manifest` step runs once per run,
before any batch, whenever `changed.txt` has a line under `systems/texlive/tlnet/`:

```sh
# Taskfile: manifest (internal). 3 MB from dante; the keyring and the signature come with it.
rsync -rLt {{.RSYNC_TIMEOUTS}} --files-from=- {{.SOURCE}} {{.RUN}}/manifest/ <<EOF
systems/texlive/tlnet/tlpkg/texlive.tlpdb.xz
systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512.asc
systems/texlive/tlnet/tlpkg/gpg/pubring.gpg
EOF
```

Checks 1 to 3 run on `RUN/manifest/`; checks 4 and 5 run per batch against the tlpdb in
`RUN/manifest/`. When the last batch lands, the `tlpkg/` files in `staging/` must be byte
for byte the ones in `RUN/manifest/` (`cmp`), or dante moved between the manifest and the
batch and the run fails; the next run retries. This is the same "mid-update" class as
today's rsync exit 23.

### The five checks, restated for a delta

1. **sha512 of the tlpdb, and of the root installers and updaters.** Unchanged, on
   whatever is local: the manifest for the tlpdb; `staging/systems/texlive/tlnet/` for any
   `install-tl*.sha512` or `update-tlmgr-latest.*.sha512` in the batch. A `.sha512` in the
   delta without its target in the same batch cannot be checked; the batch planner keeps a
   file and its `.sha512` and `.sha512.asc` in one batch (sort order already does this;
   the planner must not split a directory across a batch boundary between them).

   ```sh
   cd {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg && xz -dk texlive.tlpdb.xz && shasum -a 512 -c texlive.tlpdb.sha512
   ```
   Observed today on `staging/tlpkg`: `texlive.tlpdb: OK`.

2. **Signature, `GOODSIG` and the `VALIDSIG` pin.** Unchanged:

   ```sh
   gpgv --status-fd 1 --keyring {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg/gpg/pubring.gpg texlive.tlpdb.sha512.asc texlive.tlpdb.sha512 \
   | awk '/^\[GNUPG:\] GOODSIG /{g=1} /^\[GNUPG:\] VALIDSIG .* {{.TL_KEY}}$/{v=1} END{exit !(g&&v)}'
   ```
   Observed today:
   ```
   [GNUPG:] GOODSIG 4CE1877E19438C70 TeX Live Distribution <tex-live@tug.org>
   [GNUPG:] VALIDSIG D8F2F86057A857E42A88106A4CE1877E19438C70 2026-08-25 1787702012 0 4 0 1 10 01 C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC
   ```
   `gpgv --keyring` reads the file as given. `gpg --keyring` does not: GnuPG 2.5 prints
   `Specified keyrings are ignored due to option "use-keyboxd"` and lists nothing. Use
   `gpgv` to verify and `gpg --show-keys FILE` to inspect; never `gpg --keyring`.

3. **`.xz` byte match.** Unchanged: `xz -dc texlive.tlpdb.xz | cmp - texlive.tlpdb`.
   Observed: matches.

4. **No versioned container.** The exclude `*.r[0-9]*.tar.xz` is applied to
   `upstream.txt` before the `comm`, so a versioned name can only reach `staging/` if the
   exclude is broken. Keep the test, on the batch:
   `test -z "$(find staging/systems/texlive/tlnet/archive -name '*.r[0-9]*.tar.xz' 2>/dev/null | head -1)"`.
   In the listing today: 14,872 versioned tlnet containers, 6.62 GB, all excluded; 14,872
   unversioned ones remain, one per tlpdb entry. tlcontrib's 261 versioned containers
   (0.47 GB) are excluded the same way, so the test covers its `archive/` too.

5. **Every container the tlpdb names exists and matches.** Two halves.

   **Exists.** The question is what the bucket holds when the new tlpdb becomes visible,
   which is: the state file, plus this run's delta (every earlier batch is already in the
   state by the time `tlpkg/` lands, because `tlpkg/` is in the last batch), minus this
   run's deletions. Not dante's listing. Two cases separate them:

   - A container upstream whose upload failed in an earlier run is in the listing and not
     in the state; the listing check passes, the state check fails. The state check is
     right: a client would 404.
   - A container the new tlpdb names that upstream deleted in the same hour is in the
     state, not in the listing, and in `deleted.txt`. Without subtracting deletions the
     state check passes and the run deletes a named container. Subtract them.

   ```sh
   T=systems/texlive/tlnet/archive/
   awk '/^name /{n=$2} /^containerchecksum /{print n".tar.xz"} /^doccontainerchecksum /{print n".doc.tar.xz"} /^srccontainerchecksum /{print n".source.tar.xz"}' \
     {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg/texlive.tlpdb | LC_ALL=C sort -u > {{.RUN}}/named.txt
   cut -f1 {{.RUN}}/applied.txt {{.RUN}}/changed.txt | grep "^$T" | sed "s|^$T||" | LC_ALL=C sort -u \
     | LC_ALL=C comm -23 - <(grep "^$T" {{.RUN}}/deleted.txt | sed "s|^$T||" | LC_ALL=C sort -u) > {{.RUN}}/have.txt
   LC_ALL=C comm -23 {{.RUN}}/named.txt {{.RUN}}/have.txt > {{.RUN}}/missing.txt
   test ! -s {{.RUN}}/missing.txt || { echo "tlpdb names $(wc -l < {{.RUN}}/missing.txt) containers the bucket will not hold:"; head {{.RUN}}/missing.txt; exit 1; }
   ```

   Run today with the listing standing in for `applied ∪ changed`: 14,872 named, 14,872
   present, 0 missing, 0 orphans (`comm -13` the other way). Simulation of a bucket that
   lost one container:

   ```
   $ LC_ALL=C comm -13 <(echo a2ping.tar.xz) upstream-archive.txt > state-archive.txt
   $ LC_ALL=C comm -23 named.txt state-archive.txt
   a2ping.tar.xz
   ```

   **`tlpkg/` in the delta with no `archive/`** (a tlpdb whose containers are all already
   in the bucket, the shape of most days after a quiet hour): `changed.txt` contributes
   nothing under `archive/`, `have.txt` is the state minus deletions, and the check passes
   exactly when the bucket has every named container. Adding `a2ping.tar.xz` back to the
   simulated delta makes `missing.txt` empty again (`comm` over the union: 0 lines). Both
   directions were run.

   **Match.** Only containers in the batch can be hashed:

   ```sh
   cd staging/systems/texlive/tlnet   # the batch may hold any subset of archive/
   awk '/^name /{n=$2} /^containerchecksum /{print $2"  archive/"n".tar.xz"} /^doccontainerchecksum /{print $2"  archive/"n".doc.tar.xz"} /^srccontainerchecksum /{print $2"  archive/"n".source.tar.xz"}' \
     {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg/texlive.tlpdb \
   | grep -F -f <(find archive -type f | sed 's|^|  |') | shasum -a 512 -c --quiet
   ```

   (`grep -F -f` restricts the list to files present; the `--quiet` `-c` fails on any
   mismatch or missing file, as today.) Today on the five containers in `staging/archive`:
   all `OK`; on the full list, 14,867 `FAILED open or read` and exit 1, which is the
   partial-archive failure CLAUDE.md promises.

   Containers not in the batch were hashed when they were uploaded: every `archive/` key in
   the state file entered it through a batch, every batch ran this check against the
   manifest tlpdb of its run, and a key enters the state only after its batch's upload
   succeeded. The bytes in the bucket are the bytes that were hashed, unless something
   other than this pipeline wrote to the bucket (section 5, R2 token).

   The hole: a container whose content changed upstream with the same size and the same
   mtime. `rsync -rL --list-only` reports size and mtime and nothing else, so the delta
   would not contain it, the bucket keeps the old bytes, and the new tlpdb carries a new
   checksum for it. `tlmgr` would reject the old bytes (checksum mismatch), so the mirror
   is broken but not dangerous. `rsync -a` on a conventional mirror misses it the same way
   (quick check is size and mtime), so no CTAN mirror catches this; dante's tlnet build
   writes new files with the build's mtime and new revisions get new versioned names, so it
   is not expected. One cheap check closes it anyway: diff the new tlpdb's checksums
   against the tlpdb the bucket already serves, and fail if a checksum changed for a
   container the delta does not contain.

   ```sh
   aws s3 cp --no-progress s3://tlnet/systems/texlive/tlnet/tlpkg/texlive.tlpdb.xz - | xz -dc > {{.RUN}}/old.tlpdb   # 1 GetObject, 2.7 MB
   sums() { awk '/^name /{n=$2} /^containerchecksum /{print n".tar.xz "$2} /^doccontainerchecksum /{print n".doc.tar.xz "$2} /^srccontainerchecksum /{print n".source.tar.xz "$2}' "$1" | LC_ALL=C sort; }
   LC_ALL=C comm -13 <(sums {{.RUN}}/old.tlpdb) <(sums {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg/texlive.tlpdb) | cut -d' ' -f1 \
     | LC_ALL=C comm -23 - <(cut -f1 {{.RUN}}/changed.txt | grep '^systems/texlive/tlnet/archive/' | sed 's|.*/||' | LC_ALL=C sort) > {{.RUN}}/silent.txt
   test ! -s {{.RUN}}/silent.txt || { echo "tlpdb changed the checksum of $(wc -l < {{.RUN}}/silent.txt) containers the listing calls unchanged:"; head {{.RUN}}/silent.txt; exit 1; }
   ```

   Cost: one Class B per run that touches tlnet. If it ever fires, the fix is to delete
   the listed keys from the state file so the next run refetches them.

### The same checks for tlcontrib and the ISO

tlcontrib: the same five checks with `systems/texlive/tlcontrib/` for the path and
`TLCONTRIB_KEY: F7D8A92826E316A19FA0ACF06CACA448860CDC13` for the pin. The tlcontrib
`tlpkg/` has no `gpg/` directory (5 files, listed above), so the keyring is tlnet's, which
holds the key. Its 261 versioned containers (0.47 GB) are excluded like tlnet's. Cost:
tlcontrib changed last on 2026-04-06, so the checks run a few times a
year. The pin is a second thing to rotate; Preining's primary and subkeys currently expire
2027-07-13, the same day as the TeX Live subkey.

ISO: the two `.sha512.asc` files verify against `TL_KEY` whenever they are in the delta
(`GOODSIG` + `VALIDSIG` pin, as for the installers). When an `.iso` is in the delta the
batch is a lone-file batch (over 4.995 GiB), and `shasum -a 512 -c` on the dated `.sha512`
runs before the multipart upload; `texlive.iso` has no `.sha512` of its own and is checked
against the dated hash (`shasum -a 512 texlive.iso | cut -d' ' -f1` equal to the first
field of `texlive2026-20260301.iso.sha512`). Measured here: 1 GiB hashed in 1.9 s, so
6.78 GB in about 12 s; call it under a minute on the runner. Once a year.

### Where `verify` sits in the run

```
list -> guard(listing) -> state -> comm -> manifest(+checks 1-3, checksum diff) ->
  for each batch: fetch -> verify(batch: 4, 5-match, installers/ISO/tlcontrib as present) -> upload -> purge -> state
  -> before the last (tlpkg) batch's upload: verify 5-exists, cmp manifest vs staging
-> deletions -> smoke -> report -> ping
```

A batch that fails verification stops the run with the state as of the previous batch, the
same at-least-once model as any other failure; nothing verified-and-uploaded is undone.

## 3. The TeX Live key

| Fact | Value | How verified today |
|---|---|---|
| Pin (`TL_KEY`) | `C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC` | tug.org/texlive/verify.html shows `Primary key fingerprint: C78B 82D8 C795 12F7 9CC0 D7C8 0D5E 5D91 06BA B6BC`, key ID `0D5E5D9106BAB6BC` |
| Signing subkey | `D8F2F86057A857E42A88106A4CE1877E19438C70` | same page, `Subkey fingerprint`; the `VALIDSIG` line above |
| Subkey expiry | 2027-07-13 (`1815486861`) | same page: `sub rsa2048 2016-03-19 [S] [expires: 2027-07-13]`; `gpg --show-keys` on the tree's `pubring.gpg` and on `https://tug.org/texlive/files/texlive.asc` both give `1815486861` |
| Days left | 320 from 2026-08-26 | command below |
| Keyring in tree | `tlpkg/gpg/pubring.gpg`, 131,459 bytes, mtime 2026-01-19 | listing; holds the TL key plus Karl Berry's two keys and Norbert Preining's key |
| Rotation procedure | yearly extension of the subkey, "16m", after the next release; new `pubring.gpg` committed to the TL repository and `texlive.asc` on tug.org | `tlpkg/gpg/tl-key-extension.txt` in the tree (r79592, 2026-07-05) |

Note the keyring is the tree's and the tree is the thing being verified: an attacker who
controls the tree controls the keyring, which is why the pin, not the `GOODSIG`, is the
check. `GOODSIG` is still required because an expired or revoked key yields `VALIDSIG` with
exit 0 (also confirmed by `tl-key-extension.txt`: "exit status is zero even with expired
keys").

Days to expiry, pinned to the TL primary so Preining's signing subkey and the revoked
ones are ignored (`$2!="r"`), portable awk (no `strftime` on macOS):

```sh
gpg --show-keys --with-colons {{.RUN}}/manifest/systems/texlive/tlnet/tlpkg/gpg/pubring.gpg 2>/dev/null \
| awk -F: -v now="$(date +%s)" '$1=="pub"{k=$5} $1=="sub" && k=="0D5E5D9106BAB6BC" && $12~/s/ && $2!="r" {printf "%d\n", ($7-now)/86400}'
```

Output today: `320`. `report` prints it in the Signature row ("signing subkey expires in
320 days") every run that has a manifest, and the row is copied into the ping body
(monitoring.md). No threshold fails the run: a warning that fails the pipeline is a
mirror outage on purpose, and the failure a month later is the real alert.

On the expiry day: upstream normally publishes the extended key months ahead (the current
extension landed 2026-01-19 for a 2027-07-13 expiry). If it has, nothing happens; the
mirror's keyring is upstream's. If it has not, `gpgv` reports `EXPKEYSIG` instead of
`GOODSIG`, the awk fails, `verify` fails, no batch after the manifest is uploaded, the run
is red, GitHub emails, healthchecks fires after the grace period, and the mirror serves
yesterday's tree until upstream ships the new keyring. `tlmgr` clients pointed at any
mirror would accept the expired signature; this mirror is stricter and stops. There is
nothing to do locally except wait, unless the *pin* changed, which is the one manual
rotation: edit `TL_KEY` only against `https://www.tug.org/texlive/verify.html`, never
against the tree.

Rotation of the pin itself (a new primary key) has not happened since 2016 and would be
announced on tex-live@tug.org; the `tl-key-extension.txt` procedure extends the existing
subkey rather than replacing the primary, so `TL_KEY` is expected to stay.

## 4. The symlink-inflation guard and the storage ceiling

### The numbers

```sh
# stored set: dereferenced listing minus tlnet's versioned containers
awk 'function sz(){return (NF==4)?0:$2} $1 ~ /^-/ {n++; s+=sz(); if ($NF ~ /^systems\/texlive\/tlnet\/archive\/.*\.r[0-9]+\.tar\.xz$/){vn++; vs+=sz()}}
     END {printf "deref %d %.2f GB; versioned %d %.2f GB; stored %d %.2f GB\n", n, s/1e9, vn, vs/1e9, n-vn, (s-vs)/1e9}' ctan-list-deref.txt
awk 'function sz(){return (NF==4)?0:$2} $1 ~ /^-/ {n++; s+=sz()} $1 ~ /^l/ {l++} END {printf "real %d %.2f GB, %d symlinks\n", n, s/1e9, l}' ctan-list-nolink.txt
```

| Measure | Objects | Bytes |
|---|---:|---:|
| Real files (`rsync -r`) | 352,357 | 68.98 GB |
| Symlinks | 24,788 | |
| Dereferenced (`rsync -rL`) | 511,027 | 139.63 GB |
| tlnet versioned containers (excluded) | 14,872 | 6.62 GB |
| tlnet root `update-tlmgr-r*` (excluded, as today) | 6 | 0.01 GB |
| **Stored set** | **496,149** | **132.99 GB** |
| Inflation factor | 1.41x | 1.93x |
| Objects over 4.995 GiB | 5 | 34.1 GB |
| Worst day (2026-03-01, TeX Live release) | 1,774 | 20.94 GB |
| Second (2026-03-24, MacTeX) | 293 | 14.01 GB |
| Third (2021-03-17) | 42 | 2.32 GB |

The stored set is 496,149 objects and 132.99 GB; the 2026-03-01 release day is 20.94 GB.
The worst-day
figures count files carrying that mtime in today's listing, so they are a floor for what
was fetched that day (files since replaced do not appear).

Two parsing hazards in the listing, to be handled by the normaliser in sync-with-dante.md:
523 zero-size regular files print an empty size column
(`-rw-rw-r--              2026/03/01 13:04:18 systems/texlive/tlnet/TEXLIVE_2026`), and
25 paths contain spaces. Two files have both (`macros/latex/contrib/akktex/documentation/still to do`
and its `latex2e` alias), so "four fields means blank size" is wrong: detect a blank size
by a date-shaped second field, then take the path as everything after the time. The
`guard` below was run against a listing normalised that way (`path<TAB>size<TAB>mtime`):
496,149 objects, 132.99 GB, `timestamp` present, no object over `MAX_OBJECT_BYTES`.

### The check

Runs on `RUN/upstream.txt` after the exclude and before the state file is even read, so a
broken listing costs nothing:

```sh
guard:
  desc: Refuse a listing that would overflow the ceiling or that is too short to be CTAN
  cmds:
    - |
      n=$(wc -l < {{.RUN}}/upstream.txt); gb=$(awk -F'\t' '{s+=$2} END{printf "%.2f", s/1e9}' {{.RUN}}/upstream.txt)
      echo "upstream lists $n objects, $gb GB; ceiling {{.CEILING_OBJECTS}} objects, {{.CEILING_GB}} GB"
      test "$n" -le {{.CEILING_OBJECTS}} || { echo "GUARD: $n objects exceed CEILING_OBJECTS={{.CEILING_OBJECTS}}; nothing fetched"; exit 1; }
      awk -v gb="$gb" -v c={{.CEILING_GB}} 'BEGIN{exit !(gb+0 <= c+0)}' || { echo "GUARD: $gb GB exceeds CEILING_GB={{.CEILING_GB}}; nothing fetched"; exit 1; }
      grep -q "^timestamp$(printf '\t')" {{.RUN}}/upstream.txt || { echo "GUARD: listing has no root timestamp; refusing to treat it as CTAN"; exit 1; }
      if test -s {{.RUN}}/applied.txt; then
        m=$(wc -l < {{.RUN}}/applied.txt)
        test "$n" -ge $((m * 9 / 10)) || { echo "GUARD: $n objects is under 90% of the $m in the state; a short listing would delete $((m - n)) keys"; exit 1; }
      fi
    - awk -F'\t' -v max={{.MAX_OBJECT_BYTES}} '$2 > max {print "GUARD: " $1 " is " $2 " bytes, larger than the runner can stage"; f=1} END{exit f}' {{.RUN}}/upstream.txt
```

Variables: `CEILING_OBJECTS: 650000`, `CEILING_GB: 200` (133 GB plus a release
day plus a year of growth; $2.85/month at the ceiling, cost-estimates.md), `MAX_OBJECT_BYTES:
12000000000` (the 14 GB runner disk less headroom; the largest object today is 6.87 GB).
The ceiling is a bound on the **mirror total**, not on the delta: on 2026-03-01 the delta
was 20.94 GB and the total went from ~112 GB to ~133 GB, which passes. A ceiling on the
delta would have failed the release day, which is the one day the mirror must not skip.
There is no delta bound at all: the seed's delta is the whole tree.

Is the 4 GB batch bound a guard? It bounds disk, not storage: a run of thirty-four 4 GB
batches is the seed. The `MAX_OBJECT_BYTES` line is the only per-file guard and it is a
disk guard too. Storage has exactly one guard, the listing total, and the daily reconcile's
bucket listing is the second opinion (monitoring.md: `report` fails at the ceiling from the
Analytics API as well).

### Empty or truncated listings

`rsync --list-only` produces the list from the daemon's file-list stream; a broken stream
is exit 10, 12 or 30 (errcode.h, and the man page's exit values), not exit 0 with a short
list. The rsync retry loop covers those. A *complete* listing that is wrong with exit 0 is
possible in two ways the mirror cannot distinguish from the truth: dante's module points at
a partial tree (a rebuild, a wrong `path` in `rsyncd.conf`), or a daemon-side `exclude`
appears. Both look like mass deletion. Today's `stale` has `test -s local.txt` for the
same reason; the full-mirror equivalent is the two lines above: `timestamp` must be
present (root, 186 bytes, touched hourly), and the count must be at least 90% of the state
(446,534 against a 496,149-line state). The 90% figure allows the largest plausible real
shrink (the 24,611 alias copies of a directory symlink being removed upstream is 5%). If
the guard fires on a real change, raise the state by hand: `aws s3 rm` the state file and
let the next run reseed, or edit the threshold in a PR. Belt and braces: `deleted.txt`
over 10% of the state fails the deletion step (errors-and-issues.md).

## 5. Trust boundaries and threat model

| Boundary | Trust placed | What a compromise can do | What the mirror does about it | What it does not do |
|---|---|---|---|---|
| dante (`rsync.dante.ctan.org`) | authoritative for every byte | serve altered unsigned files; serve an old signed tlnet (rollback); withhold updates | tlnet, tlcontrib and the ISO cannot be altered without the pinned keys; everything else is copied as served | no freshness check on the tlpdb (a signed tlpdb from last month verifies); no check of unsigned content; a compromised dante is a compromised mirror for 99% of the tree, exactly as for every CTAN mirror |
| Transport, dante to runner | `rsync://` in the clear | same as a compromised dante, for one run; alter the listing (sizes, mtimes, paths) | signatures as above; the guard bounds the listing; a listing that adds a path outside CTAN's tree is still stored under it | `CTAN.sites` (2026-08-25) and the register page offer dante over `rsync://` only; no rsync-over-SSH or HTTPS is published for mirrors, so there is nothing to switch to within the tool list |
| R2 token (`R2_ACCESS_KEY_ID`/`SECRET`) | Object Read & Write, scoped to bucket `tlnet` | overwrite or delete any object (deletion is exercised by `aws s3 rm` today, so the scope includes it); write objects outside CTAN's paths | Object scope cannot create or delete buckets or change the custom domain or CORS (R2 token docs). Recovery: delete `.state/applied.txt.xz`, roll the token in the R2 dashboard, replace the two secrets, and the next run reseeds every key from dante (~34 batches, ~500k Class A, inside the free month). The daily reconcile catches wrong sizes and unknown keys; it cannot catch a same-size overwrite, hence the reseed | tlmgr clients reject tampered containers and tlpdbs on the client; browsers downloading anything else trust the bucket |
| Cloudflare API token (`CF_API_TOKEN`, `CF_ZONE_ID`) | Zone `ijosh.com`: Cache Purge, Cache Rules Edit (API name `Cache Settings Write`), Transform Rules Edit; Account: Account Analytics Read | purge, or rewrite the cache rule: more origin fetches, Class B at $0.36 per million above the free 10M, with no ceiling (Cloudflare evicts at will; the edge cache is off by default with `CACHE: off`, so a rule change can only add caching or remove it: cost-estimates.md, caching.md); rewrite the transform rule to map `/` or any path to another key **in the bucket** | zone-restricted so `ijosh.com`'s other zones and the account are out of reach; rolled from the dashboard, two secrets replaced. Client IP filtering is not usable (GitHub runner IPs vary). A TTL is optional; a silent expiry is a failed `rules` step at the head of the run, before any upload | cannot read or write R2, DNS, WAF or the origin; a transform rule cannot send users off-zone |
| GitHub Actions | the repository, its Actions allowlist, GitHub-hosted runners | a malicious PR cannot run with secrets: `check.yml` is `pull_request` and "With the exception of `GITHUB_TOKEN`, secrets are not passed to the runner when a workflow is triggered from a forked repository" (docs.github.com, fetched today); Dependabot-triggered workflows get no secrets either | `permissions: contents: read` on both workflows; actions pinned to full SHAs, allowlist GitHub-owned plus `go-task/*`; CodeQL on the workflows; no `pull_request_target` anywhere and none is to be added; `workflow_dispatch` requires write access; runners are ephemeral VMs, nothing persists between runs except what `.state/` holds in the bucket | a collaborator with write access can change the Taskfile; branch protection (section 8) makes that a reviewed PR |
| The state file `.state/applied.txt.xz` | public-readable at `https://ctan.ijosh.com/.state/applied.txt.xz` | nothing: it is dante's own listing, already public | `.state/` and `.site/` are the two reserved prefixes at the bucket root outside CTAN's paths; the constraint "objects stay under CTAN's own paths" is amended to name them, and the reconcile ignores both (taskfile-architecture.md) | |
| Landing page | CTAN's own `index.html` at the root, 10,366 bytes dated 2020-03-31, the same file every mirror serves | an XSS in a static page nobody logs in to | nothing to do; the file is upstream's. The Transform Rule `/` to `/index.html` stays; the 110 other `index.html` files in the tree are served as ordinary files (R2 has no directory index) | official-mirror-and-url.md owns the collision with the repository's own `site/index.html` |
| Users | `tlmgr`/`install-tl`: verify the tlpdb signature and every container checksum on the client. Browsers and `curl`: trust the mirror | a user who downloads `MacTeX.pkg` from the mirror trusts dante, the transport, R2 and Cloudflare, plus Apple's installer signature, plus the `.sha512` if they check it against tug.org | same exposure as any CTAN mirror; the promises in section 9 say so | |

Things checked and found harmless: one `.htaccess` in the tree
(`systems/texlive/Images/test/.htaccess`, 36 bytes: `IndexIgnore HEADER.html README.html`);
34 dot-files (`.index.html`, `.gdb_history`, `.project`), all upstream content; two CA
bundles under tlnet's `tlpkg/` (curl's and Mozilla::CA's), which are public root stores.

## 6. Content risks in a full mirror

**Licence-restricted material.** The listing has no top-level `nonfree/` tree. The 150
`nonfree` hits are package names: `fonts/vntex-nonfree/` (52 files, a font package whose
name declares its licence), `macros/unicodetex/latex/fontsetup-nonfree/` (21, support
files for commercial fonts the user must own), `macros/context/base/archives/context-nonfree.zip`,
`install/fonts/vntex-nonfree.tds.zip`, and their tlnet containers. `README.structure` at
the archive root describes CTAN as "a collection of freely-available material". CTAN's
licence catalogue page (`https://ctan.org/license`) returned HTTP 500 twice today, so the
exact wording of its nonfree classes is **unverified**; the upload page (fetched) requires
a declared licence and "verifies upload authorization before acceptance", and the upload
addendum and register page place no restriction on what mirrors carry. Every registered
mirror carries the same tree. Also checked: `obsolete/support/ghostscript/AFPL` (AFPL
permits non-commercial redistribution), `obsolete/support/xpdf` (GPL), `systems/win32/`
MiKTeX binaries and `systems/mac/mactex` (both redistributable by design, they publish to
CTAN for mirroring). Nothing a hoster would object to was found by name; the search was
for `nonfree`, `restricted`, `commercial`, `shareware`, `LICENSE`-gated directories.

**DMCA.** R2 is a Cloudflare hosting product, and for hosted content Cloudflare follows
"the notice-and-takedown process set forth in the Digital Millennium Copyright Act"
(trust-hub/abuse-approach, fetched). The bucket owner is the publisher and receives the
notice. Response: add the key to an `EXCLUDES` list in the Taskfile (the same mechanism as
the tlnet versioned-container exclude), which drops it from `upstream.txt` so the next run
deletes it and never refetches it; tell CTAN at ctan@ctan.org, since they hold the upload
authorisation. Expected frequency: none in CTAN's history that is known here (unverified).

**robots.txt.** Not in CTAN's tree (0 hits); `https://mirror.ctan.org/robots.txt`
redirected to `mirror.clarkson.edu` and returned 404; `https://ctan.org/robots.txt`
disallows only site paths (`/json/`, `/xml/`, `/search`, `/mirrors/mirmon/`). Today
`https://ctan.ijosh.com/robots.txt` returns 200 on GET with Cloudflare's managed
"content signals" comment block and no directives, and 404 on HEAD: the zone has
Cloudflare's managed robots.txt on ("Cloudflare will prepend our managed `robots.txt`
before your existing `robots.txt`", available on all plans). Decision: store no
`robots.txt`. It is not a CTAN path; the reconcile would delete it; and crawler traffic is
Class B at $0.36 per million GETs above the free 10M, with the edge cache off by default
and no ceiling (cost-estimates.md, caching.md), visible in `report`'s analytics row
(monitoring.md). If crawlers ever show up there, the free answer is Cloudflare's bot rules
on the zone, not an object in the bucket. If CTAN ever adds a `robots.txt` to the tree,
the mirror serves it with the managed block prepended.

## 7. Path safety

Every key is a CTAN path verbatim. R2 accepts any UTF-8 key up to 1,024 bytes (R2 limits
page). What the listing contains:

| Pattern | Count | Notes |
|---|---:|---|
| `..` segment or leading `/` | 0 | `rsync --list-only` cannot emit either |
| `.well-known/` | 0 | |
| `/cdn-cgi/` | 0 | Cloudflare reserves `/cdn-cgi/` on every proxied hostname ("managed and served by Cloudflare", "cannot be modified"); no CTAN path starts with it |
| `index.html` | 111 | root plus 110 in subdirectories; served as files. The root one is CTAN's landing page |
| Longest key | 151 bytes | `macros/latex2e/contrib/ualberta/03_References/Reference_PDFs/Aldrich2017-…Strain.pdf`; longest component 90 bytes |
| Non-ASCII bytes | 0 | `perl -ne 'print if /[^\x20-\x7e]/'`: none |
| Control characters | 0 | `perl -ne 'print if /[\x00-\x08\x0b-\x1f\x7f]/'`: none |
| Space in path | 25 | e.g. `biblio/bibtex/utils/BibBuild/Bibliography Builder.fp7`; `--files-from` handles them (one path per line), URLs need `%20` |
| `%` in path | 7 (5 files) | `systems/mac/textures/fonts/AMS%2fPS VF (0.5, Uwe)`, `…/Textures%ae 2.1.2 Updater.sit`: literal `%2f` and `%ae` in the key. The canonical URL is `%252f`; whether Cloudflare's URL normalisation leaves a raw `%2f` alone is **unverified** until tested after the seed. Five 2006 Mac files |
| `#` | 3 | `language/cyrtug/#disk.00` (and the `languages/` alias, and `#disk_00.dir`); URL form `%23` |
| `&` `+` `~` `$` `=` `,` `(` `!` `@` `{` `[` `>` | 8, 228, 12, 14, 154, 196, 29, 7, 18, 1, 1, 1 | all legal keys; `+` and `&` need encoding in a query but not in a path |
| Trailing `.` | 2 | `support/qfig/qfig3ple.`, `systems/mac/textures/information/FAQ.comp.text.tex.`; fine on R2 and Linux, would be rewritten on Windows |
| Dot-files | 34 | served as files |
| Zero-size objects | 523 | legal; the listing prints them with a blank size |
| Keys differing only by case | 24 pairs | e.g. `documentation/epslatex/french/Danger.eps` and `danger.eps`, `dviware/ivd2dvi/Makefile` and `makefile`. Distinct on R2 and on the runner's ext4. **A local dry run on a case-insensitive APFS volume would merge them**, so local tests of the batch fetch must use a case-sensitive volume or exclude those 12 directories |

Commands: `grep -c '\.\./'`, `grep -c ' /'`, `grep -c '\.well-known'`, `grep -c
'cdn-cgi'`, `awk '{p=...; if (length(p)>max) ...}'`, `awk 'NF>5'`, `grep -c '%'`,
the two `perl` lines, and for case pairs `awk '{print tolower(p) "\t" p}' | sort | cut -f1 |
uniq -d | wc -l`, all over `ctan-list-deref.txt` with the path taken as fields 5 (or 4 for
blank-size lines) to the end.

Nothing in the tree collides with anything Cloudflare or R2 reserves. The only path
ambiguity is the five `%` keys, and they are unreachable today on any mirror that
percent-decodes before the filesystem lookup, so the mirror is no worse.

## 8. Repository guardrails

**Stays as is.** Actions pinned to full commit SHAs with the version in a comment, the
repository setting that rejects tags, the allowlist (GitHub-owned plus `go-task/*`),
Dependabot's weekly grouped bumps, CodeQL default setup on the workflows, private
vulnerability reporting with `SECURITY.md` linking to it, `permissions: contents: read`,
`concurrency: sync`, squash merges to `main`.

**Changes.**

- `SECURITY.md` "What this mirror guarantees" becomes section 9 below, word for word. Saying
  "only tlnet is signed" would be wrong: tlcontrib, the
  ISO checksums, MiKTeX's rpm and deb metadata and a few hundred package files are signed
  too, and the page must say which of those the mirror checks (tlnet, tlcontrib, ISO) and
  which it does not (the rest), or a reader will assume the wrong thing in both directions.
- `check.yml` keeps `task --dry sync`. The two Cloudflare JSON files are data the run
  `PUT`s at its head; a malformed file fails `rules` before any fetch, which is the safe
  failure, and the next PR fixes it. Validating JSON in CI would need a parser (`jq` or
  `python3`), which is a tool outside the list; not added. Linting the Task scripts with
  `shellcheck` is out of scope for the same reason: the scripts are YAML-embedded shell
  and `task --dry` already parses them.
- `CLAUDE.md` constraints: secrets become six (`CF_API_TOKEN`, `CF_ZONE_ID` added);
  endpoints gain `api.cloudflare.com`; "objects stay under CTAN's own paths" gains the two
  reserved prefixes `.state/` and `.site/`; the tool list is unchanged. `TL_KEY` gains `TLCONTRIB_KEY` beside it
  with the same rotation rule (only against tug.org for the first; for the second, only
  against a signature on a tlcontrib tlpdb that also verifies with the first key's tree,
  which is weak, and section 10 lists it as open).
- Branch protection on `main`, all available to a personal public repository (GitHub's
  "restrict who can push" is organisation-only, the rest is not): require a pull request,
  require the `check` status, require linear history, include administrators. Dependabot
  PRs still need a click; hourly runs are not repository activity, so the 60-day schedule
  rule is unchanged and Dependabot's merges remain the heartbeat (monitoring.md).

## 9. What a user can rely on

For `README.md` and `SECURITY.md`:

> **What this mirror promises.** On every run (scheduled hourly; starts drift, see
> monitoring.md), before anything is published:
>
> - `systems/texlive/tlnet/`, the repository `tlmgr` and `install-tl` use, is verified.
>   `texlive.tlpdb` is checked against its SHA-512 and its GPG signature with the TeX Live
>   key fingerprint `C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC` pinned in the repository; an
>   expired or revoked key is rejected. `texlive.tlpdb.xz` must decompress to the verified
>   tlpdb byte for byte. Every package container that changed is checked against the
>   checksum in the signed tlpdb, and the tlpdb is never published naming a container the
>   mirror does not hold. The installers and `update-tlmgr` scripts at the tlnet root are
>   checked the same way.
> - `systems/texlive/tlcontrib/` is verified the same way with its maintainer's key.
> - The TeX Live ISO checksums are signature-checked, and each ISO is hashed against them
>   when it changes.
> - A tree that fails any check is not published; the previous good copy stays live.
>
> **What it does not promise.** Everything else on CTAN is copied from the master
> `rsync.dante.ctan.org` exactly as served, with no signature to check: CTAN does not
> sign it. Checksum files elsewhere in the tree (`systems/mac/mactex/*.sha512`, `.sig`
> files, `MD5SUMS`) are copied but not enforced, because they are unsigned and at least
> one is stale upstream. If you download an installer or a package from a browser, verify
> it as you would from any mirror: for TeX Live, follow
> https://www.tug.org/texlive/verify.html against a key you fetched from a keyserver, not
> from this mirror.
>
> `tlmgr` repeats every signature and checksum check on your machine, so a tampered
> mirror is rejected there too. Uploads are not atomic: containers land before the tlpdb
> that names them, so a `tlmgr` run that overlaps a publish can see a checksum error; run
> it again.

## 10. Open questions

- Whether pinning `TLCONTRIB_KEY` is worth the second rotation. The alternative is to copy
  tlcontrib as-is like MacTeX; tlmgr verifies on the client either way. Decided "verify"
  here because it is the same code path, but a key change by its maintainer would stall the
  whole mirror for a side repository of 530 files.
- The exact wording of CTAN's licence classes and whether any material carries a
  no-mirroring restriction: `https://ctan.org/license` returned 500 today. Ask on the
  mirror maintainers' list when registering.
- Whether Cloudflare's URL normalisation lets a request for a key containing a literal
  `%2f` reach R2 unchanged (five keys under `systems/mac/textures/`). Test after the seed;
  the answer changes nothing in the pipeline.
- Whether `MacTeX.pkg.sha512` being stale is a one-off or a habit; if upstream fixes it,
  hashing MacTeX would be possible but still unsigned, so the decision would not change.
- Whether the daily reconcile should sample-hash a few random `archive/` keys read back
  through the domain against the tlpdb (a cheap probe for same-size corruption or a
  tampered bucket). Belongs to monitoring.md if wanted; three GETs an hour.
- Whether to give `CF_API_TOKEN` a TTL. A silent expiry fails `rules` at the head of the
  run (safe, loud); a token without TTL is one fewer calendar entry. Not decided.
- Belongs to sync-with-dante.md: the normaliser must handle blank-size lines (523) and
  paths with spaces (25), two of which are both, so the blank-size test must look at the
  date-shaped second field rather than the field count; the batch planner must keep a file with its `.sha512` and
  `.sha512.asc` in one batch and `tlpkg/` last. Belongs to taskfile-architecture.md: the
  `manifest` task, the `EXCLUDES` variable, and `.state/` being ignored by the reconcile.
  Belongs to monitoring.md: the days-to-expiry and silent-change lines in `report`.
  Belongs to official-mirror-and-url.md: which `index.html` is served at the root.

## 11. Sources

Fetched 2026-08-26:

- https://www.tug.org/texlive/verify.html (fingerprint, subkey, expiry 2027-07-13; via curl, the page refuses non-browser user agents)
- https://www.tug.org/texlive/files/texlive.asc (key file; subkey expiry `1815486861` matches the tree's keyring)
- https://ctan.org/mirrors/register (saved copy `SCRATCH/register.html`: rsync-only master, Apache `DirectoryIndex disabled` advice)
- https://ctan.org/mirrors, https://ctan.org/help/mirror-selection, https://ctan.org/ctan, https://ctan.org/upload, https://ctan.org/file/help/ctan/CTAN-upload-addendum (no mirror content policy; upload licence requirement)
- https://ctan.org/help/mirror (404), https://ctan.org/license and https://www.ctan.org/license (500)
- https://ctan.org/robots.txt, https://mirror.ctan.org/robots.txt (404 at mirror.clarkson.edu), https://ctan.ijosh.com/robots.txt
- https://mirror.ctan.org/CTAN.sites, `README.structure`, `README.uploads`, and the small signed files listed in section 1
- https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions (fork PRs and Dependabot get no secrets)
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows (`pull_request` from forks, `pull_request_target` warning, 60-day schedule rule)
- https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches
- https://developers.cloudflare.com/r2/api/tokens/ (permission levels, bucket scoping)
- https://developers.cloudflare.com/fundamentals/api/get-started/create-token/ (zone restriction, client IP filtering, TTL)
- https://developers.cloudflare.com/fundamentals/reference/cdn-cgi-endpoint/
- https://developers.cloudflare.com/bots/additional-configurations/managed-robots-txt/
- https://www.cloudflare.com/trust-hub/abuse-approach/ (DMCA for hosted content)
- Cloudflare API token permissions table (saved copy `SCRATCH/cf-perms.md`) and R2 limits (`SCRATCH/r2_platform_limits.md`: key length 1,024 bytes)
- rsync manual, exit values (`SCRATCH/rsync.txt`)
- Local: `staging/tlpkg/texlive.tlpdb` (8,132 packages; 8,128 / 4,722 / 2,022 run, doc and source checksums), `staging/tlpkg/gpg/pubring.gpg`, `staging/tlpkg/gpg/tl-key-extension.txt`, `SCRATCH/ctan-list-deref.txt`, `SCRATCH/ctan-list-nolink.txt`
