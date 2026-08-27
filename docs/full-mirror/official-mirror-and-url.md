# Official mirror status and the public URL

What CTAN asks of an official mirror, how a mirror served from R2 measures up, and whether
`https://ctan.ijosh.com/` with the tree at the host root is the right public shape. Every
claim below was checked on 2026-08-26/27 against the page or listing named next to it. The
listings are the two `rsync --list-only` runs in the scratchpad (`ctan-list-deref.txt`,
`ctan-list-nolink.txt`); the scratchpad is `SCRATCH` in the commands. Email addresses are
written `name (at) host` throughout.

Decisions, in one breath: register `HTTPS://ctan.ijosh.com` as a United States mirror with
no rsync, store every alias as a copy (132.99 GB), send directory URLs to `ctan.org`'s own
browser with one Redirect Rule, keep our landing page at `/` from a reserved `.site/` prefix
and store CTAN's `index.html` as the ordinary file it is, and leave every Cloudflare
security setting at its default. Details and the numbers follow.

## 1. The rules

### Sources fetched

```
curl -sL -A 'Mozilla/5.0' https://ctan.org/mirrors/register   -> 200, 32,430 bytes
curl -sL -A 'Mozilla/5.0' https://ctan.org/mirrors            -> 200, 47,506 bytes
curl -sL -A 'Mozilla/5.0' https://ctan.org/mirrors/mirmon     -> 200, 80,909 bytes
curl -sL https://mirror.ctan.org/CTAN.sites                    -> 200, 18,398 bytes (served by mirrors.ibiblio.org)
curl -sL https://mirror.ctan.org/README.mirrors                -> same 18,398 bytes; README.mirrors is CTAN.sites
curl -sL https://ctan.org/help/mirror                          -> 404 (no such page)
curl -sL https://www.tug.org/mirroring.html                    -> 404
curl -sL https://www.tug.org/texlive/acquire-mirror.html       -> 200 (TeX Live's own mirroring page)
curl -sL https://www.mankier.com/1/mirmon                      -> 200 (mirmon manual)
```

There is no CTAN mirror-maintainer README beyond the register page; `CTAN.sites` is the
machine-readable mirror list and `README.mirrors` at the tree root is a symlink to it (same
size and mtime in the listing, `18398 2026/08/25 16:21:07`).

### Requirements, quoted from https://ctan.org/mirrors/register

1. "You need permanent Internet connectivity and at least 60 GB of hard drive space free
   (100 GB leaves room to grow)."
2. "Giving visitors access to files means running a HTTPS demon." And: "In April 2021 we
   started to support HTTPS for the redirector. From this time on the automatic redirection
   leads to a mirror server with the HTTPS protocol. The other protocols are left for manual
   selection only."
3. "You must mirror from the primary CTAN node." `rsync://rsync.dante.ctan.org/CTAN`,
   Germany.
4. "An official CTAN mirror must synchronize with the primary CTAN node at least once per
   hour."
5. "Choose a random minute when setting up your mirror and retain that minute permanently.
   Distributing mirrors across different minutes prevents them from contacting the primary
   CTAN node simultaneously and reduces avoidable load peaks."
6. The recipe is `rsync -av --delete rsync://rsync.dante.ctan.org/CTAN /var/www/tex-archive`;
   the whole tree, with deletions.
7. On the web server: "You want to keep files in the archive that happen to be named
   `index.html` from being served by your Apache as the index of that directory's page."
   The example config carries `Options ... +FollowSymLinks ... +Indexes` and
   `DirectoryIndex disabled`.
8. The form asks for Name ("usually: your host name"), contact person, contact email,
   country, region, city, "Mirror from rsync.dante.ctan.org", "Address for HTTPS (such as
   'joshua.smcvt.edu/tex-archive')", optional HTTP, FTP and Rsync addresses, and Notes.
9. "You will be enrolled in the low-traffic mailing list for our mirror maintainers."
10. "we monitor mirrors to check that they are up to date. If your mirror falls behind then
    mirrors.ctan.org will not redirect to it, and we shall have to remove it from the
    official list."

The mirrors page (https://ctan.org/mirrors) adds "Please send updates to this list to
ctan (at) ctan.org" and "the multiplexor always redirects to a https mirror."

The status page (https://ctan.org/mirrors/mirmon, 2026-08-27 00:03 UTC) says "132 sites in
40 regions", "0 bad -- 13 older than 2.2 days -- 2 unreachable for more than 5 hours",
"last probes : 125 were ok, 7 had no time". The histogram's bands are "0 <= age <= 28 hours"
(fresh) and "28h < age <= 52h" (oldish); older is "old", unreachable is "bad". The table
columns are "CTAN site -- home", "type" (every row says `https`), "mirror age, daily stats",
"last probe, probe stats", "last stat". One row, verbatim from the HTML with the home link
elided:

```
<A HREF="https://us.mirrors.cicku.me/ctan/">us.mirrors.cicku.me</A> ... <TD>https</TD>
<TD ALIGN=RIGHT><B><FONT COLOR="RED">12.5 days</FONT></B> ... <TD ALIGN=RIGHT><B>renewed</B> ... <TD>ok</TD>
```

From the mirmon manual (https://www.mankier.com/1/mirmon) the probe is "a (user specified)
program on a pipe"; "The probe should return something that looks like `1043625600 ...` that
is, a line of text starting with a timestamp. The exit status of the probe is ignored."
Default `min_sync 1d`, `max_sync 2d`; sites past `max_sync` are "old". The example probe is
`wget -q -O - -T %TIMEOUT% -t 1 %URL%TIME.txt`, a GET. "If the url part of a line in the
mirror_list doesn't end in a slash ('/'), mirmon adds a slash."

### Compliance matrix

| Requirement | Our design | Status | Evidence |
|---|---|---|---|
| HTTPS access to the whole tree at one base address | `https://ctan.ijosh.com/`, tree at the root | meets | `curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512` gives `HTTP/2 200`; `curl -sI http://ctan.ijosh.com/...` gives `301` to https |
| Mirror from `rsync.dante.ctan.org` | `SOURCE` is dante already; the full mirror lists and pulls from it only | meets | `Taskfile.yml` `SOURCE` |
| Sync at least hourly | hourly cron; the run starts 15 to 45 minutes after its slot, and a slot can be dropped | meets with caveat | GitHub: scheduled workflows "can be delayed during periods of high loads of GitHub Actions workflow runs" and may be dropped; runs stop after 60 days without a commit (https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows). Observed here: the `30 3 * * *` run of 2026-08-26 was created at 04:09:16 UTC, 39 minutes late. Worst age mirmon can see after one dropped slot is about 3 h 46 min, 7 h 46 min as displayed (arithmetic in section 7), inside the 28-hour fresh band |
| Fixed random minute, kept forever | `cron: 'M * * * *'` with M chosen once; dante is contacted 15 to 45 minutes after M | meets with caveat | workflow file; a fork must pick its own M. The rule exists to spread mirrors' contacts with dante; GitHub's drift spreads ours further, it never bunches them |
| Whole tree with deletions | list-diff with deletions; symlinks become objects; tlnet's 14,874 versioned container links are excluded | meets with caveat | `grep -c '^l.* systems/texlive/tlnet/' SCRATCH/ctan-list-nolink.txt` = 14874. `FILES.last07days` lists those versioned names (155 of its 8,229 lines, `grep -cE '\.r[0-9]+\.tar\.xz$'`), so a reader of `FILES.byname` gets a 404 for them on our mirror. See Open questions |
| 60 GB free, permanent connectivity | R2, no disk | meets | R2 has no bucket size limit (limits.md) |
| `index.html` in the tree served as a file, never as a directory's index | R2 serves keys only | meets | `curl -sI https://ctan.ijosh.com/index.html` gives 200 text/html, and `/` is a separate rewrite |
| Directory pages (`+Indexes`) | not offered by R2; one Redirect Rule sends `/dir/` to `ctan.org/tex-archive/dir/` | meets with caveat | not a stated requirement; section 2 |
| `/timestamp` fresh and reachable over HTTPS with GET | copied every run, never cached | meets, once mirrored | today `curl -sI https://ctan.ijosh.com/timestamp` gives 404 because only tlnet is mirrored |
| Register; contact person and email; country, region, city | manual, once | meets | section 9 |
| Join the maintainers' list | automatic on registration | meets | `ctan-mirrors-announce (at) ctan.org`, section 9 |
| Optional HTTP, FTP, rsync | HTTP redirects to HTTPS; no FTP; no rsync | meets | all three are marked optional on the form; 86 of 134 listed hosts offer no rsync |
| Stay listed; do not fall behind | a failed run is the alert; healthchecks after a missed run | meets | monitoring.md |

Nothing on the register page names a file type, a header, a server, a directory listing, a
robots policy or a bandwidth. The rules are the whole tree, hourly, from dante, over HTTPS,
with a probe-able `timestamp`, and a human to email.

## 2. Directory indexes

### Does CTAN require them?

No sentence on the register page or the mirrors page requires a directory listing. The
Apache example enables `+Indexes`, and every Apache or nginx mirror sampled serves one, so
users are used to it. What CTAN's own site does:

```
grep -oE 'href="[^"]*"' SCRATCH/ctan.org_pkg_hyperref.html | grep -iE 'mirror|tex-archive|\.zip'
  href="/tex-archive/macros/latex/contrib/hyperref"                            (ctan.org's own browser)
  href="https://mirrors.ctan.org/install/macros/latex/contrib/hyperref.tds.zip"
  href="https://mirrors.ctan.org/macros/latex/contrib/hyperref.zip"
  href="https://mirrors.ctan.org/macros/latex/contrib/hyperref/doc/hyperref-doc.html"
  href="https://mirrors.ctan.org/macros/latex/contrib/hyperref/doc/hyperref-doc.pdf"
  href="https://mirrors.ctan.org/macros/latex/contrib/hyperref/doc/paper.pdf"
  href="https://mirrors.ctan.org/macros/latex/contrib/hyperref/README.md"
```

The `/tex-archive/...` browser page links only the two zips through `mirrors.ctan.org`;
its per-file links stay on `ctan.org`. So every link CTAN's site sends to a mirror is a
file. `hyperref.zip` and `hyperref.tds.zip` are real files upstream, not symlinks
(`grep -E ' (macros/latex/contrib/hyperref\.zip|install/macros/latex/contrib/hyperref\.tds\.zip)' SCRATCH/ctan-list-nolink.txt`
shows `-rw-rw-r--` for both).

The redirector itself does not care what it forwards:

```
curl -sI https://mirror.ctan.org/macros/latex/contrib/hyperref/
  HTTP/1.1 307 Temporary Redirect
  Location: https://mirrors.ibiblio.org/pub/mirrors/CTAN/macros/latex/contrib/hyperref/
curl -sI https://mirror.ctan.org/
  Location: https://latex.us/
curl -sI https://mirror.ctan.org/documentation/
  Location: https://mirror.math.princeton.edu/pub/CTAN/documentation/
```

So a person who types or bookmarks a directory URL through `mirror.ctan.org` will, one
time in the number of US mirrors, land on ours and get R2's 404 page. TUG's install links
(https://www.tug.org/texlive/acquire-netinstall.html, "these links go to mirrors", "The
above links use the generic mirror.ctan.org url") are files under `systems/texlive/tlnet/`.
`install-tl`, `tlmgr` and MiKTeX's setup fetch files only.

What R2 does with a directory URL today:

```
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/      -> HTTP/2 404
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg  -> HTTP/2 404
curl -s  https://ctan.ijosh.com/systems/texlive/tlnet/nope   -> Cloudflare's R2 "Error 404 / Object not found" HTML page
```

### The numbers an index feature would have to carry

```
grep -c '^d' SCRATCH/ctan-list-nolink.txt      -> 18417   real directories
grep -c '^d' SCRATCH/ctan-list-deref.txt       -> 27262   directories once aliases are materialised (what the bucket has)
grep -cE ' index\.html$' SCRATCH/ctan-list-deref.txt -> 111  files named index.html (the root one plus 110 inside packages)
# directories touched by the 30-day churn (parents of changed files; then with all ancestors)
awk -v cut=$(date -u -v-30d +%Y/%m/%d) '$1 ~ /^-/ && $3 >= cut {p=$5; sub(/\/[^\/]*$/,"",p); d[p]=1} END{c=0; for(k in d)c++; print c}' SCRATCH/ctan-list-deref.txt
  -> 756 (875 with ancestors); 7 days, 9,163 files in 243 directories
# entries per directory
awk '$1 ~ /^[-d]/ && $5 != "." {p=$5; sub(/\/[^\/]*$/,"",p); n[p]++} END{...}' SCRATCH/ctan-list-deref.txt
  -> 27223 dirs, 19.8 average, max 29744 (systems/texlive/tlnet/archive)
```

### Options

| | Option | Cost | What it needs | What breaks | Verdict |
|---|---|---|---|---|---|
| a | None; directory URLs 404 | $0 | nothing | one redirector hit in about 15 lands on a 404 page; CTAN's own `index.html`, which links `biblio/`, `fonts/` and the rest, is a page of dead links | acceptable, but (d) is free |
| b | One generated page per directory, uploaded as an object | $0. 27,262 PutObject once; about 900 a month (dirs touched, with ancestors); about 2 KB a page, about 55 MB in all, `tlnet/archive` about 2.5 MB | `awk` over the listing already in `RUN/`, so no new tool. The key cannot be `<dir>/index.html`, since 111 real files carry that name, which is exactly why CTAN's Apache recipe says `DirectoryIndex disabled`. Use the key `<dir>/` itself (S3 allows keys ending in `/`; whether R2's custom-domain path maps `/dir/` to that key is unverified, see Open questions) or `<dir>/.index.html` plus one Transform Rule rewriting paths that end in `/` (a dynamic rewrite with `concat`, allowed on Free, no regex). The pages are not upstream files, so the state file and the reconcile must know the pattern and never delete them, `stale`'s `comm` must filter them out, and, if the cache is switched on, caching.md must exclude or purge them | a second class of object to reason about in `publish`, `stale`, reconcile and `smoke`; staleness bounded by one run | not adopted; kept in reserve |
| c | A Worker | Free plan is 100,000 requests a day, then "Error 1027" for everyone (https://developers.cloudflare.com/workers/platform/limits/). Routes are host/path prefixes, so a listing Worker fronts every request; one `scheme-full` install is 11,919 GETs (cost-estimates.md), so about eight installs a day exhaust the quota and the mirror is down until midnight UTC | `wrangler`, a second deploy path, a Worker secret with R2 access, and each listing is an R2 `list()` call, Class A | the tools list (`rsync aws gpg shasum xz curl task` only), "workflows install tools and run one task", zero cost at any traffic | reject |
| d | Redirect Rule; directory URL to `https://ctan.org/tex-archive/<path>` | $0; 1 of 10 Single Redirects on Free ("Number of rules 10, Wildcard support Yes, Regex support No", https://developers.cloudflare.com/rules/url-forwarding/) | expression `http.request.uri.path ne "/" and ends_with(http.request.uri.path, "/")`, target `concat("https://ctan.org/tex-archive", http.request.uri.path)`, status 302. Single Redirects run before URL Rewrite Rules (same page, "Execution order"), so the `ne "/"` clause is what keeps the landing-page rewrite alive | a directory URL without the trailing slash still 404s (Apache would 301 it); users leave our host for a page that is better than an Apache listing (package metadata, per-file links). Nothing on CTAN's pages forbids it; ask in the registration Notes | adopted |
| e | An R2-native listing | | none exists. https://developers.cloudflare.com/r2/buckets/public-buckets/ describes custom domains and `r2.dev` only; `grep -iE 'listing|index' SCRATCH/r2_buckets_public-buckets.md` finds nothing about directories; `r2.dev` is "rate-limited and should only be used for development purposes" | | not available |

Decision, adopted across the set: (d). It is one rule, costs nothing, and turns the one human-facing
failure of an object store into the best directory page CTAN has. Every CTAN path stays
valid on our host for files; only `.../` moves. Its JSON lives in
`cloudflare/redirect-rules.json` beside the cache and transform rules (caching.md), in the
`http_request_dynamic_redirect` phase, so a fork gets it. If a maintainer later wants
listings on our own host, (b) is the path and the `<dir>/`-key test in Open questions is
the first thing to run.

## 3. The `index.html` collision, `README*` and `robots.txt`

CTAN's root has an `index.html`:

```
awk '$1 ~ /^-/ && $5 !~ /\//' SCRATCH/ctan-list-deref.txt
  CTAN.sites 18398   FILES.byname 26740646   FILES.byname.gz 2811339   FILES.last07days 584326
  README.mirrors 18398 (link to CTAN.sites)   README.structure 3547   README.uploads 324
  index.html 10366 (2020/03/31)   tds.zip 303233 (link to info/tds.zip)   timestamp 186
awk '$5 ~ /^\./ && $5 != "."' SCRATCH/ctan-list-nolink.txt   -> nothing; CTAN has no dot-prefixed root entry
```

What it is: `curl -sL https://mirror.ctan.org/index.html` returns "The CTAN root directory"
(`<title>The CTAN archive</title>`), a description of the top-level directories with each
name a relative link (`biblio`, `digests`, `dviware`, `fonts`, ...), and a disabled
meta-refresh (`http-equiv="r-e-f-r-e-s-h"`) to `https://ctan.org/tex-archive`. What other
mirrors show at their root:

```
for h in https://ctan.math.illinois.edu/ https://mirror.clarkson.edu/ctan/ https://mirrors.mit.edu/CTAN/ \
         https://ctan.mirror.rafal.ca/ https://latex.us/ https://mirrors.ibiblio.org/pub/mirrors/CTAN/ https://ctan.net/; do
  curl -s -A 'Mozilla/5.0' -o /tmp/x.html -w "$h %{http_code} %{content_type} %{size_download}\n" "$h"; done
  illinois 200 text/html 10366   clarkson 200 10366   mit 200 10366   rafal 200 10366   ibiblio 200 10366
  latex.us 200 12267   ctan.net 200 12268        (those two add their own footer)
```

Seven of seven serve CTAN's page, five of them byte-identical. Its links are directory URLs.
On R2 without option (d) those links are 404s; with (d) they land on `ctan.org`'s browser.

Today's landing page: `site/index.html` is uploaded to the key `index.html` with
`Cache-Control: no-cache`, and a Transform Rule rewrites `/` to `/index.html`
(`curl -sI https://ctan.ijosh.com/` shows `200 text/html`, `cache-control: no-cache`,
`last-modified: Wed, 26 Aug 2026 18:07:42 GMT`). A full mirror uploads CTAN's `index.html`
to the same key every run, so the two pages collide at one key.

| Option | Effect | Cost |
|---|---|---|
| A. Replace ours with CTAN's | matches every other mirror; the `page` task and `site/` go away; the "how to use" text moves to the README; with (d) the page's links work | 0, minus one task |
| B. Ours at `/`, CTAN's at `/index.html` (adopted) | key `.site/index.html` holds our page; the existing Transform Rule's target becomes `/.site/index.html`; CTAN's `index.html` is stored at its own path like any file, byte for byte; `.site/` joins `.state/` as a reserved prefix the reconcile and `stale` ignore (CTAN has no dot-prefixed root entries, verified above) | 0; the `page` task changes one key; one line in the reconcile filter |
| C. Ours at `/about.html` | a key inside CTAN's namespace that upstream could one day create; needs the same reconcile exception as B with none of B's separation | worse than B |

Decision, adopted across the set: B. The people who reach `/` on a mirror are the ones the redirector sends
there (`mirror.ctan.org/` goes to `https://latex.us/`) or the curious; a page that says what
this host is, how to point `tlmgr` at it, and where to browse (`ctan.org/tex-archive`)
serves them better than a 2020 directory description. Anyone who wants CTAN's page has it
at `/index.html`, exactly as on every other mirror.

`README.mirrors`, `README.structure`, `README.uploads` are ordinary objects at the root,
copied as-is (`README.mirrors` becomes a copy of `CTAN.sites`, 18 KB).

`robots.txt`: CTAN's tree has none (`grep -E ' robots\.txt$' SCRATCH/ctan-list-deref.txt` is
empty); mirrors serve their own (`curl -sIL https://mirror.ctan.org/robots.txt` lands on
`latex.us/robots.txt`, `200 text/plain`). On R2 the path is a 404, which crawlers read as
"crawl everything". With no directory pages there is nothing to discover from except the two
index pages, so the exposure is small; a crawler that does find `FILES.byname` and walks it
costs Class B operations, free to 10M a month. If a `robots.txt` is wanted, it lives at
`.site/robots.txt` with a second Transform Rule (`/robots.txt` to `/.site/robots.txt`); note
that "by default Cloudflare caches a website's robots.txt"
(https://developers.cloudflare.com/cache/concepts/default-cache-behavior/), so if the cache
is switched on it needs `no-cache` like the page or a purge. Left out of the first cut; see Open questions.

## 4. The storage set and the aliases

The `--list-only` output in the scratchpad shows symlinks as `l` entries but not their
targets (`grep -c -- ' -> ' SCRATCH/ctan-list-nolink.txt` is 0), so targets below are
inferred by matching sizes and mtimes, then confirmed by count and bytes.

```
grep -c '^l' SCRATCH/ctan-list-nolink.txt                              -> 24788 symlinks
grep -c '^l.* systems/texlive/tlnet/' SCRATCH/ctan-list-nolink.txt     -> 14874 of them tlnet's versioned containers (excluded)
# the other 9,914; which resolve to files?
grep '^l' nolink | awk '{$1=$2=$3=$4=""; sub(/^ +/,""); print}' | grep -v '^systems/texlive/tlnet/' | sort > linkpaths.txt
awk '$1 ~ /^-/ {print $5"\t"$2}' deref | sort > derefsz.txt
join -t $'\t' linkpaths.txt derefsz.txt | awk -F'\t' '{n++; s+=$2} END{print n, s/1e9}'   -> 9737 file aliases, 29.26 GB
join -v1 ... > dirlinks.txt; wc -l                                                          -> 177 directory aliases
# files that exist only because of a directory alias
awk 'NR==FNR{A[$1]=1;next} $1 ~ /^-/ {p=$5; while(sub(/\/[^\/]*$/,"",p)) if(p in A){n++;s+=$2;break}} END{print n, s/1e9}' dirlinks.txt deref
                                                                                             -> 134013 files, 34.75 GB
# depth of the 177 directory aliases; 5 at the root, 37 at depth 2, 58 at 3, 55 at 4, 18 at 5, 4 deeper
```

### The root and depth-2 aliases, who uses them, and the decision

| Alias | Target (inferred) | Files | GB | Who uses the alias path | Decision |
|---|---|---:|---:|---|---|
| `documentation` | `info` | 28,888 | 2.21 | `ctan.org/tex-archive/documentation` exists and is indexed by search engines; the redirector forwards it (`mirror.ctan.org/documentation/` to princeton `.../documentation/`) | copy |
| `bibliography` | `biblio` | 2,593 | 0.78 | same pattern; redirector forwards to clarkson `.../bibliography/` | copy |
| `languages` | `language` | 9,718 | 0.45 | same | copy |
| `digests` | `info/digests` | 1,776 | 0.07 | same | copy |
| `tds`, `tds.zip` | `info/tds`, `info/tds.zip` | 15 | 0.0003 | `tds.zip` appeared 2022-09-22; `mirror.ctan.org/tds.zip` forwards | copy |
| `README.mirrors` | `CTAN.sites` | 1 | 0.00002 | historic name | copy |
| `macros/latex2e` | `macros/latex` | 43,683 | 5.74 | `ctan.org/tex-archive/macros/latex2e/contrib/<pkg>` pages exist for many packages (search hits classicthesis, texmate, changes), so the URLs circulate; redirector forwards | copy |
| `systems/windows` | `systems/win32` | 19,174 | 23.64 | the alias is `windows` (symlink dated 2007-03-02); `win32` is the real directory (`drwxrwxr-x ... systems/win32`). MiKTeX's own download page links `.../ctan/systems/win32/miktex/setup/...` (`grep -o "ctan/systems/[^'\"]*" SCRATCH/miktex.org_download.html`), and `ctan.org/tex-archive/systems/win32/miktex` is the canonical browse page. `ctan.org/tex-archive/systems/windows` also exists; the redirector forwards it | copy; this is the one worth a redirect if $0.35/month ever matters |
| `fonts/metrics` | `fonts/psfonts` | 9,672 | 0.07 | historic | copy |
| 167 deeper directory aliases | various | 18,495 | 1.79 | package-level conveniences (`fonts/cheq`, `support/xindy`, ...) | copy |
| 9,734 small file aliases | various | 9,734 | 8.83 | package zips and docs (`fonts/linearA.zip`, `info/epslatex.pdf`, ...) | copy |
| `systems/mac/mactex/MacTeX.pkg` | `mactex-20260324.pkg` | 1 | 6.87 | MacTeX's documented download URL | copy (multipart) |
| `systems/texlive/Images/texlive.iso`, `texlive2026.iso` | `texlive2026-20260301.iso` | 2 | 13.57 | TUG's documented ISO URLs; the names change every March | copy (multipart) |

Inference check for the big ones (alias file counts exceed the target's because the target
contains further symlinks that `-L` also resolves):

```
documentation [28888 files 2.207 GB]   vs info           [27215 files 1.910 GB]
macros/latex2e [43683 files 5.743 GB]  vs macros/latex   [41320 files 5.644 GB]
systems/windows [19174 files 23.644 GB] vs systems/win32 [10224 files 17.547 GB]
fonts/metrics [9672 files 0.065 GB]    vs fonts/psfonts  [9254 files 0.054 GB]   (first file Mathematica3.0.zip 264492 2005/05/20 in both)
tds/ChangeLog 9544 2004/06/23 10:24:52  ==  info/tds/ChangeLog
```

### Why copies and not redirects

Redirect capacity on Free (https://developers.cloudflare.com/rules/url-forwarding/): Single
Redirects "Number of rules 10" per zone, wildcards yes, regex no; Bulk Redirects "Bulk
Redirect Rules 15, Bulk Redirect Lists 5, URL redirects across lists 10,000" per account.
The 177 directory aliases plus 9,737 file aliases are 9,914 entries, inside the 10,000
with 86 to spare, and the list would have to track upstream symlink churn through the API
every run (one more listing, one more endpoint, and a list at 99% of its quota). Ten Single
Redirects could cover the five root aliases, `macros/latex2e`, `systems/windows`,
`fonts/metrics` and the three installers exactly, saving 55.2 GB ($0.83/month), but the two
ISO names change every March and `MacTeX.pkg`'s target every release; a rule nobody
remembers is a 404 nobody notices, and no check in the pipeline would see it unless `smoke`
reads every redirected alias back. Copies cost:

| Set | Objects | GB | Storage $/month at $0.015 after 10 GB free |
|---|---:|---:|---:|
| Fully dereferenced | 511,027 | 139.63 | 1.94 |
| **Stored set** (minus tlnet's 14,874 versioned containers and its six `update-tlmgr-r*` files, 0.01 GB) | 496,149 | 132.99 | 1.84 |
| minus the three installer aliases (20.43 GB) | 496,146 | 112.56 | 1.54 |
| minus the 177 directory aliases as well (34.75 GB) | 362,133 | 77.81 | 1.02 |
| minus every non-tlnet file alias as well (8.83 GB) | 352,399 | 68.98 | 0.88 |

The aliases are 48% of the bytes and $0.96 of the $1.84. That is the price of every CTAN
URL working verbatim with no state outside the bucket, and it is under a dollar. Store the
copies. The stored set is 496,149 objects, 132.99 GB: what `rsync -rL` lists, less tlnet's
14,874 versioned containers and the six `update-tlmgr-r*` files that `fetch` excludes
(`update-tlmgr-r79982.exe`, `.sh`, their `.sha512` and `.asc`; 0.0136 GB). tlcontrib's 261
versioned containers (`systems/texlive/tlcontrib/archive/*.r[0-9]*.tar.xz`, 0.46 GB) are
excluded as well and are not subtracted from the figures above. The directory alias that
costs 23.64 GB is `systems/windows`; `systems/win32` is the real directory.

## 5. The canonical URL

### What `CTAN.sites` looks like

```
grep -c 'URL:' CTAN.sites                                        -> 272 URL lines
grep 'URL:' CTAN.sites | awk '{print $2}' | sed 's#://.*##' | sort | uniq -c
   129 https   53 http   42 ftp   48 rsync
grep -E '^  [a-z0-9]' CTAN.sites | wc -l                          -> 134 hosts
grep 'URL: rsync' CTAN.sites | wc -l                              -> 48 hosts offer rsync
hosts whose only URL is https                                     -> 32
grep 'URL:' CTAN.sites | awk '{print $2}' | grep -E '^https?://' | sed -E 's#^https?://[^/]+##' | sort | uniq -c | sort -rn
    56 /ctan/   41 /CTAN/   27 /   14 /tex-archive/   13 /pub/CTAN/   4 /pub/tex-archive/   3 /pub/ctan/
     3 /mirrors/CTAN/   2 each /sites/ctan.org/ /pub/mirrors/CTAN/ /pub/mirror/tex-archive/ /mirror/CTAN/ /ctan/tex-archive/
     1 each /software/TeX/ /pub/tex/mirror/ftp.dante.de/pub/tex/ /pub/TeX/CTAN/ /pub/tex/ /pub/software/tex/ /pub/mirrors/latex/dante/
            /pub/mirrors/ctan/ /mirrors/ctan/ /mirror/ctan/ /latex/ /ftp/pub/mirror/ctan/
grep 'URL: https' CTAN.sites | awk '{print $2}' | grep -E '^https://[^/]+/$' | wc -l   -> 21 https URLs at the host root
grep -E '^  ctan\.' CTAN.sites | wc -l                             -> 24 hostnames begin with "ctan."
grep -c cicku CTAN.sites                                           -> 45 lines; 15 country-prefixed hosts of one operator
```

Twenty-one HTTPS URLs (27 counting their HTTP twins) sit at the host root.
`ctan.ijosh.com/` is the third most common shape and the most common among
hosts named `ctan.*`. Examples with the identical shape are `ctan.math.illinois.edu`,
`ctan.mirror.rafal.ca`, `ctan.net`, `ctan.tetaneutral.net` and `ctan.joethei.xyz`.

### What each system shows

- The redirector appends the request path to the registered base;
  `mirror.ctan.org/timestamp` goes to `https://mirror.clarkson.edu/ctan/timestamp` and
  `mirror.ctan.org/` to `https://latex.us/`. With `ctan.ijosh.com` registered at the root,
  every forwarded URL is `https://ctan.ijosh.com/<CTAN path>`.
- `ctan.org/mirrors` renders the registered address as the link
  (`href="https://ctan.math.illinois.edu/"`, `href="https://ctan.mirror.rafal.ca/"`) under
  the country, with city and region in parentheses.
- mirmon shows the hostname, `https`, age and probe history, linked to the registered URL.
- `tlmgr` prints the repository URL once per run. `tlmgr.pl` line 7482 (extracted from
  `archive/texlive.infra.tar.xz` on the mirror) is
  `info("$prg: package repository $location (" . ($verified ? "" : "not ") . "verified)\n")`,
  so users see `tlmgr: package repository https://ctan.ijosh.com/systems/texlive/tlnet (verified)`.
- `CTAN.sites` and `README.mirrors` carry the hostname into every mirror on earth.

### Shapes considered

| Shape | For | Against |
|---|---|---|
| `https://ctan.ijosh.com/` (root) | the URL already in users' `tlmgr` configs stays valid; same shape as 21 listed mirrors; every CTAN path verbatim; an R2 custom domain maps the bucket root to the host root with no rewrite | a personal hostname in a public list |
| `https://mirror.ijosh.com/ctan/` | the commonest shape (`/ctan/`, 56) | every key gains a `ctan/` prefix, or every request runs a rewrite rule; breaks `https://ctan.ijosh.com/systems/texlive/tlnet/` for existing users unless a redirect stays forever; two hostnames to keep |
| `https://ijosh.com/ctan/` | one domain | puts 133 GB of mirror traffic and a CTAN listing on the personal apex; same prefix problem |

The naming risk is real and shared by every mirror in the list; a user who sets
`tlmgr option repository https://ctan.ijosh.com/systems/texlive/tlnet/` is broken the day
the host stops, until they run `tlmgr option repository ctan`. `CTAN.sites` today lists
`ctan.joethei.xyz`, `ctan.kcvw.net` and `ctan.javinator9889.com`; personal hostnames are
normal here. The rotation protects users who never set a repository, which is most of them.
Keep `ctan.ijosh.com` at the root.

### rsync

Impossible. R2 speaks S3 and HTTP; there is no rsync daemon and Cloudflare offers none.
The form marks rsync optional, 86 of 134 listed hosts have none, and TUG's mirroring page
says "not all CTAN mirrors provide rsync access". The only people who want rsync from a
mirror are other mirrors and TeX Live administrators mirroring `tlnet`; TUG's page offers
`wget --mirror --no-parent https://somectan/somepath/systems/texlive/tlnet/` as the
alternative, and that works against R2 file by file, with the one cost noted in section 8
(`Last-Modified` is our upload time, so `wget -N` re-fetches after every run). Leave the
rsync field blank.

## 6. Regional expectations

The form asks for country, region and city; https://ctan.org/help/mirror-selection says
"CTAN has a mechanism which tries to guess the geo location of the requester. Based in this
location a server in the same region is selected" and "the mirror is chosen randomly in the
region". mirmon groups by the two-letter region; 40 regions, `us` among them.

Cloudflare's anycast serves each client from the nearest PoP (`cf-ray: ...-SEA` on requests
from Seattle), and a cached copy lives near the client; the R2 bucket itself sits in one
region. "When you create a new bucket, the data location is set to Automatic by default.
Currently, this option chooses a bucket location in the closest available region to the
create bucket request based on the location of the caller"
(https://developers.cloudflare.com/r2/reference/data-location/); hints are `wnam`, `enam`,
`weur`, `eeur`, `apac`, `oc`. The bucket's actual location is unverified from here (the
dashboard's bucket Settings shows it); it is almost certainly `wnam`.

Precedent for CDN-fronted official mirrors, from HEAD requests on `timestamp`:

```
curl -sI https://us.mirrors.cicku.me/ctan/timestamp
  server: cloudflare   cf-ray: a316ec62da22e17a-SEA   cf-cache-status: HIT   cache-control: public, max-age=3600
curl -sI https://gb.mirrors.cicku.me/ctan/timestamp
  server: cloudflare   cf-ray: ...-LHR   cf-cache-status: DYNAMIC
curl -sI https://mirror.kris.fail/CTAN/timestamp      (Japan; CTAN.sites lists /ctan/, so 404 here, but server: cloudflare)
curl -sI https://mirrors.aliyun.com/CTAN/timestamp
  server: Tengine   via: osm6.et93[...], ens-cache8.l2de3[...]   age: 4774   cache-control: max-age=7200   x-cache: MISS
curl -s -A 'Mozilla/5.0' https://us.mirrors.cicku.me/ctan/   -> 403, <title>Verification Required</title>  (a challenge page on directory listings)
```

So one operator (`cicku.me`) registered fifteen country-prefixed hostnames on one
Cloudflare zone, each under its own country, and CTAN lists all fifteen; a Japanese mirror
is on Cloudflare; Alibaba's mirror is behind its own CDN with a two-hour cache on
`timestamp`; and a Cloudflare challenge on directory pages did not get `cicku.me` delisted.
mirmon shows `us.mirrors.cicku.me` at "12.5 days" (red) with a green probe history, and
`gb.mirrors.cicku.me` at "2 hours"; a cached or stale origin behind a fresh CDN is exactly
what CTAN's monitor cannot tell apart, which is the argument for keeping `timestamp` out of the edge cache if the cache is ever
switched on.

Yihui Xie's post (https://yihui.org/en/2026/03/tinytex-ctan-mirror/) describes
`tlnet.yihui.org`: Cloudflare Pages, daily sync of `systems/texlive/tlnet/` only, files
over 25 MB on GitHub Releases behind redirects, generated directory index pages, TinyTeX's
default repository since March 2026. It says nothing about CTAN's view, it is not in
`CTAN.sites` (`grep -c yihui CTAN.sites` gives 0), and as a partial mirror it is not
eligible. No statement by CTAN about CDN-fronted mirrors was found; the fifteen `cicku.me`
rows are the evidence that they are accepted.

Decision: register one hostname under United States with the bucket's region as the
city (or the maintainer's city; the form has no "anycast" answer), and say in Notes that
the host is served from Cloudflare's network with the origin in R2. Do not register
per-country hostnames the way `cicku.me` did; it games the redirector, and one hostname is
what the users' configs carry.

## 7. What CTAN's monitor sees

What mirmon fetches is the registered base plus `timestamp`, over HTTPS (every `type` cell
says `https`), with a GET (the probe reads the body; the manual's example is `wget -q -O -`).
CTAN's `timestamp` is not the one-word file the stock probe expects:

```
curl -sL https://mirror.ctan.org/timestamp
  # This file is for administrative purposes only.
  #   The source CTAN of this site's material:
  irony.dante.de
  #   The year-month-day-hour-minute of this site's material:
  2026-08-26-09-02
```

186 bytes, rewritten at :02 every hour upstream (`timestamp 186 2026/08/26 17:02:01` in the
listing). CTAN's mirmon evidently runs its own probe that parses the last line (unverified;
the probe script is not published; the observable is that the status page reports ages
from this file). Every probe of ours therefore needs `GET https://ctan.ijosh.com/timestamp`
to answer 200 with the current upstream body.

Through Cloudflare:

- Not cached. The edge cache is off by default (`CACHE: off`, caching.md), and even with it
  on, `timestamp` has no extension and "Cloudflare only caches based on file extension"
  (default-cache-behavior page); observed `cf-cache-status: DYNAMIC` on our extensionless
  keys. If the cache is switched on, caching.md's rule must keep `/timestamp` out of its
  scope. A cached `timestamp` would add its TTL to the age CTAN sees; with hourly runs and
  the 28-hour band that is cosmetic at 1 hour and fatal at 2 days.
- Content type. R2 sends none for an extensionless key (`curl -sI .../install-tl` has no
  `content-type` line); Apache mirrors send `application/octet-stream`. A probe reads bytes;
  neither matters.
- HTTP/1.0 and odd clients. `curl --http1.0` gets `HTTP/1.1 200`; `curl -H 'User-Agent:'`,
  `-A 'Wget/1.21.4'`, `-A 'mirmon-probe'` and `-A 'python-requests/2.31'` all get 200 on
  the current mirror.
- Plain HTTP. `301` to HTTPS. If the HTTP field is filled in on the form, the redirector
  never uses it ("always redirects to a https mirror") and a probe that follows redirects is
  fine; leave the field blank to keep one URL.

Security settings that would break the probe, and the default of each:

| Setting | Default | Effect on the probe and on `tlmgr` |
|---|---|---|
| Browser Integrity Check | "enabled by default"; "challenges visitors without a user agent or with a non-standard user agent such as commonly used by abusive bots" (https://developers.cloudflare.com/waf/tools/browser-integrity-check/) | observed harmless for curl, wget and an empty UA on this zone; if a challenge ever appears in `smoke`'s output, disable BIC for the hostname with a Configuration Rule |
| Bot Fight Mode | off; "Enable Bot Fight Mode" is a step (https://developers.cloudflare.com/bots/get-started/bot-fight-mode/) | "may challenge API or mobile app traffic"; "You cannot bypass or skip Bot Fight Mode using WAF custom rules"; would challenge mirmon, `tlmgr`, `wget`. Never enable |
| Under Attack mode | off | "Cloudflare will present a Managed Challenge page"; every non-browser client fails. Never enable |
| WAF managed rulesets | Free Managed Ruleset available, not deployed | not needed for static objects; skip |
| Rate limiting rules | none | one `scheme-full` install is 11,919 GETs; any per-IP rule would cut installs off. None |
| Cloudflare Access on the hostname | off | a login page for everyone. Never |
| "Disable domain access" on the bucket's custom domain | enabled | the off switch; `smoke` catches it the same hour |
| Security Level / threat score | "the threat score is always 0 (zero)" now (https://developers.cloudflare.com/waf/tools/security-level/) | no effect |

Leave every one at its default. `smoke` reading `timestamp` back through the domain each
run is the check that they still are.

### Late runs and the age CTAN sees

GitHub starts a scheduled run 15 to 45 minutes after its cron minute and sometimes not at
all (the docs' "can be delayed during periods of high loads"; observed here 39 minutes on
2026-08-26). Upstream rewrites `timestamp` at :02 every hour, and our copy carries the
value dante held at the moment of our listing. Take cron minute M, lateness L (0 to 45
min), run length D (about 1 min for an hourly delta):

```
run k      lists at M + L1; the stamp it copies is the last :02 before that, so already up to 60 min old
run k+1    dropped
run k+2    lands at M + 120 + L2 + D
age just before run k+2 lands = 120 + L2 + D + (stamp age at run k) - L1
worst case: L1 = 0, L2 = 45, D = 1, stamp age 60   ->  226 min, about 3 h 46 min
two dropped slots                                   ->  about 4 h 46 min
a whole day of failed runs                          ->  about 25 h, still under the 28 h band
```

mirmon probes each site at most every 4 hours (`max_poll` default `4h` in the manual), so
the age it displays can lag the real one by up to 4 hours more: about 7 h 46 min displayed
after one dropped slot. The fresh band ends at 28 hours, so a dropped slot is invisible and
a full day of failures is still "fresh". The registration Notes say nothing about start-time
drift: CTAN measures age, not punctuality, and the fixed-minute rule is about not bunching
mirrors' contacts with dante, which drift spreads rather than concentrates.

## 8. HTTP semantics, R2 versus Apache

Observed on `https://ctan.ijosh.com/systems/texlive/tlnet/` and on Apache/nginx mirrors:

| Header or behaviour | Apache mirror | R2 custom domain | Who cares |
|---|---|---|---|
| `Last-Modified` | upstream mtime (`ctan.mirror.rafal.ca/timestamp`, `Wed, 26 Aug 2026 23:02:01 GMT`, the :02 stamp) | upload time (`tlpdb.sha512`, `18:07:24`, the run's minute) | `wget -N` and `curl -z` re-download unchanged files after every run; `rsync` cannot use HTTP; `tlmgr` never sends `If-Modified-Since` (it checks the signed sha512); mirmon reads the body. The README's "check `last-modified`" freshness tip stays true because it means "last sync" |
| `ETag` | inode-size-mtime (`"ba-659fb34fec90b"`) | MD5 of the object for single-part uploads (`"b802c15f..."`), which the AWS CLI's 200 MB threshold guarantees for all but five objects | `If-None-Match` works everywhere; nobody in the TeX toolchain uses it |
| `Accept-Ranges: bytes`, `206` | yes | yes (`curl -r 0-9` gives 206, 10 bytes) | resumable installer downloads |
| `Content-Type` | Apache's `mime.types` | whatever `aws s3 cp` guessed at upload from the runner's `/etc/mime.types` ("By default the mime type of a file is guessed when it is uploaded", https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html); no header at all when the guess fails | observed `.xz` `application/x-xz`, `.pm` `text/x-perl`, `.bat` `application/x-msdos-program`, `.md` `text/markdown`, `.txt` `text/plain`; none for `install-tl`, `texlive.tlpdb`, `.sha512`, `.md5`, `.po`, `config.guess`, `pubring.gpg`. Browsers render `.pdf` and `.html` docs (`application/pdf`, `text/html` are in every table) and sniff extensionless files as text; `.sty`, `.dtx`, `.cls` are `text/x-tex` on Debian's list (unverified on the runner; `python3 -c 'import mimetypes; print(mimetypes.guess_type("a.sty"))'` on `ubuntu-latest` settles it). Acceptable as-is; a per-extension `--content-type` map is one more thing to maintain for no user |
| Directory URL, trailing slash | listing | 404 (section 2, redirect rule) | humans |
| Directory URL, no slash | `301` to the slash form (`mirror.clarkson.edu/.../hyperref` to `.../hyperref/`) | 404 | humans; rare |
| `index.html` inside a package directory | served as the directory's page unless `DirectoryIndex disabled` | never auto-served; `/dir/index.html` is just a file | this is the behaviour CTAN's recipe asks for; 110 packages ship one |
| Case | case-sensitive | case-sensitive (`README.MD` gives 404) | same on both |
| HTTP/1.0 clients | fine | fine (answered as HTTP/1.1) | old `tlmgr` uses `curl`, `wget` or LWP, all 1.1 |
| Plain HTTP | many mirrors serve it | `301` to HTTPS | the redirector is HTTPS-only since 2021 |
| 404 body | Apache's | Cloudflare's R2 HTML page ("Object not found ... Is this your bucket?") | cosmetic; a custom error page is a paid feature |
| Symlinks | followed | copies (section 4) | none, once copied |

## 9. Contact, list, downtime and delisting

- Registration form. Contact person and email are required; they are published on the
  Sites page only as the hostname (the page shows host, city, region and protocol links,
  no email). Use a role address that outlives a mailbox, such as `ctan (at) ijosh.com`.
- The list. "You will be enrolled in the low-traffic mailing list for our mirror
  maintainers." That list is `ctan-mirrors-announce (at) ctan.org`, "Announcements for the
  mirror administrators worldwide", owners at `ctan-mirrors-announce-owner (at) ctan.org`
  (https://mailman.ctan.org/postorius/lists/ctan-mirrors-announce.ctan.org/). It is an
  announce list, not a discussion list; questions go to `ctan (at) ctan.org`.
- Changes and downtime. "Please send updates to this list to ctan (at) ctan.org." Nothing
  on the pages asks for advance notice of downtime; the monitor is the notice. A planned
  stop of more than two days is worth an email so the entry is removed cleanly rather than
  going "bad".
- Delisting. Automatic from the redirector when the probe shows the mirror behind ("If
  your mirror falls behind then mirrors.ctan.org will not redirect to it"), then manual
  removal from the list ("we shall have to remove it from the official list"). Getting
  back is an email.
- CTAN's own contacts (https://ctan.org/contact). CTAN Team c/o DANTE e.V., Heidelberg;
  `ctan (at) ctan.org` for the archive, `webmaster (at) ctan.org` for the site; "Please
  take care not to send any HTML mails".
- Legal. No terms are stated for mirrors. The content is redistributed under each
  package's licence, as on every mirror; the mirror adds nothing of its own except the
  landing page.

## 10. Recommendation

**URL.** `https://ctan.ijosh.com/`, tree at the host root, registered as
`HTTPS://ctan.ijosh.com` with the HTTP, FTP and rsync fields blank. Country United States,
city and region those of the bucket's location (verify in the dashboard). Notes: served
from Cloudflare's network with the origin in R2; directory URLs redirect to
`ctan.org/tex-archive`. Nothing about start-time drift (section 7).

**Storage set.** Everything `rsync -rL` yields minus tlnet's versioned containers and its
six `update-tlmgr-r*` files: 496,149 objects, 132.99 GB, $1.84/month, of which the copies that stand in for symlinks
are 64 GB and $0.96. No redirect stands in for any alias. The 23.64 GB directory alias is
`systems/windows`; its real directory `systems/win32` is what MiKTeX links.

**Index strategy.** One Single Redirect, `http.request.uri.path ne "/" and
ends_with(http.request.uri.path, "/")`, action 302 to
`concat("https://ctan.org/tex-archive", http.request.uri.path)`, in
`cloudflare/redirect-rules.json` beside the cache and transform rules. No generated pages,
no Worker.

**Landing page.** Ours at `/` via the existing Transform Rule, retargeted to
`/.site/index.html`; CTAN's `index.html` stored at `/index.html` as the file it is;
`.site/` and `.state/` are the two reserved prefixes that `stale` and the reconcile never
touch. The page says what the host is, gives the two `tlmgr` lines, links
`ctan.org/tex-archive` for browsing and `/index.html` for CTAN's own page. `robots.txt`
left out of the first cut.

**Security.** Every Cloudflare security setting at its default; never Bot Fight Mode,
Under Attack mode, rate limiting, or Access on this hostname. `smoke` reads `/timestamp`
back every run with a plain `curl`, which is the same thing mirmon does.

**Constraints touched.** `CLAUDE.md` "Objects stay under `systems/texlive/tlnet/`" becomes
"Objects stay at CTAN's own paths from the bucket root; `.state/` and `.site/` are ours".
Endpoints and secrets change only as caching.md already requires (`api.cloudflare.com`,
`CF_API_TOKEN`, `CF_ZONE_ID`); the redirect rule rides on that token with the Single
Redirect edit scope added.

## Open questions

- Does an R2 custom domain map `GET /dir/` to a key named `dir/`? One test upload of a
  small object with that key on a scratch bucket answers it; if yes, option (b) needs no
  rewrite rule and no reserved filename.
- Whether the dynamic-redirect expression editor (`ends_with`, `concat`) is available on a
  Free zone in the dashboard, or only the wildcard form. The docs say wildcards yes, regex
  no, and list `concat` as a general function; the wildcard form `https://ctan.ijosh.com/*/`
  cannot exclude `/`, so the expression form matters. Create the rule and trace it with
  Cloudflare Trace.
- Whether CTAN objects to directory URLs leaving the mirror. Ask in the registration Notes.
- Whether to store the 14,874 versioned tlnet containers so `FILES.byname` never lies
  (6.62 GB, $0.10/month; sibling files decide).
- The bucket's region (dashboard), and therefore the city to register.
- `robots.txt`: none, or a `.site/robots.txt` behind a second Transform Rule with
  `no-cache`.
- Whether the runner's `mimetypes` maps `.sty`, `.cls` and `.dtx` to `text/x-tex`
  (`python3 -c 'import mimetypes; print(mimetypes.guess_type("a.sty"))'` on `ubuntu-latest`).
- Whether Browser Integrity Check ever challenges a client we care about; it did not
  challenge curl, wget, an empty UA or `python-requests` today.

## Sources

Fetched 2026-08-26/27:

- https://ctan.org/mirrors/register
- https://ctan.org/mirrors
- https://ctan.org/mirrors/mirmon
- https://ctan.org/help/mirror-selection
- https://ctan.org/contact
- https://ctan.org/pkg/hyperref and https://ctan.org/tex-archive/macros/latex/contrib/hyperref
- https://ctan.org/tex-archive/documentation
- https://mirror.ctan.org/CTAN.sites, /README.mirrors, /index.html, /timestamp, /robots.txt, and HEADs of `/`, `/documentation/`, `/systems/windows/`, `/systems/win32/miktex/tm/packages/`, `/macros/latex2e/`, `/tds.zip`, `/macros/latex/contrib/hyperref/`
- https://mailman.ctan.org/postorius/lists/ctan-mirrors-announce.ctan.org/
- https://www.mankier.com/1/mirmon
- https://www.tug.org/texlive/acquire-netinstall.html and https://www.tug.org/texlive/acquire-mirror.html
- https://miktex.org/download
- https://yihui.org/en/2026/03/tinytex-ctan-mirror/
- https://developers.cloudflare.com/r2/buckets/public-buckets/
- https://developers.cloudflare.com/r2/reference/data-location/
- https://developers.cloudflare.com/rules/url-forwarding/ (Single and Bulk Redirect quotas, execution order)
- https://developers.cloudflare.com/rules/url-forwarding/single-redirects/create-dashboard/
- https://developers.cloudflare.com/rules/transform/ and https://developers.cloudflare.com/rules/transform/url-rewrite/
- https://developers.cloudflare.com/ruleset-engine/rules-language/functions/
- https://developers.cloudflare.com/workers/platform/limits/
- https://developers.cloudflare.com/bots/get-started/bot-fight-mode/
- https://developers.cloudflare.com/waf/tools/browser-integrity-check/
- https://developers.cloudflare.com/waf/tools/security-level/
- https://developers.cloudflare.com/waf/managed-rules/
- https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
- `tlmgr.pl` from https://ctan.ijosh.com/systems/texlive/tlnet/archive/texlive.infra.tar.xz
- HEAD/GET probes of https://ctan.ijosh.com/ and of the mirrors named in sections 3, 6 and 8 (illinois, clarkson, mit, rafal, ibiblio, latex.us, ctan.net, tex.org.uk, ctan.tikz.jp, us/gb.mirrors.cicku.me, aliyun, tencent, kris.fail, hoobly)
- `SCRATCH/ctan-list-deref.txt`, `SCRATCH/ctan-list-nolink.txt`, `SCRATCH/FILES.last07days`, `SCRATCH/CTAN.sites`, `staging/tlpkg/texlive.tlpdb`
