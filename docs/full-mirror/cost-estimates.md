# Cost estimates for the full CTAN mirror

Every number here is one of three things: computed from the listing taken 2026-08-26 ~17:00
UTC (command shown), read from a page fetched on 2026-08-26 (URL shown), or marked
*unverified* with the one thing that would settle it. Prices are from
https://developers.cloudflare.com/r2/pricing/ ("Last updated Aug 7, 2026").

Inputs, all local:

```sh
SCRATCH=$PWD/listings   # wherever the two rsync listings were saved
L=$SCRATCH/ctan-list-deref.txt     # rsync -rL --list-only rsync://rsync.dante.ctan.org/CTAN/  (538,289 lines)
N=$SCRATCH/ctan-list-nolink.txt    # rsync -r  --list-only, symlinks shown as links           (395,562 lines)
T=staging/tlpkg/texlive.tlpdb      # the tlpdb the tlnet mirror serves today
# Listing format: perms size yyyy/mm/dd hh:mm:ss path.  The stored set is every regular file
# except tlnet's versioned containers and the tlnet-root update-tlmgr-r* files, the two
# excludes today's fetch already carries.  Reused below as $STORED.
STORED='$1 ~ /^-/ && !($5 ~ /^systems\/texlive\/tlnet\// && $5 ~ /\.r[0-9]+\.tar\.xz$/) && $NF !~ /^systems\/texlive\/tlnet\/update-tlmgr-r/'
```

Two quirks of the listing that every command below allows for. macOS `awk` has no `mktime`,
so dates are compared as strings (`"2026/07/27 17:00:00"` sorts correctly). And 523 lines
(395 of them under `obsolete/`) carry no size at all; the listing was made with openrsync
and it prints a blank there. Byte sums use `$2 ~ /^[0-9]+$/` so those lines add nothing;
object counts include them. Their bytes are unknown; at the average object size they would
be 0.14 GB, under the rounding of every figure here.

## The figures in one place

| Item | Value |
|---|---|
| Stored set | 496,149 objects, 132.99 GB (123.86 GiB) |
| Storage bill at that size | $1.85/month (133 GB-month, minus 10 free, 123 × $0.015) |
| Fixed operations, any sync design that lists the bucket at most once a day | $0 |
| Hourly sync that lists the bucket twice, at 200 GB | $4.50/month (ops bill in whole millions) |
| One `scheme-full` install, uncached | 11,919 GETs, 5.51 GB; free until 27 installs a day |
| Ceiling of the cached design, if the cache is turned on (every object refetched twice a day) | $7.20 + storage, and it is not a hard ceiling |
| Seed month, seeded on the 15th | $0.98 storage, $0 operations |
| Second and third seed in one month | $4.50 for the month, together |
| Budget to adopt | $5/month; the run fails at 200 GB or 600k objects before upload; 800k Class A and 8M Class B month-to-date signal healthchecks |

## Storage set

### Derivation

```sh
# real files, symlinks shown as links; then what rsync -L yields; then the stored set
awk '$1 ~ /^-/ {n++} $1 ~ /^l/ {l++} $1 ~ /^d/ {d++} $1 ~ /^-/ && $2 ~ /^[0-9]+$/ {s+=$2} END {printf "%d files %.2f GB, %d symlinks, %d dirs\n", n, s/1e9, l, d}' $N
awk '$1 ~ /^-/ {n++} $2 ~ /^[0-9]+$/ && $1 ~ /^-/ {s+=$2} END {printf "deref: %d files %.2f GB\n", n, s/1e9}' $L
awk "$STORED"' {n++} '"$STORED"' && $2 ~ /^[0-9]+$/ {s+=$2} END {printf "stored: %d objects %d bytes %.2f GB %.2f GiB\n", n, s, s/1e9, s/2^30}' $L
```

| Measure | Objects | Bytes |
|---|---:|---:|
| Real files (`rsync -r`) | 352,357 | 68.98 GB; plus 24,788 symlinks, 18,417 directories |
| Dereferenced (`rsync -rL`) | 511,027 | 139.63 GB |
| tlnet versioned containers, excluded | 14,872 | 6.62 GB |
| `update-tlmgr-r*` at the tlnet root, excluded (today's `fetch` drops them; the `-latest` copies stay) | 6 | 0.01 GB |
| **Stored set** | **496,149** | **132,993,291,537 B = 132.99 GB = 123.86 GiB** |

Two things the numbers say about symlinks. Every one of the 24,788 symlinks resolves to a
file that already exists elsewhere in the tree, so the stored set is 133 GB of which 64 GB
(132.99 − 68.98) is copies. The adopted normaliser also drops tlcontrib's 261 versioned
containers (`systems/texlive/tlcontrib/archive/*.r[0-9]*.tar.xz`, 0.46 GB by this listing),
for the same reason as tlnet's: 495,888 objects, 132.53 GB, still 133 GB-month, so no dollar
figure below moves. The copies are in two shapes: 24,611 file symlinks that become
35.90 GB of objects (14,872 of them the tlnet containers we drop, 6.62 GB), and 177
directory symlinks whose contents become 27.15 GB more (below).

```sh
awk '$1 ~ /^l/ {print $5}' $N | LC_ALL=C sort > $SCRATCH/links.txt
awk '$1 ~ /^-/ && $2 ~ /^[0-9]+$/ {print $5, $2}' $L | LC_ALL=C sort -k1,1 > $SCRATCH/deref-sizes.txt
LC_ALL=C join -j1 $SCRATCH/links.txt $SCRATCH/deref-sizes.txt | awk '{n++; s+=$2} END {printf "file symlinks materialised: %d, %.2f GB\n", n, s/1e9}'
```

### Per top-level directory (stored set)

```sh
awk "$STORED"' && $2 ~ /^[0-9]+$/ {split($5,p,"/"); t=(p[2]==""?"(root)":p[1]); n[t]++; s[t]+=$2} END {for (t in n) printf "%-14s %8d %8.2f GB\n", t, n[t], s[t]/1e9}' $L | sort -k3 -nr
```

| Directory | Objects | GB | Note |
|---|---:|---:|---|
| `systems` | 59,264 | 92.03 | see below |
| `macros` | 107,350 | 13.43 | |
| `fonts` | 179,801 | 7.18 | 110k `tfm`, 36k `vf` |
| `obsolete` | 26,221 | 5.70 | |
| `graphics` | 19,444 | 2.95 | |
| `support` | 9,417 | 2.30 | |
| `info` | 28,888 | 2.21 | |
| `documentation` | 28,888 | 2.21 | symlink to `info` |
| `install` | 559 | 1.66 | the `.tds.zip` files |
| `bibliography` | 2,593 | 0.78 | symlink to `biblio` |
| `biblio` | 2,593 | 0.78 | |
| `languages` | 9,718 | 0.45 | symlink to `language` |
| `language` | 9,718 | 0.45 | |
| `usergrps` | 1,553 | 0.27 | |
| `web` | 3,753 | 0.20 | |
| `help` | 76 | 0.13 | |
| `indexing` | 754 | 0.10 | |
| `dviware` | 3,234 | 0.07 | |
| `digests` | 1,776 | 0.07 | symlink to `info/digests` |
| `tds` | 14 | 0.00 | symlink |
| root | 10 | 0.03 | `FILES.byname` 26.7 MB, `timestamp`, `index.html`, `tds.zip`, ... |

Inside `systems/`:

```sh
awk "$STORED"' && $5 ~ /^systems\// && $2 ~ /^[0-9]+$/ {split($5,p,"/"); t=p[1]"/"p[2]; if (p[2]=="texlive") t=t"/"p[3]; n[t]++; s[t]+=$2} END {for (t in n) printf "%-28s %8d %8.2f GB\n", t, n[t], s[t]/1e9}' $L | sort -k3 -nr | head -8
```

| Path | Objects | GB |
|---|---:|---:|
| `systems/windows` | 19,174 | 23.64 |
| `systems/win32` | 19,174 | 23.64 (symlink to `windows`) |
| `systems/texlive/Images` | 14 | 20.35 (one ISO stored three times) |
| `systems/mac` | 515 | 15.15 (one 6.87 GB `.pkg` stored twice) |
| `systems/texlive/tlnet` | 16,972 | 6.78 |
| `systems/texlive/tlcontrib` | 528 | 0.93 |
| `systems/chitex` | 5 | 0.81 |

### The alias directories

```sh
for d in bibliography digests documentation languages tds systems/win32; do
  awk -v d="$d" '$1 ~ /^-/ && index($5, d"/")==1 && $2 ~ /^[0-9]+$/ {n++; s+=$2} END {printf "%-14s %6d %7.3f GB\n", d, n, s/1e9}' $L; done
awk '$1 ~ /^l/ && $5 ~ /^(bibliography|digests|documentation|languages|tds|tds.zip|README.mirrors)$/ {print $5, $2}' $N   # link sizes = target length
```

| Alias | Target (from link length) | Objects | GB |
|---|---|---:|---:|
| `systems/win32` | `windows` | 19,174 | 23.644 |
| `documentation` | `info` (4) | 28,888 | 2.207 |
| `bibliography` | `biblio` (6) | 2,593 | 0.777 |
| `languages` | `language` (8) | 9,718 | 0.450 |
| `digests` | `info/digests` (12) | 1,776 | 0.075 |
| `tds`, `tds.zip`, `README.mirrors` | | 16 | 0.001 |
| **Total** | | **62,165** | **27.15** |

Storing the aliases costs 27 GB-month, $0.41/month. Six Cloudflare redirect rules would
save it (Redirect Rules are free on the Free plan; count not checked here, see limits.md).
Copies keep the state in the bucket; rules put it in the dashboard. The choice is
taskfile-architecture.md's; the cost of either is in the table below.

### Largest objects and the two size limits

```sh
awk "$STORED"' && $2 ~ /^[0-9]+$/ {printf "%d %s\n", $2, $5}' $L | sort -nr | head -12
awk "$STORED"' && $2 ~ /^[0-9]+$/ {if ($2>5363466240) a++; if ($2>536870912) b++; if ($2>4294967296) c++; if ($2>100e6) d++; if ($2<1e6) e++; if ($2<10000) f++} END {print ">4.995GiB", a, ">512MiB", b, ">4GB(aws.config)", c, ">100MB", d, "<1MB", e, "<10kB", f}' $L
```

| Bytes | Object |
|---:|---|
| 6,865,013,189 | `systems/mac/mactex/MacTeX.pkg` and `mactex-20260324.pkg` (2) |
| 6,784,798,720 | `systems/texlive/Images/texlive.iso`, `texlive2026.iso`, `texlive2026-20260301.iso` (3) |
| 1,138,914,783 | `obsolete/systems/windows/protext/protext.zip` and `protext-3.2-031721.zip` (2) |
| 480,437,007 | `systems/chitex/Windows/ps635Mk29a26.exe` |
| 413,638,248 | `systems/windows/w32tex/ltxpkgdocs.tar.xz` (and the `win32` copy) |
| 324,971,254 | `systems/chitex/Unix/chitexu_15a63.tar.gz` |

| Threshold | Objects | Why it matters |
|---|---:|---|
| Over 4.995 GiB (R2 single-part limit, [limits](https://developers.cloudflare.com/r2/platform/limits/), footnote 4: "5 MiB less than 5 GiB") | 5 | multipart is mandatory |
| Over 512 MiB (Free plan cacheable size, [default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#cacheable-size-limits)) | 7 | never edge-cached; every download is Class B |
| Over 4 GB (`multipart_threshold = 4GB`, the adopted `aws.config`) | 5 | the same five; nothing else goes multipart |
| Over 100 MB | 48 | |
| Under 1 MB | 485,720 (97.9%) | the bucket is small files |
| Under 10 kB | 330,484 (66.6%) | |

Average object 268,051 bytes (132.99 GB / 496,149). At that average, 200 GB is 746k objects
and 300 GB is 1,119k; the larger-bucket rows below use those counts.

### File-type mix

```sh
awk "$STORED"' && $2 ~ /^[0-9]+$/ {f=$5; sub(/.*\//,"",f); e=(f ~ /\./)? tolower(f): "(none)"; sub(/.*\./,"",e); n[e]++; s[e]+=$2} END {for (e in n) printf "%-8s %8d %8.3f GB\n", e, n[e], s[e]/1e9}' $L > $SCRATCH/ext.txt
sort -k2 -nr $SCRATCH/ext.txt | head -12; sort -k3 -nr $SCRATCH/ext.txt | head -10
```

By count: `tfm` 109,873; `vf` 35,806; `lzma` 34,764; `pdf` 33,802; `tex` 32,071; `ltx`
28,158; `xz` 15,730; `zip` 14,357; no extension 13,782; `mf` 11,818; `pfb` 11,268; `sty`
9,079. By bytes: `lzma` 24.12 GB (MiKTeX packages, twice); `zip` 20.42; `iso` 20.35; `pkg`
14.70; `xz` 14.32; `pdf` 8.34; `deb` 8.21; `rpm` 6.37; `gz` 2.81; `exe` 2.23.

Of these, Cloudflare caches `zip`, `pdf`, `gz`, `iso`, `exe`, `tar`, `7z`, `bz2`, `zst` by
default and does not cache `xz`, `lzma`, `tfm`, `sty`, `tex`, `pkg`, `deb`, `rpm`
([default extensions](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#default-cached-file-extensions),
fetched today). Without a cache rule, 33 GB of the 133 (`xz`+`lzma`+`pkg`+`deb`+`rpm`) and
every tlnet container miss the cache on every request. That is why caching.md exists.

### Smaller storage sets, priced

```sh
NOALIAS='!($5 ~ /^(bibliography|digests|documentation|languages|tds)\// || $5 ~ /^systems\/win32\// || $5=="tds.zip" || $5=="README.mirrors")'
NOINST='!($5 ~ /^systems\/texlive\/Images\// || $5 ~ /^systems\/mac\/mactex\//)'
size() { awk "$STORED && $1"' {n++} '"$STORED && $1"' && $2 ~ /^[0-9]+$/ {s+=$2} END {printf "%d objects %.2f GB\n", n, s/1e9}' $L; }
size 1; size "$NOALIAS"; size "$NOINST"; size "$NOALIAS && $NOINST"; size '$5 !~ /^obsolete\//'
size '$5 !~ /^systems\//'; size '($5 !~ /^systems\// || $5 ~ /^systems\/texlive\/tlnet\//)'; size "$NOALIAS && $NOINST"' && $5 !~ /^obsolete\//'
```

Storage price = (usage rounded up to whole GB-month − 10 free) × $0.015.

| Set | Objects | GB | $/month |
|---|---:|---:|---:|
| Stored set (everything `rsync -rL` yields minus tlnet's versioned containers and `update-tlmgr-r*`) | 496,149 | 132.99 | 1.85 |
| minus `obsolete/` | 469,928 | 127.29 | 1.77 |
| minus the alias directories | 433,984 | 105.84 | 1.44 |
| minus the installers (`systems/texlive/Images`, `systems/mac/mactex`) | 496,092 | 98.00 | 1.32 |
| minus aliases and installers | 433,927 | 70.84 | 0.92 |
| minus aliases, installers and `obsolete/` | 407,706 | 65.14 | 0.84 |
| minus all of `systems/` except tlnet | 453,857 | 47.74 | 0.57 |
| minus all of `systems/` | 436,885 | 40.96 | 0.47 |
| tlnet alone (today's mirror) | 16,972 | 6.78 | 0.00 |

Dropping the installers is the only cut that changes anything other than the storage line:
it removes the 5 multipart objects, the 7 uncacheable ones, and the two 14–21 GB release
days from the churn. Every set except tlnet-alone costs the same $0 in operations.
Whether a partial tree can be an official mirror is a question for
official-mirror-and-url.md; the register page asks for the whole archive.

## Prices, re-verified 2026-08-26

Source: https://developers.cloudflare.com/r2/pricing/ ("Last updated Aug 7, 2026"). Quoted.

| Item | Standard | Free per month |
|---|---|---|
| Storage | $0.015 / GB-month | 10 GB-month |
| Class A operations | $4.50 / million | 1 million |
| Class B operations | $0.36 / million | 10 million |
| Egress | Free | |
| Infrequent Access | $0.01 / GB-month storage, $9.00/M Class A, $0.90/M Class B, $0.01/GB retrieval, 30-day minimum; "The free tier only applies to Standard storage" | none |

- Class A: "`ListBuckets`, `PutBucket`, `ListObjects`, `PutObject`, `CopyObject`,
  `CompleteMultipartUpload`, `CreateMultipartUpload`, `LifecycleStorageTierTransition`,
  `ListMultipartUploads`, `UploadPart`, `UploadPartCopy`, `ListParts`, ..."
  So one multipart upload of *p* parts is *p* + 2 Class A.
- Class B: "`HeadBucket`, `HeadObject`, `GetObject`, `UsageSummary`, ..."
- Free: "`DeleteObject`, `DeleteBucket` and `AbortMultipartUpload`".
- GB-month: "calculated by averaging the *peak* storage per day over a billing period (30
  days)". Example on the page: "Storing 1 GB for 5 days, then 3 GB for the remaining 25 days
  will be charged as 1 GB * 5/30 month + 3 GB * 25/30 month = 2.66 GB-month".
- Rounding: "Cloudflare rounds up your usage to the next billing unit. If you have
  performed one million and one operations, you will be billed for two million operations.
  If you have used 1.1 GB-month, you will be billed for 2 GB-month."
  **The billing unit for operations is one million.** Any Class A overage costs at least
  $4.50 and any Class B overage at least $0.36. Every overage below is priced in that unit:
  1.1M Class A in a month is $4.50, not $0.45.
- Failed requests: "You are not charged for operations when the caller does not have
  permission to make the request (HTTP 401 `Unauthorized` response status code)." Nothing
  else is exempted; a 404 `GetObject` is a Class B and a `PutObject` that fails with 5xx is
  not documented as free.
- Whether "GB" in the storage meter is 10^9 or 2^30 bytes: the pricing page does not say;
  the limits page says its own GiB figures are base-2 "distinct from 1 GB (gigabyte), which
  is 10^9 bytes". This file uses 10^9 (132.99 GB), the larger bill. If the meter is base-2 the
  same bucket is 123.87 → 124 − 10 = 114 GB-month, $1.71. *Unverified*; the storage
  dataset's `payloadSize` after the seed settles it.

Cross-check of the page's own examples: asset hosting, 10M reads/day × 30 = 300M, minus 10M
free = 290M × $0.36 = **$104.40**, as printed. Standard storage example: 990 GB-month ×
$0.015 = **$14.85**, as printed. Both arithmetic checks pass with the prices above.

## Fixed monthly cost

### Storage

| Bucket | Usage | Billable GB-month | $/month |
|---|---:|---:|---:|
| Stored set | 132.99 | 123 | 1.85 |
| 175 GB | 175 | 165 | 2.48 |
| 200 GB (the ceiling proposed below) | 200 | 190 | 2.85 |
| 300 GB | 300 | 290 | 4.35 |

### Class A, per sync design

Inputs: `ListObjects` returns 1,000 keys a page, so a full listing is ⌈objects/1000⌉ Class A:
497 today, 747 at 200 GB (746k objects), 1,120 at 300 GB (1,119k). PutObjects per month are
the 30-day churn (16,490 files on the stored set, computed below), scaled by size for the
larger buckets (24,797 at 200 GB, 37,195 at 300 GB); plus 450 for the 15 keys `aws s3 sync`
re-uploads every run under the current `publish`. 30 days, 720 hourly slots; scheduled runs
start 15 to 45 minutes late and slots can be dropped (observed in this repository: the
03:30 slot of 2026-08-26 started at 04:09:16 UTC), which only lowers these counts.

| Design | Class A per month at 133 GB | at 200 GB | at 300 GB |
|---|---:|---:|---:|
| Daily, today's `publish` (sync lists once, `stale` lists once): 30 × 2 × pages + churn + 450 | 29,820 + 16,940 = **46,760 → $0** | 44,820 + 25,247 = **70,067 → $0** | 67,200 + 37,645 = **104,845 → $0** |
| Hourly, listing twice: 720 × 2 × pages + churn + 450 | 715,680 + 16,940 = **732,620 → $0** | 1,075,680 + 25,247 = **1,100,927 → $4.50** | 1,612,800 + 37,645 = **1,650,445 → $4.50** |
| Hourly, listing once: 720 × pages + churn + 450 | 357,840 + 16,940 = **374,780 → $0** | 537,840 + 25,247 = **563,087 → $0** | 806,400 + 37,645 = **844,045 → $0** (84% of the free tier) |
| Hourly, state file, daily reconcile: 30 × pages + 720 state writes + churn | 14,910 + 720 + 16,490 = **32,120 → $0** | 22,410 + 720 + 24,797 = **47,927 → $0** | 33,600 + 720 + 37,195 = **71,515 → $0** |

The churn figure is a floor for PutObjects: it counts files whose *current* mtime is inside
the window, so a file changed twice in a month counts once. An hourly sync uploads each
change. The tlnet archive shows how bursty this is (82 containers on 2026-08-15, 2 on
08-05), but the double-change rate is not measurable from one listing. *Unverified*; the
`report` counts of the first month measure it. Even at 3× it stays under 100k.

Class B from the sync itself: one `GetObject` of the state file per hourly run (720), the
`smoke` reads through the domain (≈8 a run, deliberately uncached, 5,760), and nothing
else. `aws s3 sync` and `aws s3 cp` do not read objects. ≈6.5k a month against 10M.

### Churn, from mtimes

```sh
# windows back from 2026/08/26 17:00:00; the cut is a string compare on "date time"
for c in "7 2026/08/19" "30 2026/07/27" "90 2026/05/28" "365 2025/08/26"; do set -- ${=c}
  awk -v cut="$2 17:00:00" "$STORED"' && $2 ~ /^[0-9]+$/ && ($3" "$4) >= cut {n++; s+=$2} END {printf "%3d days: %6d files %7.2f GB\n", '"$1"', n, s/1e9}' $L; done
# per-day histogram
awk -v cut="2026/06/27" "$STORED"' && $2 ~ /^[0-9]+$/ && $3 >= cut {n[$3]++; s[$3]+=$2} END {for (d in n) printf "%s %6d %8.3f\n", d, n[d], s[d]/1e9}' $L | sort
```

| Window | Stored set | Without alias directories | Without installers |
|---|---|---|---|
| 7 days | 8,824 files, 1.43 GB | 8,809 files, 1.41 GB | |
| 30 days | 16,490 files, 4.71 GB | 15,659 files, 3.88 GB | |
| 90 days | 33,463 files, 11.31 GB | | |
| 365 days | 82,586 files, 62.42 GB | 76,040 files, 55.87 GB | 82,531 files, 27.42 GB |

Of the 62.42 GB that changed in a year, 35 GB is the two installer days. 354 of the last 366
days have at least one file whose latest mtime is that day; the median day is 59 files and
14 MB.

The last 60 days, per day (files, GB):

```
06/27 39 0.005 | 06/28 860 0.751 | 06/29 568 0.025 | 06/30 57 0.138 | 07/01 189 0.076
07/02 18 0.011 | 07/03 50 0.006 | 07/04 75 0.014 | 07/05 548 0.179 | 07/06 70 0.142
07/07 63 0.005 | 07/08 1966 0.650 | 07/09 4244 0.196 | 07/10 31 0.007 | 07/11 576 0.816
07/12 42 0.003 | 07/13 91 0.007 | 07/14 32 0.006 | 07/15 30 0.003 | 07/16 24 0.003
07/17 200 0.019 | 07/18 95 0.048 | 07/19 34 0.006 | 07/20 97 0.010 | 07/21 72 0.011
07/22 403 0.021 | 07/23 89 0.073 | 07/24 22 0.001 | 07/25 13 0.001 | 07/26 566 0.173
07/27 78 0.003 | 07/28 45 0.005 | 07/29 81 0.007 | 07/30 114 0.014 | 07/31 1359 0.111
08/01 848 0.083 | 08/02 334 0.050 | 08/03 427 0.075 | 08/04 100 0.043 | 08/05 52 0.038
08/06 651 0.308 | 08/07 244 0.041 | 08/08 86 0.030 | 08/09 1005 0.893 | 08/10 272 0.184
08/11 112 0.036 | 08/12 97 0.018 | 08/13 634 0.318 | 08/14 459 0.159 | 08/15 113 0.272
08/16 99 0.376 | 08/17 420 0.131 | 08/18 8 0.073 | 08/19 127 0.030 | 08/20 768 0.071
08/21 1702 0.253 | 08/22 66 0.038 | 08/23 5911 0.606 | 08/24 110 0.210 | 08/25 97 0.137
```

Outlier days in the last year:

```sh
awk -v cut="2025/08/26" "$STORED"' && $2 ~ /^[0-9]+$/ && $3 >= cut {n[$3]++; s[$3]+=$2} END {for (d in n) printf "%s %6d %8.3f\n", d, n[d], s[d]/1e9}' $L | sort -k3 -nr | head -3
awk "$STORED"' && $2 ~ /^[0-9]+$/ && $3 == "2026/03/01" {split($5,p,"/"); t=p[1]"/"p[2]"/"p[3]; n[t]++; s[t]+=$2} END {for (t in n) printf "%-30s %5d %7.3f\n", t, n[t], s[t]/1e9}' $L | sort -k3 -nr | head -3
```

| Day | Files | GB | What |
|---|---:|---:|---|
| 2026-03-01 | 1,773 | 20.94 | TeX Live release: `systems/texlive/Images` 20.35 (three copies of one ISO) |
| 2026-03-24 | 293 | 14.01 | MacTeX: `systems/mac/mactex` 13.97 (two copies of one `.pkg`) |
| 2026-06-07 | 956 | 1.09 | the largest ordinary day |
| 2026-01-12 | 7,782 | 0.06 | most files in a day |
| 2026-08-23 | 5,911 | 0.61 | `macros/context` 4,928 files |

Cost consequence of a release day: 20.94 GB is 21 PutObject-equivalents of the 5 GB kind
(multipart, ~2,500 Class A at 8 MB parts) plus ~1,800 PutObjects: nothing on the bill.
Storage peaks do not spike either, because the ISO replaces last year's at the same size.

## Traffic-driven cost

### One user

`scheme-full` on `x86_64-linux`, closure over `depend` lines in the tlpdb, `.ARCH`
expanded to the one platform:

```sh
awk '/^name /{n=$2} /^category /{cat[n]=$2} /^depend /{dep[n]=dep[n]" "$2}
/^containersize /{cs[n]=$2} /^doccontainersize /{ds[n]=$2} /^srccontainersize /{ss[n]=$2}
/^containerchecksum /{hc[n]=1} /^doccontainerchecksum /{hd[n]=1} /^srccontainerchecksum /{hs[n]=1}
END { todo["scheme-full"]=1; ch=1
  while (ch) { ch=0; for (p in todo) { m=split(dep[p], d, " "); for (i=1;i<=m;i++) { x=d[i]; if (x=="") continue
    if (x ~ /\.ARCH$/) sub(/\.ARCH$/, ".x86_64-linux", x); if (!(x in todo) && (x in cat)) { todo[x]=1; ch=1 } } } }
  for (p in todo) { if (hc[p]) {gc++; bc+=cs[p]} if (hd[p]) {gd++; bd+=ds[p]} if (hs[p]) {gs++; bs+=ss[p]} }
  printf "run %d GETs %.3f GB; doc %d %.3f; src %d %.3f; all %d %.3f\n", gc, bc/1e9, gd, bd/1e9, gs, bs/1e9, gc+gd+gs, (bc+bd+bs)/1e9 }' $T
```

| Install | Container GETs | GB |
|---|---:|---:|
| run containers only | 5,170 | 1.683 |
| run + doc | 9,891 | 5.391 |
| run + doc + src (`scheme-full` default) | 11,913 | 5.509 |
| plus the installer's own reads: `install-tl-unx.tar.gz` + `.sha512` + `.asc`, `tlpkg/texlive.tlpdb.xz` + `.sha512` + `.asc` | 6 | 0.008 |
| **Total** | **11,919** | **5.52** |

Every GET that reaches R2 is one Class B: $0.0043 per uncached install after the free tier,
$4.29 per thousand. The free tier is 10,000,000 / 11,919 = 839 installs a month, 27 a day.
The 28th install a day trips it.

A `tlmgr update` for that user fetches the containers that changed since their last run.
From the listing, tlnet containers with an mtime in the last 30 days:

```sh
awk -v cut="2026/07/27 17:00:00" '$1 ~ /^-/ && $5 ~ /^systems\/texlive\/tlnet\/archive\// && !($5 ~ /\.r[0-9]+\.tar\.xz$/) && ($3" "$4) >= cut {n++; s+=$2; f=$5; sub(/.*\//,"",f)
  if (f ~ /\.doc\.tar\.xz$/) {nd++; sd+=$2} else if (f ~ /\.source\.tar\.xz$/) {ns++; ss+=$2}
  else if (f ~ /\.(x86_64|aarch64|i386|amd64|armhf|universal|windows)[-a-z0-9_]*\.tar\.xz$/) {na++; sa+=$2; if (f ~ /\.x86_64-linux\.tar\.xz$/) {nl++; sl+=$2}} else {nr++; sr+=$2}}
  END {printf "all %d %.3f GB; run %d %.3f; doc %d %.3f; src %d %.3f; arch %d %.3f (x86_64-linux %d %.3f)\n", n, s/1e9, nr, sr/1e9, nd, sd/1e9, ns, ss/1e9, na, sa/1e9, nl, sl/1e9}' $L
```

457 containers, 0.83 GB, changed in 30 days across all platforms: 178 run (0.077 GB), 161
doc (0.435), 74 source (0.012), 44 platform binaries (0.306, of which 4 are `x86_64-linux`).
A `scheme-full` Linux user who updates once a month fetches **417 containers, 0.55 GB**, plus
3 `tlpkg/` reads. One who runs `tlmgr update` daily adds 90 `tlpkg/` reads: ~507 GETs a
month. A `scheme-medium` user holds about a third of the packages and fetches about a third
of that.

A human sent by `ctan.org` to a package: one GET. `macros/latex/contrib/*.zip` averages
807 KB (median 281 KB, 2,710 files); `install/*.tds.zip` averages 3.0 MB. A directory URL is
a 404 and a Class B all the same.

```sh
awk '$1 ~ /^-/ && $5 ~ /^macros\/latex\/contrib\/[^\/]+\.zip$/ {print $2}' $L | sort -n | awk '{a[NR]=$1; s+=$1} END {printf "n=%d avg=%.0f median=%d\n", NR, s/NR, a[int(NR/2)]}'
```

### A mirror's share of CTAN traffic

What is published:

- https://ctan.org/mirrors/register (fetched today): "we redirect requests for file
  downloads to our mirrors ... `mirrors.ctan.org`, which sends the user to a
  randomly-selected official mirror in their region", and "The traffic is not too much".
  No figures.
- https://ctan.org/mirrors/mirmon (fetched today): 132 sites in 40 regions, 11 in the
  United States. `CTAN.sites` (fetched from `mirror.ctan.org`) lists 134 hosts, 12 marked
  USA. A new US mirror therefore receives roughly one thirteenth of US redirects.
- https://ftp.fau.de/news/ (fetched today): in that mirror's 2018 annual report CTAN was its
  10th-largest project at **95 TB for the year**, about 7.9 TB a month for one large
  European mirror. This is the only measured per-mirror figure found. It is from 2018 and
  CTAN dropped out of that site's top ten afterwards.
- Wikipedia (fetched today): "The main CTAN nodes serve downloads of more than 6 TB per
  month, not counting its 94 mirror sites worldwide." **No citation on the page.** Uncited.
- `tlmgr` resolves `mirror.ctan.org` once per invocation and then talks to the chosen host,
  so one install is one mirror choice (TLUtils.pm in `staging/tlpkg/TeXLive/`; the
  redirect observed today: `mirror.ctan.org/timestamp` → 307 → `latex.us`).

Turning 7.9 TB a month into requests needs an object mix nobody publishes. Two brackets: if
every byte were a `scheme-full` install (462 KB per GET) it is 17M GETs a month; if every byte
were an average object (268 KB) it is 30M. So a mirror carrying what ftp.fau.de carried in
2018 would see **17M to 30M GETs a month: $2.52 to $7.20 uncached, $0 cached.** A one-
thirteenth share of US redirects is *unverified* in size; the one thing that measures it is
this mirror's own zone analytics after a month on the rotation, which monitoring.md reads.

### Uncached

Class B bill = ⌈(GETs − 10M) / 1M⌉ × $0.36, zero at or under 10M. Storage $1.85 is added
in the last column.

| `scheme-full` installs/day | GETs/month | Egress/month | Class B | Total |
|---:|---:|---:|---:|---:|
| 1 | 357,570 | 166 GB | $0 | $1.85 |
| 10 | 3,575,700 | 1.66 TB | $0 | $1.85 |
| 27 | 9,654,390 | 4.5 TB | $0 | $1.85 |
| 28 | 10,011,960 | 4.6 TB | $0.36 | $2.21 |
| 100 | 35,757,000 | 16.6 TB | $9.36 | $11.21 |
| 1,000 | 357,570,000 | 166 TB | $125.28 | $127.13 |
| 10,000 | 3,575,700,000 | 1.66 PB | $1,283.76 | $1,285.61 |

Egress is free at every row ([pricing](https://developers.cloudflare.com/r2/pricing/)
footnote 1, and "Bandwidth: Included in all plans. Cloudflare does not charge for
bandwidth" at
https://developers.cloudflare.com/billing/understand/how-charges-accrue/).

### Cached: the future state

The adopted design ships with the edge cache off (`CACHE: off`, caching.md), so the
uncached table above is the bill as planned. This table is what turning the cache on buys
(cache rule for everything but the decision files, Smart Tiered Cache, purge by URL).

Model: the 6 decision-file reads per install always reach R2; the hot set (11,913 install
containers plus ~3,200 platform binaries, ~15k objects) is refilled from R2 twice a day
(miss factor 2); every purged key is refetched once per upper tier (16,490 × 2 a month).
Class B = 6 × installs × 30 + 15,000 × 2 × 30 + 33,000 ≈ 933k + 180 × installs/day.

| installs/day | Class B/month | Class B bill | Total |
|---:|---:|---:|---:|
| 1 to 1,000 | 0.93M to 1.11M | $0 | $1.85 |
| 10,000 | 2.7M | $0 | $1.85 |
| 100,000 | 18.9M | $3.24 | $5.09 |

### The "ceiling with caching", examined

The ceiling argument: every object refetched from R2 once a day, times two.
496,149 × 2 × 30 = 29,769,300 Class B → 19.77M billable → 20M → **$7.20**, plus storage
**$9.05** at 133 GB. At 200 GB (746k objects):
44.8M → 35M → $12.60 + $2.85 = $15.45. At 300 GB: 67.1M → 58M → $20.88 + $4.35 = $25.23.

Each assumption under it, checked:

| Assumption | Holds? | What was checked |
|---|---|---|
| Cache hits cost no R2 operation | Yes, first party | "Every cache hit avoids origin fetch costs, Argo routing charges, Workers execution, and R2 operations", https://developers.cloudflare.com/billing/understand/how-charges-accrue/ |
| A cache rule can make `.xz`, `.lzma`, `.pkg`, `.deb`, `.rpm` cacheable on Free | Yes | Cache Rules: Free plan, 10 rules, https://developers.cloudflare.com/cache/how-to/cache-rules/#availability; "Eligible for cache" with "Ignore cache-control header and use this TTL", https://developers.cloudflare.com/cache/how-to/cache-rules/settings/ |
| Objects over 512 MB cache | **No.** 7 objects never cache | "Free, Pro and Business customers have a limit of 512 MB", https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#cacheable-size-limits; and "Cloudflare will still enforce the plan-based cacheable file limits" under Cache Rules settings |
| Smart Tiered Cache guarantees one origin fetch per object | **No.** It "reduces the number of requests that reach R2" by routing misses through one upper tier; nothing on the page promises retention | https://developers.cloudflare.com/cache/how-to/tiered-cache/, https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/ ("Each edge data center that gets a cache miss fetches content directly from R2" without it) |
| Miss factor 2 | *Unverified.* Cloudflare publishes no eviction policy or retention for Free zones. 133 GB of long tail across hundreds of PoPs will evict; the hot 6.8 GB tlnet set at busy PoPs probably will not | the zone's `cacheStatus` breakdown after a month (monitoring.md) |
| Range requests do not fragment the cache | Mostly yes | Cloudflare serves `Range` from a cached copy as 206 when the origin sent `Content-Length` (default cache behavior page, "Client side range requests"); R2 sends `Content-Length` and `Accept-Ranges: bytes`, and a `curl -r 0-1023` through the domain returned 206 today. On a miss the docs say Cloudflare "will continue to retrieve and cache the entire content" and request-collapses concurrent misses per data center. So a ranged download of a cacheable object costs one origin fetch per data-center miss, not one per range. |
| `tlmgr` does not use Range | Yes | `FallbackDownloaderArgs` in `staging/tlpkg/TeXLive/TLConfig.pm` lines 148–155: curl gets `--retry 4 --fail --location --silent --output`, wget `--tries=4 -q -O`; no `--range`, `-r`, `-c` |
| Browsers and download managers | Browsers: no Range on a plain download. Download managers: N parallel ranges. On a **cacheable** object that is one fill then hits. On the 7 **uncacheable** objects every range is a separate `GetObject`: an ISO fetched with 16 connections is 16 Class B, and a resumed download is one more per resume | uncacheable per the 512 MB limit above |
| Decision files never cached | By design, and they scale with clients, not objects: 6 per install, 3 per `tlmgr update` run, 1 `timestamp` per mirmon probe, `FILES.*` per script | 1,000 `tlmgr update` runs a day = 90k a month; 100,000 a day = 9M, which alone fills the free tier |
| Crawlers | Not in the model. A crawler that walks the tree makes 496k GETs plus 18,417 directory 404s (each a Class B unless the 404 is cached; Cloudflare caches 404 for 3 minutes by default and a cache rule can set "Cache TTL by status code", https://developers.cloudflare.com/cache/how-to/configure-cache-status-code/). Five full crawls a month uncached = 2.6M Class B, $0; twenty-five = 12.9M, $1.08 | zone analytics by user agent |

So the ceiling is a ceiling on *object-driven* misses only. The client-driven terms
(decision files, the 7 big objects, 404s) are linear in traffic at $0.36 per million and
have no cap. At any traffic this mirror can plausibly draw (the ftp.fau.de bracket, 17–30M
GETs a month all-in) the cached bill is $0 to $0.36 and the uncached bill is $2.52 to $7.20.
Turning the cache on saves nothing at today's traffic; it caps the exposure if the mirror
becomes popular, and the exposure it does not cap is small per request. That is why the
adopted default is `CACHE: off` and the switch is caching.md's decision, not a cost one.

## Seed month

Uploads. 496,149 objects; with the adopted `aws.config` (`multipart_threshold = 4GB`,
`multipart_chunksize = 512MB`) exactly the five objects over 4.995 GiB go multipart, in 13
parts each. The CLI's own defaults are 8 MB for both
(https://docs.aws.amazon.com/cli/latest/topic/s3-config.html: "multipart_threshold Default
8MB", "multipart_chunksize Default 8MB, Minimum For Uploads 5MB"; R2: "5 MiB – 5 GiB per
part", 10,000 parts, https://developers.cloudflare.com/r2/objects/upload-objects/). One
multipart upload of *p* parts costs *p* + 2 Class A (`CreateMultipartUpload`, *p* ×
`UploadPart`, `CompleteMultipartUpload`, all listed as Class A on the pricing page).

```sh
awk -v th=4294967296 -v c=536870912 -v c8=8388608 "$STORED"' && $2 ~ /^[0-9]+$/ && $2 > th {n++; p=int(($2+c-1)/c); p8=int(($2+c8-1)/c8); t+=p+2; t8+=p8+2} END {printf "objects %d; Class A at 512 MB parts %d; at the CLI default 8 MB %d\n", n, t, t8}' $L
```

| Item | Class A |
|---|---:|
| 496,144 single `PutObject` | 496,144 |
| 5 multipart objects, 13 parts each at 512 MB: 15 Class A per installer | 75 |
| (the same five at the CLI's 8 MB default would be 4,075; at today's 200 MB threshold 14 objects go multipart) | |
| state-file writes (one per batch, ~34) | 34 |
| the ordinary hourly month on top (churn 16,490 + 720 state writes) | 17,210 |
| **Seed month total** | **≈513,500 → $0** |

A part costs the same Class A whether it is 8 MB or 512 MB, so the chunk size is worth
$0.02 a seed at $4.50/M; the 4 GB threshold and 512 MB parts are for the part count and the
retry granularity, not the bill.

Storage, prorated by the daily-peak rule. The bucket holds 6.8 GB of tlnet until the seed
lands. Seeded on day 15 and finished the same day: (14 × 6.8 + 16 × 132.99) / 30 = 74.1
GB-month → 75 − 10 = 65 × $0.015 = **$0.98**. Seeded on the 1st: $1.85. On the 25th:
(24 × 6.8 + 6 × 132.99) / 30 = 32.0 → 33 − 10 = 23 → $0.35. The seed day itself counts
at full size, because the meter takes each day's peak.

A second seed in the same calendar month: 513.5k + 513.5k = 1,027,000 Class A → 27,000 over
the free million → rounds up to 1M → **$4.50**. A third: 1,540,500 → 540,500 over → still
1M billable → **$4.50 for the month, not $4.50 more**. A fourth: 2.05M → 1.05M over → 2M →
$9.00.

Runner minutes, egress, purges, deletes: $0. Dante hands out 133 GB once.

## Checked and found free

| Item | Plan tier | Source, fetched 2026-08-26 |
|---|---|---|
| Custom domain on an R2 bucket (100 per bucket) | R2 free tier; Cloudflare Free zone | https://developers.cloudflare.com/r2/buckets/public-buckets/, https://developers.cloudflare.com/r2/platform/limits/ |
| Bandwidth through the zone | all plans | "Included in all plans. Cloudflare does not charge for bandwidth", https://developers.cloudflare.com/billing/understand/how-charges-accrue/ |
| R2 egress | free | pricing footnote 1 |
| Transform Rules | Free: 10 active rules, no regex | https://developers.cloudflare.com/rules/transform/#availability |
| Cache Rules | Free: 10 rules | https://developers.cloudflare.com/cache/how-to/cache-rules/#availability |
| Smart Tiered Cache | all plans | https://developers.cloudflare.com/cache/how-to/tiered-cache/ |
| Purge: by URL, hostname, tag, prefix, everything | Free: single-file 800 URLs/s, 100 per request; other purges 5 requests/min, bucket of 25 | https://developers.cloudflare.com/cache/how-to/purge-cache/ |
| `DeleteObject`, `AbortMultipartUpload` | free operations | pricing page |
| Unauthorized (401) requests | not charged | pricing page FAQ |
| GitHub Actions on a public repository | "free for ... public repositories that use standard GitHub-hosted runners"; larger runners always charged | https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions |
| healthchecks.io | Hobbyist, $0: 20 checks, 100 log entries per check | https://healthchecks.io/pricing/ |
| Domain | already owned | |

Not free, and not in the design: Infrequent Access (no free tier, $9/M Class A, 30-day
minimum); Cloudflare budget alerts ("available to Pay-as-you-go accounts only",
https://developers.cloudflare.com/billing/manage/budget-alerts/, whether a Free-plan account
with a card counts is *unverified* and belongs to monitoring.md); Cache Reserve (paid).

## Hidden and second-order costs

### The Free plan's terms on large files

https://www.cloudflare.com/service-specific-terms-application-services/ (fetched today),
Content Delivery Network section:

> "Unless you are an Enterprise customer, Cloudflare offers specific Paid Services (e.g.,
> the Developer Platform, Images, and Stream) that you must use in order to serve video and
> other large files via the CDN. Cloudflare reserves the right to disable or limit your
> access to or use of the CDN, or to limit your End Users' access to certain of your
> resources through the CDN, if you use or are suspected of using the CDN without such Paid
> Services to serve video or a disproportionate percentage of pictures, audio files, or
> other large files."

https://www.cloudflare.com/service-specific-terms-developer-platform/ lists R2 as part of
the Developer Platform and says nothing about content. The announcement of these terms
(https://blog.cloudflare.com/updated-tos, 2023-05-16) says: "customers can serve video and
other large files using the CDN so long as that content is hosted by a Cloudflare service
like Stream, Images, or R2."

What that means for a 133 GB bucket: the content is hosted by R2, which is the case the
blog names as allowed, and at 133 GB the account pays R2 $1.85 a month, so it is a paying
Developer Platform customer, which is what the terms sentence asks. The residual risk is
that "Paid Services" is read as requiring a paid *plan* rather than paid *usage*.
*Unverified*; a written answer from Cloudflare support would settle it. If enforced, the
remedy in the terms is "disable or limit ... the CDN" for the zone, not a bill; the cost
would be a Pro zone (price not verified here) or the mirror going dark until moved. The base
plan marks this "verified"; the terms say the opposite of what a Free zone would like and
the blog resolves it only for R2-hosted content. This is the one line item that could
change the bill by more than $10.

### Keeping tlnet's versioned containers

```sh
awk '$1 ~ /^-/ && $5 ~ /^systems\/texlive\/tlnet\// && $5 ~ /\.r[0-9]+\.tar\.xz$/ && $2 ~ /^[0-9]+$/ {n++; s+=$2} END {printf "%d objects %.2f GB\n", n, s/1e9}' $L
awk -v cut="2026/07/27 17:00:00" '$1 ~ /^-/ && $5 ~ /^systems\/texlive\/tlnet\// && $5 ~ /\.r[0-9]+\.tar\.xz$/ && ($3" "$4) >= cut {n++; s+=$2} END {printf "30-day churn %d files %.2f GB\n", n, s/1e9}' $L
```

+14,872 objects, +6.62 GB: usage 139.61 → 140 − 10 = 130 GB-month → $1.95, **+$0.10 a
month**, plus 457 more PutObjects a month (every revision bump lands twice, as
`foo.r123.tar.xz` and as `foo.tar.xz`). Nothing requests the versioned name. The only reason
to keep them is byte-for-byte fidelity with dante for `rsync` clients, which R2 cannot serve
anyway. Cheap, and pointless: the answer is no.

### Storing both `systems/win32` and `systems/windows`

19,174 objects, 23.64 GB, and 463 files / 0.44 GB of churn a month uploaded twice. Dropping
`win32`: 132.99 − 23.64 = 109.35 → 110 − 10 = 100 GB-month → $1.50, **saving $0.35 a
month** and 463 PutObjects. The alias has existed since 2007 (link mtime in the listing) and
MiKTeX's own URLs use `systems/win32/miktex/`, so a redirect rule is the only way to drop it
without breaking those links. Same trade as the other aliases: $0.41 a month for all six
against a dashboard rule.

### Class B from 404s and HEADs

Both are Class B. `HeadObject` is on the Class B list by name; a `GetObject` for a missing
key is a `GetObject`, and the only exemption on the pricing page is 401. Today's mirror
answers a directory URL and any wrong path with a 404 from R2, `cf-cache-status: DYNAMIC`
(observed on `https://ctan.ijosh.com/nonexistent-key-404` today), so each costs one Class B
until a cache rule caches 404s. 18,417 directories exist. This is the crawler line in the
table above: real at $0.36 per million, never large.

### `aws s3 ls` reconcile at scale

The one Class A item proportional to bucket size × frequency: ⌈objects / 1000⌉ per listing.

| Objects | Pages | Daily | Hourly once | Hourly twice |
|---:|---:|---:|---:|---:|
| 496k (133 GB) | 497 | 14,910 | 357,840 | 715,680 |
| 746k (200 GB) | 747 | 22,410 | 537,840 | 1,075,680 → $4.50 |
| 1,119k (300 GB) | 1,120 | 33,600 | 806,400 | 1,612,800 → $4.50 |

A listing costs the same whether it finds drift or not, so listing once a day is the whole
saving; the state-file design in sync-with-dante.md is what makes hourly sync free at any
size in this table.

### Failed and aborted uploads

- A `PutObject` that fails server-side (5xx, timeout after the request arrived): not
  documented as free; assume one Class A per attempt. The CLI's retries multiply it. At
  3 attempts on 1% of a seed that is +15k Class A, $0.
- Multipart: every `UploadPart` that completed is a Class A already spent;
  `AbortMultipartUpload` is free; "Buckets have a default lifecycle rule to expire
  multipart uploads seven days after initiation"
  (https://developers.cloudflare.com/r2/buckets/object-lifecycles/). Whether the parts of
  an incomplete upload bill storage during those seven days is *unverified* (the storage
  dataset tracks `uploadCount`, "pending multipart uploads", so they are at least
  counted). Worst case: five 6.8 GB uploads abandoned for 7 days = 8 GB-month, $0.12.
- Writes to one key faster than once a second return 429 (limits page); the CLI retries;
  whether the refused write is charged is *unverified*. The state-file key is written once
  per batch, minutes apart.

### Two more

- The 100 MB "Max upload size" of a Free zone applies to requests proxied through the zone.
  Uploads go to `<account>.r2.cloudflarestorage.com`, which is not the zone; today's mirror
  uploads a 145 MB container daily. Not a cost; noted because it looks like one.
- Late and dropped scheduled runs cost nothing: a dropped slot is one listing fewer and the
  same PutObjects an hour later. GitHub's 60-day schedule disable costs nothing either; a
  stale mirror is delisted, not billed.

## Cost sensitivity

Base: stored set, hourly sync with state file and daily reconcile, edge cache off (the
adopted `CACHE: off`), traffic under 27 installs a day, no seed this month: **$1.85**.

| Assumption | Base value | Pessimistic value | Bill at pessimistic | Moves the bill by |
|---|---|---|---|---:|
| Traffic with the cache off (the default) | 27 installs/day or fewer | 1,000 `scheme-full` installs/day | $127.13 | **unbounded, $0.36/M GETs** |
| Terms enforcement | not enforced | CDN disabled for the zone; Pro plan to restore | Pro plan price (not verified) or mirror dark | ~$20+ |
| Bucket listing frequency | once a day | twice an hour at ≥200 GB | $2.85 + $4.50 = $7.35 | +$4.50 step |
| Storage set | 133 GB | 300 GB | $4.35 | +$0.015/GB |
| Re-seeds in one month | 0 | 2 or 3 | $1.85 + $4.50 | +$4.50 step |
| Cache miss factor, once the cache is on | 2 per object per day | 10 | 149M Class B → $50.04 + $1.85 | +$0.36 per 1M |
| Uncacheable objects | 7, few downloads | 10,000 ISO downloads a month at 16 ranges | 160k Class B | $0.06 |
| Crawlers | none | 25 full crawls a month, 404s uncached | 12.9M Class B → $1.08 | +$1.08 |
| Alias directories | stored | stored (or dropped) | $1.85 (or $1.44) | −$0.41 |
| Versioned tlnet containers | dropped | kept | $1.95 | +$0.10 |
| GB meter base | 10^9 | 2^30 | $1.71 | −$0.15 |

The single assumption that moves the bill most is traffic: with the cache off, every
million GETs past 10M is $0.36 with no cap, and only caching.md's switch changes that (with
it on, no traffic this mirror can draw reaches $1). Second is anything that rounds into a Class A million: listing
twice an hour at 200 GB, or two seeds in a month, each a flat $4.50. Storage is linear,
slow and visible in every `report`.

## Budget

Adopt a hard ceiling of **$5.00 a month**, which is the storage line at the proposed 200 GB
bucket ceiling ($2.85) plus one Class A or Class B rounding accident ($4.50 would breach
it, $0.36 would not). In practice the target is the storage line alone.

Four guards, two kinds. The pre-upload guards fail the run, because a run is the only
thing that can add storage. The month-to-date guards never fail the run: a breach there is
traffic or a listing that already happened, and stopping the mirror does not undo it, so
`report` signals healthchecks (monitoring.md) and the run completes.

| Guard | Threshold | Arithmetic | What it catches |
|---|---|---|---|
| Storage, from the daily reconcile listing or `r2StorageAdaptiveGroups` (fails the run) | > 200 GB | 200 − 10 = 190 × $0.015 = $2.85; 67 GB of headroom over today for upstream growth (62 GB changed last year but the tree grew far less; the ISO and pkg replace, not add) | a symlink loop, a new alias directory, a second ISO copy |
| Objects, from the upstream listing before any upload (`verify`'s inflation guard; fails the run) | > 600,000 | 100k over today; a listing that size is still 600 pages a day | the same, before it costs anything |
| Class A month-to-date, `r2OperationsAdaptiveGroups` (signals, never fails) | > 800,000 | 80% of the free million; an hourly listing-twice design or a second seed crosses it days before the bill does | a `publish` that started listing per hour |
| Class B month-to-date (signals, never fails) | > 8,000,000 | 80% of the free 10M; at 28 installs/day uncached it trips on day 24, at 100/day on day 7 | the cache rule silently gone |

The guards are read on the next run, and a scheduled run starts 15 to 45 minutes after
its slot and can be dropped, so the worst gap between a breach and a signal is about two
hours of whatever caused it: at 1,000 uncached installs a day, two hours is 993k Class B,
$0.36. A breach that nobody acts on is unbounded only on the Class B line; storage
cannot grow past its guard because the guard stops the run that would grow it, and Class A
is bounded by the sync design at any bucket size in the tables above. How each number is read, and the GraphQL queries, are in monitoring.md; what to do
when one trips is in errors-and-issues.md.

## Open questions

- Is R2's storage meter base-10 or base-2? ($1.85 or $1.71; the first month's
  `payloadSize` answers it.)
- Does a $1.85-a-month R2 customer on a Free zone satisfy "Paid Services" in the CDN terms?
  Only Cloudflare can say; ask before registering as an official mirror, since delisting
  after registration costs CTAN's goodwill as well as the zone.
- How many requests does one thirteenth of US redirects produce? Only measurable after a
  month on the rotation.
- Are refused (429) or failed (5xx) writes charged, and do pending multipart parts bill
  storage for their 7 days? Both under $0.15 in the worst case; unverified.
- The double-change rate within a month (how many PutObjects the 16,490 monthly changes
  really are on an hourly sync).

## Sources

All fetched 2026-08-26.

- https://developers.cloudflare.com/r2/pricing/
- https://developers.cloudflare.com/r2/platform/limits/
- https://developers.cloudflare.com/r2/objects/upload-objects/
- https://developers.cloudflare.com/r2/buckets/object-lifecycles/
- https://developers.cloudflare.com/r2/buckets/public-buckets/
- https://developers.cloudflare.com/r2/platform/metrics-analytics/
- https://developers.cloudflare.com/billing/understand/how-charges-accrue/
- https://developers.cloudflare.com/billing/manage/budget-alerts/
- https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- https://developers.cloudflare.com/cache/how-to/cache-rules/
- https://developers.cloudflare.com/cache/how-to/cache-rules/settings/
- https://developers.cloudflare.com/cache/how-to/tiered-cache/
- https://developers.cloudflare.com/cache/how-to/purge-cache/
- https://developers.cloudflare.com/cache/how-to/configure-cache-status-code/
- https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/
- https://developers.cloudflare.com/rules/transform/
- https://www.cloudflare.com/service-specific-terms-application-services/
- https://www.cloudflare.com/service-specific-terms-developer-platform/
- https://blog.cloudflare.com/updated-tos
- https://docs.aws.amazon.com/cli/latest/topic/s3-config.html
- https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- https://healthchecks.io/pricing/
- https://ctan.org/mirrors/register
- https://ctan.org/mirrors/mirmon
- https://mirror.ctan.org/CTAN.sites and https://mirror.ctan.org/timestamp
- https://ftp.fau.de/news/
- https://en.wikipedia.org/wiki/CTAN
- Read-only HEAD/GET of https://ctan.ijosh.com/ (cache status, `Content-Length`, `Accept-Ranges`, a 206)
- `staging/tlpkg/texlive.tlpdb` and `staging/tlpkg/TeXLive/TLConfig.pm` (downloader arguments)
