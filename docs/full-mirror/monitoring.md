# Monitoring

How the full mirror's health, freshness, cost and correctness are observed and alerted on,
at zero running cost, with no service beyond what the pipeline already touches: the GitHub
job page, healthchecks.io, the Cloudflare GraphQL Analytics API (read with the same
`CF_API_TOKEN` the purge uses) and CTAN's own mirmon page.

The model is unchanged from today: **a failed run is the alert, and healthchecks.io is the
dead man's switch**. Everything below either feeds those two channels or is a number a human
reads weekly. Numbers dated 2026-08-26/27 were measured from the rsync listing taken that
day, from HEAD requests to `ctan.ijosh.com`, and from the documentation pages in Sources.

Scope adopted by the set. Two things in this file are specified but **deferred**: the
GraphQL `usage` step (budget, cache ratio; it needs `jq`, which is outside the tool list) and
the `health` check that would carry its signals and the mirmon age. Until they land, no run
and no check reacts to Class A, Class B, storage analytics or mirmon; those are read by a
human on the monthly pass (section 10). Two healthchecks checks exist now, `sync` and
`reconcile`. The edge cache is off by default (`CACHE: off`, `caching.md`), so the
`cf-cache-status` assertions in `smoke` run only when `CACHE=on`. Every rule below that reads
"health" or "usage" is the spec for the deferred step, marked as such.

Sibling files own the mechanisms this file only observes: the error classes are in
`errors-and-issues.md`, the cache rule and purge in `caching.md`, the state file and the
daily reconcile in `sync-with-dante.md`, the storage ceiling in `cost-estimates.md`, task
names and `RUN/` files in `taskfile-architecture.md`, registration in
`official-mirror-and-url.md`.

## 1. What must be observed

Each row is a way the mirror can be wrong or cost money without a run failing on its own.
"Fail" means the run fails (Actions failure, healthchecks `sync` goes down). "Health" means
the run continues and the healthchecks `health` check is signalled down with the reason in
the body (section 3). "Note" means a job-summary row or a `::warning::` annotation only.

| Property | Question | Signal | From | Threshold | Action |
|---|---|---|---|---|---|
| Freshness | How old is what we serve, as mirmon sees it? | age = now − first word of `/timestamp` read through the domain | `smoke` | > 4 h note; > 28 h health (deferred) | note / health (deferred) |
| Freshness | Is dante itself moving? | mtime of `timestamp` in the upstream listing | `fetch` | never alerts (tlnet goes quiet for weeks; `timestamp` moves hourly regardless) | note |
| Completeness | Bucket == state file? | reconcile join: keys missing, wrong size, unknown | daily reconcile | any missing/wrong → repaired in the same run; > 100 → note | note |
| Completeness | Did the delta land? | `upload:` lines == fetch-list lines; sampled keys served | `publish`, `smoke` | any mismatch | fail |
| Completeness | Are the five multipart objects whole? | HEAD `content-length` == listing size | `smoke` | any mismatch | fail |
| Correctness | Bytes served == bytes fetched? | `cmp` of 3 sampled uploaded keys and `/timestamp` through the domain | `smoke` | any mismatch | fail |
| Correctness | Did the purge take? | first GET of a purged key is `MISS`, or `HIT` with `Age` < seconds since purge | `smoke` | violated | fail |
| Correctness | Are the decision files uncached? | `cf-cache-status` of `tlpkg/texlive.tlpdb.sha512` and `/timestamp` is never `HIT` | `smoke` | `HIT` | fail |
| Correctness | tlnet signatures | unchanged from today (`verify`) | `verify` | any | fail before upload |
| Cost | Storage | reconcile listing sum now; max(`payloadSize`+`metadataSize`) last 24 h when `usage` exists | reconcile, `usage` (deferred) | > 150 GB note; > 175 GB health (deferred) | note / health (deferred) |
| Cost | Class A month-to-date, last 24 h | `r2OperationsAdaptiveGroups` | `usage` (deferred) | see section 8 | note / health (deferred); never fails a run |
| Cost | Class B month-to-date, last 24 h | same | `usage` (deferred) | see section 8 | note / health (deferred); never fails a run |
| Cost | Cache hit ratio, last 24 h | `httpRequestsAdaptiveGroups` by cache status (field unverified, section 2.4); only meaningful with `CACHE=on` | `usage` (deferred) | < 70 % note; < 40 % two runs health | note / health (deferred) |
| Cost | Pending multipart uploads | `uploadCount` | `usage` (deferred) | > 0 for 24 h | note |
| Liveness | Are runs happening? | healthchecks `sync` check, cron schedule, grace 2 h | `ping` | no ping for period+grace | email from healthchecks |
| Liveness | Are runs succeeding? | `/fail` ping with the run URL | workflow `if: failure()` | any | email from healthchecks (and GitHub, section 9) |
| Liveness | Duration trend | `/start` then success; healthchecks shows run time | `ping` | start without success within grace | email from healthchecks |
| Liveness | Is the reconcile happening? | healthchecks `reconcile` check, period 1 d, grace 12 h | reconcile | missed | email from healthchecks |
| Platform | Will GitHub disable the schedule? | days since last commit (`git log -1 --format=%ct`) | `report` | > 45 of 60 | note (`::warning::`) |
| Platform | Are secrets and scopes valid? | `rules` (first step) gets 401/403; `usage` gets 403 | `rules`, `usage` (deferred) | any | fail at `rules`; `usage` note only |
| Platform | Token/key expiry | `CF_API_TOKEN` TTL; TL signing subkey 2027-07-13 | calendar | 30 days before | manual (section 10) |
| External | What does CTAN see? | our row on `https://ctan.org/mirrors/mirmon` | `report` | age > 1 d 4 h while our runs are green | `::warning::` now; health when it exists |
| External | Are we in the rotation? | our host in `CTAN.sites`; `mirror.ctan.org` redirects | `report` (count only) | absent after registration | note |

Today (2026-08-27) neither `ctan.ijosh.com` nor any `ijosh` host appears on the mirmon page
(132 sites, 40 regions, 11 in the United States) or in `CTAN.sites` (129 HTTPS URLs). Those
two rows are dormant until registration.

## 2. Signals in detail

### 2.1 Freshness

The number mirmon computes is `now − timestamp`, where `timestamp` is the file at the tree
root that dante rewrites every hour at :02 (listing: `-rw-rw-r-- 186 2026/08/26 17:02:01
timestamp`). mirmon's probe reads one line and "interprets the first word on that line as a
timestamp" (mirmon manual). So our own freshness number must be computed the same way, from
the same URL a probe would use:

```sh
# in smoke; GNU date on the runner. Downloaded to a file first: a retried curl must never feed a pipe
{{.CURL}} -o {{.RUN}}/timestamp.served {{.SITE}}/timestamp
ts=$(awk 'NR==1 {print $1}' {{.RUN}}/timestamp.served)
case $ts in *[!0-9]*|'') echo "timestamp first word is not an epoch: $ts" >&2; exit 1;; esac
echo "mirror_age_s=$(( $(date -u +%s) - ts ))" >> {{.RUN}}/smoke.txt
```

Unverified: that CTAN's `timestamp` starts with an epoch. `https://ctan.org/tex-archive/timestamp`
302-redirects to `https://mirrors.ctan.org/timestamp`, which 307-redirects to a random mirror
(`https://mirror.clarkson.edu/ctan/timestamp` today), and this file did not follow to a
third-party mirror. Verify once with `curl -L https://mirror.ctan.org/timestamp`; if the first
word is not numeric, use the `last-modified` header of our own GET instead (`date -d` parses
it). The age is written to `RUN/smoke.txt`, goes in the ping body and the job summary, and
will feed the `health` check at 28 h (mirmon's "oldish" boundary, section 6) once that check
exists; until then it is a `::warning::` at the same line.

Upstream's own movement is a job-summary row only ("dante's timestamp: 2026-08-26 17:02:01",
from the listing). It never alerts: tlnet goes quiet for weeks before a release and CTAN as a
whole had an 86-hour gap with no file change (2026-01-20 06:00 to 01-23 20:00, from the
listing; next longest 55 h). Alerting on upstream silence would page for nothing.

### 2.2 Completeness

Three layers, cheapest first.

1. In `publish`, per batch: the number of `upload:` lines `aws s3 cp --recursive` prints
   equals the number of files rsync put in `staging/` (`find staging -type f | wc -l`). A
   batch that fetched 15 and uploaded 0 fails the run. This is the "runner runs but uploads
   nothing" detector, and it costs a `wc`.
2. In `smoke`, per run: three keys sampled from the run's upload list are read back and
   `cmp`ed (section 5). Because `staging/` is emptied after each batch, `publish` copies the
   sampled files to `RUN/sample/` before emptying (three files, average 268 KB per
   `cost-estimates.md`). Keys under `.state/` and `.site/` are never sampled; they are the
   pipeline's and the landing page's, not the mirror's.
3. Daily reconcile (`sync-with-dante.md`): `aws s3 ls --recursive` joined to the state file
   on key and size. Missing or wrong-sized keys go back on the fetch list; unknown keys are
   deleted. `report` prints the three counts. A count over 100 gets a `::warning::` because
   it means something other than the hourly loop wrote to the bucket.

The five objects over 4.995 GiB get a HEAD each run (`smoke`, section 5): `content-length`
must equal the size in the upstream listing. An abandoned multipart upload leaves no object,
so a HEAD 404 or a short length is the only evidence; `uploadCount` in the storage dataset
shows the pending upload itself.

### 2.3 Correctness

What is verified stays as it is (`verification-and-security.md`): the signed tlnet files and
every container checksum, before anything uploads. Nothing else on CTAN is signed, so for the
rest of the tree "correct" means "the bytes rsync gave us are the bytes the domain serves",
and that is what `smoke`'s `cmp` proves for a sample. The cache adds one more way to be wrong:
an edge holding the previous revision after a same-name overwrite. That is the purge check in
section 5, and the backstop outside the pipeline is `tlmgr`'s own checksum of every container
against the signed tlpdb, which turns a stale container into a client-side error rather than a
wrong install.

### 2.4 Cost

**Status: deferred.** The `usage` step below needs `jq` to read GraphQL JSON, and `jq` is
outside the tool list. Until it is added, no run reads these datasets, nothing fails or
signals on Class A, Class B, storage analytics or the hit ratio, and the numbers are read by
a human from the R2 metrics tab on the monthly pass (section 10). What follows is the
specification for the day it lands.

All four cost signals come from one endpoint, `https://api.cloudflare.com/client/v4/graphql`,
POST with `{"query": ..., "variables": ...}` and `Authorization: Bearer $CF_API_TOKEN`.

Token scopes. The R2 datasets are account-scoped: `Account → Account Analytics → Read` (the
analytics token guide). The zone HTTP dataset is zone-scoped: `Zone → Analytics → Read`
(listed as "Analytics Read" in the permissions reference; the GraphQL errors page says a 403
means the token lacks "Analytics: Read" for the resource). The base plan's token had only the
account scope; it needs both. Restrict the zone scope to `ijosh.com`.

Limits. 300 GraphQL queries per 5-minute window per user (GraphQL limits page); the general
API page says "Max 320/5 min" for GraphQL and 1,200 requests per 5 minutes for the token
overall. `usage` makes three queries an hour. Retention: R2 datasets "can be queried (and are
retained) for the past 31 days", which always covers month-to-date. The zone HTTP dataset's
retention and maximum time range on a Free zone are not documented; the Settings node query
below returns them.

Sampling. Both R2 datasets and `httpRequestsAdaptiveGroups` carry the `Adaptive` suffix,
which means adaptive sampling: "If the number of records is relatively small, sampling is
not used." The sampled rate is exposed as `sampleInterval` (1 = unsampled) and any `sum` or
`count` on an adaptive dataset accepts `confidence(level: 0.95) { ... { estimate lower upper
sampleSize } }`. At this mirror's volumes (thousands of R2 operations a day) sampling is
unlikely; `usage` asks for `avg { sampleInterval }` anyway and prints it, so a sampled month
is visible rather than silently imprecise. Cloudflare also states the datasets "should not be
used as a measure for usage that Cloudflare uses for billing purposes" (GraphQL API index);
they are an early warning, and the bill is the truth.

Lag. Not documented for these datasets. The analytics FAQ says "Metrics are delayed 24
hours for domains on a free Cloudflare plan" in the context of new sign-ups. Treat the last
hour as possibly incomplete: every window below ends at `now − 1 h`.

**Query 1: R2 operations, month-to-date and last 24 h, by operation and status.** Verbatim
structure from the R2 metrics page, with `actionStatus` added as a dimension.

```sh
# usage: one call, two windows, via aliases; $A is the account id, $B the bucket
now=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:00:00Z)
mtd=$(date -u +%Y-%m-01T00:00:00Z)
day=$(date -u -d '25 hours ago' +%Y-%m-%dT%H:00:00Z)
q='query ($a: string!, $b: string, $mtd: Time, $day: Time, $now: Time) {
  viewer { accounts(filter: {accountTag: $a}) {
    mtd: r2OperationsAdaptiveGroups(limit: 200, filter: {datetime_geq: $mtd, datetime_leq: $now, bucketName: $b}) {
      sum { requests } avg { sampleInterval } dimensions { actionType actionStatus } }
    day: r2OperationsAdaptiveGroups(limit: 200, filter: {datetime_geq: $day, datetime_leq: $now, bucketName: $b}) {
      sum { requests } dimensions { actionType actionStatus } }
  } } }'
jq -n --arg q "$q" --arg a "$A" --arg b "$B" --arg mtd "$mtd" --arg day "$day" --arg now "$now" \
  '{query: $q, variables: {a: $a, b: $b, mtd: $mtd, day: $day, now: $now}}' \
| curl -fsS -m 60 --retry 3 -H "Authorization: Bearer $CF_API_TOKEN" -H 'Content-Type: application/json' \
    --data @- https://api.cloudflare.com/client/v4/graphql > {{.RUN}}/r2ops.json
```

Class A and B are then sums over `actionType` names, using the lists on the R2 pricing page:
Class A is `ListBuckets PutBucket ListObjects PutObject CopyObject CompleteMultipartUpload
CreateMultipartUpload LifecycleStorageTierTransition ListMultipartUploads UploadPart
UploadPartCopy ListParts PutBucketEncryption PutBucketCors PutBucketLifecycleConfiguration`;
Class B is `HeadBucket HeadObject GetObject UsageSummary GetBucketEncryption
GetBucketLocation GetBucketCors GetBucketLifecycleConfiguration`; free is `DeleteObject
DeleteBucket AbortMultipartUpload`. Unverified: that the dataset's `actionType` strings are
these names (the metrics page does not enumerate them). `usage` sums the three known lists
and prints every name it did not recognise under "other"; the first run shows the real
vocabulary and the lists get corrected once. Also unverified: that reads served through the
custom domain on a cache miss appear here as `GetObject`. They are billed as Class B, so they
should; test by fetching an uncached key 50 times and re-running the query an hour later.

```sh
# jq: sum requests by class for one window; $names is the Class A list
jq -r --arg w mtd --arg names "$CLASS_A" '
  .data.viewer.accounts[0][$w] as $g
  | ($names | split(" ")) as $a
  | [ $g[] | select(.dimensions.actionType as $t | $a | index($t)) | .sum.requests ] | add // 0' \
  {{.RUN}}/r2ops.json
```

**Query 2: R2 storage, last 24 h.** Verbatim structure from the R2 metrics page.

```graphql
query ($a: string!, $b: string, $day: Time, $now: Time) {
  viewer { accounts(filter: {accountTag: $a}) {
    r2StorageAdaptiveGroups(limit: 48, orderBy: [datetime_DESC],
        filter: {datetime_geq: $day, datetime_leq: $now, bucketName: $b}) {
      max { objectCount uploadCount payloadSize metadataSize } dimensions { datetime } } } } }
```

Storage bills on the average of daily peaks, so `max(payloadSize + metadataSize)` over the
last 24 h is the number to compare with the ceiling, and the newest `objectCount` is a free
cross-check of the state file's line count (they differ by the `.state/` objects and, before
the reconcile has run, by anything the at-least-once model let through). The snapshot
interval is not documented; `limit: 48` covers half-hourly.

**Query 3: zone HTTP requests by cache status, last 24 h.** The dataset, the host filter,
`requestSource: "eyeball"` and `datetimeHour` are verbatim from the hostname tutorial;
`edgeResponseBytes` is verbatim from the colo migration guide. **The `cacheStatus` dimension
is unverified**: no documentation page fetched today names the cache-status field of
`httpRequestsAdaptiveGroups` (the Zone Analytics migration guide shows `cachedRequests` and
`cachedBytes` only on the older `httpRequests1mGroups`). Treat this whole query as
"to be confirmed by introspection" before it goes in the Taskfile.

```graphql
query ($z: string!, $host: string, $day: Time, $now: Time) {
  viewer { zones(filter: {zoneTag: $z}) {
    httpRequestsAdaptiveGroups(limit: 50,
        filter: {datetime_geq: $day, datetime_leq: $now, clientRequestHTTPHost: $host, requestSource: "eyeball"}) {
      count sum { edgeResponseBytes } avg { sampleInterval } dimensions { cacheStatus } } } } }
```

Hit ratio = `count(cacheStatus == hit) / count(all)`. If the field exists, a second version
adds `dimensions { edgeResponseStatus }` (a field on the 1m dataset, unverified on the
adaptive one) so a crawler walking 404s shows up as cost with no benefit.

**How to confirm the schema with a read-only token.** Two calls, both free. `CF` below is
`curl -fsS -H "Authorization: Bearer $CF_API_TOKEN" -H 'Content-Type: application/json' --data @-`
followed by the endpoint `https://api.cloudflare.com/client/v4/graphql`.

```sh
# 1. what the Free zone allows on the dataset (retention, range, page size)
echo '{"query":"query ($z: string!) { viewer { zones(filter: {zoneTag: $z}) { settings { httpRequestsAdaptiveGroups { enabled maxDuration maxNumberOfFields maxPageSize notOlderThan } } } } }","variables":{"z":"'"$CF_ZONE_ID"'"}}' | $CF

# 2. every type whose name mentions the dataset, then its fields
echo '{"query":"{ __schema { types { name } } }"}' | $CF | jq -r '.data.__schema.types[].name' | grep -i httpRequestsAdaptive
echo '{"query":"{ __type(name: \"<DimensionsTypeFromStep2>\") { fields { name } } }"}' | $CF
```

The same Settings query with `r2OperationsAdaptiveGroups` and `r2StorageAdaptiveGroups`
under `accounts(filter: {accountTag: ...}) { settings { ... } }` confirms the 31 days
(`notOlderThan: 2678400`) and the largest window one query may span.

`jq` is not in the project's tool list, which is why the step is deferred. The alternative,
`sed` over GraphQL JSON, works until Cloudflare reorders a field, and was rejected. `jq` is
on the `ubuntu-24.04` runner image (1.7, runner-images readme) and on macOS via Homebrew;
adding it to the CLAUDE.md tools line is the whole cost of un-deferring `usage`.

### 2.5 Pipeline liveness

healthchecks.io, section 3. Three facts about GitHub's schedule shape the settings
(events reference, fetched today): the `schedule` event "can be delayed during periods of high
loads ... High load times include the start of every hour. If the load is sufficiently high
enough, some queued jobs may be dropped"; scheduled workflows run only from the default
branch; and "in a public repository, scheduled workflows are automatically disabled when no
repository activity has occurred in 60 days". The drift's magnitude is not documented. Measure
it from the run history once the hourly schedule is live:

```sh
gh run list -R jshvn/ctan -w sync.yml -e schedule -L 200 --json createdAt \
| jq -r '.[].createdAt' | awk -F'[T:]' '{print $3}' | sort -n | uniq -c   # minute-of-hour histogram
```

Duration is measured by healthchecks from the `/start` ping to the success ping (shown when
the two are under 72 h apart) and recorded in `history.csv` from the runner's own clock.

### 2.6 Platform health

- **Schedule disable.** Dependabot's weekly bumps are the only routine commits; runs do not
  count as activity. `report` prints `days since last commit: N of 60` from
  `git log -1 --format=%ct` on the checkout and emits `::warning::` at 45. healthchecks
  catches the disable after the fact; this catches it before.
- **Secrets and scopes.** `rules` is the first step and makes three authenticated calls;
  a wrong or expired `CF_API_TOKEN` fails there, before `fetch`, with 401/403. R2 credentials
  fail at the state-file `GetObject`. The `usage` step (deferred) is the one that must
  *not* fail the run on 403: a token with purge and rules scopes but no analytics scope
  would otherwise stale the mirror over a graph. `usage` logs the error, leaves the budget
  rows blank and the `health` check gets `/fail` with `usage: 403` in the body.
- **Expiry.** `CF_API_TOKEN` should be created without a TTL. The TeX Live signing subkey
  expires 2027-07-13 and upstream extends it yearly; `verify` fails loudly if it lapses.
  Both go on the monthly checklist (section 10) rather than in code.
- **Runner image.** `aws --version` is already logged each run; add `rsync --version | head -1`
  and `jq --version` to the same line so a runner-image bump that changes behaviour is
  visible in the log of the first run after it.

### 2.7 External view

- **mirmon** (section 6): `report` fetches the page and greps our host; the age cell and the
  colour are a job-summary row; age past 1 d 4 h while the run is green is a `::warning::`
  now and will signal `health` when that check exists.
- **CTAN.sites**: `curl -fsS https://ctan.org/tex-archive/CTAN.sites | grep -c ctan.ijosh.com`
  once a day in the reconcile run (18 KB, changes daily). Zero after registration is a
  `::warning::`; before registration it is expected.
- **`mirror.ctan.org`** cannot be watched from our side: it redirects to a random mirror in
  the requester's region, so a probe from an Azure runner sees whichever mirror the
  redirector picks for that IP. The only observable is whether we are *eligible*, which is
  the mirmon colour plus the CTAN.sites entry (the register page: "If your mirror falls
  behind then mirrors.ctan.org will not redirect to it").

## 3. healthchecks.io design

Verified against the Pinging API, configuring-checks, signalling-failures, attaching-logs,
measuring-run-time, badges and pricing pages fetched 2026-08-27.

**Plan.** Hobbyist: $0, 20 checks, 100 log entries per check. Supporter ($5) has the same
limits. Business ($20) has 100 checks, 1,000 entries and SMS/WhatsApp/phone credits; the
Hobbyist row lists no SMS credits, so on the free plan count on email, webhooks and chat-style
integrations only (the public integrations list is behind login; the docs name email,
webhooks, Slack-style chat, Pushover and incident-management systems). Email reports
("hourly or daily email reminders if any check is down", plus weekly and monthly summaries)
are account settings, free.

**Endpoints** (all accept HEAD, GET and POST; POST bodies are stored up to the first 100 kB,
advertised in the `Ping-Body-Limit: 100000` response header):

| Purpose | URL |
|---|---|
| success | `https://hc-ping.com/<ping-key>/<slug>` |
| start | `https://hc-ping.com/<ping-key>/<slug>/start` |
| failure | `https://hc-ping.com/<ping-key>/<slug>/fail` |
| log only (status unchanged) | `https://hc-ping.com/<ping-key>/<slug>/log` |
| exit status | `https://hc-ping.com/<ping-key>/<slug>/<0-255>` |

Slug URLs rather than UUIDs: one project ping key covers every check, so the secret stays one
(`HEALTHCHECK_URL` = `https://hc-ping.com/<ping-key>`), and a slug that matches no check
returns 404 (a UUID that matches nothing returns 200), so `curl -f` turns a mistyped slug
into a failed run instead of silence. Rate limit: 5 pings per minute per check; over that the
ping returns 200 and is not recorded. This design sends at most 2 per check per run.

**Checks: two now, one deferred.**

| Slug | Schedule | Grace | Pinged by | Body |
|---|---|---|---|---|
| `sync` | cron `M * * * *`, timezone UTC (the workflow's minute) | 2 h | `/start` at the head of `sync`; success from `ping`; `/fail` from the workflow's failure step | success: the run line (below); fail: run URL and which `RUN/` files exist |
| `reconcile` | simple, period 1 d | 12 h | success at the end of a completed reconcile | counts: missing, wrong-size, unknown, objects, GB |
| `health` (deferred, with `usage`) | simple, period 1 d | 1 d | every run: success when every section-8 threshold and the mirmon age pass, `/fail` otherwise | the failing signals, one per line |

Why these settings:

- **Grace 2 h on `sync`.** healthchecks marks a cron check late when the expected ping does
  not arrive, and down after grace. GitHub may delay or drop a scheduled run; one dropped run
  leaves the mirror two hours stale, well inside mirmon's 28-hour "fresh" bound, and is not
  worth an email. Grace of 2 h plus the next run's own drift means a single dropped run is
  silent and two in a row alert. The `/start` signal makes grace also the maximum run
  duration ("Healthchecks.io will detect if the job runs longer than its configured grace
  time"), which fits `timeout-minutes: 55` on the hourly run. A run that hangs to its
  timeout produces the same `/fail` as any other failure.
- **The seed.** A 6-hour job would trip the 2-hour grace after `/start`. Pause the `sync`
  check in the UI before dispatching the seed (`seeding-and-migration.md`); a paused check
  leaves the paused state on its next ping unless "Ignore the ping, stay in the paused state"
  was ticked, so the first successful hourly run after the seed resumes monitoring with no
  further action. The Management API has `POST /api/v3/checks/<uuid>/pause`, but it needs an
  API key, which is a fifth secret for one manual event; the UI click is cheaper.
- **`reconcile` as its own check.** The reconcile is a branch inside the 03:30 run. If the
  branch condition breaks, every `sync` ping still succeeds and the bucket drifts unobserved.
  One ping a day, period 1 d, grace 12 h.
- **`health` with period 1 d (deferred).** It is created together with `usage`, because
  without `usage` its only input would be the mirmon age, which is a `::warning::` for now.
  When it exists: it is pinged hourly, but a run outage must not make it late
  too (that is `sync`'s job), so its period is a day. It exists so budget and external
  breaches produce *one* alert at the transition and stay "down" until fixed, instead of a
  run failure every hour that also stales the mirror (section 8 explains why failing the run
  is the wrong action for Class B). healthchecks alerts on state changes; a reminder while
  down is an account setting.

**Exact `curl` lines.** One Task variable for curl, as `errors-and-issues.md` proposes.
`ping` stays a no-op without `HEALTHCHECK_URL`, as today.

```yaml
vars:
  CURL: curl -fsS --connect-timeout 15 --max-time 60 --retry 6 --retry-max-time 600
  HC: '$HEALTHCHECK_URL'

tasks:
  sync:
    cmds:
      - test -z "$HEALTHCHECK_URL" || {{.CURL}} -o /dev/null "{{.HC}}/sync/start"
      - {task: rules}
      # ... fetch, verify, guard, publish, smoke, report ...
      - {task: ping}
      # - {task: usage} and {task: health}: deferred until jq is in the tool list

  ping:
    desc: Success signal for the sync check, with the run line as the body (100 kB limit; ours is ~300 bytes)
    cmds:
      - test -z "$HEALTHCHECK_URL" || {{.CURL}} -o /dev/null --data-binary @{{.RUN}}/runline.txt "{{.HC}}/sync"

  fail:
    desc: Failure signal with the run URL and the RUN files that exist (which step was reached); run by the workflow on failure
    cmds:
      - >-
        test -z "$HEALTHCHECK_URL" ||
        { printf '%s/%s/actions/runs/%s\nreached: %s\n' "$GITHUB_SERVER_URL" "$GITHUB_REPOSITORY" "$GITHUB_RUN_ID"
        "$(ls {{.RUN}} 2>/dev/null | tr '\n' ' ')" | {{.CURL}} -o /dev/null --data-binary @- "{{.HC}}/sync/fail"; }

  health:   # deferred; shown so the shape is agreed before usage lands
    desc: Budget, cache ratio and mirmon age; never fails the run, signals the health check instead
    cmds:
      - |
        r=$(cat {{.RUN}}/breaches.txt 2>/dev/null)   # written by usage and report, one line per breach
        test -n "$HEALTHCHECK_URL" || exit 0
        if [ -z "$r" ]; then {{.CURL}} -o /dev/null --data-binary @{{.RUN}}/usage.txt "{{.HC}}/health";
        else printf '%s\n' "$r" | {{.CURL}} -o /dev/null --data-binary @- "{{.HC}}/health/fail"; fi
```

The workflow gains one step, which is the one constraint change this file needs in
`.github/workflows/sync.yml` (and the CLAUDE.md sentence "workflows install tools and run one
task" becomes "run one task, and `task fail` if it failed"):

```yaml
      - run: task sync
      - if: failure()
        run: task fail
```

**The ping body** (`RUN/runline.txt`, written by `report`; the same line goes to
`history.csv`):

```
2026-08-27T01:17:04Z run=17435291 dur=38 age=412 upstream=496152 added=3 changed=11 deleted=1 uploaded=15 purged=0 batches=1 storage_gb=132.99 objects=496151 classA_mtd=- classB_mtd=- classA_24h=- classB_24h=- hit_24h=- mirmon=1h/fresh reconcile=-
```

The five `usage` fields are `-` until that step exists, and `purged` is 0 while `CACHE` is
off, so the line's shape does not change when either lands.

**Retention.** 100 log entries per check. With `/start` and success, that is 50 runs, about
two days at hourly cadence; `/fail` bodies share the same 100. So the healthchecks log is the
last two days in detail, and section 4's `history.csv` is the long record.

**Badges.** The README's existing badge (`https://healthchecks.io/badge/<id>/<key>-2.svg`)
reports up/down; the `-2` variant folds "late" into "up". Per-check and per-tag badges
exist; a `sync` badge is the honest "mirror is being maintained" indicator and, once it
exists, a `health` badge the "within budget and fresh on CTAN's monitor" one. JSON and shields.io formats are
available for the landing page if wanted.

## 4. `report`

The job summary is one Markdown table plus two `<details>` blocks, appended to
`$GITHUB_STEP_SUMMARY` (stdout elsewhere). Limits verified today: 1 MiB per step, 20 step
summaries per job; over 1 MiB "the upload for the step will fail and an error annotation will
be created", and the step still passes. So the summary never carries a list. A seed batch's
upload list is ~1.5 MB on its own; even the hourly `changed` list on a burst day (5,911 files
on 2026-08-23) is 400 KB of paths that nobody reads on a web page. Lists stay in `RUN/` and
in the job log; the summary carries counts and the first 20 lines of anything worth a glance.

**Rows for the full mirror.** Every value is a number `report` reads from a file in `RUN/`;
the file's producer is named so a blank cell points at the step that did not run.

| Row | Content | Source |
|---|---|---|
| Mirror | objects, GB at `https://ctan.ijosh.com/` | line count and size sum of the state file written last (`RUN/applied.txt`) |
| Upstream listing | lines, seconds from `rsync.dante.ctan.org`; added, changed, removed; dante's `timestamp` mtime | `RUN/upstream.txt`, `RUN/list-time.txt` (`/usr/bin/time -f %e` around the listing), the `comm` outputs |
| Published to R2 | uploaded, in N batches; deleted; URLs purged in N calls | `upload:` lines in `RUN/publish.txt`; `RUN/deleted.txt`; `RUN/purge.txt` |
| State | `.state/applied.txt.xz` written N times, final line count | `RUN/state-writes.txt` |
| Reconcile | "not this run" or missing / wrong-size / unknown counts and bucket objects, GB | `RUN/reconcile.txt` |
| Storage | bucket GB from the reconcile listing against the ceiling; when `usage` exists, max over 24 h from the storage dataset and pending multipart | `RUN/reconcile.txt`; `RUN/usage.txt` (deferred) |
| Operations | Class A and B, month-to-date and last 24 h, against 1M and 10M; "other" names if any; sample interval | `RUN/usage.txt` (deferred; the row reads "not measured" until then) |
| Cache | `off` or `on`; with `on` and `usage`: hit ratio last 24 h, requests, bytes at the edge | `CACHE`; `RUN/usage.txt` (deferred, and blank until the field is confirmed) |
| Signature | the tlnet sentence when `tlpkg/` was in the delta, else "no tlnet keys in this delta; last verified <date>" | `RUN/verify.txt` |
| Read-back | age, alias copy, sampled keys, cache statuses, 404, big-object HEADs, one word each | `RUN/smoke.txt` |
| Mirmon | "not listed" or age and colour as CTAN shows it | `RUN/mirmon.txt` |
| Platform | days since last commit of 60; runner tool versions | `git log`, the version line |
| Run | duration, runner peak disk (`df` after the largest batch) | `RUN/timing.txt` |

`<details>`: the rsync `--stats` block of the largest batch, and the first 20 lines of
`changed` and `deleted` (the top of the sorted list, which is alphabetical, not the most
interesting; still enough to recognise a release day).

Warnings go through `::warning::` annotations, which render on the run page and in the
Actions list without opening the log. `report` writes them for: days since commit > 45,
reconcile counts > 100, mirmon age > 4 h (and > 28 h, until `health` exists), and, once
`usage` exists, the budget "note" thresholds (section 8) and `usage` unavailable.

**Persistent history: comparison and choice.**

| Store | Cost per run | Retention | How a human reads it | Records failures? |
|---|---|---|---|---|
| `.state/history.csv` in the bucket, one line appended per successful run | 1 `GetObject` + 1 `PutObject` (Class B + Class A, ~1,500 of each a month); ~300 bytes per run, ~2.6 MB a year | unbounded | `curl -s https://ctan.ijosh.com/.state/history.csv \| tail`, or `aws s3 cp` | successes only (`report` runs after `smoke`); failures are in healthchecks and on the Actions page |
| healthchecks events | 0 | 100 entries per check (~2 days at hourly with `/start`) | web UI; Management API needs a key | yes |
| GitHub run list (`gh run list --json conclusion,createdAt,updatedAt`, or the REST endpoint, 100 per page) | 0 | logs and summaries 90 days by default (public repos: 1 to 90); whether run metadata outlives that is unverified | needs `gh` (on the runner, not in the tool list) and a token | yes, but no mirror figures |
| Job summaries | 0 | with the logs, 90 days | one page per run, no aggregation | per run |

Choice: **`.state/history.csv` in the bucket.** It uses the same object store and the same
credentials the state file already uses, it is public at a stable URL so the landing page or
a `task history` can show the last week without any login, its cost is two operations against
free tiers of a million and ten million, and it is the only option that keeps a year. Its gap,
failures, is filled by the `/fail` body (run URL, step reached) in healthchecks, and the run
list on GitHub for the 90 days that matter.

```yaml
  history:
    desc: Append RUN/runline.txt to .state/history.csv in the bucket (one GetObject, one PutObject)
    set: [pipefail]
    cmds:
      - >-
        { aws s3 cp {{.AWS_FLAGS}} s3://tlnet/.state/history.csv - 2>/dev/null || true; cat {{.RUN}}/runline.txt; }
        | aws s3 cp {{.AWS_FLAGS}} --content-type text/plain - s3://tlnet/.state/history.csv
```

Two consequences for sibling files: the reconcile and any stale-key logic must treat
`.state/` as the pipeline's own (`sync-with-dante.md`), and the cache rule must not cache
`.state/` (`caching.md`), or `tail` reads yesterday.

## 5. `smoke` for the full mirror

Runs after the last upload, the deletions and (with `CACHE=on`) the purge; the run fails on
any assertion. All requests go through the public domain from the runner, so every miss is
one Class B and every hit is free. Roughly a dozen requests per run.

Two rules for every check. First, no `curl` output ever feeds a pipe: `{{.CURL}}` retries,
and a retry after partial output would corrupt whatever reads the pipe (`errors-and-issues.md`),
so each response is written with `-o` to a file under `RUN/` (`-D` for headers) and `cmp`,
`awk` or `grep` read the file. Second, keys are sampled from the run's upload list with
`.state/` and `.site/` filtered out first (`grep -v '^\.s'`), so the pipeline's own objects
and the landing page never stand in for the mirror.

Checks 5, 6 and 7's cache-status assertions run only when `CACHE=on`; with the default
`CACHE: off` there is no purge and every response is `DYNAMIC`, and check 4 gains a
`Content-Length` assertion in their place.

| # | Check | Command (sketch) | Proves | Cannot prove |
|---|---|---|---|---|
| 1 | `/timestamp` read-back | `{{.CURL}} -D RUN/ts.h -o RUN/ts.b $SITE/timestamp && cmp RUN/ts.b staging/timestamp` (the file is kept from the last batch); with `CACHE=on`, `cf-cache-status` in `RUN/ts.h` must not be `HIT` | the tree root uploaded this run is what mirmon will read, and it is not edge-cached | that a PoP other than the runner's serves the same |
| 2 | Mirror age | first word of `RUN/ts.b` versus `date +%s` (section 2.1) | what CTAN will compute | the format, until verified once |
| 3 | Alias copy == target | pick one alias key from this run's upload list (`documentation/` → `info/`, `bibliography/` → `biblio/`, `languages/` → `language/`; sizes in the listing confirm each pair is byte-identical: `documentation` 28,888 files, 2.21 GB = `info`), else a fixed pair; download both to `RUN/alias.a` and `RUN/alias.b`, then `cmp` the files | the dereferenced copy landed and the two paths serve the same bytes | that every alias copy did |
| 4 | Three sampled uploads | `grep -v '^\.s' RUN/uploaded.txt \| shuf -n 3`, copies kept in `RUN/sample/` before `staging/` was emptied; download each to `RUN/sample/<n>.served` with `-D`, `cmp` with the kept copy, and assert the `Content-Length` header equals the size in the upstream listing | the bytes rsync fetched are the bytes served, for a random sample; the object is whole (the length check is the cache-off stand-in for checks 5 to 7) | the other N−3 |
| 5 | Purge took (`CACHE=on` only) | for the three keys, from the saved headers: first response must be `MISS`, or `HIT` with `Age` ≤ seconds since the purge call (`Age` is set on `HIT` and counts from admission, reset on purge; with tiered cache it can be inherited from the upper tier) | the edge that serves the runner did not keep the old revision | a PoP the runner does not hit |
| 6 | Cache rule is live (`CACHE=on` only) | one cacheable key fetched twice: second response `cf-cache-status: HIT`; `tlpkg/texlive.tlpdb.sha512` and `/timestamp`: never `HIT` (expected `DYNAMIC`, "not eligible for cache at request time"). With `CACHE=off` the inverse: no response may be `HIT`, which proves nobody left a rule on | the rule in the repo is the rule on the zone, for both halves of it | nothing about TTL length |
| 7 | 404 for a deleted key | one key from `RUN/deleted.txt`: expect HTTP 404 (today a missing key returns `404 text/plain` with `cf-cache-status: DYNAMIC`); a `200` means the delete (or, with `CACHE=on`, its purge) did not happen | deletions (and their purges) are applied at this PoP | ditto for other PoPs; with `CACHE=on`, Cloudflare caches 404s for 3 minutes by default when the path is cacheable |
| 8 | Big objects whole | `{{.CURL}} -I -o RUN/big<n>.h` each of the five keys over 4.995 GiB: `content-length` == listing size; with `CACHE=on`, `cf-cache-status` must be `BYPASS` or `DYNAMIC`, never `HIT` (over the 512 MB Free-plan limit: "the response exceeds the maximum cacheable file size for your plan" produces `BYPASS`) | the multipart uploads completed to full size; the installers are served from origin as expected | content; five `HeadObject` a run, 3,600 a month |
| 9 | tlpdb sha512 | as today, except downloaded to `RUN/sha512.served` with `-o` and then `cmp`ed against staging (today's Taskfile pipes `curl` into `cmp -`; the pipe goes) | the signed decision file is live | |

What `smoke` cannot prove with `CACHE=on`, and how much that matters. Every assertion is made from one
vantage point, so a purge that succeeded at the runner's PoP and failed at another is
invisible. Cloudflare documents purge by URL as removing the object "across all data
centers", and Smart Tiered Cache means every lower-tier PoP fills from one upper tier near
the bucket, so a stale copy can only survive in a lower-tier PoP that both cached it before
the purge and was missed by the purge. The exposure is then bounded by the Edge TTL at that
PoP; `caching.md` chooses the TTL, and this file's request is that the bound be no longer than
a day, so that "stale at one PoP" is never worse than "oldish" on mirmon. Below that bound the
client-side backstop holds: `tlmgr` checksums every container against the signed tlpdb, and
for the rest of CTAN a wrong file is a wrong file for at most one TTL. No free observation
covers a second PoP; a second runner region is not on offer. With `CACHE=off` none of this
applies: every request reaches R2, and check 4's `cmp` plus `Content-Length` is the whole
correctness proof.

## 6. Mirmon

CTAN's page (`https://ctan.org/mirrors/mirmon`, mirmon 2.11, "last check Thu Aug 27
00:03:01 2026 (UTC)") is regenerated hourly; the manual says mirmon "is intended to be run by
cron every hour" and probes sites that are new, bad, or "not probed for a specified time"
(`max_poll`, default 4 h; `min_poll` 1 h). The probe fetches the project's timestamp file
from each mirror URL and takes the first word of the first line as an epoch; the exit status
is ignored, so a 200 with the wrong content is a failed probe ("no time" in the last-stat
column, 7 of 132 sites today).

Status from the page's own legend, which matches the manual's defaults
(`min_sync` 1 d, `max_sync` 2 d, `max_poll` 4 h):

| Status | Age | Meaning |
|---|---|---|
| fresh | 0 to 1 d 4 h | `min_sync + max_poll` |
| oldish | 1 d 4 h to 2 d 4 h | `max_sync + max_poll` |
| old | over 2 d 4 h | |
| bad | site or tree never found | |

Today: 118 fresh, 1 oldish, 13 old, 0 bad; mean age 32 h, median 3 h; 125 probes ok. Age is
"based upon the last successful probe", the daily status block is appended when the last one
is 24 h old, and the probe history shows success/failure per probe. The register page adds
the consequence: "If your mirror falls behind then mirrors.ctan.org will not redirect to it,
and we shall have to remove it from the official list."

**Reading our row.** Each site is one row of five cells: host (with an at-sign link to the
site root), type (`https`), mirror age ("1 hour", "22 hours", "12.5 days"), last probe
("renewed", or an age when the last probe failed), last stat ("ok" or "no time"). "renewed"
appears on 125 of 132 rows, all with last stat "ok", and reads as "probed successfully in the
latest cycle"; that wording is an inference from the page, not from the manual.

**How `report` scrapes it.**

```sh
# once per run; never fails the run; ~80 KB HTML
curl -fsS -m 30 https://ctan.org/mirrors/mirmon -o {{.RUN}}/mirmon.html \
  && awk 'BEGIN{RS="<tr"} /ctan\.ijosh\.com/ {gsub(/<[^>]*>/," "); gsub(/[ \t\n]+/," "); print; exit}' {{.RUN}}/mirmon.html > {{.RUN}}/mirmon.txt \
  || echo "mirmon unavailable" > {{.RUN}}/mirmon.txt
test -s {{.RUN}}/mirmon.txt || echo "not listed" > {{.RUN}}/mirmon.txt
```

Rate: once a run, 720 fetches a month of a page that changes hourly; the same load as one
browser tab refreshing. It could drop to the daily reconcile run, but the signal it carries
("CTAN sees us stale while we see ourselves fresh") is the one that means the *serving* path
is wrong, not the pipeline, and an hour's notice is worth 80 KB.

**Is "stale on mirmon" alertable from our side?** Yes, and it is the most valuable external
signal we have, because it is the only probe of the domain that does not run on our runner.
The rule: if our row's age exceeds 1 d 4 h (oldish) while the run is green, `report` emits
a `::warning::` now, and once the `health` check exists (deferred with `usage`, section 3)
`health` goes down with `mirmon: <age>` in the body. That combination means `/timestamp` as served to
their probe differs from what we just verified: an edge caching `/timestamp` somewhere, a
TLS or redirect problem for their client, or a probe blocked by a WAF rule. A failed probe
("no time") three cycles running gets the same treatment. Before registration the row is
"not listed" and the rule is inert.

## 7. Silent-failure audit

Scenarios in which every run is green and the mirror is wrong or costing money, with the
cheapest observation that catches each. Rows whose catch names `usage` or `health` are caught
by the deferred step only; until it exists the catch is the monthly pass over the R2 metrics
tab (section 10). Rows about the cache rule or the purge apply only with `CACHE=on`; with
the default `CACHE: off` those scenarios cannot occur, and `smoke` #6 asserts that nothing is
`HIT`.

| Scenario | Why runs stay green | Cheapest catch | Where |
|---|---|---|---|
| Cache rule silently not applying (dashboard edit, API change, phase replaced by something else) | every GET is `DYNAMIC`; origin serves fine | `smoke` #6 second-GET `HIT`; Class B last 24 h; hit ratio | `smoke`, `usage` |
| Cache rule applying to the wrong paths (caching `tlpkg/`, `/timestamp`, `.state/`) | clients get old decision files; runs never read through the cache except in `smoke` | `smoke` #1 and #6 "never HIT"; mirmon age vs our age | `smoke`, `report` |
| Purge returns 200 but does not purge | uploads succeed; `smoke` without the `Age` rule would see a `HIT` and be happy | `smoke` #5 (`MISS` or fresh `Age`); #4 `cmp` | `smoke` |
| State file drifts from the bucket (manual delete, partial batch, a 200 that did not persist) | hourly `comm` trusts the state | daily reconcile counts; `objectCount` from the storage dataset vs state line count every hour | reconcile, `usage` |
| Deletions never applied (the delete step skipped, or the state rewritten before the delete) | nothing reads deleted keys | `smoke` #7 404; reconcile "unknown" count; storage creeping up | `smoke`, reconcile |
| Cron stopped (60-day disable, schedule dropped repeatedly, workflow disabled by hand) | no run, no failure | healthchecks `sync` late then down; `days since last commit` warning at 45 | healthchecks, `report` |
| Runner runs but uploads nothing while upstream changed (a `cp` that matches zero files, a wrong `--files-from`, an empty batch plan) | `aws s3 cp` exits 0 with nothing to do | `publish` asserts `upload:` lines == fetched files; `report` shows changed > 0 and uploaded = 0 | `publish` |
| The whole tree re-uploaded every hour (state file lost, or `.state/` deleted by the reconcile) | at-least-once makes it "correct"; 496k Class A an hour | `guard` on the delta: with a state file present, a delta over 20 % of it fails before fetch; Class A last 24 h | `guard`, `usage` |
| Storage creeping toward the ceiling (an alias directory or installer set growing) | each run is small | pre-upload guard on the listing total; `usage` storage max vs 150/175 GB; top-5 directory growth in `report` | `guard`, `usage`, `report` |
| An alias directory inflating (upstream turns a symlink into a large real tree; a new alias) | bytes arrive one batch at a time | `report` "largest directory deltas this run" (counts, not lists); guard total | `report`, `guard` |
| Mirmon red while our runs are green | our probe and theirs differ | mirmon row; `health` on age > 28 h or 3 failed probes | `report`, `health` |
| `/timestamp` served by an edge from cache | `smoke` #1 sees the runner's PoP only | mirmon (their probe, their PoP) | `report` |
| Multipart upload abandoned (job cut off mid-installer) | no object, no error | `uploadCount` > 0 in the storage dataset for over a day; `smoke` #8 HEAD 404/short | `usage`, `smoke` |
| A crawler walking directory URLs (all 404 on R2) | every 404 is a Class B miss; nothing is wrong | Class B last 24 h; `edgeResponseStatus` 404 share if the field exists | `usage` |
| Analytics token loses its scope, `usage` returns 403 | designed not to fail the run | `health` down with `usage: 403`; blank budget rows | `health` |
| healthchecks itself unreachable | `ping` retries then fails the run | the Actions failure email (section 9 keeps it as the second channel for exactly this) | GitHub |
| `sync` check left paused after the seed | pings are recorded but nothing alerts | it resumes on the first ping unless "stay paused" was ticked; monthly checklist item | manual |
| Job summary over 1 MiB (a list crept in) | annotation only, step passes | the annotation is visible on the run page; the rule "counts, never lists" | `report` |
| TL signing subkey lapses (2027-07-13) or upstream rotates the key | not silent: `verify` fails and the mirror stays a day stale | the failure email; the monthly checklist 30 days before | `verify` |

## 8. Budget alerting

**Status: deferred with `usage` (section 2.4).** Until then there is no automated budget
alert of any kind; the R2 metrics tab and the bill are read monthly (section 10). This
section is the specification. One rule holds now and later: **no run ever fails on Class B**,
or on any other analytics number.

The Free plan has no usage notifications: Cloudflare's "Usage Based Billing" notification is
"Professional plans or higher" and "available to Pay-as-you-go accounts only"; Cloudflare
Health Checks are 0 on Free. So the budget alert is `usage` plus the `health` check.

**Why breaches signal `health` instead of failing the run.** Class B is caused by readers,
not by the pipeline; failing the run cannot reduce it and makes the mirror stale on top of the
bill. Class A over budget is almost always the pipeline (a re-seed, a loop), and stopping
runs does stop it, but the same `health` alert reaches the human within the hour and the next
hourly run is at most ~25 `PutObject`. Storage past the ceiling is prevented before upload by
`guard` on the listing (that one does fail the run, because the run is about to cause it);
the post-hoc storage number from analytics goes to `health`. The net effect: one email at the
transition, the mirror keeps running, the human decides.

**Thresholds.** Baselines from `cost-estimates.md` and the base plan: ~34k Class A a month
(hourly runs plus the daily reconcile's 497 listings), ~720 state reads and negligible Class B
from the pipeline; 133 GB stored, ceiling 175 GB; a full seed is ~500k Class A in its month.

| Signal | Window | Note (`::warning::`) | `health` down | Basis |
|---|---|---|---|---|
| Storage, max(payload+metadata) | last 24 h | > 150 GB | > 175 GB | ceiling; 132.99 GB today (`cost-estimates.md`) |
| Class A | month-to-date | > 500,000 | > 900,000 | 1M free; a seed month sits at the note line by design |
| Class A | last 24 h | > 50,000 | > 250,000 | normal day ≈ 24 × 25 + 497 + 24 ≈ 1,100; a re-seed is 496k |
| Class B | month-to-date | > 5,000,000 | > 8,000,000 | 10M free; the base plan's number kept |
| Class B | last 24 h | > 300,000 | > 1,000,000 | 333k/day is the free-tier pace; 1M/day is a $7/month pace and the "cache is off" signature |
| Hit ratio, cacheable paths (`CACHE=on` only) | last 24 h, only when requests ≥ 1,000 | < 70 % | < 40 % on two consecutive runs | below 1,000 requests the ratio is noise; with `CACHE=off` the row is skipped |
| Pending multipart uploads | latest snapshot | > 0 for > 24 h | never | parts are invisible otherwise |
| "other" operation names | month-to-date | any | never | tells us the `actionType` vocabulary changed |

During the seed, pause `health` alongside `sync` (section 3), or accept one email when Class A
crosses 500k and one when it recovers.

**Windows and the month boundary.** `mtd` starts at `$(date -u +%Y-%m-01T00:00:00Z)`; `day`
starts 25 hours before `now`; both end at `now − 1 h` to leave the lagging hour out. On the
1st, month-to-date is a few hours of data and cannot breach; the 24-hour windows span the
boundary and keep working; storage is a level, not a sum, so it has no boundary. R2 retention
of 31 days always covers a full month. A mid-month re-seed shows first in the 24-hour Class A
row (the same day) and in month-to-date a few days later, which is the point of having both.

**False positives.** Three sources. Sampling: adaptive datasets return estimates when volume
is high; `usage` prints `sampleInterval` and, once the field is confirmed, the 95 %
confidence interval, and the note thresholds are 2 to 5 times the expected values so a 10 %
estimation error cannot cross them. Lag: the last hour is excluded; a longer, undocumented
lag would make the month-to-date row *low*, never high, so it cannot cause a false alarm, only
a late one. Attribution: the operations dataset counts every operation on the bucket,
including the pipeline's own and any Cloudflare-internal ones; if the first run shows
operation names outside the three lists, they appear as "other" until classified. The one
real risk of a false negative is the reverse of attribution: if custom-domain cache misses do
*not* appear as `GetObject` (unverified, section 2.4), the Class B rows understate the bill and
the hit-ratio row is the only cache alarm; the test in section 2.4 settles it in an hour.

## 9. Actions failure email versus healthchecks

Facts (GitHub docs, fetched today): the Actions notification setting is account-wide, not per
repository; it can be set to "only when a workflow run has failed"; for scheduled workflows
the email goes to the user who created the workflow, last edited the cron line, or re-enabled
it. Every failed run emails. On an hourly schedule a remote that is down for a day is 24
emails, each identical except the run number.

healthchecks with `/fail`: one email when `sync` goes down (the first failure, immediately,
with the run URL and the step reached in the body), one when it comes back up, and optional
hourly or daily reminders while down. A dante outage of a day is two emails, or ~26 with the
hourly reminder on.

| | Actions email | healthchecks |
|---|---|---|
| Emails for a day-long outage | 24 | 2 (plus optional reminders) |
| Link to the log | direct | via the run URL in the body |
| Catches a run that never starts | no | yes (late, then down) |
| Catches a hung run | at timeout, as a failure | at grace, same |
| Works when healthchecks is down | yes | no |
| Works when GitHub email is off | no | yes |
| Configurable per repository | no | yes |

**Recommendation.** Make healthchecks the primary channel (the `/fail` step, the `sync` and
`reconcile` checks, and the daily reminder on), and keep GitHub's Actions email set to "only failed"
rather than off, because it is the only channel that fires when healthchecks itself is
unreachable from the runner (`ping` retries, then fails the run). The cost is the 24-email day
when dante is down. If that ever happens twice, turn the GitHub setting off; the pipeline
keeps working either way, since `ping` fails the run on a healthchecks outage and that
failure is visible on the Actions page and the README badge even without email. This is the
base plan's open question answered: route through healthchecks, keep GitHub as backstop.

## 10. Dashboard

Nothing to build. Four pages and one file, with what to look at and how often.

**Weekly (five minutes).**

| Look at | Where | For |
|---|---|---|
| Latest job summary | `https://github.com/jshvn/ctan/actions/workflows/sync.yml` → newest run | every row green-ish: read-back, mirmon, operations against the free lines, days since commit |
| The checks page | `https://healthchecks.io/checks/` (login; per-check pages hang off it) | `sync` and `reconcile` up (and `health`, once it exists); run durations trending flat; any paused check |
| Our mirmon row | `https://ctan.org/mirrors/mirmon` | a green block, age in hours, "ok" |
| Last 7 days of runs | `curl -s https://ctan.ijosh.com/.state/history.csv \| tail -168 \| column -t` | uploads roughly tracking `changed`; `age` staying under an hour |
| Freshness, from anywhere | `curl -sI https://ctan.ijosh.com/timestamp` and read `last-modified` (today the tlnet-only mirror answers 404 here; `tlpkg/texlive.tlpdb.sha512` shows `last-modified: Wed, 26 Aug 2026 18:07:24 GMT`) | under two hours old |

**Monthly (fifteen minutes).**

| Look at | Where | For |
|---|---|---|
| R2 bucket metrics | `https://dash.cloudflare.com/?to=/:account/r2/overview` → bucket → **Metrics** tab (defaults to 24 h; widen to 30 d) | storage curve flat at ~133 GB; Class A under 1M and Class B under 10M for the month. While `usage` is deferred this glance *is* the budget check, so it is not optional |
| The bill | the account's Billing page in the dashboard (`https://dash.cloudflare.com/?to=/:account/billing`, path unverified) | R2 line ≈ $1.84; nothing else non-zero |
| Zone traffic | `https://dash.cloudflare.com/?to=/:account/:zone/analytics/traffic` (Free plan: requests, bandwidth, unique visitors; no cache-status filter; Cache Analytics is Pro+) | order of magnitude of requests; a step change means a new consumer |
| Tiered Cache (`CACHE=on` only) | `https://dash.cloudflare.com/?to=/:account/:zone/caching/tiered-cache` | Smart Tiered Cache still enabled (the `rules` step re-enables it hourly, so this is a glance); with `CACHE=off`, that no cache rule exists on the zone |
| healthchecks monthly report email | arrives on the 1st | downtimes and durations for the three checks |
| CTAN.sites | `https://ctan.org/tex-archive/CTAN.sites` | our entry present with the right URL |
| Dependabot PRs | the repository's pull requests | merged, so the 60-day clock keeps resetting |
| Expiries | `CF_API_TOKEN` (no TTL, or its date); TL subkey 2027-07-13 | 30 days' notice |

## 11. Where this differs from the base plan

- **Scope adopted by the set.** The GraphQL `usage` step and the `health` check are deferred
  (they need `jq`); the base plan's "ops budget in `report`" does not exist yet, and until
  it does the R2 metrics tab is the budget check. The edge cache is off by default
  (`CACHE: off`), so the base plan's purge and `cf-cache-status` guards are conditional on
  `CACHE=on`; with it off, `smoke` proves correctness by `cmp` plus `Content-Length` and
  asserts that nothing is `HIT`. `.state/` and `.site/` are excluded from every sampled key.
- **No `curl | cmp`.** Every read-back downloads to a file under `RUN/` and compares the
  file, including today's tlpdb check, because a retried `curl` after partial output would
  corrupt a pipe (`errors-and-issues.md`).
- **Checks.** The base plan proposed one hourly check with a 2-hour grace and floated a
  second one pinged by `smoke`. This file has two now, `sync` (with `/start` and `/fail`) and
  `reconcile`, and a third, `health`, deferred with `usage`. The smoke-only check is dropped
  as redundant: a failed `smoke` fails the run and `sync` already reports that. `reconcile`
  and `health` observe things a green `sync` cannot.
- **Budget breaches never fail the run.** The base plan had `report` fail the run at 8M
  Class B or the storage ceiling. Failing the run cannot reduce Class B and stales the
  mirror; here the pre-upload `guard` keeps failing the run for storage, and every other
  breach signals `health` once it exists, and nothing until then. The base plan's 8M number
  is kept as the `health` line and a 24-hour row is added, which catches a broken cache the
  same day instead of weeks later.
- **History.** The base plan used the healthchecks log (100 entries) as the age history.
  With `/start` that is about two days; the long record is `.state/history.csv` in the
  bucket, two operations a run.
- **Token scopes.** The zone HTTP dataset needs `Zone → Analytics → Read` in addition to the
  base plan's `Account → Account Analytics → Read`.
- **`cacheStatus` is unverified.** The base plan stated the field as fact; no documentation
  page fetched today names it on `httpRequestsAdaptiveGroups`. Section 2.4 gives the
  introspection calls that confirm or refute it before the query enters the Taskfile. On a
  Free zone the dashboard offers no cache-status view at all (Cache Analytics is Pro+), so
  GraphQL is the only route.
- **`actionType` vocabulary is unverified** and handled by an "other" bucket rather than
  assumed.
- **Mirmon's probe target.** The base plan said the probe was "observed via
  `https://mirror.ctan.org/timestamp`"; that URL is the redirector (307 to
  `mirror.clarkson.edu` today), and mirmon probes each mirror's own `/timestamp`. The
  first-word-is-an-epoch format is stated by the manual, not observed today.
- **GraphQL limit.** The base plan cited 300 per 5 minutes; the GraphQL limits page still
  says 300 and the general API limits page says "Max 320/5 min". Both are far above three an
  hour.
- **Schedule disable.** Added a direct measure (`days since last commit`) with a warning at
  45; the base plan relied on healthchecks catching the outage afterwards.
- **`smoke`.** Adds the `Age` rule on the first GET (a `HIT` is not by itself proof the purge
  failed), the 404 for a deleted key, and HEADs on the five multipart objects with a
  `content-length` equality. Notes that sampled keys must be copied aside before `staging/`
  is emptied per batch, which the base plan's "three keys from the upload list" glossed over.
- **Tools and workflow.** The workflow gains an `if: failure()` step for `/fail`, a named
  constraint change. `jq` is *not* added; that is what defers `usage`.
- **Numbers.** Stored set 496,149 objects, 132.99 GB and 268 KB average object, per
  `cost-estimates.md` (the base plan's 496,155 / 133.01 GB / 244 KB; this file's own raw
  count of the listing gives the base plan's object figure, six higher, and
  `cost-estimates.md` is authoritative). Hour-slots with at least one change in the last 30
  days: 275 of 720 (base plan: 283; the difference is the exclusion of the four generated
  root files). Longest gap with no change in 365 days: 86 h; this is why no "upstream quiet"
  alert exists.
- **Usage notifications.** The base plan said "Pro and above"; the docs add "Pay-as-you-go
  accounts only". Same conclusion.

## 12. Open questions

- Is `cacheStatus` (and `edgeResponseStatus`) a dimension of `httpRequestsAdaptiveGroups` on a
  Free zone, and what are that node's `notOlderThan` and `maxDuration` there? One Settings
  query and one introspection call (section 2.4).
- What strings does `r2OperationsAdaptiveGroups.actionType` carry, and do cache-miss reads
  through the custom domain appear as `GetObject`? The first `usage` run and the 50-fetch test.
- How far does the analytics lag run for these datasets? Compare `usage`'s last-24h Class A
  with `publish`'s own `upload:` count over a week.
- Does CTAN's `timestamp` start with an epoch? `curl -L https://mirror.ctan.org/timestamp`.
- Does "renewed" in mirmon's last-probe column mean "probed in the latest cycle"?
- Do abandoned multipart parts count toward billed storage? The pricing page does not say;
  `uploadCount` > 0 is reported either way.
- Does GitHub keep workflow-run metadata (conclusion, timestamps) beyond the 90-day log
  retention? Only matters if `history.csv` is rejected.
- Whether the owner wants GitHub's account-wide Actions email off, given other repositories.
- Whether `health`, once it exists, should be paused during a re-seed or the two seed-month
  emails accepted.
- What un-defers `usage`: adding `jq` to the tools line is the only cost; the queries and
  thresholds here are ready.
- For `caching.md`: an Edge TTL no longer than a day, so that a purge missed at one PoP is
  bounded by mirmon's "oldish" line; and `.state/` excluded from the rule.
- For `sync-with-dante.md`: `.state/` excluded from the reconcile's "unknown key" deletion,
  and a `guard` rule that a delta over 20 % of an existing state file fails before fetch.
- For `taskfile-architecture.md`: the `RUN/` files this file reads (`runline.txt`,
  `usage.txt`, `breaches.txt`, `smoke.txt`, `mirmon.txt`, `sample/`), and whether `usage`
  and `health` are separate tasks or folded into `report` and `ping`.

## 13. Sources

Fetched 2026-08-26/27.

- R2 metrics and analytics (datasets, fields, 31-day retention, example queries):
  https://developers.cloudflare.com/r2/platform/metrics-analytics/
- R2 pricing (Class A/B lists, free tier, GB-month averaging, rounding, 401 not charged):
  https://developers.cloudflare.com/r2/pricing/
- R2 limits: https://developers.cloudflare.com/r2/platform/limits/
- GraphQL Analytics API index (endpoint; "not a measure for billing"):
  https://developers.cloudflare.com/analytics/graphql-api/
- GraphQL limits (300 per 5 min; node limits): https://developers.cloudflare.com/analytics/graphql-api/limits/
- GraphQL sampling: https://developers.cloudflare.com/analytics/graphql-api/sampling/ and
  https://developers.cloudflare.com/analytics/sampling/
- Confidence intervals and `sampleInterval`: https://developers.cloudflare.com/analytics/graphql-api/features/confidence-intervals/
- Settings node: https://developers.cloudflare.com/analytics/graphql-api/features/discovery/settings/
- Introspection: https://developers.cloudflare.com/analytics/graphql-api/features/discovery/introspection/
- Filtering (`requestSource: eyeball`, operators): https://developers.cloudflare.com/analytics/graphql-api/features/filtering/
- Error responses (403 → Analytics: Read): https://developers.cloudflare.com/analytics/graphql-api/errors/
- Analytics API token (Account Analytics Read): https://developers.cloudflare.com/analytics/graphql-api/getting-started/authentication/api-token-auth/
- Execute a query with curl: https://developers.cloudflare.com/analytics/graphql-api/getting-started/execute-graphql-query/
- HTTP events by hostname (`httpRequestsAdaptiveGroups`, `clientRequestHTTPHost`):
  https://developers.cloudflare.com/analytics/graphql-api/tutorials/end-customer-analytics/
- Colo groups migration (`edgeResponseBytes`, `sampleInterval`):
  https://developers.cloudflare.com/analytics/graphql-api/migration-guides/graphql-api-analytics/
- Zone Analytics migration (`cachedRequests` on `httpRequests1mGroups`):
  https://developers.cloudflare.com/analytics/graphql-api/migration-guides/zone-analytics/
- Zone analytics dashboard, Free plan panels: https://developers.cloudflare.com/analytics/account-and-zone-analytics/zone-analytics/
- Analytics FAQ (24-hour delay on free): https://developers.cloudflare.com/analytics/faq/about-analytics/
- API token permissions (Analytics Read, Account Analytics Read): https://developers.cloudflare.com/fundamentals/api/reference/permissions/
- API rate limits (1,200/5 min; GraphQL 320/5 min): https://developers.cloudflare.com/fundamentals/api/reference/limits/
- Cache responses (`cf-cache-status` values, `Age` semantics): https://developers.cloudflare.com/cache/concepts/cache-responses/
- Default cache behavior (404 TTL 3 min; 512 MB): https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- Purge by single file ("across all data centers", result `MISS`): https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/
- Tiered Cache (Smart: single upper tier; Free availability): https://developers.cloudflare.com/cache/how-to/tiered-cache/
- Cache Analytics availability (Free: no): https://developers.cloudflare.com/cache/performance-review/cache-analytics/
- Cache with R2 custom domains: https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/
- Notifications available (usage-based billing: Pro+, PAYG): https://developers.cloudflare.com/notifications/notification-available/
- Health Checks availability (Free: 0): https://developers.cloudflare.com/health-checks/
- How charges accrue (cache hits avoid R2 operations): https://developers.cloudflare.com/billing/understand/how-charges-accrue/
- healthchecks.io Pinging API: https://healthchecks.io/docs/http_api/
- healthchecks.io configuring checks (period, grace, cron, paused-check behaviour):
  https://healthchecks.io/docs/configuring_checks/
- healthchecks.io signalling failures: https://healthchecks.io/docs/signaling_failures/
- healthchecks.io attaching logs (100 kB, `/log`): https://healthchecks.io/docs/attaching_logs/
- healthchecks.io measuring run time (`/start`, 72 h): https://healthchecks.io/docs/measuring_script_run_time/
- healthchecks.io notifications (reminders, reports): https://healthchecks.io/docs/configuring_notifications/
- healthchecks.io badges: https://healthchecks.io/docs/badges/
- healthchecks.io Management API (`/pause`): https://healthchecks.io/docs/api/
- healthchecks.io GitHub Actions guide: https://healthchecks.io/docs/github_actions/
- healthchecks.io pricing: https://healthchecks.io/pricing/
- GitHub workflow commands (job summary 1 MiB, 20 per job; annotations):
  https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions
- GitHub schedule event (delays, dropped jobs, 60 days):
  https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
- GitHub notifications for workflow runs: https://docs.github.com/en/actions/concepts/workflows-and-actions/notifications-for-workflow-runs
- GitHub notification settings (account-wide Actions toggle):
  https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications
- GitHub artifact and log retention (90 days; public 1 to 90):
  https://docs.github.com/en/organizations/managing-organization-settings/configuring-the-retention-period-for-github-actions-artifacts-and-logs-in-your-organization
- GitHub Actions limits: https://docs.github.com/en/actions/reference/limits
- GitHub REST workflow runs (`per_page` max 100): https://docs.github.com/en/rest/actions/workflow-runs
- `gh run list` (JSON fields, `-L`): https://cli.github.com/manual/gh_run_list
- Runner image contents (jq 1.7, GitHub CLI 2.97.0, AWS CLI 2.36.24, rsync 3.2.7, xz 5.4.5, gnupg 2.4.4):
  https://raw.githubusercontent.com/actions/runner-images/main/images/ubuntu/Ubuntu2404-Readme.md
- CTAN mirror status (mirmon page and legend): https://ctan.org/mirrors/mirmon
- mirmon manual: https://www.mankier.com/1/mirmon
- Becoming a CTAN mirror (monitoring, removal, hourly sync): https://ctan.org/mirrors/register
- CTAN.sites: https://ctan.org/tex-archive/CTAN.sites
- Live probes: `curl -sI https://ctan.ijosh.com/...` (2026-08-27 00:15 UTC) and
  `curl -sS -D - https://mirror.ctan.org/timestamp` (307 to `mirror.clarkson.edu`)
- Listing: `rsync -rL --list-only rsync://rsync.dante.ctan.org/CTAN/` taken 2026-08-26 ~17:00 UTC
  (`ctan-list-deref.txt`), for the stored-set size, alias directory sizes, `timestamp` size and
  mtime, and the change-hour analysis
