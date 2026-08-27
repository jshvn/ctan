# Taskfile architecture

The design of `Taskfile.yml` and the two workflows for a full CTAN mirror on R2, at the level
of a spec someone implements from. Numbers are computed from the listing taken on 2026-08-26
(`SCRATCH/ctan-list-deref.txt`, 538,289 lines) or verified against pages fetched the same day;
each is marked. Sibling files own the cost model (`cost-estimates.md`), the cache rules and
purge API (`caching.md`), the security argument (`verification-and-security.md`), the seed
day (`seeding-and-migration.md`), the state bootstrap (`sync-with-dante.md`), the landing
page and directory redirects (`official-mirror-and-url.md`) and the alert model
(`monitoring.md`).

## Key decisions

| Decision | Value | Basis |
|---|---|---|
| Stored set | 496,149 objects, 132,993,291,537 bytes (132.99 GB) | listing; `rsync -rL` minus tlnet's `*.r[0-9]*.tar.xz` (14,872) and `update-tlmgr-r*` (6) |
| State line format | `path TAB size TAB yyyy/mm/dd hh:mm:ss`, `LC_ALL=C` sorted | path first so whole-line order equals path order; `comm` needs that (demonstrated below) |
| Hourly diff | `comm -13 applied upstream` (new or changed lines), `comm -23` on path columns (deleted) | 0.1 s on 496k lines |
| Batch cap | 4 GB decimal from listing sizes; a file over the cap is its own batch without flushing the running one; the decision batch (tlnet `tlpkg/` and the root files) is last, `timestamp` uploaded last of all | 30 batches on the seed after a state bootstrap: 24 packed, 5 lone installers, 1 decision |
| Batches per run | `MAX_BATCHES`, default 4; the rest wait for the next hour | a seed is ~8 hourly runs of ~25 minutes; nothing runs to the 6-hour limit |
| Peak disk | max(4.00 GB, 6.87 GB) + ~0.2 GB text = 7.1 GB | measured every run with `df`; the run refuses a batch it cannot hold |
| Multipart | one `aws.config`: `multipart_threshold = 4GB`, `multipart_chunksize = 512MB`, `retry_mode = standard` | 13 parts per installer, 15 Class A each; no per-call override needed |
| Loop | Task `for: {var: BATCHES, split: "\n"}` over the first `MAX_BATCHES` of `RUN/batch-NNNN.txt`, one `task: batch` per item | verified on Task 3.53.1; minimum 3.28.0 |
| Verify | tlnet control files fetched once per run into `RUN/tl/` (2.9 MB), signed and pinned; each batch's containers checked against that tlpdb; before the tlpdb goes live, every container it names must be in the bucket after this run | containers land in earlier batches than the tlpdb |
| State write | after each batch's upload and purge; one `PutObject` of a 3.1 MB `.xz` | at-least-once; a cut-off repeats one batch |
| Lost state | `RECONCILE=true` rebuilds it from a bucket listing; `SEED=true` declares the bucket empty; neither given, the run fails | the seed never re-puts the 16,974 live tlnet keys |
| Reconcile | daily at 03 UTC or `RECONCILE=true`: bucket listing joined to upstream on key and size becomes the state; keys outside `.state/` and `.site/` that upstream lacks are deleted | the only place the bucket is listed |
| Cache | `CACHE=off` by default: one bypass rule, no purges; `CACHE=on`: the cache rule, purge of changed and deleted keys only | `caching.md` |
| Removed tasks | `guard` (replaced by `plan`'s ceilings), `stale` (replaced by `diff` and `reconcile`) | |
| Kept tasks | `page`, now to `.site/index.html`; CTAN's own `index.html` (10,366 bytes, 2020-03-31) is stored at `/index.html` | `official-mirror-and-url.md` |
| Workflow | `cron: '41 * * * *'`, `concurrency` group `sync` with `cancel-in-progress: false`, `timeout-minutes: 350` always, inputs `seed`, `reconcile`, `max_batches`, `cache` | scheduled runs start 15 to 45 minutes after their slot and a slot can be dropped; `report` prints each run's lateness |
| Taskfile length | 412 lines of the 800 budget; parses and dry-renders under Task 3.53.1 | the YAML below, extracted and run with `task --dry --force sync` |

## 1. The task graph

`task sync` runs, in this order and nothing in parallel:

```
clock -> rules -> list -> state -> rebuild? -> diff -> plan -> tlpdb -> batches -> delete -> reconcile? -> smoke -> report -> ping -> page
                                                                          |
                                                                 batch (first MAX_BATCHES of RUN/batch-NNNN.txt, in file order):
                                                                   fetch -> verify -> publish -> purge? -> checkpoint
```

`?` marks a task skipped by its `status` check: `rebuild` runs only when `state` asked for
it, `reconcile` daily or on request, `purge` only with `CACHE=on`.

Each task, its inputs and outputs, and the rule that fixes its place:

| Task | `desc` | Reads | Writes | Ordering rule |
|---|---|---|---|---|
| `clock` | Record the start and its lateness against the cron slot | `date -u`, `CRON_MINUTE` | `RUN/start.txt`, `RUN/late.txt` | first: everything after it is timed from here |
| `rules` | Put the zone's rulesets in place; `PUT` only when the file's sha256 differs from the version on the zone | repo JSON, `CF_*` env | nothing | a bad token or JSON fails before anything is fetched (`caching.md`) |
| `list` | List dante, normalise to `RUN/upstream.txt` | `SOURCE` | `RUN/listing.txt`, `RUN/upstream.txt` | before `state`; refuses a listing under 400k lines |
| `state` | Fetch `.state/applied.txt.xz` to `RUN/applied.txt`, or decide what a missing one means | `S3`, `SEED`, `RECONCILE` | `RUN/applied.txt` or `RUN/rebuild-now` | before `diff` |
| `rebuild` | List the bucket and make the state exactly what is there | `S3`, `upstream.txt` | `RUN/bucket.txt`, `applied.txt`, the state object | when `RUN/rebuild-now` exists: a lost state, or the daily reconcile |
| `diff` | `changed.txt`, `deleted.txt`, `paths.txt` | `upstream.txt`, `applied.txt` | three files in `RUN` | pure text; no network |
| `plan` | Ceilings, disk check, `RUN/batch-NNNN.txt` | `upstream.txt`, `changed.txt`, `df` | batch files; empties `STAGING` | before `batches`; fails on a file the disk cannot hold |
| `tlpdb` | Fetch and pin tlnet's control files once per run | `changed.txt`, `SOURCE` | `RUN/tl/tlpkg/*` | skipped when no `systems/texlive/tlnet/` line is in the delta; before any batch |
| `batches` | Loop | `RUN/batch-*.txt`, `MAX_BATCHES` | | one `batch` per file, in name order, at most `MAX_BATCHES` |
| `batch` | One batch end to end | `B` | | `fetch -> verify -> publish -> purge -> checkpoint`; skipped when `B` is empty |
| `fetch` | `rsync --files-from` into empty staging | `B`, `SOURCE` | `STAGING` | disk precondition first |
| `verify` | Delta-scoped tlnet checks; on the decision batch, the named-container check against the bucket after this run | `B`, `STAGING`, `RUN/tl`, `applied.txt`, `deleted.txt` | `RUN/tl/missing.txt` | before `publish`; a failure leaves the bucket at the last checkpoint |
| `publish` | `aws s3 cp --recursive`, then `timestamp` alone | `STAGING` | `RUN/publish.txt` (appended) | before `purge`; `timestamp` is the last key of the run to land |
| `purge` | Purge the batch's already-known URLs, 100 per call | `B`, `applied.txt` | `RUN/purge.txt` | before `checkpoint`; only with `CACHE=on`; a failed purge is a failed run |
| `checkpoint` | Rewrite and upload the state; empty staging | `B`, `applied.txt` | `applied.txt`, `.state/applied.txt.xz`, empties `STAGING` | last in the batch: the state never names what did not land |
| `delete` | `DeleteObjects` 1,000 keys per call, purge, drop from state | `RUN/deleted.txt` | `RUN/publish.txt`, state | after every batch: the tlpdb never names a deleted container |
| `reconcile` | `rebuild`, then delete keys upstream and the reserved prefixes do not own | `S3`, `upstream.txt`, `late.txt` | `RUN/bucket.txt`, state | the run whose slot is in hour 03, however late it starts; after `delete` so the listing is of a settled bucket |
| `smoke` | Read back through the domain | `URL`, `upstream.txt`, `RUN/tl` | nothing | after every write; before `ping` |
| `report` | Job summary from `RUN` | `RUN/*` | `GITHUB_STEP_SUMMARY` | before `ping`, as today |
| `ping` | healthchecks | `HEALTHCHECK_URL` | nothing | after `smoke`, before `page` |
| `page` | `site/index.html` to `.site/index.html` | `site/index.html` | one key | last: a broken landing page is one email on a fresh mirror |
| `retry` | Wrap an rsync call | `CMD` | | only rsync uses it |

Why `guard` and `stale` do not survive, and why `page` does:

- `guard` measured `du` of a full staging tree. There is no full tree. Its two jobs, "never
  publish a tree that would bill" and "print the size every run", move to `plan`, which has
  the exact byte total of the upstream listing before anything is fetched, and adds the
  check the old `guard` could not make: a single file larger than the free disk.
- `stale` diffed a bucket listing against staging. The bucket is listed once a day, in
  `rebuild`; the hourly deletion list comes from `diff` with no listing at all. `stale`'s
  one safety rule, "an empty tree must never empty the bucket", survives as `list`'s line
  count floor and `state`'s refusal to guess what a missing state file means.
- `page` keeps the landing page, at a key CTAN can never produce: `.site/index.html`. The
  Transform Rule rewrites `/` to `/.site/index.html`. CTAN's own `index.html` (10,366
  bytes, mtime 2020-03-31 in the listing) is stored at `/index.html` like every other file,
  so a client asking for it by name gets what every mirror serves. `.state/` and `.site/`
  are the two reserved prefixes; `reconcile` never deletes under them and `list` can never
  produce them because CTAN has no dot directories at its root.

### The Taskfile

Real code, written against Task 3.53.1 and the tool list. Every awk program in it was run
against the 2026-08-26 listing (section 8 has the commands). `rules` and `purge` show their
shape; the JSON bodies, the sha256-in-description convention and the purge rules are
`caching.md`'s.

```yaml
# Mirror all of CTAN into a Cloudflare R2 bucket, hourly, from a list diff: dante's listing
# against the listing the last run left in the bucket. Runs the same way locally and in
# GitHub Actions. R2 is reached through the AWS CLI, configured by environment variables:
#   AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY   (R2 API token, Object Read & Write)
#   AWS_ENDPOINT_URL=https://<account-id>.r2.cloudflarestorage.com   AWS_REGION=auto
version: '3'

vars:
  SOURCE: rsync://rsync.dante.ctan.org/CTAN/   # the master; the register page asks for it
  HOST: ctan.ijosh.com                          # a fork changes this line
  BUCKET: tlnet                                 # bucket name; objects sit at its root under CTAN's own paths
  S3: s3://{{.BUCKET}}
  URL: https://{{.HOST}}
  STATE: .state/applied.txt.xz                  # the listing of what the bucket holds; never in upstream
  RUN: /tmp/ctan-run                            # run outputs; outside staging so nothing here is uploaded
  STAGING: '{{.ROOT_DIR}}/staging'
  TL: systems/texlive/tlnet                     # the one signed subtree
  TL_KEY: C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC   # tug.org/texlive/verify.html
  CRON_MINUTE: 41                               # the minute in sync.yml's cron; lint checks they agree
  CEILING_GB: 200                               # refuse to run past this many decimal GB upstream
  BATCH_GB: 4                                   # decimal GB per batch; a file over it is a batch by itself
  MAX_BATCHES: '{{.MAX_BATCHES | default "4"}}' # batches per run; the rest wait for the next hour
  SEED: '{{.SEED | default "false"}}'           # true: a missing state file means an empty bucket
  RECONCILE: '{{.RECONCILE | default "auto"}}'  # true, false, or auto (the run whose slot is in hour 03 UTC); true also rebuilds a missing state
  CACHE: '{{.CACHE | default "off"}}'           # off: one bypass rule, no purges; on: cache rule and purges
  RSYNC: rsync --timeout=300 --contimeout=60 --no-h
  CURL: curl -fsS --connect-timeout 15 --max-time 60 --retry 6 --retry-connrefused --retry-max-time 600
  AWS_FLAGS: --no-progress --cli-connect-timeout 60 --cli-read-timeout 300
  CF: https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID

env:
  AWS_CONFIG_FILE: '{{.ROOT_DIR}}/aws.config'

tasks:
  default:
    cmds: [{task: sync}]

  sync:
    desc: clock -> rules -> list -> state -> rebuild? -> diff -> plan -> tlpdb -> batches -> delete -> reconcile? -> smoke -> report -> ping -> page
    cmds:
      - {task: clock}
      - {task: rules}
      - {task: list}
      - {task: state}
      - {task: rebuild}
      - {task: diff}
      - {task: plan}
      - {task: tlpdb}
      - {task: batches}
      - {task: delete}
      - {task: reconcile}
      - {task: smoke}
      - {task: report}
      - {task: ping}
      - {task: page}

  clock:
    desc: Record the run's start (RUN/start.txt) and its lateness against the cron slot (RUN/late.txt)
    cmds:
      - mkdir -p {{.RUN}}
      - date -u +%s > {{.RUN}}/start.txt
      # Seconds since the last :CRON_MINUTE, and the hour that slot belongs to. Arithmetic on
      # the epoch only, so it is the same on macOS and Linux. Scheduled runs start 15 to 45
      # minutes after their slot; a run over an hour late reads as the next slot's.
      - >-
        awk -v m={{.CRON_MINUTE}} '{ late = ((int($1 / 60) % 60 - m + 60) % 60) * 60 + $1 % 60;
          printf "%d %d\n", late, int(($1 - late) / 3600) % 24 }' {{.RUN}}/start.txt > {{.RUN}}/late.txt

  rules:
    desc: Put each ruleset in cloudflare/ on the zone, but only when its sha256 differs from the version there (every PUT is a new version)
    vars:
      # file=phase. CACHE=off swaps the cache rule for the one bypass rule. Smart Tiered
      # Cache is a one-time zone setting (caching.md), not a call here.
      RULESETS: '{{if eq .CACHE "on"}}cache-rules{{else}}bypass-rules{{end}}=http_request_cache_settings transform-rules=http_request_transform redirect-rules=http_request_dynamic_redirect'
    cmds:
      - for: {var: RULESETS, as: R}
        cmd: >-
          f=cloudflare/${R%%=*}.json; phase=${R#*=};
          want=$(shasum -a 256 "$f" | cut -c1-64);
          have=$({{.CURL}} -H "Authorization: Bearer $CF_API_TOKEN" "{{.CF}}/rulesets/phases/$phase/entrypoint"
          | sed -n 's/.*"description": *"sha256:\([0-9a-f]*\)".*/\1/p' | head -1);
          test "$have" = "$want" ||
          {{.CURL}} -X PUT -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json"
          --data-binary @"$f" "{{.CF}}/rulesets/phases/$phase/entrypoint" -o /dev/null

  list:
    desc: List dante and normalise it to RUN/upstream.txt as "path TAB size TAB mtime", byte-sorted
    set: [pipefail]
    cmds:
      - mkdir -p {{.RUN}} {{.STAGING}}
      - task: retry
        vars: {CMD: '{{.RSYNC}} -rL --list-only {{.SOURCE}} > {{.RUN}}/listing.txt'}
      # Regular files only. The size column may carry commas (rsync 3.x) or be blank for a
      # zero-byte file (openrsync), so it is found from the fixed-width date, not by field.
      # tlnet's versioned containers and update-tlmgr-r* are dropped here, so they are never
      # fetched, never stored, and never in the state.
      - >-
        awk -v TL="{{.TL}}" '
          /^-/ && match($0, / [0-9][0-9][0-9][0-9]\/[0-9][0-9]\/[0-9][0-9] [0-9][0-9]:[0-9][0-9]:[0-9][0-9] /) {
            path = substr($0, RSTART + RLENGTH); mtime = substr($0, RSTART + 1, RLENGTH - 2)
            size = substr($0, 11, RSTART - 11); gsub(/[ ,]/, "", size)
            if (path ~ "^" TL "/archive/.*\\.r[0-9]+\\.tar\\.xz$" || path ~ "^" TL "/update-tlmgr-r") next
            print path "\t" size + 0 "\t" mtime }' {{.RUN}}/listing.txt
        | LC_ALL=C sort > {{.RUN}}/upstream.txt
      # A truncated listing must never become a deletion list. CTAN is ~496k files.
      - test "$(wc -l < {{.RUN}}/upstream.txt)" -gt 400000

  state:
    desc: Fetch the state file to RUN/applied.txt; a missing one is rebuilt from the bucket with RECONCILE=true, or empty with SEED=true
    cmds:
      - mkdir -p {{.RUN}}
      # aws s3 ls exits 1 when nothing matches and 0 when the key exists; any other code is
      # a real error and must not be mistaken for a missing state.
      - |
        rc=0; aws s3 ls {{.S3}}/{{.STATE}} > /dev/null || rc=$?
        case $rc in
          0) aws s3 cp {{.AWS_FLAGS}} {{.S3}}/{{.STATE}} {{.RUN}}/applied.txt.xz
             xz -dc {{.RUN}}/applied.txt.xz > {{.RUN}}/applied.txt ;;
          1) case "{{.SEED}} {{.RECONCILE}}" in
               "true "*) : > {{.RUN}}/applied.txt ;;
               *" true") touch {{.RUN}}/rebuild-now ;;
               *) echo "no state at {{.STATE}}: pass RECONCILE=true to rebuild it from the bucket, or SEED=true if the bucket is empty" >&2; exit 1 ;;
             esac ;;
          *) exit $rc ;;
        esac

  rebuild:
    desc: List the bucket and make the state exactly what is there (a lost state file, or the daily reconcile)
    status: ['test ! -f {{.RUN}}/rebuild-now']
    set: [pipefail]
    cmds:
      # aws s3 ls: "date time size key"; the key may hold spaces. R2 does not list in byte
      # order (install-tl after install-tl.zip.sha512.asc), so sort before join and comm.
      - >-
        aws s3 ls --recursive {{.S3}}/
        | awk '{ if (match($0, /^[^ ]+ +[^ ]+ +[0-9]+ /)) print substr($0, RLENGTH + 1) "\t" $3 }'
        | LC_ALL=C sort > {{.RUN}}/bucket.txt
      # A key is applied when the bucket has it at upstream's size; its mtime is upstream's.
      - >-
        LC_ALL=C join -t "$(printf '\t')" -o 1.1,1.2,1.3,2.2 {{.RUN}}/upstream.txt {{.RUN}}/bucket.txt
        | awk -F'\t' -v OFS='\t' '$2 == $4 { print $1, $2, $3 }' > {{.RUN}}/applied.new
      - xz -T0 -c {{.RUN}}/applied.new > {{.RUN}}/applied.new.xz
      - aws s3 cp {{.AWS_FLAGS}} {{.RUN}}/applied.new.xz {{.S3}}/{{.STATE}}
      - mv {{.RUN}}/applied.new {{.RUN}}/applied.txt
      - rm -f {{.RUN}}/rebuild-now

  diff:
    desc: RUN/changed.txt (lines upstream has and the state lacks), deleted.txt (paths the state has and upstream lacks), paths.txt
    cmds:
      - LC_ALL=C comm -13 {{.RUN}}/applied.txt {{.RUN}}/upstream.txt > {{.RUN}}/changed.txt
      - cut -f1 {{.RUN}}/upstream.txt > {{.RUN}}/paths.txt
      - cut -f1 {{.RUN}}/applied.txt | LC_ALL=C comm -23 - {{.RUN}}/paths.txt > {{.RUN}}/deleted.txt

  plan:
    desc: Refuse a tree over CEILING_GB or a file the disk cannot hold; split changed.txt into RUN/batch-NNNN.txt
    cmds:
      - >-
        awk -F'\t' -v gb={{.CEILING_GB}} '{ s += $2 }
          END { printf "upstream: %d objects, %.2f GB, ceiling %d GB\n", NR, s / 1e9, gb; exit (s > gb * 1e9) }'
        {{.RUN}}/upstream.txt
      - mkdir -p {{.STAGING}} && find {{.STAGING}} -mindepth 1 -delete
      - df -Pk {{.STAGING}}
      # The largest single file must fit with 1 GiB to spare for RUN's text and the OS.
      - >-
        awk -F'\t' -v kb="$(df -Pk {{.STAGING}} | awk 'NR == 2 { print $4 }')"
          '$2 / 1024 > kb - 1048576 { print "no room for " $1 " (" $2 " bytes)"; bad = 1 } END { exit bad }'
        {{.RUN}}/changed.txt
      - rm -f {{.RUN}}/batch-*.txt
      # Cumulative size from the listing; nothing is touched to plan. A file over the cap is
      # a batch by itself and leaves the running batch alone. The decision batch, tlnet's
      # tlpkg/ (the tlpdb) and the root files (timestamp, FILES.*, CTAN.sites, README*,
      # index.html, tds.zip), is last, so nothing a client reads to decide what to fetch can
      # name a key that has not landed.
      - >-
        awk -F'\t' -v OFS='\t' -v CAP=$(({{.BATCH_GB}} * 1000000000)) -v DIR={{.RUN}} -v TL="{{.TL}}" '
          index($1, TL "/tlpkg/") == 1 || $1 !~ /\// { dec[++nd] = $0; next }
          $2 > CAP { f = name(++n); print > f; close(f); next }
          { if (sz + $2 > CAP) flush(); buf[++nb] = $0; sz += $2 }
          END { flush(); if (nd) { f = name(++n); for (i = 1; i <= nd; i++) print dec[i] > f; close(f) }
                printf "%d lines in %d batches\n", NR, n }
          function name(i) { return sprintf("%s/batch-%04d.txt", DIR, i) }
          function flush(  i, f) { if (!nb) return; f = name(++n)
            for (i = 1; i <= nb; i++) print buf[i] > f; close(f); nb = sz = 0; split("", buf) }'
        {{.RUN}}/changed.txt

  tlpdb:
    desc: When the delta touches tlnet, fetch its control files into RUN/tl and check the signed sha512 against the pinned key
    status:
      - 'test "$(grep -c "^{{.TL}}/" {{.RUN}}/changed.txt 2>/dev/null)" = 0'
    set: [pipefail]
    cmds:
      - mkdir -p {{.RUN}}/tl
      - printf '%s\n' tlpkg/texlive.tlpdb.xz tlpkg/texlive.tlpdb.sha512 tlpkg/texlive.tlpdb.sha512.asc tlpkg/gpg/pubring.gpg > {{.RUN}}/tl/files.txt
      - task: retry
        vars: {CMD: '{{.RSYNC}} -Lt --files-from={{.RUN}}/tl/files.txt {{.SOURCE}}{{.TL}}/ {{.RUN}}/tl/'}
      - xz -dkf {{.RUN}}/tl/tlpkg/texlive.tlpdb.xz
      # The keyring is read as a plain file from the same mirror as the signature, so the
      # pinned fingerprint is the only real check; GOODSIG too, because gpgv reports an
      # expired or revoked key as VALIDSIG with exit 0.
      - >-
        cd {{.RUN}}/tl/tlpkg && shasum -a 512 -c texlive.tlpdb.sha512 &&
        gpgv --status-fd 1 --keyring gpg/pubring.gpg texlive.tlpdb.sha512.asc texlive.tlpdb.sha512
        | awk '/^\[GNUPG:\] GOODSIG /{g=1} /^\[GNUPG:\] VALIDSIG .* {{.TL_KEY}}$/{v=1} END{exit !(g&&v)}'

  batches:
    desc: Work the first MAX_BATCHES of RUN/batch-NNNN.txt in name order; each commits before the next starts
    vars:
      BATCHES:
        sh: ls {{.RUN}}/batch-*.txt 2>/dev/null | head -n {{.MAX_BATCHES}}
    cmds:
      - for: {var: BATCHES, split: "\n", as: B}
        task: batch
        vars: {B: '{{.B}}'}

  batch:
    internal: true
    requires: {vars: [B]}
    status: ['test -z "{{.B}}"']   # an empty delta loops once with an empty item
    cmds:
      - {task: fetch, vars: {B: '{{.B}}'}}
      - {task: verify, vars: {B: '{{.B}}'}}
      - {task: publish, vars: {B: '{{.B}}'}}
      - {task: purge, vars: {B: '{{.B}}'}}
      - {task: checkpoint, vars: {B: '{{.B}}'}}

  fetch:
    internal: true
    requires: {vars: [B]}
    preconditions:
      - sh: test "$(df -Pk {{.STAGING}} | awk 'NR == 2 { print $4 }')" -gt "$(awk -F'\t' '{ s += $2 } END { print int(s / 1024) + 1048576 }' {{.B}})"
        msg: "not enough free disk for {{.B}}"
    cmds:
      - find {{.STAGING}} -mindepth 1 -delete
      - cut -f1 {{.B}} > {{.RUN}}/files.txt
      # --files-from implies --relative (parents are created) and not -r; -L turns every
      # symlink into its file. A path that vanished since the listing is skipped: its
      # state line then names a key the bucket lacks, which the next hour deletes. No
      # --partial: staging is emptied every batch, so there is nothing to resume, and a
      # partial file must never reach the bucket.
      - task: retry
        vars: {CMD: '{{.RSYNC}} -Lt --files-from={{.RUN}}/files.txt --ignore-missing-args {{.SOURCE}} {{.STAGING}}/'}

  verify:
    requires: {vars: [B]}   # public so it can be run by hand against a canned batch (section 8)
    dir: '{{.STAGING}}'
    cmds:
      # Belt to list's braces: no versioned tlnet container may reach the bucket.
      - test "$(cut -f1 {{.B}} | grep -c '^{{.TL}}/archive/.*\.r[0-9]*\.tar\.xz$')" = 0
      # Every signed sha512 in this batch (root installers and updaters, the tlpdb) checks
      # against the keyring already pinned by tlpdb. The while loop's status is the pipeline's.
      - >-
        cut -f1 {{.B}} | grep -E '^{{.TL}}/(tlpkg/texlive\.tlpdb|[^/]+)\.sha512$' | while IFS= read -r f; do
        (cd "$(dirname "$f")" && f=$(basename "$f") && shasum -a 512 -c "$f" &&
        gpgv --status-fd 1 --keyring {{.RUN}}/tl/tlpkg/gpg/pubring.gpg "$f.asc" "$f"
        | awk '/^\[GNUPG:\] GOODSIG /{g=1} /^\[GNUPG:\] VALIDSIG .* {{.TL_KEY}}$/{v=1} END{exit !(g&&v)}')
        || exit 1; done
      # Every tlnet container in this batch must match the checksum the verified tlpdb gives it.
      - >-
        if test "$(grep -c '^{{.TL}}/archive/' {{.B}})" != 0; then
        awk -F'\t' -v TL="{{.TL}}" 'NR == FNR { want[$1] = 1; next }
          /^name /{split($0, a, " "); n = a[2]}
          /^containerchecksum /{split($0, a, " "); p = TL "/archive/" n ".tar.xz"; if (p in want) print a[2] "  " p}
          /^doccontainerchecksum /{split($0, a, " "); p = TL "/archive/" n ".doc.tar.xz"; if (p in want) print a[2] "  " p}
          /^srccontainerchecksum /{split($0, a, " "); p = TL "/archive/" n ".source.tar.xz"; if (p in want) print a[2] "  " p}'
        {{.B}} {{.RUN}}/tl/tlpkg/texlive.tlpdb | shasum -a 512 -c --quiet; fi
      # The decision batch. The tlpdb it carries must be the one verified at the start of
      # the run, the .xz tlmgr downloads must decompress to it byte for byte, and every
      # container it names must be in the bucket after this run: the state, plus this
      # batch, minus this run's deletions (verification-and-security.md).
      - >-
        test ! -f {{.TL}}/tlpkg/texlive.tlpdb || {
        cmp {{.TL}}/tlpkg/texlive.tlpdb {{.RUN}}/tl/tlpkg/texlive.tlpdb &&
        xz -dc {{.TL}}/tlpkg/texlive.tlpdb.xz | cmp - {{.TL}}/tlpkg/texlive.tlpdb &&
        { cut -f1 {{.RUN}}/applied.txt; cut -f1 {{.B}}; } | LC_ALL=C sort -u
        | LC_ALL=C comm -23 - {{.RUN}}/deleted.txt > {{.RUN}}/tl/after.txt &&
        awk -v TL="{{.TL}}" '/^name /{n=$2}
          /^containerchecksum /{print TL "/archive/" n ".tar.xz"}
          /^doccontainerchecksum /{print TL "/archive/" n ".doc.tar.xz"}
          /^srccontainerchecksum /{print TL "/archive/" n ".source.tar.xz"}' {{.TL}}/tlpkg/texlive.tlpdb
        | LC_ALL=C sort | LC_ALL=C comm -23 - {{.RUN}}/tl/after.txt > {{.RUN}}/tl/missing.txt &&
        test ! -s {{.RUN}}/tl/missing.txt; }

  publish:
    internal: true
    requires: {vars: [B]}
    set: [pipefail]   # tee must never mask a failed upload
    cmds:
      # cp --recursive never lists the destination: one PutObject per file, multipart only
      # above aws.config's threshold. Keys are the paths under staging, which are CTAN's own.
      # timestamp is what mirmon reads, so it goes up alone, after everything else.
      - aws s3 cp {{.AWS_FLAGS}} --recursive --exclude timestamp {{.STAGING}}/ {{.S3}}/ | tee -a {{.RUN}}/publish.txt
      - test ! -f {{.STAGING}}/timestamp || aws s3 cp {{.AWS_FLAGS}} {{.STAGING}}/timestamp {{.S3}}/timestamp | tee -a {{.RUN}}/publish.txt

  purge:
    internal: true
    requires: {vars: [B]}
    status: ['test "{{.CACHE}}" != on']
    cmds:
      # Only keys the edge may hold: paths in B that the state already knows (changed, or
      # deleted), never new ones. Percent-encode everything outside the unreserved set and
      # "/" (23 paths carry spaces, 69 carry ( ) [ ] { } & # > ~ $).
      - >-
        cut -f1 {{.B}} | LC_ALL=C comm -12 - <(cut -f1 {{.RUN}}/applied.txt)
        | awk 'BEGIN { for (i = 0; i < 256; i++) ord[sprintf("%c", i)] = i }
          { out = ""; for (j = 1; j <= length($0); j++) { c = substr($0, j, 1)
              out = out ((c ~ /[A-Za-z0-9\/._~-]/) ? c : sprintf("%%%02X", ord[c])) }
            print "{{.URL}}/" out }' > {{.RUN}}/urls.txt
      # 100 URLs per call; the sleep keeps it under 800 URLs/s. A failed call fails the run.
      - >-
        rm -f {{.RUN}}/purge-*; test ! -s {{.RUN}}/urls.txt || { split -l 100 -a 5 {{.RUN}}/urls.txt {{.RUN}}/purge-;
        for f in {{.RUN}}/purge-*; do
        awk 'BEGIN { printf "{\"files\":[" } { printf "%s\"%s\"", (NR > 1 ? "," : ""), $0 } END { print "]}" }' "$f"
        | {{.CURL}} -X POST -H "Authorization: Bearer $CF_API_TOKEN" -H "Content-Type: application/json" --data-binary @-
          "{{.CF}}/purge_cache" -o /dev/null;
        wc -l < "$f" >> {{.RUN}}/purge.txt; sleep 0.3; done; }

  checkpoint:
    internal: true
    requires: {vars: [B]}
    set: [pipefail]
    cmds:
      # applied := applied minus these paths, plus these lines. 0.1 s on 496k lines.
      - >-
        awk -F'\t' 'NR == FNR { drop[$1] = 1; next } !($1 in drop)' {{.B}} {{.RUN}}/applied.txt
        | cat - {{.B}} | LC_ALL=C sort > {{.RUN}}/applied.new
      - xz -T0 -c {{.RUN}}/applied.new > {{.RUN}}/applied.new.xz
      # One PutObject; the old file or the new one is in the bucket, never half.
      - aws s3 cp {{.AWS_FLAGS}} {{.RUN}}/applied.new.xz {{.S3}}/{{.STATE}}
      - mv {{.RUN}}/applied.new {{.RUN}}/applied.txt
      - find {{.STAGING}} -mindepth 1 -delete

  delete:
    desc: Remove the keys in RUN/deleted.txt (1,000 per DeleteObjects call, free), purge them, drop them from the state
    status: ['test ! -s {{.RUN}}/deleted.txt']
    set: [pipefail]
    cmds:
      - rm -f {{.RUN}}/del-*; split -l 1000 -a 4 {{.RUN}}/deleted.txt {{.RUN}}/del-
      # Quiet mode returns only errors, and the CLI exits 0 even then, so grep for them.
      - >-
        for f in {{.RUN}}/del-*; do
        awk 'BEGIN { printf "{\"Objects\":[" } { gsub(/\\/, "\\\\"); gsub(/"/, "\\\"")
             printf "%s{\"Key\":\"%s\"}", (NR > 1 ? "," : ""), $0 } END { print "],\"Quiet\":true}" }' "$f"
        > "$f.json" && aws s3api delete-objects --bucket {{.BUCKET}} --delete "file://$f.json" | tee -a {{.RUN}}/publish.txt
        | { test "$(grep -c Errors)" = 0; }; done
      - {task: purge, vars: {B: '{{.RUN}}/deleted.txt'}}
      - {task: checkpoint, vars: {B: '{{.RUN}}/deleted.txt'}}

  reconcile:
    desc: Rebuild the state from a bucket listing, then delete keys that neither upstream nor a reserved prefix (.state/, .site/) owns
    status:
      - 'test "{{.RECONCILE}}" = false || { test "{{.RECONCILE}}" = auto && test "$(cut -d" " -f2 {{.RUN}}/late.txt)" != 3; }'
    cmds:
      - touch {{.RUN}}/rebuild-now
      - {task: rebuild}
      - cut -f1 {{.RUN}}/bucket.txt | grep -vE '^\.(state|site)/' | LC_ALL=C comm -23 - {{.RUN}}/paths.txt > {{.RUN}}/deleted.txt
      - {task: delete}

  smoke:
    desc: Read /timestamp and a sample of this run's keys back through the domain; sizes must match the listing, the tlpdb sha512 must match the verified copy
    cmds:
      - >-
        { echo timestamp; grep -h '^upload:' {{.RUN}}/publish.txt 2>/dev/null | sed 's|^upload: .* to s3://{{.BUCKET}}/||' | head -3; }
        | while IFS= read -r k; do
        want=$(awk -F'\t' -v k="$k" '$1 == k { print $2 }' {{.RUN}}/upstream.txt);
        got=$({{.CURL}} -I "{{.URL}}/$k" | awk 'tolower($1) == "content-length:" { sub(/\r/, "", $2); print $2 }');
        echo "$k: $got bytes (listing: $want)"; test "$got" = "$want" || exit 1; done
      - >-
        test ! -f {{.RUN}}/tl/tlpkg/texlive.tlpdb.sha512 ||
        {{.CURL}} {{.URL}}/{{.TL}}/tlpkg/texlive.tlpdb.sha512 | cmp - {{.RUN}}/tl/tlpkg/texlive.tlpdb.sha512

  report:
    desc: Append the run summary to the Actions job page (stdout elsewhere) from the counts in RUN
    silent: true
    cmds:
      - |
        n() { test -f "$1" && wc -l < "$1" || echo 0; }
        cat >> "${GITHUB_STEP_SUMMARY:-/dev/stdout}" <<EOF
        ## Sync succeeded, $(date -u '+%Y-%m-%d %H:%M UTC')

        $(paste {{.RUN}}/start.txt {{.RUN}}/late.txt | awk -v m={{.CRON_MINUTE}} -v ev="${GITHUB_EVENT_NAME:-local}" '{ printf "Started %02d:%02d UTC", int($1 / 3600) % 24, int($1 / 60) % 60 }
          ev == "schedule" { printf ", %d min after the :%02d slot", $2 / 60, m } END { print "." }')

        | Mirror | $(n {{.RUN}}/upstream.txt) objects, $(awk -F'\t' '{ s += $2 } END { printf "%.2f GB", s / 1e9 }' {{.RUN}}/upstream.txt) upstream, at [{{.URL}}/]({{.URL}}/) |
        |---|---|
        | Delta | $(n {{.RUN}}/changed.txt) new or changed in $(ls {{.RUN}}/batch-*.txt 2>/dev/null | wc -l | awk -v m={{.MAX_BATCHES}} '{ print $1 " batches" ($1 > m ? ", " $1 - m " left for the next hour" : "") }'); $(n {{.RUN}}/deleted.txt) deleted |
        | Published to R2 | $(grep -c '^upload:' {{.RUN}}/publish.txt 2>/dev/null || true) uploaded; $(awk '{ s += $1 } END { print s + 0 }' {{.RUN}}/purge.txt 2>/dev/null) URLs purged (cache {{.CACHE}}) |
        | State | $(n {{.RUN}}/applied.txt) lines in {{.STATE}} |
        | Storage | $(test -f {{.RUN}}/bucket.txt && awk -F'\t' '{ s += $2 } END { printf "%.2f GB in %d objects (reconciled)", s / 1e9, NR }' {{.RUN}}/bucket.txt || echo "not measured this run") of the {{.CEILING_GB}} GB ceiling |
        | Signature | $(test -f {{.RUN}}/tl/tlpkg/texlive.tlpdb && echo "tlnet tlpdb signed by {{.TL_KEY}}; $(n {{.RUN}}/tl/missing.txt) named containers missing from the bucket" || echo "no tlnet keys in this delta") |
        EOF

  ping:
    desc: Dead man's switch; healthchecks.io emails when two hours pass without this (skipped if HEALTHCHECK_URL is unset)
    cmds:
      - test -z "$HEALTHCHECK_URL" || curl -fsS -m 10 --retry 3 -o /dev/null "$HEALTHCHECK_URL"

  page:
    desc: Upload site/index.html with the sync date filled in to .site/index.html; the Transform Rule serves it at /
    dir: '{{.ROOT_DIR}}'
    set: [pipefail]   # a failed sed must never upload an empty page
    cmds:
      - sed "s|<!--UPDATED-->|$(date -u '+%Y-%m-%d %H:%M UTC')|" site/index.html | aws s3 cp {{.AWS_FLAGS}} --content-type text/html --cache-control no-cache - {{.S3}}/.site/index.html

  retry:
    desc: Run CMD, retrying rsync's transport exit codes (5 10 12 30 35) five times with backoff and jitter
    cmds:
      - |
        for i in 1 2 3 4 5; do
          rc=0; ( {{.CMD}} ) || rc=$?   # subshell so CMD's exit cannot end the loop; || so Task's errexit cannot
          case $rc in 0) exit 0;; 5|10|12|30|35) ;; *) exit $rc;; esac
          s=$((15 * 2 ** i + RANDOM % 30)); echo "rsync exit $rc, retry $i in ${s}s" >&2; sleep $s
        done; exit $rc
```

`aws.config`:

```ini
# Read via AWS_CONFIG_FILE. Anything under 4 GiB is one PutObject; the five CTAN files over
# R2's 4.995 GiB single-part limit go multipart in 512 MiB parts (13 parts, 15 Class A each).
[default]
retry_mode = standard
max_attempts = 10
s3 =
    multipart_threshold = 4GB
    multipart_chunksize = 512MB
    max_concurrent_requests = 32
```

The rule files in `cloudflare/`, each a whole ruleset for one phase, each carrying its own
sha256 in the ruleset `description` so `rules` can tell whether the zone already has it:
`cache-rules.json` (`http_request_cache_settings`, used with `CACHE=on`), `bypass-rules.json`
(same phase, the one bypass rule, used with `CACHE=off`), `transform-rules.json`
(`http_request_transform`: `/` to `/.site/index.html`) and `redirect-rules.json`
(`http_request_dynamic_redirect`: a directory URL to `https://ctan.org/tex-archive<path>`,
per `official-mirror-and-url.md`). Smart Tiered Cache is a one-time zone `PATCH`, done by
hand or by a `task tiered` that is never in `sync`.

### What each ordering rule protects

- `rules` first: a wrong token fails in one second with nothing fetched.
- `list` before `state`: dante is the slow, external, rate-limited side; if it refuses, no
  R2 call was made. `rebuild` also needs `upstream.txt`.
- `plan` before `tlpdb`: the ceiling and disk checks cost nothing and reject a bad run
  before the second rsync connection.
- Within a batch, `checkpoint` last: the state file only ever describes uploads and purges
  that returned success. A run killed anywhere before it leaves the state as of the previous
  batch and the next hour repeats this one (`PutObject` of the same bytes is idempotent; a
  purge of an unchanged URL is free).
- The decision batch last, and `timestamp` last within it: containers land before the
  tlpdb that names them, every file lands before the `timestamp` that says the mirror is
  current, and with `MAX_BATCHES` the decision batch simply waits for the hour in which
  everything before it has landed.
- `delete` after `batches`: a container the new tlpdb no longer names is removed only after
  the new tlpdb is live, so no client holds a tlpdb naming a missing key.
- `reconcile` after `delete`: the listing is of a settled bucket, so it never sees a key
  mid-deletion.
- `smoke`, `report`, `ping`, `page` in that order for the reasons `CLAUDE.md` already gives.

## 2. The data files

All in `RUN` (`/tmp/ctan-run`), all plain text, all one record per line. The raw listing is
538,289 lines (511,027 regular files, 27,262 directories) and takes 6.9 s from dante
(`SCRATCH/rsync-time.txt`, 2026-08-26).

| File | Written by | Line format | Sorted |
|---|---|---|---|
| `start.txt`, `late.txt` | `clock` | the run's start as epoch seconds; `lateness_seconds slot_hour` | |
| `listing.txt` | `list` | rsync `--list-only` raw: `perms size yyyy/mm/dd hh:mm:ss path` | rsync's order |
| `upstream.txt` | `list` | `path TAB size TAB yyyy/mm/dd hh:mm:ss` | `LC_ALL=C sort` |
| `applied.txt` | `state`, `rebuild`, `checkpoint` | same as `upstream.txt`; the bucket's contents | same |
| `rebuild-now` | `state`, `reconcile` | empty marker: `rebuild` runs while it exists | |
| `paths.txt` | `diff` | `path` | same (a projection of a sorted file) |
| `changed.txt` | `diff` | as `upstream.txt`: lines upstream has that the state lacks (new, or changed size or mtime) | same |
| `deleted.txt` | `diff`, `reconcile` | `path`: in the state, not upstream | same |
| `batch-NNNN.txt` | `plan` | as `upstream.txt`; a partition of `changed.txt` | same within each file |
| `files.txt` | `fetch` | `path` for rsync `--files-from` | |
| `del-NNNN`, `del-NNNN.json` | `delete` | 1,000 paths; the `DeleteObjects` body | |
| `urls.txt`, `purge-NNNNN` | `purge` | one URL per line, 100 per chunk | |
| `publish.txt` | `publish`, `delete` | `aws s3 cp` lines `upload: staging/x to s3://tlnet/x`; `delete-objects` JSON | append order |
| `purge.txt` | `purge` | one number per call: URLs in it | |
| `bucket.txt` | `rebuild` | `key TAB size` | `LC_ALL=C sort` |
| `tl/tlpkg/*` | `tlpdb` | the four control files plus the decompressed tlpdb | |
| `tl/after.txt`, `tl/missing.txt` | `verify` (decision batch) | `path`: the bucket after this run; containers the tlpdb names that it lacks | |

The state object `.state/applied.txt.xz` is `applied.txt` compressed: 37.7 MB of text, 3.1 MB
of `.xz`, 4.0 s to compress on two threads locally (computed: `xz -T0 -k upstream.txt`). It is
publicly readable at `https://ctan.ijosh.com/.state/applied.txt.xz`, which is fine: it is a
listing, and a useful one. `reconcile` skips `.state/` and `.site/` explicitly; `list` can
never produce either because CTAN has no dot directories at its root.

### Why path first

A size-first line (`size mtime path`) makes `comm` on whole lines wrong,
because `comm` compares whole lines and its merge assumes both inputs are sorted by that
comparison. Demonstrated:

```sh
printf '9\tt\ta\n1\tt\tb\n' > A     # sorted by path
printf '1\tt\tb\n' > B
LC_ALL=C comm -13 A B               # should print nothing; prints "1 t b"
```

With the path first, the whole-line byte order is the path order: paths are unique, and the
TAB (0x09) that ends a path sorts below every character that can continue one (all are
0x20 or above), so `a/b` sorts before `a/b.c` in both views. Checked on the real listing:
`cut -f1 upstream.txt | LC_ALL=C sort -c` reports sorted. That is what lets one sort serve
the whole-line `comm -13` (changes) and the path-only `comm -23` (deletions), and lets
`join` on field 1 work in `rebuild`.

### Characters in paths

Counted on the 496,149 stored paths (computed; the commands are in section 8):

| Property | Count | Consequence |
|---|---|---|
| Contains a space | 23 | TAB is the separator, never whitespace; `IFS= read -r` in loops; rsync `--files-from` takes one path per line unquoted |
| Two consecutive spaces, leading or trailing space | 0 | awk field rebuilding is never used on a path anyway |
| Contains `( ) [ ] { } & # > ~ $` | 69 | never interpolated into a shell line; only ever read from files by rsync, awk, `aws --recursive` and JSON bodies |
| Contains `"` `\` or `'` | 0 | the `DeleteObjects` awk escapes both anyway |
| Contains TAB, newline or another control character | 0 | rsync would print them as `\#ooo`; a TAB would split a line: `list` should fail if `awk -F'\t' 'NF != 3'` ever matches (add if it does) |
| Non-ASCII bytes | 0 | `LC_ALL=C` makes this moot |
| Longest path | 151 bytes | R2 allows 1,024 |
| Zero-byte files | 525 | openrsync prints a blank size column; rsync 3.2.7 prints `0`; the normaliser handles both |
| Collide under case folding | 23 pairs | a hazard only for staging on a case-insensitive filesystem (macOS default); the runner is ext4 |
| Root files (no `/`) | 10: `CTAN.sites`, `FILES.byname`, `FILES.byname.gz`, `FILES.last07days`, `README.mirrors`, `README.structure`, `README.uploads`, `index.html`, `tds.zip`, `timestamp` | the decision batch, with tlnet's `tlpkg/` |

`LC_ALL=C` on every `sort`, `comm` and `join`: byte order is the only order the three agree
on across locales, and R2 lists keys in its own order (not byte order), so the bucket listing
is always re-sorted before it meets anything else.

### How deletions are encoded

They are not encoded. The state file lists what the bucket holds; a deletion is a path in
the state and not upstream, computed each hour by `comm -23` on the path columns. After
`delete` succeeds, `checkpoint` drops those lines. There is no tombstone, no "deleted at"
column, nothing to garbage-collect. A vanished-mid-run path (rsync `--ignore-missing-args`)
leaves a state line naming a key the bucket never received; the next hour lists it as deleted,
`DeleteObjects` on an absent key succeeds, and the line goes. The daily `reconcile` removes it
the same day at the latest. CTAN touches ~23 files an hour (16,568 in 30 days, computed), so
failing the run on a vanished path would end most multi-hour pushes early for nothing.

## 3. The batching algorithm

Input: `changed.txt`, path-sorted, `path TAB size TAB mtime`. Output: `batch-NNNN.txt` files
that partition it. Three rules: a batch holds at most `CAP` bytes by listing size; a line
larger than `CAP` is a batch by itself and leaves the running batch untouched (flushing the
running batch before each lone file would leave five half-empty batches behind the five
installers); every
`systems/texlive/tlnet/tlpkg/` line and every root file goes into one final decision batch.
There is no separate 4.995 GiB rule: with `CAP` at 4 GB decimal, every file over R2's
single-part limit is over `CAP`. Only the first `MAX_BATCHES` files run in a given hour; the
decision batch is always the last file, so it runs only in the hour when nothing precedes
it. The awk is the one in `plan` above. Traces (computed with BWK awk 20200816 on macOS; the
program is POSIX awk, no gawk extensions, because Ubuntu's `awk` is mawk):

| Input | Result |
|---|---|
| The seed after `RECONCILE=true` bootstrapped the state from the live tlnet keys: 479,175 lines, 126.21 GB | **30 batches** in 1.5 s: 24 packed batches (the smallest 3.9 GB), 5 lone batches (`MacTeX.pkg` and `mactex-20260324.pkg`, 6,865,013,189 bytes; the three `texlive*.iso`, 6,784,798,720), batch 30 = the 10 root files, 0.030 GB. Sum of batch lines = 479,175 |
| The seed of an empty bucket (`SEED=true`): all 496,149 lines | 31 batches; batch 31 = `tlpkg/` plus the root files, 2,093 lines, 0.125 GB |
| Empty `changed.txt` | prints `0 lines in 0 batches`, exit 0, no files; `batches` loops once over an empty item and `batch` is skipped by `status` |
| One 7,000,000,000-byte file | one batch of one line |
| Only `timestamp` (a quiet hour would also carry ~20 files) | one batch, the decision batch |

Worst-case peak disk is `max(CAP, largest file)` plus `RUN`'s text: 6.87 GB for a lone
installer, 4.00 GB otherwise, never more whatever upstream does in an hour, because the
plan reads sizes from the listing and nothing is fetched to make it. Text in `RUN` at the
seed: `listing.txt` 50.7 MB, `upstream.txt` 37.7 MB, `changed.txt` 37.7 MB, batches 37.7 MB,
`applied.txt` and `applied.new` up to 37.7 MB each, `.xz` copies 3 MB: ~205 MB.

With `MAX_BATCHES=4` the seed is eight scheduled runs: 4 batches of ~4 GB at 22.7 MB/s from
dante and ~100 `PutObject`/s is 15 to 25 minutes of work, and the decision batch lands in
the eighth. Each scheduled run starts 15 to 45 minutes after its slot, so the wall clock is
about eight hours plus that drift; `seeding-and-migration.md` has the arithmetic and the case
for `workflow_dispatch`, which starts at once. `max_batches` raises the per-run count.

## 4. Expressing the loop in go-task

Every feature below was exercised on Task 3.53.1 (`SCRATCH/tasktest/Taskfile.yml`, 2026-08-26)
and dated against the changelog (fetched). Minimum Task version: **3.28.0** (2023-07-24), for
`for` over a variable with `split`; `requires` is 3.27.0, `set`/`shopt` 3.20.0, `internal`
3.16.0, `run: once` 3.7.0, `preconditions` 2.6.0, `status` 1.3.1. `setup-task@v2` installs
the latest, 3.53.1 today, so the floor is documentation, not a constraint.

| Feature | Verified behaviour | Used for |
|---|---|---|
| `for: {var: X, split: "\n", as: B}` + `task:` + `vars:` | one call per line, vars forwarded, each call runs (default `run: always`); `--dry` renders every iteration | `batches` |
| `for: {var: X, as: R}` with `cmd:` | splits on whitespace; the item is a plain string the shell can slice with `${R%%=*}` | `rules` |
| Empty `var` | the loop body runs **once with an empty item**; it does not run zero times | hence `status: ['test -z "{{.B}}"']` on `batch`, which was seen to skip it |
| `requires: {vars: [B]}` | fails only when the variable is **absent**; an empty string passes | catches a `task batch` typed by hand without `B=`; not a substitute for the `status` guard |
| `status:` | exit 0 means up to date, task skipped, exit 0; evaluated in `--dry` too; `--force` overrides it, also under `--dry` | `batch`, `rebuild`, `tlpdb`, `purge`, `delete`, `reconcile` |
| `vars: X: {sh: ...}` on a task | evaluated **when the task is called**, so a task called after `plan` sees `plan`'s files; also evaluated under `--dry` | `batches` (`ls RUN/batch-*.txt \| head`). Hence: no `sh:` var may touch the network |
| errexit | every `cmds` entry runs with errexit whether or not `set: [errexit]` is given; `cmd; rc=$?` exits before the assignment; `rc=0; (cmd) \|\| rc=$?` captures | `state`, `retry` |
| `set: [pipefail]` | a failing left side fails the entry; a failing pipeline inside an `if` body fails it too | `list`, `rebuild`, `tlpdb`, `publish`, `checkpoint`, `delete`, `page`; **not** `verify` or `purge`, whose `grep`/`comm` pipelines must survive an empty match |
| `preconditions: [{sh, msg}]` | non-zero fails the task with `msg`; a failing precondition in a called task fails the caller | `fetch`'s disk check |
| `defer:` | runs after the task's commands, also after a failure | not used: staging is emptied at the start of `fetch` and the end of `checkpoint`, and a failed batch must leave staging for inspection |
| `deps:` | parallel, order not guaranteed | not used: `list` and `state` could overlap for 1 s of gain and interleaved logs |
| `run: once` | a task named twice in one run runs once | not used: `purge`, `checkpoint`, `rebuild` and `delete` are meant to run again when called again |
| `silent: true` | command not echoed | `report` |
| `internal: true` | hidden from `--list` and **not callable from the command line** (`task purge` answers `Task "purge" is internal`, verified); callable only from other tasks | `batch`, `fetch`, `publish`, `purge`, `checkpoint`; `verify` and `retry` stay public because section 8 runs them by hand |
| `{{.X \| default "v"}}`, `{{if eq .X "v"}}` | `SEED`, `RECONCILE`, `CACHE`, `MAX_BATCHES`; the ruleset list | |
| `fromJson` | returns `null` on invalid JSON, no error | see `check.yml` in section 7 |

Two shell loops remain inside single `cmds` entries: `verify`'s `while read` and the `for f
in RUN/purge-*` and `del-*` loops. Today's `verify` has the same shape. They stay shell rather
than Task `for` because their chunk files are made by the same task two lines earlier, and a
task's `sh:` var is evaluated before its first command. A marker file (`rebuild-now`) rather
than a variable is what lets `state` and `reconcile` share `rebuild` without a `vars:`
handoff across `cmds`.

## 5. Multipart for the objects over 4.995 GiB

Five objects (computed): `systems/mac/mactex/MacTeX.pkg` and `mactex-20260324.pkg`
(6,865,013,189 bytes), `systems/texlive/Images/texlive.iso`, `texlive2026.iso` and
`texlive2026-20260301.iso` (6,784,798,720). R2 limits (fetched, R2 limits and multipart
pages): single upload 4.995 GiB; parts 5 MiB to 5 GiB, at most 10,000, **all parts except the
last the same size**; incomplete uploads are aborted by a default lifecycle rule after 7
days. The AWS CLI uploads uniform parts of `multipart_chunksize` with a smaller last part,
which is exactly R2's rule.

Configuration is one `aws.config`, not a per-call override:

| Setting | Value | Why |
|---|---|---|
| `multipart_threshold` | `4GB` (4 GiB; the CLI's suffixes are binary, fetched) | below R2's 4.995 GiB limit, above every other CTAN file: 2 files sit between 512 MB and 4 GiB (`protext.zip` and its alias, 1,138,914,783 bytes) and go single-part. Today's `200MB` would multipart 9 CTAN files needlessly |
| `multipart_chunksize` | `512MB` | 6,865,013,189 / 536,870,912 = 12.8, so 13 parts; Class A per installer 1 create + 13 parts + 1 complete = 15; 75 for all five, once a year. The CLI default (8 MB) would be ~820 parts each |
| `max_concurrent_requests` | 32 | parts of one file upload in parallel; 13 parts, so bandwidth, not concurrency, bounds it |
| `--cli-read-timeout 300 --cli-connect-timeout 60` | in `AWS_FLAGS`, since the config-vars page does not list them as file settings (fetched) | a 512 MiB part at a stalled socket fails in 5 minutes instead of at the job limit |
| `retry_mode = standard`, `max_attempts = 10` | file settings (fetched): 10 attempts, exponential backoff capped at 20 s, retries throttling and 5xx | the adopted mode; `adaptive` is documented as experimental |

Resumability: the AWS CLI has none. A `cp` that fails after retries aborts its own multipart
upload; a `cp` killed by the job timeout or a lost runner leaves the upload open, holding its
parts until R2's 7-day rule aborts it. Whether open parts count as billed storage in the
meantime is **unverified** (the lifecycle page does not say; at worst 6.9 GB for 7 days,
~$0.02). Either way the run failed before `checkpoint`, so the next hour uploads the file
again from the start; a lone-file batch is fetched (~5 minutes at 22.7 MB/s) and uploaded
again. `--expected-size` is not needed: it exists for streams from stdin.

## 6. The runner's disk

| Item | Figure | Status |
|---|---|---|
| Documented spec, `ubuntu-latest` (Ubuntu 24.04) | 4 vCPU, 16 GB RAM, **14 GB SSD** | verified, GitHub runner reference page (fetched) |
| A `df -h` posted on runner-images issue 2840 | `/dev/root 84G, 34G used, 50G available` on `/`; no separate `/tmp` line | 2021-03-05, Ubuntu 20.04 image: dated, secondary |
| Today's image, `20260816.277.1` | ships AWS CLI 2.36.24, rsync 3.2.7, gnupg 2.4.4, xz 5.4.5, curl 8.5.0, coreutils 9.4; no `task` (setup-task installs it) | verified, Ubuntu2404 readme (fetched) |
| Free space today | **unverified** | `plan` prints `df -Pk staging` every run and refuses a file that would not fit with 1 GiB spare; the first real run's log is the figure |
| `/tmp` and the workspace on one filesystem | **unverified**; the 2021 `df` shows one root filesystem and no `/tmp` mount | `RUN` (`/tmp/ctan-run`) is ~205 MB at worst, so it does not matter which side it lands on; `STAGING` is under the workspace and is what `df` measures |

Budget per step, seed worst case:

| Step | Disk in use | Note |
|---|---|---|
| `list` | 51 + 38 MB | raw listing, `upstream.txt` |
| `state`, `rebuild` | + 3 + 38 + 20 MB | `.xz`, `applied.txt`, `bucket.txt` |
| `diff`, `plan` | + 38 + 38 + 17 MB | `changed.txt`, batches, `paths.txt` |
| `tlpdb` | + 24 MB | `.xz`, decompressed tlpdb, keyring |
| `fetch` (ordinary batch) | + ≤ 4.00 GB | listing bytes; rsync writes each file once, in place |
| `fetch` (lone batch) | + 6.87 GB | the largest object on CTAN |
| `checkpoint` | + 38 + 3 MB, then staging emptied | `applied.new` beside `applied.txt` |
| Peak | **~7.1 GB** | half the documented 14 GB; a tenth of the 2021 `df` |

The state file and the listing fit trivially; the only thing that can be tight is a future
installer larger than the free space, and `plan` says so by name before fetching anything.

## 7. The workflows

`sync.yml`:

```yaml
name: sync
on:
  schedule:
    - cron: '41 * * * *'   # hourly at a fixed random minute (register page); after :02, when dante touches timestamp
  workflow_dispatch:
    inputs:
      seed:
        description: 'The bucket is empty and there is no state file: upload everything'
        type: boolean
        default: false
      reconcile:
        description: 'List the bucket and rebuild the state this run (also bootstraps a missing state)'
        type: boolean
        default: false
      max_batches:
        description: 'Batches to work this run (4 GB each; the rest wait for the next hour)'
        type: string
        default: '4'
      cache:
        description: 'Cache rule and purges: on or off'
        type: choice
        options: [off, on]
        default: off
permissions:
  contents: read
concurrency:
  group: sync
  cancel-in-progress: false   # one run at a time; a slot that fires during a run waits and starts the moment it ends
jobs:
  sync:
    runs-on: ubuntu-latest
    timeout-minutes: 350   # under the 6-hour job limit; every batch checkpoints, so a cut-off costs one batch
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_ACCESS_KEY }}
      AWS_ENDPOINT_URL: https://${{ secrets.R2_ACCOUNT_ID }}.r2.cloudflarestorage.com
      AWS_REGION: auto
      CF_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
      CF_ZONE_ID: ${{ secrets.CF_ZONE_ID }}
      HEALTHCHECK_URL: ${{ secrets.HEALTHCHECK_URL }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: go-task/setup-task@a00fbb05ce67b35648be3c78cbc9fd85354c757e # v2.2.0
      - run: aws --version && rsync --version | head -1 && df -h .
      - run: >-
          task sync
          SEED=${{ inputs.seed == true }}
          RECONCILE=${{ inputs.reconcile == true && 'true' || 'auto' }}
          MAX_BATCHES=${{ inputs.max_batches || '4' }}
          CACHE=${{ inputs.cache || 'off' }}
```

What each line rests on (all fetched from docs.github.com today):

- `schedule`: POSIX cron, 5-minute minimum, "can be delayed during periods of high loads
  ... High load times include the start of every hour", disabled after 60 days without
  repository activity in a public repo. The minute 41 was drawn once with `shuf -i 5-59 -n 1`
  and is fixed; any minute after :05 works, and none should be :00. The cron minute names
  the slot, not the start: the one scheduled run in this repository's history so far (cron
  `30 3 * * *`) was created at 04:09:16 UTC on 2026-08-26, 39 minutes after its slot, and
  the next was 18 minutes late and counting. Lateness of 15 to 45 minutes is normal and a
  slot can be dropped. Nothing here waits for a slot: every run does the same five things
  from wherever the state stands, `clock` records how late it was, `report` prints it, and
  the reconcile keys on the slot's hour, not the clock's. Whether an off-peak minute reduces
  lateness is unverified; `sync-with-dante.md` has the cadence figures.
- `concurrency: sync` with the default `cancel-in-progress: false`: a running job keeps
  running; at most one run is pending per group; a newer trigger **replaces** the pending
  one; pending runs start in FIFO order when the running one ends ("ordering is not
  guaranteed" refers to start times, not to two runs overlapping). So a run that starts 40
  minutes late and works past the next slot collapses the triggers behind it into one
  pending run that starts the instant it ends, and a reconcile dispatched during a run
  queues behind it. There is no external cron or paid runner to fix the lateness; the
  constraints forbid both, and the design absorbs it instead.
- `timeout-minutes: 350` always; lateness does not touch it, since it bounds the job from
  its own start. With `MAX_BATCHES=4` a run is minutes, and nothing in the
  pipeline can hang for hours: rsync has `--timeout`, curl `--max-time`, the AWS CLI a read
  timeout, and `retry` gives up in 15.5 minutes. A manual push with a high `max_batches`
  checkpoints every batch, so the 6-hour job limit (fetched: "Each job in a workflow can run
  for up to 6 hours") costs at most one batch.
- `workflow_dispatch` inputs: `boolean`, `string` and `choice` types; the `inputs` context
  keeps booleans as booleans, and on a `schedule` trigger `inputs.*` is empty, so the
  expressions above yield `SEED=false RECONCILE=auto MAX_BATCHES=4 CACHE=off` on the cron.
  Turning the cache on for good is changing the `cache` default in this file, one reviewed
  line. `RECONCILE=auto` runs the reconcile in the run whose slot is 03:41, whether it
  starts at 03:41 or 04:25; `late.txt` carries the slot's hour. A dropped 03:41 slot means
  no reconcile that day and the next day's catches up.
- Secrets: the four today plus `CF_API_TOKEN` and `CF_ZONE_ID` (`caching.md` gives the
  scopes). `permissions: contents: read` is unchanged; nothing writes to the repo.
- The `df -h .` line puts the free-space figure section 6 lacks into every run's log.

`check.yml`:

```yaml
name: check
on:
  pull_request:
permissions:
  contents: read
jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - uses: go-task/setup-task@a00fbb05ce67b35648be3c78cbc9fd85354c757e # v2.2.0
      - run: task --dry --force sync   # renders every command, including status-gated tasks, without the network
      - run: task lint
```

`task --dry` evaluates `sh:` vars and `status:` checks, so with no `RUN` the gated tasks
would print "up to date" and hide their commands; `--force` renders them (verified). The
Taskfile's `sh:` vars are `ls` and `date` only, so the dry render needs neither credentials
nor network. `lint` validates the rule files with the tool list as it stands:

```yaml
  lint:
    desc: Every cloudflare/*.json must parse, hold at least one rule, and carry its own sha256 in the ruleset description
    cmds:
      - for: [cache-rules, bypass-rules, transform-rules, redirect-rules]
        task: lint-one
        vars: {F: 'cloudflare/{{.ITEM}}.json'}

  lint-one:
    internal: true
    requires: {vars: [F]}
    vars:
      J: {sh: 'cat {{.F}}'}
    cmds:
      - test {{ (.J | fromJson).rules | len }} -ge 1
      - test "$(sed -n 's/.*"description": *"sha256:\([0-9a-f]*\)".*/\1/p' {{.F}} | head -1)" = "$(sed '/"description": *"sha256:/d' {{.F}} | shasum -a 256 | cut -c1-64)"
```

`lint` also holds the one line that ties the Taskfile to the workflow:

```yaml
      - test "$(sed -n "s/.*cron: '\([0-9]*\) \* \* \* \*'.*/\1/p" .github/workflows/sync.yml)" = "{{.CRON_MINUTE}}"
```

Task's `fromJson` yields `null` on malformed input rather than an error (verified), and
`.rules` on `null` renders as nothing, so `test  -ge 1` fails the task; a well-formed file
with no `rules` array fails the same way. The second line pins the convention `rules`
relies on: the sha256 is of the file with its own description line removed, so it does not
depend on itself (`caching.md` states the convention; `rules` above hashes the whole file
and must use the same recipe as `lint-one` once that file settles it). This is syntax and
shape only. Semantics are checked by Cloudflare at `rules`, the first step of every `sync`,
which fails before any fetch. `jq` and `python3` are on the runner and would do this in one
line each, but neither is in the tool list; adding `jq` is the alternative if the template
check proves brittle.

## 8. Local development and verification

Every task that needs no credentials has an offline check with canned files in a directory
passed as `RUN`. The commands below were run on 2026-08-26 against the real listing.

| Task | Command | What it proves |
|---|---|---|
| `list` (normaliser only) | `task list SOURCE=$PWD/fixture/` with a small tree: rsync accepts a local path as `SOURCE`; or feed `RUN/listing.txt` and run the awk by hand | `awk ... listing.txt \| LC_ALL=C sort \| cmp - upstream.txt` on the real listing: `identical`; the 525 zero-byte lines carry `0` |
| `state`, `rebuild` | none offline; `RECONCILE=true` against a scratch bucket with no state key is the live test of the bootstrap | |
| `diff` | `task diff RUN=<dir>` with canned `upstream.txt` and `applied.txt` (delete 11 lines from a copy: `changed.txt` has 11 lines, `deleted.txt` 0; delete them from `upstream` instead: 0 and 11) | `comm` 0.1 s; whole-line vs path-only semantics |
| `plan` | `task plan RUN=<dir> STAGING=<dir>` with `changed.txt` = the listing minus its `systems/texlive/tlnet/` lines | 30 batches, sums as in section 3; the last file is the 10 root files; `cat batch-*.txt \| wc -l` equals `wc -l changed.txt`; `CEILING_GB=100` must fail; a `changed.txt` with one 20 GB line must fail the disk check |
| `batches` | `task --dry --force batches RUN=<dir> MAX_BATCHES=2` with three batch files | exactly two `batch` calls render (verified) |
| `clock` | `task clock RUN=<dir>` then `cat <dir>/late.txt`; or the awk alone on a fixed epoch (below) | lateness and slot hour without `date -d` or `date -r` |
| `tlpdb` | `task tlpdb RUN=<dir> SOURCE=$PWD/staging/../` pointing `SOURCE` at a local copy of tlnet's parent (`--files-from` works on local paths) | sha512, gpgv, the pin |
| `verify` | `task verify B=<batch> RUN=<dir> STAGING=$PWD/staging` with the repo's `staging/tlpkg` copied to `<dir>/tl/tlpkg`, a `paths.txt`-shaped `applied.txt` and an empty `deleted.txt`, and a batch file naming a few `archive/` lines present in `staging/` | containers match; one byte changed in a container fails; a tlpdb differing from `RUN/tl` fails; with `tlpkg/texlive.tlpdb` in the batch and one container path removed from `applied.txt`, `missing.txt` names it and the task fails |
| `publish`, `checkpoint`, `delete`, `rebuild`, `purge`, `rules`, `page` | no offline check is honest: `file://` is not an S3 endpoint and the AWS CLI is not installed locally by default. Use a scratch bucket: `task sync BUCKET=ctan-test HOST=<its r2.dev host> SEED=true CEILING_GB=1 CACHE=off` against a `SOURCE` of one small CTAN subtree (`.../CTAN/info/lshort/`) | the whole loop, checkpoints, a second run with an empty delta, a deletion, then `RECONCILE=true` after deleting the state key |
| `smoke` | `task smoke URL=file://<dir> RUN=<dir>` with `<dir>/timestamp` and an `upstream.txt` line for it: `curl -I file://` returns `Content-Length` (verified) | size compare; the tlpdb `cmp` when `RUN/tl` exists |
| `report` | `task report RUN=<dir>` | renders from canned counts |
| `retry` | `task retry CMD='exit 5'` with the sleeps scaled down | exit 23 fails at once; a permanent exit 5 gives up after five tries |
| `check.yml` | `task --dry --force sync` locally | every command renders with no network |

One runnable check per non-trivial piece of logic, the smallest thing that fails if it
breaks:

```sh
# normaliser: byte-identical to the reference, and every line has three fields
awk -f normalise.awk listing.txt | LC_ALL=C sort | cmp - upstream.txt && awk -F'\t' 'NF != 3 { exit 1 }' upstream.txt
# plan: partition is complete, no batch over CAP unless it is one line, the last file is the decision batch
cat RUN/batch-*.txt | LC_ALL=C sort | cmp - RUN/changed.txt
for f in RUN/batch-*.txt; do awk -F'\t' '{ s += $2 } END { exit !(s <= 4e9 || NR == 1) }' "$f" || echo "over cap: $f"; done
awk -F'\t' '$1 ~ /\// && index($1, "systems/texlive/tlnet/tlpkg/") != 1 { exit 1 }' "$(ls RUN/batch-*.txt | tail -1)"
# checkpoint: drop-then-add keeps one line per path and stays sorted
awk -F'\t' 'NR == FNR { d[$1] = 1; next } !($1 in d)' B applied.txt | cat - B | LC_ALL=C sort | cut -f1 | uniq -d | wc -l   # 0
# rebuild join: a key with the wrong size is not in the new state; an unknown key outside .state/ and .site/ is extra
printf 'a\t1\tm\nb\t9\tm\n' > up; printf '.site/index.html\t3\n.state/x\t5\na\t1\nb\t8\nz\t7\n' > bk
LC_ALL=C join -t "$(printf '\t')" -o 1.1,1.2,1.3,2.2 up bk | awk -F'\t' '$2 == $4' | wc -l     # 1 (a)
cut -f1 bk | grep -vE '^\.(state|site)/' | LC_ALL=C comm -23 - <(cut -f1 up)                  # z
# purge: only paths the state knows
printf 'new\nold\n' | LC_ALL=C comm -12 - <(printf 'old\n')                                    # old
# clock: 2026-08-26 04:20:16 UTC against a :41 slot is 39 min late and belongs to the 03 slot
echo 1787718016 | awk -v m=41 '{ late = ((int($1 / 60) % 60 - m + 60) % 60) * 60 + $1 % 60; print int(late / 60), int(($1 - late) / 3600) % 24 }'   # 39 3
```

Local hazards: macOS `awk` and `sort` agree with the runner's mawk and GNU sort on
everything used here (POSIX only; no `--output`, no `\t` in `sed`). Staging on a
case-insensitive filesystem loses 23 files to case collisions
(`documentation/epslatex/french/danger.eps` vs `Danger.eps` and the like); use a
case-sensitive volume or accept it for local runs. `rsync` on macOS is openrsync, whose
listing differs from rsync 3.2.7 only in the blank zero size, which the normaliser covers.

## 9. Variables and constants

| Variable | Value | Fork changes it? |
|---|---|---|
| `SOURCE` | `rsync://rsync.dante.ctan.org/CTAN/` | no; the register page requires the master |
| `HOST` | `ctan.ijosh.com` | yes |
| `BUCKET` | `tlnet` (name only; `S3` derives `s3://tlnet`) | usually not: bucket names are per account. Renaming to `ctan` is `seeding-and-migration.md`'s call; every reference is this one line |
| `PREFIX` | **removed**. Objects sit at the bucket root under CTAN's paths; an empty `PREFIX` would leave `{{.PREFIX}}/` producing keys with a leading slash | |
| `URL` | `https://{{.HOST}}` | derived |
| `STATE` | `.state/applied.txt.xz` | no |
| `RUN` | `/tmp/ctan-run` | no |
| `STAGING` | `{{.ROOT_DIR}}/staging` | no |
| `TL` | `systems/texlive/tlnet` | no |
| `TL_KEY` | `C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC` | no; rotate only against tug.org/texlive/verify.html |
| `CEILING_GB` | 200 | `cost-estimates.md` sets the number; today's tree is 133 GB, the 30-day churn 4.71 GB (computed) |
| `BATCH_GB` | 4 | only if the runner's free disk changes |
| `CRON_MINUTE` | 41 | with the cron line in `sync.yml`; `lint` fails when they differ |
| `MAX_BATCHES` | 4 | run-time flag; raise on a manual push |
| `SEED`, `RECONCILE`, `CACHE` | `false`, `auto`, `off` | run-time flags from the workflow; `cache` becomes `on` by editing its default in `sync.yml` |
| `RSYNC` | `rsync --timeout=300 --contimeout=60 --no-h` | no; `--no-h` keeps rsync 3.x from printing commas in the listing, which the normaliser also strips |
| `CURL` | `curl -fsS --connect-timeout 15 --max-time 60 --retry 6 --retry-connrefused --retry-max-time 600` | no |
| `AWS_FLAGS` | `--no-progress --cli-connect-timeout 60 --cli-read-timeout 300` | no |
| `CF` | `https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID` | no |
| `AWS_CONFIG_FILE` | `aws.config` in the repo | no |
| Secrets | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `HEALTHCHECK_URL`, `CF_API_TOKEN`, `CF_ZONE_ID` | all six, in the fork's repository settings |

Fixed inside tasks, not variables: the 400,000-line listing floor, 1,000 keys per
`DeleteObjects`, 100 URLs per purge, `sleep 0.3`, the 1 GiB disk headroom, the 03 slot for
`auto` reconcile, the reserved prefixes `.state/` and `.site/`, the four ruleset files, and
rsync's retryable exit codes `5 10 12 30 35`.

## Open questions

- Free disk on today's `ubuntu-latest` image: the docs say 14 GB, a 2021 `df` showed 50 GB
  free. The first run's `df -h .` line settles it; until then `plan`'s check is the guard.
- Whether rsync 3.2.7 with `--files-from` and `--ignore-missing-args` exits 0 when a listed
  path is gone. The manual describes the option as "ignore missing source arguments without
  error"; confirm once on the runner with a path that does not exist.
- Whether R2 bills storage for the parts of an incomplete multipart upload during the 7-day
  window. At most 6.9 GB for a week, ~$0.02, but it belongs in `cost-estimates.md`.
- Whether to drop tlcontrib's 261 versioned containers (0.46 GB) as tlnet's are. Nothing
  requests them by that name either; doing so is one more `||` in the normaliser.
- `aws s3 ls` exit codes: 1 for "no match" and 0 for "found" is what `state` relies on;
  the return-codes page was not fetched today. If a service error also returns 1, `state`
  fails safe (no `SEED`, no `RECONCILE`), so the risk is a spurious failure, not a spurious
  seed.
- `rules` with `CACHE=off`: `caching.md` says the only call is the one bypass rule `PUT`.
  This file also `GET`s the transform and redirect rulesets each hour (no `PUT` unless the
  file changed) so the landing page and the directory redirects are code too. If
  `caching.md` wants those two applied once by hand instead, drop them from `RULESETS`.
- The sha256 convention (`rules` hashes the whole file; `lint-one` hashes it without its own
  description line) must be one recipe; `caching.md` picks it and both tasks follow.
- A run that stops at `MAX_BATCHES` with the decision batch still queued reports success
  while the live `timestamp` is an hour or more old. mirmon's threshold is 1 day 4 h, so a
  seed's eight hours are inside it; `monitoring.md` may want the "left for the next hour"
  count in the ping body.
- A run more than an hour late is read as the next slot's (the lateness folds at 60 minutes),
  so `report` understates it and `auto` may reconcile a day off. GitHub's run metadata has the
  true slot; the job summary does not, and a slot that late is one to notice in the Actions list.
- `RECONCILE=true` given to bootstrap a missing state also runs the end-of-run reconcile
  (one more bucket listing, 497 Class A). Harmless; a separate flag would avoid it.
- The `lint` task's `fromJson` check is a template trick. If it proves fragile, `jq -e`
  is on the runner and one line in the tool list away.
- For `caching.md`: purge URLs need percent-encoding for 92 paths (23 with spaces, 69 with
  `( ) [ ] { } & # > ~ $`); the awk in `purge` is one way.
- For `verification-and-security.md`: the run-start tlpdb in `RUN/tl` is what every batch
  is checked against; the final `cmp` is what ties the uploaded tlpdb to it. Whether a
  container whose name the tlpdb does not mention should be refused rather than ignored
  (today it is ignored).
- For `monitoring.md`: `report` prints counts only; the job summary must stay under 1 MiB.

## Sources

Fetched 2026-08-26:

- https://taskfile.dev/docs/guide (`for`, `requires`, `preconditions`, `status`, `set`, `run`, `internal`, dynamic vars)
- https://github.com/go-task/task/blob/main/CHANGELOG.md (feature versions; latest v3.53.1, 2026-08-18)
- https://developers.cloudflare.com/r2/platform/limits/ (4.995 GiB single part, 10,000 parts, 1 write/s per key, 1,024-byte keys)
- https://developers.cloudflare.com/r2/objects/multipart-objects/ (5 MiB to 5 GiB parts, uniform size, 7-day abort)
- https://developers.cloudflare.com/r2/buckets/object-lifecycles/ (default abort rule)
- https://docs.aws.amazon.com/cli/latest/topic/s3-config.html (`multipart_threshold`, `multipart_chunksize`, binary suffixes, config-file only)
- https://docs.aws.amazon.com/cli/latest/topic/config-vars.html (`retry_mode`, `max_attempts`)
- https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html (`--recursive`, `--exclude`, `--cli-read-timeout`, `--expected-size`, output line)
- https://docs.aws.amazon.com/cli/latest/reference/s3api/delete-objects.html (JSON shape, 1,000 keys, quiet mode)
- https://docs.github.com/en/actions/reference/runners/github-hosted-runners (4 vCPU, 16 GB, 14 GB SSD, Ubuntu 24.04)
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows (schedule, `workflow_dispatch` inputs)
- https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency (pending replaced, FIFO)
- https://docs.github.com/en/actions/reference/limits (6-hour job)
- https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md (tool versions)
- https://github.com/actions/runner-images/issues/2840 comments via the GitHub API (2021 `df`)
- https://download.samba.org/pub/rsync/rsync.1 (`--files-from`, `--ignore-missing-args`, `-L`)

Computed from `SCRATCH/ctan-list-deref.txt`, `SCRATCH/upstream.txt` and `SCRATCH/tasktest/`
with the commands quoted in sections 2, 3, 4 and 8.
