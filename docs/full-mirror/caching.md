# Caching

The Cloudflare edge cache in front of the R2 custom domain: whether the mirror needs it, what
the Free plan does with it, how it is configured from the repo, how it is kept correct, and how
`smoke` proves it. Every number is computed from the listing or the tlpdb with the command shown,
or quoted from a page fetched on 2026-08-26, or marked unverified with the experiment that
settles it. Live probes were HEAD/GET requests against `https://ctan.ijosh.com/` on the same day.

## Summary

- The mirror does not need a cache to stay free. Uncached, every GET is one Class B operation
  at $0.36 per million after 10 million a month. That is 28 `scheme-full` installs a day before
  the first cent, 36 a day before the first dollar, and about $2.50 to $7 a month at the traffic
  of one of the largest mirrors on earth (ftp.fau.de served 95 TB of CTAN in 2018). The cache is
  a latency feature for users far from the bucket and a cost feature only at traffic this mirror
  cannot draw.
- The cache does not bound the bill either. Cloudflare evicts at its discretion, purge is by
  URL, and a 404 flood is a Class B per request with or without a cache. The real
  bound is $0.36 per million requests, full stop.
- So caching is an optional layer with one switch, `CACHE: 'on'|'off'` in `Taskfile.yml`, off
  by default. Off means one rule on the zone: bypass cache for the whole host. On means two
  rules: bypass for the decision files, eligible with a one-day Edge TTL for everything else,
  plus purge-by-URL of changed and deleted keys after every batch.
- Today's mirror already has staleness windows. `.zip`, `.exe`,
  `.tar.gz` and `.gz` are on Cloudflare's default extension list, so `install-tl.zip`,
  `install-tl-unx.tar.gz`, `install-tl-windows.exe`, `update-tlmgr-latest.exe` and (in a full
  mirror) `FILES.byname.gz` and `tds.zip` are cached for 120 minutes with nothing purging them,
  and a 404 on such a path is cached for 3 minutes. Observed live. The bypass rule in the "off"
  state removes these windows; that is why "off" still has a rule.
- Purge order is settled: upload, then purge; and within the last batch, purge the changed
  `archive/` keys before `tlpkg/` lands. The seed purges nothing. Only keys that already
  existed in the bucket are purged, because 404s are never stored (`status_code_ttl` `-1`).
- Smart Tiered Cache is a persistent zone setting, `PATCH` not `POST`, needing `Zone Settings
  Write`. Turn it on once in the dashboard; the run only checks it with `Zone Settings Read`.

## 1. Does the mirror need a cache?

### What one user costs

From `staging/tlpkg/texlive.tlpdb` (2026-08-25), following `depend` lines from the scheme
through collections to packages, `.ARCH` resolved to `x86_64-linux`:

```sh
awk -v scheme=scheme-full -v arch=x86_64-linux '
/^name /{n=$2; names[n]=1} /^depend /{dep[n]=dep[n]" "$2}
/^containersize /{cs[n]=$2} /^doccontainersize /{ds[n]=$2} /^srccontainersize /{ss[n]=$2}
END{q[1]=scheme;h=1;t=1;seen[scheme]=1
 while(h<=t){p=q[h++];m=split(dep[p],d," ");for(i=1;i<=m;i++){x=d[i];sub(/\.ARCH$/,"."arch,x)
  if((x in names)&&!(x in seen)){seen[x]=1;q[++t]=x}}}
 for(p in seen){if(cs[p]>0){g++;b+=cs[p]} if(ds[p]>0){gd++;bd+=ds[p]} if(ss[p]>0){gs++;bs+=ss[p]}}
 printf "%d run (%.2f GB) %d doc (%.2f GB) %d src (%.2f GB) = %d GETs %.2f GB\n",g,b/1e9,gd,bd/1e9,gs,bs/1e9,g+gd+gs,(b+bd+bs)/1e9}' texlive.tlpdb
```

| Scheme (`x86_64-linux`) | Packages | GETs | Bytes |
|---|---:|---:|---:|
| `scheme-full` | 5,170 | 11,913 (5,170 run, 4,721 doc, 2,022 src) | 5.51 GB |
| `scheme-medium` | 1,623 | 3,456 | 1.75 GB |
| `scheme-small` | 428 | 960 | 0.65 GB |
| `scheme-basic` | 138 | 299 | 0.17 GB |

Plus five uncached GETs per run (`texlive.tlpdb.xz`, `.sha512`, `.sha512.asc`, and the
installer's own files).

### Where the money starts

R2 Class B: 10,000,000 free per month, then $0.36 per million (`r2_pricing.md`, fetched
2026-08-26). Each edge miss is one Class B; each hit is none ("Every cache hit avoids origin
fetch costs, Argo routing charges, Workers execution, and R2 operations",
[how charges accrue](https://developers.cloudflare.com/billing/understand/how-charges-accrue/)).

| Uncached bill per month | GETs/month | `scheme-full` installs/month | per day |
|---|---:|---:|---:|
| $0 (inside free tier) | 10.0M | 839 | 28.0 |
| $1 | 12.8M | 1,073 | 35.8 |
| $5 | 23.9M | 2,006 | 66.9 |
| $10 | 37.8M | 3,171 | 105.7 |

Installs/month = GETs ÷ 11,913. A `tlmgr update` user is ~100 to 150 GETs a month (an estimate from the 30-day container churn, not re-derived here), so the free tier is also ~70,000 to 100,000 regular users.

### What real CTAN traffic looks like

The only published per-mirror figure found: ftp.fau.de, one of the largest public mirrors,
lists CTAN at "95" TB for 2018 and "10 / 93" (rank / TB) for 2017 out of a site total that
"increased from 2.51 PB in 2017 to 3.22 PB in 2018"
([Mirrors generating most traffic in 2018](https://ftp.fau.de/news/posts/mirrors-generating-most-traffic-in-2018/),
2019-09-23). CTAN's register page says only that "mirror traffic is light, perhaps one visitor
is logged in at any time". Wikipedia's "more than 6 TB per month" for the core nodes is
uncited.

Converted with the stored set's average object (132.99 GB / 496,149 objects, the set adopted across
these files, = 268 KB; a `scheme-full` install averages 462 KB per GET):

| Traffic | GB/day | GETs/month at 268 KB | at 462 KB | Uncached bill |
|---|---:|---:|---:|---:|
| ftp.fau.de CTAN, 2018 | 260 | 29.1M | 16.9M | $6.90 / $2.50 |
| "6 TB/month", all of it here | 200 | 22.4M | 13.0M | $4.50 / $1.10 |
| One of ~132 mirrors' share of 6 TB | 1.5 | 0.17M | 0.10M | $0 |

The honest answer: **not until roughly 30 `scheme-full` installs a day**, and even a
world-class mirror's traffic is single-digit dollars uncached. That is the whole cost case.

### What the cache does bound, and what it does not

- It does not bound the bill. Cloudflare's cache is best effort: the Cache Reserve product
  exists to "eliminate cache evictions" ([Cache Reserve](https://developers.cloudflare.com/cache/advanced-configuration/cache-reserve/)),
  which is Cloudflare saying the ordinary cache evicts. A 133 GB long tail on a Free zone will
  be evicted constantly; the worst case with the cache is the worst case without it. A 404
  flood (`/x/<random>`) is one Class B per request under any configuration, since 404s must
  not be stored (section 4). A ceiling of the form "every object refetched from origin once a
  day" would need evictions to happen at most daily, and nothing documents any eviction
  schedule.
- It does bound origin fetches for hot objects. The ~11,900 containers of a `scheme-full`
  install are the same for every user on a platform; with a warm edge they cost zero Class B.
- It buys latency. `tlmgr` downloads sequentially over one connection (`TLDownload` persistent
  LWP, or `curl`/`wget` one file at a time), so per-request round trip is multiplied by 11,913.
  An edge hit is a few ms; an origin fetch adds the edge-to-bucket round trip on every file.
  With Smart Tiered Cache a lower-tier miss goes to the upper tier next to the bucket, so the
  latency win is only for objects warm at the user's data center.

Decision: optional layer, off until `report` shows month-to-date Class B above 5 million (half
the free tier) for two consecutive months, or a user reports install times that an edge hit
would fix. The switch is one line.

## 2. What the Free plan caches from an R2 custom domain

All quotes from pages fetched 2026-08-26.

| Fact | Quote | Page |
|---|---|---|
| Custom domain required | "The development URL (r2.dev) does not support caching, WAF, or bot management. You must use a Custom Domain for these features." | [R2 and cache](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/) |
| Default: extension list only | "Cloudflare only caches based on file extension and not by MIME type." | [Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) |
| The list | 7Z AVI AVIF APK BIN BMP BZ2 CLASS CSS CSV DOC DOCX DMG EJS EOT EPS EXE FLAC GIF GZ ICO ISO JAR JPG JPEG JS MID MIDI MKV MP3 MP4 OGG OTF PDF PICT PLS PNG PPT PPTX PS RAR SVG SVGZ SWF TAR TIF TIFF TTF WEBM WEBP WOFF WOFF2 XLS XLSX ZIP ZST | same |
| Not on it | `xz`, `tfm`, `vf`, `sty`, `tex`, `lzma`, `ini`, `pfb`, `afm`, `fd`, `dtx`, `md`, and every extensionless file (`timestamp`, `install-tl`) | same |
| Size cap | "Free, Pro and Business customers have a limit of 512 MB." | same |
| Default TTLs | "200, 206, 301 \| 120m", "302, 303 \| 20m", "404, 410 \| 3m", "All other status codes are not cached by default." | same |
| No header needed | "Cloudflare does cache the resource even if there is no Cache-Control header based on status codes." | same |
| Rule needed regardless of headers | DYNAMIC "typically happens when: The requested asset is not one of the default cached file extensions (for example, HTML or JSON) and no rule instructs Cloudflare to cache it." | [Cache responses](https://developers.cloudflare.com/cache/concepts/cache-responses/) |
| Rules on Free | "Number of rules \| 10" | [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) |
| Edge TTL modes | `respect_origin`, `override_origin` ("Ignore cache-control header and use this TTL"), `bypass_by_default` | [Settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/) |
| Status code TTL values | "`value`: An integer value that defines the duration an asset is valid in seconds or one of the following strings: `no-store` (equivalent to `-1`), `no-cache` (equivalent to `0`)." | [Cache by status code](https://developers.cloudflare.com/cache/how-to/configure-cache-status-code/) |
| Edge TTL minimum | "Minimum Edge Cache TTL \| 2 hours" on Free | [Edge and Browser TTL](https://developers.cloudflare.com/cache/how-to/edge-browser-cache-ttl/) |
| Browser TTL default | "Default Browser Cache TTL \| 4 hours"; Cloudflare "overrides those headers if ... The origin web server does not send a Cache-Control or an Expires header." | same |
| Origin Cache Control | "Free, Pro, and Business customers have this option enabled by default and cannot disable it." | [Settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/) |
| Override beats `no-store` | "A Cache Rule with an Edge Cache TTL setting that ignores origin cache-control overrides these directives, same as it does for `Cache-Control: no-store`." | [Cache responses](https://developers.cloudflare.com/cache/concepts/cache-responses/) |
| Header passes downstream anyway | "When Origin Cache Control is enabled at Cloudflare, the original `Cache-Control` header passes downstream from our edge even if Edge Cache TTL overrides are present." | [Origin Cache Control](https://developers.cloudflare.com/cache/concepts/cache-control/) |
| Ignore query string on Free | availability table: "Ignore query string \| Yes" for Free | [Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/) |
| Bypass shows as DYNAMIC | "When using Custom Cache Rules with a Bypass setting, the response header may return DYNAMIC rather than explicitly indicating a bypass." | [Settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/) |

### What R2 sends, and what happens to it

Observed through the domain today (`curl -sI`):

| Path | `cf-cache-status` | Other headers |
|---|---|---|
| `tlpkg/texlive.tlpdb.sha512` | DYNAMIC, DYNAMIC | `etag`, `last-modified`, `accept-ranges: bytes`; no `cache-control`, no `content-type` |
| `archive/latexmk.tar.xz` | DYNAMIC, DYNAMIC | `content-type: application/x-xz` |
| `README.md` | DYNAMIC | `content-type: text/markdown`, served `content-encoding: br` when asked |
| `install-tl` (no extension) | DYNAMIC | no `content-type` |
| `install-tl.zip` | REVALIDATED, then HIT `age: 0` | `cache-control: max-age=14400` added by Cloudflare |
| `install-tl-unx.tar.gz` (HEAD first) | REVALIDATED, then HIT | HEAD alone filled the cache |
| `update-tlmgr-latest.exe` | MISS, then HIT | |
| `update-tlmgr-latest.sh` | DYNAMIC | |
| `index.html` (uploaded with `--cache-control no-cache`) | DYNAMIC | `cache-control: no-cache` came back from R2 |
| `nonexistent.zip` | MISS, HIT, then EXPIRED ~3 min later | a 404 was cached |
| `nonexistent.tar.xz` | DYNAMIC | 404 not cached: not an eligible extension |
| `archive/LATEXMK.tar.xz` | 404 | paths are case sensitive at R2 |
| `install-tl.zip?x=1`, `?y=2` | MISS each, then HIT | query string is in the default cache key |

So: R2 sends `ETag` (MD5-shaped), `Last-Modified`, `Accept-Ranges` and no `Cache-Control`
unless one was stored at upload; Cloudflare adds `cache-control: max-age=14400` (the zone's
default Browser Cache TTL) to cached responses and nothing to DYNAMIC ones.

### Per-object `Cache-Control` instead of a zone rule: evaluated and rejected

R2 stores `Cache-Control` from `PutObject` and returns it (the S3 API table lists
"System Metadata: Content-Type, Cache-Control, Content-Disposition, Content-Encoding,
Content-Language, Expires" for PutObject and CreateMultipartUpload,
[S3 API](https://developers.cloudflare.com/r2/api/s3/api/); the AWS CLI's `--cache-control`
"Specifies caching behavior along the request/reply chain" and applies to `cp` and `sync`;
`index.html` proves the round trip). It costs no extra operation: the header rides on the
PutObject that happens anyway. Three reasons it cannot carry the design:

1. A header cannot make `.xz` cacheable. Eligibility is by extension or by rule (DYNAMIC quote
   above). A rule is needed either way, so the header could only carry TTL and exclusions.
2. Exclusions cannot ride on the header under an override rule: `override_origin` overrides
   `no-store` (quote above). Under `respect_origin` they could, but then the TTL must also be a
   header, and `s-maxage` (the only way to give the edge a long TTL without giving browsers one)
   "passes downstream from our edge", so every university proxy between the mirror and its users
   would hold containers for the same year. Containers change in place; a proxy cannot be purged.
3. Retrofitting 496k existing objects is 496k `CopyObject` calls (Class A), half the free month.

Kept for one thing: the landing page's `no-cache` today, and any future object that must never
be cached even by accident, since `bypass_by_default` in the "off" rule respects it.

## 3. Range requests

Docs: "Clients can send range requests to be served from the cache using the `Range` header.
Note that: If the origin response includes a `Content-Length` header, then the specified byte
range will be returned with an HTTP 206 response. If the origin response does not include the
`Content-Length` header, the cache will return the full content with an HTTP 200 response."
([Default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/)).
The docs do not say what a range *miss* fetches from the origin. Measured today on two
never-requested default-cacheable objects:

```
curl -r 0-99  .../tlpkg/installer/texlion.gif   → 206, content-range: bytes 0-99/26224, cf-cache-status: MISS
curl          .../tlpkg/installer/texlion.gif   → 200, content-length: 26224, cf-cache-status: HIT, age: 0
curl -r 100-199 ...                             → 206, cf-cache-status: HIT
```

A range miss fetched the whole object from R2 (one Class B, one object's egress, free) and
stored it whole; later ranges and the full GET are hits from that entry. R2 itself supports
`Range` on GetObject, but Cloudflare did not pass it through. `tlmgr` never sends `Range`
(`FallbackDownloaderArgs` in `tlpkg/TeXLive/TLConfig.pm`: curl gets `--user-agent texlive/curl
--retry 4 --retry-delay 4 --connect-timeout N --fail --location --silent --output`; wget gets
`--user-agent=texlive/wget --tries=4 --timeout=N -q -O`; no `-C`/`--continue`), so this matters
only for browsers and download managers fetching installers and ISOs.

Whether the 512 MB cap applies to the object or the response is not documented; the wording is
per file ("Files that exceed the cacheable size limits are not cached", BYPASS lists "The
response exceeds the maximum cacheable file size"). Unverified for a `Range` on a 6.8 GB ISO:
after the seed, `curl -r 0-99 https://ctan.ijosh.com/systems/texlive/Images/texlive.iso` twice,
expect BYPASS both times, and read R2's egress graph for that hour: 200 bytes means the range was
passed through, 13.6 GB means Cloudflare pulled the whole file twice. Seven objects are over
512 MB (36.4 GB; `awk '$1 ~ /^-/ && $2 > 536870912' ctan-list-deref.txt`), 566 are between
100 MB and 512 MB (7.4 GB).

## 4. Purge

### The API, verified

- `POST /zones/{zone_id}/purge_cache`, body `{"files": ["https://host/path", ...]}`; "All
  tiers can purge by URL"; permission "`Cache Purge`"
  ([API reference](https://developers.cloudflare.com/api/resources/cache/methods/purge/)).
- Free limits, per account: single-file "800 URLs per second", "Max operations per request
  \| 100"; hostname, tag, prefix and everything "5 requests per minute", "Bucket size \| 25",
  100 operations per request; "the thresholds for URLs are calculated using a moving average"
  ([Purge cache](https://developers.cloudflare.com/cache/how-to/purge-cache/)).
- Purge options on Free: "URL, Hostname, Tag, Prefix, and Purge Everything" (same page; all
  methods reached every plan on 2025-04-01,
  [blog](https://blog.cloudflare.com/instant-purge-for-all/)). Prefix and hostname purge exist
  on Free at 5 calls a minute; neither is used here: an hourly delta has no common prefix, and a
  hostname purge is a purge of everything.
- URL rules: "the host part of the URL is not case-sensitive ... However, the path portion is
  case-sensitive"; "Always use UTF-8 encoded URLs"; "Wildcards are not supported on single file
  purge"; with a Transform Rule "you must use the non-transform (end user) URL"
  ([Purge by single-file](https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/)).
- Result: "Purging by single-file deletes the resource, resulting in the `CF-Cache-Status`
  header being set to MISS for subsequent requests." Purge Everything instead "invalidates the
  resource, resulting in ... EXPIRED".
- Propagation: no number on the docs pages. Cloudflare's blog: "single-file purges have a P50
  of 234ms" (2024-09-24, [Instant Purge](https://blog.cloudflare.com/instant-purge/)); "The
  current P50 performance is around 250 ms" (2025-04-01). Tag/host/prefix: "less than 150ms on
  average (P50)".
- Tiered cache: purge is "across all data centers"; upper tiers are data centers. The prefix
  and hostname pages add: "If tiered cache is used, purging by prefix may return `EXPIRED`, as
  the lower tier tries to revalidate with the upper tier". Whether a single-file purge clears
  the upper tier before a lower tier revalidates against it is not stated. Unverified; the
  experiment is in section 10.
- Global API limit: "1,200 requests per five minute period per user", then 429 for five
  minutes; the page lists "Cache Purge APIs" as having their own limits but does not say they
  are exempt from the global one ([API limits](https://developers.cloudflare.com/fundamentals/api/reference/limits/)).
  Budget for both.

### 404s are cached, so do not let them be

R2's own consistency page: "By default, Cloudflare's cache will cache HTTP 404 (Not Found)
responses automatically. If you upload an object to that same path, the cache may continue to
return HTTP 404s until the cache TTL (Time to Live) expires and the new object is fetched from R2
or the cache is purged." ([Consistency model](https://developers.cloudflare.com/r2/reference/consistency/)).
Observed today on `nonexistent.zip`: MISS, HIT, EXPIRED after the 3-minute default.

With `"status_code_ttl": [{"status_code_range":
{"from": 300}, "value": -1}]` (no-store) in the rule, a 404 is never stored, so a key that did
not exist cannot be served stale after it appears, and **added keys need no purge**. Only keys
that already existed (changed) and keys removed (deleted; "An object you delete from R2, but
that is still cached, will still be available. You should purge the cache after deleting
objects") are purged.

Cost of storing 404s briefly would have been a bound on repeated probes of one bad path. Not
worth the seed stream: a probe is one Class B either way, and probes of different paths are
never bounded by a cache.

### Purge then PUT, or PUT then purge

Purge first: between the purge and the PUT a request re-fills the edge with the old bytes for a
full TTL, and nothing purges it again until the key changes again. Wrong for a day.

PUT first: between the PUT and the purge the edge serves the old bytes to anyone who asks;
after the purge (P50 ~250 ms) everyone gets the new. R2 is "strongly consistent ... readers will
immediately see the latest object globally", so the first miss after the purge is the new
object. **PUT, then purge.** Same for deletes: delete, then purge.

Within the last batch of a run, the one that carries `tlpkg/`: upload everything else, purge
the changed keys, then upload `tlpkg/`. Reason in section 9: a client holding the new tlpdb
must never find an old container in the edge, and the tlpdb is uncached so it is new the moment
it lands. This is a batch-ordering rule for `taskfile-architecture.md`: `tlpkg/` is its own
final batch, and every batch purges after its uploads.

### The seed

With `CACHE: 'off'` during the seed, nothing is cached and nothing is purged. Purging
the whole tree at the seed (496,149 URLs, 4,962 calls; at 800 URLs/s at least 620 s; against
1,200 calls per 5 minutes at least 21 minutes) would purge entries that were never made. Switch to `on` after the seed's `changed`
reaches zero. At switch-on nothing but default-extension objects (≤120 min old) is in the
cache; one Purge Everything (5 a minute on Free) clears even those.

Steady state: 16,928 keys changed in the last 30 days on the stored set
(`awk '$1 ~ /^-/ && $3 >= "2026/07/27" && $5 !~ /\.r[0-9]+\.tar\.xz$/' ctan-list-deref.txt`),
23.5 an hour on average, one purge call. Worst hour-slot in the last seven days: 4,948 keys
(2026-08-23 09:xx), 50 calls, 15 seconds at the loop's 0.3 s pause, 6 % of the 800 URLs/s
average.

## 5. Cache key subtleties

| Concern | Fact | Design |
|---|---|---|
| Query string | Default key is "URI with query string"; `?x=1` and `?y=2` each MISSed today. "Ignore query string" is on Free. | `"cache_key": {"custom_key": {"query_string": {"exclude": ["*"]}}}` so `?cachebust=N` cannot multiply origin fetches. Whether the API accepts `custom_key` on Free is unverified (the settings page files it under Enterprise "additional options", the cache-keys availability table says Free: Yes); the `PUT` answers. |
| Case | R2 keys and the cache key are case sensitive (`LATEXMK.tar.xz` → 404). CTAN paths are mixed case (`macros/latex/contrib/`, `documentation/LaTeX_Tips_und_Tricks`). | Nothing to do; a wrong-case URL is a 404, never stored. |
| `Origin` header | Part of the default key ("Origin header sent by client (for CORS support)"). | Browsers on other sites get separate entries; `tlmgr` sends none. Ignore. |
| `Accept-Encoding` | Cloudflare compresses only listed content types (`text/plain`, `text/html`, `text/x-markdown`, `application/json`, fonts...; [Compression](https://developers.cloudflare.com/speed/optimization/content/compression/)). `application/x-xz`, `application/zip`, `application/x-tar`, untyped `.tfm`/`.sty` are not compressed; `README.md` (`text/markdown`) came back `br`. Vary "values are respected ... when the vary header is vary: accept-encoding"; R2 sends no `Vary`. | Nothing. `tlmgr`'s curl and wget do not send `Accept-Encoding`. Text files may lose `Content-Length` when compressed; no tool here checks it. |
| `Content-Type` | The AWS CLI guesses from the extension ("By default the mime type of a file is guessed when it is uploaded", [`aws s3 cp`](https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html)): `.xz` → `application/x-xz`, `.zip` → `application/zip`, `.tar.gz` → `application/x-tar`, `.md` → `text/markdown`, no extension → none. R2's 404 body is `text/plain` or `text/html`. | Irrelevant to caching ("only caches based on file extension and not by MIME type") and to `tlmgr`. |
| HEAD | "Cloudflare converts `HEAD` requests to `GET` requests for cacheable requests ... cache the full response and return the response headers only" ([HEAD and Set-Cookie](https://developers.cloudflare.com/cache/concepts/cache-behavior/)). Observed: HEAD alone filled `install-tl-unx.tar.gz`. | `smoke` may use `-I` for status checks; a HEAD is a full Class B GET on a miss. |
| Trailing slash, `/` | The transform rule rewrites `/` to `/.site/index.html` before the cache phase; the redirect rule answers every other directory URL with a 302 before any cache lookup; purge uses the pre-transform URL. | `/.site/` and `/index.html` are in `D`. |
| Encoding in purge bodies | 31 stored keys contain a space, `#`, `%`, `"` or non-ASCII (`grep -c -E '[ #?%"]\|[^ -~]'` on the path column), e.g. `fonts/mtp2lite/templates/Plain MTPro2.tex`, `language/cyrtug/#disk.00`. | The purge loop percent-encodes `%`, space, `#`, `"`, `?` with `sed`; UTF-8 passes as is. |

## 6. Smart Tiered Cache

"Each edge data center that gets a cache miss fetches content directly from R2. Tiered Cache
changes this ... When a nearby data center has a cache miss, it first checks a designated
upper-tier data center before going to R2." ([R2 and cache](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/)).
"Smart Topology \| Yes" on Free ([Tiered Cache](https://developers.cloudflare.com/cache/how-to/tiered-cache/)).

Effect on the origin-fetch multiplier: without it, one fetch per object per TTL per data center
that serves a user (Cloudflare has hundreds); with it, one per object per TTL at the upper tier,
plus refills after upper-tier evictions, plus lower-tier fills that come from the upper tier
and cost no Class B. `Age` on a lower-tier HIT "can carry an `Age` inherited from an upper
tier" ([Cache responses](https://developers.cloudflare.com/cache/concepts/cache-responses/)).
It does not change latency for cold objects: the upper tier sits next to the bucket.

It is a zone setting, not a per-run action: `GET /zones/{zone_id}/cache/tiered_cache_smart_topology_enable`
returns `{"id": "tiered_cache_smart_topology_enable", "editable": true, "value": "on"|"off",
"modified_on": ...}`; `PATCH` with `{"value": "on"}` sets it. The docs' enable call is
`PATCH` and needs "`Zone Settings Write`" or "`Zone Write`"
([API](https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/edit/)).
Re-patching every run is idempotent and within limits but grants the token write access to
every zone setting for no gain. Design: turn it on once in the dashboard (Caching → Tiered
Cache → Smart), and `rules` reads it back with `Zone Settings Read` and fails the run when
`CACHE` is on and the value is off.

## 7. Configuration as code

Four JSON files in `cloudflare/` (three phases; the cache phase has an `on` and an `off` file), one task, two secrets. The transform and redirect rules are applied whatever `CACHE` says; only the cache file depends on it. `PUT .../phases/{phase}/entrypoint`
"replace[s] the entire list of rules in the ruleset. If you omit existing rules from the request
body, those rules will be removed" ([Update a ruleset](https://developers.cloudflare.com/ruleset-engine/rulesets-api/update/)),
so each file is the whole truth for its phase.

### `cloudflare/cache-on.json`

Two rules with complementary expressions. `D` is the decision-file predicate from section 8;
it appears twice because the precedence between two conflicting cache rules that both match is
not documented, and disjoint expressions need no precedence.

```json
{
  "description": "ctan cache rules @SHA@",
  "rules": [
    {
      "ref": "ctan_decision_files",
      "description": "never cache: indexes, checksums, installers @SHA@",
      "enabled": true,
      "expression": "(http.host eq \"ctan.ijosh.com\" and (starts_with(http.request.uri.path, \"/.state/\") or starts_with(http.request.uri.path, \"/.site/\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/tlpkg/\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/install-tl\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/update-tlmgr\") or starts_with(http.request.uri.path, \"/systems/texlive/tlcontrib/tlpkg/\") or (http.request.uri.path contains \"/miktex/tm/packages/\" and (http.request.uri.path contains \"/miktex-zzdb\" or ends_with(http.request.uri.path, \"/pr.ini\") or ends_with(http.request.uri.path, \"/files.csv.lzma\"))) or http.request.uri.path in {\"/\" \"/index.html\" \"/timestamp\" \"/CTAN.sites\" \"/README.mirrors\" \"/FILES.byname\" \"/FILES.byname.gz\" \"/FILES.last07days\"}))",
      "action": "set_cache_settings",
      "action_parameters": { "cache": false }
    },
    {
      "ref": "ctan_mirror_files",
      "description": "cache the tree one day; purge on change @SHA@",
      "enabled": true,
      "expression": "(http.host eq \"ctan.ijosh.com\" and not (starts_with(http.request.uri.path, \"/.state/\") or starts_with(http.request.uri.path, \"/.site/\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/tlpkg/\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/install-tl\") or starts_with(http.request.uri.path, \"/systems/texlive/tlnet/update-tlmgr\") or starts_with(http.request.uri.path, \"/systems/texlive/tlcontrib/tlpkg/\") or (http.request.uri.path contains \"/miktex/tm/packages/\" and (http.request.uri.path contains \"/miktex-zzdb\" or ends_with(http.request.uri.path, \"/pr.ini\") or ends_with(http.request.uri.path, \"/files.csv.lzma\"))) or http.request.uri.path in {\"/\" \"/index.html\" \"/timestamp\" \"/CTAN.sites\" \"/README.mirrors\" \"/FILES.byname\" \"/FILES.byname.gz\" \"/FILES.last07days\"}))",
      "action": "set_cache_settings",
      "action_parameters": {
        "cache": true,
        "edge_ttl": {
          "mode": "override_origin",
          "default": 86400,
          "status_code_ttl": [
            { "status_code_range": { "to": 299 }, "value": 86400 },
            { "status_code_range": { "from": 300 }, "value": -1 }
          ]
        },
        "browser_ttl": { "mode": "override_origin", "default": 14400 },
        "cache_key": { "custom_key": { "query_string": { "exclude": ["*"] } } },
        "serve_stale": { "disable_stale_while_updating": true }
      }
    }
  ]
}
```

Choices:

- Edge TTL one day, not "long". Free's minimum is 2 hours; the dashboard offers up to a year
  (the API schema fetched today states no maximum for `edge_ttl.default`; unverified). Purge
  keeps the cache honest while purge works; one day is the bound on how long a purge that
  silently failed can serve old bytes. Cost of the shorter TTL: one refetch per warm object per
  day, ~13k Class B a day for a hot set of ~13k objects (one install set plus binaries for a few
  platforms), 0.4M a month, free.
- Status code TTL: 2xx one day (206 included, since a range miss stores the whole object), 3xx
  and up no-store. Combining `default` with `status_code_ttl` under `override_origin` is the
  dashboard's "Edge TTL + Status code TTL" pair; that the API accepts this exact shape is
  unverified until the first `PUT`.
- Browser TTL 4 hours explicitly, equal to the zone default, so the `cache-control` a client sees
  is the same whether the switch is on or off. Purge cannot reach a browser or proxy; `tlmgr`
  has no cache; 4 hours is what Apache mirrors' clients get from heuristics anyway.
- `disable_stale_while_updating`: R2 never sends `stale-while-revalidate`, so this is a
  no-op guard against a future header.
- Not set: `respect_strong_etags` (R2's ETag is already quoted and strong; irrelevant with an
  override TTL), `origin_cache_control` (forced on for Free), `cache_reserve` (paid; not used, so not
  evaluated), `vary`.

### `cloudflare/cache-off.json`

```json
{
  "description": "ctan cache rules @SHA@",
  "rules": [
    {
      "ref": "ctan_bypass_all",
      "description": "cache off: nothing on the host is stored @SHA@",
      "enabled": true,
      "expression": "(http.host eq \"ctan.ijosh.com\")",
      "action": "set_cache_settings",
      "action_parameters": { "cache": false }
    }
  ]
}
```

This is the default. It removes today's 120-minute windows on the default-extension files and
the 3-minute 404 windows, and costs nothing: every request is DYNAMIC and one Class B.

### `cloudflare/transform.json`

```json
{
  "description": "ctan transform rules @SHA@",
  "rules": [
    {
      "ref": "root_to_index",
      "description": "/ -> /.site/index.html (the landing page; CTAN's own index.html stays at /index.html) @SHA@",
      "enabled": true,
      "expression": "(http.host eq \"ctan.ijosh.com\" and http.request.uri.path eq \"/\")",
      "action": "rewrite",
      "action_parameters": { "uri": { "path": { "value": "/.site/index.html" } } }
    }
  ]
}
```

Phase `http_request_transform`; "Active Transform Rules \| 10" on Free, "Regex support \| No"
([Transform Rules](https://developers.cloudflare.com/rules/transform/)); this is a static
rewrite, no regex. `/.site/index.html` is the landing page; CTAN's own `index.html` stays at
`/index.html`.

### `cloudflare/redirect-rules.json`

Directory URLs (`.../macros/latex/contrib/`) are 404s on R2. The Single Redirect from
`official-mirror-and-url.md` sends them to CTAN's browsable tree. It lives here so every rule
on the zone is in one place, and `task rules` applies it whatever `CACHE` says.

```json
{
  "description": "ctan redirect rules @SHA@",
  "rules": [
    {
      "ref": "directory_to_ctan",
      "description": "directory URLs -> ctan.org/tex-archive (R2 has no listings) @SHA@",
      "enabled": true,
      "expression": "(http.host eq \"ctan.ijosh.com\" and http.request.uri.path ne \"/\" and ends_with(http.request.uri.path, \"/\"))",
      "action": "redirect",
      "action_parameters": {
        "from_value": {
          "target_url": { "expression": "concat(\"https://ctan.org/tex-archive\", http.request.uri.path)" },
          "status_code": 302,
          "preserve_query_string": false
        }
      }
    }
  ]
}
```

Phase `http_request_dynamic_redirect`, zone level; shape from
[Create a redirect rule via API](https://developers.cloudflare.com/rules/url-forwarding/single-redirects/create-api/)
(`"action": "redirect"`, `from_value.target_url.expression`, `status_code`,
`preserve_query_string`). A 302 is answered before any cache lookup, so nothing here is ever
stored (3xx are `no-store` in the cache rule in any case).

### `task rules`

Every `PUT` "creat[es] a new version" (the API reference's own description of the entrypoint
update: "Updates an account or zone entry point ruleset, creating a new version"; "Each
ruleset modification creates a new version of the ruleset",
[About rulesets](https://developers.cloudflare.com/ruleset-engine/about/rulesets/)). 720
identical versions a month is noise, and the Rulesets API has unpublished rate limits on
updating one ruleset ("You should avoid making concurrent updates to the same ruleset",
[Rulesets API](https://developers.cloudflare.com/ruleset-engine/rulesets-api/)). So the task
`PUT`s only when the file changed: the file's SHA-256 prefix is stamped into every description
at `PUT` time, and a `GET` that already shows it skips the `PUT`. No `jq`: `grep` on the
response is the whole comparison.

```yaml
vars:
  # on | off. Selects cloudflare/cache-{{.CACHE}}.json, whether publish purges, and what smoke
  # expects (HIT or DYNAMIC). Off is the default; see docs/full-mirror/caching.md section 11.
  CACHE: 'off'
  CF: https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}
  CURL: curl -fsS --connect-timeout 15 --max-time 60 --retry 6 --retry-max-time 600
  AUTH: -H "Authorization: Bearer ${CF_API_TOKEN}"

rules:
  desc: Put the zone's cache and transform rules from cloudflare/*.json when a file changed; check Smart Tiered Cache
  dir: '{{.ROOT_DIR}}'
  cmds:
    - test -n "$CF_API_TOKEN" && test -n "$CF_ZONE_ID"
    # One PUT per phase, only when the stamped hash is not already on the zone. A missing
    # entrypoint answers 404, curl -f fails the GET, and the PUT creates it.
    - >-
      for p in http_request_cache_settings:cache-{{.CACHE}} http_request_transform:transform http_request_dynamic_redirect:redirect-rules; do
      f=cloudflare/${p#*:}.json; sha=$(shasum -a 256 "$f" | cut -c1-16);
      {{.CURL}} {{.AUTH}} {{.CF}}/rulesets/phases/${p%%:*}/entrypoint | grep -q "$sha"
      || sed "s/@SHA@/$sha/g" "$f" | {{.CURL}} {{.AUTH}} -X PUT --json @- {{.CF}}/rulesets/phases/${p%%:*}/entrypoint > /dev/null
      || exit 1; done
    # Smart Tiered Cache is set once in the dashboard; the run only refuses to cache without it.
    - >-
      test {{.CACHE}} = off ||
      {{.CURL}} {{.AUTH}} {{.CF}}/cache/tiered_cache_smart_topology_enable | grep -q '"value":"on"'
```

Runs at the head of `sync`, before `fetch`: a rejected file fails the run before anything is
uploaded. Three `GET`s per run; a `PUT` only after a merged change. Whether the entrypoint `PUT`
creates the phase ruleset when none exists is unverified (the create-api page says to create it
with `POST /zones/{zone_id}/rulesets` and `"kind": "zone", "phase": ...` if absent); the first
run on a fresh zone tells, and the fallback is that one `POST` by hand.

### `task purge`

```yaml
purge:
  desc: Purge RUN/purge.txt (changed and deleted keys, never added ones) from the edge, 100 URLs per call
  set: [pipefail]
  cmds:
    - >-
      test {{.CACHE}} = off || ! test -s {{.RUN}}/purge.txt ||
      sed 's/%/%25/g; s/ /%20/g; s/#/%23/g; s/"/%22/g; s/?/%3F/g; s|^|{{.SITE}}/|' {{.RUN}}/purge.txt
      | xargs -n 100 | sed 's/ /","/g; s/^/{"files":["/; s/$/"]}/'
      | while read -r body; do
      {{.CURL}} {{.AUTH}} --json "$body" {{.CF}}/purge_cache > /dev/null || exit 1; sleep 0.3; done
```

`RUN/purge.txt` is `changed ∪ deleted` from the list diff (`taskfile-architecture.md`): the
batch's uploads intersected with the state file's paths, plus the deletion list. The quotes are
added after `xargs`, which would otherwise strip them; after the encoding no URL contains a
space, quote or backslash (0 keys on CTAN contain `'` or `\`), so `xargs -n 100` splits cleanly
and keeps every call inside "Max operations per request \| 100". Dry run on the odd keys:

```
{"files":["https://ctan.ijosh.com/fonts/mtp2lite/templates/Plain%20MTPro2.tex","https://ctan.ijosh.com/language/cyrtug/%23disk.00"]}
```

`sleep 0.3` holds the stream to ~330 URLs/s and ~1,000 calls per five minutes, under both
limits. A failed call fails the run, and the batch repeats next hour because the state file is
written only after the purge.

### Token and secrets

One token, `CF_API_TOKEN`, zone resources limited to `ijosh.com`; one id, `CF_ZONE_ID`.
Permission names as they appear on the [permissions page](https://developers.cloudflare.com/fundamentals/api/reference/permissions/)
(the zone-scoped table, whose API names are `Cache Settings Write` and so on):

| Permission | Why | Source |
|---|---|---|
| Zone → Cache Rules → Edit (`Cache Settings Write`) | the cache phase `PUT` | "Zone > Cache Rules > Edit" on the create-api page; `Cache Settings Write` in the update page's list |
| Zone → Transform Rules → Edit | the transform phase `PUT` | "Zone Transform Rules Write" / "Transform Rules Write" in the same list |
| Zone → Cache Purge | `purge_cache` | "Accepted Permissions ... `Cache Purge`" |
| Zone → Single Redirect → Edit | the redirect phase `PUT` | "Single Redirect Edit" in the zone table of the permissions page |
| Zone → Zone Settings → Read | the Smart Tiered Cache `GET` | "`Zone Settings Write` `Zone Settings Read` `Zone Read` `Zone Write`" on the GET reference |
| Account → Account Analytics → Read | `report`'s usage query (`monitoring.md`) | `monitoring.md` |

The create-api page also lists "Account Rulesets > Edit" and "Account Filter Lists > Edit".
Those serve account-level rulesets and lists in expressions; this design uses neither.
Unverified whether the zone `PUT` succeeds without them; try without, add only on a 403.

### What a fork changes

`ctan.ijosh.com` appears in every expression (five, across the four files) and `HOST` in the Taskfile. A
`sed` at `PUT` time (`s/ctan.ijosh.com/{{.HOST}}/g` next to the `@SHA@` one) removes the
duplication; then a fork changes one Taskfile line and nothing in `cloudflare/`. Recommended.

### `check.yml`

`jq` is not in the tool list. `python3 -m json.tool` is on `ubuntu-latest` but is not in the
list either, and adding it for a syntax check is not worth a constraint line. What validation
looks like with the allowed tools: none offline. The `PUT` is the validator, it is the first
step of the run, it runs before `fetch`, and a bad file costs one failed run and a revert. A PR
that touches `cloudflare/` is followed by a `workflow_dispatch` run the author watches. If that
is judged too loose, the one line to change is CLAUDE.md's "Tools are exactly ..." to add
`python3` for `check.yml` only, and the check is
`for f in cloudflare/*.json; do python3 -m json.tool "$f" > /dev/null; done`.

## 8. Files that must never be cached

A "decision file" is anything a client reads to decide what else to fetch or to verify what it
fetched. The rule covers CTAN as a whole. Derived from the listing
(`ctan-list-deref.txt`, 2026-08-26) with the commands in parentheses.

| Path(s) | Reader | Why uncached | Default-cacheable today? |
|---|---|---|---|
| `/timestamp` | mirmon, the CTAN monitor | touched hourly; a stale copy delists the mirror | no (no extension) |
| `/CTAN.sites`, `/README.mirrors` | humans, `tlmgr`'s mirror query | daily | no |
| `/FILES.byname`, `/FILES.byname.gz`, `/FILES.last07days` | humans, scripts | daily indexes | **`.gz` yes**: 120 min stale today |
| `/` (rewritten to `/.site/index.html`), `/.site/**` | browsers | the landing page, kept outside CTAN's paths | no (HTML) |
| `/index.html` | browsers | CTAN's own index page, unchanged since 2020 | no (HTML) |
| `/.state/**` | the next run (`applied.txt.xz`) | the listing the last run applied, plus its history; a cached copy would replay an old delta | no (`.xz`) |
| `/systems/texlive/tlnet/tlpkg/**` | `tlmgr`, `install-tl` | `texlive.tlpdb(.xz|.sha512|.sha512.asc|.md5)`, `gpg/` keyring, `installer/`, `tlperl/`, `tltcl/`, `TeXLive/`, `translations/` | 30 `.exe`/`.gif` under `installer/`, `tlperl/`, `tltcl/` are; the rest not |
| `/systems/texlive/tlnet/install-tl*` | humans, `install-tl-windows.bat` | `install-tl`, `install-tl.zip`, `install-tl-unx.tar.gz`, `install-tl-windows.exe`, each with `.sha512` and `.asc`; rebuilt daily (all dated 2026-08-25 16:53) | **`.zip`, `.tar.gz`, `.exe` yes** |
| `/systems/texlive/tlnet/update-tlmgr*` | `tlmgr update --self` fallback, humans | `update-tlmgr-latest.sh/.exe` with checksums | **`.exe` yes** |
| `/systems/texlive/tlcontrib/tlpkg/**` | `tlmgr` with the tlcontrib repository | its own `texlive.tlpdb*` (`grep tlpdb ctan-list-deref.txt`) | no |
| `/systems/win32/miktex/tm/packages/{,next/}miktex-zzdb*`, `pr.ini`, `files.csv.lzma` (and the `systems/windows/` alias copy) | MiKTeX's package manager | the package database (`miktex-zzdb1/2/3-2.9.tar.lzma`, 2026-08-09), repository manifest and file index (`grep -E 'miktex/tm/packages/(next/)?(miktex-zzdb\|pr\.ini\|files\.csv)'`) | no (`.lzma`, `.ini` not listed) |

Checked and left cacheable:

- `README.md` files, `index.html` inside package directories (8,507 `.html`): documentation,
  not indexes; HTML is not cached anyway.
- `/tds.zip`: a rarely changing archive; cached is fine.
- `systems/win32/miktex/setup/`: installers and a Debian repository (`InRelease`, `Packages`,
  `Contents-*.gz`). `apt` verifies `InRelease` signatures against `Packages` hashes, the same
  shape as `tlmgr`. Left cacheable on purpose: those files change on MiKTeX releases, not
  hourly, and purge-on-change covers them. If a report ever shows an `apt` hash mismatch,
  add `starts_with(http.request.uri.path, "/systems/win32/miktex/setup/deb/dists/")` to `D`.
- `.ini`, `.lst`, `.json`, `.xml`, `.csv` elsewhere (124, 55, 165, 610, 129 files): package
  contents, not indexes (`csl-locales-*.xml`, `luatex.ini` formats).
- `systems/texlive/tlnet/README.md`: text, 1.2 KB, changes rarely.
- No `ls-R`, no `.mirror`, no `MIRRORS` file exists at the root (`awk '$5 !~ /\//'`).

Only `tlnet` and `tlcontrib` have checksums in the index; MiKTeX's database carries digests of
its own packages. Everything else on CTAN is a plain file with no client-side verification, so
for it a stale edge means a stale file for at most one day, the same as a mirror that syncs
daily.

## 9. Correctness, end to end

Clients: `tlmgr` (downloads `texlive.tlpdb.xz`, then `texlive.tlpdb.sha512` and `.asc` via
`TLCrypto::verify_checksum`, then containers; each container is checked against the tlpdb's
`containerchecksum` in `TLUtils::check_file_and_remove`, which on mismatch warns "checksums
differ", removes the file and returns failure; `install_packages` queues a failed package for one
retry at the end when its caller asks), `install-tl` (same code), browsers and download
managers (installers, ISOs, documentation), MiKTeX (its database, then `.tar.lzma` packages),
mirmon (`/timestamp`).

Windows in which a client can see inconsistent data, with the rule on, PUT-then-purge, purge of
changed keys before `tlpkg/` lands, 404 no-store, Edge TTL one day:

| # | Situation | Width | User sees | Today (no purge, default extensions) | Apache mirror |
|---|---|---|---|---|---|
| 1 | Container X changed; a client with the **old** tlpdb requests X between X's PUT and the purge, at a data center where X is **not** cached | seconds to minutes (the batch's upload plus purge, P50 ~0.25 s) | new bytes, old checksum: `tlmgr` warns, deletes, retries once, fails the package; rerun fixes | same window (archive before tlpdb) | same window (rsync writes archive before tlpkg) |
| 2 | Same, X **is** cached at that data center | none: the old bytes serve until the purge, and the tlpdb changes only after it | correct old X | n/a (`.xz` never cached) | n/a |
| 3 | Client with the **new** tlpdb, old X still cached | none by construction: the purge precedes the tlpdb upload | | n/a | n/a |
| 4 | `texlive.tlpdb.sha512` uploaded before `texlive.tlpdb.xz` (byte order inside `tlpkg/`) and a client fetches `.xz` old, `.sha512` new | milliseconds, once an hour at most | "checksum error", `tlmgr` aborts, rerun fixes | same | same shape (rsync order) |
| 5 | Purge returned success but did not take | until the Edge TTL: ≤ 24 h | checksum error on that key; `smoke`'s three-key sample catches most runs | n/a | n/a |
| 6 | Deleted object still cached | until the purge after the delete, ~0.25 s | old file | 120 min for default extensions | none |
| 7 | New object at a path previously requested as 404 | none: 404 is never stored | | 3 min on default extensions, observed | none |
| 8 | Installer changed (`install-tl.zip` daily rebuild) | none: bypassed | | **120 min** with its `.sha512` fresh: a manual verification fails | none |
| 9 | `FILES.byname.gz` daily rebuild | none | | 120 min (full mirror only) | none |
| 10 | Downstream proxy or browser holding a container | up to 4 h after a change (`max-age=14400`; purge cannot reach it) | checksum error for users behind that proxy | same header today on cached files; no header on DYNAMIC ones, so heuristics (often longer) | no header; heuristic freshness, typically 10 % of the file's age |
| 11 | Cloudflare data center serves a HIT with old bytes after a purge (tiered revalidation race) | unverified; docs say prefix/host purge may show EXPIRED while the lower tier revalidates with the upper | possibly one stale response | n/a | n/a |

Backstop for every row: `tlmgr` verifies every container against a signed index on the client,
so a stale edge is an error message, never a wrong file installed. Files outside `tlnet`,
`tlcontrib` and MiKTeX have no client check anywhere, on any mirror.

Window 1 is the one that exists today and on every mirror. Window 4 is fixable in
`taskfile-architecture.md` by uploading `texlive.tlpdb.xz` before the rest of `tlpkg/`
(`aws s3 cp` it first, then `cp --recursive` the directory). Windows 5 and 10 are the price of
caching; both are bounded, one by the TTL and one by the browser TTL. Windows 6 to 9 are today's
mirror's, and the "off" rule removes them.

## 10. Verification in `smoke`

`cf-cache-status` meanings ([Cache responses](https://developers.cloudflare.com/cache/concepts/cache-responses/)):

| Value | Meaning | Mirror key, 2nd GET, `CACHE=on` | Decision file, or `CACHE=off` |
|---|---|---|---|
| HIT | "found in Cloudflare's cache" | expected; `age` present | wrong: the rule is not applying |
| MISS | "eligible for cache but was not present" | tolerable once (first GET, or evicted between the two); fail on the third | wrong |
| DYNAMIC | "not eligible for cache ... without a cache lookup" | wrong: the rule is missing | expected |
| BYPASS | eligible "but the origin response was ultimately not cacheable" (over 512 MB, `no-store`, `Set-Cookie`) | wrong for a normal key; expected for the 7 files over 512 MB | wrong |
| EXPIRED | "found ... but was expired and served from the origin" | only after a hostname purge with tiered cache; tolerable once | wrong |
| REVALIDATED | origin confirmed via `If-None-Match` | should not occur with an override TTL and no revalidation directive | wrong |
| UPDATING, STALE | stale served during/after revalidation | should not occur (`disable_stale_while_updating`, R2 sends no `stale-while-revalidate`) | wrong |

`Age` "is returned when Cloudflare serves a response from cache ... Cloudflare sets `Age` on
`HIT`, `STALE`, and `UPDATING` responses. It does not set `Age` on `MISS`".

```yaml
smoke:
  desc: Read the tree back through the domain; prove the cache rule on the zone is the one in the repo
  vars:
    EXPECT: '{{if eq .CACHE "on"}}HIT{{else}}DYNAMIC{{end}}'
  cmds:
    # staging/ is emptied after every batch, so publish copies what smoke will compare into
    # RUN/sample/ at upload time: timestamp, tlpkg/texlive.tlpdb.sha512 when uploaded, and three
    # keys outside D listed in RUN/sample.txt. Every download lands in a file before cmp.
    - {{.CURL}} -o {{.RUN}}/smoke.body {{.SITE}}/timestamp && cmp {{.RUN}}/smoke.body {{.RUN}}/sample/timestamp
    # Up to three GETs per key: the first after a purge is a MISS, the second a HIT; a third
    # covers an eviction between them. Bytes must match the copy on the final GET.
    - >-
      while read -r k; do for i in 1 2 3; do
      {{.CURL}} -o {{.RUN}}/smoke.body -D {{.RUN}}/smoke.head "{{.SITE}}/$k";
      grep -qi "^cf-cache-status: {{.EXPECT}}" {{.RUN}}/smoke.head && break;
      test $i -lt 3 || { cat {{.RUN}}/smoke.head; exit 1; }; done;
      cmp {{.RUN}}/smoke.body "{{.RUN}}/sample/$k" || exit 1; done < {{.RUN}}/sample.txt
    # A decision file is never a HIT, whatever the switch says.
    - {{.CURL}} -I {{.SITE}}/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512 | grep -qi '^cf-cache-status: DYNAMIC'
    - >-
      test ! -f {{.RUN}}/sample/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512 ||
      { {{.CURL}} -o {{.RUN}}/smoke.body {{.SITE}}/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
      && cmp {{.RUN}}/smoke.body {{.RUN}}/sample/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512; }
```

What this proves: the rule on the zone makes mirror keys cacheable (HIT) and decision files not
(DYNAMIC), the purge took for the sampled keys (bytes equal staging on a HIT), and the domain
serves the tree just published. What it cannot prove: that every purged key took (sampling), or
what another data center serves (the runner sees one). The `-I` HEAD is a full GET on a miss,
which for a bypassed file is one Class B either way.

One-hour experiment for the unverified tiered-cache purge behaviour (section 4), read-only
except one purge: with the rule on, pick a changed key from a run, GET it from the runner and
from a laptop in another region until both HIT, note `age`; after the next run's purge, GET
from both within a second; both must be MISS then HIT with the new `etag`. An EXPIRED or an old
`etag` on either side is the tiered-cache race in row 11.

## 11. Fallback: caching off

`CACHE: 'off'` is the shipped default and the constraint set today, plus one rule and one
read-only check:

- `cloudflare/cache-off.json` on the zone (bypass the host). Set once by `task rules` with the
  token, or once by hand in the dashboard with the same expression; `smoke` then asserts DYNAMIC
  and fails if someone adds a caching rule by hand. With no rule at all the mirror behaves as
  today: default extensions cached 120 minutes, 404s 3 minutes, no purge; the windows in rows 6
  to 9.
- No purge step, no `purge.txt`, no rate limits to think about.
- No `CF_API_TOKEN` if the rule is set by hand; the transform rule (`/` to `/.site/index.html`) and
  the directory redirect are applied by `task rules` whatever the switch says. Keeping `task rules` with the token is
  still recommended so the zone's rules are reproducible from the repo; the token then has
  Cache Rules Edit, Transform Rules Edit, Single Redirect Edit and Zone Settings Read, not Cache Purge.

The bill at that point, with the free tier: $0 up to 28 `scheme-full` installs a day; then
$0.36 per million GETs; ftp.fau.de's 2018 CTAN traffic would be $2.50 to $7 a month. Storage is
unchanged by the switch.

Trigger to switch on: `report` (`monitoring.md`) prints month-to-date Class B; when it passes
5 million in two consecutive months, or when users far from the bucket report install times,
flip `CACHE` to `on`, merge, run once by hand, watch `smoke`'s HIT line, and purge everything
once from the dashboard to drop any default-extension copies. Switching off again is the
reverse; the bypass rule takes effect at the next `rules` run and existing entries are simply
ignored.

## Open questions

1. Whether the Free plan accepts `cache_key.custom_key.query_string.exclude` and the
   `default` + `status_code_ttl` pair under `override_origin` in one `PUT`. The first `PUT`
   answers; if refused, drop `cache_key` (a cache-buster then costs one Class B per distinct
   query string, which it does today anyway).
2. Whether a zone entrypoint `PUT` creates the phase ruleset when none exists. Fallback: one
   `POST /zones/{zone_id}/rulesets` with `kind: zone` and the phase.
3. Whether `Cache Rules Edit` alone authorises the cache phase `PUT`, without the account-level
   Rulesets and Filter Lists permissions the docs list.
4. Single-file purge and the upper tier: whether a lower tier can serve a purged object from an
   upper tier that has not yet been purged (row 11). Experiment in section 10.
5. Range requests on objects over 512 MB: whole-object fetch or pass-through (section 3
   experiment).
6. The maximum `edge_ttl.default` the API accepts (the dashboard offers one year).
7. Cache rule precedence when two rules match: avoided by disjoint expressions, but a single
   rule with a `not` would be simpler if "last matching rule wins" were documented.
8. Whether purge calls count against the global 1,200-per-5-minutes token limit. Budgeted as if
   they do.
9. For `monitoring.md`: a rate-limiting rule (Free has one) as the only real bound on a 404
   flood; the Class B threshold at which `CACHE` flips.
10. For `taskfile-architecture.md`: `tlpkg/` as its own final batch with the purge before it;
    `texlive.tlpdb.xz` uploaded before `texlive.tlpdb.sha512`; `purge.txt` as
    `changed ∪ deleted`; `RUN/sample/` (copies made at upload time) and `sample.txt` of three uploaded keys outside `D` for `smoke`.
11. For `seeding-and-migration.md`: seed with `CACHE: 'off'`; purge everything once at switch-on.
12. For `official-mirror-and-url.md`: MiKTeX's `setup/deb/` repository left cacheable; whether
    the `systems/windows/` alias copy of the MiKTeX tree should exist at all (it doubles the
    decision-file list).

## Sources

Fetched 2026-08-26 unless dated otherwise.

- https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/
- https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- https://developers.cloudflare.com/cache/concepts/cache-responses/
- https://developers.cloudflare.com/cache/concepts/cache-behavior/
- https://developers.cloudflare.com/cache/concepts/cache-control/
- https://developers.cloudflare.com/cache/concepts/revalidation/
- https://developers.cloudflare.com/cache/how-to/cache-rules/
- https://developers.cloudflare.com/cache/how-to/cache-rules/settings/
- https://developers.cloudflare.com/cache/how-to/cache-rules/create-api/
- https://developers.cloudflare.com/cache/how-to/cache-keys/
- https://developers.cloudflare.com/cache/how-to/configure-cache-status-code/
- https://developers.cloudflare.com/cache/how-to/edge-browser-cache-ttl/
- https://developers.cloudflare.com/cache/how-to/purge-cache/
- https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-single-file/
- https://developers.cloudflare.com/cache/how-to/purge-cache/purge-everything/
- https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-hostname/
- https://developers.cloudflare.com/cache/how-to/purge-cache/purge_by_prefix/
- https://developers.cloudflare.com/cache/how-to/tiered-cache/
- https://developers.cloudflare.com/cache/advanced-configuration/cache-reserve/
- https://developers.cloudflare.com/api/resources/cache/methods/purge/
- https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/get/
- https://developers.cloudflare.com/api/resources/cache/subresources/smart_tiered_cache/methods/edit/
- https://developers.cloudflare.com/api/resources/rulesets/subresources/phases/methods/update/
- https://developers.cloudflare.com/ruleset-engine/rulesets-api/
- https://developers.cloudflare.com/ruleset-engine/rulesets-api/update/
- https://developers.cloudflare.com/ruleset-engine/about/rulesets/
- https://developers.cloudflare.com/rules/transform/
- https://developers.cloudflare.com/rules/transform/url-rewrite/create-api/
- https://developers.cloudflare.com/fundamentals/api/reference/permissions/
- https://developers.cloudflare.com/fundamentals/api/reference/limits/
- https://developers.cloudflare.com/speed/optimization/content/compression/
- https://developers.cloudflare.com/r2/api/s3/api/
- https://developers.cloudflare.com/r2/buckets/public-buckets/
- https://developers.cloudflare.com/r2/reference/consistency/
- https://developers.cloudflare.com/r2/pricing/ (scratchpad copy of the same day)
- https://developers.cloudflare.com/billing/understand/how-charges-accrue/ (scratchpad copy)
- https://blog.cloudflare.com/instant-purge/ (2024-09-24)
- https://blog.cloudflare.com/instant-purge-for-all/ (2025-04-01)
- https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html
- https://ftp.fau.de/news/posts/mirrors-generating-most-traffic-in-2018/ (2019-09-23)
- https://ctan.org/mirrors/register (scratchpad copy)
- https://yihui.org/en/2026/03/tinytex-ctan-mirror/ (a Cloudflare-fronted tlnet mirror; no traffic figures published)
- `staging/tlpkg/TeXLive/TLConfig.pm`, `TLUtils.pm`, `TLPDB.pm`, `TLCrypto.pm` (tlnet, 2026-08-25)
- `staging/tlpkg/texlive.tlpdb` (2026-08-25); `ctan-list-deref.txt` (2026-08-26 ~17:00 UTC)
- Live probes: `curl -sI` / `curl -r` against `https://ctan.ijosh.com/` and
  `https://mirror.ctan.org/timestamp`, 2026-08-26
