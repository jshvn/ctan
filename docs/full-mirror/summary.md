# Full CTAN mirror: summary and decisions

Entry point to the ten specification files in this directory. Section 3 states every
non-obvious design decision with its reason and the file that owns it; the owning file has
the commands, the sources and the arithmetic. Every number here comes from one of those
files.

## 1. The decision

Turn the tlnet-only mirror into a full CTAN mirror on the same R2 bucket, served at
`https://ctan.ijosh.com/` with every CTAN path verbatim at the host root, and register it
as an official mirror. The bucket holds 496,149 objects and 132.99 GB (every symlink a
copy; tlnet's versioned containers excluded) for $1.86 a month in storage and $0 in
operations. The pipeline stays one `Taskfile.yml` run hourly by GitHub Actions: it lists
dante (7 s), diffs the listing against a 3 MB state file kept in the bucket, fetches only
the delta in batches of at most 4 GB into the 14 GB runner, verifies the signed tlnet files
against the pinned TeX Live key, uploads with `aws s3 cp`, and checkpoints the state after
every batch, so any run can die anywhere and the next hour continues. GitHub starts
scheduled runs 15 to 45 minutes late and sometimes drops a slot; the design absorbs that
rather than fighting it. Seeding is seven ordinary pull requests and one day of runs
(126 GB from dante, ~513k Class A, $0.98 prorated storage). The edge cache stays off: the
bill cannot exceed $0.36 per million GETs and no traffic this mirror can draw reaches a
dollar. Two things could stop it: Cloudflare's CDN terms on serving large files from a
Free zone (a reading of two documents, not a number; ask support before registering), and
CTAN declining a mirror with no directory listings and no rsync (neither is a stated
requirement; ask in the registration notes).

## 2. Headline numbers

| Quantity | Value | Owner |
|---|---|---|
| Stored set | 496,149 objects, 132.99 GB (123.9 GiB); 5 objects over 4.995 GiB | [taskfile-architecture.md](taskfile-architecture.md), [seeding-and-migration.md](seeding-and-migration.md) |
| Monthly cost | $1.86 storage (134 GB-month rounded, minus 10 free); operations $0 | [cost-estimates.md](cost-estimates.md#fixed-monthly-cost) |
| Seed cost | ~513k Class A (free tier 1M); storage $0.98 if seeded on the 15th | [cost-estimates.md](cost-estimates.md#seed-month) |
| Seed duration | 67 to 274 min of runner time; one dispatch of 1.5 to 5 h, or 8 hourly runs at `MAX_BATCHES=4` (about 9 to 11 h wall clock, plus 20 to 40 min of cron lateness per run) | [seeding-and-migration.md](seeding-and-migration.md#the-seed) |
| Hourly run | ~30 s quiet, ~50 s at P99, ~2 min on the busiest file hour; a release day is three lone 6.8 GB batches of ~20 min each | [sync-with-dante.md](sync-with-dante.md#8-time-budget-of-an-hourly-run) |
| Cron lateness | Scheduled runs start 15 to 45 min after the cron minute (one run in this repository started 39 min late; today's was 18+ min late); a slot can be dropped outright; the run never starts at its minute | [limits.md](limits.md), [monitoring.md](monitoring.md) |
| Thinnest limit margin | 6-hour job during the seed (resumable by design); seed month uses ~52% of free Class A; everything hourly is under 3% of any limit | [limits.md](limits.md#8-consolidated-table) |
| Storage ceiling | 175 GB and 600,000 objects, checked on the upstream listing before any fetch | [cost-estimates.md](cost-estimates.md#budget), [verification-and-security.md](verification-and-security.md#4-the-symlink-inflation-guard-and-the-storage-ceiling) |
| Secrets | 6: the four today plus `CF_API_TOKEN`, `CF_ZONE_ID` | [caching.md](caching.md#token-and-secrets) |
| Endpoints | 6: dante, R2, `ctan.ijosh.com`, `hc-ping.com`, `api.cloudflare.com`, `ctan.org` (mirmon page) | [monitoring.md](monitoring.md#6-mirmon) |

## 3. Design decisions

Each stated on its own terms, with the reason and the file that carries the evidence.

### Synchronising

- The mirror is a list-diff, never a local tree: one `rsync --list-only` of dante, one
  `comm` against the listing the last run left in the bucket, one `rsync --files-from` per
  batch, one `aws s3 cp --recursive` per batch. The runner has 14 GB and the tree is 133 GB;
  the bucket is never listed in the hourly path. [sync-with-dante.md](sync-with-dante.md)
- The state line is `path TAB size TAB mtime`, path first, `LC_ALL=C` sorted, because
  whole-line `comm` mis-pairs lines otherwise and one sort then serves both the change diff
  and the path-only deletion diff. [taskfile-architecture.md](taskfile-architecture.md#why-path-first)
- The change key is size plus mtime to the second, rsync's own quick check; a same-size
  same-second rewrite is invisible here as on every rsync mirror. [sync-with-dante.md](sync-with-dante.md#why-sizemtime-and-where-it-fails)
- Every rsync call runs with `TZ=UTC` and `--no-h`, and the listing is parsed by anchoring
  on the date column: rsync prints mtimes in the client's zone, GNU rsync prints digit
  separators, openrsync prints blank sizes for zero-byte files. [sync-with-dante.md](sync-with-dante.md#line-format), [limits.md](limits.md#9-local-runs-on-the-developers-mac)
- The batch fetch is `rsync -Lt --files-from --ignore-missing-args` with no `-r`: `-r`
  would pull an unplanned subtree when a listed path has become a directory, and a path that
  vanished between listing and fetch is skipped and left out of the state, which records
  only what landed. The listing accepts exit 23 (a dangling symlink upstream); a fetch exit
  23 is a real error. [sync-with-dante.md](sync-with-dante.md#the-flags-and-why), [errors-and-issues.md](errors-and-issues.md)
- Batches hold at most 4 GB by listing size, in key order; a file over the cap is alone;
  the decision batch (root files, tlnet root files, `tlpkg/`, then `timestamp` last of all)
  is always the final batch, so a tlpdb never names a container the bucket lacks and
  `timestamp` never claims an hour whose files are missing. [seeding-and-migration.md](seeding-and-migration.md#seed-ordering-within-the-tree), [sync-with-dante.md](sync-with-dante.md#5-the-consistency-model)
- The state is written once per batch, after that batch's upload succeeded, as one
  `PutObject`. Every write is idempotent, so a run that dies anywhere is at-least-once and
  the next hour repeats at most one batch. [sync-with-dante.md](sync-with-dante.md#5-the-consistency-model)
- A missing state file fails the run unless `SEED=true`, and a corrupt one fails closed,
  because treating either as "empty bucket" would silently re-upload 133 GB. Recovery from
  a lost state is `RECONCILE=true`, which rebuilds it from the bucket listing joined to
  upstream for 497 Class A. [taskfile-architecture.md](taskfile-architecture.md), [errors-and-issues.md](errors-and-issues.md#1-failure-catalogue)
- A delta over 10% of the state fails the run unless `FORCE=1`: that is the shape of dante
  restored from backup or a truncated listing with exit 0, and it should be a decision,
  never a refetch. [errors-and-issues.md](errors-and-issues.md#diff-comm-between-the-sorted-listing-and-the-state)
- Deletions come from the diff and run after every batch, 1,000 keys per `DeleteObjects`,
  with the per-key `Errors` array parsed because the CLI exits 0 on it. The bucket is listed
  once a day in `reconcile`, and every bucket listing is re-sorted with `LC_ALL=C` because
  R2 lists keys out of byte order. [taskfile-architecture.md](taskfile-architecture.md), [errors-and-issues.md](errors-and-issues.md)

### Verifying

- The tlnet control files are fetched into `RUN/` and verified (sha512, `gpgv` with
  `GOODSIG` and the pinned `VALIDSIG`, `.xz` match) at the start of any run whose delta
  touches tlnet; every batch's containers are checked against that tlpdb; `tlpkg/` is
  uploaded last and must `cmp` equal to the verified copy. Containers land in earlier
  batches than the tlpdb, so no other tlpdb is ever the right reference. [verification-and-security.md](verification-and-security.md#2-the-delta-scoped-tlnet-verification)
- "Every named container exists" is checked just before the decision batch against the
  bucket as it will be (state plus this run's delta minus this run's deletions); the promise
  is about what a client will fetch. A missing container fails the run. A one-GetObject diff
  of the old tlpdb's checksums catches a container whose bytes changed upstream with the
  same size and second. [verification-and-security.md](verification-and-security.md#the-five-checks-restated-for-a-delta)
- tlcontrib's tlpdb is verified with a second pin and the TeX Live ISO checksums are
  signature-checked and hashed when an ISO is in the delta; everything else on CTAN carries
  no pinnable signature and is copied as served, and `SECURITY.md` says exactly which is
  which. [verification-and-security.md](verification-and-security.md#every-index-a-client-trusts)
- The storage guard runs on the upstream listing before any fetch: 175 GB and 600,000
  objects, `timestamp` present, at least 90% of the state's line count. A release day adds
  21 GB and stays under it. [verification-and-security.md](verification-and-security.md#4-the-symlink-inflation-guard-and-the-storage-ceiling), [cost-estimates.md](cost-estimates.md#budget)
- tlnet's and tlcontrib's versioned containers are dropped in the normaliser. Nothing
  requests them by that name; keeping them is +$0.09 a month and a second upload per
  revision bump. [cost-estimates.md](cost-estimates.md#keeping-tlnets-versioned-containers), [taskfile-architecture.md](taskfile-architecture.md)

### Uploading

- One `aws.config` carries `multipart_threshold = 4GB` and `multipart_chunksize = 512MB`.
  The CLI's suffixes are binary, so 4 GiB sits below R2's 4.995 GiB single-part limit and
  above the largest ordinary file (`protext.zip`, 1.14 GB); 13 parts of 512 MiB is 15
  Class A per installer, five installers a year. [taskfile-architecture.md](taskfile-architecture.md#5-multipart-for-the-objects-over-4995-gib), [limits.md](limits.md#aws-cli-v2)
- `retry_mode = standard`, `max_attempts = 10`: adaptive mode is documented as experimental,
  its throttle is process-wide, and R2 publishes no per-bucket rate to adapt to. [errors-and-issues.md](errors-and-issues.md#aws-cli-v2)
- rsync has no retry, so one `retry` task wraps its two calls and retries only the transport
  exit codes 5, 10, 12, 30, 35 with jittered backoff; every `curl` line uses one `CURL`
  variable that honours `Retry-After` and never retries 401 or 403. [errors-and-issues.md](errors-and-issues.md#2-retry-semantics-per-tool)
- Operations bill in whole millions, so any Class A overage costs at least $4.50; the
  hourly design lists the bucket only in the daily reconcile to stay far from that line at
  any bucket size. [cost-estimates.md](cost-estimates.md#prices-re-verified-2026-08-26)

### The storage set and the URL

- Every symlink is stored as a copy (64 GB, $0.96 of the $1.86). Redirect rules for the
  aliases would save $0.83 and create dashboard state that must track upstream renames; a
  rule nobody remembers is a 404 nobody notices. `systems/win32` is the real directory and
  `systems/windows` the 23.64 GB alias. [official-mirror-and-url.md](official-mirror-and-url.md#4-the-storage-set-and-the-aliases)
- `.state/` and `.site/` are the two non-CTAN prefixes; every bucket listing excludes them,
  and CTAN has no dot-prefixed root entry, so they cannot collide. [official-mirror-and-url.md](official-mirror-and-url.md#10-recommendation), [seeding-and-migration.md](seeding-and-migration.md#bucket-layout-during-and-after)
- The landing page lives at `.site/index.html` behind the `/` rewrite; CTAN's own root
  `index.html` is stored at `/index.html` like any file. The page at `/` is the one thing
  that tells a human how to point `tlmgr` here, and it costs one key. [official-mirror-and-url.md](official-mirror-and-url.md#3-the-indexhtml-collision-readme-and-robotstxt)
- Directory URLs redirect with one Single Redirect to `https://ctan.org/tex-archive<path>`:
  R2 has no listings, CTAN's own page is better than an Apache index, and the rule's
  `ne "/"` clause keeps the landing-page rewrite alive. [official-mirror-and-url.md](official-mirror-and-url.md#2-directory-indexes)
- The hostname stays `ctan.ijosh.com` at the root, registered with the HTTP, FTP and rsync
  fields blank; 21 listed mirrors have the same shape. [official-mirror-and-url.md](official-mirror-and-url.md#5-the-canonical-url)

### Caching

- The edge cache is off. Uncached, the bill is $0.36 per million GETs after 10 million, free
  below ~28 `scheme-full` installs a day and single-digit dollars at the traffic of one of the
  largest mirrors on earth; a cache adds a purge step, a token scope and two correctness
  windows and bounds nothing, since evictions are undocumented and 404s are never cached.
  The trigger to switch on is Class B above 5 million for two months. [caching.md](caching.md#1-does-the-mirror-need-a-cache)
- The zone's three rules (bypass-all cache rule, the `/` rewrite, the directory redirect)
  live as JSON in `cloudflare/` and `task rules` `GET`s each phase and `PUT`s only when the
  file's stamped hash differs, because every `PUT` mints a ruleset version. Smart Tiered
  Cache is a persistent zone setting, set once by hand when the cache goes on. [caching.md](caching.md#7-configuration-as-code)
- When the cache is on: 404s are never stored (`status_code_ttl` no-store from 3xx up), so
  added keys need no purge; upload then purge, never the reverse; changed `archive/` keys are
  purged before `tlpkg/` lands; Edge TTL is one day so a purge that silently failed is
  bounded. [caching.md](caching.md#4-purge)

### Schedule, seed and monitoring

- The cron is `41 * * * *`, drawn once and kept, as CTAN asks; GitHub starts it 15 to 45
  minutes late and may drop a slot, so nothing in the design assumes the run begins at :41.
  `concurrency: sync` without `cancel-in-progress` keeps one run going and one pending, so a
  late run that overlaps the next slot queues it. `timeout-minutes` is 350 always: every
  batch checkpoints and nothing can hang for hours, so a short timeout gains nothing.
  `MAX_BATCHES=4` bounds each hourly run so it reports and pings. [taskfile-architecture.md](taskfile-architecture.md#7-the-workflows), [limits.md](limits.md#3-github-actions)
- `report` prints the run's lateness (actual start minus the cron slot) every run, so the
  drift is a number on the job page rather than a surprise. [taskfile-architecture.md](taskfile-architecture.md)
- Two healthchecks checks: `sync` (cron schedule, `/start` at the head, `/fail` from the
  workflow's failure step) and `reconcile` (period one day). The `sync` grace is 3 h: a
  dropped slot, a 45-minute late start and a 55-minute run reach 160 min, with 20 min to spare; monitoring.md
  carries the arithmetic. GitHub's failure email stays on
  "only failed" as the one channel that fires when healthchecks itself is unreachable. [monitoring.md](monitoring.md#3-healthchecksio-design)
- mirmon's 28-hour "fresh" band absorbs any lateness; an hour of drift on the `timestamp`
  copy is cosmetic. A mirmon row reading "old" on two consecutive runs fails the run, since
  it means the serving path differs from what `smoke` saw. [monitoring.md](monitoring.md#6-mirmon), [official-mirror-and-url.md](official-mirror-and-url.md#7-what-ctans-monitor-sees)
- No budget number fails a run except the pre-upload storage and object guard; Class B is
  caused by readers and a failed run cannot reduce it. The GraphQL `usage` step is deferred
  (its fields are unverified and its numbers matter only for a cache that is off), so the
  tool list stays as it is and `jq` comes with `usage` if `usage` ever comes. Month-to-date
  operations are read from the R2 dashboard's Metrics tab monthly. [monitoring.md](monitoring.md#8-budget-alerting), [caching.md](caching.md#checkyml)
- The seed is the hourly loop with a large delta, after six pull requests that each leave
  the tlnet mirror green; the list-diff PR lands first and runs a week on tlnet scope, so the
  state already holds tlnet and the seed pulls 126 GB. The dispatch path is preferred on the
  seed day because a dispatched run starts at once, where eight hourly runs each lose 20 to
  40 minutes to cron lateness. The cache PR comes after the seed. [seeding-and-migration.md](seeding-and-migration.md#the-migration-as-a-sequence-of-prs)

## 4. The recommended design

**Pipeline.** `task sync` runs, in order and nothing in parallel: `rules` (three `GET`s, a
`PUT` only when a `cloudflare/*.json` changed) → `list` (`TZ=UTC rsync -rL --list-only
--no-h`, normalised to `path TAB size TAB mtime`, tlnet and tlcontrib versioned containers
and `update-tlmgr-r*` dropped, floor of 400k lines) → `state` (fetch `.state/applied.txt.xz`;
missing fails unless `SEED=true`) → `diff` (`comm -13` for changed, `comm -23` on paths for
deleted; refetch-storm guard at 10%) → `plan` (ceiling 175 GB / 600k on the listing, `df`
check, batches of ≤4 GB in key order, installers alone, the decision batch of root files,
tlnet root files and `tlpkg/` last with `timestamp` last of all, at most `MAX_BATCHES`) →
`tlpdb` (when the delta touches tlnet: fetch the four control files into `RUN/tl`, sha512,
`gpgv` with `GOODSIG` and the `VALIDSIG` pin, `.xz` match, silent-change diff) → per batch
`fetch` (`rsync -Lt --files-from --ignore-missing-args`, through `retry`) → `verify`
(containers against `RUN/tl`; installers, ISO and tlcontrib pins when present; before the
decision batch, every named container in state + delta − deleted; `cmp` `tlpkg/` to
`RUN/tl`) → `publish` (`aws s3 cp --recursive`, `tee` under `pipefail`) → `checkpoint`
(rewrite and upload the state; empty staging) → `delete` (`delete-objects`, 1,000 per call,
errors parsed, state rewritten) → `reconcile` (hour 03 UTC or `RECONCILE=true`: bucket
listing ⋈ upstream becomes the state, extras deleted) → `smoke` (`/timestamp` and three
uploaded keys by `Content-Length`, the tlpdb sha512 by `cmp`, decision files `DYNAMIC`,
mirmon row) → `report` (counts from `RUN/`, lateness, mirmon, days since last commit) →
`ping`. The workflow runs `task fail` on failure.

**State.** One object, `.state/applied.txt.xz`: the listing lines of what the bucket holds,
path first, `LC_ALL=C` sorted, 3.1 MB, written once per batch after upload succeeds. Every
write is idempotent; a run that dies leaves the state as of the last batch and the next hour
repeats at most one batch. `.state/history.csv` gets one line per successful run.

**Storage set.** Everything `rsync -rL` yields minus the versioned containers: 496,149
objects, 132.99 GB, every alias a copy, five objects multipart. `.state/` and `.site/` are
the only non-CTAN keys.

**Cache.** Off. Three rules as code in `cloudflare/`: bypass-all cache rule, `/` →
`/.site/index.html` rewrite, directory-URL redirect to `ctan.org/tex-archive`. No purge
step, no tiered cache. Flip `CACHE: on` (two-rule cache JSON, purge of changed and deleted
keys per batch, `smoke` HIT assertions, Smart Tiered Cache) when the R2 dashboard shows
Class B above 5M for two months or users far from the bucket complain.

**Schedule.** `cron: '41 * * * *'`, `concurrency: sync`, `timeout-minutes: 350`,
`MAX_BATCHES=4`, a `workflow_dispatch` input to raise the batch count. A run starts 15 to
45 minutes after :41 and a slot may be dropped; the state model makes a late or missing run
a two-hour gap and nothing more, and mirmon calls anything under 28 hours fresh.

**Monitoring.** A failed run is the alert; healthchecks `sync` (`/start`, `/fail`, grace 3 h,
sized for a late start plus the run plus one dropped slot) and
`reconcile` (daily) are the dead man's switches; GitHub's failure email stays as backstop.
`report` prints counts, never lists; the lateness; the mirmon row; days since the last commit
with a warning at 45. Monthly: the R2 Metrics tab and the bill.

**Migration.** Seven PRs, each leaving the tlnet mirror green:

1. `fix(sync): bounded retries` — `standard` mode, `max_attempts = 10`, the `CURL` variable,
   the `retry` task, `--ignore-missing-args`.
2. `refactor(sync): list-diff against a state file, tlnet scope` — path-first state, `SEED`
   gate, `plan`, `tlpdb`, `delete`, `reconcile`; `stale` and `guard` go. Run for a week.
3. `ci(sync): hourly at :41` — `timeout-minutes: 350`, `MAX_BATCHES`, the dispatch input,
   the two healthchecks checks with their lateness-sized grace, `task fail`, lateness in
   `report`.
4. `feat(publish): one aws.config` — threshold 4 GiB, chunk 512 MiB; dead code until 6.
5. `feat(rules): cloudflare rules as code` — bypass, rewrite to `.site/`, redirect; `page`
   writes `.site/index.html`; the two secrets. Must precede 6 so the page moves before CTAN's
   `index.html` arrives.
6. `feat(sync): mirror the whole of CTAN` — `SOURCE`, bucket root, ceilings, tlcontrib and
   ISO checks, README, SECURITY.md, CLAUDE.md. Rehearse with `max_batches: 1`; seed by
   dispatch with the batch count raised, the hourly runs as the fallback. Write to
   `ctan@ctan.org` two days before.
7. `docs(mirror): register` — then the form, then watch mirmon at :03.

Later, at the trigger: `feat(cache): CACHE on` with the purge step.

**After the change.** Secrets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
`HEALTHCHECK_URL`, `CF_API_TOKEN`, `CF_ZONE_ID`. Endpoints: `rsync.dante.ctan.org`, R2,
`ctan.ijosh.com`, `hc-ping.com`, `api.cloudflare.com`, `ctan.org`. Tools unchanged.
CLAUDE.md lines that change: the opening paragraph (hourly, all of CTAN, 496k files, 133 GB,
largest 6.87 GB); the file list (`cloudflare/*.json`; `site/` becomes `.site/`); "Zero
running cost" becomes "storage is the only bill, $1.86 at 133 GB, ceiling 175 GB / 600k
objects"; "workflows run one task, and `task fail` if it failed"; endpoints and secrets as
above; the objects line names the two reserved prefixes; the must-knows become the
decisions in section 3 (versioned containers, the state line, the hourly path never listing
the bucket, the multipart exception, the decision batch, `.site/`, the 60-day rule, cron
lateness, and that a run stopping at `MAX_BATCHES` is a success).

## 5. Open questions that survive

Flagged **STOP** if the answer could end the project.

| Question | Cheapest experiment | Owner |
|---|---|---|
| **STOP** Does paid R2 usage on a Free zone satisfy the CDN terms' "Paid Services" clause for large files? | A written answer from Cloudflare support, before registering | cost-estimates, limits, official |
| **STOP** Will CTAN list a mirror with no directory listings, no rsync, directory URLs redirected to `ctan.org`, and a start that drifts 15 to 45 minutes past its minute? | Ask `ctan@ctan.org` with the pre-seed note; say it in the registration Notes | official, seeding |
| Is an off-peak cron minute less late than one near the hour, and by how much? | The minute-of-hour histogram of `gh run list` after a month; compare against another repository's minute | sync-with-dante, monitoring |
| Actual free disk on `ubuntu-latest` (14 GB documented; ~18 to 50 GB reported) | `df -h .` step in one run | limits, taskfile |
| `rsync 3.2.7 --files-from` with 97k paths, and `--ignore-missing-args` exit 0 on a vanished path | The rehearsal batch; one run with a bogus path | seeding, taskfile |
| Do concurrent `UploadPart` calls on one key trip R2's 1 write/s per key? | First multipart upload with `--debug` | limits |
| Does R2 accept the CLI's CRC64NVME trailer, and `delete-objects` with 1,000 keys? | One `--debug` upload; the throwaway-key test (runbook R10) | errors-and-issues |
| Is R2's storage meter base-10 or base-2 ($1.86 or $1.71)? Do abandoned multipart parts bill? | First month's dashboard | cost-estimates |
| Dante's `max connections`, daemon version, listing wire size | `--stats` from the runner; the MOTD | limits, sync-with-dante |
| Is the Single Redirect expression form (`ends_with`, `concat`) available on Free? Does R2 map `/dir/` to a key `dir/`? | Create the rule and trace it; one test upload | official |
| Does Cloudflare's URL normalisation pass a literal `%2f` key through? | After the seed, five keys under `systems/mac/textures/` | verification |
| CTAN's `timestamp` format versus mirmon's probe | Watch mirmon after registration | monitoring, official |
| Cache-rule API shape (`status_code_ttl`, `custom_key` on Free, entrypoint `PUT` creating the phase, edge TTL max), tiered-cache purge race, `Range` on >512 MB objects | Deferred with the cache; the first `PUT` and caching's section 10 experiment answer them | caching |
| Is `TLCONTRIB_KEY` worth a second pin that can stall the whole mirror for a 530-file side repository? | Decide at PR 6; the alternative is copy-as-is | verification |
| Do Dependabot commits count as repository activity for the 60-day rule? Does GitHub email on pending runs cancelled by concurrency? | Observe | errors-and-issues, monitoring |
| The bucket's region, for the city on the form | Dashboard | official |
| Whether the dispatch input survives the seed | Decide at PR 7 | seeding |

## 6. Reading order

1. [cost-estimates.md](cost-estimates.md): what it costs, where the money could start, and the budget lines.
2. [sync-with-dante.md](sync-with-dante.md): the list-diff, the contract with upstream, symlinks, the consistency argument, real start times.
3. [taskfile-architecture.md](taskfile-architecture.md): the Taskfile and workflows, task by task, with the data files.
4. [verification-and-security.md](verification-and-security.md): what is signed on CTAN, the delta-scoped checks, the threat model.
5. [errors-and-issues.md](errors-and-issues.md): every failure, every retry, the runbook.
6. [limits.md](limits.md): every wall and the margin to it, plus macOS versus the runner.
7. [seeding-and-migration.md](seeding-and-migration.md): the PR sequence, the seed day, the go/no-go checklists.
8. [official-mirror-and-url.md](official-mirror-and-url.md): CTAN's rules, the URL, the aliases, directory URLs, security settings.
9. [caching.md](caching.md): why the cache is off, the rules as code, and how `smoke` proves them.
10. [monitoring.md](monitoring.md): what is watched, how it alerts, and what to look at weekly.
