# Reference

The verified numbers behind the mirror: what the tree measures, what the platforms allow,
what it costs, how it is watched, and what to do when a run fails. Every figure carries the
date it was verified. Re-verify before leaning on a stale one; the commands that produced
the measured figures are named beside them.

Conventions: GB is 10^9 bytes, GiB and MiB are binary. R2 publishes binary figures, CTAN
decimal. "Stored set" is what `rsync -rL` yields minus tlnet's versioned containers and the
`update-tlmgr-r*` root files, which is what `normalise` drops.

## 1. Baseline

Measured 2026-08-26 from a dereferenced dante listing (538,289 lines, 6.9 s wall clock,
50.7 MB).

| Quantity | Value |
|---|---|
| Stored set | 496,149 objects, 132.99 GB (123.86 GiB) |
| Largest object | 6,865,013,189 B (`systems/mac/mactex/MacTeX.pkg`) |
| Longest key | 151 bytes |
| Distinct directories | 24,953 |
| Churn, last 30 days | 16,574 files, 4.72 GB; 23.0 files per hour on average |
| Hour-slots with any change, last 30 days | 284 of 720 |
| Hour-slots over 1,000 files, last 30 days | 3 |
| Busiest hour by count, last 365 days | 5,950 files, 0.037 GB (2026-01-12 16h) |
| Busiest hour by bytes, last 365 days | 20.35 GB in 9 files (2026-03-01 09h, ISO copies) |
| Busiest hour by bytes, installers excluded | under 0.001 GB |
| Objects over R2's 4.995 GiB single-part limit | 5: two MacTeX pkgs at 6.87 GB, three ISOs at 6.78 GB |
| Objects over Cloudflare's 512 MB cacheable limit | 7: the five above plus two `protext` zips at 1.14 GB |
| Whole tree at `BATCH_GB=4` | 25 batches plus 5 lone oversize files = 30 rsync connections; largest batch 97,569 files |

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
| Objects, storage per bucket | unlimited | 496k, 133 GB | billing only |
| Object size | 5 TiB (4.995 TiB in practice) | 6.87 GB max | reject |
| Single-part upload | 4.995 GiB = 5,363,466,240 B | 5 objects exceed it, so go multipart | `EntityTooLarge`, run fails |
| Multipart parts | 10,000, each 5 MiB–5 GiB, uniform | 13 parts per 6.87 GB file at 512 MiB | reject |
| Incomplete multipart uploads | aborted after 7 days | at most one after a cut-off job | storage held up to 7 days |
| Key length | 1,024 bytes | 151 max | reject |
| Writes per key | 1 per second | one state write per batch, minutes apart | 429 |
| `ListObjectsV2` page | 1,000 keys | 497 pages per reconcile | — |
| `DeleteObjects` batch | 1,000 keys | one call in a typical hour | `MalformedXML` |
| Class A operations | 1M/month free, then $4.50/M | ~26 an hour | bill |
| Class B operations | 10M/month free, then $0.36/M | 1 a run plus user traffic | bill |
| Custom domains per bucket | 100 | 1 | — |

`aws.config` sets `multipart_threshold = 4GB`, which the AWS CLI reads as binary — 4 GiB,
safely under the 4.995 GiB single-part ceiling — and `multipart_chunksize = 512MB`, giving
13 uniform parts for the largest file.

### Cloudflare zone, Free plan

| Limit | Value | What the mirror asks |
|---|---|---|
| Cache Rules | 10 | 1 to 2 |
| Transform Rules | 10, no regex on Free | 1 |
| Configuration Rules | 10 | 1, disabling the HTML rewriters on the mirror host |
| Cacheable object size | 512 MB | 7 objects over it, served from R2 every time |
| Purge by URL | 100 per call, 800 URLs/s per account | 0 while `CACHE=off` |
| Purge everything | 5 per minute | unused |
| API requests per token | 1,200 per 5 minutes | 3 to 6 a run |
| API requests per IP | 200/s | under 1/s |
| GraphQL | 300 per 5 minutes | 1 to 2 |
| Upload through the zone | 100 MB | 0; uploads go to the S3 endpoint |

### GitHub Actions

| Limit | Value | What the mirror asks |
|---|---|---|
| Job time | 6 h, counted from the actual start | `timeout-minutes: 350` |
| Runner disk | 14 GB documented | at most one 4 GB batch plus the 6.87 GB outlier |
| Runner RAM | 16 GB | under 300 MB |
| Concurrent jobs | 20 on Free | 1; `concurrency: sync` queues an overlapping slot |
| Job summary | 1 MiB per step, 20 steps | about 2 KB |
| Cron lateness | starts 15 to 45 min after the slot; slots can be dropped | absorbed by the healthchecks grace |
| Schedule disablement | 60 days without repository activity | a failed healthcheck ping catches it |

Log volume is undocumented and lines are silently dropped, which is why `report` counts from
`RUN/` and never from the log.

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
| `sort` memory | 68 MB for 496k lines |
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
| Storage today | 133 GB-month, minus 10 free, 123 × $0.015 = **$1.85/month** |
| At the 200 GB ceiling | 190 billable = $2.85/month |
| Class A per month | 30 reconcile listings + 720 state writes + churn ≈ 32,120 → $0 |
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

## 5. Runbook

Local commands need the four R2 variables in the environment as `sync.yml` maps them,
`AWS_CONFIG_FILE=aws.config`, and `CF_API_TOKEN` plus `CF_ZONE_ID` for the Cloudflare calls.
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
correctness, not for the budget: the delta is the whole tree, about 496k Class A and 133 GB
from dante. Pause the healthcheck first.

**Delete or purge one key.**

```sh
aws s3 rm s3://ctan/<key>
curl -fsS -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H 'Content-Type: application/json' \
  --data '{"files":["https://ctan.ijosh.com/<key>"]}'
```

If upstream still has the key, the daily reconcile finds it missing and the next hourly run
re-fetches it. Editing the state by hand to get it back sooner is not worth the risk.

**A ruleset was not applied.** `cloudflare:set` prints a warning naming the phase and moves on when
the phase holds rules this repo did not write; the run still succeeds. It PUTs the whole
phase at once, so applying it would delete them.

```sh
task cloudflare:get     # what each phase holds now, and whether its description carries our stamp
```

Fold anything worth keeping into that phase's file in `cloudflare/` — a ruleset's `rules`
array takes as many entries as the plan allows — or clear the phase in the dashboard. The
next run applies the file and stamps it, and from then on the phase is ours to rewrite.

**Rotate a secret.**

```sh
gh secret set AWS_ACCESS_KEY_ID
gh secret set AWS_SECRET_ACCESS_KEY
gh workflow run sync.yml && gh run watch
```

Delete the old token in the Cloudflare dashboard after a green run. Safe: the first call
with new credentials is a read.

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
