# Full CTAN mirror: summary and decisions

Entry point to the ten specification files in this directory, which expand
[the base plan](../ctan-full-mirror-plan.md). Where the ten disagree, this file picks one
answer and says why. Every number below comes from one of the eleven files; nothing new is
claimed.

## 1. The decision

Turn the tlnet-only mirror into a full CTAN mirror on the same R2 bucket, served at
`https://ctan.ijosh.com/` with every CTAN path verbatim at the host root, and register it
as an official mirror. The bucket holds 496,149 objects and 132.99 GB (every symlink a
copy; tlnet's versioned containers excluded) for $1.86 a month in storage and $0 in
operations. The pipeline stays one `Taskfile.yml` run hourly by GitHub Actions: it lists
dante (7 s), diffs the listing against a 3 MB state file kept in the bucket, fetches only
the delta in batches of at most 4 GB into the 14 GB runner, verifies the signed tlnet files
against the pinned TeX Live key, uploads with `aws s3 cp`, and checkpoints the state after
every batch, so any run can die anywhere and the next hour continues. Seeding is seven
ordinary pull requests and one day of hourly runs (126 GB from dante, ~513k Class A, $0.98
prorated storage). The edge cache stays off: the bill cannot exceed $0.36 per million GETs
and no traffic this mirror can draw reaches a dollar. Two things could stop it: Cloudflare's
CDN terms on serving large files from a Free zone (a reading of two documents, not a
number; ask support before registering), and CTAN declining a mirror with no directory
listings and no rsync (neither is a stated requirement; ask in the registration notes).

## 2. Headline numbers

| Quantity | Value | Owner |
|---|---|---|
| Stored set | 496,149 objects, 132.99 GB (123.9 GiB); 5 objects over 4.995 GiB | [taskfile-architecture.md](taskfile-architecture.md), [seeding-and-migration.md](seeding-and-migration.md) |
| Monthly cost | $1.86 storage (134 GB-month rounded, minus 10 free); operations $0 | [cost-estimates.md](cost-estimates.md#fixed-monthly-cost) |
| Seed cost | ~513k Class A (free tier 1M); storage $0.98 if seeded on the 15th | [cost-estimates.md](cost-estimates.md#seed-month) |
| Seed duration | 67 to 274 min of runner time; 8 hourly runs at `MAX_BATCHES=4` (P50 9 h, P90 11 h wall clock) or one dispatch of 1.5 to 5 h | [seeding-and-migration.md](seeding-and-migration.md#the-seed) |
| Hourly run | ~30 s quiet, ~50 s at P99, ~2 min on the busiest file hour; a release day is three lone 6.8 GB batches of ~20 min each | [sync-with-dante.md](sync-with-dante.md#8-time-budget-of-an-hourly-run) |
| Thinnest limit margin | 6-hour job during the seed (resumable by design); seed month uses ~52% of free Class A; everything hourly is under 3% of any limit | [limits.md](limits.md#8-consolidated-table) |
| Storage ceiling | 175 GB and 600,000 objects, checked on the upstream listing before any fetch | [cost-estimates.md](cost-estimates.md#budget), [verification-and-security.md](verification-and-security.md#4-the-symlink-inflation-guard-and-the-storage-ceiling) |
| Secrets | 6: the four today plus `CF_API_TOKEN`, `CF_ZONE_ID` | [caching.md](caching.md#token-and-secrets) |
| Endpoints | 6: dante, R2, `ctan.ijosh.com`, `hc-ping.com`, `api.cloudflare.com`, `ctan.org` (mirmon page) | [monitoring.md](monitoring.md#6-mirmon) |

## 3. Corrections to the base plan

Merged from every file's "Where this differs from the base plan"; one line each.

### Changes the design

| Correction | Owner |
|---|---|
| State line must be `path TAB size TAB mtime`, path first: whole-line `comm` is wrong with the size first | taskfile-architecture, sync-with-dante |
| Containers must be checked against the same run's tlpdb: fetch and verify `tlpkg/` first (2.9 MB into `RUN/`), upload it last, `cmp` the two | verification-and-security, taskfile-architecture, sync-with-dante |
| "Every named container exists" is checked against the bucket after the run (state + delta − deletions), not dante's listing | verification-and-security |
| A missing state file must not silently reseed 133 GB: refuse unless `SEED=true`; a corrupt one fails closed | taskfile-architecture, errors-and-issues |
| Refetch-storm guard: a delta over 10% of the state fails the run unless `FORCE=1` (dante restored from backup, truncated listing with exit 0) | errors-and-issues |
| `--ignore-missing-args` on the batch fetch; the state records only what landed; exit 23 on the listing means a dangling symlink upstream and must not fail the hour | errors-and-issues, sync-with-dante, taskfile-architecture |
| No `-r` on the batch fetch: a listed path that became a directory would pull an unplanned subtree | sync-with-dante |
| Listing times are in the client's zone; every rsync call runs with `TZ=UTC` or a laptop run re-uploads everything | sync-with-dante |
| The runner's rsync prints digit separators and `0` sizes; openrsync prints blanks: pass `--no-h`, parse by anchoring on the date | sync-with-dante, limits, errors-and-issues |
| `timestamp` and the root files form the final decision batch with `tlpkg/`, `timestamp` last; the base plan did not place `timestamp` | seeding-and-migration, sync-with-dante |
| The "$8.96 ceiling" is not a ceiling: evictions are undocumented, 404s and decision files are uncached and linear in traffic | caching, cost-estimates |
| The cache is an optional layer, off by default; the seed purges nothing | caching |
| 404s are not stored under the rule, so added keys need no purge; PUT then purge, never purge then PUT; purge changed `archive/` keys before `tlpkg/` lands | caching |
| Smart Tiered Cache is a persistent zone setting: `PATCH` with `Zone Settings Write`, set once; the base plan's `POST` per run is wrong | caching |
| The entrypoint `PUT` creates a new ruleset version every call; `GET` and `PUT` only on change | caching |
| Budget breaches should not fail the run: failing cannot reduce Class B and stales the mirror | monitoring |
| One `aws.config` covers multipart; the per-file threshold override is unnecessary | limits, taskfile-architecture |
| `retry_mode = standard`, not `adaptive` (experimental, process-wide throttle, nothing to adapt to) | errors-and-issues |
| The seed need not re-put tlnet: with list-diff landing first, the state already holds it | seeding-and-migration, sync-with-dante |
| The cache-rule PR lands after the seed, not before | seeding-and-migration |
| `page` must be removed or moved in the expansion PR itself: CTAN's root `index.html` lands in seed batch 4 and the reconcile would fight `page` daily | seeding-and-migration, official-mirror-and-url |
| CTAN's `index.html` links directory URLs, which are 404s on R2; a Single Redirect to `ctan.org/tex-archive` fixes it; the Worker option is closed (100k requests/day on Free is ~8 installs) | official-mirror-and-url |
| Mirmon probes each mirror's own `/timestamp`, not `mirror.ctan.org`; CTAN's file is not epoch-first | monitoring, official-mirror-and-url |
| `delete-objects` reports per-key errors inside a 200 with CLI exit 0; parse them | errors-and-issues, taskfile-architecture |
| Keep the GitHub failure email as backstop; route alerts through healthchecks `/start` and `/fail` | monitoring |
| `SECURITY.md` "only tlnet is signed" is wrong: tlcontrib and the ISO checksums are signed and verifiable; say which are checked | verification-and-security |
| No pre-seed post to the maintainers' list: membership comes with registration; write to `ctan@ctan.org` | seeding-and-migration |

### Changes a number

| Correction | Owner |
|---|---|
| Stored set 496,149 / 132.99 GB, not 496,155 / 133.01: the six `update-tlmgr-r*` files are excluded today | taskfile-architecture, seeding-and-migration, sync-with-dante |
| Operations bill in whole millions: "1,282k → $1.27" is $4.50; a third seed is $4.50 for the month, not $2.25 | cost-estimates |
| Storage $1.86, not $1.84; seed month $0.98, not $0.92 | cost-estimates |
| Average object 268 KB, not 244; 200 GB is ~746k objects, not 868k | cost-estimates |
| 14 to 16 objects exceed today's 200 MB threshold, not 5; 7 exceed the 512 MB cacheable limit | cost-estimates, limits |
| `tlmgr update` for `scheme-full` is ~417 GETs a month, not 100 to 150 | cost-estimates |
| 365-day churn on the stored set is 82,592 files / 62.43 GB; the base table showed the no-alias tree | cost-estimates |
| 523 to 525 listing lines have a blank size; the base plan's awk read their date as bytes | cost-estimates, limits, errors-and-issues |
| Seed batches 30, not ~34 (24 packed, 5 lone, 1 decision); seed pulls 126 GB from dante, not 133 | seeding-and-migration, limits |
| Dante is not one speed: 22.7 MB/s and 152 MB/s on the same day | seeding-and-migration |
| rsync retry worst case ~40 min, not 15.5 (attempt timeouts were not counted) | errors-and-issues |
| The AWS CLI does not retry a bare 429 in standard or adaptive mode | errors-and-issues |
| Cloudflare origin timeout 125 s, not 100 | limits |
| `systems/win32` is the real directory; `systems/windows` is the 23.64 GB alias | official-mirror-and-url, sync-with-dante, cost-estimates |
| 27,262 directories once aliases are materialised, not 18,417; 21 root-hosted HTTPS mirrors, not "a dozen" | official-mirror-and-url |
| Hour-slots with work: 276 of 727 in UTC, not 283 of 720 in PDT | sync-with-dante |
| Purge propagation P50 ~250 ms; hostname, tag and prefix purge also exist on Free | caching |
| Token needs `Zone → Analytics → Read` and `Zone Settings Read` as well | monitoring, caching |
| "Free plan may serve large files via R2: verified" is overstated; unverified with residual risk | cost-estimates, limits |
| Wikipedia's "6 TB/month" is uncited; the one measured figure is ftp.fau.de's 95 TB in 2018 | cost-estimates |
| `cacheStatus` on `httpRequestsAdaptiveGroups` and the `actionType` vocabulary are unverified | monitoring |

### Cosmetic

Listing 6.9 s not 6.6; `scheme-full` 11,913 GETs not 11,919; versioned containers +$0.09 not
$0.10; curl also retries 522 and 524; GraphQL 300 or 320 per 5 min; `split -a 3` needed for
4,962 purge chunks; rsync exit 22 named as never-retry.

## 4. Conflicts between the ten files, resolved

**a. Stored set.** 496,155 / 133.01 GB (cost-estimates, limits, verification, monitoring,
official) vs 496,149 / 132.99 GB (taskfile, seeding, sync). **496,149 / 132.99 GB**: the
six `update-tlmgr-r*` files are excluded by today's `fetch` and by every proposed
normaliser, so the larger count describes a set nobody stores; every cost rounds the same.

**b. Seed batch count.** 34 with a tlnet re-put (taskfile) vs 30 (limits 25+5; seeding
24+5+decision). **30, with no tlnet re-put.** The list-diff PR lands before the expansion PR,
so the state already holds tlnet and the expansion delta is 479,175 objects. Two missing-state
mechanisms merge: a missing state fails unless `SEED=true` (taskfile's gate), and recovery
from a lost state is `RECONCILE=true`, whose rebuild of the state as upstream ⋈ bucket is
exactly sync-with-dante's bootstrap. Errors-and-issues' "NoSuchKey means seed" yields to the
gate.

**c. Multipart config.** 4 GB / 512 MB (taskfile) vs 5 GB / 1 GB (limits) vs a second
config file with 500 MB chunks (seeding) vs per-file override (base). **One `aws.config`:
`multipart_threshold = 4GB`, `multipart_chunksize = 512MB`.** The CLI's suffixes are binary,
so limits' `5GB` is 5 GiB, above R2's 4.995 GiB single-part limit; 4 GiB sits safely between
`protext.zip` (1.14 GB) and the installers, and 13 parts of 512 MiB is 15 Class A per
installer, once a year. No second file, no override.

**d. AWS retry mode.** `adaptive` (base, taskfile, limits, seeding PR 1) vs `standard`
(errors-and-issues). **`standard`, `max_attempts = 10`.** Errors-and-issues is the owning
file and the only one that examined it: adaptive is marked experimental, its throttle is
process-wide, and R2 publishes no rate to adapt to.

**e. Purging added keys.** Never needed (caching: 404 not stored under the rule; limits:
3-minute default TTL) vs purge added too (errors-and-issues, from R2's consistency page).
**Never, and with the cache off nothing is purged at all.** The rule's `status_code_ttl` of
`-1` for 3xx and up removes the case errors-and-issues describes; if the API rejects that
shape (caching's open question 1), add `added` to the purge list, which costs one call an
hour.

**f. Seed execution.** One 6-hour dispatch with `timeout-minutes: 360` (base) vs hourly
runs with `MAX_BATCHES=4` over ~8 runs (seeding) vs `timeout-minutes: 350` always
(taskfile). **`MAX_BATCHES=4` as the hourly default, a `workflow_dispatch` input to raise it
on the seed day, and `timeout-minutes: 350` always.** Checkpoints make a long run safe and
nothing in the pipeline can hang for hours, so a low timeout gains nothing; the batch bound
is what makes each run report and ping. Pause the healthchecks `sync` check for the seed day.

**g. Landing page.** Remove `page`, serve CTAN's root `index.html` (taskfile, seeding) vs
keep ours at `/.site/index.html` behind the retargeted Transform Rule with CTAN's at
`/index.html` (official). **Official's option B.** It keeps the one page that tells a human
how to point `tlmgr` here, costs one key and one line in the reconcile filter that `.state/`
already needs, and still serves CTAN's page byte for byte at its own path.

**h. Budget breaches.** `report` fails the run (cost-estimates) vs signal a separate
`health` check (monitoring). **Neither fails the run for Class B; the pre-upload storage and
object guard still fails it.** But see k: the GraphQL `usage` step is deferred, so the
`health` check has no inputs yet and is not created (p).

**i. Edge cache.** On from the start (base) vs `CACHE: off` with one bypass rule, switched
on at 5M Class B a month for two months (caching). **Off.** Below ~28 `scheme-full`
installs a day the cache saves nothing, and on it adds a purge step, a token scope and two
correctness windows. The rules JSON, `task rules` and the token stay, so the switch is one
line.

**j. Cron minute.** `41 * * * *` (taskfile) vs "after :05" (errors-and-issues) vs mirmon at
:03 (seeding). **41.** All three agree once read together: fixed, random, not 00 to 05.

**k. Tools outside the list.** `jq` for GraphQL (monitoring); `python3 -m json.tool`
(caching). **The tool line does not change; the design avoids both.** JSON syntax is checked
by taskfile's `fromJson` lint. The GraphQL `usage` step is deferred: its numbers (Class B,
hit ratio) matter only for a cache that is off, its fields are unverified, and the R2
dashboard's Metrics tab gives the same totals monthly. If `usage` is ever adopted, add `jq`
(already on the runner image) and say so in CLAUDE.md; `sed` over GraphQL JSON is not honest.

**l. tlnet versioned containers.** Exclude (everyone) vs keep so `FILES.byname` never lies
(official raises it). **Exclude, and exclude tlcontrib's 261 as well.** Nothing requests
either name, `tlmgr` reads the tlpdb not `FILES.byname`, and the cost of keeping them is
storage plus a second upload per revision bump for no reader. The 496,149 / 132.99 GB figure still counts tlcontrib's 261 (0.46 GB); with them out the set is ~495,900 objects / 132.5 GB, which changes no dollar figure.

**m. Reserved prefixes.** `.state/` is needed by taskfile, sync, seeding, errors, monitoring
(`history.csv`), verification and caching; `.site/` by official and, with g, by all of
them. The CLAUDE.md line becomes: "Objects sit at CTAN's own paths from the bucket root.
`.state/` and `.site/` are the mirror's own; every bucket listing excludes them, and CTAN has
no dot-prefixed root entry, so they cannot collide."

**n. Directory URLs.** 404 (base) vs a Single Redirect to `https://ctan.org/tex-archive<path>`
(official). **The redirect.** One free rule as code beside the other two; its `ne "/"`
clause keeps the landing-page rewrite alive. Fallback if the expression form is not on Free:
404, as today.

**o. `verify` with a delta.** Verification, taskfile and sync describe one mechanism: fetch
`tlpkg/` into `RUN/` at the start of any run that touches tlnet, verify sha512, signature and
pin there, check each batch's containers against that tlpdb, upload `tlpkg/` last and `cmp`
it against the copy in `RUN/`. Three differences: taskfile checks "exists" against dante's
listing at run start, verification and sync against the bucket after the run just before the
decision batch (**take the latter**); sync drops the tlnet decision files and continues where
the others fail the run (**fail the run**; simpler, and the mid-update case is an hour a few
times a month); verification adds a one-GetObject diff of the old tlpdb's checksums to catch
a same-size same-second change (**adopt**).

**p. Healthchecks checks.** 1 today vs 3 (monitoring). **2: `sync` (cron, `/start`,
`/fail`, grace 2 h) and `reconcile` (period 1 d).** `health` returns with `usage`. Mirmon
"old" on two consecutive runs fails the run instead. The GitHub failure email stays on "only
failed": it is the one channel that fires when healthchecks itself is unreachable. The
workflow gains `if: failure()` → `task fail`, and CLAUDE.md's "workflows run one task" says
so.

**q. CLAUDE.md lines that change.** The opening paragraph (hourly, all of CTAN, 496k files,
133 GB, largest 6.87 GB); "four files" (add `cloudflare/*.json`; `site/` moves to
`.site/`); "Zero running cost" → "storage is the only bill, $1.86 at 133 GB, ceiling
175 GB / 600k objects"; "workflows install tools and run one task, and `task fail` if it
failed"; tools unchanged; endpoints gain `api.cloudflare.com` and `ctan.org`; the objects
line (m); secrets six. Must-knows: "No versioned containers" gains tlcontrib and the
normaliser; "Never `--size-only`" becomes "the state line carries upstream size and mtime";
"Never `sync --delete`" becomes "the hourly path never lists the bucket; `LC_ALL=C sort`
every listing"; "Single-part uploads" gains the five exceptions and the 4 GiB threshold;
"`publish` uploads in a fixed order" becomes the batch rank table with the decision batch
last; "`index.html` lives at the bucket root" becomes the `.site/` line; "A failed run is the
only alert" gains the 2-hour grace and "a run stopping at `MAX_BATCHES` is not a failure";
the `.xz` cache line becomes "the cache is off; `CACHE: on` is the purge PR". Verification
item 5 becomes the delta form. "Verifying a change" becomes taskfile's section 8 table.

Other conflicts found while reading, resolved the same way:

- **r. `.prev` state copy** (errors-and-issues) vs none (taskfile): **none**; a corrupt
  state fails closed and `RECONCILE=true` rebuilds it from the bucket for 497 Class A.
- **s. `--partial` on the batch fetch** (errors-and-issues) vs dropped (taskfile, sync):
  **dropped**; staging is emptied per batch and a partial file is never uploaded.
- **t. `smoke` sample check**: byte `cmp` of copies set aside (caching, monitoring) vs
  `Content-Length` against the listing (taskfile): **`Content-Length` while the cache is
  off** (R2 is strongly consistent); the `cmp` ships with the `CACHE: on` PR.
- **u. `rules` per run**: `POST` tiered cache each run (taskfile) vs `GET`, `PUT` on change,
  tiered cache untouched until the cache is on (caching): **caching's**.

## 5. The recommended design

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
mirmon row) → `report` → `ping`. The workflow runs `task fail` on failure.

**State.** One object, `.state/applied.txt.xz`: the listing lines of what the bucket holds,
path first, `LC_ALL=C` sorted, 3.1 MB, written once per batch after upload succeeds. Every
write is idempotent; a run that dies leaves the state as of the last batch and the next hour
repeats at most one batch. `.state/history.csv` gets one line per successful run.

**Storage set.** Everything `rsync -rL` yields minus the versioned containers: 496,149
objects, 132.99 GB, every alias a copy, five objects multipart. `.state/` and `.site/` are
the only non-CTAN keys.

**Cache.** Off. Four rule files as code in `cloudflare/`, three applied while off: bypass-all cache rule, `/` →
`/.site/index.html` rewrite, directory-URL redirect to `ctan.org/tex-archive`. No purge
step, no tiered cache. Flip `CACHE: on` (two-rule cache JSON, purge of changed and deleted
keys per batch, `smoke` HIT assertions, Smart Tiered Cache) when the R2 dashboard shows
Class B above 5M for two months or users far from the bucket complain.

**Monitoring.** A failed run is the alert; healthchecks `sync` (hourly, `/start`, `/fail`,
grace 2 h) and `reconcile` (daily) are the dead man's switches; GitHub's failure email stays
as backstop. `report` prints counts from `RUN/`, never lists; the mirmon row; days since the
last commit with a warning at 45. Monthly: the R2 Metrics tab and the bill.

**Migration.** Seven PRs, each leaving the tlnet mirror green:

1. `fix(sync): bounded retries` — `standard` mode, `max_attempts = 10`, the `CURL` variable,
   the `retry` task, `--ignore-missing-args`.
2. `refactor(sync): list-diff against a state file, tlnet scope` — path-first state, `SEED`
   gate, `plan`, `tlpdb`, `delete`, `reconcile`; `stale` and `guard` go. Run for a week.
3. `ci(sync): hourly at :41` — `timeout-minutes: 350`, `MAX_BATCHES`, the dispatch input,
   the two healthchecks checks, `task fail`.
4. `feat(publish): one aws.config` — threshold 4 GiB, chunk 512 MiB; dead code until 6.
5. `feat(rules): cloudflare rules as code` — bypass, rewrite to `.site/`, redirect; `page`
   writes `.site/index.html`; the two secrets. Must precede 6 so the page moves before CTAN's
   `index.html` arrives.
6. `feat(sync): mirror the whole of CTAN` — `SOURCE`, bucket root, ceilings, tlcontrib and
   ISO checks, README, SECURITY.md, CLAUDE.md. Rehearse with `max_batches: 1`; the seed is
   the hourly runs that follow, or one dispatch. Write to `ctan@ctan.org` two days before.
7. `docs(mirror): register` — then the form, then watch mirmon at :03.

Later, at the trigger: `feat(cache): CACHE on` with the purge step.

**After the change.** Secrets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`,
`HEALTHCHECK_URL`, `CF_API_TOKEN`, `CF_ZONE_ID`. Endpoints: `rsync.dante.ctan.org`, R2,
`ctan.ijosh.com`, `hc-ping.com`, `api.cloudflare.com`, `ctan.org`. Tools unchanged.

## 6. Open questions that survive

Flagged **STOP** if the answer could end the project.

| Question | Cheapest experiment | Owner |
|---|---|---|
| **STOP** Does paid R2 usage on a Free zone satisfy the CDN terms' "Paid Services" clause for large files? | A written answer from Cloudflare support, before registering | cost-estimates, limits, official |
| **STOP** Will CTAN list a mirror with no directory listings, no rsync, and directory URLs redirected to `ctan.org`? | Ask `ctan@ctan.org` with the pre-seed note; say it in the registration Notes | official, seeding |
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

## 7. Reading order

1. [cost-estimates.md](cost-estimates.md): what it costs, where the money could start, and the budget lines.
2. [sync-with-dante.md](sync-with-dante.md): the list-diff, the contract with upstream, symlinks, the consistency argument.
3. [taskfile-architecture.md](taskfile-architecture.md): the Taskfile and workflows, task by task, with the data files.
4. [verification-and-security.md](verification-and-security.md): what is signed on CTAN, the delta-scoped checks, the threat model.
5. [errors-and-issues.md](errors-and-issues.md): every failure, every retry, the runbook.
6. [limits.md](limits.md): every wall and the margin to it, plus macOS versus the runner.
7. [seeding-and-migration.md](seeding-and-migration.md): the PR sequence, the seed day, the go/no-go checklists.
8. [official-mirror-and-url.md](official-mirror-and-url.md): CTAN's rules, the URL, the aliases, directory URLs, security settings.
9. [caching.md](caching.md): why the cache is off, the rules as code, and how `smoke` proves them.
10. [monitoring.md](monitoring.md): what is watched, how it alerts, and what to look at weekly.
