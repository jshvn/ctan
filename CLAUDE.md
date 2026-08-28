# ctan

An hourly mirror of all of `CTAN/` (dante's `rsync://rsync.dante.ctan.org/CTAN/`) on
Cloudflare R2, served at `https://ctan.ijosh.com/` with every CTAN path at the bucket root.
About 511,000 objects and 140 GB; the largest file is 6.87 GB. Storage is the only bill,
about $1.95 a month; the pipeline refuses to run past 200 GB upstream.

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
- `.github/workflows/sync.yml`: `workflow_dispatch` alone, `timeout-minutes: 350`, inputs
  `seed`, `reconcile`, `max_batches`. Nothing in this repo starts it:
  [`jshvn/dispatch`](https://github.com/jshvn/dispatch), a Cloudflare Workflow, POSTs the
  dispatch hourly at :42. `check.yml`: `task --dry --force sync` on pull requests. Both call
  `task run --`, so the runner supplies nothing but `task` and a Docker daemon.

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
  directory, a name no upstream path can carry. Every directory also holds that page under a
  second key, the directory without its trailing slash, which no upstream path can carry
  either: upstream is a filesystem, where a name is a directory or a file and never both.
- Secrets are exactly five: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`,
  `AWS_REGION`, `HEALTHCHECK_URL`; the workflow passes each to the Taskfile by name. The four
  `AWS_*` are the whole requirement; without `HEALTHCHECK_URL`, `ping` is skipped.
- Recompute any change that adds storage against the 140 GB baseline and the 200 GB ceiling.

## Must knows

Each of these is a bug that has happened or a bill that would. Do not undo them.

- **The mirror is a list-diff, never a local tree.** The runner has 14 GB; the tree is 140 GB.
  `list` takes dante's `rsync -rL --list-only`, `diff` compares it with the state file the
  last run left in the bucket, and only the delta is fetched, in batches of at most 4 GB.
  The bucket is listed only in the daily `reconcile`.
- **The state line is `path TAB size TAB mtime`, path first, `LC_ALL=C` sorted.** Whole-line
  `comm` mis-pairs lines in any other order. Every `sort`, `comm` and `join` runs with
  `LC_ALL=C`, and every bucket listing is re-sorted because R2 lists keys out of byte order.
- **The state records what landed.** `merge` joins the batch against `find staging -type f`,
  so a path that vanished upstream between listing and fetch never enters the state.
- **Every container is served under both its names.** tlnet stores `foo.r123.tar.xz` and
  symlinks `foo.tar.xz` to it; `tlmgr` asks for the stable name, and `fetch` uses `-L`, so
  the mirror holds both, as every other CTAN mirror does. `verify` checksums both against
  the signed tlpdb, deriving the revision-stamped name from the stanza's `revision`, and
  refuses a batch carrying an `archive/` path the tlpdb does not describe. A batch
  carrying no container skips that check whole: nothing to refuse, and no `RUN/tl`.
- **A missing state file fails the run** unless `SEED=true` (empty bucket) or
  `RECONCILE=true` (rebuild it from a bucket listing joined to upstream on size). Treating a
  missing state as empty would re-upload 140 GB.
- **The decision batch is last.** tlnet's `tlpkg/` and the root files (`timestamp` last of
  all) go in the final batch, after every container, and `verify` refuses that batch unless
  every container the tlpdb names is in the bucket after this run. `delete` waits for the
  hour in which every batch has landed, so the live tlpdb never names a removed container.
  `smoke` names `/timestamp` twice and asks for it only when the state records it: a seed
  capped at `MAX_BATCHES` has not reached the decision batch, and the bucket answers 404 to a
  key it has never held.
- **Never `aws s3 sync`.** `publish` is `aws s3 cp --recursive` (one PutObject per file,
  never a destination listing). Deletions come from `diff`, 1,000 keys per `DeleteObjects`,
  with the `Errors` array checked because the CLI exits 0 on it.
- **`checkpoint` is the last step of a batch.** The state is written once per batch, after
  the upload succeeded, as one PutObject. A run that dies anywhere repeats at most one batch
  the next hour. A run that stops at `MAX_BATCHES` with batches left is a success.
- **The hour a run belongs to is the hour it started.** `clock` writes `epoch UTC-hour` to
  `RUN/start.txt` at the top of the run, `report` prints the start time from the epoch, and
  `reconcile` keys `auto` on that hour being 03 -- read at the start because `reconcile` runs
  late enough that a long run would have crossed into the next hour by then. A run queued
  behind a longer one can start in 04 and skip the day's reconcile; the next day's does it.
- **Four Cloudflare defaults must stay off for the mirror.** One zone Configuration Rule
  turns all four off for the mirror's hostname alone, and `docs/reference.md` section 6 has
  each with its expression. Three of them alter `text/html` in flight, so the bytes stop
  matching CTAN's and a mirror that alters them is not a mirror: Email Obfuscation injects a
  script and encodes mailto addresses, Rocket Loader injects another, and Automatic HTTPS
  Rewrites turns plain `http://` links into `https://` — that last one can leave the length
  unchanged while the bytes differ, because the parser it runs inside also collapses
  whitespace in the tag it touched, which is why `smoke`'s canary compares the R2 object with
  the response rather than their lengths. The fourth, Browser Integrity Check, answers 403 to
  `libwww-perl`, `LWP`, `Python-urllib` and `PycURL`, all of which every other CTAN mirror
  serves; `smoke` asks for `/timestamp` as `libwww-perl` to catch it coming back. Cloudflare
  also sends no `content-length` on `text/html` whatever these are set to, which is why
  `smoke` sizes an object from a one-byte ranged read and not a HEAD.
- **`index.html` at the root is CTAN's**, stored and served like every other file; the
  zone's transform rule rewrites `/` to it, and a second one rewrites every other directory
  URL to that directory's page. `README.md` is the documentation; there is no landing page
  of our own.
- **Directory pages are drawn from the state, never from upstream.** R2 has no listings, so
  `index` writes `<dir>/<HOST>.directory.index.html` for every directory a run changed, from
  `applied.txt` (what the bucket holds), and `.state/indexed.txt.xz` records the state the
  pages last showed, so a run that dies before advancing it redraws the same pages next
  hour. No page enters the state (`merge` joins staging against the batch) and `reconcile`
  never counts one as an orphan. A missing `indexed` redraws all 27k once, which is also the
  only way a change to the page's markup reaches pages whose directory has not changed. The
  zone's second transform rule serves `/dir/` from that key; without it the pages exist and
  nothing else changes.
- **Every page is written under both of its keys, and only one of them names itself.**
  `<dir>/<HOST>.directory.index.html` serves `/dir/`; `<dir>` serves `/dir`, where every
  other mirror answers 301 and no Cloudflare rule can, because 13,259 upstream files carry
  no extension and 212 directories carry a dot, so nothing in the URL says which is which.
  Three things hang off this and each has a reason: the page carries a `<base href>`, or a
  relative link on `/dir` resolves against the parent; the slashless copies stage one tree
  per depth under `SLASH`, because no filesystem holds both `a/b` and `a/b/`; and their
  upload gives `--content-type text/html`, because a key with no suffix would otherwise go
  up as `binary/octet-stream` and download rather than draw. `reconcile` cannot spare the
  second key by name, so it spares every bare directory of the state. `docs/reference.md`
  section 7 has the measurements.
- **Do not trust the job log for counts.** `report` counts from `RUN` (`run/`), never the log.
- **A failed run is the only alert.** The check is cron `42 * * * *` UTC with a 3 h grace,
  which absorbs a queued run plus a full one; healthchecks.io emails when the grace passes
  without `ping`. It watches the dispatcher too: nothing here starts a run, so a scheduler
  that stops firing and a pipeline that stops finishing are the same missing ping. Pause the
  check before a seed — a multi-hour run outlasts the grace.
- The edge cache is off, by a zone rule, and the pipeline has no purge step. Caching saves
  nothing below 10M reads a month and a one-hour TTL saves nothing at any volume, because
  Cloudflare caches per datacentre; `docs/reference.md` sections 3 and 6 have the
  arithmetic. Turning it on means bringing per-batch purging back.

## Verifying a change

Every offline check runs inside the toolbox image; `fixtures/` (git-excluded) holds a real
dante listing and a signed `tlpkg/` tree.

- `task run -- task --dry --force sync` renders the pipeline without touching the network.
- `task run -- task normalise RUN=/work/fixtures/run` from a canned `listing.txt`.
- `task run -- task diff RUN=<dir>` / `task plan RUN=<dir> STAGING=<dir>` from canned
  `upstream.txt`, `applied.txt`, `changed.txt`.
- `task run -- task merge B=<batch> RUN=<dir> STAGING=<dir>` for the state arithmetic.
- `task run -- task render RUN=<dir> STAGING=<dir>` from canned `applied.txt` and
  `indexed.txt`. The image bind-mounts the repo, so a real listing renders onto the host's
  filesystem and the host's case sensitivity is what counts: CTAN has
  `obsolete/support/TeXshell/` and `texshell/`, which a macOS disk merges into one entry
  holding one of the two pages. Expect a full render there to come out one page short under
  each key. The runner is ext4 and draws both. `run-root`, whose two files differ only in
  the `timestamp` line, is the hour whose one dirty directory is the root: it must exit 0
  and leave `SLASH` empty.
- `task run -- task tlpdb RUN=<dir> SOURCE=/work/fixtures/tree/ RSYNC='rsync
  --timeout=300 --no-h'` and `task verify B=<batch> RUN=<dir> STAGING=/work/fixtures/tree`
  for the signed checks. `run-tl`'s batch is `b1.txt`; `run-tl-bad`'s is `changed.txt`.
- `task run -- task verify B=/work/fixtures/run-notl/b1.txt RUN=/work/fixtures/run-notl
  STAGING=/work/fixtures/empty` -- a batch with no tlnet path and no `RUN/tl`, the hour that
  skips `tlpdb`. It must exit 0. `run-belt` is the same with an `archive/` path the tlpdb
  does not name; it must not.
- `task run -- task smoke RUN=<dir> STAGING=<dir> URL=file:///work/<dir>`; `task retry
  CMD='exit 5' RETRY_BASE=0`. `STAGING` is what sizes the page the fixture serves, so the
  page check runs rather than being skipped; over `file://` only the INDEX key is read,
  because a filesystem cannot hold both `a/b` and `a/b/`.
- `publish`, `checkpoint`, `delete`, `rebuild`, `index` need credentials; use a scratch
  bucket: `task run -- task sync BUCKET=<scratch> SEED=true MAX_BATCHES=1 BATCH_GB=1`.
- Is the mirror fresh? `curl -s https://ctan.ijosh.com/timestamp`.

Eight hazards, each of which has cost an evening:

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
  "Essentially" is the trap: an hour whose delta touches no `systems/texlive/tlnet/` path at
  all skips `tlpdb`, and then `RUN/tl` does not exist. Anything in `verify` that reads or
  writes under `RUN/tl` must be guarded on the batch carrying a tlnet path, or it dies on a
  redirect into a directory nobody made -- rare enough to pass every fixture and every CI
  run and still break a live hour.
- **GNU `xargs` runs its command once on empty input.** A guard on the file feeding the pipe
  is not a guard on what reaches `xargs`: `render`'s `SLASH` awk drops the root, which has no
  slashless key, so an hour whose only dirty directory is the root sends it nothing and it
  runs `mkdir -p` with no operands. Two ordinary hours are that hour -- one where the delta is
  root files alone (`timestamp` by itself), and one where a deletion takes the last file under
  a top-level directory, leaving no dirty directory that still exists. Every `xargs` whose
  input can be filtered down to nothing takes `-r`.
- **Only what `RUNNER` names crosses into the container.** `report` reads
  `GITHUB_STEP_SUMMARY`, whose value is a path on the runner, so the variable is passed *and*
  the file bind-mounted at that same path -- a host variable the pipeline reads and `RUNNER`
  does not name arrives empty, and the fallback hides it.
- **`-i -t` is an error when neither end is a terminal**, which is every CI run, so `RUNNER`
  adds `-t` only when both are. A container without it gets no terminal, and `task` colours
  its output only for one; the workflows pass `--color` instead.
