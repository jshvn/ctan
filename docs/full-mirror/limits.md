# Limits

Every hard limit the full mirror runs against, re-verified on 2026-08-26 from its primary
source, with what the mirror asks of it per hourly run, per day, per month and during the
seed, and the margin left. Handling belongs to the sibling file named in each row; this
file only says where the walls are.

Conventions:

- "Stored set" is what `rsync -rL` yields minus tlnet's versioned containers and the six
  `update-tlmgr-r*` root files (the two excludes today's Taskfile already carries):
  **496,149 objects, 132.99 GB**, longest object 6,865,013,189 bytes (`MacTeX.pkg`).
  Re-derived from `SCRATCH/ctan-list-deref.txt` (538,289 lines, taken today, 6.908 s wall
  clock per `SCRATCH/rsync-time.txt`); commands in [Measurements](#measurements).
- "Hourly" is one run of the list-diff pipeline at steady state; "seed" is the first run
  with no state file, whose delta is the whole stored set.
- GB is 10^9 bytes; GiB and MiB are binary. R2's figures are binary; CTAN's are decimal.
- Every number is one of: verified (URL fetched today, wording quoted), computed (command
  given), or marked unverified with the one thing that would settle it.

## Usage model the limits are measured against

Computed from the listing (mtime as a proxy for when a file landed on CTAN):

| Quantity | Value | Command |
|---|---|---|
| Changed files, last 30 days | 16,574 files, 4.72 GB; 23.0 per hour on average | `awk` over `norm.sorted` (see Measurements) |
| Hour-slots with any change, last 30 days | 284 of 720 | `awk '$3>="2026/07/27"' hours.txt \| wc -l` |
| Busiest hour by count, last 365 days | 5,950 files, 0.037 GB (2026-01-12 16h) | `sort -rn hours.txt \| head -1` |
| Busiest hour by bytes, last 365 days | 20.35 GB in 9 files (2026-03-01 09h, ISO copies); 13.97 GB in 12 files (2026-03-24 11h, MacTeX) | `sort -k2 -rn hours.txt \| head -2` |
| Busiest hour by bytes excluding installers | under 0.001 GB | same, filtered on `systems/(texlive/Images\|mac/mactex)` |
| Hour-slots over 1,000 files, last 30 days | 3 | `awk '$3>="2026/07/27" && $1>1000' hours.txt` |
| Objects over 4.995 GiB (5,363,466,240 B) | 5: two `MacTeX` pkgs at 6.87 GB, three ISOs at 6.78 GB | `awk -F'\t' '$2>5363466240' norm.sorted` |
| Objects over 512 MiB (not cacheable) | 7: the five above plus two `protext` zips at 1.14 GB | `awk -F'\t' '$2>536870912'` |
| Objects over 200 MB (the current `multipart_threshold`) | 16, 39.18 GB | `awk -F'\t' '$2>200000000'` |
| Objects over 200 MB changed in the last 30 days | 0 | same with date filter |
| Seed batch plan at 4 GB per batch | 25 batches plus 5 lone files = 30 rsync connections; largest batch 97,569 files (fonts), smallest 33,003 | awk batch pass in Measurements |
| Purge calls at 100 URLs per call | seed 4,962; largest batch 976; hourly 1 (average), 60 on the busiest hour seen | `(n+99)/100` |
| Distinct directories | 24,953 | `awk -F/ '{NF--; print}' OFS=/ keys.txt \| sort -u \| wc -l` |

The base plan's "~34 batches" is 30 when packed from the actual listing; "~23 files an hour"
and "283 of 720 hour-slots" hold (284 today).

## 1. Cloudflare R2

Source: [R2 limits](https://developers.cloudflare.com/r2/platform/limits/) (page dated
2026-06-08), [S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/),
[Multipart upload](https://developers.cloudflare.com/r2/objects/multipart-objects/),
[Public buckets](https://developers.cloudflare.com/r2/buckets/public-buckets/),
[R2 pricing](https://developers.cloudflare.com/r2/pricing/).

| Limit | Value (quoted) | Hourly | Seed | Margin |
|---|---|---|---|---|
| Objects per bucket | "Number of objects per bucket: Unlimited" | 496k held | 496k | none needed |
| Storage per bucket | "Data storage per bucket: Unlimited"; free tier "10 GB-month / month", then $0.015/GB-month | 133 GB | 133 GB | billing, not a limit; cost-estimates.md |
| Object size | "Object size: 5 TiB per object" (footnote: "5 GiB less than 5 TiB, so 4.995 TiB") | 6.87 GB max | same | 700x |
| Single-part upload | "Maximum upload size: 5 GiB (single-part)"; footnote: "The max upload size is 5 MiB less than 5 GiB, so 4.995 GiB" = 5,363,466,240 B. Applies to "uploading a file via one request, uploading a part of a multipart upload, or copying into a part" | largest single-part object 1,138,914,783 B | 5 objects exceed it | 4.7x for the single-part set. The AWS CLI reads size suffixes as binary (`GB` = 1024^3), so `5GB` would be 5 GiB, above this cap; `multipart_threshold = 4GB` (4 GiB) stays under it, and only the 5 installers lie above 4 GiB |
| Multipart part size | "Minimum part size: 5 MiB (except for the last part)", "Maximum part size: 5 GiB", "All parts except the last must be the same size" | 0 | 5 objects | AWS CLI uses a fixed `multipart_chunksize`, so parts are uniform by construction |
| Multipart part count | "Maximum upload parts: 10,000" | 0 | at 8 MiB chunks: 819 parts per 6.87 GB file; at 512 MiB: 13 | 12x at 8 MiB, 770x at 512 MiB |
| Incomplete multipart uploads | "automatically aborted after 7 days by default" | 0 | a cut-off job leaves at most one | Class A already spent; no storage after 7 days; seeding-and-migration.md |
| Multipart ETag | "hash of the concatenated binary MD5 sums of all parts, followed by a hyphen and the number of parts" | n/a | 5 objects | nothing in the pipeline reads ETags; `aws s3 sync` compares size and mtime only |
| Key length | "Object key length: 1,024 bytes" | longest key 151 bytes | same | 6.8x |
| Metadata size | "Object metadata size: 8,192 bytes" | Content-Type only | same | 100x+ |
| Writes to one key | "Maximum concurrent writes to the same object name (key): 1 per second"; footnote "Concurrent writes to the same object name (key) at a higher rate return HTTP 429" | `timestamp` once; state file once per batch, minutes apart | same | the only in-run repeat of a key is the state file, one write per batch. Whether concurrent `UploadPart` calls on one key count is unstated: **unverified**; the first multipart upload shows it (429s in `--debug` output). errors-and-issues.md |
| Bucket-level request rate (S3 API) | none published. The only published rates are per key (above), "Maximum rate of bucket management operations per bucket: 50 per second" (bucket create/delete/configure, "does not apply to reading or writing objects"), and the r2.dev throttle ("hundreds of requests/second", 429), which a custom domain avoids | ~25 PutObject | ~500k PutObject at ~96/s | no wall to hit; the AWS CLI's `max_concurrent_requests` is the self-imposed one |
| Cloudflare REST API for R2 | "The Cloudflare REST API is rate limited to 1,200 requests per five minutes across all R2 REST API operations on your account" | 0 (the pipeline uses the S3 API, not the REST API) | 0 | n/a |
| ListObjectsV2 page | `max-keys` supported; S3 wire limit "up to 1,000" keys per page ([S3 API reference](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html)); R2 lists "in lexicographical order" (UTF-8 binary, the same as `LC_ALL=C sort`) | 0 | 0; daily reconcile 497 pages | Class A, 1M/month free |
| DeleteObjects batch | S3 wire limit "The request can contain a list of up to 1,000 keys"; R2 lists the operation as supported with no R2-specific cap; "Content-MD5 request header is required" (the CLI sends it) | ≤1 call | 0 | free operation |
| Custom domains per bucket | "Number of custom domains per bucket: 100" | 1 | 1 | 100x |
| Buckets per account | 1,000,000 | 1 | 1 | n/a |
| Custom-domain request or response size | none published for R2. The zone proxy's request-body cap is "100 MB" on Free ([error 413](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)), irrelevant because the pipeline never writes through `ctan.ijosh.com`; no response-size cap is published anywhere fetched today. Files over 512 MB are served uncached (section 2) | reads only | reads only | **unverified for a 6.78 GB GET**: `curl -r 0-0 -sI https://ctan.ijosh.com/systems/texlive/Images/texlive.iso` after the seed, then a full download once |
| Free operations | "DeleteObject", "AbortMultipartUpload" free per pricing page; egress "Free" | | | |
| Class A (PutObject, ListObjects, multipart create/part/complete) | "1 million requests / month" free, then $4.50/M | ~25 + 1 state write + purge-side nothing | ~500k | cost-estimates.md |
| Class B (GetObject, HeadObject) | "10 million requests / month" free, then $0.36/M | 1 state read + user traffic | 1 | cost-estimates.md |

### What a CTAN key can contain

R2 does not publish key character rules beyond the 1,024-byte length; S3's own guidance is
that "An object key can contain any Unicode character". The stored set today, computed on
`keys.txt` (one key per line; commands in Measurements):

| Class | Keys | Example | Consequence |
|---|---|---|---|
| Non-ASCII or control characters | 0 | | `LC_ALL=C` sorting and `comm` are byte-exact; no encoding question in the listing |
| Space | 25 | `support/tex-converter/update/TeX Converter.exe` | fine in `--files-from` (one path per line) and as S3 keys; must be `%20` in purge URLs and `smoke` URLs |
| `+` | 218 | `graphics/asymptote/fftw++.cc` | fine as a key; in a URL path `+` is literal, not a space, so no encoding needed, but do not run it through a form-encoder |
| `#` | 4 | `language/cyrtug/#disk.00` | must be `%23` in any URL or curl reads a fragment; 0 keys *start* with `#` or `;`, so none is a `--files-from` comment |
| `%` | 7 | `systems/mac/textures/fonts/AMS%2fPS VF (0.5, Uwe)` | the literal `%2f` must be sent as `%252f`; an unencoded URL resolves to key `AMS/PS VF...`, a 404 |
| `&` | 8 | `systems/mac/textures/latex/latex2e/babel&ECLaTeX.sea.hqx` | fine in a path; encode in purge JSON only if a URL-parser would split it (it is JSON, so no) |
| `?`, `\`, `'`, `"`, backtick, `*`, `;`, `\|`, `<`, `^` | 0 each | | no JSON escaping needed in purge bodies (`"` and `\` absent) |
| `~` 12, `$` 14, `!` 7, `,` 196, `:` 20, `@` 18, `=` 154, `(` `)` 28, `[` `]` `{` `}` `>` 1 each | | `systems/mac/textures/fonts/AMS.vf.metrics(Art Ogawa)` | fine as keys and in curl URLs; quote every path in shell |
| Leading-dash component | 4 | `support/pcwritex/-README.1ST` | never appears as an argv element on its own (keys are always prefixed by a directory or `s3://tlnet/`); `--files-from` reads lines, not argv |
| Trailing-dot name | 2 | `support/qfig/qfig3ple.` | fine on Linux and R2; would be a problem only on Windows |
| Hidden (dot-leading) component | 37 | `documentation/beginlatex/src/.run` | `find . -type f` includes them; `rsync` includes them; nothing filters dotfiles |
| Leading or trailing whitespace, `..` or blank components, `//` | 0 | | rsync would resolve `..` away and reject residuals; none exist |
| Longest key | 151 bytes | `macros/latex2e/contrib/ualberta/03_References/Reference_PDFs/Aldrich2017-...Strain.pdf` | 1,024 limit |
| Longest component / deepest path | 90 bytes / 15 levels | | ext4 name limit 255 bytes |
| Keys over 200 bytes | 0 | | |
| Mean key length | 49.8 bytes | | a 496k-line key list is 25 MB; the `path size mtime` state file is 37.7 MB, 2.97 MB as `xz -6` |

Upshot: every key is printable ASCII, so the S3 API, `--files-from`, `comm` and the
purge JSON all work byte-for-byte. The 36 keys with a space, `#` or `%` are the ones a
URL-building step must percent-encode; that belongs to caching.md (purge URLs) and
verification-and-security.md (`smoke` sample URLs).

### S3 API gaps that touch the AWS CLI

From the compatibility page, checked operation by operation:

- `PutObject`: "Conditional Operations: If-Match, If-Modified-Since, If-None-Match,
  If-Unmodified-Since" all supported. The `aws s3` high-level commands do not send them;
  `aws s3api put-object --if-none-match '*'` would. Not needed: every write is idempotent.
- `ListObjectsV2`: `list-type`, `continuation-token`, `delimiter`, `encoding-type`,
  `fetch-owner`, `max-keys`, `prefix`, `start-after` supported. `aws s3 ls --recursive`
  paginates at 1,000 and works today (the `stale` task).
- `DeleteObjects`: supported; unsupported extras are `x-amz-mfa`,
  `x-amz-bypass-governance-retention`, `x-amz-request-payer`,
  `x-amz-expected-bucket-owner`, none of which the CLI sends. `aws s3api delete-objects
  --delete file://...` with ≤1,000 keys per call is the batch path.
- `cp --recursive`: plain `PutObject` per file (one generator, no destination listing, per
  the base plan's reading of `subcommands.py`); nothing R2 lacks.
- Checksums: `Content-MD5` and `x-amz-checksum-*` are marked unsupported on several
  operations. AWS CLI 2.36 (the runner's version) defaults to CRC-based flexible
  checksums on upload; R2 has accepted `aws s3 sync` uploads from this CLI line every day,
  so the current default is compatible. If a CLI bump ever produces
  `NotImplemented` on `PutObject`, `request_checksum_calculation = when_required` in
  `aws.config` is the switch. **Unverified** for 2.36.24 specifically; the daily tlnet run
  after the next Dependabot-free image bump is the check.
- Object tagging is unimplemented; the pipeline never tags.
- Unicode metadata needs RFC 2047 encoding (extensions page); metadata is ASCII here.

## 2. Cloudflare zone (Free plan)

Sources: [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/),
[Cache Rules settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/),
[Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/),
[Tiered Cache](https://developers.cloudflare.com/cache/how-to/tiered-cache/),
[Purge cache](https://developers.cloudflare.com/cache/how-to/purge-cache/),
[Purge by single-file](https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/),
[Transform Rules](https://developers.cloudflare.com/rules/transform/),
[API limits](https://developers.cloudflare.com/fundamentals/api/reference/limits/),
[GraphQL limits](https://developers.cloudflare.com/analytics/graphql-api/limits/),
[Rulesets API](https://developers.cloudflare.com/ruleset-engine/rulesets-api/),
[Error 524](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/),
[Enable cache in an R2 bucket](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/),
[Application Services terms](https://www.cloudflare.com/service-specific-terms-application-services/),
[Developer Platform terms](https://www.cloudflare.com/service-specific-terms-developer-platform/).

| Limit | Value (quoted) | Hourly | Seed | Margin |
|---|---|---|---|---|
| Cache Rules | "Number of rules: 10" on Free (25 Pro, 50 Business, 300 Enterprise) | 1 or 2 rules, re-applied as one `PUT` | same | 5x |
| Transform Rules | "Active Transform Rules: 10" on Free; "Regex support: No" on Free and Pro | 1 (`/` to `/index.html`), or 0 if CTAN's own `index.html` becomes the front page | same | the rule must be a plain path match, not a regex |
| Cacheable object size | "Free, Pro and Business customers have a limit of 512 MB. For Enterprise customers the default maximum cacheable file size is 5 GB" | 7 objects exceed it, served from R2 on every request | same | those 7 are 34.6 GB of the 133; each uncached GET of one is one Class B, egress free |
| Default cached extensions | "Cloudflare only caches based on file extension and not by MIME type"; the list includes `ZIP GZ TAR ISO PDF DMG EXE BZ2 7Z ZST` and not `xz`, `lzma`, `pkg`, `tfm`, `vf`, `tex`, `sty`, `dtx` | a cache rule must mark the mirror "Eligible for cache" for the 31k `.xz`, 35k `.lzma`, 110k `.tfm` | | caching.md |
| Default Edge TTL by status | 200/206/301: 120 minutes; 302/303: 20 minutes; 404/410: 3 minutes | | | a 404 cached before a key exists lives 3 minutes unless the rule's TTL applies to 404s too; answers the base plan's open question on purging added keys, caching.md |
| Edge TTL override bounds | the settings page gives no numeric minimum or maximum; Browser TTL "values available depend on your plan" | | | **unverified**: `PUT` the rule with the intended TTL and read the API error, if any |
| Smart Tiered Cache | "Yes" on Free, Pro, Business, Enterprise; "the single closest upper tier for each of your website's origins" | 1 `POST` per run, idempotent | same | |
| Purge by URL | Free: "800 URLs per second", "Max operations per request: 100", "applied per account", "thresholds for URLs are calculated using a moving average" | 0 with `CACHE: off` (the default); 1 call, ~25 URLs when on | 0 with `CACHE: off`; 4,962 calls, 496k URLs when on; at 3.3 calls/s that is 330 URLs/s | 2.4x under 800/s; a batch of 97,569 keys is 976 calls, ~5 minutes at the self-imposed pace |
| Purge everything, hostname, tag, prefix | Free: "5 requests per minute" (Pro 5/s, Business 10/s, Enterprise 50/s); "Max operations per request: 100"; token bucket | 0 | 0 | never used; caching.md explains why |
| Purge URL rules | "Always use UTF-8 encoded URLs"; "the host part of the URL is not case-sensitive ... the path portion is case-sensitive"; "Wildcards are not supported on single file purge" | | | the 36 keys with space, `#`, `%` need percent-encoding; case is preserved |
| Global API rate | "Client API per user/account token: 1200/5 minutes"; "Client API per IP: 200/second"; "all API calls for the next five minutes will be blocked, receiving a HTTP 429"; `retry-after` header "The number of seconds, rounded up, until more capacity is available"; "Some specific API calls have their own limits and are documented separately, such as ... Cache Purge APIs, GraphQL APIs, Rulesets APIs" | 3 rule calls + 1 or 2 analytics, + 1 purge when `CACHE: on` | 3, + 4,962 purges when `CACHE: on` | whether purge calls also count against the 1,200 is not stated. Counted conservatively: 3.3 calls/s is 990 per 5 minutes, under 1,200 with 21% to spare; at 3 calls/s exactly it is 900 |
| GraphQL Analytics | "300 GraphQL queries over 5-minute window" per user (API page: "Max 320/5 min"); zone queries "up to 10 zones"; retention and record caps per node from the settings query | 1 or 2 | 1 or 2 | 150x; monitoring.md |
| Rulesets API | "You should avoid making concurrent updates to the same ruleset. There are rate limits in place to prevent the same ruleset from being concurrently updated too many times" (numbers unpublished); "update the entire ruleset in a single operation" | 2 sequential `PUT`s of whole phases | same | one process, never concurrent |
| Origin read timeout (524) | "the origin did not provide an HTTP response before the default 125 seconds Proxy Read Timeout"; "Enterprise customers can increase the 524 timeout up to 6,000 seconds"; write side "30 seconds Proxy Write Timeout ... cannot be adjusted" | R2 answers in ms | | applies to time to first byte, not transfer length; a 6.78 GB stream is not cut at 125 s. The base plan's "100-second" figure is the old default; no 500-second figure exists on any page fetched today |
| Free-plan bandwidth or request cap | none published. Terms: Cloudflare "reserves the right to disable or limit your access to or use of the CDN ... if you use or are suspected of using the CDN without such Paid Services to serve video or a disproportionate percentage of pictures, audio files, or other large files"; Developer Platform terms list "R2" in the Developer Platform and say Cloudflare "may temporarily limit your storage and/or the number of requests" under undue burden | | | whether an R2 bucket on the free tier is a "Paid Service" for this clause is a reading, not a fact; official-mirror-and-url.md and cost-estimates.md own the argument |
| Health Checks, usage notifications | 0 on Free / Pro and above (base plan, not re-fetched) | | | monitoring.md |

## 3. GitHub Actions

Sources: [Limits](https://docs.github.com/en/actions/reference/limits),
[GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners),
[Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows),
[Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax),
[Concurrency](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs),
[Workflow commands](https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions),
[Ubuntu 24.04 image](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md).

| Limit | Value (quoted) | Hourly | Seed | Margin |
|---|---|---|---|---|
| Job execution time | "Each job in a workflow can run for up to 6 hours of execution time. If a job reaches this limit, the job is terminated and fails" | 30 to 40 s quiet, ~40 min on a release day | 4 to 6 h estimated, so 1 to 3 runs | the seed is designed to be cut and resumed; seeding-and-migration.md |
| `timeout-minutes` | "The maximum number of minutes to let a job run before GitHub automatically cancels it. Default: 360. If the timeout exceeds the job execution time limit for the runner, the job will be canceled when the execution time limit is met instead" | 55 | 360 | values above 360 do nothing on hosted runners |
| Workflow run time | "35 days / workflow run" | one job | one job | n/a |
| Concurrent jobs | Free plan "Total concurrent jobs: 20" (5 macOS) | 1 | 1 | `concurrency: sync` holds it to 1 by design |
| Concurrency queue | "single (default): At most one job or workflow run can be pending in the concurrency group. When a new job or workflow run is queued, any existing pending job or workflow run in the same group is canceled and replaced"; "max: Up to 100 jobs or workflow runs can be pending ... once the queue is full, any additional runs are canceled"; "queue: max cannot be combined with cancel-in-progress: true" | 0 pending | during a 6-hour seed the cron fires 5 or 6 times: each new run cancels the pending one, so at most one waits, and it starts the moment the seed job ends | the right setting is the default. Each cancelled run is a run that would have recomputed the same delta; nothing is lost. The cancelled runs show as "cancelled" in the Actions list. Whether GitHub emails on cancelled scheduled runs is **unverified** (notification settings, not docs); monitoring.md |
| Cron minimum and drift | "The shortest interval you can run scheduled workflows is once every 5 minutes"; "The schedule event can be delayed during periods of high loads of GitHub Actions workflow runs. High load times include the start of every hour. If the load is sufficiently high enough, some queued jobs may be dropped" | 1 per hour | | a fixed random minute away from :00 serves GitHub as well as CTAN; a dropped run is a two-hour gap, which is why the healthchecks grace must exceed one period; monitoring.md |
| 60-day disablement | "In a public repository, scheduled workflows are automatically disabled when no repository activity has occurred in 60 days" | | | Dependabot's weekly bump is the activity; healthchecks catches the failure mode |
| Default branch | "Scheduled workflows run on the latest commit on the default branch" | | | a PR cannot change the schedule until merged |
| Runner spec (public repo, Linux x64) | "Linux 4 16 GB 14 GB x64" (CPU, RAM, SSD); `ubuntu-latest` = Ubuntu 24.04 ("OS Version: 24.04.4 LTS", "Image Version: 20260816.277.1") | | | |
| Actual free disk | not documented by GitHub. Third-party reports for ubuntu-24.04 put the root filesystem at ~72 GB with ~18 to 22 GB free before any step ([free-disk-space](https://github.com/thiagokokada/free-disk-space), [runner-images #13189](https://github.com/actions/runner-images/issues/13189)) | peak 4 GB batch + ~130 MB of listings | one 6.87 GB installer alone | plan against the documented 14 GB: a 4 GB batch leaves 10 GB, an installer batch 7 GB. **Unverified** on this repo's runner: add `df -h /` as a step and read it once |
| Runner hardware | GitHub's 2024 announcement moved public-repo Linux runners to 4-vCPU Azure Dadsv5 VMs ([blog](https://github.blog/news-insights/product-news/github-hosted-runners-double-the-power-for-open-source/), via search); `Standard_D4ads_v5` is 4 vCPU, 16 GiB, 150 GiB temp disk at 250 MBps sequential read, "Max Network Bandwidth (Mbps): 12,500" ([Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dadsv5-series)) | | | the VM is not the network bottleneck (section 7); which SKU a given job lands on is **unverified** (`cat /proc/cpuinfo`, `lscpu` in a step) |
| Job summary | "each step is restricted to a maximum size of 1MiB"; "A maximum of 20 job summaries from steps are displayed per job"; "If more than 1MiB of content is added for a step, then the upload for the step will fail and an error annotation will be created"; "Upload failures for job summaries do not affect the overall status" | ~2 KB | ~2 KB if `report` prints counts, 1.5 MB if it printed a batch's upload list | `report` prints counts and the first 20 lines of any list, never a list |
| Log line drops | not documented. Observed in this repo (CLAUDE.md: thousands of `upload:` lines in seconds go missing); a community thread reports streaming logs capped near 4 MB until the job ends ([discussion 127903](https://github.com/orgs/community/discussions/127903)) | ~50 lines | ~500k lines if uploads were echoed | counts come from `RUN/`, never the log; `--no-progress` stays; consider `--only-show-errors` for the seed |
| Tool versions on the image | rsync 3.2.7 (protocol 31), AWS CLI 2.36.24, xz 5.4.5 (package `5.6.1+really5.4.5`), curl 8.5.0, gnupg 2.4.4, coreutils 9.4, bash 5.2.21, jq 1.7.1 | | | rsync 3.5.0 (protocol 32) is current upstream; nothing here needs it |
| Workflow file size | "A workflow file larger than 500 KB will not start runs" | 1 KB | | |
| GITHUB_TOKEN API rate | "1,000 requests per hour per repository" | 0 | 0 | the pipeline calls no GitHub API |
| Artifacts and cache | 500 MB artifact storage on Free; cache "10 GB" per repository | 0 | 0 | not used; the state file lives in the bucket |

## 4. dante and CTAN

Sources: [Becoming a CTAN mirror](https://ctan.org/mirrors/register),
[mirmon status](https://ctan.org/mirrors/mirmon), [rsyncd.conf(5)](https://download.samba.org/pub/rsync/rsyncd.conf.5),
`https://mirror.ctan.org/timestamp` (fetched), the two listings.

| Limit | Value | Hourly | Seed | Margin |
|---|---|---|---|---|
| rsync daemon `max connections` on `rsync.dante.ctan.org` | **unverified**: no CTAN page states it; rsyncd.conf(5): "The default is 0, which means no limit", refused clients "receive a message telling them to try later" (`@ERROR: max connections (N) reached -- try again later`, client exits 5) | 2 connections, sequential | 31 connections, sequential | one connection at a time from this mirror. What would settle it: the daemon's MOTD on connect, or asking on the mirror maintainers' list |
| Listing size and time | `rsync -rL --list-only`: 538,289 lines, 50.7 MB as text, 6.908 s wall clock today (`SCRATCH/rsync-time.txt`); `rsync -r --list-only`: 395,562 lines, 37.0 MB. Bytes on the wire are the compressed file list, not the text (base plan: ~19 MB, not re-measured) | 1 | 1 | the base plan's 6.6 s was a different run; 6.9 s today |
| Register page requirements | "at least 60 GB of hard drive space free (100 GB leaves room to grow)"; "must synchronize with the primary CTAN node at least once per hour"; "Choose a random minute between 0 and 59" and "retain that minute permanently"; "You must mirror from the primary CTAN node" `rsync://rsync.dante.ctan.org/CTAN`; "we monitor mirrors to check that they are up to date. If your mirror falls behind then mirrors.ctan.org will not redirect to it, and we shall have to remove it from the official list"; HTTPS supported by the redirector since April 2021 | | | the 60 GB is about a disk the mirror does not have; the bucket holds 133 GB. official-mirror-and-url.md |
| Apache `+Indexes` | appears in the page's example `Options` line, not as a requirement | | | directory URLs 404 on R2; official-mirror-and-url.md |
| mirmon freshness | legend: "0 ≤ age ≤ 28 hours" fresh, "28h < age ≤ 52h" oldish; today "132 sites in 40 regions", "13 older than 2.2 days", "2 unreachable for more than 5 hours", "mean mirror age is 32 hours, std_dev 3.4 days, median 3 hours" | | | an hourly mirror sits at the median; a 24-hour outage is still "fresh" |
| What mirmon reads | `/timestamp`; the file served by `mirror.ctan.org` begins "# This file is for administrative purposes only. The source CTAN of this site's material: irony.dante.de" | copied every run, uncached | | the key must never be cached; caching.md |
| `timestamp` cadence | listing shows `timestamp` at `2026/08/26 17:02:01`, 186 bytes; `FILES.byname` and `FILES.last07days` at 16:20, `CTAN.sites` at 2026-08-25 16:21 | | | one observation today is consistent with the base plan's "every hour at :02"; two listings an hour apart would verify the cadence |
| Blank-size entries | 525 lines in the dereferenced listing (479 in the symlink listing) print no size at all, e.g. `fonts/cm/utilityfonts/null.mf`; 395 are under `obsolete/systems`. `null.mf` is a known empty file, so these are zero-byte files that openrsync prints with an empty size column | | | a listing parser must accept an empty size as 0 (the awk and sed in Measurements do). GNU rsync on the runner prints `0` and, being ≥3.1.0, prints thousands separators ("The default is human-readable level 1", `--list-only` included): `gsub(",","",$2)` stays in the parser; sync-with-dante.md |

## 5. Tools

### rsync

Source: [rsync(1) for 3.5.0](https://download.samba.org/pub/rsync/rsync.1),
[errcode.h](https://raw.githubusercontent.com/RsyncProject/rsync/master/errcode.h),
[rsync.h](https://raw.githubusercontent.com/RsyncProject/rsync/master/rsync.h).

- `--files-from=FILE`: "The --relative (-R) option is implied"; "The --dirs (-d) option is
  implied"; "The --archive (-a) option's behavior does not imply --recursive (-r)";
  "any leading slash is removed, and '..' components are resolved away"; "Blank entries
  are ignored, as are whole-entry comments that start with ';' or '#'". No size or entry
  limit is documented. The largest seed batch's list is 97,569 lines, 4.57 MB; rsync's
  file-list memory is on the order of 100 bytes per entry, ~10 MB, against
  `--max-alloc` "about 1GB" per allocation. 0 keys start with `#` or `;` and 0 have
  leading or trailing whitespace, so no entry is misread as a comment or trimmed.
- `-L`/`--copy-links` with `--files-from`: "The sender transforms each symlink encountered
  in the transfer into the referent item, following the symlink chain to the file or
  directory that it references. If a symlink chain is broken, an error is output and the
  file is dropped from the transfer" (exit 23). Because `--relative` is implied, a listed
  path like `documentation/ling-mac.tex` whose first component is a directory symlink is
  created at the alias path, which is what the bucket needs.
- `MAXPATHLEN 1024` in `rsync.h`; the longest key is 151 bytes.
- Protocol: `PROTOCOL_VERSION 32` in 3.5.0; Ubuntu 24.04's 3.2.7 speaks 31; macOS's
  openrsync speaks 29 ("rsync version 2.6.9 compatible"). "The client and server
  automatically negotiate the newest protocol they both support". Dante's daemon version is
  **unverified** (it accepted protocol 29 today, so it is at least that permissive).
- `--timeout=SECONDS`: "If no data is transferred for the specified time then rsync will
  exit. The default is 0, which means no timeout". `--contimeout`: "the amount of time that
  rsync will wait for its connection to an rsync daemon to succeed".
- Exit values (man page and `errcode.h`): 0 ok; 1 syntax; 2 protocol incompatibility;
  3 file selection; 4 unsupported action; **5** "Error starting client-server protocol"
  (`RERR_STARTCLIENT`, the refused-handshake case); **10** socket I/O; 11 file I/O;
  **12** protocol data stream; 13 diagnostics; 14 IPC; 15/16 sibling crashed/killed;
  19/20 signals; 21 `waitpid()`; 22 "Error allocating core memory buffers"; **23** "Partial
  transfer due to error"; **24** "Partial transfer due to vanished source files"; 25
  `--max-delete`; **30** "Timeout in data send/receive"; **35** "Timeout waiting for daemon
  connection". The base plan's retry set {5, 10, 12, 30, 35} matches; 22 (memory) is a
  hard failure, correctly outside the set.

### AWS CLI v2

Source: [S3 configuration](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html),
[Retries](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-retries.html),
[Command line options](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-options.html),
[s3 cp](https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html).

| Setting | Documented | Design value | Note |
|---|---|---|---|
| `max_concurrent_requests` | "Default - 10"; no maximum stated | 32 (as today) | the one place the pipeline pushes a remote |
| `max_queue_size` | "Default - 1000"; "the task queue size is capped ... a larger max queue size will require more memory" | default | bounds memory: the CLI holds at most 1,000 pending tasks whatever the tree size. The largest batch is 97,569 files; **unverified** peak RSS: run the first seed batch under `/usr/bin/time -v` (GNU time is on the image). No aws-cli issue found today reporting unbounded memory for local-to-S3 `cp --recursive`; `sync` is the command with two generators and a comparator |
| `multipart_threshold` | "Default - 8MB"; "S3 imposes constraints on valid values" | `4GB` = 4 GiB (4,294,967,296 B; the CLI's suffixes are binary, so `5GB` would be 5 GiB, above the 4.995 GiB single-part cap) | keeps every object up to `protext.zip` (1.14 GB) a single `PutObject`; only the 5 installers are above 4 GiB. The current `200MB` would multipart 16 objects: 4,676 parts at 8 MiB |
| `multipart_chunksize` | "Default - 8MB", "Minimum For Uploads - 5MB", auto-adjusted to a valid value | `512MB` = 512 MiB | the five installers become 13 parts each (6,865,013,189 / 536,870,912 = 12.8): 5 × (1 create + 13 parts + 1 complete) = 75 Class A for the seed, against ~4.1k at 8 MiB. One `aws.config`, no per-file override |
| `max_bandwidth` | "Default - None" | unset | no reason to self-throttle bytes |
| `retry_mode` / `max_attempts` | v2 default "standard": "A default value of 2 for maximum retry attempts, making a total of 3 call attempts"; retries `RequestTimeout`, `ConnectionError`, `HTTPClientError`, `Throttling`, `TooManyRequestsException`, `SlowDown`, `RequestLimitExceeded`, ..., and HTTP 500, 502, 503, 504; "exponential backoff by a base factor of 2 for a maximum backoff time of 20 seconds". "Adaptive mode is an experimental mode and is subject to change" and adds "client-side rate limiting through the use of a token bucket" | `standard`, `max_attempts = 10` | 429 from R2 is `TooManyRequests`/`SlowDown`, on the list. Worst case per call: 9 waits capped at 20 s, ~3 minutes |
| `cli_connect_timeout` | "maximum socket connect time in seconds ... The default value is 60 seconds"; 0 blocks forever | 60 | |
| `cli_read_timeout` | "maximum socket read time in seconds ... The default value is 60 seconds" | 300 | a socket read timeout, per read, not per object: a 6.87 GB upload that keeps moving never trips it; only R2 taking more than 300 s to answer after the last byte of a part would |
| `cp --expected-size` | needed "only when a stream is being uploaded to s3 and the size is larger than 50GB" | unused | files are on disk, sizes known |
| Content-Type | "the mime type of a file is guessed when it is uploaded" | default | `.xz` maps to `application/x-xz` (observed on `latex.tar.xz` today) |
| Listing pagination | `--page-size` per call, 1,000 max on the wire | default | 497 pages for the reconcile |

### curl (8.5.0 on the runner, 8.7.1 locally)

`curl --manual` (local, 8.7.1): "Transient error means either: a timeout, an FTP 4xx
response code or an HTTP 408, 429, 500, 502, 503 or 504 response code"; "it first waits one
second and then for all forthcoming retries it doubles the waiting time until it reaches 10
minutes"; `--retry-max-time` "The retry timer is reset before the first transfer attempt.
Retries are done as usual ... as long as the timer has not reached this given limit". 401
and 403 are not transient, which is what the pipeline wants. `-d @file` has no documented
size limit; a 100-URL purge body is ~8 KB. Every flag the base plan's `CURL` variable uses
(`--retry-connrefused`, `--retry-max-time`, `--fail-with-body`) exists in both versions.

### comm, sort, awk, split, xz, shasum, gpgv

Measured on this Mac (Apple M-series, 14 cores, 48 GiB) on the real listing; the runner
has 4 vCPU and is slower per core, so take 2 to 3x for a ceiling. Commands in
[Measurements](#measurements).

| Step | Input | Time | Peak RSS |
|---|---|---|---|
| Normalise listing to `path<TAB>size<TAB>mtime` (grep + sed) | 538,289 lines | 0.84 s | small |
| `LC_ALL=C sort` of the normalised file | 496,149 lines, 37.7 MB | 0.11 s | 68 MB |
| `LC_ALL=C sort` of the raw listing | 50.7 MB | 0.31 s | 88 MB |
| `comm -13 applied upstream` | 2 × 496k lines | 0.10 s | 2 MB |
| `comm -23` on path columns | 2 × 496k lines | 0.09 s | 2 MB |
| awk batch plan (cumulative size, 4 GB cuts, lone files) | 496k lines | 1.26 s | small |
| awk reorder (`tlpkg/` last) | 496k lines | 0.72 s | small |
| awk sum of bytes over the raw listing with `gsub(",")` | 538k lines | 0.82 s | small |
| `sort -t'\t' -k2,2nr` (by size) | 496k lines | 0.31 s | small |
| `split -l 100` (purge chunks) | 496k lines | 0.15 s, 4,962 files (`aa`..`zz` exhausted: use `-a 3`) | |
| `xz -T1 -6` of the state file | 37.7 MB → 2.97 MB | 5.7 s | 100 MB |
| `xz -T1 -9` | → 2.92 MB | 7.0 s | 421 MB |
| `xz -T0 -6` | → 2.97 MB | 3.8 s wall | 240 MB |
| `xz -dc` of the state file | 2.97 MB → 37.7 MB | 0.13 s | |
| `shasum -a 512` (Perl, Digest::SHA) | 2 GiB | 3.3 s = ~650 MB/s | |
| `openssl sha512` | 2 GiB | 1.4 s = ~1.5 GB/s | not in the tool list; noted only as a bound |
| `shasum -a 512` over 200 × 1 MB files via `find -exec ... +` | 200 MB | 0.36 s | per-file overhead is negligible when batched |
| tlpdb → checksum list awk (today's `verify`) | 20.6 MB, 14,872 containers | 0.69 s | |

So a 4 GB batch hashes in ~7 s here, perhaps 20 s on the runner; `comm` on the whole tree
is instant and needs no memory; `sort` of the state needs under 100 MB; `xz -6` on one
thread is the slowest text step at ~6 s and should stay at `-6` (`-9` triples memory for
1.7% smaller output). `gpgv` runs on 144-byte files; 2.4.4 on the image, 2.5.21 locally.

## 6. healthchecks.io

Source: [Pinging API](https://healthchecks.io/docs/http_api/), [Pricing](https://healthchecks.io/pricing/).

| Limit | Value (quoted) | Hourly | Seed | Margin |
|---|---|---|---|---|
| Pings per check | "Please do not ping a single check more than 5 times per minute"; "If you ping a check more than 5 times per minute, some of the requests may get rate limited and not recorded" (response "200 OK (rate limited)") | 1 | 1 per run | 5x per minute; the seed pings once per job, not per batch |
| Body | "the first 100 kB for each received ping" kept; `Ping-Body-Limit: 100000` header | one `timestamp` line if the body is used | | 500x |
| Checks and log | Hobbyist "$0 / month", "Monitor 20 jobs", "100 log entries per job" | 1 or 2 checks | | 100 entries is ~4 days of hourly pings, the visible age history; monitoring.md |
| Methods | "HEAD, GET, and POST" | GET | | |
| Period / grace | "Period is the expected time between pings. Grace Time is the additional time to wait before sending an alert"; a 6-hour seed job with one ping at the end needs grace ≥ 6 h or an expected alert; monitoring.md | | | |

## 7. Network

| Quantity | Value | Source | Consequence |
|---|---|---|---|
| rsync from dante, observed | 6.79 GB in 299 s = 22.7 MB/s, ~400 KB average file, one TCP stream | base plan, job 32933376123 | not re-measured (no rsync to dante allowed today) |
| Runner NIC | "12,500" Mbps for `Standard_D4ads_v5` = 1.56 GB/s | Azure page | 69x the observed rsync rate: the bound is the single transatlantic rsync stream (latency, per-file round trips, dante's side), not the runner. **Unverified** directly; a `curl -o /dev/null -w '%{speed_download}' https://speed.cloudflare.com/__down?bytes=1000000000` step would show the runner's own ceiling |
| Runner local disk | 250 MBps sequential read (D4ads_v5 temp disk) | Azure page | not a bound at 22.7 MB/s in or ~100 PutObject/s out |
| R2 upload per connection | third-party benchmark: 25.6 MB/s at 1 GB single `PutObject`, 24 MiB/s aggregate at 256 KiB objects ([Tigris](https://www.tigrisdata.com/docs/overview/benchmarks/cloudflare-r2/)) | vendor benchmark, unverified here | with 32 concurrent streams the byte rate is not the seed's bound; the per-object latency (~170 ms in the same benchmark) is, which matches ~96 PutObject/s at 32-way (32 / 0.17 s ≈ 190/s ideal) |
| The 6.87 GB installer within `cli_read_timeout` | 13 parts of 512 MiB, up to 13 in flight; at 25 MB/s per stream each part sends in ~21 s; the 300 s read timeout only counts time waiting for R2's reply | computed | fits with 14x margin per part; the whole object takes ~1 to 5 minutes |
| Listing bytes | 50.7 MB text per hour = 36 GB/month of runner ingress from dante's file list stream (compressed on the wire) | listing | free on both sides; it is what every mirror's `rsync -a` does |

## 8. Consolidated table

Usage columns: H = hourly run, S = seed (first run, whole tree). "At the limit" is what the
remote or tool does, not what the pipeline does; the last column is the file that owns
the handling.

| Limit | Value | Source | H | S | Margin | At the limit | Owner |
|---|---|---|---|---|---|---|---|
| R2 objects / storage | unlimited / 10 GB-month free | R2 limits, pricing | 496k, 133 GB | same | billing only | bill | cost-estimates.md |
| R2 single-part upload | 4.995 GiB | R2 limits fn.4 | 1.14 GB max | 5 objects over | 4.7x; 5 multipart | `EntityTooLarge`, job fails | seeding-and-migration.md |
| R2 multipart parts / part size | 10,000 / 5 MiB–5 GiB, uniform | R2 multipart | 0 | 13 per object | 770x | reject | taskfile-architecture.md (`aws.config`) |
| R2 incomplete multipart | aborted after 7 days | R2 multipart | 0 | ≤1 | | storage held ≤7 days | seeding-and-migration.md |
| R2 key length | 1,024 bytes | R2 limits | 151 max | same | 6.8x | reject | sync-with-dante.md |
| R2 writes per key | 1/s | R2 limits fn.5 | 1 per key | state file 1/batch | minutes apart | 429 | errors-and-issues.md |
| R2 concurrent parts on one key | unstated | | 0 | 13 parts, up to 13 in flight | unknown | 429, retried (standard mode) | errors-and-issues.md |
| R2 bucket request rate | none published | R2 limits | 25 PUT | ~500k PUT | | 429 / `SlowDown`, retried | errors-and-issues.md |
| ListObjectsV2 page | 1,000 keys | S3 API | 0 | 0; 497/day | | | sync-with-dante.md |
| DeleteObjects batch | 1,000 keys | S3 API | ≤1 call | 0 | | `MalformedXML` | sync-with-dante.md |
| R2 Class A | 1M/month free | pricing | ~25 + 1 | ~500k + 75 | 2x in the seed month | bill at $4.50/M | cost-estimates.md |
| R2 Class B | 10M/month free | pricing | 1 + users | 1 | 12x at 100 installs/day uncached | bill at $0.36/M | cost-estimates.md, monitoring.md |
| Custom domains per bucket | 100 | R2 limits | 1 | 1 | 100x | | official-mirror-and-url.md |
| Cache Rules | 10 | Cache Rules | 1–2 | same | 5x | API rejects the PUT | caching.md |
| Transform Rules | 10, no regex on Free | Transform Rules | ≤1 | same | 10x | | caching.md |
| Cacheable object size | 512 MB | Default cache behavior | 7 objects over | | | served from R2 every time (Class B) | caching.md |
| Edge TTL bounds | unstated | Cache Rules settings | | | unverified | API error on PUT | caching.md |
| Purge by URL | 800 URLs/s, 100/call, per account, moving average | Purge cache | 0 with `CACHE: off` (default); 25 URLs, 1 call when on | 0 with `CACHE: off`; 496k URLs, 4,962 calls at ~330 URLs/s when on | 2.4x | 429 with `retry-after`; curl waits | caching.md, errors-and-issues.md |
| Purge everything | 5/min | Purge cache | 0 | 0 | unused | 429 | caching.md |
| Cloudflare API per token | 1,200 / 5 min | API limits | 3 to 6 | ≤990 per 5 min if purges count and `CACHE: on` | 21% | 429, blocked 5 min, `retry-after` | errors-and-issues.md |
| Cloudflare API per IP | 200/s | API limits | <1/s | 3.3/s | 60x | 429 | errors-and-issues.md |
| GraphQL | 300 / 5 min | GraphQL limits | 1–2 | 1–2 | 150x | 429 | monitoring.md |
| Rulesets concurrency | unpublished, "avoid concurrent updates" | Rulesets API | 2 sequential PUTs | same | one process | 429 | caching.md |
| Origin read timeout | 125 s to first byte (write 30 s) | Error 524 | ms | ms | | 524 to the client | n/a (R2 is the origin) |
| Zone upload cap | 100 MB on Free | Error 413 | 0 (S3 endpoint used) | 0 | n/a | 413 | n/a |
| Free-plan bandwidth | none published; CDN terms clause on large files, Developer Platform exception | terms | user traffic | | a reading, not a number | account action | official-mirror-and-url.md |
| GitHub job time | 6 h | Limits | ≤40 min | 4–6 h, resumable | thin by design | job killed; state as of last batch | seeding-and-migration.md |
| `timeout-minutes` | default 360, capped at 6 h | Workflow syntax | 55 | 360 | | job cancelled | taskfile-architecture.md |
| Concurrency pending queue | 1 pending (default), newest replaces | Concurrency | 0 | 1 | | older pending run cancelled | seeding-and-migration.md |
| Concurrent jobs | 20 on Free | Limits | 1 | 1 | 20x | queued | n/a |
| Cron | ≥5 min; delayed at :00; may be dropped | Events | 1/h | 1/h | | run late or missing | monitoring.md |
| Schedule disablement | 60 days without activity | Events | | | Dependabot weekly | schedule off; healthchecks alerts | monitoring.md |
| Runner disk | 14 GB documented; ~18–22 GB free reported | Runners, third party | ≤4.2 GB | 6.87 GB | 2x on 14 GB | `ENOSPC`, rsync exit 11, job fails | sync-with-dante.md |
| Runner RAM | 16 GB | Runners | <300 MB | <1 GB expected | 16x+ | OOM kill | taskfile-architecture.md |
| Job summary | 1 MiB/step, 20 steps | Workflow commands | ~2 KB | ~2 KB | 500x | summary dropped, annotation, step passes | monitoring.md |
| Log volume | undocumented; lines drop | observed | ~50 lines | ~500k if echoed | | silent drop | monitoring.md (`report` counts from `RUN/`) |
| dante `max connections` | unknown; default unlimited | rsyncd.conf(5) | 2 sequential | 31 sequential | | `@ERROR`, exit 5, retried with jitter | errors-and-issues.md |
| dante listing | 6.9 s, 50.7 MB | measured | 1 | 1 | | rsync exit 10/12/30/35, retried | sync-with-dante.md |
| CTAN hourly rule | ≥1 sync/hour, fixed minute | register page | 1/h | | | delisted after ~28 h stale | official-mirror-and-url.md |
| mirmon | fresh ≤28 h | mirmon | | | 27 h | not redirected to | monitoring.md |
| rsync `--files-from` | no documented cap; `#`/`;` lines are comments | rsync(1) | ~25 lines | 97,569 lines, 4.6 MB | | | sync-with-dante.md |
| rsync `MAXPATHLEN` | 1,024 | rsync.h | 151 | same | 6.8x | path rejected | sync-with-dante.md |
| rsync `--max-alloc` | ~1 GB per allocation | rsync(1) | | ~10 MB file list | 100x | exit 22 | errors-and-issues.md |
| rsync `--timeout` / `--contimeout` | 300 s / 60 s (self-imposed) | Taskfile | | | | exit 30 / 35, retried | errors-and-issues.md |
| AWS CLI `max_queue_size` | 1,000 tasks | s3-config | | | bounds memory | queue blocks, no failure | taskfile-architecture.md |
| AWS CLI retries | standard: 3 attempts by default, 20 s backoff cap | Retries | | | `retry_mode = standard`, `max_attempts = 10` | error after last attempt, job fails | errors-and-issues.md |
| AWS CLI timeouts | connect 60 s, read 60 s default | CLI options | 60 / 300 | same | | `ReadTimeoutError`, retried | errors-and-issues.md |
| `xz` memory | 100 MB at `-6`, 421 MB at `-9` | measured | 1 compress | 30 | 16 GB RAM | | taskfile-architecture.md |
| `sort` memory | 68 MB for 496k lines | measured | 2 sorts | same | | spills to `/tmp` | taskfile-architecture.md |
| `shasum -a 512` | ~650 MB/s here | measured | ≤4 GB/batch, ~7 s | 30 batches | | | verification-and-security.md |
| `split` suffixes | `aa`..`zz` = 676 files | measured | 1 file | 4,962 files | needs `-a 3` | "too many files" | caching.md |
| healthchecks pings | 5/min/check | Pinging API | 1 | 1 per run | 5x | 200, not recorded | monitoring.md |
| healthchecks body | 100 kB | Pinging API | ~200 B | same | 500x | truncated | monitoring.md |
| healthchecks checks / log | 20 / 100 entries | Pricing | 1–2 / ~4 days | | | oldest entries roll off | monitoring.md |

## 9. Local runs on the developer's Mac

CLAUDE.md says the Taskfile runs the same way locally. Task 3.53.1 executes `cmds` in its
own POSIX shell (mvdan/sh), so shell syntax is identical on both platforms; only external
tools differ. Probed today on macOS 26.5.2 (arm64), every line reproducible with the
command in the second column. The design should rely on none of the GNU-only forms.

| Tool | macOS today | Ubuntu 24.04 runner | Rule for the Taskfile |
|---|---|---|---|
| `rsync` | `/usr/bin/rsync` is **openrsync**, "protocol version 29", "rsync version 2.6.9 compatible". `--help` lists `--files-from`, `--list-only`, `--stats`, `--contimeout`, `--timeout`, `--delete-excluded`, `--partial`, `--no-motd`, `--out-format`, `--no-implied-dirs`, `--max-size`, `--itemize-changes`; it does **not** list `--copy-links` (only `-L`), `--relative` (only `-R`), `--info`, `--from0`, `--human-readable`, `--ignore-missing-args`. Zero-byte files list with a blank size; no thousands separators | rsync 3.2.7, protocol 31; sizes in `--list-only` carry commas | use short flags `-L`, `-R`, `-r`, `-t`; avoid `--info`, `--from0`, `-h`; the listing parser accepts an empty size and strips commas (both shown in Measurements). Everything in today's `fetch` line is on both |
| `date` | BSD: `date -d @0` fails ("illegal option -- d"); `date -r 0` and `date -j -f '%Y/%m/%d %H:%M:%S' ... +%s` work | GNU: `-d`, no `-r` epoch, no `-j` | only `date -u '+FORMAT'` (already the only use); never parse a date with `date` |
| `stat` | `stat -c` fails; `stat -f %z` works | `stat -c %s` | use `wc -c <` or the listing's size column; never `stat` |
| `sed` | BSD: `sed -i ''` needs the empty argument; `sed -i 's/a/b/' f` errors; `-E` works, `-r` also accepted; `\t` in the RHS is a literal `t`-tab on BSD (printed a tab today) but not portable | GNU: `sed -i`, `-E`/`-r` | stream through sed, never `-i` (as `page` does); use `-E`; put a literal tab in the script via `$'\t'` or awk |
| `du` | `du -b` and `--apparent-size` fail; `du -sm` and `du -A -m` work | GNU has all | `du -sm` only (as today) |
| `sort` | "2.3-Apple (199)", GNU-derived: `-S`, `--parallel`, `-s`, `-h`, `-V`, `-t $'\t' -k2,2n`, `-o same-file` all work | GNU 9.4 | always `LC_ALL=C`; `-o` for in-place |
| `comm` | BSD: `--nocheck-order` fails ("illegal option"); unsorted input gives no warning, exit 0 | GNU: warns "file 1 is not in sorted order" on unsorted input (exit status on that condition **unverified**) | sort both sides with `LC_ALL=C sort` immediately before every `comm`; never rely on `comm` to detect order |
| `awk` | BWK "20200816": no `gensub`, `strftime`, `systime`; `length()` counts bytes under `LC_ALL=C`; `printf "%d"` handles 1.4e11; `print > "a" b` needs parentheses around the target expression (the unparenthesised batch-plan line failed today) | `awk` is mawk by default; gawk may or may not be present (not in the image README) | POSIX awk only: no gawk extensions, `print > (expr)`, `LC_ALL=C` |
| `grep` | this Mac's `grep` is **ugrep 7.8.4**; stock `/usr/bin/grep` is BSD grep without `-P`; `-o -E`, `-c`, `-F` fine | GNU grep with `-P` | `-E`, `-F`, `-c`, `-o` only; never `-P` |
| `xargs` | BSD accepts `-r` (no-op: BSD never runs on empty input), `-I{}` fine | GNU: `-r` needed | keep `-r` (harmless on BSD, required on GNU), as today |
| `split` | `-l`, `-d`, `-a` work; default suffix space is 676 files | GNU same, plus `-n` | `split -a 3 -l 100` for purge chunks (4,962 files) |
| `find` | `-type f`, `-newermt`, `-mmin` work; `-printf` printed a size today (Apple's find has grown it) but is not on every macOS | GNU | `find -type f` + `sed`, never `-printf` |
| `head -c`, `tail -n +2`, `cut -f`, `tr`, `uniq -c`, `wc -l`, `cmp -`, `mktemp -d`, `readlink -f`, `seq -w`, `base64` (no wrap by default here) | all work | all work | `wc -l` pads with spaces on BSD: compare numerically or `tr -d ' '` |
| `sha512sum` | present at `/sbin/sha512sum` on this macOS; `shasum -a 512 -c --quiet` works, `--ignore-missing` and `--strict` accepted, keys with spaces verify | `sha512sum` and `shasum` (Perl) both present | `shasum -a 512` as today; it is in the tool list and on both |
| `timeout` | absent from macOS; present via Homebrew here | coreutils | never rely on `timeout`; use each tool's own timeout flags |
| `xz` | 5.8.3, `-T0` works | 5.4.5 | `xz -T1 -6` for reproducible memory; `-T0` is fine on both |
| `curl` | 8.7.1 with `--retry-all-errors`, `--retry-connrefused`, `--fail-with-body`, `--retry-max-time`, `--parallel`, `--json` | 8.5.0 has the same | |
| `gpgv` | 2.5.21 (Homebrew; not on stock macOS) | 2.4.4 | already a documented local prerequisite |
| `aws` | not installed locally | 2.36.24 | `publish` has no local path today and none in the design; a `--dry` render is the local check |
| `/bin/sh` | bash 3.2.57 in POSIX mode: `echo -n` prints `-n` | dash | irrelevant under Task (mvdan/sh runs the commands); matters only if a `cmd` is a `sh -c '...'` string, which it never should be |
| `mv -T`, `cp --parents`, `ln -r`, `install -D`, `touch -d`, `printf "%'d"`, `numfmt` | first six fail or differ on BSD; `printf "%'d"` works; `numfmt` is Homebrew-only | GNU | do not use; the `report` task's `mb()` awk is the portable formatter |
| Machine | 14 cores, 48 GiB RAM, 61 GiB free on `/`; `ulimit -n` 1,048,576 | 4 vCPU, 16 GB, 14 GB | a local run of the list-diff pipeline needs one batch (≤4 GB) or one installer (6.87 GB) on disk, never the 133 GB tree |

The one macOS-only trap that would silently corrupt data: a listing parser that assumes a
numeric size column. openrsync prints nothing for a zero-byte file, so
`sed -E 's/^[^ ]+ +([0-9]+) +.../'` skips 525 lines and passes them through unchanged.
The portable form takes `[0-9]*` and treats empty as 0 (Measurements). GNU rsync's commas
are the mirror-image trap on the runner.

## Where this differs from the base plan

- **Batches.** "~34 batches" is 25 four-GB batches plus 5 lone files = 30 rsync
  connections when packed from today's listing; the largest batch is 97,569 files (fonts).
- **Multipart cost.** With the current `multipart_threshold = 200MB`, 16 objects go
  multipart, not 5, and at the default 8 MiB chunk that is 4,676 parts. `multipart_threshold
  = 4GB` and `multipart_chunksize = 512MB` (the CLI's suffixes are binary: 4 GiB and
  512 MiB) make it 5 objects, 13 parts each, 75 Class A. The base plan's step 3 ("per-file
  `multipart_threshold` override") is unnecessary: one `aws.config` covers every file.
- **Retry mode.** `retry_mode = standard`, not the base plan's experimental `adaptive`;
  `max_attempts = 10` keeps the ~3-minute per-call budget.
- **Origin timeout.** Cloudflare's page now says 125 seconds, not 100; and it applies to
  time to first byte, so the 6.78 GB ISO is not at risk. No "500-second" limit exists on
  any page fetched.
- **Blank sizes.** 525 listing lines carry no size on macOS; the base plan's awk
  (`$1 ~ /^-/ {x=$2; ...}`) reads the date as the size for those lines. Its totals were
  computed with that error; the effect is under 1 MB and the object count is unaffected.
- **Concurrency queue.** Verified as the base plan describes, and GitHub now also offers
  `queue: max` (up to 100 pending). The default is still right: a replaced pending run
  costs nothing, and `queue: max` cannot combine with `cancel-in-progress`.
- **Purge versus the 1,200 API limit.** The API page says cache purge has its own limits;
  the base plan counts purges against both. Kept conservative here: 990 calls per 5
  minutes at 3.3/s, 21% under 1,200 even if they count.
- **Log drops.** Still undocumented by GitHub; the closest public thread caps streaming
  logs near 4 MB. The base plan's rule (count from `RUN/`) stands.
- **rsync exit 22.** Added to the "never retry" set explicitly: it is a memory failure,
  not transport.
- **`split` suffixes.** 4,962 purge chunks exceed the default two-letter suffix space;
  `-a 3` is required. Not in the base plan.
- **`comm` order checking.** BSD `comm` neither supports `--nocheck-order` nor warns; a
  Taskfile that used `--nocheck-order` would fail locally. The base plan does not use it;
  this file forbids it.
- **Everything else** in the base plan's "Remote calls, limits and error handling",
  "Self-imposed rates", "Disk" and runner facts re-verified today with the same numbers:
  R2 4.995 GiB, 10,000 parts, 1 write/s per key, 1,024-byte keys; Cache Rules 10, Transform
  Rules 10, 512 MB, Smart Tiered Cache on Free, purge 800 URLs/s and 100 per call, purge
  everything 5/min; API 1,200 per 5 min with a 5-minute block; GraphQL 300 per 5 min;
  6-hour job, 5-minute cron floor, 60-day disablement, 1 MiB summary, 14 GB disk, 4 vCPU,
  16 GB; CTAN hourly and 60 GB; mirmon 28 h; healthchecks 5/min, 100 kB, 20 checks, 100
  log entries; AWS CLI standard/adaptive semantics; curl transient set.

## Measurements

All run on 2026-08-26 in `SCRATCH`, where `ctan-list-deref.txt` is today's
`rsync -rL --list-only rsync://rsync.dante.ctan.org/CTAN/` (format
`perms size yyyy/mm/dd hh:mm:ss path`, sizes without commas, blank for zero bytes).

```sh
# Stored set as keys, one per line: regular files, size numeric or blank, minus tlnet's
# versioned containers.
grep '^-' ctan-list-deref.txt \
  | sed -E 's/^[^ ]+ +([0-9]*) +[0-9/]+ +[0-9:]+ +//' \
  | grep -v -E '^systems/texlive/tlnet/.*\.r[0-9]+[^/]*\.tar\.xz$' \
  | grep -v -E '^systems/texlive/tlnet/update-tlmgr-r' > keys.txt              # 496,149

# Normalised state-file shape: path<TAB>size<TAB>date time, sorted bytewise.
# On GNU rsync add  | awk -F'\t' 'BEGIN{OFS=FS}{gsub(",","",$2); print}'  after the sed.
grep '^-' ctan-list-deref.txt \
  | sed -E 's/^[^ ]+ +([0-9]*) +([0-9/]+) +([0-9:]+) +(.*)$/\4\t\1\t\2 \3/' \
  | grep -v -E '^systems/texlive/tlnet/[^\t]*\.r[0-9]+[^/\t]*\.tar\.xz\t' \
  | grep -v -E '^systems/texlive/tlnet/update-tlmgr-r' \
  | LC_ALL=C sort > norm.sorted                                                     # 0.84 s + 0.11 s

# Character classes and lengths.
LC_ALL=C grep -c '[^ -~]' keys.txt                       # 0 non-ASCII or control
for c in ' ' '+' '#' '%' '&' '?' '\' '~' '$' '!' ',' ':' '@' '=' '(' ')' '[' ']' '{' '}' '>'; do
  printf '[%s] %s\n' "$c" "$(grep -c -F -- "$c" keys.txt)"; done
grep -c -E '(^|/)-' keys.txt                              # 4 leading-dash components
grep -c -E '\.$' keys.txt                                 # 2 trailing-dot names
grep -c -E '(^|/)\.[^/]' keys.txt                         # 37 hidden components
grep -c '^[#;]' keys.txt; grep -c ' $' keys.txt           # 0, 0 (files-from comments, trailing space)
LC_ALL=C awk 'length($0)>m{m=length($0);k=$0} END{print m, k}' keys.txt          # 151 bytes
LC_ALL=C awk -F/ '{for(i=1;i<=NF;i++) if(length($i)>m) m=length($i)} END{print m}' keys.txt  # 90
awk -F/ 'NF>m{m=NF} END{print m}' keys.txt                # 15 levels
LC_ALL=C awk '{s+=length($0)} END{print s/NR}' keys.txt   # 49.8 mean

# Blank-size lines (zero-byte files as openrsync prints them).
grep -c -E '^[^ ]+ +[0-9]{4}/[0-9]{2}/[0-9]{2} ' ctan-list-deref.txt              # 525

# Sizes, multipart candidates, per-batch counts.
awk -F'\t' '$2>5363466240' norm.sorted                    # 5 objects over 4.995 GiB
awk -F'\t' '$2>536870912' norm.sorted | wc -l             # 7 over 512 MiB
awk -F'\t' '$2>200000000{n++; b+=$2; p8+=int(($2+8388607)/8388608)} END{print n, b/1e9, p8}' norm.sorted
awk -F'\t' 'BEGIN{b=1} {sz=$2+0; if(sz>5363466240){l++; next} if(s+sz>4e9){print b, n, s/1e9; b++; s=0; n=0} s+=sz; n++}
  END{print b, n, s/1e9; print "lone:", l}' norm.sorted   # 25 batches + 5 lone; max 97,569 files
# Batch plan writing one file per batch (note the parentheses around the redirect target):
awk -F'\t' 'BEGIN{b=1} {sz=$2+0; if(sz>5363466240){print $1 > ("batch-lone-" (++l) ".txt"); next}
  if(s+sz>4e9){b++;s=0} s+=sz; print $1 > ("batch-" b ".txt")}' norm.sorted       # 1.26 s

# Churn and peaks (mtime as proxy).
awk -F'\t' '{split($3,d," "); if(d[1]>="2026/07/27"){n++;b+=$2}} END{print n, b/1e9}' norm.sorted   # 16,574 / 4.72 GB
awk -F'\t' '{split($3,d," "); if(d[1]>="2025/08/26"){h=d[1]" "substr(d[2],1,2); n[h]++; b[h]+=$2}}
  END{for(k in n) print n[k], b[k]/1e9, k}' norm.sorted > hours.txt
sort -rn hours.txt | head -1; sort -k2 -rn hours.txt | head -1
awk '$3>="2026/07/27"' hours.txt | wc -l                  # 284 active hour-slots of 720

# comm timings and memory (a synthetic "applied" with 100 changed sizes, 50 removed, 30 added).
awk 'NR%5000==0{$2=$2+1} NR%10000!=7' norm.sorted > applied.txt
awk 'NR%20000==0{print $0"-new"}' norm.sorted >> applied.txt; LC_ALL=C sort -o applied.txt applied.txt
/usr/bin/time -l sh -c 'LC_ALL=C comm -13 applied.txt norm.sorted > changed.txt'   # 0.10 s, 2 MB
cut -f1 applied.txt > a.paths; cut -f1 norm.sorted > u.paths
/usr/bin/time -l sh -c 'LC_ALL=C comm -23 a.paths u.paths > deleted.txt'           # 0.09 s, 2 MB
/usr/bin/time -l sh -c 'LC_ALL=C sort norm.txt > /dev/null'                        # 0.11 s, 68 MB

# xz, split, shasum.
/usr/bin/time -l sh -c 'xz -T1 -6 -k -c norm.sorted > norm.6.xz'                   # 5.7 s, 100 MB, 2.97 MB out
/usr/bin/time -l sh -c 'xz -T1 -9 -k -c norm.sorted > /dev/null'                   # 7.0 s, 421 MB
cut -f1 norm.sorted | split -a 3 -l 100 - purge.                                   # 4,962 files
dd if=/dev/zero of=zero.bin bs=1048576 count=2048; time shasum -a 512 zero.bin      # 3.3 s for 2 GiB
```

Live probes today (read-only):

```sh
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512   # 200, cf-cache-status: DYNAMIC, last-modified 18:07 UTC
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/archive/latex.tar.xz         # 200, application/x-xz, DYNAMIC
curl -sI https://ctan.ijosh.com/                                                    # 200 text/html, cache-control: no-cache
curl -sI https://ctan.ijosh.com/systems/texlive/tlnet/                              # 404 (directory URLs)
curl -sL https://mirror.ctan.org/timestamp | head -3                                # "# This file is for administrative purposes only..."
```

## Open questions

- Does R2 count concurrent `UploadPart` calls on one key against "1 write per second per
  key"? Settled by the first multipart upload with `--debug` (look for 429). If yes, 13 parts of 512 MiB at 32-way concurrency still finish under standard-mode
  retries (`max_attempts = 10`); lowering `max_concurrent_requests` for that one upload is
  the fallback.
- Is the R2 free tier a "Paid Service" for the CDN terms' large-file clause? A reading of
  two documents, not a number. Owner: official-mirror-and-url.md.
- Do purge-by-URL calls count against the 1,200-per-5-minute token limit as well as their
  own 800 URLs/s? The API page says purge "has its own limits"; the pipeline is sized as if
  both apply.
- Edge TTL override minimum and maximum on Free: not on the settings page. Settled by the
  first `PUT` of `cloudflare/cache-rules.json`.
- Actual free disk on the runner this repo gets: `df -h /` in one run.
- Dante's daemon `max connections` and rsync version: the MOTD or the maintainers' list.
- Whether GitHub emails for scheduled runs cancelled by concurrency replacement during a
  seed (5 or 6 of them).
- AWS CLI 2.36's default flexible checksums against R2's `PutObject`: working today for
  tlnet; watch the first run after any CLI bump on the image.
- Peak RSS of `aws s3 cp --recursive` on a 97,569-file batch: `/usr/bin/time -v` once.
- GNU `comm`'s exit status on unsorted input; irrelevant if both inputs are always sorted,
  which the design guarantees.
- Whether the 6.78 GB ISO streams through the custom domain end to end: one full download
  after the seed. Belongs to verification-and-security.md's `smoke` design if it is to be
  routine (it should not be: 6.78 GB an hour is 4.9 TB a month of runner ingress).

## Sources

Fetched 2026-08-26.

- https://developers.cloudflare.com/r2/platform/limits/
- https://developers.cloudflare.com/r2/api/s3/api/
- https://developers.cloudflare.com/r2/api/s3/extensions/
- https://developers.cloudflare.com/r2/objects/multipart-objects/
- https://developers.cloudflare.com/r2/buckets/public-buckets/
- https://developers.cloudflare.com/r2/pricing/
- https://developers.cloudflare.com/cache/how-to/cache-rules/
- https://developers.cloudflare.com/cache/how-to/cache-rules/settings/
- https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- https://developers.cloudflare.com/cache/how-to/tiered-cache/
- https://developers.cloudflare.com/cache/how-to/purge-cache/
- https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/
- https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/
- https://developers.cloudflare.com/rules/transform/
- https://developers.cloudflare.com/fundamentals/api/reference/limits/
- https://developers.cloudflare.com/analytics/graphql-api/limits/
- https://developers.cloudflare.com/ruleset-engine/rulesets-api/
- https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/
- https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/
- https://www.cloudflare.com/service-specific-terms-application-services/
- https://www.cloudflare.com/service-specific-terms-developer-platform/
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjects.html
- https://docs.aws.amazon.com/cli/latest/topic/s3-config.html
- https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-retries.html
- https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-options.html
- https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html
- https://docs.github.com/en/actions/reference/limits
- https://docs.github.com/en/actions/reference/runners/github-hosted-runners
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
- https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs
- https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions
- https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md
- https://github.com/actions/runner-images/issues/13189 and https://github.com/thiagokokada/free-disk-space (third-party disk observations)
- https://github.com/orgs/community/discussions/127903 (streaming log cap, third party)
- https://github.blog/news-insights/product-news/github-hosted-runners-double-the-power-for-open-source/ (4-vCPU runners, via search)
- https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dadsv5-series
- https://www.tigrisdata.com/docs/overview/benchmarks/cloudflare-r2/ (R2 throughput, vendor benchmark)
- https://ctan.org/mirrors/register
- https://ctan.org/mirrors/mirmon
- https://mirror.ctan.org/timestamp
- https://download.samba.org/pub/rsync/rsync.1 (3.5.0)
- https://download.samba.org/pub/rsync/rsyncd.conf.5
- https://raw.githubusercontent.com/RsyncProject/rsync/master/errcode.h
- https://raw.githubusercontent.com/RsyncProject/rsync/master/rsync.h
- https://healthchecks.io/docs/http_api/
- https://healthchecks.io/pricing/
- `curl --manual` (8.7.1, local)
- `SCRATCH/ctan-list-deref.txt`, `SCRATCH/ctan-list-nolink.txt`, `SCRATCH/rsync-time.txt`, `staging/tlpkg/texlive.tlpdb`
