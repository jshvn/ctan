# Seeding and migration

How the tlnet-only mirror becomes a registered full CTAN mirror without breaking `task sync`
for a single run, and how the first 126 GB gets into the bucket.

Numbers are computed from the listing taken 2026-08-26 ~17:00 UTC (`SCRATCH/ctan-list-deref.txt`)
or from the two `sync.yml` runs whose logs were read today. Anything else is marked.

## In brief

- Six pull requests, in this order: hardening, list-diff on tlnet only, hourly schedule,
  multipart, the expansion to all of CTAN, registration docs. Each leaves the live tlnet
  mirror green. The cache-rule PR (`caching.md`) goes after the seed, not before.
- The seed is not a procedure. It is the first hourly runs after the expansion PR merges,
  each doing at most `MAX_BATCHES` batches and exiting cleanly with a report. Delta:
  **479,175 objects, 126.21 GB, 30 batches** (24 packed at 4 GB, 5 lone installers,
  `timestamp` last). The 16,974 tlnet keys are never re-uploaded: the state file already
  holds them.
- Duration: 67 minutes of runner time at the fast rates seen today, 274 at the slow ones.
  With `MAX_BATCHES=4` per hourly run the seed takes **8 hourly runs, P50 9 h, P90 11 h**
  wall clock, and no run comes near the 350-minute timeout. A `workflow_dispatch` input
  can raise the batch count and finish in one or two runs; it is optional.
- Seed month: ~513k Class A (free tier 1M), storage prorated to ~$0.97, then $1.84/month.
- No move of existing keys. `.state/` and `.site/` are the two keys outside CTAN's own
  paths and every listing step excludes them. CTAN ships its own root `index.html`; the
  custom page moves to `/.site/index.html` and the Transform Rule follows it.
- Rehearsal: run the expansion with `MAX_BATCHES=1`. Batch 1 is 39,145 small files under
  4 GB and proves every new code path on the real bucket for zero extra cost.

## What exists today

| Fact | Value | Source |
|---|---|---|
| tlnet stored set | 16,974 objects, 6,783,160,850 bytes (6.78 GB) | listing, tlnet subset with `fetch`'s two excludes; matches rsync `--stats` in run 32997802111 |
| of which `tlpkg/` | 2,083 objects, 94.4 MB | listing |
| of which `archive/` | 14,872 objects, 6.62 GB | listing |
| Bucket, prefix | `tlnet`, `systems/texlive/tlnet/` (`BUCKET: s3://tlnet/{{.PREFIX}}`) | `Taskfile.yml` |
| Landing page | `s3://tlnet/index.html`, `Cache-Control: no-cache`, outside the prefix | `Taskfile.yml` `page`; `curl -sI https://ctan.ijosh.com/` today: 200, `last-modified` 2026-08-26 18:07:42 |
| Transform Rule | `/` rewritten to `/index.html` | `CLAUDE.md`; `https://ctan.ijosh.com/` and `/index.html` return the same headers |
| Full pull from dante | 6.786 GB in 299 s = **22.7 MB/s** (run 32933376123, 05:16:08 to 05:21:07 UTC) and 6.786 GB in 44.6 s = **152 MB/s** (run 32997802111, 18:05:53 to 18:06:38 UTC) | job logs via `gh run view <id> --log` |
| PutObject rate | 7,891 `upload:` lines survive in the 05:15 log for an 82 s publish (05:21:23 to 05:22:45): **96/s floor**; if all 16,974 uploaded, 207/s | same log; GitHub drops lines, so the true count is unknown |
| `aws s3 sync` with nothing to do | 14 s for 14,872 `archive/` keys, 15 s for the rest; `aws s3 ls --recursive` of 17k keys 14 s | run 32997802111 |
| Runner | 6 h per job on GitHub-hosted runners | docs.github.com/en/actions/reference/limits |
| Concurrency | default `queue: single`: "at most one job or workflow run can be pending in the concurrency group"; FIFO; `queue: max` allows 100 | docs.github.com control-workflow-concurrency |

The 05:15 run is the slow end of dante, the 18:05 run the fast end, on the same day and the
same runner class. Both rates are used below; a single figure would be false precision.

## The migration as a sequence of PRs

### Order and why

| # | PR | Changes the live tlnet mirror's behaviour? | Adds bucket state? |
|---|---|---|---|
| 1 | `fix(sync): bounded retries for rsync, curl and the AWS CLI` | no | no |
| 2 | `refactor(sync): list-diff against a state file, tlnet scope only` | how it syncs, not what | `.state/applied.txt.xz` |
| 3 | `ci(sync): run hourly at a fixed random minute` | cadence | no |
| 4 | `feat(publish): multipart for objects over 4.995 GiB` | no (no such object in tlnet) | no |
| 5 | `feat(sync): mirror the whole of CTAN` | the seed starts at the next run | 479k objects |
| 6 | `docs(mirror): register as an official CTAN mirror` | no | no |

The cache-rule and purge PR from `caching.md` lands between 5 and 6, after the seed has
finished. Before the seed it would purge 479k URLs nobody has ever requested (4,792 calls,
~25 min of runner time per the base plan's rate) for no effect.

The question the caller asked: does list-diff land before the storage-set expansion? Yes,
exactly that, as PR 2 against PR 5. PR 2 runs the new pipeline on the 16,974 keys that are
already live, with `SOURCE` and `PREFIX` unchanged, for at least a week of scheduled runs.
The pipeline's every path (listing, state read, `comm`, batching, `--files-from`, `cp
--recursive`, state write, deletions, the daily reconcile) is exercised at a scale where a
bug costs one day of staleness on 17k keys, not a half-seeded 133 GB bucket. When PR 5
merges, the only new thing is the size of the delta.

Assumption: a pipeline that is correct on 17k objects is correct on 496k. What breaks if
false: batch packing on the 97k-file `fonts/` batches (never seen at tlnet scale), the
state-file write at 3.1 MB (tlnet's is 132 kB), and the reconcile's `comm` at 496k lines.
Checked: the packer is one `awk` pass over the listing and was run on the full listing
today (30 batches, table below); `comm` and `sort` on 496k lines take under a second on the
runner class. Not checked: `rsync --files-from` with 97k paths in one call; the rehearsal
in [Dry runs](#dry-runs-and-the-rehearsal) is what checks it.

### Compared with a big-bang cutover

One PR that flips `SOURCE`, `PREFIX`, the pipeline and the schedule at once, then a
dispatch that seeds. What it saves: three PRs and about a week. What it costs: the first
run of the new pipeline is also the seed, so a pipeline bug (a wrong `comm` column, a
`--files-from` path with a `#` in it, a state file written before its batch landed) is
found at batch 12 with 50 GB in the bucket, and the rollback is a revert plus a bucket
cleanup under time pressure while the live tlnet mirror is served by a pipeline nobody has
run. There is no way to tell a scale problem from a logic problem. The staged order makes
each PR's failure mode small and its rollback a revert.

### PR 1: `fix(sync): bounded retries for rsync, curl and the AWS CLI`

What changes: `aws.config` gains `retry_mode = standard`, `max_attempts = 10`,
`cli_connect_timeout = 60`, `cli_read_timeout = 300` (`standard`, not `adaptive`: adaptive
is documented as experimental, its token bucket is process-wide, and R2 publishes no
per-bucket rate to adapt to; `errors-and-issues.md`); `Taskfile.yml` gains the `CURL`
variable and the `retry` task from the base plan, and the two rsync lines and the three
curl lines use them. No task's inputs or outputs change.

Must not change: `SOURCE`, `PREFIX`, `BUCKET`, the `publish` order, `verify`, `stale`.

Verified before merge: `task --dry sync` renders; the `retry` loop is run under Task with
`CMD='exit 5'` scaled down (the base plan did this: exit 5 twice then 0 succeeds, exit 23
fails at once, permanent 5 gives up after five). `task smoke URL=file:///<dir>
STAGING=<dir>` still passes with the `CURL` variable.

Verified after merge: the next 03:30 run's `report` is identical in shape; the rsync stats
block shows the same `Number of regular files transferred`.

Rollback: revert. No bucket state.

### PR 2: `refactor(sync): list-diff against a state file, tlnet scope only`

This is the intermediate state the caller asked to have described precisely.

What changes in `Taskfile.yml`:

- `SOURCE` stays `rsync://rsync.dante.ctan.org/CTAN/systems/texlive/tlnet/`. `PREFIX`
  stays `systems/texlive/tlnet`. `BUCKET`, `URL`, `SITE` unchanged.
- `STATE: s3://tlnet/.state/applied.txt.xz`. The state file's lines are `key<TAB>size
  <TAB>mtime`, sorted with `LC_ALL=C sort` on the whole line, where `key` is the full
  CTAN-relative key, `systems/texlive/tlnet/...`, never the module-relative path. Key first
  because `comm` compares whole lines: with the size first, two keys of equal size and
  mtime would collate by size and the diff would mis-pair them
  (`taskfile-architecture.md` shows the failure). Key-first is what lets the same file carry
  over into PR 5 unchanged.
- `list` (new): `rsync -rL --list-only {{.RSYNC_TIMEOUTS}} {{.SOURCE}}` through `retry`,
  normalised by the `norm.awk` from the base plan (anchors on the date column, so blank
  sizes and any byte in a name parse), versioned containers and `update-tlmgr-r*` dropped
  with the same two patterns `fetch` uses today, paths prefixed with `{{.PREFIX}}/`,
  `LC_ALL=C sort` into `RUN/upstream.txt`.
- `state` (new): `aws s3 cp {{.STATE}} - | xz -dc > RUN/applied.txt`. A missing key fails
  the run unless `SEED=true`, which starts from an empty file (the deliberate seed path);
  a lost or corrupt state is rebuilt with `RECONCILE=true`, which joins the bucket listing
  to the upstream listing on key and size and writes the result as the state (the
  bootstrap; `taskfile-architecture.md`). A 403 always fails.
- `plan` (new): `comm -13 RUN/applied.txt RUN/upstream.txt` is the fetch list;
  `comm -23` on the key column alone is `RUN/deleted.txt`. The packer (below) writes
  `RUN/batch-NNNN.txt`, one module-relative path per line for `--files-from`, and a
  matching `RUN/batch-NNNN.tsv` to append to the state.
- `fetch` becomes per-batch: `rsync -rLt {{.RSYNC_TIMEOUTS}} --files-from=RUN/batch-NNNN.txt
  {{.SOURCE}} {{.STAGING}}/` into an emptied `staging/`. No `--delete`, no excludes needed
  (the listing already dropped them; a versioned name never enters a batch).
- `verify` runs on what is local, as the base plan's "What `verify` means with a delta"
  describes. In this phase every batch is under tlnet, so the signed checks run whenever
  `tlpkg/` is in the batch and the container check runs on every `archive/` key present.
- `publish` becomes per-batch: `aws s3 cp --recursive {{.AWS_FLAGS}} {{.STAGING}}/
  {{.BUCKET}}/` (N PutObject, no listing), then `cat RUN/batch-NNNN.tsv >> RUN/applied.txt`,
  `LC_ALL=C sort -o RUN/applied.txt RUN/applied.txt`,
  `xz -6 < RUN/applied.txt | aws s3 cp - {{.STATE}}`, then `rm -r staging/*`. The `for`
  over batches is Task's `for: { var: BATCHES, split: "\n" }`, run today in the scratch
  `tasktest/Taskfile.yml` with an empty list, a one-item list and a path with a space.
- `delete` (new) replaces the `xargs aws s3 rm` loop and its `ponytail`: `RUN/deleted.txt`
  in slices of 1,000 to `aws s3api delete-objects --bucket tlnet --delete file://...`,
  then the keys are removed from `RUN/applied.txt` and the state written again.
- `stale` is deleted. `reconcile` (new) runs only in the 03:30 slot (Task `status:` on
  the hour, or `RECONCILE=true` set by the workflow's daily job): `aws s3 ls --recursive
  {{.BUCKET}}/` (17 pages now, 497 later) to `RUN/remote.txt` as `key<TAB>size`, joined to the
  state on key and size; mismatches go on the fetch list, unknown keys on the delete list.
  Both lists exclude `^\.state/` and, in this phase, nothing under the prefix is ever
  outside CTAN, so `index.html` at the root is not listed at all (`aws s3 ls --recursive
  s3://tlnet/systems/texlive/tlnet/` cannot see it). The base plan's `guard` (a `du` of
  staging) becomes a check on the plan: fail if any single file exceeds the free disk, and
  fail if the upstream listing's object count or byte total exceeds `LIMIT` (still 10 GB
  in this phase).
- `report` counts from `RUN/upstream.txt`, `RUN/applied.txt`, the batch files and
  `RUN/deleted.txt`, never from the job log. Rows: `Upstream listing`, `Published to R2`
  (uploaded, batches, deleted), `State` (lines, bytes), and the existing `Signature` and
  `Read-back` rows.

The packer, as one `awk` (the scratch `plan.awk`, extended with the rank and `timestamp`):

```awk
# in: key \t size \t mtime, whole-line sorted (the state format). out: DIR/batch-NNNN.tsv
BEGIN { FS = OFS = "\t"; cap = CAP + 0; lone = LONE + 0 }
$1 ~ /^systems\/texlive\/tlnet\/tlpkg\// || $1 == "timestamp" { last[++nl] = $0; next }
$2 > lone { flush(); n++; print > sprintf("%s/batch-%04d.tsv", DIR, n); next }
{ if (sz && sz + $2 > cap) flush(); sz += $2; buf[++nb] = $0 }
END { flush(); if (nl) { n++; for (i = 1; i <= nl; i++) print last[i] > sprintf("%s/batch-%04d.tsv", DIR, n) } }
function flush(  i, f) { if (!nb) return; n++; f = sprintf("%s/batch-%04d.tsv", DIR, n)
  for (i = 1; i <= nb; i++) print buf[i] > f; close(f); nb = 0; sz = 0; delete buf }
```

`CAP=4000000000`, `LONE=5237247590` (4.995 GiB). `tlpkg/` and `timestamp` always land in
the final batch, so the tlpdb clients read never names a container the bucket lacks, and
`timestamp` never says "fresh" ahead of the tree it describes. Input order is the key sort,
with tlnet's `archive/` ranked first (a prefix rank column, `sort -k1,1n -k4`), so within a
batch plan the tlnet containers go before everything else, as today.

The tlnet-scoped numbers, from the listing restricted to `systems/texlive/tlnet/`:

| Quantity | Value | Command |
|---|---|---|
| Objects, bytes | 16,974; 6,783,160,850 (6.78 GB) | `awk '$1=="-" && $5 ~ /^systems\/texlive\/tlnet\// && !($5 ~ /\.r[0-9]+.*\.tar\.xz$/) && !($5 ~ /update-tlmgr-r/) {n++; s+=$2} END {print n, s}' deref.norm` |
| Batches at 4 GB, `tlpkg/` last | **3**: 8,383 files / 4.000 GB; 6,508 / 2.689 GB; 2,083 / 0.094 GB | `plan.awk` on the subset |
| State file | 16,974 lines, 1,342,628 bytes text, **131,608 bytes** as `.xz -6` | the subset's `key\tsize\tmtime` lines \| `xz -6` \| `wc -c` (field order does not change the byte count) |
| First run, dispatched with `SEED=true` | delta = all 16,974 keys, 3 batches, ~17k PutObject, ~3 to 6 min | same as a tlnet seed; run 32933376123 took 82 s to put at least 7,891 |
| Every later run | delta = the day's tlnet changes (15 keys on 2026-08-26 18:05) in 1 batch | run 32997802111 uploaded 15 |
| Class A per run | ~15 PutObject + 1 state write; the 03:30 run adds 17 list pages | free |

The first run after merge must be a `workflow_dispatch` with `SEED=true`, because a
missing state fails every other run by design. It re-puts every tlnet key once: 17k
Class A and a few minutes, and the proof that the empty-state path (the seed path) works.
The alternative, `RECONCILE=true` to bootstrap the state from the bucket listing with no
uploads, is the recovery path for a lost state later; do not use it here, because it
skips the one exercise of the seed path before the real seed.

What changes in `.github/workflows/sync.yml`: nothing. In `CLAUDE.md`: the "Never `aws s3
sync --delete`" and "`publish` uploads in a fixed order" must-knows are rewritten for the
batch loop (there is no `sync` any more, so the first becomes "never list the bucket in
the hourly path; deletions come from the state diff, the daily reconcile is the only
listing"); the "Do not trust the job log" must-know stays; "Verifying a change" gains
`task plan RUN=<dir>` against a canned `upstream.txt` and `applied.txt` (must produce the
expected batch files, must put `tlpkg/` last, must fail on an empty upstream) and
`task reconcile RUN=<dir>` against a canned `remote.txt` (must be empty when bucket and
state agree, must exclude `.state/`). Secrets: none added.

Must not change: `SOURCE`, `PREFIX`, `BUCKET`, `page`, `smoke`'s URL, `TL_KEY`, the
`verify` checks themselves.

Verified before merge (offline): the five canned-directory checks above plus `task --dry
sync`. The packer is run on the scratch tlnet subset and must produce the three batches in
the table. `norm.awk` is run on the full listing and must report zero `UNPARSED` lines
(it did today: 538,289 lines in, 538,289 out).

Verified after merge: the `SEED=true` dispatch's `report` shows `Published to R2: 16,974
uploaded in 3 batches, 0 deleted`, `State: 16,974 lines`; the second shows a delta in the
tens and `State` unchanged in count; the first 03:30 reconcile shows `0 mismatched, 0
unknown`. `smoke` passes on both. Watch for a week.

Rollback: revert. The state key `.state/applied.txt.xz` stays in the bucket; the reverted
`stale` cannot see it (it lists under the prefix) and it costs nothing measurable. Delete
it by hand with `aws s3 rm s3://tlnet/.state/applied.txt.xz` if the revert is permanent.
No object under the prefix needs undoing: every write was a PutObject of upstream bytes at
the upstream key, which is what the old pipeline would have written.

### PR 3: `ci(sync): run hourly at a fixed random minute`

What changes: `cron: '30 3 * * *'` becomes `cron: 'M * * * *'` with `M` from `shuf -i 0-59
-n 1` (the register page asks for a random, permanent minute), `timeout-minutes: 30`
becomes `350` (the value is permanent, seed or not; every call has its own timeout, and a
run that stops at `MAX_BATCHES` exits long before it), and the daily reconcile keys on the
hour `M` of 03:00 (`RECONCILE=true` when `date -u +%H` is `03`). `MAX_BATCHES: 4` appears
as a Task var with a comment. healthchecks becomes two checks, `sync` (hourly, pinged with
`/start` and `/fail`) and `reconcile` (daily); periods and grace are `monitoring.md`'s.

Must not change: anything in the pipeline's data path.

Verified before merge: `task --dry sync`; the workflow file passes `check.yml`. Verified
after merge: 24 green runs on the Actions page in a day, each `report` showing a listing
of ~6.6 s and a delta of tens of keys or zero; the 03:xx run shows the reconcile row.

Rollback: revert the cron line. No bucket state.

Note the load this puts on dante in this phase: 24 listings a day of the tlnet module
(17k entries, well under the 6.9 s full listing), and one `--files-from` pull per hour
with work. That is less than one `rsync -a` of a tlnet mirror.

### PR 4: `feat(publish): multipart for objects over 4.995 GiB`

What changes: the packer's lone-file branch already exists; `aws.config` changes from
`multipart_threshold = 200MB` to `multipart_threshold = 4GB` and gains
`multipart_chunksize = 512MB`. One file, no override, no second config: every object under
4 GB stays a single `PutObject` exactly as today (the largest non-installer on CTAN is
under 4 GB; the five over 4.995 GiB are the only ones over 4 GB), and the five installers
go multipart. The `aws.config` comment gains the exception. `guard` fails if such a file
exceeds free disk.

Why 512 MB: R2 accepts parts of 5 MiB to 5 GiB and at most 10,000 (limits page); the CLI's
default 8 MB chunk makes 4,065 parts for the five installers (`awk -F'\t' '$2 > 5237247590
{p += int(($2 + 8388607) / 8388608)}'` over the state-format stored set), 512 MB makes
**13 per installer, 65 in all** (`int(($2 + 536870911) / 536870912)`): 6,865,013,189 B and
6,784,798,720 B both round up to 13. Per installer: 13 parts + create + complete =
**15 Class A**; 75 for the five. Bigger chunks are fewer retries' worth of state on a flaky
link, not more.

Must not change: the single-part path for everything under 4 GB.

Verified before merge: `task plan RUN=<dir>` against a canned `upstream.txt` containing one
6,865,013,189-byte line must emit a one-line batch; `task --dry sync` renders unchanged.
There is no tlnet object over 145 MB, so this PR is dead code on the live mirror until
PR 5; that is the point of merging it separately.

Verified after merge: the next run's `report` is unchanged. The real check is batch 25 of
the seed: `aws s3api head-object --bucket tlnet --key systems/mac/mactex/MacTeX.pkg` must
show `ContentLength: 6865013189` and a multipart `ETag` (`-13` suffix).

Rollback: revert. If a multipart upload was left incomplete, `aws s3api
list-multipart-uploads --bucket tlnet` shows it and `abort-multipart-upload` is free.

### PR 5: `feat(sync): mirror the whole of CTAN`

What changes:

- `SOURCE: rsync://rsync.dante.ctan.org/CTAN/`. `PREFIX` becomes empty; `BUCKET:
  s3://tlnet`, `URL: https://{{.HOST}}`. The `list` task's prefixing becomes a no-op; the
  tlnet excludes are applied to paths matching `^systems/texlive/tlnet/`. The state file's
  keys are already CTAN-relative, so the delta of the first run is exactly the 479,175 keys
  the state does not know, plus whatever tlnet changed in the hour.
- `LIMIT` becomes 175 GB and 600k objects, applied to the upstream listing before any
  batch is fetched, as `guard`.
- `page` survives and changes key: it uploads `site/index.html` to `s3://tlnet/.site/index.html`,
  and the Transform Rule is retargeted from `/` → `/index.html` to `/` → `/.site/index.html`
  in the same change window (the rule is dashboard state until `caching.md`'s rules-as-code
  PR; change it right after merge, before the seed's batch 4 lands). CTAN's own `index.html`
  is stored at `/index.html` like every other CTAN key. `.site/` joins `.state/` in every
  listing exclusion. `CACHE: off` stays the default until the cache PR. See
  [Bucket layout](#bucket-layout-during-and-after).
- `smoke` adds `/timestamp` read back against staging when `timestamp` was in this run's
  last batch, and a size check by `curl -sI` on three random keys from the run's batches.
- `verify`: the tlnet checks are unchanged; the "every named container exists" check reads
  the upstream listing (base plan); a `.tsv` line under `systems/texlive/tlnet/archive/`
  in a batch is checksummed against the tlpdb only when the tlpdb is local, otherwise
  against the previous verified tlpdb held in `RUN/` from the state read (the base plan's
  "A container present upstream and not in the delta is by construction the copy already
  verified"). Details are `verification-and-security.md`'s.
- `CLAUDE.md`: the constraint "Objects stay under `systems/texlive/tlnet/`" becomes
  "Objects are CTAN's own paths from the bucket root; `.state/` and `.site/` are the only
  keys outside them". "Zero running cost" becomes "storage is the only bill, $1.84/month at
  133 GB; operations stay in the free tier". The must-know "`index.html` lives at the bucket
  root" becomes "the landing page lives at `/.site/index.html`, outside CTAN's paths, and
  `/index.html` is CTAN's"; "Single-part uploads" gains the five-object exception; "No versioned
  containers" stays verbatim. The size ceiling line changes from 10 GB to 175 GB.
- `README.md` and `SECURITY.md`: see [Post-migration cleanup](#post-migration-cleanup).
  They ship in this PR because the mirror is a CTAN mirror the moment it merges.
- Secrets: none. Endpoints: none (dante, R2, the domain, healthchecks, as today).

Must not change: the pipeline's tasks (they are PR 2's), the retry policy, `TL_KEY`, the
tlnet excludes, the `.state/` key, `site/index.html`'s content.

Verified before merge: `task plan RUN=<dir>` on the full listing must produce the 30-batch
table below; `task --dry sync` renders with the new `SOURCE` and `BUCKET`; `task guard`
against a canned listing with 600,001 lines must fail. `task smoke URL=file:///<dir>
STAGING=<dir>` with a `timestamp` file present.

Verified after merge: the seed's per-run `report` rows, [below](#the-seed). The mirror is
proven finished by the three-way `comm` in [Verifying the seed finished](#verifying-the-seed-finished).

Rollback: revert, which restores tlnet scope, then clean the bucket. The reverted
reconcile lists only under the prefix, so it never sees the 479k non-tlnet keys; they would
bill $1.84/month until deleted. One command, deletes free, ~497 listing pages:

```sh
aws s3 rm --recursive s3://tlnet/ --exclude 'systems/texlive/tlnet/*' --exclude '.state/*' --exclude '.site/*'
```

then `aws s3 cp {{.STATE}} - | xz -dc | grep '^systems/texlive/tlnet/' | xz -6 |
aws s3 cp - {{.STATE}}` to trim the state to tlnet, point the Transform Rule back at
`/index.html` (the reverted `page` overwrites CTAN's copy there on its next run), and
`aws s3 rm s3://tlnet/.site/index.html`. Rolling back mid-seed is the same: the state file is
always exactly the batches that landed.

### PR 6: `docs(mirror): register as an official CTAN mirror`

What changes: `README.md` gains the official-mirror paragraph and the `mirror.ctan.org`
sentence; `CLAUDE.md` gains "Registered at ctan.org/mirrors; mirmon probes `/timestamp`;
a mirror older than 28 hours is dropped from the rotation" as a must-know; `SECURITY.md`
gets its one line. No code. The registration itself is a form, not a commit, and happens
after this merges so the README the CTAN team reads is the final one.

Rollback: nothing to roll back.

## Bucket layout during and after

| Key | Today | During the seed | After |
|---|---|---|---|
| `systems/texlive/tlnet/...` (16,974) | present | present, untouched unless upstream changes | same key, same bytes |
| everything else CTAN (479,175) | absent | arriving in batch order | present |
| `index.html` | ours, `no-cache`, 4 kB | ours until batch 4 lands (`LC_ALL=C` puts lowercase `index.html` after `fonts/`, `graphics/` and `help/`) | CTAN's, 10,366 bytes |
| `.site/index.html` | absent | written by `page` from PR 5's first run; `/` rewrites to it | ours, `no-cache` |
| `.state/applied.txt.xz` | absent (PR 2 creates it) | rewritten after every batch | 3.1 MB, rewritten hourly |

No move is needed. Today's keys are already CTAN-relative: `BUCKET` is
`s3://tlnet/systems/texlive/tlnet` and the full mirror writes `systems/texlive/tlnet/...`
from the root. Verified by reading `Taskfile.yml` (`BUCKET: s3://tlnet/{{.PREFIX}}`) and
the `upload:` lines in run 32997802111 (`to s3://tlnet/systems/texlive/tlnet/tlpkg/...`).

`.state/` and `.site/`: dot-prefixed keys at the bucket root. CTAN has no top-level path
beginning with `.` (`awk '$5 ~ /^\./ && $5 != "."' deref.norm` prints nothing; 203 lines
contain the substring `state` and none is `.state`), so the exclusions cannot collide.
Every step that lists the bucket, which after PR 2 is only the daily reconcile, pipes
through `grep -v -e '^\.state/' -e '^\.site/'` before the `comm`. Today's `stale` never
sees either because it lists
under the prefix. Specified as a must-know in PR 2. The key is also served at
`https://ctan.ijosh.com/.state/applied.txt.xz` (404 today, 200 after PR 2); that is a
3 MB public file of paths, sizes and mtimes, all already public on every mirror, so it is
not a leak, and `caching.md` should keep it out of the cache rule so a reader gets the
current one.

`index.html`: CTAN ships one at the root: `-rw-rw-r-- 10366 2020/03/31 06:01:06
index.html` in the listing (`awk '$1=="-" && $5 !~ /\//' deref.norm`). The seed's batch 4
writes it to `/index.html`; from then on the reconcile keeps CTAN's copy there, so the
custom page cannot stay at that key. `official-mirror-and-url.md` adopted: `page` uploads
to `/.site/index.html`, a non-CTAN key excluded from listings exactly like `.state/`, and
the Transform Rule rewrites `/` to `/.site/index.html`. A visitor to the host root sees
the landing page; `https://ctan.ijosh.com/index.html` is CTAN's file, as on every mirror.
PR 5 changes `page`'s destination key and the rule; the seed changes nothing about it.
What CTAN's `index.html` contains was not fetched (it is on the mirrors, not on ctan.org);
unverified, one `curl -sL https://mirror.ctan.org/index.html` will show it.

Directory URLs: R2 returns 404 for `/systems/texlive/tlnet/` (checked today).
`official-mirror-and-url.md` adopted a Single Redirect, `path ne "/" and ends_with(path,
"/")` to `https://ctan.org/tex-archive<path>`, so a directory URL lands on CTAN's own
browser for that directory. It is dashboard state like the Transform Rule and does not
affect the seed.

## The seed

### The delta

After PR 2 the state file holds tlnet. The expansion run's `comm -13` is therefore the
stored set minus tlnet, plus `timestamp` (which changes hourly and is never in any state):

| Quantity | Value | Command |
|---|---|---|
| Stored set (all of CTAN as stored) | 496,149 objects, 132,993,291,537 bytes (132.99 GB) | `awk '$1=="-" && !($5 ~ /^systems\/texlive\/tlnet\/.*\.r[0-9]+.*\.tar\.xz$/) && !($5 ~ /update-tlmgr-r/) {n++; s+=$2} END {print n, s}' deref.norm` |
| Seed delta (stored set minus tlnet, plus `timestamp`) | **479,175 objects, 126,210,130,687 bytes (126.21 GB)** | `grep -vP '\tsystems/texlive/tlnet/' ordered.tsv` |
| Batches at 4 GB, lone over 4.995 GiB, `timestamp` last | **30** = 24 packed + 5 lone + 1 | `plan.awk` above |
| Lone-file batches | 5: `MacTeX.pkg`, `mactex-20260324.pkg` (6,865,013,189 B), `texlive.iso`, `texlive2026.iso`, `texlive2026-20260301.iso` (6,784,798,720 B) | `awk -F'\t' '$2 > 5237247590' applied.txt` |
| State file after the seed | 496,149 lines, 37.7 MB text, 3.1 MB `.xz` | `xz -6 \| wc -c` |

The base plan's 496,155 / 133.01 GB counted the six `update-tlmgr-r79982.*` files that
`fetch` excludes today; the corrected stored set is 496,149 / 132.99 GB. Its "~34 batches"
counted tlnet's 6.8 GB in the seed; the seed delta is 30 because tlnet is already there.

The 30 batches, listing order with the ranks applied, per-batch estimates at the slow
rates (22.7 MB/s, 57 files/s, 96 PutObject/s) and the fast ones (152 MB/s, 380 files/s,
207 PutObject/s), fetch and upload in series:

| Batch | Files | GB | First key | Slow, min | Fast, min |
|---:|---:|---:|---|---:|---:|
| 1 | 39,145 | 3.98 | `CTAN.sites` | 18 | 5 |
| 2 | 97,569 | 4.00 | `fonts/aboensis.zip` | 46 | 12 |
| 3 | 91,900 | 4.00 | `fonts/mpfonts/type3/...` | 43 | 11 |
| 4 | 37,281 | 4.00 | `graphics/metapost/...` | 17 | 5 |
| 5 | 33,003 | 3.98 | `info/lshort/...` | 15 | 4 |
| 6 | 28,073 | 4.00 | `macros/lamstex/...` | 13 | 4 |
| 7 | 30,778 | 4.00 | `macros/latex/contrib/pax/...` | 14 | 4 |
| 8 | 32,210 | 4.00 | `macros/latex2e/contrib/eqexam/...` | 15 | 4 |
| 9 | 32,111 | 3.96 | `macros/luatex/...` | 15 | 4 |
| 10 | 6,153 | 3.99 | `obsolete/systems/windows/protext/...` | 4 | 1 |
| 11 | 6,921 | 3.94 | `support/pdbf-toolkit/...` | 4 | 1 |
| 12 | 779 | 3.98 | `systems/texlive/tlcontrib/...` | 3 | 0.5 |
| 13 to 23 | 318 to 5,847 each | 3.93 to 4.00 | `systems/win32/...`, `systems/windows/...` | 3 to 4 each | 0.5 to 1 each |
| 24 | 5,265 | 0.43 | `usergrps/gutenberg/...` | 2.5 | 0.7 |
| 25 to 29 | 1 each | 6.87, 6.87, 6.78, 6.78, 6.78 | the five installers | 5 + upload each | 0.8 + upload each |
| 30 | 1 | 0.00 | `timestamp` | 0 | 0 |
| **Total** | **479,175** | **126.21** | | **274** | **67** |

The rsync per-file rates need a caveat. 57 files/s is the 05:15 run's 16,974 files over
299 s, but that run was bandwidth-bound (22.7 MB/s on 400 kB files), so it is a floor
that mixes two limits; 380 files/s is the 18:05 run's rate and is the only per-file
number measured on small files. `fonts/` averages 39,930 bytes a file (179,808 files,
7.18 GB), so batches 2 and 3 are where the model is least trustworthy: at 57 files/s they
are 45 minutes each, at 380 they are 12. The rehearsal measures this before it matters.
Multipart upload throughput from a runner to R2 was not measured; unverified, ~1 to 2 min
per installer is assumed, which adds 5 to 10 minutes to both columns.

### Resume, cut-off, concurrency

Each hourly run does the same five things (list, state, `comm`, batches, state) and stops
after `MAX_BATCHES` batches, then runs `smoke`, `report`, `ping` and exits 0. The report
says how many batches remain. The next hourly run recomputes the delta and continues. A
run that dies mid-batch (runner lost, 6-hour kill, an rsync exit 23 because dante moved a
file) leaves the state as of the last completed batch; the interrupted batch repeats in
full next hour. `PutObject` of the same bytes is idempotent, so nothing is retried unsafely.

With `MAX_BATCHES=4` and `timeout-minutes: 350`: 4 × 46 min at the slow rate is 184 min,
inside the timeout with room, so no scheduled seed run is ever killed; at the fast rate
4 batches is 25 to 50 min. A 184-minute run overlaps two or three cron slots, which the
concurrency rule below turns into one pending run. `MAX_BATCHES=4` makes the seed 8 runs
(30 / 4, rounded up), the last two of which are fast (lone files and `timestamp`).

The interaction with the hourly cron during a long run: `concurrency: sync` with the
default `queue: single`. Verified today on docs.github.com: "By default only one run can
be pending in a concurrency group; any additional pending runs cancel the previous one",
and pending runs start "in first-in-first-out order according to the time each one started
waiting". So if a `workflow_dispatch` seed runs for 5 hours, the 5 hourly runs that fire
meanwhile collapse to one pending run, which starts the moment the seed job ends. The
dropped ones lose nothing: every run recomputes the whole delta from the state file. This
is the behaviour wanted; do not add `queue: max`, which would run the dropped hours one
after another for no gain.

`timeout-minutes`: 350 always, from PR 3 on. GitHub's hard limit is 6 hours per job on
hosted runners (limits page); 350 sits under it, and a dispatch with `max_batches: 30` at
the slow rate (274 min) fits. The value is not tied to the input: one number, no
expression, nothing to trim after the seed.

### Wall clock

| Scenario | Runner time | Runs | Wall clock |
|---|---|---|---|
| Hourly, `MAX_BATCHES=4`, fast rates | 67 min + 8 × ~1 min overhead | 8 | first run at merge + 7 hours: **~8 h** |
| Hourly, `MAX_BATCHES=4`, slow rates | 274 min | 8; a 184-min run swallows two cron slots, so the next run starts when it ends | **~9 to 11 h** |
| Dispatch `max_batches=30`, fast | 67 min | 1 | ~1.5 h |
| Dispatch `max_batches=30`, slow | 274 min | 1 | ~5 h, inside the 350-minute timeout with ~75 min to spare |

P50 about 9 hours from merge to the `timestamp` batch, P90 about 11, if the seed is left
to the hourly runs. A dispatch finishes the same afternoon. Both are one day; the choice
is how much attention the day gets.

### The `workflow_dispatch` input

Recommended: add it, default it to the hourly value, and use it only on the seed day.

```yaml
on:
  schedule:
    - cron: 'M * * * *'
  workflow_dispatch:
    inputs:
      max_batches:
        description: Batches this run may publish (4 GB each; the hourly default is 4)
        type: number
        default: 4
jobs:
  sync:
    timeout-minutes: 350
    steps:
      - run: task sync MAX_BATCHES=${{ inputs.max_batches || 4 }}
```

`inputs` is available to `steps.run` (contexts page); on a `schedule` event
`inputs.max_batches` is empty, so the `|| 4` fallback applies. The number
type and default are per the workflow-syntax page ("the default value of the input is 0
for a number" when unset, so the explicit default matters).

Why not just leave the seed to the hourly runs and skip the input: the hourly path is
enough and this file recommends it as the baseline. The input exists so the seed day can
be finished in one sitting while someone watches, and so a future re-seed (a bucket
rebuilt from scratch, a 200 GB CTAN) does not need a workflow edit. It is four lines.

## Seed ordering within the tree

The ordering rule is the packer's rank, applied to every run, seed or hourly:

| Rank | Keys | Why |
|---:|---|---|
| 0 | `systems/texlive/tlnet/archive/` | containers before the tlpdb that names them, today's rule |
| 1 | `systems/texlive/tlnet/` except `tlpkg/` | installers and updaters, signed and checked |
| 2 | everything else, key order | no ordering constraint exists; key order keeps a batch inside one or two top-level directories, which makes a half-seeded mirror legible |
| 3 | files over 4.995 GiB | lone batches, at the end so a multipart failure does not block the ordinary files |
| 4 | `systems/texlive/tlnet/tlpkg/` and `timestamp` | the two "the tree is now complete" signals, always last |

`timestamp` last: mirmon reads it and takes the first word as the mirror's age. Before
registration nobody reads it, so during the seed it does not matter; after registration
it matters every hour, and the rule is the same rule, so nothing changes at registration.
It is 186 bytes and changes every hour at :02, so it is in every delta.

Should tlnet go first in the seed? In the seed there are no tlnet keys to order: the
state already holds them and only the hour's tlnet changes appear, ranked 0 and 1 as
always. The live prefix is therefore refreshed by every seed run exactly as by every
hourly run. The base plan's concern ("`tlpkg/` in the last batch so the live tlpdb is
never ahead of `archive/`") holds by construction: within one run, an `archive/` change
is in a rank-0 batch and its tlpdb in the rank-4 batch, and across runs the state file is
never written for a batch that did not land.

What the seed does to tlnet users: nothing. Their keys are not touched. `verify`'s
signed checks run in any run whose delta includes `tlpkg/` (a normal tlnet update day)
and are reported as "no tlnet keys in this delta" otherwise; `smoke` still reads
`texlive.tlpdb.sha512` back every run.

## Cost of the seed month

| Item | Count | Class | Cost |
|---|---:|---|---|
| Seed PutObject | 479,175 | A | |
| Multipart: 65 parts at 512 MB + 5 create + 5 complete | 75 | A | |
| State writes: 30 seed batches + 720 hourly | 750 | A | |
| Daily reconcile listing: 497 pages × 30 | 14,910 | A | |
| Hourly deltas (base plan's 18k/month at 133 GB) | ~18,000 | A | |
| **Class A total** | **~513,000** | | **$0** (free tier 1,000,000) |
| State reads and `smoke` GETs | ~1,500 | B | $0 |
| Storage, seeded on day 15 of a 30-day month | (14 × 6.8 + 16 × 133) / 30 = 74.1 GB-month, billed as 75 | | (75 − 10) × $0.015 = **$0.97** |
| Storage, every later month | 133 GB-month | | (133 − 10) × $0.015 = $1.84 |

Prices and the "average of daily peaks, whole GB-months" rule are `cost-estimates.md`'s to
verify; the arithmetic is above. A second full seed in the same month (a rollback and
redo) is ~1.0M Class A, at the edge of the free tier; a third is ~$2.25. So: one seed a
month, and the rehearsal counts as part of it, not extra.

Dante's load: 126 GB once, over one day, in ~31 rsync connections (one listing plus one
`--files-from` per batch) each held for minutes, never two at once. Plus the hourly
listing already running from PR 3. A new `rsync -a` mirror pulls 69 GB of real files with
symlinks preserved; this design pulls dereferenced copies, so 126 GB, about 1.8× what
they expect from a new mirror. Say so.

The message. The register page says maintainers are enrolled in "the low-traffic mailing
list for our mirror maintainers" upon registration, so before the seed there is no list to
post to; the address on ctan.org/mirrors for mirror matters is `ctan@ctan.org` (verified
today). Send before the seed day:

> Subject: New CTAN mirror seeding from rsync.dante.ctan.org on <date>
>
> Hello,
>
> I am setting up an HTTPS CTAN mirror at https://ctan.ijosh.com/ (United States, hosted
> on Cloudflare R2) and intend to register it once it is complete. It already serves
> systems/texlive/tlnet/ and has synced that directory from rsync.dante.ctan.org daily
> since August 2026.
>
> The initial pull of the rest of the archive will run on <date> from about <hh:00> UTC,
> from a single client, one rsync connection at a time, in ~30 pulls of about 4 GB
> each. Because the mirror is object storage without symlinks, it fetches dereferenced
> copies (`rsync -L`), so the total is about 126 GB rather than 69. After that it lists
> the archive once an hour at minute <M> (about 7 seconds, ~19 MB) and fetches only what
> changed.
>
> If a different day or a lower rate would suit dante better, tell me and I will adjust.
>
> Josh Vaughen, <email>

## Dry runs and the rehearsal

A full seed into a throwaway prefix or a second bucket: 479k Class A, which with the real
seed in the same month is ~1.0M, the whole free tier, so any slip costs $4.50 per extra
million; and storage of 133 GB for the day or two it exists, which is (133 × 2) / 30 =
8.9 GB-month, ~$0.13, not the ~$2 the caller's brief guessed (storage is averaged over
the month's daily peaks, not charged per upload). The money is trivial; the operations
are not, and a throwaway prefix proves nothing the real seed's first batch does not.
Recommendation: no.

The rehearsal is the seed's own first batch. On the seed day, before the expansion PR is
merged, run the branch's workflow with `max_batches: 1` (or merge and let the first hourly
run do 4; the point is to look at the first report before the rest runs). Batch 1 is
39,145 files, 3.98 GB: the seven root files, `biblio/` and `bibliography/` (2,593 each),
`digests/` (1,776), `documentation/` (28,898), `dviware/` (3,234) and the first 44 files
of `fonts/`, ending at `fonts/LuxiMono/ul9ro8a.pfb`. It proves:

- `rsync --files-from` with 39k paths in one call, including the 444 paths with characters
  outside `[A-Za-z0-9._/+-]` (`lt~obsolete.m4`, `@code.tex`, 218 with `+`, 11 with `#`,
  `?` or `%`; none with a space), fetched into an empty `staging/`.
- The per-file rate on small files (`documentation/` averages 76 kB), which is the one
  number the duration model lacks.
- `aws s3 cp --recursive` of 39k objects and its PutObject rate on a full bucket.
- The state write at ~1.6 MB, and that the next run's `comm` sees batch 1 as done.
- `verify` with no tlnet keys in the delta, `smoke` on three sampled keys, `report`'s new
  rows, and that the hourly tlnet path still works in the same run.
- The root files `CTAN.sites`, `FILES.byname`, `FILES.byname.gz`, `FILES.last07days` and
  the three `README.*` served at the root next to the still-custom `index.html`.

What it does not prove: multipart (batch 25), the 97k-file batch (batch 2), CTAN's
`index.html` landing at `/index.html` while `/` keeps serving `/.site/index.html`
(batch 4), and a run cut off mid-batch. Batch 2 follows in the next run; multipart is checked by `head-object`
when batch 25 lands; the cut-off path was proven in PR 2's phase or is proven by
cancelling one seed run from the Actions page mid-batch and reading the next run's report
(the interrupted batch's count reappears in "remaining"). Do that once, on purpose.

`fonts/` alone as the rehearsal (the caller's suggestion, 179,808 files, 7.18 GB, 2 batches)
would need a filter variable that exists only for the rehearsal. Batch 1 covers the same
ground without one.

## Verifying the seed finished

Three listings must agree, on key and size:

```sh
# a: what the bucket holds (497 pages, Class A, free)
aws s3 ls --recursive s3://tlnet/ | awk '{print substr($0, index($0, $4)) "\t" $3}' \
  | grep -v -e '^\.state/' -e '^\.site/' | LC_ALL=C sort > a.txt
# b: what the state file says landed (key \t size \t mtime; drop the mtime)
aws s3 cp s3://tlnet/.state/applied.txt.xz - | xz -dc | cut -f1,2 | LC_ALL=C sort > b.txt
# c: what dante lists now (the same list task the run uses)
task list RUN=. && cut -f1,2 upstream.txt | LC_ALL=C sort > c.txt
LC_ALL=C comm -3 a.txt b.txt | head    # must be empty
LC_ALL=C comm -3 b.txt c.txt | head    # must be empty or only the hour's changes
```

The first `comm` empty means the bucket is exactly the state; the second empty means the
state is exactly upstream. The 03:xx reconcile does the first one every day and its
report row (`0 mismatched, 0 unknown`) is the routine version of this check.

`smoke` per top-level directory, once: for each of the 24 top-level names in the listing
(`awk -F'\t' '{split($1,a,"/"); print a[1]}' applied.txt | sort -u`), one random key read
through the domain and `cmp`'d against a fresh `rsync --files-from` of the same key.
Twenty-four GETs, 24 one-file pulls.

The size spot check, one line, N random keys against the listing (paths have no spaces;
the 11 keys with `#`, `?` or `%` need `--data-urlencode`-style quoting, so skip them
with the `grep`):

```sh
shuf -n 200 c.txt | grep -v '[#?%]' | while IFS=$'\t' read -r key size; do
  got=$(curl -sI "https://ctan.ijosh.com/$key" | awk 'tolower($1)=="content-length:"{print $2+0}')
  [ "$got" = "$size" ] || echo "MISMATCH $key listed=$size served=$got"; done
```

Zero lines is a pass. 525 zero-byte files exist upstream and serve `content-length: 0`.

Mirmon's first probe: after registration, mirmon's `-get update` probes sites that are
"new" on its next run (mirmon manual: "the subset contains the sites that are new, bad
and/or not probed for a specified time"); CTAN's instance ran at 00:03 UTC today
(`last check: Thu Aug 27 00:03:01 2026`), so hourly at :03. The probe reads
`https://ctan.ijosh.com/timestamp` and takes the first word; the histogram on CTAN's page
counts "0 ≤ age ≤ 28 hours" as fresh. The first green block should appear within an
hour of the listing being updated; how long the human step takes is unverified.

## Registration

The form at https://ctan.org/mirrors/register (read today from a copy fetched today), its
fields and what to put in them:

| Field (`name=`) | Required | Validation | Value |
|---|---|---|---|
| `name` | yes | ≤128 | `ctan.ijosh.com` ("usually: your host name") |
| `contactName` | yes | ≤128 | Josh Vaughen |
| `contactEmail` | yes | email, ≤255 | a mailbox that is read; the maintainers' list goes here |
| `country` | yes | select | United States of America |
| `region` | no | ≤128 | state |
| `location` | no | ≤128 | city |
| `mirrorsFrom` | | select, one option | rsync.dante.ctan.org (Germany) |
| `httpsAddress` | yes | `^[a-zA-Z][\/a-zA-Z0-9.-]*$`, ≤255, shown after a fixed `HTTPS://` | `ctan.ijosh.com` (the example `joshua.smcvt.edu/tex-archive` has no trailing slash; the regex allows one) |
| `httpAddress` | no | same | empty; the site is HTTPS only and the page says redirection is HTTPS only since April 2021 |
| `ftpAddress` | no | same | empty |
| `rsyncAddress` | no | same | empty; R2 has no rsync |
| `notes` | no | ≤4096 | see below |

Notes text: "Object-storage mirror (Cloudflare R2) synced hourly at minute M from
rsync.dante.ctan.org via a listing diff, not a persistent rsync tree. Every CTAN path is
served verbatim at the root; the tree is dereferenced, so alias directories and symlinked
files are real copies. There is no autoindex; a directory URL redirects to the same
directory on ctan.org/tex-archive, and every file and .zip bundle is served directly.
Pipeline is public: github.com/jshvn/ctan."

The page states the enrolment: "You will be enrolled in the low-traffic mailing list for
our mirror maintainers." The list's address is not on the page; unverified, the welcome
mail will name it.

Timeline: unverified. The page says nothing about review time. The observable steps are:
the entry appears in `CTAN.sites` (root file, 18,398 bytes, dated 2026-08-25 16:21 in the
listing, so it is regenerated when the list changes, not daily), mirmon lists the host
with a "new" state, then a green age after its first successful probe, then
`mirror.ctan.org` redirects begin. Watch `curl -sI https://mirror.ctan.org/timestamp`
(307 to a mirror; today `mirror.clarkson.edu`) for the hostname over a few days.

If rejected or asked to change something: the plausible objections are the missing
directory indexes (the example Apache config has `+Indexes` and `DirectoryIndex disabled`;
the rules text does not require them), the missing rsync (optional on the form), and the
non-standard sync method (the rules say "Mirror from the primary CTAN node" and "at least
once per hour", both met). The index question is `official-mirror-and-url.md`'s; the
answer to "rsync?" is no, and it is optional. Nothing in a rejection undoes the mirror:
it keeps running as an unofficial full mirror, which is the state between PR 5 and
registration anyway.

## Post-migration cleanup

After the seed and registration, the tree describes the mirror on its own terms.

Tasks deleted: `stale` (PR 2), the `du`-based `guard` (PR 2). Tasks rewritten: `fetch`,
`publish`, `verify`, `smoke`, `report`, `page` (destination `/.site/index.html`). Tasks
added: `list`, `state`, `plan`, `delete`, `reconcile`, `retry`.

Vars deleted: `LIMIT_MB`. Vars changed: `SOURCE`, `PREFIX` (empty; consider
deleting it and writing `s3://tlnet` directly), `BUCKET`, `URL`. Vars added: `STATE`,
`LIMIT` (bytes and objects), `CAP`, `LONE`, `MAX_BATCHES`, `CURL`.

`ponytail:` comments: the one in `publish` ("one DeleteObject per key ... Batch with
s3api delete-objects") is resolved by `delete` in PR 2 and removed. New ones to write:
in `plan`, "batches are packed in key order, not bin-packed; ceiling: a 4 GB batch of
97k files takes up to 45 min at dante's slow rate; upgrade: a second rank for
`fonts/` when a batch's file count exceeds 60k"; in `reconcile`, "one full listing a day;
ceiling: 497 Class A pages at 133 GB, 868 at 200 GB; upgrade: none needed under 1M".

`CLAUDE.md`, by section:

- Constraints: "Zero running cost" rewritten to "storage is the only bill"; "Objects stay
  under `systems/texlive/tlnet/`" replaced by the root-paths sentence; the secrets line
  changes only if `caching.md`'s PR adds `CF_ZONE_ID` and `CF_API_TOKEN`; "Network
  endpoints" likewise.
- Must knows kept verbatim: "No versioned containers", "Never `--size-only`" (rewritten to
  say the state file compares size and mtime from the listing, which is the same
  guarantee), "Do not trust the job log for counts".
- Rewritten: "Never `aws s3 sync --delete`" becomes "The hourly path never lists the
  bucket"; "Single-part uploads" gains the five exceptions and the 4 GB threshold;
  "`publish` uploads in a fixed order" becomes the rank table; "The steps after `smoke`"
  stays; "A failed run is the only alert" gains "a run that stops at `MAX_BATCHES` is not
  a failure; `report` says what remains".
- Rewritten: "`index.html` lives at the bucket root" becomes "the landing page is
  `/.site/index.html` and the Transform Rule points `/` at it; `/index.html` is CTAN's".
- Added: "`.state/` and `.site/` are the two keys outside CTAN's paths; every listing
  excludes them";
  "`timestamp` is in the last batch of every run; mirmon drops the mirror at 28 hours";
  "`MAX_BATCHES` bounds a run to the timeout; the seed is the hourly runs after a merge
  that enlarges the delta".
- "Verifying a change": the canned checks for `plan`, `reconcile`, `smoke` with
  `timestamp`, and `guard` on a listing.

`README.md` outline: title "A CTAN mirror on Cloudflare R2"; one paragraph on what it is
and the URL; "TeX Live" section unchanged in substance (the `tlmgr` commands, the
freshness `curl`); "How it works" rewritten around the hourly listing diff and the daily
reconcile, with the seed in one sentence; "Is it fresh?" adds `curl -s
https://ctan.ijosh.com/timestamp` and the mirmon page; "Want your own?" rewritten (bucket,
token, domain, Transform Rule, the seed takes a day, ~$2/month); the healthchecks badge
stays. The `SECURITY.md` line: "Only TeX Live signs what it publishes. `systems/texlive/
tlnet/` is verified as described above; the rest of CTAN carries no upstream signature
and is copied byte for byte from the master as listed, so a tampered master is a
tampered mirror, as on every CTAN mirror." The "Uploads are not atomic" paragraph is
rewritten for the batch order and hourly cadence.

## Go/no-go checklists

Seed day, before the first run with a large delta:

- [ ] PRs 1 to 4 merged; at least 7 days of green hourly runs on tlnet scope; the last
      seven 03:xx reconciles report `0 mismatched, 0 unknown`.
- [ ] One hourly run was cancelled from the Actions page mid-batch on purpose and the next
      run's report showed the batch repeated and the state consistent.
- [ ] `task plan RUN=<dir>` on today's fresh listing (`rsync -rL --list-only`, 7 s) gives
      30 batches, `tlpkg/` and `timestamp` last, 5 lone; the count of files over `LONE`
      is 5 and the largest is under the runner's free disk (check `df -h /home/runner`
      in a run's log; unverified today, `limits.md`).
- [ ] The R2 dashboard shows Class A month-to-date under 100k, so the seed's 480k fits
      with room; it is after the 10th of the month, so a redo would not push into a
      second billing month's prorating surprise.
- [ ] The message to `ctan@ctan.org` was sent at least two days before.
- [ ] It is morning UTC on a weekday, so the eight runs finish while someone is awake and
      dante's staff are too.
- [ ] The `sync` check on healthchecks.io is paused for the day (`monitoring.md`), so a
      three-hour seed run does not page; the `reconcile` check stays live. Unpause after
      the `timestamp` batch lands.
- [ ] The Transform Rule will be retargeted to `/.site/index.html` right after PR 5
      merges, before batch 4.
- [ ] PR 5 is approved with `max_batches` defaulting to 4; the rehearsal will be its first
      dispatched run with `max_batches: 1` from the merged `main`.

Stop the seed (`gh run cancel`, then revert PR 5 and run the cleanup command) if: a batch
fails `verify`; `smoke` fails on a sampled key; the reconcile finds unknown keys; a run's
report shows a PutObject count that does not match its batch line count; Class A passes
900k.

Registration day:

- [ ] Three-way `comm` empty on both sides, run by hand and pasted into the PR 6
      description.
- [ ] Size spot check of 200 random keys: zero mismatches.
- [ ] 24-directory `smoke` pass.
- [ ] `https://ctan.ijosh.com/` shows the landing page from `/.site/index.html`,
      `/index.html` is CTAN's 10,366-byte file; `/timestamp` is under one hour old;
      `/CTAN.sites`, `/FILES.byname.gz`, `/tds.zip` serve with the listing's sizes.
- [ ] Seven consecutive days of green hourly runs on the full delta since the seed, and
      seven reconciles at zero.
- [ ] Storage on the R2 dashboard ≈ 133 GB, objects ≈ 496k (a few hundred either way as
      upstream moves).
- [ ] PR 6 merged, so the README the reviewers read is final.
- [ ] Then the form, then watch mirmon at :03.

## Where this differs from the base plan

- **Stored set** is 496,149 objects, 132.99 GB, not 496,155 / 133.01: the plan's count
  kept the six `update-tlmgr-r79982.*` files that `fetch` excludes today.
- **The seed delta** is 479,175 objects, 126.21 GB, 30 batches, not "496,155 lines,
  133 GB, ~34 batches". The plan's seed re-put tlnet; with PR 2 ahead of PR 5 the state
  file already holds tlnet and nothing under it is re-uploaded.
- **The seed is the hourly runs**, bounded by `MAX_BATCHES`, not one 4-to-6-hour run
  racing the 6-hour cap. The plan's "a cut-off at batch ~30 and a second run" becomes
  eight clean runs with reports, or one dispatch. A run stopping at its batch limit exits
  0 and pings; the plan treated the kill as the checkpoint.
- **Dante is not one speed.** The plan's 22.7 MB/s is the slow run; the same day's 18:05
  run did 152 MB/s. Both are stated and both bound the estimate (67 to 274 minutes of
  runner time). The plan's "4 to 6 hours" is inside the slow end.
- **No pre-seed post to the maintainers' list**: there is no list membership before
  registration (the register page enrols on registration). The pre-seed note goes to
  `ctan@ctan.org`.
- **Dante's load is 126 GB, not 133**: tlnet is already here.
- **Cache rules and purge come after the seed**, not before. The plan ordered them as
  step 7 of 12, before the seed; seeding through a purge step spends ~25 minutes purging
  URLs never served.
- **`page` changes key in PR 5** rather than going away: CTAN's root `index.html` is in
  batch 4 and the reconcile would fight a `page` that still wrote `/index.html`, so the
  landing page moves to `/.site/index.html` and the Transform Rule follows.
- **`timestamp` is in the last batch** with `tlpkg/`; the plan put `tlpkg/` last and did
  not place `timestamp`.
- **Multipart is one `aws.config`** with a 4 GB threshold and 512 MB chunks, 13 parts and
  15 Class A per installer; the plan mentioned the chunk size as an option.
- **`retry_mode = standard`**, not the plan's `adaptive`.
- **`timeout-minutes: 350` permanently**, not "360 for the seed and trimmed afterwards".
- **A missing state file fails the run** unless `SEED=true`; the plan let an absent state
  silently mean "seed".
- **Concurrency semantics are now verified** (`queue: single`, FIFO) rather than assumed;
  the conclusion matches the plan's.
- **The rehearsal is `max_batches: 1`**, not a throwaway prefix; the dry-run cost the
  plan did not estimate is ~$0.13 storage but half the Class A tier.
- Nowhere else. The plan's consistency model (state written after the batch lands, every
  write idempotent, the interrupted batch repeats) is kept exactly.

## Open questions

- Free disk on `ubuntu-latest` at job start: the 14 GB figure is the documented SSD; the
  6.87 GB installer plus the two listings must fit, and whether `df` reports more in
  practice is `limits.md`'s to measure from a run's log.
- Whether `rsync --files-from` with 97k paths (batch 2) hits any daemon-side limit on the
  file list; the rehearsal answers it one batch late. If it does, halve `CAP` for batches
  over 50k files (one more `awk` condition).
- How `verify` checks an `archive/` container in a batch whose delta has no `tlpkg/`:
  against the state's previous tlpdb (held where?) or by trusting the listing;
  `verification-and-security.md` must specify, and PR 2's report row depends on it.
- The name of the maintainers' list and how long registration takes; the first is in the
  welcome mail, the second is observed once.
- Whether CTAN's `index.html` is a plain directory page or references assets that the
  mirror must also serve (it is 10 kB and dated 2020, so likely self-contained);
  `official-mirror-and-url.md`.
- Whether to keep a `workflow_dispatch` input at all once the seed is done, or delete it
  in PR 6 to keep the workflow at one trigger plus dispatch-with-no-inputs.
- Multipart upload throughput from a runner to R2, which decides whether the five
  installers are 5 minutes or 30; measured by batch 25.

## Sources

- `Taskfile.yml`, `aws.config`, `.github/workflows/sync.yml`, `README.md`, `SECURITY.md`,
  `CLAUDE.md` in this repository, read today.
- `gh run view 32933376123 --log` and `gh run view 32997802111 --log` (sync.yml, 2026-08-26
  05:15 and 18:05 UTC); `gh run view 32940368883` (06:56 UTC, failed at `smoke` with a 404,
  the `sync --delete` ordering bug fixed in #9/#11).
- `SCRATCH/ctan-list-deref.txt`, normalised with `SCRATCH/norm.awk`; batch plans with
  `SCRATCH/plan.awk`; all commands inline above.
- https://ctan.org/mirrors/register (fetched today; form fields, validation regex, rules,
  enrolment sentence)
- https://ctan.org/mirrors (fetched today; `ctan@ctan.org`, monitoring statement)
- https://ctan.org/mirrors/mirmon (fetched today; probe time, 28-hour histogram)
- https://www.mankier.com/1/mirmon (fetched today; new-site probing, min/max poll, timestamp probe)
- https://docs.github.com/en/actions/concepts/workflows-and-actions/concurrency and
  https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency
  (fetched today; `queue: single` default, FIFO, `queue: max`)
- https://docs.github.com/en/actions/reference/limits (fetched today; 6 hours per job)
- https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
  (fetched today; `workflow_dispatch` input types and defaults)
- https://docs.github.com/en/actions/reference/workflows-and-actions/contexts (fetched
  today; `inputs` available to `steps.run`)
- https://developers.cloudflare.com/r2/platform/limits/ (cached copy from today; 4.995 GiB
  single part, 10,000 parts, 1 write/s per key)
- https://developers.cloudflare.com/r2/objects/multipart-objects/ (cached copy from today;
  5 MiB to 5 GiB per part)
- https://developers.cloudflare.com/rules/transform/ (cached copy from today; 10 rules on Free)
- https://docs.aws.amazon.com/cli/latest/topic/s3-config.html (fetched today;
  `multipart_threshold`, `multipart_chunksize` minimum 5 MB and auto-adjustment)
- https://www.tug.org/texlive/acquire-mirror.html (cached copy from today; tlnet rsync recipe)
- `curl -sI` today against `https://ctan.ijosh.com/`, `/index.html`, `/timestamp`,
  `/.state/applied.txt.xz`, `/systems/texlive/tlnet/`, and `https://mirror.ctan.org/`,
  `/index.html`, `/timestamp` (307s to `mirror.math.princeton.edu`, `mirrors.mit.edu`,
  `mirror.clarkson.edu`; not followed).
