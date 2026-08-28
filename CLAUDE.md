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
  `clock -> list -> state -> rebuild? -> diff -> plan -> tlpdb -> batches -> delete -> reconcile? -> index -> smoke -> report -> ping`,
  where `batches` runs `fetch -> verify -> publish -> checkpoint` per batch.
- `aws.config`: single-part uploads under 4 GiB, 512 MiB multipart parts above.
- `docker/Dockerfile`: the toolbox image, and so the pipeline's environment. Every run
  happens inside it, locally and in Actions alike; `task run -- task <args>` runs any task
  in it with the repo at `/work`.
- `.github/workflows/sync.yml`: hourly at :42, `timeout-minutes: 350`, dispatch inputs
  `seed`, `reconcile`, `max_batches`. `check.yml`: `task --dry --force sync` and
  `task lint` on pull requests. Both call `task run --`, so the runner supplies nothing but
  `task` and a Docker daemon.

`README.md` is for users and is the mirror's only documentation page; the root URL serves
CTAN's own `index.html`. Operational detail belongs here and in Taskfile comments.

## Constraints

- No shell scripts. Logic lives in `Taskfile.yml`; workflows install `task` and run one task
  inside the image.
- Tools are exactly `rsync`, `aws` (CLI v2), `gpg` (for `gpgv`), `shasum`, `xz`, `curl`,
  `task`. The pipeline's network endpoints are exactly dante, R2, the public domain and
  healthchecks.io; building the image adds `docker.io` for the pinned base and, in the image
  only, the `task` and AWS CLI releases. The zone is configured by hand; nothing here calls
  the Cloudflare API.
- Objects sit at the bucket root under CTAN's own paths. `.state/` is the one reserved
  prefix; CTAN has no dot-prefixed root entry, so it cannot collide.
  `<HOST>.directory.index.html` is the one reserved file name: the page `index` draws in every
  directory, a name no upstream path can carry.
- Secrets are exactly five: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`,
  `AWS_REGION`, `HEALTHCHECK_URL`; the workflow passes each to the Taskfile by name. The four
  `AWS_*` are the whole requirement; without `HEALTHCHECK_URL`, `ping` is skipped.
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
- **Cloudflare's HTML rewriters must stay off for the mirror.** Email Address Obfuscation
  injects a script into every `text/html` response and changes its length, so the bytes stop
  matching CTAN's. A zone Configuration Rule turns it and Rocket Loader off for the mirror's
  hostname alone; `docs/reference.md` section 6 has it. Cloudflare also sends no
  `content-length` on `text/html` either way, which is why `smoke` sizes an object from a
  one-byte ranged read and not a HEAD.
- **`index.html` at the root is CTAN's**, stored and served like every other file; the
  zone's transform rule rewrites `/` to it, and a second one rewrites every other directory
  URL to that directory's page. `README.md` is the documentation; there is no landing page
  of our own.
- **Directory pages are drawn from the state, never from upstream.** R2 has no listings, so
  `index` writes `<dir>/<HOST>.directory.index.html` for every directory a run changed, from
  `applied.txt` (what the bucket holds), and `.state/indexed.txt.xz` records the state the
  pages last showed, so a run that dies before advancing it redraws the same pages next
  hour. No page enters the state (`merge` joins staging against the batch) and `reconcile`
  never counts one as an orphan. A missing `indexed` redraws all 27k once. The zone's
  second transform rule serves `/dir/` from that key; without it the pages exist and
  nothing else changes.
- **Do not trust the job log for counts.** `report` counts from `RUN` (`run/`), never the log.
- **A failed run is the only alert.** The check is cron `42 * * * *` UTC with a 3 h grace,
  which absorbs one dropped slot plus a late full run; healthchecks.io emails when the grace
  passes without `ping`, which also catches GitHub disabling the schedule after 60
  commit-free days. Pause the check before a seed — a multi-hour run outlasts the grace.
- The edge cache is off, by a zone rule, and the pipeline has no purge step. Caching saves
  nothing below 10M reads a month and a one-hour TTL saves nothing at any volume, because
  Cloudflare caches per datacentre; `docs/reference.md` sections 3 and 6 have the
  arithmetic. Turning it on means bringing per-batch purging back.

## Verifying a change

Every offline check runs inside the toolbox image; `fixtures/` (git-excluded) holds a real
dante listing and a signed `tlpkg/` tree.

- `task run -- task --dry --force sync` renders the pipeline without touching the network.
- `task run -- task lint` checks the cron minute against `CRON_MINUTE`.
- `task run -- task normalise RUN=/work/fixtures/run` from a canned `listing.txt`.
- `task run -- task diff RUN=<dir>` / `task plan RUN=<dir> STAGING=<dir>` from canned
  `upstream.txt`, `applied.txt`, `changed.txt`.
- `task run -- task merge B=<batch> RUN=<dir> STAGING=<dir>` for the state arithmetic.
- `task run -- task render RUN=<dir> STAGING=<dir>` from canned `applied.txt` and
  `indexed.txt`; run it on a real listing only inside the image, because CTAN has
  `obsolete/support/TeXshell/` and `texshell/` and a case-insensitive disk merges their
  pages.
- `task run -- task tlpdb RUN=<dir> SOURCE=/work/fixtures/tree/` and
  `task verify B=<batch> RUN=<dir> STAGING=/work/fixtures/tree` for the signed checks.
- `task run -- task smoke RUN=<dir> URL=file:///work/<dir>`; `task retry CMD='exit 5' RETRY_BASE=0`.
- `publish`, `checkpoint`, `delete`, `rebuild`, `index` need credentials; use a scratch
  bucket: `task run -- task sync BUCKET=<scratch> SEED=true MAX_BATCHES=1 BATCH_GB=1`.
- Is the mirror fresh? `curl -s https://ctan.ijosh.com/timestamp`.

Seven hazards, each of which has cost an evening:

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
- **Only what `RUNNER` names crosses into the container.** `report` reads
  `GITHUB_STEP_SUMMARY`, whose value is a path on the runner, so the variable is passed *and*
  the file bind-mounted at that same path; `GITHUB_EVENT_NAME` rides along, without which
  every run reads as `local` and the cron lateness line never prints. A host variable the
  pipeline reads and `RUNNER` does not name arrives empty, and the fallback hides it.
- **`-i -t` is an error when neither end is a terminal**, which is every CI run, so `RUNNER`
  adds `-t` only when both are. A container without it gets no terminal, and `task` colours
  its output only for one; the workflows pass `--color` instead.
