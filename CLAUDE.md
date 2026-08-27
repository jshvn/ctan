# ctan

An hourly mirror of all of `CTAN/` (dante's `rsync://rsync.dante.ctan.org/CTAN/`) on
Cloudflare R2, served at `https://ctan.ijosh.com/` with every CTAN path at the bucket root.
About 496,000 objects and 133 GB; the largest file is 6.87 GB. Storage is the only bill,
about $1.86 a month; the pipeline refuses to run past 200 GB upstream.

`Taskfile.yml` and its comments are the design. `docs/reference.md` holds the numbers
behind it: the measured tree, the platform limits and their verification dates, the cost
model, the healthcheck settings and the runbook.

Everything is in a few files:

- `Taskfile.yml`: the whole pipeline. Bare `task` prints the menu; `task sync` runs
  `clock -> rules -> list -> state -> rebuild? -> diff -> plan -> tlpdb -> batches -> delete -> reconcile? -> smoke -> report -> ping`,
  where `batches` runs `fetch -> verify -> publish -> purge? -> checkpoint` per batch.
- `aws.config`: single-part uploads under 4 GiB, 512 MiB multipart parts above.
- `cloudflare/*.json`: the zone's rulesets (bypass cache, `/` -> `/index.html`, directory
  URLs -> ctan.org). `rules` applies one only when its stamped sha256 differs.
- `docker/Dockerfile`: the toolbox image with the runner's tool versions.
  `task run -- task <args>` runs any task inside it with the repo at `/work`.
- `.github/workflows/sync.yml`: hourly at :42, `timeout-minutes: 350`, dispatch inputs
  `seed`, `reconcile`, `max_batches`, `cache`. `check.yml`: `task --dry --force sync` and
  `task lint` on pull requests.

`README.md` is for users and is the mirror's only documentation page; the root URL serves
CTAN's own `index.html`. Operational detail belongs here and in Taskfile comments.

## Constraints

- No shell scripts. Logic lives in `Taskfile.yml`; workflows install tools and run one task.
- Tools are exactly `rsync`, `aws` (CLI v2), `gpg` (for `gpgv`), `shasum`, `xz`, `curl`,
  `task`. Network endpoints are exactly dante, R2, the public domain, healthchecks.io and
  `api.cloudflare.com`.
- Objects sit at the bucket root under CTAN's own paths. `.state/` is the one reserved
  prefix; CTAN has no dot-prefixed root entry, so it cannot collide.
- Secrets are exactly seven: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`,
  `AWS_REGION`, `HEALTHCHECK_URL`, `CF_API_TOKEN`, `CF_ZONE_ID`; the workflow passes each to the
  Taskfile by name. The last two are optional: without them `rules` is skipped.
- Recompute any change that adds storage against the 133 GB baseline and the 200 GB ceiling.

## Must knows

Each of these is a bug that has happened or a bill that would. Do not undo them.

- **The mirror is a list-diff, never a local tree.** The runner has 14 GB; the tree is 133 GB.
  `list` takes dante's `rsync -rL --list-only`, `diff` compares it with the state file the
  last run left in the bucket, and only the delta is fetched, in batches of at most 4 GB.
  The bucket is listed only in the daily `reconcile`.
- **The state line is `path TAB size TAB mtime`, path first, `LC_ALL=C` sorted.** Whole-line
  `comm` mis-pairs lines in any other order. Every `sort`, `comm` and `join` runs with
  `LC_ALL=C`, and every bucket listing is re-sorted because R2 lists keys out of byte order.
- **The state records what landed.** `merge` joins the batch against `find staging -type f`,
  so a path that vanished upstream between listing and fetch never enters the state.
- **No versioned containers.** `tlmgr` requests `archive/foo.tar.xz`; upstream stores
  `foo.r123.tar.xz` and symlinks to it. `normalise` drops tlnet's and tlcontrib's
  `*.r[0-9]*.tar.xz` and `update-tlmgr-r*`, and `fetch` uses `-L`, so each is stored once.
- **A missing state file fails the run** unless `SEED=true` (empty bucket) or
  `RECONCILE=true` (rebuild it from a bucket listing joined to upstream on size). Treating a
  missing state as empty would re-upload 133 GB.
- **The decision batch is last.** tlnet's `tlpkg/` and the root files (`timestamp` last of
  all) go in the final batch, after every container, and `verify` refuses that batch unless
  every container the tlpdb names is in the bucket after this run. `delete` waits for the
  hour in which every batch has landed, so the live tlpdb never names a removed container.
- **Never `aws s3 sync`.** `publish` is `aws s3 cp --recursive` (one PutObject per file,
  never a destination listing). Deletions come from `diff`, 1,000 keys per `DeleteObjects`,
  with the `Errors` array checked because the CLI exits 0 on it.
- **`checkpoint` is the last step of a batch.** The state is written once per batch, after
  the upload succeeded, as one PutObject. A run that dies anywhere repeats at most one batch
  the next hour. A run that stops at `MAX_BATCHES` with batches left is a success.
- **Cron lateness is normal.** Scheduled runs start 15 to 45 minutes after :42 and a slot
  can be dropped. `clock` records the lateness, `report` prints it, and `reconcile` keys on
  the slot's hour (03 UTC), not the clock's.
- **`index.html` at the root is CTAN's**, stored and served like every other file; the
  transform rule in `cloudflare/transform-rules.json` rewrites `/` to it. `README.md` is the
  documentation; there is no landing page of our own.
- **Do not trust the job log for counts.** `report` counts from `RUN` (`run/`), never the log.
- **A failed run is the only alert.** The check is cron `42 * * * *` UTC with a 3 h grace,
  which absorbs one dropped slot plus a late full run; healthchecks.io emails when the grace
  passes without `ping`, which also catches GitHub disabling the schedule after 60
  commit-free days. Pause the check before a seed — a multi-hour run outlasts the grace.
- The edge cache is off (`CACHE=off`: one bypass rule, no purges). `CACHE=on` swaps in
  `cache-rules.json` and purges changed keys per batch; switch it on only when R2 Class B
  reads exceed 5M a month for two months.

## Verifying a change

Every offline check runs inside the toolbox image; `fixtures/` (git-excluded) holds a real
dante listing and a signed `tlpkg/` tree.

- `task run -- task --dry --force sync` renders the pipeline without touching the network.
- `task run -- task lint` validates `cloudflare/*.json` and the cron minute.
- `task run -- task normalise RUN=/work/fixtures/run` from a canned `listing.txt`.
- `task run -- task diff RUN=<dir>` / `task plan RUN=<dir> STAGING=<dir>` from canned
  `upstream.txt`, `applied.txt`, `changed.txt`.
- `task run -- task merge B=<batch> RUN=<dir> STAGING=<dir>` for the state arithmetic.
- `task run -- task tlpdb RUN=<dir> SOURCE=/work/fixtures/tree/` and
  `task verify B=<batch> RUN=<dir> STAGING=/work/fixtures/tree` for the signed checks.
- `task run -- task smoke RUN=<dir> URL=file:///work/<dir>`; `task retry CMD='exit 5' RETRY_BASE=0`.
- `publish`, `checkpoint`, `delete`, `rebuild`, `rules` need credentials; use a scratch
  bucket: `task run -- task sync BUCKET=<scratch> SEED=true MAX_BATCHES=1 BATCH_GB=1`.
- Is the mirror fresh? `curl -s https://ctan.ijosh.com/timestamp`.

Five hazards, each of which has cost an evening:

- **A `>-` folded block keeps the newline** when a continuation line is indented further
  than the lines around it, and the rendered shell then splits into two commands. End the
  line with a backslash. `task --dry <task>` shows what actually renders.
- **An unquoted YAML scalar breaks on a literal `: `** anywhere inside it, including in
  embedded `sed` and `awk` text. Wrap the whole line in single quotes, doubling its own.
- **`task run -- task <x>` exits 201 for any inner failure.** go-task does not propagate the
  real code, so a test asserting a specific exit status can only assert "nonzero".
- **`--contimeout` is daemon-only**, so `{{.RSYNC}}` rejects a local-path `SOURCE=` in
  fixture tests. Override the whole var: `RSYNC='rsync --timeout=300 --no-h'`.
- **`tlpdb`'s status gate reads all of `changed.txt`**, not the batch, so it runs on
  essentially every real run. Only `verify`'s decision-batch branch keys on the batch.
