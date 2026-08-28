# Reference

The verified numbers behind the mirror: what the tree measures, what the platforms allow,
what it costs, how it is watched, and what to do when a run fails. Every figure carries the
date it was verified. Re-verify before leaning on a stale one; the commands that produced
the measured figures are named beside them.

Conventions: GB is 10^9 bytes, GiB and MiB are binary. R2 publishes binary figures, CTAN
decimal. "Stored set" is every regular file `rsync -rL` yields, which is what `normalise`
keeps.

## 1. Baseline

Measured 2026-08-26 from a dereferenced dante listing (538,289 lines, 6.9 s wall clock,
50.7 MB).

| Quantity | Value |
|---|---|
| Stored set | 511,027 objects, 139.63 GB (130.04 GiB) |
| Of which revision-stamped tlnet and tlcontrib containers | 15,133 objects, 7.09 GB, each the same bytes as its stable twin |
| Largest object | 6,865,013,189 B (`systems/mac/mactex/MacTeX.pkg`) |
| Longest key | 151 bytes |
| Distinct directories holding a file directly | 24,953 (24,952 plus the root) |
| Distinct directories, every one incl. file-less | 27,262: adds 2,309 that hold only subdirectories — what `render` draws a page for (2026-08-27) |
| Directory-page objects | 54,523: every directory's page under both keys, all but the root's, which has no slashless key |
| Churn, last 30 days | 16,574 files, 4.72 GB; 23.0 files per hour on average |
| Hour-slots with any change, last 30 days | 284 of 720 |
| Hour-slots over 1,000 files, last 30 days | 3 |
| Busiest hour by count, last 365 days | 5,950 files, 0.037 GB (2026-01-12 16h) |
| Busiest hour by bytes, last 365 days | 20.35 GB in 9 files (2026-03-01 09h, ISO copies) |
| Busiest hour by bytes, installers excluded | under 0.001 GB |
| Objects over R2's 4.995 GiB single-part limit | 5: two MacTeX pkgs at 6.87 GB, three ISOs at 6.78 GB |
| Objects over Cloudflare's 512 MB cacheable limit | 7: the five above plus two `protext` zips at 1.14 GB |
| Whole tree at `BATCH_GB=4` | 28 batches plus 5 lone oversize files = 33 rsync connections; largest batch 98,141 files |

The churn figure is a floor: a file changed twice in a month counts once, because it is
derived from current mtimes. Even at three times the measured rate it stays under 100k
PutObjects a month, well inside the free tier.

## 2. Limits

### R2

Source: [R2 limits](https://developers.cloudflare.com/r2/platform/limits/),
[S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/),
[Multipart](https://developers.cloudflare.com/r2/objects/multipart-objects/). Verified
2026-08-26.

| Limit | Value | What the mirror asks | At the limit |
|---|---|---|---|
| Objects, storage per bucket | unlimited | 511k, 140 GB | billing only |
| Object size | 5 TiB (4.995 TiB in practice) | 6.87 GB max | reject |
| Single-part upload | 4.995 GiB = 5,363,466,240 B | 5 objects exceed it, so go multipart | `EntityTooLarge`, run fails |
| Multipart parts | 10,000, each 5 MiB–5 GiB, uniform | 13 parts per 6.87 GB file at 512 MiB | reject |
| Incomplete multipart uploads | aborted after 7 days | at most one after a cut-off job | storage held up to 7 days |
| Key length | 1,024 bytes | 151 max | reject |
| Writes per key | 1 per second | one state write per batch, minutes apart | 429 |
| `ListObjectsV2` page | 1,000 keys | 497 pages per reconcile | — |
| `DeleteObjects` batch | 1,000 keys | one call in a typical hour | `MalformedXML` |
| Class A operations | 1M/month free, then $4.50/M | ~30 an hour | bill |
| Class B operations | 10M/month free, then $0.36/M | 1 a run plus user traffic | bill |
| Custom domains per bucket | 100 | 1 | — |

`aws.config` sets `multipart_threshold = 4GB`, which the AWS CLI reads as binary — 4 GiB,
safely under the 4.995 GiB single-part ceiling — and `multipart_chunksize = 512MB`, giving
13 uniform parts for the largest file.

### Cloudflare zone, Free plan

The zone is configured by hand and the pipeline never calls the Cloudflare API. Section 6
lists the rules the mirror wants and why.

| Limit | Value | What the mirror asks |
|---|---|---|
| Cache Rules | 10 | 1 |
| Transform Rules | 10, no regex on Free | 2 |
| Configuration Rules | 10 | 1 |
| Single Redirects | 10 | 0 |
| Cacheable object size | 512 MB | 7 objects over it, served from R2 every time |
| Upload through the zone | 100 MB | 0; uploads go to the S3 endpoint |

### GitHub Actions

| Limit | Value | What the mirror asks |
|---|---|---|
| Job time | 6 h, counted from the actual start | `timeout-minutes: 350` |
| Runner disk | 14 GB documented | the 1.2 GB toolbox image, plus at most one 4 GB batch and the 6.87 GB outlier |
| Runner RAM | 16 GB | under 300 MB |
| Concurrent jobs | 20 on Free | 1; `concurrency: sync` queues an overlapping slot |
| Job summary | 1 MiB per step, 20 steps | about 2 KB |
| Cron lateness | starts 15 to 45 min after the slot; slots can be dropped | absorbed by the healthchecks grace |
| Schedule disablement | 60 days without repository activity | a failed healthcheck ping catches it |

Log volume is undocumented and lines are silently dropped, which is why `report` counts from
`RUN/` and never from the log.

Nothing caches the toolbox image. Each job is a fresh VM, `docker build` writes to that VM's
own daemon store, and it dies with the job, so every run rebuilds from the pinned base: 28 s
measured 2026-08-28, against a job measured in minutes. Within a job the `image` task's
`status` guard means one build serves every `task run`. Caching it through the Actions cache
would buy back less than it costs to maintain.

### Docker Hub

Anonymous pulls are limited to 100 per hour per source IP (`ratelimit-limit: 100;w=3600`,
read from `registry-1.docker.io` on 2026-08-28). Each run pulls the pinned Ubuntu base once,
so the mirror wants 1 of the 100. The budget is per IP and GitHub's hosted runners share
egress addresses, so it is not the mirror's alone; a throttled or failed pull fails the run
at the build, before any of the pipeline has run.

### dante and CTAN

| Limit | Value |
|---|---|
| Sync frequency CTAN asks of a mirror | at least once an hour, at a fixed minute |
| mirmon freshness band | 28 h; past it the mirror is not redirected to |
| dante `max connections` | unpublished; the run opens at most 2 sequentially |
| Full listing | 6.9 s, 50.7 MB |
| rsync `MAXPATHLEN` | 1,024 bytes |
| rsync `--max-alloc` | about 1 GB per allocation; the file list is about 10 MB |

rsync exit codes the `retry` task treats as transient: 5, 10, 12, 30, 35. Exit 24 (a file
vanished mid-transfer) is a success — upstream moved under us and the next run picks it up.

### Tools

| Tool | Figure |
|---|---|
| AWS CLI retries | `retry_mode = standard`, `max_attempts = 10`; 20 s backoff cap |
| AWS CLI timeouts | connect 60 s, read 300 s as configured |
| AWS CLI `max_queue_size` | 1,000 tasks; the queue blocks rather than failing |
| `xz` memory | 100 MB at `-6`, 421 MB at `-9` |
| `sort` memory | 68 MB for 511k lines |
| `shasum -a 512` | about 650 MB/s, so about 7 s per 4 GB batch |
| `split` suffixes | `-a 4` is needed above 676 files |

### healthchecks.io

Hobbyist plan, $0: 20 checks, 100 log entries per check, 5 pings per minute per check, ping
body stored to 100 kB. The pipeline sends one ping per run, so the log holds roughly the
last four days.

## 3. Cost

Prices from [R2 pricing](https://developers.cloudflare.com/r2/pricing/), verified
2026-08-26: storage $0.015/GB-month with 10 GB-month free, Class A $4.50/M with 1M free,
Class B $0.36/M with 10M free, egress free. `DeleteObject` and `AbortMultipartUpload` are
free. Cloudflare rounds usage up to the next billing unit, and the unit for operations is
one million — so any Class A overage at all costs $4.50.

| Item | Value |
|---|---|
| Storage today | 139.8 GB-month (0.14 of it directory pages, each stored twice), minus 10 free, 129.8 × $0.015 = **$1.95/month** |
| At the 200 GB ceiling | 190 billable = $2.85/month |
| Class A per month | 30 reconcile listings + 1,440 state writes + churn + a few thousand directory pages ≈ 45k → $0; drawing every page once is 54.5k |
| Class B per month | 720 state reads + about 5,760 `smoke` reads ≈ 6.5k → $0 |
| One uncached `scheme-full` install | 11,919 GETs, 5.51 GB; free until 27 installs a day |
| Budget | $5/month; `plan` refuses a tree over `CEILING_GB` (200) before anything uploads |

Storage is the only line that is ever billed at this size. The design keeps Class A low by
never listing the bucket outside the daily reconcile: an hourly sync that listed the bucket
twice a run would cost $4.50/month at 200 GB.

Whether R2's storage meter counts GB as 10^9 or 2^30 is unverified; this file uses 10^9,
the larger bill. If it is binary the same bucket is 114 GB-month, $1.71.

## 4. Monitoring

One healthchecks.io check, pinged once at the end of a successful run. A failed run is the
only alert.

| Setting | Value |
|---|---|
| Schedule | cron `42 * * * *`, timezone UTC |
| Grace | 3 h |
| Pinged by | `ping`, the last task in `sync` |

The grace covers one dropped slot: the run at the next slot starts up to 45 minutes late and
takes up to about 75 minutes for a full `MAX_BATCHES` delta, so a ping expected at a given
slot may legitimately arrive 160 minutes after it. Three hours leaves margin for curl's
retries. A genuine stall alerts between 3 h and 4 h 40 min after the last success, which
keeps the mirror inside mirmon's 28-hour band with room to spare.

Cron rather than a simple period: a period check resets its window from the last ping, so
sustained lateness drifts the schedule silently. Cron anchors the expectation to the slot.

Nothing sends `/start`, so the grace does not cap run duration. Before a seed or a large
backlog dispatch, pause the check in the UI — a multi-hour run would otherwise blow the
grace. A paused check resumes on its next ping.

The second surface is the run's own job page, where `report` appends the delta, upload,
directory-page, state and storage counts, all read from `RUN/`. It is the first thing to
read on a run that succeeded and still looks wrong. `report` finds the page through
`GITHUB_STEP_SUMMARY`; if a run's summary turns up in the step log instead, that variable
did not reach the container.

## 5. Runbook

Local commands need the four R2 variables in the environment as `sync.yml` maps them, and
`AWS_CONFIG_FILE=aws.config`.
Every task is safe to rerun unless its entry says otherwise: a second run makes the same
writes with the same bytes, or none.

**Diagnose a failed run.**

```sh
gh run list --workflow sync.yml --limit 5
gh run view <id> --log-failed | tail -50
curl -sI https://ctan.ijosh.com/timestamp | grep -i -E 'last-modified|cf-cache-status'
aws s3 cp s3://ctan/.state/applied.txt.xz - | xz -d | wc -l
```

The failed step names the class. The state's line count against the listing's says how far
behind the mirror is.

**Re-run.** `gh workflow run sync.yml`, or `task sync`. Safe: the state is as of the last
checkpoint and the run recomputes the rest. During a seed this is the resume.

**The run failed before the pipeline started.** A failure inside `task: [image] docker build`
is the base image pull, not the mirror: Docker Hub was unreachable or throttling. Nothing was
uploaded and no state moved, so re-running is the whole fix. Section 2 has the pull budget.

**Rebuild the state** — a missing, corrupt or distrusted state file.

```sh
gh workflow run sync.yml -f reconcile=true      # or: task sync RECONCILE=true
```

Lists the bucket and joins it to the upstream listing. Same-size objects are taken as
current, so a same-size change that happened while the state was unusable is missed until
upstream touches the file again. Safe: the rebuild uploads nothing.

**Seed on purpose.**

```sh
gh workflow run sync.yml -f seed=true -f max_batches=40
```

Deleting the state object does not start a seed; it fails every run, by design. Safe for
correctness, not for the budget: the delta is the whole tree, about 511k Class A and 140 GB
from dante. Pause the healthcheck first.

**Delete one key.**

```sh
aws s3 rm s3://ctan/<key>
```

Nothing to purge: the edge holds nothing while the cache bypass rule is in place. If
upstream still has the key, the daily reconcile finds it missing and the next hourly run
re-fetches it. Editing the state by hand to get it back sooner is not worth the risk.

**Redraw every directory page.**

```sh
aws s3 rm s3://ctan/.state/indexed.txt.xz
```

The next run finds no record of what the pages show and draws all 27k directories under
both keys, about twenty minutes and 54.5k Class A. Safe; the bucket's files are untouched.
This is also how a change to the page's own markup reaches the pages already in the bucket,
which are otherwise redrawn only when their directory changes.

**A directory URL is a 404.** The page is at its key regardless of the zone:
`curl -sI https://ctan.ijosh.com/systems/knuth/ctan.ijosh.com.directory.index.html`. If that
is 200 and the directory URL is not, the second Transform Rule in section 6 is missing or
mis-scoped.

**Rotate a secret.**

```sh
gh secret set AWS_ACCESS_KEY_ID
gh secret set AWS_SECRET_ACCESS_KEY
gh workflow run sync.yml && gh run watch
```

Delete the old R2 token in the Cloudflare dashboard after a green run. Safe: the first
call with new credentials is a read.

**Abort a stray multipart upload.**

```sh
aws s3api list-multipart-uploads --bucket ctan --query 'Uploads[].[Key,UploadId,Initiated]' --output text
aws s3api abort-multipart-upload --bucket ctan --key <key> --upload-id <id>
```

Free and safe; R2 does it itself after 7 days.

**dante moved.** Edit `SOURCE` in `Taskfile.yml`, open a PR, merge. Until then every run
spends ten minutes retrying exit 5 and fails, which is the correct loud behaviour.

**The schedule stopped.** Actions tab, the workflow, "Enable workflow"; then any commit so
the 60-day clock restarts. healthchecks recovers on the next ping.

**The run did not start.**

```sh
gh run list --workflow sync.yml --limit 3       # createdAt against the cron slot
gh workflow view sync.yml                       # "disabled" means the 60-day rule or a manual stop
```

Under an hour past the slot is normal; one dropped slot is normal. Two consecutive slots
with no run: check GitHub's status page for an Actions incident, re-enable the workflow if
it is disabled, otherwise dispatch by hand. A hand dispatch is the same run the cron would
have started. Do not add an external trigger to compensate — the hour's work is not lost,
only late.

## 6. Zone configuration

Set by hand in the Cloudflare dashboard, once. The pipeline makes no Cloudflare API call and
holds no zone credentials: four rules that change roughly never are not worth a subsystem
that can fail. Recorded here so the zone can be rebuilt, or a fork configured, without
rediscovering any of it.

Every rule is scoped to the mirror's hostname. A zone usually serves more than the mirror,
and an unscoped path match reaches all of it.

| Where | Rule | Expression | What to set |
|---|---|---|---|
| Configuration Rules | rewriters and the UA filter off | `(http.host eq "ctan.ijosh.com")` | Email Obfuscation off, Rocket Loader off, Automatic HTTPS Rewrites off, Browser Integrity Check off |
| Cache Rules | cache off | `(http.host eq "ctan.ijosh.com")` | Cache eligibility: bypass cache |
| Transform Rules | `/` serves CTAN's `index.html` | `(http.host eq "ctan.ijosh.com" and http.request.uri.path eq "/")` | Rewrite path to `/index.html` |
| Transform Rules | directory URLs serve the mirror's page | `(http.host eq "ctan.ijosh.com" and ends_with(http.request.uri.path, "/") and http.request.uri.path ne "/")` | Rewrite path, dynamic: `concat(http.request.uri.path, "ctan.ijosh.com.directory.index.html")` |

**The Configuration Rule is the one that matters.** Email Address Obfuscation rewrites every
`text/html` response Cloudflare serves: it injects a script, encodes mailto addresses, and
changes the length. Measured 2026-08-27, a 4,006 byte Catalogue entry was served as 4,216
until the rule went on. About 7,300 files in the tree are HTML, and a mirror that alters
them is not a mirror. Rocket Loader is the same hazard through a second switch.

**Automatic HTTPS Rewrites is a third, and it hides from a size check.** On by default, it
rewrites plain `http://` links in HTML to `https://`. Measured 2026-08-28, about 5% of the
tree's 7,229 HTML files came off the domain with bytes the bucket does not hold — roughly
360 files, `+1` byte per rewritten link. On
`biblio/bibtex/contrib/german/dinat/dinat-index.html` the length did not move at all: the
rewriter lengthened one link by a byte and the HTML parser it runs inside collapsed a
newline within the same tag, so the file measured 7,688 bytes either way and differed at
byte 7,150. That is why `smoke`'s canary compares the object with the response rather than
their lengths, and why the sampled-key size checks alone were never going to find this.

**Browser Integrity Check is the third switch, and it fails silently.** On by default, it
answers `403` (Cloudflare error 1010) to any client whose User-Agent matches `LWP`,
`libwww-perl`, `Python-urllib` or `PycURL`. Measured 2026-08-28 against `ftp.fau.de/ctan`
and `ctan.math.illinois.edu`, which serve all four `200`: no other mirror surveyed rejects
them, so leaving it on makes this the one place the mirror is less capable than the archive
it copies. Browsers, curl, wget, Go, Java and `requests` are unaffected, which is why every
casual check passes. `tlmgr` is unaffected too — TeX Live's `TLDownload.pm` sets
`agent => "texlive/lwp"` explicitly, and its downloader order is `lwp curl wget`, so the
first thing it tries carries a string the filter allows. Verified off for the mirror's
hostname alone on 2026-08-28: `ijosh.com` and `www.ijosh.com` still answer `403` to
`libwww-perl`.

Cloudflare drops `content-length` from `text/html` responses whether or not those features
are on, which is why `smoke` sizes an object from a one-byte ranged read rather than a HEAD.

The other three are conveniences: without the cache rule the zone's default caching applies
(it answers `DYNAMIC` today), without the first transform rule `/` is a 404, and without the
second every other directory URL is.

A rule already in one of those phases that serves another hostname is not ours to replace:
a phase is written whole, so anything else in it would go. Add to it rather than over it.

**If caching is ever wanted**, section 3 has the arithmetic: below 10M requests a month the
free tier covers every read and caching saves nothing, a one-hour TTL is worthless because
Cloudflare caches per datacentre, and only a 24-hour TTL pays — first at around 100M
requests a month, which is roughly ten times what a CTAN mirror sees. Turning it on also
means purging every changed key after each batch, which the pipeline no longer does.

## 7. Why directory pages

Investigated 2026-08-27.

**Nothing on the platform can list a directory except the pipeline.** R2 serves no
directory listings and has no index-document setting; Cloudflare's own docs say a public
bucket's domain does "not let you list the bucket contents at the root of your (sub)
domain," an open feature request for years. The rules engine can't generate one either —
Transform, Redirect, Configuration and Cache rules only rewrite requests and responses that
already exist. Snippets are ruled out on their own limits: 5 ms CPU, 2 MB memory, 32 KB
package, no R2 binding. A Worker with an R2 binding could list a bucket, at the price of a
compute layer in front of a mirror that has none, plus `wrangler` against a repo whose tool
list is fixed.

**CTAN documents listings as a mirror convention, not a requirement.** Its
[instructions for mirror operators](https://ctan.org/mirrors/register/) tell them to set
`Options +Indexes` with `DirectoryIndex disabled`, so the ~111 `index.html` files scattered
through the archive's package directories don't shadow the generated listing. The stated
musts are narrower: HTTPS, rsync from `rsync.dante.ctan.org`, and hourly sync at a fixed
random minute.

**Every other mirror follows the convention.** A survey of all 107 mirrors on
[mirmon](https://ctan.org/mirrors/mirmon), run 2026-08-27, fetched `macros/latex/` from
each: 104 returned a directory listing (Apache autoindex, nginx, Caddy's browse template,
and a few themed or JavaScript indexes), 3 failed to connect, and none redirected elsewhere.
At the archive root most serve CTAN's own `index.html` instead, which is why `/` is
rewritten to it here and no other directory URL is.

**Redirecting to ctan.org instead would take the visitor off the mirror.** CTAN's browse
pages link downloads through `mirrors.ctan.org`, which hands the visitor a randomly chosen
mirror — so a directory URL sent there would cost every download that follows, not only the
listing.

**A directory URL without its trailing slash needs a second key, because no rule can
answer it.** Every other mirror 301s `/systems/knuth` to `/systems/knuth/`; Apache can,
because it knows which names are directories. Measured 2026-08-28, `ftp.fau.de`,
`mirrors.mit.edu`, `ctan.math.illinois.edu` and `mirror.las.iastate.edu` all redirect, and
`mirrors.ctan.org` passes a slashless path straight through to whichever mirror it picks, so
a visitor it sends here would land on a 404. Cloudflare's rules run before the origin and
see only the URL, and the URL does not say which of these is a directory:

    /systems/knuth                                 a directory
    /biblio/biber/base/documentation/Changes       a file

**13,259 upstream files carry no extension** (`README`, `Makefile`, `configure`, `VERSION`,
`Changes`), so a rule appending a slash to every extensionless path would 404 all of them —
worse than the 27,262 it fixed. The mirror image of the heuristic fails too: **212
directories have a dot in their own name** (`biblio/bibtex/utils/bibview-2.0/`,
`documentation/german/stammtisch/wuppertal/stybesch/pk/300/mag____0.790/`). A Worker could
retry the 404 with a slash, but it would sit in front of every request to the mirror, and
Workers Paid at $5/month is more than the whole bucket costs.

So `render` writes each page under both keys, `<dir>/INDEX` and `<dir>`, and the slashless
one answers with the listing rather than a redirect. A key can hold a page or a CTAN file
and never both, because upstream is a filesystem, where a name is a directory or a file.
Three consequences worth keeping in mind:

- **The page carries a `<base href>`.** A relative href on `/systems/knuth` would otherwise
  resolve against `/systems/`. The base makes one document correct under both URLs.
- **The slashless copies stage one tree per depth.** No filesystem holds both `a/b` and
  `a/b/`, which is the same fact that makes the two keys collision-free upstream. Within
  `SLASH/<depth>/` every page is a file with the same number of components, so no page is
  another's parent and each tree uploads whole. Max depth in the tree is 14.
- **The slashless key needs its content type given.** It has no suffix, so the CLI would
  send `binary/octet-stream` and a browser would download the page instead of drawing it.

`reconcile` cannot recognise the second key by name the way it recognises the first, so it
spares every bare directory of the state. A path that stops being a directory upstream
leaves that set and is judged as an ordinary file again.
