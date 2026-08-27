# Synchronizing with dante

How the bucket stays in step with `rsync://rsync.dante.ctan.org/CTAN/` every hour with no
local copy of the tree. The design is a list-diff: one `rsync --list-only` of the master,
one `comm` against the listing the last run left in the bucket, one `rsync --files-from`
per batch into an empty `staging/`, one `aws s3 cp --recursive` per batch, deletions last,
state written last. This file settles the contract with upstream, the symlink rules, the
exact commands, the consistency argument, the daily reconcile, the alternatives, the time
budget and the load on dante. Verification of the bytes belongs to
`verification-and-security.md`; the cache purge to `caching.md`; the Taskfile shape to
`taskfile-architecture.md`.

All measurements are from the two listings taken 2026-08-26 (local files
`ctan-list-deref.txt`, `rsync -rL --list-only`, and `ctan-list-nolink.txt`,
`rsync -r --list-only`; `SCRATCH` below is their directory). One correction applies to
every time in them: the listing client was Apple's `openrsync` on a machine in
`America/Los_Angeles`, and `rsync` prints mtimes in the client's local zone (rsync
`util1.c`, `timestring()`, uses `localtime_r`). The files carry PDT; every hour-of-day
figure below has been converted to UTC.

## The design in one paragraph

```
list      TZ=UTC rsync -rL --list-only --no-h  dante  > RUN/upstream.raw   (~7 s, exit 0 or 23)
          awk normalise -> RUN/U        one line per regular file: path<TAB>size<TAB>mtime
state     aws s3 cp s3://tlnet/.state/applied.tsv.xz - | xz -dc > RUN/S   (or empty on the first run)
diff      comm -13 S U        -> RUN/changed   (added or changed lines; their paths are the fetch list)
          comm -23 <paths S> <paths U> -> RUN/deleted
plan      awk over RUN/changed -> RUN/batch.1 .. batch.N   (<= 4 GB each, big files alone,
          the decision files in batch N by themselves; an hourly run works at most
          MAX_BATCHES=4 of them, the seed and a release day take several hours)
each batch
          rsync -Lt --no-h -s --ignore-missing-args --timeout=300 --contimeout=60 \
                --files-from=RUN/batch.i  dante  staging/
          verify what landed (tlnet only), drop what fails
          aws s3 cp --recursive staging/ s3://tlnet/        (N PutObject, no listing)
          purge those URLs if the edge cache is on          (off by default; caching.md)
          S := lines of U for every path now in staging, merged over S; write .state
          empty staging/
delete    aws s3api delete-objects, 1,000 keys per call; S := S minus deleted; write .state
smoke, report, ping
```

## 1. The upstream contract

### What dante publishes at the root

From `SCRATCH/ctan-list-nolink.txt` (times converted to UTC):

```
awk '$5 !~ /\//' SCRATCH/ctan-list-nolink.txt
```

| Entry | Type | Size | mtime (UTC) | Role |
|---|---|---:|---|---|
| `timestamp` | file | 186 | 2026-08-27 00:02:01 | mirmon reads this; content is `irony.dante.de` and `2026-08-26-23-02` |
| `FILES.byname`, `.gz` | file | 26,740,646 / 2,811,339 | 2026-08-26 23:20:14 | `date \| size \| path`, one line per real file |
| `FILES.last07days` | file | 584,326 | 2026-08-26 23:20:17 | same format, last 7 days, 8,229 lines today |
| `CTAN.sites` | file | 18,398 | 2026-08-25 23:21:07 | mirror list; `README.mirrors` is a symlink to it |
| `README.structure`, `README.uploads` | file | 3,547 / 324 | 2016 / 2013 | static |
| `index.html` | file | 10,366 | 2020-03-31 | **CTAN's own root page**; see Open questions |
| `tds.zip` | symlink | 12 | 2022-09-22 | alias |
| `bibliography`, `digests`, `documentation`, `languages`, `tds` | symlink | 6, 12, 4, 8, 8 | 2007 / 2022 | directory aliases; the listing hides targets, but the size column is the target's length and only `biblio`, `info/digests`, `info`, `language`, `info/tds` fit those lengths among real directories |
| `biblio` ... `web` | directory | | | the tree |

`timestamp` content and the `FILES.*` format were read from `https://mirror.ctan.org/` on
2026-08-27 00:15 UTC (the mirror served `2026-08-26-23-02`, one hour behind the listing's
`00:02:01` mtime, which is that mirror's lag, not dante's). The `timestamp` line format is
`year-month-day-hour-minute`; the daemon touches it at `:02` every hour (one mtime
observed at `:02:01`, and a content stamp ending `-23-02` read back from a mirror an hour
later; an hourly touch at `:02` is the only reading consistent with both). `FILES.*` regenerate once
a day around 23:20 UTC (one observation each; `CTAN.sites` the day before at 23:21).

### When files change

Hour of day, UTC, files touched in the 30 days to 2026-08-27 (stored set, see section 3):

```
awk -f SCRATCH/utc.awk -f SCRATCH/h30.awk SCRATCH/state.tsv | sort
```

| UTC hour | files | MB | UTC hour | files | MB |
|---:|---:|---:|---:|---:|---:|
| 00 | 45 | 31.7 | 12 | 874 | 218.8 |
| 01 | 140 | 40.5 | 13 | 580 | 124.8 |
| 02 | 1,433 | 70.6 | 14 | 402 | 303.8 |
| 03 | 17 | 2.5 | 15 | 331 | 258.6 |
| 04 | 51 | 12.4 | 16 | 7,261 | 694.2 |
| 05 | 21 | 35.8 | 17 | 130 | 96.3 |
| 06 | 807 | 142.7 | 18 | 379 | 415.1 |
| 07 | 94 | 48.8 | 19 | 727 | 111.5 |
| 08 | 238 | 71.7 | 20 | 990 | 95.7 |
| 09 | 60 | 18.0 | 21 | 174 | 70.0 |
| 10 | 108 | 42.6 | 22 | 33 | 7.3 |
| 11 | 1,184 | 1,042.1 | 23 | 411 | 750.9 |

Work lands in every hour of the day; the European working day (07-20 UTC) carries most of
it and the 16 UTC spike is one ConTeXt upload of 4,928 files on 2026-08-23. There is no
quiet hour to skip. Weekday, UTC, last 365 days: Sun 21,445 files / 29.0 GB (TeX Live
release day 2026-03-01 was a Sunday), Tue 10,724 / 16.5 GB (MacTeX 2026-03-24), the rest
6,400-11,900 files and 2.9-4.2 GB each. Minute of the hour is not fixed: the top minutes
in 30 days are `:10` (4,991 files, the ConTeXt burst), `:59` (2,271) and `:03` (1,379).

Hour-slots with at least one change: 2,236 of 8,760 in the year (25.5%), 276 of 727 in
the last 30 days (38%).

### How updates land

A package upload is written as a burst of files seconds apart, then the directory is
touched later by CTAN's post-processing. `pgfplots` on 2026-08-26: 120 files at
13:12:27 UTC, 2 at :28, 5 at :29; `install/graphics/pgf/contrib/pgfplots.tds.zip` at
:28; the package directory's own mtime 14:37:10, 85 minutes later.

```
awk '$3>="2026/07/27" {split($3,d,"/"); split($4,t,":"); s=(((d[1]*12+d[2])*31+d[3])*24+t[1])*3600+t[2]*60+t[3];
     p=$5; sub(/\/[^\/]*$/,"",p); if(!(p in mn)||s<mn[p])mn[p]=s; if(!(p in mx)||s>mx[p])mx[p]=s; n[p]++}
     END{for(p in n) if(n[p]>1) print mx[p]-mn[p], n[p], p}' SCRATCH/stored.norm | sort -n
```

Of 626 directories with two or more files changed in the last 30 days, the spread between
first and last mtime was: within 1 s for 441, 2-10 s for 9, 11-60 s for 26, 1-10 min for
42, 10-60 min for 30, over an hour for 78 (those are container directories such as
`macros/latex/contrib` and `systems/texlive/tlnet/archive`, which collect many uploads).
So a package is on disk within seconds; whether the daemon can serve a half-written
package to a listing that lands inside those seconds is **unverified** (it depends on
whether CTAN's installer writes into place or renames a staged directory). The design
does not need to know: a listing taken mid-burst sees some of the package's files; the
the next run's listing sees the rest as additions and any file rewritten since as a change.
Nothing in CTAN outside tlnet requires a package's files to arrive together (section 5).

`FILES.last07days` is not an hourly feed. Verified on today's copy: dates only (no time),
sizes and real paths only (0 lines under `documentation/`, `bibliography/`, `languages/`,
`digests/` or `systems/win32/`... the alias directories are absent), no deletions, and it
is written once a day. It lists 128 `pgfplots` lines for the 123 real files plus the
`install/` zip and others; the hour of the upload is not recoverable from it.

### What CTAN and TUG say

`https://ctan.org/mirrors/register`, fetched 2026-08-26:

> You must mirror from the primary CTAN node. ... `rsync://rsync.dante.ctan.org/CTAN`
> (Germany)

> `rsync -av --delete rsync://rsync.dante.ctan.org/CTAN /var/www/tex-archive`

> An official CTAN mirror must synchronize with the primary CTAN node at least once per
> hour. ... First choose a random minute between 0 and 59. ... `17 * * * * rsync -a
> --delete rsync://rsync.dante.ctan.org/CTAN /var/www/tex-archive` ... Please do not copy
> the minute used in the example unchanged. Choose a random minute when setting up your
> mirror and retain that minute permanently. Distributing mirrors across different minutes
> prevents them from contacting the primary CTAN node simultaneously and reduces avoidable
> load peaks.

The same page also says mirrors "copy our holdings every night" and "Update those files
every day"; the hourly sentence is the operative one (it is the requirement, the others are
older prose). It says nothing about symlinks beyond the Apache `+FollowSymLinks` example,
nothing about `-L`, and nothing about tlnet. `https://ctan.org/mirrors` adds only: "We
monitor mirrors to check that they are up to date. If a mirror falls behind then
`mirrors.ctan.org` will not redirect to it."

`https://www.tug.org/texlive/acquire-mirror.html` (tlnet only; fetched with curl, the page
refuses non-browser agents):

> `rsync -a --delete rsync://somectan/somepath/systems/texlive/tlnet/ /your/local/dir/`
> ... Add `-L` if your system does not support symbolic links.

TUG names any nearby rsync-capable mirror as the source for tlnet; CTAN requires the
primary for an official mirror. The primary is used for everything.

## 2. Symlinks

### Count and classes

```
awk -f SCRATCH/norm.awk SCRATCH/ctan-list-nolink.txt > SCRATCH/nolink.norm
awk -f SCRATCH/norm.awk SCRATCH/ctan-list-deref.txt  > SCRATCH/deref.norm
awk '$1=="l"{print $5}' SCRATCH/nolink.norm | LC_ALL=C sort > links.txt          # 24,788
awk '$1=="-"{print $5}' SCRATCH/deref.norm  | LC_ALL=C sort > deref-files.txt
awk '$1=="d"{print $5}' SCRATCH/deref.norm  | LC_ALL=C sort > deref-dirs.txt
LC_ALL=C comm -12 links.txt deref-files.txt | wc -l                              # 24,611 file targets
LC_ALL=C comm -12 links.txt deref-dirs.txt  | wc -l                              #    177 directory targets
LC_ALL=C comm -23 links.txt <(LC_ALL=C sort -u deref-files.txt deref-dirs.txt) | wc -l   # 0 dangling
```

| Class | Count | Notes |
|---|---:|---|
| Symlinks, total | 24,788 | in a tree of 352,357 files and 18,417 directories |
| To a regular file | 24,611 | |
| of which tlnet versioned containers (`archive/foo.tar.xz -> foo.r123.tar.xz`) | 14,872 | excluded as duplicates; the unversioned name is stored (today's rule) |
| of which under `systems/win32/` | 8,944 | aliases inside the MiKTeX/w32tex trees |
| of which elsewhere | 793 | `systems/texlive` 263, `macros/latex` 113, `info/translations` 43, ... |
| To a directory | 177 | root: `bibliography` -> `biblio`, `documentation` -> `info`, `languages` -> `language`, `digests` -> `info/digests`, `tds` -> `info/tds` (and the file `tds.zip` -> `info/tds.zip`); `systems/windows` -> `win32`; `macros/latex2e` -> `latex`; and 170 small ones. Targets inferred from target length against real directory names; real rsync prints them outright |
| Dangling or unreadable by the daemon | 0 | every symlink resolved under `-L` |
| Target string longer than 60 | 4 | longest 71 (`biblio/bibtex/contrib/economic/econometrica-fr.bst`) |

Objects the directory symlinks materialize (from `deref.norm`, files under each):
`macros/latex2e` 43,711 files / 5.74 GB, `documentation` 28,898 / 2.21 GB,
`systems/windows` 19,174 / 23.64 GB, `languages` 9,722 / 0.45 GB, `fonts/metrics` 9,672,
`fonts/greek/cbfonts-all` 2,926, `bibliography` 2,593 / 0.78 GB,
`obsolete/systems/win32/fptex/current` 2,212 / 0.48 GB, `digests` 1,776. `systems/win32` is
the real directory; `systems/windows` (target length 5) is the symlink.

Relative versus absolute, chains, and targets outside the module cannot be read from
today's listing: `openrsync` prints no `-> target`. Real rsync does (`-r --list-only`
prints `lrwxrwxrwx  6 2007/03/02 10:25:24 bibliography -> biblio`; verified with rsync
3.5.0 built locally, section 2.3). Two things can be said without the targets:

- Every one of the 9,737 non-tlnet file symlinks has a real file somewhere in the module
  with the same size and mtime (521 in the same directory, 9,216 elsewhere, 0 without one):
  ```
  awk '$1=="-"{k=$2" "$3" "$4; d=$5; sub(/\/[^\/]*$/,"",d); real[k]=real[k]" "d"|"} END{for(k in real) print k"\t"real[k]}' SCRATCH/nolink.norm > realkeys.txt
  awk 'NR==FNR{l[$0]=1;next} $1=="-" && ($5 in l) && $5 !~ /^systems\/texlive\/tlnet\// {print $2" "$3" "$4"\t"$5}' links-file.txt SCRATCH/deref.norm > linkderef.txt
  awk -F'\t' 'NR==FNR{m[$1]=$2;next} {k=$1;p=$2;d=p;sub(/\/[^\/]*$/,"",d); if(!(k in m))none++; else if(index(m[k]," "d"|"))same++; else other++} END{print same,other,none}' realkeys.txt linkderef.txt
  ```
  The inference is that targets stay inside the module. It is an inference, not a proof.
- The daemon resolved all 24,788 with `-L`, so whatever `use chroot` and `munge symlinks`
  dante has set, the referents are reachable to it (rsyncd.conf(5): a chrooted daemon
  cannot follow "symbolic links that are either absolute or outside of the new root path").

To get the targets: `TZ=UTC rsync -r --list-only --no-h rsync://rsync.dante.ctan.org/CTAN/`
from the runner (real rsync) once, and grep `' -> '`. Chains and absolute targets fall out
of that file. Nothing in the design needs them; `-L` hides them.

### What `rsync -L --files-from` does with each class

From rsync(1) (`https://download.samba.org/pub/rsync/rsync.1`, the 3.5.0 manual, fetched
2026-08-26):

> `--copy-links, -L`: The sender transforms each symlink encountered in the transfer into
> the referent item, following the symlink chain to the file or directory that it
> references. If a symlink chain is broken, an error is output and the file is dropped
> from the transfer. This option supersedes any other options that affect symlinks in the
> transfer, since there are no symlinks left in the transfer.

> `--files-from=FILE`: ... The `--relative` (`-R`) option is implied ... The `--dirs`
> (`-d`) option is implied ... The `--archive` (`-a`) option's behavior does not imply
> `--recursive` (`-r`), so specify it explicitly, if you want it. ... Blank entries are
> ignored, as are whole-entry comments that start with ';' or '#'. ... NOTE: sorting the
> list of files in the `--files-from` input helps rsync to be more efficient.

> `--relative, -R`: ... Rsync always sends these implied directories as real directories
> in the file list, even if a path element is really a symlink on the sending side.

> `--no-implied-dirs`: ... the attributes of the implied directories from the source names
> are not included in the transfer ... This even allows these implied path elements to
> have big differences, such as being a symlink to a directory on the receiving side.

> `--from0, -0`: This tells rsync that the rules or filenames it reads from a file are
> terminated by a null ('\0') character, not a NL, CR, or CR+LF.

> `--secluded-args, -s`: This option sends all filenames and most options to the remote
> rsync via the protocol (rather than via the remote shell command line) ... used to be
> called `--protect-args` (before 3.2.6).

> `--ignore-missing-args`: When rsync is first processing the explicitly requested source
> files (e.g. command-line arguments or `--files-from` entries), it is normally an error if
> the file cannot be found. This option suppresses that error, and does not try to transfer
> the file. This does not affect subsequent vanished-file errors.

> `--timeout=SECONDS`: ... maximum I/O timeout in seconds. If no data is transferred for
> the specified time then rsync will exit. `--contimeout=SECONDS`: ... the amount of time
> that rsync will wait for its connection to an rsync daemon to succeed.

Checked against rsync 3.5.0 (built from
`https://download.samba.org/pub/rsync/src/rsync-3.5.0.tar.gz`, `./configure
--disable-openssl --disable-xxhash --disable-zstd --disable-lz4 --disable-md2man`), local
source to local destination, on a fixture with every class. A daemon test was attempted
(`rsync --daemon --port=8876`) but the sandbox drops inbound connections, so daemon-side
behaviour rests on the manual. Fixture: `real/a.txt`, `real/sub/b.txt`, `real/zero.txt`
(0 bytes), `filelink.txt -> real/a.txt`, `dirlink -> real`, `abslink.txt -> /abs/.../a.txt`,
`outlink.txt -> ../outside/o.txt`, `dangling -> nowhere`, `chain.txt -> filelink.txt`,
`viadirlink.txt -> dirlink/sub/b.txt`, `sp ace/na me.txt`, `#hash.txt`, `;semi.txt`,
` lead.txt`, `d/#inner.txt`; the list names all of them plus `dirlink/a.txt`,
`dirlink/sub/b.txt` and a `missing.txt`.

| Case | `rsync -Lt --files-from=list src/ dst/` (3.5.0) |
|---|---|
| file symlink, chain, absolute, outside-tree | copied as regular files with the referent's mtime (`-t` follows through `-L`: `filelink.txt` and `chain.txt` got 2001-01-01, the referent's time) |
| path through a directory symlink (`dirlink/a.txt`) | `dirlink` created as a real directory, file inside it; identical with and without `--no-implied-dirs` on an empty destination |
| directory symlink named itself (`dirlink`) without `-r` | an empty real directory; nothing under it (`-d` implied, no recursion) |
| directory named itself with `-r` | the whole subtree |
| dangling symlink | `link_stat ... No such file or directory`, exit 23; everything else still copied |
| missing path | same error, exit 23; with `--ignore-missing-args` exit 0 |
| entry starting with `#` or `;` | silently dropped, **also with `--from0`** |
| entry with a leading space, a space inside, `/#` inside | copied intact |
| destination has a file where the source has a directory (`real` file, `real/a.txt` listed) | file replaced by the directory, exit 0 |
| destination has a directory where the source has a file | directory replaced by the file, exit 0 |
| `-rL --list-only` with a dangling symlink | prints `symlink has no referent`, lists everything else, **exit 23** |

CTAN today has 0 entries starting with `#` or `;`, 4 with a `/#` component
(`language/cyrtug/#disk.00` and its alias), 25 names containing spaces, 0 tabs, 0
non-ASCII bytes, 0 backslashes or quotes, longest path 151 characters (R2 allows 1,024
bytes per key, `https://developers.cloudflare.com/r2/platform/limits/`).

```
cut -f1 SCRATCH/state.tsv | grep -c '^[#;]'; cut -f1 SCRATCH/state.tsv | grep -c '/[#;]'
LC_ALL=C grep -c '[^ -~]' SCRATCH/ctan-list-deref.txt; grep -c $'\t' SCRATCH/ctan-list-deref.txt
```

### The flags, and why

| Flag | On the listing | On a batch fetch | Why |
|---|---|---|---|
| `-r` | yes | **no** | the listing must recurse; a fetch names files only, and without `-r` a path that has become a directory yields an empty directory instead of an unplanned subtree |
| `-L` | yes | yes | the stored set is the dereferenced tree; `-L` also makes `-t` carry the referent's mtime, which is what the listing shows |
| `-t` | n/a | yes | so `aws s3 cp` sees upstream mtimes in staging (not needed for the diff, useful for a local eye) |
| `--no-h` | yes | yes | real rsync defaults to digit separators and a 14-wide column (`--list-only` text in the manual); `--no-h` gives plain digits in 11 columns so the parser is the same for every rsync |
| `TZ=UTC` (env) | yes | yes | mtimes print in the client's zone; the state file must not depend on the runner's zone or a local run on a laptop would re-upload everything |
| `--files-from=RUN/batch.i` | no | yes | the batch; sorted, which the manual asks for |
| `--from0` | no | no | no CTAN name contains a newline; `--from0` does not rescue `#`/`;` entries anyway |
| `--no-implied-dirs` | no | no | no effect on an empty destination (verified); left out |
| `-s` | no | yes | harmless with a daemon; keeps any odd name off the command line if the design ever names paths as arguments |
| `--ignore-missing-args` | no | yes | a file deleted upstream between the listing and the fetch must not fail the batch; the state is built from what landed (section 5), so a skipped file is simply not recorded |
| `--timeout=300 --contimeout=60` | yes | yes | today's values; a stalled sender fails in 5 minutes |
| `--partial`, `--inplace` | no | no | staging is emptied per batch; there is nothing to resume into, and a partial file must never be uploaded |
| `--exclude` | no | no | the versioned-container exclusion is done on the listing text (one `grep -v`), which also keeps the state file honest; `--files-from` never names them |
| `--stats` | no | yes | bytes and rates for `report` |

Accepted exit codes: listing 0 or 23 (a dangling symlink upstream during a package
install would otherwise fail every mirror run for that hour; the listing is complete apart
from that link), followed by a sanity check that `RUN/U` has at least 90% as many lines as
`RUN/S` and contains `timestamp`; batch fetch 0, 23 or 24 (23 with `--ignore-missing-args`
can only be a permission or I/O error on a file that was found; 24 is a file that vanished
mid-transfer; either way the file is absent from staging and not recorded). Transport
codes 5, 10, 12, 30, 35 are the retry loop's business (`errors-and-issues.md`).

## 3. The listing as the source of truth

### Line format

`rsync --list-only` prints one line per entry: permissions, size, mtime, name. From the
manual:

> `drwxrwxr-x          4,096 2022/09/30 12:53:11 support`
> `-rw-rw-r--             80 2005/01/11 10:37:37 support/Makefile`
> The only option that affects this output style is the `--human-readable` (`-h`) option.
> The default is to output sizes as byte counts with digit separators (in a
> 14-character-width column). ... If you want old-style bytecount sizes without digit
> separators (and an 11-character-width column) use `--no-h`.

The time is `%4d/%02d/%02d %02d:%02d:%02d` from `localtime_r` (rsync `util1.c`,
`timestring()`): second resolution, local zone, no sub-second part. Names print
as-is; control characters are escaped as `\#ooo` (manual, `--8-bit-output`), and CTAN has
none. `openrsync` prints a blank size for zero-byte files (525 of them today); real rsync
prints `0`. Symlinks without `-L` print `name -> target` in real rsync and no target in
`openrsync`.

The parser anchors on the date-time columns rather than on field positions, so it reads
both widths, both size styles, and names with spaces:

```
# SCRATCH/norm.awk: --list-only line -> "type size date time path"
{ m = match($0, /[0-9][0-9][0-9][0-9]\/[0-9][0-9]\/[0-9][0-9] [0-9][0-9]:[0-9][0-9]:[0-9][0-9] /)
  if (!m) { print "UNPARSED: " $0 > "/dev/stderr"; next }
  size = substr($0, 11, m-11); gsub(/[ ,]/, "", size); if (size == "") size = 0
  print substr($0,1,1), size, substr($0,m,10), substr($0,m+11,8), substr($0,m+20) }
```

Anchoring matters because field counting does not work: 525 zero-byte files print a
blank size in `openrsync` output, 25 names contain spaces, and two lines have both
(`macros/latex/contrib/akktex/documentation/still to do` and its `macros/latex2e/` alias),
so "four fields means blank size" and any `NF`-based split mis-parse them. The anchored
parser reads them as size 0 with the spaces kept, and the full pipeline still yields
496,149 lines:

```
grep 'still to do' SCRATCH/ctan-list-deref.txt | awk -f norm.awk
# - 0 2006/10/23 07:50:00 macros/latex/contrib/akktex/documentation/still to do
# - 0 2006/10/23 07:50:00 macros/latex2e/contrib/akktex/documentation/still to do
awk -f norm.awk SCRATCH/ctan-list-deref.txt 2>unparsed.txt | awk '$1=="-" && $2==0 && NF>5'   # exactly those two
wc -l < unparsed.txt                                                                        # 0
```

A path cannot contain a date-time-space sequence before the real one because the columns
precede it. The one thing the anchor cannot survive is a name containing the pattern
`dddd/dd/dd dd:dd:dd ` after a space in a *directory* component; CTAN has no such name,
and `--no-h` plus fixed columns (`substr($0, 24, 10)` for the date) is the fallback.

The state line the design keeps is `path<TAB>size<TAB>yyyy/mm/dd hh:mm:ss`, path first:

```
TZ=UTC rsync -rL --list-only --no-h --timeout=300 --contimeout=60 rsync://rsync.dante.ctan.org/CTAN/ > RUN/upstream.raw
awk -f norm.awk RUN/upstream.raw \
| awk '$1=="-"' \
| grep -v ' systems/texlive/tlnet/archive/.*\.r[0-9][0-9]*\.tar\.xz$' \
| grep -v ' systems/texlive/tlnet/update-tlmgr-r' \
| awk '{p=$5; for(i=6;i<=NF;i++) p=p" "$i; printf "%s\t%s\t%s %s\n", p, $2, $3, $4}' \
| LC_ALL=C sort > RUN/U
```

Today that is 496,149 lines, 132.99 GB, 37.7 MB as text, **3.10 MB as `.xz`** (`xz -T0`,
3.5 s to compress, 0.1 s to decompress; `comm` of two such files 0.1 s). Without the
`update-tlmgr-r*` exclude (6 files) it is 496,155 / 133.01 GB.

Path first, tab-separated, `LC_ALL=C sort`: because the tab (0x09) sorts below every
printable byte, sorting whole lines is the same order as sorting paths alone, so one sort
serves both the line-level `comm` (changes) and the path-level `comm` (deletions), and
`cut -f1` of a sorted state is itself sorted:

```
cut -f1 SCRATCH/state.tsv | LC_ALL=C sort -c && echo sorted     # holds today
```

### Why size+mtime, and where it fails

The key is the pair (size, mtime to the second). It is exactly rsync's own "quick check"
(manual, `--size-only`: "the default of transferring files with either a changed size or
a changed last-modified time"), so every `rsync -a` CTAN mirror has the same blind spots.

| Event upstream | Listing shows | Diff does | Outcome |
|---|---|---|---|
| content rewritten, any size change | different size | fetch | correct |
| content rewritten, same size, different second | different mtime | fetch | correct |
| content rewritten, same size, same second | identical line | nothing | **missed** until the file changes again. Needs two writes of equal length inside one second with the listing between them, or an editor that preserves mtime. Same blind spot as every rsync mirror; accepted |
| `touch` without change | different mtime | fetch, re-upload identical bytes | one wasted PutObject |
| mtime set backwards (restore from backup, `cp -p` of an old copy) | different mtime | fetch | correct; the key is inequality, not order |
| restore from backup with mtimes preserved and content unchanged | identical line | nothing | correct |
| dante's daemon offline | no listing | run fails, next run | mirror stale until it |
| file deleted and re-added within the hour, same size and mtime | identical line | nothing | content is the same in every real case (the file came back from the same source); the only theoretical miss is a same-size same-second different-content re-add |
| file replaced by a directory of the same name | `foo` gone, `foo/...` new | delete `foo`, add `foo/...` | correct; R2 keys `foo` and `foo/bar` can coexist, so order is irrelevant |
| directory replaced by a file | reverse | reverse | correct |
| case-only rename (`Makefile` -> `makefile`) | old path gone, new path new | delete one key, add the other | correct on R2 (keys are byte strings) |

Compare `aws s3 sync`'s rule
(`https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html`): "A local file will
require uploading if the size of the local file is different than the size of the S3
object, the last modified time of the local file is newer than the last modified time of
the S3 object, or the local file does not exist under the specified bucket and prefix."
That rule needs the local tree and the bucket listing and compares upstream mtime with R2
upload time, which is why a backwards mtime is invisible to it and why it costs 497
`ListObjects` per run. `--exact-timestamps` does not help: "When syncing from S3 to local,
same-sized items will be ignored only when the timestamps match exactly" (that direction
only). The list-diff never calls `sync`: it compares two upstream listings with each
other, so the change key is purely upstream and the bucket is never listed in the hourly
path.

Case: R2 keys are case-sensitive; so is dante's filesystem. CTAN holds 28 lowercased
paths that occur twice (56 files, 14 distinct pairs before aliasing), all old
(`dviware/ivd2dvi/Makefile` and `makefile`, `info/epslatex/french/Danger.eps` and
`danger.eps`, six `bakoma` pairs under `systems/win32` mirrored under `systems/windows`,
`obsolete/support/pdftexenc/Symbol.enc` and `symbol.enc`, ...):

```
awk -F'\t' '{print tolower($1) "\t" $0}' SCRATCH/state.tsv | LC_ALL=C sort \
| awk -F'\t' '{if ($1==prev) {print prevline; print $0} prev=$1; prevline=$0}' | cut -f2- | uniq
```

On the Linux runner these are 56 distinct keys and nothing happens. On a macOS run with
`staging/` on a case-insensitive APFS volume, a batch that fetches both halves of a pair
writes one over the other and uploads wrong bytes under one key, and the daily reconcile
would not notice the three pairs whose sizes are equal (`Symbol.enc`, the `bakoma`
`PNG`/`GIF` pairs). A local run must therefore refuse a case-insensitive `staging/`:

```
- touch {{.STAGING}}/.Case {{.STAGING}}/.case && test "$(ls -a {{.STAGING}} | grep -ci '^\.case$')" -eq 2   # refuse a case-folding staging/
```

## 4. The diff

```
# S: the state the last successful run left (RUN/S), U: this run's listing (RUN/U); both LC_ALL=C sorted.
LC_ALL=C comm -13 RUN/S RUN/U                      > RUN/changed    # lines in U not in S: added or changed
cut -f1 RUN/S > RUN/S.paths; cut -f1 RUN/U > RUN/U.paths            # both still sorted (tab < any byte)
LC_ALL=C comm -23 RUN/S.paths RUN/U.paths          > RUN/deleted    # paths in S not in U
cut -f1 RUN/changed                                > RUN/fetch      # the fetch list, sorted
```

Three lists fall out: **added** (`RUN/changed` lines whose path is not in `RUN/S.paths`),
**changed** (the rest of `RUN/changed`), **deleted** (`RUN/deleted`). `report` counts
added and changed with one `join -v`:

```
LC_ALL=C join -t "$(printf '\t')" -v2 RUN/S.paths RUN/changed | wc -l   # added
```

State transition. Let S and U be sets of lines `(p, s, m)` with unique paths. The run
uploads every line in `U \ S` and deletes every path in `paths(S) \ paths(U)`. Afterwards
the bucket holds, for each path p:

- p in `paths(U)`, line in `S ∩ U`: untouched; it held that version before (induction) and
  the line says upstream still has it.
- p in `paths(U)`, line in `U \ S`: uploaded this run at U's version.
- p not in `paths(U)`: either it was in `paths(S)` and was deleted, or it was never there.

So the bucket's key set equals `paths(U)` and each key's version is U's line, which is
what "the bucket equals the listing" means. The new state is U itself, restricted to what
actually landed (section 5); a line the fetch could not deliver is left out and the next
run's `comm` produces it again. The same argument covers an empty S (the seed) and a
truncated S (a run cut off mid-way): both are just larger `U \ S`.

A path that is deleted and re-added within the hour appears as a change if its line
differs and as nothing if not (section 3). A path that changes type is a deletion plus an
addition, and both R2 and an empty `staging/` accept either order.

The merge that produces the new state, batch by batch, keeps U's line for every path the
batch delivered and S's line otherwise (`sort -s -u` keeps the first line of each equal
key, so the batch's lines come first):

```
cd {{.STAGING}} && find . -type f | sed 's|^\./||' | LC_ALL=C sort > {{.RUN}}/landed
LC_ALL=C join -t "$(printf '\t')" {{.RUN}}/landed {{.RUN}}/U > {{.RUN}}/applied.i        # U's lines for what landed
cat {{.RUN}}/applied.i {{.RUN}}/S | LC_ALL=C sort -t "$(printf '\t')" -k1,1 -s -u > {{.RUN}}/S.new
```

Verified on this machine's `sort` and `join` (BSD); GNU coreutils document the same `-s`
and `-u` semantics.

## 5. The consistency model

**At-least-once, checkpoint per batch, state last.** Every write is idempotent: a
`PutObject` of the same bytes, a `DeleteObjects` of an absent key, a purge of an
unchanged URL, a `PutObject` of the state file. The state is written only after the
batch's upload (and the purge, when the edge cache is on) returned success, as one `PutObject` (3 MB, single-part, so it is
the old object or the new one, never half; R2's 1 write/s/key limit is far from a batch
every few minutes). A run that dies anywhere leaves the state as of the last completed
batch; the next run's `comm` reproduces the unfinished batch and repeats it in full. The
cost of a repeat is bounded by one batch (4 GB, ~16k `PutObject`, free).

**The listing-to-fetch race.** The fetch copies whatever upstream holds at fetch time, not
at listing time. The state records U's line (listing time). Three cases:

- Unchanged in between: line and bytes agree.
- Changed in between: the bytes are newer than the line. The next run's listing has a new
  line, `comm` re-fetches, and the second upload is identical bytes. One wasted PutObject.
- Gone in between: `--ignore-missing-args` skips it, it is absent from `landed`, the
  state does not gain it; if S had it, S keeps its old line until this run's deletion step
  or the next run's `comm` removes it. The bucket keeps the old version one hour longer
  than upstream, which is what an hourly `rsync --delete` mirror does too.

**What a client can see.** Between batches the bucket is a mixture: some keys at U's
version, the rest at S's version, and the keys `paths(S) \ paths(U)` still present until
the deletion step. Formally the bucket is always a superset of `paths(S) ∩ paths(U)` with
each key at one of its two most recent upstream versions, and never holds a key or a byte
that upstream did not publish. It is not always "a CTAN snapshot": no mirror is, because
dante has no snapshots either (packages land as file bursts, section 1, and an
`rsync -a --delete` mirror is in exactly this mixed state during its run). What matters is
ordering where a consumer reads one file to decide what to fetch next:

| Consumer | Decision file | Files it names | Ordering needed | How the design provides it |
|---|---|---|---|---|
| `tlmgr`, `install-tl` | `systems/texlive/tlnet/tlpkg/texlive.tlpdb(.xz, .sha512, .asc)` | `archive/*.tar.xz` | containers before the tlpdb that names them (today's rule) | `tlpkg/` is in the decision batch, uploaded after every other batch; containers a tlpdb names but which are not yet in the bucket hold the tlpdb back (below) |
| `tlmgr update --self`, installers | root `install-tl*`, `update-tlmgr-latest.*` with their `.sha512(.asc)` | nothing else | none across files (each carries its own signature) | decision batch |
| mirmon (`https://ctan.org/mirrors/mirmon`) | `/timestamp` | the whole tree, implicitly | the stamp must not claim an hour whose files are not there | `timestamp` is the last line of the decision batch; the state's `timestamp` line is the run's "done" mark |
| ctan.org "download" links, browsers | none (direct URLs), `install/.../x.tds.zip` | none | none | none needed; a `.zip` and its unpacked directory land within the same second upstream and in the same batch here (both are in `RUN/changed`, sorted, so they fall in adjacent batches at worst) |
| `FILES.byname` readers (scripts) | `FILES.byname` | every real path | nothing atomic exists upstream either (generated once a day from a moving tree) | decision batch, after the files it lists that this run carried |

The decision batch is the set of lines in `RUN/changed` whose path matches
`^[^/]+$` (root files) or `^systems/texlive/tlnet/(tlpkg/|[^/]+$)`, in that order with
`timestamp` last. It is always the final batch and always its own batch, so no ordering
inside `aws s3 cp --recursive` (which the design does not rely on; its walk order is not
documented) can put a tlpdb ahead of a container. The mid-update guard is the delta form
of today's fifth `verify` check: before uploading the decision batch, every container the
fetched tlpdb names must be in `RUN/S.new` (already uploaded, this run or before). If any
is missing, **the run fails** before the decision batch is uploaded, exactly as today: the
state stands as of the last committed batch, the live tlpdb is the old one, and the next
hour recomputes the delta and tries again with whatever containers have arrived
(`verification-and-security.md` and `taskfile-architecture.md` adopt this over dropping
the tlnet decision files and continuing). Which checks run, and what happens to a
container that fails its checksum, is `verification-and-security.md`'s; what this file
fixes is that the tlpdb is fetched (into `RUN/`, not `staging/`) and verified at the
start of a run that touches tlnet, used as the reference for every batch, and uploaded
last.

**Deletions after uploads.** A rename upstream is an addition plus a deletion; uploading
first means the new key exists before the old one goes, so a client following an old link
gets the old bytes for an hour rather than a 404 for a minute. Deletions and the
reconcile's unknown-key deletion never touch the two reserved root prefixes `.state/`
(this file's state) and `.site/` (the landing page, `official-mirror-and-url.md`); the
root `index.html` is CTAN's own file and is mirrored like any other.

**Where the model is weaker than a snapshot, and why that is acceptable.** A package's
files can be split across two hourly runs when the listing lands inside the seconds of an
upload; a user who fetches that package in that hour from this mirror sees what a user of
dante saw in those seconds, which CTAN already tolerates. tlnet is the one place where
half a package is an error a user notices (`tlmgr` reports a checksum mismatch), and the
decision batch plus the tlpdb guard closes it exactly as today's ordering rule does. The
edge cache is off by default; if it is turned on it can widen the window, which is
`caching.md`'s problem.

## 6. The daily reconcile

The state says what the bucket should hold. Once a day the run also asks the bucket:

```
aws s3api list-objects-v2 --bucket tlnet --query 'Contents[].[Key,Size]' --output text \
| LC_ALL=C sort > RUN/bucket           # 497 ListObjectsV2 calls (Class A), auto-paginated; key<TAB>size
grep -v -e '^\.state/' -e '^\.site/' RUN/bucket > RUN/bucket.mirror    # the two reserved root prefixes; everything else is CTAN's
cut -f1,2 RUN/S > RUN/S.keysize
LC_ALL=C comm -23 RUN/bucket.mirror RUN/S.keysize | cut -f1 | LC_ALL=C comm -23 - RUN/S.paths > RUN/reconcile.delete   # keys the state does not know
LC_ALL=C comm -13 RUN/bucket.mirror RUN/S.keysize | cut -f1 > RUN/reconcile.refetch                                  # missing or wrong size
```

`s3api list-objects-v2` rather than `aws s3 ls --recursive`: the latter prints
`2013-09-02 21:37:53         10 a.txt` (date, time, size, key; verified on the `ls`
reference page) and today's Taskfile strips three fields with `sed`, which works but
leaves the key unquoted; `--output text` gives a tab between key and size and nothing
else. R2 returns keys in a different order from byte order (the CLAUDE.md `sync --delete`
bug), so both sides are re-sorted with `LC_ALL=C` before `comm`.

What the listing offers per object: `Key`, `Size`, `ETag`, `LastModified`, `StorageClass`.
`LastModified` is the upload time, useless against upstream mtimes. `ETag` is the MD5 of
the bytes for a single-part upload on S3; for R2 the multipart page
(`https://developers.cloudflare.com/r2/objects/multipart-objects/`) states the multipart
formula and that "ETags for objects uploaded via multipart differ from those uploaded with
a single PUT", and the Workers API reference says "The MD5 checksum will be included by
default for non-multipart objects"; the single-part ETag = MD5 statement itself is
**unverified** on a first-party page (check: `aws s3api head-object --bucket tlnet --key
timestamp` against `md5` of the staged file after the first run). It does not matter: the
upstream listing carries no checksum, the tool list has no MD5 program (`shasum` does
SHA), so there is nothing to compare an ETag with. The reconcile joins on **key and size
only**.

What size-only misses: an object whose bytes are wrong but whose length is right. How that
could arise: (a) R2 corrupting data at rest, which is R2's durability promise to keep;
(b) a `PutObject` that stored wrong bytes with the right length: the CLI sends the object
with its length and R2 verifies `Content-MD5` when sent (PutObject row on the S3 API
compatibility page lists `Content-MD5` as implemented); whether the runner's CLI sends
`Content-MD5` or a CRC checksum by default, and whether R2 accepts the CRC variant, is a
question for `errors-and-issues.md`; (c) a case-folding local run (section 3), which is
refused; (d) the same-size same-second upstream edit, which no mirror sees. None of these
is a reconcile's job. The reconcile exists for what the hourly path cannot see at all:

- **State loss or corruption.** If `.state/applied.tsv.xz` is missing, the run bootstraps
  S from the bucket: `RUN/bucket.mirror` joined to `RUN/U` on key and size takes U's
  line for every key whose size matches, so a fresh runner or a deleted state costs one
  listing (497 Class A) and re-uploads only the mismatches, not 133 GB. This is also how a
  seed resumes from a bucket that already holds tlnet: the 17k tlnet keys join and are not
  re-put.
- **A manual edit** in the dashboard, a bucket-side lifecycle, or a bug in the diff.
- **Storage measurement** for the ceiling and `report` (sum of `Size`), the only place the
  mirror's size is measured.
- **Nothing to do with partial multipart uploads.** An incomplete multipart upload creates
  no key; R2 aborts it by default seven days after initiation
  (`https://developers.cloudflare.com/r2/buckets/object-lifecycles/`: "Buckets have a
  default lifecycle rule to expire multipart uploads seven days after initiation"). There
  is nothing for the reconcile to catch.

Storing upstream mtime as object metadata and comparing via `HEAD`: `aws s3 cp --metadata
mtime=...` is supported (`x-amz-meta-*` on PutObject), but `HEAD` is Class B and the
metadata is not in listings (rclone's S3 page documents the same limitation for its
`X-Amz-Meta-Mtime`: "reading this from the object takes an additional HEAD request as the
metadata isn't returned in object listings"). 496,149 `HEAD` a day is 14.9M a month, past
the 10M free Class B, about $1.76/month, and hours of sequential calls. Rejected; the
state file is that metadata, kept in one object.

Decision: size-only join, daily, in the existing 03:30 UTC slot; refetch list merged into
that run's `RUN/fetch`, delete list into that run's deletions; bootstrap when the state
is absent. Which run reconciles is chosen by the workflow (`taskfile-architecture.md`).

## 7. Alternatives

**Full `rsync -a --delete` into a persistent tree.** The recipe CTAN publishes, and the
right one wherever a disk exists. It needs 133 GB (69 GB without `-L`, but then every
alias needs resolving before upload, and the 134k files under directory symlinks become
CopyObject work) that survives between hourly runs. GitHub runners have 14 GB of
scratch that is destroyed at job end. The Actions cache is "10 GB per repository" and
"GitHub will remove any cache entries that have not been accessed in over 7 days"
(`docs.github.com`, caching guide): too small by an order of magnitude even before the
tlnet share. An R2 FUSE mount (rclone mount, s3fs; neither in the tool list) turns
`rsync -a`'s walk into bucket listings: 27,262 directories is ~27k `ListObjects` an hour,
19.6M a month, Class A, about $84/month, and every file compare a `HEAD`. A free-tier VM
elsewhere is a second system with its own disk quota (Oracle's always-free 200 GB block
storage, unverified), its own monitoring, and a shell script; it breaks "no shell scripts",
"free GitHub Actions" and "one alert". All three break the constraints in CLAUDE.md; the
list-diff exists because they do.

**`FILES.last07days` as the change feed.** Verified today: dates without times, no
deletions, only real paths (no alias directories), regenerated once a day at ~23:20 UTC.
It cannot drive an hourly mirror and cannot delete. It is a good cross-check for `report`
("today's `FILES.last07days` lines dated today are all in S"). Rejected as source.

**`rsync --itemize-changes --dry-run` against a local stub tree.** rsync's quick check
reads only size and mtime, so a tree of 496k sparse files (`truncate -s SIZE`,
`touch -d MTIME`) rebuilt from the state each run makes `rsync -rLtin --delete dante
stubs/` print exactly the diff (`>f+++++++++` new, `>f.st......` changed, `*deleting`).
It is the same computation as `comm`, done by creating half a million inodes an hour
(minutes of runner time, 1.9 GB of directory metadata is fine, disk blocks are zero for
sparse files) and parsing an 11-letter code. It has one real use: a once-off
cross-check that `comm` and rsync agree on a given hour. Rejected for the hourly path.

**`rsync --write-batch`.** Records a transfer against an existing destination and needs
that destination; the batch file contains the file data, so it is a fetch by another name.
Rejected.

**rclone.** Current release v1.75.0 (2026-07-31, `rclone.org/changelog`); the S3 backend
lists Cloudflare R2 as a provider (`rclone.org/s3`, "Cloudflare R2" section), so the
memory note about 1.60.1 and HTTP 501 is history. rclone cannot read an rsync daemon, so
it could only replace the AWS CLI on the upload side. Its change detection against R2 is
size plus `X-Amz-Meta-Mtime` via `HEAD` (Class B per object), or `--size-only`, or
`--checksum` (MD5 from ETag, free with the listing but needing a local MD5); none of
those replaces the upstream-vs-upstream diff, and `rclone copy --files-from` is what
`aws s3 cp --recursive` already does. Adding it buys `--checksum` reconciles at the price
of one more binary outside the tool list. Rejected.

**`aws s3 sync --exact-timestamps`.** The flag applies only when downloading ("When
syncing from S3 to local"); uploads use size-or-newer against R2's upload time. `sync`
also needs the full local tree and lists the bucket (497 Class A) every run. Rejected.

**A secondary mirror as source.** "You must mirror from the primary CTAN node"
(register page). Secondaries lag by up to their own cron period plus dante's; the Taskfile
already notes "Secondary mirrors can lag it by a day; the master cannot". Rejected.

**Server-side copies for aliases.** Real rsync's `-r --list-only` shows symlink targets;
with that second listing, every file symlink whose target is a real file could become an
`aws s3 cp s3://tlnet/target s3://tlnet/alias` (CopyObject; `UploadPartCopy` with
`x-amz-copy-source-range` is implemented on R2 for the five objects over 5 GiB) instead
of a second fetch and upload. It would cut the release-day hour from three ISO transfers
(20.4 GB) to one plus two copies, and the seed from 133 GB fetched to 97 GB. It costs a
second listing (only on runs whose `RUN/changed` holds a file over, say, 1 GB), target
resolution including chains, and a `smoke` that proves a copy equals its source. Not in
the first version; the batch checkpoint makes the slow path correct (section 8), and two
days a year do not justify the logic. Open question.

**Skipping quiet hours.** `timestamp` changes every hour, so U never equals S. The run
is ~30 s when nothing else changed; nothing to skip.

## 8. Time budget of an hourly run

"Next run" below is not "next hour": a scheduled run starts 15 to 45 minutes after its
slot and a slot can be dropped (section 9), so the gap between two runs is anywhere from
~15 minutes to ~2 h 45 min. The budgets are per run and none of them needs the next run
to begin on time.

Slot statistics, UTC hour-slots over the 365 days to 2026-08-27, stored set:

```
awk -f SCRATCH/utc.awk -f SCRATCH/slot.awk SCRATCH/state.tsv | sort > SCRATCH/slots-utc.txt
awk '{print $4}' SCRATCH/slots-utc.txt | sort -n | awk '{a[NR]=$1} END{print a[int(NR*.5)], a[int(NR*.95)], a[int(NR*.99)], a[NR]}'
```

| Slot | Files | MB | Where |
|---|---:|---:|---|
| P50 (of slots with work) | 4 | 0.8 | |
| P95 | 91 | 50.0 | |
| P99 | 679 | 226.2 | |
| Most files | 5,950 | 36.7 | 2026-01-13 00 UTC |
| Most bytes, no installers | 918 | 1,064.7 | 2026-06-07 14 UTC |
| Most bytes | 9 | 20,354.4 | 2026-03-01 17 UTC, TeX Live release |
| Second | 12 | 13,973.1 | 2026-03-24 18 UTC, MacTeX |

Release days by UTC hour (`awk -v d=2026/03/01 -f utc.awk -f hday.awk state.tsv`):

| 2026-03-01 | files | MB | 2026-03-24 | files | MB |
|---|---:|---:|---|---:|---:|
| 02 | 8 | 21.3 | 00 | 11 | 1.5 |
| 13 | 15 | 10.3 | 14 | 245 | 26.1 |
| 14 | 3 | 105.0 (source tarball) | 17 | 2 | 5.6 |
| 15 | 1,517 | 31.1 (tlnet rollover) | **18** | **12** | **13,973.1** (`MacTeX.pkg`, `mactex-20260324.pkg`, 6,865 MB each; Ghostscript pkgs 47-74 MB, doubled by aliases) |
| 16 | 3 | 2.8 | 22 | 14 | 0.0 |
| **17** | **9** | **20,354.4** (`texlive.iso`, `texlive2026.iso`, `texlive2026-20260301.iso`, 6,785 MB each) | | | |
| 23 | 2 | 280.8 (`BasicTeX.pkg` twice) | | | |

Per-run fixed costs, measured or derived: listing 6.9 s wall (`SCRATCH/rsync-time.txt`,
one sample, `openrsync`; real rsync **unverified**, expected similar since the daemon does
the walk), normalise and sort 496k lines ~1 s, state `GetObject` and `xz -dc` ~1 s,
`comm` 0.1 s, `xz -T0` of the new state 3.5 s per batch on this laptop (a runner is
slower; call it 5 s), state `PutObject` ~1 s. Under 30 s before any file moves.

Transfer rates have one measurement. The tlnet run of 2026-08-26 05:15 UTC (job
32933376123) moved 6.79 GB from dante in 299 s, **22.7 MB/s**, and `aws s3 sync` put at
least 7,891 objects in 82 s, **~96 `PutObject`/s** at 32-way concurrency (the log drops
lines, so a floor). Those are the working figures. Every `sync` job's summary contains
rsync's `--stats` block with `bytes/sec`; the first hourly runs add to them.

| Slot | Fetch at 22.7 MB/s | Upload at 96 obj/s | Whole run |
|---|---|---|---|
| P50 | 0 s | 0 s | ~30 s |
| P95 | 2 s | 1 s | ~35 s |
| P99 | 10 s | 7 s | ~50 s |
| Most files (5,950; 36.7 MB) | 2 s, plus per-file overhead | 62 s | ~2 min |
| Worst without installers (1.06 GB, 918 files) | 47 s | 10 s | ~1.5 min |
| MacTeX hour (13.97 GB) | 615 s, but see below | multipart, ~615 s at the same rate | **two 6.9 GB batches of ~20 min each, plus a small batch** |
| Release hour (20.35 GB) | 897 s | ~897 s | **three 6.8 GB batches of ~20 min each** |

Does `timeout-minutes: 55` hold on a release day? Not as a single run, and it need not.
Each installer copy is its own batch (any file over 4 GB is alone) with its own checkpoint;
at 22.7 MB/s a batch is 5 min down and 5 min up, three of them 30 min, so the release
hour fits with margin, but if the rate that day is a third of that (dante is busiest on
release day) the run dies at 55 min with one or two batches committed and the next run
finishes the rest, whenever it starts. The condition that must hold is per batch, not per run: one 6.87 GB
file down and up within 55 minutes, i.e. a sustained **4.2 MB/s** combined. Anything
slower than that on a release day leaves the ISO copies for later runs and everything
else proceeds, because the small batches come first in the batch plan. At 22.7 MB/s the
whole release day fits one run; the checkpoint is what makes the rate unimportant.

Ordering of batches: small batches first, installer batches after, the decision batch
last, so a slow hour delays only installers. `MAX_BATCHES=4` by default on an hourly run
(`taskfile-architecture.md`): the run commits up to four batches and stops, leaving the
rest to the next run, so the release hour above (one small batch, three ISO batches, the
decision batch) takes two runs and `timestamp` lags by the gap between them, 15 minutes to
2 h 45 min, that day; the seed raises the cap. Disk: one batch (≤4 GB, or one file ≤6.87 GB)
plus ~100 MB of listings, under the 14 GB runner disk at all times.

## 9. Load on dante

What an `rsync -a --delete` mirror costs dante each hour: one connection; the daemon
walks 395,562 entries (no `-L`), builds and sends the file list, then, on the receiver's
request, reads and sends the changed files (P50 0.8 MB, P95 50 MB). The receiver does the
comparison; the daemon's CPU is the walk and the list.

What this design costs dante each hour: one connection for the listing (the daemon walks
538,289 entries because `-L` expands the 177 directory symlinks and stats 24,788
referents, ~36% more `stat` calls than the recipe's walk, and the file list is ~36%
longer), then one connection per batch (usually one, ~34 during the seed, never two at
once) in which the daemon stats only the named paths and their implied directories and
sends those files. The bytes sent are the same as the recipe's. The listing's wire size is
**unverified** (`rsync -rL --list-only --stats` from the runner prints `File list size`
and `Total bytes received`).

So per hour it is the recipe's list plus a third, and the recipe's transfer; per month 720
listings plus ~760 fetch connections. An `rsync -a` mirror that uses `-L` (TUG's advice for
symlink-less systems) costs exactly this listing. It is within what CTAN asks each mirror
to do. What CTAN asks in return is a fixed random minute, kept: the workflow cron is
`M * * * *` with M drawn once by `shuf -i 0-59 -n 1` and never changed, and not in 00-05
(the `:02` `timestamp` touch). That minute is nominal, which the next section is about.

### The fixed minute is nominal

Observed in this repository: the one scheduled `sync` run in the history so far (cron
`30 3 * * *`) was created at 04:09:16 UTC on 2026-08-26, 39 minutes after its slot; the
next day's was 18 minutes late and not yet started when this was written. GitHub's
schedule-event documentation says scheduled workflows "can be delayed during periods of
high loads of GitHub Actions workflow runs", that "High load times include the start of
every hour", and that a slot can be dropped. Treat 15 to 45 minutes late as normal
operation and a dropped slot as possible; never assume a run starts at its cron minute.
No external cron or paid runner is inside the constraints, so the design absorbs it:

- **Listing cadence.** Runs are hourly on average, not hourly on the minute. Consecutive
  listings can be ~15 minutes apart (a 45-minute-late run followed by an on-time one;
  `concurrency: sync` without `cancel-in-progress` queues the second and starts it when
  the first ends) and up to 2 h 45 min apart (a dropped slot, then a 45-minute-late run).
  The diff is indifferent: `U \ S` is the same set whether computed 15 or 165 minutes
  after the last run; only its size changes, and `MAX_BATCHES` bounds the run either way.
  Dante sees one listing per run at a drifting minute, still one connection an hour on
  average, and never synchronised with another mirror's cron.
- **The `timestamp` copy.** The mirror never writes a stamp of its own; it copies
  dante's, whose content is the last `:02` touch before the listing. The age mirmon
  computes (probe time minus the stamp's content time) is therefore honest: our gap since
  the last listing plus dante's own 0-59 minutes of stamp lag. Worst case: a run lists on
  time at H:M, the H+1 slot is dropped, the H+2 run starts 45 minutes late and lists at
  H+2:M+45. Until its decision batch lands the stamp still reads H:02 (H-1:02 if M < 2),
  so the age reaches 2 h 45 min + 59 min = **3 h 44 min**, plus that run's own duration
  before the decision batch (a minute normally, up to 55 on a release day) and mirmon's
  probe interval: call it 4 to 5 hours. mirmon's bands (`https://ctan.org/mirrors/mirmon`,
  2026-08-27): green up to 28 hours, then 28-52 hours, then "old", then "bad"; today's
  median mirror age is 3 hours and ctan.org's own is 2 hours. On a good run this mirror
  sits at the median; on the worst it is inside the green band by a factor of six.
- **The cron minute.** The register page wants a fixed random minute so mirrors do not
  hit dante together; GitHub's delay turns the real start into a spread of 15 to 45
  minutes, which serves that purpose on its own. Whether a minute outside 00-05 reduces
  the lateness is **unverified**: the documentation names the top of the hour as the busy
  time, but the one observed run was 39 minutes late from a `:30` slot, so M is chosen for
  CTAN's reason, not GitHub's. What the registration notes should say about the drift is
  `official-mirror-and-url.md`'s.
- The healthchecks period and grace that cover a late start, a long run and a dropped
  slot without paging are `monitoring.md`'s; `report` prints the lateness (slot minute
  from the workflow file against `date -u`), `taskfile-architecture.md`'s. The maintainers' list etiquette (announce a seed of 133 GB beforehand, ask
about `max connections` for the seed's 34 sequential connections) is advice,
**unverified** against the list's archive, which was not fetched.

## Open questions

- **Reserved prefixes.** CTAN publishes its own root `index.html` (10,366 bytes, 2020),
  so the landing page cannot live there; it lives under `.site/` and the state under
  `.state/`. CTAN has no root entry beginning with a dot (0 today), so neither can
  collide, but CLAUDE.md's "Objects stay under `systems/texlive/tlnet/`" must become
  "Objects stay at CTAN's own paths; `.state/` and `.site/` are the two exceptions".
- **Runner rsync version.** `ubuntu-latest` is expected to carry rsync 3.2.7 or later;
  `--secluded-args`, `--ignore-missing-args`, `--no-h` and `--contimeout` all predate 3.2;
  add `rsync --version` next to `aws --version` in the workflow to record it.
- **Listing wire size and real-rsync listing time** from the runner (`--stats`).
- **Does dante serve a package mid-write?** Unknowable from a listing; harmless to the
  design, relevant to how often a package straddles two hours.
- **ETag = MD5 for single-part R2 objects**: one `head-object` after the first run.
- **AWS CLI default checksums against R2** (`request_checksum_calculation`): for
  `errors-and-issues.md`.
- **Server-side copies for the five installers and the alias files**: worth it only if
  release-day hours prove slow.
- **Transfer rates** from dante and to R2 on the runner: the first hourly runs' `--stats`.
- For siblings: purge per batch and the width of the tlpdb-before-purge window
  (`caching.md`); `multipart_threshold`/`multipart_chunksize` for the 16 objects over
  200 MB and the 5 over 5 GiB, and the Class A they cost (`limits.md`,
  `cost-estimates.md`); how the workflow picks the reconcile hour and passes it to `task`
  (`taskfile-architecture.md`); the delta form of the five `verify` checks and the
  "drop from staging, do not record" rule (`verification-and-security.md`); the seed as
  an ordinary run with an empty or bootstrapped state (`seeding-and-migration.md`).

## Appendix: the listing analysis scripts

`SCRATCH/norm.awk` is in section 3. The others, referenced above:

```
# SCRATCH/utc.awk: Pacific local time (what the listings carry) -> UTC "yyyy/mm/dd hh"
function utc(d, t,   a, b, y, m, dd, h, off, mdays) {
  split(d, a, "/"); split(t, b, ":"); y=a[1]+0; m=a[2]+0; dd=a[3]+0; h=b[1]+0
  off = (d >= "2026/03/08" && d < "2026/11/01") || (d >= "2025/03/09" && d < "2025/11/02") ? 7 : 8
  h += off; if (h >= 24) { h -= 24; dd++; split("31 28 31 30 31 30 31 31 30 31 30 31", mdays, " ")
    if (y%4==0) mdays[2]=29; if (dd > mdays[m]) { dd=1; m++; if (m>12) { m=1; y++ } } }
  return sprintf("%04d/%02d/%02d %02d", y, m, dd, h) }

# SCRATCH/h30.awk: hour-of-day histogram, last 30 days, over state.tsv (path<TAB>size<TAB>"date time")
BEGIN{FS="\t"} {split($3,x," "); k=utc(x[1],x[2]); if (k >= "2026/07/28") {h=substr(k,12,2); n[h]++; b[h]+=$2}}
END{for(h in n) printf "%s %6d %8.1f\n", h, n[h], b[h]/1e6}

# SCRATCH/hday.awk: one day by hour; awk -v d=2026/03/01 -f utc.awk -f hday.awk state.tsv
BEGIN{FS="\t"} {split($3,x," "); k=utc(x[1],x[2]); if (substr(k,1,10)==d) {h=substr(k,12,2); n[h]++; b[h]+=$2}}
END{for(h in n) printf "%s %6d %9.1f\n", h, n[h], b[h]/1e6}

# SCRATCH/slot.awk: files and bytes per UTC hour-slot, last 365 days
BEGIN{FS="\t"} {split($3,x," "); k=utc(x[1],x[2]); if (k >= "2025/08/27") {n[k]++; b[k]+=$2}}
END{for(k in n) print k, n[k], b[k]}
```

`state.tsv` is `RUN/U` built by the pipeline in section 3 from `ctan-list-deref.txt`;
`stored.norm` is the same set in `norm.awk`'s five-column form. The mtime-spread and
symlink commands in sections 1 and 2 read `stored.norm`, `nolink.norm` and `deref.norm`.

## Sources

Fetched 2026-08-26/27:

- https://ctan.org/mirrors/register (raw HTML via curl; quoted above)
- https://ctan.org/mirrors
- https://www.tug.org/texlive/acquire-mirror.html (curl with a browser user agent; the
  page returns 403 to other agents)
- https://mirror.ctan.org/timestamp, /README.structure, /README.uploads, /README.mirrors,
  /CTAN.sites, /FILES.last07days, /FILES.byname (first 400 bytes)
- https://download.samba.org/pub/rsync/rsync.1 and /rsyncd.conf.5 (3.5.0 manuals)
- https://download.samba.org/pub/rsync/src/rsync-3.5.0.tar.gz (built locally for the
  fixture tests)
- https://raw.githubusercontent.com/RsyncProject/rsync/master/util1.c (`timestring`),
  /NEWS.md (`--list-only` and `--human-readable` history)
- https://developers.cloudflare.com/r2/api/s3/api/ (PutObject/CopyObject/UploadPartCopy
  feature tables), /r2/objects/multipart-objects/ (ETag formula),
  /r2/api/workers/workers-api-reference/ (MD5 on non-multipart objects),
  /r2/buckets/object-lifecycles/ (default multipart abort), /r2/platform/limits/ (key
  length, single-part size)
- https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html and /s3/ls.html
- https://rclone.org/s3/ and https://rclone.org/changelog/
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
  (schedule delays, 60-day disable) and
  https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows
  (10 GB, 7-day eviction)
- Local: `SCRATCH/ctan-list-deref.txt`, `SCRATCH/ctan-list-nolink.txt`,
  `SCRATCH/rsync-time.txt`, `staging/tlpkg/` (the 20.6 MB `texlive.tlpdb`), the
  repository's `Taskfile.yml`, `aws.config`, `.github/workflows/sync.yml`, `CLAUDE.md`;
  the Actions run history of this repository (scheduled run created 2026-08-26 04:09:16
  UTC for the `30 3 * * *` slot)
- https://ctan.org/mirrors/mirmon (age bands and today's ages, read 2026-08-27 03:03 UTC)
