# Full CTAN mirror: plan and cost analysis

Measured 2026-08-26 against `rsync://rsync.dante.ctan.org/CTAN/`. Prices from Cloudflare's
pages on the same day. Every number here is reproducible with the commands in
[Regenerating the measurements](#regenerating-the-measurements).

## Summary

- CTAN is 69 GB of real files and **133 GB** stored the way `rsync -L` yields it: every
  symlink a real object, only tlnet's versioned containers excluded because nothing requests
  them by that name. 496k objects. No aliasing lives outside the bucket.
- R2 cost for that: **$1.84/month** storage, $0 operations on a daily or hourly sync.
  Popularity adds nothing until ~28 `scheme-full` installs a day uncached; with a cache rule,
  Smart Tiered Cache and purge-on-publish it adds nothing at any traffic we can model, with a
  hard ceiling of **$8.96/month** (every object refetched from origin every day).
- Hourly sync costs one 6.6-second listing from the master and zero bucket listings: the
  previous listing is kept in the bucket as a 3 MB object and the hour's work is a `comm`
  between the two. A daily reconcile listing (497 Class A) catches drift.
- The cache configuration is a JSON file in the repo that every run applies; the run fails
  if the purge fails, and `smoke` proves both the cached and the uncached paths.
- At 200 GB: $2.85/month, ceiling $18.00; an hourly sync that lists the bucket twice per run
  costs $1.27 more, listing once is free.
- Seeding is the first hourly run with no state file: ~34 batches, 4 to 6 hours of runner
  time, resumable across the 6-hour job limit, ~$1 in the seed month. Errors follow one
  model: each tool retries transient errors for minutes, then the run fails and the next
  hour retries; every write is idempotent, so nothing is ever retried unsafely.
- The work is not the money. The 14 GB runner disk forces a list-diff pipeline instead of a
  staged tree; five objects need multipart; the landing page collides with CTAN's own
  `index.html`; official status requires hourly runs and a purge step that adds an endpoint
  and a secret the constraints currently forbid.

## What CTAN is

| Measure | Value |
|---|---|
| Real regular files | 352,357 files, 68.98 GB (64.25 GiB) |
| Directories / symlinks | 18,417 / 24,788 |
| Fully dereferenced (`rsync -L`) | 511,027 files, 139.63 GB |
| File-symlinks materialized | 24,611 files, 35.90 GB (14,872 of them tlnet `*.r[0-9]*.tar.xz`, 6.62 GB) |
| Directory-symlinks materialized | 134,013 files, 34.75 GB (`systems/win32` 23.64, `macros/latex2e` 5.74, `documentation` 2.21, `bibliography` 0.78, `languages` 0.45) |
| **Stored set**: dereferenced minus tlnet versioned containers | 496,155 objects, 133.01 GB |
| Same without the alias directories (`documentation`, `bibliography`, `languages`, `digests`, `tds`, `systems/win32`) | 433,992 objects, 105.85 GB |
| Same without the ISO/pkg alias copies too | 85.42 GB |
| Same without any installers (`systems/texlive/Images`, `systems/mac/mactex`) | 70.39 GB |
| tlnet as mirrored today | 16,978 files, 6.80 GB |

The stored set is what `rsync -rL` produces with one exclude. Every symlink, file or
directory, becomes a real object at the alias path, so every CTAN URL that works on an
Apache mirror works here with no configuration outside the bucket. The alternative, six
Cloudflare redirect rules for the directory aliases, saves 27 GB ($0.41/month) and creates
state in the dashboard that has to track upstream renames (`tds.zip` appeared in 2022, the
`win32`/`windows` pair in 2007); a rule nobody remembers is a 404 nobody notices. The
copies cost less than the attention. The large file aliases are documented download URLs
in any case: `systems/mac/mactex/MacTeX.pkg` is MacTeX's own link and
`systems/texlive/Images/texlive.iso` the ISO's.

Where the bytes are (dereferenced): `systems` 98.7 GB (`texlive/Images` 20.4 = three copies of
one ISO, `windows`+`win32` 47.3, `mac` 15.2, `texlive/tlnet` 13.4 with versioned copies),
`macros` 13.4, `fonts` 7.2, `obsolete` 5.7, `graphics` 3.0, `support` 2.3, `info` 2.2,
`install` 1.7.

Objects over R2's 4.995 GiB single-part limit: `MacTeX.pkg` and `mactex-20260324.pkg`
(6.87 GB), `texlive.iso`, `texlive2026.iso`, `texlive2026-20260301.iso` (6.78 GB). Five
objects, multipart only.

File mix by count: `tfm` 110k, `vf` 36k, `lzma` 35k, `pdf` 34k, `tex` 32k, `xz` 31k, `ltx`
28k, `zip` 14k. Average object 244 KB.

### Churn

| Window | Files touched | Bytes |
|---|---|---|
| 7 days | 8,922 | 1.44 GB |
| 30 days | 15,744 | 3.90 GB |
| 365 days | 76,047 | 55.87 GB |

Those are for the tree without alias directories; on the stored set the 30-day figure is
16,574 files, 4.72 GB, because `systems/win32` churns with `systems/windows`. Bursty: 5,911 files on 2026-08-23, 7 on 2026-08-18. Two outlier days a year: the TeX Live
release (2026-03-01, 20.9 GB, mostly ISO copies) and MacTeX (2026-03-24, 14.0 GB). Without
installers the worst day in the year was 0.72 GB. Root `timestamp` changes every hour at
:02; `FILES.byname`, `FILES.last07days` and `CTAN.sites` change daily.

### What one user costs

From the tlpdb in `staging/`, one `scheme-full` install on `x86_64-linux`:
**11,919 GETs, 5.51 GB** (docs 4,720 GETs / 3.71 GB, sources 2,022 / 0.12 GB; without them
5,177 GETs / 1.68 GB). A `tlmgr update` fetches only changed containers: 461 containers /
0.83 GB changed across all 13 platforms in the last 30 days, so ~100 to 150 GETs per user per
month. Each GET that is not an edge cache hit is one Class B operation.

## Cost model

### Prices (R2 Standard, [pricing page](https://developers.cloudflare.com/r2/pricing/))

| Item | Free per month | Then |
|---|---|---|
| Storage | 10 GB-month | $0.015 per GB-month, billed in whole GB-months, averaged from daily peaks over 30 days |
| Class A (`PutObject`, `ListObjects`, multipart create/part/complete, ...) | 1,000,000 | $4.50 per million |
| Class B (`GetObject`, `HeadObject`, ...) | 10,000,000 | $0.36 per million |
| `DeleteObject`, `AbortMultipartUpload` | free | free |
| Egress, any path (S3 API, r2.dev, custom domain) | free | free |
| Infrequent Access | not used: $0.01/GB retrieval, 30-day minimum, Class A $9/M |

Cross-check: the page's own asset-hosting example (10M reads/day, 290M billable) prices at
$104.40; 290 × $0.36 = $104.40.

### Fixed costs

| Mirror | Objects | Storage | Class A, daily sync | Class A, hourly, listing twice | Class A, hourly, listing once | Class A, hourly, state file + daily reconcile |
|---|---|---|---|---|---|---|
| 133 GB | 496k | $1.84 | 48k → $0 | 734k → $0 | 375k → $0 | 34k → $0 |
| 200 GB | 868k | $2.85 | 84k → $0 | 1,282k → **$1.27** | 657k → $0 | 59k → $0 |

Class A per run = list pages (objects / 1000) × listings per run + PutObjects. Today
`publish` lists twice (`aws s3 sync` and `aws s3 ls` for `stale`). The design in
[Synchronizing with dante](#synchronizing-with-dante) lists the bucket once a day and never
in the hourly path, so the last column is the plan. PutObjects: 18k/month at 133 GB (3.7% of objects rewritten, observed), 32k at
200 GB. Multipart for the five large objects: a few Class A a year.

### Popularity (133 GB, hourly sync, listing once)

| `scheme-full` installs/day | GETs/month | Egress/month | Uncached | Cache rule + tiered + daily purge | Cache rule + tiered + purge-by-URL |
|---:|---:|---:|---:|---:|---:|
| 1 | 0.4M | 0.17 TB | $1.84 | $1.84 | $1.84 |
| 10 | 3.6M | 1.65 TB | $1.84 | $1.84 | $1.84 |
| 100 | 35.8M | 16.5 TB | $11.11 | $1.84 | $1.84 |
| 1,000 | 358M | 165 TB | $126.97 | $1.84 | $1.84 |
| 10,000 | 3.58B | 1.65 PB | $1,285.49 | $1.84 | $1.84 |
| 100,000 | 35.8B | 16.5 PB | $12,870.76 | $3.92 | $3.66 |

Ceiling with caching (every object fetched from origin once a day, ×2 for upper-tier misses
and evictions): **$8.96/month at 133 GB, $18.00 at 200 GB**. Whatever traffic does, the bill
cannot pass that unless the cache is broken.

Assumptions: hot set 13k objects/day (one install set plus binaries for a few platforms);
miss factor 2; five root/`tlpkg/` GETs per install never cached. The 10k+ rows are hundreds
of times CTAN's whole core-node traffic (Wikipedia, uncited: "more than 6 TB per month, not
counting its 94 mirror sites"; mirmon lists 132 mirrors today). As one official mirror you
receive a random share of your region's redirects, split among ~15 to 20 US mirrors.

### Claims and their sources

| Claim | Source | Status |
|---|---|---|
| Storage $0.015/GB-month after 10 GB; Class A $4.50/M after 1M; Class B $0.36/M after 10M; egress free; deletes free | [R2 pricing](https://developers.cloudflare.com/r2/pricing/) | verified |
| GB-month averages daily peaks; rounded up to whole GB-month | same page, "Storage usage" | verified |
| `ListObjects` is Class A; `GetObject` and `HeadObject` are Class B | same page | verified |
| Single-part upload limit 4.995 GiB; unlimited objects per bucket | [R2 limits](https://developers.cloudflare.com/r2/platform/limits/) | verified |
| Cache hits do not incur R2 operations | [How charges accrue](https://developers.cloudflare.com/billing/understand/how-charges-accrue/): "Every cache hit avoids origin fetch costs, Argo routing charges, Workers execution, and R2 operations" | verified (first party); community threads report misconfigurations that still hit R2, so confirm with the R2 operations graph after enabling |
| Free plan may serve large files via R2 | [CDN terms](https://www.cloudflare.com/service-specific-terms-application-services/) require a Paid Service (Developer Platform, Images, Stream) for large files; [Developer Platform terms](https://www.cloudflare.com/service-specific-terms-developer-platform/) list R2 in the Developer Platform | verified |
| Cache Rules: 10 on Free, "Eligible for cache" with Edge TTL override | [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/), [settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/) | verified |
| Cacheable file size 512 MB on Free/Pro/Business | [Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) | verified |
| `.xz` not cached by default; `.zip`, `.pdf`, `.gz`, `.iso`, `.tar` are | same page | verified |
| Smart Tiered Cache on all plans | [Tiered Cache](https://developers.cloudflare.com/cache/how-to/tiered-cache/) | verified |
| Purge Everything 5/min on Free; purge by URL 800 URLs/s, 100 per request | [Purge cache](https://developers.cloudflare.com/cache/how-to/purge-cache/) | verified |
| Transform Rules 10 on Free (one is in use, `/` → `/index.html`) | [Transform Rules](https://developers.cloudflare.com/rules/transform/) | verified |
| Cache Rules are settable by API: `PUT /zones/{zone}/rulesets/phases/http_request_cache_settings/entrypoint` replaces every rule in the phase; token scope `Zone → Cache Rules → Edit` | [Cache Rules via API](https://developers.cloudflare.com/cache/how-to/cache-rules/create-api/), [Rulesets API: update](https://developers.cloudflare.com/ruleset-engine/rulesets-api/update/) | verified |
| Smart Tiered Cache is settable by API: `POST /zones/{zone}/cache/tiered_cache_smart_topology_enable` | [API reference](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/) | verified; token scope not stated on the page, confirm when creating the token |
| Runner: 14 GB SSD, 6-hour job limit, free and unlimited on public repos | [Runner specs](https://docs.github.com/en/actions/reference/runners/github-hosted-runners), [Limits](https://docs.github.com/en/actions/reference/limits) | verified |
| Cron: 5-minute minimum, delayed under load, disabled after 60 days without activity | [Schedule event](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows) | verified |
| Official mirror: 60 GB disk, hourly sync from the primary, HTTPS, monitored | [Becoming a CTAN mirror](https://ctan.org/mirrors/register) | verified |
| Monitor probes `/timestamp`; fresh under 1 day 4 h | [mirmon status](https://ctan.org/mirrors/mirmon), [mirmon manual](https://www.mankier.com/1/mirmon), probe observed via `https://mirror.ctan.org/timestamp` | verified |
| API: 1,200 requests per 5 minutes per token, 429 and a 5-minute block when exceeded, `retry-after` header; 200/s per IP; GraphQL 320 per 5 min | [API limits](https://developers.cloudflare.com/fundamentals/api/reference/limits/) | verified |
| GraphQL Analytics: 300 queries per 5-minute window per user | [GraphQL limits](https://developers.cloudflare.com/analytics/graphql-api/limits/) | verified |
| Rulesets API: rate limits on concurrent updates of one ruleset, numbers unpublished; update whole rulesets in one call | [Rulesets API](https://developers.cloudflare.com/ruleset-engine/rulesets-api/) | verified |
| R2: 1 write per second per key, 429 over that; 10,000 parts per multipart upload | [R2 limits](https://developers.cloudflare.com/r2/platform/limits/) | verified |
| AWS CLI v2: `standard` is the default (3 attempts, base-2 backoff capped at 20 s, retries throttling, 5xx, timeouts); `adaptive` adds client-side rate limiting, marked experimental; `retry_mode` and `max_attempts` in the config file | [AWS CLI retries](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-retries.html) | verified |
| curl `--retry`: transient means timeout, 408, 429, 500, 502, 503, 504; waits 1 s and doubles to 10 min; honours `Retry-After` since 7.66.0; `--retry-max-time` caps the total | `curl --manual`, curl 8.7.1 | verified |
| rsync exit codes: 5 start of client-server protocol, 10 socket I/O, 12 protocol stream, 23 partial, 24 vanished, 30 I/O timeout, 35 daemon connect timeout; the daemon refuses with `@ERROR: max connections (N) reached -- try again later`; no built-in retry | [errcode.h](https://github.com/RsyncProject/rsync/blob/master/errcode.h), [clientserver.c](https://github.com/RsyncProject/rsync/blob/master/clientserver.c), [main.c](https://github.com/RsyncProject/rsync/blob/master/main.c) (a refused handshake exits `RERR_STARTCLIENT`) | verified from source; confirm exit 5 on the runner once |
| healthchecks ping: at most 5 per minute per check, over that 200 and not recorded; body kept to 100 kB | [Ping API](https://healthchecks.io/docs/http_api/) | verified |
| Job summary: 1 MiB per step, 20 per job; over 1 MiB the summary is dropped with an annotation, the step still passes | [Workflow commands](https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions) | verified |

### Costs checked and found absent

Custom domain on the bucket, DNS, Transform Rules, Smart Tiered Cache, purge,
WAF basics: free on the Free zone. GitHub Actions on a public repository: free and unlimited.
healthchecks.io Hobbyist: 20 checks free. Domain: already owned. No per-object storage fee,
no request fee for 404s that is different from a GET (a miss is still a Class B). Not
available on Free and therefore not in the plan: Cloudflare Health Checks (0 on Free),
usage-based billing notifications (Pro and above), Cache Reserve (paid, and it does not apply
to R2 custom domains).

## What the Free plan can cache

- Any response on a proxied custom domain that a cache rule marks "Eligible for cache",
  including extensions Cloudflare skips by default (`.xz`, `.sty`, `.tfm`, `.tex`). Ten rules.
- Up to 512 MB per object. The five installers over that are served from R2 every time; they
  are 5 of 496k objects.
- Edge TTL can be overridden per rule regardless of origin headers; R2 sends none unless set
  per object at upload.
- Smart Tiered Cache: lower-tier PoPs fill from one upper tier near the bucket, so origin
  fetches are roughly one per object per TTL globally instead of one per PoP.
- Purge: by URL (800 URLs/s, 100 per call) or everything (5/min). Both free.
- Not on `r2.dev`; only on a custom domain.

## Caching design

1. Cache rule: hostname `ctan.ijosh.com`, path not starting with `/timestamp`, `/FILES.`,
   `/CTAN.sites`, `/index.html`, `/systems/texlive/tlnet/tlpkg/`, and not the root
   `install-tl*`/`update-tlmgr*` files. Eligible for cache, Edge TTL override, long (the
   purge keeps it honest). Everything a client uses to decide what to fetch stays uncached.
2. Smart Tiered Cache on for the zone.
3. In `publish`, after the last upload and before `smoke`: purge by URL every key the run
   uploaded (the `upload:` lines already in `RUN/publish.txt`, 100 per request). A failed
   purge fails the run.

Why not Purge Everything: on an hourly cadence it empties the cache 24 times a day, which
raises the ceiling 24× and makes every install after a run cold. Per-URL keeps the cache warm
and only the changed keys refetch.

Failure mode that remains: a client fetches the new tlpdb in the seconds between the tlpdb
upload and the purge and asks for a container an edge still holds at the old revision.
`tlmgr` reports a checksum mismatch and the user reruns. This is the same width as today's
archive-then-tlpdb window.

### Cache configuration as code

The rule's scope is what keeps the mirror correct and the bill flat, and a dashboard is the
wrong place for something that important: a click changes it, nothing reviews it, and a
fork cannot reproduce it. So the configuration lives in the repo and every run applies it.

- `cloudflare/cache-rules.json`: the `http_request_cache_settings` phase, one or two rules
  (`cache: true` with an Edge TTL override for the mirror paths, `cache: false` for the
  decision files). `PUT .../rulesets/phases/http_request_cache_settings/entrypoint` replaces
  every rule in the phase, so the file is the whole truth and the call is idempotent.
- `cloudflare/transform-rules.json`: the `/` → `/index.html` rewrite, same mechanism
  (`http_request_transform` phase), so the one rule that exists today is also in the repo.
- `task rules`: two `curl` calls (the two PUTs) plus the Smart Tiered Cache `POST`, run at
  the start of `sync` before `fetch`. It costs nothing and takes a second. If Cloudflare
  rejects the JSON the run fails before anything is uploaded.
- `check.yml` renders it with `task --dry` like everything else, and a PR that touches the
  JSON is reviewed like a Taskfile change.

Token: one `CF_API_TOKEN` with `Zone → Cache Purge`, `Zone → Cache Rules → Edit`,
`Zone → Transform Rules → Edit` and `Account → Account Analytics → Read`, restricted to
`ijosh.com`, plus `CF_ZONE_ID`. Two secrets, one endpoint.

### Guarding against cache errors

Two things can go wrong. The edge serves old bytes (correctness) or the rule stops applying
and every GET reaches R2 (cost). Each has a guard in the pipeline and a backstop outside it.

Correctness:

1. The rule never covers the files a client uses to decide what to fetch (`timestamp`,
   `FILES.*`, `CTAN.sites`, `tlpkg/`, the root installers and updaters). A client always
   sees the current tlpdb.
2. Purge by URL of every uploaded key, and a failed purge fails the run. The mirror is then
   a day stale rather than inconsistent, which is the alerting model already in place.
3. `smoke` fetches three keys from the run's `upload:` list through the domain and `cmp`s
   them with staging; a purge that returned 200 but did not take is caught within the run.
4. Backstop: `tlmgr` checks every container against the signed tlpdb on the client. A stale
   edge produces an error the user sees, never a wrong file installed.

Cost:

1. `smoke` fetches an `archive/` key twice and asserts `cf-cache-status: HIT` on the second
   response, and fetches `tlpkg/texlive.tlpdb.sha512` and asserts it is not `HIT`. This is
   the check that the rule in the repo is the rule on the zone.
2. `report` reads month-to-date Class B and storage from the Analytics API and fails the
   run at 8M Class B or the storage ceiling. That is the budget alert the Free plan does not
   provide.
3. Backstop: uncached traffic costs $0.36 per million GETs and nothing else. There is no
   runaway line item; the 100-installs-a-day row is $11 uncached, and a broken cache at the
   traffic this mirror can realistically draw is a few dollars, found within a month by the
   `report` check.

## Official mirror rules against this design

| Rule ([register page](https://ctan.org/mirrors/register)) | Design |
|---|---|
| HTTPS access to the whole tree at a base address | `https://ctan.ijosh.com/`, tree at the root, paths identical to CTAN's |
| Mirror from the primary node with rsync | `SOURCE` is already `rsync.dante.ctan.org`; list-diff still reads only from it |
| Sync at least hourly, fixed random minute | cron `M * * * *`; 720 runs/month, free |
| Stay fresh; `mirrors.ctan.org` drops stale mirrors | monitor reads `/timestamp` (root, uncached); fresh means under 1 day 4 h |
| Register, join the maintainers' list | manual |
| Apache `+Indexes` in the example config | not a stated requirement; ctan.org links only files and `.zip`s (2,699 of 2,732 `macros/latex/contrib` packages ship one); directory URLs return 404 on R2 |

## Synchronizing with dante

CTAN requires a sync at least hourly and says to mirror from the primary node. Measured
facts that shape the design:

- A full dereferenced listing (`rsync -rL --list-only`) of 523k entries takes **6.6 s** wall
  clock from the master and moves ~19 MB. It is 50.7 MB as text, 3.2 MB as `.xz`. Every
  `rsync -a` mirror pulls the same list every hour, so this is the load CTAN already expects.
- Changes land throughout the day: 283 of the last 720 hour-slots had at least one changed
  file. An hourly run has work about 40% of the time.
- `timestamp` is touched every hour at :02 whether or not anything changed, so it is not a
  "nothing to do" signal, and it must be copied every hour because it is what mirmon reads.
- `FILES.last07days` (8,130 lines) and `FILES.byname` are generated once a day, carry a date
  but no time, and do not list deletions. They are not an hourly change feed.
- `rsync --files-from` with `-L` pulls named paths into an empty directory, creating parents,
  and resolves both file symlinks (`bibliography/.../btxdoc.pdf`) and directory symlinks
  (`documentation/ling-mac.tex`). A path that vanished upstream between the listing and the
  fetch exits 23.
- `aws s3 cp --recursive` from a local directory never lists the destination; only `sync`
  builds a second generator and a comparator (AWS CLI v2 source, `subcommands.py`). So a
  delta upload is exactly N `PutObject` calls and nothing else.

The two sides each have one tool: rsync is the only way dante hands out a listing or a file,
and the AWS CLI is the only way R2 accepts one. Neither tool compares the two trees; that is
`comm` between two sorted text files.

### The hourly run

1. `rsync -rL --list-only` the master into `RUN/upstream.txt`, drop tlnet's versioned
   containers, normalise to `size mtime path`, `LC_ALL=C sort`. 6.6 s.
2. `aws s3 cp s3://tlnet/.state/applied.txt.xz -` and decompress: the listing the last
   successful run left in place. One `GetObject`.
3. `comm -13 applied upstream` is every added or changed line; its paths are the fetch list.
   `comm -23` on the path columns alone is the deletion list. Both are plain text in `RUN/`,
   so `report` can count them.
4. Split the fetch list into batches by cumulative size from the listing: at most 4 GB per
   batch, any file over 4.995 GiB alone in its own batch, and every `tlpkg/` line in the
   last batch so the tlpdb still lands after the containers it names. Sizes are in the
   listing, so this is one `awk` pass; no file is touched to plan it.
5. For each batch: `rsync -rLt --files-from` from the master into an empty `staging/`;
   `verify` on what is local (see below); `aws s3 cp --recursive staging/ s3://tlnet/`
   (N `PutObject`, no listing; the lone-file batches use multipart); purge those URLs; append
   the batch's lines to `applied.txt` and `aws s3 cp` it to `.state/applied.txt.xz`; empty
   `staging/`. Exit 23 from rsync means upstream moved during the run; the run fails and the
   next hour retries.
6. Deletions with `s3api delete-objects`, 1,000 keys per call, free; purge those URLs;
   remove the lines from `applied.txt` and write it again.
7. `smoke`.
8. `report`, `ping`.

The state file is written only after the uploads and the purge it describes have succeeded,
one batch at a time. That is the whole consistency model: a run that dies mid-batch leaves
the state as of the previous batch, so the next hour recomputes the remaining delta and the
interrupted batch repeats, which is harmless because `PutObject` is idempotent and a purge
of an unchanged URL is free. The same loop seeds an empty bucket: the first run's delta is
the whole tree, ~34 batches, and a run cut off by the 6-hour limit resumes at the next batch
an hour later.

### Disk

Runners have 14 GB. Peak use in this design is one batch plus the two listings: at most
4 GB of ordinary files, or one installer (6.87 GB, the largest object on CTAN) alone, plus
~100 MB of text. The bound holds whatever upstream does in an hour; it does not depend on
the churn history. For reference, the worst hour-slot of the last year outside the installer
days was under 0.72 GB, and the two installer days would be one to three lone-file batches
each. `guard` becomes a check on the batch plan, not on `du`: any single file larger than
the runner's free space is a hard failure with a clear message, and there is no such file
today.

Per hour: one listing from dante, one `GetObject`, N+1 `PutObject`, one purge call. Per month
at 133 GB: ~18k `PutObject` plus 720 state writes plus 720 state reads, all inside the free
tiers with room to spare. Nothing in the hourly path is proportional to the size of the
mirror except the 6.6-second listing.

### The daily reconcile

The state file says what the bucket should hold; once a day (the existing 03:30 slot) the
run also asks the bucket what it does hold: `aws s3 ls --recursive` (497 `ListObjects`,
Class A, free) joined to the state file on key and size. Keys missing or wrong-sized go on
the fetch list; keys the state file does not know are deleted. This catches a manual bucket
edit, a partial multipart, or anything the at-least-once model let through, and it is the
only place the mirror's size is measured, so it feeds the storage ceiling and `report`.
~15k Class A a month.

### What `verify` means with a delta

The signed tlnet checks are unchanged where the file is local: the tlpdb's sha512 and
signature, the `.xz` byte match, the root installers and updaters, and the container checksum
of every container in the delta. The one check that needed the whole `archive/` on disk,
"every container the tlpdb names exists", is done against the upstream listing instead: a
named container absent from what dante just listed means dante is mid-update, and the run
fails and retries next hour, exactly as today. A container present upstream and not in the
delta is by construction the copy already verified when it was uploaded.

### Seeding

There is no seed procedure. With no state file, the first hourly run's delta is the whole
tree: 496,155 lines, 133 GB, ~34 batches. The run works through them, checkpointing after
each; if the 6-hour job limit cuts it off, the next hourly run picks up at the next batch.
The steps in [How the project would have to change](#how-the-project-would-have-to-change)
before the seed must be merged first, because the seed exercises all of them.

Runner: GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM, 14 GB SSD, 6 hours per job,
free on a public repository). Nothing else is an option: larger runners cost money and a
self-hosted one breaks the zero-cost constraint. One runner at a time; `concurrency: sync`
without `cancel-in-progress` queues the hourly run that fires during the seed and starts it
the moment the seed job ends, so a resume begins within seconds, not an hour later.
`timeout-minutes` is 360 for the seed and trimmed afterwards.

Duration. The plan has no measurement of a full pull; the estimate scales two numbers from
the tlnet run of 2026-08-26 05:15 UTC (job 32933376123): rsync from dante moved 6.79 GB in
299 s (**22.7 MB/s**, ~400 KB average file) and `aws s3 sync` put at least 7,891 objects in
82 s (**~96 PutObject/s** at 32 concurrent; the log drops lines, so this is a floor).

| Phase | Basis | Estimate |
|---|---|---|
| Fetch 133 GB | 22.7 MB/s → 98 min; CTAN averages 244 KB per object and 146k of them are `tfm`/`vf`, so per-file overhead adds | 2 to 3 h |
| Upload 496k objects | ~100/s → 83 min; bytes at 32-way concurrency are not the bound | 1.5 to 2.5 h |
| Five installers, multipart | 34 GB fetched at 22.7 MB/s, then uploaded | ~0.5 h |
| Purge 496k URLs | 4,962 calls of 100, sequential, held to ~3 calls/s | 25 to 45 min |
| Listing, verify, 34 state writes | | minutes |

Fetch and upload do not overlap (each batch fetches, then uploads), so **4 to 6 hours of
runner time**, right at the 6-hour cap: one run if it goes well, or a cut-off at batch ~30
and a second run of ~40 minutes. Worst realistic wall clock from first run to last is ~12
hours. If the small-file directories halve the upload rate the seed takes three runs;
nothing else changes. The first real seed's `report` replaces these numbers.

Cost of the seed month. ~496k `PutObject` plus 34 state writes plus multipart parts (with
the CLI's default 8 MB chunk, ~860 parts per installer, ~4.3k Class A; a 500 MB
`multipart_chunksize` makes it ~70), so **~500k Class A, inside the free 1M** with the
month's hourly traffic on top. A second full seed in the same calendar month reaches 1M; a
third costs ~$2.25. Storage bills as the average of daily peaks, so the first month is
prorated: seeded on the 15th, ≈71 GB-month, (71 − 10) × $0.015 ≈ **$0.92**, then
$1.84/month. Runner minutes, egress, purges, deletes: $0. Dante hands out 133 GB once,
which is what every new mirror asks of it; say so on the maintainers' list beforehand.

The live tlnet mirror during the seed. The seed writes into the same bucket that already
serves `systems/texlive/tlnet/`. With no state file the delta includes those ~17k keys, so
they are re-put with identical or newer bytes: idempotent, harmless. `tlpkg/` is in the
last batch, so the tlpdb clients see is the pre-seed one until the very end and `archive/`
only ever runs ahead of it, never behind, the same direction as today's ordering rule.
Users are unaffected. Until that last batch lands, `verify` has no `tlpkg/` in its delta
and `report` shows the tlnet checks as skipped; that is the intended behaviour.

Order of the day:

1. Merge everything before the seed step; confirm a normal hourly run works with a state
   file present.
2. Raise `timeout-minutes` to 360. Post to the maintainers' list.
3. `workflow_dispatch` mid-month, mid-morning UTC, so a second run lands the same day.
4. Watch `report` per run: `upstream`, `changed` remaining, storage from the reconcile.
5. When `changed` reaches zero: trim `timeout-minutes`, register, watch mirmon.

### The run after the seed

Neither run below knows whether it is the seed, the resume, or a quiet hour; each does the
same five things (list dante, read the state, `comm`, work the batches, write the state)
and only the size of the delta differs. Delta sizes are illustrative, taken from the churn
table: 16,574 files per 30 days on the stored set is ~23 an hour on average, bursty, and
`timestamp` changes every hour at :02.

**After a clean seed.** The state file holds all 496,155 lines.

```
rules    3 curl calls                                                        ~1 s
list     rsync -rL --list-only  → 496,158 lines                              6.6 s
state    aws s3 cp .state/applied.txt.xz -  → 496,155 lines                   1 GetObject
comm     changed: 15 (timestamp, one new package under macros/ and its alias
         copies, 11 modified); deleted: 1
plan     1 batch, 2.3 MB, no tlpkg/ lines
batch 1  rsync --files-from (2 s) → verify: no tlnet keys, inflation guard ok
         → aws s3 cp --recursive: 15 PutObject (1 s) → purge: 1 call, 15 URLs
         → state: append 15 lines, 1 PutObject → empty staging/
delete   delete-objects: 1 key (free) → purge 1 URL → state rewritten
smoke    /timestamp; one alias copy against its target; archive key twice
         (second is HIT); tlpkg/texlive.tlpdb.sha512 (not HIT); 3 sampled keys
report   ping                                                   total ≈ 30 to 40 s
```

Summary the run leaves on the job page:

```
| Mirror           | 496,158 objects, 133.01 GB at https://ctan.ijosh.com/           |
| Upstream listing | 6.6 s from rsync.dante.ctan.org; 3 added, 11 changed, 1 removed |
| Published to R2  | 15 uploaded in 1 batch, 1 deleted, 16 URLs purged               |
| State            | .state/applied.txt.xz written, 496,157 lines                     |
| Storage          | not measured this run (daily reconcile at 03:30)                |
| Signature        | no tlnet keys in this delta; last verified <time>               |
| Read-back        | timestamp, 1 alias copy, 3 sampled keys match; cache HIT/MISS as configured |
```

The 03:30 run adds the reconcile: `aws s3 ls --recursive` (497 pages) joined to the state
file on key and size; after a clean seed it finds nothing and fills in the Storage row.

**After a seed cut off at six hours.** Say batches 1 to 30 (~120 GB, ~440k lines)
committed and the job died 60% through batch 31's upload. What that leaves: the bucket has
batches 1 to 30 plus ~2.4 GB of batch 31 that nothing records; the state file is exactly
batches 1 to 30, because it is written only after a batch's upload and purge succeed, and
the write is one `PutObject`, so it is the old file or the new one, never half; the live
tlpdb is untouched, since `tlpkg/` was in batch 34; the job is a failure, so GitHub emails
and healthchecks starts its clock; the queued hourly run starts at once.

```
list     → 496,171 lines (dante moved on: 16 changed, 3 deleted, 19 added in 6 h)
state    → ~440,000 lines
comm     changed: ~56,200 = batches 31 to 34 as planned (~13 GB, all of tlpkg/)
         + batch 31's uploaded 60% (the state does not know them)
         + the 16 files whose size/mtime no longer match their state line
         + timestamp;  deleted: 3
plan     re-packed from scratch: 4 batches of ≤4 GB, tlpkg/ still last; a lone-file
         installer batch, if outstanding, is still alone
batch 1  rsync ~4 GB (~3 min) → verify → cp ~15k PutObject (~2.5 min) → purge → state
         (batch 31's 60% is re-put with identical bytes; PutObject is idempotent and a
          purge of an unrequested URL is free)
batch 2  same;  batch 3  same
batch 4  the tlpkg/ tail: verify runs the real tlnet checks here (sha512, gpgv, GOODSIG,
         the pin, the .xz match, every archive/ key in the delta against its checksum,
         every named container present in the listing) → cp → purge → state
delete   3 keys → purge → state
smoke    report  ping                                            total ≈ 25 to 40 min
```

The run after that is the clean-seed shape. The interrupted batch repeats in full rather
than resuming mid-batch; that costs up to 4 GB of redundant `PutObject` (~15k Class A,
free) and ~5 minutes, the price of one checkpoint per batch. A job killed between the
purge and the state write repeats the batch the same way. A job killed during the daily
reconcile's listing leaves nothing to repair; the reconcile is read-only until it produces
its lists.

### Rejected alternatives

- `aws s3 sync` hourly: 497 `ListObjects` per run, 358k a month, still free, but it also
  needs the full tree on local disk to compare against, and the runner has 14 GB.
- Full `rsync -a` into a persistent tree: the standard mirror recipe; impossible without a
  disk. It is also strictly more load on dante than the listing plus a delta.
- `FILES.last07days` as the change feed: daily, no deletions, no times. Useful as a
  cross-check in `report`, not as the source of truth.
- Skipping the hour when `timestamp` is unchanged: it is never unchanged.
- Actions cache for the state file: works, but evicts after seven idle days and is one more
  action; the bucket is already there and the file is 3 MB.

## Remote calls, limits and error handling

Every remote the pipeline touches, every call it makes there, and the limit that applies.
Counts are per hourly run at 133 GB; the seed column is the first run.

| Remote | Tool | Call | Hourly | Seed | Limit |
|---|---|---|---|---|---|
| `rsync.dante.ctan.org` | rsync | `--list-only` listing | 1 | 1 | None published. The register page asks for hourly at a fixed random minute. The daemon's `max connections` refuses with `@ERROR: max connections (N) reached -- try again later`; the client exits 5. |
| same | rsync | `--files-from` pull | 1 per batch, usually 1 | ~34 | same; one connection at a time from this mirror |
| R2 S3 API | aws | `GetObject` (state file) | 1 | 1 | Class B, 10M/month free |
| same | aws | `PutObject` (`cp --recursive`) | ~25 average, up to ~6k on a burst day | ~496k | Class A, 1M/month free; 1 write/s per key, over that 429 |
| same | aws | `PutObject` (state file) | 1 per batch | ~34 | 1 write/s per key; batches are minutes apart |
| same | aws | multipart create, part, complete | 0 | 5 objects | 10,000 parts per upload |
| same | aws | `DeleteObjects` | ≤1 per 1,000 keys | 0 | free |
| same | aws | `ListObjectsV2` (03:30 reconcile) | 497 pages, once a day | 0 | Class A |
| `api.cloudflare.com` | curl | `PUT` cache rules, `PUT` transform rules, `POST` tiered cache | 3 | 3 | 1,200 requests per 5 minutes per token; over that, 429 and every call is refused for 5 minutes, with `retry-after`. Rulesets: no concurrent updates to one ruleset |
| same | curl | `POST` purge, 100 URLs per call | ⌈N/100⌉, usually 1 | ~4,962 | Free: 800 URLs/s, moving average, token bucket; and the 1,200 per 5 minutes above |
| same | curl | GraphQL analytics (`report`) | 1 or 2 | 1 or 2 | 300 queries per 5 minutes per user |
| `ctan.ijosh.com` | curl | `GET` (`smoke`) | ~8 | ~8 | none for our own traffic |
| `hc-ping.com` | curl | `GET` (`ping`) | 1 | 1 | 5 pings per minute per check; over that, 200 but not recorded |
| GitHub | runner | the job | 1 | 1 to 3 | 6 hours per job; job summary 1 MiB per step, over that the summary is dropped with an annotation; the log drops lines under load |

The hourly path uses 1 to 3% of every limit. The seed uses half the Class A month and is
the only place a limit is within reach: its purge stream, held to ~3 calls a second, stays
under both purge limits with a factor of two to spare.

### Two loops, no third

In-run retries handle blips of seconds to minutes; the hourly schedule handles outages. A
tool retries a transient error with exponential backoff for at most ~15 minutes, then the
run fails, the state file stands as of the last committed batch, and the next hour
recomputes the delta and continues. There is no longer backoff than the hour and no
retry-forever anywhere: each hourly attempt costs one 6.6-second listing, it is what every
other mirror's cron does, and a fixed cadence is what mirmon and healthchecks expect. The
failure email and the 2-hour healthchecks grace are the alert; nothing else is added.

This is also the answer to "rate limit ourselves regardless of the call": the pipeline is
one process making sequential calls, except the AWS CLI's 32-way upload concurrency, which
is the one place the mirror can push a remote and the one place the tool throttles itself.

### The inner loop, per tool

Each tool's own retry is used where it exists (two of three have one, and both do more than
a loop in the Taskfile could: `curl` reads `Retry-After`, the AWS CLI adapts its rate).

- **aws.** `retry_mode = adaptive` and `max_attempts = 10` in `aws.config`. Standard mode
  retries throttling errors (`SlowDown`, `TooManyRequests`, ...), 500/502/503/504,
  connection errors and timeouts, exponential base 2, capped at 20 s a wait; adaptive adds
  a client-side token bucket that lowers the request rate when R2 answers 429 or 503, so
  the CLI rate-limits itself with no code of ours. `max_concurrent_requests = 32` is the
  self-imposed ceiling. `cli_connect_timeout = 60` and `cli_read_timeout = 300` bound a
  single call. Adaptive is documented as experimental; if it misbehaves, `standard` with
  the same `max_attempts` loses only the client-side throttling.
- **curl.** One Task variable, `CURL: curl -fsS --connect-timeout 15 --max-time 60 --retry
  6 --retry-connrefused --retry-max-time 600`, used by every curl line. curl retries
  408, 429, 500, 502, 503, 504, timeouts and refused connections, doubling from 1 s, and
  when the response carries `Retry-After` it waits exactly that (so a Cloudflare 429 with
  `retry-after: 300` waits 300 s once, inside the 600 s budget). Not `--retry-all-errors`:
  401 and 403 mean the token is wrong, and that must fail at once.
- **rsync.** No native retry. A `retry` task wraps the two rsync calls (the listing and the
  batch pull) and nothing else:

  ```yaml
  retry:
    internal: true   # CMD: the command; retries on rsync's transport exit codes only
    cmds:
      - |
        for i in 1 2 3 4 5; do
          rc=0; ( {{.CMD}} ) || rc=$?   # subshell so CMD's exit cannot end the loop; || so Task's errexit cannot
          case $rc in 0) exit 0;; 5|10|12|30|35) ;; *) exit $rc;; esac
          s=$((15 * 2 ** i + RANDOM % 30)); echo "rsync exit $rc, retry $i in ${s}s" >&2; sleep $s
        done; exit $rc
  ```

  Exit 5 (daemon refused, including `max connections`), 10 (socket), 12 (protocol stream),
  30 (I/O timeout) and 35 (connection timeout) retry: 30, 60, 120, 240, 480 s plus jitter,
  15.5 minutes in all. Everything else fails the run: 23 and 24 mean a listed path changed
  or vanished, so upstream is mid-update and the hour is the right retry; 11 is our disk.
  The jitter matters against a daemon at its connection limit: without it a refused
  client comes back on the same beat as everyone else it was refused with.
  `--timeout=300 --contimeout=60` stay on every call.

The `retry` task takes any command, so it is the fallback if a future call has no native
retry; today only rsync uses it, and the Taskfile carries one retry loop, not three. The
loop was run under Task 3.53.1 with the sleep scaled down: exit 5 twice then 0 succeeds
after two retries, exit 23 fails on the first attempt, a permanent exit 5 gives up after
five. Task runs every `cmds` entry with errexit even without `set: [errexit]`, which is
why the exit code is captured with `|| rc=$?`; `cmd; rc=$?` never sees a failure.

### Self-imposed rates

- dante: one rsync connection at a time, never parallel; the listing and each batch are
  separate connections, ~2 an hour and ~35 during the seed. The cron minute is fixed and
  random as the register page asks.
- R2: 32 concurrent requests; adaptive mode backs off on any 429 or 503. The state key is
  written once per batch, never more than once a second.
- Cloudflare API: every call sequential from one process. Purge calls carry 100 URLs each
  with a fixed pause between calls (`sleep 0.3` in the loop that feeds them), so at most
  ~3.3 calls and ~330 URLs a second and at most ~1,000 calls in any 5 minutes, under the
  800 URLs/s purge limit and the 1,200 per 5 minutes token limit with margin. A 4 GB batch
  of average objects is ~16k URLs, 164 calls, ~80 s; the hourly delta is one call. Rules
  are two `PUT`s and one `POST` an hour, never concurrent, replacing whole phases, which
  is what the Rulesets API asks. Analytics is one or two queries an hour against 300 per 5
  minutes.
- healthchecks: one ping a run against 5 a minute.
- GitHub: `report` prints counts and the first 20 lines of any list, never a list; a
  summary over 1 MiB is dropped, and a seed batch's upload list alone would be 1.5 MB.

### Timeouts and budgets

Nested, each one inside the next:

| Scope | Bound | Why |
|---|---|---|
| One call | rsync `--timeout=300 --contimeout=60`; curl `--connect-timeout 15 --max-time 60`; aws `cli_read_timeout = 300` | a stalled peer fails in minutes, not at the job limit |
| One step's retries | rsync 15.5 min; curl 10 min; aws 10 attempts at ≤20 s each, ~3 min | a remote down for longer than this is an outage, and the hour handles outages |
| The job | `timeout-minutes: 55` hourly, 360 for the seed | under one cron period, so a stuck run never delays the next by more than itself; a release day (20.9 GB, ~6 batches, ~40 min) fits |
| Across jobs | the hourly cron, the state file, at-least-once | any run may die at any point and the next recomputes the rest |

### Error classes

| Class | Examples | In the run | The run | Alert |
|---|---|---|---|---|
| Transient transport | rsync 5, 10, 12, 30, 35; HTTP 408, 429, 5xx; refused or reset connections; R2 `SlowDown` | retried with backoff | fails if the step's budget runs out | failure email; healthchecks after 2 h |
| Upstream mid-update | rsync 23 or 24; a container the tlpdb names absent from the listing; a fresh container failing its checksum | not retried | fails; nothing uploaded past the last good batch | same; expected a few times a month, clears itself |
| Integrity | bad sha512; bad, expired or revoked signature; wrong fingerprint; `.xz` mismatch | never retried | fails before any upload | failure email; needs a human |
| Configuration | 401 or 403 from Cloudflare or R2; rules JSON rejected | never retried | fails at `rules`, the first step, before anything is fetched | failure email; needs a human |
| Budget | object count or bytes over the ceiling; Class B over 8M; a single file larger than the runner's disk | not retried | fails | failure email; needs a decision |
| Cut off | job timeout; runner lost; GitHub incident | n/a | state as of the last batch; the next hour resumes | none needed unless it repeats |

Every retry is safe because every write is idempotent and the run is at-least-once:
`PutObject` of the same bytes, `DeleteObjects` of a missing key, a purge of an unrequested
URL, a `PUT` of the same ruleset, `rsync --files-from` into an empty directory, and the
state file written last. A request that succeeded but timed out on the way back is repeated
with no effect.

When a remote is down for a day: 24 hourly runs fail and GitHub sends an email for each.
healthchecks sends one. Whether to turn off the Actions failure email and let healthchecks
be the single alert is an open question below.

## How the project would have to change

Each step is a PR that leaves the tlnet mirror working; the order matters.

1. **Decide the storage set and record it.** The stored set is what `rsync -rL` yields
   minus tlnet's versioned containers (133 GB, 496k objects); no aliasing outside the
   bucket. Update `CLAUDE.md` constraints: bucket ceiling 175 GB, endpoints gain
   `api.cloudflare.com`, secrets gain `CF_ZONE_ID` and `CF_API_TOKEN` (scopes listed under
   Cache configuration as code, `ijosh.com` only).
2. **Replace staged rsync with the list-diff `fetch`** from Synchronizing with dante: the
   master's listing against the state file in the bucket, `comm` for the delta,
   `rsync --files-from` into `staging/`, `aws s3 cp --recursive` for the uploads,
   `s3api delete-objects` for deletions (the `ponytail` in `publish` already names this), and
   the state file written after each batch. The daily reconcile replaces `stale`'s bucket
   listing. Retries and timeouts as in
   [Remote calls, limits and error handling](#remote-calls-limits-and-error-handling):
   `retry_mode = adaptive` in `aws.config`, one `CURL` variable, one `retry` task for rsync.
3. **Stream the five large objects.** Anything over 4.995 GiB is fetched and uploaded one at
   a time with a per-file `multipart_threshold` override, then removed from `staging/`, so the
   14 GB runner never holds two. The `aws.config` comment gains the exception.
4. **Keep `publish` ordering; extend it.** tlnet `archive/` first, then the rest, then the
   tlpdb-carrying `tlpkg/`, then deletions, then purge-by-URL, then `smoke`. `PREFIX` becomes
   empty and `BUCKET` the bucket root; `stale` now sees the root, so the landing page must
   either be CTAN's own `index.html` (recommended; every mirror shows it) or move to a key
   `stale` ignores. `page` goes away in the recommended case.
5. **Scope `verify`.** The signed tlnet checks stay exactly as they are and still gate the
   whole publish. The rest of the tree has no upstream signatures; `verify` gains only a
   symlink-inflation guard (object count and bytes from the upstream list against the
   ceiling, before anything uploads) in place of `guard`'s `du`.
6. **Hourly schedule.** `cron: 'M * * * *'` with a fixed random minute; `timeout-minutes`
   sized to the seed, then trimmed. `concurrency: sync` already prevents overlap.
7. **Cache configuration as code, purge step.** `cloudflare/*.json`, `task rules` at the
   head of `sync`, purge-by-URL in `publish`. Ship as one PR with the CLAUDE.md line about a
   purge before `smoke` rewritten as done.
8. **`smoke` for the full tree.** Read back `/timestamp` and compare with staging; read
   back one alias copy (`documentation/...`) and `cmp` it with its target; the two
   `cf-cache-status` assertions and the three-key `cmp` from Guarding against cache errors;
   keep the tlpdb check.
9. **`report`.** Counts from the list-diff (`upstream`, `changed`, `deleted`, `purged`),
   bytes from the upstream list, storage from the bucket listing instead of `du`.
10. **Seed.** No separate procedure: with no state file the hourly loop's delta is the
    whole tree. Runner, duration, cost and the day's order are in [Seeding](#seeding); what
    the next run looks like, whether the seed finished or was cut off, is in
    [The run after the seed](#the-run-after-the-seed).
11. **Docs.** README becomes a CTAN mirror README (tlnet section stays), SECURITY.md states
    that only tlnet is signed, CLAUDE.md "Must knows" gain the list-diff and multipart rules.
12. **Register** at ctan.org/mirrors/register with `HTTPS://ctan.ijosh.com`, then watch
    mirmon for the first green block.

What stays: `task sync` as the only entry point, no shell scripts, the tool list plus
nothing (purge and rules are `curl`; the two JSON files are data, not code), AWS CLI single-part uploads for all but five keys,
never `--size-only`, never `sync --delete`, `report` before `ping` before anything else.

## Monitoring

Today: a failed run is the alert; healthchecks.io emails when a day passes without `ping`;
`smoke` proves the domain serves the tree just published.

Improvements, cheapest first:

1. **Hourly ping with a 2-hour grace** on the existing check; cron delays at the top of the
   hour are documented, so grace must exceed one period.
2. **Age in the ping body.** `ping` posts the `timestamp` line from staging; healthchecks
   keeps 100 log entries per check free, so the mirror's age history is readable there
   without any new service.
3. **Mirmon in `report`.** `curl https://ctan.org/mirrors/mirmon` and grep the host; the
   summary shows the same colour CTAN sees. One new read-only endpoint.
4. **Ops budget in `report`.** The GraphQL Analytics API (`r2OperationsAdaptiveGroups`,
   `r2StorageAdaptiveGroups`, 31-day retention; token scope `Account Analytics → Read`) gives
   month-to-date Class A and B and bytes stored. Print them next to the free-tier lines and
   fail the run when Class B passes 8M or storage passes the ceiling; that is the budget alert
   the Free plan lacks (usage notifications start at Pro). The purge token can carry this
   scope, so it is still one secret.
5. **Cache effectiveness.** The same API's `cacheStatus` on the zone's HTTP requests shows the
   hit ratio; a drop after a publish means the purge or the rule broke. Log it; alert when it
   falls under a threshold for two runs.
6. **Read-back samples.** `smoke` fetches three random keys from the run's `upload:` list
   through the domain and `cmp`s them; catches a purge that silently failed.
7. **Keep the 60-day rule in view.** Dependabot's weekly bumps are the only commits; hourly
   runs do not count as activity. healthchecks catches the outage, and the fix is a commit.

Not on offer at Free: Cloudflare Health Checks (0), usage-based billing notifications
(Pro+). A second healthchecks check pinged only by `smoke` gives a domain-level probe without
either.

## Is `ctan.ijosh.com` a good canonical URL?

Shape: `CTAN.sites` lists 182 HTTP(S) mirror URLs. Most carry a path (`/CTAN/`, `/ctan/`,
`/tex-archive/`, `/pub/CTAN/`); about a dozen serve the tree at the host root
(`ctan.net`, `ctan.tetaneutral.net`, `ctan.mirror.rafal.ca`, `ctan.math.illinois.edu`).
`ctan.ijosh.com` at the root is the cleanest of the common shapes: `ctan.` prefix plus a
personal domain, no path, and every CTAN path is valid verbatim. It is the same shape as
`ctan.math.utah.edu` and `ctan.joethei.xyz`.

Traffic to `ijosh.com`: almost none. The redirector sends users to files; `tlmgr` prints the
host in its output and nothing else. Nobody browses to the apex from a mirror. The visibility
is the hostname in `ctan.org/mirrors`, in `tlmgr` logs, and in `CTAN.sites` inside every
mirror on earth. That is brand presence, not referrals. If referrals matter, the one place a
human lands is a directory URL, and those return 404 on R2; a directory index page is the
only feature that would turn the mirror into a site.

Risks of a personal hostname: users bake it into `tlmgr` config; if the mirror stops, CTAN
delists it and those users see errors until they run `tlmgr option repository ctan`. That
is true of every mirror. The rotation (`mirror.ctan.org`) protects users who never set a
repository, which is most of them.

Recommendation: keep `ctan.ijosh.com`, register it at the root, and let CTAN's own
`index.html` be the front page.

## Open questions

- Whether the hourly listing against dante is welcome at 720/month; it is what every
  mirror's `rsync -a` does and takes 6.6 s, but say so on the maintainers' list.
- Whether to keep tlnet's versioned containers for byte fidelity (6.6 GB, $0.10/month).
  Nothing requests them by that name.
- Whether to turn off the Actions failure email once runs are hourly (a remote down for a
  day is 24 emails) and let healthchecks be the one alert, as it already is for a stalled
  schedule.
- Whether to purge only keys the state file already knew (`changed`), not `added`: an added
  key has never been served, so the seed and every new package would purge nothing. Left
  out until it is known whether the cache rule's Edge TTL applies to a 404 served before
  the key existed.
- Whether directory listings are worth a Worker (not in the tool list) or generated index
  objects (18k more objects, Class A on every changed directory).

## Regenerating the measurements

```sh
rsync -r  --list-only rsync://rsync.dante.ctan.org/CTAN/ > ctan-list-nolink.txt   # real files, symlinks as links
rsync -rL --list-only rsync://rsync.dante.ctan.org/CTAN/ > ctan-list-deref.txt    # what R2 would store
# bytes and counts: awk '$1 ~ /^-/ {x=$2; gsub(",","",x); s+=x; n++} END {print n, s/1e9 " GB"}'
# churn: filter on the date column; per-directory: split($5, p, "/"); hour-of-day: split($4, h, ":")
time rsync -rL --list-only rsync://rsync.dante.ctan.org/CTAN/ > /dev/null   # 6.6 s on 2026-08-26
rsync -rLt --files-from=files.txt rsync://rsync.dante.ctan.org/CTAN/ delta/   # delta pull through aliases
awk '/^name /{n=$2} /^containersize |^doccontainersize |^srccontainersize /{...}' staging/tlpkg/texlive.tlpdb   # per-install GETs and bytes
curl -sL https://mirror.ctan.org/timestamp        # what mirmon probes
curl -sL https://mirrors.ctan.org/CTAN.sites      # mirror URL shapes
```

## Sources

- [R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [R2 limits](https://developers.cloudflare.com/r2/platform/limits/)
- [R2 public buckets and custom domains](https://developers.cloudflare.com/r2/buckets/public-buckets/)
- [R2 metrics and analytics](https://developers.cloudflare.com/r2/platform/metrics-analytics/)
- [How charges accrue](https://developers.cloudflare.com/billing/understand/how-charges-accrue/)
- [Enable cache in an R2 bucket](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/)
- [Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/)
- [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) and [settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/)
- [Tiered Cache](https://developers.cloudflare.com/cache/how-to/tiered-cache/)
- [Purge cache](https://developers.cloudflare.com/cache/how-to/purge-cache/)
- [Transform Rules](https://developers.cloudflare.com/rules/transform/)
- [Cache Rules via API](https://developers.cloudflare.com/cache/how-to/cache-rules/create-api/)
- [Rulesets API: update a ruleset](https://developers.cloudflare.com/ruleset-engine/rulesets-api/update/)
- [Smart Tiered Cache API](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/)
- [Health Checks](https://developers.cloudflare.com/health-checks/)
- [Notifications available](https://developers.cloudflare.com/notifications/notification-available/)
- [GraphQL Analytics API token](https://developers.cloudflare.com/analytics/graphql-api/getting-started/authentication/api-token-auth/)
- [Application Services terms (CDN)](https://www.cloudflare.com/service-specific-terms-application-services/)
- [Developer Platform terms](https://www.cloudflare.com/service-specific-terms-developer-platform/)
- [Becoming a CTAN mirror](https://ctan.org/mirrors/register)
- [Status of CTAN mirrors](https://ctan.org/mirrors/mirmon)
- [mirmon manual](https://www.mankier.com/1/mirmon)
- [CTAN on Wikipedia](https://en.wikipedia.org/wiki/CTAN)
- [TeX Live: downloading and mirroring](https://www.tug.org/texlive/acquire-mirror.html)
- [Yihui Xie, A CDN-backed CTAN mirror](https://yihui.org/en/2026/03/tinytex-ctan-mirror/)
- [AWS CLI v2 `s3` subcommands source (cp has one file generator, sync has two and a comparator)](https://github.com/aws/aws-cli/blob/v2/awscli/customizations/s3/subcommands.py)
- [Cloudflare API limits](https://developers.cloudflare.com/fundamentals/api/reference/limits/)
- [GraphQL Analytics API limits](https://developers.cloudflare.com/analytics/graphql-api/limits/)
- [Rulesets API](https://developers.cloudflare.com/ruleset-engine/rulesets-api/)
- [AWS CLI retries](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-retries.html)
- [rsync errcode.h](https://github.com/RsyncProject/rsync/blob/master/errcode.h), [rsyncd.conf(5)](https://download.samba.org/pub/rsync/rsyncd.conf.5)
- [healthchecks.io ping API](https://healthchecks.io/docs/http_api/)
- [GitHub workflow commands (job summaries)](https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions)
- [GitHub-hosted runner specs](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- [GitHub Actions limits](https://docs.github.com/en/actions/reference/limits)
- [Schedule event](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
- [healthchecks.io pricing](https://healthchecks.io/pricing/)
- Community reports of cache misconfiguration still hitting R2: [R2 files not cached via custom domain](https://community.cloudflare.com/t/cloudflare-r2-files-are-not-cached-when-requested-via-custom-domain/449605), [Hit on R2 Class B with caching](https://community.cloudflare.com/t/hit-on-r2-class-b-with-caching/861466)
