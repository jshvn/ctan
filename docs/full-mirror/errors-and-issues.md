# Errors and issues

Every way the full mirror can fail, and what the pipeline does about each. Sibling files
own the mechanisms this file only names: the pipeline itself (`taskfile-architecture.md`), the
diff and batch model (`sync-with-dante.md`), the alerts (`monitoring.md`), the cache rule and
purge (`caching.md`), signatures (`verification-and-security.md`), limits (`limits.md`).

Everything here was checked on 2026-08-26 against the sources listed at the end, or computed from
the listing taken that day, or is marked unverified with the one thing that would verify it.

## The model in one paragraph

Two loops. Inside a run, each tool retries a transient error for minutes with exponential backoff
and jitter, then the run fails. Outside the run, the hourly cron is the retry, whenever GitHub
starts it (15 to 45 minutes after its slot as a matter of course, occasionally not at all): the
state file in the bucket stands as of the last committed batch, so the next run recomputes the remaining delta
and continues. Every write is idempotent (or its unsafe window is named below), so no retry is
ever unsafe. A failed run is loud (GitHub email); a run that does not happen is caught by
healthchecks. The dangerous failures are the ones that do neither; section 8 lists every one and
the cheapest way to see it.

## 1. Failure catalogue

The pipeline, in order: `rules`, `list`, `state`, `diff`, `plan`, then per batch `fetch`,
`verify`, `upload`, `purge`, `checkpoint`; then `delete`, `reconcile` (daily), `smoke`, `report`,
`ping`. Each table walks one step. Columns: what fails; how it shows (exit code, HTTP status, log
line); whether the run retries it; what the bucket and the state file look like afterwards; what
the next hourly run does; what a human must do. "Nothing" in the last column means the hour
heals it.

Exit codes are quoted from the tools' own documentation (section 2). "CLI 1" is the AWS CLI's
"one or more Amazon S3 transfer operations failed"; "CLI 254" is "the service returned an
error"; "CLI 253" is "missing configuration or credentials".

### rules (three `curl` calls to `api.cloudflare.com`, before anything is fetched)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| API down or slow | curl exit 28 (timeout), 7 (connect), 52/56 (reset); 5xx | yes: 408/429/5xx/timeouts/refused, up to 6 tries in 600 s | untouched | reruns | nothing |
| 1,200/5 min exceeded, 5-minute block | 429 with `retry-after` | curl waits `retry-after` (about 300 s) once, tries again, then gives up at 600 s | untouched | reruns; clears itself | nothing unless it repeats: something else is using the token |
| Token expired, revoked, wrong scope | 401/403, curl exit 22 | no (`-f`, not `--retry-all-errors`) | untouched | fails every hour | rotate `CF_API_TOKEN` (runbook R7) |
| Rules JSON rejected | 400, exit 22, body names the rule | no | untouched | fails every hour | fix the JSON in a PR; `check.yml` renders it |
| Ruleset concurrently updated elsewhere | 429 or 409 from the Rulesets API | curl retries a 429 | untouched | reruns | stop the other updater; the repo is the only writer by design |
| Dashboard edit since last hour | none: the PUT replaces every rule in the phase | n/a | rule reverted to the repo | same | if the edit was wanted, make it a PR (section 7) |
| Job killed here | run cancelled | n/a | untouched | reruns | nothing |

Subtle: `rules` runs first so a bad token or bad JSON fails before a byte is fetched, and so a
rule deleted in the dashboard is back within the hour. The cost of that is the silent revert in
row 6; section 7 says why that is the right trade.

### list (`rsync -rL --list-only` of the master into `RUN/upstream.txt`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| dante down, DNS gone, port closed | rsync exit 10 (socket), 35 (connect timeout), 5 (`@ERROR` / refused handshake) | yes: `retry` task, 5 attempts, at most 570 s of sleeps | untouched | reruns | nothing for a blip; a day of this is 24 failed runs (section 6) |
| dante at `max connections` | stderr `@ERROR: max connections (N) reached -- try again later`; exit 5 (from `clientserver.c`, which prints `@ERROR` and returns -1, and `errcode.h`; not observed on the runner) | yes, with jitter so we do not return on the same beat as the other refused clients | untouched | reruns | nothing |
| Transfer stalls | exit 30 after `--timeout` seconds of no data | yes | untouched | reruns | nothing |
| Protocol stream error, daemon restarted mid-list | exit 12 | yes | untouched | reruns | nothing |
| Module renamed or path moved | stderr `@ERROR: Unknown module`, exit 5 | yes, wastefully: 5 attempts before failing | untouched | fails every hour | edit `SOURCE` (runbook R9). Exit 5 is shared by a transient and a permanent cause; read the log line |
| Daemon rejects an option | exit 4 | no | untouched | fails every hour | fix the flags |
| Listing truncated but exit 0 | none from rsync; the line count is far below the state file's | run fails on the refetch-storm guard (under `diff`) if the shortfall is over 10% | untouched | reruns | nothing unless it repeats |
| Local disk full writing the 51 MB listing | rsync exit 11, `No space left on device` | no | untouched | reruns on a fresh runner | nothing |
| Listing has a line the parser cannot read | `verify-listing` fails: a line whose size field is not an integer | no | untouched | fails every hour | see "Listing format" below |
| Job killed here | cancelled | n/a | untouched | reruns | nothing |

Listing format. The parser must accept what the runner's GNU rsync prints, and that differs
from the macOS `openrsync` used for the 2026-08-26 listing in two ways. GNU rsync 3.x prints
sizes with thousands separators (`1,234,567`); openrsync prints none. openrsync prints an empty
size field for 525 files (`grep -cE '^[-dl][rwxsStT-]{9} +[0-9]{4}/' ctan-list-deref.txt`), for
example `info/fontname/fontname.fls`, and shows no file with size `0` anywhere in the
non-dereferenced listing (`awk '$1 ~ /^-/ && $2 == "0"' ctan-list-nolink.txt | wc -l` is 0), so
these are almost certainly zero-byte files. A field-based `awk` shifts every column on those
lines. Unverified: a GNU `rsync --list-only rsync://rsync.dante.ctan.org/CTAN/info/fontname/`
would settle it. The design does not care which tool is right: the normaliser strips commas,
and `verify-listing` fails the run if any line's size is not an integer or any path contains
rsync's control-character escape `\#` followed by three octal digits (rsync escapes every
control character in output, so a newline in a name would appear as `\#012`). The listing has
zero such escapes, 25 paths with spaces, and no non-ASCII bytes today
(`LC_ALL=C grep -c '[^ -~]' ctan-list-deref.txt` is 0). Fail closed on the first one that
appears: `--files-from` reads names literally and would ask dante for a file named with a
backslash, which exits 23.

### state (`aws s3 cp s3://tlnet/.state/applied.txt.xz - | xz -d`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| R2 down or slow | 5xx / `SlowDown` / read timeout; CLI exit 1 or 254 after retries | yes: CLI standard mode, `max_attempts = 10` | untouched | reruns | nothing |
| Credentials wrong or rotated | 403 `InvalidAccessKeyId` / `SignatureDoesNotMatch`, CLI 254; missing env, CLI 253 | no | untouched | fails every hour | rotate (runbook R7). This is the first R2 call, so nothing is written with bad credentials |
| Object missing | 404 `NoSuchKey`, CLI 1 | no | untouched | fails every hour: **fail closed**; only a dispatch with `SEED=true` treats no state as the seed | first run: dispatch with `SEED=true`. Otherwise the state was lost: rebuild it from the bucket with `RECONCILE=true` (runbook R4) |
| Object present but corrupt or truncated | `xz: Unexpected end of input` or `File format not recognized`, exit 1 (measured with xz 5.8.3) | no | untouched | fails every hour: **fail closed**, see below | rebuild it from the bucket with `RECONCILE=true` (runbook R4) |
| Download truncated in transit | same xz error; CLI may also report `IncompleteReadError` | the transfer manager retries streaming errors 5 times; xz catches what slips through | untouched | reruns | nothing |
| Clock skew over 15 min on the runner | 403 `RequestTimeTooSkewed` | no | untouched | reruns on another runner | nothing; GitHub runners are NTP-synced |
| Job killed here | cancelled | n/a | untouched | reruns | nothing |

Fail closed on a missing or corrupt state file. Treating "unreadable" or "absent" as "no
state" would turn a 3 MB glitch, or one mistaken `aws s3 rm`, into a full reseed: 496k
`PutObject` (half the free Class A month), 132.99 GB from dante, five to six hours of runner
time. Failing the run instead costs one hour of staleness per hour until a human dispatches once
with `RECONCILE=true`, which bootstraps a fresh state from the bucket listing joined to the
upstream listing (`sync-with-dante.md`; 497 `ListObjects`, free). `SEED=true` is the only way
to start from nothing, and nothing implies it. R2 lists and reads are strongly consistent, so a
`NoSuchKey` is never a transport artefact: the object is gone, and a human should know why.

### diff (`comm` between the sorted listing and the state)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| Locale not `C` on one side | silent: GNU `comm` "honors the rules specified by LC_COLLATE"; a file sorted in `en_US.UTF-8` (`a B c`) is misordered for `C` (`B a c`) and every line after the first misorder becomes "changed" or "deleted" | none | wrong delta is fetched and, worse, wrong deletions computed | same | `LC_ALL=C` on every `sort` and `comm` (already the rule in `stale`); `comm --check-order` so GNU comm warns `file 1 is not in sorted order` when it notices. Whether that warning is exit 1 in current coreutils is unverified; the `LC_ALL=C` invariant makes it moot |
| Listing much shorter than the state (dante half-listed, module emptied, wrong path) | `deleted` count is a large fraction of the state | run fails on the refetch-storm guard | untouched | reruns | nothing unless it repeats |
| Listing mtimes all older (dante restored from backup) | `changed` is most of the tree | run fails on the refetch-storm guard | untouched | fails every hour until forced | decide: `FORCE=1` to accept the refetch (section 6), or wait for dante |
| Empty listing with exit 0 | `test -s upstream.txt` fails | no | untouched | reruns | nothing |

Refetch-storm guard. The delta is trusted only when it is small: if `changed` + `deleted` exceeds
10% of the state's lines (about 50k today) and a state file exists, the run fails with the
counts and does nothing. `FORCE=1` skips the guard for one run. The seed (no state) is exempt. A
10% threshold is 8 times the worst churn day on record (5,911 files on 2026-08-23); nothing
legitimate reaches it in an hour. The guard is what turns "dante restored from backup" and
"listing truncated but exit 0" from silent 132.99 GB refetches into a decision.

### plan (batching by cumulative size from the listing)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| A single file larger than the runner's free disk | `guard` fails with the path and size before any fetch | no | untouched | fails every hour | nothing to do on our side; a file that big does not exist today (largest 6.87 GB against about 14 GB) |
| Runner disk smaller than 14 GB | `guard` prints `df` and fails if free space is under the batch cap plus margin | no | untouched | reruns on another runner | if it persists, lower the batch cap. The 14 GB figure is GitHub's documented SSD size; the free space actually seen by a job is unverified from here: `df -m .` in the job log has it |
| Sizes unparsable | caught at `list` | | | | |

### fetch (per batch: `rsync -rLt --files-from=batch.txt` into an empty `staging/`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| Transport (down, refused, stalled, stream error) | exit 5, 10, 12, 30, 35 | yes, `retry` task; each attempt restarts the batch into the same directory, so already-fetched files are skipped by the quick check (size and mtime) and the one file that was mid-transfer is sent again whole (no `--partial`; objects average 244 KB) | untouched; state as of the previous batch | reruns, repeats this batch from the state | nothing |
| A listed path no longer exists on dante | with `--ignore-missing-args`: stderr `link_stat ... failed: No such file or directory`, the file is skipped, exit 0; without it, exit 23 | no | the file is not in staging, so it is not uploaded and not in the state (the state is built from what landed) | its absence or return shows in the next listing | nothing |
| A file vanishes during the transfer | exit 24, `file has vanished` | no; treated as success if the missing-count check passes | as above | as above | nothing |
| More than 1% of the batch missing | run fails with the count | no | nothing uploaded from this batch | reruns | the listing is stale or dante is mid-move; if it persists, look at dante's news |
| Any other partial error (permission denied, I/O on dante) | exit 23 | no | nothing uploaded | reruns | if it persists, mail the maintainers' list with the path |
| Transferred file fails rsync's whole-file checksum | `WARNING: <file> failed verification -- update discarded (will try again)`; rsync re-requests it once; on the second failure `ERROR: ... failed verification -- update discarded`, exit 23 | once, by rsync itself | nothing uploaded | reruns | a dante-side disk problem if it repeats on the same file |
| Local disk full | exit 11, `No space left on device` | no | nothing uploaded | reruns; the batch cap should have prevented it | lower the cap if `df` shows less than 14 GB |
| Job killed here | cancelled | n/a | partial `staging/`, gone with the runner; state as of the previous batch | repeats the batch | nothing |

Why `--ignore-missing-args` and a state built from what landed. Without the option, a listed
path that vanished between the listing and the pull exits 23 and fails the whole run. dante
deleted 3 files in six hours on 2026-08-26; a vanished path in a 4 GB batch would cost the batch
and the hour. With
`--ignore-missing-args` rsync skips the path and exits 0 ("This does not affect subsequent
vanished-file errors if a file was initially found to be present and later is no longer there",
which is exit 24, treated the same way). The checkpoint appends the lines for files present in
`staging/` after the pull (`find staging -type f`, joined back to the listing for size and
mtime; state lines are `path<TAB>size<TAB>mtime`), never the planned lines, so the state
cannot name a file the bucket lacks. The 1% cap
keeps the option from hiding a bad `--files-from` (a wrong prefix would make every path
"missing"). Exit 23 is then reserved for real errors and still fails the run.

### verify (per batch, on what is local)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| tlnet tlpdb sha512 or signature bad, wrong fingerprint, expired key | `shasum: FAILED`; gpgv `BADSIG`/`EXPKEYSIG`/no `GOODSIG`; awk exit 1 | never | nothing from this batch uploaded (`tlpkg/` is the last batch, so every earlier batch is already live and consistent with the old tlpdb) | fails every hour | `verification-and-security.md`; key expiry is section 6 |
| tlpdb names a container the listing lacks | `verify` prints the container name | no | as above | reruns; clears when dante finishes its update | nothing; expected a few times a month |
| A fetched container fails its tlpdb checksum | `shasum: FAILED` with the path | no | as above | reruns | if it repeats, dante has a bad file; mail the list |
| `.xz` does not match `texlive.tlpdb` | `cmp` differs | no | as above | reruns | as above |
| Symlink-inflation guard: objects or bytes over the ceiling | counts printed, exit 1 | no | untouched | fails every hour | `cost-estimates.md`: raise the ceiling or exclude a directory |

### upload (per batch: `aws s3 cp --recursive staging/ s3://tlnet/`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| R2 5xx, `SlowDown`, connection reset, read timeout | CLI retries silently (`--debug` shows `Retry needed, retrying request after delay of`); after 10 attempts on one object, `upload failed:` line and CLI exit 1 at the end | yes, per request; the body is a seekable file and is rewound before each attempt | some objects of the batch are in the bucket, each one whole (`PutObject` is atomic: R2 reads are strongly consistent and never expose a partial object; the atomicity itself is implied, not stated, by Cloudflare's consistency page); state as of the previous batch | repeats the batch; the objects already there are re-put with identical bytes | nothing |
| R2 down for the whole step | as above, but the CLI's retry quota (500 tokens, 5 to 14 per retry) empties within a few dozen failures and later requests are not retried (`Retry needed but retry quota reached`) | yes, then fast-fail | as above | repeats the batch | nothing |
| 429 on a key written twice within a second | CLI 1 with a 429 in `--debug` | standard mode retries by parsed error code, not by status, so a 429 whose body code is not in its list is **not** retried (legacy mode retries any 429). Whether R2's body code is in the list is unverified | as above | repeats the batch | none needed: no key is written twice in a run except the state file, once per batch |
| Trailing-checksum request rejected | 400 `InvalidRequest` or similar on every object, CLI 1 | no | nothing new | fails every hour | section 5; set `request_checksum_calculation = when_required` and open an issue |
| A file vanishes from staging mid-upload | CLI exit 2 ("files marked for transfer were skipped") | no | others uploaded | repeats | impossible unless something else writes to the runner |
| Object over 4.995 GiB sent single-part | 400 `EntityTooLarge` | no | not stored | fails every hour | cannot happen with `multipart_threshold = 4GB` in `aws.config`; if it does, `AWS_CONFIG_FILE` is not pointing at the repo's file |
| Multipart interrupted (job killed, R2 error after 10 attempts) | CLI 1 | parts are retried like any request; on failure the transfer manager registers `AbortMultipartUpload` as cleanup (unverified from source; a killed process skips it either way) | an incomplete upload sits in the bucket; R2's default lifecycle rule "expire multipart uploads seven days after initiation" removes it; whether its parts bill as storage meanwhile is undocumented (worst case 7 GB for 7 days, about 1.6 GB-month, about $0.02) | repeats the file from scratch | runbook R8 to abort it sooner |
| Job killed here | cancelled | n/a | a partial batch, each object whole; state as of the previous batch | repeats the batch | nothing |

### purge (per batch, only with `CACHE` on: `POST /zones/{zone}/purge_cache`, 100 URLs per call, changed keys only)

With the adopted default `CACHE: off` this step is skipped and none of the rows below can
occur (`caching.md`). When the cache is on, only `changed` keys are purged, never `added`
ones: the cache rule stores no 3xx or 4xx response (`status_code_ttl` no-store), so an edge
cannot be holding a 404 for a key that did not exist yet, and a key that was never served has
nothing to purge.

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| API blip | 5xx, timeout | curl retries | batch is in the bucket, state not yet written | repeats the batch: re-put (idempotent), re-purge | nothing |
| 1,200/5 min block | 429, `retry-after` | curl waits it out once inside 600 s | as above | repeats the batch | if the seed trips it, raise the pause between calls (0.4 s gives 750 calls per 5 minutes) |
| 800 URLs/s purge limit | 429 from the purge endpoint | curl retries | as above | as above | the 0.3 to 0.4 s pause keeps us under 330 URLs/s |
| Purge accepted (200) but an edge still serves old bytes | none here; `smoke` catches it on the three sampled keys | no | bucket right, cache wrong | the next run's purge of the same key only happens if the key changes again | `caching.md`; purge by hand (runbook R6) |
| Token lacks `Cache Purge` | 403, exit 22 | no | batch in bucket, state not written | repeats the batch and fails again | fix the token; the batch is re-put every hour until then (at most 4 GB an hour of free Class A, so fix it the same day) |
| Job killed here | cancelled | n/a | batch in bucket, state not written | repeats the batch | nothing |

Note the ordering: purge is between upload and checkpoint. A purge that fails leaves the bucket
ahead of the state, which is the safe direction (the next run re-puts and re-purges). A
checkpoint before the purge would leave the state believing in a purge that never happened, and
nothing would ever redo it.

### checkpoint (per batch: write the new state, one `PutObject`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| R2 error on the state write | CLI 1/254 after retries | yes | batch in bucket; state unchanged | repeats the batch | nothing |
| Write accepted, response lost | the CLI retries and gets 200 the second time | yes | state written once or twice with identical bytes | continues | nothing |
| Job killed before the write | cancelled | n/a | batch in bucket; state = previous batch | repeats the batch | nothing |
| Two writes to `.state/applied.txt.xz` within a second | 429 | see upload row 3 | | | cannot happen: batches take minutes |

### delete (`s3api delete-objects`, 1,000 keys per call, then purge, then state rewrite)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| R2 error | CLI 254 after retries | yes | some keys gone; state still lists them | the listing still lacks them, so they are deleted again (a delete of a missing key is reported as `Deleted`, free) | nothing |
| Per-key errors inside a 200 | none from the CLI: **exit 0** with an `Errors` array in the JSON | no | some keys remain | the run fails on `test "$(... --query 'length(Errors || \`[]\`)' --output text)" = 0` | read the error code; `AccessDenied` means the token lost delete rights |
| Key was re-added upstream between the listing and now | none; the key is deleted anyway | n/a | key gone, bucket briefly behind upstream | the listing has it, the state does not: fetched again. Window is one hour at most | nothing |
| Purge of the deleted URLs fails (`CACHE` on) | curl exit | curl retries | keys gone; edge may serve them until TTL; state still lists them | deleted again (no-op), purged again | nothing |
| `delete-objects` against R2 from CLI 2.23 or later | unverified: the current pipeline uses `aws s3 rm` (single `DeleteObject`); `DeleteObjects` sends a CRC64NVME trailer and R2's compatibility page lists the operation as implemented without noting checksum headers | | | | test once with a throwaway key before merging (runbook R10) |

### reconcile (daily: `aws s3 ls --recursive` joined to the state on key and size)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| Listing fails partway | CLI 254 | yes, per page | untouched (read-only until it produces lists) | the hourly run skips the reconcile; tomorrow retries; the `reconcile` healthchecks check alerts if a day passes without one | nothing |
| Keys in bucket not in state (manual upload, a batch whose checkpoint never landed) | listed in `RUN/orphans.txt` | no | deleted this run (free) | | if it deletes something you put there by hand, it was in the wrong place: the bucket holds only CTAN plus `.state/` |
| Keys in state not in bucket, or wrong size | listed in `RUN/missing.txt` | no | removed from the state, so the next hour's delta re-fetches them | fetched | nothing |
| Same-size, different-content drift | invisible: the reconcile compares key and size only | never | stale object stays | stays until upstream changes the file's size or mtime | none available without reading every object; accepted. tlnet is covered by its checksums in `verify` |
| Listing exceeds 1 MiB in the job summary | GitHub drops the summary with an annotation; the step passes | n/a | | | `report` prints counts and the first 20 lines only |

### smoke (through `https://ctan.ijosh.com/`)

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| Domain down, DNS, Cloudflare incident | curl 6/7/28/5xx after retries | curl retries | everything committed; only the proof failed | reruns; `smoke` alone is cheap | nothing; if Cloudflare is down the mirror is down for users too |
| `timestamp` read back differs from staging | `cmp` differs | no | committed | reruns | the edge served a cached copy: `timestamp` must be excluded from the cache rule (`caching.md`) |
| A sampled key differs | `cmp` differs | no | committed | reruns; the key is not re-purged unless it changes | purge by hand (R6); if it repeats, the purge is not taking: `caching.md` |
| `cf-cache-status` not `HIT` on the second read of an archive key (`CACHE` on) | header mismatch | no | committed | reruns | the cache rule is not applying: the zone's rule differs from the repo (someone changed the zone plan, the domain, or the rule outside the phase we own) |
| `tlpkg/texlive.tlpdb.sha512` or `timestamp` is `HIT` | header mismatch | no | committed | reruns | the rule caches what it must not; fix the JSON (with `CACHE: off`, any `HIT` at all is this row) |
| Retry with output to a pipe | duplicate bytes: curl warns that on retry it "removes output data from a failed partial transfer that was written to an output file", but not from a pipe | | | | every `smoke` fetch writes `-o RUN/file` and `cmp`s the file, never `curl | cmp` |

`smoke` failing does not undo anything and does not make the mirror wrong; it withholds the
`ping`, so a broken domain never looks alive. After a `smoke` failure the bucket and state are
consistent and the next run re-checks.

### report and ping

| Failure | Symptom | Retried in run | Bucket / state after | Next hour | Human |
|---|---|---|---|---|---|
| Analytics query fails (`report`) | curl exit | curl retries | committed | reruns | nothing; the budget check is skipped that run and says so |
| Summary over 1 MiB | dropped with an annotation, step passes | n/a | | | keep lists to 20 lines |
| Budget check trips (Class B over 8M, storage over ceiling) | `report` exits 1 | no | committed | fails every hour until the month rolls or the ceiling is raised | `cost-estimates.md`; this is the intended alert |
| healthchecks unreachable | curl exit after `--retry 3` | yes | committed | reruns | nothing; one missed ping is inside the grace |
| `HEALTHCHECK_URL` wrong | 404, exit 22 | no | committed | fails every hour, and healthchecks alerts too | fix the secret |
| More than 5 pings a minute | 200 but not recorded | n/a | | | cannot happen at one ping an hour |

### Cross-cutting: the job

| Failure | Symptom | State after | Next hour | Human |
|---|---|---|---|---|
| `timeout-minutes` reached | run **cancelled** (not failed); GitHub notifies on cancelled runs by default, unless the account is set to failures only | as of the last checkpoint | resumes | nothing during the seed; afterwards, look at which step ran long |
| Runner lost, GitHub incident | run failed or stuck | as of the last checkpoint | resumes | nothing |
| Scheduled run starts 15 to 45 minutes late | normal operation, not a failure. GitHub: scheduled workflows "can be delayed during periods of high loads"; observed in this repository: the one scheduled run so far (cron `30 3 * * *`) was created at 04:09:16 UTC on 2026-08-26, 39 minutes after its slot, and the next was 18+ minutes late and not yet started at the time of writing. `report` prints the lateness (start minus slot) | unaffected | the same work, later; the delta is a little larger | nothing. No alert may fire on lateness alone (section 7) |
| One slot dropped (no run for that hour) | possible under load; nothing in the job list for the slot | unaffected | the next run's delta covers two hours | nothing. Two consecutive slots without a started run is the incident threshold (section 7) |
| A late run overlaps the next slot | a run that starts at :10 and needs 55 minutes (a release day, a seed) is still running when the next cron fires | unaffected | `concurrency: sync` with `cancel-in-progress: false` queues exactly one pending run, which starts the moment this one ends: no overlap, no lost hour, and the pending run's start is "late" by however long it waited | nothing |
| Schedule disabled after 60 days without repository activity | no runs; healthchecks alerts after period plus grace | | none | any commit re-enables it, plus "Enable workflow" in the Actions tab (runbook R11). Whether Dependabot's weekly commits count as activity is unverified; they are commits on the default branch, so they should |

## 2. Retry semantics per tool

All three tools were checked against their current documentation or source on 2026-08-26.

### AWS CLI v2

Facts, from the retries page and `botocore/retries/standard.py`:

- Default mode is `standard`; `max_attempts` default 3 (the first call counts). `legacy` is
  CLI v1's default (5 attempts, retries 429/500/502/503/504/509 by status). `adaptive` "is an
  experimental mode and is subject to change, both in features and behavior" (that warning is
  on the page today), and adds a client-side token bucket that lowers the request rate after
  any throttling response.
- Standard retries: error codes `RequestTimeout`, `RequestTimeoutException`,
  `PriorRequestNotComplete`; HTTP status 500, 502, 503, 504; exceptions `ConnectionError`,
  `HTTPClientError` (which cover resets and read timeouts); throttling error codes `Throttling`,
  `ThrottlingException`, `ThrottledException`, `RequestThrottledException`,
  `TooManyRequestsException`, `ProvisionedThroughputExceededException`,
  `TransactionInProgressException`, `RequestLimitExceeded`, `BandwidthLimitExceeded`,
  `LimitExceededException`, `RequestThrottled`, `SlowDown`, `EC2ThrottledException`. Throttling is
  matched on the parsed error code only, never on a bare status, so a 429 with an unlisted body
  code is not retried in standard or adaptive mode. It is in legacy.
- Backoff: `t_i = rand(0, 1) * min(2 ** attempt, 20)` seconds, full jitter, cap 20 s. The
  current source also carries a `non_throttling_base_scale` of 0.05 for 5xx and connection errors
  (so those retries wait a fraction of a second early on); whether the CLI build on the runner
  has that constant is unverified (`aws --version` in the job log names the botocore).
- Retry quota: a process-wide bucket of 500 tokens; a retry costs 5 (14 for the newer cost model,
  10 for a timeout), a success refunds 1. With 32 concurrent uploads against a dead R2, the quota
  empties in about a minute and further requests fail on their first attempt. This is why an R2
  outage fails the run fast instead of grinding for the whole `max_attempts` schedule per object.
- Timeouts: `--cli-connect-timeout` and `--cli-read-timeout` both default to 60 s; 0 means block
  forever. Settable in the config file as `cli_connect_timeout`, `cli_read_timeout`.
- Transfer manager (`s3transfer`): uploads have no retry of their own; botocore retries the
  `PutObject`/`UploadPart` and calls `reset_stream()` to seek the file back to its start first,
  so a mid-file connection reset re-sends the whole object (or part). Downloads retry
  `socket.timeout`, socket errors, `ReadTimeoutError`, `IncompleteReadError`,
  `ResponseStreamingError` up to `num_download_attempts = 5` on top of botocore's retries.
  `max_attempts` is per request and applies to every API call the transfer manager makes; there
  is no second `max_attempts` for the transfer as a whole.
- Return codes: 0 success; 1 one or more S3 transfers failed; 2 files skipped; 252 bad syntax;
  253 missing configuration or credentials; 254 the service returned an error; 255 other failure.

Recommendation, in `aws.config`:

```ini
[default]
retry_mode = standard
max_attempts = 10
cli_connect_timeout = 60
cli_read_timeout = 300
s3 =
    multipart_threshold = 4GB
    multipart_chunksize = 512MB
    max_concurrent_requests = 32
```

One `aws.config` for every upload: the 4 GB threshold keeps all but the five installers single
`PutObject`, and 512 MB chunks make each installer about 14 parts instead of 860.

Standard, not adaptive. Adaptive is still experimental, and
its rate limiter is process-wide: one 429 on the state key would slow all 32 upload workers for
the rest of the run. R2 publishes no per-bucket write rate to adapt to, only 1 write/s per key,
which the design never approaches. Adaptive buys nothing here and its behavior may change
under us. The loss is nothing we use.

Worst case per request with 10 attempts: sleeps of at most 1+2+4+8+16+20+20+20+20 = 111 s, plus
up to 10 read timeouts of 300 s if R2 accepts connections and never answers: about 55 minutes
for one pathological object. In practice the retry quota ends the run long before that, and the
job timeout is the backstop. `cli_read_timeout = 300` rather than the 60 s default because a
145 MB `PutObject` over a slow link can legitimately see 60 s of socket silence; unverified
whether a read timeout mid-upload is measured per socket read (likely) rather than per request,
in which case 60 s would also do.

### curl

From the option documentation in the curl source (`docs/cmdline-opts/`):

- `--retry N`: "Transient error means either: a timeout, an FTP 4xx response code or an HTTP
  408, 429, 500, 502, 503, 504, 522 or 524 response code." First wait 1 s, doubling to a 10-minute
  cap. "curl complies with the Retry-After: response header if one was present to know when to
  issue the next retry (added in 7.66.0)."
- `--retry-connrefused`: adds `ECONNREFUSED` to the transient set; a refused connection is not
  transient otherwise.
- `--retry-all-errors`: "the sledgehammer of retrying"; retries any error, and with `-f` every
  4xx too. Not used: 401 and 403 mean the token is wrong and must fail at once.
- `--retry-max-time S`: the timer includes sleeps; "Before each new retry is started, curl checks
  whether the elapsed time has reached the specified limit"; a transfer already started runs to
  completion.
- `--max-time S`: per attempt; "the maximum time counter is reset each time the transfer is
  retried."
- `-f`: exit 22 on HTTP 400 or above with no body output; "not fail-safe" for 401/407. With
  `--retry`, the transient codes above are retried even under `-f`.
- Retry and output: on retry curl truncates an `-o` file but cannot un-write a pipe. Every curl
  whose output is compared writes to a file.
- Exit codes: 6 could not resolve; 7 failed to connect; 22 HTTP error; 28 timeout; 35 SSL; 52
  empty reply; 55/56 send/receive failure; 92 HTTP/2 stream error.

Recommendation, one Task variable used by every curl line:

```
CURL: curl -fsS --connect-timeout 15 --max-time 60 --retry 6 --retry-connrefused --retry-max-time 600
```

Worst case per call: the retry timer stops new attempts at 600 s; the attempt running at that
moment may take up to 60 s more: 660 s. A Cloudflare 429 with `retry-after: 300` is waited out
once (300 < 600), tried again, and if it is still blocked the second wait would cross 600 s and
curl gives up: 600 s plus one attempt. That is the right shape: a 5-minute block that is still
in force after 5 minutes means something else is burning the token and the hour should fail.
`--retry 6` without `--retry-max-time` would wait 1+2+4+8+16+32 = 63 s; the max-time is what
lets the `retry-after` wait fit.

### rsync

rsync has no retry. From `errcode.h` and the manual's EXIT VALUES:

| Code | Name | Meaning | Class |
|---|---|---|---|
| 0 | RERR_OK | success | |
| 1 | RERR_SYNTAX | syntax or usage error | permanent |
| 2 | RERR_PROTOCOL | protocol incompatibility | permanent |
| 3 | RERR_FILESELECT | errors selecting input/output files, dirs | permanent |
| 4 | RERR_UNSUPPORTED | requested action not supported (option the daemon rejects, 64-bit files) | permanent |
| 5 | RERR_STARTCLIENT | error starting client-server protocol: the daemon said `@ERROR` (max connections, unknown module, access denied) | transport (retried; wasteful for the permanent causes) |
| 10 | RERR_SOCKETIO | error in socket IO | transport |
| 11 | RERR_FILEIO | error in file IO: local disk full, unwritable staging | local |
| 12 | RERR_STREAMIO | error in rsync protocol data stream: the daemon died or the connection was cut mid-stream | transport |
| 13 | RERR_MESSAGEIO | errors with program diagnostics | permanent |
| 14 | RERR_IPC | error in IPC code | permanent |
| 15, 16 | RERR_CRASHED, RERR_TERMINATED | sibling crashed or killed by a signal | local |
| 19, 20 | RERR_SIGNAL1, RERR_SIGNAL | SIGUSR1; SIGINT/SIGTERM/SIGHUP: the job was cancelled | cut off |
| 21 | RERR_WAITCHILD | waitpid() error | permanent |
| 22 | RERR_MALLOC | error allocating core memory buffers | local |
| 23 | RERR_PARTIAL | partial transfer due to error: at least one file failed (missing `--files-from` entry without `--ignore-missing-args`, permission denied, a file that failed its whole-file checksum twice) and the rest were transferred | content |
| 24 | RERR_VANISHED | partial transfer due to vanished source files: a file existed when the file list was built and was gone when its turn came | content, benign |
| 25 | RERR_DEL_LIMIT | `--max-delete` stopped deletions | n/a |
| 30 | RERR_TIMEOUT | timeout in data send/receive (`--timeout`) | transport |
| 35 | RERR_CONTIMEOUT | timeout waiting for daemon connection (`--contimeout`) | transport |
| 124 to 127 | remote command exited 255, killed, cannot run, not found | ssh transports only; not reachable with `rsync://` |

23 versus 24, precisely: both mean "the transfer finished, some files did not." 24 is only
vanished files: found in the initial scan, missing at transfer time. 23 is everything else,
including a `--files-from` name that was never found; the manual for `--ignore-missing-args`
says "it is normally an error if the file cannot be found. This option suppresses that error,
and does not try to transfer the file. This does not affect subsequent vanished-file errors".
So with the option on, a stale listing produces exit 0 or 24, never 23, and 23 is reserved for
real trouble.

Other options that matter:

- `--ignore-errors`: only tells `--delete` to proceed despite I/O errors. Not relevant; we never
  `--delete` in the list-diff design.
- `--partial`: not used on batch pulls. rsync's default deletes the partially transferred file
  when a transfer is interrupted, so a retried batch re-sends that one file whole; at 244 KB
  average there is nothing worth resuming, and no partial file can ever sit in `staging/` to
  be uploaded.
- `--timeout=300 --contimeout=60`: an I/O timeout is "if no data is transferred for the
  specified time"; the connection timeout is for the daemon handshake only.
- `--from0`: names in `--files-from` delimited by NUL. Not needed while `verify-listing` refuses
  any path with a control-character escape; noted in case CTAN ever ships one.
- Whole-file verification: "rsync normally verifies that each transferred file was correctly
  reconstructed on the receiving side by checking a whole-file checksum that is generated as
  the file is transferred". On mismatch (`receiver.c`) it logs `WARNING: <name> failed
  verification -- update discarded (will try again)`, re-requests the file once (`MSG_REDO`),
  and on the second failure logs the `ERROR:` form and exits 23. So bytes corrupted in transit
  from dante cannot land in `staging/`. Bytes already wrong on dante's disk copy faithfully;
  only tlnet's signed checksums catch those.

Recommended flags for the batch pull:

```
rsync -rLt --timeout=300 --contimeout=60 --ignore-missing-args --files-from={{.RUN}}/batch.txt {{.SOURCE}} {{.STAGING}}/
```

and for the listing the same timeouts with `--list-only`. Worst case per rsync call under the
`retry` task below: 5 attempts, each at most 60 s to connect plus a 300 s stall, plus sleeps of
at most 570 s: 5 × 360 + 570 = 2,370 s, about 40 minutes (the sleeps alone are 9.5 minutes; the
attempts' own timeouts dominate). Only a half-dead peer that accepts connections and sends nothing reaches this;
dante has never shown it, and the hourly job's `timeout-minutes` (55) is the backstop. A run
that spends 40 minutes on one listing has nothing left for batches and fails at the job limit,
which is the correct outcome for an hour in which dante is unusable.

## 3. The `retry` Task wrapper

Exactly as tested with go-task 3.53.1 on this machine (`which task`: `/opt/homebrew/bin/task`).

```yaml
vars:
  # Backoff base in seconds. The self-check sets it to 0.
  RETRY_BASE: '{{.RETRY_BASE | default 15}}'

tasks:
  retry:
    # No desc, so `task --list` hides it; not `internal`, so `task -x retry` stays callable
    # for the self-check below. CMD is the command; only rsync's transport exit codes retry.
    cmds:
      - |
        for i in 1 2 3 4 5; do
          rc=0; ( {{.CMD}} ) || rc=$?
          case $rc in
            0) exit 0 ;;
            5|10|12|30|35) ;;
            *) exit $rc ;;
          esac
          test $i -lt 5 || break
          s=$(( {{.RETRY_BASE}} * (1 << i) + RANDOM % (2 * {{.RETRY_BASE}} + 1) ))
          echo "rsync exit $rc, retry $i of 4 in ${s}s" >&2
          sleep $s
        done
        exit $rc
```

Policy: exit 0 returns at once; 5, 10, 12, 30, 35 sleep and retry; anything else returns that
code on the first attempt. Five attempts. Sleeps with base 15 are 30, 60, 120, 240 s plus 0 to
30 s of jitter each: at most 570 s. The subshell keeps `CMD`'s `exit` from ending the loop; the
`|| rc=$?` keeps Task's errexit (on by default for every `cmds` entry) from ending it. Task's
shell (`mvdan/sh`) supports `$RANDOM`, `<<` and `**` (checked: `task retry RETRY_BASE=0
CMD='echo $RANDOM $((1<<3)) $((2**3))'` prints a random number, 8, 8). The jitter is what keeps
a client refused at `max connections` from returning on the same beat as everyone refused with
it.

Self-check, a task in the same file. `FAKE` is a stand-in rsync that counts its calls in a file;
`n` is the count before the current call.

```yaml
vars:
  FAKE: 'n=$(cat /tmp/retry-count 2>/dev/null || echo 0); echo $((n+1)) > /tmp/retry-count'

tasks:
  retry-check:
    desc: Prove retry's exit-code policy with a fake rsync (exits 5 twice then 0; 23 once; 5 forever)
    cmds:
      - rm -f /tmp/retry-count
      - task: retry
        vars:
          RETRY_BASE: 0
          CMD: '{{.FAKE}}; test $((n+1)) -ge 3 || exit 5'
      - test "$(cat /tmp/retry-count)" = 3 && echo "ok - exit 5 twice then 0 succeeds after 2 retries"
      - rm -f /tmp/retry-count
      - rc=0; task -x retry RETRY_BASE=0 CMD='{{.FAKE}}; exit 23' 2>/dev/null || rc=$?; test "$rc" = 23 && test "$(cat /tmp/retry-count)" = 1 && echo "ok - exit 23 makes one attempt and returns 23"
      - rm -f /tmp/retry-count
      - rc=0; task -x retry RETRY_BASE=0 CMD='{{.FAKE}}; exit 5' 2>/dev/null || rc=$?; test "$rc" = 5 && test "$(cat /tmp/retry-count)" = 5 && echo "ok - permanent exit 5 makes five attempts and returns 5"
      - rm -f /tmp/retry-count
```

Output of `task retry-check` here:

```
rsync exit 5, retry 1 of 4 in 0s
rsync exit 5, retry 2 of 4 in 0s
ok - exit 5 twice then 0 succeeds after 2 retries
ok - exit 23 makes one attempt and returns 23
ok - permanent exit 5 makes five attempts and returns 5
```

Two things learned writing it. `task` exits 201 when a command fails, not with the command's
code; `task -x` ("Pass-through the exit code of the task command") is required wherever a caller
reads the code, which includes the two failing cases above. And a YAML plain scalar with `: ` in
it (an `echo "ok: ..."`) is parsed as a mapping and Task reports "invalid keys in command";
strings inside `cmds` avoid colon-space or are quoted whole.

## 4. Idempotency

Every write the pipeline makes, why a repeat is safe, and what a repeat costs.

| Write | Repeat is safe because | Repeat costs |
|---|---|---|
| `PutObject`, same bytes | last writer wins; the object is byte-identical; readers see one whole object at any moment | 1 Class A; the object's `LastModified` moves (nothing reads it) |
| `PutObject`, newer bytes | same; the later run always carries what dante listed later, so "newer" is the right winner | 1 Class A |
| `DeleteObjects`, key already gone | S3 semantics, which R2 follows on its compatibility page: "If the object specified in the request isn't found, Amazon S3 confirms the deletion by returning the result as deleted" | free |
| Purge of a URL never cached | the purge is a cache invalidation, not a bucket write; an unknown URL is a no-op (Cloudflare's page does not state this; unverified beyond the absence of any documented error for it) | one URL of the 800/s and one call of the 1,200/5 min |
| Ruleset `PUT`, unchanged rules | "The update operations described in this page (PUT requests) replace the entire list of rules in the ruleset"; the same list yields the same rules. Whether an identical PUT mints a new ruleset version is unverified; versions are free and unbounded as far as the page says | 1 API call |
| State file overwrite | one `PutObject`; atomic, so a reader sees the old file or the new, never half; the content is a pure function of the batches committed so far | 1 Class A |
| Multipart abort | `AbortMultipartUpload` of an unknown upload ID is a 404 `NoSuchUpload`, harmless; of a known one, the parts are discarded | free |
| `rsync --files-from` into an empty directory | rsync's quick check skips files already present with matching size and mtime; an interrupted file was deleted and is sent again whole | bandwidth for what was not yet fetched |

Where a repeat is not safe, the window and the mitigation:

- **Delete, then re-add upstream.** The deletion list is computed from the listing taken at the
  run's start. If dante removes a file, our run lists, and dante restores the file before our
  `delete` step, we delete a key upstream has. Window: from the listing to the delete step, a
  few minutes usually, up to the job's life. The next hour's listing has the key and the state
  does not: it is fetched again. Users see a 404 for at most an hour on a file that was
  genuinely gone for a moment. No mitigation needed beyond the hour.
- **Purge landing before the PUT is visible.** Not possible: R2 is strongly consistent
  ("readers will immediately see the latest object globally") and the purge is issued after
  the `cp` returns. What can happen is the opposite: a client hits the edge between the PUT
  and the purge and refills the cache with the new bytes, which is fine. The other edge
  hazard is a **404 for a newly added key** cached before the object existed; Cloudflare's R2
  consistency page: "If you upload an object to that same path, the cache may continue to
  return HTTP 404s until the cache TTL expires". That is why the cache rule stores no 3xx or
  4xx response (`status_code_ttl` no-store, `caching.md`). It is not a reason to purge
  `added` keys, and the pipeline never does.
- **Two runners at once.** `concurrency: sync` without `cancel-in-progress` allows one running
  and one pending; "any existing pending job or workflow in the same concurrency group will be
  canceled and the new queued job or workflow will take its place", so there is never a second
  runner. If the group were removed by mistake, two runs would read the same state, put the
  same objects (harmless, twice the Class A), and both write a state; whichever lands last may
  lack the other's batches, which the next hour re-puts, and the reconcile would catch anything
  else. Deletions are the same list on both sides. The damage is money during a seed (two
  seeds is 1M Class A, the whole free tier) and nothing at any other time. A lease object with
  `If-None-Match: *` (R2 supports the conditional headers on `PutObject`) would make the second
  runner fail at once; it needs an expiry rule for a crashed holder and is left as an open
  question because the concurrency group already does the job.
- **A checkpoint after a purge that lied.** If Cloudflare returns 200 and does not purge, the
  state records the batch and nothing re-purges those keys until they change again. `smoke`
  samples three of the run's keys to catch it in the same run; beyond that the stale copy
  lives until TTL. That is the accepted residual, and the TTL is the bound (`caching.md`).

## 5. Data-corruption classes

| Class | Can it reach users? | Defense |
|---|---|---|
| dante sends a truncated or corrupted stream | no | rsync's whole-file checksum on every transferred file; a mismatch is re-requested once, then exit 23. A cut connection is exit 12 or 30 and the batch is retried |
| dante's own copy is wrong (bad disk, bad upload to CTAN) | yes, for the unsigned tree; no, for tlnet | tlnet containers are checked against the signed tlpdb in `verify`; the rest of CTAN has no upstream integrity data and copies as-is, like every other mirror |
| R2 stores a truncated object | no | AWS CLI 2.23.0 and later (January 2025) "always calculate CRC64NVME checksum by default for operations that support it, such as PutObject or UploadPart"; botocore sends it as a trailer over TLS for streaming operations (`x-amz-trailer`, `aws-chunked`); R2 supports "CRC-64/NVME (CRC64NVME) FULL_OBJECT" since 2025-07-03. The service rejects a body that does not match. The CLI "will no longer automatically compute and populate the Content-MD5 header". Whether R2 accepts the trailer form of the checksum is not stated on its compatibility page (zero hits for "trailer" or "aws-chunked" in the page text); the evidence is that the tlnet pipeline has published daily with the runner's CLI since the change, and the job log's `aws --version` line records which build. If R2 ever rejects it, every upload fails with a 4xx on the same day and the fallback is `request_checksum_calculation = when_required`, which sends no integrity header at all; open an issue rather than live with that |
| R2 returns a truncated object to the state read | no | `response_checksum_validation = when_supported` (default) validates a returned checksum if R2 sends one (unverified whether `GetObject` on R2 returns `x-amz-checksum-crc64nvme`); independent of that, xz's own integrity check fails on truncation, exit 1, and the run fails closed |
| The edge caches a partial response | unverified whether Cloudflare can cache an incomplete origin response; the design assumes it does not | `smoke` reads three sampled keys through the domain and `cmp`s them; `tlmgr` checks every container against the signed tlpdb on the client, so a bad tlnet byte is an error the user sees, never a wrong install |
| `comm` on a non-C locale | yes: wrong deletions | `LC_ALL=C` on every `sort` and `comm`; `--check-order` |
| A listing path containing a newline | no | rsync escapes control characters as `\#NNN` in all output; `verify-listing` fails the run on any such escape. Today: zero |
| A listing line with a blank size | no | `verify-listing` requires an integer size on every file line (525 lines fail this with openrsync's output; GNU rsync's output on the runner is expected to pass, unverified) |
| State file xz-corrupted or missing | no | fail closed; rebuild with `RECONCILE=true`, runbook R4 |
| Log lines dropped by GitHub | counts only | `report` counts from `RUN/`, never from the log |

## 6. Upstream anomalies

- **tlnet mid-update.** The tlpdb names a container the listing lacks, or a fetched container
  fails its checksum: `verify` fails the run before the `tlpkg/` batch uploads; every earlier
  batch is live and consistent with the tlpdb clients already have, because `tlpkg/` is always
  the last batch. Next hour clears it. Expected a few times a month; nothing to do.
- **A package removed then restored.** Within one hour: nothing happens. Across hours: one
  `DeleteObjects` and a purge, then one `PutObject` and a purge. The cache serves nothing stale
  because both transitions purge.
- **dante down for a day.** 24 failed runs. The first sends the `sync` check a `/fail`, so
  healthchecks alerts once at the start and once on recovery; GitHub's failure email stays as
  the backstop, which by default is 24 emails on top. `monitoring.md` owns that preference;
  the data point here is that day-long dante outages happen. The mirror ages; mirmon
  (`min_sync` 1 day by default) marks it old after a day and CTAN removes mirrors that fall
  behind, on a schedule the register page does not state.
- **dante restored from backup with older mtimes.** Every restored file's listing line differs
  from its state line: the whole tree is "changed". The refetch-storm guard fails the run with
  the counts. A human runs once with `FORCE=1` (132.99 GB from dante, 496k Class A, five to six
  hours over two or three runs, inside the free tier if the month has room) or waits for dante
  to touch the files. Comparing size only would avoid the refetch but would then miss every
  same-size change forever; rsync itself treats an mtime difference in either direction as a
  change, and the state does the same.
- **dante changes the module name or path.** `@ERROR: Unknown module`, exit 5, retried five
  times (about 10 minutes wasted), then the run fails, every hour. Fix `SOURCE`. The register
  page gives `rsync://rsync.dante.ctan.org/CTAN` as the source and it has not changed in the
  project's history.
- **The TeX Live key.** tug.org's verify page today shows the primary
  `C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC` and `sub rsa2048 2016-03-19 [S] [expires:
  2027-07-13]`. Past that date, unless upstream extends the subkey again, `gpgv` reports
  `EXPKEYSIG` and no `GOODSIG`, and `verify` fails every hour: the whole run, so the entire
  mirror stalls, not only tlnet. That is deliberate: a mirror whose signed part cannot be
  verified should not publish. Because the keyring (`tlpkg/gpg/pubring.gpg`) arrives in the
  `tlpkg/` batch, an extension published upstream is picked up by the same run that needs it.
  If the primary key rotates, `TL_KEY` changes in a PR after checking tug.org, never from the
  mirror's own copy. Put 2027-06-13 in a calendar.
- **`CTAN.sites` changes our entry.** Nothing in the pipeline reads it. `report` can grep our
  host in `https://ctan.org/mirrors/mirmon` (`monitoring.md`); the register page is the
  contact path (`official-mirror-and-url.md`).
- **mirmon marks us stale while our runs pass.** Causes, in order of likelihood: the probe of
  `/timestamp` is answered from cache (the rule must exclude it; `smoke` asserts not `HIT`);
  Cloudflare challenges or blocks mirmon's probe (a WAF or bot rule on the zone; the probe has
  a 300 s timeout and a failed probe is state "bad"); our cron slid past mirmon's `max_poll`.
  Detection is the mirmon grep in `report`. Unverified: whether the zone's current settings
  challenge a non-browser `GET /timestamp`; `curl -A wget https://ctan.ijosh.com/timestamp`
  from outside would show it.

## 7. Platform anomalies

- **Scheduled runs start late, and sometimes not at all.** "The schedule event can be delayed
  during periods of high loads of GitHub Actions workflow runs. High load times include the
  start of every hour." Observed here: 39 minutes late on 2026-08-26, 18+ minutes and counting
  on the run after. So the design treats 15 to 45 minutes of lateness as the normal start and a
  dropped slot as something that happens, and never assumes a run begins at its cron minute.
  There is no fix inside the constraints (no external cron, no paid runners), so the design
  absorbs it: the state model already makes any hour's work resumable by the next; the
  `concurrency` group turns an overlap into one queued run; mirmon's 28-hour freshness band
  does not notice an hour; `report` prints the lateness so the trend is in every job page.
  What must not fire on lateness alone: GitHub sends nothing for a run that has not started, so
  it cannot; the healthchecks `sync` check must not either, so its period plus grace must cover
  a 45-minute late start, the run itself (up to `timeout-minutes`, 55) and one dropped slot
  (60 minutes) before it pages, which is at least 2 h 40 min of grace on a 1-hour period; the
  exact values and the `/start` handling are in `monitoring.md`. The threshold that turns
  lateness into an incident is two consecutive slots with no run started, which is what that
  grace encodes: at that point either the workflow is disabled (next bullet), GitHub Actions is
  down, or the repository is in a state where the cron no longer fires, and a human dispatches
  by hand (runbook R13). Whether a minute away from `:00` to `:05` reduces the lateness is
  unverified; it costs nothing to choose one.
- **60-day disablement.** "In a public repository, scheduled workflows are automatically
  disabled when no repository activity has occurred in 60 days." Hourly runs are not activity.
  healthchecks catches it; a commit and "Enable workflow" fix it (R11).
- **Runner disk smaller than documented.** 14 GB SSD is the documented figure for the standard
  public-repo runner. `guard` reads `df` before every batch and fails with the numbers if free
  space is under the batch cap plus 1 GB. Base the cap on what the first real run logs.
- **Killed at 6 hours or at `timeout-minutes`.** "Each job in a workflow can run for up to 6
  hours of execution time. If a job reaches this limit, the job is terminated and fails";
  `timeout-minutes` cancels. Either way the state is as of the last checkpoint and the pending
  hourly run starts at once. Steps guarded with `if: always()` still run ("returns true, even
  when canceled"), which is how the `sync` check's `/fail` ping is sent from a cancelled or
  failed job (`monitoring.md`: `/start` at the top of the run, success or `/fail` at the end).
- **The `concurrency` group drops a pending run.** By design: at most one pending; the newer
  pending replaces the older, which shows as cancelled. During the seed, each hourly trigger
  cancels the previous hour's pending run, one cancellation notification an hour unless
  notifications are failures-only. Harmless: the pending run would have done identical work.
- **Secrets rotated.** R2 credentials: the first R2 call is the state read, 403, no write is
  made. Cloudflare token: `rules` fails first. `HEALTHCHECK_URL`: `ping` fails after everything
  else has succeeded, and healthchecks alerts as well. All loud.
- **Cloudflare API token expired.** Same as rotated. Create the token without an expiry, or with
  one and a calendar entry.
- **Cloudflare 1,200 per 5 minutes.** "If you exceed this limit, all API calls for the next five
  minutes will be blocked, receiving a HTTP 429"; the `retry-after` header "is only returned
  when the request has exceeded the rate limit". Budget per hourly run: 3 rules calls, 1 or 2
  analytics, and with `CACHE` on 1 to a few purge calls: single digits. A seed with `CACHE`
  on: the purge stream at one call per 0.3 s is 1,000 calls per 5 minutes plus the rest, a 15%
  margin; 0.4 s (750) is safer and adds about 10 minutes to the seed. With `CACHE: off` the
  seed makes no purge calls. Per-IP limit 200/s, irrelevant at 3/s.
- **R2 429 per key.** "Concurrent writes to the same object name (key) at a higher rate return
  HTTP 429". Only the state key is written more than once per run, minutes apart.
- **R2 outage.** 5xx retried, quota empties, run fails in minutes, next hour. `PutObject` is
  atomic so nothing half-written exists; the reconcile is the backstop for anything strange.
- **Zone plan changed.** The pipeline stays inside Free limits (10 cache rules, 5 purge-
  everything a minute, 800 URLs/s), so any plan works. `smoke`'s `HIT`/not-`HIT` assertions
  prove the rule still applies after a plan change.
- **Transform Rule deleted in the dashboard.** Back within the hour; `rules` PUTs the transform
  phase from `cloudflare/transform-rules.json`. Until then `/` is a 404 and the mirror works.
- **A cache rule edited in the dashboard.** Overwritten within the hour, silently. Desirable:
  the repo is the only truth, a fork reproduces it, and a bad click cannot quietly cost money
  for a month. The cost is that a person "fixing" something in the dashboard is undone without
  being told. Two mitigations, both cheap: every rule's `description` field says "managed by
  the ctan repo; edits here are overwritten hourly"; and the break-glass path is documented in
  CLAUDE.md: disable the workflow, then edit, then PR the JSON, then re-enable.

## 8. The alerting contract

Fails loud: the run exits non-zero and GitHub emails. Degrades silently: the run passes and
nothing is wrong on the job page. healthchecks covers the third case, "no run".

Loud, by construction: every transport failure past its budget; bad credentials and tokens;
rejected rules JSON; integrity failures in `verify`; the refetch-storm guard; disk and size
guards; `delete-objects` per-key errors (because we parse them); `smoke` mismatches and cache
status assertions; budget trips in `report`; a wrong healthchecks URL; a job cut off (as
cancelled).

Silent, and what sees each:

| Silent failure | Why the run passes | Cheapest detection | Owner |
|---|---|---|---|
| Cache rule wrong in the repo: every GET hits R2 | uploads and purges succeed | `smoke` asserts `cf-cache-status: HIT` on a second read; `report` reads month-to-date Class B and fails over 8M | in-run; `monitoring.md` |
| Purge returns 200 and does not take (`CACHE` on) | the API said yes | `smoke` `cmp`s three sampled uploaded keys through the domain | in-run |
| A 404 cached for a key added this run | the object is there; the edge remembers the miss | the cache rule stores no 3xx or 4xx (`status_code_ttl` no-store); `added` keys are never purged and need not be | `caching.md` |
| State drifts from the bucket (a checkpoint that never landed, a manual edit) | the hourly path never lists the bucket | daily reconcile: key and size join | in-run, daily |
| Same-size content drift after a state rebuild | reconcile compares size only | none cheap; tlnet covered by checksums; accepted | |
| dante-side corrupt file outside tlnet | copied faithfully | none; same as every mirror | |
| Cron stops (60 days, disabled workflow, GitHub incident) | no run at all | the `sync` healthchecks check goes late (period 1 h plus grace); the `reconcile` check covers a day without a reconcile | `monitoring.md` |
| Scheduled run 15 to 45 minutes late, or one slot dropped | normal operation; nothing failed and nothing must page | `report` prints the lateness (start time minus the cron slot); the `sync` check's grace absorbs a late start, the run and one dropped slot; two consecutive missed slots page | `monitoring.md` |
| healthchecks itself down | nothing pings the pinger | a second, independent probe of `/timestamp` age, or accept | `monitoring.md` |
| mirmon marks us bad (WAF challenge, cache) while runs pass | the probe is not ours | `report` greps mirmon for our host and prints its state; fail the run if "bad" two runs in a row | `monitoring.md` |
| Cancelled pending runs during the seed | not failures | nothing needed; note it in the seed day's checklist | `seeding-and-migration.md` |
| Dashboard edits reverted hourly | that is the design | rule descriptions say so | this file, section 7 |
| Job summary over 1 MiB dropped | step passes | keep lists to 20 lines | in-run |
| Log lines dropped | cosmetic | counts from `RUN/` | in-run |
| `timestamp` copied is the previous hour's | a run that lists before dante's `:02` touch copies the old one; in practice runs start 15 to 45 minutes late, so this needs a run on its exact minute | choose a cron minute after `:05`; mirmon's 28-hour band absorbs an hour either way | `sync-with-dante.md` |
| An incomplete multipart upload left behind | free abort never ran | R2 lifecycle expires it after 7 days; `report` lists `list-multipart-uploads` daily (Class A, one call) | in-run, daily |

Every row above is either in the run (free, no new dependency) or in `monitoring.md`. Nothing
here proposes a new service.

## 9. Runbook

Prerequisites for the local commands: the four R2 variables in the environment as `sync.yml`
maps them (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT_URL`, `AWS_REGION=auto`),
`AWS_CONFIG_FILE=aws.config`, and for Cloudflare `CF_API_TOKEN` and `CF_ZONE_ID`. `gh` for the
GitHub side. Every task is safe to rerun unless its entry says otherwise: "safe" means a second
run makes the same writes with the same bytes, or none.

**R1. Diagnose a failed run.**
```
gh run list --workflow sync.yml --limit 5
gh run view <id> --log-failed | tail -50
curl -sI https://ctan.ijosh.com/timestamp | grep -i -E 'last-modified|cf-cache-status'
aws s3 cp s3://tlnet/.state/applied.txt.xz - | xz -d | wc -l
```
The failed step names the class (section 1). The state line count against the listing's tells
you how far behind the mirror is.

**R2. Re-run.** `gh workflow run sync.yml` or `task sync`. Safe: the state is as of the last
checkpoint and the run recomputes the rest. During a seed this is the resume.

**R3. Force past the refetch-storm guard.** `gh workflow run sync.yml -f force=1` (a
`workflow_dispatch` input mapped to `FORCE`), or `task sync FORCE=1`. Safe but expensive: up to
the whole tree is re-put. Do it once you have read the counts the guard printed.

**R4. Rebuild the state file** (missing or corrupt state, or a state you distrust).
```
gh workflow run sync.yml -f reconcile=true     # or: task sync RECONCILE=true
```
Lists the bucket (497 `ListObjects`, free) and joins it to the upstream listing to write a
fresh state (`sync-with-dante.md`); same-size objects are taken as current, so a same-size
change that happened while the state was unusable is missed until upstream touches the file.
Safe: the rebuild uploads nothing, and the next hourly delta covers what the join left out.

**R5. Seed on purpose.**
```
gh workflow run sync.yml -f seed=true          # or: task sync SEED=true
```
Deleting the state object does not start a seed; it fails every run (fail closed). Safe for
correctness, not for the budget: the delta is the whole tree, about 496k Class A and 132.99 GB
from dante. Do it mid-month with room in the free tier, after posting to the maintainers' list,
per `seeding-and-migration.md`.

**R6. Delete a key, or purge one.**
```
aws s3 rm s3://tlnet/<key>
curl -fsS -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H 'Content-Type: application/json' \
  --data '{"files":["https://ctan.ijosh.com/<key>"]}'
```
Safe. If upstream still has the key, the daily reconcile finds it missing and the hourly run
after that re-fetches it; editing the state by hand to get it back sooner is not worth the
risk; wait for the reconcile.

**R7. Rotate a secret.**
```
gh secret set AWS_ACCESS_KEY_ID   # paste the new token's id
gh secret set AWS_SECRET_ACCESS_KEY
gh secret set CF_API_TOKEN
gh workflow run sync.yml && gh run watch
```
Then delete the old token in the Cloudflare dashboard after a green run. Safe: the first call
with the new credentials is a read.

**R8. Abort a stray multipart upload.**
```
aws s3api list-multipart-uploads --bucket tlnet --query 'Uploads[].[Key,UploadId,Initiated]' --output text
aws s3api abort-multipart-upload --bucket tlnet --key <key> --upload-id <id>
```
Free and safe; R2 does it itself after 7 days.

**R9. dante moved.** Edit `SOURCE` in `Taskfile.yml`, PR, merge. Until merged every run wastes 10
minutes retrying exit 5 and fails; that is the correct loud behaviour.

**R10. Prove `delete-objects` works against R2 with the runner's CLI** (once, before merging the
delete step).
```
echo x | aws s3 cp - s3://tlnet/.state/probe
aws s3api delete-objects --bucket tlnet --delete '{"Objects":[{"Key":".state/probe"},{"Key":".state/never-existed"}]}'
```
Expect both keys under `Deleted` and no `Errors`. Safe: `.state/` is outside the mirror tree.

**R11. The schedule stopped.** Actions tab, the workflow, "Enable workflow"; then any commit
(a docs line is enough) so the 60-day clock restarts. healthchecks recovers on the next ping.

**R12. Purge everything** (last resort, after a rule change that cached what it should not).
```
curl -fsS -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" -H 'Content-Type: application/json' \
  --data '{"purge_everything":true}'
```
Five a minute on Free. Safe for correctness; every install after it is cold for a while.

**R13. The run did not start.**
```
gh run list --workflow sync.yml --limit 3      # createdAt against the cron slot
gh workflow view sync.yml                      # "disabled" means the 60-day rule or a manual stop
```
Less than an hour past the slot: normal, wait. One slot with no run: normal, wait for the next.
Two consecutive slots with no run started: check the GitHub status page for an Actions
incident; if the workflow is disabled, R11; otherwise dispatch by hand with
`gh workflow run sync.yml`, which starts at once and is safe (the state model makes it the same
run the cron would have started). Do not add an external trigger to compensate; the constraints
forbid it and the hour's work is not lost, only late.

## Open questions

- Does R2 accept the `aws-chunked` trailer form of `x-amz-checksum-crc64nvme` that the CLI
  sends for `PutObject`? The daily tlnet publish says yes in practice; no Cloudflare page says
  so. One `--debug` upload against R2 showing the request headers and a 200 would settle it.
- What error code does R2 put in the body of its per-key 429, and is it in botocore's
  throttled list? Only matters if a key is ever written twice in a second.
- Does GNU `comm` exit non-zero when it detects unsorted input without `--check-order`? Moot
  under `LC_ALL=C`, but the runbook would like to know what a stray warning means.
- Is a purge of a never-cached URL a documented success? Assumed; free either way.
- Does an identical ruleset `PUT` create a new version every hour, and is there a cap on
  versions? Cloudflare's page is silent.
- Do incomplete multipart parts bill as storage during the 7 days before R2 expires them?
  Worst case is cents; unverified.
- Is the exit code for `@ERROR` from the daemon 5 on the runner's rsync? Inferred from
  `clientserver.c` and `errcode.h`, not observed; the retry list would need no change if it were
  another transport code.
- Would a WAF or bot setting on the zone challenge mirmon's probe of `/timestamp`?
  (`official-mirror-and-url.md`, `monitoring.md`.)
- Do Dependabot commits count as "repository activity" for the 60-day rule?
  (`monitoring.md`.)
- Whether to add a lease object with `If-None-Match: *` against a mis-set concurrency group.

## Sources

Fetched 2026-08-26.

- AWS CLI retries: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-retries.html
- AWS CLI config file settings: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
- AWS CLI command line options (`--cli-connect-timeout`, `--cli-read-timeout`): https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-options.html
- AWS CLI return codes: https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-returncodes.html
- AWS CLI S3 configuration: https://docs.aws.amazon.com/cli/latest/topic/s3-config.html
- AWS SDK data integrity protections: https://docs.aws.amazon.com/sdkref/latest/guide/feature-dataintegrity.html
- AWS CLI v2 changelog, 2.23.0 entries on CRC64NVME and Content-MD5: https://github.com/aws/aws-cli/blob/v2/CHANGELOG.rst
- botocore `retries/standard.py` (checkers, backoff, quota): https://github.com/boto/botocore/blob/develop/botocore/retries/standard.py
- botocore `httpchecksum.py` (header versus trailer): https://github.com/boto/botocore/blob/develop/botocore/httpchecksum.py
- botocore `awsrequest.py` (`reset_stream`): https://github.com/boto/botocore/blob/develop/botocore/awsrequest.py
- s3transfer `utils.py`, `download.py`, `manager.py` (`S3_RETRYABLE_DOWNLOAD_ERRORS`, `num_download_attempts`): https://github.com/boto/s3transfer/tree/develop/s3transfer
- S3 `DeleteObjects` API: https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjects.html
- curl option docs: https://github.com/curl/curl/tree/master/docs/cmdline-opts (`retry.md`, `retry-all-errors.md`, `retry-connrefused.md`, `retry-delay.md`, `retry-max-time.md`, `max-time.md`, `fail.md`, `_EXITCODES.md`)
- rsync manual source: https://github.com/RsyncProject/rsync/blob/master/rsync.1.md
- rsync `errcode.h`: https://github.com/RsyncProject/rsync/blob/master/errcode.h
- rsync `receiver.c` (failed verification, redo): https://github.com/RsyncProject/rsync/blob/master/receiver.c
- rsync `clientserver.c` (`max connections` refusal): https://github.com/RsyncProject/rsync/blob/master/clientserver.c
- R2 S3 API compatibility (checksum types, conditional headers, `DeleteObjects`): https://developers.cloudflare.com/r2/api/s3/api/
- R2 limits: https://developers.cloudflare.com/r2/platform/limits/
- R2 consistency: https://developers.cloudflare.com/r2/reference/consistency/
- R2 durability: https://developers.cloudflare.com/r2/reference/durability/
- R2 changelog (CRC64NVME 2025-07-03, `DeleteObjects` 1000 keys): https://developers.cloudflare.com/r2/reference/changelog/
- R2 object lifecycles (7-day multipart expiry): https://developers.cloudflare.com/r2/buckets/object-lifecycles/
- R2 pricing (free operations): https://developers.cloudflare.com/r2/pricing/
- R2 with the AWS CLI: https://developers.cloudflare.com/r2/examples/aws/aws-cli/
- Cloudflare API limits: https://developers.cloudflare.com/fundamentals/api/reference/limits/
- Cloudflare purge cache limits: https://developers.cloudflare.com/cache/how-to/purge-cache/
- Cloudflare purge by single file: https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/
- Rulesets API, update a ruleset: https://developers.cloudflare.com/ruleset-engine/rulesets-api/update/
- Rulesets API overview (concurrent update limits): https://developers.cloudflare.com/ruleset-engine/rulesets-api/
- GitHub Actions limits: https://docs.github.com/en/actions/reference/limits
- GitHub-hosted runners: https://docs.github.com/en/actions/reference/runners/github-hosted-runners
- Events that trigger workflows (schedule): https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows
- Concurrency: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs
- Workflow syntax (`timeout-minutes`): https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions
- Expressions (`always()`, `cancelled()`): https://docs.github.com/en/actions/reference/evaluate-expressions-in-workflows-and-actions
- Notifications for workflow runs: https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/monitoring-workflows/notifications-for-workflow-runs
- healthchecks.io ping API: https://healthchecks.io/docs/http_api/
- healthchecks.io configuring checks: https://healthchecks.io/docs/configuring_checks/
- healthchecks.io signaling failures: https://healthchecks.io/docs/signaling_failures/
- TeX Live signature verification (key and expiry): https://www.tug.org/texlive/verify.html
- Becoming a CTAN mirror: https://ctan.org/mirrors/register
- mirmon manual: https://www.mankier.com/1/mirmon
- GNU `comm` man page (LC_COLLATE): https://manpages.debian.org/bookworm/coreutils/comm.1.en.html
- Local: `ctan-list-deref.txt`, `ctan-list-nolink.txt` (2026-08-26 listing); go-task 3.53.1, xz 5.8.3, curl 8.7.1 on this machine for the measurements quoted.
